# Decision log

Each entry records what was decided, why, and what was given up. Newest entries go at the bottom.
A decision is changed by adding a new entry that replaces it, never by editing the old one. The
table below is the only part that is updated in place; it shows which decisions currently hold.

## Current status

| ID | Decision | Status |
|---|---|---|
| D-001 | Real payments through a licensed gateway | **Partly replaced by D-012.** Confirmation through the gateway's signed notification still holds. A live account in the owner's name is on hold. |
| D-002 | Custom circuit board and 3D-printed case | Holds |
| D-003 | Wi-Fi only | Holds |
| D-004 | Public repository from day one | Holds |
| D-005 | Device draws the QR; gateway keys stay on the backend | Holds. Whether the raw string is available depends on the gateway chosen under D-012. |
| D-006 | Razorpay as the gateway | **Replaced by D-012** |
| D-007 | No battery | Holds |
| D-008 | Weak USB power: run limited and warn | Holds |
| D-009 | On-chip watchdogs count as hardware watchdogs | Holds |
| D-010 | 66 mm speaker | Holds |
| D-011 | ESD protection on the keypad | Holds |
| D-012 | Razorpay paused; find a no-KYC sandbox; no live merchant account just for the project | Holds |
| D-013 | MCU: ESP32-S3-WROOM-1-N16 | Holds, pending schematic review |
| D-014 | Display: 2.8 in IPS panel, one resistor per backlight LED | Holds, pending schematic review |
| D-015 | Audio: MAX98357A into 8 Ω | Holds, pending schematic review |
| D-016 | Keypad: 16 sealed discrete switches | Holds, pending schematic review |
| D-017 | Power chain and protection | Holds, pending schematic review |

## D-001 · Take real payments through a licensed payment gateway

*2 October 2026 · decided by the owner*

The device takes real money through a payment gateway account opened in the owner's name. All
development happens in the gateway's test mode. The finished demo takes real ₹1 payments.

**Why:** a plain QR that pays a personal UPI ID can't tell the shopkeeper who paid. The only
workarounds read bank SMS or e-mails, which break easily. A gateway reports every payment through
a signed notification and lets us query its status, and that confirmation is the core of the
project.

**Given up:** the owner must complete the gateway's KYC, and the gateway may charge a fee on each
payment.

## D-002 · Custom circuit board and a 3D-printed case

*2 October 2026 · decided by the owner*

The hardware is our own board designed in KiCad, not a development kit wired to modules. The
owner's hardware rules apply:

- reliability and safety first
- then the fewest parts and the simplest circuit
- then parts in stock in India
- every rating taken from guaranteed datasheet limits
- the design checked on paper before layout, with no bench prototypes

**Why:** a dev-kit build shows less engineering and doesn't look or behave like a real product.

**Given up:** more design time, and every mistake costs a board revision.

## D-003 · Wi-Fi only, no cellular

*2 October 2026 · decided by the owner*

The device connects over the ESP32's built-in Wi-Fi. A shop without Wi-Fi can use a phone hotspot.

**Why:** fewer parts and a simpler power supply. A 4G module would add a SIM, a data plan, an
antenna and current bursts of about 2 A.

**Given up:** commercial soundboxes work in shops with no Wi-Fi. This one won't, unless a later
revision adds cellular.

## D-004 · Public repository from the first commit

*2 October 2026 · decided by the owner*

All work is public on GitHub from day one.

**Why:** a project history that anyone can follow is part of the portfolio.

**Consequence:** no secret may ever be committed. Gateway keys, Wi-Fi passwords and device keys
live only in local files that git ignores, or in the hosting platform's secret store. Secret
scanning and push protection are enabled on the repository.

## D-005 · The device draws the QR itself; the gateway keys never leave the backend

*2 October 2026 · follows from D-001 and the security goals*

The backend gets a raw `upi://pay?...` string from the gateway, and the device draws the QR from
that string on its own screen.

**Why:**

- A QR drawn from the string is sharp at any screen size.
- The device downloads a short string, not an image.
- The gateway's API keys stay on the server, where a stolen device can't expose them.

**Open:** whether the chosen gateway returns the raw string or only an image. This is part of the
gateway research in [`research/payment-gateways.md`](research/payment-gateways.md).

## D-006 · Razorpay is the payment gateway

*2 October 2026 · decided by the owner*

The owner signs up with Razorpay as "Individual / Unregistered", then asks support to enable QR
Codes and the `qr_image_content` flag. The backend creates single-use, fixed-amount `upi_qr` codes
and confirms payment only through the signed `qr_code.credited` webhook.

**Why:** Razorpay is the only gateway whose own documentation shows all of these together:
- individual onboarding, with the website step skippable
- single-use, fixed-amount UPI QR codes
- the raw `upi://` string
- HMAC-SHA256 signed webhooks
- RBI authorisation for in-person payments

See [`research/payment-gateways.md`](research/payment-gateways.md).

**Given up:** test-mode QR codes can't be scanned, so the first full test is a live ₹1 payment.

**Fallback:** if Razorpay refuses the QR features, register on Udyam and apply to PhonePe Offline
Dynamic QR or PayU DBQR. Both return the raw string.

## D-007 · No battery

*2 October 2026 · decided by the owner*

The board has no lithium cell. For power cuts, the device runs from any 5 V USB-C source, such as
a power bank or the USB port of a router mini-UPS that also keeps the Wi-Fi up.

**Why:** when mains fails, the shop's router usually fails too, so a battery in the device alone
can't take payments. Leaving it out saves 12–20 parts, a BIS-certified cell, a charger that must
stop charging above 45 °C, and the only fire risk the product would otherwise have.

## D-008 · On weak USB power, run limited and warn

*2 October 2026 · decided by the owner*

The firmware reads the CC pins. If the source advertises only Default USB power (500 mA), as an old
USB-A charger with an A-to-C cable does, the device keeps working with the backlight dimmed and
the volume capped. It shows "Use a 5 V USB-C adapter for full volume".

**Why:** nobody is locked out, and the warning steers them towards the right adapter.

**Given up:** Wi-Fi transmit peaks may briefly exceed 500 mA on such a source. The reset
supervisor (TLV809E) handles any brownout cleanly.

## D-009 · The ESP32-S3's on-chip watchdogs count as hardware watchdogs

*2 October 2026 · decided by the owner*

There is no separate watchdog chip. The chip's own watchdog timers (one RTC watchdog and two main
system watchdogs) reset the system if the firmware hangs.

**Firmware rule:** the firmware never disables them, and the bootloader watchdog stays on.

**Why:** these timers run independently of the CPU. The dangerous firmware faults are already
contained by other hardware:
- the eFuse limits current
- the speaker survives the amplifier's worst DC output
- the amplifier switches itself off whenever the MCU resets

A hang costs only availability, and the watchdog recovers it.

## D-010 · 66 mm speaker

*2 October 2026 · settled by the owner's rule "reliability first"*

The speaker is the PUI AS06608PS-R (66 mm, 8 Ω, 5 W maximum).

**Why:** if the firmware fails, the amplifier can put up to 4.45 W of DC or square wave into the
speaker. The 66 mm speaker survives that. The 40 mm alternative is rated 4 W maximum and would not.

**Given up:** a bigger, heavier case (204 g speaker).

## D-011 · Keep ESD protection on the keypad

*2 October 2026 · settled by the owner's rule "reliability first"*

Two TPD4E05U06 arrays protect the keypad rows and columns, in addition to the switches' ground
terminals.

**Why:** the ESP32's pins are rated for only 2 kV of static (human-body model), and Omron gives no
static rating for the switches' ground terminal. Paper can't close that risk without the arrays.

**Given up:** two extra parts.

## D-012 · Pause Razorpay; develop on a sandbox that needs no merchant KYC

*3 October 2026 · agreed with the owner · replaces D-006 and the live-account part of D-001*

Nobody opens a live merchant account for this project. Development moves to whichever gateway
sandbox provides all three of these without merchant KYC:

- a per-sale, fixed-amount UPI QR code returned as a raw `upi://` string
- a way to simulate the customer's payment
- a signed payment notification

Candidates to check: Setu, Cashfree, PhonePe and Decentro. The owner may also ask Razorpay support
whether it will enable QR Codes, `qr_image_content` and the BharatQR test-payment route on a
test-mode account before KYC.

**Why:**

- **Razorpay's test mode can't support development.** A fresh test account does not have the QR
  Codes product. First-hand developer logs from August 2026 show the QR Codes endpoint answering
  "URL not found" on a plain test account, and the only server-side test-payment route needs a
  support ticket. See section 10 of
  [`research/payment-gateways.md`](research/payment-gateways.md). D-006 assumed test mode would
  cover development, and that was wrong.
- **A live account is a merchant account.** KYC asks what you sell, and RBI requires the gateway to
  check that payments match the business declared. The owner doesn't run a shop, so describing
  "counter collection" would be untrue. An earlier suggestion in this project said exactly that,
  and is withdrawn here. Paying your own merchant account has no sale behind it.

**Still holds from D-001:** a payment is confirmed only by the gateway's signed notification, and
there is never a fallback to a personal UPI ID.

**Live demo, to decide later.** One of:

- **(a) Sandbox-only demo,** clearly labelled as test mode.
- **(b) Pilot in a real shop,** whose owner opens the account in their own name. Their business is
  real, so the description is true.
- **(c) The owner's own account,** described truthfully as "individual developer testing a UPI
  payment terminal", if a gateway approves it knowing that.

**Facts checked on 3 October 2026 for the record:**

- **Razorpay charges no setup, annual or monthly fee.** Fees are deducted from each payment: "UPI
  QR (standard)" 0.99%, or a 2% "platform fee" line (which one applies is unverified), plus 18% GST.
- **KYC processing** may cost ₹199 + tax.
- **Test mode** is free and needs no KYC.

## D-013 · MCU: ESP32-S3-WROOM-1-N16

*2 October 2026 · proposed in hardware research, accepted in review*

**Why:**

- It is the only candidate with enough pins. The board needs 22 GPIO; the C3 has 15, and the C6
  has 23 only by using its strapping pins.
- It has Secure Boot v2, XTS-AES-256 flash encryption and hardware crypto.
- Native USB means no USB-to-serial chip.
- 16 MB of flash holds two 3 MB firmware slots plus two copies of the voice pack (about 3.8 MB of
  English and Hindi audio).

**No PSRAM:** the octal-PSRAM variants are rated only to 65 °C ambient, against 85 °C for the N16
(module datasheet Table 1-1), and nothing in the design needs PSRAM.

See [`research/hardware-components.md`](research/hardware-components.md) section 1.

## D-014 · Display: 2.8 in IPS panel, with one resistor per backlight LED

*2 October 2026 · proposed in hardware research; backlight drive changed in review*

The display is the Newhaven NHD-2.8-240320AF-CSXP-F (2.8 in IPS, 240×320, 4-wire SPI) on a Molex
54132-4062 FPC connector.

**Why this panel:**

- It has a real datasheet with min/max limits.
- QR codes up to version 8 (payloads up to 192 bytes) get 0.72 mm modules, readable by a 1080p
  phone camera at about 50 cm.
- E-paper was rejected: a full refresh takes 5 s, the maker recommends at least 180 s between
  refreshes, and it is rated only 0–50 °C.

**Changed in review:** each of the four backlight LED cathodes gets its own 130 Ω resistor,
replacing the draft's single shared 22 Ω.

- Parallel LEDs with no resistor of their own don't share current: the one with the lowest
  forward voltage takes the most and ages first.
- The datasheet gives two different current figures (100 mA and 160 mA) and no maximum, so the
  design respects the lower one until Newhaven confirms otherwise.

**Given up:**

- **Single source.** The panel is stocked only at DigiKey India, with an 18-week lead time.
- **Brightness.** About 390 cd/m² nominal, against up to 1000. If Newhaven confirms 160 mA, only
  the resistor value changes; the footprint stays the same.

**Requirement on the backend:** the QR payload must be at most 192 bytes.

## D-015 · Audio: MAX98357A amplifier into an 8 Ω speaker

*2 October 2026 · proposed in hardware research, accepted in review*

**Why:**

- **Simple interface.** Audio goes in over I2S, with no master clock and no register setup.
- **Silent when the MCU resets.** The amplifier's own SD_MODE pull-down holds it off.
- **8 Ω, not 4 Ω.** With 8 Ω the worst-case fault current is 0.84 A, inside the amplifier's
  ±1.6 A absolute maximum; a 4 Ω speaker would exceed it.

The speaker choice itself is D-010.

**Given up:** the amplifier's output power is specified only as a typical value, so the loudness
margin rests on typical figures.

## D-016 · Keypad: 16 sealed discrete switches

*2 October 2026 · proposed in hardware research, accepted in review*

The keypad is a 4×4 matrix of 16 Omron/Aratas B3W-4150 switches (IP67, rated for at least
3 million presses, with a ground terminal), B32 key tops, and 2.7 kΩ column pull-ups.

**Why:**

- **The Grayhill sealed keypad was rejected:** it is rated only IP42, has hex legends, and costs
  ₹8,246 against about ₹1,970 for 16 switches and caps.
- **Capacitive pads were rejected:** water causes false touches, and the chip has only 14 touch
  channels.
- **The pull-ups** keep contact current at 1 mA or more, inside the silver contacts' 1–50 mA
  rating.

**Given up:** 16 holes in the case top. The spill path is handled in the case design: a raised
key island, a drip gutter, and possibly a conformal coat.

## D-017 · Power chain and protection

*2 October 2026 · proposed in hardware research, accepted in review*

The chain from USB-C to the 3.3 V rail:

- **USB-C sink:** a 5.1 kΩ Rd resistor on each CC pin. The ADC reads the CC voltage to learn
  the source's current limit (D-008).
- **TPS259531 eFuse:**
  - 2 A current limit.
  - Overvoltage clamped to 5.7 V; input rated to 20 V.
  - Its EN pin goes through a 200k/100k divider, because EN's absolute maximum is 7 V.
- **TPS62162 3.3 V buck** with a Murata DFE252012F-2R2M=P2 inductor, replacing TI's reference
  inductor, which is obsolete.
- **TLV809EA29 reset supervisor** on CHIP_PU. Espressif recommends one where power is cut often.
- **TPD4E05U06 ESD protection** on the USB lines.

**Left out on purpose:**

- **A VBUS TVS diode.** A 5–6 V TVS would turn a survivable 9–18 V adapter fault into a destroyed
  diode.
- **A USB-to-serial chip and BOOT/RESET buttons.** USB Serial/JTAG enters download mode by
  itself, and a GPIO0 test pad covers recovery.
- **An LDO.** It would dissipate 1.14 W.

**Adapter:** a 5 V USB-C adapter that advertises at least 1.5 A. Worst-case draw is 1.40 A, even
if the firmware fails.

**Residual risk:** ESD protection on VBUS rests on a capacitor-charge calculation, not a datasheet
rating, and must be confirmed in testing.
