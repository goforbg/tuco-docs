# Campaign: Pending vs “In list only” — Flow + Code

ADHD-friendly: flowchart first, then only the code that changed.

---

## 1. Flowchart (what counts as what)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  LIST (e.g. "My List" with 10 leads)                                         │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                    Campaign LAUNCHED with this list
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  initializeCampaignMessages() runs ONCE                                       │
│  → Creates messages for every lead IN THE LIST at launch time                │
│  → Those leads get status: queued → pending → sent/delivered/etc.            │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
         ┌────────────────────────────┼────────────────────────────┐
         │                            │                            │
         ▼                            ▼                            ▼
   Lead was in list            Lead was in list            NEW lead added
   at launch                   at launch                   to list AFTER
   → has campaign              → has campaign               launch
   messages                    messages                     → NO campaign
   → some still                → all sent/delivered/         messages
     queued/pending              failed/fallback
         │                            │                            │
         ▼                            ▼                            ▼
   PENDING (1)                 REACHED / SENT / etc.        IN LIST ONLY (2)
   "In queue, will             "We contacted them"         "In list but NOT
   be sent"                                                    in campaign"
```

**(1) Pending** = lead has ≥1 campaign message, none sent yet (queued/pending/scheduled).  
**(2) In list only** = lead is on the list but has zero campaign messages → they will NOT get messages unless you add them via “Add to campaign”.

---

## 2. Where the numbers come from (one picture)

```
GET /api/campaigns/[id]
        │
        ├─ totalRecipients = count(leads where listId = campaign.listId)
        ├─ totalWithMessages = count(leads that have ≥1 message for this campaign)  [aggregation]
        │
        ├─ pending  = count(leads whose BEST message status is queued/pending/scheduled)  ← AGGREGATION ONLY
        │              (no longer includes “list-only” leads)
        │
        └─ listOnly = totalRecipients - totalWithMessages
```

---

## 3. Code snippets (only what changed)

### 3.1 Pending = queue only. List-only = separate count.

**File:** `app/src/lib/campaignStatusAggregation.ts` — `mergeStatusCounts()`

```ts
// BEFORE: pending included "list-only" (misleading)
// const pendingFromNoMessages = Math.max(0, totalRecipients - totalWithMessages);
// const pending = pendingFromNoMessages + (countsByStatus.pending ?? 0);

// AFTER: pending = only in-campaign queue; list-only is its own number
const pending = countsByStatus.pending ?? 0;
const listOnly = Math.max(0, totalRecipients - totalWithMessages);

return {
  pending,
  listOnly,  // new
  sent,
  // ...
};
```

---

### 3.2 Filter “Pending” = only leads with messages not yet sent (exclude list-only)

**File:** `app/src/app/api/campaigns/[id]/route.ts`

```ts
if (statusFilter === 'pending') {
  // Pending = in campaign with message(s) not yet sent (exclude list-only)
  const [leadIdsWithMessages, leadIdsWithSentMessage] = await Promise.all([
    getLeadIdsWithMessages(db, batchIdPattern),
    getLeadIdsWithSentMessage(db, batchIdPattern),
  ]);
  const withSentSet = new Set(leadIdsWithSentMessage.map((id) => id.toString()));
  const pendingIds = leadIdsWithMessages.filter((id) => !withSentSet.has(id.toString()));
  paginationTotal = pendingIds.length;
  pageLeadIds = pendingIds.slice(skip, skip + limit);
}
```

---

### 3.3 Filter “In list only” = leads in list with no campaign messages

**File:** `app/src/app/api/campaigns/[id]/route.ts`

```ts
// Allow 'listonly' in query (param is lowercased)
const validStatuses = [..., 'listonly'];

// ...
} else if (statusFilter === 'listonly') {
  const leadIdsWithMessages = await getLeadIdsWithMessages(db, batchIdPattern);
  paginationTotal = Math.max(0, totalRecipients - leadIdsWithMessages.length);
  const listOnlyCursor = db.collection<ILead>(LeadCollection).find(
    { listId, workspaceId: orgId, _id: { $nin: leadIdsWithMessages } },
    { projection: { _id: 1 }, skip, limit }
  );
  const listOnlyPage = await listOnlyCursor.toArray();
  pageLeadIds = listOnlyPage.map((l) => l._id!);
}
```

---

### 3.4 UI: “In list only” in strip + overview; badge for list-only leads

**File:** `app/src/app/campaigns/[id]/page.tsx`

**Strip (tabs under Recipients):**
```ts
...(typeof statusCounts.listOnly === 'number' && statusCounts.listOnly > 0
  ? [{ key: 'listOnly' as const, label: 'In list only', count: statusCounts.listOnly }]
  : []),
```

**Badge (per row):** if lead has no campaign messages → show “In list only” instead of “Pending”:
```ts
const isListOnly = status === 'pending' && (messageCount ?? 0) === 0;
const displayLabel = isListOnly ? 'In list only' : (labels[status] || status);
```

---

## 4. Where “list only” count comes from (no separate MongoDB query)

**`pendingFromNoMessages` was never a MongoDB query.** It was computed in JS as:

```ts
const pendingFromNoMessages = Math.max(0, totalRecipients - totalWithMessages);
```

The two inputs come from:

| Variable             | Where it comes from | MongoDB |
|----------------------|----------------------|--------|
| **totalRecipients**  | GET campaign `[id]` route | `db.collection(LeadCollection).countDocuments({ listId, workspaceId: orgId })` — count of leads in the list. |
| **totalWithMessages**| Aggregation result   | `getCampaignStatusCountsFromAggregation()` runs a pipeline on the **messages** collection (match by `batchId`, group by `leadId`), then `$facet` → `total: [{ $count: 'totalWithMessages' }]` — count of distinct leads that have ≥1 campaign message. |

So:

- **totalRecipients** = `app/src/app/api/campaigns/[id]/route.ts` (around line 139).
- **totalWithMessages** = `app/src/lib/campaignStatusAggregation.ts` → `getCampaignStatusCountsFromAggregation()` (pipeline on `messages`; returned as `result.total[0].totalWithMessages`).

We now use that same formula as **listOnly** (and no longer add it to **pending**):

```ts
// app/src/lib/campaignStatusAggregation.ts — mergeStatusCounts()
const listOnly = Math.max(0, totalRecipients - totalWithMessages);
```

---

## 5. Quick reference

| Term           | Meaning                                                                 |
|----------------|-------------------------------------------------------------------------|
| **Pending**    | In campaign; has message(s) still queued/pending/scheduled (will be sent). |
| **In list only** | In list, zero campaign messages; won’t get messages unless added to campaign. |
| **totalRecipients** | Current list size.                                      |
| **totalWithMessages** | Leads that have ≥1 campaign message (from aggregation). |

---

## 6. Files touched (checklist)

- [ ] `app/src/lib/campaignStatusFromMessages.ts` — `StatusCounts` + optional `listOnly`
- [ ] `app/src/lib/campaignStatusAggregation.ts` — `mergeStatusCounts`: pending vs listOnly
- [ ] `app/src/app/api/campaigns/[id]/route.ts` — pending filter logic, listonly filter, validStatuses
- [ ] `app/src/app/campaigns/[id]/page.tsx` — strip item, overview card, badge “In list only”
