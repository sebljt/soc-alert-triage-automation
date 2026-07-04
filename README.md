# SOC Alert Correlation & Triage Automation

Automating the path from firewall IPS event to enriched, triaged security case in a live
SME environment — using Splunk, TheHive, Cortex and Microsoft Teams, glued together with a
small Python service.

**→ Full write-up: [`soc-case-study.md`](./soc-case-study.md)**

## What it does

```
UTM firewall ──syslog──▶ SC4S ──▶ Splunk ──▶ TheHive ──▶ Cortex ──▶ Microsoft Teams
```

A scheduled Python service polls TheHive for new alerts, enriches each source IP through
Cortex (AbuseIPDB + VirusTotal), promotes anything above a reputation threshold into a full
case with a standard task list, and notifies the SOC channel in Teams. Blocking decisions
stay with a human analyst — the automation never touches the firewall.

## Highlights

- End-to-end pipeline: detection → correlation → enrichment → case → notification
- Human-in-the-loop by design; no automated changes to production traffic
- Least-privilege API access; scoped keys supplied via environment variables
- Consistent, auditable triage via a reusable case template and task list

## Tech

`Splunk` · `SC4S` · `TheHive 5` · `Cortex 4` · `Python 3` · `Docker` · `Microsoft Teams`

## Sanitisation

This repository is a **sanitised** account of a system deployed in production. All
hostnames, IP addresses, credentials, account names and organisation-specific details have
been replaced with generic placeholders. No live credential appears anywhere in this
repository or its history.

## Author

Sebastian Waldron — Level 4 Cyber Security Technologist end-point assessment project.
