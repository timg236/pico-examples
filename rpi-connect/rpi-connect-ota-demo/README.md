# rpi-connect-ota-demo — OTA integration example

A complete, working example of adding **Raspberry Pi Connect** over-the-air
(OTA) firmware updates to a Pico application: sign in, listen for deployments,
download to the inactive flash slot, verify, and reboot into the new image.

If you are adding OTA to your own application, start with
`rpi_connect_ota_demo.c` — it is the smallest piece of code that drives the
`pico_rpi_connect_ota` API end to end.

| Page                                           | Covers                                                         |
|------------------------------------------------|----------------------------------------------------------------|
| This page                                      | What the example does, and a quickstart                        |
| [Provisioning with picotool](docs/picotool.md) | Device identity (including on-device key generation), auth keys, sign-in, WiFi |
| [API walkthrough](docs/api.md)                 | The `pico_rpi_connect_ota` call sequence and deployment lifecycle |
| [Debugging](docs/debugging.md)                 | Serial console, log levels, debug overrides, host-side testing |

---

## What it demonstrates

* Obtaining a Connect **access token** at runtime from a P-256 **device identity
  key held in OTP**, so no key or token is carried in the firmware image.
* Caching the token and WiFi credentials in **FFS** (flash file storage).
* Listening for deployments over a persistent SSE connection.
* Streaming an image into the **inactive A/B partition**, verifying its SHA-256
  and rebooting into it via the RP2350 bootrom.
* **Try-before-you-buy**: an update is only committed, and reported as
  successful, once the new image has booted and reached the Connect API —
  otherwise the bootrom rolls back.
* **Resuming** a deployment interrupted by a reboot.

## Requirements

* **Board:** Pico 2 W. OTA is RP2350-only — it relies on the bootrom
  flash-update mechanism, A/B partitions and OTP.
* **Host:** the Pico SDK with submodules initialised, and a
  [`picotool`](https://github.com/raspberrypi/picotool) with the
  `provision connect` command.
* **Raspberry Pi Connect account.** To use a device identity (recommended) you
  need an **organisation** and an **organisation token**. A personal account is
  enough if you provision with an auth key or sign-in instead.

## Where the secrets live

Nothing secret is compiled into the firmware:

| Secret               | Stored in        | Provisioned by                                                    |
|----------------------|------------------|-------------------------------------------------------------------|
| Device identity key  | OTP (row `0xc0`) | `picotool provision connect --create-identity` (generated on the device) |
| WiFi credentials     | FFS              | the combined UF2, or `picotool provision connect --wifi-ssid ...` |
| Connect access token | FFS              | derived from the identity key at boot and cached, or provisioned directly |

At boot the device signs a *device-identity exchange* with its OTP key and
receives an access token, which is cached in FFS until it expires. Because OTP
survives a flash erase (including one caused by an OTA update), the device can
always obtain a fresh token. A token provisioned directly, without an identity,
lives only in flash and is lost with it.

---

## Quickstart

### 1. Build

Build the `rpi_connect_ota_demos` target in pico-examples. Optionally pass your
WiFi credentials so they are included in the FFS image:

```sh
cd <pico-examples-root>
cmake -S . -B build -DPICO_BOARD=pico2_w \
    -DWIFI_SSID="<ssid>" -DWIFI_PASSWORD="<password>"
cmake --build build --target rpi_connect_ota_demos
```

WiFi credentials only go into the `*_combined.uf2` files (via their FFS image),
never into the application UF2s.

### 2. Flash

Hold BOOTSEL while plugging in the board, then drag and drop
`rpi_connect_ota_demo_combined.uf2` onto the `RP2350` drive. This writes the
A/B partition table, the application, the WiFi firmware and the FFS image.

> Re-flashing the combined UF2 resets FFS, which clears the WiFi credentials
> and any cached token.

### 3. Provision

Put the board back into BOOTSEL mode and register it with your organisation.
If OTP has no identity key yet, the device generates one and writes it to OTP —
the private key never leaves the device:

```sh
picotool provision connect --create-identity \
    --org-token "<your-organisation-token>" \
    --device-name "pico-ota-01"
picotool reboot
```

> **This programs OTP, which cannot be undone.** To avoid using OTP, or to use
> an auth key or browser sign-in instead, see
> [Provisioning with picotool](docs/picotool.md).

Add `--wifi-ssid <ssid> --wifi-password <password>` if you didn't build them in.

### 4. Run

Open the serial console (115200 8N1) and reset the board. You should see:

```
rpi_connect_ota_demo version: 1.0.0 (image v1.<build>)
WiFi SSID: MyNetwork
Connecting to WiFi MyNetwork (client_id=... serial=<board id>)
Connected
No pending deployment
Starting event listener
```

The device has signed in, registered for OTA, checked for any deployment
assigned while it was offline, and is now waiting for new ones.

### 5. Deploy an update

The build copies update UF2s into the `updates` directory, named
`<target>-<version>-<board>-<date>.uf2` with a matching `.sha256sum`. These
have the try-before-you-buy bit set, so a bad image is only tried once.

In the Raspberry Pi Connect dashboard, create a deployment targeting the device
with one of these files. The device downloads it, verifies it and reboots:

```
Starting deployment ID=...
Download progress: 65536 / 524288 bytes (12%)
...
Download succeeded for deployment ID=... (524288 bytes sha256=...)
Rebooting to apply update...
```

> **Only deploy UF2s from `updates`.** Never upload a `*_combined.uf2`: it
> contains your WiFi credentials and would overwrite the running partition.

The image version comes from the git commit count, and the bootrom always boots
the higher version, so a deployment must be built from a later commit than the
image it replaces (or override with `-DRPI_CONNECT_BUILD_NUMBER=<n>`).

The deployment is reported successful only after the new image has booted and
signed in; see [API walkthrough](docs/api.md#deployment-lifecycle) for how
failure and rollback are handled.

---

## Code map

| File                            | Role                                                              |
|---------------------------------|-------------------------------------------------------------------|
| `rpi_connect_ota_demo.c`        | **The OTA integration** — the part to copy into your application. |
| `main_pico.c`                   | Device entry point: network/FFS bring-up, WiFi, sign-in, main loop. |
| `main.c`                        | Host build: registration and debug CLI (see [Debugging](docs/debugging.md#testing-on-the-host)). |
| `CMakeLists.txt`                | Targets, combined/update UF2 outputs and log levels.             |
| `debug_options.cmake`           | Development-only build-time identity key override.                |
| `rpi-connect-ota-host-build.sh` | Builds the host CLI.                                              |

The build produces three variants, each with its own combined and update UF2s:

* `rpi_connect_ota_demo` — the standard application.
* `rpi_connect_ota_bootsel_demo` — signs in, completes boot sync (committing
  any pending update), then returns to BOOTSEL.
* `rpi_connect_ota_packaged_demo` — a packaged `no_flash` binary, signed with
  `private.pem` for secure boot.
