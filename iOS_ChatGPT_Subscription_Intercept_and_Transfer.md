# iOS ChatGPT — Intercept & Transfer

> `hiroto/security-analysis` · RevenueCat token manipulation
>
> Last updated: 2026-09-19

---

## Quick Navigation

- [Core Principles](#core-principles)
- [Intercept — Blocking Callbacks](#intercept--blocking-callbacks)
- [Transfer — Cross-Account Activation](#transfer--cross-account-activation)
- [Key Requests](#key-requests)
- [Response Structure](#response-structure)

---

## Core Principles

The full operation has two phases: **Intercept** and **Transfer**. Understanding both is required.

### Payment Flow

```
┌──────────────────────────────────────────────────────────────┐
│  SUBSCRIPTION PAYMENT PIPELINE                               │
│  ─────────────────────────────────────────────────────────── │
│                                                              │
│  iPhone ──▶ App Store Purchase                               │
│             │                                                 │
│             ├─▶ Apple returns fetch_token (JWS signed)       │
│             │                                                 │
│             ▼                                                 │
│  ChatGPT App ──▶ RevenueCat POST /receipts                  │
│             │          carries app_user_id                   │
│             │          binds subscription to account         │
│             ▼                                                 │
│  Entitlement activated ✅                                    │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### Key Roles

| Entity | Responsibility |
|--------|----------------|
| **App Store** | Processes real payment, returns Apple-signed `fetch_token` |
| **RevenueCat** | Third-party subscription manager — validates token, activates entitlement |
| **fetch_token** | Apple Signed Transaction (JWS). Proves payment completed. |
| **app_user_id** | ChatGPT Account UUID. Determines which account gets the subscription. |

---

## Intercept — Blocking Callbacks

### Why Intercept?

After payment, ChatGPT App **automatically** POSTs `fetch_token` to RevenueCat, binding the subscription to the current account. To transfer, we must block this auto-send.

### URLs to Block in Reqable

| # | URL | Purpose |
|---|-----|---------|
| 1 | `https://api.revenuecat.com/v1/receipts` | Blocks token callback to RevenueCat |
| 2 | `https://ios.chat.openai.com/backend-api/payments/rc/ios/verify/v4-2023-04-27` | Blocks backend subscription verification |

> **⚠️ Required:** Both URLs must be blocked. If not blocked, the token auto-binds to the current account and transfer becomes impossible.

### Flow Comparison

```
  Normal (no intercept):
  ──────────────────────
  iPhone pays ──▶ Token auto-sent ──▶ RevenueCat ──▶ Bound to current account ✅
                                                                 (cannot transfer)

  Intercepted:
  ───────────
  iPhone pays ──▶ Token sent ──✖ Reqable blocks ──▶ RevenueCat never receives it
                                                    │
                                                    ▼
                                     Manually capture, modify, resend
                                                    │
                                                    ▼
                                             Transfer to target account ✅
```

---

## Transfer — Cross-Account Activation

### Principle

After payment, ChatGPT App POSTs to RevenueCat:

```
POST https://api.revenuecat.com/v1/receipts
```

Two critical fields in the body:

| Field | Meaning | What it controls |
|-------|---------|------------------|
| `fetch_token` | Apple receipt (JWS) | Proves payment via App Store — the activation "key" |
| `app_user_id` | ChatGPT Account UUID | **Which account** receives the subscription |

> **Key insight:** `fetch_token` is tied to Apple ID (who paid), but RevenueCat only uses `app_user_id` to decide who gets the entitlement. The two are not cross-verified.

### Transfer Steps

1. Capture full callback request via Reqable (must contain valid `fetch_token`)
2. Replace `app_user_id` with **target account** UUID
3. POST to `https://api.revenuecat.com/v1/receipts`
4. RevenueCat validates token → activates subscription on target account

### Transfer Diagram

```
  Original:  fetch_token + app_user_id = "Account A"
                                    │
                                    │  ← change app_user_id
                                    ▼
  Transfer:  fetch_token + app_user_id = "Account B"
                                    │
                                    ▼
                         Account B gets Pro 20x ✅
```

### Finding Target `app_user_id`

1. Log into ChatGPT App with the **target account**
2. Capture any RevenueCat request via Reqable
3. Extract `app_user_id` from body or URL

---

## Key Requests

### RevenueCat Receipt Callback

```
POST https://api.revenuecat.com/v1/receipts
```

**Headers:**

```
authorization        : Bearer appl_rQLChslWRSKCUPBLPJtHCGjvujc
x-client-bundle-id   : com.openai.chat
x-platform           : iOS
x-storekit2-enabled  : true
```

**Body fields:**

| Field | Example | Description |
|-------|---------|-------------|
| `fetch_token` | `eyJhbGci...` | Apple JWS receipt — **core of transfer** |
| `app_user_id` | `46ee5a3b-...f1b25` | ChatGPT UUID — **replace for transfer** |
| `product_id` | `oai_chatgpt_go_1000_1m` | Subscription product ID |
| `price` | `8` | Price in USD |
| `store_country` | `USA` | Store region |
| `transaction_id` | `440003340653016` | App Store transaction ID |

### Fields to Modify

| Field | Action | Reason |
|-------|--------|--------|
| `app_user_id` | ✏️ **Change** | Target account's UUID |
| `fetch_token` | ❌ Keep | Apple-signed, must remain valid |
| `app_transaction` | ❌ Keep | Apple application transaction token |
| Other fields | ❌ Keep | All headers and body params unchanged |

---

## Response Structure

### Success Response (RevenueCat)

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

### Response Fields

| Field | Meaning |
|-------|---------|
| `entitlements` | Active subscription entitlements for the account |
| `original_app_user_id` | Bound account ID (should be target account after transfer) |
| `expires_date` | Subscription expiration timestamp |
| `ownership_type` | `PURCHASED` = successfully activated |

---

> `hiroto/security-analysis` · iOS subscription manipulation · 2026
