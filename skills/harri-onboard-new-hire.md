---
name: Onboard a new hire into Harri
description: Create an employee in Harri, attach them to a location, assign a job title and position, set their pay type and hourly rate, and stamp the payroll and POS identifiers your downstream systems need.
api: openapi/harri-employee-openapi.yml
operations: [GetAllLocations, GetAllJobTitles, GetAllPositions, CreateEmployeeV7, AttachEmployeeToLocation, AttachJobTitleToEmployee, AttachEmployeePosition, SetPrimaryEmployeePosition, GetEmployeePayTypes, CreateEmployeePayRate, EditEmployeeLocationInfo, GetEmployeeByIdV6]
---

# Onboard a new hire into Harri

Use this when a new employee has been hired and must exist in Harri with enough structure that
scheduling, timekeeping and payroll export all work.

## Before you start

- Get a token: `POST https://oauth.harri.com/oauth2/token` with `client_id`, `client_secret` and
  `grant_type=client_credentials`, form-encoded. Send `Authorization: Bearer <access_token>` on every
  call. The token lives 1800 seconds — mint one and reuse it for the whole window. Harri explicitly
  discourages a token per request.
- Base URL is `https://gateway.harri.com/open-api-hub`.
- **There is no idempotency key.** If a create call times out, do not blindly retry — re-read with
  `ListEmployeesV4` first, or you will create a duplicate employee.
- Rate limit is 400 requests/minute. A `429` (a `403` on deployments before 4 June 2026) means you were
  throttled; back off. There are no rate-limit response headers to read, only the status code.
- If you operate franchises, every operation below has a mirror at
  `/api/v{n}/franchisees/{franchiseeId}/...`. Pick one path family and stay in it.

## Steps

1. **Resolve the reference data.** `GetAllLocations` (`GET /api/v1/locations`),
   `GetAllJobTitles` (`GET /api/v1/job_titles`) and `GetAllPositions` (`GET /api/v1/positions`).
   Cache these — they change rarely and each call spends rate-limit budget.
2. **Create the employee.** `CreateEmployeeV7` (`POST /api/v7/employees`). Use v7, not the v3/v4/v5/v6
   create operations — those are all flagged `deprecated: true` in the spec.
3. **Attach them to their home location.** `AttachEmployeeToLocation`
   (`POST /api/v2/employees/{employeeId}/locations/{locationId}`).
4. **Assign the job title.** `AttachJobTitleToEmployee`
   (`POST /api/v1/employees/{employeeId}/job_titles/{jobTitleId}`), then `UpdateEmployeeJobTitle` if you
   need to set title-related detail.
5. **Attach the position and make it primary.** `AttachEmployeePosition`
   (`POST /api/v3/employees/{employeeId}/locations/{locationId}/positions/{positionCode}`), then
   `SetPrimaryEmployeePosition` (`POST /api/v2/employees/{employeeId}/positions/{positionCode}/set_primary`).
6. **Set pay.** Read the pay types with `GetEmployeePayTypes`
   (`GET /api/v1/employees/{employeeId}/pay_types`), then create the rate with `CreateEmployeePayRate`
   (`POST /api/v3/employees/{employeeId}/pay_types/{payTypeId}/locations/{locationId}/positions/{positionCode}/hourly_rates`).
   Salaried employees use the AnnualRates operations instead.
7. **Stamp the external identifiers.** `EditEmployeeLocationInfo`
   (`PUT /api/v2/employees/{employeeId}/locations/{locationId}/info`) sets `payroll_id` and `pos_id`.
   Do this — the payroll and POS exports key on these fields, and an employee without them will fall out
   of downstream reconciliation.
8. **Verify.** `GetEmployeeByIdV6` (`GET /api/v6/employees/{employeeId}`) and confirm the record reads
   back the way you wrote it.

## Errors

- `400` — invalid ID supplied, or a malformed body. The spec gives a description only; there is no error
  code and no `application/problem+json` envelope, so branch on the status code.
- `403` — unauthorized access, or (before 4 June 2026) a rate-limit rejection. Read the body to tell them apart.
- `404` — employee not found.
- `401` — bad credentials, malformed `Authorization` header, or an expired token. Re-mint the token.

See `errors/harri-problem-types.yml` and `conventions/harri-conventions.yml`.
