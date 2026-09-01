# Real Estate Management System — Meridian Estates

A working front-end prototype of the Reporting Module & CEO Dashboard. No backend and no
network calls: every figure is computed in the browser from a fixed in-memory transaction
set, so it runs from a static host or straight off a laptop.

**Read [AUDIT.md](AUDIT.md) before the client meeting.** It records what was wrong or
missing in the requirements document, what changed as a result, and six open questions
that need the client's answer.

---

## What it is

A product, not a report. Left sidebar for sections, one page at a time, tabs for
sub-views, dropdown flyouts for quick jumps, and forms that actually save.

```
┌────────────┬────────────────────────────────────────────┐
│ Meridian   │  Properties / Inventory      🔍  ＋New  👤 │
│────────────├────────────────────────────────────────────┤
│ MAIN       │  Inventory │ Purchase register │ Perform.  │
│ ▸ Dashboard│────────────────────────────────────────────│
│ OPERATIONS │  Property inventory        [+ Add property]│
│ ▸ Properties  اسٹاک و مالیت                              │
│ ▸ Sales    │  ┌────────┐┌────────┐┌────────┐┌────────┐  │
│ ▸ Agents   │  │ 25     ││ 56 Cr  ││ 70.6Cr ││ 14.6Cr │  │
│ FINANCE    │  └────────┘└────────┘└────────┘└────────┘  │
│ ▸ Finance  │  [search] [filters]      CSV  Excel  PDF   │
│ ▸ Costs    │  ────────────────────────────────────────  │
│ SYSTEM     │  table…                                    │
│ ▸ Admin    │                                            │
│ 👤 M.Salman│                                            │
└────────────┴────────────────────────────────────────────┘
```

### Sections

| Section | Tabs |
|---|---|
| **Dashboard** | Overview · Alerts |
| **Properties** | Inventory · Purchase register · Performance |
| **Sales** | Sales register · Receivables |
| **Agents** | Agent directory · Commissions |
| **Finance** | Profit & loss · Profit tracking · Cash flow · Payables |
| **Costs** | Expenses · Salaries · Bills · Tax · Zakat |
| **Admin** | Transactions · Audit trail · Users & roles |

Clicking a sidebar section opens it; its chevron opens a flyout so you can jump straight
to any sub-view. A section the signed-in role cannot see anywhere is not rendered at all —
there are no dead menu items.

### Four accounts, four different products

Switch accounts from the user card at the bottom of the sidebar. Each role gets its own
dashboard and its own navigation, not the same screen with things greyed out.

| Account | Role | Lands on |
|---|---|---|
| M. Salman | CEO | Full financial picture — revenue, profit, cash, receivables, payables |
| Faisal Nadeem | Accountant | Cash, payables, bills due, tax and Zakat, quick data entry |
| Omar Farooq | Manager | Properties, sales, agents, portfolio — no company financials |
| Sana Malik | Agent | Own sales, own customers, own commission only |

`Admin → Users & roles` shows the full permission matrix and lets you sign in as anyone.

### Data entry that works

**＋ New** in the top bar, or the action button on each page:

- **Add property** — every §6 field, acquisition costs rolled into total cost
- **Record sale** — creates the sale, the agent commission entry and the receipt together
- **Add expense** — posts to the expense ledger and, if paid, the cash ledger
- **Record payment** — a single money-in / money-out entry

Each form validates (paid cannot exceed total, received cannot exceed selling price, no
future dates, sale cannot pre-date purchase), shows a live calculation panel as you type,
and posts to the real ledgers. Anything you add appears immediately in the KPIs, charts,
tables and audit trail, and is pinned to the top of its table so you can see it.

Transactions can be **voided**, never deleted (§31) — the original record stays and the
audit trail keeps both amounts.

---

## Running it

No build step, no dependencies.

```bash
cd "d:/Aesthetic CEO Reporting Dashboard"
python -m http.server 8080      # or: npx serve .
# open http://localhost:8080
```

Opening `index.html` over `file://` will **not** work — it is an ES module and browsers
block module imports from the filesystem. Use a server.

### Deploy on Vercel via GitHub
1. Push this folder to a GitHub repository, keeping the file names.
2. Vercel → **Add New → Project → Import** the repo.
3. Framework preset **Other**; build command empty; output directory empty (root).
4. Deploy. Everything is static.

---

## Files

| File | What it is |
|---|---|
| `index.html` | Application shell. |
| `app.js` | The whole UI: sidebar, pages, tabs, charts, forms, roles, exports. |
| `app.css` | Design system. |
| `re-data.js` | Data set **and** all financial logic — ranges, filters, KPI aggregation, series, ageing, cash ledger, run rate, permissions, mutations, CSV. |
| `AUDIT.md` | The requirements review: findings, decisions, open questions. |
| `CEO Reporting Dashboard.dc.html`, `support.js` | The original Claude Design prototype. Not the entry point; kept for reference. |

---

## Deep links

Every screen is linkable, which is useful for sending the client one specific view.

```
#dashboard/overview      #properties/inventory     #finance/pnl
#sales/receivables       #admin/users              #account
```

---

## Design

One committed direction, **Ledger Editorial**: warm ivory paper, deep emerald ink, a
single saffron accent, engraved seal and procedural property artwork.
Type is **Poppins** for the interface, **Sora** for headings and figures, **JetBrains
Mono** for table numerals, **Noto Nastaliq Urdu** for Urdu labels.

- **Money is shown in Crore / Lakh by default** — the convention Pakistani real estate,
  the State Bank and the Bureau of Statistics use. Million and full-digit formats are one
  click away in the top bar.
- **Urdu labels** on every page heading.
- **Colour is tokenised**, and UI colours and chart series colours are separate sets — a
  status colour never impersonates a data series.
- **The chart palette is validated, not eyeballed** — lightness band, chroma floor,
  colour-blind separation and contrast, checked against this page's actual ivory surface.
  Every chart also ships a direct-labelled value list, so no number needs a hover.
- **Nothing starts invisible.** No scroll-triggered reveals, no entrance animations from
  zero opacity. Hover and control transitions animate from a visible state.
- **All artwork is inline SVG** — seal, icons, paper grain, and property illustrations
  generated from each property's ID. No image requests; works offline and prints.
- Charts are hand-built SVG. No chart library.

---

## Testing

`app.js` and `re-data.js` are exercised by a browser harness that drives the real UI:
sidebar flyouts, navigation, form validation and saving, cross-entity creation (a sale
also creating its commission and receipt), role switching, permission gating on deep
links, search, and voiding. 29 assertions, all passing, plus a 23-page sweep for
JavaScript errors and stray `undefined` / `NaN`.

---

## Rules enforced in the code

- Outstanding = total − paid, never negative; paid can never exceed the total.
- No payment or audit entry is dated in the future (today is fixed at 01 Sep 2026).
- Dates from form inputs are parsed as **local** calendar dates, so a same-day entry is
  never rejected as "in the future" in Pakistan Standard Time.
- Profit is computed in three stages, so Zakat is never an operating expense:
  `revenue − cost of sales = gross`, `gross − operating costs = operating profit`,
  `operating profit − tax − Zakat = net profit`.
- Gross profit has two defensible bases (AUDIT A3). The default is the cost of the
  properties actually sold; the document's own wording is one click away in the P&L and
  switches **every** profit figure, never just one panel.
- Withholding tax on a sale and corporate income tax are never added together.
- Potential profit on unsold stock is labelled unrealised and never enters net profit.
- When a property/project/agent filter is active — or the Agent role is in use — company
  overheads are not attributed to that slice; the screen says so.
- Every export (CSV / Excel / PDF) carries company name, report name, period and
  generated date (§29).

---

## Known limitations

- The demo data set only trades from January 2026, so a full-year view has no comparable
  prior year and year-on-year deltas correctly show as unavailable.
- Data lives in memory: records you add persist until the page is reloaded. That is
  deliberate for a demo — reload to get a clean data set.
- "Excel" export is an HTML-table `.xls`, which Excel and LibreOffice both open. PDF uses
  the browser's print dialogue.
- The Zakat base is illustrative and **must be confirmed with the client's accountant**
  (AUDIT A10).
