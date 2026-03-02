# Lead Uniqueness: Email/Phone Per Workspace (Truth Table)

**Rule:** In a workspace, no two leads may share the same **email** or **phone** when normalized. “Same” includes primary and all alt fields (altEmail1–3, altPhone1–3). Uniqueness is **per workspace**, not per list.

Used for: Unibox reply lookup (one lead per recipient), campaign targeting, and consistent contact identity.

---

## Normalization

| Input type | Normalization |
|------------|----------------|
| Email      | Trim, lowercase |
| Phone      | E.164 via `normalizePhoneToE164` (e.g. `+15551234567`) |

Empty/whitespace is ignored. A lead must have at least one non-empty identifier (email or phone).

---

## Truth Table: Can we create/update this lead?

| Workspace | Existing lead(s) | New/updated lead identifiers | Allowed? | Reason |
|-----------|------------------|------------------------------|----------|--------|
| W1        | None             | email: a@x.com               | Yes      | No conflict |
| W1        | Lead A: a@x.com  | email: a@x.com               | No       | Duplicate email in workspace |
| W1        | Lead A: a@x.com  | phone: +15551111111           | Yes      | Different identifier |
| W1        | Lead A: a@x.com  | email: a@x.com, list L2      | No       | Same workspace; list does not matter |
| W1        | Lead A: phone +15551111111 | phone: +15551111111  | No       | Duplicate phone |
| W1        | Lead A: altEmail1 b@x.com   | email: b@x.com       | No       | b@x.com already on A (alt) |
| W1        | Lead A: email a@x.com       | altEmail1: a@x.com   | No       | a@x.com already on A (primary) |
| W1        | Lead A: a@x.com  | email: A@X.COM               | No       | Normalized to a@x.com → duplicate |
| W1        | Lead A: +15551234567 | phone: 5551234567         | No       | Same number after E.164 |
| W2        | Lead in W1: a@x.com | email: a@x.com (in W2)   | Yes      | Different workspace |

---

## Where enforcement happens

| Action | Enforcement |
|--------|-------------|
| **POST /api/leads** (import) | `importLeads`: workspace-wide duplicate check by `contactIdentifiers`; same-batch duplicates (same identifier in two rows) rejected; index `workspaceId + contactIdentifiers` (unique multikey). |
| **PUT /api/leads** (update) | If any of email/phone/alt* change: merge with current doc, recompute `contactIdentifiers`, find other lead in workspace with any of those identifiers; if found → 409. Duplicate key (11000) → 409. |
| **POST /api/campaigns/[id]/leads** (add lead) | Workspace-wide check before creating new lead; if identifier exists → 409 with `existingLeadId` (client can use that leadId to add to campaign). New lead gets `contactIdentifiers` set. |
| **Quick send / createQuickSendLead** | `findLeadByRecipient` first; if lead exists for that email/phone (or alts), return existing lead id (no new lead). New lead gets `contactIdentifiers` set. |

---

## Lookup (Unibox / reply)

- **findLeadByRecipient(workspaceId, phone?, email?)**  
  Builds identifiers from the given phone/email, then finds one lead where any of `contactIdentifiers` or legacy email/phone/alt* matches.  
  So one lead per recipient per workspace; reply always attaches to the same lead.

---

## Index

- **Unique multikey:** `{ workspaceId: 1, contactIdentifiers: 1 }`  
  Ensures at most one lead per (workspaceId, identifier) for any identifier in `contactIdentifiers`.  
  New/updated leads get `contactIdentifiers` set; legacy leads without it are still checked via email/phone/alt* in app logic.
