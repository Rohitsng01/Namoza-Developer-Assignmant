# OrthoNow HubSpot & WhatsApp Integration Design

This document outlines how I would set up the backend integration to connect our landing page with HubSpot CRM and the Karix WhatsApp API.

---

## 1. System Architecture & Tech Stack

I recommend using a custom **Node.js/Express server (or Cloudflare Workers)** as a webhook middleware, combined with a simple database queue (like Supabase or PostgreSQL) for backups.

### How the Data Flows:
1. **Form Submission:** The user submits the form. JavaScript fires the Google Tag Manager (GTM) event for Google Ads conversion tracking, and sends the lead details (Name, Phone, Clinic) to our backend via an AJAX POST request.
2. **Database Backup (Outbox Pattern):** The backend server immediately saves the lead to our database. It returns a `200 OK` response to the user's browser right away (within 100-200ms) so they see the success message instantly, without waiting for the CRM or WhatsApp APIs to load.
3. **HubSpot Search & Update:** In the background, a worker picks up the lead, formats the phone number, checks HubSpot for an existing contact, and either updates the existing contact or creates a new one.
4. **WhatsApp Dispatch:** Once HubSpot is updated, the worker calls the Karix API to send the confirmation WhatsApp message.

### Why a Direct API Call over Zapier, Make, or HubSpot Native Forms?
* **Why not HubSpot Native Embed or Forms API?** HubSpot only deduplicates contacts by email. Because our landing page collects *only* name and phone, using HubSpot's native forms would create duplicate contact cards every single time a user submits or resubmits the form.
* **Why not Zapier or Make?** 
  * Running a "Search-before-write" flow (searching HubSpot by phone number, formatting it, updating or creating, and adding timeline history notes) is very clunky to build in Zapier/Make. You end up having to write custom JavaScript code within their steps anyway.
  * Zapier and Make add another third-party dependency, which increases network latency (bad for our 2-minute SLA) and exposes patient data (PII) to more compliance risks.
  * High execution costs at scale. A custom Node.js middleware on Cloudflare Workers is virtually free and runs in milliseconds.

---

## 2. The HubSpot Phone Deduplication Trap (Search-Before-Write)

Since HubSpot doesn't natively deduplicate contacts by phone number, our backend uses a custom lookup flow before inserting data:

1. **Clean the Phone Number:** Standardize the phone number into E.164 format (e.g. adding `+91` and stripping spaces/dashes).
2. **Search HubSpot:** Call HubSpot’s Search API (`/crm/v3/objects/contacts/search`) to search both `phone` and `mobilephone` properties for the match:
   ```json
   {
     "filterGroups": [
       {
         "filters": [
           {
             "propertyName": "phone",
             "operator": "EQ",
             "value": "+919876543210"
           }
         ]
       },
       {
         "filters": [
           {
             "propertyName": "mobilephone",
             "operator": "EQ",
             "value": "+919876543210"
           }
         ]
       }
     ]
   }
   ```
3. **Conditional Update/Create:**
   * **If Contact Exists:** Use a `PATCH` request to update the existing contact's preference and set lead status to "New Enquiry". 
     * *Name Conflicts:* If the name is different (e.g., a family member using the same phone number), the backend updates the name to the latest one and adds a note in the HubSpot contact history: `"Name updated from [Old Name] via landing page resubmission."`
   * **If Contact is New:** Use a `POST` request to create a new contact with all the details (firstname, phone, clinic preference, lead status = "New Enquiry", source = "Google Ads - Consultation Landing Page").

---

## 3. Failure Points & Fallback Plans

### 1. Downstream API Outages (HubSpot or Karix is down)
* **The Problem:** If HubSpot or Karix experiences downtime, leads might get lost.
* **The Fallback:** Because we save the raw lead details to our database *before* sending them to HubSpot/Karix, the data is safe. The backend queue worker will automatically retry failed API calls using an exponential backoff retry system (retrying at 1, 5, 15, and 30-minute intervals). If a major outage occurs, the sales team can download a CSV of the leads from the database and call them manually.

### 2. Upstream Middleware Failure (The server itself crashes)
* **The Problem:** This is the single biggest failure point. If our backend server goes down or fails to respond, the form submission fails on the browser, and the lead is lost before reaching the database.
* **The Fallback:**
  * **Client-Side Storage:** If the AJAX call to our server fails, JavaScript catches the error and saves the lead details in the browser's `localStorage` as a temporary backup, attempting a background retry if the user stays on the page.
  * **Dynamic UI CTA Fallover:** If the network failure persists, the frontend JavaScript dynamically changes the form CTA button into a direct **Click-to-Call** button and a direct **WhatsApp Link CTA** (via `wa.me`) passing the lead's details as pre-filled text. This ensures the user can still connect with a single click even if our entire server stack is down.

---

## 4. WhatsApp 2-Minute SLA & Monitoring

### What could break the SLA?
* **Account Balance:** Our prepaid Karix account runs out of money.
* **Meta Template Violations:** Meta flags or rejects a template message update, halting sends.
* **Queue Bottlenecks:** A sudden traffic spike backing up the server queue.
* **Incorrect Formatting:** Users entering invalid numbers or landlines that cannot receive WhatsApp messages.

### How to Monitor and Prevent Delays:
1. **Latency Loop Tracking:** We record a timestamp when the lead is created ($T_{created}$). When Karix hits our delivery receipt webhook (confirming the user got the message), we calculate the delta ($T_{delivered} - T_{created}$). If the 95th percentile ($P_{95}$) of this delta exceeds 90 seconds, we trigger a Slack/PagerDuty alert.
2. **Wallet Balance Alerts:** A daily cron script checks the Karix Wallet Balance API. If the balance falls below ₹1,000, it sends a Slack warning. If it falls below ₹250, it triggers a critical email alert to top it up immediately.
3. **Invalid Number Alerts:** If Karix returns a delivery failure code (e.g. invalid number), the worker moves the lead to a Dead-Letter Queue (DLQ) and sends a Slack alert so the sales desk knows to dial the number manually right away.
