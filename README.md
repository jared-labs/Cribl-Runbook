# Deploy VMCRIB01 — Cribl Stream → Graylog (GELF HTTP)

Single-VM deployment of Cribl Stream on Ubuntu Server 24.04, running as `vmcrib01` on Proxmox, forwarding logs to Graylog (`vmgray01`) via GELF HTTP on port `12201`. Copy/paste friendly.

> Guardrail: `vmcrib01` runs **Cribl Stream only**. Graylog, OpenSearch, and MongoDB stay on `vmgray01`.
> Guardrail: **Do not** use the Cribl `https://sh.cribl.io/install.sh` script on this box — it failed to create `/opt/cribl` and `cribl.service` here. Use the manual tarball method in this runbook.

* * *

## 0) VM definition in Proxmox (before OS install)

Create a new VM on your Proxmox cluster:

- Name: `VMCRIB01`
- Guest OS: Linux → Ubuntu → 24.04
- CPU: 2 vCPUs (4+ recommended if you expect higher volume)
- RAM: 4 GB (2 GB minimum, 4–8 GB recommended)
- Disk: 20–40 GB on `local-lvm` (or your thin pool)
- BIOS / Machine: cluster standard (e.g., `OVMF (UEFI)` or `SeaBIOS`, `q35` if preferred)
- Network: VirtIO on `vmbr0` (flat LAN)
- QEMU Guest Agent: Enabled (tick “Qemu Agent” checkbox)

Install **Ubuntu Server 24.04 (minimal)** inside this VM.

* * *

## 1) Base OS prep & hostname (Ubuntu 24.04)

Log in as your normal sudo user on `vmcrib01`.

### 1.1 Update packages & basic tools

    sudo apt update
    sudo apt -y full-upgrade

    # Helpful base packages
    sudo apt install -y qemu-guest-agent vim curl tar ca-certificates netcat-openbsd

    # Enable guest agent for Proxmox
    sudo systemctl enable --now qemu-guest-agent

Reboot if the kernel was updated:

    sudo reboot

### 1.2 Ensure hostname and /etc/hosts are correct

After reboot:

    hostnamectl set-hostname vmcrib01

Fix the `127.0.1.1` line in `/etc/hosts` so it matches the hostname:

    sudo sed -i 's/^127\.0\.1\.1.*/127.0.1.1\tvmcrib01/' /etc/hosts
    grep '^127\.0\.1\.1' /etc/hosts
    hostnamectl

> If you changed the hostname after install, log out and back in so your shell prompt matches.

* * *

## 2) Install Cribl Stream under /opt/cribl (manual tarball only)

> Important: The official script  
> 
> `curl https://sh.cribl.io/install.sh | sudo bash`  
> 
> previously **failed** on this environment (no `/opt/cribl`, no `cribl.service`).  
> 
> **Always** use the tarball method below on `vmcrib01`.

### 2.1 Create `/opt/cribl` and give ownership to your user

    sudo mkdir -p /opt/cribl
    sudo chown "$USER":"$USER" /opt/cribl
    cd /opt/cribl

### 2.2 Download latest Cribl Stream tarball

    curl -L "$(curl -s https://cdn.cribl.io/dl/latest)" -o cribl.tgz

### 2.3 Extract into `/opt/cribl` and clean up

    tar xzf cribl.tgz --strip-components=1
    rm cribl.tgz

You should now have:

- Binaries in `/opt/cribl/bin`
- Config in `/opt/cribl/local`

Quick sanity check:

    ls -l /opt/cribl/bin
    /opt/cribl/bin/cribl version

* * *

## 3) Register Cribl with systemd (boot-start)

### 3.1 Enable boot-start with systemd

    cd /opt/cribl
    sudo /opt/cribl/bin/cribl boot-start enable -m systemd

This should create `/etc/systemd/system/cribl.service`.

Verify:

    grep -n 'cribl' /etc/systemd/system/cribl.service
    sudo systemctl daemon-reload

### 3.2 Enable and start the service (ensure persistence)

    sudo systemctl enable cribl
    sudo systemctl start cribl
    sudo systemctl status cribl --no-pager

Expected:

- `Active: active (running)`
- No errors about missing `/opt/cribl`.

> If `cribl.service` is missing or `/opt/cribl` is not found, re-run Section 2 and 3.1.  
> Do **not** fall back to the `sh.cribl.io` script — that’s the known-bad path on this VM.

* * *

## 4) Networking & firewall on VMCRIB01

Cribl will use:

- Cribl UI: TCP **9000**
- Optional TCP test source: TCP **10090** (if you enable it in Cribl)
- Outbound to Graylog GELF HTTP: TCP **12201** to `vmgray01`

If `ufw` is enabled, open ports:

    sudo ufw allow 9000/tcp comment 'Cribl UI'
    sudo ufw allow 10090/tcp comment 'Cribl TCP test input (optional)'
    sudo ufw reload

You can verify listeners later with:

    sudo ss -ltnp | grep -E '9000|10090' || echo "No Cribl listeners yet"

* * *

## 5) Cribl Web UI – first login

Once the `cribl` service is running:

From your browser (on your LAN):

- `http://<vmcrib01-ip>:9000/`  
  or, if you have DNS:  
- `http://vmcrib01.lan:9000/`

Log in with the credentials you set during initial Cribl setup (or the defaults from Cribl docs if fresh).

Confirm **Status → Workers** shows your local worker running.

* * *

## 6) Graylog side — GELF HTTP input on VMGRAY01

On `vmgray01` (Graylog Web UI):

1. Go to **System → Inputs**.
2. Select **GELF HTTP** from the dropdown.
3. Click **Launch new input**.
4. Configure:
   - **Title:** `gelf_http_from_cribl`
   - **Node / Global:** `Global` (or bound to the Graylog node you prefer)
   - **Bind address:** `0.0.0.0`
   - **Port:** `12201`
5. Click **Save**.
6. Confirm the new GELF HTTP input shows as **Running**.

> In this homelab, Cribl treats Graylog as a GELF HTTP endpoint on `vmgray01:12201`.

* * *

## 7) Cribl → Graylog wiring (destination + route)

### 7.1 Create a destination called `graylog_gelf`

In Cribl UI on `vmcrib01`:

#### Case A: GELF destination type exists

1. Go to **Destinations**.
2. Click **New Destination**.
3. Select **GELF HTTP** (or Cribl’s GELF-type destination).
4. Configure:
   - **Name:** `graylog_gelf`
   - **Host:** `vmgray01`
   - **Port:** `12201`
   - **Protocol:** `HTTP`
5. Save the destination.

#### Case B: No GELF destination type — use HTTP POST

1. Go to **Destinations**.
2. Click **New Destination → HTTP**.
3. Configure:
   - **Name:** `graylog_gelf`
   - **URL:** `http://vmgray01:12201/gelf`
   - **Method:** `POST`
   - **Headers:** add `Content-Type: application/json`
4. Save the destination.

> Either way, the goal is: Cribl sends GELF-compatible JSON to Graylog on `vmgray01:12201`.

### 7.2 Create a route `all_to_graylog`

1. Go to **Routes** (for the relevant Worker Group / pipeline set).
2. Create a new Route:
   - **Name:** `all_to_graylog`
   - **Condition:** `true`
   - **Pipeline:** `default`
   - **Destination(s):** `graylog_gelf`
3. Place this route in the order you want (usually towards the bottom as a catch-all).
4. Click **Save**.

### 7.3 Commit & deploy the configuration

1. Click **Commit** in the Cribl UI.
2. Add a message, e.g. `Initial wiring to Graylog GELF HTTP`.
3. Click **Commit**.
4. Click **Deploy**.
5. Select your worker / worker group (e.g. `local`), then **Deploy**.

* * *

## 8) Optional: TCP test source on port 10090

### 8.1 Create the TCP source in Cribl

In Cribl UI:

1. Go to **Sources → TCP** (or **Sources → Add Source → TCP**).
2. Create a new source:
   - **Name:** `tcp_test_10090`
   - **Port:** `10090`
   - **Listen address:** `0.0.0.0`
   - **Transport:** `TCP`
3. Save and ensure the source is **Enabled**.

This source feeds events into the pipeline where `all_to_graylog` will route them to Graylog.

### 8.2 Send a test event with `nc`

On `vmcrib01`:

    echo '{"short_message":"cribl test event","host":"vmcrib01","level":6}' | nc -q0 vmcrib01 10090

Notes:

- `netcat-openbsd` uses `-q0` to close the connection immediately after stdin is sent.
- The payload is a GELF-style JSON that Graylog understands on the GELF HTTP input.

* * *

## 9) Validation in Graylog (end-to-end)

On `vmgray01` (Graylog Web UI):

1. Go to **Search**.
2. Set the time range to **Last 5 minutes** (or similar).
3. Search for:

       "cribl test event" AND host:vmcrib01

4. Confirm you see events from:

   - **Source / host:** `vmcrib01`
   - **Message:** `cribl test event`
   - Fields appropriate for your GELF layout (e.g. `_level`, `_source`).

> Once this works, you have confirmed:  
> Cribl source → Cribl route → `graylog_gelf` destination → Graylog GELF HTTP input → Graylog search.

* * *

## 10) Service persistence & health checks on VMCRIB01

### 10.1 Confirm `cribl` is enabled at boot

    sudo systemctl is-enabled cribl || echo "cribl is NOT enabled"

Expected:

    enabled

If not:

    sudo systemctl enable cribl

### 10.2 Check current service status

    sudo systemctl status cribl --no-pager

If it’s not running:

    sudo systemctl start cribl

### 10.3 Verify listening ports

    sudo ss -ltnp | grep 9000 || echo "No listener on 9000 (UI)"
    sudo ss -ltnp | grep 10090 || echo "No listener on 10090 (test source, optional)"

Expected:

- `0.0.0.0:9000` → Cribl UI
- `0.0.0.0:10090` → TCP test input (if configured)

* * *

## 11) Known installation quirk (documented behavior)

This section records the real-world issue that prompted the manual install method.

### 11.1 Failed Cribl scripted installer

Running:

    curl https://sh.cribl.io/install.sh | sudo bash

On `vmcrib01` resulted in:

- No `/opt/cribl` directory created.
- No `cribl.service` systemd unit.
- `systemctl` could not manage Cribl, and the service was not persistent across reboots.

Because of this, the supported path for this homelab node is:

1. **Always** install Cribl via tarball into `/opt/cribl` (Section 2).
2. Register Cribl with systemd using:

       sudo /opt/cribl/bin/cribl boot-start enable -m systemd

3. Enable + start via systemd:

       sudo systemctl enable cribl
       sudo systemctl start cribl

Once this is in place, Cribl runs as a normal systemd service and persists across reboots.

* * *

## 12) Quick reference (commands)

- Check service:

      sudo systemctl status cribl --no-pager

- Restart service:

      sudo systemctl restart cribl

- Show Cribl version:

      /opt/cribl/bin/cribl version

- Check ports:

      sudo ss -ltnp | grep -E '9000|10090'

- Send test event (from `vmcrib01`):

      echo '{"short_message":"cribl test event","host":"vmcrib01","level":6}' | nc -q0 vmcrib01 10090

This completes the `VMCRIB01` Cribl Stream deployment runbook in the same style as the Proxmox and VMVULN01 runbooks.
