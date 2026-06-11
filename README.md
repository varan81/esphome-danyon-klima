# esphome-danyon-klima

ESPHome component for **Danyon split air conditioners** via UART, based on [KG3RK3N/esphome-kaeltebringer](https://github.com/KG3RK3N/esphome-kaeltebringer).

## What's different in this fork

The original component could read swing state correctly but **never actually moved the louvers** when sending commands. The root cause: the `_mv` (motion) bits were never set in the outgoing SET command, so the device silently ignored swing requests.

### Bug fix: `hswing_mv` / `vswing_mv`

In `build_set_cmd`, two lines were missing:

```cpp
// Original (broken)
m_set_cmd.data.vswing = get_cmd_resp->data.vswing ? 0x07 : 0x00;
m_set_cmd.data.hswing = get_cmd_resp->data.hswing;

// Fixed
m_set_cmd.data.vswing    = get_cmd_resp->data.vswing ? 0x07 : 0x00;
m_set_cmd.data.vswing_mv = get_cmd_resp->data.vswing ? 0x01 : 0x00;  // ← NEW
m_set_cmd.data.hswing    = get_cmd_resp->data.hswing;
m_set_cmd.data.hswing_mv = get_cmd_resp->data.hswing ? 0x01 : 0x00;  // ← NEW
```

This was identified by comparing UART packet captures with swing ON vs OFF using VERBOSE logging.

### New methods

**`test_vswing_fix(uint8_t fix_val, uint8_t mv_val)`**  
Sets a fixed vertical louver position. Discovered by systematic testing of `vswing_fix` values 1–7. Positions 1–5 produce distinct angles on Danyon units (1 = nearly closed, 5 = fully down).

**`set_fan_mute()`**  
Sets fan to Mute mode with explicit `power=1` and `mode=FAN_ONLY`. Safe to call even when the device was previously OFF.

**`send_clean()`** *(experimental)*  
Attempts to trigger the device's built-in self-clean cycle. Based on UART packet analysis during a clean cycle initiated via the original remote. May not work on all units.

---

## Hardware

Tested on: **Danyon split air conditioner, OG hallway unit**

Connection: **diymore ESP32 NodeMCU (USB-C)** → bidirectional level shifter (3.3V ↔ 5V) → indoor unit UART

```
ESP32 GPIO26  →  TX  →  level shifter  →  AC unit RX
ESP32 GPIO27  →  RX  →  level shifter  →  AC unit TX
Baud: 9600, Parity: EVEN
```

**Parts used:**
- [ESP32 NodeMCU USB-C (diymore)](https://amzn.to/4fFoX1l)
- [Bidirectional Level Shifter 3.3V ↔ 5V](https://amzn.to/4ehPtLU)

## Hardware photos

Mainboard (indoor unit control board):
![Mainboard](imagesmainboard.jpg.png)

Level shifter module (3.3V ↔ 5V):
![Level Shifter](imageslevelshifter.jpg.png)

---

## Configuration

```yaml
external_components:
  - source:
      type: git
      url: https://github.com/varan81/esphome-danyon-klima
    components: [kaeltebringer]

uart:
  id: uart_bus
  tx_pin: GPIO26
  rx_pin: GPIO27
  baud_rate: 9600
  parity: EVEN
  rx_buffer_size: 512

climate:
  - platform: kaeltebringer
    uart_id: uart_bus
    name: "Danyon Klimaanlage"
    update_interval: 10s
    beep_enabled: false
```

---

## Supported features

| Feature | Status |
|---|---|
| COOL / HEAT / DRY / FAN_ONLY / AUTO | ✅ |
| Swing OFF / HORIZONTAL / VERTICAL / BOTH | ⚠️ fix applied, untested (remote does not support swing on test device) |
| Fixed vertical louver positions (1–5) | ✅ tested on Danyon unit |
| Fan modes: Automatic / 1–5 / Turbo / Mute | ✅ |
| Current temperature (internal sensor) | ✅ |
| Target temperature | ✅ |
| Self-clean trigger | ⚠️ experimental |

---

## Dashboard

A ready-to-use Home Assistant dashboard is included under [`examples/dashboard.yaml`](examples/dashboard.yaml).

![Dashboard](imagesdashboard.png)

### How the external temperature averaging works

This setup is designed for a **central hallway unit** that serves multiple adjacent rooms. Instead of relying on the AC's built-in sensor (which only measures air temperature at the unit itself), the system averages Bluetooth temperature sensors placed in each room:

- One **BTHome Bluetooth sensor** per room
- The ESP32 reads all sensor values via Home Assistant
- An **average temperature** is calculated from all available sensors
- The AC's target temperature is automatically corrected based on the difference between the internal sensor and the room average

This means the AC cools or heats to the actual room temperature rather than the temperature at the unit. If one sensor is unavailable, it is excluded from the average automatically.

The dashboard shows three room sensors (labeled Room 1, Room 2, Room 3 in the example – rename to match your setup). The external average is shown directly in the "Ext. Average Control" card.

---

## Credits

- Original component: [KG3RK3N/esphome-kaeltebringer](https://github.com/KG3RK3N/esphome-kaeltebringer)  
- Based on work by [lNikazzzl](https://github.com/lNikazzzl/tcl_ac_esphome)

---

<a href="https://www.buymeacoffee.com/varan81" target="_blank"><img src="https://cdn.buymeacoffee.com/buttons/v2/default-yellow.png" alt="Buy Me A Coffee" style="height: 50px !important;width: 200px !important;" ></a>
