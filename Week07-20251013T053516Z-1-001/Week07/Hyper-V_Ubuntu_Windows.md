# Ubuntu Installation on Hyper-V using PowerShell

This guide walks you through installing **Ubuntu on Hyper-V** fully using **PowerShell**.

---

## 1. Enable Hyper-V

Open **PowerShell as Administrator** and run:

```powershell
Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V -All
```

Reboot your system after enabling Hyper-V.

---

## 2. Download Ubuntu ISO

Download the latest Ubuntu Desktop ISO from the [Ubuntu Official Website](https://ubuntu.com/download/desktop).

For this guide, save the file to:

```ini
E:\ISOs\ubuntu-22.04.5-desktop-amd64.iso

# or from prebuilt Hyper-V images
https://cloud-images.ubuntu.com/jammy/
# find the .vhd file "jammy-server-cloudimg-amd64-azure.vhd.tar.gz"
https://cloud-images.ubuntu.com/jammy/20250620/jammy-server-cloudimg-amd64-azure.vhd.tar.gz
```

---

## 3. Create a Virtual Switch

A virtual switch is needed for network access.

```powershell
# Variables
$SwitchName   = "UbuntuSwitch"
$NATSubnet    = "192.168.100.0/24"
$GatewayIP    = "192.168.100.1"
$PrefixLength = 24

# 1) Create Internal vSwitch
New-VMSwitch -Name $SwitchName -SwitchType Internal

# 2) Assign IP to host-side vEthernet adapter
New-NetIPAddress -InterfaceAlias "vEthernet ($SwitchName)" -IPAddress $GatewayIP -PrefixLength $PrefixLength

# 3) Create NAT mapping
New-NetNat -Name $SwitchName -InternalIPInterfaceAddressPrefix $NATSubnet

```

---

## 4. Create the Ubuntu VM

```powershell
New-VM -Name "UbuntuVM" -MemoryStartupBytes 4GB -Generation 2 -NewVHDPath "E:\VMs\UbuntuVM\Ubuntu.vhdx" -NewVHDSizeBytes 40GB -SwitchName "UbuntuSwitch"
```

---

## 5. Attach Ubuntu ISO

```powershell
Add-VMDvdDrive -VMName "UbuntuVM" -Path "E:\ISOs\ubuntu-22.04.5-desktop-amd64.iso"
#Verify
Get-VMDvdDrive -VMName "UbuntuVM" 
# --> Must return UbuntuVM SCSI .. path to .iso
```

---

## 6. Configure Boot Order

```powershell
$dvd = Get-VMDvdDrive -VMName "UbuntuVM"
Set-VMFirmware -VMName "UbuntuVM" -FirstBootDevice $dvd
(Get-VMFirmware -VMName "UbuntuVM").BootOrder
#--> DvdDrive must place in the 1st

```
---

## 7. Start the VM

```powershell
Set-VMFirmware -VMName "UbuntuVM" -EnableSecureBoot Off
Start-VM -Name "UbuntuVM"
```

---

## 8. Connect to VM Console

You can open the VM console with:

```powershell
vmconnect localhost UbuntuVM
```

---

## 9. Install Ubuntu

Inside the VM console:

* Select **Install Ubuntu**
* Choose language, keyboard, and disk options
* Create your username and password
* Complete the installation

## 10. Network setting use New-NetNat

# Or one-line

```powershel
$SwitchName="UbuntuNATSwitch";$NatName="UbuntuNAT";$HostIP="192.168.200.1";$PrefixLen=24;$NatPrefix="192.168.200.0/24";if(!(Get-VMSwitch -Name $SwitchName -ErrorAction SilentlyContinue)){New-VMSwitch -Name $SwitchName -SwitchType Internal|Out-Null};$IfAlias="vEthernet ($SwitchName)";if(!(Get-NetAdapter -Name $IfAlias -ErrorAction SilentlyContinue)){throw "Host vNIC '$IfAlias' not found"};Get-NetIPAddress -AddressFamily IPv4 -ErrorAction SilentlyContinue|?{ $_.IPAddress -eq $HostIP -and $_.InterfaceAlias -like "vEthernet (*)" -and $_.InterfaceAlias -ne $IfAlias }|Remove-NetIPAddress -Confirm:$false -ErrorAction SilentlyContinue;Get-NetIPAddress -InterfaceAlias $IfAlias -AddressFamily IPv4 -ErrorAction SilentlyContinue|Remove-NetIPAddress -Confirm:$false -ErrorAction SilentlyContinue;New-NetIPAddress -InterfaceAlias $IfAlias -IPAddress $HostIP -PrefixLength $PrefixLen -AddressFamily IPv4 -Type Unicast|Out-Null;Get-NetNat -Name $NatName -ErrorAction SilentlyContinue|Remove-NetNat -Confirm:$false;Get-NetNat -ErrorAction SilentlyContinue|?{ $_.InternalIPInterfaceAddressPrefix -eq $NatPrefix }|Remove-NetNat -Confirm:$false;New-NetNat -Name $NatName -InternalIPInterfaceAddressPrefix $NatPrefix|Out-Null;Get-VMSwitch -Name $SwitchName|ft Name,SwitchType;Get-NetIPAddress -InterfaceAlias $IfAlias -AddressFamily IPv4|ft IPAddress,PrefixLength;Get-NetNat|ft Name,InternalIPInterfaceAddressPrefix,Active

```

## 11. Connect

```powershell
Get-VM 
Get-VMSwitch
Connect-VMNetworkAdapter -VMName "your VM" -SwitchName <Switch>
vmconnect localhost UbuntuVM
---
Note (delete): Remove-VM -Name <"your VM">


## ✅ Summary

You now have an **Ubuntu VM running on Hyper-V** with:

* 4 GB RAM
* 40 GB virtual disk
* Network connectivity via NAT/External switch
* Bootable from Ubuntu ISO

---

## 🔧 Optional: Full Automation Script

Save as Install-UbuntuVM.ps1 and run in PowerShell (Admin)

```powershell
# -------------------
# Variables
# -------------------
$VMName   = "UbuntuVM"
$VMPath   = "E:\VMs\$VMName"
$VHDPath  = "$VMPath\Ubuntu.vhdx"
$ISOPath  = "E:\ISOs\ubuntu-22.04.5-desktop-amd64.iso"
$SwitchName = "UbuntuSwitch"
$NatName  = "UbuntuNAT"

# -------------------
# Enable Hyper-V
# -------------------
Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V -All -NoRestart

# -------------------
# Create VM folder
# -------------------
if (-not (Test-Path $VMPath)) {
    New-Item -Path $VMPath -ItemType Directory | Out-Null
}

# -------------------
# Create Virtual Switch (Internal + NAT)
# -------------------
if (-not (Get-VMSwitch -Name $SwitchName -ErrorAction SilentlyContinue)) {
    New-VMSwitch -Name $SwitchName -SwitchType Internal
    New-NetIPAddress -IPAddress 192.168.200.1 -PrefixLength 24 -InterfaceAlias $SwitchName
    New-NetNat -Name $NatName -InternalIPInterfaceAddressPrefix 192.168.200.0/24
}

# -------------------
# Create VM
# -------------------
if (-not (Get-VM -Name $VMName -ErrorAction SilentlyContinue)) {
    New-VM -Name $VMName -MemoryStartupBytes 2GB -Generation 2 -NewVHDPath $VHDPath -NewVHDSizeBytes 40GB -SwitchName $SwitchName -Path $VMPath
}

# -------------------
# Attach ISO
# -------------------
Set-VMDvdDrive -VMName $VMName -Path $ISOPath

# -------------------
# Fix Boot Order (boot from DVD first)
# -------------------
$fw = Get-VMFirmware -VMName $VMName
Set-VMFirmware -VMName $VMName -FirstBootDevice ($fw.BootOrder | Where-Object { $_.Device -like "*DVD*" })

# -------------------
# Disable Secure Boot (Ubuntu won’t boot with default template)
# -------------------
Set-VMFirmware -VMName $VMName -EnableSecureBoot Off

# -------------------
# Start VM
# -------------------
Start-VM -Name $VMName

# -------------------
# Open console
# -------------------
vmconnect localhost $VMName

```

---

💡 You now have a fully scripted and manual way to deploy Ubuntu on Hyper-V using PowerShell.
