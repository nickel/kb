**Subject:** FIN-21 — straight-line fee amortisation: the auditor's materiality argument, plus one early-repayment defect it surfaced

Hi,

I know we already made a decision about this, but while working on capturing the daily interest accrual, I keep running into questions about whether interest accrual can only be done under the effective interest rate framework according to IFRS 9.

The 1 September decision — contractual interest, upfront fees amortised straight-line, approximating effective interest at loan-book level — is built. Interest income posts as `contractual interest + origination fee amortisation − broker fee amortisation`, exactly as relayed. Nothing has been originated yet, so this is a decision about the book we are about to write rather than one already on our balance sheet. That timing helps: everything below is cheap to change now and expensive to change later.

One thing the decision needs and does not yet have: the accounting authority behind it. I read the primary texts rather than working from memory, and the position is narrower than "an approximation of IFRS 9".

**IFRS 9 does not permit it.** Paragraph 5.4.1 says interest revenue *shall* be calculated using the effective interest method, applied to the gross carrying amount, with exactly two exceptions — purchased-or-originated credit-impaired assets, and assets that subsequently became credit-impaired. Both are credit-impairment cases. Neither is a materiality carve-out, and neither describes a performing personal loan. B5.4.1 requires fees integral to the rate to adjust the *rate* rather than be recognised separately; B5.4.4 governs the amortisation *period* when applying that method, so it is not authority for a straight-line substitute. The contractual-cash-flow fallback in Appendix A is available only "in those rare cases when it is not possible to reliably estimate the cash flows or the expected life", which a 24-month fixed-rate amortising loan with a published schedule is not. IFRS 9 contains no materiality provision at all.

So this is a **departure from IFRS 9 as written**, and the route that makes it available is IAS 8 rather than IFRS 9. That is a statement about the standard, not about whether the decision is wrong.

**IAS 8.8 asks two questions, and the second is the harder one.** It carves out policies whose effect is immaterial, and then adds that "it is inappropriate to make, or leave uncorrected, immaterial departures from IFRSs to achieve a particular presentation of an entity's financial position, financial performance or cash flows."

So the sign-off owes two answers:

1. **Will the effect be immaterial across this book?** It has to be established rather than assumed, and since there is no book yet it has to be argued from the planned product mix. On your own reference case — 100,000 SEK, 6% nominal, 24 level instalments, 44-day first period — the two bases differ by 205.38 SEK in the first period alone (723.29 contractual against 517.91 effective). Whether that aggregates to something immaterial depends on the launch book's projected fee-to-interest ratio and term profile, which is a finance judgement; nothing in our systems forecasts it.
2. **Does the departure clear IAS 8.8's second sentence?** As relayed, the motive is operational simplicity, not presentation. I think that is the right characterisation, and it is worth stating explicitly in the sign-off, because it is the sentence an auditor will read first.

**The scope is smaller than it was on 1 September.** The policy originally sat at entity level, which moved every interest-bearing product at once — including BNPL `pay_in_installments`, whose fee-to-interest mix nobody had assessed. It is now a per-product setting defaulting to the effective-interest method, and `personal_loan` is the only product flagged. Interest-bearing BNPL is back on IFRS 9's method. That makes the population you have to argue immateriality over both smaller and far more homogeneous. Zero-rate BNPL loans, which carry a fee and no interest, are not a departure at all and should not be part of the argument.

**Because nothing is originated, rejection is cheap.** If the auditor signs the argument, we record the materiality basis and drop the unverified caveat from the decision record — no code changes. If not, we set one product row back to the effective-interest method and the engine is already built to do it. There is no restatement question and no migration, which there would be the moment the first loan is written on this basis: the recognition basis is frozen at origination by design. That is the main reason to get the answer before launch rather than after.

**Separately, one defect the same change introduced, which we should fix before we originate anything.** This is a consumer-rights issue rather than an IFRS one, but it came out of the same decision and part of it needs a finance answer, so it belongs in the same conversation.

Releasing the fee on time rather than on payment means the two sides of the deferred-fee position now drain on different triggers: the receivable drains when the consumer pays that instalment's fee portion, while the income releases on the instalment's due date. They can no longer stay in step mid-life. The opening fee is deferred whole at origination and billed on the first instalment, so after the first payment the income side runs ahead of the receivable by roughly the opening fee less one period's share.

Our exit calculator treats income ahead of receivable by more than 1.00 as an integrity violation and refuses to produce a settlement plan. Early repayment rolls its whole transaction back when that happens. In production that would mean a consumer asking to pay off early is refused — not warned, refused — whenever the nightly release job is behind. CCD II Article 29(1) says Member States "shall ensure that the consumer is at any time entitled to early repayment", so an internal batch lag cannot lawfully make that right unavailable. Our reviewers rated it the highest severity.

No consumer is affected today, because there is no live book. It is a defect to close before launch, and two decisions unblock it — neither of which is ours to take alone:

- **The operational fix** is to catch the release up to the payoff date before settling. It decides no accounting; it is purely a batch-lag fix. It is blocked because the operation that releases the fee is currently callable only by the nightly job's own system identity, and letting the early-repayment path call it needs a permission grant that compliance should review rather than one we quietly add.
- **The accounting question** is what an unreleased fee balance *means* on each remaining exit. Withdrawal is settled — CCD II Art 26(5) says the creditor "shall not be entitled to any other compensation", so the unreleased part is recognised and the position nets to zero. Write-off, early repayment and modification still refuse, deliberately, because the right treatment differs by exit and only withdrawal has an Article that answers it. On a write-off, for instance, the unreleased income is not contractually relieved, so contra-revenue looks wrong and a credit-loss treatment may be right — that is a call for finance, not engineering.

I would take the first of those now and the second alongside FIN-21, since both touch the same position.

One dating note for later years: IFRS 18 deletes IAS 1 for financial years beginning on or after 1 January 2027, so the materiality framing should be re-checked against IFRS 18 for the launch book's later reporting years. IAS 8 itself is unaffected.

Could you take the two IFRS questions to the auditor, and give me a view on the exit treatment above? I can put the working, the reference-case figures and a worked example of the exit case into a one-pager if that helps either conversation.

Thanks,
Juan
