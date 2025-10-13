# AWS Lab — Cloud‑Native DFIR Artifact Collection
Student lab sheet (print & follow-along). Mirrors Week 8 topics. Copy/paste friendly.

> You will create two EC2 instances in the default VPC:
> - `ir-forensics` (jump host; collectors)
> - `ir-app` (runs a Dockerized app to investigate)

---

## Task 0 — Environment Setup

**Prereqs**
- AWS account with `AdministratorAccess` (or equivalent permissions for EC2, CloudWatch Logs, VPC, IAM read, ECR read, Secrets Manager read).
- AWS CLI v2 configured (`aws configure`) and key pair present for SSH.
- Default VPC exists in your region.

**Set variables (adjust region/profile/key)**
```bash
REGION=ap-southeast-1
PROFILE=default
KEY_NAME=mykey          # must already exist in EC2 > Key Pairs
AMI_AL2023=ami-0c9c942bd7bf113a2  # Amazon Linux 2023 example; verify in your region
SG_NAME=wk8-dfir-sg
TAG="wk8-dfir"
```

**Create a security group (SSH 22 + HTTP 80) in default VPC**
```bash
VPC_ID=$(aws ec2 describe-vpcs --filters Name=isDefault,Values=true \
  --query "Vpcs[0].VpcId" --output text --region $REGION --profile $PROFILE)
SG_ID=$(aws ec2 create-security-group --group-name "$SG_NAME" \
  --description "wk8 DFIR SG" --vpc-id "$VPC_ID" \
  --query GroupId --output text --region $REGION --profile $PROFILE)
aws ec2 authorize-security-group-ingress --group-id "$SG_ID" \
  --ip-permissions IpProtocol=tcp,FromPort=22,ToPort=22,IpRanges='[{CidrIp=0.0.0.0/0}]' \
  --region $REGION --profile $PROFILE
aws ec2 authorize-security-group-ingress --group-id "$SG_ID" \
  --ip-permissions IpProtocol=tcp,FromPort=80,ToPort=80,IpRanges='[{CidrIp=0.0.0.0/0}]' \
  --region $REGION --profile $PROFILE
```

**Launch instances**
```bash
# Forensics host
IR_FOR_ID=$(aws ec2 run-instances --image-id $AMI_AL2023 --instance-type t3.small \
  --key-name "$KEY_NAME" --security-group-ids "$SG_ID" \
  --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=ir-forensics},{Key=wk,Value=$TAG}]" \
  --query "Instances[0].InstanceId" --output text --region $REGION --profile $PROFILE)

# App host
IR_APP_ID=$(aws ec2 run-instances --image-id $AMI_AL2023 --instance-type t3.small \
  --key-name "$KEY_NAME" --security-group-ids "$SG_ID" \
  --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=ir-app},{Key=wk,Value=$TAG}]" \
  --query "Instances[0].InstanceId" --output text --region $REGION --profile $PROFILE)

# Allocate & associate Elastic IPs
EIP_FOR=$(aws ec2 allocate-address --domain vpc --query PublicIp --output text --region $REGION --profile $PROFILE)
EIP_APP=$(aws ec2 allocate-address --domain vpc --query PublicIp --output text --region $REGION --profile $PROFILE)
aws ec2 associate-address --instance-id "$IR_FOR_ID" --public-ip "$EIP_FOR" --region $REGION --profile $PROFILE
aws ec2 associate-address --instance-id "$IR_APP_ID" --public-ip "$EIP_APP" --region $REGION --profile $PROFILE

echo "Forensics: $EIP_FOR   App: $EIP_APP"
```

**Prepare hosts**
```bash
# SSH to forensics host (Amazon Linux); then to app host if needed
ssh -o StrictHostKeyChecking=no ec2-user@$EIP_FOR

# On ir-forensics: install tools
sudo yum -y install docker tcpdump jq git tar
sudo systemctl enable --now docker
sudo usermod -aG docker $USER
# Install extras (cosign + trivy)
curl -sSL https://raw.githubusercontent.com/sigstore/cosign/main/install.sh | sudo sh -s -- -b /usr/local/bin
curl -sSL https://aquasecurity.github.io/trivy-repo/rpm/install.sh | sudo bash
sudo yum -y install trivy

# On ir-app: install Docker and run sample app
ssh ec2-user@$EIP_APP 'sudo yum -y install docker && sudo systemctl enable --now docker && sudo usermod -aG docker $USER'
ssh ec2-user@$EIP_APP 'docker run -d --name app -p 80:80 nginx:stable'
```

Deliverable: both instances reachable; `curl http://$EIP_APP` returns the nginx welcome page.

---

## Task 1 — Build‑Time Artifacts (local demo)

**On `ir-forensics`**
```bash
mkdir -p ~/CASE/10_build && cd ~/CASE/10_build
TZ=UTC date -u +"%Y-%m-%dT%H:%M:%SZ" | tee collected_at_utc.txt

cat > Dockerfile <<'EOF'
FROM nginx:stable
ARG MSG="wk8"
RUN echo "$MSG" > /usr/share/nginx/html/wk8.txt
EOF
git init . && git add Dockerfile && git commit -m "wk8 demo"
git rev-parse HEAD | tee git_head.txt

docker build -t local/wk8:1.0 .
# Demo identity (local ID). In real DFIR, prefer ECR digest.
docker images --digests local/wk8:1.0 --no-trunc | tee image_index.txt
TRIVY_DB_DIR=/tmp/trivy trivy image -q --format json --output vuln.json local/wk8:1.0 || true
```

---

## Task 2 — Runtime Artifacts (on `ir-app` target)

**On `ir-app`**
```bash
mkdir -p ~/CASE/20_runtime && cd ~/CASE/20_runtime
CID=$(docker ps --filter name=app -q)
docker ps --no-trunc > docker_ps.txt
docker inspect "$CID" > ${CID}.inspect.json
docker logs --since 24h "$CID" > ${CID}.logs.txt
docker export "$CID" -o ${CID}.fs.tar
pid=$(docker inspect -f '{{.State.Pid}}' "$CID"); echo $pid | tee host_pid.txt
```

Copy artifacts back to `ir-forensics` if desired.

---

## Task 3 — Networking (Host & AWS SDN)

### 3A. Host dataplane (on `ir-app`)
```bash
mkdir -p ~/CASE/30_network_host && cd ~/CASE/30_network_host
ip -d link show > links.txt
ip addr > addrs.txt
ip route > routes.txt
sudo iptables-save > iptables_rules.txt 2>/dev/null || true
sudo nft list ruleset > nft_ruleset.txt 2>/dev/null || true
sudo conntrack -S > conntrack_stats.txt 2>/dev/null || true
ss -tpna > sockets.txt
TZ=UTC date -u +"%Y-%m-%dT%H:%M:%SZ" | tee capture_utc.txt
sudo timeout 60 tcpdump -i any -nn -s 0 -U -w host_60s.pcap
```

### 3B. AWS SDN (from `ir-forensics`)
```bash
mkdir -p ~/CASE/31_network_cloud && cd ~/CASE/31_network_cloud
START=$(date -u -d '-24 hours' +%Y-%m-%dT%H:%M:%SZ); END=$(date -u +%Y-%m-%dT%H:%M:%SZ)

# Instance + ENIs + SGs + routes
aws ec2 describe-instances --instance-ids "$IR_APP_ID" --region $REGION --profile $PROFILE > app_instance.json
aws ec2 describe-network-interfaces --filters Name=attachment.instance-id,Values=$IR_APP_ID \
  --region $REGION --profile $PROFILE > app_enis.json
aws ec2 describe-security-groups --group-ids "$SG_ID" --region $REGION --profile $PROFILE > sg.json
aws ec2 describe-route-tables --region $REGION --profile $PROFILE > routes.json

# VPC Flow Logs (if enabled): filter by time window
# Find the log group name in your environment; common: /aws/vpc/flow-logs
LOG_GROUP=/aws/vpc/flow-logs
aws logs filter-log-events --log-group-name "$LOG_GROUP" \
  --start-time $(date -u -d "$START" +%s000) \
  --end-time   $(date -u -d "$END"   +%s000) \
  --region $REGION --profile $PROFILE > vpc_flowlog_events.json || true
```

Deliverables: host network snapshots + AWS ENI/SG/route data + VPC Flow Logs (if available).

---

## Task 4 — Storage (EBS snapshot, RO attach)

**Create and attach an EBS volume to `ir-app`, write data, snapshot, attach to `ir-forensics` read‑only.**
```bash
# From ir-forensics
VOL_ID=$(aws ec2 create-volume --availability-zone $(aws ec2 describe-instances --instance-ids $IR_APP_ID \
  --query "Reservations[0].Instances[0].Placement.AvailabilityZone" --output text --region $REGION --profile $PROFILE) \
  --size 2 --tag-specifications "ResourceType=volume,Tags=[{Key=Name,Value=wk8-vol}]" \
  --query VolumeId --output text --region $REGION --profile $PROFILE)
aws ec2 wait volume-available --volume-ids $VOL_ID --region $REGION --profile $PROFILE
aws ec2 attach-volume --volume-id $VOL_ID --instance-id $IR_APP_ID --device /dev/xvdf --region $REGION --profile $PROFILE
```

**On `ir-app`** (format, mount, write)
```bash
lsblk
sudo mkfs.ext4 /dev/xvdf
sudo mkdir -p /data && sudo mount /dev/xvdf /data
echo "secret-wk8" | sudo tee /data/secret.txt
```

**Snapshot and attach to `ir-forensics`**
```bash
# From ir-forensics
SNAP_ID=$(aws ec2 create-snapshot --volume-id $VOL_ID --description "IR-snap" \
  --query SnapshotId --output text --region $REGION --profile $PROFILE)
aws ec2 wait snapshot-completed --snapshot-ids $SNAP_ID --region $REGION --profile $PROFILE
# Create a temp volume from snapshot in same AZ as ir-forensics
AZ_FOR=$(aws ec2 describe-instances --instance-ids $IR_FOR_ID \
  --query "Reservations[0].Instances[0].Placement.AvailabilityZone" --output text --region $REGION --profile $PROFILE)
VOL_IR=$(aws ec2 create-volume --snapshot-id $SNAP_ID --availability-zone $AZ_FOR \
  --query VolumeId --output text --region $REGION --profile $PROFILE)
aws ec2 wait volume-available --volume-ids $VOL_IR --region $REGION --profile $PROFILE
aws ec2 attach-volume --volume-id $VOL_IR --instance-id $IR_FOR_ID --device /dev/xvdg --region $REGION --profile $PROFILE
```

**On `ir-forensics`** (mount read‑only & hash)
```bash
lsblk
sudo mkdir -p /mnt/ir
# ext4: prevent journal replay
echo "Mounting read-only (ext4, noload)"
sudo mount -o ro,noload /dev/xvdg /mnt/ir
sha256sum /mnt/ir/* | tee ~/CASE/40_storage/SHA256SUMS.txt
sudo umount /mnt/ir
```

---

## Task 5 — Security (Supply Chain, IAM/Secrets)

**Supply chain (digest preferred if using ECR)**
```bash
mkdir -p ~/CASE/50_security && cd ~/CASE/50_security
# Example (local demo); for ECR images, use repo@sha256:<digest> with cosign/syft/grype
docker images local/wk8:1.0 --no-trunc | tee images.txt
TRIVY_DB_DIR=/tmp/trivy trivy image -q --format json --output vuln.json local/wk8:1.0 || true
```

**IAM and Secrets (read-only snapshots)**
```bash
aws iam list-roles --region $REGION --profile $PROFILE > iam_roles.json
aws iam list-users --region $REGION --profile $PROFILE > iam_users.json
aws secretsmanager list-secrets --region $REGION --profile $PROFILE > secrets_list.json || true
```

**CloudTrail (control plane audit)**
```bash
START=$(date -u -d '-24 hours' +%Y-%m-%dT%H:%M:%SZ); END=$(date -u +%Y-%m-%dT%H:%M:%SZ)
aws cloudtrail lookup-events --start-time "$START" --end-time "$END" \
  --region $REGION --profile $PROFILE > cloudtrail.json
```

---

## Task 6 — Cloud Control Plane (Describe assets)

```bash
mkdir -p ~/CASE/31_network_cloud
aws ec2 describe-instances --instance-ids $IR_FOR_ID $IR_APP_ID --region $REGION --profile $PROFILE > instances.json
aws ec2 describe-volumes --region $REGION --profile $PROFILE > volumes.json
aws ec2 describe-snapshots --owner-ids self --region $REGION --profile $PROFILE > snapshots.json
aws ec2 describe-network-interfaces --region $REGION --profile $PROFILE > enis.json
aws ec2 describe-security-groups --region $REGION --profile $PROFILE > sgs.json
```

---

## Task 7 — Integrity & Turn‑in

```bash
cd ~/CASE
TZ=UTC date -u +"%Y-%m-%dT%H:%M:%SZ" | tee 00_meta_utc.txt
find . -type f ! -name SHA256SUMS.txt -exec sha256sum {} \; | sort -k2 > SHA256SUMS.txt
tar -czf wk8_aws_case.tgz .
```

**Submit:** `wk8_aws_case.tgz` + README with: instance IDs, ENI IDs, SG ID, volume/snapshot IDs, VPC Flow Logs group name (if used), and 3–5 UTC events.

---

## Scoring rubric
| Area | Points |
|---|---|
| Build-time bundle (Dockerfile, git, image id, SBOM/scan) | 20 |
| Runtime bundle (inspect/logs/FS export) | 20 |
| Networking host + AWS SDN attribution (joins) | 20 |
| Storage snapshot RO mount + hashes | 20 |
| Security (CloudTrail/IAM/Secrets + supply chain) | 10 |
| Integrity (UTC + SHA256SUMS + clear README) | 10 |
