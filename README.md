# ESP32-S3 Environmental Sensing Station with e-Paper Display

A custom, compact, high-efficiency PCB designed for advanced Indoor Air Quality (IAQ) and atmospheric pressure monitoring based on the **ESP32-S3** SoC. The station features a low-power **Waveshare 2.9" e-Paper Display** ($296 \times 128$ px) for persistent, sunlight-readable data visualization without drawing significant power.

---

## 🛠 Main Features

* **Microcontroller:** ESP32-S3-WROOM-1-N4 (2.4 GHz Wi-Fi + BLE 5.0, Native USB).
* **Display Interface:** Waveshare 2.9" e-Paper Module ($296 \times 128$ px, Black/White) via 4-wire SPI.
* **Environmental Sensors:**
  * **SCD40 (Sensirion):** Photoacoustic $\text{CO}_2$, Temperature, and Relative Humidity sensor ($3.3\text{ V}$).
  * **SGP41 (Sensirion):** Indoor gas sensor for **VOC** and **$\text{NO}_x$** indices ($3.3\text{ V}$).
  * **BMP580 (Bosch Sensortec):** High-precision barometric pressure sensor ($3.3\text{ V}$).
  * **SEN50 (Sensirion - Optional):** Dedicated connector for particulate matter sensor (**PM1.0, PM2.5, PM4.0, PM10**).
* **Power Management:** High-current **UZ1086L-33** LDO regulator (TO-263-3 package, up to $1.5\text{ A}$) step-down from $5\text{ V}$ USB-C.

---

## 📌 ESP32-S3 GPIO Mapping

| GPIO | Signal / Function | Hardware Notes |
| :---: | :--- | :--- |
| **IO1** | `INT_PRESSURE` | BMP580 Hardware Interrupt |
| **IO8** | `I2C_DATA` | I2C Data Line (SDA) — $2.2\text{ k}\Omega$ Pull-up to $3.3\text{ V}$ |
| **IO9** | `I2C_CLK` | I2C Clock Line (SCL) — $2.2\text{ k}\Omega$ Pull-up to $3.3\text{ V}$ |
| **IO10** | `DISPLAY_CS` | SPI Chip Select for e-Paper Display (P1 Pin 5) |
| **IO11** | `DISPLAY_DIN` | SPI Data (MOSI) for e-Paper Display (P1 Pin 3) |
| **IO12** | `DISPLAY_CLK` | SPI Clock (SCK) for e-Paper Display (P1 Pin 4) |
| **IO13** | `DISPLAY_DC` | Data / Command Selection for e-Paper (P1 Pin 6) |
| **IO14** | `DISPLAY_RST` | Hardware Reset for e-Paper (P1 Pin 7) |
| **IO21** | `DISPLAY_BUSY` | Status Busy Line for e-Paper (P1 Pin 8) |
| **IO0** | `BOOT_SW` | BOOT Button (`SW1`) with RC Debouncing Filter |
| **IO19** | `USB_D-` | Native USB Data Line D- |
| **IO20** | `USB_D+` | Native USB Data Line D+ |
| **EN** | `RESET_MCU` | System Hardware Reset Button (`SW2`) |

---

## 🗺 I2C Address Map

All onboard sensors share the same $3.3\text{ V}$ I2C bus (`IO8` / `IO9`). Bus addresses are pre-configured to prevent conflicts:

| Sensor | Measurements | I2C Address (Hex) | Configuration Notes |
| :--- | :--- | :---: | :--- |
| **BMP580** | Pressure & Temp | **`0x47`** | `SDO` and `CSB` pins pulled High to `3V3` |
| **SGP41** | VOC & $\text{NO}_x$ | **`0x59`** | Factory Fixed |
| **SCD40** | $\text{CO}_2$, Temp, Humidity | **`0x62`** | Factory Fixed |
| **SEN50** | Particulate Matter (PM) | **`0x69`** | Factory Fixed (`SEL` pin pulled Low to `GND`) |

---

## 🔌 External Module Connectors

### 1. e-Paper Display Interface (Waveshare 2.9" Module - LCSC `C359943`)
The PCB footprint supports an 8-pin connector (**$2.00\text{ mm}$ JST PH** or standard **$2.54\text{ mm}$ Pin Header**).

| Pin (Connector P1) | Display Signal | PCB / ESP32-S3 Connection |
| :---: | :--- | :--- |
| **1** | VCC | `3V3` Rail (with C12 $100\text{ nF}$ decoupling) |
| **2** | GND | Ground Plane (`GND`) |
| **3** | DIN (MOSI) | `IO11` |
| **4** | CLK (SCK) | `IO12` |
| **5** | CS | `IO10` |
| **6** | DC | `IO13` |
| **7** | RST | `IO14` |
| **8** | BUSY | `IO21` |

---

### 2. Optional PM Sensor Connector (Sensirion SEN50)
The PCB includes an SMT **JST GH 6-pin connector ($1.25\text{ mm}$ pitch)** for attaching an external particulate matter sensor.

| JST GH Pin | SEN50 Pin Name | PCB Connection | Electrical Specifications |
| :---: | :--- | :--- | :--- |
| **1** | VDD | **`+5V`** Rail | Sourced directly from USB-C (Powers fan and laser) |
| **2** | GND | `GND` | Common Ground |
| **3** | SDA | `I2C_DATA` (`IO8`) | $3.3\text{ V}$ Logic I2C Data Line |
| **4** | SCL | `I2C_CLK` (`IO9`) | $3.3\text{ V}$ Logic I2C Clock Line |
| **5** | SEL | `GND` | **Forces I2C Protocol Mode** |
| **6** | NC | Not Connected | Unused Pin |

---

## ⚡ Power Supply Architecture

* **Input:** $5\text{ V}$ via USB-C Type Connector.
* **Main LDO ($3.3\text{ V}$):** **UZ1086L-33-TQ3-R** (TO-263-3 / D2PAK Package).
  * *Input Filtering:* $10\mu\text{F}$ (X5R) + $100\text{ nF}$ (X7R).
  * *Output Filtering:* $22\mu\text{F}$ (X5R) + $100\text{ nF}$ (X7R).
  * *Thermal Management:* Tab soldered to a large ground copper flood with 6 thermal vias.

---

## 💻 Sample ESPHome Configuration

```yaml
esphome:
  name: env-sensor-station

esp32:
  board: esp32-s3-devkitc-1
  framework:
    type: arduino

i2c:
  sda: GPIO8
  scl: GPIO9
  scan: true

spi:
  clk_pin: GPIO12
  mosi_pin: GPIO11

display:
  - platform: waveshare_epaper
    cs_pin: GPIO10
    dc_pin: GPIO13
    reset_pin: GPIO14
    busy_pin: GPIO21
    model: 2.9in
    full_update_every: 30

sensor:
  - platform: scd4x
    co2:
      name: "Indoor CO2"
    temperature:
      name: "SCD40 Temperature"
    humidity:
      name: "SCD40 Humidity"

  - platform: bmp580
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

## 📁 Repository Structure

```text
.
├── hardware/
│   ├── schematics/       # KiCad Schematic Files (.kicad_sch)
│   ├── pcb/              # KiCad PCB Layout Files (.kicad_pcb)
│   └── gerber/           # Manufacturing Gerber Files (JLCPCB ready)
├── docs/                 # Component Datasheets and Wiring Guides
└── firmware/             # Firmware Examples for ESPHome & Arduino (GxEPD2)
```