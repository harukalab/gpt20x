# iOS ChatGPT Pro 20x Activation Guide

Built & maintained by **hiroto** 👑

---

> **⚠️ Disclaimer**: This guide is for educational and research purposes. Please comply with relevant laws, regulations, and terms of service.

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Network Configuration](#network-configuration)
3. [iOS Jailbroken Device Setup](#ios-jailbroken-device-setup)
4. [Subscription Process](#subscription-process)
5. [Key Request Fields Reference](#key-request-fields-reference)
---

## Prerequisites
Before you begin, ensure you have the following ready:

| # | Requirement | Description |
|:---:|------|------|
| 1 | A **jailbroken** iOS device | Must have Sileo package manager installed |
| 2 | **ChatGPT** App working on the device | Ensure the app version launches normally |
| 3 | A computer with **Reqable** installed | Supports Windows / macOS |
| 4 | **Clash** proxy tool on the computer | For internet access via proxy |
| 5 | Computer and phone on the **same Wi-Fi** network | For man-in-the-middle packet capture |

---

## Network Configuration
### Computer Setup

1. **Start Clash**
   - ✅ Ensure Clash is running (default port `7890`)
   - ❌ **Disable** system proxy
   - ❌ **Disable** virtual network adapter (TUN mode)

2. **Configure Reqable as a Secondary Proxy**
   - Open Reqable
   - Create a **secondary proxy rule** with the following settings:
     ```
     Protocol: HTTP
     Address: 127.0.0.1
     Port: 7890 (Clash's running port)
     ```
   - Reqable listens on port `9000`

### Phone Setup

1. Open **Settings → Wi-Fi → Tap the connected network → Proxy**
2. Select **Manual**
3. Fill in proxy details:

| Field | Value |
|------|-----|
| Server | Computer's LAN IP address (e.g., `192.168.x.x`) |
| Port | `9000` (Reqable's listening port) |

> **💡 Tip**: Run `ifconfig` (macOS) or `ipconfig` (Windows) on your computer to find your LAN IP.

### Network Flow Diagram

```
iPhone ──(Wi-Fi Proxy 9000)──▶ Reqable ──(Secondary Proxy 7890)──▶ Clash ──▶ Internet
```

---

## iOS Jailbroken Device Setup
### Install Required Plugins

Install these two plugins in the **Sileo** store on your jailbroken device:

| Plugin Name | Purpose |
|--------|------|
| **SSL Kill Switch 3** | Bypass SSL Pinning to enable HTTPS request interception |
| **Choicy** | Control which processes Tweak is injected into |

### Configure SSL Kill Switch 3

1. Open SSL Kill Switch 3
2. Ensure the toggle is **ON**
3. Default global mode is fine

### Configure Choicy

Open **Choicy → Daemons**, and enable **SSL Kill Switch 3** for the following 5 system processes:

```
✅ cloudd
✅ amsaccountsd
✅ identityservicesd (skip if not found)
✅ akd
✅ nsurlsessiond
```

> **⚠️ Important**: These processes handle Apple ID authentication and network communication. You must inject SSL Kill Switch into them for Reqable to correctly intercept App Store subscription requests.

---

## Subscription Process
### Prerequisite Check

ChatGPT Pro 20x subscription works for these **two account types**:

- **Case A**: Apple ID that **has previously subscribed** to ChatGPT (any tier: Go, Plus, Pro 5x)
- **Case B**: **Never subscribed** to ChatGPT before (brand new account)

> **📌 Note**: For Case B (never subscribed), you must first subscribe to a **Go or Plus** membership, then proceed with the steps below. If you already have a subscription history, skip this step.

### Step-by-Step

#### Step 1: Open the Subscription Page

```
ChatGPT App → Settings → Subscription → View All Plans
```

#### Step 2: Select Any Plan and Tap Subscribe

- Choose any plan from the list (e.g., Plus Annual)
- Tap the **Subscribe** button
- ⚠️ **Do NOT confirm payment immediately!**

#### Step 3: Intercept the Key Request

In Reqable on your computer, find this request:

```
POST https://p44-buy.itunes.apple.com/WebObjects/MZBuy.woa/wa/buyProduct
```

#### Step 4: Rewrite the Request Body

Intercept and modify these 3 key fields in the **Request Body**:

| Field | Original Value (Example: Plus Annual) | Replace With (Pro 20x Monthly) |
|------|------------------------|----------------------|
| `offerName` | `oai_chatgpt_plus_20000_1y` | `oai_chatgpt_pro_20000_1m` |
| `price` | `200000` | `200000` (unchanged) |
| `salableAdamId` | `6745416289` | `6657954405` |

> **📌 Note**: The ChatGPT Pro 20x plan is hidden on the frontend page, so you cannot select it directly in the App. You must intercept the App Store purchase request and replace the product identifier fields in the request body.

#### Step 5: Confirm Subscription

After rewriting the request body, release the request. If successful:

```
🎉 Congratulations! The ChatGPT Pro 20x subscription confirmation popup will appear on your phone!
```

---

## Key Request Fields Reference
### ChatGPT Plan Comparison Table

All ChatGPT iOS subscription plan fields below. Replace as needed:

| Plan | `offerName` | `salableAdamId` | `mtSubscriptionAdamId` | `price` |
|------|-------------|-----------------|------------------------|---------|
| **Go** Monthly $8 | `oai_chatgpt_go_1000_1m` | `6749460546` | ❌  | `8000` |
| **Plus** Monthly $19.99 | `oai_chatgpt_plus_1999_1m` | `6448311597` | `6749460546` | `19990` |
| **Plus** Annual $200 | `oai_chatgpt_plus_20000_1y` | `6745416289` | `6749460546` | `200000` |
| **Pro 5x** Monthly $100 | `oai_chatgpt_pro_10000_1m` | `6759817441` | `6749460546` | `100000` |
| **Pro 20x** Monthly $200 | `oai_chatgpt_pro_20000_1m` | `6657954405` | `6749460546` | `200000` |

> **💡 Note**: `price` unit is **0.001 USD** (e.g., `200000` = $200.00). Go plans do not have the `mtSubscriptionAdamId` field; all other plans share the subscription group ID `6749460546`.

### App Store Purchase Request

```
POST https://p44-buy.itunes.apple.com/WebObjects/MZBuy.woa/wa/buyProduct
```

**Request body is Apple Plist XML format. Key fields:**

| Field | Description | Example Value (Pro 20x) |
|------|------|------------------|
| `appAdamId` | ChatGPT App ID | `6448311069` |
| `bid` | Bundle ID | `com.openai.chat` |
| `offerName` | Subscription plan name | `oai_chatgpt_pro_20000_1m` |
| `price` | Price (in 0.001 USD) | `200000` |
| `salableAdamId` | Saleable Product ID | `6657954405` |
| `mtSubscriptionAdamId` | Subscription Group ID | `6749460546` |
| `buySubscription` | Is a subscription purchase | `true` |

---

## Process Overview
```
┌──────────────────────────────────────────────────────────┐
│                      Preparation Phase                     │
│  1. Jailbroken iOS device                                │
│  2. Install SSL Kill Switch 3                            │
│  3. Install Clash + Reqable on computer                   │
│  4. Configure network proxy chain                         │
└──────────────────────┬───────────────────────────────────┘
                       ▼
┌──────────────────────────────────────────────────────────┐
│                      Configuration Phase                   │
│  1. Set Reqable secondary proxy (→ Clash:7890)            │
│  2. Point phone Wi-Fi proxy to Reqable:9000               │
└──────────────────────┬───────────────────────────────────┘
                       ▼
┌──────────────────────────────────────────────────────────┐
│                      Execution Phase                       │
│  1. ChatGPT App → Settings → Subscription → View Plans    │
│  2. Select any plan, tap Subscribe                         │
│  3. Reqable intercepts buyProduct request                  │
│  4. Replace offerName / salableAdamId fields              │
│  5. Release request → Pro 20x subscription popup 🎉       │
└──────────────────────────────────────────────────────────┘
```

---

> **Last Updated**: 2026-09-19
