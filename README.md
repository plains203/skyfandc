# skyfandc

Shared ESPHome package files for SkyFan DC ceiling fans (ESP32-C6 + Tuya
MCU), with or without the integrated light. Devices pull these via
ESPHome's `packages:` git support, so the whole fleet is updated from one
place - a device's local YAML carries only its identity (name, area, IP).

## Layout

| File | Contents | Needed on |
|---|---|---|
| `common_core.yaml` | ESPHome/ESP32 core, API, OTA, logger, I2C, Improv | all devices |
| `common_network.yaml` | WiFi, static IP, BLE tracker, Bermuda/bluetooth_proxy | all devices |
| `common_tuya_bus.yaml` | UART + Tuya MCU link | all devices |
| `common_diagnostics.yaml` | WiFi signal/strength, CPU temperature, uptime | all devices |
| `tuya_fan.yaml` | 6-speed fan entity + mode/timer | all devices |
| `sensor_power_fan.yaml` | Estimated fan power draw from datasheet tables (needs `skyfan_model`) | all devices |
| `tuya_light.yaml` | Light entity with brightness + colour temperature (3000/4000/5000 K) | fans with a light only |
| `sensor_power_light.yaml` | Estimated light power draw (untested/incomplete) | fans with a light only |

`sensor_power_light.yaml` references the light entity, so it must always
be included/omitted **together with** `tuya_light.yaml`. Omitting one
without the other fails the build (loudly, on purpose).

## Entity naming

HA prefixes each entity with the device's `friendly_name`, so entity
names in the package files are deliberately generic:

| Entity | Name in package | Appears in HA as (device "Main Bedroom Fan") |
|---|---|---|
| Fan | `None` (takes device name) | Main Bedroom Fan (`fan.main_bedroom_fan`) |
| Light | Light | Main Bedroom Fan Light |
| Fan power | Fan Power | Main Bedroom Fan Fan Power |
| Light power | Light Power | Main Bedroom Fan Light Power |
| Mode / Timer | Fan Mode / Fan Timer | ... (Configuration section) |
| Diagnostics | WiFi Signal, WiFi Strength, CPU Temperature, Uptime | ... (Diagnostic section) |

The dp-level helpers (switch dp1/dp15, speed dp3, dimmer dp16, direction
dp8, colour temp dp19) are `internal` and never appear in HA.

Per-device name overrides don't need substitutions: packages merge
components by `id`, so a device's local YAML can override e.g. the light
name with:

```yaml
light:
  - id: !extend skyfan_light
    name: Bedside Light
```

## Adding a device

### Fan with light

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
  device_name: "main-bedroom-fan"
  device_friendly_name: "Main Bedroom Fan"
  area_name: "Main Bedroom"
  ip_address: "10.1.0.148"
  skyfan_model: "SKY1203"
  wifi_ssid: !secret wifi_ssid
  wifi_password: !secret wifi_password
```

### Fan without light

Same, minus the two light files:

```yaml
    files:
      - common_core.yaml
      - common_network.yaml
      - common_tuya_bus.yaml
      - common_diagnostics.yaml
      - tuya_fan.yaml
      - sensor_power_fan.yaml
```

`wifi_ssid` / `wifi_password` must be set in the device's local YAML via
`!secret` - ESPHome doesn't allow secret lookups inside remote packages.

## Updating the fleet

Changes committed here are picked up by every device on its next compile
(subject to the `refresh: 1d` cache; set `refresh: 0s` on one device while
actively iterating, but don't leave it there - the dashboard re-validates
constantly and will hammer the repo). `ref: main` pins the branch; switch
to a tag once stable so a bad commit can't reach the whole fleet.

## Behaviour notes

**MCU is the source of truth.** Both the fan and light use a guard flag
that starts true at boot: the entity's initial output write is swallowed,
and the MCU's datapoint dump seeds the real state via a sync script
(which clears the guard). An ESPHome reboot or OTA update therefore never
turns off a running fan or light - HA just re-learns the hardware state.
The same guard suppresses echo loops from wall/RF remote changes.

**Colour temperature.** dp19 is an enum (not an int), which the native
tuya light platform can't write (esphome/issues#3476), so the light is
built from the `color_temperature` platform with template outputs. HA's
continuous CT slider quantises to the hardware's three steps
(3000/4000/5000 K) and snaps to the confirmed value. Set each light's
favourite colours in HA to exactly 3000/4000/5000 K (long-press a swatch
in the more-info dialog to edit) so the presets land on real steps.

## Entity renames (one-time migration per device)

The generic names above replaced the old per-device names ("skyfan",
"ceilinglight", "C6 WiFi Signal", ...). Renaming changes each entity's
unique_id, so on the first flash with these packages HA creates new
entities and orphans the old ones. Per device, once:

1. Flash, then open the device page in HA and confirm the new entities.
2. Settings > Devices & Services > Entities: filter by the device,
   delete the unavailable (orphaned) old entities - including the old
   "... Color Temperature" select, which is now internal.
3. Repoint dashboards, automations, Node-RED flows, and InfluxDB
   queries from the old entity_ids to the new ones. Long-term
   statistics do not carry across entity_ids.
