# Securing and best practices in **Proxmox VE (Virtual Environment)** 

is critical, especially in production or internet-facing deployments. Here's a comprehensive guide divided into **best practices** and **security recommendations**:

---

## 🔧 **Best Practices for Proxmox VE**

### 1. **Installation and Base Configuration**

* **Use the latest Proxmox ISO** to benefit from the newest features and fixes.
* **Use ZFS** for storage if you want snapshots, replication, and data integrity (scrubbing, checksumming).
* **Configure a static IP** during installation to avoid DHCP-related disruptions.

### 2. **Storage Management**

* Use **LVM-Thin** for local storage if you want better snapshot and space management.
* Keep **VM disks separate** from ISO/backup repositories.
* Monitor disk I/O and use `iotop`, `zpool iostat`, or `pveperf` to benchmark and diagnose.

### 3. **Backup Strategy**

* Set **automated scheduled backups** using vzdump (daily or weekly depending on criticality).
* Store backups **on separate storage** (NFS, CIFS, or external ZFS).
* Test restore **periodically** to ensure backups are valid.

### 4. **High Availability (HA)**

* Use **Proxmox Cluster + HA Manager** for critical VM workloads.
* Ensure **shared storage** is available if HA is required.
* At least **3 nodes for quorum** in clusters to prevent split-brain.

---

## 🔒 **Security Best Practices**

### 1. **Authentication & Access Control**

* **Disable root SSH login**:

  ```bash
  PermitRootLogin no
  ```
* Use **SSH key authentication** instead of passwords.
* Enforce **2FA** on Proxmox web UI (`Datacenter > Permissions > Two-Factor Authentication`).
* Create **non-root user accounts** with appropriate roles using Proxmox ACLs.

### 2. **Firewall Configuration**

* Enable and configure the **Proxmox Firewall** (node level + datacenter level).
* Apply **"drop all except..."** rules for management interfaces.
* Limit access to port **8006 (Proxmox web UI)** using the firewall or reverse proxy.

### 3. **HTTPS & Certificates**

* Replace self-signed SSL certs with **Let's Encrypt** or internal CA certificates.
* Regularly **rotate TLS certificates** and disable older TLS versions (e.g., TLS 1.0, 1.1).

### 4. **System Hardening**

* Keep **Proxmox and all packages up to date**:

  ```bash
  apt update && apt full-upgrade
  ```
* Disable unused services (e.g., NFS, CIFS, Avahi if not in use).
* Configure **AppArmor** or **SELinux** where possible.
* Consider **auditing** with tools like `auditd`.

### 5. **Network Isolation**

* Isolate VM traffic from management traffic using **VLANs** or **separate bridges**.
* Do not expose the management interface to the public internet.
* Use **VPN access** (e.g., WireGuard or OpenVPN) for remote administration.

### 6. **Logging and Monitoring**

* Use **remote syslog** or **ELK stack** for centralized logging.
* Monitor via **Prometheus + Grafana**, or use Proxmox’s built-in metrics server.
* Set up **alerts** for high load, failed backups, disk space issues, etc.

### 7. **Updates and Patch Management**

* Regularly **update the kernel** and reboot for security patches.
* Subscribe to the **Proxmox Enterprise Repository** for stable updates (recommended for production).

---

## 🔁 Optional: Advanced Hardening

* Use **fail2ban** with custom filters for Proxmox web UI and SSH.
* Restrict `sudo` access with tight controls and logging.
* Implement **IP-based rate limiting** via reverse proxy (e.g., NGINX in front of the web GUI).

---

# Ansible-compatible playbook

Here are both versions you asked for:

---

## ✅ **Proxmox VE Best Practices & Security Checklist**

### 🔧 **Base Configuration**

* [ ] Install latest Proxmox VE ISO
* [ ] Use static IP address
* [ ] Use ZFS (or LVM-Thin) for storage
* [ ] Separate storage for VM disks and backups

### 💾 **Backup & Restore**

* [ ] Schedule regular `vzdump` backups
* [ ] Store backups on external storage (NFS/CIFS)
* [ ] Test restore process periodically

### 🖧 **Clustering and High Availability**

* [ ] Minimum 3 nodes for quorum
* [ ] Enable HA for critical VMs
* [ ] Use shared storage for HA

---

### 🔐 **Security**

#### 🔐 SSH and Access

* [ ] Disable root SSH login
* [ ] Use SSH keys instead of passwords
* [ ] Create non-root user accounts with correct roles
* [ ] Enforce 2FA in web UI

#### 🔒 Firewall & Access Control

* [ ] Enable Proxmox firewall (node and datacenter levels)
* [ ] Limit port 8006 to trusted IPs
* [ ] Isolate management and VM traffic (VLANs or bridges)

#### 🌐 TLS & Certificates

* [ ] Replace self-signed certs with Let’s Encrypt or internal CA
* [ ] Disable TLS 1.0 and 1.1

#### 🛡️ System Hardening

* [ ] Regularly update OS and Proxmox packages
* [ ] Disable unused services (Avahi, NFS, etc.)
* [ ] Enable AppArmor
* [ ] Install and configure `auditd`

#### 🧰 Monitoring & Logging

* [ ] Forward logs to remote syslog or ELK
* [ ] Install Prometheus + Grafana (or use built-in metrics)
* [ ] Set alerts for system health, backups, etc.

#### 🔁 Advanced Hardening (Optional)

* [ ] Install `fail2ban` for SSH and Proxmox UI
* [ ] Use reverse proxy (e.g., NGINX) to rate-limit and expose UI via HTTPS
* [ ] Restrict sudo access and log usage

---

## ⚙️ **Ansible-Compatible Hardening Playbook (Basic Version)**

Here’s a simplified **Ansible playbook** for securing a single-node Proxmox VE host.

```yaml
---
- name: Harden Proxmox VE Host
  hosts: proxmox
  become: true
  vars:
    ssh_port: 22
    allow_ips: ["192.168.1.0/24"]  # Adjust to your trusted IPs

  tasks:

    - name: Update all packages
      apt:
        update_cache: yes
        upgrade: full

    - name: Disable root SSH login
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^PermitRootLogin'
        line: 'PermitRootLogin no'
        backup: yes
      notify: Restart SSH

    - name: Set up UFW firewall rules (allow SSH & Proxmox UI)
      ufw:
        rule: allow
        port: "{{ ssh_port }}"
        proto: tcp

    - name: Allow Proxmox Web UI from trusted IPs
      ufw:
        rule: allow
        port: 8006
        proto: tcp
        from_ip: "{{ item }}"
      loop: "{{ allow_ips }}"

    - name: Enable UFW
      ufw:
        state: enabled

    - name: Install fail2ban
      apt:
        name: fail2ban
        state: present

    - name: Ensure AppArmor is enabled
      command: aa-status
      register: aa_status
      changed_when: false
      failed_when: "'disabled' in aa_status.stdout"

    - name: Install auditd
      apt:
        name: auditd
        state: present

  handlers:
    - name: Restart SSH
      service:
        name: ssh
        state: restarted
```

---

Great! Here's a **comprehensive Ansible setup** for:

1. A **detailed Proxmox role** (`roles/proxmox_hardening`)
2. A **multi-node cluster setup**
3. A **template inventory file** with `group_vars`

This is designed to help bootstrap secure Proxmox VE clusters in a reproducible way.

---

## 📁 Directory Structure

```
proxmox-ansible/
├── inventory/
│   └── hosts.yml
├── group_vars/
│   └── proxmox_nodes.yml
├── playbooks/
│   └── cluster_setup.yml
├── roles/
│   └── proxmox_hardening/
│       ├── tasks/
│       │   ├── main.yml
│       │   ├── firewall.yml
│       │   └── acl.yml
│       └── handlers/
│           └── main.yml
```

---

## 🔐 `roles/proxmox_hardening/tasks/main.yml`

```yaml
---
- name: Ensure latest packages are installed
  apt:
    update_cache: yes
    upgrade: full

- name: Disable root SSH login
  lineinfile:
    path: /etc/ssh/sshd_config
    regexp: '^PermitRootLogin'
    line: 'PermitRootLogin no'
  notify: Restart SSH

- name: Install fail2ban and auditd
  apt:
    name: "{{ item }}"
    state: present
  loop:
    - fail2ban
    - auditd

- name: Configure Proxmox firewall
  include_tasks: firewall.yml

- name: Set up Proxmox ACLs
  include_tasks: acl.yml
```

---

## 🔥 `roles/proxmox_hardening/tasks/firewall.yml`

```yaml
---
- name: Enable firewall at datacenter level
  command: pve-firewall enable

- name: Enable firewall at host level
  command: pve-firewall enable --host

- name: Allow SSH and web UI (port 8006)
  shell: |
    pve-firewall add --group proxmox-allow --macro SSH
    pve-firewall add --group proxmox-allow --macro web
  args:
    executable: /bin/bash
```

---

## 🔐 `roles/proxmox_hardening/tasks/acl.yml`

```yaml
---
- name: Create 'adminuser' user if not exists
  command: "pveum user add adminuser@pve --password yourSecurePassword"
  ignore_errors: yes

- name: Assign admin role to user
  command: "pveum aclmod / -user adminuser@pve -role Administrator"
```

---

## 🛠️ `roles/proxmox_hardening/handlers/main.yml`

```yaml
---
- name: Restart SSH
  service:
    name: ssh
    state: restarted
```

---

## 🌐 `group_vars/proxmox_nodes.yml`

```yaml
---
ssh_port: 22
allow_ips:
  - "192.168.1.0/24"
admin_user: "adminuser"
admin_password: "yourSecurePassword"
```

---

## 🖧 `inventory/hosts.yml`

```yaml
all:
  children:
    proxmox_nodes:
      hosts:
        proxmox1.local:
        proxmox2.local:
        proxmox3.local:
```

---

## 📋 `playbooks/cluster_setup.yml`

```yaml
---
- name: Harden Proxmox and setup cluster
  hosts: proxmox_nodes
  become: true

  roles:
    - proxmox_hardening

- name: Create Proxmox cluster (run only on first node)
  hosts: proxmox1.local
  become: true
  tasks:
    - name: Create cluster if not exists
      shell: |
        pvecm create mycluster
      args:
        creates: /etc/pve/corosync.conf

- name: Join other nodes to cluster
  hosts: proxmox_nodes:!proxmox1.local
  become: true
  tasks:
    - name: Join node to cluster
      shell: |
        pvecm add proxmox1.local -use_ssh yes -force
      args:
        creates: /etc/pve/corosync.conf
```

---

## 🏁 Usage

Run the playbook with:

```bash
ansible-playbook -i inventory/hosts.yml playbooks/cluster_setup.yml
```

---

## 🚨 Notes

* Ensure passwordless SSH (or SSH keys) are configured among the nodes.
* `pvecm add` assumes `proxmox1.local` is reachable and trusted.
* You may want to add retry logic and health checks for production use.

---




