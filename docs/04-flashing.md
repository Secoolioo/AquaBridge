# 4 · Flashing and first start

[← 3 · AES key](03-aes-key.md) · next: [5 · Home Assistant →](05-home-assistant.md)

## 4.1 Install ESPHome

You can use the ESPHome Device Builder app in Home Assistant, but I recommend the command line on a
PC: the first build is big, and you want the ESP32 on a USB cable next to you anyway.

**Linux / macOS**

```bash
pipx install esphome          # or: python3 -m pip install --user esphome
esphome version               # tested with 2026.10.1
```

**Windows**

```powershell
py -m pip install --user pipx
py -m pipx install esphome
esphome version
```

ESPHome 2026.10 needs Python 3.12 to 3.14.

### USB driver and permissions

- The board uses a **CH340**. Linux and macOS have the driver built in, Windows installs it via
  Windows Update. The port shows up as `/dev/ttyUSB0`, `/dev/cu.wchusbserial*` or `COM3`.
- **Linux:** your user must be in the `dialout` group:

  ```bash
  sudo usermod -aG dialout $USER     # log out and in again
  ```

- **Ubuntu / Mint:** if the port disappears a second after plugging in, `brltty` (braille driver) grabbed
  it. Plug the board in directly instead of through a cheap USB hub, or `sudo apt remove brltty`.

## 4.2 Get the files

```bash
git clone https://github.com/Secoolioo/AquaBridge.git
cd AquaBridge/esphome
cp secrets.example.yaml secrets.yaml
```

## 4.3 Fill in `secrets.yaml`

```yaml
wifi_ssid: "YourWiFi"
wifi_password: "YourWiFiPassword"
wmbus_key: "00112233445566778899AABBCCDDEEFF"   # from your utility
api_encryption_key: "...="                      # see below
ap_password: "choose-a-password"                # fallback hotspot
```

Generate the API key (32 random bytes, base64):

```bash
openssl rand -base64 32
# or
python3 -c "import base64,os;print(base64.b64encode(os.urandom(32)).decode())"
```

The ESP32 only supports **2.4 GHz** WiFi.

## 4.4 Set your meter ID

The meter ID is the last eight digits of the serial number on the label. Write it as **hex**, i.e. put
`0x` in front of the digits without changing them:

| Label | meter_id |
|---|---|
| `8 DME76 1234 5678` | `0x12345678` |
| `DME 7012 3456` (older format) | check the first frame in the log, see 4.7 |

```yaml
substitutions:
  meter_id: "0x12345678"
  meter_driver: hydrus
```

> [!WARNING]
> `meter_id: 12345678` (without `0x`) is read as a decimal number and becomes a completely different ID.
> The receiver then ignores your meter and you see nothing.

For other meters set `meter_driver` to the wmbusmeters driver name (`izar`, `multical21`, `apator162`,
`iperl`, `sensostar`, …). If you are not sure, see 4.7.

## 4.5 Validate and build

```bash
esphome config aquabridge.yaml    # -> INFO Configuration is valid!
esphome compile aquabridge.yaml
```

The first `compile` downloads ESP-IDF 5.5 and the Xtensa toolchain (several GB) and takes
10–20 minutes. Later builds take about a minute. At the end:

```text
RAM:   [====      ]  42.4% (used 76628 bytes from 180736 bytes)
Flash: [======    ]  64.9% (used 1191375 bytes from 1835008 bytes)
INFO Successfully compiled program.
```

## 4.6 Flash over USB

```bash
esphome upload aquabridge.yaml --device /dev/ttyUSB0
```

```text
Chip type:          ESP32-D0WD-V3 (revision v3.1)
Wrote 1257072 bytes (794601 compressed) at 0x00000000 in 21.8 seconds
Hash of data verified.
INFO Successfully uploaded program.
```

- If it hangs at `Connecting....`, hold the **BOOT** button until the upload starts.
- If it fails with `MD5 of file does not match data in flash`, just run it again.
- The first USB flash also replaces the factory firmware (on my board an old AT firmware).

## 4.7 First boot: is the meter in the air?

Watch the log:

```bash
esphome logs aquabridge.yaml --device /dev/ttyUSB0
```

You should see:

```text
[C][wmbus.transceiver]: Transceiver: CC1101
[C][wmbus.transceiver]:   IRQ Pin: GPIO5
[C][wmbus.transceiver]:   Frequency: 868.950 MHz
[C][wmbus_meter]: wM-Bus Meter:
[C][wmbus_meter]:   ID: 0x12345678
[C][wmbus_meter]:   Driver: hydrus
[I][wmbus]: Receiver task created
[C][wifi]:   IP Address: 192.168.1.163
```

With the production config (`logger: INFO`) the log stays quiet. To **see the telegrams** during
commissioning, copy the blocks from [`debug-snippets.yaml`](../esphome/debug-snippets.yaml) into
`aquabridge.yaml` (DEBUG level, `on_frame` and `on_telegram` logging) and flash again. Then:

```text
[D][main]: Frame RSSI=-32 dBm mode=T1 len=84 hex=5344a5117856341276078c00c9900f002c25...
[D][main]: Meter JSON: {"_":"telegram","media":"water","meter":"hydrus","id":"12345678",
           "at_date":"2025-12-31","flow_temperature_c":13.8,"remaining_battery_life_y":10.4,
           "total_m3":1234.567,"total_at_date_m3":1136.137,"status":"UNKNOWN_80","rssi_dbm":-32}
```

**Reading the frame header** (`5344a5117856341276078c…`):

| Bytes | Meaning |
|---|---|
| `53` | length |
| `44` | the meter sends (SND_NR) |
| `a511` | manufacturer DME = Diehl |
| `78563412` | meter ID, little endian → **12345678** |
| `76` | version 0x76 |
| `07` | water meter |

If your meter is in the air but you are not sure about ID or driver: frames of meters without a
configured `wmbus_meter` show up as `Telegram not handled by any handler` with the header bytes.

> [!TIP]
> Close to the PC there might be **no** complete frame at all, or only frames that fail the 3-of-6
> decoding. That is a range problem, not a configuration problem. Take the receiver to the meter first.

## 4.8 Put it next to the meter

1. Unplug from the PC, take it to the meter, plug into a decent USB charger.
2. Antenna up, 0.5 to 2 m from the meter, not inside a metal cabinet.
3. Within a minute Home Assistant shows values. Compare **`Total`** with the display of the meter.
   They must match to the litre.

Then remove the debug snippets again (or keep the production `aquabridge.yaml`) and flash once more
over USB.

## 4.9 Updates later

```bash
esphome run aquabridge.yaml --device /dev/ttyUSB0      # USB, always safe
esphome run aquabridge.yaml --device 192.168.1.163      # over WiFi (OTA)
```

OTA is encrypted with the API key. It works fine on a good power supply. If your board has a picky
flash chip like mine, read [this section](06-troubleshooting.md#ota-update-fails-or-the-board-is-dead-after-an-update)
before you update a receiver in the basement over the air.
