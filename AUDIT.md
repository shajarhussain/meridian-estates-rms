# Requirements Audit — Reporting Module & CEO Dashboard

The requirements document was reviewed line by line **before** any of it was built.
This file records what was found: where the document contradicts itself, where it is
accounting-wrong, and what a working CEO dashboard needs that the document never
mentions. Every decision below is implemented in the code and traceable to a section
number in the source document.

Reviewed: 01 Sep 2026 · Source: `doc-text.txt` (§ numbers refer to it)

---

## A. Problems found in the document

### A1 — The Net Profit formula is stated two different ways
**Where:** §2.1 vs §37
§2.1 says `Net Profit = Gross Profit − Commission − Taxes − Zakat − Salaries − Office
Expenses − Bills − Other Expenses`. §37 adds **Marketing** and **Property Expenses**
to the same subtraction.

**Why it matters:** §2.1's list leaks two real cost categories. On this dataset that is
roughly PKR 2.4 Cr per year of spending that would silently never reach net profit — the
CEO would be shown a profit the company did not make.

**Decision:** §37's list is authoritative. All ten categories are subtracted.

---

### A2 — Zakat is treated as an operating expense, which contradicts §17
**Where:** §2.1 and §37 subtract Zakat in the same list as rent and salaries. §17
explicitly requires Zakat to be "kept separate from normal operating expenses".

**Why it matters:** Zakat is an appropriation of profit, not a cost of running the
business. Burying it in operating costs makes the operating margin look worse than it is
and makes the company impossible to compare with any other business.

**Decision:** profit is computed in three stages instead of one:

```
Sales revenue − cost of property sold      = Gross profit
Gross profit − operating costs             = Operating profit
Operating profit − tax − Zakat             = Net profit
```

Operating costs = commission, salaries, office, marketing, property, employee, bills,
other. Tax and Zakat sit below the operating line, and the waterfall on the dashboard
shows all three stages, so §17 is honoured while Zakat still appears in the CEO's
overall summary exactly as §17 also requires.

---

### A3 — Gross Profit is defined in a way that produces wrong answers
**Where:** §2.1 says `Gross Profit = Total Selling Revenue − Total Purchase Cost` for the
period. §7 and §25 say property profit is `Selling Price − that property's Total Cost`.

**Why it matters:** these only agree if you buy and sell the same properties in the same
period, which never happens in real estate — stock is bought one year and sold the next.
Under §2.1, a month where the company buys heavily and sells nothing reports a **loss**,
and a month where it sells old stock and buys nothing reports a **windfall**. Neither is
true. The correct measure is cost of goods sold: the cost of the properties actually sold.

**Decision:** both are computed and both are shown.
- `grossProfitCOGS` — revenue minus the cost of the properties actually sold. This is
  the accounting-correct figure and is the one the dashboard leads with.
- `grossProfitDoc` — the literal §2.1 figure, shown beside it so the client can see the
  document has been implemented and *why* the two differ.

This is the single most important finding in this review. It should be confirmed with the
client's accountant before the module goes to development.

---

### A4 — Tax is counted twice
**Where:** §7 records a per-sale tax as a selling expense. §16 records company tax paid.
§2.1 then subtracts "Taxes" from gross profit.

**Why it matters:** withholding tax on a sale is already inside that sale's net revenue.
Subtracting the §16 total on top of it charges the same rupee twice and understates net
profit.

**Decision:** the two are kept in separate buckets and never added together.
Transaction-level tax (withholding on sale) stays a selling cost inside net sale revenue.
The tax line in the profit waterfall is corporate/advance income tax only (§16). The Tax
report labels which is which.

---

### A5 — Potential Profit is presented as if it were profit
**Where:** §8 and §33 list "Potential Portfolio Profit" in the same KPI set as "Net Profit".

**Why it matters:** potential profit is an unrealised valuation estimate on stock nobody
has bought yet. Putting it next to banked profit invites the CEO to add them together.
On this dataset that would overstate the year by around PKR 14 Cr.

**Decision:** potential profit lives in the portfolio chapter, is labelled **unrealised
estimate**, and is never added into any profit figure. `profitSplit()` returns the two
separately with an explicit note.

---

### A6 — The expense taxonomy overlaps itself
**Where:** §2.1's "Total Office Expenses" example list includes Marketing and
Transportation. §13 then makes Marketing a top-level category of its own.

**Why it matters:** the same spend can land in two buckets, so the breakdown chart will
not sum to total expenses.

**Decision:** §13's five-group taxonomy (Office / Employee / Marketing / Property / Other)
is authoritative and mutually exclusive. §2.1's list is treated as illustrative only.

---

### A7 — A permission rule that cannot be implemented as written
**Where:** §30, Accountant: "Salary information can be restricted according to company policy."

**Why it matters:** "according to company policy" is not a specification. A developer
cannot build it and a tester cannot test it.

**Decision:** implemented as an explicit, visible policy switch. The Accountant role has
salary access denied by default, and the Permissions screen states the rule in plain
words so the client can confirm or change it in the demo.

---

### A8 — The suggested layout defeats the purpose of the dashboard
**Where:** §23 puts 5 KPI cards in row 1, 5 in row 2 and 4 in row 3 — **fourteen numbers
before a single chart**, with no stated order of importance.

**Why it matters:** this is exactly what made the first prototype unreadable. Dashboard
research is consistent here: a summary screen should carry 6–8 metrics, the single most
important figure should dominate, and everything else should be reachable rather than
present. Fourteen equal-weight numbers means the CEO reads none of them.

**Decision:** §23's *content* is all delivered, but as a sequence rather than a wall.
The overview is nine numbered chapters that answer one question each, in the order a
person actually asks them:

| # | Chapter | The question it answers |
|---|---------|--------------------------|
| 0 | The bottom line | Did we make money? |
| 1 | What we own | What is the company holding? |
| 2 | What we bought and sold | What moved this period? |
| 3 | What we earned | What did the trading make? |
| 4 | What we spent | Where did it go? |
| 5 | What is left | What is actually ours? |
| 6 | Money in and out | Do we have cash? |
| 7 | Who owes us, whom we owe | What is outstanding? |
| 8 | Who is performing | Which agents and properties? |
| 9 | What needs attention | What do I do today? |

Every figure in §23 and §33 is present; nothing was dropped. Detail moves one click away
(progressive disclosure) instead of onto the first screen.

---

### A9 — The date filters are missing the ones businesses use most
**Where:** §28 lists today, yesterday, this/last week, this/last month, this quarter,
this/last year, custom.

**Why it matters:** no rolling window. "Last 30 days" is the most-used filter in almost
every business report, and calendar months are useless on the 1st of a month — which is
the demo date. Opening on "This month" on 01 Sep shows an empty dashboard.

**Decision:** added **Last 7 / 30 / 90 days**. The dashboard defaults to a range with
data in it, and any range that genuinely has no activity now shows a plain-language empty
state instead of a screen of zeros.

---

### A10 — The Zakat basis is under-specified
**Where:** §17 asks for eligible assets, zakatable amount and rate but never states what
makes an asset zakatable, or that the rate is 2.5% on a lunar year.

**Why it matters:** in a real estate business, stock held for resale is zakatable but
property held for rental income generally is not. Getting this wrong is a religious and
legal exposure, not a rounding error.

**Decision:** implemented at 2.5% on a stated zakatable base, with the basis shown on
screen. **Flagged for the client's accountant to confirm before development** — this is
the one figure in the module that should not be signed off from a prototype.

---

## B. Flows the document never asked for, added because the report does not work without them

| # | Flow | Why it is necessary | Where |
|---|------|---------------------|-------|
| E1 | **Period-over-period comparison** on every KPI | The document specifies bare numbers. "PKR 2 Cr profit" cannot be judged without last period's figure beside it. Every KPI now carries a delta and the comparison base is named. | `compare()`, `prevRange()`, `deltaPct()` |
| E2 | **Receivables ageing buckets** | §19 gives "days overdue" per row, which cannot be summed or acted on. Buckets (not due / 1–30 / 31–60 / 61–90 / 90+) are what drives a collection call. | `aging()` |
| E3 | **Cash balance, opening and closing** | §18 reports inflow and outflow but never a balance, so the CEO cannot answer "how much money do we have?" — the most common question an owner asks. | `cashLedger()` |
| E4 | **Overhead run rate and months of cover** | Answers "if we sell nothing else, how long can we pay salaries and bills?" Profit does not answer this; cash divided by monthly overhead does. Break-even monthly sales included. | `runRate()` |
| E5 | **Realised vs unrealised split** | See A5. Makes the distinction structural rather than a footnote. | `profitSplit()` |
| E6 | **Per-property profitability maths** | §25 describes the screen but never the calculation. Implemented with acquisition costs, commission, tax and selling costs all attributed to the property. | `propertyPerf()` |
| E7 | **Drill-down provenance count** | §35 requires every figure to be clickable. It does not require telling the CEO *how many records* are behind the figure — which is what makes the number trustworthy rather than just a link. Every drill-down states its record count. | `sourceCount()` |
| E8 | **Empty and short-range states** | An empty period must say "no activity in this period", not show a screen of zeros that reads as a broken system. | UI |
| E9 | **Plain-language summary sentence** | The dashboard opens with one sentence in ordinary English stating what happened. First-glance comprehension is the whole point of an executive screen. | UI |
| E10 | **Crore / Lakh number format** | Pakistani real estate, the State Bank of Pakistan and the Bureau of Statistics all report in crore and lakh. Showing "PKR 125.0M" forces the reader to convert in their head on every figure. Default is Crore/Lakh; Million and full digits remain available. | `fmtParts()` |
| E11 | **Urdu sub-labels** | Every KPI and chapter carries an Urdu label beneath the English one, so a non-English-first reader can navigate the screen. | UI |
| E12 | **Scope honesty on filtered views** | When a property, project or agent filter is active, company overheads cannot be attributed to that slice. The result bar states this in words rather than silently reporting a misleading net profit. | `scopedToProperty()` |

---

## C. Requirements implemented exactly as written

§1–2 KPI set and all filters · §3–5 weekly / monthly / yearly profit with trend ·
§6–7 purchase and sale registers with every listed field · §8–9 inventory, cost value,
market value, potential profit · §10–12 commission ledger, agent-wise, company total ·
§13 five-group expense taxonomy · §14 salary register · §15 bills with overdue alert ·
§16 tax register · §17 Zakat register (see A2, A10) · §18 cash flow (see E3) ·
§19 receivables (see E2) · §20 payables with breakdown · §21 P&L for all five period
types · §22 all four charts · §23 all content, resequenced (see A8) · §24 top agents ·
§25 property performance · §26 transaction ledger with all fields · §27 payment methods
and accounts · §28 all date filters plus rolling windows (see A9) · §29 CSV / Excel /
PDF-Print with company name, report name, period, generated date · §30 four roles ·
§31 audit trail with void / reverse / adjustment and previous vs new amount ·
§32 separated financial buckets · §33 full KPI list · §34 summary with drill-down ·
§35 every figure clickable · §36 alerts · §37 core formula (see A1, A2).

---

## D. Open questions for the client

These cannot be answered from the document and should be settled in the demo meeting:

1. **A3 — cost basis for gross profit.** Cost of properties sold, or period purchase
   spend? This changes every profit figure in the system.
2. **A10 — Zakat basis.** Which assets are zakatable? Lunar or financial year?
3. **A4 — tax presentation.** Should withholding tax on a sale appear as a selling cost,
   a tax line, or both (shown separately)?
4. **A7 — salary visibility.** Should the Accountant role see salaries? Currently denied.
5. **Commission on unpaid sales.** Is commission earned on contract, or on collection?
   Currently earned on contract date, which is why commission can be outstanding on a
   sale that is itself unpaid.
6. **Multi-currency / inflation restatement.** Not in scope; confirm not needed.
