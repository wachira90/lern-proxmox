# **ตัวอย่าง Terraform “แบบ production”** 

สำหรับ Proxmox ที่ใช้งานจริงได้ (เน้น: แยกไฟล์, ไม่ hardcode secret, ใช้ Cloud-Init, ทำเป็น module, รองรับหลาย VM)

> ตัวอย่างนี้ใช้ provider **bpg/proxmox** (ค่อนข้างนิยมและ feature ครบ) อ้างอิงจาก Terraform Registry และเอกสาร provider ที่แนะนำไม่ hardcode credential และรองรับ API token auth ([Terraform Registry][1])

---

## โครงสร้างโปรเจกต์ (แนะนำ)

```
proxmox-iac/
  ├── versions.tf
  ├── providers.tf
  ├── variables.tf
  ├── main.tf
  ├── outputs.tf
  ├── terraform.tfvars.example
  └── modules/
      └── vm-cloudinit/
          ├── main.tf
          ├── variables.tf
          └── outputs.tf
```

---

## 0) ตั้งค่า Secret แบบปลอดภัย (ไม่ใส่ในไฟล์)

แนะนำให้ export env vars (ตามแนวทางใน doc ของ bpg/proxmox และ registry) ([Terraform Registry][2])

```bash
export PROXMOX_VE_ENDPOINT="https://pve1.example.com:8006/"
export PROXMOX_VE_API_TOKEN="terraform@pve!prod=YOUR_TOKEN_SECRET"
# ถ้าใช้ CA/ใบ cert ที่ถูกต้องให้เปิด verify ตามปกติ (อย่าใช้ insecure ถ้า production)
```

> **ควรสร้าง user/token เฉพาะสำหรับ Terraform + RBAC แบบ least privilege** (ไม่ใช้ root)
> แนวคิดเดียวกันกับคำแนะนำในเอกสาร provider/บทความแนวปฏิบัติ ([Terraform Registry][3])

---

## 1) versions.tf

```hcl
terraform {
  required_version = ">= 1.6.0"

  required_providers {
    proxmox = {
      source  = "bpg/proxmox"
      version = "~> 0.89"
    }
  }
}
```

---

## 2) providers.tf

```hcl
provider "proxmox" {
  # ปล่อยให้ดึงจาก env vars: PROXMOX_VE_ENDPOINT, PROXMOX_VE_API_TOKEN
  # เอกสาร provider รองรับ API token auth และแนะนำไม่ hardcode credential :contentReference[oaicite:3]{index=3}
}
```

---

## 3) variables.tf (root)

```hcl
variable "target_node" { type = string }          # เช่น "pve1"
variable "template_vmid" { type = number }        # VMID ของ cloud-init template เช่น 9000
variable "vm_storage" { type = string }           # เช่น "local-lvm" หรือ "ceph-vm"
variable "bridge" { type = string, default = "vmbr0" }
variable "ssh_public_keys" { type = list(string) }

variable "vms" {
  description = "Map of VMs to create"
  type = map(object({
    vmid      = number
    name      = string
    cores     = number
    memory_mb = number
    disk_gb   = number
    vlan_tag  = optional(number)
    ip        = string         # เช่น "10.10.10.21/24"
    gw        = string         # เช่น "10.10.10.1"
    dns       = list(string)   # เช่น ["1.1.1.1","8.8.8.8"]
  }))
}
```

---

## 4) main.tf (root) — เรียกใช้ module

```hcl
module "vm" {
  for_each = var.vms
  source   = "./modules/vm-cloudinit"

  target_node     = var.target_node
  template_vmid   = var.template_vmid
  vm_storage      = var.vm_storage
  bridge          = var.bridge
  ssh_public_keys = var.ssh_public_keys

  vmid      = each.value.vmid
  name      = each.value.name
  cores     = each.value.cores
  memory_mb = each.value.memory_mb
  disk_gb   = each.value.disk_gb
  vlan_tag  = try(each.value.vlan_tag, null)

  ip  = each.value.ip
  gw  = each.value.gw
  dns = each.value.dns
}
```

---

## 5) modules/vm-cloudinit/main.tf (ตัว “production VM” จริง)

> resource ชื่อและ field อาจต่างกันเล็กน้อยตามเวอร์ชัน provider แต่ concept นี้คือ “clone template + cloud-init + agent + network/disk”
> (ดู argument ของ VM resource ใน registry ได้) ([Terraform Registry][1])

```hcl
resource "proxmox_virtual_environment_vm" "this" {
  node_name = var.target_node
  vm_id     = var.vmid
  name      = var.name

  # Clone จาก template VM ที่เตรียม cloud-init ไว้
  clone {
    vm_id = var.template_vmid
  }

  cpu {
    cores = var.cores
    type  = "host"
  }

  memory {
    dedicated = var.memory_mb
  }

  agent {
    enabled = true
  }

  disk {
    datastore_id = var.vm_storage
    interface    = "scsi0"
    size         = "${var.disk_gb}G"
    iothread     = true
  }

  network_device {
    bridge  = var.bridge
    model   = "virtio"
    vlan_id = var.vlan_tag
  }

  initialization {
    # cloud-init user
    user_account {
      username = "ubuntu"
      keys     = var.ssh_public_keys
    }

    ip_config {
      ipv4 {
        address = var.ip
        gateway = var.gw
      }
    }

    dns {
      servers = var.dns
    }
  }

  # ลด “drift” บางอย่างใน production (ปรับตามความจริง)
  lifecycle {
    ignore_changes = [
      initialization[0].user_account[0].keys,
    ]
  }

  timeouts {
    create = "30m"
    update = "30m"
  }
}
```

### modules/vm-cloudinit/variables.tf

```hcl
variable "target_node" { type = string }
variable "template_vmid" { type = number }
variable "vm_storage" { type = string }
variable "bridge" { type = string }
variable "ssh_public_keys" { type = list(string) }

variable "vmid" { type = number }
variable "name" { type = string }
variable "cores" { type = number }
variable "memory_mb" { type = number }
variable "disk_gb" { type = number }
variable "vlan_tag" { type = number, default = null }

variable "ip" { type = string }
variable "gw" { type = string }
variable "dns" { type = list(string) }
```

### modules/vm-cloudinit/outputs.tf

```hcl
output "vm_id" {
  value = proxmox_virtual_environment_vm.this.vm_id
}
```

---

## 6) terraform.tfvars.example

```hcl
target_node   = "pve1"
template_vmid = 9000
vm_storage    = "local-lvm"
bridge        = "vmbr0"

ssh_public_keys = [
  "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI... you@laptop"
]

vms = {
  web01 = {
    vmid      = 201
    name      = "web01"
    cores     = 2
    memory_mb = 2048
    disk_gb   = 30
    vlan_tag  = 10
    ip        = "10.10.10.21/24"
    gw        = "10.10.10.1"
    dns       = ["1.1.1.1","8.8.8.8"]
  }
  web02 = {
    vmid      = 202
    name      = "web02"
    cores     = 2
    memory_mb = 2048
    disk_gb   = 30
    vlan_tag  = 10
    ip        = "10.10.10.22/24"
    gw        = "10.10.10.1"
    dns       = ["1.1.1.1","8.8.8.8"]
  }
}
```

---

## Checklist แบบ production (สำคัญมาก)

* ใช้ **API Token** และ **ไม่ hardcode secret** ใน repo ([GitHub][4])
* สร้าง user/role/token แบบ **least privilege** (ไม่ใช้ root, จำกัด path/permissions) ([OpenTofu][5])
* ทำ **template cloud-init** ไว้ก่อน (เช่น Ubuntu cloud image + qemu-guest-agent)
* ใช้ **remote backend** (เช่น S3/MinIO/Terraform Cloud) เพื่อ state locking ในทีม
* แยก module / variables / tfvars ตาม environment (dev/stage/prod)

---

ถ้าบอกผมเพิ่มอีกนิดว่า:

1. OS template เป็นอะไร (Ubuntu/Debian/RHEL)
2. ใช้ DHCP หรือ Static IP
3. Storage เป็น local-lvm / ceph / zfs
   ผมจะปรับตัวอย่างให้ “ตรง infra ของคุณเป๊ะ ๆ” (รวม cloud-init snippets, multiple NIC, extra disks, tag/notes, และ output inventory สำหรับ Ansible ได้)

[1]: https://registry.terraform.io/providers/bpg/proxmox/latest/docs/resources/virtual_environment_vm?utm_source=chatgpt.com "proxmox_virtual_environment_vm | Resources | bpg/proxmox | Terraform ..."
[2]: https://registry.terraform.io/providers/bpg/proxmox/0.89.1/docs?utm_source=chatgpt.com "Docs overview | bpg/proxmox | Terraform | Terraform Registry"
[3]: https://registry.terraform.io/providers/Telmate/proxmox/latest/docs?utm_source=chatgpt.com "Docs overview | Telmate/proxmox - Terraform Registry"
[4]: https://github.com/bpg/terraform-provider-proxmox/blob/main/docs/index.md?utm_source=chatgpt.com "terraform-provider-proxmox/docs/index.md at main · bpg ... - GitHub"
[5]: https://library.tf/providers/Telmate/proxmox/latest?utm_source=chatgpt.com "Telmate/proxmox | Providers | OpenTofu and Terraform Registry"
