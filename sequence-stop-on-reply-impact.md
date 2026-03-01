# Sequence stop-on-reply: end-to-end impact

When a lead replies to a campaign message, all **later** steps in that sequence are now marked `status: 'cancelled'` so the worker never sends them. This doc lists every place that is affected by the new status or the new behavior.

---

## ⚠️ REQUIRED BEFORE DEPLOYMENT

### 1. Add MongoDB Index

Run the migration script **before** deploying:

```bash
npx ts-node scripts/migrations/add-message-cancel-index.ts
```

Or manually via MongoDB shell:

```javascript
db.messages.createIndex(
  { leadId: 1, batchId: 1, status: 1, stepIndex: 1 },
  { name: "idx_cancel_future_steps", background: true }
)
```

**Without this index, the cancel query does a full collection scan and will kill MongoDB at scale.**

---

## 1. Where `cancelled` is set

| Location | What happens |
|----------|--------------|
| `app/src/app/api/webhooks/v1-device-callbacks/route.ts` | On reply with original message: `cancelFutureCampaignStepsOnReply()` runs with 5s timeout. It `updateMany` messages: same lead, same campaign run, `stepIndex >` replied step, status in `queued`/`pending`/`scheduled` → set `status: 'cancelled'`. |

### Security hardening

- **batchId length capped** at 200 chars to prevent regex DoS (logged at `warn` level)
- **Null bytes and control characters stripped** from batchId before regex
- **5-second timeout** on cancel operation to prevent hanging webhooks

### Timeout behavior (documented, intentional)

The 5-second timeout uses `Promise.race` which only stops *waiting* for the MongoDB operation—it does NOT cancel the operation itself. If timeout triggers:
- The `updateMany` continues executing in the background (dangling promise)
- The operation will still complete and cancel messages (just slower)
- We log `reply.sequence.cancel_error` so ops can investigate slow queries

This is acceptable because proper cancellation (AbortController with MongoDB) adds significant complexity for minimal benefit. The dangling promise will eventually complete successfully or fail silently (either way, messages get cancelled if DB is up).

### Known race condition (acceptable)

If a message is in `'sending'` status (worker is actively transmitting), we intentionally do NOT cancel it. Including `'sending'` would cause a worse problem: message sends successfully but DB shows `'cancelled'`. The race window is small (seconds) and the consequence (one extra message after reply) is minor and acceptable.

### Stats handling (race condition avoided)

We do **NOT** update `campaign.stats.pending` when cancelling. This avoids a race condition where concurrent recalc could cause negative pending counts. Stats will self-correct on the next `recalculateCampaignStats` call.

### Backwards compatibility

The function handles old data gracefully:

| Old data scenario | Behavior | Log level | Log event |
|-------------------|----------|-----------|-----------|
| Message has no `batchId` | Skip (not a campaign message) | debug | `reply.sequence.cancel_skip` reason=`no_batch_id` |
| `batchId` doesn't start with `campaign_` | Skip (not a campaign sequence) | debug | `reply.sequence.cancel_skip` reason=`not_campaign_message` |
| `batchId` too long (>200 chars) | Skip (security) | **warn** | `reply.sequence.cancel_skip` reason=`batch_id_too_long` |
| Message has no `leadId` | Skip (can't identify lead) | debug | `reply.sequence.cancel_skip` reason=`no_lead_id` |
| `batchId` has no `_step_N` suffix | Skip (old format) | debug | `reply.sequence.cancel_skip` reason=`no_step_suffix` |
| Message has no `stepIndex` | Cancel ALL queued steps (use -1) | **warn** | `reply.sequence.cancel_no_stepindex` |

Supports both batchId formats:
- `campaign_{id}_{timestamp}_step_N` (from `processCampaign`)
- `campaign_{id}_lead_{leadId}_{timestamp}_step_N` (from `processLeadThroughCampaign`)

---

## 2. Where `cancelled` is handled (so behavior is correct)

| Location | Change / behavior |
|----------|-------------------|
| **Worker – campaign-daily** | Queries only `status: 'queued'` for messages to send. Cancelled messages are never selected → **no send**. |
| **Worker – checkAndCompleteCampaign** | Remaining count uses `status: { $in: ['queued','pending','scheduled'] }`. Cancelled not included → **cancelled steps don't block campaign completion**. |
| **Worker – syncCampaignStatsFromMessages** | `pendingStatuses = ['queued','pending','scheduled']`. Cancelled not in list → **cancelled not counted as pending** in campaign stats. |
| **Unibox – conversations** | Exclude `cancelled` from messages shown in thread (same as queued/pending/scheduled) → **no "ghost" unsent step in the thread**. Also treat `cancelled` like unsent for "is received" logic. |
| **Unibox – mark-read** | Mark-read query uses `status: { $nin: ['queued','pending','scheduled','cancelled'] }` → **we don't mark cancelled messages as read**. |
| **Webhook – delivery/open** | `unsentStatuses` includes `'cancelled'` → **we never record delivery/open for a cancelled message** (defense in depth). |
| **campaignStats** | `unsentStatuses` in "never count opens" includes `'cancelled'`. Recalc `pendingCount` uses only queued/pending/scheduled. `statusBreakdown` now includes `cancelled` → **opens not counted for cancelled; recalc correct; breakdown consistent**. |
| **campaignStatusFromMessages** | No branch for `message.status === 'cancelled'` → **cancelled messages don't change recipient status** (lead shows from last sent/delivered/failed step). |
| **campaignStatusAggregation** | Aggregation matches sent/delivered/failed/fallback. Cancelled not matched → **cancelled not shown as sent/failed; leads still categorized by actual sent messages**. |

---

## 3. Schema / types

| Location | Change |
|----------|--------|
| `app/src/models/Message.ts` | `status` union includes `'cancelled'`. |
| `workers/src/services/campaign-daily.ts` | Local `IMessage.status` includes `'cancelled'`. |

---

## 4. Not affected (no code change)

- **Reply processing** (worker reply job, email/Telegram, classification): unchanged; reply path does not read/write message status for other steps.
- **Campaign creation / message creation**: still create with `status: 'queued'`; nothing sets `cancelled` at create time.
- **Worker send flow**: only picks `status: 'queued'`; never sees `cancelled` in that query.
- **HubSpot plugin**: Maps Tuco status to queued/sent/delivered/read/failed; `cancelled` is only set in DB on reply, not returned from send API → **no change needed**.
- **scheduled-messages.ts / message-processing-service**: Handle single-message scheduling, not campaign steps; they don't use `cancelled`.
- **Line limits, dashboard stats, timeseries**: Count only `sent`/`delivered` (or similar); they don't need to treat `cancelled` specially.

---

## 5. Summary

- **Sending**: Worker only sends `queued`; cancelled steps are never sent.
- **Unibox**: Cancelled steps are hidden from the thread and not used for "received" or mark-read.
- **Campaign stats / completion**: Completion logic excludes cancelled; recalc handles stats correctly.
- **Opens/delivery**: Cancelled is treated as unsent everywhere we check.

No other code paths assume a fixed set of statuses in a way that breaks for `cancelled`; the additions above are the ones needed for consistent end-to-end behavior.

---

## 6. Loki events for tracing

Query sequence cancellation flow:

```logql
{job="tuco-app"} |= `reply.sequence` | json
```

| Event | Level | Meaning |
|-------|-------|---------|
| `reply.sequence.cancel_check` | debug | Entry point - checking if reply should cancel steps |
| `reply.sequence.cancel_skip` | debug/warn | Early exit - not a campaign message or missing data (see `reason` field). **`batch_id_too_long` is `warn` level** (security event). |
| `reply.sequence.cancel_no_stepindex` | warn | Message has no stepIndex, cancelling ALL queued steps |
| `reply.sequence.cancel_query` | debug | About to execute updateMany |
| `reply.sequence.cancel_none_found` | debug | No messages matched the cancel query |
| `reply.sequence.cancelled_future_steps` | **info** | Success: N messages cancelled |
| `reply.sequence.cancel_db_error` | **error** | MongoDB updateMany failed (returns `success: false`) |
| `reply.sequence.cancel_failed` | **error** | Cancel returned `success: false` (e.g., DB error) |
| `reply.sequence.cancel_error` | **error** | Cancel threw or timed out (includes 5s timeout) |

---

## 7. Error handling

The `cancelFutureCampaignStepsOnReply` function returns a result object:

```typescript
{ success: true, cancelledCount: number }     // Success (0 = no matches)
{ success: false, cancelledCount: 0, error: string }  // DB error
```

The caller wraps this in `Promise.race` with a 5-second timeout:
- **Timeout**: Throws, caught by caller, logged as `cancel_error`
- **DB error**: Returns `success: false`, logged as `cancel_failed`
- **Success**: Returns `success: true`, logged as `cancelled_future_steps` (if count > 0)

In all failure cases, the webhook continues processing (reply is recorded). The failure is logged at `error` level for monitoring.

---

## 8. Known limitations

1. **Race with 'sending'**: If worker is mid-send when reply arrives, that message still sends. Small window, acceptable.
2. **No un-cancel**: Cancelled messages can't be retried via UI. Intentional: lead replied = don't spam them.
3. **iMessage only**: This feature only applies to iMessage campaigns (BlueBubbles webhook). Email campaigns would need separate handling if added later.
4. **Stats not immediate**: We don't update `stats.pending` on cancel to avoid race conditions. Stats correct on next recalc.
5. **Dangling promise on timeout**: If cancel times out, the MongoDB operation continues in background. This is documented and acceptable (see "Timeout behavior" above).

---

## 9. Production checklist

- [ ] Run index migration: `npx ts-node scripts/migrations/add-message-cancel-index.ts`
- [ ] Verify index exists: `db.messages.getIndexes()` should show `idx_cancel_future_steps`
- [ ] Deploy app changes
- [ ] Monitor Loki for `reply.sequence.cancel_error` events
- [ ] Verify with test: 2-step campaign, reply after step 1, confirm step 2 shows `cancelled` in DB
