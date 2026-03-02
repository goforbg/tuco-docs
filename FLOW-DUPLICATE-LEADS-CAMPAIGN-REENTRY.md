# Duplicate Leads + Campaign Re-Entry (ADHD-Friendly Flowchart)

Summary of behavior since last commit: **never 409 on duplicate leads** (return 200 + existing IDs), **no campaign re-entry**, **new leads run from top**, plus Loki for debugging.

---

## 1. Create lead (POST /api/leads) – duplicates → 200

```
POST /api/leads { leads: [...], listId? }
    │
    ▼
importLeads(...)
    │
    ├─ All new → 201, leadIds, savedCount
    │
    └─ All duplicates (savedCount === 0, duplicateCount >= 1)
           │
           ▼
       Return 200 (not 409)
       Body: message, duplicateOnly, duplicateCount, existingLeadIds, leadIds, duplicates
       Loki: lead.import.result { duplicateOnly: true, existingLeadIds }
```

**Snippet (app):**

```ts
// app/src/app/api/leads/route.ts
if (allDuplicates && result.duplicates?.length) {
  const existingLeadIds = result.duplicates
    .map((d) => d.existingLead._id)
    .filter((id): id is string => id != null);
  return NextResponse.json(
    { message: '...', duplicateOnly: true, existingLeadIds, leadIds: existingLeadIds, ... },
    { status: 200 }
  );
}
```

---

## 2. Add lead to campaign (POST /api/campaigns/[id]/leads)

### 2a. By leadId

```
POST /api/campaigns/[id]/leads { leadId }
    │
    ▼
processLeadThroughCampaign(campaignId, leadId, ...)
    │
    ├─ Lead has messages in this campaign? (batchId ~ ^campaign_<id>_)
    │      │
    │      YES → alreadyInCampaign: true, messageCount: 0
    │            Return 200: "Lead already in campaign (re-entry not allowed).", existingLeadIds
    │            Loki: campaign.process_lead.already_in_campaign
    │
    └─ NO → Ensure lead in list, pick line, create messages from step 1
           Return 201, messageCount
           Loki: campaign.lead.added
```

### 2b. By lead payload (create or use existing)

```
POST /api/campaigns/[id]/leads { lead: { firstName, email|phone, ... } }
    │
    ▼
Workspace duplicate? (same email/phone)
    │
    ├─ YES → Use existing lead (leadObjectId = existing._id), do NOT insert
    │        Loki: campaign.lead.duplicate_use_existing
    │        Then → processLeadThroughCampaign (same as 2a: re-entry check, then step 1 if new)
    │
    └─ NO  → Insert new lead, inc list leadCount
             Then → processLeadThroughCampaign (re-entry check, then step 1)
```

**Re-entry check (campaignProcessor):**

```ts
// app/src/lib/campaignProcessor.ts – processLeadThroughCampaign
const campaignBatchPrefix = `campaign_${campaignId.toString()}_`;
const existingInCampaign = await db.collection<IMessage>(MessageCollection).countDocuments({
  leadId,
  batchId: { $regex: `^${escapedPrefix}` },
});
if (existingInCampaign > 0) {
  return { success: true, messageCount: 0, alreadyInCampaign: true };
}
```

---

## 3. New leads run from top (verified)

- **Single add:** `processLeadThroughCampaign` always creates a new batch and messages for **step 1, 2, …** (from top).
- **Bulk init:** `initializeCampaignMessages` only creates messages for leads that **do not** already have any message in this campaign (skipped leads already have messages → no duplicate sequence).
- So: **new leads always start from step 1**; re-entry is blocked so they never “re-run from top” by being re-added.

---

## 4. Send message (POST /api/messages) – Quick Sends duplicate → 200

```
POST /api/messages (recipientPhone/Email, no leadId)
    │
    ▼
findLeadByRecipient → found? use it
    │
    └─ Not found → Check Quick Sends list by email/phone
           │
           ├─ Match in list → Use existing lead (resolvedLeadId, leadSource = 'duplicate')
           │                  Loki: message.api.duplicate_lead_used
           │                  Continue to create message; return 200 (not 409)
           │                  Body: duplicateOnly, existingLeadIds, duplicateMessage
           │
           └─ No match → createQuickSendLead(), then create message
                         Return 201
```

**Snippet:**

```ts
// app/src/app/api/messages/route.ts
if (match) {
  lead = await db.collection<ILead>(LeadCollection).findOne({ _id: existing._id, workspaceId: orgId });
  if (lead) {
    resolvedLeadId = (existing._id as ObjectId).toString();
    leadSource = 'duplicate';
    lokiLogger.info({ event: 'message.api.duplicate_lead_used', existingLeadId: resolvedLeadId, ... });
  }
  break;
}
// ... later
const statusCode = leadSource === 'duplicate' ? 200 : 201;
return NextResponse.json(payload, { status: statusCode });
```

---

## 5. Loki events (debugging)

| Event | When |
|-------|------|
| `lead.import.result` | After import; `duplicateOnly: true` + `existingLeadIds` when all duplicates |
| `campaign.lead.duplicate_use_existing` | Campaign add with lead payload and workspace duplicate; use existing lead |
| `campaign.process_lead.already_in_campaign` | Add to campaign but lead already has messages in this campaign |
| `campaign.lead.already_in_campaign` | API response 200 for re-entry |
| `campaign.lead.added` | Lead added and sequence started (201) |
| `message.api.duplicate_lead_used` | Quick Sends duplicate; message sent with existing lead |
| `message.send.summary` | Includes `leadSource` ('provided' | 'found' | 'created' | 'duplicate') |

---

## 6. Code safety (quick check)

- **Lead create:** 200 + `existingLeadIds` when all duplicates; no 409 → consumers (e.g. n8n) don’t stop.
- **Campaign add:** Re-entry guarded by existing campaign messages (any `batchId` for that campaign + lead); duplicate workspace lead is reused and still goes through re-entry check.
- **Messages:** Duplicate in Quick Sends reuses lead and sends; response 200 with `existingLeadIds` and `duplicateMessage`.
- **batchId:** “Already in campaign” uses `^campaign_<id>_` so it matches both bulk init and single-add batchIds.

---

## Files touched (this round)

- `app/src/app/api/leads/route.ts` – all-duplicates → 200, existingLeadIds
- `app/src/app/api/campaigns/[id]/leads/route.ts` – duplicate workspace lead → use existing; alreadyInCampaign → 200
- `app/src/lib/campaignProcessor.ts` – already-in-campaign check, return `alreadyInCampaign`, Loki
- `app/src/app/api/messages/route.ts` – Quick Sends duplicate → use existing lead, 200, Loki
