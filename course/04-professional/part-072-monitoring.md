# Part 072: Monitoring — pg_stat Views, Prometheus + Grafana, pganalyze

หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับมืออาชีพ | Part 072

---

## เป้าหมายการเรียนรู้

เมื่อเรียนจบบทนี้ ผู้เรียนจะสามารถ:

1. อธิบายความแตกต่างระหว่าง **proactive monitoring** กับ **reactive troubleshooting** และเหตุผลที่ระบบ production ทุกระบบต้องมี monitoring
2. ใช้ `pg_stat_activity` เพื่อดู session ที่กำลังทำงาน ตรวจจับ query ที่ค้างนาน (long-running query) และ query ที่ถูก lock
3. อ่านค่าจาก `pg_stat_database` เพื่อประเมินสุขภาพของฐานข้อมูลระดับ database เช่น transaction rate, cache hit ratio, deadlock count
4. ใช้ `pg_stat_user_tables` และ `pg_stat_user_indexes` เพื่อ monitor การใช้งานตารางและ index อย่างต่อเนื่อง (ต่อยอดจาก Part 043 และ Part 060)
5. คำนวณและตีความ **Cache Hit Ratio** พร้อมทั้งรู้ว่าค่าที่ดีควรอยู่ที่เท่าไร
6. ใช้ `pg_stat_statements` เจาะลึกเพื่อสร้าง dashboard ติดตาม query performance อย่างต่อเนื่อง (ต่อยอดจาก Part 057)
7. เข้าใจสถาปัตยกรรมของ `postgres_exporter` สำหรับ export metric ไปยัง Prometheus
8. เข้าใจภาพรวมสถาปัตยกรรม Prometheus + Grafana และวิธีสร้าง dashboard สำหรับ visualize metric ของ PostgreSQL
9. เปรียบเทียบข้อดี-ข้อเสียของ managed monitoring SaaS อย่าง pganalyze กับการสร้าง monitoring stack เอง (self-hosted)
10. ออกแบบ monitoring stack แบบเต็มรูปแบบ พร้อม alert rule ที่เหมาะสมสำหรับระบบระดับ production เช่น e-commerce

> **หมายเหตุสำคัญเกี่ยวกับเนื้อหาบทนี้**: คำสั่ง SQL ทุกคำสั่งในบทนี้ (`pg_stat_*`) เป็นคำสั่งที่ **รันได้จริง** บน PostgreSQL server และ output ตัวอย่างที่แสดงเป็นข้อมูลจำลองที่ใกล้เคียงของจริง ส่วนเนื้อหาเกี่ยวกับ Prometheus, Grafana, postgres_exporter, และ pganalyze จะเป็น **แนวทางเชิงสถาปัตยกรรม (architectural / conceptual guidance)** — ไฟล์ config และคำสั่งที่แสดงเป็นตัวอย่างมาตรฐานที่ใช้ในอุตสาหกรรมจริง แต่ผู้เรียนต้องปรับให้เข้ากับ environment ของตนเอง (เวอร์ชัน, network, credentials ฯลฯ)

---

## บริบท: ทำไมบทนี้ถึงสำคัญ

ตลอดหลักสูตรที่ผ่านมา เราได้เรียนรู้การออกแบบ schema (Part 010-020), การเขียน query (Part 020-040), การทำ indexing (Part 043, 060), การทำ backup/recovery (Part 061-062), replication (Part 063-064), high availability (Part 065), connection pooling (Part 066-067), roles/security (Part 068-070) และ SSL (Part 070) มาแล้ว

แต่ระบบที่ "ทำงานได้" ในวันแรกที่ deploy ไม่ได้แปลว่าจะ "ทำงานได้ดี" ตลอดไป ข้อมูลเติบโต, pattern การใช้งานเปลี่ยน, index bloat สะสม, connection pool เริ่มเต็ม — สิ่งเหล่านี้ค่อย ๆ กัดกร่อนประสิทธิภาพระบบแบบไม่มีใครสังเกตเห็น จนกระทั่งวันหนึ่งระบบล่มกลางดึกตอนที่ยอดขายพีคที่สุด

**Monitoring คือสิ่งที่ทำให้เรารู้ล่วงหน้า ก่อนที่ผู้ใช้จะรู้**

---

## Step 711: ทำไมต้อง Monitoring — Proactive vs Reactive

### แนวคิดหลัก

มี 2 แนวทางในการดูแลระบบฐานข้อมูล:

| แนวทาง | ความหมาย | ตัวอย่าง |
|---|---|---|
| **Reactive (ตั้งรับ)** | แก้ปัญหา "หลังจาก" มันเกิดขึ้นแล้ว มักรู้ตัวจาก user complaint หรือระบบล่ม | ลูกค้าโทรมาบอกว่าเว็บช้า → DBA ล็อกอินไปดู → พบว่า disk เต็ม 100% → เร่งลบ log |
| **Proactive (เชิงรุก)** | ตรวจจับสัญญาณเตือนล่วงหน้า และแก้ไขก่อนที่ผู้ใช้จะได้รับผลกระทบ | Alert แจ้งเตือนเมื่อ disk usage แตะ 80% → ทีมขยาย storage ล่วงหน้า 2 สัปดาห์ก่อนเต็ม |

### เปรียบเทียบต้นทุนของทั้งสองแนวทาง

```
Reactive Monitoring (ไม่มี monitoring หรือ monitoring ไม่ครอบคลุม)
─────────────────────────────────────────────────────────────────
เวลา 02:00  →  Disk เต็ม, ระบบหยุดรับ write
เวลา 02:15  →  ลูกค้า/on-call เริ่มได้รับ error, แจ้งเตือนผ่าน support
เวลา 02:45  →  DBA ตื่นมาดู, เริ่มวินิจฉัยปัญหา (ไม่รู้สาเหตุล่วงหน้า)
เวลา 03:30  →  หาสาเหตุเจอ (เช่น WAL ล้นเพราะ replication slot ค้าง)
เวลา 04:00  →  แก้ปัญหาเสร็จ, ระบบกลับมาใช้งานได้

รวมเวลา Downtime: ~2 ชั่วโมง | Impact: สูญเสียยอดขาย, ความเชื่อมั่นลูกค้า

Proactive Monitoring (มี monitoring + alerting ที่ดี)
─────────────────────────────────────────────────────────────────
วันที่ -14  →  Alert: "Disk usage 75% and growing 2%/day"
วันที่ -13  →  ทีมตรวจสอบพบ replication slot ค้างจาก server ที่ปิดไปแล้ว
วันที่ -13  →  ลบ replication slot ที่ไม่ใช้แล้ว, WAL เริ่มลดลงปกติ
วันที่ -13  →  จบ (ไม่มี downtime เกิดขึ้นเลย)

รวมเวลา Downtime: 0 ชั่วโมง | Impact: ไม่มี
```

### เสาหลักของ Monitoring ที่ดี (The Four Golden Signals ปรับใช้กับ Database)

1. **Latency** — query ใช้เวลานานแค่ไหน (เช่นจาก `pg_stat_statements`)
2. **Traffic** — มีกี่ transaction/connection ต่อวินาที (`pg_stat_database`)
3. **Errors** — มี deadlock, rollback, connection refused มากแค่ไหน
4. **Saturation** — resource ใกล้เต็มแค่ไหน (connection pool, disk, cache hit ratio ต่ำ)

### สิ่งที่ต้อง monitor ในระบบ PostgreSQL production

```
┌─────────────────────────────────────────────────────────────┐
│                     PostgreSQL Monitoring Layers               │
├─────────────────────────────────────────────────────────────┤
│ 1. OS-level      : CPU, Memory, Disk I/O, Network              │
│ 2. PostgreSQL     : Connections, Cache Hit, Locks, Replication │
│ 3. Query-level    : Slow queries, query plans (pg_stat_statements) │
│ 4. Application     : Error rate, response time (จาก app เอง)    │
│ 5. Business        : Order/min, Revenue/hour (business metric)  │
└─────────────────────────────────────────────────────────────┘
```

บทนี้จะเน้นที่ layer 2 และ 3 เป็นหลัก ซึ่งเป็นส่วนที่ PostgreSQL เก็บข้อมูลให้เราโดยอัตโนมัติผ่าน **statistics collector**

### คำสั่งพื้นฐานที่ควรรู้ก่อนเริ่ม

PostgreSQL เก็บสถิติการทำงานไว้ใน view ตระกูล `pg_stat_*` และ `pg_statio_*` โดยอัตโนมัติ (เปิดใช้งานเป็น default) เราสามารถ query ดูได้ทันทีโดยไม่ต้องติดตั้งอะไรเพิ่ม:

```sql
-- ตรวจสอบว่า statistics collector เปิดใช้งานอยู่หรือไม่
SHOW track_activities;
SHOW track_counts;
SHOW track_io_timing;
```

**ตัวอย่างผลลัพธ์:**
```
 track_activities
-------------------
 on
(1 row)

 track_counts
---------------
 on
(1 row)

 track_io_timing
------------------
 off
(1 row)
```

> **หมายเหตุ:** `track_io_timing = off` เป็นค่า default ในหลาย distro แต่แนะนำให้เปิดในระบบ production เพื่อให้ `pg_stat_statements` และ `pg_stat_database` เก็บเวลาที่ใช้ในการอ่าน/เขียน I/O ได้ละเอียดขึ้น (มี overhead เล็กน้อย แต่คุ้มค่าในการ monitor)

```sql
-- เปิด track_io_timing (ต้อง superuser และอาจต้อง restart หรือ reload)
ALTER SYSTEM SET track_io_timing = on;
SELECT pg_reload_conf();
```

---

## Step 712: pg_stat_activity — ดู Session ที่กำลังทำงานอยู่

### `pg_stat_activity` คืออะไร

`pg_stat_activity` เป็น view ที่แสดง **snapshot แบบ real-time** ของทุก connection/process ที่เชื่อมต่อกับ PostgreSQL server ในขณะนั้น เป็นเครื่องมือชิ้นแรกที่ DBA ทุกคนเปิดดูเมื่อสงสัยว่าระบบทำงานผิดปกติ

### คำสั่งพื้นฐาน

```sql
-- ดูทุก session ที่กำลัง active
SELECT
    pid,
    usename,
    application_name,
    client_addr,
    state,
    query,
    query_start
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY query_start;
```

**ตัวอย่างผลลัพธ์:**
```
  pid  |  usename  | application_name |  client_addr  | state  |                     query                      |          query_start
-------+-----------+-------------------+----------------+--------+------------------------------------------------+-------------------------------
 18422 | app_user  | order-service     | 10.0.1.15      | active | SELECT * FROM orders WHERE customer_id = 8821  | 2026-09-25 09:12:03.128+07
 18455 | app_user  | payment-service   | 10.0.1.22      | active | UPDATE payments SET status = 'completed' WHERE...| 2026-09-25 09:12:04.771+07
 18470 | report_user| reporting-job    | 10.0.2.5       | active | SELECT sum(amount) FROM order_items JOIN order...| 2026-09-25 09:09:12.004+07
(3 rows)
```

สังเกตว่า session `pid=18470` เริ่ม query ตั้งแต่ 09:09:12 แต่ตอนนี้เวลาผ่านไปหลายนาทีแล้วยัง active อยู่ — นี่คือสัญญาณของ **long-running query** ที่เราต้องเฝ้าดู

### คอลัมน์สำคัญใน `pg_stat_activity`

| คอลัมน์ | ความหมาย |
|---|---|
| `pid` | process ID ของ backend process (ใช้สำหรับ `pg_terminate_backend()` หรือ `pg_cancel_backend()`) |
| `usename` | ชื่อ role/user ที่เชื่อมต่อ |
| `application_name` | ชื่อ application ที่ตั้งค่ามาจาก connection string (`application_name=...`) |
| `client_addr` | IP ของ client ที่เชื่อมต่อเข้ามา |
| `backend_start` | เวลาที่ connection นี้ถูกสร้างขึ้น |
| `xact_start` | เวลาที่ transaction ปัจจุบันเริ่มต้น (สำคัญมากสำหรับหา long transaction) |
| `query_start` | เวลาที่ query ปัจจุบันเริ่มรัน |
| `state` | สถานะของ backend: `active`, `idle`, `idle in transaction`, `idle in transaction (aborted)`, `fastpath function call`, `disabled` |
| `wait_event_type` / `wait_event` | ถ้า backend กำลังรออะไรอยู่ (เช่น lock, I/O) จะบอกใน 2 คอลัมน์นี้ |
| `query` | ข้อความ SQL ล่าสุดที่ backend นี้รัน (หรือกำลังรันอยู่) |

### การหา Query ที่ค้างนาน (Long-Running Query)

นี่คือ query ที่ DBA ใช้บ่อยที่สุด — หา query ที่รันนานเกินไป:

```sql
-- หา query ที่ทำงานนานเกิน 5 นาที
SELECT
    pid,
    usename,
    now() - query_start AS duration,
    state,
    query
FROM pg_stat_activity
WHERE state != 'idle'
  AND now() - query_start > interval '5 minutes'
ORDER BY duration DESC;
```

**ตัวอย่างผลลัพธ์:**
```
  pid  |   usename    |    duration     | state  |                        query
-------+--------------+------------------+--------+-------------------------------------------------------
 18470 | report_user  | 00:12:34.881204  | active | SELECT sum(amount) FROM order_items JOIN order...
 19102 | batch_user   | 00:07:02.331005  | active | UPDATE inventory SET stock = stock - 1 WHERE prod...
(2 rows)
```

### การหา Transaction ที่ค้างนาน (Long-Running Transaction)

Long-running transaction อันตรายกว่า long-running query เพราะมันจะทำให้ VACUUM ไม่สามารถลบ dead tuple ได้ (ย้อนกลับไปดู Part เรื่อง VACUUM) และอาจทำให้เกิด table bloat สะสม:

```sql
-- หา transaction ที่เปิดค้างไว้นานเกิน 10 นาที (รวมถึง idle in transaction)
SELECT
    pid,
    usename,
    state,
    now() - xact_start AS transaction_duration,
    query
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
  AND now() - xact_start > interval '10 minutes'
ORDER BY transaction_duration DESC;
```

**ตัวอย่างผลลัพธ์:**
```
  pid  |  usename  |        state        | transaction_duration |                query
-------+-----------+----------------------+------------------------+---------------------------------
 20031 | app_user  | idle in transaction  | 00:45:12.220331        | SELECT * FROM carts WHERE id = 91
(1 row)
```

> **สัญญาณอันตราย:** `state = 'idle in transaction'` ที่ค้างนานมาก มักเกิดจาก bug ในโค้ด application ที่เปิด transaction แล้วลืม commit/rollback (เช่น connection leak หรือรอ user input ระหว่าง transaction เปิดอยู่) ควรตั้ง alert สำหรับกรณีนี้โดยเฉพาะ

### การยกเลิก Query หรือ Terminate Session

เมื่อพบ query/session ที่มีปัญหา เรามี 2 คำสั่งให้ใช้:

```sql
-- ยกเลิกเฉพาะ query ที่กำลังรัน (transaction ยังอยู่ ยกเลิกแค่ statement)
SELECT pg_cancel_backend(18470);

-- ตัด connection ทิ้งทั้งหมด (ใช้เมื่อ cancel ไม่ได้ผล หรือ session ค้างแบบ idle in transaction)
SELECT pg_terminate_backend(20031);
```

**ตัวอย่างผลลัพธ์:**
```
 pg_cancel_backend
--------------------
 t
(1 row)
```

`t` แปลว่าคำสั่งถูกส่งไปสำเร็จ (แต่ backend อาจใช้เวลาสักครู่ในการยกเลิกจริง ๆ)

### การดู Lock ที่เกี่ยวข้องกับ Session

เมื่อ query ค้าง มักเกิดจากการรอ lock จาก session อื่น เราสามารถ join `pg_stat_activity` กับ `pg_locks` เพื่อดูว่าใครกำลังบล็อกใคร:

```sql
-- หา session ที่กำลังถูกบล็อก (blocked) และตัวที่บล็อก (blocking)
SELECT
    blocked.pid AS blocked_pid,
    blocked.usename AS blocked_user,
    blocked.query AS blocked_query,
    blocking.pid AS blocking_pid,
    blocking.usename AS blocking_user,
    blocking.query AS blocking_query
FROM pg_stat_activity AS blocked
JOIN pg_locks AS blocked_locks
    ON blocked_locks.pid = blocked.pid
JOIN pg_locks AS blocking_locks
    ON blocking_locks.locktype = blocked_locks.locktype
   AND blocking_locks.database IS NOT DISTINCT FROM blocked_locks.database
   AND blocking_locks.relation IS NOT DISTINCT FROM blocked_locks.relation
   AND blocking_locks.pid != blocked_locks.pid
   AND blocking_locks.granted
JOIN pg_stat_activity AS blocking
    ON blocking.pid = blocking_locks.pid
WHERE NOT blocked_locks.granted;
```

**ตัวอย่างผลลัพธ์:**
```
 blocked_pid | blocked_user |          blocked_query           | blocking_pid | blocking_user |          blocking_query
-------------+--------------+-----------------------------------+---------------+----------------+------------------------------------
       21044 | app_user     | UPDATE orders SET status = 'ship'| 20031         | app_user       | UPDATE orders SET status = 'pack'
(1 row)
```

จากผลลัพธ์นี้เราเห็นชัดว่า `pid=20031` กำลังถือ lock อยู่และทำให้ `pid=21044` ต้องรอ — ถ้า `20031` เป็น session ที่ค้าง เราสามารถใช้ `pg_terminate_backend(20031)` เพื่อปลดบล็อกได้ทันที

### สรุปคำสั่งสำคัญ Step 712

```sql
-- Dashboard สรุป session ทั้งหมดแบ่งตาม state
SELECT state, count(*) AS session_count
FROM pg_stat_activity
GROUP BY state
ORDER BY session_count DESC;
```

**ตัวอย่างผลลัพธ์:**
```
        state        | session_count
----------------------+----------------
 idle                 |            42
 active               |             8
 idle in transaction  |             3
 (null)                |             2
(4 rows)
```

Dashboard แบบนี้ควร monitor ต่อเนื่อง — ถ้า `idle in transaction` เพิ่มขึ้นเรื่อย ๆ คือสัญญาณเตือนว่ามี connection leak

---

## Step 713: pg_stat_database — สถิติระดับฐานข้อมูล

### `pg_stat_database` คืออะไร

`pg_stat_database` เก็บสถิติสะสม (cumulative counter) ในระดับ **database ทั้งก้อน** ตั้งแต่ PostgreSQL server เริ่มทำงาน (หรือตั้งแต่ counter ถูก reset ล่าสุด) เหมาะสำหรับดูภาพรวม เช่น throughput, cache hit ratio, deadlock count

### คำสั่งพื้นฐาน

```sql
SELECT
    datname,
    numbackends,
    xact_commit,
    xact_rollback,
    blks_read,
    blks_hit,
    tup_returned,
    tup_fetched,
    tup_inserted,
    tup_updated,
    tup_deleted,
    conflicts,
    deadlocks,
    temp_files,
    temp_bytes,
    stats_reset
FROM pg_stat_database
WHERE datname = 'ecommerce_prod';
```

**ตัวอย่างผลลัพธ์:**
```
    datname     | numbackends | xact_commit | xact_rollback | blks_read |  blks_hit  | tup_returned | tup_fetched | tup_inserted | tup_updated | tup_deleted | conflicts | deadlocks | temp_files | temp_bytes |         stats_reset
-----------------+--------------+--------------+-----------------+------------+-------------+----------------+---------------+----------------+---------------+---------------+------------+------------+-------------+-------------+-------------------------------
 ecommerce_prod |          53 |    18442091 |           1204 |    892011 | 481293841 |    2109332211 |  1889321044 |       912304 |      2841022 |       19204 |          0 |         2 |         14 |  120942848 | 2026-08-01 03:00:00.112+07
(1 row)
```

### คอลัมน์สำคัญและวิธีอ่านค่า

| คอลัมน์ | ความหมาย | สิ่งที่ควรสังเกต |
|---|---|---|
| `numbackends` | จำนวน connection ปัจจุบันที่เชื่อมกับ database นี้ | ใกล้ `max_connections` แปลว่าเสี่ยง connection exhaustion |
| `xact_commit` | จำนวน transaction ที่ commit สำเร็จ (สะสม) | ใช้คำนวณ TPS (transactions per second) |
| `xact_rollback` | จำนวน transaction ที่ rollback (สะสม) | ถ้าสัดส่วนสูงเทียบกับ commit แปลว่า application มี error บ่อย |
| `blks_read` | จำนวน block ที่อ่านจาก disk | ใช้คำนวณ cache hit ratio (ดู Step 715) |
| `blks_hit` | จำนวน block ที่อ่านเจอใน shared_buffers (ไม่ต้องอ่าน disk) | ยิ่งสูงเทียบกับ `blks_read` ยิ่งดี |
| `deadlocks` | จำนวนครั้งที่เกิด deadlock (สะสม) | ควรเป็น 0 หรือใกล้ 0; ถ้าเพิ่มขึ้นเรื่อย ๆ ต้องตรวจสอบ transaction logic |
| `conflicts` | จำนวน query ที่ถูกยกเลิกเพราะ conflict กับ recovery (มักเจอใน replica) | สำคัญมากบน read replica |
| `temp_files` / `temp_bytes` | จำนวนครั้ง/ขนาดที่ query ต้องใช้ temp file บน disk (เพราะ `work_mem` ไม่พอ) | ถ้าสูงมาก อาจต้องเพิ่ม `work_mem` |
| `stats_reset` | เวลาที่ counter ถูก reset ล่าสุด | สำคัญมากเพื่อรู้ว่าตัวเลขสะสมมาตั้งแต่เมื่อไร |

### คำนวณ TPS (Transactions Per Second) แบบ real-time

เนื่องจาก `xact_commit` เป็นค่าสะสม เราต้อง query 2 ครั้งห่างกันเพื่อคำนวณ rate:

```sql
-- ครั้งที่ 1: บันทึกค่าปัจจุบัน
SELECT xact_commit, xact_rollback, now() AS ts
FROM pg_stat_database
WHERE datname = 'ecommerce_prod';
```
```
 xact_commit | xact_rollback |              ts
--------------+-----------------+-------------------------------
    18442091 |            1204 | 2026-09-25 10:00:00.000000+07
(1 row)
```

```sql
-- รอ 60 วินาที แล้ว query อีกครั้ง
SELECT xact_commit, xact_rollback, now() AS ts
FROM pg_stat_database
WHERE datname = 'ecommerce_prod';
```
```
 xact_commit | xact_rollback |              ts
--------------+-----------------+-------------------------------
    18449891 |            1211 | 2026-09-25 10:01:00.000000+07
(1 row)
```

จากตัวอย่าง: `(18449891 - 18442091) / 60 = 130 TPS` โดยประมาณ — ในทางปฏิบัติ เราจะใช้ Prometheus + `postgres_exporter` เพื่อคำนวณ rate นี้อัตโนมัติแบบ real-time (อธิบายใน Step 717-718)

### สรุปทุก Database ในเครื่องเดียว

```sql
-- ดูภาพรวมทุก database พร้อมคำนวณ rollback ratio
SELECT
    datname,
    numbackends,
    xact_commit,
    xact_rollback,
    round(
        100.0 * xact_rollback / NULLIF(xact_commit + xact_rollback, 0), 2
    ) AS rollback_pct,
    deadlocks,
    temp_files
FROM pg_stat_database
WHERE datname NOT IN ('template0', 'template1', 'postgres')
ORDER BY xact_commit DESC;
```

**ตัวอย่างผลลัพธ์:**
```
     datname      | numbackends | xact_commit | xact_rollback | rollback_pct | deadlocks | temp_files
-------------------+--------------+--------------+-----------------+---------------+------------+-------------
 ecommerce_prod   |          53 |    18442091 |           1204 |          0.01 |        14 |          2
 analytics_dw     |           4 |      982104 |             89 |          0.01 |         0 |        891
(2 rows)
```

สังเกตว่า `analytics_dw` มี `temp_files = 891` สูงผิดปกติเมื่อเทียบกับ transaction count — บ่งชี้ว่า workload แบบ analytical query มักต้อง sort/hash ข้อมูลจำนวนมากเกินกว่าที่ `work_mem` จะรองรับได้ใน memory จึงต้อง spill ลง disk บ่อย

---

## Step 714: pg_stat_user_tables / pg_stat_user_indexes — เจาะลึกเพื่อ Monitoring

> ใน **Part 043 (Indexing พื้นฐาน)** และ **Part 060 (Index เชิงลึก)** เราเคยใช้ view เหล่านี้เพื่อ "วิเคราะห์" การออกแบบ index ในบทนี้เราจะกลับมาใช้ view เดิม แต่เปลี่ยนมุมมองเป็นการ **monitor อย่างต่อเนื่อง** เพื่อตรวจจับปัญหาที่ค่อย ๆ ก่อตัวขึ้น เช่น table bloat, sequential scan ที่มากผิดปกติ, หรือ index ที่ไม่เคยถูกใช้

### pg_stat_user_tables — สุขภาพของตาราง

```sql
SELECT
    schemaname,
    relname AS table_name,
    seq_scan,
    seq_tup_read,
    idx_scan,
    idx_tup_fetch,
    n_tup_ins,
    n_tup_upd,
    n_tup_del,
    n_live_tup,
    n_dead_tup,
    last_vacuum,
    last_autovacuum,
    last_analyze,
    last_autoanalyze
FROM pg_stat_user_tables
WHERE schemaname = 'public'
ORDER BY seq_scan DESC
LIMIT 5;
```

**ตัวอย่างผลลัพธ์:**
```
 schemaname | table_name | seq_scan | seq_tup_read | idx_scan | idx_tup_fetch | n_tup_ins | n_tup_upd | n_tup_del | n_live_tup | n_dead_tup |     last_vacuum      |     last_autovacuum      | last_analyze |     last_autoanalyze
-------------+-------------+-----------+----------------+-----------+-----------------+------------+------------+------------+-------------+-------------+------------------------+----------------------------+---------------+----------------------------
 public     | order_logs |    89211 |     1220984 |     4021 |        142000 |    920100 |          0 |          0 |    2109221 |     421002 | (null)                | 2026-09-24 22:01:12.003+07 | (null)       | 2026-09-24 22:05:00.120+07
 public     | orders     |      412 |         8221 |  1092011 |      9821004 |    182004 |     92104 |       1200 |     982104 |        8021 | 2026-09-20 03:00:00+07| 2026-09-25 02:00:00.331+07 | 2026-09-20    | 2026-09-25 02:05:00.221+07
(2 rows)
```

### สัญญาณเตือนที่ต้องจับตา

**1. `seq_scan` สูงผิดปกติเทียบกับ `idx_scan`**

ตาราง `order_logs` ในตัวอย่างมี `seq_scan = 89211` แต่ `idx_scan` เพียง `4021` — สัดส่วนนี้บ่งชี้ว่า query ส่วนใหญ่ที่เข้าถึงตารางนี้ทำ **full table scan** แทนที่จะใช้ index ซึ่งอาจเกิดจาก:
- ไม่มี index ที่เหมาะกับ query pattern (ต้องกลับไปดู `pg_stat_statements` เพื่อหา query ที่เป็นต้นเหตุ)
- Table เล็กเกินไปจน planner เลือก seq scan เอง (ในกรณีนี้ไม่ใช่ปัญหา)
- Query ใช้ WHERE clause ที่ไม่สามารถใช้ index ได้ เช่น `WHERE lower(email) = ...` แต่ไม่มี expression index

```sql
-- คำนวณสัดส่วน sequential scan ต่อ index scan ของทุกตาราง เพื่อจัดลำดับความสำคัญ
SELECT
    relname AS table_name,
    seq_scan,
    idx_scan,
    n_live_tup,
    round(100.0 * seq_scan / NULLIF(seq_scan + idx_scan, 0), 2) AS seq_scan_pct
FROM pg_stat_user_tables
WHERE n_live_tup > 1000     -- กรองตารางเล็กออก เพราะ seq scan บนตารางเล็กไม่ใช่ปัญหา
ORDER BY seq_scan_pct DESC, n_live_tup DESC
LIMIT 10;
```

**ตัวอย่างผลลัพธ์:**
```
 table_name | seq_scan | idx_scan | n_live_tup | seq_scan_pct
-------------+-----------+-----------+-------------+---------------
 order_logs |    89211 |      4021 |     2109221 |         95.69
 audit_trail|    12094 |       120 |      892104 |         99.02
(2 rows)
```

**2. `n_dead_tup` สูงเทียบกับ `n_live_tup` (Table Bloat)**

```sql
-- หาตารางที่มี dead tuple สะสมมาก (ทบทวนจาก Part เรื่อง VACUUM)
SELECT
    relname AS table_name,
    n_live_tup,
    n_dead_tup,
    round(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 2) AS dead_tup_pct,
    last_autovacuum
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000
ORDER BY dead_tup_pct DESC
LIMIT 10;
```

**ตัวอย่างผลลัพธ์:**
```
 table_name | n_live_tup | n_dead_tup | dead_tup_pct |      last_autovacuum
-------------+-------------+-------------+----------------+------------------------------
 sessions   |      42104 |      98211 |          69.98 | 2026-09-10 00:00:00.001+07
 order_logs |     2109221 |      421002 |          16.63 | 2026-09-24 22:01:12.003+07
(2 rows)
```

ตาราง `sessions` มี `dead_tup_pct = 69.98%` และ `last_autovacuum` ห่างไปเกือบ 2 สัปดาห์ — บ่งชี้ว่า autovacuum ไม่สามารถตามทันอัตราการเปลี่ยนแปลงข้อมูล อาจต้องปรับ `autovacuum_vacuum_scale_factor` ให้ aggressive ขึ้นสำหรับตารางนี้โดยเฉพาะ (`ALTER TABLE sessions SET (autovacuum_vacuum_scale_factor = 0.02)`)

### pg_stat_user_indexes — สุขภาพของ Index

```sql
SELECT
    schemaname,
    relname AS table_name,
    indexrelname AS index_name,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
ORDER BY idx_scan ASC
LIMIT 10;
```

**ตัวอย่างผลลัพธ์:**
```
 schemaname | table_name |            index_name             | idx_scan | idx_tup_read | idx_tup_fetch | index_size
-------------+-------------+------------------------------------+-----------+----------------+-----------------+-------------
 public     | orders     | idx_orders_legacy_status           |        0 |              0 |               0 | 892 MB
 public     | products   | idx_products_deprecated_sku_alt    |        0 |              0 |               0 | 340 MB
 public     | orders     | idx_orders_customer_id             |  1092011 |        9821004 |         9820900 | 2145 MB
(3 rows)
```

### หา Index ที่ไม่เคยถูกใช้เลย (Unused Index)

Index ที่ `idx_scan = 0` คือ index ที่ **ไม่เคยถูกใช้เลยตั้งแต่ statistics ถูก reset ล่าสุด** — index เหล่านี้กิน storage และทำให้ INSERT/UPDATE ช้าลงโดยไม่ได้ประโยชน์อะไรกลับมา:

```sql
-- หา index ที่ไม่เคยถูกใช้ เรียงตามขนาดจากใหญ่ไปเล็ก (ตัวที่ควรพิจารณาลบก่อน)
SELECT
    schemaname,
    relname AS table_name,
    indexrelname AS index_name,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size,
    idx_scan
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND indexrelname NOT LIKE '%_pkey'   -- อย่าลืมกันการนับ primary key ที่ยังไม่เคยถูก scan โดยตรง
ORDER BY pg_relation_size(indexrelid) DESC;
```

**ตัวอย่างผลลัพธ์:**
```
 schemaname | table_name |            index_name             | index_size | idx_scan
-------------+-------------+------------------------------------+-------------+-----------
 public     | orders     | idx_orders_legacy_status           | 892 MB     |        0
 public     | products   | idx_products_deprecated_sku_alt    | 340 MB     |        0
(2 rows)
```

> **ข้อควรระวัง:** ก่อนลบ index ที่ `idx_scan = 0` ต้องตรวจสอบว่า statistics ไม่ได้เพิ่งถูก reset (ดูจาก `pg_stat_database.stats_reset`) และต้องแน่ใจว่า index นั้นไม่ได้ใช้เพื่อ enforce `UNIQUE constraint` หรือ `EXCLUDE constraint` ที่จำเป็นต่อความถูกต้องของข้อมูล แม้จะไม่เคยถูกใช้ในการ query ก็ตาม นอกจากนี้ index ที่ query rarely (เช่น รันปีละครั้งสำหรับ batch job) อาจแสดง `idx_scan = 0` ได้เช่นกันถ้าเพิ่ง reset ค่าไปเมื่อไม่นาน จึงควรสังเกตต่อเนื่องเป็นระยะเวลานานพอสมควร (เช่น 30 วันขึ้นไป) ก่อนตัดสินใจลบจริง

### รวม Dashboard ตรวจสุขภาพ Table + Index ในคิวรีเดียว

```sql
-- Dashboard รวม: ตารางไหนต้องการความสนใจมากที่สุด
SELECT
    t.relname AS table_name,
    pg_size_pretty(pg_total_relation_size(t.relid)) AS total_size,
    t.n_live_tup,
    t.n_dead_tup,
    round(100.0 * t.n_dead_tup / NULLIF(t.n_live_tup + t.n_dead_tup, 0), 2) AS dead_pct,
    t.seq_scan,
    t.idx_scan,
    t.last_autovacuum,
    t.last_autoanalyze
FROM pg_stat_user_tables t
ORDER BY pg_total_relation_size(t.relid) DESC
LIMIT 10;
```

**ตัวอย่างผลลัพธ์:**
```
 table_name | total_size | n_live_tup | n_dead_tup | dead_pct | seq_scan | idx_scan |      last_autovacuum      |     last_autoanalyze
-------------+-------------+-------------+-------------+------------+-----------+-----------+------------------------------+------------------------------
 order_logs |     3821 MB|     2109221 |      421002 |     16.63 |    89211 |     4021 | 2026-09-24 22:01:12.003+07 | 2026-09-24 22:05:00.120+07
 orders     |     4102 MB|      982104 |        8021 |      0.81 |      412 |  1092011 | 2026-09-25 02:00:00.331+07 | 2026-09-25 02:05:00.221+07
(2 rows)
```

---

## Step 715: Cache Hit Ratio — การคำนวณและความสำคัญ

### Cache Hit Ratio คืออะไร

**Cache Hit Ratio** คือสัดส่วนของครั้งที่ PostgreSQL สามารถอ่านข้อมูลจาก **shared_buffers (memory)** ได้โดยตรง เทียบกับจำนวนครั้งทั้งหมดที่ต้องอ่านข้อมูล (ทั้งจาก memory และจาก disk) นี่คือหนึ่งใน metric ที่สำคัญที่สุดในการวัดประสิทธิภาพของ PostgreSQL เพราะการอ่านจาก memory เร็วกว่าการอ่านจาก disk เป็นพัน ๆ เท่า

### สูตรคำนวณ

```
Cache Hit Ratio = blks_hit / (blks_hit + blks_read)
```

### คำนวณระดับ Database

```sql
SELECT
    datname,
    blks_hit,
    blks_read,
    round(
        100.0 * blks_hit / NULLIF(blks_hit + blks_read, 0), 2
    ) AS cache_hit_ratio_pct
FROM pg_stat_database
WHERE datname NOT IN ('template0', 'template1')
ORDER BY cache_hit_ratio_pct ASC;
```

**ตัวอย่างผลลัพธ์:**
```
     datname      | blks_hit  | blks_read | cache_hit_ratio_pct
-------------------+------------+------------+------------------------
 analytics_dw     |   9821004 |    892104 |                 91.68
 ecommerce_prod   | 481293841 |    892011 |                 99.82
(2 rows)
```

### เกณฑ์การตีความ (Threshold Guideline)

| Cache Hit Ratio | สถานะ | คำแนะนำ |
|---|---|---|
| **≥ 99%** | ดีเยี่ยม | เป็นเป้าหมายมาตรฐานสำหรับระบบ OLTP (transactional) ส่วนใหญ่ |
| **95–99%** | ยอมรับได้ | ควรเฝ้าดู แนวโน้ม ถ้าลดลงเรื่อย ๆ ควรตรวจสอบ |
| **90–95%** | ควรปรับปรุง | อาจต้องเพิ่ม `shared_buffers` หรือปรับ query ที่อ่านข้อมูลจำนวนมากเกินจำเป็น |
| **< 90%** | มีปัญหา | ตรวจสอบ working set กับขนาด `shared_buffers` อย่างเร่งด่วน หรือ workload เป็นแบบ analytical ที่คาดหวังค่าต่ำโดยธรรมชาติ |

> **ข้อควรระวังในการตีความ:** ตัวเลข analytical workload (data warehouse, reporting) มักมี cache hit ratio ต่ำกว่า OLTP โดยธรรมชาติ เพราะ query ต้อง scan ข้อมูลจำนวนมากที่ไม่ได้อยู่ใน cache อยู่แล้ว (เช่น full table scan บนตาราง fact ขนาดใหญ่) ดังนั้นค่า < 95% ใน analytics database **ไม่ได้แปลว่ามีปัญหาเสมอไป** ต้องดูบริบทของ workload ประกอบด้วย

### คำนวณ Cache Hit Ratio ระดับตาราง (Table-level)

บางครั้งภาพรวมของทั้ง database ดูดี แต่ตารางบางตัวมีปัญหาเฉพาะจุด เราสามารถดูรายละเอียดผ่าน `pg_statio_user_tables`:

```sql
SELECT
    relname AS table_name,
    heap_blks_read,
    heap_blks_hit,
    round(
        100.0 * heap_blks_hit / NULLIF(heap_blks_hit + heap_blks_read, 0), 2
    ) AS table_cache_hit_pct
FROM pg_statio_user_tables
WHERE heap_blks_read + heap_blks_hit > 0
ORDER BY table_cache_hit_pct ASC
LIMIT 10;
```

**ตัวอย่างผลลัพธ์:**
```
 table_name |  heap_blks_read | heap_blks_hit | table_cache_hit_pct
-------------+-------------------+-----------------+------------------------
 order_logs |           420112 |       1200884 |                 74.08
 orders     |             8021 |     98210441 |                 99.99
(2 rows)
```

ตาราง `order_logs` มี cache hit ratio ต่ำเพียง 74.08% ทั้งที่ database โดยรวมสูงถึง 99.82% — บ่งชี้ว่าตารางนี้มีขนาดใหญ่เกินกว่าจะ fit ใน `shared_buffers` ได้ทั้งหมด (เช่นเป็นตาราง log ที่โตเร็วมาก) และควรพิจารณาแยกไปเก็บใน storage อื่น หรือทำ partitioning + archiving ข้อมูลเก่าออก (ทบทวนแนวคิด partitioning จากบทก่อนหน้า)

### คำนวณ Index Cache Hit Ratio

```sql
SELECT
    relname AS table_name,
    indexrelname AS index_name,
    idx_blks_read,
    idx_blks_hit,
    round(
        100.0 * idx_blks_hit / NULLIF(idx_blks_hit + idx_blks_read, 0), 2
    ) AS index_cache_hit_pct
FROM pg_statio_user_indexes
WHERE idx_blks_read + idx_blks_hit > 0
ORDER BY index_cache_hit_pct ASC
LIMIT 5;
```

**ตัวอย่างผลลัพธ์:**
```
 table_name |         index_name          | idx_blks_read | idx_blks_hit | index_cache_hit_pct
-------------+-------------------------------+-----------------+----------------+------------------------
 order_logs | idx_order_logs_created_at   |           9821 |        21004 |                 68.14
(1 row)
```

### เหตุผลที่ Cache Hit Ratio สำคัญมาก

1. **สะท้อน I/O latency โดยอ้อม** — ถ้า cache hit ratio ต่ำ แปลว่า PostgreSQL ต้องเข้า disk บ่อย ซึ่งมี latency สูงกว่าการอ่าน memory มาก (แม้เป็น NVMe SSD ก็ยังช้ากว่า RAM หลายเท่า)
2. **บ่งชี้ว่า `shared_buffers` ตั้งค่าเหมาะสมหรือไม่** — ถ้า working set (ข้อมูลที่ query เข้าถึงบ่อย) ใหญ่กว่า `shared_buffers` มาก cache hit ratio จะตกต่ำลงเรื่อย ๆ เมื่อข้อมูลโต
3. **เป็น leading indicator** — cache hit ratio มักลดลง **ก่อน** ที่ query latency จะแย่ลงอย่างเห็นได้ชัด การจับตาดู metric นี้จึงเป็นการ monitor เชิง proactive ตามที่กล่าวใน Step 711

---

## Step 716: pg_stat_statements เจาะลึกสำหรับ Dashboard Monitoring ต่อเนื่อง

> ใน **Part 057** เราได้เรียนรู้การใช้ `pg_stat_statements` เพื่อ "หา" query ที่ช้าที่สุดแบบครั้งเดียว (one-time analysis) ในบทนี้เราจะขยายไปสู่การใช้งานแบบ **dashboard ที่ monitor ต่อเนื่อง** เพื่อจับความผิดปกติแบบ real-time

### ทบทวน: เปิดใช้งาน pg_stat_statements

```sql
-- ตรวจสอบว่า extension ถูกติดตั้งแล้วหรือยัง
SELECT * FROM pg_extension WHERE extname = 'pg_stat_statements';
```

```
      extname       | extversion
----------------------+-------------
 pg_stat_statements  | 1.10
(1 row)
```

หากยังไม่มี ต้องเพิ่ม `shared_preload_libraries = 'pg_stat_statements'` ใน `postgresql.conf` แล้ว restart server ก่อน จากนั้นจึงรัน `CREATE EXTENSION pg_stat_statements;`

### Query Top-N ที่ใช้เวลารวมมากที่สุด (Total Time)

สำหรับ dashboard monitoring เราควรดูทั้ง **total_exec_time** (ผลกระทบต่อระบบโดยรวม) และ **mean_exec_time** (ประสบการณ์ต่อ 1 ครั้งของผู้ใช้) แยกกัน:

```sql
SELECT
    round(total_exec_time::numeric, 2) AS total_ms,
    calls,
    round(mean_exec_time::numeric, 2) AS mean_ms,
    round((100 * total_exec_time / sum(total_exec_time) OVER ())::numeric, 2) AS pct_of_total,
    left(query, 80) AS query_snippet
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;
```

**ตัวอย่างผลลัพธ์:**
```
 total_ms  |  calls  | mean_ms | pct_of_total |                              query_snippet
------------+----------+----------+----------------+-------------------------------------------------------------------------------
 8210442.10|  1092011 |     7.52 |         38.21 | SELECT * FROM orders WHERE customer_id = $1
 3921004.88|      412 |  9518.94 |         18.24 | SELECT sum(oi.amount) FROM order_items oi JOIN orders o ON o.id = oi.order_id...
 2109887.21|    89210 |    23.65 |          9.81 | UPDATE inventory SET stock = stock - $1 WHERE product_id = $2
(3 rows)
```

**การตีความ:** query แรก (`SELECT * FROM orders WHERE customer_id = $1`) แม้จะมี `mean_ms` ต่ำเพียง 7.52ms ต่อครั้ง แต่ถูกเรียกถึง 1,092,011 ครั้ง ทำให้กิน **38.21%** ของ execution time ทั้งหมดของระบบ — นี่คือตัวอย่างคลาสสิกของ "death by a thousand cuts" ที่การ optimize query เล็ก ๆ แต่ถูกเรียกบ่อยมาก อาจให้ผลตอบแทนสูงกว่าการไป optimize query ใหญ่ที่เรียกไม่บ่อย

### Query ที่มี Variance สูง (ไม่เสถียร)

Query ที่บางครั้งเร็ว บางครั้งช้ามาก (เช่นเพราะ query plan เปลี่ยนไปตาม parameter หรือ data skew) เป็นสัญญาณของปัญหาที่ซ่อนอยู่:

```sql
SELECT
    left(query, 80) AS query_snippet,
    calls,
    round(mean_exec_time::numeric, 2) AS mean_ms,
    round(stddev_exec_time::numeric, 2) AS stddev_ms,
    round(min_exec_time::numeric, 2) AS min_ms,
    round(max_exec_time::numeric, 2) AS max_ms
FROM pg_stat_statements
WHERE calls > 100
ORDER BY stddev_exec_time DESC
LIMIT 5;
```

**ตัวอย่างผลลัพธ์:**
```
                              query_snippet                              |  calls  | mean_ms | stddev_ms |  min_ms |  max_ms
---------------------------------------------------------------------------+----------+----------+-------------+----------+-----------
 SELECT * FROM orders WHERE status = $1 AND created_at > $2               |    8921 |   142.33 |     891.02 |     0.41 |  12042.88
(1 row)
```

Query นี้มี `min_ms = 0.41` แต่ `max_ms = 12042.88` — ความแตกต่างมหาศาลนี้มักเกิดจาก **parameter sniffing** (planner cache แผนที่เหมาะกับ parameter หนึ่ง แต่ใช้ผิดกับอีก parameter หนึ่งที่มี data distribution ต่างกันมาก เช่น `status = 'pending'` มีแค่ 100 แถว แต่ `status = 'completed'` มีล้านแถว)

### Query ที่ใช้ Temp Files บ่อย (work_mem ไม่พอ)

```sql
SELECT
    left(query, 80) AS query_snippet,
    calls,
    temp_blks_written,
    round(mean_exec_time::numeric, 2) AS mean_ms
FROM pg_stat_statements
WHERE temp_blks_written > 0
ORDER BY temp_blks_written DESC
LIMIT 5;
```

**ตัวอย่างผลลัพธ์:**
```
                                query_snippet                                | calls | temp_blks_written | mean_ms
--------------------------------------------------------------------------------+--------+---------------------+----------
 SELECT customer_id, sum(amount) FROM order_items GROUP BY customer_id ORDE... |   412 |             982104 | 9518.94
(1 row)
```

### สร้าง View สรุปสำหรับ Dashboard ที่รีเฟรชอัตโนมัติ

เพื่อให้ง่ายต่อการต่อ dashboard เครื่องมือภายนอก (Grafana หรือ script ภายใน) เราสามารถห่อ query ที่ใช้บ่อยไว้เป็น view:

```sql
CREATE OR REPLACE VIEW monitoring.top_queries_summary AS
SELECT
    queryid,
    left(query, 100) AS query_snippet,
    calls,
    round(total_exec_time::numeric, 2) AS total_ms,
    round(mean_exec_time::numeric, 2) AS mean_ms,
    round(stddev_exec_time::numeric, 2) AS stddev_ms,
    rows,
    round((rows::numeric / NULLIF(calls, 0)), 2) AS avg_rows_per_call,
    shared_blks_hit,
    shared_blks_read,
    round(
        100.0 * shared_blks_hit / NULLIF(shared_blks_hit + shared_blks_read, 0), 2
    ) AS cache_hit_pct
FROM pg_stat_statements
ORDER BY total_exec_time DESC;
```

```sql
-- ใช้งาน view นี้จาก monitoring script หรือ Grafana panel ได้ทันที
SELECT * FROM monitoring.top_queries_summary LIMIT 10;
```

### สำคัญ: การ Reset สถิติเป็นระยะ

```sql
-- Reset สถิติทั้งหมดใน pg_stat_statements (มักใช้ก่อนเริ่ม load test หรือหลังทำ deploy ใหญ่)
SELECT pg_stat_statements_reset();
```

> **แนวปฏิบัติที่ดี:** อย่า reset สถิติบ่อยเกินไปในระบบ production เพราะจะทำให้สูญเสีย trend ระยะยาว หากต้องการเก็บ historical data ควร snapshot ค่าจาก `pg_stat_statements` ไปเก็บใน table แยกเป็นระยะ (เช่นทุกชั่วโมง ผ่าน cron job หรือ pg_cron) หรือใช้เครื่องมืออย่าง Prometheus/pganalyze ที่เก็บ time-series ให้อัตโนมัติ ซึ่งเป็นสิ่งที่เราจะเรียนรู้ใน Step ถัดไป

---

## Step 717: postgres_exporter — Export Metrics ไปให้ Prometheus

> **หมายเหตุ:** เนื้อหาตั้งแต่ Step นี้เป็นต้นไปเป็น **แนวทางเชิงสถาปัตยกรรม (architectural guidance)** คำสั่งและไฟล์ config ที่แสดงเป็นตัวอย่างมาตรฐานของอุตสาหกรรม แต่ผู้เรียนต้องปรับ version, network, credentials ให้เข้ากับ environment จริงของตนเอง และควรทดสอบใน environment staging ก่อนนำไปใช้ production

### ปัญหาของการ query pg_stat_* ด้วยมือ

การ query view เหล่านี้ด้วยมือ (ตามที่เราทำใน Step 712-716) เหมาะสำหรับการวินิจฉัยปัญหาแบบ ad-hoc แต่ไม่เหมาะกับการ monitor ต่อเนื่องแบบ 24/7 เพราะ:

1. ไม่มีใครนั่งรัน query ตลอดเวลา
2. ไม่มี historical trend ให้ดูย้อนหลัง
3. ไม่มีระบบ alert อัตโนมัติเมื่อค่าผิดปกติ

นี่คือเหตุผลที่เราต้องมี **metrics exporter** ที่คอย query view เหล่านี้ให้อัตโนมัติ แปลงเป็น time-series data แล้วส่งให้ระบบเก็บข้อมูล เช่น **Prometheus**

### postgres_exporter คืออะไร

`postgres_exporter` (โดยทีม [prometheus-community](https://github.com/prometheus-community/postgres_exporter)) เป็นโปรแกรมตัวกลางที่:

1. เชื่อมต่อไปยัง PostgreSQL server ด้วย connection string ปกติ
2. รัน query กับ `pg_stat_*` views เป็นระยะ (ตาม interval ที่ Prometheus scrape)
3. แปลงผลลัพธ์เป็น metrics format ที่ Prometheus อ่านได้ (ผ่าน HTTP endpoint `/metrics`)

```
┌───────────────────┐   query pg_stat_*  ┌────────────────────┐   scrape (HTTP)  ┌─────────────┐
│  PostgreSQL Server │ ◄──────────────────│  postgres_exporter │ ◄─────────────────│  Prometheus │
│                    │ ──────────────────► │  (port 9187)       │ ─────────────────►│  Server     │
└───────────────────┘   result rows      └────────────────────┘   /metrics text   └─────────────┘
```

### การติดตั้ง (ตัวอย่างแนวทาง — Docker)

```yaml
# docker-compose.yml (ตัวอย่างแนวทาง — ปรับตาม environment จริง)
version: "3.8"
services:
  postgres_exporter:
    image: prometheuscommunity/postgres-exporter:v0.15.0
    container_name: postgres_exporter
    environment:
      DATA_SOURCE_NAME: "postgresql://monitoring_user:CHANGE_ME_SECRET@db-host:5432/ecommerce_prod?sslmode=require"
    ports:
      - "9187:9187"
    restart: unless-stopped
```

> **แนวปฏิบัติด้านความปลอดภัย:** ควรสร้าง role แยกสำหรับ monitoring โดยเฉพาะ (ไม่ใช้ superuser) และให้สิทธิ์แค่ที่จำเป็น เช่น:

```sql
-- สร้าง role สำหรับ monitoring โดยเฉพาะ (ทบทวนแนวคิดจาก Part 068 Roles & Privileges)
CREATE ROLE monitoring_user WITH LOGIN PASSWORD 'CHANGE_ME_SECRET';
GRANT pg_monitor TO monitoring_user;
```

`pg_monitor` เป็น predefined role ของ PostgreSQL (ตั้งแต่เวอร์ชัน 10) ที่ให้สิทธิ์อ่านค่าสถิติทั้งหมดที่จำเป็นสำหรับ monitoring โดยไม่ต้องให้สิทธิ์ superuser

### Metric ที่ postgres_exporter ให้มาโดยค่าเริ่มต้น (Default Metrics)

`postgres_exporter` มาพร้อม default query ที่แปลง view สำคัญ ๆ ให้เป็น metric โดยอัตโนมัติ เช่น:

| Metric (Prometheus name) | มาจาก view | ความหมาย |
|---|---|---|
| `pg_stat_database_xact_commit` | `pg_stat_database` | จำนวน transaction commit สะสม (ใช้คำนวณ rate เป็น TPS) |
| `pg_stat_database_blks_hit` / `pg_stat_database_blks_read` | `pg_stat_database` | ใช้คำนวณ cache hit ratio |
| `pg_stat_activity_count` | `pg_stat_activity` | จำนวน connection แบ่งตาม state |
| `pg_locks_count` | `pg_locks` | จำนวน lock แบ่งตามประเภท |
| `pg_stat_user_tables_n_dead_tup` | `pg_stat_user_tables` | จำนวน dead tuple ต่อตาราง |
| `pg_up` | (internal) | 1 = exporter เชื่อมต่อ database ได้สำเร็จ, 0 = เชื่อมต่อไม่ได้ |

### Custom Query — เพิ่ม Metric ที่เราต้องการเอง

จุดแข็งของ `postgres_exporter` คือสามารถเพิ่ม custom SQL query เองผ่านไฟล์ `queries.yaml` เพื่อ expose metric ที่เกี่ยวข้องกับ business logic โดยเฉพาะ:

```yaml
# queries.yaml (ตัวอย่างแนวทาง)
pg_long_running_queries:
  query: |
    SELECT count(*) AS count
    FROM pg_stat_activity
    WHERE state != 'idle'
      AND now() - query_start > interval '5 minutes'
  metrics:
    - count:
        usage: "GAUGE"
        description: "Number of queries running longer than 5 minutes"

pg_cache_hit_ratio:
  query: |
    SELECT
      datname,
      round(100.0 * blks_hit / NULLIF(blks_hit + blks_read, 0), 2) AS ratio
    FROM pg_stat_database
    WHERE datname NOT IN ('template0', 'template1')
  metrics:
    - datname:
        usage: "LABEL"
        description: "Database name"
    - ratio:
        usage: "GAUGE"
        description: "Cache hit ratio percentage"
```

สังเกตว่า query ทั้งสองตัวนี้คือ query แบบเดียวกับที่เราเขียนด้วยมือใน Step 712 และ Step 715 — สิ่งที่ `postgres_exporter` ทำคือ "รันคำสั่งเหล่านี้ให้เราอัตโนมัติทุก ๆ กี่วินาที" แทนที่จะต้องนั่งรันเอง

### ตรวจสอบ Metrics ที่ Export ออกมา (Endpoint ทดสอบ)

หลังจากติดตั้งเรียบร้อยแล้ว เราสามารถทดสอบว่า exporter ทำงานถูกต้องได้โดยเรียก HTTP endpoint ตรง ๆ:

```
$ curl http://localhost:9187/metrics | grep pg_stat_database_xact_commit
```

**ตัวอย่างผลลัพธ์ (ข้อความ metric format ของ Prometheus):**
```
# HELP pg_stat_database_xact_commit Number of transactions in this database that have been committed
# TYPE pg_stat_database_xact_commit counter
pg_stat_database_xact_commit{datname="ecommerce_prod"} 1.8442091e+07
pg_stat_database_xact_commit{datname="analytics_dw"} 982104
```

### กำหนดค่า Prometheus ให้ Scrape จาก postgres_exporter

```yaml
# prometheus.yml (ตัวอย่างแนวทาง)
scrape_configs:
  - job_name: "postgresql"
    scrape_interval: 15s
    static_configs:
      - targets: ["postgres_exporter:9187"]
        labels:
          environment: "production"
          service: "ecommerce-db"
```

---

## Step 718: Prometheus + Grafana — สถาปัตยกรรมโดยรวม

> เนื้อหาใน Step นี้เป็น **แนวทางเชิงสถาปัตยกรรม** ทั้งหมด เพื่อให้ผู้เรียนเข้าใจภาพรวมว่าส่วนประกอบต่าง ๆ ทำงานร่วมกันอย่างไร โดยไม่ลงรายละเอียดการติดตั้งแบบ step-by-step ที่ผูกกับ environment เฉพาะ

### ภาพรวมสถาปัตยกรรม

```
┌──────────────────┐     ┌────────────────────┐     ┌─────────────┐     ┌───────────┐
│  PostgreSQL       │────►│  postgres_exporter  │────►│  Prometheus  │────►│  Grafana   │
│  (pg_stat_* views)│     │  (metrics adapter)  │     │  (TSDB +     │     │  (visualize│
│                    │     │                     │     │   scraper)   │     │  + alert)  │
└──────────────────┘     └────────────────────┘     └─────────────┘     └───────────┘
                                                              │
                                                              ▼
                                                     ┌──────────────────┐
                                                     │  Alertmanager     │
                                                     │  (ส่งแจ้งเตือนไป   │
                                                     │  Slack/Email/     │
                                                     │  PagerDuty)        │
                                                     └──────────────────┘
```

### บทบาทของแต่ละส่วนประกอบ

| Component | บทบาท |
|---|---|
| **PostgreSQL** | แหล่งข้อมูลต้นทาง เก็บสถิติผ่าน `pg_stat_*` views |
| **postgres_exporter** | แปลงข้อมูลจาก SQL query ให้เป็น metrics format ที่ Prometheus เข้าใจ (ตามที่เรียนใน Step 717) |
| **Prometheus** | ระบบ pull-based time-series database ที่ "ดึง" (scrape) ข้อมูลจาก exporter เป็นระยะตาม interval ที่กำหนด แล้วเก็บเป็น time-series พร้อม label |
| **Grafana** | เครื่องมือ visualize ที่ query ข้อมูลจาก Prometheus มาสร้างเป็น dashboard กราฟต่าง ๆ |
| **Alertmanager** | รับ alert rule ที่ trigger จาก Prometheus แล้วจัดการเรื่อง grouping, deduplication, และส่งแจ้งเตือนไปยังช่องทางต่าง ๆ (Slack, Email, PagerDuty) |

### ทำไมต้องใช้สถาปัตยกรรมแบบ Pull (Prometheus) แทนที่จะ Push

Prometheus ใช้โมเดล **pull-based** คือตัว Prometheus server เป็นฝ่าย "เดินไปขอข้อมูล" จาก exporter เป็นระยะ ๆ (ต่างจากระบบ push-based ที่ metric ต้องถูกส่งเข้ามาเอง) ข้อดีของโมเดลนี้คือ:

1. Prometheus รู้ทันทีถ้า scrape ไม่สำเร็จ (target down) — กลายเป็น signal ของปัญหาได้เลย (`up == 0`)
2. ควบคุม load บน target ได้ง่าย เพราะ Prometheus เป็นฝ่ายกำหนด interval เอง
3. Debug ง่าย เพราะสามารถเปิด `/metrics` endpoint ดูตรง ๆ ได้ (ตามที่ทำใน Step 717)

### ตัวอย่างการสร้าง PromQL Query สำหรับ Grafana Panel

**PromQL** คือภาษา query ของ Prometheus ที่ใช้ดึงข้อมูล time-series มาแสดงผล:

```promql
# คำนวณ TPS (Transactions Per Second) จาก counter xact_commit
# rate() คำนวณอัตราการเปลี่ยนแปลงต่อวินาทีโดยอัตโนมัติจาก counter สะสม
rate(pg_stat_database_xact_commit{datname="ecommerce_prod"}[1m])
```

```promql
# คำนวณ Cache Hit Ratio แบบ real-time โดยใช้ rate() ของทั้ง blks_hit และ blks_read
# (แม่นยำกว่าการหารค่าสะสมตรง ๆ เพราะสะท้อนพฤติกรรมปัจจุบัน ไม่ถูก skew ด้วยข้อมูลสะสมในอดีต)
sum(rate(pg_stat_database_blks_hit{datname="ecommerce_prod"}[5m]))
/
(
  sum(rate(pg_stat_database_blks_hit{datname="ecommerce_prod"}[5m]))
  +
  sum(rate(pg_stat_database_blks_read{datname="ecommerce_prod"}[5m]))
) * 100
```

```promql
# จำนวน connection ปัจจุบัน เทียบกับ max_connections (เพื่อดูว่าใกล้เต็มหรือยัง)
pg_stat_activity_count{datname="ecommerce_prod"} / pg_settings_max_connections * 100
```

### โครงสร้าง Dashboard ที่แนะนำใน Grafana

Dashboard สำหรับ monitor PostgreSQL production ควรแบ่งเป็นโซนต่อไปนี้:

```
┌───────────────────────────────────────────────────────────────┐
│                   PostgreSQL Production Dashboard                │
├───────────────────────┬───────────────────────┬──────────────────┤
│  Row 1: Overview        │                          │                    │
│  [TPS]  [Cache Hit %]  │  [Active Connections]   │  [Replication Lag]│
├───────────────────────┴───────────────────────┴──────────────────┤
│  Row 2: Query Performance                                          │
│  [Top 10 Slowest Queries (table)] [Query Latency p50/p95/p99]     │
├─────────────────────────────────────────────────────────────────┤
│  Row 3: Resource Saturation                                        │
│  [Connections / max_connections]  [Disk Usage]  [WAL Generation]  │
├─────────────────────────────────────────────────────────────────┤
│  Row 4: Table & Index Health                                       │
│  [Dead Tuple % by table]  [Unused Indexes]  [Autovacuum activity] │
└─────────────────────────────────────────────────────────────────┘
```

### ตัวอย่าง Dashboard JSON Panel (แนวคิด — ไม่ใช่ config สมบูรณ์)

```yaml
# ตัวอย่างแนวคิดของ panel definition ใน Grafana dashboard (ย่อ)
panel:
  title: "Cache Hit Ratio (%)"
  type: "gauge"
  targets:
    - expr: |
        sum(rate(pg_stat_database_blks_hit{datname="ecommerce_prod"}[5m]))
        /
        (sum(rate(pg_stat_database_blks_hit{datname="ecommerce_prod"}[5m]))
         + sum(rate(pg_stat_database_blks_read{datname="ecommerce_prod"}[5m]))) * 100
  thresholds:
    - color: "red"
      value: 0
    - color: "yellow"
      value: 90
    - color: "green"
      value: 99
```

### ตัวอย่าง Alert Rule ใน Prometheus

```yaml
# alert_rules.yml (ตัวอย่างแนวทาง — ใช้กับ Prometheus Alertmanager)
groups:
  - name: postgresql_alerts
    rules:
      - alert: PostgreSQLCacheHitRatioLow
        expr: |
          (
            sum(rate(pg_stat_database_blks_hit{datname="ecommerce_prod"}[5m]))
            /
            (sum(rate(pg_stat_database_blks_hit{datname="ecommerce_prod"}[5m]))
             + sum(rate(pg_stat_database_blks_read{datname="ecommerce_prod"}[5m])))
          ) * 100 < 95
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "PostgreSQL cache hit ratio ต่ำกว่า 95% นานเกิน 10 นาที"
          description: "Database ecommerce_prod มี cache hit ratio {{ $value | printf \"%.2f\" }}% ซึ่งอาจบ่งชี้ว่า shared_buffers ไม่เพียงพอ"

      - alert: PostgreSQLTooManyConnections
        expr: |
          pg_stat_activity_count{datname="ecommerce_prod"}
          / pg_settings_max_connections * 100 > 80
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "จำนวน connection ใกล้เต็ม max_connections"
          description: "ใช้ connection ไปแล้ว {{ $value | printf \"%.1f\" }}% ของ max_connections"

      - alert: PostgreSQLLongRunningQuery
        expr: pg_long_running_queries_count > 0
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "พบ query ที่รันนานเกิน 5 นาที"
```

---

## Step 719: pganalyze และ Managed Monitoring SaaS อื่น ๆ

> เนื้อหาใน Step นี้เป็น **แนวทางเชิงแนวคิด (conceptual)** เพื่อช่วยผู้เรียนตัดสินใจเลือกแนวทาง ไม่ใช่คู่มือการติดตั้งแบบละเอียด เพราะแต่ละ vendor มีขั้นตอนและราคาที่เปลี่ยนแปลงตลอดเวลา ผู้เรียนควรตรวจสอบเอกสารล่าสุดของแต่ละผลิตภัณฑ์โดยตรง

### ปัญหาของการสร้าง Monitoring Stack เอง (Self-Hosted)

สถาปัตยกรรม Prometheus + Grafana ที่เรียนใน Step 717-718 นั้นทรงพลังและฟรี (open source) แต่ก็มีต้นทุนแฝงที่ต้องพิจารณา:

1. **ต้องดูแลเอง (operational overhead)** — Prometheus, Grafana, Alertmanager, exporter ล้วนเป็นระบบที่ต้อง deploy, monitor, upgrade, backup เองทั้งหมด (ใครจะ monitor ตัว monitoring stack?)
2. **ต้องเขียน query/dashboard/alert เอง** — ทีมต้องมีความรู้ PostgreSQL internals ลึกพอที่จะเขียน PromQL และตีความ metric ได้ถูกต้อง
3. **ขาด domain-specific insight** — Prometheus/Grafana เป็นเครื่องมือ generic ที่ไม่ได้เข้าใจ "ความหมาย" ของ query plan หรือ index recommendation โดยอัตโนมัติ

### pganalyze คืออะไร

**pganalyze** เป็นตัวอย่างของ **managed SaaS monitoring solution** ที่สร้างมาเฉพาะสำหรับ PostgreSQL โดยเฉพาะ (ต่างจาก Prometheus/Grafana ที่เป็นเครื่องมือ generic) จุดเด่นหลัก ๆ ได้แก่:

1. **Automated EXPLAIN plan analysis** — เก็บ query plan ของ query ที่ช้าโดยอัตโนมัติ พร้อมคำอธิบายเป็นภาษาที่เข้าใจง่าย (ไม่ต้องอ่าน `EXPLAIN ANALYZE` output ดิบเอง)
2. **Index advisor** — วิเคราะห์ query pattern แล้วแนะนำ index ที่ควรสร้างหรือควรลบโดยอัตโนมัติ
3. **VACUUM monitoring** — ติดตามการทำงานของ autovacuum แบบละเอียด พร้อมแจ้งเตือนเมื่อ vacuum ตามไม่ทัน
4. **Connection tracing** — เชื่อมโยง query ที่ช้ากับ application code ต้นทาง (เช่น ORM call stack)
5. **ไม่ต้องดูแล infrastructure เอง** — vendor จัดการเรื่อง storage, scaling, upgrade ของระบบ monitoring ให้ทั้งหมด

### เปรียบเทียบ Self-Hosted (Prometheus+Grafana) vs Managed SaaS (pganalyze และอื่น ๆ)

| ประเด็น | Self-Hosted (Prometheus + Grafana) | Managed SaaS (pganalyze และเทียบเคียง) |
|---|---|---|
| **ต้นทุนเริ่มต้น** | ต่ำ (open source, ไม่มีค่า license) | มีค่าใช้จ่ายรายเดือน/รายปี ตาม tier |
| **ต้นทุนแฝง (operational)** | สูง — ต้องมีคนดูแล infra เอง | ต่ำ — vendor ดูแลให้ |
| **ความยืดหยุ่นในการปรับแต่ง** | สูงมาก — ปรับได้ทุกอย่าง (custom query, custom dashboard) | จำกัดตามฟีเจอร์ที่ vendor มีให้ |
| **ความรู้ลึกด้าน PostgreSQL ที่ต้องมี** | สูง — ต้องเข้าใจ metric เองเพื่อสร้าง dashboard/alert | ต่ำกว่า — เครื่องมือช่วยตีความให้ |
| **Data residency / privacy** | ควบคุมได้เต็มที่ (ข้อมูลอยู่ใน infra ตัวเอง) | ต้องส่งข้อมูล query/metric ออกไปยัง third-party (ต้องพิจารณาเรื่อง compliance) |
| **Time-to-value** | ช้ากว่า — ต้อง setup เยอะก่อนได้ dashboard ที่ใช้งานได้จริง | เร็วกว่า — เชื่อมต่อแล้วได้ insight ทันที |
| **เหมาะกับ** | องค์กรที่มีทีม DBA/SRE แข็งแรง ต้องการควบคุมเต็มรูปแบบ, หรือมี compliance requirement ที่ห้ามส่งข้อมูลออกนอกองค์กร | ทีมขนาดเล็ก-กลางที่ต้องการ insight เร็ว ไม่มีทรัพยากรดูแล infra monitoring เอง |

### SaaS Monitoring อื่น ๆ ที่ควรรู้จัก (ภาพรวม)

นอกจาก pganalyze ยังมีเครื่องมือ SaaS อื่นที่มีแนวคิดคล้ายกันหรือครอบคลุม PostgreSQL เป็นส่วนหนึ่งของ platform ที่ใหญ่กว่า:

- **Datadog Database Monitoring** — เป็นส่วนหนึ่งของ Datadog APM platform ครอบคลุม ecosystem กว้างกว่า (ไม่ใช่แค่ PostgreSQL) เหมาะกับองค์กรที่ใช้ Datadog อยู่แล้วสำหรับ monitor ทั้ง application และ infrastructure
- **New Relic** — คล้าย Datadog เน้น full-stack observability
- **Amazon RDS Performance Insights** — สำหรับผู้ที่ใช้ PostgreSQL บน Amazon RDS/Aurora โดยเฉพาะ เป็น built-in monitoring ที่ผูกกับ managed service ของ AWS
- **Crunchy Data / EDB monitoring tools** — จาก vendor ที่เชี่ยวชาญ PostgreSQL โดยเฉพาะ มักมาพร้อมกับ managed PostgreSQL service ของตนเอง

### แนวทางการตัดสินใจ (Decision Framework)

```
เริ่มต้น
   │
   ▼
มีทีม DBA/SRE ที่เชี่ยวชาญ PostgreSQL internals หรือไม่?
   │
   ├── ไม่มี ──► พิจารณา Managed SaaS (pganalyze หรือเทียบเคียง)
   │              เพื่อลด operational burden และได้ insight เร็ว
   │
   └── มี ──► มี compliance requirement ที่ห้ามส่งข้อมูล query ออกนอกองค์กรหรือไม่?
                 │
                 ├── มี ──► Self-hosted (Prometheus + Grafana) เท่านั้น
                 │
                 └── ไม่มี ──► พิจารณาทั้งสองทาง ตามงบประมาณและ
                                ความต้องการ customization
```

> **ในทางปฏิบัติ** หลายองค์กรใช้ทั้งสองแนวทางร่วมกัน: Prometheus + Grafana สำหรับ infrastructure-wide monitoring (CPU, memory, network ของทุกระบบในองค์กร รวม PostgreSQL) และใช้ pganalyze หรือเครื่องมือ SaaS เฉพาะทางสำหรับ deep-dive วิเคราะห์ query performance ที่ต้องการความเชี่ยวชาญเฉพาะด้าน PostgreSQL

---

## สรุปท้ายบท

### ตารางสรุป Metric สำคัญที่ต้อง Monitor พร้อม Threshold แนะนำ

| Metric | View/แหล่งข้อมูล | สูตร/วิธีดู | Threshold แนะนำ | Action เมื่อผิดปกติ |
|---|---|---|---|---|
| **Cache Hit Ratio** | `pg_stat_database` | `blks_hit / (blks_hit + blks_read)` | ≥ 99% (OLTP) | เพิ่ม `shared_buffers`, ตรวจสอบ query ที่ scan ข้อมูลเกินจำเป็น |
| **Connection Usage** | `pg_stat_activity` vs `max_connections` | `count(*) / max_connections * 100` | < 80% | เพิ่ม connection pooling (PgBouncer — ดู Part 066), เพิ่ม `max_connections` อย่างระมัดระวัง |
| **Long-Running Query** | `pg_stat_activity` | `now() - query_start > 5 min` | จำนวน = 0 เป็นอุดมคติ | ตรวจสอบ query plan, พิจารณา `pg_cancel_backend()` |
| **Idle in Transaction** | `pg_stat_activity` | `state = 'idle in transaction'` นานเกิน 10 นาที | จำนวน = 0 | ตรวจสอบ application code ที่ลืม commit/rollback |
| **Deadlocks** | `pg_stat_database` | `deadlocks` (สะสม, ดู delta) | ใกล้ 0 | ตรวจสอบ transaction logic, ลำดับการ lock ตาราง |
| **Dead Tuple %** | `pg_stat_user_tables` | `n_dead_tup / (n_live_tup + n_dead_tup)` | < 10-20% | ปรับ `autovacuum_vacuum_scale_factor` ให้ aggressive ขึ้น |
| **Rollback Ratio** | `pg_stat_database` | `xact_rollback / (xact_commit + xact_rollback)` | < 1-5% | ตรวจสอบ error rate ใน application |
| **Temp Files** | `pg_stat_database` / `pg_stat_statements` | `temp_files`, `temp_bytes` | ต่ำ / คงที่ | เพิ่ม `work_mem` หรือ optimize query ที่ sort/hash ข้อมูลมาก |
| **Sequential Scan Ratio** | `pg_stat_user_tables` | `seq_scan / (seq_scan + idx_scan)` | ต่ำสำหรับตารางใหญ่ | เพิ่ม index ที่เหมาะสม |
| **Unused Indexes** | `pg_stat_user_indexes` | `idx_scan = 0` (สังเกตต่อเนื่อง ≥ 30 วัน) | 0 index ที่ไม่ใช้ | พิจารณาลบ index เพื่อลด write overhead |
| **Query Latency (p95/p99)** | `pg_stat_statements` | `mean_exec_time`, `stddev_exec_time` | ตาม SLA ของระบบ | หา query ที่ variance สูง ตรวจสอบ parameter sniffing |
| **Replication Lag** | `pg_stat_replication` (ทบทวนจาก Part 063) | `pg_wal_lsn_diff()` | < 2-5 วินาที (ตาม RPO) | ตรวจสอบ network, disk I/O บน replica |
| **Autovacuum Freshness** | `pg_stat_user_tables` | `last_autovacuum`, `last_autoanalyze` | ไม่เกิน 24-48 ชม. สำหรับตารางที่เปลี่ยนแปลงบ่อย | ปรับ autovacuum settings ให้เหมาะกับ workload |

### สรุปเครื่องมือแต่ละชั้น (Monitoring Stack Layers)

```
┌────────────────────────────────────────────────────────────────┐
│  Layer 1: Ad-hoc SQL queries (pg_stat_*)                          │
│  → ใช้วินิจฉัยปัญหาแบบทันที (Step 712-716)                        │
├────────────────────────────────────────────────────────────────┤
│  Layer 2: Metrics Export (postgres_exporter)                       │
│  → แปลง pg_stat_* ให้เป็น time-series metric (Step 717)          │
├────────────────────────────────────────────────────────────────┤
│  Layer 3: Time-series Storage + Visualization                      │
│  (Prometheus + Grafana) → เก็บ trend, สร้าง dashboard (Step 718) │
├────────────────────────────────────────────────────────────────┤
│  Layer 4: Alerting                                                  │
│  (Alertmanager / SaaS built-in alert) → แจ้งเตือนก่อนปัญหาลุกลาม   │
├────────────────────────────────────────────────────────────────┤
│  Layer 5 (ทางเลือก): Managed SaaS (pganalyze ฯลฯ)                 │
│  → deep insight เฉพาะทาง PostgreSQL โดยไม่ต้องดูแล infra เอง (Step 719) │
└────────────────────────────────────────────────────────────────┘
```

### หลักการสำคัญที่ควรจำจากบทนี้

1. **Proactive ดีกว่า Reactive เสมอ** — การลงทุนสร้าง monitoring ตั้งแต่ต้น คุ้มค่ากว่าการเสียเวลาดับไฟตอนระบบล่ม
2. **PostgreSQL เก็บข้อมูลให้เราอยู่แล้ว** — `pg_stat_*` views ไม่ต้องติดตั้งอะไรเพิ่ม (ยกเว้น `pg_stat_statements`) แค่ต้องรู้จักอ่านให้เป็น
3. **Cache Hit Ratio คือ metric ที่คุ้มค่าที่สุดที่จะดูก่อน** — ค่าเดียวที่บอกสุขภาพ memory/disk balance ได้ดี
4. **การ query ด้วยมือใช้ได้กับการวินิจฉัย แต่ไม่ใช่การ monitor ต่อเนื่อง** — ต้องมีระบบอัตโนมัติอย่าง Prometheus หรือ SaaS
5. **เลือกเครื่องมือตามบริบทองค์กร** — ไม่มีคำตอบเดียวที่ถูกสำหรับทุกที่ ระหว่าง self-hosted กับ managed SaaS

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1

เขียน SQL query เพื่อหา session ทั้งหมดที่ `state = 'active'` และรันมานานเกิน 2 นาที พร้อมแสดง `pid`, `usename`, ระยะเวลาที่รัน (duration), และ query ที่กำลังรัน

<details>
<summary>เฉลย</summary>

```sql
SELECT
    pid,
    usename,
    now() - query_start AS duration,
    query
FROM pg_stat_activity
WHERE state = 'active'
  AND now() - query_start > interval '2 minutes'
ORDER BY duration DESC;
```

คำอธิบาย: กรองเฉพาะ `state = 'active'` (ไม่รวม idle หรือ idle in transaction) แล้วเปรียบเทียบ `query_start` กับเวลาปัจจุบัน ใช้ `interval '2 minutes'` เป็นเกณฑ์

</details>

---

### แบบฝึกหัดที่ 2

เขียน SQL query เพื่อคำนวณ Cache Hit Ratio ของ database ชื่อ `shop_db` และให้บอกว่าผลลัพธ์อยู่ในเกณฑ์ "ดีเยี่ยม", "ยอมรับได้", "ควรปรับปรุง" หรือ "มีปัญหา" ตามเกณฑ์ที่เรียนในบทนี้

<details>
<summary>เฉลย</summary>

```sql
SELECT
    datname,
    blks_hit,
    blks_read,
    round(100.0 * blks_hit / NULLIF(blks_hit + blks_read, 0), 2) AS cache_hit_ratio_pct,
    CASE
        WHEN 100.0 * blks_hit / NULLIF(blks_hit + blks_read, 0) >= 99 THEN 'ดีเยี่ยม'
        WHEN 100.0 * blks_hit / NULLIF(blks_hit + blks_read, 0) >= 95 THEN 'ยอมรับได้'
        WHEN 100.0 * blks_hit / NULLIF(blks_hit + blks_read, 0) >= 90 THEN 'ควรปรับปรุง'
        ELSE 'มีปัญหา'
    END AS assessment
FROM pg_stat_database
WHERE datname = 'shop_db';
```

คำอธิบาย: ใช้สูตรมาตรฐาน `blks_hit / (blks_hit + blks_read)` แล้วใช้ `CASE WHEN` แบ่งช่วงตามตารางเกณฑ์ใน Step 715 (`NULLIF` ป้องกันการหารด้วยศูนย์)

</details>

---

### แบบฝึกหัดที่ 3

ทีม dev รายงานว่า transaction บางตัวค้างนานผิดปกติ จงเขียน query เพื่อหา session ที่มี `state = 'idle in transaction'` ที่ค้างนานเกิน 15 นาที และอธิบายว่าทำไม state นี้ถึงอันตรายต่อระบบมากกว่า long-running query ธรรมดา

<details>
<summary>เฉลย</summary>

```sql
SELECT
    pid,
    usename,
    client_addr,
    now() - xact_start AS idle_duration,
    query AS last_query
FROM pg_stat_activity
WHERE state = 'idle in transaction'
  AND now() - xact_start > interval '15 minutes'
ORDER BY idle_duration DESC;
```

คำอธิบาย: `idle in transaction` อันตรายกว่า long-running query ธรรมดา เพราะ:
1. มันกัน VACUUM ไม่ให้ลบ dead tuple ที่ transaction เก่านั้นยังอาจต้องเห็น (transaction ยังไม่ commit/rollback) ทำให้เกิด table bloat สะสม
2. มันอาจถือ lock ค้างไว้โดยไม่จำเป็น ทำให้ transaction อื่นต้องรอ (blocking)
3. มักเป็นสัญญาณของ bug ในโค้ด application (เช่น connection leak หรือลืม commit)

</details>

---

### แบบฝึกหัดที่ 4

จงเขียน query เพื่อหา 5 ตารางที่มีสัดส่วน sequential scan สูงที่สุด (เทียบกับ index scan) โดยกรองเฉพาะตารางที่มีข้อมูลมากกว่า 10,000 แถว เพื่อไม่ให้ตารางเล็กที่ seq scan เป็นเรื่องปกติมารบกวนผลลัพธ์

<details>
<summary>เฉลย</summary>

```sql
SELECT
    relname AS table_name,
    seq_scan,
    idx_scan,
    n_live_tup,
    round(100.0 * seq_scan / NULLIF(seq_scan + idx_scan, 0), 2) AS seq_scan_pct
FROM pg_stat_user_tables
WHERE n_live_tup > 10000
ORDER BY seq_scan_pct DESC
LIMIT 5;
```

คำอธิบาย: การกรอง `n_live_tup > 10000` สำคัญมาก เพราะ planner มักเลือก seq scan บนตารางเล็กโดยเจตนา (เร็วกว่า index scan เมื่อข้อมูลน้อย) ซึ่งไม่ใช่ปัญหา การกรองนี้ทำให้เราโฟกัสเฉพาะตารางใหญ่ที่ seq scan สูงจริง ๆ เป็นปัญหา

</details>

---

### แบบฝึกหัดที่ 5

อธิบายความแตกต่างระหว่าง `pg_cancel_backend()` และ `pg_terminate_backend()` พร้อมยกตัวอย่างสถานการณ์ที่ควรใช้แต่ละคำสั่ง

<details>
<summary>เฉลย</summary>

- **`pg_cancel_backend(pid)`**: ยกเลิกเฉพาะ query ที่กำลังรันอยู่ใน backend นั้น แต่ **connection และ transaction ยังคงอยู่** เหมาะกับกรณีที่ query เดียวรันนานเกินไปแต่ session ยังต้องใช้งานต่อ เช่น analyst รัน query ผิดที่ scan ข้อมูลมหาศาลโดยไม่ตั้งใจ เราต้องการแค่หยุด query นั้น แต่ไม่อยากตัด connection ทิ้ง

- **`pg_terminate_backend(pid)`**: ตัด connection ทิ้งทั้งหมดทันที (เทียบเท่าการ kill process) ใช้เมื่อ `pg_cancel_backend()` ไม่ได้ผล (เช่น backend ค้างในสถานะที่ cancel ไม่ทำงาน) หรือเมื่อ session อยู่ในสถานะ `idle in transaction` ที่ค้างนานผิดปกติและไม่มี query กำลังรันให้ cancel (เพราะ cancel ใช้ได้กับ query ที่กำลัง active เท่านั้น)

โดยทั่วไปควรลอง `pg_cancel_backend()` ก่อนเสมอ เพราะทำลายน้อยกว่า แล้วค่อยใช้ `pg_terminate_backend()` เป็นทางเลือกสุดท้าย

</details>

---

### แบบฝึกหัดที่ 6

จงเขียน query จาก `pg_stat_statements` เพื่อหา 5 query ที่กิน total execution time สูงสุดของระบบ พร้อมแสดงเปอร์เซ็นต์ของ total time ทั้งหมดที่แต่ละ query ใช้ไป

<details>
<summary>เฉลย</summary>

```sql
SELECT
    left(query, 80) AS query_snippet,
    calls,
    round(total_exec_time::numeric, 2) AS total_ms,
    round(mean_exec_time::numeric, 2) AS mean_ms,
    round((100 * total_exec_time / sum(total_exec_time) OVER ())::numeric, 2) AS pct_of_total
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 5;
```

คำอธิบาย: ใช้ window function `sum(total_exec_time) OVER ()` เพื่อคำนวณผลรวมของทุกแถวโดยไม่ต้อง `GROUP BY` ทำให้สามารถคำนวณสัดส่วนเปอร์เซ็นต์ของแต่ละ query เทียบกับทั้งระบบได้ในคิวรีเดียว query ที่ total_exec_time สูงสุดคือตัวที่ "คุ้มค่า" ที่สุดในการ optimize ก่อน เพราะให้ผลกระทบต่อระบบโดยรวมมากที่สุด

</details>

---

### แบบฝึกหัดที่ 7

จงอธิบายว่าเหตุใด cache hit ratio ที่ต่ำใน database ที่ทำหน้าที่เป็น analytics/reporting (เช่น data warehouse) จึง **ไม่จำเป็น** ต้องแปลว่ามีปัญหาเสมอไป ต่างจาก OLTP database ที่ใช้รับ order แบบ real-time

<details>
<summary>เฉลย</summary>

Analytical workload (เช่น data warehouse, reporting) มักต้อง scan ข้อมูลจำนวนมากในตาราง fact ขนาดใหญ่เพื่อทำ aggregation หรือ join ข้อมูลย้อนหลังหลายปี ซึ่งข้อมูลจำนวนมากขนาดนี้มักใหญ่กว่า `shared_buffers` ที่ตั้งไว้อย่างมาก ทำให้ต้องอ่านจาก disk เป็นปกติ (ไม่ใช่ความผิดปกติ) นี่คือพฤติกรรมที่คาดหวังได้ของ analytical workload

ในขณะที่ OLTP workload (เช่นระบบรับ order) มัก access ข้อมูล "ร้อน" (hot data) ชุดเดิมซ้ำ ๆ เช่น order ล่าสุด, สินค้าขายดี ซึ่งควร fit อยู่ใน `shared_buffers` ได้เกือบทั้งหมด ถ้า cache hit ratio ของ OLTP ต่ำ จึงบ่งชี้ปัญหาจริง เช่น working set ใหญ่เกินกว่า memory ที่จัดสรรไว้ หรือ query pattern เปลี่ยนไปจนต้องอ่านข้อมูลกระจายมากขึ้น

ดังนั้นการตีความ metric ต้องพิจารณาบริบทของ workload ประกอบเสมอ ไม่ควรใช้ threshold เดียวกันตายตัวกับทุกประเภทฐานข้อมูล

</details>

---

### แบบฝึกหัดที่ 8

จงอธิบายบทบาทของแต่ละ component ในสถาปัตยกรรม `PostgreSQL → postgres_exporter → Prometheus → Grafana → Alertmanager` และบอกว่าถ้า `postgres_exporter` หยุดทำงาน จะเกิดอะไรขึ้นกับ dashboard ใน Grafana

<details>
<summary>เฉลย</summary>

- **PostgreSQL**: แหล่งข้อมูลต้นทาง เก็บสถิติผ่าน `pg_stat_*` views
- **postgres_exporter**: query ข้อมูลจาก PostgreSQL แล้วแปลงเป็น metrics format ที่ Prometheus scrape ได้ ผ่าน HTTP endpoint `/metrics`
- **Prometheus**: pull (scrape) ข้อมูลจาก exporter เป็นระยะ เก็บเป็น time-series database พร้อม label
- **Grafana**: query ข้อมูลจาก Prometheus (ผ่าน PromQL) มา visualize เป็น dashboard กราฟต่าง ๆ
- **Alertmanager**: รับ alert ที่ trigger จาก Prometheus rule แล้วจัดการ grouping/deduplication และส่งแจ้งเตือนไปยัง Slack/Email/PagerDuty

ถ้า `postgres_exporter` หยุดทำงาน: Prometheus จะ scrape ไม่สำเร็จ (target down) ทำให้ metric `up{job="postgresql"}` เปลี่ยนเป็น `0` — Grafana จะแสดงข้อมูลเป็น "gap" (ไม่มีข้อมูลใหม่) ในกราฟ ไม่ใช่ error ทันที แต่ข้อมูลจะหยุดอัพเดท และถ้ามี alert rule ที่ตรวจจับ `up == 0` ก็จะ trigger แจ้งเตือนว่า exporter ล่ม ซึ่งเป็น pattern ที่ดีที่ควรมีเสมอ (monitor ตัว monitoring เอง)

</details>

---

### แบบฝึกหัดที่ 9

บริษัท A มีทีม DBA เพียง 1 คน ไม่มีทีม SRE เฉพาะทาง และต้องการ insight เกี่ยวกับ query performance และ index recommendation แบบรวดเร็วโดยไม่ต้องเขียน PromQL หรือ dashboard เอง ในขณะที่บริษัท B เป็นธนาคารที่มีข้อกำหนด compliance ห้ามส่งข้อมูล query ใด ๆ ออกนอกองค์กรโดยเด็ดขาด และมีทีม SRE ขนาดใหญ่ จงแนะนำแนวทาง monitoring ที่เหมาะสมสำหรับแต่ละบริษัท พร้อมเหตุผล

<details>
<summary>เฉลย</summary>

**บริษัท A**: ควรเลือก **Managed SaaS เช่น pganalyze** เพราะ:
- มีทีม DBA เพียงคนเดียว ไม่มีทรัพยากรพอที่จะดูแล Prometheus/Grafana stack เอง (operational overhead สูง)
- ต้องการ insight เร็ว เช่น index recommendation และ query plan analysis ที่ SaaS ทำให้อัตโนมัติ โดยไม่ต้องมีความเชี่ยวชาญ PromQL
- ไม่มีข้อจำกัดเรื่อง compliance ที่ห้ามส่งข้อมูลออกนอกองค์กร

**บริษัท B**: ควรเลือก **Self-hosted (Prometheus + Grafana)** เพราะ:
- มีข้อกำหนด compliance ที่ห้ามส่งข้อมูล query ออกนอกองค์กร ซึ่งตัด Managed SaaS ออกจากตัวเลือกโดยอัตโนมัติ (เพราะข้อมูลต้องถูกส่งไปยัง third-party server)
- มีทีม SRE ขนาดใหญ่ที่มีความสามารถดูแล infrastructure และเขียน custom dashboard/alert เองได้
- การควบคุมข้อมูลทั้งหมดภายในองค์กร (data residency) เป็นสิ่งจำเป็นสำหรับธุรกิจธนาคารที่มีการกำกับดูแลเข้มงวด

</details>

---

### แบบฝึกหัดที่ 10 (แบบฝึกหัดรวม: ออกแบบ Monitoring Stack แบบเต็มรูปแบบ)

บริษัท e-commerce แห่งหนึ่งกำลังจะ launch ระบบ production ใหม่ที่ใช้ PostgreSQL เป็นฐานข้อมูลหลัก (มี table หลักคือ `orders`, `order_items`, `products`, `customers`, `payments`) ทีมขอให้คุณออกแบบ monitoring stack แบบเต็มรูปแบบ ประกอบด้วย:

1. รายชื่อ metric อย่างน้อย 6 ตัวที่ควร monitor พร้อมเหตุผลว่าทำไมสำคัญกับธุรกิจ e-commerce
2. ตัวอย่าง alert rule อย่างน้อย 3 ข้อ (รวมถึงกรณี cache hit ratio ต่ำ และ connection ใกล้เต็มตามที่โจทย์กำหนด) พร้อมระบุ threshold และ severity
3. อธิบายว่าจะใช้เครื่องมือใดบ้าง (self-hosted/SaaS) และทำไม

<details>
<summary>เฉลย (แนวทางตัวอย่าง)</summary>

**1. Metric ที่ควร Monitor**

| Metric | เหตุผลเชิงธุรกิจ |
|---|---|
| **Cache Hit Ratio** (`pg_stat_database`) | ระบบ e-commerce ต้องตอบสนองเร็วโดยเฉพาะหน้า checkout — cache hit ต่ำ = latency สูง = ลูกค้าทิ้งตะกร้าสินค้า (cart abandonment) |
| **Connection Usage** (`pg_stat_activity`/`max_connections`) | ช่วง flash sale หรือโปรโมชั่น traffic พุ่งสูง connection เต็มจะทำให้ order ใหม่รับไม่ได้ทันที เสียรายได้ตรง |
| **Long-Running Query / Idle in Transaction** (`pg_stat_activity`) | Query ค้างบนตาราง `orders`/`payments` อาจ lock ตารางทำให้ลูกค้าคนอื่นสั่งซื้อไม่ได้ |
| **Deadlocks** (`pg_stat_database`) | Deadlock บนตาราง `inventory`/`payments` ระหว่างการตัดสต็อกพร้อมกันหลาย order ทำให้ transaction ล้มเหลว ลูกค้าจ่ายเงินไม่สำเร็จ |
| **Dead Tuple % บนตาราง orders/payments** (`pg_stat_user_tables`) | ตารางเหล่านี้ update บ่อยมาก (เปลี่ยนสถานะ order) ถ้า autovacuum ตามไม่ทันจะเกิด bloat ทำให้ query ช้าลงเรื่อย ๆ |
| **Top Slow Queries** (`pg_stat_statements`) | ระบุ query ที่กระทบ checkout flow โดยตรง เพื่อ optimize ก่อนที่ผู้ใช้จะรู้สึกถึงความช้า |
| **Replication Lag** (`pg_stat_replication`) | ถ้าใช้ read replica สำหรับหน้า product listing ต้องมั่นใจว่าข้อมูล stock ไม่ล้าหลังจนขายสินค้าเกินสต็อก (oversell) |

**2. ตัวอย่าง Alert Rule**

```yaml
groups:
  - name: ecommerce_postgresql_alerts
    rules:
      # Alert 1: Cache hit ratio ต่ำ (ตามที่โจทย์กำหนด)
      - alert: EcommerceDBCacheHitRatioLow
        expr: |
          (
            sum(rate(pg_stat_database_blks_hit{datname="ecommerce_prod"}[5m]))
            /
            (sum(rate(pg_stat_database_blks_hit{datname="ecommerce_prod"}[5m]))
             + sum(rate(pg_stat_database_blks_read{datname="ecommerce_prod"}[5m])))
          ) * 100 < 95
        for: 10m
        labels:
          severity: warning
          team: platform
        annotations:
          summary: "Cache hit ratio ต่ำกว่า 95% นาน 10 นาที"
          description: "อาจกระทบความเร็วหน้า checkout ตรวจสอบ shared_buffers และ query pattern"

      # Alert 2: Connection ใกล้เต็ม (ตามที่โจทย์กำหนด)
      - alert: EcommerceDBConnectionsNearFull
        expr: |
          pg_stat_activity_count{datname="ecommerce_prod"}
          / pg_settings_max_connections * 100 > 85
        for: 3m
        labels:
          severity: critical
          team: platform
        annotations:
          summary: "Connection ใช้ไปเกิน 85% ของ max_connections"
          description: "เสี่ยงรับ order ใหม่ไม่ได้ ตรวจสอบ connection pool (PgBouncer) และ connection leak"

      # Alert 3: Long-running query บนตารางสำคัญ
      - alert: EcommerceDBLongRunningQuery
        expr: pg_long_running_queries_count > 0
        for: 5m
        labels:
          severity: warning
          team: platform
        annotations:
          summary: "พบ query ที่รันนานเกิน 5 นาที"
          description: "อาจกระทบ order/payment flow ตรวจสอบด้วย pg_stat_activity"

      # Alert 4: Deadlock เพิ่มขึ้น
      - alert: EcommerceDBDeadlockDetected
        expr: increase(pg_stat_database_deadlocks{datname="ecommerce_prod"}[5m]) > 0
        for: 1m
        labels:
          severity: warning
          team: platform
        annotations:
          summary: "ตรวจพบ deadlock ในช่วง 5 นาทีที่ผ่านมา"
          description: "ตรวจสอบลำดับการ lock ในกระบวนการตัดสต็อกและชำระเงิน"

      # Alert 5: Table bloat บนตารางสำคัญ
      - alert: EcommerceDBHighDeadTuplePct
        expr: |
          pg_stat_user_tables_n_dead_tup{relname=~"orders|payments|inventory"}
          / (pg_stat_user_tables_n_dead_tup{relname=~"orders|payments|inventory"}
             + pg_stat_user_tables_n_live_tup{relname=~"orders|payments|inventory"}) * 100 > 20
        for: 30m
        labels:
          severity: warning
          team: platform
        annotations:
          summary: "Dead tuple เกิน 20% บนตารางสำคัญ"
          description: "ตรวจสอบว่า autovacuum ตามทันหรือไม่ พิจารณาปรับ autovacuum_vacuum_scale_factor"
```

**3. เครื่องมือที่เลือกใช้และเหตุผล**

เนื่องจากเป็น e-commerce ทั่วไป (ไม่มีข้อจำกัด compliance สูงเหมือนธนาคาร) แนวทางที่สมดุลคือ:

- **Layer พื้นฐาน**: ใช้ **Prometheus + `postgres_exporter` + Grafana + Alertmanager** เป็นแกนหลัก เพราะ:
  - ทีม engineering ของ e-commerce ส่วนใหญ่มักใช้ Prometheus/Grafana อยู่แล้วสำหรับ monitor ส่วนอื่นของระบบ (application, Kubernetes, ฯลฯ) การรวม PostgreSQL เข้ามาใน stack เดียวกันทำให้เห็นภาพรวมทั้งระบบในที่เดียว (correlate database metric กับ application metric ได้ง่าย เช่น ดูว่า latency ของ checkout API สัมพันธ์กับ cache hit ratio ที่ตกลงหรือไม่)
  - ต้นทุนต่ำ เหมาะกับช่วงเริ่มต้นธุรกิจที่ยังต้องควบคุมค่าใช้จ่าย
- **เสริมด้วย Managed SaaS (เช่น pganalyze) แบบเลือกใช้เฉพาะจุด**: หากทีมไม่มี DBA ที่เชี่ยวชาญลึกพอจะตีความ query plan หรือแนะนำ index เอง ควรพิจารณาเพิ่ม pganalyze หรือเทียบเคียง เพื่อช่วย deep-dive วิเคราะห์ query performance โดยเฉพาะช่วงก่อน/หลัง major sale event (เช่น 11.11, Black Friday) ที่ต้องการความมั่นใจสูงสุดว่าระบบจะรับ load ได้

แนวทางนี้ให้ความสมดุลระหว่างต้นทุน ความยืดหยุ่น และความเชี่ยวชาญเฉพาะทางที่ทีมมีอยู่จริง

</details>

---

## บทถัดไป

เมื่อเราสามารถ monitor และตรวจจับปัญหาได้แล้ว ขั้นตอนต่อไปคือการนำข้อมูลที่ได้จาก monitoring ไปใช้ **tune ค่า configuration ของ PostgreSQL** ให้เหมาะสมกับ workload และ hardware ของระบบจริง ไปต่อกันที่:

**[Part 073: Tuning postgresql.conf](./part-073-tuning-postgresql-conf.md)**
