# CEO Technical Review: February 6, 2026 Release

**Status:** Ready for Deploy  
**Impact:** HIGH — Customer-facing bug fixes + infrastructure hardening  
**Affected Systems:** App, Workers, Webhooks, Unibox

---

## TL;DR — What This Release Contains

This is NOT just a scheduled campaign fix. It's a comprehensive hardening of the entire messaging pipeline.

| Area | What Broke / What Was Missing | What We Fixed |
|------|------------------------------|---------------|
| **Scheduled Campaigns** | Never auto-activated. Manual activation also broken. | Worker now auto-activates with atomic race prevention. Manual activation works. |
| **Campaign Stats** | Webhook processing was slow. Duplicate counts possible on retries. | Stats now processed via BullMQ worker. TTL-based idempotency prevents duplicates. |
| **Stop on Reply** | Lead replies, but still gets follow-up messages. | Now cancels all future campaign steps when lead replies. |
| **Unibox** | Could load 50k+ messages into memory. No pagination. Archived convos hid replies. | Memory capped at 10k. Cursor pagination. Replies un-archive conversations. |
| **Webhook Floods** | No rate limiting. BlueBubbles flood could crash us. | Redis token-bucket rate limiter. Fail-open on Redis down. |
| **Race Conditions** | `createMessageFromOutgoing` could create duplicate messages. | Atomic upsert via `findOneAndUpdate`. |
| **Silent Failures** | Missing REDIS_URL/MONGODB_URI fell back to localhost silently. | Now crashes on startup with clear error. |
| **Index Drift** | Manual index management. Could be missing in prod. | Daily auto-verification via BullMQ maintenance job. |

---

## Feature 1: Scheduled Campaign Activation (3 Fixes)

### Problem
Scheduled campaigns NEVER auto-activated. The code just logged them:
```
"Found 5 scheduled campaigns" 
"Note: Main app handles activation - this is just a check"  // LIE
```

When user clicked "Activate" manually, it marked leads complete WITHOUT sending messages.

### Fix 1A: Internal Activation API

| File | `app/src/app/api/campaigns/[id]/activate-internal/route.ts` |
|------|-------------------------------------------------------------|
| **What** | New endpoint for worker to trigger message initialization |
| **Auth** | `SERVICE_TOKEN` (existing, no new secrets) |
| **Why not in worker?** | Message initialization is ~200 lines in app. Single source of truth. |

### Fix 1B: Worker Atomic Activation

| File | `workers/src/services/campaign-daily.ts` |
|------|------------------------------------------|
| **What** | `activateScheduledCampaigns()` now actually activates |
| **Race Prevention** | Atomic `status: scheduled → activating` transition |
| **Failure Recovery** | On error, reverts to `scheduled` for retry |

```
Worker A                         Worker B                       User clicks "Activate"
──────────────────────────────────────────────────────────────────────────────────────
updateOne({status:'scheduled'})   updateOne({status:'scheduled'})   updateOne({status:'scheduled'})
    modifiedCount: 1              modifiedCount: 0                  modifiedCount: 0
    ✅ I claimed it               ❌ Skip                           ❌ Skip
```

### Fix 1C: Manual Activation Fix

| File | `app/src/app/api/campaigns/route.ts` |
|------|--------------------------------------|
| **What** | Activate action now handles `scheduled` and `draft` campaigns |
| **Before** | Only handled `paused` → called `processCampaign()` (assumes messages exist) |
| **After** | For `scheduled`/`draft` → calls `initializeCampaignMessages()` first |

### New Status: `activating`

| File | `app/src/models/Campaign.ts` |
|------|------------------------------|
| **What** | New intermediate status during activation |
| **Why** | Prevents race conditions during 2-second initialization window |

---

## Feature 2: Campaign Stats Overhaul

### Problem
- Webhook processing was slow — stats updates blocked webhook responses
- BullMQ retries could cause duplicate stat counts
- `openedRecipients[]` and `repliedRecipients[]` arrays on campaign document caused bloat

### Fix

| Component | What Changed |
|-----------|--------------|
| **BullMQ Queue** | New `campaign-stats` queue with concurrency 20 |
| **Worker Service** | `workers/src/services/campaign-stats.ts` — processes stats in background |
| **Idempotency** | `workers/src/lib/idempotency.ts` — TTL collection pattern |
| **Campaign Model** | Removed `openedRecipients[]` and `repliedRecipients[]` arrays |

### Idempotency Pattern

```
idempotency_keys collection:
─────────────────────────────
_id: "campaign:abc123:opened:lead_xyz"
createdAt: 2026-02-06T10:00:00Z  ← TTL index expires after 7 days
```

- First insert succeeds → process stats
- Duplicate insert fails (duplicate key) → skip
- No manual cleanup — MongoDB TTL auto-expires

### Webhooks Now Queue Stats

| Before | After |
|--------|-------|
| `updateCampaignStatsOnDelivery(messageId)` inline | `addCampaignStatsJob({ operation: 'delivery', messageId })` |
| Blocked webhook response | Webhook returns immediately |
| No retry on failure | BullMQ retries 3x with exponential backoff |

---

## Feature 3: Stop Sequence on Reply

### Problem
Lead replies to Step 1, but still receives Steps 2, 3, 4 (queued/scheduled messages).

### Fix

| File | `app/src/app/api/webhooks/v1-device-callbacks/route.ts` |
|------|--------------------------------------------------------|
| **Function** | `cancelFutureCampaignStepsOnReply()` |
| **What** | Marks all queued/pending/scheduled messages with `stepIndex > repliedStep` as `cancelled` |
| **Required Index** | `{ leadId: 1, batchId: 1, status: 1, stepIndex: 1 }` |

### New Message Status: `cancelled`

| File | `app/src/models/Message.ts` |
|------|----------------------------|
| **What** | New status for cancelled campaign steps |
| **Filtered from** | Unibox, stats, opens |

---

## Feature 4: Unibox Improvements

### Problem
- Could load 50k+ messages into memory (OOM risk)
- No cursor pagination (page drift if new messages arrive)
- Archived conversations hid replies (user missed messages)
- Showed "pending" messages that weren't sent yet

### Fixes

| Fix | Details |
|-----|---------|
| **Memory Cap** | 10,000 message limit at DB level with `$nin: ['queued', 'pending', 'scheduled', 'cancelled']` |
| **Cursor Pagination** | `?cursor=base64({lastMessageAt, recipient})` — stable, no page drift |
| **3-State Conversations** | `active` (replied in 30 days), `pending` (no reply or stale), `archived` (manual) |
| **Un-archive on Reply** | `updateConversationOnReply()` sets status to `active` when reply arrives |
| **Track `lastReplyAt`** | New field on Conversation model for engagement tracking |

### New Conversation Status Logic

| Status | Meaning |
|--------|---------|
| `active` | Prospect has replied at least once (ever) |
| `pending` | We sent messages but prospect hasn't replied yet |
| `archived` | Manually archived by user |

---

## Feature 5: Webhook Hardening

### 5A: Rate Limiting

| File | `app/src/lib/rateLimiter.ts` |
|------|------------------------------|
| **Algorithm** | Redis token bucket (100 tokens, 20/sec refill) |
| **Key** | Per-device (`serverUrl` from webhook) |
| **Fail-Open** | If Redis down, allow requests (log for monitoring) |
| **Circuit Breaker** | 30s cooldown before retry if Redis fails |

### 5B: Atomic Message Creation

| File | `app/src/app/api/webhooks/v1-device-callbacks/route.ts` |
|------|--------------------------------------------------------|
| **Function** | `createMessageFromOutgoing()` |
| **Before** | `insertOne()` — race condition could create duplicates |
| **After** | `findOneAndUpdate({ upsert: true })` — atomic |

### 5C: Unsent Message Filtering

All paths now filter out unsent messages at DB level:

| Query Filter | Effect |
|--------------|--------|
| `status: { $nin: ['queued', 'pending', 'scheduled', 'cancelled'] }` | Prevents matching scheduled Step 2 when Step 1's open arrives |

---

## Feature 6: Infrastructure Hardening

### 6A: No Silent Localhost Fallbacks

| File | Location |
|------|----------|
| `app/src/lib/bullmq.ts` | Throws if `REDIS_URL` missing |
| `workers/src/index.ts` | Throws if `REDIS_URL` or `MONGODB_URI` missing |

**Before:** Silent fallback to `localhost:6379` → jobs silently fail in production
**After:** Crash on startup with clear error → Railway deploy fails visibly

### 6B: Automated Index Verification

| Component | Details |
|-----------|---------|
| **Worker** | `workers/src/services/db-maintenance.ts` |
| **Schedule** | Daily at 3:00 AM UTC via BullMQ recurring job |
| **Indexes** | 6 critical indexes including idempotency TTL |

---

## Files Changed Summary

### App (Modified: 9, New: 3)

| File | Change |
|------|--------|
| `src/app/api/campaigns/[id]/activate-internal/route.ts` | **NEW** — Internal activation endpoint |
| `src/app/api/campaigns/route.ts` | Fix activate for scheduled/draft |
| `src/app/api/unibox/conversations/route.ts` | Cursor pagination, memory cap, 3-state |
| `src/app/api/unibox/mark-read/route.ts` | Filter cancelled messages |
| `src/app/api/webhooks/v1-device-callbacks/route.ts` | Rate limit, cancel-on-reply, atomic upsert, queue stats |
| `src/lib/bullmq.ts` | New queues, fail-loud on missing REDIS_URL |
| `src/lib/campaignProcessor.ts` | Comment update (idempotency pattern) |
| `src/lib/campaignStats.ts` | Add `cancelled` to unsent statuses |
| `src/lib/ensureIndexes.ts` | **NEW** — Fire-and-forget index creation |
| `src/lib/rateLimiter.ts` | **NEW** — Redis token bucket rate limiter |
| `src/models/Campaign.ts` | Add `activating` status, remove bloat arrays |
| `src/models/Conversation.ts` | Add `pending` status, `lastReplyAt`, `ACTIVE_REPLY_WINDOW_DAYS` |
| `src/models/Message.ts` | Add `cancelled` status |

### Workers (Modified: 2, New: 3)

| File | Change |
|------|--------|
| `src/index.ts` | New services, fail-loud env validation |
| `src/services/campaign-daily.ts` | Atomic scheduled activation, call app API |
| `src/lib/idempotency.ts` | **NEW** — TTL-based idempotency |
| `src/services/campaign-stats.ts` | **NEW** — Background stats processing |
| `src/services/db-maintenance.ts` | **NEW** — Daily index verification |

---

## Environment Variables

**No new environment variables required.** Uses existing:

| Variable | Used By | Purpose |
|----------|---------|---------|
| `REDIS_URL` | App + Workers | Rate limiting, BullMQ queues |
| `MONGODB_URI` | Workers | Database connection |
| `SERVICE_TOKEN` | Worker → App | Internal API authentication |
| `APP_URL` / `MAIN_APP_URL` | Workers | App API base URL |

---

## What Happens on Failure?

| Failure Mode | System Response | User Impact |
|--------------|-----------------|-------------|
| Worker can't reach app API | Reverts to `scheduled`, retries next cycle | Campaign activates on recovery |
| Stats queue fails | BullMQ retries 3x, logs error | Stats may lag but self-correct |
| Rate limit Redis down | Fail open, allow requests | No rate limiting (logged) |
| Cancel-on-reply DB error | Log error, continue processing | Lead may get extra messages |
| Index creation fails | Log warning, continue | Query may be slower |

---

## Loki Events to Monitor

| Event | Meaning | Alert Threshold |
|-------|---------|-----------------|
| `campaign.scheduled.activation_failed` | Auto-activation failed | > 3 for same campaign |
| `rate_limiter.fail_open` | Rate limiting disabled | > 10/min |
| `reply.sequence.cancel_db_error` | Failed to cancel follow-ups | Any |
| `conversation.update_on_reply.failed` | Reply may not show in inbox | Any (severity: critical) |
| `db.maintenance.index_failed` | Index creation failed | Any |

---

## Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Race condition in activation | **Eliminated** | High | Atomic status transition |
| Duplicate stats on retry | **Eliminated** | Medium | TTL idempotency |
| Memory OOM in Unibox | **Mitigated** | High | 10k message cap |
| Webhook flood | **Mitigated** | High | Token bucket rate limiter |
| Missing indexes | **Mitigated** | Medium | Daily auto-verification |

---

## Verdict

# SHIP

**Justification:** Fixes customer-facing bug (scheduled campaigns), adds defense-in-depth across entire webhook/stats pipeline, eliminates multiple race conditions, and hardens infrastructure against silent failures. All failure modes are recoverable. No new secrets required. Comprehensive logging for debugging.

---

## Post-Deploy Checklist

### Immediate Verification
- [ ] Monitor Loki for `campaign.scheduled.activating` events
- [ ] Verify idempotency index created: `db.idempotency_keys.getIndexes()`
- [ ] Test scheduled campaign activation (create campaign for 2 min from now)
- [ ] Test stop-on-reply (send Step 1, reply, verify Steps 2-N cancelled)
- [ ] Verify Unibox shows `pending` status for new conversations
- [ ] Check `/stats` endpoint shows new queues (`campaign-stats`, `db-maintenance`)

### Grafana Alerts to Configure (within 24h)
- [ ] `campaign.scheduled.activation_failed` where `isTimeout: true` — 3+ in 10 min = app may be down
- [ ] `reply.sequence.cancel_db_error` — any occurrence = lead may get extra messages
- [ ] `conversation.update_on_reply.failed` — any occurrence (severity: critical) = user may miss replies

---

## Code Fixes Applied (Post-Review)

| Fix | Location | Description |
|-----|----------|-------------|
| Fetch timeout | `workers/src/services/campaign-daily.ts` | 30s timeout on `triggerCampaignInitialization()` with AbortController |
| Timeout flag | Same file | `isTimeout: true` added to failure logs for Grafana alerting |
| Unibox type fix | `app/src/app/unibox/page.tsx` | Added `pending` status to Conversation interface |

---

## Backward Compatibility Analysis

### Existing Campaigns

| Scenario | Impact | Action |
|----------|--------|--------|
| Campaigns with `status: 'scheduled'` and `scheduledFor` in past | Will auto-activate on next worker cycle | **INTENDED** — this is the fix |
| Campaigns with `openedRecipients[]` array | Array still works, will continue to grow | No action (harmless cruft) |
| Campaigns without `openedRecipients[]` array | Array created on first open | No action |

### Existing Conversations

| Scenario | Before | After | Impact |
|----------|--------|-------|--------|
| Conversation with any reply | `active` | `active` | No change |
| Conversation with no replies | (not explicit) | `pending` | Now explicitly categorized |
| Archived conversation | `archived` | `archived` | No change |

### Existing Messages

| Scenario | Impact | Action |
|----------|--------|--------|
| Messages without `stepIndex` field | Cancel-on-reply cancels ALL queued steps | Logged as warning |
| Messages without `batchId` field | Skip stats processing | No action |
| Messages with `cancelled` status | N/A — none exist yet | New status |

### v1-message Webhook (Known Gap)

The `v1-message` webhook still uses old inline stats functions instead of BullMQ queue.
- **Impact:** No idempotency protection on this path
- **Risk:** Low — this webhook has fewer retries
- **Migrate:** Within 1 week post-deploy

---

*Document prepared for CEO review — February 6, 2026*
