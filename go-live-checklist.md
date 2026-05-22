# RunWise Go-Live Checklist — Paid Pilot

Work through each section in order. Check off every item before moving to the next section.

---

## 1. Environment & Credentials

- [ ] n8n credential for GHL API key is created and tested
- [ ] n8n credential for Retell API key is created and tested
- [ ] n8n credential for Postgres (host, port, DB name, user, password) is created and tested
- [ ] n8n credential for email sending (SMTP or provider) is created and tested
- [ ] No API keys, tokens, or passwords exist as plain text in `ai_voice_client_config` rows
- [ ] All sensitive values live in n8n credentials or environment variables only

---

## 2. Webhook URLs

- [ ] **Lead Intake** — GHL webhook points to `/webhook/hvac-lead-intake` (not `/webhook-test/...`)
- [ ] **Retell Result** — Retell result webhook points to `/webhook/retell-call-result` (not `/webhook-test/...`)
- [ ] **Booking Tool** — Retell custom function URLs point to `/webhook/retell-booking-tool` (not `/webhook-test/...`)
- [ ] **Magic Email** — GHL inbound email webhook points to `/webhook/magic-email` (not `/webhook-test/...`)
- [ ] All five webhook paths confirmed active and returning 200 on a test POST

---

## 3. Database

- [ ] `ai_voice_leads` table exists with all required columns: `id`, `client_key`, `lead_name`, `phone`, `issue`, `source`, `outcome`, `transcript`, `first_name`, `last_name`, `email`, `created_at`, `last_submitted_at`, `call_id`, `recording_url`, `summary`, `appointment_booked`, `appointment_window`, `updated_at`
- [ ] `ai_voice_client_config` table exists with all required columns
- [ ] Views created: `ai_voice_lead_summary`, `ai_voice_recent_leads`, `ai_voice_recent_leads_readable`
- [ ] All DB lookups and updates use `client_key + phone` composite (not phone alone) everywhere n8n supports it or via SQL
- [ ] `client_key` is written and updated consistently on every insert/upsert to `ai_voice_leads`
- [ ] Dedupe check queries on `client_key + phone + last_submitted_at` window — not phone alone

---

## 4. GHL Setup

- [ ] GHL form webhook trigger is set to POST to the production lead intake URL
- [ ] GHL form / custom data includes `client_key` field set to the correct client key (e.g. `abc_heating_air`)
- [ ] GHL contact create/update node tested — contact appears or updates correctly in GHL
- [ ] All outcome tags exist in GHL: `retell_call_analyzed`, `retell_no_answer`, `retell_failed`, `retell_booked`, `needs_dispatcher_review`
- [ ] GHL calendar is connected and returning free slots via the booking tool
- [ ] GHL appointment creation tested — appointment appears correctly in GHL calendar
- [ ] GHL call note is written correctly after a call result
- [ ] (Optional) GHL follow-up workflows reviewed — none will fire prematurely on new leads

---

## 5. Retell Setup

- [ ] Agent name/persona confirmed (Jordan, calm/professional)
- [ ] Agent system prompt reviewed: no pricing quotes, no HVAC diagnosis, arrival windows only, safety escalation language present
- [ ] Dynamic variables mapped: `customer_first_name`, `customer_name`, `customer_phone`, `phone_last_4_spoken`, `service_requested`, `lead_source`
- [ ] Metadata fields passing: `ghl_contact_id`, `lead_source`, `customer_first_name`, `customer_name`, `customer_phone`, `phone_last_4`, `phone_last_4_spoken`, `service_requested`
- [ ] Custom function `check_available_times` URL points to production booking tool webhook
- [ ] Custom function `book_appointment` URL points to production booking tool webhook
- [ ] Retell result webhook URL set to production `/webhook/retell-call-result`
- [ ] Retell from-number confirmed and active
- [ ] Test call placed — agent answers, uses correct persona and client name
- [ ] `phone_last_4_spoken` confirmed formatted correctly (e.g. "four four four four")

---

## 6. Client Config Row

Confirm every column in `ai_voice_client_config` is populated for this client:

- [ ] `client_key` — matches key used in GHL custom data and all n8n nodes
- [ ] `company_name` — matches what the agent should say on calls
- [ ] `ghl_location_id`
- [ ] `ghl_calendar_id`
- [ ] `retell_agent_id`
- [ ] `retell_from_number`
- [ ] `owner_email`
- [ ] `dispatcher_email`
- [ ] `timezone` (e.g. `America/Chicago`)
- [ ] `arrival_window_minutes` (e.g. `120` for a 2-hour window)
- [ ] `service_area_zips` (comma-separated or JSON array, tested with at least one in-area and one out-of-area ZIP)
- [ ] `business_hours_start` (e.g. `08:00`)
- [ ] `business_hours_end` (e.g. `17:00`)
- [ ] `business_days` (e.g. `Mon-Fri`)
- [ ] `after_hours_urgent_mode` — decision made and set: `book_only`, `escalate_only`, or `book_and_escalate`
- [ ] `active` set to `true`

---

## 7. Notification Routing

- [ ] Owner email address in config is correct and reachable
- [ ] Dispatcher email address in config is correct and reachable
- [ ] Owner notification email received on a test booked call
- [ ] Dispatcher review email received on a test needs-review call
- [ ] Daily client report email received by owner (manual trigger or wait for schedule)
- [ ] Error alert email received when a test workflow error is triggered

---

## 8. End-to-End Test Matrix

Run each scenario and verify the full result in GHL, the DB, and email.

| Scenario | Tested | GHL Tag | DB Updated | Email Sent |
|---|---|---|---|---|
| GHL form lead → call launched | [ ] | [ ] | [ ] | — |
| Magic email lead → call launched | [ ] | [ ] | [ ] | — |
| Normal booking (in-area ZIP, available slot) | [ ] | `retell_booked` [ ] | `appointment_booked = true` [ ] | Owner notified [ ] |
| No answer / voicemail | [ ] | `retell_no_answer` [ ] | `outcome` updated [ ] | — |
| Failed call | [ ] | `retell_failed` [ ] | `outcome` updated [ ] | — |
| Needs dispatcher review | [ ] | `needs_dispatcher_review` [ ] | `outcome` updated [ ] | Dispatcher email [ ] |
| Out-of-area ZIP | [ ] | — | — | Agent responds out-of-area [ ] |
| After-hours lead (urgent mode) | [ ] | correct tag [ ] | [ ] | correct routing [ ] |
| Daily client report (scheduled) | [ ] | — | — | Report received [ ] |
| n8n error alert | [ ] | — | — | Alert email received [ ] |

---

## 9. Client Decision Points

Confirm each decision has been made and is reflected in config and workflow logic:

- [ ] **After-hours urgent behavior** decided: `book_only` / `escalate_only` / `book_and_escalate`
- [ ] **Hard booking vs. dispatcher confirmation** decided: agent books directly / or all bookings require dispatcher review
- [ ] **Booking system** decided: GHL calendar / ServiceTitan / Housecall Pro / Zapier/Make / direct API / dispatcher-review only
- [ ] Decisions documented and config row updated to match

---

## 10. Client QA Sign-off

- [ ] Client (or designated rep) has watched at least one full end-to-end test live
- [ ] Client understands GHL outcome tags and what each means
- [ ] Client knows how to find and action `needs_dispatcher_review` leads in GHL
- [ ] Client knows where to find call notes, transcript summary, and recording in GHL
- [ ] Client has received and reviewed a sample daily report email
- [ ] Client has confirmed company name, from-number, and agent persona sound correct
- [ ] Client sign-off obtained (signature, email confirmation, or written approval)

---

## 11. Post-Launch Monitoring — First 48 Hours

- [ ] First live inbound lead monitored end-to-end in real time
- [ ] Retell call placed within 60 seconds of lead submission
- [ ] DB row created with correct `client_key`, `phone`, `source`, and `outcome`
- [ ] GHL contact created or updated correctly
- [ ] Correct outcome tag applied in GHL
- [ ] Call note and summary written to GHL contact
- [ ] If booked: appointment visible in GHL calendar with correct arrival window
- [ ] Owner/dispatcher notifications delivered as expected
- [ ] No unexpected n8n errors or error alert emails
- [ ] After 48 hours: review DB for any leads with missing `outcome`, `client_key`, or `call_id` and investigate

---

*Mark this document complete only when every checkbox above is checked and client sign-off is obtained.*
