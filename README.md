# ESPHome Central Heating Controller

ESPHome configuration for a whole-house central heating controller that coordinates an oil boiler, a solid-fuel stove/boiler, four heating/hot-water zones, circulation pumps, temperature monitoring, and Home Assistant control.

The controller is built around a KinCony A8 V3 / ESP32-S3 platform and is designed to keep the core heating logic local on the ESP32 rather than depending on Home Assistant for real-time control.

## Hardware

- KinCony A8 V3 / ESP32-S3
- ESP-IDF framework
- 16 MB flash with octal PSRAM
- W5500 Ethernet
- PCF8575 I/O expander at `0x22`
- Eight relay outputs
- Multiple DS18B20 temperature probes using the `dallasng` external component
- External thermostat/call-for-heat inputs
- Home Assistant API and ESPHome OTA updates

## Controlled outputs

The eight physical relay outputs are currently assigned as follows:

| Output | Function |
| --- | --- |
| Relay 1 | Zone 1 valve / downstairs heating |
| Relay 2 | Zone 2 valve / upstairs heating |
| Relay 3 | Zone 3 valve / top-floor heating |
| Relay 4 | Hot-water zone valve |
| Relay 5 | Oil boiler burner enable |
| Relay 6 | Stove boiler circulation pump |
| Relay 7 | Oil boiler circulation pump |
| Relay 8 | Spare / auxiliary output |

ESPHome also exposes logical Zone 1-4 switches. These set internal demand flags; the central control loop decides which physical relays, pumps, and heat source should actually operate.

## Heat sources

### Oil boiler

When the stove is inactive and there is heating demand, the controller enables the oil-boiler circulation pump and manages the burner locally.

Burner cycling is controlled using the oil-boiler return temperature, with configurable Home Assistant entities for:

- Oil Return Protect OFF Threshold
- Oil Return Protect ON Threshold
- Burner Minimum ON Time
- Burner Minimum OFF Time

The separate ON and OFF thresholds provide hysteresis, while the minimum run and rest periods reduce short cycling.

Hot-water demand receives priority while the cylinder is below its configured maximum temperature.

### Solid-fuel stove / boiler

When the stove boiler becomes active, the controller automatically takes priority away from the oil system:

- oil burner is inhibited
- oil-boiler pump is stopped
- stove circulation pump is forced on
- active heating-zone demand is served from the stove

A short startup period keeps the zone valves closed initially to reduce hydraulic noise / knocking while stove circulation establishes.

Because a solid-fuel appliance cannot simply be switched off when heat is no longer required, the controller always provides a heat-dump path. If there is no active zone demand it will preferentially dump heat into hot water while the cylinder/store can accept it, otherwise Zone 1 is opened as the fallback heat sink.

## Heating zones

The configuration currently provides four logical demand zones:

1. Zone 1 - downstairs heating
2. Zone 2 - upstairs heating
3. Zone 3 - top-floor heating
4. Zone 4 - domestic hot water

The three space-heating zones are restored from their live thermostat inputs after boot, so a controller restart does not leave the software demand state out of sync with the thermostats.

## Hot water

Domestic hot water has configurable minimum and maximum setpoints exposed through Home Assistant.

The controller also reads an external Home Assistant temperature sensor for the top of the solar-heated water store. That value is used by the stove heat-dump logic together with the cylinder temperature to decide whether hot water can safely accept additional heat.

## Temperature monitoring

Several DS18B20 buses are used to monitor the heating system, including temperatures associated with:

- stove boiler flow / return
- oil boiler flow / return
- heating-zone flow / return
- domestic hot water

The configuration uses the external [`nrandell/dallasng`](https://github.com/nrandell/dallasng) ESPHome component.

## Main control loop

The primary heating arbitration runs locally once per second.

At a high level it:

1. reads the four demand flags
2. determines whether the stove boiler is active
3. selects the stove or oil operating path
4. controls zone valves and circulation pumps
5. applies hot-water priority
6. applies oil return-temperature protection and burner anti-short-cycle timing
7. publishes the overall call-for-heat state

This means normal heating operation continues even if Home Assistant is temporarily unavailable.

## Network and integrations

The current controller uses W5500 Ethernet rather than Wi-Fi.

ESPHome provides:

- native Home Assistant API
- OTA firmware updates
- uptime reporting
- ESP32 reset-reason reporting

The API reboot timeout is disabled so loss of Home Assistant connectivity does not reboot the heating controller.

## Repository layout

```text
central-heating.yaml   Main ESPHome configuration
README.md              Project documentation
```

`central-heating.yaml` on the `main` branch should be treated as the current authoritative configuration.

## Building and flashing

A normal ESPHome installation can validate and compile the configuration, for example:

```bash
esphome config central-heating.yaml
esphome compile central-heating.yaml
```

OTA flashing can then be performed through ESPHome once the controller is reachable on the network.

## Important safety note

This project controls pumps, motorised heating zones, an oil burner, and a solid-fuel heat source. The ESPHome logic should not be used as the sole safety mechanism for a boiler or solid-fuel appliance.

Independent mechanical/electrical protections such as boiler thermostats, over-temperature cut-outs, pressure protection, correctly designed gravity or dump circuits where required, and manufacturer-specified safety controls should remain in place.

## Development workflow

Changes to the controller are developed through Git branches and pull requests before being merged into `main`. Functional changes should be reviewed carefully against the live hydraulic and electrical behaviour of the heating system before deployment.
