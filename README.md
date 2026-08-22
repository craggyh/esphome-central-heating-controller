# ESPHome Central Heating Controller

ESPHome configuration for a whole-house central heating controller coordinating an oil boiler, a solid-fuel stove/boiler, four heating/hot-water zones, circulation pumps, temperature monitoring and Home Assistant control.

The controller is built around a KinCony A8 V3 / ESP32-S3 and keeps the real-time heating logic local on the controller rather than depending on Home Assistant.

## Hardware

- KinCony A8 V3 / ESP32-S3
- ESP-IDF framework
- 16 MB flash with octal PSRAM
- W5500 Ethernet
- PCF8575 I/O expander at `0x22`
- eight relay outputs
- DS18B20 temperature probes using ESPHome native `one_wire` / `dallas_temp`
- thermostat/call-for-heat inputs
- motorised-valve end-switch feedback inputs
- Home Assistant API and ESPHome OTA

## Relay assignments

| Output | Function |
| --- | --- |
| Relay 1 | Zone 1 / downstairs valve |
| Relay 2 | Zone 2 / upstairs valve |
| Relay 3 | Zone 3 / top-floor valve |
| Relay 4 | Domestic hot-water valve |
| Relay 5 | Oil boiler burner enable |
| Relay 6 | Stove boiler circulation pump |
| Relay 7 | Oil boiler circulation pump |
| Relay 8 | Spare / auxiliary |

The four logical zone switches set demand flags. The central control loop decides which physical relays, pumps and heat source may operate.

## Oil boiler control

When the stove is inactive, the oil system uses valve feedback and return-temperature control.

The controller:

- commands the required motorised valve(s)
- waits for at least one demanded valve to physically prove open
- only then permits the oil circulation pump and burner
- locks the burner out if the oil-return temperature sensor is invalid
- applies configurable condensing return-temperature hysteresis during space heating
- applies configurable minimum burner ON and OFF times
- gives domestic hot water thermal priority without bypassing safety interlocks

The Home Assistant controls **Condensing Return - Burner ON** and **Condensing Return - Burner OFF** are efficiency controls for space heating, not boiler safety limits. Domestic-hot-water demand deliberately bypasses these condensing thresholds so the cylinder can reach its configured maximum temperature.

Valve-open timeout faults and temperature-sensor faults are exposed to Home Assistant.

## Solid-fuel stove control

The stove has priority whenever its boiler stat is active. The oil burner and oil circulation pump are inhibited immediately.

Stove startup is **valve first, pump second**:

1. determine the currently demanded heating path, or choose a safe heat-dump path if there is no demand
2. command the required valve(s)
3. wait for a commanded valve to physically prove open
4. start the stove circulation pump

The old 15-second all-valves-closed startup delay has been removed.

If a commanded stove heat path does not prove open before the configurable valve timeout, the controller enters a latched emergency dump mode: all zone valves are requested, a fault is exposed to Home Assistant, and the stove circulation pump is started immediately. The fault remains latched until stove mode ends.

With no normal demand, the controller prefers domestic hot water as the heat sink while the cylinder/store can accept heat; otherwise Zone 1 is used as the normal fallback dump path.

## Heating zones

The controller provides:

1. Zone 1 — downstairs heating
2. Zone 2 — upstairs heating
3. Zone 3 — top-floor heating
4. Zone 4 — domestic hot water

The three space-heating demands are restored from their live thermostat inputs after boot.

## Hot water

Domestic hot water has configurable minimum and maximum setpoints. The controller also reads the top temperature of the solar-heated water store from Home Assistant for stove heat-dump decisions.

If the hot-water temperature sensor becomes invalid, stale DHW demand is cleared rather than allowing an uncontrolled heating request to remain latched.

## Temperature monitoring

The controller uses ESPHome's native `one_wire` and `dallas_temp` components across four 1-Wire buses. DS18B20 probes monitor stove flow/return, oil-boiler flow/return, heating-zone flow/return and domestic hot water.

The native implementation was real-device tested before being promoted to `main`. The previous DallasNG compatibility fork has been removed from the production configuration.

## Repository layout

```text
central-heating.yaml                    Full hardware/entity base configuration
controller.yaml                         Canonical assembled production entry point
overrides/remove-legacy-interval.yaml   Removes the historical control interval
overrides/hardened-control-loop.yaml    Current valve-aware control loop
overrides/native-dallas-sensors.yaml    Native one_wire / dallas_temp definitions
overrides/remove-dallasng-sensors.yaml  Removes legacy DallasNG definitions from the base
overrides/condensing-control-names.yaml Clear Home Assistant names for condensing controls
examples/central-heating-local.yaml     Small Home Assistant/ESPHome loader example
.github/workflows/esphome.yml           Validation and full compile CI
```

**`controller.yaml` on `main` is the canonical configuration to build and deploy.**

`central-heating.yaml` remains the hardware/entity base. Package overlays replace its historical control loop and legacy DallasNG definitions with the hardened controller and native ESPHome temperature stack.

## Use from the ESPHome dashboard

Instead of maintaining a second full copy under `/config/esphome`, keep only a small loader such as:

```yaml
packages:
  central_heating:
    url: https://github.com/craggyh/esphome-central-heating-controller
    files:
      - controller.yaml
    ref: main
    refresh: 5min
```

The same example is stored in `examples/central-heating-local.yaml`.

## Building

Validate and compile the assembled project with:

```bash
esphome config controller.yaml
esphome compile controller.yaml
```

GitHub Actions performs both commands automatically for pull requests and pushes to `main`, currently pinned to ESPHome 2026.8.0.

## Network and integration

The controller uses W5500 Ethernet. ESPHome provides the native Home Assistant API, OTA updates, uptime reporting and ESP32 reset-reason reporting. API reboot timeout is disabled so loss of Home Assistant does not reboot the heating controller.

## Safety

This project controls pumps, motorised heating zones, an oil burner and a solid-fuel heat source. ESPHome should not be the sole safety mechanism for either appliance.

Independent boiler thermostats, over-temperature cut-outs, pressure protection, correctly designed gravity/dump protection where required, and manufacturer-specified electrical/mechanical safety controls must remain in place.

## Development workflow

Changes are made on Git branches and reviewed through pull requests. The CI workflow validates and compiles the exact assembled `controller.yaml` before changes are published to `main`.
