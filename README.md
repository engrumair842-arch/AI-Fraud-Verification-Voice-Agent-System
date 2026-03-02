# 🔍 Fraud Verification Voice Agent — n8n + ElevenLabs + HubSpot

An automated outbound fraud verification system that calls customers via AI voice agent (ElevenLabs), verifies their order details, and updates their fraud status in HubSpot — all orchestrated through n8n.

---

## 📋 Overview

When a new deal is created in HubSpot, this system automatically:
1. Retrieves the associated contact's phone number
2. Adds them to a call queue
3. Places an outbound AI voice call (via ElevenLabs + Twilio)
4. Collects billing/shipping address details from the customer
5. Records the verification outcome back to HubSpot (`Passed`, `Failed`, or `Pending`)
6. Cleans up the queue and handles repeat-call logic

---

## 🗂️ Workflow Architecture

```
[HubSpot Deal Created]
        │
        ▼
0. deal → get contact → add to queue
        │
        ▼
[Customer Numbers DataTable Queue]
        │
        ▼
1. Trigger Voice Agent (Cron: every 5 min, 9am–10pm)
        │
        ▼
[ElevenLabs Outbound Call via Twilio]
        │
        ▼
2. Voice Agent Tool (Webhook — post-conversation result)
        │
   ┌────┴────┐
   │         │
Approved   Fraud
   │         │
   └────┬────┘
        ▼
[HubSpot Contact Updated] → [Queue Row Deleted]

        ▼ (if no answer / max 4 calls reached)
3. Post Call Handler
        │
        ▼
[Increment call count] → [If 4 calls: mark Confirmed Fraud + delete row]
```

---

## 📁 Workflows

### `0. deal → get contact → add to queue`
**Trigger:** HubSpot webhook on deal creation (`POST /get-lead-data`)

| Step | Description |
|------|-------------|
| Webhook | Receives deal payload from HubSpot |
| Wait | 30-second delay to allow HubSpot data to propagate |
| Get a Deal | Fetches deal details by `objectId` |
| Edit Fields | Extracts associated contact ID (`associatedVids[0]`) |
| If2 | Checks if a contact ID exists |
| Get a Contact | Fetches contact properties (phone, email, firstname) |
| Edit Fields1 | Maps contact properties to clean variables |
| If (phone exists?) | Routes based on whether phone number is present |
| Insert Row | Adds contact to `Customer Numbers` DataTable queue with `number_of_calls: 0` |
| Pending Verification | Updates HubSpot contact `fraud_check` → `Pending` (if no phone) |

---

### `1. Trigger Voice Agent`
**Trigger:** Schedule — every 5 minutes between 9:00 AM and 10:00 PM

| Step | Description |
|------|-------------|
| Get Phone Numbers | Fetches all rows from `Customer Numbers` DataTable |
| Filter | Passes rows where `number_of_calls = 0` **OR** last updated > 3 hours ago |
| Limit | Throttles to 1 call per execution (prevents call flooding) |
| Initiate Outbound Call | `POST` to ElevenLabs API — triggers Twilio outbound call with `customer_name` and `customer_phone` as dynamic variables |

> **Note:** This workflow is currently set to **inactive** — activate when ready to go live.

---

### `2. Voice Agent Tool`
**Trigger:** ElevenLabs webhook on conversation end (`POST /cutomer-response-details`)

| Step | Description |
|------|-------------|
| Webhook | Receives call outcome from ElevenLabs agent |
| Get Row | Looks up contact in DataTable by phone number |
| If (status check) | Checks if status is `"Approved No Fraud"` |
| Approved No Fraud | Updates HubSpot `fraud_check` → `Passed` |
| Confirmed Fraud | Updates HubSpot `fraud_check` → `Failed` |
| Delete Row | Removes contact from call queue |

---

### `3. Post Call`
**Trigger:** ElevenLabs post-call webhook (`POST /54d7d0d0-...`)

Handles cases where calls are not completed (no answer, voicemail, etc.)

| Step | Description |
|------|-------------|
| Webhook | Receives post-call data including `user_id` (phone number) |
| Get Row | Fetches contact record from DataTable |
| Update Row | Increments `number_of_calls` by 1 |
| If (`number_of_calls = 4`) | Checks if max retry limit reached |
| Confirmed Fraud | Updates HubSpot `fraud_check` → `Failed` after 4 unanswered calls |
| Delete Row | Removes contact from queue |

---

## 🤖 ElevenLabs Voice Agent — Alexis

**Agent ID:** `agent_4001kj6zsqhjf4h9m08zmnchbwyr`

Alexis is a calm, professional fraud-verification specialist. The verification logic is entirely conversation-based — no CRM data is retrieved or cross-referenced during the call.

### Verification Logic

| Scenario | Outcome |
|----------|---------|
| Billing address = Shipping address | ✅ `Approved No Fraud` |
| Addresses differ + customer gives **any** reason | ✅ `Approved No Fraud` |
| Addresses differ + customer **refuses** to give a reason / hangs up | ❌ `Confirmed Fraud` |

### Tool Used by Agent
`store_call_result` — called at the end of every conversation with:
- `status`: `"Approved No Fraud"` or `"Confirmed Fraud"`

---

## 🛢️ Data Store

**DataTable:** `Customer Numbers` (`rQ7teCERv10tzIPT`)

| Column | Type | Description |
|--------|------|-------------|
| `phone` | string | Customer phone number (primary key for lookups) |
| `firstname` | string | Customer first name (passed to voice agent) |
| `email` | string | Customer email (used for HubSpot updates) |
| `number_of_calls` | number | Call attempt counter (max 4 before auto-flagged) |

---

## 🔗 Integrations

| Service | Usage |
|---------|-------|
| **HubSpot** | Source of deal/contact data; destination for `fraud_check` status updates |
| **ElevenLabs** | Hosts conversational AI agent (Alexis); initiates outbound calls |
| **Twilio** | Telephony provider for outbound calls (via ElevenLabs integration) |
| **n8n DataTable** | Temporary call queue storage |

---

## ⚙️ Setup

### Prerequisites
- n8n instance (cloud or self-hosted)
- HubSpot account with App Token (`CcxlCYgN2FSucElZ`)
- ElevenLabs account with a configured Conversational AI agent
- Twilio number connected to ElevenLabs (`phnum_4101kjd3ypr3fv3vs735kaxmxaj8`)

### HubSpot Configuration
1. Create a custom contact property: `fraud_check` (string)
   - Values used: `Pending`, `Passed`, `Failed`
2. Set up a workflow in HubSpot to trigger the n8n webhook on deal creation

### n8n Setup
1. Import all 4 workflow JSON files
2. Configure the HubSpot App Token credential
3. Create the `Customer Numbers` DataTable with the schema above
4. Update webhook URLs in HubSpot and ElevenLabs to match your n8n instance
5. Activate workflows `0`, `2`, and `3` — activate `1` when ready to start calling

### ElevenLabs Setup
1. Create a Conversational AI agent using the system prompt (Alexis)
2. Add the `store_call_result` tool pointing to your workflow `2` webhook URL
3. Connect your Twilio number to the agent
4. Note the `agent_id` and `agent_phone_number_id` and update workflow `1`

---

## 🔄 Call Flow Summary

```
HubSpot Deal Created
  → n8n Webhook (wait 30s)
  → Fetch Deal + Contact
  → Add to Queue (number_of_calls = 0)
  → Set HubSpot fraud_check = "Pending"

Every 5 min (9am–10pm):
  → Check queue for new or stale entries (>3h since last attempt)
  → Place 1 outbound call via ElevenLabs/Twilio

After call completes:
  → ElevenLabs sends result to Workflow 2
  → HubSpot updated (Passed / Failed)
  → Row deleted from queue

If no answer:
  → ElevenLabs triggers Workflow 3
  → Call count incremented
  → After 4 missed calls → auto-mark Failed + remove from queue
```

---

## 📝 Notes

- The `Filter` node in Workflow 1 uses an **OR** condition — a contact is eligible for a call if it's brand new (`number_of_calls = 0`) **or** if 3+ hours have passed since the last attempt. This ensures retries without over-calling.
- Workflow 1 includes a `Limit` node to process only **1 contact per cron tick**, preventing simultaneous outbound calls.
- The agent (Alexis) never accesses or references CRM data during the call — verification is based solely on what the customer states.
