# LeadRescue — Client Setup SOP
Version 8.0 — production-readiness rebuild.
Standard is missed-call recovery and lead qualification, not calendar booking.

## Use two Twilio numbers per client
1. **Intake/forwarding number:** the business line forwards unanswered calls here. Its Incoming Voice webhook points to n8n WF1, which returns TwiML Hangup. Do not assign this number to Vapi.
2. **Callback/SMS number:** a separate Twilio number imported into Vapi and assigned to the saved assistant. Use it for Vapi outbound callbacks and owner SMS to avoid conflicting inbound webhook routing.

## Required placeholders
Replace the Sheet ID, Google Sheets/Twilio/Gmail/Vapi credentials, assistant ID, Vapi callback phone-number ID, callback/SMS number, owner/admin contacts, booking link, business name, AI name, and webhook-auth credential references. Placeholders in workflow JSON are not working credentials.

## 1. Google Sheet
Create a tab named exactly Leads and paste the 36 headers from LeadRescue_Sheets_Headers.csv into Row 1. Back up existing sheets and add missing columns rather than deleting any existing data. Ensure the n8n Google account has Editor access.

## 2. Webhook credentials
Create a separate n8n HTTP Basic Auth credential for WF1. In Twilio, configure the intake number's Voice webhook to use HTTPS and send the matching Basic Auth credentials. In a simple setup this can be embedded in the webhook URL as https://USERNAME:PASSWORD@YOUR_N8N_DOMAIN/webhook/leadrescue-missed-call (URL-encode special characters); do not commit this real URL to GitHub or logs. Alternatively use Twilio's Webhooks Configuration API / auth profile. Twilio supports HTTP Basic/Digest authentication. These credentials authenticate delivery but do not replace request-signature validation; add Twilio signature validation at a trusted gateway/middleware when available.
Create an HTTP Header Auth credential for WF2 and configure Vapi's server custom credential to send the same strong Authorization bearer value. Set the saved assistant server URL to https://YOUR_N8N_DOMAIN/webhook/leadrescue-vapi-callback and enable end-of-call-report event delivery.

## 3. Saved Vapi assistant
Configure OpenAI gpt-6-luna, Cartesia sonic-3.5 with a valid voice ID, and Deepgram nova-3 (English). Paste LeadRescue_Vapi_Prompts_FINAL.md and configure LeadRescue_Vapi_Schema_FINAL.json. The workflows do not override model settings. Keep call recording disabled until recording notice/consent, access control, and retention are approved.

## 4. Import workflows
Import WF1, WF2, WF3, WF4 scheduled-callback dispatcher and WF5 error handler. All exported workflows are inactive intentionally. Bind all placeholders and credentials in n8n. In Settings for WF1–WF4, select WF5 as Error Workflow manually after import because n8n assigns workflow IDs at import time.

## 5. Twilio configuration
The intake number receives carrier call forwarding and is not assigned to Vapi. The callback/SMS number is assigned to the Vapi assistant and configured as REPLACE_TWILIO_FROM_NUMBER for owner alerts. Complete any required messaging registration (such as A2P 10DLC) and test with your own phones, not customers.

Ask the contractor's carrier for exact no-answer forwarding instructions; short codes vary by carrier/account. Test actual forwarding before launch.

## 6. Status and activation
Lifecycle: SCHEDULED → CALL_REQUESTED → VAPI_FIRED → PROCESSING_CALLBACK → COMPLETE. Failures use CALL_FAILED; stale states are monitored. Activate WF5 first, test all flows in staging, connect WF5 as the Error Workflow in WF1–WF4, then activate the other workflows only after the regression test matrix passes.

## 7. Legal and safety
Do not claim TCPA “safe harbor” or complete compliance without qualified U.S. legal advice. Review callback consent, Do Not Call and state laws, recording consent/retention, SMS opt-outs and privacy. Standard sends no automatic customer SMS and does not claim appointments are booked, technicians are dispatched, or guaranteed emergency response.