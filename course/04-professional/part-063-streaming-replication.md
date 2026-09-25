# Streaming Replication (Physical Replication)

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับมืออาชีพ | Part 063

---

## เป้าหมายการเรียนรู้

เมื่อเรียนจบบทนี้ ผู้เรียนจะสามารถ:

- อธิบายหลักการทำงานของ **Streaming Replication** และความแตกต่างจาก Logical Replication ได้อย่างถูกต้อง
- ออกแบบสถาปัตยกรรม Primary-Standby ได้หลายรูปแบบ ทั้ง single standby, cascading replication และ multiple standby
- ตั้งค่า Primary server ให้พร้อมรองรับการเชื่อมต่อจาก Standby ผ่าน `wal_level`, `max_wal_senders`, และ replication user
- กำหนดค่า `pg_hba.conf` เพื่ออนุญาตการเชื่อมต่อแบบ replication อย่างปลอดภัย
- ใช้ `pg_basebackup` พร้อม flag `-R` เพื่อสร้าง Standby server แบบอัตโนมัติ (ได้ `standby.signal` และ `primary_conninfo` โดยไม่ต้องตั้งค่าเอง)
- เปรียบเทียบและเลือกใช้ Synchronous กับ Asynchronous Replication ให้เหมาะกับ business requirement
- ตรวจสอบและวัด Replication Lag ด้วย `pg_stat_replication` และ `pg_stat_wal_receiver`
- เปิดใช้งาน Hot Standby เพื่อรองรับ read-only query บน Standby ระหว่าง replication ทำงานอยู่
- ทำ Failover เบื้องต้นด้วย `pg_promote()` หรือ trigger file เพื่อ promote Standby เป็น Primary ใหม่
- ออกแบบและจัดทำเอกสารขั้นตอนการตั้งค่า Primary-Standby replication แบบเต็มรูปแบบสำหรับระบบจริง

---

## บริบทของบทนี้: การจำลอง Primary + Standby บนสภาพแวดล้อมเดียว

ในสภาพแวดล้อมการฝึกฝนจริง เรามักไม่มีเซิร์ฟเวอร์แยกกันสองเครื่อง บทนี้จะอธิบายทุกขั้นตอนราวกับว่าเรามี **สองเซิร์ฟเวอร์จริง** คือ

| Role | Hostname (ตัวอย่าง) | IP (ตัวอย่าง) | Port |
|---|---|---|---|
| Primary | `pg-primary01` | `192.168.10.11` | 5432 |
| Standby | `pg-standby01` | `192.168.10.12` | 5432 |

ในทางปฏิบัติ ผู้เรียนสามารถจำลองสถานการณ์นี้บนเครื่องเดียวได้โดยการรัน PostgreSQL instance สองตัวคนละ port และคนละ data directory (เช่น `/var/lib/postgresql/16/primary` และ `/var/lib/postgresql/16/standby`) แต่แนวคิด คำสั่ง และไฟล์ config ที่แสดงในบทนี้เขียนขึ้นสำหรับกรณีใช้งานจริงบนเครื่องแยกกัน เพื่อให้ผู้เรียนนำไปใช้ใน production ได้ทันที

---

## Step 621: Streaming Replication คืออะไร

### 621.1 นิยามและหลักการทำงาน

**Streaming Replication** คือกลไกการทำสำเนาข้อมูลระดับ **physical (block-level)** ที่ PostgreSQL ใช้ในการส่ง **WAL (Write-Ahead Log) records** จากเซิร์ฟเวอร์ **Primary** ไปยังเซิร์ฟเวอร์ **Standby** แบบ **real-time** (หรือใกล้เคียง real-time มากที่สุด) ผ่าน network connection โดยตรง แทนที่จะต้องรอให้ WAL segment เต็มไฟล์ (16MB โดย default) แล้วค่อย archive/copy ไปทีหลังแบบ log shipping แบบดั้งเดิม

หลักการพื้นฐานคือ:

1. ทุกการเปลี่ยนแปลงข้อมูลใน PostgreSQL (INSERT, UPDATE, DELETE, DDL) จะถูกบันทึกลง **WAL** ก่อนเสมอ (Write-Ahead Logging) เพื่อความทนทานของข้อมูล (durability)
2. Primary จะมี process ชื่อ **`walsender`** ทำหน้าที่อ่าน WAL record จาก WAL buffer/WAL file แล้ว **สตรีม** ส่งไปยัง Standby ทันทีที่มีการ commit หรือ flush WAL
3. Standby จะมี process ชื่อ **`walreceiver`** ทำหน้าที่รับ WAL stream จาก Primary แล้วเขียนลง WAL file ของตัวเอง
4. จากนั้น process **`startup`** (recovery process) บน Standby จะ replay WAL record เหล่านั้นเข้าไปใน data files เพื่อให้ข้อมูลบน Standby ตรงกับ Primary

```
┌─────────────────────┐                              ┌─────────────────────┐
│      PRIMARY         │                              │      STANDBY         │
│                       │                              │                       │
│  Client Write (COMMIT)│                              │                       │
│         │             │                              │                       │
│         ▼             │                              │                       │
│  ┌────────────────┐   │                              │  ┌────────────────┐   │
│  │  WAL Buffer     │   │                              │  │  WAL Receiver   │   │
│  │  (in memory)    │   │                              │  │  (walreceiver)  │   │
│  └───────┬─────────┘   │                              │  └───────┬─────────┘   │
│          ▼             │      Streaming WAL           │          ▼             │
│  ┌────────────────┐   │   (TCP replication protocol) │  ┌────────────────┐   │
│  │  WAL Files      │───┼──────────────────────────────▶│  │  WAL Files      │   │
│  │  pg_wal/        │   │      (walsender process)     │  │  pg_wal/        │   │
│  └────────────────┘   │                              │  └───────┬─────────┘   │
│                       │                              │          ▼             │
│                       │                              │  ┌────────────────┐   │
│                       │                              │  │  Recovery       │   │
│                       │                              │  │  (startup proc) │   │
│                       │                              │  │  REDO / Replay  │   │
│                       │                              │  └───────┬─────────┘   │
│                       │                              │          ▼             │
│  ┌────────────────┐   │                              │  ┌────────────────┐   │
│  │  Data Files     │   │                              │  │  Data Files     │   │
│  │  base/          │   │                              │  │  base/          │   │
│  └────────────────┘   │                              │  └────────────────┘   │
└─────────────────────┘                              └─────────────────────┘
```

### 621.2 ทำไมต้องใช้ Streaming Replication

| ประโยชน์ | รายละเอียด |
|---|---|
| **High Availability (HA)** | เมื่อ Primary ล่ม สามารถ promote Standby ขึ้นมาเป็น Primary ใหม่ได้อย่างรวดเร็ว ลด downtime |
| **Disaster Recovery (DR)** | ตั้ง Standby ไว้คนละ datacenter/region เพื่อป้องกันความเสียหายระดับ site |
| **Read Scalability** | Standby ที่เปิด Hot Standby สามารถรับ read-only query ได้ ช่วยกระจายโหลดจาก Primary |
| **Backup Offloading** | สามารถรัน `pg_dump` หรือ backup process บน Standby แทนที่จะรบกวน Primary |
| **Near Real-time Lag** | ต่างจาก log shipping (file-based) ที่ lag เป็นนาทีหรือชั่วโมง Streaming Replication มักมี lag เพียงมิลลิวินาทีถึงวินาที |

### 621.3 เปรียบเทียบกับ Logical Replication (ทบทวนล่วงหน้า)

Streaming Replication (Physical Replication) ที่เราเรียนในบทนี้ ทำงานที่ระดับ **byte-for-byte block copy** ของทั้ง cluster ในขณะที่ **Logical Replication** (ซึ่งจะเรียนใน Part 064) ทำงานที่ระดับ **logical change (row-level)** ผ่าน publication/subscription และสามารถเลือก replicate เฉพาะบางตารางได้

| คุณสมบัติ | Physical (Streaming) Replication | Logical Replication |
|---|---|---|
| ระดับการ replicate | ทั้ง database cluster (block-level) | เลือกเฉพาะ table/database (row-level) |
| PostgreSQL version | Primary/Standby ควรเป็น version เดียวกัน | รองรับต่าง version ได้ในระดับหนึ่ง |
| Standby query ได้หรือไม่ | ได้ (read-only ผ่าน Hot Standby) | ได้ (read-write บน subscriber แต่ปกติใช้ read) |
| Schema changes | Replicate อัตโนมัติ (DDL ก็ replicate เพราะเป็น block-level) | DDL ไม่ replicate อัตโนมัติ (ต้องจัดการเอง) |
| Use case หลัก | HA/DR, failover, read replica | Selective sync, migration, multi-master แบบจำกัด, ETL |

> **หมายเหตุสำคัญ:** ในบทนี้เราจะโฟกัสเฉพาะ Physical Streaming Replication เท่านั้น ส่วน Logical Replication จะเจาะลึกในบทถัดไป (Part 064)

### 621.4 กลไกเบื้องหลัง: Replication Protocol

Streaming Replication ใช้ **Replication Protocol** ที่วิ่งอยู่บน connection เดียวกับ libpq (port 5432 เดียวกับที่ client ใช้เชื่อมต่อฐานข้อมูลปกติ) แต่ระบุ parameter พิเศษคือ `replication=true` (หรือ `replication=database` สำหรับ logical) ใน connection string

Standby จะส่งคำสั่งพิเศษเช่น `START_REPLICATION` ไปยัง Primary ผ่าน replication protocol นี้ เพื่อขอเริ่มรับ WAL stream จากตำแหน่ง LSN (Log Sequence Number) ที่กำหนด

```sql
-- ตัวอย่างคำสั่งระดับ protocol ที่ walreceiver ส่งไปยัง walsender (ภายใน ไม่ต้องเรียกเอง)
IDENTIFY_SYSTEM;
START_REPLICATION SLOT "standby01_slot" PHYSICAL 0/3000000 TIMELINE 1;
```

---

## Step 622: สถาปัตยกรรม Primary-Standby เบื้องต้น

### 622.1 Topology แบบที่ 1 — Single Standby (พื้นฐานที่สุด)

รูปแบบพื้นฐานที่สุดคือ Primary หนึ่งตัว ส่ง WAL ไปยัง Standby หนึ่งตัว เหมาะสำหรับระบบขนาดเล็กถึงกลางที่ต้องการ HA พื้นฐาน

```
┌───────────────┐         WAL Streaming          ┌───────────────┐
│   PRIMARY     │ ──────────────────────────────▶│   STANDBY      │
│  (Read/Write)  │                                │  (Read-Only)   │
└───────────────┘                                └───────────────┘
```

**ข้อดี:** ตั้งค่าง่าย ดูแลรักษาง่าย
**ข้อเสีย:** รองรับ read scalability ได้จำกัด (Standby เดียว)

### 622.2 Topology แบบที่ 2 — Cascading Replication

**Cascading Replication** คือการให้ Standby ตัวหนึ่งทำหน้าที่เป็น "walsender" ส่งต่อ WAL ไปยัง Standby ตัวอื่นอีกที แทนที่ Standby ทุกตัวจะต้องเชื่อมต่อกับ Primary โดยตรง วิธีนี้ช่วยลดภาระ network และ CPU บน Primary เมื่อมี Standby จำนวนมาก โดยเฉพาะเมื่อ Standby อยู่คนละ datacenter/region

```
┌───────────────┐         WAL Streaming          ┌───────────────┐
│   PRIMARY     │ ──────────────────────────────▶│  STANDBY-1     │
│  (Datacenter A)│                                │ (Datacenter A) │
└───────────────┘                                └───────┬───────┘
                                                            │ WAL Streaming
                                                            │ (Cascading)
                                                            ▼
                                                   ┌───────────────┐
                                                   │  STANDBY-2     │
                                                   │ (Datacenter B) │
                                                   └───────┬───────┘
                                                            │ WAL Streaming
                                                            ▼
                                                   ┌───────────────┐
                                                   │  STANDBY-3     │
                                                   │ (Datacenter B) │
                                                   └───────────────┘
```

**ข้อดี:**
- ลดภาระ network/CPU บน Primary เมื่อมี Standby จำนวนมาก
- เหมาะสำหรับ multi-region deployment (Standby-1 อยู่ region เดียวกับ Primary ทำหน้าที่ relay ไปยัง region อื่น)

**ข้อควรระวัง:** ถ้า Standby-1 (ตัว relay) ล่ม Standby-2 และ Standby-3 จะขาดการเชื่อมต่อ WAL ทันที (ต้องมี failover plan สำหรับ relay node ด้วย)

### 622.3 Topology แบบที่ 3 — Multiple Standby (Fan-out)

Primary ส่ง WAL ไปยัง Standby หลายตัวพร้อมกันโดยตรง (ไม่ผ่าน cascading) เหมาะสำหรับกรณีที่ต้องการทั้ง HA (Standby ตัวหนึ่งสำหรับ failover) และ read scalability (Standby ตัวอื่นสำหรับกระจาย read load)

```
                              ┌───────────────┐
                     ┌───────▶│  STANDBY-1     │  (Synchronous - สำหรับ failover)
                     │        │  (same DC)     │
┌───────────────┐    │        └───────────────┘
│   PRIMARY     │────┤
│  (Read/Write)  │    │        ┌───────────────┐
└───────────────┘    ├───────▶│  STANDBY-2     │  (Asynchronous - สำหรับ Read Replica)
                     │        │  (same DC)     │
                     │        └───────────────┘
                     │
                     │        ┌───────────────┐
                     └───────▶│  STANDBY-3     │  (Asynchronous - สำหรับ DR ต่าง Region)
                              │  (remote DC)    │
                              └───────────────┘
```

**ข้อดี:** ยืดหยุ่นสูง สามารถผสมผสาน synchronous (สำหรับ HA) กับ asynchronous (สำหรับ read scaling/DR) ได้ในระบบเดียว
**ข้อเสีย:** ใช้ทรัพยากร network/CPU บน Primary มากกว่า (walsender process แยกต่างหากต่อ Standby หนึ่งตัว)

### 622.4 การเลือก Topology ให้เหมาะกับงาน

| สถานการณ์ | Topology ที่แนะนำ |
|---|---|
| ระบบเล็ก ต้องการ HA พื้นฐาน | Single Standby |
| ระบบใหญ่ มี Standby หลายตัวคนละ region | Cascading Replication |
| ต้องการทั้ง HA และ Read Scaling พร้อมกัน | Multiple Standby (Fan-out) |
| ต้องการ Zero Data Loss (RPO=0) | Single/Multiple Standby + Synchronous mode |

---

## Step 623: การตั้งค่า Primary

การเตรียม Primary server ให้พร้อมรองรับ Streaming Replication ต้องตั้งค่า 3 ส่วนหลัก: `wal_level`, `max_wal_senders` (และ parameter ที่เกี่ยวข้อง) และสร้าง **replication user**

### 623.1 wal_level

Parameter `wal_level` กำหนดปริมาณข้อมูลที่จะถูกเขียนลง WAL โดยมีค่าที่เป็นไปได้ 3 ค่าใน PostgreSQL 16/17:

| ค่า | คำอธิบาย |
|---|---|
| `minimal` | เขียน WAL น้อยที่สุด ใช้ได้เฉพาะกรณีไม่ต้องการ replication หรือ PITR เลย |
| `replica` | **(default)** เขียน WAL เพียงพอสำหรับ Streaming Replication และ PITR (point-in-time recovery) — **ใช้ค่านี้สำหรับ physical replication** |
| `logical` | เขียน WAL เพิ่มเติมสำหรับรองรับ logical decoding (จำเป็นสำหรับ Logical Replication ใน Part 064) |

แก้ไขไฟล์ `postgresql.conf` บน Primary:

```conf
# /etc/postgresql/16/main/postgresql.conf (Primary)

# ---------------------------------------
# WRITE-AHEAD LOG (WAL) SETTINGS
# ---------------------------------------
wal_level = replica              # ค่า default ตั้งแต่ PostgreSQL 9.6 เป็นต้นมา
                                  # replica = เพียงพอสำหรับ streaming replication + PITR

# ---------------------------------------
# REPLICATION SETTINGS
# ---------------------------------------
max_wal_senders = 10             # จำนวน walsender process สูงสุดที่รันพร้อมกันได้
                                  # ควรมากกว่าจำนวน standby ที่คาดว่าจะเชื่อมต่อจริง
                                  # (รวม standby สำหรับ pg_basebackup ชั่วคราวด้วย)

max_replication_slots = 10       # จำนวน replication slot สูงสุด
                                  # ควร >= max_wal_senders เพื่อรองรับทุก standby ที่ใช้ slot

wal_keep_size = 1GB              # ปริมาณ WAL ที่เก็บไว้ในกรณีไม่ได้ใช้ replication slot
                                  # (หน่วยเป็น MB ใน PostgreSQL 13 ขึ้นไป ใช้ wal_keep_size แทน wal_keep_segments)

hot_standby = on                 # อนุญาตให้ standby รับ read-only query ได้ (มีผลเมื่อฝั่งนั้นเป็น standby)
                                  # ต้องตั้งทั้งบน primary และ standby (จะละเอียดใน Step 628)

# ---------------------------------------
# ARCHIVING (แนะนำให้เปิดควบคู่ เพื่อความปลอดภัยเพิ่มเติม)
# ---------------------------------------
archive_mode = on
archive_command = 'test ! -f /var/lib/postgresql/wal_archive/%f && cp %p /var/lib/postgresql/wal_archive/%f'
```

> **หมายเหตุ:** `max_wal_senders`, `max_replication_slots` และ `wal_level` เป็น parameter ที่ต้อง **restart** PostgreSQL หลังแก้ไข (ไม่ใช่แค่ `reload`) เนื่องจากเป็น parameter ประเภท `postmaster` context

ตรวจสอบ context ของ parameter ได้ด้วย:

```sql
SELECT name, setting, context
FROM pg_settings
WHERE name IN ('wal_level', 'max_wal_senders', 'max_replication_slots', 'hot_standby');
```

ผลลัพธ์ตัวอย่าง:

```
        name         | setting |  context
----------------------+---------+------------
 wal_level            | replica | postmaster
 max_wal_senders      | 10      | postmaster
 max_replication_slots| 10      | postmaster
 hot_standby          | on      | postmaster
(4 rows)
```

### 623.2 การสร้าง Replication User

ควรสร้าง user เฉพาะสำหรับ replication แยกจาก superuser หรือ application user เพื่อความปลอดภัยตามหลัก least privilege:

```sql
-- เชื่อมต่อ Primary ด้วย superuser แล้วสร้าง replication role
CREATE ROLE replicator WITH
    REPLICATION
    LOGIN
    ENCRYPTED PASSWORD 'S3cur3R3pl1c4t10nP@ss!';
```

**คำอธิบาย attribute:**

| Attribute | ความหมาย |
|---|---|
| `REPLICATION` | อนุญาตให้ role นี้เชื่อมต่อผ่าน replication protocol ได้ (ใช้สำหรับ `walreceiver`, `pg_basebackup`, `pg_receivewal` เป็นต้น) |
| `LOGIN` | อนุญาตให้ login ได้ (จำเป็น เพราะ role ต้องเชื่อมต่อจริง) |
| `ENCRYPTED PASSWORD` | เก็บรหัสผ่านแบบ hash (SCRAM-SHA-256 โดย default ใน PostgreSQL 14 ขึ้นไป) |

ตรวจสอบว่า role ถูกสร้างสำเร็จ:

```sql
SELECT rolname, rolreplication, rolcanlogin
FROM pg_roles
WHERE rolname = 'replicator';
```

```
  rolname   | rolreplication | rolcanlogin
-------------+----------------+-------------
 replicator  | t              | t
(1 row)
```

> **แนวปฏิบัติที่ดี (Best Practice):** ไม่ควรใช้ superuser (`postgres`) สำหรับ replication connection ในระบบ production เพราะหากรหัสผ่านรั่วไหล ผู้ไม่หวังดีจะสามารถอ่านข้อมูล WAL ทั้งหมด (ซึ่งรวมถึงข้อมูลทุกตารางในระบบ) ได้ทันที การใช้ role เฉพาะ `REPLICATION` ช่วยจำกัดขอบเขตความเสียหาย

### 623.3 ตรวจสอบ walsender process หลัง restart

หลัง restart PostgreSQL ให้ตรวจสอบว่าการตั้งค่าถูกโหลดเข้าไปแล้ว:

```bash
sudo systemctl restart postgresql@16-main
```

```sql
SHOW wal_level;
SHOW max_wal_senders;
```

---

## Step 624: pg_hba.conf สำหรับ Replication Connection

### 624.1 หลักการของ pg_hba.conf กับ Replication

ไฟล์ `pg_hba.conf` (host-based authentication) ควบคุมว่า client (หรือในกรณีนี้คือ Standby server) ตัวใดสามารถเชื่อมต่อเข้ามาได้บ้าง โดยการเชื่อมต่อแบบ replication จะใช้ **database field พิเศษ** ชื่อ `replication` (ไม่ใช่ชื่อฐานข้อมูลจริง) เพื่อแยกแยะจาก connection ปกติ

รูปแบบทั่วไปของบรรทัดใน `pg_hba.conf`:

```
# TYPE      DATABASE        USER            ADDRESS                 METHOD
```

### 624.2 การกำหนดค่าใน pg_hba.conf บน Primary

```conf
# /etc/postgresql/16/main/pg_hba.conf (Primary)

# ---------------------------------------------------------------------------
# TYPE    DATABASE        USER            ADDRESS                  METHOD
# ---------------------------------------------------------------------------

# Local connections (ปกติ)
local   all             all                                        peer
host    all             all             127.0.0.1/32               scram-sha-256
host    all             all             ::1/128                    scram-sha-256

# ---------------------------------------------------------------------------
# REPLICATION CONNECTIONS
# ---------------------------------------------------------------------------
# อนุญาตให้ replicator user เชื่อมต่อผ่าน replication protocol
# จาก standby server ที่มี IP ระบุไว้ชัดเจนเท่านั้น (ไม่ใช้ 0.0.0.0/0)

host    replication     replicator      192.168.10.12/32           scram-sha-256
host    replication     replicator      192.168.10.13/32           scram-sha-256

# สำหรับ cascading replication (standby-2 เชื่อมกับ standby-1 แทน primary)
# ต้องเปิดในไฟล์ pg_hba.conf ของ standby-1 ด้วย ไม่ใช่แค่ primary

# กรณีใช้ subnet ทั้งหมดในวง replication (เหมาะกับ internal network ที่ปลอดภัยแล้ว)
# host  replication     replicator      192.168.10.0/24            scram-sha-256
```

**คำอธิบายแต่ละคอลัมน์:**

| คอลัมน์ | ค่า | ความหมาย |
|---|---|---|
| TYPE | `host` | เชื่อมต่อผ่าน TCP/IP (ใช้ `hostssl` หากต้องการบังคับ SSL) |
| DATABASE | `replication` | keyword พิเศษ หมายถึง replication connection เท่านั้น ไม่ใช่ชื่อฐานข้อมูล |
| USER | `replicator` | ชื่อ role ที่สร้างไว้ใน Step 623 |
| ADDRESS | `192.168.10.12/32` | IP address ของ standby server (ระบุให้เจาะจงที่สุดเพื่อความปลอดภัย) |
| METHOD | `scram-sha-256` | วิธีการยืนยันตัวตน (แนะนำสำหรับ PostgreSQL 14+ แทน `md5` ที่เก่ากว่า) |

### 624.3 การใช้ SSL/TLS สำหรับ Replication Connection (Production-grade)

สำหรับระบบ production ที่ standby อยู่คนละ network segment หรือคนละ datacenter ควรบังคับใช้ SSL:

```conf
# บังคับ SSL สำหรับ replication connection จากภายนอก network เดียวกัน
hostssl replication     replicator      203.0.113.50/32            scram-sha-256 clientcert=verify-full
```

### 624.4 Reload การตั้งค่า

ต่างจาก `postgresql.conf` บางส่วน การแก้ไข `pg_hba.conf` ไม่จำเป็นต้อง restart เพียง reload ก็เพียงพอ:

```bash
sudo systemctl reload postgresql@16-main
# หรือ
sudo -u postgres psql -c "SELECT pg_reload_conf();"
```

ตรวจสอบว่า config ถูกโหลดถูกต้อง (ไม่มี syntax error):

```sql
SELECT * FROM pg_hba_file_rules WHERE error IS NOT NULL;
```

ถ้าไม่มีแถวใดถูกคืนกลับมา แปลว่าไม่มี error ใน `pg_hba.conf`

```sql
-- ดูรายการ rule ทั้งหมดที่เกี่ยวข้องกับ replication
SELECT line_number, type, database, user_name, address, auth_method
FROM pg_hba_file_rules
WHERE database @> ARRAY['replication'];
```

```
 line_number | type |   database    |  user_name  |    address     | auth_method
-------------+------+---------------+-------------+-----------------+-------------
          12 | host | {replication} | {replicator}| 192.168.10.12/32| scram-sha-256
          13 | host | {replication} | {replicator}| 192.168.10.13/32| scram-sha-256
(2 rows)
```

---

## Step 625: การตั้งค่า Standby ด้วย pg_basebackup

### 625.1 pg_basebackup คืออะไร

**`pg_basebackup`** เป็นเครื่องมือ command-line ที่มากับ PostgreSQL ใช้สำหรับสร้าง **base backup** (สำเนาข้อมูลทั้งหมดของ cluster ณ จุดเวลาหนึ่ง) จาก Primary (หรือ Standby อื่นที่เปิด replication ไว้) เพื่อนำไปตั้งเป็น Standby server ใหม่

ข้อดีของ `pg_basebackup` คือทำงานผ่าน replication protocol โดยตรง ไม่ต้องหยุด Primary หรือใช้ filesystem snapshot ที่ซับซ้อน

### 625.2 เตรียม Standby Server ก่อนรัน pg_basebackup

บน Standby server ต้องติดตั้ง PostgreSQL version เดียวกับ Primary (สำคัญมาก — version ต้องตรงกันสำหรับ physical replication) และต้องแน่ใจว่า data directory เป้าหมายว่างเปล่า:

```bash
# บน pg-standby01
sudo systemctl stop postgresql@16-main
sudo rm -rf /var/lib/postgresql/16/main/*
```

### 625.3 รัน pg_basebackup พร้อม -R Flag

```bash
# รันคำสั่งนี้บน pg-standby01 (เชื่อมต่อไปยัง pg-primary01)
sudo -u postgres pg_basebackup \
    -h 192.168.10.11 \
    -p 5432 \
    -U replicator \
    -D /var/lib/postgresql/16/main \
    -Fp \
    -Xs \
    -P \
    -R \
    --checkpoint=fast
```

**คำอธิบายแต่ละ flag อย่างละเอียด:**

| Flag | ความหมาย |
|---|---|
| `-h 192.168.10.11` | host ของ Primary server |
| `-p 5432` | port ของ Primary |
| `-U replicator` | replication user ที่สร้างไว้ใน Step 623 |
| `-D /var/lib/postgresql/16/main` | ปลายทางที่จะเขียน data directory ของ standby |
| `-Fp` | รูปแบบ output เป็น `plain` (copy ไฟล์ตรงๆ ไม่ใช่ tar) |
| `-Xs` | (`--wal-method=stream`) สตรีม WAL ระหว่างทำ backup พร้อมกัน เพื่อไม่ให้ WAL หายไประหว่าง backup ใช้เวลานาน |
| `-P` | (`--progress`) แสดง progress bar ระหว่าง backup |
| **`-R`** | **(`--write-recovery-conf`) — flag สำคัญที่สุดในบทนี้** สร้างไฟล์ `standby.signal` และเขียน parameter `primary_conninfo` ลงใน `postgresql.auto.conf` ให้อัตโนมัติ |
| `--checkpoint=fast` | บังคับให้ Primary ทำ checkpoint ทันทีก่อนเริ่ม backup (ลดเวลารอ แต่เพิ่ม I/O spike ชั่วขณะบน Primary) |

จะปรากฏ progress ระหว่างรัน:

```
36123/36123 kB (100%), 1/1 tablespace
```

### 625.4 สิ่งที่ -R สร้างให้อัตโนมัติ

เมื่อใช้ `-R` แล้ว `pg_basebackup` จะสร้างสองสิ่งนี้ให้อัตโนมัติ ซึ่งในเวอร์ชันก่อน PostgreSQL 12 ต้องสร้างเองด้วยมือ:

**1. ไฟล์ `standby.signal`** (ไฟล์เปล่า ไม่มีเนื้อหา) วางอยู่ใน data directory:

```bash
ls -la /var/lib/postgresql/16/main/standby.signal
# -rw------- 1 postgres postgres 0 Sep 25 10:15 standby.signal
```

ไฟล์นี้เป็น **สัญญาณบอก PostgreSQL** ว่าเมื่อ start ขึ้นมา ให้เข้าสู่ **standby mode** แทนที่จะเป็น primary mode ปกติ (ต่างจาก PostgreSQL 11 ลงไปที่ใช้ไฟล์ `recovery.conf` แบบรวมทุกอย่างไว้ในไฟล์เดียว)

**2. Parameter `primary_conninfo`** ถูกเขียนลงใน `postgresql.auto.conf`:

```bash
cat /var/lib/postgresql/16/main/postgresql.auto.conf
```

```conf
# Do not edit this file manually!
# It will be overwritten by the ALTER SYSTEM command.
primary_conninfo = 'user=replicator password=S3cur3R3pl1c4t10nP@ss! host=192.168.10.11 port=5432 sslmode=prefer sslcompression=0 gssencmode=prefer krbsrvname=postgres target_session_attrs=any'
```

> **คำเตือนด้านความปลอดภัย:** `postgresql.auto.conf` จะเก็บรหัสผ่านแบบ plain text หากใช้ password authentication ในระบบ production ที่ต้องการความปลอดภัยสูง ควรพิจารณาใช้ `.pgpass` file หรือ certificate-based authentication (`clientcert=verify-full`) แทนการฝังรหัสผ่านในไฟล์ config

### 625.5 (ทางเลือก) ใช้ .pgpass แทนการฝังรหัสผ่าน

```bash
# สร้างไฟล์ .pgpass บน standby (ก่อนรัน pg_basebackup)
echo "192.168.10.11:5432:replication:replicator:S3cur3R3pl1c4t10nP@ss!" > ~/.pgpass
chmod 600 ~/.pgpass
```

เมื่อมี `.pgpass` แล้ว รัน `pg_basebackup` โดยไม่ต้องใส่รหัสผ่านในคำสั่ง และ `primary_conninfo` ที่ถูกสร้างจะไม่มีรหัสผ่านฝังอยู่ (ใช้ `.pgpass` แทนตอน connect จริง)

### 625.6 เพิ่มค่าที่แนะนำเพิ่มเติมบน Standby

หลังจาก `pg_basebackup -R` สร้างไฟล์พื้นฐานให้แล้ว แนะนำให้ตรวจสอบและเพิ่มเติม parameter ต่อไปนี้ใน `postgresql.conf` ของ Standby:

```conf
# /etc/postgresql/16/main/postgresql.conf (Standby)

hot_standby = on                         # อนุญาต read-only query (Step 628)
primary_slot_name = 'standby01_slot'     # ใช้ replication slot เพื่อป้องกัน WAL ถูกลบก่อน standby รับครบ
```

หากต้องการใช้ replication slot (แนะนำสำหรับ production) ต้องสร้าง slot บน Primary ก่อน:

```sql
-- รันบน Primary
SELECT pg_create_physical_replication_slot('standby01_slot');
```

```sql
SELECT slot_name, slot_type, active
FROM pg_replication_slots;
```

```
   slot_name    | slot_type | active
-----------------+-----------+--------
 standby01_slot  | physical  | f
(1 row)
```

(`active` จะเปลี่ยนเป็น `t` เมื่อ standby เชื่อมต่อและเริ่ม stream แล้ว)

### 625.7 Start Standby Server

```bash
sudo systemctl start postgresql@16-main
```

ตรวจสอบ log บน standby:

```bash
sudo tail -f /var/log/postgresql/postgresql-16-main.log
```

```
2026-09-25 10:20:01.123 UTC [1842] LOG:  starting PostgreSQL 16.4 on x86_64-pc-linux-gnu
2026-09-25 10:20:01.145 UTC [1842] LOG:  listening on IPv4 address "0.0.0.0", port 5432
2026-09-25 10:20:01.201 UTC [1845] LOG:  database system was interrupted; last known up at 2026-09-25 10:15:00 UTC
2026-09-25 10:20:01.350 UTC [1845] LOG:  entering standby mode
2026-09-25 10:20:01.412 UTC [1845] LOG:  redo starts at 0/3000028
2026-09-25 10:20:01.489 UTC [1848] LOG:  started streaming WAL from primary at 0/3000000 on timeline 1
2026-09-25 10:20:01.501 UTC [1845] LOG:  consistent recovery state reached at 0/30000A0
2026-09-25 10:20:01.502 UTC [1842] LOG:  database system is ready to accept read-only connections
```

สังเกตบรรทัดสำคัญ: `entering standby mode`, `started streaming WAL from primary`, และ `database system is ready to accept read-only connections` แสดงว่า Standby เริ่มทำงานสำเร็จ

---

## Step 626: Synchronous เทียบกับ Asynchronous Replication

### 626.1 ความแตกต่างพื้นฐาน

| โหมด | คำอธิบาย |
|---|---|
| **Asynchronous** (default) | Primary commit transaction ทันทีที่ WAL ถูกเขียนลง disk ของตัวเอง โดย**ไม่รอ** ให้ Standby ยืนยันว่าได้รับ WAL แล้ว — เร็วที่สุด แต่เสี่ยงข้อมูลสูญหายหากเกิด failover ระหว่างที่ WAL ยังไม่ถูกส่งไปถึง Standby |
| **Synchronous** | Primary จะ**รอ**ให้ Standby (อย่างน้อยหนึ่งตัวตามที่กำหนด) ยืนยันว่าได้รับ (หรือเขียน/apply) WAL แล้ว ก่อนจะแจ้ง client ว่า commit สำเร็จ — ปลอดภัยกว่า (RPO=0) แต่ latency สูงขึ้น |

### 626.2 Trade-off: ความเร็ว vs ความปลอดภัยของข้อมูล

```
Asynchronous:
  Client ──COMMIT──▶ Primary ──เขียน WAL local──▶ ตอบกลับ client ทันที (OK)
                          │
                          └──ส่ง WAL ไป Standby (ไม่รอผล)──▶ Standby (อาจ lag)

  ความเสี่ยง: หาก Primary ล่มก่อน WAL ไปถึง Standby → transaction ล่าสุดอาจหายไป
  RPO (Recovery Point Objective) > 0

Synchronous:
  Client ──COMMIT──▶ Primary ──เขียน WAL local──▶ รอ ACK จาก Standby ──▶ ตอบกลับ client (OK)
                                                          │
                                                          ▼
                                                    Standby ยืนยันรับ WAL แล้ว

  ความเสี่ยง: latency เพิ่มขึ้นตาม network RTT ระหว่าง Primary-Standby
  RPO = 0 (ไม่มีข้อมูลสูญหายหาก failover)
```

### 626.3 การตั้งค่า Synchronous Replication

**ขั้นตอนที่ 1:** กำหนด `application_name` ให้ Standby แต่ละตัวใน `primary_conninfo` (บน Standby)

```conf
# postgresql.auto.conf (Standby) - เพิ่ม application_name เข้าไปใน primary_conninfo
primary_conninfo = 'user=replicator password=S3cur3R3pl1c4t10nP@ss! host=192.168.10.11 port=5432 application_name=standby01'
```

**ขั้นตอนที่ 2:** กำหนด `synchronous_standby_names` บน Primary

```conf
# /etc/postgresql/16/main/postgresql.conf (Primary)

synchronous_standby_names = 'standby01'
```

รูปแบบของ `synchronous_standby_names` มีหลายแบบตามความต้องการ:

```conf
# แบบที่ 1: ระบุ standby ตัวเดียวแบบเจาะจงชื่อ (FIRST เป็นค่า default ถ้าไม่ระบุ)
synchronous_standby_names = 'standby01'

# แบบที่ 2: FIRST N - รอ ACK จาก N ตัวแรกในลิสต์ที่พร้อมใช้งาน (priority-based)
synchronous_standby_names = 'FIRST 1 (standby01, standby02, standby03)'

# แบบที่ 3: ANY N - รอ ACK จาก N ตัวใดก็ได้จากลิสต์ (quorum-based, ยืดหยุ่นกว่า)
synchronous_standby_names = 'ANY 2 (standby01, standby02, standby03)'

# แบบที่ 4: ใช้ wildcard รับ standby ใดๆ ก็ได้ที่เชื่อมต่อเข้ามา (ไม่แนะนำใน production จริงจัง)
synchronous_standby_names = '*'
```

**ขั้นตอนที่ 3:** Reload configuration บน Primary

```bash
sudo -u postgres psql -c "SELECT pg_reload_conf();"
```

### 626.4 synchronous_commit — ระดับความเข้มงวดที่ปรับได้

นอกจาก `synchronous_standby_names` แล้ว ยังมี parameter `synchronous_commit` ที่กำหนดว่า "รอ" ถึงระดับไหนก่อนตอบกลับ client:

| ค่า | ความหมาย |
|---|---|
| `off` | ไม่รอแม้แต่ local WAL flush (เร็วที่สุด แต่เสี่ยงสูญเสียข้อมูลแม้ Primary เพียงตัวเดียวล่ม) |
| `local` | รอเฉพาะ local WAL flush บน Primary (ไม่รอ standby เลย = asynchronous โดยพฤตินัย) |
| `remote_write` | รอจน Standby เขียน WAL ลง OS buffer (ยังไม่ fsync) |
| `on` | **(default)** รอจน Standby fsync WAL ลง disk จริง |
| `remote_apply` | เข้มงวดที่สุด — รอจน Standby **apply/replay** WAL เสร็จ (query บน standby จะเห็นข้อมูลล่าสุดทันที) |

```sql
-- ตั้งค่าระดับ session ได้ด้วย เพื่อความยืดหยุ่นต่อ transaction
SET synchronous_commit = 'remote_apply';

BEGIN;
INSERT INTO orders (customer_id, total_amount) VALUES (1001, 2599.00);
COMMIT;  -- transaction นี้จะรอจน standby apply เสร็จก่อนตอบกลับ
```

> **แนวทางปฏิบัติ:** สำหรับ transaction ที่สำคัญมาก (เช่น การชำระเงิน) สามารถตั้ง `synchronous_commit = remote_apply` เฉพาะ session/transaction นั้น ในขณะที่ transaction ทั่วไป (เช่น log, analytics insert) ใช้ `local` หรือ default เพื่อคง throughput สูงไว้ — เทคนิคนี้เรียกว่า **mixed durability**

### 626.5 ตรวจสอบสถานะ Synchronous Standby

```sql
SELECT application_name, client_addr, state, sync_state, sync_priority
FROM pg_stat_replication;
```

```
 application_name |  client_addr   |   state   | sync_state | sync_priority
-------------------+-----------------+-----------+------------+---------------
 standby01         | 192.168.10.12   | streaming | sync       | 1
 standby02         | 192.168.10.13   | streaming | async      | 0
(2 rows)
```

คอลัมน์ `sync_state` จะบอกว่า standby ตัวนั้นเป็น `sync` (synchronous), `async` (asynchronous), `potential` (ตัวสำรองหาก sync ตัวหลักหลุด), หรือ `quorum` (สำหรับโหมด `ANY N`)

### 626.6 เมื่อ Synchronous Standby ไม่ตอบสนอง

**ข้อควรระวังสำคัญ:** หากตั้งค่า `synchronous_standby_names` แบบเจาะจงตัวเดียวแล้ว Standby ตัวนั้นล่มหรือ network ขาด **Primary จะค้าง (hang) ที่ COMMIT** ของทุก transaction ที่รอ synchronous replication จนกว่า Standby จะกลับมา หรือจนกว่า admin จะเปลี่ยนค่า `synchronous_standby_names` เป็นค่าว่างชั่วคราว

```sql
-- แก้ปัญหาฉุกเฉิน: ปิด synchronous replication ชั่วคราวเมื่อ standby ล่ม
ALTER SYSTEM SET synchronous_standby_names = '';
SELECT pg_reload_conf();
```

ด้วยเหตุนี้ การใช้โหมด `FIRST N` หรือ `ANY N` กับ Standby หลายตัว จึงปลอดภัยกว่าการระบุ Standby ตัวเดียวตายตัว เพราะมี fallback ให้เลือก

---

## Step 627: Replication Lag — การวัดความล่าช้า

### 627.1 ทำไมต้องวัด Lag

แม้ Streaming Replication จะทำงานแบบ real-time แต่ในทางปฏิบัติ **lag** (ความล่าช้า) สามารถเกิดขึ้นได้จากหลายสาเหตุ เช่น network latency, standby I/O ช้า, standby CPU ไม่พอ replay WAL ทัน หรือ query ที่ยาวบน standby ที่ block การ apply WAL (จะกล่าวถึงใน Step 628) การวัด lag อย่างสม่ำเสมอเป็นสิ่งจำเป็นสำหรับ monitoring และ alerting ใน production

### 627.2 วัด Lag จากฝั่ง Primary ด้วย pg_stat_replication

```sql
-- รันบน Primary
SELECT
    application_name,
    client_addr,
    state,
    sync_state,
    pg_current_wal_lsn()                       AS primary_lsn,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn,
    pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) AS replay_lag_bytes,
    write_lag,
    flush_lag,
    replay_lag
FROM pg_stat_replication;
```

ผลลัพธ์ตัวอย่าง:

```
 application_name | client_addr   |   state   | sync_state | primary_lsn |  sent_lsn  | write_lsn  | flush_lsn  | replay_lsn | replay_lag_bytes | write_lag | flush_lag | replay_lag
-------------------+---------------+-----------+------------+-------------+------------+------------+------------+------------+-------------------+-----------+-----------+-------------
 standby01         | 192.168.10.12 | streaming | sync       | 0/5A12340   | 0/5A12340  | 0/5A12340  | 0/5A12340  | 0/5A12210  |               304 | 00:00:00.002134 | 00:00:00.003021 | 00:00:00.015442
 standby02         | 192.168.10.13 | streaming | async      | 0/5A12340   | 0/5A12200  | 0/5A12200  | 0/5A121C0  | 0/5A11F00  |              1088 | 00:00:00.041233 | 00:00:00.089110 | 00:00:00.512300
(2 rows)
```

**คำอธิบายคอลัมน์สำคัญ:**

| คอลัมน์ | ความหมาย |
|---|---|
| `sent_lsn` | ตำแหน่ง WAL ล่าสุดที่ Primary **ส่ง** ไปแล้ว |
| `write_lsn` | ตำแหน่ง WAL ล่าสุดที่ Standby **เขียน** ลง OS (ยังไม่ fsync) |
| `flush_lsn` | ตำแหน่ง WAL ล่าสุดที่ Standby **fsync** ลง disk แล้ว |
| `replay_lsn` | ตำแหน่ง WAL ล่าสุดที่ Standby **apply/replay** เข้าไปใน data files แล้ว (สำคัญที่สุดสำหรับ query correctness) |
| `write_lag` / `flush_lag` / `replay_lag` | **เวลา** (interval) ที่ต่างกันระหว่างแต่ละ stage — วัดเป็นเวลาจริง ไม่ใช่ byte |
| `replay_lag_bytes` | จำนวน **byte** ที่ยังไม่ถูก apply (คำนวณเองด้วย `pg_wal_lsn_diff`) |

### 627.3 วัด Lag จากฝั่ง Standby ด้วย pg_stat_wal_receiver

```sql
-- รันบน Standby
SELECT
    status,
    receive_start_lsn,
    received_lsn,
    received_tli,
    last_msg_send_time,
    last_msg_receipt_time,
    latest_end_lsn,
    latest_end_time,
    slot_name,
    sender_host,
    sender_port
FROM pg_stat_wal_receiver;
```

ผลลัพธ์ตัวอย่าง:

```
-[ RECORD 1 ]----------+------------------------------
status                 | streaming
receive_start_lsn      | 0/3000000
received_lsn           | 0/5A12340
received_tli           | 1
last_msg_send_time     | 2026-09-25 10:45:12.123456+00
last_msg_receipt_time  | 2026-09-25 10:45:12.125891+00
latest_end_lsn          | 0/5A12340
latest_end_time         | 2026-09-25 10:45:12.123456+00
slot_name               | standby01_slot
sender_host             | 192.168.10.11
sender_port             | 5432
```

คำนวณ **network lag โดยประมาณ** จากผลต่างระหว่าง `last_msg_send_time` และ `last_msg_receipt_time`:

```sql
SELECT
    now() - last_msg_receipt_time AS time_since_last_receipt,
    last_msg_receipt_time - last_msg_send_time AS network_delay
FROM pg_stat_wal_receiver;
```

### 627.4 วัด Lag แบบเวลานาฬิกาจริง (Time-based Lag)

วิธีที่ตรงไปตรงมาและเข้าใจง่ายที่สุดสำหรับ dashboard/alerting คือใช้ฟังก์ชัน `pg_last_xact_replay_timestamp()` รันบน Standby เพื่อดูว่า transaction ล่าสุดที่ replay เข้าไปนั้น เกิดขึ้นบน Primary เมื่อไหร่ แล้วเทียบกับเวลาปัจจุบัน:

```sql
-- รันบน Standby
SELECT
    now() AS current_time,
    pg_last_xact_replay_timestamp() AS last_replayed_transaction_time,
    now() - pg_last_xact_replay_timestamp() AS replication_lag;
```

```
          current_time          | last_replayed_transaction_time |   replication_lag
----------------------------------+-----------------------------------+----------------
 2026-09-25 10:45:20.502113+00   | 2026-09-25 10:45:20.480221+00     | 00:00:00.021892
(1 row)
```

วิธีนี้เป็นที่นิยมมากในเครื่องมือ monitoring เช่น Prometheus + `postgres_exporter` เพราะให้ค่าที่เข้าใจง่าย ("ล่าช้าอยู่กี่วินาที") โดยไม่ต้องแปลง LSN เป็นหน่วยเวลาเอง

### 627.5 ฟังก์ชันตรวจสอบ LSN ที่เป็นประโยชน์

```sql
-- ตรวจสอบว่า standby ทัน primary หรือยัง (byte diff = 0 คือทันสนิท)
SELECT pg_wal_lsn_diff(
    pg_current_wal_lsn(),           -- เรียกบน primary
    '0/5A12210'                     -- replay_lsn ที่ได้จาก standby
);

-- บน standby: ตรวจว่ากำลัง recovery อยู่หรือไม่ (true = เป็น standby, false = เป็น primary)
SELECT pg_is_in_recovery();

-- บน standby: ตรวจว่า WAL replay หยุด (paused) อยู่หรือไม่
SELECT pg_is_wal_replay_paused();
```

### 627.6 การตั้ง Alert สำหรับ Lag สูงผิดปกติ

แนวทางปฏิบัติในระบบ production คือกำหนด threshold และเชื่อมกับระบบ monitoring (เช่น Prometheus/Grafana, Zabbix, Nagios) เช่น:

```sql
-- Query สำหรับ monitoring script/exporter (รันบน primary ทุกๆ 10-30 วินาที)
SELECT
    application_name,
    CASE
        WHEN replay_lag > interval '30 seconds' THEN 'CRITICAL'
        WHEN replay_lag > interval '5 seconds'  THEN 'WARNING'
        ELSE 'OK'
    END AS lag_status,
    replay_lag
FROM pg_stat_replication;
```

| ระดับ Lag | ความหมาย | Action แนะนำ |
|---|---|---|
| < 1 วินาที | ปกติ | ไม่ต้องดำเนินการ |
| 1-5 วินาที | เริ่มสังเกต | ตรวจสอบ network/I/O load |
| 5-30 วินาที | Warning | ตรวจสอบ standby query ที่ block replay, disk I/O |
| > 30 วินาที | Critical | ตรวจสอบทันที อาจต้องพิจารณา failover หากเกี่ยวข้องกับ SLA |

---

## Step 628: Hot Standby — Read-Only Query บน Standby

### 628.1 Hot Standby คืออะไร

**Hot Standby** คือความสามารถของ PostgreSQL ที่อนุญาตให้ Standby server รับ **read-only query** ได้ในขณะที่ยัง apply WAL จาก Primary อยู่พร้อมกัน (ต่างจาก "warm standby" แบบเก่าที่ Standby จะไม่รับ connection ใดๆ เลยจนกว่าจะถูก promote)

การเปิดใช้งาน Hot Standby ทำให้ Standby กลายเป็น **Read Replica** ที่มีประโยชน์มากสำหรับการกระจายโหลด query ประเภท reporting, analytics, หรือ read-heavy API endpoint

### 628.2 การเปิดใช้งาน hot_standby

```conf
# /etc/postgresql/16/main/postgresql.conf (Standby)
hot_standby = on
```

> **หมายเหตุ:** ตั้งแต่ PostgreSQL 10 เป็นต้นมา `hot_standby = on` เป็นค่า **default** อยู่แล้ว แต่ควรระบุไว้ชัดเจนในไฟล์ config เพื่อความชัดเจนและป้องกันการเผลอปิดโดยไม่ตั้งใจ parameter นี้เป็น `postmaster` context จึงต้อง restart หากเปลี่ยนค่า

ตรวจสอบสถานะ hot_standby:

```sql
-- รันบน standby
SHOW hot_standby;
SELECT pg_is_in_recovery();  -- คืนค่า t เพราะเป็น standby
```

### 628.3 การเชื่อมต่อและ Query บน Standby

Client สามารถเชื่อมต่อ Standby ได้เหมือนเชื่อมต่อ PostgreSQL ปกติทุกประการ:

```bash
psql -h 192.168.10.12 -p 5432 -U app_readonly -d ecommerce_db
```

```sql
-- Query อ่านข้อมูลได้ปกติ
SELECT COUNT(*) FROM orders WHERE order_date >= CURRENT_DATE - INTERVAL '7 days';

-- แต่หากพยายาม write จะถูกปฏิเสธทันที
INSERT INTO orders (customer_id, total_amount) VALUES (1, 100.00);
```

```
ERROR:  cannot execute INSERT in a read-only transaction
```

PostgreSQL จะปฏิเสธคำสั่งเขียนข้อมูลทุกประเภทบน Hot Standby โดยอัตโนมัติ (INSERT, UPDATE, DELETE, DDL, และแม้แต่ `SELECT ... FOR UPDATE`) เนื่องจากระบบอยู่ในสถานะ read-only transaction เสมอ

### 628.4 Application Routing — แยก Read/Write Connection

ในระบบ production มักใช้เทคนิค **connection routing** เพื่อส่ง write query ไปที่ Primary และ read query ไปที่ Standby โดยอัตโนมัติ:

**แนวทางที่ 1: ใช้ libpq target_session_attrs (เหมาะกับ failover-aware client)**

```bash
# Connection string ที่ระบุหลาย host พร้อม target_session_attrs
psql "host=192.168.10.11,192.168.10.12 port=5432 target_session_attrs=read-write dbname=ecommerce_db user=app"
```

`target_session_attrs=read-write` จะบอก libpq client ให้ลองเชื่อมต่อไปทีละ host ในลิสต์ แล้วเลือก host แรกที่เป็น **read-write** (คือ Primary) เท่านั้น เหมาะสำหรับ automatic failover scenario

**แนวทางที่ 2: ใช้ Connection Pooler เช่น PgBouncer หรือ HAProxy แยก pool**

```ini
# pgbouncer.ini (ตัวอย่างแนวคิด - แยก pool write/read)
[databases]
ecommerce_write = host=192.168.10.11 port=5432 dbname=ecommerce_db
ecommerce_read  = host=192.168.10.12 port=5432 dbname=ecommerce_db
```

Application กำหนดเองในระดับโค้ดว่า query ไหนใช้ connection pool ใด (write pool สำหรับ transaction ที่เขียนข้อมูล, read pool สำหรับ SELECT ที่ไม่ critical)

### 628.5 ข้อจำกัดของ Hot Standby ที่ควรรู้

| ข้อจำกัด | คำอธิบาย |
|---|---|
| **Query Conflict** | หาก Primary ทำ VACUUM แล้วลบ row ที่ query บน Standby กำลังอ่านอยู่ (long-running query) อาจเกิด **replication conflict** ทำให้ query บน standby ถูก cancel |
| **max_standby_streaming_delay** | ควบคุมว่า Standby จะรอ query เสร็จนานแค่ไหนก่อนจะ cancel query เพื่อให้ WAL replay ดำเนินต่อได้ (default 30 วินาที) |
| **Temporary Tables** | ไม่สามารถสร้าง temporary table บน Hot Standby ได้ (เพราะต้องเขียนข้อมูล catalog) |
| **Sequence values** | การเรียก `nextval()` ไม่สามารถทำบน Standby ได้ (เป็นการเขียนข้อมูล) |

### 628.6 การจัดการ Query Conflict

```conf
# postgresql.conf (Standby) - ปรับค่าให้เหมาะกับ workload
max_standby_streaming_delay = 30s   # รอ query บน standby ได้นานเท่าไหร่ก่อน cancel
                                       # ค่าสูง = query มีโอกาส run จบมากขึ้น แต่ replication lag อาจเพิ่ม
                                       # ค่า -1 = รอไม่จำกัด (ไม่แนะนำ เพราะ lag จะพุ่งไม่จำกัด)
```

ตรวจสอบ conflict ที่เกิดขึ้นในอดีตด้วย:

```sql
-- รันบน standby
SELECT * FROM pg_stat_database_conflicts WHERE datname = 'ecommerce_db';
```

```
 datid  |    datname    | confl_tablespace | confl_lock | confl_snapshot | confl_bufferpin | confl_deadlock
--------+----------------+-------------------+-------------+------------------+-------------------+-----------------
 16401  | ecommerce_db   |                 0 |           0 |                3 |                 0 |               0
(1 row)
```

คอลัมน์ `confl_snapshot` ที่ไม่เป็นศูนย์ บ่งบอกว่ามี query ถูก cancel เนื่องจาก snapshot conflict (มักเกิดจาก VACUUM บน Primary ที่ไปลบ row ที่ query บน standby ต้องการอ่าน)

---

## Step 629: Failover เบื้องต้น — Promote Standby เป็น Primary

### 629.1 Failover คืออะไร

**Failover** คือกระบวนการเปลี่ยน Standby server ให้กลายเป็น **Primary** ใหม่ เมื่อ Primary เดิมล่มหรือใช้งานไม่ได้ กระบวนการนี้เรียกว่า **Promotion**

> **ข้อสำคัญ:** บทนี้ครอบคลุมเฉพาะ **manual failover** (การ promote ด้วยมือโดย admin) เท่านั้น ส่วน **automated failover** ด้วยเครื่องมือเช่น Patroni, repmgr, หรือ pg_auto_failover จะเจาะลึกใน **Part 065**

### 629.2 วิธีที่ 1 — ใช้ฟังก์ชัน pg_promote()

ตั้งแต่ PostgreSQL 12 เป็นต้นมา มีฟังก์ชัน SQL `pg_promote()` ที่สามารถเรียกจากภายใน SQL session ได้โดยตรง โดยไม่ต้อง shell access ไปที่เครื่อง standby (สะดวกมากสำหรับระบบที่ใช้ connection pooler หรือ automation script):

```sql
-- รันบน Standby ที่ต้องการ promote เป็น Primary ใหม่
SELECT pg_promote(wait => true, wait_seconds => 60);
```

**พารามิเตอร์:**

| พารามิเตอร์ | ความหมาย |
|---|---|
| `wait` | ถ้า `true` (default) ฟังก์ชันจะรอจนกว่า promotion เสร็จสมบูรณ์ก่อน return ค่า |
| `wait_seconds` | ระยะเวลาสูงสุดที่รอ (default 60 วินาที) |

ผลลัพธ์:

```
 pg_promote
------------
 t
(1 row)
```

ตรวจสอบว่า promote สำเร็จ:

```sql
SELECT pg_is_in_recovery();
```

```
 pg_is_in_recovery
--------------------
 f
(1 row)
```

ค่า `f` (false) หมายความว่าเซิร์ฟเวอร์นี้ไม่ได้อยู่ในสถานะ recovery/standby อีกต่อไป — กลายเป็น **Primary** เรียบร้อยแล้ว

### 629.3 วิธีที่ 2 — ใช้คำสั่ง pg_ctl promote

หากไม่สามารถเชื่อมต่อผ่าน SQL ได้ (เช่น server หยุดตอบสนอง) สามารถใช้คำสั่ง shell โดยตรงบนเครื่อง Standby:

```bash
# รันบนเครื่อง standby โดยตรง (shell access)
sudo -u postgres pg_ctl promote -D /var/lib/postgresql/16/main
```

```
waiting for server to promote.... done
server promoted
```

### 629.4 วิธีที่ 3 — ใช้ Trigger File (วิธีดั้งเดิม รองรับย้อนหลังไปถึง PostgreSQL เก่า)

ในเวอร์ชันเก่ากว่า PostgreSQL 12 (ก่อนมี `standby.signal`) การ promote ใช้วิธี touch ไฟล์ trigger ที่กำหนดไว้ใน `recovery.conf` อย่างไรก็ตาม **ใน PostgreSQL 16/17 วิธีนี้ยังคงใช้ได้** ผ่าน parameter `promote_trigger_file`:

**ขั้นตอนที่ 1:** กำหนด `promote_trigger_file` ใน `postgresql.conf` ของ Standby (ตั้งไว้ล่วงหน้าก่อนเกิดเหตุ):

```conf
# postgresql.conf (Standby)
promote_trigger_file = '/tmp/pg_promote_trigger'
```

**ขั้นตอนที่ 2:** เมื่อต้องการ promote จริง ให้สร้างไฟล์นั้นขึ้นมา:

```bash
sudo -u postgres touch /tmp/pg_promote_trigger
```

PostgreSQL จะตรวจพบไฟล์นี้ภายในไม่กี่วินาที (ตาม polling interval ภายใน) แล้วเริ่มกระบวนการ promote โดยอัตโนมัติ

### 629.5 สิ่งที่เกิดขึ้นระหว่างกระบวนการ Promote

```
┌─────────────────────────────────────────────────────────────────┐
│  1. Standby หยุดรับ WAL stream จาก Primary เดิม                     │
│  2. Standby replay WAL ที่เหลือค้างอยู่ให้ครบ (ถ้ามี)                  │
│  3. Standby สร้าง new timeline (timeline switch) เช่น จาก           │
│     timeline 1 เป็น timeline 2                                     │
│  4. ลบไฟล์ standby.signal ออก                                      │
│  5. เขียน checkpoint ใหม่                                          │
│  6. เปิดรับ read-write connection (กลายเป็น Primary เต็มรูปแบบ)      │
└─────────────────────────────────────────────────────────────────┘
```

ตรวจสอบ log หลัง promote:

```
2026-09-25 11:02:15.201 UTC [1845] LOG:  received promote request
2026-09-25 11:02:15.203 UTC [1845] LOG:  redo done at 0/5A12340
2026-09-25 11:02:15.210 UTC [1845] LOG:  selected new timeline ID: 2
2026-09-25 11:02:15.301 UTC [1845] LOG:  archive recovery complete
2026-09-25 11:02:15.350 UTC [1842] LOG:  database system is ready to accept connections
```

สังเกตบรรทัด `selected new timeline ID: 2` — นี่คือแนวคิด **timeline** ของ PostgreSQL ที่ใช้แยกแยะ "ประวัติศาสตร์" ของ WAL หลัง promote เพื่อป้องกันความสับสนหาก Primary เดิมกลับมาออนไลน์อีกครั้ง (Primary เดิมจะต้องถูกตั้งเป็น standby ใหม่ ไม่สามารถกลับมาเป็น Primary คู่ขนานได้ทันที — ต้องทำ `pg_rewind` หรือสร้างใหม่จาก base backup)

### 629.6 สิ่งที่ต้องทำหลัง Promote (Checklist เบื้องต้น)

1. **ตรวจสอบว่า Primary ใหม่รับ write ได้จริง**

   ```sql
   CREATE TABLE IF NOT EXISTS _failover_test (id serial, checked_at timestamptz DEFAULT now());
   INSERT INTO _failover_test DEFAULT VALUES;
   DROP TABLE _failover_test;
   ```

2. **อัปเดต DNS/Load Balancer/connection string ของ application** ให้ชี้มาที่ Primary ใหม่

3. **จัดการ Primary เดิม (ถ้ากลับมาออนไลน์)** — ห้ามปล่อยให้กลับมาเป็น Primary คู่ขนาน (จะเกิด **split-brain**) ต้องตั้งเป็น Standby ใหม่โดยใช้ `pg_rewind` หรือ `pg_basebackup` ใหม่ทั้งหมด:

   ```bash
   # ใช้ pg_rewind เพื่อ sync primary เดิมให้ตามทัน timeline ใหม่ (เร็วกว่า basebackup ใหม่)
   sudo -u postgres pg_rewind \
       --target-pgdata=/var/lib/postgresql/16/main \
       --source-server="host=192.168.10.12 port=5432 user=replicator dbname=postgres"

   # จากนั้นสร้าง standby.signal และตั้งค่า primary_conninfo ให้ชี้ไปยัง primary ใหม่
   touch /var/lib/postgresql/16/main/standby.signal
   ```

4. **ตั้ง Standby ตัวอื่น (ถ้ามีหลายตัว) ให้เชื่อมต่อกับ Primary ใหม่** แทน Primary เดิม

### 629.7 ข้อควรระวังสำคัญเรื่อง Split-Brain

**Split-brain** คือสถานการณ์อันตรายที่สุดใน replication topology — เกิดขึ้นเมื่อมี **Primary สองตัวพร้อมกัน** รับ write connection จาก application คนละตัว ทำให้ข้อมูลแตกแขนงออกจากกัน (diverge) และไม่สามารถ merge กลับมาเป็นชุดเดียวกันได้อีกโดยอัตโนมัติ

สาเหตุที่พบบ่อย: Admin promote Standby เป็น Primary ใหม่ โดยไม่ได้ตรวจสอบให้แน่ใจว่า Primary เดิมถูกปิดหรือตัดขาดจาก network เรียบร้อยแล้ว (เทคนิคที่เรียกว่า **STONITH — Shoot The Other Node In The Head** ในระบบ HA แบบดั้งเดิม)

> **กฎเหล็กของ manual failover:** ก่อน promote Standby ทุกครั้ง ต้อง**ยืนยันแน่ชัด**ว่า Primary เดิมถูกปิดหรือตัดขาดจาก network โดยสมบูรณ์แล้วเท่านั้น — ในบทถัดไป (Part 065) เราจะเรียนรู้เครื่องมือ automated failover ที่มีกลไกป้องกัน split-brain แบบ built-in (fencing)

---

## สรุปท้ายบท

ในบทนี้เราได้เรียนรู้แนวคิดและการปฏิบัติจริงของ **Streaming Replication (Physical Replication)** ใน PostgreSQL อย่างครบถ้วน ตั้งแต่หลักการพื้นฐานไปจนถึงการ promote failover:

1. **Streaming Replication** ส่ง WAL แบบ real-time จาก Primary ไปยัง Standby ผ่าน `walsender`/`walreceiver` process ทำให้ lag ต่ำกว่า log shipping แบบดั้งเดิมมาก
2. **สถาปัตยกรรม** มีได้หลายแบบ: single standby (ง่ายที่สุด), cascading replication (ลดภาระ Primary), และ multiple standby fan-out (ยืดหยุ่นสูงสุด)
3. การตั้งค่า **Primary** ต้องมี `wal_level = replica`, `max_wal_senders`, และสร้าง replication user ด้วย `REPLICATION` attribute
4. **`pg_hba.conf`** ต้องอนุญาต connection ประเภท `replication` แยกจาก connection ปกติ โดยเจาะจง IP ของ Standby
5. **`pg_basebackup -R`** เป็นเครื่องมือหลักในการสร้าง Standby ใหม่ — สร้าง `standby.signal` และ `primary_conninfo` ให้อัตโนมัติ
6. **Synchronous vs Asynchronous** เป็น trade-off ระหว่างความปลอดภัยของข้อมูล (RPO=0) กับ latency — ควบคุมผ่าน `synchronous_standby_names` และ `synchronous_commit`
7. **Replication Lag** วัดได้จาก `pg_stat_replication` (ฝั่ง Primary) และ `pg_stat_wal_receiver` (ฝั่ง Standby) รวมถึง `pg_last_xact_replay_timestamp()` สำหรับ time-based lag
8. **Hot Standby** เปิดให้ query แบบ read-only บน Standby ได้พร้อมกับ replication ที่ยังทำงานอยู่ — ใช้สำหรับ read scaling
9. **Failover** เบื้องต้นทำผ่าน `pg_promote()`, `pg_ctl promote`, หรือ trigger file — ต้องระวังเรื่อง **split-brain** อย่างเคร่งครัด
10. Automated failover (Patroni, repmgr, pg_auto_failover) จะเรียนเจาะลึกใน **Part 065**

Streaming Replication เป็นรากฐานสำคัญที่สุดอย่างหนึ่งของการทำ **High Availability** ใน PostgreSQL — เป็นทักษะที่ DBA และ Platform Engineer ระดับมืออาชีพทุกคนต้องเชี่ยวชาญ ก่อนจะต่อยอดไปสู่ automated failover, load balancing, และ multi-region deployment ในบทถัดๆ ไป

---

## แบบฝึกหัด

<details>
<summary><strong>แบบฝึกหัดที่ 1:</strong> Streaming Replication ทำงานโดยส่งข้อมูลระดับใด และ process ใดบน Primary/Standby ที่ทำหน้าที่ส่ง/รับ WAL?</summary>

**เฉลย:**

Streaming Replication ทำงานที่ระดับ **physical (block-level)** คือ copy ข้อมูล WAL record แบบ byte-for-byte ไม่ใช่ระดับ SQL statement หรือ row

- บน **Primary**: process ชื่อ **`walsender`** ทำหน้าที่อ่าน WAL แล้วสตรีมส่งไปยัง Standby
- บน **Standby**: process ชื่อ **`walreceiver`** ทำหน้าที่รับ WAL stream แล้วเขียนลง WAL file ของตัวเอง จากนั้น process `startup` (recovery process) จะ replay WAL เข้าไปใน data files

สามารถตรวจสอบ process เหล่านี้ได้ด้วย `ps aux | grep walsender` (บน Primary) หรือ `ps aux | grep walreceiver` (บน Standby)

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 2:</strong> จงอธิบายความแตกต่างระหว่าง Single Standby, Cascading Replication, และ Multiple Standby (Fan-out) พร้อมยกตัวอย่างสถานการณ์ที่เหมาะสมกับแต่ละแบบ</summary>

**เฉลย:**

| Topology | ลักษณะ | เหมาะกับ |
|---|---|---|
| **Single Standby** | Primary ส่ง WAL ไปยัง Standby ตัวเดียวโดยตรง | ระบบขนาดเล็ก-กลาง ต้องการ HA พื้นฐาน ไม่ซับซ้อน |
| **Cascading Replication** | Standby ตัวหนึ่งทำหน้าที่ relay WAL ต่อไปยัง Standby ตัวอื่น (ไม่ใช่ทุกตัวเชื่อมกับ Primary โดยตรง) | ระบบใหญ่ที่มี Standby หลายตัวคนละ region/datacenter — ลดภาระ network/CPU บน Primary |
| **Multiple Standby (Fan-out)** | Primary ส่ง WAL ไปยัง Standby หลายตัวพร้อมกันโดยตรง | ต้องการทั้ง HA (Standby สำหรับ failover) และ Read Scaling (Standby สำหรับกระจาย read load) พร้อมกัน |

ตัวอย่าง: บริษัท e-commerce ที่มี datacenter หลักในกรุงเทพฯ และ DR site ที่สิงคโปร์ พร้อมทั้งต้องการ read replica สำหรับ reporting — อาจใช้ Multiple Standby แบบผสม (Standby-1 synchronous ในกรุงเทพฯ สำหรับ failover, Standby-2 asynchronous ในกรุงเทพฯ สำหรับ reporting, Standby-3 asynchronous ที่สิงคโปร์สำหรับ DR)

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 3:</strong> เขียนคำสั่ง SQL สำหรับสร้าง replication user ชื่อ `repl_user` พร้อมรหัสผ่าน และอธิบายว่าทำไมไม่ควรใช้ superuser สำหรับ replication connection</summary>

**เฉลย:**

```sql
CREATE ROLE repl_user WITH
    REPLICATION
    LOGIN
    ENCRYPTED PASSWORD 'MyStr0ngP@ssw0rd!';
```

**เหตุผลที่ไม่ควรใช้ superuser:**

1. **หลัก Least Privilege** — replication connection ต้องการเพียงสิทธิ์อ่าน WAL stream เท่านั้น ไม่จำเป็นต้องมีสิทธิ์เต็มระดับ superuser ที่สามารถแก้ไขข้อมูล, สร้าง/ลบ database, หรือเปลี่ยนแปลง system configuration ได้
2. **ลดความเสียหายหากรั่วไหล** — หากรหัสผ่านของ replication user รั่วไหล (เช่น ใน `postgresql.auto.conf`) ผู้ไม่หวังดีจะสามารถอ่าน WAL ได้ (ซึ่งก็ถือว่าอันตรายมากอยู่แล้วเพราะเห็นข้อมูลทุกตาราง) แต่จะไม่สามารถ**เขียน**หรือ**ทำลาย**ข้อมูลบน Primary ได้โดยตรงเหมือนกรณีที่รั่วไหลเป็น superuser credential

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 4:</strong> เขียนบรรทัด `pg_hba.conf` ที่อนุญาตให้ standby server ที่ IP `10.0.5.20` เชื่อมต่อแบบ replication ด้วย user `repl_user` โดยใช้ SCRAM authentication</summary>

**เฉลย:**

```conf
host    replication     repl_user       10.0.5.20/32            scram-sha-256
```

คำอธิบาย: คอลัมน์ `DATABASE` ต้องเป็นคำว่า `replication` (keyword พิเศษ ไม่ใช่ชื่อฐานข้อมูล) และควรระบุ `ADDRESS` เป็น `/32` (host เดียว) เพื่อจำกัดขอบเขตความปลอดภัยให้แคบที่สุดเท่าที่จำเป็น

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 5:</strong> คำสั่ง `pg_basebackup -R` ทำอะไรบ้าง และไฟล์ใดที่ถูกสร้างขึ้นมาโดยอัตโนมัติ</summary>

**เฉลย:**

Flag `-R` (หรือ `--write-recovery-conf`) จะสร้างสองสิ่งให้อัตโนมัติในกระบวนการ base backup:

1. **`standby.signal`** — ไฟล์เปล่าวางไว้ใน data directory เพื่อบอก PostgreSQL ว่าเมื่อ start ขึ้นมาให้เข้าสู่ standby mode (ต่างจาก PostgreSQL 11 ลงไปที่ใช้ `recovery.conf`)
2. **`primary_conninfo`** — parameter ที่ถูกเขียนลงใน `postgresql.auto.conf` โดยเก็บข้อมูล connection string สำหรับเชื่อมต่อกลับไปยัง Primary (host, port, user, password) เพื่อให้ `walreceiver` ใช้ในการเริ่มรับ WAL stream ทันทีที่ standby start

ทั้งสองสิ่งนี้ทำให้ผู้ดูแลระบบไม่ต้องสร้างไฟล์เองด้วยมือเหมือนใน PostgreSQL เวอร์ชันเก่า

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 6:</strong> ตั้งค่า Synchronous Replication ให้ Primary รอ ACK จาก standby อย่างน้อย 2 ตัว จากทั้งหมด 3 ตัว (standby-a, standby-b, standby-c) โดยไม่สนใจว่าตัวไหนตอบก่อน</summary>

**เฉลย:**

ใช้รูปแบบ `ANY N` (quorum-based) ใน `synchronous_standby_names`:

```conf
# postgresql.conf (Primary)
synchronous_standby_names = 'ANY 2 (standby-a, standby-b, standby-c)'
```

รูปแบบนี้ต่างจาก `FIRST N` ตรงที่ `FIRST N` จะรอ ACK จาก N ตัว**แรกตามลำดับ priority**ในลิสต์เท่านั้น ในขณะที่ `ANY N` จะรอ ACK จาก **ตัวใดก็ได้ N ตัว** จากลิสต์ทั้งหมด ทำให้มีความยืดหยุ่นสูงกว่าในกรณีที่ standby บางตัวช้าหรือหลุดชั่วคราว

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 7:</strong> เขียน SQL query ที่รันบน Standby เพื่อดูว่า replication lag เป็นเวลากี่วินาที โดยใช้ `pg_last_xact_replay_timestamp()`</summary>

**เฉลย:**

```sql
SELECT
    now() - pg_last_xact_replay_timestamp() AS replication_lag;
```

หรือแบบละเอียดพร้อมแปลงเป็นวินาที:

```sql
SELECT
    EXTRACT(EPOCH FROM (now() - pg_last_xact_replay_timestamp())) AS lag_seconds;
```

ฟังก์ชัน `pg_last_xact_replay_timestamp()` คืนค่า timestamp ของ transaction ล่าสุดที่ถูก apply (replay) บน standby ตามที่ commit ไว้บน Primary — เมื่อนำมาลบกับเวลาปัจจุบัน (`now()`) จะได้ค่า lag เป็นระยะเวลาที่เข้าใจง่าย เหมาะสำหรับใช้ใน monitoring dashboard

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 8:</strong> เมื่อ query บน Hot Standby ถูก cancel ด้วยข้อความ error เกี่ยวกับ conflict สาเหตุที่เป็นไปได้คืออะไร และ parameter ใดที่ควรปรับ</summary>

**เฉลย:**

สาเหตุที่พบบ่อยที่สุดคือ **VACUUM conflict** — เมื่อ Primary ทำ VACUUM แล้วลบ dead row ที่ query บน Standby (ซึ่งใช้เวลานาน เช่น query สำหรับ reporting) กำลังอ่านอยู่ (ต้องการ snapshot ของข้อมูล ณ เวลาที่ query เริ่ม) PostgreSQL จะต้องเลือกระหว่างให้ WAL replay ดำเนินต่อ (คง lag ต่ำ) หรือให้ query บน standby ทำงานต่อจนจบ (ยอมให้ lag เพิ่มขึ้น)

Parameter ที่ควรปรับคือ **`max_standby_streaming_delay`** (สำหรับ conflict ระหว่าง streaming replication) กำหนดว่า Standby จะยอมรอ query ให้ทำงานต่อได้นานแค่ไหนก่อนจะ cancel query นั้นเพื่อให้ WAL replay ดำเนินต่อไปได้:

```conf
max_standby_streaming_delay = 60s   # เพิ่มจาก default 30s หากต้องการให้ query ยาวๆ มีโอกาสรันจบมากขึ้น
```

ทางเลือกอื่น: พิจารณาใช้ `hot_standby_feedback = on` เพื่อให้ Standby แจ้ง Primary ว่ามี query ที่ยังต้องการ row เก่าอยู่ ทำให้ Primary หน่วง VACUUM ของ row เหล่านั้นออกไป (แลกกับ table bloat ที่อาจเพิ่มขึ้นบน Primary)

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 9:</strong> เปรียบเทียบวิธี promote Standby เป็น Primary สามวิธีที่เรียนในบทนี้ (pg_promote(), pg_ctl promote, trigger file) — แต่ละวิธีเหมาะกับสถานการณ์ใด</summary>

**เฉลย:**

| วิธี | ลักษณะการใช้งาน | เหมาะกับ |
|---|---|---|
| **`pg_promote()`** | เรียกผ่าน SQL session ที่เชื่อมต่อกับ standby ได้ปกติ | เหมาะกับ automation script/application ที่มี SQL access อยู่แล้ว ไม่ต้องมี shell access ไปยังเครื่อง standby โดยตรง |
| **`pg_ctl promote`** | รันเป็น shell command บนเครื่อง standby โดยตรง | เหมาะกับ admin ที่มี SSH/shell access และต้องการ promote ทันทีโดยไม่ต้องพึ่ง SQL connection (เช่นกรณี PostgreSQL ไม่ตอบสนอง connection แต่ process ยังอยู่) |
| **Trigger file** (`promote_trigger_file`) | ต้องตั้งค่า parameter ไว้ล่วงหน้า แล้ว touch ไฟล์เมื่อต้องการ promote | เหมาะกับระบบเก่าที่ใช้แนวทางดั้งเดิม หรือ automation framework ที่ทำงานผ่าน filesystem (เช่น เขียน script monitor ไฟล์แล้ว trigger เอง) รองรับย้อนหลังไปถึง PostgreSQL เวอร์ชันเก่ามาก |

โดยทั่วไปในระบบ production สมัยใหม่ `pg_promote()` เป็นที่นิยมที่สุดเพราะเรียกผ่าน SQL ได้ง่ายและ integrate เข้ากับ automation tool ได้สะดวก

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 10 (แบบฝึกหัดรวม):</strong> จงออกแบบและ document ขั้นตอนการตั้งค่า Primary-Standby replication แบบเต็มรูปแบบสำหรับระบบ e-commerce ที่มีความต้องการดังนี้: (1) Primary ที่กรุงเทพฯ รับ write ทั้งหมด (2) Standby-A ที่กรุงเทพฯ (synchronous) สำหรับ failover แบบ zero data loss (3) Standby-B ที่เชียงใหม่ (asynchronous) สำหรับ read replica ของทีม reporting</summary>

**เฉลย:**

## เอกสารการตั้งค่า Primary-Standby Replication — ระบบ E-Commerce

### 1. ภาพรวม Topology

```
┌─────────────────────┐
│   PRIMARY            │
│   pg-primary-bkk      │
│   192.168.10.11       │
│   (Read/Write)        │
└──────────┬───────────┘
            │
    ┌───────┴────────┐
    │                 │
    ▼                 ▼
┌─────────────┐  ┌─────────────┐
│ STANDBY-A    │  │ STANDBY-B    │
│ pg-standby-  │  │ pg-standby-  │
│ bkk          │  │ cnx          │
│ 192.168.10.12│  │ 192.168.20.15│
│ SYNCHRONOUS  │  │ ASYNCHRONOUS │
│ (Failover)   │  │ (Reporting)  │
└─────────────┘  └─────────────┘
```

### 2. ขั้นตอนการตั้งค่า Primary (pg-primary-bkk)

**2.1 แก้ไข `postgresql.conf`:**

```conf
wal_level = replica
max_wal_senders = 10
max_replication_slots = 10
wal_keep_size = 2GB
hot_standby = on
synchronous_standby_names = 'standby_a'
archive_mode = on
archive_command = 'cp %p /mnt/wal_archive/%f'
```

**2.2 สร้าง replication user:**

```sql
CREATE ROLE replicator WITH REPLICATION LOGIN ENCRYPTED PASSWORD 'ChangeMe_UseVault!';
```

**2.3 แก้ไข `pg_hba.conf`:**

```conf
host    replication     replicator      192.168.10.12/32           scram-sha-256
host    replication     replicator      192.168.20.15/32           scram-sha-256
```

**2.4 สร้าง replication slot สำหรับแต่ละ standby:**

```sql
SELECT pg_create_physical_replication_slot('standby_a_slot');
SELECT pg_create_physical_replication_slot('standby_b_slot');
```

**2.5 Restart และ reload:**

```bash
sudo systemctl restart postgresql@16-main   # จำเป็นเพราะแก้ wal_level, max_wal_senders
```

### 3. ขั้นตอนการตั้งค่า Standby-A (Synchronous, กรุงเทพฯ)

```bash
sudo systemctl stop postgresql@16-main
sudo rm -rf /var/lib/postgresql/16/main/*

sudo -u postgres pg_basebackup \
    -h 192.168.10.11 -p 5432 -U replicator \
    -D /var/lib/postgresql/16/main \
    -Fp -Xs -P -R --checkpoint=fast

# แก้ primary_conninfo เพิ่ม application_name และ slot
sudo -u postgres psql -c "ALTER SYSTEM SET primary_slot_name = 'standby_a_slot';"
```

แก้ `postgresql.auto.conf` ให้มี `application_name=standby_a`:

```conf
primary_conninfo = 'user=replicator password=ChangeMe_UseVault! host=192.168.10.11 port=5432 application_name=standby_a'
primary_slot_name = 'standby_a_slot'
```

```conf
# postgresql.conf (Standby-A)
hot_standby = on
```

```bash
sudo systemctl start postgresql@16-main
```

### 4. ขั้นตอนการตั้งค่า Standby-B (Asynchronous, เชียงใหม่, สำหรับ Reporting)

ทำเหมือน Standby-A แต่ `application_name=standby_b` และ**ไม่ต้อง**อยู่ใน `synchronous_standby_names`:

```conf
primary_conninfo = 'user=replicator password=ChangeMe_UseVault! host=192.168.10.11 port=5432 application_name=standby_b'
primary_slot_name = 'standby_b_slot'
```

```conf
# postgresql.conf (Standby-B)
hot_standby = on
```

### 5. การตรวจสอบหลังตั้งค่าเสร็จ

```sql
-- รันบน Primary
SELECT application_name, client_addr, state, sync_state, replay_lag
FROM pg_stat_replication;
```

ผลลัพธ์คาดหวัง:

```
 application_name |  client_addr   |   state   | sync_state | replay_lag
-------------------+-----------------+-----------+------------+-------------
 standby_a         | 192.168.10.12   | streaming | sync       | 00:00:00.01
 standby_b         | 192.168.20.15   | streaming | async      | 00:00:00.08
(2 rows)
```

### 6. Monitoring และ Alert ที่แนะนำ

| Metric | Threshold | Action |
|---|---|---|
| `replay_lag` ของ standby_a | > 5 วินาที | แจ้งเตือนทันที (กระทบ zero-data-loss SLA) |
| `replay_lag` ของ standby_b | > 60 วินาที | แจ้งเตือน (กระทบความสดของข้อมูล reporting) |
| `sync_state` ของ standby_a | ไม่ใช่ `sync` | Critical alert — failover safety net หายไป |

### 7. แผน Failover (สรุปสั้น อ้างอิง Step 629)

1. ยืนยันว่า Primary เดิมล่ม/ตัดขาดสมบูรณ์ (ป้องกัน split-brain)
2. Promote standby_a (synchronous, zero data loss) ด้วย `SELECT pg_promote();`
3. อัปเดต application connection string / DNS ให้ชี้ไปยัง standby_a (Primary ใหม่)
4. ตั้ง standby_b ให้เชื่อมต่อกับ Primary ใหม่ (แก้ `primary_conninfo`)
5. เมื่อ Primary เดิมกลับมา ใช้ `pg_rewind` เพื่อตั้งเป็น standby ตัวใหม่ ห้ามให้กลับมาเป็น Primary คู่ขนานเด็ดขาด

**เหตุผลของการออกแบบ:**
- Standby-A ใช้ **synchronous** เพราะอยู่ใน datacenter เดียวกัน (network latency ต่ำ) ทำให้การรอ ACK ไม่กระทบ performance มากนัก ในขณะที่ได้ **RPO=0** สำหรับ failover
- Standby-B ใช้ **asynchronous** เพราะอยู่คนละจังหวัด (latency สูงกว่า) และงาน reporting ไม่ต้องการข้อมูล real-time เป๊ะ ยอมรับ lag เล็กน้อยได้เพื่อไม่ให้กระทบ write performance ของ Primary

</details>

---

**บทถัดไป:** [Part 064 — Logical Replication](./part-064-logical-replication.md)
