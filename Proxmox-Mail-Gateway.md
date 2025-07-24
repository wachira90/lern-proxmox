**Proxmox Mail Gateway** คือระบบ **Email Security Gateway** แบบโอเพ่นซอร์ส ที่ใช้สำหรับ **กรองและป้องกันอีเมลขาเข้าและขาออก** จาก **สแปม (Spam), มัลแวร์ (Malware), ฟิชชิ่ง (Phishing)** และภัยคุกคามอื่น ๆ ที่มากับอีเมล

---

### 🔧 คุณสมบัติหลักของ Proxmox Mail Gateway:

1. **Antispam และ Antivirus:**

   * ใช้ **SpamAssassin**, **ClamAV**, **Razor**, **Pyzor**, และ **DKIM/DMARC/SPF** เพื่อตรวจจับสแปมและไวรัส
   * มีระบบเรียนรู้ (Bayesian filtering) เพื่อตรวจจับสแปมอย่างแม่นยำ

2. **กรองนโยบาย (Policy-based Filtering):**

   * สามารถกำหนดกฎ (rules) ตามผู้ส่ง, ผู้รับ, ขนาด, เนื้อหา ฯลฯ ได้อย่างยืดหยุ่น

3. **รองรับการทำงานแบบคลัสเตอร์ (Cluster):**

   * รองรับการทำงานหลายโหนด (High Availability / Load Balancing)

4. **Web Interface ใช้งานง่าย:**

   * UI จัดการนโยบาย, ดู logs, และรายงานต่าง ๆ ได้ผ่านเว็บ

5. **Email Quarantine และ User Quarantine Web Interface:**

   * ผู้ใช้งานสามารถเข้าไปดูและปล่อยอีเมลที่ถูกกักไว้เองได้

6. **Relay Mode:**

   * ทำหน้าที่เป็น mail proxy: รับอีเมลจากภายนอก → ตรวจสอบ → ส่งเข้า mail server ภายใน (เช่น Microsoft Exchange, Zimbra, Postfix เป็นต้น)

---

### 🛡️ สถานการณ์ที่นิยมใช้:

* องค์กรที่ต้องการ **ปกป้อง Mail Server หลัก** จากอีเมลอันตราย
* ใช้กับระบบอีเมลภายในองค์กร เพื่อช่วยกรอง spam และ virus ก่อนถึง user
* ใช้แทน email security appliance เชิงพาณิชย์ เช่น Barracuda, FortiMail, หรือ Proofpoint

---

### 🔄 โครงสร้างการทำงานโดยทั่วไป:

```plaintext
[Internet] → [Proxmox Mail Gateway] → [Mail Server ภายใน เช่น Exchange/Postfix]
```

===

แน่นอนครับ ด้านล่างนี้คือ **ตัวอย่างการติดตั้งและตั้งค่าเบื้องต้นของ Proxmox Mail Gateway (PMG)** ครอบคลุมตั้งแต่การติดตั้ง OS, การตั้งค่า Mail Relay, ไปจนถึงการทดสอบการทำงานเบื้องต้น

---

## ✅ ขั้นตอนที่ 1: การติดตั้ง Proxmox Mail Gateway

### วิธีที่แนะนำ:

> ดาวน์โหลด **ISO จากเว็บไซต์ทางการ**:
> 🔗 [https://www.proxmox.com/en/downloads](https://www.proxmox.com/en/downloads)

### 1.1 ติดตั้งจาก ISO

* เบิร์นไฟล์ ISO ลง USB
* Boot จาก USB แล้วติดตั้งเหมือน Debian-based Linux ปกติ
* ตั้งค่าระบบพื้นฐาน: hostname, IP, timezone

---

## ✅ ขั้นตอนที่ 2: เข้าสู่ Web UI

หลังจากติดตั้งเสร็จ:

* เปิด browser → เข้าสู่ Web UI ผ่าน

  ```
  https://<your-pmg-ip>:8006
  ```
* เข้าด้วย root + รหัสผ่านที่ตั้งไว้ตอนติดตั้ง

---

## ✅ ขั้นตอนที่ 3: การตั้งค่า Mail Relay เบื้องต้น

### 3.1 ตั้งค่า Mail Domain และ Mail Relay

* ไปที่ **Configuration > Mail Proxy > Relay Domains**

  * เพิ่มโดเมนของคุณ (เช่น `example.com`)
* ไปที่ **Configuration > Mail Proxy > Transports**

  * เพิ่ม Mail Server ปลายทางที่คุณต้องการส่งเมลให้ภายในองค์กร
  * เช่น:

    ```
    Domain: example.com
    Target: 192.168.10.5 (Mail Server ภายใน)
    Port: 25
    ```

### 3.2 ตั้งค่า Mail Relay Network

* ไปที่ **Configuration > Mail Proxy > Networks**

  * เพิ่ม IP range ที่อนุญาตให้ส่งอีเมลผ่าน PMG เช่น:

    ```
    192.168.10.0/24
    ```

---

## ✅ ขั้นตอนที่ 4: ทดสอบรับส่งเมล

1. ทดสอบส่งเมลจากภายนอกเข้ามา:

   * Email → `user@example.com`
     → เข้า PMG → ตรวจสอบ Log ว่าถูกส่งไป Mail Server ภายในสำเร็จ

2. ทดสอบส่งเมลจากภายในออก:

   * Mail Server ภายในตั้งค่าให้ใช้ PMG เป็น Smart Host หรือ SMTP Relay
   * ตรวจสอบว่า mail ออกได้และถูกกรองตามนโยบาย

---

## ✅ ขั้นตอนที่ 5: ตั้งค่านโยบายกรองสแปม/ไวรัส

* ไปที่ **Configuration > Mail Filter > Who Objects**

  * สร้าง object เช่น กลุ่มผู้ส่ง/ผู้รับ

* ไปที่ **Mail Filter Rules**

  * สร้าง Rule เช่น:

    * IF: email มาจากภายนอก
    * THEN: ตรวจสอบ Spam + Virus
    * ACTION: Block หรือ Quarantine

---

## ✅ ตัวอย่างการตั้งค่าผ่าน CLI

```bash
# ดู log การรับส่งเมล
tail -f /var/log/mail.log

# ตรวจสอบสถานะบริการ
systemctl status pmgproxy
systemctl status postfix

# Restart mail gateway service
systemctl restart pmg-smtp-filter
```

---

## 🔐 แนะนำเพิ่มเติม

* ตั้งค่า SPF, DKIM, DMARC ใน DNS เพื่อเพิ่มความน่าเชื่อถือ
* ใช้ระบบ Quarantine เพื่อให้ผู้ใช้สามารถจัดการเมลต้องสงสัยได้เอง
* สำรองค่าคอนฟิกผ่าน `Backup` ใน Web UI หรือ `pmgbackup` CLI

---

ด้านล่างนี้คือ **ตัวอย่างไฟล์ configuration สำหรับ Proxmox Mail Gateway (PMG)** ที่ครอบคลุมการตั้งค่าเบื้องต้นในการรับ-ส่งเมล, Mail Relay, และนโยบายกรองอีเมล

---

## 📁 1. `/etc/pmg/pmg.conf` – การตั้งค่า PMG หลัก

```ini
hostname example-pmg.example.com
email admin@example.com
timezone Asia/Bangkok
```

---

## 📁 2. `/etc/postfix/main.cf` – การตั้งค่า Postfix (Mail Proxy)

PMG ใช้ Postfix เป็น SMTP engine ด้านล่าง

```ini
myhostname = example-pmg.example.com
mydomain = example.com
myorigin = $mydomain
inet_interfaces = all
inet_protocols = all

relay_domains = proxy:mysql:/etc/postfix/mysql-relay-domains.cf
transport_maps = proxy:mysql:/etc/postfix/mysql-transport.cf

mydestination = localhost
mynetworks = 127.0.0.0/8, 192.168.10.0/24
smtpd_recipient_restrictions =
    permit_mynetworks,
    reject_unauth_destination

# Spam/Virus Headers
receive_override_options = no_address_mappings
content_filter = smtp:[127.0.0.1]:10024
```

---

## 📁 3. `/etc/postfix/mysql-relay-domains.cf` – Relay Domains

```ini
user = pmg
password = your-db-password
hosts = 127.0.0.1
dbname = pmg
query = SELECT domain FROM relay_domains WHERE domain='%s'
```

PMG จะจัดการฐานข้อมูลนี้อัตโนมัติผ่าน Web UI → **Mail Proxy > Relay Domains**

---

## 📁 4. `/etc/postfix/mysql-transport.cf` – Mail Routing

```ini
user = pmg
password = your-db-password
hosts = 127.0.0.1
dbname = pmg
query = SELECT target FROM transport WHERE domain='%s'
```

ใช้กำหนดว่าจะ forward email ไปที่ Mail Server ภายในที่ใด

ตัวอย่างจาก Web UI:

| Domain      | Target          |
| ----------- | --------------- |
| example.com | 192.168.10.5:25 |

---

## 📁 5. `/etc/pmg/pmg-smtp-filter.conf` – ฟิลเตอร์ไวรัส/สแปม

```ini
clamav = yes
spamassassin = yes
sa_rule_score_overwrite = no

quarantine = yes
quarantine_unsupported_types = yes
quarantine_attachments = yes
```

---

## 📁 6. `/etc/pmg/pmg-template.conf` (ไม่จำเป็นต้องแก้ แต่สามารถปรับได้)

ใช้ปรับแต่งเทมเพลต Quarantine Report / Footer / Reject Message เป็นต้น

---

## ✅ ตัวอย่างการสร้าง Policy ผ่าน Web UI (Mail Filter)

### Rule: Block Executable Attachments

```plaintext
IF: attachment filetype is .exe OR .bat
THEN: Action = Block
```

หรือ

### Rule: Quarantine Spam

```plaintext
IF: Spam Score ≥ 5.0
THEN: Quarantine + Add header
```

---

## 🔧 คำสั่ง CLI ที่ช่วยในการ Debug/ตรวจสอบ

```bash
# ตรวจสอบอีเมลที่ถูกกรอง
journalctl -u pmg-smtp-filter -f

# ตรวจสอบ log postfix
tail -f /var/log/mail.log

# ทดสอบรับเมล (ใช้เครื่องอื่นส่งเข้า)
telnet <PMG_IP> 25
```


