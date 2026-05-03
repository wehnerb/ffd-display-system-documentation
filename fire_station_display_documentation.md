# Fire Station Display Board — System Documentation

*Technical Reference Guide for Department Staff, Administrators, and IT Support*

Last Updated: May 2, 2026

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
- [11. Shared Utilities](#11-shared-utilities)
- [12. Deployment & Maintenance Workflow](#12-deployment--maintenance-workflow)
- [13. IT Support Reference](#13-it-support-reference)
- [14. Planned Enhancements](#14-planned-enhancements)

# 1. System Overview

## 1.1 Purpose

The Fire Station Display Board system is a set of digital display screens deployed throughout the fire station that provide firefighters with real-time and regularly updated operational information. Content displayed includes traffic camera feeds, river gauge levels and hydrographs, Google Slides presentations with department announcements, rotating safety messages, a station calendar with weather data, a probationary firefighter spotlight, department and station-specific news, and a full weather display with animated radar.

The system was designed with two core constraints in mind:

- Display screens can only be configured with a single, static endpoint URL — no runtime parameters or user interaction is possible at the screen level.

- All infrastructure and tooling used must be free or within free tier service limits.

## 1.2 How the System Works

Because the display screens cannot resize images, authenticate to APIs, or make intelligent decisions at runtime, a layer of Cloudflare Workers sits between the screens and the data sources. Each Worker is a lightweight serverless function that runs at Cloudflare's edge network and handles the logic that the display screen cannot.

The general data flow for every display is:

| **Step**             | **Description**                                             |
|----------------------|-------------------------------------------------------------|
| Display Screen       | Loads a single, fixed Cloudflare Worker URL                 |
| Cloudflare Worker    | Processes the request, fetches data from external sources   |
| External Data Source | Returns camera images, river gauge data, slide counts, etc. |
| Response to Screen   | Worker returns the processed content ready for display      |

Each Worker is deployed independently and has its own URL. A display screen is configured with a Worker URL and from that point on requires no further maintenance unless the underlying data source or configuration changes.

## 1.3 Technology Stack

| **Technology**          | **Purpose**                                                    | **Cost**                    |
|-------------------------|----------------------------------------------------------------|-----------------------------|
| Cloudflare Workers      | Serverless edge functions that process requests                | Free tier (100,000 req/day) |
| GitHub                  | Source code storage and version control                        | Free                        |
| GitHub Actions          | Automatic deployment pipeline to Cloudflare                    | Free                        |
| Google Sheets API       | Content storage for daily messages, news, and firefighter bios | Free (service account auth) |
| Google Drive API        | Image storage for daily messages and firefighter photos        | Free (service account auth) |
| Google Slides API       | Slide count queries for the slide timing proxy                 | Free (service account auth) |
| Nextcloud WebDAV        | ICS calendar file delivery from Nextcloud to Worker            | Free (city infrastructure)  |
| NWS Public API          | Weather conditions, forecast, and alerts (no API key required) | Free / Public               |
| AirNow API              | Air Quality Index data for the weather display                 | Free (API key required)     |
| RainViewer API          | Animated radar tile data for the weather display               | Free / Public               |
| NOAA NWPS API           | River gauge data (stage, flood thresholds, forecasts)          | Free / Public               |
| ND DOT / USGS           | Traffic camera and river camera image sources                  | Free / Public               |
| UptimeRobot             | Uptime monitoring for all 8 Workers with email alerts          | Free tier                   |

# 2. Account & Access Guide

## 2.1 Accounts and Where to Find Them

| **Account**   | **URL / Login**                                | **What It Controls**                                                    |
|---------------|------------------------------------------------|-------------------------------------------------------------------------|
| Cloudflare    | dash.cloudflare.com – bwehner                  | Worker deployment, subdomain (bwehner), secrets storage                 |
| GitHub        | github.com/wehnerb                             | All source code repositories and deployment workflows                   |
| Google Cloud  | console.cloud.google.com — bwehner@fargond.gov | Google Slides, Sheets, and Drive API access and service account credentials |
| AirNow        | docs.airnowapi.org — bwehner@fargond.gov       | AirNow API key for weather display AQI data                             |
| UptimeRobot   | uptimerobot.com — bwehner@fargond.gov          | Uptime monitoring for all 8 Workers, email alerts on downtime           |

## 2.2 GitHub Repositories

| **Repository**                      | **Worker URL**                                        | **Purpose**                                                   |
|-------------------------------------|-------------------------------------------------------|---------------------------------------------------------------|
| station-image-proxy                 | station-image-proxy.bwehner.workers.dev               | Image resizing proxy for traffic camera and river feeds       |
| slide-timing-proxy                  | slide-timing-proxy.bwehner.workers.dev                | Dynamic Google Slides per-slide timing                        |
| river-level-display                 | river-level-display.bwehner.workers.dev               | River gauge hydrograph display (NOAA NWPS data)               |
| daily-message-display               | daily-message-display.bwehner.workers.dev             | Daily safety message and image display                        |
| calendar-display                    | calendar-display.bwehner.workers.dev                  | Station calendar display from exported ICS file with weather  |
| probationary-firefighter-display    | probationary-firefighter-display.bwehner.workers.dev  | Rotating probationary firefighter spotlight display           |
| department-news-display             | department-news-display.bwehner.workers.dev           | Department-wide and station-specific news card display        |
| weather-display                     | weather-display.bwehner.workers.dev                   | Full weather display with radar, conditions, and alerts       |
| ffd-station-display-utils           | (shared library — no Worker URL)                      | Shared design tokens, helpers, and utilities for all Workers  |

## 2.3 GitHub Secrets Reference

Each GitHub repository stores secrets that are injected into the Cloudflare Worker at deployment time. These are never exposed in code and must be set per repository under Settings → Secrets and variables → Actions.

| **Secret Name**              | **Repository**                          | **Description**                                                                                                                                                                                               |
|------------------------------|-----------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| CLOUDFLARE_API_TOKEN         | All                                     | Cloudflare API token with Workers edit permissions                                                                                                                                                            |
| CLOUDFLARE_ACCOUNT_ID        | All                                     | Cloudflare account ID (found on any zone page in dashboard)                                                                                                                                                   |
| GOOGLE_SERVICE_ACCOUNT_EMAIL | slide-timing-proxy only (GitHub)        | Service account email from Google Cloud JSON key file. Also required for daily-message-display, probationary-firefighter-display, department-news-display — set in Cloudflare dashboard for those workers.    |
| GOOGLE_PRIVATE_KEY           | slide-timing-proxy only (GitHub)        | Private key from Google Cloud JSON key file (include \n characters). Also required for daily-message-display, probationary-firefighter-display, department-news-display — set in Cloudflare dashboard.         |
| GOOGLE_SHEET_ID              | daily-message-display, department-news-display, probationary-firefighter-display | The ID of the relevant Google Sheet. Found in the Sheet URL between /d/ and /edit. Set in Cloudflare dashboard for these workers. |
| GOOGLE_DRIVE_FOLDER_ID       | daily-message-display, probationary-firefighter-display | The ID of the Google Drive folder containing images. Found in the folder URL after /folders/. Set in Cloudflare dashboard. |
| PRESENTATION_ID              | slide-timing-proxy only                 | Google Slides presentation ID — the alphanumeric string between /d/ and /edit in the presentation URL.                                                                                                        |
| PUBLISHED_ID                 | slide-timing-proxy only                 | Google Slides published embed ID — the long string between /d/e/ and /pubembed in the File → Share → Publish to web embed URL.                                                                                |
| NEXTCLOUD_URL                | calendar-display only                   | Full WebDAV URL to the ICS file on Nextcloud. Format: https://fileshare.fargond.gov/remote.php/dav/files/USERNAME/FFD%20Calendar%20Export/FFD%20Calendar%20Calendar.ics                                       |
| NEXTCLOUD_USERNAME           | calendar-display only                   | Nextcloud login username (shown when creating an app password — not the display name).                                                                                                                        |
| NEXTCLOUD_PASSWORD           | calendar-display only                   | Nextcloud app password. Generate at: Nextcloud → Settings → Security → Devices & sessions → Create new app password.                                                                                          |
| AIRNOW_API_KEY               | weather-display only                    | AirNow API key for AQI data. Set in Cloudflare dashboard (not GitHub Actions). Register at docs.airnowapi.org. Free tier is sufficient.                                                                       |

| **Where to Find These Secret Values**                                                                                                                                                                                                                                                                                                                                                                                                                              |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| CLOUDFLARE_API_TOKEN: Log in to dash.cloudflare.com, go to My Profile → API Tokens → Create Token. Use the "Edit Cloudflare Workers" template. Copy the generated token — it is only shown once.                                                                                                                                                                                                                                                                   |
| CLOUDFLARE_ACCOUNT_ID: Log in to dash.cloudflare.com and go to Workers & Pages → Overview. The Account ID is displayed in the right sidebar.                                                                                                                                                                                                                                                                                                                       |
| GOOGLE_SERVICE_ACCOUNT_EMAIL and GOOGLE_PRIVATE_KEY: Log in to console.cloud.google.com and open the slide-timing project. Go to IAM & Admin → Service Accounts → slide-timing-worker → Keys → Add Key → Create new key (JSON). Download the JSON file. The client_email field is your GOOGLE_SERVICE_ACCOUNT_EMAIL and the private_key field is your GOOGLE_PRIVATE_KEY. Store the JSON file securely and delete it after copying the values into the Cloudflare dashboard. |
| Note: river-level-display does not require any Google credentials. daily-message-display, probationary-firefighter-display, and department-news-display all use the same service account as slide-timing-proxy — set GOOGLE_SERVICE_ACCOUNT_EMAIL and GOOGLE_PRIVATE_KEY in the Cloudflare dashboard (Workers & Pages → [Worker Name] → Settings → Variables and Secrets) for each of those three workers. calendar-display uses Nextcloud WebDAV and does not require Google credentials. weather-display requires AIRNOW_API_KEY set in the Cloudflare dashboard. |

| **Important — Cloudflare Worker Secrets**                                                                                                                                                                                                                                                      |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| After the first deployment of a new Worker environment (e.g. staging), the secrets must also be set in the Cloudflare dashboard under Workers & Pages → \[Worker Name\] → Settings → Variables and Secrets. GitHub Actions deploys the code but does not set secrets — they must be configured in the Cloudflare dashboard manually. |

## 2.4 Transferring Ownership to a New Administrator

If a new person takes over maintenance of this system, the following steps must be completed to transfer full control. Each step must be completed in order.

### Step 1 — GitHub

1.  Create a GitHub account for the new administrator if they do not have one.

2.  Go to github.com/wehnerb and transfer ownership of each repository to the new account under Settings → Danger Zone → Transfer ownership. Alternatively, add the new person as an owner of the organization.

3.  The new administrator must re-add all GitHub Secrets listed in Section 2.3 to each repository under their account.

### Step 2 — Cloudflare

1.  Log in to dash.cloudflare.com and go to Manage Account → Members.

2.  Add the new administrator's email and assign the Administrator role.

3.  Create a new API token for the new administrator using the Edit Cloudflare Workers template and provide it to them for their GitHub secrets.

4.  In the Cloudflare dashboard, set the following secrets for each Worker that requires them (under Workers & Pages → [Worker Name] → Settings → Variables and Secrets): GOOGLE_SERVICE_ACCOUNT_EMAIL and GOOGLE_PRIVATE_KEY for daily-message-display, probationary-firefighter-display, and department-news-display; GOOGLE_SHEET_ID and GOOGLE_DRIVE_FOLDER_ID for daily-message-display and probationary-firefighter-display; GOOGLE_SHEET_ID for department-news-display; AIRNOW_API_KEY for weather-display; NEXTCLOUD_URL, NEXTCLOUD_USERNAME, NEXTCLOUD_PASSWORD for calendar-display.

### Step 3 — Google Cloud

1.  Go to console.cloud.google.com and open the slide-timing project.

2.  Go to IAM & Admin → IAM and add the new administrator's Google account with the Owner role.

3.  The new administrator should create a new service account key (IAM & Admin → Service Accounts → slide-timing-worker → Keys → Add Key). The old key should then be deleted.

4.  Update GOOGLE_SERVICE_ACCOUNT_EMAIL and GOOGLE_PRIVATE_KEY in the Cloudflare dashboard for all workers that require those secrets.

### Step 4 — AirNow

1. Go to docs.airnowapi.org and register for a new API key using the new administrator's email.

2. Update AIRNOW_API_KEY in the Cloudflare dashboard for weather-display and weather-display-staging.

### Step 5 — UptimeRobot

1. Go to uptimerobot.com and create a new account for the new administrator.

2. Recreate all 8 monitors (see Section 13.4) under the new account.

3. Set the alert contact to the new administrator's department email.

*Note: river-level-display has no Google Cloud dependency. calendar-display uses Nextcloud WebDAV and has no Google Cloud dependency — the new administrator only needs to update the NEXTCLOUD_URL, NEXTCLOUD_USERNAME, and NEXTCLOUD_PASSWORD secrets in the Cloudflare dashboard.*

# 3. Project: Image Resizing Proxy

## 3.1 Purpose & Problem Solved

The station display system cannot resize images natively. Without intervention, images either fail to fill a display column entirely or overflow beyond its boundaries.

The Image Proxy Worker solves this by returning a self-contained HTML page rather than a raw image. Each page contains the source image URL and CSS that scales the image to fit the display column using object-fit: contain. Because the browser on the display hardware handles all scaling locally, no external image processing service is required.

Image freshness is handled entirely by the display hardware. Source image URLs for ND DOT cameras, USGS, and NOAA are designed to always return the latest image when loaded without additional query parameters. The display browser requests a fresh copy of each source image each time the page is rendered.

## 3.2 Repository & Deployment

| **Item**          | **Value**                                                |
|-------------------|----------------------------------------------------------|
| GitHub Repository | github.com/wehnerb/station-image-proxy                   |
| Production URL    | https://station-image-proxy.bwehner.workers.dev/         |
| Staging URL       | https://station-image-proxy-staging.bwehner.workers.dev/ |
| Worker File       | src/index.js                                           |
| Secrets Required  | CLOUDFLARE_API_TOKEN, CLOUDFLARE_ACCOUNT_ID              |

## 3.3 How It Works

When a display screen loads an image URL, the request goes to the Cloudflare Worker. The Worker reads the URL parameters to determine which image to display and what layout dimensions to use. It returns a self-contained HTML page containing the source image URL and CSS rules that scale the image to fill the slot using object-fit: contain with a transparent background.

The display browser fetches the source image directly from its origin server (ND DOT, USGS, or NOAA) and handles all scaling locally. No external image processing service is involved.

Cache-Control: no-store is set on all HTML responses so the display browser always requests a fresh copy of the page rather than serving a locally cached version.

## 3.4 URL Parameters

| **Parameter** | **Default** | **Options**          | **Description**                                                                               |
|---------------|-------------|----------------------|-----------------------------------------------------------------------------------------------|
| img           | (required)  | Any key from MAPPING | The image key name to display. Combine two keys with + for stacking.                          |
| layout        | split       | wide, split, tri     | Column width: 1-column (wide), 2-column (split), 3-column (tri). Default is split (2-column). |


## 3.5 Layout Dimensions

| **Layout Key** | **Width (px)** | **Height (px)** | **Use Case**                     |
|----------------|----------------|-----------------|----------------------------------|
| wide           | 1735           | 720             | Full-width single column display |
| split          | 852            | 720             | Two-column display (default)     |
| tri            | 558            | 720             | Three-column display             |
| full           | 1920           | 1075            | Full-screen display              |

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

All images are defined in the MAPPING object near the top of worker_code.js. To add a new camera or data image:

1.  Open the staging branch of the station-image-proxy repository in GitHub.

2.  Edit src/index.js and locate the MAPPING object.

3.  Add a new line inside the MAPPING object following the existing format:

`"your-key-name": "https://full-url-to-the-source-image.jpg",`

4.  Key names should be lowercase with hyphens — for example i29MainAve-north for an I-29 camera at Main Ave facing north.

5.  Commit the change to staging and test the new key using the staging Worker URL before merging to main.

## 3.9 Current Image Library

The following images are currently configured in the MAPPING object. See Section 3.8 for instructions on adding new images.

### NOAA River Gauges

| **Key Name**        | **Readable Name**            | **Source URL**                                            |
|---------------------|------------------------------|-----------------------------------------------------------|
| riverlevel-redriver | Red River Level Gauge (NOAA) | https://water.noaa.gov/resources/hydrographs/fgon8_hg.png |

### USGS River Images

| **Key Name**   | **Readable Name**                | **Source URL**                                                                                                                     |
|----------------|----------------------------------|------------------------------------------------------------------------------------------------------------------------------------|
| river-redriver | Red River at Fargo (USGS Camera) | https://usgs-nims-images.s3.amazonaws.com/overlay/ND_Red_River_of_the_North_at_Fargo/ND_Red_River_of_the_North_at_Fargo_newest.jpg |

### ND DOT Cameras — I-94

| **Key Name**          | **Readable Name**                     | **Source URL**                                                                   |
|-----------------------|---------------------------------------|----------------------------------------------------------------------------------|
| i94VeteransBlvd-south | I-94 at Veterans Blvd (Facing South)  | https://www.dot.nd.gov/travel-info/cameras/I94@347.601Fargo9thStEWBSouth.jpg     |
| i94VeteransBlvd-east  | I-94 at Veterans Blvd (Facing East)   | https://www.dot.nd.gov/travel-info/cameras/I94RP347.565Fargo9thStEEBEast.jpg     |
| i9445thStS-south      | I-94 at 45th St S (Facing South)      | https://www.dot.nd.gov/travel-info/cameras/I94RP348.602Fargo45thStSouth.jpg      |
| i9445thStS-east       | I-94 at 45th St S (Facing East)       | https://www.dot.nd.gov/travel-info/cameras/I94RP348.602Fargo45thStEast.jpg       |
| i9445thStS-west       | I-94 at 45th St S (Facing West)       | https://www.dot.nd.gov/travel-info/cameras/I94RP348.602Fargo45thStWest.jpg       |
| i9442ndStS-east       | I-94 at 42nd St S (Facing East)       | https://www.dot.nd.gov/travel-info/cameras/I94RP349.145Fargo42ndStSWBEast.jpg    |
| i9442ndStS-west       | I-94 at 42nd St S (Facing West)       | https://www.dot.nd.gov/travel-info/cameras/I94RP349.145Fargo42ndStSWBWest.jpg    |
| i29i94-north          | I-29/I-94 Interchange (Facing North)  | https://www.dot.nd.gov/travel-info/cameras/fargotrilevelnorth.jpg                |
| i29i94-south          | I-29/I-94 Interchange (Facing South)  | https://www.dot.nd.gov/travel-info/cameras/fargotrilevelsouth.jpg                |
| i94i29-east           | I-94/I-29 Interchange (Facing East)   | https://www.dot.nd.gov/travel-info/cameras/fargotrileveleast.jpg                 |
| i94i29-west           | I-94/I-29 Interchange (Facing West)   | https://www.dot.nd.gov/travel-info/cameras/fargotrilevelwest.jpg                 |
| i9425thStS-east       | I-94 at 25th St S (Facing East)       | https://www.dot.nd.gov/travel-info/cameras/I94RP350.611Fargo25thStEast.jpg       |
| i9425thStS-west       | I-94 at 25th St S (Facing West)       | https://www.dot.nd.gov/travel-info/cameras/I94RP350.603Fargo25thStWest.jpg       |
| i94UniversityDrS-east | I-94 at University Dr S (Facing East) | https://www.dot.nd.gov/travel-info/cameras/I94RP351.617FargoUniversityDrEast.jpg |
| i94UniversityDrS-west | I-94 at University Dr S (Facing West) | https://www.dot.nd.gov/travel-info/cameras/I94RP351.617FargoUniversityDrWest.jpg |
| i94RedRiver-east      | I-94 at Red River (Facing East)       | https://www.dot.nd.gov/travel-info/cameras/I94RP352FargoRedRiverEast.jpg         |
| i94RedRiver-west      | I-94 at Red River (Facing West)       | https://www.dot.nd.gov/travel-info/cameras/I94RP352FargoRedRiverWest.jpg         |

### ND DOT Cameras — I-29

| **Key Name**      | **Readable Name**                 | **Source URL**                                                                       |
|-------------------|-----------------------------------|--------------------------------------------------------------------------------------|
| i2970thAveS-north | I-29 at 70th Ave S (Facing North) | https://www.dot.nd.gov/travel-info/cameras/I29RP58.765FargoSouthof64thAveSNorth.jpg  |
| i2970thAveS-south | I-29 at 70th Ave S (Facing South) | https://www.dot.nd.gov/travel-info/cameras/I29RP58.765FargoSouthof64thAveSSouth.jpg  |
| i2952ndAveS-north | I-29 at 52nd Ave S (Facing North) | https://www.dot.nd.gov/travel-info/cameras/I29RP60.293Fargo52ndAveSSBNorth.jpg       |
| i2952ndAveS-south | I-29 at 52nd Ave S (Facing South) | https://www.dot.nd.gov/travel-info/cameras/I29RP60.293Fargo52ndAveSSBSouth.jpg       |
| i2952ndAveS-east  | I-29 at 52nd Ave S (Facing East)  | https://www.dot.nd.gov/travel-info/cameras/I29RP60.293Fargo52ndAveSSBEast.jpg        |
| i2952ndAveS-west  | I-29 at 52nd Ave S (Facing West)  | https://www.dot.nd.gov/travel-info/cameras/I29RP60.293Fargo52ndAveSSBWest.jpg        |
| i2940thAveS-north | I-29 at 40th Ave S (Facing North) | https://www.dot.nd.gov/travel-info/cameras/I29RP61.408Fargo40thAveNorth.jpg          |
| i2940thAveS-south | I-29 at 40th Ave S (Facing South) | https://www.dot.nd.gov/travel-info/cameras/I29RP61.408Fargo40thAveSouth.jpg          |
| i2932ndAveS-north | I-29 at 32nd Ave S (Facing North) | https://www.dot.nd.gov/travel-info/cameras/I29RP62.627Fargo32ndAveSEastRampNorth.jpg |
| i2932ndAveS-south | I-29 at 32nd Ave S (Facing South) | https://www.dot.nd.gov/travel-info/cameras/I29RP62.627Fargo32ndAveSEastRampSouth.jpg |
| i2932ndAveS-west  | I-29 at 32nd Ave S (Facing West)  | https://www.dot.nd.gov/travel-info/cameras/I29RP62.627Fargo32ndAveSEastRampWest.jpg  |
| i2913thAveS-north | I-29 at 13th Ave S (Facing North) | https://www.dot.nd.gov/travel-info/cameras/I29RP64.725FargoNorthof9thAveNorth.jpg    |
| i2913thAveS-south | I-29 at 13th Ave S (Facing South) | https://www.dot.nd.gov/travel-info/cameras/I29RP64.135Fargo13thAveSSouth.jpg         |
| i29MainAve-north  | I-29 at Main Ave (Facing North)   | https://www.dot.nd.gov/travel-info/cameras/I29RP65.272FargoMainAveSBNorth.jpg        |
| i29MainAve-south  | I-29 at Main Ave (Facing South)   | https://www.dot.nd.gov/travel-info/cameras/I29RP65.272FargoMainAveSBSouth.jpg        |
| i297thAveN-south  | I-29 at 7th Ave N (Facing South)  | https://www.dot.nd.gov/travel-info/cameras/I29RP65.741Fargo7thAveNSouth.jpg          |
| i2919thAveN-north | I-29 at 19th Ave N (Facing North) | https://www.dot.nd.gov/travel-info/cameras/I29RP66.894Fargo19thAveNNorth.jpg         |
| i2919thAveN-south | I-29 at 19th Ave N (Facing South) | https://www.dot.nd.gov/travel-info/cameras/I29RP66.894Fargo19thAveNSouth.jpg         |
| i2919thAveN-east  | I-29 at 19th Ave N (Facing East)  | https://www.dot.nd.gov/travel-info/cameras/I29RP67.241Fargo19thAveNEastRampEast.jpg  |
| i2919thAveN-west  | I-29 at 19th Ave N (Facing West)  | https://www.dot.nd.gov/travel-info/cameras/I29RP67.241Fargo19thAveNEastRampWest.jpg  |

# 4. Project: Slide Timing Proxy

## 4.1 Purpose & Problem Solved

Display screens are configured with a fixed time slot for showing a Google Slides presentation — for example, 60 seconds. However, the number of slides in the presentation changes over time. If the presentation has 6 slides and each is displayed for 10 seconds, the full 60 seconds is used. If there are only 2 slides but each is still set to 10 seconds, only 20 of the 60 seconds is used before it loops.

The Slide Timing Proxy Worker solves this by dynamically calculating the correct per-slide timing at request time. Each time the display loads the Worker URL, the Worker queries the Google Slides API to count the current number of slides, divides the total allotted time equally, and redirects the display directly to the Google Slides published embed URL with the correct timing parameter already set.

## 4.2 Repository & Deployment

| **Item**          | **Value**                                                                                     |
|-------------------|-----------------------------------------------------------------------------------------------|
| GitHub Repository | github.com/wehnerb/slide-timing-proxy                                                         |
| Production URL    | https://slide-timing-proxy.bwehner.workers.dev/                                               |
| Staging URL       | https://slide-timing-proxy-staging.bwehner.workers.dev/                                       |
| Worker File       | src/index.js                                                                                  |
| Secrets Required  | CLOUDFLARE_API_TOKEN, CLOUDFLARE_ACCOUNT_ID, GOOGLE_SERVICE_ACCOUNT_EMAIL, GOOGLE_PRIVATE_KEY |

## 4.3 Example URL

All stations use the same Worker URL with no additional parameters:

`https://slide-timing-proxy.bwehner.workers.dev/`

## 4.4 How It Works

1.  The Worker authenticates with the Google Slides API using a service account, generating a short-lived OAuth2 access token using the RSA-signed JWT method via Cloudflare's built-in Web Crypto API.

2.  The Worker checks the Workers Cache API for a previously stored slide count. If a cached value exists and the SLIDE_CACHE_VERSION matches, the cached count is used and the Google API is not called.

3.  On a cache miss, the Worker calls the Slides API to retrieve the current number of slides and stores the result in the cache for SLIDE_CACHE_SECONDS (default 1 hour).

4.  The per-slide delay is calculated: total seconds divided by slide count, clamped to the configured minimum.

5.  The Worker issues a 302 redirect directly to the Google Slides published embed URL (pubembed format) with the calculated delayms parameter. The presentation always starts from slide 1 on every fresh load.

## 4.5 Timing Logic

| **Slide Count** | **Total Seconds** | **Per-Slide Delay** | **Notes**                                         |
|-----------------|-------------------|---------------------|---------------------------------------------------|
| 0               | 60                | N/A                 | No-content screen shown, auto-refreshes every 60s |
| 1               | 60                | 60s                 | Single slide uses full allotted time              |
| 2               | 60                | 30s                 | Normal equal division                             |
| 6               | 60                | 10s                 | Normal equal division                             |
| 12              | 60                | 5s                  | Hits minimum cap (MIN_SECONDS = 5)                |
| API fails       | 60                | 60s                 | Safe fallback: slideCount defaults to 1           |

## 4.6 Configuration

| **Constant**                     | **Description**                                                                                                                                                                                                                                                                                                                                                                                                  |
|----------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| PRESENTATION_ID and PUBLISHED_ID | Stored as Cloudflare Worker secrets and injected at runtime via the env object. PRESENTATION_ID is the alphanumeric ID between /d/ and /edit in the Google Slides URL. PUBLISHED_ID is the long string between /d/e/ and /pubembed in the File → Share → Publish to web embed URL. |
| TOTAL_SECONDS                    | Total seconds the display system allocates to the slideshow slot.                                                                                                                                                                                                                                                                                                                                                |
| MIN_SECONDS                      | Minimum seconds per slide regardless of how many slides exist.                                                                                                                                                                                                                                                                                                                                                   |
| SLIDE_CACHE_SECONDS              | How long (seconds) the slide count is cached using the Workers Cache API. Default is 3600 (1 hour).                                                                                                                                                                                                                                                                                                              |
| SLIDE_CACHE_VERSION              | Integer cache-buster. Increment by 1 to immediately invalidate the cached slide count.                                                                                                                                                                                                                                                                                                                           |

## 4.7 No-Content Screen

If the presentation has zero slides, the Worker returns a styled HTML page instead of redirecting. This page displays a dark screen with the message NO CONTENT AVAILABLE and automatically refreshes every 60 seconds.

## 4.8 Google Service Account

The Google Slides API requires a service account for authentication. The service account credentials are stored as GitHub secrets and injected into the Worker at deployment time. The service account only has read-only access to the Slides API scope.

| **Security Note**                                                                                                                                                                                                                                                                                                                                                                                                     |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| The service account private key (GOOGLE_PRIVATE_KEY) is a sensitive credential. It is stored only in GitHub Secrets and Cloudflare Worker secrets — never in the code itself. If a key is ever suspected to be compromised, it should be deleted in Google Cloud Console (IAM & Admin → Service Accounts → Keys) and a new one generated immediately. |

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

| **Item**          | **Value**                                                |
|-------------------|----------------------------------------------------------|
| GitHub Repository | github.com/wehnerb/river-level-display                   |
| Production URL    | https://river-level-display.bwehner.workers.dev/         |
| Staging URL       | https://river-level-display-staging.bwehner.workers.dev/ |
| Worker File       | src/index.js                                             |
| Secrets Required  | CLOUDFLARE_API_TOKEN, CLOUDFLARE_ACCOUNT_ID              |

## 5.3 URL Parameters

| **Parameter** | **Default** | **Options**                  | **Description**                                                         |
|---------------|-------------|------------------------------|-------------------------------------------------------------------------|
| gauge         | fargo       | Any key from GAUGES registry | Which river gauge to display. See Section 5.7 for the current registry. |
| layout        | split       | wide, split, tri, full       | Column width matching display hardware.                                  |

## 5.4 How It Works

1.  The Worker makes two parallel requests to the NOAA NWPS public API: one for gauge metadata (name, flood thresholds) and one for observed and forecast stage data. No authentication is required.

2.  The Worker processes the response: it trims observed history to the configured window, combines observed and forecast data, detects a crest if the river has peaked, and determines the current flood status.

3.  All processed data is injected as a JSON literal into a self-contained HTML page. No further API calls are made by the display browser.

4.  The browser renders the hydrograph on a canvas element using the injected data.

5.  The page auto-refreshes every 15 minutes via a meta refresh tag.

If the NOAA API is unreachable or returns an error, the Worker returns a styled error page that automatically retries every 60 seconds.

## 5.5 Configuration

| **Constant**            | **Description**                                                                                                                                                               |
|-------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| GAUGES                  | Registry mapping URL-friendly keys to NOAA gauge IDs and display names. Add new gauges here.                                                                                  |
| DEFAULT_GAUGE           | The gauge used when the ?gauge= parameter is omitted.                                                                                                                         |
| OBSERVED_HOURS          | How many hours of observed history to show on the chart (default: 120).                                                                                                       |
| CACHE_SECONDS           | How long Cloudflare edge-caches the NOAA API response (default: 900 = 15 minutes).                                                                                           |
| Y_AXIS_PADDING          | Feet of padding added above and below the data range on the Y axis.                                                                                                           |
| THRESHOLD_LOOKAHEAD_FT  | A flood threshold line is only shown when it falls within this many feet of the data maximum.                                                                                 |
| CREST_MIN_FLANK_POINTS  | Number of data points required on each side of the peak before a crest is confirmed.                                                                                         |
| CREST_MIN_PROMINENCE_FT | The peak must be at least this many feet above the average of the flanking data to be labeled as a crest.                                                                    |

## 5.6 Adding New Gauges

1.  Find the NOAA gauge ID at water.noaa.gov (e.g. FGON8 for Fargo, ND).
2.  Edit src/index.js on the staging branch and add a new entry to the GAUGES object: `'your-key': { id: 'NWSID', name: 'Human-readable name' },`
3.  Test using the staging URL with ?gauge=your-key before merging to main.

## 5.7 Current Gauge Registry

| **Key (?gauge=)** | **NOAA Gauge ID** | **Display Name**   |
|-------------------|-------------------|--------------------|
| fargo             | FGON8             | Red River at Fargo |

# 6. Project: Daily Message Display

## 6.1 Purpose & Problem Solved

Station displays previously had no mechanism for showing rotating safety messages, quotes, or imagery. The Daily Message Display Worker provides a dedicated full-screen display that rotates content every 3 days — aligned to the department's 9-day shift rotation — ensuring all three shifts see every message before it advances. Content is managed by shift officers without any technical knowledge required.

## 6.2 Repository & Deployment

| **Item**          | **Value**                                                                                                                              |
|-------------------|----------------------------------------------------------------------------------------------------------------------------------------|
| GitHub Repository | github.com/wehnerb/daily-message-display                                                                                               |
| Production URL    | https://daily-message-display.bwehner.workers.dev/                                                                                     |
| Staging URL       | https://daily-message-display-staging.bwehner.workers.dev/                                                                             |
| Worker File       | src/index.js                                                                                                                           |
| Secrets Required  | CLOUDFLARE_API_TOKEN, CLOUDFLARE_ACCOUNT_ID (GitHub); GOOGLE_SERVICE_ACCOUNT_EMAIL, GOOGLE_PRIVATE_KEY, GOOGLE_SHEET_ID, GOOGLE_DRIVE_FOLDER_ID (Cloudflare dashboard) |

## 6.3 URL Parameters

The ?layout= parameter accepts: full (1920x1075, default), wide (1735x720), split (852x720), tri (558x720). The Daily Safety Message title label is only shown in the full layout.

## 6.4 How It Works

1. The Worker authenticates with Google using the shared service account, generating a short-lived OAuth2 access token.

2. The Worker fetches the Messages tab from Google Sheets and lists image files in the Google Drive folder in parallel.

3. Date override entries are checked first. If today's date matches a pinned image filename prefix or a sheet row's Date column, that entry is selected. Images take priority over text if both are pinned to the same date.

4. If no date override matches, a combined rotation pool is built by interleaving active text entries and image files evenly. The pool index is: floor(daysElapsed / ROTATION_DAYS) % poolSize, anchored to January 23, 2026 in America/Chicago time.

5. For image entries, the Worker locates the file in Google Drive and serves it via a streaming proxy route (/image/{fileId}). The display browser requests the image from the Worker, which fetches it from Drive server-side using the service account token and streams the bytes directly to the browser. No base64 encoding is performed — this keeps CPU usage low and avoids Cloudflare's 10ms CPU limit even for large images.

6. A self-contained HTML page is returned. The meta-refresh interval is set to the exact seconds until the next 7:30 AM Central rotation.

## 6.5 Rotation Logic

Messages rotate every ROTATION_DAYS calendar days (default: 3). With 3-day blocks aligned to the 9-day shift rotation, each message is seen by all three shifts before advancing. Day boundaries use America/Chicago time via Intl.DateTimeFormat, so DST transitions are handled correctly.

## 6.6 Configuration

Key constants in src/index.js: ROTATION_DAYS (default 3), ROTATION_ANCHOR (2026-01-23), IMAGE_SOURCE ("drive" or "network"), DEFAULT_LAYOUT ("full").

## 6.7 Content Management

**Text messages:** Add rows to the Messages tab of the Google Sheet "Fire Station Display — Daily Messages". Set Active to "yes". Date and Attribution are optional.

**Images:** Drop image files into the "Fire Station Display — Daily Images" Drive folder. Images under 2 MB are recommended for best display performance — larger files will work but may cause a noticeable slight delay when the page first loads on hardware. Move files into any subfolder to deactivate.

**Date pinning:** For text, enter YYYY-MM-DD in the Date column. For images, prefix the filename: YYYY-MM-DD-filename.jpg. Images win if both sources are pinned to the same date.

## 6.8 Google Sheet Columns

Required columns: Content, Active. Optional columns: Date, Attribution. The header row is protected with an edit warning. Columns are read by header name so they can be reordered without code changes.

## 6.9 Network Share (Future Use)

The Worker includes a stubbed code path for an internal network share. To activate: set IMAGE_SOURCE = "network" in src/index.js and add secrets NETWORK_SHARE_URL, NETWORK_SHARE_USERNAME, and NETWORK_SHARE_PASSWORD to Cloudflare Worker settings.

# 7. Project: Calendar Display

## 7.1 Purpose & Problem Solved

The station display system did not previously have a way to display the department's calendar or current weather conditions. The Calendar Display Worker fetches the FFD Calendar ICS file from Nextcloud via WebDAV and renders it as a styled HTML calendar page. For wide and full layouts, the Worker also fetches live NWS weather data — daily high/low temperatures, conditions, and wind for each day, an hourly forecast strip for today, and active/upcoming weather alert banners and badges. No technical knowledge is required to keep the calendar current.

## 7.2 Repository & Deployment

| **Item**          | **Value**                                                                                          |
|-------------------|----------------------------------------------------------------------------------------------------|
| GitHub Repository | github.com/wehnerb/calendar-display                                                                |
| Production URL    | https://calendar-display.bwehner.workers.dev/                                                      |
| Staging URL       | https://calendar-display-staging.bwehner.workers.dev/                                              |
| Worker File       | src/index.js                                                                                       |
| Secrets Required  | CLOUDFLARE_API_TOKEN, CLOUDFLARE_ACCOUNT_ID, NEXTCLOUD_URL, NEXTCLOUD_USERNAME, NEXTCLOUD_PASSWORD |

## 7.3 URL Parameters

The ?layout= parameter controls which design is rendered. wide and full use the split view (today detail on the left, next days on the right) and include NWS weather data. split and tri use the strip view (compact upcoming list) with no weather data. Default is wide.

## 7.4 How It Works

1. The Worker checks the Workers Cache API for a previously rendered page matching the requested layout. If a valid cached response exists it is returned immediately.

2. On a cache miss, data is fetched. For split and tri layouts, only the ICS file is fetched. For wide and full layouts, the ICS file and all NWS endpoints (daily forecast, hourly forecast, active alerts) are fetched in parallel.

3. The raw ICS text is fetched server-side from Nextcloud using HTTP Basic authentication with a Nextcloud app password. The display browser never contacts Nextcloud directly. Windows timezone names emitted by Exchange (e.g. "Central Standard Time") are automatically mapped to IANA timezone identifiers.

4. Filter rules are applied, a self-contained HTML page is rendered, stored in the Workers Cache API for CACHE_SECONDS (default 15 minutes), and returned to the display.

## 7.5 Automatic Calendar Update System

The calendar is kept current by a three-component system that runs automatically at login on the designated department computer:

- **Outlook VBA macro** (ThisOutlookSession in the Outlook VBA editor): Runs automatically when Outlook opens. Exports the next 30 days of the FFD Calendar public folder to U:\Fire\BWehner\FFD Calendar Export\FFD Calendar Calendar.ics.

- **Nextcloud desktop app**: Syncs the FFD Calendar Export folder to Nextcloud automatically. The file is typically synced within seconds of the macro writing it.

Full setup instructions for rebuilding this system on a new computer are in U:\Fire\BWehner\FFD Calendar Export\FFD Calendar Export Setup.txt.

## 7.6 Configuration

Key constants in src/index.js: DAYS_TO_SHOW (default 6), CACHE_SECONDS (default 900 = 15 minutes), CACHE_VERSION (increment to bust all cached pages immediately), FILTER_EXACT, FILTER_CONTAINS, ALLDAY_COLORS. NWS constants: NWS_OFFICE (FGF), NWS_GRID_X (65), NWS_GRID_Y (57), NWS_ALERT_ZONE (NDZ039 — Cass County ND), SHOW_WEATHER (boolean to enable/disable all weather fetches).

## 7.7 NWS Weather Display

Weather data is fetched from api.weather.gov with no API key required. All NWS fetches are edge-cached separately from the page cache.

**Daily forecast:** Every column header shows the high temperature (orange), low temperature (blue), a condition icon and short forecast text, and wind direction and speed.

**Hourly forecast strip:** The today panel body shows remaining hours at configurable intervals. Each slot shows the hour label, temperature, and an inline SVG weather icon.

**Alert banners:** Active NWS alerts for Cass County (NDZ039) appear as full-width colored banners above the calendar panels, sorted by severity. Red = Warning, Orange = Watch, Yellow = Advisory. Each banner shows the alert name and when it ends.

**Alert badges:** Upcoming alerts not yet active appear as severity-colored pills in affected day column headers.

## 7.8 Event Filtering & All-Day Colors

FILTER_EXACT and FILTER_CONTAINS control which events are hidden. ALLDAY_COLORS assigns custom colors per event title. Current shift colors: A Shift = dark green, B Shift = off-white with dark text, C Shift = dark red.

## 7.9 Manual Calendar Update

To update the calendar outside of a normal login: open Outlook, open the VBA editor (Developer tab → Visual Basic), click anywhere inside Application_Startup, and press F5. The Nextcloud desktop app will sync the updated file automatically within seconds.

To force an immediate cache refresh: increment CACHE_VERSION in src/index.js by 1, deploy to staging, test, and merge to main.

# 8. Project: Probationary Firefighter Display

## 8.1 Purpose & Problem Solved

The Probationary Firefighter Display Worker automates the introduction of new hires department-wide by reading firefighter bio data from a Google Sheet and photos from a Google Drive folder, then rendering a rotating spotlight page that cycles through every active new hire on the same 3-day rotation used by the Daily Message Display. Firefighters are considered active for 365 days from their hire date and drop off automatically with no manual action required.

## 8.2 Repository & Deployment

| **Item**          | **Value**                                                                                                                                                                     |
|-------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| GitHub Repository | github.com/wehnerb/probationary-firefighter-display                                                                                                                           |
| Production URL    | https://probationary-firefighter-display.bwehner.workers.dev/                                                                                                                 |
| Staging URL       | https://probationary-firefighter-display-staging.bwehner.workers.dev/                                                                                                         |
| Worker File       | src/index.js                                                                                                                                                                  |
| Secrets Required  | CLOUDFLARE_API_TOKEN, CLOUDFLARE_ACCOUNT_ID (GitHub); GOOGLE_SERVICE_ACCOUNT_EMAIL, GOOGLE_PRIVATE_KEY, GOOGLE_SHEET_ID, GOOGLE_DRIVE_FOLDER_ID (Cloudflare dashboard) |

## 8.3 URL Parameters

The ?layout= parameter controls which design is rendered. Default is wide. The full layout adds a "PROBATIONARY FIREFIGHTER SPOTLIGHT" title bar at the top.

## 8.4 How It Works

1. The Worker authenticates with Google using the shared service account, generating a short-lived OAuth2 access token.

2. Firefighter records from the Google Sheet and the Drive photo file listing are fetched in parallel.

3. The active list is built: only firefighters whose hire date is within 365 days of today are included. The list is sorted by hire date ascending, then badge number ascending, for a stable consistent rotation order.

4. The current firefighter is selected using the same rotation block index as daily-message-display: floor(daysElapsed / ROTATION_DAYS) % activeCount, anchored to January 23, 2026 in America/Chicago time.

5. A self-contained HTML page is returned with the photo on the left and bio information on the right (wide/full layouts) or stacked (split/tri layouts). The photo is served via a streaming proxy route (/photo/{fileId}) — the display browser requests the photo from the Worker, which fetches it from Drive server-side using the service account token and streams the bytes directly to the browser. No base64 encoding is performed. The service account token never appears in any client-visible URL.

6. The meta-refresh interval is set to the exact seconds until the next 7:30 AM Central rotation boundary. No server-side caching is used — every request generates a fresh page to ensure the display always reflects the current rotation.

## 8.5 Rotation Logic

The rotation is identical to daily-message-display in all respects: 3-day blocks, anchored to January 23, 2026, advancing at 7:30 AM Central (DST-safe). After the last active firefighter the list loops back to the first. When a new firefighter is added or an existing one ages out, the active list is recalculated from scratch on the next request.

## 8.6 Data Sources

**Google Sheet:** Fire Station Display – Probationary Firefighter Information (tab: Firefighters). Both the sheet and the Drive folder must be shared with the service account email (Viewer permission).

**Google Drive folder:** Fire Station Display – Probationary Firefighter Images. Photos are named firstnamelastname.jpg (e.g. makenziekotchman.jpg). The filename entered in the Photo column of the sheet must match the Drive filename (case-insensitive).

## 8.7 Google Sheet Column Reference

| **Column Header**    | **Required?** | **Notes**                                                                                                                              |
|----------------------|---------------|----------------------------------------------------------------------------------------------------------------------------------------|
| Name                 | Yes           | Full name                                                                                                                              |
| Hire Date            | Yes           | Format: YYYY-MM-DD (e.g. 2026-01-19). Used to calculate active status.                                                                 |
| Rank                 | No            | e.g. Probationary Firefighter.                                                                                                        |
| Shift                | No            | A, B, C, Days, or Recruit Academy                                                                                                     |
| Badge                | No            | Plain badge number. Used for secondary sort within the same hire date.                                                                |
| Photo                | No            | Filename of photo in Drive folder (e.g. makenziekotchman.jpg). Match is case-insensitive.                                             |
| Hometown             | No            | Displayed in the fixed fields group alongside Hire Date, Shift, and Badge                                                             |
| Any other column     | No            | Treated as a dynamic Q&A pair. Header = question, cell = answer. Displayed below the fixed fields in column order.                    |

## 8.8 Configuration

| **Constant**    | **Description**                                                                                                              |
|-----------------|------------------------------------------------------------------------------------------------------------------------------|
| ROTATION_DAYS   | Consecutive days each firefighter displays before advancing (default 3). Must match daily-message-display.                   |
| ROTATION_ANCHOR | Anchor date for rotation (2026-01-23). Matches daily-message-display. Do not change unless intentionally resetting the cycle.|
| HIRE_ACTIVE_DAYS| Days after hire date a firefighter remains on the display (default 365). Firefighters drop off automatically.                |
| SHEET_TAB_NAME  | Name of the tab in the Google Sheet (default: Firefighters).                                                                 |

## 8.9 Adding a New Hire

1.  Add a new row to the Firefighters tab in the Google Sheet with at minimum Name and Hire Date filled in.

2.  Upload the firefighter's photo to the Drive folder named firstnamelastname.jpg. squoosh.app can be used to resize images, but larger images do load reliably.

3.  Enter the filename in the Photo column of the sheet. Fill in any other available fields. Blank fields are automatically omitted from the display.

4.  The Worker will pick up the changes on the next page load. No code change or redeployment is needed.

## 8.10 Adding a New Class with Different Questions

Add new column headers to row 1 of the sheet for the new questions. Blank cells are simply omitted — no code changes are required.

# 9. Project: Department News Display

## 9.1 Purpose & Problem Solved

Station displays previously had no way to show department-wide announcements or station-specific news in a structured, easy-to-manage format. The Department News Display Worker renders a card-based news feed sourced from a Google Sheet, with separate tabs for department-wide news and each individual fire station. Content is managed directly in the sheet by authorized personnel and appears on displays within minutes of being saved.

## 9.2 Repository & Deployment

| **Item**          | **Value**                                                                                                                    |
|-------------------|------------------------------------------------------------------------------------------------------------------------------|
| GitHub Repository | github.com/wehnerb/department-news-display                                                                                   |
| Production URL    | https://department-news-display.bwehner.workers.dev/                                                                         |
| Staging URL       | https://department-news-display-staging.bwehner.workers.dev/                                                                 |
| Worker File       | src/index.js                                                                                                                 |
| Secrets Required  | CLOUDFLARE_API_TOKEN, CLOUDFLARE_ACCOUNT_ID (GitHub); GOOGLE_SERVICE_ACCOUNT_EMAIL, GOOGLE_PRIVATE_KEY, GOOGLE_SHEET_ID (Cloudflare dashboard) |

## 9.3 URL Parameters

| **Parameter** | **Default** | **Options**                   | **Description**                                                        |
|---------------|-------------|-------------------------------|------------------------------------------------------------------------|
| station       | dept        | dept, 1, 2, 3, 4, 5, 6, 7, 8 | Which tab to display. dept = department-wide news; 1–8 = station news. |
| layout        | wide        | wide, split, tri, full        | Display column width.                                                  |

Example URLs:

`https://department-news-display.bwehner.workers.dev/?station=dept` — department-wide news

`https://department-news-display.bwehner.workers.dev/?station=7` — Station 7 news

## 9.4 How It Works

1. The Worker authenticates with Google using the shared service account and fetches the relevant sheet tab from the Google Sheet.

2. Rows are processed: expired items are hidden, recurring items are expanded based on their interval and stop-after date, and items are sorted with new items first (sorted by posted date descending), then regular items (sorted by posted date descending).

3. Items posted within NEW_ITEM_DAYS of today are highlighted with a "NEW" badge and a distinct card background.

4. A self-contained HTML page is returned with all active items displayed as cards. Each card shows the title, body text, and metadata (posted date, expires date, posted by). Line breaks entered in the sheet using Alt+Enter are preserved on the display.

5. If content overflows the visible area, a CSS keyframe animation automatically scrolls the content upward and loops continuously, allowing all items to be seen within the display cycle.

## 9.5 Google Sheet Structure

The Google Sheet contains one tab per feed: a "Department News" tab for department-wide news and tabs named "FS#1" through "FS#8" for each station. Both the sheet must be shared with the service account email (Viewer permission).

## 9.6 Google Sheet Column Reference

| **Column Header** | **Required?** | **Notes**                                                                                                                   |
|-------------------|---------------|-----------------------------------------------------------------------------------------------------------------------------|
| Title             | Yes           | Headline shown at the top of the card in bold                                                                               |
| Content           | Yes           | Body text of the news item. Alt+Enter line breaks are preserved on display.                                                 |
| Posted            | Yes           | Date/time the item was posted. Format: M/D/YY H:MM AM/PM. Used for sorting and "new" badge calculation.                    |
| Expires           | No            | Date/time the item stops showing. Format: M/D/YY H:MM AM/PM. Leave blank for items that do not expire.                     |
| Posted By         | No            | Name of the person who posted the item. Shown in card metadata.                                                             |
| Recurring         | No            | Interval in days for recurring items (e.g. 7 for weekly). Leave blank for non-recurring items.                              |
| Stop After        | No            | Date after which a recurring item no longer repeats. Format: M/D/YY.                                                        |

## 9.7 Configuration

| **Constant**              | **Description**                                                                                                            |
|---------------------------|----------------------------------------------------------------------------------------------------------------------------|
| NEW_ITEM_DAYS             | Number of days after posting that an item shows the NEW badge (default: 3).                                                |
| DISPLAY_DURATION_SECONDS  | How long the display shows this Worker before cycling. Used to calculate scroll speed (default: 20).                       |
| SCROLL_PAUSE_SECONDS      | Seconds to pause at the top and bottom of the scroll animation before reversing (default: 3).                              |
| MIN_SCROLL_SPEED_PX_PER_SEC | Minimum scroll speed in pixels per second (default: 20).                                                                 |
| MAX_SCROLL_SPEED_PX_PER_SEC | Maximum scroll speed in pixels per second (default: 120).                                                                |
| CARD_BODY_LINE_HEIGHT     | Line height for card body text (default: 1.25). Adjust if line spacing appears too tight or too loose on hardware.         |
| CARD_META_LINE_HEIGHT     | Line height for card metadata (default: 1.6).                                                                              |
| DELETE_EXPIRED_AFTER_DAYS | Days after expiry before expired non-recurring rows are automatically deleted by the scheduled cron job. -1 disables.      |

## 9.8 Adding News Items

1. Open the Google Sheet and navigate to the appropriate tab (Department News or the relevant station tab).

2. Add a new row with at minimum Title, Content, and Posted filled in.

3. Fill in Expires if the item should stop showing after a specific date/time. Leave blank for permanent items.

4. The Worker fetches sheet data on every request (with a short cache). New items appear on displays within 5 minutes.

## 9.9 Recurring Items

To create a recurring item (e.g. a weekly reminder), fill in the Recurring column with the interval in days (e.g. 7 for weekly) and optionally set a Stop After date. The item will repeat from its original Posted date at the configured interval until the Stop After date is reached.

# 10. Project: Weather Display

## 10.1 Purpose & Problem Solved

Station displays needed a dedicated full-screen weather display showing current conditions, a 3-day forecast, an animated radar map, and weather alerts in a single view. The Weather Display Worker fetches data from the NWS public API, AirNow API, and RainViewer API and renders a responsive HTML page tailored to each display layout. No manual updates are required — all data refreshes automatically.

## 10.2 Repository & Deployment

| **Item**          | **Value**                                                                                                                                           |
|-------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------|
| GitHub Repository | github.com/wehnerb/weather-display                                                                                                                  |
| Production URL    | https://weather-display.bwehner.workers.dev/                                                                                                        |
| Staging URL       | https://weather-display-staging.bwehner.workers.dev/                                                                                                |
| Worker File       | src/index.js                                                                                                                                        |
| Secrets Required  | CLOUDFLARE_API_TOKEN, CLOUDFLARE_ACCOUNT_ID (GitHub); AIRNOW_API_KEY (Cloudflare dashboard)                                                         |

## 10.3 URL Parameters

| **Parameter** | **Default** | **Options**                  | **Description**                                                                                  |
|---------------|-------------|------------------------------|--------------------------------------------------------------------------------------------------|
| layout        | wide        | wide, full, split, tri       | Display size. wide and full show the full display (radar + conditions + hourly strip). split and tri show either radar or conditions only, controlled by the view parameter. |
| view          | radar       | radar, conditions            | For split and tri layouts only. Selects which half of the display to render.                     |
| bg            | (none)      | dark                         | Testing parameter. Adds a dark background for browser-based testing. Not for production use.     |

## 10.4 How It Works

1. The Worker fetches data in parallel from multiple sources: NWS current conditions, NWS daily forecast, NWS hourly forecast, NWS active alerts for Cass County (NDZ039), AirNow AQI for Fargo (zip 58102), and RainViewer radar frame metadata.

2. Each data source fails gracefully — if any upstream source is unavailable, the relevant section is omitted rather than returning an error page.

3. Weather data is edge-cached at Cloudflare to limit upstream API calls across all stations.

4. A self-contained HTML page is rendered for the requested layout and returned to the display.

## 10.5 Display Sections

**Current Conditions:** Large temperature display with apparent temperature, AQI badge, condition icon and text. Wind, humidity, dew point, pressure, and visibility stats. Today's high/low temperature and sunrise/sunset times.

**3-Day Forecast:** Three forecast rows showing day name, condition icon, short forecast text, wind speed, precipitation probability, and high/low temperatures. Future weather alerts that overlap a forecast day appear as color-coded badges in that day's row.

**12-Hour Strip:** A horizontal strip at the bottom of the wide/full layouts showing 12 hourly slots. Each slot displays the time label, condition icon, temperature, and precipitation probability.

**Radar:** An animated radar map using Leaflet.js with the CartoDB Dark Matter base layer and RainViewer radar tile overlays. The animation uses a double-buffer approach (two alternating tile layers) for smooth frame transitions. The radar shows the most recent 18 frames in a loop with a hold on the final (most recent) frame.

**Alert Banners:** Active NWS weather alerts appear as full-width colored banners at the top of the display. Banner colors: red = Warning, orange = Watch, yellow = Advisory. Up to MAX_DISPLAY_ALERTS banners are shown simultaneously; the most severe alerts take priority. Future alerts (not yet active) fill any remaining banner slots with a "begins Day H:MM AM" label. When alerts are present, all content below scales proportionally to fit the remaining vertical space.

## 10.6 Configuration

| **Constant**          | **Description**                                                                                                         |
|-----------------------|-------------------------------------------------------------------------------------------------------------------------|
| LOCATION_LAT          | Latitude of the display location (Fargo: 46.8772).                                                                     |
| LOCATION_LON          | Longitude of the display location (Fargo: -96.7898).                                                                   |
| NWS_OFFICE            | NWS forecast office identifier (FGF for Fargo).                                                                        |
| NWS_GRID_X / Y        | NWS grid point coordinates (65, 57 for Fargo).                                                                         |
| NWS_ALERT_ZONE        | NWS alert zone identifier (NDZ039 = Cass County ND).                                                                   |
| RADAR_ZOOM            | Leaflet map zoom level for the radar display (default: 7).                                                              |
| RADAR_FRAMES_COUNT    | Number of historical radar frames to animate (default: 18).                                                             |
| RADAR_FRAME_MS        | Milliseconds between radar animation frames (default: 200).                                                             |
| RADAR_HOLD_MS         | Milliseconds to hold on the final (most recent) frame before looping (default: 2000).                                   |
| RADAR_INIT_DELAY_MS   | Milliseconds to wait after page load before initializing the Leaflet map. Allows Pi hardware layout to complete first.  |
| RADAR_OPACITY         | Opacity of the radar overlay tiles (0.0–1.0).                                                                           |
| MAX_DISPLAY_ALERTS    | Maximum number of alert banners displayed simultaneously (default: 3).                                                  |
| ALERT_BANNER_HEIGHT_PX| Vertical pixels reserved per active alert banner for proportional content scaling (default: 52).                        |
| HOURLY_COUNT          | Number of hourly slots shown in the bottom strip (default: 12).                                                         |

## 10.7 AirNow API Key

The AIRNOW_API_KEY secret must be set in the Cloudflare dashboard. Register for a free key at docs.airnowapi.org. If the key is missing or the AirNow API is unavailable, the AQI badge is silently omitted.

## 10.8 NWS Configuration

NWS_USER_AGENT is set as a plain [vars] entry in wrangler.toml and identifies the Worker to the NWS API as required by NWS terms of service.

# 11. Shared Utilities

## 11.1 Overview

The ffd-station-display-utils repository contains shared design tokens, helper functions, and utility modules used by all 8 Worker projects. All Workers sync from this repository automatically via a GitHub Actions pipeline whenever the shared utilities are updated.

## 11.2 Repository

| **Item**          | **Value**                                            |
|-------------------|------------------------------------------------------|
| GitHub Repository | github.com/wehnerb/ffd-station-display-utils         |
| Worker File       | N/A — this is a shared library, not a Worker         |
| Branches          | main only (no staging branch needed)                 |

## 11.3 Shared Modules

| **File**           | **Purpose**                                                                               |
|--------------------|-------------------------------------------------------------------------------------------|
| colors.js          | Design tokens: background colors, font stacks, text opacity hierarchy, border levels, card elevation levels, accent color |
| alert-colors.js    | Alert color tokens: warning/watch/advisory backgrounds, borders, and text colors          |
| layouts.js         | Layout dimensions for wide, full, split, and tri display sizes                           |
| fetch-helpers.js   | fetchWithTimeout helper with AbortController for all upstream API fetches                |
| html.js            | escapeHtml and sanitizeParam helper functions for safe HTML output                       |
| google-auth.js     | getAccessToken function for Google service account JWT authentication                    |
| rotation.js        | getTodayString, getBlockIndex, getSecondsUntilNextRotation helpers for 3-day rotation     |

## 11.4 Auto-Sync Pipeline

When a change is merged to main in ffd-station-display-utils, a GitHub Actions workflow (notify-workers.yml) automatically opens a pull request in each of the 8 Worker repositories to pull in the updated shared files. These PRs target the staging branch of each Worker. Each Worker's sync-utils.yml workflow merges the PR to staging and then auto-merges staging to main.

To trigger the pipeline: update VERSION in ffd-station-display-utils, commit to main, and the pipeline runs automatically.

## 11.5 Design Token Summary

| **Token**        | **Value**                        | **Usage**                                                  |
|------------------|----------------------------------|------------------------------------------------------------|
| DARK_BG_COLOR    | #111111                          | Solid background for full layout and ?bg=dark testing      |
| FONT_STACK       | "Segoe UI", Arial, Helvetica     | Primary sans-serif font stack (web fonts avoided on Pi)    |
| FONT_STACK_SERIF | Georgia, "Times New Roman"       | Serif stack for message body text                          |
| ACCENT_COLOR     | #FF0000                          | FFD red — used only for functional elements                |
| TEXT_PRIMARY     | rgba(255,255,255,0.92)           | Main body text, values, titles                             |
| TEXT_SECONDARY   | rgba(255,255,255,0.68)           | Labels, metadata, subdued content                          |
| TEXT_TERTIARY    | rgba(255,255,255,0.50)           | Timestamps, hints, least-important content                 |
| TEXT_SUPPORTING  | rgba(255,255,255,0.75)           | Supporting content between SECONDARY and PRIMARY           |
| BORDER_SUBTLE    | rgba(255,255,255,0.10)           | Standard card borders, dividers                            |
| BORDER_STRONG    | rgba(255,255,255,0.18)           | Emphasized separators, today-panel borders                 |
| CARD_RECESSED    | rgba(255,255,255,0.04)           | Subtly inset cells within a card                           |
| CARD_BASE        | rgba(255,255,255,0.07)           | Standard card surface                                      |
| CARD_ELEVATED    | rgba(255,255,255,0.11)           | Slightly raised blocks within a card                       |
| CARD_HEADER      | rgba(255,255,255,0.16)           | Today/primary panel header — highest card-level elevation  |

| **Design Rule**                                                                                                                                                     |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| ACCENT_COLOR is used functionally only — never as a decorative top border or purely ornamental element. Only font weights 400 and 700 are portable across the Pi font fallback chain. Do not use 300, 500, or 600. |

# 12. Deployment & Maintenance Workflow

## 12.1 Branch Strategy

Each Worker repository uses two branches. All changes must go through staging before being merged to main.

| **Branch** | **Deploys To**                         | **Purpose**                                          |
|------------|----------------------------------------|------------------------------------------------------|
| staging    | \[worker\]-staging.bwehner.workers.dev | Testing and validation of changes before production  |
| main       | \[worker\].bwehner.workers.dev         | Live production environment used by station displays |

## 12.2 Making a Code Change

Follow these steps in order for all code changes:

1.  Create a new branch from staging in the GitHub repository (or edit directly in staging for simple one-line changes).

2.  Edit the relevant file(s) using the GitHub browser editor.

3.  Commit to the new branch and open a pull request targeting staging.

4.  GitHub Actions will automatically deploy the change to the staging Worker within about 30–45 seconds.

5.  Test the staging Worker URL in a browser to confirm the change works as expected.

6.  For display-dependent changes, temporarily update the endpoint URL on a single station display to the staging Worker URL. Confirm the change works correctly on actual display hardware before proceeding.

7.  Once both browser and display tests pass, merge the pull request to staging, then merge staging to main.

8.  GitHub Actions will automatically deploy to the production Worker within about 30–45 seconds.

9.  Verify the production Worker URL is functioning correctly. Return any display temporarily set to staging to the production URL.

### Rolling Back a Change

- **Simple rollback:** Edit the file in the staging branch in GitHub to restore the previous value.
- **Commit history rollback:** Go to the branch in GitHub, click the commit history, find the target commit, and use Revert.
- **Production rollback:** Go to the main branch commit history, find the last known-good commit, and use GitHub's Revert feature. This triggers a redeployment automatically.
- **Re-sync branches after rollback:** After reverting main, merge main back into staging immediately.

## 12.3 Important Deployment Notes

| **Important**                                                                                                                                                              |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Never deploy directly to main without first testing in staging. If a deployment fails, the previous version remains live — Cloudflare does not replace the running Worker until a new deployment succeeds. |

| **Important — X-Frame-Options Header**                                                                                                                                                                                                                                                                                                                                                                                                               |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| The X-Frame-Options: SAMEORIGIN header must never be added to any of these Workers. All Workers are loaded as full-screen iframes by the display system, and this header causes the browser to block the page from rendering entirely, producing an immediate white error screen. |

## 12.4 Monitoring Deployments

Every deployment is logged in the Actions tab of the GitHub repository. All 8 Workers are monitored by UptimeRobot via their /healthz endpoints. Alerts are sent by email when any Worker goes down or recovers.

# 13. IT Support Reference

## 13.1 Overview for IT Staff

This system runs entirely on external cloud services and does not require any on-premises server infrastructure, VPNs, or internal network changes. The display screens require only outbound internet access to reach the Cloudflare Worker URLs. There is no software to install, no servers to patch, and no internal ports to open.

IT involvement would typically only be needed for:

- Network connectivity issues at the station that prevent displays from reaching external URLs.
- Display hardware or operating system issues at the screen level.
- Account access issues if the department member managing this system is unavailable.

## 13.2 Network Requirements

The display screens must have outbound internet access on port 443 (HTTPS) to the following domains:

- \*.workers.dev — Cloudflare Worker endpoints (all 8 Worker projects)
- docs.google.com — Google Slides embeds
- www.googleapis.com — Google Drive API (photo and image requests proxied through the Worker)
- www.dot.nd.gov — ND DOT traffic camera images (fetched directly by the display browser)
- usgs-nims-images.s3.amazonaws.com — USGS river camera images (fetched directly by the display browser)
- water.noaa.gov — NOAA river gauge images (fetched directly by the display browser)
- cdnjs.cloudflare.com — Leaflet.js CSS and JavaScript for the weather display radar map
- \*.basemaps.cartocdn.com — CartoDB Dark Matter base map tiles for the radar display
- tilecache.rainviewer.com — RainViewer radar overlay tiles

Note: api.weather.gov, api.rainviewer.com, and www.airnowapi.org are fetched by the Cloudflare Worker on its own network — not by the display screens directly.

## 13.3 Troubleshooting

| **Symptom**                                       | **Likely Cause**                                       | **Resolution**                                                                                                                                                                                          |
|---------------------------------------------------|--------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Display shows blank/white screen                  | Network connectivity or display hardware issue         | Check internet connectivity at the screen. Verify the Worker URL loads in a browser on the same network.                                                                                                |
| Images show INVALID IMAGE KEY error               | Invalid or missing image key in the URL                | Check the img parameter in the display URL against the MAPPING list in worker_code.js (Section 3.9).                                                                                                    |
| Images show IMAGE UNAVAILABLE                     | Source camera or data feed is offline                  | Check the source URL directly. This is typically a third-party outage outside department control.                                                                                                       |
| River gauge shows error page                      | NOAA API temporarily unavailable                       | The page retries automatically every 60 seconds. Check api.water.noaa.gov directly if persistent.                                                                                                       |
| Slides cycling at wrong speed                     | Slide count API call failing or secrets missing        | Check Cloudflare Worker logs. Verify GOOGLE_SERVICE_ACCOUNT_EMAIL and GOOGLE_PRIVATE_KEY are present in the Cloudflare dashboard for slide-timing-proxy.                                                |
| Calendar shows no weather data                    | NWS API temporarily unavailable                        | The calendar renders without weather rather than showing an error. Typically self-resolves.                                                                                                             |
| Probationary firefighter photo does not load      | Drive permissions or file not found                    | Verify the Drive folder is shared with the service account email. Verify the Photo column filename matches the Drive filename. Check Cloudflare Worker logs for errors.                                 |
| Probationary firefighter display shows no content | No firefighters hired within the past 365 days         | Check that hire dates are in YYYY-MM-DD format. Verify the sheet is shared with the service account email.                                                                                              |
| Department news shows "No current news"           | No active items in the sheet tab for this station      | Check the relevant tab in the Google Sheet. Verify Posted and Expires dates are correct and items are not expired.                                                                                      |
| Weather radar map is white or partially loaded    | Leaflet map initialization timing issue on Pi hardware | This is a known intermittent issue. The display refreshes automatically and typically self-resolves on the next cycle.                                                                                  |
| Weather display shows no AQI badge               | AIRNOW_API_KEY missing or AirNow API unavailable       | Check that AIRNOW_API_KEY is set in the Cloudflare dashboard for weather-display. The badge is silently omitted when unavailable.                                                                        |
| UptimeRobot shows a Worker as down               | Worker returning non-200 status or timing out          | Load the /healthz URL directly in a browser. If it returns "status: healthy", the alert may have been a transient blip. If it returns "status: degraded", check Cloudflare Worker logs for details.     |

## 13.4 Uptime Monitoring

All 8 Workers are monitored via UptimeRobot at 5-minute intervals. Each monitor checks the Worker's /healthz endpoint and alerts by email when the Worker is down or recovers.

| **Monitor Name**                       | **URL**                                                                     |
|----------------------------------------|-----------------------------------------------------------------------------|
| FFD Station Image Proxy                | https://station-image-proxy.bwehner.workers.dev/healthz                     |
| FFD Slide Timing Proxy                 | https://slide-timing-proxy.bwehner.workers.dev/healthz                      |
| FFD River Level Display                | https://river-level-display.bwehner.workers.dev/healthz                     |
| FFD Daily Message Display              | https://daily-message-display.bwehner.workers.dev/healthz                   |
| FFD Calendar Display                   | https://calendar-display.bwehner.workers.dev/healthz                        |
| FFD Department News Display            | https://department-news-display.bwehner.workers.dev/healthz                 |
| FFD Probationary Firefighter Display   | https://probationary-firefighter-display.bwehner.workers.dev/healthz        |
| FFD Weather Display                    | https://weather-display.bwehner.workers.dev/healthz                         |

## 13.5 Service Limits & Monitoring

| **Service**                              | **Free Tier Limit**                   | **Est. Daily Usage (8 stations)**                                              | **Where to Check**                                     |
|------------------------------------------|---------------------------------------|--------------------------------------------------------------------------------|--------------------------------------------------------|
| Cloudflare Workers (combined total)      | 100,000 req/day                       | ~20,000–30,000 req/day (varies by layout and cache hit rate)                   | dash.cloudflare.com → Workers & Pages → Overview       |
| Google Slides API                        | 300 req/minute                        | At most 1 request per hour per cache version                                   | console.cloud.google.com → APIs & Services → Dashboard |
| Google Sheets API                        | 300 req/minute per project            | Low — edge-cached to limit calls                                               | console.cloud.google.com → APIs & Services → Dashboard |
| Google Drive API                         | 1,000 req/100 seconds                 | Low — one listing call per Worker request, results edge-cached                 | console.cloud.google.com → APIs & Services → Dashboard |
| NOAA NWPS / NWS APIs                     | No documented hard limit              | Low — all responses edge-cached for 15–60 minutes                              | Monitor via displays on screens                        |
| AirNow API                               | 500 req/hour                          | ~8–16 req/hour across 8 stations — well within limit                           | docs.airnowapi.org → My Account                        |
| RainViewer API                           | No documented hard limit              | Low — metadata fetched per Worker request; tiles fetched by display browser    | Monitor via weather display on screens                 |
| UptimeRobot                              | 50 monitors, 5-min interval (free)    | 8 monitors active — well within limit                                          | uptimerobot.com → Dashboard                            |

## 13.6 Contact & Escalation

For issues with application code, configuration, or Cloudflare/GitHub accounts, contact Brandon Wehner.

For issues with display screen hardware or operating system, contact the shift personnel responsible for the station alerting and communications system.

For local network connectivity issues preventing displays from reaching external URLs, contact the City of Fargo Information Services (IS) Department.

# 14. Planned Enhancements

## 14.1 Incident & Performance Data (First Due)

| **Enhancement**                    | **Description**                                                                                                                                   | **Priority** | **Status**          |
|------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------|--------------|---------------------|
| Turnout Time Display               | Graphical display showing the percentage of calls where the crew met the department's turnout time benchmark.                                    | High         | Under Consideration |
| Turnout Time Trending              | Graphical representation showing turnout time trends over time.                                                                                  | High         | Under Consideration |
| Incident Type by Final Disposition | Graphical breakdown of incident types using final report disposition.                                                                            | High         | Under Consideration |
| Time-of-Day Call Distribution      | Chart showing when calls are most frequent throughout the day.                                                                                    | Medium       | Under Consideration |
| Year-over-Year Call Volume         | Comparison of call volume across years to identify growth trends.                                                                                 | Medium       | Under Consideration |
| Geographic Call Clustering         | Heat map or cluster map showing where calls are concentrated within the response area.                                                            | Low          | Under Consideration |

## 14.2 Display Content Enhancements

| **Enhancement**                           | **Description**                                                                                                                                                                          | **Priority** | **Status**          |
|-------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------|---------------------|
| Daily Safety Message Display              | Daily messages, quotes, and images displayed on station screens, rotating every 3 days.                                                                                                  | Medium       | Complete            |
| Calendar Display                          | Station calendar with weather data. Exported from Outlook automatically via VBA macro, synced to Nextcloud, fetched by the Worker via WebDAV.                                            | Medium       | Complete            |
| NOAA Weather Alerts                       | Active and upcoming NWS weather alerts on calendar and weather displays with severity-colored banners and future alert badges.                                                           | Medium       | Complete            |
| Probationary Firefighter Display          | Rotating spotlight with photo and bio sourced from Google Sheets and Drive. Cycles through all active new hires on a 3-day rotation. Drops off automatically after 365 days from hire.  | Medium       | Complete            |
| Department News Display                   | Card-based department-wide and station-specific news feed sourced from Google Sheets. Supports recurring items, new item highlighting, and auto-scrolling for overflow content.          | Medium       | Complete            |
| Weather Display                           | Full weather display with animated RainViewer radar, NWS current conditions, 3-day forecast, 12-hour strip, AirNow AQI, and active/future alert banners.                                | Medium       | Complete            |
| Birthday & Department Anniversary Display | Display birthdays and department hire anniversaries for the current week or month.                                                                                                       | Low          | Under Consideration |
| Retirement Countdown                      | Countdown display for upcoming retirements (with member permission).                                                                                                                     | Low          | Under Consideration |
| Additional River Gauges                   | Add additional NOAA gauge locations to the river-level-display GAUGES registry (e.g., Moorhead, MN — gauge MHDN8).                                                                       | Low          | Under Consideration |
