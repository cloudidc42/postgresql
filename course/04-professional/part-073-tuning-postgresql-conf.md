# Performance Tuning: postgresql.conf ระดับ Production

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับมืออาชีพ | Part 073

---

## เป้าหมายการเรียนรู้

เมื่อจบบทนี้ ผู้เรียนจะสามารถ:

1. อธิบายได้ว่าทำไม PostgreSQL จึงไม่ตั้งค่า default ให้ "เร็วที่สุด" ตั้งแต่แรก และเหตุใดการ tune ต้องอิงจาก workload จริง ไม่ใช่สูตรสำเร็จ
2. คำนวณและตั้งค่า `shared_buffers`, `effective_cache_size`, `work_mem`, `maintenance_work_mem` ให้เหมาะสมกับขนาด RAM ของเซิร์ฟเวอร์และจำนวน concurrent connection
3. ปรับจูน WAL และ checkpoint (`wal_buffers`, `checkpoint_timeout`, `checkpoint_completion_target`) สำหรับงานที่เขียนข้อมูลหนัก (write-heavy)
4. กำหนด `max_connections` อย่างสมเหตุสมผล และเข้าใจว่าทำไมต้องพึ่ง connection pooling แทนการเพิ่มค่านี้แบบไม่จำกัด
5. เลือกค่า `random_page_cost` และ `effective_io_concurrency` ให้เหมาะกับประเภท storage (HDD, SSD, NVMe, cloud block storage)
6. ตั้งค่าพารามิเตอร์ parallel query (`max_parallel_workers_per_gather`, `max_worker_processes` ฯลฯ) ให้ใช้ CPU หลายคอร์อย่างมีประสิทธิภาพ
7. เขียนไฟล์ `postgresql.conf` ที่ใช้งานได้จริงสำหรับเซิร์ฟเวอร์ขนาด 8GB, 32GB และ 128GB RAM สำหรับระบบ e-commerce

---

## บทนำ: เชื่อมโยงกับสิ่งที่เรียนมา

ใน Part 066 เราเรียนเรื่อง connection pooling (PgBouncer/pgcat) ไปแล้ว ซึ่งเป็นเลเยอร์ที่อยู่ "หน้า" PostgreSQL บทนี้จะเจาะลึกเข้าไปที่ตัว PostgreSQL เองผ่านไฟล์ `postgresql.conf` — ไฟล์ config หลักที่ควบคุมพฤติกรรมเกือบทุกด้านของฐานข้อมูล ตั้งแต่การใช้หน่วยความจำ การเขียน WAL ไปจนถึงการวางแผน query

ก่อนเริ่ม เรามาดูกันก่อนว่าไฟล์นี้อยู่ที่ไหน และแก้ไขอย่างไร:

```sql
-- หาตำแหน่งไฟล์ postgresql.conf ที่ใช้งานจริง
SHOW config_file;

-- หาตำแหน่ง data directory
SHOW data_directory;

-- ดูค่าพารามิเตอร์ทั้งหมดที่ไม่ใช่ default
SELECT name, setting, unit, source
FROM pg_settings
WHERE source NOT IN ('default', 'override')
ORDER BY name;
```

ผลลัพธ์ตัวอย่าง:

```
        config_file
-----------------------------------------
 /etc/postgresql/16/main/postgresql.conf
```

หลังแก้ไขไฟล์ `postgresql.conf` แล้ว พารามิเตอร์ส่วนใหญ่ต้อง **restart** service (บางตัว reload ได้ด้วย `SELECT pg_reload_conf();` หรือ `pg_ctl reload`) เราจะระบุไว้ทุกพารามิเตอร์ว่าต้อง restart หรือ reload พอ

```sql
-- ตรวจสอบว่าพารามิเตอร์ใดต้อง restart, reload, หรือแก้ได้แบบ session
SELECT name, context, unit, short_desc
FROM pg_settings
WHERE name IN ('shared_buffers', 'work_mem', 'max_connections', 'wal_buffers')
ORDER BY name;
```

```
       name        | context |  unit  |               short_desc
--------------------+---------+--------+------------------------------------------
 max_connections    | postmaster | (null) | Sets the maximum number of concurrent...
 shared_buffers     | postmaster | 8kB    | Sets the number of shared memory buffers...
 wal_buffers        | postmaster | 8kB    | Sets the number of disk-page buffers...
 work_mem           | user       | kB     | Sets the maximum memory to be used for...
```

- `context = postmaster` → ต้อง **restart PostgreSQL** เท่านั้น (shared_buffers, max_connections, wal_buffers)
- `context = sighup` → `pg_ctl reload` หรือ `SELECT pg_reload_conf();` พอ (checkpoint_timeout, random_page_cost)
- `context = user` → เปลี่ยนได้ระดับ session/query ด้วย `SET` (work_mem, effective_cache_size)

---

## Step 721: ภาพรวมการ Tune PostgreSQL — ไม่มีค่าเดียวที่เหมาะกับทุกงาน

### ทำไม default ของ PostgreSQL ถึง "ช้า"

ค่า default ใน `postgresql.conf` ที่มาพร้อมการติดตั้งใหม่ถูกออกแบบมาให้รันได้บนเครื่องที่มี**ทรัพยากรจำกัดมาก** (เช่น container เล็ก ๆ หรือเครื่องพัฒนาที่มี RAM 128MB) เพื่อให้ PostgreSQL "start ติด" บนทุกสภาพแวดล้อม ไม่ใช่เพราะเป็นค่าที่แนะนำสำหรับ production

ตัวอย่างค่า default ที่มักทำให้มือใหม่งงว่าทำไม PostgreSQL "ช้า":

```sql
SHOW shared_buffers;        -- 128MB (default)
SHOW work_mem;               -- 4MB (default)
SHOW effective_cache_size;   -- 4GB (default)
SHOW maintenance_work_mem;   -- 64MB (default)
```

บนเซิร์ฟเวอร์ production ที่มี RAM 32GB ค่าเหล่านี้ **ต่ำเกินไปมาก** — PostgreSQL จะใช้ RAM เพียงเสี้ยวเดียวของเครื่อง แล้วต้องพึ่งการอ่านดิสก์บ่อยกว่าที่ควร

### หลักการสำคัญ: Tune ตาม Workload ไม่ใช่ตามสูตรตายตัว

ก่อนแตะพารามิเตอร์ใด ๆ ต้องตอบคำถามเหล่านี้ให้ได้ก่อน:

| คำถาม | เหตุผลที่สำคัญ |
|---|---|
| เซิร์ฟเวอร์นี้มี RAM เท่าไหร่ และแบ่งใช้กับ service อื่นหรือไม่ | กำหนดเพดานของ shared_buffers, work_mem รวม |
| Storage เป็น HDD, SSD, NVMe หรือ cloud block storage (EBS/PD) | กำหนด random_page_cost, effective_io_concurrency |
| จำนวน CPU core | กำหนด max_parallel_workers, max_worker_processes |
| เป็น OLTP (transaction สั้น จำนวนมาก) หรือ OLAP/Analytics (query ซับซ้อน อ่านหนัก) หรือ Mixed | กำหนดสัดส่วน work_mem, checkpoint tuning |
| จำนวน concurrent connection สูงสุดที่คาดไว้ (หรือมี connection pooler อยู่แล้วหรือไม่) | กำหนด max_connections และ work_mem ต่อ connection |
| อัตราการเขียน (write-heavy) เช่น order, payment, log vs อ่านหนัก (read-heavy) เช่น catalog, report | กำหนด checkpoint, WAL tuning |
| ต้องทน downtime ได้แค่ไหน (RTO/RPO) | กระทบ checkpoint_timeout, wal tuning (เชื่อมโยง Part 061-062) |

> **แนวคิดหลักของบทนี้**: การ tune ที่ดีไม่ใช่การ "copy ค่าจาก blog มาวาง" แต่คือการเข้าใจว่าพารามิเตอร์แต่ละตัวควบคุมพฤติกรรมอะไร แล้วคำนวณค่าจากสเปคเครื่องและลักษณะ workload จริงของเรา

### เครื่องมือช่วยคิด: PGTune แนวคิด

เว็บไซต์อย่าง PGTune ใช้สูตรคำนวณจาก RAM/CPU/ประเภท workload เพื่อ generate ค่าตั้งต้น — เราจะเรียนรู้ "สูตรเบื้องหลัง" เหล่านั้นในบทนี้ เพื่อให้ปรับแต่งเองได้โดยไม่ต้องพึ่งเครื่องมือภายนอก

### ลำดับความสำคัญของการ Tune (Priority Order)

จากประสบการณ์จริงของ DBA ระดับ production เรียงลำดับผลกระทบจากมากไปน้อย:

1. **shared_buffers + effective_cache_size** — ผลกระทบต่อ cache hit ratio โดยตรง
2. **work_mem** — ป้องกัน query ที่ sort/hash แล้ว spill ลงดิสก์ (temp file)
3. **checkpoint tuning** — ลด I/O spike และ WAL replay time
4. **random_page_cost / effective_io_concurrency** — ทำให้ query planner เลือก index scan อย่างถูกต้อง
5. **max_connections + pooling** — ป้องกัน memory exhaustion และ context-switch overhead
6. **parallel query** — เร่งความเร็ว query ขนาดใหญ่บนเครื่องหลายคอร์
7. **maintenance_work_mem** — เร่ง VACUUM/CREATE INDEX (impact ต่อ user query น้อยกว่าตัวอื่น)

บทนี้จะไล่เรียงพารามิเตอร์เหล่านี้ทีละตัว พร้อมสูตรคำนวณและตัวอย่าง config จริง

### เช็คสเปคเครื่องก่อนเริ่ม tune

```bash
# RAM ทั้งหมด
free -h

# จำนวน CPU core
nproc

# ประเภท storage (ตรวจสอบว่าเป็น SSD/NVMe หรือ rotational HDD)
lsblk -d -o name,rota
# rota=0 คือ SSD/NVMe, rota=1 คือ HDD แบบจาน
```

```sql
-- ตรวจสอบเวอร์ชันและ platform ของ PostgreSQL ที่กำลังรัน
SELECT version();

-- จำนวน CPU ที่ PostgreSQL "เห็น" (ต้องอ่านจาก OS เอง ไม่มีฟังก์ชันในตัว)
SHOW max_worker_processes;
```

---

## Step 722: shared_buffers — พื้นที่ Cache หลักของ PostgreSQL

### shared_buffers คืออะไร

`shared_buffers` คือหน่วยความจำที่ PostgreSQL จองไว้ใน shared memory เพื่อเก็บ **หน้าข้อมูล (page)** ของตาราง/index ที่เพิ่งถูกอ่านหรือเขียน แทนที่จะต้องไปอ่านจากดิสก์ทุกครั้ง เมื่อ query ต้องการ page ที่อยู่ใน shared_buffers อยู่แล้ว (cache hit) จะเร็วกว่าการไปอ่านจากดิสก์ (cache miss) มาก

```sql
-- ดูค่าปัจจุบัน (unit เป็น 8kB block แต่แสดงผลเป็น MB/GB)
SHOW shared_buffers;

-- ดู cache hit ratio ของฐานข้อมูลปัจจุบัน (ควร > 99% สำหรับ OLTP)
SELECT
    sum(heap_blks_read) AS heap_read,
    sum(heap_blks_hit)  AS heap_hit,
    round(
        sum(heap_blks_hit)::numeric /
        nullif(sum(heap_blks_hit) + sum(heap_blks_read), 0) * 100,
    2) AS cache_hit_ratio_pct
FROM pg_statio_user_tables;
```

ผลลัพธ์ตัวอย่าง:

```
 heap_read | heap_hit  | cache_hit_ratio_pct
-----------+-----------+----------------------
    182340 | 98213847  |                99.82
```

ถ้า `cache_hit_ratio_pct` ต่ำกว่า 95% อย่างต่อเนื่อง มักเป็นสัญญาณว่า `shared_buffers` เล็กเกินไปเมื่อเทียบกับขนาด working set (ส่วนของข้อมูลที่ถูก query บ่อย)

### สูตรคำนวณมาตรฐาน

> **shared_buffers ≈ 25% ของ RAM ทั้งเครื่อง** (สำหรับเซิร์ฟเวอร์ที่รัน PostgreSQL เป็น service หลัก)

| RAM เครื่อง | shared_buffers แนะนำ |
|---|---|
| 4 GB | 1 GB |
| 8 GB | 2 GB |
| 16 GB | 4 GB |
| 32 GB | 8 GB |
| 64 GB | 16 GB |
| 128 GB | 32 GB (ไม่ควรเกิน 32-40GB แม้ RAM จะมากกว่านี้) |
| 256 GB+ | 32-40 GB (เพดาน ไม่ใช่ 25% อีกต่อไป) |

### ทำไมถึงเป็น 25% ไม่ใช่ 80-90%

หลายคนเข้าใจผิดว่า "PostgreSQL คือฐานข้อมูล ยิ่งให้ RAM cache มากยิ่งดี" แต่ความจริงมีเหตุผลทางเทคนิคที่ทำให้ 25% เป็นจุดสมดุลที่ดีที่สุด:

1. **PostgreSQL พึ่งพา OS page cache ร่วมด้วย** — ต่างจาก Oracle/SQL Server ที่จัดการ buffer cache เองแบบเบ็ดเสร็จ PostgreSQL ใช้ `write()`/`read()` ผ่าน OS filesystem ปกติ ทำให้ข้อมูลถูก cache ซ้ำทั้งใน shared_buffers และ OS page cache (double buffering) การให้ shared_buffers มากเกินไปจึงเป็นการ "แย่งพื้นที่" จาก OS cache ที่ทำงานร่วมกันอยู่แล้ว โดยไม่ได้ประโยชน์เพิ่ม
2. **Checkpoint ที่มี dirty page ใน shared_buffers มากจะเขียนดิสก์หนักขึ้น** — buffer ยิ่งใหญ่ ยิ่งมี dirty page สะสมมาก เมื่อถึงรอบ checkpoint ต้อง flush ลงดิสก์ทีเดียวจำนวนมาก ทำให้เกิด I/O spike
3. **shared_buffers ที่ใหญ่เกินไปทำให้ PostgreSQL restart ช้าลง** (ต้อง initialize shared memory) และในบาง OS/kernel การจัดการ shared memory ขนาดใหญ่มาก ๆ (>40GB) มี overhead จาก TLB miss และ lock contention ภายใน buffer manager เอง

### ข้อยกเว้นของกฎ 25%

| สถานการณ์ | คำแนะนำ |
|---|---|
| RAM เครื่องใหญ่มาก (256GB+) | อย่าเกิน ~40GB เพดานจริง ให้ใช้ RAM ส่วนที่เหลือเป็น OS cache ผ่าน effective_cache_size แทน |
| เซิร์ฟเวอร์ dedicated PostgreSQL 100% (ไม่มี service อื่นแชร์ RAM) | อาจขยับเป็น 30-40% ได้ถ้า workload เป็น read-heavy และ working set ใหญ่กว่า 25% ของ RAM มาก |
| แชร์เครื่องกับ application server หรือ service อื่น | ลดเหลือ 15-20% เพื่อเผื่อ RAM ให้ service อื่น |
| ใช้ container ที่มี memory limit (Docker/K8s) | คำนวณจาก memory limit ของ container ไม่ใช่ RAM ของ host node |
| Workload เป็น bulk load / ETL ที่ write มหาศาลครั้งเดียว | shared_buffers ที่เล็กกว่าอาจดีกว่า เพราะลดปัญหา checkpoint I/O spike |

### วิธีตั้งค่า

```ini
# postgresql.conf
# ตัวอย่างเครื่อง 32GB RAM, dedicated PostgreSQL server
shared_buffers = 8GB
```

```sql
-- ตั้งค่าผ่าน ALTER SYSTEM (เขียนไปที่ postgresql.auto.conf) แล้ว restart
ALTER SYSTEM SET shared_buffers = '8GB';
-- ต้อง restart เพราะ context = postmaster
```

```bash
sudo systemctl restart postgresql
```

```sql
-- ตรวจสอบหลัง restart
SHOW shared_buffers;
```

> **คำเตือน**: `shared_buffers` เป็น `context = postmaster` เปลี่ยนแล้วต้อง restart เสมอ ต่างจาก `work_mem` ที่ reload หรือแม้แต่ SET ระดับ session ได้

---

## Step 723: effective_cache_size — บอก Planner ว่า OS Cache มีเท่าไหร่

### ความเข้าใจผิดที่พบบ่อยที่สุด

`effective_cache_size` **ไม่ได้จองหน่วยความจำจริง** — มันเป็นเพียง "คำใบ้" (hint) ที่บอก query planner ว่า "โดยรวมแล้ว เครื่องนี้น่าจะมีหน่วยความจำสำหรับ cache ข้อมูล (shared_buffers + OS page cache) ประมาณเท่านี้" เพื่อให้ planner ตัดสินใจได้แม่นยำขึ้นว่า index scan หรือ sequential scan ตัวไหนจะเร็วกว่าในทางปฏิบัติ

```sql
SHOW effective_cache_size;
```

ค่า default มักอยู่ที่ 4GB ซึ่งอาจต่ำหรือสูงเกินจริงมากเมื่อเทียบกับเครื่อง production

### ทำไมค่านี้ถึงสำคัญต่อ Query Plan

ถ้า `effective_cache_size` ตั้งค่าต่ำเกินไป planner จะ "คิดว่า" ข้อมูลส่วนใหญ่ต้องอ่านจากดิสก์เสมอ จึงมักเลือก **sequential scan** แทน **index scan** แม้ในสถานการณ์จริงข้อมูลจะถูก cache อยู่แล้วเกือบทั้งหมด (เพราะ RAM เหลือเฟือ)

ลองทดสอบผลกระทบ:

```sql
-- ตั้งค่าต่ำเกินจริงเพื่อดูผล (ทดสอบใน session เดียว ไม่กระทบ production)
SET effective_cache_size = '128MB';

EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders WHERE customer_id = 12345;
```

```
Seq Scan on orders  (cost=0.00..45230.00 rows=8 width=120)
                     (actual time=0.045..312.884 rows=6 loops=1)
  Filter: (customer_id = 12345)
  Rows Removed by Filter: 4999994
```

```sql
-- ตั้งค่าให้สมจริงกับเครื่อง (32GB RAM)
SET effective_cache_size = '24GB';

EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders WHERE customer_id = 12345;
```

```
Index Scan using idx_orders_customer_id on orders
                     (cost=0.43..8.45 rows=8 width=120)
                     (actual time=0.021..0.028 rows=6 loops=1)
  Index Cond: (customer_id = 12345)
```

จะเห็นว่า planner เปลี่ยนจาก Seq Scan → Index Scan เมื่อ effective_cache_size สมจริงขึ้น เพราะมันเชื่อว่าการอ่าน index page ซ้ำ ๆ จะ "โดน cache" ไม่ต้องไปอ่านดิสก์จริงทุกครั้ง

### สูตรคำนวณ

> **effective_cache_size ≈ 50-75% ของ RAM ทั้งเครื่อง**

แนวคิด: shared_buffers (25%) + OS page cache ที่เหลือจาก RAM ที่ไม่ได้ใช้งานอื่น ๆ (ประมาณ 25-50% เพิ่มเติม) รวมกันแล้วมักอยู่ที่ 50-75% ของ RAM ทั้งหมด

| RAM เครื่อง | shared_buffers (25%) | effective_cache_size (60-70%) |
|---|---|---|
| 8 GB | 2 GB | 5 GB |
| 16 GB | 4 GB | 10-11 GB |
| 32 GB | 8 GB | 22-24 GB |
| 64 GB | 16 GB | 44-46 GB |
| 128 GB | 32 GB | 88-96 GB |

### วิธีตั้งค่า

```ini
# postgresql.conf — เครื่อง 32GB RAM
effective_cache_size = 24GB
```

```sql
-- ปรับได้แบบ reload ไม่ต้อง restart (context = user แต่จริง ๆ มักตั้งระดับ system)
ALTER SYSTEM SET effective_cache_size = '24GB';
SELECT pg_reload_conf();

SHOW effective_cache_size;
```

> **ข้อสังเกต**: ต่างจาก `shared_buffers` ตรงที่ `effective_cache_size` เป็น `context = user` เปลี่ยนได้ทันทีด้วย reload หรือแม้แต่ `SET` ระดับ session เพื่อทดลองผลกระทบต่อ query plan ก่อนตั้งเป็นค่าถาวร

---

## Step 724: work_mem — หน่วยความจำต่อ Sort/Hash Operation

### work_mem คืออะไร

`work_mem` กำหนดหน่วยความจำสูงสุดที่แต่ละ **operation** (เช่น `ORDER BY`, `DISTINCT`, `hash join`, `hash aggregate`, `merge join`) สามารถใช้ได้ก่อนที่จะต้อง spill ข้อมูลลง **temp file บนดิสก์**

```sql
SHOW work_mem;
```

Default มักอยู่ที่ 4MB ซึ่งเล็กมากสำหรับ query ที่ sort ข้อมูลจำนวนมาก

### ข้อสำคัญ: work_mem คูณด้วยจำนวน operation ไม่ใช่จำนวน connection

นี่คือจุดที่คน tune ผิดพลาดบ่อยที่สุด — `work_mem` **ไม่ได้ถูกจองแค่ครั้งเดียวต่อ connection** แต่ query เดียวอาจมีหลาย sort/hash operation พร้อมกัน (เช่น JOIN 3 ตารางที่แต่ละ join ใช้ hash table ของตัวเอง, subquery ที่มี ORDER BY ซ้อนกัน) และถ้ามี parallel worker แต่ละ worker ก็ใช้ work_mem ของตัวเองเพิ่มอีก

```
หน่วยความจำสูงสุดที่ query หนึ่งอาจใช้ ≈ work_mem × จำนวน sort/hash node ใน plan × (1 + จำนวน parallel worker)
```

ลองดูตัวอย่าง query ที่ซับซ้อน:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT c.customer_name, o.order_date, SUM(oi.quantity * oi.unit_price) AS total
FROM customers c
JOIN orders o ON o.customer_id = c.customer_id
JOIN order_items oi ON oi.order_id = o.order_id
WHERE o.order_date >= '2026-01-01'
GROUP BY c.customer_name, o.order_date
ORDER BY total DESC
LIMIT 100;
```

```
Limit  (cost=125430.12..125430.37 rows=100 width=48)
  ->  Sort  (cost=125430.12..126890.45 rows=584132 width=48)
        Sort Key: (sum((oi.quantity * oi.unit_price))) DESC
        ->  HashAggregate  (cost=98234.00..104075.32 rows=584132 width=48)
              Group Key: c.customer_name, o.order_date
              ->  Hash Join  (cost=15230.00..85430.00 rows=1200000 width=32)
                    Hash Cond: (o.order_id = oi.order_id)
                    ->  Hash Join  (cost=520.00..45230.00 rows=1200000 width=24)
                          Hash Cond: (o.customer_id = c.customer_id)
                          ->  Seq Scan on orders o ...
                          ->  Hash  (cost=380.00..380.00 rows=50000 width=16)
                                ->  Seq Scan on customers c ...
                    ->  Hash  (cost=12000.00..12000.00 rows=1200000 width=16)
                          ->  Seq Scan on order_items oi ...
```

Query นี้มี **Sort 1 ตัว + HashAggregate 1 ตัว + Hash Join 2 ตัว** = 4 operation ที่อาจใช้ work_mem แยกกัน ถ้า `work_mem = 4MB` แต่ operation แต่ละตัวต้องการมากกว่านั้น จะเกิด temp file

### วิธีตรวจว่า work_mem เล็กเกินไป (temp file spill)

```sql
-- ดู log ว่ามี temp file เกิดขึ้นหรือไม่ (ต้องเปิด log_temp_files ก่อน)
SHOW log_temp_files;

-- แนะนำให้ตั้งเป็น 0 เพื่อ log ทุก temp file (production ตั้ง 10MB ขึ้นไปเพื่อลด noise)
ALTER SYSTEM SET log_temp_files = '10MB';
SELECT pg_reload_conf();
```

จากนั้นดูใน log ว่ามีบรรทัดแบบนี้หรือไม่:

```
LOG:  temporary file: path "base/pgsql_tmp/pgsql_tmp12345.0", size 45238272
STATEMENT:  SELECT ... ORDER BY total DESC ...
```

หรือดูผ่าน `EXPLAIN (ANALYZE, BUFFERS)` โดยตรง — ถ้าเห็นบรรทัด `Sort Method: external merge  Disk: 45230kB` แปลว่า sort นั้น spill ลงดิสก์เพราะ work_mem ไม่พอ:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM order_items ORDER BY unit_price DESC;
```

```
Sort  (cost=125430.00..127930.00 rows=1200000 width=32)
      (actual time=850.123..920.456 rows=1200000 loops=1)
  Sort Key: unit_price DESC
  Sort Method: external merge  Disk: 45230kB
  ->  Seq Scan on order_items ...
```

ถ้าเปรียบเทียบกับกรณีที่ work_mem พอ (`Sort Method: quicksort  Memory: 25600kB`) จะเห็นว่า external merge ที่ใช้ disk ช้ากว่า quicksort บน memory มาก

### สูตรคำนวณ work_mem

> **work_mem ≈ (RAM ทั้งหมด × 0.25) ÷ (max_connections × ค่าเฉลี่ยจำนวน operation ต่อ query)**

สูตรที่ใช้งานได้จริงและปลอดภัยกว่า (ระมัดระวังไม่ให้ RAM หมด):

```
work_mem = (Total RAM - shared_buffers - OS reserve) / (max_connections × avg_ops_per_query)
```

โดยทั่วไปใช้ตัวเลขประมาณการอนุรักษ์นิยม `avg_ops_per_query = 2-3` (สมมติแต่ละ query มี sort/hash operation พร้อมกันโดยเฉลี่ย 2-3 ตัว)

**ตัวอย่างคำนวณ** เครื่อง 32GB RAM, shared_buffers = 8GB, max_connections = 200, connection pooling ทำให้ concurrent active query จริง ๆ ประมาณ 50 (ไม่ใช่ 200 เต็ม):

```
พื้นที่ที่เหลือสำหรับ work_mem ≈ 32GB - 8GB (shared_buffers) - 2GB (OS reserve) = 22GB
work_mem = 22GB / (50 concurrent × 3 ops) ≈ 22GB / 150 ≈ 150MB
```

แต่ในทางปฏิบัติ ค่าที่ปลอดภัยและใช้กันทั่วไปคือช่วง **4-64MB** สำหรับ OLTP ทั่วไป และสามารถตั้งสูงขึ้น (128MB-1GB) เฉพาะ session ที่รัน analytics/report

| ลักษณะ workload | work_mem แนะนำ (global) |
|---|---|
| OLTP, connection สูง (100+), มี pooler | 4-16 MB |
| OLTP ทั่วไป, connection ปานกลาง (20-50) | 16-64 MB |
| Mixed OLTP + reporting เบา | 32-128 MB |
| Analytics/OLAP, connection น้อย (< 20), query ซับซ้อน | 256MB - 1GB (ตั้งเฉพาะ session) |

### ผลกระทบถ้าตั้งค่าผิด

| กรณี | ผลกระทบ |
|---|---|
| work_mem **ต่ำเกินไป** | Sort/hash spill ลง disk (temp file) → query ช้าลงมาก, I/O เพิ่ม, disk เต็มได้ถ้า concurrent สูง |
| work_mem **สูงเกินไป** | หลาย query รันพร้อมกัน แต่ละตัวจอง work_mem เต็มเพดาน → **RAM หมด (OOM)** → OS kill process → PostgreSQL crash/restart ทั้งเซิร์ฟเวอร์ |

> **กฎเหล็ก**: อย่าคิดแค่ `RAM ÷ max_connections` เพราะลืมคูณด้วยจำนวน operation ต่อ query — นี่คือสาเหตุอันดับ 1 ที่ทำให้ production database ล่มจาก OOM หลังคนตั้ง work_mem สูงเกินไปโดยไม่คิดเผื่อ concurrent query ที่ซับซ้อน

### วิธีตั้งค่า

```ini
# postgresql.conf — ค่า global สำหรับ OLTP ทั่วไป
work_mem = 32MB
```

```sql
ALTER SYSTEM SET work_mem = '32MB';
SELECT pg_reload_conf();
```

**ตั้งเฉพาะ session สำหรับ query หนัก** (แนะนำมากกว่าการเพิ่ม global):

```sql
-- ก่อนรัน report ที่ sort ข้อมูลจำนวนมาก
BEGIN;
SET LOCAL work_mem = '512MB';
SELECT customer_id, SUM(amount) FROM orders
GROUP BY customer_id
ORDER BY SUM(amount) DESC;
COMMIT;
-- work_mem กลับเป็นค่า global อัตโนมัติหลัง COMMIT/ROLLBACK
```

การใช้ `SET LOCAL` ภายใน transaction เป็นวิธีที่ปลอดภัยที่สุดในการให้ query เฉพาะบางตัวใช้ work_mem สูง โดยไม่กระทบ query อื่นที่รันพร้อมกัน

---

## Step 725: maintenance_work_mem — หน่วยความจำสำหรับ VACUUM, CREATE INDEX, ALTER TABLE

### maintenance_work_mem คืออะไร

พารามิเตอร์นี้กำหนดหน่วยความจำสูงสุดสำหรับ**งานบำรุงรักษา (maintenance)** ได้แก่:

- `VACUUM` (โดยเฉพาะการเก็บ dead tuple ID ก่อน vacuum index)
- `CREATE INDEX` / `REINDEX`
- `ALTER TABLE ... ADD FOREIGN KEY` (ตรวจสอบข้อมูลที่มีอยู่)
- `pg_restore` ตอนสร้าง index ใหม่

```sql
SHOW maintenance_work_mem;
```

Default 64MB มักน้อยเกินไปสำหรับตารางขนาดใหญ่ระดับ production

### ทำไมค่านี้ควรสูงกว่า work_mem มาก

ต่างจาก `work_mem` ที่ต้องคูณด้วยจำนวน concurrent operation, `maintenance_work_mem` มักใช้งานพร้อมกันได้จำนวนจำกัด (เพราะ autovacuum มี worker จำกัด และ DBA มักไม่รัน CREATE INDEX พร้อมกันหลายสิบตัว) จึงสามารถตั้งค่าให้สูงกว่า work_mem ได้มาก โดยไม่เสี่ยง OOM เท่า work_mem

### สูตรคำนวณ

> **maintenance_work_mem ≈ 5-10% ของ RAM** (เพดานที่นิยมใช้คือไม่เกิน 1-2GB สำหรับงานทั่วไป และเพิ่มเป็น session-level สำหรับงานสร้าง index ขนาดใหญ่)

| RAM เครื่อง | maintenance_work_mem แนะนำ |
|---|---|
| 8 GB | 256-512 MB |
| 16 GB | 512MB - 1GB |
| 32 GB | 1-2 GB |
| 64 GB | 2 GB |
| 128 GB | 2-4 GB |

### ความสัมพันธ์กับ autovacuum_max_workers

ข้อควรระวัง: `maintenance_work_mem` คูณกับ `autovacuum_max_workers` ได้ เพราะแต่ละ autovacuum worker ใช้ maintenance_work_mem ของตัวเอง (แต่มี autovacuum_work_mem แยกต่างหากถ้าต้องการจำกัดเฉพาะ autovacuum ไม่ให้กระทบ CREATE INDEX ที่ต้องการ memory มาก):

```sql
SHOW autovacuum_max_workers;
SHOW autovacuum_work_mem;  -- -1 แปลว่าใช้ค่าเดียวกับ maintenance_work_mem
```

```
หน่วยความจำสูงสุดที่ autovacuum ทั้งหมดอาจใช้พร้อมกัน
    = autovacuum_work_mem (หรือ maintenance_work_mem ถ้า autovacuum_work_mem = -1) × autovacuum_max_workers
```

ตัวอย่าง: `autovacuum_max_workers = 3`, `maintenance_work_mem = 2GB` → autovacuum อาจใช้ RAM สูงสุด 6GB พร้อมกัน ต้องเผื่อพื้นที่นี้ไว้เสมอเมื่อคำนวณงบ RAM รวม

### แยกค่าสำหรับ autovacuum โดยเฉพาะ (แนะนำสำหรับ production)

```ini
# postgresql.conf
maintenance_work_mem = 2GB       # ใช้สำหรับ manual CREATE INDEX / REINDEX
autovacuum_work_mem = 512MB      # จำกัด autovacuum ไม่ให้กิน RAM มากเกินไปตอนรันพร้อมกันหลาย worker
autovacuum_max_workers = 3
```

การแยกค่านี้ทำให้เรา "ให้ RAM เยอะ" กับงาน CREATE INDEX ที่รันครั้งคราว โดยไม่เสี่ยง autovacuum (ที่รันอัตโนมัติบ่อยและอาจมีหลาย worker พร้อมกัน) กิน RAM จนกระทบระบบ

### ตัวอย่างการใช้งานจริง: สร้าง index บนตารางใหญ่

```sql
-- ตารางมี 500 ล้านแถว ถ้าใช้ maintenance_work_mem default (64MB) จะช้ามาก
-- เพราะต้อง sort ข้อมูลหลายรอบ (external merge sort)

-- วิธีที่ 1: เพิ่มค่าระดับ session ก่อนสร้าง index
SET maintenance_work_mem = '4GB';
CREATE INDEX CONCURRENTLY idx_orders_created_at ON orders (created_at);
RESET maintenance_work_mem;
```

```sql
-- เปรียบเทียบเวลา: maintenance_work_mem = 64MB (default) vs 4GB
-- 64MB:  CREATE INDEX ใช้เวลา ~45 นาที (หลาย external sort pass)
-- 4GB:   CREATE INDEX ใช้เวลา ~8 นาที (sort ทำใน memory เกือบทั้งหมด)
```

> **เคล็ดลับ**: สำหรับการสร้าง index ครั้งเดียวบนตารางใหญ่มาก ให้ `SET maintenance_work_mem` สูงมาก (เช่น 8-16GB) ชั่วคราวใน session นั้น แทนที่จะตั้งเป็นค่า global เพราะเป็นงานที่ไม่ได้รันพร้อมกันหลาย session

---

## Step 726: wal_buffers, checkpoint_timeout, checkpoint_completion_target

### ทบทวน: WAL คืออะไร (เชื่อมโยง Part 061-062)

ใน Part 061-062 เราเรียนเรื่อง WAL (Write-Ahead Log) สำหรับ backup และ PITR ไปแล้ว บทนี้เราจะดูมุมมองด้าน **performance** ของ WAL — ทุกการเปลี่ยนแปลงข้อมูล (INSERT/UPDATE/DELETE) ต้องถูกเขียนลง WAL ก่อนเสมอ (write-ahead) เพื่อความทนทาน (durability) แต่การเขียน WAL และการทำ checkpoint มีผลต่อ throughput โดยตรง

### wal_buffers

`wal_buffers` คือ buffer ในหน่วยความจำสำหรับเก็บ WAL record ก่อนที่จะถูก flush ลงดิสก์จริง (คล้าย shared_buffers แต่สำหรับ WAL แทนข้อมูลตาราง)

```sql
SHOW wal_buffers;
```

Default คือ `-1` ซึ่งหมายถึง PostgreSQL จะคำนวณอัตโนมัติเป็น **1/32 ของ shared_buffers** (แต่มีเพดานขั้นต่ำ 64kB และสูงสุด 16MB โดยอัตโนมัติ)

```sql
-- ดูค่าจริงที่ระบบคำนวณให้ (ถ้าตั้ง -1)
SELECT setting, unit FROM pg_settings WHERE name = 'wal_buffers';
```

**คำแนะนำ**: สำหรับ production ที่มี write-heavy workload แนะนำตั้งค่าคงที่ **16MB** โดยตรง แทนการปล่อยให้คำนวณอัตโนมัติ เพราะ auto-calculate มักได้ค่าที่เล็กกว่าที่ควรจะเป็นสำหรับ workload เขียนหนัก

```ini
wal_buffers = 16MB
```

> ค่าที่สูงกว่า 16MB มักไม่ได้ประโยชน์เพิ่มเติมมากนัก เพราะ PostgreSQL flush WAL buffer ทุกครั้งที่ transaction commit (หรือทุก `wal_writer_delay`) อยู่แล้ว 16MB เพียงพอสำหรับ workload ที่มี transaction rate สูงมาก

### checkpoint คืออะไร และทำไมต้อง tune

Checkpoint คือกระบวนการที่ PostgreSQL เขียน dirty page ทั้งหมดใน shared_buffers ลงดิสก์จริง (data file) เพื่อให้แน่ใจว่าข้อมูลที่ commit แล้วปลอดภัยแม้ระบบ crash — และเพื่อจำกัดปริมาณ WAL ที่ต้อง replay ตอน crash recovery

```sql
SHOW checkpoint_timeout;
SHOW checkpoint_completion_target;
SHOW max_wal_size;
SHOW min_wal_size;
```

Default: `checkpoint_timeout = 5min`, `checkpoint_completion_target = 0.9`, `max_wal_size = 1GB`

### ปัญหาของ checkpoint ที่ถี่เกินไป

ถ้า checkpoint เกิดถี่ (เพราะ `checkpoint_timeout` สั้น หรือ `max_wal_size` เล็กเกินไปจนเขียน WAL ถึงเพดานเร็ว) จะเกิดปัญหา:

1. **I/O spike ถี่** — เขียน dirty page ลงดิสก์บ่อยเกินจำเป็น กระทบ query ที่กำลังรันพร้อมกัน (I/O contention)
2. **WAL เขียนซ้ำมากขึ้น** — full-page write เกิดขึ้นใหม่ทุกครั้งหลัง checkpoint สำหรับ page แรกที่ถูกแก้ไข (เพื่อป้องกัน torn page) ยิ่ง checkpoint ถี่ ยิ่งมี full-page write บ่อย ทำให้ WAL volume สูงขึ้นโดยไม่จำเป็น

ตรวจสอบว่า checkpoint เกิดถี่แค่ไหนและเกิดจากสาเหตุใด:

```sql
-- ต้องเปิด log_checkpoints ก่อน (แนะนำเปิดเสมอใน production)
SHOW log_checkpoints;

ALTER SYSTEM SET log_checkpoints = 'on';
SELECT pg_reload_conf();
```

ดู log:

```
LOG:  checkpoint starting: time
LOG:  checkpoint complete: wrote 12453 buffers (76.0%);
      0 WAL file(s) added, 0 removed, 3 recycled;
      write=45.123 s, sync=2.345 s, total=48.012 s;
      sync files=87, longest=0.412 s, average=0.027 s;
      distance=524288 kB, estimate=524288 kB
```

- `checkpoint starting: time` → checkpoint เกิดจาก `checkpoint_timeout` หมดเวลา (ปกติ)
- `checkpoint starting: xlog` → checkpoint เกิดเพราะเขียน WAL ถึงเพดาน `max_wal_size` (บ่งชี้ว่า max_wal_size เล็กเกินไปสำหรับ write rate ปัจจุบัน)

ถ้าเห็น `checkpoint starting: xlog` บ่อย ๆ แปลว่า workload เขียนข้อมูลเร็วกว่าที่ `max_wal_size` จะรองรับได้ในช่วงเวลา `checkpoint_timeout` — ต้องเพิ่ม `max_wal_size`

### สูตร/แนวทาง tune สำหรับ write-heavy workload

```ini
# postgresql.conf — สำหรับระบบที่มี transaction เขียนหนัก เช่น order/payment system
checkpoint_timeout = 15min          # เพิ่มจาก default 5min ลดความถี่ checkpoint
checkpoint_completion_target = 0.9  # กระจายการเขียน dirty page ตลอดช่วง 90% ของ checkpoint_timeout
max_wal_size = 4GB                  # เพิ่มเพดาน WAL ก่อน trigger checkpoint (เครื่อง 32GB RAM)
min_wal_size = 1GB                  # ขนาดขั้นต่ำที่เก็บไว้ recycle แทนการลบสร้างใหม่
wal_buffers = 16MB
```

**คำอธิบาย `checkpoint_completion_target = 0.9`**: หมายถึงให้ PostgreSQL กระจายการเขียน dirty page ออกไปให้เสร็จภายใน 90% ของช่วงเวลา `checkpoint_timeout` (เช่นถ้า timeout = 15min ก็เขียนกระจายให้เสร็จใน 13.5 นาที ไม่ใช่รีบเขียนทีเดียวจบใน 1 นาทีแรก) การกระจายแบบนี้ช่วยลด I/O spike ทำให้ throughput ของ query อื่นสม่ำเสมอกว่า

| ลักษณะ workload | checkpoint_timeout | checkpoint_completion_target | max_wal_size |
|---|---|---|---|
| OLTP ทั่วไป (read/write ผสม) | 10-15 min | 0.9 | 2-4 GB |
| Write-heavy (order, payment, IoT ingest) | 15-30 min | 0.9 | 4-16 GB |
| Read-heavy / reporting | 5-10 min (default ใกล้เคียง) | 0.7-0.9 | 1-2 GB |
| ต้องการ RTO สั้นมาก (crash recovery เร็ว) | 5 min (สั้นลง) | 0.9 | 1 GB (เล็กลง เพื่อจำกัด WAL ที่ต้อง replay) |

> **Trade-off สำคัญ**: `checkpoint_timeout`/`max_wal_size` ที่ใหญ่ขึ้น = throughput ดีขึ้น (checkpoint ถี่น้อยลง) แต่ = **crash recovery ช้าลง** (ต้อง replay WAL มากขึ้นตอน restart หลัง crash) ต้องสมดุลกับ RTO ที่ระบบต้องการ (เชื่อมโยง Part 062 เรื่อง PITR)

### ตรวจสอบว่า checkpoint tuning ได้ผล

```sql
-- ดูสถิติ background writer และ checkpoint (PostgreSQL 17+ อยู่ใน pg_stat_checkpointer)
-- PostgreSQL 16 และก่อนหน้า อยู่ใน pg_stat_bgwriter
SELECT
    checkpoints_timed,      -- checkpoint ที่เกิดจาก timeout (ปกติ ดี)
    checkpoints_req,        -- checkpoint ที่เกิดจาก WAL เต็ม (ควรน้อย)
    checkpoint_write_time,
    checkpoint_sync_time,
    buffers_checkpoint
FROM pg_stat_bgwriter;
```

```
 checkpoints_timed | checkpoints_req | checkpoint_write_time | checkpoint_sync_time | buffers_checkpoint
--------------------+-----------------+------------------------+-----------------------+---------------------
               1840 |              12 |                452300 |                 8320 |            4823910
```

อัตราส่วนที่ดี: `checkpoints_timed` ควรมากกว่า `checkpoints_req` อย่างชัดเจน (เช่น 95%+ เป็น timed) ถ้า `checkpoints_req` สูงเทียบเท่าหรือมากกว่า แปลว่าต้องเพิ่ม `max_wal_size`

---

## Step 727: max_connections และความสัมพันธ์กับ Connection Pooling

### ทบทวนจาก Part 066

ใน Part 066 เราเรียนเรื่อง PgBouncer และ connection pooling ไปแล้ว บทนี้จะอธิบายว่า **ทำไม max_connections ไม่ควรตั้งสูงมาก** และทำไม connection pooling จึงเป็นทางแก้ที่ถูกต้อง แทนการเพิ่ม max_connections เรื่อย ๆ

```sql
SHOW max_connections;

-- ดูจำนวน connection ที่ใช้งานจริงตอนนี้
SELECT count(*) AS current_connections,
       (SELECT setting::int FROM pg_settings WHERE name = 'max_connections') AS max_connections
FROM pg_stat_activity;

-- แยกตามสถานะ
SELECT state, count(*)
FROM pg_stat_activity
GROUP BY state
ORDER BY count(*) DESC;
```

```
     state      | count
-----------------+-------
 idle            |   142
 active          |    18
 idle in transaction |  3
```

### ทำไม max_connections สูง ๆ ถึงเป็นปัญหา

แต่ละ connection ใน PostgreSQL คือ **1 OS process** (ไม่ใช่ thread แบบฐานข้อมูลอื่น) ซึ่งมี overhead แน่นอน:

1. **หน่วยความจำต่อ connection** — แต่ละ backend process ใช้ RAM พื้นฐานประมาณ 5-10MB (ไม่รวม work_mem ที่อาจจองเพิ่มตอนรัน query) ถ้า `max_connections = 1000` แม้ connection ส่วนใหญ่ idle ก็ยังกิน RAM รวมหลาย GB โดยไม่ได้ใช้ประโยชน์
2. **Context-switching overhead** — ยิ่งจำนวน process ที่ active พร้อมกันมาก (active connection ไม่ใช่ idle) OS ยิ่งต้องสลับ CPU ระหว่าง process บ่อยขึ้น ทำให้ throughput รวมลดลงเมื่อเทียบกับการจำกัดจำนวน concurrent query ให้พอดีกับจำนวน CPU core
3. **Lock contention ภายใน PostgreSQL เอง** — โครงสร้างข้อมูลภายใน เช่น `ProcArray` (ใช้ตรวจสอบ transaction visibility สำหรับ MVCC) ต้องวนลูปตรวจสอบทุก connection ที่ active ยิ่งจำนวนเยอะ ยิ่งกระทบ performance ของทุก query ที่ต้องเช็ค snapshot

```sql
-- ทดสอบแนวคิด: ดูว่า process แต่ละตัวใน backend ใช้ RAM เท่าไหร่ (โดยประมาณ)
SELECT pid, usename, state,
       pg_size_pretty(pg_backend_memory_contexts_check()) -- แนวคิดเชิงตัวอย่าง
FROM pg_stat_activity
LIMIT 5;
```

> ในทางปฏิบัติ การประมาณ memory ต่อ backend ทำได้ผ่าน `pg_backend_memory_contexts` (ต้องรันใน session นั้นเอง) หรือดูจาก OS โดยตรงด้วย `ps` / `pmap` ต่อ PID ของ backend process

### สูตรคำนวณ max_connections ที่เหมาะสม

> **max_connections ควรใกล้เคียงกับจำนวน concurrent active query ที่ต้องการจริง ไม่ใช่จำนวน client ทั้งหมด**

แนวทางคำนวณ (คล้ายสูตรที่ใช้ตั้งค่า thread pool ทั่วไป):

```
max_connections ที่เหมาะสม ≈ จำนวน CPU core × 2 ถึง 4  (สำหรับ query ที่ CPU-bound)
```

หรือถ้าเป็น I/O-bound (query รอ disk บ่อย) อาจสูงกว่านี้ได้เล็กน้อย แต่หลักการคือ **จำนวน active connection ที่ทำงานพร้อมกันจริง ๆ ไม่ควรเกิน 2-4 เท่าของ CPU core** — ส่วนจำนวน client ที่ "เชื่อมต่อ" ทั้งหมด (idle ส่วนใหญ่) ให้ผ่าน connection pooler แทน

**ตัวอย่างการออกแบบ**: เครื่อง 16 core, แอปพลิเคชันมี 500 instance ที่แต่ละตัวเปิด connection pool ของตัวเอง 20 connection (= 10,000 connection ตามทฤษฎี):

```
max_connections (PostgreSQL จริง) = 16 core × 3 ≈ 50-100
PgBouncer pool_size (ต่อ database) = 50-100
Application connection pool ทั้งหมด (10,000 conn) → เชื่อมต่อผ่าน PgBouncer แบบ transaction pooling
```

```ini
# postgresql.conf — เครื่อง 16 core
max_connections = 100
```

```
Application (10,000 logical connections)
        ↓
   PgBouncer (transaction pooling mode)
        ↓  (pool_size = 80-100)
   PostgreSQL (max_connections = 100)
```

### เผื่อ connection สำหรับงาน maintenance และ superuser

```sql
SHOW superuser_reserved_connections;
```

Default = 3 — เผื่อไว้ให้ superuser เข้าถึง database ได้เสมอแม้ connection เต็ม (สำคัญมากตอนแก้ปัญหาฉุกเฉิน เช่น connection leak ทำให้ pool เต็ม)

```ini
max_connections = 100
superuser_reserved_connections = 5   # เผื่อสำหรับ DBA เข้าถึงตอนฉุกเฉิน
```

> **หลักการสำคัญ**: อย่าตั้ง `max_connections` สูง ๆ เพื่อ "แก้ปัญหา too many connections" — นั่นคือการรักษาอาการ ไม่ใช่สาเหตุ สาเหตุที่แท้จริงมักเป็น connection leak ในแอปพลิเคชัน หรือไม่มี connection pooling ที่เหมาะสม การแก้ที่ถูกต้องคือใส่ PgBouncer/pgcat (ตาม Part 066) ไม่ใช่เพิ่ม max_connections ไปเรื่อย ๆ จนกระทบ memory และ performance โดยรวม

### ตรวจสอบ connection ที่ค้าง (idle in transaction) ซึ่งเป็นสาเหตุ leak บ่อยที่สุด

```sql
-- หา connection ที่ idle in transaction นานผิดปกติ (อาจเป็น bug ในแอป ไม่ COMMIT/ROLLBACK)
SELECT pid, usename, application_name, state,
       now() - state_change AS idle_duration,
       query
FROM pg_stat_activity
WHERE state = 'idle in transaction'
  AND now() - state_change > interval '5 minutes'
ORDER BY idle_duration DESC;
```

```sql
-- ตั้ง timeout อัตโนมัติเพื่อป้องกัน idle in transaction ค้างนานเกินไป
ALTER SYSTEM SET idle_in_transaction_session_timeout = '10min';
SELECT pg_reload_conf();
```

---

## Step 728: random_page_cost, effective_io_concurrency — Tune ตามประเภท Storage

### random_page_cost คืออะไร

`random_page_cost` คือค่าที่บอก query planner ว่า "การอ่านข้อมูลแบบ random (กระโดดไปมา เช่น index scan)" มีต้นทุนสัมพัทธ์เท่าไหร่ เมื่อเทียบกับการอ่านแบบ sequential (`seq_page_cost = 1.0` เป็นค่าอ้างอิงคงที่)

```sql
SHOW random_page_cost;
SHOW seq_page_cost;
```

Default: `random_page_cost = 4.0`, `seq_page_cost = 1.0` — หมายความว่า planner คิดว่าการอ่านแบบ random แพงกว่า sequential ถึง 4 เท่า ซึ่ง**เป็นค่าที่ออกแบบมาสำหรับ HDD แบบจานหมุน** (ที่หัวอ่านต้องขยับไปมาจริง ทำให้ random access ช้ากว่ามาก)

### ทำไม default 4.0 ไม่เหมาะกับ SSD/NVMe

SSD และ NVMe ไม่มีหัวอ่านที่ต้องขยับทางกายภาพ การอ่านแบบ random และ sequential จึงมีต้นทุนใกล้เคียงกันมาก (ต่างจาก HDD อย่างมาก) ถ้ายังใช้ `random_page_cost = 4.0` บน SSD, planner จะ "กลัว" การใช้ index scan เกินความจำเป็น และเลือก sequential scan บ่อยเกินไป ทั้งที่ index scan น่าจะเร็วกว่าในความเป็นจริง

### ทดสอบผลกระทบจริง

```sql
-- สร้างตารางทดสอบและ index
CREATE TABLE test_orders (
    order_id serial PRIMARY KEY,
    customer_id int,
    order_date date,
    amount numeric
);
INSERT INTO test_orders (customer_id, order_date, amount)
SELECT (random() * 100000)::int, current_date - (random() * 365)::int, random() * 1000
FROM generate_series(1, 5000000);
CREATE INDEX idx_test_orders_customer ON test_orders (customer_id);
ANALYZE test_orders;
```

```sql
-- ทดสอบกับ random_page_cost แบบ HDD (default)
SET random_page_cost = 4.0;
EXPLAIN SELECT * FROM test_orders WHERE customer_id = 5000;
```

```
Seq Scan on test_orders  (cost=0.00..104083.00 rows=48 width=24)
  Filter: (customer_id = 5000)
```

```sql
-- ทดสอบกับ random_page_cost แบบ SSD/NVMe
SET random_page_cost = 1.1;
EXPLAIN SELECT * FROM test_orders WHERE customer_id = 5000;
```

```
Index Scan using idx_test_orders_customer on test_orders
                          (cost=0.43..8.95 rows=48 width=24)
  Index Cond: (customer_id = 5000)
```

เห็นชัดว่าเมื่อ `random_page_cost` ลดลง planner เปลี่ยนมาเลือก index scan ซึ่งในกรณีนี้เร็วกว่ามากสำหรับ storage ที่เป็น SSD จริง

### ค่าแนะนำตามประเภท storage

| ประเภท Storage | random_page_cost แนะนำ | หมายเหตุ |
|---|---|---|
| HDD แบบจานหมุน (rotational) | 4.0 (default) | ค่า default ถูกออกแบบมาสำหรับกรณีนี้อยู่แล้ว |
| SATA SSD | 1.5 - 2.0 | เร็วกว่า HDD มาก แต่ยังมี overhead บาง |
| NVMe SSD (local, on-premise) | 1.1 - 1.5 | latency ต่ำมาก random ≈ sequential เกือบเท่ากัน |
| Cloud block storage (AWS EBS gp3/io2, GCP PD-SSD, Azure Premium SSD) | 1.1 - 1.3 | โดย technical เป็น network-attached SSD แต่ throughput/IOPS สม่ำเสมอ |
| Cloud NVMe local instance store (เช่น AWS i3/i4 instance store) | 1.0 - 1.1 | เกือบเทียบเท่า sequential |

```ini
# postgresql.conf — เครื่องที่ใช้ NVMe SSD หรือ cloud block storage สมัยใหม่
random_page_cost = 1.1
seq_page_cost = 1.0
```

### effective_io_concurrency

พารามิเตอร์นี้บอก PostgreSQL ว่า storage รองรับการอ่านแบบ **parallel/concurrent I/O request** ได้กี่ตัวพร้อมกัน มีผลกับ operation ที่อ่านหลาย page พร้อมกันได้ เช่น bitmap heap scan

```sql
SHOW effective_io_concurrency;
```

Default = 1 (ไม่มี concurrent I/O) เหมาะกับ HDD ตัวเดียวที่อ่านทีละคำสั่ง

| ประเภท Storage | effective_io_concurrency แนะนำ |
|---|---|
| HDD เดี่ยว | 1-2 |
| HDD แบบ RAID (หลายจาน) | จำนวนจานใน RAID (เช่น RAID 10 4 จาน → 4-8) |
| SATA SSD | 100-200 |
| NVMe SSD | 200-300 |
| Cloud block storage (EBS gp3, PD-SSD) | 200 (คู่มือ cloud provider หลายเจ้าแนะนำค่านี้) |

```ini
# postgresql.conf — NVMe SSD หรือ cloud storage สมัยใหม่
effective_io_concurrency = 200
```

> **หมายเหตุสำคัญ**: `effective_io_concurrency` มีผลเฉพาะกับ operation ที่ใช้ **prefetch** เช่น bitmap heap scan เท่านั้น ไม่ได้ช่วยทุก query แต่การตั้งค่าให้สอดคล้องกับ storage จริงก็ยังสำคัญสำหรับ workload ที่มี query ประเภทนี้บ่อย และในเวอร์ชันใหม่ยังมี `maintenance_io_concurrency` แยกสำหรับงาน VACUUM ด้วย

```sql
SHOW maintenance_io_concurrency;
```

```ini
maintenance_io_concurrency = 200   # สำหรับ VACUUM/CREATE INDEX บน SSD/NVMe
```

### สรุปการเลือกค่าตามประเภท storage

```ini
# ===== HDD แบบจานหมุน =====
random_page_cost = 4.0
effective_io_concurrency = 2

# ===== SATA SSD =====
random_page_cost = 1.5
effective_io_concurrency = 150

# ===== NVMe SSD / Cloud Block Storage สมัยใหม่ (แนะนำสำหรับ production ส่วนใหญ่ในปี 2026) =====
random_page_cost = 1.1
effective_io_concurrency = 200
maintenance_io_concurrency = 200
```

---

## Step 729: Parallel Query Parameters

### แนวคิด Parallel Query

PostgreSQL สามารถแบ่งงานของ query เดียว (เช่น sequential scan บนตารางใหญ่, hash join, aggregate) ให้หลาย **worker process** ทำงานพร้อมกันบนหลาย CPU core แล้วรวมผลลัพธ์ ทำให้ query ที่ประมวลผลข้อมูลจำนวนมากเร็วขึ้นอย่างมีนัยสำคัญ

```sql
SHOW max_worker_processes;
SHOW max_parallel_workers;
SHOW max_parallel_workers_per_gather;
SHOW min_parallel_table_scan_size;
SHOW min_parallel_index_scan_size;
```

Default: `max_worker_processes = 8`, `max_parallel_workers = 8`, `max_parallel_workers_per_gather = 2`

### ลำดับชั้นของพารามิเตอร์ parallel worker

เข้าใจความสัมพันธ์ระหว่าง 3 พารามิเตอร์นี้เป็นสิ่งสำคัญที่สุด:

```
max_worker_processes (เพดานสูงสุด background process ทั้งหมด รวม parallel worker, replication worker, logical replication, extension workers ฯลฯ)
    └── max_parallel_workers (เพดาน worker process ที่ใช้สำหรับ parallel query เท่านั้น ต้อง ≤ max_worker_processes)
            └── max_parallel_workers_per_gather (เพดาน worker ที่ query เดียวใช้ได้สูงสุด ต้อง ≤ max_parallel_workers)
```

```sql
-- ตัวอย่าง: ตรวจสอบว่าค่าทั้ง 3 สอดคล้องกันหรือไม่
SELECT name, setting FROM pg_settings
WHERE name IN ('max_worker_processes', 'max_parallel_workers', 'max_parallel_workers_per_gather')
ORDER BY name;
```

ถ้า `max_worker_processes = 8` แต่ `max_parallel_workers = 8` ด้วย จะไม่เหลือ worker process สำหรับงานอื่น เช่น logical replication apply worker หรือ extension background worker — ต้องเผื่อพื้นที่ไว้เสมอ

### ทดสอบผลของ parallel query

```sql
-- ตารางทดสอบ 5 ล้านแถว (จาก step ก่อนหน้า)
SET max_parallel_workers_per_gather = 0;  -- ปิด parallel เพื่อดู baseline
EXPLAIN ANALYZE
SELECT count(*) FROM test_orders WHERE amount > 500;
```

```
Aggregate  (cost=104083.00..104083.01 rows=1 width=8)
           (actual time=612.345..612.346 rows=1 loops=1)
  ->  Seq Scan on test_orders  (cost=0.00..91583.00 rows=2500000 width=0)
                                (actual time=0.012..480.234 rows=2498451 loops=1)
        Filter: (amount > '500'::numeric)
Planning Time: 0.234 ms
Execution Time: 612.512 ms
```

```sql
SET max_parallel_workers_per_gather = 4;  -- เปิด parallel ใช้ 4 worker
EXPLAIN ANALYZE
SELECT count(*) FROM test_orders WHERE amount > 500;
```

```
Finalize Aggregate  (cost=48583.50..48583.51 rows=1 width=8)
                     (actual time=165.234..165.235 rows=1 loops=1)
  ->  Gather  (cost=48583.00..48583.41 rows=4 width=8)
              (actual time=165.012..165.220 rows=5 loops=1)
        Workers Planned: 4
        Workers Launched: 4
        ->  Partial Aggregate  (cost=47583.00..47583.01 rows=1 width=8)
                                (actual time=158.345..158.346 rows=1 loops=5)
              ->  Parallel Seq Scan on test_orders
                              (cost=0.00..45833.00 rows=625000 width=0)
                              (actual time=0.045..120.123 rows=499690 loops=5)
                    Filter: (amount > '500'::numeric)
Planning Time: 0.198 ms
Execution Time: 165.312 ms
```

จาก 612ms เหลือ 165ms (~3.7 เท่า) เพราะ 4 worker ช่วยกันสแกนตารางคนละส่วนพร้อมกัน

### เมื่อไหร่ที่ parallel query ไม่ถูกใช้

Planner จะไม่ใช้ parallel query ถ้า:

1. **ตารางเล็กเกินไป** — ต่ำกว่า `min_parallel_table_scan_size` (default 8MB) planner คิดว่า overhead การแบ่งงานไม่คุ้ม
2. **Query อยู่ใน transaction ที่มี write** — parallel query ใช้ได้เฉพาะ read-only query (SELECT ที่ไม่มี side effect หรือ CTE ที่มี data modification)
3. **ใช้ function ที่ไม่ใช่ `PARALLEL SAFE`** — function ที่ระบุเป็น `PARALLEL UNSAFE` หรือ `PARALLEL RESTRICTED` (ค่า default ของ user-defined function คือ UNSAFE) จะบล็อก parallel plan
4. **cost ของ query ต่ำกว่า `parallel_setup_cost` + `parallel_tuple_cost`** — query เล็กเกินไปจน overhead การสร้าง/สื่อสารกับ worker ไม่คุ้มค่า

```sql
SHOW parallel_setup_cost;
SHOW parallel_tuple_cost;
SHOW min_parallel_table_scan_size;
SHOW min_parallel_index_scan_size;
```

### สูตรคำนวณสำหรับ Production

> **max_worker_processes ≈ จำนวน CPU core (หรือมากกว่าเล็กน้อยเผื่องานอื่น)**
> **max_parallel_workers ≈ 50-75% ของ max_worker_processes**
> **max_parallel_workers_per_gather ≈ 2-4** (ไม่ควรสูงเกินไปเพราะ query เดียวจะแย่ง CPU core จาก connection อื่นที่รันพร้อมกัน)

| CPU Core | max_worker_processes | max_parallel_workers | max_parallel_workers_per_gather |
|---|---|---|---|
| 4 | 8 | 4 | 2 |
| 8 | 12 | 8 | 2-4 |
| 16 | 20 | 12-16 | 4 |
| 32 | 36 | 24 | 4-8 |
| 64 | 68 | 48 | 8 |

```ini
# postgresql.conf — เครื่อง 16 core, OLTP + analytics ผสม
max_worker_processes = 20          # เผื่อ replication/extension workers เพิ่มจาก 16
max_parallel_workers = 14
max_parallel_workers_per_gather = 4
parallel_setup_cost = 1000         # default ปกติไม่ต้องแก้ เว้นแต่มีเหตุผลเฉพาะ
parallel_tuple_cost = 0.1          # default ปกติไม่ต้องแก้
min_parallel_table_scan_size = 8MB
min_parallel_index_scan_size = 512kB
```

### ข้อควรระวัง: parallel_workers_per_gather สูงเกินไปในระบบ OLTP ที่มี concurrent สูง

ถ้าระบบมี concurrent query จำนวนมาก (OLTP ที่มี 50-100 active connection พร้อมกัน) การตั้ง `max_parallel_workers_per_gather` สูง (เช่น 8) อาจทำให้ query เดียวแย่ง CPU core จาก connection อื่นจนกระทบ throughput โดยรวม — ในกรณีนี้ควรตั้งค่าต่ำ (2) หรือปิด parallel query สำหรับ table บางตัวด้วย:

```sql
-- ปิด parallel query เฉพาะตาราง OLTP ที่มี concurrent เข้าถึงสูง (เช่น ตาราง session/cart)
ALTER TABLE shopping_cart SET (parallel_workers = 0);
```

```sql
-- ตั้งเฉพาะ session สำหรับ query analytics ที่ต้องการ parallel เต็มที่
SET max_parallel_workers_per_gather = 8;
-- รัน query analytics ...
RESET max_parallel_workers_per_gather;
```

---

## Step 730: แบบฝึกหัดรวม — คำนวณและเขียน postgresql.conf สำหรับ Server ขนาดต่าง ๆ

สถานการณ์: ระบบ **e-commerce** ที่มี workload ผสม (OLTP เป็นหลัก: การสั่งซื้อ ตะกร้าสินค้า สต็อก + รายงานยอดขายเบา ๆ ตอนกลางคืน) ใช้ PostgreSQL 16 บน SSD/NVMe (cloud block storage) มี connection pooling (PgBouncer) อยู่หน้าแอปพลิเคชันอยู่แล้ว

### กรณีที่ 1: Server ขนาดเล็ก — 8GB RAM, 4 CPU core (Startup / Staging)

**การคำนวณ**:

```
shared_buffers        = 8GB × 25%        = 2GB
effective_cache_size  = 8GB × 65%        = 5GB
work_mem              = (8GB - 2GB - 1GB reserve) / (30 concurrent × 2.5 ops) ≈ 68MB → ปรับลงเหลือ 16MB (ปลอดภัยไว้ก่อน)
maintenance_work_mem  = 8GB × 6%         ≈ 512MB
max_connections        = 4 core × 5 (เผื่อ I/O wait เยอะเพราะเครื่องเล็ก) = 40 (ผ่าน PgBouncer)
max_worker_processes   = 4 core + 4 เผื่อ = 8
max_parallel_workers   = 4
max_parallel_workers_per_gather = 2
random_page_cost        = 1.1 (NVMe/cloud SSD)
effective_io_concurrency = 200
```

```ini
# ============================================
# postgresql.conf — Server 8GB RAM / 4 CPU core
# e-commerce (staging / small production)
# ============================================

# --- Memory ---
shared_buffers = 2GB
effective_cache_size = 5GB
work_mem = 16MB
maintenance_work_mem = 512MB
autovacuum_work_mem = 256MB

# --- Checkpoint / WAL ---
wal_buffers = 16MB
checkpoint_timeout = 10min
checkpoint_completion_target = 0.9
max_wal_size = 2GB
min_wal_size = 512MB

# --- Connections ---
max_connections = 40
superuser_reserved_connections = 3
idle_in_transaction_session_timeout = 10min

# --- Storage / Planner ---
random_page_cost = 1.1
seq_page_cost = 1.0
effective_io_concurrency = 200
maintenance_io_concurrency = 200

# --- Parallel Query ---
max_worker_processes = 8
max_parallel_workers = 4
max_parallel_workers_per_gather = 2

# --- Logging (แนะนำเปิดเสมอใน production) ---
log_checkpoints = on
log_temp_files = 10MB
log_lock_waits = on
log_min_duration_statement = 500ms
```

### กรณีที่ 2: Server ขนาดกลาง — 32GB RAM, 16 CPU core (Production ทั่วไป)

**การคำนวณ**:

```
shared_buffers        = 32GB × 25%       = 8GB
effective_cache_size  = 32GB × 70%       = 22GB
work_mem              = (32GB - 8GB - 2GB reserve) / (60 concurrent × 3 ops) ≈ 122MB → ตั้ง global 32MB (อนุรักษ์นิยม), ให้ session-level สูงขึ้นสำหรับ report
maintenance_work_mem  = 32GB × 6%        ≈ 2GB
max_connections        = 16 core × 4     = 64 (ผ่าน PgBouncer, pool ฝั่งแอปอาจมีหลักพัน)
max_worker_processes   = 16 core + 4     = 20
max_parallel_workers   = 14
max_parallel_workers_per_gather = 4
random_page_cost        = 1.1
effective_io_concurrency = 200
```

```ini
# ============================================
# postgresql.conf — Server 32GB RAM / 16 CPU core
# e-commerce production (traffic ปานกลาง-สูง)
# ============================================

# --- Memory ---
shared_buffers = 8GB
effective_cache_size = 22GB
work_mem = 32MB
maintenance_work_mem = 2GB
autovacuum_work_mem = 512MB

# --- Checkpoint / WAL ---
wal_buffers = 16MB
checkpoint_timeout = 15min
checkpoint_completion_target = 0.9
max_wal_size = 4GB
min_wal_size = 1GB

# --- Connections ---
max_connections = 64
superuser_reserved_connections = 5
idle_in_transaction_session_timeout = 10min

# --- Storage / Planner ---
random_page_cost = 1.1
seq_page_cost = 1.0
effective_io_concurrency = 200
maintenance_io_concurrency = 200

# --- Parallel Query ---
max_worker_processes = 20
max_parallel_workers = 14
max_parallel_workers_per_gather = 4

# --- Autovacuum (เผื่อสำหรับตาราง order/order_items ที่ update/insert หนัก) ---
autovacuum_max_workers = 4
autovacuum_naptime = 15s

# --- Logging ---
log_checkpoints = on
log_temp_files = 10MB
log_lock_waits = on
log_min_duration_statement = 500ms
log_autovacuum_min_duration = 1000ms
```

### กรณีที่ 3: Server ขนาดใหญ่ — 128GB RAM, 32 CPU core (High-traffic Production)

**การคำนวณ**:

```
shared_buffers        = คำนวณ 25% = 32GB แต่ชนเพดานแนะนำ → ใช้ 32GB (ไม่ขยับเกิน)
effective_cache_size  = 128GB × 70%      = 90GB
work_mem              = (128GB - 32GB - 4GB reserve) / (150 concurrent × 3 ops) ≈ 205MB → ตั้ง global 64MB (สูงกว่ากรณีก่อนเพราะ RAM เหลือเยอะ), session-level ขึ้นได้ถึง 512MB-1GB สำหรับ analytics
maintenance_work_mem  = 128GB × 3%       ≈ 4GB (เปอร์เซ็นต์ลดลงเพราะฐาน RAM ใหญ่ ไม่จำเป็นต้องคูณตรง ๆ)
max_connections        = 32 core × 4     = 128 (ผ่าน PgBouncer)
max_worker_processes   = 32 core + 8     = 40
max_parallel_workers   = 28
max_parallel_workers_per_gather = 8 (เครื่องใหญ่ รองรับได้มากกว่า)
random_page_cost        = 1.1
effective_io_concurrency = 200-300 (NVMe ระดับ enterprise)
```

```ini
# ============================================
# postgresql.conf — Server 128GB RAM / 32 CPU core
# e-commerce production (traffic สูงมาก, Black Friday-scale)
# ============================================

# --- Memory ---
shared_buffers = 32GB
effective_cache_size = 90GB
work_mem = 64MB
maintenance_work_mem = 4GB
autovacuum_work_mem = 1GB

# --- Checkpoint / WAL ---
wal_buffers = 16MB
checkpoint_timeout = 20min
checkpoint_completion_target = 0.9
max_wal_size = 8GB
min_wal_size = 2GB

# --- Connections ---
max_connections = 128
superuser_reserved_connections = 5
idle_in_transaction_session_timeout = 10min

# --- Storage / Planner ---
random_page_cost = 1.1
seq_page_cost = 1.0
effective_io_concurrency = 300
maintenance_io_concurrency = 300

# --- Parallel Query ---
max_worker_processes = 40
max_parallel_workers = 28
max_parallel_workers_per_gather = 8

# --- Autovacuum (สำคัญมากในเครื่อง scale นี้ ตารางใหญ่ update ถี่) ---
autovacuum_max_workers = 6
autovacuum_naptime = 10s
autovacuum_vacuum_cost_limit = 2000
autovacuum_vacuum_cost_delay = 2ms

# --- Logging ---
log_checkpoints = on
log_temp_files = 10MB
log_lock_waits = on
log_min_duration_statement = 500ms
log_autovacuum_min_duration = 1000ms
```

### ตรวจสอบค่าที่ตั้งไว้จริงหลัง apply

```sql
-- สคริปต์ตรวจสอบภาพรวมพารามิเตอร์สำคัญทั้งหมดในครั้งเดียว
SELECT name,
       setting,
       unit,
       CASE context
           WHEN 'postmaster' THEN 'ต้อง restart'
           WHEN 'sighup'     THEN 'reload พอ'
           WHEN 'user'       THEN 'SET ได้ทันที'
           ELSE context
       END AS วิธีเปลี่ยนค่า
FROM pg_settings
WHERE name IN (
    'shared_buffers', 'effective_cache_size', 'work_mem', 'maintenance_work_mem',
    'wal_buffers', 'checkpoint_timeout', 'checkpoint_completion_target',
    'max_wal_size', 'min_wal_size', 'max_connections',
    'random_page_cost', 'effective_io_concurrency',
    'max_worker_processes', 'max_parallel_workers', 'max_parallel_workers_per_gather'
)
ORDER BY name;
```

```sql
-- ตรวจสอบว่ามีการตั้งค่าใน postgresql.auto.conf ที่ conflict กับไฟล์หลักหรือไม่
SELECT name, setting, source, sourcefile, sourceline
FROM pg_settings
WHERE source = 'configuration file'
ORDER BY sourcefile, sourceline;
```

---

## สรุปท้ายบท

### ตารางสรุปพารามิเตอร์สำคัญพร้อมสูตรคำนวณ

| พารามิเตอร์ | สูตร/แนวทาง | Context | หมายเหตุ |
|---|---|---|---|
| `shared_buffers` | 25% ของ RAM (เพดาน ~32-40GB) | postmaster (ต้อง restart) | อย่าให้เกิน 40GB แม้ RAM เยอะ |
| `effective_cache_size` | 50-75% ของ RAM | user (reload พอ) | ไม่จองหน่วยความจำจริง เป็นแค่ hint ให้ planner |
| `work_mem` | (RAM - shared_buffers - reserve) / (concurrent × avg_ops) | user (SET ได้ทันที) | ระวัง OOM ถ้าตั้งสูงเกินไปตอนมี concurrent query เยอะ |
| `maintenance_work_mem` | 5-10% ของ RAM (เพดาน 2-4GB ทั่วไป) | user (SET ได้ทันที) | ตั้งสูงชั่วคราวตอน CREATE INDEX ได้ |
| `autovacuum_work_mem` | ต่ำกว่า maintenance_work_mem | user (reload พอ) | แยกจาก maintenance_work_mem เพื่อจำกัด autovacuum |
| `wal_buffers` | ตั้งคงที่ 16MB | postmaster (ต้อง restart) | ไม่ต้องคำนวณซับซ้อน ตั้ง 16MB พอสำหรับเกือบทุกกรณี |
| `checkpoint_timeout` | 10-30 min ตาม write rate | sighup (reload พอ) | ยาวขึ้น = throughput ดีขึ้น แต่ crash recovery ช้าลง |
| `checkpoint_completion_target` | 0.9 (คงที่) | sighup (reload พอ) | กระจายการเขียน dirty page ลด I/O spike |
| `max_wal_size` | 2-4x ของ WAL volume ต่อ checkpoint interval | sighup (reload พอ) | ดู log checkpoint ว่าเกิดจาก timed หรือ xlog |
| `max_connections` | CPU core × 2-4 (ที่เหลือผ่าน pooling) | postmaster (ต้อง restart) | อย่าใช้แทน connection pooling |
| `random_page_cost` | 1.1 (SSD/NVMe) / 4.0 (HDD) | user (SET ได้ทันที) | สำคัญมากสำหรับ index scan vs seq scan |
| `effective_io_concurrency` | 200 (SSD/NVMe) / 1-2 (HDD) | user (SET ได้ทันที) | มีผลกับ bitmap heap scan prefetch |
| `max_worker_processes` | CPU core + เผื่อ 20-25% | postmaster (ต้อง restart) | ต้อง ≥ max_parallel_workers |
| `max_parallel_workers` | 50-75% ของ max_worker_processes | sighup (reload พอ) | ต้อง ≤ max_worker_processes |
| `max_parallel_workers_per_gather` | 2-8 ตามขนาดเครื่อง | user (SET ได้ทันที) | สูงไปกระทบ concurrent query อื่น |

### หลักการที่ต้องจำ 5 ข้อ

1. **ไม่มีค่าสำเร็จรูปที่ใช้ได้ทุกที่** — ต้องคำนวณจาก RAM, CPU, storage, และลักษณะ workload ของระบบตัวเองเสมอ
2. **shared_buffers ไม่ใช่ยิ่งเยอะยิ่งดี** — 25% ของ RAM เป็นจุดสมดุลที่ผ่านการพิสูจน์แล้วในงานส่วนใหญ่ ปล่อยให้ OS page cache ทำงานร่วมด้วยผ่าน effective_cache_size
3. **work_mem ต้องคูณด้วยจำนวน operation ไม่ใช่แค่จำนวน connection** — ตั้งสูงเกินไปคือสาเหตุอันดับต้น ๆ ของ OOM ใน production
4. **random_page_cost ต้องสอดคล้องกับ storage จริง** — ค่า default 4.0 ออกแบบมาสำหรับ HDD เท่านั้น บน SSD/NVMe ต้องลดลงมาเหลือ 1.1-1.5
5. **max_connections ไม่ใช่ทางแก้ของปัญหา "too many connections"** — ใช้ connection pooling (Part 066) แทนการเพิ่มค่านี้ไปเรื่อย ๆ

### Checklist ก่อนนำไป Production

- [ ] วัด RAM, CPU, ประเภท storage ของเซิร์ฟเวอร์จริงแล้ว (ไม่ใช่สเปคที่คาดเดา)
- [ ] ตั้งค่า `shared_buffers` และ `effective_cache_size` ตามสูตร แล้ว restart
- [ ] ตั้งค่า `work_mem` แบบอนุรักษ์นิยม (global) และใช้ `SET LOCAL` สำหรับ query หนักเฉพาะจุด
- [ ] เปิด `log_checkpoints`, `log_temp_files`, `log_lock_waits` เพื่อ monitor ผลของการ tune
- [ ] ตรวจสอบ `pg_stat_bgwriter` ว่า checkpoint ส่วนใหญ่เป็น `timed` ไม่ใช่ `req`
- [ ] ตั้ง `random_page_cost`/`effective_io_concurrency` ตามประเภท storage จริง (ทดสอบด้วย `EXPLAIN ANALYZE`)
- [ ] มี connection pooling (PgBouncer) อยู่หน้า PostgreSQL แล้ว ไม่พึ่ง `max_connections` สูง ๆ
- [ ] ตั้ง parallel query parameters ให้สอดคล้องกับจำนวน CPU core และลักษณะ concurrent workload
- [ ] ทดสอบ load test ก่อนและหลัง tune เพื่อยืนยันผลลัพธ์จริง (อย่าเชื่อทฤษฎีอย่างเดียว)
- [ ] บันทึกค่าที่ตั้งทั้งหมดไว้ใน version control (เช่น เก็บไฟล์ `postgresql.conf` ใน git) เพื่อ track การเปลี่ยนแปลงย้อนหลัง

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1

เซิร์ฟเวอร์มี RAM 16GB เป็น dedicated PostgreSQL server (ไม่มี service อื่นแชร์) ควรตั้งค่า `shared_buffers` และ `effective_cache_size` เท่าไหร่?

<details>
<summary>เฉลย</summary>

```
shared_buffers = 16GB × 25% = 4GB
effective_cache_size = 16GB × 65-70% = 10-11GB
```

```ini
shared_buffers = 4GB
effective_cache_size = 10GB
```

เพราะเป็น dedicated server จึงใช้สูตรมาตรฐาน 25%/65-70% ได้ตรง ๆ โดยไม่ต้องลดสัดส่วนเผื่อ service อื่น
</details>

### แบบฝึกหัดที่ 2

ทำไมการตั้ง `work_mem = 500MB` แบบ global บนเซิร์ฟเวอร์ที่มี `max_connections = 200` และ RAM 16GB จึงเป็นความเสี่ยงร้ายแรง แม้ในสถานการณ์ปกติ query ส่วนใหญ่จะไม่ได้ใช้ work_mem เต็มเพดาน?

<details>
<summary>เฉลย</summary>

เพราะในสถานการณ์ที่มี **traffic สูงพร้อมกัน (peak load)** หรือมี query ที่ซับซ้อน (หลาย JOIN/sort พร้อมกัน) จำนวนมากรันพร้อมกันจริง ๆ หน่วยความจำที่ต้องใช้อาจสูงถึง:

```
200 connections × 500MB × (สมมติเฉลี่ย 2 operation ต่อ query) = 200,000MB = ~195GB
```

ซึ่งเกิน RAM ของเครื่อง (16GB) มหาศาล แม้ในสถานการณ์ปกติจะไม่ถึงจุดนี้ แต่ถ้าเกิด traffic spike หรือ query ที่ผิดปกติ (เช่น query ที่ไม่มี WHERE clause ทำ full scan + sort) พร้อมกันหลายตัว ระบบจะเข้าสู่ **OOM (Out of Memory)** และ Linux OOM killer อาจ kill PostgreSQL process ทำให้ทั้งฐานข้อมูล crash

ทางแก้ที่ถูกต้อง: ตั้ง `work_mem` แบบอนุรักษ์นิยม (เช่น 16-32MB) เป็นค่า global แล้วใช้ `SET LOCAL work_mem = '500MB'` เฉพาะ session/query ที่ต้องการจริง ๆ ภายใน transaction เพื่อจำกัดผลกระทบ
</details>

### แบบฝึกหัดที่ 3

เขียนคำสั่ง SQL เพื่อตรวจสอบว่า query หนึ่งเกิด temp file spill (เพราะ work_mem ไม่พอ) หรือไม่ โดยไม่ต้องไปเปิดดู log file

<details>
<summary>เฉลย</summary>

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM order_items ORDER BY unit_price DESC;
```

ดูที่บรรทัด `Sort Method:` ใน output — ถ้าเป็น `Sort Method: external merge  Disk: XXXkB` แปลว่าเกิด spill ลงดิสก์เพราะ work_mem ไม่พอ ถ้าเป็น `Sort Method: quicksort  Memory: XXXkB` แปลว่าทำใน memory ทั้งหมด ไม่มีปัญหา

นอกจากนี้ยังสามารถเปิด `log_temp_files = 0` (หรือค่า threshold ที่ต้องการ) เพื่อดูทุก temp file ที่เกิดขึ้นใน log ได้อีกทางหนึ่ง
</details>

### แบบฝึกหัดที่ 4

เซิร์ฟเวอร์ใช้ AWS EBS gp3 (cloud block storage แบบ SSD) แต่ `postgresql.conf` ยังใช้ค่า default `random_page_cost = 4.0` จะเกิดผลกระทบอะไรต่อ query plan และควรแก้อย่างไร?

<details>
<summary>เฉลย</summary>

**ผลกระทบ**: planner จะคิดว่าการอ่านแบบ random (index scan) แพงกว่าการอ่านแบบ sequential ถึง 4 เท่า ซึ่งไม่ตรงกับความจริงของ SSD ที่ random ≈ sequential ทำให้ planner เลือก **sequential scan บ่อยเกินความจำเป็น** แม้ query ที่ filter ข้อมูลเฉพาะบางส่วน (selective query) ควรใช้ index scan เพื่อความเร็ว

**วิธีแก้**:

```sql
ALTER SYSTEM SET random_page_cost = 1.1;
SELECT pg_reload_conf();
```

หรือใน `postgresql.conf`:

```ini
random_page_cost = 1.1
```

เป็น `context = user` จึงแค่ `reload` ก็มีผลทันที ไม่ต้อง restart
</details>

### แบบฝึกหัดที่ 5

อธิบายความแตกต่างระหว่าง `checkpoints_timed` และ `checkpoints_req` ใน `pg_stat_bgwriter` และบอกว่าค่าไหนที่บ่งชี้ปัญหา

<details>
<summary>เฉลย</summary>

- `checkpoints_timed` = จำนวน checkpoint ที่เกิดขึ้นเพราะครบเวลา `checkpoint_timeout` (พฤติกรรมปกติ ที่ต้องการ)
- `checkpoints_req` = จำนวน checkpoint ที่เกิดขึ้นเพราะเขียน WAL ถึงเพดาน `max_wal_size` ก่อนครบเวลา (แปลว่า workload เขียนข้อมูลเร็วกว่าที่ `max_wal_size` จะรองรับได้)

ถ้า `checkpoints_req` มีสัดส่วนสูง (ใกล้เคียงหรือมากกว่า `checkpoints_timed`) แปลว่า `max_wal_size` เล็กเกินไปเมื่อเทียบกับอัตราการเขียนจริงของระบบ ควรเพิ่มค่า `max_wal_size` เพื่อลดความถี่ของ checkpoint และลด I/O spike ที่เกิดถี่เกินไป
</details>

### แบบฝึกหัดที่ 6

ระบบมี CPU 8 core และปัจจุบันแอปพลิเคชันมี connection pool รวมกันทุก instance ประมาณ 2,000 connection ควรตั้ง `max_connections` ของ PostgreSQL เท่าไหร่ และต้องมีองค์ประกอบอะไรเพิ่มเติม?

<details>
<summary>เฉลย</summary>

```
max_connections = 8 core × 3-4 ≈ 24-32
```

ตั้ง `max_connections = 40` (เผื่อ headroom เล็กน้อย) ไม่ใช่ 2,000 เพราะ 2,000 connection ที่แอปเปิดไว้ ส่วนใหญ่ idle และไม่ได้ query พร้อมกันจริง

**ต้องมี PgBouncer (หรือ pooler อื่น) อยู่ระหว่างแอปกับ PostgreSQL** โดยใช้ transaction pooling mode เพื่อรับ connection 2,000 ตัวจากแอป แล้ว "ยืม-คืน" connection จริงกับ PostgreSQL จากพูลที่มีขนาดใกล้เคียง `max_connections` (เช่น pool_size = 30-40) — สอดคล้องกับที่เรียนใน Part 066
</details>

### แบบฝึกหัดที่ 7

ทำไม `maintenance_work_mem` ถึงสามารถตั้งค่าสูงกว่า `work_mem` ได้มาก โดยไม่เสี่ยง OOM เท่ากับการตั้ง `work_mem` สูง?

<details>
<summary>เฉลย</summary>

เพราะ `work_mem` ถูกใช้โดยแทบทุก query ที่มี sort/hash/join (เกิดพร้อมกันได้หลายสิบ-หลายร้อยครั้งตาม concurrent connection) ในขณะที่ `maintenance_work_mem` ถูกใช้เฉพาะงานบำรุงรักษา (VACUUM, CREATE INDEX, REINDEX) ที่โดยธรรมชาติ**ไม่ได้รันพร้อมกันจำนวนมาก** — จำนวน autovacuum worker ถูกจำกัดด้วย `autovacuum_max_workers` (ปกติ 3-6 ตัว) และ manual CREATE INDEX มักรันทีละ 1-2 คำสั่งไม่ใช่ร้อยคำสั่งพร้อมกัน จึงคำนวณเพดานหน่วยความจำรวมได้ง่ายกว่าและตั้งสูงได้อย่างปลอดภัยกว่า
</details>

### แบบฝึกหัดที่ 8

เขียนคำสั่ง SQL เพื่อตรวจสอบว่าพารามิเตอร์ `shared_buffers` เปลี่ยนแล้วต้อง restart หรือแค่ reload พอ โดยไม่ต้องไปเดาหรือค้นเอกสารภายนอก

<details>
<summary>เฉลย</summary>

```sql
SELECT name, context
FROM pg_settings
WHERE name = 'shared_buffers';
```

```
      name      |  context
-----------------+------------
 shared_buffers  | postmaster
```

`context = postmaster` หมายความว่าต้อง **restart PostgreSQL service** เท่านั้นถึงจะมีผล (`pg_ctl reload` หรือ `SELECT pg_reload_conf()` ไม่พอ) — สามารถใช้วิธีนี้ตรวจสอบพารามิเตอร์ใดก็ได้ก่อนแก้ไขจริง เพื่อวางแผน maintenance window ให้ถูกต้อง
</details>

### แบบฝึกหัดที่ 9

เซิร์ฟเวอร์มี 32 CPU core ตั้ง `max_worker_processes = 32` และ `max_parallel_workers = 32` พอดีเป๊ะ ปัญหาที่อาจเกิดขึ้นคืออะไร?

<details>
<summary>เฉลย</summary>

`max_worker_processes` คือเพดานสูงสุดของ **background process ทั้งหมด** ไม่ใช่แค่ parallel query worker เท่านั้น แต่ยังรวมถึง:

- Logical replication worker (apply worker, sync worker)
- Extension background worker (เช่น pg_cron, pg_partman)
- Autovacuum launcher/worker บางส่วน (แม้ autovacuum มี pool แยก แต่ก็ยังนับรวมในบาง PostgreSQL version)

ถ้าตั้ง `max_parallel_workers = max_worker_processes` พอดีเป๊ะโดยไม่เผื่อพื้นที่ เมื่อระบบมี parallel query กำลังใช้ worker เต็มเพดานพร้อมกับที่ logical replication หรือ extension ต้องการ worker process เพิ่ม จะทำให้ **ขอ background worker process ไม่สำเร็จ** (worker process ไม่ launch) ส่งผลกระทบต่อ replication หรือ extension นั้น ๆ

**วิธีแก้**: เผื่อ headroom เสมอ เช่น `max_worker_processes = 40` (มากกว่า CPU core) แล้ว `max_parallel_workers = 28-32` เพื่อเหลือพื้นที่ให้ background worker ประเภทอื่นด้วย
</details>

### แบบฝึกหัดที่ 10

ระบบ e-commerce มี RAM 64GB, 24 CPU core, ใช้ NVMe SSD (cloud), รองรับ concurrent connection ผ่าน PgBouncer (pool_size ที่ PostgreSQL เห็นจริงประมาณ 80) และมี workload OLTP เป็นหลักบวกรายงานสรุปยอดขายตอนกลางคืน จงคำนวณและเขียนค่า `shared_buffers`, `effective_cache_size`, `work_mem`, `maintenance_work_mem`, `random_page_cost`, `max_parallel_workers_per_gather` แบบครบถ้วน

<details>
<summary>เฉลย</summary>

```
shared_buffers        = 64GB × 25%       = 16GB
effective_cache_size  = 64GB × 70%       = 44-45GB
work_mem              = (64GB - 16GB - 3GB reserve) / (80 × 3 ops) ≈ 187MB
                         → ตั้ง global แบบอนุรักษ์นิยม 48MB
                         (เพราะ 187MB คือเพดานทฤษฎีสูงสุด ไม่ใช่ค่าที่ควรตั้งจริงสำหรับทุก query
                          ควรเผื่อ margin ไว้สำหรับ peak load และ query ที่ผิดปกติ)
maintenance_work_mem  = 64GB × 5%        ≈ 2-3GB → ตั้ง 2GB
random_page_cost      = 1.1 (NVMe)
max_parallel_workers_per_gather = 4-6 (เครื่องมี 24 core รองรับได้ แต่ยังต้องเผื่อ concurrent OLTP)
```

```ini
# postgresql.conf — 64GB RAM / 24 core / NVMe / OLTP + reporting
shared_buffers = 16GB
effective_cache_size = 44GB
work_mem = 48MB
maintenance_work_mem = 2GB
autovacuum_work_mem = 512MB

wal_buffers = 16MB
checkpoint_timeout = 15min
checkpoint_completion_target = 0.9
max_wal_size = 6GB
min_wal_size = 1GB

max_connections = 100
superuser_reserved_connections = 5

random_page_cost = 1.1
seq_page_cost = 1.0
effective_io_concurrency = 200
maintenance_io_concurrency = 200

max_worker_processes = 32
max_parallel_workers = 20
max_parallel_workers_per_gather = 4
```

สำหรับ query รายงานสรุปยอดขายตอนกลางคืนที่ซับซ้อนกว่า OLTP ทั่วไป แนะนำให้ใช้ session-level override แทนการเพิ่มค่า global:

```sql
BEGIN;
SET LOCAL work_mem = '512MB';
SET LOCAL max_parallel_workers_per_gather = 8;
-- รัน query รายงานยอดขาย ...
COMMIT;
```

วิธีนี้ทำให้ query รายงานได้ resource สูงเฉพาะตอนที่จำเป็น โดยไม่กระทบ OLTP query ที่รันพร้อมกันในช่วงเวลาอื่น
</details>

---

## บทถัดไป

การ tune `postgresql.conf` เป็นการปรับที่ระดับ**เซิร์ฟเวอร์** แต่ประสิทธิภาพของระบบจริงยังขึ้นอยู่กับการเขียน query และออกแบบ index ที่ดีในระดับ**คำสั่ง SQL แต่ละคำสั่ง** ด้วย บทถัดไปจะพาไปเจาะลึกการอ่าน `EXPLAIN`/`EXPLAIN ANALYZE` การหา query ที่ช้าด้วย `pg_stat_statements` และเทคนิคการ tune query ระดับ production

**บทถัดไป**: [Part 074 — Query-Level Performance Tuning](./part-074-query-level-tuning.md)
