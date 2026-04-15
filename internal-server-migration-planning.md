# Internal Server Migration Planning

> **Status:** Early planning — no migration has begun. This document collects thoughts, ideas, and open questions to inform future migration work. It will grow over time as details are clarified with IT and as individual worker migrations are planned.

---

## General Migration Notes

### Server Overview
- Server has been identified by IT but full specs are not yet confirmed
- Server will run Node.js (based on preliminary IT discussion)
- Server will be LAN-accessible internally and will have a public-facing endpoint that serves only final rendered display output to external display hardware
- The server will be able to reach external resources (traffic cameras, NOAA APIs, etc.) as needed
- IT will manage the server and applications; the developer will submit code changes through a defined process for IT to package and deploy

### Code Submission Process
> **Placeholder — process not yet defined.** A workflow for submitting code changes to IT for packaging and deployment needs to be established. This section should be updated once that process is documented.

### Display Hardware
- Display hardware and loading mechanism are not changing
- Displays will still load full-screen iframe URLs as they do today
- The only change will be the base URL (e.g., internal IP or internal hostname instead of a Cloudflare subdomain)
- All existing URL parameters should be preserved — only the base URL changes

### Migration Strategy
- Migrate one worker at a time: deploy, test, confirm, then move to the next
- The First Due display (see below) will be built as a net-new internal project first, before any existing workers are migrated — this gives the team a chance to establish the internal build/deploy workflow before touching production workers
- Calendar data export process is being streamlined first and is not blocked by this migration

### General Dependency Reduction Goals
- Eliminate all Google APIs (Sheets, Drive, Slides) — move all data files and folders to internal server storage
- Eliminate `images.weserv.nl` — no longer used and should not be reintroduced; image resizing handled via HTML/CSS
- Eliminate Cloudflare Workers and associated free-tier constraints (CPU limits, request limits, etc.)
- External dependencies that are acceptable to keep: NOAA APIs (river, weather, radar), traffic camera feeds, USGS river camera feeds, NWS weather alerts
- Workers that currently use only internally-produced data (department news, probationary firefighter, daily message, slide timing) should have zero external dependencies after migration
- The PowerPoint/slides presentation file should be stored on the internal server rather than hosted externally, making it easier to update and eliminating mixed hosting between internal and external sources

---

## Workers

---

### `department-news-display`
**Cloudflare repo:** `wehnerb/department-news-display`

#### Current State
- Serves department news cards as a full-screen display
- Acts as the design language reference baseline for all other workers
- No Google API or external file dependencies in current form

#### Migration Notes
- Straightforward migration — no external dependencies to eliminate
- Node.js HTTP server replaces the Cloudflare Worker handler

#### Dependencies to Eliminate
- None beyond Cloudflare itself

#### Open Questions
- None at this time

---

### `calendar-display`
**Cloudflare repo:** `wehnerb/calendar-display`

#### Current State
- Displays shift calendar and weather alerts
- Fetches ICS data from Nextcloud (`fileshare.fargond.gov`) via WebDAV
- Fetches weather alerts from NOAA NWS API
- Uses Windows timezone name mapping for ICS/Exchange compatibility

#### Migration Notes
- Nextcloud ICS feed could remain as-is (already internal infrastructure) or be replaced with a direct server-side read if the calendar file is accessible on the internal server's LAN
- NWS weather alerts are an acceptable external dependency — keep as-is
- Evaluate whether Nextcloud WebDAV fetch can be simplified when running on the internal LAN

#### Dependencies to Eliminate
- Cloudflare Worker infrastructure

#### Open Questions
- Will the internal server have direct LAN access to the Nextcloud instance, or will it still fetch via WebDAV over the network?
- Is Nextcloud the long-term home for the ICS feed, or will it move as part of the broader calendar data export streamlining effort?

---

### `weather-display`
**Cloudflare repo:** `wehnerb/weather-display`

#### Current State
- Displays current conditions, forecast, weather alerts, and NEXRAD radar
- Uses NOAA NWS API for forecast and alerts
- Uses NOAA NEXRAD / nowCOAST for radar imagery (client-side fetch required — Cloudflare datacenter IPs are blocked by NOAA)
- Leaflet.js loaded via cdnjs.cloudflare.com for map rendering

#### Migration Notes
- NOAA radar timestamp fetch was moved client-side due to Cloudflare IP blocks; on the internal server, server-side fetches to NOAA may work without issue — worth testing and potentially reverting to server-side if it simplifies the architecture
- Leaflet.js CDN dependency could be eliminated by bundling or self-hosting the library on the internal server
- NWS API and radar feeds remain acceptable external dependencies

#### Dependencies to Eliminate
- Cloudflare Worker infrastructure
- External CDN for Leaflet.js (bundle or self-host instead)

#### Open Questions
- Will the internal server's public IP be blocked by NOAA the same way Cloudflare datacenter IPs are? If not, radar timestamp fetch can move back server-side
- Should Leaflet.js be bundled into the worker or self-hosted on the internal server?

---

### `river-level-display`
**Cloudflare repo:** `wehnerb/river-level-display`

#### Current State
- Displays river gauge chart and river camera image
- Fetches gauge data from NOAA NWPS API
- Fetches river camera still image from USGS HIVIS / NIMS API
- NOAA sentinel value `-9999` is filtered before charting

#### Migration Notes
- All data sources are external (NOAA, USGS) and remain acceptable dependencies — no internal files to migrate
- Straightforward migration otherwise

#### Dependencies to Eliminate
- Cloudflare Worker infrastructure

#### Open Questions
- None at this time

---

### `daily-message-display`
**Cloudflare repo:** `wehnerb/daily-message-display`

#### Current State
- Displays a daily message fetched from a Google Sheet
- Uses Google Sheets API with a service account

#### Migration Notes
- Google Sheets dependency should be eliminated — message data should be stored in a file or simple data store on the internal server
- Need to define a replacement workflow for how daily messages get authored and updated (currently done via Google Sheets)
- Georgia serif font is kept for message body text — no change needed

#### Dependencies to Eliminate
- Google Sheets API
- Google service account credentials

#### Open Questions
- What replaces Google Sheets as the data entry point for daily messages? Options include: a simple text/JSON file edited directly, a lightweight internal web form, or a shared internal document
- Who authors the daily messages and what level of technical access do they have?

---

### `probationary-firefighter-display`
**Cloudflare repo:** `wehnerb/probationary-firefighter-display`

#### Current State
- Displays probationary firefighter information including photos
- Uses Google Sheets and Google Drive for data and image hosting
- Image sizing handled via HTML/CSS (no external image proxy in use)

#### Migration Notes
- Google Sheets and Drive should be replaced with internal server storage for both data and images
- Images stored and served directly from the internal server — larger image sizes are acceptable without the Cloudflare free-tier CPU constraints
- Image resizing continues to be handled via HTML/CSS, not server-side processing

#### Dependencies to Eliminate
- Google Sheets API
- Google Drive (images and data files)
- Google service account credentials

#### Open Questions
- What is the process for adding/updating probationary firefighter records? Currently done via Google Sheets — needs a replacement workflow
- Where will images be stored on the internal server and who will have access to upload them?

---

### `slide-timing-proxy`
**Cloudflare repo:** `wehnerb/slide-timing-proxy`

#### Current State
- Proxies slide timing data for a slideshow display
- Currently depends on Google Slides API and an externally hosted presentation file

#### Migration Notes
- The presentation file should be moved to internal server storage — this is a priority change for this worker
- Google Slides API dependency should be eliminated
- Having the presentation file internal makes it easier to update and removes the mixed hosting situation (some files internal, some on Google)
- IT display mechanism for PowerPoint/presentation files has not been fully clarified yet — architecture cannot be fully committed until that is confirmed

#### Dependencies to Eliminate
- Google Slides API
- Externally hosted presentation file (move to internal server)
- Google service account credentials

#### Open Questions
- What mechanism does IT use to display PowerPoint/presentation files? This needs to be confirmed before the replacement architecture can be designed
- Will the presentation be a PowerPoint file, a different format, or something else entirely on the internal server?

---

### `station-image-proxy`
**Cloudflare repo:** `wehnerb/station-image-proxy`

#### Current State
- Proxies station images with resizing/caching
- Previously used `images.weserv.nl`; that dependency has been removed
- Security fixes have been applied on main branch

#### Migration Notes
- Images should be stored directly on the internal server and served from there — no proxy needed
- The proxy worker itself may become unnecessary once images are hosted internally
- Evaluate whether this worker still needs to exist at all post-migration or whether images are just served as static files

#### Dependencies to Eliminate
- Cloudflare Worker infrastructure
- Potentially the proxy pattern entirely, if images are served as static files from the internal server

#### Open Questions
- Is the proxy pattern still needed on the internal server, or will images simply be static files served directly?

---

## New Internal Projects

---

### First Due Display
**Cloudflare repo:** N/A — this project will never touch Cloudflare

#### Overview
- A net-new display worker to show First Due data (turnout times and related department data)
- Will be built entirely on the internal server from the start
- IT prefers all First Due data remain internal; only the final rendered display page is served externally to display hardware

#### Migration Notes
- N/A — this is a greenfield internal project

#### Dependencies
- First Due API (internal access only — no public API calls)
- All data stays internal; no external dependencies anticipated

#### Open Questions
- API access confirmation with First Due account manager is still pending
- What specific data points from First Due should be displayed? (Turnout times confirmed; others TBD)
- What does the display layout look like? No design work has started

---

*Last updated: April 14, 2026*
*This document is a living file — update it as plans are refined, IT details are confirmed, and migration work progresses.*
