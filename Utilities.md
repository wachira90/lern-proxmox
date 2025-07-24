# ตัวอย่าง ha-manager config export

# การทำ Live Migration อัตโนมัติเมื่อ CPU/Memory สูง

# Monitoring integration กับ Zabbix, Prometheus

---

## 📦 1. **ตัวอย่าง `ha-manager config` (Export Format)**

`ha-manager config` ใช้ดูการตั้งค่าทั้งหมดของ HA resources (VMs), HA groups และ options

### 📋 ตัวอย่าง output:

```text
group: hagrp1
    nodes pve1,pve2
    nofailback 1

resource: vm:100
    group hagrp1
    state started
    max_relocate 3
    max_restart 2

resource: vm:101
    group hagrp1
    state disabled
```

### 🔧 ความหมาย:

* `group`: กลุ่มที่ VM สามารถย้ายไป-มาได้ระหว่าง node
* `nofailback 1`: เมื่อ node ที่ล่มกลับมา VM จะไม่ย้ายกลับ
* `state started`: VM จะเริ่มทันทีเมื่อ node พร้อม
* `max_restart`, `max_relocate`: จำกัดจำนวน restart / relocate ก่อนถือว่าล้มเหลวจริง

> คุณสามารถ export เป็นไฟล์ด้วย:

```bash
ha-manager config > /root/ha-config-backup.txt
```

หรือ parse ผ่าน `pvesh`:

```bash
pvesh get /cluster/ha/resources
```

---

## 🚀 2. **Live Migration อัตโนมัติเมื่อ CPU/Memory สูง (Script-Based Automation)**

Proxmox VE **ไม่มี built-in Auto Live Migration ตาม resource usage** แต่สามารถทำได้ด้วย **Custom Script + Cron + API**

### 🧠 แนวคิด:

1. ใช้ `pvesh` หรือ `pveperf`, `top`, `psutil` (หรือ Prometheus node exporter)
2. ตรวจสอบ CPU/Mem บนแต่ละ node
3. ถ้าเกิน threshold → ใช้ `qm migrate` เพื่อย้าย VM ไป node ที่โล่ง

---

### 🔧 ตัวอย่าง Bash Script: `auto-migrate.sh`

```bash
#!/bin/bash

THRESHOLD=85  # % CPU ที่ถือว่าเกิน
NODE=$(hostname)

CPU_USAGE=$(mpstat 1 1 | awk '/Average/ && $12 ~ /[0-9.]+/ { print 100 - $12 }' | cut -d. -f1)

if [[ $CPU_USAGE -gt $THRESHOLD ]]; then
    echo "[INFO] $NODE CPU is high: $CPU_USAGE%"

    for vmid in $(qm list | awk 'NR>1 {print $1}'); do
        TARGET=$(pvesh get /nodes | jq -r '.[] | select(.node != "'$NODE'" and .status == "online") | .node' | head -n 1)

        echo "[ACTION] Migrating VM $vmid to $TARGET"
        qm migrate $vmid $TARGET --online
        sleep 10
    done
fi
```

### 📌 ใช้งาน:

* ติดตั้ง `sysstat` (`apt install sysstat`)
* ตั้ง Cron job:

```bash
*/5 * * * * /root/auto-migrate.sh >> /var/log/auto-migrate.log 2>&1
```

---

## 📊 3. **Monitoring Integration กับ Zabbix / Prometheus**

### ✅ A. **Zabbix Integration (Proxmox VE Template)**

Proxmox สามารถใช้ร่วมกับ Zabbix ได้ผ่าน:

#### วิธีที่ 1: ติดตั้ง Zabbix Agent บนแต่ละ Node

```bash
apt install zabbix-agent
```

#### วิธีที่ 2: ใช้ Proxmox API + External Script

* ใช้ `zabbix_sender` ดึงข้อมูลจาก `pvesh`
* Zabbix Template ที่นิยม:

  * [Zabbix Proxmox Template by kletellier](https://github.com/kletellier/Zabbix-Template-Proxmox)

#### ตัวอย่าง Key ที่น่าสนใจ:

| Key                      | คำอธิบาย          |
| ------------------------ | ----------------- |
| `vfs.fs.size[/,used]`    | ใช้ disk เท่าไหร่ |
| `proc.num[]`             | จำนวน process     |
| `system.cpu.util[,user]` | การใช้งาน CPU     |
| `custom.pve.vm.cpu[100]` | CPU ของ VM 100    |

---

### ✅ B. **Prometheus Integration**

#### วิธีที่นิยม:

1. ติดตั้ง **Prometheus Node Exporter** บนทุก Node

```bash
apt install prometheus-node-exporter
```

2. เพิ่ม Proxmox Node เข้า Prometheus:

```yaml
- job_name: 'proxmox'
  static_configs:
    - targets: ['pve1:9100', 'pve2:9100']
```

3. ใช้ Grafana Dashboard:

* Dashboard ID: `10048`
  (ชื่อ: “Proxmox VE Nodes - CPU, RAM, Disk”)

4. หรือใช้ **Prometheus PVE Exporter**:

```bash
git clone https://github.com/prometheus-pve/prometheus-pve-exporter
```

* เข้าถึงข้อมูลผ่าน Proxmox API (ใส่ API Token)
* รองรับ metric เช่น:

  * VM CPU/RAM
  * Node Health
  * HA status

---

## ✅ สรุปแบบชัดเจน

| Feature                | วิธี                                                |
| ---------------------- | --------------------------------------------------- |
| Export HA Config       | `ha-manager config > file.txt`                      |
| Auto Live Migration    | Bash Script + `qm migrate`                          |
| Zabbix Integration     | Agent + Template + API script                       |
| Prometheus Integration | Node Exporter + Grafana Dashboard หรือ API Exporter |

---

หากคุณต้องการ:

* ตัวอย่าง config พร้อม Dashboard JSON
* Script สำหรับ Zabbix auto-discovery VMs
* Prometheus Alert Rules (เช่นแจ้งเตือน CPU > 90%, HA down)

สามารถแจ้งเพิ่มเติมได้เลยครับ ผมจัดให้ได้แบบเต็มระบบครับ ✅
