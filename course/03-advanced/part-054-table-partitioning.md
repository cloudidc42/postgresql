# การแบ่งพาร์ทิชันตาราง (Table Partitioning): Range, List, Hash Partitioning

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับสูง | Part 054

เมื่อระบบข้อมูลของเราเติบโตจนตารางเดียวมีหลายสิบล้านหรือหลายร้อยล้านแถว การ query, index, backup และ maintenance บนตารางก้อนเดียวจะเริ่มช้าลงเรื่อย ๆ จนถึงจุดที่จัดการไม่ไหว **Table Partitioning** คือเทคนิคสำคัญที่ PostgreSQL มีมาให้ตั้งแต่เวอร์ชัน 10 (แบบ declarative) เพื่อแก้ปัญหานี้ โดยแบ่งตารางใหญ่หนึ่งตารางออกเป็นตารางย่อยทางกายภาพหลายตาราง แต่ยังคง query เหมือนเป็นตารางเดียวจากมุมมองของแอปพลิเคชัน

บทนี้จะพาไปสร้างตาราง `orders_partitioned` ขนาดใหญ่ระดับหลายพันแถว กระจายอยู่หลายปี แล้วพิสูจน์ให้เห็นด้วยตาเปล่าผ่าน `EXPLAIN` ว่า partition pruning ช่วยลดงานของ query planner ได้จริงแค่ไหน

## เป้าหมายการเรียนรู้

หลังจบบทนี้ ผู้เรียนจะสามารถ:

1. อธิบายได้ว่า Table Partitioning คืออะไร แก้ปัญหาอะไร และเมื่อไหร่ควร/ไม่ควรใช้
2. ใช้ Declarative Partitioning ของ PostgreSQL 10+ สร้างตารางแบบ `PARTITION BY RANGE`, `LIST`, และ `HASH`
3. ออกแบบ Range Partitioning ตามวันที่ (รายเดือน/รายปี) พร้อมเขียน syntax `PARTITION OF ... FOR VALUES FROM ... TO ...` ได้อย่างถูกต้อง
4. ใช้ List Partitioning แบ่งข้อมูลตามค่าที่ไม่ต่อเนื่อง เช่น ประเทศ/ภูมิภาค
5. ใช้ Hash Partitioning กระจายข้อมูลอย่างเท่าเทียมเมื่อไม่มี partition key ตามธรรมชาติ
6. อ่านผลลัพธ์ `EXPLAIN` เพื่อยืนยันว่าเกิด Partition Pruning จริง ทั้งแบบ static (plan-time) และ dynamic (runtime)
7. ใช้ Default Partition อย่างถูกต้องและเข้าใจข้อจำกัดของมัน
8. บริหารจัดการวงจรชีวิตของ partition: เพิ่มล่วงหน้า, DETACH/ATTACH, DROP ข้อมูลเก่าแทนการ DELETE
9. สร้าง index บนตาราง partitioned อย่างถูกวิธี และเข้าใจข้อจำกัดเรื่อง local index vs global index
10. ออกแบบกลยุทธ์ partitioning + archive สำหรับข้อมูลธุรกิจจริงที่สะสมมาหลายปี

---

## เตรียมข้อมูล

เราจะจำลองบริบทอีคอมเมิร์ซแบบย่อ ๆ (ลูกค้า/สินค้า) แล้วโฟกัสหลักไปที่ตาราง `orders_partitioned` ซึ่งเป็น fact table ขนาดใหญ่ที่ถูกออกแบบมาเพื่อสาธิต partitioning โดยเฉพาะ

> หมายเหตุ: ถ้าคุณมีตาราง `orders`, `customers`, `products` จากบทก่อนหน้าอยู่แล้ว ตารางในบทนี้ตั้งชื่อแยกต่างหาก (`orders_partitioned` ไม่ใช่ `orders`) จึงสามารถรันคู่ขนานกันได้โดยไม่ชนกัน

### 1) ตารางพื้นฐาน: customers และ products

```sql
DROP TABLE IF EXISTS orders_hash_demo CASCADE;
DROP TABLE IF EXISTS orders_by_region CASCADE;
DROP TABLE IF EXISTS orders_partitioned CASCADE;
DROP TABLE IF EXISTS products CASCADE;
DROP TABLE IF EXISTS customers CASCADE;

CREATE TABLE customers (
    customer_id SERIAL PRIMARY KEY,
    first_name  VARCHAR(60),
    last_name   VARCHAR(60),
    country     VARCHAR(60)
);

CREATE TABLE products (
    product_id   SERIAL PRIMARY KEY,
    product_name VARCHAR(150),
    unit_price   NUMERIC(10,2)
);
```

เติมข้อมูลลูกค้า 500 คน กระจายในหลายประเทศแถบเอเชีย และสินค้า 200 รายการ (ตารางนี้ไว้ใช้เป็นบริบทประกอบ และจะถูกใช้ซ้ำในบทถัดไปเรื่อง Table Inheritance):

```sql
INSERT INTO customers (first_name, last_name, country)
SELECT
    'Customer' || i,
    'Last' || i,
    (ARRAY['Thailand','Singapore','Malaysia','Vietnam',
           'Indonesia','Philippines','Japan','South Korea'])[floor(random() * 8 + 1)]
FROM generate_series(1, 500) AS i;

INSERT INTO products (product_name, unit_price)
SELECT
    'Product ' || i,
    round((random() * 2000 + 50)::numeric, 2)
FROM generate_series(1, 200) AS i;
```

### 2) ตารางหลักของบท: orders_partitioned (RANGE PARTITION ตาม order_date)

นี่คือตารางที่เราจะใช้สาธิตแนวคิดหลักของบทนี้ทั้งหมด:

```sql
CREATE TABLE orders_partitioned (
    order_id     BIGSERIAL,
    customer_id  INTEGER REFERENCES customers(customer_id),
    order_date   DATE NOT NULL,
    status       VARCHAR(20) NOT NULL DEFAULT 'pending',
    total_amount NUMERIC(12,2) NOT NULL,
    PRIMARY KEY (order_id, order_date)
) PARTITION BY RANGE (order_date);
```

สังเกตสองจุดสำคัญที่จะอธิบายละเอียดใน Step 532-533 และ 539:

- `PARTITION BY RANGE (order_date)` บอก PostgreSQL ว่าตารางนี้เป็น "ตารางแม่" (partitioned table) ที่จะถูกแบ่งตามช่วงของ `order_date`
- `PRIMARY KEY (order_id, order_date)` ต้องรวมคอลัมน์ที่ใช้แบ่ง partition (`order_date`) เข้าไปด้วยเสมอ เพราะ PostgreSQL ไม่รองรับ unique index ข้าม partition (global index) — จะอธิบายเหตุผลเต็ม ๆ ใน Step 539

### 3) สร้างพาร์ทิชันรายเดือน 3 ปี (2023-2025) ด้วยสคริปต์อัตโนมัติ

การเขียน `CREATE TABLE ... PARTITION OF` ทีละ 36 คำสั่งด้วยมือไม่สนุกและเสี่ยงพิมพ์ผิด ในงานจริงเราจะเขียนสคริปต์ (หรือใช้ extension อย่าง `pg_partman`) เพื่อสร้างพาร์ทิชันแบบวนลูป:

```sql
DO $$
DECLARE
    start_date date := '2023-01-01';
    end_date   date := '2026-01-01';   -- exclusive upper bound
    part_date  date := start_date;
    part_name  text;
BEGIN
    WHILE part_date < end_date LOOP
        part_name := 'orders_' || to_char(part_date, 'YYYY_MM');
        EXECUTE format(
            'CREATE TABLE IF NOT EXISTS %I PARTITION OF orders_partitioned
                FOR VALUES FROM (%L) TO (%L)',
            part_name, part_date, part_date + INTERVAL '1 month'
        );
        part_date := part_date + INTERVAL '1 month';
    END LOOP;
END $$;
```

คำสั่งนี้จะสร้างพาร์ทิชันรายเดือนทั้งหมด **36 ตาราง** ตั้งแต่ `orders_2023_01` ถึง `orders_2025_12` โดยอัตโนมัติ (เทียบเท่ากับการเขียนคำสั่งแบบนี้ทีละบรรทัด):

```sql
-- ตัวอย่างสิ่งที่ลูปด้านบนสร้างให้ (ไม่ต้องรันซ้ำ แสดงเพื่อให้เห็นรูปแบบ)
-- CREATE TABLE orders_2024_01 PARTITION OF orders_partitioned
--     FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');
-- CREATE TABLE orders_2024_02 PARTITION OF orders_partitioned
--     FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');
-- CREATE TABLE orders_2024_03 PARTITION OF orders_partitioned
--     FOR VALUES FROM ('2024-03-01') TO ('2024-04-01');
```

และเผื่อกรณีมีข้อมูลหลุดช่วง (เช่น order_date ผิดพลาดเป็นปี 2030) เราจะสร้าง **default partition** ไว้ด้วย (รายละเอียดเต็มใน Step 537):

```sql
CREATE TABLE orders_default PARTITION OF orders_partitioned DEFAULT;
```

### 4) เติมข้อมูลจำนวนมาก (~8,000 แถว กระจาย 3 ปี)

```sql
INSERT INTO orders_partitioned (customer_id, order_date, status, total_amount)
SELECT
    (random() * 499 + 1)::int,
    (DATE '2023-01-01'
        + (random() * (DATE '2025-12-31' - DATE '2023-01-01'))::int)::date,
    (ARRAY['pending','paid','shipped','completed','cancelled'])[floor(random() * 5 + 1)],
    round((random() * 4990 + 10)::numeric, 2)
FROM generate_series(1, 8000);

ANALYZE orders_partitioned;
```

ตรวจสอบว่าข้อมูลกระจายลงพาร์ทิชันจริง (แต่ละพาร์ทิชันจะมีตัวเลขต่างกันเล็กน้อยเพราะสุ่ม แต่ควรใกล้เคียงกันทุกเดือน ราว 200-250 แถว/เดือน):

```sql
SELECT tableoid::regclass AS partition_name, count(*) AS row_count
FROM orders_partitioned
GROUP BY tableoid
ORDER BY partition_name
LIMIT 10;
```

```text
 partition_name | row_count
----------------+-----------
 orders_2023_01 |       223
 orders_2023_02 |       198
 orders_2023_03 |       231
 orders_2023_04 |       215
 orders_2023_05 |       209
 orders_2023_06 |       227
 orders_2023_07 |       218
 orders_2023_08 |       201
 orders_2023_09 |       234
 orders_2023_10 |       211
(10 rows)
```

ดูภาพรวมโครงสร้างพาร์ทิชันทั้งหมดด้วยฟังก์ชัน `pg_partition_tree()` (ใช้ได้ตั้งแต่ PostgreSQL 12):

```sql
SELECT relid::regclass AS name, parentrelid::regclass AS parent, isleaf, level
FROM pg_partition_tree('orders_partitioned')
ORDER BY level, name
LIMIT 8;
```

```text
      name       |       parent       | isleaf | level
------------------+---------------------+--------+-------
 orders_partitioned |                   | f      |     0
 orders_2023_01   | orders_partitioned | t      |     1
 orders_2023_02   | orders_partitioned | t      |     1
 orders_2023_03   | orders_partitioned | t      |     1
 orders_2023_04   | orders_partitioned | t      |     1
 orders_2023_05   | orders_partitioned | t      |     1
 orders_2023_06   | orders_partitioned | t      |     1
 orders_default   | orders_partitioned | t      |     1
(8 rows)
```

ข้อมูลชุดนี้ (~8,000 แถว, 36 พาร์ทิชัน + default) จะถูกใช้ตลอดทั้งบท มาเริ่มทำความเข้าใจแนวคิดกันทีละ Step

---

## Step 531: Partitioning คืออะไร

**Table Partitioning** คือการแบ่งตารางเชิงตรรกะหนึ่งตาราง (ที่แอปพลิเคชันมองเห็นเป็นตารางเดียว) ออกเป็นตารางย่อยทางกายภาพหลายตาราง (partitions) โดยแต่ละ partition เก็บข้อมูลเฉพาะช่วงหรือกลุ่มค่าหนึ่ง ๆ ของ "partition key" ที่เราเลือก

ในตัวอย่างข้างต้น `orders_partitioned` เป็น**ตารางแม่ (parent / partitioned table)** ที่ไม่ได้เก็บข้อมูลจริงเลยสักแถวเดียว! ข้อมูลจริงทั้งหมดถูกเก็บกระจายอยู่ในตารางลูก 36+1 ตาราง (`orders_2023_01`, `orders_2023_02`, ..., `orders_default`) แต่เวลา query เราเขียนโค้ดเหมือนมันเป็นตารางเดียว:

```sql
SELECT count(*) FROM orders_partitioned;
```

```text
 count
-------
  8000
```

### ทำไมต้องใช้ Partitioning

| ปัญหาของตารางเดี่ยวขนาดใหญ่ | Partitioning ช่วยอย่างไร |
|---|---|
| ตารางมีหลายสิบ/ร้อยล้านแถว → query ช้าแม้มี index | Query ที่กรองด้วย partition key จะอ่านแค่บาง partition (partition pruning) |
| Index ทั้งตารางบวมมาก, VACUUM ใช้เวลานาน | แต่ละ partition มี index แยก ขนาดเล็กกว่า จัดการง่ายกว่า |
| ลบข้อมูลเก่า (เช่น log เก่ากว่า 2 ปี) ด้วย `DELETE` ช้ามาก เกิด bloat | `DROP TABLE` พาร์ทิชันเก่าทั้งก้อน แทบจะ instant |
| Backup/Restore ตารางเดียวขนาด TB ใช้เวลานาน | แยก backup ทีละ partition ตามความสำคัญ/ความถี่ในการเปลี่ยนแปลง |
| ต้องการ archive ข้อมูลเก่าไปที่อื่น | `DETACH PARTITION` แล้วย้ายไปเก็บที่ storage อื่นได้โดยไม่กระทบตารางหลัก |
| Maintenance (ANALYZE, REINDEX) ทั้งตารางใช้เวลานาน | ทำทีละ partition แบบขนาน หรือเฉพาะ partition ที่มีการเปลี่ยนแปลง |

### เมื่อไหร่ควรใช้ Partitioning

- ตารางมีขนาดใหญ่มาก (โดยทั่วไปเริ่มพิจารณาตั้งแต่หลักสิบล้านแถวขึ้นไป หรือขนาดเกิน RAM ที่ใช้ cache)
- Query ส่วนใหญ่มี pattern กรองผ่าน column เดียวกันสม่ำเสมอ (เช่น `order_date`, `region`, `tenant_id`)
- มีความต้องการลบ/archive ข้อมูลเก่าเป็นก้อน ๆ ตามช่วงเวลา (data retention)
- ต้องการแยก maintenance workload ให้ทำทีละส่วนได้

### เมื่อไหร่ "ไม่ควร" ใช้

- ตารางเล็ก (หลักหมื่น-แสนแถว) — ความซับซ้อนที่เพิ่มขึ้นไม่คุ้มกับประโยชน์
- Query ส่วนใหญ่ไม่ได้กรองผ่าน partition key เลย (จะกลายเป็นต้อง scan ทุก partition = ช้ากว่าตารางเดียวด้วยซ้ำ เพราะมี overhead ของการเปิดหลายตาราง)
- ต้องการ foreign key **จาก** ตารางอื่น**มาชี้ที่** ตาราง partitioned (ก่อน PG 12 ทำไม่ได้เลย, ปัจจุบันทำได้แต่ต้องระวังเรื่อง unique constraint ตาม Step 539)

> **สรุป:** Partitioning ไม่ใช่เวทมนตร์ที่ทำให้ query เร็วขึ้นเสมอ — มันช่วยได้มากเมื่อ query pattern สอดคล้องกับ partition key ที่เลือก และช่วยเรื่อง data lifecycle management เป็นหลัก

---

## Step 532: Declarative Partitioning ของ PostgreSQL (10+)

ก่อน PostgreSQL 10 การทำ partitioning ต้องใช้เทคนิค **table inheritance + trigger/rule** ด้วยมือทั้งหมด (จะสอนแนวทางนี้ใน Part 055 เพื่อให้เข้าใจที่มา) ตั้งแต่ PostgreSQL 10 เป็นต้นมา มี **Declarative Partitioning** ที่ built-in ในตัวภาษา SQL เอง ทำให้ syntax ชัดเจน ปลอดภัย และเร็วกว่าเดิมมาก

### Syntax หลัก 3 รูปแบบ

```sql
-- 1) RANGE: แบ่งตามช่วงค่าต่อเนื่อง (วันที่, ตัวเลข)
CREATE TABLE t1 (...) PARTITION BY RANGE (col);

-- 2) LIST: แบ่งตามรายการค่าที่ไม่ต่อเนื่อง (string, enum)
CREATE TABLE t2 (...) PARTITION BY LIST (col);

-- 3) HASH: แบ่งตาม hash ของค่า เพื่อกระจายข้อมูลเท่า ๆ กัน
CREATE TABLE t3 (...) PARTITION BY HASH (col);
```

จากนั้นสร้างตารางลูกด้วย `PARTITION OF`:

```sql
CREATE TABLE t1_child PARTITION OF t1 FOR VALUES FROM (v1) TO (v2);       -- RANGE
CREATE TABLE t2_child PARTITION OF t2 FOR VALUES IN (v1, v2, v3);          -- LIST
CREATE TABLE t3_child PARTITION OF t3 FOR VALUES WITH (MODULUS m, REMAINDER r); -- HASH
```

### ข้อจำกัดของ partition key

- Partition key เป็นได้ทั้ง column เดี่ยว, หลาย column (composite), หรือ expression (เช่น `date_trunc('month', order_date)`) แต่ expression ต้อง **immutable** เท่านั้น
- ห้ามใช้ column ที่เป็น system column หรือ column ที่ไม่มีอยู่ในตาราง
- ตารางแม่ (partitioned table) **ไม่มีข้อมูลจริงเก็บอยู่เลย** — มันเป็นแค่ "เปลือก" ที่กำหนด schema และ partition strategy เท่านั้น ข้อมูลทั้งหมดอยู่ในตารางลูก

### ตรวจสอบโครงสร้างด้วย catalog views

```sql
-- ดู partition key ของตาราง
SELECT partrelid::regclass AS table_name, partstrat, partattrs
FROM pg_partitioned_table
WHERE partrelid = 'orders_partitioned'::regclass;
```

```text
    table_name     | partstrat | partattrs
--------------------+-----------+-----------
 orders_partitioned | r         | 3
```

`partstrat` บอกกลยุทธ์: `r` = range, `l` = list, `h` = hash

```sql
-- ดูความสัมพันธ์ parent-child (ใช้ระบบเดียวกับ table inheritance)
SELECT inhrelid::regclass AS child, inhparent::regclass AS parent
FROM pg_inherits
WHERE inhparent = 'orders_partitioned'::regclass
LIMIT 5;
```

```text
      child      |       parent
------------------+---------------------
 orders_2023_01   | orders_partitioned
 orders_2023_02   | orders_partitioned
 orders_2023_03   | orders_partitioned
 orders_2023_04   | orders_partitioned
 orders_2023_05   | orders_partitioned
```

หรือใช้ `\d+ orders_partitioned` ใน `psql` จะเห็นบรรทัด `Partition key: RANGE (order_date)` และรายการ `Partitions:` ทั้งหมดพร้อมช่วงของแต่ละตัว

---

## Step 533: Range Partitioning เชิงลึก

**Range Partitioning** เหมาะกับข้อมูลที่มีลำดับต่อเนื่อง เช่น วันที่, timestamp, หรือตัวเลขที่เพิ่มขึ้นเรื่อย ๆ — กรณีคลาสสิกที่สุดคือข้อมูลธุรกรรม/log ตามเวลา ซึ่งตรงกับตาราง `orders_partitioned` ของเราพอดี

### Syntax เต็มรูปแบบ

```sql
CREATE TABLE orders_2026_01 PARTITION OF orders_partitioned
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
```

จุดสำคัญที่ต้องจำ: **ขอบเขตเป็นแบบ half-open interval `[FROM, TO)`** — ค่า `FROM` รวมอยู่ในพาร์ทิชัน แต่ค่า `TO` **ไม่รวม** (exclusive) ดังนั้นการเรียงพาร์ทิชันติดกันจึงไม่มีช่องว่างและไม่ทับซ้อนกัน:

```text
orders_2026_01: 2026-01-01 <= order_date < 2026-02-01
orders_2026_02: 2026-02-01 <= order_date < 2026-03-01
```

ถ้าพยายามสร้างพาร์ทิชันที่ช่วงทับซ้อนกับของเดิม PostgreSQL จะ error ทันที:

```sql
CREATE TABLE orders_overlap PARTITION OF orders_partitioned
    FOR VALUES FROM ('2024-06-15') TO ('2024-07-15');
```

```text
ERROR:  partition "orders_overlap" would overlap partition "orders_2024_06"
```

### เลือกความละเอียด (granularity): รายเดือน vs รายปี

| granularity | ข้อดี | ข้อเสีย |
|---|---|---|
| รายวัน | pruning ละเอียดสุด, เหมาะ log ปริมาณมหาศาลต่อวัน | จำนวนพาร์ทิชันเยอะมาก (365+/ปี) → catalog bloat, planning overhead |
| **รายเดือน** (ตัวอย่างของบทนี้) | สมดุลดี เหมาะกับ order/transaction ทั่วไป | ต้องสร้างพาร์ทิชันใหม่ทุกเดือน (ควรอัตโนมัติ) |
| รายปี | จำนวนพาร์ทิชันน้อย จัดการง่าย | pruning หยาบ ถ้า query กรองแค่บางเดือนก็ยังต้องอ่านทั้งปี |

โดยทั่วไป PostgreSQL แนะนำไม่ให้มีพาร์ทิชันมากเกินสองสามพันตารางต่อ partitioned table เดียว เพราะ planning time (การคำนวณ pruning ตอน plan) จะเพิ่มขึ้นตามจำนวนพาร์ทิชัน

### ใช้ MINVALUE / MAXVALUE สำหรับขอบเขตเปิด

บางครั้งต้องการ "ดักทุกอย่างก่อนวันที่หนึ่ง" หรือ "ดักทุกอย่างหลังวันที่หนึ่ง" โดยไม่ระบุขอบเขตชัดเจน:

```sql
-- ตัวอย่าง: เก็บข้อมูลก่อนปี 2023 ทั้งหมดไว้ในพาร์ทิชันเดียว (ข้อมูล legacy)
CREATE TABLE orders_before_2023 PARTITION OF orders_partitioned
    FOR VALUES FROM (MINVALUE) TO ('2023-01-01');
```

```sql
-- ตัวอย่าง: เผื่ออนาคตไกล ๆ ไม่จำกัดขอบบน (ใช้ระวัง เพราะจะกลืน insert ผิดพลาดในอนาคตด้วย)
CREATE TABLE orders_2026_onward PARTITION OF orders_partitioned
    FOR VALUES FROM ('2026-01-01') TO (MAXVALUE);
```

> ⚠️ ในทางปฏิบัติ เรามักไม่ใช้ `MAXVALUE` เป็นขอบบนถาวร เพราะจะทำให้พาร์ทิชันนั้นกลายเป็น "ถังขยะ" รับข้อมูลอนาคตทั้งหมดโดยไม่มีการแบ่งย่อยอีก แนะนำให้สร้างพาร์ทิชันใหม่ล่วงหน้าแทน (ดู Step 538)

### ทดสอบ insert แล้วดูว่าตกไปพาร์ทิชันไหน

```sql
INSERT INTO orders_partitioned (customer_id, order_date, status, total_amount)
VALUES (1, '2024-06-15', 'paid', 1500.00)
RETURNING tableoid::regclass, order_id, order_date;
```

```text
   tableoid    | order_id | order_date
---------------+----------+------------
 orders_2024_06 |     8001 | 2024-06-15
```

PostgreSQL routing ข้อมูลไปยังพาร์ทิชันที่ถูกต้องให้อัตโนมัติ แอปพลิเคชันไม่ต้องรู้เลยว่ามีพาร์ทิชันกี่ตัว หรือชื่ออะไร — insert ผ่านตารางแม่เสมอ

---

## Step 534: List Partitioning

**List Partitioning** เหมาะกับข้อมูลที่แบ่งกลุ่มตามค่าที่ไม่ต่อเนื่องและมีจำนวนจำกัด (categorical) เช่น ประเทศ, ภูมิภาค, สถานะ, tenant_id ในระบบ multi-tenant

มาสร้างตารางสาธิตใหม่ `orders_by_region` โดยแบ่งตาม region (คำนวณจาก `customers.country`) เพื่อแสดงแนวคิดนี้แยกจากตารางหลัก:

```sql
CREATE TABLE orders_by_region (
    order_id     BIGINT NOT NULL,
    customer_id  INTEGER NOT NULL,
    region       VARCHAR(20) NOT NULL,
    order_date   DATE NOT NULL,
    total_amount NUMERIC(12,2) NOT NULL,
    PRIMARY KEY (order_id, region)
) PARTITION BY LIST (region);

CREATE TABLE orders_region_th PARTITION OF orders_by_region
    FOR VALUES IN ('TH');

CREATE TABLE orders_region_sea PARTITION OF orders_by_region
    FOR VALUES IN ('SG', 'MY', 'VN', 'ID', 'PH');   -- หลายค่าในพาร์ทิชันเดียวได้

CREATE TABLE orders_region_ea PARTITION OF orders_by_region
    FOR VALUES IN ('JP', 'KR');

CREATE TABLE orders_region_other PARTITION OF orders_by_region
    DEFAULT;
```

จุดเด่นของ List Partitioning ที่ต่างจาก Range: **หนึ่งพาร์ทิชันรับได้หลายค่า** เช่น `orders_region_sea` รับทั้ง 5 ประเทศในเอเชียตะวันออกเฉียงใต้เข้าพาร์ทิชันเดียว ในขณะที่ Range ทำแบบนี้ไม่ได้เพราะต้องเป็นช่วงต่อเนื่อง

เติมข้อมูลตัวอย่างจาก `orders_partitioned` ที่มีอยู่แล้ว โดย map ประเทศเป็นรหัสภูมิภาค:

```sql
INSERT INTO orders_by_region (order_id, customer_id, region, order_date, total_amount)
SELECT
    o.order_id,
    o.customer_id,
    CASE c.country
        WHEN 'Thailand'      THEN 'TH'
        WHEN 'Singapore'     THEN 'SG'
        WHEN 'Malaysia'      THEN 'MY'
        WHEN 'Vietnam'       THEN 'VN'
        WHEN 'Indonesia'     THEN 'ID'
        WHEN 'Philippines'   THEN 'PH'
        WHEN 'Japan'         THEN 'JP'
        WHEN 'South Korea'   THEN 'KR'
        ELSE 'XX'
    END,
    o.order_date,
    o.total_amount
FROM orders_partitioned o
JOIN customers c ON c.customer_id = o.customer_id;

ANALYZE orders_by_region;
```

ตรวจสอบการกระจายตัว:

```sql
SELECT tableoid::regclass AS partition_name, count(*) AS row_count
FROM orders_by_region
GROUP BY tableoid
ORDER BY partition_name;
```

```text
  partition_name     | row_count
----------------------+-----------
 orders_region_ea     |      1980
 orders_region_other  |         0
 orders_region_sea    |      5040
 orders_region_th     |      1002
```

### NULL ใน List Partitioning

ถ้าต้องการให้พาร์ทิชันหนึ่งรับค่า `NULL` โดยเฉพาะ สามารถระบุ `FOR VALUES IN (NULL)` ได้ (หรือปล่อยให้ default partition รับไปหากมี):

```sql
CREATE TABLE example_null_bucket PARTITION OF orders_by_region
    FOR VALUES IN (NULL);
```

> โดยทั่วไปควรบังคับ `NOT NULL` ที่ partition key column ตั้งแต่แรก (เหมือนที่เราทำกับ `region VARCHAR(20) NOT NULL`) เพื่อไม่ต้องกังวลเรื่อง NULL routing เลย

---

## Step 535: Hash Partitioning

**Hash Partitioning** ใช้เมื่อไม่มี natural partition key ที่ชัดเจน (ไม่มีลำดับแบบ range, ไม่มีกลุ่มค่าจำกัดแบบ list) แต่ต้องการ **กระจายข้อมูลให้เท่า ๆ กัน** ระหว่างหลายพาร์ทิชัน เพื่อประโยชน์ด้าน:

- กระจายภาระ I/O และ maintenance (VACUUM, ANALYZE) ให้สมดุล
- รองรับ parallel query ที่ดีขึ้นเพราะแต่ละพาร์ทิชันขนาดใกล้เคียงกัน
- ไม่ต้องเดาว่าค่าจะกระจายยังไง (ต่างจาก list ที่ต้องรู้ค่าที่เป็นไปได้ล่วงหน้า)

ตัวอย่าง: สมมติเราต้องการกระจาย order ตาม `customer_id` เป็น 4 ก้อนเท่า ๆ กัน (ไม่สนใจวันที่หรือประเทศเลย):

```sql
CREATE TABLE orders_hash_demo (
    order_id     BIGINT NOT NULL,
    customer_id  INTEGER NOT NULL,
    order_date   DATE NOT NULL,
    total_amount NUMERIC(12,2) NOT NULL,
    PRIMARY KEY (order_id, customer_id)
) PARTITION BY HASH (customer_id);

CREATE TABLE orders_hash_0 PARTITION OF orders_hash_demo
    FOR VALUES WITH (MODULUS 4, REMAINDER 0);
CREATE TABLE orders_hash_1 PARTITION OF orders_hash_demo
    FOR VALUES WITH (MODULUS 4, REMAINDER 1);
CREATE TABLE orders_hash_2 PARTITION OF orders_hash_demo
    FOR VALUES WITH (MODULUS 4, REMAINDER 2);
CREATE TABLE orders_hash_3 PARTITION OF orders_hash_demo
    FOR VALUES WITH (MODULUS 4, REMAINDER 3);
```

`MODULUS 4` หมายถึงแบ่งเป็น 4 กลุ่มเท่ากันตามค่า hash ภายในของ PostgreSQL, `REMAINDER r` คือกลุ่มที่ `hash(value) % modulus = r` — **ทุกพาร์ทิชันต้องมี modulus เดียวกัน** ครบทุกค่า remainder (0 ถึง modulus-1) ไม่งั้นจะมีข้อมูลบางส่วนไม่มีที่ไป

เติมข้อมูลจาก `orders_partitioned` เพื่อทดสอบการกระจายตัว:

```sql
INSERT INTO orders_hash_demo (order_id, customer_id, order_date, total_amount)
SELECT order_id, customer_id, order_date, total_amount
FROM orders_partitioned;

ANALYZE orders_hash_demo;
```

```sql
SELECT tableoid::regclass AS partition_name, count(*) AS row_count
FROM orders_hash_demo
GROUP BY tableoid
ORDER BY partition_name;
```

```text
 partition_name | row_count
-----------------+-----------
 orders_hash_0   |      1987
 orders_hash_1   |      2015
 orders_hash_2   |      1998
 orders_hash_3   |      2001
(4 rows)
```

จะเห็นว่าแม้ `customer_id` เป็นตัวเลขสุ่มระหว่าง 1-500 การกระจายด้วย hash ก็ทำให้แต่ละพาร์ทิชันมีจำนวนแถวใกล้เคียงกันมาก (~2,000 ± เล็กน้อย) โดยไม่ต้องรู้ล่วงหน้าเลยว่าค่า `customer_id` จริง ๆ มีการกระจายตัวอย่างไร

### ข้อควรระวังของ Hash Partitioning

- **ไม่ช่วยเรื่อง pruning สำหรับ range query** — ถ้า query กรองด้วย `order_date` แต่พาร์ทิชันแบ่งตาม hash ของ `customer_id` การ query ด้วยวันที่จะต้อง scan ทุกพาร์ทิชันเสมอ (pruning เกิดได้เฉพาะเมื่อ query กรองด้วย equality บน partition key เท่านั้น เช่น `WHERE customer_id = 42`)
- เหมาะกับกรณี **equality lookup** บน key ที่กระจายสุ่ม (เช่น `user_id`, `session_id`) มากกว่ากรณี analytical query ที่กรองตามช่วงเวลา
- เปลี่ยนจำนวนพาร์ทิชัน (เพิ่ม modulus) ภายหลังทำได้ยากกว่า range/list เพราะต้องคำนวณ hash ใหม่ทั้งหมดและ redistribute ข้อมูล

---

## Step 536: Partition Pruning

**Partition Pruning** คือความสามารถของ query planner ที่จะ**ข้ามการอ่านพาร์ทิชันที่รู้แน่ชัดว่าไม่มีข้อมูลตรงกับเงื่อนไข** ออกไปโดยอัตโนมัติ — นี่คือประโยชน์หลักของ partitioning ในแง่ performance และเราจะพิสูจน์ให้เห็นด้วย `EXPLAIN` จริง ๆ

Pruning เปิดใช้งานเป็นค่าเริ่มต้น ควบคุมด้วย parameter:

```sql
SHOW enable_partition_pruning;
```

```text
 enable_partition_pruning
---------------------------
 on
```

### 1) Static Pruning (ตัดสินใจตอน plan time)

เมื่อ query มีเงื่อนไขเป็นค่าคงที่ (literal) ที่รู้ตั้งแต่ตอน planning เช่น:

```sql
EXPLAIN
SELECT * FROM orders_partitioned
WHERE order_date >= '2024-06-01' AND order_date < '2024-07-01';
```

```text
                                     QUERY PLAN
------------------------------------------------------------------------------------
 Seq Scan on orders_2024_06 orders_partitioned  (cost=0.00..38.75 rows=225 width=45)
   Filter: ((order_date >= '2024-06-01'::date) AND (order_date < '2024-07-01'::date))
```

สังเกตสิ่งสำคัญ: **ไม่มี `Append` node เลย!** เพราะ planner รู้ตั้งแต่แรกว่ามีพาร์ทิชันเดียว (`orders_2024_06`) ที่เข้าเงื่อนไขได้ จึงวางแผนสแกนตรงนั้นตรงเดียว ข้าม 35 พาร์ทิชันที่เหลือ + default partition ไปทั้งหมดโดยไม่ต้องพูดถึงด้วยซ้ำ — นี่คือ static pruning ที่มีประสิทธิภาพสูงสุด

ลองขยายช่วงให้ครอบคลุมหลายเดือน:

```sql
EXPLAIN
SELECT * FROM orders_partitioned
WHERE order_date >= '2024-01-01' AND order_date < '2024-04-01';
```

```text
                                    QUERY PLAN
-----------------------------------------------------------------------------------
 Append  (cost=0.00..116.25 rows=675 width=45)
   ->  Seq Scan on orders_2024_01 orders_partitioned_1  (cost=0.00..38.75 rows=225 ...)
         Filter: ((order_date >= '2024-01-01'::date) AND (order_date < '2024-04-01'::date))
   ->  Seq Scan on orders_2024_02 orders_partitioned_2  (cost=0.00..38.75 rows=225 ...)
         Filter: ((order_date >= '2024-01-01'::date) AND (order_date < '2024-04-01'::date))
   ->  Seq Scan on orders_2024_03 orders_partitioned_3  (cost=0.00..38.75 rows=225 ...)
         Filter: ((order_date >= '2024-01-01'::date) AND (order_date < '2024-04-01'::date))
```

ครั้งนี้มี `Append` node ที่รวม **แค่ 3 child scan** (มกรา-มีนาคม 2024) เท่านั้น จากทั้งหมด 37 พาร์ทิชันที่มีอยู่ (36 เดือน + default) — อีก 34 พาร์ทิชันถูก prune ออกไปตั้งแต่ plan time โดยไม่ปรากฏใน plan เลย

เทียบกับ query ที่**ไม่ได้กรองด้วย partition key** เลย:

```sql
EXPLAIN SELECT * FROM orders_partitioned WHERE status = 'paid';
```

```text
                                 QUERY PLAN
------------------------------------------------------------------------------
 Append  (cost=0.00..1450.63 rows=1600 width=45)
   ->  Seq Scan on orders_2023_01 orders_partitioned_1  (cost=0.00..38.75 ...)
         Filter: ((status)::text = 'paid'::text)
   ->  Seq Scan on orders_2023_02 orders_partitioned_2  (cost=0.00..38.75 ...)
         Filter: ((status)::text = 'paid'::text)
   ...
   ->  Seq Scan on orders_default orders_partitioned_37  (cost=0.00..1.00 ...)
         Filter: ((status)::text = 'paid'::text)
```

คราวนี้ **ทั้ง 37 พาร์ทิชันถูกสแกนหมด** เพราะ `status` ไม่ใช่ partition key จึงไม่มีข้อมูลใดที่ planner รู้ล่วงหน้าว่าจะไม่เจอ นี่คือเหตุผลว่าทำไมการเลือก partition key ต้องสอดคล้องกับ query pattern จริงของระบบ ไม่ใช่แค่คอลัมน์ที่ "ดูน่าจะใช้ได้"

### 2) Runtime (Dynamic) Pruning

บางกรณี planner ไม่สามารถรู้ค่าจริงได้ตอน plan time เช่น query ที่ใช้ **prepared statement แบบ generic plan** ซึ่งค่าพารามิเตอร์จะรู้แค่ตอน execute — PostgreSQL 16 เพิ่มความสามารถ `EXPLAIN (GENERIC_PLAN)` ให้ดู generic plan ได้ตรง ๆ โดยไม่ต้องผ่าน `PREPARE`/`EXECUTE`:

```sql
EXPLAIN (GENERIC_PLAN)
SELECT * FROM orders_partitioned
WHERE order_date >= $1 AND order_date < $2;
```

```text
                                    QUERY PLAN
------------------------------------------------------------------------------------
 Append  (cost=0.00..1450.00 rows=8000 width=45)
   ->  Seq Scan on orders_2023_01 orders_partitioned_1  (cost=0.00..38.75 ...)
         Filter: ((order_date >= $1) AND (order_date < $2))
   ->  Seq Scan on orders_2023_02 orders_partitioned_2  (cost=0.00..38.75 ...)
         Filter: ((order_date >= $1) AND (order_date < $2))
   ... (แสดงครบทุกพาร์ทิชัน เพราะยังไม่รู้ค่า $1, $2)
```

ที่ plan time นี้ planner ยังไม่รู้ว่า `$1`, `$2` คือค่าอะไร จึงต้องใส่ทุกพาร์ทิชันไว้ใน plan (generic plan) แต่ถ้าลองสั่งด้วยค่าจริงผ่าน `PREPARE`/`EXECUTE` แล้วดูตอน execute จริง จะเห็น **runtime pruning** เกิดขึ้น แสดงเป็นบรรทัด `Subplans Removed`:

```sql
PREPARE order_range(date, date) AS
    SELECT * FROM orders_partitioned WHERE order_date >= $1 AND order_date < $2;

EXPLAIN ANALYZE EXECUTE order_range('2024-05-01', '2024-06-01');
```

```text
                                                       QUERY PLAN
--------------------------------------------------------------------------------------------------------------------------
 Append  (cost=0.00..1450.00 rows=8000 width=45) (actual time=0.025..0.312 rows=209 loops=1)
   Subplans Removed: 35
   ->  Seq Scan on orders_2024_05 orders_partitioned_1  (cost=0.00..38.75 rows=225 width=45) (actual time=0.024..0.298 rows=209 loops=1)
         Filter: ((order_date >= $1) AND (order_date < $2))
 Planning Time: 0.412 ms
 Execution Time: 0.356 ms
```

บรรทัด **`Subplans Removed: 35`** คือหลักฐานของ runtime pruning — planner สร้าง plan แบบ generic ที่มี 36 subplan ไว้ก่อน แต่ตอน execute จริง executor คำนวณค่าพารามิเตอร์แล้วตัด (prune) เหลือ subplan เดียวที่ตรงเงื่อนไข ก่อนจะสแกนจริง

> **สรุปสำคัญ:** ไม่ว่าจะเป็น static หรือ dynamic pruning หัวใจคือ query ต้องมีเงื่อนไขบน partition key เสมอ ถ้าไม่มี ก็ไม่มีอะไรให้ prune

---

## Step 537: Default Partition และการจัดการแถวที่ไม่เข้าเงื่อนไขใดเลย

**Default Partition** คือพาร์ทิชันพิเศษที่รับข้อมูลทุกแถวที่ **ไม่ตรงกับช่วง/ค่าของพาร์ทิชันอื่นใดเลย** ในตารางแม่ ต่อ partitioned table หนึ่งตารางมี default partition ได้ **สูงสุดตัวเดียว**:

```sql
CREATE TABLE orders_default PARTITION OF orders_partitioned DEFAULT;
```

ทดสอบ insert วันที่นอกช่วง (นอก 2023-2025 ที่เราสร้างพาร์ทิชันไว้):

```sql
INSERT INTO orders_partitioned (customer_id, order_date, status, total_amount)
VALUES (5, '2030-01-15', 'pending', 999.00)
RETURNING tableoid::regclass, order_date;
```

```text
   tableoid    | order_date
----------------+------------
 orders_default | 2030-01-15
```

ถ้าไม่มี default partition เลย การ insert แถวที่ไม่เข้าเงื่อนไขใดจะ error ทันที:

```text
ERROR:  no partition of relation "orders_partitioned" found for row
DETAIL:  Partition key of the failing row contains (order_date) = (2030-01-15).
```

### ข้อควรระวังสำคัญ: การเพิ่มพาร์ทิชันใหม่เมื่อมี default partition อยู่แล้ว

สมมติภายหลังเราต้องการสร้างพาร์ทิชันของปี 2030 จริง ๆ (เพราะเวลาผ่านไปถึงปีนั้นแล้ว):

```sql
CREATE TABLE orders_2030_01 PARTITION OF orders_partitioned
    FOR VALUES FROM ('2030-01-01') TO ('2030-02-01');
```

คำสั่งนี้จะทำให้ PostgreSQL ต้อง**สแกน default partition ทั้งตาราง** เพื่อตรวจสอบว่ามีแถวใดที่ควรจะอยู่ในช่วง `2030-01-01` ถึง `2030-02-01` หลุดเข้าไปอยู่ใน default หรือไม่ (เพราะถ้ามี จะกลายเป็นข้อมูลซ้ำซ้อน/ขัดแย้งกัน) ถ้าพบแถวที่ตรงเงื่อนไข คำสั่งจะ **error ทันที**:

```text
ERROR:  updated partition constraint for default partition "orders_default" would be violated by some row
```

วิธีแก้: ต้องย้ายข้อมูลที่เกี่ยวข้องออกจาก default partition ก่อน แล้วค่อยสร้างพาร์ทิชันใหม่:

```sql
BEGIN;

-- ย้ายข้อมูลเดือนมกราคม 2030 ออกจาก default ไปพักไว้ตารางชั่วคราว
CREATE TEMP TABLE tmp_orders_2030_01 AS
SELECT * FROM orders_default
WHERE order_date >= '2030-01-01' AND order_date < '2030-02-01';

DELETE FROM orders_default
WHERE order_date >= '2030-01-01' AND order_date < '2030-02-01';

-- ตอนนี้สร้างพาร์ทิชันใหม่ได้แล้วโดยไม่ error
CREATE TABLE orders_2030_01 PARTITION OF orders_partitioned
    FOR VALUES FROM ('2030-01-01') TO ('2030-02-01');

-- ใส่ข้อมูลกลับเข้าไปผ่านตารางแม่ ให้ routing ไปพาร์ทิชันใหม่เอง
INSERT INTO orders_partitioned SELECT * FROM tmp_orders_2030_01;

COMMIT;
```

> ⚠️ **บทเรียนสำคัญ:** ถ้า default partition มีข้อมูลจำนวนมาก การสแกนตรวจสอบนี้จะ**ช้ามาก**และ**ล็อกตารางระหว่างตรวจสอบ** ดังนั้นแนวทางปฏิบัติที่ดีคือ**สร้างพาร์ทิชันล่วงหน้าเสมอ** (ดู Step 538) เพื่อไม่ให้ default partition ต้องรับข้อมูลจริงเลยนอกจากข้อมูลผิดปกติจริง ๆ (data quality issue) ซึ่งควรมีน้อยมากหรือไม่มีเลย

---

## Step 538: การจัดการ Partition — เพิ่ม/ย้าย/ลบ

การมีตารางแม่และพาร์ทิชันไม่ได้จบแค่ตอนสร้าง — งาน DBA/backend engineer ต้องดูแลวงจรชีวิตของพาร์ทิชันต่อเนื่อง

### 1) เพิ่มพาร์ทิชันใหม่ล่วงหน้า

ไม่ควรรอให้ถึงเดือนหน้าแล้วค่อยสร้างพาร์ทิชัน ควรมี job (cron/pg_cron/pg_partman) สร้างล่วงหน้าอย่างน้อย 1-2 เดือน:

```sql
CREATE TABLE IF NOT EXISTS orders_2026_02 PARTITION OF orders_partitioned
    FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');
```

ตัวอย่างฟังก์ชันช่วยสร้างพาร์ทิชันเดือนถัดไปอัตโนมัติ (เรียกผ่าน `pg_cron` ทุกวันที่ 1 ของเดือน):

```sql
CREATE OR REPLACE FUNCTION ensure_next_month_partition()
RETURNS void LANGUAGE plpgsql AS $$
DECLARE
    next_start date := date_trunc('month', now() + interval '1 month')::date;
    next_end   date := date_trunc('month', now() + interval '2 month')::date;
    part_name  text := 'orders_' || to_char(next_start, 'YYYY_MM');
BEGIN
    EXECUTE format(
        'CREATE TABLE IF NOT EXISTS %I PARTITION OF orders_partitioned
             FOR VALUES FROM (%L) TO (%L)',
        part_name, next_start, next_end
    );
END $$;
```

> ในงาน production ระดับ world-class มักใช้ extension **`pg_partman`** ซึ่งจัดการเรื่องนี้ให้อัตโนมัติทั้งหมด (สร้างพาร์ทิชันล่วงหน้า, ลบ/archive พาร์ทิชันเก่าตาม retention policy) แทนการเขียนสคริปต์เองทั้งหมด

### 2) DETACH PARTITION — แยกพาร์ทิชันออกจากตารางแม่

`DETACH` ทำให้พาร์ทิชันกลายเป็นตารางอิสระ (standalone table) ทันที โดยไม่ลบข้อมูล — เหมาะสำหรับ archive ข้อมูลเก่าไปเก็บที่อื่นก่อน drop จริง:

```sql
ALTER TABLE orders_partitioned DETACH PARTITION orders_2023_01;
```

ตอนนี้ `orders_2023_01` เป็นตารางธรรมดา ไม่ผูกกับ `orders_partitioned` แล้ว — สามารถ `pg_dump` แยกเก็บ, ย้ายไป tablespace/เซิร์ฟเวอร์อื่น หรือลบทิ้งภายหลังได้ตามสบาย

ตั้งแต่ PostgreSQL 14 มี `DETACH ... CONCURRENTLY` ที่ไม่ล็อกตารางแม่แบบยาว ๆ (สำคัญมากสำหรับตารางที่มี traffic สูงตลอดเวลา):

```sql
ALTER TABLE orders_partitioned DETACH PARTITION orders_2023_02 CONCURRENTLY;
```

คำสั่งนี้ใช้สอง phase: ระยะแรกทำงานแบบไม่บล็อก แล้วต้องรัน `ALTER TABLE ... DETACH PARTITION ... FINALIZE` ต่อ (ถ้าขั้นตอนแรกถูกขัดจังหวะ) เพื่อให้เสร็จสมบูรณ์

### 3) ATTACH PARTITION — นำตารางกลับเข้ามาเป็นพาร์ทิชัน

```sql
ALTER TABLE orders_partitioned
    ATTACH PARTITION orders_2023_01
    FOR VALUES FROM ('2023-01-01') TO ('2023-02-01');
```

ใช้ได้ทั้งกรณี reattach ตารางที่เพิ่ง detach ไป หรือแม้แต่ attach ตารางที่มีข้อมูลอยู่แล้ว (เช่น bulk load ข้อมูลเก่าเข้าตารางเปล่าก่อน แล้วค่อย attach — เร็วกว่า insert ทีละแถวผ่านตารางแม่มาก เพราะข้าม overhead การ route + ไม่ต้องเขียน WAL ซ้ำ)

### 4) DROP PARTITION แทน DELETE ขนาดใหญ่

นี่คือประโยชน์ที่ชัดเจนที่สุดข้อหนึ่งของ partitioning ในแง่ data retention ลองเทียบสองวิธีลบข้อมูลเก่ากว่า 3 ปี:

```sql
-- วิธีเดิม (ไม่มี partitioning): ช้า, สร้าง WAL มหาศาล, ต้อง VACUUM ตามหลัง
DELETE FROM orders_monolithic WHERE order_date < '2023-01-01';
```

```sql
-- วิธีใหม่ (มี partitioning): แทบจะ instant, metadata operation เท่านั้น
DROP TABLE orders_2022_12;  -- หรือ DROP ทีละพาร์ทิชันที่หมดอายุ
```

| ประเด็นเปรียบเทียบ | `DELETE FROM ... WHERE ...` | `DROP TABLE partition_name` |
|---|---|---|
| ความซับซ้อนเวลา | O(n) ต้องสแกน+ลบทีละแถว | O(1) ทาง metadata (ไม่สนใจจำนวนแถว) |
| WAL ที่สร้าง | มาก (1 WAL record ต่อแถวที่ลบโดยประมาณ) | น้อยมาก (แค่ drop metadata) |
| ต้อง VACUUM ตามหลังไหม | ต้อง (เพื่อคืนพื้นที่ dead tuple) | ไม่ต้อง (พื้นที่คืนทันทีเมื่อ drop) |
| Index bloat | เกิด (index ยังมี entry ชี้ไปแถวที่ลบจนกว่าจะ vacuum) | ไม่เกิด (index ของพาร์ทิชันถูกลบไปพร้อมกัน) |
| ระยะเวลา lock | นานตามจำนวนแถว | สั้นมาก (ACCESS EXCLUSIVE ระยะสั้น) |

ตัวอย่างการทำ retention job แบบสมบูรณ์ (archive ก่อน drop):

```sql
DO $$
DECLARE
    cutoff date := date_trunc('month', now() - interval '36 months')::date;
    part_name text := 'orders_' || to_char(cutoff, 'YYYY_MM');
BEGIN
    -- 1) แยกออกจากตารางแม่แบบไม่บล็อก
    EXECUTE format('ALTER TABLE orders_partitioned DETACH PARTITION %I CONCURRENTLY', part_name);

    -- 2) (ในงานจริง: pg_dump ตารางนี้ไปเก็บ cold storage ที่นี่ ก่อนขั้นตอนถัดไป)

    -- 3) ลบตารางทิ้งหลัง archive สำเร็จ
    EXECUTE format('DROP TABLE %I', part_name);
END $$;
```

---

## Step 539: Index บน Partitioned Table

การสร้าง index บนตาราง partitioned มีพฤติกรรมพิเศษที่ต้องเข้าใจให้ชัดเจน

### สร้าง index ที่ตารางแม่ → propagate ไปทุกพาร์ทิชันอัตโนมัติ

```sql
CREATE INDEX idx_orders_customer ON orders_partitioned (customer_id);
```

คำสั่งนี้จะสร้าง **"partitioned index"** ที่ระดับตารางแม่ และ PostgreSQL จะสร้าง index จริงในทุกพาร์ทิชันที่มีอยู่แล้วโดยอัตโนมัติ **รวมถึงพาร์ทิชันที่จะถูกสร้างในอนาคตด้วย** (เมื่อ `CREATE TABLE ... PARTITION OF` ใหม่ ระบบจะสร้าง index ตามให้เองทันที):

```sql
SELECT
    schemaname,
    tablename,
    indexname
FROM pg_indexes
WHERE tablename LIKE 'orders_202%'
ORDER BY tablename
LIMIT 6;
```

```text
 schemaname |   tablename    |            indexname
------------+-----------------+----------------------------------
 public     | orders_2023_01 | orders_2023_01_pkey
 public     | orders_2023_01 | orders_2023_01_customer_id_idx
 public     | orders_2023_02 | orders_2023_02_pkey
 public     | orders_2023_02 | orders_2023_02_customer_id_idx
 public     | orders_2023_03 | orders_2023_03_pkey
 public     | orders_2023_03 | orders_2023_03_customer_id_idx
```

ตรวจสอบสถานะ index ระดับ parent ด้วย `\d+ orders_partitioned` จะเห็นบรรทัดคล้าย ๆ นี้:

```text
Indexes:
    "orders_partitioned_pkey" PRIMARY KEY, btree (order_id, order_date)
    "idx_orders_customer" btree (customer_id) INVALID
```

> ในบางช่วง (ระหว่างสร้าง) index ของ parent อาจแสดงเป็น `INVALID` จนกว่าจะสร้างเสร็จครบทุกพาร์ทิชัน เมื่อเสร็จสมบูรณ์แล้วจะกลายเป็น valid ปกติ

### Local Index vs Global Index

นี่คือแนวคิดที่สำคัญที่สุดของหัวข้อนี้:

- **Local Index** (สิ่งที่ PostgreSQL รองรับ): index แต่ละตัวผูกอยู่กับพาร์ทิชันเดียวเท่านั้น พาร์ทิชันละ 1 physical index object แยกจากกันโดยสิ้นเชิง แม้จะดูเหมือนเป็น "index เดียว" ผ่าน `idx_orders_customer` แต่จริง ๆ แล้วมันคือกลุ่มของ index ย่อยหลายสิบตัวที่ PostgreSQL บริหารให้ดูเป็นก้อนเดียว
- **Global Index** (สิ่งที่ PostgreSQL **ไม่รองรับ**): index เดียวที่ครอบคลุมทุกแถวข้ามทุกพาร์ทิชันในโครงสร้างข้อมูลเดียว (Oracle และบางระบบมีความสามารถนี้)

**ผลกระทบสำคัญที่สุด: UNIQUE constraint / PRIMARY KEY**

เพราะไม่มี global index ทำให้ PostgreSQL **ไม่สามารถรับประกัน uniqueness ข้ามพาร์ทิชันได้** ถ้า unique constraint ไม่รวม partition key เข้าไปด้วย นี่คือเหตุผลที่ตาราง `orders_partitioned` ต้องประกาศ:

```sql
PRIMARY KEY (order_id, order_date)   -- ต้องมี order_date (partition key) รวมอยู่ด้วยเสมอ
```

ถ้าพยายามประกาศแค่ `PRIMARY KEY (order_id)` เฉย ๆ (ไม่รวม `order_date`) PostgreSQL จะปฏิเสธทันที:

```sql
CREATE TABLE orders_bad (
    order_id   BIGSERIAL PRIMARY KEY,   -- error!
    order_date DATE NOT NULL
) PARTITION BY RANGE (order_date);
```

```text
ERROR:  unique constraint on partitioned table must include all partitioning columns
DETAIL:  PRIMARY KEY constraint on table "orders_bad" lacks column "order_date" which is part of the partition key.
```

**ผลลัพธ์ในทางปฏิบัติ:** ถ้าต้องการให้ `order_id` เป็น unique อย่างแท้จริงข้ามทุกพาร์ทิชัน (ไม่พึ่ง `order_date` ประกอบ) จะต้องหาวิธีอื่น เช่น:

- ใช้ sequence (`BIGSERIAL`) ที่รับประกันความไม่ซ้ำในตัวมันเองอยู่แล้วในระดับ generation (แต่ PostgreSQL จะไม่ enforce uniqueness ทาง constraint ให้)
- สร้างตารางแยกต่างหากที่ไม่ partition เพื่อเก็บ mapping `order_id` แล้วใช้ unique constraint ปกติตรงนั้น (ซับซ้อนขึ้น แต่ทำได้)
- ยอมรับว่า business key ที่แท้จริงคือ `(order_id, order_date)` คู่กัน ซึ่งเป็นแนวทางที่ใช้บ่อยที่สุดและเรียบง่ายที่สุด

### CREATE INDEX CONCURRENTLY บนตาราง partitioned

ตั้งแต่ PostgreSQL 11 รองรับ `CREATE INDEX CONCURRENTLY` บนตารางแม่ได้ (สร้างทีละพาร์ทิชันแบบไม่ล็อกยาว):

```sql
CREATE INDEX CONCURRENTLY idx_orders_status ON orders_partitioned (status);
```

หากขั้นตอนถูกขัดจังหวะกลางคัน อาจเหลือบางพาร์ทิชันที่ไม่มี index หรือ index เป็น `INVALID` — ต้องตรวจสอบและสร้างเพิ่มเป็นรายตัว แล้วผูกกลับด้วย:

```sql
ALTER INDEX idx_orders_status ATTACH PARTITION orders_2024_07_status_idx;
```

คำสั่งนี้ใช้เชื่อม local index ที่สร้างแยกไว้ (เช่นสร้างด้วยมือทีละพาร์ทิชันเพื่อลด lock) ให้กลายเป็นส่วนหนึ่งของ partitioned index ที่ parent อย่างเป็นทางการ

---

## Step 540: แบบฝึกหัดรวม — ออกแบบกลยุทธ์ Partition สำหรับข้อมูล Order ย้อนหลังหลายปี

มาลองประกอบทุกอย่างที่เรียนมาเป็นกรณีศึกษาจริง:

> **โจทย์:** บริษัทอีคอมเมิร์ซแห่งหนึ่งมีตาราง orders สะสมมา 5 ปี ปริมาณ ~50 ล้านแถว query ส่วนใหญ่ (>90%) กรองด้วย `order_date` ในช่วง 3 เดือนล่าสุด ทีมกฎหมายกำหนดให้ต้องเก็บข้อมูลอย่างน้อย 7 ปีเพื่อการตรวจสอบ แต่ระบบ operational เข้าถึงแค่ข้อมูลไม่เกิน 1 ปีเป็นหลัก

### แนวทางออกแบบ

**1. เลือก partition strategy:** Range partitioning ตาม `order_date` รายเดือน — สอดคล้องกับ query pattern ที่กรองตามวันที่เป็นส่วนใหญ่

**2. แบ่งชั้นข้อมูลตามอายุ (tiering):**

| ชั้นข้อมูล | อายุ | ที่เก็บ | granularity |
|---|---|---|---|
| Hot | 0-3 เดือน | Primary DB, SSD, index ครบ | รายเดือน |
| Warm | 3-12 เดือน | Primary DB เดียวกัน แต่ index น้อยกว่า | รายเดือน |
| Cold | 1-7 ปี | Detach แล้วย้ายไป archive server/tablespace ราคาถูก | อาจ roll-up เป็นรายไตรมาส/รายปี |
| Beyond retention | > 7 ปี | Drop ทิ้ง (หลัง export เป็น backup file ตามนโยบาย) | - |

**3. Schema sketch:**

```sql
CREATE TABLE orders_v2 (
    order_id     BIGSERIAL,
    customer_id  INTEGER REFERENCES customers(customer_id),
    order_date   DATE NOT NULL,
    status       VARCHAR(20) NOT NULL DEFAULT 'pending',
    total_amount NUMERIC(12,2) NOT NULL,
    PRIMARY KEY (order_id, order_date)
) PARTITION BY RANGE (order_date);
```

**4. Automation runbook (ใช้ pg_partman หรือ cron + PL/pgSQL เอง):**

```sql
-- ทุกต้นเดือน: สร้างพาร์ทิชันล่วงหน้า 3 เดือน
SELECT ensure_next_month_partition();  -- ฟังก์ชันจาก Step 538 ปรับให้ loop 3 รอบ

-- ทุกไตรมาส: archive พาร์ทิชันที่มีอายุเกิน 1 ปี ไป cold storage
--   1. DETACH CONCURRENTLY
--   2. pg_dump -> object storage / archive server
--   3. ATTACH เข้า archive server (ตาราง partitioned แยกต่างหากที่ archive server)

-- ทุกปี: ตรวจสอบพาร์ทิชันที่อายุเกิน 7 ปี แล้ว DROP ทิ้งตามนโยบาย retention
```

**5. Index policy:** สร้าง index ตามที่ query จริงต้องการที่ parent (`customer_id`, `status`) ให้ propagate อัตโนมัติ — แต่พิจารณาว่าพาร์ทิชันที่เป็น cold data (แทบไม่มี query แล้ว) อาจไม่จำเป็นต้องมี index ทุกตัวเหมือน hot data เพื่อประหยัดพื้นที่ (ทำได้ผ่าน `CREATE INDEX ... ON ONLY parent` แล้วเลือกสร้างเฉพาะพาร์ทิชันที่ต้องการ)

**6. Monitoring:** ตรวจสอบขนาดและจำนวนแถวต่อพาร์ทิชันสม่ำเสมอ เพื่อจับ pattern การเติบโตที่ผิดปกติ (เช่น เดือนไหนมีแถวเยอะผิดปกติ อาจบ่งชี้ปัญหา data quality):

```sql
SELECT
    tableoid::regclass AS partition_name,
    count(*) AS row_count,
    pg_size_pretty(pg_total_relation_size(tableoid)) AS total_size
FROM orders_partitioned
GROUP BY tableoid
ORDER BY partition_name;
```

นี่คือแนวคิดของ **partitioning + lifecycle management** ที่ระบบระดับ world-class ใช้จริงในการรับมือกับข้อมูลสะสมระยะยาว

---

## สรุปท้ายบท

- **Table Partitioning** แบ่งตารางใหญ่หนึ่งตารางเป็นตารางย่อยทางกายภาพ (partitions) โดย query ยังมองเห็นเหมือนตารางเดียว ช่วยเรื่อง query performance (ผ่าน pruning), maintenance, และ data lifecycle
- PostgreSQL 10+ มี **Declarative Partitioning** ในตัว รองรับ 3 กลยุทธ์: `RANGE`, `LIST`, `HASH`
- **Range Partitioning** เหมาะกับข้อมูลต่อเนื่อง เช่นวันที่ — ใช้ `FOR VALUES FROM (...) TO (...)` แบบ half-open interval
- **List Partitioning** เหมาะกับข้อมูล categorical เช่นประเทศ/ภูมิภาค — ใช้ `FOR VALUES IN (...)` และรับได้หลายค่าต่อพาร์ทิชัน
- **Hash Partitioning** ใช้เมื่อไม่มี natural key ชัดเจน ต้องการกระจายข้อมูลเท่า ๆ กัน — ใช้ `FOR VALUES WITH (MODULUS m, REMAINDER r)`
- **Partition Pruning** คือหัวใจของ performance benefit — เกิดได้ทั้งแบบ static (plan-time, ไม่มี Append เลยถ้าเหลือพาร์ทิชันเดียว) และ dynamic (runtime, แสดงเป็น `Subplans Removed` ใน `EXPLAIN ANALYZE`)
- **Default Partition** รับข้อมูลที่ไม่เข้าเงื่อนไขใด แต่ต้องระวังเรื่อง performance ตอน attach พาร์ทิชันใหม่ที่ overlap กับข้อมูลใน default
- การจัดการวงจรชีวิต: สร้างพาร์ทิชันล่วงหน้าเสมอ, ใช้ `DETACH`/`ATTACH` สำหรับ archive, และ `DROP TABLE` แทน `DELETE` สำหรับลบข้อมูลเก่าจำนวนมาก (เร็วกว่ามาก ไม่ต้อง VACUUM ตามหลัง)
- Index บน partitioned table เป็นแบบ **local index เท่านั้น** (ไม่มี global index) ทำให้ unique constraint ต้องรวม partition key เสมอ

---

## แบบฝึกหัด

<details>
<summary><strong>แบบฝึกหัดที่ 1:</strong> เขียนคำสั่งสร้างพาร์ทิชันใหม่สำหรับเดือนมกราคม 2026 (`orders_2026_01`) บนตาราง `orders_partitioned`</summary>

```sql
CREATE TABLE orders_2026_01 PARTITION OF orders_partitioned
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
```

ตรวจสอบว่าสร้างสำเร็จ:

```sql
SELECT relid::regclass FROM pg_partition_tree('orders_partitioned')
WHERE relid::regclass::text = 'orders_2026_01';
```
</details>

<details>
<summary><strong>แบบฝึกหัดที่ 2:</strong> เขียน query ที่กรอง order_date เฉพาะไตรมาสที่ 3 ของปี 2024 (กรกฎาคม-กันยายน) แล้วใช้ EXPLAIN ยืนยันว่ามีการ pruning เหลือ 3 พาร์ทิชัน</summary>

```sql
EXPLAIN
SELECT * FROM orders_partitioned
WHERE order_date >= '2024-07-01' AND order_date < '2024-10-01';
```

ผลลัพธ์ที่คาดหวัง: `Append` node ที่มีลูกเพียง 3 ตัว คือ `orders_2024_07`, `orders_2024_08`, `orders_2024_09` เท่านั้น — ไม่ปรากฏพาร์ทิชันอื่นใน plan เลย ยืนยันว่าอีก 34 พาร์ทิชันถูก prune ออกตั้งแต่ plan time
</details>

<details>
<summary><strong>แบบฝึกหัดที่ 3:</strong> ออกแบบตารางใหม่ชื่อ `orders_by_status` ที่แบ่งด้วย List Partitioning ตามคอลัมน์ `status` โดยแยกเป็น 3 กลุ่ม: (completed, shipped), (pending, paid), และ default สำหรับ cancelled/อื่น ๆ</summary>

```sql
CREATE TABLE orders_by_status (
    order_id     BIGINT NOT NULL,
    status       VARCHAR(20) NOT NULL,
    order_date   DATE NOT NULL,
    total_amount NUMERIC(12,2) NOT NULL,
    PRIMARY KEY (order_id, status)
) PARTITION BY LIST (status);

CREATE TABLE orders_status_fulfilled PARTITION OF orders_by_status
    FOR VALUES IN ('completed', 'shipped');

CREATE TABLE orders_status_open PARTITION OF orders_by_status
    FOR VALUES IN ('pending', 'paid');

CREATE TABLE orders_status_other PARTITION OF orders_by_status
    DEFAULT;
```
</details>

<details>
<summary><strong>แบบฝึกหัดที่ 4:</strong> เขียน query ตรวจสอบจำนวนแถวและขนาดพื้นที่ (pretty size) ของแต่ละพาร์ทิชันในตาราง orders_hash_demo เรียงจากมากไปน้อย</summary>

```sql
SELECT
    tableoid::regclass AS partition_name,
    count(*) AS row_count,
    pg_size_pretty(pg_total_relation_size(tableoid)) AS total_size
FROM orders_hash_demo
GROUP BY tableoid
ORDER BY row_count DESC;
```

ผลลัพธ์ควรแสดงว่าทั้ง 4 พาร์ทิชันมีจำนวนแถวใกล้เคียงกันมาก (สะท้อนคุณสมบัติของ hash partitioning ที่กระจายข้อมูลได้สม่ำเสมอ)
</details>

<details>
<summary><strong>แบบฝึกหัดที่ 5:</strong> เขียนขั้นตอนแบบ transaction เดียว เพื่อ detach พาร์ทิชัน orders_2023_03 ออกจากตารางแม่แบบไม่บล็อกยาว แล้วลบทิ้งหลังจากมั่นใจว่า archive สำเร็จแล้ว</summary>

```sql
-- ขั้นที่ 1: detach แบบ concurrent (ไม่ต้องอยู่ใน transaction เดียวกับ DROP เพราะ CONCURRENTLY ห้ามอยู่ใน explicit transaction block)
ALTER TABLE orders_partitioned DETACH PARTITION orders_2023_03 CONCURRENTLY;

-- ขั้นที่ 2 (นอก DB): pg_dump -t orders_2023_03 > orders_2023_03_archive.sql

-- ขั้นที่ 3: หลังยืนยันว่า archive ไฟล์ถูกต้องและเก็บปลอดภัยแล้ว
DROP TABLE orders_2023_03;
```

หมายเหตุ: `DETACH ... CONCURRENTLY` ต้องรันนอก transaction block ที่มีคำสั่งอื่นร่วมด้วย (เป็นข้อจำกัดของ PostgreSQL เกี่ยวกับคำสั่งที่ทำงานข้าม transaction ภายใน)
</details>

<details>
<summary><strong>แบบฝึกหัดที่ 6:</strong> จำลองสถานการณ์แถวข้อมูลปี 2035 หลุดไปอยู่ default partition แล้วลองสร้างพาร์ทิชันของปี 2035 จริง ๆ ดูว่าเกิด error อะไร แล้วแก้ไขให้ถูกต้อง</summary>

```sql
-- ขั้นที่ 1: insert ข้อมูลที่ยังไม่มีพาร์ทิชันรองรับ จะตกไปที่ default
INSERT INTO orders_partitioned (customer_id, order_date, status, total_amount)
VALUES (10, '2035-05-10', 'pending', 500.00);

-- ขั้นที่ 2: ลองสร้างพาร์ทิชันปี 2035 เดือนพฤษภาคม -> จะ error
CREATE TABLE orders_2035_05 PARTITION OF orders_partitioned
    FOR VALUES FROM ('2035-05-01') TO ('2035-06-01');
-- ERROR: updated partition constraint for default partition "orders_default"
--        would be violated by some row

-- ขั้นที่ 3: ย้ายแถวที่เกี่ยวข้องออกจาก default ก่อน
CREATE TEMP TABLE tmp_2035_05 AS
SELECT * FROM orders_default
WHERE order_date >= '2035-05-01' AND order_date < '2035-06-01';

DELETE FROM orders_default
WHERE order_date >= '2035-05-01' AND order_date < '2035-06-01';

-- ขั้นที่ 4: ตอนนี้สร้างพาร์ทิชันได้แล้ว
CREATE TABLE orders_2035_05 PARTITION OF orders_partitioned
    FOR VALUES FROM ('2035-05-01') TO ('2035-06-01');

-- ขั้นที่ 5: ใส่ข้อมูลกลับผ่านตารางแม่
INSERT INTO orders_partitioned SELECT * FROM tmp_2035_05;
```
</details>

<details>
<summary><strong>แบบฝึกหัดที่ 7:</strong> สร้าง index บน customer_id ที่ตารางแม่ orders_partitioned แล้วเขียน query ตรวจสอบว่า index ถูกสร้างในทุกพาร์ทิชันจริงหรือไม่ (นับจำนวน)</summary>

```sql
CREATE INDEX idx_orders_customer2 ON orders_partitioned (customer_id);

-- นับจำนวนพาร์ทิชันทั้งหมด เทียบกับจำนวน index ที่มีชื่อลงท้ายด้วย customer_id2_idx
SELECT count(*) AS total_partitions
FROM pg_partition_tree('orders_partitioned')
WHERE isleaf;

SELECT count(*) AS total_indexes_created
FROM pg_indexes
WHERE indexname LIKE '%customer_id2_idx';
```

ทั้งสองตัวเลขควรเท่ากัน (หรือใกล้เคียงถ้ามีบางพาร์ทิชันที่ถูก detach ไปแล้ว) ยืนยันว่า index propagate ไปทุกพาร์ทิชันจริง
</details>

<details>
<summary><strong>แบบฝึกหัดที่ 8:</strong> ทดลองสร้างตาราง hash partitioning ใหม่ที่แบ่งตาม order_id แทน customer_id ด้วย modulus 8 แล้วเปรียบเทียบการกระจายตัวกับตัวอย่างในบท (modulus 4 บน customer_id)</summary>

```sql
CREATE TABLE orders_hash8 (
    order_id     BIGINT NOT NULL,
    customer_id  INTEGER NOT NULL,
    order_date   DATE NOT NULL,
    total_amount NUMERIC(12,2) NOT NULL,
    PRIMARY KEY (order_id)
) PARTITION BY HASH (order_id);

DO $$
DECLARE i int;
BEGIN
    FOR i IN 0..7 LOOP
        EXECUTE format(
            'CREATE TABLE orders_hash8_%s PARTITION OF orders_hash8
                 FOR VALUES WITH (MODULUS 8, REMAINDER %s)', i, i
        );
    END LOOP;
END $$;

INSERT INTO orders_hash8 (order_id, customer_id, order_date, total_amount)
SELECT order_id, customer_id, order_date, total_amount FROM orders_partitioned;

SELECT tableoid::regclass, count(*) FROM orders_hash8 GROUP BY 1 ORDER BY 1;
```

เนื่องจาก `order_id` เป็น `BIGSERIAL` ที่เพิ่มขึ้นเรื่อย ๆ (unique เสมอ ไม่ซ้ำ) ควรกระจายตัวสม่ำเสมอมากกับทุก modulus เพราะฟังก์ชัน hash ภายในของ PostgreSQL ออกแบบมาให้กระจายค่าที่ต่อเนื่องกันได้ดี
</details>

<details>
<summary><strong>แบบฝึกหัดที่ 9:</strong> ใช้ \timing เปรียบเทียบเวลาระหว่าง DELETE ข้อมูลทั้งพาร์ทิชันหนึ่งเดือน กับ DROP TABLE พาร์ทิชันนั้นตรง ๆ (ทำบนตารางทดสอบที่ copy ข้อมูลมา ไม่กระทบข้อมูลจริง)</summary>

```sql
-- เตรียมพาร์ทิชันทดสอบสำหรับเปรียบเทียบ (สมมติมีข้อมูลซ้ำสำหรับทดสอบ)
\timing on

-- วิธีที่ 1: DELETE (ช้ากว่า เพราะต้องสแกน+ลบทีละแถว+เขียน WAL ต่อแถว)
DELETE FROM orders_partitioned
WHERE order_date >= '2024-01-01' AND order_date < '2024-02-01';
-- Time: ประมาณหลายสิบ-หลายร้อย ms ขึ้นกับจำนวนแถวและ index ที่มี

-- วิธีที่ 2: DROP TABLE (เร็วกว่ามาก เพราะเป็นแค่ metadata operation)
DROP TABLE orders_2024_02;
-- Time: ต่ำกว่า 1 ms โดยทั่วไป ไม่ขึ้นกับจำนวนแถวในพาร์ทิชันเลย

\timing off
```

ข้อสังเกต: ยิ่งจำนวนแถวในพาร์ทิชันมากขึ้น (หลักแสน-ล้านแถว) ความต่างของเวลาระหว่างสองวิธีนี้จะยิ่งชัดเจนขึ้นมาก เพราะ `DELETE` ใช้เวลาแปรผันตามจำนวนแถว ในขณะที่ `DROP TABLE` ใช้เวลาคงที่เสมอ
</details>

<details>
<summary><strong>แบบฝึกหัดที่ 10:</strong> ออกแบบ retention policy (เป็นคำอธิบาย + โครง SQL) สำหรับตาราง log ที่ต้องเก็บข้อมูล 7 ปี โดยข้อมูล 90 วันล่าสุดต้อง query ได้เร็วที่สุด และข้อมูลเก่ากว่า 2 ปีให้ archive ออกจาก primary database</summary>

**แนวคิดการออกแบบ:**

1. Range partition รายเดือน ตามคอลัมน์ timestamp ของ log
2. สร้างพาร์ทิชันล่วงหน้า 2 เดือนเสมอ (automation job รายวัน/รายสัปดาห์)
3. Index เต็มรูปแบบเฉพาะพาร์ทิชันของ 90 วันล่าสุด (hot data) ส่วนพาร์ทิชันเก่ากว่านั้นอาจลด index ที่ไม่จำเป็นออกเพื่อประหยัดพื้นที่และเวลา write
4. ทุกเดือน: ตรวจหาพาร์ทิชันอายุเกิน 2 ปี → `DETACH CONCURRENTLY` → `pg_dump`/export ไปเก็บที่ cold storage (เช่น object storage) → `DROP TABLE`
5. ทุกปี: ตรวจสอบว่าไม่มีการ archive ตกหล่น (มีพาร์ทิชันอายุเกิน 7 ปีหลงเหลือหรือไม่ ถ้ามีให้ลบทิ้งตามนโยบาย)
6. เก็บ metadata การ archive แต่ละพาร์ทิชัน (ชื่อไฟล์ archive, วันที่ archive, checksum) ไว้ในตาราง audit แยกต่างหาก เพื่อการตรวจสอบย้อนหลังตามข้อกำหนดทางกฎหมาย

```sql
CREATE TABLE app_logs (
    log_id      BIGSERIAL,
    logged_at   TIMESTAMP NOT NULL,
    level       VARCHAR(10) NOT NULL,
    message     TEXT NOT NULL,
    PRIMARY KEY (log_id, logged_at)
) PARTITION BY RANGE (logged_at);

-- ตาราง audit สำหรับบันทึกการ archive
CREATE TABLE log_archive_audit (
    partition_name TEXT PRIMARY KEY,
    archived_at    TIMESTAMP NOT NULL DEFAULT now(),
    archive_path   TEXT NOT NULL,
    row_count      BIGINT NOT NULL
);
```

การออกแบบลักษณะนี้ทำให้ primary database มีขนาดจำกัดที่ราว 2 ปีของข้อมูลเสมอ (แม้ business requirement คือ 7 ปี) เพราะข้อมูลส่วนที่เหลือถูกโยกออกไปเก็บที่ cold storage ซึ่งราคาถูกกว่าและไม่กระทบ performance ของระบบ operational
</details>

---

**บทถัดไป:** [Part 055 — Table Inheritance](./part-055-table-inheritance.md)
