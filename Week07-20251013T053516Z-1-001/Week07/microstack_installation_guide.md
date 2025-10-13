# 🚀 Quick OpenStack Setup with MicroStack (Ubuntu 22.04+)

This guide walks you through installing **OpenStack** the easiest way: using **MicroStack**.  
MicroStack is an **all-in-one snap package** that sets up OpenStack on a single machine in minutes — perfect for students and labs.  

---

## 0) Requirements

- Ubuntu 20.04 or 22.04 (desktop or server).
- At least **8 GB RAM**, **4 vCPUs**, **50 GB disk** (more is better).
- Internet connection.
- Sudo privileges.

---

## 1) Install MicroStack

Update system:

```bash
sudo apt update && sudo apt -y upgrade
```

Install MicroStack:

```bash
sudo snap install microstack --classic
```

---

## 2) Initialize OpenStack

Run initialization (non-interactive):

```bash
sudo microstack init --auto
```

This will:

- Configure Keystone (Identity).
- Set up Nova (Compute), Neutron (Networking), Glance (Image), Horizon (Dashboard).
- Start services automatically.

---

## 3) Access the Dashboard

- Open browser: [http://127.0.0.1](http://127.0.0.1) (if installed locally).  
- Or use the server’s IP: `http://<your-server-ip>`  

Login credentials:  

- **Username:** `admin`  
- **Password:** printed at the end of `microstack init`.  

You can also retrieve it later:

```bash
sudo snap get microstack config.credentials.keystone-password
```

---

## 4) OpenStack Client CLI

MicroStack includes the OpenStack CLI.  
To use it, source the admin credentials:

```bash
source /var/snap/microstack/common/etc/microstack.rc
```

Now you can run commands, e.g.:

```bash
openstack image list
openstack network list
```

---

## 5) Upload a Test Image

Download **CirrOS** (tiny Linux VM):

```bash
wget http://download.cirros-cloud.net/0.6.2/cirros-0.6.2-x86_64-disk.img
openstack image create "cirros"   --file cirros-0.6.2-x86_64-disk.img   --disk-format qcow2 --container-format bare --public
```

---

## 6) Launch a VM

Create a flavor:

```bash
openstack flavor create --id 1 --vcpus 1 --ram 512 --disk 1 m1.tiny
```

Boot a VM:

```bash
openstack server create --flavor m1.tiny --image cirros   --network test --security-group default vm1
```

Check status:

```bash
openstack server list
```

---

## 7) Manage MicroStack

Common commands:

```bash
# Show status
sudo microstack status

# Restart services
sudo systemctl restart snap.microstack.*

# Remove MicroStack completely
sudo snap remove microstack
```

---

## 8) References

- [MicroStack Documentation](https://microstack.run/)
- [OpenStack Docs](https://docs.openstack.org/)

---

✅ That’s it — in ~10 minutes, students have a **fully working OpenStack cloud** on their laptop or VM.
