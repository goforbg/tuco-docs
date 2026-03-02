# Lead deletion: what gets removed (truth table)

When a lead is deleted (single lead delete or list delete with “delete leads”), the following data is removed so the workspace stays clean for handoff and testing.

## Single lead delete: `DELETE /api/leads` with `{ ids: [leadId, ...] }`

| Data | Collection | Deleted? | Notes |
|------|------------|----------|--------|
| Lead record | `leads` | ✅ Yes | Lead document removed. |
| **Opens** (email/iMessage open events) | `activities` | ✅ Yes | All activities with `leadId` in set, including `type: 'message_opened'`. |
| **Message counts / sent messages** | `messages` | ✅ Yes | All messages with this `leadId`: campaign steps, unibox sent, quicksend. |
| **Unibox replies** (inbound from prospect) | `messages` | ✅ Yes | Stored in same `messages` collection with `leadId`; removed with above. |
| **Unibox messages we sent** | `messages` | ✅ Yes | Same as “message counts” – all outbound with this `leadId`. |
| **Campaign recipient status** | `campaigns.recipientStatuses` | ✅ Yes | Entries for this lead are `$pull`ed; campaign stats decremented. |
| **Campaign stats** (totalRecipients, pending, sent, etc.) | `campaigns.stats` | ✅ Adjusted | Decremented so totals stay correct. |
| **List lead count** | `lists.leadCount` | ✅ Adjusted | Decremented for the list the lead was in. |
| **Unibox conversation record** (status, tags, lastReplyAt) | `conversations` | ✅ Yes | Conversation doc keyed by recipient (email/phone); removed when recipient matches any of the lead’s contact identifiers (email, phone, alts). |

So: **opens, message counts, campaign messages, unibox replies, unibox messages we sent, and all stats for that lead are removed.** Conversation metadata for that lead’s contact(s) is also removed.

## List delete with “delete leads”: `DELETE /api/lists` with `{ listId, deleteLeads: true }`

Same as above, applied to every lead in the list:

- `activities` – all with `leadId` in list’s leads  
- `messages` – all with `leadId` in list’s leads  
- `conversations` – all where `recipient` is in the set of contact identifiers (email, phone, alts) of any lead in the list  
- `campaigns` – `recipientStatuses` and `stats` updated for each affected campaign  
- `lists.leadCount` – N/A (list is deleted)  
- `leads` – all leads in the list  

## What is *not* tied to `leadId`

- **Conversations** are keyed by `recipient` (email/phone), not `leadId`. Deletion uses the lead’s contact identifiers (from `getLeadContactIdentifiers`) so that the right conversation rows are removed. Email matching is case-insensitive so Conversation docs stored with original-case email (e.g. from webhooks) are still removed.
- **outgoing_messages** (BlueBubbles delivery cache) has no `leadId`; it’s keyed by `guid`. It is not deleted on lead delete. Orphaned rows are harmless; messages for that lead are already gone.
- **Line / workspace config, webhooks, API keys, etc.** are not per-lead and are unchanged.

## Implementation reference

- **Single lead delete:** `app/src/app/api/leads/route.ts` – `DELETE` handler (steps 1–5 + conversation cleanup).
- **List delete with deleteLeads:** `app/src/app/api/lists/route.ts` – same cleanup for all leads in the list.
- **Contact identifiers:** `app/src/lib/leadIdentifiers.ts` – `getLeadContactIdentifiers(lead)` (email, phone, alt*).
- **Conversation recipient format:** Same normalization as Unibox (E.164 for phone, normalized email).

## Summary

| Item | On lead delete |
|------|----------------|
| Opens | ✅ Deleted (activities) |
| Message counts we sent to them | ✅ Deleted (messages) |
| Campaign messages | ✅ Deleted (messages) + campaign stats updated |
| Unibox replies | ✅ Deleted (messages) |
| Unibox messages we sent | ✅ Deleted (messages) |
| Unibox conversation (status/tags) | ✅ Deleted (conversations) |
| Campaign recipient status + stats | ✅ Removed / decremented |
| List lead count | ✅ Decremented |

Everything for that lead is cleaned so the workspace is ready for client handoff.

---

## End-to-end verification checklist

When you delete a lead (or a list with “delete leads”), the following are verified in code:

| Step | Location | What is done |
|------|----------|--------------|
| 1 | `DELETE /api/leads` | Load leads to delete (with `listId` + contact fields for conversations). |
| 2 | Same | Delete all `activities` where `leadId` ∈ ids (includes opens, sends, failures). |
| 3 | Same | Delete all `messages` where `leadId` ∈ ids (campaign, unibox sent/replies, quicksend). |
| 4 | Same | Build recipient set from `getLeadContactIdentifiers(lead)` for each lead; delete `conversations` where `recipient` matches (exact + case-insensitive for emails). |
| 5 | Same | For each campaign with these leads in `recipientStatuses`: `$pull` those entries, decrement `stats.totalRecipients` and `stats.<status>`. |
| 6 | Same | Delete lead documents from `leads`. |
| 7 | Same | For each list that had a deleted lead, decrement `lists.leadCount`. |
| — | `DELETE /api/lists` with `deleteLeads: true` | Same steps 2–6 for all leads in the list (list itself is then deleted; no list count decrement). |
