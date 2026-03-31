# AR Reconciliation Analysis

> Matching customer ledger data against company books to surface invoice mismatches, deductions, and open balances — with a reusable Power Query pipeline.

**Tools:** Excel · Power Query · SQL · Power BI  |  **Domain:** B2B FMCG · Accounts Receivable

---

## Problem

In B2B trade — especially with large retail chains and aggregator platforms — a supplier's AR ledger and the customer's payment records rarely match cleanly. Customers deduct freight, raise short-supply claims, apply unapproved credit notes, or split invoices across payment runs. Without a structured reconciliation process, these gaps compound month over month, eroding collections efficiency and making it impossible to dispute deductions before they age out.

This project builds a repeatable reconciliation workflow that compares invoice-level data from both sides, classifies every line as matched, partially settled, deducted, or unmatched, and produces a clean exception report ready for the collections team.

---

## Scope & Scale

| Metric | Value |
|--------|-------|
| Invoice lines per cycle | 200–400 |
| Reconciliation cadence | Monthly |
| Mismatch categories tracked | 4 (Matched / Partial / Deduction / Unmatched) |
| Manual matching time reduction | ~75% |

---

## Data Sources

**A — Company AR ledger** (Tally ERP export → Excel)
- Columns: Vch No. (renamed `ClearingDoc`), Party Name, Debit, Credit, Balance
- Contains invoices raised, credit notes, and payment receipts from company books

**B — Customer payment statement** (vendor portal download or customer-provided Excel/CSV)
- Columns: Customer Ref No., Invoice Ref, Amount Paid, Deduction Amount, Deduction Reason
- Represents what the customer has actually paid and what they have deducted

---

## Reconciliation Process

### Step 1 — Data cleaning (Power Query)
Strip leading/trailing spaces from invoice numbers, standardise date formats, remove duplicate rows, convert debit/credit columns to signed numeric values.

> Key challenge: invoice numbers from portals often carry prefixes ("INV-" vs bare number). A custom M column strips and normalises these before any join.

### Step 2 — Invoice-level merge
Left join the company ledger to the customer statement on the normalised invoice number. Rows with no match on the customer side surface immediately as unmatched.

```
Table.NestedJoin(CompanyLedger, "ClearingDoc", CustomerStatement, "InvoiceRef", "CustomerData", JoinKind.Left)
```

### Step 3 — Variance calculation & classification

For each matched row: `Variance = Invoice Amount − Amount Paid`

| Status | Condition |
|--------|-----------|
| ✅ Matched | Variance = 0 |
| ⚠️ Partial | 0 < variance ≤ invoice amount, no deduction code |
| 🔵 Deduction | Customer has provided a deduction reason code |
| ❌ Unmatched | No customer record found for this invoice |

### Step 4 — Deduction analysis
Group deducted rows by reason code (freight, short supply, quality, scheme discount). Short-supply deductions are cross-referenced against GRN quantities vs invoice quantities and flagged separately for operations review.

### Step 5 — Aging overlay
Calculate days outstanding (`Today − Invoice Date`) and bucket into 0–30 / 31–60 / 61–90 / 90+ day bands. Unmatched and partial rows in the 60+ bucket are escalated in the output report.

### Step 6 — Summary output
Pivot classified data into a one-page reconciliation summary: total matched, total short-paid, deductions by type, and total unmatched. Exported as formatted Excel and optionally loaded into Power BI.

---

## Sample Output

| Invoice No. | Invoice Date | Invoice Amt (₹) | Amt Received (₹) | Variance (₹) | Deduction Reason | Days Outstanding | Status |
|-------------|--------------|-----------------|------------------|--------------|------------------|-----------------|--------|
| INV-BLR-01 | 02-Jan-2025 | 48,500 | 48,500 | — | — | 12 | ✅ Matched |
| INV-BLR-02 | 08-Jan-2025 | 62,000 | 57,200 | 4,800 | Short supply (GRN mismatch) | 18 | 🔵 Deduction |
| INV-BLR-03 | 15-Jan-2025 | 31,750 | 29,000 | 2,750 | Freight deduction | 25 | 🔵 Deduction |
| INV-BLR-04 | 22-Jan-2025 | 75,000 | 70,500 | 4,500 | — | 32 | ⚠️ Partial |
| INV-BLR-05 | 29-Jan-2025 | 44,200 | — | 44,200 | — | 67 | ❌ Unmatched |
| INV-BLR-06 | 05-Feb-2025 | 88,000 | 88,000 | — | — | 8 | ✅ Matched |

*Sample data — illustrative of structure and classification logic. Actual volumes: 200–400 rows per cycle.*

---

## Output Files

| File | Description |
|------|-------------|
| `AR_Reconciliation.xlsx` | 3 tabs: cleaned data · classified line items · summary pivot with aging |
| `AR_Dashboard.pbix` | Power BI file connected to Excel output — aging trends, deduction breakdown |
| `AR_Reconciliation.sql` | SQL equivalent — LEFT JOIN + CASE classification + GROUP BY aging bucket |
| `Reconciliation_Template.md` | Reusable prompt template for ad-hoc customer dispute reconciliation |

---

## Business Impact

Before this process, reconciliation was done manually by comparing PDF statements line by line — taking 2–3 days per customer per month. The Power Query pipeline reduces this to under an hour per cycle, with consistent classification logic that makes deduction disputes easier to escalate. Every line has a documented reason code and age, and the output feeds directly into the monthly collections review without reformatting.

---

## Skills Demonstrated

`Power Query (M)` `Table.NestedJoin` `Conditional column logic` `Aging bucket analysis` `Deduction classification` `Excel pivot tables` `SQL LEFT JOIN` `AR domain knowledge` `GRN vs invoice matching`
