# LeadRescue — Production Readiness Audit
Date: 2026-10-17
Branch: fix/production-readiness-audit

## Changes included in this branch
- WF1 uses deterministic lead ID derived from Twilio CallSid, authenticated webhook placeholder, TwiML Hangup, duplicate-call lookup, one canonical Sheet row, callback-request state and scheduled queue.
- Architecture uses two Twilio numbers: intake/forwarding vs callback/SMS.
- WF2 parses Vapi message.type/message.call/message.artifact payloads, checks matching lead before processing, claims PROCESSING_CALLBACK, routes qualified vs unqualified leads, and does not text ghost customers by default.
- WF3 watches stale and failed intermediate states with a one-hour alert cooldown.
- WF4 dispatches due after-hours callbacks.
- WF5 notifies admin when an n8n execution throws.
- Prompts/schema/docs align on Standard not booking appointments. All workflow exports remain inactive.

## Static checks completed on the branch
- Parsed all five workflow exports and the Vapi structured-output schema as JSON.
- Checked all workflow connection targets refer to existing node names.
- Compiled all n8n Code-node JavaScript and both dynamic Vapi JSON request expressions for syntax.
- Ran synthetic Code-node fixtures for a valid Twilio intake, invalid phone rejection, qualified Vapi call, unqualified Vapi call, gas-smell emergency classification, due scheduled-row filtering, and failed-callback stale monitoring. These are mock-data tests, not provider or live n8n tests.

## Required staging regression tests
1. Valid in-hours Twilio inbound -> TwiML Hangup and exactly one PENDING row.
2. Post same CallSid again -> no second lead/callback.
3. Inbound after hours -> SCHEDULED for next 8am America/Chicago; test daylight-saving change dates.
4. Due scheduled row -> CALL_REQUESTED then VAPI_FIRED with Vapi Call ID.
5. Vapi end-of-call-report updates the matching original lead row.
6. Repeat the same completed report -> no second owner alert.
7. Qualified caller -> owner SMS/email that says NOT BOOKED.
8. Unqualified / wrong number -> row finalized without qualified lead alert.
9. No answer / short call with no customer speech -> owner ghost alert and no automatic customer SMS.
10. Gas/fire/CO/smoke -> emergency route and advice to contact 911/local emergency services.
11. Vapi API failure -> CALL_FAILED and owner alert.
12. Processing failure -> WF5 admin alert; stuck processing -> WF3 alert.
13. Missing or invalid CallSid/phone -> no outbound call.
14. Invalid WF1 Basic Auth / WF2 Authorization secret -> 401/unauthorized.
15. Check Vapi webhook logs and retry behavior using a duplicated end-of-call report.
16. Confirm all 36 sheet headers match exactly and existing rows survive migration.
17. Scan Git history for secrets; rotate credentials that were ever committed.
18. Obtain qualified U.S. legal review of callback consent, recording/retention, texting/opt-out and privacy.

## Limitations and launch gates
- These export JSONs have not been imported or tested in the live Hostinger n8n instance. Node schemas/expression evaluation can vary with n8n version; validate in staging first.
- Credentials and IDs are placeholders; they need to be configured and tested in n8n/Vapi/Twilio/Google.
- WF1 Basic Auth is configured; signature validation at a trusted gateway is recommended. Confirm the configured Twilio webhook sends authenticated requests as expected.
- WF2 relies on the lead ID being preserved in Vapi call assistantOverrides variable values. An unmatched callback goes to admin email for manual investigation and should not create an orphan lead row.
- Google Sheets is not transactional; concurrent duplicate callbacks can race. For strict exactly-once behavior, use PostgreSQL or a data store with a unique key/atomic claim. SMS/email provider retries can duplicate if a send succeeded but its HTTP response was lost.
- Standard has no calendar booking integration and no automatic customer SMS. This code does not establish legal compliance. Keep all workflows inactive until tests pass.
