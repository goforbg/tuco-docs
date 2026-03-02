# Problems we solved (shortform)

One-line problem → one-line fix. For full flow see reply-lead-fix-proof-and-flowchart.md, FLOW-DUPLICATE-LEADS-CAMPAIGN-REENTRY.md, debug-reply-loki-queries.md.

---

1. **Reply had no leadId** → recent activity + campaign reply count wrong, webhook had no lead.  
   **Fix:** Set reply lead from original.leadId or `findLeadByRecipient(workspaceId, phone, email)` in both paths (with/without original). Same logic as Unibox.

2. **Webhook message.reply** missing parentMessages, wrong shape, not like GET /api/replies.  
   **Fix:** `message` = reply text only. `data.reply` = full object (same as API). `parentMessages` always sent (array). Top-level: firstName, lastName, phone, listId, campaignId, fromLineId, recipientName, leadId.

3. **Customer webhooks** fired on delivered; we only want sent.  
   **Fix:** Customer webhooks get `message.sent` when message is sent. No webhook on delivery.

4. **Quicksend outbound** had no leadId when client sent only email/phone (no leadId).  
   **Fix:** In POST /api/leads/send, if no leadId: `findLeadByRecipient` or create Quick Sends lead; set leadId on message.

5. **Campaign follow-ups** not cancelled when lead replied and original had no leadId.  
   **Fix:** Cancel uses resolved `replyLeadId` when original.leadId missing (messageForCancel with resolved id).

6. **Duplicate leads** returned 409 → n8n/consumers stopped.  
   **Fix:** POST /api/leads all duplicates → 200 + existingLeadIds. POST campaigns/[id]/leads duplicate workspace lead → use existing, 200. Already in campaign (re-entry) → 200, no new sequence.

7. **POST /api/messages** Quick Sends duplicate → 409.  
   **Fix:** If recipient in Quick Sends list → use existing lead, send, return 200 + existingLeadIds, duplicateMessage.

8. **Lead identifiers** for dedup and reply lookup (email/phone + alts).  
   **Fix:** leadIdentifiers.ts: getLeadContactIdentifiers, leadIdentifierQuery, conversationDeleteFilter. Same norm as Unibox/reply.

---

**Loki (main):** reply.with_original.lead_resolved, reply.without_original.lead_mapped, reply.complete, lead.send.lead_resolved_by_recipient, campaign.lead.duplicate_use_existing, campaign.process_lead.already_in_campaign, message.api.duplicate_lead_used.

**Repos:** app (device-callbacks, webhookDelivery, quickSendsList, leads/send, leads, campaigns/[id]/leads, messages, campaignProcessor, leadIdentifiers). docs (this + flow + Loki docs).
