# Log iMessage RECEIVED — direct GHL API

Add the lead’s **inbound reply** to GHL Conversations. No plugin.

---

## Option A — preferred: Inbound message (contact-based)

```
POST https://services.leadconnectorhq.com/conversations/inbound-messages
```

**Headers**

| Header | Value |
|--------|--------|
| `Authorization` | `Bearer {GHL_ACCESS_TOKEN}` |
| `Content-Type` | `application/json` |
| `Version` | `2021-07-28` |

**Body (JSON)**

| Field | Type | Required | Notes |
|-------|------|----------|--------|
| `contactId` | string | ✅ | GHL contact ID |
| `channel` | string | ✅ | `"sms"` |
| `endpoint` | object | ✅ | `{ "phone": "+1..." }` and/or `{ "email": "..." }` |
| `content` | object | ✅ | `{ "text": "reply text" }` |
| `metadata` | object | no | e.g. `{ "providerMessageId": "...", "timestamp": "2025-03-13T12:00:00.000Z" }` |
| `idempotencyKey` | string | no | Unique per message (e.g. hash of messageId+time) |

**Auth:** GHL OAuth access token (scope: `conversations.write` / `conversations/message.write`).

---

## Example (inbound-messages)

```bash
curl -X POST "https://services.leadconnectorhq.com/conversations/inbound-messages" \
  -H "Authorization: Bearer YOUR_GHL_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -H "Version: 2021-07-28" \
  -d '{
    "contactId": "contact456",
    "channel": "sms",
    "endpoint": { "phone": "+12025551234" },
    "content": { "text": "Yes, interested!" },
    "metadata": {
      "providerMessageId": "msg789",
      "timestamp": "2025-03-13T12:05:00.000Z"
    },
    "idempotencyKey": "a1b2c3d4e5f6"
  }'
```

---

## Option B — fallback: Create conversation with reply as last message

If inbound-messages returns 404/400 (e.g. wrong provider), use:

```
POST https://services.leadconnectorhq.com/conversations/
```

Same headers. Body:

```json
{
  "locationId": "abc123",
  "contactId": "contact456",
  "lastMessageBody": "Yes, interested!",
  "lastMessageType": "TYPE_SMS",
  "lastMessageDate": "1710331500000",
  "type": "TYPE_SMS"
}
```

`lastMessageDate` = reply time in Unix ms.

---

## Option C — conversation exists (400 “already exists”)

Add message to existing conversation:

```
POST https://services.leadconnectorhq.com/conversations/messages/inbound
```

**Headers:** same, but `Version: 2021-04-15`.

**Body:**

| Field | Type | Required |
|-------|------|----------|
| `conversationId` | string | ✅ |
| `type` | string | ✅ `"SMS"` |
| `message` | string | ✅ Reply text |
| `date` | string | ✅ ISO (e.g. `"2025-03-13T12:05:00.000Z"`) |
| `altId` | string | no Your message ID |

---

**Response:** `conversationId` / `messageId` in response.
