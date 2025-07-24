# **Step-by-Step Walkthrough แบบละเอียด** 

การตั้งค่า **Proxmox VE Cluster + HA (High Availability)** เหมาะสำหรับใช้งานใน Production เบื้องต้น พร้อมคำอธิบายแต่ละขั้นตอน:

---

# 🧭 คู่มือการตั้งค่า Proxmox VE Cluster + HA (Step-by-Step)

## 🧱 ความต้องการเบื้องต้น (Infrastructure Requirements)

* ✅ Proxmox VE เวอร์ชันล่าสุดติดตั้งบน **อย่างน้อย 2-3 Node**
* ✅ Network ที่เชื่อมต่อกันได้ (แนะนำให้แยกเครือข่าย Cluster, Storage, และ Public)
* ✅ Shared Storage เช่น NFS, iSCSI, Ceph, ZFS over iSCSI (จำเป็นสำหรับ Live Migration และ HA)
* ✅ Time sync (ใช้ `chrony` หรือ `ntp`)

---

## 🔧 STEP 1: เตรียม Node แต่ละเครื่อง

### 1.1 ตั้งชื่อ hostname ที่ไม่ซ้ำกัน

```bash
hostnamectl set-hostname pve1     # ทำบน node แรก
hostnamectl set-hostname pve2     # ทำบน node ที่สอง
```

### 1.2 แก้ไขไฟล์ `/etc/hosts`

ให้แน่ใจว่าแต่ละ node รู้จักกัน:

```bash
192.168.1.101 pve1
192.168.1.102 pve2
```

> แนะนำให้ใช้ IP Address ของ interface ที่ใช้เชื่อมต่อ Cluster (เช่น LAN หรือ dedicated interface)

---

## 🔧 STEP 2: สร้าง Cluster บน Node แรก (pve1)

```bash
pvecm create my-cluster
```

* `my-cluster` คือชื่อ Cluster ที่คุณตั้งเอง
* เมื่อสร้างเสร็จ ระบบจะเริ่มบริการ `corosync` อัตโนมัติ

ตรวจสอบสถานะ:

```bash
pvecm status
```

---

## 🔧 STEP 3: Join Node อื่นเข้ามาใน Cluster

### 3.1 บน node ใหม่ (pve2):

```bash
pvecm add 192.168.1.101
```

> ใช้ IP หรือ hostname ของ node แรกที่สร้าง Cluster

### 3.2 ตรวจสอบจาก node แรกว่า node เข้ามาร่วมแล้ว

```bash
pvecm nodes
```

> คุณควรเห็น Node 1 และ Node 2 อยู่ใน Cluster

---

## 🗄️ STEP 4: เพิ่ม Shared Storage (NFS, iSCSI, Ceph)

**ตัวอย่างการเพิ่ม NFS (ผ่าน Web UI):**

1. ไปที่ `Datacenter` → `Storage`
2. Click “Add” → เลือก `NFS`
3. ใส่:

   * ID: ชื่อ storage (เช่น `shared-nfs`)
   * Server: IP ของ NFS Server
   * Export: Path ที่ export เช่น `/mnt/pve/nfs`
   * Content: เลือกให้รองรับ VM, ISO, Backup ตามต้องการ
4. Click “Add”

---

## 📦 STEP 5: สร้าง VM และเก็บไว้ใน Shared Storage

1. ไปที่ `Create VM`
2. ตั้งค่าทั่วไป (VM ID, Name)
3. เลือก ISO และ Disk → ให้แน่ใจว่า Disk ไปอยู่บน Shared Storage (`shared-nfs` หรืออื่น ๆ)
4. Finish

> **ต้องอยู่บน shared storage เท่านั้นถึงจะย้าย VM ได้!**

---

## 🔁 STEP 6: เปิดใช้งาน HA (High Availability)

### 6.1 เปิด “HA Manager” บนทุก Node

ตรวจสอบว่า HA Manager ทำงาน:

```bash
systemctl status pve-ha-lrm
```

### 6.2 เปิดใช้งาน HA ผ่าน Web UI

1. ไปที่ `Datacenter` → `HA`
2. Click “Add”
3. เลือก VM ที่ต้องการ → ตั้งค่า:

   * **Group**: กำหนดกลุ่ม Node ที่สามารถรัน VM นี้ได้
   * **Recovery Policy**:

     * `restart` – หาก VM crash จะ restart ที่เดิม
     * `migrate` – หาก Node ตาย VM จะย้ายไป node อื่น
4. Click “Add”

---

## 🧪 STEP 7: ทดสอบ HA

1. สร้าง VM ที่อยู่บน shared storage และเปิด HA
2. ปิด node ที่รัน VM อยู่ (shutdown หรือ unplug network)
3. ดูว่า VM ถูกย้าย (migrate) ไป Node อื่นโดยอัตโนมัติหรือไม่

---

## 📋 คำสั่ง CLI ที่เกี่ยวข้อง

* ดูสถานะ cluster:

  ```bash
  pvecm status
  ```
* ดู Node ใน cluster:

  ```bash
  pvecm nodes
  ```
* ดู VM HA status:

  ```bash
  ha-manager status
  ```

---

## 🛠️ ข้อควรระวัง

* อย่าใช้ local disk ใน VM ที่ต้องการ HA
* อย่าลืม sync เวลาในทุก node (`chrony` หรือ `ntp`)
* ควรมี 3 Node ขึ้นไปเพื่อ quorum ที่เสถียร (2 node จะไม่มี quorum หาก node ใด node หนึ่งตาย)
* ถ้าใช้ Ceph ต้องมี disk redundancy ด้วย (เช่น 3 OSD ขึ้นไป)

---

## 🎯 บทสรุป

Proxmox VE ทำให้การตั้งค่า Cluster และ HA ง่ายขึ้นมาก โดยไม่ต้อง config `pacemaker` เอง:

| สิ่งที่ได้                | ความง่าย         |
| ------------------------- | ---------------- |
| Cluster Communication     | ✅ ง่าย           |
| HA Manager                | ✅ Built-in       |
| Live Migration            | ✅ เพียงคลิกเดียว |
| Storage แบบ Cluster-aware | ✅ รองรับหลายชนิด |
| UI จัดการ HA              | ✅ Web UI ครบถ้วน |

---

=== 

แน่นอนครับ! ด้านล่างนี้คือ **Sample Config และตัวอย่างใช้งาน** สำหรับ:

1. ✅ **HA Group Script บน Proxmox VE**
2. ✅ **Ceph Integration (พร้อม config storage pool)**
3. ✅ **Log การทำ Failover และการวิเคราะห์ HA Event**

---

## 🧩 1. **HA Group Script (Proxmox VE)**

**HA Group** ใช้กำหนดว่า VM ใดสามารถรันอยู่บน Node ไหนได้บ้าง รวมถึง Priority ด้วย

### 📌 สร้าง HA Group (ผ่าน CLI)

```bash
ha-manager groupadd hagrp1 --nodes pve1,pve2 --nofailback 1
```

* `hagrp1`: ชื่อกลุ่ม
* `--nodes`: รายชื่อ node ที่อยู่ใน group
* `--nofailback 1`: เมื่อ node ฟื้น VM ไม่กลับไป node เดิม (optional)

### 📌 เพิ่ม VM เข้า HA + ผูกกับ Group

```bash
ha-manager add vm:100 --group hagrp1 --state started
```

* `vm:100`: ID ของ VM
* `--group hagrp1`: ให้ VM นี้อยู่ในกลุ่ม HA group นี้
* `--state started`: บูทอัตโนมัติใน Cluster

### 📋 ตรวจสอบ:

```bash
ha-manager config
ha-manager status
```

---

## 🪵 2. **Ceph Integration (Storage แบบ Hyper-Converged)**

Proxmox VE รองรับ **Ceph** แบบ Native โดยใช้ Web UI หรือ CLI ตั้งค่าได้ง่ายมาก

### 📌 ขั้นตอนสั้นๆ:

#### A. ติดตั้ง Ceph บน Node ทั้งหมด

```bash
pveceph install
```

#### B. สร้าง Cluster Ceph (node แรก)

```bash
pveceph init --network 10.10.10.0/24
```

#### C. สร้าง Monitor + Manager

```bash
pveceph mon create
pveceph mgr create
```

#### D. สร้าง OSD (บน disk เปล่า)

```bash
pveceph osd create /dev/sdb
```

(ทำซ้ำบนทุก node ที่มี disk ให้ Ceph)

#### E. สร้าง Ceph Pool

```bash
pveceph pool create cephpool --size 3 --min_size 2
```

#### F. เพิ่ม Ceph Pool เป็น Storage

```bash
pvesh create /storage \
    -storage cephstore \
    -type rbd \
    -monhost 10.10.10.1 \
    -pool cephpool \
    -content images,rootdir \
    -krbd 1
```

> หรือทำผ่าน UI: `Datacenter > Storage > Add > RBD (Ceph)`

---

## 📊 3. **Log การทำ Failover และการวิเคราะห์ HA Events**

Proxmox จะบันทึก HA events ใน log หลายจุด:

### 📌 ตรวจสอบสถานะ HA แบบ real-time:

```bash
ha-manager status
```

### 📌 Log หลักของ HA:

```bash
journalctl -u pve-ha-lrm
journalctl -u pve-ha-crm
```

* `pve-ha-lrm`: Local Resource Manager – ติดตาม VM บนแต่ละ node
* `pve-ha-crm`: Cluster Resource Manager – รับผิดชอบการตัดสินใจ failover

### 📌 ตัวอย่าง Log Failover:

```text
Jul 24 14:03:21 pve2 pve-ha-lrm[1234]: vm:100: state changed from started to error
Jul 24 14:03:22 pve1 pve-ha-crm[4567]: info: requesting vm:100 to recover on node pve1
Jul 24 14:03:25 pve1 pve-ha-lrm[7890]: vm:100 started successfully
```

### 📌 วิเคราะห์เหตุการณ์:

1. VM เปลี่ยน state → error (บน node เดิม)
2. CRM เรียก failover ไป node อื่น
3. LRM บน node ใหม่ start VM สำเร็จ

### 📦 วิเคราะห์ HA แบบระยะยาว:

คุณสามารถเขียน script เช็ค failover events ด้วย `grep`:

```bash
journalctl -u pve-ha-lrm | grep 'state changed from'
```

หรือส่ง log เข้าระบบ ELK / Loki / Grafana สำหรับ visualize

---

## 🧠 Tip เสริม

| เป้าหมาย              | คำแนะนำ                                                                     |
| --------------------- | --------------------------------------------------------------------------- |
| **การทดสอบ Failover** | ใช้ `shutdown -h now` หรือ `iptables -A INPUT -j DROP` ตัด network ชั่วคราว |
| **Ceph Redundancy**   | ใช้ `size=3`, `min_size=2` เพื่อ redundancy 1 node                          |
| **HA ไม่ทำงาน?**      | ตรวจสอบว่า VM อยู่บน **shared storage** หรือ Ceph เท่านั้น                  |

---

ถ้าคุณต้องการ:

* ตัวอย่าง `ha-manager config` export
* ตัวอย่างการทำ Live Migration อัตโนมัติเมื่อ CPU/Memory สูง
* Monitoring integration กับ Zabbix, Prometheus

แจ้งได้เลยครับ จะจัดตัวอย่างให้เพิ่มเติมครับ ✅

