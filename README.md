# ESP32-S3 Air Quality & Weather Station with 2.0" TFT Display

🇮🇹 Italian version: [README.it.md](README.it.md)

A compact, USB-C powered PCB for indoor air quality (IAQ) monitoring and a simple
barometric weather forecast, based on the **ESP32-S3**. It reads CO2, VOC/NOx,
temperature, humidity, pressure and (optionally) particulate matter, keeps time via
NTP, and shows everything on an external **2.0" IPS TFT** that wakes up on a
capacitive touch button.

> **Design status:** this README describes the target design (v2). Items marked
> **[TODO]** are decided but not yet implemented in the schematic/PCB. See
> [Design status](#-design-status--todo) at the bottom.

---

## 🛠 Main Features

* **Microcontroller:** ESP32-S3-WROOM-1-N4 (2.4 GHz Wi-Fi + BLE 5.0, native USB).
* **Display:** external 2.0" IPS TFT LCD module, 240 x 320, ST7789 driver, 4-wire SPI,
  7-pin header (CS, DC, RST, SDA, SCL, VCC, GND). **No backlight pin**, so the module
  is switched on/off through its supply (see [Display power switch](#display-power-switch)).
* **User input:** one separate capacitive touch button (TTP223 module, mounted on the
  enclosure): wakes the display after the timeout and switches screen.
* **Sensors (I2C):**
  * **SCD40 (Sensirion):** CO2, temperature, relative humidity.
  * **SGP41 (Sensirion):** VOC and NOx indices (uses SCD40 temperature/humidity for compensation).
  * **BMP580 (Bosch Sensortec):** barometric pressure, used for the weather forecast.
  * **SEN50 (Sensirion, optional):** PM1.0 / PM2.5 / PM4.0 / PM10 on a 6-pin JST GH connector.
* **Time:** NTP over Wi-Fi (no external RTC or crystal). Interval timing uses the
  ESP32 hardware timers.
* **Weather forecast:** short-term forecast from sea-level-corrected pressure and its
  3-hour trend (Zambretti-style algorithm, in firmware).
* **Power:** 5 V from USB-C, 3.3 V from an LDL1117S33R LDO, with input protection.

---

## ⚡ Power Supply Architecture

```
USB-C VBUS ─► TVS (SMF5.0A) ─► PTC 0.75 A (1812) ─► 5V rail ─► LDL1117S33R ─► 3V3
                  │                                     └──────► SEN50 (5 V)
               ESD USBLC6-2SC6 on D+/D- (VBUS pin to the pre-fuse node)
```

| Block | Part / value | Notes |
| :--- | :--- | :--- |
| Input | USB-C 16-pin receptacle, back side | CC1/CC2: 5.1 kΩ to GND (one each) |
| Surge protection | TVS **SMF5.0A** on VBUS, **before** the PTC | cathode to VBUS |
| Overcurrent | Resettable **PTC 0.75 A hold, 1812** | LCSC code: *to be confirmed* |
| Data-line ESD | **USBLC6-2SC6** (SOT-23-6) on USB_DATA+/- | placed next to the connector, short ground via |
| LDO | **LDL1117S33R** (ST), 3.3 V, 1.2 A, SOT-223 (LCSC `C435835`) | tab = VOUT (per ST datasheet) |
| Input caps | 10 µF + 100 nF, ceramic X5R/X7R, **16 V** | 10 µF in 0805 or larger |
| Output caps | 10 µF + 100 nF, ceramic X5R/X7R, **16 V** | LDL1117 is designed for ceramic output caps; keep ≥ 4.7 µF effective |
| Test points | 5V, 3V3, GND | |

**Thermal note:** the LDO dissipates (5 V − 3.3 V) x I (about 0.5 W at 300 mA). The tab
is **VOUT**, so it goes on a large **3V3** copper area (not GND), with thermal vias to a
3V3 area on another layer if you want to spread the heat.

---

## 📌 ESP32-S3 GPIO Mapping

| GPIO | Signal | Notes |
| :---: | :--- | :--- |
| **IO0** | `BOOT_SW` | BOOT button (SW1) with RC debounce |
| **IO1** | `INT_PRESSURE` | BMP580 interrupt (optional) |
| **IO4** | `TOUCH_BTN` | TTP223 module output (active high, momentary), 100 kΩ pull-down |
| **IO8** | `I2C_SDA` | 2.2 kΩ pull-up to 3V3 |
| **IO9** | `I2C_SCL` | 2.2 kΩ pull-up to 3V3 |
| **IO10** | `TFT_CS` | TFT chip select |
| **IO11** | `TFT_SDA` (MOSI) | SPI data |
| **IO12** | `TFT_SCL` (SCK) | SPI clock |
| **IO13** | `TFT_DC` | Data / command |
| **IO14** | `TFT_RST` | TFT reset |
| **IO21** | `TFT_PWR_EN` | Gate of the P-MOSFET that switches the TFT supply (active low, 100 kΩ pull-up) |
| **IO19 / IO20** | `USB_D-` / `USB_D+` | Native USB |
| **IO38** | `STATUS_LED` | LED1 through R9 |
| **EN** | `RESET_MCU` | Reset button (SW2) |

`IO2` is currently **not connected** to anything (free). Avoid **IO3, IO45, IO46**
(strapping pins). Signals marked **[TODO]** in the status section still have to be
routed in the schematic.

---

## 🗺 I2C Address Map

All sensors share the 3.3 V I2C bus (`IO8` / `IO9`).

| Sensor | Measurements | Address | Notes |
| :--- | :--- | :---: | :--- |
| **BMP580** | Pressure & temperature | `0x47` | `SDO` and `CSB` pulled high to 3V3 |
| **SGP41** | VOC & NOx | `0x59` | Fixed |
| **SCD40** | CO2, temperature, humidity | `0x62` | Fixed |
| **SEN50** | Particulate matter | `0x69` | Fixed (`SEL` pin to GND = I2C mode) |

---

## 🔌 External Connectors

### 1. TFT display (7-pin)
The header order **matches the module** (CS, DC, RST, SDA, SCL, VCC, GND), so a straight
1:1 cable can be used. Check the pin order printed on the back of your module.

| Pin | Module pin | PCB connection |
| :---: | :--- | :--- |
| 1 | CS | `IO10` |
| 2 | DC | `IO13` |
| 3 | RST | `IO14` |
| 4 | SDA | `IO11` |
| 5 | SCL | `IO12` |
| 6 | VCC | `TFT_3V3` (switched 3V3, with 10 µF + 100 nF at the connector) |
| 7 | GND | `GND` |

Suggested connector: JST PH 2.0 mm, 7 pins (e.g. `B7B-PH-K-S`, LCSC `C157981` — *verify
stock and whether it is a Basic/Extended part*). Optional 22-47 Ω series resistors on
SCL, SDA and CS.

### 2. Capacitive touch button (3-pin)

| Pin | Signal |
| :---: | :--- |
| 1 | 3V3 |
| 2 | GND |
| 3 | `TOUCH_BTN` (`IO4`, series resistor 100-470 Ω, 100 kΩ pull-down) |

The TTP223 module is glued behind the enclosure wall (1-3 mm, no air gap). Check the
module's jumpers for active-high / momentary mode.

### 3. SEN50 particulate matter sensor (JST GH, 6-pin, 1.25 mm)

| Pin | SEN50 pin | PCB connection |
| :---: | :--- | :--- |
| 1 | VDD | `+5V` (after the PTC, with a local 10 µF) |
| 2 | GND | `GND` |
| 3 | SDA | `IO8` |
| 4 | SCL | `IO9` |
| 5 | SEL | `GND` (forces I2C) |
| 6 | NC | not connected |

> ⚠️ Use a **6-pin JST GH** receptacle. LCSC `C157928` is a 3-pin JST XH (2.5 mm) and is
> **not** suitable for the SEN50.

---

## 🖥 Display Power Switch

The TFT module has no backlight pin: its backlight is on whenever the module is powered.
To turn the display off after the timeout, the module supply is switched by a high-side
**P-MOSFET** (logic-level, e.g. AO3401A class — verify Vgs(th)): source on 3V3, drain on
`TFT_3V3`, gate on `IO21` with a 100 kΩ pull-up to 3V3 (display off at boot).

Firmware notes:
* The TFT must be **re-initialized after every power-up**. Drivers that initialize the
  display once at boot (such as ESPHome's built-in display component) are not ideal
  with power gating; an Arduino sketch with LovyanGFX or TFT_eSPI is easier.
* While the display is off, keep the SPI lines to the TFT low or high-impedance so the
  module is not back-powered through its signal pins.

---

## 📊 Power Budget (estimates)

| Load | Typical | Notes |
| :--- | :--- | :--- |
| ESP32-S3 | ~0.1 A, Wi-Fi peaks up to ~0.5 A | |
| TFT (backlight on) | 25-60 mA | depends on the module; measure it |
| TFT (off) | ~0 | via the P-MOSFET |
| Sensors | ~20 mA | |
| SEN50 | ~60-70 mA at 5 V | optional |

The LDO is 1.2 A, so there is plenty of margin. Heat is set by (5 V − 3.3 V) x I.

---

## 🌦 Weather Forecast

The pressure measured by the BMP580 is corrected to sea level using your altitude and
compared over time. The 3-hour trend selects a forecast text (rising: improving,
falling: worsening, steady: unchanged). The pressure history is kept in flash so it
survives power loss (see [Pressure History in Flash](#-pressure-history-in-flash)).

Place the BMP580 and SCD40 **away from the LDO and from the TFT**, and consider a PCB
slot between them: self-heating biases temperature and humidity.

---

## 💾 Pressure History in Flash

The 3-hour pressure trend needs a history that must survive a reboot or a power loss.
It is stored in the 4 MB flash of the module (NVS or LittleFS); the amount of data is
tiny.

| Item | Choice |
| :--- | :--- |
| Storage | NVS (key-value, with wear levelling) or LittleFS |
| Data | Circular buffer of samples: `{unix timestamp, raw pressure}` (a few bytes each) |
| Sampling | One sample every 10-15 min, keep the last 6 h (about 24-36 samples) |
| Flash writes | Samples are kept in RAM and the buffer is written to flash every 30 min |
| What is stored | **Raw** sensor pressure (the sea-level correction is applied when reading) |

**Boot sequence**
1. Wait for the NTP time sync (timestamps are only valid after it).
2. Load the buffer from flash.
3. Discard samples that are too old (e.g. older than 6 h) or have an invalid timestamp.
4. If at least 3 h of valid history remain, the trend and the forecast are available
   immediately; otherwise they are shown after the history has been rebuilt.

**Notes**
* **Wear:** one write every 30 min is about 17,500 writes per year. NOR flash is
  typically rated for around 100,000 erase cycles per sector (check the flash datasheet
  of your module) and NVS spreads the writes over several sectors, so wear is not a
  practical concern. Do **not** write to flash at every measurement.
* **Power loss:** without a battery-backed RTC, the samples taken since the last flash
  write (at most 30 min) are lost.
* **No timestamp, no sample:** do not store samples before the time is synchronized.

---

## 💻 Sample ESPHome Configuration (sensors, time, touch)

The TFT part is not included because of the power-gating note above. Check the platform
names for your ESPHome version (e.g. `bmp581` for the BMP58x family).

```yaml
esphome:
  name: air-weather-station

esp32:
  board: esp32-s3-devkitc-1
  framework:
    type: arduino

i2c:
  sda: GPIO8
  scl: GPIO9
  scan: true

time:
  - platform: sntp
    id: sntp_time
    timezone: "CET-1CEST,M3.5.0,M10.5.0/3"

binary_sensor:
  - platform: gpio
    id: touch_button
    name: "Touch Button"
    pin:
      number: GPIO4
      mode: INPUT
    filters:
      - delayed_on: 30ms

switch:
  - platform: gpio
    id: tft_power
    name: "TFT Power"
    restore_mode: ALWAYS_OFF
    pin:
      number: GPIO21
      inverted: true

sensor:
  - platform: scd4x
    co2:
      name: "Indoor CO2"
    temperature:
      name: "SCD40 Temperature"
      id: scd_temp
    humidity:
      name: "SCD40 Humidity"
      id: scd_hum

  - platform: sgp4x
    voc:
      name: "VOC Index"
    nox:
      name: "NOx Index"
    compensation:
      humidity_source: scd_hum
      temperature_source: scd_temp

  - platform: bmp581        # verify support for the BMP580 variant
    address: 0x47
    pressure:
      name: "Atmospheric Pressure"

  - platform: sen5x
    id: sen50_sensor
    address: 0x69
    pm_1_0:
      name: "PM 1.0"
    pm_2_5:
      name: "PM 2.5"
    pm_10_0:
      name: "PM 10.0"
```

---

## ✅ Design status / TODO

- [ ] Replace the old 8-pin display connector with the 7-pin TFT connector (IO10-IO14 reused).
- [ ] Add the TFT power switch (P-MOSFET on `IO21`) and decoupling on `TFT_3V3`.
- [ ] Add the 3-pin touch connector (`IO4`, series resistor, pull-down).
- [ ] Replace the SEN50 connector with a true 6-pin JST GH part.
- [ ] Connect the USBLC6 VBUS pin to the pre-PTC node (CC1/CC2 are left without ESD
  protection: they only have the 5.1 kΩ resistors, no ICs to protect).
- [ ] Confirm the USB-C receptacle part (vertical, back side) and its footprint.
- [ ] PCB: LDO tab on a 3V3 copper area; ground vias on every GND pad; sensors away from LDO/TFT.
- [ ] Run ERC (add `PWR_FLAG` if needed), DRC with the JLCPCB 4-layer rules, update PCB from schematic.
- [ ] Optional: SEN50 load switch (not planned yet; `IO2` and other GPIOs are free).

---

## 📁 Repository Structure

```text
.
├── hardware/
│   ├── schematics/       # KiCad schematic files (.kicad_sch)
│   ├── pcb/              # KiCad PCB layout files (.kicad_pcb)
│   └── gerber/           # Manufacturing files (JLCPCB ready)
├── docs/                 # Datasheets and wiring guides
└── firmware/             # Firmware: ESPHome (sensors) and Arduino (TFT, LovyanGFX/TFT_eSPI)
```
