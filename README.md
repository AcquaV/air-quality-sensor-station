# ESP32-S3 Air Quality & Weather Station with 2.0" TFT Display

A compact, USB-C powered PCB for indoor air quality (IAQ) monitoring and a simple
barometric weather forecast, based on the **ESP32-S3**. It reads CO2, VOC/NOx,
temperature, humidity, pressure and (optionally) particulate matter, keeps time via
NTP, and shows everything on an external **2.0" IPS TFT** that wakes up on a
capacitive touch button.

---

## 🛠 Main Features

* **Microcontroller:** ESP32-S3-WROOM-1-N4 (2.4 GHz Wi-Fi + BLE 5.0, native USB).
* **Display:** external 2.0" IPS TFT LCD module, 240 x 320, ST7789 driver, 4-wire SPI.
  The module has **no backlight pin**, so it is switched on/off through its supply
  (see [Display Power Switch](#-display-power-switch)).
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

| Block | Part / value | Notes |
| :--- | :--- | :--- |
| Input | USB-C 16-pin receptacle, back side | CC1/CC2: 5.1 kΩ to GND (one each) |
| Surge protection | TVS **SMF5.0A** on VBUS, **before** the PTC | cathode to VBUS |
| Overcurrent | Resettable **PTC 0.75 A hold, 1812** | |
| Data-line ESD | **USBLC6-2SC6** (SOT-23-6) on USB_DATA+/- | next to the connector, short ground via; VBUS pin on the pre-PTC node |
| LDO | **LDL1117S33R** (ST), 3.3 V, 1.2 A, SOT-223 (LCSC `C435835`) | tab = VOUT (per ST datasheet) |
| Input caps | 10 µF + 100 nF, ceramic X5R/X7R, **16 V** | 10 µF in 0805 or larger |
| Output caps | 10 µF + 100 nF, ceramic X5R/X7R, **16 V** | LDL1117 is designed for ceramic output caps; keep ≥ 4.7 µF effective |
| Test points | 5V, 3V3, GND | |

**Thermal note:** the LDO dissipates (5 V − 3.3 V) x I (about 0.5 W at 300 mA). The tab
is **VOUT**, so it goes on a large **3V3** copper area (not GND), with thermal vias to a
3V3 area on another layer if you want to spread the heat.

---

## 📌 ESP32-S3 GPIO Mapping

| GPIO | Net label | Function | Notes |
| :---: | :--- | :--- | :--- |
| **IO0** | `IO0` | BOOT button (SW1) | 10 kΩ pull-up (R2), 100 nF debounce (C11) |
| **IO1** | `INT_PRESS` | BMP580 interrupt | optional |
| **IO4** | `TOUCH` | Capacitive button output (CN3) | 100 kΩ pull-down (R4) |
| **IO8** | `I2C_DATA` | I2C SDA | 2.2 kΩ pull-up (R5), test point TP6 |
| **IO9** | `I2C_CLK` | I2C SCL | 2.2 kΩ pull-up (R6), test point TP7 |
| **IO10** | `SCL_D` | TFT SPI clock (SCK) | |
| **IO11** | `SDA_D` | TFT SPI data (MOSI) | |
| **IO12** | `RST_D` | TFT reset | |
| **IO13** | `DC_D` | TFT data / command | |
| **IO14** | `CS_D` | TFT chip select | |
| **IO21** | `D_ENABLE` | TFT power switch control | active low, see [Display Power Switch](#-display-power-switch) |
| **IO19** | `USB_DATA-` | Native USB D- | |
| **IO20** | `USB_DATA+` | Native USB D+ | |
| **IO38** | | Status LED (LED1) | through 2.2 kΩ (R9), LED to GND |
| **EN** | | Reset button (SW2) | 10 kΩ pull-up (R1), 1 µF (C2) |

Unused GPIOs: `IO2`, `IO5`-`IO7`, `IO15`-`IO18`, `IO35`-`IO37`, `IO39`-`IO42`, `IO47`,
`IO48`. Avoid **IO3, IO45, IO46** (strapping pins).

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

### 1. TFT display (CN2, 8 pins)

| Pin | Signal | ESP32-S3 |
| :---: | :--- | :--- |
| 1 | GND | GND |
| 2 | VCC | switched 3V3 (display supply, see below) |
| 3 | SCL | `IO10` (`SCL_D`) |
| 4 | SDA | `IO11` (`SDA_D`) |
| 5 | RST | `IO12` (`RST_D`) |
| 6 | DC | `IO13` (`DC_D`) |
| 7 | CS | `IO14` (`CS_D`) |
| 8 | NC | not connected |

The module header has 7 pins, read from left to right on the back of the module (header
at the top): CS, DC, RST, SDA, SCL, VCC, GND. Wire the cable so that **CN2 pin 1 (GND)
goes to the module's GND pin** and CN2 pin 7 (CS) to the module's CS pin, i.e. the order
is reversed with respect to the module. Check the labels printed on your module before
making the cable.

### 2. Capacitive touch button (CN3, 3 pins)

| Pin | Signal |
| :---: | :--- |
| 1 | GND |
| 2 | 3V3 |
| 3 | `TOUCH` (`IO4`, 100 kΩ pull-down) |

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

---

## 🖥 Display Power Switch

The TFT module has no backlight pin: its backlight is on whenever the module is powered.
To turn the display off after the timeout, the module supply is switched by a high-side
**P-MOSFET** (Q1, AO3401A, SOT-23, LCSC `C15127`).

| Part | Value | Function |
| :--- | :--- | :--- |
| Q1 | AO3401A | source on 3V3, drain on the display supply |
| R11 | 10 kΩ | in series from `D_ENABLE` (`IO21`) to the gate |
| R10 | 100 kΩ | pull-up from `D_ENABLE` to 3V3: display off at boot / reset |
| C17 | 100 nF | gate to 3V3 (source): soft start with R11 (about 1 ms) |
| C12 + C16 | 10 µF + 100 nF | decoupling on the display supply, next to CN2 |
| Discharge | 2 x 100 kΩ in parallel (50 kΩ) | across C12/C16: discharges the display supply when off |

`D_ENABLE` low = display on, high or high-impedance = display off. The soft start limits
the inrush current into the module capacitors, which would otherwise dip the 3V3 rail
and could reset the ESP32.

Firmware notes:
* The TFT must be **re-initialized after every power-up**: set `D_ENABLE` low, wait
  20-50 ms, re-enable the SPI pins, pulse `RST_D` low for about 10 ms, wait about 120 ms,
  then run the driver init. Drivers that initialize the display once at boot (such as
  ESPHome's built-in display component) are not ideal with power gating; an Arduino
  sketch with LovyanGFX or TFT_eSPI is easier.
* Before switching the display off, drive `CS_D`, `DC_D`, `RST_D`, `SDA_D` and `SCL_D`
  **low** (or set them as inputs), so the module is not back-powered through its signal
  pins.
* Touch behaviour: a short touch wakes the display when it is off, or changes screen
  when it is on; an inactivity timeout (for example 30-60 s) switches it off again.

---

## 📊 Power Budget (estimates)

| Load | Typical | Notes |
| :--- | :--- | :--- |
| ESP32-S3 | ~0.1 A, Wi-Fi peaks up to ~0.5 A | |
| TFT (backlight on) | 25-80 mA | depends on the module; measure it |
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
