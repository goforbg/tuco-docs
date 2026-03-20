# API documentation truth table

This table aligns the **Mintlify docs** and **OpenAPI** with the **real app APIs** (and N8N usage). Use it to verify request/response and error docs.

---

## 1. Send iMessage / SMS / Email — `POST /api/messages`

| Item | N8N / Before | Real API (app) | Docs / OpenAPI now |
|------|----------------|----------------|---------------------|
| **Auth** | Bearer token | `authenticateRequest` (API key or Clerk) | Bearer in OpenAPI + MDX |
| **Request: message** | Required in old docs | Optional if `attachmentUrls` provided; either message or attachmentUrls required | OpenAPI: no longer `required: ["message"]`; MDX says "Required unless attachmentUrls" |
| **Request: attachmentUrls** | Not in old OpenAPI | Supported, max 2, validated | Added to OpenAPI schema + send-message.mdx |
| **Response: success** | Yes | Yes | Yes |
| **Response: message** | Object with _id, status, etc. | Full IMessage | Yes |
| **Response: leadId** | Used in N8N | Always in MessageSendResponse | Added to OpenAPI + MDX example |
| **Response: ghlContactId, ghlLocationId, hsPortalId, hsContactId** | Used in N8N | From getIntegrationIdsForApiResponse | Added to OpenAPI SendMessageResponse + MDX |
| **Response: status** | sent, pending, etc. | sent \| pending \| scheduled \| fallback \| failed | Documented |
| **Response: duplicateOnly, existingLeadIds, leadIds, duplicateMessage** | — | Returned when duplicate lead used (200) | OpenAPI 200 example + MDX |
| **Status 200** | Not documented | Returned when duplicate lead used | Documented (200 + duplicateOnly) |
| **Error: empty message** | "Missing required field: message" | "Message and attachmentUrls cannot both be empty" | Fixed in OpenAPI + errors.mdx |

---

## 2. Create / import leads — `POST /api/leads`

| Item | N8N / Before | Real API (app) | Docs / OpenAPI now |
|------|----------------|----------------|---------------------|
| **Request: leads** | Array of leads | Required (or `body` as alias) | OpenAPI + create.mdx |
| **Request: listId** | Optional in N8N | Optional; omitted → Quick Sends list | MDX: "Omit to use Quick Sends" |
| **Request: source, ghlLocationId** | In N8N body | Accepted | In OpenAPI CreateLeadsRequest |
| **Response 201** | savedCount, leadIds, listId | + leadId, leadListIds, ghlContactId, ghlLocationId, hsPortalId, hsContactId, duplicates? | OpenAPI + MDX updated |
| **Response 200 (all duplicates)** | Old docs said 409 | **200** with duplicateOnly, existingLeadIds, leadIds, duplicates | **Fixed**: create.mdx now says 200 (not 409); OpenAPI has CreateLeadsDuplicateResponse |
| **Error 404** | List not found | "List not found. The specified listId..." | Documented |

---

## 3. Add lead to campaign — `POST /api/campaigns/{id}/leads`

| Item | N8N / Before | Real API (app) | Docs / OpenAPI now |
|------|----------------|----------------|---------------------|
| **Request: leadId** | String (existing lead) | Optional if `lead` provided | OpenAPI: leadId or lead |
| **Request: lead** | Object with firstName, lastName, email, phone | Creates lead and adds to campaign | OpenAPI AddLeadToCampaignRequest |
| **Response 201** | success, leadId, campaignId, messageCount, message | + ghlContactId, ghlLocationId, hsPortalId, hsContactId | OpenAPI + campaign-add-lead.mdx |
| **Response 200 (already in campaign)** | Not in old docs | alreadyInCampaign: true, messageCount: 0 | OpenAPI 200 example + MDX |
| **Errors 400, 404** | — | Either leadId or lead required; campaign/lead not found | OpenAPI responses |

---

## 4. Get all lines — `GET /api/lines/by-user`

| Item | N8N / Before | Real API (app) | Docs / OpenAPI now |
|------|----------------|----------------|---------------------|
| **Auth** | Bearer | authenticateRequest | OpenAPI + get-lines.mdx |
| **Response** | `{ lines: [...] }` with usage, healthCheck | sanitizeLineForFrontend + usage + healthCheck | OpenAPI GetLinesResponse + get-lines.mdx |
| **Doc page** | Only in lines.mdx (overview) | — | New endpoint page get-lines.mdx + OpenAPI path |

---

## 5. Search / get leads — `GET /api/leads`

| Item | N8N / Before | Real API (app) | Docs / OpenAPI now |
|------|----------------|----------------|---------------------|
| **Query: id** | Not in old OpenAPI | Single lead by ID -> `{ lead }` | OpenAPI GET /api/leads params + GetLeadsResponse oneOf |
| **Query: search, listId, page, limit** | In N8N (search=goforbg) | Supported | OpenAPI params + get.mdx |
| **Response (list)** | leads, lists, pagination | Same | OpenAPI + get.mdx |
| **Response (single)** | — | `{ lead: leadJson }` | OpenAPI oneOf |
| **Error 503** | — | Search timeout | OpenAPI 503 |

---

## 6. Check replies — `GET /api/replies`

| Item | N8N / Before | Real API (app) | Docs / OpenAPI now |
|------|----------------|----------------|---------------------|
| **Query: leadId** | In N8N | Optional; 404 if lead not found | OpenAPI + get-replies.mdx |
| **Query: campaignId, recipientEmail, recipientPhone, page, limit** | — | Supported, limit max 100 | OpenAPI params |
| **Response** | replies[], total, page, limit, totalPages | Same; each reply: leadId, messageId, respondedAtUtc, name, parentMessages[] | OpenAPI GetRepliesResponse + get-replies.mdx |
| **Doc page** | Only in message-webhooks (webhook payload) | — | New get-replies.mdx + OpenAPI path |

---

## 7. Send SMS using GHL as fallback (external)

| Item | N8N | Docs |
|------|-----|------|
| **Trigger** | Webhook "Failed & Fallback" (message.fallback) | failures-and-fallbacks.mdx: section "Send SMS via GoHighLevel (GHL) as fallback" |
| **GHL endpoint** | POST services.leadconnectorhq.com/conversations/messages | Same; body params: locationId, contactId, message, type: SMS, lastMessageDate |
| **Payload source** | body.ghlContactId, body.ghlLocationId, body.message.message | Documented from webhook body |

---

## Summary of changes

### What was wrong or missing before

1. **Send message**: Response lacked `leadId`, `ghlContactId`, `ghlLocationId`; error for empty body was "Missing message" instead of "Message and attachmentUrls cannot both be empty"; `attachmentUrls` not documented; 200 for duplicate lead not documented.
2. **Create leads**: All-duplicates case was documented as 409; real API returns 200 with duplicateOnly and existingLeadIds. Response lacked `leadId`, integration IDs; `listId` optional (Quick Sends) and `body` alias not clear.
3. **Add lead to campaign**: Response lacked integration IDs; 200 "already in campaign" case not documented.
4. **Get lines**: No dedicated OpenAPI path or endpoint MDX; only overview in lines.mdx.
5. **Get leads**: Single-lead by `id` query not in OpenAPI; 503 timeout not documented.
6. **Check replies**: No OpenAPI path and no dedicated endpoint MDX.
7. **GHL fallback**: No doc for "when you get message.fallback, call GHL to send SMS."

### What was added

- **OpenAPI**: Paths for GET /api/leads, GET /api/lines/by-user, GET /api/replies, POST /api/campaigns/{id}/leads; all with correct request/response/error schemas and examples.
- **Mintlify**: `openapi` in docs.json for API Reference tab so Mintlify generates **interactive Try it** pages from the spec.
- **New MDX**: get-replies.mdx, get-lines.mdx.
- **Updated MDX**: send-message (response example, attachmentUrls, 200 duplicate), create (200 all duplicates, listId optional, response fields), campaign-add-lead (200 alreadyInCampaign, integration IDs), errors (400 message/attachments empty).
- **New section**: "Send SMS via GoHighLevel (GHL) as fallback" in failures-and-fallbacks.mdx.

### Request/response body changes (real API vs old docs)

| Endpoint | Request change | Response change |
|----------|----------------|------------------|
| POST /api/messages | message not required when attachmentUrls present; attachmentUrls, attachmentNames added | leadId, ghlContactId, ghlLocationId, hsPortalId, hsContactId, reason, scheduledDate, duplicateOnly, existingLeadIds, leadIds, duplicateMessage, error; 200 for duplicate |
| POST /api/leads | listId optional; body alias for leads; ghlLocationId | leadId, leadListIds, ghlContactId, ghlLocationId, hsPortalId, hsContactId; 200 + duplicateOnly when all duplicates |
| POST /api/campaigns/{id}/leads | lead object for create-and-add | ghlContactId, ghlLocationId, hsPortalId, hsContactId; 200 + alreadyInCampaign |
| GET /api/leads | id for single lead | Single: `{ lead }`; list: leads, lists, pagination; 503 on timeout |
| GET /api/lines/by-user | — | lines[] with usage, healthCheck (no sensitive fields) |
| GET /api/replies | leadId, campaignId, recipientEmail, recipientPhone, page, limit | replies[], total, page, limit, totalPages; each reply shape as in webhook data.reply |

---

## Interactive docs (Mintlify)

- **docs.json**: `"openapi": "api-reference/openapi.json"` added to the API Reference tab so Mintlify builds interactive API reference pages with **Try it** against `https://app.tuco.ai`.
- **Auth**: Bearer token; configured in docs.json `api.mdx.auth.method: "bearer"`.
- All six core APIs (send message, create leads, get leads, get lines, get replies, add lead to campaign) are in the OpenAPI spec and will show up in the reference with request/response examples and error responses.
