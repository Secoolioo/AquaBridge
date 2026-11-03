# 5 · Home Assistant

[← 4 · Flashing](04-flashing.md) · next: [6 · Troubleshooting →](06-troubleshooting.md)

## 5.1 Add the device

1. **Settings → Devices & services.** A card *"Discovered: Water Meter (ESPHome)"* appears within a few
   minutes. Click **Configure**.
2. Not discovered? **Add integration → ESPHome**, host = the IP of the receiver, port `6053`.
3. Enter the **encryption key** (`api_encryption_key` from your `secrets.yaml`).

You get one device *Water Meter* with these entities:

| Entity | Unit | Notes |
|---|---|---|
| `sensor.water_meter_total` | m³ | the register, `total_increasing`, use this for everything |
| `sensor.water_meter_total_at_billing_date` | m³ | reading at the last billing date (Hydrus: 31.12.) |
| `sensor.water_meter_usage_since_billing_date` | m³ | total minus billing date value |
| `sensor.water_meter_water_temperature` | °C | water temperature inside the meter |
| `sensor.water_meter_battery_life` | a | remaining battery years |
| `sensor.water_meter_meter_signal` | dBm | RSSI of the last telegram (diagnostic) |
| `sensor.water_meter_meter_status` | – | e.g. `UNKNOWN_80`, vendor bits, not an error |
| `sensor.water_meter_last_telegram` | timestamp | when the last telegram was decoded |
| `binary_sensor.water_meter_meter_reception` | – | off after 10 min without a telegram |
| `binary_sensor.water_meter_status` | – | ESP32 connected to HA |
| `sensor.water_meter_wifi_signal`, `_uptime`, `_ip_address`, `_firmware` | | diagnostics |
| `button.water_meter_restart`, `_restart_safe_mode` | | |

Entity IDs follow `friendly_name` + sensor name. If you rename the device, adjust the helpers below.

## 5.2 Energy dashboard

**Settings → Dashboards → Energy → Water consumption → Add water source**

- Water usage: `sensor.water_meter_total`
- Flow rate (optional): `sensor.water_flow` from the helpers below
- Cost: a fixed price per m³ (water + waste water) if you like

Statistics start from the moment you add the sensor. "Yesterday", "last week" etc. fill up over time.

## 5.3 Helpers

You can click them together in the UI or drop in the YAML package. Both give the same result.

### Option A: UI

**Settings → Devices & services → Helpers → Create helper**

| Helper | Type | Settings |
|---|---|---|
| Water today | Utility meter | input `sensor.water_meter_total`, cycle *daily*, **Periodically resetting: off** |
| Water this month | Utility meter | cycle *monthly*, periodically resetting off |
| Water this year | Utility meter | cycle *yearly*, periodically resetting off |
| Water flow | Derivative | input `sensor.water_meter_total`, time unit *h*, time window *10 min*, max sub-interval *1 min*, round 3 |
| Water flow hourly average | Derivative | same, time window *1 h*, max sub-interval *2 min*, round 4 |
| Water running | Threshold | input `sensor.water_flow`, upper *0.012*, hysteresis *0.006* |
| Water used without a break | Threshold | input `sensor.water_flow_hourly_average`, upper *0.0006*, hysteresis *0.0003* |

*Periodically resetting: off* matters: the meter register never resets, and with this setting no water
is lost when the receiver was offline for a while.

**Nice units:** open a helper → ⚙️ → *Unit of measurement*. Water today in **L**, flow in **L/min**.
Home Assistant converts the value, the statistics keep the original unit.

> [!NOTE]
> The derivative helpers and the automations below use **m³/h**. If you switch the flow display
> to L/min, the threshold helper sees L/min too, so adjust its numbers (0.3 / 0.1 L/min) accordingly.

### Option B: YAML package

```yaml
# configuration.yaml
homeassistant:
  packages: !include_dir_named packages
```

Copy [`homeassistant/packages/aquabridge.yaml`](../homeassistant/packages/aquabridge.yaml) to
`/config/packages/aquabridge.yaml`, check the configuration and restart.

### Why derive the flow?

The Hydrus does not transmit the current flow, only the register (1 litre steps) every ~20 seconds.
The derivative over 10 minutes turns that into a smooth flow value. It reacts to anything above
roughly 6 litres per hour. A dripping tap is below that, which is why the night check (5.4) looks at
"was any water used in each hour" instead.

## 5.4 Automations

Both files paste straight into **Settings → Automations → Create automation → ⋮ → Edit in YAML**.
Replace `notify.mobile_app_your_phone` with your phone (Settings → Companion app shows the name).

### Leak and burst pipe alarm

[`homeassistant/automations/leak_alarm.yaml`](../homeassistant/automations/leak_alarm.yaml)

| Trigger | Condition | Push |
|---|---|---|
| Burst pipe | flow above 2 m³/h (33 L/min) for 10 min | 🚨 critical |
| Running | *Water running* on for 2 h | 💧 time-sensitive |
| Night | at 04:00, *used without a break* on for 3 h (water in every hour since 01:00) | 💧 time-sensitive |
| Frost | water temperature below 3 °C for 30 min | 🥶 critical |

Critical pushes on an iPhone ring even in silent mode and focus:

```yaml
data:
  push:
    sound:
      name: default
      critical: 1
      volume: 1
```

On Android use instead:

```yaml
data:
  ttl: 0
  priority: high
  channel: alarm_stream
```

Filling a pool or a hot tub takes hours. Switch the alarm off from the dashboard for that evening.

### Reception watchdog

[`homeassistant/automations/reception_watchdog.yaml`](../homeassistant/automations/reception_watchdog.yaml)

- *Meter reception* off for 5 min (= 15 min without a telegram) → push
- ESP32 offline for 15 min → push
- reception back → push

> [!NOTE]
> When an ESPHome device disconnects, its `Status` binary sensor goes to **off**, not to
> *unavailable*. The watchdog triggers on both.

## 5.5 Dashboard

[`homeassistant/dashboard/water-dashboard.yaml`](../homeassistant/dashboard/water-dashboard.yaml) is a
complete dashboard with three views, built only from standard cards (no HACS needed):

- **Overview:** status line, meter reading with 7-day trend, flow gauge, water temperature,
  today / month / year, billing date, yesterday / this week / last week / last month, 14-day chart,
  leak protection with on/off switches.
- **Statistics:** consumption per hour (48 h), per day (30 days), per week, per month, year over year,
  flow and temperature history, Energy dashboard water graph.
- **Device:** online, radio reception, last telegram, meter signal, WiFi, IP, uptime, firmware, restart,
  battery life, meter status.

**Settings → Dashboards → Add dashboard → New dashboard from scratch**, open it,
**⋮ → Edit dashboard → ⋮ → Raw configuration editor**, paste the file, save.

## 5.6 Template examples

Litres used since midnight as text:

```jinja
{{ (states('sensor.water_today') | float(0) * 1000) | round(0) }} l
```

Average consumption per day this month:

```jinja
{{ (states('sensor.water_this_month') | float(0) * 1000 / now().day) | round(0) }} l/day
```

Cost so far this year (example price 4.80 € per m³ incl. waste water):

```jinja
{{ (states('sensor.water_this_year') | float(0) * 4.80) | round(2) }} €
```

Days until the meter battery is empty:

```jinja
{{ (states('sensor.water_meter_battery_life') | float(0) * 365) | round(0) }} days
```
