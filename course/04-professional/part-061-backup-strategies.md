# Part 061: Backup Strategies: pg_dump, pg_dumpall, pg_basebackup

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับมืออาชีพ | Part 061

ยินดีต้อนรับเข้าสู่ **ระดับมืออาชีพ (Professional)** ของหลักสูตรนี้ นับจากบทนี้เป็นต้นไป เราจะเปลี่ยนมุมมองจาก "การเขียนโปรแกรมที่คุยกับฐานข้อมูล" ไปสู่ "การดูแลรักษาฐานข้อมูลให้มีชีวิตอยู่ได้อย่างปลอดภัย" หรือที่เรียกว่า **Database Administration (DBA)** และ **Database Operations (DBOps)**

ทักษะแรกที่ DBA มืออาชีพทุกคนต้องรู้ลึกและรู้จริงคือ **Backup** เพราะไม่ว่าคุณจะออกแบบ schema ได้สวยงามแค่ไหน, เขียน query ได้เร็วเพียงใด, หรือ tune performance ได้ดีเยี่ยมเพียงไร — ถ้าข้อมูลหายแล้วกู้คืนไม่ได้ ทุกอย่างที่ทำมาไม่มีความหมายเลย

บทนี้จะพาคุณไปรู้จักเครื่องมือ backup หลักของ PostgreSQL ทั้งสามตัว ได้แก่ `pg_dump`, `pg_dumpall`, และ `pg_basebackup` อย่างละเอียด พร้อมตัวอย่างคำสั่งจริงที่ใช้กับฐานข้อมูลระบบ e-commerce (`ecommerce_prod`) ซึ่งมีตาราง `products`, `orders`, `customers`, `order_items`, `categories` เป็นต้น

---

## เป้าหมายการเรียนรู้

เมื่อเรียนจบบทนี้ คุณจะสามารถ:

1. อธิบายความสำคัญของ backup และแนวคิด RPO/RTO ที่ใช้ออกแบบกลยุทธ์ backup ได้
2. ใช้ `pg_dump` สำรองข้อมูลฐานข้อมูลเดียวได้ทั้ง 4 รูปแบบ (plain, custom, directory, tar)
3. ใช้ตัวเลือกขั้นสูงของ `pg_dump` เช่น backup เฉพาะตาราง/schema, การ exclude ตาราง, และการตั้งค่า compression
4. ใช้ `pg_restore` กู้คืนข้อมูลจาก custom/directory format รวมถึงการทำ parallel restore ด้วย `-j`
5. ใช้ `pg_dumpall` สำรอง cluster ทั้งหมดรวมถึง roles และ tablespaces ที่ `pg_dump` ทำไม่ได้
6. เปรียบเทียบข้อดี-ข้อเสียของ Logical backup (`pg_dump`) กับ Physical backup (`pg_basebackup`)
7. ใช้ `pg_basebackup` ทำ physical backup ทั้ง data directory พร้อมตัวเลือกสำคัญ
8. เข้าใจว่าทำไมต้องทดสอบ restore backup เป็นประจำ และวิธีทดสอบอย่างเป็นระบบ
9. ออกแบบนโยบาย retention แบบ 3-2-1 rule และเก็บ backup หลายช่วงเวลา
10. เขียน backup script อัตโนมัติด้วย bash + cron สำหรับระบบ production จริง

---

## Step 601: ทำไม Backup สำคัญ — ภาพรวมกลยุทธ์ backup

### ความเสี่ยงที่ backup ต้องป้องกัน

ฐานข้อมูล production ของระบบ e-commerce เก็บข้อมูลที่สำคัญที่สุดขององค์กร ทั้งคำสั่งซื้อ, ข้อมูลลูกค้า, ประวัติการชำระเงิน ความเสี่ยงที่อาจทำให้ข้อมูลเสียหายหรือสูญหายมีหลายรูปแบบ:

| ประเภทความเสี่ยง | ตัวอย่าง | Backup ช่วยได้อย่างไร |
|---|---|---|
| **Hardware failure** | Disk เสีย, RAID controller พัง, เซิร์ฟเวอร์ไฟไหม้ | กู้คืนจาก backup ที่เก็บแยกที่เก็บข้อมูลจริง |
| **Human error** | พนักงานรัน `DELETE FROM orders;` โดยไม่มี `WHERE` | กู้คืนข้อมูล ณ เวลาก่อนเกิดเหตุ |
| **Software bug** | Deploy โค้ดใหม่แล้วมี bug เขียนข้อมูลผิด | Point-in-time recovery ย้อนกลับไปก่อน bug ทำงาน |
| **Malicious attack** | Ransomware เข้ารหัสข้อมูล, insider ลบข้อมูลโดยเจตนา | Backup ที่เก็บแยก network (offline/immutable) |
| **Data corruption** | Bit rot, filesystem corruption | Backup ที่ผ่านการ verify ความถูกต้อง |
| **ภัยธรรมชาติ** | Datacenter ถูกน้ำท่วม, แผ่นดินไหว | Backup ที่เก็บคนละภูมิภาค (offsite) |

> **หลักการสำคัญ**: "Backup ไม่ใช่ทางเลือก แต่เป็นข้อบังคับ" ระบบ production ที่ไม่มี backup ไม่ใช่แค่เสี่ยง แต่คือ "ระเบิดเวลา" ที่รอวันระเบิด

### แนวคิด RPO และ RTO

ก่อนออกแบบกลยุทธ์ backup ต้องตอบคำถามสำคัญสองข้อก่อนเสมอ:

**RPO (Recovery Point Objective)** — "เรายอมรับให้ข้อมูลหายไปได้มากที่สุดกี่นาที/ชั่วโมง?"

ตัวอย่าง: ถ้า RPO = 1 ชั่วโมง หมายความว่าหากระบบล่มตอน 14:35 น. เราต้องกู้คืนข้อมูลได้อย่างน้อยถึงเวลา 13:35 น. (ข้อมูลระหว่าง 13:35-14:35 อาจหายไปได้)

**RTO (Recovery Time Objective)** — "เรายอมรับให้ระบบ down ได้นานที่สุดกี่นาที/ชั่วโมงก่อนต้องกลับมาใช้งานได้?"

ตัวอย่าง: ถ้า RTO = 30 นาที หมายความว่าเมื่อเกิดเหตุ เราต้องกู้ระบบให้กลับมาใช้งานได้ภายใน 30 นาที

```
Timeline ตัวอย่าง:

  Backup ล่าสุด          ระบบล่ม            กู้คืนเสร็จ
       │                    │                    │
       ▼                    ▼                    ▼
───────●────────────────────●────────────────────●──────► เวลา
       │◄──── RPO Window ──►│◄──── RTO ─────────►│
       │  (ข้อมูลที่อาจหาย)   │   (เวลาที่ระบบ down)  │
```

ตารางด้านล่างแสดงว่า RPO/RTO ที่ต้องการ ส่งผลต่อการเลือกกลยุทธ์ backup อย่างไร:

| ความต้องการ | RPO | RTO | กลยุทธ์ที่เหมาะสม |
|---|---|---|---|
| เว็บบล็อกส่วนตัว | 24 ชม. | หลายชม. | `pg_dump` ทุกวัน |
| ระบบ e-commerce ขนาดกลาง | 1 ชม. | 1 ชม. | `pg_basebackup` รายวัน + WAL archiving (PITR) |
| ระบบธนาคาร/การเงิน | วินาที | นาที | Streaming replication + PITR + synchronous commit |
| ระบบที่ห้าม down เด็ดขาด | 0 | 0 | Multi-region replication + automatic failover |

บทนี้จะเน้นเครื่องมือพื้นฐาน (`pg_dump`, `pg_dumpall`, `pg_basebackup`) ส่วนเรื่อง WAL archiving และ Point-in-Time Recovery (PITR) ที่ช่วยให้ RPO ต่ำลงมากจะอยู่ใน **Part 062**

### สามเสาหลักของกลยุทธ์ backup

1. **Logical backup** — export ข้อมูลออกมาเป็นคำสั่ง SQL หรือ archive format (`pg_dump`, `pg_dumpall`)
2. **Physical backup** — คัดลอกไฟล์ data directory ทั้งหมดในระดับ byte (`pg_basebackup`)
3. **Continuous archiving (WAL)** — เก็บ Write-Ahead Log ต่อเนื่องเพื่อทำ PITR (Part 062)

บทนี้ครอบคลุมข้อ 1 และ 2 อย่างละเอียด

### เช็คเวอร์ชันก่อนเริ่มต้น

```bash
psql --version
# psql (PostgreSQL) 17.2

pg_dump --version
# pg_dump (PostgreSQL) 17.2

pg_basebackup --version
# pg_basebackup (PostgreSQL) 17.2
```

> **คำเตือน**: `pg_dump`/`pg_restore` ควรใช้เวอร์ชันเดียวกับหรือ**ใหม่กว่า**เซิร์ฟเวอร์ที่ backup เสมอ ถ้าใช้ client เวอร์ชันเก่ากว่า server อาจ dump ข้อมูลได้ไม่ครบหรือ error

---

## Step 602: pg_dump พื้นฐาน — backup database เดียว, format ต่างๆ

`pg_dump` คือเครื่องมือหลักในการทำ **logical backup** ของฐานข้อมูลหนึ่งฐาน (single database) มันจะอ่านโครงสร้าง (schema) และข้อมูล (data) แล้วแปลงเป็นไฟล์ output ตามฟอร์แมตที่เลือก

### รูปแบบ output ทั้ง 4 แบบของ pg_dump

| Format | Flag | นามสกุลไฟล์ทั่วไป | ใช้ pg_restore ได้ไหม | Parallel dump/restore |
|---|---|---|---|---|
| Plain SQL | `-Fp` (default) | `.sql` | ไม่ได้ (ใช้ `psql` แทน) | ไม่ได้ |
| Custom | `-Fc` | `.dump` / `.backup` | ได้ | Restore ได้ (`-j`) |
| Directory | `-Fd` | โฟลเดอร์ | ได้ | Dump และ Restore ได้ (`-j`) |
| Tar | `-Ft` | `.tar` | ได้ | Restore ได้ (`-j`) |

### 1. Plain SQL format

ผลลัพธ์เป็นไฟล์ `.sql` ที่อ่านได้ด้วยตาเปล่า ประกอบด้วยคำสั่ง `CREATE TABLE`, `INSERT`, `COPY` ฯลฯ

```bash
pg_dump -h localhost -U postgres -d ecommerce_prod -f ecommerce_prod_$(date +%Y%m%d).sql
```

```
Password:
```

ตรวจสอบผลลัพธ์:

```bash
head -n 30 ecommerce_prod_20260925.sql
```

```
--
-- PostgreSQL database dump
--

-- Dumped from database version 17.2
-- Dumped by pg_dump version 17.2

SET statement_timeout = 0;
SET lock_timeout = 0;
SET idle_in_transaction_session_timeout = 0;
SET client_encoding = 'UTF8';
SET standard_conforming_strings = on;
SELECT pg_catalog.set_config('search_path', '', false);
SET check_function_bodies = false;
SET xmloption = content;
SET client_min_messages = warning;
SET row_security = off;

--
-- Name: ecommerce_prod; Type: DATABASE PROPERTIES; Schema: -; Owner: postgres
--

CREATE TABLE public.categories (
    category_id integer NOT NULL,
    ...
```

ข้อดีของ plain format คือ **อ่านและแก้ไขได้ด้วยตาเปล่า** เหมาะกับฐานข้อมูลเล็กหรือใช้ตรวจสอบ diff ของ schema แต่ข้อเสียคือไฟล์ใหญ่ (ไม่บีบอัด default), restore ได้ช้าเพราะเป็น sequential SQL statement เท่านั้น

Restore plain format ทำผ่าน `psql` (ไม่ใช่ `pg_restore`):

```bash
createdb -h localhost -U postgres ecommerce_restore_test
psql -h localhost -U postgres -d ecommerce_restore_test -f ecommerce_prod_20260925.sql
```

### 2. Custom format (แนะนำเป็นค่าเริ่มต้นสำหรับงานส่วนใหญ่)

Custom format เป็น binary format ที่ถูกบีบอัดอัตโนมัติ และสามารถใช้กับ `pg_restore` เพื่อเลือก restore เฉพาะบางส่วนได้ (เช่น เฉพาะตาราง หรือเฉพาะ schema)

```bash
pg_dump -h localhost -U postgres -d ecommerce_prod \
  -Fc \
  -f ecommerce_prod_$(date +%Y%m%d).dump
```

```
Password:
```

ตรวจสอบขนาดไฟล์เทียบกับ plain format:

```bash
ls -lh ecommerce_prod_20260925.*
```

```
-rw-r--r-- 1 postgres postgres  842M ecommerce_prod_20260925.sql
-rw-r--r-- 1 postgres postgres  156M ecommerce_prod_20260925.dump
```

จะเห็นว่า custom format เล็กกว่าอย่างเห็นได้ชัดเพราะบีบอัดข้อมูลด้วย gzip โดย default (compression level 6 ใน PostgreSQL รุ่นเก่า หรือใช้ zlib/gzip level ปรับได้ใน PG16+)

ดูรายละเอียดสิ่งที่อยู่ในไฟล์ dump โดยไม่ต้อง restore จริง ด้วย `pg_restore -l` (list):

```bash
pg_restore -l ecommerce_prod_20260925.dump | head -n 20
```

```
;
; Archive created at 2026-09-25 09:12:44 UTC
;     dbname: ecommerce_prod
;     TOC Entries: 48
;     Compression: gzip
;     Dump Version: 1.15-0
;     Format: CUSTOM
;     Integer: 4 bytes
;     Offset: 8 bytes
;     Dumped from database version: 17.2
;     Dumped by pg_dump version: 17.2
;
;
; Selected TOC Entries:
;
3; 2615 2200 SCHEMA - public postgres
217; 1259 24580 TABLE public categories postgres
218; 1259 24589 TABLE public products postgres
219; 1259 24601 TABLE public customers postgres
220; 1259 24615 TABLE public orders postgres
221; 1259 24630 TABLE public order_items postgres
```

### 3. Directory format

Directory format เก็บแต่ละตารางเป็นไฟล์แยกกันในโฟลเดอร์ ข้อดีคือ **สามารถ dump แบบ parallel ได้** (`-j`) ซึ่งเร็วกว่ามากสำหรับฐานข้อมูลขนาดใหญ่ที่มีหลายตาราง

```bash
mkdir -p /backup/ecommerce_prod_20260925_dir

pg_dump -h localhost -U postgres -d ecommerce_prod \
  -Fd \
  -j 4 \
  -f /backup/ecommerce_prod_20260925_dir
```

```
Password:
```

โครงสร้างไฟล์ที่ได้:

```bash
ls -la /backup/ecommerce_prod_20260925_dir
```

```
total 165000
drwxr-xr-x 2 postgres postgres     4096 Sep 25 09:15 .
drwxr-xr-x 3 postgres postgres     4096 Sep 25 09:15 ..
-rw-r--r-- 1 postgres postgres    28450 Sep 25 09:15 toc.dat
-rw-r--r-- 1 postgres postgres 45120000 Sep 25 09:15 3245.dat.gz
-rw-r--r-- 1 postgres postgres 62300000 Sep 25 09:15 3246.dat.gz
-rw-r--r-- 1 postgres postgres 12400000 Sep 25 09:15 3247.dat.gz
-rw-r--r-- 1 postgres postgres 33900000 Sep 25 09:15 3248.dat.gz
```

`toc.dat` คือ Table of Contents ที่บอกว่าไฟล์ไหนคือตารางอะไร ส่วนไฟล์ `.dat.gz` คือข้อมูลของแต่ละตาราง (compressed)

> **Directory format คือตัวเลือกที่แนะนำที่สุดสำหรับฐานข้อมูลขนาดใหญ่ใน production** เพราะรองรับทั้ง parallel dump และ parallel restore

### 4. Tar format

Tar format คล้าย custom format แต่ห่อด้วย tar archive มาตรฐาน ข้อจำกัดคือ**ไม่รองรับการบีบอัดในตัว**และ**ไม่รองรับ parallel dump** (แต่ restore แบบ parallel ได้)

```bash
pg_dump -h localhost -U postgres -d ecommerce_prod \
  -Ft \
  -f ecommerce_prod_$(date +%Y%m%d).tar
```

ปัจจุบัน tar format ใช้งานน้อยลงมาก เพราะ custom format และ directory format ทำได้ทุกอย่างที่ tar format ทำได้ และดีกว่า จึงแนะนำให้เลือกใช้ **custom** หรือ **directory** format เป็นหลัก

### ตัวเลือกพื้นฐานที่ใช้บ่อย

```bash
pg_dump \
  -h localhost \      # host ของเซิร์ฟเวอร์
  -p 5432 \            # port
  -U postgres \        # username
  -d ecommerce_prod \  # ชื่อฐานข้อมูล
  -Fc \                # format
  -v \                 # verbose (แสดง progress)
  -f backup.dump
```

ตัวอย่างผล output เมื่อใช้ `-v`:

```
pg_dump: last built-in OID is 16383
pg_dump: reading extensions
pg_dump: identifying extension members
pg_dump: reading schemas
pg_dump: reading user-defined tables
pg_dump: reading user-defined functions
pg_dump: reading user-defined types
pg_dump: reading indexes
pg_dump: reading constraints
pg_dump: reading triggers
pg_dump: reading dumping out the contents of table "orders"
pg_dump: dumping contents of table "public.orders"
pg_dump: dumping contents of table "public.order_items"
pg_dump: dumping contents of table "public.products"
```

### ใช้ environment variable แทนการพิมพ์รหัสผ่านทุกครั้ง

การพิมพ์รหัสผ่านซ้ำๆ ไม่เหมาะกับการทำ script อัตโนมัติ วิธีมาตรฐานคือใช้ไฟล์ `.pgpass`:

```bash
cat >> ~/.pgpass <<'EOF'
localhost:5432:ecommerce_prod:postgres:S3cur3P@ssw0rd
EOF
chmod 600 ~/.pgpass
```

```
# รูปแบบ: hostname:port:database:username:password
```

หลังจากนี้ `pg_dump` จะไม่ถาม password อีก (ตราบใดที่ host/port/database/user ตรงกับบรรทัดใน `.pgpass`)

> **คำเตือนความปลอดภัย**: ไฟล์ `.pgpass` ต้องมี permission `600` เท่านั้น (เจ้าของอ่าน/เขียนได้คนเดียว) ไม่เช่นนั้น PostgreSQL จะปฏิเสธการใช้งานไฟล์นี้ทันที

---

## Step 603: pg_dump ขั้นสูง — เฉพาะตาราง, เฉพาะ schema, exclude table, compression

เมื่อฐานข้อมูลมีขนาดใหญ่มาก การ dump ทั้งฐานข้อมูลทุกครั้งอาจไม่จำเป็นหรือใช้เวลานานเกินไป `pg_dump` มีตัวเลือกให้เลือก backup เฉพาะบางส่วนได้

### Backup เฉพาะตาราง (`-t` / `--table`)

```bash
pg_dump -h localhost -U postgres -d ecommerce_prod \
  -t public.orders \
  -t public.order_items \
  -Fc \
  -f orders_only_$(date +%Y%m%d).dump
```

รองรับ wildcard pattern ด้วย (ใช้เครื่องหมาย `*`):

```bash
# backup ทุกตารางที่ขึ้นต้นด้วย "order"
pg_dump -h localhost -U postgres -d ecommerce_prod \
  -t 'public.order*' \
  -Fc \
  -f orders_wildcard_$(date +%Y%m%d).dump
```

```
pg_dump: dumping contents of table "public.orders"
pg_dump: dumping contents of table "public.order_items"
pg_dump: dumping contents of table "public.order_status_history"
```

> **หมายเหตุ**: ถ้าชื่อ pattern มีเครื่องหมาย `*` ต้องใส่ single quote เสมอ เพื่อกันไม่ให้ shell ตีความ wildcard เองก่อนส่งให้ `pg_dump`

### Backup เฉพาะ schema (`-n` / `--schema`)

สมมติระบบ e-commerce แยก schema สำหรับ reporting ออกจาก schema หลัก:

```bash
pg_dump -h localhost -U postgres -d ecommerce_prod \
  -n public \
  -n reporting \
  -Fc \
  -f schemas_selected_$(date +%Y%m%d).dump
```

รองรับ pattern เช่นกัน:

```bash
# backup ทุก schema ที่ขึ้นต้นด้วย "audit_"
pg_dump -h localhost -U postgres -d ecommerce_prod \
  -n 'audit_*' \
  -Fc \
  -f audit_schemas_$(date +%Y%m%d).dump
```

### Exclude ตาราง (`-T` / `--exclude-table`)

บางครั้งมีตารางขนาดใหญ่ที่ไม่จำเป็นต้อง backup ทุกวัน เช่น ตาราง log หรือ ตาราง audit_trail ที่มีข้อมูลมหาศาลแต่ไม่สำคัญเท่าตาราง orders

```bash
pg_dump -h localhost -U postgres -d ecommerce_prod \
  -T public.audit_log \
  -T public.session_events \
  -Fc \
  -f ecommerce_no_logs_$(date +%Y%m%d).dump
```

### Exclude เฉพาะข้อมูล แต่เก็บโครงสร้างตาราง (`--exclude-table-data`)

บางครั้งต้องการเก็บ **โครงสร้าง** ของตาราง log ไว้ (เผื่อ restore แล้วให้แอปทำงานได้ปกติ) แต่ไม่ต้องการ backup ข้อมูลที่มีขนาดใหญ่มากในตารางนั้น:

```bash
pg_dump -h localhost -U postgres -d ecommerce_prod \
  --exclude-table-data=public.audit_log \
  --exclude-table-data='public.event_*' \
  -Fc \
  -f ecommerce_structure_kept_$(date +%Y%m%d).dump
```

ผลลัพธ์: `pg_dump` จะสร้างคำสั่ง `CREATE TABLE audit_log (...)` แต่จะไม่มีข้อมูล (`COPY` เปล่า) ในไฟล์ backup

### Backup เฉพาะโครงสร้าง (schema-only) หรือเฉพาะข้อมูล (data-only)

```bash
# เฉพาะโครงสร้าง (ไม่มีข้อมูล) — เหมาะสำหรับสร้าง staging/dev environment
pg_dump -h localhost -U postgres -d ecommerce_prod \
  --schema-only \
  -f ecommerce_schema_only.sql

# เฉพาะข้อมูล (ไม่มี DDL) — เหมาะสำหรับ migrate ข้อมูลเข้าฐานข้อมูลที่มี schema อยู่แล้ว
pg_dump -h localhost -U postgres -d ecommerce_prod \
  --data-only \
  -Fc \
  -f ecommerce_data_only.dump
```

### การตั้งค่า Compression level

Custom และ directory format รองรับการปรับระดับการบีบอัดด้วย `-Z` / `--compress`:

```bash
# ไม่บีบอัดเลย (เร็วที่สุด แต่ไฟล์ใหญ่ที่สุด) — เหมาะเมื่อ disk I/O เร็วแต่ CPU จำกัด
pg_dump -h localhost -U postgres -d ecommerce_prod -Fc -Z 0 -f fast_nozip.dump

# บีบอัดสูงสุด (ไฟล์เล็กที่สุด แต่ใช้ CPU และเวลามากที่สุด)
pg_dump -h localhost -U postgres -d ecommerce_prod -Fc -Z 9 -f small_maxzip.dump
```

ตั้งแต่ PostgreSQL 16 เป็นต้นไป สามารถระบุ **compression method** ได้ด้วย ไม่จำกัดแค่ gzip (ถ้า build มากับ `lz4` หรือ `zstd`):

```bash
# ใช้ zstd compression level 5 (เร็วกว่า gzip มาก ที่ ratio ใกล้เคียงกัน)
pg_dump -h localhost -U postgres -d ecommerce_prod \
  -Fc \
  --compress=zstd:5 \
  -f ecommerce_zstd_$(date +%Y%m%d).dump
```

```
pg_dump: error: unrecognized compression algorithm "zstd"
```

> หากเจอ error ด้านบน แปลว่า build ของ PostgreSQL ในเครื่องไม่ได้ compile พร้อม `--with-zstd` ให้ตรวจสอบด้วย `pg_config --configure` หรือใช้ `gzip`/`lz4` แทน

เปรียบเทียบเวลาและขนาดไฟล์ที่ compression level ต่างกัน (ตัวอย่างจากฐานข้อมูล ecommerce_prod ขนาด ~8GB):

| Compression | เวลาที่ใช้ | ขนาดไฟล์ |
|---|---|---|
| `-Z 0` (none) | 42 วินาที | 8.1 GB |
| `-Z 1` | 58 วินาที | 2.3 GB |
| `-Z 6` (default) | 1 นาที 40 วินาที | 1.6 GB |
| `-Z 9` (max) | 4 นาที 12 วินาที | 1.5 GB |
| `--compress=zstd:5` | 51 วินาที | 1.6 GB |

**ข้อสรุป**: `-Z 1` มักคุ้มค่าที่สุดในทางปฏิบัติ เพราะลดขนาดไฟล์ได้เยอะแล้วโดยใช้เวลาเพิ่มขึ้นไม่มาก ส่วน `-Z 9` มักไม่คุ้ม เพราะใช้เวลาเพิ่มขึ้นมากแต่ขนาดไฟล์ลดลงเล็กน้อยเท่านั้น

### รวมตัวเลือกขั้นสูงในสถานการณ์จริง

```bash
# Backup ทุกตารางยกเว้น log/audit ด้วย parallel jobs และ zstd compression
pg_dump -h db-primary.internal -U backup_user -d ecommerce_prod \
  -Fd \
  -j 8 \
  --exclude-table-data='public.audit_*' \
  --exclude-table-data='public.session_events' \
  -f /backup/ecommerce_prod_$(date +%Y%m%d_%H%M%S)
```

---

## Step 604: pg_restore — restore จาก custom/directory format, parallel restore

`pg_restore` ใช้กู้คืนไฟล์ backup ที่อยู่ในฟอร์แมต **custom**, **directory**, หรือ **tar** เท่านั้น (ไม่ใช้กับ plain SQL — ฟอร์แมตนั้นใช้ `psql` แทน)

### Restore ทั้งฐานข้อมูลแบบพื้นฐาน

ขั้นแรกต้องสร้างฐานข้อมูลเปล่าไว้ก่อนเสมอ (`pg_restore` ไม่สร้างฐานข้อมูลให้เอง เว้นแต่ใช้ `-C`):

```bash
createdb -h localhost -U postgres ecommerce_restore_test

pg_restore -h localhost -U postgres \
  -d ecommerce_restore_test \
  -v \
  ecommerce_prod_20260925.dump
```

```
pg_restore: connecting to database for restore
pg_restore: creating SCHEMA "public"
pg_restore: creating TABLE "public.categories"
pg_restore: creating TABLE "public.products"
pg_restore: creating TABLE "public.customers"
pg_restore: creating TABLE "public.orders"
pg_restore: creating TABLE "public.order_items"
pg_restore: processing data for table "public.categories"
pg_restore: processing data for table "public.products"
pg_restore: processing data for table "public.customers"
pg_restore: processing data for table "public.orders"
pg_restore: processing data for table "public.order_items"
pg_restore: creating INDEX "public.idx_orders_customer_id"
pg_restore: creating INDEX "public.idx_order_items_order_id"
pg_restore: creating CONSTRAINT "public.orders_customer_id_fkey"
pg_restore: creating CONSTRAINT "public.order_items_order_id_fkey"
```

### สร้างฐานข้อมูลอัตโนมัติด้วย `-C`

```bash
pg_restore -h localhost -U postgres \
  -d postgres \
  -C \
  -v \
  ecommerce_prod_20260925.dump
```

เมื่อใช้ `-C` (`--create`) จะต้อง connect ผ่านฐานข้อมูลอื่นก่อน (มักใช้ `postgres`) แล้ว `pg_restore` จะรันคำสั่ง `CREATE DATABASE ecommerce_prod` ให้เองโดยอิงชื่อจากตอน dump

### Parallel restore ด้วย `-j`

การ restore แบบ parallel ช่วยลดเวลาได้มากโดยเฉพาะฐานข้อมูลขนาดใหญ่ที่มีหลายตาราง เพราะ `pg_restore` จะสร้างตารางและโหลดข้อมูลหลายตารางพร้อมกัน

```bash
time pg_restore -h localhost -U postgres \
  -d ecommerce_restore_test \
  -j 8 \
  -v \
  /backup/ecommerce_prod_20260925_dir
```

```
pg_restore: connecting to database for restore
pg_restore: creating TABLE "public.categories"
pg_restore: creating TABLE "public.products"
...
pg_restore: processing data for table "public.orders"
pg_restore: processing data for table "public.order_items"
pg_restore: processing data for table "public.products"
pg_restore: processing data for table "public.customers"

real    2m18.412s
user    0m4.221s
sys     0m1.098s
```

เทียบกับ sequential restore:

```bash
time pg_restore -h localhost -U postgres -d ecommerce_restore_test -v ecommerce_prod_20260925.dump
```

```
real    9m47.033s
user    0m6.114s
sys     0m1.532s
```

> **ข้อจำกัดของ `-j`**: parallel restore ใช้ได้เฉพาะกับ **custom** และ **directory** format เท่านั้น (ไม่ใช่ plain SQL หรือ tar สำหรับ dump) นอกจากนี้ index และ constraint จะถูกสร้าง**หลังจาก**โหลดข้อมูลเสร็จทุกตารางเสมอ ไม่ว่าจะ parallel หรือไม่

### Restore เฉพาะบางตาราง

เหมือนกับ `pg_dump`, `pg_restore` ก็มี `-t` สำหรับเลือกเฉพาะตารางที่ต้องการ restore จากไฟล์ backup ที่มีทุกตารางอยู่:

```bash
pg_restore -h localhost -U postgres \
  -d ecommerce_restore_test \
  -t orders \
  -t order_items \
  -v \
  ecommerce_prod_20260925.dump
```

### Restore แบบ list-driven (เลือกเฉพาะบาง object)

ใช้ `-l` เพื่อ list เนื้อหาในไฟล์ backup ออกมาเป็นไฟล์ text แล้วแก้ไข (comment ออกในสิ่งที่ไม่ต้องการ) ก่อนใช้ `-L` restore เฉพาะที่เลือก:

```bash
pg_restore -l ecommerce_prod_20260925.dump > restore_list.txt
```

แก้ไขไฟล์ `restore_list.txt` (comment บรรทัดที่ไม่ต้องการด้วย `;` นำหน้า):

```bash
# แก้ไขให้ restore แค่ตาราง products และ categories
sed -i '/TABLE public orders/s/^/;/' restore_list.txt
sed -i '/TABLE public order_items/s/^/;/' restore_list.txt
sed -i '/TABLE public customers/s/^/;/' restore_list.txt
```

```bash
pg_restore -h localhost -U postgres \
  -d ecommerce_restore_test \
  -L restore_list.txt \
  -v \
  ecommerce_prod_20260925.dump
```

### Restore ทับฐานข้อมูลเดิม (Clean + Restore)

ใช้ `--clean` (`-c`) เพื่อลบ object เดิมก่อน restore ใหม่ทับ (มีประโยชน์เมื่อต้องการ refresh ฐานข้อมูล staging ให้ตรงกับ production):

```bash
pg_restore -h localhost -U postgres \
  -d ecommerce_staging \
  --clean \
  --if-exists \
  -v \
  ecommerce_prod_20260925.dump
```

`--if-exists` ป้องกัน error กรณี object บางตัวยังไม่มีอยู่ตอนสั่ง `DROP`

### จัดการ error ระหว่าง restore

Default พฤติกรรมของ `pg_restore` คือหยุดทันทีเมื่อเจอ error ร้ายแรง แต่จะ**ข้าม error เล็กน้อยแล้วทำงานต่อ** (เช่น object ซ้ำ) ถ้าต้องการให้หยุดทันทีเมื่อเจอ error ใดๆ ให้ใช้ `--exit-on-error`:

```bash
pg_restore -h localhost -U postgres \
  -d ecommerce_restore_test \
  --exit-on-error \
  -v \
  ecommerce_prod_20260925.dump
```

ถ้าต้องการรันทั้งหมดใน transaction เดียว (all-or-nothing) ใช้ `--single-transaction`:

```bash
pg_restore -h localhost -U postgres \
  -d ecommerce_restore_test \
  --single-transaction \
  -v \
  ecommerce_prod_20260925.dump
```

> **ข้อควรระวัง**: `--single-transaction` ใช้ร่วมกับ `-j` (parallel) ไม่ได้ เพราะ parallel jobs ใช้ connection แยกกัน ไม่สามารถอยู่ใน transaction เดียวกันได้

### ตารางสรุปตัวเลือกสำคัญของ pg_restore

| ตัวเลือก | ความหมาย |
|---|---|
| `-d dbname` | ฐานข้อมูลปลายทางที่จะ restore เข้าไป |
| `-C` | สร้างฐานข้อมูลใหม่อัตโนมัติตามชื่อที่ dump ไว้ |
| `-j N` | จำนวน parallel jobs (เฉพาะ custom/directory format) |
| `-t table` | restore เฉพาะตารางที่ระบุ |
| `--clean` / `-c` | DROP object เดิมก่อน restore |
| `--if-exists` | ใช้คู่กับ `--clean` เพื่อไม่ error ถ้า object ไม่มีอยู่ |
| `--no-owner` | ไม่ restore เจ้าของ object เดิม (ใช้ user ปัจจุบันแทน) |
| `--no-privileges` | ไม่ restore สิทธิ์ GRANT/REVOKE เดิม |
| `--schema-only` | restore เฉพาะโครงสร้าง ไม่เอาข้อมูล |
| `--data-only` | restore เฉพาะข้อมูล ไม่เอาโครงสร้าง |
| `-l` | list เนื้อหาในไฟล์ backup โดยไม่ restore จริง |
| `-L file` | restore ตามรายการใน list file |

---

## Step 605: pg_dumpall — backup ทั้ง cluster รวม roles และ tablespaces

`pg_dump` มีข้อจำกัดสำคัญ: **มันสำรองได้แค่ข้อมูลของฐานข้อมูลเดียว** และ**ไม่สำรอง roles/users ระดับ cluster** เช่น `CREATE ROLE`, สิทธิ์ superuser, password hash ของ user ต่างๆ ที่ใช้ร่วมกันทั้ง cluster

`pg_dumpall` ถูกออกแบบมาเพื่ออุดช่องว่างนี้ — มันสำรอง**ทั้ง PostgreSQL cluster** รวมถึง:

- Roles/Users ทั้งหมดพร้อมสิทธิ์และ password hash
- Tablespaces (นิยาม location แต่ไม่รวมไฟล์ข้อมูลจริง)
- ทุกฐานข้อมูลใน cludoster (databases ทั้งหมด)
- Global objects อื่นๆ เช่น `pg_hba.conf` **ไม่ได้** ถูกรวมด้วย (ต้อง backup แยก)

### Backup ทั้ง cluster แบบเต็ม

```bash
pg_dumpall -h localhost -U postgres -f cluster_full_$(date +%Y%m%d).sql
```

```
Password:
```

`pg_dumpall` output เป็น **plain SQL เท่านั้น** (ไม่รองรับ custom/directory format เหมือน `pg_dump`) เพราะฉะนั้นไฟล์ผลลัพธ์จะไม่ถูกบีบอัดโดยอัตโนมัติ นิยมบีบอัดเองด้วย `gzip`:

```bash
pg_dumpall -h localhost -U postgres | gzip > cluster_full_$(date +%Y%m%d).sql.gz
```

```bash
ls -lh cluster_full_20260925.sql.gz
```

```
-rw-r--r-- 1 postgres postgres 312M cluster_full_20260925.sql.gz
```

### Backup เฉพาะ roles (globals)

ในทางปฏิบัติ กลยุทธ์ที่นิยมที่สุดคือ **ใช้ `pg_dump` (custom format) แยก backup แต่ละฐานข้อมูล + ใช้ `pg_dumpall --globals-only` backup แค่ roles/tablespaces** เพราะ `pg_dumpall` แบบเต็มไม่รองรับ parallel และไม่รองรับ compressed format ทำให้ backup ฐานข้อมูลใหญ่ช้ากว่ามาก

```bash
pg_dumpall -h localhost -U postgres \
  --globals-only \
  -f globals_$(date +%Y%m%d).sql
```

```bash
cat globals_20260925.sql
```

```sql
--
-- PostgreSQL database cluster dump
--

SET default_transaction_read_only = off;
SET client_encoding = 'UTF8';
SET standard_conforming_strings = on;

--
-- Roles
--

CREATE ROLE app_readonly;
ALTER ROLE app_readonly WITH NOSUPERUSER INHERIT NOCREATEROLE NOCREATEDB LOGIN NOREPLICATION NOBYPASSRLS PASSWORD 'SCRAM-SHA-256$4096:xxxxxx';
CREATE ROLE backup_user;
ALTER ROLE backup_user WITH NOSUPERUSER INHERIT NOCREATEROLE NOCREATEDB LOGIN REPLICATION NOBYPASSRLS PASSWORD 'SCRAM-SHA-256$4096:yyyyyy';
CREATE ROLE ecommerce_app;
ALTER ROLE ecommerce_app WITH NOSUPERUSER INHERIT NOCREATEROLE NOCREATEDB LOGIN NOREPLICATION NOBYPASSRLS PASSWORD 'SCRAM-SHA-256$4096:zzzzzz';

--
-- Role memberships
--

GRANT app_readonly TO ecommerce_app GRANTED BY postgres;

--
-- Tablespaces
--

CREATE TABLESPACE fast_ssd OWNER postgres LOCATION '/mnt/ssd_data/pg_tbs';

--
-- Databases
--

CREATE DATABASE ecommerce_prod WITH TEMPLATE = template0 ENCODING = 'UTF8' LOCALE_PROVIDER = libc LOCALE = 'en_US.UTF-8';
CREATE DATABASE ecommerce_staging WITH TEMPLATE = template0 ENCODING = 'UTF8' LOCALE_PROVIDER = libc LOCALE = 'en_US.UTF-8';
```

### Backup เฉพาะ tablespaces หรือ roles แยกกัน

```bash
# เฉพาะ roles
pg_dumpall -h localhost -U postgres --roles-only -f roles_only.sql

# เฉพาะ tablespaces
pg_dumpall -h localhost -U postgres --tablespaces-only -f tablespaces_only.sql
```

### กลยุทธ์ backup แบบผสมผสาน (แนะนำสำหรับ production)

```bash
#!/bin/bash
# backup_cluster.sh — กลยุทธ์แนะนำสำหรับ production

BACKUP_DIR="/backup/$(date +%Y%m%d)"
mkdir -p "$BACKUP_DIR"

# 1. Backup globals (roles + tablespaces) — เล็ก เร็ว
pg_dumpall -h localhost -U postgres --globals-only \
  -f "$BACKUP_DIR/globals.sql"

# 2. Backup แต่ละฐานข้อมูลแยกกันด้วย pg_dump (parallel, compressed)
for DB in ecommerce_prod ecommerce_analytics; do
  pg_dump -h localhost -U postgres -d "$DB" \
    -Fd -j 4 \
    -f "$BACKUP_DIR/${DB}_dir"
done
```

```
$ ./backup_cluster.sh
$ ls -la /backup/20260925/
drwxr-xr-x  4 postgres postgres  4096 Sep 25 02:00 .
-rw-r--r--  1 postgres postgres  8420 Sep 25 02:00 globals.sql
drwxr-xr-x  2 postgres postgres  4096 Sep 25 02:03 ecommerce_prod_dir
drwxr-xr-x  2 postgres postgres  4096 Sep 25 02:05 ecommerce_analytics_dir
```

วิธี restore กลับต้อง**เรียง globals ก่อนเสมอ** เพราะฐานข้อมูลอาจอ้างอิง role ที่เป็นเจ้าของ object:

```bash
# 1. Restore roles/tablespaces ก่อน
psql -h localhost -U postgres -f /backup/20260925/globals.sql

# 2. สร้างฐานข้อมูลเปล่าแล้ว restore ตามหลัง
createdb -h localhost -U postgres ecommerce_prod
pg_restore -h localhost -U postgres -d ecommerce_prod -j 4 \
  /backup/20260925/ecommerce_prod_dir
```

### ตารางเปรียบเทียบ pg_dump กับ pg_dumpall

| คุณสมบัติ | `pg_dump` | `pg_dumpall` |
|---|---|---|
| ขอบเขต | ฐานข้อมูลเดียว | ทั้ง cluster (ทุกฐานข้อมูล) |
| Roles/Users | ไม่รวม | รวม |
| Tablespaces (นิยาม) | ไม่รวม | รวม |
| Output format | plain, custom, directory, tar | plain เท่านั้น |
| Parallel (`-j`) | ได้ (custom/directory) | ไม่ได้ |
| เหมาะกับ | Backup ปกติของแต่ละฐานข้อมูล production | Backup roles/tablespaces เสริม ก่อน disaster recovery เต็มรูปแบบ |

---

## Step 606: Logical backup (pg_dump) เทียบกับ Physical backup (pg_basebackup)

ก่อนเรียนรู้ `pg_basebackup` ในรายละเอียด เราต้องเข้าใจความแตกต่างเชิงแนวคิดระหว่าง backup สองประเภทนี้ให้ชัดเจนก่อน เพราะมันส่งผลต่อการเลือกกลยุทธ์อย่างมาก

### Logical Backup (pg_dump / pg_dumpall)

**หลักการทำงาน**: อ่านข้อมูลผ่าน SQL query (เหมือน `SELECT * FROM table`) แล้วแปลงเป็นคำสั่ง `INSERT`/`COPY` หรือ archive format เพื่อ "สร้างข้อมูลใหม่" ตอน restore

```
┌─────────────────┐     SQL queries      ┌──────────────────┐
│  PostgreSQL DB   │ ───────────────────► │  pg_dump process  │
│  (running)       │                       │                   │
└─────────────────┘                       └────────┬──────────┘
                                                     │ แปลงเป็น SQL/archive
                                                     ▼
                                           ┌──────────────────┐
                                           │  backup.dump     │
                                           │  (logical repr.) │
                                           └──────────────────┘
```

### Physical Backup (pg_basebackup)

**หลักการทำงาน**: คัดลอกไฟล์ดิบ (raw files) ในระดับ byte จาก data directory ของ PostgreSQL (`$PGDATA`) โดยตรง ไม่ผ่านการตีความเป็น SQL เลย

```
┌─────────────────┐   ไฟล์ดิบ (byte-level)  ┌──────────────────┐
│  PostgreSQL DB   │ ───────────────────►   │ pg_basebackup     │
│  $PGDATA/        │                         │ process           │
│    base/         │                         └────────┬──────────┘
│    pg_wal/       │                                   │ คัดลอกตรงๆ
│    global/       │                                   ▼
└─────────────────┘                         ┌──────────────────┐
                                             │ backup_dir/      │
                                             │  base/           │
                                             │  pg_wal/         │
                                             │  (exact copy)    │
                                             └──────────────────┘
```

### ตารางเปรียบเทียบละเอียด

| คุณสมบัติ | Logical Backup (`pg_dump`) | Physical Backup (`pg_basebackup`) |
|---|---|---|
| **สิ่งที่สำรอง** | ข้อมูลระดับตาราง/แถว | ทั้ง data directory (ไฟล์ระบบ) |
| **ขนาดไฟล์** | เล็กกว่า (บีบอัดได้ดี, ไม่มี index bloat) | ใหญ่กว่า (รวม index, bloat, WAL) |
| **ความเร็วในการ backup** | ช้ากว่าสำหรับ DB ใหญ่มาก (ต้องอ่านผ่าน SQL engine) | เร็วกว่า (I/O copy ตรงๆ) |
| **ความเร็วในการ restore** | ช้ากว่า (ต้องสร้าง index/constraint ใหม่ทั้งหมด) | เร็วกว่ามาก (แค่ start PostgreSQL ด้วยไฟล์ที่คัดลอกมา) |
| **ข้ามเวอร์ชัน PostgreSQL ได้ไหม** | ได้ (dump จาก PG15 restore เข้า PG17 ได้) | **ไม่ได้** (major version ต้องตรงกันเป๊ะ) |
| **ข้ามสถาปัตยกรรม/OS ได้ไหม** | ได้ | ไม่ได้ (ต้อง platform เดียวกัน) |
| **เลือก backup บางส่วนได้ไหม** | ได้ (เฉพาะตาราง/schema) | ไม่ได้ (ต้องเป็นทั้ง cluster เสมอ) |
| **ใช้ทำ replication ได้ไหม** | ไม่ได้โดยตรง | ได้ (ใช้เป็น base สำหรับ streaming replication) |
| **ใช้ทำ PITR ได้ไหม** | ไม่ได้ | ได้ (ร่วมกับ WAL archiving — ดู Part 062) |
| **Consistency ระหว่าง backup** | Snapshot ผ่าน MVCC ที่จุดเริ่ม transaction | ต้องใช้กลไก checkpoint + WAL เพื่อความ consistent |
| **โหลดขณะ backup** | ใช้ CPU/memory ของ query engine พอสมควร | เบากว่า (I/O bound ล้วนๆ) |
| **เหมาะกับ** | Backup รายวัน, migrate ข้อมูล, ย้ายข้าม version, backup บางส่วน | Disaster recovery แบบเต็ม, การตั้ง replica, RPO/RTO ต่ำ |

### เมื่อไหร่ควรใช้แบบไหน

**ใช้ Logical Backup (`pg_dump`/`pg_dumpall`) เมื่อ:**
- ต้องการ backup เฉพาะบางตาราง/schema
- ต้องการย้ายข้อมูลข้าม PostgreSQL version (เช่น upgrade จาก 15 เป็น 17)
- ต้องการย้ายข้อมูลข้าม OS/สถาปัตยกรรม (เช่น จาก x86 ไป ARM)
- ฐานข้อมูลมีขนาดไม่ใหญ่มาก (ไม่กี่สิบ GB ถึงร้อย GB) และ RTO ไม่ต้องเร็วมาก

**ใช้ Physical Backup (`pg_basebackup`) เมื่อ:**
- ต้องการ RTO ต่ำมาก (ฐานข้อมูลใหญ่ระดับ TB ขึ้นไป ที่ restore ด้วย pg_dump ใช้เวลาเป็นวัน)
- ต้องการตั้งค่า streaming replication / standby server
- ต้องการทำ Point-in-Time Recovery (PITR) ร่วมกับ WAL archiving
- ต้องการความเร็วในการ backup สูงสุด (I/O copy เร็วกว่า query-based dump)

> **แนวทางที่ดีที่สุดสำหรับ production จริง**: ใช้**ทั้งสองแบบร่วมกัน** — `pg_basebackup` + WAL archiving เป็นกลยุทธ์หลักสำหรับ disaster recovery และ PITR ส่วน `pg_dump` ใช้เป็น backup เสริมสำหรับการย้ายข้อมูล, backup เฉพาะตารางสำคัญ, หรือเป็น "safety net" ที่อ่านง่ายเมื่อต้องการตรวจสอบข้อมูลแบบ manual

---

## Step 607: pg_basebackup — physical backup ทั้ง data directory

`pg_basebackup` เชื่อมต่อไปยัง PostgreSQL server ผ่าน **replication protocol** แล้วคัดลอกไฟล์ทั้งหมดใน data directory (`$PGDATA`) มาไว้ที่ปลายทาง พร้อมกับ WAL segments ที่จำเป็นเพื่อให้ backup นั้น**สมบูรณ์และ consistent ได้ด้วยตัวเอง**

### เตรียม permission ก่อนใช้งาน

`pg_basebackup` ต้องใช้ user ที่มีสิทธิ์ `REPLICATION`:

```sql
-- รันบนเซิร์ฟเวอร์ต้นทาง (primary)
CREATE ROLE backup_user WITH REPLICATION LOGIN PASSWORD 'S3cur3P@ss';
```

และต้องอนุญาตใน `pg_hba.conf`:

```
# pg_hba.conf
host    replication     backup_user     10.0.1.0/24        scram-sha-256
```

รีโหลด config หลังแก้ไข:

```bash
psql -h localhost -U postgres -c "SELECT pg_reload_conf();"
```

```
 pg_reload_conf
────────────────
 t
(1 row)
```

ตรวจสอบว่ามี WAL sender slot เพียงพอ (`max_wal_senders` ต้อง > จำนวน backup/replica connection ที่ใช้พร้อมกัน):

```sql
SHOW max_wal_senders;
```

```
 max_wal_senders
──────────────────
 10
```

### คำสั่งพื้นฐาน

```bash
pg_basebackup \
  -h db-primary.internal \
  -U backup_user \
  -D /backup/basebackup_$(date +%Y%m%d) \
  -Fp \
  -Xs \
  -P \
  -v
```

```
Password:
pg_basebackup: initiating base backup, waiting for checkpoint to complete
pg_basebackup: checkpoint completed
pg_basebackup: write-ahead log start point: 0/3A000028 on timeline 1
pg_basebackup: starting background WAL receiver
26847283/26847283 kB (100%), 1/1 tablespace
pg_basebackup: write-ahead log end point: 0/3A000138
pg_basebackup: waiting for background process to finish streaming changes
pg_basebackup: syncing data to disk ...
pg_basebackup: renaming backup_manifest.tmp to backup_manifest
pg_basebackup: base backup completed
```

### ตัวเลือกสำคัญอธิบายทีละตัว

| Option | ความหมาย |
|---|---|
| `-D, --pgdata` | โฟลเดอร์ปลายทางที่จะเก็บ backup (**ต้องว่างเปล่า** หากใช้ `-Fp`) |
| `-F, --format` | `p` = plain (คัดลอกไฟล์ตรงๆ), `t` = tar (บีบเป็น `.tar`) |
| `-X, --wal-method` | วิธีจัดการ WAL: `fetch`, `stream` (`s`), หรือ `none` (`n`) |
| `-P, --progress` | แสดง progress bar ระหว่าง backup |
| `-v, --verbose` | แสดงรายละเอียดขั้นตอนการทำงาน |
| `-c, --checkpoint` | `fast` หรือ `spread` — วิธี checkpoint ก่อนเริ่ม backup |
| `-z, --gzip` | บีบอัดด้วย gzip (ใช้กับ `-Ft` เท่านั้น) |
| `-Z, --compress` | ระดับ/วิธีการบีบอัด (`gzip:6`, `zstd:5`, `server-gzip:6` เป็นต้น) |
| `-R, --write-recovery-conf` | สร้างไฟล์ `standby.signal` และตั้งค่า `primary_conninfo` ให้อัตโนมัติ (ใช้สร้าง replica) |
| `-T, --tablespace-mapping` | remap ตำแหน่ง tablespace ไปยัง path อื่น |
| `-j, --jobs` (PG17+) | จำนวน connection คู่ขนานสำหรับดึงข้อมูล (เร็วขึ้นสำหรับ storage เร็ว) |

### รายละเอียด `-X` (WAL method)

WAL (Write-Ahead Log) ที่เกิดขึ้น**ระหว่าง**การ backup กำลังดำเนินอยู่ จำเป็นต้องถูกเก็บไปพร้อมกับ backup ด้วย ไม่เช่นนั้น backup จะไม่ consistent

```bash
# stream (แนะนำ default) — เปิด connection ที่สองพร้อมกัน คอย stream WAL ระหว่าง backup ทำงาน
pg_basebackup -h db-primary.internal -U backup_user \
  -D /backup/basebackup_stream \
  -Fp -Xs -P

# fetch — ดึง WAL ทั้งหมดหลังจาก backup ไฟล์เสร็จแล้วเท่านั้น (เสี่ยง WAL หมุนทับก่อนดึงทัน ถ้า backup ใช้เวลานาน)
pg_basebackup -h db-primary.internal -U backup_user \
  -D /backup/basebackup_fetch \
  -Fp -Xf -P

# none — ไม่ดึง WAL เลย (backup จะใช้งานไม่ได้ด้วยตัวเอง ต้องมี WAL archiving แยกต่างหาก)
pg_basebackup -h db-primary.internal -U backup_user \
  -D /backup/basebackup_none \
  -Fp -Xn -P
```

> **แนะนำใช้ `-Xs` (stream) เป็นค่า default เสมอ** เพราะปลอดภัยที่สุดและใช้งานง่ายที่สุด — ทำให้ backup ที่ได้ **สมบูรณ์ในตัวเอง (self-contained)** พร้อม restore ได้ทันทีโดยไม่ต้องพึ่ง WAL archive ภายนอก

### Backup แบบ tar พร้อมบีบอัด

```bash
mkdir -p /backup/basebackup_tar_$(date +%Y%m%d)

pg_basebackup \
  -h db-primary.internal \
  -U backup_user \
  -D /backup/basebackup_tar_$(date +%Y%m%d) \
  -Ft \
  -z \
  -Z 6 \
  -Xs \
  -P \
  -v
```

```
pg_basebackup: initiating base backup, waiting for checkpoint to complete
pg_basebackup: checkpoint completed
pg_basebackup: write-ahead log start point: 0/44000028 on timeline 1
pg_basebackup: starting background WAL receiver
24601/24601 kB (100%), 1/1 tablespace
pg_basebackup: write-ahead log end point: 0/44000138
pg_basebackup: syncing data to disk ...
pg_basebackup: base backup completed
```

```bash
ls -la /backup/basebackup_tar_20260925/
```

```
-rw------- 1 postgres postgres 1847293841 Sep 25 03:05 base.tar.gz
-rw------- 1 postgres postgres   16778240 Sep 25 03:05 pg_wal.tar.gz
-rw------- 1 postgres postgres      12583 Sep 25 03:05 backup_manifest
```

### Backup พร้อม server-side compression (PostgreSQL 15+)

```bash
pg_basebackup \
  -h db-primary.internal \
  -U backup_user \
  -D /backup/basebackup_serverzip_$(date +%Y%m%d) \
  -Ft \
  --compress=server-gzip:6 \
  -Xs \
  -P
```

การบีบอัดฝั่ง server ช่วยลด network bandwidth ที่ใช้ส่งข้อมูลจาก primary ไปยังเครื่อง backup โดยเฉพาะเมื่อ backup ข้าม datacenter

### backup_manifest — การตรวจสอบความถูกต้อง

ตั้งแต่ PostgreSQL 13 เป็นต้นมา `pg_basebackup` จะสร้างไฟล์ `backup_manifest` เสมอ (มี checksum ของทุกไฟล์) ใช้ตรวจสอบความสมบูรณ์ของ backup ได้ด้วย `pg_verifybackup`:

```bash
pg_verifybackup /backup/basebackup_20260925
```

```
pg_verifybackup: backup successfully verified
```

หากมีไฟล์เสียหายหรือหายไป:

```
pg_verifybackup: error: "base/16384/24580" is present in the manifest but not on disk
pg_verifybackup: error: "base/16384/24601" has checksum of length 64 on disk but expected 64
pg_verifybackup: fatal: WAL parsing failed
```

> ควรรัน `pg_verifybackup` เป็นขั้นตอนอัตโนมัติหลัง `pg_basebackup` ทุกครั้งใน script backup

### การ Restore physical backup

การ restore ทำได้ง่ายมาก แค่คัดลอกไฟล์กลับไปเป็น `$PGDATA` แล้ว start PostgreSQL:

```bash
# หยุด PostgreSQL เดิม (ถ้ามี) และสำรอง/ลบ data directory เก่า
sudo systemctl stop postgresql

sudo rm -rf /var/lib/postgresql/17/main/*

# แตกไฟล์ backup กลับเข้า data directory
sudo tar -xzf /backup/basebackup_tar_20260925/base.tar.gz \
  -C /var/lib/postgresql/17/main/

sudo tar -xzf /backup/basebackup_tar_20260925/pg_wal.tar.gz \
  -C /var/lib/postgresql/17/main/pg_wal/

# ตั้งค่า permission ให้ถูกต้อง
sudo chown -R postgres:postgres /var/lib/postgresql/17/main
sudo chmod 700 /var/lib/postgresql/17/main

# start PostgreSQL
sudo systemctl start postgresql
```

```bash
sudo journalctl -u postgresql -n 15 --no-pager
```

```
Sep 25 03:20:11 db-server postgres[8821]: database system was interrupted; last known up at 2026-09-25 03:00:02 UTC
Sep 25 03:20:11 db-server postgres[8821]: database system was not properly shut down; automatic recovery in progress
Sep 25 03:20:12 db-server postgres[8821]: redo starts at 0/44000028
Sep 25 03:20:13 db-server postgres[8821]: redo done at 0/44000138
Sep 25 03:20:13 db-server postgres[8821]: database system is ready to accept connections
```

> ถ้าใช้ format `-Fp` (plain) การ restore ยิ่งง่ายกว่านั้นอีก — เพียงแค่คัดลอกทั้งโฟลเดอร์ที่ `pg_basebackup` สร้างไว้ไปวางแทน data directory เดิมได้เลย ไม่ต้องแตกไฟล์ tar

---

## Step 608: การทดสอบ Backup — ทำไมต้อง restore ทดสอบเป็นประจำ

### หลักการที่สำคัญที่สุดในบทนี้

> **"Backup ที่ไม่เคยทดสอบ restore ถือว่าไม่มี backup"**

ประโยคนี้อาจฟังดูเกินจริง แต่ในความเป็นจริงของงาน DBA มันคือความจริงเจ็บปวดที่เกิดขึ้นซ้ำแล้วซ้ำเล่า เหตุผลที่ backup ที่ไม่เคยทดสอบเป็น "ระเบิดเวลา" มีดังนี้:

1. **ไฟล์ backup อาจเสียหายโดยไม่รู้ตัว** — disk error, network transfer ขาดหาย, หรือ process ถูก kill กลางคัน
2. **Script backup อาจมี bug** — เช่น ลืม escape ตัวแปร, path ผิด, เชื่อมต่อฐานข้อมูลผิดตัว (backup ฐาน staging แทน production โดยไม่รู้ตัว)
3. **Permission หรือ credential เปลี่ยน** — user ที่ใช้ backup ถูก revoke สิทธิ์ไปแล้วโดยไม่มีใครสังเกต ทำให้ backup "สำเร็จ" แต่ได้ไฟล์เปล่า
4. **เวอร์ชัน PostgreSQL ไม่ตรงกัน** — restore ไม่ได้เพราะ major version ไม่ compatible (โดยเฉพาะ physical backup)
5. **พื้นที่ดิสก์ไม่พอตอน restore จริง** — ไม่มีใครรู้จนกว่าจะลองจริง

### เคสตัวอย่างที่เกิดขึ้นจริงในวงการ (แบบทั่วไปที่พบบ่อย)

```
สถานการณ์: บริษัท e-commerce แห่งหนึ่งตั้ง cron job รัน pg_dump ทุกคืน
เก็บไฟล์ไว้ใน S3 มา 8 เดือน ไม่เคยมีใคร restore ทดสอบเลย

วันหนึ่ง: มีคนรัน DROP TABLE orders โดยไม่ตั้งใจใน production

ทีมดึงไฟล์ backup ล่าสุดมา restore → พบว่าไฟล์ backup มีขนาด 4KB
(cron job เชื่อมต่อฐานข้อมูลผิด host มา 8 เดือนติดต่อกัน
เพราะมีการเปลี่ยน DNS record ของเซิร์ฟเวอร์ database
แต่ script ไม่มีการเช็ค exit code หรือขนาดไฟล์เลย)

ผลลัพธ์: ข้อมูล orders สูญหายถาวร ย้อนกลับไม่ได้
```

นี่คือเหตุผลว่าทำไม "การทดสอบ restore" ต้องเป็นส่วนหนึ่งของ backup pipeline ไม่ใช่ทางเลือกเสริม

### สิ่งที่ต้องตรวจสอบเมื่อทดสอบ restore

การทดสอบ restore ที่ดีต้องตอบคำถามเหล่านี้ให้ได้ทุกครั้ง:

1. **ไฟล์ backup restore ได้จริงหรือไม่** (ไม่มี error/corruption)
2. **ข้อมูลที่ restore มาครบถ้วนหรือไม่** (จำนวนแถวตรงกับที่คาดหวัง)
3. **ใช้เวลานานแค่ไหน** (ตรงกับ RTO ที่ตั้งไว้หรือไม่)
4. **Application เชื่อมต่อและทำงานกับข้อมูลที่ restore มาได้จริงหรือไม่**

### สคริปต์ทดสอบ restore อัตโนมัติ

```bash
#!/bin/bash
# test_restore.sh — ทดสอบ restore backup ล่าสุดแบบอัตโนมัติ พร้อม verify
set -euo pipefail

BACKUP_FILE="/backup/latest/ecommerce_prod.dump"
TEST_DB="ecommerce_restore_verify_$(date +%Y%m%d)"
LOG_FILE="/var/log/pg_backup/restore_test_$(date +%Y%m%d).log"
EXPECTED_MIN_ORDERS=100000   # ค่าประมาณต่ำสุดที่ควรมีใน production

exec > >(tee -a "$LOG_FILE") 2>&1

echo "=== เริ่มทดสอบ restore: $(date) ==="

# 1. ตรวจสอบว่าไฟล์ backup มีอยู่จริงและมีขนาดสมเหตุสมผล
if [ ! -f "$BACKUP_FILE" ]; then
  echo "FATAL: ไม่พบไฟล์ backup ที่ $BACKUP_FILE"
  exit 1
fi

FILE_SIZE=$(stat -c%s "$BACKUP_FILE")
if [ "$FILE_SIZE" -lt 1048576 ]; then   # น้อยกว่า 1MB ถือว่าผิดปกติ
  echo "FATAL: ไฟล์ backup มีขนาดเล็กผิดปกติ ($FILE_SIZE bytes) — อาจ backup ล้มเหลว"
  exit 1
fi
echo "OK: ไฟล์ backup ขนาด $(numfmt --to=iec $FILE_SIZE)"

# 2. ตรวจสอบความถูกต้องของ archive ด้วย pg_restore -l
if ! pg_restore -l "$BACKUP_FILE" > /dev/null 2>&1; then
  echo "FATAL: ไฟล์ backup เสียหาย ไม่สามารถอ่าน table of contents ได้"
  exit 1
fi
echo "OK: archive structure ถูกต้อง"

# 3. สร้างฐานข้อมูลทดสอบ (ลบก่อนถ้ามีอยู่แล้ว)
dropdb -h localhost -U postgres --if-exists "$TEST_DB"
createdb -h localhost -U postgres "$TEST_DB"
echo "OK: สร้างฐานข้อมูลทดสอบ $TEST_DB"

# 4. Restore จริง วัดเวลา
START_TIME=$(date +%s)
pg_restore -h localhost -U postgres -d "$TEST_DB" -j 4 "$BACKUP_FILE"
END_TIME=$(date +%s)
DURATION=$((END_TIME - START_TIME))
echo "OK: restore เสร็จภายใน ${DURATION} วินาที"

# 5. ตรวจสอบจำนวนแถวในตารางสำคัญ
ORDER_COUNT=$(psql -h localhost -U postgres -d "$TEST_DB" -tAc "SELECT COUNT(*) FROM orders;")
PRODUCT_COUNT=$(psql -h localhost -U postgres -d "$TEST_DB" -tAc "SELECT COUNT(*) FROM products;")
CUSTOMER_COUNT=$(psql -h localhost -U postgres -d "$TEST_DB" -tAc "SELECT COUNT(*) FROM customers;")

echo "จำนวนแถว: orders=$ORDER_COUNT, products=$PRODUCT_COUNT, customers=$CUSTOMER_COUNT"

if [ "$ORDER_COUNT" -lt "$EXPECTED_MIN_ORDERS" ]; then
  echo "FATAL: จำนวน orders ($ORDER_COUNT) น้อยกว่าค่าที่คาดหวัง ($EXPECTED_MIN_ORDERS) — ข้อมูลอาจไม่ครบ"
  exit 1
fi

# 6. ตรวจสอบ referential integrity แบบง่าย
ORPHAN_ITEMS=$(psql -h localhost -U postgres -d "$TEST_DB" -tAc \
  "SELECT COUNT(*) FROM order_items oi LEFT JOIN orders o ON oi.order_id = o.order_id WHERE o.order_id IS NULL;")
if [ "$ORPHAN_ITEMS" -gt 0 ]; then
  echo "WARNING: พบ order_items ที่ไม่มี order แม่ ($ORPHAN_ITEMS แถว)"
fi

# 7. ล้างฐานข้อมูลทดสอบทิ้ง
dropdb -h localhost -U postgres "$TEST_DB"

echo "=== ทดสอบ restore สำเร็จ: $(date) ==="
```

```
$ ./test_restore.sh
=== เริ่มทดสอบ restore: Fri Sep 25 04:00:01 UTC 2026 ===
OK: ไฟล์ backup ขนาด 156M
OK: archive structure ถูกต้อง
OK: สร้างฐานข้อมูลทดสอบ ecommerce_restore_verify_20260925
OK: restore เสร็จภายใน 134 วินาที
จำนวนแถว: orders=1284392, products=48210, customers=392841
=== ทดสอบ restore สำเร็จ: Fri Sep 25 04:02:21 UTC 2026 ===
```

### ความถี่ในการทดสอบ

| ระดับความสำคัญของระบบ | ความถี่ในการทดสอบ restore |
|---|---|
| ระบบทดลอง/dev | ไม่จำเป็นต้องทดสอบสม่ำเสมอ |
| ระบบ production ทั่วไป | ทดสอบอัตโนมัติทุกวัน (เหมือนตัวอย่างข้างต้น) + ทดสอบ full disaster recovery ทุกไตรมาส |
| ระบบ critical (การเงิน, สุขภาพ) | ทดสอบอัตโนมัติทุกวัน + ทดสอบ full disaster recovery ทุกเดือน + ทำ "game day" จำลองสถานการณ์จริงทุก 6 เดือน |

> **แนวคิด "Game Day"**: บางองค์กรระดับโลก (เช่น Netflix, Amazon) จัดวันจำลองภัยพิบัติ (Chaos Engineering) โดยตั้งใจปิดระบบ production จริง (หรือ replica ที่เหมือนจริงที่สุด) แล้วให้ทีมกู้คืนตามขั้นตอนจริงทั้งหมด เพื่อพิสูจน์ว่า backup และ runbook ใช้งานได้จริงเมื่อเกิดเหตุฉุกเฉินจริง ไม่ใช่แค่ทฤษฎีบนกระดาษ

---

## Step 609: กลยุทธ์ Backup Retention — 3-2-1 rule

การมี backup อย่างเดียวไม่พอ ต้องมี**นโยบายการเก็บรักษา (retention policy)** ที่ชัดเจนด้วยว่าจะเก็บ backup ไว้นานแค่ไหน กี่ชุด และเก็บไว้ที่ไหนบ้าง

### กฎ 3-2-1

กฎ 3-2-1 เป็นหลักการพื้นฐานที่ใช้กันทั่วโลกในวงการ backup:

```
        3-2-1 Backup Rule

    3 ─── สำเนาข้อมูลอย่างน้อย 3 ชุด
          (1 ต้นฉบับ production + 2 backup)

    2 ─── เก็บบน media/storage type ที่แตกต่างกันอย่างน้อย 2 ชนิด
          (เช่น local disk + cloud object storage)

    1 ─── เก็บอย่างน้อย 1 ชุดไว้นอกสถานที่ (offsite)
          (คนละ datacenter/region จาก production)
```

ตัวอย่างการ implement กฎ 3-2-1 สำหรับระบบ e-commerce:

| สำเนา # | ที่เก็บ | Media type | Offsite? |
|---|---|---|---|
| 1 (ต้นฉบับ) | `db-primary.internal` | Local SSD | ไม่ (คือ production เอง) |
| 2 | `backup-server.internal` (เครื่องแยก) | Local disk บนเครื่อง backup dedicated | ไม่ (แต่คนละเครื่อง) |
| 3 | AWS S3 (bucket แยก region) | Cloud object storage | ใช่ (คนละ region/datacenter) |

```bash
#!/bin/bash
# backup_with_321.sh — ตัวอย่างการกระจาย backup ตามกฎ 3-2-1

BACKUP_FILE="/backup/local/ecommerce_prod_$(date +%Y%m%d).dump"

# สำเนา 1: ทำ pg_dump เก็บไว้ที่เครื่อง backup server (local)
pg_dump -h db-primary.internal -U backup_user -d ecommerce_prod \
  -Fc -j 4 -f "$BACKUP_FILE"

# สำเนา 2: sync ไปยัง NAS แยกเครื่อง (media type ต่างกัน)
rsync -avz "$BACKUP_FILE" nas-server.internal:/mnt/backup_nas/pg/

# สำเนา 3: อัปโหลดไปยัง S3 คนละ region (offsite)
aws s3 cp "$BACKUP_FILE" \
  s3://ecommerce-db-backups-ap-southeast/daily/ \
  --storage-class STANDARD_IA
```

```
upload: /backup/local/ecommerce_prod_20260925.dump to s3://ecommerce-db-backups-ap-southeast/daily/ecommerce_prod_20260925.dump
```

### การเก็บ backup หลายช่วงเวลา (Grandfather-Father-Son)

นอกจากกระจาย location แล้ว ยังต้องออกแบบว่าจะ**เก็บ backup เก่าแค่ไหน**และ**ลบทิ้งเมื่อไหร่** รูปแบบที่นิยมที่สุดคือ **Grandfather-Father-Son (GFS)**:

```
Daily (Son)     ──► เก็บ 7 วันล่าสุด (backup ทุกคืน)
Weekly (Father) ──► เก็บ 4 สัปดาห์ล่าสุด (backup วันอาทิตย์)
Monthly (Grandfather) ──► เก็บ 12 เดือนล่าสุด (backup วันที่ 1 ของเดือน)
Yearly (บางองค์กร) ──► เก็บถาวรหรือหลายปี (backup วันที่ 1 มกราคม)
```

ภาพรวมตารางแสดงจำนวน backup ที่เก็บไว้ ณ เวลาใดเวลาหนึ่ง:

| ระดับ | ความถี่ | เก็บกี่ชุด | ตัวอย่างวันที่เก็บ |
|---|---|---|---|
| Daily | ทุกวัน | 7 | 19, 20, 21, 22, 23, 24, 25 ก.ย. |
| Weekly | ทุกสัปดาห์ (อาทิตย์) | 4 | 31 ส.ค., 7, 14, 21 ก.ย. |
| Monthly | ทุกเดือน (วันที่ 1) | 12 | ม.ค.–ธ.ค. 2569 |
| Yearly | ทุกปี | ตามนโยบาย/กฎหมาย | 2567, 2568, 2569 |

### Script จัดการ retention อัตโนมัติ

```bash
#!/bin/bash
# rotate_backups.sh — ลบ backup เก่าตามนโยบาย GFS
set -euo pipefail

BACKUP_ROOT="/backup"
DAILY_DIR="$BACKUP_ROOT/daily"
WEEKLY_DIR="$BACKUP_ROOT/weekly"
MONTHLY_DIR="$BACKUP_ROOT/monthly"

DAILY_RETENTION_DAYS=7
WEEKLY_RETENTION_DAYS=28
MONTHLY_RETENTION_DAYS=365

echo "=== เริ่ม rotate backups: $(date) ==="

# ลบ daily backup ที่เก่ากว่า 7 วัน
find "$DAILY_DIR" -name "*.dump" -mtime +$DAILY_RETENTION_DAYS -print -delete

# ลบ weekly backup ที่เก่ากว่า 28 วัน
find "$WEEKLY_DIR" -name "*.dump" -mtime +$WEEKLY_RETENTION_DAYS -print -delete

# ลบ monthly backup ที่เก่ากว่า 365 วัน
find "$MONTHLY_DIR" -name "*.dump" -mtime +$MONTHLY_RETENTION_DAYS -print -delete

echo "=== rotate backups เสร็จสิ้น: $(date) ==="
```

```
=== เริ่ม rotate backups: Fri Sep 25 05:00:01 UTC 2026 ===
/backup/daily/ecommerce_prod_20260918.dump
/backup/weekly/ecommerce_prod_20260817.dump
=== rotate backups เสร็จสิ้น: Fri Sep 25 05:00:03 UTC 2026 ===
```

### ปัจจัยอื่นที่ต้องพิจารณาในนโยบาย retention

1. **ข้อกำหนดทางกฎหมาย (Compliance)** — บางธุรกิจ (การเงิน, สุขภาพ) มีกฎหมายบังคับให้เก็บข้อมูลย้อนหลังหลายปี (เช่น PDPA, PCI-DSS อาจกำหนดระยะเวลาขั้นต่ำ)
2. **ต้นทุนพื้นที่เก็บข้อมูล** — ยิ่งเก็บนาน ยิ่งใช้พื้นที่มาก ต้องคำนวณต้นทุนเทียบกับความเสี่ยง
3. **Encryption ของ backup ที่เก็บนอกสถานที่** — ข้อมูล backup มักมีข้อมูลอ่อนไหว (PII ของลูกค้า) ต้อง encrypt ก่อนอัปโหลดขึ้น cloud เสมอ

```bash
# ตัวอย่างการเข้ารหัสไฟล์ backup ก่อนอัปโหลดด้วย GPG
gpg --symmetric --cipher-algo AES256 \
  --output ecommerce_prod_20260925.dump.gpg \
  ecommerce_prod_20260925.dump

aws s3 cp ecommerce_prod_20260925.dump.gpg \
  s3://ecommerce-db-backups-ap-southeast/daily/
```

4. **Immutability** — ป้องกัน ransomware ด้วยการตั้งค่า S3 Object Lock หรือ WORM (Write Once Read Many) storage เพื่อไม่ให้ใครลบ/แก้ไข backup ได้แม้แต่ admin เอง ภายในช่วงเวลาที่กำหนด

```bash
aws s3api put-object \
  --bucket ecommerce-db-backups-ap-southeast \
  --key daily/ecommerce_prod_20260925.dump.gpg \
  --body ecommerce_prod_20260925.dump.gpg \
  --object-lock-mode COMPLIANCE \
  --object-lock-retain-until-date 2026-10-25T00:00:00Z
```

---

## Step 610: แบบฝึกหัดรวม — ออกแบบและเขียน backup script อัตโนมัติสำหรับระบบ e-commerce production

ในขั้นตอนสุดท้ายนี้ เราจะรวมทุกสิ่งที่เรียนมาในบทนี้เข้าด้วยกัน เพื่อสร้าง **backup script อัตโนมัติระดับ production** สำหรับระบบ e-commerce ที่มีฐานข้อมูลหลัก `ecommerce_prod`

### ข้อกำหนดของระบบ (requirements)

- RPO: 24 ชั่วโมง (ยอมรับข้อมูลหายได้ไม่เกิน 1 วัน สำหรับ logical backup — PITR แบบละเอียดจะอยู่ใน Part 062)
- RTO: 2 ชั่วโมง
- ต้อง backup ทุกคืนเวลา 02:00 น. (เวลาที่ traffic ต่ำสุด)
- ต้องเก็บ backup ตามนโยบาย GFS (daily 7 วัน, weekly 4 สัปดาห์, monthly 12 เดือน)
- ต้อง sync backup ไปยัง cloud storage (offsite) ทุกครั้ง
- ต้อง verify backup ทุกครั้งหลัง backup เสร็จ
- ต้องแจ้งเตือนทีมเมื่อ backup ล้มเหลว

### โครงสร้างไฟล์ script

```bash
mkdir -p /opt/pg_backup_scripts
mkdir -p /backup/{daily,weekly,monthly}
mkdir -p /var/log/pg_backup
```

### ไฟล์ config: `/opt/pg_backup_scripts/backup.conf`

```bash
# backup.conf — ค่าตั้งค่าสำหรับ backup script ทั้งหมด

# การเชื่อมต่อฐานข้อมูล
PG_HOST="db-primary.internal"
PG_PORT="5432"
PG_USER="backup_user"
DATABASES=("ecommerce_prod" "ecommerce_analytics")

# Path เก็บ backup
BACKUP_ROOT="/backup"
DAILY_DIR="${BACKUP_ROOT}/daily"
WEEKLY_DIR="${BACKUP_ROOT}/weekly"
MONTHLY_DIR="${BACKUP_ROOT}/monthly"
LOG_DIR="/var/log/pg_backup"

# Retention (จำนวนวัน)
DAILY_RETENTION_DAYS=7
WEEKLY_RETENTION_DAYS=28
MONTHLY_RETENTION_DAYS=365

# Cloud storage (offsite)
S3_BUCKET="s3://ecommerce-db-backups-ap-southeast"

# การแจ้งเตือน
SLACK_WEBHOOK_URL="https://hooks.slack.com/services/XXXX/YYYY/ZZZZ"
ALERT_EMAIL="dba-team@example.com"

# Compression
PARALLEL_JOBS=4
COMPRESS_LEVEL=6
```

### ไฟล์หลัก: `/opt/pg_backup_scripts/run_backup.sh`

```bash
#!/bin/bash
# run_backup.sh — สคริปต์ backup อัตโนมัติหลักสำหรับ ecommerce production
#
# วิธีใช้: run_backup.sh [daily|weekly|monthly]
# เรียกผ่าน cron ทุกคืน

set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
source "${SCRIPT_DIR}/backup.conf"

BACKUP_TYPE="${1:-daily}"       # daily / weekly / monthly
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
DATE_ONLY=$(date +%Y%m%d)
LOG_FILE="${LOG_DIR}/backup_${BACKUP_TYPE}_${TIMESTAMP}.log"

# ----------------------------------------------------------------
# ฟังก์ชันช่วยเหลือ
# ----------------------------------------------------------------

log() {
  echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" | tee -a "$LOG_FILE"
}

notify_slack() {
  local status="$1"
  local message="$2"
  local color="good"
  [ "$status" = "FAILED" ] && color="danger"

  curl -s -X POST "$SLACK_WEBHOOK_URL" \
    -H 'Content-Type: application/json' \
    -d "{\"attachments\":[{\"color\":\"${color}\",\"title\":\"PostgreSQL Backup: ${status}\",\"text\":\"${message}\"}]}" \
    > /dev/null || log "WARNING: ส่ง Slack notification ไม่สำเร็จ"
}

fail() {
  local message="$1"
  log "FATAL: $message"
  notify_slack "FAILED" "$message (ดู log: $LOG_FILE)"
  echo "$message" | mail -s "[URGENT] PostgreSQL Backup ล้มเหลว - $(hostname)" "$ALERT_EMAIL" || true
  exit 1
}

# ----------------------------------------------------------------
# เลือกโฟลเดอร์ปลายทางตามประเภท backup
# ----------------------------------------------------------------

case "$BACKUP_TYPE" in
  daily)   TARGET_DIR="$DAILY_DIR" ;;
  weekly)  TARGET_DIR="$WEEKLY_DIR" ;;
  monthly) TARGET_DIR="$MONTHLY_DIR" ;;
  *) fail "ประเภท backup ไม่ถูกต้อง: $BACKUP_TYPE (ต้องเป็น daily/weekly/monthly)" ;;
esac

log "=== เริ่ม backup ประเภท '$BACKUP_TYPE' ==="

# ----------------------------------------------------------------
# ขั้นตอนที่ 1: ตรวจสอบว่าเชื่อมต่อฐานข้อมูลได้ก่อนเริ่ม
# ----------------------------------------------------------------

if ! pg_isready -h "$PG_HOST" -p "$PG_PORT" -U "$PG_USER" > /dev/null 2>&1; then
  fail "ไม่สามารถเชื่อมต่อไปยัง PostgreSQL server ที่ $PG_HOST:$PG_PORT ได้"
fi
log "OK: เชื่อมต่อฐานข้อมูลได้ปกติ"

# ----------------------------------------------------------------
# ขั้นตอนที่ 2: backup globals (roles + tablespaces) ก่อนเสมอ
# ----------------------------------------------------------------

GLOBALS_FILE="${TARGET_DIR}/globals_${TIMESTAMP}.sql"
if ! pg_dumpall -h "$PG_HOST" -U "$PG_USER" --globals-only -f "$GLOBALS_FILE" 2>> "$LOG_FILE"; then
  fail "pg_dumpall --globals-only ล้มเหลว"
fi
log "OK: backup globals เสร็จ -> $GLOBALS_FILE"

# ----------------------------------------------------------------
# ขั้นตอนที่ 3: backup แต่ละฐานข้อมูลด้วย pg_dump (parallel, directory format)
# ----------------------------------------------------------------

for DB in "${DATABASES[@]}"; do
  DB_BACKUP_DIR="${TARGET_DIR}/${DB}_${TIMESTAMP}"
  log "กำลัง backup ฐานข้อมูล: $DB -> $DB_BACKUP_DIR"

  if ! pg_dump -h "$PG_HOST" -U "$PG_USER" -d "$DB" \
       -Fd \
       -j "$PARALLEL_JOBS" \
       -Z "$COMPRESS_LEVEL" \
       -f "$DB_BACKUP_DIR" 2>> "$LOG_FILE"; then
    fail "pg_dump ล้มเหลวสำหรับฐานข้อมูล $DB"
  fi

  # ตรวจสอบว่าไฟล์ toc.dat มีอยู่จริง (สัญญาณว่า dump สำเร็จ)
  if [ ! -f "${DB_BACKUP_DIR}/toc.dat" ]; then
    fail "ไม่พบไฟล์ toc.dat ใน $DB_BACKUP_DIR — backup อาจไม่สมบูรณ์"
  fi

  DB_SIZE=$(du -sh "$DB_BACKUP_DIR" | cut -f1)
  log "OK: backup $DB เสร็จ (ขนาด: $DB_SIZE)"
done

# ----------------------------------------------------------------
# ขั้นตอนที่ 4: ตรวจสอบความถูกต้องของ backup (verify)
# ----------------------------------------------------------------

for DB in "${DATABASES[@]}"; do
  DB_BACKUP_DIR="${TARGET_DIR}/${DB}_${TIMESTAMP}"
  if ! pg_restore -l "$DB_BACKUP_DIR" > /dev/null 2>> "$LOG_FILE"; then
    fail "ตรวจสอบ archive ของ $DB_BACKUP_DIR ล้มเหลว — archive อาจเสียหาย"
  fi
done
log "OK: ตรวจสอบความถูกต้องของ archive ทั้งหมดผ่าน"

# ----------------------------------------------------------------
# ขั้นตอนที่ 5: sync ไปยัง offsite storage (S3)
# ----------------------------------------------------------------

log "กำลัง sync backup ไปยัง $S3_BUCKET/${BACKUP_TYPE}/"
if ! aws s3 sync "$TARGET_DIR" "${S3_BUCKET}/${BACKUP_TYPE}/" \
     --exclude "*" --include "*_${TIMESTAMP}*" \
     --storage-class STANDARD_IA >> "$LOG_FILE" 2>&1; then
  log "WARNING: sync ไป S3 ล้มเหลว (backup ในเครื่อง local ยังใช้งานได้)"
  notify_slack "WARNING" "Sync backup ไป S3 ล้มเหลวสำหรับ $BACKUP_TYPE backup วันที่ $DATE_ONLY"
else
  log "OK: sync ไป S3 เสร็จสิ้น"
fi

# ----------------------------------------------------------------
# ขั้นตอนที่ 6: ลบ backup เก่าตามนโยบาย retention
# ----------------------------------------------------------------

case "$BACKUP_TYPE" in
  daily)
    find "$DAILY_DIR" -maxdepth 1 -name "*_2*" -mtime +$DAILY_RETENTION_DAYS -print -exec rm -rf {} \; >> "$LOG_FILE"
    ;;
  weekly)
    find "$WEEKLY_DIR" -maxdepth 1 -name "*_2*" -mtime +$WEEKLY_RETENTION_DAYS -print -exec rm -rf {} \; >> "$LOG_FILE"
    ;;
  monthly)
    find "$MONTHLY_DIR" -maxdepth 1 -name "*_2*" -mtime +$MONTHLY_RETENTION_DAYS -print -exec rm -rf {} \; >> "$LOG_FILE"
    ;;
esac
log "OK: ลบ backup เก่าตามนโยบาย retention เสร็จสิ้น"

# ----------------------------------------------------------------
# สรุปผล
# ----------------------------------------------------------------

TOTAL_SIZE=$(du -sh "$TARGET_DIR" | cut -f1)
log "=== backup ประเภท '$BACKUP_TYPE' เสร็จสมบูรณ์ | ขนาดรวม: $TOTAL_SIZE ==="
notify_slack "SUCCESS" "Backup ${BACKUP_TYPE} สำหรับ $(hostname) เสร็จสมบูรณ์ (ขนาดรวม: ${TOTAL_SIZE})"

exit 0
```

```bash
chmod +x /opt/pg_backup_scripts/run_backup.sh
```

### ตั้งค่า cron job

```bash
crontab -e -u postgres
```

```cron
# ─────────── Backup schedule สำหรับ ecommerce_prod ───────────
# Daily backup: ทุกวันเวลา 02:00 น.
0 2 * * * /opt/pg_backup_scripts/run_backup.sh daily >> /var/log/pg_backup/cron.log 2>&1

# Weekly backup: ทุกวันอาทิตย์เวลา 03:00 น.
0 3 * * 0 /opt/pg_backup_scripts/run_backup.sh weekly >> /var/log/pg_backup/cron.log 2>&1

# Monthly backup: วันที่ 1 ของทุกเดือนเวลา 04:00 น.
0 4 1 * * /opt/pg_backup_scripts/run_backup.sh monthly >> /var/log/pg_backup/cron.log 2>&1

# ทดสอบ restore อัตโนมัติทุกวันเวลา 05:00 น. (หลัง backup เสร็จ)
0 5 * * * /opt/pg_backup_scripts/test_restore.sh >> /var/log/pg_backup/restore_test.log 2>&1
```

ตรวจสอบว่า cron ถูกตั้งไว้ถูกต้อง:

```bash
crontab -l -u postgres
```

```
0 2 * * * /opt/pg_backup_scripts/run_backup.sh daily >> /var/log/pg_backup/cron.log 2>&1
0 3 * * 0 /opt/pg_backup_scripts/run_backup.sh weekly >> /var/log/pg_backup/cron.log 2>&1
0 4 1 * * /opt/pg_backup_scripts/run_backup.sh monthly >> /var/log/pg_backup/cron.log 2>&1
0 5 * * * /opt/pg_backup_scripts/test_restore.sh >> /var/log/pg_backup/restore_test.log 2>&1
```

### ทดสอบรันด้วยมือก่อน deploy จริง

ก่อนปล่อยให้ cron รันอัตโนมัติ ควรทดสอบรัน script ด้วยมือก่อนเสมอเพื่อยืนยันว่าทำงานถูกต้อง:

```bash
sudo -u postgres /opt/pg_backup_scripts/run_backup.sh daily
```

```
[2026-09-25 02:00:01] === เริ่ม backup ประเภท 'daily' ===
[2026-09-25 02:00:01] OK: เชื่อมต่อฐานข้อมูลได้ปกติ
[2026-09-25 02:00:03] OK: backup globals เสร็จ -> /backup/daily/globals_20260925_020001.sql
[2026-09-25 02:00:03] กำลัง backup ฐานข้อมูล: ecommerce_prod -> /backup/daily/ecommerce_prod_20260925_020001
[2026-09-25 02:03:47] OK: backup ecommerce_prod เสร็จ (ขนาด: 1.6G)
[2026-09-25 02:03:47] กำลัง backup ฐานข้อมูล: ecommerce_analytics -> /backup/daily/ecommerce_analytics_20260925_020001
[2026-09-25 02:05:12] OK: backup ecommerce_analytics เสร็จ (ขนาด: 892M)
[2026-09-25 02:05:13] OK: ตรวจสอบความถูกต้องของ archive ทั้งหมดผ่าน
[2026-09-25 02:05:13] กำลัง sync backup ไปยัง s3://ecommerce-db-backups-ap-southeast/daily/
[2026-09-25 02:08:41] OK: sync ไป S3 เสร็จสิ้น
[2026-09-25 02:08:42] OK: ลบ backup เก่าตามนโยบาย retention เสร็จสิ้น
[2026-09-25 02:08:42] === backup ประเภท 'daily' เสร็จสมบูรณ์ | ขนาดรวม: 2.5G ===
```

### เช็คลิสต์ก่อนนำ script ไปใช้จริงใน production

- [ ] ทดสอบรัน script ด้วยมือสำเร็จ ไม่มี error
- [ ] ตั้งค่า `.pgpass` หรือ credential ให้ user ที่รัน cron (มักเป็น `postgres`) เชื่อมต่อได้โดยไม่ต้องพิมพ์ password
- [ ] ตรวจสอบว่า user ที่รัน backup มีสิทธิ์เพียงพอ (`pg_read_all_data` หรือเป็นเจ้าของ object)
- [ ] ตรวจสอบพื้นที่ disk เพียงพอสำหรับเก็บ backup ตามนโยบาย retention ทั้งหมด
- [ ] ทดสอบ Slack/email notification ทำงานจริง (ลองทำให้ backup fail โดยตั้งใจดูว่าแจ้งเตือนไหม)
- [ ] ตั้งค่า monitoring แยกต่างหาก (เช่น alert ถ้าไม่มี backup ใหม่ภายใน 25 ชั่วโมง — เผื่อ cron ไม่ทำงานเลย)
- [ ] เข้ารหัส backup ก่อน sync ไป cloud (โดยเฉพาะถ้ามีข้อมูลส่วนบุคคลของลูกค้า)
- [ ] ทดสอบ restore เต็มรูปแบบอย่างน้อยเดือนละครั้ง (ตาม Step 608)
- [ ] เขียน runbook แยกต่างหากสำหรับขั้นตอน disaster recovery แบบเต็ม ให้ทีมทุกคนเข้าถึงได้ (ไม่ใช่แค่คนเขียน script)

---

## สรุปท้ายบท

บทนี้พาคุณผ่านเครื่องมือ backup หลักทั้งสามตัวของ PostgreSQL อย่างละเอียด ตั้งแต่แนวคิดพื้นฐาน (RPO/RTO) ไปจนถึงการเขียน production script ที่ใช้งานได้จริง

### ตารางเปรียบเทียบเครื่องมือ backup ทั้งหมด

| เครื่องมือ | ประเภท | ขอบเขต | Format output | Parallel | เหมาะกับ |
|---|---|---|---|---|---|
| `pg_dump` (plain) | Logical | ฐานข้อมูลเดียว | `.sql` (text) | ไม่ได้ | Backup เล็ก, อ่าน/แก้ไขได้ด้วยตา, diff schema |
| `pg_dump` (custom) | Logical | ฐานข้อมูลเดียว | `.dump` (binary, compressed) | Restore ได้ | Backup ทั่วไปที่ใช้บ่อยที่สุด, restore เลือกบางส่วนได้ |
| `pg_dump` (directory) | Logical | ฐานข้อมูลเดียว | โฟลเดอร์ (compressed) | Dump + Restore ได้ | ฐานข้อมูลใหญ่ ต้องการความเร็วสูงสุดด้วย parallel |
| `pg_dump` (tar) | Logical | ฐานข้อมูลเดียว | `.tar` | Restore ได้ | ใช้น้อยในปัจจุบัน (custom/directory ดีกว่า) |
| `pg_dumpall` | Logical | ทั้ง cluster | `.sql` เท่านั้น | ไม่ได้ | Roles, tablespaces, backup ทั้ง cluster แบบง่าย |
| `pg_basebackup` | Physical | ทั้ง data directory | plain หรือ tar | Restore ทั้งชุด, dump `-j` (PG17+) | Disaster recovery, ตั้ง replica, PITR, ฐานข้อมูลขนาดใหญ่ที่ต้องการ RTO ต่ำ |

### สิ่งที่ควรจำจากบทนี้

1. **RPO/RTO ต้องกำหนดก่อนเลือกเครื่องมือ** ไม่ใช่เลือกเครื่องมือก่อนแล้วค่อยดูว่า RPO/RTO ได้เท่าไหร่
2. **Custom หรือ Directory format คือค่าเริ่มต้นที่แนะนำสำหรับ `pg_dump`** เพราะยืดหยุ่นและใช้ `pg_restore` ได้เต็มที่
3. **`pg_dumpall --globals-only` ต้องรันคู่กับ `pg_dump` เสมอ** เพราะ `pg_dump` ไม่สำรอง roles และ tablespaces
4. **Physical backup (`pg_basebackup`) เร็วกว่าและเหมาะกับ RTO ต่ำ** แต่ข้าม major version ไม่ได้
5. **Backup ที่ไม่เคยทดสอบ restore เท่ากับไม่มี backup** — ต้องมี automated restore test เป็นส่วนหนึ่งของ pipeline
6. **กฎ 3-2-1** — 3 สำเนา, 2 media type, 1 offsite — คือมาตรฐานขั้นต่ำสำหรับระบบ production ใดๆ ก็ตาม

บทถัดไป (Part 062) เราจะขยายความสามารถของ physical backup ไปสู่ **Point-in-Time Recovery (PITR)** ซึ่งใช้ WAL archiving ร่วมกับ `pg_basebackup` เพื่อกู้คืนข้อมูลกลับไป ณ วินาทีใดก็ได้ก่อนเกิดเหตุ ลด RPO ให้เหลือแทบเป็นศูนย์

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1

คำถาม: อธิบายความแตกต่างระหว่าง RPO และ RTO พร้อมยกตัวอย่างสถานการณ์ที่ระบบล่มเวลา 15:00 น. ถ้า RPO = 4 ชั่วโมง และ RTO = 1 ชั่วโมง หมายความว่าอย่างไร

<details>
<summary>เฉลย</summary>

**RPO (Recovery Point Objective)** คือระยะเวลาสูงสุดของข้อมูลที่ยอมรับให้สูญหายได้ นับย้อนจากเวลาที่เกิดเหตุ ส่วน **RTO (Recovery Time Objective)** คือระยะเวลาสูงสุดที่ยอมรับให้ระบบ down ได้ก่อนต้องกลับมาใช้งานได้ตามปกติ

ในสถานการณ์: ระบบล่มเวลา 15:00 น.

- RPO = 4 ชั่วโมง หมายความว่า เราต้องกู้คืนข้อมูลได้อย่างน้อยถึงเวลา 11:00 น. (15:00 - 4 ชม.) ข้อมูลที่เกิดขึ้นระหว่าง 11:00-15:00 น. อาจสูญหายไปได้ (แต่ต้องไม่เกินนี้)
- RTO = 1 ชั่วโมง หมายความว่า ระบบต้องกลับมาใช้งานได้ภายในเวลา 16:00 น. (15:00 + 1 ชม.) ไม่เช่นนั้นถือว่าไม่ผ่านเป้าหมาย

ทั้งสองค่านี้เป็นตัวกำหนดว่าเราต้อง backup บ่อยแค่ไหน (RPO) และต้องใช้กลยุทธ์กู้คืนแบบไหนที่เร็วพอ (RTO) — เช่น ถ้า RPO ต้องต่ำมาก อาจต้องใช้ WAL archiving/streaming replication แทนการ `pg_dump` วันละครั้ง

</details>

### แบบฝึกหัดที่ 2

คำถาม: เขียนคำสั่ง `pg_dump` เพื่อ backup ฐานข้อมูล `ecommerce_prod` เป็น custom format พร้อมบีบอัดระดับ 1 บันทึกไฟล์ชื่อ `ecommerce_backup.dump`

<details>
<summary>เฉลย</summary>

```bash
pg_dump -h localhost -U postgres -d ecommerce_prod \
  -Fc \
  -Z 1 \
  -f ecommerce_backup.dump
```

หมายเหตุ: `-Fc` เลือก custom format และ `-Z 1` ตั้งระดับ compression เป็น 1 ซึ่งในทางปฏิบัติมักคุ้มค่าที่สุด (ลดขนาดไฟล์ได้มากโดยใช้เวลาเพิ่มขึ้นไม่มาก เทียบกับระดับสูงสุด `-Z 9`)

</details>

### แบบฝึกหัดที่ 3

คำถาม: ถ้าต้องการ backup เฉพาะตาราง `orders` และ `order_items` โดยไม่เอาตาราง `audit_log` เข้ามาด้วย (สมมติว่าทั้งสามตารางอยู่ใน schema `public`) จะเขียนคำสั่ง `pg_dump` อย่างไร

<details>
<summary>เฉลย</summary>

มีสองวิธีที่เป็นไปได้ วิธีแรกคือระบุเฉพาะตารางที่ต้องการด้วย `-t` (วิธีนี้ตรงกับโจทย์ที่สุด เพราะระบุเฉพาะที่ต้องการ ไม่ต้อง exclude):

```bash
pg_dump -h localhost -U postgres -d ecommerce_prod \
  -t public.orders \
  -t public.order_items \
  -Fc \
  -f orders_backup.dump
```

วิธีที่สอง (ถ้าต้องการ backup ทุกตารางยกเว้น `audit_log`) ใช้ `-T` เพื่อ exclude:

```bash
pg_dump -h localhost -U postgres -d ecommerce_prod \
  -T public.audit_log \
  -Fc \
  -f no_audit_backup.dump
```

</details>

### แบบฝึกหัดที่ 4

คำถาม: มีไฟล์ backup `ecommerce_prod.dump` (custom format) ต้องการ restore เข้าฐานข้อมูลใหม่ชื่อ `ecommerce_test` แบบใช้ parallel 6 jobs จะเขียนคำสั่งอย่างไร (สมมติว่ายังไม่มีฐานข้อมูล `ecommerce_test` อยู่)

<details>
<summary>เฉลย</summary>

ต้องสร้างฐานข้อมูลเปล่าก่อน แล้วจึง restore ด้วย `-j`:

```bash
createdb -h localhost -U postgres ecommerce_test

pg_restore -h localhost -U postgres \
  -d ecommerce_test \
  -j 6 \
  -v \
  ecommerce_prod.dump
```

หรือใช้ `-C` เพื่อให้ `pg_restore` สร้างฐานข้อมูลให้อัตโนมัติ (ต้อง connect ผ่านฐานข้อมูลอื่น เช่น `postgres`):

```bash
pg_restore -h localhost -U postgres \
  -d postgres \
  -C \
  -j 6 \
  -v \
  ecommerce_prod.dump
```

</details>

### แบบฝึกหัดที่ 5

คำถาม: อธิบายว่าทำไม `pg_dump` เพียงอย่างเดียวไม่เพียงพอสำหรับการทำ disaster recovery แบบเต็มรูปแบบของทั้ง PostgreSQL cluster และต้องใช้เครื่องมืออะไรเพิ่มเติม

<details>
<summary>เฉลย</summary>

`pg_dump` สำรองได้แค่ **ข้อมูลระดับฐานข้อมูลเดียว** เท่านั้น มันไม่ได้สำรอง:

1. **Roles/Users ระดับ cluster** — `CREATE ROLE`, password hash, สิทธิ์ superuser ที่ใช้ร่วมกันทุกฐานข้อมูลใน cluster
2. **Tablespaces** (นิยาม location ของ tablespace แม้จะไม่รวมไฟล์ข้อมูลจริง)
3. **ฐานข้อมูลอื่นๆ** ใน cluster เดียวกัน (ถ้ามีหลายฐานข้อมูล ต้อง `pg_dump` แยกทีละฐาน)
4. **การตั้งค่าระดับ server** เช่น `postgresql.conf`, `pg_hba.conf` (ไฟล์ config เหล่านี้ไม่เกี่ยวกับ `pg_dump` เลย ต้อง backup แยกต่างหาก)

เพื่อให้ backup ทั้ง cluster ครบถ้วน ต้องใช้ `pg_dumpall --globals-only` เพื่อสำรอง roles และ tablespaces เพิ่มเติม (ควบคู่กับ `pg_dump` แยกแต่ละฐานข้อมูล) และต้อง backup ไฟล์ config (`postgresql.conf`, `pg_hba.conf`) แยกต่างหากด้วยตนเอง (เช่น เก็บไว้ใน version control หรือ backup ไฟล์ตรงๆ)

</details>

### แบบฝึกหัดที่ 6

คำถาม: เปรียบเทียบข้อดี-ข้อเสียของ Logical backup (`pg_dump`) กับ Physical backup (`pg_basebackup`) อย่างน้อยฝั่งละ 3 ข้อ

<details>
<summary>เฉลย</summary>

**Logical Backup (`pg_dump`)**

ข้อดี:
- เลือก backup เฉพาะบางตาราง/schema ได้
- ย้ายข้อมูลข้าม PostgreSQL major version ได้ (เช่น จาก 15 ไป 17)
- ย้ายข้อมูลข้าม OS/สถาปัตยกรรมได้ (x86 ↔ ARM)

ข้อเสีย:
- restore ช้ากว่ามาก เพราะต้องสร้าง index/constraint ใหม่ทั้งหมด
- ไม่สามารถใช้ตั้งค่า streaming replication ได้โดยตรง
- ไม่รองรับ Point-in-Time Recovery

**Physical Backup (`pg_basebackup`)**

ข้อดี:
- backup และ restore เร็วกว่ามาก (I/O copy ตรงๆ ไม่ผ่าน query engine)
- ใช้เป็นฐานสำหรับตั้งค่า streaming replication/standby ได้
- ใช้ร่วมกับ WAL archiving เพื่อทำ Point-in-Time Recovery ได้

ข้อเสีย:
- ข้าม major version ของ PostgreSQL ไม่ได้ (ต้องตรงกันเป๊ะ)
- ไม่สามารถเลือก backup เฉพาะบางตาราง/schema ได้ (ต้องเป็นทั้ง cluster เสมอ)
- ขนาดไฟล์ใหญ่กว่า (รวม index bloat และไฟล์ระบบทั้งหมด)

</details>

### แบบฝึกหัดที่ 7

คำถาม: เขียนคำสั่ง `pg_basebackup` เพื่อ backup ทั้ง cluster ไปยังโฟลเดอร์ `/backup/basebackup_today` แบบ plain format ใช้ WAL streaming method และแสดง progress

<details>
<summary>เฉลย</summary>

```bash
pg_basebackup \
  -h db-primary.internal \
  -U backup_user \
  -D /backup/basebackup_today \
  -Fp \
  -Xs \
  -P \
  -v
```

อธิบาย:
- `-D /backup/basebackup_today` — โฟลเดอร์ปลายทาง (ต้องว่างเปล่า)
- `-Fp` — plain format (คัดลอกไฟล์ตรงๆ ไม่ห่อ tar)
- `-Xs` — ใช้ WAL streaming method (เปิด connection ที่สอง stream WAL ระหว่าง backup ทำงาน เพื่อความ consistent)
- `-P` — แสดง progress bar
- `-v` — verbose

</details>

### แบบฝึกหัดที่ 8

คำถาม: ทำไม "backup ที่ไม่เคยทดสอบ restore" จึงถือว่าไม่ใช่ backup ที่ใช้งานได้จริง ยกตัวอย่างความเสี่ยง 3 อย่างที่อาจเกิดขึ้นกับ backup ที่ไม่เคยทดสอบ

<details>
<summary>เฉลย</summary>

เพราะ backup ที่ "สำเร็จ" (exit code 0, มีไฟล์ output) ไม่ได้แปลว่าไฟล์นั้น**ใช้ restore ได้จริง** ความเสี่ยงที่อาจเกิดขึ้นได้แก่:

1. **ไฟล์เสียหาย (corruption)** — เช่น disk error ระหว่างเขียนไฟล์, การ transfer ไฟล์ขาดหายกลางคัน ทำให้ไฟล์ backup ที่ดูเหมือนปกติ แต่จริงๆ restore ไม่ได้
2. **Script มี bug ที่ backup ผิดเป้าหมาย** — เช่น เชื่อมต่อฐานข้อมูลผิด host (เช่น backup staging แทน production โดยไม่ตั้งใจ) หรือ credential หมดอายุทำให้ backup ได้ไฟล์เปล่าแต่ script ไม่ตรวจสอบ exit code
3. **เวอร์ชัน PostgreSQL ไม่ compatible** — โดยเฉพาะ physical backup ที่ major version ของเครื่องปลายทางไม่ตรงกับต้นทาง ทำให้ restore ไม่ได้เลย

การทดสอบ restore เป็นประจำ (เช่น automated restore test ทุกวัน) จะช่วยจับปัญหาเหล่านี้ได้ตั้งแต่เนิ่นๆ ก่อนที่จะต้องใช้ backup จริงในสถานการณ์ฉุกเฉิน

</details>

### แบบฝึกหัดที่ 9

คำถาม: อธิบายกฎ 3-2-1 ของการทำ backup และยกตัวอย่างการ implement กฎนี้สำหรับระบบ e-commerce ที่มี production server อยู่ที่ datacenter กรุงเทพฯ

<details>
<summary>เฉลย</summary>

กฎ 3-2-1 ประกอบด้วย:

- **3** — เก็บสำเนาข้อมูลอย่างน้อย 3 ชุด (1 ต้นฉบับ + 2 backup)
- **2** — เก็บบน media/storage type ที่แตกต่างกันอย่างน้อย 2 ชนิด
- **1** — เก็บอย่างน้อย 1 ชุดไว้นอกสถานที่ (offsite)

ตัวอย่างการ implement สำหรับระบบที่ production อยู่กรุงเทพฯ:

1. **สำเนา 1 (ต้นฉบับ)**: ฐานข้อมูล production ที่ datacenter กรุงเทพฯ (local SSD)
2. **สำเนา 2**: `pg_dump`/`pg_basebackup` เก็บไว้บนเครื่อง backup server แยกต่างหาก ภายใน datacenter เดียวกัน (media type: local disk บนเครื่องอื่น)
3. **สำเนา 3**: sync backup ไปยัง cloud object storage (เช่น AWS S3) ที่ region สิงคโปร์หรือภูมิภาคอื่น (offsite, media type: cloud storage)

การกระจายแบบนี้ทำให้ถ้าเกิดเหตุที่ datacenter กรุงเทพฯ ทั้งหมด (เช่น น้ำท่วม, ไฟไหม้) ยังมีสำเนาที่สามที่ปลอดภัยอยู่ที่อื่น

</details>

### แบบฝึกหัดที่ 10

คำถาม: ออกแบบ backup script แนวคิด (pseudo-code หรือ bash) ที่ครอบคลุมขั้นตอนต่อไปนี้: (1) ตรวจสอบการเชื่อมต่อฐานข้อมูลก่อน backup (2) backup ด้วย pg_dump แบบ parallel (3) ตรวจสอบความถูกต้องของไฟล์ backup หลัง dump เสร็จ (4) แจ้งเตือนถ้าล้มเหลว

<details>
<summary>เฉลย</summary>

```bash
#!/bin/bash
set -euo pipefail

PG_HOST="db-primary.internal"
PG_USER="backup_user"
DB_NAME="ecommerce_prod"
BACKUP_FILE="/backup/${DB_NAME}_$(date +%Y%m%d_%H%M%S)"
SLACK_WEBHOOK="https://hooks.slack.com/services/XXX"

notify_fail() {
  curl -s -X POST "$SLACK_WEBHOOK" \
    -H 'Content-Type: application/json' \
    -d "{\"text\": \"Backup ล้มเหลว: $1\"}" > /dev/null
  exit 1
}

# 1. ตรวจสอบการเชื่อมต่อก่อน backup
pg_isready -h "$PG_HOST" -U "$PG_USER" || notify_fail "เชื่อมต่อฐานข้อมูลไม่ได้"

# 2. backup ด้วย pg_dump แบบ parallel (directory format)
pg_dump -h "$PG_HOST" -U "$PG_USER" -d "$DB_NAME" \
  -Fd -j 4 -f "$BACKUP_FILE" \
  || notify_fail "pg_dump ล้มเหลว"

# 3. ตรวจสอบความถูกต้องของไฟล์ backup
[ -f "${BACKUP_FILE}/toc.dat" ] || notify_fail "ไม่พบ toc.dat ไฟล์ backup อาจไม่สมบูรณ์"
pg_restore -l "$BACKUP_FILE" > /dev/null || notify_fail "archive เสียหาย อ่าน TOC ไม่ได้"

# 4. แจ้งเตือนเมื่อสำเร็จ (optional)
curl -s -X POST "$SLACK_WEBHOOK" \
  -H 'Content-Type: application/json' \
  -d "{\"text\": \"Backup $DB_NAME สำเร็จ: $BACKUP_FILE\"}" > /dev/null

echo "Backup สำเร็จ: $BACKUP_FILE"
```

หลักการสำคัญคือทุกขั้นตอนต้องมีการตรวจสอบผลลัพธ์ (exit code, ไฟล์ที่คาดว่าต้องมี) และเมื่อล้มเหลวต้องหยุดทันทีพร้อมแจ้งเตือน ไม่ปล่อยให้ script รันต่อไปแบบเงียบๆ

</details>

---

## บทถัดไป

จบบทนี้แล้ว เราได้เรียนรู้เครื่องมือ backup พื้นฐานที่จำเป็นครบทั้งหมดแล้ว แต่ backup แบบที่เรียนมายังมีข้อจำกัดเรื่อง RPO อยู่ (backup ได้แค่ ณ จุดเวลาที่ backup เท่านั้น ไม่สามารถกู้คืนไปยังวินาทีใดวินาทีหนึ่งได้)

บทถัดไปเราจะเรียนรู้ **Point-in-Time Recovery (PITR)** ซึ่งใช้ WAL archiving ร่วมกับ physical backup เพื่อให้สามารถกู้คืนข้อมูลกลับไปยังเวลาใดก็ได้ก่อนเกิดเหตุ (ไม่ใช่แค่เวลาที่ backup ล่าสุด) — เทคนิคนี้คือหัวใจสำคัญของระบบ production ระดับ enterprise ที่ต้องการ RPO ต่ำมากๆ

**บทถัดไป**: [Part 062: Point-in-Time Recovery](./part-062-point-in-time-recovery.md)
