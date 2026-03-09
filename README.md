# OHIS BLE Bridge — KiCad 7 Schematics
## Rev B | ESP32-WROOM-32E | NAU8822 | MCP73871 | TPS63020

---

## Files

| File | Description |
|---|---|
| `ohis-ble-bridge.kicad_pro` | KiCad 7 project file — open this |
| `sheet1_power.kicad_sch` | USB-C input, MCP73871 charger+power-path, TPS63020 buck-boost |
| `sheet2_interface_audio.kicad_sch` | OHIS RJ-45, ESD, PTT FET, NAU8822 codec, display, buttons, LEDs |
| `sheet3_esp32.kicad_sch` | ESP32-WROOM-32E, CP2102N USB-UART, auto-reset, decoupling |

---

## Opening in KiCad 7

1. Open KiCad 7.x
2. File → Open Project → select `ohis-ble-bridge.kicad_pro`
3. Open Schematic Editor (eeschema)
4. All three sheets accessible via the Sheets panel

> **Note:** All component symbols are defined **inline** in each schematic's
> `lib_symbols` section. No external symbol library is required.
> KiCad 7.0+ will open these files without library warnings.

---

## Inter-Sheet Global Labels

These nets cross between sheets via global labels:

| Label | Sheet 1 | Sheet 2 | Sheet 3 |
|---|---|---|---|
| `+3V3_TO_ALL` | Source (TPS63020 out) | Sink | Sink |
| `VSYS_RAIL` | Source (MCP73871 out) | — | — |
| `USB_DP` / `USB_DM` | — | — | CP2102N |
| `PTT_GPIO` | — | 2N7002 gate (sink) | ESP32 GPIO33 (source) |
| `I2S_BCLK/LRCLK/ADCDAT/DACDAT` | — | NAU8822 | ESP32 |
| `I2C_SDA/SCL` | — | NAU8822 | ESP32 |
| `DISP_CS/SCK/MOSI/EXTCOMIN` | — | Display header | ESP32 |
| `BTN_SELECT/BACK/UP/DOWN` | — | Tactile buttons | ESP32 GPIOs |
| `LED_BT/PTT/PWR/CHG` | — | LED anodes | ESP32 GPIOs |
| `FL_PWM` | — | BSS138 FET gate | ESP32 GPIO4 |
| `LDR_ADC` | — | GL5528 divider mid | ESP32 GPIO34 |
| `CHRG_PG` | MCP73871 PG | — | ESP32 GPIO35 |

---

## Component Notes

### Sheet 1 — Power
- **MCP73871-2AA/ML**: VIN and VBUS both connect to USB VBUS. CE=HIGH (always charge when USB present). SEL1=SEL2=GND selects USB as power source. STAT1/STAT2 drive LEDs through 1kΩ pull-ups to 3V3.
- **TPS63020**: Feedback for 3.3V — R7=560kΩ (top), R8=100kΩ (bottom). Vout = 0.5 × (1 + 560/100) = 3.3V. EN pin pulled HIGH via 100kΩ to VSYS (always on). PFM/PWM=GND (auto mode).
- **USB-C**: CC1 and CC2 each pulled to GND via 5.1kΩ. This identifies the device as a valid USB-C sink requesting 5V/500mA. No USB-PD required.

### Sheet 2 — Interface & Audio
- **OHIS Grounds**: MicGND (Pin 3), HPGND (Pin 5), PwrGND (Pin 7) must be **separate copper pours** on the PCB, joining only at a single star-ground point near the power supply decoupling. Mixing these grounds causes audio crosstalk.
- **NAU8822**: CSIF pin to GND = I2C mode, address 0x1A. CLKSEL to GND = use MCLK pin. HP output pads (LOUT1) used for mic signal path back to OHIS Pin 6. DAC → pad resistors → 100nF DC block → OHIS Pin 6.
- **PTT FET (Q2 2N7002)**: Gate driven by ESP32 GPIO33 through 10kΩ series resistor. Gate-source 100kΩ pull-down ensures FET is off when GPIO floats (boot). Drain connects to OHIS PTT Pin 2. Source to OHIS PwrGND (Pin 7).
- **Display (J3)**: Sharp LS013B7DH03 — 3-wire SPI (MOSI+SCK+CS). EXTCOMIN must be toggled 1–60Hz continuously. Failure to toggle EXTCOMIN damages the display over time via DC bias.

### Sheet 3 — ESP32
- **ESP32-WROOM-32E**: Pre-certified module (FCC/CE ID on module). PCB antenna — maintain 3mm clearance from any metal, ground planes, or components.
- **Strapping pins at boot**: GPIO0=HIGH (normal), GPIO2=LOW OK (will be HIGH via LED resistor), GPIO5=HIGH (via pull-up, correct for VSPI), GPIO12=LOW (pull-down, correct for 3.3V flash voltage), GPIO15=HIGH (pull-up, correct).
- **CP2102N auto-reset**: BC847BS dual NPN (Q4) coupled via 100nF caps on RTS/DTR. This is the standard ESP32 auto-reset circuit — RTS triggers EN reset, DTR controls GPIO0 (download mode). Compatible with esptool.py automatic upload.
- **Ferrite bead FB1** (BLM21PG300SN1D or similar): Place on 3.3V line feeding ESP32 VDD to reduce RF emissions coupling back into audio supply.

---

## Recommended PCB Stack-up

4-layer board recommended:
- L1: Signal (top) — components, short traces
- L2: Ground plane — continuous pour, no splits except at OHIS star-ground point
- L3: Power plane (+3V3)
- L4: Signal (bottom) — BT antenna keepout zone

**ESP32 antenna keepout**: No copper (any layer) within 3mm of the module's PCB antenna end.

---

## Bill of Materials Summary (LCSC)

| Ref | Part | LCSC | Qty | ~Cost |
|---|---|---|---|---|
| U7 | ESP32-WROOM-32E | C473012 | 1 | $3.50 |
| U5 | NAU8822L | C964789 | 1 | $1.80 |
| U1 | MCP73871-2AA/ML | C92084 | 1 | $1.20 |
| U2 | TPS63020DSJT | C130626 | 1 | $1.50 |
| U8 | CP2102N-A02-GQFN24 | C8286 | 1 | $1.10 |
| U3,U4 | PRTR5V0U2X | C12333 | 2 | $0.50 |
| U6 | 12.288MHz MEMS OSC | C2662765 | 1 | $0.90 |
| Q1 | SI2301 P-MOSFET | C10487 | 1 | $0.08 |
| Q2 | 2N7002 N-MOSFET | C8545 | 1 | $0.05 |
| Q3 | BSS138 N-MOSFET | C52895 | 1 | $0.05 |
| Q4 | BC847BS Dual NPN | C49549 | 1 | $0.08 |
| J2 | 8P8C RJ-45 shielded | C111452 | 1 | $0.40 |
| J3 | 6-pin FFC 0.5mm | C262657 | 1 | $0.15 |
| D1-D7 | SMD LED various | — | 7 | $0.35 |
| SW1-SW5 | 6x6mm tactile | C318884 | 5 | $0.25 |
| RV1 | GL5528 LDR | C202233 | 1 | $0.10 |
| L1 | 2.2µH 3A inductor | C349609 | 1 | $0.20 |
| FB1 | Ferrite bead 300Ω | C1017 | 1 | $0.05 |
| BT1 | JST-PH 2-pin connector | C131337 | 1 | $0.10 |
| R,C misc | 0402 passives | — | ~40 | $1.00 |
| Display | Sharp LS013B7DH03 | (aliexpress/Digi-Key) | 1 | $3.00 |
| Battery | LiPo 3.7V 2000mAh | (external) | 1 | ~$5.00 |
| **Total PCB BOM** | | | | **~$21.36** |

---

*Generated by OHIS BLE Bridge schematic generator. Rev B 2025.*
*OHIS standard: https://ohis.org | ESP-IDF: https://docs.espressif.com*
