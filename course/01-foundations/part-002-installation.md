# การติดตั้ง PostgreSQL บน Windows/macOS/Linux และผ่าน Docker

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับพื้นฐาน | Part 002

---

## เป้าหมายการเรียนรู้

เมื่อเรียนจบ Part นี้ ผู้เรียนจะสามารถ:

1. เลือกเวอร์ชัน PostgreSQL ที่เหมาะสมกับงาน และเข้าใจข้อกำหนดระบบ (system requirements) ก่อนติดตั้ง
2. ติดตั้ง PostgreSQL บน **Windows** ด้วย EnterpriseDB installer ได้อย่างถูกต้องและปลอดภัย
3. ติดตั้ง PostgreSQL บน **macOS** ด้วยทั้ง Homebrew และ Postgres.app พร้อมเข้าใจข้อแตกต่าง
4. ติดตั้ง PostgreSQL บน **Ubuntu/Debian** ผ่าน apt และ PostgreSQL APT repository (PGDG) เพื่อให้ได้เวอร์ชันล่าสุด
5. ติดตั้ง PostgreSQL บน **RHEL/CentOS/Fedora** ผ่าน dnf/yum และ PGDG repository
6. รัน PostgreSQL ผ่าน **Docker** และ **Docker Compose** พร้อมตั้งค่า volume, environment variables และ healthcheck อย่างถูกต้อง
7. เข้าใจขั้นตอน `initdb` และควบคุม service ด้วย `systemctl`, `pg_ctl`, และ `brew services`
8. รู้ตำแหน่งไฟล์สำคัญ (`postgresql.conf`, `pg_hba.conf`, data directory) ในแต่ละระบบปฏิบัติการ
9. เชื่อมต่อ PostgreSQL ครั้งแรกด้วย `psql` และแก้ปัญหาการเชื่อมต่อที่พบบ่อยได้ด้วยตนเอง
10. เข้าใจแนวคิดการอัปเกรดเวอร์ชัน (`pg_upgrade`) และขั้นตอนการถอนการติดตั้งอย่างปลอดภัย

---

## Step 11: เตรียมความพร้อมก่อนติดตั้ง — เลือกเวอร์ชัน PostgreSQL และข้อกำหนดระบบ

### 11.1 ทำไมการเลือกเวอร์ชันจึงสำคัญ

PostgreSQL มีการออก major version ใหม่ทุกปี (ประมาณเดือนกันยายน–ตุลาคม) และแต่ละ major version จะได้รับการซัพพอร์ต (security + bug fix) เป็นเวลา **5 ปี** นับจากวันเปิดตัว ณ วันที่เขียนหลักสูตรนี้ (กันยายน 2026) เวอร์ชันที่ควรพิจารณาคือ:

| เวอร์ชัน | สถานะ | คำแนะนำ |
|---|---|---|
| PostgreSQL 17 | Stable, ซัพพอร์ตระยะยาว | **แนะนำสำหรับโปรเจกต์ใหม่** — มี performance improvement ด้าน vacuum, incremental backup |
| PostgreSQL 16 | Stable | ตัวเลือกที่มั่นคง ใช้งานจริงในโปรดักชันจำนวนมากแล้ว เหมาะกับองค์กรที่ต้องการความเสถียรสูงสุด |
| PostgreSQL 15 หรือต่ำกว่า | ใกล้หมดซัพพอร์ตหรือหมดแล้ว | ใช้เฉพาะกรณีต้อง maintain ระบบเดิม (legacy) เท่านั้น |

> **หลักการเลือก**: ถ้าเริ่มโปรเจกต์ใหม่และไม่มีข้อจำกัดเรื่อง extension/driver compatibility ให้เลือก **เวอร์ชันล่าสุด (17)** เสมอ เพราะจะได้ระยะเวลาซัพพอร์ตยาวที่สุดและฟีเจอร์ใหม่ที่ช่วยเรื่อง performance และ observability

ตลอดหลักสูตรนี้ เราจะใช้ **PostgreSQL 17** เป็นหลัก แต่คำสั่งเกือบทั้งหมดใช้ได้กับ PostgreSQL 16 เช่นกัน (แค่เปลี่ยนเลขเวอร์ชันในคำสั่ง)

### 11.2 ข้อกำหนดระบบ (System Requirements)

**Hardware ขั้นต่ำสำหรับการเรียนรู้/พัฒนา (development):**

| รายการ | ขั้นต่ำ | แนะนำสำหรับเรียน |
|---|---|---|
| CPU | 1 core | 2+ core |
| RAM | 512 MB | 4 GB ขึ้นไป |
| Disk | 100 MB (ตัวโปรแกรม) + พื้นที่สำหรับข้อมูล | SSD อย่างน้อย 10 GB ว่าง |
| OS | Windows 10/11 (64-bit), macOS 12+, Linux kernel 5.x+ | เวอร์ชันล่าสุดของแต่ละ OS |

**Software ที่ควรมีก่อนเริ่ม:**

- สิทธิ์ผู้ดูแลระบบ (Administrator บน Windows, `sudo` บน Linux/macOS)
- Terminal / PowerShell / Command Prompt ที่ใช้งานได้
- (สำหรับ Docker) Docker Desktop หรือ Docker Engine เวอร์ชัน 24+ และ Docker Compose v2

### 11.3 แนวทางการเลือกวิธีติดตั้งตามสถานการณ์

```
┌─────────────────────────────────────────────────────────────┐
│  คุณกำลังจะ...                        │  วิธีติดตั้งที่แนะนำ      │
├─────────────────────────────────────────────────────────────┤
│  เรียนรู้/ทดลอง บน Windows            │  EnterpriseDB installer │
│  พัฒนาแอปบน macOS                      │  Homebrew หรือ Postgres.app │
│  รัน production server บน Linux        │  APT/DNF + PGDG repo     │
│  ต้องการ environment ที่ทำซ้ำได้       │  Docker / Docker Compose │
│  ทำงานในทีมที่ใช้ container อยู่แล้ว   │  Docker Compose          │
└─────────────────────────────────────────────────────────────┘
```

ในบทนี้เราจะติดตั้งทีละวิธีอย่างละเอียด ผู้เรียนสามารถเลือกอ่านเฉพาะ OS ของตนเอง หรืออ่านทั้งหมดเพื่อความเข้าใจภาพรวม (แนะนำให้อ่านส่วน Docker ด้วยเสมอ เพราะเป็นทักษะที่ใช้บ่อยมากในการทำงานจริง)

---

## Step 12: ติดตั้งบน Windows ด้วย EnterpriseDB Installer

EnterpriseDB (EDB) เป็นผู้จัดทำ installer อย่างเป็นทางการสำหรับ Windows ที่ได้รับการรับรองจากทีม PostgreSQL

### 12.1 ดาวน์โหลด Installer

1. เปิดเบราว์เซอร์ไปที่ `https://www.postgresql.org/download/windows/`
2. คลิก **"Download the installer"** ซึ่งจะพาไปยังหน้าของ EnterpriseDB
3. เลือกเวอร์ชัน **PostgreSQL 17.x** และสถาปัตยกรรม **Windows x86-64**
4. ดาวน์โหลดไฟล์ เช่น `postgresql-17.x-1-windows-x64.exe`

### 12.2 ขั้นตอนการติดตั้งทีละขั้น

**ขั้นที่ 1 — เริ่มการติดตั้ง**

คลิกขวาที่ไฟล์ installer แล้วเลือก **"Run as administrator"** จากนั้นคลิก **Next** ที่หน้าจอต้อนรับ

**ขั้นที่ 2 — เลือก Installation Directory**

ค่าเริ่มต้นคือ:
```
C:\Program Files\PostgreSQL\17
```
แนะนำให้ใช้ค่า default เว้นแต่มีเหตุผลเฉพาะ (เช่น disk C: มีพื้นที่จำกัด) เพื่อให้ path สอดคล้องกับเอกสารและเครื่องมืออื่น ๆ

**ขั้นที่ 3 — เลือก Components**

หน้าจอนี้จะให้เลือก component ที่จะติดตั้ง:

- ✅ **PostgreSQL Server** — ตัวฐานข้อมูลหลัก (จำเป็น)
- ✅ **pgAdmin 4** — เครื่องมือ GUI สำหรับจัดการฐานข้อมูล (แนะนำให้ติดตั้ง)
- ✅ **Stack Builder** — เครื่องมือติดตั้ง extension/driver เพิ่มเติมภายหลัง
- ✅ **Command Line Tools** — `psql`, `pg_dump`, `pg_restore` ฯลฯ (**จำเป็นมาก** อย่าเอาออก)

**ขั้นที่ 4 — เลือก Data Directory**

ค่าเริ่มต้น:
```
C:\Program Files\PostgreSQL\17\data
```
นี่คือตำแหน่งที่ข้อมูลจริงทั้งหมดจะถูกเก็บ (ดูรายละเอียดใน Step 18)

**ขั้นที่ 5 — ตั้งรหัสผ่านสำหรับ superuser**

หน้าจอนี้สำคัญมาก — ระบบจะขอให้ตั้งรหัสผ่านสำหรับ database superuser ชื่อ `postgres`

```
Password: ********
Retype password: ********
```

> **คำแนะนำด้านความปลอดภัย**: ใช้รหัสผ่านที่คาดเดายาก อย่างน้อย 12 ตัวอักษร ผสมตัวพิมพ์ใหญ่-เล็ก ตัวเลข และสัญลักษณ์ **จดบันทึกรหัสผ่านนี้ไว้ในที่ปลอดภัย** เพราะจะต้องใช้ทุกครั้งที่เชื่อมต่อในฐานะ superuser

**ขั้นที่ 6 — กำหนด Port**

ค่าเริ่มต้นคือ **5432** ซึ่งเป็น port มาตรฐานของ PostgreSQL ควรใช้ค่านี้ต่อไปเว้นแต่มี PostgreSQL instance อื่นที่ใช้ port นี้อยู่แล้วในเครื่อง (ในกรณีนั้นอาจเปลี่ยนเป็น 5433)

**ขั้นที่ 7 — เลือก Locale**

หน้าจอนี้ให้เลือก locale สำหรับ database cluster เริ่มต้น

```
Locale: [Default locale]
```

> **สำคัญสำหรับผู้เรียนภาษาไทย**: หากต้องการให้การเรียงลำดับ (sort order) และการเปรียบเทียบข้อความรองรับภาษาไทยอย่างถูกต้อง อาจเลือก `Thai, Thailand` หรือใช้ `C` locale เพื่อความเร็วสูงสุดและพฤติกรรมที่คาดเดาได้ง่าย (เรียงตาม byte value) — ในระดับ production หลายทีมนิยมใช้ `C` หรือ `en_US.UTF-8` แล้วจัดการการเรียงลำดับภาษาไทยที่ชั้น application หรือด้วย collation เฉพาะ column แทน เพราะการเปลี่ยน locale ของทั้ง cluster ภายหลังทำได้ยาก (ต้องสร้าง cluster ใหม่)

**ขั้นที่ 8 — สรุปและติดตั้ง**

ตรวจสอบค่าทั้งหมดในหน้าสรุป แล้วคลิก **Next** เพื่อเริ่มติดตั้งจริง กระบวนการนี้ใช้เวลาประมาณ 2–5 นาที

**ขั้นที่ 9 — Stack Builder (ทางเลือก)**

หลังติดตั้งเสร็จ ระบบจะถามว่าต้องการเปิด Stack Builder หรือไม่ — สามารถข้ามได้ในตอนนี้ (จะกลับมาใช้ภายหลังหากต้องการติดตั้ง extension เช่น PostGIS)

### 12.3 ตรวจสอบการติดตั้งบน Windows

เปิด **SQL Shell (psql)** จาก Start Menu หรือใช้ PowerShell:

```powershell
# ตรวจสอบเวอร์ชันที่ติดตั้ง
& "C:\Program Files\PostgreSQL\17\bin\psql.exe" --version

# ผลลัพธ์ที่คาดหวัง
# psql (PostgreSQL) 17.x
```

เพิ่ม `bin` directory เข้า PATH เพื่อให้เรียก `psql` ได้จากทุกที่:

```powershell
# เพิ่ม PATH แบบถาวรผ่าน PowerShell (ต้องเปิดใหม่ในฐานะ Administrator)
[Environment]::SetEnvironmentVariable(
    "Path",
    $env:Path + ";C:\Program Files\PostgreSQL\17\bin",
    [EnvironmentVariableTarget]::Machine
)
```

จากนั้นเปิด terminal ใหม่แล้วทดสอบ:

```powershell
psql --version
psql -U postgres -h localhost
```

ระบบจะถามรหัสผ่านที่ตั้งไว้ในขั้นที่ 5 — เมื่อใส่ถูกต้องจะเข้าสู่ prompt `postgres=#`

### 12.4 ตรวจสอบ Windows Service

PostgreSQL บน Windows จะถูกติดตั้งเป็น **Windows Service** ที่ชื่อ `postgresql-x64-17` และเริ่มทำงานอัตโนมัติเมื่อบูตเครื่อง

```powershell
# ตรวจสอบสถานะ service
Get-Service -Name "postgresql*"

# หยุด service
Stop-Service -Name "postgresql-x64-17"

# เริ่ม service
Start-Service -Name "postgresql-x64-17"

# รีสตาร์ท service
Restart-Service -Name "postgresql-x64-17"
```

หรือใช้ `services.msc` (Run → พิมพ์ `services.msc`) เพื่อจัดการผ่าน GUI

---

## Step 13: ติดตั้งบน macOS ด้วย Homebrew และ Postgres.app

macOS มีสองวิธีหลักที่นิยมใช้: **Homebrew** (เหมาะกับ developer ที่คุ้นเคยกับ command line) และ **Postgres.app** (เหมาะกับผู้ที่ต้องการ GUI app แบบ double-click เปิด-ปิด)

### 13.1 วิธีที่ 1: ติดตั้งด้วย Homebrew

**ขั้นที่ 1 — ตรวจสอบว่ามี Homebrew แล้วหรือยัง**

```bash
brew --version
```

หากยังไม่มี ให้ติดตั้งก่อน:

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

**ขั้นที่ 2 — อัปเดต Homebrew**

```bash
brew update
```

**ขั้นที่ 3 — ติดตั้ง PostgreSQL**

```bash
# ติดตั้ง PostgreSQL 17
brew install postgresql@17
```

> หมายเหตุ: Homebrew ตั้งชื่อ formula ตาม major version เช่น `postgresql@17`, `postgresql@16` ทำให้สามารถติดตั้งหลายเวอร์ชันขนานกันได้ในเครื่องเดียว

**ขั้นที่ 4 — เพิ่ม PostgreSQL เข้า PATH**

Homebrew บน Apple Silicon (M1/M2/M3/M4) จะติดตั้งใน `/opt/homebrew` ส่วนบน Intel Mac จะอยู่ใน `/usr/local`

```bash
# สำหรับ Apple Silicon (zsh - default shell ของ macOS)
echo 'export PATH="/opt/homebrew/opt/postgresql@17/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc

# สำหรับ Intel Mac
echo 'export PATH="/usr/local/opt/postgresql@17/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
```

**ขั้นที่ 5 — เริ่มต้นใช้งาน service ด้วย brew services**

```bash
# เริ่ม PostgreSQL และตั้งให้รันอัตโนมัติเมื่อ login
brew services start postgresql@17

# ตรวจสอบสถานะ
brew services list

# หยุดการทำงาน
brew services stop postgresql@17

# รีสตาร์ท
brew services restart postgresql@17
```

ผลลัพธ์ของ `brew services list` ควรแสดง:

```
Name            Status  User    File
postgresql@17   started youruser ~/Library/LaunchAgents/homebrew.mxcl.postgresql@17.plist
```

**ขั้นที่ 6 — ทดสอบการเชื่อมต่อ**

Homebrew จะสร้าง default database ให้ตรงกับชื่อ macOS username ของคุณโดยอัตโนมัติ (ไม่มี superuser `postgres` password ตั้งไว้ตั้งแต่แรก เพราะใช้ระบบ authentication แบบ `trust`/peer ผ่าน local user)

```bash
# เชื่อมต่อด้วย username ปัจจุบันของระบบ
psql postgres

# หรือระบุ database ชัดเจน
psql -d postgres -U $(whoami)
```

### 13.2 วิธีที่ 2: ติดตั้งด้วย Postgres.app

Postgres.app เหมาะกับผู้ที่ต้องการความง่าย ไม่ต้องยุ่งกับ command line ในการเปิด-ปิด service

**ขั้นที่ 1 — ดาวน์โหลด**

ไปที่ `https://postgresapp.com` แล้วดาวน์โหลดไฟล์ `.dmg` เวอร์ชันล่าสุด (รองรับหลาย PostgreSQL version ในแอปเดียว)

**ขั้นที่ 2 — ติดตั้ง**

1. เปิดไฟล์ `.dmg` แล้วลาก **Postgres.app** ไปยังโฟลเดอร์ **Applications**
2. เปิดแอปจาก Applications (macOS อาจถามยืนยันเนื่องจากดาวน์โหลดจากอินเทอร์เน็ต — คลิก **Open**)

**ขั้นที่ 3 — Initialize**

เมื่อเปิดแอปครั้งแรก จะเห็นหน้าต่างแสดงรายการเวอร์ชันที่สามารถสร้าง server ได้ คลิก **"Initialize"** เพื่อสร้าง PostgreSQL 17 server ใหม่

**ขั้นที่ 4 — เพิ่ม command line tools เข้า PATH**

Postgres.app แนะนำคำสั่งให้ใน UI แต่โดยทั่วไปคือ:

```bash
sudo mkdir -p /etc/paths.d && \
echo /Applications/Postgres.app/Contents/Versions/latest/bin \
| sudo tee /etc/paths.d/postgresapp
```

จากนั้นเปิด terminal ใหม่แล้วทดสอบ:

```bash
psql --version
```

**ขั้นที่ 5 — เปิด/ปิด server**

การเปิด-ปิด PostgreSQL ทำผ่าน GUI ของแอปโดยตรง — มีปุ่ม start/stop สำหรับแต่ละ server ที่สร้างไว้ ไม่ต้องใช้คำสั่ง terminal เลย

### 13.3 เปรียบเทียบ Homebrew vs Postgres.app

| หัวข้อ | Homebrew | Postgres.app |
|---|---|---|
| การจัดการ | command line (`brew services`) | GUI (double-click) |
| หลายเวอร์ชันพร้อมกัน | ได้ (ติดตั้งแยก formula) | ได้ (ในแอปเดียว) |
| เหมาะกับ | developer ที่ใช้ terminal เป็นหลัก | ผู้เริ่มต้น/ต้องการความง่าย |
| การอัปเดต | `brew upgrade` | ดาวน์โหลดเวอร์ชันใหม่ |
| Auto-start ตอน boot | ได้ (ผ่าน `brew services start`) | ต้องตั้งค่าใน macOS Login Items เอง |

---

## Step 14: ติดตั้งบน Linux (Ubuntu/Debian) ด้วย apt และ PGDG Repository

Ubuntu และ Debian มี PostgreSQL ใน default repository อยู่แล้ว แต่มักเป็นเวอร์ชันที่ **เก่ากว่า** เวอร์ชันล่าสุด ดังนั้นวิธีที่แนะนำสำหรับการเรียนรู้และ production คือใช้ **PostgreSQL APT Repository (PGDG)** อย่างเป็นทางการ

### 14.1 วิธีง่าย: ติดตั้งจาก Default Repository (ไม่แนะนำสำหรับ production)

```bash
sudo apt update
sudo apt install -y postgresql postgresql-contrib
```

วิธีนี้จะได้เวอร์ชันที่ผูกกับ Ubuntu release นั้น ๆ เช่น Ubuntu 24.04 LTS จะได้ PostgreSQL 16 ซึ่งอาจไม่ใช่เวอร์ชันล่าสุด

### 14.2 วิธีแนะนำ: ติดตั้งผ่าน PGDG Repository (ได้เวอร์ชันล่าสุดเสมอ)

**ขั้นที่ 1 — ติดตั้ง prerequisite**

```bash
sudo apt update
sudo apt install -y curl ca-certificates gnupg lsb-release
```

**ขั้นที่ 2 — เพิ่ม PGDG GPG key**

```bash
sudo install -d /usr/share/postgresql-common/pgdg
sudo curl -o /usr/share/postgresql-common/pgdg/apt.postgresql.org.asc \
  --fail https://www.postgresql.org/media/keys/ACCC4CF8.asc
```

**ขั้นที่ 3 — เพิ่ม PGDG repository**

```bash
sudo sh -c 'echo "deb [signed-by=/usr/share/postgresql-common/pgdg/apt.postgresql.org.asc] \
https://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" \
> /etc/apt/sources.list.d/pgdg.list'
```

**ขั้นที่ 4 — อัปเดต package list และติดตั้ง**

```bash
sudo apt update

# ติดตั้ง PostgreSQL 17 พร้อม contrib modules (extension มาตรฐาน)
sudo apt install -y postgresql-17 postgresql-contrib-17

# หรือถ้าต้องการเวอร์ชัน 16
sudo apt install -y postgresql-16 postgresql-contrib-16
```

**ขั้นที่ 5 — ตรวจสอบว่า service ทำงานแล้ว**

```bash
sudo systemctl status postgresql

# ผลลัพธ์ที่คาดหวัง (บางส่วน)
# ● postgresql.service - PostgreSQL RDBMS
#      Loaded: loaded (/lib/systemd/system/postgresql.service; enabled)
#      Active: active (exited) since ...
```

> หมายเหตุ: บน Debian/Ubuntu บริการ `postgresql.service` เป็น meta-service ที่ควบคุม cluster ย่อยจริง ๆ ซึ่งชื่อจะเป็น `postgresql@17-main` (ดูรายละเอียดเพิ่มเติมใน Step 17)

**ขั้นที่ 6 — ตั้งรหัสผ่าน superuser (`postgres`)**

Debian/Ubuntu package สร้าง OS user ชื่อ `postgres` และตั้งค่า `peer` authentication ให้อัตโนมัติ (เชื่อมต่อผ่าน Unix socket โดยไม่ต้องใส่รหัสผ่าน หากใช้ OS user ชื่อเดียวกัน)

```bash
# เข้าสู่ psql ในฐานะ OS user postgres
sudo -u postgres psql

# จากใน psql prompt ให้ตั้งรหัสผ่าน
postgres=# ALTER USER postgres WITH PASSWORD 'YourStrongPassword123!';
postgres=# \q
```

### 14.3 ตรวจสอบเวอร์ชันที่ติดตั้ง

```bash
psql --version
sudo -u postgres psql -c "SELECT version();"
```

---

## Step 15: ติดตั้งบน Linux (RHEL/CentOS/Fedora) ด้วย dnf/yum และ PGDG Repository

Red Hat Enterprise Linux, CentOS Stream, Rocky Linux, AlmaLinux และ Fedora ใช้ package manager `dnf` (หรือ `yum` ในเวอร์ชันเก่า) ระบบเหล่านี้มักมาพร้อม PostgreSQL module ที่ **เก่ากว่า** เวอร์ชันล่าสุดเช่นกัน จึงแนะนำให้ใช้ PGDG repository เช่นเดียวกับ Debian/Ubuntu

### 15.1 ขั้นตอนสำหรับ RHEL 9 / Rocky Linux 9 / AlmaLinux 9

**ขั้นที่ 1 — ปิดโมดูล postgresql ที่มากับระบบ (AppStream module)**

```bash
sudo dnf module list postgresql
sudo dnf module disable -y postgresql
```

**ขั้นที่ 2 — ติดตั้ง PGDG repository RPM**

```bash
sudo dnf install -y \
  https://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86_64/pgdg-redhat-repo-latest.noarch.rpm
```

> สำหรับ RHEL/Rocky/Alma **8** ให้เปลี่ยน `EL-9` เป็น `EL-8` ในลิงก์ด้านบน

**ขั้นที่ 3 — ติดตั้ง PostgreSQL 17**

```bash
sudo dnf install -y postgresql17-server postgresql17-contrib
```

**ขั้นที่ 4 — Initialize database cluster (initdb)**

ต่างจาก Debian/Ubuntu ตรงที่ RPM package ของ RHEL family **ไม่ initialize database cluster ให้อัตโนมัติ** ต้องรันคำสั่งเองครั้งแรก:

```bash
sudo /usr/pgsql-17/bin/postgresql-17-setup initdb
```

ผลลัพธ์ที่คาดหวัง:

```
Initializing database ... OK
```

**ขั้นที่ 5 — เปิดใช้งานและเริ่ม service**

```bash
sudo systemctl enable postgresql-17
sudo systemctl start postgresql-17

# ตรวจสอบสถานะ
sudo systemctl status postgresql-17
```

**ขั้นที่ 6 — ตั้งรหัสผ่าน superuser**

```bash
sudo -u postgres psql -c "ALTER USER postgres WITH PASSWORD 'YourStrongPassword123!';"
```

### 15.2 ขั้นตอนสำหรับ Fedora

Fedora มักมี PostgreSQL เวอร์ชันค่อนข้างใหม่ใน repository มาตรฐานอยู่แล้ว แต่หากต้องการเวอร์ชันเฉพาะเจาะจงหรือเวอร์ชันล่าสุดที่สุด แนะนำใช้ PGDG เช่นกัน:

```bash
# ติดตั้งจาก Fedora repository มาตรฐาน (เวอร์ชันอาจไม่ใช่ล่าสุด)
sudo dnf install -y postgresql-server postgresql-contrib

# initialize
sudo postgresql-setup --initdb

# เปิดใช้งานและ start
sudo systemctl enable --now postgresql
```

หากต้องการเวอร์ชันเฉพาะจาก PGDG บน Fedora ให้ใช้ RPM ที่ระบุ `F-<version>` แทน `EL-<version>` จากหน้า `https://download.postgresql.org/pub/repos/yum/reporpms/`

### 15.3 เปิด Firewall (ถ้าต้องการเชื่อมต่อจากเครื่องอื่น)

```bash
sudo firewall-cmd --permanent --add-port=5432/tcp
sudo firewall-cmd --reload
```

> **คำเตือนด้านความปลอดภัย**: อย่าเปิด port 5432 สู่อินเทอร์เน็ตสาธารณะโดยไม่มีการป้องกันเพิ่มเติม (VPN, SSH tunnel, หรือ firewall rule จำกัด IP) — จะกล่าวถึงรายละเอียดด้านความปลอดภัยเพิ่มเติมใน Part ที่ว่าด้วย Security

---

## Step 16: ติดตั้งผ่าน Docker และ Docker Compose

Docker เป็นวิธีที่ได้รับความนิยมสูงมากในการเรียนรู้และพัฒนา เพราะ **ไม่ทิ้งร่องรอยในระบบ**, ลบทิ้งง่าย, และสามารถรันหลายเวอร์ชัน/หลาย instance พร้อมกันได้โดยไม่ชนกัน

### 16.1 ติดตั้ง Docker (ถ้ายังไม่มี)

**Windows/macOS**: ดาวน์โหลด Docker Desktop จาก `https://www.docker.com/products/docker-desktop/`

**Ubuntu/Debian**:
```bash
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER
# ออกจากระบบแล้วเข้าใหม่เพื่อให้สิทธิ์กลุ่ม docker มีผล
```

ตรวจสอบ:
```bash
docker --version
docker compose version
```

### 16.2 รัน PostgreSQL แบบ Quick Start ด้วย docker run

```bash
docker run --name pg17-demo \
  -e POSTGRES_PASSWORD=YourStrongPassword123! \
  -e POSTGRES_USER=appuser \
  -e POSTGRES_DB=appdb \
  -p 5432:5432 \
  -v pgdata_demo:/var/lib/postgresql/data \
  -d postgres:17
```

อธิบายแต่ละ flag:

| Flag | ความหมาย |
|---|---|
| `--name pg17-demo` | ตั้งชื่อ container เพื่ออ้างอิงง่าย |
| `-e POSTGRES_PASSWORD=...` | รหัสผ่านของ superuser (**จำเป็น** ต้องตั้งเสมอ) |
| `-e POSTGRES_USER=appuser` | สร้าง user เริ่มต้นชื่อ `appuser` (ถ้าไม่ระบุ default คือ `postgres`) |
| `-e POSTGRES_DB=appdb` | สร้าง database เริ่มต้นชื่อ `appdb` |
| `-p 5432:5432` | เชื่อม port 5432 ของ container ออกมาที่ host |
| `-v pgdata_demo:/var/lib/postgresql/data` | ใช้ named volume เก็บข้อมูลถาวร (ไม่หายเมื่อ container ถูกลบ) |
| `-d postgres:17` | รันแบบ background (`detached`) จาก official image เวอร์ชัน 17 |

ตรวจสอบว่า container ทำงานแล้ว:

```bash
docker ps
docker logs pg17-demo
```

เชื่อมต่อผ่าน psql ที่รันอยู่ **ภายใน container**:

```bash
docker exec -it pg17-demo psql -U appuser -d appdb
```

หรือเชื่อมต่อจาก host (หากมี `psql` client ติดตั้งในเครื่อง):

```bash
psql -h localhost -p 5432 -U appuser -d appdb
```

### 16.3 Docker Compose — วิธีที่แนะนำสำหรับการเรียนรู้และพัฒนา

Docker Compose ช่วยให้กำหนดค่าทั้งหมดไว้ในไฟล์เดียว ทำซ้ำได้ และง่ายต่อการแชร์กับทีม

**สร้างโฟลเดอร์โปรเจกต์:**

```bash
mkdir pg-learning-lab && cd pg-learning-lab
```

**สร้างไฟล์ `docker-compose.yml`:**

```yaml
services:
  postgres:
    image: postgres:17
    container_name: pg17-course
    restart: unless-stopped
    environment:
      POSTGRES_USER: appuser
      POSTGRES_PASSWORD: YourStrongPassword123!
      POSTGRES_DB: appdb
      PGDATA: /var/lib/postgresql/data/pgdata
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./init-scripts:/docker-entrypoint-initdb.d:ro
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U appuser -d appdb"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 10s

  pgadmin:
    image: dpage/pgadmin4:latest
    container_name: pgadmin-course
    restart: unless-stopped
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@example.com
      PGADMIN_DEFAULT_PASSWORD: AdminPassword123!
    ports:
      - "8080:80"
    depends_on:
      postgres:
        condition: service_healthy

volumes:
  pgdata:
    name: pg17_course_data
```

อธิบายส่วนสำคัญ:

- **`PGDATA`**: กำหนด subdirectory ย่อยภายใน volume เพื่อหลีกเลี่ยงปัญหาบางระบบไฟล์ที่มี lost+found ปะปนอยู่ใน root ของ volume mount
- **`volumes: pgdata:/var/lib/postgresql/data`**: ใช้ **named volume** ที่ Docker จัดการเอง ข้อมูลจะอยู่ถาวรแม้ลบ container (ต่างจาก bind mount ที่ผูกกับ path บน host โดยตรง)
- **`./init-scripts:/docker-entrypoint-initdb.d:ro`**: ไฟล์ `.sql` หรือ `.sh` ในโฟลเดอร์นี้จะถูกรันอัตโนมัติ **ครั้งแรกที่สร้าง cluster เท่านั้น** (เหมาะสำหรับ seed ข้อมูลเริ่มต้นหรือสร้าง schema)
- **`healthcheck`**: ใช้คำสั่ง `pg_isready` ตรวจสอบว่า PostgreSQL พร้อมรับ connection จริง ๆ แล้วหรือยัง — สำคัญมากเมื่อมี service อื่น (เช่น pgAdmin หรือ application) ที่ต้อง `depends_on` PostgreSQL
- **`pgadmin`**: เพิ่ม pgAdmin 4 เข้ามาเป็น web-based GUI สำหรับจัดการฐานข้อมูลผ่านเบราว์เซอร์ที่ `http://localhost:8080`

**สร้างโฟลเดอร์ init script (ทางเลือก):**

```bash
mkdir init-scripts
```

ตัวอย่างไฟล์ `init-scripts/01-create-schema.sql`:

```sql
CREATE SCHEMA IF NOT EXISTS sales;
CREATE TABLE IF NOT EXISTS sales.customers (
    customer_id SERIAL PRIMARY KEY,
    full_name   TEXT NOT NULL,
    email       TEXT UNIQUE NOT NULL,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### 16.4 คำสั่งจัดการ Docker Compose ที่ใช้บ่อย

```bash
# เริ่มการทำงานทั้งหมดแบบ background
docker compose up -d

# ดู log แบบ real-time
docker compose logs -f postgres

# ตรวจสอบสถานะและ health
docker compose ps

# หยุดการทำงาน (ยังคง container และ volume ไว้)
docker compose stop

# เริ่มต่อจากที่หยุดไว้
docker compose start

# หยุดและลบ container (volume ยังอยู่)
docker compose down

# หยุดและลบทั้ง container และ volume (ข้อมูลหายทั้งหมด — ระวัง!)
docker compose down -v

# เชื่อมต่อ psql เข้าไปใน container
docker compose exec postgres psql -U appuser -d appdb
```

### 16.5 ตรวจสอบว่าติดตั้งสำเร็จ

```bash
docker compose exec postgres psql -U appuser -d appdb -c "SELECT version();"
```

ผลลัพธ์ที่คาดหวัง (ตัวเลขเวอร์ชันย่อยอาจต่างกัน):

```
                                                 version
-----------------------------------------------------------------------------------------------
 PostgreSQL 17.x on x86_64-pc-linux-musl, compiled by gcc ...
(1 row)
```

> **ข้อดีของ Docker สำหรับการเรียน**: สามารถลบทิ้งและสร้างใหม่ได้ในไม่กี่วินาที (`docker compose down -v && docker compose up -d`) เหมาะมากสำหรับการทดลองคำสั่งที่มีความเสี่ยง หรือฝึกกู้คืนข้อมูลจาก backup โดยไม่กลัวพังระบบจริง

---

## Step 17: การตั้งค่าเริ่มต้นหลังติดตั้ง — initdb และการควบคุม Service

### 17.1 initdb คืออะไร

`initdb` คือคำสั่งที่ใช้สร้าง **database cluster** ใหม่ (โครงสร้างไฟล์และ system catalog เริ่มต้นของ PostgreSQL) กระบวนการนี้จะ:

1. สร้าง data directory และไฟล์ configuration เริ่มต้น (`postgresql.conf`, `pg_hba.conf`, `pg_ident.conf`)
2. สร้าง database ระบบ 3 ตัว: `postgres`, `template0`, `template1`
3. สร้าง superuser role (ค่าเริ่มต้นชื่อ `postgres` หรือชื่อผู้ใช้ปัจจุบันของระบบปฏิบัติการ)

โดยทั่วไป installer หรือ package manager ของแต่ละ OS จะรัน `initdb` ให้อัตโนมัติ (ยกเว้น RHEL/Rocky/Alma ที่ต้องรันเอง ดัง Step 15) แต่การเข้าใจว่ามันทำอะไรมีประโยชน์มากเมื่อต้องสร้าง cluster เพิ่มเติมหรือ troubleshoot

**ตัวอย่างการรัน initdb ด้วยตนเอง (manual):**

```bash
# ตัวอย่างบน Linux — สร้าง cluster ใหม่ในตำแหน่งที่กำหนดเอง
sudo -u postgres /usr/lib/postgresql/17/bin/initdb \
  -D /var/lib/postgresql/17/custom_cluster \
  --locale=en_US.UTF-8 \
  --encoding=UTF8 \
  -U postgres \
  --pwprompt
```

พารามิเตอร์สำคัญ:

| พารามิเตอร์ | ความหมาย |
|---|---|
| `-D` | ตำแหน่ง data directory ที่จะสร้างใหม่ |
| `--locale` | locale สำหรับการเรียงลำดับข้อความ |
| `--encoding` | character encoding (แนะนำ `UTF8` เสมอ) |
| `-U` | ชื่อ superuser ที่จะสร้าง |
| `--pwprompt` | ให้ระบบถามรหัสผ่าน superuser ทันที |

### 17.2 การควบคุม Service ด้วย systemctl (Linux)

**Debian/Ubuntu** (multi-cluster aware — service หลักชื่อ `postgresql`, service ของแต่ละ cluster ชื่อ `postgresql@<version>-<clustername>`):

```bash
# ควบคุมทุก cluster พร้อมกัน
sudo systemctl start postgresql
sudo systemctl stop postgresql
sudo systemctl restart postgresql
sudo systemctl status postgresql

# ควบคุมเฉพาะ cluster ใดคลัสเตอร์หนึ่ง
sudo systemctl restart postgresql@17-main
sudo systemctl status postgresql@17-main

# ให้เริ่มอัตโนมัติเมื่อบูตเครื่อง
sudo systemctl enable postgresql
```

**RHEL/Rocky/Alma/Fedora** (service ชื่อระบุเวอร์ชันตรง ๆ):

```bash
sudo systemctl start postgresql-17
sudo systemctl stop postgresql-17
sudo systemctl restart postgresql-17
sudo systemctl status postgresql-17
sudo systemctl enable postgresql-17
```

### 17.3 การควบคุมด้วย pg_ctl (ทุก OS, ระดับต่ำสุด)

`pg_ctl` เป็นเครื่องมือดั้งเดิมที่ใช้ควบคุม PostgreSQL server process โดยตรง ไม่ผ่านระบบ service manager ของ OS มีประโยชน์เมื่อต้องการควบคุมแบบละเอียด หรือรันในสภาพแวดล้อมที่ไม่มี systemd

```bash
# เริ่ม server (ระบุ data directory ด้วย -D และไฟล์ log ด้วย -l)
pg_ctl -D /var/lib/postgresql/17/main -l /var/log/postgresql/startup.log start

# หยุด server แบบปกติ (รอ transaction ที่ค้างอยู่เสร็จก่อน)
pg_ctl -D /var/lib/postgresql/17/main stop -m smart

# หยุด server แบบเร็ว (ตัด connection ทันทีแต่ rollback ธุรกรรมที่ค้างอย่างปลอดภัย)
pg_ctl -D /var/lib/postgresql/17/main stop -m fast

# หยุดแบบฉุกเฉิน (ไม่แนะนำ ใช้เฉพาะกรณีจำเป็น)
pg_ctl -D /var/lib/postgresql/17/main stop -m immediate

# restart
pg_ctl -D /var/lib/postgresql/17/main restart

# ตรวจสอบสถานะ
pg_ctl -D /var/lib/postgresql/17/main status

# reload configuration โดยไม่ต้อง restart (ใช้เมื่อแก้ postgresql.conf บางค่า)
pg_ctl -D /var/lib/postgresql/17/main reload
```

> **Windows**: `pg_ctl.exe` อยู่ใน `C:\Program Files\PostgreSQL\17\bin\pg_ctl.exe` ใช้งานลักษณะเดียวกัน แต่โดยทั่วไปบน Windows แนะนำให้ควบคุมผ่าน Windows Service (`Start-Service`/`Stop-Service`) ตามที่กล่าวใน Step 12 มากกว่า

### 17.4 การควบคุมด้วย brew services (macOS)

ดังที่กล่าวใน Step 13:

```bash
brew services start postgresql@17
brew services stop postgresql@17
brew services restart postgresql@17
brew services list
```

### 17.5 สรุปเครื่องมือควบคุม Service ตาม OS

| OS / วิธีติดตั้ง | เครื่องมือหลัก | เครื่องมือระดับต่ำ |
|---|---|---|
| Windows (EDB installer) | `Start-Service` / `Stop-Service` / services.msc | `pg_ctl.exe` |
| macOS (Homebrew) | `brew services` | `pg_ctl` |
| macOS (Postgres.app) | GUI ปุ่ม start/stop | `pg_ctl` |
| Ubuntu/Debian | `systemctl ... postgresql` | `pg_ctlcluster` (wrapper เฉพาะ Debian) |
| RHEL/Rocky/Fedora | `systemctl ... postgresql-17` | `pg_ctl` |
| Docker | `docker compose start/stop/restart` | `pg_ctl` (ภายใน container) |

---

## Step 18: โครงสร้างไฟล์สำคัญ — postgresql.conf, pg_hba.conf และ Data Directory

การรู้ตำแหน่งไฟล์ configuration เป็นทักษะพื้นฐานที่จำเป็นสำหรับการ troubleshoot และปรับแต่งค่าต่าง ๆ

### 18.1 ไฟล์สำคัญ 3 ไฟล์ที่ต้องรู้จัก

| ไฟล์ | หน้าที่ |
|---|---|
| **`postgresql.conf`** | ไฟล์ configuration หลัก ควบคุม memory, connection limit, logging, WAL, ฯลฯ |
| **`pg_hba.conf`** | ควบคุมว่า "ใคร" เชื่อมต่อจาก "ที่ไหน" เข้า database "ใด" ได้ ด้วยวิธี authentication แบบใด (Host-Based Authentication) |
| **`pg_ident.conf`** | ใช้คู่กับ `peer`/`ident` authentication เพื่อ map ชื่อ OS user กับ PostgreSQL role |

### 18.2 ตำแหน่งไฟล์แยกตามระบบปฏิบัติการ

**Windows (EDB installer):**
```
C:\Program Files\PostgreSQL\17\data\postgresql.conf
C:\Program Files\PostgreSQL\17\data\pg_hba.conf
C:\Program Files\PostgreSQL\17\data\           <- data directory ทั้งหมด
```

**macOS (Homebrew, Apple Silicon):**
```
/opt/homebrew/var/postgresql@17/postgresql.conf
/opt/homebrew/var/postgresql@17/pg_hba.conf
/opt/homebrew/var/postgresql@17/               <- data directory
```

**macOS (Homebrew, Intel):**
```
/usr/local/var/postgresql@17/postgresql.conf
/usr/local/var/postgresql@17/pg_hba.conf
```

**macOS (Postgres.app):**
```
~/Library/Application Support/Postgres/var-17/postgresql.conf
~/Library/Application Support/Postgres/var-17/pg_hba.conf
```

**Ubuntu/Debian (จาก PGDG หรือ default repo):**
```
/etc/postgresql/17/main/postgresql.conf
/etc/postgresql/17/main/pg_hba.conf
/var/lib/postgresql/17/main/                   <- data directory (ไฟล์ข้อมูลจริง)
```

> หมายเหตุพิเศษสำหรับ Debian/Ubuntu: **ไฟล์ configuration แยกออกจาก data directory** โดยเจตนา (ต่างจาก OS อื่น) เพื่อให้จัดการหลาย cluster และหลายเวอร์ชันได้ง่ายขึ้น

**RHEL/Rocky/Alma/Fedora (จาก PGDG):**
```
/var/lib/pgsql/17/data/postgresql.conf
/var/lib/pgsql/17/data/pg_hba.conf
/var/lib/pgsql/17/data/                        <- config และ data อยู่รวมกัน
```

**Docker (official postgres image):**
```
/var/lib/postgresql/data/postgresql.conf
/var/lib/postgresql/data/pg_hba.conf
```
(หรือ `/var/lib/postgresql/data/pgdata/...` หากตั้งค่า `PGDATA` เป็น subdirectory ตามตัวอย่างใน Step 16.3)

### 18.3 วิธีหา Data Directory จริงด้วยคำสั่ง SQL (ใช้ได้ทุก OS)

หากจำ path ไม่ได้ วิธีที่แม่นยำที่สุดคือถาม PostgreSQL เอง:

```sql
SHOW data_directory;
SHOW config_file;
SHOW hba_file;
```

ตัวอย่างผลลัพธ์:

```
postgres=# SHOW data_directory;
        data_directory
-------------------------------
 /var/lib/postgresql/17/main
(1 row)

postgres=# SHOW hba_file;
              hba_file
-------------------------------------
 /etc/postgresql/17/main/pg_hba.conf
(1 row)
```

### 18.4 โครงสร้างภายใน Data Directory

```
data/
├── base/               # ไฟล์ข้อมูลจริงของแต่ละ database (แยกเป็นโฟลเดอร์ตาม OID)
├── global/             # system catalog ที่ใช้ร่วมกันทั้ง cluster
├── pg_wal/             # Write-Ahead Log — สำคัญมากสำหรับ crash recovery และ replication
├── pg_tblspc/          # symlink ไปยัง tablespace ที่สร้างเพิ่มเติม (ถ้ามี)
├── pg_stat/            # สถิติการทำงานแบบถาวร
├── postgresql.conf     # (บางระบบ) ไฟล์ configuration หลัก
├── pg_hba.conf          # (บางระบบ) กฎ authentication
├── PG_VERSION           # ไฟล์ text บอกเลข major version ของ cluster นี้
└── postmaster.pid       # PID ของ process หลักที่กำลังรันอยู่ (มีเฉพาะตอน server ทำงาน)
```

> **คำเตือน**: ห้ามแก้ไขหรือลบไฟล์ใน `base/`, `global/`, `pg_wal/` ด้วยตนเองเด็ดขาด การแก้ไขไฟล์เหล่านี้โดยตรง (นอกเหนือจากผ่านคำสั่ง SQL หรือเครื่องมือของ PostgreSQL) อาจทำให้ข้อมูลเสียหายถาวร

### 18.5 ตัวอย่างการแก้ไข postgresql.conf ที่พบบ่อย

```ini
# postgresql.conf (ตัวอย่างค่าที่มักปรับแต่งช่วงเริ่มต้น)

listen_addresses = 'localhost'   # เปลี่ยนเป็น '*' หากต้องการรับ connection จากเครื่องอื่น
port = 5432
max_connections = 100
shared_buffers = 128MB           # โดยทั่วไปตั้งเป็น ~25% ของ RAM เครื่อง
log_destination = 'stderr'
logging_collector = on
log_directory = 'log'
```

หลังแก้ไข ต้อง reload หรือ restart (บางค่า เช่น `shared_buffers` ต้อง **restart** เท่านั้น reload ไม่พอ):

```bash
# Linux
sudo systemctl reload postgresql
# หรือ
sudo systemctl restart postgresql
```

---

## Step 19: การเชื่อมต่อครั้งแรกด้วย psql และการแก้ปัญหาที่พบบ่อย

### 19.1 คำสั่งเชื่อมต่อพื้นฐาน

```bash
# รูปแบบทั่วไป
psql -h <host> -p <port> -U <username> -d <database>

# ตัวอย่าง: เชื่อมต่อ local server
psql -h localhost -p 5432 -U postgres -d postgres

# เชื่อมต่อผ่าน Unix socket (ไม่ระบุ -h) — ใช้ peer authentication บน Linux
psql -U postgres -d postgres
```

เมื่อเชื่อมต่อสำเร็จจะเห็น prompt:

```
psql (17.x)
Type "help" for help.

postgres=#
```

### 19.2 คำสั่ง psql พื้นฐานที่ควรรู้ทันที

```sql
\l              -- แสดงรายชื่อ database ทั้งหมด
\c dbname       -- เปลี่ยนไปเชื่อมต่อ database อื่น
\dt             -- แสดงรายชื่อ table ใน schema ปัจจุบัน
\du             -- แสดงรายชื่อ role/user ทั้งหมด
\d tablename    -- แสดงโครงสร้างของ table
\conninfo       -- แสดงข้อมูลการเชื่อมต่อปัจจุบัน
\q              -- ออกจาก psql
\?              -- แสดงความช่วยเหลือคำสั่ง psql ทั้งหมด
\h              -- แสดงความช่วยเหลือคำสั่ง SQL
```

### 19.3 Troubleshooting: ปัญหาที่พบบ่อยและวิธีแก้

#### ปัญหาที่ 1: `connection refused`

```
psql: error: connection to server at "localhost" (127.0.0.1), port 5432 failed:
Connection refused
	Is the server running on that host and accepting TCP/IP connections?
```

**สาเหตุที่เป็นไปได้และวิธีแก้:**

1. **PostgreSQL service ไม่ได้ทำงาน**
   ```bash
   # ตรวจสอบสถานะ
   sudo systemctl status postgresql        # Linux
   brew services list                       # macOS
   Get-Service postgresql*                  # Windows PowerShell
   docker compose ps                        # Docker

   # แก้ไข: เริ่ม service
   sudo systemctl start postgresql
   ```

2. **`listen_addresses` ไม่ได้เปิดรับการเชื่อมต่อ**

   ตรวจสอบใน `postgresql.conf`:
   ```ini
   listen_addresses = 'localhost'   # หรือ '*' หากต้องการรับจากทุก network interface
   ```
   หากแก้ไขแล้วต้อง restart service (ไม่ใช่แค่ reload)

3. **Firewall block port 5432**
   ```bash
   sudo ufw allow 5432/tcp          # Ubuntu (ufw)
   sudo firewall-cmd --add-port=5432/tcp --permanent && sudo firewall-cmd --reload   # RHEL
   ```

4. **ใช้ port ผิด** (เช่นตั้ง custom port ไว้ตอนติดตั้ง) — ตรวจสอบด้วย `SHOW port;` หลัง login ด้วยวิธีอื่น หรือดูใน `postgresql.conf`

#### ปัญหาที่ 2: `password authentication failed for user`

```
psql: error: connection to server at "localhost" (127.0.0.1), port 5432 failed:
FATAL:  password authentication failed for user "postgres"
```

**สาเหตุและวิธีแก้:**

1. **รหัสผ่านผิดจริง** — ลองพิมพ์ใหม่อย่างระมัดระวัง (ระวัง Caps Lock, keyboard layout)

2. **ยังไม่เคยตั้งรหัสผ่าน** (พบบ่อยบน Linux ที่ใช้ `peer` authentication เป็นค่าเริ่มต้น) — ให้เข้าผ่าน OS user ก่อนแล้วตั้งรหัสผ่าน:
   ```bash
   sudo -u postgres psql -c "ALTER USER postgres WITH PASSWORD 'NewPassword123!';"
   ```

3. **`pg_hba.conf` ตั้งค่า authentication method ไม่ตรงกับที่ client คาดหวัง** — ดูรายละเอียดหัวข้อถัดไป

#### ปัญหาที่ 3: เข้าใจ Authentication Methods ใน pg_hba.conf (peer / trust / md5 / scram-sha-256)

ไฟล์ `pg_hba.conf` มีรูปแบบแต่ละบรรทัดดังนี้:

```
# TYPE  DATABASE  USER  ADDRESS         METHOD
local   all       all                   peer
host    all       all   127.0.0.1/32    scram-sha-256
host    all       all   ::1/128         scram-sha-256
```

| METHOD | ความหมาย | ความปลอดภัย |
|---|---|---|
| `trust` | เชื่อมต่อได้เลยโดยไม่ต้องยืนยันตัวตนใด ๆ | **ไม่ปลอดภัย** ใช้เฉพาะ local dev/testing เท่านั้น |
| `peer` | ใช้ OS username ของระบบปฏิบัติการจับคู่กับ PostgreSQL role โดยตรง (เฉพาะ local socket connection) | ปลอดภัยระดับ local เหมาะกับ admin task บนเครื่อง server เอง |
| `md5` | ยืนยันด้วยรหัสผ่านแบบ hash MD5 (แบบเก่า) | ใช้ได้ แต่ล้าสมัยกว่า scram-sha-256 |
| `scram-sha-256` | ยืนยันด้วยรหัสผ่านแบบ SCRAM (มาตรฐานใหม่ ปลอดภัยกว่า md5) | **แนะนำ** เป็นค่า default ตั้งแต่ PostgreSQL 14 เป็นต้นไป |
| `reject` | ปฏิเสธการเชื่อมต่อเสมอ | ใช้กันไม่ให้บาง pattern เชื่อมต่อได้ |

**วิธีแก้ปัญหาเมื่อ authentication method ไม่ตรงกับที่ต้องการ:**

ตัวอย่าง: ต้องการเปลี่ยนจาก `peer` เป็น `md5`/`scram-sha-256` เพื่อให้เชื่อมต่อด้วยรหัสผ่านได้แม้ไม่ได้ login เป็น OS user นั้น

```bash
sudo nano /etc/postgresql/17/main/pg_hba.conf
```

แก้บรรทัด:
```diff
- local   all             all                                     peer
+ local   all             all                                     scram-sha-256
```

จากนั้น reload configuration:
```bash
sudo systemctl reload postgresql
```

> **ข้อควรระวัง**: หากเปลี่ยน `local` เป็น `scram-sha-256` แต่ยังไม่เคยตั้งรหัสผ่านให้ role นั้น จะเชื่อมต่อไม่ได้เลยแม้แต่ผ่าน OS user เดิม — ควรตั้งรหัสผ่านให้ครบทุก role ที่เกี่ยวข้องก่อนเปลี่ยนวิธี authentication

#### ปัญหาที่ 4: `FATAL: database "xxx" does not exist`

```
psql: error: connection to server ... failed: FATAL:  database "myapp" does not exist
```

**วิธีแก้**: สร้าง database ก่อนใช้งาน

```sql
CREATE DATABASE myapp;
```

หรือจาก command line โดยตรง:
```bash
createdb -U postgres myapp
```

#### ปัญหาที่ 5: `role "xxx" does not exist`

```
psql: error: FATAL:  role "myuser" does not exist
```

**วิธีแก้**: สร้าง role/user ก่อน

```sql
CREATE ROLE myuser WITH LOGIN PASSWORD 'SomePassword123!';
```

หรือจาก command line:
```bash
createuser -U postgres --interactive myuser
```

#### ปัญหาที่ 6: Docker container ขึ้น แต่เชื่อมต่อจาก host ไม่ได้

**ตรวจสอบ:**

```bash
# 1. container ยังรันอยู่จริงไหม
docker compose ps

# 2. port ถูก map ออกมาถูกต้องไหม
docker port pg17-course

# 3. ดู log ว่า PostgreSQL start สำเร็จหรือมี error
docker compose logs postgres

# 4. healthcheck ผ่านหรือยัง (สถานะควรเป็น "healthy" ไม่ใช่ "starting" ค้างอยู่)
docker inspect --format='{{.State.Health.Status}}' pg17-course
```

**สาเหตุที่พบบ่อย**: มี PostgreSQL instance อื่นในเครื่อง host ใช้ port 5432 อยู่แล้ว (เช่นติดตั้งผ่าน Homebrew ไว้ด้วย) ทำให้ port ชนกัน — แก้โดยเปลี่ยน port mapping ใน `docker-compose.yml`:

```yaml
ports:
  - "5433:5432"   # เปลี่ยน host port เป็น 5433 แทน
```

แล้วเชื่อมต่อด้วย `psql -h localhost -p 5433 ...`

### 19.4 เครื่องมือช่วยวินิจฉัยปัญหาโดยรวม

```bash
# ตรวจสอบว่า PostgreSQL process กำลังฟัง port อะไรอยู่
sudo ss -tlnp | grep postgres      # Linux
sudo lsof -i :5432                  # macOS/Linux
netstat -ano | findstr 5432         # Windows PowerShell/CMD

# ดู log ล่าสุดของ PostgreSQL (Linux)
sudo tail -n 50 /var/log/postgresql/postgresql-17-main.log
```

---

## Step 20: การอัปเกรดเวอร์ชัน PostgreSQL เบื้องต้นและการถอนการติดตั้ง

### 20.1 ความแตกต่างระหว่าง Minor Upgrade และ Major Upgrade

| ประเภท | ตัวอย่าง | ความซับซ้อน | Downtime |
|---|---|---|---|
| **Minor upgrade** | 17.1 → 17.2 | ต่ำมาก — เพียง update package แล้ว restart | สั้นมาก (แค่เวลา restart) |
| **Major upgrade** | 16.x → 17.x | สูง — โครงสร้าง data directory เปลี่ยน ต้องใช้เครื่องมือแปลงข้อมูล | ขึ้นกับวิธีที่เลือก (นาทีถึงชั่วโมง) |

> **หลักการสำคัญ**: Minor version ของ major version เดียวกัน (เช่น 17.1, 17.2, 17.3) ใช้โครงสร้าง data directory แบบเดียวกันเสมอ จึง**ควรอัปเดต minor version อยู่เสมอ**ทันทีที่มีออกใหม่ เพราะมักแก้ security vulnerability และ bug โดยไม่มีความเสี่ยงเรื่อง data compatibility

### 20.2 Minor Upgrade — ทำได้ง่ายในทุก OS

**Ubuntu/Debian:**
```bash
sudo apt update
sudo apt upgrade postgresql-17
sudo systemctl restart postgresql
```

**RHEL/Rocky/Fedora:**
```bash
sudo dnf upgrade postgresql17-server postgresql17-contrib
sudo systemctl restart postgresql-17
```

**macOS (Homebrew):**
```bash
brew update
brew upgrade postgresql@17
brew services restart postgresql@17
```

**Docker**: เพียงเปลี่ยน tag ของ image เป็นเวอร์ชันย่อยใหม่ (หรือใช้ `postgres:17` แบบไม่ระบุ patch version เพื่อรับ minor update อัตโนมัติทุกครั้งที่ pull ใหม่):

```bash
docker compose pull postgres
docker compose up -d
```

### 20.3 Major Upgrade — แนวคิดและตัวเลือก

การอัปเกรด major version (เช่น 16 → 17) มีความซับซ้อนเพราะ **โครงสร้างไฟล์ภายใน data directory เปลี่ยนแปลง** ไม่สามารถใช้ data directory เดิมกับ binary เวอร์ชันใหม่ตรง ๆ ได้ มี 3 แนวทางหลัก:

#### แนวทางที่ 1: pg_dump / pg_restore (ปลอดภัยที่สุด แต่ downtime นานที่สุด)

```bash
# Export ข้อมูลทั้งหมดจาก PostgreSQL เวอร์ชันเก่า
pg_dumpall -U postgres -h localhost -p 5432 > full_backup.sql

# ติดตั้ง PostgreSQL เวอร์ชันใหม่ (คนละ cluster/port)
# จากนั้น import กลับเข้าเวอร์ชันใหม่
psql -U postgres -h localhost -p 5433 -f full_backup.sql
```

เหมาะกับฐานข้อมูลขนาดเล็ก-กลาง ที่รับ downtime ระหว่าง dump/restore ได้

#### แนวทางที่ 2: pg_upgrade (เร็วกว่า เหมาะกับฐานข้อมูลขนาดใหญ่)

`pg_upgrade` เป็นเครื่องมือที่มากับ PostgreSQL เอง ใช้แปลง data directory จากเวอร์ชันเก่าไปเวอร์ชันใหม่โดยตรง (ในโหมด `--link` จะใช้ hard link ทำให้เร็วมากและใช้พื้นที่ดิสก์เพิ่มน้อยมาก)

**แนวคิดขั้นตอน (ตัวอย่างบน Linux, อัปเกรด 16 → 17):**

```bash
# 1. ติดตั้ง PostgreSQL 17 คู่ขนานกับ 16 (ยังไม่ initdb เอง — pg_upgrade จะจัดการ)
sudo apt install -y postgresql-17

# 2. หยุด PostgreSQL ทั้งสองเวอร์ชันก่อนเริ่ม
sudo systemctl stop postgresql

# 3. initialize cluster ใหม่สำหรับ 17 (หากยังไม่มี)
sudo -u postgres /usr/lib/postgresql/17/bin/initdb -D /var/lib/postgresql/17/main

# 4. รัน pg_upgrade (ตัวอย่างค่า path อาจต่างกันตามระบบ)
sudo -u postgres /usr/lib/postgresql/17/bin/pg_upgrade \
  --old-datadir=/var/lib/postgresql/16/main \
  --new-datadir=/var/lib/postgresql/17/main \
  --old-bindir=/usr/lib/postgresql/16/bin \
  --new-bindir=/usr/lib/postgresql/17/bin \
  --link

# 5. เริ่ม cluster ใหม่และตรวจสอบ
sudo systemctl start postgresql@17-main

# 6. รัน analyze ใหม่ตามที่ pg_upgrade แนะนำ (สคริปต์ analyze_new_cluster.sh)
```

> **ข้อควรรู้**: `pg_upgrade` มีโหมด `--check` ให้รันตรวจสอบความเข้ากันได้ก่อนโดยไม่แก้ไขอะไรจริง แนะนำให้รันตรวจสอบก่อนเสมอ:
> ```bash
> pg_upgrade --check --old-datadir=... --new-datadir=... --old-bindir=... --new-bindir=...
> ```
> และ **สำรองข้อมูล (backup) ก่อนอัปเกรดเสมอ** ไม่ว่าจะมั่นใจแค่ไหนก็ตาม

#### แนวทางที่ 3: Logical Replication (สำหรับ zero-downtime หรือ near-zero-downtime upgrade)

สำหรับระบบ production ขนาดใหญ่ที่รับ downtime แทบไม่ได้ สามารถใช้ **logical replication** สร้าง PostgreSQL เวอร์ชันใหม่เป็น replica ที่ sync ข้อมูลแบบ real-time จากเวอร์ชันเก่า แล้วค่อย switch traffic มาเมื่อพร้อม — เทคนิคนี้จะกล่าวถึงรายละเอียดในบทที่ว่าด้วย Replication ระดับ Advanced ของหลักสูตรนี้

#### สรุปเปรียบเทียบ

| วิธี | Downtime | ความซับซ้อน | เหมาะกับ |
|---|---|---|---|
| pg_dump/pg_restore | สูง | ต่ำ | database ขนาดเล็ก, การเรียนรู้ |
| pg_upgrade --link | ต่ำ (นาที) | ปานกลาง | database ขนาดกลาง-ใหญ่ในเครื่องเดียว |
| Logical Replication | ต่ำมาก/แทบไม่มี | สูง | production ที่ต้องการ near-zero downtime |
| Docker (เปลี่ยน image + dump/restore) | ขึ้นกับข้อมูล | ต่ำ | dev/staging environment |

### 20.4 การถอนการติดตั้ง (Uninstall)

#### Windows

1. เปิด **Settings → Apps → Installed apps**
2. ค้นหา "PostgreSQL 17" แล้วเลือก **Uninstall**
3. ทำตาม wizard ของ EnterpriseDB uninstaller
4. ลบ data directory ที่เหลือด้วยตนเอง (uninstaller มักถามว่าจะลบ data directory ด้วยหรือไม่ — **ระวัง** หากยังต้องการข้อมูลอยู่ ให้ backup ก่อนตอบตกลง):
   ```powershell
   Remove-Item -Recurse -Force "C:\Program Files\PostgreSQL\17\data"
   ```

#### macOS (Homebrew)

```bash
# หยุด service ก่อน
brew services stop postgresql@17

# ถอนการติดตั้ง
brew uninstall postgresql@17

# ลบข้อมูลที่เหลือ (ถ้าต้องการล้างทั้งหมด — backup ก่อนถ้าจำเป็น)
rm -rf /opt/homebrew/var/postgresql@17
```

#### macOS (Postgres.app)

ลาก **Postgres.app** จากโฟลเดอร์ Applications ไปที่ Trash จากนั้นลบข้อมูลที่เหลือ (ถ้าต้องการ):

```bash
rm -rf ~/Library/Application\ Support/Postgres
```

#### Ubuntu/Debian

```bash
# หยุด service
sudo systemctl stop postgresql

# ถอนการติดตั้ง package (เก็บไฟล์ configuration ไว้)
sudo apt remove postgresql-17 postgresql-contrib-17

# ถอนการติดตั้งแบบสมบูรณ์ รวม configuration (purge)
sudo apt purge postgresql-17 postgresql-contrib-17
sudo apt autoremove

# ลบ data directory ที่เหลือ (backup ก่อนถ้าจำเป็น!)
sudo rm -rf /var/lib/postgresql/17
sudo rm -rf /etc/postgresql/17
```

#### RHEL/Rocky/Alma/Fedora

```bash
sudo systemctl stop postgresql-17
sudo dnf remove postgresql17-server postgresql17-contrib

# ลบ data directory ที่เหลือ (backup ก่อนถ้าจำเป็น!)
sudo rm -rf /var/lib/pgsql/17
```

#### Docker

```bash
# หยุดและลบ container พร้อม volume (ข้อมูลหายทั้งหมด)
docker compose down -v

# หรือลบเฉพาะ image ถ้าต้องการเก็บ volume ไว้ใช้ต่อภายหลัง
docker compose down
docker image rm postgres:17
```

> **คำเตือนสุดท้าย**: ไม่ว่าจะถอนการติดตั้งด้วยวิธีใด **ให้สำรองข้อมูลด้วย `pg_dumpall` หรือ `pg_dump` ก่อนเสมอ** หากมีข้อมูลที่มีค่าอยู่ใน database การลบ data directory เป็นการกระทำที่ย้อนกลับไม่ได้

---

## สรุปท้ายบท

ใน Part นี้ เราได้เรียนรู้การติดตั้ง PostgreSQL อย่างครบถ้วนในทุกแพลตฟอร์มหลักที่ใช้งานจริง:

- **การเตรียมความพร้อม**: เลือกเวอร์ชัน PostgreSQL 17 (หรือ 16 สำหรับความเสถียรสูงสุด) และตรวจสอบข้อกำหนดระบบก่อนเริ่ม
- **Windows**: ใช้ EnterpriseDB installer ซึ่งมี wizard ที่ครบถ้วน ตั้งแต่ password, port, ไปจนถึง locale และจัดการผ่าน Windows Service
- **macOS**: มีสองทางเลือกคือ Homebrew (เหมาะกับ CLI-first developer) และ Postgres.app (เหมาะกับผู้ต้องการ GUI)
- **Linux**: ทั้ง Ubuntu/Debian (apt + PGDG) และ RHEL family (dnf + PGDG) ควรใช้ PGDG repository เพื่อให้ได้เวอร์ชันล่าสุดเสมอ และ RHEL family ต้องรัน `initdb` เอง
- **Docker/Docker Compose**: วิธีที่ทำซ้ำได้และสะอาดที่สุด เหมาะกับการเรียนรู้และพัฒนา พร้อมตัวอย่าง `docker-compose.yml` ที่มี volume, environment variables และ healthcheck ครบถ้วน
- **การจัดการ service**: `systemctl`, `pg_ctl`, และ `brew services` คือเครื่องมือหลักในการควบคุมการทำงานของ PostgreSQL
- **ไฟล์สำคัญ**: `postgresql.conf`, `pg_hba.conf`, และ data directory มีตำแหน่งต่างกันในแต่ละ OS — ใช้ `SHOW data_directory;` เพื่อหาตำแหน่งจริงได้เสมอ
- **การเชื่อมต่อและ troubleshooting**: เข้าใจความแตกต่างของ authentication method (`trust`, `peer`, `md5`, `scram-sha-256`) และวิธีแก้ปัญหา `connection refused` และ `authentication failed` ที่พบบ่อยที่สุด
- **การอัปเกรดและถอนการติดตั้ง**: minor upgrade ทำได้ง่ายและควรทำสม่ำเสมอ ส่วน major upgrade ต้องวางแผนด้วย `pg_dump`/`pg_restore` หรือ `pg_upgrade` และสำรองข้อมูลเสมอก่อนถอนการติดตั้ง

ทักษะการติดตั้งเหล่านี้เป็นพื้นฐานสำคัญที่จะทำให้ทุกบทถัดไปในหลักสูตรราบรื่น เพราะไม่ว่าจะเรียนรู้ SQL, การออกแบบฐานข้อมูล, Performance Tuning หรือ Replication ก็ล้วนต้องมี PostgreSQL instance ที่ใช้งานได้พร้อมอยู่เสมอ

---

## แบบฝึกหัด

**แบบฝึกหัดที่ 1**

จงอธิบายความแตกต่างระหว่าง minor upgrade และ major upgrade ของ PostgreSQL พร้อมยกตัวอย่างเลขเวอร์ชันประกอบ

<details>
<summary>เฉลย</summary>

Minor upgrade คือการอัปเดตภายใน major version เดียวกัน (เช่น 17.1 → 17.2) ซึ่งใช้โครงสร้าง data directory แบบเดียวกัน จึงทำได้ง่ายเพียงอัปเดต package แล้ว restart service โดยไม่ต้องแปลงข้อมูล

Major upgrade คือการอัปเดตข้ามเลขเวอร์ชันหลัก (เช่น 16.x → 17.x) ซึ่งโครงสร้างไฟล์ภายใน data directory อาจเปลี่ยนแปลง ต้องใช้เครื่องมือแปลงข้อมูล เช่น `pg_dump`/`pg_restore` หรือ `pg_upgrade` และมีความซับซ้อนกับ downtime มากกว่า
</details>

---

**แบบฝึกหัดที่ 2**

บนเครื่อง Ubuntu 24.04 หากติดตั้ง PostgreSQL ด้วย `sudo apt install postgresql` โดยไม่เพิ่ม PGDG repository ก่อน จะได้เวอร์ชันใด และเพราะเหตุใดจึงแนะนำให้เพิ่ม PGDG repository ก่อนติดตั้งสำหรับการเรียนรู้หลักสูตรนี้

<details>
<summary>เฉลย</summary>

จะได้ PostgreSQL เวอร์ชันที่ผูกกับ Ubuntu release นั้น ๆ (สำหรับ Ubuntu 24.04 LTS คือ PostgreSQL 16) ซึ่งอาจไม่ใช่เวอร์ชันล่าสุด แนะนำให้เพิ่ม PGDG repository ก่อน เพราะจะทำให้สามารถเลือกติดตั้งเวอร์ชันล่าสุด (เช่น 17) ได้โดยตรง ทำให้เนื้อหาที่เรียนตรงกับฟีเจอร์ใหม่ล่าสุดของหลักสูตร และได้รับ security patch เร็วกว่า
</details>

---

**แบบฝึกหัดที่ 3**

จงเขียนคำสั่ง `docker run` (หรือปรับจาก docker-compose.yml ในบทเรียน) เพื่อรัน PostgreSQL 17 ที่มี:
- superuser password เป็น `MySecret2026!`
- database เริ่มต้นชื่อ `training_db`
- เปิด port ที่ host เป็น 5555 (map ไปยัง container port 5432)
- ใช้ named volume ชื่อ `training_pgdata`

<details>
<summary>เฉลย</summary>

```bash
docker run --name pg-training \
  -e POSTGRES_PASSWORD=MySecret2026! \
  -e POSTGRES_DB=training_db \
  -p 5555:5432 \
  -v training_pgdata:/var/lib/postgresql/data \
  -d postgres:17
```

เชื่อมต่อทดสอบ:
```bash
psql -h localhost -p 5555 -U postgres -d training_db
```
</details>

---

**แบบฝึกหัดที่ 4**

เมื่อเจอ error ข้อความว่า `FATAL: password authentication failed for user "postgres"` บน Linux ที่เพิ่งติดตั้งเสร็จใหม่ ๆ และยังไม่เคยตั้งรหัสผ่านให้ role `postgres` เลย ควรแก้ไขอย่างไร

<details>
<summary>เฉลย</summary>

ให้เข้าสู่ psql ผ่าน OS user `postgres` ก่อน (ซึ่งใช้ `peer` authentication ไม่ต้องใช้รหัสผ่าน) แล้วตั้งรหัสผ่านใหม่:

```bash
sudo -u postgres psql -c "ALTER USER postgres WITH PASSWORD 'NewSecurePassword123!';"
```

จากนั้นจึงสามารถเชื่อมต่อด้วยรหัสผ่านที่ตั้งใหม่ได้ผ่าน `psql -U postgres -h localhost`
</details>

---

**แบบฝึกหัดที่ 5**

จงอธิบายความแตกต่างระหว่าง authentication method `trust`, `peer`, และ `scram-sha-256` ใน `pg_hba.conf` พร้อมบอกว่าแบบใดควรใช้ใน production

<details>
<summary>เฉลย</summary>

- `trust`: เชื่อมต่อได้ทันทีโดยไม่ต้องยืนยันตัวตนใด ๆ ไม่ปลอดภัย ควรใช้เฉพาะ local development/testing เท่านั้น
- `peer`: จับคู่ OS username ของระบบปฏิบัติการกับ PostgreSQL role โดยตรง ใช้ได้เฉพาะการเชื่อมต่อผ่าน local Unix socket ปลอดภัยระดับหนึ่งสำหรับงาน admin บนเครื่อง server เอง
- `scram-sha-256`: ยืนยันตัวตนด้วยรหัสผ่านแบบ SCRAM ซึ่งเป็นมาตรฐานที่ปลอดภัยที่สุดในบรรดา password-based authentication ของ PostgreSQL ปัจจุบัน

สำหรับ production **ควรใช้ `scram-sha-256`** เสมอสำหรับการเชื่อมต่อผ่านเครือข่าย (TCP/IP) และหลีกเลี่ยง `trust` โดยสิ้นเชิง
</details>

---

**แบบฝึกหัดที่ 6**

บน RHEL/Rocky Linux หลังติดตั้ง package `postgresql17-server` แล้ว เมื่อรัน `sudo systemctl start postgresql-17` กลับพบว่า service ไม่สามารถ start ได้ ให้สันนิษฐานสาเหตุที่เป็นไปได้มากที่สุด และวิธีแก้

<details>
<summary>เฉลย</summary>

สาเหตุที่พบบ่อยที่สุดคือยังไม่ได้รัน `initdb` เพื่อสร้าง database cluster เนื่องจาก RPM package ของ RHEL family ไม่ initialize cluster ให้อัตโนมัติ (ต่างจาก Debian/Ubuntu) วิธีแก้คือรัน:

```bash
sudo /usr/pgsql-17/bin/postgresql-17-setup initdb
sudo systemctl start postgresql-17
```
</details>

---

**แบบฝึกหัดที่ 7**

หากต้องการทราบตำแหน่งจริงของ data directory และไฟล์ `pg_hba.conf` ของ PostgreSQL instance ที่กำลังรันอยู่ (โดยไม่ทราบว่าติดตั้งด้วยวิธีใด) ควรใช้คำสั่งใด

<details>
<summary>เฉลย</summary>

เชื่อมต่อเข้า psql แล้วรันคำสั่ง SQL:

```sql
SHOW data_directory;
SHOW hba_file;
SHOW config_file;
```

วิธีนี้แม่นยำที่สุดเพราะถามจาก PostgreSQL server โดยตรง ไม่ต้องเดาตาม path มาตรฐานของแต่ละ OS
</details>

---

**แบบฝึกหัดที่ 8**

จงอธิบายว่าทำไม `docker-compose.yml` ในบทเรียนจึงใช้ **named volume** (`pgdata:/var/lib/postgresql/data`) แทนที่จะ mount โฟลเดอร์บน host โดยตรง (bind mount) และ healthcheck ที่กำหนดไว้มีประโยชน์อย่างไร

<details>
<summary>เฉลย</summary>

Named volume ให้ Docker จัดการพื้นที่จัดเก็บข้อมูลเองอย่างมีประสิทธิภาพ ทำให้ข้อมูลคงอยู่ถาวรแม้ container จะถูกลบหรือสร้างใหม่ และหลีกเลี่ยงปัญหาเรื่อง permission หรือ path compatibility ระหว่าง OS ที่ต่างกัน (โดยเฉพาะบน Windows/macOS ที่ bind mount อาจมีปัญหาประสิทธิภาพหรือสิทธิ์การเข้าถึงไฟล์)

Healthcheck ที่ใช้ `pg_isready` ช่วยให้ Docker (และ service อื่นที่ `depends_on` เช่น pgAdmin หรือ application container) รู้ได้แน่ชัดว่า PostgreSQL **พร้อมรับ connection จริง ๆ แล้ว** ไม่ใช่แค่ process เริ่มทำงาน (ซึ่งอาจยังอยู่ระหว่างขั้นตอน initialize) ช่วยป้องกันปัญหา service อื่นพยายามเชื่อมต่อก่อนที่ database จะพร้อมจริง
</details>

---

**แบบฝึกหัดที่ 9**

ระหว่าง `pg_ctl stop -m smart`, `-m fast`, และ `-m immediate` แบบใดเหมาะกับการปิด production server ก่อนทำ maintenance ตามแผนที่วางไว้มากที่สุด และเพราะเหตุใด

<details>
<summary>เฉลย</summary>

`-m fast` เหมาะที่สุดสำหรับ maintenance ตามแผน เพราะจะตัดการเชื่อมต่อ client ทันที (ไม่ต้องรอ session ที่ idle อยู่เฉย ๆ เหมือน `smart`) แต่ยัง rollback transaction ที่ค้างอยู่อย่างปลอดภัยก่อนปิด (ต่างจาก `immediate` ที่ปิดทันทีโดยไม่ shutdown อย่างสะอาด ซึ่งอาจทำให้ crash recovery ต้องทำงานตอน start ใหม่) `-m smart` อาจรอนานเกินไปหากมี session ค้างอยู่ ส่วน `-m immediate` ควรสงวนไว้เฉพาะกรณีฉุกเฉินเท่านั้น
</details>

---

**แบบฝึกหัดที่ 10**

จงเรียงลำดับขั้นตอนต่อไปนี้ให้ถูกต้องสำหรับการทำ major upgrade ด้วย `pg_upgrade` บน Linux (Ubuntu/Debian):

ก) รัน `pg_upgrade` พร้อมระบุ old/new datadir และ bindir
ข) ติดตั้ง PostgreSQL เวอร์ชันใหม่คู่ขนานกับเวอร์ชันเดิม
ค) หยุด PostgreSQL ทั้งสองเวอร์ชัน
ง) initdb สร้าง cluster ใหม่สำหรับเวอร์ชันใหม่
จ) เริ่ม cluster ใหม่และรัน analyze

<details>
<summary>เฉลย</summary>

ลำดับที่ถูกต้องคือ: **ข → ค → ง → ก → จ**

1. (ข) ติดตั้ง PostgreSQL เวอร์ชันใหม่คู่ขนานกับเวอร์ชันเดิม
2. (ค) หยุด PostgreSQL ทั้งสองเวอร์ชันก่อนเริ่มกระบวนการ
3. (ง) initdb สร้าง cluster ใหม่สำหรับเวอร์ชันใหม่ (ยังไม่มีข้อมูล)
4. (ก) รัน `pg_upgrade` เพื่อแปลง/ย้ายข้อมูลจาก cluster เก่าไปยัง cluster ใหม่
5. (จ) เริ่ม cluster ใหม่และรัน analyze script ตามที่ pg_upgrade แนะนำ

และควร**สำรองข้อมูลก่อนเริ่มกระบวนการทั้งหมดเสมอ** แม้จะใช้ `pg_upgrade` ซึ่งค่อนข้างปลอดภัยก็ตาม
</details>

---

## บทถัดไป

เมื่อคุณติดตั้ง PostgreSQL สำเร็จและเชื่อมต่อได้แล้ว ขั้นตอนต่อไปคือการทำความรู้จักกับเครื่องมือที่ใช้ทำงานร่วมกับ PostgreSQL ในชีวิตประจำวัน ทั้ง `psql` แบบเจาะลึก, pgAdmin, DBeaver, และเครื่องมือ command line อื่น ๆ ที่จำเป็น

**บทถัดไป**: [`part-003-tools.md`](./part-003-tools.md) — เครื่องมือสำหรับทำงานกับ PostgreSQL (psql, pgAdmin, DBeaver และอื่น ๆ)
