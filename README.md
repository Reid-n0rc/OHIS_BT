# OHIS LE Audio Bridge — KiCad 7 Schematics
## Rev C | ESP32-H4 | NAU8822 | MCP73871 | TPS63020

---

## Files

| File | Description |
|---|---|
| `ohis-ble-bridge.kicad_pro` | KiCad 7 project file — open this |
| `sheet1_power.kicad_sch` | USB-C input, MCP73871 charger+power-path, TPS63020 buck-boost |
| `sheet2_interface_audio.kicad_sch` | OHIS RJ-45, ESD, PTT FET, NAU8822 codec, display, buttons, LEDs |
| `sheet3_esp32.kicad_sch` | ESP32-H4 module, native USB-OTG, decoupling |

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
| `USB_DP` / `USB_DM` | — | — | ESP32-H4 USB-OTG (native) |
| `PTT_GPIO` | — | 2N7002 gate (sink) | ESP32-H4 PTT_GPIO (source) |
| `I2S_BCLK/LRCLK/ADCDAT/DACDAT` | — | NAU8822 | ESP32-H4 |
| `I2C_SDA/SCL` | — | NAU8822 | ESP32-H4 |
| `DISP_CS/SCK/MOSI/EXTCOMIN` | — | Display header | ESP32-H4 |
| `BTN_SELECT/BACK/UP/DOWN` | — | Tactile buttons | ESP32-H4 GPIOs |
| `LED_BT/PTT/PWR/CHG` | — | LED anodes | ESP32-H4 GPIOs |
| `FL_PWM` | — | BSS138 FET gate | ESP32-H4 FL_PWM GPIO |
| `LDR_ADC` | — | GL5528 divider mid | ESP32-H4 LDR_ADC GPIO |
| `CHRG_PG` | MCP73871 PG | — | ESP32-H4 CHRG_PG GPIO |

---

## Component Notes

### Sheet 1 — Power
- **MCP73871-2AA/ML**: VIN and VBUS both connect to USB VBUS. CE=HIGH (always charge when USB present). SEL1=SEL2=GND selects USB as power source. STAT1/STAT2 drive LEDs through 1kΩ pull-ups to 3V3.
- **TPS63020**: Feedback for 3.3V — R7=560kΩ (top), R8=100kΩ (bottom). Vout = 0.5 × (1 + 560/100) = 3.3V. EN pin pulled HIGH via 100kΩ to VSYS (always on). PFM/PWM=GND (auto mode).
- **USB-C**: CC1 and CC2 each pulled to GND via 5.1kΩ. This identifies the device as a valid USB-C sink requesting 5V/500mA. No USB-PD required.

### Sheet 2 — Interface & Audio
- **OHIS Grounds**: MicGND (Pin 3), HPGND (Pin 5), PwrGND (Pin 7) must be **separate copper pours** on the PCB, joining only at a single star-ground point near the power supply decoupling. Mixing these grounds causes audio crosstalk.
- **NAU8822**: CSIF pin to GND = I2C mode, address 0x1A. CLKSEL to GND = use MCLK pin. HP output pads (LOUT1) used for mic signal path back to OHIS Pin 6. DAC → pad resistors → 100nF DC block → OHIS Pin 6.
- **PTT FET (Q2 2N7002)**: Gate driven by ESP32-H4 PTT_GPIO through 10kΩ series resistor. Gate-source 100kΩ pull-down ensures FET is off when GPIO floats (boot). Drain connects to OHIS PTT Pin 2. Source to OHIS PwrGND (Pin 7).
- **Display (J3)**: Sharp LS013B7DH03 — 3-wire SPI (MOSI+SCK+CS). EXTCOMIN must be toggled 1–60Hz continuously. Failure to toggle EXTCOMIN damages the display over time via DC bias.

### Sheet 3 — ESP32-H4
- **ESP32-H4**: Dual-core 32-bit RISC-V SoC (up to 96 MHz). Features Bluetooth 5.4 LE with full LE Audio support, IEEE 802.15.4 (Thread/Zigbee/Matter), 320 KB SRAM, 128 KB ROM, up to 40 GPIOs, I2C, I2S, SPI, UART, ADC, native USB-OTG, hardware crypto, secure boot, and flash encryption. Use the ESP32-H4 module variant with an integrated PCB antenna; maintain 3 mm clearance from any metal, ground planes, or components.
- **USB interface**: The ESP32-H4's native USB-OTG port connects directly to the USB-C D+/D− lines (via the `USB_DP`/`USB_DM` global labels). This eliminates the CP2102N USB-UART bridge, Q4 (BC847BS dual NPN auto-reset circuit), and all associated passives from Rev B.
- **Strapping pins at boot (ESP32-H4 / RISC-V)**: GPIO8=HIGH selects SPI flash boot (normal run mode); GPIO9=HIGH enables the internal pull-up on the USB D+ line for USB-OTG device enumeration. Ensure these pins are not driven LOW at power-on unless deliberately entering download/ROM mode. Consult the ESP32-H4 datasheet for the full strapping-pin table as pin assignments may differ between module variants.
- **Ferrite bead FB1** (BLM21PG300SN1D or similar): Place on the 3.3 V line feeding ESP32-H4 VDD to reduce RF emissions coupling back into the audio supply.

---

## LE Audio & Hearing Aid Streaming

The ESP32-H4 provides native Bluetooth 5.4 LE support with a full LE Audio stack, making it the ideal SoC for modern hearing aid connectivity.

### LC3 Codec
LE Audio mandates the **LC3 (Low Complexity Communication Codec)** as its baseline codec, replacing SBC. LC3 delivers better audio quality at lower bitrates (as low as 16 kbps per channel), which is critical for hearing aids where power consumption and latency are tightly constrained. The LC3 encoder/decoder runs in software (or optionally in hardware-accelerated form) on the ESP32-H4.

### LE Isochronous Channels (ISO)
LE Audio uses isochronous transport to guarantee deterministic latency:

- **CIS (Connected Isochronous Streams)** — unicast point-to-point streaming from the bridge to one or both hearing aids. Used for direct, private audio streaming with the Hearing Aid Profile (HAP).
- **BIS (Broadcast Isochronous Streams)** — Auracast™ broadcast audio. The bridge transmits a BIS that any compatible hearing aid (or other LE Audio receiver) in range can join without pairing. Ideal for public-venue or assistive-listening scenarios.

### Hearing Aid Profile (HAP) & Hearing Access Service (HAS)
The **Hearing Aid Profile (HAP)** and its associated **Hearing Access Service (HAS)** are the Bluetooth SIG standards for LE Audio hearing aids. HAP defines the roles (Hearing Aid, Hearing Aid Unicast Client), the audio stream configuration, and volume/preset control via HAS. The ESP32-H4 firmware implements the Hearing Aid Unicast Client role, initiating CIS streams to paired hearing aids.

### ASHA Backward Compatibility
**ASHA (Audio Streaming for Hearing Aids)** is Google's Android-specific BLE 4.x protocol used by first-generation BLE hearing aids. It is not part of the Bluetooth SIG standard and is being superseded by LE Audio / HAP. The firmware can support both LE Audio/HAP (primary) and ASHA (fallback) simultaneously to maintain backward compatibility with older hearing aids that have not yet been updated to LE Audio.

### Audio Signal Path
```
OHIS RJ-45 ──► NAU8822 ADC ──► I2S ──► ESP32-H4 ──► LC3 Encode ──► BLE 5.4 LE Audio (CIS/BIS) ──► Hearing Aid

Hearing Aid ──► BLE 5.4 LE Audio ──► ESP32-H4 ──► LC3 Decode ──► I2S ──► NAU8822 DAC ──► OHIS RJ-45
```

The **NAU8822L** codec is retained for analog ↔ I2S conversion on the OHIS wired side (microphone input, receiver output). The **LC3 codec** runs entirely on the ESP32-H4 and is transparent to the NAU8822. No hardware codec changes are required.

---

## Recommended PCB Stack-up

4-layer board recommended:
- L1: Signal (top) — components, short traces
- L2: Ground plane — continuous pour, no splits except at OHIS star-ground point
- L3: Power plane (+3V3)
- L4: Signal (bottom) — BT antenna keepout zone

**ESP32-H4 antenna keepout**: No copper (any layer) within 3 mm of the module's PCB antenna end. The ESP32-H4 module footprint may differ from the WROOM-32E; verify the exact keepout boundary against the module datasheet.

---

## Bill of Materials Summary (LCSC)

| Ref | Part | LCSC | Qty | ~Cost |
|---|---|---|---|---|
| U7 | ESP32-H4 module | TBD | 1 | $3.00 |
| U5 | NAU8822L | C964789 | 1 | $1.80 |
| U1 | MCP73871-2AA/ML | C92084 | 1 | $1.20 |
| U2 | TPS63020DSJT | C130626 | 1 | $1.50 |
| U3,U4 | PRTR5V0U2X | C12333 | 2 | $0.50 |
| U6 | 12.288MHz MEMS OSC | C2662765 | 1 | $0.90 |
| Q1 | SI2301 P-MOSFET | C10487 | 1 | $0.08 |
| Q2 | 2N7002 N-MOSFET | C8545 | 1 | $0.05 |
| Q3 | BSS138 N-MOSFET | C52895 | 1 | $0.05 |
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
| **Total PCB BOM** | | | | **~$19.68** |

---

*Generated by OHIS LE Audio Bridge schematic generator. Rev C 2026. LE Audio / HAP / Auracast™ ready.*
*OHIS standard: https://ohis.org | ESP-IDF: https://docs.espressif.com | Bluetooth LE Audio: https://www.bluetooth.com/learn-about-bluetooth/recent-enhancements/le-audio/*
