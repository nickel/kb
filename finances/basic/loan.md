# Loan

Grounded in this repo, not textbook.

## 1. A loan as data

`loans` row. The fields that carry money meaning:

| Column                            | What it is                                                                    |
| --------------------------------- | ----------------------------------------------------------------------------- |
| `amount_cents`                    | Gross credit. What the consumer's purchase/loan is worth.                     |
| `term_months`, `due_day_of_month` | How long, and when each instalment lands.                                     |
| `interest_rate`                   | Nominal contractual rate. The number in the contract.                         |
| `apr`                             | Consumer-facing cost rate. Regulatory.                                        |
| `total_cost_of_credit_cents`      | Everything the consumer pays above the credit amount.                         |
| `eir`                             | Accounting rate. Internal. Never shown to the consumer.                       |
| `income_recognition`              | `eir_integral` or `straight_line_fee`. Decides which accounting machine runs. |
| `day_count_convention`            | How a "year" is measured for the maths (actual/365 here, FIN-14).             |
| `fee_amortisation_periods`        | Frozen denominator for spreading fees.                                        |

`installments` rows are the schedule: dated amounts the consumer owes.

## 2. Money flows

Three directions, easy to confuse:

- **Out**: net advance. `net_advance = amount_cents - down_payment`. Cash you actually deployed on day 0.
- **In**: instalments from the consumer, over time.
- **Out again**: broker commission to Lendo, merchant fees. Cost of getting the loan.

Profit = money in minus money out. Everything below is about *when you're allowed to say you earned it*.

## 3. Fees, by who pays whom

| Fee                     | Direction                               | Where                                                     |
| ----------------------- | --------------------------------------- | --------------------------------------------------------- |
| Setup / origination fee | Consumer pays you                       | `pricing_schedules.setup_fee_pct` / `setup_fee_min_cents` |
| Instalment fee          | Consumer pays you, per instalment       | `installment_fee_pct`                                     |
| Broker commission       | **You pay Lendo**                       | `broker_fee_charges`                                      |
| Merchant fee            | Merchant pays you                       | `merchant_fee_charges`                                    |
| Late fee                | Consumer pays you, only if they're late | `late_fees`                                               |

Key split: the first four are **known at origination**. Late fees are not, they're contingent, so they stay out of every rate calculation.

## 4. Three rates, three questions

All three come out of the same solver (`lib/irr_solver.rb`), different inputs.

**Nominal rate** (`interest_rate`). "What does the contract say?" Fees excluded. Used to build the schedule.

**APR** (`apr`, `EuCcdAprCalculator`). "What does this actually cost the consumer?" Fees included. CCD II Annex III fixes the formula. Consumer-facing on the SECCI. Also feeds `CreditCapChecker`, so a wrong APR makes an offer unlawful, not just mis-disclosed.

**EIR** (`eir`, `EffectiveInterestRateCalculator`). "At what rate did I actually earn money?" IFRS 9. Fees you charged and fees you paid both fold into it. Never shown to anyone outside accounting.

APR and EIR look alike and are not the same number. Different cashflow sets, different time bases.

## 5. What EIR actually is, mechanically

It's an IRR. You have:
- day 0: you hand over `net_advance` (negative cashflow)
- day N1, N2...: consumer pays you instalments (positive)

EIR = the annual rate that makes the discounted sum of those payments equal exactly the net advance. NPV = 0. Newton-Raphson finds it.

```
net_advance = Σ payment_i / (1 + eir)^years_i
```

`EirTimeBasis` converts dates to `years_i` on the loan's day count convention. Same basis is reused when applying the rate, which is what keeps the numbers self-consistent.

Returns `0.0` when total payments ≤ net advance. No economic interest, and searching for a negative rate is pointless.

## 6. EIR through the lifecycle

**Origination** (`CreateLoanOperation`)
1. Price the loan. Fees resolved from the pricing schedule.
2. Build the schedule: instalment dates and amounts.
3. Compute APR from those cashflows. Check against the cap.
4. Compute EIR from net advance + receipts. Store in `loans.eir`.
5. Freeze everything. `IMMUTABLE_PRICING_FIELDS` blocks later edits. The config that produced the number must survive so an auditor can re-derive it.

**Servicing** (`AmortisationWalk`)

Per period:
```
interest_income = opening_carrying × eir × period_years
principal       = instalment_amount − interest_income
closing_carrying = opening_carrying − principal
```

Early periods: mostly interest. Late periods: mostly principal. Not because anyone decided that, it falls out of a shrinking balance times a constant rate.

Last period gets pinned so the balance closes at exactly zero. `AmortisationRoundingWash` hands back the stray öre.

**Exit**
- Paid off: carrying amount hit zero, done.
- Early repayment: remaining deferred fees drain (`ReverseDeferredFeesOperation`).
- Default / charge-off: accrual stops, a loss allowance carries it instead.
- Modification: IFRS 9 5.4.3 re-measurement. `Remeasurement` in the walk bumps the carrying amount mid-life instead of re-solving the rate.

## 7. The fork that matters

`loan.eir_mode?` reads `income_recognition`. Two entirely different accounting paths:

**`eir_integral`** (the default)

Fees are *inside* the rate. No separate fee accounting at all. `CreateLoanOperation` skips `recognise_deferred_fees` when `eir_mode?`. Interest recognised at EIR already contains the amortised fee. One number, one mechanism.

**`straight_line_fee`**

Interest accrues on the contractual schedule. Fees are held on a matched ledger pair at origination:
```
debit  loan:N:deferred_fee_receivable
credit loan:N:deferred_fee_income
```
and released a bit at a time by `DeferredFeeAllocation`. Entry types `fee_amortisation` (your income) and `broker_fee_amortisation` (your cost).

The reason for the split: you collected a 200 kr setup fee on day 0, but you didn't *earn* it on day 0. You earn it by servicing the loan for twelve months. Deferring it and releasing it monthly is what makes the P&L honest.

`DeferredFeeAllocation` has a subtlety worth understanding: `periods` **sizes** each share, but **days** release it. Interest accrues daily, so a share dropped whole on a due date would recognise nothing for a cutoff landing between two due dates. That's the PR you merged this morning.

## 8. Traps

- **EIR is not APR.** Different regime, different inputs, different consumer.
- **Never recompute EIR on a live loan.** It's frozen at origination. A modification re-measures the carrying amount; the rate stays.
- **`periods` is frozen too** (`loans.fee_amortisation_periods`, `broker_fee_charges.expected_life_months`). A product config change must not re-cut a running loan.
- **Both halves of the subtraction share one allocator.** Interest income nets your fee income against the broker cost. Two allocators would let those halves drift onto different period bases and nobody could reconcile the result.
- **Late fees never enter any rate.** Contingent, not known at origination.

All numbers below come from running this repo's own calculators (`RepaymentScheduleBuilder`, `EuCcdAprCalculator`, `EffectiveInterestRateCalculator`, `AmortisationWalk`). Nothing hand-computed.

## 9. The loan example

Swedish personal loan, amounts in öre.

```
amount_cents         2_000_000   (20,000 SEK)
term_months          12
interest_rate        12.0%       nominal, contractual
setup fee            49_500      (495 SEK)
down payment         none
day_count_convention actual_365
advance date         2026-01-15
first due            2026-02-15
```

### Step 1: build the contract schedule

What the consumer signs. Annuity, so principal+interest is level. The fee rides on top.

```
#   due           principal   interest      fee      TOTAL
1   2026-02-15       157267      20384    49500     227151
2   2026-03-15       160688      16963        0     177651
3   2026-04-15       160508      17143        0     177651
4   2026-05-15       162644      15007        0     177651
5   2026-06-15       163801      13850        0     177651
6   2026-07-15       165864      11787        0     177651
7   2026-08-15       167161      10490        0     177651
8   2026-09-15       168865       8786        0     177651
9   2026-10-15       170814       6837        0     177651
10  2026-11-15       172327       5324        0     177651
11  2026-12-15       174198       3453        0     177651
12  2027-01-15       175863       1792        0     177655
    TOTAL           2000000     131816    49500    2181316
```

Three things to notice:

**Instalment 1 is 227,151, the rest are 177,651.** The whole setup fee lands on line one. `RepaymentScheduleBuilder#fee_split` does `fees[0] += @opening_fee`. Not spread.

**Period 3 interest (17,143) is *higher* than period 2 (16,963)** even though the balance shrank. Because Feb 15 → Mar 15 is 28 days and Mar 15 → Apr 15 is 31 days. That's `actual_365` doing its job. Under `nominal_monthly` this could not happen.

**Last instalment is 177,655, four öre more.** `residual_final` policy. Rounding has to land somewhere and the final line absorbs it.

Total cost of credit: `2_181_316 - 2_000_000 = 181_316` öre (1,813.16 SEK).

### Step 2: the two rates

Same cashflow series. Two answers.

```
APR (CCD II Annex III):  18.00%
EIR (IFRS 9):            18.0845%
```

Both are IRRs over the same money. They differ because the **time bases differ**:

- APR uses Annex III Remark (c): whole months first, days only as residual. Feb 15 is `1/12` of a year.
- EIR uses `EirTimeBasis` on the loan's own convention. Feb 15 is `31/365 = 0.08493` years.

EIR exponents from the run:
```
[0.08493, 0.16164, 0.24658, 0.32877, 0.41370, 0.49589,
 0.58082, 0.66575, 0.74795, 0.83288, 0.91507, 1.00000]
```

A 31-day first period is shorter than 1/12 of a year, so the same money arrives "sooner" on the EIR basis, so the solved rate is higher. Nominal was 12%. EIR is 18.08%. The gap is the fee.

### Step 3: the EIR walk (what accounting recognises)

Per period: `interest = opening × ((1 + eir)^period_years − 1)`, payment minus that is principal.

```
#   due               opening    payment     interest  principal      closing
1   2026-02-15        2000000     227151        28437     198714      1801286
2   2026-03-15        1801286     177651        23117     154534      1646752
3   2026-04-15        1646752     177651        23414     154237      1492515
4   2026-05-15        1492515     177651        20532     157119      1335396
5   2026-06-15        1335396     177651        18987     158664      1176732
6   2026-07-15        1176732     177651        16188     161463      1015269
7   2026-08-15        1015269     177651        14435     163216       852053
8   2026-09-15         852053     177651        12115     165536       686517
9   2026-10-15         686517     177651         9444     168207       518310
10  2026-11-15         518310     177651         7369     170282       348028
11  2026-12-15         348028     177651         4788     172863       175165
12  2027-01-15         175165     177655         2490     175165            0

sum interest  181316
sum principal 2000000
```

**The load-bearing line:**

```
interest at EIR      181316
contract interest    131816
setup fee           + 49500
                    = 181316   ✓
```

The fee has disappeared as a separate thing. It's *inside* the interest line, spread across twelve periods in proportion to the outstanding balance. That is what `eir_integral` means. No `deferred_fee` ledger accounts exist on this loan. `CreateLoanOperation` skips `recognise_deferred_fees` entirely when `eir_mode?`.

Closing balance ends at exactly 0. Not luck: `AmortisationWalk#pinned` forces the last period, and `AmortisationRoundingWash` shuffles stray öre back if the pin drives the last interest negative.

### Step 4: same loan, `straight_line_fee` instead

Interest accrues on the contract schedule. The fee sits on the deferred pair and releases `49500/12 = 4125` per period.

```
#     EIR income  contract int    fee rel     SL total     cumulative diff
1          28437        20384       4125        24509       +3928
2          23117        16963       4125        21088       +5957
3          23414        17143       4125        21268       +8103
4          20532        15007       4125        19132       +9503
5          18987        13850       4125        17975      +10515
6          16188        11787       4125        15912      +10791  ← peak
7          14435        10490       4125        14615      +10611
8          12115         8786       4125        12911       +9815
9           9444         6837       4125        10962       +8297
10          7369         5324       4125         9449       +6217
11          4788         3453       4125         7578       +3427
12          2490         1792       4125         5917           0

totals    181316                               181316
```

**Same lifetime income. Different monthly P&L.** At month 6 you'd have booked 10,791 öre (~108 SEK) more under EIR than under straight-line. Converges to zero at maturity because the same money is the same money.

EIR front-loads because the fee is earned in proportion to the balance you have out, which is largest early. Straight-line is flat by definition.

On a book of 10,000 loans that's not 108 SEK, it's a visible number in the monthly accounts. Which is why `income_recognition` is in `IMMUTABLE_PRICING_FIELDS`. Flipping a live loan's mode would restate history.

### Step 5: the broker fee, which is not in any of this

Say Lendo took 400 SEK for the introduction. Textbook IFRS 9 calls that an incremental transaction cost and folds it into the net advance, raising the EIR.

**This repo does not.** `EirPersister` computes:

```ruby
net_advance = @loan.amount_cents - down_payment&.amount_cents.to_i
```

`amount_cents` only. The commission lives on `broker_fee_charges`, capitalises separately, and amortises straight-line as a *cost* via `AmortiseBrokerFeeOperation`:

```
debit  Platform:amortised_acquisition_fees
credit Loan:deferred_fee_receivable
```

Both directions share `DeferredFeeAllocation`, deliberately. Your fee income and your broker cost are the two halves of one subtraction, and two allocators would let them drift onto different period bases.

### Step 6: things that would change these numbers

| Event                      | What moves                                                                                                                                                                                               |
| -------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Consumer pays late         | Nothing here. A `late_fee` row appears, outside every rate.                                                                                                                                              |
| Term extension             | Old instalments go `rescheduled`, new ones appended above the old numbers. `DeferredFeeAllocation` keys on **position**, not `installment.number`, or it would release nothing for the rest of the loan. |
| Payment holiday            | `Remeasurement` in the walk. Carrying amount adjusts on a date; `eir` does not change. IFRS 9 5.4.3.                                                                                                     |
| Early repayment at month 6 | Carrying is 1,015,269. Remaining deferred fee drains via `ReverseDeferredFeesOperation`. Under EIR mode the walk simply stops.                                                                           |
| Default                    | Accrual stops at `servicing?`. A loss allowance carries the balance instead.                                                                                                                             |

### The one-sentence version

The contract says the consumer owes 131,816 öre of interest plus a 49,500 öre fee. Accounting says you earned 181,316 öre of interest and no fee at all. Same money, and EIR is the rate that makes the second statement arithmetically true.
