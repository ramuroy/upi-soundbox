# upi-soundbox

An open-source payment terminal for shop counters in India. The shopkeeper types in an amount, the
device shows a fresh UPI QR code for exactly that amount, and a speaker announces the payment as
soon as it is confirmed.

> **Status: design phase, started 2 October 2026.** Nothing has been built or tested yet. This page
> describes the goal. Results will be added only once they are real and anyone can reproduce them.

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
2. The backend asks the payment gateway for a single-use QR with that exact amount and an expiry
   time.
3. The device draws the QR on its own screen.
4. The customer pays with any UPI app.
5. The gateway notifies the backend. The backend checks the notification's signature, records the
   payment once, and sends the device an authenticated message.
6. The speaker announces the amount received.

## Security goals

- The payment gateway's API keys live only on the backend. The device never sees them.
- Every sale gets its own short-lived QR on a screen, so there is no sticker to cover.
- The backend accepts a gateway notification only if its signature is valid, and processes each
  payment exactly once.
- The device accepts only authenticated messages from the backend, so a forged message cannot make
  it announce a payment.
- Secure Boot and flash encryption on the ESP32, so the firmware and stored credentials can't be
  swapped or read out.

## Hardware (planned)

A custom circuit board designed in KiCad, inside a 3D-printed case, on Wi-Fi, powered over USB-C.
Parts are being chosen from datasheet limits and Indian stock; the notes are in
[`docs/research`](docs/research).

## Roadmap

1. **Research and design.** Choose the gateway and the parts, and fix the architecture. *(in progress)*
2. **Backend** running against the gateway's test mode.
3. **Circuit board** designed and checked on paper, then fabricated.
4. **Firmware** for the ESP32.
5. **Case** design.
6. **Live demo** with real ₹1 payments.

Every design decision and the reason behind it are logged in [`docs/decisions.md`](docs/decisions.md).

## Licence

Apache License 2.0. See [`LICENSE`](LICENSE).
