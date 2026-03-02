# Debugging reply → lead mapping with Loki

When a lead replies and the reply shows in Unibox but **recent activity** or **campaign reply count** don’t show it, the usual cause is the reply message being stored **without `leadId`**. These Loki events let you confirm that and trace the flow.

## Events in the reply flow

| Event | Meaning |
|-------|--------|
| `reply.extract_contact.result` | Phone/email/chatIdentifier extracted from webhook. |
| `reply.original_message.lookup` | Looking for an outbound message we sent to this contact (last 30 days). |
| `reply.original_message.result` | Found (or not) an original message; if found, reply will get that message’s `leadId`. |
| `reply.skipped_no_original_message` | No outbound message found → we take the “without original” path. |
| `reply.branch` | Path taken: `with_original` or `without_original`. |
| `reply.without_original.lead_mapped` | (No original) Lead resolved by recipient; reply will have `leadId`. |
| `reply.without_original.no_lead_mapped` | (No original) No lead found for recipient; reply stored **without** `leadId` → recent activity and campaign reply count will not include it. |
| `reply.with_original.lead_resolved` | (With original but original had no `leadId`) Lead resolved by recipient; reply will have `leadId`. |
| `reply.with_original.no_lead_mapped` | (With original but no `leadId`) No lead found; reply stored without `leadId`. |
| `reply.complete` | Reply recorded; includes `leadId` when present. |
| `reply.webhook.fired` | `message.reply` webhook sent (payload includes `leadId`/`lead` when we have them). |

## Example: incident 2026-03-02 (bg@foxwellpierce.com, messageId 69a553b96fbd952617b0e40c)

To see what happened for that reply in Loki (adjust time range and label matchers to your Loki setup):

```logql
{app="tuco-app"} 
  |~ "69a553b96fbd952617b0e40c|bg@foxwellpierce.com|org_3ABCDbdGyF6w99hikyVN4cmo6H0"
  | json
  | event =~ "reply\\..*"
```

Or by event only (broad time window around 2026-03-02 09:09 UTC):

```logql
{app="tuco-app"} 
  | json
  | event =~ "reply\\.(branch|without_original\\.no_lead_mapped|without_original\\.lead_mapped|with_original\\.lead_resolved|with_original\\.no_lead_mapped|complete)"
  | messageId = "69a553b96fbd952617b0e40c" or workspaceId = "org_3ABCDbdGyF6w99hikyVN4cmo6H0"
```

If you see `reply.without_original.no_lead_mapped` (or `reply.with_original.no_lead_mapped`) for that request, the reply was stored without `leadId` because no lead in that workspace matched the recipient (e.g. email/phone). With the fix (resolve lead by recipient in both “with original” and “without original” paths), you should see `reply.without_original.lead_mapped` or `reply.with_original.lead_resolved` when a lead exists for that email/phone.

## MongoDB check for the same incident

To confirm the stored reply document for that message:

```javascript
db.messages.findOne(
  { _id: ObjectId("69a553b96fbd952617b0e40c") },
  { leadId: 1, recipientEmail: 1, recipientPhone: 1, direction: 1, createdAt: 1 }
)
```

If `leadId` is missing, that explains why recent activity and campaign reply count didn’t include it (both rely on `leadId` on the reply message).
