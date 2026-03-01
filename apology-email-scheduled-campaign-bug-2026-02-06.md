# Apology Email: Scheduled Campaign Activation Bug

**Date:** February 6, 2026  
**Issue:** Scheduled campaigns failed to auto-activate and manual activation marked leads as complete without sending

---

## Email Template

**Subject:** Apology: Your scheduled campaign did not send - issue resolved

---

Dear [Client Name],

We owe you an apology and an explanation for what happened with your scheduled campaign.

### What Happened

When you scheduled your campaign, you expected it to automatically start sending at the scheduled time. Unfortunately, due to a bug in our system, this did not happen:

1. **The automatic activation failed** - Our system found your scheduled campaign at the right time but did not properly activate it. The code that was supposed to handle this was incomplete.

2. **Manual activation also failed** - When you noticed the issue and clicked "Activate" to start the campaign manually, a second bug caused all leads to be immediately marked as "complete" without sending any messages.

As a result, your leads were never contacted. They remained in "pending" status while the campaign was incorrectly marked as completed.

### Why This Happened

Our scheduled campaign activation feature had a critical gap in its implementation. The worker process that monitors scheduled campaigns was only logging that campaigns were ready, but not actually activating them. When we added the manual "Activate" button as a workaround, we missed adding the necessary step to create the messages before processing.

### What We've Fixed

We have identified and deployed fixes for both issues:

1. Scheduled campaigns now automatically activate when their scheduled time arrives
2. The "Activate" button now properly creates messages before processing
3. We've added safeguards to prevent race conditions during activation
4. Comprehensive logging has been added to catch any similar issues in the future

### Making This Right

To address the impact on your business:

- **We will reprocess your campaign** - All leads will receive messages as originally intended
- **No additional charges** - This reprocessing will be at no cost to you
- **Your choice on timing** - We can start the reprocessing immediately, or wait until your preferred time
- **Message review** - If you'd like to adjust the messaging before resending, we're happy to help

### Next Steps

Please reply to this email or reach out to us directly to let us know:
1. When you'd like us to reprocess the campaign (immediately or at a specific time)
2. Whether you'd like to review or modify the messages before sending
3. Any questions or concerns you have

### Our Commitment

We understand that reliable campaign delivery is critical to your business. This should not have happened, and we take full responsibility. We've also added this issue to our monitoring and testing to ensure it doesn't happen again.

Thank you for your patience and understanding.

Sincerely,

[Your Name]  
[Your Title]  
Tuco

---

## Internal Notes

**Bug Details:**
- `activateScheduledCampaigns()` in worker only logged campaigns, didn't activate them
- Manual activation in PUT /api/campaigns didn't call `initializeCampaignMessages()` for scheduled/draft campaigns
- Result: 0 messages created → campaign marked complete → "Outside sending window" badge never showed

**Fix Deployed:**
- Worker now atomically claims campaigns (scheduled → activating) and calls app API
- App API endpoint `/api/campaigns/[id]/activate-internal` handles message initialization (uses existing SERVICE_TOKEN auth)
- Manual activation now calls `initializeCampaignMessages()` for scheduled/draft campaigns
- New `activating` status prevents race conditions

**Environment Variables (existing, no changes needed):**
- `APP_URL` or `MAIN_APP_URL` - Base URL of the app
- `SERVICE_TOKEN` - Already used for worker-to-app authentication

**Loki Events to Monitor:**
- `campaign.scheduled.activating` - activation started
- `campaign.scheduled.activated` - activation successful
- `campaign.scheduled.activation_failed` - activation failed (will retry)
- `campaign.activate.initialized` - manual activation initialized messages


# Tuco AI — Receive iMessage, Reply & Add to Campaign

Import **`tuco-imessage-receive-reply-campaign.json`** into n8n to:

1. **Receive** incoming iMessages from your Tuco webhook (e.g. `message.reply` event).
2. **Reply** via a separate **Send iMessage** flow (no GHL Contact ID): picks an active line and sends a reply using `POST /api/messages`.
3. **Add lead to campaign**: if a campaign ID is provided, the workflow finds the lead or **creates the lead using the same params as send-iMessage** (firstName, lastName, phone, email), stores them in **Quick Sends** (and creates the Quick Sends list if it doesn’t exist), then adds the lead to the given campaign.

## Setup

1. **Credentials**  
   Create an **HTTP Header Auth** credential named **"Tuco API Key"** with your Tuco API key (e.g. header `Authorization: Bearer <key>` or your app’s header scheme).

2. **Webhook URL**  
   After activating the workflow, use the webhook URL (e.g. `https://<your-n8n>/webhook/tuco-imessage-receive`) in **Tuco app → Integrations / Webhooks** and subscribe to **message.reply** (or the event that sends incoming iMessage data).

3. **Campaign ID (optional)**  
   - To add repliers to a campaign, the webhook payload can include **`campaignId`** or **`campaignIdToAdd`** (the campaign they should be added to).  
   - If you always add to the same campaign, you can set a default in the **Set — Prepare Input** node (`campaignIdToAdd`).

## Webhook payload (from Tuco)

The workflow expects a JSON body compatible with Tuco’s **message.reply** webhook. When **adding to campaign**, if the lead doesn’t exist it uses the **same params as send-iMessage** to create the lead and put them in Quick Sends:

- **Lead / send-iMessage params:** `body.firstName`, `body.lastName`, `body.phone`, `body.email` (or `body.lead.firstName`, `body.lead.lastName`, `body.lead.phone`, `body.lead.email`). Same shape as the “Add Lead & Send” webhook.
- `body.leadId` – existing lead ID (if known); if present, that lead is added to the campaign.
- `body.lead` – full lead object (phone, email, firstName, lastName, etc.).
- `body.message` – message object (`message` = text, `recipientPhone`, `recipientEmail`).
- `body.campaignId` / `body.campaignIdToAdd` – campaign to add the lead to.
- `body.replyMessage` – (optional) custom reply text; otherwise a default “Thanks for your message…” is used.

## Flow summary

- **Reply path:** Webhook → Set → Get Lines → Pick Active Line → **Send iMessage** (POST /api/messages, no GHL/lead ID) → Respond to Webhook.
- **Campaign path (only if `campaignIdToAdd` is set):**
  - If `body.leadId` exists → add that lead to the campaign.
  - Else → GET `/api/leads?search=<phone>&limit=1`.
    - If lead found → add to campaign.
    - If not → **same as send-iMessage:** GET lists → find or create **Quick Sends** list → POST `/api/leads` with **same params** (firstName, lastName, phone, email) and that listId → GET lead by phone → add to campaign.

## Files

- **`tuco-imessage-receive-reply-campaign.json`** – n8n workflow (import in n8n).
- **`tuco-imessage-workflow.json`** – original “Add Lead & Send” / events workflow (separate use case).

