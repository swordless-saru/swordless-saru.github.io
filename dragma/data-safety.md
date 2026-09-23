---
title: Data Safety
parent: Dragma
nav_order: 2
---

# Data Safety - what Dragma does and doesn't collect

This page summarizes Dragma's data practices in the same categories Google
Play uses on its store listing. See [Privacy Policy](privacy-policy.md) for
the full user-facing policy.

## Short answer

Dragma does not collect or share any user data.

| Category | Collected? | Why |
|---|---|---|
| Location | No | Not requested, not used |
| Personal info | No | No name, email, address, phone, or any identifiers collected |
| Financial info | No | No payment details reach the app - Google Play Billing handles purchases entirely |
| Health and fitness | No | N/A |
| Messages | No | N/A |
| Photos and videos | No, user-managed | Receipt attachments are chosen by you and stored in your own on-device `Attachments/` folder; never uploaded |
| Audio files | No | N/A |
| Files and docs | No, user-managed | Your spreadsheet workspace (spreadsheet file + `Backups/` + `Attachments/`) is chosen and owned by you via Android's file picker; the app never uploads any of it |
| Calendar | No | N/A |
| Contacts | No | N/A |
| App activity | No | No analytics, no in-app search history collection, no install/uninstall tracking |
| Web browsing | No | N/A |
| App info and performance | No | No automatic crash log collection (no Crashlytics/Sentry/equivalent) |
| Device or other IDs | No | No device identifiers collected |

## Files and docs, in more detail

Dragma reads and writes files you explicitly select, entirely on-device, and
never transmits them anywhere. That's meaningfully different from
"collecting" data - the app never receives or has access to a copy. It stays
on your device; you control where the file and folders live and whether
they're synced through your own cloud provider.

## Crash reports

No automatic crash reporting service is used. If the app crashes, a report
is written to its own private on-device storage. Nothing is sent unless you
explicitly tap "Send" and complete an email through your own email app - a
user-initiated, user-controlled action, functionally identical to manually
emailing a screenshot.

## In-app purchases (Dragma Pro)

Purchases are processed entirely by Google Play Billing. Dragma never
receives card numbers, billing addresses, or any other payment detail - the
app only receives a purchase/entitlement state (active or not) from the Play
Billing library. That entitlement flag is not financial info collected by
Dragma; Google processes and holds the actual payment data under its own
privacy policy.

## Biometric App Lock

App Lock authentication goes through Android's own `BiometricPrompt` API.
Fingerprint/face data is managed by your device's operating system and
secure hardware - the app only receives a pass/fail authentication result.
No biometric data is collected, stored, or transmitted by Dragma.

## Is data encrypted in transit?

Not applicable - no user data is transmitted by the app at all.

## Can you delete your data?

Yes, trivially. All financial data lives in a file you already own and
control. Deleting the spreadsheet workspace (or clearing individual rows)
*is* deleting the data. There's no account and no server-side copy to
separately request deletion of. (Dragma Pro's purchase record lives in your
own Google Play account, managed through Google's own tools - Dragma never
held it and couldn't delete it even if asked.)

## Security practices

- Data is not encrypted in transit (not applicable - nothing is transmitted)
- You can request deletion at any time (see above)
- No independent third-party security review has been conducted (small
  independent project)

---

*Verified against the app's manifest (2026-08-27): no `INTERNET` or
`ACCESS_NETWORK_STATE` permission is declared anywhere in Dragma's own
manifest source. Only legacy, SDK-version-gated storage permissions are
declared. Google Play Billing's own manifest entries (for its Play Store
communication channel) are not app-declared network permissions and don't
change this.*
