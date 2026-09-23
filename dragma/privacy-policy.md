---
title: Privacy Policy
parent: Dragma
nav_order: 1
---

# Privacy Policy

**Last updated:** 2026-09-17

Dragma is built on one rule: **your data never leaves your device unless you
put it there yourself.** This policy is short because the app doesn't do much
with your data - that's deliberate.

## What data the app touches

- **Your accounts, transactions, categories, recurring templates, and
  Sub-Accounts** live entirely inside an `.xlsx` spreadsheet file that *you*
  choose the location of, using Android's standard file picker (Storage
  Access Framework). The app reads and writes that file, plus two companion
  folders next to it (`Backups/` for automatic pre-write safety snapshots and
  manual backups you trigger, and `Attachments/` for receipt images/documents
  you attach to transactions). None of this is copied anywhere else, uploaded,
  or sent to a server - Dragma doesn't have one.
- If you store your workspace in a cloud-synced folder (OneDrive, Google
  Drive, etc.), syncing is handled entirely by your own cloud storage app and
  account - not by Dragma. We never see that data; it's a transfer between
  your device and your own cloud provider.
- **Crash reports**, if the app ever crashes, are written to a file inside the
  app's private storage on your device only. Nothing is sent automatically. On
  next launch you're asked if you'd like to send it - if you say yes, your
  device's own email app opens with the report pre-filled so you can review or
  edit it before choosing to send it yourself. If you say no (or never open the
  app again), the report just sits on your device until it's cleared.
- **App Lock (biometric)**, if you turn it on, uses Android's own
  `BiometricPrompt` system. Your fingerprint/face data is managed entirely by
  your device's operating system and secure hardware - Dragma never sees it,
  stores it, or has access to it. The app only receives a yes/no "did this
  device's owner authenticate" result.
- **Dragma Pro** (the optional one-time in-app purchase that unlocks premium
  features) is handled entirely by Google Play Billing. Dragma never collects
  or sees your payment details (card numbers, billing address, etc.) - that's
  between you and Google. The app only receives a yes/no "is this purchase
  active" entitlement flag from Google Play.

## What the app does NOT do

- No account creation, no sign-in, no user identifiers of any kind
- No analytics, no ads, no third-party tracking SDKs
- No network access at all for the app's core functionality - Dragma does not
  declare the Android `INTERNET` permission itself. (Google Play Billing,
  used only for the optional Pro purchase, communicates with the Play Store
  app on your device directly rather than requiring the app to hold its own
  network permission.)
- No data is sold, shared, or given to any third party, because none is
  collected in the first place

## Permissions the app requests, and why

| Permission | Why |
|---|---|
| Storage (legacy, Android 12 and below only) | Reading/writing the spreadsheet workspace folder you choose |
| Biometric (system-level, only used if you enable App Lock) | Authenticating you to unlock the app - handled entirely by Android, see above |

## Changes to this policy

If this policy ever changes, the update will be reflected here with a new
"last updated" date and, once available, a matching changelog entry.

## Contact

Questions about this policy: **tritaildigital@gmail.com**
