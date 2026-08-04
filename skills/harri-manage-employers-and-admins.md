---
name: Manage Harri employers, brands and above-store admin users
description: Create and maintain employer/brand records and the above-store admin users attached to them, and reconcile employer identifiers with an external system.
api: openapi/harri-employer-openapi.json
operations: [listEmployersV2, getEmployerV2, createEmployerV2, updateEmployerV2, deleteEmployerV2, createUser, retrieveUsersByBusinessId, retrieveUser, updateUser, deleteUser, api_v1_employers_mappings_list, ListCorporateUsers, CreateCorporateUser]
---

# Manage Harri employers, brands and above-store admin users

The External Brand Management API is the second half of the Harri Open API Hub. It manages employer/brand
records and the above-store administrators who work across locations.

Auth, base URL and rate limits: see `harri-onboard-new-hire.md`. Note the path prefix here is `/v1/` and
`/v2/`, **not** `/api/v1/` as on the Employee API.

## Before anything else: get your corporate ID associated

Harri gates this API on credential-to-corporate-ID association. If you get a `422` complaining that a
corporate ID is not associated or not found, that is not a bug in your request — contact Harri Support
and have the ID associated with your API credentials. Nothing on this API will work until that is done.

## Employer / brand records (v2)

- `listEmployersV2` — `GET /v2/employers`
- `getEmployerV2` — `GET /v2/employers/{id}`
- `createEmployerV2` — `POST /v2/employers`
- `updateEmployerV2` — `PUT /v2/employers/{id}`
- `deleteEmployerV2` — `DELETE /v2/employers/{id}`

Franchise operators use the mirrored set: `listFranchiseeEmployersV2`, `getFranchiseeEmployerV2`,
`createFranchiseeEmployerV2`, `updateFranchiseeEmployerV2`, `deleteFranchiseeEmployerV2`.

## Above-store admin users (v1)

- `retrieveUsersByBusinessId` — `GET /v1/employers`
- `retrieveUser` — `GET /v1/employers/{user_external_id}`
- `createUser` — `POST /v1/employers`
- `updateUser` — `PUT /v1/employers/{user_external_id}`
- `deleteUser` — `DELETE /v1/employers/{user_external_id}`

These are keyed on **your** `user_external_id`, not a Harri id. That is deliberate: it lets you drive the
user set from your own identity system without holding a Harri id map. Choose a stable external id and
never reuse one.

## Read the warnings array

The success envelope on this API carries a `data.warnings[]` array of `{code, message}` — for example
`INVALID_BRAND_EXTERNAL_ID`. A `200` is not automatically a clean result. Always inspect `warnings[]`
before you record the operation as successful; partial success is reported there, not in the status code.

## Reconcile identifiers

`api_v1_employers_mappings_list` (`GET /v1/employers/mappings`) lists the Harri-to-external employer
identifier mappings. See `harri-reconcile-payroll-ids.md` for the wider mapping pattern.

## Corporate users

Corporate-level users live on the Employee API, not this one: `ListCorporateUsers`
(`GET /api/v1/corporates/users`), `CreateCorporateUser`, `RetrieveCorporateUser`, `UpdateCorporateUser`,
`DeleteCorporateUser`. Do not confuse them with the above-store admin users here.

## Errors

`400` bad request, `403` unauthorized, `404` not found, `422` corporate ID not associated. No structured
error envelope. See `errors/harri-problem-types.yml`.
