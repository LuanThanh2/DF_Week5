# Docker-Based Forensic Lab & OWASP Juice Shop — Revised Guide

> A clean, copy–paste guide for setting up a DFIR-friendly Docker environment on Ubuntu and deploying OWASP Juice Shop behind an OpenStack Floating IP. This revision fixes typos, clarifies the logic/sequence, adds route/rp_filter checks for multi-NIC VMs, and tightens security notes.

---

## Table of Contents

1. [SSH Access (Server-wide vs Client-side, Key/Password)](#1-ssh-access)
2. [Install Docker on Ubuntu (Official Repo)](#2-install-docker-on-ubuntu)
3. [Group Membership & Basic Sanity Tests](#3-group-membership--basic-sanity-tests)
4. [DFIR Principles & Evidence Handling](#4-dfir-principles--evidence-handling)
5. [Essential DFIR Containers (Recipes)](#5-essential-dfir-containers-recipes)
6. [OWASP Juice Shop on an OpenStack VM](#6-owasp-juice-shop-on-an-openstack-vm)
   - [Identify Fixed IP & Variables](#identify-fixed-ip--variables)
   - [Create/Verify Container](#createverify-container)
   - [Routing Check (Floating IP path)](#routing-check-floating-ip-path)
   - [Reverse-Path Filter (rp_filter)](#reverse-path-filter-rp_filter)
   - [Host Firewall Quick Checks](#host-firewall-quick-checks)
   - [External Test via Floating IP](#external-test-via-floating-ip)
   - [If It Times Out — Minimal Packet Trace](#if-it-times-out--minimal-packet-trace)
   - [Alternative: Host Networking](#alternative-host-networking)
7. [Troubleshooting Appendix](#7-troubleshooting-appendix)

---

## 1) SSH Access
### Server-wide vs Client-side
- **Server config (affects all users):** `/etc/ssh/sshd_config` and drop-ins `/etc/ssh/sshd_config.d/*.conf`.
- **Client config (per user):** `~/.ssh/config` (does **not** change server policy).

Check effective server policy:
```bash
sudo sshd -T | egrep -i 'passwordauthentication|usepam|authenticationmethods'
```

### Key-based login (recommended)
Find keys and authorized keys:
```bash
ls -la ~/.ssh
cat ~/.ssh/authorized_keys
```

### Password login (if needed; safest via Match)
Create a drop-in that loads after cloud-init:
```bash
sudo tee /etc/ssh/sshd_config.d/99-custom.conf >/dev/null <<'EOF'
# Keep global policy secure
PasswordAuthentication no
UsePAM yes

# Allow password ONLY for 'student' (example)
Match User student
    PasswordAuthentication yes
EOF

sudo systemctl restart ssh
```

Set a password for the account you permit:
```bash
sudo passwd student
```

**Client-side force password test** (bypasses remembered keys):
```bash
ssh -o PreferredAuthentications=password -o PubkeyAuthentication=no -o IdentitiesOnly=yes student@<VM-IP>
```

---

## 2) Install Docker on Ubuntu
```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg lsb-release

# Keyring dir
sudo install -m 0755 -d /etc/apt/keyrings

# Docker’s repo key (binary keyring)
curl -fsSL https://download.docker.com/linux/ubuntu/gpg |   sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod 0644 /etc/apt/keyrings/docker.gpg

# Official repo (single line!)
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release; echo $UBUNTU_CODENAME) stable" | sudo tee /etc/apt/sources.list.d/docker.list >/dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

---

## 3) Group Membership & Basic Sanity Tests
```bash
# Put your user in the docker group (equivalent to root privileges)
sudo usermod -aG docker "$USER"
newgrp docker   # or log out/in

# Confirm group + who’s in it
id -nG "$USER"
getent group docker

# Add/remove specific users
sudo gpasswd -a <username> docker
sudo gpasswd -d <username> docker

# Enable & test
sudo systemctl enable --now docker
docker run --rm hello-world
docker --help
```

**Security note:** membership in `docker` ≈ root-equivalent on that host. Only add trusted users.

---

## 4) DFIR Principles & Evidence Handling

**Core principles**
- **Reproducible:** pin image digests/tags; record `docker images --digests`.
- **Non‑destructive:** mount evidence **read‑only**.
- **Attribution:** hash **before/after**, expect identical; keep run logs.
- **Least privilege:** avoid `--privileged`; drop unnecessary capabilities.
- **Time & locale:** record `date -u`, set `TZ=UTC`.
- **Isolation:** one tool per container; named volumes for output.

**Folder layout**
```
/evidence/      # READ-ONLY inputs (images, pcaps, logs)
/work/          # writable outputs (reports, timelines, hashes)
```

**Hashing**
```bash
mkdir -p /work
sha256sum /evidence/images/disk.dd         | tee -a /work/hashes.pre.txt
sha256sum /evidence/pcaps/case1.pcapng     | tee -a /work/hashes.pre.txt
# ...work...
sha256sum /evidence/images/disk.dd         | tee -a /work/hashes.post.txt
sha256sum /evidence/pcaps/case1.pcapng     | tee -a /work/hashes.post.txt
```

**Generic run template**
```bash
docker run --rm   --name dfir-tool-1   --user $(id -u):$(id -g)   --read-only   --mount type=bind,src=/evidence,dst=/evidence,ro   --mount type=bind,src=/work,dst=/out   --cpus="4" --memory="8g"   --env TZ=UTC   <image>:<tag-or-digest> <tool-cmd> ...
```

---

## 5) Essential DFIR Containers (Recipes)

### Volatility 3 (Memory Forensics)
```dockerfile
# Dockerfile.vol3 (example build for repeatability)
FROM python:3.11-slim
RUN apt-get update && apt-get install -y --no-install-recommends     git build-essential libcapstone-dev libyara-dev yara &&     rm -rf /var/lib/apt/lists/*
RUN pip install --no-cache-dir capstone pefile pycryptodome yara-python openpyxl
RUN git clone --depth=1 https://github.com/volatilityfoundation/volatility3 /opt/vol3
ENV PATH="/opt/vol3:${PATH}"
WORKDIR /data
ENTRYPOINT ["python","-m","volatility3"]
```
```bash
docker build -f Dockerfile.vol3 -t vol3:class .
docker run --rm -v /evidence:/evidence:ro -v /work:/out vol3:class   -f /evidence/images/host1.mem windows.pslist > /out/host1_pslist.txt
```

### Sleuth Kit / TSK
```dockerfile
# Dockerfile.tsk
FROM debian:stable-slim
RUN apt-get update && apt-get install -y --no-install-recommends     sleuthkit afflib-tools ewf-tools bulk-extractor exiftool python3-minimal ca-certificates &&     rm -rf /var/lib/apt/lists/*
WORKDIR /data
ENTRYPOINT ["/bin/bash"]
```
```bash
docker run --rm -v /evidence:/evidence:ro -v /work:/out tsk:class   fls -o 2048 -r /evidence/images/disk.dd > /out/disk_fls.txt
```

### Plaso / log2timeline
```bash
docker run --rm -v /evidence:/evidence:ro -v /work:/out plaso:latest   log2timeline.py /out/case1.plaso /evidence/images/disk_mountpoint_or_export
docker run --rm -v /work:/work plaso:latest   psort.py -o l2tcsv -w /work/case1_timeline.csv /work/case1.plaso
```

### Zeek (PCAP)
```bash
docker run --rm -v /evidence:/evidence:ro -v /work:/out zeek:latest   zeek -r /evidence/pcaps/case1.pcapng && mv *.log /out/zeek_logs_case1/
```

### Suricata (PCAP IDS)
```bash
docker run --rm -v /evidence:/evidence:ro -v /work:/out suricata:latest   suricata -r /evidence/pcaps/case1.pcapng -l /out/suricata_case1/
```

### YARA (Rule Scanning)
```bash
docker run --rm -v /evidence:/evidence:ro -v /rules:/rules:ro -v /work:/out yara:latest   yara -r /rules/malware_rules.yar /evidence/images/exported_files | tee /out/yara_hits.txt
```

### CyberChef (Web UI)
```bash
docker run -d --rm -p 8080:80 --name cyberchef ghcr.io/gchq/cyberchef:latest
# Open http://localhost:8080
```

**Compose mini-stack (Zeek + Suricata + CyberChef)** — pin digests for reproducibility.

---

## 6) OWASP Juice Shop on an OpenStack VM

### Identify Fixed IP & Variables
```bash
ip a
ip route
# Example (replace with yours)
FIXED_IP=10.0.0.155       # ens3 (DNAT target)
FIP=172.24.4.155          # Floating IP
HOST_PORT=8080
CONTAINER_PORT=3000
NAME=juice-shop
IMAGE=bkimminich/juice-shop:latest
```

### Create/Verify Container
```bash
docker pull "$IMAGE"

docker rm -f "$NAME" 2>/dev/null || true
docker run -d --name "$NAME"   -p ${FIXED_IP}:${HOST_PORT}:${CONTAINER_PORT}   --restart unless-stopped   "$IMAGE"

# Verify locally (NOT localhost)
ss -tlnp | grep :${HOST_PORT} || echo "port ${HOST_PORT} not listening"
curl -I --max-time 5 http://${FIXED_IP}:${HOST_PORT}/
docker logs -n 50 "$NAME"
```

### Routing Check (Floating IP path)
Find the FIP router (often visible in tcpdump as the source, e.g., `172.24.4.1`), then:
```bash
FIP_ROUTER=172.24.4.1
ip route get ${FIP_ROUTER}
# Expect: via 10.0.0.1 dev ens3 src ${FIXED_IP}
```
If you have two defaults with equal metrics, bump the non-DMZ one:
```bash
sudo ip route change default via 10.0.0.1 dev ens3 metric 100
sudo ip route change default via 10.0.1.1 dev ens4 metric 200
```

### Reverse-Path Filter (rp_filter)
Strict mode (1) drops DNAT flows on multi-NIC hosts. Use loose mode (2):
```bash
sudo sysctl -w net.ipv4.conf.all.rp_filter=2
sudo sysctl -w net.ipv4.conf.default.rp_filter=2
sudo sysctl -w net.ipv4.conf.ens3.rp_filter=2

cat <<EOF | sudo tee /etc/sysctl.d/60-rpfilter.conf >/dev/null
net.ipv4.conf.all.rp_filter=2
net.ipv4.conf.default.rp_filter=2
net.ipv4.conf.ens3.rp_filter=2
EOF
sudo sysctl --system
```

### Host Firewall Quick Checks
```bash
sudo ufw status verbose
sudo iptables -L DOCKER-USER -n -v
```

### External Test via Floating IP
```bash
curl -I --max-time 5 http://${FIP}:${HOST_PORT}/
```

### If It Times Out — Minimal Packet Trace
```bash
sudo tcpdump -ni ens3 tcp port ${HOST_PORT}
# In parallel from outside:
# curl -Iv --max-time 5 http://${FIP}:${HOST_PORT}/
# If SYNs arrive but no SYN-ACK, fix rp_filter/route as above.
```

### Alternative: Host Networking
```bash
docker rm -f "$NAME"
docker run -d --name "$NAME" --network host -e PORT=${HOST_PORT} --restart unless-stopped "$IMAGE"
ss -tlnp | grep :${HOST_PORT}
curl -I http://${FIXED_IP}:${HOST_PORT}/
curl -I http://${FIP}:${HOST_PORT}/
```

---

## 7) Troubleshooting Appendix

- **“Permission denied” on output** → ensure `--user $(id -u):$(id -g)` and `/work` ownership.
- **Container can’t see evidence** → fix mount paths (`-v /evidence:/evidence:ro`).
- **Timestamps wrong** → set `TZ=UTC`, record `date -u`.
- **HTTP 8080 unreachable over FIP** → bind to fixed IP, verify route, set `rp_filter=2`, try port 80 if upstream limits ports, or use `--network host`.
- **Security groups** → ensure your OpenStack SG allows required TCP ports (22/80/443/8080 etc.).
- **Docker name/port conflicts** → `docker rm -f <name>` then recreate; check `ss -tlnp`.

---

### Lab Checklist (printable)
- [ ] Evidence mounted **read-only**.
- [ ] Pre-/post-hashes recorded; match verified.
- [ ] Container image + digest recorded.
- [ ] Commands & parameters logged.
- [ ] Outputs saved to `/work`.
- [ ] Time recorded in **UTC**.
- [ ] Route checked (`ip route get <FIP router>`).
- [ ] `rp_filter=2` persisted.
- [ ] Service reachable via FIP.
