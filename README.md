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
- (Optional but recommended) Install the WatchGuard Firebox add‑on so fields like `src_ip`, `dest_ip`, `src_port`, `dest_port`, `protocol`, `rule_id`, `action`, `policy_name` are automatically parsed.

### Baseline normal behaviour
- Run searches over a sample period on the WatchGuard index to understand normal inbound/outbound traffic and typical IPS alerts.
- Capture notes and screenshots as a baseline to justify anomaly rules later.

---

## Stage 3 – Threat Intelligence and Enrichment (S25)

### MISP setup
- Install or provision a MISP instance (VM/container/server) accessible from your lab network.
- Complete basic configuration: base URL, organisation, admin account.
- Create an API key for integrations (TheHive and/or Splunk) and store it securely.
- Enable at least one feed or import a sample event so you have IOCs (IPs, domains, hashes) to work with.

### AbuseIPDB setup
- Create an AbuseIPDB account for your project and generate an API key.
- Install or configure an integration in Splunk (app, script, or custom command) that can query AbuseIPDB using that key.
- Test with a known IP address to confirm enrichment fields (confidence score, categories, last reported) are returned.

### TheHive ↔ MISP integration setup
- Configure TheHive to connect to MISP using the MISP URL and API key.
- Verify connectivity by:
  - pulling a MISP event into TheHive as an alert, and/or
  - pushing an observable from a test case in TheHive into MISP as an attribute.

### Implement enrichment in Splunk
- Build lookups or search commands that enrich WatchGuard IPS events with:
  - AbuseIPDB scores and categories for `src_ip` (and if needed `dest_ip`);
  - MISP tags or matches where IPs appear in MISP events.
- Create example searches showing IPS events before and after enrichment so you can evidence the correlation outcome.

### Tool configuration driven by threat intelligence
- Use specific intelligence (e.g. an IP with high AbuseIPDB score or a MISP match) to justify changes to:
  - Splunk correlation rule thresholds,
  - WatchGuard IPS/firewall policy (e.g. stricter rule or new block),
  - severity/tagging mapping in Splunk.
- Document the chain: intel source → configuration change → observed effect (e.g. fewer noisy alerts, stronger blocking).

---

## Stage 4 – Anomaly Detection and Correlation (S26)

### Correlation rules using WatchGuard IPS alerts
- Design 2–3 Splunk correlation searches based on WatchGuard IPS “Action: drop” events, for example:
  - Multiple IPS drops from the same `src_ip` against different internal hosts within N minutes.
  - IPS drops on HTTP/HTTPS to a critical web server from unusual countries/ASNs.
  - Spikes in certain `rule_id` values linked to known exploit attempts.
- For each rule, record:
  - purpose,
  - fields used (IPs, ports, rule ID, action, policy name),
  - time window and thresholds,
  - rationale for why it indicates suspicious behaviour.

### Link correlation to enrichment
- Extend these searches to:
  - increase severity when `src_ip` (or `dest_ip`) has a high AbuseIPDB confidence score;
  - flag events where IPs match MISP indicators or tags.
- Keep sample events where the presence of threat intel changed triage (e.g. similar IPS alert, but only the enriched/known‑bad one opens a case).

---

## Stage 5 – Automated Triage and Ticket Creation (S21)

### TheHive setup
- Install or provision a TheHive instance (VM/container/server).
- Configure:
  - organisation, users, and basic roles for analysts/support desk;
  - a dedicated **“WatchGuard IPS incident”** case template with fields for:
    - source/destination IPs and ports,
    - rule ID and policy name,
    - enrichment results (AbuseIPDB, MISP),
    - actions taken and outcome.
- Generate an API key for Splunk to create cases programmatically.

### SIEM → TheHive integration
- In Splunk, configure alert actions or a script that:
  - triggers on your chosen high‑severity correlation searches;
  - calls TheHive’s API to create a new case using the “WatchGuard IPS incident” template;
  - populates fields with log details and enrichment results.
- Test with a controlled IPS alert to ensure cases appear correctly in TheHive with all required context.

### Formal Incident Response Plan
- Write an Incident Response Plan organised around NIST CSF phases (Identify, Protect, Detect, Respond, Recover).
- For each phase, define:
  - what the support desk/analyst does when a WatchGuard IPS alert is raised;
  - where they check enrichment (Splunk dashboards, AbuseIPDB, MISP, TheHive case data);
  - how and when they escalate or change firewall/IPS policies and Splunk rules;
  - documentation and communication expectations inside the case.

---

## Stage 6 – Incident Demonstration (S30)

### Run the WatchGuard‑based scenario
- Simulate or wait for a suitable non‑major IPS “Action: drop” event against a key service that matches your planned scenario.
- Confirm the end‑to‑end flow:
  - WatchGuard generates the IPS alert;
  - Splunk ingests and correlates it and applies enrichment;
  - TheHive case is automatically created with full context.

### Apply the Incident Response Plan
- Follow the Incident Response Plan step‑by‑step:
  - confirm detection in Splunk using the IPS alert and related events;
  - review AbuseIPDB/MISP enrichment;
  - decide on containment/monitoring actions (e.g. tighter IPS rule, IP/range block, monitoring only);
  - record all actions, rationales and communications in TheHive.

### Incident manager report
- Write an incident manager‑style report for this scenario, covering:
  - overview and timeline;
  - detection path (from WatchGuard alert to Splunk correlation to TheHive case);
  - threat intel used and how it influenced decisions;
  - final actions and lessons learned.

---

## Stage 7 – Root Cause Analysis and Tuning (S29)

### Root cause analysis for several events
- Select:
  - the main IPS‑based incident;
  - at least one noisy IPS alert that turned out to be a false positive;
  - at least one case where logs indicate a near‑miss/late detection (potential false negative).
- For each, perform RCA:
  - underlying technical cause;
  - how monitoring/correlation behaved;
  - where enrichment helped or failed.

### Reducing false positives and false negatives
- Tune:
  - WatchGuard IPS/firewall policies (e.g. exceptions or stricter signatures);
  - Splunk correlation thresholds and time windows;
  - severity mappings based on intel (e.g. high‑risk IPs always high severity).
- Capture before/after examples demonstrating:
  - reduced false positives;
  - improved detection of meaningful IPS events.

---

## Stage 8 – Final Write‑Up and Declarations

### Project report structure
- Suggested sections:
  - Introduction and project description.
  - Architecture and WatchGuard/Splunk monitoring configuration (S28).
  - Threat intelligence and correlation with MISP/AbuseIPDB (S25).
  - Anomaly detection and IPS‑based correlation rules (S26).
  - Incident Response Plan (S21).
  - IPS incident demonstration and incident manager report (S30).
  - Root cause analysis and tuning (S29).
- Clearly signpost where each KSB/outcome and Defend & Respond requirement is evidenced.

### Declarations
- Complete and sign the Scenario Demonstrations declaration and the Project Report declaration with your employer/responsible person.
- Ensure all evidence is your own work and sources are acknowledged.
