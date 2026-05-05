# 📞 Outbound Call Agent

An AI-powered outbound calling system built with **VAPI**, **n8n**, **Google Gemini**, and **Google Sheets/Calendar**. Automatically initiates calls to leads, handles live tool calls mid-call, analyzes transcripts post-call, logs results, and books discovery calls — all hands-free.

---

## 🧠 What It Does

- Triggers an outbound call via VAPI when a lead submits a form
- Handles live tool calls mid-call (e.g. ticket creation) via webhook
- Waits for the call to end, then fetches the full transcript
- Passes the transcript to a Gemini AI Agent for analysis
- Extracts call status, summary, and booking details
- Logs everything to Google Sheets
- Creates a Google Calendar event if the lead booked
- Sends confirmation or follow-up emails based on outcome

---

## 🏗️ Architecture

```
Form Submission
      ↓
Edit Fields (extract lead data)
      ↓
Initiate Call (VAPI API)
      ↓
Wait → Poll Call Status (HTTP Request to VAPI)
      ↓
IF status == "ended"
  ├── TRUE  → AI Agent (Gemini) analyzes transcript
  │               ↓
  │           Structured Output Parser
  │               ↓
  │           Append Row in Google Sheets
  │               ↓
  │           IF status == "Booked"
  │             ├── TRUE  → Create Google Calendar Event → Send confirmation email
  │             └── FALSE → Send follow-up email
  └── FALSE → Wait 2min → retry (max 5 retries)

Webhook (VAPI tool calls / end-of-call-report)
      ↓
IF message.type == "tool-calls"
  └── Extract args via bracket notation → Handle tool (e.g. ticket to Sheets)
```

---

## 🛠️ Tech Stack

| Tool | Purpose |
|------|---------|
| [VAPI](https://vapi.ai) | Outbound voice call orchestration |
| [n8n](https://n8n.io) | Workflow automation |
| Google Gemini (via n8n) | Transcript analysis & structured output |
| Google Sheets | Lead & call logging |
| Google Calendar | Booking discovery calls |
| Gmail | Confirmation & follow-up emails |

---

## 🔄 n8n Workflow Breakdown

### Bottom Flow — Call Initiation
1. **On Form Submission** — trigger on new lead
2. **Edit Fields** — extract `userName`, `email`, `formattedNumber`
3. **Form-call Wait** — brief delay before initiating
4. **Initiate a Call** — POST to `https://api.vapi.ai/call` with assistant config
5. **Wait2** — wait for call to progress
6. **HTTP Request** — GET call details from VAPI (`/call/{id}`)
7. **IF2** — check if `status == "ended"`
   - **False** → increment retry counter → Wait 2min → retry GET
   - **True** → pass to AI Agent

### Top Flow — Webhook Handler
1. **Webhook** — receives all VAPI events
2. **Edit Fields3** — route based on `message.type`
   - `tool-calls` → extract args using bracket notation → handle tool
   - `end-of-call-report` → extract transcript → AI Agent

### AI Analysis Flow
1. **AI Agent (Gemini)** — analyzes transcript, returns structured JSON
2. **Structured Output Parser** — enforces consistent schema
3. **Code Node** — normalizes output structure across runs
4. **Append Row in Sheet** — logs name, email, phone, transcript, summary, status, booking
5. **IF (Booked?)** — branches on status
   - **Booked** → Google Calendar event + confirmation email
   - **Unsure/Rejected** → follow-up email

---

## 📋 AI Agent Output Schema

```json
{
  "analysis": {
    "status": "Booked | Unsure | Rejected",
    "booking_details": {
      "date": "YYYY-MM-DD",
      "time": "HH:MM AM/PM"
    },
    "summary": "Brief summary of the call"
  }
}
```

---

## ⚙️ Environment Setup

### VAPI
- Create an assistant with your system prompt
- Add tool definitions with parameters: `user_name`, `user_email`, `user_query`
- Set **Server URL** to your n8n production webhook URL
- Use **test webhook URL** only during development

### n8n
- Connect **Google Sheets**, **Google Calendar**, **Gmail** credentials
- Set workflow webhook to **Production** URL for live calls
- Configure **Structured Output Parser** schema to match AI output

### Google Sheets
Ensure your sheet has these columns:
```
Name | Email | Phone Number | Call Transcript | Call Summary | Status | Booking Date/Time | Follow-up
```

---

## ⚠️ Known Gotchas

**`arguments` security error in n8n**
n8n blocks direct access to properties named `arguments` in expressions. Use bracket notation instead:
```javascript
toolCall.function["arguments"].user_name
```
Or use a Code node / Edit Fields node with `["arguments"]` syntax.

**Transcript is null mid-call**
VAPI only provides the transcript after the call ends and processing completes. Always poll for `status == "ended"` before fetching transcript — never use a fixed wait time.

**AI output structure inconsistency**
Gemini can return varying JSON structures across runs. Use a Code node to normalize:
```javascript
const analysis = data.output?.analysis || data.analysis || data.output || data;
```

**Google Calendar date format**
VAPI/AI returns human-readable dates like `"Wednesday at 10:00 AM"`. Convert to ISO 8601 before passing to Google Calendar:
```javascript
bookingDate.toISOString() // → "2026-05-07T10:00:00.000Z"
```

---

## 📁 Project Structure

```
outbound-call-agent/
├── README.md
├── workflow/
│   └── outbound_call_agent.json   # n8n workflow export
└── prompts/
    └── agent_system_prompt.txt    # VAPI assistant system prompt
```

---

## 🚀 Deployment

1. Import `outbound_call_agent.json` into your n8n instance
2. Connect all credentials (Google, VAPI)
3. Set production webhook URL in VAPI server settings
4. Activate the workflow
5. Submit a test lead via your form

---

## 👤 Author

Built by **[Uzikhan ](https://github.com/muhammad-uzair-khan7)**  
AI Automation & Integration Development
