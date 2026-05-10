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

## Known Edge Cases and Existing Issues

> This section should be populated with the specific edge cases identified through past discussions with the current vendor. These must be captured as explicit requirements before development begins.

- [ ] Stations Re-Alerting for Old Incidents - After server maintenance periods, stations would re-alert for any incident in their station that was still in the active folder. APS' fix was to create a hash of the file contents, then compare that hash to a memory of previous hashes (memory was about 30 days), and if it was a match, not alert. This meant that any calls that had already been closed would have had a matching hash after the server came back online and it would not alert for those incidents. IS found that file contents did change after server maintenance, but the APS fix did seem to take care of the issue
- [ ] Delayed Station Notification - CAD Exporter does not reliably export data within a reasonable time (up to 10-12 seconds). APS will be implementing a fix (as of 4/2026) that utilizes the email feature from CAD for quick notification (if the email server responds quickly enough), then merges that data with the XML file once that is created. This gives us quick notification with the full data of the XML file
- [ ] Station Re-Alerted - Station 5 was realerted over a day after they had cleared from a call. We determined that the call was still active for PD and when a PD unit was updated, that caused the station to be realerted. A new flag was added to the listener so that if the call was closed for fire, the station did not alert, even if the file was updated
- [ ] Station Not Alerted When a Unit was Re-Added - Station 5 was not alerted for a call after they had cleared from it. They initially responded to a call, but were cleared by PD, so they cleared and the call was closed for fire. They were then re-added to the call, but the station did not alert because it had already been shown as closed for fire. APS implemented a fix, but I don't know how they fixed it.
- [ ] Old Calls Causing Activation when Updated by Dispatch - Station 5 was alerted at 17:08:31 on 11/22/25 due to an incident being detected from 10/12/25. APS app showed the 10/12 date, but data in the controller log doesn't seem to reference this date. Per Todd in IS on 11/24: *It looks like the call was reactivated on 11/22 @ 17:08 by Dispatch, the call source was changed from Telephone to 911, and then the call was closed again. This might be a conversation on two fronts, APS on if there is a way to exclude scenarios like this, and Dispatch on a process to address re-opening of old calls.* APS implemented a fix that ignores any calls older than 30 days
- [ ] Station Missed Alerts - Station 8 missed alerts because the OS had rebooted and had not fully connected back to the network. Per APS: *On investigation I found the following. The controller was rebooted, the OS attempted to reconnect to the network share on boot, which partially failed. The cause of the failure was timing issue. When the system rebooted it tried to remount the network share, just before or as the network interface was connecting. This in turn caused the network share to appear mounted when it wasn’t. This required manual intervention to resolve. For now, I have added a new flag to wait for network connectivity, before mounting the network share. I also have created story to prevent this issue in the future by identifying when it happens and auto correcting.*

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
