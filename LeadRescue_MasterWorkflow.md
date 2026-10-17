# LeadRescue — Master Workflow
# The complete business flow from missed call to booked appointment
# Version: Final 6.0 | Voice Stack: Cartesia Sonic-3.5 + Deepgram Nova-3
# ─────────────────────────────────────────────────────────────────────

---

## THE BIG PICTURE

```
Customer calls client's business
         │
         ▼
   No one answers
         │
         ▼ (carrier forwards after ~20s ring)
         │
   LeadRescue Twilio Number
         │
         ▼
   n8n WF1 receives call
         │
    ┌────┴────┐
    │ Validate│
    │ + TCPA  │
    └────┬────┘
         │
    Valid + Business hours?
    ├── NO  ──► Log as INVALID or SCHEDULED. Stop.
    │
    └── YES
         │
    ┌────┴────────────────────────────────────────┐
    │  Log lead as PENDING in Google Sheets        │
    │  Wait 2 minutes (n8n Wait node)              │
    │  Fire Vapi outbound call to customer         │
    └────┬────────────────────────────────────────┘
         │
         ▼
   Aria calls customer back
   (Cartesia Sonic-3.5 voice — 90ms TTS)
   (Deepgram Nova-3 STT — <300ms)
   (GPT-4o mini LLM — <300ms)
   Total round-trip latency: ~690ms
         │
   Aria qualifies the customer:
   - What service do they need?
   - How urgent?
   - Which part of DFW?
   - Do they want to book?
         │
   Call ends
         │
         ▼
   Vapi sends callback payload to:
   https://YOUR_N8N_DOMAIN/webhook/leadrescue-vapi-callback
         │
         ▼
   n8n WF2 receives Vapi data
         │
    ┌────┴───────────────────────────────────────────┐
    │  Parse call data                                │
    │  Extract: structuredData.customerLocation       │
    │  Build: locationDisplay fallback chain          │
    │  Flag: isGhost / isEmergency / isQualified      │
    └────┬───────────────────────────────────────────┘
         │
    ┌────┴─────────────────────────────────────────┐
    │             ROUTE BY PATH                     │
    └──────────────────────────────────────────────┘
         │
    ─────┼──────────────────────────────────────────
    │              │                    │
    ▼              ▼                    ▼
  GHOST        EMERGENCY            QUALIFIED
    │              │                    │
    │   Call < 15s │  isEmergency=true  │  isInterested=true
    │   No summary │  Leak/flood/gas    │  OR appointmentRequested
    │   No-answer  │  Fire/smoke etc    │  OR serviceType filled
    │              │                    │
    ▼              ▼                    ▼
  SMS:           SMS:                 SMS:
  ⚠️ Ghost       🚨 Emergency         ✅ Qualified lead
  No action      Call NOW             + booking link
                 + phone              + service + location
                                         │
                     ▼                   ▼
               Gmail:               Gmail:
               Emergency            Qualified lead
               email to owner       email to owner
                    │                    │
                    └────────┬───────────┘
                             │
                             ▼
                  Log full call to Google Sheets
                  (30-column row — all structured data)
                             │
                             ▼
                  Update lead status: COMPLETE

─────────────────────────────────────────────────────────

   RUNNING IN PARALLEL (every 30 minutes):

   n8n WF3 — Stale Lead Monitor
         │
    Read all rows from Google Sheets
         │
    Find any row where:
    - Call Status = PENDING
    - Timestamp > 30 minutes ago
         │
    ├── None found ──► Stop quietly
    │
    └── Found stale leads
         │
         ▼
    SMS to owner:
    ⚠️ X lead(s) stuck as PENDING for 30+ min
    Vapi callback may have failed
    Check n8n dashboard
```

---

## FULL SYSTEM COMPONENTS

### Infrastructure Layer

| Component | Provider | Role |
|---|---|---|
| Workflow automation | n8n (self-hosted) | Orchestrates entire flow |
| Inbound telephony | Twilio | Receives forwarded missed calls |
| Outbound voice AI | Vapi.ai | Fires and manages Aria's calls |
| TTS (voice) | Cartesia Sonic-3.5 | Converts text → speech (90ms) |
| STT (transcription) | Deepgram Nova-3 | Converts speech → text (<300ms) |
| LLM (reasoning) | GPT-4o mini | Powers Aria's conversation (<300ms) |
| SMS alerts | Twilio | Notifies client owner |
| Email alerts | Gmail OAuth2 | Detailed lead emails to owner |
| Lead database | Google Sheets | 30-column real-time dashboard |

### Per-Client Setup (what changes per client)

| Item | Where it lives |
|---|---|
| Twilio phone number | Twilio account |
| Vapi assistant | Vapi dashboard |
| Cartesia voice (Kira/Ariana/cloned) | Vapi → Cartesia |
| Google Sheet | Client's Google Drive |
| WF1 + WF2 + WF3 (duplicated) | n8n instance |
| AI name (Reese, Quinn, etc.) | n8n WF1 assistantOverrides |

### What Never Changes (shared across all clients)

| Item | Notes |
|---|---|
| n8n instance | One instance, separate workflows per client |
| Vapi API key | One key, multiple assistants |
| Cartesia API key | One key, multiple voices |
| Deepgram API key | One key, shared across all assistants |
| Gmail credential | Agency Gmail for all client alerts |
| Twilio account | One account, separate numbers per client |

---

## CALL FLOW DETAIL — ARIA'S CONVERSATION

```
Aria calls customer
        │
        ▼
"Hey is this [Name]? Hi! This is [AI Name] calling
back from [Business] — I saw you tried to reach us
and wanted to make sure we got back to you right away.
What were you calling about today?"
        │
        ▼
Customer responds
        │
   ┌────┴─────────────────────────────────────┐
   │         Aria's qualification goals        │
   │  1. What service/issue?                   │
   │  2. How urgent? (emergency keywords?)     │
   │  3. What part of DFW?                     │
   │     → Ask for neighborhood/city only      │
   │     → NEVER ask for ZIP code              │
   │  4. Preferred appointment window?         │
   │  5. Confirm team will follow up           │
   └────────────────────────────────────────┘
        │
        ▼
Call ends → Vapi sends structured data:
  callOutcome:         booked / interested / callback_requested
                       not_interested / wrong_number / no_answer / incomplete
  isEmergency:         true/false
  serviceType:         AC repair / furnace / plumbing / etc.
  isInterested:        true/false
  appointmentRequested: true/false
  urgencyLevel:        emergency / urgent / standard / flexible
  issueDescription:    customer's own words
  preferredTimeWindow: morning / afternoon / ASAP / etc.
  customerLocation:
    serviceLocation:   exactly as customer said it
    city:              if identifiable
    areaReference:     neighborhood or landmark
    locationConfirmed: true/false
  additionalNotes:     property type, pets, access notes, etc.
```

---

## NOTIFICATION MATRIX

| Path | SMS | Gmail | Sheets |
|---|---|---|---|
| GHOST | ⚠️ Ghost alert (phone, duration, reason) | ✗ | ✅ Full row |
| EMERGENCY | 🚨 Emergency (phone, issue, location, call NOW) | ✅ Emergency email | ✅ Full row |
| QUALIFIED | ✅ Lead summary (service, urgency, location, booking link) | ✅ Full lead email | ✅ Full row |
| UNQUALIFIED | ✗ | ✗ | ✅ Full row |
| STALE (WF3) | ⚠️ Stale lead alert | ✗ | ✗ |

---

## DATA FLOW — GOOGLE SHEETS ROW

Every completed call logs one row with these 30 columns:

```
Lead ID             → LR-1719123456789
Timestamp           → ISO 8601 datetime
Customer Phone      → +12145551234
Customer Name       → "Customer" (Vapi doesn't always capture)
Client Number       → Client's original business number
Call SID            → Twilio call reference
Call Status         → COMPLETE / PENDING / VAPI_FIRED / INVALID
Resolved Path       → GHOST / EMERGENCY / QUALIFIED / UNQUALIFIED
Is Ghost            → TRUE/FALSE
Is Emergency        → TRUE/FALSE
Is Qualified        → TRUE/FALSE
Call Duration (s)   → Integer seconds
Ended Reason        → voicemail / no-answer / customer-ended / etc.
AI Summary          → Aria's natural language call summary
Service Type        → AC repair, furnace, plumbing, etc.
Issue Description   → Customer's exact words
Urgency Level       → emergency / urgent / standard / flexible
Service Location    → Exactly as customer said it
City                → Plano / Frisco / Arlington / etc.
Area Reference      → Bishop Arts / near Love Field / etc.
Location Confirmed  → TRUE/FALSE
Preferred Time      → morning / afternoon / ASAP / etc.
Appointment Requested → TRUE/FALSE
Call Outcome        → booked / interested / not_interested / etc.
Additional Notes    → Property type, pets, gate codes, etc.
Ghost SMS Sent      → (logged separately if needed)
Booking SMS Sent    → (logged separately if needed)
Owner Alerted       → (logged separately if needed)
Gmail Sent          → (logged separately if needed)
Processed At        → ISO 8601 datetime
```

---

## LATENCY BREAKDOWN

```
Customer answers Aria's callback call
        │
  Deepgram Nova-3 transcribes speech ──────► <300ms
        │
  GPT-4o mini processes + generates reply ──► <300ms
        │
  Cartesia Sonic-3.5 renders speech ─────────► 90ms
        │
  Total round-trip ──────────────────────────► ~690ms
                                               (natural conversation)

  Human conversation pause = 200–300ms
  ITU-T G.114 acceptable voice delay = <150ms one-way
  Sub-700ms round-trip = production-grade natural feel
```

---

## GHOST DETECTION LOGIC

A call is flagged GHOST if ANY of these are true:

1. `callDuration < 15 seconds`
   (Call connected but customer hung up immediately)

2. `summary.length === 0`
   (Vapi returned no summary — no real conversation happened)

3. `endedReason` is one of:
   - no-answer
   - voicemail
   - machine
   - no-answer-machine
   - customer-hangup-early

Why 15 seconds (not 10)?
Phone network handshakes take 5–8 seconds on real carrier connections.
A 10s threshold created false ghost flags on legitimate calls.
15s gives Aria enough time to deliver the opening line.

---

## EMERGENCY DETECTION LOGIC

A call is flagged EMERGENCY if EITHER:

1. `structuredData.isEmergency === true`
   (Aria's LLM determined it was an emergency — most reliable)

2. OR any of these keywords appear in summary + transcript:
   emergency, urgent, leak, no heat, burst, flooding, flood,
   gas, fire, no hot water, broken pipe, sewage, overflow,
   smoke, sparks, no ac, no air, carbon monoxide

Both checks run. Either one triggers the emergency path.

---

## TCPA COMPLIANCE

LeadRescue operates within TCPA safe harbor:

- Business hours gate: 8am–9pm CT (America/Chicago timezone)
- All SMS messages include: "Reply STOP to opt out."
- Callback is to a number that called the client first
  (customer initiated the contact — this is a warm callback,
  not a cold outreach)
- One callback attempt per missed call, not retries

---

## FILE INVENTORY

| File | Purpose |
|---|---|
| LeadRescue_WF1_MissedCallTrigger.json | n8n WF1 — import directly |
| LeadRescue_WF2_PostCallProcessor.json | n8n WF2 — import directly |
| LeadRescue_WF3_StaleMonitor.json | n8n WF3 — import directly |
| LeadRescue_Vapi_Prompts_FINAL.md | Paste into Vapi assistant |
| LeadRescue_Vapi_Schema_FINAL.json | Paste into Vapi Analysis Plan |
| LeadRescue_Sheets_Headers.csv | Paste into Google Sheets Row 1 |
| LeadRescue_Client_Setup_SOP.md | Follow this for every new client |
| LeadRescue_MasterWorkflow.md | This file — the big picture |
