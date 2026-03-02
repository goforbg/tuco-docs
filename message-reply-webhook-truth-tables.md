# message.reply Webhook: Truth Tables

Reference for when `repliedAtUtc` and `parentMessages` are present and how backward compatibility is preserved.

---

## 1. Matching & payload presence

When we fire the `message.reply` webhook, these tables describe **when each new field is present** and **what parent messages are matched**.

### 1.1 Reply path and context

| Reply path | Has `leadId` | Has campaign (original has batchId) | `repliedAtUtc` in payload | `parentMessages` in payload | Parent scope |
|------------|--------------|-------------------------------------|---------------------------|------------------------------|--------------|
| With original | Yes | Yes | ✅ Always | ✅ If we have ≥1 outbound to that lead in **that campaign** | Campaign (`batchId` regex) |
| With original | Yes | No | ✅ Always | ✅ If we have ≥1 outbound to that lead in workspace | Workspace (any outbound to lead) |
| With original | No | — | ✅ Always | ❌ Omitted (we don’t fetch parents without lead) | — |
| Without original | Yes | N/A | ✅ Always | ✅ If we have ≥1 outbound to that lead in workspace | Workspace (any outbound to lead) |
| Without original | No | N/A | ✅ Always | ❌ Omitted | — |

- **repliedAtUtc**: Always sent. Derived from `replyMessage.createdAt` (or from `extras.repliedAtUtc` when provided).
- **parentMessages**: Only added when we have at least one parent message; otherwise the field is omitted (not sent). Old consumers that don’t expect it are unaffected.

### 1.2 Parent-message query (matching logic)

| Context | `getParentMessagesForReplyWebhook(..., campaignId?)` | Messages matched |
|---------|------------------------------------------------------|------------------|
| Reply **with** original, campaign known | `campaignId` = campaign `_id` | Last 2 outbound with `batchId` matching `^campaign_<id>_`, same `leadId`, status sent/delivered |
| Reply **with** original, no campaign | `campaignId` = undefined | Last 2 outbound to that `leadId` in workspace (any batchId) |
| Reply **without** original | `campaignId` = undefined | Last 2 outbound to that `leadId` in workspace (any batchId) |

So: **in campaign** → parents are campaign outbound only; **outside campaign** → parents are last 2 outbound to that lead in the workspace. Matching is by `workspaceId`, `leadId`, and optional `batchId` regex.

---

## 2. Backward compatibility

Existing payload fields are unchanged. New fields are additive.

### 2.1 Consumer behavior

| Consumer | Reads | `repliedAtUtc` | `parentMessages` | Still works? |
|----------|--------|----------------|------------------|--------------|
| Old (pre-change) | Only `event`, `messageId`, `message`, `originalMessageId`, `originalMessage`, `leadId`, `lead`, `campaignId`, `campaignName` | Ignores | Ignores | ✅ Yes |
| New | Same + `repliedAtUtc` | Always present | Optional; treat as `T[] \| undefined` | ✅ Yes |

- Old code never expected `repliedAtUtc` or `parentMessages`; extra keys in JSON are ignored, so behavior is unchanged.
- New code: `repliedAtUtc` is always present; `parentMessages` may be absent, so check for presence/array length before use.

### 2.2 Call-site compatibility

| Call site | Passes `extras`? | `repliedAtUtc` source | `parentMessages` source |
|-----------|------------------|------------------------|--------------------------|
| `processReplyWithOriginal` | Yes | `replyMessage.createdAt` → ISO | `getParentMessagesForReplyWebhook(..., campaign?._id)`; omit if length 0 |
| `processReplyWithoutOriginal` | Yes | `replyMessage.createdAt` → ISO | `getParentMessagesForReplyWebhook(..., undefined)`; omit if length 0 |
| Legacy (if 4-arg only) | No | Fallback in `fireMessageReplyWebhook`: `replyMessage.createdAt` → ISO | Not added (conditional spread) |

So even when `extras` is not passed, `repliedAtUtc` is still set from the reply message; `parentMessages` is only set when explicitly provided and non-empty. Backward compatible at both consumer and call-site level.

---

## Summary

- **Matching**: Parents are last 2 outbound to the lead; in campaign context they are scoped to that campaign’s `batchId`; outside campaign they are workspace-wide. `repliedAtUtc` always reflects the reply’s received time.
- **Backward compatible**: Existing fields and behavior unchanged; new fields are optional for consumers; 4-arg calls still work and still get `repliedAtUtc` from the reply message.
