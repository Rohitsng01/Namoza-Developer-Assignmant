# OrthoNow GTM Event Schema

Tracking event specification for GA4 and Google Ads conversion setups.

---

## 1. Event Schema Matrix

| Event Name | GTM Trigger Type | Key Parameters (min 3) | Target GA4 Report / Audience / Usage |
| :--- | :--- | :--- | :--- |
| `booking_step_complete` | **Custom Event**<br>(Fires on step completion) | 1. `step_number` (Integer)<br>2. `step_name` (String)<br>3. `clinic_location` (String)<br>4. `specialty` (String) | **Funnel Exploration Report** & **Conversions**.<br>Used to track booking funnel drop-offs and build "Booking Abandoners" audiences for retargeting. |
| `call_initiated` | **Just Links** or **Click**<br>(Clicks on `tel:` links) | 1. `click_url` (String: e.g. `tel:+91...`)<br>2. `link_text` (String: e.g. "Call Now")<br>3. `page_path` (String: page URL path) | **Engagement -> Events**.<br>Feeds into "High Intent Leads" audience. Used to measure offline phone call conversions. |
| `whatsapp_chat_initiated` | **Just Links** or **Click**<br>(Clicks on `wa.me` links) | 1. `click_url` (String)<br>2. `widget_position` (String: e.g. `floating_widget`)<br>3. `page_path` (String) | **Engagement -> Events**.<br>Tracks direct chat inquiries, popular starting points for mobile users. |
| `patient_guide_form_submitted`| **Custom Event** / **Form Submission**<br>(Fires on form success) | 1. `form_id` (String)<br>2. `guide_name` (String)<br>3. `page_path` (String) | **Lead Generation Reports**.<br>Feeds "Information Seekers" audience. Used for nurturing via email/SMS. |
| `clinic_page_viewed` | **Page View** / **History Change**<br>(Viewing location details) | 1. `clinic_name` (String)<br>2. `city` (String)<br>3. `page_path` (String) | **Engagement -> Pages and screens**.<br>Allows location-based segmentation and geo-targeted ad campaigns. |
| `blog_article_scroll` | **Scroll Depth**<br>(Fires at scroll thresholds) | 1. `article_title` (String)<br>2. `scroll_depth_percent` (Integer: e.g. 50, 90)<br>3. `page_path` (String) | **Engagement -> Events**.<br>Measures content consumption. High scroll depth (>75%) signals high engagement. |

---

## 2. Multi-Step Booking Form Funnel Tracking

The booking form runs dynamically on a single page (no page reloads or URL changes). We trigger custom `dataLayer` pushes via JavaScript on step completions so GTM can record the transitions.

### dataLayer Payloads (JSON)

#### Step 1: Location & Specialty Selected
Triggers when the patient selects a clinic location and specialty and clicks "Next".
```json
{
  "event": "booking_step_complete",
  "step_number": 1,
  "step_name": "location_specialty_selected",
  "clinic_location": "Indiranagar, Bengaluru",
  "specialty": "Spine & Back Care"
}
```

#### Step 2: Contact Information & Date Entered
Triggers when the patient inputs their contact details, selects a preferred date, and clicks "Next".
* **PII Compliance Note**: Under Google's terms of service and GDPR/HIPAA guidelines, **never** push personally identifiable information (PII) such as names, phone numbers, or emails into the dataLayer. Only non-sensitive parameters should be recorded.
```json
{
  "event": "booking_step_complete",
  "step_number": 2,
  "step_name": "contact_info_entered",
  "preferred_day_of_week": "Wednesday",
  "booking_lead_time_days": 3
}
```

#### Step 3: Booking Confirmed (Final Step)
Triggers when the appointment is successfully written to the database and the confirmation screen is displayed.
```json
{
  "event": "booking_step_complete",
  "step_number": 3,
  "step_name": "booking_confirmed",
  "booking_id": "BK-884920",
  "clinic_location": "Indiranagar, Bengaluru",
  "specialty": "Spine & Back Care"
}
```

---

### GTM Trigger Setup

To capture these events, configure a single Custom Event trigger:

1. **Trigger Name**: `CE - Booking Step Complete`
2. **Trigger Type**: `Custom Event`
3. **Event Name**: `booking_step_complete`
4. **Fires on**: `All Custom Events`
5. **Variables**:
   - `dlv - booking.step_number` (Data Layer Variable: `step_number`)
   - `dlv - booking.step_name` (Data Layer Variable: `step_name`)
   - `dlv - booking.clinic_location` (Data Layer Variable: `clinic_location`)
   - `dlv - booking.specialty` (Data Layer Variable: `specialty`)

---

### GA4 Funnel Configuration

Set up a Funnel Exploration in GA4 using the custom steps:

* **Step 1: Location & Specialty Selected**
  * Event: `booking_step_complete` where `step_number` = 1
* **Step 2: Contact Info Entered**
  * Event: `booking_step_complete` where `step_number` = 2
* **Step 3: Booking Confirmed**
  * Event: `booking_step_complete` where `step_number` = 3

This helps analyze conversion drop-offs and track elapsed time between form steps.

---

## 3. Google Ads Conversion Setup

### Conversion Target
* Event `booking_step_complete` (specifically when `step_number` = 3) or `consultation_form_submitted`.

### Rationale
* **Bottom-of-Funnel Focus**: Optimizing for confirmed bookings ensures Google Ads Smart Bidding (Target CPA or Maximize Conversions) focuses budget on qualified, actual leads instead of early clicks.
* **Avoid Early Step Optimization**: Bidding on Step 1 or Step 2 runs the risk of optimizing for users who start the form but abandon it, leading to wasted ad spend.
* **Call/WhatsApp Click Limitations**: Phone call or WhatsApp link clicks have post-click friction (e.g. hanging up, not sending the message). The completed form is a guaranteed lead record.
