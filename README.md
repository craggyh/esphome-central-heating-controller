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

When the stove is inactive, the controller commands the required motorised valves and waits for at least one demanded valve to physically prove open before permitting the oil circulation pump or burner. A failed oil-return temperature sensor locks the burner out while leaving circulation available through an open zone.

Space heating uses configurable return-temperature hysteresis through **Condensing Return - Burner ON** and **Condensing Return - Burner OFF**, together with configurable minimum burner ON/OFF times. These are efficiency controls, not boiler safety limits. Domestic-hot-water demand deliberately bypasses the condensing thresholds so the cylinder can reach its configured maximum temperature while still respecting valve, sensor and burner anti-cycle interlocks.

Valve-open timeout faults and temperature-sensor faults are exposed to Home Assistant.

## Solid-fuel stove control

The stove takes priority whenever its boiler stat is active, immediately inhibiting the oil burner and oil circulation pump.

Stove startup is **valve first, pump second**: the controller selects the demanded heating path (or a safe heat-dump path), commands the valve, waits for physical open feedback, and then starts the stove circulation pump. The former 15-second all-valves-closed startup delay has been removed.

If a commanded stove heat path does not prove open before the configurable timeout, the controller enters a latched emergency dump mode: all zone valves are requested, a Home Assistant fault is raised, and the stove pump starts immediately. With no normal demand, domestic hot water is preferred as a heat sink while it can accept heat; otherwise Zone 1 is the normal fallback.

## Heating zones and hot water

The controller provides downstairs, upstairs, top-floor and domestic-hot-water zones. The three space-heating demands are restored from their live thermostat inputs after boot.

DHW has configurable minimum and maximum setpoints. The controller also reads the top temperature of the solar-heated water store from Home Assistant for stove heat-dump decisions. If the DHW sensor becomes invalid, stale DHW demand is cleared rather than being left latched.

## Temperature monitoring

Four native ESPHome 1-Wire buses use `one_wire` and `dallas_temp` to monitor stove flow/return, oil-boiler flow/return, heating-zone flow/return and domestic hot water. The native implementation was real-device tested before being promoted to production; the previous DallasNG external component and compatibility fork are no longer required.

## Repository layout

```text
central-heating.yaml                    Single authoritative controller configuration
examples/central-heating-local.yaml     ESPHome dashboard loader example
.github/workflows/esphome.yml           Validation, single-loop assertion and compile CI
```

**`central-heating.yaml` on `main` is the only configuration to build and deploy.**

## Use from the ESPHome dashboard

Keep only a small local loader rather than a second full copy:

```yaml
packages:
  central_heating:
    url: https://github.com/craggyh/esphome-central-heating-controller
    files:
      - central-heating.yaml
    ref: main
    refresh: 5min
```

The same example is stored in `examples/central-heating-local.yaml`.

## Building

```bash
esphome config central-heating.yaml
esphome compile central-heating.yaml
```

GitHub Actions runs both commands for pull requests and pushes to `main`, currently pinned to ESPHome 2026.8.0. CI also checks the resolved configuration has exactly one heating control interval..

## Network and safety

The controller uses W5500 Ethernet, Home Assistant's native ESPHome API and OTA updates. API reboot timeout is disabled so loss of Home Assistant does not reboot the heating controller.

This project controls pumps, motorised valves, an oil burner and a solid-fuel heat source. ESPHome must not be the sole safety mechanism. Independent boiler thermostats, over-temperature cut-outs, pressure protection, correctly designed gravity/dump protection where required, and manufacturer-specified safety controls must remain in place.

## Development workflow

Changes are developed on Git branches and reviewed through pull requests. CI validates and compiles the exact assembled `central-heating.yaml` before changes are published to `main`.
