# Task 03 — Integration Design: OrthoNow Landing Page → HubSpot + WhatsApp + Google Ads

## End-to-End Integration Architecture

When a patient submits the OrthoNow consultation form, three things must happen automatically: a HubSpot contact is created/updated, a WhatsApp confirmation goes out within 2 minutes, and a Google Ads conversion fires. Here is exactly how I would architect this.

### The Flow

**Landing Page → Serverless Function → HubSpot API → Karix WhatsApp API**

The form's JavaScript `fetch()` call posts `name`, `phone`, and `clinic_preference` to a **serverless function** (AWS Lambda or Vercel Edge Function). This is the right choice over alternatives for three reasons:

- **HubSpot native embed / HubSpot Forms JS API**: Cannot be used here because the form doesn't collect email — HubSpot Forms requires email as the deduplicate key. Embedding a HubSpot form directly would break our custom deduplication logic.
- **Zapier / Make**: Adds latency and a dependency that can fail silently. Make's webhook → HubSpot → Karix chain introduces 3 possible failure points with no native retry logic. Unacceptable for a 2-minute WhatsApp SLA.
- **Direct HubSpot Contacts API + Karix API call from a serverless function**: The right choice. One function, full control, sub-second execution, no third-party middleware.

### Step-by-Step Execution

**Step 1 — Deduplication (the critical step):** The serverless function calls HubSpot's **Contacts Search API** (`POST /crm/v3/objects/contacts/search`) filtering by the custom property `phone` (mapped in HubSpot as `mobilephone`). If a match exists, we call `PATCH /crm/v3/objects/contacts/{id}` to update. If not, we call `POST /crm/v3/objects/contacts` to create. Properties set: `firstname`, `mobilephone`, `hs_lead_status = NEW_ENQUIRY`, and a custom property `lead_source_detail = "Google Ads - Consultation Landing Page"`.

> **⚠️ Phone Deduplication Trap:** HubSpot deduplicates on email by default. If two patients submit with the same phone number but different names, HubSpot's native logic creates two separate contacts. Our Search-first approach prevents this — we search by `mobilephone` before creating.

**Step 2 — WhatsApp via Karix:** Immediately after the HubSpot write (or in parallel using `Promise.all()`), the function calls the Karix WhatsApp Business API to send a templated message to the patient's number. The approved message template is: *"Hi {name}, your consultation request at OrthoNow {clinic} has been received. Our team will call you within 2 hours. — OrthoNow Team."*

**Step 3 — Google Ads Conversion:** This fires client-side via `gtag.js` (loaded through GTM), triggered by the `consultation_form_submitted` dataLayer event. The GTM tag calls `gtag('event', 'conversion', { send_to: 'AW-XXXXXXXX/XXXX' })`. This happens on the browser — no server involvement needed.

---

## Single Biggest Failure Point

**The Karix WhatsApp API call.** If Karix is down, rate-limited, or the patient's number is not WhatsApp-registered, the message fails silently — and the SLA breaks without anyone knowing.

**Fallback:** Wrap the Karix call in a try/catch. On failure, push the job to an **SQS queue** (or any message queue — BullMQ on Redis works too) with the patient's number and a retry policy of 3 attempts over 5 minutes. If all retries fail, fall back to sending an SMS via Karix's SMS API. Log every failure to CloudWatch (or equivalent) with `phone_last4` and `error_code` for monitoring.

---

## WhatsApp 2-Minute SLA: What Could Break It & How to Monitor

**What could break it:**
1. **Karix API latency or downtime** — their API SLA is 99.9%, so ~8.7 hours of downtime per year. Covered by the queue/retry fallback above.
2. **Unregistered WhatsApp number** — ~15–20% of Indian mobile numbers are not on WhatsApp. The delivery receipt will return `undelivered`. Fallback to SMS.
3. **Serverless cold start** — Lambda cold starts can add 2–4 seconds. Mitigated by keeping the function warm with a scheduled ping every 5 minutes.
4. **HubSpot API rate limit** — HubSpot's free tier allows 10 API calls/second. At scale this could throttle. Solution: use HubSpot's Professional/Enterprise tier or implement exponential backoff.

**How to monitor the SLA:**
- Log a `whatsapp_dispatch_attempt` event with `timestamp` and `patient_phone_hash` at the moment of the Karix call.
- Log a `whatsapp_delivered` or `whatsapp_failed` event from Karix's delivery webhook.
- Set a **CloudWatch alarm** that fires if `(whatsapp_dispatch_attempt - form_submit_timestamp) > 120 seconds` for any record.
- Alert the on-call engineer via Slack webhook. Dashboard in Grafana or CloudWatch to track delivery rate and median dispatch latency daily.
