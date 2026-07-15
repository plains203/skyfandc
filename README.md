# skyfandc

Shared ESPHome package files for SkyFanDC ceiling fan/light fixtures
(ESP32-C6 + Tuya MCU), designed to be pulled in via ESPHome's `packages:`
git support so a fleet of these devices can be updated from one place.

## Layout

| File | Contents | Varies per device? |
|---|---|---|
| `common_core.yaml` | ESPHome/ESP32 core, API, OTA, logger, I2C, Improv | Only via substitutions (name/friendly_name/area) |
| `common_network.yaml` | WiFi, static IP, BLE tracker, Bermuda/bluetooth_proxy | Only via substitutions (IP, secrets) |
| `common_tuya_bus.yaml` | UART + Tuya MCU link | No |
| `common_diagnostics.yaml` | Wi-Fi signal/strength, CPU temp, uptime | No |
| `tuya_fan.yaml` | Fan entity + Tuya datapoint logic (mode/timer/speed) | Only `fan_name`/`skyfan_model` |
| `tuya_light.yaml` | Light entity + colour temperature select | Only `light_name` |
| `sensor_power_fan.yaml` | Estimated fan power draw from datasheet tables | Only `skyfan_model` |
| `sensor_power_light.yaml` | Estimated light power draw (untested/incomplete) | No |

## Adding a new device

A device's local YAML only needs to reference the package files it wants
and supply substitutions that are genuinely unique to that device. See
`avas-fan.yaml` in this repo's parent conversation for a full example -
in short:

```yaml
packages:
  remote_package_files:
    url: https://github.com/plains203/skyfandc
    ref: main
    refresh: 1d
    files:
      - common_core.yaml
      - common_network.yaml
      - common_tuya_bus.yaml
      - common_diagnostics.yaml
      - tuya_fan.yaml
      - tuya_light.yaml
      - sensor_power_fan.yaml
      - sensor_power_light.yaml

substitutions:
  device_name: "avas-fan"
  device_friendly_name: "Avas Fan"
  area_name: "Games Room"
  ip_address: "10.1.0.148"
  skyfan_model: "SKY1203"
  wifi_ssid: !secret wifi_ssid
  wifi_password: !secret wifi_password
```

`wifi_ssid` / `wifi_password` have to be set locally (not in this repo) -
ESPHome doesn't allow `!secret` lookups inside remote packages, only in the
local file that references them. See `secrets.yaml.example`.

## Updating the fleet

Because every device pulls these files fresh (subject to the `refresh:
1d` cache), changing something here - a diagnostic sensor, the BLE scan
interval, the fallback AP password - and recompiling any device picks up
the change automatically. Set `refresh: 0s` temporarily on a device while
actively iterating on these files if you don't want to wait for the cache
(don't leave it at `0s` long-term - ESPHome re-checks the repo on every
config validation, which slows down the dashboard/VS Code extension).

`ref: main` pins to the branch explicitly - worth switching to a tag once
this is stable, so a bad commit here can't brick the whole fleet on next
compile.

## A note on entity names

`fan_name`, `light_name`, and the diagnostic sensor names in
`common_diagnostics.yaml` are deliberately kept as their current values
("skyfan", "ceilinglight", "C6 WiFi Signal", etc.) rather than renamed to
something cleaner, because changing an ESPHome entity's `name:` changes
its Home Assistant unique_id - HA registers a new entity and orphans the
old one. If you want nicer names (e.g. "Fan"/"Light", "Wi-Fi Signal"
without the device-specific prefix - redundant since HA already prefixes
the device's friendly_name), that's a one-time, one-device-at-a-time
migration: update the name, recompile, then in HA go to Settings >
Devices & Services > Entities, delete the now-unavailable old entity, and
repoint any dashboards/Node-RED flows/InfluxDB queries that referenced
the old entity_id.
