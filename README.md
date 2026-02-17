# Build Plan – Phase 1 (MISP + TheHive First)

Goal: Get MISP and TheHive fully deployed and talking to each other, then add Splunk and WatchGuard data once enrichment and case management are ready.

---

## Stage 1 – Core Platforms First

### 1.1 Deploy MISP

- Provision a small Linux VM (Ubuntu/Debian).
- Install MISP following the official or community guide:
  - Install dependencies (web server, DB, PHP, Redis, Git, Python). [web:52]
  - Clone the MISP repo into `/var/www/MISP` and run the install script. [web:52]
  - Set base URL (e.g. `https://misp.lab.local`) and create the admin account.
- Log in to the MISP web UI and:
  - set organisation name and contact details;
  - enable at least one feed or create a small test event with a couple of IP IOCs. [web:52][web:55]

### 1.2 Configure MISP API Access

- In MISP: **Administration → List Auth Keys → Add authentication key**. [web:6]
- Create a dedicated key for integrations (e.g. “TheHive‑integration”).
- Record:
  - MISP URL (e.g. `https://misp.lab.local`),
  - auth key (store securely, not in Git).

---

## Stage 2 – Deploy and Configure TheHive

### 2.1 Install TheHive

- Provision a VM or container for TheHive.
- Install TheHive 5 (or your chosen version) with its required back‑end (e.g. Cassandra/Elasticsearch).
- Confirm you can reach the TheHive web UI in a browser.

### 2.2 Initial TheHive setup

- Create:
  - an organisation,
  - at least one analyst/support user,
  - an admin account for configuration.
- Define basic roles/permissions so only admins can change system config.

### 2.3 Create case templates

Create a **“Generic TI Case”** template and a **“WatchGuard IPS Incident”** template:

- Generic TI Case:
  - Fields: IOC type, source (MISP feed name), risk level, notes.
- WatchGuard IPS Incident:
  - Fields: src IP, dest IP, ports, protocol, rule ID, policy name, enrichment (AbuseIPDB/MISP), actions taken, outcome.

These templates will be used later when Splunk starts creating cases.

---

## Stage 3 – Connect TheHive to MISP (core integration)

### 3.1 Configure MISP server in TheHive

- On the TheHive host, open `application.conf` (or equivalent config): [web:6][web:15]

  - Add a block similar to:

    ```hocon
    misp {
      interval: 2m
      servers: [
        {
          name = "Lab-MISP"
          url  = "https://misp.lab.local"
          auth {
            type = key
            key  = "PASTE_MISP_API_KEY_HERE"
          }
          wsConfig {
            ssl {
              # adjust SSL/trust as needed for your lab
            }
          }
        }
      ]
    }
    ```

- Replace URL and key with your lab values.
- Restart TheHive service.

### 3.2 Verify MISP ↔ TheHive link

- In TheHive UI, check the MISP section:
  - confirm the server appears as “connected” / green. [web:6]
- Test pull:
  - publish a small test event in MISP,
  - ensure TheHive can list or import MISP events as alerts.
- Test push:
  - create a dummy case in TheHive,
  - add a couple of IP observables,
  - mark them as IOCs and export to MISP into a new event. [web:15]

At this point:
- MISP has data,
- TheHive can pull from and push to MISP,
- You have case templates ready for later Splunk automation.

---

## Stage 4 – Plan Splunk Integration (design only for now)

*(Implementation later, once MISP+TheHive are stable.)*

### 4.1 Decide data flow

- Target future flow:
  - WatchGuard IPS/syslog → Splunk.
  - Splunk correlations → enrichment (MISP feed data + AbuseIPDB).
  - High‑severity results → create cases in TheHive (using “WatchGuard IPS Incident” template).
  - Confirmed IOCs in cases → pushed back into MISP.

Document this design now so your project report shows clear intent even before Splunk is configured.

### 4.2 Define what Splunk will use from MISP/TheHive

- From MISP:
  - IP/domain IOCs and tags pulled as:
    - CSV/JSON exports into Splunk lookups, or
    - direct API queries (later).
- From TheHive:
  - Case IDs and attributes for:
    - manual reporting,
    - root cause analysis.

---

## Stage 5 – Evidence and Repo Structure

### 5.1 Capture evidence for this phase

- Screenshots (for later report):
  - MISP dashboard with feeds or test events.
  - MISP auth keys page (with key redacted).
  - TheHive UI showing:
    - organisations and users,
    - case templates,
    - connected MISP server,
    - a test case created from MISP data and a case exported to MISP.

### 5.2 Store in your private GitHub repo

- Files to add now:
  - `plan/Phase1-MISP-TheHive.md` (this file).
  - `misp/setup-notes.md` – commands/config you ran (no secrets).
  - `thehive/setup-notes.md` – install/config notes (no secrets).
- Create a simple checklist in `README.md` for Phase 1:

  ```markdown
  ## Phase 1 – MISP + TheHive

  - [ ] MISP VM deployed and reachable
  - [ ] MISP admin configured and test event created
  - [ ] MISP API key created for integrations
  - [ ] TheHive installed and UI reachable
  - [ ] Org/users/roles created in TheHive
  - [ ] Case templates created
  - [ ] MISP server configured in TheHive
  - [ ] Pull from MISP to TheHive tested
  - [ ] Push from TheHive to MISP tested
