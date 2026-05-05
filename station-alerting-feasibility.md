# In-House Fire Station Alerting System — Feasibility & Project Notes

**Project:** Replace current vendor alerting system with an in-house solution  
**Driver:** Cost reduction (per-station annual subscription fees)  
**Status:** Early feasibility / pre-planning  
**Last Updated:** May 2026

---

## Overview

The goal is to replace the current third-party alerting software with an in-house system built and maintained in partnership with the city IT department. The existing hardware (Raspberry Pi units, I/O relay board) would be retained. Only the software stack would be replaced.

This document captures feasibility considerations, known risks, and design requirements discussed during early planning. It should be updated as the project progresses through discussions with IT and other stakeholders.

---

## Hardware (Existing)

- **Alerting Pi** — Handles XML polling, parsing, and relay activation
- **Display Pi** — Runs the in-station display system
- **I/O Relay Board** — Controls physical alerting hardware (exact part numbers TBD); likely GPIO-based but may use SPI or I2C — confirm communication protocol early
- Hardware is retained as-is; this is a software replacement project

---

## System Requirements

### 1. XML Parsing & Relay Activation
- Pi polls a folder on a city server for new or updated XML files
- System must parse XML, evaluate alert conditions, and activate the correct relays
- XML files are moved to an archive folder after 3 days, then deleted after 7 additional days — **the system cannot rely on XML files as a long-term data source**
- All parsed event data must be captured to a local or network database immediately on processing
- Known edge cases from the current vendor system must be documented and built into requirements before development begins

### 2. Data Retention & Statistics
- A local database (e.g., SQLite on the Pi, or a city network database) must store parsed event records persistently
- Statistics displayed on station screens are calculated from this retained data, not from the XML files directly
- Retention policy for this database should be defined — recommend keeping data significantly longer than the 10-day XML window

### 3. Display System Integration
- The display Pi runs a browser-based kiosk connected to the existing display framework (Cloudflare Workers infrastructure already in place)
- On alert receipt, a call information screen must override the standard display
- The alerting Pi triggers the display override via an event mechanism (WebSocket push, HTTP call, MQTT, or similar — TBD)
- Statistics and standard display content continue to function when no active alert is present

### 4. Configuration GUI
- A web-based admin interface (hosted on the city network) for station configuration
- Must allow mapping of relay outputs to alert conditions (station ID, apparatus type, priority, time of day, known edge cases, etc.)
- Must be usable by non-developers
- Must include input validation to prevent misconfiguration that could cause a missed or incorrect alert
- Must maintain a full audit log of configuration changes (who changed what, and when)

### 5. Logging
- Robust, timestamped logging is a hard requirement — not optional
- Log entries must capture:
  - Raw XML content on receipt
  - Parsed output
  - Relay activation logic and result
  - Any errors or unexpected conditions
  - Configuration changes via the admin GUI
- Logs must be queryable for troubleshooting and anomaly review
- Architecture: local logs on the Pi (resilient to network outages) + forwarding to centralized city server logging infrastructure
- Log retention policy to be defined — recommend longer retention than XML files given logs may be the only forensic record of an alerting event

### 6. Reliability Monitoring
- System must integrate with city IT monitoring infrastructure
- Alerts should fire if the alerting service stops, polling fails, or the Pi becomes unreachable
- Watchdog processes should be considered to auto-recover from software hangs without requiring manual reboot

---

## Architecture Notes

- All infrastructure runs on internal city servers and city-managed network
- City IT department is a primary partner — responsible for server infrastructure, deployment, security review, and ongoing maintenance
- FFD role: requirements definition, edge case documentation, testing, and operational validation
- Display framework already established via Cloudflare Workers — new system should conform to existing patterns where possible

---

## Risk Register

| Risk | Severity | Notes |
|---|---|---|
| Transition / cutover period | High | Cannot simply switch off current system; parallel operation and validation plan required before decommissioning |
| Life-safety reliability requirements | High | Missed or incorrect alerts have operational consequences; reliability design must be treated with appropriate rigor |
| IT staff turnover | Medium | Custom system knowledge must be thoroughly documented; institutional knowledge cannot live only in one developer's head |
| Unknown edge cases | Medium | Known edge cases from vendor discussions must be captured as requirements; logging strategy will help surface unknown ones post-deployment |
| I/O board communication protocol | Medium | Confirm early whether relay board uses standard GPIO or a specific protocol (SPI, I2C, proprietary) — affects development approach |
| Scope creep | Medium | Initial deployment should be feature-minimal and stable; enhancements added only after proven reliability |
| City IT approval and security review | Low-Medium | IT is already a project partner, but formal approval processes may affect timeline |

---

## Known Edge Cases (To Be Documented)

> This section should be populated with the specific edge cases identified through past discussions with the current vendor. These must be captured as explicit requirements before development begins.

- [ ] Edge case 1 — *description TBD*
- [ ] Edge case 2 — *description TBD*
- [ ] ...

---

## Open Questions

- [ ] Exact I/O board model and communication protocol
- [ ] Number of stations in scope (affects cost/ROI calculation)
- [ ] City IT capacity and timeline availability
- [ ] Preferred database platform (local SQLite vs. networked DB)
- [ ] Log retention policy duration
- [ ] Event mechanism for display override trigger (WebSocket, MQTT, HTTP, etc.)
- [ ] Parallel operation / cutover plan details
- [ ] Formal IT project intake process and timeline

---

## Cost & ROI Notes

- Current cost: per-station annual subscription fee × number of stations
- In-house ongoing cost: primarily IT staff time for maintenance
- One-time cost: IT development time
- Breakeven analysis should be completed with IT department once development effort is estimated
- Hardware costs are zero (existing hardware retained)

---

## Decision Log

| Date | Decision | Notes |
|---|---|---|
| May 2026 | Feasibility discussion initiated | Cost-driven; IT partnership model confirmed as preferred approach |

---

*This document is a living reference. Update as project discussions progress.*
