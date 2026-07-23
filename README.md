# Log Routing: Cribl Stream

## Overview

This document summarizes a Cribl Stream deployment used as the central log routing layer in a home lab security stack. It is written for a portfolio audience, focusing on architecture, pipeline design, and the role Cribl plays between log sources and the SIEM.

Cribl Stream acts as a data broker: it receives events from multiple sources (syslog from 24+ Linux hosts, vulnerability scanners, network device telemetry, NAS storage events), applies lightweight transformations, noise filtering, and threat-intel enrichment, then routes structured JSON to Graylog via dedicated TCP outputs — one per source category.

## Environment

| Field | Value |
|-------|-------|
| System | VMCRIB01 |
| IP Address | (internal) |
| OS | Ubuntu Server 24.04 (minimal) |
| vCPU / RAM / Disk | 2 / 4 GB / 20 GB |
| Key Ports | 9000 (Web UI), 514/1514 (syslog), 20000 (Omada), 20010 (Proxmox), 20020 (OpenVAS JSON) |
| Install Method | Manual tarball under /opt/cribl |
| Output Protocol | TCP JSON to Graylog (per-source ports) |
| Service | systemd (cribl.service) |
| Version | 4.18.x (free tier, single worker) |

## Architecture

```text
┌────────────────────────────────────────────────────────────────────┐
│  Log Sources                                                        │
│                                                                    │
│  24 Linux Hosts ──── rsyslog TCP ──────┐                           │
│  8 Proxmox Hosts ── rsyslog TCP ───────┼──► Syslog :1514          │
│  TrueNAS ─────────── syslog UDP ──────────► Syslog :1516          │
│  Omada SDN ────────── syslog TCP ─────────► Syslog :20000         │
│  Network Devices ─── syslog UDP ──────────► Syslog :514           │
│  OpenVAS exporter ── NDJSON TCP ──────────► TCP JSON :20020       │
│  OpenCanary ──────── direct JSON TCP ─────► Graylog :1514 (bypass)│
└────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌────────────────────────────────────────────────────────────────────┐
│  VMCRIB01 — Cribl Stream 4.18                                       │
│                                                                    │
│  Inputs:                                                            │
│  ├── linux-syslog (TCP/UDP :1514) — 24 Linux hosts + Proxmox      │
│  ├── truenas-syslog (UDP :1516) — NAS storage events               │
│  ├── omada-sdn (syslog :20000) — SDN controller IDS/IPS flows     │
│  ├── network-syslog (UDP :514, TCP :1515) — router/APs (future)   │
│  └── openvas-json (TCP JSON :20020) — vulnerability scan results   │
│                                                                    │
│  Pipelines (per-source processing):                                 │
│  ├── linux-syslog — tag source_host, drop CRON/DHCP/rsyslog noise  │
│  ├── truenas-syslog — drop SMART OK, routine NFS mount noise       │
│  ├── omada-sdn — regex parse AP flows, MISP IOC lookup, enrich     │
│  ├── network-syslog — drop DHCP/NTP, tag router vs AP              │
│  └── open-vas — passthrough (already structured JSON)              │
│                                                                    │
│  Outputs (TCP JSON, one per Graylog input):                         │
│  ├── linux-syslog-to-graylog ──────► Graylog :5140                 │
│  ├── truenas-syslog-to-graylog ────► Graylog :5141                 │
│  ├── network-syslog-to-graylog ────► Graylog :5142                 │
│  ├── omada-sdn-to-graylog ─────────► Graylog :20005               │
│  ├── proxmox-to-graylog ──────────► Graylog :20015                │
│  └── openvas-to-graylog ──────────► Graylog :20025                │
│                                                                    │
│  Noise Reduction (~40% drop rate):                                  │
│  • CRON session open/close                                         │
│  • DHCP lease renewals                                             │
│  • NTP synchronization                                             │
│  • rsyslog reconnect/resume messages                               │
│  • Routine SMART OK checks (TrueNAS)                               │
│  • Multicast/broadcast traffic (Omada)                             │
└───────────────────────────────┬────────────────────────────────────┘
                                │
                                ▼
┌────────────────────────────────────────────────────────────────────┐
│  VMGRAY01 — Graylog 7.1 SIEM                                       │
│                                                                    │
│  7 dedicated inputs (Raw TCP / Syslog TCP)                         │
│  11 streams with content-based routing                              │
│  Pipeline rules extract structured fields per source type          │
│  69 event definitions for detection and alerting                   │
└────────────────────────────────────────────────────────────────────┘
```

## Design Decisions

- **TCP JSON outputs, not GELF.** Earlier iterations used GELF HTTP. The current design uses Cribl's `tcpjson` output type with one output per source category. This gives Graylog separate inputs per source (enabling input-based stream routing) and avoids GELF's field naming constraints (`_` prefix requirement). Each output maps 1:1 to a Graylog Raw TCP input.

- **Cribl as relay + noise filter, not heavy transformer.** Source scripts handle normalization before sending. Cribl's role is routing, noise removal (~40% of events dropped before indexing), and lightweight enrichment (MISP IOC lookups). This keeps the pipeline evolvable without Cribl redeployments for schema changes.

- **Per-source pipelines with dedicated routes.** Each input has a named pipeline and output. This provides independent control: you can disable, tune thresholds, or add enrichment to one source without affecting others. Routes match on `__inputId` prefix.

- **MISP IOC enrichment on Omada flows.** MISP exports IP and domain IOCs as CSV lookup tables that Cribl loads in-memory. Every Omada SDN flow event gets checked against the IOC list in-flight. Hits get `misp_hit: true` + indicator metadata before reaching Graylog — no extra SIEM processing needed.

- **rsyslog forwarding, not agents.** Every Linux host already has rsyslog. No new software to install, no agent licensing. Ansible deploys a 5-line config file that forwards `auth,authpriv.*`, `*.err`, `kern.*`, and service journals to Cribl.

- **Single-worker free tier.** Lab event volume (~5-10K events/hour across all sources) doesn't justify a distributed deployment or paid license. One worker handles the full load with headroom.

## Integration Points

| Integration | Direction | Protocol | Port | Purpose |
|-------------|-----------|----------|------|---------|
| Linux Hosts → Cribl | Inbound | Syslog TCP | 1514 | Auth, kernel, service logs |
| Proxmox → Cribl | Inbound | Syslog TCP | 1514 | Hypervisor events (shares linux-syslog input) |
| TrueNAS → Cribl | Inbound | Syslog UDP | 1516 | NAS storage/health events |
| Omada SDN → Cribl | Inbound | Syslog TCP | 20000 | IDS/IPS flow logs from controller |
| OpenVAS → Cribl | Inbound | TCP JSON | 20020 | Vulnerability scan findings |
| MISP → Cribl | File sync | CSV lookup | — | IOC enrichment tables |
| Cribl → Graylog | Outbound | TCP JSON | 5140-5142, 20005, 20015, 20025 | All events to SIEM |

## Omada SDN Pipeline (Deep Dive)

The most complex pipeline. Omada's controller logs IDS/IPS-style flow data:

1. **Regex extraction** — Parses raw syslog into structured fields: `ap_mac`, `client_mac`, `src_ip`, `dst_ip`, `ip_proto`, `src_port`, `dest_port`
2. **Noise drop** — Removes multicast (224.x), broadcast (255.255.255.255), and null-byte events
3. **Field normalization** — Renames to consistent schema, converts port strings to integers, maps protocol numbers to names (6→tcp, 17→udp)
4. **MISP IOC lookup** — Checks `dst_ip` and `src_ip` against exported MISP indicators. Adds `misp_hit`, `misp_dst_hit`, `misp_src_hit`, indicator metadata (category, type, event_id)
5. **Metadata** — Adds `vendor: tp-link`, `product: omada_sdn`, `cribl_pipe: omada-sdn`

Result: every network flow event arrives at Graylog as a flat JSON object ready for field-level search and correlation.

## Quirks and Gotchas

- **Manual tarball install required.** The official `sh.cribl.io/install.sh` script failed on this environment. The tarball method (`tar xf cribl-*.tgz -C /opt && /opt/cribl/bin/cribl boot-start`) is reliable and reproducible.
- **Free tier blocks user management via API.** Can't create users programmatically. Local auth uses HMAC-SHA256. Deleting auth files + `cribl.secret` forces a password reset.
- **Commit + Deploy workflow.** UI changes aren't live until you Commit and Deploy to the worker. Config files in `local/` are the source of truth after deployment.
- **`sendHeader` on tcpjson outputs.** If enabled, Cribl sends a JSON metadata handshake when connecting. Graylog Raw TCP inputs treat this as a message. Disable `sendHeader` to avoid junk events.
- **netcat testing.** Use `netcat-openbsd` for testing (`-q0` flag). The `netcat-traditional` package behaves differently with TCP JSON framing.
- **Omada regex is fragile.** The SDN controller's log format changes between firmware versions. The regex pattern needs updating when Omada updates its IDS log format.

## What This Demonstrates

- **Log pipeline architecture** — source diversity (syslog, JSON, direct TCP), protocol bridging, noise engineering, structured delivery to SIEM
- **In-flight threat intelligence enrichment** — MISP IOC correlation happens at the routing layer, not the SIEM, reducing detection latency
- **Noise engineering** — knowing what to drop is as important as what to collect. ~40% reduction in indexed events without losing security signal
- **Per-source isolation** — each log source gets its own pipeline, output, and Graylog input. Problems in one don't cascade to others
- **Infrastructure as code** — Ansible deploys rsyslog configs to all hosts; Cribl routes are version-controlled YAML

---

Sanitized for public portfolio use. Real IPs replaced with `(internal)` notation.
