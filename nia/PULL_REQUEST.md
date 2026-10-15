# Count daily repayment steps in working days when enabled

**Branch:** `nia/twice-weekly-repayments` → base: Apache Fineract `1.10.1`
**Commit:** 1 commit, 10 files, +467 / −15
**Author:** Nia IT <dylanhove4@gmail.com>
**Status:** ready for review. Unit tests and end-to-end tests pass on the 8444 test instance; not deployed to production.

## Description

Nia's 14 bi-weekly loan products must repay **twice a week on fixed day pairs, with Sunday off**:

| First repayment on | Repayments alternate |
|---|---|
| Monday / Thursday | Mon ↔ Thu |
| Tuesday / Friday | Tue ↔ Fri |
| Wednesday / Saturday | Wed ↔ Sat |

Stock 1.10.1 can't produce this. With Sunday as a non-working day, Fineract only moves a due date that lands on Sunday, and the next date is still counted from the unmoved date. A product repaying every 3 days therefore cycles through the whole week (Thu, Mon, Wed, Sat, Tue, Fri…) instead of alternating between two days.

### The change

New global configuration **`repayment-steps-in-working-days`**, disabled by default. When it is enabled, loans that repay **every N days** (and have no meeting calendar) count those N days in **working days** taken from Admin → Working Days. Every 3 working days with Monday to Saturday working gives the pairs above.

- `WorkingDaysUtil`: new `plusWorkingDays`, `minusWorkingDays`, `nextWorkingDayOnOrAfter`.
- `DefaultScheduledDateGenerator`: next due date and ideal disbursement date step in working days (`isRepaymentStepInWorkingDays`).
- `LoanScheduleAssembler.deriveFirstRepaymentDate`: the first repayment uses the same rule, and the minimum-days date is moved to a working day. The schedule shown at application is therefore the same one kept after disbursement.
- `HolidayDetailDTO`: carries the flag. It's set in `LoanScheduleAssembler` and `LoanUtilService`, because the date generator is also created with `new` outside Spring (holiday job, reschedule services).
- `ConfigurationDomainService(Jpa)`: `isRepaymentStepsInWorkingDaysEnabled()`.
- Liquibase `0141_add_configuration_repayment_steps_in_working_days.xml`: adds the setting row, disabled.

**Not affected:** weekly, monthly and yearly products, loans linked to a meeting calendar, and everything else when the setting is off (identical to stock 1.10.1).

## Tests

**Unit tests:** new `DefaultScheduledDateGeneratorWorkingDayStepsTest` (20 tests):
- every disbursement day from Monday to Saturday gives its pair
- the application schedule equals the schedule regenerated after disbursement
- Thursday disbursement gives Mon, Thu, Mon…
- with the setting off, the old behaviour is unchanged
- a 7-day working week keeps plain days, and weekly products are unaffected
- holidays of both types ("next repayment date" and "specific date")
- ideal disbursement date and the working-day helpers

All existing unit tests in `fineract-core`, `fineract-loan` and `fineract-provider` pass: 760 tests, 0 failures. Spotless (Java) and Checkstyle pass.

**End-to-end on a 1.10.1 test instance** with real loans (submit, approve, disburse, repay):
- 14 bi-weekly products converted to every 3 days; Hilton and the 15 weekly products unchanged
- disbursed Mon to Sat: 10 repayments each, correct pair, application schedule equal to the disbursed one
- product with a 7-day minimum: first repayment on the same weekday a week later, then the pair
- disbursed a day later than expected: schedule regenerated on the new pair
- a repayment doesn't move any due date
- disbursement on Sunday is rejected
- setting off gives stock behaviour; setting on again gives the pairs
- holidays: a payment on a holiday moves to the next date in its pair and later payments shift; nothing is doubled; the holiday job applies the same to existing loans

## How to apply

**From this repository:** merge the pull request from `nia/twice-weekly-repayments` into `main`.

**Onto another Fineract 1.10.1 checkout** (for example the server's source):

```bash
git fetch https://github.com/Tashinga4/nia-fineract-1.10.1.git nia/twice-weekly-repayments
git cherry-pick FETCH_HEAD
```

Or download the commit as a patch (`.../commit/<id>.patch` on GitHub) and apply it with `git am`.

## Deployment

1. Back up the database and keep the current `fineract-provider` build/image.
2. Deploy the new build. The setting is off, so behaviour is unchanged.
3. Enable **Admin → System → Configurations → `repayment-steps-in-working-days`**.
4. Change one bi-weekly product to repay every **3 Days** and book a test loan. Then convert the rest:
   HP4B, 25wb, W6WB, H6BW, WC4B, 2W5B, G4WB, G5WB, SFBW, 2W6B, 5G5B, 2HB4, 2H8B, 5H8B. **Do not convert Hilton (HBW4).**
5. Loan term = 3 × number of repayments, in days (for example 24 days for 8 repayments).

Working days need no change if they are already Monday to Saturday with "move to next working day".

**Rollback:** disable the setting (new schedules revert immediately), or redeploy the previous build. The setting row can stay.

## Reviewer notes

- Existing active loans keep their stored schedules. Only new loans on converted products use the new rhythm.
- Holidays: use **"Reschedule to next repayment date"**. Activate the holiday, then immediately run **Scheduler Jobs → "Apply Holidays To Loans"**. In stock 1.10.1, a loan booked between activation and the job ends up with a payment on the holiday (reproduced on weekly products too; not caused by this change).
- Found during testing, stock 1.10.1 and not fixed here: products with a **down payment** generate fewer repayments than configured (e.g. 6 of 8), which lowers "whole term" interest (15.00 instead of 20.00 on 100). Affects weekly products too, so worth checking in production.
- "Per month" interest products (WC4B, 2H8B) charge less once the term is shorter (WC4B 36.92 → 15.78 on 100). That's a business decision before conversion.
- Products with a minimum gap of 7 days (G4WB, G5WB, 5G5B) get their first repayment a week after disbursement.
