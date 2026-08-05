---
name: Change employee pay safely in OnPay
description: Apply a raise, a deduction change, or a leave-balance change to an OnPay employee without clobbering another integration's write, using the required optimistic-concurrency `version` field.
api: openapi/onpay-api-openapi.json
generated: '2026-08-04'
method: generated
source: openapi/onpay-api-openapi.json + https://onpay.readme.io/reference/versioning
operations:
  - GET /employees/{employee_id}
  - GET /employees/{employee_id}/wages
  - PATCH /employees/{employee_id}/wages
  - POST /employees/{employee_id}/wages/schedule
  - GET /employees/{employee_id}/wages/schedule
  - GET /employees/{employee_id}/deductions
  - PATCH /employees/{employee_id}/deductions/{deduction_id}
  - POST /employees/{employee_id}/deductions/{deduction_id}/schedule
  - GET /employees/{employee_id}/leave
  - PATCH /employees/{employee_id}/leave
---

# Change employee pay safely in OnPay

> Identified by METHOD + path — the OnPay API v2 spec declares no `operationId` on any operation.

## Why this skill exists

OnPay's own documentation tells the story of an employee whose raise was silently reverted because a
second application PATCHed an address using a request body that still carried the old pay rate.
OnPay's answer is an **optimistic-concurrency `version` field**, and it is mandatory. If you ignore
it you will eventually underpay somebody.

## Preconditions

- Partner credentials, `Authorization: Bearer {access_token}`, base URL `https://api.onpay.com/v2`.
- Role `Owner` or `Approver` — wage and deduction writes accept no lesser scope.
- Access tokens expire in 2 hours; refresh tokens are single-use.

## The read-modify-write loop

1. **Read the current object.** `GET /employees/{employee_id}/wages` (or
   `GET /employees/{employee_id}/deductions`, `GET /employees/{employee_id}/leave`). Capture the
   `version` value from the response.
2. **Modify only the fields you intend to change**, on the object you just read. Never assemble a
   PATCH body from a cached copy — that is the exact failure OnPay documents.
3. **Send the PATCH with the `version` you just read.**
   `PATCH /employees/{employee_id}/wages`, `PATCH /employees/{employee_id}/deductions/{deduction_id}`,
   or `PATCH /employees/{employee_id}/leave`.
4. **On a version mismatch, do not retry blindly.** Go back to step 1, re-read, re-apply your delta on
   top of the newer object, and resend. A mismatch means somebody else changed the record — treat it
   as a merge, not a transient error.
5. **Store the new `version`.** The `PatchSuccessful` response returns the updated `version`; that is
   the token for your next write.

## Future-dated changes go through the schedule endpoints

A raise effective next pay period is **not** a PATCH. Use:

- `POST /employees/{employee_id}/wages/schedule` — "Schedule changes to employee wages for approval."
- `POST /employees/{employee_id}/deductions/{deduction_id}/schedule` — schedule a deduction-batch
  change.

Read back what is pending with `GET /employees/{employee_id}/wages/schedule` and
`GET /employees/{employee_id}/deductions/{deduction_id}/schedule` before submitting another one, so
you do not stack two conflicting scheduled changes on the same employee.

## Rules that will bite you

- **No idempotency key exists.** A retried `POST .../schedule` schedules the change twice. Always
  `GET` the schedule first to check whether your previous attempt landed.
- **A missing token returns HTTP 400** with `error_code: 100` and `message: "Missing token"`; an
  invalid or expired one can return 401 or 403 with `"expired token"`. Refresh and replay rather than
  hammering.
- **`PUT /employees/{employee_id}/bank-accounts` is a full replace**, unlike everything else here.
- **Nothing is transactional.** Wage, deduction, and leave changes are separate calls with separate
  versions. If step 3 of a multi-object change fails, the earlier writes have already committed —
  reconcile by re-reading, never by replaying the whole batch.
