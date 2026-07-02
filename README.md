# OrthoNow Developer Assignment (Client Web & Martech)

This repository contains the completed deliverables for the **OrthoNow** client developer assignment.

---

## 📂 Repository Directory Structure

* **`index.html`** - The primary, high-converting landing page (a single, self-contained file containing all styling and JavaScript as requested in the assignment brief).
* **`task_01_gtm_schema.md`** - Task 1: Complete GTM Event Schema table and dynamic dataLayer pushes.
* **`task_03_integration_design.md`** - Task 3: In-depth integration design report.
* **`Loom_Walkthrough_Script.pdf`** - Walkthrough script guide (PDF) for recording the presentation.
* **`pagespeed_mobile_screenshot.png`** - *[Place your PageSpeed Insights Mobile score screenshot here before submitting]*

---

## 🛠️ Verification & Interactive Guide

You can run the landing page entirely on your local machine.

### Testing GTM dataLayer Event Tracking:
1. Open **`index.html`** in any modern web browser (Google Chrome is recommended).
2. Open **Developer Tools** (Press `F12` or right-click and select `Inspect`) and select the **Console** tab.
3. Interact with the page elements to see GTM custom events push payloads log in real-time:
   * **3-Step Appointment Form**: Fill out Step 1 (specialty + location) and click Next; fill out Step 2 (patient name + phone + date) and click Next; review Step 3 and click Confirm Booking.
   * **Call Buttons**: Click the call button in the header.
   * **WhatsApp Chat**: Click the floating WhatsApp icon in the bottom-right corner.
   * **Clinic Locations**: Click "View Clinic" on any card in the clinics grid.
   * **Gated Patient Guide**: Click the "Download Free Guide" button, enter details in the modal, and submit.
   * **Scroll Depth**: Scroll down the page past the blog article to watch scroll depth events log at 25%, 50%, 75%, and 90%.

---

## Task 03: Integration Design Answers

Below is a summary of the integration architecture design to connect the landing page with HubSpot CRM and Karix WhatsApp API.

### 1. End-to-End Flow & Tech Choice
We use a custom Node.js middleware server (e.g., hosted on Cloudflare Workers) alongside a PostgreSQL/Supabase database for queueing backups.
* **Flow:** The landing page form sends the lead to the Node.js server. The server immediately saves the lead to the database queue and replies with a success message to the browser within 150ms. A background worker then cleans the phone formatting, queries HubSpot's Search API to deduplicate by phone number, patches/posts the contact card, and fires the WhatsApp message via Karix.
* **Why Direct API?** HubSpot native forms only deduplicate by email, which would cause duplicate contacts since we only collect phone numbers. Zapier and Make were rejected because implementing conditional lookups across phone/mobile fields is clunky, they add extra third-party latency, cost more at scale, and route private health information (PII) through additional systems.

### 2. Failure Points and Fallbacks
* **Downstream Failures (HubSpot or Karix down):** Because we save leads to our database queue first, we don't lose data. The background worker retries failed API calls automatically using exponential backoff.
* **Upstream Server Outage (Single Biggest Failure Point):** If the middleware server goes down, the client-side AJAX fails.
  * **Fallback:** We store failed submissions in the browser's `localStorage` and retry. If errors continue, JavaScript dynamically replaces the form button with a direct **Click-to-Call** button and a pre-filled **direct WhatsApp link** (`wa.me`) so the lead can still reach us with a single click.

### 3. WhatsApp 2-Minute SLA Monitoring
* **SLA Risks:** Unfunded prepaid balance, Meta template flags, queue bottlenecks, or users typing invalid numbers.
* **Monitoring:**
  1. We measure delivery latency as the time delta between lead creation and Karix's delivery receipt webhook. If the 95th percentile ($P_{95}$) delay exceeds 90 seconds, it triggers a Slack/PagerDuty alarm.
  2. A cron script checks the Karix Wallet API twice daily, triggering Slack warnings if the balance drops below ₹1,000 and email alerts if it goes below ₹250.
  3. Delivery failures on invalid numbers are moved to a Dead-Letter Queue (DLQ) to notify the sales desk immediately.
