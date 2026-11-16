# Reference: Logic Carried Over from the Accounting Product

Bahi re-implements, in Go, the money and ledger rules first written in Hardik's (private, separate) TypeScript
accounting product. **No code or data is copied**; this note records the *rules* so they can be rebuilt and tested here.
GST, GSTIN and bill-numbering rules belong to Munshi (P2) and are recorded in `munshi/docs/reference/`.

---

## 1. Money

| Rule in the accounting product | How Bahi applies it |
|---|---|
| Money is **integer paise**; never a float in calculations | `Amount{Minor int64, Asset string}`; INR has scale 2. Securities use the same type with scale 0 (see handbook `06-simulated-ecosystem.md`) |
| Parsing `"2,345.00"` / `"  2317.85"` strips spaces/commas; max 2 decimals; rejects anything else | Same parser for statement import (CSV recon); table-driven tests from the original cases |
| Converting a float from a source file: round half **away from zero**; never produce `-0` | Only at import boundaries (CSV/MT940 amounts); never inside the ledger |
| Indian digit grouping for display (`₹1,42,300.00`) | Only in reports/UI; the API always returns minor units |
| Round to rupee: 50 paise rounds **away from zero**, so a credit note cancels its bill exactly; difference posted to a *round-off* account | Bahi doesn't round; producers (Munshi) send explicit round-off lines. Bahi only enforces balance |

## 2. Chart of accounts (default template for an Indian trading SMB)

The accounting product's fixed accounts become Bahi's **default chart template** (`POST /v1/ledgers` with `template: "in-smb-trading"`):

| Code (suggested) | Name | Type |
|---|---|---|
| 1000 | Cash in hand | asset |
| 1010 | Bank account(s) | asset |
| 1200 | Stock | asset |
| 1310 / 1311 / 1312 | Input CGST / SGST / IGST (ITC) | asset |
| 1400:<party> | Sundry debtors (one sub-account per customer) | asset |
| 2100:<party> | Sundry creditors (one sub-account per supplier) | liability |
| 2310 / 2311 / 2312 | Output CGST / SGST / IGST | liability |
| 3000 | Capital and opening balances | equity |
| 4000 | Sales | income |
| 4010 | Sales returns (contra-income) | income |
| 4100 | Charges recovered from parties | income |
| 4200 | Discount received | income |
| 5000 | Cost of goods sold | expense |
| 5100 | Stock written off | expense |
| 5200 | Discount allowed | expense |
| 5900 | Round off | expense |
| 1999 | Suspense (new in Bahi: unidentified bank items during recon) | asset |

Notes:
- Parties are **sub-ledger accounts** (`party:<id>` in the original). Keep that idea: debtors/creditors control account + per-party sub-accounts.
- Tax and stock accounts were **not manually postable** in the original (only documents post to them). Bahi equivalent: an account flag `SYSTEM_ONLY`, so only entries carrying an `external_ref` from an authorised producer (e.g. Munshi) may touch them.

## 3. Posting rules (become test fixtures and Munshi's posting templates)

Each document's postings balance **on their own**. That is the original's core invariant, and it is Bahi's invariant #1.

| Document | Debit | Credit |
|---|---|---|
| Opening balances | Stock, Cash, Bank | Capital |
| Sales bill | Party (total) · Round-off (if negative) | Sales (taxable) · Output CGST/SGST/IGST · Round-off (if positive) |
| …cost side of the same bill | COGS | Stock (at cost) |
| Purchase | Stock (value) · Input CGST/SGST/IGST · Round-off (if diff > 0) | Party (total) · Round-off (if diff < 0) |
| Sales return (credit note) | Sales returns · Output CGST/SGST/IGST | Party |
| …good stock coming back | Stock (at cost) | COGS |
| Purchase return (debit note) | Party | Stock · Input CGST/SGST/IGST · Round-off as needed |
| Receipt from party | Cash/Bank · Discount allowed | Party (amount + discount) |
| Payment to party | Party (amount + discount) | Cash/Bank · Discount received |
| Cheque bounced | **reversal** of the receipt/payment on the bounce date | |
| Stock correction | Stock / Stock written off (depending on direction) | the other |
| Manual journal voucher | as entered (must balance) | |

Purchase tax split in the original: intra-state → CGST = ⌊tax/2⌋, SGST = tax − CGST (the odd paisa goes to SGST); inter-state → IGST = tax.

## 4. What changes in Bahi (and why: ADR material)

| Accounting product | Bahi | Why |
|---|---|---|
| Postings are **derived on the fly** from documents (`postings(state)` recomputes everything) | Entries are **stored, immutable**, with idempotency keys | Audit trail, concurrency, performance at scale; documents and ledger become separate systems (Munshi owns documents) |
| Cancelling a bill drops its postings (`if (b.cancelled) continue`) | Cancelling = **reversal entry** on the cancel date | History must never change; a cancelled bill in a closed period can't vanish from it |
| Cheque bounce = extra opposite postings | Same idea, formalised as `POST /entries/{id}/reverse` with its own effective date | Generic mechanism |
| `lockedUpto` setting blocks edits before a date | **Period close** (`periods` table, closed status) | Same concept, enforced in the database transaction |
| Trial balance / P&L / balance sheet computed by scanning all postings | Running totals on accounts + snapshots for point-in-time | Scale; and the hot-account trade-off becomes ADR-0002 |
| Single tenant in browser storage | Multi-tenant Postgres with RLS | Each simulated bank, depository and business is a tenant |
| Party with opposite balance "moves sides" in the balance sheet (advances) | Keep in reporting: classify by sign of the net balance at report time | Faithful reporting |

## 5. Test vectors to port

Port the original Vitest cases (money parsing/formatting/rounding) into Go table-driven tests in `internal/money/`, plus:
- Every row of §3 as a fixture entry that must balance and produce the expected trial balance.
- The original's cheque-bounce sequence as a reversal test.
Keep fixtures **synthetic** (no real party names, GSTINs or amounts).
