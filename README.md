# ESPHome EM24 Meter Emulator

An [ESPHome](https://esphome.io/) external component that makes a cheap ESP32 + IR reading head look like a **Carlo Gavazzi EM24 Ethernet energy meter** over Modbus TCP — so a **Victron Cerbo GX / Venus OS** device detects it as a native grid meter, no extra drivers or dbus hacks required.

## Why?

Victron's ESS (Energy Storage System) needs a grid meter to regulate battery charge/discharge. The officially supported Carlo Gavazzi EM24 costs €200+. If you already have a smart meter with an SML IR interface (common in Germany/Austria), an ESP32 with an IR reading head (~€15) can read the meter data and serve it to the Cerbo GX directly.

## How it works

```
┌──────────────┐   SML/IR    ┌──────────┐  Modbus TCP   ┌──────────┐
│  Smart Meter  │───────────▶│   ESP32   │◀─────────────▶│ Cerbo GX │
│  (e.g. ISKRA) │            │ (ESPHome) │   port 502    │          │
└──────────────┘             └──────────┘               └──────────┘
```

The ESP32 reads power/energy values via SML over UART, then exposes them as Modbus TCP holding registers that exactly match the EM24's register map. The Cerbo GX auto-detects it (or you add it manually) as a Carlo Gavazzi meter.

## Hardware

- **ESP32** (any variant — ESP32, ESP32-C3, ESP32-S3, etc.)
- **IR reading head** with UART/TTL output (e.g. Hichi or Bitshake SmartMeterReader)
- Your smart meter must support **SML** output via the IR interface

## Quick Start

### 1. Clone or reference this repo

**Option A — as a git reference (recommended):**

```yaml
external_components:
  - source:
      type: git
      url: https://github.com/Fexiven/esphome-em24-emulator
      ref: main
    components: [em24_meter]
```

**Option B — local copy:**

Copy the `components/em24_meter/` folder into your ESPHome config directory.

```yaml
external_components:
  - source:
      type: local
      path: components
    components: [em24_meter]
```

### 2. Configure your YAML

See [`example.yaml`](example.yaml) for a full working config. The key part:

```yaml
em24_meter:
  power: sml_power
  import_energy: sml_import
  export_energy: sml_export
  phases: "1P"
```

### 3. Flash and add to Victron

1. Flash the ESP32 with `esphome run your_config.yaml`
2. On the Cerbo GX go to **Settings → Modbus TCP/UDP devices**
3. Add the ESP32's IP address, port 502, unit ID 1
4. It should appear as a Carlo Gavazzi EM24 grid meter

> **Tip:** Many Cerbo GX firmware versions also auto-detect Modbus TCP meters on the local network. If auto-scan is enabled the meter may just show up.

## Component Configuration

| Option | Required | Default | Description |
|---|---|---|---|
| `power` | **yes** | — | Sensor ID for grid power in **W** (positive = import, negative = export) |
| `import_energy` | **yes** | — | Sensor ID for total imported energy in **kWh** |
| `export_energy` | **yes** | — | Sensor ID for total exported energy in **kWh** |
| `voltage` | no | 230 V | Sensor ID for grid voltage (used to synthesize current) |
| `frequency` | no | 50 Hz | Sensor ID for grid frequency |
| `port` | no | `502` | Modbus TCP listen port |
| `unit_id` | no | `1` | Modbus unit/slave ID |
| `phases` | no | `"1P"` | Phase configuration: `"1P"`, `"3P.n"`, or `"3P"` |
| `serial` | no | `"EM24EMU0000001"` | Serial number reported to GX (max 14 chars) |
| `invert_power` | no | `false` | Flip the sign of the power value |

## Adapting to other meter types

This component is not limited to SML meters. Any ESPHome sensor can be wired to the `power`, `import_energy`, and `export_energy` inputs — for example:

- **Modbus RTU meters** (SDM120, SDM630, etc.) via `modbus_controller`
- **Pulse counters** with energy integration
- **HTTP/MQTT sensors** from other devices
- **Template sensors** that combine multiple sources

As long as you provide power in W and energy in kWh, the EM24 emulator handles the rest.

## Troubleshooting

**Cerbo GX doesn't detect the meter:**
- Check that the ESP32 is on the same network/VLAN as the Cerbo
- Verify port 502 is reachable: `nc -zv <ESP_IP> 502`
- Check the ESPHome logs for "EM24 Modbus TCP server listening on port 502"
- Try adding the meter manually on the GX instead of relying on auto-detection

**Power values are inverted in Victron:**
- Set `invert_power: true` in the config

**Energy values don't update:**
- SML meters often report energy in Wh with a scale factor — make sure your sensor filters produce **kWh**
- Check that `state_class: total_increasing` is set on your energy sensors

## Technical details

The emulator implements the subset of the Carlo Gavazzi EM24 Modbus register map that Victron's `dbus-modbus-client` ([`carlo_gavazzi.py`](https://github.com/victronenergy/dbus-modbus-client)) probes during detection and ongoing polling:

- Model ID register `0x000B` → returns `1651` (EM24DINAV53XE1X)
- Application register `0xA000` → returns `7` ("H" application type)
- System power `0x0028-0x0029` (s32, ×10)
- Forward energy `0x0034-0x0035` (s32, ×10)
- Reverse energy `0x004E-0x004F` (s32, ×10)
- Frequency `0x0033` (u16, ×10)
- L1 voltage/current/power/energy phase block
- Phase config `0x1002`, serial `0x5000-0x5006`

Supports function codes `0x03` (Read Holding Registers), `0x04` (Read Input Registers), `0x06` (Write Single Register), and `0x10` (Write Multiple Registers).

## License

MIT — see [LICENSE](LICENSE).
