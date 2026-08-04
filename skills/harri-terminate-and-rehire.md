---
name: Terminate or rehire an employee in Harri
description: Terminate an employee with the correct reason code, correct a termination that was entered in error, and rehire someone who is returning — keeping location periods and schedule inclusion consistent.
api: openapi/harri-employee-openapi.yml
operations: [ListEmployeesV4, GetEmployeeByIdV6, GetTerminationReasonGroups, GetTerminationReasonsV2, TerminateEmployeeV3, UndoTermination, RehireEmployeeV5, EditHireDate, IncludeInSchedule, UpdateEmployeeLocationPeriod, DetachEmployeeToLocation]
---

# Terminate or rehire an employee in Harri

Employment status in Harri drives scheduling eligibility and payroll export, so this flow has to be
exact. Auth, base URL, rate limit and the no-idempotency warning are the same as
`harri-onboard-new-hire.md`.

## Terminate

1. **Find the employee.** `ListEmployeesV4` (`GET /api/v4/employees/`) with `status=ACTIVE`, or go
   straight to `GetEmployeeByIdV6` if you already hold the id.
2. **Resolve a valid reason.** `GetTerminationReasonGroups` (`GET /api/v1/termination_reason_groups`)
   then `GetTerminationReasonsV2` (`GET /api/v2/termination_reasons`). Do not send a free-text reason;
   Harri expects one of its own reason ids.
3. **Terminate.** `TerminateEmployeeV3` (`POST /api/v3/employees/{employeeId}/terminate`).
4. **Optionally close the location period.** `UpdateEmployeeLocationPeriod`
   (`PUT /api/v2/employees/{employeeId}/locations/{locationId}/location_periods`) sets the end date, or
   `DetachEmployeeToLocation` removes the association entirely. Prefer setting an end date — detaching
   loses the history that labor reporting reads.
5. **Confirm.** `ListEmployeesV4` with `status=TERMINATED`.

## Undo a termination entered in error

`UndoTermination` (`POST /api/v1/employees/{employeeId}/undo_termination`). This is the correction path —
do not rehire someone who was never actually terminated, because a rehire opens a new employment period
and splits their history.

## Rehire

1. **Confirm they are terminated.** `GetEmployeeByIdV6`.
2. **Rehire.** `RehireEmployeeV5` (`POST /api/v5/employees/{employeeId}/rehire`). Use v5 — `RehireEmployee`
   at `/api/v3/` is flagged deprecated.
3. **Correct the hire date if needed.** `EditHireDate` (`PUT /api/v2/employees/{employeeId}/hire_date`).
4. **Put them back on the schedule.** `IncludeInSchedule`
   (`PUT /api/v2/employees/{employeeId}/include_in_schedule`). A rehired employee who is not included in
   the schedule will silently not appear to schedulers.
5. **Re-verify pay and position.** Rehire does not restore pay rates — re-check with
   `GetEmployeePayTypes` and `GetEmployeePositions` and re-create what is missing.

## Errors

`400` invalid id or reason; `403` unauthorized or throttled; `404` employee not found. No structured error
body — branch on status. See `errors/harri-problem-types.yml`.
