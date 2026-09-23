---
title: Data Safety
parent: Paraclete
nav_order: 2
---

# Data Safety - what Paraclete does and doesn't collect

This page summarizes Paraclete's data practices in the same categories
Google Play uses on its store listing. See
[Privacy Policy](privacy-policy.md) for the full user-facing policy.

## Short answer

Paraclete does not collect or share any user data.

| Category | Collected? | Why |
|---|---|---|
| Location | No | Not requested, not used |
| Personal info | No | No name, email, address, phone, or any identifiers collected |
| Financial info | No | N/A |
| Health and fitness | No | N/A |
| Messages | No | N/A |
| Photos and videos | No | N/A |
| Audio files | No | N/A |
| Files and docs | No, user-managed | Your spreadsheet file is chosen and owned by you via Android's file picker; the app never uploads any of it |
| Calendar | No | N/A |
| Contacts | No | N/A |
| App activity | No | No analytics, no in-app search history collection, no install/uninstall tracking |
| Web browsing | No | N/A |
| App info and performance | No | No automatic crash log collection (no Crashlytics/Sentry/equivalent) |
| Device or other IDs | No | No device identifiers collected |

## Files and docs, in more detail

Paraclete reads and writes a file you explicitly select, entirely
on-device, and never transmits it anywhere. That's meaningfully different
from "collecting" data - the app never receives or has access to a copy. It
stays on your device; you control where the file lives and whether it's
synced through your own cloud provider.

## Crash reports

No automatic crash reporting service is used. If the app crashes, a report
is written to its own private on-device storage. Nothing is sent unless you
explicitly tap "Send" and complete an email through your own email app - a
user-initiated, user-controlled action, functionally identical to manually
emailing a screenshot.

## Is data encrypted in transit?

Not applicable - no user data is transmitted by the app at all.

## Can you delete your data?

Yes, trivially. All your reminders and categories live in a file you
already own and control. Deleting the spreadsheet file (or clearing
individual rows) *is* deleting the data. There's no account and no
server-side copy to separately request deletion of.

## Security practices

- Data is not encrypted in transit (not applicable - nothing is transmitted)
- You can request deletion at any time (see above)
- No independent third-party security review has been conducted (small
  independent project)

---

*Verified against the app's manifest (2026-08-27): no `INTERNET` or
`ACCESS_NETWORK_STATE` permission is declared anywhere in Paraclete's own
manifest source.*
