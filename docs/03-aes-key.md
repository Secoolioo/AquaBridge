# 3 · Getting the AES key

[← 2 · Wiring](02-wiring.md) · next: [4 · Flashing →](04-flashing.md)

Almost every modern water meter encrypts its telegrams with AES-128 (OMS security mode 5 or 7).
Without the key you can see *that* your meter is sending, but not *what*.

## Who has the key

Not the manufacturer, but whoever operates the meter:

- your **water utility** (Stadtwerke, Wasserwerk, Wasserzweckverband, …) or
- the **metering service** they hired (often listed on your water bill).

Many utilities hand out the individual key of *your* meter for private use. Some need a form, some want
proof that you live there, some only answer letters. It is your consumption data, so ask.

## What to send

- your name, address, customer number if you have one
- manufacturer and type, e.g. *Diehl Metering Hydrus, Type 173*
- the **serial number** from the label, e.g. `8 DME76 1234 5678`
- that you only want to **read** your own meter for your home automation

## Letter template (German)

```text
Betreff: Bitte um Mitteilung des AES-Schlüssels für meinen Wasserzähler (wM-Bus)

Sehr geehrte Damen und Herren,

in meinem Haushalt ist ein Wasserzähler mit Funkschnittstelle (wM-Bus, 868 MHz) verbaut:

- Hersteller/Typ:   [z. B. Diehl Metering Hydrus, Type 173]
- Seriennummer:     [z. B. 8 DME76 1234 5678]
- Verbrauchsadresse: [Straße, Ort]
- Kundennummer:     [falls bekannt]

Ich möchte meinen Wasserverbrauch für den Eigengebrauch in meine Hausautomation einbinden. Da die
Funktelegramme AES-verschlüsselt sind, bitte ich Sie um den individuellen Schlüssel dieses Zählers.

Ich verwende den Schlüssel ausschließlich lesend für meinen eigenen Zähler. Eine Beeinflussung der
Abrechnung oder des Zählers ist damit nicht möglich. Einen Nachweis, dass ich Anschlussnutzer bin,
reiche ich gerne nach.

Mit freundlichen Grüßen
[Name]
```

## Letter template (English)

```text
Subject: Request for the AES key of my water meter (wireless M-Bus)

Dear Sir or Madam,

my household has a water meter with a wireless M-Bus interface (868 MHz):

- Manufacturer/type: [e.g. Diehl Metering Hydrus, Type 173]
- Serial number:     [e.g. 8 DME76 1234 5678]
- Address:           [street, city]
- Customer number:   [if known]

I would like to read my own consumption in my home automation system. As the radio telegrams are
AES encrypted, I kindly ask you for the individual key of this meter. I will only use it to read my
own meter; it does not allow any change to the meter or the billing.

Kind regards
[Name]
```

## What the key looks like

32 hexadecimal characters, e.g.

```text
00112233445566778899AABBCCDDEEFF
```

Put it into `esphome/secrets.yaml` as `wmbus_key`. If the key is wrong, the log says
`telegram mac check failed` (mode 7) or `failed decryption. Wrong key?` (mode 5).

> [!CAUTION]
> Treat the key like a password. Do not paste raw telegrams together with the key into online
> analyzers, and do not commit `secrets.yaml`. This repository's `.gitignore` already excludes it.
