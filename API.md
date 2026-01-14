# *Proxmox VE - REST API*

✅ **Proxmox VE มี REST API สำหรับสร้างและจัดการ VM/Container ได้โดยตรง**

### สรุปสั้น ๆ

* ✔️ สร้าง / ลบ / แก้ไข **VM (QEMU/KVM)** และ **Container (LXC)**
* ✔️ จัดการ disk, network, snapshot, start/stop
* ✔️ ใช้งานผ่าน **HTTPS REST API**
* ✔️ รองรับ automation tools เช่น **Terraform, Ansible, Packer**

---

## รายละเอียด Proxmox VE API

* เรียกว่า **Proxmox VE REST API**
* เอกสาร API (built-in):

  ```
  https://<proxmox-host>:8006/api2/json/
  ```

  หรือดูแบบ interactive:

  ```
  https://<proxmox-host>:8006/api-docs/
  ```

### Authentication

รองรับหลายแบบ:

1. **Ticket (username/password)** – เหมาะกับ manual / script สั้น ๆ
2. **API Token (แนะนำสำหรับ automation)**

   * ปลอดภัยกว่า
   * ใช้ร่วมกับ RBAC ได้ดี

---

## ตัวอย่าง: สร้าง VM ด้วย API (QEMU)

### 1. สร้าง VM เปล่า

```bash
curl -k -X POST https://pve:8006/api2/json/nodes/pve/qemu \
  -H "Authorization: PVEAPIToken=USER@pve!TOKENID=TOKENVALUE" \
  -d vmid=100 \
  -d name=test-vm \
  -d memory=2048 \
  -d cores=2 \
  -d net0=virtio,bridge=vmbr0
```

### 2. เพิ่ม disk ให้ VM

```bash
curl -k -X POST https://pve:8006/api2/json/nodes/pve/qemu/100/config \
  -H "Authorization: PVEAPIToken=..." \
  -d scsi0=local-lvm:20
```

### 3. Start VM

```bash
curl -k -X POST https://pve:8006/api2/json/nodes/pve/qemu/100/status/start \
  -H "Authorization: PVEAPIToken=..."
```

---

## Automation ที่นิยมใช้กับ Proxmox

ถ้าคุณทำสาย DevOps / Infra (เหมาะกับคุณมาก 😄)

* **Terraform**

  * provider: `Telmate/proxmox`
* **Ansible**

  * module: `community.general.proxmox_kvm`
* **Packer**

  * สร้าง template VM (Golden Image)

---

## เหมาะกับ use case ไหนบ้าง

* Auto-provision VM
* CI/CD infra
* Lab / Homelab
* Private cloud แบบ lightweight

ถ้าต้องการ:

* ตัวอย่าง **Terraform**
* ตัวอย่าง **Ansible playbook**
* หรือทำ **Cloud-Init + Template**

บอกได้เลยครับ เดี๋ยวจัดให้ตรง use case 👍
