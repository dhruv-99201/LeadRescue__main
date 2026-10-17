# LeadRescue — Client Setup SOP
# Version: Final 6.0 (Core Product)
# Voice Stack: Cartesia Sonic-3.5 + Deepgram Nova-3 + GPT-4o mini
# ─────────────────────────────────────────────────────────────────────

Use this SOP for every new client you onboard. Complete all 8 phases
in order. Do not activate any workflow until all phases are complete.

Estimated setup time: 45–60 minutes per client.

---

## PLACEHOLDERS TO REPLACE IN ALL FILES

Before starting, collect these values. You will paste them into
workflows, prompts, and the Vapi assistant during setup.

| Placeholder | What to replace with |
|---|---|
| REPLACE_SHEETS_DOC_ID | Google Sheets document ID (from URL) |
| REPLACE_SHEETS_CRED_ID | n8n Google Sheets credential ID |
| REPLACE_VAPI_ASSISTANT_ID | Vapi assistant ID for this client |
| REPLACE_VAPI_PHONE_NUMBER_ID | Vapi phone number ID |
| REPLACE_VAPI_CRED_ID | n8n HTTP Header Auth credential ID for Vapi |
| REPLACE_CARTESIA_VOICE_ID | Cartesia voice ID (Kira or Ariana or cloned) |
| REPLACE_TWILIO_FROM_NUMBER | Client's LeadRescue Twilio number (+1XXXXXXXXXX) |
| REPLACE_TWILIO_CRED_ID | n8n Twilio credential ID |
| REPLACE_GMAIL_CRED_ID | n8n Gmail OAuth2 credential ID |
| REPLACE_OWNER_PHONE | Client owner's mobile number for SMS alerts |
| REPLACE_OWNER_EMAIL | Client owner's email for Gmail alerts |
| REPLACE_BOOKING_LINK | Client's booking/scheduling link |
| REPLACE_BUSINESS_NAME | Client's business display name |
| REPLACE_AI_NAME | Client's chosen AI name (e.g. Reese, Quinn, Casey) |

---

## PHASE 1 — BUY CLIENT'S TWILIO NUMBER

1. Log into Twilio → Phone Numbers → Buy a Number
2. Search for Dallas-Fort Worth area codes: 214, 469, or 972
3. Select a local number (not toll-free)
4. Name it clearly: "LeadRescue — [Client Business Name]"
5. Do NOT configure the webhook here — Vapi will handle that
6. Record the number (E.164 format: +1XXXXXXXXXX)
   → This is your REPLACE_TWILIO_FROM_NUMBER

---

## PHASE 2 — CREATE CLIENT'S GOOGLE SHEET

1. Create a new Google Sheet named:
   "LeadRescue — [Client Business Name]"
2. Rename the default tab to: Leads
3. Paste all headers from LeadRescue_Sheets_Headers.csv into Row 1
   (30 columns — paste starting from cell A1)
4. Bold Row 1. Freeze Row 1 (View → Freeze → 1 row).
5. Copy the Sheet ID from the URL:
   docs.google.com/spreadsheets/d/[THIS_IS_THE_ID]/edit
   → This is your REPLACE_SHEETS_DOC_ID
6. Share the sheet with the Gmail account you connected to n8n
   (Editor access)

---

## PHASE 3 — ADD n8n CREDENTIALS (if not already added)

In n8n → Settings → Credentials:

### Google Sheets OAuth2
- Type: Google Sheets OAuth2 API
- Authenticate with the Gmail account that owns the client sheet
- Note the credential ID → REPLACE_SHEETS_CRED_ID

### Twilio
- Type: Twilio API
- Account SID + Auth Token from Twilio Console
- Note the credential ID → REPLACE_TWILIO_CRED_ID

### Gmail OAuth2
- Type: Gmail OAuth2 API
- Authenticate with client owner's Gmail (or your agency Gmail)
- Note the credential ID → REPLACE_GMAIL_CRED_ID

### Vapi API Key (HTTP Header Auth)
- Type: Header Auth
- Name: Authorization
- Value: Bearer YOUR_VAPI_API_KEY
- Note the credential ID → REPLACE_VAPI_CRED_ID

---

## PHASE 4 — CREATE VAPI ASSISTANT

1. Vapi Dashboard → Assistants → Create Assistant
2. Name it: "[AI Name] — [Client Business Name]"
   Example: "Reese — DFW Cool Air Services"

### System Prompt
Paste contents of Section 1 from LeadRescue_Vapi_Prompts_FINAL.md
Replace [YOUR BUSINESS NAME] with actual client business name

### First Message
Paste contents of Section 2 from LeadRescue_Vapi_Prompts_FINAL.md
Replace {{businessName}} with actual client business name

### Voice Settings
- Provider: Cartesia
- Model: sonic-3.5
- Voice: Kira (warm) or Ariana (professional)
  (or paste cloned voice ID if client purchased voice cloning)
- Speed: 0.95

### Transcriber Settings
- Provider: Deepgram
- Model: nova-3
- Language: en
- Smart Format: ON
- End of Speech Sensitivity: HIGH

### Model Settings
- Provider: OpenAI
- Model: gpt-4o-mini

### Analysis Plan
- Summary Prompt: paste from Section 3 of Prompts file
- Structured Data: paste LeadRescue_Vapi_Schema_FINAL.json
  (Result Format: Multiple fields / Object)

### Server URL
https://YOUR_N8N_DOMAIN/webhook/leadrescue-vapi-callback

4. Save the assistant
5. Record the Assistant ID → REPLACE_VAPI_ASSISTANT_ID

---

## PHASE 5 — ASSIGN TWILIO NUMBER IN VAPI

1. Vapi Dashboard → Phone Numbers → Import
2. Import the Twilio number you bought in Phase 1
   (Vapi will configure the Twilio webhook automatically)
3. Assign to the assistant you just created
4. Record the Phone Number ID → REPLACE_VAPI_PHONE_NUMBER_ID
5. Record the Cartesia Voice ID from the assistant voice settings
   → REPLACE_CARTESIA_VOICE_ID

---

## PHASE 6 — IMPORT AND CONFIGURE n8n WORKFLOWS

Import each workflow JSON file into n8n:

n8n → Workflows → Import from File

Import in this order:
1. LeadRescue_WF1_MissedCallTrigger.json
2. LeadRescue_WF2_PostCallProcessor.json
3. LeadRescue_WF3_StaleMonitor.json

### For EACH workflow, replace ALL placeholders:

Open each workflow and replace every REPLACE_ value using your
collected values table from the top of this SOP.

#### WF1 specific replacements:
- Webhook node: URL will be auto-generated — note it for Phase 7
- Log PENDING Lead node: REPLACE_SHEETS_DOC_ID + REPLACE_SHEETS_CRED_ID
- Fire Vapi Outbound Call node:
  - REPLACE_VAPI_PHONE_NUMBER_ID
  - REPLACE_VAPI_ASSISTANT_ID
  - REPLACE_CARTESIA_VOICE_ID
  - REPLACE_BUSINESS_NAME
  - REPLACE_AI_NAME
  - REPLACE_VAPI_CRED_ID
- Update Sheet node: REPLACE_SHEETS_DOC_ID + REPLACE_SHEETS_CRED_ID

#### WF2 specific replacements:
- All SMS nodes: REPLACE_TWILIO_FROM_NUMBER, REPLACE_OWNER_PHONE,
  REPLACE_TWILIO_CRED_ID
- Qualified SMS: REPLACE_BOOKING_LINK
- Emergency Email: REPLACE_OWNER_EMAIL, REPLACE_GMAIL_CRED_ID
- Qualified Email: REPLACE_OWNER_EMAIL, REPLACE_GMAIL_CRED_ID,
  REPLACE_BOOKING_LINK
- Sheets node: REPLACE_SHEETS_DOC_ID, REPLACE_SHEETS_CRED_ID

#### WF3 specific replacements:
- Sheets read node: REPLACE_SHEETS_DOC_ID, REPLACE_SHEETS_CRED_ID
- SMS node: REPLACE_TWILIO_FROM_NUMBER, REPLACE_OWNER_PHONE,
  REPLACE_TWILIO_CRED_ID

### Rename each workflow:
- "[Client Name] — WF1: Missed Call Trigger"
- "[Client Name] — WF2: Post-Call Processor"
- "[Client Name] — WF3: Stale Lead Monitor"

### Activate all three workflows
Toggle each from Inactive → Active

---

## PHASE 7 — SET UP CLIENT'S CALL FORWARDING

Give the client these instructions based on their phone carrier.

The WF1 webhook URL is shown in n8n → WF1 → Webhook node.
But the client forwards to their TWILIO number — not the webhook directly.
Vapi handles the Twilio → Webhook connection automatically.

### AT&T Mobile
Client dials from their business phone:
*92[LeadRescue Twilio Number]
Example: *92+12145551234

### Verizon Mobile
Client dials: *71[LeadRescue Twilio Number]

### T-Mobile
Client dials: **004*+1XXXXXXXXXX#

### VoIP (RingCentral, Grasshopper, OpenPhone, Google Voice)
Dashboard → Call Forwarding → On No Answer
→ Forward to: [LeadRescue Twilio Number]
→ Ring time: 20 seconds before forwarding

### Office Landline (AT&T Business, Spectrum Business)
Call carrier support → Request "Conditional Call Forwarding — No Answer"
→ Forward to: [LeadRescue Twilio Number]

### Test forwarding is working:
Call the client's business number from another phone.
Do not answer. After ~20 seconds it should forward.
You should receive an inbound Twilio log in your Twilio console.

---

## PHASE 8 — END-TO-END TESTING

Run all three paths before handing off to the client.

### Test 1 — Full Qualified Call
1. Call the client's business number. Do not answer.
2. Wait for Vapi to call back (within 2 min)
3. Answer as a customer. Say: "Yeah hi, my AC isn't cooling the house.
   I'm in Frisco, can someone come out tomorrow morning?"
4. Complete the call naturally.

Expected results:
- ✅ Google Sheet: new row with status COMPLETE, resolved path QUALIFIED
- ✅ SMS to owner: qualified lead alert with service type and location
- ✅ Gmail to owner: full qualified lead email with booking link
- ✅ Vapi dashboard: call recording + structured data visible

### Test 2 — Emergency Path
1. Trigger another missed call
2. Answer as customer. Say: "My AC unit is leaking water everywhere,
   it's flooding my utility room, I need someone now."

Expected results:
- ✅ Sheet: resolved path EMERGENCY
- ✅ SMS: 🚨 EMERGENCY alert with customer phone
- ✅ Gmail: emergency email with issue description

### Test 3 — Ghost Path
1. Trigger another missed call
2. Answer Vapi's callback and immediately hang up

Expected results:
- ✅ Sheet: resolved path GHOST, duration < 15s
- ✅ SMS: ghost alert (no action needed)

### Test 4 — Stale Monitor (optional)
1. Create a fake PENDING row in Sheets with a timestamp 31+ min ago
2. Manually trigger WF3
Expected: SMS stale lead alert fires

---

## HANDOFF CHECKLIST

Before calling the client done:

- [ ] All 3 workflows active
- [ ] Call forwarding confirmed working (tested live)
- [ ] All 3 test paths passed (Qualified / Emergency / Ghost)
- [ ] Google Sheet URL sent to client
- [ ] Client confirmed they received the test SMS alerts
- [ ] Client confirmed they received the test Gmail alerts
- [ ] Vapi call recording visible and voice sounds correct
- [ ] Booking link in SMS/email is correct and working

---

## QUICK REFERENCE — VOICE STACK SUMMARY

| Layer | Provider | Model | Latency |
|---|---|---|---|
| TTS | Cartesia | sonic-3.5 | 90ms |
| STT | Deepgram | nova-3 | <300ms |
| LLM | OpenAI | gpt-4o-mini | <300ms |
| Total round-trip | | | ~690ms |

⚠️ Sonic-2, Sonic-Turbo, Sonic-3-2025-10-27 sunset October 20, 2026.
Always use sonic-3.5 going forward.
