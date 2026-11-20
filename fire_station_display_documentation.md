# Fire Station Display Board — System Documentation

*Technical Reference Guide for Department Staff, Administrators, and IT Support*

Last Updated: November 19, 2026

Maintained by: Brandon Wehner

# Table of Contents

- [1. System Overview](#1-system-overview)
  - [1.1 Purpose](#11-purpose)
  - [1.2 How the System Works](#12-how-the-system-works)
  - [1.3 Technology Stack](#13-technology-stack)
- [2. Account & Access Guide](#2-account--access-guide)
  - [2.1 Accounts and Where to Find Them](#21-accounts-and-where-to-find-them)
  - [2.2 GitHub Repositories](#22-github-repositories)
  - [2.3 GitHub Secrets Reference](#23-github-secrets-reference)
  - [2.4 Transferring Ownership to a New Administrator](#24-transferring-ownership-to-a-new-administrator)
- [3. Project: Image Resizing Proxy](#3-project-image-resizing-proxy)
- [4. Project: Slide Timing Proxy](#4-project-slide-timing-proxy)
- [5. Project: River Level Display](#5-project-river-level-display)
- [6. Project: Daily Message Display](#6-project-daily-message-display)
- [7. Project: Calendar Display](#7-project-calendar-display)
- [8. Project: Probationary Firefighter Display](#8-project-probationary-firefighter-display)
- [9. Project: Department News Display](#9-project-department-news-display)
- [10. Project: Weather Display](#10-project-weather-display)
- [11. Project: Newsletter Slides Display](#11-project-newsletter-slides-display)
- [12. Shared Utilities](#12-shared-utilities)
- [13. Deployment & Maintenance Workflow](#13-deployment--maintenance-workflow)
- [14. IT Support Reference](#14-it-support-reference)
- [15. Planned Enhancements](#15-planned-enhancements)

# 1. System Overview

## 1.1 Purpose

The Fire Station Display Board system is a set of digital display screens deployed throughout the fire station that provide firefighters with real-time and regularly updated operational information. Content displayed includes traffic camera feeds, river gauge levels and hydrographs, Google Slides presentations with department announcements, rotating safety messages, a station calendar with weather data, a probationary firefighter spotlight, department and station-specific news, a full weather display with animated radar, and newsletter slide images that start and expire automatically on their own dates.

The system was designed with two core constraints in mind:

- Display screens can only be configured with a single, static endpoint URL — no runtime parameters or user interaction is possible at the screen level.
- All infrastructure and tooling used must be free or within free tier service limits.

## 1.2 How the System Works

Because the display screens cannot resize images, authenticate to APIs, or make intelligent decisions at runtime, a layer of Cloudflare Workers sits between the screens and the data sources. Each Worker is a lightweight serverless function that runs at Cloudflare’s edge network and handles the logic that the display screen cannot.

The general data flow for every display is:

|**Step**            |**Description**                                            |
|--------------------|-----------------------------------------------------------|
|Display Screen      |Loads a single, fixed Cloudflare Worker URL                |
|Cloudflare Worker   |Processes the request, fetches data from external sources  |
|External Data Source|Returns camera images, river gauge data, slide counts, etc.|
|Response to Screen  |Worker returns the processed content ready for display     |

Each Worker is deployed independently and has its own URL. A display screen is configured with a Worker URL and from that point on requires no further maintenance unless the underlying data source or configuration changes.

## 1.3 Technology Stack

|**Technology**    |**Purpose**                                                   |**Cost**                   |
|------------------|--------------------------------------------------------------|---------------------------|
|Cloudflare Workers|Serverless edge functions that process requests               |Free tier (100,000 req/day)|
|GitHub            |Source code storage and version control                       |Free                       |
|GitHub Actions    |Automatic deployment pipeline to Cloudflare                   |Free                       |
|Google Sheets API |Content storage for daily messages, news, and firefighter bios|Free (service account auth)|
|Google Drive API  |Image storage for daily messages, firefighter photos, and newsletter slides       |Free (service account auth)|
|Google Slides API |Slide count queries for the slide timing proxy                |Free (service account auth)|
|Nextcloud WebDAV  |ICS calendar file delivery from Nextcloud to Worker           |Free (city infrastructure) |
|NWS Public API    |Weather conditions, forecast, and alerts (no API key required)|Free / Public              |
|AirNow API        |Air Quality Index data for the weather display                |Free (API key required)    |
|RainViewer API    |Animated radar tile data for the weather display              |Free / Public              |
|NOAA NWPS API     |River gauge data (stage, flood thresholds, forecasts)         |Free / Public              |
|ND DOT / USGS     |Traffic camera and river camera image sources                 |Free / Public              |
|UptimeRobot       |Uptime monitoring for all 9 Workers with email alerts         |Free tier                  |

# 2. Account & Access Guide

## 2.1 Accounts and Where to Find Them

|**Account** |**URL / Login**                                                             |**What It Controls**                                                       |
|------------|----------------------------------------------------------------------------|---------------------------------------------------------------------------|
|Cloudflare  |dash.cloudflare.com – bwehner                                               |Worker deployment, subdomain (bwehner), secrets storage                    |
|GitHub      |github.com/wehnerb                                                          |All source code repositories and deployment workflows                      |
|Google Cloud|console.cloud.google.com — [bwehner@fargond.gov](mailto:bwehner@fargond.gov)|Google Slides, Sheets, and Drive API access and service account credentials|
|AirNow      |docs.airnowapi.org — [bwehner@fargond.gov](mailto:bwehner@fargond.gov)      |AirNow API key for weather display AQI data                                |
|UptimeRobot |uptimerobot.com — [bwehner@fargond.gov](mailto:bwehner@fargond.gov)         |Uptime monitoring for all 9 Workers, email alerts on downtime              |

## 2.2 GitHub Repositories

|**Repository**                  |**Worker URL**                                      |**Purpose**                                                 |
|--------------------------------|----------------------------------------------------|------------------------------------------------------------|
|station-image-proxy             |station-image-proxy.bwehner.workers.dev             |Image resizing proxy for traffic camera and river feeds     |
|slide-timing-proxy              |slide-timing-proxy.bwehner.workers.dev              |Dynamic Google Slides per-slide timing                      |
|river-level-display             |river-level-display.bwehner.workers.dev             |River gauge hydrograph display (NOAA NWPS data)             |
|daily-message-display           |daily-message-display.bwehner.workers.dev           |Daily safety message and image display                      |
|calendar-display                |calendar-display.bwehner.workers.dev                |Station calendar display from exported ICS file with weather|
|probationary-firefighter-display|probationary-firefighter-display.bwehner.workers.dev|Rotating probationary firefighter spotlight display         |
|department-news-display         |department-news-display.bwehner.workers.dev         |Department-wide and station-specific news card display      |
|weather-display                 |weather-display.bwehner.workers.dev                 |Full weather display with radar, conditions, and alerts     |
|newsletter-slides-display       |newsletter-slides-display.bwehner.workers.dev       |Expiring newsletter slide images from a Drive folder, with automatic start and end dates|
|ffd-station-display-utils       |(shared library — no Worker URL)                    |Shared design tokens, helpers, and utilities for all Workers|

## 2.3 GitHub Secrets Reference

Each GitHub repository stores secrets that are injected into the Cloudflare Worker at deployment time. These are never exposed in code and must be set per repository under Settings → Secrets and variables → Actions. Which secrets GitHub pushes to Cloudflare automatically is decided by each repository’s .github/workflows/deploy.yml — see the Important box below the table.

|**Secret Name**             |**Repository**                                                                  |**Description**                                                                                                                                                                                           |
|----------------------------|--------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
|CLOUDFLARE_API_TOKEN        |All                                                                             |Cloudflare API token with Workers edit permissions                                                                                                                                                        |
|CLOUDFLARE_ACCOUNT_ID       |All                                                                             |Cloudflare account ID (found on any zone page in dashboard)                                                                                                                                               |
|GOOGLE_SERVICE_ACCOUNT_EMAIL|slide-timing-proxy, daily-message-display, newsletter-slides-display (GitHub)|Service account email from Google Cloud JSON key file. For daily-message-display and newsletter-slides-display, deploy.yml pushes it from the GitHub secret to Cloudflare on every deploy. Also required for probationary-firefighter-display and department-news-display, where it is set manually in the Cloudflare dashboard.|
|GOOGLE_PRIVATE_KEY          |slide-timing-proxy, daily-message-display, newsletter-slides-display (GitHub)|Private key from Google Cloud JSON key file (include \n characters). For daily-message-display and newsletter-slides-display, deploy.yml pushes it from the GitHub secret to Cloudflare on every deploy. Also required for probationary-firefighter-display and department-news-display, where it is set manually in the Cloudflare dashboard.|
|GOOGLE_SHEET_ID             |daily-message-display, department-news-display, probationary-firefighter-display|The ID of the relevant Google Sheet. Found in the Sheet URL between /d/ and /edit. For daily-message-display it is a GitHub secret that deploy.yml pushes to Cloudflare on every deploy; for the other two Workers it is set manually in the Cloudflare dashboard.|
|GOOGLE_DRIVE_FOLDER_ID      |daily-message-display, probationary-firefighter-display, newsletter-slides-display|The ID of the Google Drive folder containing the images; each Worker uses its own folder. Found in the folder URL after /folders/. For daily-message-display and newsletter-slides-display it is a GitHub secret that deploy.yml pushes to Cloudflare on every deploy; for probationary-firefighter-display it is set manually in the Cloudflare dashboard.|
|PRESENTATION_ID             |slide-timing-proxy only                                                         |Google Slides presentation ID — the alphanumeric string between /d/ and /edit in the presentation URL.                                                                                                    |
|PUBLISHED_ID                |slide-timing-proxy only                                                         |Google Slides published embed ID — the long string between /d/e/ and /pubembed in the File → Share → Publish to web embed URL.                                                                            |
|NEXTCLOUD_URL               |calendar-display only                                                           |Full WebDAV URL to the ICS file on Nextcloud. Format: <https://fileshare.fargond.gov/remote.php/dav/files/USERNAME/FFD%20Calendar%20Export/FFD%20Calendar%20Calendar.ics>                                 |
|NEXTCLOUD_USERNAME          |calendar-display only                                                           |Nextcloud login username (shown when creating an app password — not the display name).                                                                                                                    |
|NEXTCLOUD_PASSWORD          |calendar-display only                                                           |Nextcloud app password. Generate at: Nextcloud → Settings → Security → Devices & sessions → Create new app password.                                                                                      |
|AIRNOW_API_KEY              |weather-display only                                                            |AirNow API key for AQI data. Set in Cloudflare dashboard (not GitHub Actions). Register at docs.airnowapi.org. Free tier is sufficient.                                                                   |
|WORKER_REPO_DISPATCH_TOKEN  |ffd-station-display-utils only                                                  |Fine-grained GitHub personal access token with actions: write and contents: write on every Worker repository. Used by notify-workers.yml to start each Worker’s utils sync (Section 12.4). Create it under GitHub → Settings → Developer settings → Personal access tokens → Fine-grained tokens. Fine-grained tokens expire: note the expiration date, and when it passes (or a new Worker repository is added) update the token’s repository access or create a replacement and update the secret.|

|**Where to Find These Secret Values**                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
|CLOUDFLARE_API_TOKEN: Log in to dash.cloudflare.com, go to My Profile → API Tokens → Create Token. Use the “Edit Cloudflare Workers” template. Copy the generated token — it is only shown once.                                                                                                                                                                                                                                                                                                                                                                   |
|CLOUDFLARE_ACCOUNT_ID: Log in to dash.cloudflare.com and go to Workers & Pages → Overview. The Account ID is displayed in the right sidebar.                                                                                                                                                                                                                                                                                                                                                                                                                       |
|GOOGLE_SERVICE_ACCOUNT_EMAIL and GOOGLE_PRIVATE_KEY: Log in to console.cloud.google.com and open the slide-timing project. Go to IAM & Admin → Service Accounts → slide-timing-worker → Keys → Add Key → Create new key (JSON). Download the JSON file. The client_email field is your GOOGLE_SERVICE_ACCOUNT_EMAIL and the private_key field is your GOOGLE_PRIVATE_KEY. Store the JSON file securely and delete it after copying the values into the GitHub repository secrets (and into the Cloudflare dashboard for any Worker whose deploy.yml does not push them).                                                                                       |
|Note: river-level-display does not require any Google credentials. daily-message-display, probationary-firefighter-display, department-news-display, and newsletter-slides-display all use the same service account as slide-timing-proxy. For daily-message-display and newsletter-slides-display, set the values as GitHub repository secrets (deploy.yml pushes them to Cloudflare on every deploy). For probationary-firefighter-display and department-news-display, set the values in the Cloudflare dashboard (Workers & Pages → [Worker Name] → Settings → Variables and Secrets), for both the production Worker and its -staging counterpart. calendar-display uses Nextcloud WebDAV and does not require Google credentials. weather-display requires AIRNOW_API_KEY set in the Cloudflare dashboard.|

|**Important — Where Worker Secrets Are Stored**|
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
|How a Worker receives a secret depends on that repository’s .github/workflows/deploy.yml. A secret named in the secrets: list of deploy.yml is a GitHub repository secret that GitHub Actions pushes to the Worker in Cloudflare on every deploy. For those secrets GitHub is the source of truth: to change a value, update the GitHub secret and then re-run the latest deploy for both the staging and main branches. A change made only in the Cloudflare dashboard is overwritten by the next deploy. A secret that deploy.yml does not name (for example AIRNOW_API_KEY) must be set manually in the Cloudflare dashboard under Workers & Pages → [Worker Name] → Settings → Variables and Secrets, separately for the production Worker and its -staging counterpart. In practice: daily-message-display and newsletter-slides-display push their GOOGLE_ secrets from GitHub on every deploy, while the secrets of probationary-firefighter-display, department-news-display, weather-display (AIRNOW_API_KEY), and calendar-display (the Nextcloud secrets) are set manually in the Cloudflare dashboard, as listed in the table above. When in doubt, open the Worker’s deploy.yml and look at its secrets: list.|

## 2.4 Transferring Ownership to a New Administrator

If a new person takes over maintenance of this system, the following steps must be completed to transfer full control. Each step must be completed in order.

### Step 1 — GitHub

1. Create a GitHub account for the new administrator if they do not have one.
1. Go to github.com/wehnerb and transfer ownership of each repository to the new account under Settings → Danger Zone → Transfer ownership. Alternatively, add the new person as an owner of the organization.
1. The new administrator must re-add all GitHub Secrets listed in Section 2.3 to each repository under their account. GitHub cannot display a stored secret, so the values must be collected again from their original sources (see “Where to Find These Secret Values”).
1. The WORKER_REPO_DISPATCH_TOKEN fine-grained token belongs to the account that created it, so the new administrator must create a new one with actions: write and contents: write on all 9 Worker repositories and save it as a secret in ffd-station-display-utils.

### Step 2 — Cloudflare

1. Log in to dash.cloudflare.com and go to Manage Account → Members.
1. Add the new administrator’s email and assign the Administrator role.
1. Create a new API token for the new administrator using the Edit Cloudflare Workers template and provide it to them for their GitHub secrets.
1. For daily-message-display and newsletter-slides-display, the Google secrets come from GitHub: after the new administrator has set the GitHub secrets, re-run the latest deploy for the staging and main branches of each repository.
1. In the Cloudflare dashboard, set every secret that a Worker’s deploy.yml does not push (under Workers & Pages → [Worker Name] → Settings → Variables and Secrets, for each Worker and its -staging counterpart). These are: GOOGLE_SERVICE_ACCOUNT_EMAIL and GOOGLE_PRIVATE_KEY for probationary-firefighter-display and department-news-display; GOOGLE_SHEET_ID and GOOGLE_DRIVE_FOLDER_ID for probationary-firefighter-display; GOOGLE_SHEET_ID for department-news-display; AIRNOW_API_KEY for weather-display; NEXTCLOUD_URL, NEXTCLOUD_USERNAME, NEXTCLOUD_PASSWORD for calendar-display.

### Step 3 — Google Cloud

1. Go to console.cloud.google.com and open the slide-timing project.
1. Go to IAM & Admin → IAM and add the new administrator’s Google account with the Owner role.
1. The new administrator should create a new service account key (IAM & Admin → Service Accounts → slide-timing-worker → Keys → Add Key). The old key should then be deleted.
1. Update GOOGLE_SERVICE_ACCOUNT_EMAIL and GOOGLE_PRIVATE_KEY wherever each Worker stores them: in the GitHub repository secrets (then re-run the deploy for staging and main) for Workers whose deploy.yml pushes them, and in the Cloudflare dashboard for any that do not.
1. Make sure every Google Sheet and Drive folder used by the Workers (including the newsletter slides folder) is owned by, or fully accessible to, the new administrator and stays shared with the service account email.

### Step 4 — AirNow

1. Go to docs.airnowapi.org and register for a new API key using the new administrator’s email.
1. Update AIRNOW_API_KEY in the Cloudflare dashboard for weather-display and weather-display-staging.

### Step 5 — UptimeRobot

1. Go to uptimerobot.com and create a new account for the new administrator.
1. Recreate all 9 monitors (see Section 14.4) under the new account.
1. Set the alert contact to the new administrator’s department email.

*Note: river-level-display has no Google Cloud dependency. calendar-display uses Nextcloud WebDAV and has no Google Cloud dependency — the new administrator only needs to update the NEXTCLOUD_URL, NEXTCLOUD_USERNAME, and NEXTCLOUD_PASSWORD secrets in the Cloudflare dashboard.*

# 3. Project: Image Resizing Proxy

## 3.1 Purpose & Problem Solved

The station display system cannot resize images natively. Without intervention, images either fail to fill a display column entirely or overflow beyond its boundaries.

The Image Proxy Worker solves this by returning a self-contained HTML page rather than a raw image. Each page contains an image element pointing at the Worker’s own /proxy endpoint, with CSS that scales the image to fit the display column using object-fit: contain. Because the browser on the display hardware handles all scaling locally, no external image processing service is required.

The /proxy endpoint streams each upstream image (ND DOT, USGS, or NOAA) through the Worker so that all image fetches originate from the same workers.dev origin as the HTML page. This avoids a Chrome Private Network Access (PNA) restriction that blocks public-origin pages from fetching subresources from domains that resolve to private IP addresses on the local network — a behavior triggered on networks with split DNS or transparent proxies.

Image freshness is preserved: the /proxy endpoint disables Cloudflare edge caching on every request and sets Cache-Control: no-store on its responses, so each refresh of the display retrieves a fresh frame from the upstream source. No image bytes are buffered, decoded, or modified by the Worker — the upstream response is streamed through unchanged.

The HTML page itself is handled differently: for a given URL it is static (a few lines of CSS and the image tags), so display browsers are allowed to keep it privately for one hour (the PAGE_CACHE_SECONDS setting). This removes the page requests that would otherwise be made every display cycle. Camera and gauge images are never cached. See Section 3.10.

## 3.2 Repository & Deployment

|**Item**         |**Value**                                                 |
|-----------------|----------------------------------------------------------|
|GitHub Repository|github.com/wehnerb/station-image-proxy                    |
|Production URL   |<https://station-image-proxy.bwehner.workers.dev/>        |
|Staging URL      |<https://station-image-proxy-staging.bwehner.workers.dev/>|
|Worker File      |src/index.js                                              |
|Secrets Required |CLOUDFLARE_API_TOKEN, CLOUDFLARE_ACCOUNT_ID               |
|Page Cache       |Pages cached privately by the display browser for 1 hour (PAGE_CACHE_SECONDS); images never cached (Section 3.10)|

## 3.3 How It Works

When a display screen loads an image URL, the request goes to the Cloudflare Worker. The Worker reads the URL parameters to determine which image to display and what layout dimensions to use. It returns a self-contained HTML page containing an image element whose src points at the Worker’s own /proxy endpoint (e.g., /proxy?key=i29MainAve-north), plus CSS rules that scale the image to fill the slot using object-fit: contain with a transparent background.

The display browser then requests the image from the /proxy endpoint. The Worker validates the requested key against the MAPPING table, fetches the corresponding upstream URL (ND DOT, USGS, or NOAA), and streams the response body back to the browser unchanged. No image processing, decoding, or buffering occurs — the upstream bytes pass straight through, keeping CPU usage well under the 10 ms Worker free-tier limit.

This proxy-through pattern exists to avoid Chrome Private Network Access (PNA) blocking. When the HTML page is loaded from a public workers.dev origin and DNS on the local network resolves upstream domains (such as [www.dot.nd.gov](http://www.dot.nd.gov)) to private IP addresses, Chrome blocks the cross-origin image fetches as a security measure. Routing all image fetches through the Worker means the browser only ever contacts the workers.dev origin, sidestepping the issue entirely.

Every /proxy response carries Cache-Control: no-store (and Pragma: no-cache), and Cloudflare edge caching is explicitly disabled on /proxy upstream fetches (cf.cacheTtl: 0), so every time a page is shown the browser fetches a fresh frame from the upstream source. The HTML page is different: when every image key in the URL is valid, it is sent with Cache-Control: private, max-age=3600 so the display browser can reuse it for an hour. Pages with an invalid key, and all error pages, are sent with no-store (Section 3.10).

## 3.4 URL Parameters

|**Parameter**|**Default**|**Options**         |**Description**                                                                              |
|-------------|-----------|--------------------|---------------------------------------------------------------------------------------------|
|img          |(required) |Any key from MAPPING|The image key name to display. Combine two keys with + for stacking.                         |
|layout       |split      |wide, split, tri    |Column width: 1-column (wide), 2-column (split), 3-column (tri). Default is split (2-column).|

The Worker also exposes an internal /proxy endpoint used by the rendered HTML page to fetch upstream image bytes. It accepts a single key parameter (e.g., /proxy?key=i29MainAve-north) that must match an entry in the MAPPING table. This endpoint is not intended for direct configuration on a display screen — displays must always use the ?img= endpoint, which renders the HTML page that in turn references /proxy. The /proxy endpoint only accepts keys defined in MAPPING; arbitrary URLs cannot be fetched through it, preventing the Worker from being used as an open proxy.

## 3.5 Layout Dimensions

|**Layout Key**|**Width (px)**|**Height (px)**|**Use Case**                    |
|--------------|--------------|---------------|--------------------------------|
|wide          |1735          |720            |Full-width single column display|
|split         |852           |720            |Two-column display (default)    |
|tri           |558           |720            |Three-column display            |
|full          |1920          |1075           |Full-screen display             |

## 3.6 Example URLs

Single camera image in a two-column layout:

`https://station-image-proxy.bwehner.workers.dev/?img=i29MainAve-north`

Two stacked camera images in a two-column layout:

`https://station-image-proxy.bwehner.workers.dev/?img=i29MainAve-north+i29MainAve-south&layout=split`

River level gauge in a single-column layout:

`https://station-image-proxy.bwehner.workers.dev/?img=riverlevel-redriver&layout=wide`

## 3.7 Image Stacking

Two images can be stacked vertically within a single column by separating two image keys with a + in the img parameter. The Worker automatically splits the column height equally between the two images with a 10-pixel gap between them. A maximum of 2 images can be stacked. Attempting to stack 3 or more will return an error image.

## 3.8 Adding New Images

All images are defined in the MAPPING object near the top of index.js. To add a new camera or data image:

1. Open the staging branch of the station-image-proxy repository in GitHub.
1. Edit src/index.js and locate the MAPPING object.
1. Add a new line inside the MAPPING object following the existing format:

`"your-key-name": "https://full-url-to-the-source-image.jpg",`

1. Key names should be lowercase with hyphens — for example i29MainAve-north for an I-29 camera at Main Ave facing north.
1. Commit the change to staging and test the new key using the staging Worker URL before merging to main.

## 3.9 Current Image Library

The following images are currently configured in the MAPPING object. See Section 3.8 for instructions on adding new images.

### NOAA River Gauges

|**Key Name**       |**Readable Name**           |**Source URL**                                             |
|-------------------|----------------------------|-----------------------------------------------------------|
|riverlevel-redriver|Red River Level Gauge (NOAA)|<https://water.noaa.gov/resources/hydrographs/fgon8_hg.png>|

### USGS River Images

|**Key Name**  |**Readable Name**               |**Source URL**                                                                                                                      |
|--------------|--------------------------------|------------------------------------------------------------------------------------------------------------------------------------|
|river-redriver|Red River at Fargo (USGS Camera)|<https://usgs-nims-images.s3.amazonaws.com/overlay/ND_Red_River_of_the_North_at_Fargo/ND_Red_River_of_the_North_at_Fargo_newest.jpg>|

### ND DOT Cameras — I-94

|**Key Name**         |**Readable Name**                    |**Source URL**                                                                    |
|---------------------|-------------------------------------|----------------------------------------------------------------------------------|
|i94VeteransBlvd-south|I-94 at Veterans Blvd (Facing South) |<https://www.dot.nd.gov/travel-info/cameras/I94@347.601Fargo9thStEWBSouth.jpg>    |
|i94VeteransBlvd-east |I-94 at Veterans Blvd (Facing East)  |<https://www.dot.nd.gov/travel-info/cameras/I94RP347.565Fargo9thStEEBEast.jpg>    |
|i9445thStS-south     |I-94 at 45th St S (Facing South)     |<https://www.dot.nd.gov/travel-info/cameras/I94RP348.602Fargo45thStSouth.jpg>     |
|i9445thStS-east      |I-94 at 45th St S (Facing East)      |<https://www.dot.nd.gov/travel-info/cameras/I94RP348.602Fargo45thStEast.jpg>      |
|i9445thStS-west      |I-94 at 45th St S (Facing West)      |<https://www.dot.nd.gov/travel-info/cameras/I94RP348.602Fargo45thStWest.jpg>      |
|i9442ndStS-east      |I-94 at 42nd St S (Facing East)      |<https://www.dot.nd.gov/travel-info/cameras/I94RP349.145Fargo42ndStSWBEast.jpg>   |
|i9442ndStS-west      |I-94 at 42nd St S (Facing West)      |<https://www.dot.nd.gov/travel-info/cameras/I94RP349.145Fargo42ndStSWBWest.jpg>   |
|i29i94-north         |I-29/I-94 Interchange (Facing North) |<https://www.dot.nd.gov/travel-info/cameras/fargotrilevelnorth.jpg>               |
|i29i94-south         |I-29/I-94 Interchange (Facing South) |<https://www.dot.nd.gov/travel-info/cameras/fargotrilevelsouth.jpg>               |
|i94i29-east          |I-94/I-29 Interchange (Facing East)  |<https://www.dot.nd.gov/travel-info/cameras/fargotrileveleast.jpg>                |
|i94i29-west          |I-94/I-29 Interchange (Facing West)  |<https://www.dot.nd.gov/travel-info/cameras/fargotrilevelwest.jpg>                |
|i9425thStS-east      |I-94 at 25th St S (Facing East)      |<https://www.dot.nd.gov/travel-info/cameras/I94RP350.611Fargo25thStEast.jpg>      |
|i9425thStS-west      |I-94 at 25th St S (Facing West)      |<https://www.dot.nd.gov/travel-info/cameras/I94RP350.603Fargo25thStWest.jpg>      |
|i94UniversityDrS-east|I-94 at University Dr S (Facing East)|<https://www.dot.nd.gov/travel-info/cameras/I94RP351.617FargoUniversityDrEast.jpg>|
|i94UniversityDrS-west|I-94 at University Dr S (Facing West)|<https://www.dot.nd.gov/travel-info/cameras/I94RP351.617FargoUniversityDrWest.jpg>|
|i94RedRiver-east     |I-94 at Red River (Facing East)      |<https://www.dot.nd.gov/travel-info/cameras/I94RP352FargoRedRiverEast.jpg>        |
|i94RedRiver-west     |I-94 at Red River (Facing West)      |<https://www.dot.nd.gov/travel-info/cameras/I94RP352FargoRedRiverWest.jpg>        |

### ND DOT Cameras — I-29

|**Key Name**     |**Readable Name**                |**Source URL**                                                                        |
|-----------------|---------------------------------|--------------------------------------------------------------------------------------|
|i2970thAveS-north|I-29 at 70th Ave S (Facing North)|<https://www.dot.nd.gov/travel-info/cameras/I29RP58.765FargoSouthof64thAveSNorth.jpg> |
|i2970thAveS-south|I-29 at 70th Ave S (Facing South)|<https://www.dot.nd.gov/travel-info/cameras/I29RP58.765FargoSouthof64thAveSSouth.jpg> |
|i2952ndAveS-north|I-29 at 52nd Ave S (Facing North)|<https://www.dot.nd.gov/travel-info/cameras/I29RP60.293Fargo52ndAveSSBNorth.jpg>      |
|i2952ndAveS-south|I-29 at 52nd Ave S (Facing South)|<https://www.dot.nd.gov/travel-info/cameras/I29RP60.293Fargo52ndAveSSBSouth.jpg>      |
|i2952ndAveS-east |I-29 at 52nd Ave S (Facing East) |<https://www.dot.nd.gov/travel-info/cameras/I29RP60.293Fargo52ndAveSSBEast.jpg>       |
|i2952ndAveS-west |I-29 at 52nd Ave S (Facing West) |<https://www.dot.nd.gov/travel-info/cameras/I29RP60.293Fargo52ndAveSSBWest.jpg>       |
|i2940thAveS-north|I-29 at 40th Ave S (Facing North)|<https://www.dot.nd.gov/travel-info/cameras/I29RP61.408Fargo40thAveNorth.jpg>         |
|i2940thAveS-south|I-29 at 40th Ave S (Facing South)|<https://www.dot.nd.gov/travel-info/cameras/I29RP61.408Fargo40thAveSouth.jpg>         |
|i2932ndAveS-north|I-29 at 32nd Ave S (Facing North)|<https://www.dot.nd.gov/travel-info/cameras/I29RP62.627Fargo32ndAveSEastRampNorth.jpg>|
|i2932ndAveS-south|I-29 at 32nd Ave S (Facing South)|<https://www.dot.nd.gov/travel-info/cameras/I29RP62.627Fargo32ndAveSEastRampSouth.jpg>|
|i2932ndAveS-west |I-29 at 32nd Ave S (Facing West) |<https://www.dot.nd.gov/travel-info/cameras/I29RP62.627Fargo32ndAveSEastRampWest.jpg> |
|i2913thAveS-north|I-29 at 13th Ave S (Facing North)|<https://www.dot.nd.gov/travel-info/cameras/I29RP64.725FargoNorthof9thAveNorth.jpg>   |
|i2913thAveS-south|I-29 at 13th Ave S (Facing South)|<https://www.dot.nd.gov/travel-info/cameras/I29RP64.135Fargo13thAveSSouth.jpg>        |
|i29MainAve-north |I-29 at Main Ave (Facing North)  |<https://www.dot.nd.gov/travel-info/cameras/I29RP65.272FargoMainAveSBNorth.jpg>       |
|i29MainAve-south |I-29 at Main Ave (Facing South)  |<https://www.dot.nd.gov/travel-info/cameras/I29RP65.272FargoMainAveSBSouth.jpg>       |
|i297thAveN-south |I-29 at 7th Ave N (Facing South) |<https://www.dot.nd.gov/travel-info/cameras/I29RP65.741Fargo7thAveNSouth.jpg>         |
|i2919thAveN-north|I-29 at 19th Ave N (Facing North)|<https://www.dot.nd.gov/travel-info/cameras/I29RP66.894Fargo19thAveNNorth.jpg>        |
|i2919thAveN-south|I-29 at 19th Ave N (Facing South)|<https://www.dot.nd.gov/travel-info/cameras/I29RP66.894Fargo19thAveNSouth.jpg>        |
|i2919thAveN-east |I-29 at 19th Ave N (Facing East) |<https://www.dot.nd.gov/travel-info/cameras/I29RP67.241Fargo19thAveNEastRampEast.jpg> |
|i2919thAveN-west |I-29 at 19th Ave N (Facing West) |<https://www.dot.nd.gov/travel-info/cameras/I29RP67.241Fargo19thAveNEastRampWest.jpg> |

## 3.10 Request Volume, Page Caching & Display Timing (November 2026)

All Workers share Cloudflare’s Free-plan limit of 100,000 requests per day (Section 14.5). The Image Proxy is the largest consumer, because every image screen makes several requests each time it is shown.

**What one image screen costs.** A standard image screen has 2 columns. Each column is an Image Proxy URL with 2 stacked images, so each time the screen is shown the display makes 2 page requests (one per column) plus 4 image requests, which is 6 Worker requests. Stations have between 1 and 4 such screens. With page caching, the 2 page requests are normally answered from the display browser’s cache, leaving the 4 image requests.

**Page caching (added November 2026).**

- The setting is PAGE_CACHE_SECONDS near the top of src/index.js. It is 3600 (1 hour).
- Only the HTML page is cached, privately, in each display’s browser. The cache is keyed by the full URL, so every column URL is cached separately; nothing is shared between screens or stations.
- Never cached: the /proxy images (always no-store), pages that contain an invalid image key (the “INVALID IMAGE KEY” card), error pages, and /healthz. A two-image page is cached only if both keys are valid.
- A change to the page itself (layout, CSS, markup) reaches the displays up to one hour late, and a rollback has the same delay, because the displays’ caches cannot be cleared remotely. To make a page change visible sooner, lower PAGE_CACHE_SECONDS first, wait for the old value to expire, then make the change.
- A browser reload (F5 or Ctrl+R) and the DevTools “Disable cache” option bypass the cache, so they will always show a real page request.

**How to verify caching on a desktop browser.** Open DevTools → Network and select the **All** filter (not **Img**, which only lists the /proxy image request, which correctly shows no-store). Click the first request, named like ?img=…, and check Response Headers for Cache-Control: private, max-age=3600. Open the URL again in a new tab (or press Enter in the address bar) and the page request should show disk cache or memory cache as its source, while the image request is still a real network request. This was verified on production on November 19, 2026.

**Measured effect.** On November 18, 2026 at about 07:15, four of the eight stations were pointed at the staging Worker running the cache code, while the other four stayed on production without it. Combined Image Proxy traffic fell from about 650 to about 535 invocations per 15 minutes (about 18% lower), and the split between the two Workers matched the expected 4 versus 6 requests per screen. Camera images continued to update normally at the test stations. On November 19 all stations were moved back to the production Worker and staging was merged into main. With all eight stations cached, the Image Proxy is estimated at about 445 invocations per 15 minutes (about 43,000 per day) compared with about 650 (about 62,000 per day) without caching at the current display timing. This is an estimate; the first full day of measurements after the change has not yet been recorded.

**Why the images are not cached.** Camera frames are often already old at the source. Informally, depending on the camera, a frame can be up to roughly 30 minutes old when the Worker fetches it, and this is normal for these cameras. The Worker must not add any delay on top of that: the goal is for each new frame to appear on the display as soon as the source publishes it. Image caching was tried earlier and rejected because new images took too long to appear. A short image cache (for example 5 minutes) would save little, because a given screen is shown only once per display cycle and a cycle can be about that long, yet it would still make images staler.

**Preferred way to reduce requests: longer screen display times.** Each image screen costs a fixed number of requests every time it is shown, so total requests are roughly proportional to 1 ÷ the length of the display cycle. On the morning of November 17, 2026 the display cycle was shortened (about 40 seconds removed per cycle on each display, adjusted display by display starting at about 10:30), and by that afternoon Image Proxy traffic had risen from about 400 to about 650 invocations per 15 minutes (about 62,000 per day). Requests from the other Workers also rose, bringing the account total to about 1,070 per 15 minutes before page caching. Lengthening the time each screen is displayed, especially the image screens, lowers requests in proportion and needs no code change. As a rule of thumb, lengthening the full cycle by 25% cuts requests by about 20%. This is the preferred first response if daily usage approaches the limit (Section 14.5).

**Options considered and not adopted.**

- Caching the proxied images (camera images in particular) in the browser or at Cloudflare’s edge: rejected, for the freshness reasons above.
- Newsletter Slides Display (Section 11): its slide images could be cached by the browser, with an estimated saving of about 4,000 requests per day. Not implemented; available if needed. Its page cannot use a long cache because the page changes at every rotation slot.

**Configuration (src/index.js).**

|**Constant**      |**Value**|**Description**                                                                                      |
|------------------|---------|-----------------------------------------------------------------------------------------------------|
|STACK_GAP         |10       |Pixel gap between vertically stacked images                                                          |
|PAGE_CACHE_SECONDS|3600     |Seconds a display browser may reuse a valid image page before asking the Worker again (Section 3.10)|

# 4. Project: Slide Timing Proxy

## 4.1 Purpose & Problem Solved

Display screens are configured with a fixed time slot for showing a Google Slides presentation — for example, 60 seconds. However, the number of slides in the presentation changes over time. If the presentation has 6 slides and each is displayed for 10 seconds, the full 60 seconds is used. If there are only 2 slides but each is still set to 10 seconds, only 20 of the 60 seconds is used before it loops.

The Slide Timing Proxy Worker solves this by dynamically calculating the correct per-slide timing at request time. Each time the display loads the Worker URL, the Worker queries the Google Slides API to count the current number of slides, divides the total allotted time equally, and redirects the display directly to the Google Slides published embed URL with the correct timing parameter already set.

## 4.2 Repository & Deployment

|**Item**         |**Value**                                                                                    |
|-----------------|---------------------------------------------------------------------------------------------|
|GitHub Repository|github.com/wehnerb/slide-timing-proxy                                                        |
|Production URL   |<https://slide-timing-proxy.bwehner.workers.dev/>                                            |
|Staging URL      |<https://slide-timing-proxy-staging.bwehner.workers.dev/>                                    |
|Worker File      |src/index.js                                                                                 |
|Secrets Required |CLOUDFLARE_API_TOKEN, CLOUDFLARE_ACCOUNT_ID, GOOGLE_SERVICE_ACCOUNT_EMAIL, GOOGLE_PRIVATE_KEY|

## 4.3 Example URL

All stations use the same Worker URL with no additional parameters:

`https://slide-timing-proxy.bwehner.workers.dev/`

## 4.4 How It Works

1. The Worker authenticates with the Google Slides API using a service account, generating a short-lived OAuth2 access token using the RSA-signed JWT method via Cloudflare’s built-in Web Crypto API.
1. The Worker checks the Workers Cache API for a previously stored slide count. If a cached value exists and the SLIDE_CACHE_VERSION matches, the cached count is used and the Google API is not called.
1. On a cache miss, the Worker calls the Slides API to retrieve the current number of slides and stores the result in the cache for SLIDE_CACHE_SECONDS (default 1 hour).
1. The per-slide delay is calculated: total seconds divided by slide count, clamped to the configured minimum.
1. The Worker issues a 302 redirect directly to the Google Slides published embed URL (pubembed format) with the calculated delayms parameter. The presentation always starts from slide 1 on every fresh load.

## 4.5 Timing Logic

|**Slide Count**|**Total Seconds**|**Per-Slide Delay**|**Notes**                                        |
|---------------|-----------------|-------------------|-------------------------------------------------|
|0              |60               |N/A                |No-content screen shown, auto-refreshes every 60s|
|1              |60               |60s                |Single slide uses full allotted time             |
|2              |60               |30s                |Normal equal division                            |
|6              |60               |10s                |Normal equal division                            |
|12             |60               |5s                 |Hits minimum cap (MIN_SECONDS = 5)               |
|API fails      |60               |60s                |Safe fallback: slideCount defaults to 1          |

## 4.6 Configuration

|**Constant**                    |**Description**                                                                                                                                                                                                                                                                   |
|--------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
|PRESENTATION_ID and PUBLISHED_ID|Stored as Cloudflare Worker secrets and injected at runtime via the env object. PRESENTATION_ID is the alphanumeric ID between /d/ and /edit in the Google Slides URL. PUBLISHED_ID is the long string between /d/e/ and /pubembed in the File → Share → Publish to web embed URL.|
|TOTAL_SECONDS                   |Total seconds the display system allocates to the slideshow slot.                                                                                                                                                                                                                 |
|MIN_SECONDS                     |Minimum seconds per slide regardless of how many slides exist.                                                                                                                                                                                                                    |
|SLIDE_CACHE_SECONDS             |How long (seconds) the slide count is cached using the Workers Cache API. Default is 3600 (1 hour).                                                                                                                                                                               |
|SLIDE_CACHE_VERSION             |Integer cache-buster. Increment by 1 to immediately invalidate the cached slide count.                                                                                                                                                                                            |

## 4.7 No-Content Screen

If the presentation has zero slides, the Worker returns a styled HTML page instead of redirecting. This page displays a dark screen with the message NO CONTENT AVAILABLE and automatically refreshes every 60 seconds.

## 4.8 Google Service Account

The Google Slides API requires a service account for authentication. The service account credentials are stored as GitHub secrets and injected into the Worker at deployment time. For slide-timing-proxy, the service account requests the presentations.readonly scope. Each worker requests the minimum OAuth2 scope for its needs: daily-message-display and probationary-firefighter-display use drive.readonly; department-news-display uses the spreadsheets scope (read and write), which is required by its automated expired-row deletion feature.

|**Security Note**                                                                                                                                                                                                                                                                                                                                    |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
|The service account private key (GOOGLE_PRIVATE_KEY) is a sensitive credential. It is stored only in GitHub Secrets and Cloudflare Worker secrets — never in the code itself. If a key is ever suspected to be compromised, it should be deleted in Google Cloud Console (IAM & Admin → Service Accounts → Keys) and a new one generated immediately.|

# 5. Project: River Level Display

## 5.1 Purpose & Problem Solved

Static NOAA river gauge images show only a current stage and a basic hydrograph image. They cannot be customized for display resolution, do not show forecast data in a readable way at station screen sizes, and do not highlight flood threshold proximity relative to current conditions.

The River Level Display Worker solves this by fetching raw gauge data from the NOAA NWPS public API and rendering a fully custom, canvas-based hydrograph HTML page server-side. The page includes:

- 120 hours of observed stage history with a gradient fill
- NWS forecast data displayed as a dashed line
- Flood threshold lines (action/minor/moderate/major) shown only when within range
- A crest marker when the river has peaked and is confirmed falling
- An adaptive X-axis that adjusts label density based on available width
- A real-time flood status badge in the header (Normal / Action / Minor / Moderate / Major)
- Auto-refresh every 15 minutes matching the NOAA data update cycle

## 5.2 Repository & Deployment

|**Item**         |**Value**                                                 |
|-----------------|----------------------------------------------------------|
|GitHub Repository|github.com/wehnerb/river-level-display                    |
|Production URL   |<https://river-level-display.bwehner.workers.dev/>        |
|Staging URL      |<https://river-level-display-staging.bwehner.workers.dev/>|
|Worker File      |src/index.js                                              |
|Secrets Required |CLOUDFLARE_API_TOKEN, CLOUDFLARE_ACCOUNT_ID               |

## 5.3 URL Parameters

|**Parameter**|**Default**|**Options**                 |**Description**                                                        |
|-------------|-----------|----------------------------|-----------------------------------------------------------------------|
|gauge        |fargo      |Any key from GAUGES registry|Which river gauge to display. See Section 5.7 for the current registry.|
|layout       |split      |wide, split, tri, full      |Column width matching display hardware.                                |

## 5.4 How It Works

1. The Worker makes two parallel requests to the NOAA NWPS public API: one for gauge metadata (name, flood thresholds) and one for observed and forecast stage data. No authentication is required.
1. The Worker processes the response: it trims observed history to the configured window, combines observed and forecast data, detects a crest if the river has peaked, and determines the current flood status.
1. All processed data is injected as a JSON literal into a self-contained HTML page. No further API calls are made by the display browser.
1. The browser renders the hydrograph on a canvas element using the injected data.
1. The page auto-refreshes every 15 minutes via a meta refresh tag.

If the NOAA API is unreachable or returns an error, the Worker returns a styled error page that automatically retries every 60 seconds.

## 5.5 Configuration

|**Constant**           |**Description**                                                                                          |
|-----------------------|---------------------------------------------------------------------------------------------------------|
|GAUGES                 |Registry mapping URL-friendly keys to NOAA gauge IDs and display names. Add new gauges here.             |
|DEFAULT_GAUGE          |The gauge used when the ?gauge= parameter is omitted.                                                    |
|OBSERVED_HOURS         |How many hours of observed history to show on the chart (default: 120).                                  |
|CACHE_SECONDS          |How long Cloudflare edge-caches the NOAA API response (default: 900 = 15 minutes).                       |
|Y_AXIS_PADDING         |Feet of padding added above and below the data range on the Y axis.                                      |
|THRESHOLD_LOOKAHEAD_FT |A flood threshold line is only shown when it falls within this many feet of the data maximum.            |
|CREST_MIN_FLANK_POINTS |Number of data points required on each side of the peak before a crest is confirmed.                     |
|CREST_MIN_PROMINENCE_FT|The peak must be at least this many feet above the average of the flanking data to be labeled as a crest.|

## 5.6 Adding New Gauges

1. Find the NOAA gauge ID at water.noaa.gov (e.g. FGON8 for Fargo, ND).
1. Edit src/index.js on the staging branch and add a new entry to the GAUGES object: `'your-key': { id: 'NWSID', name: 'Human-readable name' },`
1. Test using the staging URL with ?gauge=your-key before merging to main.

## 5.7 Current Gauge Registry

|**Key (?gauge=)**|**NOAA Gauge ID**|**Display Name**  |
|-----------------|-----------------|------------------|
|fargo            |FGON8            |Red River at Fargo|

# 6. Project: Daily Message Display

## 6.1 Purpose & Problem Solved

Station displays previously had no mechanism for showing rotating safety messages, quotes, or imagery. The Daily Message Display Worker provides a dedicated full-screen display that rotates content every 3 days — aligned to the department’s 9-day shift rotation — ensuring all three shifts see every message before it advances. Content is managed by shift officers without any technical knowledge required.

## 6.2 Repository & Deployment

|**Item**         |**Value**                                                                                                                                                             |
|-----------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------|
|GitHub Repository|github.com/wehnerb/daily-message-display                                                                                                                              |
|Production URL   |<https://daily-message-display.bwehner.workers.dev/>                                                                                                                  |
|Staging URL      |<https://daily-message-display-staging.bwehner.workers.dev/>                                                                                                          |
|Worker File      |src/index.js                                                                                                                                                          |
|Secrets Required |CLOUDFLARE_API_TOKEN, CLOUDFLARE_ACCOUNT_ID, GOOGLE_SERVICE_ACCOUNT_EMAIL, GOOGLE_PRIVATE_KEY, GOOGLE_SHEET_ID, GOOGLE_DRIVE_FOLDER_ID — all GitHub repository secrets; deploy.yml pushes the four GOOGLE_ values to Cloudflare on every deploy|

## 6.3 URL Parameters

The ?layout= parameter accepts: full (1920x1075), wide (1735x720, default), split (852x720), tri (558x720). The Daily Safety Message title label is only shown in the full layout.

## 6.4 How It Works

1. The Worker authenticates with Google using the shared service account, generating a short-lived OAuth2 access token.
1. The Worker fetches the Messages tab from Google Sheets and lists image files in the Google Drive folder in parallel.
1. Date override entries are checked first. If today’s date matches a pinned image filename prefix or a sheet row’s Date column, that entry is selected. Images take priority over text if both are pinned to the same date.
1. If no date override matches, a combined rotation pool is built by interleaving active text entries and image files evenly. The pool index is: floor(daysElapsed / ROTATION_DAYS) % poolSize, anchored to January 23, 2026 in America/Chicago time.
1. For image entries, the Worker locates the file in Google Drive and serves it via a streaming proxy route (/image/{fileId}). The display browser requests the image from the Worker, which fetches it from Drive server-side using the service account token and streams the bytes directly to the browser. No base64 encoding is performed — this keeps CPU usage low and avoids Cloudflare’s 10ms CPU limit even for large images.
1. A self-contained HTML page is returned. The meta-refresh interval is set to the exact seconds until the next 7:30 AM Central rotation.

## 6.5 Rotation Logic

Messages rotate every ROTATION_DAYS calendar days (default: 3). With 3-day blocks aligned to the 9-day shift rotation, each message is seen by all three shifts before advancing. Day boundaries use America/Chicago time via Intl.DateTimeFormat, so DST transitions are handled correctly.

## 6.6 Configuration

Key constants in src/index.js: ROTATION_DAYS (default 3), ROTATION_ANCHOR (2026-01-23), IMAGE_SOURCE (“drive” or “network”), DEFAULT_LAYOUT (“wide”).

## 6.7 Content Management

**Text messages:** Add rows to the Messages tab of the Google Sheet “Fire Station Display — Daily Messages”. Set Active to “yes”. Date and Attribution are optional.

**Images:** Drop image files into the “Fire Station Display — Daily Images” Drive folder. Images under 2 MB are recommended for best display performance — larger files will work but may cause a noticeable slight delay when the page first loads on hardware. Move files into any subfolder to deactivate.

**Date pinning:** For text, enter YYYY-MM-DD in the Date column. For images, prefix the filename: YYYY-MM-DD-filename.jpg. Images win if both sources are pinned to the same date.

## 6.8 Google Sheet Columns

Required columns: Content, Active. Optional columns: Date, Attribution. The header row is protected with an edit warning. Columns are read by header name so they can be reordered without code changes.

## 6.9 Network Share (Future Use)

The Worker includes a stubbed code path for an internal network share. To activate: set IMAGE_SOURCE = “network” in src/index.js and add secrets NETWORK_SHARE_URL, NETWORK_SHARE_USERNAME, and NETWORK_SHARE_PASSWORD to Cloudflare Worker settings.

# 7. Project: Calendar Display

## 7.1 Purpose & Problem Solved

The station display system did not previously have a way to display the department’s calendar or current weather conditions. The Calendar Display Worker fetches the FFD Calendar ICS file from Nextcloud via WebDAV and renders it as a styled HTML calendar page. For wide and full layouts, the Worker also fetches live NWS weather data — daily high/low temperatures, conditions, and wind for each day, an hourly forecast strip for today, and active/upcoming weather alert banners and badges. No technical knowledge is required to keep the calendar current.

## 7.2 Repository & Deployment

|**Item**         |**Value**                                                                                         |
|-----------------|--------------------------------------------------------------------------------------------------|
|GitHub Repository|github.com/wehnerb/calendar-display                                                               |
|Production URL   |<https://calendar-display.bwehner.workers.dev/>                                                   |
|Staging URL      |<https://calendar-display-staging.bwehner.workers.dev/>                                           |
|Worker File      |src/index.js                                                                                      |
|Secrets Required |CLOUDFLARE_API_TOKEN, CLOUDFLARE_ACCOUNT_ID, NEXTCLOUD_URL, NEXTCLOUD_USERNAME, NEXTCLOUD_PASSWORD|

## 7.3 URL Parameters

The ?layout= parameter controls which design is rendered. wide and full use the split view (today detail on the left, next days on the right) and include NWS weather data. split and tri use the strip view (compact upcoming list) with no weather data. Default is wide.

## 7.4 How It Works

1. The Worker checks the Workers Cache API for a previously rendered page matching the requested layout. If a valid cached response exists it is returned immediately.
1. On a cache miss, data is fetched. For split and tri layouts, only the ICS file is fetched. For wide and full layouts, the ICS file and all NWS endpoints (daily forecast, hourly forecast, active alerts) are fetched in parallel when SHOW_WEATHER is true (see Section 7.6).

**Note:** SHOW_WEATHER is currently set to false in src/index.js. The weather integration described in steps 2 and in Section 7.7 is fully implemented in the code but is currently disabled. It was disabled after the standalone weather-display worker was deployed. To re-enable, set SHOW_WEATHER = true and deploy.

1. The raw ICS text is fetched server-side from Nextcloud using HTTP Basic authentication with a Nextcloud app password. The display browser never contacts Nextcloud directly. Windows timezone names emitted by Exchange (e.g. “Central Standard Time”) are automatically mapped to IANA timezone identifiers. Only events that fall within the displayed days (plus a one-day buffer) are fully parsed; the rest of the 30-day export is skipped early to save CPU time (see Section 7.12).
1. Filter rules are applied, a self-contained HTML page is rendered, stored in the Workers Cache API for CACHE_SECONDS (default 15 minutes), and returned to the display.

## 7.5 Automatic Calendar Update System

The calendar is kept current by a three-component system that runs automatically at login on the designated department computer:

- **Outlook VBA macro** (ThisOutlookSession in the Outlook VBA editor): Runs automatically when Outlook opens. Exports the next 30 days of the FFD Calendar public folder to U:\Fire\BWehner\FFD Calendar Export\FFD Calendar Calendar.ics. Since October 2026 the macro builds the file in memory, writes it to a local temporary file, checks that it is complete, and copies it into place with automatic retries; any problem is written to export-error.log in the same folder (see Section 7.13).
- **Nextcloud desktop app**: Syncs the FFD Calendar Export folder to Nextcloud automatically. The file is typically synced within seconds of the macro writing it.

Full setup instructions for rebuilding this system on a new computer are in U:\Fire\BWehner\FFD Calendar Export\FFD Calendar Export Setup.txt.

## 7.6 Configuration

Key constants in src/index.js: DAYS_TO_SHOW (default 6), CACHE_SECONDS (default 900 = 15 minutes), CACHE_VERSION (increment to bust all cached pages immediately), FILTER_EXACT, FILTER_CONTAINS, ALLDAY_COLORS. NWS constants: NWS_OFFICE (FGF), NWS_GRID_X (65), NWS_GRID_Y (57), NWS_ALERT_ZONE (NDZ039 — Cass County ND forecast zone, retained for reference only), NWS_COUNTY_CODE (NDC017 — Cass County ND county zone, used for all alert queries — see Section 7.6.1), SHOW_WEATHER (boolean to enable/disable all weather fetches). Internal constant: ICS_WINDOW_BUFFER_DAYS (default 1 — extra days kept on each side of the displayed days when skipping out-of-range calendar events; not normally edited — see Section 7.12).

## 7.6.1 NWS Alert Geolocation: Zone vs. County (June 2026 Fix)

This Worker's NWS alert query was updated using the same fix applied to weather-display (see Section 10.6.1 for full background). In summary: NWS Tornado Watches and Severe Thunderstorm Watches are county-based products that are never returned by a forecast-zone query (`?zone=NDZ039`). The Worker now queries by county zone (`?zone=NDC017`), which returns both county-based alerts and all zone-based alerts mapped to Cass County. `NWS_ALERT_ZONE` is retained in the code as a reference constant only. The alert processing logic also falls back to the `ends` field when `expires` is absent, matching the weather-display fix.

## 7.7 NWS Weather Display

> **Currently disabled.** SHOW_WEATHER = false in src/index.js. The features below are built and available in the code but do not render on deployed displays. To re-enable, set SHOW_WEATHER = true in src/index.js, deploy to staging, test, and merge to main.

Weather data is fetched from api.weather.gov with no API key required. All NWS fetches are edge-cached separately from the page cache.

**Daily forecast:** Every column header shows the high temperature (orange), low temperature (blue), a condition icon and short forecast text, and wind direction and speed.

**Hourly forecast strip:** The today panel body shows remaining hours at configurable intervals. Each slot shows the hour label, temperature, and an inline SVG weather icon.

**Alert banners:** Active NWS alerts for Cass County (queried via county zone NDC017 — see Section 7.6.1) appear as full-width colored banners above the calendar panels, sorted by severity. Red = Warning, Orange = Watch, Yellow = Advisory. Each banner shows the alert name and when it ends.

**Alert badges:** Upcoming alerts not yet active appear as severity-colored pills in affected day column headers.

## 7.8 Event Filtering & All-Day Colors

FILTER_EXACT and FILTER_CONTAINS control which events are hidden. ALLDAY_COLORS assigns custom colors per event title. Current shift colors: A Shift = dark green, B Shift = off-white with dark text, C Shift = dark red.

## 7.9 Manual Calendar Update

To update the calendar outside of a normal login: open Outlook, open the VBA editor (Developer tab → Visual Basic), click on a line inside the body of the Application_Startup procedure (for example the “Dim strIcs As String” line, not the comment block at the top of the module), and press F5. If F5 opens an empty list of macros instead of running the export, the cursor is not inside the procedure. The Nextcloud desktop app will sync the updated file automatically within seconds.

To force an immediate cache refresh: increment CACHE_VERSION in src/index.js by 1, deploy to staging, test, and merge to main.

## 7.10 Health Check Behavior

The /healthz endpoint probes the Nextcloud WebDAV URL before reporting status. It also probes api.weather.gov (NWS reachability), but only when SHOW_WEATHER is true. While SHOW_WEATHER is false, the NWS probe is skipped and /healthz reports “nws: skipped (SHOW_WEATHER is false)”, so an NWS outage cannot mark the calendar as degraded when weather is not being used. Setting SHOW_WEATHER back to true resumes the NWS probe automatically with no other change.

The Nextcloud probe distinguishes between three outcomes:

- **Reachable (healthy):** The probe receives an ok response or HTTP 207 (Multi-Status, the normal WebDAV response).
- **Authentication failed (degraded):** The probe receives HTTP 401 or 403. This specifically indicates the NEXTCLOUD_PASSWORD secret is invalid, expired, or has been revoked in Nextcloud — even though Nextcloud itself is online and reachable. The detail message returned by /healthz reads “nextcloud: authentication failed (401) — check NEXTCLOUD_PASSWORD” (or 403).
- **Unexpected status (degraded):** Any other non-ok status (e.g. 404, 500) is reported as “nextcloud: unexpected status NNN”.

This distinction matters because a 401/403 means the calendar page itself will fail and display the “CALENDAR UNAVAILABLE” error page to firefighters — it is not a transient issue and will not self-resolve. Earlier versions of this Worker treated 401 as “reachable” (healthy), which meant UptimeRobot would report the Worker as healthy even while the live calendar displays were showing “CALENDAR UNAVAILABLE.” This was corrected in June 2026 and is live on main.

## 7.11 Nextcloud App Password Rotation

If UptimeRobot reports calendar-display as degraded with “nextcloud: authentication failed,” or if station displays show “CALENDAR UNAVAILABLE” while /healthz for other Workers is healthy, the Nextcloud app password has likely expired or been revoked. To resolve:

1. Log into Nextcloud as the account the Worker uses (the account associated with NEXTCLOUD_USERNAME).
1. Click the avatar (top right) → Settings → Security (left sidebar) → Devices & sessions.
1. Enter a descriptive name (e.g. “FFD Calendar Display”) in the App name field and click Create new app password.
1. Copy the generated password immediately — Nextcloud will not show it again.
1. (Recommended) Revoke the old app password by finding it in the Devices & sessions list and clicking the trash/X icon next to it.
1. In the Cloudflare dashboard, go to Workers & Pages → calendar-display → Settings → Variables and Secrets, find NEXTCLOUD_PASSWORD, click Edit, and paste the new password. Save.
1. Repeat step 6 for the calendar-display-staging environment so both environments stay in sync.

No redeployment is required — secret changes take effect on the next request. Because the error page is served with Cache-Control: no-store, there is no stale cached error to clear; the display will recover on its next refresh cycle (within CACHE_SECONDS, default 15 minutes, since a fresh page is rendered and re-cached only on success).

**Note:** Updating the GitHub repository secrets does not affect this Worker. NEXTCLOUD_PASSWORD, NEXTCLOUD_USERNAME, and NEXTCLOUD_URL are Cloudflare Worker secrets only — they are read directly by the Worker via env at runtime and are never referenced by deploy.yml or any GitHub Actions step. The Cloudflare dashboard is the only place these three secrets need to be updated.

## 7.12 CPU Time Budget & October 2026 Optimization

**Background.** Cloudflare's Workers Free plan allows 10 ms of CPU time per request. CPU time is time spent computing; it does not include time spent waiting on network requests such as the Nextcloud fetch. On October 1, 2026, Cloudflare emailed a “Workers CPU limit exceeded” notice reporting that the Free-plan limit had been hit at least 100 times in the previous 24 hours. According to Cloudflare's documentation, a Worker that exceeds the limit returns Error 1102 to the client. The Metrics page for calendar-display nevertheless showed 0 “Exceeded CPU Time Limits” errors during that period; the reason for that mismatch is not known.

**Diagnosis.** calendar-display was the only Worker clearly over the limit, with a P90 CPU time of roughly 54 ms (its own Metrics page, last 24 hours) to 70 ms (Cloudflare's all-Workers view), and a P99 of about 106 ms. Every other Worker was between 1 and 9 ms (P90), several of them close to the limit. The page cache is kept separately in each Cloudflare data center, so each data center that serves a display rebuilds the page once per CACHE_SECONDS. These rebuilds (cache misses) were roughly 10–15% of requests (estimate from Oct 1 metrics); the other requests were cache hits costing about 1 ms. On every cache miss the Worker downloads and parses the full ICS export (about 45 KB and 147 events at the time of diagnosis) and renders the page. Code review found two likely causes, confirmed only by the before/after measurements below: (1) Intl.DateTimeFormat objects, which are expensive to construct, were being built repeatedly (per event and per displayed day), and (2) every event in the 30-day export was fully parsed even though only DAYS_TO_SHOW days are displayed. The contribution of each individual change was not measured separately.

**What changed.** Deployed to main on October 1, 2026 (CACHE_VERSION raised from 24 to 25 so all cached pages were rebuilt):

- **Reused date formatters.** The date and time formatters are created once when the Worker loads (module-level `FMT_*` constants) and reused. Timezone formatters used for converting ICS times are cached by `getZonePartsFormatter()`.
- **Date-windowed ICS parsing.** `parseIcs()` now receives a date window (the displayed days plus `ICS_WINDOW_BUFFER_DAYS`, default 1). Using only the raw date text at the start of each event's DTSTART/DTEND, events entirely outside the window are skipped before any timezone conversion or text unescaping. Multi-day all-day events that began before the window but run into it are kept. An event whose raw date cannot be read is always kept so the full parser decides about it.
- **Per-day event lookup.** `getEventsForDate()` now reuses the Central-time date string computed when each event was parsed instead of re-formatting every event for every displayed day.
- **Health check.** The NWS probe in /healthz runs only when SHOW_WEATHER is true (see Section 7.10).

The rendered page is unchanged. Before deployment the old and new code produced identical HTML in 72 test renders (synthetic 147-event calendar, four layouts, clock times including midnight and daylight-saving boundaries).

**The 30-day export window is intentional and must not be shortened.** The Outlook macro exports the next 30 days so the ICS file stays current even while the person who maintains it is away (see Section 7.5). The Worker now skips out-of-range events itself, so the export window does not need to be reduced to control CPU time.

**Results (October 1 – 5, 2026).**

|**Measurement**                                  |**Before (Sep 30 – Oct 1, 2026)**                            |**After (Oct 1 – Oct 5, 2026; 4.89k requests)**|
|-------------------------------------------------|-------------------------------------------------------------|--------------------------------------------|
|P90 CPU time                                     |About 54 ms (Worker Metrics) to 70 ms (all-Workers view)     |3.99 ms for the period (one early 15-minute bucket on Oct 1 reached 9.93 ms)|
|P99 CPU time                                     |About 106 ms                                                 |9.98 ms for the period; highest hourly values reviewed were 16.1 ms (Oct 2, 07:00) and 15.05 ms (deployment hour)|
|P999 CPU time                                    |Not recorded                                                 |12.5 ms for the period; highest hourly values reviewed were 15.7–17.8 ms|
|“Exceeded CPU Time Limits” errors shown in Metrics|0                                                            |0                                           |
|Cloudflare “CPU limit exceeded” email            |Received October 1, 2026 (at least 100 occurrences in 24 h)  |None received through October 5, 2026       |

A staging test in which nearly every request was forced to be a cache miss (using ?bg=dark) showed a legend P90 of 9.21 ms but a chart point of 12.43 ms for one time bucket, so some cache misses can still exceed 10 ms.

**Status (as of October 5, 2026): resolved.** Over the first four days on main (4.89k requests, version a30e23d9) there were no Cloudflare CPU-limit emails, 0 errors, 0 “Exceeded CPU Time Limits,” and all station displays were checked and showed the calendar correctly. CPU time is about 14 times lower at P90 and about 10 times lower at P99 than before the change. A small share of requests (about 1%, mostly page-cache rebuilds) still uses about 10 ms of CPU time or more, with occasional peaks of 15–18 ms; none of these produced an error, a failed page, or a Cloudflare email during the review period. The issue is considered closed. It should be reopened only if Cloudflare sends another “Workers CPU limit exceeded” email; in that case start with Section 14.3 and the options below. The reason the notice first arrived on October 1, when the code had not changed in about a month, was not determined. Possible causes, none verified, are a change in the ICS file (size, event count, or timestamp format), a change in how many requests were cache misses, or a change on Cloudflare's side in notification or measurement.

**If the CPU-limit email returns (options not implemented or tested in production).** Local profiling of the current code (Node, not Cloudflare's runtime, so indicative only) showed the remaining cost is mostly in the ICS parser: stepping through every line of the full 30-day file was the largest single cost, followed by timezone conversion. Options, most to least promising: (1) locate each event's block in the raw text and check its start date before splitting and unfolding the whole file, so skipped events cost almost nothing; (2) parse an event's end time (DTEND) only for events that start today, since it is needed only to hide events already finished today — this would remove only a small share of the remaining cost. In a local test, the first renders in a freshly started process were several times slower than later ones, which is consistent with the higher peaks seen after quiet periods (an inference, not verified on Cloudflare); this means the total work done per cache miss matters more than how fast the code runs once warm.

**Reading the CPU Time charts.**

- **P90** is the 90th percentile: within a time bucket, 90% of requests used that much CPU time or less, and the slowest 10% used more. The 10 ms limit applies to each individual request, so P90 alone can understate the problem. Watch P99, P999, and the highest values on the chart.
- Because most requests are cache hits (about 1 ms), P90 swings with the share of cache misses in each bucket. A low P90 does not by itself prove that cache misses are under the limit.
- When checking a change, use the version dropdown on the Metrics page to select only the newest version. Otherwise the chart mixes old and new code.
- The legend values can differ from individual chart points. Hover over the chart for the actual value of each bucket and use the highest as the conservative figure.
- Requests with ?bg=dark bypass the page cache entirely, so loading the staging URL with that parameter several times exercises the expensive cache-miss path. These requests do not write to the cache, so they slightly understate a real cache miss.

**Rules for future changes to this Worker.**

1. Create Intl.DateTimeFormat objects once at module level; never inside a function that runs per event or per request.
2. Filter or skip data before doing expensive work on it (parsing, timezone conversion, regular expressions).
3. Keep /healthz light; it is not cached and is requested every 5 minutes by UptimeRobot.
4. Before merging any change to the parse or render path, load the staging URL with ?bg=dark several times and compare CPU Time for the staging version against the figures above.
5. Increment CACHE_VERSION when the rendered output changes (Section 7.9).

The system is designed to stay within the Cloudflare Workers Free plan. Moving to a paid plan raises the limit but is not currently in use; reducing per-request work comes first.

## 7.13 Outlook Export Macro: Lock Errors and Truncated Exports (October 2026 Fix)

**Symptom.** On October 6, 2026 the exported ICS file, and therefore the displays, had no events after October 9, even though the Outlook calendar had many later events.

**Cause.** The ICS file was incomplete. It ended in the middle of a line (the end time of an October 9 event), had 35 BEGIN:VEVENT lines but only 34 END:VEVENT lines, and had no END:VCALENDAR line. The macro writes events in date order, so everything after the point where the write stopped was missing. The macro's error log (export-error.log) recorded the Windows error “another process has locked a portion of the file” (Automation error -2147024863) at 07:42:22 that morning, and the same error on August 28, September 4, September 17, October 5, and October 6, 2026. The macro had not been modified since it was installed (February/March 2026); on October 6 the macro running in Outlook was compared with the saved copy of its source and the two matched. The original macro wrote the file one line at a time directly inside the folder watched by the Nextcloud desktop app. The most likely explanation (an inference, not confirmed) is that the sync app or antivirus opened the file while it was being written. Which program was responsible was not determined, and neither was why the errors began in late August.

**Why the displays did not show an error.** The calendar-display Worker reads whatever the file contains. It ignores an event that is cut off, and it has no check for a complete file, so an incomplete export silently shortens the calendar instead of producing an error page. The macro runs only when Outlook starts, so an incomplete file stays in place until Outlook is next started or the export is run by hand (Section 7.9).

**The fix (macro rewritten October 2026).** The ICS content and the 30-day export window are unchanged, so the Worker needed no change. The macro now:

1. Reads the Outlook calendar and builds the entire ICS text in memory, without touching the synced folder.
2. Writes the text to a temporary file in the local Windows temp folder, which the sync app does not watch.
3. Checks the temporary file is complete: it ends with END:VCALENDAR and every BEGIN:VEVENT has a matching END:VEVENT.
4. Copies the checked file over the export file in a single operation. If the copy fails (for example the file is locked) or the result is not the expected size, it waits and tries again.
5. Leaves the existing export untouched if fewer than the minimum number of events were found, so an empty read of the calendar can never blank the displays.
6. Writes every failure, and every retry that was needed, to export-error.log, including which step was running.

Settings at the top of the macro:

|**Constant**           |**Value**              |**Purpose**                                                                              |
|-----------------------|-----------------------|-----------------------------------------------------------------------------------------|
|EXPORT_PATH            |U:\Fire\BWehner\FFD Calendar Export\FFD Calendar Calendar.ics|Where the finished ICS file is placed (synced to Nextcloud).|
|ERROR_LOG_PATH         |U:\Fire\BWehner\FFD Calendar Export\export-error.log|Log of errors, retries, and recoveries.|
|DAYS_AHEAD             |30                     |Days of calendar exported. Intentional; must not be shortened (Section 7.12).            |
|MIN_EXPECTED_EVENTS    |1                      |Fewer events than this and the existing export file is left unchanged.                   |
|MAX_COPY_ATTEMPTS      |5                      |Number of tries to copy the finished file into place.                                    |
|RETRY_DELAY_MS         |4000                   |Wait between tries, in milliseconds (4 seconds).                                         |
|TEMP_FILE_NAME         |FFD_Calendar_Export.tmp|Name of the temporary file in the local Windows temp folder.                             |

If every retry fails, Outlook is unresponsive for about 16 seconds during startup (four waits of four seconds). The macro's old header comment mentioned rclone and Google Drive; that was out of date and has been corrected. The file is synced by the Nextcloud desktop app (Section 7.5).

**Reading export-error.log.**

- **ERROR** means the export failed. The message names the step that was running. If the failure happened while copying the file into place, the export file may be incomplete, and the message gives the path of a complete temporary copy that can be copied over the export file by hand.
- **WARNING** means one copy attempt failed (usually a temporary lock) and the macro is waiting to retry.
- **OK ... after a temporary lock** means the export succeeded on a retry. This shows the fix working and needs no action.
- No new entries means the export ran without problems.

**Checking that an export is healthy.** Open the ICS file in Notepad and press Ctrl+End: the last line must be END:VCALENDAR. Searching the file for BEGIN:VEVENT shows how many events it contains. Check export-error.log for new ERROR lines.

**Installing or changing the macro.**

1. Back up the current macro source (a saved .txt copy) and, with Outlook closed, the VbaProject.OTM file.
2. In Outlook press Alt+F11, expand Project1 (VbaProject.OTM), then Microsoft Outlook Objects, and double-click ThisOutlookSession. Select all existing code and replace it with the new macro source.
3. Choose Debug → Compile VBAProject. Fix any error it highlights. Save the project.
4. Run the export by hand (Section 7.9). Confirm export-error.log has no new ERROR lines, the ICS file ends with END:VCALENDAR, the event count looks right, and the Nextcloud app has synced it.
5. Close and reopen Outlook to confirm the export also runs automatically at startup.
6. Keep a reference copy of the macro source as a .txt file in the FFD Calendar Export folder. The setup instructions in that folder (FFD Calendar Export Setup.txt) should match the current macro.

**Status (as of October 10, 2026): fix installed and working.** After installation the macro ran at Outlook startup with no errors and the ICS file updated. The lock errors were intermittent (about five in six weeks), so a short test period cannot prove they are gone. The retry logic is designed to absorb them and logs a WARNING when it does. Reopen this item if export-error.log shows an ERROR entry, or the calendar is again missing events.

**Known limitations (accepted).**

- If all copy attempts fail, the export file can be left incomplete and nobody is alerted automatically; the log must be checked.
- The Worker does not check the ICS file for completeness. A Worker-side check (reject an incomplete file and keep showing the last good page) was considered and not implemented, because the macro fix addresses the cause. If incomplete exports recur, this is the next step.
- The export runs only when Outlook starts or when run by hand. This is unchanged.

# 8. Project: Probationary Firefighter Display

## 8.1 Purpose & Problem Solved

The Probationary Firefighter Display Worker automates the introduction of new hires department-wide by reading firefighter bio data from a Google Sheet and photos from a Google Drive folder, then rendering a rotating spotlight page that cycles through every active new hire on the same 3-day rotation used by the Daily Message Display. Firefighters are considered active for 365 days from their hire date and drop off automatically with no manual action required.

## 8.2 Repository & Deployment

|**Item**         |**Value**                                                                                                                                                             |
|-----------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------|
|GitHub Repository|github.com/wehnerb/probationary-firefighter-display                                                                                                                   |
|Production URL   |<https://probationary-firefighter-display.bwehner.workers.dev/>                                                                                                       |
|Staging URL      |<https://probationary-firefighter-display-staging.bwehner.workers.dev/>                                                                                               |
|Worker File      |src/index.js                                                                                                                                                          |
|Secrets Required |CLOUDFLARE_API_TOKEN, CLOUDFLARE_ACCOUNT_ID (GitHub); GOOGLE_SERVICE_ACCOUNT_EMAIL, GOOGLE_PRIVATE_KEY, GOOGLE_SHEET_ID, GOOGLE_DRIVE_FOLDER_ID (Cloudflare dashboard)|

## 8.3 URL Parameters

The ?layout= parameter controls which design is rendered. Default is wide. The full layout adds a “PROBATIONARY FIREFIGHTER SPOTLIGHT” title bar at the top.

## 8.4 How It Works

1. The Worker authenticates with Google using the shared service account, generating a short-lived OAuth2 access token.
1. Firefighter records from the Google Sheet and the Drive photo file listing are fetched in parallel.
1. The active list is built: only firefighters whose hire date is within 365 days of today are included. The list is sorted by hire date ascending, then badge number ascending, for a stable consistent rotation order.
1. The current firefighter is selected using the same rotation block index as daily-message-display: floor(daysElapsed / ROTATION_DAYS) % activeCount, anchored to January 23, 2026 in America/Chicago time.
1. A self-contained HTML page is returned with the photo on the left and bio information on the right (wide/full layouts) or photo stacked above bio information (split/tri layouts). The photo is served via a streaming proxy route (/photo/{fileId}) — the display browser requests the photo from the Worker, which fetches it from Drive server-side using the service account token and streams the bytes directly to the browser. No base64 encoding is performed. The service account token never appears in any client-visible URL. Photo scaling is layout-aware: wide/full layouts use object-fit: cover (the photo fills the tall narrow column, anchored to the top so faces are not cropped); split/tri layouts use object-fit: contain (the photo is scaled down to fit the short wide strip so the full image is always visible, with the dark panel background filling the sides).
1. If Q&A answers are long enough to exceed the available panel height, a ResizeObserver measures the rendered content height after document.fonts.ready fires — ensuring the hardware has finished font rendering before any measurement is taken. If overflow is confirmed across two consecutive animation frames, a unique CSS @keyframes animation scrolls the Q&A content upward once: pausing at the top, scrolling through all content at a speed clamped between QA_MIN_SCROLL_SPEED_PX_PER_SEC and QA_MAX_SCROLL_SPEED_PX_PER_SEC, then pausing at the bottom. No scroll occurs when all Q&A content fits on screen.
1. The meta-refresh interval is set to the exact seconds until the next 7:30 AM Central rotation boundary. No server-side caching is used — every request generates a fresh page to ensure the display always reflects the current rotation.

## 8.5 Rotation Logic

The rotation is identical to daily-message-display in all respects: 3-day blocks, anchored to January 23, 2026, advancing at 7:30 AM Central (DST-safe). After the last active firefighter the list loops back to the first. When a new firefighter is added or an existing one ages out, the active list is recalculated from scratch on the next request.

## 8.6 Data Sources

**Google Sheet:** Fire Station Display – Probationary Firefighter Information (tab: Firefighters). Both the sheet and the Drive folder must be shared with the service account email (Viewer permission).

**Google Drive folder:** Fire Station Display – Probationary Firefighter Images. Photos are named firstnamelastname.jpg (e.g. makenziekotchman.jpg). The filename entered in the Photo column of the sheet must match the Drive filename (case-insensitive).

## 8.7 Google Sheet Column Reference

|**Column Header**|**Required?**|**Notes**                                                                                                         |
|-----------------|-------------|------------------------------------------------------------------------------------------------------------------|
|Name             |Yes          |Full name                                                                                                         |
|Hire Date        |Yes          |Format: YYYY-MM-DD (e.g. 2026-01-19). Used to calculate active status.                                            |
|Rank             |No           |e.g. Probationary Firefighter.                                                                                    |
|Shift            |No           |A, B, C, Days, or Recruit Academy                                                                                 |
|Badge            |No           |Plain badge number. Used for secondary sort within the same hire date.                                            |
|Photo            |No           |Filename of photo in Drive folder (e.g. makenziekotchman.jpg). Match is case-insensitive.                         |
|Hometown         |No           |Displayed in the fixed fields group alongside Hire Date, Shift, and Badge                                         |
|Any other column |No           |Treated as a dynamic Q&A pair. Header = question, cell = answer. Displayed below the fixed fields in column order.|

## 8.8 Configuration

|**Constant**                  |**Description**                                                                                                                                                              |
|------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
|ROTATION_DAYS                 |Consecutive days each firefighter displays before advancing (default 3). Must match daily-message-display.                                                                   |
|ROTATION_ANCHOR               |Anchor date for rotation (2026-01-23). Matches daily-message-display. Do not change unless intentionally resetting the cycle.                                                |
|HIRE_ACTIVE_DAYS              |Days after hire date a firefighter remains on the display (default 365). Firefighters drop off automatically.                                                                |
|SHEET_TAB_NAME                |Name of the tab in the Google Sheet (default: Firefighters).                                                                                                                 |
|QA_SCROLL_DURATION_SECONDS    |Total seconds budgeted for one complete Q&A scroll cycle (pause top → scroll → pause bottom). Controls scroll speed — a longer value produces slower scrolling (default: 60).|
|QA_SCROLL_PAUSE_SECONDS       |Seconds to pause at the top and bottom of the Q&A scroll before and after scrolling (default: 12).                                                                           |
|QA_MIN_SCROLL_SPEED_PX_PER_SEC|Minimum Q&A scroll speed in pixels per second. Prevents imperceptibly slow scrolling when content only slightly overflows (default: 5).                                      |
|QA_MAX_SCROLL_SPEED_PX_PER_SEC|Maximum Q&A scroll speed in pixels per second. Prevents uncomfortably fast scrolling when answers are very long (default: 50).                                               |

## 8.9 Adding a New Hire

1. Add a new row to the Firefighters tab in the Google Sheet with at minimum Name and Hire Date filled in.
1. Upload the firefighter’s photo to the Drive folder named firstnamelastname.jpg. squoosh.app can be used to resize images, but larger images do load reliably.
1. Enter the filename in the Photo column of the sheet. Fill in any other available fields. Blank fields are automatically omitted from the display.
1. The Worker will pick up the changes on the next page load. No code change or redeployment is needed.

## 8.10 Adding a New Class with Different Questions

Add new column headers to row 1 of the sheet for the new questions. Blank cells are simply omitted — no code changes are required.

# 9. Project: Department News Display

## 9.1 Purpose & Problem Solved

Station displays previously had no way to show department-wide announcements or station-specific news in a structured, easy-to-manage format. The Department News Display Worker renders a card-based news feed sourced from a Google Sheet, with separate tabs for department-wide news and each individual fire station. Content is managed directly in the sheet by authorized personnel and appears on displays within minutes of being saved.

## 9.2 Repository & Deployment

|**Item**         |**Value**                                                                                                                                     |
|-----------------|----------------------------------------------------------------------------------------------------------------------------------------------|
|GitHub Repository|github.com/wehnerb/department-news-display                                                                                                    |
|Production URL   |<https://department-news-display.bwehner.workers.dev/>                                                                                        |
|Staging URL      |<https://department-news-display-staging.bwehner.workers.dev/>                                                                                |
|Worker File      |src/index.js                                                                                                                                  |
|Secrets Required |CLOUDFLARE_API_TOKEN, CLOUDFLARE_ACCOUNT_ID (GitHub); GOOGLE_SERVICE_ACCOUNT_EMAIL, GOOGLE_PRIVATE_KEY, GOOGLE_SHEET_ID (Cloudflare dashboard)|

## 9.3 URL Parameters

|**Parameter**|**Default**|**Options**                 |**Description**                                                                                                                  |
|-------------|-----------|----------------------------|---------------------------------------------------------------------------------------------------------------------------------|
|station      |(required) |dept, 1, 2, 3, 4, 5, 6, 7, 8|Which tab to display. dept = department-wide news; 1–8 = station news. This parameter is required — omitting it returns HTTP 400.|
|layout       |split      |wide, split, tri, full      |Display column width.                                                                                                            |

Example URLs:

`https://department-news-display.bwehner.workers.dev/?station=dept` — department-wide news

`https://department-news-display.bwehner.workers.dev/?station=7` — Station 7 news

## 9.4 How It Works

1. The Worker authenticates with Google using the shared service account and fetches the relevant sheet tab from the Google Sheet. Sheet data is cached server-side for CACHE_SECONDS (default 5 minutes) to reduce API calls across display cycles.
1. Rows are processed: expired items are hidden, recurring items are expanded to their current active occurrence based on the recurrence interval and stop-after date, and items are sorted with new items first (sorted by posted date descending), then regular items (sorted by posted date descending).
1. Items posted within NEW_ITEM_THRESHOLD_DAYS of today are highlighted with a “NEW” badge and a distinct lighter card background.
1. A self-contained HTML page is returned with all active items displayed as cards. Each card shows the title, body text, posted date, and expires date. Posted By is shown only when the field is not blank — it is omitted entirely from the card when empty. Line breaks entered in the sheet using Alt+Enter are preserved on the display. All dates are displayed in America/Chicago time.
1. If content overflows the visible area, a ResizeObserver measures the rendered content height after document.fonts.ready fires — ensuring the Pi hardware has finished font rendering before any measurement is taken. If overflow is detected, a unique CSS @keyframes animation is injected that scrolls content upward once, pausing at the top for SCROLL_PAUSE_SECONDS, scrolling through all content, then pausing at the bottom. No scroll occurs when content fits on screen. This approach is reliable under CPU load because the ResizeObserver + CSS animation runs independently of JavaScript’s main thread after initialization.

## 9.5 Google Sheet Structure

The Google Sheet contains one tab per feed: a “Department News” tab for department-wide news and tabs named “FS#1” through “FS#8” for each station. The sheet must be shared with the service account email (Viewer permission).

## 9.6 Google Sheet Column Reference

|**Column Header**|**Required?**|**Notes**                                                                                                                                                          |
|-----------------|-------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------|
|Title            |Yes          |Headline shown at the top of the card in bold                                                                                                                      |
|Content          |Yes          |Body text of the news item. Alt+Enter line breaks are preserved on display.                                                                                        |
|Posted           |Yes          |Date/time the item was posted. Format: M/D/YY H:MM AM/PM or M/D/YYYY H:MM AM/PM. Used for sorting and “new” badge calculation. Interpreted as America/Chicago time.|
|Expires          |Yes          |Date/time the item stops showing. Same format as Posted. Items without a valid expiry are not shown. Interpreted as America/Chicago time.                          |
|Posted By        |No           |Name of the person who posted the item. Shown in card metadata only when not blank — omitted entirely when empty.                                                  |
|Recurring        |No           |Interval in days for recurring items (e.g. 7 for weekly). Leave blank for non-recurring items.                                                                     |
|Stop After       |No           |Date after which a recurring item no longer repeats. Format: M/D/YY H:MM AM/PM.                                                                                    |

## 9.7 Configuration

|**Constant**               |**Description**                                                                                                      |
|---------------------------|---------------------------------------------------------------------------------------------------------------------|
|NEW_ITEM_THRESHOLD_DAYS    |Number of days after posting that an item shows the NEW badge (default: 3).                                          |
|DISPLAY_DURATION_SECONDS   |How long the display shows this Worker before cycling. Used to calculate scroll animation speed (default: 20).       |
|SCROLL_PAUSE_SECONDS       |Seconds to pause at top and bottom of scroll animation (default: 5).                                                 |
|MIN_SCROLL_SPEED_PX_PER_SEC|Minimum scroll speed in pixels per second (default: 20).                                                             |
|MAX_SCROLL_SPEED_PX_PER_SEC|Maximum scroll speed in pixels per second (default: 75).                                                             |
|CARD_BODY_LINE_HEIGHT      |Line height for card body text (default: 1.25). Adjust if line spacing appears too tight or too loose on hardware.   |
|CARD_META_LINE_HEIGHT      |Line height for card metadata (default: 1.6).                                                                        |
|CACHE_SECONDS              |Seconds to cache sheet data server-side (default: 300 = 5 minutes). Does not cache the rendered HTML page.           |
|DELETE_EXPIRED_AFTER_DAYS  |Days after expiry before expired non-recurring rows are automatically deleted by the scheduled cron job. -1 disables.|

**Scheduled Cleanup:** The automated row deletion runs on a Cloudflare cron trigger configured in wrangler.toml — it is not triggered by display requests. If the cron trigger is removed from wrangler.toml or is not active in the Cloudflare dashboard, expired rows will not be deleted automatically regardless of the DELETE_EXPIRED_AFTER_DAYS value. The cleanup iterates all tabs (Department News and FS#1–8) and deletes non-recurring rows that expired more than DELETE_EXPIRED_AFTER_DAYS days ago. The Google service account requires the spreadsheets scope (read and write) for this feature.

## 9.8 Adding News Items

1. Open the Google Sheet and navigate to the appropriate tab (Department News or the relevant station tab).
1. Add a new row with at minimum Title, Content, Posted, and Expires filled in.
1. Fill in Expires with the date/time the item should stop showing. Items without a valid Expires date are not shown on the display.
1. New items appear on displays within 5 minutes (one CACHE_SECONDS interval).

## 9.9 Recurring Items

To create a recurring item (e.g. a weekly reminder), fill in the Recurring column with the interval in days (e.g. 7 for weekly) and optionally set a Stop After date. The item will repeat from its original Posted date at the configured interval until the Stop After date is reached. Recurring items are never auto-deleted by the cleanup cron job — they must be removed manually from the sheet.

# 10. Project: Weather Display

## 10.1 Purpose & Problem Solved

Station displays needed a dedicated full-screen weather display showing current conditions, a 3-day forecast, an animated radar map, and weather alerts in a single view. The Weather Display Worker fetches data from the NWS public API, AirNow API, and RainViewer API and renders a responsive HTML page tailored to each display layout. No manual updates are required — all data refreshes automatically.

## 10.2 Repository & Deployment

|**Item**         |**Value**                                                                                  |
|-----------------|-------------------------------------------------------------------------------------------|
|GitHub Repository|github.com/wehnerb/weather-display                                                         |
|Production URL   |<https://weather-display.bwehner.workers.dev/>                                             |
|Staging URL      |<https://weather-display-staging.bwehner.workers.dev/>                                     |
|Worker File      |src/index.js                                                                               |
|Secrets Required |CLOUDFLARE_API_TOKEN, CLOUDFLARE_ACCOUNT_ID (GitHub); AIRNOW_API_KEY (Cloudflare dashboard)|

## 10.3 URL Parameters

|**Parameter**|**Default**|**Options**           |**Description**                                                                                                                                                                                   |
|-------------|-----------|----------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
|layout       |wide       |wide, full, split, tri|Display size. wide and full show the full display (radar + conditions + forecast band). split and tri show either radar or conditions only, controlled by the view parameter.                     |
|view         |conditions |radar, conditions     |For split and tri layouts only. Selects which half of the display to render.                                                                                                                      |
|bg           |(none)     |dark                  |Testing parameter. Forces a fully opaque dark background (#111111) for browser-based testing, bypassing the semi-transparent hardware background and the Cloudflare cache. Not for production use.|

## 10.4 How It Works

1. The Worker fetches data in parallel from multiple sources: NWS current conditions, NWS daily forecast, NWS active alerts for Cass County (queried via county zone NDC017 — see Section 10.6.1), AirNow AQI for Fargo (by latitude/longitude), and RainViewer radar frame metadata.
1. Each data source fails gracefully — if any upstream source is unavailable, the relevant section is omitted rather than returning an error page.
1. Weather data is edge-cached at Cloudflare to limit upstream API calls across all stations.
1. A self-contained HTML page is rendered for the requested layout and returned to the display.

## 10.5 Display Sections

**Layout Structure (wide/full):** The display is organized as four stacked horizontal bands from top to bottom: alert banners, hero zone, accent divider, and 3-day forecast.

**Alert Banners:** Active NWS weather alerts appear as full-width colored banners stacked at the top of the display. Banner colors follow NWS severity: red for Warnings, orange for Watches, amber for Advisories. Up to MAX_DISPLAY_ALERTS banners are shown simultaneously; the most severe alerts take priority. Future alerts (not yet active) fill any remaining banner slots with a “begins Day H:MM AM/PM” label. When alerts are present, hero zone content scales proportionally to fit the remaining vertical space. The forecast band has a fixed height that does not change with alert count.

**Hero Zone:** Two columns side by side. The left column (~41.5% of display width) is the current conditions panel. The right column (~58.5%) is the radar map.

**Current Conditions Panel:** Three stacked regions inside the left column.

- *Hero row:* Four columns — (1) current temperature with feels-like temperature below it; (2) condition icon and condition text with AQI badge below; (3) today’s high and low temperatures; (4) sunrise and sunset times with icons.
- *Stats grid:* A 3×2 grid showing Wind Speed, Humidity (with dew point), Pressure, Wind Gust, Visibility, and UV Index. Labels use tertiary text; values use primary text. Wind Gust shows “None” when no gust is reported. Pressure always displays two decimal places. UV Index includes the EPA category label (Low, Moderate, High, Very High, Extreme).
- *Accent divider:* A 4px solid bar in the department accent color spanning the width of the conditions panel, visually aligning with the radar progress bar on the right.

**Radar:** An animated radar map using Leaflet.js with the CartoDB Dark Matter base layer and RainViewer radar tile overlays. The animation uses a double-buffer approach for smooth frame transitions. The radar shows the most recent RADAR_FRAME_COUNT frames in a loop with a hold on the final (most recent) frame.

**3-Day Forecast:** A full-width band below the hero zone showing three equal-width day cards side by side. Each card shows the day name, condition icon, condition text, wind line, precipitation probability, and high/low temperatures in a single compact row. If any forecast day has an active or future alert, a full-width colored alert banner appears at the top of that day’s card. Days without an alert show a blank placeholder of equal height to keep the card content vertically aligned across all three cards. Alert banner colors follow NWS severity and include a bottom border in the matching severity color.

**Split/Tri Layout:** The conditions panel and a stacked 3-day forecast (three cards stacked vertically) fill the conditions-only view. The radar-only view is unchanged.

## 10.6 Configuration

|**Constant**          |**Description**                                                                                                                                                                                                                                                              |
|----------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
|LOCATION_LAT          |Latitude of the display location (Fargo: 46.8772).                                                                                                                                                                                                                           |
|LOCATION_LON          |Longitude of the display location (Fargo: -96.7898).                                                                                                                                                                                                                         |
|NWS_OFFICE            |NWS forecast office identifier (FGF for Fargo).                                                                                                                                                                                                                              |
|NWS_GRID_X / Y        |NWS grid point coordinates (65, 57 for Fargo).                                                                                                                                                                                                                               |
|NWS_ALERT_ZONE        |NWS forecast zone identifier (NDZ039 = Cass County ND). Retained for reference only — alert queries use NWS_COUNTY_CODE (see note below).                                                                                                                                  |
|NWS_COUNTY_CODE       |NWS county zone identifier used for all alert queries (NDC017 = Cass County ND). County-zone queries return both county-based alerts (Tornado Watch, Severe Thunderstorm Watch) and all zone-based alerts mapped to the county — see Section 10.6.1.                       |
|RADAR_ZOOM            |Leaflet map zoom level for the radar display (default: 8). At zoom 8, RainViewer 512-px tiles are used automatically.                                                                                                                                                        |
|RADAR_FRAME_COUNT     |Number of historical radar frames to animate (default: 18).                                                                                                                                                                                                                  |
|RADAR_FRAME_MS        |Milliseconds between radar animation frames (default: 200).                                                                                                                                                                                                                  |
|RADAR_HOLD_MS         |Milliseconds to hold on the final (most recent) frame before looping (default: 2500).                                                                                                                                                                                        |
|RADAR_INIT_DELAY_MS   |Milliseconds to wait after page load before initializing the Leaflet map. Allows Pi hardware layout to complete first.                                                                                                                                                       |
|RADAR_OPACITY         |Opacity of the radar overlay tiles (0.0–1.0).                                                                                                                                                                                                                                |
|MAX_DISPLAY_ALERTS    |Maximum number of alert banners displayed simultaneously (default: 3).                                                                                                                                                                                                       |
|ALERT_BANNER_HEIGHT_PX|Vertical pixels reserved per active alert banner for proportional hero zone scaling (default: 52). The forecast band is not affected by alert count.                                                                                                                         |
|FORECAST_BAND_HEIGHT  |Height in pixels of the 3-day forecast band by layout key — `{ full: 160, wide: 130 }`. Adjust to control the vertical size of the forecast cards.                                                                                                                           |
|HI_TEMP_COLOR         |Worker-local hex color for today’s high temperature label and value (default: `#f0a060`). Independent from shared utils so it can be tuned without affecting other workers.                                                                                                  |
|LO_TEMP_COLOR         |Worker-local hex color for tonight’s low temperature label and value (default: `#80c8f0`).                                                                                                                                                                                   |
|SOLAR_COLOR           |Worker-local hex color for sunrise/sunset values and the hourly temperature curve (default: `#f0c040`).                                                                                                                                                                      |
|DISPLAY_BG_COLOR      |Worker-local hex color for the page background on hardware displays (default: `#111111`). Used together with DISPLAY_BG_OPACITY to control how much of the display hardware shows through. Must be a 6-digit hex string.                                                     |
|DISPLAY_BG_OPACITY    |Opacity of the hardware display background on a 0–1 scale (default: 0.25). 0 = fully transparent (hardware fully visible), 1 = fully opaque (no hardware visible). Does not affect the `?bg=dark` or full layout backgrounds, which always use DARK_BG_COLOR at full opacity.|

## 10.6.1 NWS Alert Geolocation: Zone vs. County (June 2026 Fix)

**Background:** The NWS API uses two separate, non-overlapping geolocation systems for alerts:

- **Forecast zone queries** (`?zone=NDZ039`) return zone-based alerts only — winter storms, heat advisories, fire weather, and similar large-scale, longer-duration hazards.
- **County zone queries** (`?zone=NDC017`) return *both* county-based alerts (Tornado Watch, Severe Thunderstorm Watch — issued by the Storm Prediction Center against county FIPS codes) *and* all zone-based alerts mapped to that county.

Per the official NWS Geolocation primer (weather.gov/media/documentation/docs/NWS_Geolocation.pdf): *"county based alerts are not mapped to zones... requests such as `?zone=NDZ039` will not contain county-based products."*

**The bug:** Prior to this fix, the Worker queried alerts using `NWS_ALERT_ZONE` (`NDZ039`, the forecast zone). This meant Tornado Watches and Severe Thunderstorm Watches — arguably the two most operationally important alert types for a fire station display — were silently never returned by the API, regardless of whether one was active. This went undetected until a live Tornado Watch failed to appear on the display in June 2026.

**The fix:** Alert queries were switched to use `NWS_COUNTY_CODE` (`NDC017`, the Cass County zone), which returns the full set of applicable alerts — both county-based and zone-based — for the station's location. `NWS_ALERT_ZONE` is retained in the code as a reference constant but is no longer used in any fetch call.

**Secondary fix (same change):** The alert processing logic previously discarded any alert whose `expires` field was null, with no fallback. Some SPC-issued products populate `ends` instead of `expires`. The processing logic was updated to fall back to `ends` when `expires` is absent, preventing valid alerts from being silently dropped.

**Note on display ordering:** Alerts are sorted by NWS `severity` field only (Extreme → Severe → Moderate → Minor). Tornado Watch and Severe Thunderstorm Warning are both assigned "Severe" severity by NWS, so when both are active they are treated as tied and displayed in API return order rather than by alert type. This is intentional and matches NWS's own severity classification — it is not a bug. A secondary sort by `urgency` (Immediate vs. Future) could be added in the future if a different ordering is preferred, but has not been implemented.

This same zone-vs-county issue and fix also applies to the calendar-display Worker's weather integration — see Section 7.6.1.

## 10.7 AirNow API Key


The AIRNOW_API_KEY secret must be set in the Cloudflare dashboard for both the production and staging workers. Register for a free key at docs.airnowapi.org. If the key is missing or the AirNow API is unavailable, the AQI badge is silently omitted and all other weather data renders normally.

### AirNow Endpoint (June 2026 Migration)

In June 2026, AirNow retired several legacy API endpoints effective September 30, 2026. The Worker was updated to use the new consolidated endpoint before the deadline.

|**Item**|**Value**|
|--------|---------|
|Current endpoint|`https://www.airnowapi.org/aq/observation/current/ziplatlong/`|
|Retired endpoint (do not use)|`https://www.airnowapi.org/aq/observation/latLong/current/`|
|Response field for AQI value|`nowcastAQI` (was `AQI` in the old API)|
|Response field for pollutant name|`parameterName` (was `ParameterName` in the old API)|

The new endpoint accepts `latitude` and `longitude` parameters. The `distance` parameter is no longer accepted — lookup boundary is now server-controlled (defaulting to 25 miles).

### Rotating the API Key

If the AirNow API key needs to be rotated (e.g., if it was ever exposed in a screenshot, log, or shared environment):

1. Log in to docs.airnowapi.org.
2. Click **Generate New API Key** in the top right corner. The old key is immediately invalidated.
3. Copy the new key — it is only shown once.
4. Go to Cloudflare dashboard → Workers & Pages → `weather-display` → Settings → Variables and Secrets → edit `AIRNOW_API_KEY` → paste the new key → Save.
5. Repeat for `weather-display-staging`.
6. Verify by loading `/healthz` on both Workers and confirming `airnow: reachable` appears in the response.

## 10.8 NWS Configuration

NWS_USER_AGENT is set as a plain [vars] entry in wrangler.toml and identifies the Worker to the NWS API as required by NWS terms of service.

# 11. Project: Newsletter Slides Display

## 11.1 Purpose & Problem Solved

Pieces of the department newsletter are static images that are only relevant for a limited time. The Slide Timing Proxy shows a fixed Google Slides presentation and cannot remove a slide after a date, and the Daily Message Display rotates by day, so neither fits content that expires. The Newsletter Slides Display Worker shows zero, one, or several slide images from a dedicated Google Drive folder, starts each one on its start date, takes it down automatically after its end date, and rotates between the slides that are active through the day. When no slides are active it shows a “no current slides” screen, optionally with a QR code that links to the full newsletter. Nothing in the code or a spreadsheet has to be edited to add or retire a slide: the file name carries the dates.

## 11.2 Repository & Deployment

|**Item**         |**Value**                                                                                                                                                                                                     |
|-----------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
|GitHub Repository|github.com/wehnerb/newsletter-slides-display (public; default branch: staging)                                                                                                                              |
|Production URL   |<https://newsletter-slides-display.bwehner.workers.dev/>                                                                                                                                                      |
|Staging URL      |<https://newsletter-slides-display-staging.bwehner.workers.dev/>                                                                                                                                              |
|Health Check     |<https://newsletter-slides-display.bwehner.workers.dev/healthz>                                                                                                                                               |
|Worker File      |src/index.js                                                                                                                                                                                                  |
|Google Drive Folder|“Fire Station Display - Newsletter Slides” — a folder used only by this Worker, shared with the service account email (Viewer). Staff who add slides need Editor access.                                  |
|Secrets Required |CLOUDFLARE_API_TOKEN, CLOUDFLARE_ACCOUNT_ID, GOOGLE_SERVICE_ACCOUNT_EMAIL, GOOGLE_PRIVATE_KEY, GOOGLE_DRIVE_FOLDER_ID — all GitHub repository secrets. The three GOOGLE_ values are pushed to Cloudflare on every deploy by deploy.yml.|

Repository layout: .github/workflows/deploy.yml, .github/workflows/sync-utils.yml, src/index.js, src/shared/ (copied automatically from ffd-station-display-utils — never edit it by hand), README.md, wrangler.toml.

## 11.3 URL Parameters

The ?layout= parameter accepts: full (1920x1075), wide (1735x720, default), split (852x720), tri (558x720). The ?bg=dark parameter gives a solid dark background for browser testing. The Worker serves three paths: / (the display page), /image/{fileId} (streams an image from Drive) and /healthz (health check). Any other path returns a 404 and only GET and HEAD requests are accepted.

## 11.4 How It Works

1. The Worker authenticates with Google using the shared service account (read-only Drive access) and lists the files directly inside the slide folder. Subfolders are ignored.
1. Files that are not JPG, PNG, GIF, or WebP images are ignored without comment (this is why a text file of instructions can safely live in the folder). Images with more than 16 million pixels are skipped.
1. Each remaining file name is checked against the naming convention in Section 11.6. A file named qr-code with an image extension is reserved for the QR code and is never treated as a slide. An image whose name breaks the convention is skipped and its name is written to the Worker logs.
1. A slide is active if today’s date in Central time falls within its start and end dates.
1. The slide whose turn it is (Section 11.5) is chosen and a self-contained HTML page is returned that shows it, scaled to the layout. The image itself is served through the streaming route /image/{fileId}, which fetches it from Drive server-side and streams the bytes to the display browser with no base64 encoding, keeping CPU usage low and avoiding Cloudflare’s 10 ms CPU limit.
1. The page reloads itself at the next slot change, and always just after midnight Central, so slides start and expire on time.

The Worker does no server-side caching: every page load reads the folder fresh, so a new file is picked up on the next load.

## 11.5 Rotation Logic

Active slides take turns, one per time slot. Slots are counted from midnight Central time: slot number = floor(minutes since midnight ÷ ROTATION_MINUTES), and the slide shown is the active slide at position (slot number mod number of active slides) in the list. The list is ordered by start date, then by file name; slides with no start date (the until_ and always_ patterns) come before dated ones.

Because the slide shown depends only on the clock, it does not matter when or how often a display loads the page, so SmartHub preload timing has no effect. All dates and times use America/Chicago time and follow Daylight Saving Time.

As deployed in November 2026, ROTATION_MINUTES is 15, and each station’s display cycle (3–5 minutes depending on the station) shows this screen for about 15 seconds. A given slide therefore stays up across several cycles before the next slide takes its turn. Keep ROTATION_MINUTES comfortably longer than a full display cycle so no slide can be skipped.

If slides are added or expire during the day, the list changes and the rotation order shifts; this is harmless.

## 11.6 File Naming Convention

The file name tells the Worker when to show the slide. Dates are always YYYY-MM-DD. Both dates are included in the range, and a slide expires at midnight Central time at the end of its end date. The words to, until, and always are not case-sensitive.

|**Pattern**      |**Example**                                |**Shown**                                  |
|-----------------|-------------------------------------------|-------------------------------------------|
|`START_to_END_name`|`2026-11-18_to_2026-11-25_flu-clinic.jpg`|From the start date through the end date   |
|`until_END_name` |`until_2026-11-25_flu-clinic.jpg`          |Immediately, through the end date          |
|`always_name`    |`always_station-hours.jpg`                 |Until the file is removed                  |

An image is never shown if its name matches none of the patterns, if a date is not a real calendar date (for example 2026-02-30), or if the end date is before the start date. The reserved QR code name is described in Section 11.7.

## 11.7 QR Code and the No-Slides Screen

When there are no active slides, the Worker shows a “no current slides” screen: a title and message, and — if a QR code image is present — the QR code with a short caption underneath. The QR code is simply an image named qr-code with an image extension (for example qr-code.png) stored in the same Drive folder as the slides. It is never shown as a slide. PNG is recommended, at least 600 x 600 pixels, with a plain white border so it scans reliably; SVG is not supported. To change where the QR code leads, replace the file in the folder. If more than one qr-code file exists, the first by file name is used and a warning is logged.

The title, message, and caption wording are constants near the top of src/index.js (Section 11.10). The same title and message appear whether or not a QR code is present; only the QR code and its caption are added. The caption and QR code are not part of the slide rotation.

The same screen is shown if Google Drive cannot be read (without a QR code, because the folder listing is unavailable).

## 11.8 Image Requirements

- **Dimensions:** any width and height works. The image is scaled to fit the layout and centered, never cropped or stretched. If its shape differs from the layout’s, empty bars appear at the sides or at the top and bottom. For the best fit with no empty space on the wide layout, use 1735 x 720 pixels.
- **Pixel limit:** images with more than 16,000,000 pixels in total (for example, larger than 4000 x 4000) are skipped, to protect the display hardware from decoding enormous images. Drive reports an image’s dimensions shortly after upload; an image without reported dimensions is allowed.
- **File size (storage):** under 2 MB is recommended. The Worker refuses files over 5 MB.
- **Types:** JPG, PNG, GIF, and WebP. SVG is deliberately not allowed.

## 11.9 Content Management

**Adding a slide:** name the image using one of the patterns in Section 11.6 and drop it into the Drive folder. No other step is required.

**Removing a slide:** slides come down automatically after their end date. To remove one sooner, move it into any subfolder or delete it. Expired files may stay in the folder, but clearing them out occasionally keeps the folder tidy (the Worker reads at most 1,000 files and logs a warning beyond that).

**Instructions for staff:** a text file named “HOW TO NAME SLIDE FILES.txt” lives in the folder and explains the naming convention in plain language. The Worker ignores it because it is not an image.

**Access control:** anyone who can edit the folder controls what the displays show, including where the QR code leads. Keep edit access limited to the people who manage newsletter content.

**Testing note:** the staging and production Workers read the same Drive folder, so a test image is visible on production displays while it is in the folder. Remove test files when testing is finished.

## 11.10 Configuration

Key constants near the top of src/index.js:

|**Constant**                                      |**Description**                                                                                                  |
|--------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
|ROTATION_MINUTES                                  |Minutes each slide stays up before the next takes its turn (whole minutes, 1–1440). Currently 15.                  |
|DEFAULT_LAYOUT_KEY                                |Layout used when no ?layout= parameter is given (default: wide).                                                 |
|DISPLAY_TIME_ZONE                                 |Time zone for all date and time logic (America/Chicago).                                                         |
|NO_SLIDES_TITLE, NO_SLIDES_MESSAGE                |Wording of the no-slides screen. An empty message line is omitted.                                               |
|NO_SLIDES_QR_CAPTION                              |Line shown under the QR code. Leave empty for no caption.                                                        |
|QR_FILE_BASENAME                                  |File name (without extension) reserved for the QR code image (default: qr-code).                                 |
|MIN_REFRESH_SECONDS                               |Smallest page auto-refresh interval, to prevent reload loops near a slot change (default: 30).                   |
|ERROR_RETRY_SECONDS                               |Wait before retrying after a Drive failure or a failed image (default: 60).                                      |
|NO_SLIDES_REFRESH_SECONDS                         |How often the no-slides screen re-checks the folder for new files (default: 300).                                |
|IMAGE_RETRY_DELAY_MS                              |Wait before the single automatic retry when an image fails to load (default: 4000).                              |
|MAX_IMAGE_BYTES                                   |Largest image file the Worker will serve (default: 5 MB).                                                        |
|MAX_IMAGE_PIXELS                                  |Largest image, in total pixels, that will be displayed (default: 16,000,000).                                    |

Changing any constant is a normal code change: edit src/index.js on the staging branch, test, then merge to main (Section 13.2).

## 11.11 Failure Behavior

|**Situation**                                |**What the display shows**                                                                                                                             |
|---------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------|
|Google Drive cannot be read                  |The no-slides screen (no QR code). The page reloads and tries again after ERROR_RETRY_SECONDS. The HTTP status stays 200 so the display never shows a browser error.|
|A slide image fails to load in the browser   |One automatic retry after IMAGE_RETRY_DELAY_MS (covers the short delay Google Drive can have after an upload). If it fails again, the no-slides screen is shown and the page reloads after ERROR_RETRY_SECONDS.|
|A required secret is missing or malformed    |A CONFIGURATION ERROR screen that names the problem (never a secret value).                                                                           |
|A file name breaks the convention            |The file is not shown. Its name and the reason are written to the Worker logs.                                                                         |

## 11.12 Security Notes

- No secret, key, token, or folder ID is written to a page or a log.
- Drive access is read-only (drive.readonly scope) through the shared service account.
- The /image/{fileId} route serves a file only if it is in the configured folder, not in the trash, an allowed image type, under the byte and pixel limits, and either an active slide or the QR code image. Any other Drive file is refused even if its ID is known. Refusals use generic responses that reveal nothing about the file.
- The Worker does not set X-Frame-Options (the displays load it in an iframe) and accepts only GET and HEAD requests.
- Because images are streamed through the Worker, the Drive folder can stay private and does not need public sharing.
- The folder remains the control point: restrict who has Editor access (Section 11.9).

## 11.13 Setting Up or Rebuilding This Worker

The full process, including the order in which files must be uploaded, is in Section 13.5. Worker-specific items: create the dedicated Drive folder and share it with the service account email, create the five GitHub secrets listed in Section 11.2, and put the three GOOGLE_ values in GitHub (deploy.yml pushes them to Cloudflare; see Section 2.3).

# 12. Shared Utilities

## 12.1 Overview

The ffd-station-display-utils repository contains shared design tokens, helper functions, and utility modules used by all 9 Worker projects. All Workers sync from this repository automatically via a GitHub Actions pipeline whenever the shared utilities are updated (Section 12.4). In each Worker repository the shared files live in src/shared/, which the pipeline overwrites — never edit files there by hand.

## 12.2 Repository

|**Item**         |**Value**                                   |
|-----------------|--------------------------------------------|
|GitHub Repository|github.com/wehnerb/ffd-station-display-utils|
|Worker File      |N/A — this is a shared library, not a Worker|
|Branches         |staging and main. The sync pipeline is triggered from and reads staging; changes are promoted to main afterward.|
|Visibility       |Public. Each Worker’s sync-utils.yml reads this repository using that Worker repository’s built-in token, which can only read public repositories other than its own.|
|Contents         |utils/ folder (the shared modules) and a VERSION file at the repository root; .github/workflows/notify-workers.yml|
|Secret           |WORKER_REPO_DISPATCH_TOKEN (Section 2.3)|

## 12.3 Shared Modules

|**File**        |**Purpose**                                                                                                                                   |
|----------------|----------------------------------------------------------------------------------------------------------------------------------------------|
|colors.js       |Design tokens: background colors, font stacks, text opacity hierarchy, border levels, card elevation levels, accent color                     |
|alert-colors.js |Alert color tokens: warning/watch/advisory backgrounds, borders, and text colors                                                              |
|layouts.js      |Layout dimensions for wide, full, split, and tri display sizes                                                                                |
|fetch-helpers.js|fetchWithTimeout helper with AbortController for all upstream API fetches                                                                     |
|html.js         |escapeHtml and sanitizeParam helper functions for safe HTML output                                                                            |
|google-auth.js  |getAccessToken function for Google service account JWT authentication                                                                         |
|rotation.js     |getTodayString, getDaysElapsed, getBlockIndex, getSecondsUntilNextRotation, formatHireDate helpers for 3-day rotation and hire date formatting|

## 12.4 Auto-Sync Pipeline

Shared files flow from this repository to every Worker repository through two workflows.

**1. notify-workers.yml (in ffd-station-display-utils).** It runs when the VERSION file changes on the staging branch, or when started by hand from the Actions tab. It sends a repository_dispatch event (type utils-updated) to each of the 9 Worker repositories listed in its REPOS list, using the WORKER_REPO_DISPATCH_TOKEN secret. A repository that is missing from the list, misspelled, or outside the token’s repository access is not notified (a misspelled name makes the job fail with a 404 on that entry).

**2. sync-utils.yml (in each Worker repository).** It runs when it receives that event, or when started by hand. It:

1. Checks out the Worker’s staging branch and the utils repository’s staging branch.
1. Replaces src/shared/ with the utils/ folder plus the VERSION file.
1. If nothing changed, stops with “Already up to date.” Otherwise it commits the change to a branch named sync-utils-v followed by the version, opens a pull request to staging, and merges it.
1. Merges staging into main with a merge commit, then deletes the sync branch.

To trigger the pipeline: update VERSION in ffd-station-display-utils on the staging branch together with the utility change. To sync a single Worker repository by hand: Actions → Sync shared utils → Run workflow in that repository.

Requirements and behaviors to know:

- sync-utils.yml must be on each Worker repository’s default branch (staging). The events that start it only run workflow files from the default branch.
- Each Worker repository must allow GitHub Actions to create and approve pull requests (Settings → Actions → General). If it does not, the pull request is not created and src/shared/ is not updated.
- The steps that open and merge the pull request are written to continue after a failure and only print a message. A green check therefore does not prove the sync worked: confirm that the sync pull request shows as merged and that src/shared/ is current.
- The final merge into main uses a merge commit. In a brand-new repository the first run fails at that step with “refusing to merge unrelated histories,” which is harmless because staging is already updated; it goes away once staging has been merged into main with a merge commit. Squash or rebase merges between staging and main break the shared history and make this step fail on every future sync.
- GitHub documents that actions taken with the built-in GITHUB_TOKEN do not start other workflow runs. After a sync, check the Worker repository’s Actions tab to confirm that its deploy workflow ran; if it did not, the Worker keeps running the previous code until the next push or a manual re-run of the latest deploy.
- To add a new Worker to the pipeline, see Section 13.5.

## 12.5 Design Token Summary

|**Token**       |**Value**                   |**Usage**                                                |
|----------------|----------------------------|---------------------------------------------------------|
|DARK_BG_COLOR   |#111111                     |Solid background for full layout and ?bg=dark testing    |
|FONT_STACK      |“Segoe UI”, Arial, Helvetica|Primary sans-serif font stack (web fonts avoided on Pi)  |
|FONT_STACK_SERIF|Georgia, “Times New Roman”  |Serif stack for message body text                        |
|ACCENT_COLOR    |#FF0000                     |FFD red — used only for functional elements              |
|TEXT_PRIMARY    |rgba(255,255,255,0.92)      |Main body text, values, titles                           |
|TEXT_SECONDARY  |rgba(255,255,255,0.68)      |Labels, metadata, subdued content                        |
|TEXT_TERTIARY   |rgba(255,255,255,0.50)      |Timestamps, hints, least-important content               |
|TEXT_SUPPORTING |rgba(255,255,255,0.75)      |Supporting content between SECONDARY and PRIMARY         |
|BORDER_SUBTLE   |rgba(255,255,255,0.10)      |Standard card borders, dividers                          |
|BORDER_STRONG   |rgba(255,255,255,0.18)      |Emphasized separators, today-panel borders               |
|CARD_RECESSED   |rgba(255,255,255,0.04)      |Subtly inset cells within a card                         |
|CARD_BASE       |rgba(255,255,255,0.07)      |Standard card surface                                    |
|CARD_ELEVATED   |rgba(255,255,255,0.11)      |Slightly raised blocks within a card                     |
|CARD_HEADER     |rgba(255,255,255,0.16)      |Today/primary panel header — highest card-level elevation|

|**Design Rule**                                                                                                                                                                                                   |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
|ACCENT_COLOR is used functionally only — never as a decorative top border or purely ornamental element. Only font weights 400 and 700 are portable across the Pi font fallback chain. Do not use 300, 500, or 600.|

# 13. Deployment & Maintenance Workflow

## 13.1 Branch Strategy

Each Worker repository uses two branches. All changes must go through staging before being merged to main. Staging is the default branch in every Worker repository, so edits and uploads made in the GitHub web interface land on staging rather than main.

|**Branch**|**Deploys To**                      |**Purpose**                                         |
|----------|------------------------------------|----------------------------------------------------|
|staging   |[worker]-staging.bwehner.workers.dev|Testing and validation of changes before production |
|main      |[worker].bwehner.workers.dev        |Live production environment used by station displays|

## 13.2 Making a Code Change

Follow these steps in order for all code changes:

1. Create a new branch from staging in the GitHub repository (or edit directly in staging for simple one-line changes).
1. Edit the relevant file(s) using the GitHub browser editor.
1. Commit to the new branch and open a pull request targeting staging.
1. GitHub Actions will automatically deploy the change to the staging Worker within about 30–45 seconds.
1. Test the staging Worker URL in a browser to confirm the change works as expected.
1. For display-dependent changes, temporarily update the endpoint URL on a single station display to the staging Worker URL. Confirm the change works correctly on actual display hardware before proceeding.
1. Once both browser and display tests pass, merge the pull request to staging, then merge staging to main. Merge staging into main with a merge commit (GitHub’s “Create a merge commit”), never a squash or rebase, so the shared-utilities sync keeps working (Section 12.4).
1. GitHub Actions will automatically deploy to the production Worker within about 30–45 seconds.
1. Verify the production Worker URL is functioning correctly. Return any display temporarily set to staging to the production URL.

### Rolling Back a Change

- **Simple rollback:** Edit the file in the staging branch in GitHub to restore the previous value.
- **Commit history rollback:** Go to the branch in GitHub, click the commit history, find the target commit, and use Revert.
- **Production rollback:** Go to the main branch commit history, find the last known-good commit, and use GitHub’s Revert feature. This triggers a redeployment automatically.
- **Re-sync branches after rollback:** After reverting main, merge main back into staging immediately.

## 13.3 Important Deployment Notes

|**Important**                                                                                                                                                                                             |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
|Never deploy directly to main without first testing in staging. If a deployment fails, the previous version remains live — Cloudflare does not replace the running Worker until a new deployment succeeds.|

|**Important — X-Frame-Options Header**                                                                                                                                                                                                                                           |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
|The X-Frame-Options: SAMEORIGIN header must never be added to any of these Workers. All Workers are loaded as full-screen iframes by the display system, and this header causes the browser to block the page from rendering entirely, producing an immediate white error screen.|

|**Important — Cached Pages on the Image Proxy**|
|-----------------------------------------------|
|station-image-proxy sends its HTML pages with Cache-Control: private, max-age=3600 (PAGE_CACHE_SECONDS). After a change to the page itself (layout, CSS, markup) is deployed, displays that already hold a cached copy keep it for up to one hour, and a rollback has the same delay. To make a page change visible sooner, lower PAGE_CACHE_SECONDS first, wait for the old value to expire, then deploy the change. Camera images and the /proxy route are never cached, so changes to MAPPING keys or proxy behavior take effect immediately (Section 3.10).|

## 13.4 Monitoring Deployments

Every deployment is logged in the Actions tab of the GitHub repository. All 9 Workers are monitored by UptimeRobot via their /healthz endpoints. Alerts are sent by email when any Worker goes down or recovers.

## 13.5 Adding a New Worker to the System

This is the process used to add newsletter-slides-display (Section 11) in November 2026. It assumes the new Worker follows the same structure as the existing ones. Every upload and edit goes to the staging branch until step 12. Create folders in the GitHub web editor by typing the path (for example .github/workflows/file.yml) into the file name box; a slash creates the folder.

**Required repository layout:**

|**Path**                         |**Purpose**                                                                                  |
|---------------------------------|---------------------------------------------------------------------------------------------|
|.github/workflows/deploy.yml     |Deploys to Cloudflare: pushes to staging deploy the -staging Worker, pushes to main deploy production|
|.github/workflows/sync-utils.yml |Pulls shared utilities from ffd-station-display-utils (Section 12.4)                         |
|src/index.js                     |The Worker code                                                                              |
|src/shared/                      |Shared utilities — created by the sync workflow, never edited or uploaded by hand            |
|wrangler.toml                    |Worker name, entry point, and the [env.staging] block                                        |
|README.md                        |Repository overview and content-management instructions                                     |

**Steps:**

1. Create any external resources the Worker needs (for example, a Google Drive folder) and share them with the service account email. Record any IDs the Worker needs.
1. Create a public GitHub repository under wehnerb, initialized with a README (this creates the main branch). Create a staging branch from main, then go to Settings → Branches and make staging the default branch. The default branch matters: the sync workflow is started by events that only run workflow files on the default branch.
1. Go to Settings → Actions → General and set the workflow permissions to match an existing Worker repository, including the option that allows GitHub Actions to create and approve pull requests. If that option is off, the sync cannot open its pull request.
1. On the staging branch, create the .github/workflows folder and upload sync-utils.yml. Inside the file, the repository name is read from github.repository, so the same file works in any repository. (The copy in daily-message-display instead names its own repository in two places, which must be changed if that copy is reused.)
1. Run the sync by hand: Actions → Sync shared utils → Run workflow (branch staging). Confirm src/shared/ on staging now contains VERSION plus the shared modules (currently alert-colors.js, colors.js, fetch-helpers.js, google-auth.js, html.js, layouts.js, and rotation.js), and that the sync pull request shows as merged. Do not rely on the green ticks alone (Section 12.4).
1. Expect the last step of that first run, “Merge staging into main,” to fail with “refusing to merge unrelated histories.” It is harmless in a brand-new repository: staging is already updated and main receives everything in step 12. Optionally delete the leftover sync-utils-v* branch.
1. Add the GitHub repository secrets: CLOUDFLARE_API_TOKEN and CLOUDFLARE_ACCOUNT_ID, plus any Worker-specific secrets. List each Worker-specific secret in the secrets: block of deploy.yml so GitHub pushes it to Cloudflare on every deploy (Section 2.3).
1. Upload, one file at a time to staging: src/index.js (into the existing src folder, not src/shared), wrangler.toml, README.md, and finally .github/workflows/deploy.yml. Uploading deploy.yml last means its first run starts only after everything it needs is present.
1. Confirm the Actions run is green, the staging Worker exists in the Cloudflare dashboard with its secrets, and the staging /healthz URL reports healthy. Test the Worker thoroughly in a browser, then on one station display using the staging URL.
1. Remove any test content (staging and production may share external resources such as the Drive folder).
1. Add the new repository to notify-workers.yml in ffd-station-display-utils: put the entry at the end of the REPOS list with the exact spelling “wehnerb/repository-name” (a misspelled owner or name makes the notify job fail with a 404), and update the repository count in its comments. Then edit the WORKER_REPO_DISPATCH_TOKEN fine-grained token (GitHub → Settings → Developer settings → Personal access tokens → Fine-grained tokens) so its repository access includes the new repository.
1. Open a pull request from staging to main and merge it with “Create a merge commit.” Do not squash or rebase, which breaks the shared history and makes the sync’s automatic merge into main fail on every future utils update. Confirm the production deploy and the production /healthz URL.
1. Add an UptimeRobot monitor on the production /healthz URL (5-minute interval, same alert contacts as the others) and add the production URL to the station displays following the normal rollout.
1. Update this documentation: add the repository to Sections 2.2 and 2.3, add a project section like Section 11, update Sections 12.4, 14.4, and 14.5, and update every count of Workers.

# 14. IT Support Reference

## 14.1 Overview for IT Staff

This system runs entirely on external cloud services and does not require any on-premises server infrastructure, VPNs, or internal network changes. The display screens require only outbound internet access to reach the Cloudflare Worker URLs. There is no software to install, no servers to patch, and no internal ports to open.

IT involvement would typically only be needed for:

- Network connectivity issues at the station that prevent displays from reaching external URLs.
- Display hardware or operating system issues at the screen level.
- Account access issues if the department member managing this system is unavailable.

## 14.2 Network Requirements

The display screens must have outbound internet access on port 443 (HTTPS) to the following domains:

- *.workers.dev — Cloudflare Worker endpoints (all 9 Worker projects)
- docs.google.com — Google Slides embeds
- cdnjs.cloudflare.com — Leaflet.js CSS and JavaScript for the weather display radar map
- *.basemaps.cartocdn.com — CartoDB Dark Matter base map tiles for the radar display
- tilecache.rainviewer.com — RainViewer radar overlay tiles

Note: [www.dot.nd.gov](http://www.dot.nd.gov), usgs-nims-images.s3.amazonaws.com, and water.noaa.gov are fetched by the station-image-proxy Worker on its own network — not by the display screens directly. The display browser receives these images through the Worker’s /proxy endpoint on workers.dev. api.weather.gov, api.rainviewer.com, and [www.airnowapi.org](http://www.airnowapi.org) are similarly fetched by other Cloudflare Workers on their own network, not by the display screens directly. [www.googleapis.com](http://www.googleapis.com) is also accessed only by the Cloudflare Worker server-side (for Google Slides, Sheets, and Drive API calls). Photo and image proxy routes (/photo/ and /image/) on the daily-message-display, probationary-firefighter-display, and newsletter-slides-display Workers resolve back to those Workers — the display browser never contacts googleapis.com directly and does not require firewall access to that domain.

## 14.3 Troubleshooting

|**Symptom**                                      |**Likely Cause**                                                                        |**Resolution**                                                                                                                                                                                                                                                                                                                  |
|-------------------------------------------------|----------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
|Display shows blank/white screen                 |Network connectivity or display hardware issue                                          |Check internet connectivity at the screen. Verify the Worker URL loads in a browser on the same network.                                                                                                                                                                                                                        |
|Images show INVALID IMAGE KEY error              |Invalid or missing image key in the URL                                                 |Check the img parameter in the display URL against the MAPPING list in index.js (Section 3.9).                                                                                                                                                                                                                                  |
|Images show IMAGE UNAVAILABLE                    |Source camera or data feed is offline                                                   |Check the source URL directly. This is typically a third-party outage outside department control.                                                                                                                                                                                                                               |
|River gauge shows error page                     |NOAA API temporarily unavailable                                                        |The page retries automatically every 60 seconds. Check api.water.noaa.gov directly if persistent.                                                                                                                                                                                                                               |
|Slides cycling at wrong speed                    |Slide count API call failing or secrets missing                                         |Check Cloudflare Worker logs. Verify GOOGLE_SERVICE_ACCOUNT_EMAIL and GOOGLE_PRIVATE_KEY are present in the Cloudflare dashboard for slide-timing-proxy.                                                                                                                                                                        |
|Calendar shows “CALENDAR UNAVAILABLE”            |Nextcloud app password expired/revoked, ICS file empty/missing, or Nextcloud unreachable|Check /healthz for calendar-display first — if it reports “nextcloud: authentication failed,” rotate the Nextcloud app password (Section 7.11). If /healthz is healthy, check Cloudflare Worker logs for the specific fetch error and verify the ICS file exists in Nextcloud.                                                  |
|Cloudflare emails “Workers CPU limit exceeded,” or a page shows Cloudflare Error 1102|A Worker used more than the Free plan’s 10 ms of CPU time on a request. As of October 2026 the most likely source is a calendar-display cache miss|See Section 7.12. In Cloudflare, open the Worker → Metrics, filter to the newest version, and check CPU Time (P99 and highest values) and Errors by invocation status (“Exceeded CPU Time Limits”). The system is designed to stay on the Free plan, so reduce per-request work first.|
|Calendar is missing events within the displayed days, or the ICS file ends partway through an event|The Outlook export did not finish writing the file (for example the file was locked while it was being replaced). The Worker uses whatever the file contains and does not detect an incomplete file|Open export-error.log in the FFD Calendar Export folder and look for ERROR lines. Open the ICS file in Notepad: the last line must be END:VCALENDAR. Re-run the export (Section 7.9), then confirm the file and a display. If the log gives the path of a complete temporary copy, that file can be copied over the export file by hand. See Section 7.13.|
|Calendar shows no weather data                   |NWS API temporarily unavailable                                                         |The calendar renders without weather rather than showing an error. Typically self-resolves.                                                                                                                                                                                                                                     |
|Probationary firefighter photo does not load     |Drive permissions or file not found                                                     |Verify the Drive folder is shared with the service account email. Verify the Photo column filename matches the Drive filename. Check Cloudflare Worker logs for errors.                                                                                                                                                         |
|Probationary firefighter display shows no content|No firefighters hired within the past 365 days                                          |Check that hire dates are in YYYY-MM-DD format. Verify the sheet is shared with the service account email.                                                                                                                                                                                                                      |
|Department news shows “No current news”          |No active items in the sheet tab for this station                                       |Check the relevant tab in the Google Sheet. Verify Posted and Expires dates are correct and items are not expired.                                                                                                                                                                                                              |
|Tornado Watch or Severe Thunderstorm Watch not showing as an alert banner|Pre-June-2026 code querying NWS by forecast zone instead of county zone (fixed — see Sections 10.6.1 / 7.6.1)|Confirm the deployed Worker is running code from main after the June 2026 fix. If still missing, verify NWS_COUNTY_CODE (NDC017) is used in the alert fetch URL, not NWS_ALERT_ZONE.                                                                                                                            |
|Weather radar map is white or partially loaded   |Leaflet map initialization timing issue on Pi hardware                                  |This is a known intermittent issue. The display refreshes automatically and typically self-resolves on the next cycle.                                                                                                                                                                                                          |
|Weather display shows no AQI badge               |AIRNOW_API_KEY missing or AirNow API unavailable                                        |Check that AIRNOW_API_KEY is set in the Cloudflare dashboard for weather-display. The badge is silently omitted when unavailable.                                                                                                                                                                                               |
|UptimeRobot shows a Worker as down               |Worker returning non-200 status or timing out                                           |Load the /healthz URL directly in a browser. If it returns “status: healthy”, the alert may have been a transient blip. If it returns “status: degraded”, check the detail lines — for calendar-display, “nextcloud: authentication failed” points directly to Section 7.11. Otherwise check Cloudflare Worker logs for details.|
|Newsletter slide is not appearing               |The file name breaks the naming convention, a date is not a real date or the date range does not include today, the image is over the pixel limit, or the file is inside a subfolder|Check the name against the table in Section 11.6. Open the Worker’s logs in Cloudflare and look for an “Ignored file” line that names the file and the reason. Images over 16 million pixels are skipped, and files in subfolders are ignored on purpose.|
|Newsletter display shows the no-slides screen although slides exist|Google Drive could not be read (the page retries every 60 seconds), or every slide is outside its dates|Verify the Drive folder is shared with the service account email, GOOGLE_DRIVE_FOLDER_ID is correct in the GitHub secrets, and the latest deploy ran after any secret change (Section 2.3). Compare each slide’s dates with today’s date in Central time.|
|Newsletter display shows CONFIGURATION ERROR    |A required secret is missing or the folder ID is malformed                              |The screen names the problem. Fix the GitHub repository secret, then re-run the latest deploy for the staging and main branches.|
|QR code does not appear on the no-slides screen  |No file named qr-code with an image extension in the folder, an unsupported format such as SVG, or the file exceeds the size limits|Add a PNG named qr-code.png (at least 600 x 600 pixels, white border) to the Drive folder (Section 11.7).|
|Shared utils sync fails at “Merge staging into main” with “refusing to merge unrelated histories”|First sync in a new Worker repository, or staging was merged into main with a squash or rebase|Expected on the first sync of a new repository: staging is already updated. Merge staging into main with a merge commit and later syncs work. Delete the leftover sync-utils-v branch (Section 12.4).|
|“Notify worker repos” job fails with a 404 on one repository|The repository name is misspelled in notify-workers.yml, or the WORKER_REPO_DISPATCH_TOKEN token cannot access that repository|Check the spelling of every entry in the REPOS list, then check the token’s repository access and expiration date (Section 2.3).|
|Every Worker shows Error 1027, or daily requests approach 100,000|The combined requests of all Workers exceeded Cloudflare’s Free-plan limit of 100,000 per day (account-wide, resets at 00:00 UTC, which is 6 PM CST or 7 PM CDT)|Check the totals (Section 14.5). Displays recover automatically at the reset. To prevent a repeat, first lengthen the time each screen is displayed, especially image screens (Section 3.10); other options are listed there.|
|A camera image on a display looks old|Normal for these cameras: frames can be up to roughly 30 minutes old at the source. The Image Proxy never caches images|Open /proxy?key=KEYNAME on the Worker and compare the time overlay in the image with the current time. If the overlay is old, the source has not published a newer frame yet.|

## 14.4 Uptime Monitoring

All 9 Workers are monitored via UptimeRobot at 5-minute intervals. Each monitor checks the Worker’s /healthz endpoint and alerts by email when the Worker is down or recovers.

|**Monitor Name**                    |**URL**                                                               |
|------------------------------------|----------------------------------------------------------------------|
|FFD Station Image Proxy             |<https://station-image-proxy.bwehner.workers.dev/healthz>             |
|FFD Slide Timing Proxy              |<https://slide-timing-proxy.bwehner.workers.dev/healthz>              |
|FFD River Level Display             |<https://river-level-display.bwehner.workers.dev/healthz>             |
|FFD Daily Message Display           |<https://daily-message-display.bwehner.workers.dev/healthz>           |
|FFD Calendar Display                |<https://calendar-display.bwehner.workers.dev/healthz>                |
|FFD Department News Display         |<https://department-news-display.bwehner.workers.dev/healthz>         |
|FFD Probationary Firefighter Display|<https://probationary-firefighter-display.bwehner.workers.dev/healthz>|
|FFD Weather Display                 |<https://weather-display.bwehner.workers.dev/healthz>                 |
|FFD Newsletter Slides Display       |<https://newsletter-slides-display.bwehner.workers.dev/healthz>       |

## 14.5 Service Limits & Monitoring

|**Service**                        |**Free Tier Limit**               |**Est. Daily Usage (8 stations)**                                          |**Where to Check**                                    |
|-----------------------------------|----------------------------------|---------------------------------------------------------------------------|------------------------------------------------------|
|Cloudflare Workers (combined total)|100,000 req/day                   |About 89,000 req/day measured for the 24 hours to Nov 19, 2026 11:15 with 4 of the 8 stations on the cached Image Proxy; estimated 82,000–88,000 once all stations are cached (Section 14.5.1)               |dash.cloudflare.com → Workers & Pages → Overview      |
|Cloudflare Workers CPU time (per request)|10 ms per request (Free plan)|calendar-display after the Oct 2026 optimization (Oct 1–5): P99 9.98 ms, P999 12.5 ms (was P99 ~106 ms); typical request ~1 ms. Other Workers were 1–9 ms (P90) on Oct 1, 2026, several close to the limit. newsletter-slides-display (added November 2026) has not yet been measured — check its Metrics → CPU Time|dash.cloudflare.com → Workers & Pages → (Worker) → Metrics → CPU Time|
|Google Slides API                  |300 req/minute                    |At most 1 request per hour per cache version                               |console.cloud.google.com → APIs & Services → Dashboard|
|Google Sheets API                  |300 req/minute per project        |Low — edge-cached to limit calls                                           |console.cloud.google.com → APIs & Services → Dashboard|
|Google Drive API                   |1,000 req/100 seconds             |Low — one listing call per Worker request, results edge-cached. newsletter-slides-display is not cached: one listing call per page load plus two Drive calls per image served             |console.cloud.google.com → APIs & Services → Dashboard|
|NOAA NWPS / NWS APIs               |No documented hard limit          |Low — all responses edge-cached for 15–60 minutes                          |Monitor via displays on screens                       |
|AirNow API                         |500 req/hour                      |~8–16 req/hour across 8 stations — well within limit                       |docs.airnowapi.org → My Account                       |
|RainViewer API                     |No documented hard limit          |Low — metadata fetched per Worker request; tiles fetched by display browser|Monitor via weather display on screens                |
|UptimeRobot                        |50 monitors, 5-min interval (free)|9 monitors active — well within limit                                      |uptimerobot.com → Dashboard                           |

### 14.5.1 Measured Worker Request Volume (November 2026)

Cloudflare counts all Workers together against the 100,000 requests per day Free-plan limit. The count resets at 00:00 UTC (6 PM CST, 7 PM CDT), and when the limit is exceeded every Worker returns Error 1027 until the reset, so all station displays would fail and UptimeRobot would alert for all 9 monitors.

Measured invocations for the 24 hours ending Nov 19, 2026 at about 11:15 (dashboard: Workers & Pages → Overview → Worker Invocations). At that time 4 of the 8 stations were on the cached Image Proxy build running on the staging Worker, so Image Proxy traffic is split between two rows:

|**Worker**                       |**Invocations (24 h)**|
|---------------------------------|----------------------|
|station-image-proxy (production) |24,730                |
|station-image-proxy-staging (4 test stations, cached build)|25,440|
|newsletter-slides-display        |9,550                 |
|department-news-display          |8,010                 |
|probationary-firefighter-display |4,770                 |
|slide-timing-proxy               |4,180                 |
|river-level-display              |4,160                 |
|daily-message-display            |4,150                 |
|weather-display                  |2,740                 |
|calendar-display                 |1,160                 |
|**Total (top 10 Workers)**       |**88,890**            |

Notes:

- The newsletter-slides-display runs at about 100 invocations per 15 minutes (about 9,600 per day).
- probationary-firefighter-display roughly doubled (about 45 to about 85 per 15 minutes) at about 07:30 on Nov 19, when its display switched to a new person and all stations reloaded the images and data. The 24-hour total therefore understates the rate at that time; the combined rate was about 970–1,000 per 15 minutes (about 93,000–96,000 per day) before all stations were cached. The volume of this Worker may depend on which person is being shown.
- Before the display cycle was shortened on Nov 17, 2026, Image Proxy traffic was about 400 per 15 minutes; afterward it was about 650 (Section 3.10).
- After page caching reaches all eight stations the combined rate is estimated at about 880–910 per 15 minutes (about 82,000–88,000 per day). This is an estimate and should be replaced by a measured value.

**Daily check.** Around 5:30 PM Central, add up the Worker Invocations for the last 24 hours (the quota day ends at 6 PM). Working thresholds (rules of thumb, not Cloudflare values): under about 85,000 needs no action; 85,000–95,000 means lengthening screen display times (Section 3.10); over about 95,000 means acting the same day.

## 14.6 Contact & Escalation

For issues with application code, configuration, or Cloudflare/GitHub accounts, contact Brandon Wehner.

For issues with display screen hardware or operating system, contact the shift personnel responsible for the station alerting and communications system.

For local network connectivity issues preventing displays from reaching external URLs, contact the City of Fargo Information Services (IS) Department.

# 15. Planned Enhancements

## 15.1 Incident & Performance Data (First Due)

|**Enhancement**                   |**Description**                                                                                              |**Priority**|**Status**         |
|----------------------------------|-------------------------------------------------------------------------------------------------------------|------------|-------------------|
|Turnout Time Display              |Graphical display showing the percentage of calls where the crew met the department’s turnout time benchmark.|High        |Under Consideration|
|Turnout Time Trending             |Graphical representation showing turnout time trends over time.                                              |High        |Under Consideration|
|Incident Type by Final Disposition|Graphical breakdown of incident types using final report disposition.                                        |High        |Under Consideration|
|Time-of-Day Call Distribution     |Chart showing when calls are most frequent throughout the day.                                               |Medium      |Under Consideration|
|Year-over-Year Call Volume        |Comparison of call volume across years to identify growth trends.                                            |Medium      |Under Consideration|
|Geographic Call Clustering        |Heat map or cluster map showing where calls are concentrated within the response area.                       |Low         |Under Consideration|

## 15.2 Display Content Enhancements

|**Enhancement**                          |**Description**                                                                                                                                                                       |**Priority**|**Status**         |
|-----------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------|-------------------|
|Daily Safety Message Display             |Daily messages, quotes, and images displayed on station screens, rotating every 3 days.                                                                                               |Medium      |Complete           |
|Calendar Display                         |Station calendar with weather data. Exported from Outlook automatically via VBA macro, synced to Nextcloud, fetched by the Worker via WebDAV.                                         |Medium      |Complete           |
|NOAA Weather Alerts                      |Active and upcoming NWS weather alerts on calendar and weather displays with severity-colored banners and future alert badges.                                                        |Medium      |Complete           |
|Probationary Firefighter Display         |Rotating spotlight with photo and bio sourced from Google Sheets and Drive. Cycles through all active new hires on a 3-day rotation. Drops off automatically after 365 days from hire.|Medium      |Complete           |
|Department News Display                  |Card-based department-wide and station-specific news feed sourced from Google Sheets. Supports recurring items, new item highlighting, and auto-scrolling for overflow content.       |Medium      |Complete           |
|Weather Display                          |Full weather display with animated RainViewer radar, NWS current conditions, 3-day forecast, AirNow AQI, and active/future alert banners.                                             |Medium      |Complete           |
|Newsletter Slides Display                |Expiring newsletter slide images from a dedicated Drive folder. File names carry start and end dates; active slides rotate through the day; a no-slides screen with an optional QR code appears when none are active.|Medium      |Complete           |
|Birthday & Department Anniversary Display|Display birthdays and department hire anniversaries for the current week or month.                                                                                                    |Low         |Under Consideration|
|Retirement Countdown                     |Countdown display for upcoming retirements (with member permission).                                                                                                                  |Low         |Under Consideration|
|Additional River Gauges                  |Add additional NOAA gauge locations to the river-level-display GAUGES registry (e.g., Moorhead, MN — gauge MHDN8).                                                                    |Low         |Under Consideration|