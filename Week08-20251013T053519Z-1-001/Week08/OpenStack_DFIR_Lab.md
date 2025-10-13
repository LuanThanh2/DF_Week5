# OpenStack Lab — Cloud‑Native DFIR Artifact Collection
Student lab sheet (print & follow-along). Mirrors Week 8 topics. Copy/paste friendly.

---

## Task 0 — Environment Setup (Tenant/Project)

**Goal:** create an isolated project, network, and two VMs:
- `vm-app` (runs Docker app to investigate)
- `vm-forensics` (jump host; collects artifacts)

**Prereqs**
- `openstack` CLI configured (`source <your>_openrc.sh`)
- Keypair uploaded; security group egress allowed; ingress SSH from your IP

**Create project-scoped resources**
```bash
# Vars (adjust)
PRJ=wk8-dfir
NET=${PRJ}-net
SUB=${PRJ}-sub
ROUT=${PRJ}-router
SG=${PRJ}-sg
IMG="Ubuntu 22.04"          # must exist in Glance
FLAVOR="m1.small"           # adjust
KEY="mykey"                 # must exist

# Network & router
openstack network create $NET
openstack subnet create $SUB --network $NET --subnet-range 10.88.0.0/24
openstack router create $ROUT
openstack router set --external-gateway public $ROUT
openstack router add subnet $ROUT $SUB

# Security group (SSH + HTTP)
openstack security group create $SG
openstack security group rule create --ingress --protocol tcp --dst-port 22 $SG
openstack security group rule create --ingress --protocol tcp --dst-port 80 $SG
openstack security group rule create --egress $SG
```

**Boot VMs**
```bash
# Forensics VM with floating IP
openstack server create --image "$IMG" --flavor "$FLAVOR" \
  --key-name $KEY --security-group $SG --network $NET vm-forensics
openstack floating ip create public
FIP_F=$(openstack floating ip list -f value -c "Floating IP Address" | head -n1)
openstack server add floating ip vm-forensics $FIP_F

# App VM with floating IP
openstack server create --image "$IMG" --flavor "$FLAVOR" \
  --key-name $KEY --security-group $SG --network $NET vm-app
openstack floating ip create public
FIP_A=$(openstack floating ip list -f value -c "Floating IP Address" | sed -n '2p')
openstack server add floating ip vm-app $FIP_A

echo "Forensics: $FIP_F   App: $FIP_A"
```

**Prepare hosts**
```bash
# SSH to vm-forensics, then from there to vm-app via $FIP_A if needed.
ssh ubuntu@$FIP_F

# On vm-forensics (install tooling)
sudo apt-get update && sudo apt-get install -y jq tcpdump conntrack ethtool \
  docker.io cosign trivy curl git coreutils
sudo usermod -aG docker $USER

# On vm-app (via SSH from forensics): install Docker and start sample app
ssh ubuntu@$FIP_A 'sudo apt-get update && sudo apt-get install -y docker.io && sudo usermod -aG docker $USER'
ssh ubuntu@$FIP_A 'sudo docker run -d --name app -p 80:80 nginx:stable'
```
Deliverable: **IPs recorded**, both VMs reachable by SSH; `curl http://$FIP_A` returns nginx page.

---

## Task 1 — Build-Time Artifacts (simulate CI locally)

**On `vm-forensics`**
```bash
mkdir -p ~/CASE/10_build && cd ~/CASE/10_build
TZ=UTC date -u +"%Y-%m-%dT%H:%M:%SZ" | tee collected_at_utc.txt

# Minimal repo + Dockerfile
cat > Dockerfile <<'EOF'
FROM nginx:stable
ARG MSG="wk8"
RUN echo "$MSG" > /usr/share/nginx/html/wk8.txt
EOF

git init . && git add Dockerfile && git commit -m "wk8 demo"
git rev-parse HEAD | tee git_head.txt

# Build+push to local daemon only (no registry): create local image
docker build -t local/wk8:1.0 .

# Resolve digest via local ID (demo). In real DFIR, use registry digest.
img_id=$(docker images --digests local/wk8:1.0 --no-trunc | awk 'NR==2{print $3}')
echo "LOCAL_IMAGE_ID=$img_id" | tee image_ref.txt

# SBOM/scan (tag for demo; registry digest for real)
trivy image -q --format json --output vuln.json local/wk8:1.0 || true
```

Deliverables: `git_head.txt`, `image_ref.txt`, `vuln.json`, `collected_at_utc.txt`.

---

## Task 2 — Runtime Artifacts (on `vm-app` target)

**On `vm-app`**
```bash
mkdir -p ~/CASE/20_runtime && cd ~/CASE/20_runtime
CID=$(docker ps --filter name=app -q)
docker ps --no-trunc > docker_ps.txt
docker inspect "$CID" > ${CID}.inspect.json
docker logs --since 24h "$CID" > ${CID}.logs.txt
docker export "$CID" -o ${CID}.fs.tar

# Map host PID (for optional memory—only with approval)
pid=$(docker inspect -f '{{.State.Pid}}' "$CID"); echo $pid | tee host_pid.txt
```
Copy artifacts back to `vm-forensics` (scp or `rsync`).

---

## Task 3 — Networking (Host & OpenStack SDN)

### 3A. Host dataplane (on `vm-app`)
```bash
mkdir -p ~/CASE/30_network_host && cd ~/CASE/30_network_host
ip -d link show > links.txt
ip addr > addrs.txt
ip route > routes.txt
sudo iptables-save > iptables_rules.txt 2>/dev/null || true
sudo nft list ruleset > nft_ruleset.txt 2>/dev/null || true
sudo conntrack -S > conntrack_stats.txt 2>/dev/null || true
ss -tpna > sockets.txt
# 60s bounded pcap
TZ=UTC date -u +"%Y-%m-%dT%H:%M:%SZ" | tee capture_utc.txt
sudo timeout 60 tcpdump -i any -nn -s 0 -U -w host_60s.pcap
```

### 3B. OpenStack SDN (from `vm-forensics`, using CLI)
```bash
mkdir -p ~/CASE/31_network_cloud && cd ~/CASE/31_network_cloud
# Map server -> Neutron port(s) -> security groups -> router path
openstack server list --name vm-app -f json > os_servers.json
srv_id=$(jq -r '.[0].ID' os_servers.json)

openstack port list --server $srv_id -f json > os_ports.json
port_id=$(jq -r '.[0].ID' os_ports.json)
openstack port show $port_id -f json > os_port_show.json

# SGs, network, subnet, router, routes
jq -r '.[0]."Security Group"' os_ports.json | xargs -I{} openstack security group show {} -f json >> os_sg.json
openstack network show $NET -f json > os_net.json
openstack subnet show $SUB -f json > os_sub.json
openstack router show $ROUT -f json > os_router.json
```
Deliverables: host network snapshots + Neutron objects linking **process ⇄ IP ⇄ port ⇄ SG ⇄ router**.

---

## Task 4 — Storage (Volumes/Snapshots)

**Bind a Cinder volume to `vm-app`, write data, snapshot, attach RO to `vm-forensics`.**
```bash
# From forensics host (CLI)
openstack volume create --size 2 wk8-vol
openstack server add volume vm-app wk8-vol
openstack volume list -f json > cinder_vols.json
```

**On `vm-app`** (find and write)
```bash
lsblk           # find new disk, e.g., /dev/vdb
sudo mkfs.ext4 /dev/vdb
sudo mkdir -p /data && sudo mount /dev/vdb /data
echo "secret-wk8" | sudo tee /data/secret.txt
```

**Snapshot & attach RO**
```bash
# From forensics host
VOL_ID=$(openstack volume list -f value -c ID | head -n1)
openstack volume snapshot create --volume $VOL_ID wk8-snap
SNAP_ID=$(openstack snapshot list -f value -c ID | head -n1)
# Create temp volume from snapshot and attach to vm-forensics
openstack volume create --snapshot $SNAP_ID --size 2 wk8-snap-copy
openstack server add volume vm-forensics wk8-snap-copy
```

**On `vm-forensics`** (mount read-only, hash)
```bash
lsblk    # identify device, e.g., /dev/vdb or partition
sudo mkdir -p /mnt/ir
# ext4: avoid journal replay
sudo mount -o ro,noload /dev/vdb /mnt/ir
sha256sum /mnt/ir/* | tee ~/CASE/40_storage/SHA256SUMS.txt
sudo umount /mnt/ir
```

---

## Task 5 — Security (Supply Chain, Policies, Secrets)

**Supply chain (if using a registry; otherwise demonstrate locally)**
```bash
mkdir -p ~/CASE/50_security && cd ~/CASE/50_security
# Example uses local image tag; in real DFIR use registry digest:
docker images local/wk8:1.0 --no-trunc | tee images.txt
trivy image -q --format json --output vuln.json local/wk8:1.0 || true
```

**OpenStack identity/audit & secrets (if services enabled)**
```bash
mkdir -p ~/CASE/50_security/os
# Keystone role/assignment snapshots (who can do what)
openstack role list -f json > ~/CASE/50_security/os/roles.json
openstack project list -f json > ~/CASE/50_security/os/projects.json
openstack user list -f json > ~/CASE/50_security/os/users.json
# Barbican (if used for secrets)
openstack secret list -f json > ~/CASE/50_security/os/barbican_secrets.json 2>/dev/null || true
```

---

## Task 6 — Cloud Control Plane (OpenStack objects, events)
```bash
mkdir -p ~/CASE/31_network_cloud
openstack usage show --project $OS_PROJECT_ID -f json > os_usage.json
openstack server show vm-app -f json > os_vm_app.json
openstack server show vm-forensics -f json > os_vm_forensics.json
openstack image list -f json > os_images.json
openstack volume list -f json > os_volumes.json
openstack snapshot list -f json > os_snapshots.json
```

---

## Task 7 — Integrity & Turn-in
```bash
cd ~/CASE
TZ=UTC date -u +"%Y-%m-%dT%H:%M:%SZ" | tee 00_meta/collected_at_utc.txt
find . -type f ! -name SHA256SUMS.txt -exec sha256sum {} \; | sort -k2 > SHA256SUMS.txt
tar -czf wk8_openstack_case.tgz .
```
**Submit:** `wk8_openstack_case.tgz` + README with: app FIP, VM IDs, Neutron port ID, SG ID, volume/snapshot IDs, and 3–5 UTC events.

---

## Scoring rubric
| Area | Points |
|---|---|
| Build-time bundle (Dockerfile, git, image id, SBOM/scan) | 20 |
| Runtime bundle (inspect/logs/FS export) | 20 |
| Networking host + Neutron attribution (joins) | 20 |
| Storage snapshot RO mount + hashes | 20 |
| Security/identity snapshots (roles/users; optional Barbican) | 10 |
| Integrity (UTC + SHA256SUMS + clear README) | 10 |
