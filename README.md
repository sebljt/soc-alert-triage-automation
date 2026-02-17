# Project Plan – Intelligence-Driven Alert Correlation and Triage Automation

## Stage 1 – Planning and Design

### Scope and success criteria
- Project title: *Developing an Intelligence-Driven Alert Correlation and Triage Automation System*.
- Outcomes covered: S21, S25, S26, S28, S29, S30.
- Key deliverables: Incident Response Plan, configured SIEM, enrichment logic, anomaly rules, demonstration incident, root cause analysis write-ups.

### Logical architecture
- Components and roles:
  - WatchGuard Firebox / WatchGuard Cloud – primary network monitor and IPS alert source.
  - Splunk – SIEM for ingesting WatchGuard syslog, correlation and dashboards.
  - MISP and AbuseIPDB – threat intelligence and IP reputation for source/destination IPs.
  - TheHive – case/ticketing style workflow for incidents (PSA-like).
- Data flow: **WatchGuard IPS/syslog → Splunk → enrichment (MISP/AbuseIPDB) → TheHive → analyst**.

### Incident scenario selection
- Select one non‑major but realistic scenario based on a WatchGuard IPS “Action: drop” alert to a critical web server.
- Re‑use this scenario for detection, analysis, IR Plan demonstration, and the incident manager report.

---

## Stage 2 – Monitoring Architecture (S28)

### WatchGuard setup
- Enable logging of IPS, firewall and system events on the WatchGuard device.
- Configure a syslog/log server entry pointing to the Splunk/syslog host (IP, port 514 UDP or chosen port).
- Generate a small test event (e.g. policy change, test IPS trigger) and confirm it appears in the device logs.

### Splunk setup for WatchGuard logs
- Create a UDP/TCP input in Splunk for the WatchGuard log port and assign a dedicated index (e.g. `watchguard`).
- Set an appropriate sourcetype (e.g. `watchguard:firebox` or `syslog`) to keep IPS and firewall data consistent.
- (Optional but recommended) Install the WatchGuard Firebox add‑o
