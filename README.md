# FFD Display Board — System Documentation

This repository is the central documentation hub for the **Fargo Fire Department Station Display Board** system — a set of Cloudflare Workers that power digital display screens throughout the fire station.

## 📄 Documentation

| File | Description |
|---|---|
| [`fire_station_display_documentation.md`](./fire_station_display_documentation.md) | Full technical reference (Markdown) |
| [`fire_station_display_documentation.docx`](./fire_station_display_documentation.docx) | Same document in Word format for non-technical staff |

The documentation covers system architecture, all five Worker projects, account and access management, deployment workflows, IT support reference, and planned enhancements.

## 🗂️ Code Repositories

| Project | Purpose | Repository |
|---|---|---|
| Image Resizing Proxy | Resizes and caches traffic camera and river images | [station-image-proxy](https://github.com/wehnerb/station-image-proxy) |
| Slide Timing Proxy | Calculates per-slide timing and redirects to Slides embed | [slide-timing-proxy](https://github.com/wehnerb/slide-timing-proxy) |
| River Level Display | Canvas-based NOAA hydrograph for Red River gauges | [river-level-display](https://github.com/wehnerb/river-level-display) |
| Daily Message Display | Rotating safety messages and images from Google Sheets/Drive | [daily-message-display](https://github.com/wehnerb/daily-message-display) |
| Calendar Display | FFD Calendar rendered as HTML from ICS via Nextcloud | [calendar-display](https://github.com/wehnerb/calendar-display) |

## 👤 Maintained By

Brandon Wehner — Fargo Fire Department

For questions about this system, see **Section 9.5** of the documentation.
