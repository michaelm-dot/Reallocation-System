# Reallocation Desk

A single-file web app for working the O2I reallocation queue: Amazon ungating → Walmart → Dom,
with offer sheets generated at each step.

Everything is in `reallocation-desk.html`. No build step, no server. Open it in a browser, or
serve it with GitHub Pages and use the URL.

## First run

The app ships **without** the store roster. On first open it will ask for it:

1. Open the app
2. **Load / replace the store budget** on the Upload tab
3. Pick `stores-budget.csv` (or the Stores Budget workbook)

It is saved on that device and you will not be asked again. Everyone who uses the app does this
once. Needs `Store_Code`, `Partner_Name`, `Appetite_AMZ`, `Appetite_WM` and `G&G_Ungate_Date`;
stores with a blank G&G date are ignored.

> `stores-budget.csv` holds partner names and appetite figures. **Keep it out of this repo** and
> share it internally. The app file itself contains no business data, which is what makes it
> safe to publish.

---

## The flow

```
Upload  →  All items  →  Amazon ungating  →  Walmart  →  Dom
                                  ↓              ↓
                             Offer sheets ←──────┘
```

Upload one sheet — the O2I export. Everything downstream is built from it. The store roster
is already baked in, so there is nothing else to load.

---

## Amazon ungating

The worksheet is generated from the upload: one row per ASIN, one column per store, ordered
newest to oldest by `G&G_Ungate_Date`.

**The old five decide the row.** Five stores are drawn at random from the oldest pool and pinned
left of the divider. Work those first.

| What you find | What happens |
|---|---|
| Listing status is anything but `No Error` | straight to Walmart |
| All five old clients gated | `No ungated clients` → Walmart |
| One of the five is ungated | that client holds the item, and the sweep opens |

When an old client holds an item, every client with a **newer** G&G date still has to be checked,
because a newer ungated client outranks them and takes the offer sheet instead. The Result cell
counts down how many are left and turns green when the sweep clears.

Other controls:

- **Account problem** — the `!` on any column header drops that store from the roster. It stops
  receiving items and the next store by G&G date takes its place.
- **Find a client** — scrolls to and highlights a column without moving anything.
- **Shuffle** — redraws the random old five. Marks already made are kept.

## Offer sheet criteria

1. The newest ungated client takes it
2. Items accumulate per client until the total clears **$100**
3. An item that exceeds a client's remaining `Appetite_AMZ` passes whole to the next ungated
   client — never split
4. Stores with a blank `G&G_Ungate_Date` are excluded entirely

Approval is assumed, so a qualifying basket becomes an offer sheet immediately.

---

## Walmart

Rows arrive from a listing issue, an all-gated sweep, or a Walmart-sourced identifier with no ASIN.

Typed in per row: WM #, UPC, Unit price, Walmart Link, Bundle size, ROI, Profit,
Target sell price, Breakeven, and a listing status that is **separate** from the Amazon one.

Clients are added as **columns** — type a store code once and it applies across every row, each
with its own Gated / Ungated dropdown.

| Condition | Outcome |
|---|---|
| Listing status is anything but `No Error` | Dom |
| WM # is `No WM # found` | Dom |
| An ungated client is found | Walmart offer sheet |
| Every listed client gated | Dom |

No $100 floor and no appetite check on this side — Walmart is a manual call.

---

## Dom

Everything that found no home. Same 18-column layout as an offer sheet, but carrying the **real
tracking codes** from the O2I upload rather than generated ones.

---

## Offer sheet format

18 columns, matching `Offer Sheet Format.xlsx`:

```
Store Code | Tracking Code | ASIN | ASIN Eligible | ASIN Desired | Total Price | UPC | Unit Cost |
Recommended Sellable Qty | 90Days Average | Status | Notes | Bundle Size | ROI % |
Profit Per Unit | 30D Profit | Brand | Distributor
```

Formulas:

| Field | Rule |
|---|---|
| Recommended Sellable Qty | `Order_quantity ÷ Bundle_size` |
| Total Price | `Unit Cost × Qty × Bundle Size` |
| 30D Profit | `Profit Per Unit × Qty` |
| Summary ROI | amount-weighted: `SUMPRODUCT(ROI, Amount) ÷ SUM(Amount)` |
| Status | `IF(P<0,"Not Profitable",IF(P<50,"Low Profit","Profitable"))` |
| Notes | `IF(P<0,"Negative ROI",IF(P<50,"Low Profit","Good to Order"))` |

`ASIN Eligible` and `ASIN Desired` are always `TRUE`. Status and Notes stay editable per row.

Each sheet has a **Print / PDF** button that isolates it, drops the app furniture and lays it out
A4 landscape for sending to a client.

---

## Reallocation results

The O2I layout back out — 21 original columns plus **Client** — for rows that landed with one.
Walmart rows carry whatever was typed in.

| Field | Rule |
|---|---|
| Bundle_price | `Unit_price × Bundle_size` |
| Initial_Sellable | `Order_quantity ÷ Bundle_size` |
| Order_confirmed_amount | `Unit_price × Order_quantity` |

For an untouched row these reproduce the source values exactly, so the file round-trips cleanly.

---

## All items

Every row on one screen with its current stage, and the place to edit. Bundle, Unit,
90d average, Breakeven, Profit and ROI are all typed here and flow through to the offer sheets
and the results export. Grey placeholders show the original O2I figure; an empty box means
"use the original".

Edits move numbers that matter — dropping a unit price can pull an item back under the $100
minimum, and raising it can push it past a client's appetite and out to Walmart.

---

## Multiple files

Each upload is saved as its own file, listed on the Upload tab with row counts and progress.
Open, rename and delete from there. Ungating marks, Walmart entries, client columns, edits and
the drawn old five are all kept per file, so reopening yesterday's export finds it exactly as
you left it.

---

## Known limits

- The `.xlsx` export cannot carry colours or bold — SheetJS's free build drops cell styling.
  Use Print / PDF for anything client-facing.
- The **All clients** view renders a dropdown per store per row. At 299 stores and a large export
  that gets heavy; the narrower views stay fast.
- A bundle size that does not divide the order quantity gives a fractional sellable count.
  It is left visible rather than rounded, as a signal that the bundle size is wrong.
