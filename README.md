# iOS ChatGPT Pro 20x — Activation Guide

> `hiroto/security-analysis` · iOS subscription manipulation
>
> Last updated: 2026-09-19

---

## Quick Navigation

```
┌──────────────────────────────────────────────────────┐
│  [1] Prerequisites           [2] Network Config      │
│  [3] Jailbreak Setup         [4] Subscription Flow   │
│  [5] Field Reference         [6] Process Overview    │
└──────────────────────────────────────────────────────┘
```

- [Prerequisites](#prerequisites)
- [Network Configuration](#network-configuration)
- [iOS Jailbroken Device Setup](#ios-jailbroken-device-setup)
- [Subscription Process](#subscription-process)
- [Key Request Fields Reference](#key-request-fields-reference)
- [Process Overview](#process-overview)

---

## Prerequisites

<!-- start -->

| # | Requirement | Details |
|:---:|------|------|
| 1 | **Jailbroken iOS device** | Sileo package manager required |
| 2 | **ChatGPT App** | Must launch without crash |
| 3 | **Computer with Reqable** | Windows or macOS |
| 4 | **Clash proxy** | On the same computer |
| 5 | **Same Wi-Fi network** | Phone + computer must be on the same LAN |

<!-- end -->

---

## Network Configuration

### Computer Setup

**1. Start Clash**

```
✅ Clash running on port 7890
❌ System proxy: OFF
❌ TUN mode: OFF
```

**2. Configure Reqable Secondary Proxy**

```
Protocol : HTTP
Address  : 127.0.0.1
Port     : 7890          ← Clash upstream
Listen   : 9000          ← Reqable itself
```

### Phone Setup

```
Settings → Wi-Fi → (active network) → Proxy → Manual
```

| Field | Value |
|-------|-------|
| Server | Computer LAN IP (`192.168.x.x`) |
| Port | `9000` (Reqable) |

> **💡 Tip:** Run `ifconfig` (macOS) or `ipconfig` (Windows) to find your LAN IP.

### Network Topology

```
┌──────┐  Wi-Fi:9000   ┌─────────┐  HTTP:7890   ┌──────┐
│ iPhone │ ──────────▶ │ Reqable │ ──────────▶ │ Clash│
└──────┘             └─────────┘              └──────┘
                                               │
                                               ▼
                                          Internet
```

---

## iOS Jailbroken Device Setup

### Required Plugins (Sileo)

| Plugin | Purpose |
|--------|---------|
| **SSL Kill Switch 3** | Bypass SSL pinning → intercept HTTPS |
| **Choicy** | Control which processes receive the tweak |

### SSL Kill Switch 3

1. Open the plugin
2. Ensure toggle is **ON**
3. Global mode is sufficient

### Choicy — Daemon Injection

Open **Choicy → Daemons**, enable **SSL Kill Switch 3** for:

```
  ✅ cloudd
  ✅ amsaccountsd
  ✅ identityservicesd  (skip if missing)
  ✅ akd
  ✅ nsurlsessiond
```

> **⚠️ Critical:** These processes handle Apple ID auth and network. SSL Kill Switch must be injected into ALL of them for Reqable to capture App Store subscription requests.

---

## Subscription Process

### Prerequisite Check

Pro 20x works for two account types:

**Case A** — Apple ID has subscribed to ChatGPT before
(Any tier: Go, Plus, Pro 5x — doesn't matter)

**Case B** — Never subscribed to ChatGPT
→ First subscribe to **Go** or **Plus**, then continue

### Execution Steps

#### Step 1 — Open Subscription Page

```
ChatGPT App → Settings → Subscription → View All Plans
```

#### Step 2 — Select & Tap Subscribe

- Pick any plan (Plus Annual recommended)
- Tap **Subscribe**
- ⚠️ Do NOT confirm payment yet

#### Step 3 — Capture in Reqable

On your computer, look for:

```
POST https://p44-buy.itunes.apple.com/WebObjects/MZBuy.woa/wa/buyProduct
```

#### Step 4 — Rewrite Request Body

Intercept the request and modify these 3 fields:

| Field | Original (Plus Annual) | Target (Pro 20x) |
|-------|------------------------|------------------|
| `offerName` | `oai_chatgpt_plus_20000_1y` | `oai_chatgpt_pro_20000_1m` |
| `price` | `200000` | `200000` (unchanged) |
| `salableAdamId` | `6745416289` | `6657954405` |

> **Why it works:** Pro 20x is hidden on the frontend. By swapping `offerName` and `salableAdamId` while keeping the same price ($200), we trick the system into activating a Pro tier on a Plus payment.

#### Step 5 — Confirm

Release the intercepted request. On success:

```
🎉 ChatGPT Pro 20x subscription popup appears on iPhone
```

---

## Key Request Fields Reference

### Plan Comparison

| Plan | `offerName` | `salableAdamId` | `mtSubscriptionAdamId` | `price` |
|------|-------------|-----------------|------------------------|---------|
| Go · $8/mo | `oai_chatgpt_go_1000_1m` | `6749460546` | — | `8000` |
| Plus · $19.99/mo | `oai_chatgpt_plus_1999_1m` | `6448311597` | `6749460546` | `19990` |
| Plus · $200/yr | `oai_chatgpt_plus_20000_1y` | `6745416289` | `6749460546` | `200000` |
| Pro 5x · $100/mo | `oai_chatgpt_pro_10000_1m` | `6759817441` | `6749460546` | `100000` |
| **Pro 20x · $200/mo** | `oai_chatgpt_pro_20000_1m` | `6657954405` | `6749460546` | `200000` |

> Price unit = **0.001 USD** (`200000` = $200.00). Go plans lack `mtSubscriptionAdamId`; all others share group ID `6749460546`.

### App Store Purchase Request

```
POST https://p44-buy.itunes.apple.com/WebObjects/MZBuy.woa/wa/buyProduct
```

Apple Plist XML body — key fields for Pro 20x:

| Field | Value |
|-------|-------|
| `appAdamId` | `6448311069` |
| `bid` | `com.openai.chat` |
| `offerName` | `oai_chatgpt_pro_20000_1m` |
| `price` | `200000` |
| `salableAdamId` | `6657954405` |
| `mtSubscriptionAdamId` | `6749460546` |
| `buySubscription` | `true` |

---

## Process Overview

```
┌──────────────────────────────────────────────────────────────┐
│  PHASE 1 — PREPARATION                                       │
│  ─────────────────────────────────────────────────────────── │
│  1. Jailbroken iOS device                                    │
│  2. Install SSL Kill Switch 3 + Choicy                       │
│  3. Install Clash + Reqable on computer                      │
│  4. Configure proxy chain (Wi-Fi → Reqable → Clash)          │
└──────────────────────────┬───────────────────────────────────┘
                           ▼
┌──────────────────────────────────────────────────────────────┐
│  PHASE 2 — CONFIGURATION                                     │
│  ─────────────────────────────────────────────────────────── │
│  1. Reqable secondary proxy → Clash:7890                     │
│  2. Phone Wi-Fi proxy → Reqable:9000                         │
└──────────────────────────┬───────────────────────────────────┘
                           ▼
┌──────────────────────────────────────────────────────────────┐
│  PHASE 3 — EXECUTION                                         │
│  ─────────────────────────────────────────────────────────── │
│  1. ChatGPT → Settings → Subscription → View Plans           │
│  2. Select plan → tap Subscribe (don't confirm)              │
│  3. Reqable catches buyProduct request                       │
│  4. Rewrite offerName + salableAdamId                        │
│  5. Release → Pro 20x popup on iPhone 🎉                     │
└──────────────────────────────────────────────────────────────┘
```

---

> `hiroto/security-analysis` · iOS subscription manipulation · 2026
