# LeadRescue — Master Workflow
Version 8.0 — corrected Standard architecture.

## Product promise
Standard recovers missed calls, calls the customer back once, qualifies the need, and alerts the owner. It records a preferred time window but does not reserve a calendar slot. Pro booking is a separate future module.

## Two-number routing
- **Intake Twilio number:** the contractor's carrier forwards unanswered calls here. Its incoming Voice webhook points to WF1 and returns TwiML Hangup. Do not assign the intake number to a Vapi assistant.
- **Callback/SMS number:** a separate Twilio number imported into Vapi and assigned to the saved assistant. Vapi uses its phoneNumberId for outbound calls; Twilio uses it for owner notification SMS.

## Workflow path
1. Authenticated Twilio intake enters WF1. WF1 returns TwiML Hangup promptly, validates the phone and CallSid, and uses deterministic Lead ID LR-{CallSid}.
2. WF1 checks for an existing row, skips duplicate inbound CallSid, and writes one canonical Lead ID row.
3. During the default daily 8am–9pm America/Chicago callback window, WF1 records CALL_REQUESTED, waits two minutes, then calls Vapi. Outside the window, it records SCHEDULED for 8am Central; WF4 dispatches due rows every five minutes.
4. WF1 and WF4 store Vapi Call ID and status. Failed call creation becomes CALL_FAILED and notifies the owner.
5. Vapi sends message.type=end-of-call-report to authenticated WF2. WF2 parses message.call, message.artifact transcript/messages and structured outputs.
6. WF2 matches the existing lead before proceeding, ignores reports for COMPLETE/PROCESSING_CALLBACK rows, claims PROCESSING_CALLBACK before notification, then routes emergency/ghost/qualified/unqualified.
7. Qualification requires a meaningful service type and explicit customer interest or a scheduling request. Summary length alone is not qualification. Ghost results alert the owner only; Standard does not send automatic customer SMS.
8. WF3 monitors stale and failed states every ten minutes with a one-hour row alert cooldown. WF5 is the workflow error handler; select it manually as the Error Workflow for WF1–WF4 after import.

## Saved assistant models
Configure Vapi with OpenAI gpt-6-luna, Cartesia sonic-3.5, and Deepgram nova-3. The n8n workflows do not override voice/model provider settings. Verify each is selectable in the actual account before deployment.

## Reliability limits
Google Sheets append-or-update is not transactional; concurrent webhook retries can race. For stronger idempotency, use PostgreSQL or an n8n Data Table with a unique key/atomic claim. Provider-accepted but timed-out SMS requests can still duplicate during retries; track outcomes and reconcile.

## Security and compliance
WF1 uses HTTP Basic Auth over HTTPS; add Twilio signature validation at a trusted gateway/middleware where available. WF2 requires an Authorization bearer credential also configured in Vapi. Keep credentials, transcripts, recordings and real customer data out of GitHub. This repository does not establish TCPA compliance or a universal “safe harbor”; obtain qualified U.S. legal review for consent basis, Do Not Call/state laws, recording/retention, SMS and privacy. Keep recording off until approved.
Use the exact 36 columns in LeadRescue_Sheets_Headers.csv. See LeadRescue_Production_Readiness_Audit.md for launch tests.