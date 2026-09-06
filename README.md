# ESPHome TaiXia Custom Component — RD-18FC Fork

TaiSEIA (TaiXia) protocol component for [ESPHome](https://esphome.io), enabling
Home Assistant integration of Hitachi / Daikin / Panasonic appliances over UART.

This is a **fork of [tsunglung/taixia](https://github.com/tsunglung/taixia)**
maintained for the **Hitachi RD-18FC dehumidifier**, with one display-layer
change on top of upstream `master` (commit `581bd57`).

## Changes vs upstream

| Item | Upstream | This fork |
|------|----------|-----------|
| Mode/preset enum value 4 | `baby` | `air_purifier` |
| Affected files | — | `components/taixia/select/__init__.py`, `components/taixia/fan/taixia_fan.cpp`, `components/taixia/fan/__init__.py` |
| Protocol behavior | mapping = 4 | unchanged (display string only) |

On the RD-18FC, mode value `4` is the **pure air-purifier mode**. Upstream
labels it `baby`, which does not match this model. The rename only changes the
string exposed to Home Assistant; the value written to / read from the
appliance is still `4`.

Upstream will be kept in sync; rebase this single commit after pulling.

## Known RD-18FC limitations (hardware, not this component)

- Mode writes (service `0x81`) are **rejected** by the appliance with `FF FF`
  for all standard values (`normal` / `home` / `boost` / `sleep` / `eco`),
  including while powered off. Mode switching therefore works from the
  appliance panel only; the select entity reports the current mode.
- `sleep` (mode `5`) is not accepted by this model.
- Beeper switch (service `0x18`) is acknowledged but has no audible effect on
  this unit.

## Hardware

### Hitachi dehumidifier — JST PAP-04-V (4-pin)

| Pin | Signal | Notes |
|-----|--------|-------|
| 1 | 5V | Powers the ESP (via level shifter HV or directly) |
| 2 | GND | Common ground |
| 3 | RX to appliance | 5V logic — connect to ESP TX |
| 4 | TX from appliance | 5V logic — connect to ESP RX |

> The appliance UART is 5 V logic. The ESP32-C3 is **not** 5 V tolerant on all
> pins. On the RD-18FC a direct connection (optionally with a 1 kΩ series
> resistor on the appliance TX line) was tested working; a TXS0108E level
> shifter caused line oscillation and random RX noise and is **not
> recommended** for this appliance. A BSS138-type MOSFET shifter is a safe
> alternative.

### Panasonic

Pinout undocumented here; refer to upstream.

## Installation

1. Set `wifi_ssid` and `wifi_password` in your ESPHome `secrets.yaml`.
2. Point `external_components` at this fork:

```yaml
external_components:
  - source: github://Nyquist1992/taixia@master
    components: [ taixia ]
```

3. Build your device YAML (examples below or in upstream).

## Configuration Example (air conditioner)

```yaml
logger:
  baud_rate: 0  # Disable UART logger if using UART0 (pins 1,3)

external_components:
  - source: github://Nyquist1992/taixia@master
    components: [ taixia ]

uart:
  id: uart_taixia
  tx_pin: GPIO0
  rx_pin: GPIO1
  baud_rate: 9600
  debug:
    direction: BOTH

taixia:
#  max_length: 28   # Option to limit the max length of RX buffer

climate:
  - platform: taixia
    name: My Daikin Climate
    supported_modes:
      - COOL
      - HEAT
      - DRY
      - FAN_ONLY
    supported_fan_modes:
      - LOW
      - MEDIUM
      - HIGH
      - AUTO
    supported_presets:
      - ECO
      - BOOST
      - AWAY
      - SLEEP

switch:
  - platform: taixia
    type: airconditioner
    power:
      name: "${upper_devicename} Power Switch"

sensor:
  - platform: taixia
    type: airconditioner
    temperature_indoor:
      name: My Daikin Inside Temperature
    humidity_indoor:
      name: My Daikin Outside Temperature

number:
  - platform: taixia
    type: airconditioner
    vertical_fan_speed:
      name: "${upper_devicename} Vertical Fan Speed"
    horizontal_fan_speed:
      name: "${upper_devicename} Horizontal Fan Speed"

select:
  - platform: taixia
    type: airconditioner
    fuzzy_mode:
      name: "${upper_devicename} Fuzzy Mode"

text_sensor:
  - platform: taixia
    sa_id:
      name: "${upper_devicename} SA ID"
    brand:
      name: "${upper_devicename} SA Brand"
    model:
      name: "${upper_devicename} SA Model"
    version:
      name: "${upper_devicename} SA Version"
    services:
      name: "${upper_devicename} SA Services"
```

The climate example exported to Home Assistant:

![climate](https://github.com/tsunglung/taixia/raw/master/pictures/climate.png)

## Dehumidifier (RD-18FC) notes

The appliance reports its supported TaiSEIA services on `Get Info` (frame
`0x07`). RD-18FC exposes: status, mode (`4`), operating time, relative
humidity, humidity indoor, water tank full, filter notify, fan level
(`0x0E`), sound mode, defrost, error code, beeper (`0x98`), energy
consumption (`0x9D`), light level (`0xA7`).

Declare only the entities above that your model actually reports; unsupported
entities stay `unknown`.

## Credits

All credit for the TaiSEIA protocol reverse engineering and the component
itself belongs to [tsunglung](https://github.com/tsunglung) and the
contributors listed upstream.

## License

Same as upstream (MIT).
