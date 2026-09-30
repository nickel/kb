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

## Messages on the UI removed

- Subledger reporting view: non-authoritative; books of record live in the GL. The statutory account numbers come from a mapping that is still awaiting Finance sign-off, and the file covers this subledger only, so it is not the legal entity's books.
- One verification per event type and day, with transaction lines summed per statutory account. Every journal of the same entry_type value-dated on the same day becomes one #VER, and each account it touches becomes one #TRANS carrying the day's net movement. Bokföringslagen (1999:1078) 5 kap. 6 § permits this common verification for several similar business events; 5 kap. 7 § requires that the link from it back to each individual event stay establishable, which is what the verification report below is for.
- Every figure in the file is the window's movement, never a position. #IB is written as zero, #UB carries the window's net movement per balance sheet account, and #RES the window's movement on the result accounts, so opening plus the verifications shown still equals closing and the file states the delta for the day or range and nothing cumulative. The chart declares only the accounts the window moved. #RAR remains the full calendar year, which is why a window crossing a year end is refused.
- This departs from SIE, which defines #IB and #UB as the financial year's opening and closing balances. A reader that imports them as balances will take the window's movement for the account's position, so the file suits a system that posts the verifications and computes its own balances, not one that trusts the balance records.
- Three things it does not carry, so it is not a complete type 4 export (arch-20260912095439): no comparative-year records, only the year the window falls in; no period-close or reopen vouchers, which are local previews the GL performs itself; and the year is taken as the calendar year, which is wrong for an entity on a broken or extended financial year.
