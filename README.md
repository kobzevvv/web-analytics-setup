# Skillset.ae Web Analytics Setup

Public documentation of the analytics infrastructure for [skillset.ae](https://skillset.ae).

## Services Overview

| Service | Purpose | Account |
|---------|---------|---------|
| **Google Tag Manager** | Tag management, event routing | Account: `skillset` (ID: `6339269098`) |
| **Google Analytics 4** | Web analytics, user behavior tracking | Account ID: `384586011` |
| **Yandex.Metrika** | Web analytics (RU audience), session replays | Counter ID: `106836145` |

## Architecture

```
Browser (skillset.ae)
  │
  ├── GTM Container loaded via <script> snippet
  │     │
  │     ├── GA4 tag  ──→  Google Analytics 4
  │     │   (page views, user_id, SPA navigation)
  │     │
  │     └── Yandex.Metrika tag  ──→  Yandex.Metrika
  │         (page views, webvisor, click maps)
  │
  └── Frontend pushes events to dataLayer
        dataLayer.push({ event: 'virtual_pageview', page_path: '/...', user_id: '...' })
```

## GTM Containers

We use **two separate GTM containers** — one for the clients-facing app, one for the candidates-facing app. Both are under the same GTM account `skillset`.

### 1. skillset.ae clients

| Property | Value |
|----------|-------|
| Container ID | `GTM-TWM9H2Z6` |
| GA4 Measurement ID | `G-PS0RD5V3FV` |
| GA4 Property ID | `524676123` |
| Current Live Version | 6 |
| JSON Export | [`gtm/clients/GTM-TWM9H2Z6-v6.json`](gtm/clients/GTM-TWM9H2Z6-v6.json) |

### 2. skillset candidates

| Property | Value |
|----------|-------|
| Container ID | `GTM-PBJM68LL` |
| GA4 Measurement ID | `G-HJ6J6NVZ79` |
| GA4 Property ID | `524698498` |
| Current Live Version | 5 |
| JSON Export | [`gtm/candidates/GTM-PBJM68LL-v5.json`](gtm/candidates/GTM-PBJM68LL-v5.json) |

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

### Yandex.Metrika

- **Type:** Custom HTML
- **Counter ID:** `106836145` (shared between both containers)
- **Fires on:** All Pages (built-in trigger)
- **Features enabled:** WebVisor, ClickMap, eCommerce via dataLayer, accurate bounce tracking, link tracking

## Triggers

| Name | Type | Condition | Used by |
|------|------|-----------|---------|
| `CE - virtual_pageview` | Custom Event | `event` equals `virtual_pageview` | GA4 tag |
| All Pages | Built-in (Page View) | Every page load | Yandex.Metrika |

## Variables

### User-Defined Variables

| Name | Type | Data Layer Key / Value | Description |
|------|------|----------------------|-------------|
| `CONST - GA4 Measurement ID` | Constant | `G-PS0RD5V3FV` (clients) / `G-HJ6J6NVZ79` (candidates) | GA4 Measurement ID, extracted to a variable for easy management |
| `DLV - page_path` | Data Layer Variable | `page_path` | Current SPA route path, pushed by frontend on navigation |
| `DLV - user_id` | Data Layer Variable | `user_id` | SHA-256 hashed user email, pushed by frontend after authentication |

### Built-In Variables (enabled)

| Name | Type |
|------|------|
| Page URL | URL |
| Page Hostname | URL |
| Page Path | URL |
| Referrer | HTTP Referrer |
| Event | Custom Event |

## GA4 Configuration

- **Enhanced Measurement:** Disabled on both GA4 properties (to avoid duplicate page view tracking with SPA setup)
- **Data Streams:**
  - Clients: Stream ID `13605027359`
  - Candidates: Stream ID `13605059800`

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
- [Yandex.Metrika — Counter 106836145](https://metrica.yandex.com/dashboard?id=106836145)
