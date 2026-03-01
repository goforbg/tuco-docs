# TUCO AI – iMessage Automation Platform Test Specification

**Target:** API and data flows against **https://dev.tuco.ai** (iMessage automation does not fire webhooks to local, so tests run against dev).  
**TestSprite-style output:** For each test: API request/response, MongoDB state before/after, webhook payloads (when applicable), Pass/Fail, edge cases.  
**Runner:** Use `tests/platform/run-api-tests.mjs` (no TestSprite MCP in workspace; script produces TestSprite-style output).

---

## System Overview

- **Next.js API:** Campaign management, lead import, line management
- **BullMQ workers:** Message processing, job queue
- **MongoDB:** campaigns, leads, messages, lines (DB: `tuco-ai-dev`)
- **BlueBubbles:** Mac Mini iMessage gateway (not testable via API)
- **Webhook.site:** Receive webhook notifications for testing

---

## Authentication

- **Leads, Lists, Campaigns, Lines:** Clerk session only (no API key on these routes).
- **Messages, Check-availability:** API key allowed (`Authorization: Bearer tuco_xxx` or `X-API-Key`).
- For API tests against dev: use **Clerk session cookie** (browser login then copy `__session` or use Playwright to capture cookies).

---

## 1. Lead Import & Duplicate Detection

**Endpoint:** `POST /api/leads` (not `/api/leads/import`)

**Request:**
```json
{
  "leads": [
    { "firstName": "Test", "lastName": "User", "email": "test@example.com", "phone": "+15551234567" }
  ],
  "listId": "<ObjectId of list>",
  "source": "csv",
  "defaultCountryCode": "+1"
}
```

**Response (success):**
```json
{
  "message": "Leads saved successfully",
  "savedCount": 10,
  "duplicateCount": 0,
  "totalProcessed": 10,
  "listId": "<listId>"
}
```

| Test Case | Request | Expected |
|-----------|--------|----------|
| 10 unique contacts | 10 leads, all unique phone/email | `savedCount: 10`, `duplicateCount: 0` |
| 10 contacts, 3 duplicates | 10 leads, 3 repeat phone/email in same list | `savedCount: 7`, `duplicateCount: 3` |
| Invalid phone numbers | Leads with invalid/missing phone | Validation errors, 400 |
| Same CSV twice | Import same list twice | Second run: all duplicates, `savedCount: 0`, `duplicateCount: 10` |

**MongoDB (collection: `leads`):**
- After first import: count by `listId` + `workspaceId` = saved count.
- No duplicate `phone` (normalized) per `listId` + `workspaceId`.
- `duplicateCount` in response matches `totalProcessed - savedCount`.

**TestSprite output:** Request body (sanitized), response status + JSON, MongoDB lead count before/after, Pass/Fail.

---

## 2. Campaign Creation & Rate Limiting

**Endpoint:** `POST /api/campaigns`

**Request:**
```json
{
  "listId": "<ObjectId>",
  "name": "Test Campaign",
  "steps": [
    { "message": "Hello {{firstName}}", "delay": 1, "delayUnit": "days" }
  ],
  "lineIds": ["<lineId1>", "<lineId2>"],
  "settings": {
    "sendImmediately": true,
    "randomizeOrder": false
  }
}
```

**Response:** `{ "campaign": { "_id", "status", "listId", "lineIds", "steps", ... }, "message": "..." }`

| Test Case | Setup | Expected |
|-----------|--------|----------|
| 6 contacts, 2 lines, 5 msgs/day | Create campaign, start immediately | 5 messages queued on line 1, then switch to line 2 (per line daily limit). |
| Limit already hit | Lines at `dailyLimit` for today | Messages queue for next day (or campaign stays pending). |

**MongoDB:**
- **campaigns:** `status` (e.g. `pending` → `active_not_finished` / `running` → `completed`), `messagesSent` / `totalContacts`, `stats.pending`, `stats.sent`, `stats.failed`.
- **lines:** `dailyLimit`, usage (e.g. `messagesUsedToday` or derived from `messages` collection).

**TestSprite output:** Request, response, campaign doc before/after, line usage, Pass/Fail.

---

## 3. Line Health & Failure Detection

**Endpoint:** `GET /api/health-check` (checks all active lines; no auth in current impl – confirm in code).

**Behavior:** For each active line with `serverUrl` + `guid`, calls BlueBubbles ping and iMessage availability; updates DB.

| Test Case | Setup | Expected |
|-----------|--------|----------|
| One line down | Mock BlueBubbles unavailable for one line | Campaigns using that line: pause or use other lines; healthy lines continue. |
| All lines down | All lines fail health check | Campaigns pause, `status` → PAUSED or equivalent. |

**MongoDB (`lines`):**
- `healthCheck.status`: `healthy` | `down`
- `healthCheck.consecutiveFailures`
- When marking healthy: `healthCheck.sendEmailOnNextDown: true` (see workspace rule).

**TestSprite output:** Health-check response (or worker/log behavior), line docs before/after, campaign statuses, Pass/Fail.

---

## 4. Campaign Recovery After Line Restoration

**Endpoint:** `PUT /api/campaigns` with `action: 'resume'`

**Request:**
```json
{
  "campaignId": "<id>",
  "action": "resume"
}
```

| Test Case | Expected |
|-----------|----------|
| Resume paused campaign after lines recover | Campaign resumes; no duplicate sends to contacts who already received messages. |

**MongoDB (`messages`):**
- No duplicate `(leadId, campaignId)` for same step.
- `status`: `queued` → `sent` / `delivered` (or `failed` with retry path).

**TestSprite output:** PUT request/response, message counts and sample docs before/after, Pass/Fail.

---

## 5. Message Tracking & Stats

**Endpoint:** `GET /api/campaigns/[id]` (campaign detail includes stats)

**Response:** Campaign object including `stats`: `pending`, `sent`, `delivered`, `opened`, `replied`, `failed`, etc.

| Test Case | Setup | Expected |
|-----------|--------|----------|
| Mock events | 10 sent, 8 delivered, 3 opened, 1 replied (via webhooks or DB) | GET campaign stats match: e.g. sent=10, delivered=8, opened=3, replied=1. |

**MongoDB (`messages`):**
- `deliveredAt`, `openedAt`, `repliedAt` set where applicable.
- Campaign `stats` aggregate matches message counts.

**TestSprite output:** GET response (stats), message aggregates from DB, Pass/Fail.

---

## 6. Retry Failed Messages

**Endpoint:** `PUT /api/campaigns` with `action: 'retry_failed'`

**Request:**
```json
{
  "campaignId": "<id>",
  "action": "retry_failed"
}
```

| Test Case | Setup | Expected |
|-----------|--------|----------|
| 10 contacts, 3 failed | Create campaign, simulate 3 failures | Retry: only 3 messages retried; 7 successful contacts not re-sent. |

**MongoDB (`messages`):**
- `retryCount` increments for retried messages.
- `status`: `failed` → `queued` → `sent`.

**TestSprite output:** PUT request/response, message docs (retryCount, status) before/after, Pass/Fail.

---

## Correct API Endpoints (Reference)

| Spec / Doc | Actual endpoint |
|------------|-----------------|
| Lead import | `POST /api/leads` |
| Campaign create | `POST /api/campaigns` |
| Campaign get (stats) | `GET /api/campaigns/[id]` |
| Campaign resume | `PUT /api/campaigns` body `{ campaignId, action: 'resume' }` |
| Campaign retry failed | `PUT /api/campaigns` body `{ campaignId, action: 'retry_failed' }` |
| Line health check | `GET /api/health-check` |
| Lists | `GET /api/lists`, `POST /api/lists` |
| Lines | `GET /api/lines` |

---

## Environment (Dev)

- **API Base URL:** https://dev.tuco.ai  
- **MongoDB:** `tuco-ai-dev` (e.g. `MONGODB_URI` as provided).  
- **Test lines:** Foxwell, Goforbg (use real `lineIds` from `GET /api/lines` after login).

---

## Success Criteria (Summary)

- All listed API endpoints respond with expected status and body.
- MongoDB state matches expected after each operation (counts, status, no duplicate messages).
- Webhooks (e.g. Webhook.site) receive correct payloads when events fire (dev only; not local).
- No duplicate messages in DB; rate limits enforced; campaign pauses when all lines down; resume without duplicates.

---

## TestSprite-Style Output Format

For each test, output:

1. **API request:** method, URL, headers (no secrets), body (sanitized).
2. **API response:** status, body.
3. **MongoDB state:** before/after (relevant collections and fields).
4. **Webhook payloads:** if applicable (e.g. message events).
5. **Verdict:** Pass / Fail.
6. **Edge cases or bugs:** short note if any.
