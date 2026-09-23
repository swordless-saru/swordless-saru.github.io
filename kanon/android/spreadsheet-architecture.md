---
title: Spreadsheet Architecture
grand_parent: Kanon
parent: Android
nav_order: 6
---

# Spreadsheet Architecture

Patterns for apps where a plain `.xlsx` file (via Apache POI) *is* the
database — no Room, no SQLDelight, no server. Proven in production across
Dragma and Paraclete.

---

## 1. Prefix app-managed tabs with `_`, reserve plain names for the user

Every tab the app creates and writes to gets an underscore prefix:
`_Accounts`, `_Categories`, `_Nudges`. Tabs without a prefix are the
user's — a pivot table, a chart sheet, a hand-built report.

**Why this matters:** a spreadsheet-first app is, by definition, handing
the user a real file they're encouraged to open in Excel/Sheets directly.
The moment that's true, tab-name collision becomes a real risk: a user
names their own tab `Summary` for a personal dashboard, then a future app
version adds its *own* `Summary` tab and either overwrites the user's
work or silently misreads it as app data. Reserving the entire
unprefixed namespace up front makes that collision structurally
impossible, this release and every future one — not just unlikely.

**This is a decision to make before first release, not a migration to
run later.** Once real users have real files with real (unprefixed)
app-managed tabs in them, renaming those tabs becomes a breaking schema
change with actual user data at stake. Both Dragma and Paraclete adopted
this convention while still pre-launch specifically to make it free.

**Implementation:**
- Tab names live as named constants in one reader/schema class
  (`SpreadsheetReader.SHEET_ACCOUNTS = "_Accounts"`, etc.) — never as
  string literals scattered across writers, templates, or formula
  strings. See §4.
- Applies to *every* app-created tab, including a human-readable
  "Instructions" tab — it's still app-managed content, so it still needs
  the guarantee.
- Underscore-prefixing doesn't affect tab *order*. Neither Excel nor
  Google Sheets auto-sorts tabs by name, so an "Instructions" tab meant
  to appear first still appears first as `_Instructions`.

## 2. Custom columns: reserve trailing space, never touch it

Each managed sheet defines a `LAST_MANAGED_COL` constant. The app reads
and writes columns up to that point and leaves everything after it
completely alone — never overwritten, never validated, never assumed to
be empty.

Put a visible marker in the header row after the last managed column
(e.g. a literal `"-- add custom columns after this point --"` cell) so a
user editing the file directly can see exactly where the boundary is
without needing to read documentation first.

**Why a hard boundary beats "just don't touch columns you don't
recognize":** the latter requires re-deriving, on every write, which
columns are "yours" vs. "app's" — genuinely ambiguous once a user adds a
column the app can't distinguish from a not-yet-implemented app column.
A single position-based cutoff is unambiguous by construction and costs
one constant per sheet.

## 3. Schema version lives in the file, not just in code

Stamp a version string in a well-known cell (`_Summary!B1`, or an
equivalent single-purpose sheet) every time the file is written. This is
what lets a future migration path detect an old file shape and upgrade
it, rather than guessing from column count or crashing on the first
missing column.

Bump this version deliberately whenever the *managed* column/tab shape
changes — not on every release, only on a real schema change. A tab
rename (see §1) is exactly this kind of change.

## 4. No hardcoded sheet-name strings outside the reader

Every sheet name, in every writer, template builder, and formula string
(`"=SUM(_Accounts!E:E)"`), must resolve through the same constant the
reader uses — never a second literal typed by hand elsewhere. This is a
plain DRY violation waiting to happen: a sheet gets renamed once, in one
place, and every hardcoded copy elsewhere silently goes stale. Apache
POI formula strings are especially easy to miss here, since they're just
`String`s the compiler can't check against the sheet name they
reference — grep for the old name portfolio-wide after any rename,
across writers *and* formula strings, not just the constants file.

## 5. Encourage power users — this is a real spreadsheet

Once §1–4 are in place, there's no remaining reason to treat the file as
a black box the user shouldn't touch. Say so explicitly, in the app and
in its docs:

- **README / docs site:** a schema table naming every managed tab, plus
  a short "power users" section pointing at pivot tables, custom
  formulas referencing the managed tabs, and custom report tabs — with
  one or two copy-pasteable formula examples.
- **In-file instructions tab:** the same pitch, since not every user
  reads external docs before opening the file.

This costs nothing beyond the writing — the guarantees in §1 and §2 are
exactly what make it *safe* to actively invite this kind of use, rather
than merely tolerate it.
