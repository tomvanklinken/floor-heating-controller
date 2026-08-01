# Jaga DBH controller (ESPHome package)

ESPHome-configuratie voor een [Jaga DBH](https://www.jaga.nl/) radiator
(ventilator-ondersteund, verwarmen + light cooling), als herbruikbaar
**package**: in Home Assistant definieer je per radiator alleen de
substitutions.

Dit is een fork van
[nliaudat/floor-heating-controller](https://github.com/nliaudat/floor-heating-controller),
omgebouwd naar één klepkanaal op de hardware van
[esp32_8ch_motor_shield](https://github.com/nliaudat/esp32_8ch_motor_shield).

## Hardware

| Onderdeel | Functie |
| --- | --- |
| Nologo ESP32-C3 SuperMini | controller (op het motor shield) |
| esp32_8ch_motor_shield, kanaal 1 | proportionele klep (BEMF-eindstopdetectie) |
| GP8403 DAC (I2C, `0x5F`) | 0-10V aansturing Jaga ventilator, kanaal 0 |
| 2x DS18B20 (1-wire, GPIO6) | aanvoer- en retourtemperatuur |
| BTHome thermometer (pvvx firmware) | kamertemperatuur + luchtvochtigheid via BLE |

Standaard I2C-pinnen voor de GP8403: SDA = GPIO8, SCL = GPIO9
(aanpasbaar via de substitutions `i2c_sda_pin` / `i2c_scl_pin`). GPIO8 is
daardoor bezet; er is dus geen status-LED.

## Werking

- **Thermostaat** (climate entity) met verwarmen én koelen. De ventilator
  (3 standen, zoals de Jaga DBH) zit als fan mode in de climate card en
  draait alleen bij warmte- of koelvraag: LOW/MEDIUM/HIGH zijn vaste
  standen, **AUTO** (standaard) kiest de stand op basis van het verschil
  t.o.v. het setpoint (MEDIUM vanaf `fan_auto_medium_error` = 1 °C, HIGH
  vanaf `fan_auto_high_error` = 2 °C).
- **Verwarmen**: klep volledig open, ventilator op de gekozen stand.
- **Koelen**: een PID-regelaar moduleert de proportionele klep zodat de
  **retourtemperatuur op dauwpunt + marge** blijft (standaard +0,75 °C,
  instelbaar via de number-entity "Dauwpunt marge"). Het dauwpunt wordt
  berekend uit de BTHome kamertemperatuur en -luchtvochtigheid. Valt het
  dauwpunt weg (BLE-sensor onbereikbaar), dan stopt het koelen automatisch.
- De PID-parameters (Kp/Ki/Kd) zijn live instelbaar via number-entities.

## Gebruik in Home Assistant

Maak in het ESPHome dashboard een nieuw apparaat aan met (zie
[example.yaml](example.yaml)):

```yaml
substitutions:
  kamer: "stein"                     # -> device heet radiator-stein
  bthome_mac: "38:1F:8D:C5:B5:E2"
  bthome_bindkey: "geheim"
  api_key: "geheim"                  # openssl rand -base64 32
  dallas_aanvoer: "0x7b0725400b8d9528"
  dallas_retour: "0x7807254009846228"

packages:
  jaga_dbh:
    url: https://github.com/tomvanklinken/floor-heating-controller
    ref: main
    files: [jaga-dbh.yaml]
    refresh: 1d
```

WiFi komt uit de `secrets.yaml` van je ESPHome config-map
(`wifi_ssid` / `wifi_password`). Als wifi niet lukt start een fallback
hotspot `radiator-<kamer>` (wachtwoord: `jaga-fallback`, aanpasbaar via de
substitution `fallback_ap_password`).

Alle overige defaults (doeltemperaturen, PID-parameters, pinnen, BEMF-trigger,
enz.) staan in [jaga-dbh.yaml](jaga-dbh.yaml) en kun je per apparaat overriden
in de `substitutions`.

### Kalibratie

De klepkalibratie draait alleen handmatig: druk eenmalig bij installatie
(gemonteerd op de radiator) op de button **"Klep kalibreren"**. De positie
blijft bewaard over reboots en hersynchroniseert bovendien vanzelf via de
BEMF-eindstop bij elke volledige open/dicht-beweging. Tip: druk na het
kalibreren op **"Herstart"**, dan wordt de positie direct naar flash
geschreven (anders gebeurt dat op zijn laatst na een uur).

### Dallas-adressen vinden

Druk op de button **"Zoek 1-wire adressen"** en kijk in de logs, of laat
`dallas_aanvoer`/`dallas_retour` op de placeholder staan bij de eerste flash.

## Structuur

```
jaga-dbh.yaml            # hoofd-package: defaults + includes
packages/
  core.yaml              # board (ESP32-C3), api, ota, logger
  wifi.yaml              # wifi + fallback AP + captive portal
  time.yaml              # sntp + wekelijkse onderhouds-reboot
  diagnostics.yaml       # wifi-signaal, uptime, herstart-button
  bthome.yaml            # BLE tracker, bthome_mithermometer, dauwpunt
  dallas.yaml            # 1-wire aanvoer/retour sensoren
  fan.yaml               # GP8403 DAC + fan (3 standen)
  valve.yaml             # sn74hc595, BEMF, cover, PID-output, kalibratie
  climate.yaml           # thermostaat + koel-PID + regel-scripts
example.yaml             # HA-config met remote package (dit gebruik je)
example-local.yaml       # zelfde, maar met lokale includes (ontwikkeling)
```

## Lokaal ontwikkelen / testen

```bash
pip install --upgrade esphome
esphome config example-local.yaml    # valideren
esphome compile example-local.yaml   # compileren
esphome run example-local.yaml       # flashen
```

Eerste keer flashen: ESP32 los van het shield via USB, boot-knop 2-3 s
ingedrukt houden voordat de seriële verbinding start. Daarna gaat alles via
OTA.
