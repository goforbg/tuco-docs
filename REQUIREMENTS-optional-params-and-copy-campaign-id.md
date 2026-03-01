# Requirements: Optional Params (Empty String = Omit) + Quick Sends Default + Copy Campaign ID

**Status:** Draft for approval. Do not implement until approved.

---

## 1. Add Lead To Campaign — Is List ID Optional?

**Clarification:** For **POST /api/campaigns/{campaignId}/leads** (Add Lead to Campaign), there is **no `listId` in the request body**. The campaign already has a `listId`; new leads created inline via `lead: { ... }` are added to that campaign’s list. So “listId” is not a parameter of this route.

For **POST /api/leads** (Create Lead), `listId` **is** optional: when omitted (or empty, per below), the API should use the Quick Sends list (create if needed). This is already implemented; this doc asks to also treat **empty string** as “use default”.

---

## 2. Treat Empty String as “Not Provided” (Same as Null/Undefined)

**Requirement:** In the following routes and for the following parameters, treat **empty string** the same as **null** or **undefined**: use default behavior (RR, Quick Sends, etc.) instead of validating the value or returning 404.

| Route | Parameter(s) | Current / desired behavior when value is `""`, `null`, or omitted |
|-------|----------------|-------------------------------------------------------------------|
| **POST /api/messages** (send iMessage) | `fromLineId` | Use round-robin over active healthy lines. (Already: `rawFromLineId = fromLineId.trim()` and `!resolvedFromLineId` → RR. Confirm empty string is not treated as a valid ID.) |
| **POST /api/leads** (create leads) | `listId` | Use Quick Sends list (create if needed). (Already: `trimmedListId = listId != null && listId !== '' ? String(listId).trim() : ''` and `!trimmedListId` → default. Confirm no edge case where `""` is passed through and causes 404.) |
| **POST /api/leads/send** (quick send) | `lineId` | Use round-robin over active healthy lines. (Verify empty string triggers RR, not “line not found”.) |
| **POST /api/campaigns/{id}/leads** (add lead to campaign) | `leadId` (when using existing lead) | N/A for listId. If `leadId` is sent as `""`, treat as “not provided” and require `lead` data instead of looking up by ID. |

**Implementation rule:** For each of these params, normalize before use, e.g.:

- `const effective = (param != null && String(param).trim() !== '') ? String(param).trim() : undefined;`
- Then: if `effective` is undefined/null, apply default (RR, Quick Sends, etc.); otherwise use `effective`.

No 404 or “list not found” when the client sends an empty string for an optional param that has a default.

---

## 3. POST /api/leads — Default to Quick Sends (Same as Messages)

**Requirement:** When `listId` is **omitted, null, or empty string**, POST /api/leads must use the **Quick Sends** list (create it if it does not exist), same as POST /api/messages behavior. No “list not found” or “listId required” for empty/omitted.

**Current state:** The route already defaults to Quick Sends when `trimmedListId` is falsy. This doc asks to:

- Explicitly treat `listId: ""` and `listId: "   "` as “use default” (no 404).
- Document this in API docs and in the “easy example” below.

---

## 4. Easy Request/Response Example — Add Lead to Campaign

**Requirement:** Provide a single, copy-paste friendly request/response example for **adding a lead to a campaign** (minimal fields, no optional params unless needed).

**Proposed example (for docs):**

**Request**

```http
POST /api/campaigns/{campaignId}/leads
Content-Type: application/json
Authorization: Bearer <API_KEY>
```

**Body (option A — existing lead):**

```json
{
  "leadId": "507f1f77bcf86cd799439011"
}
```

**Body (option B — create new lead and add to campaign):**

```json
{
  "lead": {
    "firstName": "Jane",
    "lastName": "Doe",
    "phone": "+14155551234",
    "email": "jane@example.com"
  }
}
```

**Response (201):**

```json
{
  "success": true,
  "leadId": "507f1f77bcf86cd799439011",
  "campaignId": "507f1f77bcf86cd799439022",
  "messageCount": 1,
  "message": "Lead added to campaign and sequence started"
}
```

**Note:** `listId` is not a parameter of this endpoint; the lead is added to the campaign’s list. For **creating leads** (POST /api/leads), `listId` is optional and defaults to Quick Sends.

---

## 5. UI — Copy Campaign ID (Like Copy Phone in Unibox)

**Requirement:** In the app UI, provide an **easy way to copy the campaign ID** to the clipboard, similar to how Unibox lets users copy the phone number (e.g. button or icon that runs `navigator.clipboard.writeText(campaignId)` and shows a short “Copied” toast).

**Proposed placement:**

- **Campaign detail page** (`/campaigns/[id]`): Show the campaign ID (e.g. in header or in a “Campaign details” section) with a **Copy** button/icon next to it. On click: copy `campaign._id` (or string equivalent) to clipboard and show a “Campaign ID copied” toast.

**Reference (Unibox):** Unibox copies `selectedConversation.recipient` (phone or email) with a copy button and toast. Reuse the same pattern (copy button + toast) for campaign ID.

---

## 6. Summary Checklist (For Implementation After Approval)

- [x] **POST /api/messages:** Ensure `fromLineId === ""` or `fromLineId === "   "` is treated as “not provided” and triggers RR (no lookup by empty string).
- [x] **POST /api/leads:** Ensure `listId === ""` or whitespace-only is treated as “not provided” and triggers Quick Sends default (no 404).
- [x] **POST /api/leads/send:** Ensure `lineId === ""` or whitespace-only is treated as “not provided” and triggers RR.
- [x] **POST /api/campaigns/{id}/leads:** If `leadId` is sent as `""` or whitespace-only, treat as “not provided” and require `lead` object (or return 400 “Either leadId or lead data is required”).
- [x] **Docs:** Add the “Add lead to campaign” request/response example (e.g. in `docs/integrations/n8n.mdx` or `docs/api-reference/`).
- [x] **UI:** Add “Copy campaign ID” on campaign detail page (e.g. next to campaign ID), with “Copied” toast.

---

## Approval

Once this doc is approved, implementation can start. Any changes to the requirements should be reflected in this doc before coding.
