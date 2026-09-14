# ECL

## The Idea

You lend 10,000. You don't know who will default, but you know some will. ECL (Expected Credit Loss) says: don't wait to find out.
Book a loss reserve **today** for the slice of that money you statistically expect to lose, and update it every day as the picture changes.

Before IFRS 9, banks waited for evidence of trouble before booking a loss. That was too late and too lumpy — 2008 is the
reason the rule changed. ECL is loss recognition moved forward in time.

Important: the reserve is not money moving anywhere. It's a contra-asset called `loss_allowance` that sits against the loan
on the balance sheet. The loan says 10,000, the allowance says -165, so the balance sheet says the loan is really worth 9,835.
The other side of that entry is a P&L expense.

## Three numbers, multiplied

```ruby
new_provision = (ead * adjusted_pd * rates.lgd).round
```

That's the whole calculation, from `recalculate_provision_operation.rb:74`.

- **EAD** — Exposure At Default. How much is at risk. Here it's the loan's outstanding balance.
- **PD** — Probability of Default. What fraction of borrowers like this one won't pay.
- **LGD** — Loss Given Default. When one does default, what fraction do you actually lose after recoveries. Never 100%,
    because you chase the debt and get some back.

10,000 × 3% × 55% = **165**. That's your reserve on a healthy loan.

## The three stages are where it gets interesting

The same loan gets a wildly different reserve depending on which "stage" it's in.
Your repo's fallback curve (`pd_lgd_resolver.rb:30-33`, benchmarked against Klarna's 2023 annual report):

| Stage | Meaning                               | PD   | LGD | Reserve on 10,000 |
| ----- | ------------------------------------- | ---- | --- | ----------------- |
| 1     | Healthy                               | 3%   | 55% | **165**           |
| 2     | Risk has significantly increased      | 35%  | 55% | **1,925**         |
| 3     | Credit-impaired, default has happened | 100% | 60% | **6,000**         |

Same borrower, same balance, reserve moves 36x on stage alone. So the whole game is **which stage a loan is in**, and that is
what most of your open finance questions are actually about.

Stage 3 PD is 1.0 because default already happened — there's nothing left to predict, and LGD is the only free variable.

## Stage 2 is the hard one

Stage 1 is easy (new and fine). Stage 3 is easy (they've defaulted). Stage 2 means "meaningfully worse than when we lent,
but not yet broken", and nobody can define that objectively. This is **SICR** — Significant Increase in Credit Risk.

Your `Ifrs9StageResolver` triggers Stage 2 on: 30+ days past due, a risk grade in the Stage-2 band, forbearance, or a relative downgrade.
The 30-day rule is a rebuttable presumption — the code has `rebut_sicr_presumption_operation.rb` for arguing a specific loan out
of it, citing IFRS 9.5.5.11.

Two subtleties in your implementation worth knowing:

**The daily ticks disagree on purpose.** `resolve_for_arrears_tick` *ratchets* — stage can only go up. `resolve_for_migration_tick`
*recalculates* — stage can come back down after the loan cures. Two entry points, deliberately not merged.

**Coming back down is slow.** A cured Stage 2 loan serves three payments of "cure probation" before returning to Stage 1.
Releasing a reserve is releasing profit, so the standard makes you earn it.

## Where it lives in the code

`packs/lending/provisions/` — `RecalculateProvisionOperation` does the multiplication under a loan lock, `Ifrs9StageResolver`
decides the stage, `PdLgdResolver` looks up the rates by (segment, country, stage), `MacroOverlayResolver` applies the economy adjustment.

And `VerifyEclLedgerConsistencyOperation` checks the model and the ledger still agree — it runs daily and **blocks period close**
if it hasn't passed clean on every business day. That's the control that catches the classic failure: `loan.provision_amount_cents` and the `loss_allowance` ledger balance drifting apart.

## ECL Recalculation

In a example, every figure reconciles exactly:

|                     |                                                                    |
| ------------------- | ------------------------------------------------------------------ |
| Stage / calibration | 1, version 1.0, SE `personal_loan`                                 |
| PD × LGD            | 0.03 × 0.55 = **0.0165**                                           |
| EAD at origination  | 22,686,480 → ×0.0165 = **374,327** (the 2026-06-28 posting)        |
| EAD now             | 22,056,300 → ×0.0165 = **363,929** (`loan.provision_amount_cents`) |
| Delta               | **10,398**                                                         |

**Cause:** two installments were paid on 2026-08-28, totalling 630,180 cents. `OutstandingBalanceCalculator` (`packs/lending/app/public/queries/outstanding_balance_calculator.rb:26`) sums `amount_cents - paid_cents` over still-owed installments, so EAD fell by exactly that, and `RecalculateProvisionOperation:76` recomputed `(ead * pd * lgd).round`. Stage never moved — DPD 0 throughout, still stage 1. Pure EAD effect.

Two things worth knowing about what you're looking at:

**The 15-day lag is a seed artifact.** Payment landed 2026-08-28, release posted 2026-09-12. `created_at` and `posted_at` are both exactly `2026-09-12 00:00:00 UTC`, and there is no `DailyJobRun` row for 2026-09-12 at all — the last is 2026-09-11. So this row came from the seed's terminal pass stamped at "today", not from a nightly batch. Don't read the date as production behaviour. Separately though: `RecordInstallmentPaymentOperation` does not recalc the provision inline, so even in production the release waits for `DailyArrearsBatchJob` the following night.

**EAD includes unearned future interest, and that looks wrong.** `amount_cents` per installment is principal + interest, so EAD is the undiscounted contractual total: 22,686,480 against a principal of 17,500,000 — 29.6% higher than the loan's carrying amount. IFRS 9 measures ECL on the gross carrying amount (principal plus accrued-but-unpaid interest), as the present value of cash shortfalls discounted at the EIR; interest not yet earned is not an exposure. If that reading holds, every stage-1 allowance on an interest-bearing loan is overstated by roughly the unearned-interest ratio.
