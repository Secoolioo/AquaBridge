# 6 · Troubleshooting

[← 5 · Home Assistant](05-home-assistant.md) · next: [7 · How it works →](07-how-it-works.md)

Most problems fall into one of four groups: **nothing received**, **received but not decoded**,
**decoded but no values**, **board trouble**. For the first three, flash the
[debug snippets](../esphome/debug-snippets.yaml) and watch the log.

---

## Nothing received at all

**`[E][CC1101]: Invalid part number`** — the ESP32 cannot talk to the CC1101.
Check MOSI/MISO/SCK/CSN against the [wiring table](02-wiring.md), and that VCC is on **3V3**.

**No `Frame RSSI=` lines, only `Have data from radio (8 bytes)`** — those 8-byte packets are noise
that happens to look like a sync word. Completely normal, a few per second. Real frames are longer.

- Is the meter actually transmitting? Many show a radio symbol on the display.
- Some utilities configure "fixed network" mode where the meter sends only every 5–60 minutes.
  Wait an hour before you give up.
- `GDO0` must be connected to GPIO5 (`irq_pin`). Without the interrupt nothing is read.
- 868 MHz module? A 433 MHz CC1101 hears almost nothing.

## Received, but not decoded

**Long packets (e.g. 144 bytes) but no `Frame RSSI=` line** — the frame arrived with bit errors and
failed the 3-of-6 decoding or the CRC. The component drops these silently. This is a **signal**
problem: move the receiver closer to the meter. On my desk two floors up exactly this happened; next
to the meter every single frame decodes.

**`Telegram not handled by any handler`** — a valid frame from a meter you did not configure. Either a
neighbour's meter (ignore it) or yours with a different ID. Compare the ID bytes in the header with
your label, see [4.7](04-flashing.md#47-first-boot-is-the-meter-in-the-air).

**`(driver unknown!)` but the ID matches** — the driver does not know your meter version. Set
`meter_driver` explicitly (e.g. `hydrus`); Hydrus Type 173 also exists as version 0x63, 0x61, 0x75 and
decodes fine with `hydrus`.

**`telegram mac check failed` / `failed decryption. Wrong key?`** — wrong AES key, or a typo in it.
32 hex characters, no spaces.

## Decoded, but sensors stay "unknown"

- The `field:` names include the unit: `total_m3`, `flow_temperature_c`, `remaining_battery_life_y`,
  `rssi_dbm`. `total` alone does not work on the current component.
- Not every meter sends every field. Look at the decoded JSON (debug snippets) and only keep what
  appears there. My Hydrus does **not** send `flow_m3h` or `external_temperature_c`.
- `meter_id` without `0x`? Then the meter is not matched at all.

## The total in Home Assistant jumps or the statistics look wrong

The firmware already drops `0`, `NaN` and values lower than the last one, so a broken telegram can never
reset your statistics. If you changed `unit_of_measurement` or `state_class` after the first
statistics were written, Home Assistant shows a repair issue: fix it there.

---

## OTA update fails, or the board is dead after an update

This one cost me an evening, so here is the full story.

**Symptoms**

- OTA ends with `ERROR receiving update end result: Finishing update failed`.
- The device log shows `esp_image: image at 0x1d0000 has invalid magic byte (nothing flashed here?)`.
- The upload is very slow (2 minutes instead of 10 seconds) with pauses of several seconds.
- In the worst case, after the next power cycle the board only shows its power LED: no WiFi, no fallback
  hotspot. The serial console prints in a loop:

  ```text
  rst:0x10 (RTCWDT_RTC_RESET),boot:0x13 (SPI_FAST_FLASH_BOOT)
  wrong chip id 0xffff
  wrong chip id 0xffff
  ```

**Cause**

My board has an **XTX XT25F32B** flash chip (`esptool flash-id` → manufacturer `0b`, device `4016`).
On a weak USB charger with a long cable, the supply voltage sags while WiFi receives the update and the
flash erases and writes at the same time. The chip then erases slowly (3–4 s per 64 KB block instead of
0.2 s) and silently loses writes. I dumped the flash: 88 % of the new image was missing, and once the
first 64 KB block (bootloader and partition table) was hit as well. The running firmware keeps working
from RAM/cache, so you only notice it at the next reboot.

On the PC's USB port the same OTA went through in 12 seconds.

**What to do**

- Do firmware updates with the receiver on the **PC's USB port**:
  `esphome run aquabridge.yaml --device /dev/ttyUSB0`. In normal operation the flash is only read,
  which is uncritical.
- If you want OTA in place: good branded charger, short thick cable, and test it once on the desk.
- If `esphome upload` over USB fails with `MD5 of file does not match`, run it again.
- A failed OTA itself is harmless: the ESP32 keeps running the old firmware.

**Repair after "wrong chip id 0xffff"**

Connect it to the PC and flash over USB. This rewrites everything from address 0 (bootloader, partition
table, OTA data, app):

```bash
esphome upload aquabridge.yaml --device /dev/ttyUSB0
```

Hold **BOOT** if it does not connect. One minute later it is back.

**Long term:** if a board keeps doing this, replace it. Boards with GigaDevice or Winbond flash cost the
same and the configuration does not change.

## Other board issues

- **Boot loop with "Brownout detector was triggered"** — the power supply is too weak. Different charger
  or cable.
- **Port disappears on Linux** — `brltty` took it, see [4.1](04-flashing.md#usb-driver-and-permissions).
- **`version_4` of the component** — boot loops on ESPHome 2026.x (drivers get dropped at link time).
  Use the `main` branch pinned to a commit, like [`aquabridge.yaml`](../esphome/aquabridge.yaml).
- **Build error about `require_usb_serial_jtag_secondary`** — the external component list loaded the
  repository's outdated `esp32` override. List the components explicitly:
  `components: [wmbus_radio, wmbus_common, wmbus_meter]`.

## Still stuck?

Open an issue with:

- meter manufacturer, type, the version byte from the header
- the log at DEBUG level with the debug snippets, **AES key and meter ID redacted**
- your `aquabridge.yaml` without secrets
