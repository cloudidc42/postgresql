# Part 078: TimescaleDB สำหรับ Time-Series Data

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับมืออาชีพ | Part 078

---

## เป้าหมายการเรียนรู้

เมื่อเรียนจบบทนี้ ผู้เรียนจะสามารถ:

1. อธิบายได้ว่า **time-series data** คืออะไร และเหตุใดข้อมูลประเภทนี้จึงมีลักษณะการใช้งานที่ต่างจากข้อมูลเชิงธุรกรรม (OLTP) ทั่วไป
2. อธิบายข้อจำกัดของ PostgreSQL แบบมาตรฐานเมื่อต้องรับมือกับ time-series data ขนาดใหญ่มาก ๆ เช่น ปัญหา index bloat และการ query ช่วงเวลา
3. เข้าใจว่า **TimescaleDB** คือ extension ของ PostgreSQL ที่เพิ่มความสามารถด้าน time-series โดยยังคงใช้ SQL มาตรฐานได้ทั้งหมด
4. สร้างและใช้งาน **hypertable** ด้วย `create_hypertable()` และเข้าใจกลไกการแบ่ง **chunk** อัตโนมัติตามเวลา
5. เข้าใจความสัมพันธ์ระหว่าง chunk ของ TimescaleDB กับ table partitioning ที่เรียนไปแล้วใน Part 054
6. สร้างและใช้งาน **Continuous Aggregate** เพื่อทำ pre-aggregation ข้อมูล time-series แบบ incremental
7. ใช้ **Compression policy** เพื่อบีบอัดข้อมูลเก่าโดยอัตโนมัติ และประหยัดพื้นที่ดิสก์
8. ใช้ **Data Retention Policy** เพื่อลบข้อมูลเก่าที่ไม่จำเป็นต้องเก็บอีกต่อไปโดยอัตโนมัติ
9. ใช้ฟังก์ชัน `time_bucket()` เพื่อทำ downsampling และวิเคราะห์ข้อมูลตามช่วงเวลาที่ยืดหยุ่น
10. ออกแบบและสร้างระบบวิเคราะห์ page view / analytics event แบบเต็มรูปแบบสำหรับเว็บ e-commerce โดยใช้ TimescaleDB

---

## หมายเหตุสำคัญก่อนเริ่มบทเรียน

**TimescaleDB ไม่ใช่ส่วนหนึ่งของ PostgreSQL core** แต่เป็น **extension** ที่ต้องติดตั้งเพิ่มเติมแยกต่างหาก ซึ่งต่างจาก extension มาตรฐานอย่าง `pgcrypto` หรือ `uuid-ossp` ที่มักจะมากับ PostgreSQL distribution ทั่วไป TimescaleDB ต้อง:

- ติดตั้ง package ของ TimescaleDB ลงในเครื่อง PostgreSQL server (เช่นผ่าน `apt`, `yum`, Docker image `timescale/timescaledb`, หรือใช้บริการ managed cloud อย่าง Timescale Cloud)
- แก้ไข `postgresql.conf` เพื่อเพิ่ม `timescaledb` เข้าไปใน `shared_preload_libraries`
- รีสตาร์ท PostgreSQL server
- จากนั้นจึงรัน `CREATE EXTENSION timescaledb;` ในแต่ละ database ที่ต้องการใช้งาน

> **สภาพแวดล้อมของผู้เรียนหลายคนอาจไม่มี TimescaleDB ติดตั้งไว้** เช่น sandbox ฝึกฝนทั่วไป, บริการ PostgreSQL แบบ managed บางเจ้าที่ไม่รองรับ extension นี้ หรือ container PostgreSQL มาตรฐานที่ยังไม่ได้ build ด้วย TimescaleDB หากรัน `CREATE EXTENSION timescaledb;` แล้วได้ error ประมาณ:
>
> ```
> ERROR:  could not open extension control file
>         ".../timescaledb.control": No such file or directory
> ```
>
> แสดงว่าเครื่องนั้นยังไม่ได้ติดตั้ง TimescaleDB ผู้เรียนควรไปติดตั้งตาม **เอกสารทางการ (official documentation)** ที่ https://docs.timescale.com ก่อน จึงจะรันโค้ดในบทนี้ได้จริงบนเครื่องของตนเอง บทนี้เขียนโค้ด SQL ให้ตรงกับพฤติกรรมจริงของ TimescaleDB เวอร์ชันปัจจุบัน (2.x) ทุกประการ เพื่อให้ผู้เรียนนำไปใช้งานได้ทันทีเมื่อมีสภาพแวดล้อมที่ติดตั้งไว้แล้ว

TimescaleDB เหมาะสำหรับกรณีที่ธุรกิจมีข้อมูลจำนวนมหาศาลที่ผูกกับเวลา เช่น log, metrics, sensor data, financial tick data, หรือในบริบทของคอร์สนี้คือ **web analytics event ของร้านค้าออนไลน์** — บทนี้จึงถือเป็นบทเสริมความรู้เฉพาะทาง (specialized extension) ที่ผู้เรียนควรรู้จักไว้ เพื่อเลือกใช้ให้ถูกสถานการณ์ ไม่ใช่สิ่งที่ต้องใช้ในทุกโปรเจกต์

---

## เตรียมข้อมูล

เราจะสร้าง schema ใหม่สำหรับกรณีศึกษา **การติดตามพฤติกรรมผู้เข้าชมหน้าสินค้า (product page view analytics)** ของเว็บ e-commerce ซึ่งเป็นตัวอย่างคลาสสิกของ time-series data: ทุกครั้งที่มีคนเข้าดูหน้าสินค้า ระบบจะบันทึก event หนึ่งแถว พร้อม timestamp, สินค้าที่ดู, ลูกค้า (ถ้า login), ประเภทอุปกรณ์ และระยะเวลาที่อยู่ในหน้านั้น

```sql
-- ต้องรันโดย superuser หรือ role ที่มีสิทธิ์ CREATE EXTENSION
CREATE EXTENSION IF NOT EXISTS timescaledb;
```

```
CREATE EXTENSION
```

> หากใช้ `psql` แล้วพิมพ์ `\dx` จะเห็น `timescaledb` อยู่ในรายการ extension ที่ติดตั้งในฐานข้อมูลนี้แล้ว

สร้างตารางเก็บ event แบบธรรมดาก่อน (ยังไม่ใช่ hypertable):

```sql
CREATE TABLE page_views (
    view_time            TIMESTAMPTZ NOT NULL,
    product_id           INTEGER,
    customer_id          INTEGER,
    device_type          VARCHAR(20),
    session_duration_sec INTEGER
);
```

```
CREATE TABLE
```

แปลงตารางนี้ให้เป็น **hypertable** (รายละเอียดกลไกจะอธิบายใน Step 774):

```sql
SELECT create_hypertable('page_views', 'view_time');
```

```
     create_hypertable
------------------------------
 (1,public,page_views,t)
(1 row)
```

จากนั้นสร้างข้อมูลตัวอย่างจำลอง page view ย้อนหลังหลายเดือน (สมมติว่าร้านค้ามีสินค้า 200 รายการ, ลูกค้าที่ล็อกอิน 5,000 คน, และมี event ประมาณ 2 ล้านแถวกระจายในช่วง 6 เดือนที่ผ่านมา) โดยใช้ `generate_series` เพื่อจำลองข้อมูลจำนวนมากอย่างรวดเร็ว:

```sql
INSERT INTO page_views (view_time, product_id, customer_id, device_type, session_duration_sec)
SELECT
    -- สุ่มเวลาแบบกระจายทั่วทั้ง 6 เดือนย้อนหลังจนถึงตอนนี้
    now() - (random() * interval '180 days'),
    (random() * 199 + 1)::int,                         -- product_id: 1-200
    CASE WHEN random() < 0.7                            -- 70% ของ event มีลูกค้าที่ login
         THEN (random() * 4999 + 1)::int
         ELSE NULL END,
    (ARRAY['desktop', 'mobile', 'tablet'])[floor(random() * 3 + 1)],
    (random() * 590 + 10)::int                          -- อยู่ในหน้า 10-600 วินาที
FROM generate_series(1, 2000000) AS s;
```

```
INSERT 0 2000000
```

ตรวจสอบขนาดข้อมูลและช่วงเวลาที่ได้:

```sql
SELECT
    count(*)                    AS total_rows,
    min(view_time)              AS earliest,
    max(view_time)              AS latest,
    pg_size_pretty(hypertable_size('page_views')) AS total_size
FROM page_views;
```

```
 total_rows |            earliest             |            latest             | total_size
------------+----------------------------------+--------------------------------+------------
    2000000 | 2026-03-29 08:12:41.203112+00   | 2026-09-25 08:12:41.203112+00 | 184 MB
(1 row)
```

> `hypertable_size()` เป็นฟังก์ชันของ TimescaleDB ที่คำนวณขนาดรวมของ hypertable ทั้งหมด (รวมทุก chunk และ index) ต่างจาก `pg_relation_size()` ที่ใช้กับตารางธรรมดา เพราะ hypertable จริง ๆ แล้วประกอบด้วยตารางลูกหลายตัว (chunk)

เพิ่ม index ที่ใช้บ่อยสำหรับ query วิเคราะห์ตามสินค้า:

```sql
CREATE INDEX idx_page_views_product_time
    ON page_views (product_id, view_time DESC);
```

```
CREATE INDEX
```

ข้อมูลชุดนี้จะถูกใช้ตลอดทั้งบท ทั้งใน Step 771-780 และในแบบฝึกหัดท้ายบท

---

## Step 771: Time-Series Data คืออะไร

**Time-series data** คือข้อมูลที่มี **timestamp เป็นมิติหลัก (primary dimension)** ของทุกแถว กล่าวคือแต่ละแถวแทน "เหตุการณ์หรือค่าที่วัดได้ ณ จุดเวลาหนึ่ง" และข้อมูลใหม่จะถูกเขียนเพิ่มเข้ามาเรื่อย ๆ ตามลำดับเวลา (append-only เป็นหลัก) มากกว่าจะถูกแก้ไขย้อนหลัง

ตัวอย่างของ time-series data ที่พบได้ทั่วไป:

| ประเภทข้อมูล | ตัวอย่าง |
|---|---|
| Web/App analytics | page view, click event, session log |
| IoT / Sensor | อุณหภูมิ, ความชื้น, ตำแหน่ง GPS ที่ส่งเข้ามาทุกวินาที |
| Infrastructure metrics | CPU usage, memory, network throughput ของ server |
| Financial data | ราคาหุ้น, tick data, exchange rate ที่เปลี่ยนทุกวินาที |
| E-commerce (กรณีศึกษาบทนี้) | page view, add-to-cart event, search query log |

ลักษณะเฉพาะของ time-series data ที่ต่างจากข้อมูลเชิงธุรกรรมทั่วไป (เช่นตาราง `orders`, `customers` ที่เราใช้มาตลอดคอร์ส):

1. **Write-heavy, append-mostly** — ข้อมูลใหม่จะถูก insert เข้ามาตลอดเวลา แทบไม่มีการ `UPDATE` แถวเก่า
2. **Volume สูงมาก** — ระบบ IoT หรือ analytics จริงอาจสร้าง event หลักล้านถึงพันล้านแถวต่อวัน
3. **Query ส่วนใหญ่มีเงื่อนไขเรื่องเวลา** — เช่น "ยอดวิว 7 วันล่าสุด", "ค่าเฉลี่ยรายชั่วโมงของเดือนที่แล้ว"
4. **ข้อมูลเก่ามีค่าลดลงตามเวลา** — ข้อมูลอายุ 2 ปีอาจไม่จำเป็นต้องมี resolution ละเอียดเท่าข้อมูลของเมื่อวาน (นำไปสู่แนวคิด downsampling และ retention policy ที่จะเรียนใน Step 778-779)
5. **มักต้องการ aggregation ตามช่วงเวลา** เช่น รายชั่วโมง รายวัน รายสัปดาห์ มากกว่าการดึงข้อมูลรายแถว

ตัวอย่าง query ที่เป็นแบบฉบับของ time-series analysis บนตาราง `page_views` ที่เราเตรียมไว้:

```sql
-- ยอดวิวหน้าสินค้ารายวัน ใน 7 วันล่าสุด
SELECT
    date_trunc('day', view_time) AS day,
    count(*)                     AS views
FROM page_views
WHERE view_time >= now() - interval '7 days'
GROUP BY 1
ORDER BY 1;
```

```
          day           | views
-------------------------+-------
 2026-09-19 00:00:00+00 | 27134
 2026-09-20 00:00:00+00 | 27021
 2026-09-21 00:00:00+00 | 26988
 2026-09-22 00:00:00+00 | 27350
 2026-09-23 00:00:00+00 | 27196
 2026-09-24 00:00:00+00 | 27267
 2026-09-25 00:00:00+00 | 12043
(7 rows)
```

นี่คือรูปแบบ query ที่ TimescaleDB ถูกออกแบบมาให้ทำงานได้เร็วและมีประสิทธิภาพเป็นพิเศษ เมื่อเทียบกับ PostgreSQL แบบมาตรฐานที่ scale ข้อมูลขนาดนี้ได้ยากกว่า (รายละเอียดใน Step 772)

---

## Step 772: ทำไม PostgreSQL ธรรมดาไม่พอสำหรับ Time-Series ขนาดใหญ่มาก

PostgreSQL แบบมาตรฐาน (แม้จะใช้ table partitioning ที่เรียนใน Part 054 แล้ว) ยังคงมีข้อจำกัดหลายอย่างเมื่อข้อมูล time-series เติบโตถึงระดับหลักร้อยล้านถึงพันล้านแถว:

### 1. ปัญหา Index Bloat

เมื่อ insert ข้อมูลใหม่เข้ามาตลอดเวลาในตารางเดียวที่มีขนาดใหญ่ B-tree index (เช่น index บน `view_time`) จะยิ่งใหญ่ขึ้นเรื่อย ๆ และการ insert แต่ละครั้งต้องแก้ไข index page ที่อาจกระจัดกระจายทั่ว index ทั้งต้น ทำให้:

- Index มีขนาดใหญ่เกินกว่าจะ cache ไว้ใน memory (`shared_buffers`) ได้ทั้งหมด
- การ insert ช้าลงเพราะต้อง random I/O เข้าไปแก้ไข index page ที่ไม่อยู่ใน cache
- `VACUUM` ต้องประมวลผลตารางขนาดมหาศาลทุกครั้ง ใช้เวลานานและกิน I/O สูง

```sql
-- ตัวอย่าง: ถ้า page_views เป็นตารางธรรมดาขนาด 500 ล้านแถว
-- index บน view_time อาจมีขนาดหลายสิบ GB
-- และการ insert ใหม่ทุกครั้งจะต้องแก้ไขหน้า index ที่กระจายอยู่ทั่วดิสก์
SELECT pg_size_pretty(pg_relation_size('idx_page_views_product_time'));
```

```
 pg_size_pretty
----------------
 43 MB
```

(ในตัวอย่าง 2 ล้านแถวของเรายังเล็กอยู่ แต่ลองจินตนาการขยายเป็น 500 เท่า)

### 2. การ Query ช่วงเวลาไม่มีประสิทธิภาพเท่าที่ควร

แม้จะมี index บน `view_time` แต่เมื่อ query ต้องการเฉพาะข้อมูล "7 วันล่าสุด" จากตารางที่มีข้อมูลสะสม 2 ปี PostgreSQล ยังต้องอาศัย B-tree index scan ซึ่งประสิทธิภาพจะลดลงเมื่อ index มีขนาดใหญ่มาก ต่างจากการที่ข้อมูลถูกจัดเก็บแยกเป็นก้อนตามช่วงเวลาไว้ล่วงหน้า ซึ่งจะสามารถ "ข้าม" ก้อนข้อมูลที่ไม่เกี่ยวข้องได้ทั้งก้อนทันที (chunk exclusion / partition pruning)

### 3. Manual Partitioning ทำได้ แต่ต้องดูแลเอง

Part 054 สอนเรื่อง declarative partitioning ของ PostgreSQL ซึ่งช่วยได้มากในการแบ่งข้อมูลตามช่วงเวลา แต่ผู้ดูแลระบบยังต้อง:

- เขียน cron job หรือ scheduler มาสร้าง partition ใหม่ล่วงหน้าเองทุกเดือน/สัปดาห์
- จัดการการลบ partition เก่าด้วยตนเอง (data retention)
- ไม่มี pre-aggregation ให้ในตัว ต้องสร้าง materialized view และ refresh เองทั้งหมด (ซึ่งมักจะ refresh ใหม่ทั้งหมดทุกครั้ง ไม่ efficient)
- ไม่มีกลไก compression สำหรับข้อมูลเก่าที่มีในตัว (built-in)

### 4. Aggregation แบบ Real-time บนข้อมูลจำนวนมากทำได้ช้า

```sql
-- Query แบบนี้ ถ้าทำบนตารางธรรมดาขนาดหลายร้อยล้านแถว
-- อาจต้อง scan ข้อมูลจำนวนมากทุกครั้งที่เรียก
SELECT date_trunc('hour', view_time) AS hour, count(*)
FROM page_views
GROUP BY 1
ORDER BY 1;
```

หากมี dashboard ที่ต้อง refresh ข้อมูลนี้ทุกไม่กี่วินาที การคำนวณซ้ำจากศูนย์ทุกครั้งจะกิน CPU และ I/O มหาศาลโดยไม่จำเป็น

### สรุปช่องว่างที่ TimescaleDB เข้ามาเติมเต็ม

| ปัญหา | วิธีแก้แบบ PostgreSQL ธรรมดา | วิธีแก้ของ TimescaleDB |
|---|---|---|
| Index bloat จากตารางใหญ่ | Manual partitioning (Part 054) | Automatic chunking (hypertable) |
| ต้องสร้าง partition ล่วงหน้าเอง | เขียน cron/scheduler เอง | สร้าง chunk อัตโนมัติเมื่อ insert |
| Aggregation ซ้ำซ้อนทุกครั้ง | Materialized view + refresh เอง (full refresh) | Continuous Aggregate (incremental refresh) |
| ข้อมูลเก่ากินพื้นที่มาก | บีบอัดเองหรือย้ายไป archive เอง | Native compression policy |
| ลบข้อมูลเก่าตามนโยบาย | DELETE/DROP partition เอง | Retention policy อัตโนมัติ |

Step ถัดไปจะแนะนำว่า TimescaleDB แก้ปัญหาเหล่านี้อย่างไรในเชิงสถาปัตยกรรม

---

## Step 773: TimescaleDB คืออะไร

**TimescaleDB** คือ **PostgreSQL extension แบบ open-source** ที่พัฒนาโดยบริษัท Timescale ซึ่งเพิ่มความสามารถเฉพาะทางด้าน time-series ให้กับ PostgreSQL โดยไม่ต้องเปลี่ยนไปใช้ฐานข้อมูลใหม่ทั้งหมด จุดเด่นที่สำคัญที่สุดคือ:

> **TimescaleDB ยังคงเป็น PostgreSQL 100%** — ใช้ SQL มาตรฐาน, join กับตารางอื่นได้ปกติ, ใช้ driver/ORM เดิมที่ใช้กับ PostgreSQL ได้ทันที, และรองรับ extension อื่น ๆ ของ PostgreSQL ร่วมกันได้ (เช่น `PostGIS`, `pg_stat_statements`)

ความสามารถหลักที่ TimescaleDB เพิ่มเข้ามา:

1. **Hypertable** — ตารางเสมือนที่แบ่งข้อมูลออกเป็น chunk ย่อยตามเวลาโดยอัตโนมัติ (Step 774)
2. **Continuous Aggregate** — materialized view พิเศษที่ update แบบ incremental ไม่ต้อง refresh ใหม่ทั้งหมด (Step 776)
3. **Native Compression** — บีบอัดข้อมูลแบบ columnar สำหรับ chunk เก่า ลดขนาดได้มากถึง 90%+ ในหลายกรณี (Step 777)
4. **Data Retention Policy** — ลบ chunk เก่าอัตโนมัติตามอายุ (Step 778)
5. **Time-series specific functions** เช่น `time_bucket()`, `first()`, `last()`, `time_bucket_gapfill()` (Step 779)
6. **Automated background job scheduler** — TimescaleDB มี background worker ของตัวเองที่คอยรัน policy ต่าง ๆ (compression, retention, continuous aggregate refresh) ตามตารางเวลาที่กำหนดไว้ โดยไม่ต้องพึ่ง cron ภายนอก

### ตรวจสอบว่า TimescaleDB ติดตั้งอยู่หรือไม่ และเวอร์ชันใด

```sql
SELECT extname, extversion
FROM pg_extension
WHERE extname = 'timescaledb';
```

```
   extname   | extversion
-------------+------------
 timescaledb | 2.15.3
(1 row)
```

```sql
-- ดูข้อมูลสรุปทั้งหมดของ TimescaleDB
SELECT * FROM timescaledb_information.hypertables;
```

```
 hypertable_schema | hypertable_name | owner  | num_dimensions | num_chunks | compression_enabled | tablespaces
--------------------+-----------------+--------+----------------+------------+----------------------+-------------
 public            | page_views      | app    |              1 | 26         | f                    |
(1 row)
```

### แนวคิดสำคัญ: "It's just PostgreSQL"

เพราะ TimescaleDB เป็นเพียง extension เมื่อเรา query ตาราง `page_views` ทีมพัฒนาแอปพลิเคชันแทบไม่รู้สึกถึงความต่างจากการ query ตารางธรรมดาเลย:

```sql
-- Query นี้ทำงานได้เหมือน table ปกติทุกประการ
SELECT product_id, count(*) AS views
FROM page_views
WHERE view_time >= '2026-09-01'
GROUP BY product_id
ORDER BY views DESC
LIMIT 5;
```

```
 product_id | views
------------+-------
        142 |  1823
         77 |  1799
        203 |  1788
         12 |  1765
         98 |  1750
(5 rows)
```

TimescaleDB จะทำงานเบื้องหลังในการเลือกเฉพาะ chunk ที่เกี่ยวข้อง (chunk exclusion) โดยที่ผู้ใช้ไม่ต้องเขียนโค้ดพิเศษใด ๆ เพิ่มเติม — ความแตกต่างที่แท้จริงอยู่ที่ **ประสิทธิภาพ** และ **ความสามารถพิเศษเพิ่มเติม** (continuous aggregate, compression, retention) ที่จะได้เรียนต่อไป

---

## Step 774: Hypertable — แนวคิดหลักของ TimescaleDB

**Hypertable** คือแนวคิดหลักและเป็นหัวใจของ TimescaleDB มันคือ **ตารางเสมือน (virtual table)** ที่ผู้ใช้มองเห็นและใช้งานเหมือนตารางธรรมดาตัวเดียว แต่เบื้องหลัง TimescaleDB จะแบ่งข้อมูลออกเป็นตารางลูกจำนวนมากที่เรียกว่า **chunk** โดยอัตโนมัติ ตามช่วงเวลา (และอาจรวมมิติอื่นด้วย เช่น `product_id` แบบ hash partition)

### การสร้าง Hypertable

```sql
-- ขั้นตอนพื้นฐาน: สร้างตารางธรรมดาก่อน แล้วค่อยแปลงเป็น hypertable
CREATE TABLE page_views (
    view_time            TIMESTAMPTZ NOT NULL,
    product_id           INTEGER,
    customer_id          INTEGER,
    device_type          VARCHAR(20),
    session_duration_sec INTEGER
);

SELECT create_hypertable('page_views', 'view_time');
```

```
     create_hypertable
------------------------------
 (1,public,page_views,t)
(1 row)
```

พารามิเตอร์ของ `create_hypertable()` ที่ใช้บ่อย:

```sql
SELECT create_hypertable(
    'page_views',            -- ชื่อตาราง (regclass)
    'view_time',             -- คอลัมน์เวลาที่จะใช้แบ่ง chunk
    chunk_time_interval => INTERVAL '7 days',  -- กำหนดขนาดช่วงเวลาต่อ 1 chunk (ค่า default คือ 7 วัน)
    if_not_exists => TRUE    -- ไม่ error หากเป็น hypertable อยู่แล้ว
);
```

> ตั้งแต่ TimescaleDB 2.13 เป็นต้นไป ยังมี syntax แบบใหม่ที่ใช้ dimension builder เช่น `create_hypertable('page_views', by_range('view_time'))` ซึ่งให้ผลลัพธ์เทียบเท่ากับ syntax แบบเดิมข้างต้น สำหรับกรณีพื้นฐานทั้งสองแบบใช้แทนกันได้

### แปลงตารางที่มีข้อมูลอยู่แล้วให้เป็น Hypertable

หากมีตารางที่มีข้อมูลอยู่แล้ว (ไม่ใช่ตารางว่าง) ต้องระบุ `migrate_data => TRUE`:

```sql
SELECT create_hypertable(
    'page_views',
    'view_time',
    migrate_data => TRUE
);
```

> ในโปรเจกต์จริงที่มีข้อมูลจำนวนมาก การ migrate ข้อมูลจำนวนมหาศาลด้วยวิธีนี้อาจใช้เวลานานและ lock ตาราง ควรทำในช่วง maintenance window หรือพิจารณาใช้วิธี "ตารางใหม่ + ทยอยย้ายข้อมูล + สลับชื่อ" แทน

### Hypertable ยังคงรองรับ Constraint และ Index เกือบทั้งหมด

```sql
-- Index ธรรมดายังใช้ได้ปกติ TimescaleDB จะสร้าง index นี้ในทุก chunk ให้อัตโนมัติ
CREATE INDEX idx_page_views_customer
    ON page_views (customer_id, view_time DESC);
```

```
CREATE INDEX
```

> ข้อจำกัดที่ควรรู้: **PRIMARY KEY หรือ UNIQUE constraint ต้องรวมคอลัมน์เวลา (partitioning column) อยู่ด้วยเสมอ** เนื่องจากข้อมูลถูกกระจายไปตาม chunk หลายตัว การบังคับ uniqueness แบบ global (ไม่รวมคอลัมน์เวลา) ข้ามทุก chunk จึงทำไม่ได้อย่างมีประสิทธิภาพ

```sql
-- ตัวอย่างที่ "ทำไม่ได้" ถ้า id ไม่มีคอลัมน์เวลาประกอบ
-- ALTER TABLE page_views ADD PRIMARY KEY (view_id);  -- ERROR หาก view_id ไม่รวม view_time

-- ตัวอย่างที่ถูกต้อง: รวมคอลัมน์เวลาไว้ใน composite key
-- ALTER TABLE page_views ADD PRIMARY KEY (view_id, view_time);
```

### ตรวจสอบ Hypertable ที่มีอยู่

```sql
SELECT hypertable_name, num_dimensions, num_chunks
FROM timescaledb_information.hypertables
WHERE hypertable_name = 'page_views';
```

```
 hypertable_name | num_dimensions | num_chunks
------------------+----------------+------------
 page_views       |              1 |         26
(1 row)
```

### ทำไม Hypertable จึงแก้ปัญหา Index Bloat ได้

เพราะแต่ละ chunk คือตารางลูกที่ **แยก index ของตัวเอง** เมื่อ insert ข้อมูลใหม่ (ซึ่งมักจะมี timestamp ล่าสุดเสมอ) ข้อมูลจะถูกเขียนลงเฉพาะ chunk ล่าสุดเท่านั้น ทำให้ index ของ chunk นั้นมีขนาดเล็กพอที่จะอยู่ใน memory cache ได้ตลอดเวลา ต่างจากตารางเดี่ยวขนาดมหาศาลที่ index ใหญ่เกินจะ cache ทั้งหมด

---

## Step 775: Chunk คืออะไร — ความสัมพันธ์กับ Partition ใน Part 054

**Chunk** คือหน่วยย่อยที่ hypertable ถูกแบ่งออกมาจริง ๆ ในระดับกายภาพ โดยแต่ละ chunk คือ **ตารางจริงหนึ่งตาราง** ที่ถูกสร้างขึ้นด้วยกลไก **table inheritance / declarative partitioning** ของ PostgreSQL เอง (ใช้กลไกเดียวกับที่เรียนใน Part 054 เป๊ะ ๆ) เพียงแต่ TimescaleDB เป็นผู้จัดการสร้าง จัดระเบียบ และ maintain ให้อัตโนมัติทั้งหมด

### ความสัมพันธ์กับ Range Partitioning (Part 054)

| แนวคิด (Part 054: Manual Partitioning) | แนวคิดเทียบเท่าใน TimescaleDB |
|---|---|
| Partitioned table (parent) | Hypertable |
| Partition (child table) แต่ละตัว | Chunk แต่ละตัว |
| `CREATE TABLE ... PARTITION OF ... FOR VALUES FROM ... TO ...` | สร้างอัตโนมัติเมื่อ insert ข้อมูลใหม่เข้าช่วงเวลาที่ยังไม่มี chunk รองรับ |
| ต้องเขียน script/cron สร้าง partition ล่วงหน้าเอง | TimescaleDB สร้าง chunk ใหม่ให้อัตโนมัติ "แบบขี้เกียจ" (lazy) ทันทีที่มีข้อมูลเข้ามาในช่วงเวลาที่ยังไม่มี chunk |
| Partition pruning ตอน query | Chunk exclusion (concept เดียวกัน) |

พูดง่าย ๆ คือ **hypertable = partitioned table ที่ TimescaleDB บริหารจัดการให้แบบอัตโนมัติทั้งหมด** ผู้ใช้ไม่ต้องกำหนดขอบเขตของแต่ละ chunk เอง ไม่ต้องคอยสร้าง chunk ใหม่ล่วงหน้า และไม่ต้องเขียน trigger สำหรับ routing ข้อมูลเหมือนวิธี manual partitioning แบบเก่า

### ดู Chunk ทั้งหมดของ Hypertable

```sql
SELECT show_chunks('page_views');
```

```
              show_chunks
-----------------------------------------
 _timescaledb_internal._hyper_1_1_chunk
 _timescaledb_internal._hyper_1_2_chunk
 _timescaledb_internal._hyper_1_3_chunk
 ...
 _timescaledb_internal._hyper_1_26_chunk
(26 rows)
```

ดูรายละเอียดของแต่ละ chunk พร้อมช่วงเวลาและขนาด:

```sql
SELECT
    chunk_name,
    range_start,
    range_end,
    pg_size_pretty(total_bytes) AS size
FROM timescaledb_information.chunks
JOIN chunks_detailed_size('page_views') USING (chunk_name)
WHERE hypertable_name = 'page_views'
ORDER BY range_start
LIMIT 5;
```

```
       chunk_name        |        range_start         |         range_end          |  size
--------------------------+------------------------------+------------------------------+---------
 _hyper_1_1_chunk        | 2026-03-26 00:00:00+00      | 2026-04-02 00:00:00+00      | 7104 kB
 _hyper_1_2_chunk        | 2026-04-02 00:00:00+00      | 2026-04-09 00:00:00+00      | 7096 kB
 _hyper_1_3_chunk        | 2026-04-09 00:00:00+00      | 2026-04-16 00:00:00+00      | 7112 kB
 _hyper_1_4_chunk        | 2026-04-16 00:00:00+00      | 2026-04-23 00:00:00+00      | 7080 kB
 _hyper_1_5_chunk        | 2026-04-23 00:00:00+00      | 2026-04-30 00:00:00+00      | 7136 kB
(5 rows)
```

### Chunk Time Interval — ขนาดของแต่ละ Chunk

ค่า default ของ `chunk_time_interval` คือ **7 วัน** ต่อ 1 chunk แต่สามารถปรับได้ตามปริมาณข้อมูลที่เขียนเข้ามาจริง หลักการคร่าว ๆ ที่ TimescaleDB แนะนำคือ ควรตั้งค่าให้แต่ละ chunk (รวม index) มีขนาดพอดีกับ 25% ของ memory ที่มี (เพื่อให้ chunk ล่าสุดที่ถูก insert บ่อยที่สุดยังคง cache อยู่ใน memory ได้ทั้งหมด)

```sql
-- ปรับ chunk_time_interval สำหรับ hypertable ที่มีอยู่แล้ว
-- (จะมีผลกับ chunk ใหม่ที่จะถูกสร้างต่อจากนี้เท่านั้น ไม่กระทบ chunk เก่า)
SELECT set_chunk_time_interval('page_views', INTERVAL '1 day');
```

```
 set_chunk_time_interval
--------------------------

(1 row)
```

ตัวอย่างการเลือกขนาด chunk ตามปริมาณข้อมูล:

| ปริมาณ insert ต่อวัน | chunk_time_interval ที่เหมาะสม |
|---|---|
| หลักพันแถว/วัน | 7-30 วัน |
| หลักแสนถึงล้านแถว/วัน (เช่น page_views ของเรา) | 1-7 วัน |
| หลักสิบล้านแถว/วันขึ้นไป (IoT scale ใหญ่) | 1 ชั่วโมง - 1 วัน |

### ทำไมการแบ่ง Chunk จึงช่วยเรื่อง Query Performance

เมื่อ query มีเงื่อนไข `WHERE view_time >= now() - interval '7 days'` planner ของ TimescaleDB จะรู้ทันทีว่า chunk ที่มีช่วงเวลาเก่ากว่านั้นทั้งหมดไม่เกี่ยวข้อง และจะ **ข้าม (exclude)** chunk เหล่านั้นออกจาก query plan ไปเลย — เป็นแนวคิดเดียวกับ partition pruning ใน Part 054 เพียงแต่เกิดขึ้นอัตโนมัติในทุก query ที่มีเงื่อนไขเวลา

```sql
EXPLAIN (COSTS OFF)
SELECT count(*) FROM page_views
WHERE view_time >= now() - interval '7 days';
```

```
                         QUERY PLAN
------------------------------------------------------------
 Aggregate
   ->  Append
         ->  Seq Scan on _hyper_1_24_chunk
               Filter: (view_time >= (now() - '7 days'::interval))
         ->  Seq Scan on _hyper_1_25_chunk
               Filter: (view_time >= (now() - '7 days'::interval))
         ->  Seq Scan on _hyper_1_26_chunk
               Filter: (view_time >= (now() - '7 days'::interval))
(7 rows)
```

สังเกตว่า plan นี้แสดงเฉพาะ chunk ที่ 24-26 เท่านั้น (จากทั้งหมด 26+ chunk) ซึ่งพิสูจน์ว่า chunk เก่า ๆ ถูก exclude ออกไปตั้งแต่ชั้น planning แล้ว โดยไม่ต้องสแกนเลย

---

## Step 776: Continuous Aggregate

**Continuous Aggregate** คือ materialized view รูปแบบพิเศษของ TimescaleDB ที่ออกแบบมาสำหรับ time-series โดยเฉพาะ จุดต่างที่สำคัญที่สุดจาก materialized view ธรรมดาของ PostgreSQL คือ **การ refresh แบบ incremental** — แทนที่จะคำนวณ aggregation ใหม่ทั้งหมดทุกครั้ง (เหมือน `REFRESH MATERIALIZED VIEW` ปกติ) TimescaleDB จะคำนวณเฉพาะช่วงเวลาที่มีข้อมูลใหม่หรือเปลี่ยนแปลงเท่านั้น

### สร้าง Continuous Aggregate

```sql
CREATE MATERIALIZED VIEW page_views_hourly
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', view_time) AS bucket,
    product_id,
    device_type,
    count(*)                          AS total_views,
    count(DISTINCT customer_id)       AS unique_customers,
    avg(session_duration_sec)         AS avg_session_sec
FROM page_views
GROUP BY bucket, product_id, device_type;
```

```
CREATE MATERIALIZED VIEW
```

> สังเกตว่าต้องใช้ `time_bucket()` (จะอธิบายละเอียดใน Step 779) แทน `date_trunc()` ในการ group by ช่วงเวลาสำหรับ continuous aggregate โดยเฉพาะ และ `GROUP BY` ต้องมี `time_bucket()` เป็นหนึ่งใน expression เสมอ

### กำหนด Policy ให้ Refresh อัตโนมัติ

การสร้าง continuous aggregate เพียงอย่างเดียวยังไม่ทำให้มัน update อัตโนมัติ ต้องเพิ่ม policy:

```sql
SELECT add_continuous_aggregate_policy('page_views_hourly',
    start_offset      => INTERVAL '3 days',   -- ย้อนหลังไปคำนวณใหม่กี่ช่วงจากปัจจุบัน
    end_offset        => INTERVAL '1 hour',   -- เว้นข้อมูล 1 ชั่วโมงล่าสุดไว้ (ยังไม่ finalize)
    schedule_interval  => INTERVAL '1 hour'    -- รัน refresh job ทุกกี่ชั่วโมง
);
```

```
 add_continuous_aggregate_policy
-----------------------------------
                              1000
(1 row)
```

การตั้งค่านี้หมายความว่า: ทุก ๆ 1 ชั่วโมง TimescaleDB จะรัน background job มา refresh ข้อมูลของ 3 วันล่าสุด (ยกเว้นชั่วโมงล่าสุดที่ข้อมูลยังไม่นิ่ง) โดยอัตโนมัติ ไม่ต้องมีใครสั่ง manual

### Query Continuous Aggregate เหมือน View ทั่วไป

```sql
SELECT bucket, product_id, total_views, unique_customers, round(avg_session_sec, 1) AS avg_sec
FROM page_views_hourly
WHERE product_id = 142
  AND bucket >= now() - interval '1 day'
ORDER BY bucket DESC
LIMIT 5;
```

```
         bucket          | product_id | total_views | unique_customers | avg_sec
--------------------------+------------+-------------+-------------------+---------
 2026-09-25 07:00:00+00  |        142 |          14 |                10 |   298.3
 2026-09-25 06:00:00+00  |        142 |          16 |                12 |   312.7
 2026-09-25 05:00:00+00  |        142 |          15 |                11 |   287.1
 2026-09-25 04:00:00+00  |        142 |          13 |                 9 |   301.5
 2026-09-25 03:00:00+00  |        142 |          17 |                14 |   295.0
(5 rows)
```

### Manual Refresh (กรณีต้องการข้อมูลล่าสุดทันที ไม่รอ policy)

```sql
CALL refresh_continuous_aggregate('page_views_hourly',
    now() - interval '2 hours',
    now()
);
```

```
CALL
```

### เปรียบเทียบกับ Materialized View ธรรมดา

| | Materialized View ธรรมดา (PostgreSQL core) | Continuous Aggregate (TimescaleDB) |
|---|---|---|
| การ refresh | `REFRESH MATERIALIZED VIEW` คำนวณใหม่ทั้งหมดทุกครั้ง | Incremental — คำนวณเฉพาะช่วงที่เปลี่ยน |
| Concurrent read ระหว่าง refresh | ต้องใช้ `CONCURRENTLY` ถึงจะไม่ lock read | อ่านได้ปกติเสมอ ไม่ block |
| การตั้งเวลา refresh อัตโนมัติ | ต้องใช้ `pg_cron` หรือ scheduler ภายนอก | มี policy scheduler ในตัว (`add_continuous_aggregate_policy`) |
| เหมาะกับ | ข้อมูลที่ change ไม่บ่อย, ขนาดไม่ใหญ่มาก | Time-series ขนาดใหญ่ ที่ append ต่อเนื่อง |

### สร้าง Continuous Aggregate อีกระดับสำหรับภาพรวมรายวัน

```sql
CREATE MATERIALIZED VIEW page_views_daily
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 day', view_time) AS bucket,
    count(*)                         AS total_views,
    count(DISTINCT product_id)       AS distinct_products_viewed,
    count(DISTINCT customer_id)      AS unique_customers
FROM page_views
GROUP BY bucket;

SELECT add_continuous_aggregate_policy('page_views_daily',
    start_offset      => INTERVAL '30 days',
    end_offset        => INTERVAL '1 day',
    schedule_interval  => INTERVAL '6 hours'
);
```

```
CREATE MATERIALIZED VIEW
 add_continuous_aggregate_policy
-----------------------------------
                              1001
(1 row)
```

---

## Step 777: Compression — บีบอัดข้อมูลเก่าอัตโนมัติ

Time-series data มักมีลักษณะสำคัญคือ **ข้อมูลเก่ามักถูก query น้อยลงเรื่อย ๆ แต่ยังต้องเก็บไว้** (เพื่อการตรวจสอบย้อนหลัง, การรายงาน หรือข้อกำหนดทางกฎหมาย) TimescaleDB มีความสามารถ **Native Compression** ที่จะแปลงข้อมูลของ chunk เก่าจากรูปแบบ row-based ทั่วไป ให้เป็นรูปแบบ **columnar compression** ซึ่งสามารถลดขนาดพื้นที่จัดเก็บได้มากถึง 90%+ ในหลายกรณี

### เปิดใช้งาน Compression บน Hypertable

```sql
ALTER TABLE page_views SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'product_id',
    timescaledb.compress_orderby   = 'view_time DESC'
);
```

```
ALTER TABLE
```

คำอธิบายพารามิเตอร์:

- **`compress_segmentby`** — คอลัมน์ที่ใช้แบ่งกลุ่มข้อมูลก่อนบีบอัด ควรเลือกคอลัมน์ที่มักถูกใช้ใน `WHERE` หรือ `GROUP BY` บ่อย ๆ (ในที่นี้คือ `product_id` เพราะเรามักจะ query แยกตามสินค้า) ข้อมูลที่มีค่า segmentby เดียวกันจะถูกจัดกลุ่มไว้ด้วยกัน ทำให้ query ที่ filter ด้วยคอลัมน์นี้ไม่ต้องอ่านข้อมูลที่ไม่เกี่ยวข้อง
- **`compress_orderby`** — คอลัมน์ที่ใช้เรียงลำดับข้อมูลภายในแต่ละกลุ่มก่อนบีบอัด (มักเป็นคอลัมน์เวลา) ช่วยให้การบีบอัดมีประสิทธิภาพสูงขึ้นเพราะค่าที่เรียงต่อกันมักใกล้เคียงกัน (เช่น timestamp ที่ต่อเนื่องกัน)

### กำหนด Compression Policy ให้ทำงานอัตโนมัติ

```sql
-- บีบอัด chunk ที่มีข้อมูลเก่ากว่า 7 วัน โดยอัตโนมัติ
SELECT add_compression_policy('page_views', INTERVAL '7 days');
```

```
 add_compression_policy
-------------------------
                    2000
(1 row)
```

TimescaleDB จะมี background job คอยตรวจสอบและบีบอัด chunk ที่เข้าเงื่อนไข (อายุเกิน 7 วัน) ให้เองโดยอัตโนมัติตามตารางเวลา ไม่ต้องมีใครสั่งด้วยมือ

### บีบอัดด้วยตนเอง (Manual Compression)

ในกรณีที่ต้องการบีบอัดทันที ไม่รอ policy job:

```sql
-- บีบอัด chunk ทั้งหมดที่มีข้อมูลเก่ากว่า 7 วัน ด้วยตนเอง
SELECT compress_chunk(c)
FROM show_chunks('page_views', older_than => INTERVAL '7 days') AS c;
```

```
                compress_chunk
------------------------------------------
 _timescaledb_internal._hyper_1_1_chunk
 _timescaledb_internal._hyper_1_2_chunk
 _timescaledb_internal._hyper_1_3_chunk
 ...
(19 rows)
```

### ตรวจสอบผลลัพธ์ของการบีบอัด

```sql
SELECT
    pg_size_pretty(before_compression_total_bytes) AS before_size,
    pg_size_pretty(after_compression_total_bytes)  AS after_size,
    round(
        100 * (1 - after_compression_total_bytes::numeric
                    / nullif(before_compression_total_bytes, 0)), 1
    ) AS pct_saved
FROM hypertable_compression_stats('page_views');
```

```
 before_size | after_size | pct_saved
-------------+------------+-----------
 184 MB      | 21 MB      |      88.6
(1 row)
```

การบีบอัดได้ผลดีถึง ~88.6% ในตัวอย่างนี้ เพราะข้อมูลมีคอลัมน์ซ้ำ ๆ กันมาก (เช่น `device_type` มีแค่ 3 ค่า, `product_id` มีแค่ 200 ค่า) ซึ่งเหมาะกับการบีบอัดแบบ columnar เป็นอย่างมาก

### ข้อควรระวังเรื่อง Compressed Chunk

- **Chunk ที่ถูกบีบอัดแล้วจะไม่รองรับ `INSERT` ตรง ๆ อีกต่อไป** (ต้อง decompress ก่อน หรือใช้กลไก out-of-order insert ของ TimescaleDB เวอร์ชันใหม่ที่รองรับบางส่วน)
- การ `UPDATE`/`DELETE` บน compressed chunk ทำได้ แต่มี overhead มากกว่าปกติเพราะต้อง decompress ส่วนที่เกี่ยวข้องก่อน
- ควรบีบอัดเฉพาะ chunk ที่ "นิ่งแล้ว" คือไม่มีการเขียนเพิ่มอีกต่อไป (ข้อมูลเก่ากว่าหลายวันเป็นต้นไป)

```sql
-- ดู chunk ไหนถูกบีบอัดแล้วบ้าง
SELECT chunk_name, is_compressed
FROM timescaledb_information.chunks
WHERE hypertable_name = 'page_views'
ORDER BY range_start
LIMIT 5;
```

```
       chunk_name        | is_compressed
--------------------------+---------------
 _hyper_1_1_chunk        | t
 _hyper_1_2_chunk        | t
 _hyper_1_3_chunk        | t
 _hyper_1_4_chunk        | t
 _hyper_1_5_chunk        | f
(5 rows)
```

---

## Step 778: Data Retention Policy — ลบข้อมูลเก่าอัตโนมัติ

หลายองค์กรมีนโยบายชัดเจนว่าข้อมูล analytics แบบละเอียด (raw event) ไม่จำเป็นต้องเก็บไว้ตลอดไป เช่น "เก็บ raw page view event ไว้แค่ 90 วัน ส่วนข้อมูลสรุปรายวัน/รายเดือนเก็บไว้ถาวร" TimescaleDB มี **Data Retention Policy** ที่จะลบ chunk เก่าทั้งก้อนโดยอัตโนมัติเมื่อครบกำหนดเวลา

### เพิ่ม Retention Policy

```sql
-- ลบ chunk ที่มีข้อมูลเก่ากว่า 90 วัน โดยอัตโนมัติ
SELECT add_retention_policy('page_views', INTERVAL '90 days');
```

```
 add_retention_policy
-----------------------
                  3000
(1 row)
```

### ทำไมการลบทั้ง Chunk จึงเร็วกว่า DELETE ทีละแถวมาก

```sql
-- แบบที่ "ไม่ควรทำ" กับข้อมูลจำนวนมาก: DELETE ทีละแถวด้วยเงื่อนไขเวลา
-- DELETE FROM page_views WHERE view_time < now() - interval '90 days';
-- วิธีนี้ต้อง scan และลบทีละแถว (row-by-row), สร้าง dead tuple จำนวนมาก, ต้อง VACUUM ตามหลัง

-- สิ่งที่ retention policy ทำเบื้องหลังจริง ๆ คือเทียบเท่ากับ:
-- DROP TABLE _timescaledb_internal._hyper_1_1_chunk;
-- ซึ่งเป็นการลบทั้งตารางลูก (chunk) ทันที ไม่ต้อง scan ทีละแถว ไม่มี dead tuple เหลือ
```

การ `DROP` chunk ทั้งก้อนเป็นการดำเนินการระดับ metadata (คล้ายกับการ `DROP TABLE` ธรรมดา) จึงเร็วกว่าการ `DELETE` แบบ row-by-row มาก และไม่ทิ้ง dead tuple ไว้ให้ต้อง `VACUUM` เพิ่มเติม — นี่คือประโยชน์สำคัญของการที่ข้อมูลถูกจัดเก็บเป็น chunk รายช่วงเวลาอยู่แล้ว

### ตรวจสอบ Policy ที่ตั้งไว้ทั้งหมด

```sql
SELECT
    hypertable_name,
    j.proc_name  AS policy_type,
    j.schedule_interval,
    j.config
FROM timescaledb_information.jobs j
JOIN timescaledb_information.hypertables h
     ON h.hypertable_name = (j.hypertable_schema || '.' || j.hypertable_name)
       OR j.hypertable_name = h.hypertable_name
WHERE h.hypertable_name = 'page_views';
```

```
 hypertable_name |        policy_type         | schedule_interval |                  config
------------------+------------------------------+--------------------+-------------------------------------------
 page_views      | policy_compression          | 12:00:00           | {"hypertable_id": 1, "compress_after": "7 days"}
 page_views      | policy_retention            | 1 day              | {"hypertable_id": 1, "drop_after": "90 days"}
(2 rows)
```

### ลบ Retention Policy (หากต้องการยกเลิก)

```sql
SELECT remove_retention_policy('page_views');
```

```
 remove_retention_policy
--------------------------

(1 row)
```

### ลบ Chunk เก่าด้วยตนเอง (Manual)

```sql
-- ลบ chunk ที่เก่ากว่า 90 วัน ด้วยตนเองทันที โดยไม่ต้องรอ policy job
SELECT drop_chunks('page_views', older_than => INTERVAL '90 days');
```

```
              drop_chunks
------------------------------------------
 _timescaledb_internal._hyper_1_1_chunk
 _timescaledb_internal._hyper_1_2_chunk
(2 rows)
```

### สถาปัตยกรรมที่แนะนำ: Retention บน Raw Data + เก็บ Continuous Aggregate ถาวร

แนวทางที่ดีที่สุดสำหรับกรณีศึกษา e-commerce ของเรา คือรวม 3 เทคนิคจาก Step 776-778 เข้าด้วยกัน:

```sql
-- 1. Raw event (page_views) เก็บแค่ 90 วัน แล้วลบทิ้ง (ลด storage)
SELECT add_retention_policy('page_views', INTERVAL '90 days');

-- 2. ข้อมูลสรุปรายวัน (page_views_daily) ไม่ตั้ง retention policy
--    เพราะขนาดเล็กกว่ามาก และมีคุณค่าระยะยาวสำหรับดู trend ย้อนหลังหลายปี
--    (ไม่เรียก add_retention_policy กับ page_views_daily)

-- 3. ข้อมูลอายุ 7-90 วัน ถูกบีบอัดไว้ก่อนที่จะถูกลบ เพื่อประหยัดพื้นที่ระหว่างทาง
SELECT add_compression_policy('page_views', INTERVAL '7 days');
```

ด้วยแนวทางนี้ raw data จะไม่โตไม่หยุด แต่ข้อมูลสรุปเชิงธุรกิจ (แนวโน้มยอดวิวรายวัน/รายเดือน) ยังคงอยู่ถาวรสำหรับการวิเคราะห์ระยะยาว

---

## Step 779: time_bucket() — ฟังก์ชันหลักสำหรับ Downsampling

`time_bucket()` คือฟังก์ชันที่ TimescaleDB เพิ่มเข้ามา ทำหน้าที่คล้ายกับ `date_trunc()` ของ PostgreSQL มาตรฐาน แต่ **ยืดหยุ่นกว่ามาก** เพราะรองรับช่วงเวลาที่กำหนดเองได้อย่างอิสระ ไม่จำกัดแค่หน่วยปฏิทินมาตรฐาน (วินาที, นาที, ชั่วโมง, วัน ฯลฯ)

### เปรียบเทียบ `date_trunc()` กับ `time_bucket()`

```sql
-- date_trunc(): จำกัดเฉพาะหน่วยเวลามาตรฐานเท่านั้น (hour, day, week, month, ...)
SELECT date_trunc('hour', view_time) AS bucket, count(*)
FROM page_views
WHERE view_time >= now() - interval '1 day'
GROUP BY 1 ORDER BY 1;
```

```sql
-- time_bucket(): กำหนดขนาดช่วงเวลาได้อย่างอิสระ เช่น ทุก 15 นาที, ทุก 6 ชั่วโมง, ทุก 3 วัน
SELECT time_bucket('15 minutes', view_time) AS bucket, count(*)
FROM page_views
WHERE view_time >= now() - interval '1 day'
GROUP BY 1 ORDER BY 1;
```

```
         bucket          | count
--------------------------+-------
 2026-09-24 08:00:00+00  |   287
 2026-09-24 08:15:00+00  |   295
 2026-09-24 08:30:00+00  |   271
 2026-09-24 08:45:00+00  |   303
 ...
(96 rows)
```

`date_trunc('hour', ...)` **ไม่สามารถ** ทำ bucket ทุก 15 นาทีได้ (มีแค่ตัวเลือกมาตรฐานอย่าง minute, hour, day) แต่ `time_bucket('15 minutes', ...)` ทำได้ทันที — นี่คือความยืดหยุ่นที่สำคัญที่สุดของฟังก์ชันนี้

### ตัวอย่างการ Downsampling หลายระดับความละเอียด

```sql
-- ยอดวิวทุก 6 ชั่วโมง ของ 3 วันล่าสุด (เหมาะกับกราฟ dashboard ระยะสั้น)
SELECT
    time_bucket('6 hours', view_time) AS bucket,
    count(*)                          AS views,
    count(DISTINCT customer_id)       AS unique_visitors
FROM page_views
WHERE view_time >= now() - interval '3 days'
GROUP BY bucket
ORDER BY bucket;
```

```
         bucket          | views | unique_visitors
--------------------------+-------+------------------
 2026-09-22 06:00:00+00  |  6801 |             4312
 2026-09-22 12:00:00+00  |  6789 |             4298
 2026-09-22 18:00:00+00  |  6754 |             4276
 2026-09-23 00:00:00+00  |  6812 |             4301
 ...
(12 rows)
```

```sql
-- ยอดวิวทุก 1 สัปดาห์ ของ 6 เดือนล่าสุด (เหมาะกับกราฟภาพรวมระยะยาว)
SELECT
    time_bucket('7 days', view_time) AS week_bucket,
    count(*)                          AS views
FROM page_views
GROUP BY week_bucket
ORDER BY week_bucket;
```

```
       week_bucket        | views
--------------------------+--------
 2026-03-26 00:00:00+00  |  77482
 2026-04-02 00:00:00+00  |  77398
 2026-04-09 00:00:00+00  |  77621
 ...
 2026-09-24 00:00:00+00  |  38214
(26 rows)
```

### time_bucket() กับ Offset (การเลื่อนจุดเริ่ม bucket)

บางครั้งต้องการให้ bucket รายวันเริ่มไม่ตรงเที่ยงคืน UTC เช่นต้องการให้ "วันธุรกิจ" เริ่มที่ 08:00 น. (เวลาไทย = 01:00 UTC):

```sql
SELECT
    time_bucket('1 day', view_time, INTERVAL '1 hour') AS business_day_bucket,
    count(*) AS views
FROM page_views
WHERE view_time >= now() - interval '3 days'
GROUP BY 1
ORDER BY 1;
```

> พารามิเตอร์ตัวที่สามคือค่า offset ที่ใช้เลื่อนจุดเริ่มต้นของแต่ละ bucket จากค่า origin เริ่มต้น (ค่าเริ่มต้นคือ 2000-01-03 ซึ่งเป็นวันจันทร์ เพื่อให้ bucket รายสัปดาห์เริ่มวันจันทร์เสมอ)

### time_bucket_gapfill() — เติมช่วงเวลาที่ไม่มีข้อมูล

ปัญหาที่พบบ่อยของการทำ time-series aggregation คือ ถ้าไม่มี event เกิดขึ้นในบาง bucket (เช่น สินค้าบางตัวไม่มีคนดูในบางชั่วโมง) ผลลัพธ์จะ "ขาดแถวไปเฉย ๆ" ทำให้กราฟเส้นขาดช่วง `time_bucket_gapfill()` ช่วยเติมช่วงที่ขาดหายให้ครบ:

```sql
SELECT
    time_bucket_gapfill('1 hour', view_time) AS bucket,
    product_id,
    count(*) AS views
FROM page_views
WHERE view_time >= now() - interval '1 day'
  AND product_id = 199   -- สมมติสินค้านี้มีคนดูน้อยมาก
GROUP BY bucket, product_id
ORDER BY bucket;
```

```
         bucket          | product_id | views
--------------------------+------------+-------
 2026-09-24 08:00:00+00  |        199 |     2
 2026-09-24 09:00:00+00  |        199 |
 2026-09-24 10:00:00+00  |        199 |     1
 2026-09-24 11:00:00+00  |        199 |
 ...
(24 rows)
```

> `time_bucket_gapfill()` ต้องใช้คู่กับ `WHERE` ที่ระบุช่วงเวลาต้น-ปลายชัดเจน (start/end ต้องเป็นค่าคงที่หรือคำนวณได้ล่วงหน้า) และมักใช้ร่วมกับ `locf()` (last observation carried forward) หรือ `interpolate()` เพื่อเติมค่าที่ขาดหายด้วยค่าที่สมเหตุสมผลแทน `NULL`

---

## Step 780: แบบฝึกหัดรวม — ระบบวิเคราะห์ Page View แบบเต็มรูปแบบ

ในหัวข้อนี้เราจะประกอบทุกเทคนิคที่เรียนมาใน Step 771-779 เข้าด้วยกัน เพื่อสร้าง **ระบบวิเคราะห์ page view ของเว็บ e-commerce แบบครบวงจร** ตั้งแต่การเก็บ raw event ไปจนถึง dashboard สรุปผล

### ภาพรวมสถาปัตยกรรมที่จะสร้าง

```
Raw events (page_views hypertable)
    │
    ├── chunk_time_interval = 1 day
    ├── compression policy: บีบอัดหลังจากอายุ 7 วัน
    └── retention policy: ลบทิ้งหลังจากอายุ 90 วัน
    │
    ▼
Continuous Aggregate ระดับที่ 1: page_views_hourly (ราย product/device/ชั่วโมง)
    │
    ▼
Continuous Aggregate ระดับที่ 2: page_views_daily (ภาพรวมรายวันทั้งร้าน — เก็บถาวร ไม่มี retention)
```

### ขั้นตอนที่ 1: ตั้งค่า Hypertable ให้เหมาะกับ Workload

```sql
-- (สมมติว่ายังไม่เคยสร้าง hypertable มาก่อน)
CREATE TABLE page_views (
    view_time            TIMESTAMPTZ NOT NULL,
    product_id           INTEGER,
    customer_id          INTEGER,
    device_type          VARCHAR(20),
    session_duration_sec INTEGER
);

SELECT create_hypertable(
    'page_views',
    'view_time',
    chunk_time_interval => INTERVAL '1 day'
);

CREATE INDEX idx_page_views_product_time
    ON page_views (product_id, view_time DESC);
```

### ขั้นตอนที่ 2: สร้าง Continuous Aggregate สองระดับ

```sql
-- ระดับรายชั่วโมง แยกตามสินค้าและอุปกรณ์ (สำหรับ dashboard เจ้าของสินค้า)
CREATE MATERIALIZED VIEW page_views_hourly
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', view_time) AS bucket,
    product_id,
    device_type,
    count(*)                          AS total_views,
    count(DISTINCT customer_id)       AS unique_customers,
    avg(session_duration_sec)         AS avg_session_sec
FROM page_views
GROUP BY bucket, product_id, device_type
WITH NO DATA;

SELECT add_continuous_aggregate_policy('page_views_hourly',
    start_offset      => INTERVAL '3 days',
    end_offset        => INTERVAL '1 hour',
    schedule_interval  => INTERVAL '1 hour'
);

-- ระดับรายวัน ภาพรวมทั้งร้าน (สำหรับผู้บริหาร ดู trend ระยะยาว)
CREATE MATERIALIZED VIEW page_views_daily
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 day', view_time)  AS bucket,
    count(*)                          AS total_views,
    count(DISTINCT product_id)        AS distinct_products_viewed,
    count(DISTINCT customer_id)       AS unique_customers
FROM page_views
GROUP BY bucket
WITH NO DATA;

SELECT add_continuous_aggregate_policy('page_views_daily',
    start_offset      => INTERVAL '30 days',
    end_offset        => INTERVAL '1 day',
    schedule_interval  => INTERVAL '6 hours'
);
```

> `WITH NO DATA` หมายถึงสร้าง view ไว้ก่อนแบบยังไม่คำนวณข้อมูล เหมาะเมื่อมีข้อมูลย้อนหลังจำนวนมากอยู่แล้ว และต้องการควบคุมการคำนวณครั้งแรกด้วยตนเอง (ผ่าน `refresh_continuous_aggregate`) แทนที่จะให้คำนวณข้อมูลย้อนหลังทั้งหมดทันทีตอนสร้าง

```sql
-- คำนวณข้อมูลย้อนหลังครั้งแรกด้วยตนเอง (initial backfill)
CALL refresh_continuous_aggregate('page_views_hourly', NULL, now());
CALL refresh_continuous_aggregate('page_views_daily', NULL, now());
```

### ขั้นตอนที่ 3: ตั้งค่า Compression และ Retention

```sql
-- เปิดใช้งาน compression บน raw table
ALTER TABLE page_views SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'product_id',
    timescaledb.compress_orderby   = 'view_time DESC'
);

SELECT add_compression_policy('page_views', INTERVAL '7 days');

-- raw event เก็บแค่ 90 วัน (ข้อมูลสรุปยังอยู่ถาวรใน continuous aggregate)
SELECT add_retention_policy('page_views', INTERVAL '90 days');
```

### ขั้นตอนที่ 4: Query สำหรับ Dashboard จริง

**4.1 สินค้ายอดนิยม 5 อันดับใน 24 ชั่วโมงล่าสุด (ใช้ continuous aggregate เพื่อความเร็ว)**

```sql
SELECT
    product_id,
    sum(total_views)        AS views_24h,
    sum(unique_customers)   AS approx_unique_customers
FROM page_views_hourly
WHERE bucket >= now() - interval '24 hours'
GROUP BY product_id
ORDER BY views_24h DESC
LIMIT 5;
```

```
 product_id | views_24h | approx_unique_customers
------------+-----------+---------------------------
        142 |       389 |                       301
         77 |       382 |                       295
        203 |       378 |                       288
         12 |       371 |                       284
         98 |       365 |                       279
(5 rows)
```

**4.2 สัดส่วนอุปกรณ์ที่ใช้เข้าชม (mobile vs desktop vs tablet) รายสัปดาห์**

```sql
SELECT
    time_bucket('7 days', bucket) AS week,
    device_type,
    sum(total_views)              AS views
FROM page_views_hourly
WHERE bucket >= now() - interval '28 days'
GROUP BY week, device_type
ORDER BY week, views DESC;
```

```
          week           | device_type | views
--------------------------+-------------+--------
 2026-08-28 00:00:00+00  | mobile      |  38210
 2026-08-28 00:00:00+00  | desktop     |  25877
 2026-08-28 00:00:00+00  | tablet      |  12987
 2026-09-04 00:00:00+00  | mobile      |  38401
 ...
(12 rows)
```

**4.3 Trend ยอดวิวรายวันย้อนหลังทั้งหมด (จาก continuous aggregate ที่เก็บถาวร)**

```sql
SELECT bucket, total_views, distinct_products_viewed, unique_customers
FROM page_views_daily
ORDER BY bucket DESC
LIMIT 10;
```

```
         bucket           | total_views | distinct_products_viewed | unique_customers
---------------------------+-------------+----------------------------+-------------------
 2026-09-24 00:00:00+00   |       11132 |                        200 |              4821
 2026-09-23 00:00:00+00   |       11098 |                        200 |              4798
 2026-09-22 00:00:00+00   |       11145 |                        200 |              4812
 ...
(10 rows)
```

**4.4 ตรวจสอบสถานะและขนาดพื้นที่ของทั้งระบบ**

```sql
SELECT
    hypertable_name,
    pg_size_pretty(hypertable_size(format('%I.%I', hypertable_schema, hypertable_name)::regclass)) AS raw_size,
    compression_enabled
FROM timescaledb_information.hypertables
WHERE hypertable_name IN ('page_views');
```

```
 hypertable_name | raw_size | compression_enabled
------------------+----------+-----------------------
 page_views      | 21 MB    | t
(1 row)
```

ระบบที่สร้างเสร็จนี้จะสามารถ:

- รับ page view event เข้ามาต่อเนื่องด้วย insert ความเร็วสูง โดย index ไม่บวมเพราะ chunk ใหม่มีขนาดเล็กเสมอ
- ให้ dashboard เจ้าของร้านดึงข้อมูลสรุปรายชั่วโมง/รายวันได้เร็วมาก เพราะอ่านจาก continuous aggregate ที่คำนวณไว้ล่วงหน้าแล้ว ไม่ต้อง scan raw data ทุกครั้ง
- ประหยัดพื้นที่ดิสก์อัตโนมัติด้วย compression สำหรับข้อมูลอายุเกิน 7 วัน
- ควบคุมขนาดฐานข้อมูลระยะยาวด้วย retention policy ที่ลบ raw data อายุเกิน 90 วันทิ้งอัตโนมัติ โดยไม่กระทบข้อมูลสรุปเชิงธุรกิจที่ต้องเก็บถาวร

---

## สรุปท้ายบท

ในบทนี้เราได้เรียนรู้ **TimescaleDB** ซึ่งเป็น extension เฉพาะทางของ PostgreSQL สำหรับรับมือกับข้อมูล time-series ขนาดใหญ่ ประเด็นสำคัญที่ต้องจำ:

1. **TimescaleDB ต้องติดตั้งแยกต่างหาก** ไม่ได้มากับ PostgreSQL core ต้องรัน `CREATE EXTENSION timescaledb;` หลังจากติดตั้ง package และตั้งค่า `shared_preload_libraries` ตามเอกสารทางการแล้วเท่านั้น
2. **Time-series data** มีลักษณะ write-heavy, append-only เป็นหลัก และมักถูก query ด้วยเงื่อนไขช่วงเวลาและ aggregation
3. PostgreSQL ธรรมดามีข้อจำกัดเรื่อง **index bloat** และการดูแล partition ด้วยตนเองเมื่อข้อมูลโตมาก
4. **Hypertable** คือตารางเสมือนที่ TimescaleDB แบ่งเป็น **chunk** ให้อัตโนมัติตามเวลา ซึ่งใช้กลไก partitioning เดียวกับที่เรียนใน Part 054 แต่บริหารจัดการให้เองทั้งหมด
5. **Continuous Aggregate** คือ materialized view พิเศษที่ update แบบ incremental ประหยัด CPU/I/O กว่าการ refresh ใหม่ทั้งหมด
6. **Compression** บีบอัดข้อมูลเก่าแบบ columnar อัตโนมัติ ลดพื้นที่ได้มากในหลายกรณี
7. **Retention Policy** ลบ chunk เก่าทั้งก้อนโดยอัตโนมัติ เร็วกว่า `DELETE` แบบ row-by-row มาก
8. **`time_bucket()`** ยืดหยุ่นกว่า `date_trunc()` เพราะกำหนดขนาดช่วงเวลาได้อย่างอิสระ เหมาะกับการ downsampling ทุกระดับความละเอียด

TimescaleDB เหมาะกับสถานการณ์ที่มีข้อมูล time-series ปริมาณมหาศาลจริง ๆ (เช่น IoT, monitoring, analytics ขนาดใหญ่) หากปริมาณข้อมูลของระบบยังไม่ถึงระดับที่ PostgreSQL มาตรฐาน + partitioning (Part 054) จัดการไม่ไหว การเพิ่มความซับซ้อนของ extension นี้อาจยังไม่คุ้มค่า ผู้พัฒนาที่ดีควรประเมินปริมาณและรูปแบบข้อมูลจริงก่อนตัดสินใจนำ TimescaleDB มาใช้งาน

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1

คำสั่งใดใช้เปิดใช้งาน TimescaleDB extension ในฐานข้อมูลปัจจุบัน?

<details>
<summary>เฉลย</summary>

```sql
CREATE EXTENSION IF NOT EXISTS timescaledb;
```

ต้องรันโดย role ที่มีสิทธิ์เพียงพอ (มักต้องเป็น superuser หรือ role ที่ได้รับสิทธิ์เฉพาะ) และต้องติดตั้ง TimescaleDB package ไว้ในเครื่อง server ล่วงหน้าแล้วเท่านั้น มิฉะนั้นจะได้ error ว่าไม่พบ extension control file

</details>

---

### แบบฝึกหัดที่ 2

จงอธิบายว่าทำไมข้อมูล time-series ถึงมีแนวโน้มทำให้เกิดปัญหา index bloat มากกว่าข้อมูลเชิงธุรกรรมทั่วไป

<details>
<summary>เฉลย</summary>

เพราะข้อมูล time-series มีลักษณะ **write-heavy แบบ append ต่อเนื่อง** ปริมาณมหาศาลตลอดเวลา เมื่อ index (เช่นบนคอลัมน์เวลา) มีขนาดใหญ่เกินกว่าจะ cache ไว้ใน memory ได้ทั้งหมด การ insert แต่ละครั้งอาจต้องเข้าถึง index page ที่ไม่ได้อยู่ใน memory (random I/O) ทำให้การเขียนช้าลง และ index ก็ยิ่งโตขึ้นเรื่อย ๆ ตามข้อมูลที่เพิ่ม นอกจากนี้การ `VACUUM` ตารางขนาดมหาศาลก็ใช้เวลานานและกิน I/O สูงตามไปด้วย

</details>

---

### แบบฝึกหัดที่ 3

เขียนคำสั่งแปลงตาราง `sensor_readings` (คอลัมน์เวลาชื่อ `reading_time`) ที่มีข้อมูลอยู่แล้วให้เป็น hypertable โดยกำหนด `chunk_time_interval` เป็น 12 ชั่วโมง

<details>
<summary>เฉลย</summary>

```sql
SELECT create_hypertable(
    'sensor_readings',
    'reading_time',
    chunk_time_interval => INTERVAL '12 hours',
    migrate_data         => TRUE
);
```

ต้องใส่ `migrate_data => TRUE` เพราะตารางมีข้อมูลอยู่แล้ว มิฉะนั้นจะได้ error แจ้งว่าตารางไม่ว่าง

</details>

---

### แบบฝึกหัดที่ 4

Chunk ของ TimescaleDB สัมพันธ์กับแนวคิดใดที่เคยเรียนใน Part 054 และต่างกันอย่างไรในทางปฏิบัติ

<details>
<summary>เฉลย</summary>

Chunk สัมพันธ์กับแนวคิด **partition (child table)** ใน declarative partitioning ของ PostgreSQL ที่เรียนใน Part 054 โดยตรง เพราะ chunk แต่ละตัวคือตารางลูกจริงที่ใช้กลไก partitioning เดียวกัน ความต่างในทางปฏิบัติคือ:

- Manual partitioning (Part 054) ผู้ใช้ต้องสร้างแต่ละ partition และกำหนดขอบเขต `FOR VALUES FROM ... TO ...` ด้วยตนเอง รวมถึงต้องเขียน job มาสร้าง partition ใหม่ล่วงหน้าเอง
- TimescaleDB สร้าง chunk ใหม่ให้อัตโนมัติทันทีที่มีข้อมูล insert เข้ามาในช่วงเวลาที่ยังไม่มี chunk รองรับ โดยผู้ใช้ไม่ต้องดูแลเอง

</details>

---

### แบบฝึกหัดที่ 5

จงเขียนคำสั่งสร้าง Continuous Aggregate ชื่อ `page_views_15min` ที่สรุปยอดวิวทุก 15 นาที แยกตาม `device_type` จากตาราง `page_views`

<details>
<summary>เฉลย</summary>

```sql
CREATE MATERIALIZED VIEW page_views_15min
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('15 minutes', view_time) AS bucket,
    device_type,
    count(*) AS total_views
FROM page_views
GROUP BY bucket, device_type;
```

</details>

---

### แบบฝึกหัดที่ 6

Continuous Aggregate ต่างจาก Materialized View ธรรมดาของ PostgreSQL อย่างไร ในแง่ของการ refresh?

<details>
<summary>เฉลย</summary>

Materialized view ธรรมดาต้องใช้คำสั่ง `REFRESH MATERIALIZED VIEW` ซึ่งจะคำนวณผลลัพธ์ **ใหม่ทั้งหมด** ทุกครั้งที่ refresh (เว้นแต่จะเขียน logic เพิ่มเติมเอง) ส่วน Continuous Aggregate ของ TimescaleDB ทำการ refresh แบบ **incremental** คือคำนวณใหม่เฉพาะช่วงเวลาที่มีข้อมูลเปลี่ยนแปลงเท่านั้น ทำให้ประหยัด CPU/I/O มากกว่ามากเมื่อข้อมูลมีขนาดใหญ่ และยังสามารถตั้ง policy ให้ refresh อัตโนมัติตามตารางเวลาได้ในตัว ผ่าน `add_continuous_aggregate_policy()`

</details>

---

### แบบฝึกหัดที่ 7

เขียนคำสั่งเปิดใช้งาน compression บนตาราง `page_views` โดยแบ่งกลุ่มข้อมูลตาม `device_type` และเรียงลำดับภายในกลุ่มตาม `view_time` จากใหม่ไปเก่า จากนั้นตั้ง policy ให้บีบอัดข้อมูลที่อายุเกิน 14 วันโดยอัตโนมัติ

<details>
<summary>เฉลย</summary>

```sql
ALTER TABLE page_views SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'device_type',
    timescaledb.compress_orderby   = 'view_time DESC'
);

SELECT add_compression_policy('page_views', INTERVAL '14 days');
```

</details>

---

### แบบฝึกหัดที่ 8

เพราะเหตุใดการลบข้อมูลเก่าด้วย Retention Policy ของ TimescaleDB จึงมีประสิทธิภาพดีกว่าการรัน `DELETE FROM page_views WHERE view_time < ...` โดยตรง?

<details>
<summary>เฉลย</summary>

เพราะ Retention Policy ทำงานโดยการ **ลบทั้ง chunk (DROP TABLE ของตารางลูก)** ซึ่งเป็นการดำเนินการระดับ metadata ที่รวดเร็ว ไม่ต้อง scan หรือลบทีละแถว และไม่ทิ้ง dead tuple ไว้ให้ต้อง `VACUUM` ตามหลัง ในขณะที่ `DELETE ... WHERE` แบบธรรมดาต้อง scan หาแถวที่ตรงเงื่อนไขและลบทีละแถว (row-by-row) ซึ่งช้ากว่ามากเมื่อข้อมูลมีจำนวนมาก และยังสร้าง dead tuple จำนวนมากที่ต้องรอ `VACUUM` มาจัดการภายหลัง

</details>

---

### แบบฝึกหัดที่ 9

จงอธิบายความต่างระหว่าง `date_trunc('hour', view_time)` กับ `time_bucket('90 minutes', view_time)` และบอกว่าทำไม `date_trunc()` เพียงอย่างเดียวไม่สามารถทำ bucket แบบ 90 นาทีได้

<details>
<summary>เฉลย</summary>

`date_trunc()` รองรับเฉพาะหน่วยเวลาตามปฏิทินมาตรฐานเท่านั้น (เช่น second, minute, hour, day, week, month, year) ไม่สามารถกำหนดขนาดช่วงเวลาแบบกำหนดเองได้ ในขณะที่ `time_bucket()` ของ TimescaleDB รับค่า interval แบบใดก็ได้ตามที่ต้องการ เช่น `'90 minutes'`, `'15 minutes'`, `'6 hours'`, `'3 days'` เป็นต้น ทำให้ยืดหยุ่นกว่ามากในการทำ downsampling ตามความละเอียดที่ธุรกิจต้องการจริง ๆ ซึ่งไม่จำเป็นต้องตรงกับหน่วยเวลาปฏิทินมาตรฐานเสมอไป

</details>

---

### แบบฝึกหัดที่ 10

จากสถาปัตยกรรมในหัวข้อ Step 780 (แบบฝึกหัดรวม) หากต้องการเพิ่มระบบให้เก็บ **สรุปยอดวิวรายเดือน** ไว้ถาวรเพิ่มอีกหนึ่งระดับ (สำหรับทำรายงานประจำปี) จะต้องเขียนโค้ดอย่างไร โดยให้สร้างต่อยอดจาก `page_views_daily` continuous aggregate ที่มีอยู่แล้ว

<details>
<summary>เฉลย</summary>

TimescaleDB รองรับการสร้าง continuous aggregate ซ้อนบน continuous aggregate อีกชั้นได้ (hierarchical continuous aggregate) ตั้งแต่เวอร์ชัน 2.9 เป็นต้นไป:

```sql
CREATE MATERIALIZED VIEW page_views_monthly
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 month', bucket)     AS month_bucket,
    sum(total_views)                    AS total_views,
    max(distinct_products_viewed)       AS distinct_products_viewed,
    sum(unique_customers)               AS unique_customers_approx
FROM page_views_daily
GROUP BY month_bucket
WITH NO DATA;

SELECT add_continuous_aggregate_policy('page_views_monthly',
    start_offset      => INTERVAL '3 months',
    end_offset        => INTERVAL '1 day',
    schedule_interval  => INTERVAL '1 day'
);

CALL refresh_continuous_aggregate('page_views_monthly', NULL, now());
```

โดยไม่ตั้ง retention policy ให้กับ `page_views_monthly` เพื่อให้ข้อมูลสรุปรายเดือนนี้ถูกเก็บไว้ถาวรสำหรับทำรายงานประจำปีในอนาคต

</details>

---

## บทถัดไป

เมื่อเข้าใจ TimescaleDB สำหรับงาน time-series โดยเฉพาะแล้ว บทถัดไปจะพาไปเรียนรู้เรื่อง **กลยุทธ์การ migration** ของฐานข้อมูล PostgreSQL ในภาพรวม ตั้งแต่การย้ายเวอร์ชัน ไปจนถึงการย้ายจากฐานข้อมูลอื่นมาสู่ PostgreSQL

**บทถัดไป:** [Part 079: Migration Strategies](./part-079-migration-strategies.md)
