# Forensic Lab & OWASP Juice Shop on AWS EC2 — Guide

This guide adapts the Docker-based DFIR and Juice Shop lab for **AWS EC2 virtual machines**.

---

## Table of Contents
1. [Prerequisites on AWS](#1-prerequisites-on-aws)  
2. [Launch an Ubuntu VM on EC2](#2-launch-an-ubuntu-vm-on-ec2)  
3. [SSH Access (Key Pair / Password)](#3-ssh-access-key-pair--password)  
4. [Install Docker on Ubuntu EC2](#4-install-docker-on-ubuntu-ec2)  
5. [DFIR Evidence Handling](#5-dfir-evidence-handling)  
6. [DFIR Containers on AWS VM](#6-dfir-containers-on-aws-vm)  
7. [OWASP Juice Shop on AWS VM](#7-owasp-juice-shop-on-aws-vm)  
8. [Networking & Security Groups](#8-networking--security-groups)  
9. [Troubleshooting Appendix](#9-troubleshooting-appendix)  

---

## 1) Prerequisites on AWS
- **AWS Account** with IAM permissions to launch EC2, manage Security Groups, Elastic IPs.  
- **Key Pair** (PEM file) downloaded for SSH access.  
- **Elastic IP** (recommended) if you need stable external IP.  
- AWS CLI installed (optional, for automation).  

---

## 2) Launch an Ubuntu VM on EC2
From AWS Console or CLI:

- **AMI**: `Ubuntu Server 22.04 LTS (amd64)`  
- **Instance type**: t3.medium or larger for DFIR labs  
- **Storage**: 50 GB+ recommended if handling images/pcaps  
- **Network**: VPC + subnet with internet access  
- **Security Groups**: allow SSH (22), HTTP (80), HTTPS (443), and custom DFIR ports if needed (e.g., 8080).  

Elastic IP (optional but recommended):  
```bash
aws ec2 allocate-address
aws ec2 associate-address --instance-id i-xxxx --allocation-id eipalloc-xxxx
```

---

## 3) SSH Access (Key Pair / Password)
From your local system:
```bash
chmod 400 mykey.pem
ssh -i mykey.pem ubuntu@<Elastic-IP>
```

If you need password login (less secure):  
```bash
sudo passwd ubuntu
# Edit /etc/ssh/sshd_config.d/60-cloudimg-settings.conf
#   PasswordAuthentication yes
sudo systemctl restart ssh
```

---

## 4) Install Docker on Ubuntu EC2
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
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
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

## 5) DFIR Evidence Handling
Mount **EBS volume** or **S3 bucket** for evidence:  

### Example: Attach an EBS volume
```bash
lsblk         # identify volume (e.g., /dev/xvdf)
sudo mkfs.ext4 /dev/xvdf
sudo mkdir /evidence
sudo mount /dev/xvdf /evidence
```

### Example: Sync from S3
```bash
aws s3 sync s3://my-dfir-bucket /evidence
```

Keep `/work` directory for analysis output:
```bash
sudo mkdir -p /work
sudo chown ubuntu:ubuntu /work
```

---

## 6) DFIR Containers on AWS VM
Examples (same as local/WSL):

### Volatility 3
```bash
docker run --rm -v /evidence:/evidence:ro -v /work:/out vol3:class \
  -f /evidence/mem.raw windows.pslist > /out/pslist.txt
```

### Zeek
```bash
docker run --rm -v /evidence:/evidence:ro -v /work:/out zeek:latest \
  zeek -r /evidence/traffic.pcap && mv *.log /out/
```

### Suricata
```bash
docker run --rm -v /evidence:/evidence:ro -v /work:/out suricata:latest \
  suricata -r /evidence/traffic.pcap -l /out/
```

---

## 7) OWASP Juice Shop on AWS VM
Deploy:
```bash
docker pull bkimminich/juice-shop:latest
docker run -d --name juice-shop -p 0.0.0.0:8080:3000 bkimminich/juice-shop
```

Test inside VM:
```bash
curl -I http://127.0.0.1:8080/
```

Test from outside (browser):
```
http://<Elastic-IP>:8080/
```

---

## 8) Networking & Security Groups
- AWS **Security Group** must allow inbound 8080/tcp for Juice Shop.  
- Best practice: restrict to your IP (`My IP`) instead of `0.0.0.0/0`.  

Check listeners:
```bash
ss -tlnp | grep 8080
```

---

## 9) Troubleshooting Appendix
- **Connection timeout**: likely missing inbound SG rule.  
- **Port 8080 blocked**: map container to port 80 (SG usually allows HTTP).  
- **Evidence too large**: attach bigger EBS or use S3 sync.  
- **No Docker perms**: re-run `newgrp docker`.  
- **Performance**: choose larger instance type (t3.large or m5.xlarge).  

---

## Checklist
- [ ] Elastic IP attached.  
- [ ] Key pair secure (`chmod 400`).  
- [ ] Docker installed + tested (`hello-world`).  
- [ ] Evidence mounted (`/evidence`, `/work`).  
- [ ] Juice Shop reachable at `http://<Elastic-IP>:8080/`.  
- [ ] Security Group rules configured.  
