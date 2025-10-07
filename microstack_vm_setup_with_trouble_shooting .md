# MicroStack Setup & VM Deployment Guide

## 1. Delete old version

```bash
sudo microstack.remove --auto
sudo microstack.remove --auto --purge
```

## 2. Setup new one

```bash
sudo snap install microstack --edge
```

## 3. Initialize the system

```bash
sudo microstack.init --auto --control
```

## 4. Check services

```bash
microstack.openstack network list
microstack.openstack flavor list
microstack.openstack keypair list
microstack.openstack image list   # --> includes cirros
microstack.openstack security group rule list
```

## 5. Deploy

### 5.1 Create security group

```bash
microstack.openstack security group create --description Allowall all-allow
```

### 5.2 Allow inbound traffic

```bash
microstack.openstack security group rule create --proto tcp --dst-port 1:65535 --ingress all-allow
microstack.openstack security group rule create --proto udp --dst-port 1:65535 --ingress all-allow
microstack.openstack security group rule create --proto icmp --ingress all-allow
```

### 5.3 Allow outbound traffic

```bash
microstack.openstack security group rule create --proto tcp --dst-port 1:65535 --egress all-allow
microstack.openstack security group rule create --proto udp --dst-port 1:65535 --egress all-allow
microstack.openstack security group rule create --proto icmp --egress all-allow
```

### 5.4 Check security group and rules

```bash
microstack.openstack security group list
microstack.openstack security group rule list
```

## 6. Download Ubuntu image OS

```bash
sudo apt install aria2
aria2c https://cloud-images.ubuntu.com/jammy/current/jammy-server-cloudimg-amd64-disk-kvm.img -o ubuntu22_kvm.img
```

## 7. Upload image to OpenStack

```bash
microstack.openstack image create "ubuntu22.04" \
  --disk-format qcow2 \
  --container-format bare \
  --file ubuntu22_kvm.img \
  --public
```

## 8. Create VMs

### 8.1 SSH key pair (no pass)

```bash
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519
chmod 600 ~/.ssh/id_ed25519
chmod 700 ~/.ssh
```

### 8.2 Upload key to OpenStack

```bash
microstack.openstack keypair create --public-key ~/.ssh/id_ed25519.pub ecckey-u2204
# Verify
microstack.openstack keypair list
```

Must include `ecckey-u2204`.

### 8.3 Create VM instance

- Select flavor:

```bash
microstack.openstack flavor list
```

- Select security group:

```bash
microstack.openstack security group list
```

- Select image:

```bash
microstack.openstack image list
```

- Select keypair:

```bash
microstack.openstack keypair list
```

- Select network:

```bash
microstack.openstack network list
```

#### Create VM

```bash
microstack.openstack server create vm-u2204 \
  --flavor m1.small \
  --image ubuntu22.04 \
  --key-name ecckey-u2204 \
  --security-group all-allow \
  --network test
```

#### Create a Floating IP for the VM

```bash
microstack.openstack floating ip create external
# Example result: 10.20.20.121
```

#### Assign Floating IP to VM

```bash
microstack.openstack server add floating ip vm-u2204 <FLOATING_IP>
# Example:
# microstack.openstack server add floating ip vm-u2204 10.20.20.121
```

#### Check VMs

```bash
microstack.openstack server list
```

Should return `vm-u2204: ACTIVE`.

#### Connect to VM using SSH

```bash
ssh -i ~/.ssh/id_ed25519 ubuntu@<FLOATING_IP>
# Example:
ssh -i ~/.ssh/id_ed25519 ubuntu@10.20.20.121
```

---

## Troubleshooting `Permission denied (publickey)`

Check:

```bash
microstack.openstack keypair show ecckey-u2204 -f yaml
```

If `updated_at: null` → your key was not uploaded to OpenStack.

### Action 1: Must set `--config-drive true` when creating a VM

1. Delete old key:

```bash
microstack.openstack keypair delete ecckey-u2204
```

2. Re-upload:

```bash
microstack.openstack keypair create --public-key ~/.ssh/id_ed25519.pub ecckey-u2204
```

3. Verify (may ignore `updated_at: null` ):

```bash
microstack.openstack keypair show ecckey-u2204 -f yaml
```

4. Delete and rebuild your VM, then set floating IP again:

```bash
microstack.openstack server delete <VM_ID>

microstack.openstack server create --flavor m1.small \
  --image 43369044-4652-4460-9f15-7abcc612688d \
  --key-name ecckey-u2204 \
  --security-group all-allow \
  --network test \
  --config-drive true \
  vm-u2204

microstack.openstack server add floating ip vm-u2204 10.20.20.121
```

5. Delete old fingerprint and reconnect via SSH:

```bash
ssh-keygen -f ~/.ssh/known_hosts -R 10.20.20.121
ssh -i ~/.ssh/id_ed25519 ubuntu@10.20.20.121
```
