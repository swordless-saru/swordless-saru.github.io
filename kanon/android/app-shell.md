---
title: App Shell
grand_parent: Kanon
parent: Android
nav_order: 3
---

# App Shell

Scaffold placement, window insets, and navigation structure. The insets
rules here were all learned from real on-device bugs that looked correct
in a preview.

---

## 1. Scaffold placement: per-screen, not shared

**Give every screen its own `Scaffold`. Do not wrap a `NavHost` in one
shared `Scaffold`.**

This is the highest-value rule in this document. It's counter-intuitive —
the shared version looks DRYer — and getting it wrong produces insets bugs
that resist every obvious fix.

### The tempting-but-wrong shape

```kotlin
// AVOID
Scaffold(
    topBar    = { if (currentRoute in routesWithTopBar) AppTopBar() },
    bottomBar = { if (currentRoute in routesWithBottomBar) AppBottomBar() }
) { padding ->
    NavHost(...) { /* every screen */ }
}
```

It's compact, and the per-route `if` checks look like reasonable
conditional rendering. The problem is invisible: **every route now shares
one `contentWindowInsets` value and one insets-bookkeeping subtree** —
including routes that render no bars at all.

The failure mode is genuinely nasty. A screen like onboarding, with no
top or bottom bar, still has its insets mediated by that ancestor
`Scaffold`. Apply a correct, device-proven inset fix directly to the
broken button and *nothing changes*, because the ancestor is overriding
it. You'll conclude your fix was wrong. It wasn't; it was being applied at
the wrong level.

### The shape to use

```kotlin
// PREFER
NavHost(...) {
    composable(Routes.Home) { HomeScreen() }
    composable(Routes.Settings) { SettingsScreen() }
}

@Composable
fun HomeScreen() {
    Scaffold(
        topBar = { AppTopBar() },
        bottomBar = { AppBottomBar() }
    ) { padding -> /* content */ }
}
```

The bare `NavHost` has no wrapping `Scaffold`. Each screen owns its own.

**Yes, this repeats `Scaffold(...)` per screen.** That's the trade:
a little boilerplate for insets isolation that *cannot* leak between
routes. It is a good trade. DRY applies to knowledge and behavior, not to
structural scaffolding whose whole purpose is to be locally scoped.

**Share the bar composables, not the `Scaffold` call.** `AppBottomBar()`
is defined once and reused; the decision to include it is made per screen.

**Already have a shared Scaffold?** It isn't automatically a bug — it
works fine until it doesn't. Don't undertake a large refactor without a
reason, especially with no local build. Just don't let *new* screens
inherit the shape by habit.

## 2. Window insets

Edge-to-edge is mandatory on `targetSdk 35`, so insets handling is no
longer optional polish.

### Baseline: every full-screen composable

```kotlin
Modifier.fillMaxSize().systemBarsPadding()
```

### Pinned bottom buttons need more than `navigationBarsPadding()`

For a button pinned to the bottom of non-scrolling content — an
onboarding "Next", a paywall's purchase CTA:

```kotlin
Modifier.windowInsetsPadding(
    WindowInsets.navigationBars.union(WindowInsets.systemGestures)
)
```

**Why the union:** on gesture-navigation devices the OS's edge-swipe
*detection* zone is taller than the visibly-drawn navigation bar. A button
padded only for `navigationBars` clears the bar visually and still sits
inside the gesture zone — so the first tap gets eaten by a swipe-back.
The bug reads as "the button ignores my first tap," which sounds like a
Compose click-handling issue and sends you looking in entirely the wrong
place.

### ModalBottomSheet doesn't auto-inset

```kotlin
ModalBottomSheet(...) {
    Column(Modifier.navigationBarsPadding()) { /* content */ }
}
```

A full-screen `Scaffold`-hosted route handles this for you; a
`ModalBottomSheet` does not. Its action button will sit under the nav bar
unless you say otherwise.

### Nested Scaffolds: zero the inner one

```kotlin
Scaffold(contentWindowInsets = WindowInsets(0, 0, 0, 0)) { ... }
```

When a screen's `Scaffold` sits inside a structure that might also account
for system bars, zero one of them out. Otherwise both apply padding and
you get a double gap — which looks like a styling mistake rather than an
insets bug.

### Don't reach for `consumeWindowInsets` first

`consumeWindowInsets(paddingValues)` on a shared `NavHost` is the
"officially correct" answer to some insets ambiguity, but it has app-wide
blast radius. Every insets bug described above was fixed correctly by a
change scoped to the one broken composable. **Prefer the narrowest fix
that works**, and be suspicious of a fix whose failure mode is "subtly
wrong padding on unrelated screens."

## 3. Bottom navigation with a center FAB

The standard shell for an app with **2 or more primary screens**.

```kotlin
Scaffold(
    bottomBar = {
        NavigationBar {
            NavigationBarItem(...)   // left items
            Spacer(Modifier.width(72.dp))   // gap for the FAB
            NavigationBarItem(...)   // right items
        }
    },
    floatingActionButton = { AppFab(currentRoute) },
    floatingActionButtonPosition = FabPosition.Center
) { ... }
```

**Mechanics:** the FAB lives in the `Scaffold`'s own `floatingActionButton`
slot; the `NavigationBar` just leaves a literal 72dp `Spacer` for it to
float above. The FAB isn't inside the bar.

**The FAB's action is contextual, never fixed.** Its icon, action, and
`contentDescription` all change with the current route — "Add Transaction"
on one screen, "Add Account" on another. A single global action wastes the
most prominent control in the app.

**Hide the bar on non-primary routes** — settings, onboarding, detail
screens. With per-screen Scaffolds this is free: those screens simply
don't declare a `bottomBar`.

**One primary screen? Don't add this yet.** There's nothing to navigate
between, and an empty-feeling tab bar is worse than no tab bar. Add it
when the second primary screen arrives.

## 4. Navigation

- **Type-safe routes via a `Routes` object** of constants, not string
  literals scattered across call sites.
- **Screens take lambdas, not a `NavController`.** `HomeScreen(onItemClick:
  (String) -> Unit)` is previewable and testable; a screen holding a
  `NavController` is neither, and it couples your UI to the nav library.
- **Hoist navigation decisions to the `NavHost`**, where all the routing
  lives together and can be read in one place.
