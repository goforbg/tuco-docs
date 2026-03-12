# Log iMessage SENT — direct GHL API

Create/update a GHL conversation so the **outbound** message appears in the thread. No plugin.

---

## Request

```
POST https://services.leadconnectorhq.com/conversations/
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
| `locationId` | string | ✅ | GHL location ID |
| `contactId` | string | ✅ | GHL contact ID |
| `lastMessageBody` | string | ✅ | Message text you sent |
| `lastMessageType` | string | ✅ | `"TYPE_SMS"` |
| `lastMessageDate` | string | ✅ | Unix ms (e.g. `"1710331200000"`) |
| `type` | string | ✅ | `"TYPE_SMS"` |

**Auth:** GHL OAuth access token for the location (scope: `conversations.write`).

---

## Example

```bash
curl -X POST "https://services.leadconnectorhq.com/conversations/" \
  -H "Authorization: Bearer YOUR_GHL_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -H "Version: 2021-07-28" \
  -d '{
    "locationId": "abc123",
    "contactId": "contact456",
    "lastMessageBody": "Hi, this is the message we sent.",
    "lastMessageType": "TYPE_SMS",
    "lastMessageDate": "1710331200000",
    "type": "TYPE_SMS"
  }'
```

**Response:** `conversation.id` in response = conversation ID.
