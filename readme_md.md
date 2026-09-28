# PXE Server Setup Lab — Ubuntu 22.04.5

A comprehensive step-by-step guide to setting up a Network Boot (PXE) environment using VMware Workstation, Ubuntu 22.04.5 LTS, ISC DHCP, TFTP, GRUB/PXELINUX, and Apache HTTP Server.

---

## 📌 Lab Architecture

This lab uses two virtual machines running inside VMware Workstation:

* **VM1 — PXE Server:** Ubuntu 22.04.5 LTS (Dual NICs)
* **VM2 — PXE Client:** UEFI Boot Client (No OS installed)

### Network Overview

| Device | Interface | Purpose | IP / Subnet |
| :--- | :--- | :--- | :--- |
| **PXE Server** | `ens33` | NAT / Internet Access | `192.168.61.130/24` (DHCP) |
| **PXE Server** | `ens37` | Private PXE Network | `192.168.62.133/24` (Static) |
| **PXE Client** | Virtual NIC | Private PXE Network | Dynamic (Assigned via PXE Server) |

* **PXE Subnet:** `192.168.62.0/24`
* **VMware Network:** Custom (`VMnet2`)
* **VMware DHCP:** **Disabled** *(The custom Ubuntu PXE server handles DHCP)*

---

## 🔄 PXE Boot Flow

```text
               VMware Workstation (VMnet2)
                            │
        ┌───────────────────┴───────────────────┐
        │                                       │
   PXE Server                              PXE Client
 (Ubuntu 22.04)                          (UEFI Boot)
  192.168.62.133                                │
        │                                       │
        ├───────────────── DHCP Request ────────┤
        │ ── DHCP Reply (IP + BOOTX64.EFI) ───► │
        │                                       │
        ├───────────────── TFTP Request ────────┤
        │ ──────────── BOOTX64.EFI ───────────► │
        │                                       │
        ├────────────────── GRUB Menu ──────────┤
        │ ──────── vmlinuz + initrd ──────────► │
        │                                       │
        ├───────────── HTTP ISO Fetch ──────────┤
        │ ──────── Ubuntu Live Image ─────────► │
        ▼                                       ▼
  Server Ready                        Ubuntu Live Desktop
```

---

## 🛠️ Step-by-Step Setup Guide

### Step 1: VMware Network Configuration

#### 1.1 PXE Server Virtual Network Adapters
* **Adapter 1:** `NAT` (Used for internet access during software setup).
* **Adapter 2:** `Custom: VMnet2` (Isolated host-only network for PXE traffic).

#### 1.2 PXE Client Virtual Network Adapter
* **Adapter 1:** `Custom: VMnet2`
* Ensure **Connected** and **Connect at power on** are enabled.

#### 1.3 VMware Virtual Network Editor (`VMnet2`)
* **Subnet IP:** `192.168.62.0`
* **Subnet Mask:** `255.255.255.0`
* **Use local DHCP service:** **Disabled** (Uncheck this option)

---

### Step 2: Verify Server Network Interfaces

On the PXE Server, list active network interfaces and routing:

```bash
ip addr
```

*Expected output:*
* `ens33`: `192.168.61.130/24` (or similar NAT IP)
* `ens37`: Needs static configuration in Step 4.

Verify default routing:

```bash
ip route
```

*Expected route:* `default via 192.168.61.2 dev ens33` (Ensures traffic routes out to the Internet through `ens33`).

---

### Step 3: Install ISC DHCP Server

```bash
sudo apt update
sudo apt install isc-dhcp-server -y
```

---

### Step 4: Configure Static IP Using Netplan

The PXE server interface (`ens37`) requires a static IP address to consistently serve DHCP, TFTP, and HTTP requests.

#### 4.1 Locate Netplan Configuration

```bash
ls -l /etc/netplan/
```

#### 4.2 Edit Netplan Configuration

```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```

Update the configuration:

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    ens33:
      dhcp4: true
    ens37:
      dhcp4: false
      addresses:
        - 192.168.62.133/24
```

> **Note:** Do not define a `gateway4` or `routes` entry for `ens37`. The default gateway must remain on `ens33`.

#### 4.3 Apply Netplan Changes

```bash
sudo netplan generate
sudo netplan apply
```

Verify the static IP assignment:

```bash
ip addr show ens37
```

---

### Step 5: Configure DHCP Interface

Restrict DHCP service exclusively to the private network interface (`ens37`).

```bash
sudo nano /etc/default/isc-dhcp-server
```

Set the interface:

```text
INTERFACESv4="ens37"
```

---

### Step 6: Configure DHCP for PXE Booting

Edit `/etc/dhcp/dhcpd.conf`:

```bash
sudo nano /etc/dhcp/dhcpd.conf
```

Add the following configuration block:

```text
default-lease-time 600;
max-lease-time 7200;

subnet 192.168.62.0 netmask 255.255.255.0 {
    range 192.168.62.150 192.168.62.200;
    option routers 192.168.62.133;
    option broadcast-address 192.168.62.255;

    class "pxeclients" {
        match if substring (option vendor-class-identifier, 0, 9) = "PXEClient";
        next-server 192.168.62.133;
        filename "BOOTX64.EFI";
    }
}
```

#### DHCP Configuration Summary

| Parameter | Value |
| :--- | :--- |
| **Subnet** | `192.168.62.0/24` |
| **PXE Server IP** | `192.168.62.133` |
| **DHCP Pool Range** | `192.168.62.150` – `192.168.62.200` |
| **Boot File Target** | `BOOTX64.EFI` |

#### Validate and Restart DHCP Service

```bash
sudo dhcpd -t -cf /etc/dhcp/dhcpd.conf
sudo systemctl restart isc-dhcp-server
sudo systemctl status isc-dhcp-server
```

---

### Step 7: Install and Configure TFTP

#### 7.1 Install TFTP Server

```bash
sudo apt install tftpd-hpa -y
```

#### 7.2 Configure TFTP Service

Edit `/etc/default/tftpd-hpa`:

```bash
sudo nano /etc/default/tftpd-hpa
```

Set configuration values:

```text
TFTP_USERNAME="tftp"
TFTP_DIRECTORY="/var/lib/tftpboot"
TFTP_ADDRESS=":69"
TFTP_OPTIONS="--secure"
```

Restart TFTP service:

```bash
sudo systemctl restart tftpd-hpa
```

---

### Step 8: Install PXE Bootloaders

Install Syslinux and GRUB binaries for EFI boot support:

```bash
sudo apt install syslinux syslinux-common grub-efi-amd64 -y
```

Copy essential boot binaries into the TFTP directory:

```bash
# Copy Syslinux / Legacy binaries
sudo cp /usr/lib/PXELINUX/pxelinux.0 /var/lib/tftpboot/
sudo cp /usr/lib/syslinux/modules/bios/ldlinux.c32 /var/lib/tftpboot/
sudo cp /usr/lib/syslinux/modules/bios/menu.c32 /var/lib/tftpboot/

# Copy GRUB EFI Bootloader
sudo cp /usr/lib/grub/x86_64-efi/monolithic/grubnetx64.efi /var/lib/tftpboot/
sudo cp /var/lib/tftpboot/grubnetx64.efi /var/lib/tftpboot/BOOTX64.EFI

# Set permissions
sudo chown -R tftp:tftp /var/lib/tftpboot
sudo systemctl restart tftpd-hpa
```

---

### Step 9: Install and Configure Apache HTTP Server

#### 9.1 Install Apache

```bash
sudo apt install apache2 -y
sudo systemctl enable --now apache2
```

#### 9.2 Prepare Web Directory

```bash
sudo mkdir -p /var/www/html/ubuntu
```

#### 9.3 Place and Mount the Ubuntu ISO

Download or copy `ubuntu-22.04.5-desktop-amd64.iso` to `/var/www/html/ubuntu/`.

Create a mount point and mount the ISO:

```bash
sudo mkdir -p /mnt/ubuntu-iso
sudo mount -o loop /var/www/html/ubuntu/ubuntu-22.04.5-desktop-amd64.iso /mnt/ubuntu-iso
```

#### 9.4 Extract Kernel and Initrd to TFTP Boot Directory

```bash
sudo mkdir -p /var/lib/tftpboot/ubuntu2204

sudo cp /mnt/ubuntu-iso/casper/vmlinuz /var/lib/tftpboot/ubuntu2204/
sudo cp /mnt/ubuntu-iso/casper/initrd /var/lib/tftpboot/ubuntu2204/
```

---

### Step 10: Configure Bootloader Menus

#### 10.1 Configure GRUB Menu (UEFI Booting)

Create directory and configuration file:

```bash
sudo mkdir -p /var/lib/tftpboot/grub
sudo nano /var/lib/tftpboot/grub/grub.cfg
```

Add the following configuration:

```text
set timeout=5

menuentry "Ubuntu 22.04.5 Desktop Live" {
    linux /ubuntu2204/vmlinuz ip=dhcp url=http://192.168.62.133/ubuntu/ubuntu-22.04.5-desktop-amd64.iso quiet splash ---
    initrd /ubuntu2204/initrd
}
```

#### 10.2 Configure PXELINUX Menu (Legacy BIOS Booting)

```bash
sudo mkdir -p /var/lib/tftpboot/pxelinux.cfg
sudo nano /var/lib/tftpboot/pxelinux.cfg/default
```

Add the following configuration:

```text
DEFAULT menu.c32
PROMPT 0
TIMEOUT 50

LABEL ubuntu2204
    MENU LABEL Ubuntu 22.04.5 Desktop Live
    KERNEL ubuntu2204/vmlinuz
    INITRD ubuntu2204/initrd
    APPEND ip=dhcp url=http://192.168.62.133/ubuntu/ubuntu-22.04.5-desktop-amd64.iso quiet splash ---
```

#### 10.3 Permissions Finalization

```bash
sudo chown -R tftp:tftp /var/lib/tftpboot
sudo systemctl restart tftpd-hpa
```

---

## 🧪 Testing and Verification

### 1. Verify HTTP Access
Ensure Apache can serve the ISO image:

```bash
curl -I http://192.168.62.133/ubuntu/ubuntu-22.04.5-desktop-amd64.iso
```
*Expected response:* `HTTP/1.1 200 OK`

### 2. Verify TFTP Access
Install TFTP client and attempt downloading boot files:

```bash
sudo apt install tftp-hpa -y
tftp localhost -c get BOOTX64.EFI
ls -lh BOOTX64.EFI
```

### 3. Monitor Network Packets Live
On the PXE server, capture live network traffic to observe DHCP requests and responses:

```bash
sudo tcpdump -i ens37 -n -e
```

### 4. Boot the PXE Client VM
1. Power on **VM2 (PXE Client)**.
2. Ensure Firmware is set to **UEFI** and **Secure Boot** is **disabled**.
3. Observe client acquiring `192.168.62.150`, fetching `BOOTX64.EFI`, loading GRUB, downloading the ISO via HTTP, and booting into Ubuntu Live Desktop.

---

## 🔍 Troubleshooting Guide

| Issue | Root Cause | Solution |
| :--- | :--- | :--- |
| **`PXE-E51: No DHCP or proxyDHCP offers`** | DHCP server not listening or wrong VMware VMnet interface. | Verify `INTERFACESv4="ens37"` in `/etc/default/isc-dhcp-server`, verify `VMnet2` settings, and check `tcpdump -i ens37`. |
| **`PXE-E79: NBP is too big`** | UEFI client attempting to boot Legacy BIOS bootloader files. | Set client firmware to UEFI or verify target bootfile is set to `BOOTX64.EFI`. |
| **TFTP File Not Found** | Incorrect paths or permissions inside `/var/lib/tftpboot`. | Run `sudo chown -R tftp:tftp /var/lib/tftpboot` and check path case-sensitivity. |
| **ISO Download Failed** | Apache web server offline or invalid path in `grub.cfg`. | Verify `http://192.168.62.133/ubuntu/...` using `curl` and check `apache2` logs via `journalctl -u apache2`. |

---

## 📂 Final File and Directory Structure

```text
/etc/
├── dhcp/
│   └── dhcpd.conf
├── default/
│   ├── isc-dhcp-server
│   └── tftpd-hpa
└── netplan/
    └── 50-cloud-init.yaml

/var/lib/tftpboot/
├── BOOTX64.EFI
├── grubnetx64.efi
├── ldlinux.c32
├── menu.c32
├── pxelinux.0
├── grub/
│   └── grub.cfg
├── pxelinux.cfg/
│   └── default
└── ubuntu2204/
    ├── vmlinuz
    └── initrd

/var/www/html/
└── ubuntu/
    └── ubuntu-22.04.5-desktop-amd64.iso
```