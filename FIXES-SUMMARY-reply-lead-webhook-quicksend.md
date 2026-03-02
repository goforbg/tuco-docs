# Summary: Reply→Lead, Webhook, Quicksend Fixes (ADHD-Friendly)

Short recap of the 2–3 fixes we did and where they live.

---

## 1. Reply → lead association (same logic as Unibox)

**Problem:** Reply messages were often stored without `leadId`. Unibox showed them (groups by recipient), but recent activity and campaign reply counts did not.

**Fix:** For every reply we set `leadId` from (1) the original message’s `leadId` if present, else (2) `findLeadByRecipient(workspaceId, phone, email)` — same workspace + email/phone logic as Unibox.

- **With original:** `processReplyWithOriginal` — if `existingMessage.leadId` is missing, resolve with `findLeadByRecipient`, set `replyLeadId`, save reply and fire webhook with that lead.
- **Without original:** `processReplyWithoutOriginal` — before inserting reply, call `findLeadByRecipient`, set `leadId` on the new message, pass resolved lead into webhook.

**Code:** `app/src/app/api/webhooks/v1-device-callbacks/route.ts`; lookup in `app/src/lib/quickSendsList.ts` (`findLeadByRecipient`).

**Doc:** `docs/reply-lead-fix-proof-and-flowchart.md` (ADHD flowchart + proof).

---

## 2. Webhook delivery (backward-compatible, matches GET /api/replies)

**Problem:** `message.reply` webhook didn’t match API shape; missing `parentMessages` and full reply object; no backward compatibility.

**Fix:**

- **`message`** = reply text only (string).
- **`data.reply`** = full reply message object, same shape as GET /api/replies item (body, recipient, timestamps, `leadId`, etc.).
- **`parentMessages`** always included (array, possibly `[]`).
- Top-level: `firstName`, `lastName`, `phone`, `listId`, `campaignId`, `fromLineId`, `recipientName`, `leadId`, etc.

**Code:** `app/src/lib/webhookDelivery.ts`; callers in device-callbacks route always pass `parentMessages`.

**Docs:** `docs/api-reference/message-webhooks.mdx`, `docs/message-reply-webhook-truth-tables.md`, `docs/replies-api-and-webhook-examples.md`.

---

## 3. Quicksend: set `leadId` on outgoing message when client doesn’t send it

**Problem:** For quicksend (`POST /api/leads/send`), if the client sent only `recipientEmail`/`recipientPhone` (no `leadId`), the message was stored without `leadId`. So the first outbound had no lead link → reply couldn’t inherit it and stats were wrong.

**Fix:** When the request has no `leadId` but has recipient (phone and/or email), call `findLeadByRecipient(db, orgId, recipientPhone, recipientEmail)` and set `leadOid` on the created message when a lead is found.

**Code:** `app/src/app/api/leads/send/route.ts`.

---

## 4. Campaign stops when they reply (cancel follow-ups)

**Problem:** When the original outbound had no `leadId` (e.g. quicksend), we still passed that original into `cancelFutureCampaignStepsOnReply`, so cancel hit “no_lead_id” and did not cancel follow-ups.

**Fix:** Pass a `messageForCancel` that uses the **resolved** `replyLeadId` when the original had no `leadId`, so cancel finds and cancels queued/pending/scheduled steps for that lead.

**Code:** `app/src/app/api/webhooks/v1-device-callbacks/route.ts` (same file as fix 1).

---

## ADHD flowchart (one picture)

```
INBOUND REPLY
    │
    ▼
SET LEAD ON REPLY (original.leadId or findLeadByRecipient)
    │
    ├──► STOP CAMPAIGN: cancel future steps (use resolved leadId when original had none)
    ├──► STATS: getRepliedLeadIds() sees reply.leadId → reply/open counts correct
    └──► NOTIFICATIONS: webhook gets lead by replyMessage.leadId → email/Telegram right lead
```

**Pseudocode / ADHD docs:** `docs/reply-lead-fix-proof-and-flowchart.md`, `docs/campaign-list-only-pending-flow.md`, `app/docs/plans/ghl_instant_status_pseudocode.md`.

---

## Repos touched

- **app:** device-callbacks route, webhookDelivery, quickSendsList, leads/send route, leadIdentifiers, etc.
- **docs:** message-webhooks.mdx, reply-lead-fix-proof-and-flowchart.md, debug-reply-loki-queries.md, truth tables, examples.
- **workers:** no code changes (reply handling and lead resolution are in the app).
