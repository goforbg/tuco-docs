# n8n: Tuco Send iMessage + iMessage Received (no GHL app)

Use **n8n + Tuco API key** (and optionally **GHL API key** in n8n for contacts). No GHL plugin.

- **Send iMessage** → Tuco app `POST /api/messages` (Tuco API key).
- **iMessage Received** → n8n Webhook; register its URL in Tuco app (Settings → Webhooks, event **message.reply**).

GHL: use GHL API from n8n (e.g. get contact, get `contactId`/`locationId`) and pass those into the send node if you want GHL Conversations to stay in sync.

---

## Import

Import **`docs/n8n-ghl-send-imessage-and-receive.json`** in n8n. Copy either node into your workflows.

---

## Node 1: Tuco Send iMessage

**Type:** HTTP Request  
**Purpose:** Send an iMessage via the Tuco app API (one request = one message).

### Request

| Field | Value |
|--------|--------|
| **Method** | POST |
| **URL** | `https://app.tuco.ai/api/messages` (or your app URL) |
| **Auth** | In the node: set header `Authorization` = `Bearer YOUR_TUCO_API_KEY`. Or use n8n **Header Auth** credential (name `Authorization`, value `Bearer <key>`) so the key isn’t stored in the workflow. Key from Tuco app → Settings → API keys (starts with `tuco_`). |

### Body (JSON)

```json
{
  "message": "Your message text",
  "recipientPhone": "+12025551234",
  "recipientEmail": "",
  "recipientName": "Optional display name",
  "fromLineId": "",
  "ghlContactId": "",
  "ghlLocationId": ""
}
```

- **message** (required): Text to send.
- **recipientPhone** or **recipientEmail**: At least one required (or use **leadId** to send to an existing lead).
- **fromLineId**: Optional. Omit for round-robin across workspace lines.
- **ghlContactId** / **ghlLocationId**: Optional. If you use GHL and want the thread in GHL Conversations, get these from the GHL API in n8n (e.g. “Get Contact” or your trigger) and pass them here. Tuco will log the outbound message and replies to GHL when these are set.

### Response

- **201**: `{ "message": { "_id": "...", "message": "...", ... } }`
- **4xx/5xx**: `{ "error": "..." }`

---

## Node 2: Tuco iMessage Received

**Type:** Webhook (trigger)  
**Purpose:** Run a workflow when a lead replies to an iMessage (Tuco fires **message.reply** to your URL).

### Setup

1. Add the **Webhook** node to your workflow (or use the imported one).
2. Set path to e.g. `tuco-imessage-received` (or leave default).
3. **Activate** the workflow and copy the **Production** webhook URL (e.g. `https://your-n8n.com/webhook/tuco-imessage-received`).
4. In **Tuco app**: **Settings → Integrations / Webhooks** → Add webhook URL → subscribe to **message.reply**.
5. When a lead replies on iMessage, Tuco POSTs to that URL.

### Payload (from Tuco)

Tuco sends a POST body like:

```json
{
  "event": "message.reply",
  "timestamp": "2025-03-13T12:00:00.000Z",
  "workspaceId": "...",
  "message": "lead's reply text",
  "messageId": "...",
  "leadId": "...",
  "fromLineId": "...",
  "phone": "+1...",
  "firstName": "...",
  "lastName": "...",
  "repliedAtUtc": "2025-03-13T12:00:00.000Z",
  "parentMessages": [],
  "data": {
    "reply": { ... },
    "repliedAtUtc": "...",
    "parentMessages": []
  }
}
```

Use `message` for the reply text, `leadId`, `fromLineId`, `phone`, etc. in the rest of your flow. To **reply back** from n8n, call **Tuco Send iMessage** again with the same `fromLineId` and the recipient phone/email.

---

## GHL in n8n (optional)

If you use GHL and want contacts/threads in sync:

1. In n8n, use **GHL API** (with your GHL API key) to get or create contacts and read **contactId** and **locationId**.
2. Pass **ghlContactId** and **ghlLocationId** into the **Tuco Send iMessage** body. Tuco will log the outbound message and inbound replies to GHL Conversations (when the workspace has GHL configured).
3. No GHL plugin needed; everything is driven from n8n with Tuco API key + optional GHL API key.

---

## Summary

| Node | What it does |
|------|----------------|
| **Tuco Send iMessage** | HTTP Request → Tuco `POST /api/messages` with Tuco API key. Optionally include ghlContactId/ghlLocationId from GHL API. |
| **Tuco iMessage Received** | Webhook; add its URL in Tuco app for event **message.reply**. Run automation when a lead replies. |

No GHL app/plugin required; orchestration is n8n + Tuco API (+ GHL API if you want GHL contacts/threads).
