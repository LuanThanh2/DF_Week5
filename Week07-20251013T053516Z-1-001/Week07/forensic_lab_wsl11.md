# Forensic Lab & OWASP Juice Shop on Windows 11 WSL2 — Guide

This guide adapts the Docker-based DFIR and Juice Shop lab to **Windows Subsystem for Linux 2 (WSL2)** on Windows 11.

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Install & Configure WSL2](#2-install--configure-wsl2)
3. [Install Docker in WSL2](#3-install-docker-in-wsl2)
4. [Managing Docker from Windows (Docker Desktop)](#4-managing-docker-from-windows-docker-desktop)
5. [Evidence Handling & DFIR Principles](#5-evidence-handling--dfir-principles)
6. [DFIR Containers in WSL2](#6-dfir-containers-in-wsl2)
7. [OWASP Juice Shop in WSL2](#7-owasp-juice-shop-in-wsl2)
8. [Networking in WSL2](#8-networking-in-wsl2)
9. [Troubleshooting Appendix](#9-troubleshooting-appendix)

---

## 1) Prerequisites

- **Windows 11 Pro/Enterprise/Education** (WSL2 requires Hyper-V features)
- BIOS: virtualization enabled (Intel VT-x or AMD-V)
- Latest Windows Updates

---

## 2) Install & Configure WSL2

```powershell
# Run in PowerShell/cmd (Admin)
wsl --install -d Ubuntu-22.04

# Verify
wsl --status
wsl --list --verbose

# Set Ubuntu as default
wsl --set-default Ubuntu-22.04
```

Confirm kernel version in Ubuntu shell:

```bash
uname -r   # should be a WSL2 kernel (5.x+)
```

---

## 3) Install Docker in WSL2

Inside Ubuntu (WSL2 shell):

```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg lsb-release

# Keyring
sudo install -m 0755 -d /etc/apt/keyrings

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
  
sudo chmod 0644 /etc/apt/keyrings/docker.gpg

# Repo
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
https://download.docker.com/linux/ubuntu $(. /etc/os-release; echo $UBUNTU_CODENAME) stable" \
| sudo tee /etc/apt/sources.list.d/docker.list >/dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io 
sudo apt install -y docker-buildx-plugin docker-compose-plugin
```

Enable user for Docker group:

```bash
sudo usermod -aG docker $USER
newgrp docker
```

Test:

```bash
docker run --rm hello-world
```

---

## 4) Managing Docker from Windows (Docker Desktop)

- Option A: Use **Docker Desktop** for Windows with **WSL integration** (simpler for GUI users).
- Option B: Run Docker **directly inside WSL2** as a service (manual but more isolated).

To check Docker Desktop integration:

```powershell
# PowerShell (Admin)
wsl --list --verbose
# Docker Desktop should list Ubuntu as "integrated"
```

---

## 5) OWASP Juice Shop in WSL2

```bash
docker pull bkimminich/juice-shop:latest

# Run on port 3000 mapped to Windows localhost:8080
docker run -d --name juice-shop -p 127.0.0.1:8080:3000 bkimminich/juice-shop

# Test inside WSL2
curl -I http://127.0.0.1:8080/

# Test from Windows browser
http://localhost:8080/
```

**Note:** WSL2 forwards `127.0.0.1` automatically from VM to Windows host.

---

## 6) Networking in WSL2

- By default, WSL2 apps bound to `127.0.0.1` are reachable on Windows host at `localhost`.
- For LAN access: bind to `0.0.0.0` instead of `127.0.0.1`:

  ```bash
  docker run -d --name juice-shop -p 0.0.0.0:8080:3000 bkimminich/juice-shop
  ```

- Windows Firewall must allow inbound TCP 8080.

Check listeners:

```bash
ss -tlnp | grep 8080
```

---

## 7) Evidence acquisition (WSL2 / Hyper‑V)

> Goal: acquire artifacts maintained by the **Hyper‑V layer** for a WSL2 distro.
> Reality: **WSL2’s utility VM is *not* a normal, user‑managed Hyper‑V VM.** Hyper‑V memory/state cmdlets work for *standard* VMs, but **do not apply** to WSL2’s hidden VM. For WSL2, rely on **virtual disks** and **filesystem exports**; only standard Hyper‑V VMs support full memory/state collection.

---

### 1) Direct dump from hypervisor host (live or dead WSL)

### Standard Hyper‑V VM (supported path)

Create a **Saved State** (writes **.vmrs** memory + **.vmgs** guest state) and export everything, including disks:

```powershell
# Admin PowerShell, Hyper‑V module
Get-VM | Select-Object Name, State
Save-VM -Name "<VMName>"                    # creates .vmrs/.vmgs for that VM
Export-VM -Name "<VMName>" -Path C:\evidence\VM_Export
# Result: C:\evidence\VM_Export\... includes .vmrs, .vmgs, .vhdx, config
```

> Optionally convert the saved state to debugger‑readable dumps with Microsoft SDK tooling (outside scope).

### WSL2 utility VM (what you can/can’t do)

* The WSL2 VM **is NOT** exposed as a normal `Get-VM` object. You **cannot** `Save-VM`, `Checkpoint-VM`, or `Export-VM` it.
* Therefore, **direct hypervisor memory dump of WSL2 is not supported** via public Hyper‑V workflows. Treat WSL2 RAM acquisition as **N/A** unless you have private/unsupported hypervisor instrumentation.
* Proceed to disk‑level acquisition below (what WSL actually persists and what you can reliably collect).

---

### 2) Collect virtual hard disk (persistent Linux filesystem)

WSL2 persists the Linux filesystem in a **VHDX** (typically `ext4.vhdx`) under the user’s package path.

```powershell
# Stop the distro for consistency
wsl --terminate Ubuntu-22.04
wsl --list --running     # should show none

# Locate the backing VHDX (example search)
Get-ChildItem -Path $env:LOCALAPPDATA\Packages -Recurse -Filter ext4.vhdx -ErrorAction SilentlyContinue | Select-Object FullName

# Copy to evidence and hash
$src = "<full path to>\ext4.vhdx"
example:
$src = "C:\Users\NGOCTUPC\AppData\Local\Packages\CanonicalGroupLimited.Ubuntu22.04LTS_79rhkp1fndgsc\LocalState\ext4.vhdx"

$dst = "C:\evidence\Ubuntu-22.04-ext4.vhdx"

Copy-Item -Path $src -Destination $dst -Force
Get-FileHash $dst -Algorithm SHA256 | Out-File C:\evidence\Ubuntu-22.04-ext4.vhdx.sha256.txt
```

**Alternative (clean, supported filesystem export):**

```powershell
wsl --terminate Ubuntu-22.04
wsl --export Ubuntu-22.04 C:\evidence\Ubuntu-22.04-export.tar
Get-FileHash C:\evidence\Ubuntu-22.04-export.tar -Algorithm SHA256 | Out-File C:\evidence\Ubuntu-22.04-export.tar.sha256.txt
```

> Use **VHDX** when you need low‑level filesystem artifacts (slack space, deleted inodes, journal). Use **`wsl --export` tar** for a clean logical filesystem copy.

---

### 3) Collect differencing files

There are **two meanings** of “differences” — collect both when relevant:

### A) Hyper‑V differencing disks (`.avhdx`) — *standard Hyper‑V VMs with checkpoints*

If the target is a normal VM with checkpoints, capture the full chain: all **.avhdx** plus the **base .vhdx** (do **not** merge on the source).

```powershell
# Example (standard VM, not WSL2):
Get-VHD -Path "C:\Hyper-V\VMs\MyVM\Snapshots\*.avhdx" | Format-List Path,ParentPath
# Copy every avhdx + base vhdx to evidence; hash each file.
```

> **WSL2:** the user distro typically persists as a single dynamic **`ext4.vhdx`**, not a chain of `.avhdx`. If you do find a chain (rare/advanced), preserve the full chain exactly.

### B) Filesystem diffs across two exports (what changed over time)

Create **time‑stamped exports** and compute diffs on your analysis workstation:

```bash
# On analysis host (Linux), comparing two tar exports:
mkdir -p /mnt/A /mnt/B
sudo tar -xpf Ubuntu-22.04-export-T0.tar -C /mnt/A
sudo tar -xpf Ubuntu-22.04-export-T1.tar -C /mnt/B
sudo diff -urN /mnt/A /mnt/B > /analysis/wsl_fs_delta.diff
```

For VHDX, mount two raw images (read‑only) and diff directory trees (`diff -urN`, `rsync --dry-run`), or build timelines (Plaso) to highlight change windows.

---

### 4)Collect state files

### Standard Hyper‑V VM (supported path)

State is represented by **`.vmrs`** (runtime memory/state) and **`.vmgs`** (guest metadata/state). Produce them by **saving** the VM and then **exporting**:

```powershell
Save-VM -Name "<VMName>"                   # generates .vmrs/.vmgs
Export-VM -Name "<VMName>" -Path C:\evidence\VM_Export_WithState
# Hash all exported artifacts
Get-ChildItem C:\evidence\VM_Export_WithState -Recurse |
  Get-FileHash -Algorithm SHA256 |
  Tee-Object -FilePath C:\evidence\VM_Export_WithState.hashes.txt
```

### WSL2 utility VM

* WSL2’s VM **does not expose** `.vmrs/.vmgs` through Hyper‑V cmdlets; you cannot create/collect them via supported interfaces.
* Treat **state collection as N/A** for WSL2. Instead, capture **logical state** from within WSL *prior to termination* and persist to Windows storage:

```bash
# Inside WSL (logical/volatile artifacts)
ps aux                  > /mnt/c/evidence/wsl_pslist.txt
ss -tulpen              > /mnt/c/evidence/wsl_sockets.txt
journalctl -xe          > /mnt/c/evidence/wsl_journal.txt  2>/dev/null || true
docker ps -a            > /mnt/c/evidence/wsl_docker_ps.txt
docker logs juice-shop  > /mnt/c/evidence/juice-logs.txt
```

---

### Mounting & analysis (forensic workstation)

**VHDX path: convert & mount read‑only**

```bash
sudo apt-get install -y qemu-utils
qemu-img convert -O raw /analysis/Ubuntu-22.04-ext4.vhdx /analysis/Ubuntu-22.04-ext4.raw
sudo mkdir -p /mnt/ubuntu_vhd
sudo mount -o ro,loop /analysis/Ubuntu-22.04-ext4.raw /mnt/ubuntu_vhd
```

**TAR export: extract and analyze**

```bash
mkdir -p /mnt/ubuntu_export
sudo tar -xpf /analysis/Ubuntu-22.04-export.tar -C /mnt/ubuntu_export
```

Run DFIR tooling (TSK, Plaso, YARA, Zeek/Suricata on pcaps) against these **read‑only** mounts. Always verify **pre-/post‑transfer SHA256** hashes.

---

### Notes & guardrails

* **WSL2 ≠ standard Hyper‑V VM**: memory/state capture that works for a named Hyper‑V VM **does not** apply to WSL2’s hidden VM.
* Prefer **`wsl --export`** (logical FS) or **copying `ext4.vhdx`** (physical FS artifacts).
* If you encounter a non‑default WSL deployment that *is* registered as a Hyper‑V VM, switch to the **standard VM workflow** (save state, export `.vmrs/.vmgs/.vhdx/.avhdx`).
* Work from **copies only**; never modify originals; preserve and verify hashes.

## 8) Docker Desktop for Windows & DFIR Tooling (Hyper‑V backend, pull tools first)

Goal: analyze evidence on **Windows** without touching the WSL distro under investigation. We run Docker Desktop with the **Hyper‑V backend** (WSL Integration OFF), then **pull DFIR tool images first**, and only then run analyses with evidence mounted **read‑only**.

---

### A — Install & configure Docker Desktop (Hyper‑V backend)

1. Download & install: [https://www.docker.com/products/docker-desktop/](https://www.docker.com/products/docker-desktop/)

2. Enable Windows features (Admin PowerShell) and reboot:

```powershell
dism /online /enable-feature /featurename:Microsoft-Hyper-V /all /norestart
dism /online /enable-feature /featurename:Containers       /all /norestart
Restart-Computer
```

3. Configure backend isolation:

* Docker Desktop → **Settings → General**: **uncheck** “Use the WSL 2 based engine”.
* Docker Desktop → **Settings → Resources → WSL Integration**: **OFF** for all distros (đặc biệt distro WSL nghi phạm).
* **Apply & Restart** Docker Desktop; wait until the whale icon says **Running**.

4. Verify daemon is up (named pipe exists):

```powershell
# Wait for the engine to start
docker version    # must show both Client and Server
```

If you see `open //./pipe/docker_engine: The system cannot find the file specified`, open Docker Desktop UI and let it finish starting.

---

### B — Prepare host folders (Windows paths)

* `C:\evidence` → acquired images / pcaps / logs (mounted **read‑only**)
* `C:\analysis` → DFIR outputs (writable)

```powershell
New-Item -ItemType Directory -Path C:\evidence -Force | Out-Null
New-Item -ItemType Directory -Path C:\analysis -Force | Out-Null
```

---

### C — **Pull DFIR tool images (first)**

Run these pulls once to cache tools locally (can work offline later):

```powershell
# Network forensics
docker pull zeek/zeek:latest
docker pull jasonish/suricata:latest

# Filesystem / disk
docker pull aswinvk28/sleuthkit:latest
# (optional) log2timeline / Plaso via dockerhub mirror
docker pull log2timeline/plaso:latest

# Memory
docker pull cincan/volatility3:latest

# Scanning / utilities
docker pull blacktop/yara:latest
# Handy web tool
docker pull ghcr.io/gchq/cyberchef:latest
```

> Tip: record image digests for reproducibility: `docker images --digests`.

---

### D — Run examples (evidence read‑only)

**Zeek to analize PCAP files for protocol log:**
Output: Rich logs (conn.log, http.log, dns.log, ssl.log, etc.).

```powershell
docker run --rm `
  -v C:\evidence:/evidence:ro `
  -v C:\analysis:/out `
  zeek/zeek:latest `
  bash -lc "zeek -r /evidence/capture.pcap && mv *.log /out/"
# or in one line
docker run --rm -v C:\evidence:/evidence:ro -v C:\analysis:/out zeek/zeek:latest bash -lc "zeek -r /evidence/capture.pcap && mv *.log /out/"
```

**Suricata for signature-based IDS:**
Output: Alerts (eve.json, fast.log) + some protocol records.

```powershell
docker run --rm `
  -v C:\evidence:/evidence:ro `
  -v C:\analysis:/out `
  jasonish/suricata:latest `
  -r /evidence/capture.pcap -l /out/
#or one line

```

**Volatility 3 (if you have a memory image):**

```powershell
docker run --rm -v C:\evidence:/evidence:ro cincan/volatility3:latest --help

docker run --rm -v "C:\evidence:/evidence:ro" cincan/volatility3:latest `
  -f /evidence/memdump.raw windows.pslist 2>&1 | Out-File -Encoding utf8 "C:\analysis\pslist.txt"

```

**Sleuth Kit (file system listing from disk image):**

```powershell
docker run --rm -v "C:\evidence:/evidence:ro" -v "C:\analysis:/out" ubuntu:22.04 `
  bash -lc "apt-get update && apt-get install -y sleuthkit && fls -r /evidence/disk.dd > /out/file_listing.txt"

**Plaso / log2timeline (timeline):**

```powershell
docker run --rm -v C:\evidence:/evidence:ro -v C:\analysis:/out ` 
  log2timeline/plaso:latest log2timeline /out/case.plaso /evidence/target_dir_or_image

docker run --rm -v C:\analysis:/out log2timeline/plaso:latest ` 
  psort -o l2tcsv -w /out/case_timeline.csv /out/case.plaso
```

**YARA (recursive scan):**

```powershell
docker run --rm -v C:\evidence:/evidence:ro -v C:\analysis:/out ` 
  blacktop/yara:latest -r /rules /evidence/target_dir > C:\analysis\yara_hits.txt
```

**CyberChef (web UI on localhost:8080):**

```powershell
docker run -d --rm -p 8080:80 --name cyberchef ghcr.io/gchq/cyberchef:latest
# Open http://localhost:8080
```

---

### E — Evidence‑handling rules (don’t contaminate)

* Mount evidence **read‑only** (`:ro`); write outputs to `C:\analysis`.
* Hash artifacts **before/after** processing; expect identical.
* Run Docker Desktop on **Hyper‑V backend**; keep **WSL Integration OFF** to avoid touching the suspect WSL.
* Prefer a separate forensic workstation if possible.

---

### F — Troubleshooting (Windows)

* **`open //./pipe/docker_engine`** → Docker daemon not running. Open Docker Desktop UI; wait until **Running**.
* **Backend check:**

  ```powershell
  docker info | findstr /i "Operating System"
  ```

  Should indicate Hyper‑V backend when WSL engine is disabled.
* **Hyper‑V services:**

  ```powershell
  Get-Service vmms, HvHost | ft Name, Status, StartType
  Start-Service vmms; Start-Service HvHost
  ```
* **Reset Docker Desktop:** Settings → Troubleshoot → **Restart** (hoặc **Factory reset** cuối cùng).

---

### G — Isolation checklist

* [ ] WSL suspect distro **terminated**: `wsl --terminate <Distro>`
* [ ] Docker Desktop **WSL Integration OFF** for all distros
* [ ] **Hyper‑V backend** active (WSL2 engine unchecked)
* [ ] Evidence mounted `C:\evidence:/evidence:ro`
* [ ] Outputs in `C:\analysis` only
* [ ] Image digests recorded (`docker images --digests`)


## 9) Troubleshooting Appendix

- **Docker fails to start**: Ensure `wsl --status` shows WSL2 backend. Restart `wsl --shutdown` then relaunch.
- **Cannot access from Windows browser**: ensure container bound to `127.0.0.1` or `0.0.0.0`, not only a Linux IP.
- **Port conflicts**: stop/remove old containers (`docker rm -f juice-shop`).
- **Evidence mount fails**: use `/mnt/c/...` paths to reference Windows drives.
- **Performance tips**: use WSL2 `.wslconfig` to adjust memory/CPU if DFIR tools are heavy.

---

## Checklist

- [ ] Docker running in WSL2 or Docker Desktop integrated.
- [ ] Evidence mounted read-only from `/mnt/c/evidence`.
- [ ] Outputs saved to `/mnt/c/work`.
- [ ] Juice Shop reachable at `http://localhost:8080` on Windows.
- [ ] Hashes pre/post match.
