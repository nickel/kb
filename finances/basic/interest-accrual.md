# Interest Accrual

**Interest accrual, as norrsken defines it today** (`packs/lending/servicing/app/public/operations/accrue_installment_interest_operation.rb`):

Recognition of interest income as revenue, per installment, on its due date — not at origination, not on the day batch runs.

## Mechanics

- **Trigger:** `daily_interest_accrual` recurring job (`config/recurring.yml:149`) → `DailyInterestAccrualJob` → op per loan. Loan must be `servicing`.
- **What's due:** `InterestAccrualMethod.for(interest_accrual_method:)` dispatches by jurisdiction; only `"eir"` registered, fails closed on anything else. `EirAccrualStrategy` picks installments `due_date <= Date.current` and splits two ways:
  - `loan.eir_mode?` → `period.interest_at_eir_cents` from `AmortisationScheduleCalculator` (IFRS 9 amortised cost)
  - else → `installment.interest_cents` (cash-basis / straight-line, fee-only zero-EIR loans)
- **Posting:** DR `loan:<id>:loan_interest`, CR `platform:interest_revenue`, `entry_type: "interest_accrual"`, `value_date: installment.due_date` (late batch still books last month in last month), `tax_code` from `VatResolver` — interest VAT-exempt, Art 135(1)(b) VAT Directive.
- **Idempotency:** one journal per installment, `JournalEntryFinder#exists_for_reference?("Installment", id, entry_type: "interest_accrual")`.
- **Atomicity:** all due installments of a loan or none (one transaction).
- **Closed period:** skipped permanently, returned as `closed_period_installment_ids`, warn-logged. Not a failure — raising made the job fail nightly forever (Sentry NORRSKEN-1) and withheld the loan's postable legs.
- **Day count:** product-level `day_count_convention`. `actual_365` decided platform-wide 2026-08-27, forward-only, **not rolled out** — live products still `nominal_monthly`.

## Not Settled (Don't State These as Defined)

- **Granularity.** Lumpy per-installment, so every month-end misstates revenue: earned-but-not-due interest unrecognised, in-period lumps carry prior-period interest. Fork is daily accrual vs period-end reversing cut-off — CFO leans daily. `docs/pending/finance-signoff/010-revenue-recognition-cutoff-and-stage3-basis.md`, blocked on finance.
- **Stage 3 basis.** Accrues gross, should be net-of-allowance or stop-and-disclose. Same item.
- **Late accrual into closed period.** `docs/pending/finance-signoff/20260823091545-interest-accrual-into-closed-period.md`, blocked on finance.

## Accruing Methods

Effect on revenue: interest income each period = carrying amount × EIR. Fees and interest fold into one rate, so no separate fee leg, and income is front-loaded as carrying declines.

Two separate "alternative" axes — don't conflate them:

1. Accrual method (jurisdiction). InterestAccrualMethod::METHODS, packs/lending/servicing/app/public/queries/interest_accrual_method.rb:
    - "eir" → EirAccrualStrategy. Only one registered. Every current entity/country resolves here.
    - "simple" → simple interest (US consumer/Reg Z, much of LatAm): rate × outstanding principal × days. Commented out, deferred.
    - "flat" → flat rate on original principal for the whole term, some LatAm products. Commented out, deferred.
2. Income recognition within the eir method. Derived at origination (PricingData#income_recognition), frozen on the loan, read by Loan#eir_mode? (packs/lending/app/models/loan.rb:491):
    - eir_integral (interest-bearing) → EIR path, amount from period.interest_at_eir_cents (AmortisationScheduleCalculator)
    - straight_line_fee (fee-only, 0% interest) → cash basis, installment.interest_cents, plus deferred-fee straight-line amortisation as origination_fee_revenue with fee VAT
    - zero_cost → nothing to recognise
