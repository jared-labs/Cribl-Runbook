# VMCRIB01 — Cribl Stream → Graylog (GELF HTTP)

> Single-VM deployment of Cribl Stream on Ubuntu Server 24.04, running on Proxmox, forwarding logs to Graylog (`vmgray01`) via GELF HTTP on port `12201`.

---

## At a Glance

| Field | Value |
|-------|-------|
| VM/CT Name | `VMCRIB01` |
| Host Node | Proxmox cluster |
| IP Address | `<vmcrib01-ip>` |
| OS | Ubuntu Server 24.04 (minimal) |
| vCPU / RAM / Disk | 2 (4+ recommended) / 4 GB / 20–40 GB |
| Key Ports | `9000` (UI), `10090` (TCP test), `12201` (outbound GELF) |
| Credentials | Bitwarden: "VMCRIB01" |
| Depends On | Proxmox, Graylog (`vmgray01`) |

> **Warning:** `vmcrib01` runs **Cribl Stream only**. Graylog, OpenSearch, and MongoDB stay on `vmgray01`.

> **Warning:** **Do not** use the Cribl `https://sh.cribl.io/install.sh` script on this box — it failed to create `/opt/cribl` and `cribl.service` here. Use the manual tarball method in this runbook.

---

## Prerequisites

- [ ] Proxmox node with available resources (2 vCPU, 4 GB RAM, 20–40 GB disk)
- [ ] Network: static IP reserved or DHCP reservation configured on `vmbr0`
- [ ] Ubuntu Server 24.04 ISO uploaded to Proxmox storage
- [ ] Graylog (`vmgray01`) running and accessible on the LAN

---

## 0 — Create the VM in Proxmox

Create a new VM on your Proxmox cluster with the following settings:

| Setting | Value |
|---------|-------|
| Name | `VMCRIB01` |
| Guest OS | Linux → Ubuntu → 24.04 |
| CPU | 2 vCPUs (4+ recommended for higher volume) |
| RAM | 4 GB (2 GB minimum, 4–8 GB recommended) |
| Disk | 20–40 GB on `local-lvm` (or your thin pool) |
| BIOS / Machine | Cluster standard (e.g., `OVMF (UEFI)` or `SeaBIOS`, `q35` if preferred) |
| Network | VirtIO on `vmbr0` (flat LAN) |
| QEMU Guest Agent | Enabled |

Install **Ubuntu Server 24.04 (minimal)** inside this VM.

---

## 1 — Base OS Setup

Log in as your normal sudo user on `vmcrib01`.

### 1.1 Update packages & install base tools

```bash
sudo apt update
sudo apt -y full-upgrade

# Helpful base packages
sudo apt install -y qemu-guest-agent vim curl tar ca-certificates netcat-openbsd

# Enable guest agent for Proxmox
sudo systemctl enable --now qemu-guest-agent
```

Reboot if the kernel was updated:

```bash
sudo reboot
```

### 1.2 Set hostname and fix /etc/hosts

```bash
hostnamectl set-hostname vmcrib01

sudo sed -i 's/^127\.0\.1\.1.*/127.0.1.1\tvmcrib01/' /etc/hosts
grep '^127\.0\.1\.1' /etc/hosts
hostnamectl
```

> **Note:** If you changed the hostname after install, log out and back in so your shell prompt matches.

---

## 2 — Install Cribl Stream (Manual Tarball)

> **Warning:** The official script `curl https://sh.cribl.io/install.sh | sudo bash` previously **failed** on this environment (no `/opt/cribl`, no `cribl.service`). **Always** use the tarball method below on `vmcrib01`.

### 2.1 Create `/opt/cribl` and set ownership

```bash
sudo mkdir -p /opt/cribl
sudo chown "$USER":"$USER" /opt/cribl
cd /opt/cribl
```

### 2.2 Download latest Cribl Stream tarball

```bash
curl -L "$(curl -s https://cdn.cribl.io/dl/latest)" -o cribl.tgz
```

### 2.3 Extract and clean up

```bash
tar xzf cribl.tgz --strip-components=1
rm cribl.tgz
```

You should now have:

- Binaries in `/opt/cribl/bin`
- Config in `/opt/cribl/local`

Quick sanity check:

```bash
ls -l /opt/cribl/bin
/opt/cribl/bin/cribl version
```

---

## 3 — Register Cribl with systemd (Boot-Start)

### 3.1 Enable boot-start

```bash
cd /opt/cribl
sudo /opt/cribl/bin/cribl boot-start enable -m systemd
```

This creates `/etc/systemd/system/cribl.service`. Verify:

```bash
grep -n 'cribl' /etc/systemd/system/cribl.service
sudo systemctl daemon-reload
```

### 3.2 Enable and start the service

```bash
sudo systemctl enable cribl
sudo systemctl start cribl
sudo systemctl status cribl --no-pager
```

Expected output:

- `Active: active (running)`
- No errors about missing `/opt/cribl`

> **Note:** If `cribl.service` is missing or `/opt/cribl` is not found, re-run Section 2 and 3.1. Do **not** fall back to the `sh.cribl.io` script — that's the known-bad path on this VM.

---

## 4 — Networking & Firewall

Cribl uses:

| Port | Protocol | Purpose |
|------|----------|---------|
| `9000` | TCP | Cribl Web UI |
| `10090` | TCP | TCP test source (optional) |
| `12201` | TCP (outbound) | GELF HTTP to `vmgray01` |

If `ufw` is enabled, open ports:

```bash
sudo ufw allow 9000/tcp comment 'Cribl UI'
sudo ufw allow 10090/tcp comment 'Cribl TCP test input (optional)'
sudo ufw reload
```

Verify listeners:

```bash
sudo ss -ltnp | grep -E '9000|10090' || echo "No Cribl listeners yet"
```

---

## 5 — Cribl Web UI First Login

Once the `cribl` service is running, access the UI from your browser:

- `http://<vmcrib01-ip>:9000/`
- Or with DNS: `http://vmcrib01.lan:9000/`

Log in with the credentials set during initial Cribl setup (or the defaults from Cribl docs if fresh).

Confirm **Status → Workers** shows your local worker running.

---

## 6 — Configure Graylog GELF HTTP Input (on VMGRAY01)

On `vmgray01` (Graylog Web UI):

1. Go to **System → Inputs**
2. Select **GELF HTTP** from the dropdown
3. Click **Launch new input**
4. Configure:
   - **Title:** `gelf_http_from_cribl`
   - **Node / Global:** `Global` (or bound to the Graylog node you prefer)
   - **Bind address:** `0.0.0.0`
   - **Port:** `12201`
5. Click **Save**
6. Confirm the new GELF HTTP input shows as **Running**

> **Note:** In this homelab, Cribl treats Graylog as a GELF HTTP endpoint on `vmgray01:12201`.

---

## 7 — Cribl → Graylog Wiring (Destination + Route)

### 7.1 Create destination `graylog_gelf`

In Cribl UI on `vmcrib01`:

**Case A: GELF destination type exists**

1. Go to **Destinations**
2. Click **New Destination**
3. Select **GELF HTTP** (or Cribl's GELF-type destination)
4. Configure:
   - **Name:** `graylog_gelf`
   - **Host:** `vmgray01`
   - **Port:** `12201`
   - **Protocol:** `HTTP`
5. Save the destination

**Case B: No GELF destination type — use HTTP POST**

1. Go to **Destinations**
2. Click **New Destination → HTTP**
3. Configure:
   - **Name:** `graylog_gelf`
   - **URL:** `http://vmgray01:12201/gelf`
   - **Method:** `POST`
   - **Headers:** add `Content-Type: application/json`
4. Save the destination

> **Note:** Either way, the goal is: Cribl sends GELF-compatible JSON to Graylog on `vmgray01:12201`.

### 7.2 Create route `all_to_graylog`

1. Go to **Routes** (for the relevant Worker Group / pipeline set)
2. Create a new Route:
   - **Name:** `all_to_graylog`
   - **Condition:** `true`
   - **Pipeline:** `default`
   - **Destination(s):** `graylog_gelf`
3. Place this route in the desired order (usually towards the bottom as a catch-all)
4. Click **Save**

### 7.3 Commit & deploy the configuration

1. Click **Commit** in the Cribl UI
2. Add a message, e.g. `Initial wiring to Graylog GELF HTTP`
3. Click **Commit**
4. Click **Deploy**
5. Select your worker / worker group (e.g. `local`), then **Deploy**

---

## 8 — Optional: TCP Test Source on Port 10090

### 8.1 Create the TCP source in Cribl

In Cribl UI:

1. Go to **Sources → TCP** (or **Sources → Add Source → TCP**)
2. Create a new source:
   - **Name:** `tcp_test_10090`
   - **Port:** `10090`
   - **Listen address:** `0.0.0.0`
   - **Transport:** `TCP`
3. Save and ensure the source is **Enabled**

This source feeds events into the pipeline where `all_to_graylog` will route them to Graylog.

### 8.2 Send a test event with `nc`

```bash
echo '{"short_message":"cribl test event","host":"vmcrib01","level":6}' | nc -q0 vmcrib01 10090
```

> **Note:** `netcat-openbsd` uses `-q0` to close the connection immediately after stdin is sent. The payload is a GELF-style JSON that Graylog understands on the GELF HTTP input.

---

## 9 — Validation

### End-to-end verification in Graylog

On `vmgray01` (Graylog Web UI):

1. Go to **Search**
2. Set the time range to **Last 5 minutes**
3. Search for:

```text
"cribl test event" AND host:vmcrib01
```

4. Confirm you see events with:
   - **Source / host:** `vmcrib01`
   - **Message:** `cribl test event`
   - Fields appropriate for your GELF layout (e.g. `_level`, `_source`)

> **Note:** Once this works, you have confirmed: Cribl source → Cribl route → `graylog_gelf` destination → Graylog GELF HTTP input → Graylog search.

### Service persistence checks

```bash
# Confirm cribl is enabled at boot
sudo systemctl is-enabled cribl || echo "cribl is NOT enabled"

# Check current service status
sudo systemctl status cribl --no-pager

# Verify listening ports
sudo ss -ltnp | grep 9000 || echo "No listener on 9000 (UI)"
sudo ss -ltnp | grep 10090 || echo "No listener on 10090 (test source, optional)"
```

- [ ] Web UI accessible at `http://<vmcrib01-ip>:9000`
- [ ] `cribl.service` enabled and active
- [ ] Test event visible in Graylog search
- [ ] Logs are clean (`journalctl -u cribl --no-pager -n 20`)

---

## 10 — Post-Install

- [ ] Change default Cribl credentials
- [ ] Configure log forwarding to Graylog via the GELF route
- [ ] Add `vmcrib01` to OpenVAS scan targets
- [ ] Add `vmcrib01` to Open-AudIT discovery

---

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| `cribl.service` missing after install | Used `sh.cribl.io` script (known-bad) | Remove leftover files, re-install via tarball (Section 2), then register with systemd (Section 3) |
| No `/opt/cribl` directory | Scripted installer failed silently | `sudo mkdir -p /opt/cribl && sudo chown "$USER":"$USER" /opt/cribl`, then re-run tarball install |
| Service won't start | Missing `/opt/cribl` or broken symlink | Verify tarball extracted correctly: `ls /opt/cribl/bin/cribl` |
| Web UI not loading on :9000 | Service not running or port blocked | `sudo systemctl start cribl` and check `sudo ufw status` |
| No events in Graylog | Route not deployed or Graylog input not running | Verify Cribl route is committed + deployed; check Graylog input is **Running** on port `12201` |
| `nc` test event not arriving | Wrong port or netcat variant | Ensure `netcat-openbsd` is installed; use `-q0` flag |

---

## Quick Reference

```bash
# Start/stop/restart
sudo systemctl restart cribl

# View logs
journalctl -u cribl -f

# Check status
sudo systemctl status cribl --no-pager

# Show Cribl version
/opt/cribl/bin/cribl version

# Check ports
sudo ss -ltnp | grep -E '9000|10090'

# Send test event (from vmcrib01)
echo '{"short_message":"cribl test event","host":"vmcrib01","level":6}' | nc -q0 vmcrib01 10090
```

---

## Quirks & Gotchas

- **Scripted installer is broken on this VM:** `curl https://sh.cribl.io/install.sh | sudo bash` fails silently — no `/opt/cribl`, no `cribl.service`. Always use the manual tarball method.
- **Two destination options for GELF:** Cribl may or may not have a native GELF destination type depending on version. Fall back to generic HTTP POST to `http://vmgray01:12201/gelf` if needed.
- **netcat variant matters:** Use `netcat-openbsd` (provides `-q0` flag). The `netcat-traditional` package behaves differently.
- **Hostname/hosts mismatch:** If hostname was changed post-install, the `127.0.1.1` line in `/etc/hosts` must be updated or DNS resolution issues may occur.

---

*Last updated: 2025-07-14*
