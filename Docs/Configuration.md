# 🔑 Configuration Guide

This document covers every credential, API key, and configurable field needed to run **Dental Clinic – AI Appointment Booking Agent**. Use it alongside [installation.md](./installation.md) when setting up the workflow for the first time, or whenever you need to rotate a key, switch BSPs, or migrate to a new environment.

> ℹ️ n8n stores credentials in its own encrypted **Credentials Manager**, and this workflow keeps BSP secrets out of node parameters entirely via n8n's **Settings → Variables**. Neither of these should ever be hardcoded into node fields or committed to this repository.

---

## 1. Credential Reference Table

| Credential | Type in n8n | Used By Node(s) | Description |
|---|---|---|---|
| OpenAI API Key | `OpenAI API` | OpenAI Chat Model | Authenticates chat completion requests that power the Dental Booking Agent's reasoning. |
| Google Gemini API Key | `Google Gemini (PaLM) API` | Embeddings Google Gemini | Authenticates embedding generation for RAG lookups. |
| Pinecone API Key | `Pinecone API` | Tool: Search Knowledge Base | Grants access to your Pinecone project. |
| Pinecone Index Name | *(node field, not a credential)* | Tool: Search Knowledge Base | Name of the index holding your clinic's FAQ knowledge base (e.g. `bright-dental-rag`). |
| Google Calendar OAuth2 | `Google Calendar OAuth2 API` | Check/Book/Reschedule/Cancel Appointment tools | Grants read/write access to the clinic's shared booking calendar. |
| Airtable OAuth2 | `Airtable OAuth2 API` | Save Lead to CRM, Log Appointment in CRM, Log Error to CRM | Grants access to your Airtable base and its tables. |
| Gmail OAuth2 | `Gmail OAuth2` | Email Clinic Staff | Authenticates sending failure-alert emails. |
| HTTP Header Auth (Bearer token) | `HTTP Header Auth` | Send Reply, Send Fallback Reply | Authenticates outbound requests to your BSP's send-message API. |
| `CLOUDSTATION_API_URL` | n8n environment variable | Send Reply, Send Fallback Reply | Base URL of your BSP's send-message endpoint. |
| `CLOUDSTATION_API_TOKEN` | n8n environment variable | Send Reply, Send Fallback Reply | Bearer token for authenticating outbound WhatsApp sends. |

---

## 2. OpenAI Configuration

**Used by:** `OpenAI Chat Model`

1. Generate an API key at [platform.openai.com/api-keys](https://platform.openai.com/api-keys).
2. In n8n: **Credentials → Add Credential → OpenAI API**.
3. Paste the key and save.
4. Open the **OpenAI Chat Model** node and select the credential.
5. Set your preferred model (e.g. `gpt-4o` / `gpt-4o-mini`) — lower temperature is recommended here since the agent is following strict booking rules rather than free-form chatting.

---

## 3. Google Gemini Configuration

**Used by:** `Embeddings Google Gemini`

1. Generate an API key at [Google AI Studio](https://aistudio.google.com/app/apikey).
2. In n8n: **Credentials → Add Credential → Google Gemini (PaLM) API**.
3. Paste the key and save.
4. Open the **Embeddings Google Gemini** node and select the credential.

> ⚠️ The embedding model used here **must match** whatever model you used to embed your clinic's FAQ documents into Pinecone — mismatched embedding dimensions will cause vector search to fail.

---

## 4. Pinecone Configuration

**Used by:** `Tool: Search Knowledge Base`

1. Create a Pinecone account and project at [pinecone.io](https://www.pinecone.io).
2. Create an index with a **dimension size matching your embedding model's output**.
3. In n8n: **Credentials → Add Credential → Pinecone API**, paste your API key.
4. Open the **Tool: Search Knowledge Base** node:
   - Select the Pinecone credential.
   - Set **Pinecone Index** to match your created index (e.g. `bright-dental-rag`).
   - Confirm it's set to **retrieve-as-tool** mode so the agent can call it dynamically for FAQ questions.
5. Populate the index with clinic knowledge (see [installation.md](./installation.md#7-populate-your-pinecone-knowledge-base)).

---

## 5. Google Calendar Configuration

**Used by:** `Tool: Check Availability`, `Tool: Book Appointment`, `Tool: Reschedule Appointment`, `Tool: Cancel Appointment`

1. In Google Cloud Console, enable the **Google Calendar API** for your project.
2. Create an **OAuth2 Client ID** and complete the consent flow.
3. In n8n: **Credentials → Add Credential → Google Calendar OAuth2 API**.
4. Create (or choose) a **shared calendar** for the clinic — this workflow assumes a single shared calendar for all appointments, so no per-doctor routing is needed by default.
5. Open each of the four calendar tool nodes and:
   - Select the credential.
   - Set the **Calendar ID** to your shared clinic calendar's ID (found in Google Calendar → Settings → Integrate calendar).

> 💡 **Check Availability** uses `getAll` with `timeMin`/`timeMax` — the agent is instructed to only offer slots this tool actually returns, so double-check your calendar's working hours/availability settings reflect your real clinic hours.

---

## 6. Airtable Configuration

**Used by:** `Tool: Save Lead to CRM`, `Tool: Log Appointment in CRM`, `Log Error to CRM`

1. Create an Airtable base with three tables:

   **Leads**

   | Field | Type |
   |---|---|
   | `id` | Single line text (matching column, session ID) |
   | `Name` | Single line text |
   | `Phone` | Single line text |
   | `Last Message` | Long text |

   **Appointment**

   | Field | Type |
   |---|---|
   | `id` | Single line text (matching column, session ID) |
   | `Phone` | Single line text |
   | `Google Event ID` | Single line text |
   | `Doctor` | Single line text |
   | `Appointment Time` | Single line text / Date |

   **Errors**

   | Field | Type |
   |---|---|
   | `Error` | Long text |
   | `Phone` | Single line text |
   | `Message` | Long text |
   | `Timestamp` | Date/time |
   | `Workflow` | Single line text |

2. In n8n: **Credentials → Add Credential → Airtable OAuth2 API**, complete the OAuth consent flow.
3. Open each of the three Airtable nodes (**Tool: Save Lead to CRM**, **Tool: Log Appointment in CRM**, **Log Error to CRM**) and:
   - Select the credential.
   - Set the **Base** to your Airtable base.
   - Set the **Table** to the corresponding table above.
   - For the two `upsert` tool nodes, confirm the **matching column** is set to `id` (the session ID), so repeated messages from the same patient update the same row instead of duplicating it.

---

## 7. Gmail Configuration

**Used by:** `Email Clinic Staff`

1. In n8n: **Credentials → Add Credential → Gmail OAuth2**, complete the OAuth consent flow with the account you want alerts sent *from*.
2. Open the **Email Clinic Staff** node:
   - Select the credential.
   - Set **Send To** to your clinic's real staff alert email address (replace any placeholder value).
   - Review the subject/body template — it includes the workflow name, failed node, error message, and timestamp by default.

---

## 8. WhatsApp BSP Configuration (Cloud Station or equivalent)

**Used by:** `Incoming Message`, `Send Reply`, `Send Fallback Reply`

### 8.1 Get Your BSP's API Details
1. In your BSP's dashboard (e.g. Cloud Station), locate the **send-message API endpoint URL** and the **authentication method** (this workflow assumes Bearer token auth — confirm this matches your provider).

### 8.2 Set the HTTP Header Auth Credential
1. In n8n: **Credentials → Add Credential → HTTP Header Auth**.
2. Set the header name to `Authorization` and the value to `Bearer <your-token>` (or per your BSP's exact auth scheme).
3. Attach this credential to both **Send Reply (Cloud Station HTTP Request)** and **Send Fallback Reply (Cloud Station HTTP Request)**.

### 8.3 Set Environment Variables
1. In n8n: **Settings → Variables**.
2. Add `CLOUDSTATION_API_URL` (your BSP's base send-message URL) and `CLOUDSTATION_API_TOKEN` (if your BSP's auth is referenced via variable rather than the header credential above).

### 8.4 Register the Webhook
1. Copy the **Production Webhook URL** from the **Incoming Message** node in n8n.
2. Paste it into your BSP's dashboard as the inbound webhook/callback URL for your WhatsApp number.
3. Complete any verification handshake your BSP requires (see [installation.md](./installation.md#9-register-the-webhook-with-your-bsp)).

### 8.5 Verify the Payload Shape
Different BSPs format inbound WhatsApp messages differently. This workflow assumes a Meta Cloud API–style shape by default. Before going live, confirm the real payload against the **Normalize Input** node's expressions — see [installation.md](./installation.md#8-verify-the-incoming-payload-shape) for the exact steps.

---

## 9. Node Field Reference

Quick reference for non-credential fields you'll need to fill in manually on the canvas:

| Node | Field | Example Value |
|---|---|---|
| Tool: Check/Book/Reschedule/Cancel Appointment | Calendar ID | `yourclinic@group.calendar.google.com` |
| Tool: Search Knowledge Base | Pinecone Index | `bright-dental-rag` |
| Tool: Save Lead to CRM | Base / Table | Your Airtable Base ID / `Leads` table |
| Tool: Log Appointment in CRM | Base / Table | Your Airtable Base ID / `Appointment` table |
| Log Error to CRM | Base / Table | Your Airtable Base ID / `Errors` table |
| Email Clinic Staff | Send To | Your clinic's real staff email |
| Send Reply / Send Fallback Reply | URL | Your BSP's send-message endpoint |
| Dental Booking Agent | System Prompt | Your clinic's real name, tone, services, doctor(s) |

---

## 10. Rotating & Securing Credentials

- Rotate your **BSP access token** periodically per your provider's recommended schedule.
- Never hard-code API keys, tokens, calendar IDs, or Airtable base IDs directly into node parameters — always use n8n's Credentials Manager and Variables so secrets stay encrypted and out of the exported JSON.
- If you fork or share this workflow's JSON file, double-check that no real clinic-specific IDs (Calendar ID, Airtable Base ID, staff email) were left in before publishing.

---

## Next Steps

- 🚀 [Installation Guide](./installation.md) — full setup walkthrough from import to activation.
- 🗺️ [Workflow Architecture](./workflow-architecture.md) — technical breakdown of every node and connection.