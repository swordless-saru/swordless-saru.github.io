---
title: Troubleshooting
grand_parent: Kanon
parent: Android
nav_order: 7
---

# Troubleshooting

Symptom to root cause to fix. Every entry here came from a real bug where
the obvious diagnosis was wrong.

---

## Layout and insets

**A pinned bottom button clears the nav bar visually, but eats the first
tap on a gesture-nav device.**
*Cause:* the OS edge-swipe detection zone is taller than the drawn
navigation bar, so the button sits inside the gesture region.
*Fix:* pad with the union, not just the nav bar:
```kotlin
Modifier.windowInsetsPadding(
    WindowInsets.navigationBars.union(WindowInsets.systemGestures)
)
```

**That exact fix is applied and the bug persists on-device.**
*Cause:* an ancestor `Scaffold` — commonly one wrapping an entire
`NavHost` — owns insets bookkeeping for the route, even if that route
renders no bars of its own. Your correct fix is being overridden from
above.
*Fix:* zero the ancestor's insets for that route
(`contentWindowInsets = WindowInsets(0, 0, 0, 0)`), or move to per-screen
Scaffolds. See [`app-shell.md`](app-shell.md).
*Note:* this is the one to remember. The symptom makes you doubt a fix
that was right all along, and the real cause is in a different file.

**A `ModalBottomSheet`'s action button sits under the nav bar.**
*Cause:* `ModalBottomSheet` doesn't auto-inset the way a full-screen
`Scaffold` route does.
*Fix:* `Modifier.navigationBarsPadding()` on the sheet's content column.

**Doubled padding at the top or bottom of a screen.**
*Cause:* two nested `Scaffold`s both applying system-bar insets.
*Fix:* zero the inner one.

## Vectors and assets

**A converted brand-mark vector still looks padded or off-center after a
correct-looking fix.**
*Cause:* the source SVG's `<g transform>` was a full affine matrix (scale
plus translate), but the port only applied the translation. The resulting
numbers look plausible and are wrong.
*Fix:* apply the full affine (`x' = a*x + c*y + e`, `y' = b*x + d*y + f`)
and verify with a bounding-box measurement, not by eye.

**An asset "looks the same" but behaves differently after a re-check.**
*Cause:* it was silently overwritten with new content at the same path.
*Fix:* re-read the file's actual contents. A directory listing, filename,
or timestamp is not confirmation.

## Build and dependencies

**`NoSuchMethodError` or `NoClassDefFoundError` at runtime in Compose code
that compiled fine.**
*Cause:* a Compose artifact version was overridden individually while the
rest came from the BOM, producing a mutually-incompatible set.
*Fix:* remove the individual pin; bump the BOM as a unit.

**Gradle sync fails after a dependency change.**
*Fix:* `--refresh-dependencies`, then clean and delete `.gradle/` if it
persists. Check the Kotlin/Compose-compiler compatibility table before
assuming a cache problem — a version mismatch there produces confusing
downstream errors.

**No committed Gradle wrapper, and no local JDK to generate one.**
*Cause:* a valid `gradle-wrapper.jar` can only be produced by actually
running Gradle once (`gradle wrapper`). Without a local JDK/Gradle
install, that command has nowhere to run — leaving CI to silently
provision whatever Gradle is newest, with no version pin anywhere.
*Fix:* generate it on a CI runner instead of a laptop. Add a
`workflow_dispatch` workflow that checks out the repo on `ubuntu-latest`
(which has a JDK), installs Gradle via `gradle/actions/setup-gradle`,
runs `gradle wrapper --gradle-version <version>`, and uploads
`gradlew` / `gradlew.bat` / `gradle/wrapper/gradle-wrapper.{jar,properties}`
as a build artifact. Trigger it via the GitHub API, download the
artifact, and commit the four files locally. See Kanon's own
`.github/workflows/regenerate-wrapper.yml` and `DECISIONS.md` D-04 for a
worked example — copy the workflow file verbatim into any repo with the
same gap.
*Two things that silently break this if skipped:* the executable bit on
`gradlew` doesn't survive a Windows `git add` unless set explicitly
(`git update-index --chmod=+x gradlew`), and a CRLF-converted shebang
line fails on Linux CI with "bad interpreter" — pin `gradlew` to LF via
`.gitattributes` regardless of any one contributor's local `autocrlf`
setting.
*Also check:* a Gradle major-version bump usually raises the *minimum*
supported Kotlin Gradle Plugin version too (Gradle 9 needs Kotlin Gradle
Plugin 2.0.0+, up from 1.6.10 under Gradle 8) — bump the plugin version
in the same change, or the wrapper upgrade trades one sync failure for
another.

## Git

**`git push` succeeds but the commit isn't on the remote.**
*Cause:* a chained `git commit && git push` can report success (exit 0,
no error) while the push doesn't land. Also possible: a stale tracking
ref sent it to a repo you renamed away from.
*Fix:* `git fetch` before and after any push that matters, as separate
verified steps. Check `git remote -v` and `git branch -vv` in any repo
that has ever been renamed.

**`git push` 403s, but committing locally worked fine.**
*Cause:* the repo is archived on GitHub. Local git has no idea.
*Fix:* unarchive it, or move the work to the successor repo. Don't try to
work around the flag.

**A full-file rewrite silently deleted someone else's change.**
*Cause:* the rewrite was drafted from a copy of the file read before the
other change landed. A blind overwrite cannot detect this.
*Fix:* `git fetch` and diff immediately before pushing any full-file
rewrite — not merely at the start of the session. A plan made an hour ago
can already be stale.

## Process

**A documented "standard" turns out to contradict the actual code.**
*Cause:* the doc recorded an intention, or the code moved and the doc
didn't. Both are common; the second is near-certain over months.
*Fix:* verify against the code before writing anything down, and again
before acting on it. A grep costs five minutes; building on a false
premise costs considerably more.

**A stale fact keeps reappearing after being fixed.**
*Cause:* foundational claims get copy-pasted into every consuming
document — READMEs, architecture docs, context files — and then drift
independently. Fixing the canonical source doesn't fix the copies.
*Fix:* when a foundational fact changes, grep every related repo for the
old claim rather than only fixing the document that owns it.
