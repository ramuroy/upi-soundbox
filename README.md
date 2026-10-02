# upi-soundbox

An open-source payment terminal for shop counters in India. The shopkeeper types in an amount, the
device shows a fresh UPI QR code for exactly that amount, and a speaker announces the payment as
soon as it is confirmed.

> **Status: design phase (started 2 October 2026).** Research is done and the main circuit-board
> parts are chosen. Nothing has been built or tested yet. The current state and the remaining work
> are in [`docs/status.md`](docs/status.md). Results will be added only once they are real and
> anyone can reproduce them.

## Why build it

A printed UPI QR sticker on a shop counter has three problems:

- The customer types the amount by hand, so mistakes are easy.
- The shopkeeper has to check a phone to see whether, and by whom, a payment arrived.
- A fraudster can paste their own QR sticker over the shop's, and a customer can flash a fake
  "payment successful" screen.

This project builds the whole chain in the open: the circuit board, the firmware, the server and
the case.

## How it will work

```
 keypad ──► device (ESP32) ──HTTPS──► backend ──API──► payment gateway
              │  draws the QR            ▲  │                │
              │  on its screen           │  └─◄── webhook ◄──┘  (signature checked)
              ▼                          │
           speaker ◄── "paid" message ───┘  (authenticated)

 customer's UPI app scans the on-screen QR and pays the gateway
```

1. The shopkeeper types the amount.
2. The backend asks a licensed payment gateway for a single-use QR with that exact amount and an
   expiry time.
3. The device draws the QR on its own screen.
4. The customer pays with any UPI app.
5. The gateway notifies the backend. The backend checks the notification's signature, records the
   payment once, and sends the device an authenticated message.
6. The speaker announces the amount received.

The gateway is being chosen. Razorpay was first choice but is paused, because its test mode doesn't
include QR codes. The search is now for a sandbox that offers per-sale UPI QR codes without
merchant KYC (decision D-012).

## Security goals

- The payment gateway's API keys live only on the backend. The device never sees them.
- Every sale gets its own short-lived QR on a screen, so there is no sticker to cover.
- The backend accepts a gateway notification only if its signature is valid, and processes each
  payment exactly once.
- The device accepts only authenticated messages from the backend, so a forged message cannot make
  it announce a payment.
- Secure Boot and flash encryption on the ESP32, so the firmware and stored credentials can't be
  swapped or read out.

## Hardware (chosen, not yet built)

A custom KiCad board in a 3D-printed case, on Wi-Fi, powered from USB-C. Every rating comes from
guaranteed datasheet limits, and every part is in stock in India.

| Function | Part |
|---|---|
| MCU | ESP32-S3-WROOM-1-N16 (Secure Boot v2, flash encryption, native USB) |
| Display | Newhaven 2.8 in IPS, 240×320, SPI. Shows QR codes at 0.72 mm per module. |
| Audio | MAX98357A I2S amplifier, 66 mm 8 Ω speaker |
| Keypad | 16 sealed (IP67) Omron switches |
| Power | USB-C with current-limit sensing, TPS259531 eFuse (2 A limit, 5.7 V overvoltage clamp), TPS62162 3.3 V buck, TLV809E reset supervisor |

Details, the power budget and the draft BOM are in
[`docs/research/hardware-components.md`](docs/research/hardware-components.md).

## Roadmap

| Step | State |
|---|---|
| 1. Research and design | **Done** for the gateway comparison and the parts. **Next:** choose a gateway sandbox, then write the architecture document. |
| 2. Backend against a gateway sandbox | Not started |
| 3. Circuit board: schematic checked on paper, then layout and fabrication | Parts chosen; schematic not started |
| 4. Firmware for the ESP32 | Not started |
| 5. Case | Not started |
| 6. Live demo | Approach to be chosen (D-012) |

## Documents

- [`docs/status.md`](docs/status.md): what is done, what is left, open questions.
- [`docs/decisions.md`](docs/decisions.md): every decision with its reasons, and which ones still
  hold.
- [`docs/research/payment-gateways.md`](docs/research/payment-gateways.md): Indian gateways, NPCI
  and RBI rules.
- [`docs/research/hardware-components.md`](docs/research/hardware-components.md): part choices from
  datasheet limits.

## Licence

Apache License 2.0. See [`LICENSE`](LICENSE).
