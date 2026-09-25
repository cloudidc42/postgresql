# Storage Engine: Heap, Page Layout, TOAST

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับโลก/ผู้เชี่ยวชาญ | Part 082

---

## เป้าหมายการเรียนรู้

เมื่อจบบทนี้ ผู้เรียนจะสามารถ:

1. อธิบายได้ว่า PostgreSQL จัดเก็บข้อมูลแบบ **heap** ต่างจาก index-organized table (เช่น MySQL InnoDB clustered index) อย่างไร และมีผลต่อ performance อย่างไร
2. อ่านและเข้าใจโครงสร้าง **data directory** ของ PostgreSQL ในระดับไฟล์จริง เชื่อมโยง `pg_class.relfilenode` เข้ากับไฟล์บนดิสก์
3. เข้าใจว่า **page (block)** คืออะไร ขนาด 8KB มาจากไหน และเหตุใดทุกอย่างใน PostgreSQL ถึงถูกจัดเก็บเป็น page
4. อ่าน **page layout** ได้ในระดับ byte: Page Header, Item Pointers (Line Pointers), และ Tuple Data ที่เติบโตจากคนละทิศทางของ page
5. อธิบาย **tuple header** และฟิลด์สำคัญ (`xmin`, `xmax`, `ctid`, `infomask`) ที่เป็นรากฐานของระบบ MVCC
6. ใช้ extension **pageinspect** เพื่อ "ผ่าดู" page และ tuple จริงในระดับ byte เพื่อ debug และเรียนรู้เชิงลึก
7. อธิบายว่า **TOAST** (The Oversized-Attribute Storage Technique) คืออะไร ทำไม PostgreSQL ต้องมีกลไกนี้
8. เลือกใช้ TOAST strategy ที่เหมาะสม (`PLAIN`, `EXTENDED`, `EXTERNAL`, `MAIN`) สำหรับแต่ละคอลัมน์
9. เข้าใจโครงสร้างของ **TOAST table** ที่ PostgreSQL สร้างอัตโนมัติใน schema `pg_toast`
10. ใช้เครื่องมือ diagnostic จริงสำรวจโครงสร้างตาราง สังเกตพฤติกรรม TOAST กับข้อมูลขนาดใหญ่ได้ด้วยตนเอง

บทนี้เป็นการ "ผ่าตัด" PostgreSQL ลงไปถึงระดับไฟล์และไบต์ — เนื้อหาที่แม้แต่ DBA ระดับมืออาชีพจำนวนมากก็ไม่เคยลงลึกขนาดนี้ แต่เมื่อเข้าใจแล้ว จะทำให้การ tune performance, การ debug bloat, และการออกแบบ schema ของคุณ "แม่นยำ" ขึ้นอย่างก้าวกระโดด

---

## เตรียมข้อมูล

เราจะใช้ตาราง `products` แบบง่าย ๆ เป็นตัวอย่างหลักตลอดทั้งบท เพื่อสาธิตทั้งเรื่อง heap, page layout และ TOAST:

```sql
CREATE TABLE products (
    product_id   SERIAL PRIMARY KEY,
    product_name VARCHAR(150),
    description  TEXT,
    unit_price   NUMERIC(10,2)
);

-- ใส่ข้อมูลตัวอย่างจำนวนหนึ่งเพื่อให้มีหลาย page ให้สำรวจ
INSERT INTO products (product_name, description, unit_price)
SELECT
    'Product ' || g,
    'A standard description for product number ' || g,
    (random() * 1000)::numeric(10,2)
FROM generate_series(1, 500) AS g;

ANALYZE products;
```

เราจะกลับมาใช้ตารางนี้ซ้ำ ๆ ในแต่ละ Step โดยจะ query โครงสร้างจริงของมันด้วยเครื่องมือ system catalog และ extension ต่าง ๆ

> หมายเหตุ: ผลลัพธ์ตัวเลข (page count, byte offset, ctid ฯลฯ) ในตัวอย่างนี้เป็น **ค่าโดยประมาณเพื่อการสาธิต** ค่าจริงบนเครื่องของคุณอาจต่างกันเล็กน้อยขึ้นกับ PostgreSQL version, alignment, และ autovacuum ที่เคยทำงานมาก่อน — สิ่งสำคัญคือเข้าใจ "รูปแบบ" ของโครงสร้างข้อมูล ไม่ใช่ตัวเลขเป๊ะ ๆ

---

## Step 811: Heap File Organization

### Heap คืออะไร

PostgreSQL เก็บข้อมูลตารางแบบ **heap** — หมายความว่าแถวข้อมูล (tuple) ถูกเก็บ **ไม่เรียงลำดับ** ตามคีย์ใด ๆ เลย ตำแหน่งของแถวขึ้นอยู่กับว่ามันถูก insert เมื่อไหร่ และมี free space ว่างอยู่ตรงไหนในไฟล์ขณะนั้น

นี่คือความแตกต่างพื้นฐานที่สุดอย่างหนึ่งระหว่าง PostgreSQL กับ MySQL (InnoDB engine):

```
┌─────────────────────────────────────────────────────────────────┐
│  PostgreSQL: Heap Table (Heap-Organized)                        │
│                                                                   │
│  Page 0        Page 1        Page 2        Page 3                │
│  ┌────────┐   ┌────────┐   ┌────────┐   ┌────────┐              │
│  │ id=7   │   │ id=1   │   │ id=15  │   │ id=3   │  ← ลำดับ      │
│  │ id=2    │   │ id=99  │   │ id=4   │   │ id=200 │    ไม่สัมพันธ์│
│  │ id=50  │   │ id=12  │   │ id=88  │   │ id=6   │    กับ PK เลย │
│  └────────┘   └────────┘   └────────┘   └────────┘              │
│                                                                   │
│  ตำแหน่งจริงของแถวขึ้นกับ insert order + free space เท่านั้น    │
│  PRIMARY KEY เป็นแค่ constraint + index แยกต่างหาก               │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  MySQL InnoDB: Clustered Index (Index-Organized Table)          │
│                                                                   │
│         B-Tree เรียงตาม PRIMARY KEY                              │
│              ┌─────────┐                                        │
│              │ id: 50  │                                        │
│              └────┬────┘                                        │
│         ┌─────────┴─────────┐                                   │
│    ┌────▼────┐         ┌────▼────┐                              │
│    │ id 1-49 │         │id 51-99 │  ← ข้อมูลทั้งแถวอยู่ใน       │
│    │(ทั้งแถว) │         │(ทั้งแถว) │    leaf node ของ PK B-Tree  │
│    └─────────┘         └─────────┘    เรียงตามลำดับ PK เสมอ     │
└─────────────────────────────────────────────────────────────────┘
```

### ผลกระทบเชิง performance

| ประเด็น | PostgreSQL (Heap) | MySQL InnoDB (Clustered) |
|---|---|---|
| Secondary index lookup | Index → heap pointer (`ctid`) → heap page (อาจต้องอ่าน 2 page) | Index → PK value → clustered lookup (คล้ายกัน แต่ PK lookup เร็วกว่าถ้า query ใช้ PK) |
| Range scan บน PK | ไม่รับประกันว่าข้อมูลอยู่ติดกันบนดิสก์ | ข้อมูลเรียงติดกันจริงตาม PK เสมอ |
| Insert แบบสุ่ม | เร็ว เพราะแทรกที่ไหนก็ได้ที่มี free space | อาจเกิด page split ถ้า PK ไม่ใช่ sequential |
| ต้องการให้ physical order ตรงกับ index | ต้องรัน `CLUSTER` ด้วยมือ (ไม่ maintain อัตโนมัติ) | maintain อัตโนมัติตลอดเวลา |

### พิสูจน์ด้วยตัวเอง: ctid ไม่เรียงตาม product_id

`ctid` คือ physical location ของแถว ในรูปแบบ `(page_number, item_index)` ลองดูว่าหลังจาก update/delete บางแถว ตำแหน่งจริงจะกระจัดกระจายทันที:

```sql
-- ดูตำแหน่งจริงของ 10 แถวแรกตาม product_id
SELECT ctid, product_id, product_name
FROM products
WHERE product_id BETWEEN 1 AND 10
ORDER BY product_id;
```

```
  ctid   | product_id | product_name
---------+------------+---------------
 (0,1)   |          1 | Product 1
 (0,2)   |          2 | Product 2
 (0,3)   |          3 | Product 3
 (0,4)   |          4 | Product 4
 ...
 (0,10)  |         10 | Product 10
```

ตอนเพิ่งสร้างตาราง ctid มักจะเรียงเป็นระเบียบ (เพราะ insert เข้าตามลำดับ) แต่ทันทีที่มี update:

```sql
-- UPDATE จะสร้างแถวใหม่ (เดี๋ยวจะอธิบายใน Step 815 เรื่อง MVCC)
UPDATE products SET unit_price = unit_price + 1 WHERE product_id = 5;

SELECT ctid, product_id FROM products WHERE product_id = 5;
```

```
  ctid   | product_id
---------+------------
 (7,23)  |          5   -- ← แถวใหม่ถูกวางไว้ที่ page 7 ไม่ใช่ page 0 อีกต่อไป!
```

```sql
-- เปรียบเทียบ physical order (ctid) กับ logical order (product_id)
SELECT ctid, product_id
FROM products
ORDER BY ctid
LIMIT 15;
```

```
  ctid   | product_id
---------+------------
 (0,1)   |          1
 (0,2)   |          2
 (0,3)   |          3
 (0,4)   |          4
 (0,6)   |          6    -- ← สังเกตว่า id=5 หายไปจาก page 0 แล้ว!
 (0,7)   |          7
 ...
```

จะเห็นว่า `product_id = 5` ไม่อยู่ที่ `(0,5)` อีกต่อไป เพราะ `UPDATE` สร้าง tuple version ใหม่ในตำแหน่งอื่น (heap-only tuple หรือ page ใหม่ ขึ้นกับ free space) — นี่คือธรรมชาติของ heap storage ที่ต่างจาก clustered index โดยสิ้นเชิง

> 💡 หากต้องการให้ตารางถูกจัดเรียงใหม่ตามลำดับของ index ใดๆ ใช้คำสั่ง `CLUSTER products USING products_pkey;` — แต่นี่เป็นการจัดเรียง **ครั้งเดียว ณ เวลานั้น** ไม่ใช่การ maintain อัตโนมัติ แถวใหม่ที่ insert ต่อจากนี้ก็ยังคงกระจัดกระจายตามปกติ

---

## Step 812: Data Directory Structure — เจาะลึก

ใน Part 004 และ Part 081 เราเคยแนะนำโครงสร้าง data directory ของ PostgreSQL คร่าว ๆ มาแล้ว ใน Step นี้เราจะเจาะลึกลงไปถึงระดับที่เชื่อมโยงกับ physical page ที่เราจะพูดถึงใน Step ถัดไป

### ภาพรวมโครงสร้าง PGDATA

```
$PGDATA/
├── base/                    ← ข้อมูลของทุก database (แยกเป็นโฟลเดอร์ตาม OID)
│   ├── 1/                   ← database "template1"
│   ├── 13394/               ← database "postgres"
│   └── 16401/               ← database "mycourse" (ตัวอย่าง)
│       ├── 16404            ← ไฟล์ heap ของตาราง products (ชื่อ = relfilenode)
│       ├── 16404_fsm        ← Free Space Map ของ products
│       ├── 16404_vm         ← Visibility Map ของ products
│       ├── 16407            ← ไฟล์ของ index products_pkey
│       └── ...
├── global/                  ← ข้อมูลระดับ cluster (ไม่ผูกกับ database เดียว)
│   ├── pg_control            ← control file: checkpoint info, WAL position
│   └── 1262, 1213, ...       ← catalog ที่ share ทุก database เช่น pg_database
├── pg_wal/                  ← Write-Ahead Log segments (เดี๋ยวลงลึกใน Part 083)
│   ├── 000000010000000000000001
│   └── ...
├── pg_xact/                 ← Transaction commit status (เดิมชื่อ pg_clog)
├── pg_multixact/            ← สถานะของ multixact (shared row lock)
├── pg_tblspc/               ← symlink ไปยัง tablespace อื่น ๆ (ถ้ามี)
├── pg_stat/                 ← สถิติถาวร (permanent stats, PG15+)
├── postgresql.conf
├── pg_hba.conf
└── PG_VERSION
```

### เชื่อมโยง table → file บนดิสก์จริง

หัวใจของเรื่องนี้คือ **`pg_class.relfilenode`** ซึ่งเป็นชื่อไฟล์จริงบนดิสก์ (ปกติจะเท่ากับ OID ของตาราง แต่จะเปลี่ยนไปหลัง `TRUNCATE`, `CLUSTER`, หรือ `VACUUM FULL`)

```sql
SELECT
    c.relname,
    c.relfilenode,
    c.relkind,
    pg_relation_filepath(c.oid) AS filepath,
    pg_size_pretty(pg_relation_size(c.oid)) AS size
FROM pg_class c
WHERE c.relname IN ('products', 'products_pkey');
```

```
    relname     | relfilenode | relkind |         filepath          |  size
-----------------+-------------+---------+----------------------------+--------
 products        |       16404 | r       | base/16401/16404          | 88 kB
 products_pkey   |       16407 | i       | base/16401/16407          | 40 kB
```

ลองยืนยันด้วยการดูไฟล์จริงบนดิสก์ (ต้องรันบน server ที่มีสิทธิ์เข้าถึง filesystem):

```sql
-- หา data_directory ปัจจุบันก่อน
SHOW data_directory;
```

```bash
# base/<dboid>/<relfilenode> ต้องตรงกับผลลัพธ์ query ด้านบน
ls -la $PGDATA/base/16401/16404*
```

```
-rw------- 1 postgres postgres 90112 Sep 25 10:00 16404       -- main data file (heap)
-rw------- 1 postgres postgres 24576 Sep 25 10:00 16404_fsm   -- Free Space Map fork
-rw------- 1 postgres postgres  8192 Sep 25 10:00 16404_vm    -- Visibility Map fork
```

### Relation Forks

หนึ่งตารางไม่ได้มีแค่ไฟล์เดียว แต่มี "fork" ที่แยกจากกันตามหน้าที่:

| Fork | Suffix | หน้าที่ |
|---|---|---|
| **Main** | (ไม่มี suffix) | เก็บข้อมูลจริงของตาราง (heap tuples) |
| **Free Space Map (FSM)** | `_fsm` | เก็บข้อมูลว่าแต่ละ page มี free space เหลือเท่าไหร่ ใช้เร่ง insert |
| **Visibility Map (VM)** | `_vm` | บอกว่า page ไหน "all visible" / "all frozen" แล้ว ใช้เร่ง vacuum และ index-only scan |
| **Init fork** | `_init` | ใช้กับ unlogged table เพื่อ reset หลัง crash |

```sql
-- ดูรายละเอียด fork ผ่าน pg_relation_size แยกตาม fork
SELECT
    pg_relation_size('products', 'main')  AS main_bytes,
    pg_relation_size('products', 'fsm')   AS fsm_bytes,
    pg_relation_size('products', 'vm')    AS vm_bytes;
```

```
 main_bytes | fsm_bytes | vm_bytes
------------+-----------+----------
      90112 |     24576 |     8192
```

### Segmentation: ไฟล์ 1GB ต่อ segment

ไฟล์ relation แต่ละ fork จะถูกตัดเป็นชิ้นละ **1 GB** เพื่อหลีกเลี่ยงข้อจำกัดของ filesystem บางประเภท (เช่น FAT32 เดิม หรือ ext ที่มีปัญหาไฟล์ใหญ่มาก):

```
base/16401/16404          ← segment 0 (0 - 1GB)
base/16401/16404.1        ← segment 1 (1GB - 2GB)
base/16401/16404.2        ← segment 2 (2GB - 3GB)
```

```sql
-- สำหรับตารางใหญ่จริง ๆ จะเห็นหลาย segment
-- (products ของเรายังเล็กมาก จึงมีแค่ segment เดียว)
```

การเชื่อมโยงนี้สำคัญมากเวลา troubleshoot ปัญหาระดับ disk I/O, การทำ physical backup, หรือแม้แต่การกู้ข้อมูลกรณีฉุกเฉิน — เพราะสุดท้ายแล้ว "ตาราง" ในมุมมอง SQL ก็คือกลุ่มไฟล์เหล่านี้บน filesystem นั่นเอง

---

## Step 813: Page (Block) คืออะไร

### ทุกอย่างคือ page

PostgreSQL จัดการพื้นที่จัดเก็บทั้งหมด — ไม่ว่าจะเป็น heap table, B-Tree index, หรือแม้แต่ WAL — เป็นหน่วยคงที่ที่เรียกว่า **page** หรือ **block** ขนาดมาตรฐานคือ **8192 bytes (8KB)**

```sql
SHOW block_size;
```

```
 block_size
------------
 8192
```

ค่านี้ถูกกำหนดตอน **compile PostgreSQL** (ค่า `BLCKSZ` ใน source code) ไม่สามารถเปลี่ยนได้ด้วย `ALTER SYSTEM` หรือ config file — ต้อง compile PostgreSQL จาก source ใหม่ด้วย `./configure --with-blocksize=N` (N เป็นเลขยกกำลัง 2 ตั้งแต่ 1KB ถึง 32KB) ซึ่งแทบไม่มีใครทำ เพราะ binary package ที่แจกจ่ายทั่วไป (apt, yum, Docker image) ล้วนใช้ 8KB เป็นค่ามาตรฐาน

### ทำไมต้อง 8KB

| เหตุผล | รายละเอียด |
|---|---|
| **สมดุลกับ OS page size** | Linux ใช้ page size 4KB เป็นหลัก 8KB คือ 2 เท่าพอดี ลด overhead การจัดการ memory mapping |
| **สมดุล I/O throughput vs latency** | page เล็กเกินไป → I/O operation ถี่เกินไป, page ใหญ่เกินไป → อ่าน/เขียนข้อมูลที่ไม่จำเป็นเยอะ (read amplification) |
| **จำกัดขนาด tuple สูงสุด** | ป้องกันไม่ให้แถวเดียวใหญ่จนไม่มีที่เก็บ metadata อื่นในหน้าเดียวกันเลย (ดู TOAST ใน Step 817-819) |
| **เข้ากันได้กับ WAL record** | LSN (Log Sequence Number) และกลไก WAL ออกแบบให้อ้างอิง page-level changes ได้พอดี |

### ตารางคือ array ของ page

```
┌─────────────────────────────────────────────────────────────────┐
│  ไฟล์ base/16401/16404 (ตาราง products)                          │
│                                                                   │
│  Block 0        Block 1        Block 2        ...   Block N     │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐         ┌──────────┐  │
│  │  8192 B  │  │  8192 B  │  │  8192 B  │   ...   │  8192 B  │  │
│  └──────────┘  └──────────┘  └──────────┘         └──────────┘  │
│                                                                   │
│  file offset:   0            8192          16384        N*8192  │
└─────────────────────────────────────────────────────────────────┘
```

Block number คือสิ่งที่ปรากฏใน `ctid` ตัวแรก เช่น `(7,23)` หมายถึง block 7, item index 23 ในหน้านั้น

### จำนวน page ของตารางจริง

```sql
SELECT
    relname,
    relpages,                                  -- จำนวน page ที่ query planner "จำ" ไว้ (อาจไม่ real-time)
    pg_relation_size(oid) / current_setting('block_size')::int AS actual_pages,
    pg_size_pretty(pg_relation_size(oid)) AS size
FROM pg_class
WHERE relname = 'products';
```

```
 relname  | relpages | actual_pages |  size
----------+----------+--------------+--------
 products |       12 |           11 | 88 kB
```

> `relpages` มาจากสถิติที่ `ANALYZE`/`VACUUM` เก็บไว้ล่าสุด อาจไม่ตรงกับขนาดไฟล์จริง ณ ขณะนี้แบบ real-time ถ้าต้องการค่าที่แม่นยำ ให้ใช้ `pg_relation_size()` โดยตรงหรือรัน `ANALYZE` ก่อน

### page คือหน่วย I/O พื้นฐาน

เวลา PostgreSQL อ่านข้อมูล มันไม่ได้อ่านทีละแถว แต่อ่านทีละ page เข้าสู่ **shared_buffers** (จะพูดถึงเชิงลึกใน Part เกี่ยวกับ buffer manager) แม้ query จะต้องการแค่ 1 แถว หากแถวนั้นอยู่ใน page ที่มี 50 แถว ระบบก็ต้องดึงทั้ง page (8KB) เข้า memory มาก่อน นี่คือเหตุผลที่ **sequential scan บนตารางที่ column กว้าง (TEXT ยาว ๆ) จะช้ากว่าที่คิด** เพราะแต่ละ page บรรจุจำนวนแถวได้น้อยลง ต้องอ่าน page เยอะขึ้นเพื่อครอบคลุมจำนวนแถวเท่าเดิม

```sql
-- ประมาณจำนวนแถวเฉลี่ยต่อ page ของตาราง products
SELECT
    relname,
    reltuples::bigint AS estimated_rows,
    relpages,
    ROUND(reltuples / NULLIF(relpages, 0), 1) AS avg_rows_per_page
FROM pg_class
WHERE relname = 'products';
```

```
 relname  | estimated_rows | relpages | avg_rows_per_page
----------+----------------+----------+--------------------
 products |            500 |       12 |               41.7
```

---

## Step 814: Page Layout เจาะลึก

นี่คือ Step ที่สำคัญที่สุดของบทนี้ — โครงสร้างภายใน page หนึ่งหน้า

### ภาพรวม page layout

```
┌───────────────────────────────────────────────────────────────────┐
│  PAGE (8192 bytes)                                                 │
│                                                                     │
│  offset 0    ┌─────────────────────────────────────────┐          │
│              │  PageHeaderData (24 bytes)               │          │
│              │  pd_lsn, pd_checksum, pd_flags,          │          │
│              │  pd_lower, pd_upper, pd_special,         │          │
│              │  pd_pagesize_version, pd_prune_xid       │          │
│              ├─────────────────────────────────────────┤ ◄ pd_lower│
│              │  Item Pointers (Line Pointers)           │          │
│              │  [1] → offset, length, flags             │          │
│              │  [2] → offset, length, flags             │  เติบโต  │
│              │  [3] → offset, length, flags             │  ลงล่าง ▼│
│              │  ...  (4 bytes ต่อรายการ)                │          │
│              ├─────────────────────────────────────────┤          │
│              │                                           │          │
│              │        ░░░ FREE SPACE ░░░                │  ◄── ช่องว่างสำหรับ
│              │                                           │      tuple ใหม่   │
│              ├─────────────────────────────────────────┤ ◄ pd_upper│
│              │  Tuple #3 (HeapTupleHeader + data)       │  เติบโต  │
│              ├─────────────────────────────────────────┤  ขึ้นบน ▲│
│              │  Tuple #2 (HeapTupleHeader + data)       │          │
│              ├─────────────────────────────────────────┤          │
│              │  Tuple #1 (HeapTupleHeader + data)       │          │
│  offset 8191 └─────────────────────────────────────────┘ ◄ pd_special
└───────────────────────────────────────────────────────────────────┘
```

**หลักการสำคัญ**: Item Pointers เติบโตจาก **บนลงล่าง** (หลัง header) ในขณะที่ Tuple Data จริงเติบโตจาก **ล่างขึ้นบน** (จากท้าย page) ทั้งสองฝั่งวิ่งเข้าหากัน ตรงกลางคือ free space — เมื่อ `pd_lower` และ `pd_upper` ชนกัน page นั้นก็เต็ม

### PageHeaderData ฟิลด์สำคัญ

| ฟิลด์ | ขนาด | ความหมาย |
|---|---|---|
| `pd_lsn` | 8 bytes | LSN ล่าสุดที่แก้ไข page นี้ (เชื่อมกับ WAL, Part 083) |
| `pd_checksum` | 2 bytes | checksum ของ page (ถ้าเปิด `data_checksums`) |
| `pd_flags` | 2 bytes | flag บอกสถานะ page เช่น มี free line pointer หรือไม่ |
| `pd_lower` | 2 bytes | offset จุดสิ้นสุดของ item pointer array (จุดเริ่มต้น free space) |
| `pd_upper` | 2 bytes | offset จุดเริ่มต้นของ tuple data ที่อยู่ใกล้ท้ายสุด (จุดสิ้นสุด free space) |
| `pd_special` | 2 bytes | offset ของ "special space" ใช้โดย index AM (เช่น B-Tree เก็บ sibling pointer) heap table ไม่ใช้ส่วนนี้ |
| `pd_pagesize_version` | 2 bytes | ขนาด page + เวอร์ชัน layout |
| `pd_prune_xid` | 4 bytes | xid ที่เก่าที่สุดที่อาจ prune ได้ ใช้เร่งการตัดสินใจของ VACUUM |

**Free space ของ page = `pd_upper - pd_lower`**

### Item Pointer (Line Pointer)

แต่ละ item pointer มีขนาด **4 bytes** ประกอบด้วย:

```
struct ItemIdData {
    unsigned lp_off:15,   /* byte offset ของ tuple ใน page */
             lp_flags:2,  /* LP_UNUSED / LP_NORMAL / LP_REDIRECT / LP_DEAD */
             lp_len:15;   /* ความยาวของ tuple เป็น byte */
};
```

Item pointer คือสิ่งที่ทำให้ `ctid` ทำงานได้ — `ctid = (block_number, item_pointer_index)` เมื่อระบบต้องการอ่านแถวจาก ctid มันจะ: (1) เปิด page ตาม block_number (2) มองที่ item pointer ตำแหน่งที่ระบุ (3) กระโดดไปอ่าน tuple ที่ offset นั้น

`lp_flags` มี 4 สถานะ:

| ค่า | ความหมาย |
|---|---|
| `LP_UNUSED` (0) | ช่องว่าง ไม่ได้ใช้งาน พร้อมนำไป reuse |
| `LP_NORMAL` (1) | ชี้ไปยัง tuple ที่ใช้งานได้จริง |
| `LP_REDIRECT` (2) | ใช้ใน HOT chain — ชี้ต่อไปยัง item pointer อื่นแทนที่จะชี้ไป tuple โดยตรง |
| `LP_DEAD` (3) | tuple ตายแล้ว (dead) รอ vacuum เก็บกวาด |

### ดูค่าจริงด้วย pageinspect (preview — รายละเอียดเต็มใน Step 816)

```sql
CREATE EXTENSION IF NOT EXISTS pageinspect;

SELECT lower, upper, special, pagesize, version
FROM page_header(get_raw_page('products', 0));
```

```
 lower | upper | special | pagesize | version
-------+-------+---------+----------+---------
   208 |  6920 |    8192 |     8192 |       4
```

การอ่าน:
- `lower = 208` → มี item pointer ~(208-24)/4 = **46 รายการ** (24 คือขนาด header)
- `upper = 6920` → tuple data เริ่มที่ offset 6920 นับจากบนสุด
- **free space** = `upper - lower` = 6920 - 208 = **6712 bytes** ยังว่างอยู่เยอะ พอรับ tuple ใหม่ได้อีกหลายสิบแถว
- `special = 8192` → เท่ากับ pagesize แปลว่าไม่มี special space (เพราะนี่คือ heap ไม่ใช่ index)

```sql
-- นับ item pointers จริงจาก heap_page_items
SELECT count(*) FROM heap_page_items(get_raw_page('products', 0));
```

```
 count
-------
    46
```

ตรงกับที่คำนวณไว้พอดี — นี่คือหลักฐานว่าเราเข้าใจ layout ถูกต้อง

---

## Step 815: Tuple Header

ทุกแถวข้อมูล (tuple) ใน heap page ไม่ได้มีแค่ข้อมูล column ของเราเท่านั้น แต่มี **HeapTupleHeaderData** นำหน้าเสมอ — นี่คือ metadata ที่ทำให้ MVCC (ทบทวน Part 058) ทำงานได้

### โครงสร้าง HeapTupleHeaderData (23 bytes ก่อน padding)

```
┌────────────────────────────────────────────────────────────────┐
│  HeapTupleHeaderData                                            │
│  ┌──────────────┬──────────────┬──────────────┬──────────────┐ │
│  │  t_xmin (4B) │  t_xmax (4B) │  t_cid (4B)  │ t_ctid (6B)   │ │
│  │  ใคร insert  │  ใคร delete/ │ command id   │  ชี้ไปยัง      │ │
│  │  ทำให้เกิดแถว│  update แถวนี้│  ภายใน tx    │  tuple version │ │
│  │  นี้ (xid)   │  (xid หรือ 0)│              │  ล่าสุด (self  │ │
│  │              │              │              │  ถ้ายังไม่ถูก  │ │
│  │              │              │              │  update)       │ │
│  ├──────────────┼──────────────┼──────────────┼──────────────┤ │
│  │ t_infomask2  │  t_infomask  │   t_hoff     │  null bitmap  │ │
│  │   (2B)       │    (2B)      │   (1B)       │  (variable,   │ │
│  │  จำนวน       │  flag บอก    │  offset ที่   │  ถ้ามี NULL   │ │
│  │  attribute   │  สถานะ tuple │  ข้อมูลจริง   │  column)      │ │
│  │              │  (HOT, etc.) │  เริ่มต้น     │               │ │
│  └──────────────┴──────────────┴──────────────┴──────────────┘ │
├────────────────────────────────────────────────────────────────┤
│  User Data (ค่าจริงของแต่ละ column ตามที่นิยามในตาราง)          │
└────────────────────────────────────────────────────────────────┘
```

### ฟิลด์ที่สำคัญที่สุดสำหรับ MVCC

| ฟิลด์ | ความหมาย | เชื่อมโยง |
|---|---|---|
| **`t_xmin`** | Transaction ID ที่ insert แถวนี้ (หรือ update ที่สร้างแถวนี้ขึ้นมา) | แถวจะ "มองเห็นได้" ก็ต่อเมื่อ `t_xmin` commit แล้วและอยู่ก่อน snapshot ของ transaction ที่กำลังอ่าน |
| **`t_xmax`** | Transaction ID ที่ delete/update แถวนี้ (0 ถ้ายังไม่ถูกลบ) | แถวจะ "หายไป" จาก snapshot เมื่อ `t_xmax` commit และอยู่ก่อน snapshot นั้น |
| **`t_ctid`** | ชี้ไปยัง tuple version ถัดไป (หลัง update) หรือชี้ตัวเองถ้าเป็น version ล่าสุด | ใช้ไล่ **update chain** — Part 084 จะเจาะลึกเรื่อง HOT chain |
| **`t_infomask`** | bit flags เช่น `HEAP_XMIN_COMMITTED`, `HEAP_XMAX_INVALID`, `HEAP_HASNULL` | บอกสถานะ commit ของ transaction ที่เกี่ยวข้อง (hint bits) |

> เรื่อง MVCC เต็มรูปแบบเคยอธิบายไว้ใน **Part 058** ที่นี่เราแค่โยงให้เห็นว่า "ทฤษฎี MVCC" ที่เคยเรียนนั้น ในระดับ physical จริง ๆ แล้วถูก implement ด้วยฟิลด์ `t_xmin`/`t_xmax`/`t_ctid` นี่เอง — และ Part 084 (ถัดจาก WAL ใน Part 083) จะพาไปดู **HOT update, xmin/xmax หมุนเวียน, freeze** แบบละเอียดที่สุด

### ดูค่าจริงจากตาราง products

```sql
SELECT
    lp AS item_id,
    t_ctid,
    t_xmin,
    t_xmax,
    t_infomask,
    t_infomask2
FROM heap_page_items(get_raw_page('products', 0))
LIMIT 5;
```

```
 item_id | t_ctid | t_xmin | t_xmax | t_infomask | t_infomask2
---------+--------+--------+--------+------------+-------------
       1 | (0,1)  |    752 |      0 |       2306 |           4
       2 | (0,2)  |    752 |      0 |       2306 |           4
       3 | (0,3)  |    752 |      0 |       2306 |           4
       4 | (0,4)  |    752 |      0 |       2306 |           4
       5 | (7,23) |    752 |    801 |       2306 |           4
```

การอ่าน:
- `t_xmin = 752` คือ transaction ที่ insert แถวเหล่านี้ (ตอนเรารัน `INSERT ... generate_series`)
- แถวที่ 5 มี `t_xmax = 801` และ `t_ctid = (7,23)` ซึ่งไม่ใช่ `(0,5)` — นี่คือแถวที่เราเคย `UPDATE` ไปใน Step 811! `t_xmax` บอกว่า version นี้ "ถูกแทนที่" โดย transaction 801 และ `t_ctid` ชี้ไปหา version ใหม่ที่ block 7, item 23
- แถวอื่น (`t_xmax = 0`) ยังเป็น version ล่าสุด ไม่เคยถูกแก้ไข

ลองตามไปดู version ใหม่ที่ `(7,23)`:

```sql
SELECT lp, t_ctid, t_xmin, t_xmax
FROM heap_page_items(get_raw_page('products', 7))
WHERE lp = 23;
```

```
 lp | t_ctid | t_xmin | t_xmax
----+--------+--------+--------
 23 | (7,23) |    801 |      0
```

`t_ctid = (7,23)` ชี้กลับมาที่ตัวเอง → นี่คือ **tuple version ล่าสุด (head of chain)** และ `t_xmax = 0` แปลว่ายังไม่ถูกลบ/แก้ไขอีก — นี่คือวิธีที่ PostgreSQL implement "update chain" ในระดับ physical จริง ๆ

---

## Step 816: pageinspect Extension

`pageinspect` คือ extension มาตรฐานของ PostgreSQL ที่ให้เรา "ผ่าดู" page และ tuple ได้ในระดับ byte โดยไม่ต้องเขียนโค้ด C เอง เหมาะสำหรับการเรียนรู้ internals และการ debug ปัญหาเชิงลึก (เช่น corruption, bloat)

### ติดตั้ง

```sql
CREATE EXTENSION IF NOT EXISTS pageinspect;
```

> ต้องมีสิทธิ์ superuser หรือสิทธิ์ที่ได้รับมอบหมายให้สร้าง extension นี้ และบางฟังก์ชันต้องมีสิทธิ์ `pg_read_server_files` หรือ superuser เพราะเข้าถึงข้อมูลระดับไฟล์โดยตรง

### ฟังก์ชันหลักที่ควรรู้จัก

| ฟังก์ชัน | หน้าที่ |
|---|---|
| `get_raw_page(relname, blkno)` | ดึง page ดิบ (raw bytes) ของ block ที่ระบุ |
| `page_header(raw_page)` | แยกฟิลด์ของ PageHeaderData ออกมาอ่านง่าย |
| `heap_page_items(raw_page)` | แสดง item pointer + tuple header ของทุก tuple ใน page (heap table) |
| `heap_page_item_attrs(raw_page, relname)` | เหมือนข้างบนแต่ถอดค่า attribute (column) จริงออกมาด้วย |
| `tuple_data_split(...)` | แยก raw tuple data ออกเป็นแต่ละ attribute ระดับ byte |
| `bt_metap(indexname)` | อ่าน metadata page ของ B-Tree index |
| `bt_page_stats(indexname, blkno)` | สถิติของ page หนึ่งใน B-Tree index |
| `bt_page_items(indexname, blkno)` | รายการ item ใน B-Tree page |
| `page_checksum(raw_page, blkno)` | คำนวณ checksum ของ page (ถ้าเปิด `data_checksums`) |

### ตัวอย่างการใช้งานแบบครบวงจรกับ products

**1) ดู page header:**

```sql
SELECT * FROM page_header(get_raw_page('products', 0));
```

```
   lsn    | checksum | flags | lower | upper | special | pagesize | version | prune_xid
----------+----------+-------+-------+-------+---------+----------+---------+-----------
 0/1A3F210|        0 |     4 |   208 |  6920 |    8192 |     8192 |       4 |         0
```

**2) ดู item pointers + tuple headers ทั้งหมดใน page:**

```sql
SELECT
    lp,
    lp_off,
    lp_len,
    t_xmin,
    t_xmax,
    t_ctid
FROM heap_page_items(get_raw_page('products', 0))
ORDER BY lp
LIMIT 8;
```

```
 lp | lp_off | lp_len | t_xmin | t_xmax | t_ctid
----+--------+--------+--------+--------+--------
  1 |   8130 |     62 |    752 |      0 | (0,1)
  2 |   8068 |     62 |    752 |      0 | (0,2)
  3 |   8006 |     61 |    752 |      0 | (0,3)
  4 |   7944 |     61 |    752 |      0 | (0,4)
  5 |   6920 |     56 |    752 |    801 | (7,23)
  6 |   7878 |     62 |    752 |      0 | (0,6)
  7 |   7816 |     62 |    752 |      0 | (0,7)
  8 |   7754 |     62 |    752 |      0 | (0,8)
```

สังเกต `lp_off` ลดลงเรื่อย ๆ (8130 → 8068 → 8006 ...) — ยืนยัน diagram ใน Step 814 ว่า tuple data เขียนจาก **ท้าย page ขึ้นมาบน** โดย tuple แรกอยู่ใกล้ offset 8192 มากที่สุด

**3) ดูค่า column จริงด้วย `heap_page_item_attrs`:**

```sql
SELECT
    lp,
    t_attrs[1] AS product_id_raw,
    t_attrs[2] AS product_name_raw
FROM heap_page_item_attrs(get_raw_page('products', 0), 'products')
LIMIT 3;
```

```
 lp | product_id_raw | product_name_raw
----+-----------------+-------------------
  1 | \x01000000       | \x0950726f647563...
  2 | \x02000000       | \x0950726f647563...
  3 | \x03000000       | \x0950726f647563...
```

ค่าที่ได้เป็น **raw bytes** (bytea) — `\x01000000` คือเลข 1 แบบ little-endian 4-byte integer (product_id) และค่าถัดมาคือ VARCHAR ที่เก็บแบบ varlena (1-byte length prefix + data) นี่คือสิ่งที่อยู่ "จริง ๆ" บนดิสก์ ก่อนที่ PostgreSQL จะแปลงกลับมาเป็นค่าที่มนุษย์อ่านได้ให้เรา

**4) รวมทุกอย่างเป็น query สรุปสภาพ page:**

```sql
SELECT
    'products' AS table_name,
    blkno,
    (page_header(get_raw_page('products', blkno))).lower,
    (page_header(get_raw_page('products', blkno))).upper,
    (page_header(get_raw_page('products', blkno))).upper -
        (page_header(get_raw_page('products', blkno))).lower AS free_space_bytes,
    (SELECT count(*) FROM heap_page_items(get_raw_page('products', blkno))) AS tuple_count
FROM generate_series(0, 3) AS blkno;
```

```
 table_name | blkno | lower | upper | free_space_bytes | tuple_count
------------+-------+-------+-------+-------------------+-------------
 products   |     0 |   208 |  6920 |              6712 |          46
 products   |     1 |   204 |  6980 |              6776 |          45
 products   |     2 |   212 |  6890 |              6678 |          47
 products   |     3 |   200 |  7050 |              6850 |          44
```

query นี้คือเครื่องมือระดับ production จริง ที่ DBA ใช้ตรวจสอบว่า page มี free space เหลือแค่ไหน (เพื่อวิเคราะห์ bloat) โดยไม่ต้องพึ่ง extension เสริมอื่นเลย นอกจาก pageinspect

> ⚠️ **คำเตือนเรื่อง production**: `pageinspect` อ่านข้อมูลตรงจากดิสก์ผ่าน shared buffer เหมือน query ปกติ จึงค่อนข้างปลอดภัยสำหรับการอ่าน (read-only) แต่ไม่ควรรันวนลูปกับตารางใหญ่มากบน production เพราะจะทำให้ buffer cache ถูกแย่งไปเก็บ page จำนวนมากโดยไม่จำเป็น

---

## Step 817: TOAST คืออะไร

### ปัญหา: ข้อมูลใหญ่กว่า page เดียว

เรารู้จาก Step 813 แล้วว่า page มีขนาดคงที่ **8KB** แล้วถ้า column `description TEXT` ของเรามีข้อมูลยาว 50KB ล่ะ? จะเก็บยังไงในเมื่อทั้งแถวต้องพอดีกับ page เดียว (PostgreSQL heap **ไม่รองรับ row-spanning ข้าม page แบบตรงไปตรงมา**)?

คำตอบคือ **TOAST — The Oversized-Attribute Storage Technique**

### กฎเหล็ก: TOAST_TUPLE_THRESHOLD

PostgreSQL มีกฎว่า **อย่างน้อยต้องเก็บ tuple ได้ 4 แถวต่อ 1 page** (เพื่อรักษาประสิทธิภาพของ page-based storage ไม่ให้ page ถูกครอบครองโดยแถวเดียว) นั่นหมายความว่าแถวหนึ่งไม่ควรใหญ่เกิน **ประมาณ 2KB** (8KB / 4)

ค่าที่ใช้จริงคือ `TOAST_TUPLE_THRESHOLD` = **2KB** โดยประมาณ (ค่าจริงคำนวณจาก `BLCKSZ / 4` ลบด้วย overhead ต่าง ๆ) — เมื่อ tuple **รวมทุก column แล้วมีขนาดเกินเกณฑ์นี้** PostgreSQL จะเริ่มกระบวนการ TOAST กับ column ที่เป็น **varlena type** (คือ type ที่มีความยาวแปรผันได้ เช่น `TEXT`, `VARCHAR`, `BYTEA`, `JSONB`, array types) เพื่อลดขนาด tuple ลงให้พอดีกับ page

### กระบวนการ TOAST (ลำดับขั้น)

```
┌─────────────────────────────────────────────────────────────────┐
│  Tuple ใหญ่เกิน TOAST_TUPLE_THRESHOLD (~2KB)                      │
└───────────────────────────┬─────────────────────────────────────┘
                             │
                             ▼
        ┌────────────────────────────────────────┐
        │  ขั้นที่ 1: ลองบีบอัด (compress)          │
        │  ใช้ pglz (default) หรือ lz4 (PG14+)     │
        │  เลือก column ที่ใหญ่ที่สุดก่อน           │
        └───────────────────┬──────────────────────┘
                             │ ยังใหญ่เกินอยู่?
                             ▼
        ┌────────────────────────────────────────┐
        │  ขั้นที่ 2: ย้ายออกนอก tuple (out-of-line)│
        │  เก็บ column นั้นไว้ใน "TOAST table"      │
        │  แยกต่างหาก แล้วเก็บแค่ "pointer"          │
        │  (18 bytes) ไว้ในตำแหน่งเดิม               │
        └───────────────────┬──────────────────────┘
                             │ ทำซ้ำกับ column ถัดไป
                             │ จนกว่า tuple จะเล็กพอ
                             ▼
                  Tuple พอดีกับ page แล้ว
```

### ภาพรวมความสัมพันธ์

```
┌───────────────────────────────────────────────────────────────────┐
│  Main table: products (heap)                                       │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │ product_id | product_name | description        | unit_price│    │
│  │     1      | "Widget A"   | [TOAST POINTER] ────┼──────┐   │    │
│  │            |              | (18 bytes เท่านั้น) │      │   │    │
│  └──────────────────────────────────────────────────────────┘     │
│                                                          │          │
└──────────────────────────────────────────────────────────┼──────────┘
                                                             │
                                                             ▼
┌───────────────────────────────────────────────────────────────────┐
│  TOAST table: pg_toast.pg_toast_16404                              │
│  ┌────────────┬───────────┬──────────────────────────────┐        │
│  │ chunk_id    │ chunk_seq │ chunk_data (≤ ~2000 bytes)   │        │
│  ├────────────┼───────────┼──────────────────────────────┤        │
│  │  16500      │     0     │ "ข้อมูล description ท่อนที่ 1"│        │
│  │  16500      │     1     │ "ข้อมูล description ท่อนที่ 2"│        │
│  │  16500      │     2     │ "ข้อมูล description ท่อนที่ 3"│        │
│  └────────────┴───────────┴──────────────────────────────┘        │
└───────────────────────────────────────────────────────────────────┘
```

ข้อมูลใหญ่จะถูก "สับ" เป็นชิ้น ๆ (chunk) ขนาดประมาณ 2000 bytes เก็บใน TOAST table ที่มี B-Tree index ทำให้การ "ประกอบร่างกลับ" (reassemble) เวลา query ทำได้เร็ว

### ทำไมออกแบบแบบนี้ถึงฉลาด

1. **Query ที่ไม่แตะ column ใหญ่ ไม่ต้องจ่าย cost** — ถ้า query `SELECT product_id, unit_price FROM products` ไม่ได้เลือก `description` เลย ระบบไม่จำเป็นต้องไปอ่าน TOAST table เลย เพราะ pointer 18 bytes อยู่ใน main tuple แต่ไม่ได้ dereference
2. **Compression ลดพื้นที่และ I/O** — ข้อความยาวมักบีบอัดได้ดี (โดยเฉพาะ text ซ้ำ ๆ) การบีบอัดก่อนช่วยประหยัดทั้ง disk และ cache
3. **Main table page แน่น กระชับ** — ทำให้ sequential scan และ index scan บน column ปกติ (ไม่ TOAST) ยังคงเร็ว เพราะไม่ต้องแบกข้อมูลใหญ่ไปด้วยทุกครั้ง

---

## Step 818: TOAST Strategy — PLAIN, EXTENDED, EXTERNAL, MAIN

PostgreSQL ให้เราเลือก **storage strategy** ต่อ column ได้ ว่าจะอนุญาตให้ compress และ/หรือ move out-of-line หรือไม่

### ตารางเปรียบเทียบ 4 กลยุทธ์

| Strategy | Compression | Out-of-line (ย้ายไป TOAST table) | ใช้กับ type ไหนเป็นค่า default |
|---|---|---|---|
| **PLAIN** | ❌ ไม่บีบอัด | ❌ ไม่ย้ายออก (บังคับให้อยู่ใน tuple เสมอ) | fixed-length types เช่น `INTEGER`, `NUMERIC` ที่ไม่ใช่ varlena จริง ๆ, `boolean` |
| **MAIN** | ✅ บีบอัดก่อน | ⚠️ ย้ายออกเป็นทางเลือกสุดท้าย (พยายามคงไว้ใน main tuple ให้นานที่สุด) | `NUMERIC`, บาง type ที่อยากให้อยู่ใกล้แถวหลักเพราะ query บ่อย |
| **EXTENDED** | ✅ บีบอัดก่อน | ✅ ย้ายออกได้ถ้าจำเป็น (ลำดับปกติ: compress ก่อน แล้วค่อย externalize) | **ค่า default ของ `TEXT`, `VARCHAR`, `JSONB`, `BYTEA`, array ส่วนใหญ่** |
| **EXTERNAL** | ❌ ไม่บีบอัด | ✅ ย้ายออกทันทีถ้าเกิน threshold | ไม่ใช่ default ของ type ไหน ต้องตั้งเอง (เหมาะกับข้อมูลที่ query บ่อยแบบ substring และไม่บีบอัดง่าย เช่น ข้อมูล binary ที่บีบอัดแล้วไม่ลดขนาด) |

### ทำไม EXTERNAL ถึงเร่ง substring operation

ถ้า column เป็น **EXTENDED** (บีบอัดแล้ว) การดึงข้อมูลบางส่วน เช่น `substring(description, 1, 100)` จะต้อง **decompress ข้อมูลทั้งก้อนก่อน** ถึงจะตัดท่อนที่ต้องการได้ แต่ถ้าเป็น **EXTERNAL** (ไม่บีบอัด) ระบบสามารถอ่านเฉพาะ chunk ที่ต้องการจาก TOAST table ได้โดยไม่ต้อง decompress ทั้งหมด — เหมาะกับข้อมูลใหญ่ที่มักถูก query แบบ partial บ่อย ๆ แต่แลกมาด้วยพื้นที่ disk ที่มากกว่า

### ดู strategy ปัจจุบันของแต่ละ column

```sql
SELECT
    a.attname AS column_name,
    t.typname AS data_type,
    CASE a.attstorage
        WHEN 'p' THEN 'PLAIN'
        WHEN 'm' THEN 'MAIN'
        WHEN 'e' THEN 'EXTENDED'
        WHEN 'x' THEN 'EXTERNAL'  -- หมายเหตุ: 'x' ในที่นี้คือรหัสเดิม แต่ EXTERNAL จริงคือ 'e'->'x' สลับกันตาม version
    END AS storage_strategy
FROM pg_attribute a
JOIN pg_type t ON a.atttypid = t.oid
WHERE a.attrelid = 'products'::regclass
  AND a.attnum > 0
ORDER BY a.attnum;
```

```
 column_name  | data_type | storage_strategy
--------------+-----------+-------------------
 product_id   | int4      | PLAIN
 product_name | varchar   | EXTENDED
 description  | text      | EXTENDED
 unit_price   | numeric   | MAIN
```

> หมายเหตุเรื่อง code: ใน `pg_attribute.attstorage` ค่าที่แท้จริงคือ `'p'` = PLAIN, `'e'` = EXTENDED, `'x'` = EXTERNAL, `'m'` = MAIN (ตัวอักษรไม่ได้ตรงกับชื่อเต็มทุกตัว เป็น legacy naming จากซอร์สโค้ดดั้งเดิม) จุดสำคัญคือค่า default ของ `text`/`varchar` คือ **EXTENDED** และของ `numeric` คือ **MAIN**

### เปลี่ยน strategy ด้วยตัวเอง

```sql
-- ตัวอย่าง: บังคับให้ description ใช้ EXTERNAL แทน EXTENDED
-- (เหมาะถ้าเรารู้ว่าจะ query แบบ LIKE '%...%' หรือ substring บ่อย ๆ)
ALTER TABLE products ALTER COLUMN description SET STORAGE EXTERNAL;

-- ตรวจสอบผล
SELECT attname, attstorage
FROM pg_attribute
WHERE attrelid = 'products'::regclass AND attname = 'description';
```

```
  attname    | attstorage
-------------+------------
 description | x
```

> ⚠️ `ALTER ... SET STORAGE` เปลี่ยนแค่ **strategy สำหรับข้อมูลใหม่ที่จะเขียนต่อจากนี้** ข้อมูลเดิมที่มีอยู่แล้วจะไม่ถูกแปลงย้อนหลัง จนกว่าจะมีการ `UPDATE` แถวนั้นใหม่ หรือรัน `VACUUM FULL`/`CLUSTER` (ซึ่ง rewrite ทั้งตาราง)

### ทดสอบ compression จริง

```sql
-- ลองเปรียบเทียบขนาดก่อน/หลัง compress ด้วย pg_column_size
SELECT
    product_id,
    pg_column_size(description) AS compressed_or_inline_size,
    length(description) AS logical_char_length
FROM products
WHERE product_id = 1;
```

```
 product_id | compressed_or_inline_size | logical_char_length
------------+-----------------------------+-----------------------
          1 |                          48 |                    44
```

ในกรณีนี้ description สั้นมาก (44 ตัวอักษร) ยังไม่ถึงเกณฑ์ TOAST เลย เก็บแบบ inline ธรรมดา (`compressed_or_inline_size` ใกล้เคียง `logical_char_length` บวก overhead 4 bytes ของ varlena header) — เดี๋ยว Step 820 จะทดสอบกับข้อมูลที่ใหญ่จริง ๆ ให้เห็นความแตกต่างชัดเจน

---

## Step 819: TOAST Table

### pg_toast schema

ทุกตารางที่มี column ซึ่งอาจต้อง TOAST (คือมี varlena column ที่ไม่ใช่ PLAIN ล้วน) PostgreSQL จะสร้าง **TOAST table คู่กัน** โดยอัตโนมัติตั้งแต่ตอน `CREATE TABLE` — อยู่ใน schema พิเศษชื่อ `pg_toast` ชื่อตามรูปแบบ `pg_toast_<oid ของตารางหลัก>`

```sql
-- หา TOAST table ของ products
SELECT
    c.relname AS main_table,
    c.reltoastrelid,
    t.relname AS toast_table_name
FROM pg_class c
JOIN pg_class t ON c.reltoastrelid = t.oid
WHERE c.relname = 'products';
```

```
 main_table | reltoastrelid | toast_table_name
------------+----------------+-------------------
 products   |          16410 | pg_toast_16404
```

```sql
-- ดูรายละเอียดผ่าน pg_class ตรง ๆ ใน schema pg_toast
SELECT relname, relnamespace::regnamespace, relkind, pg_size_pretty(pg_relation_size(oid))
FROM pg_class
WHERE relname = 'pg_toast_16404';
```

```
     relname     | relnamespace | relkind | pg_size_pretty
------------------+--------------+---------+-----------------
 pg_toast_16404   | pg_toast     | t       | 0 bytes
```

`relkind = 't'` คือรหัสพิเศษสำหรับ TOAST table โดยเฉพาะ (ต่างจาก `'r'` = ordinary table, `'i'` = index) ตอนนี้ขนาด 0 bytes เพราะยังไม่มีข้อมูลใหญ่พอที่จะ trigger TOAST เลย

### โครงสร้างของ TOAST table

TOAST table ทุกตัวมีโครงสร้างเหมือนกันหมด ไม่ว่าตารางหลักจะหน้าตาอย่างไร:

```sql
CREATE TABLE pg_toast.pg_toast_16404 (
    chunk_id    oid,      -- อ้างอิงกลุ่มของค่า TOAST เดียวกัน (ค่าเดียวกันทุก chunk ของ 1 attribute)
    chunk_seq   int4,     -- ลำดับชิ้นส่วน (0, 1, 2, ...)
    chunk_data  bytea     -- ข้อมูลจริงในชิ้นนี้ (สูงสุดประมาณ 2000 bytes)
);
-- และมี unique index บน (chunk_id, chunk_seq) เสมอ เพื่อให้ reassemble เร็ว
```

```sql
-- TOAST table แต่ละตัวก็มี index ของตัวเอง
SELECT indexrelid::regclass, indrelid::regclass
FROM pg_index
WHERE indrelid = 'pg_toast.pg_toast_16404'::regclass;
```

```
       indexrelid        |       indrelid
--------------------------+-------------------------
 pg_toast.pg_toast_16404_index | pg_toast.pg_toast_16404
```

### Pointer ที่เก็บไว้ใน main tuple

เมื่อ column ถูก TOAST แบบ out-of-line ค่าที่เก็บใน main tuple จริง ๆ ไม่ใช่ข้อมูล แต่คือ **`varatt_external` struct** ขนาด 18 bytes ประกอบด้วย:

| ฟิลด์ | ความหมาย |
|---|---|
| `va_rawsize` | ขนาดข้อมูลต้นฉบับก่อนบีบอัด (uncompressed size) |
| `va_extsize` | ขนาดข้อมูลที่เก็บจริงใน TOAST table (หลังบีบอัด ถ้ามี) |
| `va_valueid` | ตรงกับค่า `chunk_id` ใน TOAST table — ใช้หาชุด chunk ที่เกี่ยวข้อง |
| `va_toastrelid` | OID ของ TOAST table ที่เก็บข้อมูลนี้ |

### ดู pointer จริงผ่าน pageinspect

```sql
-- ต้องมีแถวที่ description ใหญ่พอจะ TOAST ก่อน (ดู Step 820 สำหรับการสร้างข้อมูลทดสอบ)
SELECT
    lp,
    t_attrs[3] AS description_raw   -- attribute ลำดับที่ 3 คือ description
FROM heap_page_item_attrs(get_raw_page('products', 0), 'products')
WHERE lp = 1;
```

เมื่อค่ายังไม่ถูก TOAST (inline) จะเห็น raw bytea ของข้อความเต็ม ๆ แต่เมื่อค่าถูก TOAST ออกไปแล้ว ค่าที่เห็นจะเป็นแค่ struct 18-byte pointer ที่กล่าวถึงข้างต้น (เข้ารหัสในรูป bytea เช่นกัน แต่สั้นมาก และตัวข้อมูลจริงต้องไปดูที่ TOAST table)

### query TOAST table โดยตรง (ปกติไม่ทำ แต่มีประโยชน์เชิง diagnostic)

```sql
-- นับจำนวน chunk ทั้งหมดที่ถูกเก็บ (ถ้ามี)
SELECT count(*), count(DISTINCT chunk_id) AS distinct_values
FROM pg_toast.pg_toast_16404;
```

```
 count | distinct_values
-------+-------------------
     0 |                0
```

ตอนนี้ยังว่างเปล่า เพราะ description ของเรายังสั้นเกินไป — มาสร้างสถานการณ์จริงกันใน Step ถัดไป

> 💡 ข้อควรรู้: การ query `pg_toast.*` โดยตรงเป็นสิ่งที่ **ผู้ใช้งานทั่วไปไม่ควรทำในระบบ production** เพราะเป็น internal implementation detail — แอปพลิเคชันควรอ่านผ่าน main table เสมอ (`SELECT description FROM products`) แล้วให้ PostgreSQL reassemble ให้อัตโนมัติ การเข้าถึง TOAST table ตรง ๆ มีประโยชน์เฉพาะตอน debug/เรียนรู้ internals เท่านั้น

---

## Step 820: แบบฝึกหัดรวม — สำรวจโครงสร้างจริงด้วย pageinspect และสังเกตพฤติกรรม TOAST

มาสร้างสถานการณ์ทดสอบแบบครบวงจร เพื่อเห็นทุกอย่างที่เรียนมาทำงานร่วมกันจริง

### ขั้นที่ 1: เตรียมข้อมูลขนาดใหญ่พอที่จะ TOAST

```sql
-- คืนค่า storage strategy เป็นค่า default ก่อน (EXTENDED)
ALTER TABLE products ALTER COLUMN description SET STORAGE EXTENDED;

-- เพิ่มแถวใหม่ที่มี description ยาวมาก (ประมาณ 5000 ตัวอักษร บีบอัดยาก เพราะสุ่ม)
INSERT INTO products (product_name, description, unit_price)
VALUES (
    'Bulk Description Product',
    (SELECT string_agg(md5(random()::text), '') FROM generate_series(1, 160)), -- ~ 5120 chars, บีบอัดยาก
    999.99
);
```

### ขั้นที่ 2: เปรียบเทียบขนาด logical vs physical

```sql
SELECT
    product_id,
    length(description) AS char_length,
    pg_column_size(description) AS stored_size_bytes,
    octet_length(description) AS byte_length
FROM products
WHERE product_name = 'Bulk Description Product';
```

```
 product_id | char_length | stored_size_bytes | byte_length
------------+--------------+---------------------+--------------
        501 |         5120 |                  18 |         5120
```

**สังเกต**: `stored_size_bytes = 18` — นี่คือขนาดของ **TOAST pointer** ไม่ใช่ข้อมูลจริง! ข้อมูล 5120 ตัวอักษรถูกย้ายออกไปเก็บใน TOAST table หมดแล้ว เพราะ md5 hash เป็นข้อมูลสุ่มบีบอัดได้ไม่ดี (เกิน threshold ~2KB จึงต้อง externalize)

### ขั้นที่ 3: ยืนยันด้วยการดู TOAST table โดยตรง

```sql
SELECT
    chunk_id,
    count(*) AS num_chunks,
    sum(octet_length(chunk_data)) AS total_bytes
FROM pg_toast.pg_toast_16404
GROUP BY chunk_id;
```

```
 chunk_id | num_chunks | total_bytes
----------+-------------+--------------
    16550 |           3 |        5120
```

ข้อมูล 5120 bytes ถูกสับเป็น **3 chunks** (chunk ละประมาณ 2000 bytes ตามที่อธิบายใน Step 817) และรวมกันได้พอดี 5120 bytes ตรงกับต้นฉบับ (ไม่ได้ถูกบีบอัดเลย เพราะ md5 hash แบบสุ่มบีบอัดไม่ได้ ดังนั้น va_rawsize = va_extsize)

### ขั้นที่ 4: ทดสอบ TOAST กับข้อมูลที่บีบอัดได้ดี

```sql
INSERT INTO products (product_name, description, unit_price)
VALUES (
    'Repetitive Text Product',
    repeat('PostgreSQL is a powerful open source object-relational database system. ', 100), -- ~7500 chars ซ้ำๆ
    499.50
);

SELECT
    product_id,
    length(description) AS char_length,
    pg_column_size(description) AS stored_size_bytes
FROM products
WHERE product_name = 'Repetitive Text Product';
```

```
 product_id | char_length | stored_size_bytes
------------+--------------+---------------------
        502 |         7500 |                  18
```

ยังคงเป็น 18 bytes (pointer) แต่คราวนี้มาดูว่าข้อมูลจริงใน TOAST table ถูก**บีบอัด**ไปเหลือเท่าไหร่:

```sql
SELECT
    chunk_id,
    count(*) AS num_chunks,
    sum(octet_length(chunk_data)) AS compressed_bytes_stored
FROM pg_toast.pg_toast_16404
GROUP BY chunk_id
ORDER BY chunk_id DESC
LIMIT 1;
```

```
 chunk_id | num_chunks | compressed_bytes_stored
----------+-------------+---------------------------
    16551 |           1 |                      312
```

**7500 ตัวอักษรที่ซ้ำ ๆ ถูกบีบอัดเหลือแค่ 312 bytes (chunk เดียวพอ!)** เพราะ pattern ซ้ำเยอะมาก บีบอัดได้ดีเป็นพิเศษ (compression ratio ~24:1) เทียบกับข้อมูลสุ่ม md5 ก่อนหน้านี้ที่บีบอัดไม่ได้เลย — นี่คือหลักฐานชัดเจนว่า **EXTENDED strategy พยายาม compress ก่อนเสมอ** และ TOAST อัจฉริยะพอที่จะไม่สร้าง chunk เกินความจำเป็น

### ขั้นที่ 5: สรุปภาพรวมด้วย query เดียว

```sql
SELECT
    p.product_id,
    p.product_name,
    length(p.description) AS logical_chars,
    pg_column_size(p.description) AS main_tuple_bytes,
    pg_column_size(p.description::text) - pg_column_size(p.description::text)
        AS placeholder, -- (แสดงไว้เผื่อขยายวิเคราะห์เพิ่มเติม)
    CASE
        WHEN pg_column_size(p.description) < length(p.description) THEN 'TOASTED (compressed/external)'
        ELSE 'INLINE'
    END AS toast_status
FROM products p
WHERE p.product_name IN (
    'Bulk Description Product',
    'Repetitive Text Product'
)
ORDER BY p.product_id;
```

```
 product_id |       product_name        | logical_chars | main_tuple_bytes | placeholder |          toast_status
------------+-----------------------------+----------------+--------------------+--------------+---------------------------------
        501 | Bulk Description Product   |           5120 |                 18 |           0 | TOASTED (compressed/external)
        502 | Repetitive Text Product    |           7500 |                 18 |           0 | TOASTED (compressed/external)
```

### สรุปสิ่งที่เราพิสูจน์ได้จริงในแบบฝึกหัดนี้

1. TOAST เกิดขึ้น**อัตโนมัติ** เมื่อข้อมูลเกินเกณฑ์ ~2KB โดยไม่ต้องตั้งค่าอะไรเพิ่ม
2. main tuple เก็บแค่ **pointer 18 bytes** ทำให้ query ที่ไม่แตะ column นั้นยังคงเร็ว
3. ข้อมูลที่บีบอัดได้ดี (ซ้ำ ๆ) จะถูกบีบอัดก่อนเสมอ (`EXTENDED` strategy) ลด storage ได้มหาศาล
4. ข้อมูลสุ่ม (เช่น hash, ข้อมูล encrypt แล้ว) บีบอัดไม่ได้ จะถูก externalize ตรง ๆ โดยไม่ประหยัดพื้นที่จาก compression
5. `pg_toast.pg_toast_<oid>` เป็นตารางจริงที่ query ได้ (แม้ไม่ควรทำใน production) และมีโครงสร้าง `chunk_id`, `chunk_seq`, `chunk_data` เหมือนกันทุกตาราง

---

## สรุปท้ายบท

ในบทนี้เราได้ "ผ่าตัด" PostgreSQL ลงไปถึงชั้นการจัดเก็บข้อมูลระดับกายภาพที่แท้จริง:

- **Heap organization**: PostgreSQL เก็บแถวแบบไม่เรียงลำดับ ต่างจาก clustered index ของ MySQL InnoDB โดยพื้นฐาน — `ctid` คือหลักฐานที่พิสูจน์ได้ว่าตำแหน่งจริงไม่สัมพันธ์กับ primary key
- **Data directory**: ตารางหนึ่งตัวจริง ๆ คือกลุ่มไฟล์ `base/<dboid>/<relfilenode>` พร้อม fork เสริม (`_fsm`, `_vm`) ที่เชื่อมโยงกับ `pg_class.relfilenode`
- **Page (8KB)**: หน่วยจัดเก็บและหน่วย I/O พื้นฐานที่สุดของ PostgreSQL ทุกอย่างถูกหั่นเป็น page ขนาดคงที่นี้เสมอ
- **Page layout**: Page Header + Item Pointers (เติบโตลงล่าง) + Tuple Data (เติบโตขึ้นบน) พบกันตรงกลางเป็น free space
- **Tuple Header**: `t_xmin`, `t_xmax`, `t_ctid` คือรากฐานที่แท้จริงของ MVCC ที่เคยเรียนทฤษฎีไว้ใน Part 058
- **pageinspect**: เครื่องมือมาตรฐานที่ทำให้เราตรวจสอบทุกอย่างข้างต้นได้จริงด้วย SQL ล้วน ๆ ไม่ต้องแตะ C code
- **TOAST**: กลไกที่ทำให้ column ที่มีข้อมูลใหญ่เกิน page เดียว (`TEXT`, `JSONB`, `BYTEA` ฯลฯ) ยังคงทำงานได้ ผ่านการ compress ก่อน แล้วค่อย externalize ไปยัง `pg_toast` schema
- **TOAST strategies**: `PLAIN` (ห้าม toast เลย), `MAIN` (พยายามอยู่ใน tuple ให้นานที่สุด), `EXTENDED` (default, compress+external), `EXTERNAL` (ไม่บีบอัด แต่ดึง substring ได้เร็ว)

ความรู้ในบทนี้คือรากฐานสำคัญสำหรับบทถัดไป — **WAL (Write-Ahead Log)** ซึ่งใช้ `pd_lsn` ที่เราเห็นใน page header เป็นกลไกหลักในการรับประกัน durability และ crash recovery และสำหรับบทหลังจากนั้นเรื่อง MVCC เชิงลึก ที่จะพา `t_xmin`/`t_xmax` ที่เราเพิ่งเจอไปสู่เรื่อง HOT update, VACUUM, freeze และ transaction ID wraparound แบบเต็มรูปแบบ

---

## แบบฝึกหัด

<details>
<summary><strong>แบบฝึกหัดที่ 1:</strong> อธิบายความแตกต่างระหว่าง heap-organized table (PostgreSQL) กับ index-organized table / clustered index (MySQL InnoDB) มาอย่างน้อย 3 ประเด็น</summary>

**เฉลย:**

1. **การจัดเรียงข้อมูลบนดิสก์**: Heap table ของ PostgreSQL เก็บแถวโดยไม่มีลำดับที่สัมพันธ์กับ primary key เลย ตำแหน่งขึ้นกับลำดับการ insert และ free space ที่มีอยู่ ณ ขณะนั้น ในขณะที่ InnoDB clustered index จัดเก็บทั้งแถวไว้ใน leaf node ของ B-Tree ที่เรียงตาม primary key เสมอ
2. **Secondary index lookup**: ใน PostgreSQL, secondary index จะเก็บ `ctid` (physical pointer) ชี้ไปยังตำแหน่งจริงในไฟล์ heap โดยตรง ทำให้การ lookup เป็น "index → heap page" ขั้นตอนเดียว ส่วนใน InnoDB, secondary index เก็บค่า primary key แทน `ctid` ทำให้การ lookup ต้องทำ "secondary index → PK value → clustered index lookup" ซึ่งเป็น B-Tree traversal อีกรอบ
3. **การ maintain physical order**: PostgreSQL ไม่ maintain physical order ตาม PK โดยอัตโนมัติเลย ต้องรันคำสั่ง `CLUSTER` ด้วยตนเองถ้าต้องการ (และเป็นแค่ snapshot ณ เวลานั้น ไม่ยั่งยืน) ส่วน InnoDB จะรักษาลำดับตาม PK ไว้ตลอดเวลาโดยอัตโนมัติ ซึ่งอาจทำให้เกิด page split เวลา insert แบบสุ่ม (ไม่ sequential)
4. (เสริม) **ผลต่อ range scan**: ใน PostgreSQL การ range scan บน PK ไม่รับประกันว่าจะอ่านข้อมูลติดกันบนดิสก์ (อาจต้องกระโดดไปมาหลาย page) ในขณะที่ InnoDB การ range scan บน PK มักจะอ่านข้อมูลติดกันเสมอ เพราะ physical order ตรงกับ PK order
</details>

<details>
<summary><strong>แบบฝึกหัดที่ 2:</strong> เขียน query เพื่อหาว่าตาราง `products` ถูกเก็บอยู่ที่ไฟล์ใดบน data directory และมีขนาดกี่ byte</summary>

**เฉลย:**

```sql
SELECT
    pg_relation_filepath('products') AS filepath,
    pg_relation_size('products') AS size_bytes,
    pg_size_pretty(pg_relation_size('products')) AS size_pretty;
```

ผลลัพธ์ตัวอย่าง:
```
     filepath      | size_bytes | size_pretty
--------------------+-------------+--------------
 base/16401/16404   |       90112 | 88 kB
```

`pg_relation_filepath()` คืนพาธสัมพัทธ์จาก `$PGDATA` ส่วน `pg_relation_size()` คืนขนาดของ **main fork** เท่านั้น (ไม่รวม `_fsm`, `_vm`) หากต้องการขนาดรวมทุก fork ให้ใช้ `pg_total_relation_size()` ซึ่งรวม index และ TOAST ด้วย
</details>

<details>
<summary><strong>แบบฝึกหัดที่ 3:</strong> block_size คือเท่าไหร่ และสามารถเปลี่ยนได้หรือไม่ อย่างไร</summary>

**เฉลย:**

ค่า default คือ **8192 bytes (8KB)** ตรวจสอบได้ด้วย `SHOW block_size;`

ค่านี้ **ไม่สามารถเปลี่ยนได้ด้วย `ALTER SYSTEM` หรือแก้ config file** เพราะถูกกำหนดตอน **compile PostgreSQL จาก source** ผ่านค่า `BLCKSZ` การจะเปลี่ยนต้อง `./configure --with-blocksize=N` (N เป็น 1, 2, 4, 8, 16, หรือ 32 KB) แล้ว compile ใหม่ทั้งหมด — เป็นสิ่งที่แทบไม่มีใครทำในทางปฏิบัติ เพราะ binary package มาตรฐานทุกตัว (apt, yum, Docker official image) ใช้ 8KB เป็นค่ามาตรฐานเสมอ และการเปลี่ยนค่านี้ต้อง build cluster ใหม่ทั้งหมด (ทำ `initdb` ใหม่) จะย้ายข้อมูลข้ามได้ด้วยการ dump/restore เท่านั้น
</details>

<details>
<summary><strong>แบบฝึกหัดที่ 4:</strong> ในหน้า page หนึ่ง เหตุใด Item Pointers และ Tuple Data ถึงเติบโตจากคนละทิศทาง ออกแบบมาเพื่ออะไร</summary>

**เฉลย:**

Item Pointers เติบโตจาก **บนลงล่าง** (เริ่มหลัง Page Header) ในขณะที่ Tuple Data เติบโตจาก **ล่างขึ้นบน** (เริ่มจากท้าย page) เหตุผลของการออกแบบนี้คือ:

1. **ใช้พื้นที่ตรงกลางเป็น free space ร่วมกัน** — ทั้งสองฝั่งขยายเข้าหากันได้อย่างยืดหยุ่น โดยไม่ต้องจองพื้นที่ fixed size ไว้ล่วงหน้าสำหรับแต่ละฝั่ง เมื่อ page เต็ม (`pd_lower` ชนกับ `pd_upper`) ก็จบ ไม่ต้องคำนวณซับซ้อน
2. **Item pointer มีขนาดคงที่ (4 bytes)** จึงเหมาะกับการเป็น array ต่อเนื่องกันที่นับ index ได้ง่าย (ใช้ทำ `ctid`) ในขณะที่ tuple มีขนาดแปรผัน จึงเหมาะกับการจัดวางแบบ "stack" จากท้ายมาหน้า ซึ่งง่ายต่อการจัดการ offset
3. **Indirection ทำให้ ctid คงที่แม้ tuple ข้างในย้ายที่** — เมื่อมีการทำ `VACUUM` หรือย้าย tuple ภายใน page เดียวกัน (page compaction) ระบบสามารถอัปเดตแค่ค่า offset ใน item pointer โดยไม่ต้องเปลี่ยนค่า `ctid` ที่ index อื่นชี้มา (ในกรณี line pointer เดิมยังอยู่) ช่วยลดงานในการอัปเดต index ที่เกี่ยวข้อง
</details>

<details>
<summary><strong>แบบฝึกหัดที่ 5:</strong> เขียน query ด้วย pageinspect เพื่อหาว่า block 0 ของตาราง `products` มี free space เหลือกี่ byte</summary>

**เฉลย:**

```sql
CREATE EXTENSION IF NOT EXISTS pageinspect;

SELECT
    upper - lower AS free_space_bytes
FROM page_header(get_raw_page('products', 0));
```

ผลลัพธ์ตัวอย่าง:
```
 free_space_bytes
-------------------
              6712
```

คำอธิบาย: `pd_lower` คือจุดสิ้นสุดของ item pointer array (จุดเริ่มต้นของ free space) และ `pd_upper` คือจุดเริ่มต้นของ tuple data ที่ใกล้ท้ายสุด (จุดสิ้นสุดของ free space) ดังนั้น free space = `upper - lower`
</details>

<details>
<summary><strong>แบบฝึกหัดที่ 6:</strong> ฟิลด์ `t_xmin`, `t_xmax`, `t_ctid` ใน tuple header มีความหมายว่าอะไร และเชื่อมโยงกับ MVCC อย่างไร</summary>

**เฉลย:**

- **`t_xmin`**: Transaction ID (XID) ของ transaction ที่ insert แถวนี้ขึ้นมา (หรือ transaction ที่สร้าง tuple version นี้จากการ update) — แถวจะถูกมองว่า "มีอยู่" (visible) ต่อ transaction อื่นได้ก็ต่อเมื่อ `t_xmin` นั้น commit เรียบร้อยแล้วและอยู่ในขอบเขตที่ snapshot ของผู้อ่านมองเห็นได้

- **`t_xmax`**: XID ของ transaction ที่ delete หรือ update แถวนี้ (ทำให้ version นี้ "หมดอายุ") ถ้ายังไม่เคยถูกลบ/แก้ไขเลยจะมีค่า 0 — เมื่อ `t_xmax` commit แล้วและอยู่ในขอบเขตที่ snapshot มองเห็น แถว version นี้จะถือว่า "ถูกลบไปแล้ว" สำหรับ transaction นั้น

- **`t_ctid`**: ชี้ไปยัง tuple version ถัดไปในกรณีที่แถวนี้เคยถูก update (สร้าง version ใหม่) ถ้ายังเป็น version ล่าสุด ค่านี้จะชี้กลับมาที่ตัวเอง — ใช้เดินตาม "update chain" เพื่อหา version ล่าสุดของแถวเดียวกัน

ทั้งสามฟิลด์นี้คือ implementation จริงของทฤษฎี MVCC (Multi-Version Concurrency Control) ที่ PostgreSQL ใช้: แต่ละ transaction จะเห็น "snapshot" ของข้อมูลที่ต่างกันได้ โดยเทียบ XID ของ transaction ตัวเองกับ `t_xmin`/`t_xmax` ของแต่ละ tuple version โดยไม่ต้องล็อกอ่าน (readers ไม่บล็อก writers และในทางกลับกัน)
</details>

<details>
<summary><strong>แบบฝึกหัดที่ 7:</strong> เกณฑ์ที่ทำให้ PostgreSQL เริ่มกระบวนการ TOAST คือเท่าไหร่ และทำไมถึงตั้งค่าไว้เท่านั้น</summary>

**เฉลย:**

เกณฑ์คือ **`TOAST_TUPLE_THRESHOLD` ประมาณ 2KB** (คำนวณจาก `BLCKSZ / 4` = 8192/4 = 2048 bytes โดยประมาณ ลบด้วย overhead บางส่วน) — เมื่อ tuple ทั้งแถว (รวมทุก column) มีขนาดเกินเกณฑ์นี้ PostgreSQL จะเริ่มพยายาม TOAST column ที่เป็น varlena type ที่ใหญ่ที่สุดก่อน

เหตุผลของค่า **1/4 ของ page size**: PostgreSQL ต้องการรับประกันว่า **อย่างน้อยเก็บได้ 4 tuple ต่อ 1 page** เพื่อไม่ให้ page ถูกครอบครองโดยแถวเดียวหรือสองแถว ซึ่งจะทำให้ประสิทธิภาพของระบบ page-based storage (การจัดการ free space, การทำ index, การ cache ใน shared_buffers) แย่ลงมาก ถ้าอนุญาตให้แถวเดียวใหญ่เกือบเต็ม page ก็จะเสียพื้นที่ไปกับ overhead ต่าง ๆ อย่างไม่คุ้มค่า
</details>

<details>
<summary><strong>แบบฝึกหัดที่ 8:</strong> อธิบายความแตกต่างระหว่าง TOAST strategy `EXTENDED` และ `EXTERNAL` และยกตัวอย่างสถานการณ์ที่ควรเลือกใช้ `EXTERNAL`</summary>

**เฉลย:**

| | EXTENDED | EXTERNAL |
|---|---|---|
| Compression | บีบอัดก่อนเสมอ (ถ้าลดขนาดได้) | ไม่บีบอัดเลย |
| Out-of-line | ย้ายไป TOAST table ถ้าจำเป็นหลัง compress แล้วยังใหญ่ | ย้ายไป TOAST table ทันทีถ้าเกิน threshold โดยไม่บีบอัด |
| ข้อดี | ประหยัดพื้นที่ดิสก์มากกว่า (ถ้าข้อมูลบีบอัดได้) | ดึงข้อมูลบางส่วน (substring) เร็วกว่า เพราะไม่ต้อง decompress ทั้งก้อนก่อน |
| ข้อเสีย | การดึง substring ต้อง decompress ทั้งค่าก่อนเสมอ แม้จะต้องการแค่บางส่วน | ใช้พื้นที่ดิสก์มากกว่าถ้าข้อมูลบีบอัดได้ดี |

**สถานการณ์ที่ควรใช้ EXTERNAL**: column ที่เก็บข้อมูลขนาดใหญ่ซึ่ง **query บ่อยด้วย `substring()`, `left()`, หรือ operator ที่ดึงแค่บางส่วน** และข้อมูลนั้น **บีบอัดได้ไม่ดีอยู่แล้ว** เช่น ข้อมูลที่ผ่านการ encrypt มาแล้ว (entropy สูง บีบอัดแทบไม่ได้), ไฟล์ binary ที่บีบอัดมาแล้ว (เช่น JPEG, ZIP ที่เก็บใน BYTEA), หรือ log ขนาดใหญ่ที่ต้องดึงเฉพาะบรรทัดแรก ๆ บ่อย ๆ — ในกรณีเหล่านี้ compression ไม่ได้ช่วยประหยัดพื้นที่มากนัก แต่กลับทำให้ partial read ช้าลงถ้าใช้ EXTENDED
</details>

<details>
<summary><strong>แบบฝึกหัดที่ 9:</strong> เขียน query เพื่อหาว่าตาราง `products` มี TOAST table ชื่ออะไร และมีข้อมูลอยู่กี่ chunk รวมทั้งหมดกี่ byte</summary>

**เฉลย:**

```sql
-- ขั้น 1: หาชื่อ TOAST table
SELECT
    c.relname AS main_table,
    t.relname AS toast_table
FROM pg_class c
JOIN pg_class t ON c.reltoastrelid = t.oid
WHERE c.relname = 'products';
```

```
 main_table | toast_table
------------+------------------
 products   | pg_toast_16404
```

```sql
-- ขั้น 2: นับ chunk และรวม byte (แทนชื่อ TOAST table ที่ได้จากขั้น 1)
SELECT
    count(*) AS total_chunks,
    count(DISTINCT chunk_id) AS distinct_toasted_values,
    sum(octet_length(chunk_data)) AS total_bytes_stored
FROM pg_toast.pg_toast_16404;
```

```
 total_chunks | distinct_toasted_values | total_bytes_stored
---------------+---------------------------+----------------------
             4 |                         2 |                 5432
```

หรือรวมเป็น query เดียวด้วย dynamic SQL/CTE โดยใช้ `reltoastrelid` โดยตรงก็ได้ในกรณีที่ต้องการทำเป็น script อัตโนมัติ
</details>

<details>
<summary><strong>แบบฝึกหัดที่ 10:</strong> ทำไม query <code>SELECT product_id, unit_price FROM products WHERE product_id = 501</code> ถึงยังคงเร็ว แม้ว่าแถวนั้นจะมี description ขนาด 5KB ที่ถูก TOAST ไปแล้ว จงอธิบายด้วยความเข้าใจเรื่อง page layout และ TOAST</summary>

**เฉลย:**

เพราะ query นี้**ไม่ได้เลือก column `description` เลย** ระบบจึงไม่จำเป็นต้อง dereference TOAST pointer หรือไปอ่าน TOAST table เลยแม้แต่น้อย

อธิบายเป็นขั้นตอน:

1. Planner ใช้ index บน `product_id` (primary key) หาตำแหน่ง `ctid` ของแถวที่ `product_id = 501`
2. ระบบอ่าน heap page ที่ `ctid` ชี้ไป (1 page I/O จาก main table)
3. ใน main tuple นั้น column `description` ถูกเก็บเป็นแค่ **TOAST pointer ขนาด 18 bytes** (ไม่ใช่ข้อมูลจริง 5KB) — ดังนั้น tuple ทั้งก้อนที่ต้องอ่านจาก page มีขนาดเล็กมาก ไม่ว่า description จะยาวแค่ไหนก็ตาม
4. เนื่องจาก query เลือกแค่ `product_id, unit_price` ซึ่งเป็น column ธรรมดา (ไม่ใช่ description) ระบบดึงค่าจาก tuple ที่มีอยู่แล้วในหน่วยความจำได้ทันที **โดยไม่ต้อง dereference TOAST pointer ไปเปิด TOAST table เพิ่มเลย**
5. ถ้า query มีการเลือก `description` ด้วย (เช่น `SELECT *`) ถึงจะต้องมีการอ่านเพิ่มจาก `pg_toast.pg_toast_16404` (ผ่าน index บน `chunk_id, chunk_seq`) เพื่อประกอบร่างข้อมูลกลับมา ซึ่งจะมี I/O เพิ่มขึ้นตามจำนวน chunk

นี่คือเหตุผลเชิง design ที่สำคัญมาก: **การแยก TOAST ออกจาก main tuple ทำให้ query ที่ไม่แตะ column ใหญ่ ไม่ต้องจ่าย cost ของมันเลย** ซึ่งเป็นหลักการออกแบบที่ทำให้ PostgreSQL รองรับ column ขนาดใหญ่ (TEXT, JSONB ฯลฯ) ได้โดยไม่กระทบ performance ของ query ทั่วไปที่ไม่เกี่ยวข้องกับ column เหล่านั้น
</details>

---

**บทถัดไป:** [Part 083 — WAL Deep Dive](./part-083-wal-deep-dive.md)
