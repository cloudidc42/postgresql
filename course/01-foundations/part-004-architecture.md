# Part 004: โครงสร้างฐานข้อมูล — Cluster, Database, Schema, Table, Catalog

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับพื้นฐาน | Part 004

---

## เป้าหมายการเรียนรู้

เมื่อเรียนจบ Part นี้ ผู้เรียนจะสามารถ:

- อธิบายความหมายของคำว่า **Database Cluster** ในบริบทของ PostgreSQL ได้อย่างถูกต้อง และไม่สับสนกับคำว่า "cluster" ในความหมายของ high-availability
- เข้าใจว่า `initdb` สร้างอะไรขึ้นมาบ้างเมื่อเริ่มต้น cluster ใหม่
- อธิบายลำดับชั้น (hierarchy) ของโครงสร้างฐานข้อมูลใน PostgreSQL ตั้งแต่ระดับ Cluster ลงไปจนถึง Table
- เข้าใจความแตกต่างระหว่าง `postgres`, `template0`, `template1` และรู้วิธีสร้างฐานข้อมูลใหม่จาก template
- เข้าใจแนวคิดของ **Schema** ว่าทำไมต้องมี และใช้แก้ปัญหาอะไร
- ใช้งานและปรับแต่ง `search_path` เพื่อควบคุมการค้นหาออบเจ็กต์ข้าม schema
- แยกแยะออบเจ็กต์ประเภทต่างๆ ที่อยู่ภายใน schema เช่น Table, View, Sequence, Index, Function
- อ่านและ query ข้อมูลจาก **System Catalog** (`pg_catalog`) และ `information_schema` เพื่อสำรวจโครงสร้างฐานข้อมูล
- เข้าใจแนวคิดพื้นฐานเรื่อง Ownership, `current_user`, `session_user`
- วางแผนออกแบบ schema สำหรับโปรเจกต์จริงได้อย่างเป็นระบบ

---

## ภาพรวมโครงสร้าง: Cluster → Database → Schema → Table

ก่อนลงรายละเอียดแต่ละ Step ขอให้ผู้เรียนดูภาพรวมของโครงสร้างฐานข้อมูลใน PostgreSQL ก่อน เพื่อให้เห็นว่าทุกอย่างที่จะเรียนใน Part นี้ประกอบกันเป็นภาพใหญ่อย่างไร

```
PostgreSQL Server (postmaster process)
│
└── Database Cluster (1 instance ต่อ 1 data directory, เช่น $PGDATA)
    │
    ├── Database: postgres            (default maintenance database)
    ├── Database: template0           (ต้นแบบที่ "แช่แข็ง" ห้ามแก้ไข)
    ├── Database: template1           (ต้นแบบที่ปรับแต่งได้ ใช้เป็น default เวลา CREATE DATABASE)
    │
    └── Database: myshop              (ฐานข้อมูลที่ผู้ใช้สร้างขึ้นเอง)
        │
        ├── Schema: public            (schema เริ่มต้น)
        │   ├── Table: customers
        │   ├── Table: orders
        │   ├── View: v_active_orders
        │   ├── Sequence: orders_order_id_seq
        │   ├── Index: idx_orders_customer_id
        │   └── Function: fn_calculate_total()
        │
        ├── Schema: sales              (จัดกลุ่มตามโดเมนธุรกิจ)
        │   ├── Table: sales.invoices
        │   └── Table: sales.payments
        │
        ├── Schema: audit              (แยกส่วน logging/history)
        │   └── Table: audit.change_log
        │
        └── Schema: pg_catalog         (schema ระบบ เก็บ metadata ทั้งหมด)
            ├── Table: pg_class        (ทุก relation ในฐานข้อมูล)
            ├── Table: pg_namespace    (ทุก schema)
            ├── Table: pg_attribute    (ทุกคอลัมน์)
            └── Table: pg_type         (ทุกชนิดข้อมูล)
```

สังเกตว่า **หนึ่ง PostgreSQL server (หนึ่ง cluster) มีได้หลาย database**, **หนึ่ง database มีได้หลาย schema**, และ **หนึ่ง schema มีได้หลายออบเจ็กต์** (table, view, sequence, index, function, ฯลฯ) นี่คือ "namespace hierarchy" 3 ชั้นที่ PostgreSQL ใช้จัดระเบียบทุกสิ่งที่เราสร้าง

---

## Step 31: Database Cluster คืออะไร

### ความหมายที่มักสับสน

คำว่า "cluster" ในโลกไอทีทั่วไปมักหมายถึงกลุ่มของเครื่องเซิร์ฟเวอร์หลายเครื่องที่ทำงานร่วมกันเพื่อ high availability หรือ load balancing เช่น "database cluster" ใน MySQL Cluster หรือ etcd cluster

**แต่ใน PostgreSQL คำว่า "Database Cluster" มีความหมายเฉพาะที่แตกต่างไปโดยสิ้นเชิง**

> **Database Cluster** ใน PostgreSQL หมายถึง **กลุ่มของฐานข้อมูลทั้งหมดที่ถูกจัดการโดย PostgreSQL server instance เดียวกัน** ซึ่งถูกเก็บอยู่ใน **data directory** เดียวกัน (มักเรียกว่า `$PGDATA`) และถูกควบคุมโดย process หลักที่เรียกว่า `postmaster`

พูดง่ายๆ คือ เมื่อเรารัน `initdb` หนึ่งครั้ง เราจะได้ "cluster" หนึ่งชุด ซึ่งภายในนั้นสามารถมีได้หลาย database (postgres, template1, myshop, blog, ฯลฯ) แต่ทั้งหมดถูก serve โดย PostgreSQL server process ชุดเดียวกัน ฟังพอร์ตเดียวกัน (ปกติคือ 5432) และใช้ shared memory, WAL (Write-Ahead Log), background processes ร่วมกัน

ดังนั้นเมื่อเราพูดว่า "PostgreSQL server ตัวหนึ่งกำลังรันอยู่ที่พอร์ต 5432" นั่นหมายถึง cluster หนึ่งชุดกำลังทำงานอยู่ ไม่ใช่ database เดียว

> **ข้อควรระวังเรื่องศัพท์**: เวลาผู้เรียนไปอ่านเอกสารเกี่ยวกับ replication, failover หรือ high-availability (เช่น Patroni, repmgr, pgpool) คำว่า "cluster" ในบริบทนั้นจะหมายถึงกลุ่มของ PostgreSQL server หลายตัว (primary + replica) ที่ทำงานร่วมกัน ซึ่งเป็นคนละความหมายกับ "database cluster" ที่เราเรียนใน Step นี้ เราจะเรียนเรื่อง replication cluster ในภาคที่ว่าด้วย High Availability ในระดับ Expert ต่อไป

### initdb คืออะไร และสร้างอะไรบ้าง

`initdb` เป็นโปรแกรม (utility) ที่ใช้สร้าง database cluster ใหม่ทั้งหมด โดยจะสร้าง data directory และเติมโครงสร้างพื้นฐานที่จำเป็นทั้งหมดลงไป ปกติเมื่อติดตั้ง PostgreSQL ผ่าน package manager หรือ initialize ผ่าน container เช่น Docker ขั้นตอนนี้จะถูกทำให้อัตโนมัติแล้ว แต่ผู้เรียนควรเข้าใจว่าเบื้องหลังมันทำอะไร

ลองสั่งรันด้วยตนเอง (ตัวอย่างบนเครื่อง Linux):

```bash
# สร้าง cluster ใหม่ในไดเรกทอรี /usr/local/pgsql/data
initdb -D /usr/local/pgsql/data -U postgres --encoding=UTF8 --locale=en_US.UTF-8
```

เมื่อรันเสร็จ `initdb` จะสร้างสิ่งต่อไปนี้ภายใน data directory:

| รายการ | คำอธิบาย |
|---|---|
| `PG_VERSION` | ไฟล์บอกเวอร์ชัน major ของ PostgreSQL ที่สร้าง cluster นี้ |
| `base/` | ไดเรกทอรีเก็บไฟล์ข้อมูลของแต่ละ database (แต่ละ database มี subdirectory ของตัวเอง ตั้งชื่อตาม OID) |
| `global/` | ข้อมูลระดับ cluster ที่ใช้ร่วมกันทุก database เช่น `pg_database`, `pg_authid` (role/user) |
| `pg_wal/` | Write-Ahead Log สำหรับความทนทานของข้อมูล (durability) และใช้ใน replication |
| `pg_xact/` | สถานะของ transaction (commit/abort) |
| `pg_tblspc/` | symbolic link ไปยัง tablespace อื่นๆ (ถ้ามี) |
| `postgresql.conf` | ไฟล์ config หลักของ server |
| `pg_hba.conf` | กติกาการควบคุมการเชื่อมต่อ (host-based authentication) |
| `pg_ident.conf` | mapping สำหรับ identity-based authentication |
| databases เริ่มต้น | `postgres`, `template0`, `template1` |

```bash
$ ls /usr/local/pgsql/data
PG_VERSION      pg_dynshmem    pg_notify      pg_stat        pg_wal
base            pg_hba.conf    pg_replslot    pg_stat_tmp    pg_xact
global          pg_ident.conf  pg_serial      pg_subtrans    postgresql.auto.conf
pg_commit_ts    pg_logical     pg_snapshots   pg_tblspc      postgresql.conf
pg_dynshmem     pg_multixact   pg_snapshots   pg_twophase
```

**ประเด็นสำคัญที่ต้องจำ:**

- **หนึ่ง data directory = หนึ่ง cluster เท่านั้น** ห้ามรัน server สองตัวชี้ไปที่ data directory เดียวกันพร้อมกันเด็ดขาด เพราะจะทำให้ข้อมูลเสียหาย
- **การตั้งค่า encoding และ locale ถูกกำหนดตอน initdb** และมีผลกับทั้ง cluster (แม้ว่าใน PostgreSQL เวอร์ชันใหม่จะสามารถกำหนด locale เฉพาะ database ได้บางส่วนผ่าน collation แต่ default locale ของ cluster ยังคงถูกกำหนดตอนนี้)
- **ทรัพยากรระดับ process** เช่น shared_buffers, max_connections, WAL เป็นการตั้งค่าระดับ cluster ไม่ใช่ระดับ database — จึงกำหนดใน `postgresql.conf` ไฟล์เดียวสำหรับทุก database ใน cluster นั้น

---

## Step 32: ลำดับชั้นโครงสร้าง — Cluster → Database → Schema → Table

ตอนนี้เราเห็นภาพรวมทั้งหมดแล้ว เรามาแจกแจงความสัมพันธ์แต่ละชั้นให้ชัดเจนยิ่งขึ้น

```
┌─────────────────────────────────────────────────────────────┐
│                    Database Cluster                          │
│                 (1 instance / 1 $PGDATA)                      │
│                                                                 │
│   ┌───────────────┐  ┌───────────────┐  ┌───────────────┐    │
│   │ Database A     │  │ Database B     │  │ Database C     │  │
│   │ (postgres)     │  │ (myshop)       │  │ (blog)         │  │
│   │                │  │                │  │                │  │
│   │  ┌──────────┐  │  │  ┌──────────┐  │  │  ┌──────────┐  │  │
│   │  │ public   │  │  │  │ public   │  │  │  │ public   │  │  │
│   │  └──────────┘  │  │  │ sales    │  │  │  └──────────┘  │  │
│   │                │  │  │ audit    │  │  │                │  │
│   │                │  │  └──────────┘  │  │                │  │
│   └───────────────┘  └───────────────┘  └───────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

### ระดับที่ 1: Cluster

- คือ PostgreSQL server instance หนึ่งชุด (หนึ่ง `postmaster` process พร้อม background workers)
- มี data directory ของตัวเอง
- มีพอร์ตของตัวเอง (default 5432)
- ควบคุม role/user, tablespace และการตั้งค่าที่เป็น global ผ่าน `postgresql.conf`

### ระดับที่ 2: Database

- อยู่ภายใน cluster หนึ่งชุด สามารถมีได้หลาย database
- **แต่ละ database แยกจากกันโดยสมบูรณ์ (isolated)** — คุณไม่สามารถ query ข้าม database ได้โดยตรงด้วย SQL ธรรมดา (ต้องใช้ `dblink`, `postgres_fdw` หรือเชื่อมต่อใหม่)
- แต่ละ connection (session) จะเชื่อมต่อไปยัง **database เดียว** เท่านั้นตลอดอายุของ connection นั้น
- Role/User เป็น global ต่อ cluster (ใช้ login ได้ทุก database ถ้ามีสิทธิ์) แต่สิทธิ์การเข้าถึงออบเจ็กต์ภายในต้อง grant แยกต่อ database

### ระดับที่ 3: Schema

- อยู่ภายใน database หนึ่ง เป็น "namespace" ย่อยสำหรับจัดกลุ่มออบเจ็กต์
- หนึ่ง database มีได้หลาย schema
- ออบเจ็กต์ในต่าง schema สามารถมีชื่อซ้ำกันได้ (เช่น `sales.orders` กับ `archive.orders`)
- การอ้างอิงออบเจ็กต์แบบเต็มรูปแบบ (fully qualified name) คือ `database.schema.object` แต่เนื่องจากเราเชื่อมต่อ database เดียวอยู่แล้ว โดยทั่วไปจะเขียนแค่ `schema.object`

### ระดับที่ 4: Table / View / Sequence / Index / Function / ...

- คือออบเจ็กต์จริงที่เราใช้งาน อยู่ภายใน schema หนึ่ง

ตารางสรุปเปรียบเทียบ:

| ระดับ | ตัวอย่างคำสั่งสร้าง | จำนวนต่อระดับบน | แยกข้อมูลกันหรือไม่ |
|---|---|---|---|
| Cluster | `initdb` | — | ใช้ resource ร่วมกันหมด |
| Database | `CREATE DATABASE myshop;` | หลาย database ต่อ 1 cluster | แยกกันสมบูรณ์ (isolated) |
| Schema | `CREATE SCHEMA sales;` | หลาย schema ต่อ 1 database | อยู่ใน database เดียวกัน query ร่วมกันได้ |
| Table | `CREATE TABLE sales.orders (...);` | หลาย table ต่อ 1 schema | เป็นออบเจ็กต์จริง |

---

## Step 33: Database ใน PostgreSQL — postgres, template0, template1

เมื่อเราสร้าง cluster ใหม่ด้วย `initdb` จะมี database เริ่มต้น 3 ตัวถูกสร้างขึ้นมาโดยอัตโนมัติ ลองดูด้วยคำสั่ง `\l` ใน psql:

```sql
\l
```

```
                                   List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges
-----------+----------+----------+-------------+-------------+-----------------------
 postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
 template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
(3 rows)
```

### postgres

Database ที่สร้างมาเพื่อใช้เป็น **default database สำหรับงานดูแลระบบ (maintenance/administrative)** เวลา utility อย่าง `psql`, `pg_dump`, หรือ client library เชื่อมต่อโดยไม่ระบุชื่อ database มันมักจะ fallback มาที่ `postgres` โดย convention ไม่มีอะไรพิเศษทางเทคนิค เป็นเพียง database ธรรมดาที่สร้างไว้ให้สะดวก — ผู้เรียนสามารถสร้างตารางลงใน `postgres` ได้ แต่ **ไม่แนะนำให้ใช้เก็บข้อมูลแอปพลิเคชันจริง** ควรสร้าง database แยกต่างหากเสมอ

### template0

- เป็น database ต้นแบบที่ **"แช่แข็ง" (pristine/frozen)** ไม่มีการเปลี่ยนแปลงใดๆ นับตั้งแต่ initdb
- **ห้ามเชื่อมต่อเข้าไปแก้ไขโดยตรง** (`ALLOW_CONNECTIONS = false` ตามค่าเริ่มต้น)
- มีไว้เพื่อเป็น "safety net" — ถ้า `template1` ถูกแก้ไขจนเพี้ยนไปหรือมีปัญหาเรื่อง encoding/locale เราสามารถใช้ `template0` เป็นฐานสร้าง database ใหม่ที่ "สะอาด" ได้เสมอ
- ยังมีประโยชน์เวลาต้องการสร้าง database ด้วย encoding/locale ที่ต่างจาก `template1` เพราะ `template1` อาจมี encoding-dependent object อยู่แล้วซึ่งจะขัดขวางการสร้าง database ด้วย encoding อื่น ส่วน `template0` ไม่มีสิ่งเหล่านี้เจือปน

### template1

- เป็น database ต้นแบบที่ **`CREATE DATABASE` ใช้เป็นค่าเริ่มต้นเมื่อไม่ได้ระบุ `TEMPLATE`**
- สามารถแก้ไขได้ (เชื่อมต่อเข้าไป `CREATE EXTENSION`, สร้างตาราง, สร้าง function ได้ตามปกติ)
- **ข้อควรระวัง**: หากคุณสร้างออบเจ็กต์ใดๆ ไว้ใน `template1` ออบเจ็กต์นั้นจะถูกก็อปปี้ไปอยู่ใน **ทุก database ใหม่** ที่สร้างขึ้นต่อจากนี้โดยอัตโนมัติ! เทคนิคนี้มีประโยชน์มากถ้าต้องการให้ทุก database ใหม่มี extension หรือ schema มาตรฐานติดมาด้วยเสมอ เช่น

```sql
-- เชื่อมต่อไปยัง template1 แล้วติดตั้ง extension ที่อยากให้ทุก database ใหม่มีติดมาด้วย
\c template1
CREATE EXTENSION IF NOT EXISTS pgcrypto;
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
```

จากนี้ไป ทุกครั้งที่สร้าง database ใหม่ด้วย `CREATE DATABASE` (โดยไม่ระบุ template อื่น) database ใหม่นั้นจะมี extension `pgcrypto` และ `uuid-ossp` ติดตั้งมาให้อัตโนมัติ

### การสร้าง Database ใหม่

```sql
-- รูปแบบพื้นฐาน
CREATE DATABASE myshop;

-- รูปแบบเต็มพร้อมระบุรายละเอียด
CREATE DATABASE myshop
    OWNER = app_admin
    ENCODING = 'UTF8'
    LC_COLLATE = 'en_US.UTF-8'
    LC_CTYPE = 'en_US.UTF-8'
    TEMPLATE = template0
    CONNECTION LIMIT = 100;
```

**คำอธิบายพารามิเตอร์สำคัญ:**

| พารามิเตอร์ | ความหมาย |
|---|---|
| `OWNER` | role ที่เป็นเจ้าของ database (default คือ role ที่รันคำสั่ง) |
| `ENCODING` | character encoding เช่น `UTF8` (แนะนำเกือบทุกกรณี) |
| `LC_COLLATE` | กติกาการเรียงลำดับ (sort order) — **กำหนดได้ตอนสร้าง database เท่านั้น เปลี่ยนภายหลังไม่ได้** (ยกเว้นทำผ่าน dump/restore ใหม่) |
| `LC_CTYPE` | กติกาการจำแนกอักขระ (ตัวพิมพ์เล็ก/ใหญ่, ตัวอักษร/ตัวเลข) |
| `TEMPLATE` | database ต้นแบบ (default = `template1`) |
| `CONNECTION LIMIT` | จำนวน connection สูงสุดที่อนุญาต (-1 = ไม่จำกัด) |
| `IS_TEMPLATE` | ถ้าตั้งเป็น `true` จะทำให้ database นี้ใช้เป็น template ให้ database อื่นสร้างต่อได้ |

```sql
-- ทำไมต้องใช้ template0 แทน template1 บ่อยๆ?
-- เพราะถ้าต้องการเปลี่ยน LC_COLLATE/LC_CTYPE หรือ ENCODING ที่ต่างจาก template1
-- ต้องสร้างจาก template0 (ซึ่งไม่มี encoding-dependent object ติดมา)
CREATE DATABASE analytics_th
    TEMPLATE = template0
    ENCODING = 'UTF8'
    LC_COLLATE = 'th_TH.UTF-8'
    LC_CTYPE = 'th_TH.UTF-8';
```

**ลบ database:**

```sql
-- ต้องไม่มีใครเชื่อมต่ออยู่ในขณะนั้น
DROP DATABASE IF EXISTS old_test_db;

-- PostgreSQL 13+ สามารถบังคับตัดการเชื่อมต่อทั้งหมดก่อนลบได้
DROP DATABASE IF EXISTS old_test_db WITH (FORCE);
```

**เปลี่ยน database ที่กำลังใช้งานใน psql:**

```sql
\c myshop
-- หรือระบุ user/host ด้วย
\c myshop app_admin localhost 5432
```

---

## Step 34: Schema คืออะไร และทำไมต้องมี Schema

### นิยาม

**Schema** คือ "เนมสเปซ" (namespace) ภายในหนึ่ง database ที่ใช้จัดกลุ่มออบเจ็กต์ เช่น table, view, function, sequence, index, type ให้อยู่รวมกันเป็นหมวดหมู่ที่มีความหมาย โดยไม่ต้องสร้าง database แยกให้ยุ่งยาก

ลองจินตนาการว่า **database เปรียบเหมือนฮาร์ดดิสก์หนึ่งลูก** และ **schema เปรียบเหมือนโฟลเดอร์ (folder) ภายในฮาร์ดดิสก์นั้น** — ไฟล์ (table) หลายไฟล์ในคนละโฟลเดอร์สามารถชื่อเหมือนกันได้ ไม่ชนกัน เพราะ path เต็มต่างกัน

### ทำไมต้องมี Schema — ปัญหาที่ Schema แก้ไข

1. **หลีกเลี่ยงชื่อชนกัน (name collision)**
   ถ้าทีม Sales และทีม HR ต่างมีตารางชื่อ `employees` ที่มีโครงสร้างต่างกัน หากไม่มี schema จะต้องตั้งชื่อ เช่น `sales_employees`, `hr_employees` ซึ่งดูรกและอ่านยาก แต่ด้วย schema สามารถมี `sales.employees` และ `hr.employees` แยกกันชัดเจนโดยไม่ชนกัน

2. **จัดกลุ่มตามโดเมนธุรกิจหรือ module** — เช่น `billing`, `inventory`, `reporting` ทำให้โครงสร้างฐานข้อมูลอ่านง่าย เข้าใจง่ายเมื่อโปรเจกต์ใหญ่ขึ้น

3. **ควบคุมสิทธิ์ (permission) เป็นกลุ่มได้ง่ายขึ้น** — สามารถ grant สิทธิ์ระดับ schema แทนที่จะต้อง grant ทีละตาราง

4. **หลีกเลี่ยงการต้องสร้าง database แยก** — การสร้าง database ใหม่มีต้นทุนสูงกว่า (แยก connection, แยก transaction, query ข้าม database ทำไม่ได้ตรงๆ) schema จึงเบากว่าและยืดหยุ่นกว่ามากสำหรับการแบ่งส่วนภายในระบบเดียวกัน

5. **รองรับ multi-tenancy แบบง่าย** — บาง pattern ใช้หนึ่ง schema ต่อหนึ่งลูกค้า (tenant) ภายใน database เดียว

### Schema เริ่มต้น: public

ทุก database ใหม่ที่สร้างขึ้น (ตั้งแต่ PostgreSQL 15 เป็นต้นไปมีการเปลี่ยนแปลงสิทธิ์เล็กน้อย — จะอธิบายด้านล่าง) จะมี schema ชื่อ `public` ติดมาให้อัตโนมัติ เมื่อเราสร้างตารางโดยไม่ระบุ schema เช่น

```sql
CREATE TABLE customers (
    customer_id SERIAL PRIMARY KEY,
    customer_name TEXT NOT NULL
);
```

ตารางนี้จะถูกสร้างใน schema `public` โดยอัตโนมัติ เทียบเท่ากับการเขียน:

```sql
CREATE TABLE public.customers (
    customer_id SERIAL PRIMARY KEY,
    customer_name TEXT NOT NULL
);
```

> **หมายเหตุสำคัญเรื่องความปลอดภัย (ตั้งแต่ PostgreSQL 15)**: ในเวอร์ชันก่อน 15 ทุก role (รวมถึง `PUBLIC` ซึ่งหมายถึงทุกคน) มีสิทธิ์ `CREATE` บน schema `public` โดยอัตโนมัติ ทำให้ใครก็ตามที่ connect เข้า database ได้สามารถสร้างตารางใน `public` ได้ — ซึ่งเป็นความเสี่ยงด้านความปลอดภัย ตั้งแต่ PostgreSQL 15 เป็นต้นไป database ใหม่จะไม่ให้สิทธิ์ `CREATE` แก่ `PUBLIC` บน schema `public` อีกต่อไป (แต่ยังให้สิทธิ์ `USAGE` อยู่) ผู้เรียนที่ใช้ PostgreSQL 15+ จึงต้อง `GRANT` สิทธิ์ `CREATE` ให้ role ที่ต้องการสร้างตารางใน `public` อย่างชัดเจน เราจะเจาะลึกเรื่องสิทธิ์ทั้งหมดใน Part 068-069

### การสร้าง Schema

```sql
-- สร้าง schema พื้นฐาน
CREATE SCHEMA sales;

-- สร้างพร้อมกำหนด owner
CREATE SCHEMA sales AUTHORIZATION app_admin;

-- สร้างแบบ "ถ้ายังไม่มี ค่อยสร้าง"
CREATE SCHEMA IF NOT EXISTS audit;

-- สร้าง schema พร้อมสร้างตารางภายในทันที (schema element)
CREATE SCHEMA hr
    CREATE TABLE employees (
        employee_id SERIAL PRIMARY KEY,
        full_name TEXT NOT NULL
    )
    CREATE VIEW v_employee_names AS
        SELECT employee_id, full_name FROM employees;
```

ดูรายการ schema ทั้งหมดในฐานข้อมูลปัจจุบัน:

```sql
\dn
```

```
   List of schemas
   Name   |  Owner
----------+----------
 audit    | postgres
 hr       | postgres
 public   | postgres
 sales    | app_admin
(4 rows)
```

หรือดูแบบละเอียดพร้อม access privileges:

```sql
\dn+
```

การสร้างตารางภายใน schema ที่ระบุชัดเจน ทำโดยใส่ prefix ชื่อ schema นำหน้า:

```sql
CREATE TABLE sales.orders (
    order_id SERIAL PRIMARY KEY,
    customer_id INTEGER NOT NULL,
    order_date DATE NOT NULL DEFAULT CURRENT_DATE,
    total_amount NUMERIC(12,2) NOT NULL
);

CREATE TABLE audit.change_log (
    log_id BIGSERIAL PRIMARY KEY,
    table_name TEXT NOT NULL,
    changed_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    changed_by TEXT NOT NULL
);
```

### ลบ Schema

```sql
-- ลบ schema ที่ว่างเปล่า (ไม่มีออบเจ็กต์ข้างใน)
DROP SCHEMA IF EXISTS temp_schema;

-- ลบ schema พร้อมออบเจ็กต์ทั้งหมดข้างในแบบ cascade (อันตราย! ต้องแน่ใจก่อนใช้)
DROP SCHEMA IF EXISTS old_module CASCADE;
```

---

## Step 35: search_path และการค้นหาออบเจ็กต์ข้าม Schema

### ปัญหา: เมื่อเขียนชื่อตารางโดยไม่ระบุ schema, PostgreSQL หาที่ไหน?

เวลาเราเขียน `SELECT * FROM orders;` โดยไม่ระบุ schema นำหน้า PostgreSQL ต้องมีกติกาว่าจะไปค้นหาตาราง `orders` ที่ schema ไหนก่อน กติกานี้ถูกควบคุมด้วยพารามิเตอร์ที่ชื่อ **`search_path`**

### ดูค่า search_path ปัจจุบัน

```sql
SHOW search_path;
```

```
   search_path
-----------------
 "$user", public
(1 row)
```

ค่าเริ่มต้นคือ `"$user", public` ซึ่งหมายความว่า:

1. PostgreSQL จะค้นหา schema ที่มีชื่อ**เดียวกับ current user ก่อน** (ถ้ามี schema ชื่อนั้นอยู่จริง) — ฟีเจอร์นี้มีประโยชน์สำหรับ pattern ที่ให้แต่ละ user มี "personal schema" ของตัวเอง
2. ถ้าไม่เจอใน schema ของ user (หรือไม่มี schema ชื่อนั้น) จะค้นหาต่อใน `public`
3. ถ้าไม่เจอใน `public` อีก จะเกิด error `relation "orders" does not exist`

### การเปลี่ยน search_path

```sql
-- ตั้ง search_path สำหรับ session ปัจจุบัน
SET search_path TO sales, public;

-- ตอนนี้ SELECT * FROM orders จะหาใน sales.orders ก่อน แล้วค่อย public.orders
SELECT * FROM orders;   -- เทียบเท่า SELECT * FROM sales.orders (ถ้ามี)

-- กลับไปใช้ค่า default
RESET search_path;
```

### ตัวอย่างการทำงานแบบเป็นขั้นตอน

```sql
CREATE SCHEMA sales;
CREATE SCHEMA archive;

CREATE TABLE sales.orders (order_id INT, note TEXT DEFAULT 'from sales');
CREATE TABLE archive.orders (order_id INT, note TEXT DEFAULT 'from archive');

INSERT INTO sales.orders (order_id) VALUES (1);
INSERT INTO archive.orders (order_id) VALUES (1);

SET search_path TO sales, public;
SELECT * FROM orders;
```

```
 order_id |    note
----------+-------------
        1 | from sales
(1 row)
```

```sql
SET search_path TO archive, public;
SELECT * FROM orders;
```

```
 order_id |     note
----------+--------------
        1 | from archive
(1 row)
```

จะเห็นว่าคำสั่ง `SELECT * FROM orders;` เดียวกันเป๊ะ ให้ผลลัพธ์ต่างกันตาม `search_path` ที่ตั้งไว้ — นี่คือเหตุผลว่าทำไมในโค้ดโปรดักชันจริงจึงนิยม **ระบุชื่อ schema แบบเต็ม (fully qualified name)** เสมอสำหรับ query ที่สำคัญ เพื่อป้องกันความกำกวม

### ตั้ง search_path แบบถาวรให้ user หรือ database

```sql
-- ตั้งให้ role นี้ ทุกครั้งที่ login จะได้ search_path นี้เสมอ
ALTER ROLE app_user SET search_path TO sales, public;

-- ตั้งระดับ database (มีผลกับทุก role ที่เชื่อมต่อ database นี้ เว้นแต่ override)
ALTER DATABASE myshop SET search_path TO app, public;
```

### schema pg_catalog อยู่ใน search_path เสมอโดยนัย

แม้ `search_path` จะไม่ได้ระบุ `pg_catalog` ไว้ แต่ PostgreSQL จะ**ค้นหา `pg_catalog` ก่อนเสมอโดยอัตโนมัติ** (ไม่ว่าจะตั้ง search_path เป็นอะไรก็ตาม) เพื่อความปลอดภัยและความสอดคล้อง หากต้องการควบคุมตำแหน่งของ `pg_catalog` ใน search order อย่างชัดเจน สามารถใส่ชื่อมันลงใน `search_path` ตรงๆ ได้เช่นกัน:

```sql
SET search_path TO pg_catalog, sales, public;
```

การใส่ schema ไว้ท้ายๆ แบบนี้มีประโยชน์เรื่องความปลอดภัย — ป้องกันการโจมตีแบบ "schema poisoning" ที่ผู้ไม่หวังดีสร้างฟังก์ชันหรือ operator ปลอมชื่อซ้ำกับของระบบไว้ใน schema ที่ตนเขียนได้ แล้วหวังให้ query ของคนอื่นเรียกใช้ของปลอมโดยไม่รู้ตัว (เนื้อหาเชิงลึกเรื่อง security จะกล่าวถึงอีกครั้งใน Part ที่ว่าด้วย security hardening)

---

## Step 36: Table, View, Sequence, Index, Function — ออบเจ็กต์ในระดับ Schema

ทุก schema สามารถบรรจุออบเจ็กต์หลายประเภท โดยแต่ละประเภทมีบทบาทต่างกัน มาดูภาพรวมของแต่ละประเภท (รายละเอียดการใช้งานเชิงลึกจะอยู่ใน Part ถัดๆ ไป — ที่นี่เพียงปูพื้นให้เห็นว่าทุกอย่างล้วน "อยู่ใน schema")

### Table

โครงสร้างหลักที่เก็บข้อมูลจริงเป็นแถว (row) และคอลัมน์ (column)

```sql
CREATE TABLE sales.products (
    product_id SERIAL PRIMARY KEY,
    product_name TEXT NOT NULL,
    price NUMERIC(10,2) NOT NULL CHECK (price >= 0)
);
```

### View

"คำสั่ง SELECT ที่ถูกเก็บชื่อไว้" ใช้เหมือนตารางแต่ไม่มีการเก็บข้อมูลจริง (ยกเว้น Materialized View) — ทุกครั้งที่ query view ระบบจะไปรัน SELECT เบื้องหลังใหม่เสมอ

```sql
CREATE VIEW sales.v_expensive_products AS
SELECT product_id, product_name, price
FROM sales.products
WHERE price > 1000;
```

### Sequence

ออบเจ็กต์สร้างเลขลำดับอัตโนมัติ (auto-increment) มักใช้คู่กับ primary key เวลาสร้างคอลัมน์ชนิด `SERIAL`/`BIGSERIAL` หรือ `GENERATED AS IDENTITY` PostgreSQL จะสร้าง sequence ให้อัตโนมัติเบื้องหลัง

```sql
CREATE SEQUENCE sales.invoice_number_seq START WITH 1000 INCREMENT BY 1;

SELECT nextval('sales.invoice_number_seq');  -- ได้ 1000
SELECT nextval('sales.invoice_number_seq');  -- ได้ 1001
```

### Index

โครงสร้างช่วยเร่งความเร็วในการค้นหาข้อมูล ไม่ใช่ข้อมูลหลัก แต่เป็นโครงสร้างเสริม (จะเรียนลึกในภาค Performance)

```sql
CREATE INDEX idx_products_name ON sales.products (product_name);
```

### Function (และ Procedure)

โค้ดที่เก็บไว้ในฐานข้อมูล (stored routine) สามารถเขียนด้วย SQL, PL/pgSQL หรือภาษาอื่นที่รองรับ

```sql
CREATE FUNCTION sales.fn_total_revenue()
RETURNS NUMERIC
LANGUAGE sql
AS $$
    SELECT COALESCE(SUM(price), 0) FROM sales.products;
$$;

SELECT sales.fn_total_revenue();
```

### สรุปออบเจ็กต์ที่อาศัยอยู่ใน Schema

| ประเภทออบเจ็กต์ | คำสั่งสร้าง | ตัวอย่างการอ้างอิงแบบเต็ม |
|---|---|---|
| Table | `CREATE TABLE` | `sales.orders` |
| View | `CREATE VIEW` | `sales.v_expensive_products` |
| Materialized View | `CREATE MATERIALIZED VIEW` | `sales.mv_monthly_summary` |
| Sequence | `CREATE SEQUENCE` | `sales.invoice_number_seq` |
| Index | `CREATE INDEX` | (index ไม่ query ตรงๆ แต่ถูกอ้างถึงผ่านชื่อ) |
| Function / Procedure | `CREATE FUNCTION` / `CREATE PROCEDURE` | `sales.fn_total_revenue` |
| Custom Type (Domain/Enum/Composite) | `CREATE TYPE`, `CREATE DOMAIN` | `sales.order_status_enum` |
| Trigger | `CREATE TRIGGER` | ผูกอยู่กับ table เฉพาะ ไม่มี schema ของตัวเอง แต่ trigger function อยู่ใน schema |

ดูรายการออบเจ็กต์ทั้งหมดใน schema ด้วย psql meta-command:

```sql
\dt sales.*      -- list tables ใน schema sales
\dv sales.*      -- list views
\ds sales.*      -- list sequences
\df sales.*      -- list functions
\di sales.*      -- list indexes
\d sales.orders  -- ดูโครงสร้างตารางแบบละเอียด
```

---

## Step 37: System Catalog (pg_catalog) และ information_schema

### System Catalog คืออะไร

PostgreSQL เก็บ **metadata ของตัวมันเอง** (ข้อมูลเกี่ยวกับ database, schema, table, column, index, function, user, ฯลฯ) ไว้ในรูปแบบตารางธรรมดาที่ query ได้ด้วย SQL ปกติ! กลุ่มตารางเหล่านี้อยู่ใน schema พิเศษที่ชื่อ **`pg_catalog`** ซึ่งมีอยู่ในทุก database โดยอัตโนมัติ

นี่คือจุดเด่นสำคัญของ PostgreSQL — แทนที่จะต้องใช้คำสั่งพิเศษเพื่อดูโครงสร้างฐานข้อมูล (เหมือนบางระบบ) เราสามารถ `SELECT` จากตาราง catalog ได้เลยเหมือนตารางทั่วไป

### ตาราง Catalog ที่สำคัญที่สุด

| ตาราง | เก็บข้อมูลเกี่ยวกับ |
|---|---|
| `pg_database` | รายชื่อ database ทั้งหมดใน cluster |
| `pg_namespace` | รายชื่อ schema ทั้งหมด (namespace) |
| `pg_class` | ทุก relation — table, view, sequence, index, materialized view |
| `pg_attribute` | ทุกคอลัมน์ของทุก relation |
| `pg_type` | ทุกชนิดข้อมูล (data type) ทั้ง built-in และที่ผู้ใช้สร้างเอง |
| `pg_index` | ข้อมูลเชิงลึกของ index แต่ละตัว |
| `pg_proc` | ทุก function/procedure |
| `pg_constraint` | ทุก constraint (primary key, foreign key, check, unique) |
| `pg_roles` | ทุก role/user ใน cluster |
| `pg_tables` | (view สรุป) รายการตารางทั้งหมด อ่านง่ายกว่า pg_class |
| `pg_views` | (view สรุป) รายการ view ทั้งหมด |

### ตัวอย่างการ query pg_catalog โดยตรง

```sql
-- ดูรายชื่อ schema ทั้งหมด (ไม่รวม schema ระบบ)
SELECT nspname AS schema_name
FROM pg_catalog.pg_namespace
WHERE nspname NOT LIKE 'pg_%'
  AND nspname <> 'information_schema'
ORDER BY nspname;
```

```
 schema_name
-------------
 audit
 hr
 public
 sales
(4 rows)
```

```sql
-- ดูตารางทั้งหมด พร้อม schema ที่มันสังกัดอยู่ (join pg_class กับ pg_namespace)
SELECT n.nspname AS schema_name,
       c.relname AS table_name,
       c.relkind AS kind
FROM pg_catalog.pg_class c
JOIN pg_catalog.pg_namespace n ON n.oid = c.relnamespace
WHERE c.relkind = 'r'                 -- 'r' = ordinary table
  AND n.nspname NOT IN ('pg_catalog', 'information_schema')
ORDER BY n.nspname, c.relname;
```

```
 schema_name | table_name | kind
-------------+------------+------
 audit       | change_log | r
 hr          | employees  | r
 sales       | orders     | r
 sales       | products   | r
(4 rows)
```

> **ความหมายของ `relkind`**: `r` = ordinary table, `v` = view, `m` = materialized view, `i` = index, `S` = sequence, `c` = composite type, `f` = foreign table, `p` = partitioned table

```sql
-- ดูคอลัมน์ทั้งหมดของตาราง sales.orders โดยตรงจาก pg_attribute
SELECT a.attname AS column_name,
       t.typname AS data_type,
       a.attnotnull AS not_null
FROM pg_catalog.pg_attribute a
JOIN pg_catalog.pg_class c ON c.oid = a.attrelid
JOIN pg_catalog.pg_namespace n ON n.oid = c.relnamespace
JOIN pg_catalog.pg_type t ON t.oid = a.atttypid
WHERE n.nspname = 'sales'
  AND c.relname = 'orders'
  AND a.attnum > 0            -- ตัด system column ออก (attnum <= 0)
  AND NOT a.attisdropped       -- ตัดคอลัมน์ที่ถูกลบไปแล้วออก
ORDER BY a.attnum;
```

```
 column_name  | data_type | not_null
--------------+-----------+----------
 order_id     | int4      | t
 customer_id  | int4      | t
 order_date   | date      | t
 total_amount | numeric   | t
(4 rows)
```

### information_schema — มาตรฐาน SQL ที่พกพาข้ามระบบได้

`pg_catalog` เป็นโครงสร้างเฉพาะของ PostgreSQL (แต่ละ DBMS มี catalog ของตัวเองในรูปแบบต่างกัน) แต่ SQL Standard ได้กำหนด **`information_schema`** ไว้เป็นมาตรฐานกลาง ซึ่ง PostgreSQL, MySQL, SQL Server ต่างก็รองรับ (แม้รายละเอียดจะต่างกันบ้าง) ทำให้ query ที่เขียนบน `information_schema` มีโอกาส portable ข้ามระบบได้มากกว่า

```sql
-- ดูตารางทั้งหมดผ่าน information_schema (อ่านง่ายกว่า pg_catalog มาก)
SELECT table_schema, table_name, table_type
FROM information_schema.tables
WHERE table_schema NOT IN ('pg_catalog', 'information_schema')
ORDER BY table_schema, table_name;
```

```
 table_schema | table_name | table_type
--------------+------------+------------
 audit        | change_log | BASE TABLE
 hr           | employees  | BASE TABLE
 sales        | orders     | BASE TABLE
 sales        | products   | BASE TABLE
(4 rows)
```

```sql
-- ดูคอลัมน์ของตาราง sales.orders ผ่าน information_schema
SELECT column_name, data_type, is_nullable, column_default
FROM information_schema.columns
WHERE table_schema = 'sales'
  AND table_name = 'orders'
ORDER BY ordinal_position;
```

```
 column_name  |     data_type     | is_nullable |           column_default
--------------+-------------------+-------------+--------------------------------------
 order_id     | integer           | NO          | nextval('sales.orders_order_id_seq')
 customer_id  | integer           | NO          |
 order_date   | date              | NO          | CURRENT_DATE
 total_amount | numeric           | NO          |
(4 rows)
```

### เปรียบเทียบ pg_catalog กับ information_schema

| ประเด็น | pg_catalog | information_schema |
|---|---|---|
| มาตรฐาน | เฉพาะ PostgreSQL | SQL Standard (portable) |
| ความละเอียด | ละเอียดมาก เข้าถึงข้อมูลภายในทุกอย่าง | ครอบคลุมเฉพาะสิ่งที่ standard กำหนด |
| ความเร็ว | เร็วกว่า (query ตรงกับ internal table) | อาจช้ากว่าเล็กน้อย (เป็น view ที่ join ซับซ้อนกว่า) |
| ใช้เมื่อไร | ต้องการข้อมูลเชิงลึกเฉพาะของ PostgreSQL เช่น OID, storage, index internals | ต้องการเขียน query ที่พกพาข้ามระบบได้ หรือทำงานทั่วไปที่ standard ครอบคลุมพอ |

---

## Step 38: การ Query Metadata เพื่อสำรวจโครงสร้างฐานข้อมูล

ใน Step นี้เราจะรวบรวม query ที่มีประโยชน์จริงในการทำงาน สำหรับ "สำรวจ" โครงสร้างฐานข้อมูลโดยไม่ต้องพึ่งเครื่องมือ GUI

### ดูขนาดของแต่ละ database ใน cluster

```sql
SELECT datname AS database_name,
       pg_size_pretty(pg_database_size(datname)) AS size
FROM pg_database
ORDER BY pg_database_size(datname) DESC;
```

```
 database_name |  size
----------------+---------
 myshop         | 45 MB
 postgres       | 7621 kB
 template1      | 7469 kB
 template0      | 7461 kB
(4 rows)
```

### ดูขนาดของแต่ละตาราง (รวม index) ใน schema ปัจจุบัน

```sql
SELECT n.nspname || '.' || c.relname AS table_name,
       pg_size_pretty(pg_total_relation_size(c.oid)) AS total_size,
       pg_size_pretty(pg_relation_size(c.oid)) AS table_only_size
FROM pg_class c
JOIN pg_namespace n ON n.oid = c.relnamespace
WHERE c.relkind = 'r'
  AND n.nspname NOT IN ('pg_catalog', 'information_schema')
ORDER BY pg_total_relation_size(c.oid) DESC;
```

### ดู primary key และ foreign key ทั้งหมดของ schema

```sql
SELECT tc.table_schema,
       tc.table_name,
       tc.constraint_type,
       kcu.column_name,
       ccu.table_name  AS foreign_table_name,
       ccu.column_name AS foreign_column_name
FROM information_schema.table_constraints tc
JOIN information_schema.key_column_usage kcu
     ON tc.constraint_name = kcu.constraint_name
    AND tc.table_schema = kcu.table_schema
LEFT JOIN information_schema.constraint_column_usage ccu
     ON tc.constraint_name = ccu.constraint_name
    AND tc.constraint_type = 'FOREIGN KEY'
WHERE tc.table_schema = 'sales'
  AND tc.constraint_type IN ('PRIMARY KEY', 'FOREIGN KEY')
ORDER BY tc.table_name, tc.constraint_type;
```

### ดูจำนวน row โดยประมาณของแต่ละตาราง (จาก statistics ไม่ใช่ COUNT(*) จริง จึงเร็วมาก)

```sql
SELECT schemaname, relname AS table_name, n_live_tup AS estimated_rows
FROM pg_stat_user_tables
WHERE schemaname = 'sales'
ORDER BY n_live_tup DESC;
```

### ดู index ทั้งหมดของตารางหนึ่ง พร้อมคำนิยาม (definition)

```sql
SELECT indexname, indexdef
FROM pg_indexes
WHERE schemaname = 'sales'
  AND tablename = 'orders';
```

### ใช้ psql meta-command แทน query เองในงานประจำวัน

ในทางปฏิบัติ เมื่อทำงานผ่าน `psql` เราไม่จำเป็นต้องเขียน query ยาวๆ ข้างต้นเองทุกครั้ง — `psql` มี meta-command (backslash command) ที่ทำหน้าที่เหล่านี้ให้แล้ว โดยเบื้องหลังมันก็คือการ query จาก `pg_catalog`/`information_schema` เช่นกัน (ผู้เรียนสามารถเปิดดู source query จริงได้ด้วยการรัน `psql` แบบ `-E` เพื่อแสดง query ที่ backslash command สร้างขึ้น)

```sql
\d sales.orders          -- โครงสร้างตารางแบบละเอียด (column, type, index, constraint)
\d+ sales.orders          -- แบบละเอียดยิ่งขึ้น (รวม storage, size, description)
\dt sales.*                -- รายการตารางใน schema sales
\di sales.*                -- รายการ index
\df+ sales.fn_total_revenue -- นิยามของ function
```

```bash
# ตัวอย่างการดู query จริงที่ \d ใช้
psql -E -d myshop -c '\d sales.orders'
```

---

## Step 39: Ownership และการเป็นเจ้าของออบเจ็กต์

### แนวคิดพื้นฐาน

ทุกออบเจ็กต์ใน PostgreSQL (database, schema, table, function, ฯลฯ) **ต้องมีเจ้าของ (owner) เสมอ** — เจ้าของคือ role ที่สร้างออบเจ็กต์นั้น (โดย default) หรือ role ที่ถูกระบุไว้ตอนสร้างด้วยคำสั่ง `AUTHORIZATION`/`OWNER`

**สิทธิ์ของเจ้าของออบเจ็กต์:**

- เจ้าของสามารถทำ **อะไรก็ได้กับออบเจ็กต์นั้น** โดยไม่ต้องขอสิทธิ์เพิ่ม (SELECT, INSERT, UPDATE, DELETE, DROP, ALTER, GRANT สิทธิ์ต่อให้คนอื่น ฯลฯ)
- role ที่เป็น **superuser** (เช่น `postgres`) มีสิทธิ์เหนือทุกออบเจ็กต์เสมอ ไม่ว่าจะเป็นเจ้าของหรือไม่

### ดูเจ้าของออบเจ็กต์

```sql
-- เจ้าของ database
\l
-- เจ้าของ schema
\dn+
-- เจ้าของตาราง
\dt+ sales.*
```

หรือ query โดยตรง:

```sql
SELECT c.relname AS table_name, pg_get_userbyid(c.relowner) AS owner
FROM pg_class c
JOIN pg_namespace n ON n.oid = c.relnamespace
WHERE n.nspname = 'sales' AND c.relkind = 'r';
```

```
 table_name |  owner
------------+-----------
 orders     | app_admin
 products   | app_admin
(2 rows)
```

### การเปลี่ยนเจ้าของ

```sql
ALTER TABLE sales.orders OWNER TO app_readonly_admin;
ALTER SCHEMA sales OWNER TO app_admin;
ALTER DATABASE myshop OWNER TO app_admin;
```

### current_user vs session_user

PostgreSQL มีฟังก์ชันพิเศษสองตัวที่มักถูกเข้าใจสับสน:

```sql
SELECT current_user, session_user;
```

- **`session_user`** — role ที่ใช้ **login เข้ามาจริงๆ ตอนเริ่มต้น connection** ค่านี้จะไม่เปลี่ยนตลอดอายุของ session (เว้นแต่ reconnect ใหม่)
- **`current_user`** — role ที่ **กำลังถูกใช้ตรวจสอบสิทธิ์ ณ ขณะนี้** ซึ่งสามารถเปลี่ยนแปลงได้ชั่วคราวภายใน session ด้วยคำสั่งเช่น `SET ROLE` หรือภายในฟังก์ชันที่ทำงานแบบ `SECURITY DEFINER`

ตัวอย่าง:

```sql
-- login มาด้วย role app_admin
SELECT current_user, session_user;
```

```
 current_user | session_user
--------------+---------------
 app_admin    | app_admin
(1 row)
```

```sql
-- สวมบทบาทชั่วคราวเป็น role อื่น (ต้องมีสิทธิ์ MEMBER OF role นั้นก่อน)
SET ROLE app_readonly;
SELECT current_user, session_user;
```

```
 current_user  | session_user
---------------+---------------
 app_readonly  | app_admin
(1 row)
```

```sql
-- กลับสู่ role เดิม
RESET ROLE;
```

จะเห็นว่า `session_user` ยังคงเป็น `app_admin` เหมือนเดิม (เพราะเป็นคนที่ login เข้ามาจริง) แต่ `current_user` เปลี่ยนเป็น `app_readonly` ชั่วคราว — สิทธิ์การเข้าถึงออบเจ็กต์ต่างๆ ในขณะนี้จะถูกตรวจสอบตาม `current_user` ไม่ใช่ `session_user`

> **หมายเหตุ**: หัวข้อเรื่องระบบสิทธิ์แบบเจาะลึก — `GRANT`/`REVOKE`, role hierarchy, row-level security, `SECURITY DEFINER` vs `SECURITY INVOKER`, default privileges — จะอยู่ใน **Part 068-069** ของหลักสูตรนี้ ในที่นี้ผู้เรียนเพียงต้องเข้าใจแนวคิดพื้นฐานว่าออบเจ็กต์ทุกชิ้นมีเจ้าของ และมีความแตกต่างระหว่าง identity ที่ login เข้ามา (`session_user`) กับ identity ที่ใช้ตรวจสอบสิทธิ์ ณ ขณะทำงาน (`current_user`)

---

## Step 40: แนวทางออกแบบ Schema ที่ดีสำหรับโปรเจกต์จริง

เมื่อเข้าใจกลไกทางเทคนิคแล้ว ประเด็นสำคัญคือ "แล้วเราควรออกแบบ schema อย่างไรในโปรเจกต์จริง?" ต่อไปนี้คือแนวทางที่ใช้ได้จริงในทีมพัฒนาระดับมืออาชีพ

### 1. กลยุทธ์ Multi-Schema (แบ่ง schema ตามอะไรดี)

**แบ่งตามโดเมนธุรกิจ/บริบทการทำงาน (business domain)**

```
myshop (database)
├── core       -- ตารางหลักของระบบ: customers, products, orders
├── billing    -- ใบแจ้งหนี้ การชำระเงิน
├── inventory  -- คลังสินค้า สต็อก
├── reporting  -- materialized view สำหรับรายงาน แยกจาก transactional data
└── audit      -- log การเปลี่ยนแปลงข้อมูล ประวัติการแก้ไข
```

**แบ่งตามระดับความเสถียรของข้อมูล (stability/lifecycle)**

```
myshop (database)
├── public         -- โครงสร้างหลักที่เสถียร
├── staging        -- ตารางชั่วคราวสำหรับ ETL/data loading
└── deprecated     -- ตารางเก่าที่รอ migrate/ลบ แยกไว้ให้ชัดเจนว่าไม่ควรใช้ต่อ
```

**แบ่งตาม tenant (สำหรับ multi-tenant SaaS แบบง่าย)**

```
saas_platform (database)
├── tenant_001
├── tenant_002
└── shared      -- ข้อมูลที่ใช้ร่วมกันทุก tenant เช่น ตาราง config กลาง
```

> **ข้อควรระวัง**: การแบ่งแบบ 1 schema ต่อ 1 tenant เหมาะกับจำนวน tenant ไม่มากนัก (หลักสิบถึงหลักร้อย) หาก tenant มีจำนวนหลักพันขึ้นไป การจัดการ schema จำนวนมากจะซับซ้อนขึ้นมาก ควรพิจารณาแนวทางอื่น เช่น row-level security ร่วมกับคอลัมน์ `tenant_id` แทน — จะกล่าวถึงรายละเอียดใน Part ที่ว่าด้วย Multi-Tenancy Architecture ในระดับ Expert

### 2. Naming Convention ที่แนะนำ

| หลักการ | ตัวอย่างที่ดี | ตัวอย่างที่ไม่ควรทำ |
|---|---|---|
| ใช้ตัวพิมพ์เล็กทั้งหมด (snake_case) | `customer_orders` | `CustomerOrders` (ต้อง quote ตลอด) |
| ชื่อ schema สั้น สื่อความหมาย | `billing`, `hr`, `sales` | `schema_for_billing_stuff` |
| ชื่อตารางเป็นพหูพจน์หรือเอกพจน์ให้สม่ำเสมอทั้งโปรเจกต์ | `customers`, `orders` (เลือกแบบใดแบบหนึ่งแล้วใช้ตลอด) | ผสมกัน `customer` กับ `orders` |
| ชื่อ primary key ให้สื่อถึงตาราง | `customer_id` | `id` (กำกวมเวลา join หลายตาราง) |
| ชื่อ foreign key ตรงกับ primary key ที่อ้างอิง | `orders.customer_id` อ้างถึง `customers.customer_id` | `orders.cust_fk` |
| Prefix ชื่อ view ด้วย `v_`, materialized view ด้วย `mv_` (ถ้าทีมตกลงกัน) | `v_active_orders`, `mv_monthly_sales` | ชื่อซ้ำกับตารางจนสับสน |
| หลีกเลี่ยงคำสงวนของ SQL เป็นชื่อ | `order_date`, `user_role` | `order`, `user`, `group` |

### 3. หลักการ Separation of Concerns

- **แยกข้อมูล transactional (OLTP) ออกจากข้อมูลสรุปเพื่อรายงาน (OLAP-ish)** — เก็บ materialized view หรือตารางสรุปไว้ใน schema `reporting` แยกจาก schema `core` เพื่อไม่ให้ query รายงานหนักๆ ไปกระทบ performance ของระบบหลัก
- **แยก audit/log ออกจากข้อมูลธุรกิจ** — วางไว้ใน schema `audit` ต่างหาก ทำให้ backup/retention policy ต่างกันได้ (เช่น เก็บ log ไว้แค่ 90 วัน แต่เก็บข้อมูลธุรกิจถาวร)
- **แยก extension บางตัวไว้ schema เฉพาะ** — เช่น ติดตั้ง extension `pgcrypto`, `uuid-ossp` ไว้ใน schema `extensions` แทนที่จะปนกับ `public` เพื่อความเป็นระเบียบและควบคุม `search_path` ได้ง่ายขึ้น

```sql
CREATE SCHEMA IF NOT EXISTS extensions;
CREATE EXTENSION IF NOT EXISTS pgcrypto SCHEMA extensions;
```

- **จำกัดสิทธิ์การเขียนใน `public`** — โดยเฉพาะใน PostgreSQL 15+ ที่ default ไม่ได้ให้สิทธิ์ `CREATE` แก่ `PUBLIC` บน schema `public` อยู่แล้ว ควรใช้พฤติกรรมนี้ให้เป็นประโยชน์ อย่าคืนสิทธิ์นั้นกลับไปโดยไม่จำเป็น

### 4. Checklist สำหรับออกแบบ Schema โปรเจกต์ใหม่

1. กำหนดโดเมนธุรกิจหลักของระบบ แล้วร่างรายชื่อ schema ที่สอดคล้องกัน (ไม่ควรเกิน 5-10 schema สำหรับโปรเจกต์ขนาดกลาง มิฉะนั้นจะจัดการยาก)
2. ตัดสินใจ naming convention ตั้งแต่ต้น และเขียนเป็นเอกสารให้ทีมทุกคนใช้ตรงกัน
3. วางแผนสิทธิ์ (ownership/grant) ตาม schema เช่น service account ของแต่ละ module ควรมีสิทธิ์เฉพาะ schema ของตัวเอง ไม่ควรมีสิทธิ์ทั่วทั้ง database
4. ตั้ง `search_path` ให้ role ของแอปพลิเคชันชี้ไปยัง schema หลักที่ใช้งานบ่อยที่สุดก่อน เพื่อให้เขียน query สั้นกระชับ แต่ query ที่สำคัญ (เช่นใน migration script หรือ cross-schema join) ควรระบุชื่อ schema แบบเต็มเสมอเพื่อความชัดเจน
5. เตรียม schema สำหรับ audit/logging ตั้งแต่ต้น ไม่ควรผัดวันไปเพิ่มทีหลังเมื่อระบบใหญ่ขึ้นแล้ว
6. ทบทวนโครงสร้าง schema เป็นระยะเมื่อระบบเติบโต — โครงสร้างที่ดีตอนเริ่มต้นอาจไม่เหมาะสมอีกต่อไปเมื่อทีมและข้อมูลขยายตัว

---

## สรุปท้ายบท

ใน Part นี้เราได้เรียนรู้โครงสร้างพื้นฐานที่สุดของ PostgreSQL ตั้งแต่ระดับบนสุดลงไปถึงระดับล่างสุด:

- **Database Cluster** คือกลุ่มของ database ทั้งหมดที่ถูกจัดการโดย PostgreSQL server instance เดียวกัน สร้างขึ้นด้วย `initdb` ซึ่งเป็นความหมายเฉพาะของ PostgreSQL ที่ต่างจาก "cluster" แบบ high-availability ที่พบในระบบอื่น
- ลำดับชั้นโครงสร้างคือ **Cluster → Database → Schema → Table/View/Function/...** โดยแต่ละ database แยกจากกันสมบูรณ์ ส่วน schema เป็นเพียง namespace ย่อยภายใน database เดียวกัน
- Cluster หนึ่งชุดมี database เริ่มต้น 3 ตัวคือ **postgres** (สำหรับงานดูแลระบบ), **template0** (ต้นแบบที่แช่แข็ง), **template1** (ต้นแบบที่ปรับแต่งได้ และเป็นค่า default เวลา `CREATE DATABASE`)
- **Schema** ช่วยจัดกลุ่มออบเจ็กต์ หลีกเลี่ยงชื่อชนกัน และควบคุมสิทธิ์เป็นกลุ่มได้ง่ายขึ้น โดย schema เริ่มต้นคือ `public`
- **search_path** ควบคุมลำดับการค้นหา schema เวลาอ้างอิงออบเจ็กต์แบบไม่ระบุ schema นำหน้า
- ออบเจ็กต์หลายประเภท (Table, View, Sequence, Index, Function) ล้วนอาศัยอยู่ภายใน schema
- **System Catalog** (`pg_catalog`) และ **`information_schema`** คือกลไกที่ทำให้เรา query metadata ของฐานข้อมูลได้ด้วย SQL ธรรมดา ทำให้ PostgreSQL โปร่งใสและตรวจสอบได้ในทุกมิติ
- ทุกออบเจ็กต์มี **เจ้าของ (owner)** เสมอ และ `current_user`/`session_user` ช่วยแยกแยะระหว่าง identity ที่ login เข้ามากับ identity ที่ใช้ตรวจสอบสิทธิ์ ณ ขณะทำงาน
- การออกแบบ schema ที่ดีต้องคำนึงถึงการแบ่งตามโดเมนธุรกิจ, naming convention ที่สม่ำเสมอ, และหลักการ separation of concerns

ความเข้าใจโครงสร้างนี้เป็นรากฐานสำคัญที่จะทำให้ Part ถัดไป ซึ่งพูดถึงการบริหารจัดการฐานข้อมูลในเชิงปฏิบัติ (การสร้าง เปลี่ยนแปลง สำรองข้อมูล) เข้าใจได้ง่ายและมีบริบทที่ชัดเจนยิ่งขึ้น

---

## แบบฝึกหัด

**1.** จงอธิบายว่าทำไม "Database Cluster" ใน PostgreSQL จึงมีความหมายต่างจากคำว่า "cluster" ที่ใช้ในบริบท high-availability

**2.** เขียนคำสั่ง SQL เพื่อสร้าง database ชื่อ `company_db` โดยกำหนด owner เป็น `hr_admin`, encoding เป็น `UTF8`, และสร้างจาก `template0`

**3.** จงอธิบายความแตกต่างระหว่าง `template0` และ `template1` พร้อมยกตัวอย่างสถานการณ์ที่ควรใช้แต่ละตัว

**4.** เขียนคำสั่งสร้าง schema ชื่อ `inventory` แล้วสร้างตาราง `warehouses` ภายใน schema นั้น โดยมีคอลัมน์ `warehouse_id` (primary key แบบ auto-increment) และ `warehouse_name` (ห้ามเป็นค่าว่าง)

**5.** กำหนดให้ `search_path` เป็น `"$user", public` และมี schema ชื่อ `finance_manager` (ตรงกับชื่อ role ที่ login เข้ามา) จงอธิบายว่า PostgreSQL จะค้นหาตาราง `invoices` ที่ไหนก่อนหากเขียน `SELECT * FROM invoices;`

**6.** เขียน query ที่ใช้ `pg_catalog.pg_class` และ `pg_catalog.pg_namespace` เพื่อแสดงรายชื่อ **view** (ไม่ใช่ table) ทั้งหมดในฐานข้อมูล ไม่รวม schema ระบบ (`pg_catalog`, `information_schema`)

**7.** เขียน query โดยใช้ `information_schema.columns` เพื่อแสดงชื่อคอลัมน์และชนิดข้อมูลทั้งหมดของตาราง `inventory.warehouses` ที่สร้างในข้อ 4

**8.** จงอธิบายความแตกต่างระหว่าง `current_user` และ `session_user` พร้อมยกตัวอย่างสถานการณ์ที่ค่าทั้งสองแตกต่างกัน

**9.** สมมติว่าคุณกำลังออกแบบฐานข้อมูลสำหรับระบบ e-commerce ที่มีโมดูลหลัก 4 ส่วนคือ สินค้า (product), คำสั่งซื้อ (order), การชำระเงิน (payment), และรายงาน (reporting) จงเสนอโครงสร้าง schema ที่เหมาะสม พร้อมเหตุผล

**10.** จงอธิบายว่าเหตุใดตั้งแต่ PostgreSQL 15 เป็นต้นไป การให้สิทธิ์สร้างตารางใน schema `public` จึงเปลี่ยนแปลงไปจากเดิม และมีผลดีต่อความปลอดภัยอย่างไร

---

### เฉลยแบบฝึกหัด

**1.** ใน PostgreSQL คำว่า "Database Cluster" หมายถึงกลุ่มของ database ทั้งหมดที่ถูกจัดการโดย PostgreSQL server instance เดียว (postmaster process เดียว) ซึ่งเก็บอยู่ใน data directory เดียวกัน สร้างขึ้นด้วยคำสั่ง `initdb` เพียงครั้งเดียว ในขณะที่ "cluster" ในบริบท high-availability หมายถึงกลุ่มของ server หลายเครื่อง (เช่น primary + replica หลายตัว) ที่ทำงานร่วมกันเพื่อความพร้อมใช้งานสูง ทั้งสองคำนี้ไม่เกี่ยวข้องกันโดยตรง แม้จะใช้คำว่า "cluster" เหมือนกัน

**2.**
```sql
CREATE DATABASE company_db
    OWNER = hr_admin
    ENCODING = 'UTF8'
    TEMPLATE = template0;
```

**3.** `template0` เป็น database ต้นแบบที่ถูกแช่แข็งไว้ตั้งแต่ initdb ห้ามเชื่อมต่อเข้าไปแก้ไขโดยตรง ใช้เป็น "จุดเริ่มต้นที่สะอาด" เมื่อต้องการสร้าง database ด้วย encoding/locale ที่ต่างจาก template1 หรือเมื่อ template1 ถูกปนเปื้อนด้วยออบเจ็กต์ที่ไม่ต้องการ ส่วน `template1` เป็น database ต้นแบบที่แก้ไขได้และเป็นค่า default เมื่อสร้าง database ใหม่โดยไม่ระบุ template — มีประโยชน์เมื่อต้องการให้ทุก database ใหม่มี extension หรือออบเจ็กต์มาตรฐานติดมาด้วยโดยอัตโนมัติ ควรใช้ `template0` เมื่อต้องการ database ที่ "สะอาดล้วนๆ" หรือเปลี่ยน encoding/locale และใช้ `template1` เมื่อต้องการให้มีสิ่งที่ปรับแต่งไว้ล่วงหน้าติดมาด้วย

**4.**
```sql
CREATE SCHEMA inventory;

CREATE TABLE inventory.warehouses (
    warehouse_id SERIAL PRIMARY KEY,
    warehouse_name TEXT NOT NULL
);
```

**5.** PostgreSQL จะค้นหา `invoices` ใน schema ที่ชื่อตรงกับ role ที่ login เข้ามาก่อน (ในที่นี้คือ schema `finance_manager` เพราะ `"$user"` ใน search_path จะถูกแทนที่ด้วยชื่อ role ปัจจุบัน) หากพบตาราง `invoices` ใน schema `finance_manager` ระบบจะใช้ตัวนั้น หากไม่พบจึงจะค้นหาต่อใน schema `public` ตามลำดับใน search_path

**6.**
```sql
SELECT n.nspname AS schema_name, c.relname AS view_name
FROM pg_catalog.pg_class c
JOIN pg_catalog.pg_namespace n ON n.oid = c.relnamespace
WHERE c.relkind = 'v'
  AND n.nspname NOT IN ('pg_catalog', 'information_schema')
ORDER BY n.nspname, c.relname;
```

**7.**
```sql
SELECT column_name, data_type
FROM information_schema.columns
WHERE table_schema = 'inventory'
  AND table_name = 'warehouses'
ORDER BY ordinal_position;
```

**8.** `session_user` คือ role ที่ใช้ login เข้ามาจริงตอนเริ่มต้น connection และจะไม่เปลี่ยนแปลงตลอดอายุ session ส่วน `current_user` คือ role ที่กำลังถูกใช้ตรวจสอบสิทธิ์ ณ ขณะนั้น ซึ่งสามารถเปลี่ยนแปลงชั่วคราวได้ด้วยคำสั่ง `SET ROLE` หรือเมื่อรันฟังก์ชันแบบ `SECURITY DEFINER` ตัวอย่างเช่น หาก login มาด้วย role `app_admin` แล้วรัน `SET ROLE app_readonly;` ค่า `session_user` จะยังคงเป็น `app_admin` แต่ `current_user` จะเปลี่ยนเป็น `app_readonly` และสิทธิ์การเข้าถึงข้อมูลในขณะนั้นจะถูกตรวจสอบตาม `current_user`

**9.** สามารถออกแบบ schema แยกตามโดเมนธุรกิจ เช่น `product` (เก็บ catalog สินค้า), `order` (เก็บคำสั่งซื้อ), `payment` (เก็บข้อมูลการชำระเงิน ซึ่งมักต้องการการควบคุมสิทธิ์ที่เข้มงวดกว่าโมดูลอื่น), และ `reporting` (เก็บ materialized view/ตารางสรุปสำหรับรายงาน แยกออกจาก transactional data เพื่อไม่ให้ query รายงานหนักๆ กระทบ performance ของระบบหลัก) การแยกเช่นนี้ทำให้ควบคุมสิทธิ์เป็นรายโมดูลได้ง่าย (เช่น service account ของ payment ควรมีสิทธิ์เฉพาะ schema `payment` เท่านั้น) และทำให้โครงสร้างฐานข้อมูลอ่านเข้าใจง่ายเมื่อระบบขยายใหญ่ขึ้น

**10.** ก่อน PostgreSQL 15 ทุก role (ผ่าน `PUBLIC` ซึ่งหมายถึงทุกคน) มีสิทธิ์ `CREATE` บน schema `public` โดยอัตโนมัติ ทำให้ผู้ใช้ใดๆ ที่เชื่อมต่อ database ได้ก็สามารถสร้างตารางหรือออบเจ็กต์อื่นใน `public` ได้ทันที ซึ่งอาจนำไปสู่ความเสี่ยงด้านความปลอดภัย เช่น การสร้างออบเจ็กต์ที่เป็นอันตรายหรือชื่อชนกับของระบบ (schema poisoning) ตั้งแต่ PostgreSQL 15 เป็นต้นไป database ใหม่จะไม่ให้สิทธิ์ `CREATE` แก่ `PUBLIC` บน `public` โดยอัตโนมัติอีกต่อไป (ยังคงให้สิทธิ์ `USAGE` เพื่อให้ query ข้อมูลที่มีอยู่ได้) ทำให้ผู้ดูแลระบบต้อง `GRANT` สิทธิ์ `CREATE` อย่างชัดเจนให้เฉพาะ role ที่ต้องการเท่านั้น ซึ่งช่วยลดความเสี่ยงด้านความปลอดภัยได้อย่างมีนัยสำคัญ

---

**บทถัดไป:** [Part 005: การบริหารจัดการฐานข้อมูล (Database Management)](./part-005-database-management.md)
