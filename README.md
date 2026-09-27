# ESPHome LD2450 radar on ESP32-S3

An ESPHome configuration for the Hi-Link HLK-LD2450 24 GHz radar. This project covers **only the LD2450 radar**. It does not modify, extend, improve, or redistribute [ESPectre](https://github.com/francescopace/espectre).

The Kitchen prototype happens to run this radar on the same ESP32-S3 as [Francesco Pace's ESPectre Wi-Fi CSI project](https://github.com/francescopace/espectre). They are two independent sensing systems sharing a device. ESPectre provides Wi-Fi CSI motion sensing; the LD2450 provides mmWave target detection and tracking. The radar package can run on a device without ESPectre.

**Status:** This package transcribes the LD2450 portion of a working Home Assistant ESPHome configuration. It has not yet been compiled or flashed from this repository. Review your pins and run ESPHome **Validate** before installing. There is no firmware binary here.


## Standard device architecture

This repository now separates **immutable hardware identity** from **deployment identity** so a physical sensor can move between rooms, sites, or roles without losing its underlying identity.

### Immutable device identity

- `device_id`: last six hexadecimal characters of the ESP32 Wi-Fi MAC address (for example, MAC `44:1B:F6:81:F3:24` becomes `81F324`).
- The full ESP32 MAC is exposed as a diagnostic entity so the identifier can be audited against the hardware.
- Product family, hardware model, ESPHome version, IP address, SSID/BSSID, Wi-Fi signal, uptime, and controller status are exposed as diagnostics.
- Do not encode a room number, customer, site, or other deployment-specific information into the immutable device identifier.

Deployment information such as site, building, floor, room, area, role, and customer asset number belongs in Home Assistant or a future fleet-management layer. Moving a sensor from Room 324 to Room 310 should change its deployment metadata, not its hardware identity.

### Shared status LED standard

The reusable `packages/device_status.yaml` controller owns status indication and applies one priority order so competing events cannot overwrite a more important state.

| Priority | State | LED |
| ---: | --- | --- |
| 1 | Critical hardware fault | Red, 100%, fast flash |
| 2 | Firmware updating | Yellow, 100%, solid |
| 3 | Connectivity/recoverable error | Orange, slow pulse |
| 4 | Booting | Blue, 100%, solid |
| 5 | Presence/active | Green, 10%, solid |
| 6 | Idle/no presence | White, 3%, solid |

The controller also publishes a `Device Status` diagnostic text sensor. API disconnects use a grace period before declaring a connection error so brief client reconnects do not immediately present as a fault. Critical-fault scripts are provided for component packages to call only when they have evidence of a genuine local hardware failure.

### Package layering

- `packages/device_status.yaml`: reusable ESP32 identity, diagnostics, lifecycle state, status LED, API connection handling, OTA indication, and fault-state controller.
- `packages/ld2450.yaml`: LD2450 UART, radar entities, target tracking, zones, controls, and radar-specific behavior.
- Device YAML: hardware identity substitutions, ESP32 board definition, Wi-Fi, shared API encryption secret, logger, and provisioning.

This layering is intended to keep residential builds simple while avoiding assumptions that would make later fleet or commercial deployments difficult to support.
\n## Hardware

| Part | Purpose |
| --- | --- |
| ESP32-S3 development board | ESPHome host, with a 5 V supply rail for the radar |
| Hi-Link HLK-LD2450 | Three-target 24 GHz mmWave tracking module |
| Jumper wires and suitable power supply | UART, 5 V, and common ground |

### Wiring used on the Kitchen prototype

| ESP32-S3 | LD2450 |
| --- | --- |
| GPIO14 (TX) | RX |
| GPIO13 (RX) | TX |
| 5 V | VCC |
| GND | GND |

TX and RX cross. Confirm your particular ESP32-S3 board exposes these pins and can supply the radar appropriately. The UART is configured at **256000 baud**. Changing the LD2450 baud-rate control without changing ESPHome's UART setting will break communication.

## Quick start in the ESPHome GUI

1. Open a new ESPHome device's YAML in Home Assistant. Keep your own `esphome`, `esp32`, Wi-Fi, `api`, logger, and provisioning configuration. OTA lifecycle handling is supplied by the shared device-status package.
2. Add this top-level block. If your device already has `packages:`, add the `ld2450_radar` entry inside it.

   ```yaml
   packages:
     device_status: github://fhugh501/esphome-ld2450-radar/packages/device_status.yaml@main
     ld2450_radar: github://fhugh501/esphome-ld2450-radar/packages/ld2450.yaml@main
   ```

3. Remove existing LD2450 `uart`, `ld2450`, and LD2450 platform entries from the **same** device YAML before including the package; duplicate entries would create duplicate entities or conflicting UART definitions. Leave any unrelated UARTs, ESPectre CSI settings, LED behavior, and ESP32 diagnostics alone.
4. Use **Validate**, then install to the ESP32-S3 when ready. An independent example is at [`examples/esp32-s3-ld2450.yaml`](examples/esp32-s3-ld2450.yaml).

The package is pinned to `main` above for easy updates. To freeze a known version, replace `main` with a commit SHA. Your actual Wi-Fi credentials and Home Assistant API encryption key belong in ESPHome `secrets.yaml`, never in a public repository.

## Entities and units

- Presence, moving target, and still target binary sensors use the ESPHome LD2450 component defaults. No manual icons are set, so Home Assistant can render the component's native classes and icons.
- Up to three targets: X/Y and resolution in inches, distance in feet, speed in ft/s, plus angle and direction. The raw module reports millimeters and mm/s. These are **display** conversions; writable zone coordinates remain in millimeters.
- Total and per-zone counts, presence timeout, Bluetooth and multi-target switches, zone mode, firmware details, and radar restart are exposed.
- Target detail sensors retain the Kitchen prototype's `timeout: 1s` and `throttle_with_priority: 1000ms` filters. Inspect the underlying ESPHome LD2450 documentation when changing them.
- LD2450 zone mode should stay **Disabled** while mapping the room. `Detection` counts targets inside zones; `Filter` excludes targets inside zones.

The package keeps the Kitchen prototype's entity names. Home Assistant entity IDs may vary with your ESPHome device name and entity registry.

## Parts links

The [Hi-Link AliExpress catalog shared for this build](https://a.aliexpress.com/_mNoXb4J) shows multiple products, including the HLK-LD2450/LD2460. **Check the selected product and variant before ordering.** This link has not been verified as an affiliate link. Affiliate links, if added later, will be labeled as such here.

## Credits and project boundaries

- **ESPectre:** Created and maintained by [Francesco Pace and its contributors](https://github.com/francescopace/espectre). See the [ESPectre documentation](https://github.com/francescopace/espectre#documentation), [ESPHome frontend guide](https://github.com/francescopace/espectre/blob/main/src/cpp/frontend/esphome/README.md), and [licensing information](https://github.com/francescopace/espectre/blob/main/LICENSING.md) for their software. ESPectre handles Wi-Fi CSI motion sensing. This radar repository has no affiliation with or endorsement from the ESPectre maintainers.
- **ESPHome:** The radar YAML uses ESPHome's built-in [`ld2450` component](https://esphome.io/components/sensor/ld2450/). The LD2450 hardware and firmware are made by [Hi-Link](https://www.hlktech.net/index.php?id=1157).
- **This repository:** It supplies an ESPHome package, wiring notes, and display-unit choices for the LD2450. It contains no ESPectre code, firmware, models, or modified ESPectre component. The Kitchen ESPectre configuration also runs BLE telemetry and an onboard LED; neither is needed for this radar package. No credentials or complete household device configuration are included.

No license has been selected yet. Public access to a repository by itself does not grant a general license to reuse its contents.
