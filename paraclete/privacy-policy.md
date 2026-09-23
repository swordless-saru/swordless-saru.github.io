---
title: Privacy Policy
parent: Paraclete
nav_order: 1
---

# Privacy Policy

**Last updated:** 2026-09-17

Paraclete (dev codename Kerameion) is built on one rule: **your data never
leaves your device unless you put it there yourself.** This policy is short
because the app doesn't do much with your data - that's deliberate.

## What data the app touches

- **Your nudges and categories** live entirely inside a `.xlsx` spreadsheet
  file that *you* choose the location of, using Android's standard file
  picker (Storage Access Framework). The app reads and writes that file. It
  does not copy your data anywhere else, does not upload it, and has no
  server to upload it to.
- If you store that file in a cloud-synced folder (OneDrive, Google Drive,
  etc.), syncing is handled entirely by your own cloud storage app and
  account - not by Paraclete. We never see that data; it's a transfer
  between your device and your own cloud provider.
- **Crash reports**, if the app ever crashes, are written to a file inside
  the app's private storage on your device only. Nothing is sent
  automatically. On next launch you're asked if you'd like to send it - if
  you say yes, your device's own email app opens with the report pre-filled
  so you can review or edit it before choosing to send it yourself. If you
  say no (or never open the app again), the report just sits on your device
  until it's cleared.

## What the app does NOT do

- No account creation, no sign-in, no user identifiers of any kind
- No analytics, no ads, no third-party tracking SDKs
- No network access at all - the app does not declare the Android
  `INTERNET` permission, so it is technically incapable of sending data
  over a network by itself, automatic or otherwise
- No data is sold, shared, or given to any third party, because none is
  collected in the first place

## Permissions the app requests, and why

| Permission | Why |
|---|---|
| Notifications | To show you nudge reminders |
| Boot-completed | To reschedule your nudges after your device restarts |
| Vibrate | Notification vibration |
| Storage (legacy, older Android versions only) | Reading/writing the spreadsheet file you choose |

## Changes to this policy

If this policy ever changes, the update will be reflected here with a new
"last updated" date and, once available, a matching changelog entry.

## Contact

Questions about this policy: **tritaildigital@gmail.com**
