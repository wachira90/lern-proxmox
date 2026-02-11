# การตั้งค่า **Clustering และ High Availability (HA)** 

จะแตกต่างกันระหว่าง **Proxmox VE** และ **Pure Linux KVM** ค่อนข้างมาก ทั้งในเรื่องของขั้นตอน ความซับซ้อน และเครื่องมือที่ใช้ ด้านล่างคือรายละเอียดแบบแยกตามแพลตฟอร์ม:

---

## 🖥️ **1. Clustering & HA บน Proxmox VE**

Proxmox มีระบบ Clustering และ HA **แบบ built-in** ทำให้การตั้งค่าง่ายและรวดเร็ว

### ✅ สิ่งที่ต้องเตรียม

* Server 2–3 เครื่อง ที่ติดตั้ง Proxmox VE
* Network ระหว่าง Node ต้องเชื่อมต่อกันได้ (ควรมีเครือข่ายเฉพาะสำหรับ cluster traffic)
* Shared Storage เช่น NFS, iSCSI, Ceph (เพื่อใช้ Live Migration และ HA)

### 🔧 ขั้นตอนการทำ Cluster + HA

#### 1. ติดตั้ง Proxmox VE บนทุก Node

```bash
apt update && apt full-upgrade
```

#### 2. ตั้งชื่อ hostname และ IP ให้ไม่ซ้ำกัน

#### 3. บน Node แรก (เช่น `pve1`) สร้าง Cluster:

```bash
pvecm create my-cluster
```

#### 4. บน Node ถัดไป (เช่น `pve2`, `pve3`) เข้าร่วม Cluster:

```bash
pvecm add <IP-ของ-nodeแรก>
```

#### 5. ตรวจสอบ Cluster:

```bash
pvecm status
```

#### 6. เพิ่ม Shared Storage ผ่าน Web UI:

* ไปที่ `Datacenter > Storage` > Add NFS/iSCSI/Ceph/ZFS

#### 7. เปิด HA Manager:

* ไปที่ `Datacenter > HA`
* กำหนดว่า VM ไหนต้องการ HA
* ตั้งค่ากฎ Failover/Restart

#### 8. ทดสอบ Failover:

* ปิด Node หนึ่ง แล้วดูว่า VM ถูกย้ายไป Node อื่นหรือไม่

> 💡 Proxmox ใช้ `corosync` เป็น backend สำหรับ clustering และใช้ `ha-manager` เป็น daemon สำหรับ HA

---

## 🧱 **2. Clustering & HA บน Linux KVM (libvirt/pacemaker)**

สำหรับ pure Linux KVM ไม่มีระบบ clustering/HA แบบ built-in จำเป็นต้องใช้ **หลายเครื่องมือผสมกัน** เช่น:

* `libvirt` / `virsh` สำหรับจัดการ VM
* `corosync` + `pacemaker` สำหรับ cluster communication และ resource management
* `DRBD` หรือ shared storage (NFS/iSCSI/GlusterFS) สำหรับเก็บ disk image ของ VM
* `crmsh` หรือ `pcs` สำหรับจัดการ pacemaker

### ✅ สิ่งที่ต้องเตรียม

* Server 2–3 เครื่อง ติดตั้ง Linux (เช่น CentOS, RHEL, Debian)
* ติดตั้ง `libvirt`, `qemu`, `pacemaker`, `corosync`, `pcs`
* Network connectivity ระหว่าง node
* Shared storage หรือ replicated storage

### 🔧 ขั้นตอน (โดยสรุป)

#### 1. ติดตั้ง Package ที่จำเป็น:

```bash
apt install qemu-kvm libvirt-daemon corosync pacemaker pcs
```

#### 2. ตั้ง hostname และ `/etc/hosts` ให้สามารถ resolve ได้ทุก node

#### 3. กำหนด corosync config (`/etc/corosync/corosync.conf`) และ start cluster:

```bash
pcs cluster auth node1 node2
pcs cluster setup --name mycluster node1 node2
pcs cluster start --all
```

#### 4. สร้าง VM ด้วย `virsh` หรือ `virt-manager`

#### 5. เพิ่ม VM เข้าระบบ pacemaker:

```bash
pcs resource create myvm VirtualDomain \
  hypervisor="qemu:///system" \
  config="/etc/libvirt/qemu/myvm.xml" \
  migration_transport="ssh" \
  op start timeout=120s \
  op stop timeout=120s \
  op monitor interval=30s
```

#### 6. กำหนด resource constraints และ failover rules:

```bash
pcs resource location myvm prefers node1
pcs constraint order start mydrbd then myvm
```

#### 7. ทดสอบ Failover:

* ปิด node ที่รัน VM แล้วดูว่า pacemaker จะย้าย VM ไป node อื่นหรือไม่

> 💡 การจัดการ HA ใน KVM ต้องใช้ CLI เยอะ และต้องมีความรู้ด้าน cluster มากกว่าการใช้ Proxmox

---

## 📌 สรุปเปรียบเทียบแบบรวบรัด

| ด้าน             | Proxmox VE           | Linux KVM + Pacemaker        |
| ---------------- | -------------------- | ---------------------------- |
| ความง่าย         | ✅ สูง (Web UI)       | ❌ ซับซ้อน ต้อง CLI           |
| เวลาในการตั้งค่า | เร็วมาก (15–30 นาที) | นาน (1–2 ชั่วโมงหรือมากกว่า) |
| ความยืดหยุ่น     | กลาง                 | สูง                          |
| ความเสถียร       | สูง                  | สูง (หาก config ดี)          |
| เหมาะกับ         | Admin ทุกระดับ       | Admin เชี่ยวชาญ Linux, HA    |

---

หากคุณต้องการ walkthrough แบบละเอียดสำหรับ **ระบบใดระบบหนึ่ง** เช่น Proxmox VE หรือ Pacemaker + KVM ผมสามารถจัดทำคู่มือแบบ step-by-step เพิ่มเติมได้ครับ ✅

# การทำ HA ใน proxmox 

การทำ **HA (High Availability) ใน Proxmox VE** คือการทำให้ VM หรือ Container สามารถย้ายไปทำงานบน node อื่นได้อัตโนมัติเมื่อ node หลักล่ม (node failure) เพื่อลด downtime

ผมจะสรุปแบบใช้งานจริงสำหรับสาย Infra/DevOps ให้เลย 👇

---

# 🔹 1. Concept ของ Proxmox HA

Proxmox HA ใช้ component หลัก ๆ คือ:

* **Cluster (pve-cluster / corosync)** → ใช้ quorum ตัดสินว่า node ไหนยังอยู่
* **HA Manager (pve-ha-crm, pve-ha-lrm)** → คุม resource (VM/CT)
* **Shared Storage** → สำคัญมาก! ทุก node ต้องเข้าถึง disk เดียวกันได้

เมื่อ node ตาย:

* HA Manager จะ detect
* VM จะถูก start บน node อื่นอัตโนมัติ

---

# 🔹 2. Requirements สำคัญ

## ✅ 1) ต้องมี Proxmox Cluster อย่างน้อย 3 node (แนะนำ)

> 2 node ทำได้ แต่ต้องมี QDevice เพิ่ม ไม่งั้น quorum จะมีปัญหา

ตรวจสอบ cluster:

```bash
pvecm status
```

---

## ✅ 2) ต้องมี Shared Storage

HA จะทำงานไม่ได้ถ้า disk เป็น local storage

ตัวเลือกยอดนิยม:

* Ceph (ดีที่สุดสำหรับ production)
* NFS
* iSCSI + LVM
* ZFS over iSCSI

เช็ค storage:

```bash
pvesm status
```

---

# 🔹 3. ขั้นตอนทำ HA

## Step 1️⃣ สร้าง Cluster

บน node แรก:

```bash
pvecm create my-cluster
```

บน node อื่น:

```bash
pvecm add <IP-node1>
```

---

## Step 2️⃣ ตรวจสอบ Quorum

```bash
pvecm status
```

ต้องเห็น:

```
Quorate: Yes
```

---

## Step 3️⃣ ตั้งค่า Shared Storage

ตัวอย่างเพิ่ม NFS:

Datacenter → Storage → Add → NFS

หรือ CLI:

```bash
pvesm add nfs nfs-storage \
  --server 10.0.0.10 \
  --export /data/proxmox \
  --content images
```

---

## Step 4️⃣ ย้าย VM ไป Shared Storage

```bash
qm move_disk 100 scsi0 nfs-storage
```

---

## Step 5️⃣ เปิด HA ให้ VM

### ผ่าน GUI:

Datacenter → HA → Add
เลือก VM → ตั้งค่า state = started

### ผ่าน CLI:

```bash
ha-manager add vm:100
```

เช็ค status:

```bash
ha-manager status
```

---

# 🔹 4. ทดสอบ HA

Test ง่าย ๆ:

```bash
systemctl stop pve-cluster
```

หรือ simulate node down (ปิดเครื่องเลย)

ดูว่า VM ถูก start บน node อื่นหรือไม่:

```bash
ha-manager status
```

---

# 🔹 5. HA Policy & Behavior

ค่า default:

* restart max 1 ครั้ง
* ถ้า fail หลายครั้งจะ freeze

สามารถตั้ง group ได้:

```bash
ha-manager groupadd web-group --nodes node1,node2
```

เพิ่ม VM เข้า group:

```bash
ha-manager add vm:100 --group web-group
```

---

# 🔹 6. Production Best Practice (สำคัญ)

### 🔥 1) ใช้ Ceph ถ้าเป็น Production จริง

* ไม่มี single point of failure
* performance ดีกว่า NFS

### 🔥 2) แยก Network

* Corosync network แยก VLAN
* Storage network แยก
* VM traffic แยก

### 🔥 3) อย่าใช้ Local Disk กับ HA

ถ้า VM อยู่ local disk แล้วเปิด HA → VM จะ start ไม่ได้

---

# 🔹 7. ปัญหาที่พบบ่อย

### ❌ No Quorum

แก้โดยเพิ่ม node หรือ QDevice

### ❌ HA Service stuck

เช็ค:

```bash
journalctl -u pve-ha-crm
journalctl -u pve-ha-lrm
```

### ❌ VM ไม่ย้าย

90% เพราะใช้ local storage

---

# 🔹 8. Architecture ตัวอย่าง Production

```
        +-------------+
        |  LoadBalancer |
        +------+------+
               |
  --------------------------------
  |              |               |
+-----+        +-----+         +-----+
| PVE1|        | PVE2|         | PVE3|
+--+--+        +--+--+         +--+--+
   |              |               |
   +------ Ceph Cluster -----------+
```

---

# 🔹 สรุปแบบสั้น

| สิ่งที่ต้องมี  | จำเป็น   |
| -------------- | -------- |
| Cluster        | ✅        |
| Quorum         | ✅        |
| Shared Storage | ✅        |
| HA Manager     | Built-in |

---

ถ้า Ar-Ty ต้องการ ผมสามารถทำ:

* 🔥 Lab topology สำหรับ 2 node + QDevice
* 🔥 Best practice สำหรับ Enterprise
* 🔥 HA + Ceph design แบบ production
* 🔥 เปรียบเทียบ Proxmox HA vs VMware HA
* 🔥 Script auto deploy cluster

อยากได้ระดับไหน? Lab / Production / Enterprise architecture?



