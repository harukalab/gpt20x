# iOS ChatGPT Subscription Intercept & Transfer

> **⚠️ Disclaimer**: This guide is for educational and research purposes. Please comply with relevant laws, regulations, and terms of service.

---

## Table of Contents

1. [Core Principles](#1-core-principles)
2. [Intercept: Blocking Callback Requests](#2-intercept-blocking-callback-requests)
3. [Transfer: Modifying User ID for Cross-Account Activation](#3-transfer-modifying-user-id-for-cross-account-activation)
4. [Key Requests Detailed](#4-key-requests-detailed)
5. [Response Data Structure](#5-response-data-structure)

---

## 1. Core Principles

The entire process involves two key operations: **Intercept** and **Transfer**. Understanding these concepts is essential for successful operation.

### 1.1 Subscription Payment Flow

When a user subscribes to ChatGPT on iOS, the complete payment chain is:

```
┌─────────────────────────────────────────────────────────────────┐
│                       Payment Flow                               │
│                                                                 │
│  iPhone ──▶ App Store Purchase ──▶ Apple returns payment token  │
│                                       (fetch_token)             │
│                                       │                         │
│                                       ▼                         │
│              ChatGPT App sends token to RevenueCat ──▶          │
│              Activates subscription                              │
│              (carries app_user_id to bind to specific account)   │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 Key Roles

| Role | Description |
|------|------|
| **App Store** | Handles real payment, returns Apple-signed payment token |
| **RevenueCat** | Third-party subscription management platform used by ChatGPT; validates tokens and activates subscriptions |
| **fetch_token** | Apple payment token (Apple Signed Transaction), proves payment was completed |
| **app_user_id** | ChatGPT account Account ID, determines which account the subscription binds to |

---

## 2. Intercept: Blocking Callback Requests

### 2.1 Why Intercept

After Apple payment completes, the ChatGPT App **automatically** sends the payment token to RevenueCat, binding the subscription to the currently logged-in ChatGPT account. If you want to transfer the subscription to another account, you must **prevent this automatic callback**.

### 2.2 URLs to Block

Set up **gateway blocking** (intercept / block) in Reqable for these two URLs:

| # | URL to Block | Reason |
|:---:|------------|----------|
| 1 | `https://api.revenuecat.com/v1/receipts` | Blocks Apple payment token automatic callback to RevenueCat server |
| 2 | `https://ios.chat.openai.com/backend-api/payments/rc/ios/verify/v4-2023-04-27` | Blocks ChatGPT backend automatic subscription status verification |

> **⚠️ Important**: Blocking these two URLs prevents automatic completion of the payment callback. If not blocked, the payment token automatically binds to the current ChatGPT account, making transfer impossible.

### 2.3 Intercept Sequence Diagram

```
Normal flow (no intercept):
iPhone pays ──▶ Token sent automatically ──▶ RevenueCat ──▶ Bound to current account ✅ (can't transfer)

Intercept flow:
iPhone pays ──▶ Token sent ──✖ Reqable blocks ──▶ Token doesn't reach RevenueCat
                                │
                                ▼
                    Manually capture token, modify, resend ──▶ Transfer to target account ✅
```

---

## 3. Transfer: Modifying User ID for Cross-Account Activation

### 3.1 Transfer Principle

After Apple payment completes, the ChatGPT App sends a token callback request to RevenueCat:

```
POST https://api.revenuecat.com/v1/receipts
```

This request body has two core fields:

| Field | Meaning | Purpose |
|------|------|------|
| `fetch_token` | Apple payment token (Apple Receipt) | Proves the user completed real payment via App Store; it's the "key" for subscription activation |
| `app_user_id` | ChatGPT Account ID | Determines **which ChatGPT account** this subscription binds to |

> **💡 Key Insight**: `fetch_token` is tied to the Apple ID (proves who paid), but unrelated to the ChatGPT account. RevenueCat **only uses `app_user_id`** to determine which user gets the subscription benefit.

### 3.2 Transfer Steps

1. Capture the complete token callback request via Reqable (must contain a valid `fetch_token`)
2. Replace `app_user_id` in the request body with the **target account's** Account ID
3. Resend the request to `https://api.revenuecat.com/v1/receipts`
4. RevenueCat receives a valid token and activates the subscription on the target account

### 3.3 Transfer Flow Diagram

```
Original request: fetch_token (payment token) + app_user_id = "Account A's ID"
                                        │
                                  Modify app_user_id
                                        │
                                        ▼
Transfer request: fetch_token (payment token) + app_user_id = "Account B's ID"
                                        │
                                        ▼
                            Account B gets Pro 20x subscription ✅
```

### 3.4 How to Get Target Account's app_user_id

The `app_user_id` for the target ChatGPT account can be obtained by:

1. With the target ChatGPT account logged in on the app
2. Use Reqable to capture any request sent to RevenueCat
3. Find the `app_user_id` field in the request body or URL

---

## 4. Key Requests Detailed

### 4.1 Apple Payment Token Callback Request

```
POST https://api.revenuecat.com/v1/receipts
```

**Key Headers:**

| Header | Value | Description |
|--------|----|------|
| `authorization` | `Bearer appl_rQLChslWRSKCUPBLPJtHCGjvujc` | RevenueCat API key |
| `x-client-bundle-id` | `com.openai.chat` | ChatGPT's Bundle ID |
| `x-platform` | `iOS` | Platform identifier |
| `x-storekit2-enabled` | `true` | Using StoreKit 2 |

**Key Body Fields:**

| Field | Value | Description |
|------|----|------|
| `fetch_token` | `eyJhbGci...` (JWS format) | Apple-signed payment token, **the core of transfer** |
| `app_user_id` | `46ee5a3b-97d3-4da0-baa8-061ecf9f1b25` | ChatGPT Account ID, **this field must be replaced** for transfer |
| `product_id` | `oai_chatgpt_go_1000_1m` | Subscription product ID |
| `price` | `8` | Price (USD) |
| `store_country` | `USA` | Store region |
| `transaction_id` | `440003340653016` | App Store transaction ID |

### 4.2 Fields to Modify During Transfer

| Field | Action | Description |
|------|------|------|
| `app_user_id` | ✏️ **Must modify** | Replace with target account's Account ID |
| `fetch_token` | ❌ Keep unchanged | This is Apple's signed valid payment token |
| `app_transaction` | ❌ Keep unchanged | Apple's application transaction token |
| Other fields | ❌ Keep unchanged | Including all parameters in headers |

---

## 5. Response Data Structure

### 5.1 Successful Response Example

After sending the token callback successfully, RevenueCat returns subscription info:

```json
{
  "request_date": "2026-09-17T07:47:20Z",
  "subscriber": {
    "entitlements": {
      "chatgpt_go": {
        "product_identifier": "oai_chatgpt_go_1000_1m",
        "purchase_date": "2026-09-17T07:38:26Z",
        "expires_date": "2026-10-17T07:38:26Z"
      }
    },
    "original_app_user_id": "46ee5a3b-97d3-4da0-baa8-061ecf9f1b25",
    "subscriptions": {
      "oai_chatgpt_go_1000_1m": {
        "price": { "amount": 8.0, "currency": "USD" },
        "store": "app_store",
        "period_type": "normal",
        "ownership_type": "PURCHASED",
        "original_purchase_date": "2026-09-17T07:38:26Z",
        "expires_date": "2026-10-17T07:38:26Z"
      }
    }
  }
}
```

### 5.2 Key Response Fields

| Field | Description |
|------|------|
| `entitlements` | Subscription entitlements owned by the current account |
| `original_app_user_id` | Account ID the subscription is bound to (should be the target account after transfer) |
| `expires_date` | Subscription expiration time |
| `ownership_type` | `PURCHASED` indicates successfully purchased |

---

> **Last Updated**: 2026-09-17
