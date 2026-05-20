# PAYable IPG — Direct API Integration Guide

PAYable **Direct API** facilitates you to start a hosted checkout using JSON over HTTPS. You have to first obtain a short-lived **access token**, then create a payment session, and **redirect the customer** to the returned payment page. This guide describes that flow end-to-end.

---

## The PAYable Direct API integration

You need to obtain the following credentials from PAYable to proceed:

- **Merchant Key** — Identifier for your merchant account.  
- **Merchant Token** — A secret used only on your server to build the **`checkValue`** digests. Make sure not expose this token in the browser or mobile app.

- **Business Key** - Provided by PAYable
- **Business Token** - Provided by PAYable

When the customer pays, they complete the card details entry process on PAYable’s hosted pages. Your app never collects raw card data. After you create a session, navigate the customer to the **`paymentPage`** URL. When payment completes, PAYable can notify your backend at the URL you configure (**`webhookUrl`**).

---

## Environments and base URLs

All routes in this service are mounted under **`/ipg`**.

| Item | Value |
|------|--------|
| **Sandbox** |
| **Auth endpoint** | `POST https://sandboxipgpayment.payable.lk/ipg/auth/direct-api` |
| **Direct payment API endpoint** | `POST https://sandboxipgpayment.payable.lk/ipg/sandbox/direct-api` |
| **Live** |
| **Auth endpoint** | `POST https://ipgpayment.payable.lk/ipg/auth/direct-api` |
| **Direct payment API endpoint** | `POST https://ipgpayment.payable.lk/ipg/pro/direct-api` |

---

## Implementation

### 1. Obtain an access token (Direct Auth)

**`POST /ipg/auth/direct-api`**

**Headers**

- `Content-Type: application/json`
- `Authorization: <base64(businessKey:businessToken)>` — Concatenate your **businessKey**, a single colon **`:`**, and the **businessToken**. Base64-encode the result.

**Body (JSON)**

| Field | Required | Description |
|-------|----------|-------------|
| `grant_type` | Yes | Grant type string accepted by PAYable for your integration. Refer to the example below. |
| `packageName` | *Conditional* | Should be non-empty for **native / app** flows (mobile integration). |
| `originDomain` | *Conditional* | Should be non-empty for **web** flows. |

**Note:** Please do not enter both `pakcageName` and `originDomain`. Only one is needed.

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
- `Authorization: Bearer <accessToken>` — This need to be the same token returned by step 1, for the same **`merchantKey`** in the body. Make sure it is not expired.

**Body**

Send a JSON object with below mandatory and optional fields based on your payment type. Payment type 1 indicates one time payments, type 2 indicates recurring payments, and type 3 indicates tokenized payments.

---

#### 2.1 Common fields

**Mandatory for all three payment types (1, 2, 3)**

| Field | Notes |
|-------|--------|
| `merchantKey` | Must match the merchant used in step 1. |
| `checkValue` | Server-side digest (see **section 4. Check Value**). |
| `invoiceId` | Customer's invoice id. |
| `currencyCode` | `LKR`, `USD`, `GBP`, or `EUR`. |
| `paymentType` | `1` for one-time. |
| `amount` | String with two decimals, e.g. `"10.00"`. |
| `orderDescription` | Short description. |
| `logoUrl` | HTTPS logo URL. |
| `returnUrl` | HTTPS customer return URL. |
| `originDomain` | **Web:** required if web (HTTPS merchant URL to indicate where the request originates from) (must match step 1). |
| `packageName` | **Mobile:** required if mobile (must match step 1). |
| `webhookUrl` | HTTPS server callback URL. |

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
| `billingAddressCountry` | Required (e.g. `LK`) | **[Optional Mode]** Optional (may omit or empty). |
| `billingCompanyName` | Optional | Optional |
| `billingAddressStreet2` | Optional | Optional |
| `billingAddressStateProvince` | Optional | Optional |

**Rule (When Optional Mode is switched on and when `paymentType` = `1` only):** at least **one** of **`customerEmail`** or **`customerMobilePhone`** fields must be non-empty.

**Shipping** — All shipping fields are optional (`shippingContactFirstName`,`shippingContactLastName`, `shippingContactMobilePhone`, `shippingContactPhone`, `shippingContactEmail`, `shippingCompanyName`, `shippingAddressStreet`, `shippingAddressStreet2`, `shippingAddressCity`, `shippingAddressStateProvince`, `shippingAddressCountry`, shippingAddressPostcodeZip`).

---

#### 2.2 Recurring Payments — `paymentType` = `2`

**Additional mandatory fields**

| Field | Requirement | Notes |
|-------|-------------|--------|
| `interval` | Mandatory | `MONTHLY`, `QUARTERLY`, and `ANNUALLY`. |
| `doFirstPayment` | Mandatory | `"0"` or `"1"`. |
| `recurringAmount` | Mandatory | Two-decimal string. |
| `startDate` | Mandatory | Date `YYYY-MM-DD`. |
| `endDate` | Mandatory | Date `YYYY-MM-DD` or string **`FOREVER`**. |
| `isRetry` | Mandatory | **`"0"`** or **`"1"`** only. |
| `retryAttempts` | Mandatory |  |

**Customer & billing — `paymentType` = `2`**

| Field | Standard | With Optional Mode **on** |
|-------|----------|---------------------------|
| `customerFirstName`, `customerLastName`, `customerEmail` | Required | **Still required** (not optional for recurring payments). |
| `customerMobilePhone` | Required | **[Optional Mode]** Optional. |
| `billingAddressStreet`, `billingAddressCity`, `billingAddressPostcodeZip`, `billingAddressCountry` | Required | **[Optional Mode]** Optional (may omit or empty). |
| Other customer/billing/shipping | Same as 2.1 | Same pattern: company / street2 / state / phone / shipping optional; **[Optional Mode]** also relaxes the billing address lines above. |


---

#### 2.3 Tokenized Payments — `paymentType` = `3`

**Additional mandatory fields**

| Field | Requirement | Notes |
|-------|-------------|--------|
| `isSaveCard` | Mandatory | `0` or `1`. |
| `doFirstPayment` | Mandatory if `isSaveCard` = `1` | `"0"` or `"1"`. |
| `customerRefNo` | Mandatory if `isSaveCard` = `1` | Your customer reference number. |
| `amount` | Mandatory | If `isSaveCard` = `1` and `doFirstPayment` = `0`, amount must be **`0.00`**. |

**Customer & billing — `paymentType` = `3`**

| Field | Standard | With Optional Mode **on** |
|-------|----------|---------------------------|
| `customerFirstName`, `customerLastName`, `customerEmail` | Required | **Still required** (not optional for tokenized payments). |
| `customerMobilePhone` | Required | **[Optional Mode]** Optional. |
| Billing address lines / country | Required | **[Optional Mode]** Optional (same pattern as 2.2). |

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

### 4. Check Value

#### 4.1 Format for regular one time and recurring payments:

`UPPERCASE(SHA512[<merchantKey>|<invoiceId>|<amount>|<currencyCode>|UPPERCASE(SHA512[<merchantToken>])])`

#### 4.2 Format for tokenize payments (paymentType = 3 )

`UPPERCASE(SHA512[<merchantKey>|<invoiceId>|<amount>|<currencyCode>|<customerRefNo>|UPPERCASE(SHA512[<merchantToken>])])`

- Pipe `|` is the literal separator between segments.

---

### 5. Additional Tokenization Features

- **Live Root URL**: `POST https://ipgpayment.payable.lk`
- **Sandbox Root URL**: `POST https://sandboxipgpayment.payable.lk`

#### 5.1. List Saved Cards

Retrieve all saved cards for a customer.

**Endpoint**: `POST {{rootUrl}}/ipg/v2/tokenize/listCard`

**Headers Required**:

```
Content-Type: application/json
```

**Required Parameters**:

- `merchantId` - Your merchant ID
- `customerId` - Customer ID
- `checkValue` - Security hash

**CheckValue Generation:**

```
UPPERCASE(SHA512[merchantId|customerId|UPPERCASE(SHA512[merchantToken])])
```

#### 5.2. Delete Saved Card

Remove a saved card from customer's account.

**Endpoint**: `POST {{rootUrl}}/ipg/v2/tokenize/deleteCard`

**Headers Required**:

```
Content-Type: application/json
```

**Required Parameters**:

- `merchantId` - Your merchant ID
- `customerId` - Customer ID
- `tokenId` - Token ID to delete
- `checkValue` - Security hash

**CheckValue Generation:**

```
UPPERCASE(SHA512[merchantId|customerId|tokenId|UPPERCASE(SHA512[merchantToken])])
```

#### 5.3. Edit Saved Card

Update card nickname or set as default card.

**Endpoint**: `POST {{rootUrl}}/ipg/v2/tokenize/editCard`

**Headers Required**:

```
Content-Type: application/json
Authorization: Bearer {access_token}
```

**How to Generate JWT Access Token**:

1. Create Basic Auth token: `base64(businessKey:businessToken)`
2. POST to `{{rootUrl}}/ipg/v2/auth/tokenize` with headers:
   ```
   Content-Type: application/json
   Authorization: {basicAuthToken}
   ```
   Body: `{"grant_type": "client_credentials"}`
3. Extract `accessToken` from response and use in Authorization header

**Required Parameters**:

- `customerId` - Get it from 1st callback
- `tokenId` - Get it from list card API
- `nickName` - Optional nickname for the card
- `isDefaultCard` - Set as default card (0 or 1)
- `checkValue` - Security hash

**CheckValue Generation:**

```
UPPERCASE(SHA512[merchantId|customerId|tokenId|UPPERCASE(SHA512[merchantToken])])
```

#### 5.4. Pay with Saved Card

Process payment using a previously saved card token.

**Endpoint**: `POST {{rootUrl}}/ipg/v2/tokenize/pay`

**Headers Required**:

```
Content-Type: application/json
Authorization: Bearer {access_token}
```

**How to Generate Access Token**:

1. Create Basic Auth token: `base64(businessKey:businessToken)`
2. POST to `{{rootUrl}}/ipg/v2/auth/tokenize` with headers:
   ```
   Content-Type: application/json
   Authorization: {basicAuthToken}
   ```
   Body: `{"grant_type": "client_credentials"}`
3. Extract `accessToken` from response and use in Authorization header

**Required Parameters**:

- `merchantId` - Get it from 1st callback
- `customerId` - Get it from 1st callback
- `tokenId` - Get it from list card API
- `invoiceId` - Invoice ID
- `amount` - Payment amount
- `currencyCode` - Currency code
- `checkValue` - Security hash
- `webhookUrl` - Webhook URL for notifications (https://yoursite.com/webhook/payment)

**Optional Parameters**:

- `custom1` - Custom field 1 for merchant-specific data
- `custom2` - Custom field 2 for merchant-specific data

**CheckValue Generation:**

```
UPPERCASE(SHA512[merchantId|invoiceId|amount|currencyCode|customerId|tokenId|UPPERCASE(SHA512[merchantToken])])
```

#### 5.5. Add Card

**Note:** Follow the same process as section 2.3 to save the card.

---

## Sample Session Payload (one-time, web)

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
  "webhookUrl": "https://domain/your_webhook_url",
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
### Check Value Validation

**Formula for Webhook Validation (One-Time Payment):**

```
UPPERCASE(SHA512[merchantKey|payableOrderId|payableTransactionId|payableAmount|currencyCode|invoiceNo|statusCode|UPPERCASE(SHA512[merchantToken])])
```

**Formula for Tokenize Payment Webhook Validation (paymentType = 3):**

```
UPPERCASE(SHA512[merchantKey|payableOrderId|payableTransactionId|payableAmount|currencyCode|invoiceNo|statusCode|customerRefNo|UPPERCASE(SHA512[merchantToken])])
```

```javascript
// Validate webhook response
function validateWebhook(webhookData, merchantToken) {
  const calculatedCheckValue = CryptoJS.SHA512(
    webhookData.merchantKey +
      "|" +
      webhookData.payableOrderId +
      "|" +
      webhookData.payableTransactionId +
      "|" +
      webhookData.payableAmount +
      "|" +
      webhookData.payableCurrency +
      "|" +
      webhookData.invoiceNo +
      "|" +
      webhookData.statusCode +
      "|" +
      CryptoJS.SHA512(merchantToken).toString().toUpperCase()
  )
    .toString()
    .toUpperCase();

  return calculatedCheckValue === webhookData.checkValue;
}
```
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

PAYable calls your **webhook** URL server-to-server (the URL you supply as **`webhookUrl`**).

- The callback is **not** loaded in the browser. You can test by updating your database when the endpoint is hit.
- **`webhookUrl`** must use a **public HTTPS** host. **localhost** will not receive production callbacks.

### Typical callback payload (one time, recurring)

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

### Typical callback payload (tokenized)

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
| `paymentId` | Your payment id |
| `customerRefNo` | Your customer reference number |
| `token`, `tokenId`, `maskedCardNo`, `exp`, `reference`, `nickname`, `tokenStatus`, `defaultCard` | Token details |
| `merchantId` | Your merchant id |
| `customerId` | Your customer id |
| `uid` | Your uid |
| `statusIndicator` |  |

### Acknowledging the callback

Respond with JSON such as:

```json
{
  "Status": 200
}


```

---
