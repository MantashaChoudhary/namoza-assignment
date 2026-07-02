# Task 01 — GTM Event Schema for OrthoNow

## Full GTM Event Schema

| Event Name | Trigger Type | Key Parameters | GA4 Report / Audience |
|---|---|---|---|
| `booking_step_complete` | Custom Event (dataLayer.push) | `step_number`, `step_name`, `clinic_location` | Funnel Exploration, Conversion Events |
| `booking_confirmed` | Custom Event (dataLayer.push) | `clinic_location`, `specialty`, `booking_date` | Conversions, Google Ads import |
| `call_now_click` | Click — All Clicks (CSS selector `.call-now-btn, a[href^="tel:"]`) | `page_location`, `button_position`, `clinic_name` | Events Report, High-Intent Audience |
| `whatsapp_chat_open` | Click — Just Links (`a[href*="wa.me"]`) | `page_location`, `widget_type`, `link_url` | Engagement Report, Re-engagement Audience |
| `patient_guide_form_submit` | Custom Event (dataLayer.push on form submit) | `page_location`, `guide_name`, `form_id` | Lead Gen Funnel, Top-of-Funnel Audience |
| `patient_guide_download` | Custom Event (dataLayer.push after gate clears) | `guide_name`, `page_location`, `user_city` | Lead Nurture Audience |
| `clinic_page_view` | Page View (URL contains `/clinic/`) | `clinic_name`, `city`, `page_path` | Location Engagement, City-Segment Audience |
| `blog_scroll_depth` | Scroll Depth (25%, 50%, 75%, 90%) | `scroll_depth_percent`, `article_title`, `time_on_page` | Content Engagement, Retargeting Audience |

---

## 3-Step Booking Form Funnel Tracking

### How it works
Each form step fires a custom `dataLayer.push()` from the **front-end developer's code** — GTM cannot natively detect multi-step form progression. GTM listens for the custom event name `booking_step_complete` via a **Custom Event trigger**, and tags fire accordingly.

---

### GTM Trigger Configuration (per step)

| Step | GTM Trigger Name | Trigger Type | Event Name Condition |
|---|---|---|---|
| Step 1 | `Trigger - Booking Step 1` | Custom Event | Event name equals `booking_step_complete` AND `step_number` equals `1` |
| Step 2 | `Trigger - Booking Step 2` | Custom Event | Event name equals `booking_step_complete` AND `step_number` equals `2` |
| Step 3 | `Trigger - Booking Confirmed` | Custom Event | Event name equals `booking_step_complete` AND `step_number` equals `3` |

---

### DataLayer Push — Step 1: Location & Specialty Selected

```json
{
  "event": "booking_step_complete",
  "step_number": 1,
  "step_name": "location_specialty_selected",
  "clinic_location": "Koramangala, Bengaluru",
  "specialty": "Knee Replacement",
  "page_location": "/book-consultation"
}
```

### DataLayer Push — Step 2: Contact Details Entered

```json
{
  "event": "booking_step_complete",
  "step_number": 2,
  "step_name": "contact_details_entered",
  "clinic_location": "Koramangala, Bengaluru",
  "specialty": "Knee Replacement",
  "preferred_date": "2026-07-10",
  "page_location": "/book-consultation"
}
```

> Note: Do NOT push `patient_name` or `patient_phone` into the dataLayer — these are PII fields. Only push non-PII metadata.

### DataLayer Push — Step 3: Booking Confirmed (Conversion)

```json
{
  "event": "booking_step_complete",
  "step_number": 3,
  "step_name": "booking_confirmed",
  "clinic_location": "Koramangala, Bengaluru",
  "specialty": "Knee Replacement",
  "preferred_date": "2026-07-10",
  "booking_id": "ORN-20260701-4821",
  "page_location": "/book-consultation"
}
```

---

### Surfacing Step-Level Drop-off in GA4 Funnel Exploration

1. Go to **GA4 → Explore → Funnel Exploration**
2. Set funnel steps:
   - Step 1: Event = `booking_step_complete` with parameter `step_number = 1`
   - Step 2: Event = `booking_step_complete` with parameter `step_number = 2`
   - Step 3: Event = `booking_step_complete` with parameter `step_number = 3`
3. Enable **"Open funnel"** to allow users who enter mid-funnel
4. Breakdown dimension: `clinic_location` → reveals which clinic pages lose users at which step
5. Date comparison: Week-over-week to catch regressions after deployments

---

## Recommended Google Ads Conversion Action

**Import: `booking_step_complete` where `step_number = 3` (booking_confirmed)**

**Why this one over others:**

- `call_now_click` is high-intent but unverifiable — we can't confirm the call connected or resulted in a booking
- `whatsapp_chat_open` is even weaker as a signal — a user might open and immediately close
- `patient_guide_download` is a top-of-funnel action; optimising Google Ads toward it would bring in research-phase users who don't convert to appointments
- `booking_confirmed` (Step 3) is the only action that represents **confirmed patient intent with contact details submitted** — making it the highest-quality signal to feed into Google Ads Smart Bidding (Target CPA or Target ROAS). This directly maps to OrthoNow's business goal: booked consultations.
