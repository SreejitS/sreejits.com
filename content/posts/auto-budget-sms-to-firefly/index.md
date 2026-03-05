---
title: "Auto-Budget: Turning Bank SMS into a Self-Hosted Budget Dashboard"
date: 2026-03-03
tags: ["automation", "python", "self-hosted", "macos", "raspberry-pi"]
draft: false
---

I bank with Riyad Bank in Saudi Arabia. Every transaction triggers an SMS - in Arabic. I wanted these to automatically appear in a budget dashboard without trusting a third-party app with my financial data.

The result is [auto-budget](https://github.com/SreejitS/auto-budget): a pipeline that reads bank SMS, parses Arabic transaction messages with regex, categorizes merchants using rules (and optionally Claude AI), and pushes everything to a self-hosted [Firefly III](https://www.firefly-iii.org/) instance.

No cloud. No fintech app. No screen-scraping. Just SMS in, budget out.

## The Problem

Riyad Bank sends SMS notifications for every transaction. They look like this:

```
شراء إنترنت
من: AMAZON.SA
بطاقة: 5109*
مبلغ: SAR 247.00
رصيد: SAR 6,612.19
```

That says: online purchase from Amazon, card ending 5109, amount 247 SAR, remaining balance 6,612.19 SAR.

There are about 14 different transaction types - purchases, ATM withdrawals, transfers, salary deposits, refunds, bill payments - each with slightly different formatting. Some include foreign currency. Some have the amount before the currency code, some after. Some use commas as thousand separators.

I wanted all of these parsed and categorized automatically.

## The Architecture

The system has two paths to get SMS into Firefly III:

**Mac (backup, every 15 minutes):** Reads directly from the macOS iMessage database (`~/Library/Messages/chat.db`), parses the SMS text with Python regex, categorizes, and pushes to Firefly III.

**iPhone (real-time):** iOS Shortcuts automation triggers on bank SMS, extracts the safe fields (amount, merchant, currency - never card numbers or OTPs), and POSTs to a Flask API running on a Raspberry Pi. The RPi handles categorization and pushes to Firefly III.

Both paths use the same categorizer and Firefly client. Deduplication prevents double-counting when both paths process the same message.

## Parsing Arabic SMS

The parser handles 3000+ observed SMS formats from Riyad Bank. The core extraction is straightforward regex:

**Amount** - multiple formats because the bank isn't consistent:
```python
# مبلغ: SAR 1,574.75  OR  مبلغ: 412.60 SAR  OR  مبلغ SAR 241.14
amount_patterns = [
    r'مبلغ[:\s]*([A-Z]{3})\s*([\d,]+\.\d{2})',   # currency first
    r'مبلغ[:\s]*([\d,]+\.\d{2})\s*([A-Z]{2,3})',  # amount first
]
```

**Merchant** - extracted from one of several fields:
```python
# من: AMAZON.SA  OR  لدى: STARBUCKS  OR  إلى: JOHN DOE
merchant_patterns = [
    r'من:\s*(.+)',    # from (most common)
    r'لدى:\s*(.+)',   # at (payments)
    r'إلى:\s*(.+)',   # to (transfers)
]
```

**Transaction type** - the first line of the SMS:
```python
type_map = {
    'شراء إنترنت': 'online_purchase',
    'شراء عبر نقاط بيع': 'pos_purchase',
    'سحب فرع/صراف': 'atm_withdrawal',
    'حوالة صادرة': 'outgoing_transfer',
    'نوع العملية: راتب': 'salary',
    # ... 14 types total
}
```

Before any of this runs, a pre-filter checks ~40 patterns for non-transaction messages (OTPs, promotions, card activations, insufficient balance alerts) and skips them.

## Categorization: Rules First, AI Second

Categorization uses a three-tier approach:

**Tier 1 - Merchant cache:** If we've seen this merchant before, reuse the cached category. One API call per unique merchant, ever.

**Tier 2 - Keyword rules:** 200+ keyword mappings across 13 categories. Covers ~90% of transactions with zero API calls:

```python
KEYWORD_RULES = {
    "Dining": ["hungerstation", "talabat", "starbucks", "mcdonald", ...],
    "Groceries": ["panda", "tamimi", "danube", "lulu", "carrefour", ...],
    "Transport": ["careem", "uber", "bolt", "fuel", "saudia", ...],
    "Shopping": ["amazon", "noon", "shein", "namshi", "h&m", ...],
    # ...
}
```

**Tier 3 - Claude AI:** For unknown merchants, send only the merchant name to Claude's API. Never the SMS text, never card numbers, never balances. The result gets cached, so each unknown merchant costs exactly one API call.

If the API is down or out of credits, the system still works - it just files unknowns under "Other".

## The Mac Route: Why It Failed

The initial idea was simple: bank SMS arrives on iPhone, forwards to Mac via iMessage, Python reads `~/Library/Messages/chat.db`, parses, categorizes, done.

Two problems killed this approach. One was annoying. The other was fatal.

### Problem 1: macOS Won't Let You Read chat.db

All iMessage data lives in `~/Library/Messages/chat.db` - a SQLite database protected by macOS TCC (Transparency, Consent, and Control). Any process that wants to read this file needs Full Disk Access (FDA).

This sounds simple. It is not.

**LaunchAgent with Python script.** Python tries to read chat.db, gets "authorization denied". The Python binary doesn't have FDA - the user granted FDA to Terminal.app, not to `/usr/local/bin/python3`.

**Wrapping Python in a .app bundle.** Built a macOS app bundle with a C launcher that `exec()`s Python. Granted the .app FDA. Still denied - `exec()` creates a new process, and TCC checks the "responsible process", which is the C binary, not the .app bundle.

**AppleScript app via osacompile.** Created a `.app` that copies chat.db using `do shell script "cp ..."`. FDA does not propagate to child processes. The `do shell script` spawns `/bin/sh` as a child.

**AppleScript with native file I/O.** Used `open for access` / `read` / `write` instead of shelling out. Got "File permission error (-54)". Even AppleScript's native file operations are blocked by TCC.

**cron + FDA.** macOS blocks adding SIP-protected system binaries like `/usr/sbin/cron` to the FDA list.

**Ad-hoc signed binary.** `codesign -s -` produces a CDHash that changes every time you recompile. The TCC grant silently becomes invalid. No error, no warning - it just stops working.

The root cause: macOS TCC uses a "responsible process" model. Terminal.app has FDA, so anything spawned inside Terminal inherits that access. But LaunchAgents, .app bundles, and cron jobs each become their own responsible process, and unless that exact binary (exact CDHash) has FDA, access is denied.

The eventual workaround: a `.command` file added as a Login Item. macOS opens `.command` files in Terminal.app, so the Python process inherits Terminal's FDA. A bash while loop with `sleep 900` runs the sync every 15 minutes.

Weeks of debugging C binaries, AppleScript, code signing, and TCC databases - solved by a bash while loop inside Terminal.

### Problem 2: SMS Forwarding Just Stopped

This is what actually killed the Mac approach.

The entire pipeline depends on bank SMS forwarding from iPhone to Mac via iMessage. After setting everything up, the forwarding silently stopped working. No error, no notification - messages just stopped appearing on the Mac.

iMessage (blue bubble) messages synced fine. SMS (green bubble) messages from the bank did not. The Text Message Forwarding toggle in iPhone settings was on. Both devices were on the same Apple ID. Restarting both devices didn't help.

This is a known flaky behavior in Apple's SMS forwarding - it can break after iOS updates, network changes, or seemingly nothing at all. And there's no way to monitor or alert on it. You just stop getting messages and don't notice until you check.

A pipeline that silently stops receiving data without any indication is not a pipeline you can trust for financial tracking. The Mac route was dead.

## The RPi Route: What Actually Works (Mostly)

Cut out the Mac entirely. The SMS arrives on the iPhone - process it right there and send the result to a server.

A Flask API runs on a Raspberry Pi behind Tailscale (VPN mesh network). An iOS Shortcut on the home screen takes the copied SMS text, extracts the safe fields (amount, merchant, currency, transaction type), and POSTs them to the RPi. The RPi runs the same categorizer and pushes to Firefly III.

Raw SMS text, card numbers, OTPs, balances - none of that leaves the phone.

```
POST /api/transaction
{
    "amount": 48.00,
    "currency": "SAR",
    "merchant": "VOX CINEMAS",
    "type": "withdrawal",
    "date": "2026-03-03T14:30:00"
}
```

The server categorizes ("Entertainment"), creates the Firefly transaction, and returns:

```json
{
    "status": "created",
    "category": "Entertainment",
    "firefly_id": 1234
}
```

No iMessage forwarding. No FDA. No Terminal window sitting open.

### The Automation Gap

There's a catch: iOS Shortcuts automations can only trigger on iMessage, not SMS. Bank notifications come as SMS. So the "automation" isn't fully automatic - when a bank SMS arrives, I copy the text, tap the Shortcut on my home screen, and it handles the rest.

It's low friction - two taps instead of zero - but it's not the hands-free pipeline I wanted. I'm still looking for ways to close this gap. Options I'm exploring:

- A notification-based trigger (if Apple ever opens SMS to Shortcuts automations)
- An accessibility-layer approach that watches for bank notifications
- Just accepting the two-tap workflow - it takes 3 seconds and I haven't missed a transaction yet

## What I'd Do Differently

**Start with the iPhone + RPi path.** The Mac route was a months-long detour. The FDA fight was solvable, but SMS forwarding breaking silently made the whole approach unreliable. The iPhone is where the SMS arrives - process it there.

**Accept "semi-automatic" earlier.** I spent a lot of time chasing full automation. The reality is that iOS doesn't let you trigger Shortcuts on SMS - only iMessage. The two-tap workflow (copy SMS, tap Shortcut) is good enough. Perfect automation was never on the table without jailbreaking.

**Skip AI categorization initially.** The keyword rules cover 90% of transactions. The Claude integration is nice but not essential - I could have shipped without it.

**Don't over-parse.** I handle 14 transaction types and multiple currency formats. For a personal budget, I probably needed 3: purchases, transfers, and salary. The rest could have been "Other" and I'd have been fine.

The code is at [github.com/SreejitS/auto-budget](https://github.com/SreejitS/auto-budget).
