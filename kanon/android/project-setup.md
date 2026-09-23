---
title: Project Setup
grand_parent: Kanon
parent: Android
nav_order: 1
---

# Project Setup

Starting a new Android app: baseline versions, `MainActivity` shape, and
the development workflow.

---

## 1. Version baseline

Pin these together. The exact numbers age; the *policy* is the durable
part. **For AGP/Gradle/Kotlin/KSP/Hilt specifically, check
[Proven Version Pairings](version-pairings.md) first** - those five move
together and fast enough that a table here would just go stale; that
page is the maintained source and explains *why* each combo is safe, not
just what it is.

| Setting | Value | Notes |
|---|---|---|
| `compileSdk` / `targetSdk` | 35 | Android 15 |
| `minSdk` | 26 | Android 8.0. Covers ~95% of devices and avoids a long tail of pre-Oreo workarounds |
| `jvmTarget` | 17 | Stable across every AGP/Kotlin bump so far - unlike the fast-moving group above, this one rarely needs to change |
| Compose BOM | `2024.06.00` | **Use the BOM.** Never pin individual Compose artifact versions |
| `navigation-compose` | `2.7.6` | Not covered by the BOM |
| `lifecycle-*` | `2.7.0` | Not covered by the BOM |
| `kotlinx-coroutines-android` | `1.7.3` | |

**Version policy:**

- **Bump the Compose BOM as a unit, never piecemeal.** The BOM exists
  precisely to guarantee a mutually-compatible set. Overriding one
  artifact inside a BOM-managed set is how you get a `NoSuchMethodError`
  at runtime rather than a compile error.
- **Keep every app in a related family on identical AGP/Gradle/Kotlin/
  KSP/Hilt versions, full stop - not "where practical."** When they
  drift, a fix proven in one app stops being evidence for another, and
  every bump has to re-derive a compatibility matrix that's already been
  solved once. This is the whole reason
  [Proven Version Pairings](version-pairings.md) exists: record a combo
  once, reuse it everywhere it applies, including in unrelated client
  work if it turns out to be genuinely reusable there too.
- **Targeting SDK 35 is not optional for Play Store releases.** Google
  enforces a target-API floor that rises annually; treat the bump as
  routine maintenance rather than a project.

## 2. MainActivity

```kotlin
@AndroidEntryPoint
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        enableEdgeToEdge()
        setContent {
            AppTheme {
                Surface(
                    modifier = Modifier.fillMaxSize(),
                    color = MaterialTheme.colorScheme.background
                ) {
                    AppNavHost()
                }
            }
        }
    }
}
```

**Call `enableEdgeToEdge()` explicitly**, even though `targetSdk 35`
forces edge-to-edge at the OS level regardless of what your code does.
Two reasons: it documents the intent rather than relying on invisible OS
behavior, and it's what sets correct status/nav bar icon contrast. Relying
on the implicit version means the day you lower `targetSdk` for an
unrelated reason, your app silently changes layout.

**Base class:** `ComponentActivity` by default. Use `FragmentActivity`
**only** if you actually ship `androidx.biometric.BiometricPrompt`, which
requires it. Don't widen the base class pre-emptively — it's easy to
change later, and `ComponentActivity` is the leaner default.

**One activity, many composables.** Multiple activities buy you nothing
in a Compose app and cost you navigation complexity, transition control,
and shared state.

## 3. Dependency injection

Hilt, with `@AndroidEntryPoint` on `MainActivity`. Standard, well-documented,
and its compile-time validation catches wiring mistakes at build time
rather than as a runtime crash on a user's device.

Koin is a defensible alternative in Kotlin Multiplatform projects, where
Hilt's Android-only annotation processing doesn't fit. Outside that case,
prefer Hilt — the compile-time safety is worth more than the slightly
lighter setup.

## 4. Package structure

```
com.example.app/
  data/
    local/        # Room DAOs, entities, DataStore
    remote/       # API services
    repository/   # Repository implementations
    util/         # Pure helpers, no Android imports
  domain/         # Use cases, domain models (optional for small apps)
  ui/
    screens/      # One package per screen: Screen + ViewModel together
    components/   # Shared composables
    theme/        # Color.kt, Theme.kt, Type.kt
  di/             # Hilt modules
  FeatureFlags.kt # See below
```

**Group by feature, not by type,** inside `ui/screens/`. Keeping
`SettingsScreen.kt` and `SettingsViewModel.kt` adjacent beats scattering
them across parallel `screens/` and `viewmodels/` trees — you edit them
together, so store them together.

**`data/util/` must have no Android imports.** Pure logic there is
unit-testable without instrumentation, which is the difference between
tests that run in CI in seconds and tests that need an emulator.

## 5. Trunk-based development with feature flags

**Build on `main` in small increments. No long-lived feature branches.**

Gate anything a user could stumble into — new screens, settings rows,
buttons, notification actions — behind a compile-time boolean:

```kotlin
/**
 * Compile-time feature flags. A flag lives here from the moment a feature
 * starts until the moment it ships, then gets deleted along with its
 * branches - a flag that's been `true` for six months is dead code
 * pretending to be a configuration option.
 */
object FeatureFlags {
    const val SOME_FEATURE_LIVE = false
}
```

**Why:** long-lived branches drift from `main` and create a risky merge
day. Worse, fixes landing on `main` aren't tested against in-progress
feature code until that merge. Shipping code dark — present in the build,
invisible to users — avoids both. This is what trunk-based development
means in practice, and no remote config service is warranted at
single-developer scale.

**Keep paid/Pro gating separate** from `FeatureFlags`. Billing state is a
runtime concern driven by a purchase; a feature flag is a compile-time
constant. Conflating them makes "is this hidden because it's unfinished,
or because they haven't paid?" unanswerable.

**Delete flags after launch.** The flag and its `false` branch both go.
This is the step everyone skips, and it's how a codebase accumulates
permanently-dead alternate paths.

## 6. Verification workflow

Assume **no local JDK or Gradle**, with GitHub Actions as the only build
verification. This constraint is more common than it sounds — a locked-down
work machine, a Chromebook, a tablet — and it enforces good habits:

- **Keep changes small and diff-reviewable.** A change you can't verify
  locally is a change you must be able to *read* correctly.
- **Prefer precedent over first-principles reasoning** when a fix is
  ambiguous. Copying a fix already proven on-device beats deriving a
  cleverer one you can't test.
- **State verification status honestly.** "Compiles in CI, not yet
  device-tested" is useful. Implying something was tested when it wasn't
  is how a regression reaches a user.
- **Unit-test the pure logic.** With no emulator available, pure-JVM tests
  are the only automated correctness signal you have. This is a strong
  argument for keeping logic out of composables.

## 7. Git hygiene

- **`git remote -v` and `git branch -vv` before pushing in any repo that
  has ever been renamed.** A stale tracking ref sends a push to the old
  repo and reports success.
- **A chained `git commit && git push` can report success while the push
  silently doesn't land.** For anything that matters, `git fetch` before
  and after as separate verified steps.
- **`git fetch` and diff before any full-file rewrite**, not just at the
  start of a work session. A plan made an hour ago can be stale, and a
  blind overwrite has no way to detect a concurrent change.
- **An archived GitHub repo 403s on push** while `git commit` succeeds
  locally. If a repo is superseded, move the work rather than fighting
  the archive flag.
