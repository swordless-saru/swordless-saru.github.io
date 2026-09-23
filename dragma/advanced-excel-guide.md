---
title: Advanced Excel Guide
parent: Dragma
nav_order: 4
---

# Advanced Excel Guide

Dragma writes a plain `.xlsx` file - the same format Excel, Google Sheets,
LibreOffice, and Numbers all read natively. The app only touches its own
tabs (all prefixed with `_`, like `_Income` and `_Accounts`) and, within
those tabs, only the columns up to each sheet's "add custom columns after
this point" marker. Everything else in the file is yours to shape however
you want, and the app will never overwrite it.

This page is a starting set of things power users can do. It'll grow as
more patterns get worked out - if you build something useful, it's worth
sharing back so it can be added here.

## Before you start

- **Work on a copy first** while you're experimenting with anything
  structural (a new pivot table, a formula you're not sure about). Dragma
  keeps its own pre-write safety snapshots in `Backups/`, but those exist
  to protect *app* data, not to be your only safety net for hand-edits.
- **Never rename an `_`-prefixed tab.** The app finds its own sheets by
  exact name; renaming one just makes the app treat the file as if that
  sheet doesn't exist. Any tab *without* the underscore is fair game to
  rename, reorder, or delete freely.
- **Formulas referencing app tabs are stable across app updates** for a
  given major schema version - the tab names and the columns up to each
  sheet's managed boundary don't change without a version bump (see
  `Summary!B1`). A future release can still *add* columns after the
  marker; it won't move or remove the ones your formulas depend on.

## Pivot tables

A pivot table built against `_Expenses` or `_Income` keeps working after
every save - Dragma rewrites the underlying rows, not the pivot table
itself, and both Excel and Sheets recalculate a pivot automatically (or on
next open) when its source range's data changes.

**Spending by category, by month:**
1. Select the `_Expenses` tab.
2. Insert → PivotTable, source range `_Expenses!A:M` (or the whole sheet).
3. Rows: `Category`. Columns: `Date` (grouped by Month). Values: `Amount`
   (Sum).

Put the pivot table on its own new tab (anything not starting with `_` is
safe) so it doesn't fight with the app's own layout of `_Expenses`.

## Formulas you can copy directly

These are the same techniques the app's own `_Summary` tab uses - safe to
reuse anywhere:

**Net worth trend (put this on your own tab):**
```
=SUM(_Accounts!E:E)
```
`_Accounts` column E is Current Balance (a formula result itself, summing
each account's initial balance plus its transaction history). Snapshot
this value into a dated row periodically (manually, or by copy-pasting
values) to build a net worth history the app doesn't track natively.

**Total spent in a specific category, year to date:**
```
=SUMPRODUCT((YEAR(_Expenses!B2:B1000)=YEAR(TODAY()))*(_Expenses!E2:E1000="Groceries")*(_Expenses!D2:D1000))
```
Swap `"Groceries"` for any category name, or reference a cell instead of
hardcoding it so a whole report can be driven by one dropdown.

**Average transaction size for an account:**
```
=AVERAGEIF(_Expenses!G:G,"Checking",_Expenses!D:D)
```
`_Expenses` column G is Account.

## Conditional formatting ideas

- Highlight `_Expenses` rows where `Amount` (column D) exceeds a threshold
  - useful for spotting an unusually large purchase at a glance.
- Color-scale `_Categories`' `% used` column (formula-driven, so this
  updates live) to make over-budget categories jump out without opening
  the app.

## A custom tab: simple monthly report

Add a new tab (call it `Monthly Report`, `Dashboard`, whatever you like -
no `_` prefix needed since it's yours) and build a printable one-page
summary using the formula patterns above, laid out however you want. Since
it's a normal tab, it survives every app save untouched, same as any
custom column.

## Something not covered here?

This guide is deliberately a starting point, not a complete reference.
The underscore-prefix convention (see the
[Kanon standards](../kanon/android/) this
app follows) exists specifically so this kind of extension is safe by
construction - if you build a pattern worth sharing, it can be added to
this page in a future update.
