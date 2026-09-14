# Exporting Subledgers

Check the file `./../../resources/finances/erp/export.se`.

## Validator Online

https://sietest.sie.se/


## What Is Missing

1. No entity-wide journals-with-legs range read. JournalEntryFinder#in_period_with_legs:171
   requires entry_types: and clamps to one page.
2. No voucher number. journal_entries has no series/verno column.
3. No org number anywhere — not a column, not a meta key, not in branch_registration.
   Already filed as prod-004 seam 4.
4. No Dr−Cr balance aggregate. LedgerAccount.balances_at signs by normal_balance; SIE wants
   raw Dr−Cr.
5. No SIE anything — SIE, CP437, IBM437 appear nowhere in the repo.
