# AI Lead Qualification Agent for B2B Marketplaces

> A conversational n8n automation that turns vague buyer inquiries into complete, structured leads. It asks follow-up questions until nothing is missing, then saves the lead to a Google Sheets CRM.

**Stack:** n8n | OpenAI (gpt-4o-mini) | Google Sheets | Webhooks

---

## 1. Problem Statement

On B2B marketplaces such as IndiaMART, sales teams receive a constant stream of raw buyer inquiries. Typical messages look like:

> "Need stainless steel pipes, 500 kg."

These inquiries share a few problems:

- **Incomplete information.** Buyers often skip critical specs such as size, grade, power rating or delivery location. Without them, the seller can't quote.
- **Slow manual follow-up.** A salesperson has to read each inquiry, spot what's missing and write back. Replies are delayed, especially outside working hours, and interested buyers drift to competitors.
- **Inconsistent data.** Details are scattered across chats, emails and notes. There's no single place to see which leads are ready to quote and which still need information.
- **Wasted sales time.** Teams spend effort on back-and-forth clarification instead of closing deals.

**Goal:** automatically collect every critical detail from each buyer, in conversation, before a human gets involved, and record clean, structured leads in one place.

---

## 2. Solution Overview

The automation receives each inquiry through a webhook and runs it through an AI agent. The agent:

1. Extracts structured fields: product, quantity, delivery location.
2. Decides whether the lead is **Qualified** or **Needs Clarification**.
3. Writes a polite, professional reply to the buyer. If details are missing, the reply asks one or two sharp questions about them.

The reply goes straight back to the caller. The conversation repeats until all required information is collected. Only then is the lead saved to the CRM.

---

## 3. Workflow Architecture

```
            +-----------------------+
            |  Webhook (POST)       |  <- buyer message + session_id
            +-----------+-----------+
                        |
            +-----------v-----------+
            |  AI Agent             |  <- OpenAI chat model
            |  + Conversation Memory|     + per-session history
            +-----------+-----------+
                        |  (JSON text)
            +-----------v-----------+
            |  Code: parse output   |  <- clean, validate, safe fallback
            +-----------+-----------+
                        |
            +-----------v-----------+
            |  Is Lead Complete?    |
            +-----+-----------+-----+
              No  |           | Yes
   +--------------v--+     +--v------------------+
   | Respond to      |     | Append row to       |
   | webhook:        |     | Google Sheets       |
   | ask for missing |     +--+------------------+
   | info            |        |
   +-----------------+     +--v------------------+
                           | Respond to webhook: |
                           | confirmation        |
                           +---------------------+
```




### Components

| Node | Role |
|---|---|
| **Webhook** | Entry point. Accepts a POST with the buyer's message and a `session_id`, `email` or `phone`. It responds only after the workflow finishes. |
| **AI Agent** | Extracts product, quantity and location from the conversation, sets the lead status, and drafts the reply. |
| **Conversation Memory** | Stores earlier messages per session, so the agent merges new answers with old ones and never repeats a question. |
| **Code (JavaScript)** | Parses the agent's JSON output. If the output is malformed, it falls back to a safe "Needs Clarification" result instead of failing. |
| **Is Lead Complete?** | Branches on `lead_status`: `Qualified` or `Needs Clarification`. |
| **Respond to Webhook (missing info)** | Returns `status: needs_more_info` with the follow-up question. The caller sends the answer with the same `session_id`. |
| **Google Sheets (append row)** | Saves complete leads with a timestamp. |
| **Respond to Webhook (complete)** | Returns `status: complete` with the confirmation message. |

### Design decisions

- **Multi-turn loop through the caller.** The workflow is stateless apart from the memory node. Each request is one turn, so it works with any front end: a web form, a chat widget or a WhatsApp bot.
- **Structured output.** The agent is instructed to return strict JSON, and the Code node guards against bad output.
- **Only complete leads reach the CRM.** This keeps the sheet clean and quote-ready.

---

## 4. Business Impact

These are the expected benefits of the design. Measure them in your own pipeline before quoting numbers.

| Area | Impact |
|---|---|
| **Response time** | Buyers get a clarifying reply right away, at any hour, instead of waiting for a salesperson. |
| **Lead quality** | Every lead in the CRM already has the key details, so it's ready to quote. |
| **Sales productivity** | Reps stop doing routine clarification and spend their time on qualified, high-intent buyers. |
| **Data consistency** | All leads are stored in one structured sheet with the same columns. |
| **Scalability** | Inquiry volume can grow without hiring more people for first-line follow-up. |
| **Buyer experience** | Replies are polite, specific and professional, which builds trust before a human takes over. |

**Suggested metrics to track:** time to first response, percentage of leads qualified without human help, average number of follow-up turns, and quote-to-close rate before and after.

---

## 5. API Usage

**Request**

```bash
curl -X POST https://<your-n8n-domain>/webhook/<your-webhook-path> \
  -H "Content-Type: application/json" \
  -d '{
    "session_id": "buyer-123",
    "message": "I need 500 kg of stainless steel pipes"
  }'
```

**Response while information is missing**

```json
{
  "status": "needs_more_info",
  "reply": "Thank you for your inquiry! Could you please specify the delivery location and the required size and grade?",
  "missing_info": "delivery location, specifications (size, grade)"
}
```

**Response when the lead is complete**

```json
{
  "status": "complete",
  "reply": "Thank you, we have received your requirement.",
  "data": { "lead_status": "Qualified" }
}
```

---

## 6. Google Sheet Schema

`Timestamp` | `Raw Buyer Inquiry` | `Product Name` | `Quantity` | `Delivery Location` | `Lead Status` | `Missing Info` | `Agent Response`

---

## 7. Setup

1. Import the workflow into n8n.
2. Add your **OpenAI** credentials to the Chat Model node.
3. Connect **Google Sheets** and select your sheet, using the columns above as headers.
4. Publish the workflow and use the production webhook URL.

---

## 8. Roadmap

- Notify the sales team on Slack or email when a lead becomes qualified.
- Store the full conversation transcript with each lead.
- Add webhook authentication.
- Connect WhatsApp or a website chat widget as the front end.
- Score leads by order size and urgency.

---

## Author

**Jaiverdhan Singh Shekhawat**
