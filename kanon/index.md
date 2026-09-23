---
title: Kanon
nav_order: 4
has_children: true
---

# Android Development Standards

Conventions for building Android apps with Kotlin and Jetpack Compose,
extracted from real shipped applications rather than written from theory.

Every rule here earned its place by being read out of working code. Where
a rule exists because of a specific bug, the bug's mechanism is described
— not just the rule.

---

## Contents

| Doc | Covers |
|---|---|
| [Project Setup](android/project-setup.md) | Version baseline, `MainActivity`, DI, package structure, feature flags, git hygiene |
| [Proven Version Pairings](android/version-pairings.md) | Ready-to-use AGP/Gradle/Kotlin/KSP/Hilt combos, verified not guessed |
| [App Shell](android/app-shell.md) | Scaffold placement, window insets, bottom nav + FAB, navigation |
| [Theming](android/theming.md) | Semantic palette, contrast requirements, typography, icons, spacing, settings rows |
| [Feature Patterns](android/feature-patterns.md) | One-time-unlock billing, in-house crash reporting, onboarding tour, shake gesture |
| [Spreadsheet Architecture](android/spreadsheet-architecture.md) | Apache POI `.xlsx`-as-database patterns (Dragma, Paraclete) |
| [Troubleshooting](android/troubleshooting.md) | Symptom to root cause to fix |

**Starting a new app:** read Project Setup, then check Proven Version
Pairings for a toolchain combo, then App Shell before the second screen
exists, then Theming when branding lands, then Feature Patterns once
billing/crash-reporting/onboarding comes up. Spreadsheet-first app? Read
Spreadsheet Architecture before the first tab gets written.

**Read [App Shell](android/app-shell.md) section 1 before writing any
navigation.** It's the one decision that's genuinely painful to reverse
later, and the intuitive choice is the wrong one.

---

## The rule that makes this work

> **Verify against the code before you write it down.**

Every convention here was read out of a working app, not recalled from
memory. Two separate audits found "standards" the code had already
contradicted. A five-minute grep beats a confident assumption.

That applies to the guidance itself. While writing the contrast section,
a plausible-looking claim — that `#767676` fails WCAG AA — turned out to
be wrong; it clears at 4.54:1. Checking the actual math found the real
worst case (`#7D7D7D`, at 4.16:1) and changed the recommendation.

## What makes something a standard here

1. **Verified against real code.** Read the file; don't recall it.
2. **States the reasoning, not just the rule.** A rule without a "why"
   gets cargo-culted into places it doesn't fit, then blamed when it
   fails there.
3. **Records the honest cost.** Every real trade-off has one, and a
   standard that claims to be free is hiding something.
4. **No war stories.** "App X's button broke on device Y" is abstracted
   to the underlying cause. The mechanism transfers; the anecdote doesn't.

## License

Dual-licensed: prose under
[CC BY 4.0](https://creativecommons.org/licenses/by/4.0/), code samples
under MIT. Copy the samples freely, no attribution needed. See
[`LICENSE`](LICENSE).

---

*Content originates from Kanon, a private working repository - published
here as part of the shared Tri-Tail Digital portfolio site rather than
from a separate standalone site. Corrections and disagreements are
welcome; particularly with device-tested evidence that contradicts
something here.*
