---
name: Onboard an employee in OnPay
description: Create a new employee in an OnPay company with wages, deductions, direct-deposit accounts, tax fields, and a worksite, using the partner-only OnPay API v2.
api: openapi/onpay-api-openapi.json
generated: '2026-08-04'
method: generated
source: openapi/onpay-api-openapi.json + https://onpay.readme.io/reference/authorization
operations:
  - POST /employees
  - PUT /employees/{employee_id}/bank-accounts
  - PATCH /employees/{employee_id}/wages
  - PATCH /employees/{employee_id}/tax-fields
  - PATCH /employees/{employee_id}/tax-fields/worksite
  - GET /employees/{employee_id}
  - GET /company/worksites
  - GET /company/departments
  - GET /company/positions
---

# Onboard an employee in OnPay

> The OnPay API has no `operationId` on any operation, so every step below is identified by
> METHOD + path exactly as it appears in `openapi/onpay-api-openapi.json`.

## Before you start

- **Access is partner-only.** OnPay grants API credentials to approved partners; there is no
  self-serve key. See `https://onpay.readme.io/reference/authorization`.
- **Base URL:** `https://api.onpay.com/v2`. Do **not** use the `onpaydev.com` host in the spec's
  `servers[]` — that domain is parked (see `sandbox/onpay-sandbox.yml`).
- **Auth:** OAuth 2.0 authorization code. Send `Authorization: Bearer {access_token}` on every call.
  Access tokens last **7200 seconds (2 hours)**; the refresh token is **single-use** and each exchange
  returns a new access/refresh pair. Store the new refresh token or you lose the connection.
- **One token = one company.** The token response carries `company_id`; every path below is scoped to
  that company.
- **Scope:** employee writes require the `Owner` or `Approver` role. Worksite/department reads also
  accept `Controller` and `Manager`.
- **There is no idempotency key.** A retried `POST /employees` creates a second employee. Confirm with
  `GET /employees` before retrying a write that may have partially succeeded.

## Steps

1. **Resolve the company's structure first.** Call `GET /company/departments`,
   `GET /company/positions`, and `GET /company/worksites` and pick the ids you will attach the new
   hire to. Creating these on the fly (`POST /company/departments`, `POST /company/positions`,
   `POST /company/worksites`) is possible but pollutes the company's org data — prefer matching an
   existing record.
2. **Create the employee.** `POST /employees` takes the composite `EmployeePost` body: an `employee`
   object plus `wages[]`, `deductions[]` and `bank_accounts[]` arrays. Sending them together in one
   call is the safest path, because the API has no transaction and no idempotency — a single create is
   one retry risk instead of four.
3. **Read back the created record.** The insert response (`InsertSuccessful`) returns `id` and
   `version`. Call `GET /employees/{employee_id}` to confirm the record and capture the current
   `version` of every sub-object before you change anything.
4. **Set direct deposit if it was not in the create.** `PUT /employees/{employee_id}/bank-accounts`
   **replaces** the whole set — it is a PUT, not a PATCH. Send every account you want the employee to
   keep, with `routing`, `account`, `savings`, and either `percent` or `amount`.
5. **Set pay items.** `PATCH /employees/{employee_id}/wages` with the pay items (`pay_type_id`,
   `hours`, `rate`, `override_amount`). To make a raise effective on a future date instead, use
   `POST /employees/{employee_id}/wages/schedule`, which routes the change through approval.
6. **Set tax fields.** `PATCH /employees/{employee_id}/tax-fields` for federal status, allowances,
   exemptions and the state fields. Then `PATCH /employees/{employee_id}/tax-fields/worksite` to bind
   the employee to a worksite id — the worksite drives state and local tax treatment, so do not skip
   it for a remote hire.
7. **Attach deductions.** `GET /company/deductions` for the company's deduction catalog, then
   `POST /employees/{employee_id}/deductions/{deduction_id}` to create the employee's deduction batch.

## Rules that will bite you

- **`version` is required on every PATCH.** Every mutable object carries a `version`. Patching with a
  stale `version` is rejected with a version mismatch — re-read the object, take the new `version`,
  and resend. This is deliberate: it stops your write from silently reverting a pay change another
  integration just made. See `conventions/onpay-conventions.yml`.
- **Errors are not RFC 9457.** Failures come back as
  `{"resp":"error","error_code":100,"error_message":"...","message":"...","more_info":"..."}`. A
  missing token returns **400**, not 401. The `more_info` URL OnPay returns
  (`https://docs.onpay.com`) does not resolve — do not follow it. See
  `errors/onpay-problem-types.yml`.
- **No pagination on `GET /employees`.** The collection endpoints return a bare array with no cursor
  or total. Expect the whole roster in one response and budget memory accordingly.
- **No rate-limit headers.** OnPay signals nothing about throttling; use conservative client-side
  pacing and treat any non-200 as a stop condition rather than a retry loop.
