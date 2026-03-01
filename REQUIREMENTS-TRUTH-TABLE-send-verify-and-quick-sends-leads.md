# Requirements Truth Table: Send Verification Latency + Quick Sends Leads in UI

## Parallel verify: does it exist for send via API?

**Yes.** Send-via-API uses **parallel verify** in both the app and the worker.

| Path | Uses parallel verify? | Code |
|------|------------------------|------|
| **App** (POST /api/messages → createMessage → processPendingMessage) | Yes | `app/src/lib/messageSender.ts`: text-only and text-first call `runSendWithParallelVerify(sendMessageViaAPI, verifyDeliveryWithMessageQuery, { delayMs: 2000 })` (lines 1506, 1432). |
| **Worker** (POST /api/leads/send → worker sync, or queued job) | Yes | `workers/src/services/scheduled-messages.ts`: `runSendWithParallelVerify(this.sendMessageViaAPI, verifyDeliveryWithMessageQuery, { delayMs: PARALLEL_VERIFY_START_DELAY_MS })` (lines 3972, 1731, 2060). |
| **Unibox** (send-attachments.ts) | Yes | `app/src/app/actions/send-attachments.ts`: `runSendWithParallelVerify(..., { delayMs: PARALLEL_VERIFY_START_DELAY_MS })` (lines 352, 474). |

**How it works:** Send (message/text) and verify (message/query) run in parallel. Verify starts after a **2s** delay. We return as soon as **either** (1) send returns success, or (2) verify finds the message delivered. So we do **not** wait for the full 60s send timeout and then 90s verify; we typically return when verify finds it. With a **2s** poll interval: 2s + 2s = **4s**, 2+4=**6s**, 2+6=**8s**. If verify never finds it, we still wait until both finish (worst case ~90s for verify window).

---

## Requirement 1: iMessage API / Add Lead to Campaign — Slow Response (Verify Polling)

| Scenario | Current behavior | Desired behavior | Truth |
|----------|------------------|------------------|--------|
| Send iMessage via API (n8n node, POST /api/messages) | App/worker waits for delivery verification before returning; message/text often times out (~60s), then we poll message/query up to **90s** every **15s** → total wait can be **60 + 90 = 150s** | Response returns sooner; message is already sent, verification can be async or shorter | Poll **duration** and **interval** are too long for API callers |
| Add lead to campaign (same flow) | Same sync verify flow; caller blocks until verify or timeout | Same as above | Same |
| Normal text (no attachments) | `verifyDeliveryWithMessageQuery`: **pollWindowMs = 90_000** (90s), **pollIntervalMs = 15_000** (15s). Plus message/text timeout ~60s before verify starts | Shorter window and/or interval so API returns in &lt;30s when possible; or return 202 and verify in background | Reduce window/interval or make verify async for API |

**Code locations (normal text):**

- **Poll window (duration):** `app/src/lib/bluebubblesVerifyDelivery.ts` — `pollWindowMs = options.pollWindowMs ?? 90000`, `pollIntervalMs = options.pollIntervalMs ?? 15000`.
- **Callers (defaults):**
  - **Unibox / app sync fallback:** `app/src/app/actions/send-attachments.ts`: `UNIBOX_QUICK_SEND_VERIFY_POLL_WINDOW_MS = 90000`, `UNIBOX_QUICK_SEND_VERIFY_POLL_INTERVAL_MS = 15000`.
  - **App messageSender (sync fallback):** `app/src/lib/messageSender.ts`: `SYNC_FALLBACK_VERIFY_POLL_WINDOW_MS = 90000`, `SYNC_FALLBACK_VERIFY_POLL_INTERVAL_MS = 15000`.
- **Worker** uses same verify helpers (worker codebase); if app calls worker sync (process-message-sync), worker does the verify and blocks the HTTP response.

**Summary:** For normal texts we poll for **90 seconds**. Current default interval is **15s** (6 polls); recommended is **2s** (45 polls in 90s). With **parallel verify**, we return as soon as verify finds the message — with 2s that’s typically **4s, 6s, 8s**. If the message appears slowly we still wait up to the full verify window (90s).

### Ideal polling (decision)

**Constraint:** message/text often times out; we must verify via message/query. The “~150/line/day” figure refers to **total messages** (sends) per line per day, not to the message/query API. There is no documented rate limit for message/query, so polling more often (e.g. 2s) is fine. We want the API to return as soon as verify finds the message.

**Decision: poll every 2 seconds.**

| Option | Poll interval | Polls in 90s | Typical first success | Rate-limit impact |
|--------|----------------|--------------|------------------------|-------------------|
| Current | 15s | 6 | 2+15=17s, 2+30=32s | Low |
| **Ideal** | **2s** | **45** | **2+2=4s, 2+4=6s, 2+6=8s** | No documented BlueBubbles limit; 2s is fine |
| Alternative | 5s | 18 | 2+5=7s, 2+10=12s | Lower |

**Recommendation:** Use **2s** poll interval for verify (was 15s). Keep **90s** window so we don’t give up too early. Same behavior everywhere (app sync, worker, Unibox) so one set of env vars. At 2s we get the snappiest response (~4–8s typical). No official BlueBubbles rate limit for message/query is documented; if you observe throttling in practice, override via env to 5000 (5s).

**Env vars to change (defaults):**

- `UNIBOX_REPLY_VERIFY_POLL_INTERVAL_MS` (worker, Unibox): default **2000** (was 15000)
- `SYNC_FALLBACK_VERIFY_POLL_INTERVAL_MS` (app messageSender): default **2000** (was 15000)
- App `send-attachments.ts`: `UNIBOX_QUICK_SEND_VERIFY_POLL_INTERVAL_MS`: default **2000** (was 15000)

**Code locations to update:**

- `app/src/lib/bluebubblesVerifyDelivery.ts`: default `pollIntervalMs ?? 15000` → `pollIntervalMs ?? 2000` (callers can still override via env).
- `app/src/lib/messageSender.ts`: `SYNC_FALLBACK_VERIFY_POLL_INTERVAL_MS` default 2000.
- `app/src/app/actions/send-attachments.ts`: `UNIBOX_QUICK_SEND_VERIFY_POLL_INTERVAL_MS` default 2000 (or use same env `UNIBOX_REPLY_VERIFY_POLL_INTERVAL_MS`).
- `workers/src/services/scheduled-messages.ts`: `UNIBOX_REPLY_VERIFY_POLL_INTERVAL_MS` default 2000.

---

## Requirement 2: Quick Sends Leads Not Visible in Leads UI

| Scenario | Current behavior | Desired behavior | Truth |
|----------|------------------|------------------|--------|
| Lead created via "Send iMessage" node (n8n) or POST /api/messages (no leadId) | Lead is created via `createQuickSendLead` and added to **Quick Sends** list; visible in Unibox | Same lead visible in **Leads** page (list table) | User cannot find these leads in UI; they appear in Unibox |
| GET /api/leads (no listId) | Query = `{ workspaceId: orgId }` → returns **all** leads including Quick Sends | No change needed | API already returns all leads when "All Lists" is selected |
| GET /api/leads?listId=&lt;id&gt; | Query = `{ workspaceId: orgId, listId }` → returns leads in that list (e.g. Quick Sends) | No change needed | API correctly filters by list |
| Lists dropdown (Filter by list) | Lists from GET /api/leads (all lists for workspace); Quick Sends list created on first use by API | Quick Sends list must appear and show correct count | If user never refreshed after first API send, **Quick Sends** list might be missing from dropdown until refresh; lead count is updated via `$inc` when lead is created |
| Default view (selectedList) | `selectedList` initial state = `''` → "All Lists" | No change | Default shows all leads |
| Lead source | Quick Sends leads have `source: 'api'`; LeadData in UI allows various sources | UI may not show or filter by `api`; if UI filters by source, API leads could be hidden | Need to ensure no client-side or server-side filter excludes `source: 'api'` |

**Conclusion:**  
- Backend: Leads are saved with `listId` = Quick Sends list; list is created if missing; `leadCount` is incremented.  
- Visibility: With **"All Lists"** selected, GET /api/leads returns all leads (including Quick Sends). If the **Quick Sends** list was created only after the user loaded the Leads page, the dropdown won’t include "Quick Sends" until the next load (e.g. refresh or revisit).  
- **Fix direction:** (1) Ensure no filter excludes `source: 'api'`. (2) Refetch lists (and optionally leads) when the Leads page is focused/visible so "Quick Sends" appears after an API-created lead. (3) Optionally show "Quick Sends" in the list filter even if count is 0 (e.g. after creating the list on first send).

### Why the list isn’t visible (proof)

**Observed:** Lead created via API (Send iMessage node) is visible in **Unibox** but not in **Leads** page.

**Proof:**

1. **Unibox**  
   Unibox does **not** use the Lists or Leads list API for the conversation list. It loads conversations from **messages** (`/api/unibox/conversations` or similar), grouping by recipient. So as soon as a message exists for that recipient (created by the API send), Unibox shows that conversation. The lead exists in DB and has a message → Unibox shows it.

2. **Leads page**  
   The Leads page shows `leads` and `lists` from React state. That state is filled **only** by `loadLeadsAndLists()`, which:
   - Calls `GET /api/leads?page=…&limit=50` (and optionally `listId`, `search`).
   - Sets `setLeads(data.leads)` and `setLists(data.lists)`.

   **When is `loadLeadsAndLists` called?**  
   In `app/src/app/leads/page.tsx`:
   - `useEffect(() => { … loadLeadsAndLists(); }, [page, selectedList]);` — when **page** or **selectedList** changes.
   - `useEffect` with debounced **searchTerm** — when search changes (after 300ms).

   There is **no** `visibilitychange`, `focus`, or route/segment listener. So when the user:
   1. Opens Leads (initial load: lists + leads),
   2. n8n (or another tab) creates a lead via API → Quick Sends list and lead are created in DB,
   3. User goes to Unibox → Unibox fetches messages/conversations → sees the new conversation,
   4. User returns to Leads tab,

   …the Leads component is still mounted with **old** `leads` and `lists` state. No refetch runs, so the new Quick Sends list and the new lead never appear until the user changes page, list filter, or search (or does a full refresh).

3. **API correctness**  
   When `listId` is **not** sent, GET /api/leads uses `query = { workspaceId: orgId }` and returns **all** leads (`app/src/app/api/leads/route.ts`). Lists are `find({ workspaceId: orgId })`. So once the Leads page **does** refetch, it would get the Quick Sends list and the new lead. The bug is **stale client state**, not the API.

**Root cause:** Leads page **never refetches when the user returns to the tab**. Lists and leads are refetched only when `page`, `selectedList`, or `searchTerm` changes.

**Fix:** Refetch when the window/tab becomes visible again (e.g. `document.addEventListener('visibilitychange', …)` and call `loadLeadsAndLists()` when `document.visibilityState === 'visible'`). Then returning from Unibox (or any other tab) will show the latest lists and leads, including Quick Sends and API-created leads.

### API-created lead: firstName / lastName not split

When creating a lead via API (e.g. Send iMessage node) with `recipientName: "Tuco test"`, we were storing the whole string in `firstName` and leaving `lastName` null. **Fix:** In `createQuickSendLead` (`app/src/lib/quickSendsList.ts`), split `recipientName` on the **first space**: `firstName` = part before first space, `lastName` = rest (trimmed). Examples: `"Tuco test"` → firstName `"Tuco"`, lastName `"test"`; `"Tuco"` → firstName `"Tuco"`, lastName undefined.

---

## Summary

| # | Requirement | Root cause | Fix direction |
|---|-------------|------------|----------------|
| 1 | Long response time for send iMessage / add to campaign | Poll interval 15s → only 6 polls in 90s; first success often at 17s, 32s, 47s | **Ideal:** Poll interval **2s** (45 polls in 90s); first success typically 4s, 6s, 8s. Update defaults in app + worker. |
| 2 | Quick Sends leads not visible in Leads UI | **Proven:** Leads page refetches only when `page`, `selectedList`, or `searchTerm` changes. No refetch on tab focus. So after API creates lead, user sees it in Unibox (Unibox fetches messages); when they return to Leads tab, state is stale. | Refetch on visibility: `visibilitychange` → when `document.visibilityState === 'visible'` call `loadLeadsAndLists()`. |
