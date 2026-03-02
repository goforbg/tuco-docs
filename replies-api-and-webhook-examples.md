# Replies: API + Webhook Examples (short form)

**ADHD short form:**
- **GET /api/replies** → list who replied; optional `?campaignId=` filter; pagination `page`, `limit`; times in UTC.
- **message.reply webhook** → fired when a reply is recorded; always has `repliedAtUtc`; `parentMessages` = last 2 we sent (campaign-scoped if in campaign, else workspace).
- **Campaign reply** → webhook has `campaignId` / `campaignName`; `parentMessages` = last 2 **campaign** messages to that lead.
- **Individual reply** → reply to a one-off message; `parentMessages` = last 2 outbound to that lead (any); may have no `campaignId`.
- **Inbound only (no original)** → reply with no matched original; we may still have `leadId`; `parentMessages` present only if we have outbound to that lead, else **omitted**.

---

## 1. GET /api/replies

**Request (workspace-wide, page 1):**
```http
GET /api/replies?page=1&limit=20
```

**Request (filter by campaign):**
```http
GET /api/replies?campaignId=674a1b2c3d4e5f6789&page=1&limit=20
```

**Response (200):**
```json
{
  "replies": [
    {
      "leadId": "667f1f77bcf86cd799439012",
      "messageId": "674abc123def456",
      "respondedAtUtc": "2025-03-02T14:59:58.000Z",
      "recipientEmail": "jane@example.com",
      "recipientPhone": null,
      "name": "Jane Doe",
      "parentMessages": [
        {
          "messageId": "674def000111",
          "message": "Hi Jane, we have a limited offer...",
          "sentAtUtc": "2025-03-01T10:00:00.000Z",
          "batchId": "campaign_674a1b2c_0",
          "stepIndex": 0
        }
      ]
    }
  ],
  "total": 42,
  "page": 1,
  "limit": 20,
  "totalPages": 3
}
```

---

## 2. Webhook: message.reply (campaign message)

Reply matched to a **campaign** outbound. `parentMessages` = last 2 we sent in **that campaign** to that lead.

```json
{
  "event": "message.reply",
  "timestamp": "2025-03-02T15:00:05.000Z",
  "workspaceId": "org_2abc",
  "messageId": "674reply001",
  "message": {
    "message": "Yes, I'm interested!",
    "direction": "received",
    "recipientEmail": "jane@example.com",
    "leadId": "667f1f77bcf86cd799439012"
  },
  "originalMessageId": "674sent001",
  "originalMessage": { "message": "Hi Jane, we have an offer...", "batchId": "campaign_674a1b2c_0" },
  "leadId": "667f1f77bcf86cd799439012",
  "lead": { "firstName": "Jane", "lastName": "Doe", "email": "jane@example.com" },
  "campaignId": "674a1b2c3d4e5f6789",
  "campaignName": "Q1 Outreach",
  "repliedAtUtc": "2025-03-02T14:59:58.000Z",
  "parentMessages": [
    { "messageId": "674sent001", "message": "Hi Jane, we have an offer...", "sentAtUtc": "2025-03-01T10:00:00.000Z", "batchId": "campaign_674a1b2c_0", "stepIndex": 0 }
  ]
}
```

---

## 3. Webhook: message.reply (individual message)

Reply to a **one-off / non-campaign** message. `parentMessages` = last 2 outbound to that lead in the workspace (any). No `campaignId` (or campaign not tied to this reply).

```json
{
  "event": "message.reply",
  "timestamp": "2025-03-02T15:01:00.000Z",
  "workspaceId": "org_2abc",
  "messageId": "674reply002",
  "message": {
    "message": "Got it, thanks!",
    "direction": "received",
    "recipientPhone": "+12025551234"
  },
  "originalMessageId": "674sent002",
  "originalMessage": { "message": "Here’s the info you asked for.", "batchId": null },
  "leadId": "667f1f77bcf86cd799439013",
  "lead": { "firstName": "Bob", "lastName": "Smith" },
  "campaignId": null,
  "campaignName": null,
  "repliedAtUtc": "2025-03-02T15:00:55.000Z",
  "parentMessages": [
    { "messageId": "674sent002", "message": "Here's the info you asked for.", "sentAtUtc": "2025-03-02T14:50:00.000Z" }
  ]
}
```

---

## 4. Webhook: message.reply (inbound only — no original, no parent messages)

Reply with **no matched original** (inbound-only). We may still resolve `leadId`. If we have no outbound to that lead, **`parentMessages` is omitted**.

```json
{
  "event": "message.reply",
  "timestamp": "2025-03-02T15:02:00.000Z",
  "workspaceId": "org_2abc",
  "messageId": "674reply003",
  "message": {
    "message": "Hi, I want to learn more.",
    "direction": "received",
    "recipientPhone": "+12025559999"
  },
  "originalMessageId": null,
  "originalMessage": null,
  "leadId": "667f1f77bcf86cd799439014",
  "lead": { "firstName": "Alex", "lastName": "Lee" },
  "campaignId": null,
  "campaignName": null,
  "repliedAtUtc": "2025-03-02T15:01:58.000Z"
}
```

No `parentMessages` key → no outbound to this lead yet (or we didn’t have a lead); `repliedAtUtc` is still always present.
