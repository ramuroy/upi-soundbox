# Payment gateways for upi-soundbox

Research date: 2026-10-02. Method: official gateway documentation, RBI and NPCI pages, read with WebSearch/WebFetch. No browser, no sign-ups, no logins.

Notes on sources:
- Every claim carries its source URL. Anything not confirmed from a source is marked **UNVERIFIED**.
- npci.org.in blocks automated fetches (HTTP 403 or an empty page). So the contents of NPCI circulars are taken from compliance trackers (TaxGuru, TeamLease RegTech) or news, and are marked as secondary.
- Documentation changes often. Re-check each gateway's docs before relying on a field name.
- **Second check, 2026-10-02.** The claims the Razorpay recommendation rests on were re-read at their sources and confirmed:
  - Individual/Unregistered onboarding and its documents.
  - QR Codes enabled on request, and the `qr_image_content` flag.
  - The `image_content` example string, and fixed-amount enforcement.
  - The `close_by` conflict (2 min on the create page, 15 min on the image-content pages).
  - "Only Live Mode" QR scanning.
  - Webhook HMAC-SHA256 over the raw body, with `x-razorpay-event-id` de-duplication.
  - The 15 Oct 2026 UPI MDR rule.

---

## 1. Short answer

> **Status, 3 October 2026: the Razorpay recommendation below is on hold.**
> - A fresh Razorpay test account does not have the QR Codes product, so it cannot support development (section 10).
> - The owner should not open a live merchant account just for this project (decision D-012 in [`../decisions.md`](../decisions.md)).
>
> The comparison below is still accurate. The next step is to find a sandbox that offers per-sale UPI QR codes without merchant KYC.


**Apply to Razorpay first.** It is the only gateway found that meets all four needs:
- (a) It documents self-serve onboarding for an "Individual / Unregistered" business with PAN, Aadhaar and a bank account.
- (b) It lets you skip the website at sign-up.
- (c) Its QR Codes API creates a **single-use, fixed-amount UPI QR** with an expiry.
- (d) It can return the **raw `upi://pay?...` string** in the `image_content` field.

There are two catches:
- **Both features are enabled on request.** The QR Codes product needs a request, and the raw string needs the extra `qr_image_content` flag. If the flag is refused, the backend can decode the returned QR image instead.
- **Test-mode QR codes cannot be scanned.** The end-to-end check has to be a live ₹1 payment.

**Second choice: PhonePe Offline Dynamic QR (DQR).** It fits the design best technically: built for offline merchants, raw `qrString`, a UAT simulator app, and expiry up to one month. But credentials come through PhonePe's sales team, and whether an individual is eligible is UNVERIFIED.

**Third choice: PayU's DBQR API.** It returns a raw `qrString` and PayU accepts an "Individual" entity. The DBQR flag has to come from a key account manager, and PayU warns that its test-environment QR flow is unreliable.

**Not suitable for this owner, or a poor fit:**
- Cashfree: needs a live website or app.
- Paytm Dynamic QR: enterprise customers only.
- Easebuzz: needs a website plus business proofs.
- Instamojo: no QR API, and the minimum payment is ₹9.
- Decentro: the minimum is ₹5.
- Setu and Pine Labs: onboarding is sales-led.

---

## 2. Comparison table

| Gateway / product | Individual (unregistered) onboarding | Website or app needed | Single-use fixed-amount QR via API | Raw `upi://` string | Expiry control | Test-mode QR payment | UPI fee to merchant | RBI PA-P authorised? |
|---|---|---|---|---|---|---|---|---|
| **Razorpay** QR Codes API | Yes ([docs][rzp-kyc]) | No ("Add later") ([docs][rzp-setup]) | Yes (`usage: single_use`, `fixed_amount: true`) ([docs][rzp-create]) | Yes (`image_content`, feature flag) ([docs][rzp-ic-entity]) | `close_by`; docs conflict: min 2 min, max 2 h ([create][rzp-create]) vs min 15 min ([image-content][rzp-ic-create]) | **No**: "only Live Mode" QRs can be scanned ([FAQ][rzp-faq]) | 2% "platform fee" + GST, MDR 0; also a 0.99% "UPI QR" line. Conflicting, so UNVERIFIED ([pricing][rzp-pricing]) | Yes ([RBI list][rbi-list]) |
| **PhonePe** Offline DQR | UNVERIFIED (via sales; guidelines list Sole Proprietor) | UNVERIFIED | Yes (`amount`, `expiresIn`, single scan) ([init][pp-init], [FAQ][pp-faq]) | Yes (`data.qrString`) ([init][pp-init]) | `expiresIn` (s), up to 1 month ([FAQ][pp-faq]) | Yes (PhonePe Simulator app) ([getting started][pp-start]) | UNVERIFIED | Yes ([RBI list][rbi-list]) |
| **PayU** DBQR | Yes, "Individual" ([docs][payu-docs]) | No website limits you to links, invoices and buttons ([docs][payu-kyc]) | Yes (`pg=DBQR`, `expiry_time`) ([docs][payu-dbqr]) | Yes (`qrString`) ([docs][payu-dbqr]) | `expiry_time` (s), default 30 min | Doc warns test QR "may show failures" ([docs][payu-qr-idx]) | Not published | Yes ([RBI list][rbi-list]) |
| **Cashfree** Offline (softPOS terminal) | Yes, "Individual" ([FAQ][cf-faq]) | **Yes, a live site or app** ([FAQ][cf-faq]) | Yes (order + terminal txn `QR_CODE`) ([docs][cf-term]) | No: base64 PNG only (decodable) ([docs][cf-term]) | `order_expiry_time` ([docs][cf-order]) | UNVERIFIED | 1.95% + GST; festive 0% offer ([pricing][cf-pricing]) | Yes ([RBI list][rbi-list]) |
| **Paytm** Dynamic QR | PAN + bank account with a ₹50k/month cap ([support][ptm-noreg]) | Not for payment links | Yes, but **"select enterprise customers" only** ([docs][ptm-dqr]) | Yes (`qrData`) ([API][ptm-create]) | Default 10 min | Not documented | "0.00%" MDR on UPI ([pricing][ptm-pricing]) | Yes ([RBI list][rbi-list]) |
| **Easebuzz** seamless UPI QR | No individual type; proprietor needs 2 business proofs or CPV ([docs][ezb-kyc]) | **Yes** ([docs][ezb-kyc]) | Fixed amount yes; expiry not documented | Yes (`qr_link`) ([docs][ezb-seamless]) | UNVERIFIED | Not documented | "Customized pricing" | Yes ([RBI list][rbi-list]) |
| **Instamojo** | Yes (PAN + bank) ([blog][im-kyc]) | No | No QR API; hosted checkout only; **min ₹9** ([docs][im-pr]) | No | `expires_at` ≤ 600 s | No UPI simulation | 2% + ₹3 + GST ([pricing][im-pricing]) | **Not listed** ([RBI list][rbi-list]) |
| **Setu** UPI DeepLinks | UNVERIFIED (sales-led) | UNVERIFIED | Yes (`amountExactness: EXACT`) ([docs][setu-qs]) | Yes (`upiLink`) | `expiryDate` | Yes (mock-credit API) ([docs][setu-qs]) | Not published | Setu is not listed; it runs under Pine Labs' PA ([support][setu-pa]) |
| **Decentro** dynamic QR | "Sole Proprietorship, Individuals", but asks for GST/Udyam ([docs][dec-onb]) | Not stated | Yes; **min ₹5** ([docs][dec-qr]) | QR API: image link only; payment-link API: raw `common_uri` ([docs][dec-link]) | `expiry_time` ≤ 1440 min | Staging testbed | UNVERIFIED | Yes ([RBI list][rbi-list]) |
| **Pine Labs Online** | UNVERIFIED | UNVERIFIED | Order + UPI QR pay | Yes (`challenge_url`) ([docs][pl-upi]) | Not documented | UNVERIFIED | UNVERIFIED | Yes ([RBI list][rbi-list]) |

---

## 3. Per-gateway notes

### 3.1 Razorpay (recommended first)

**Onboarding**
- There are 13 business types; the first is "Individual/Unregistered Businesses". https://razorpay.com/docs/payments/business-types-kyc-documents/
- Documents for an unregistered individual: PAN and mobile number (or a CKYC auto-fetch) for identity, and DigiLocker / Aadhaar / photo ID / passport for address. Bank account number and IFSC are mandatory. If bank verification fails, a video of a cancelled cheque is asked for. Same source.
- A proprietorship instead needs at least 2 business documents (MSME/Udyam, GST, Shop & Establishment, ...). Same source.
- Website/app step: "If you wish to add these later, click **Add later**." https://razorpay.com/docs/payments/set-up/
- A KYC processing fee of ₹199 + tax "may" be charged, subject to document verification. https://razorpay.com/terms/90-day-free-pg-offer/
- Per-transaction or volume limits for unregistered accounts: **UNVERIFIED**. A ₹40,000 per-transaction figure appeared only in a search snippet.

**Gating**
- QR Codes is an on-demand feature: "Please raise a request with our Support team to get this feature activated on your account." https://razorpay.com/docs/api/qr-codes/create/
- The raw string is behind a second flag: "once the `qr_image_content` feature is enabled, you can get the Create QR Code response…" with `image_content`. https://razorpay.com/docs/payments/qr-codes/apis/
- Whether Razorpay grants either flag to an *unregistered* account: **UNVERIFIED**. Ask support at activation.

**Offline use**
- The product page markets QR codes for checkout counters: "Display QR codes at checkout for quick, touch-free mobile payments." https://razorpay.com/qr-code/
- Razorpay Payments Pvt Ltd is authorised as PA-O, PA-P and PA-CB on the RBI list. https://rbi.org.in/Scripts/PublicationsView.aspx?id=12043
- Razorpay announced the offline PA licence for Razorpay POS on 22 Jan 2026. https://razorpay.com/newsroom/razorpay-pos-receives-rbi-approval-for-offline-payment-aggregator-licence/
- Whether an online-onboarded individual may use the QR Codes API at a physical counter without extra merchant checks (such as CPV): **UNVERIFIED**.

**API**
- Create: `POST https://api.razorpay.com/v1/payments/qr_codes`, Basic auth (key_id:key_secret). https://razorpay.com/docs/api/qr-codes/create/
  - Fields: `type: "upi_qr"`, `name`, `usage: "single_use"`, `fixed_amount: true`, `payment_amount` (paise), `description`, `customer_id`, `close_by` (unix time), `notes` (max 15 pairs, 256 characters each).
  - Rule: "When `usage` is `single_use`, `fixed_amount` must be `true`."
- **Raw string.** `image_content` is "The link encoded to the payable QR Code using any QR Code generator". Example: `upi://pay?pa=dmart.razorpay@hdfcbank&pn=TestAccount&tr=RZPGT5viB4WHeoUuuqrv2&tn=TestAccountRaftarSoft&am=100&cu=INR&mc=5411`. https://razorpay.com/docs/api/qr-codes/image-content/entity/
  - Without the flag the response has only `image_url` (a short link to the PNG). https://razorpay.com/docs/api/qr-codes/create/
  - The backend could fetch that PNG and decode it (e.g. zbar/OpenCV) before sending the string to the device. That this works on Razorpay's branded image is **UNVERIFIED** until tried.
- **Amount enforcement.** For `payment_amount`: "any transaction of an amount less than or more than this value is not allowed." https://razorpay.com/docs/api/qr-codes/image-content/entity/
- **Minimum amount** (whether ₹1, 100 paise, is accepted for a QR): **UNVERIFIED**.
- **Expiry: the docs conflict.**
  - The create page says a minimum of 2 minutes, a maximum of 2 hours, and a 2-hour default. https://razorpay.com/docs/api/qr-codes/create/
  - The image-content pages say `close_by` must be "at least 15 minutes after the current time". https://razorpay.com/docs/api/qr-codes/image-content/create/
  - Safe choice: `close_by = now + 15 min`, plus an explicit Close call (below) when the shopkeeper cancels.
- **States.** `status` is `active` or `closed`. `close_reason` is `on_demand`, `paid` (auto-closed after a single-use payment) or `null`. `payments_amount_received` and `payments_count_received` are also returned. https://razorpay.com/docs/api/qr-codes/image-content/entity/
- **Other endpoints** (https://razorpay.com/docs/api/qr-codes/):
  - Close: `POST /v1/payments/qr_codes/:id/close`
  - Fetch: `GET /v1/payments/qr_codes/:id`
  - Payments on a QR: `GET /v1/payments/qr_codes/:id/payments`
  - Refund: `POST /v1/payments/:id/refund`
- **Webhooks.**
  - Events: `qr_code.created`, `qr_code.credited`, `qr_code.closed`. https://razorpay.com/docs/payments/qr-codes/subscribe-to-webhooks/
  - `qr_code.closed` fires only on a manual close, not on auto-expiry. The `qr_code.credited` payload carries `payment.entity` (amount, status `captured`, `method: upi`, `vpa`, `acquirer_data.rrn`) and `qr_code.entity`. https://razorpay.com/docs/webhooks/qr-codes/
- **Webhook signature.**
  - Header `X-Razorpay-Signature`: "HMAC with SHA256 algorithm; with your webhook secret set as the key and the webhook request body as the message" (hex digest in the code samples).
  - Validate against the raw body: "Do not parse or cast the webhook request body".
  - De-duplicate on `x-razorpay-event-id`.
  - Source: https://razorpay.com/docs/webhooks/validate-test/
  - Retry schedule: **UNVERIFIED**.
- **Rate limits.** Not published; the API returns 429. https://razorpay.com/docs/api/pagination/

**Test mode**
- "You can scan QR Codes that are created only in Live Mode." https://razorpay.com/docs/payments/qr-codes/faqs/
- A test "pay" endpoint (`POST /v1/bharatqr/pay/test`) is documented for BharatQR only. https://razorpay.com/docs/payments/payment-methods/bharatqr/testing/
- Whether it works for `upi_qr`: **UNVERIFIED**.
- Plan: build the backend against test mode (create, fetch, close, and webhook delivery via test triggers), then prove the full loop with live ₹1 payments.

**Cost**
- Setup fee ₹0 and AMC ₹0. https://razorpay.com/pricing/
- UPI fee: the docs contradict each other, so the actual rate on QR payments is **UNVERIFIED**. For a ₹1 demo the fee is at most a few paise.
  - The pricing page says "Zero MDR — 2% platform fee applies" and "UPI is MDR-free per RBI policy. Razorpay's 2% applies as a platform/technology fee". The same fetch also showed a "UPI QR (standard) 0.99%" line. https://razorpay.com/pricing/
  - An earlier fetch of the same page's embedded data said "UPI QR payments are typically at 0% MDR".
  - A 90-day 0% offer applies to accounts activated from 1 Jul 2026, up to ₹5 lakh GMV. https://razorpay.com/terms/90-day-free-pg-offer/
- Settlement:
  - The pricing page says "T+1 or instant". https://razorpay.com/pricing/
  - The settlements doc and a Feb 2026 blog say T+2. https://razorpay.com/docs/payments/settlements/ and https://razorpay.com/blog/razorpay-payment-gateway-pricing-explained/
  - Which applies: **UNVERIFIED**.

**Prior art**
- ESP32 UPI payment terminal with Razorpay: https://www.hackster.io/Himanshudada/esp32-upi-payment-terminal-with-razorpay-30e8ec
- C++ Razorpay dynamic-QR POS, MIT licence: https://github.com/hemangjoshi37a/Razorpay-Dynamic-QR-HMI-POS

### 3.2 PhonePe

**Offline Dynamic QR (DQR): the best technical fit**
- Positioned for "any merchant who has internet [and] a display". https://developer.phonepe.com/offline-integration/dynamic-qr-solution/introduction
- **Init.** https://developer.phonepe.com/offline-integration/dynamic-qr-solution/dqr-init-api
  - Endpoints: `POST https://mercury-t2.phonepe.com/v3/qr/init` (production) and `https://mercury-uat.phonepe.com/enterprise-sandbox/v3/qr/init` (UAT).
  - Body: `merchantId`, `transactionId` (<35 characters), `merchantOrderId`, `amount` (paise), `expiresIn` (seconds), `storeId` (≤38), `terminalId`, `message`, plus optional `gstBreakup` and `invoiceDetails`.
  - Header: `X-VERIFY = SHA256(base64Payload + "/v3/qr/init" + saltKey) + "###" + saltIndex`.
- **Raw string returned** in `data.qrString`. Sample: `upi://pay?pa=MERCHANTUAT@ybl&pn=...&am=10.00&mam=10.00&tr=TX...&tn=...&mc=5311&mode=04&purpose=00...`. Same source.
  - The `am` = `mam` pair locks the amount (see the NPCI spec, section 5).
- **Behaviour.** Maximum expiry is 1 month. "Customer can pay via any UPI app." Once a payment has been attempted the QR cannot be scanned again. States: SUCCESS / FAILED / PENDING / EXPIRED. https://developer.phonepe.com/offline-integration/faqs/dynamic-qr-flow
- **Status:** `GET /v3/transaction/{merchantId}/{transactionId}/status`. https://developer.phonepe.com/offline-integration/response-capturing-api/check-payment-status-api
- **Cancel:** `POST /v3/charge/{merchantId}/{transactionId}/cancel`. https://developer.phonepe.com/offline-integration/cancel-payment-request-api
- **Refund:** `POST /v3/credit/backToSource`. https://developer.phonepe.com/offline-integration/refund-flow/refund-api
- **Callback.** https://developer.phonepe.com/offline-integration/response-capturing-api/server-to-server-callback
  - Body: `{"response": "<base64 JSON>"}`.
  - Header: `X-VERIFY = SHA256(response + saltKey)###saltIndex`. This is a salted hash, **not an HMAC**.
  - Retry count conflicts: 2 or 3. https://developer.phonepe.com/offline-integration/faqs/general
- **Test:** UAT has moved to the "PhonePe Simulator" app. https://developer.phonepe.com/offline-integration/others/getting-started-with-your-api
- **Onboarding.**
  - Credentials (MID, salt key, salt index) come from PhonePe's business or sales team. Third-party source: https://www.zoho.com/en-in/pos/retail/resources/help/phonepe.html
  - PhonePe's merchant guidelines list Sole Proprietor, which needs two of: Udyam, Shop & Establishment, ITR, GST, a recent utility bill, and others. They list no "individual" type. https://www.phonepe.com/apollo/pdf/Merchant_Profiling_Guidelines.pdf
  - Whether an individual can get DQR access: **UNVERIFIED**.
  - Pricing and settlement: **UNVERIFIED**.
- **Authorisation:** PhonePe Ltd holds PA-O and PA-P on the RBI list (authorised 19 Sep 2025). https://rbi.org.in/Scripts/PublicationsView.aspx?id=12043

**PhonePe PG (online)**
- Standard Checkout returns a hosted redirect URL, not a raw QR. https://developer.phonepe.com/payment-gateway/website-integration/standard-checkout/api-integration/api-reference/create-payment
- A PhonePe article says freelancers can sign up as "Individual". https://business.phonepe.com/articles/payment-gateway-without-gst-do-you-need-registration-to-accept-payments
  - This conflicts with the guidelines PDF above.
- No public SmartSpeaker or soundbox API was found (**UNVERIFIED** that none exists).

### 3.3 PayU

**Onboarding**
- An "Individual" entity needs PAN, address proof (masked Aadhaar is accepted) and bank proof with the name matching the PAN. https://docs.payu.in/docs/documents-checklist-for-account-activation
- Choosing "I don't have a website/app" limits the account to Payment Links, Invoices and Buttons. https://docs.payu.in/docs/complete-your-kyc

**Gating**
- "UPI QR transactions require the `DBQR` flag to be activated on your merchant account... contact your KAM". https://docs.payu.in/reference/dynamic-qr-generation-api
- Whether this is available to an individual-tier account: **UNVERIFIED**.

**API** (https://docs.payu.in/reference/dynamic-qr-generation-api)
- Endpoint: `POST https://secure.payu.in/_payment` (form-encoded), with `pg=DBQR`, `bankcode=UPIDBQR`, `txn_s2s_flow=4`, `expiry_time` (seconds, default 30 min) and `udf1`–`udf5`.
- Hash: `sha512(key|txnid|amount|productinfo|firstname|email|udf1..udf5||||||SALT)`.
- Response includes `qrString`, e.g. `upi://pay?pa=...&pn=...&tr=...&tid=...&am=3223.00&cu=INR&tn=UPI Transaction`.
- The doc says it "returns a UPI QR which can be used for offline payment collections".

**Other QR endpoints**
- Invoice QR with `outputType=string`. https://docs.payu.in/reference/print-invoice-qr-api
- Cancel QR. https://docs.payu.in/reference/cancel-qr-transaction-api-1
- Avoid "Dynamic Storefront QR". It "is not a UPI-QR". https://docs.payu.in/docs/integrated-dynamic-storefront

**Callback, status and refunds**
- Callback: form POST with a reverse SHA-512 hash, `salt|status||||||udf5..udf1|email|firstname|productinfo|amount|txnid|key`. https://docs.payu.in/reference/transaction-callback-api
- Webhook source IPs are published. https://docs.payu.in/docs/webhook-events-and-sample-payloads
- Status: `command=check_bqr_txn_status`. https://docs.payu.in/reference/transaction-status-check-api-2
- Refund: `cancel_refund_transaction`. https://docs.payu.in/reference/refund_transaction_api

**Test mode**
- "UPI QR in the test environment may show failures that do not replicate in production." https://docs.payu.in/docs/apis-for-upi-qr-integration

**Cost**
- No setup or annual fee. UPI pricing "varies by business type and volume" (no published number). Settlement T+2. https://payu.in/pricing/

### 3.4 Cashfree

**Onboarding**
- "If you do not have a business PAN, register as an Individual using your personal PAN": PAN, Aadhaar and bank account. https://www.cashfree.com/docs/help/onboarding-related/onboarding-faqs
- **Blocker:** "Yes, you must have either a live website or an app published on the App Store or Play Store", with T&C, About, Contact, Privacy policy and business name. Same source.

**Main Order Pay QR**
- The QR channel of `POST /pg/orders/sessions` needs the S2S flag: "submit a GST return showing an annual turnover of more than Rs. 1 crore." https://www.cashfree.com/docs/api-reference/payments/latest/payments/pay
- It returns a base64 image, not the raw string.

**Offline Payments (softPOS) route**
- The FAQ confirms this route gives a dynamic QR without S2S: create a Terminal, create an Order with the terminal details, then create a Terminal Transaction with `payment_method: QR_CODE`. https://www.cashfree.com/docs/help/softpos/softpos
- It needs account-manager activation. Each collection point needs KYC; a storefront needs shop address proof and a shop-front photo. https://www.cashfree.com/docs/payments/softpos/collection-point-setup
- The response is a `qrcode` base64 PNG with `timeout`. https://www.cashfree.com/docs/api-reference/payments/latest/offline-payments/create-terminal-transaction
- The doc's sample PNG decodes to a `upi://pay?pa=...` string, so decoding works. That the sample lacks `am`/`tr` means it does not prove the live content (**UNVERIFIED**).

**Webhooks**
- Events include `PAYMENT_SUCCESS_WEBHOOK`. https://www.cashfree.com/docs/api-reference/payments/latest/payments/webhooks
- Signature: `x-webhook-signature` = Base64(HMAC-SHA256(`x-webhook-timestamp` + rawBody, clientSecret)). This includes a timestamp, so replays can be rejected. https://www.cashfree.com/docs/api-reference/webhooks/payloads-and-signatures

**Rate limits and cost**
- Rate limits are published, e.g. production Create Order 200/min. https://www.cashfree.com/docs/api-reference/payments/rate-limits
- UPI 1.95% + GST, with a festive 0% offer. Offline settlement T+2. https://www.cashfree.com/docs/help/account/pricing and https://www.cashfree.com/docs/help/softpos/softpos

### 3.5 Paytm (Paytm Payments Services Ltd)

**Onboarding**
- Possible with PAN and a bank account, but the monthly limit is ₹50,000 and settlement is held until business documents are submitted. https://business.paytm.com/support/i-dont-have-a-registered-business-can-i-still-get-the-payment-gateway

**Dynamic QR is enterprise-only**
- "Paytm Dynamic QR is only available to the select enterprise customers with high transaction volumes and established businesses." https://www.paytmpayments.com/docs/dynamic-qr-code-payments/

**API** (https://www.paytmpayments.com/docs/api/create-qr-code-api)
- Endpoint: `POST https://securegw.paytmpayments.in/paymentservices/qr/create`, with `businessType: "UPI_QR_CODE"`, `posId`, `amount`, `orderId` and `expiryDate`.
- Expiry defaults to 10 minutes.
- Returns `qrData` (raw `upi://pay?...`) and an optional `image`.

**Webhook**
- Form POST with `CHECKSUMHASH`. The checksum is SHA-256 with a salt, then AES-128-CBC with a fixed IV; it is not an HMAC. https://www.paytmpayments.com/docs/payment-status/ and https://www.paytmpayments.com/docs/checksum/

**Cost**
- "0.00%" MDR on UPI and zero setup fee. https://business.paytm.com/pricing
- PPSL holds PA-O, PA-P and PA-CB. https://rbi.org.in/Scripts/PublicationsView.aspx?id=12043

**Soundbox**
- Paytm Soundbox is a closed device with no developer API found (**UNVERIFIED**).

### 3.6 Easebuzz

**Onboarding**
- The KYC requires a "Website or Mobile App URL", with T&C, privacy policy and refund policy. https://docs.easebuzz.in/docs/get-started/6v6i0r0x14zly-kyc-documents
- There is no unregistered-individual type. A sole proprietor needs two business proofs, or contact point verification (CPV) with geotag and shop photos. Same source, and https://docs.easebuzz.in/docs/get-started/par15oupue6xy-complete-your-cpv
- Rejection grounds include "poor physical setup with no signboard". https://docs.easebuzz.in/docs/get-started/5vilf2q2arhvs-grounds-for-rejection

**API**
- Seamless UPI QR returns `qr_link` with the raw string, e.g. `upi://pay?pa=...&am=2.0&mam=2.0&...`. The doc says "Use the deeplink received in the response as-is." https://docs.easebuzz.in/docs/payment-gateway/k3ho860cy66zh-seamless-integration-merchant-hosted
- QR expiry is not documented.

**Webhook**
- Reverse SHA-512 hash. Retries every 30 minutes, 5 times. https://docs.easebuzz.in/docs/payment-gateway/paw9n1qc3kuoz-transaction-webhook

**Cost**
- "Customized pricing", settlement T+1. https://easebuzz.in/pricing/

**Soundbox and POS**
- Both exist, but are arranged through a relationship manager, with no API. https://docs.easebuzz.in/docs/payment-gateway/3539lksldq6ap-soundbox

### 3.7 Instamojo: not viable

- Onboarding with PAN and bank details; no website needed. https://www.instamojo.com/blog/how-to-update-kyc-instamojo/
- The support page failed TLS on fetch, so the full document list is **UNVERIFIED**.
- The Payment Request API returns a hosted checkout URL only, not a UPI QR or string. "The minimum amount is 9", which **rules out ₹1 demos**. https://docs.instamojo.com/reference/create-a-payment-request-1
- The webhook `mac` is HMAC-SHA1. https://docs.instamojo.com/reference/payments-api
- 2% + ₹3 + GST. https://www.instamojo.com/pricing/
- Not on the RBI PA list. https://rbi.org.in/Scripts/PublicationsView.aspx?id=12043

### 3.8 Other providers

**Setu UPI DeepLinks**
- `POST /payment-links` with `amountExactness: EXACT` and `expiryDate`. It returns `upiLink`, the "QR code source". https://docs.setu.co/payments/upi-deeplinks/quickstart
- The docs warn that exact-amount enforcement varies by UPI app. Same source.
- The sandbox has a mock-credit trigger, `POST /triggers/funds/addCredit`. Same source.
- Webhook signature for DeepLinks is not documented. UMAP uses `x-setu-signature`. https://docs.setu.co/payments/umap/notifications/verify-signature
- Merchants sign Pine Labs' PA agreement. https://support.setu.co/support/solutions/articles/81000410169-current-onboarding-requirement
- Onboarding for individuals and pricing: **UNVERIFIED**.

**Decentro**
- Accepts "Non-entities: Sole Proprietorship, Individuals", but the document list asks for GST/Udyam and a business PAN. https://docs.decentro.tech/docs/payments-collections-pa-onboarding
- The dynamic QR API returns an image link, **minimum ₹5**. https://docs.decentro.tech/reference/payments_api-collectionsv3-dynamicqr
- The payment-link API returns a raw `common_uri`. https://docs.decentro.tech/reference/payments_api-collectionsv3-paymentlink
- The callback has no HMAC; it uses a custom header token plus IP allow-listing. https://docs.decentro.tech/reference/payments_api-collectionsv3-statuscallback

**Pine Labs Online**
- The UPI QR flow returns `challenge_url` as the raw string. https://www.pinelabs.com/docs/online-payments/use-cases/accept-payments-upi
- Webhook headers `webhook-id`, `webhook-timestamp` and `webhook-signature`. https://www.pinelabs.com/docs/online-payments/developer-tools/webhooks/signature-verification
- Onboarding and pricing: **UNVERIFIED**.

**Juspay**
- An enterprise orchestrator. The QR API returns `sdk_params` and the merchant builds the string. https://docs.juspay.io/upi-qr-code/docs/scan--pay/transaction-api
- Not a fit.

**Zaakpay**
- QR comes back as a base64 PNG only. https://developer.zaakpay.com/docs/seamless-flow
- PA-O only, not PA-P. https://rbi.org.in/Scripts/PublicationsView.aspx?id=12043

**Zoho Payments**
- Static QR with a sound box (Oct 2025). Payment-link QRs open hosted checkout. https://zoho.com/in/payments/whats-new.html
- No raw dynamic-QR API found (**UNVERIFIED**).

**PayGlocal**
- Focused on cross-border payments. https://payglocal.in/

**Bank-direct (ICICI, Axis, Kotak)**
- ICICI: "you need an ICICI Bank current account to use UPI QR or Sound Box". https://www.icici.bank.in/business-banking/cms/merchant-solutions/upi-collections
- Corporate API onboarding goes through an NDA, then UAT (**UNVERIFIED** detail).
- Not practical for an individual.

**BharatPe**
- No public developer API. Third-party source: https://github.com/api-evangelist/bharatpe

**Soundbox and POS vendors** (Razorpay POS/Ezetap, Mswipe, iServeU)
- Their APIs drive the vendor's own hardware and are offered to enterprise or bank partners. Example: https://developer.iserveu.in/docs/payment-soundbox-api
- Not suitable.

**"0% fee" UPI gateway services** (UPIGateway/ekqr and similar)
- UPIGateway states that it "does not provide payment gateway service, nor does it provide UPI ID and UPI Merchant account". https://upigateway.com/
- How it verifies payments is not disclosed.
- **Do not use** for a design that relies on a signed, trustworthy payment confirmation.

---

## 4. RBI Payment Aggregator directions (PA-P)

**Draft, 16 Apr 2024**
- Draft directions for PA-Physical, plus amendments for small merchants. https://www.rbi.org.in/Scripts/BS_PressReleaseDisplay.aspx?prid=57713
  - Draft PA-P: https://www.rbi.org.in/Scripts/bs_viewcontent.aspx?Id=4418
  - Amendments: https://www.rbi.org.in/Scripts/bs_viewcontent.aspx?Id=4419

**Final: Master Direction on Regulation of Payment Aggregators**, RBI/DPSS/2025-26/141, 15 Sep 2025. https://www.rbi.org.in/Scripts/BS_ViewMasDirections.aspx?id=12896
- **Definitions.**
  - **PA-P** "facilitates transaction(s) where both the acceptance device and payment instrument are physically present in close proximity".
  - **PA-O** covers those that are not in close proximity.
  - A counter screen scanned by a customer's phone is a **PA-P-type transaction**.
- **No PA-O-only ban on offline use.** The summary of the fetched text found no explicit rule stopping PA-O entities from serving physical transactions (**UNVERIFIED** in the full text). Each gateway's own product rules decide in practice.
- **Small-merchant due diligence** (turnover ≤ ₹40 lakh):
  - PAN or Form 60 verified with the issuing authority.
  - **Contact point verification** (CPV: physical verification of the address or place of business).
  - One officially valid document (OVD) of the proprietor.
  - New merchants must comply **from 1 Jan 2026**. Same source.
  - **Consequence:** a gateway may ask the owner for CPV (a site visit or geotagged photos) before allowing offline/QR use. Whether Razorpay does so for an individual account: **UNVERIFIED**.
- **Who holds PA-P.** The RBI list (fetched 2026-10-02) shows Razorpay, Cashfree, PhonePe, Paytm PSL, PayU, Easebuzz, Pine Labs, Decentro and Juspay with PA-P. Zaakpay holds PA-O only. Instamojo and Setu are not listed. https://rbi.org.in/Scripts/PublicationsView.aspx?id=12043
- **Effect on this design.**
  - Every shortlisted gateway is authorised for physical transactions, so the regulation does not block it.
  - The practical effects:
    - (a) the merchant profile should truthfully declare in-person/counter collection;
    - (b) expect CPV-style checks.

---

## 5. NPCI UPI QR / deep-link specification

**Sources**
- The latest public spec found is **"UPI Linking Specifications 1.6 (Draft)", Nov 2017**. Mirror: https://www.labnol.org/files/linking.pdf
- Field lengths come from the UPI API Technology Specs v1.2.3 (2017 copy): https://s3-ap-southeast-1.amazonaws.com/he-public-data/NPCI%20API%20Descriptionsb9bceb7.pdf
- No newer public NPCI version could be fetched. npci.org.in blocks automated access.

**Format**
- `upi://pay?param=value&...`, URL-encoded (spaces as `%20`). `sign`, if present, must be last.

| Param | Meaning | Static QR / Dynamic QR | Length / format | P2M vs P2P |
|---|---|---|---|---|
| `pa` | Payee VPA | M / M | 1–255 | both |
| `pn` | Payee name | M / M | 1–99 | both. Apps now show the bank (CBS) name instead (see section 6) |
| `mc` | Merchant category code | O / O | 4 digits (ISO 18245) | `0000` in P2P examples |
| `tid` | Transaction ID (PSP-generated) | O / O | 1–35 | merchant gets it from its PSP |
| `tr` | Transaction / order reference | O / **mandatory for merchant and dynamic** | 1–35 | P2M |
| `tn` | Note | O / O | 1–50 per the API spec; Google Pay docs say max 80 ([GPay][gpay]) | both |
| `am` | Amount | O / **M** | decimal, 2 fraction digits | both |
| `mam` | Minimum amount | O / C | | if absent or null the amount is **not editable** |
| `cu` | Currency | O / O | `INR` | both |
| `url` | Transaction / invoice URL | O / O | http(s) | P2M |
| `mode` | Initiation mode | M / M | 2 digits | see below |
| `orgid` | Originator org ID | M / M | 6 digits; `000000` for merchant-generated | |
| `sign` | Base64 RSA signature | M / M in the spec | | gateway QRs often omit it, e.g. PayU's sample ([PayU][payu-static]) |
| `mid`/`msid`/`mtid` | Merchant, store, terminal IDs | O | max 20 | P2M reconciliation |

**Mode codes**
- In the 1.6 spec: `00` default, `01` QR, `02` secure QR, `04` intent, `05` secure intent, `06` NFC, `07` BLE, `08` UHF, `15` SEBI.
- A current list from Juspay (not an NPCI document): `15` offline dynamic QR, `16` offline dynamic secure QR, `22`/`23` online dynamic QR. https://juspay.io/in/docs/upi-consumer-stack/docs/resources/initiation-mode

**Newer parameters**
- Gateway strings also carry `ver`, `qrMedium`, `purpose`, `QRexpire`, `invoiceNo`, `invoiceDate`, `gstIn` and `gstBrkUp`. Seen in https://docs.payu.in/docs/integrated-static-bharat-qr-generation-api and https://developer.phonepe.com/offline-integration/dynamic-qr-solution/dqr-init-api
- No public NPCI definition was found (**UNVERIFIED** semantics).
- **Design rule:** the device renders the gateway's string byte-for-byte and never edits it.

**Signed QR / signed intent** (spec 1.6, §1.3)
- The merchant or its acquiring bank generates an RSA key pair, and the acquirer registers the public key with UPI.
- The merchant signs the whole string except `&sign=` (SHA256withRSA, Base64).
- Payer apps download the keys and verify. With a valid signature the app can show the merchant as verified. A tampered string is declined. An unsigned string may trigger a "source could not be verified" warning.
- PSP apps sign customer-generated (P2P) QRs with their own key.
- **Only a merchant whose acquirer has registered its key, or a PSP, can sign. An individual with a personal VPA cannot.**
- Whether current apps show a warning for unsigned gateway QRs: **UNVERIFIED**.

**Verified merchant**
- Banks must whitelist an acquired merchant's VPA in the central UPI system so that apps can show an "address verified" icon. UPI Procedural Guidelines: https://yashada.org/yashada_2019/pdfs/e_library_cit/edpri_UPI_Procedural_Guidelines.pdf
- The current exact NPCI definition: **UNVERIFIED**.

**P2M vs P2P**
- In the UPI API the payee `type` is PERSON or ENTITY, and a merchant payee carries an MCC (API spec above).
- `tr` is mandatory only for merchant transactions (spec 1.6).
- A third class, **P2PM** (small merchants on personal accounts), must move to full P2M acquiring once UPI inward credits reach ₹1,00,000 or more per month for 3 consecutive months. NPCI/UPI/OC-192/2023-24, secondary source: https://www.teamleaseregtech.com/updates/article/31002/
- **A gateway's dynamic QR is a P2M transaction from a verified (bank-acquired) merchant.** That is why it avoids the P2P restrictions below.

---

## 6. Restrictions that could bite us

**Confirmed (secondary sources for the NPCI circular text)**

1. **NPCI/UPI/2023-24/OC/76A** (12 Mar 2024, effective 1 Apr 2024). https://taxguru.in/finance/npci-circular-revision-transaction-limits-upi-merchants.html
   - "Payer PSP shall ensure **P2P Intent** based transactions (Initiation mode '04' and '05') shall be disallowed."
   - "Payee PSP shall ensure Intent based transactions (Initiation mode '04') shall be disallowed for all 'Offline' non-verified merchants."
   - "QR share & Pay" (paying from a gallery image) is capped at ₹2,000 for all P2P, and for non-verified offline P2M.
   - A developer reports that intent payments to personal IDs fail with "Payment failed as per upi risk policy". https://wordpress.org/support/topic/payment-failed-as-per-upi-risk-policy-to-keep-your-account-safe/
   - **Effects:**
     - A personal-VPA design cannot use intent.
     - Never take an *intent* link from a gateway and draw it as a QR (`mode=04`). Use the gateway's dedicated QR product.
     - PhonePe's DQR sample does carry `mode=04`. It is issued to a verified merchant, so this should not apply, but that is **UNVERIFIED**.
2. **Camera-scan of a P2P QR with a pre-filled amount.**
   - No NPCI circular and no reliable report was found saying apps block this. **UNVERIFIED either way.**
   - What *is* confirmed for P2P is narrower: intent is blocked, gallery uploads are capped at ₹2,000, and collect is gone (item 3).
   - P2P also gives no webhook, so the soundbox would have nothing trustworthy to announce. **Use a gateway (P2M).**
3. **P2P collect discontinued** from 1 Oct 2025 (NPCI/UPI/OC-220/2025-26, 29 Jul 2025). Secondary sources: https://www.teamleaseregtech.com/updates/article/45492/ and https://www.medianama.com/2025/08/223-npci-p2p-collect-payments-oct-1-what-it-means/
   - Merchant collect is being phased out too:
     - PayU says collect is deprecated from 28 Feb 2026. https://docs.payu.in/docs/upi-collect-disablement-information
     - Cashfree blocked collect on Android and desktop from 28 Feb 2026. https://www.cashfree.com/docs/payments/manage/payment-methods/upi-collect
   - Dynamic QR is the supported path.
4. **Check-status limits** (OC-215/2025-26, 26 Apr 2025). Secondary source: https://teamleaseregtech.com/updates/article/42023/
   - The first status check is allowed 90 s after the transaction, with at most 3 checks, preferably within 2 h.
   - The circular applies to PSPs and banks.
   - **Design:** use webhooks as the main path, and keep any status-API fallback slow and rare.
5. **Response-time cut** to 10–15 s for the Pay and status APIs from 16 Jun 2025 (OC-214/2025-26). Secondary source: https://teamleaseregtech.com/updates/article/42040/
6. **Beneficiary-name display** (OC-101A/2025-26, from 30 Jun 2025): apps show the CBS-verified name for P2P and P2PM, so `pn` is cosmetic. Secondary source: https://www.teamleaseregtech.com/updates/article/41953/
7. **P2P receiver limits** (OC-181/2023-24): 25 credits and ₹4 lakh per 24 h are reported. Wording **UNVERIFIED**. https://www.teamleaseregtech.com/updates/article/28037/
8. **UPI MDR from 15 Oct 2026**, per "NPCI FAQs on UPI MDR, dated 15-9-2026". Secondary source: https://www.scconline.com/blog/post/2026/09/16/npci-released-upi-mdr-faqs-explained/
   - 0.4% on P2M payments **above ₹2,000**, capped at ₹300.
   - Payments of ₹2,000 or less, and P2PM merchants up to ₹1 lakh/month, stay at zero.
   - The article also says "UPI application providers are expressly prohibited from charging any platform fee or similar charge on UPI transactions". Whether this affects gateway "platform fees" such as Razorpay's 2%: **UNVERIFIED** (it may refer to consumer UPI apps only).
   - The ₹1 demo is unaffected by the MDR itself.
9. **BHIM UPI brand guidelines** (updated 4 Jun 2026). https://www.npci.org.in/uploads/BHIM_UPI_Guidelines_2026_012a0b1bce.pdf
   - Layout for a "Sound Box / Dynamic QR" display:
     - QR at least 60% of the design height.
     - The line "Scan & Pay with any UPI app".
     - "Any form of custom messages / images are strictly prohibited".
   - The logos are NPCI trademarks. **Do not commit BHIM/UPI logo files to the open-source repository.**
10. **Not yet in force or unconfirmed:**
    - An NPCI unified soundbox platform (report only). https://inc42.com/buzz/npci-to-roll-out-unified-soundbox-infrastructure-for-merchants-report/
    - NPCI approval of offline QR designs under OC-190 (search snippet only): **UNVERIFIED**.

---

## 7. Ranked recommendation

1. **Razorpay. Apply first.**
   - **Why:** it is the only gateway with all of these documented: self-serve unregistered-individual onboarding, the website step skippable, a single-use fixed-amount UPI QR API, the raw `upi://` string, an HMAC-SHA256 webhook on the raw body, close/fetch/refund APIs, and RBI PA-P authorisation.
   - **Weaknesses:**
     - Two features are enabled on request.
     - Test-mode QRs cannot be scanned.
     - The expiry docs conflict.
     - The UPI fee wording conflicts.
2. **PhonePe Offline DQR.** Apply in parallel only if the owner registers on Udyam (see below).
   - **Why:** purpose-built for a counter display, raw `qrString`, amount locked with `am`=`mam`, real UAT simulator, expiry up to 1 month.
   - **Weaknesses:** access is sales-led; the guidelines list Sole Proprietor (two business proofs) and no individual type; the callback is a salted SHA-256, not an HMAC.
3. **PayU DBQR.**
   - **Why:** an "Individual" entity exists, it returns a raw `qrString`, and it is explicitly for offline collections.
   - **Weaknesses:** the DBQR flag comes via a key account manager; testing is unreliable; pricing is unpublished.
4. **Cashfree (softPOS terminal QR).**
   - Usable only if the owner publishes a compliant website (T&C, privacy, contact) and passes storefront KYC. The image only has to be decoded on the backend.
5. **Paytm Dynamic QR.** Enterprise-only.
6. **Easebuzz.** Needs a website plus two business proofs or CPV.
7. **Setu / Decentro / Pine Labs.** Sales-led or unclear; Decentro's ₹5 minimum blocks ₹1 demos.
8. **Instamojo.** No QR API, ₹9 minimum.

### What the owner should have ready for Razorpay KYC

> **On hold (D-012).** Do not start KYC for this project.
> - If a live account is ever opened, its business description must be literally true. "Collects payments in person at a counter" is true only for a real shop.
> - The checklist below is kept for that case, for example a shop owner opening an account for a pilot.

Sources: https://razorpay.com/docs/payments/business-types-kyc-documents/ and https://razorpay.com/docs/payments/set-up/

- Personal **PAN**, with the name exactly as on the bank account.
- **Aadhaar** with the current mobile number linked, for the OTP/DigiLocker address check (CKYC may auto-fetch).
- A **savings or current account in the owner's own name**: account number and IFSC. Be ready to record a cancelled-cheque video if penny-drop verification fails.
- Business type "Individual / Unregistered", a **brand name**, and a **truthful business description and category** that declares in-person (counter) collection.
  - Do not describe it as an online website business. RBI's directions require the PA to check that transactions match the merchant profile (MD para 13).
- Optional but helpful: a project page (e.g. GitHub Pages) with contact details, terms, privacy and refund policy. Razorpay lets you add it later, and Cashfree/Easebuzz require one if they are ever used.
- Possibly ₹199 + GST KYC fee. Possibly CPV (geotagged photos or a visit) because of the RBI small-merchant rules (UNVERIFIED for Razorpay).
- **After activation, ask support** to enable the **QR Codes (`upi_qr`)** feature and the **`qr_image_content`** flag. Confirm that ₹1 fixed-amount QRs and a 15-minute minimum `close_by` are acceptable.

**Fallback that unlocks options 2, 3 and 6:** register on Udyam (free, Aadhaar-based; Paytm describes it as needing only Aadhaar: https://business.paytm.com/support/i-dont-have-a-registered-business-can-i-still-get-the-payment-gateway). Then onboard as a sole proprietor with Udyam plus one more proof (e.g. a utility bill or Shop & Establishment).

---

## 8. Design implications for the device and backend

- **Draw the gateway string exactly.** The device draws the exact string the backend sends and never adds or edits parameters (NPCI signing and amount locking; see section 5). The backend logs the string length. String lengths seen in doc samples range from about 130 characters (Razorpay) to over 300 (PhonePe with GST/invoice fields), so the QR encoder must handle the longest string the chosen gateway emits.
- **Payment confirmation comes only from a verified webhook.**
  - For Razorpay: `qr_code.credited`, checked with HMAC-SHA256 over the raw body and de-duplicated on `x-razorpay-event-id`.
  - The backend checks `payment.amount == order amount`, `status == captured` and that the QR id matches before pushing "received ₹X" to the device.
  - Use the status/fetch API only as a slow fallback (in the spirit of NPCI OC-215: first check after 90 s or more, few retries).
- **Cancel and timeout.**
  - Create with `close_by = now + 15 min` (meets both documented minimums), and call Close when the shopkeeper cancels.
  - **Race risk:** a customer may be entering their PIN when the QR is closed. What Razorpay does then (decline or auto-refund) is **UNVERIFIED**. Handle late `qr_code.credited` events as "paid after cancel → refund or alert".
- **No P2P fallback.** Never fall back to the owner's personal UPI ID: intent is disallowed, nothing is verified, and there is no webhook.
- **Screen layout.** Keep the QR screen clean (QR large, "Scan & Pay with any UPI app", merchant name). Leave NPCI logos out of the repository (section 6, item 9).

---

## 9. Open questions

1. Will Razorpay enable QR Codes and `qr_image_content` for an *unregistered individual* account? Is there a per-transaction or monthly cap?
2. What is Razorpay's real fee on UPI QR payments: 0%, 0.99% or 2%? Is it T+1 or T+2? Does NPCI's 2026 MDR FAQ ban on "platform fees" apply to gateways?
3. Which minimum `close_by` is right for Razorpay, 2 min or 15 min? Is ₹1 (100 paise) accepted as `payment_amount`?
4. Can a Razorpay `upi_qr` be paid in test mode at all, for example through the BharatQR test-pay endpoint? If not, the first end-to-end test is live ₹1.
5. Does a gateway require CPV for a home-based individual declaring counter collection under the RBI MD (from 1 Jan 2026)?
6. Can an individual or a Udyam-registered proprietor get PhonePe Offline DQR credentials? What are PhonePe's DQR pricing and settlement?
7. Will PayU enable DBQR for an Individual account?
8. Self-payment: will repeated ₹1 payments from the owner's own UPI account to their own merchant account trip gateway fraud rules? Use a second payer, and keep the count small.
9. What are the current official NPCI mode, purpose and `qrMedium` code tables (newer than spec 1.6, 2017)? Not publicly fetchable; ask the gateway if needed.
10. Does OC-190 (NPCI approval of offline QR designs) apply to a single merchant's own display device? UNVERIFIED.

[rzp-kyc]: https://razorpay.com/docs/payments/business-types-kyc-documents/
[rzp-setup]: https://razorpay.com/docs/payments/set-up/
[rzp-create]: https://razorpay.com/docs/api/qr-codes/create/
[rzp-ic-entity]: https://razorpay.com/docs/api/qr-codes/image-content/entity/
[rzp-ic-create]: https://razorpay.com/docs/api/qr-codes/image-content/create/
[rzp-faq]: https://razorpay.com/docs/payments/qr-codes/faqs/
[rzp-pricing]: https://razorpay.com/pricing/
[rbi-list]: https://rbi.org.in/Scripts/PublicationsView.aspx?id=12043
[pp-init]: https://developer.phonepe.com/offline-integration/dynamic-qr-solution/dqr-init-api
[pp-faq]: https://developer.phonepe.com/offline-integration/faqs/dynamic-qr-flow
[pp-start]: https://developer.phonepe.com/offline-integration/others/getting-started-with-your-api
[payu-docs]: https://docs.payu.in/docs/documents-checklist-for-account-activation
[payu-kyc]: https://docs.payu.in/docs/complete-your-kyc
[payu-dbqr]: https://docs.payu.in/reference/dynamic-qr-generation-api
[payu-qr-idx]: https://docs.payu.in/docs/apis-for-upi-qr-integration
[payu-static]: https://docs.payu.in/docs/integrated-static-bharat-qr-generation-api
[cf-faq]: https://www.cashfree.com/docs/help/onboarding-related/onboarding-faqs
[cf-term]: https://www.cashfree.com/docs/api-reference/payments/latest/offline-payments/create-terminal-transaction
[cf-order]: https://www.cashfree.com/docs/api-reference/payments/latest/orders/create-order
[cf-pricing]: https://www.cashfree.com/docs/help/account/pricing
[ptm-noreg]: https://business.paytm.com/support/i-dont-have-a-registered-business-can-i-still-get-the-payment-gateway
[ptm-dqr]: https://www.paytmpayments.com/docs/dynamic-qr-code-payments/
[ptm-create]: https://www.paytmpayments.com/docs/api/create-qr-code-api
[ptm-pricing]: https://business.paytm.com/pricing
[ezb-kyc]: https://docs.easebuzz.in/docs/get-started/6v6i0r0x14zly-kyc-documents
[ezb-seamless]: https://docs.easebuzz.in/docs/payment-gateway/k3ho860cy66zh-seamless-integration-merchant-hosted
[im-kyc]: https://www.instamojo.com/blog/how-to-update-kyc-instamojo/
[im-pr]: https://docs.instamojo.com/reference/create-a-payment-request-1
[im-pricing]: https://www.instamojo.com/pricing/
[setu-qs]: https://docs.setu.co/payments/upi-deeplinks/quickstart
[setu-pa]: https://support.setu.co/support/solutions/articles/81000410169-current-onboarding-requirement
[dec-onb]: https://docs.decentro.tech/docs/payments-collections-pa-onboarding
[dec-qr]: https://docs.decentro.tech/reference/payments_api-collectionsv3-dynamicqr
[dec-link]: https://docs.decentro.tech/reference/payments_api-collectionsv3-paymentlink
[pl-upi]: https://www.pinelabs.com/docs/online-payments/use-cases/accept-payments-upi
[gpay]: https://developers.google.com/pay/india/api/web/create-payment-method

---

## 10. Update 2026-10-03: QR Codes is not available on a plain test-mode account

This changes the Razorpay plan, because the original plan assumed test mode would cover development.

**Evidence from developers who tried it, first-hand logs:**

- **PunarArjan, decision D0021, August 2026.** On a fresh Razorpay test account:
  - `POST /v1/payments/qr_codes`, and even a plain list call, returned `400 BAD_REQUEST_ERROR: "The requested URL was not found on the server"`.
  - The same keys created an Order seconds later. The project concluded that QR Codes "is not provisioned on this account".
  - UPI was not offered as a payment method on that test account either.
  - Source: https://github.com/divyanshi-sachan/PunarArjan/blob/f7daf964615e69364aaa37c47110e3da8f262c16/docs/decisions.md
- **dukaan-mcp, issue 0005, August 2026.** The only documented server-side way to simulate a payment is `POST /v1/bharatqr/pay/test`, and "BharatQR itself needs a support ticket to activate".
  - The same note says UPI Payment Links are not supported in test mode, and UPI Collect (`success@razorpay`) was deprecated on 28 Feb 2026.
  - Source: https://github.com/dharminnagar/dukaan-mcp/blob/9f4714152c8bc85cd6222179f569054355f857d8/.projectmem/issues/0005-blocker-a-headless-agent-cannot-authorize-a-paym.md
- **Razorpay's own FAQ:** QR codes created in test mode cannot be scanned (section 3.1).

**Consequences:**

- With a fresh, un-activated account, our core flow cannot be developed or demonstrated: create a per-sale `upi_qr`, get the raw string, simulate the payment, receive `qr_code.credited`.
- Whether Razorpay support will enable QR Codes, `qr_image_content` and BharatQR test-pay on a test-only account without KYC: **UNVERIFIED**. Ask them before relying on it.
- The account-purpose problem compounds this. A live merchant account must describe a real business, so the owner should not do KYC just to unlock a feature for a project.

**Next step:** find a gateway whose sandbox gives a dynamic UPI QR (raw string), a simulated payment and a signed notification without merchant KYC. The candidates to check first are Setu (sandbox mock-credit trigger, section 3.8), Cashfree, PhonePe and Decentro. In parallel, ask Razorpay support the question above.

**Razorpay pricing, re-checked 2026-10-03** (https://razorpay.com/pricing/ and https://razorpay.com/terms/90-day-free-pg-offer/):

| Item | Amount |
|---|---|
| Setup fee | ₹0 |
| Annual maintenance | ₹0 |
| Monthly fee | None |
| Fees on payments | Deducted from each payment before settlement |
| "UPI QR (standard)" | **0.99%** per transaction |
| "Platform fee (all domestic instruments)" | **2%** per transaction. This line appears alongside the 0.99% line, and which one applies to QR Codes payments is **UNVERIFIED**. |
| GST on fees | 18% |
| KYC processing | ₹199 + tax, "subject to verification" |
| Offer | 0% platform fee for 90 days or ₹5 lakh, for accounts activated from 1 Jul 2026, one per PAN. QR Codes are not listed as excluded. |
| Test mode | Free. No KYC and no money involved. |
