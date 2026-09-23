---
title: Proven Version Pairings
grand_parent: Kanon
parent: Android
nav_order: 2
---

# Proven Version Pairings

A version combo (AGP + Gradle + Kotlin + KSP + Compose compiler + Hilt)
only needs to be worked out **once**. Deriving a safe pairing from
scratch means reading release notes for every piece and usually still
getting burned by an undocumented interaction between two of them - a
recent AGP bump across two sibling Compose+Hilt apps took **four separate
wrong-guess-then-CI-fix rounds** before landing on a working combo, each
one a real compile failure, not a hypothetical.

**Once a combo is confirmed working - CI green, or device-verified - it
goes in the table below, permanently, regardless of which project proved
it.** That includes client work outside any personal app portfolio. The
next time any project needs a similar bump, check here first. If
something close enough is already listed, copy it and skip the
research entirely. If nothing matches, do the verification work once,
then add the row - the whole point is that nobody (including a future
you) should have to solve the same compatibility puzzle twice.

---

## How to use this table

1. **Before bumping any app's Android toolchain, check here first** for
   an already-proven combo at or near the target versions.
2. **If nothing matches closely enough, verify it properly** (real CI
   run at minimum, device/Studio sync if available), then add a row -
   don't skip recording it just because it was "just a small bump."
3. **This is an append-only history, not just "latest."** An older row
   can still be the right answer for a project stuck below some
   constraint the newest row doesn't have (see the AGP 8.13 row below).
4. **A pairing proven in one project is evidence, not proof, for
   another.** Different dependencies (Room, WorkManager, a specific
   Compose library) can still introduce a new wrinkle. Treat a matching
   row as a strong starting point that skips the *research*, not as a
   guarantee that skips *verification* entirely.

## Table

| Proven | AGP | Gradle | Kotlin | KSP | Compose compiler plugin | Hilt | Verified in | Notes |
|---|---|---|---|---|---|---|---|---|
| 2026-09-11 | `9.4.0` | `9.7.1` | `2.3.20` (built-in Kotlin - see note) | `2.3.11` | `2.3.20` | `2.60.1` | Two sibling Compose+Hilt apps (CI: unit tests + assembleDebug, both green) | **Current recommended baseline for a new AGP-9-era Compose+Hilt+KSP app.** AGP 9 ships "built-in Kotlin": do NOT apply `org.jetbrains.kotlin.android` - declare the Kotlin version instead via `buildscript { dependencies { classpath("org.jetbrains.kotlin:kotlin-gradle-plugin:2.3.20") } }` at the top of the root `build.gradle.kts`. The Compose compiler plugin is still applied separately and still needs its version matched to Kotlin - built-in Kotlin does NOT manage it automatically. Confirmed via grep of both apps before migrating: this combo assumes no use of the old Variant API (`applicationVariants`, `variantFilter`, `BaseExtension` casts), `kotlin-kapt`, or `packagingOptions`/`lintOptions`/`aaptOptions` - if a project uses any of those, expect extra migration work AGP's release notes are the not-yet-verified source for. |
| 2026-09-08 | `8.13.2` | `8.13` | `2.3.20` | `2.3.11` | `2.3.20` | `2.58` | Two sibling Compose+Hilt apps (CI: unit tests + assembleDebug, both green) | Superseded by the row above but kept - the correct answer for any project stuck below AGP 9 for some real reason. AGP 8.13.x's plugin code calls a Gradle-internal API (`org.gradle.api.problems.internal.InternalProblems`) that was **removed in Gradle 9.6.0** - confirmed by an actual CI failure. Google's own AGP 8.13 docs list "Default Gradle: 8.13," which is what's used here, not a higher number that happens to also be available. Hilt is capped at `2.58` specifically because Dagger/Hilt 2.59 raised the Hilt Gradle plugin's own minimum AGP requirement to 9.0.0 - also confirmed by a real CI failure, not a guess. |
| 2026-09-07 | n/a - pure JVM, no AGP dependency | `9.7.1` | `2.4.20` | n/a | n/a | n/a | Kanon (`snippets/jvm`) | A pure-JVM Kotlin module has no AGP-driven ceiling and can run ahead of whatever the Android-app baseline is - don't assume a JVM module and an Android app in the same portfolio need matching Gradle/Kotlin versions just because they're related projects. Gradle 9+ needs `testRuntimeOnly("org.junit.platform:junit-platform-launcher")` declared explicitly - it's no longer added to the test runtime classpath automatically, and skipping this produces a confusing "Failed to load JUnit Platform" failure at `:test` even though compilation succeeds. |

## Why lockstep pays off for a family of similar apps

If you maintain more than one app that share the same rough shape
(same DI framework, same UI toolkit, similar dependency set), keeping
them on identical toolchain versions is worth the minor coordination
cost:

- **A fix proven in one app is genuine evidence for the others**, not
  just a hint - port it and expect it to work first try, the way the
  2026-09-11 AGP 9 bump above went green on the *first CI attempt* for
  both apps it touched, once the compatibility matrix was already known
  from the first app.
- **Letting them drift means solving the same puzzle twice** - a
  version combo that's already painstakingly verified in App A provides
  zero shortcut for App B if App B is sitting on different versions to
  begin with.
- **New apps in the family should start from whatever's in the table
  above**, not from whatever tutorial or template was most recent when
  that app was scaffolded.

This does **not** mean every project everywhere should match. A client
project has its own constraints, timeline, and existing dependencies -
it should be pinned to whatever's right for *that* engagement, verified
independently, and only then contributed back to this table if it
turns out to be a genuinely reusable combo.
