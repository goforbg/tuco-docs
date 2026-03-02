# Truth table: Sending iMessage — find or create Quick Send lead everywhere

**Policy:** No place is allowed to send without a `leadId`. Every send path **finds or creates** a Quick Send lead (Quick Sends list is created if it doesn’t exist). All messages are stored with a `leadId`.

---

## Summary

| # | Entry point | Find or create Quick Send lead? | Notes |
|---|-------------|----------------------------------|--------|
| 1 | **POST /api/leads/send** | **Yes** | No `leadId` + has recipient: `findLeadByRecipient` then **`createQuickSendLead`** if not found. Message always has `leadId`. |
| 2 | **POST /api/messages** | **Yes** | No `leadId` + has recipient: find then **create Quick Send lead** if not found (with duplicate check in Quick Sends list). |
| 3 | **POST /api/unibox/send-reply** | **Yes** | Reply in existing conversation: use `existingMessage.leadId` if set; else **find or create Quick Send lead** by recipient. Message always has `leadId`. |
| 4 | **Unibox send-attachments** (server action) | **Yes** | Existing conversation: find then **create Quick Send lead** if not found. |
| 5 | **Campaign send** (worker / daily) | **N/A** | Only sends to leads already in the campaign. |

---

## 1. POST /api/leads/send

- **File:** `app/src/app/api/leads/send/route.ts`
- **Behavior:** If client sends `recipientPhone` and/or `recipientEmail` without `leadId`, we call `findLeadByRecipient`. If found we set `leadOid`; if **not** found we call **`createQuickSendLead`** (Quick Sends list created if needed) and set `leadOid` to the new lead. Message is always created with `leadId`.
- **Result:** **Find or create.** No send without `leadId`.

---

## 2. POST /api/messages

- **File:** `app/src/app/api/messages/route.ts`
- **Behavior:** If no `leadId` but recipient provided: `findLeadByRecipient` then **`createQuickSendLead`** if not found (after duplicate check in Quick Sends list). Always have a lead before send.
- **Result:** **Find or create.** No send without `leadId`.

---

## 3. POST /api/unibox/send-reply

- **File:** `app/src/app/api/unibox/send-reply/route.ts`
- **Behavior:** Reply in existing conversation. We use `existingMessage.leadId` if present. If **missing**, we **find or create**: `findLeadByRecipient(recipientPhone, recipientEmail)` then **`createQuickSendLead`** if not found. `messageRequest.leadId` is always set before calling `createMessage`.
- **Result:** **Find or create.** No send without `leadId`.

---

## 4. Unibox send-attachments (server action)

- **File:** `app/src/app/actions/send-attachments.ts`
- **Behavior:** Existing conversation. “Ensure recipient exists as lead under Quick Sends. Create if missing.” `findLeadByRecipient` then **`createQuickSendLead`** if not found.
- **Result:** **Find or create.** No send without `leadId`.

---

## 5. Campaign send

- **Path:** Worker / campaign daily processor.
- **Behavior:** Only sends to leads added to the campaign. No “send to non-lead” path.
- **Result:** **N/A.**

---

## Helpers

- **getOrCreateQuickSendsList:** Ensures the Quick Sends list exists for the workspace; creates it if not. Used by `createQuickSendLead`.
- **findLeadByRecipient:** Look up lead by workspace + phone/email (primary + alt). Does not create.
- **createQuickSendLead:** Finds by recipient (returns existing if found); otherwise creates a new lead in the Quick Sends list (creating the list if needed). Used everywhere we need a lead for send.
