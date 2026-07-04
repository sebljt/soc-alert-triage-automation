# Intelligence-Driven Alert Correlation & Triage Automation

**An engineering case study — automating firewall-to-case security operations in a live SME managed-service environment.**

> This is a sanitised write-up of a system I designed and deployed in production. All
> hostnames, IP addresses, credentials, account names and organisation-specific details
> have been replaced with generic placeholders. The architecture, engineering decisions and
> outcomes are unchanged.

---

## Summary

The environment relied on a next-generation UTM firewall and Splunk to monitor
internet-facing services. Firewall security events were visible in Splunk, but there was no
consistent path from *event* to *investigated case*: engineers copied details into tickets
by hand and ran ad-hoc reputation lookups, which was slow, inconsistent and hard to audit.

I built an end-to-end pipeline that turns firewall IPS events into enriched, triaged cases
automatically — while deliberately keeping a human in the loop for any blocking decision:

```
UTM firewall ──syslog──▶ SC4S ──▶ Splunk ──▶ TheHive ──▶ Cortex ──▶ Microsoft Teams
 (IPS events)           (collector) (correlation) (case mgmt) (enrichment) (SOC notification)
```

A Python service polls TheHive for new alerts, enriches each source IP through Cortex
(AbuseIPDB + VirusTotal), promotes anything above a reputation threshold into a full case
with a standard task list, and posts a formatted card to the support team's Teams channel.
The automation **never changes the firewall itself** — analysts make the final block/no-block
call, following a documented response plan.

## Design principles

- **Human-in-the-loop for anything destructive.** The system enriches, correlates and
  routes; it does not auto-block. Automatic firewall changes against production traffic were
  explicitly out of scope.
- **Least privilege.** The automation authenticates with a scoped API key that can create
  and update cases but cannot change global settings. Analyst and administrator roles are
  separated.
- **Consistency over cleverness.** Every case starts from the same template and task list,
  so triage is repeatable and produces a clean audit trail — the actual business problem
  being solved.

## Architecture

| Component | Role | Deployment |
|---|---|---|
| UTM firewall | Perimeter firewall / IPS; log source | Network edge |
| SC4S | Splunk Connect for Syslog; syslog collector | Linux host |
| Splunk | SIEM; ingestion and correlation searches | Linux host |
| TheHive 5 | Case management and triage | Docker on Linux host |
| Cortex 4 | Observable enrichment | Docker on Linux host |
| `auto_enrich` (Python) | Automation between TheHive, Cortex and Teams | Scheduled job |
| Microsoft Teams | SOC notification channel | Microsoft 365 tenant |

Internal service endpoints in this write-up use placeholder hostnames (e.g.
`http://thehive.internal:9000` for TheHive, `:9001` for Cortex). Real addressing is not published.

## Pipeline detail

**1. Ingestion & correlation (Splunk).** Firewall IPS events are shipped via syslog through
SC4S into a dedicated index. Field extractions expose source IP, destination IP, port,
policy name and rule ID. A correlation search groups repeated IPS hits by source IP and rule
so that sustained activity from a single origin stands out, and raises a Splunk alert.

**2. Alert to case (TheHive).** Splunk alerts land in TheHive as alert objects. A case
template and a standard **IPS Block** task list capture the key firewall fields and break the
investigation into consistent steps: triage, enrichment review, SIEM searches, risk
classification and response.

**3. Enrichment (Cortex).** For each source IP, the automation runs the AbuseIPDB analyser
(returning an Abuse Confidence Score, 0–100) and VirusTotal analysers for additional context,
then reads the score back from the Cortex job report.

**4. Promotion & notification.** If the score meets or exceeds a threshold (50), the service
promotes the alert to a case, attaches the IPS Block task list, and posts a Teams Adaptive
Card to the support channel. Below threshold, it marks the alert handled so it is not
reprocessed — avoiding alert fatigue rather than adding to it.

**5. Human response.** The Teams card is an early heads-up; the real work happens in TheHive.
Analysts confirm the finding, review Splunk history, identify affected hosts, decide on
containment, and — only then — add the IP to the firewall's blocked list, recording the case
reference as the reason. The case closes once monitoring confirms the activity has stopped.

## The automation service (sanitised)

Written in Python 3 with `requests`, `json` and `time` for a small, maintainable footprint,
run on a schedule (once per minute) so new alerts are picked up quickly without a
long-running daemon. Logging is written to a local file for review and troubleshooting.

Configuration is isolated at the top of the module. **Secrets are supplied via environment
variables, never hard-coded** — in the sanitised form:

```python
import os

THEHIVE_URL   = os.environ["THEHIVE_URL"]     # e.g. http://thehive.internal:9000
THEHIVE_KEY   = os.environ["THEHIVE_KEY"]      # scoped API key
CORTEX_URL    = os.environ["CORTEX_URL"]       # e.g. http://cortex.internal:9001
CORTEX_KEY    = os.environ["CORTEX_KEY"]
ABUSEIPDB_ID  = os.environ["ABUSEIPDB_ANALYSER_ID"]
TEAMS_WEBHOOK = os.environ["TEAMS_WEBHOOK"]    # Adaptive Card destination
SCORE_THRESHOLD = 50
```

> **Engineering note for reviewers:** in the original production build the keys were held in
> a config block. Moving them to environment variables (shown above) is the correct pattern
> and is how any published version should ship — no live credential ever belongs in source
> control.

Core loop, in brief:

1. Query TheHive for alerts with status `New`.
2. For each alert, extract IP observables.
3. For each IP, run the AbuseIPDB analyser in Cortex and poll for the result.
4. Read the Abuse Confidence Score from the Cortex report.
5. If `score >= SCORE_THRESHOLD`: promote to case, attach the IPS Block task list, send the
   Teams card.
6. Otherwise: mark the alert handled so it is not processed again.

## Outcome

- Firewall IPS activity now follows a single, consistent path from detection to documented
  case, replacing manual, per-engineer handling.
- Every incident carries the same triage structure and a clear audit trail suitable for
  review by auditors or customers.
- Time-to-detect and time-to-acknowledge dropped to seconds/minutes as measured in the case
  metadata during testing.
- The design surfaced root causes such as repeated scanning of public services and
  short-lived test systems left exposed — feeding back into hardening.

## Skills demonstrated

Monitoring architecture in a live environment (syslog → SIEM → case management); multi-source
correlation (firewall, SIEM, threat intelligence, chat); REST API automation across TheHive
and Cortex; containerised deployment (Docker); incident-response process design; and root
cause analysis.

## Screenshots

Screenshots from the production system are **omitted** from the public version. The original
report's figures contained live internal addressing, account names and third-party tool
references. Any figures added here should be either (a) cropped tightly and redacted with
flattened, irreversible boxes, or (b) reproduced from a synthetic lab rebuild.

---

*Sebastian Waldron — completed as a Level 4 Cyber Security Technologist end-point assessment
project. Published as a sanitised case study; no client-identifying information is included.*
