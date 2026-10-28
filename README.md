# Payable IPG - Pre-authorization (card hold)

This guide walks you through **authorizing** a card amount (hold), then **capturing** or **voiding** that hold from your server.

Build checkout **`checkValue`** with your **Merchant Token**. Build capture and void **`checkValue`** with the same token. Call **preauth auth**, **capture**, and **void** from your server and keep your **Business Token** secret.

---

## Prerequisites

1. Ask Payable to enable **pre-authorization** for your merchant account.
2. Obtain **Business Key** and **Business Token** for the preauth from **Settings** -> **Business Integration** in the portal.

**Note:** When generating the Business Token, enable the **Pre Authentication** permission from Business Integration. If the Business Token is already generated, update the permission through **Edit Permission** in the grid.
<img width="1149" height="908" alt="image" src="https://github.com/user-attachments/assets/49bcc4e1-e23b-423a-bd02-f60bc7f04775" />


You will also need:

- **Merchant Key** — Portal → Settings → API Integration
- **Merchant Token** — Portal → Settings → API Integration (this is a secret)

**Rules:**

- Send **`paymentOperation`: `"AUTHORIZE"`** only with **`paymentType`** **`1`** (one-time). Do not set **`isSaveCard`** to **`1`**.
- Pre-authorization **cannot** be used with **Easy Payment Plan (EPP)** fields (`eppConfigId`, etc.).
- Pre-authorization **can** be used with **card discounts** when **`currencyCode`** is **`LKR`**.

---

## Authorize at checkout (card hold)

Start a payment session that places a **hold** on the customer’s card instead of capturing immediately.

**Note:** Use the sandbox URLs while you build and test. Switch to the live URLs when you go live. Use the merchant key and merchant token that belong to that same environment.

**Required parameters** (in addition to your usual one-time checkout fields):

- **`paymentOperation`** — **`"AUTHORIZE"`**

Other mandatory fields match a normal one-time payment (`merchantKey`, `invoiceId`, `amount`, `currencyCode`, `checkValue`, URLs, customer details, etc.). Refer to the preferred SDK or Direct API documentation for complete examples on the full payment request.

### Check value (checkout, authorize only)

Build the check value on your server, in this order, with `|` between each part:

```
UPPERCASE(SHA512[merchantKey|invoiceId|amount|currencyCode|AUTHORIZE|UPPERCASE(SHA512[merchantToken])])
```

- **`amount`** — two decimal places, for example `750.00`.
- Include the literal segment **`AUTHORIZE|`** immediately before the hashed merchant token.

```javascript
const crypto = require("crypto");

function sha512(value) {
  return crypto.createHash("sha512").update(String(value), "utf8").digest("hex").toUpperCase();
}

function buildPreAuthCheckoutCheckValue(merchantKey, invoiceId, amount, currencyCode, merchantToken) {
  const tokenHash = sha512(merchantToken);
  return sha512(
    merchantKey + "|" + invoiceId + "|" + amount + "|" + currencyCode + "|AUTHORIZE|" + tokenHash,
  );
}
```

### Checkout with a card discount

If you send **`discountId`**, use the card discount checkValue format, then add **`AUTHORIZE|`** before the merchant token hash:

```
UPPERCASE(SHA512[
  merchantKey|invoiceId|amount|currencyCode|discountId|originalAmount|discountAmount|AUTHORIZE|UPPERCASE(SHA512[merchantToken])
])
```

### Sample request

```json
{
  "merchantKey": "YOUR_MERCHANT_KEY",
  "invoiceId": "Inv-preauth-001",
  "paymentType": 1,
  "paymentOperation": "AUTHORIZE",
  "currencyCode": "LKR",
  "amount": "750.00",
  "checkValue": "YOUR_CHECK_VALUE",
  "notifyUrl": "https://your-domain.com/payable/webhook",
  "returnUrl": "https://your-domain.com/payable/return"
}
```

```bash
curl -X POST "https://sandboxipgpayment.payable.lk/ipg/sandbox/direct-api" \
  -H "Content-Type: application/json" \
  -d "{\"merchantKey\":\"YOUR_MERCHANT_KEY\",\"invoiceId\":\"Inv-preauth-001\",\"paymentType\":1,\"paymentOperation\":\"AUTHORIZE\",\"currencyCode\":\"LKR\",\"amount\":\"750.00\",\"checkValue\":\"YOUR_CHECK_VALUE\",\"notifyUrl\":\"https://your-domain.com/webhook\",\"returnUrl\":\"https://your-domain.com/return\"}"
```

### Success response

A successful session create returns **HTTP 200** with:

```json
{
  "status": "PENDING",
  "uid": "SESSION_UID",
  "statusIndicator": "RANDOM_STATUS_INDICATOR",
  "paymentPage": "https://sandboxipgpayment.payable.lk/ipg/sandbox/?uid=SESSION_UID"
}
```

**Note:** Webhook payload and webhook **`checkValue`** rules are the same as for a standard one-time payment without pre-authorization.

---

## Obtain an access token (capture and void)

Before capture or void, request an access token with your business credentials.

**Sandbox**: `POST https://sandboxipgpayment.payable.lk/ipg/preauth/auth`

**Live**: `POST https://ipgpayment.payable.lk/ipg/preauth/auth`

**Headers**:

```
Content-Type: application/json
Authorization: BASE64(businessKey:businessToken)
```

Encode **`businessKey:businessToken`** as UTF-8, Base64-encode the string.

**Body**:

```json
{
  "grant_type": "client_credentials"
}
```

### Success response

```json
{
  "accessToken": "ACCESS_TOKEN",
  "tokenType": "Bearer",
  "expiresIn": 300,
  "environment": "sandbox",
  "merchantKey": "YOUR_MERCHANT_KEY"
}
```

Use **`Authorization: Bearer {accessToken}`** on capture and void until the token expires.

---

## Capture a hold

Settle all or part of an authorized amount.

**Sandbox**: `POST https://sandboxipgpayment.payable.lk/ipg/preauth/capture`

**Live**: `POST https://ipgpayment.payable.lk/ipg/preauth/capture`

**Headers**:

```
Content-Type: application/json
Authorization: Bearer ACCESS_TOKEN
```

**Required parameters**:

- **`merchantKey`** — your merchant key.
- **`orderId`** — Payable order id from the authorization (for example from the payment webhook **`payableOrderId`**).
- **`amount`** — amount to capture; two decimal places, for example `750.00`.
- **`currencyCode`** — same currency as the authorization.
- **`clientRequestId`** — your unique id for this capture request.
- **`checkValue`** — security hash below.

### Check value (capture)

```
UPPERCASE(SHA512[
  merchantKey|orderId|CAPTURE|amount|currencyCode|clientRequestId|UPPERCASE(SHA512[merchantToken])
])
```

```javascript
function buildPreAuthCaptureCheckValue(
  merchantKey,
  orderId,
  amount,
  currencyCode,
  clientRequestId,
  merchantToken,
) {
  const tokenHash = sha512(merchantToken);
  return sha512(
    merchantKey + "|" + orderId + "|CAPTURE|" + amount + "|" + currencyCode + "|" + clientRequestId + "|" + tokenHash,
  );
}
```

### Sample request

```json
{
  "merchantKey": "YOUR_MERCHANT_KEY",
  "orderId": "PAYABLE_ORDER_ID",
  "amount": "750.00",
  "currencyCode": "LKR",
  "clientRequestId": "capture-req-001",
  "checkValue": "YOUR_CHECK_VALUE"
}
```

### Success response

A successful call returns **HTTP 200**. **`status`** reflects how much of the hold was captured (for example **`PARTIALLY_CAPTURED`** for a partial capture). **`captureMode`** is **`PARTIAL`** or **`FULL`** depending on the request.

```json
{
  "orderId": "oid-xxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "captureMode": "FULL",
  "capturedAmount": "40.00",
  "currencyCode": "LKR",
  "status": "CAPTURED",
  "transactionId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "gatewayCode": "APPROVED",
  "checkValue": ""
}
```

**Note:** It is recommended to validate the **`checkValue`** before you treat the capture as final.

---

## Void a hold (release)

Release an authorization without capturing.

**Sandbox**: `POST https://sandboxipgpayment.payable.lk/ipg/preauth/void`

**Live**: `POST https://ipgpayment.payable.lk/ipg/preauth/void`

**Headers**:  

```
Content-Type: application/json
Authorization: Bearer ACCESS_TOKEN
```

**Required parameters**:

- **`merchantKey`**
- **`orderId`**
- **`currencyCode`**
- **`clientRequestId`**
- **`checkValue`**
- **`reason`**

### Check value (void)

```
UPPERCASE(SHA512[
  merchantKey|orderId|VOID|currencyCode|clientRequestId|UPPERCASE(SHA512[merchantToken])
])
```

```javascript
function buildPreAuthVoidCheckValue(
  merchantKey,
  orderId,
  currencyCode,
  clientRequestId,
  merchantToken,
) {
  const tokenHash = sha512(merchantToken);
  return sha512(
    merchantKey + "|" + orderId + "|VOID|" + currencyCode + "|" + clientRequestId + "|" + tokenHash,
  );
}
```

### Sample request

```json
{
  "merchantKey": "YOUR_MERCHANT_KEY",
  "orderId": "PAYABLE_ORDER_ID",
  "currencyCode": "LKR",
  "clientRequestId": "void-req-001",
  "reason": "Customer cancelled",
  "checkValue": "YOUR_CHECK_VALUE"
}
```

### Success response

A successful void returns **HTTP 200** with **`status`** **`CANCELLED`** when the hold is released.

```json
{
  "orderId": "oid-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "status": "CANCELLED",
  "transactionId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "gatewayCode": "APPROVED",
  "checkValue": ""
}
```

**Note:** It is recommended to validate the **`checkValue`** before you treat the void as final.

---

## Pay with saved card (token pay authorize)

You can place a **hold** on a saved card. Use the **tokenization** access token for pay, then the **preauth** APIs in this guide to capture or void.

### Step 1 — Tokenization access token

Obtain a Bearer token as described in the tokenization documentation.

**Sandbox**: `POST https://sandboxipgpayment.payable.lk/ipg/v2/auth/tokenize`

**Live**: `POST https://ipgpayment.payable.lk/ipg/v2/auth/tokenize`

**Headers**:

```
Content-Type: application/json
Authorization: BASE64(businessKey:businessToken)
```

Use the **business key** and **business token** from your portal Settings -> Business Integration.

**Body**:

```json
{
  "grant_type": "client_credentials"
}
```

**Success response**:

```json
{
  "accessToken": "ACCESS_TOKEN",
  "tokenType": "Bearer",
  "expiresIn": 300,
  "environment": "sandbox"
}
```

### Step 2 — Authorize with a saved card

**Sandbox**: `POST https://sandboxipgpayment.payable.lk/ipg/v2/tokenize/pay`

**Live**: `POST https://ipgpayment.payable.lk/ipg/v2/tokenize/pay`

**Headers**:

```
Content-Type: application/json
Authorization: Bearer ACCESS_TOKEN
```

**Required parameters**:

- **`merchantId`**
- **`customerId`**
- **`tokenId`**
- **`invoiceId`**
- **`amount`** — two decimal places
- **`currencyCode`**
- **`paymentOperation`** — **`"AUTHORIZE"`**
- **`checkValue`** — see below
- **`webhookUrl `**

### Check value (token pay authorize)

Build the check value on your server with your **Merchant Token**:

```
UPPERCASE(SHA512[
  merchantId|invoiceId|amount|currencyCode|customerId|tokenId|AUTHORIZE|UPPERCASE(SHA512[merchantToken])
])
```

Use the **exact** strings you send in the pay request body. Include the literal segment **`AUTHORIZE|`** immediately before the hashed merchant token.

```javascript
function buildTokenPayPreAuthCheckValue(
  merchantId,
  invoiceId,
  amount,
  currencyCode,
  customerId,
  tokenId,
  merchantToken,
) {
  const tokenHash = sha512(merchantToken);
  return sha512(
    merchantId + "|" + invoiceId + "|" + amount + "|" + currencyCode + "|" +
    customerId + "|" + tokenId + "|AUTHORIZE|" + tokenHash,
  );
}
```

### Sample pay request

```json
{
  "merchantId": "YOUR_MERCHANT_ID",
  "customerId": "YOUR_CUSTOMER_ID",
  "tokenId": "YOUR_TOKEN_ID",
  "invoiceId": "Inv-token-preauth-001",
  "amount": "750.00",
  "currencyCode": "LKR",
  "paymentOperation": "AUTHORIZE",
  "webhookUrl": "https://your-domain.com/payable/webhook",
  "checkValue": "YOUR_CHECK_VALUE"
}
```

**Note:** Webhook payload and webhook **`checkValue`** for token pay follow the same rules as pay with saved card without pre-authorization.

---

## Errors

### Validation error

Missing or badly formatted fields return **HTTP 400**:

```json
{
  "status": 400,
  "errors": {
    "checkValue": [
      "Check Value is a required field."
    ],
  }
}
```

### Request failed

Invalid business credentials, token, or check value on auth / capture / void often returns **HTTP 404**:

```json
{
  "status": 404,
  "error": "Invalid authentication"
}
```

Check the following and try again:

1. **Merchant key**, **merchant token**, and **business** credentials match the environment (sandbox vs live).
3. Capture and void use **`Bearer`** with a fresh **access token**.
4. Checkout hash includes **`AUTHORIZE|`**; capture includes **`CAPTURE|`**; void includes **`VOID|`**.
5. All hashes use uppercase hex and `|` between each segment in the order shown above.
6. **`orderId`** is the Payable order id from the authorization, not your **`invoiceId`**.
