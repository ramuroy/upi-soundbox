# Decision log

Each entry records what was decided, why, and what was given up. Newest entries go at the bottom.
A decision is changed by adding a new entry that replaces it, never by editing the old one.

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
