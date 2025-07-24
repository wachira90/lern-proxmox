# Lerning Proxmox

**step-by-step guide to install Proxmox Virtual Environment (Proxmox VE)** on a bare-metal server:

---

## 🔧 **Prerequisites**

* A 64-bit capable CPU (Intel VT-x or AMD-V recommended)
* Minimum 2 GB RAM (4 GB+ recommended)
* At least 1 storage drive (SSD/HDD)
* Bootable USB drive (4 GB+)
* A system where you can wipe all data (Proxmox uses full disk by default)
* Internet connection (optional but recommended for updates)

---

## 🪛 Step-by-Step Installation

### **Step 1: Download Proxmox VE ISO**

1. Go to the official Proxmox VE download page:
   [https://www.proxmox.com/en/downloads](https://www.proxmox.com/en/downloads)
2. Download the latest **Proxmox VE ISO Installer**.

---

### **Step 2: Create a Bootable USB**

Use a tool like:

* **Rufus** (Windows)
* **balenaEtcher** (macOS/Linux/Windows)
* **dd** (Linux/macOS)

Example using `dd` (Linux/macOS):

```bash
sudo dd if=proxmox-ve_*.iso of=/dev/sdX bs=4M status=progress && sync
```

> Replace `/dev/sdX` with your actual USB device (e.g., `/dev/sdb`)

---

### **Step 3: Boot From USB**

1. Insert the USB into your server.
2. Power on and enter BIOS/UEFI (usually `F2`, `F10`, `Del`, or `Esc`)
3. Set USB as the first boot device.
4. Save and exit BIOS to boot into the Proxmox installer.

---

### **Step 4: Install Proxmox VE**

1. Select **“Install Proxmox VE”** from the boot menu.
2. Accept the **license agreement**.
3. Choose the **target disk** (this will be wiped).
4. Set:

   * **Country**, **timezone**, and **keyboard layout**
   * **Admin password** and **email**
5. Set network configuration:

   * Hostname (e.g., `proxmox.local`)
   * IP address (static recommended), gateway, DNS
6. Click **Install** and wait (approx. 5–15 mins)

---

### **Step 5: First Boot & Access the Web UI**

1. After installation, the system will reboot.

2. Remove the USB.

3. Once booted, you’ll see a screen showing the **Web UI address**, e.g.:

   ```
   You can now connect to https://192.168.1.100:8006
   ```

4. Open that URL in your browser (ignore the HTTPS warning).

5. Login with:

   * **User:** `root`
   * **Password:** (the one you set during install)
   * **Realm:** `pam`

---

### ✅ **You're in!**

You can now:

* Create virtual machines (VMs)
* Set up containers (LXC)
* Configure storage (ZFS, LVM, etc.)
* Add nodes for clustering
* Manage backups and networking

---

## 🧩 Post-Installation Tips

* **Update your system**:

  ```bash
  apt update && apt full-upgrade -y
  ```

* **Add Proxmox No-Subscription Repo** (optional for home use):

  ```bash
  echo "deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription" > /etc/apt/sources.list.d/pve-no-subscription.list
  apt update
  ```

* **Remove the enterprise repo** (optional):

  ```bash
  sed -i "s/^deb/#deb/" /etc/apt/sources.list.d/pve-enterprise.list
  ```

---

Would you like help with setting up your first VM or container as the next step?
