# PAYable IPG — Direct API Integration Guide

PAYable **Direct API** lets your **server** start a hosted checkout using JSON over HTTPS: first obtain a short-lived **access token**, then create a session and **redirect the customer** to the returned payment page. This guide describes that flow end-to-end.

---

## The PAYable Direct API integration

You need credentials from PAYable:

- **Merchant Key** — identifier for your merchant account (provided by PAYable).  
- **Merchant Token** — secret used only on your server to build **`checkValue`** digests (never expose it in the browser or mobile app) (provided by PAYable).

- **Business Key** - provided by PAYable
- **Business Token** - provided by PAYable

When the customer pays, they complete card entry on PAYable’s hosted pages. Your app never collects raw card data. After you create a session, send the customer to the **`paymentPage`** URL; when payment completes, PAYable can notify your backend at the URL you configure (**`notifyUrl`**).

---

## Environments and base URL

All routes in this service are mounted under **`/ipg`**.

| Item | Value |
|------|--------|
| **Sandbox** |
| **Token endpoint** | `POST https://sandboxipgpayment.payable.lk/ipg/auth/direct-api` |
| **Session endpoint** | `POST https://sandboxipgpayment.payable.lk/ipg/sandbox/direct-api` |
| **Hosted checkout** | `GET https://sandboxipgpayment.payable.lk/sandbox/?uid={SESSION_UID}` (returned as `paymentPage`) |
| **Live** |
| **Token endpoint** | `POST https://ipgpayment.payable.lk/ipg/auth/direct-api` |
| **Session endpoint** | `POST https://ipgpayment.payable.lk/ipg/live/direct-api` |
| **Hosted checkout** | `GET https://ipgpayment.payable.lk/ipg/live/?uid={SESSION_UID}` (returned as `paymentPage`) |

---

## Implementation

### 1. Obtain an access token (Direct Auth)

**`POST /ipg/auth/direct-api`**

**Headers**

- `Content-Type: application/json`
- `Authorization: <base64(businessKey:businessToken)>` — concatenate **businessKey**, a single colon **`:`**, and **businessToken**; Base64-encode the result.

**Body (JSON)**

| Field | Required | Description |
|-------|----------|-------------|
| `grant_type` | Yes | Grant type string accepted by PAYable for your integration. |
| `packageName` | *Conditional* | Non-empty for **native / app** flows (Mobile integration). |
| `originDomain` | *Conditional* | For **web** flows, your HTTPS origin. |

**Example (web)**

```json
{
  "grant_type": "client_credentials",
  "originDomain": "https://testsite.test"
}
```

**Success (200)** — example shape:

```json
{
  "accessToken": "<access token>",
  "tokenType": "Bearer",
  "expiresIn": "300",
  "environment": "<env>"
}
```

Note: The Access Token expires in 5 minutes.

---

### 2. Create a checkout session

**`POST /ipg/{environment}/direct-api`**

**Headers**

- `Content-Type: application/json`
- `Authorization: Bearer <accessToken>` — must be the same token returned by step 1, for the same **`merchantKey`** in the body. It should not be expired.

**Body**

Send a JSON object with below mandatory and optional fields.

---

#### 2.1 General (one-time) — `paymentType` = `1`

**Mandatory**

| Field | Notes |
|-------|--------|
| `merchantKey` | Must match the merchant used in step 1. |
| `checkValue` | Server-side digest (see **checkValue**). |
| `invoiceId` | Merchant invoice / order id. |
| `currencyCode` | `USD`, `GBP`, `EURO`, `LKR`, or `EUR`. |
| `paymentType` | `1` for one-time. |
| `amount` | String with two decimals, e.g. `"10.00"`. |
| `orderDescription` | Short description. |
| `logoUrl` | HTTPS logo URL. |
| `returnUrl` | HTTPS customer return URL. |
| `originDomain` | **Web:** required if web (HTTPS merchant URL where the request originates from) (must match step 1). |
| `packageName` | **Mobile:** required if mobile (must match step 1). |
| `notifyUrl` | HTTPS server callback URL. |

**Optional**

| Field | Notes |
|-------|--------|
| `cancelUrl` | HTTPS cancel URL. |

**Customer & billing — `paymentType` = `1`**

| Field | Standard (Optional Mode **off**) | With Optional Mode **on** |
|-------|----------------------------------|---------------------------|
| `customerFirstName` | Required | **[Optional Mode]** Optional (may omit or empty). |
| `customerLastName` | Required | **[Optional Mode]** Optional (may omit or empty). |
| `customerEmail` | Required | **[Optional Mode]** Optional (may omit or empty). |
| `customerMobilePhone` | Required | **[Optional Mode]** Optional (may omit or empty). |
| `customerPhone` | Optional | Optional |
| `billingAddressStreet` | Required | **[Optional Mode]** Optional (may omit or empty). |
| `billingAddressCity` | Required | **[Optional Mode]** Optional (may omit or empty). |
| `billingAddressPostcodeZip` | Required | **[Optional Mode]** Optional (may omit or empty). |
| `billingAddressCountry` | Required (e.g. `LKR`) | **[Optional Mode]** Optional (may omit or empty). |
| `billingCompanyName` | Optional | Optional |
| `billingAddressStreet2` | Optional | Optional |
| `billingAddressStateProvince` | Optional | Optional |

**Rule (Optional Mode on, `paymentType` = `1` only):** at least **one** of **`customerEmail`** or **`customerMobilePhone`** must be non-empty.

**Shipping** — all fields optional (`shippingContact*`, `shippingCompanyName`, `shippingAddress*`).

---

#### 2.2 Recurring — `paymentType` = `2`

| Field | Requirement | Notes |
|-------|-------------|--------|
| `interval` | Mandatory | `MONTHLY`, `QUARTERLY`, `ANNUALLY`, `WEEKLY`, or `DAILY`. |
| `doFirstPayment` | Mandatory | `"0"` or `"1"`. |
| `recurringAmount` | Mandatory | Two-decimal string. |
| `startDate` | Mandatory | Date `YYYY-MM-DD`. |
| `endDate` | Mandatory | Date `YYYY-MM-DD` or string **`FOREVER`**. |
| `isRetry` | Mandatory | **`"0"`** or **`"1"`** only. |
| `retryAttempts` | Mandatory |  |
| `amount` | Mandatory | Two-decimal string. |

**Customer & billing — `paymentType` = `2`**

| Field | Standard | With Optional Mode **on** |
|-------|----------|---------------------------|
| `customerFirstName`, `customerLastName`, `customerEmail` | Required | **Still required** (not relaxed for recurring). |
| `customerMobilePhone` | Required | **[Optional Mode]** Optional. |
| `billingAddressStreet`, `billingAddressCity`, `billingAddressPostcodeZip`, `billingAddressCountry` | Required | **[Optional Mode]** Optional (may omit or empty). |
| Other customer/billing/shipping | Same as 2.1 | Same pattern: company / street2 / state / phone / shipping optional; **[Optional Mode]** also relaxes the billing address lines above. |


---

#### 2.3 Tokenized one-time — `paymentType` = `3`

| Field | Requirement | Notes |
|-------|-------------|--------|
| `isSaveCard` | Mandatory | `0` or `1`. |
| `doFirstPayment` | Mandatory if `isSaveCard` = `1` | `"0"` or `"1"`. |
| `customerRefNo` | Mandatory if `isSaveCard` = `1` | Your customer reference. |
| `amount` | Mandatory | If `isSaveCard` = `1` and `doFirstPayment` = `0`, amount must be **`0.00`**. |

**Customer & billing — `paymentType` = `3`**

| Field | Standard | With Optional Mode **on** |
|-------|----------|---------------------------|
| `customerFirstName`, `customerLastName`, `customerEmail` | Required | **Still required** (optional mode does not relax these for type `3`). |
| `customerMobilePhone` | Required | **[Optional Mode]** Optional. |
| Billing address lines / country | Required | **[Optional Mode]** Optional (same pattern as §2.1 / §2.2). |

---

### 3. Open the hosted checkout

**Success (200)** — example:

```json
{
  "status": "PENDING",
  "uid": "XXXXXXXXXXX",
  "statusIndicator": "XXXXXXX",
  "paymentPage": "https://xxxxxx/ipg/{environment}}/?uid=xxxxxxx-..."
}
```

**Redirect** the customer to **`paymentPage`** to complete the payment.

---

## checkValue

**Format:**

`UPPERCASE(SHA512[<merchantKey>|<invoiceId>|<amount>|<currencyCode>|UPPERCASE(SHA512[<merchantToken>])])`

- Pipe `|` is the literal separator between segments.

---

## Sample session payload (one-time, web)

```json
{
  "merchantKey": "YOUR_MERCHANT_KEY",
  "currencyCode": "LKR",
  "checkValue": "XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX",
  "invoiceId": "INV00000001",
  "paymentType": 1,
  "amount": "100.00",
  "orderDescription": "Order 123",
  "logoUrl": "https://domain/your_logo_url",
  "returnUrl": "https://domain/your_return_url",
  "notifyUrl": "https://domain/your_notify_url",
  "originDomain": "https://your_origin_domain",
  "customerFirstName": "Customer First Name",
  "customerLastName": "Customer Last Name",
  "customerMobilePhone": "07XXXXXXXX",
  "customerEmail": "testmail@example.com",
  "billingAddressStreet": "Main St",
  "billingAddressCity": "Colombo",
  "billingAddressCountry": "LKA",
  "billingAddressPostcodeZip": "XXXXX",
}
```

Adjust fields to match your environment and merchant settings. Mobile flows include **`packageName`** and omit **`originDomain`**.

---

## Payment-related errors

Validation failures return **`400`** with a per-field map:

```json
{
  "status": 400,
  "errors": {
    "isRetry": ["Invalid format for Is Retry."]
  }
}
```

Authentication or origin/package mismatch may return **`401`** or **`400`** with a single `error` string. Upstream or configuration issues may return **`404`** or **`500`**.

---

## Listening to payment notifications

PAYable calls your **webhook** URL server-to-server (the URL you supply as **`notifyUrl`**).

- The callback is **not** loaded in the browser; test by updating your database when the endpoint is hit.
- **`notifyUrl`** must use a **public HTTPS** host; **localhost** will not receive production callbacks.

### Typical callback payload

| Field | Description |
|-------|-------------|
| `merchantKey` | Merchant identifier |
| `payableOrderId` | PAYable order id |
| `payableTransactionId` | Transaction reference |
| `payableAmount` | Amount |
| `payableCurrency` | Currency |
| `invoiceNo` | Your invoice id |
| `statusCode` | Numeric status |
| `statusMessage` | e.g. `SUCCESS` / `FAILURE` |
| `paymentType`, `paymentMethod`, `paymentScheme` | Payment metadata |
| `custom1`, `custom2` |  |
| `cardHolderName`, `cardNumber` | Present when applicable (masked PAN) |
| `checkValue` |  |

### Acknowledging the callback

Respond with JSON such as:

```json
{
  "Status": 200
}
```

---
