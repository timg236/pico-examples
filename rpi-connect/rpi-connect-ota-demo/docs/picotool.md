# Provisioning with picotool

[← Back to the overview](../README.md)

`picotool provision connect` gives a device its Raspberry Pi Connect
credentials. It loads a small provisioning binary into the device's SRAM and
runs it; any secrets you pass are written only into that RAM image, never to
flash.

Choose one of:

| Method                                          | Survives FFS erase? | Needs                       |
|-------------------------------------------------|---------------------|-----------------------------|
| [Device identity, key generated on the device](#device-identity-key-generated-on-the-device) (recommended) | Yes | Organisation token, programs OTP |
| [Device identity, key generated on the host](#device-identity-key-generated-on-the-host) | Yes | Organisation token, programs OTP |
| [Auth key](#auth-key)                           | No                  | Connect auth key            |
| [Browser sign-in](#browser-sign-in)             | No                  | A browser                   |

With a device identity the device can always derive a fresh access token from
its OTP key. With an auth key or sign-in, the token lives only in FFS: if FFS is
erased (for example by re-flashing the combined UF2) or the token expires, the
device must be provisioned again. Use those for development only.

## Before you start

* Flash `rpi_connect_ota_demo_combined.uf2` first (see the
  [quickstart](../README.md#2-flash)). The provisioning binary needs its FFS
  partition, and loads the WiFi firmware from its partition.
* Put the device into BOOTSEL mode.
* Use a `picotool` that has the `provision` command
  (`picotool help provision connect` lists every option).

Options common to every method:

| Option                                    | Purpose                                                   |
|-------------------------------------------|-----------------------------------------------------------|
| `--wifi-ssid <ssid> --wifi-password <pw>` | Store WiFi credentials in FFS first (if not built into the combined UF2) |
| `--device-name <name>`                    | Device name; defaults to `pico-<board id>`               |
| `--board <name>`                          | Take UART, LED and WiFi pin defaults from an SDK board (otherwise Pico 2 W pins) |
| `--uart`, `--led`, `--wl-*`, ...          | Override individual pins                                  |

Progress is printed on the device's USB and UART consoles. When finished the
LED flashes — **slowly on success, quickly on failure** — and the device
reboots to BOOTSEL. Run `picotool reboot` to start the application.

---

## Device identity, key generated on the device

The simplest and most secure option. If OTP holds no identity key, the device
generates a P-256 key pair with its own entropy source, writes the private key
to OTP (with ECC, rows `0xc0`–`0xcf`), and registers the public key with your
organisation. The private key never leaves the chip.

```sh
picotool provision connect --create-identity \
    --org-token   "<your-organisation-token>" \
    --description "Pico 2 W OTA demo" \
    --device-name "pico-ota-01"
```

> **OTP is one-time programmable.** Once a key is written it can never be
> changed or erased. Running `--create-identity` again reuses the existing key
> rather than generating a new one.

`--create-identity` also deletes any access token already stored in FFS, so the
application uses the new identity on its next boot.

To check the registration worked without starting the application, exchange the
identity for a token on the device (this also caches the token in FFS):

```sh
picotool provision connect --identity-exchange
```

## Device identity, key generated on the host

Generate the key on the host if you need a copy of it — for example to run the
[host-side debugging tools](debugging.md#testing-on-the-host), or to build a
development image with the key compiled in instead of using OTP
([see Debugging](debugging.md#bypassing-otp-with-a-build-time-key)).

```sh
openssl ecparam -name prime256v1 -genkey -noout -out device-priv-key.pem
openssl ec -in device-priv-key.pem -pubout -out device-pub-key.pem
```

> **Keep the private key safe.** Anyone holding it can impersonate the device.
> Store it with restricted access (ideally a key store or HSM in production) and
> back it up.

Write it to OTP (the key occupies 16 rows from `0xc0`, matching
`RPI_CONNECT_IDENTITY_OTP_ROW`):

```sh
picotool otp load -s 0xc0 device-priv-key.pem
```

Then register it exactly as above — `--create-identity` finds the key already in
OTP and uses it:

```sh
picotool provision connect --create-identity \
    --org-token "<your-organisation-token>" --device-name "pico-ota-01"
```

Alternatively, register from the host without the device attached, using the
host CLI (`rpi-connect-test`, see [Debugging](debugging.md#testing-on-the-host)):

```sh
rpi-connect-test --create-device-identity \
    --org-token      "<your-organisation-token>" \
    --device-privkey device-priv-key.pem \
    --device-pubkey  device-pub-key.pem \
    --description    "Pico 2 W OTA demo" \
    --device-name    "pico-ota-01"
```

The request is signed with the private key to prove ownership of the pair.

## Auth key

Exchange a Raspberry Pi Connect auth key for an access token, stored in FFS:

```sh
picotool provision connect --auth-key "<your-auth-key>" --device-name "pico-ota-01"
```

## Browser sign-in

```sh
picotool provision connect --signin
```

Open the device's USB or UART console. Once on WiFi it prints:

```
User code: XXXX-XXXX
Visit https://... in your browser and enter the user code
```

Open the URL, sign in and enter the code. The device polls for up to 15
minutes, then stores the access token in FFS.

## Other operations

| Command                                              | Effect                                                    |
|------------------------------------------------------|-----------------------------------------------------------|
| `picotool provision connect --token <token>`         | Store an existing access token in FFS                     |
| `picotool provision connect --clear`                 | Delete the stored token and any in-progress deployment state |
| `picotool provision connect --wifi-only --wifi-ssid <ssid> --wifi-password <pw>` | Only update the WiFi credentials |

## Token priority

At boot `rpi_connect_ota_init()` uses the first credential it finds:

1. An access token in FFS (cached from an earlier exchange, or provisioned).
2. A build-time identity key ([debug only](debugging.md#bypassing-otp-with-a-build-time-key)).
3. The OTP identity key, via a device-identity exchange.

A token in FFS therefore takes precedence over the identity key. If the server
rejects it (HTTP 401) the demo deletes it and reboots, so the next boot performs
a fresh identity exchange. This is skipped on a try-before-you-buy boot, so a
broken update cannot discard a good token. `--create-identity` clears any stored
token for you; use `--clear` to discard one that is still valid.
