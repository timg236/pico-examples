# Debugging

[← Back to the overview](../README.md)

## Serial console

All output goes to the SDK's standard I/O — UART or USB CDC, depending on
`pico_enable_stdio_uart` / `pico_enable_stdio_usb` — at **115200 baud, 8N1**.
The default `INFO` and `ERROR` output is enough to follow the whole flow; the
[quickstart](../README.md#4-run) shows the milestones to look for.

`ERROR` goes to `stderr` and `INFO` to `stdout` (see `rpi_connect_ota_demo.h`),
so they can be filtered separately where your console distinguishes the streams.

## Log levels

The application and library log namespaces are separate. Each level is a
compile-time switch in the logging block of `CMakeLists.txt`:

| Flag                                | Enables                                    | Default |
|-------------------------------------|--------------------------------------------|---------|
| `RPI_CONNECT_ERROR_ENABLE`          | Core `rpi_connect` ERROR                   | On      |
| `RPI_CONNECT_OTA_ERROR_ENABLE`      | OTA library ERROR                          | On      |
| `RPI_CONNECT_INFO_ENABLE`           | Core `rpi_connect` INFO                    | On      |
| `RPI_CONNECT_OTA_INFO_ENABLE`       | OTA library INFO                           | On      |
| `RPI_CONNECT_OTA_DEMO_DEBUG_ENABLE` | Demo DEBUG — per-chunk download progress   | Off     |
| `RPI_CONNECT_DEBUG_ENABLE`          | Core `rpi_connect` DEBUG, incl. HTTP layer | Off     |
| `RPI_CONNECT_VERBOSE_DEBUG_ENABLE`  | ...plus full header/body dumps             | Off     |
| `RPI_CONNECT_OTA_DEBUG_ENABLE`      | OTA library DEBUG                          | Off     |
| `HTTPC_DEBUG=LWIP_DBG_ON`           | lwIP HTTP client DEBUG                     | Off     |
| `RPI_CONNECT_MBEDTLS_DEBUG_LEVEL`   | mbedTLS verbosity, `0` (off) to `4`        | `0`     |

The demo's own `ERROR` and `INFO` output is always compiled in.

Where to start:

* **Deployment misbehaves** — enable `RPI_CONNECT_OTA_DEBUG_ENABLE` and
  `RPI_CONNECT_DEBUG_ENABLE`.
* **TLS handshake fails** — raise `RPI_CONNECT_MBEDTLS_DEBUG_LEVEL` (very verbose).

## Bypassing OTP with a build-time key

For bring-up you can compile a PEM identity key pair into the image instead of
programming OTP. **Development only** — the key ends up in the UF2.

Pass the key files from [a host-generated key](picotool.md#device-identity-key-generated-on-the-host)
on the CMake command line or as environment variables of the same name (see
`debug_options.cmake`):

```sh
cmake -S . -B build -DPICO_BOARD=pico2_w \
    -DRPI_CONNECT_IDENTITY_PRIVKEY_PEM=$PWD/device-priv-key.pem \
    -DRPI_CONNECT_IDENTITY_PUBKEY_PEM=$PWD/device-pub-key.pem
```

The public key must still be registered with your organisation. A token cached
in FFS takes precedence over this key; clear it with
`picotool provision connect --clear` if needed.

## Testing on the host

The host build of this example (`main.c`) runs the same `pico_rpi_connect`
library against host OpenSSL and curl. It is the quickest way to tell an API or
authentication problem apart from a device networking or provisioning problem.

### Build

Needs the libcurl and OpenSSL development packages; without them the SDK skips
`pico_rpi_connect` and the target does not exist. Use a separate build
directory and `PICO_BOARD=none`, so a device board in your environment doesn't
conflict with the host platform. `rpi-connect-ota-host-build.sh` runs:

```sh
cmake -S $PICO_EXAMPLES_PATH -B $PICO_EXAMPLES_PATH/build-host -DPICO_PLATFORM=host -DPICO_BOARD=none
cmake --build $PICO_EXAMPLES_PATH/build-host --target rpi_connect_ota_demo
alias rpi-connect-test=$PICO_EXAMPLES_PATH/build-host/rpi-connect/rpi-connect-ota-demo/rpi_connect_ota_demo
```

### Use

`rpi-connect-test --help` lists every option; most also read an
`RPI_CONNECT_*` environment variable. Add `-v` for request verbosity.

```sh
# Check a registered identity can obtain a token (prints RPI_CONNECT_TOKEN=...)
rpi-connect-test -v --device-identity-exchange --serial <serial> \
    --device-privkey device-priv-key.pem --device-pubkey device-pub-key.pem

# Run the full OTA loop with a token
RPI_CONNECT_TOKEN=<token> rpi-connect-test --ota --serial <serial>
```

The identity exchange needs the private key on the host, so it only works with a
[host-generated key](picotool.md#device-identity-key-generated-on-the-host). For
a key generated on the device, use
`picotool provision connect --identity-exchange` instead, which runs the same
exchange on the device and reports the result on its console.

If the host path works but the device does not, look at provisioning (OTP/FFS)
or connectivity rather than the API exchange.
