---
title: Advanced Excel Guide
parent: Paraclete
nav_order: 4
---

# Advanced Excel Guide

Paraclete writes a plain `.xlsx` file - the same format Excel, Google
Sheets, LibreOffice, and Numbers all read natively. The app only touches
its own tabs (`_Nudges`, `_Categories`, `_Settings`) and, within those
tabs, only the columns up to each sheet's "-- custom columns below --"
marker. Everything else in the file is yours to shape however you want,
and the app will never overwrite it.

This page is a starting set of things power users can do. It'll grow as
more patterns get worked out - if you build something useful, it's worth
sharing back so it can be added here.

## Before you start

- **Work on a copy first** while you're experimenting with anything
  structural (a new pivot table, a formula you're not sure about).
- **Never rename an `_`-prefixed tab or a header cell.** The app finds its
  own sheets and columns by exact name/position; renaming one just makes
  the app treat that data as missing. Any tab *without* the underscore is
  fair game to rename, reorder, or delete freely.
- **`_Nudges` column order** (for writing your own formulas directly
  against cell ranges, rather than named columns): `ID`, `Name`,
  `CategoryID`, `Enabled`, `WindowMode`, `TargetDateTime`,
  `SpreadMinutes`, `Direction`, `WindowStart`, `WindowEnd`, `SpacingType`,
  `NudgeCount`, `FixedIntervalMinutes`, `RecurrenceMode`, `IntervalCount`,
  `IntervalUnit`, `WeeklyDays`, `MonthlyDays`, `ReminderMode`, `SoundUri`,
  `QuietHoursEnabled`, `QuietHoursStart`, `QuietHoursEnd`, `Notes`,
  `CreatedAt`, `UpdatedAt`, `DeletedAt` - columns A through AA in that
  order. `_Categories` is `ID`, `Name`, `Icon`, `SortOrder`, `Notes` -
  columns A through E.

## The one join you'll want: CategoryID → Category Name

`_Nudges`' `CategoryID` (column C) is a foreign key into `_Categories`'
`ID` (column A) - a pivot table can't follow that join on its own, so add
one helper column first, in the custom-columns area of `_Nudges`:

```
=IFERROR(VLOOKUP(C2,_Categories!$A:$B,2,FALSE),"Uncategorized")
```

Fill that down the sheet, then every technique below that wants "category
name" can just point at your helper column instead of repeating the
lookup.

## Pivot tables

**Nudge count by category:**
1. Select the `_Nudges` tab (including your new helper column).
2. Insert → PivotTable, source range covering `_Nudges!A:Z` plus the
   helper column.
3. Rows: your helper column. Values: `Name` (Count).

Put the pivot table on its own new tab (anything not starting with `_` is
safe) so it doesn't fight with the app's own layout of `_Nudges`.

## Formulas you can copy directly

**How many nudges are currently enabled:**
```
=COUNTIF(_Nudges!D:D,"Yes")
```

**Enabled AROUND-mode nudges targeting the next 7 days:**
```
=COUNTIFS(_Nudges!F:F,">="&TODAY(),_Nudges!F:F,"<"&TODAY()+7,_Nudges!D:D,"Yes")
```
`_Nudges` column F is `TargetDateTime` - this only makes sense for rows
where `WindowMode` (column E) is `AROUND`; `BETWEEN`-mode rows use
`WindowStart`/`WindowEnd` (columns I/J) instead.

**Count of nudges per category, once you have the helper column from
above (say it's column AB):**
```
=COUNTIF(AB:AB,"Health")
```
Swap `"Health"` for any category name, or reference a cell instead of
hardcoding it so a whole report can be driven by one dropdown.

## Conditional formatting ideas

- Highlight `_Nudges` rows where `ReminderMode` (column S) is `CRITICAL`
  - useful for seeing at a glance which nudges bypass Do Not Disturb.
- Highlight rows where `Enabled` (column D) is `No` - a quick visual list
  of everything currently paused, without opening the app.

## A custom tab: simple "this week" dashboard

Add a new tab (call it `This Week`, `Dashboard`, whatever you like - no
`_` prefix needed since it's yours) and build a printable summary using
the formula patterns above, laid out however you want. Since it's a
normal tab, it survives every app save untouched, same as any custom
column.

## Something not covered here?

This guide is deliberately a starting point, not a complete reference.
The underscore-prefix convention (see the
[Kanon standards](../kanon/android/) this
app follows) exists specifically so this kind of extension is safe by
construction - if you build a pattern worth sharing, it can be added to
this page in a future update.
