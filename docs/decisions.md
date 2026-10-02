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
