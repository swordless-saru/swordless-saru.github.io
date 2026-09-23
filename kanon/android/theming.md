---
title: Theming
grand_parent: Kanon
parent: Android
nav_order: 4
---

# Theming

Color architecture, typography, iconography, and spacing.

---

## 1. Semantic palette wrapper

Expose a small `object` that names colors by **role**, aliasing into the
already-resolved `MaterialTheme.colorScheme`:

```kotlin
object AppColors {
    val card: Color        @Composable get() = MaterialTheme.colorScheme.surfaceVariant
    val muted: Color       @Composable get() = MaterialTheme.colorScheme.onSurfaceVariant
    val onSurface: Color   @Composable get() = MaterialTheme.colorScheme.onSurface
}
```

**Critical detail: alias, don't re-derive.** It is tempting to write
`if (isSystemInDarkTheme()) DarkCard else LightCard` inside this object.
Don't. Your theme already resolved light/dark once; a second
`isSystemInDarkTheme()` check duplicates that logic and will eventually
disagree with it — typically when someone adds an in-app theme override
and the palette keeps following the *system* setting.

Alias into the resolved scheme and there is exactly one source of truth
for "what mode are we in right now."

**Why bother at all?** `AppColors.card` states intent;
`MaterialTheme.colorScheme.surfaceVariant` states a mechanism. When you
later decide cards should use `surface` instead, you change one line
rather than auditing every call site to work out which `surfaceVariant`
usages meant "card."

## 2. Contrast is a hard requirement, not a nicety

**Any feature where a color is chosen at runtime** — user accent colors,
category colors, data-driven tints — **must compute its foreground color
against real WCAG contrast math.** Never hardcode `Color.White` and assume
it reads.

The math is short enough to own outright — relative luminance, a contrast
ratio, and a "pick the better foreground" helper:

```kotlin
/** WCAG AA: 4.5 for normal text, 3.0 for large (18pt+, or 14pt+ bold). */
const val AA_NORMAL = 4.5
const val AA_LARGE = 3.0

/** Contrast ratio in 1.0..21.0. Order-independent. */
fun contrastRatio(a: Color, b: Color): Double {
    val la = relativeLuminance(a)
    val lb = relativeLuminance(b)
    return (maxOf(la, lb) + 0.05) / (minOf(la, lb) + 0.05)
}

private fun relativeLuminance(color: Color): Double {
    // The 0.03928 branch and 2.4 exponent linearize sRGB's gamma curve;
    // the weights reflect eye sensitivity. Straight from the spec -
    // don't "simplify" these constants.
    fun channel(c: Float): Double {
        val v = c.toDouble()
        return if (v <= 0.03928) v / 12.92 else ((v + 0.055) / 1.055).pow(2.4)
    }
    return 0.2126 * channel(color.red) +
        0.7152 * channel(color.green) +
        0.0722 * channel(color.blue)
}

/** Whichever candidate contrasts better against [background]. */
fun bestOnColor(background: Color, vararg candidates: Color): Color =
    candidates.maxBy { contrastRatio(background, it) }
```

**Keep this math out of your composables.** Written against plain RGB
components rather than a Compose `Color`, it's pure logic that unit-tests
on the JVM in milliseconds with no emulator. Tying it to a Compose type
makes genuinely platform-independent code artificially Android-only.

```kotlin
val onAccent = ColorContrast.bestOnColor(accentColor.toRgb())
```

**Thresholds:** WCAG 2.2 AA requires **4.5:1** for normal text and
**3.0:1** for large text (18pt+, or 14pt+ bold). Both are constants in the
snippet.

**Honest limitation:** `bestOnColor` returns the *better* of two
candidates, which is not the same as guaranteeing compliance.

Concretely: **`#7D7D7D` is the worst-case neutral gray** — the exact point
where the white and near-black contrast curves cross — and its best
available foreground still only reaches **4.16:1**, short of the 4.5
threshold. There is no legible text color for that background. Every gray
from roughly `#7A7A7A` to `#808080` has the same problem.

That's the argument for curated swatches below: if users can pick
arbitrary colors, some of their choices are simply unfixable, and no
amount of runtime math rescues them. Either constrain the input or show
the user the actual ratio.

## 3. Accent color overrides: curate, don't free-form

If you offer accent customization, offer **a curated set of preset
swatches** — not a free hex field.

**Why:** every preset can be contrast-verified once, by a unit test that
asserts the whole set clears AA. Free hex entry makes that impossible —
and as shown above, some colors have *no* compliant foreground at all, so
runtime math cannot rescue a choice the user shouldn't have been allowed
to make. A determined user will find exactly the gray that looks broken.

A live contrast-preview widget is a reasonable *mitigation* for free-form
entry, but it's a fair amount of UI to build in service of a feature few
users want. Curated swatches solve the same problem by construction. Add
free entry only if users actually ask.

**Label swatches with names, not bare hex.** An unlabeled color swatch is
invisible to a screen reader, and `#3D78CE` means nothing to anyone.

## 4. Typography

Define the full scale explicitly, even where a value matches the Material
default — omission reads as an oversight, while an explicit value reads as
a decision.

| Style | Size | Typical role |
|---|---|---|
| `displayLarge` | 48sp | One hero number per app (a balance, a total) |
| `headlineLarge` | 24sp | Screen titles |
| `titleLarge` | 22sp | Section headers, top-app-bar titles |
| `bodyLarge` | 16sp | Primary body text |
| `bodyMedium` | 14sp | Secondary body text |
| `labelMedium` | 12sp | Buttons, chips |
| `labelSmall` | 11sp | Timestamps, captions |

**Don't invent a UI role to justify a style.** If an app has no hero
number, it has no `displayLarge` usage — that's fine. Define it for
consistency; don't manufacture a giant number to use it.

**Never go below 11sp**, and remember system font scaling can multiply
these substantially. Use `sp` for text always, `dp` for everything else.

## 5. Iconography

**Material Icons Extended is the default.** Nav icons, buttons, list rows,
status indicators — all of it. You get consistency, accessibility, and
full coverage at no cost.

**Reserve custom icons for brand marks** and genuinely novel concepts
Material has no equivalent for. Custom icons for generic actions is a
maintenance burden with no upside.

**Never use emoji in shipped UI.** Not in buttons, list rows, empty
states, chips, or badges. Emoji render differently per platform and
vendor, don't respond to theme colors, and are announced unpredictably by
screen readers. (Emoji in internal markdown docs is a separate, harmless
matter.)

**Brand mark vectors — convert carefully:**
- Copy `pathData` **verbatim** from the source SVG. Never hand-edit or
  re-derive a coordinate string.
- If the SVG has a `<g transform>`, apply the **full affine** transform
  (`x' = a*x + c*y + e`, `y' = b*x + d*y + f`), not just the translation.
  A partial port produces plausible-looking numbers that are subtly wrong
  and manifest as "the logo still looks oddly padded" — no build error.
- **Verify with a bounding-box measurement**, not by eyeballing the render.
- Re-read a designer-supplied asset's *contents* before assuming it's
  unchanged. A matching filename and timestamp prove nothing.

**Launcher icons:** adaptive icon XML (`mipmap-anydpi-v26/ic_launcher.xml`
plus `ic_launcher_round.xml`). Non-negotiable on API 26+.

## 6. Spacing and sizing

No `Dimens`/`Spacing` token object is required at small scale — inline
`.dp` literals are fine. What matters is consistency of the numbers.

| Element | Value |
|---|---|
| Primary CTA button height | `52.dp` |
| In-app brand mark (onboarding, empty states) | `120.dp` |
| Screen edge padding | `16.dp`–`24.dp` horizontal |
| Bottom button block padding | `32.dp` horizontal, `24.dp` vertical |
| Top bar row padding | `16.dp` horizontal, `8.dp` vertical |
| Settings row icon avatar | `40.dp` circle, `20.dp` icon inside |
| Page indicator dot | `8.dp` tall; `24.dp` wide selected, `8.dp` unselected; `300ms` tween |
| Corner radii | Material 3 defaults — don't override without reason |

**Minimum touch target is 48x48dp**, regardless of the icon's visual size.
A 20dp icon still needs a 48dp tappable area. This is a WCAG 2.2 AA
requirement (SC 2.5.8), not a suggestion.

## 7. Settings information architecture

Before writing a single row composable, decide the *shape* of Settings —
the screen's information architecture matters more than its pixels, and
is harder to fix once screens and routes exist around it.

### Grouped list is the standard for a reason

Section headers + rows (icon/title/subtitle, chevron or switch) is what
iOS Settings, Android Settings, and nearly every mobile app uses. Don't
invent a different top-level shape for Settings specifically — it's the
one screen where users already have the strongest mental model.

### A near-universal starting taxonomy

| Category | Typical content |
|---|---|
| Account / Data | Whatever the app treats as its core connected resource — an account, a data source, a workspace |
| Preferences / Appearance | Theme, accent color, display options |
| Notifications | Reminders, alerts, quiet hours |
| Privacy & Security | Lock, biometrics, permissions |
| Data & Storage | Backup/restore, export, storage usage |
| About / Support | Version, help, licenses, upgrade/Pro |

Treat this as a starting point, not a checklist to fill in. An app with
no login has no "Account" category to build — don't manufacture one.
Rename freely: **"Appearance" reads better than either "Custom Theme" or
"Accent Color"** once an app has more than one visual setting, and
picking the more general name up front avoids a rename later when the
section grows to cover dark-mode overrides or font scale.

### Dedicated screen vs. inline section

Promote a category to its own screen (own route, own back button) only
when it has real complexity: a list you can add to / remove from /
rename, a picker grid with genuine breadth, or anything likely to grow.
Keep it **inline** — a few rows or a single picker grid rendered right in
the main list — when it's genuinely small, even if it *feels* like it
deserves more ceremony. A 5-swatch color picker is inline content; a
multi-item list users actively manage is not.

**Don't decide this once and treat it as final.** A concrete lesson from
production: a color-preset picker was initially promoted to its own
dedicated screen, then reverted to inline on the very next revision once
it was actually on-device — a full screen for five swatches turned out to
be more ceremony than the content warranted. Confirm dedicated-vs-inline
against the *actual* content each time, not by matching whatever a
previous plan or a similar app already did.

### Entry rows should preview their own content

A dedicated screen's entry row in the main list should show a compact
live summary value — "Profiles → 2 profiles", "Backups → 3 saved" — not a
bare label. It costs one extra bound value and turns a plain menu item
into something that answers a question before the screen even opens.

### The back-button chrome is app-specific, the shape isn't

Whether a dedicated Settings sub-screen supplies its own `Scaffold`/
`TopAppBar`, or inherits one from a shared navigation shell (see the App
Shell doc), is a detail of that app's own shell — not part of this
pattern. The portable rule is just "a dedicated screen has a back
button"; confirm which mechanism a given app actually uses before
porting this detail, the two aren't interchangeable.

### Sequencing and destructive actions

- Put whatever answers "what context/data am I looking at right now"
  first, ahead of every other setting, if the app has that kind of state.
- Isolate destructive actions (disconnect, delete, reset) at the very
  bottom, visually distinct (a warning tint), and always behind a
  confirmation — never a single tap.
- Skip a settings search bar until the list is genuinely long (tens of
  entries). Adding search to a dozen-section screen solves a problem that
  doesn't exist yet — YAGNI.

### Don't copy another app's screen inventory wholesale

Even across two apps sharing this taxonomy, matching treatment isn't
automatic. Confirm dedicated-vs-inline, and which categories even apply,
against each app's own content — not by mirroring whatever a reference
implementation happened to choose.

## 8. Settings row styling

A consistent, reusable settings row shape:

```kotlin
@Composable
fun SettingsItem(
    icon: ImageVector,
    title: String,
    subtitle: String,
    iconTint: Color = AppColors.muted,
    onClick: (() -> Unit)? = null,
    trailingContent: (@Composable () -> Unit)? = null
) {
    Row(
        modifier = Modifier
            .fillMaxWidth()
            .then(if (onClick != null) Modifier.clickable(onClick = onClick) else Modifier)
            .padding(horizontal = 16.dp, vertical = 14.dp),
        verticalAlignment = Alignment.CenterVertically,
        horizontalArrangement = Arrangement.spacedBy(16.dp)
    ) {
        Box(
            modifier = Modifier
                .size(40.dp)
                .clip(CircleShape)
                .background(iconTint.copy(alpha = 0.1f)),
            contentAlignment = Alignment.Center
        ) {
            Icon(icon, contentDescription = null, Modifier.size(20.dp), tint = iconTint)
        }
        Column(Modifier.weight(1f)) {
            Text(title, style = MaterialTheme.typography.bodyLarge, fontWeight = FontWeight.Medium)
            Text(subtitle, style = MaterialTheme.typography.bodyMedium, color = AppColors.muted)
        }
        trailingContent?.invoke()
    }
}
```

**Key details:** card-grouped sections (a `Surface` with the semantic
`card` color, not a bare `Column`); a 40dp circular icon avatar tinted at
10% alpha of the icon's own color; two-line title/subtitle; an optional
`trailingContent` slot for switches or chevrons.

`contentDescription = null` on the icon is deliberate — the adjacent text
already conveys the meaning, and announcing both is redundant noise for
screen-reader users.

## 9. Dynamic color (Material You)

Wire it as an **opt-in parameter defaulted to off**:

```kotlin
@Composable
fun AppTheme(
    darkTheme: Boolean = isSystemInDarkTheme(),
    dynamicColor: Boolean = false,
    content: @Composable () -> Unit
)
```

Dynamic color derives the palette from the user's wallpaper, which
overrides your brand identity entirely. For a branded app that's usually
the wrong default. Having the parameter present costs nothing and makes
it a one-line change if you want it later.
