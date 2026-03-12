# Send SMS (fallback/failed) — direct GHL API

Send an SMS into the contact’s GHL conversation (e.g. after iMessage fallback or failed). No plugin. Triggers/workflows are in GHL; this call only sends the message.

---

## Request

```
POST https://services.leadconnectorhq.com/conversations/messages
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
| `type` | string | ✅ | `"SMS"` |
| `contactId` | string | ✅ | GHL contact ID |
| `locationId` | string | ✅ | GHL location ID |
| `phone` | string | ✅ | E.164 (e.g. `"+12025551234"`) |
| `message` | string | ✅ | SMS body |

**Auth:** GHL OAuth access token for the location (scope: `conversations/message.write`).

---

## Example

```bash
curl -X POST "https://services.leadconnectorhq.com/conversations/messages" \
  -H "Authorization: Bearer YOUR_GHL_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -H "Version: 2021-07-28" \
  -d '{
    "type": "SMS",
    "contactId": "contact456",
    "locationId": "abc123",
    "phone": "+12025551234",
    "message": "Hi, this is the message we tried to send via iMessage."
  }'
```

---

**Note:** To run GHL workflows on fallback/failed (e.g. `tuco_imessage_fallback`), configure those triggers in GHL; this endpoint only sends the SMS into the conversation.
