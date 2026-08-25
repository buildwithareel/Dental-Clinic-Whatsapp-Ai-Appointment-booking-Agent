# 🚀 Installation Guide

This guide walks you through setting up **Dental Clinic – AI Appointment Booking Agent** from a blank n8n instance to a fully running WhatsApp booking assistant.

> 📖 New to the project? Start with the [main README](../README.md) for an architecture overview before following the steps below.

---

## 1. Prerequisites Checklist

Before you begin, make sure you have the following ready. Each of these is required — the workflow will not run correctly with anything missing.

| # | Requirement | Where to get it |
|---|---|---|
| 1 | An **n8n instance** (Cloud or self-hosted) with LangChain community nodes enabled | [n8n.io](https://n8n.io) |
| 2 | A **WhatsApp Business number** connected through a BSP such as **Cloud Station** | Your BSP's dashboard |
| 3 | An **OpenAI API key** | [platform.openai.com](https://platform.openai.com) |
| 4 | A **Google Gemini API key** | [Google AI Studio](https://aistudio.google.com) |
| 5 | A **Pinecone account** with a vector index created and populated with clinic FAQ content | [pinecone.io](https://www.pinecone.io) |
| 6 | A **Google Calendar** with a shared clinic booking calendar | Google Cloud Console |
| 7 | An **Airtable base** with `Leads`, `Appointment`, and `Errors` tables | [airtable.com](https://airtable.com) |
| 8 | **Gmail (or other email) credentials** for staff error alerts | Google Cloud Console |
| 9 | Your **BSP's send-message API URL and access token** | Cloud Station (or equivalent) dashboard |

📎 For full configuration details on each of the above, see [configuration.md](./configuration.md).

---

## 2. Get the Workflow File

Clone this repository, or simply download the workflow JSON directly:

```bash
git clone https://github.com/<your-username>/dental-clinic-ai-booking-agent.git
cd dental-clinic-ai-booking-agent
```

The importable workflow file is:

```
Dental_Clinic___Ai_Appointment_booking_Agent.json
```

---

## 3. Import the Workflow into n8n

1. Open your n8n instance.
2. Go to **Workflows → Import from File** (or **⋯ menu → Import from File** on n8n Cloud).
3. Select `Dental_Clinic___Ai_Appointment_booking_Agent.json` from the cloned repo.
4. The full canvas will load — **Incoming Message → Is Text Message? → Normalize Input → Dental Booking Agent → Agent Succeeded? → Send Reply / Fallback path** — along with the connected model, memory, and tool nodes.

> ⚠️ At this point the workflow is **inactive** and none of the credentials are connected yet — that's expected. Continue to the next step.

---

## 4. Connect Credentials to Each Node

Open each of the following nodes and attach (or create) the matching credential:

| Node | Credential to attach |
|---|---|
| OpenAI Chat Model | OpenAI API Key |
| Embeddings Google Gemini | Google Gemini API Key |
| Tool: Search Knowledge Base | Pinecone API Key |
| Tool: Check Availability | Google Calendar OAuth2 |
| Tool: Book Appointment | Google Calendar OAuth2 |
| Tool: Reschedule Appointment | Google Calendar OAuth2 |
| Tool: Cancel Appointment | Google Calendar OAuth2 |
| Tool: Save Lead to CRM | Airtable OAuth2 |
| Tool: Log Appointment in CRM | Airtable OAuth2 |
| Log Error to CRM | Airtable OAuth2 |
| Email Clinic Staff | Gmail OAuth2 |
| Send Reply (Cloud Station HTTP Request) | HTTP Header Auth (Bearer token) |
| Send Fallback Reply (Cloud Station HTTP Request) | HTTP Header Auth (Bearer token) |

Detailed field-by-field setup for each credential is covered in [configuration.md](./configuration.md).

---

## 5. Set n8n Environment Variables

This workflow keeps the BSP's send-message endpoint and token out of the workflow body by referencing n8n environment variables:

1. In n8n, go to **Settings → Variables**.
2. Add:
   - `CLOUDSTATION_API_URL` — your BSP's send-message API base URL.
   - `CLOUDSTATION_API_TOKEN` — your BSP's Bearer token for authenticating sends.

---

## 6. Update Clinic-Specific Values

Before activating, edit these node fields to match your own clinic:

- **Tool: Check Availability / Book Appointment / Reschedule Appointment / Cancel Appointment** → set your clinic's **Google Calendar ID**.
- **Tool: Save Lead to CRM / Log Appointment in CRM / Log Error to CRM** → set your **Airtable Base ID** and correct **Table IDs**.
- **Tool: Search Knowledge Base** → set your Pinecone **Index Name** (e.g. `bright-dental-rag`).
- **Dental Booking Agent** → update the system prompt with your clinic's real name, tone, services, and doctor(s).
- **Email Clinic Staff** → replace the placeholder recipient with your clinic's real alert email address.

---

## 7. Populate Your Pinecone Knowledge Base

The agent can only answer FAQs grounded in what's stored in Pinecone. Before going live:

1. Prepare your knowledge documents (clinic hours, location, services, pricing, insurance policy, doctor specialties) as plain text or structured chunks.
2. Generate embeddings using the **same embedding model** referenced in the workflow (Google Gemini Embeddings).
3. Upsert those embeddings into your Pinecone index under the index name configured in step 6.

---

## 8. Verify the Incoming Payload Shape

This workflow assumes a **Meta Cloud API–style** payload shape (`entry[0].changes[0].value.messages[0]...`) since most BSPs mirror it closely — but this is the one part of the workflow tied specifically to your provider.

1. Send yourself one test WhatsApp message to your connected number.
2. Open the **Normalize Input** node's input panel in n8n and inspect the actual payload Cloud Station sent.
3. Compare it against the expressions used for `sessionId`, `customerName`, and `message`.
4. If your BSP's field names differ, update those three expressions **and** the condition inside **Is Text Message?** to match.

---

## 9. Register the Webhook with Your BSP

1. Copy the **Production Webhook URL** shown on the **Incoming Message** node.
2. In your BSP's dashboard (e.g. Cloud Station), paste this URL as the inbound webhook/callback URL for your WhatsApp number.
3. If your BSP requires a verification handshake (some send a `GET` request first to confirm the URL), add a second Webhook node set to `GET` with **Respond Immediately** enabled to echo back their challenge parameter — check your BSP's docs for the exact requirement.

---

## 10. Set the Workflow's Error Workflow

1. Open this workflow's **Settings → Error Workflow**.
2. Select (or create) a global error-handling workflow so any failure that slips past the built-in fallback logic is still caught and alerted on via email.

---

## 11. Activate the Workflow

1. In n8n, toggle the workflow to **Active** (top-right switch).
2. Send a real test WhatsApp message asking to book an appointment.
3. Confirm:
   - The message reaches the **Incoming Message** node (check the **Executions** tab).
   - The **Dental Booking Agent** asks a clarifying question or checks availability correctly.
   - A confirmed booking creates a real event in **Google Calendar**.
   - The lead/appointment appears in the correct **Airtable** table.
   - The reply is delivered back into the WhatsApp chat via **Send Reply**.

---

## 12. Verify End-to-End

Run through this quick smoke test after activation:

- [ ] A real WhatsApp message triggers an execution in n8n.
- [ ] Non-text/irrelevant events are correctly ignored (no wasted AI calls).
- [ ] The agent only offers time slots that `Check Availability` actually returned.
- [ ] A confirmed booking creates a real Google Calendar event.
- [ ] Leads and appointments appear correctly in Airtable.
- [ ] Knowledge-base questions (hours, pricing, etc.) are answered accurately from Pinecone, not guessed.
- [ ] Simulate a tool failure (e.g. revoke a credential temporarily) and confirm the fallback message, error log, and staff email all fire correctly.
- [ ] The webhook always returns `200 OK`, even during simulated failures.

If any step fails, double check the credential mapping in [configuration.md](./configuration.md) and review execution logs in n8n's **Executions** tab.

---

## Next Steps

- 🔧 [Configuration Guide](./configuration.md) — detailed credential and environment setup.
- 🗺️ [Workflow Architecture](./workflow-architecture.md) — full technical breakdown of every node.