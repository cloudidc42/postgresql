# Logical Replication และ Publication/Subscription

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับมืออาชีพ | Part 064

## เป้าหมายการเรียนรู้

เมื่อเรียนจบบทนี้ ผู้เรียนจะสามารถ:

- อธิบายความแตกต่างระหว่าง **Logical Replication** กับ **Physical Replication** ได้อย่างชัดเจน ทั้งในแง่กลไกภายในและการใช้งานจริง
- ระบุ use case ที่เหมาะกับ Logical Replication เช่น การอัปเกรดเวอร์ชัน PostgreSQL แบบ near-zero-downtime, การ replicate เฉพาะบางตาราง, การส่งข้อมูลข้าม major version
- ตั้งค่าเซิร์ฟเวอร์ PostgreSQL ให้รองรับ Logical Replication ผ่านพารามิเตอร์ `wal_level = logical`
- สร้างและจัดการ `PUBLICATION` บนฐานข้อมูลต้นทาง (publisher) ทั้งแบบ publish ทุกตาราง และแบบเจาะจงตาราง
- สร้างและจัดการ `SUBSCRIPTION` บนฐานข้อมูลปลายทาง (subscriber) พร้อมทำความเข้าใจ connection string และพารามิเตอร์สำคัญ
- เข้าใจข้อกำหนดของตารางที่จะ replicate ได้ เช่น Primary Key และ `REPLICA IDENTITY`
- ตรวจสอบสถานะการทำงานของ replication ผ่าน `pg_stat_subscription`, `pg_publication_tables` และ view อื่น ๆ ที่เกี่ยวข้อง
- รู้ข้อจำกัดของ Logical Replication เช่น DDL ไม่ถูก replicate อัตโนมัติ, sequence ไม่ถูก replicate, และการจัดการ `TRUNCATE`
- เข้าใจกลไกการเกิด conflict บนฝั่ง subscriber และวิธีรับมือ
- ลงมือปฏิบัติจริง: ตั้งค่า logical replication ระหว่างสองฐานข้อมูลในเครื่องเดียวกัน แล้วทดสอบ INSERT/UPDATE/DELETE แบบ end-to-end

ก่อนเริ่ม ขอแนะนำภาพรวมสั้น ๆ: logical replication ใน PostgreSQL เปิดตัวอย่างเป็นทางการตั้งแต่ PostgreSQL 10 ด้วยกลไก Publication/Subscription (แนวคิดคล้ายกับ pub/sub ในระบบ messaging) และได้รับการพัฒนาต่อเนื่องมาจนถึง PostgreSQL 16/17 ที่เพิ่มความสามารถอย่าง `pg_createsubscriber` (สร้าง subscriber จาก physical standby), row filter, column list, และการปรับปรุงประสิทธิภาพของ apply worker เนื้อหาทั้งหมดในบทนี้อ้างอิงไวยากรณ์ที่ใช้งานได้จริงบน PostgreSQL 16 และ 17

---

## Step 631: Logical Replication คืออะไร — ต่างจาก Physical Replication อย่างไร

### แนวคิดพื้นฐาน

PostgreSQL มีกลไก replication อยู่สองรูปแบบหลักที่ทำงานต่างกันโดยสิ้นเชิงในระดับภายใน:

| หัวข้อ | Physical Replication | Logical Replication |
|---|---|---|
| สิ่งที่ส่ง | WAL (Write-Ahead Log) แบบ byte-for-byte ของทั้ง cluster | การเปลี่ยนแปลงระดับแถว (row-level change) ที่ถอดรหัสจาก WAL |
| ขอบเขต | ทั้ง cluster (ทุกฐานข้อมูล ทุกตาราง) | เลือกได้เป็นรายตาราง หรือรายฐานข้อมูล |
| เวอร์ชัน PostgreSQL | ต้องเป็นเวอร์ชันและสถาปัตยกรรมเดียวกันทุกประการ (same major version, same architecture) | ต่าง major version กันได้ (เช่น publisher 15 → subscriber 17) |
| โหมดฐานข้อมูลปลายทาง | read-only (standby) จนกว่าจะ promote | เขียนได้ปกติ (read-write) ตลอดเวลา |
| กลไกเบื้องหลัง | Streaming Replication ผ่าน `pg_basebackup` + WAL streaming | `CREATE PUBLICATION` (ต้นทาง) + `CREATE SUBSCRIPTION` (ปลายทาง) ผ่าน logical decoding |
| DDL | replicate อัตโนมัติ (เพราะเป็น WAL ดิบทั้งหมด) | **ไม่** replicate อัตโนมัติ ต้องจัดการเอง |
| Sequence | replicate อัตโนมัติ | **ไม่** replicate อัตโนมัติ (ตั้งแต่ PG 16 มีคำสั่งช่วย sync ได้บางส่วน) |
| Use case หลัก | High Availability (HA), Disaster Recovery (DR) | Selective replication, zero-downtime upgrade, data integration, multi-master (ผ่านเครื่องมือเสริม) |

เนื้อหาเรื่อง Physical Replication (Streaming Replication, Standby, `pg_basebackup`) ได้กล่าวถึงไปแล้วในบทก่อนหน้า บทนี้จะโฟกัสเฉพาะ Logical Replication

### กลไกภายใน: Logical Decoding

หัวใจของ Logical Replication คือกระบวนการที่เรียกว่า **Logical Decoding** ซึ่งทำงานดังนี้:

1. เมื่อมีการ `INSERT`, `UPDATE`, `DELETE` เกิดขึ้นบน publisher ข้อมูลการเปลี่ยนแปลงจะถูกเขียนลง WAL ตามปกติ (เหมือนทุกครั้ง)
2. PostgreSQL มีปลั๊กอินถอดรหัส (output plugin) ชื่อ `pgoutput` (เป็นค่าเริ่มต้นที่มาพร้อมกับ core ตั้งแต่ PG 10) ทำหน้าที่ "แปล" WAL ดิบให้กลายเป็นชุดคำสั่งเปลี่ยนแปลงระดับแถวที่มีความหมาย (logical change) เช่น "แถวนี้ถูก insert ด้วยค่า X, Y, Z"
3. กระบวนการนี้ใช้กลไก **Replication Slot** แบบ logical (ต่างจาก physical slot) เพื่อรับประกันว่า WAL ที่ยังไม่ถูกส่งจะไม่ถูกลบทิ้งไปก่อน
4. Subscriber จะมี process ชื่อ **logical replication worker** (apply worker) คอยรับการเปลี่ยนแปลงเหล่านี้แล้ว apply เป็นคำสั่ง SQL ปกติ (INSERT/UPDATE/DELETE) ลงในตารางปลายทาง

ภาพรวมของกระบวนการ:

```
[Publisher DB]
   Transaction เขียนข้อมูล → WAL
        │
        ▼
   WAL Sender process ──(อ่าน WAL ผ่าน logical decoding / pgoutput)──┐
        │                                                            │
   Logical Replication Slot (กันไม่ให้ WAL ถูกลบก่อนส่งครบ)          │
                                                                      ▼
                                                          [Subscriber DB]
                                                          WAL Receiver
                                                                │
                                                                ▼
                                                     Logical Replication
                                                     Apply Worker
                                                                │
                                                                ▼
                                                     INSERT/UPDATE/DELETE
                                                     ลงตารางปลายทางจริง
```

### ทำไมต้องเข้าใจว่ามันคือ "การเปลี่ยนแปลงระดับแถว" ไม่ใช่ "WAL ดิบ"

ความแตกต่างนี้สำคัญมากเพราะมันอธิบายข้อดี-ข้อจำกัดของ logical replication ได้เกือบทั้งหมด:

- **ข้อดี**: เพราะส่งเป็น "การเปลี่ยนแปลงระดับแถว" ไม่ใช่ byte ดิบของไฟล์ฐานข้อมูล จึงไม่จำเป็นต้องมี on-disk format เดียวกัน → ข้าม major version ได้, เลือก replicate เฉพาะบางตารางได้, ฐานข้อมูลปลายทางเขียนข้อมูลอื่นเพิ่มเองได้ (read-write)
- **ข้อจำกัด**: เพราะไม่ได้ส่ง WAL ดิบทั้งหมด สิ่งที่ไม่ได้อยู่ในรูปแบบ "การเปลี่ยนแปลงข้อมูลระดับแถวของตารางที่ publish" จึงไม่ถูกส่งไปด้วย เช่น การเปลี่ยนโครงสร้างตาราง (DDL), การเปลี่ยนค่า sequence, large object เป็นต้น (จะกล่าวถึงรายละเอียดใน Step 638)

> **หมายเหตุเชิงคำศัพท์**: คำว่า "publisher" และ "subscriber" ในบทนี้หมายถึง **ฐานข้อมูล** (database) ไม่ใช่ instance ทั้งเซิร์ฟเวอร์ เพราะ logical replication ทำงานในระดับฐานข้อมูลเดียว (single database) เท่านั้น — publication ถูกสร้างในฐานข้อมูลใดฐานข้อมูลหนึ่ง และจะครอบคลุมเฉพาะตารางในฐานข้อมูลนั้น

---

## Step 632: Use case ของ Logical Replication

Logical Replication ไม่ได้ถูกออกแบบมาแทนที่ Physical Replication แต่ถูกออกแบบมาแก้ปัญหาที่ Physical Replication ทำไม่ได้ ลองดู use case หลัก ๆ ที่ใช้งานจริงในองค์กร:

### 1. Near-Zero-Downtime Major Version Upgrade

นี่คือ use case ที่ได้รับความนิยมมากที่สุด เพราะการอัปเกรด PostgreSQL major version แบบดั้งเดิม (เช่นใช้ `pg_dump`/`pg_restore` หรือ `pg_upgrade`) มักต้องหยุดระบบเป็นเวลานานพอสมควร โดยเฉพาะฐานข้อมูลขนาดใหญ่

ด้วย logical replication เราสามารถ:

1. สร้างเซิร์ฟเวอร์ PostgreSQL เวอร์ชันใหม่ (เช่นจาก 15 → 17)
2. ตั้งค่า publication บนเซิร์ฟเวอร์เก่า และ subscription บนเซิร์ฟเวอร์ใหม่
3. ปล่อยให้ข้อมูลซิงค์ตามทันกันแบบ real-time (initial sync + streaming changes)
4. เมื่อข้อมูลซิงค์ทันแล้ว ทำการสลับ (cutover) แอปพลิเคชันให้ชี้ไปที่เซิร์ฟเวอร์ใหม่ ซึ่งใช้เวลาหยุดระบบเพียงไม่กี่วินาทีถึงไม่กี่นาที (แทนที่จะเป็นชั่วโมง)

ตั้งแต่ PostgreSQL 16 ยังมีเครื่องมือ `pg_createsubscriber` ที่ช่วยแปลง physical standby ให้กลายเป็น logical subscriber ได้โดยตรง ทำให้กระบวนการนี้ง่ายและเร็วขึ้นอีก (ไม่ต้องทำ initial data copy ใหม่ทั้งหมด)

### 2. Replicate เฉพาะบางตาราง (Selective Replication)

Physical Replication ต้อง replicate ทั้ง cluster เสมอ แต่บางครั้งเราต้องการส่งข้อมูลแค่บางส่วนไปยังปลายทางอื่น เช่น:

- ส่งเฉพาะตาราง `products` และ `orders` ไปยังฐานข้อมูล reporting/analytics แยกต่างหาก โดยไม่ต้องส่งตาราง `audit_logs` หรือตารางที่มีข้อมูลอ่อนไหว (sensitive) ไปด้วย
- แต่ละทีมงาน (microservice) มีฐานข้อมูลของตัวเอง แต่ต้องการ "เห็น" ข้อมูลบางตารางจากทีมอื่นแบบ near real-time โดยไม่ต้องพึ่ง ETL batch job

### 3. รวมข้อมูลจากหลายแหล่ง (Data Consolidation / Fan-in)

subscriber หนึ่งตัวสามารถ subscribe จาก publisher ได้หลายตัว (multiple publications จากหลายฐานข้อมูล/เซิร์ฟเวอร์) ทำให้สร้างฐานข้อมูลกลางที่รวมข้อมูลจากหลายระบบเข้าด้วยกันได้ (เช่น รวมข้อมูลจากหลาย region เข้ามาที่ data warehouse กลาง)

### 4. ส่งข้อมูลข้าม Major Version หรือแม้แต่ข้าม OS/Architecture

เพราะ logical replication ไม่ผูกกับ on-disk binary format จึงสามารถ replicate ระหว่าง PostgreSQL คนละ major version ได้ (เช่น 14 → 17) และแม้แต่คนละสถาปัตยกรรมของเครื่อง (เช่น x86 → ARM) ซึ่ง physical replication ทำไม่ได้เลย

### 5. Testing / Staging Environment ที่มีข้อมูลจริงแบบ near real-time

ใช้ logical replication ส่งข้อมูลจาก production ไปยัง staging (เลือกเฉพาะตารางที่ไม่มีข้อมูลอ่อนไหว หรือ mask ข้อมูลบางส่วนด้วย row filter/column list ที่จะกล่าวถึงใน Step 634) เพื่อให้ทีมพัฒนาทดสอบกับข้อมูลที่ใกล้เคียงของจริง

### สรุปเปรียบเทียบเมื่อควรใช้อะไร

| ต้องการ | ใช้ |
|---|---|
| HA/DR ทั้ง cluster, failover อัตโนมัติ | Physical Replication (Streaming Replication) |
| อัปเกรด major version แบบ downtime สั้น | Logical Replication |
| replicate เฉพาะบางตาราง | Logical Replication |
| ส่งข้อมูลไปข้าม major version / คนละสถาปัตยกรรม | Logical Replication |
| standby อ่านได้แบบ byte-identical กับ primary (hot standby) | Physical Replication |

---

## Step 633: การตั้งค่าเริ่มต้น — wal_level=logical และการสร้าง PUBLICATION

### พารามิเตอร์ wal_level

PostgreSQL มีพารามิเตอร์ `wal_level` ที่กำหนดว่า WAL จะบันทึกข้อมูลละเอียดแค่ไหน มีสามค่า:

| ค่า | ความหมาย |
|---|---|
| `minimal` | บันทึกน้อยที่สุด เพียงพอสำหรับ crash recovery เท่านั้น (ไม่รองรับ replication ใด ๆ) |
| `replica` | บันทึกเพียงพอสำหรับ **Physical Replication** (ค่าเริ่มต้นของ PostgreSQL ตั้งแต่เวอร์ชัน 9.6 เป็นต้นมา) |
| `logical` | บันทึกข้อมูลเพิ่มเติมที่จำเป็นสำหรับ **Logical Decoding** เพียงพอสำหรับทั้ง Physical และ Logical Replication |

การจะใช้ Logical Replication ได้ **ต้อง** ตั้งค่า `wal_level = logical` เท่านั้น (ค่า `replica` ไม่พอ)

### ตรวจสอบค่าปัจจุบัน

```sql
SHOW wal_level;
```

ผลลัพธ์ตัวอย่าง (ค่าเริ่มต้นก่อนตั้งค่า):

```
 wal_level
-----------
 replica
(1 row)
```

### เปลี่ยนค่า wal_level

พารามิเตอร์นี้เป็นระดับ **server-wide** (ทั้ง cluster) และต้อง **restart** เซิร์ฟเวอร์หลังเปลี่ยนค่า (ไม่ใช่แค่ reload)

แก้ไขไฟล์ `postgresql.conf`:

```conf
wal_level = logical
```

หรือใช้คำสั่ง SQL (ต้องเป็น superuser) แล้ว restart เซิร์ฟเวอร์:

```sql
ALTER SYSTEM SET wal_level = 'logical';
```

จากนั้น restart เซิร์ฟเวอร์ (ตัวอย่างบน Linux ที่ใช้ systemd):

```bash
sudo systemctl restart postgresql
```

หรือถ้าใช้ `pg_ctl` โดยตรง:

```bash
pg_ctl restart -D /var/lib/postgresql/17/main
```

ตรวจสอบอีกครั้งหลัง restart:

```sql
SHOW wal_level;
```

```
 wal_level
-----------
 logical
(1 row)
```

> **หมายเหตุสำคัญ**: หากเปลี่ยนจาก `logical` กลับไปเป็น `replica` หรือ `minimal` ในภายหลัง ระบบจะปฏิเสธไม่ให้เปลี่ยนถ้ายังมี logical replication slot อยู่ในระบบ (ต้องลบ slot ก่อน) เพื่อป้องกันไม่ให้ subscriber ที่ยังใช้งานอยู่พังโดยไม่รู้ตัว

### พารามิเตอร์เสริมที่เกี่ยวข้อง

| พารามิเตอร์ | ความหมาย | ค่าที่แนะนำ |
|---|---|---|
| `max_replication_slots` | จำนวน replication slot สูงสุด (ใช้ร่วมกันทั้ง physical และ logical) | ต้อง ≥ จำนวน subscription/slot ที่ต้องใช้ (ค่าเริ่มต้น 10 มักเพียงพอสำหรับเริ่มต้น) |
| `max_wal_senders` | จำนวน process ที่ส่ง WAL ออกไปพร้อมกันได้สูงสุด | ต้อง ≥ จำนวน subscriber ที่เชื่อมเข้ามาพร้อมกัน |
| `max_logical_replication_workers` | จำนวน logical replication worker สูงสุด (ครอบคลุมทั้ง apply worker และ sync worker) | ปรับตามจำนวน subscription และตารางที่ sync พร้อมกัน |
| `max_worker_processes` | จำนวน background worker process สูงสุดทั้งหมดใน cluster | ต้องมากพอรองรับค่าด้านบนรวมกับ worker อื่น ๆ |

ตัวอย่างการตั้งค่าที่ครบถ้วนใน `postgresql.conf` สำหรับ lab ในบทนี้:

```conf
wal_level = logical
max_replication_slots = 10
max_wal_senders = 10
max_logical_replication_workers = 4
max_worker_processes = 8
```

### เตรียม Lab: สร้างฐานข้อมูล publisher และ subscriber บนเครื่องเดียวกัน

สำหรับบทนี้ เราจะจำลองสถานการณ์จริงโดยสร้างฐานข้อมูลสองฐานบน PostgreSQL instance เดียวกัน (คนละ database แต่ instance เดียวกันจริง ๆ ใช้งานได้เพราะ logical replication เชื่อมผ่าน connection string ธรรมดา ไม่จำเป็นต้องเป็นคนละเซิร์ฟเวอร์) ชื่อ `ecommerce_publisher` และ `ecommerce_subscriber`

```bash
createdb ecommerce_publisher
createdb ecommerce_subscriber
```

ตรวจสอบว่าถูกสร้างแล้ว:

```bash
psql -l | grep ecommerce
```

ผลลัพธ์ตัวอย่าง:

```
 ecommerce_publisher  | postgres | UTF8     | ...
 ecommerce_subscriber | postgres | UTF8     | ...
```

### สร้างตารางบนฝั่ง publisher

เชื่อมต่อไปที่ `ecommerce_publisher` แล้วสร้างตาราง `products` และ `orders`:

```bash
psql -d ecommerce_publisher
```

```sql
CREATE TABLE products (
    product_id   SERIAL PRIMARY KEY,
    product_name VARCHAR(150),
    unit_price   NUMERIC(10,2)
);

CREATE TABLE orders (
    order_id    SERIAL PRIMARY KEY,
    customer_id INTEGER,
    order_date  TIMESTAMPTZ DEFAULT now(),
    status      VARCHAR(20)
);

-- ใส่ข้อมูลตัวอย่างเริ่มต้นก่อน replicate
INSERT INTO products (product_name, unit_price) VALUES
    ('เมาส์ไร้สาย', 350.00),
    ('คีย์บอร์ดกลไก', 1290.00),
    ('จอมอนิเตอร์ 27 นิ้ว', 6990.00);

INSERT INTO orders (customer_id, status) VALUES
    (1001, 'pending'),
    (1002, 'completed');

SELECT * FROM products;
```

ผลลัพธ์:

```
 product_id |    product_name    | unit_price
------------+---------------------+------------
          1 | เมาส์ไร้สาย         |     350.00
          2 | คีย์บอร์ดกลไก       |    1290.00
          3 | จอมอนิเตอร์ 27 นิ้ว |    6990.00
(3 rows)
```

### สร้างโครงสร้างตารางเดียวกันบนฝั่ง subscriber (ล่วงหน้า)

จุดสำคัญที่ต้องเข้าใจ: **Logical Replication ไม่สร้างตารางให้อัตโนมัติ** ฝั่ง subscriber ต้องมีตารางที่มีโครงสร้าง (schema) ตรงกัน (อย่างน้อยต้องมีคอลัมน์ที่ตรงกับที่ publish) รออยู่ก่อนแล้ว

```bash
psql -d ecommerce_subscriber
```

```sql
CREATE TABLE products (
    product_id   SERIAL PRIMARY KEY,
    product_name VARCHAR(150),
    unit_price   NUMERIC(10,2)
);

CREATE TABLE orders (
    order_id    SERIAL PRIMARY KEY,
    customer_id INTEGER,
    order_date  TIMESTAMPTZ DEFAULT now(),
    status      VARCHAR(20)
);

-- ตรวจสอบว่าว่างเปล่า (ยังไม่มี subscription)
SELECT * FROM products;
```

```
 product_id | product_name | unit_price
------------+--------------+------------
(0 rows)
```

ถูกต้อง — ตอนนี้ตาราง `products` บน subscriber ว่างเปล่า เพราะยังไม่ได้สร้าง publication/subscription เลย

---

## Step 634: CREATE PUBLICATION — publish ทุกตาราง เทียบกับ publish เฉพาะตารางที่กำหนด

`PUBLICATION` คือ object ที่กำหนดว่า "จะส่งการเปลี่ยนแปลงของตารางไหนบ้างออกไป" สร้างบนฝั่ง publisher (ฐานข้อมูลต้นทาง) เท่านั้น

### รูปแบบไวยากรณ์ (syntax)

```sql
CREATE PUBLICATION name
    [ FOR ALL TABLES
      | FOR TABLE [ ONLY ] table_name [ * ] [, ...]
      | FOR TABLES IN SCHEMA schema_name [, ...] ]
    [ WITH ( publish = 'insert, update, delete, truncate' [, publish_via_partition_root = ...] ) ];
```

### 1) FOR ALL TABLES — publish ทุกตารางในฐานข้อมูล

```sql
CREATE PUBLICATION pub_all_tables FOR ALL TABLES;
```

วิธีนี้สะดวกแต่ต้องระวัง เพราะจะ publish **ทุกตาราง** ในฐานข้อมูลนั้นโดยอัตโนมัติ รวมถึงตารางที่สร้างขึ้นใหม่ในอนาคตด้วย (dynamic — ไม่ต้องแก้ publication ทุกครั้งที่สร้างตารางใหม่) เหมาะกับกรณีอัปเกรดเวอร์ชันที่ต้องการ replicate ทั้งฐานข้อมูล

การสร้าง `FOR ALL TABLES` ต้องมีสิทธิ์ superuser หรือ role ที่มี attribute `pg_write_all_data` (ตั้งแต่ PostgreSQL 16 เป็นต้นมา role ธรรมดาที่ได้รับสิทธิ์ `pg_create_subscription` และเป็นเจ้าของตารางสามารถทำงานบาง scenario ได้ แต่ `FOR ALL TABLES` ยังคงจำกัดสิทธิ์สูงเป็นค่าเริ่มต้น)

### 2) FOR TABLE — publish เฉพาะตารางที่ระบุ

สำหรับบทเรียนนี้ เราจะเลือกใช้วิธีนี้เพราะควบคุมได้ชัดเจนกว่า และตรงกับ use case "replicate เฉพาะบางตาราง" ที่กล่าวถึงใน Step 632:

```sql
-- เชื่อมต่อที่ ecommerce_publisher
\c ecommerce_publisher

CREATE PUBLICATION pub_ecommerce
    FOR TABLE products, orders;
```

ผลลัพธ์:

```
CREATE PUBLICATION
```

ตรวจสอบว่าสร้างสำเร็จ:

```sql
SELECT pubname, puballtables, pubinsert, pubupdate, pubdelete, pubtruncate
FROM pg_publication;
```

```
   pubname     | puballtables | pubinsert | pubupdate | pubdelete | pubtruncate
---------------+--------------+-----------+-----------+-----------+-------------
 pub_ecommerce | f            | t         | t         | t         | t
(1 row)
```

ตรวจสอบว่าตารางไหนอยู่ใน publication บ้าง ผ่าน `pg_publication_tables`:

```sql
SELECT pubname, schemaname, tablename
FROM pg_publication_tables
WHERE pubname = 'pub_ecommerce';
```

```
    pubname    | schemaname | tablename
---------------+------------+-----------
 pub_ecommerce | public     | products
 pub_ecommerce | public     | orders
(2 rows)
```

### 3) FOR TABLES IN SCHEMA — publish ทุกตารางใน schema ที่กำหนด (PostgreSQL 15+)

```sql
CREATE PUBLICATION pub_public_schema
    FOR TABLES IN SCHEMA public;
```

วิธีนี้อยู่กึ่งกลางระหว่างสองแบบแรก: ครอบคลุมทุกตารางใน schema ที่ระบุ (dynamic เหมือน `FOR ALL TABLES` แต่จำกัดขอบเขตแค่ schema เดียว)

### เพิ่ม/ลดตารางจาก publication ภายหลัง (ALTER PUBLICATION)

```sql
-- เพิ่มตาราง customers เข้าไปใน publication เดิม
ALTER PUBLICATION pub_ecommerce ADD TABLE customers;

-- ลบตาราง orders ออกจาก publication
ALTER PUBLICATION pub_ecommerce DROP TABLE orders;

-- ตั้งค่าตารางใหม่ทั้งหมด (แทนที่ของเดิม)
ALTER PUBLICATION pub_ecommerce SET TABLE products, orders, order_items;
```

### ควบคุมประเภทการเปลี่ยนแปลงที่ publish (WITH publish = ...)

โดยค่าเริ่มต้น publication จะส่งครบทั้ง `insert, update, delete, truncate` แต่สามารถจำกัดได้ เช่น กรณีต้องการทำ **append-only replication** (ส่งเฉพาะ insert ไม่ส่ง update/delete เพื่อเก็บ audit trail):

```sql
CREATE PUBLICATION pub_orders_insert_only
    FOR TABLE orders
    WITH (publish = 'insert');
```

### Column List และ Row Filter (PostgreSQL 15+)

ฟีเจอร์ใหม่ที่เพิ่มความยืดหยุ่นสูงมาก คือการเลือก publish เฉพาะบางคอลัมน์ (column list) และเฉพาะแถวที่ตรงเงื่อนไข (row filter):

```sql
-- publish เฉพาะบางคอลัมน์ของ products (ไม่ส่ง unit_price ออกไป)
CREATE PUBLICATION pub_products_public_cols
    FOR TABLE products (product_id, product_name);

-- publish เฉพาะ orders ที่ status = 'completed' เท่านั้น (row filter)
CREATE PUBLICATION pub_orders_completed
    FOR TABLE orders WHERE (status = 'completed');
```

ฟีเจอร์นี้มีประโยชน์มากสำหรับ use case ที่ 5 ใน Step 632 (ส่งข้อมูลไป staging โดยไม่ส่งคอลัมน์อ่อนไหว หรือส่งเฉพาะข้อมูลบางส่วน) — สำหรับ lab หลักของบทนี้เราจะใช้ `pub_ecommerce` (publish ทั้งตาราง ทุกคอลัมน์ ทุกแถว) เพื่อความเรียบง่ายในการทดสอบ end-to-end

---

## Step 635: การสร้าง SUBSCRIPTION บนฐานข้อมูลปลายทาง

`SUBSCRIPTION` คือ object ที่สร้างบนฝั่ง **subscriber** (ฐานข้อมูลปลายทาง) เพื่อเชื่อมต่อไปหา publication ที่สร้างไว้บน publisher

### รูปแบบไวยากรณ์

```sql
CREATE SUBSCRIPTION subscription_name
    CONNECTION 'conninfo'
    PUBLICATION publication_name [, ...]
    [ WITH ( option [= value] [, ... ] ) ];
```

### สร้าง Subscription จริงในเครื่องเดียวกัน

เชื่อมต่อไปที่ `ecommerce_subscriber`:

```bash
psql -d ecommerce_subscriber
```

```sql
CREATE SUBSCRIPTION sub_ecommerce
    CONNECTION 'host=localhost port=5432 dbname=ecommerce_publisher user=postgres password=yourpassword'
    PUBLICATION pub_ecommerce;
```

ผลลัพธ์ (เมื่อสำเร็จ):

```
NOTICE:  created replication slot "sub_ecommerce" on publisher
CREATE SUBSCRIPTION
```

สังเกตว่า PostgreSQL สร้าง **replication slot** ชื่อ `sub_ecommerce` บนฝั่ง publisher ให้อัตโนมัติ (ชื่อ slot ตรงกับชื่อ subscription โดยค่าเริ่มต้น) และเมื่อสร้างสำเร็จ ระบบจะเริ่มกระบวนการ **initial data synchronization** ทันที — คือการ copy ข้อมูลที่มีอยู่แล้วทั้งหมดในตารางที่ publish ไปยัง subscriber ก่อน แล้วจึงเริ่ม stream การเปลี่ยนแปลงต่อเนื่อง (incremental changes) ตามหลัง

### ตรวจสอบว่าข้อมูลเริ่มต้นถูก copy มาแล้ว

รอสักครู่ (สำหรับข้อมูลน้อย ๆ ใช้เวลาแค่เสี้ยววินาที) แล้วตรวจสอบ:

```sql
SELECT * FROM products;
```

```
 product_id |    product_name    | unit_price
------------+---------------------+------------
          1 | เมาส์ไร้สาย         |     350.00
          2 | คีย์บอร์ดกลไก       |    1290.00
          3 | จอมอนิเตอร์ 27 นิ้ว |    6990.00
(3 rows)
```

```sql
SELECT * FROM orders;
```

```
 order_id | customer_id |          order_date           |  status
----------+-------------+--------------------------------+-----------
        1 |        1001 | 2026-09-25 10:15:22.123456+00 | pending
        2 |        1002 | 2026-09-25 10:15:22.156789+00 | completed
(2 rows)
```

ข้อมูลที่มีอยู่ก่อนหน้าถูก copy มาแล้วครบถ้วน! ต่อไปเราจะทดสอบว่าการเปลี่ยนแปลงใหม่ ๆ ถูก replicate แบบ real-time หรือไม่ (รายละเอียดการทดสอบเต็มรูปแบบอยู่ใน Step 640)

### พารามิเตอร์สำคัญของ CREATE SUBSCRIPTION

| พารามิเตอร์ (WITH option) | ความหมาย | ค่าเริ่มต้น |
|---|---|---|
| `enabled` | เริ่มทำงานทันทีหรือไม่ | `true` |
| `create_slot` | ให้สร้าง replication slot บน publisher อัตโนมัติหรือไม่ | `true` |
| `slot_name` | ชื่อ slot ที่จะใช้/สร้าง (ถ้าไม่ระบุ ใช้ชื่อเดียวกับ subscription) | ชื่อ subscription |
| `copy_data` | ให้ copy ข้อมูลเดิมที่มีอยู่แล้วหรือไม่ (initial sync) | `true` |
| `connect` | ให้เชื่อมต่อไปยัง publisher จริง ๆ ตอนสร้างหรือไม่ | `true` |
| `synchronous_commit` | ระดับความปลอดภัยของการ commit ฝั่ง subscriber | `off` |
| `binary` | ส่งข้อมูลในรูปแบบ binary แทน text (เร็วกว่า แต่ต้องเวอร์ชัน/type เข้ากันได้) | `false` |
| `streaming` | streaming การเปลี่ยนแปลงของ transaction ขนาดใหญ่แบบ in-progress (PG 14+) | `off` (PG16) / `parallel` เป็นตัวเลือกใน PG17 |
| `password_required` | บังคับให้ connection string ต้องมี password (เพื่อความปลอดภัย) | `true` |
| `origin` | ควบคุมว่าจะรับการเปลี่ยนแปลงที่มี origin จากที่ไหนบ้าง (สำคัญมากสำหรับป้องกัน infinite loop ใน bi-directional replication) | `any` |

ตัวอย่างการสร้างแบบระบุ options เพิ่มเติม:

```sql
CREATE SUBSCRIPTION sub_ecommerce_v2
    CONNECTION 'host=localhost port=5432 dbname=ecommerce_publisher user=postgres password=yourpassword'
    PUBLICATION pub_ecommerce
    WITH (
        copy_data = true,
        create_slot = true,
        enabled = true,
        slot_name = 'sub_ecommerce_v2_slot'
    );
```

### จัดการ Subscription ที่มีอยู่

```sql
-- หยุดการทำงานชั่วคราว (หยุด apply worker แต่ slot ยังอยู่)
ALTER SUBSCRIPTION sub_ecommerce DISABLE;

-- เริ่มทำงานต่อ
ALTER SUBSCRIPTION sub_ecommerce ENABLE;

-- เปลี่ยน connection string (เช่น publisher ย้ายเครื่อง)
ALTER SUBSCRIPTION sub_ecommerce
    CONNECTION 'host=new-publisher-host port=5432 dbname=ecommerce_publisher user=repl_user password=xxx';

-- เพิ่ม publication อื่นเข้ามาใน subscription เดิม
ALTER SUBSCRIPTION sub_ecommerce
    ADD PUBLICATION pub_orders_completed;

-- refresh รายการตาราง (เมื่อ publication เพิ่ม/ลดตาราง ต้อง refresh subscriber ให้รับรู้)
ALTER SUBSCRIPTION sub_ecommerce REFRESH PUBLICATION;

-- ลบ subscription (จะลบ slot บน publisher ให้อัตโนมัติด้วย ถ้ายังเชื่อมต่อได้)
DROP SUBSCRIPTION sub_ecommerce;
```

> **ข้อควรระวัง**: `DROP SUBSCRIPTION` จะพยายามเชื่อมต่อไปที่ publisher เพื่อลบ replication slot ให้ด้วยอัตโนมัติ หาก publisher เชื่อมต่อไม่ได้ในขณะนั้น คำสั่งจะ error ทางแก้คือใช้ `ALTER SUBSCRIPTION ... SET (slot_name = NONE)` ก่อน แล้วค่อย `DROP SUBSCRIPTION` และไปลบ slot บน publisher ด้วยตนเองภายหลัง

---

## Step 636: ข้อกำหนดของตารางที่จะ replicate ได้ — Primary Key และ REPLICA IDENTITY

### ทำไมต้องมี Primary Key

Logical Replication apply การเปลี่ยนแปลงบน subscriber ในรูปของคำสั่ง SQL (ไม่ใช่ WAL ดิบ) เมื่อเกิดคำสั่ง `UPDATE` หรือ `DELETE` บน publisher ระบบจำเป็นต้อง "บอก" subscriber ว่า **แถวไหน** ที่ต้อง update/delete บนฝั่งปลายทาง

สำหรับ `INSERT` ไม่มีปัญหา เพราะแค่เพิ่มแถวใหม่เข้าไป แต่สำหรับ `UPDATE`/`DELETE` ระบบต้องมีวิธีระบุแถวเป้าหมายอย่างแม่นยำ ซึ่งค่าเริ่มต้นคือใช้ **Primary Key**

### REPLICA IDENTITY คืออะไร

`REPLICA IDENTITY` คือ setting ระดับตารางที่กำหนดว่า เมื่อเกิด `UPDATE`/`DELETE` จะบันทึกค่าคอลัมน์ชุดไหนลง WAL เพื่อใช้ระบุแถวเก่า (old row) มี 4 โหมด:

| โหมด | ความหมาย |
|---|---|
| `DEFAULT` | ใช้ Primary Key ของตาราง (ถ้ามี) — เป็นค่าเริ่มต้นของทุกตาราง |
| `USING INDEX index_name` | ใช้ unique index ที่ระบุแทน Primary Key (ต้องเป็น unique, not-null, not partial, not deferrable) |
| `FULL` | บันทึกค่า**ทุกคอลัมน์**ของแถวเดิมลง WAL (ใช้ได้แม้ไม่มี PK แต่กิน WAL และ overhead มากกว่ามาก) |
| `NOTHING` | ไม่บันทึกข้อมูลระบุแถวเดิมเลย → ตารางนี้จะ**ไม่รองรับ** UPDATE/DELETE ผ่าน logical replication (ทำได้แค่ publish INSERT) |

### ตรวจสอบ REPLICA IDENTITY ปัจจุบันของตาราง

```sql
SELECT relname, relreplident
FROM pg_class
WHERE relname IN ('products', 'orders');
```

```
 relname  | relreplident
----------+---------------
 products | d
 orders   | d
(2 rows)
```

ค่า `relreplident` มีความหมายดังนี้: `d` = default (ใช้ PK), `f` = full, `i` = index, `n` = nothing

เพราะตาราง `products` และ `orders` ในบทนี้มี `PRIMARY KEY` (`product_id`, `order_id`) อยู่แล้ว ค่า default (`d`) จึงเพียงพอสำหรับการ replicate UPDATE/DELETE ได้อย่างถูกต้องโดยไม่ต้องตั้งค่าเพิ่มเติมใด ๆ

### กรณีตารางไม่มี Primary Key

สมมติมีตารางที่ไม่มี PK (เช่นตาราง log ชั่วคราว) แล้วต้องการ replicate ต้องตั้งค่า `REPLICA IDENTITY FULL`:

```sql
-- ตัวอย่างตารางไม่มี PK
CREATE TABLE order_events (
    order_id   INTEGER,
    event_type VARCHAR(30),
    event_time TIMESTAMPTZ DEFAULT now()
);

-- ต้องตั้ง REPLICA IDENTITY FULL เพื่อให้ UPDATE/DELETE replicate ได้
ALTER TABLE order_events REPLICA IDENTITY FULL;
```

ตรวจสอบผล:

```sql
SELECT relname, relreplident
FROM pg_class
WHERE relname = 'order_events';
```

```
   relname    | relreplident
--------------+---------------
 order_events | f
(1 row)
```

> **คำเตือนเรื่องประสิทธิภาพ**: `REPLICA IDENTITY FULL` ทำให้ทุกครั้งที่ `UPDATE`/`DELETE` ต้องบันทึกค่าทุกคอลัมน์ของแถวเดิมลง WAL (เพิ่มขนาด WAL อย่างมีนัยสำคัญ) และฝั่ง subscriber เมื่อรับคำสั่ง `UPDATE`/`DELETE` แบบ `FULL` จะต้องทำ **sequential scan หรือ full table match** เพื่อหาแถวที่ตรงกันทุกคอลัมน์ (เพราะไม่มี index รับประกันความ unique) ซึ่งช้ามากถ้าตารางมีขนาดใหญ่ — ทางออกที่ดีกว่าคือเพิ่ม Primary Key หรือ unique index ให้ตารางนั้นแทนถ้าเป็นไปได้

### กรณีใช้ Unique Index แทน Primary Key

ถ้าตารางไม่มี PK แต่มี unique index ที่เหมาะสม (เช่น unique index บน `product_code`) สามารถใช้แทนได้:

```sql
CREATE UNIQUE INDEX idx_products_code_unique ON products (product_code);
ALTER TABLE products REPLICA IDENTITY USING INDEX idx_products_code_unique;
```

วิธีนี้ให้ประสิทธิภาพใกล้เคียงกับการมี Primary Key เพราะ subscriber ยังใช้ index lookup ในการหาแถวได้เหมือนเดิม

### สรุปกฎสำคัญ

1. ตารางที่จะ replicate ผ่าน publication **ควรมี Primary Key** เสมอ (เป็น best practice) — ตาราง `products`, `orders` ในบทเรียนนี้ทำถูกต้องแล้วตั้งแต่แรก
2. ถ้าไม่มี PK และต้องการรองรับ UPDATE/DELETE ต้องตั้ง `REPLICA IDENTITY FULL` หรือ `USING INDEX` (unique index)
3. ถ้าตารางไม่มี PK และไม่ได้ตั้ง REPLICA IDENTITY ใด ๆ (ยังเป็น `DEFAULT` แต่ไม่มี PK) การพยายาม `UPDATE`/`DELETE` บนตารางนั้นบน publisher จะ**เกิด error** ทันที ไม่ใช่แค่ replicate ไม่ได้เฉย ๆ:

```sql
-- ตัวอย่าง error ที่จะเกิดถ้าพยายาม UPDATE ตารางที่ publish อยู่ แต่ไม่มี PK/REPLICA IDENTITY
UPDATE order_events_no_pk SET event_type = 'shipped' WHERE order_id = 1;
```

```
ERROR:  cannot update table "order_events_no_pk" because it does not have a replica identity and publishes updates
HINT:  To enable updating the table, set REPLICA IDENTITY using ALTER TABLE.
```

ข้อความ error นี้ชัดเจนมาก — PostgreSQL จะป้องกันไม่ให้เกิดสถานการณ์ที่ข้อมูล publish ไม่ครบถ้วนโดยการปฏิเสธคำสั่งตั้งแต่ต้นทาง

---

## Step 637: การ monitor สถานะ Subscription — pg_stat_subscription, pg_publication_tables

การมอนิเตอร์ logical replication เป็นเรื่องสำคัญมากในการใช้งานจริง เพราะถ้า apply worker ค้างหรือ lag สะสม อาจทำให้ข้อมูลปลายทางไม่ทันสมัย หรือแย่กว่านั้นคือ WAL บน publisher โตขึ้นเรื่อย ๆ จนดิสก์เต็ม (เพราะ replication slot กัน WAL ไม่ให้ถูกลบจนกว่าจะถูกส่งไปหมด)

### ฝั่ง Publisher: ตรวจสอบ Publication และ Replication Slot

**1) `pg_publication`** — รายการ publication ทั้งหมด

```sql
SELECT pubname, pubowner::regrole, puballtables,
       pubinsert, pubupdate, pubdelete, pubtruncate
FROM pg_publication;
```

```
   pubname     | pubowner | puballtables | pubinsert | pubupdate | pubdelete | pubtruncate
---------------+----------+--------------+-----------+-----------+-----------+-------------
 pub_ecommerce | postgres | f            | t         | t         | t         | t
(1 row)
```

**2) `pg_publication_tables`** — รายการตารางในแต่ละ publication

```sql
SELECT pubname, schemaname, tablename, attnames, rowfilter
FROM pg_publication_tables
WHERE pubname = 'pub_ecommerce';
```

```
    pubname    | schemaname | tablename |         attnames          | rowfilter
---------------+------------+-----------+----------------------------+-----------
 pub_ecommerce | public     | products  | {product_id,product_name,unit_price} |
 pub_ecommerce | public     | orders    | {order_id,customer_id,order_date,status} |
(2 rows)
```

**3) `pg_replication_slots`** — สถานะ replication slot (สำคัญมาก ต้องเช็คเป็นประจำ)

```sql
SELECT slot_name, plugin, slot_type, active, active_pid,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS retained_wal
FROM pg_replication_slots
WHERE slot_type = 'logical';
```

```
   slot_name   | plugin   | slot_type | active | active_pid | retained_wal
---------------+----------+-----------+--------+------------+--------------
 sub_ecommerce | pgoutput | logical   | t      |      54321 | 0 bytes
(1 row)
```

คอลัมน์ `retained_wal` คือขนาด WAL ที่ยังไม่ถูกส่งไปให้ subscriber (ถูกกันไว้ไม่ให้ลบ) — ค่านี้ยิ่งเยอะแปลว่า subscriber ยิ่ง lag มาก ถ้าค่านี้โตขึ้นเรื่อย ๆ ไม่หยุด (เช่น subscriber ปิดเครื่องไปนาน) ต้องรีบตรวจสอบก่อนดิสก์บน publisher เต็ม

**4) `pg_stat_replication`** — สถานะการเชื่อมต่อของ WAL sender (ใช้ร่วมกับทั้ง physical และ logical)

```sql
SELECT pid, usename, application_name, client_addr, state,
       sent_lsn, write_lsn, flush_lsn, replay_lsn
FROM pg_stat_replication;
```

```
  pid  | usename  | application_name | client_addr | state     | sent_lsn  | write_lsn | flush_lsn | replay_lsn
-------+----------+------------------+-------------+-----------+-----------+-----------+-----------+------------
 54321 | postgres | sub_ecommerce    | 127.0.0.1   | streaming | 0/1A2B3C0 | 0/1A2B3C0 | 0/1A2B3C0 | 0/1A2B3C0
(1 row)
```

`application_name` จะเป็นชื่อ subscription โดยค่าเริ่มต้น ทำให้ระบุตัวตนได้ง่าย

### ฝั่ง Subscriber: ตรวจสอบ Subscription

**1) `pg_subscription`** — รายการ subscription ทั้งหมด

```sql
SELECT subname, subenabled, subconninfo, subpublications
FROM pg_subscription;
```

```
    subname    | subenabled |                        subconninfo                         | subpublications
---------------+------------+-------------------------------------------------------------+------------------
 sub_ecommerce | t          | host=localhost port=5432 dbname=ecommerce_publisher user=postgres | {pub_ecommerce}
(1 row)
```

**2) `pg_stat_subscription`** — สถานะการทำงานของ apply worker (สำคัญที่สุดสำหรับมอนิเตอร์ lag)

```sql
SELECT subid, subname, pid, relid, received_lsn,
       last_msg_send_time, last_msg_receipt_time,
       latest_end_lsn, latest_end_time
FROM pg_stat_subscription;
```

```
 subid |    subname    |  pid  | relid | received_lsn | last_msg_send_time            | last_msg_receipt_time         | latest_end_lsn | latest_end_time
-------+---------------+-------+-------+---------------+--------------------------------+--------------------------------+----------------+-----------------
 16400 | sub_ecommerce | 54322 |       | 0/1A2B3C0     | 2026-09-25 10:20:01.123456+00 | 2026-09-25 10:20:01.125000+00 | 0/1A2B3C0      | 2026-09-25 10:20:01.125000+00
(1 row)
```

การคำนวณ **replication lag** ทำได้จากผลต่างระหว่าง `last_msg_receipt_time` กับ `last_msg_send_time` หรือเปรียบเทียบ LSN ระหว่างฝั่ง publisher (`pg_current_wal_lsn()`) กับ `received_lsn`/`latest_end_lsn` ของ subscriber:

```sql
-- รันบนฝั่ง publisher เพื่อดู current LSN
SELECT pg_current_wal_lsn();

-- เทียบกับ latest_end_lsn บน subscriber (pg_stat_subscription)
-- ถ้าค่าต่างกันมาก แปลว่า subscriber ตามหลัง
```

**3) `pg_stat_subscription_stats`** (PostgreSQL 15+) — สถิติสะสม เช่นจำนวน error, จำนวนครั้งที่ apply worker ถูก restart

```sql
SELECT subname, apply_error_count, sync_error_count,
       confl_insert_exists, confl_update_origin_differs,
       confl_update_exists, confl_update_missing, confl_delete_origin_differs,
       confl_delete_missing, confl_multiple_unique_conflicts
FROM pg_stat_subscription_stats;
```

```
    subname    | apply_error_count | sync_error_count | confl_insert_exists | ...
---------------+-------------------+-------------------+----------------------+-----
 sub_ecommerce |                 0 |                 0 |                    0 | ...
(1 row)
```

view นี้มีประโยชน์มากในการตรวจจับ conflict ที่เกิดขึ้น (รายละเอียดเรื่อง conflict อยู่ใน Step 639) — คอลัมน์ `confl_*` ต่าง ๆ ถูกเพิ่มเข้ามาใน PostgreSQL 18 เพื่อแยกประเภท conflict อย่างละเอียด ในเวอร์ชัน 16/17 ตารางนี้จะมีเฉพาะ `apply_error_count` และ `sync_error_count` (ใช้ log เป็นหลักในการดูรายละเอียด conflict)

**4) ตรวจสอบ process บนฝั่ง subscriber ผ่าน pg_stat_activity**

```sql
SELECT pid, backend_type, state, query
FROM pg_stat_activity
WHERE backend_type LIKE '%logical replication%';
```

```
  pid  |          backend_type           | state  | query
-------+----------------------------------+--------+-------
 54322 | logical replication apply worker | idle   |
(1 row)
```

### ตารางสรุป view สำคัญที่ต้องจำ

| View | รันที่ | ใช้ดูอะไร |
|---|---|---|
| `pg_publication` | Publisher | รายการ publication |
| `pg_publication_tables` | Publisher | ตารางในแต่ละ publication |
| `pg_replication_slots` | Publisher | สถานะ slot, WAL ที่ค้างอยู่ (`retained_wal`) |
| `pg_stat_replication` | Publisher | สถานะการเชื่อมต่อ WAL sender แต่ละตัว |
| `pg_subscription` | Subscriber | รายการ subscription |
| `pg_stat_subscription` | Subscriber | สถานะ apply worker, LSN ล่าสุด |
| `pg_stat_subscription_stats` | Subscriber | สถิติ error/conflict สะสม |

---

## Step 638: ข้อจำกัดของ Logical Replication

แม้ Logical Replication จะยืดหยุ่นมาก แต่ก็มีข้อจำกัดสำคัญหลายอย่างที่ต้องเข้าใจก่อนนำไปใช้งานจริง มิฉะนั้นอาจเจอปัญหาที่ไม่คาดคิด

### 1. DDL ไม่ถูก replicate อัตโนมัติ

นี่คือข้อจำกัดที่สำคัญที่สุดและมักทำให้ผู้เริ่มต้นสับสน เพราะ logical decoding ทำงานจาก WAL ที่แปลงเป็น "การเปลี่ยนแปลงข้อมูลระดับแถว" เท่านั้น คำสั่ง DDL เช่น `CREATE TABLE`, `ALTER TABLE`, `DROP TABLE`, `CREATE INDEX` จะ**ไม่**ถูกส่งไปยัง subscriber โดยอัตโนมัติ

ทดสอบให้เห็นภาพ — เพิ่มคอลัมน์ใหม่บน publisher:

```sql
-- รันบน ecommerce_publisher
ALTER TABLE products ADD COLUMN category VARCHAR(50);
```

```
ALTER TABLE
```

ตรวจสอบบน subscriber:

```sql
-- รันบน ecommerce_subscriber
\d products
```

```
                                     Table "public.products"
    Column    |          Type          | Collation | Nullable |               Default
--------------+-------------------------+-----------+----------+--------------------------------------
 product_id   | integer                 |           | not null | nextval('products_product_id_seq'...)
 product_name | character varying(150)  |           |          |
 unit_price   | numeric(10,2)           |           |          |
```

จะเห็นว่าคอลัมน์ `category` **ไม่ปรากฏ** บน subscriber เลย — นี่คือพฤติกรรมที่คาดหวัง เพราะ DDL ไม่ replicate อัตโนมัติ

ถ้าตอนนี้ลอง insert ข้อมูลใหม่บน publisher (ที่มีค่า `category`) จะเกิดอะไรขึ้น:

```sql
-- รันบน ecommerce_publisher
INSERT INTO products (product_name, unit_price, category)
VALUES ('หูฟังบลูทูธ', 890.00, 'อิเล็กทรอนิกส์');
```

apply worker บน subscriber จะพยายาม insert แถวนี้ แต่เนื่องจาก subscriber มีคอลัมน์แค่ `product_id, product_name, unit_price` การส่งค่าคอลัมน์ที่ subscriber ไม่มีจะไม่ทำให้เกิด error (PostgreSQL logical replication ฉลาดพอที่จะ map เฉพาะคอลัมน์ที่มีชื่อตรงกัน) แต่ถ้า subscriber มีคอลัมน์ที่ publisher ไม่มีและคอลัมน์นั้น `NOT NULL` โดยไม่มีค่า default จะทำให้ apply worker error ทันที ดังนั้น **แนวปฏิบัติที่ถูกต้อง** คือต้องรัน DDL เดียวกันบน subscriber ด้วยตนเองก่อนหรือพร้อมกับที่รันบน publisher:

```sql
-- รันบน ecommerce_subscriber ด้วยตนเองก่อน
ALTER TABLE products ADD COLUMN category VARCHAR(50);
```

หลังจากนั้นข้อมูล `category` ของแถวใหม่ที่ insert หลังจากนี้จะถูก replicate ปกติ (แต่แถวเก่าที่เคย insert ไปแล้วตอนที่คอลัมน์ยังไม่ sync กัน จะไม่ backfill ค่าย้อนหลังให้ ต้องจัดการเองถ้าจำเป็น)

> **แนวปฏิบัติมาตรฐานในองค์กร**: กำหนดขั้นตอนการ deploy DDL ที่ต้องรันบน publisher และ subscriber "พร้อมกัน" หรือ "subscriber ก่อน publisher" เสมอ (เพิ่มคอลัมน์ที่ nullable/มี default บน subscriber ก่อน แล้วค่อยเพิ่มบน publisher) เพื่อป้องกัน replication หยุดชะงักจาก DDL drift

### 2. Sequence ไม่ถูก replicate

ค่าปัจจุบันของ `SEQUENCE` (เช่น sequence ที่ใช้กับ `SERIAL`/`GENERATED ... AS IDENTITY`) **ไม่**ถูก replicate ไปด้วย ทดสอบ:

```sql
-- รันบน ecommerce_publisher
SELECT last_value FROM products_product_id_seq;
```

```
 last_value
------------
          4
(1 row)
```

```sql
-- รันบน ecommerce_subscriber
SELECT last_value FROM products_product_id_seq;
```

```
 last_value
------------
          1
(1 row)
```

ค่าไม่ตรงกัน! นี่เป็นเรื่องปกติของ logical replication เพราะ sequence ไม่ได้เป็นส่วนหนึ่งของ "การเปลี่ยนแปลงข้อมูลระดับแถว" ของตาราง — ปัญหานี้สำคัญมากในสถานการณ์ **failover/cutover** (เช่นตอนอัปเกรดเวอร์ชันแล้วสลับให้แอปเขียนที่ subscriber แทน) เพราะถ้า sequence บน subscriber ยังไม่ทัน ค่า PK ใหม่ที่ generate อาจไปชนกับค่าที่เคย replicate มาจาก publisher (duplicate key)

**วิธีแก้ไข**: ก่อน cutover ต้อง sync ค่า sequence ด้วยตนเอง

```sql
-- รันบน ecommerce_subscriber ก่อน cutover
SELECT setval('products_product_id_seq',
    (SELECT last_value FROM dblink('host=localhost dbname=ecommerce_publisher',
     'SELECT last_value FROM products_product_id_seq') AS t(last_value bigint)));
```

หรือวิธีที่ง่ายกว่าคือ query ค่าจาก publisher มาด้วยตนเองแล้วตั้งค่าตรง ๆ:

```sql
-- Query บน publisher ก่อน
SELECT last_value, is_called FROM products_product_id_seq;
-- ได้ last_value = 4, is_called = t

-- แล้วรันบน subscriber
SELECT setval('products_product_id_seq', 4, true);
```

ตั้งแต่ PostgreSQL 17 เป็นต้นมา logical replication รองรับการ replicate ค่า sequence ได้บางส่วนผ่านคำสั่ง `ALTER SUBSCRIPTION ... REFRESH PUBLICATION SEQUENCES` (ใช้ร่วมกับเครื่องมือ `pg_createsubscriber` สำหรับ scenario การอัปเกรดโดยเฉพาะ) แต่ในการใช้งาน publication/subscription แบบทั่วไปยังคงต้อง sync sequence ด้วยตนเองเป็นหลัก

### 3. TRUNCATE ต้องเปิดใช้งานพิเศษ

คำสั่ง `TRUNCATE` จะถูก replicate ก็ต่อเมื่อ publication เปิด option `truncate` ไว้ (ค่าเริ่มต้นเปิดอยู่แล้วถ้าไม่ได้ระบุ `publish` เป็นอย่างอื่น) ทดสอบ:

```sql
-- ตรวจสอบว่า publication เรารองรับ truncate หรือไม่
SELECT pubname, pubtruncate FROM pg_publication WHERE pubname = 'pub_ecommerce';
```

```
   pubname     | pubtruncate
---------------+-------------
 pub_ecommerce | t
(1 row)
```

ถ้า `pubtruncate = t` คำสั่ง `TRUNCATE` บน publisher จะถูก replicate ไปยัง subscriber ด้วย:

```sql
-- รันบน publisher (ตัวอย่าง — ไม่แนะนำให้รันจริงถ้าต้องการเก็บข้อมูล lab ไว้ทดสอบต่อ)
-- TRUNCATE TABLE orders;
```

แต่ถ้าสร้าง publication ด้วย `WITH (publish = 'insert, update, delete')` (ไม่มี `truncate`) คำสั่ง `TRUNCATE` บน publisher จะ**ไม่**ถูกส่งไป subscriber เลย ทำให้ข้อมูลบน subscriber ยังอยู่ครบในขณะที่ publisher ว่างเปล่า — ต้องระวังเรื่องนี้ให้ดีเพราะเป็นสาเหตุของข้อมูลไม่ตรงกันที่ตรวจพบยาก

นอกจากนี้ `TRUNCATE ... CASCADE` ก็มีพฤติกรรมพิเศษ: ถ้า publication publish ตารางที่มี foreign key เชื่อมกัน (เช่น `orders` อ้างอิงไปยังตารางอื่น) การ TRUNCATE แบบ cascade จะพยายาม replicate เฉพาะตารางที่อยู่ใน publication เท่านั้น ถ้าตารางที่ต้อง cascade ไปไม่ได้อยู่ใน publication ด้วย อาจทำให้เกิด error บนฝั่ง subscriber ได้

### 4. ข้อจำกัดอื่น ๆ ที่ควรรู้

| ข้อจำกัด | รายละเอียด |
|---|---|
| **Large Objects** | ไม่ replicate (large object API ไม่ผ่านกลไก logical decoding) |
| **Unlogged/Temporary tables** | replicate ไม่ได้ (ตาราง unlogged ไม่มี WAL ตั้งแต่ต้น) |
| **View / Materialized View** | publish ไม่ได้โดยตรง ต้อง publish ตารางฐานที่แท้จริง |
| **Partition** | รองรับได้ตั้งแต่ PG 13+ แต่ต้องเข้าใจพารามิเตอร์ `publish_via_partition_root` ให้ดี |
| **Bi-directional replication** | ทำได้ (subscriber ฝั่งหนึ่งเป็น publisher ให้อีกฝั่งด้วย) แต่ต้องจัดการ conflict และ loop ด้วยตนเอง (parameter `origin`) — PostgreSQL ไม่มี built-in multi-master conflict resolution แบบอัตโนมัติสมบูรณ์ในเวอร์ชัน 16/17 |
| **การ replicate สิทธิ์/role/tablespace** | ไม่ replicate — เป็นระดับ cluster ไม่ใช่ระดับข้อมูล |
| **Trigger บน subscriber** | ค่าเริ่มต้น trigger ที่เป็น `ENABLE` ปกติจะ**ไม่ทำงาน**ระหว่าง apply โดย apply worker (ทำงานเหมือน session role เป็น replica) ยกเว้น trigger ที่ตั้งเป็น `ENABLE REPLICA` หรือ `ENABLE ALWAYS` |

---

## Step 639: Conflict handling — เมื่อข้อมูลปลายทางขัดแย้งกัน

### Conflict เกิดขึ้นได้อย่างไร

Logical replication แบบมาตรฐาน (one-way, publisher → subscriber) โดยทั่วไปไม่ควรเกิด conflict ถ้า subscriber เป็น **read-only** สำหรับตารางที่ subscribe อยู่ (ซึ่งเป็นแนวทางที่แนะนำ) แต่ในทางปฏิบัติ conflict สามารถเกิดขึ้นได้จากสาเหตุเหล่านี้:

1. **มีคนเขียนข้อมูลตรงเข้าไปที่ตาราง subscriber โดยตรง** (ไม่ผ่าน replication) ทำให้ข้อมูลชนกับสิ่งที่ apply worker กำลังจะ apply เข้ามา
2. **ตั้งค่าแบบ bi-directional replication** (สอง node replicate หากันทั้งคู่) แล้วมีการแก้ไขข้อมูลแถวเดียวกันในเวลาใกล้เคียงกันจากทั้งสองฝั่ง
3. **initial data sync ทับซ้อนกับข้อมูลที่มีอยู่แล้ว** บน subscriber ก่อนสร้าง subscription

### ประเภทของ Conflict ที่พบบ่อย

| ประเภท Conflict | สาเหตุ |
|---|---|
| `insert_exists` (unique violation) | apply worker พยายาม INSERT แถวที่มี PK/unique key ซ้ำกับที่มีอยู่แล้วบน subscriber |
| `update_missing` | apply worker พยายาม UPDATE แถวที่หาไม่เจอบน subscriber (ถูกลบไปแล้ว หรือไม่เคย sync มา) |
| `update_origin_differs` | แถวที่จะ UPDATE ถูกแก้ไขไปแล้วจากแหล่งอื่น (พบใน bi-directional setup) |
| `delete_missing` | apply worker พยายาม DELETE แถวที่หาไม่เจอบน subscriber แล้ว |
| `multiple_unique_conflicts` | ข้อมูลชนกับ unique constraint มากกว่าหนึ่งตัวพร้อมกัน |

### จำลอง Conflict จริง: Unique Violation

มาทดสอบสถานการณ์จริงกัน — insert ข้อมูลตรงเข้าไปที่ subscriber โดยใช้ PK ที่จะชนกับข้อมูลที่ publisher กำลังจะส่งมา:

```sql
-- รันบน ecommerce_subscriber
-- แอบใส่แถวที่มี product_id = 5 ไว้ล่วงหน้า (ปกติไม่ควรทำ แต่จำลองเหตุการณ์จริง)
INSERT INTO products (product_id, product_name, unit_price)
VALUES (5, 'สินค้าที่แอบใส่เอง', 100.00);
```

จากนั้นบน publisher ก็ insert แถวใหม่ที่บังเอิญได้ `product_id = 5` เหมือนกัน (เพราะ sequence เดินมาถึงเลขนั้นพอดี):

```sql
-- รันบน ecommerce_publisher
SELECT setval('products_product_id_seq', 4, true); -- บังคับให้ next id คือ 5
INSERT INTO products (product_name, unit_price, category)
VALUES ('ลำโพงบลูทูธ', 1590.00, 'อิเล็กทรอนิกส์');
```

เมื่อ apply worker บน subscriber พยายาม apply INSERT นี้ จะเกิด conflict และ apply worker จะ **หยุดทำงาน** พร้อม log error:

```
ERROR:  duplicate key value violates unique constraint "products_pkey"
DETAIL:  Key (product_id)=(5) already exists.
CONTEXT:  processing remote data for replication origin "pg_16400" during message type "INSERT" for replication target relation "public.products" in transaction 754, finished at 0/1A3F210
```

ตรวจสอบสถานะที่ subscriber:

```sql
SELECT subname, pid FROM pg_stat_subscription;
```

```
    subname    | pid
---------------+-----
 sub_ecommerce |
(1 row)
```

สังเกตว่า `pid` เป็นค่าว่าง — apply worker หยุดทำงานไปแล้ว! และตราบใดที่ยัง error ค้างอยู่ **การ replicate ข้อมูลทั้งหมดจะหยุดชะงัก** (ไม่ใช่แค่แถวที่ conflict) เพราะ apply worker ทำงานตามลำดับ transaction (in-order) เมื่อ transaction หนึ่ง error apply worker จะพยายามใหม่ (retry) ไปเรื่อย ๆ ด้วย backoff และค้างอยู่ตรงจุดเดิมจนกว่าจะแก้ไข

ตรวจสอบ log ของ PostgreSQL เพื่อดูรายละเอียด error ซ้ำ ๆ (แสดงว่า apply worker กำลัง retry):

```bash
tail -f /var/lib/postgresql/17/main/log/postgresql-*.log
```

### วิธีแก้ไข Conflict

**วิธีที่ 1: แก้ไขข้อมูลที่ subscriber ให้ตรงกับที่ควรจะเป็น** (พบบ่อยที่สุด)

ลบหรือแก้ไขแถวที่ทำให้เกิด conflict บน subscriber ให้ apply worker สามารถ apply ต่อได้:

```sql
-- รันบน ecommerce_subscriber
-- ลบแถวที่แอบใส่เองออกไป เพื่อให้ apply worker apply แถวจริงจาก publisher ได้
DELETE FROM products WHERE product_id = 5;
```

หลังจากลบแล้ว apply worker (ที่ retry อยู่เบื้องหลัง) จะ apply สำเร็จโดยอัตโนมัติในรอบถัดไป โดยไม่ต้องสั่งอะไรเพิ่ม ตรวจสอบ:

```sql
SELECT subname, pid FROM pg_stat_subscription;
```

```
    subname    |  pid
---------------+-------
 sub_ecommerce | 54322
(1 row)
```

`pid` กลับมามีค่าแล้ว — apply worker ทำงานต่อปกติ

**วิธีที่ 2: ข้าม transaction ที่มีปัญหา (skip)**

ถ้าไม่สามารถแก้ไขข้อมูลให้ตรงกันได้ และยอมรับที่จะ "ข้าม" transaction ที่ error ไปเลย (เสี่ยงข้อมูลไม่ตรงกันถาวร) สามารถใช้ `ALTER SUBSCRIPTION ... SKIP` ได้ (ตั้งแต่ PostgreSQL 15+):

```sql
-- ต้องรู้ LSN ของ transaction ที่ error ก่อน (ดูจาก log error)
ALTER SUBSCRIPTION sub_ecommerce SKIP (lsn = '0/1A3F210');
```

คำสั่งนี้จะบอกให้ apply worker ข้าม transaction ที่ LSN นั้นไปเลย โดยไม่ apply การเปลี่ยนแปลงในนั้นทั้งหมด (ใช้ด้วยความระมัดระวังสูง เพราะข้อมูลของ transaction นั้นทั้งก้อนจะไม่ถูก apply)

### แนวทางป้องกัน Conflict ในระยะยาว

1. **ห้ามเขียนข้อมูลตรงเข้าตาราง subscriber ที่ subscribe อยู่** — ควรถือว่า subscriber เป็น read-only สำหรับตารางเหล่านั้น (ยกเว้นตั้งใจทำ bi-directional replication และมีกลยุทธ์ conflict resolution ที่ชัดเจน)
2. **ตั้ง monitoring/alert** บน `pg_stat_subscription` ให้แจ้งเตือนทันทีเมื่อ `pid` เป็น null หรือ apply worker หยุดทำงานนานผิดปกติ
3. **เก็บ log ของ PostgreSQL** และตั้ง alert เมื่อพบ error pattern ที่เกี่ยวกับ replication (`logical replication worker`, `duplicate key value`) 
4. **ใช้ `pg_stat_subscription_stats`** ตรวจสอบ `apply_error_count` เป็นระยะ ถ้าค่ามากกว่า 0 และเพิ่มขึ้นเรื่อย ๆ ต้องรีบตรวจสอบ

---

## Step 640: แบบฝึกหัดรวม — ตั้งค่า Logical Replication จริงระหว่างสอง Database ในเครื่องเดียวกัน

ในหัวข้อนี้เราจะประกอบทุกอย่างที่เรียนมาเป็น workflow เดียวที่รันได้จริงครบวงจร ตั้งแต่ศูนย์ พร้อมทดสอบ INSERT/UPDATE/DELETE แบบครบถ้วน

### ขั้นตอนที่ 1: เตรียมสภาพแวดล้อม (ทำความสะอาดของเก่าก่อนถ้ามี)

```bash
# ลบฐานข้อมูลเดิมถ้ามี (สำหรับเริ่มแบบฝึกหัดใหม่)
dropdb --if-exists ecommerce_publisher
dropdb --if-exists ecommerce_subscriber

# ตรวจสอบว่า wal_level เป็น logical แล้ว
psql -d postgres -c "SHOW wal_level;"
```

```
 wal_level
-----------
 logical
(1 row)
```

ถ้ายังไม่ใช่ `logical` ให้กลับไปทำตาม Step 633 ก่อน (ตั้งค่าแล้ว restart เซิร์ฟเวอร์)

### ขั้นตอนที่ 2: สร้างฐานข้อมูลและตารางทั้งสองฝั่ง

```bash
createdb ecommerce_publisher
createdb ecommerce_subscriber
```

```sql
-- รันบน ecommerce_publisher
\c ecommerce_publisher

CREATE TABLE products (
    product_id   SERIAL PRIMARY KEY,
    product_name VARCHAR(150),
    unit_price   NUMERIC(10,2)
);

CREATE TABLE orders (
    order_id    SERIAL PRIMARY KEY,
    customer_id INTEGER,
    order_date  TIMESTAMPTZ DEFAULT now(),
    status      VARCHAR(20)
);

INSERT INTO products (product_name, unit_price) VALUES
    ('เมาส์ไร้สาย', 350.00),
    ('คีย์บอร์ดกลไก', 1290.00),
    ('จอมอนิเตอร์ 27 นิ้ว', 6990.00);

INSERT INTO orders (customer_id, status) VALUES
    (1001, 'pending'),
    (1002, 'completed');
```

```sql
-- รันบน ecommerce_subscriber
\c ecommerce_subscriber

CREATE TABLE products (
    product_id   SERIAL PRIMARY KEY,
    product_name VARCHAR(150),
    unit_price   NUMERIC(10,2)
);

CREATE TABLE orders (
    order_id    SERIAL PRIMARY KEY,
    customer_id INTEGER,
    order_date  TIMESTAMPTZ DEFAULT now(),
    status      VARCHAR(20)
);
```

### ขั้นตอนที่ 3: สร้าง Publication บน publisher

```sql
-- รันบน ecommerce_publisher
CREATE PUBLICATION pub_ecommerce FOR TABLE products, orders;

SELECT pubname FROM pg_publication;
```

```
   pubname
---------------
 pub_ecommerce
(1 row)
```

### ขั้นตอนที่ 4: สร้าง Subscription บน subscriber

```sql
-- รันบน ecommerce_subscriber
CREATE SUBSCRIPTION sub_ecommerce
    CONNECTION 'host=localhost port=5432 dbname=ecommerce_publisher user=postgres password=yourpassword'
    PUBLICATION pub_ecommerce;
```

```
NOTICE:  created replication slot "sub_ecommerce" on publisher
CREATE SUBSCRIPTION
```

### ขั้นตอนที่ 5: ตรวจสอบ Initial Sync

```sql
-- รันบน ecommerce_subscriber
SELECT * FROM products ORDER BY product_id;
SELECT * FROM orders ORDER BY order_id;
```

```
 product_id |    product_name    | unit_price
------------+---------------------+------------
          1 | เมาส์ไร้สาย         |     350.00
          2 | คีย์บอร์ดกลไก       |    1290.00
          3 | จอมอนิเตอร์ 27 นิ้ว |    6990.00
(3 rows)

 order_id | customer_id |          order_date           |  status
----------+-------------+--------------------------------+-----------
        1 |        1001 | 2026-09-25 11:00:00.111111+00 | pending
        2 |        1002 | 2026-09-25 11:00:00.222222+00 | completed
(2 rows)
```

ข้อมูลเริ่มต้นถูก sync ครบแล้ว! ต่อไปทดสอบ real-time replication

### ขั้นตอนที่ 6: ทดสอบ INSERT

```sql
-- รันบน ecommerce_publisher
INSERT INTO products (product_name, unit_price)
VALUES ('แท่นชาร์จไร้สาย', 590.00);

INSERT INTO orders (customer_id, status)
VALUES (1003, 'pending');
```

รอสักครู่ (โดยทั่วไปน้อยกว่า 1 วินาทีสำหรับข้อมูลเล็ก ๆ ในเครื่องเดียวกัน) แล้วตรวจสอบบน subscriber:

```sql
-- รันบน ecommerce_subscriber
SELECT * FROM products ORDER BY product_id;
```

```
 product_id |    product_name     | unit_price
------------+-----------------------+------------
          1 | เมาส์ไร้สาย           |     350.00
          2 | คีย์บอร์ดกลไก         |    1290.00
          3 | จอมอนิเตอร์ 27 นิ้ว   |    6990.00
          4 | แท่นชาร์จไร้สาย       |     590.00
(4 rows)
```

```sql
SELECT * FROM orders ORDER BY order_id;
```

```
 order_id | customer_id |          order_date           |  status
----------+-------------+--------------------------------+-----------
        1 |        1001 | 2026-09-25 11:00:00.111111+00 | pending
        2 |        1002 | 2026-09-25 11:00:00.222222+00 | completed
        3 |        1003 | 2026-09-25 11:05:12.333333+00 | pending
(3 rows)
```

INSERT ถูก replicate สำเร็จ!

### ขั้นตอนที่ 7: ทดสอบ UPDATE

```sql
-- รันบน ecommerce_publisher
UPDATE orders SET status = 'shipped' WHERE order_id = 1;
UPDATE products SET unit_price = 320.00 WHERE product_id = 1;
```

ตรวจสอบบน subscriber:

```sql
-- รันบน ecommerce_subscriber
SELECT order_id, status FROM orders WHERE order_id = 1;
```

```
 order_id | status
----------+---------
        1 | shipped
(1 row)
```

```sql
SELECT product_id, product_name, unit_price FROM products WHERE product_id = 1;
```

```
 product_id | product_name | unit_price
------------+--------------+------------
          1 | เมาส์ไร้สาย  |     320.00
(1 row)
```

UPDATE ถูก replicate สำเร็จ (การที่ replicate ได้ถูกต้องเพราะทั้งสองตารางมี Primary Key ทำให้ REPLICA IDENTITY เป็น DEFAULT ใช้งานได้ตามที่อธิบายใน Step 636)

### ขั้นตอนที่ 8: ทดสอบ DELETE

```sql
-- รันบน ecommerce_publisher
DELETE FROM orders WHERE order_id = 3;
```

ตรวจสอบบน subscriber:

```sql
-- รันบน ecommerce_subscriber
SELECT * FROM orders ORDER BY order_id;
```

```
 order_id | customer_id |          order_date           |  status
----------+-------------+--------------------------------+-----------
        1 |        1001 | 2026-09-25 11:00:00.111111+00 | shipped
        2 |        1002 | 2026-09-25 11:00:00.222222+00 | completed
(2 rows)
```

DELETE ถูก replicate สำเร็จ — order_id = 3 หายไปจากทั้งสองฝั่งตรงกัน

### ขั้นตอนที่ 9: ตรวจสอบสถานะโดยรวมทั้งระบบ

```sql
-- รันบน ecommerce_publisher: ตรวจสอบ slot และ lag
SELECT slot_name, active,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS retained_wal
FROM pg_replication_slots;
```

```
   slot_name   | active | retained_wal
---------------+--------+--------------
 sub_ecommerce | t      | 0 bytes
(1 row)
```

```sql
-- รันบน ecommerce_subscriber: ตรวจสอบ apply worker
SELECT subname, pid, received_lsn, latest_end_time
FROM pg_stat_subscription;
```

```
    subname    |  pid  | received_lsn | latest_end_time
---------------+-------+---------------+--------------------------------
 sub_ecommerce | 54322 | 0/1A5D8F0     | 2026-09-25 11:10:45.456789+00
(1 row)
```

`retained_wal = 0 bytes` และมี `pid` ที่ active แปลว่าระบบ replicate ทันเวลาสมบูรณ์แบบ ไม่มี lag ค้างอยู่

### ขั้นตอนที่ 10: ทำความสะอาด (ถ้าต้องการยกเลิก lab)

```sql
-- รันบน ecommerce_subscriber
DROP SUBSCRIPTION sub_ecommerce;
```

```
NOTICE:  drop subscription "sub_ecommerce" ... dropping the replication slot "sub_ecommerce" on publisher
DROP SUBSCRIPTION
```

```sql
-- รันบน ecommerce_publisher (ถ้าจำเป็น — เผื่อ slot ไม่ถูกลบอัตโนมัติเพราะเชื่อมต่อไม่ได้)
DROP PUBLICATION pub_ecommerce;
SELECT slot_name FROM pg_replication_slots; -- ควรว่างเปล่าแล้ว
```

```bash
dropdb ecommerce_publisher
dropdb ecommerce_subscriber
```

### สคริปต์รวมสำหรับรันทั้งหมดในครั้งเดียว (bash)

สำหรับผู้ที่ต้องการรันทั้ง lab แบบอัตโนมัติ (สมมติมี `psql` alias ที่ตั้งค่า connection ไว้แล้ว และ trust auth หรือ `.pgpass` พร้อม):

```bash
#!/bin/bash
set -e

echo "== เตรียมฐานข้อมูล =="
dropdb --if-exists ecommerce_publisher
dropdb --if-exists ecommerce_subscriber
createdb ecommerce_publisher
createdb ecommerce_subscriber

echo "== สร้างตารางฝั่ง publisher =="
psql -d ecommerce_publisher <<'SQL'
CREATE TABLE products (product_id SERIAL PRIMARY KEY, product_name VARCHAR(150), unit_price NUMERIC(10,2));
CREATE TABLE orders (order_id SERIAL PRIMARY KEY, customer_id INTEGER, order_date TIMESTAMPTZ DEFAULT now(), status VARCHAR(20));
INSERT INTO products (product_name, unit_price) VALUES ('เมาส์ไร้สาย', 350.00), ('คีย์บอร์ดกลไก', 1290.00);
INSERT INTO orders (customer_id, status) VALUES (1001, 'pending');
CREATE PUBLICATION pub_ecommerce FOR TABLE products, orders;
SQL

echo "== สร้างตารางฝั่ง subscriber =="
psql -d ecommerce_subscriber <<'SQL'
CREATE TABLE products (product_id SERIAL PRIMARY KEY, product_name VARCHAR(150), unit_price NUMERIC(10,2));
CREATE TABLE orders (order_id SERIAL PRIMARY KEY, customer_id INTEGER, order_date TIMESTAMPTZ DEFAULT now(), status VARCHAR(20));
CREATE SUBSCRIPTION sub_ecommerce CONNECTION 'host=localhost port=5432 dbname=ecommerce_publisher user=postgres' PUBLICATION pub_ecommerce;
SQL

echo "== รอ initial sync 2 วินาที =="
sleep 2

echo "== ตรวจสอบผล =="
psql -d ecommerce_subscriber -c "SELECT * FROM products; SELECT * FROM orders;"
```

---

## สรุปท้ายบท

ในบทนี้เราได้เรียนรู้ Logical Replication ของ PostgreSQL อย่างครบถ้วน ตั้งแต่แนวคิดพื้นฐานไปจนถึงการปฏิบัติจริง สามารถสรุปประเด็นสำคัญได้ดังนี้:

1. **Logical Replication ต่างจาก Physical Replication** ตรงที่ส่ง "การเปลี่ยนแปลงข้อมูลระดับแถว" (ผ่าน logical decoding และ `pgoutput` plugin) แทนที่จะส่ง WAL ดิบทั้งหมด ทำให้ยืดหยุ่นกว่ามาก — เลือก replicate เฉพาะบางตารางได้ ข้ามเวอร์ชันได้ และ subscriber ยังเขียนข้อมูลอื่นได้ปกติ (read-write)

2. **Use case หลัก**: near-zero-downtime major version upgrade, selective replication, data consolidation จากหลายแหล่ง, และการส่งข้อมูลข้าม architecture

3. **การตั้งค่าเริ่มต้น** ต้องตั้ง `wal_level = logical` แล้ว restart เซิร์ฟเวอร์ ก่อนจะสร้าง `PUBLICATION` และ `SUBSCRIPTION` ได้

4. **CREATE PUBLICATION** เลือกได้ทั้ง `FOR ALL TABLES`, `FOR TABLE` (เจาะจงตาราง), และ `FOR TABLES IN SCHEMA` พร้อม column list และ row filter (PG 15+) เพื่อควบคุมข้อมูลที่จะส่งออกได้ละเอียดยิ่งขึ้น

5. **CREATE SUBSCRIPTION** สร้างบนฝั่งปลายทางพร้อม connection string ไปยัง publisher — ระบบจะทำ initial data sync ให้อัตโนมัติ แล้วเริ่ม stream การเปลี่ยนแปลงต่อเนื่อง

6. **ข้อกำหนดสำคัญ**: ตารางควรมี Primary Key เสมอ ถ้าไม่มีต้องตั้ง `REPLICA IDENTITY FULL` หรือ `USING INDEX` มิฉะนั้น UPDATE/DELETE จะ error ตั้งแต่ต้นทาง

7. **การมอนิเตอร์**: ใช้ `pg_stat_subscription`, `pg_publication_tables`, `pg_replication_slots` และ `pg_stat_subscription_stats` เป็นเครื่องมือหลักในการตรวจสอบสุขภาพของระบบ replication อย่างสม่ำเสมอ

8. **ข้อจำกัดสำคัญที่ต้องจำ**: DDL ไม่ replicate อัตโนมัติ, sequence ไม่ replicate (ต้อง sync manual ก่อน cutover), และ TRUNCATE ต้องเปิด option `truncate` ใน publication

9. **Conflict** มักเกิดจากการเขียนข้อมูลตรงเข้า subscriber โดยไม่ผ่าน replication — เมื่อเกิด conflict apply worker จะหยุดทำงานทั้งหมด (ไม่ใช่แค่ transaction ที่มีปัญหา) ต้องแก้ไขข้อมูลให้ตรงกันหรือใช้ `ALTER SUBSCRIPTION ... SKIP` เพื่อข้าม

10. เราได้ฝึกปฏิบัติจริงตั้งค่า logical replication ระหว่างสองฐานข้อมูลบนเครื่องเดียวกัน (`ecommerce_publisher` → `ecommerce_subscriber`) และทดสอบ INSERT/UPDATE/DELETE ครบทุกกรณี พร้อมตรวจสอบผลลัพธ์ในแต่ละขั้นตอน

บทถัดไปจะพาไปสู่หัวข้อ **High Availability** ซึ่งเป็นการนำ Physical Replication และเทคนิคขั้นสูงมาผสมผสานกันเพื่อสร้างระบบที่ทนทานต่อความล้มเหลว (fault-tolerant) ในระดับ production จริง

**บทถัดไป**: [Part 065 — High Availability](./part-065-high-availability.md)

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1

จงอธิบายว่าทำไม Logical Replication จึงสามารถ replicate ข้อมูลข้าม PostgreSQL major version ที่ต่างกันได้ (เช่น จาก PostgreSQL 14 ไปยัง PostgreSQL 17) ในขณะที่ Physical Replication ทำไม่ได้

<details>
<summary>เฉลย</summary>

Physical Replication ทำงานโดยส่ง WAL แบบ byte-for-byte ซึ่งมีโครงสร้างที่ผูกกับ on-disk format ภายในของแต่ละ major version (page format, tuple header, ฯลฯ) ที่อาจเปลี่ยนแปลงระหว่างเวอร์ชัน ทำให้ standby ต้องเป็น PostgreSQL เวอร์ชันเดียวกันเป๊ะกับ primary เท่านั้น

ในทางกลับกัน Logical Replication ไม่ได้ส่ง WAL ดิบ แต่ใช้กระบวนการ **logical decoding** แปลง WAL ให้กลายเป็น "การเปลี่ยนแปลงระดับแถว" ที่มีความหมายอิสระจากรูปแบบไฟล์ภายใน (เช่น "INSERT ค่า X, Y, Z เข้าตาราง products") แล้วฝั่ง subscriber จะ apply เป็นคำสั่ง SQL ปกติ (เหมือนรันคำสั่ง INSERT/UPDATE/DELETE เอง) ซึ่งไม่ขึ้นกับ on-disk format ของเวอร์ชันใดเวอร์ชันหนึ่ง ทำให้ publisher และ subscriber เป็นคนละ major version กันได้ตราบใดที่โครงสร้างตาราง (schema) เข้ากันได้

</details>

### แบบฝึกหัดที่ 2

จงเขียนคำสั่งเปลี่ยนค่า `wal_level` เป็น `logical` ผ่าน SQL (ไม่ใช่แก้ไฟล์ `postgresql.conf` โดยตรง) พร้อมระบุว่าหลังรันคำสั่งแล้วต้องทำอะไรต่อ

<details>
<summary>เฉลย</summary>

```sql
ALTER SYSTEM SET wal_level = 'logical';
```

คำสั่งนี้จะเขียนค่าไปที่ไฟล์ `postgresql.auto.conf` แต่เนื่องจาก `wal_level` เป็นพารามิเตอร์ประเภท `postmaster` (ต้องใช้ตอนเริ่ม process หลักเท่านั้น) การ `reload` ไม่เพียงพอ — จำเป็นต้อง **restart เซิร์ฟเวอร์ทั้งหมด** เช่น

```bash
sudo systemctl restart postgresql
```

หรือ

```bash
pg_ctl restart -D /path/to/data
```

หลัง restart แล้วตรวจสอบด้วย `SHOW wal_level;` ว่าค่าถูกต้องเป็น `logical`

</details>

### แบบฝึกหัดที่ 3

กำหนดให้มีตาราง `order_items` ที่ต้องการ publish เฉพาะคอลัมน์ `order_id`, `product_id`, `quantity` (ไม่ต้องการส่งคอลัมน์ `internal_notes` ที่มีข้อมูลอ่อนไหวออกไป) จงเขียนคำสั่ง `CREATE PUBLICATION` ที่เหมาะสม

<details>
<summary>เฉลย</summary>

```sql
CREATE PUBLICATION pub_order_items_safe
    FOR TABLE order_items (order_id, product_id, quantity);
```

นี่คือการใช้ **column list** ฟีเจอร์ที่เพิ่มเข้ามาตั้งแต่ PostgreSQL 15 ซึ่งช่วยให้เลือก publish เฉพาะบางคอลัมน์ได้ โดยคอลัมน์ที่ไม่ได้ระบุ (`internal_notes`) จะไม่ถูกส่งไปยัง subscriber เลย ข้อควรระวังคือคอลัมน์ที่เป็น Primary Key จำเป็นต้องรวมอยู่ใน column list เสมอ (เพื่อให้ระบุแถวสำหรับ UPDATE/DELETE ได้)

</details>

### แบบฝึกหัดที่ 4

ตารางชื่อ `session_logs` ไม่มี Primary Key และไม่มี unique index ใด ๆ เลย แต่ทีมงานต้องการ publish ตารางนี้ผ่าน logical replication รวมถึงต้องการให้ UPDATE/DELETE ทำงานได้ปกติ จะต้องทำอย่างไร และมีข้อควรระวังอะไรบ้าง

<details>
<summary>เฉลย</summary>

ต้องตั้งค่า `REPLICA IDENTITY FULL` ให้ตารางนั้น:

```sql
ALTER TABLE session_logs REPLICA IDENTITY FULL;
```

การตั้งค่านี้ทำให้ PostgreSQL บันทึกค่า**ทุกคอลัมน์**ของแถวเดิมลง WAL เมื่อเกิด UPDATE/DELETE เพื่อให้ subscriber ใช้ระบุแถวเป้าหมายได้ (แทนที่จะใช้ PK)

ข้อควรระวัง:
1. **ประสิทธิภาพต่ำ** — ขนาด WAL จะโตขึ้นมากเพราะบันทึกทุกคอลัมน์ ไม่ใช่แค่ PK
2. **การ apply บน subscriber ช้าลง** — เพราะต้องหาแถวที่ตรงกัน "ทุกคอลัมน์" ซึ่งอาจไม่มี index รองรับ ทำให้เกิด sequential scan
3. **ทางออกที่ดีกว่า** ถ้าเป็นไปได้ คือเพิ่ม Primary Key หรือสร้าง unique index ให้ตารางแล้วใช้ `ALTER TABLE ... REPLICA IDENTITY USING INDEX` แทน ซึ่งให้ประสิทธิภาพดีกว่า `FULL` มาก

</details>

### แบบฝึกหัดที่ 5

หลังจากสร้าง subscription และข้อมูลถูก sync เรียบร้อยแล้ว ทีมพัฒนาได้รัน `ALTER TABLE products ADD COLUMN discount_percent NUMERIC(5,2) DEFAULT 0;` บน publisher เท่านั้น (ลืมรันบน subscriber) จะเกิดอะไรขึ้น และควรแก้ไขอย่างไร

<details>
<summary>เฉลย</summary>

เนื่องจาก DDL ไม่ถูก replicate อัตโนมัติในระบบ logical replication คอลัมน์ `discount_percent` จะปรากฏเฉพาะบน publisher เท่านั้น ไม่ปรากฏบน subscriber

ผลกระทบ: เมื่อมีการ INSERT/UPDATE แถวใหม่บน publisher ที่มีค่าในคอลัมน์นี้ apply worker จะพยายาม apply ข้อมูลมายัง subscriber แต่เนื่องจากเป็นคอลัมน์เพิ่มเติมที่มี `DEFAULT` (ไม่ใช่ NOT NULL ที่ไม่มี default) logical replication ของ PostgreSQL จะ map เฉพาะคอลัมน์ที่ subscriber มีอยู่จริง ทำให้ยังคง apply ได้โดยไม่ error (ข้อมูลคอลัมน์ที่ subscriber ไม่มีจะถูกละทิ้งไปเงียบ ๆ)

วิธีแก้ไข: รันคำสั่ง DDL เดียวกันบน subscriber ทันทีที่ทำได้

```sql
-- รันบน ecommerce_subscriber
ALTER TABLE products ADD COLUMN discount_percent NUMERIC(5,2) DEFAULT 0;
```

แนวปฏิบัติที่ดีคือกำหนดขั้นตอน deploy ที่บังคับให้รัน DDL บนทั้งสองฝั่งพร้อมกันหรือ subscriber ก่อนเสมอ เพื่อป้องกันข้อมูลสูญหายแบบเงียบ ๆ (silent data loss) ในลักษณะนี้

</details>

### แบบฝึกหัดที่ 6

จงเขียนคำสั่ง SQL เพื่อตรวจสอบว่า replication slot ชื่อ `sub_ecommerce` บน publisher มี WAL ค้างส่ง (retained WAL) อยู่เท่าไหร่ และอธิบายว่าค่านี้มีความหมายอย่างไรถ้ามันเพิ่มขึ้นเรื่อย ๆ ไม่หยุด

<details>
<summary>เฉลย</summary>

```sql
SELECT slot_name, active,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS retained_wal
FROM pg_replication_slots
WHERE slot_name = 'sub_ecommerce';
```

ค่า `retained_wal` คือปริมาณ WAL ที่ replication slot กำลังกันไว้ไม่ให้ระบบลบทิ้ง เพราะยังไม่ถูกส่งไปให้ subscriber ครบ ถ้าค่านี้เพิ่มขึ้นเรื่อย ๆ ไม่หยุด แปลว่า:

1. subscriber อาจไม่ได้เชื่อมต่ออยู่ (ปิดเครื่อง, network มีปัญหา) หรือ apply worker หยุดทำงานเพราะเกิด error/conflict
2. ถ้าปล่อยไว้นานโดยไม่แก้ไข **ดิสก์บน publisher อาจเต็ม** เพราะ WAL จะถูกสะสมไว้เรื่อย ๆ ไม่ถูกลบ ซึ่งอาจทำให้เซิร์ฟเวอร์ publisher ทั้งตัวหยุดทำงาน (ไม่ใช่แค่ replication พัง)
3. ควรตั้ง monitoring/alert ให้แจ้งเตือนเมื่อ `retained_wal` เกินเกณฑ์ที่กำหนด และตรวจสอบสถานะ subscriber (`pg_stat_subscription`) ทันทีที่พบความผิดปกติ

</details>

### แบบฝึกหัดที่ 7

Apply worker บน subscriber หยุดทำงานเพราะเกิด unique constraint violation จากแถวที่ถูกใส่เข้าไปตรง ๆ บน subscriber โดยไม่ผ่าน replication จงอธิบายขั้นตอนการวินิจฉัยและแก้ไขปัญหานี้

<details>
<summary>เฉลย</summary>

ขั้นตอนวินิจฉัยและแก้ไข:

1. **ตรวจสอบสถานะ apply worker** ด้วย `SELECT subname, pid FROM pg_stat_subscription;` — ถ้า `pid` เป็นค่าว่าง แสดงว่า worker หยุดทำงาน
2. **ดู log ของ PostgreSQL** เพื่อหารายละเอียด error เช่น `duplicate key value violates unique constraint "products_pkey" DETAIL: Key (product_id)=(5) already exists.` ซึ่งจะบอกทั้งชื่อ constraint และค่าที่ชนกัน
3. **ระบุแถวที่เป็นปัญหา** บน subscriber โดยใช้ค่า key จาก error message เช่น `SELECT * FROM products WHERE product_id = 5;`
4. **ตัดสินใจวิธีแก้**: 
   - ถ้าแถวที่มีอยู่บน subscriber เป็นข้อมูลที่ผิด (ไม่ควรมีอยู่) → `DELETE` แถวนั้นออก เพื่อให้ apply worker apply แถวจริงจาก publisher ได้สำเร็จในรอบ retry ถัดไป
   - ถ้าต้องการข้าม transaction ที่มีปัญหาไปเลย → ใช้ `ALTER SUBSCRIPTION sub_ecommerce SKIP (lsn = '<LSN จาก log>');`
5. **ตรวจสอบผลหลังแก้ไข** ด้วย `SELECT subname, pid FROM pg_stat_subscription;` อีกครั้ง — ถ้า `pid` กลับมามีค่า แสดงว่า apply worker ทำงานต่อได้ปกติแล้ว
6. **ป้องกันไม่ให้เกิดซ้ำ**: กำหนดนโยบายห้ามเขียนข้อมูลตรงเข้าตารางที่เป็น subscriber ของ logical replication โดยไม่ผ่านกระบวนการที่ควบคุมได้

</details>

### แบบฝึกหัดที่ 8

เปรียบเทียบว่า `CREATE PUBLICATION pub_x FOR ALL TABLES;` กับ `CREATE PUBLICATION pub_y FOR TABLE products, orders;` ต่างกันอย่างไร และควรเลือกใช้แบบไหนในสถานการณ์ "อัปเกรด PostgreSQL จากเวอร์ชัน 15 ไป 17 สำหรับทั้งฐานข้อมูล" เทียบกับสถานการณ์ "ส่งข้อมูลเฉพาะตาราง products ไปยัง data warehouse"

<details>
<summary>เฉลย</summary>

**ความแตกต่าง**:
- `FOR ALL TABLES` publish ทุกตารางในฐานข้อมูลโดยอัตโนมัติ รวมถึงตารางที่จะถูกสร้างขึ้นใหม่ในอนาคตด้วย (แบบ dynamic ไม่ต้องแก้ publication เพิ่ม) แต่ต้องใช้สิทธิ์สูง (superuser หรือมี attribute พิเศษ)
- `FOR TABLE` publish เฉพาะตารางที่ระบุไว้อย่างชัดเจนเท่านั้น ตารางใหม่ที่สร้างขึ้นภายหลังจะไม่ถูกรวมเข้ามาอัตโนมัติ ต้องเพิ่มด้วย `ALTER PUBLICATION ... ADD TABLE` เอง

**การเลือกใช้ในแต่ละสถานการณ์**:
1. **อัปเกรดทั้งฐานข้อมูลจาก 15 → 17**: ควรใช้ `FOR ALL TABLES` เพราะต้องการ replicate ทุกตารางในฐานข้อมูลไปยังเซิร์ฟเวอร์ใหม่ครบถ้วน ไม่ต้องมาคอยจำว่ามีตารางไหนตกหล่นไปบ้าง
2. **ส่งเฉพาะตาราง products ไป data warehouse**: ควรใช้ `FOR TABLE products` (หรือระบุเฉพาะตารางที่ต้องการ) เพราะต้องการควบคุมขอบเขตข้อมูลที่ส่งออกอย่างชัดเจน ไม่ต้องการให้ตารางอื่น ๆ (ที่อาจมีข้อมูลอ่อนไหวหรือไม่เกี่ยวข้อง) ถูกส่งไปด้วยโดยไม่ตั้งใจ

</details>

### แบบฝึกหัดที่ 9

จงอธิบายว่าทำไมค่า sequence (เช่น `products_product_id_seq`) จึงไม่ถูก replicate ผ่าน logical replication และอธิบายผลกระทบที่อาจเกิดขึ้นถ้าไม่จัดการเรื่องนี้ก่อนทำ cutover (สลับให้แอปพลิเคชันเขียนที่ subscriber แทน publisher)

<details>
<summary>เฉลย</summary>

Sequence ไม่ถูก replicate เพราะการเปลี่ยนแปลงค่าของ sequence (เช่นการเรียก `nextval()`) ไม่ได้ถูกมองว่าเป็น "การเปลี่ยนแปลงข้อมูลระดับแถวของตาราง" ที่ logical decoding จะจับมาส่งต่อ — มันเป็นกลไกภายในของ sequence object ที่แยกออกจากกลไก logical replication โดยสิ้นเชิง

ผลกระทบถ้าไม่จัดการก่อน cutover: หลังจากสลับให้แอปพลิเคชันเขียนที่ subscriber ค่า sequence บน subscriber (ซึ่งอาจยังค้างอยู่ที่ค่าต่ำ เช่น 1) จะเริ่ม generate PK ใหม่จากค่านั้น ในขณะที่ subscriber มีข้อมูลที่ replicate มาแล้วซึ่งมี PK สูงกว่ามาก (เช่นถึง 1000) ทำให้เมื่อ insert แถวใหม่ ค่า PK ที่ generate จะไปชนกับ PK ที่มีอยู่แล้ว เกิด **unique constraint violation** ทันที

วิธีป้องกัน: ก่อน cutover ต้อง query ค่า `last_value` ของ sequence บน publisher แล้วใช้ `setval()` ตั้งค่าเดียวกัน (หรือสูงกว่าเล็กน้อยเพื่อความปลอดภัย) บน subscriber ทุกตัวที่เกี่ยวข้อง ก่อนจะเปิดให้แอปพลิเคชันเขียนข้อมูลจริง

</details>

### แบบฝึกหัดที่ 10

จงเขียนขั้นตอนแบบ end-to-end (เป็นคำสั่ง SQL/bash) เพื่อตั้งค่า logical replication ตาราง `orders` เพียงตารางเดียว (ไม่รวม `products`) จากฐานข้อมูล `ecommerce_publisher` ไปยัง `ecommerce_subscriber` โดยให้ publish เฉพาะ order ที่มี `status = 'completed'` เท่านั้น (ใช้ row filter)

<details>
<summary>เฉลย</summary>

```sql
-- ขั้นตอนที่ 1: รันบน ecommerce_publisher — สร้าง publication พร้อม row filter
CREATE PUBLICATION pub_orders_completed_only
    FOR TABLE orders WHERE (status = 'completed');
```

```sql
-- ขั้นตอนที่ 2: ตรวจสอบว่า publication ครอบคลุมตารางและเงื่อนไขถูกต้อง
SELECT pubname, tablename, rowfilter
FROM pg_publication_tables
WHERE pubname = 'pub_orders_completed_only';
```

ผลลัพธ์ที่คาดหวัง:

```
         pubname           | tablename |       rowfilter
----------------------------+-----------+------------------------
 pub_orders_completed_only  | orders    | (status = 'completed')
(1 row)
```

```sql
-- ขั้นตอนที่ 3: รันบน ecommerce_subscriber (ต้องมีตาราง orders สร้างไว้ล่วงหน้าแล้ว)
CREATE SUBSCRIPTION sub_orders_completed
    CONNECTION 'host=localhost port=5432 dbname=ecommerce_publisher user=postgres password=yourpassword'
    PUBLICATION pub_orders_completed_only;
```

```sql
-- ขั้นตอนที่ 4: ตรวจสอบผล — เฉพาะ order ที่ status = 'completed' เท่านั้นที่ควรปรากฏบน subscriber
SELECT * FROM orders;
```

ข้อสังเกตสำคัญ: row filter จะทำงานเฉพาะตอน apply การเปลี่ยนแปลง (และตอน initial sync) เท่านั้น — ถ้าภายหลัง order บน publisher ถูก `UPDATE` จาก `status = 'pending'` เป็น `status = 'completed'` แถวนั้นจะถูกส่งเป็น INSERT ใหม่ไปยัง subscriber (เพราะเพิ่งเข้าเงื่อนไข filter) และในทางกลับกันถ้า `UPDATE` จาก `completed` เป็น `pending` แถวนั้นจะถูกส่งเป็น DELETE ไปยัง subscriber แทน (เพราะไม่เข้าเงื่อนไข filter อีกต่อไป)

</details>

---

**บทถัดไป**: [Part 065 — High Availability](./part-065-high-availability.md)
