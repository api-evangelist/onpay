---
name: Reconcile an OnPay pay run
description: Pull an approved OnPay pay run, page through its payroll checks, and reconcile wages, taxes, deductions and retirement contributions for accounting or 401(k) reporting.
api: openapi/onpay-api-openapi.json
generated: '2026-08-04'
method: generated
source: openapi/onpay-api-openapi.json
operations:
  - GET /reports/approved-pay-runs
  - GET /reports/listing
  - GET /reports/pagination/listing
  - GET /reports/retirement-summary
  - GET /company/pay-schedules
  - GET /company/pay-schedule-dates
  - GET /company
---

# Reconcile an OnPay pay run

> Identified by METHOD + path — the OnPay API v2 spec declares no `operationId` on any operation.

## Preconditions

- Partner credentials, `Authorization: Bearer {access_token}`, base URL `https://api.onpay.com/v2`.
- Role `Owner` or `Approver` for the reporting endpoints (`GET /company/pay-schedule-dates` also
  accepts `Controller` and `Manager`).
- You need the `company_id` from the OAuth token response — the reporting endpoints take it as a
  **required** query parameter.

## Steps

1. **Establish the calendar.** `GET /company/pay-schedules` returns each schedule with `frequency`,
   `check_date`, `period_start`, `period_end`, and `submission_deadline`.
   `GET /company/pay-schedule-dates?pay_schedule_id=&number_of_periods=&first_date=` projects the
   important dates forward.
2. **Find the run.** `GET /reports/approved-pay-runs?from=&to=` (optionally `runid`) returns
   `ApprovedPayRunsItem` records: `runid`, `check_date`, `period_start`/`period_end`,
   `tax_sweep_date` and `tax_sweep_amount`, `direct_deposit_sweep_date` and
   `direct_deposit_sweep_amount`, `total_wages`, `total_cash_requirement`. These sweep amounts are
   what you reconcile the bank debits against.
3. **Size the check listing before you pull it.** `GET /reports/pagination/listing` takes the same
   filters (`company_id`, `from`, `to`, `employee_ids`, `position_id`, `department_id`, `location_id`)
   and exists so you can plan the paging.
4. **Page the checks.** `GET /reports/listing` requires **`company_id`, `from`, `to`, `limit` and
   `start`** — all five are mandatory. Walk `start` forward by `limit` until the returned array is
   short. Each `PayrollCheck` carries `chkid`, `runid`, the employee identity block, and nested
   `wages[]`, `employee_taxes[]`, `employer_taxes[]`, `deductions[]`, `benefits[]`, `garnishments[]`
   and `leave_accruals[]` — every line has both a period `amount` and a `ytd`.
5. **Reconcile.** Sum the check-level `wages[].amount` for a `runid` and compare with that run's
   `total_wages`; sum employee + employer taxes and compare with `tax_sweep_amount`; sum net pay and
   compare with `direct_deposit_sweep_amount`.
6. **Retirement reporting.** `GET /reports/retirement-summary` (same required
   `company_id`/`from`/`to`/`limit`/`start` shape) returns per-employee
   `employee_deferral_amount`, `employee_pretax_deferral_amount`, `employee_roth_deferral_amount`,
   `employer_retirement_contribution`, `safe_harbor_*`, `simple_ira_amount`,
   `employee_401k_loan_amount` and `wages_in_retirement_calculation` — this is the feed for a 401(k)
   record-keeper, not the check listing.

## Rules that will bite you

- **Only these reporting endpoints paginate.** `limit`/`start` are required here and absent
  everywhere else in the API; `GET /employees` returns the entire roster unpaged.
- **`from`/`to` are required strings** on every report call — there is no "current period" default.
- **Read-only, but still rate-blind.** OnPay publishes no rate-limit policy and returns no
  `RateLimit`/`Retry-After` headers, so pace your paging conservatively.
- **Errors are the proprietary OnPay envelope**, not problem+json; a missing token surfaces as HTTP
  **400** with `error_code: 100`. See `errors/onpay-problem-types.yml`.
- **These payloads carry SSNs and bank routing/account numbers.** `PayrollCheck` includes `ssn`;
  employee bank accounts include `routing` and `account`. Do not persist or log them outside a system
  cleared for that data.
