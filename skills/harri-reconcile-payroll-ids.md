---
name: Reconcile Harri records with an external payroll or POS system
description: Use Harri's mappings endpoints and the payroll_id / pos_id fields to line Harri employees, locations and positions up with the identifiers your payroll or point-of-sale system uses, and to find the records that are not mapped yet.
api: openapi/harri-employee-openapi.yml
operations: [api_v2_employees_mappings_list, api_v1_employees_mappings_create, api_v1_employees_mappings_updateV2, api_v1_locations_mappings_list, api_v1_locations_mappings_create, api_v1_locations_mappings_update, api_v1_positions_mappings_list, api_v1_positions_mappings_create, api_v1_positions_mappings_update, EditEmployeeLocationInfo, GetAllLocations, GetAllPositions]
---

# Reconcile Harri records with an external payroll or POS system

Harri has no free-form metadata bag. Instead it exposes first-class mapping endpoints plus the
`payroll_id`, `pos_id` and `geid` fields. This is the integration seam for payroll processors, POS
vendors and daily-pay providers.

Auth, base URL and rate limits: see `harri-onboard-new-hire.md`.

## Find what is not mapped yet

Every `*/mappings` list operation accepts `mode=UNMAPPED`. Start there — it is far cheaper than paging
the whole employee set and diffing.

- `api_v2_employees_mappings_list` (`GET /api/v2/employees/mappings`)
- `api_v1_locations_mappings_list` (`GET /api/v1/locations/mappings`)
- `api_v1_positions_mappings_list` (`GET /api/v1/positions/mappings`)

The list operations also accept `internal_values` and `external_values` so you can look a specific
identifier up in either direction rather than scanning.

## Create and maintain the mappings

- Employees: `api_v1_employees_mappings_create` (`POST /api/v1/employees/mappings`), then
  `api_v1_employees_mappings_updateV2` (`PUT /api/v2/employees/mappings/{id}`) to change one.
- Locations: `api_v1_locations_mappings_create` / `api_v1_locations_mappings_update`.
- Positions: `api_v1_positions_mappings_create` / `api_v1_positions_mappings_update`.

Map locations and positions **before** employees. An employee mapping that points at an unmapped
location will import, but the labor export it feeds will not resolve.

## The per-record identifier fields

`EditEmployeeLocationInfo` (`PUT /api/v2/employees/{employeeId}/locations/{locationId}/info`) sets
`payroll_id` and `pos_id` on the employee-at-location record. These are distinct from the mappings
endpoints: mappings are a translation table, these fields live on the record itself and are what the
export templates emit. Keep both in step.

## Operating notes

- Paginate with `limit` and `page` where the operation declares them. Responses are bare JSON arrays with
  no `has_more` flag and no total — you are done when a page comes back shorter than `limit`.
- Batch reconciliation runs are the most likely thing to hit the 400 requests/minute ceiling. Pace the
  loop and treat `429` (or `403` before 4 June 2026) as back-pressure.
- There is no idempotency key. Guard every create with a `mode=UNMAPPED` read so a retry does not create
  a second mapping row.
