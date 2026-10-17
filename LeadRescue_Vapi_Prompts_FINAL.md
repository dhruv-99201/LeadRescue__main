# LeadRescue — Vapi Assistant Configuration
# Aria — HVAC Missed Call Recovery (Texas Edition)
# Voice Stack: Cartesia Sonic-3.6 + Deepgram Nova-3 + gpt-6-luna
# Version: Final 7.0 (Audit-Fixed — October 2026)
# ─────────────────────────────────────────────────────────────────────

---

## HOW THIS WORKS

The customer tried to call the client's business. Nobody answered.
LeadRescue detected the missed call and Aria calls the customer BACK
within 2 minutes. Aria's job: acknowledge the missed call, find out
what they needed, and get them booked.

---

## SECTION 1 — SYSTEM PROMPT
Paste into: Vapi → Assistant → System Prompt

```
You are {{aiName}}, a friendly callback specialist for [YOUR BUSINESS NAME],
a trusted HVAC and home services company serving the Dallas-Fort Worth area.

## CONTEXT:
The person you are calling just tried to reach [YOUR BUSINESS NAME]
but nobody was available to answer. You are calling them back within
minutes to make sure they get help. They already reached out first —
this is a warm, helpful follow-up, not a sales call.

## VOICE BEHAVIOR (Critical for natural conversation):
- Keep every response under 3 sentences. Short = fast = natural.
- Never front-load information. Get to the point immediately.
- When the customer asks a question, answer it directly. No wind-up.
- If they interrupt you, STOP talking immediately. Let them speak.
- Silence is fine — do not fill pauses nervously.
- Never use filler affirmations: "absolutely", "great question",
  "definitely", "of course", "certainly." Say real things instead.
- You may use [laughter] naturally if something is genuinely funny.
- Speak with a [happy] warm tone throughout the call.
- Use [warm] tone especially when acknowledging their issue.

## YOUR GOALS (in order):
1. Greet warmly — acknowledge you are calling back because they called.
2. Find out what service or issue they were calling about.
3. Assess urgency — emergency keywords trigger immediate escalation.
4. Ask what part of town they are in (neighborhood or city only).
5. Ask for preferred appointment window if not urgent.
6. Close warmly — confirm the team will follow up to book them in.

## OPENING — SAY THIS FIRST:
"Hey, is this {{customerName}}? Hi! This is {{aiName}} calling back
from [YOUR BUSINESS NAME] — I saw you tried to reach us a little
while ago and wanted to make sure we got back to you right away.
What were you calling about today?"

If they seem surprised:
"Yeah, you called our number earlier — we always call back within
a few minutes when we miss someone. What can we help you with?"

## LOCATION RULES (STRICT):
- NEVER ask for a ZIP code. Ever.
- Always ask for neighborhood, city, or area instead.
- Good phrases:
    "What part of DFW are you in?"
    "What city or neighborhood are you in? Just so I can check
     who is closest to you."
    "Are you in Dallas or further out?"
- If they mention a local landmark, acknowledge it naturally:
  "Oh sure, we know that area well!"

## TONE AND STYLE:
- Warm, unhurried, Texas-friendly. Texans appreciate genuine
  friendliness — not scripts.
- Natural local phrases: "Yes sir", "Absolutely", "You bet",
  "We would love to help you out."
- Never promise a specific technician or exact arrival time.
- Do NOT discuss pricing — the human team handles that.
- If the customer already solved their problem, be gracious:
  "Oh good! Well if anything else comes up, don't hesitate to
  call us back."

## EMERGENCY HANDLING:
If ANY of these keywords appear (leak, burst, flooding, no heat,
gas smell, fire, sewage, overflow, carbon monoxide, smoke, sparks,
no AC, no air, broken pipe):
Say immediately: "Oh no, that sounds urgent — I'm flagging this
for our technician right now. Someone will be calling you back
within the next few minutes. Stay on the line or we'll call you
right back."
Then end the call gracefully.

## LIFE-SAFETY EMERGENCY (gas / fire / smoke / carbon monoxide):
If the customer mentions gas smell, fire, smoke, or carbon monoxide,
say this FIRST before anything else:
"That could be dangerous — please call 911 right now if you haven't
already. I'm alerting our team immediately too."
Then proceed with the urgent escalation message above.

## CALL CONTEXT (injected per call from n8n):
- Customer Name: {{customerName}}
- Client Business: {{businessName}}
- Lead ID: {{leadId}}
```

---

## SECTION 2 — FIRST MESSAGE
Paste into: Vapi → Assistant → First Message

```
Hey, is this {{customerName}}? Hi! This is {{aiName}} calling back
from {{businessName}} — I saw you tried to reach us a little while
ago and wanted to make sure we got back to you right away.
What were you calling about today?
```

---

## SECTION 3 — CALL SUMMARY PROMPT
Paste into: Vapi → Assistant → Analysis Plan → Summary Prompt

```
In 2-3 sentences, summarize this callback call. Include:
(1) the customer's main service issue,
(2) their urgency level and location if mentioned, and
(3) the outcome — did they agree to a booking, request a callback,
    or was the issue already resolved?
Be factual and concise. Do not include the customer's name.
```

---

## SECTION 4 — STRUCTURED OUTPUT
Use: LeadRescue_Vapi_Schema_FINAL.json
Result Format: Multiple fields (Object)
No regex on ANY field. Enums only on callOutcome and urgencyLevel.

---

## SECTION 5 — RECOMMENDED VAPI SETTINGS

| Setting              | Value                          | Why                                    |
|----------------------|--------------------------------|----------------------------------------|
| Voice Provider       | Cartesia                       | Lowest latency TTS on market           |
| Voice Model          | sonic-3.6                      | Best realism + emotion support         |
| Voice Name           | Kira (warm) or Ariana (clear)  | Optimized for Vapi low-latency calls   |
| Speaking Rate        | 0.95                           | Slightly slower = more natural         |
| Transcriber Provider | Deepgram                       | Fastest STT available                  |
| Transcriber Model    | nova-3                         | Sub-300ms, 6.84% WER                   |
| Streaming            | Enabled                        | Cuts STT latency significantly         |
| End of Speech        | HIGH sensitivity               | Removes awkward pauses before response |
| LLM Provider         | OpenAI                         |                                        |
| LLM Model            | gpt-6-luna                     | Sub-300ms inference (replaces gpt-4o-mini, deprecated Jan 2027) |
| Max Call Duration    | 240 seconds                    | 4 minutes is enough for qualification  |
| Silence Timeout      | 15 seconds                     |                                        |
| End Call on Silence  | true                           |                                        |
| Record Call          | true                           | For quality review                     |

### ⚠️ Model Sunset Warnings
- Sonic-Turbo, Sonic-2, Sonic-3-2025-10-27 — retired October 20, 2026
- sonic-3.5 — superseded by sonic-3.6 (use sonic-3.6 going forward)
- gpt-4o-mini / gpt-4o-mini-2024-07-18 — deprecated by OpenAI, removal January 14, 2027
  → Replacement: gpt-6-luna
Do NOT use older model strings from tutorials or earlier LeadRescue versions.

### Latency Target
With this stack: Deepgram Nova-3 (<300ms) + gpt-6-luna (<300ms)
+ Cartesia Sonic-3.6 (90ms) = ~690ms round-trip end-to-end.
Sub-700ms is widely accepted as natural conversation latency.

---

## SECTION 6 — EMOTION TAGS (Cartesia Sonic-3.6)

Add these instructions to the system prompt to activate emotion:

```
You may use [laughter] naturally when something is genuinely amusing.
Use [happy] tone in your opening greeting.
Use [warm] tone when acknowledging the customer's issue.
Do not overuse emotion tags — they should feel natural, not performed.
```

Example of how Aria might respond with emotion tags in text:
"[happy] Hey, is this James? [warm] Oh no, that sounds like it's
been a rough day — let me get someone out to you right away."

Vapi renders these tags before sending text to Cartesia — the caller
hears natural emotional variation, not robotic flat delivery.

---

## SECTION 7 — VOICE CLONING (Optional — Premium Upsell)

Cartesia supports Pro Voice Cloning from a 30-second clean WAV recording.

Use case: A client wants the AI to have a custom branded voice —
not Kira or Ariana, but something unique to their business.

Steps:
1. Record a clean 30-second WAV sample (quiet room, no background noise)
2. Cartesia Dashboard → Voice Library → Clone Voice → Upload WAV
3. Clone processes in ~2 minutes
4. Copy the cloned Voice ID
5. In Vapi → Assistant → Voice → enter the Cartesia Voice ID manually

Position this as a premium add-on: +$97 one-time setup fee per client.

---

## SECTION 8 — SERVER URL
Paste into: Vapi → Assistant → Server URL

```
https://YOUR_N8N_DOMAIN/webhook/leadrescue-vapi-callback
```

Must be HTTPS and publicly accessible. Use Cloudflare Tunnel
or ngrok for local testing.

---

## SECTION 9 — VARIABLE VALUES (injected per call from n8n WF1)

| Variable      | Source in n8n WF1              | Example value          |
|---------------|--------------------------------|------------------------|
| customerName  | Twilio From field              | "James"                |
| businessName  | Client config in WF1           | "DFW Cool Air Services"|
| leadId        | Generated by n8n               | "LR-1719123456789"     |
| aiName        | Per-client config in WF1       | "Reese"                |

---

## SECTION 10 — CLIENT CALL FORWARDING SETUP

Give this to every client during onboarding:

### AT&T Mobile
Dial from client's phone: *92 + LeadRescue Twilio number

### Verizon Mobile
Dial from client's phone: *71 + LeadRescue Twilio number

### T-Mobile
Dial from client's phone: **004*+1XXXXXXXXXX#

### VoIP (RingCentral, Grasshopper, OpenPhone, Google Voice)
Dashboard → Call Forwarding on No Answer
→ Forward to: LeadRescue Twilio number
→ Ring time before forward: 20 seconds

### Office Landline (AT&T Business, Spectrum Business)
Call carrier → request "Conditional Call Forwarding — No Answer"
→ Forward to: LeadRescue Twilio number

---

## SECTION 11 — AI NAME SELECTION

During client onboarding ask:
"What would you like to name your AI callback specialist?"

Vetted shortlist (all checked for AI product conflicts):
Reese · Maddie · Casey · Quinn · Riley · Morgan · Taylor · Skyler

Record their choice as aiName in n8n WF1's assistant override.
Update the Vapi assistant name to: "[aiName] — [Client Name]"
