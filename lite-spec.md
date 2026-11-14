# Freelancer Client & Invoice Tracker — Lite Build Spec

> Purpose: exact build instructions for constructing the **Lite** version in Notion.
> The next browser task follows this spec step by step. Do not improvise field names,
> types, or option labels — copy them verbatim.
>
> Lite scope: **two databases only** — Clients + Invoices. No Projects database
> in Lite (that is a full-version feature).
> Pricing context: full version $19 (one-time, Gumroad). Lite is free.
> Verification window: 7 days, Lite downloads ≥ 20 (excluding self-downloads) = pass.

## 0. Workspace setup

- Notion account: `alapha888@gmail.com` (create the account if none exists).
- Create a new page titled **"Freelancer Client & Invoice Tracker — Lite"**.
- Inside that page, create the two databases below as **full-page databases**
  (not inline), so their URLs stay stable for the Duplicate link.
- Language: all database names, property names, and option labels in **English**.
- Currency: all sample amounts in **USD**.
- "Today" for all sample-date logic: **2026-11-14**.

## 1. Clients database

Database name: `Clients`

| Property name | Type | Example value | Notes |
|---|---|---|---|
| Name | Title | Northwind Studio | Client / company name |
| Email | Email | sarah@northwindstudio.example | Main billing contact |
| Currency | Select | USD | Options: `USD`, `EUR`, `GBP`. Lite sample data uses USD only. |
| Payment terms | Select | Net 14 | Options: `Due on receipt`, `Net 7`, `Net 14`, `Net 30` |
| Notes | Text | Prefers invoices by email on Fridays. | Free text; keep short |

Optional (nice to have, add only after the required fields work):
`Total outstanding` — Rollup, see §3.

## 2. Invoices database

Database name: `Invoices`

| Property name | Type | Example value | Notes |
|---|---|---|---|
| Invoice # | Title | INV-2026-001 | Text title. Format `INV-YYYY-NNN`, sequential. |
| Client | Relation → Clients | Northwind Studio | Single relation (one client per invoice). Enable "Show on Clients" so each client shows its invoices. |
| Amount | Number | 1850 | Format: Number, USD (`$`), 2 decimals. |
| Currency | Select | USD | Options: `USD`, `EUR`, `GBP`. Must match the linked client's Currency; if they differ, the client's Currency wins and the invoice value is corrected, not silently kept. |
| Issue date | Date | 2026-11-02 | Date the invoice was issued. |
| Due date | Date | 2026-11-16 | Computed from Issue date + client payment terms when creating real invoices; sample data uses the fixed dates in §5. |
| Status | Select | Sent | Options (exact labels): `Draft`, `Sent`, `Paid`, `Overdue`, `Void`. **Do not set `Overdue` by hand** — see §3: `Overdue?` is a formula. `Overdue` exists as a status option only for manual override / legacy imports. |
| Paid date | Date | — (empty until paid) | Fill only when Status becomes `Paid`. |
| Overdue? | Formula (Checkbox) | `true` / `false` | See §3. |
| Days overdue | Formula (Number) | `9` | See §3. |
| Outstanding amount | Formula (Number) | `1850` | See §3. |

### Relation + Rollup usage

1. `Invoices.Client` is a **Relation** to `Clients` (single, not multi).
2. On `Clients`, add a **Rollup** named `Total outstanding`:
   - Relation: `Invoices`
   - Property: `Outstanding amount`
   - Calculate: `Sum`
   - This is the per-client "who owes me how much" number. It sums only
     unpaid, non-void invoices because `Outstanding amount` is already 0
     for Paid/Void rows (see §3).
3. Also on `Clients`, optionally add a Rollup `Invoices count`:
   Relation `Invoices`, Property `Invoice #`, Calculate `Count all`.
4. Do **not** use a Rollup directly on `Amount` with a status filter — the
   formula-field route above is the robust one and works in every Notion plan.

## 3. Formula fields (copy verbatim)

### 3.1 `Overdue?` (Checkbox)

```notion
and(prop("Status") == "Sent", prop("Due date") < now())
```

- `true` only when the invoice was sent and its due date is in the past.
- Draft invoices are never overdue; Paid/Void invoices are never overdue.
- Note: `now()` is evaluated at view time; an invoice flips to overdue
  automatically on the morning after its due date. No manual update needed.

### 3.2 `Days overdue` (Number)

```notion
if(prop("Overdue?"), dateBetween(now(), prop("Due date"), "days"), 0)
```

- Whole days past the due date; `0` when not overdue.
- Example: due 2026-11-05, viewed 2026-11-14 → `9`.

### 3.3 `Outstanding amount` (Number, USD)

```notion
if(or(prop("Status") == "Paid", prop("Status") == "Void"), 0, prop("Amount"))
```

- The amount still owed on this invoice; `0` once paid or voided.
- This field is what the Clients rollup sums (§2).

### 3.4 Per-client total (Rollup on Clients, not a formula)

- Property `Total outstanding` on `Clients`: Rollup → Relation `Invoices`,
  Property `Outstanding amount`, Calculate `Sum`.
- No formula text needed; configure in the Rollup property dialog.

## 4. Views

### Invoices database

| View name | Type | Filter | Sort / Group | Purpose |
|---|---|---|---|---|
| All invoices | Table | — (none) | Sort: Issue date, descending | Full list, default view |
| Overdue | Table | `Overdue?` is checked | Sort: Days overdue, descending | **Differentiator #1.** Every unpaid, past-due invoice in one place, worst first. This is the view free templates usually lack. |
| Unbilled work | Table | `Status` is `Draft` | Sort: Issue date, ascending | **Differentiator #2.** Work done, invoice not yet sent. Oldest draft first so nothing sits unsent. |
| By client | Board or Table grouped by Client | — (none) | Group by `Client` | Per-client outstanding at a glance; combine with the Clients rollup. |

### Clients database

| View name | Type | Notes |
|---|---|---|
| All clients | Table | Default. Show `Total outstanding` column. |
| Owes me | Table | Filter: `Total outstanding` > 0. Sort: `Total outstanding`, descending. |

### View decision (recorded trade-off): Unbilled work = Status `Draft`

- **Chosen:** the "Unbilled work" view filters on `Status = Draft`. No separate
  "Unbilled" checkbox field in Lite.
- **Why:** fewer fields to maintain; one status drives both the view and the
  workflow (Draft → Sent → Paid). Lite stays a two-database build.
- **Cost:** Lite cannot distinguish "work logged but not yet invoiced" from
  "invoice drafted but not sent" — in Lite they are the same thing.
- **Full version fix:** the $19 version adds a **Projects** database; unbilled
  work is then derived from logged project work with no invoice attached, and
  Draft keeps its narrower meaning (invoice written, not sent).

## 5. Sample data (enter exactly; dates in the 2026-11 window)

### Clients (5)

| Name | Email | Currency | Payment terms | Notes |
|---|---|---|---|---|
| Northwind Studio | sarah@northwindstudio.example | USD | Net 14 | Prefers invoices by email on Fridays. |
| Brightline Consulting | hello@brightline.example | USD | Net 30 | Monthly retainer, invoices on the 1st. |
| Cedar & Co. | accounts@cedarandco.example | USD | Net 14 | — |
| Pixel Forge | billing@pixelforge.example | USD | Net 30 | Requires PO number on invoice. |
| Harbor Books | hello@harborbooks.example | USD | Due on receipt | Small press, pays fast. |

### Invoices (8 — covers Paid / Sent-not-due / Overdue / Draft)

| Invoice # | Client | Amount (USD) | Currency | Issue date | Due date | Status | Paid date |
|---|---|---|---|---|---|---|---|
| INV-2026-001 | Northwind Studio | 1850 | USD | 2026-11-02 | 2026-11-16 | Sent | — |
| INV-2026-002 | Brightline Consulting | 2400 | USD | 2026-10-28 | 2026-11-05 | Sent | — |
| INV-2026-003 | Cedar & Co. | 620 | USD | 2026-11-03 | 2026-11-17 | Paid | 2026-11-10 |
| INV-2026-004 | Pixel Forge | 3100 | USD | 2026-10-20 | 2026-11-10 | Sent | — |
| INV-2026-005 | Harbor Books | 480 | USD | 2026-11-06 | 2026-11-20 | Sent | — |
| INV-2026-006 | Northwind Studio | 950 | USD | 2026-11-09 | 2026-11-23 | Draft | — |
| INV-2026-007 | Brightline Consulting | 1200 | USD | 2026-11-01 | 2026-11-08 | Paid | 2026-11-07 |
| INV-2026-008 | Cedar & Co. | 350 | USD | 2026-11-12 | 2026-11-26 | Draft | — |

### Expected results after entering the data (verify before publishing)

- Viewed 2026-11-14: **Overdue** view shows exactly 2 rows —
  INV-2026-002 (Days overdue `9`) and INV-2026-004 (Days overdue `4`).
- **Unbilled work** view shows exactly 2 rows — INV-2026-006 ($950) and
  INV-2026-008 ($350). Draft total: **$1,300** not yet invoiced.
- **Total outstanding** (all clients): 1850 + 2400 + 3100 + 480 = **$7,830**.
  Overdue portion: 2400 + 3100 = **$5,500**.
- Per-client outstanding: Northwind $1,850 · Brightline $2,400 ·
  Cedar & Co. $0 · Pixel Forge $3,100 · Harbor Books $480.
- **All invoices** shows 8 rows; **By client** groups them under 5 clients.

## 6. Dashboard page structure

Page title: **Freelancer Client & Invoice Tracker — Lite**

From top to bottom:

1. **Intro block (text, 2–3 lines):** what this template is for —
   one place to answer "who owes me, how much, and what is overdue".
   State plainly that this is the free Lite version (Clients + Invoices).
2. **Quick start (numbered list, 4 steps):** Duplicate → replace the 5 sample
   clients with your own → add your invoices → check the Overdue view weekly.
3. **Linked views (3 linked database views, in this order):**
   - `Overdue` (Invoices, filtered as §4)
   - `Unbilled work` (Invoices, filtered as §4)
   - `All clients` (Clients, with Total outstanding visible)
4. **What's in Lite / What's in the full $19 version (table):**

   | Feature | Lite (free) | Full version ($19) |
   |---|---|---|
   | Clients database | ✓ | ✓ |
   | Invoices database | ✓ | ✓ |
   | Projects database | — | ✓ (work log → unbilled work, derived) |
   | Overdue view (auto, formula-driven) | ✓ | ✓ |
   | Unbilled work view | ✓ (Status = Draft) | ✓ (derived from Projects, more precise) |
   | Per-client total outstanding (rollup) | ✓ | ✓ |
   | Multi-currency tracking | Basic (per-client / per-invoice select) | Same, plus per-client currency rollups (planned) |
   | Usage guide | 1-page PDF | Same PDF + FAQ updates |
   | Support | Async email + FAQ | Async email + FAQ |

   Do not add rows for features that are not actually planned. Do not claim
   counts (e.g. number of computed fields) that have not been built.
5. **Footer line:** "Lite is free. The full version ($19, one-time) adds the
   Projects database and a more precise unbilled-work view. Support is async
   email + FAQ — no one-on-one setup."

## 7. Differentiation requirement (must survive simplification)

Lite keeps the two features that separate it from the free Notion Marketplace
template "Never Forget an Unpaid Invoice Again" (Clients / Projects / Invoices,
$0) and from the $1.99–$10 Gumroad alternatives:

1. **Overdue view** — formula-driven (`Overdue?` + `Days overdue`), not a
   manual status. In Lite it is the first linked view on the dashboard and a
   named view in the Invoices database. Verification: §5 expects exactly 2
   rows on 2026-11-14.
2. **Unbilled work view** — Draft-status filter surfacing money not yet
   invoiced. In Lite it is the second linked view on the dashboard and a named
   view in the Invoices database. Verification: §5 expects exactly 2 rows,
   $1,300 total.

If either view cannot be built as specified, stop and report back — do not
ship a Lite without them; they are the reason the Lite exists.

## 8. Publish checklist (browser task)

- [ ] Both databases built per §1–§2, formulas pasted verbatim per §3.
- [ ] All 4 Invoices views + 2 Clients views created per §4.
- [ ] Sample data entered per §5; expected results verified (§5, bottom).
- [ ] Dashboard page assembled per §6, including the Lite vs Full table.
- [ ] Page set to public with **Allow duplicate as template** enabled;
      copy the Duplicate link.
- [ ] Duplicate link pasted back into `README.md` (replacing the placeholder)
      and into the GitHub Release `trial-v1` description.
- [ ] Duplicate link tested in a fresh logged-out browser session (page loads,
      Duplicate button works).
