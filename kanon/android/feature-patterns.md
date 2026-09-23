---
title: Feature Patterns
grand_parent: Kanon
parent: Android
nav_order: 5
---

# Feature Patterns

Three features nearly every consumer app eventually needs, each with a
real, working shape below rather than a description of one: one-time-unlock
billing, in-house crash reporting, and a first-run onboarding tour.

---

## 1. One-time-unlock billing (Play Billing)

### Prefer one non-consumable purchase over a subscription or SKU matrix

If the app's whole value proposition doesn't depend on recurring revenue,
**one non-consumable "Pro" unlock that covers every premium feature —
current and future — for a single price** is simpler to build, simpler to
explain to a user, and avoids subscription-management UI entirely. Resist
splitting Pro features into separate SKUs "in case someone only wants one
of them" until real demand says otherwise — YAGNI.

### The manager: entitlement as a `StateFlow`, with a debug override layered on top

```kotlin
@Singleton
class BillingManager @Inject constructor(
    @ApplicationContext private val context: Context,
    private val billingPreferences: BillingPreferences
) : PurchasesUpdatedListener {

    private val _realIsPro = MutableStateFlow(billingPreferences.isProCached())
    private val _debugProOverride = MutableStateFlow(
        if (BuildInfo.isDebug(context)) billingPreferences.getDebugProOverride() else null
    )

    /** True entitlement, with an optional debug-only override layered on top. */
    val isPro: StateFlow<Boolean> = combine(_realIsPro, _debugProOverride) { real, override ->
        override ?: real
    }.stateIn(scope, SharingStarted.Eagerly, _debugProOverride.value ?: _realIsPro.value)
}
```

**Why a debug override matters:** it lets Pro features be tested on a real
device before any internal-testing track exists, without touching a real
payment method. Gate it so it can only ever be non-null on a debuggable
build — check this in the manager itself (`if (!BuildInfo.isDebug(context))
return` inside the setter), not at the UI call site. That way there is no
code path, however indirect, that lets it affect a release build.

### Lifecycle: connect once, sync twice

```kotlin
fun initialize() {
    billingClient.startConnection(object : BillingClientStateListener {
        override fun onBillingSetupFinished(billingResult: BillingResult) {
            if (billingResult.responseCode == BillingClient.BillingResponseCode.OK) {
                scope.launch {
                    queryProDetails()   // for price display
                    syncPurchases()     // for current entitlement
                }
            }
        }
        override fun onBillingServiceDisconnected() {
            // Play retries via the next initialize() call (e.g. next
            // foreground) - no reconnect logic needed here.
        }
    })
}
```

Call `initialize()` once at app start. Query product details (for the
price shown on the paywall) and query existing purchases (to restore
entitlement on a reinstall or new device) as two separate calls after
connection succeeds — conflating them makes failures harder to diagnose.

### Acknowledge every purchase — Play auto-refunds unacknowledged ones

```kotlin
private suspend fun acknowledgeIfNeeded(purchase: Purchase) {
    if (purchase.purchaseState != Purchase.PurchaseState.PURCHASED || purchase.isAcknowledged) return
    val params = AcknowledgePurchaseParams.newBuilder()
        .setPurchaseToken(purchase.purchaseToken)
        .build()
    billingClient.acknowledgePurchase(params)
}
```

This is not optional cleanup. **Play Billing automatically refunds any
purchase that isn't acknowledged within three days.** Call this from both
`onPurchasesUpdated` (a fresh purchase) and `syncPurchases` (a purchase
made on another session/device) — a purchase can reach the app through
either path.

### Gate the entry point until a Pro feature actually exists

```kotlin
companion object {
    const val PRODUCT_ID_PRO = "your_app_pro"

    /**
     * Whether any Pro feature actually exists yet. The upgrade entry
     * point and paywall are hidden entirely while this is false -
     * letting someone pay for a purchase that unlocks nothing is a
     * refund magnet and a store-policy problem.
     */
    const val PRO_FEATURES_LIVE = false
}
```

**Concrete lesson, found by a pre-launch audit:** a paywall advertised
four premium features before any of them existed in code. The fix is this
one boolean, checked wherever the upgrade entry point renders. Existing
purchasers should always see their "thanks, you're Pro" state regardless
of this flag — it only ever gates the entry point for people who haven't
bought yet.

### The paywall UI: one feature list, one source of truth

```kotlin
private val PRO_FEATURES = listOf(
    "Feature one",
    "Feature two",
    "All future Pro features - forever, no extra purchase",
)

@Composable
fun PaywallSheet(priceText: String?, onPurchaseClick: () -> Unit, onDismiss: () -> Unit) {
    ModalBottomSheet(onDismissRequest = onDismiss) {
        // icon, title, "one-time purchase, no subscription" subtitle,
        // PRO_FEATURES rendered as a checklist, then:
        Button(onClick = onPurchaseClick, enabled = priceText != null) {
            Text(if (priceText != null) "Unlock Pro - $priceText" else "Loading...")
        }
    }
}
```

Keep `PRO_FEATURES` as the **one** list that has to be updated when a Pro
feature ships — not a comment, not a second copy in a store listing draft
that quietly drifts out of sync.

**Pin the purchase button's insets to the gesture-navigation union**, same
fix as any pinned bottom CTA — see [App Shell §2](app-shell.md#2-window-insets)
rather than re-deriving it here.

---

## 2. In-house crash reporting (no third-party SDK)

### When to reach for this instead of Crashlytics/Sentry

This pattern trades **aggregate, cross-device crash analytics** for **zero
third-party data collection and zero server cost.** It's the right choice
when an app's positioning already promises "your data never leaves this
device," or for a small app where a handful of self-reported crashes is
plenty of signal. It is the *wrong* choice for an app that needs to know
its crash-free rate across thousands of installs without asking each user
individually — that's exactly the problem Crashlytics/Sentry solve. Choose
deliberately; this isn't a strictly-better replacement.

### Install once, chain to whatever handler already existed

```kotlin
object CrashReporter {
    fun install(context: Context) {
        val appContext = context.applicationContext
        val previousHandler = Thread.getDefaultUncaughtExceptionHandler()
        Thread.setDefaultUncaughtExceptionHandler { thread, throwable ->
            try {
                writeReport(appContext, throwable)
            } catch (e: Exception) {
                // Never let the crash handler itself crash - worst case
                // we just lose this one report.
                Log.e("CrashReporter", "Failed to write crash report", e)
            }
            // Hand off so the crash still surfaces normally - "app has
            // stopped," process death, everything already expected.
            previousHandler?.uncaughtException(thread, throwable)
                ?: Runtime.getRuntime().exit(1)
        }
    }
}
```

Call `install()` once from `Application.onCreate`, as early as possible.
**Never replace the previous handler outright** — chain to it. Swallowing
the OS's own crash behavior is a worse bug than the one being reported.

### Write locally, only leave the device on explicit user action

```kotlin
private fun writeReport(context: Context, throwable: Throwable) {
    val report = buildString {
        appendLine("Time: ${timestamp()}")
        appendLine("App version: ${AppVersion.name(context)} (${AppVersion.code(context)})")
        appendLine("Device: ${Build.MANUFACTURER} ${Build.MODEL}")
        appendLine("Android: ${Build.VERSION.RELEASE} (SDK ${Build.VERSION.SDK_INT})")
        appendLine()
        appendLine(Log.getStackTraceString(throwable))
    }
    reportFile(context).writeText(report)
}

fun buildEmailIntent(reportText: String): Intent = Intent(Intent.ACTION_SENDTO).apply {
    data = Uri.parse("mailto:")
    putExtra(Intent.EXTRA_EMAIL, arrayOf(DEV_INBOX))
    putExtra(Intent.EXTRA_SUBJECT, "Crash report")
    putExtra(Intent.EXTRA_TEXT, reportText)
}
```

**No network permission, no automatic upload.** The report is written to
the app's own private files dir and sits there until the user is shown a
one-time prompt on the *next* launch and explicitly taps "Send." Use
`ACTION_SENDTO`, not a direct network call — the user's own email client
handles the send, so they can read or edit the report first, and the app
never needs email or network permissions of its own. Delete the pending
report file once the prompt has been shown and answered, sent or
dismissed, so a single crash never asks twice.

### Give yourself a way to test the whole pipe without a real bug

```kotlin
/** Debug-only. Gate the *caller* (e.g. a Settings button) behind BuildInfo.isDebug. */
fun forceTestCrash(): Nothing =
    throw RuntimeException("Test crash - CrashReporter.forceTestCrash()")
```

Keep this function itself free of any debug/release branching — gate the
call site instead. That way "remove the test hook" is a one-line deletion
of a Settings row, not a hunt for scattered `if (BuildInfo.isDebug)` checks.

---

## 3. First-run onboarding tour

### The shape: `HorizontalPager` + animated dot indicator

```kotlin
val pagerState = rememberPagerState(pageCount = { pages.size })
val isLastPage = pagerState.currentPage == pages.size - 1

HorizontalPager(state = pagerState, modifier = Modifier.weight(1f)) { page -> pages[page]() }

repeat(pages.size) { index ->
    val isSelected = pagerState.currentPage == index
    val width by animateDpAsState(if (isSelected) 24.dp else 8.dp, tween(300))
    val color by animateColorAsState(
        if (isSelected) MaterialTheme.colorScheme.primary
        else MaterialTheme.colorScheme.onSurface.copy(alpha = 0.2f),
        tween(300)
    )
    Box(Modifier.padding(horizontal = 3.dp).height(8.dp).width(width).clip(CircleShape).background(color))
}
```

A widening, color-animating dot per page is a cheap, standard progress
indicator — animate both width and color so the current page is legible
even to someone not distinguishing the color difference alone.

### Every page but the last needs an escape hatch

Put a "skip" / "I'll explore on my own" `TextButton` in the top corner on
every page except the last — forcing a full read-through of a tour before
reaching the app is a common source of user irritation. On the last page,
replace it with the real call-to-action ("Get Started").

### Treat "replay" as a distinct mode, not the same flow reused blindly

```kotlin
@Composable
fun OnboardingScreen(onComplete: () -> Unit, isReplay: Boolean = false) {
    // isReplay changes only the top-right button's label/visibility:
    // "Skip" (visible even on the last page) instead of "I'll explore on
    // my own" (hidden on the last page, where the real CTA takes over).
}
```

A returning user replaying the tour from Settings already knows the app —
don't make them scroll past a persuasive CTA to back out again, and don't
let replaying touch the same "has completed onboarding" flag the real
first run uses. Give replay its **own nav route** so it can't be confused
with, or accidentally reset, real first-run state.

### The completion flag is its own tiny class

```kotlin
@Singleton
class OnboardingPreferences @Inject constructor(@ApplicationContext context: Context) {
    private val prefs = context.getSharedPreferences("app_onboarding_prefs", Context.MODE_PRIVATE)
    fun isOnboardingCompleted(): Boolean = prefs.getBoolean(KEY_COMPLETED, false)
    fun setOnboardingCompleted(completed: Boolean) = prefs.edit().putBoolean(KEY_COMPLETED, completed).apply()
}
```

Device-local, single boolean, single responsibility. Don't bolt this onto
a preferences class scoped to something else (a data-connection or
workspace class, say) just because it's small — a different concern
deserves its own class even at one field.

### Conditional pages are fine as a plain list build

If one page offers "new to this kind of app? see two extra primer pages"
as a checkbox, keep the page list itself dead simple:

```kotlin
val pages = remember(includeBasics) {
    buildList<@Composable () -> Unit> {
        add { WelcomePage(includeBasics, onToggle) }
        if (includeBasics) { add { PrimerPageOne() }; add { PrimerPageTwo() } }
        add { CoreFeaturePageA() }
        // ...
        add { LetsGoPage() }
    }
}
```

A plain conditional `buildList` reads clearly; resist reaching for a more
"flexible" page-graph abstraction for what is, in practice, a short linear
sequence with one optional branch.

### Don't duplicate a flow that already exists elsewhere

If the app already has an empty-state screen that walks a user through
"connect your data" or an equivalent first-action prompt, the onboarding
tour's last page can simply call `onComplete()` and hand off to it — it
doesn't need its own copy of that guidance. Building a second, onboarding
-only version of the same flow is duplicated maintenance for no benefit —
DRY applies to user-facing flows, not just code.

**Reuse the same gesture-navigation inset fix** on the pinned bottom
button/page-indicator block as any other pinned bottom CTA — see
[App Shell §2](app-shell.md#2-window-insets).

---

## 4. Shake gesture as a secondary, optional trigger

A shake-to-reveal gesture (shake the device to surface a random tip,
easter egg, or piece of hidden content) is a fun, low-cost feature to add
— but it comes with two failure modes worth designing around up front
rather than discovering later.

### Never let it be the only way in

**A shake gesture is unavailable to some users** — anyone who can't
physically shake the device, or has it mounted/docked, or simply never
discovers the gesture exists. Whatever the shake reveals must already be
reachable another way (a tap, a menu entry) before the shake trigger is
added on top. The shake is a shortcut, never a gate.

### False positives are the real design problem, not detection accuracy

Phones get shaken in pockets, on runs, in cars, being set down harder
than intended. The fix is a **threshold plus a debounce**, not a lower
sensitivity: require a g-force magnitude clearly above ordinary handling
(strong enough to filter out walking and pocket jostling), and refuse to
fire again within a short cooldown window, since a single real shake
crosses the threshold on many consecutive sensor readings in a row.

The math for both is pure logic with no Android dependency, so it's a
real, compiled, unit-tested snippet - `kanon.sensor.ShakeMath` in
`snippets/jvm/` (`gForce`, `isShake`, `shouldEmit`, verified against the
real `SensorManager.GRAVITY_EARTH` physics constant, not just checked for
self-consistency).

### The `SensorManager` wiring, as reference code

Unlike `ShakeMath`, this half genuinely needs the Android SDK to compile
- the same reason `BillingManager`, `CrashReporter`, and `OnboardingScreen`
above are shown as illustrative code rather than shipped as compiled
snippets. `snippets/` stays pure-JVM by design (`DECISIONS.md` D-03); this
is reference code to copy and compile inside your own Android project,
where the real SDK already exists.

```kotlin
class ShakeDetector(
    private val sensorManager: SensorManager,
    private val thresholdG: Float = ShakeMath.DEFAULT_THRESHOLD_G,
    private val minIntervalMs: Long = ShakeMath.DEFAULT_MIN_INTERVAL_MS
) {
    /** Emits once per detected, debounced shake. No separate start()/stop() -
     * the Flow's own collection lifecycle registers and unregisters the listener. */
    fun shakes(): Flow<Unit> = callbackFlow {
        val accelerometer = sensorManager.getDefaultSensor(Sensor.TYPE_ACCELEROMETER)
        if (accelerometer == null) {
            close()  // no accelerometer - emit nothing, don't error
            return@callbackFlow
        }

        var lastEmittedAtMs = 0L
        val listener = object : SensorEventListener {
            override fun onSensorChanged(event: SensorEvent) {
                val gForce = ShakeMath.gForce(event.values[0], event.values[1], event.values[2])
                if (!ShakeMath.isShake(gForce, thresholdG)) return

                val now = SystemClock.elapsedRealtime()
                if (!ShakeMath.shouldEmit(now, lastEmittedAtMs, minIntervalMs)) return

                lastEmittedAtMs = now
                trySend(Unit)
            }
            override fun onAccuracyChanged(sensor: Sensor?, accuracy: Int) {}
        }

        sensorManager.registerListener(listener, accelerometer, SensorManager.SENSOR_DELAY_UI)
        awaitClose { sensorManager.unregisterListener(listener) }
    }
}
```

**Deliberately framework-agnostic** - a plain class with a constructor and
a `Flow`, no Hilt/Koin annotations. Wire it into whichever DI container
the app already uses (`@Provides`, `single { }`, or manual construction
all work identically), which matters the moment a portfolio has apps on
different DI frameworks - it drops into any of them unmodified.

### Gate it the same way as an unshipped Pro feature

If the shake trigger is wired up before the content behind it is ready,
apply the exact same pattern as billing's `PRO_FEATURES_LIVE` flag in
§1: a single boolean that hides the trigger entirely until there's
something real to reveal. Shipping a gesture that does nothing yet is
the same anti-vaporware mistake as shipping a paywall for features that
don't exist.

### What's a portable snippet vs. what stays app-specific

The **detection** math (is this a shake? has enough time passed?) is
generic enough to be a real snippet - `kanon.sensor.ShakeMath`. The
**selection** logic, if there are multiple things a shake could reveal, is
equally generic - `kanon.random.weightedPick`, also in `snippets/jvm/`.
**What happens when a shake fires** — which screen it navigates to, what
content it reveals, how that interacts with the rest of the app's
navigation — is inherently app-specific and doesn't belong in a shared
snippet. Keep the trigger and the payload cleanly separated at that
boundary.
