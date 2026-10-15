# Nia Fineract 1.10.1: twice-weekly repayments

Internal Nia Financial Services work on Apache Fineract 1.10.1, the backend behind the Mifos web-app.

**Status:** implemented and tested on a Fineract 1.10.1 test instance (unit tests and end-to-end tests pass). **Not deployed to production.**

## The problem

Nia's bi-weekly products (for example **20% WC 4 Weeks (Bi-Weekly)**) must repay **twice a week on fixed pairs of days, with Sunday off**:

| First repayment on | Repayments alternate |
|---|---|
| Monday / Thursday | Mon ↔ Thu |
| Tuesday / Friday | Tue ↔ Fri |
| Wednesday / Saturday | Wed ↔ Sat |

Stock Fineract 1.10.1 can't produce this:

- Set up as **weekly**, the products repay once a week, so a "4 Weeks (Bi-Weekly)" loan runs for 8 weeks.
- Set up as **every 3 days**, Fineract counts calendar days. A date that lands on Sunday is moved to Monday, but the next date is still counted from the Sunday, so the repayments wander through the whole week (Thu, Mon, Wed, Sat, Tue, Fri…).

## The fix

A new global setting, **`repayment-steps-in-working-days`** (off by default). When it is on, loans that repay **every N days** count those N days in **working days** (Admin → Working Days). With Monday to Saturday as working days, "every 3 days" gives exactly the pairs above, and a "4 Weeks" loan with 8 repayments really runs 4 weeks.

- The first repayment follows the same rule, so the schedule shown at application is the one kept after disbursement.
- Weekly and monthly products, loans linked to a meeting calendar, and everything else with the setting off behave exactly like stock 1.10.1.

Code change: one commit, 10 files, +467 / −15. See the pull request from `nia/twice-weekly-repayments` into `main`, and [PULL_REQUEST.md](PULL_REQUEST.md) for the file-by-file description.

## Tests

- **Unit tests:** `DefaultScheduledDateGeneratorWorkingDayStepsTest`, 20 tests, all pass. All existing unit tests in `fineract-core`, `fineract-loan` and `fineract-provider` pass (760 tests, 0 failures).
- **End-to-end** on a 1.10.1 test instance with real loans (submit, approve, disburse, repay): disbursement on every day from Monday to Saturday, a product with a 7-day minimum gap, late disbursement, repayments, setting off/on, and holidays. Scripts and results are in [`e2e/`](e2e/):
  - `e2e-run2.log`: schedule checks (all PASS). The error at the very end is the script stopping when it tried to re-create a test holiday that already existed; the holiday checks were then run separately.
  - `e2e-run2-holiday.log`: holiday checks, 8/8 PASS.
  - The scripts use Fineract's default test login and a local test URL. Don't point them at production.

## Repository layout

| Branch | Contents |
|---|---|
| `main` | Apache Fineract 1.10.1 source (tag `1.10.1`, without upstream history) and this `nia/` folder |
| `nia/twice-weekly-repayments` | `main` + the fix (pull request into `main`) |

## Deploying

See [release-notes.md](release-notes.md) and the "Deployment" section of [PULL_REQUEST.md](PULL_REQUEST.md). In short: back up the database, deploy the build (the setting is off, so nothing changes), enable the setting, convert one bi-weekly product to "every 3 days" and test it, then convert the rest. **Do not convert Hilton (HBW4).**

**Rollback:** turn the setting off, or redeploy the previous build.

## Other Fineract versions

The live server runs an older Fineract version. A port of this fix to **Fineract 1.9.0** is being built and tested separately and will be published as its own repository.
