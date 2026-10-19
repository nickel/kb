# Finance Decision Master: review plan

CMC Sweden AB · Prepared 16 October 2026 · Covers the 64 open rows (61 in the sheet plus FIN-90, FIN-91 and FIN-92, new on 15 October) and the 10 decided rows still flagged DEVIATION (FIN-40 and FIN-43 in Session 0, eight more in Session 9).

## How to run it

- **One audience per session.** Each session brings together the people who own its rows (CFO, controller, IFRS specialist, Risk), so nobody sits through topics they cannot decide.
- **Upstream before downstream.** Within a session, decide the row others depend on first: the prepayment assumption before the rate-reset rows, the definition of default before the loss provision, the sub-ledger boundary before the close rows.
- **Prepare from the sheet.** Filter the Finance Decision Master to the session's rows; each row's doc carries the question, the options and what the platform does today.
- **Record the same day.** Change the sheet first, then the doc, then the engineering decisions register.

## Overview

| # | Session | Who | Rows | Count |
|---|---|---|---|---|
| 0 | Quick confirmations | CFO, Juan Gallego | FIN-64, 32, 31, 35, 49, 78, 40, 43 | 8 |
| 1 | Dated and launch gates | CFO, controller | FIN-85, 9, 8, 1, 52, 50 | 6 |
| 2 | Classification and the effective rate | IFRS specialist, auditor, CFO | FIN-2, 89, 27, 84, 39, 29, 83, 6 | 8 |
| 3 | Changes during a loan's life | Controller, IFRS specialist | FIN-45, 46, 61, 71, 87, 73 | 6 |
| 4 | Broker commission and fees | CFO | FIN-22, 72, 76, 26, 90 | 5 |
| 5 | Early exit, withdrawal and refunds | CFO, Compliance | FIN-23, 25, 42, 53, 69, 91 | 6 |
| 6 | Credit risk and loss provision | CFO, Jonas Adolfsson (Risk) | FIN-63, 75, 3, 86, 92, 16, 24, 65, 66, 10 | 10 |
| 7 | Ledger, close and chart of accounts | Controller, CFO | FIN-7, 17, 30, 51, 67, 81, 11, 77, 4, 18, 70, 33 | 12 |
| 8 | Treasury, tax and governance | CFO, tax adviser, Compliance | FIN-5, 44, 68, 34, 36 | 5 |
| 9 | Deviations sweep | Juan Gallego, CFO | FIN-13, 19, 37, 41, 47, 62, 74, 88 | 8 |

Sessions 1, 2 and 6 matter most: together they cover the launch gates, the income-recognition foundation and the only live conflict (FIN-86). Sessions 3 to 5 are mostly confirming or adjusting defaults the platform already runs.

## Session 0. Quick confirmations (30 minutes)

Each row needs one answer: confirm or reject what is already true or already built.

| Row | Question | Answer needed |
|---|---|---|
| FIN-64 | Withdrawal fee overcharge remediation | Confirm no real consumer was affected (the window ran only on the demo deploy), so no remediation is owed. |
| FIN-32 | Modification remeasurement on straight-line loans | Confirm it is moot now that FIN-21 settled on effective interest; only demo books hold straight-line loans. |
| FIN-31 | P&L line for the early repayment difference | Ratify or reject: the difference posts to interest revenue. |
| FIN-35 | Unpaid late fee at loan exit | Ratify or reject: an unpaid late fee is booked as cash received on exit. |
| FIN-49 | GL accounts for Stage 3 late-fee suspense | Approve GL 1719, 1711 and 1718, chosen by engineering. |
| FIN-78 | Withdrawal receivable ageing and segment | Confirm engineering's reading of FIN-15: days past due from the repay-by day, the loan's own segment. |
| FIN-40 | Daily release of deferred fees | Ratify the daily release engineering decided on 22 September. |
| FIN-43 | Withdrawal repayment allocation order | Confirm capital first, then interest. |

## Session 1. Dated and launch gates

| Row | Question | Why now |
|---|---|---|
| FIN-85 | Liquidity loan extract column definitions | First return is as of 31 October 2026. |
| FIN-9 | Real-money dress rehearsal sign-off | Pre-launch gate; no real money has moved yet. |
| FIN-8 | Pilot funding timing and liquidity | Needed before go-live. |
| FIN-1 | Upfront fees on early loan exit | Marked launch-blocking in the register. |
| FIN-52 | FINREP brought-forward profit scope | Marked pre-launch in the register. |
| FIN-50 | First financial year and year boundary | Sets the SIE export period and retention years. |

## Session 2. Classification and the effective rate

The foundation for every income-recognition row. Decide in this order.

| Order | Row | Question |
|---|---|---|
| 1 | FIN-2 | Classification methodology, and whether the IFRS specialist countersigns the personal loan |
| 2 | FIN-89 | Classification of pay in instalments and pay later (before either is originated) |
| 3 | FIN-27 | Prepayment assumption: the three questions the CFO left open on 15 October |
| 4 | FIN-84 | Initial recognition date: creation or payout |
| 5 | FIN-39 | Per-instalment fee in or out of the effective rate |
| 6 | FIN-29 | Effective-interest income above the period payment |
| 7 | FIN-83 | Interest rounding per calendar month |
| 8 | FIN-6 | Interest cut-off, and the CFO's five follow-up questions on Stage 3 |

## Session 3. Changes during a loan's life

Each row re-applies the Session 2 answers after something changes the schedule.

| Row | Question |
|---|---|
| FIN-45 | Discount rate for a modification after a rate reset |
| FIN-46 | Overdue principal in a rate reset |
| FIN-61 | Interest on due dates rolled to a banking day |
| FIN-71 | Booked interest after a mid-period reduction |
| FIN-87 | Which day carries interest a replaced instalment never recognised |
| FIN-73 | Stage 3 adjustment coverage basis |

## Session 4. Broker commission and fees

Builds on FIN-27's expected life.

| Row | Question |
|---|---|
| FIN-22 | Broker commission rate against expected life, now that the Lendo agreement is signed |
| FIN-72 | Broker commission release when the curve falls |
| FIN-76 | Forfeited broker clawback credit |
| FIN-26 | Merchant fee on merchant-funded plans (with the external auditor) |
| FIN-90 | Backdated merchant fee reversal: net onto the next invoice or issue a credit note |

## Session 5. Early exit, withdrawal and refunds

FIN-1 is in Session 1.

| Row | Question |
|---|---|
| FIN-23 | Evidence of the cost reduction at early repayment (the register dates it November 2026) |
| FIN-25 | Sizing exit reversals dated at the event |
| FIN-42 | Withdrawal refund on a closed loan (the register dates it November 2026) |
| FIN-53 | Merchant refund relief as an IFRS 9 modification |
| FIN-69 | Held refund on a withdrawn loan |
| FIN-91 | Who absorbs a chargeback larger than the captured amount |

## Session 6. Credit risk and loss provision

Decide in this order: what default is, how a loan cures, when risk has increased, what is measured, then how it is booked.

| Order | Row | Question |
|---|---|---|
| 1 | FIN-63 | Default materiality threshold per entity |
| 2 | FIN-75 | Default cure probation and assessment |
| 3 | FIN-3 | Significant increase in credit risk and the macro overlay |
| 4 | FIN-86 | **CONFLICT.** Loss provision exposure basis: principal or carrying amount |
| 5 | FIN-92 | How the month-end loss provision is booked (depends on FIN-86) |
| 6 | FIN-16 | Days past due when the collection error is ours |
| 7 | FIN-24 | Recalibration reach for charged-off loans |
| 8 | FIN-65 | Accelerated termination: the loan and the books |
| 9 | FIN-66 | Write-off legal effect and debt sale |
| 10 | FIN-10 | Clawed-back payment on a kept promise to pay |

## Session 7. Ledger, close and chart of accounts

FIN-7 decides what the platform owns at close; the rest follows from it.

| Order | Row | Question |
|---|---|---|
| 1 | FIN-7 | Sub-ledger to general ledger boundary and close |
| 2 | FIN-17 | Interest accrual into a closed period |
| 3 | FIN-30 | Source reconciliation checks gating close |
| 4 | FIN-51 | Scheme-date SLAs per payment provider and merchant |
| 5 | FIN-67 | Bankgirot settlement to bank matching owner |
| 6 | FIN-81 | Which reversing journals are corrections (rättelse) |
| 7 | FIN-11 | Suspense routing for unbookable cash |
| 8 | FIN-77 | Reporting-date amounts after late postings |
| 9 | FIN-4 | Chart of accounts approval (Sweden): questions 10 to 13 and the countersignature |
| 10 | FIN-18 | FINREP datapoint mapping of the Swedish chart |
| 11 | FIN-70 | Branch local charts of accounts (France, Italy, Portugal) |
| 12 | FIN-33 | Chargebacks against refunds on settlement statements |

## Session 8. Treasury, tax and governance

FIN-34 needs outside tax advice: book the adviser early.

| Row | Question |
|---|---|
| FIN-5 | Match-funded books per branch |
| FIN-44 | Withdrawal repayment account per market |
| FIN-68 | Per-payment principal, interest and fee split |
| FIN-34 | VAT place of supply for branches |
| FIN-36 | Sign-off of audit-log count drops |

## Session 9. Deviations sweep

Already decided; confirm each build or paperwork item has an owner and a date.

| Row | Decided | Still owed |
|---|---|---|
| FIN-13 | Broker commission accounting treatment | Documents still carry the earlier treatment |
| FIN-19 | Personal loan expected life per term | Declared, not measured |
| FIN-37 | Past-due interest collected at early settlement | Not built; restating closed settlements still open |
| FIN-41 | Interest to the settlement date at early payoff | Not built |
| FIN-47 | Broker commission clawback terms and booking | Paperwork alignment |
| FIN-62 | Rebooking out-of-order withdrawal repayments | Closed-month and GL month-split paths not built |
| FIN-74 | Broker commission payable and payee | Paperwork alignment |
| FIN-88 | Who countersigns loss-provision parameter changes | Other approval flows keep the one-person Risk gap |

