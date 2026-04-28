---
title: "iMessage Double-Send Guard & Phone-Sharing Behavior"
description: "How Tuco handles multiple contacts (e.g. coworkers) sharing one phone number on send-imessage, and how content-based double-send suppression works."
---

# iMessage Double-Send Guard

> **Status:** Spec, in development on `spec/imessage-double-send-guard` branch (tuco-app, dev). Not yet shipped.

## TL;DR

- **One phone = one Tuco lead per workspace.** This is the contract. Phone is the unique key.
- When the same phone arrives via send-imessage with a **different name or different GHL/HubSpot contact ID**, Tuco merges that name + CRM ID onto the existing lead's `aliases[]` instead of silently dropping it.
- A **60-second content-based double-send guard** prevents the same message body from being sent twice to the same phone in rapid succession (the typical "5 GHL workflows fire at once for 5 contacts sharing a desk phone" case).
- The first send goes through. The other 4 return `{ skipped: true, reason: 'duplicate_send_recent' }` with a pointer to the original message — your CRM workflow can branch on this.

## Why this exists

Real-world scenario: a B2B sender uploads a sales sequence to GHL. Five contacts at the same company share one office phone (front desk, shared cell, etc.). When the GHL workflow fires, all five enrollments call Tuco's send-imessage endpoint within seconds. Without protection, the office phone receives the same SMS five times.

Tuco's job is to deliver one message to the human, while preserving the audit trail that all five CRM contacts were involved.

## How phone-sharing is handled

### Lead resolution

| Incoming payload | Result |
|---|---|
| Phone X, never seen before | New lead created with the incoming `firstName` / `lastName` / `ghlContactId` |
| Phone X, seen before, same name + same GHL contactId | Existing lead reused, no changes |
| Phone X, seen before, **different name or different GHL contactId** | Existing lead reused. New name + GHL contactId pushed onto `aliases[]` (no duplicates). `integrationIds.ghlRecordIds[]` extended via `$addToSet`. `firstName`/`lastName` on the lead are **not** changed (first writer wins, intentional). |

### Conversation threading

One conversation thread per `(workspaceId, recipientPhone)`. Doesn't matter how many CRM contacts share the phone — your team sees one thread, mirroring how iMessage / SMS actually work at the carrier level.

## Double-send suppression

### Trigger

Before inserting a new outbound Message, Tuco checks:

```
Has this exact (workspaceId, recipientPhone, sha256(message body)) been sent
in the last 60 seconds?
```

If yes → suppress, return `{ skipped: true, reason: 'duplicate_send_recent' }`.

### What "same body" means

The hash is computed on a normalized version of the message:

- Lowercased
- Trimmed
- Internal whitespace collapsed to single spaces

So `"Hi there!"` and `"  hi  THERE!  "` collapse to the same hash. Different content (e.g. `"Hi"` vs `"Hi there"`) produces different hashes and both go through.

### Window

60 seconds, hardcoded for now. If your use case needs a different window, let us know and we'll make it a workspace setting.

### What gets suppressed vs. allowed

| Scenario | Behavior |
|---|---|
| 5 GHL workflows fire send-imessage for 5 contacts sharing phone, same template renders to identical body | 1 sent, 4 suppressed |
| Same phone, two different messages within 60s ("Hi" then "Are you there?") | Both sent |
| Same phone, same message, sent again 65 seconds later | Both sent (window expired) |
| Same phone, same message, dispatched on a manual retry within 60s | Suppressed — use `Idempotency-Key` header to bypass for legitimate retries |

## Response shape

### Successful send

```json
{
  "success": true,
  "messageId": "65f...",
  "leadId": "65f...",
  "skipped": false
}
```

### Suppressed double-send

```json
{
  "success": true,
  "skipped": true,
  "reason": "duplicate_send_recent",
  "originalMessageId": "65f...",
  "leadId": "65f...",
  "secondsSinceOriginal": 12
}
```

Note: HTTP status is **200**, not an error. Suppression is intentional behavior, not a failure.

## GHL workflow integration

When you use the Tuco GHL plugin's `send-imessage` workflow action, the response surfaces the suppression to your GHL workflow so you can branch:

- On `skipped: true`, the workflow action returns success but with a note: `"Skipped — same message sent to this phone Xs ago via GHL contact <id>"`.
- Your downstream workflow steps can read this and decide whether to log the suppression on the GHL contact, skip subsequent enrollments, etc.

## Aliases — what's stored

When a second name / second CRM contact lands on the same phone, the existing lead gets:

```ts
aliases: [
  {
    name: "Bob Johnson",
    firstName: "Bob",
    lastName: "Johnson",
    ghlRecordId: "ghl-contact-id-2",
    ghlLocationId: "loc-1",
    firstSeenAt: "2026-04-29T..."
  },
  // ... up to N more
]

integrationIds: {
  ghlRecordId: "ghl-contact-id-1",          // first writer (display)
  ghlRecordIds: ["ghl-contact-id-1", "ghl-contact-id-2", ...],   // all
}
```

This is read-only metadata — not exposed in the UI yet. It exists for audit, future inbound reply attribution, and CRM round-tripping.

## Inbound replies

Inbound replies still attribute to the single Tuco lead (one per phone). The original GHL/HubSpot contact that started the conversation gets the reply logged via existing CRM activity sync. Multi-CRM-contact attribution on replies is **not** in scope for this change.

## Loki events

Searchable in Grafana with the prefix `imessage.send.*`:

- `imessage.send.alias_added` — a new name / CRM contact was appended to an existing lead
- `imessage.send.suppressed_duplicate` — a send was suppressed by the 60s content guard
- `imessage.send.delivered` — existing event, unchanged

## What's NOT changing

- The `(workspaceId, contactIdentifiers)` unique index on Lead. Phone stays unique.
- Conversation threading (one thread per phone).
- Existing `Idempotency-Key` header behavior for exact-request replay protection. The new content-hash guard is **in addition to**, not a replacement for, this.
- HubSpot push-back (uses lead's display `firstName`/`lastName`, not aliases).

## See also

- [`features/send-messages`](/features/send-messages)
- [`features/replies`](/features/replies)
- [`lead-uniqueness-truth-table`](/lead-uniqueness-truth-table)
- [`IMESSAGE_SEND_WITHOUT_LEAD_TRUTH_TABLE`](/IMESSAGE_SEND_WITHOUT_LEAD_TRUTH_TABLE)
