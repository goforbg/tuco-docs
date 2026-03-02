# POST /api/leads – Response IDs and Status Codes

**Summary:** POST `/api/leads` now returns created lead IDs and uses clearer HTTP status codes. No request-body or auth order changes.

---

## Changes

### 1. Response includes `leadIds` and `leadListIds`

- **Before:** Success response had `message`, `savedCount`, `duplicateCount`, `totalProcessed`, `listId`, and optionally `duplicates`.
- **After:** Success response also includes **`leadIds`**: an array of Tuco lead IDs for **newly created** leads only, in the same order as inserted; and **`leadListIds`**: array of list IDs the new leads were added to (one list per request).
- **Use:** Callers can link created leads to their system and know which list(s) they belong to; call GET/PUT with these IDs without an extra round-trip.

**Files:**  
- `app/src/lib/importLeads.ts` – `ImportLeadsResult` extended with `savedLeadIds?: string[]`; after `insertMany`, IDs are taken from `insertResult.insertedIds` and returned.  
- `app/src/app/api/leads/route.ts` – success JSON includes `leadIds: result.savedLeadIds ?? []` and `leadListIds: result.listId ? [result.listId] : []`.

### 2. HTTP status codes

- **201 Created** – Returned when at least one lead was created (`savedCount > 0`).
- **200 OK** – Returned when the request is successful but no new leads were inserted (e.g. theoretical empty-batch path).
- **409 Conflict** – Unchanged: when **all** submitted leads are duplicates in the list; body includes `code: "DUPLICATE_LEADS"`, `duplicateCount`, and `duplicates` (with `existingLead._id`).
- **401, 402, 404, 400, 500** – Unchanged (unauthorized, payment required, list not found, validation, server error).

### 3. Loki logging

- Success log for POST `/api/leads` now includes **`leadIdsCount`** (number of IDs returned). The full `leadIds` array is **not** logged (keeps logs small and API fast).

---

## Types and behavior

- **Types:** `tsc --noEmit` passes. `insertMany` returns `insertedIds: { [key: number]: ObjectId }`; we use `Object.values(...).map(id => id.toString())` for `savedLeadIds`.
- **Performance:** No extra DB calls; IDs come from `insertMany`’s result. No large log payloads.

---

## Documentation

- **Customer-facing:** `docs/api-reference/endpoint/create.mdx` updated with `leadIds`, 201/200/409, and error status codes.
