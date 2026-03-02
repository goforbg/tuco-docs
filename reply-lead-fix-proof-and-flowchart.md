# Reply → lead association fix (proof + flowchart)

## What was broken

- Lead exists in DB (e.g. quicksend/API/CSV) with `email` / `phone`.
- They reply. Reply shows in Unibox (grouped by recipient).
- **Reply message in DB had no `leadId`** → recent activity and campaign reply count ignored it; webhook had no lead.

**Why:** We only set reply’s `leadId` from the **original outbound message**. If that original had no `leadId` (legacy, bug, or some send path that didn’t set it), or we didn’t find an original at all, we never looked up the lead by email/phone. Unibox works because it groups by **recipient** (email/phone), not by `leadId`.

---

## The fix (same idea as Unibox: match by email/phone in workspace)

One function does the lookup. Same for both reply paths.

**File:** `app/src/lib/quickSendsList.ts`

```ts
// Lines 70–91: find lead by recipient (email or phone) in workspace
export async function findLeadByRecipient(
  db: Db,
  workspaceId: string,
  recipientPhone: string | undefined,
  recipientEmail: string | undefined
): Promise<ILead | null> {
  if (!recipientPhone && !recipientEmail) return null;
  const normalizedPhone = normalizePhoneForLookup(recipientPhone);
  const normalizedEmail = recipientEmail?.toLowerCase().trim();
  const orConditions = [];
  if (normalizedPhone) orConditions.push({ phone: normalizedPhone });
  if (normalizedEmail) orConditions.push({ email: normalizedEmail });
  if (orConditions.length === 0) return null;
  return await db.collection<ILead>(LeadCollection).findOne({
    workspaceId,
    $or: orConditions,
  });
}
```

So: **email or phone from the reply** (webhook) is used to find the **lead in that workspace**. No dependency on the original message having `leadId`.

---

## Fix 1: Reply **with** original (99% case)

**File:** `app/src/app/api/webhooks/v1-device-callbacks/route.ts`  
**Function:** `processReplyWithOriginal`  
**Lines:** ~2646–2682

**Before:** Reply’s `leadId` = `existingMessage.leadId` only. If original had no `leadId`, reply stayed without `leadId`.

**After:** Use original’s `leadId` if present; **else** resolve by email/phone and use that.

```ts
// Use original message's leadId when present; otherwise resolve by recipient (same as Unibox)
let replyLeadId = existingMessage.leadId;
if (!replyLeadId && (phoneNumber || emailAddress)) {
  const resolvedLead = await findLeadByRecipient(db, workspaceId, phoneNumber ?? undefined, emailAddress ?? undefined);
  replyLeadId = resolvedLead?._id ?? undefined;
  // ... log lead_resolved or no_lead_mapped
}

const replyMessageObj = createReplyMessageObject(..., replyLeadId, ...);
```

So: **If the original has no `leadId`, we still set the reply’s `leadId`** by looking up the lead with the same email/phone from the webhook.

---

## Fix 2: Reply **without** original (no matching outbound)

**File:** `app/src/app/api/webhooks/v1-device-callbacks/route.ts`  
**Function:** `processReplyWithoutOriginal`  
**Lines:** ~2166–2199, 2301

**Before:** We never set `leadId` on the reply (no original → no `leadId`).

**After:** Before inserting the reply we resolve the lead by email/phone and set `leadId` on the new message; we also pass that lead into the webhook.

```ts
// Resolve lead by recipient (same logic as Unibox)
const resolvedLead = await findLeadByRecipient(db, line.workspaceId, phoneNumber ?? undefined, emailAddress ?? undefined);
const resolvedLeadId = resolvedLead?._id ?? undefined;
// ... log lead_mapped or no_lead_mapped

const replyMessage: IMessage = {
  ...
  ...(resolvedLeadId && { leadId: resolvedLeadId }),
  ...
};
// ...
await fireMessageReplyWebhook(replyMessage, undefined, resolvedLead ?? undefined, ...);
```

So: **Even with no original, the reply gets `leadId`** when a lead exists for that email/phone in the workspace, and the webhook gets the lead.

---

## ADHD flowchart (small)

```
Incoming reply (webhook)
    │
    ▼
Extract phone + email from webhook
    │
    ▼
Find original outbound? (by recipientEmail/recipientPhone in workspace)
    │
    ├── YES (with original) ──────────────────────────────────────────┐
    │         │                                                         │
    │         ▼                                                         │
    │    replyLeadId = original.leadId                                  │
    │         │                                                         │
    │         ▼                                                         │
    │    original.leadId missing? ──YES──► findLeadByRecipient(email,phone)
    │         │                                    │                    │
    │         NO                                   ▼                    │
    │         │                         replyLeadId = resolved._id      │
    │         ▼                                                         │
    │    Create reply message WITH replyLeadId                          │
    │         │                                                         │
    │         ▼                                                         │
    │    Fire webhook (leadId + lead when we have them)                 │
    │                                                                   │
    └── NO (without original) ─────────────────────────────────────────┤
              │                                                         │
              ▼                                                         │
         findLeadByRecipient(email, phone) in line.workspaceId          │
              │                                                         │
              ▼                                                         │
         Create reply message WITH resolvedLeadId (or none)             │
              │                                                         │
              ▼                                                         │
         Fire webhook (resolvedLead so leadId + lead when found)        │
                                                                        │
◄────────────────────────────────────────────────────────────────────────┘
Reply in DB has leadId when a lead exists for that email/phone in workspace.
Recent activity + campaign reply count + webhook all get lead.
```

**One-liner:** For every reply we set `leadId` from (1) the original message’s `leadId` if present, else (2) `findLeadByRecipient(workspaceId, phone, email)`. Same workspace + email/phone logic as Unibox.

---

## Campaign stops when they reply (follow-ups cancelled)

**Worry:** Lead is in campaign, follow-up in 2 days; they reply → campaign must stop.

**What we do:** When we record a reply (with or without original), we **cancel all future campaign steps** for that lead so the worker never sends them.

```
Reply recorded (reply message has leadId)
    │
    ▼
cancelFutureCampaignStepsOnReply(db, messageForCancel, requestId)
    │
    ├── messageForCancel = original message
    │   BUT if original.leadId was null we use resolved leadId
    │   (so cancel can find queued steps for that lead)
    │
    ▼
Query: messages where
  leadId = messageForCancel.leadId,
  batchId matches campaign,
  stepIndex > replied step,
  status in [queued, pending, scheduled]
    │
    ▼
UPDATE those messages → status = 'cancelled'
    │
    ▼
Worker’s campaign-daily only processes status = 'queued' → follow-ups never send.
```

**Fix added:** If the **original** outbound had no `leadId` (e.g. quicksend), we were still calling cancel with that original → cancel skipped (no_lead_id). We now pass a `messageForCancel` that uses the **resolved** `replyLeadId` so cancel finds and cancels the queued follow-up steps.

**Assurance:** Once the reply is stored with the correct `leadId`, cancel uses that same leadId (original or resolved). Follow-ups for that lead are cancelled. No extra messages after they reply.

---

## Batch vs non-batch (2 leads vs 200)

**Worry:** Missing leadId because campaign had only 2 leads (not “processed in batches”)?

**Answer:** Same code path. Campaign messages are **created in the app** (createMessage or createQueuedMessage), not in the worker. Worker only **reads** queued messages and sends them. So:

- **2 leads:** App creates 2 messages (each with leadId when we have the lead). Worker picks them up, same logic.
- **200 leads:** App creates 200 messages (each with leadId). Worker processes in batches (gap enforcement, line rotation). Still same message shape.

The bug was never “batch vs non-batch”. It was **quicksend** creating the **first** outbound message without `leadId` when the client didn’t send `leadId` (only recipient email/phone). We fixed that in `POST /api/leads/send`: we now resolve lead by recipient and set `leadId` on the message when we can.

---

## Stats (reply count, open count) — how they’re fixed

**Worry:** Wrong replied count, wrong open count on campaign list and reply page.

**Reply count:** Comes from `getRepliedLeadIds()`:

- Query: messages with `direction: 'received'` and `leadId` in workspace.
- Then narrow to leads who also have a **sent/delivered campaign message** (same batchId pattern).

So if the **reply** message has no `leadId`, that reply is **never** counted. Fix: we now set `leadId` on every reply (from original or from `findLeadByRecipient`). So reply count is correct.

**Open count:** Derived from message status (opened/replied). Repliers are included in “opened”. No separate bug there; once reply is tied to lead, aggregates are correct.

**One-liner:** Fixing reply (and outbound) lead association fixes **replied** and **opened** stats, because both rely on messages having the right `leadId`.

---

## Email + Telegram notifications (right lead)

**Worry:** Do email and Telegram notifications find the right lead and associate the reply to it?

**Flow:** After we store the reply (with `leadId`), we:

1. Load **lead** by `replyMessage.leadId` (so we have the resolved lead when we resolved it).
2. Load **campaign** from batchId when it’s a campaign.
3. Call `fireMessageReplyWebhook(replyMessage, originalMessage, lead, campaign, …)`.

So the **webhook payload** (and thus any email/Telegram that consumes it) gets the same **lead** we used for the reply. If we resolved by email/phone, `replyMessage.leadId` is set, we load that lead, and pass it into the webhook. Notifications get the correct lead and association.

---

## ADHD summary (flowchart)

```
┌─────────────────────────────────────────────────────────────────────────┐
│  INBOUND REPLY                                                           │
└─────────────────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  SET LEAD ON REPLY                                                       │
│  • With original: use original.leadId else findLeadByRecipient(phone,email) │
│  • Without original: findLeadByRecipient(phone,email) → reply.leadId    │
└─────────────────────────────────────────────────────────────────────────┘
    │
    ├──────────────────────────────────┬──────────────────────────────────┐
    ▼                                  ▼                                  ▼
┌─────────────────┐  ┌─────────────────────────────┐  ┌─────────────────────────┐
│ STOP CAMPAIGN   │  │ STATS                        │  │ NOTIFICATIONS            │
│ Cancel future   │  │ getRepliedLeadIds() counts  │  │ Webhook gets lead from   │
│ steps using     │  │ replies by leadId → reply   │  │ replyMessage.leadId →    │
│ resolved leadId │  │ count correct               │  │ email/Telegram get right  │
│ when original   │  │ Open count includes         │  │ lead                      │
│ had none        │  │ repliers                     │  │                           │
└─────────────────┘  └─────────────────────────────┘  └─────────────────────────┘
```

**Assurance checklist:**

| Concern | Status |
|--------|--------|
| Campaign stops when contact replies (no follow-up in 2 days) | Yes. Cancel uses resolved leadId when original had none. |
| Reply counted in campaign list / reply page | Yes. Reply has leadId → getRepliedLeadIds includes them. |
| Open count correct | Yes. Same message/lead data. |
| Email/Telegram get right lead | Yes. Lead loaded by replyMessage.leadId and passed to webhook. |
| Batch vs 2 leads | Same path. leadId set at message creation (app); quicksend fix covers missing leadId. |
