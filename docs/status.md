# Project status

*Last updated: 3 October 2026*

**Where things stand.** The research phase is done. Both research reports were checked at their
primary sources, 17 decisions are recorded, and the main parts of the circuit board are chosen. One
question blocks the server work: which payment gateway's sandbox lets us create a per-sale UPI QR
and simulate its payment without merchant KYC. Razorpay is paused (D-012). Nothing has been built
or tested yet.

Read alongside this:
- [`decisions.md`](decisions.md): every decision with its reasons, and a status table.
- [`research/payment-gateways.md`](research/payment-gateways.md): gateway comparison, NPCI and RBI
  rules, the test-mode finding (section 10), Razorpay pricing.
- [`research/hardware-components.md`](research/hardware-components.md): part choices, power
  budget, QR-size calculation, India stock, draft BOM.

---

## Done

### 2 October 2026

- **Scope agreed.** The project is a counter device and its backend:
  - The device takes the amount on a keypad, draws a fresh UPI QR for each sale on its own screen,
    and announces the payment on a speaker.
  - The backend confirms payments only through the gateway's signed notification.
- **Founding decisions D-001 to D-005:**
  - a licensed gateway
  - a custom circuit board and 3D-printed case
  - Wi-Fi only
  - a public repository
  - the device draws the QR and the gateway keys stay on the server
- **Repository.** Created public at github.com/ramuroy/upi-soundbox, with secret scanning and push
  protection on, and `.gitignore` covering `.env`, keys and KiCad local files.
- **Gateway research.** Ten providers were compared against four needs: individual onboarding,
  a single-use fixed-amount UPI QR, the raw `upi://` string, and signed notifications. NPCI and
  RBI rules were summarised. Razorpay's load-bearing claims were checked a second time at source.
  Razorpay was chosen (D-006).
- **Hardware research.**
  - Parts were chosen from guaranteed datasheet limits only.
  - Stock was checked at DigiKey India, LCSC and Evelta. LCSC look-alike parts are flagged, and
    the obsolete TI reference inductor was replaced.
  - Key ratings were re-checked in the primary datasheets.
- **Review fix.** The display backlight's four LEDs had shared one resistor. Each now has its own,
  sized to the datasheet's lower current figure (D-014).
- **Owner decisions D-007 to D-011:**
  - no battery
  - run limited on a weak USB source
  - on-chip watchdogs are enough
  - 66 mm speaker
  - keep the keypad ESD arrays

### 3 October 2026

- **Razorpay costs checked.** No setup, annual or monthly fee. Fees are taken from each payment
  (0.99% or 2%, plus GST), and KYC may cost ₹199 + tax.
- **Account purpose questioned by the owner.** A live account is a merchant account and needs a
  real business behind it. The earlier advice to describe "counter collection" was withdrawn.
- **Test mode checked.** First-hand developer logs from August 2026 show that a fresh Razorpay test
  account has no QR Codes product. Razorpay is paused, and the gateway choice is reopened (D-012).
- **Docs brought up to date.** Hardware decisions D-013 to D-017 recorded, all docs updated, and
  this status page added.

---

## What's left, in order

1. **Gateway sandbox research.** Find a sandbox that provides all of these without merchant KYC:
   - a per-sale, fixed-amount UPI QR returned as a raw `upi://` string
   - a simulated payment
   - a signed notification

   Candidates: Setu (its documented sandbox has a mock-payment trigger), Cashfree, PhonePe and
   Decentro. Confirm each one with first-hand evidence, not documentation alone.
2. **Architecture document:**
   - the device-to-backend protocol
   - authenticated messages, so a forged "payment received" can't make the speaker announce
   - where the backend is hosted
   - handling a payment that lands after the QR is closed
   - Wi-Fi setup and device provisioning
   - over-the-air updates
3. **Schematic** in KiCad, checked on paper against the datasheets. The GPIO map is drafted in the
   hardware report, section 1.5.
4. **Backend** against the chosen sandbox:
   - order creation and cancel/expiry
   - signature-checked notifications, de-duplicated so each payment counts once
   - pushing an authenticated message to the device
5. **PCB layout,** then fabrication and assembly at PCB Power.
6. **Firmware:**
   - keypad, QR rendering and the voice announcer
   - the CC-pin power policy (D-008)
   - Secure Boot, flash encryption and OTA
7. **Case:**
   - the keypad drip island
   - at least 15 mm clearance around the antenna
   - the speaker mount
   - the FPC path
8. **Live demo.** Choose one of the D-012 options: sandbox-only, a real shop pilot, or a
   truthfully described account of the owner's own.

## Open questions

| Question | How it gets answered |
|---|---|
| Maximum backlight current of the NHD-2.8 panel (100 or 160 mA) | Owner e-mails Newhaven. A draft can be prepared on request. |
| Stock at Robu, element14 India and Mouser India | Check in the owner's browser; ask which browser first. |
| Stock of the SXH-001T-P0.6 crimp contacts | Same browser check |
| Can PCB Power assemble the leadless packages (WSON, USON, TQFN)? | Check with PCB Power |
| Payload length of the chosen gateway's QR string (must be at most 192 bytes) | First call in its sandbox |
| Will Razorpay enable QR Codes on a test-mode account before KYC? | Optional question to Razorpay support. A draft can be prepared on request. |
| Does NPCI require a QR error-correction level? | Nothing found yet. Ask the chosen gateway. |
| Conformal coat on the keypad zone? | Decide at case design |

## Owner action items

- *(Optional)* Ask Razorpay support the question above.
- *(When ready)* E-mail Newhaven about the backlight's maximum current.
- Do not start any merchant KYC for this project (D-012).

## How to resume

1. Read this page, then the status table at the top of [`decisions.md`](decisions.md).
2. Continue from step 1 of "What's left".
3. Never commit secrets. Gateway keys go in a local `.env` file, which git ignores.
