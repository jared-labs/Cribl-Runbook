# Log Routing: Cribl Stream

## Overview

This document summarizes a Cribl Stream deployment used as the central log routing layer in a home lab security stack. It is written for a portfolio audience, focusing on architecture, pipeline design, and the role Cribl plays between log sources and the SIEM.

Cribl Stream acts as a data broker: it receives events from multiple sources (vulnerability scanners, syslog, automation pipelines), applies lightweight transformations or enrichment, and routes them to Graylog for search, correlation, and alerting.

## Environment

| Field | Value |
|-------|-------|
| System | VMCRIB01 |
| IP Address | (internal) |
| OS | Ubuntu Server 24.04 (minimal) |
| vCPU / RAM / Disk | 2 (4+ rec) / 4 GB / 20–40 GB |
| Key Ports | 9000 (Web UI), 10090 (TCP test), 20020 (TCP JSON in) |
| Install Method | Manual tarball under /opt/cribl |
| Output Protocol | GELF HTTP to Graylog on port 12201 |
| Service | systemd (cribl.service) |

## Architecture

```text
┌────────────────────────────────────────────────────────────────┐
│  Log Sources                                                    │
│                                                                │
│  OpenVAS exporter ──► TCP JSON (port 20020)                    │
│  rsyslog hosts    ──► Syslog UDP/TCP                           │
│  MISP IOC export  ──► Lookup table enrichment                  │
│  Omada controller ──► Syslog                                    │
└───────────────────────────────┬────────────────────────────────┘
                                │
                                ▼
┌────────────────────────────────────────────────────────────────┐
│  VMCRIB01 — Cribl Stream                                        │
│                                                                │
│  Sources:                                                       │
│  ├── TCP JSON (20020) — structured security events              │
│  ├── TCP (10090) — test/generic events                          │
│  └── Syslog (514/1514) — system logs                           │
│                                                                │
│  Pipelines:                                                     │
│  ├── openvas_passthrough — route vuln data to SIEM              │
│  ├── misp_enrich — IOC lookup against MISP export tables        │
│  └── default — catch-all routing                                │
│                                                                │
│  Destinations:                                                  │
│  └── graylog_gelf — GELF HTTP to vmgray01:12201                 │
└───────────────────────────────┬────────────────────────────────┘
                                │
                                ▼
┌────────────────────────────────────────────────────────────────┐
│  VMGRAY01 — Graylog SIEM                                        │
│  GELF HTTP input on port 12201                                  │
│  JSON flattening pipeline → searchable fields                   │
└────────────────────────────────────────────────────────────────┘
```

## Design Decisions

- **Cribl as relay, not heavy transformer:** The exporter scripts (OpenVAS, MISP) handle normalization before sending to Cribl. Cribl adds routing metadata and lookup enrichment but avoids deep parsing. This keeps the pipeline evolvable without redeploying Cribl for every schema change.
- **Manual tarball over install script:** The official `sh.cribl.io/install.sh` script failed on this environment (no `/opt/cribl`, no systemd unit created). The tarball method is reliable and reproducible.
- **GELF HTTP output to Graylog:** GELF is Graylog's native structured format. Using it avoids the JSON flattening step that raw TCP inputs require.
- **Lookup table enrichment (MISP IOCs):** MISP exports IP and domain IOCs as CSV files that Cribl loads as lookup tables. Events matching known IOCs get enriched in-flight before reaching the SIEM — no extra Graylog pipeline needed.
- **Single-worker deployment:** The lab's event volume (~1K events/hour) doesn't justify a distributed Cribl deployment. A single worker on one VM handles the full load.

## Integration Points

| Integration | Direction | Protocol | Purpose |
|-------------|-----------|----------|---------|
| OpenVAS → Cribl | Inbound | TCP JSON (20020) | Vulnerability findings |
| MISP → Cribl | File sync | CSV lookup tables | IOC enrichment |
| rsyslog → Cribl | Inbound | Syslog | System events |
| Cribl → Graylog | Outbound | GELF HTTP (12201) | All events to SIEM |

## Quirks and Gotchas

- **Scripted installer is broken on this VM:** `curl https://sh.cribl.io/install.sh | sudo bash` fails silently. Always use the manual tarball method.
- **Two destination options for GELF:** Cribl may or may not have a native GELF destination type depending on version. Fall back to generic HTTP POST to `/gelf` if needed.
- **netcat variant matters:** Use `netcat-openbsd` for testing (`-q0` flag). The `netcat-traditional` package behaves differently.
- **Commit + Deploy workflow:** Cribl separates configuration authoring from deployment. Changes in the UI are not live until you Commit and then Deploy to the worker.

## What This Demonstrates

This deployment shows practical log routing architecture: source diversity, protocol bridging, in-flight enrichment with external threat intelligence, and structured delivery to a SIEM. The design prioritizes operational simplicity (single node, minimal transformation in the relay) while still providing the routing flexibility that a multi-source security environment needs.

---

Sanitized for public portfolio use.

For step-by-step deployment instructions, see [OPERATIONS.md](./OPERATIONS.md).
