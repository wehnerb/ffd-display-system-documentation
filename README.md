# FFD Display Board System Documentation

This repository is the central documentation hub for the **Fargo Fire Department Station Display Board** system — a set of Cloudflare Workers that power digital display screens throughout the fire station.

## 📄 Documentation

| File | Description |
|---|---|
| [`fire_station_display_documentation.md`](./fire_station_display_documentation.md) | Full technical reference (Markdown) |

The documentation covers system architecture, all eight Worker projects, the shared utilities library, account and access management, deployment workflows, IT support reference, and planned enhancements.

## 🗂️ Code Repositories

| Project | Purpose | Repository |
|---|---|---|
| Image Resizing Proxy | Resizes traffic camera and river images for display hardware | [station-image-proxy](https://github.com/wehnerb/station-image-proxy) |
| Slide Timing Proxy | Calculates per-slide timing and redirects to Slides embed | [slide-timing-proxy](https://github.com/wehnerb/slide-timing-proxy) |
| River Level Display | Canvas-based NOAA hydrograph for Red River gauges | [river-level-display](https://github.com/wehnerb/river-level-display) |
| Daily Message Display | Rotating safety messages and images from Google Sheets/Drive | [daily-message-display](https://github.com/wehnerb/daily-message-display) |
| Calendar Display | FFD Calendar rendered as HTML from ICS via Nextcloud, with NWS weather | [calendar-display](https://github.com/wehnerb/calendar-display) |
| Probationary Firefighter Display | Rotating new hire spotlight with bio and photo from Google Sheets/Drive | [probationary-firefighter-display](https://github.com/wehnerb/probationary-firefighter-display) |
| Department News Display | Department-wide and station-specific news cards from Google Sheets | [department-news-display](https://github.com/wehnerb/department-news-display) |
| Weather Display | Full weather display with animated radar, NWS conditions, and AirNow AQI | [weather-display](https://github.com/wehnerb/weather-display) |
| Shared Utilities | Design tokens, helpers, and utilities shared across all Workers | [ffd-station-display-utils](https://github.com/wehnerb/ffd-station-display-utils) |

## 👤 Maintained By

Brandon Wehner — Fargo Fire Department

For questions about this system, see **Section 13.6** of the documentation.