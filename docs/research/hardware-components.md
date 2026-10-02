# Hardware component research: upi-soundbox main board

- **Date:** 2026-10-02
- **Status:** paper design only. Nothing here has been built or measured.
- **Scope:** part choices for the custom KiCad board (PCB Power fabrication, 3D-printed case, Wi-Fi only, USB-C power).
- **Method:** every rating below is taken from a manufacturer datasheet or a standard, with the page and table cited. Where a datasheet gives only a typical value, the text says **typ** and the value is not used as a guarantee. Anything not confirmed from a primary source is marked **UNVERIFIED**.
- **Second check (2026-10-02).** These ratings were re-read in the primary datasheets and confirmed:
  - TPS259531: 5.7 V clamp variant with auto-retry; VOVC 5.5–5.9 V; VCLAMP 5.2–5.7 V; IN 20 V and EN 7 V absolute maximum.
  - TPS62162: ILIMF 1.45–2.45 A.
  - Murata DFE252012F-2R2M=P2: 82 mΩ max, 3.3 A rated, 2.3 A thermal.
  - NHD-2.8 backlight table.
  - B3W: 1–50 mA rating.
  - ESP32-S3-WROOM-1: R8/R16V variants rated −40 to 65 °C, N16 rated −40 to 85 °C.

  The review changed one design: the backlight drive (Section 2.5).
- **Owner rules applied:** reliability and safety first; then fewest parts; then India stock; then cost; then size. Guaranteed (min/max) limits only. No speculative DNP parts. Protection against firmware failure stays in hardware.

---

## 0. Summary

### 0.1 Chosen parts

| Function | Part | Why, in one line |
|---|---|---|
| MCU | Espressif **ESP32-S3-WROOM-1-N16** (16 MB quad flash, no PSRAM, −40 to 85 °C) | Only candidate with enough GPIO. Has hardware AES/SHA/RSA/HMAC/DS, Secure Boot v2, XTS-AES-256 flash encryption and native USB. |
| Display | Newhaven **NHD-2.8-240320AF-CSXP-F** (2.8 in IPS TFT, 240×320, ST7789Vi, 4-wire SPI, 1000 cd/m² typ, −20 to 70 °C) | Bare FPC panel with a real datasheet. 0.72 mm QR modules up to QR version 8. |
| Display connector | Molex **54132-4062** (40-pin, 0.5 mm, FFC/FPC ZIF) | The connector Newhaven names for this panel. |
| Backlight switch | AOS **AO3400A** N-MOSFET + **4 × 130 Ω 1206** ballast resistors, one per LED cathode, from 5 V (revised in review, see 2.5) | RDS(on) ≤ 48 mΩ guaranteed at VGS = 2.5 V, so a 3.3 V GPIO drives it fully. One resistor per LED forces current sharing. |
| Audio amp | Analog Devices **MAX98357AETE+T** (I2S in, filterless class D, TQFN-16) | No MCLK and no I2C setup. Internal SD_MODE pull-down gives a hardware fail-off. |
| Speaker | PUI Audio **AS06608PS-R** (66 mm, 8 Ω ±15 %, 4 W rated / 5 W max, 95 ±3 dBA at 1 W / 0.5 m) | Loud enough for a shop, and it survives the amplifier's worst-case DC output (see 3.2). |
| Speaker connector | JST **B2B-XH-A(LF)(SN)** + XHP-2 housing | Polarised, latching, and lets the case come apart without desoldering. |
| Keypad | 16 × Omron/Aratas **B3W-4150** (12 mm, IP67, ground terminal, 3 000 000 ops min) + **B32-1310** key tops, 4×4 matrix, with 4 × 2.7 kΩ column pull-ups | Switches are sealed and have real datasheets. The Grayhill sealed keypad was rejected: ₹8 246, hex legends, only IP42 (Section 4). |
| USB-C receptacle | GCT **USB4105-GF-A** (USB 2.0, 16-pin, 5 A VBUS, 20 000 cycles) | Through-hole shell stakes for strength, with a full datasheet. |
| Input protection | TI **TPS259531DSGR** eFuse (5.7 V overvoltage clamp, auto-retry, 20 V abs max, current limit set by one resistor) | One part does over-current, over-voltage, inrush and thermal protection. |
| 3.3 V regulator | TI **TPS62162DSGR** buck (fixed 3.3 V, 1 A, VIN 3–17 V) + Murata **DFE252012F-2R2M=P2** 2.2 µH (Isat 3.3 A rated, DCR 82 mΩ max) | Low heat. Survives an overvoltage fault, because VIN is rated to 17 V. The inductor does not saturate even at the buck's current limit. |
| Reset supervisor | TI **TLV809EA29DBZR** (2.93 V, push-pull active-low, 200 ms, SOT-23-3; Active in TI package addendum) | Espressif asks for a power monitor when power cycles often (power cuts). |
| ESD | TI **TPD4E05U06DQAR** ×3 (USB D+/D−/CC1/CC2; keypad rows; keypad columns) | One part number reused. IEC 61000-4-2 ±12 kV contact. |

Every part in this table is in stock at DigiKey India on 2026-10-02 (Section 9). The panel, speaker and switches have no second source found.

### 0.2 Key numbers

- **QR code.**
  - A Razorpay-style string of 123–161 bytes encodes as QR version 6–8 at error-correction level L, or version 8–9 at level M.
  - On the 2.8 in panel, versions up to 8 get **4 px per module = 0.72 mm**.
  - That gives about 2.1 camera pixels per module at 50 cm (1080p stream) and 3.5 pixels at 30 cm. ML Kit needs at least 2 pixels per module.
  - Versions 9–13 drop to 3 px = 0.54 mm, which works at about 30 cm only.
- **Power budget, worst case.**
  - VBUS ≤ 0.97 A in normal use (firmware-limited audio).
  - ≤ 1.40 A in the worst firmware-fault case (full-scale square wave or DC into the speaker).
  - Both are below 1.5 A.
- **Adapter.** The adapter must advertise at least 1.5 A on CC (Rp 22 kΩ). A 5 V / 3 A USB-C adapter with a C-to-C cable is recommended. A legacy A-to-C cable advertises only "Default" power (500 mA), which this device cannot stay within.
- **Flash.** English plus Hindi number words at 16 kHz / 16-bit is about 120 s, or about 3.8 MB of PCM. A/B audio partitions plus two 3 MB app slots fit in 16 MB.
- **Speaker level.** About 89 ±3 dBA at 1 m per watt. The amplifier gives about 1.4 W **typ** into 8 Ω at 5 V (1 % THD+N), which is enough over typical shop noise.

### 0.3 Owner decisions (resolved 2026-10-02)

Each of these was an open question in the first draft. The outcomes are recorded in [`../decisions.md`](../decisions.md).

| # | Question | Outcome | Decision |
|---|---|---|---|
| 1 | Battery backup | **No battery.** Run from any 5 V USB-C source: a power bank, or a router mini-UPS that also keeps the Wi-Fi up (Section 6). | D-007 |
| 2 | Behaviour on a "Default USB power" (500 mA) source | **Run limited and warn:** backlight dimmed, volume capped, message "Use a 5 V USB-C adapter for full volume" (Section 5.7). | D-008 |
| 3 | Keypad spill path through 16 key holes | **Handled in the case design:** raised key island, drip gutter, tight cap-to-hole clearance. Conformal coating of the keypad zone is decided at case design. The Grayhill sealed overlay stays rejected: IP42, hex legends, ₹8 246. | D-016 |
| 4 | Speaker size | **66 mm AS06608PS-R.** It is the only option that survives the amplifier's 4.45 W worst-case fault; the 40 mm alternative is rated 4 W max. | D-010 |
| 5 | Keypad ESD arrays | **Kept** (2 × TPD4E05U06). Omron gives no IEC rating for the switch ground terminal. | D-011 |
| 6 | External watchdog | **None.** The ESP32-S3's on-chip RWDT and MWDTs count as the hardware watchdog; firmware must never disable them. | D-009 |

---

## 1. MCU module

### 1.1 Requirements and GPIO count

| Need | Pins |
|---|---|
| Native USB D−/D+ | 2 (fixed: GPIO19/20 on S3) |
| SPI display: SCLK, MOSI, CS, D/C, RESET (no MISO needed) | 5 |
| Backlight PWM | 1 |
| I2S: BCLK, LRCLK, DOUT | 3 |
| Amp SD_MODE (enable) | 1 |
| 4×4 keypad matrix | 8 |
| USB-C CC1/CC2 voltage sense (ADC), needed to draw more than 500 mA legally (Section 5.2) | 2 |
| **Total** | **22**, plus UART0 TX/RX to test points |

### 1.2 Comparison

All three have Wi-Fi 4 at 2.4 GHz, Secure Boot v2, flash encryption and a USB Serial/JTAG controller, so none needs a USB-UART bridge.

| | ESP32-S3-WROOM-1 | ESP32-C3-MINI-1 / C3-WROOM-02 | ESP32-C6-WROOM-1 |
|---|---|---|---|
| CPU / SRAM | 2× Xtensa LX7 240 MHz / 512 KB | 1× RISC-V 160 MHz / 400 KB | 1× RISC-V 160 MHz (+ LP core) / 512 KB |
| GPIO on module | 36 (IO0–21, IO35–48) | **15** (datasheet cover, both modules) | **23** (datasheet cover); 5 are strapping pins |
| Fits the 22-pin need? | **Yes**, with margin | **No** | Only by using strapping pins and UART0. No margin. |
| Crypto accelerators | AES-128/256, SHA, RSA, HMAC, RSA Digital Signature (S3 datasheet v2.2 §4.1.4, p.47–49) | AES, SHA, RSA, HMAC, DS (C3 datasheet v2.4 §4.1.4) | AES, SHA, RSA, **ECC**, HMAC, RSA-DS (C6 datasheet v1.5 §4.1.4) |
| Secure Boot v2 | RSA-PSS (S3 datasheet §4.1.4.4, p.48) | RSA-3072 | RSA-3072 **or** ECDSA-P256 (ESP-IDF C6 Secure Boot v2 docs) |
| Flash encryption | XTS-AES-128 **or -256** (ESP-IDF S3 flash-encryption docs) | XTS-AES-128 (not re-checked this session) | XTS-AES-128 only (ESP-IDF C6 docs) |
| USB | USB-OTG FS + USB Serial/JTAG | USB Serial/JTAG | USB Serial/JTAG |
| Flash options | 4/8/16 MB | 4 MB (MINI-1 N4X, H4X), 8 MB (H8X) | 4/8/16 MB |
| Ambient temp | −40–85 °C (N4/N8/N16); **−40–65 °C for R8/R16V** (octal PSRAM) (WROOM-1 v1.8, Table 1-1, p.3) | −40–85 / −40–105 °C | −40–85 °C |
| Wi-Fi TX, 802.11b 1 Mbps | 355 mA "Peak" at 20.5 dBm (Table 6-4, p.28) | 350 mA (MINI-1 v2.2, Table 6-4, p.22) | 382 mA (WROOM-1 v1.4, Table 6-4, p.27) |
| Supply requirement (guaranteed design input) | IVDD ≥ 0.5 A, VDD 3.0–3.6 V (Table 6-2, p.27) | same | same |

**Decision: ESP32-S3-WROOM-1.**

- The C3 fails on GPIO count.
- The C6 has nicer crypto (ECC accelerator, ECDSA secure boot) but no pin margin, and 382 mA TX.
- The S3's extra CPU is not needed, but it is not a cost either. TLS handshakes with RSA are hardware-assisted, and ECDHE runs in software at 240 MHz.

> Espressif's Table 6-4 numbers are labelled "Peak" with no min/max column, measured at 25 °C, 3.3 V and 100 % TX duty. They are not a guaranteed maximum. The design input is therefore Table 6-2: the supply must deliver **≥ 0.5 A**.

### 1.3 Flash size: dual OTA plus audio clips

Clip inventory, using Indian numbering (lakh/crore):

| Set | Clips | Average length (estimate) | Seconds |
|---|---|---|---|
| English: one–nineteen, twenty–ninety, hundred, thousand, lakh, crore, rupees, paise, and | ~35 | 0.5 s | 17.5 |
| Hindi: 1–99 (each is an irregular word: ek … ninyānave), sau, hazār, lākh, karoṛ, rupaye, paise | ~105 | 0.55 s | 58 |
| Phrases in both languages: "payment received", "payment failed", "connect power adapter", Wi-Fi status, … (~10 per language) | ~20 | 2 s | 40 |
| **Total** | | | **≈ 115–120 s** |

- **Sample rate: 16 kHz mono.**
  - This is wideband speech, up to 8 kHz of bandwidth.
  - It is supported by the MAX98357A (8–96 kHz, datasheet p.1).
  - It covers the speaker's 230 Hz–12 kHz range.
  - 8 kHz would be telephone quality, which is worse for intelligibility in noise.
- **Encoding options:**
  - PCM16: 32 KB/s × 120 s ≈ **3.8 MB**.
  - IMA-ADPCM (4-bit): 8 KB/s ≈ **0.96 MB**. Decode is trivial, but it adds some hiss.

Proposed partition table for **16 MB**:

| Partition | Size |
|---|---|
| bootloader + partition table | 0.1 MB |
| nvs + nvs_keys + otadata + phy_init | 0.1 MB |
| ota_0 / ota_1 (app with Wi-Fi + TLS + display + QR) | 3 MB + 3 MB |
| audio_a / audio_b (A/B voice packs, PCM16) | 4 MB + 4 MB |
| coredump | 64 KB |
| **Total** | **≈ 14.3 MB of 16 MB** |

- **N16** holds PCM16 audio in A/B slots (so a voice-pack update is atomic) and leaves room for one more regional language. The cost difference over N8 is small.
- **N8** works only with ADPCM audio.

**PSRAM: not needed.**

- The display is driven straight over SPI with partial buffers. A 240×32 RGB565 strip is 15 KB.
- A QR bitmap at version 10 is under 1 KB.
- mbedTLS needs about 40–60 KB per session.
- Audio streams from flash.
- 512 KB of SRAM is enough.
- Leaving out PSRAM also avoids the octal-PSRAM 65 °C ambient limit and frees GPIO35–37.

### 1.4 Choice

- **Primary: ESP32-S3-WROOM-1-N16.**
  - Datasheet: https://www.espressif.com/sites/default/files/documentation/esp32-s3-wroom-1_wroom-1u_datasheet_en.pdf (v1.8).
  - Ratings relied on:
    - VDD 3.0–3.6 V and IVDD ≥ 0.5 A (Table 6-2, p.27).
    - VDD abs max 3.6 V (Table 6-1, p.27).
    - Ambient −40–85 °C (Table 1-1, p.3).
    - VOH ≥ 0.8 × VDD = 2.64 V and VOL ≤ 0.1 × VDD (Table 6-3, p.27).
    - ADC after calibration: ±10 mV in ATTEN2 over 0–1600 mV (chip datasheet Table 5-6, p.66, measured at 25 °C with Wi-Fi off).
  - The internal pull-ups are 45 kΩ **typ only** (Table 6-3), so nothing relies on them. The keypad has external 2.7 kΩ pull-ups (Section 4).
- **Alternatives:**
  - **ESP32-S3-WROOM-1-N8**, with ADPCM audio.
  - **ESP32-S3-WROOM-1-N16R8** with PSRAM ECC enabled, which restores the 85 °C rating (Table 1-1 note, p.3). Use it only if N16 cannot be bought, because it costs 3 GPIOs.
- **Antenna placement:**
  - Put the module at the board edge with the antenna over or beyond the edge, and keep copper out of the keepout zone.
  - Keep **≥ 15 mm clearance in all directions** inside the housing (ESP32-S3 HW Design Guidelines §1.4, p.30).
  - The speaker magnet and the display frame must be outside this zone.

### 1.5 Draft GPIO map (for the schematic pass)

| Function | GPIO | Reason |
|---|---|---|
| USB D− / D+ | 19 / 20 | Fixed |
| CC1 / CC2 sense | 1 / 2 (ADC1_CH0/1) | ADC1 works while Wi-Fi is active |
| LCD SCLK, MOSI, CS, D/C, RESET | 12, 11, 10, 13, 14 | Low-level 60 µs power-up glitch is harmless here (HW guide Table 10, p.17) |
| Backlight gate | 38 | No power-up glitch, no pull at reset |
| I2S BCLK, LRCLK, DOUT | 4, 5, 6 | |
| Amp SD_MODE | 39 | No glitch, no pull at reset, so the amp's internal 100 kΩ pull-down holds it off |
| Keypad rows (outputs) / columns (inputs, external 2.7 kΩ pull-up) | 7, 8, 9, 15 / 16, 17, 21, 40 | GPIO18 avoided (it has a high-level power-up glitch) |
| UART0 TX / RX | 43 / 44 | Test points |
| Strapping: GPIO0, 3, 45, 46 | Left at defaults; GPIO0 goes to a test pad | GPIO45 must stay low for 3.3 V flash (internal pull-down) |

Spare pins: 35, 36, 37, 41, 42, 47, 48.

---

## 2. Display

### 2.1 QR size: version and module count

QR versions were computed with the `segno` encoder in byte mode (UPI URIs contain `?`, `&`, `=`, `@` and lower case, so byte mode is required):

| Payload | Bytes | EC L | EC M | EC Q |
|---|---|---|---|---|
| Minimal UPI URI | 91 | V5 (37×37) | V6 (41) | V8 (49) |
| Razorpay doc example (`image_content`, see payment-gateways.md) | 123 | **V6 (41)** | **V8 (49)** | V9 (53) |
| Razorpay-style with a long shop name and paise | 161 | **V8 (49)** | V9 (53) | V11 (61) |
| Synthetic | 150 | V7 (45) | V8 (49) | V10 (57) |
| Synthetic | 200 | V9 (53) | V10 (57) | V12 (65) |
| Synthetic | 250 | V10 (57) | V11 (61) | V14 (73) |
| PhonePe with GST/invoice fields (per gateway research) | >300 | ≥ V11 | ≥ V12 | — |

- Module count = 17 + 4 × version.
- ISO/IEC 18004 needs a 4-module quiet zone on each side, so the white square is modules + 8.

**Error-correction level.**

- A clean screen needs less redundancy than a printed sticker.
- Firmware should choose the **highest EC level that keeps the version ≤ 8**, which segno's `boost_error` does automatically.
- No NPCI rule on EC level was found in a primary source. Blog claims that "UPI QR uses level H" refer to codes with logos. Treat as **UNVERIFIED**.

### 2.2 How big a module the phone needs at 30–50 cm

- **Decoder requirement (primary source).** Google ML Kit (used by many UPI apps) says: "the smallest meaningful unit of the barcode should be at least 2 pixels wide, and for 2-dimensional codes, 2 pixels tall". It recommends 1280×720 or 1920×1080 input "to scan from a larger distance". https://developers.google.com/ml-kit/vision/barcode-scanning/android
- **Camera geometry (assumption, UNVERIFIED for any specific phone).** A 26 mm-equivalent main camera gives a horizontal FOV of about 67°. The field width at distance d is 1.33 × d.

| Distance | Stream | mm per camera pixel | Module for 2 px (minimum) | Module for 3 px (margin) |
|---|---|---|---|---|
| 30 cm | 1280 px | 0.31 | 0.62 mm | 0.94 mm |
| 30 cm | 1920 px | 0.21 | 0.42 mm | 0.62 mm |
| 50 cm | 1280 px | 0.52 | 1.04 mm | 1.56 mm |
| 50 cm | 1920 px | 0.35 | 0.69 mm | 1.04 mm |

### 2.3 Physical module size per panel

Each panel can show the QR at an integer number of pixels per module = floor(short side px / (modules + 8)).

| QR version | 2.4 in 240×320 (pitch 0.153) | **2.8 in 240×320 NHD (pitch 0.180)** | 3.2 in 240×320 (0.2025) | 3.5 in 320×480 (0.153) | 4.2 in e-paper 400×300 (0.212) |
|---|---|---|---|---|---|
| V6 | 4 px = 0.61 mm | **4 px = 0.72 mm** | 0.81 mm | 6 px = 0.92 mm | 1.27 mm |
| V8 | 4 px = 0.61 mm | **4 px = 0.72 mm** | 0.81 mm | 5 px = 0.77 mm | 1.06 mm |
| V9 | 3 px = 0.46 mm | 3 px = 0.54 mm | 0.61 mm | 5 px = 0.77 mm | 0.85 mm |
| V10–V12 | 0.46 mm | 0.54 mm | 0.61 mm | 4 px = 0.61 mm | 0.85 mm |

**Result.** On the 2.8 in NHD panel, with QR ≤ V8 (≤ 152 bytes at M, ≤ 192 bytes at L):

- 0.72 mm modules give **2.1 camera px/module at 50 cm on 1080p** and **3.5 px at 30 cm**. At 720p they give 2.3 px at 30 cm but only 1.4 px at 50 cm.
- The QR is about 35 mm square, plus the quiet zone.
- At V9 and above, modules fall to 0.54 mm, which is good to about 30 cm on 1080p.

No SPI panel up to 3.5 in meets 2 px at 50 cm on a **720p** stream for V8 and above; that needs modules of at least 1.04 mm. The 30–50 cm target therefore holds for 1080p scanners and shorter payloads.

**Requirement passed to the backend:** keep the QR payload ≤ 192 bytes and log its length (the gateway research already asks for this). Razorpay's documented sample is 123 bytes.

### 2.4 Technology comparison

| | Bare FPC TFT panel (chosen) | Breakout TFT module (e.g. generic ILI9341/ST7796 red-PCB) | E-paper (e.g. 4.2 in 400×300) |
|---|---|---|---|
| Datasheet quality | Full electrical and optical tables with min/typ/max (Newhaven) | Usually only a controller datasheet. The panel, backlight and regulator on the module are unspecified, and controllers vary by batch. | Good-to-fair |
| Parts on our board | FPC connector + backlight MOSFET + resistor + decoupling | 1 module + header. The module brings an unneeded LDO, touch controller and SD slot. | Bare panel: about 10–15 parts of booster circuit. Module: 1 + header. |
| Refresh | Instant (response 30 ms typ, 40 ms max: NHD p.6) | Instant | **Full refresh 5 s.** Maker recommends **≥ 180 s between refreshes** (Waveshare 4.2 in manual). This fails a per-order QR plus live keypad echo. |
| Readability | IPS, 1000 cd/m² typ (800 min), anti-glare, 80° viewing angles | Often TN, 200–300 cd/m², glossy | Excellent in daylight, unreadable in the dark |
| Operating temperature | −20–70 °C (NHD p.6) | Unspecified | **0–50 °C**, marginal for a hot shop |

**Decision: bare FPC TFT, Newhaven NHD-2.8-240320AF-CSXP-F.**

- Datasheet (rev 10, 2026-03-11): https://newhavendisplay.com/content/specs/NHD-2.8-240320AF-CSXP-F.pdf
- Ratings relied on (Electrical and Optical Characteristics, p.6; mechanical drawing p.3):
  - VDD 2.4–3.6 V; IDD 15 mA max at 3.3 V.
  - Logic VIH ≥ 0.7 × VDDI.
  - Operating −20 to +70 °C.
  - Luminance ≥ 800 cd/m² at 160 mA.
  - Contrast ≥ 600.
  - Backlight VLED **2.8–3.4 V at 160 mA**; lifetime ≥ 20 000 h to half brightness at 160 mA and 25 °C.
  - Active area 43.20 × 57.60 mm (pitch 0.180 mm); outline 50.00 × 69.20 × 2.39 mm.
  - FPC 0.30 ±0.03 mm, 40 pins at 0.5 mm.
- Interface: 4-wire SPI is selected by strapping IM0 = 0, IM1 = 1, IM2 = 1 on the PCB, with no parts (p.4).
- **Datasheet inconsistency:** the pin table says "LED-A 100 mA @ 3.1 V", while the electrical table and the drawing say 160 mA. No maximum backlight current is given. The design stays well below 160 mA (2.5). Ask Newhaven for the maximum (**UNVERIFIED**).
- **Alternative:** NHD-2.4-240320AF-CSXP (2.4 in, same controller, 0.61 mm modules at V8, about $14 list).

### 2.5 Backlight drive

The four LEDs share an anode (LED-A) and have separate cathodes (K1–K4). Vf can reach 3.4 V, so they cannot run from 3.3 V.

> **Revised in review (2026-10-02).** The first draft tied K1–K4 together behind a single 22 Ω resistor. That had two problems:
> - **No current sharing.** Four LEDs in parallel with no resistor of their own share current only by luck. The LED with the lowest Vf takes the most and ages first.
> - **Above the lower figure.** The datasheet says "drive voltage must be selected to ensure backlight current drain is below MAX level stated", but it states no maximum. Its two figures disagree: 160 mA (electrical table, a **typ** column) and 100 mA (pin table). Under the guaranteed-limits rule the design must respect the **lower** figure until Newhaven confirms otherwise, and 22 Ω allowed up to 123 mA.
>
> The fix costs three resistors. Reliability outranks part count.

Circuit: **5 V → LED-A**. Each cathode K1–K4 goes through its own **130 Ω 1 % 1206** resistor to a common node, which is the **AO3400A** drain; source to GND. The gate is driven by GPIO38 (PWM), with a **100 kΩ pull-down** so the backlight is off while the MCU is in reset.

Per-LED current = (rail − Vf) / 130 Ω. The datasheet's Vf of 2.8–3.4 V is specified at 40 mA per LED (160 mA ÷ 4). At the lower drive used here Vf is somewhat lower, so the max corner assumes **2.6 V** (an estimate, **UNVERIFIED**).

| Corner | Rail | Vf per LED | Per LED | Total |
|---|---|---|---|---|
| Max (eFuse clamp ceiling) | 5.7 V | 2.6 V | 23.8 mA | **95 mA**, below the conservative 100 mA figure |
| Max (USB spec ceiling) | 5.5 V | 2.6 V | 22.3 mA | 89 mA |
| Nominal | 5.0 V | 3.0 V | 15.4 mA | 62 mA, about 390 cd/m² by linear estimate from 1000 cd/m² typ at 160 mA |
| Min | 4.0 V | 3.4 V | 4.6 mA | 18 mA. Dim but readable; this is the brown corner of the Type-C range. |

- About 390 cd/m² is normal indoor-display brightness.
- **If Newhaven confirms 160 mA as a maximum,** only the resistor value changes. About 68 Ω would roughly double the brightness, on the same footprint, so the layout does not wait on the answer.
- Resistor power at the max corner: 23.8² × 130 = 74 mW in a 250 mW 1206, about 30 % loading. This also covers a firmware fault that leaves PWM at 100 %.
- AO3400A ratings (AOS datasheet Rev 3.1, p.2):
  - VGS(th) 0.65–1.45 V.
  - RDS(on) ≤ 48 mΩ at VGS = 2.5 V, which is below the GPIO's guaranteed VOH of 2.64 V.
  - VDS 30 V.
- Connector: **Molex 54132-4062**, as named on the NHD drawing (p.3). DigiKey lists it as **bottom contact**, for 0.30 mm FFC, with a slider (Easy-On) actuator. The NHD FPC is 0.30 ±0.03 mm.
- **FPC orientation must be fixed in the mechanical layout.**
  - The NHD side view marks the FPC "contact side" on the panel's **back** face, with the stiffener on the front.
  - A bottom-contact connector on the PCB's top side needs the FPC contacts facing the PCB.
  - So route the tail so that its back face meets the PCB. A single 180° fold under the panel would flip it and need a top-contact connector.
  - This can be closed on paper when the case and FPC path are drawn.

---

## 3. Audio

### 3.1 Amplifier: MAX98357AETE+T

- Datasheet: https://www.analog.com/media/en/technical-documentation/data-sheets/MAX98357A-MAX98357B.pdf
- analog.com blocked automated download. The copy read here is **Rev 7 (2/16)** via https://cdn-shop.adafruit.com/product-files/3006/MAX98357A-MAX98357B.pdf. Re-check the current revision before layout.

**Guaranteed (min/max) values relied on:**

| Parameter | Value | Source |
|---|---|---|
| VDD operating | 2.5–5.5 V ("guaranteed by PSRR test") | EC table p.4 |
| VDD abs max | 6 V | Abs max p.4 |
| Continuous current in/out of VDD/GND/OUT | ±1.6 A | Abs max p.4 |
| Output short to GND/VDD/each other | continuous | Abs max p.4 |
| Operating temperature | −40 to +85 °C | Abs max p.4 |
| Quiescent current | 3.35 mA max (25 °C) | EC table p.4 |
| Gain, GAIN_SLOT = GND | 11.4–12.6 dB (12 dB nominal) | EC table p.5 |
| SD_MODE internal pull-down | 92–108 kΩ | EC table p.7 |
| SD_MODE trip points | B0 0.08–0.355 V; B1 0.65–0.825 V; B2 1.245–1.5 V | EC table p.7 |
| θJA, TQFN on a 4-layer JEDEC board | 48 °C/W | p.4 |

The datasheet gives **output power as typical only, with no minimum**. All of these are at VDD = 5 V and gain 12 dB, measured with an inductive dummy load (p.5):

| Load | THD+N 1 % | THD+N 10 % |
|---|---|---|
| 4 Ω + 33 µH | 2.5 W **typ** | 3.2 W **typ** |
| 8 Ω + 68 µH | 1.4 W **typ** | 1.8 W **typ** |

- Efficiency 92 % **typ** (8 Ω, 10 % THD).
- THD+N 0.02 % at 1 W **typ**.
- The current limit of 2.8 A is **typ** (p.5). It is not a usable guarantee.
- The power is in practice set by the supply rail. The ideal sine limit is VDD² / 2R = 1.56 W into 8 Ω at 5 V.

**Supply current used in the budget (bounds, not typicals):**

- **Firmware-limited speech:** peak below clipping, about 4.5 V into 6.8 Ω (8 Ω −15 %) gives 1.49 W sine, which with ≥ 85 % efficiency is about **0.35 A at 5 V**.
- **Fault bound:** a full-scale square wave or DC across 6.8 Ω at 5.5 V gives 4.45 W, which is about **0.9 A**.
  - The datasheet warns: "Removing LRCLK while BCLK is present can cause … a large DC output voltage" (p.16).
  - Peak output current 5.7 V / 6.8 Ω = 0.84 A, below the ±1.6 A abs max. A **4 Ω speaker would exceed** it (5.7 / 3.4 = 1.68 A), which is one more reason to use 8 Ω.

**Fail-safe behaviour (hardware):**

- SD_MODE has an internal 100 kΩ pull-down. If the MCU is in reset (GPIO high-impedance), the amp is in shutdown, with outputs high-impedance (p.16–17).
- If BCLK stops, the amp enters standby automatically, also with outputs high-impedance (p.16).
- SD_MODE is driven from GPIO39 through **2.2 kΩ**:
  - The datasheet recommends about 2 kΩ when VDDIO can exceed VDD (p.17). That can happen briefly at unplug, when 3.3 V back-feeds through the buck's high-side body diode.
  - High level: 2.64 V × 92 k / 94.2 k = 2.58 V. This is above B2 max 1.5 V, so it selects left-channel mode. Firmware sends mono in the left slot.
- GAIN_SLOT tied to **GND (12 dB)**. A floating pin is avoided. Firmware caps the digital level so the output does not clip.

**Filtering / EMI:**

- No output filter is needed. The datasheet shows EN55022B-class emissions with **12 in (30 cm) of speaker cable and no filter** (Figure 14, p.28).
- It requires a speaker inductance > 10 µH (p.33).
- Keep the speaker leads short (< 15 cm) and twisted.
- VDD bypass: 0.1 µF + 10 µF at the pin (p.33).

**Thermal.** At 1.5 W out and 85 % efficiency, about 0.26 W is dissipated, so ΔT ≈ 13 °C (θJA 48 °C/W). The fault bound (about 0.45 W) gives ≈ 22 °C. Both are far below the 150 °C junction limit.

**Alternative:** MAX98360A (ADI successor family). Pinout and ordering code are **UNVERIFIED**.

### 3.2 Speaker: PUI Audio AS06608PS-R

- Datasheet: https://puiaudio.com/file/specs-AS06608PS-R.pdf
- Ratings (single-page drawing, rev D 2020):
  - Rated input 4 W, max input 5 W.
  - Impedance 8 Ω ±15 %.
  - SPL **95 ±3 dBA at 1 W / 0.5 m** (average of 1.0, 1.4, 1.7 and 2.0 kHz).
  - Distortion ≤ 5 %; f0 230 Hz ±20 %; range 230 Hz–12 kHz.
  - **Operating −20 to +50 °C.**
  - 66.8 × 66.8 × 26.5 mm, 204 g.
- **Level at the counter:**
  - 1 W at 1 m: 95 − 6 = **89 ±3 dBA** (inverse-square from 0.5 m; free field).
  - At about 1.4 W **typ** from the amp: ≈ 90.5 dBA (worst speaker tolerance 87.5 dBA).
  - Typical busy-shop or street noise of 65–75 dBA (**UNVERIFIED** generic figure) leaves +12–20 dB of speech-to-noise margin.
- **Survives the amplifier's worst case:** the DC or square-wave fault puts ≤ 5.5² / 6.8 = 4.45 W into the speaker. That is below the 5 W max input, so the speaker is the hardware safety net if firmware fails.
- **Risks:**
  - The 50 °C ambient limit is tight for a hot shop counter.
  - DC resistance (Re) is not given. The 8 Ω −15 % = 6.8 Ω figure is used in its place; if Re is lower, the fault power rises above 4.45 W.
  - The magnet must sit ≥ 15 mm from the antenna.
- **Alternative (compact):** PUI **AS04008PR-6**: 40 mm, 8 Ω ±15 %, 3 W rated / 4 W max, 86 ±3 dB at 1 W / 0.5 m (≈ 80 dB at 1 m), −25 to 50 °C (datasheet ©2024). About 9 dB quieter, and its 4 W max is below the 4.45 W fault bound.

---

## 4. Amount entry

| | PCB sealed tactile switches + key tops (chosen) | Sealed matrix keypad (Grayhill 88) | Generic membrane keypad | Capacitive pads on the PCB |
|---|---|---|---|---|
| Datasheet life | Omron B3W-4000 (12 mm): **3 000 000 ops min** at 1.96 N | **3 000 000 ops per button** | None (hobby parts have no datasheet) | No wear |
| Sealing | **IP67** switch, terminals excluded (B3W p.1) | **IP42** (DigiKey listing), with a polyester overlay and an optional panel gasket | Sealed surface. Adhesive and the flex tail are weak points. | No holes |
| Liquid on the case | **16 key holes**: a spill can reach the PCB unless the case design prevents it | No holes in the case | No holes | Water films and wet fingers cause **false touches** |
| Contacts | Silver-plated; rated 1–50 mA at 3–24 VDC | Gold-plated stainless domes; 10 mA at 24 Vdc; "compatible with MOS" | Unspecified | — |
| Temperature | −25 to +70 °C | −40 to +80 °C | Unspecified | Drift with humidity and temperature |
| GPIO | 8 (4×4 matrix) | 8 | 8 | 1 per key. The S3 has 14 touch channels, so a 4×4 is impossible and a 3×4 uses 12 of them. |
| Parts | 16 switches + 16 caps + 1 resistor array | **1** | 1 + connector | 0 (copper) + 12 series resistors (Espressif recommends 470 Ω–2 kΩ; HW guide p.20) |
| Legends | On the case or on coloured key tops | Stock legends are **hex 0–9 / A–F** (DigiKey); custom legends need a blank keypad + overlay + inserts | Printed | Printed on the case |
| India price (DigiKey, 2026-10-02) | 16 × ₹87.91 + 16 × ₹35.35 ≈ **₹1 970** | **₹8 245.97** (88BB2-072) | ≈ ₹60–100 (UNVERIFIED) | ≈ 0 |
| Feedback | Tactile | Snap dome, audible | Weak | None (beep only) |

**Decision: 16 × Omron (now Aratas) B3W-4150 + 16 × B32-1310, as a 4×4 matrix on the main PCB.**

The Grayhill 88 was the first choice on paper (one part, no holes). It was rejected for three reasons:

- Its rating is only **IP42**, not better than the switches.
- Its stock legends are hex.
- It costs ₹8 246, about four times the whole Omron set.

**B3W-4150 ratings** (datasheet https://omronfs.omron.com/en_US/ecb/products/pdf/en-b3w.pdf, p.1–2):

- 12 × 12 mm, projected plunger 7.3 mm, operating force 1.96 N max.
- Ground terminal ("to protect against static electricity").
- IP67 (IEC 60529), excluding the terminals.
- Durability **3 000 000 operations min** at 1.96 N.
- Rating 1–50 mA at 3–24 VDC resistive.
- Contact resistance ≤ 100 mΩ initial.
- Bounce ≤ 5 ms.
- −25 to +70 °C.

**B32-1310** is a 12 × 12 mm black key top. Omron lists the B32 12 × 12 tops for "B3W-4000" projected-plunger models (B32 datasheet https://omronfs.omron.com/en_US/ecb/products/pdf/en-b32.pdf, p.1). The B3W-4150 is the projected-plunger, ground-terminal model of that series, so the fit is confirmed by Omron's own tables. Panel thickness is 1.0–2.0 mm.

**Contact current must stay inside the rating.**

- The silver contacts are rated **1–50 mA**. Omron's 10 µA at 1 VDC is only a "reference value" for minimum load.
- Silver can tarnish (sulfide film) in polluted air at dry-circuit currents.
- So the 4 column lines get **2.7 kΩ 1 % pull-ups** to 3V3, as one 4-resistor array:
  - Pressed-key current is 3.18 / 2.727 = **1.17 mA min** and 3.43 / 2.673 = 1.28 mA max.
  - This is inside 1–50 mA.
  - The ESP32's internal 45 kΩ (**typ**) pull-ups would give only about 73 µA.
- Scanning drives one row low at a time. Current flows only while a key in that row is pressed.

**Key layout (16 keys):**

- 0–9 and "."
- Clear / backspace
- Enter (generate QR)
- Cancel
- Repeat announcement
- Menu

Use coloured B32 tops for the function keys: B32-1350 green for Enter, B32-1380 red for Cancel, B32-1330 yellow for Clear (same series, p.1). Emboss the digits on the 3D-printed case beside each key, where they cannot wear off.

**Spill path.** This is the one weakness against a sealed overlay. The case design should:

- put the key holes on a raised island, about 2 mm, with a drip gutter around it;
- keep a tight cap-to-hole clearance;
- drain liquid away from the PCB rather than towards it.

The B3W itself survives washing ("Washing: possible", p.2). The board under the keypad is what needs protecting. Conformal-coating the keypad zone is an option; the owner should decide whether that process step is wanted.

**ESD:** the ground terminals connect to GND, plus the 2 × TPD4E05U06 in 7.4.

**Alternative:** Grayhill **88BB2-072** sealed keypad (datasheet https://grayhill.com/wp-content/uploads/media/88-datasheet.pdf: 3 M ops per button, −40 to 80 °C, contact resistance ≤ 10 Ω, bounce < 4 ms make / < 10 ms break). Use it only if the owner wants no case holes and accepts the cost, the IP42 rating and a hex-legend overlay (or a blank 88BB2 + overlay 88-001 + legend inserts 87AC2046).

---

## 5. Power

### 5.1 Topology

```
USB-C (GCT USB4105) ─ VBUS ─ 4.7 µF ─ TPS259531 eFuse ─ 5V0 ─┬─ TPS62162 buck ─ 3V3 ─┬─ ESP32-S3 module (22 µF + 0.1 µF)
   │ CC1, CC2 ─ 5.1 kΩ 1 % each to GND, + ADC sense          │  (10 µF in, 2.2 µH,   ├─ LCD VDD/VDDI
   │ D+/D− ─ ESP32-S3 GPIO19/20                              │   22 µF out)          ├─ TLV809E → CHIP_PU
   └ TPD4E05U06 on D+, D−, CC1, CC2                          ├─ MAX98357A (10 µF + 0.1 µF) └─ power LED
     shell → GND                                             └─ backlight: 22 Ω → LED-A, AO3400A low side
```

### 5.2 USB-C sink and CC

Source: USB Type-C Cable and Connector Specification Release 2.0, Aug 2019. https://www.usb.org/sites/default/files/USB%20Type-C%20Spec%20R2.0%20-%20August%202019.pdf

- **Rd = 5.1 kΩ to GND on each CC pin, separately.** Only the **±10 %** resistor implementation "can detect power capability" (Table 4-25, p.236), so use **1 %** resistors.
- The sink sees these CC voltages (Table 4-36, p.241):

| Detection | Range | Threshold |
|---|---|---|
| vRd-USB | 0.25–0.61 V | 0.66 V |
| vRd-1.5 | 0.70–1.16 V | 1.23 V |
| vRd-3.0 | 1.31–2.04 V | |

- With Default advertisement and no USB enumeration, BC 1.2 or PD, a sink "may draw up to 500 mA" (§4.6.2.1, p.218).
- "A Sink that takes advantage of the additional current offered (e.g., 1.5 A or 3.0 A) **shall monitor the CC pins** and shall adjust its current consumption within tSinkAdj" (p.219).
- **CC sensing costs no parts.** CC1 and CC2 go straight to ADC1_CH0/1 in ATTEN2 (0–1600 mV, ±10 mV calibrated error). That cleanly separates the bands, whose gaps are ≥ 90 mV. Anything above 1.6 V reads as full scale, which means 3 A.
- With Rd present, the CC pin is at most 2.04 V (Table 4-25), below the ESP32's 3.6 V abs max.
- **Charger output range:** a Type-C current-advertising charger "shall output … 4.75 V – 5.5 V when no current is being drawn and between 4.0 V – 5.5 V at 3 A" (p.226). **The design must work from 4.0 to 5.5 V.**
- **Sink VBUS capacitance ≤ 10 µF** at the receptacle when not attached (Table 4-3, p.142). Hence 4.7 µF before the eFuse; downstream capacitance is hidden behind the eFuse.

### 5.3 Input protection: eFuse vs polyfuse

| | Polyfuse (PTC) | **eFuse TPS259531 (chosen)** |
|---|---|---|
| Over-current | Slow (seconds); hold current derates with heat | Active limit, ±7.5 % class, set by 1 resistor; 5 µs short-circuit response |
| Over-voltage (faulty adapter) | **None.** A 9–12 V fault kills the amp (6 V abs max). | **Clamp: output held at 5.2–5.7 V for VIN up to 18 V** |
| Inrush | None | Soft-start, 21.9 V/ms default |
| Thermal | — | Over-temperature shutdown, auto-retry |
| Parts | 1 | 1 IC + RILM + 2 EN resistors |

**TPS259531DSGR**

- Datasheet: https://www.ti.com/lit/ds/symlink/tps2595.pdf (SLVSE57C).
- Ratings relied on:
  - VIN abs max **20 V**; EN abs max **7 V** (p.5).
  - VOVC 5.5–5.9 V; VCLAMP 5.2–5.7 V (p.7).
  - RON ≤ 53 mΩ over −40 to 125 °C (p.7).
  - ILIM at 1780 Ω: 1.09–1.24 A (p.7).
  - RILM = 2000 / (ILIM − 0.04) (Eq. 4, p.21).
  - IQ ≤ 250 µA.
  - RθJA 65.6 °C/W (p.6).
- **RILM = 1.02 kΩ 1 %** gives 2.0 A nominal, about 1.85–2.15 A by the ±7.5 % accuracy class. 1.02 kΩ lies between the tested points, so this range is an interpolation.
  - The minimum is above the 1.40 A worst-case load (5.6).
  - Dissipation at 1.4 A: 1.4² × 0.053 = 0.10 W, so ΔT ≈ 7 °C.
- **EN/UVLO divider 200 kΩ / 100 kΩ, which is required.** The datasheet allows EN tied to IN only below 6 V (p.5 note 2). In an overvoltage fault, a direct tie would break EN's 7 V abs max.
  - With the divider, VIN = 20 V gives EN = 6.67 V (< 7 V).
  - Turn-on is at VIN 3.39–3.81 V and turn-off at 3.09–3.51 V (VUVLO 1.13–1.27 V rising, 1.03–1.17 V falling).
- FLT (open-drain) is unused. dVdt is left open.
- **ESD on VBUS: no low-voltage TVS.** A 5–6 V TVS would turn a survivable 9–18 V adapter fault into a destroyed or shorted TVS.
  - The 4.7 µF X7R absorbs IEC 61000-4-2 charge: 150 pF × 8 kV = 1.2 µC, which raises about 4 µF effective by about 0.3 V.
  - Hot-plug ringing with a ceramic input capacitor is at most about 2 × 5.5 V = 11 V, below the 20 V abs max.
  - Both are engineering calculations, not datasheet guarantees. This is a residual risk (Section 10).
- **Alternative:** TI **TPS259470** (TPS25947: 28 V abs max, −15 V reverse, adjustable OVLO cut-off instead of a clamp). It survives a broken 20 V PD charger, at the cost of more pins and resistors. Datasheet: https://www.ti.com/lit/ds/symlink/tps25947.pdf

### 5.4 3.3 V regulator: buck vs LDO (thermal check)

- **Load:** 0.5 A (ESP32 IVDD design minimum) + 15 mA (LCD) + ≈ 2 mA (LED, supervisor, pull-ups) ≈ **0.52 A**.
- **LDO option.** Dissipation is (5.5 − 3.3) × 0.52 = **1.14 W**.
  - In SOT-223 (θJA about 60 °C/W on generous copper; **UNVERIFIED** generic figure), that is ΔT ≈ 69 °C.
  - At 55–60 °C inside the case, the junction would reach 124–129 °C, at or over the usual 125 °C limit.
  - It also draws 0.52 A from USB, against about 0.40 A for a buck.
  - **Rejected.**
- **Buck: TPS62162DSGR (fixed 3.3 V).**
  - Datasheet: https://www.ti.com/lit/ds/symlink/tps62162.pdf (SLVSAM2E).
  - VIN 3–17 V (p.5), VIN abs max 20 V, so it survives anything the eFuse passes.
  - 1 A output; high-side current limit 1.45–2.45 A (p.6).
  - Output accuracy −3 %/+3 % in PWM and −3.5 %/+4 % in power-save (p.6), so 3.18–3.43 V. That is inside the ESP32's 3.0–3.6 V.
  - UVLO 2.6–2.82 V falling.
  - RDS(on) high side ≤ 600 mΩ and low side ≤ 200 mΩ (at VIN ≥ 6 V; p.6).
  - RθJA 61.8 °C/W (DSG; p.5).
  - FB to AGND on fixed versions (p.4).
  - Parts: CIN 10 µF/25 V, L 2.2 µH, COUT 22 µF (a recommended combination, Table 2, p.14). EN tied to VIN; PG unused.
- **Thermal check:**
  - Conduction loss at 0.52 A out, VIN 4.5 V (D ≈ 0.73), with RDS(on) extrapolated to about 0.72 Ω at the lower gate drive (estimate): 0.27 × (0.73 × 0.72 + 0.27 × 0.25) + 0.27 × 0.082 (DCR max) ≈ 0.19 W.
  - Switching loss is estimated at 0.05 W. The total, about **0.24 W**, gives ΔT ≈ 15 °C and Tj ≈ 75 °C at 60 °C internal ambient.
  - Efficiency is therefore ≥ 85 % (used in the budget).
- **Inductor: Murata DFE252012F-2R2M=P2.**
  - Datasheet: https://www.murata.com/~/media/webrenewal/products/inductor/chip/tokoproducts/wirewoundmetalalloychiptype/m_dfe252012f.ashx
  - Ratings: 2.2 µH ±20 %; DCR **82 mΩ max**; inductance-decrease current (ΔL/L = 30 %) **3.3 A max rating** (3.6 A typ); temperature-rise current (ΔT 40 °C) 2.3 A; −40 to +125 °C; 2.5 × 2.0 × 1.2 mm.
  - The 30 % criterion is the same one TI's inductor table uses (p.15, note 2).
  - TI's own BOM inductor, TDK VLF3012ST-2R2M1R4, is **obsolete** at DigiKey. Coilcraft XFL3012-222MEC is only at LCSC (130 pcs), and LCSC lists its Isat as 1.6 A.
- **Sizing per TI Eq. 7/8 (p.14–15):**
  - ΔIL = 3.3 × (1 − 3.3 / 5.7) / (1.76 µH [L −20 %] × 2.25 MHz) = 0.35 A.
  - IL(max) = 0.52 + 0.18 = 0.70 A. With TI's recommended 20 % margin that is **0.84 A**.
  - fSW is typical only; at 1.5 MHz the margined figure is 0.94 A.
- **Current-limit case (stricter):** the buck's static limit is ILIMF ≤ 2.45 A (p.6). The dynamic peak per Eq. 2 (p.10) is 2.45 + (5.7 − 3.3) × 30 ns / 1.76 µH = **2.49 A**.
  - The 3.3 A rating covers this, so the inductor stays out of saturation even during a short on the 3.3 V rail. That keeps the current limit effective.
  - The 2.3 A thermal rating is above the 0.70 A operating current.
- Dropout at VBUS 4.0 V: 4.0 − 0.52 × (0.72 + 0.1) ≈ 3.57 V available. The rail still regulates, but the "VIN ≥ VOUT + 1 V" condition for the accuracy spec is not met below 4.3 V.
- **Alternative:** TI TLV62569DBV (SOT-23-5, adjustable, 2 A). Its VIN max of 5.5 V gives no margin against the eFuse's 5.7 V clamp ceiling, so it is second choice.

### 5.5 Reset supervisor

- Espressif: "If the user application has … slow power rise or fall … frequent power on/off operations … unstable power supply … the RC circuit alone may not meet the timing requirements … flash erase operations [may] occasionally fail … reserve a power monitor chip … threshold … around 3.0 V" (HW Design Guidelines §1.3.3, p.8–9). Indian power cuts are exactly this case.
- **TLV809EA29DBZR (2.93 V, delay variant A, push-pull active-low)**, driving CHIP_PU directly. The datasheet's package addendum lists it as Active. This replaces the 10 kΩ / 1 µF RC, so the part count is the same or lower.
  - Datasheet: https://www.ti.com/lit/ds/symlink/tlv803e.pdf (SLVSES2J).
  - Threshold accuracy ±2 % (2.871–2.989 V); hysteresis 0.9–1.5 % (p.8).
  - Delay 130–270 ms (p.9). This is ≥ the 50 µs tSTBL (HW guide Table 2, p.8).
  - Output defined down to VDD = 0.7 V (VPOR, p.8).
  - IDD ≤ 1 µA.
- **Margin check:**
  - VIT+ max = 2.989 × 1.015 = 3.034 V, below the buck's 3.18 V minimum output, so reset always releases.
  - Reset asserts by 2.87 V at the latest. The ESP32's recommended 3.0 V minimum is crossed 0–130 mV earlier; the on-module flash is assumed rated to 2.7 V (**UNVERIFIED**).
- The buck's PG pin was considered and rejected. It goes high-impedance when VIN falls below UVLO (p.4), so CHIP_PU would float up with the dying rail during power-down. The TLV809E keeps the output defined down to 0.7 V.

### 5.6 Power budget (maximum values)

| Load | Rail | Basis | Max |
|---|---|---|---|
| ESP32-S3-WROOM-1 | 3.3 V | IVDD ≥ 0.5 A design input (Table 6-2). Covers the 355 mA "Peak" TX. | 500 mA |
| LCD logic (VDD + VDDI) | 3.3 V | IDD 15 mA max (NHD p.6) | 15 mA |
| Power LED | 3.3 V | (3.3 − 1.8) / 1.5 kΩ | 1 mA |
| TLV809E + keypad pull-ups | 3.3 V | 1 µA + ≤ 4 × 1.28 mA (only while a row's keys are pressed) | ≤ 5.2 mA |
| **3.3 V subtotal** | | | **≈ 0.52 A = 1.71 W** |
| Buck input (η ≥ 85 %) | 5 V | 1.71 W / 0.85 / VIN | 0.40 A @ 5.0 V; 0.45 A @ 4.5 V; 0.50 A @ 4.0 V |
| Backlight | 5 V | 4 × 130 Ω per-LED ballast, Vf ≥ 2.6 V (est.), rail ≤ 5.7 V | 95 mA (budget keeps 123 mA from the first draft, as extra margin) |
| MAX98357A, firmware-limited (no clipping) | 5 V | 1.49 W / 0.85 + IDD 3.35 mA | 0.35 A |
| MAX98357A, firmware-fault bound | 5 V | 5.5² / 6.8 Ω / 0.9 | 0.90 A |
| eFuse IQ | VBUS | ≤ 250 µA | — |
| **VBUS total, normal worst case** | | 0.50 (at 4.0 V) + 0.123 + 0.35 | **≈ 0.97 A** |
| **VBUS total, firmware-fault worst case** | | 0.37 (at 5.5 V) + 0.123 + 0.90 | **≈ 1.40 A** |

- Every row is a maximum or a bound, and rows that peak at different input voltages are added together, so the totals are conservative.
- The design stays below **1.5 A in every case**, including firmware failure, so a 1.5 A-advertising source is never overloaded.

### 5.7 Adapter requirement

| Source advertisement | Allowed draw | Device behaviour (firmware reads CC) |
|---|---|---|
| Default USB (56 kΩ Rp; any USB-A port or **A-to-C cable**) | 500 mA (p.218) | Cannot be guaranteed: Wi-Fi alone needs up to about 0.45 A at 5 V. **Decided (D-008):** run with volume and backlight capped and show "Use a 5 V USB-C adapter for full volume". Wi-Fi TX peaks may briefly exceed 500 mA; the TLV809E handles any brownout cleanly. |
| 1.5 A (22 kΩ) | 1.5 A | Full function. Worst-case 1.40 A fits. |
| 3.0 A (10 kΩ) | 3 A | Full function |

- **Requirement: 5 V USB-C adapter advertising ≥ 1.5 A, with a USB-C to USB-C cable.**
- **Recommended:** a 5 V / 3 A USB-C adapter. Most USB-C PD chargers advertise 3 A on Rp at 5 V; this is general practice, **UNVERIFIED** per model.

---

## 6. Battery backup (decided: none, D-007)

**What it would take:**

- A Li-ion cell (BIS-registered under IS 16046, which is compulsory in India).
- A charger with power-path. For example, TI BQ24074 needs 6–8 passives.
- Cell protection: a protected cell, or a DW01-class IC plus a dual MOSFET.
- A 3.3 V path that works from 3.0–4.2 V. A buck-boost or low-dropout regulator would replace the TPS62162.
- Either a boost to 5 V for the amplifier, or accepting lower speaker power at about 3.7 V (MAX98357A: 0.77 W **typ** into 8 Ω at 3.7 V, p.5).
- About 12–20 extra parts, a thermal design for the cell, and a fuel-gauge or ADC divider.

**Safety:**

- Li-ion cells must be **charged only at 0–45 °C** cell surface temperature; for example, the Samsung INR18650-35E datasheet gives charge 0–45 °C and discharge −10–60 °C (https://www.orbtronic.com/content/samsung-35e-datasheet-inr18650-35e.pdf).
- A shop counter in summer can approach this limit, so the charger needs an NTC-gated charge cut-off (another part).
- A swollen or punctured cell is the only fire hazard the product would otherwise not have.

**Usefulness:**

- When mains fails, the shop's Wi-Fi router usually fails too, unless it has its own UPS. A battery in the soundbox does not keep payments flowing by itself.
- It helps only where the shop uses a phone hotspot, or where the router already has a UPS.

**Recommendation: no battery on the board.**

- Document that the device runs from any 5 V USB-C source. That includes a USB power bank, and the 5 V USB port of a router mini-UPS, which then backs up router and soundbox together.
- This adds no parts and no Li-ion risk inside our case, and needs no BIS cell sourcing.
- Caveat: some power banks switch off at low load. The device's idle draw of about 0.1–0.2 A (**typ**, estimate) is above the usual auto-off threshold of about 50–100 mA (**UNVERIFIED**, varies by model).
- **This is the owner's decision.** If a battery is wanted, it should be a separate design pass with the charger chosen on guaranteed thermal limits.

---

## 7. Other items a reliable product needs

### 7.1 Status LED

- No GPIO status LED: the display and voice already show status.
- **One power LED on 3.3 V** (0603 AlInGaP red or green with Vf ≤ 2.4 V, + 1.5 kΩ, about 1 mA). It tells a shopkeeper "powered but firmware dead" apart from "no power" without any firmware. This costs 2 parts.

### 7.2 BOOT / RESET buttons

- **Not fitted.**
- ESP-IDF: "The USB Serial/JTAG Controller is able to put the ESP32-S3 into download mode automatically." Recovery is needed only if the application reconfigures USB pins or disables the controller: "pull low GPIO0 and reset the chip" (https://docs.espressif.com/projects/esp-idf/en/stable/esp32s3/api-guides/usb-serial-jtag-console.html).
- So: a **GPIO0 test pad**. Hold it to GND while plugging in USB.
- Reset is a power cycle. Do not short CHIP_PU to GND: the TLV809E output is push-pull.
- Security: in Release-mode flash encryption with ROM download in **Secure Download Mode**, new images arrive only by OTA (ESP-IDF flash-encryption docs). The pad stays useful for recovery within those limits.

### 7.3 Watchdog

- The ESP32-S3 has two MWDTs and an RTC watchdog (RWDT) in the RTC domain, clocked from the RTC slow clock. "During the flash boot process, RWDT and the first MWDT are enabled automatically." The RWDT can force a **system reset**, and the timers have write protection (S3 datasheet §4.1.3.8, p.45).
- **No external watchdog is recommended.** The hardware already limits the worst firmware faults without any watchdog:
  - The speaker takes the DC or square-wave fault (4.45 W < 5 W max).
  - The eFuse limits current.
  - The backlight resistor is rated for 100 % duty.
  - The amp is held off by its own pull-down whenever the MCU resets.
- A hang only costs availability. The RWDT plus the ESP-IDF Task and Interrupt watchdogs recover it.
- Firmware rule: never disable the RWDT or MWDT, and keep the bootloader watchdog enabled.
- The owner should confirm this reading of "watchdogs stay in hardware" (the RWDT is on-chip hardware).

### 7.4 ESD on exposed parts

| Exposed part | Protection |
|---|---|
| USB-C shell | Tied to GND (plastic case, no chassis) |
| USB D+/D−, CC1/CC2 | TPD4E05U06 at the connector. VRWM 5.5 V; VBR 6.5–8.5 V (p.7); IEC 61000-4-2 ±12 kV contact / ±15 kV air (p.6); 0.5 pF typ. |
| VBUS | eFuse + 4.7 µF (5.3) |
| Keypad (human-touched key tops; ESP32 pins are only 2 kV HBM) | B3W-4150 ground terminals to GND, plus 2 × TPD4E05U06 next to the switch matrix (rows and columns). Same ESD part, reused. Omron states no IEC kV figure for the ground terminal, so paper cannot close this risk without the arrays. |
| Display glass / frame | The case window overlaps the glass ledge over the chip-on-glass driver. Ground the panel frame if it has a tab (**UNVERIFIED**: none shown). |
| Speaker | Behind a grille. Cone and leads are not reachable. |

### 7.5 Test points

Pads only, no BOM cost: GND ×2, VBUS, 5V0, 3V3, CHIP_PU (measure only), GPIO0, U0TXD, U0RXD.

### 7.6 Assembly note

- Four packages are leadless:
  - WSON-8 2×2 mm: TPS2595 and TPS62162.
  - USON-10: TPD4E05U06.
  - TQFN-16 3×3 mm: MAX98357A.
- They need stencil and reflow, so use PCB Power's assembly service or a hot-air station. PCB Power's assembly capability is **UNVERIFIED** here.

---

## 8. Draft BOM

| # | Qty | Ref (draft) | Part (MPN) | Function |
|---|---|---|---|---|
| 1 | 1 | U1 | Espressif ESP32-S3-WROOM-1-N16 | MCU + Wi-Fi + 16 MB flash |
| 2 | 1 | U2 | TI TPS259531DSGR | eFuse: OCP, OVP clamp, soft-start |
| 3 | 1 | U3 | TI TPS62162DSGR | 3.3 V buck |
| 4 | 1 | U4 | TI TLV809EA29DBZR | Reset supervisor → CHIP_PU |
| 5 | 1 | U5 | ADI MAX98357AETE+T | I2S class-D amplifier |
| 6 | 3 | U6–U8 | TI TPD4E05U06DQAR | ESD: USB/CC; keypad rows; keypad columns |
| 7 | 1 | Q1 | AOS AO3400A | Backlight low-side switch |
| 8 | 1 | L1 | Murata DFE252012F-2R2M=P2 | Buck inductor 2.2 µH, Isat 3.3 A rated, DCR ≤ 82 mΩ |
| 9 | 1 | J1 | GCT USB4105-GF-A | USB-C receptacle |
| 10 | 1 | J2 | Molex 54132-4062 | 40-pin 0.5 mm FPC (display) |
| 11 | 1 | J3 + housing | JST B2B-XH-A(LF)(SN) + XHP-2 + 2 × SXH-001T-P0.6 | Speaker |
| 12 | 1 | DS1 | Newhaven NHD-2.8-240320AF-CSXP-F | 2.8 in IPS TFT |
| 13 | 16 | SW1–SW16 | Omron/Aratas B3W-4150 | 4×4 keypad, IP67 sealed switches with ground terminal |
| 13a | 16 | — | Omron/Aratas B32-1310 (black); function keys B32-1350 / -1380 / -1330 | Key tops |
| 13b | 1 | RN1 | 4 × 2.7 kΩ 1 % resistor array (e.g. 0603 ×4 convex; MPN at schematic) | Column pull-ups, keeping contact current ≥ 1 mA |
| 14 | 1 | LS1 | PUI Audio AS06608PS-R | 8 Ω 4 W speaker |
| 15 | 1 | D1 | 0603 AlInGaP LED, Vf ≤ 2.4 V | Power indicator |
| 16 | 2 | R1, R2 | 5.1 kΩ 1 % 0402/0603 | CC Rd |
| 17 | 1 | R3 | 1.02 kΩ 1 % | eFuse RILM (2.0 A) |
| 18 | 1 | R4 | 200 kΩ 1 % | eFuse EN divider, top |
| 19 | 1 | R5 | 100 kΩ 1 % | eFuse EN divider, bottom |
| 20 | 4 | R6a–R6d | 130 Ω 1 % 1206 | Backlight ballast, one per LED cathode (2.5) |
| 21 | 1 | R7 | 100 kΩ | Q1 gate pull-down (backlight off in reset) |
| 22 | 1 | R8 | 2.2 kΩ | SD_MODE series (datasheet p.17) |
| 23 | 1 | R9 | 1.5 kΩ | LED |
| 24 | 1 | C1 | 4.7 µF 25 V X7R 0805 | VBUS (≤ 10 µF per Type-C Table 4-3) |
| 25 | 1 | C2 | 10 µF 25 V X7R 0805 | 5 V rail / buck CIN (Espressif "10 µF at power entrance") |
| 26 | 2 | C3, C4 | 22 µF 10 V X7R 0805 | Buck COUT; module 3V3 |
| 27 | 1 | C5 | 10 µF 10 V X7R | MAX98357A VDD bulk |
| 28 | ~5 | C6–C10 | 0.1 µF 0402 X7R | Module, LCD VDD/VDDI, amp, supervisor |
| — | — | — | Test pads ×9 | |

- **Count:**
  - 9 IC/transistor lines: U1–U8 and Q1.
  - 3 connectors: USB-C, FPC and speaker.
  - 1 inductor.
  - 16 switches + 16 key tops.
  - About 24 passives.
  - 2 off-board items: panel and speaker.
- **Indicative cost.** The main parts, at DigiKey India single-unit prices on 2026-10-02 (ex-GST, ex-duty and shipping), total **≈ ₹7 420**. The display (₹2 742) and the keypad switches and tops (₹1 972) dominate. Passives and crimps are excluded.
- Not fitted, on purpose:
  - USB-UART bridge.
  - BOOT/RESET buttons.
  - External watchdog.
  - VBUS TVS.
  - Output LC filter.
  - PSRAM.
  - Battery.

---

## 9. India stock and price (checked 2026-10-02)

Prices are INR per piece before GST, read from each vendor's product page. LCSC prices are USD, before shipping and customs.

- **Mouser India, element14 India and Robu blocked automated access**, so their stock is **UNVERIFIED**. The team lead will check them in a browser.
- Evelta was readable but carries few of these parts.
- **LCSC look-alikes:** LCSC also sells AO3400A and TPD4E05U06 made by other companies under the same part number. Order only the original-maker C numbers listed here.

| Part | DigiKey India: stock, ₹ qty 1 / qty 10, status | Second source |
|---|---|---|
| ESP32-S3-WROOM-1-N16 | 591; ₹582.86 / ₹504.60; Active. https://www.digikey.in/en/products/detail/espressif-systems/ESP32-S3-WROOM-1-N16/16162647 | LCSC C2913199: 436, US$5.17. Evelta: not listed. |
| (alt) ESP32-S3-WROOM-1-N8 | 1 182; ₹540.81 / ₹467.91; Active | — |
| (alt) ESP32-S3-WROOM-1-N16R8 | 0 (650 due 14 Oct 2026); ₹645.92 | Evelta: 669, ₹345.00 |
| NHD-2.8-240320AF-CSXP-F | 441; ₹2 742.29; Active; **18-week lead** | **None found (single source)** |
| Molex 54132-4062 | 29 613; ₹236.96 / ₹201.23; Active; bottom contact, 0.30 mm FFC | None found (LCSC no listing) |
| AO3400A | 292 013; ₹49.69 / ₹30.77; Active | LCSC C20917 (AOS): 539 925, US$0.0898 |
| MAX98357AETE+T | 39 357; ₹389.84 / ₹294.68; Active; 11-week factory lead | LCSC C910544: 27 353, US$1.3282. Evelta: modules only. |
| PUI AS06608PS-R | 2 098; ₹795.93 / ₹610.66; Active; **30-week lead** | None found (single source) |
| JST B2B-XH-A(LF)(SN) | 293 978; ₹9.56 / ₹8.31 | LCSC C158012: 291 280 |
| JST XHP-2 | 244 319; ₹9.56 / ₹5.64 | LCSC C144401: 88 950 |
| JST SXH-001T-P0.6 crimp | **not checked** | — |
| Omron/Aratas B3W-4150 | 149; ₹87.91 / ₹75.20; Active (149 covers 9 boards) | None found |
| Omron/Aratas B32-1310 | 9 470; ₹35.35 / ₹31.72; Active | None found |
| GCT USB4105-GF-A | 116 353; ₹76.44 / ₹64.59; Active | LCSC C3020560: out of stock |
| TI TPS259531DSGR | 5 093; ₹99.37 / ₹71.38; Active | LCSC C2155674: 3 197, US$0.8785 |
| TI TPS62162DSGR | 16 515; ₹168.17 / ₹123.07; Active; 20-week lead | LCSC C40256: 6 667, US$1.29. Evelta: out of stock (₹101.82). |
| Murata DFE252012F-2R2M=P2 | 180 353; ₹23.89 / ₹19.87; Active. https://www.digikey.in/en/products/result?keywords=DFE252012F-2R2M%3DP2 | LCSC C576403: 17 970, US$0.1495 (qty 5) |
| TI TLV809EA29DBZR | 8 999; ₹29.62 / ₹20.35; Active | LCSC C1852110: 5 000, US$0.3013 (min 5) |
| TI TPD4E05U06DQAR | 340 367; ₹78.35 / ₹48.64; Active | LCSC C138714 (TI): 142 330, US$0.0847 |
| (rejected) Grayhill 88BB2-072 | 169; ₹8 245.97; IP42; legends 0–9 / A–F | — |
| (obsolete) TDK VLF3012ST-2R2M1R4 | OBSOLETE, 0 | LCSC C19080: out of stock |

**DigiKey product URLs (from the stock checks):**

- NHD-2.8: https://www.digikey.in/en/products/detail/newhaven-display-intl/NHD-2-8-240320AF-CSXP-F/9849907
- Molex: https://www.digikey.in/en/products/detail/molex/0541324062/2404784
- PUI: https://www.digikey.in/en/products/detail/pui-audio-inc/AS06608PS-R/1745563
- B3W-4150: https://www.digikey.in/en/products/detail/omron-electronics-inc-emc-div/B3W-4150/368409
- B32-1310: https://www.digikey.in/en/products/detail/aratas-formerly-omron-components/B32-1310/11535
- USB4105: https://www.digikey.in/en/products/detail/gct/USB4105-GF-A/11198441
- TPS259531: https://www.digikey.in/en/products/detail/texas-instruments/TPS259531DSGR/8021096
- TPS62162: https://www.digikey.in/en/products/detail/texas-instruments/TPS62162DSGR/2833447
- TLV809EA29: https://www.digikey.in/en/products/detail/texas-instruments/TLV809EA29DBZR/11308779
- TPD4E05U06: https://www.digikey.in/en/products/detail/texas-instruments/TPD4E05U06DQAR/3996774
- AO3400A: https://www.digikey.in/en/products/detail/alpha-omega-semiconductor-inc/AO3400A/1855772
- JST: https://www.digikey.in/en/products/detail/jst-sales-america-inc/B2B-XH-A-LF-SN/1651045 and https://www.digikey.in/en/products/detail/jst-sales-america-inc/XHP-2/555485
- MAX98357A: https://www.digikey.in/en/products/detail/analog-devices-inc-maxim-integrated/MAX98357AETE-T/4936122

**Single-source parts and their fallbacks:**

| Part | Fallback |
|---|---|
| NHD-2.8 panel | NHD-2.4-240320AF-CSXP. Same controller and connector family; modules shrink to 0.61 mm. Its stock was not checked. |
| AS06608PS-R speaker | PUI AS04008PR-6. About 9 dB quieter, and its 4 W max is below the 4.45 W fault bound. Not checked in stock. |
| B3W-4150 (149 pcs) | B3W-4050: same switch without the ground terminal. Not checked in stock. |
| Molex 54132-4062 | Any 40-pin 0.5 mm bottom-contact ZIF for 0.3 mm FPC, re-checked against the NHD drawing |

## 10. Risks and open questions

1. **QR payload length.** The display choice assumes ≤ 192 bytes (V8 at EC L). PhonePe-style strings over 300 bytes would drop modules to 0.54 mm (about 30 cm scan range). The backend must log the real length once Razorpay `image_content` is enabled.
2. **USB "Default" power sources.** The A-to-C cable case. Decided: run limited and warn (D-008, 5.7).
3. **NHD backlight maximum current** is not stated, and the pin table and electrical table disagree (100 vs 160 mA). Ask Newhaven.
4. **MAX98357A output power is typical only.** Loudness margin rests on typical amp power and the speaker's ±3 dB tolerance.
5. **Speaker operating limit is 50 °C.** It is also heavy (204 g), so the case needs a solid mount.
6. **VBUS ESD** relies on capacitor charge absorption, which is a calculation, not a datasheet guarantee. It must be confirmed at EMC test. The alternative is TPS25947 + TVS2200 (22 V flat-clamp, 28.4 V max at 40 A), sized for a 28 V-abs-max eFuse.
7. **Leadless packages** need reflow assembly.
8. **Keypad spill path.** 16 key holes in the case top (Section 4). This is handled in the case design (D-016): drip island and gutter. Whether to conformal-coat is decided then.
9. **Stock and lead times.** The panel (441 pcs, 18-week lead) and the speaker (30-week lead) are single-source at DigiKey India. The N16 module is at DigiKey and LCSC but not at Evelta. If only the N16R8 can be had, enable PSRAM ECC to keep the 85 °C rating, and lose GPIO35–37.
10. **Antenna** needs ≥ 15 mm clearance from the speaker magnet, the display frame and the case walls. Plan the case around it.
11. **FPC orientation.** The Molex 54132-4062 is bottom-contact (DigiKey), and the NHD contacts are on the panel's back face. The FPC path must present that face to the PCB (2.5).
12. **Older TI datasheets.** TPS62162 (2017) and TPS2595 (2018) are both Active at DigiKey India. TI's own reference inductor for the TPS62162 is now obsolete, and the Murata replacement was sized from TI's equations (5.4).
13. **LCSC look-alikes** of AO3400A and TPD4E05U06 exist. Order only the original-maker C numbers (Section 9).
14. **TLV809EA29DBZR pinout** is the default SOT-23 one: 1 = GND, 2 = RESET, 3 = VDD. The -R and -V variants differ, so the footprint must match this one.

## 11. UNVERIFIED items (collected)

- The NPCI-mandated QR error-correction level, if any.
- Phone camera FOV and scan-stream resolution: an assumed 67° HFOV at 720p/1080p.
- XTS-AES key size on ESP32-C3. Not re-checked; it does not affect the decision.
- MAX98360A as a drop-in alternative.
- SOT-223 θJA of about 60 °C/W used in the LDO rejection. This is a generic figure; the decision does not depend on it.
- Shop ambient noise of 65–75 dBA.
- The on-module flash minimum voltage of 2.7 V.
- Power-bank auto-off thresholds.
- Most USB-C chargers advertising 3 A at 5 V.
- NHD panel frame ground tab.
- IEC ESD rating of the B3W ground terminal.
- Stock at Mouser India, element14 India and Robu (blocked to automated reads).
- SXH-001T-P0.6 crimp stock.
- PCB Power assembly capability.

## 12. Sources

- ESP32-S3-WROOM-1/1U datasheet v1.8: https://www.espressif.com/sites/default/files/documentation/esp32-s3-wroom-1_wroom-1u_datasheet_en.pdf
- ESP32-S3 Series datasheet v2.2: https://www.espressif.com/sites/default/files/documentation/esp32-s3_datasheet_en.pdf
- ESP32-S3 Hardware Design Guidelines (release master, 2026-09-29): https://docs.espressif.com/projects/esp-hardware-design-guidelines/en/latest/esp32s3/
- ESP32-C3-MINI-1 datasheet v2.2: https://www.espressif.com/sites/default/files/documentation/esp32-c3-mini-1_datasheet_en.pdf
- ESP32-C3-WROOM-02 datasheet v1.7: https://www.espressif.com/sites/default/files/documentation/esp32-c3-wroom-02_datasheet_en.pdf
- ESP32-C6-WROOM-1 datasheet v1.4: https://www.espressif.com/sites/default/files/documentation/esp32-c6-wroom-1_datasheet_en.pdf
- ESP-IDF S3 flash encryption: https://docs.espressif.com/projects/esp-idf/en/stable/esp32s3/security/flash-encryption.html
- ESP-IDF C6 Secure Boot v2: https://docs.espressif.com/projects/esp-idf/en/stable/esp32c6/security/secure-boot-v2.html
- ESP-IDF C6 flash encryption: https://docs.espressif.com/projects/esp-idf/en/stable/esp32c6/security/flash-encryption.html
- ESP-IDF S3 USB Serial/JTAG console: https://docs.espressif.com/projects/esp-idf/en/stable/esp32s3/api-guides/usb-serial-jtag-console.html
- MAX98357A/B datasheet (Rev 7 copy): https://cdn-shop.adafruit.com/product-files/3006/MAX98357A-MAX98357B.pdf
- USB Type-C Cable and Connector Spec R2.0: https://www.usb.org/sites/default/files/USB%20Type-C%20Spec%20R2.0%20-%20August%202019.pdf
- TI TPS2595: https://www.ti.com/lit/ds/symlink/tps2595.pdf
- TI TPS25947: https://www.ti.com/lit/ds/symlink/tps25947.pdf
- TI TPS62162: https://www.ti.com/lit/ds/symlink/tps62162.pdf
- TI TLV809E: https://www.ti.com/lit/ds/symlink/tlv803e.pdf
- TI TPD4E05U06: https://www.ti.com/lit/ds/symlink/tpd4e05u06.pdf
- TI TVS2200: https://www.ti.com/lit/ds/symlink/tvs2200.pdf
- AOS AO3400A: https://www.aosmd.com/res/datasheets/AO3400A.pdf
- GCT USB4105: https://gct.co/files/drawings/usb4105.pdf
- Newhaven NHD-2.8-240320AF-CSXP-F rev 10: https://newhavendisplay.com/content/specs/NHD-2.8-240320AF-CSXP-F.pdf
- Grayhill Series 88: https://grayhill.com/wp-content/uploads/media/88-datasheet.pdf
- Murata DFE252012F: https://www.murata.com/~/media/webrenewal/products/inductor/chip/tokoproducts/wirewoundmetalalloychiptype/m_dfe252012f.ashx
- Omron B3W: https://omronfs.omron.com/en_US/ecb/products/pdf/en-b3w.pdf
- Omron B32: https://omronfs.omron.com/en_US/ecb/products/pdf/en-b32.pdf
- PUI AS06608PS-R: https://puiaudio.com/file/specs-AS06608PS-R.pdf
- PUI AS04008PR-6: https://puiaudio.com/file/specs-AS04008PR-6.pdf
- Google ML Kit barcode input guidelines: https://developers.google.com/ml-kit/vision/barcode-scanning/android
- Waveshare 4.2 in e-Paper manual: https://www.waveshare.com/wiki/4.2inch_e-Paper_Module_Manual
- Samsung INR18650-35E: https://www.orbtronic.com/content/samsung-35e-datasheet-inr18650-35e.pdf
- BIS CRS for Li-ion (IS 16046), secondary summary: https://www.agileregulatory.com/blogs/bis-registration-for-lithium-ion-batteries-under-crs
