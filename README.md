# GTM Assignment 2 — Workflow Documentation

**Pipeline:** Anakin AI Cold Outreach + Reply Tracking  
**Tool:** n8n (self-hosted)  
**Nodes:** 13  
**Last updated:** March 2026

---

## Overview

This workflow runs two independent automation flows inside a single n8n export.

**Flow A — Outreach Pipeline** starts on a manual trigger, pulls contacts from HubSpot, generates a personalised cold email per contact using an LLM, sends the email via Resend, and appends the full record to Google Sheets.

**Flow B — Reply Tracking Pipeline** listens on a POST webhook. When Resend detects a reply and fires the event, this flow extracts the reply metadata, looks up the original contact row in Sheets, and updates that row with the reply timestamp and status.

---

## Flow A — Outreach Pipeline

### Node 1 · Trigger
**Type:** Manual Trigger  
**Position:** Start of Flow A

Execution starts here. Click **Execute Workflow** in n8n to kick off a full outreach run. Can be swapped for a Schedule Trigger to run automatically (e.g., nightly).

---

### Node 2 · Hubspot API
**Type:** HubSpot — Get All Contacts  
**Credential:** HubSpot App Token

Fetches every contact from HubSpot with `returnAll: true`. The following properties are requested per contact:

- `hs_linkedin_url`
- `firstname`
- `lastname`
- `jobtitle`
- `company`
- `about`
- `linkedin_headline`

Each contact is output as a separate item and flows individually through the rest of the pipeline.

---

### Node 3 · Clean and clear
**Type:** Code (JavaScript)  
**Input:** Raw HubSpot contact items

Normalises the HubSpot response structure into a flat object. The HubSpot API nests values under `properties.<field>.value` and emails under `identity-profiles[0].identities`. This node flattens all of that into:

```
{
  firstname, lastname, email,
  linkedin_url, jobtitle, company,
  about, bio
}
```

Missing fields default to an empty string. `bio` maps to `linkedin_headline`.

**Outputs two branches simultaneously:**
- Branch 1 → **Personalized email** (for LLM generation)
- Branch 2 → **Merge** input 0 (carries the contact data forward to be merged with the generated email later)

---

### Node 4 · Personalized email
**Type:** HTTP Request (POST)  
**URL:** `https://openrouter.ai/api/v1/chat/completions`  
**Model:** `openai/gpt-4o-mini`  
**Credential:** OpenRouter API key

Sends a structured prompt to OpenRouter for each contact. The system prompt positions the sender as a GTM automation specialist. The user prompt passes in:

- `firstname`, `lastname`
- `jobtitle`, `company`
- `bio` (LinkedIn headline)
- `about`

**Prompt instructions to the model:**
- Reference something specific from the recipient's role or expertise
- Connect it to how Anakin helps teams build AI apps and automate workflows
- Stay under 90 words
- End with a friendly call-to-action
- Include a `Subject:` line at the top of the output

The raw LLM response (`choices[0].message.content`) is passed directly to the next node.

---

### Node 5 · Clean v2
**Type:** Code (JavaScript)  
**Input:** Raw OpenRouter response

Parses the LLM output into two clean fields:

1. Extracts the `Subject:` line using a regex match
2. Strips the subject line and any boilerplate sign-offs (e.g. `Best, [Your Name]`) from the body
3. Falls back to `"AI automation idea"` as subject if no match found

Output fields: `firstname`, `email`, `company`, `subject`, `email_body`

Feeds into → **Merge** input 1

---

### Node 6 · Merge
**Type:** Merge — Combine by Position  
**Inputs:** Clean and clear (input 0) + Clean v2 (input 1)

Combines the full contact record from Node 3 with the parsed email content from Node 5. Both streams must have the same item count (one item per contact) for `combineByPosition` to work correctly.

The merged record now contains everything needed for sending and logging:
`firstname, lastname, email, linkedin_url, jobtitle, company, about, bio, subject, email_body`

**Outputs two branches simultaneously:**
- Branch 1 → **Merge1** input 0 (for CRM logging)
- Branch 2 → **Resend API** (for sending)

---

### Node 7 · Resend API
**Type:** HTTP Request (POST)  
**URL:** `https://api.resend.com/emails`  
**Credential:** HTTP Header Auth (Resend API key as Bearer token)  
**Batch size:** 1 (one email per API call)

Constructs and sends the email. Key parameters:

- **From:** `Aryan <hello@aryancore.in>`
- **To:** `{{ $json.email }}`
- **Subject:** `{{ $json.subject }}`
- **HTML body:** Greeting + email body with newlines converted to `<br>` tags + hardcoded signature block (Aryan, GTM Automation Engineer, aryan@aryancore.in)

The node strips the duplicate greeting and boilerplate sign-off that the LLM may re-include in the body.

Returns a Resend email ID (`id`) on success, which is captured in the next merge.

---

### Node 8 · Merge1
**Type:** Merge — Combine by Position  
**Inputs:** Merge (input 0) + Resend API response (input 1)

Combines the full contact + email record with the Resend API response. This makes the Resend-assigned `id` available for logging alongside the contact data.

Feeds into → **Inserting of data**

---

### Node 9 · Inserting of data
**Type:** Google Sheets — Append Row  
**Sheet:** `ANAKIN OUTREACH` (gid=0)  
**Spreadsheet:** GTM Assignment 2

Appends one row per contact to the `ANAKIN OUTREACH` tab with all of the following fields:

`id` (Resend email ID), `firstname`, `lastname`, `email`, `linkedin_url`, `jobtitle`, `company`, `about`, `bio`, `subject`, `email_body`

The `id` field (Resend email ID) is the match key and is used later by Flow B when a reply comes in.

---

## Flow B — Reply Tracking Pipeline

### Node 10 · Resend Reply Webhook
**Type:** Webhook — POST  
**Path:** `/resend-replies`  
**Webhook ID:** `8d0e1070-9db7-4ecc-bd14-53f31e2025a9`

Receives POST events from Resend when a recipient replies to a sent email. Must be registered in the Resend dashboard as the reply webhook URL:

```
https://<your-n8n-host>/webhook/resend-replies
```

The raw event payload is passed to the next node.

---

### Node 11 · Clean and Extraction node
**Type:** Code (JavaScript)  
**Input:** Raw Resend webhook payload

Extracts the relevant fields from the nested Resend event structure (`body.data`):

```
{
  email:          data.from
  subject:        data.subject
  resend_email_id: data.email_id
  message_id:     data.message_id
  received_at:    body.created_at
  event_type:     body.type
}
```

**Outputs two branches simultaneously:**
- Branch 1 → **Get action** (to look up the original row)
- Branch 2 → **Merge2** input 0 (carries reply data forward)

---

### Node 12 · Get action
**Type:** Google Sheets — Get Row(s) by Filter  
**Sheet:** `ANAKIN OUTREACH`  
**Filter:** `email = {{ $json.email }}`

Looks up the original outreach row using the sender's email address from the reply. Returns the full row including `row_number` — the key used to update the correct row.

Feeds into → **Merge2** input 1

---

### Node 13 · Merge2
**Type:** Merge — Combine by Position  
**Inputs:** Clean and Extraction node (input 0) + Get action (input 1)

Combines the reply metadata with the looked-up row data (including `row_number`).

Feeds into → **Final Update in sheet**

---

### Node 14 · Final Update in sheet
**Type:** Google Sheets — Update Row  
**Sheet:** `ANAKIN OUTREACH`  
**Match key:** `row_number`

Updates the existing row in `ANAKIN OUTREACH` with:

- `id` ← `resend_email_id` (from reply event)
- `reply_time` ← `received_at`
- `status` ← `$json.status`
- `reply_subject` ← (currently empty — add `$json.subject` here if needed)

The update is matched by `row_number` to ensure the correct row is modified.

---

## Data Flow Diagram

```
FLOW A — OUTREACH
─────────────────────────────────────────────────────────────────

Trigger
  └─► Hubspot API (get all contacts)
        └─► Clean and clear (flatten HubSpot structure)
              ├─► Personalized email (OpenRouter / GPT-4o-mini)
              │     └─► Clean v2 (parse subject + body)
              │           └─► Merge ◄────────────────────────────┐
              └───────────────────────────────────────────────────┘
                              │
                    ┌─────────┴──────────┐
                    ▼                    ▼
               Resend API           Merge1 ◄── Resend API response
               (send email)             │
                                        ▼
                                 Inserting of data
                                 (append to Sheets)


FLOW B — REPLY TRACKING
─────────────────────────────────────────────────────────────────

Resend Reply Webhook (POST /resend-replies)
  └─► Clean and Extraction node (parse reply payload)
        ├─► Get action (lookup row in Sheets by email)
        │     └─► Merge2 ◄──────────────────────────────────────┐
        └──────────────────────────────────────────────────────┘
                          │
                          ▼
                  Final Update in sheet
                  (update status + reply_time by row_number)
```

---

## Error Handling Notes

- If HubSpot returns a contact with a missing email, the Resend API call will fail for that item. Add an **IF** node before Resend API to filter out empty emails.
- The `Personalized email` node has no retry configured. If OpenRouter times out, the contact is silently dropped. Add `onError: continueRegularOutput` or a retry policy on that node for production.
- The `Clean v2` subject fallback is `"AI automation idea"` — ensure this is acceptable as a default before scaling.
- `combineByPosition` in all Merge nodes requires both input branches to produce the same item count. If one branch errors partway through, the merge will misalign. Monitor execution logs carefully.

---

## Quick Reference — Nodes by Function

| Node | Type | Function |
|---|---|---|
| Trigger | Manual Trigger | Starts Flow A |
| Hubspot API | HubSpot | Fetches all contacts |
| Clean and clear | Code (JS) | Flattens HubSpot response |
| Personalized email | HTTP Request | LLM email generation |
| Clean v2 | Code (JS) | Parses LLM output |
| Merge | Merge | Joins contact + email content |
| Resend API | HTTP Request | Sends email |
| Merge1 | Merge | Joins merged record + Resend response |
| Inserting of data | Google Sheets | Appends row to CRM sheet |
| Resend Reply Webhook | Webhook | Receives reply events (Flow B) |
| Clean and Extraction node | Code (JS) | Parses reply payload |
| Get action | Google Sheets | Looks up original row by email |
| Merge2 | Merge | Joins reply data + row lookup |
| Final Update in sheet | Google Sheets | Updates reply status + timestamp |

---

*Internal use · GTM Automation · March 2026*
