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
