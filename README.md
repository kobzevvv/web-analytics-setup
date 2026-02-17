# Skillset.ae Web Analytics Setup

Public documentation of the analytics infrastructure for [skillset.ae](https://skillset.ae).

## Services Overview

| Service | Purpose | Account |
|---------|---------|---------|
| **Google Tag Manager** | Tag management, event routing | Account: `skillset` (ID: `6339269098`) |
| **Google Analytics 4** | Web analytics, user behavior tracking | Account ID: `384586011` |
| **Google Ads** | Paid search campaigns | Account: `vladimir@skillset.ae` |
| **Yandex.Metrika** | Web analytics (RU audience), session replays | Counter ID: `106836145` |

## Architecture

```
Browser (skillset.ae)
  │
  ├── GTM Container loaded via <script> snippet
  │     │
  │     ├── GA4 - Page View (SPA) tag  ──→  Google Analytics 4
  │     │   (page views, user_id, SPA navigation)
  │     │
  │     ├── GA4 - Click tag  ──→  Google Analytics 4
  │     │   (click events with click_text parameter)
  │     │
  │     ├── GA4 - Lead Form Submit  ──→  Google Analytics 4
  │     │   (email captured / form submission)
  │     │
  │     ├── GA4 - Demo Booked  ──→  Google Analytics 4
  │     │   (calendar booking completed)
  │     │
  │     ├── GA4 - Pricing Page View  ──→  Google Analytics 4
  │     │   (visited pricing page)
  │     │
  │     ├── GA4 - Engaged Visit  ──→  Google Analytics 4
  │     │   (scroll >50% OR timer >30s)
  │     │
  │     ├── Google Ads Remarketing  ──→  Google Ads
  │     │   (audience building for retargeting)
  │     │
  │     └── Yandex.Metrika tag  ──→  Yandex.Metrika
  │         (page views, webvisor, click maps)
  │
  └── Frontend pushes events to dataLayer
        dataLayer.push({ event: 'virtual_pageview', page_path: '/...', user_id: '...' })
        dataLayer.push({ event: 'lead_form_submit', form_type: 'email_capture' })
        dataLayer.push({ event: 'demo_booked', booking_source: 'calendly' })
```

## Conversion Goals

### Goal Hierarchy (for Google Ads Smart Bidding)

| # | Event Name | Goal Type | Google Ads Action | When to Fire | Value |
|---|-----------|-----------|-------------------|-------------|-------|
| 1 | `demo_booked` | **Primary** | Optimize for | Calendar booking completed | $50 |
| 2 | `lead_form_submit` | **Primary** | Optimize for | Email captured (any form) | $20 |
| 3 | `pricing_page_view` | **Secondary** | Observe only | User visits /pricing page | $5 |
| 4 | `engaged_visit` | **Secondary** | Observe only | Scroll >50% OR time >30s | $1 |

> **Why this hierarchy?** Smart Bidding needs ~30 conversions/month per goal to learn effectively. Start with both `demo_booked` + `lead_form_submit` as Primary. Once you get 30+ demo bookings/month, switch `lead_form_submit` to Secondary and keep only `demo_booked` as Primary.

> **Why Secondary goals?** They appear in Google Ads reports but Smart Bidding does NOT optimize for them. This prevents the algorithm from chasing easy goals (engaged visits) instead of real conversions (demo bookings).

### Goal Implementation Details

#### 1. `demo_booked` — Calendar Booking Completed

**When:** User completes a booking via the calendar widget (Calendly / Cal.com / Webflow).

**dataLayer push (frontend):**
```javascript
dataLayer.push({
  event: 'demo_booked',
  booking_source: 'calendly',  // or 'cal_com', 'webflow'
  page_path: window.location.pathname
});
```

**Implementation options:**
- **Calendly:** Listen for `calendly.event_scheduled` postMessage event
- **Cal.com:** Listen for `Cal.event.bookingSuccessful` event
- **Thank-you page redirect:** Fire on page_path containing `/thank-you` or `/booking-confirmed`

**GTM Setup:**
- Trigger: Custom Event `demo_booked`
- Tag: GA4 Event with event name `demo_booked`, parameter `booking_source`

#### 2. `lead_form_submit` — Email Captured

**When:** User submits any form with their email (newsletter, contact, waitlist, gated content).

**dataLayer push (frontend):**
```javascript
dataLayer.push({
  event: 'lead_form_submit',
  form_type: 'email_capture',  // or 'contact_form', 'waitlist'
  page_path: window.location.pathname
});
```

**GTM Setup:**
- Trigger: Custom Event `lead_form_submit`
- Tag: GA4 Event with event name `generate_lead` (GA4 recommended event name), parameters: `form_type`, `page_path`

#### 3. `pricing_page_view` — Pricing Page Visited

**When:** User navigates to the pricing page.

**Option A — SPA (React app):** The frontend already pushes `virtual_pageview` with `page_path`. Create a GA4 event that fires when `page_path` contains `/pricing`.

**Option B — Landing pages (Webflow):** Use GTM Page View trigger where Page Path contains `/pricing`.

**GTM Setup:**
- Trigger: Custom Event `virtual_pageview` where `DLV - page_path` contains `pricing`
  (OR Page View trigger where Page Path contains `pricing` for non-SPA pages)
- Tag: GA4 Event with event name `pricing_page_view`

#### 4. `engaged_visit` — Active User Signal

**When:** User shows engagement — scrolled >50% of the page OR spent >30 seconds on site.

**GTM Setup (no frontend code needed):**
- Trigger A: Scroll Depth trigger — Vertical, 50% threshold
- Trigger B: Timer trigger — Interval 30000ms, Limit 1, fires on All Pages
- Combine with Trigger Group OR fire separate events and deduplicate in GA4
- Tag: GA4 Event with event name `engaged_visit`, parameter `engagement_type` = `scroll_50` or `timer_30s`

### Yandex.Metrika Goals

Mirror the same goals in Yandex.Metrika for consistency:

| Metrika Goal | Type | Condition |
|-------------|------|-----------|
| Demo Booked | JavaScript event | `ym(106836145, 'reachGoal', 'demo_booked')` |
| Lead Form Submit | JavaScript event | `ym(106836145, 'reachGoal', 'lead_form_submit')` |
| Pricing Page View | URL contains | `/pricing` |
| Engaged Visit | JavaScript event | `ym(106836145, 'reachGoal', 'engaged_visit')` |

**Frontend implementation for Metrika goals:**
```javascript
// After demo booking
ym(106836145, 'reachGoal', 'demo_booked');

// After form submission
ym(106836145, 'reachGoal', 'lead_form_submit');

// For engaged visit (handled via GTM Custom HTML tag)
```

---

## GA4 Property & Data Streams

We use **one GA4 property** with **two data streams** — one for the clients-facing app, one for the candidates-facing app.

| Property | Value |
|----------|-------|
| GA4 Property ID | `524676123` |
| GA4 Account ID | `384586011` |

### Data Streams

| Stream | Measurement ID | Stream ID |
|--------|---------------|-----------|
| skillset.ae clients | `G-PS0RD5V3FV` | `13605027359` |
| skillset candidates | `G-HJ6J6NVZ79` | `13605059800` |

## GTM Containers

We use **two separate GTM containers** — one per app, each sending data to its own GA4 data stream within the same property. Both are under the same GTM account `skillset`.

### 1. skillset.ae clients

| Property | Value |
|----------|-------|
| Container ID | `GTM-TWM9H2Z6` |
| GA4 Measurement ID | `G-PS0RD5V3FV` (clients stream) |
| Current Live Version | 7 |
| JSON Export | [`gtm/clients/GTM-TWM9H2Z6-v7.json`](gtm/clients/GTM-TWM9H2Z6-v7.json) |

### 2. skillset candidates

| Property | Value |
|----------|-------|
| Container ID | `GTM-PBJM68LL` |
| GA4 Measurement ID | `G-HJ6J6NVZ79` (candidates stream) |
| Current Live Version | 6 |
| JSON Export | [`gtm/candidates/GTM-PBJM68LL-v6.json`](gtm/candidates/GTM-PBJM68LL-v6.json) |

## Tags

Both containers have identical tag structure:

### GA4 - Page View (SPA)

- **Type:** Google Tag (`googtag`)
- **Tag ID:** `{{CONST - GA4 Measurement ID}}` (stored as a Constant variable)
- **Fires on:** Custom Event trigger `CE - virtual_pageview`
- **Configuration parameters:**

| Parameter | Value | Description |
|-----------|-------|-------------|
| `page_location` | `{{Page URL}}` | Full URL of the current page |
| `page_path` | `{{DLV - page_path}}` | SPA route path from dataLayer |
| `user_id` | `{{DLV - user_id}}` | SHA-256 hashed email for cross-device tracking |

> **Why SPA tracking?** Both apps are single-page applications (React). Standard page view tracking doesn't work because the browser doesn't perform full page loads on navigation. Instead, the frontend pushes a `virtual_pageview` event to dataLayer on each route change.

### GA4 - Click

- **Type:** Google Analytics: GA4 Event
- **Measurement ID:** `{{CONST - GA4 Measurement ID}}`
- **Event Name:** `click`
- **Fires on:** `All Clicks` trigger (Click - All Elements)
- **Event parameters:**

| Parameter | Value | Description |
|-----------|-------|-------------|
| `click_text` | `{{Click Text}}` | Text content of the clicked element (built-in variable) |

> **Why track clicks?** Capturing `click_text` on every click provides valuable insight into user interactions — which buttons, links, and elements users engage with most. This data appears as a custom event parameter in GA4 and can be used in explorations and reports.

### Yandex.Metrika

- **Type:** Custom HTML
- **Counter ID:** `106836145` (shared between both containers)
- **Fires on:** All Pages (built-in trigger)
- **Features enabled:** WebVisor, ClickMap, eCommerce via dataLayer, accurate bounce tracking, link tracking

## Triggers

| Name | Type | Condition | Used by |
|------|------|-----------|---------|
| `CE - virtual_pageview` | Custom Event | `event` equals `virtual_pageview` | GA4 - Page View (SPA) |
| `CE - demo_booked` | Custom Event | `event` equals `demo_booked` | GA4 - Demo Booked |
| `CE - lead_form_submit` | Custom Event | `event` equals `lead_form_submit` | GA4 - Lead Form Submit |
| `CE - pricing_pageview` | Custom Event | `virtual_pageview` where page_path contains `pricing` | GA4 - Pricing Page View |
| `Scroll Depth 50%` | Scroll Depth | Vertical, 50% threshold | GA4 - Engaged Visit |
| `Timer 30s` | Timer | 30000ms interval, limit 1 | GA4 - Engaged Visit |
| `All Clicks` | Click - All Elements | Every click on any element | GA4 - Click |
| All Pages | Built-in (Page View) | Every page load | Yandex.Metrika |

## Variables

### User-Defined Variables

| Name | Type | Data Layer Key / Value | Description |
|------|------|----------------------|-------------|
| `CONST - GA4 Measurement ID` | Constant | `G-PS0RD5V3FV` (clients) / `G-HJ6J6NVZ79` (candidates) | GA4 Measurement ID, extracted to a variable for easy management |
| `DLV - page_path` | Data Layer Variable | `page_path` | Current SPA route path, pushed by frontend on navigation |
| `DLV - user_id` | Data Layer Variable | `user_id` | SHA-256 hashed user email, pushed by frontend after authentication |
| `DLV - form_type` | Data Layer Variable | `form_type` | Type of form submitted (email_capture, contact_form, waitlist) |
| `DLV - booking_source` | Data Layer Variable | `booking_source` | Calendar widget source (calendly, cal_com, webflow) |

### Built-In Variables (enabled)

| Name | Type |
|------|------|
| Page URL | URL |
| Page Hostname | URL |
| Page Path | URL |
| Referrer | HTTP Referrer |
| Event | Custom Event |
| Click Text | Auto-Event Variable |
| Scroll Depth Threshold | Scroll |

## GA4 Configuration

- **Enhanced Measurement:** Disabled on both data streams (to avoid duplicate page view tracking with SPA setup)
- **BigQuery Export:** Enabled — daily export of event data and user data to GCP project `skillset-analytics-487510` (US region)
- **Google Ads Linking:** Link GA4 property to Google Ads account for conversion import and audience sharing

### GA4 Conversion Events

Mark the following events as conversions in GA4 Admin > Events > Mark as conversion:

| Event Name | Marked as Conversion |
|-----------|---------------------|
| `generate_lead` | Yes |
| `demo_booked` | Yes |
| `pricing_page_view` | Yes |
| `engaged_visit` | No (tracked but not a conversion) |

## Google Ads Conversion Setup

### Step-by-Step Setup

1. **Link Google Ads ↔ GA4**
   - In GA4: Admin > Product Links > Google Ads > Link
   - Select your Google Ads account
   - Enable auto-tagging (adds `gclid` to URLs)

2. **Import GA4 conversions into Google Ads**
   - In Google Ads: Tools > Measurement > Conversions > New conversion action > Import > Google Analytics 4
   - Import: `generate_lead`, `demo_booked`, `pricing_page_view`
   - Set conversion action categories:
     - `demo_booked` → **Primary**, category: "Submit lead form", value: $50
     - `generate_lead` → **Primary**, category: "Submit lead form", value: $20
     - `pricing_page_view` → **Secondary**, category: "Page view", value: $5

3. **Add Google Ads Remarketing tag in GTM**
   - Create a new tag: Google Ads Remarketing
   - Conversion ID: (get from Google Ads account)
   - Trigger: All Pages
   - This builds remarketing audiences automatically

4. **Enable Enhanced Conversions** (optional, recommended)
   - In Google Ads: Tools > Measurement > Conversions > Settings > Enhanced conversions
   - Provides first-party data matching for better attribution
   - Send hashed email on conversion events

### Conversion Window Settings (in Google Ads)

| Setting | Value | Reason |
|---------|-------|--------|
| Click-through window | 30 days | B2B sales cycle is long |
| View-through window | 1 day | Conservative for search |
| Attribution model | Data-driven | Let Google optimize |
| Count | One per click | Lead gen (not ecommerce) |

## Google Ads Campaign Tracking

### UTM Parameters

Google Ads auto-tagging (`gclid`) handles most tracking, but for Yandex.Metrika and other tools, add UTM parameters to Final URLs:

```
?utm_source=google&utm_medium=cpc&utm_campaign={campaignid}&utm_content={adgroupid}&utm_term={keyword}
```

Google Ads ValueTrack parameters (auto-populated):
- `{campaignid}` — campaign ID
- `{adgroupid}` — ad group ID
- `{keyword}` — keyword that triggered the ad
- `{matchtype}` — match type (e, p, b)
- `{device}` — device (c, m, t)

### Campaign Spreadsheet

All Google Ads campaigns are documented in Google Sheets:
- **Spreadsheet ID:** `1bpbZhL1wh0huFHeEP9XjcPjZy4KGfrWAABveAVmrL5s`
- **Tabs:** bulk import to gads, Value Propositions, Keywords & Bids, Search Campaigns, Competitor Campaigns, Competitor Analysis, Competitor Ads & Keywords, Landing Pages, Negative Keywords

## user_id Tracking

### Overview

We track authenticated users across devices using GA4's `user_id` feature. The frontend pushes a SHA-256 hash of the user's email to dataLayer — **never the raw email** (Google policy compliance).

### Flow

```
1. User logs in on frontend
2. Frontend hashes email:  SHA-256("user@example.com") → "a3f5b8c1d2e4..."
3. Frontend pushes to dataLayer:
     dataLayer.push({ user_id: 'a3f5b8c1d2e4...' })
4. GTM captures via DLV - user_id variable
5. GA4 tag sends user_id as configuration parameter
6. GA4 uses it for cross-device user identification
```

### Frontend Implementation

```javascript
// Hash the email using Web Crypto API
async function hashEmail(email) {
  const normalized = email.trim().toLowerCase();
  const encoder = new TextEncoder();
  const data = encoder.encode(normalized);
  const hashBuffer = await crypto.subtle.digest('SHA-256', data);
  const hashArray = Array.from(new Uint8Array(hashBuffer));
  return hashArray.map(b => b.toString(16).padStart(2, '0')).join('');
}

// After user authentication
const hashedEmail = await hashEmail(user.email);
dataLayer.push({ user_id: hashedEmail });
```

### dataLayer Contract

The frontend must push the following events/data to `dataLayer`:

```javascript
// On every SPA route change (required for page view tracking)
dataLayer.push({
  event: 'virtual_pageview',
  page_path: '/current/route/path'
});

// After user authentication (required for user_id tracking)
dataLayer.push({
  user_id: 'sha256_hash_of_email'
});

// After form submission (required for lead tracking)
dataLayer.push({
  event: 'lead_form_submit',
  form_type: 'email_capture'  // or 'contact_form', 'waitlist'
});

// After demo booking (required for conversion tracking)
dataLayer.push({
  event: 'demo_booked',
  booking_source: 'calendly'  // or 'cal_com', 'webflow'
});
```

## Yandex.Metrika Configuration

| Setting | Value |
|---------|-------|
| Counter ID | `106836145` |
| WebVisor | Enabled |
| ClickMap | Enabled |
| eCommerce | Enabled (via `dataLayer`) |
| Accurate Bounce | Enabled |
| Track Links | Enabled |
| SSR mode | Enabled |

## BigQuery Export

GA4 events are exported daily to BigQuery for advanced analysis and data warehousing.

| Setting | Value |
|---------|-------|
| GCP Project | `skillset-analytics-487510` (SkillSet Analytics) |
| GCP Project Number | `287078842321` |
| Data Location | United States (us) |
| Event Data Export | Daily |
| User Data Export | Daily |
| Streams Exported | Both (clients + candidates) |

### BigQuery Tables

After export is enabled, GA4 creates tables in the format:
- `skillset-analytics-487510.analytics_<property_id>.events_YYYYMMDD` — daily event tables
- `skillset-analytics-487510.analytics_<property_id>.pseudonymous_users_YYYYMMDD` — daily user tables

> **Note:** Data starts flowing ~24 hours after linking. The first export includes only data from the day of linking onward.

## Setup Checklist

### Done
- [x] GTM containers installed (clients + candidates)
- [x] GA4 property with two data streams
- [x] SPA page view tracking via virtual_pageview
- [x] Click tracking with click_text parameter
- [x] user_id tracking (SHA-256 hashed email)
- [x] Yandex.Metrika with WebVisor
- [x] BigQuery daily export
- [x] Google Ads campaign planning (spreadsheet)

### To Do
- [ ] Link GA4 ↔ Google Ads account
- [ ] Add GTM tags: demo_booked, lead_form_submit, pricing_page_view, engaged_visit
- [ ] Add GTM triggers: CE - demo_booked, CE - lead_form_submit, Scroll 50%, Timer 30s
- [ ] Add GTM variables: DLV - form_type, DLV - booking_source
- [ ] Mark events as conversions in GA4
- [ ] Import GA4 conversions into Google Ads (set Primary/Secondary)
- [ ] Add Google Ads Remarketing tag in GTM
- [ ] Add Yandex.Metrika goals
- [ ] Frontend: implement dataLayer pushes for lead_form_submit and demo_booked
- [ ] Build landing pages: /ai-recruiting, /resume-screening, /ai-sourcing, /automation, /compare, /dubai, /agencies, /small-business, /demo
- [ ] Enable Enhanced Conversions in Google Ads (after first conversions)
- [ ] Publish new GTM container version (v8)
- [ ] Export GTM v8 JSON to this repo

## How to Import/Export GTM Containers

### Export (backup)

1. Open [Google Tag Manager](https://tagmanager.google.com/)
2. Select the container
3. Go to **Admin** > **Export Container**
4. Choose the latest version
5. Click **Export**
6. Save the JSON file to `gtm/clients/` or `gtm/candidates/`

### Import (restore)

1. Open [Google Tag Manager](https://tagmanager.google.com/)
2. Select the target container
3. Go to **Admin** > **Import Container**
4. Upload the JSON file
5. Choose **Overwrite** or **Merge** as needed
6. Review changes and publish

## Links

- [GTM — skillset.ae clients](https://tagmanager.google.com/#/container/accounts/6339269098/containers/243571296/workspaces)
- [GTM — skillset candidates](https://tagmanager.google.com/#/container/accounts/6339269098/containers/243653940/workspaces)
- [GA4 — Admin](https://analytics.google.com/analytics/web/#/a384586011p524676123/admin)
- [GA4 — BigQuery Links](https://analytics.google.com/analytics/web/#/a384586011p524676123/admin/integrations/big-query)
- [BigQuery — GCP Console](https://console.cloud.google.com/bigquery?project=skillset-analytics-487510)
- [Yandex.Metrika — Counter 106836145](https://metrica.yandex.com/dashboard?id=106836145)
- [Google Ads Campaign Spreadsheet](https://docs.google.com/spreadsheets/d/1bpbZhL1wh0huFHeEP9XjcPjZy4KGfrWAABveAVmrL5s/)
