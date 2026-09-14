Income recognition = the frozen classification on a loan that says which IFRS 9 basis turns its contractual cost (interest, fees, or neither) into revenue over time, and therefore which ledger legs post. Two registries and one promotion rule produce three stored values but four behavioral classes.

## The Registries

- **Product** (`products.income_recognition_policy`) — `INCOME_RECOGNITION_POLICIES = ["eir_integral", "straight_line_fee"]`, default `eir_integral`. The *permission* to use the fee basis for an interest-bearing product. No entity column, nothing inherits, no fallback.
- **Loan** (`loans.income_recognition`) — `INCOME_RECOGNITION_MODELS = ["zero_cost", "straight_line_fee", "eir_integral"]`. The *outcome*, in `IMMUTABLE_PRICING_FIELDS`, `allow_nil` transitional.
- **Resolution** (`CreateLoanOperation#resolve_income_recognition:663`): derive from cost composition, then promote `eir_integral` → `straight_line_fee` if `product_data.approximates_effective_interest?`.

## The Four Classes

1. eir_integral. Reached when interest > 0 and the product policy is eir_integral. Interest revenue is carrying × EIR, posted per installment on its due date, amount from period.interest_at_eir_cents (AmortisationScheduleCalculator). No separate fee revenue — the fee is folded into the rate.
2. straight_line_fee, fee-only shape. Reached when interest == 0 and fees > 0. No interest revenue, there is none to recognise. Fee revenue releases straight-line over fee_amortisation_periods.
3. straight_line_fee, promoted shape. Reached when interest > 0 but the product policy is straight_line_fee. Interest revenue is the contractual installment.interest_cents, posted per installment on its due date. Fee revenue releases straight-line, same as the fee-only shape.
4. zero_cost. Reached when there is neither interest nor fees. Nothing posts either side — recognise_deferred_fees returns early on a zero total.

## Definitions

- **`eir_integral`** — IFRS 9 effective interest method, amortised cost. One blended rate (`loan.eir`, solved by `EffectiveInterestRateCalculator`) carries interest *and* fees, so income is front-loaded as carrying declines and there is no fee leg at all. `Loan#eir_mode?` true.
- **`straight_line_fee`** — the two-part basis: the origination fee is deferred at creation and released evenly, while interest, *where there is any*, accrues on the contractual schedule rather than at EIR. Its claim is that straight-line *approximates* the effective interest method closely enough to be immaterial — hence the method name `approximates_effective_interest?`. `Loan#time_based_fee_amortisation?` true, `Loan.fee_amortising` scope.
- **`zero_cost`** — 0% and no fees. Present so the registry is exhaustive and `eir_mode?` is false without a special case, not because anything posts.

## Ledger Legs

`straight_line_fee` / `zero_cost` at origination (`recognise_deferred_fees`, gated `unless loan.eir_mode?`):
```
DR loan:<id>:deferred_fee_receivable  CR loan:<id>:deferred_fee_income
entry_type: fee_recognition, value_date: loan.created_at
```
then per installment, nightly via `DailyOriginationFeeAmortisationJob` → `AmortiseOriginationFeeOperation`:
```
DR loan:<id>:deferred_fee_income  CR platform:origination_fee_revenue
entry_type: fee_amortisation, value_date: installment.due_date, fee VAT tax_code
```

Interest, both `eir_integral` and promoted `straight_line_fee`:
```
DR loan:<id>:loan_interest  CR platform:interest_revenue
entry_type: interest_accrual, value_date: installment.due_date, exempt VAT tax_code
```

## Two Things to Keep Straight

Recognition basis is **not** the accrual method. `interest_accrual_method` (`eir` / deferred `simple` / `flat`) is the jurisdiction axis; income recognition is the product axis. A loan resolves both.

And `straight_line_fee` ≠ "no interest" — the promoted shape is the trap I fell into earlier. `Loan#interest_bearing?` (`:485`) is the right predicate for "does interest exist", deliberately independent of `eir_mode?`, because IFRS 9 5.4.3 discounts at the original EIR for every loan that persists one.

Open: ADR-019's materiality argument is UNVERIFIED, awaiting FIN-21.
