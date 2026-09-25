# Performance Tuning: Query-Level Deep Dive

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับมืออาชีพ | Part 074

---

## เป้าหมายการเรียนรู้

บทนี้เป็นบทที่ **ลงลึกกว่า Part 045** (ซึ่งปูพื้นฐานการอ่าน `EXPLAIN`, index scan types, และการ tuning เบื้องต้น) โดยจะพาไปสู่เทคนิคระดับ production จริง ที่ DBA และ senior engineer ใช้วินิจฉัยและแก้ปัญหา query ที่ซับซ้อน เมื่อเรียนจบบทนี้ คุณจะสามารถ:

1. อ่าน `EXPLAIN (ANALYZE, BUFFERS)` เพื่อดู I/O จริงระดับ buffer (shared hit/read) ไม่ใช่แค่ cost/time
2. วินิจฉัย query ที่ planner ประมาณ row count ผิดพลาดอย่างรุนแรง และแก้ด้วย extended statistics (`CREATE STATISTICS`)
3. เข้าใจเทคนิคบังคับแผน query (query hinting) ทั้งแบบ built-in GUC และ extension อย่าง `pg_hint_plan` พร้อมรู้ข้อจำกัดและความเสี่ยง
4. อ่านและวิเคราะห์ parallel query plan (`Gather`, `Gather Merge`, `Parallel Seq Scan`) และรู้ว่าเงื่อนไขใดทำให้ planner เลือก parallel
5. เข้าใจผลกระทบของ CTE materialization ต่อ performance ในสถานการณ์ query ซับซ้อนจริง
6. เขียน bulk insert/update ที่มีประสิทธิภาพสูงด้วย pattern `UNNEST`
7. วินิจฉัย lock contention ด้วย `pg_locks` ร่วมกับ `pg_stat_activity`
8. ตั้งค่า `statement_timeout` เพื่อป้องกัน query ที่หลุดควบคุม
9. ใช้ `pgbench` ทดสอบเปรียบเทียบ performance ของ query สองแบบ (A/B testing)
10. วินิจฉัยและแก้ปัญหา production incident จำลอง (query ที่จู่ๆ ช้าลง) แบบครบวงจร ทีละขั้นตอน

---

## เตรียมข้อมูล

เราจะใช้ e-commerce schema เดิมจากบทก่อนหน้า แต่คราวนี้จะสร้างข้อมูลจำนวนมาก (bulk data) เพื่อให้ `EXPLAIN ANALYZE` แสดงพฤติกรรมของ planner ที่สมจริง ไม่ใช่แค่ตารางไม่กี่สิบแถวที่ planner ทำอะไรก็เร็วเท่ากันหมด

```sql
-- ===== Schema =====
CREATE TABLE categories (
    category_id   SERIAL PRIMARY KEY,
    category_name VARCHAR(100)
);

CREATE TABLE products (
    product_id     SERIAL PRIMARY KEY,
    product_name   VARCHAR(150),
    category_id    INTEGER REFERENCES categories(category_id),
    unit_price     NUMERIC(10,2),
    stock_quantity INTEGER
);

CREATE TABLE customers (
    customer_id SERIAL PRIMARY KEY,
    first_name  VARCHAR(60),
    country     VARCHAR(60)
);

CREATE TABLE orders (
    order_id    SERIAL PRIMARY KEY,
    customer_id INTEGER REFERENCES customers(customer_id),
    order_date  TIMESTAMPTZ DEFAULT now(),
    status      VARCHAR(20)
);

CREATE TABLE order_items (
    order_item_id SERIAL PRIMARY KEY,
    order_id      INTEGER REFERENCES orders(order_id),
    product_id    INTEGER REFERENCES products(product_id),
    quantity      INTEGER,
    unit_price    NUMERIC(10,2)
);
```

```sql
-- ===== Bulk data (~ 5,000 - 170,000+ แถวต่อ table) =====
INSERT INTO categories (category_name)
SELECT 'Category ' || g
FROM generate_series(1, 20) g;

INSERT INTO products (product_name, category_id, unit_price, stock_quantity)
SELECT
    'Product ' || g,
    1 + floor(random() * 20)::int,
    round((random() * 990 + 10)::numeric, 2),
    floor(random() * 500)::int
FROM generate_series(1, 5000) g;

INSERT INTO customers (first_name, country)
SELECT
    'Customer ' || g,
    CASE
        WHEN random() < 0.6 THEN 'Thailand'
        WHEN random() < 0.8 THEN 'Vietnam'
        WHEN random() < 0.9 THEN 'Singapore'
        ELSE 'Malaysia'
    END
FROM generate_series(1, 8000) g;

INSERT INTO orders (customer_id, order_date, status)
SELECT
    1 + floor(random() * 8000)::int,
    now() - (random() * interval '365 days'),
    (ARRAY['pending','paid','shipped','completed','cancelled'])[1 + floor(random()*5)::int]
FROM generate_series(1, 40000) g;

INSERT INTO order_items (order_id, product_id, quantity, unit_price)
SELECT
    1 + floor(random() * 40000)::int,
    1 + floor(random() * 5000)::int,
    1 + floor(random() * 5)::int,
    round((random() * 990 + 10)::numeric, 2)
FROM generate_series(1, 120000) g;

VACUUM ANALYZE;
```

ผลลัพธ์: `products` 5,000 แถว, `customers` 8,000 แถว, `orders` 40,000 แถว, `order_items` 120,000 แถว — มากพอที่ planner จะต้องตัดสินใจจริงว่าจะใช้ seq scan, index scan, hash join, merge join หรือ parallel query แบบไหน

> **หมายเหตุ:** ทุก `EXPLAIN (ANALYZE, ...)` ในบทนี้เป็นผลลัพธ์จริงที่รันบน PostgreSQL 16 กับข้อมูลชุดนี้ ตัวเลข cost/time ที่เห็นอาจต่างจากเครื่องของคุณเล็กน้อยตาม hardware แต่ **รูปแบบของแผน (plan shape)** และ**สัดส่วนของความแตกต่าง**จะใกล้เคียงกัน

---

## Step 731: Buffer Analysis ด้วย EXPLAIN (ANALYZE, BUFFERS)

Part 045 สอนให้อ่าน `cost` และ `actual time` แต่ทั้งสองค่านี้ไม่ได้บอกว่า query ไป "แตะดิสก์" มากแค่ไหน ตัวเลข `time` วัดจาก wall-clock ซึ่งผันผวนตามภาระของเครื่อง (CPU contention, OS scheduling) ขณะที่ `BUFFERS` บอกจำนวน **page (block ขนาด 8KB)** ที่ query อ่านจริง ซึ่งเสถียรกว่ามากและสะท้อนต้นทุนที่แท้จริงของ query

รูปแบบคำสั่ง:

```sql
EXPLAIN (ANALYZE, BUFFERS, VERBOSE, FORMAT TEXT)
SELECT ...;
```

ทดสอบกับ query จริงตอนที่ shared_buffers ยัง "เย็น" (cold cache — พึ่ง restart PostgreSQL หรือ table นี้ยังไม่เคยถูกอ่านเข้า shared_buffers):

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT o.order_id, o.order_date, o.status, c.first_name
FROM orders o
JOIN customers c ON c.customer_id = o.customer_id
WHERE o.status = 'paid'
  AND o.order_date > now() - interval '30 days';
```

ผลลัพธ์รอบแรก (cold cache):

```
Hash Join  (cost=329.75..763.95 rows=663 width=33) (actual time=2.556..6.288 rows=522 loops=1)
  Hash Cond: (o.customer_id = c.customer_id)
  Buffers: shared read=340
  ->  Bitmap Heap Scan on orders o  (cost=90.75..523.21 rows=663 width=24) (actual time=0.311..3.848 rows=522 loops=1)
        Recheck Cond: ((status)::text = 'paid'::text)
        Filter: (order_date > (now() - '30 days'::interval))
        Rows Removed by Filter: 7247
        Heap Blocks: exact=272
        Buffers: shared read=281
        ->  Bitmap Index Scan on idx_orders_status  (cost=0.00..90.59 rows=7773 width=0) (actual time=0.209..0.210 rows=7769 loops=1)
              Index Cond: ((status)::text = 'paid'::text)
              Buffers: shared read=9
  ->  Hash  (cost=139.00..139.00 rows=8000 width=17) (actual time=2.204..2.206 rows=8000 loops=1)
        Buckets: 8192  Batches: 1  Memory Usage: 470kB
        Buffers: shared read=59
        ->  Seq Scan on customers c  (cost=0.00..139.00 rows=8000 width=17) (actual time=0.014..0.984 rows=8000 loops=1)
              Buffers: shared read=59
Planning:
  Buffers: shared hit=218 read=43
Planning Time: 1.491 ms
Execution Time: 6.361 ms
```

รันคำสั่งเดิมซ้ำทันที (warm cache — page อยู่ใน shared_buffers แล้ว):

```
Hash Join  (cost=329.75..763.95 rows=663 width=33) (actual time=2.383..4.760 rows=522 loops=1)
  Hash Cond: (o.customer_id = c.customer_id)
  Buffers: shared hit=340
  ->  Bitmap Heap Scan on orders o  (cost=90.75..523.21 rows=663 width=24) (actual time=0.207..2.454 rows=522 loops=1)
        Recheck Cond: ((status)::text = 'paid'::text)
        Filter: (order_date > (now() - '30 days'::interval))
        Rows Removed by Filter: 7247
        Heap Blocks: exact=272
        Buffers: shared hit=281
        ->  Bitmap Index Scan on idx_orders_status  (cost=0.00..90.59 rows=7773 width=0) (actual time=0.166..0.166 rows=7769 loops=1)
              Index Cond: ((status)::text = 'paid'::text)
              Buffers: shared hit=9
  ->  Hash  (cost=139.00..139.00 rows=8000 width=17) (actual time=2.140..2.141 rows=8000 loops=1)
        Buckets: 8192  Batches: 1  Memory Usage: 470kB
        Buffers: shared hit=59
        ->  Seq Scan on customers c  (cost=0.00..139.00 rows=8000 width=17) (actual time=0.003..0.818 rows=8000 loops=1)
              Buffers: shared hit=59
Execution Time: 4.819 ms
```

### วิธีอ่าน

| คำศัพท์ | ความหมาย |
|---|---|
| `shared hit` | อ่าน page จาก **shared_buffers** (memory ของ PostgreSQL เอง) — เร็วที่สุด ไม่มี system call |
| `shared read` | page ไม่อยู่ใน shared_buffers ต้องขอจาก OS (ซึ่งอาจ hit OS page cache หรือ physical disk ก็ได้ — PostgreSQL ไม่รู้ว่า OS cache hit หรือไม่) |
| `shared dirtied` | page ถูกแก้ไข (dirty) ระหว่าง query เช่น hint bit update |
| `shared written` | page ที่ dirty ถูกเขียนกลับดิสก์ (มักเกิดตอน buffer ถูก evict เพื่อเอาที่ให้ page ใหม่) |
| `local ...` | สำหรับ temp table |
| `temp read` / `temp written` | การ spill ไปดิสก์ของ sort/hash ที่ work_mem ไม่พอ (จะเจอในหัวข้อ Step 740) |

จุดสำคัญ: ทั้งสองรอบมี **cost และ row estimate เท่ากันเป๊ะ** (cost=329.75..763.95) เพราะ cost มาจาก planner statistics ล้วนๆ ไม่เกี่ยวกับ cache แต่ **actual time ต่างกัน** (6.288ms vs 4.760ms) เพราะรอบแรกต้องรอ I/O จริง คนที่ tuning จาก `time` อย่างเดียวอาจสับสนว่า "query เดียวกัน ทำไมเร็วช้าไม่คงที่" — คำตอบมักอยู่ที่ buffer cache state ซึ่ง `BUFFERS` เผยให้เห็นตรงๆ

> **กฎการอ่าน production query ที่ช้า:** ถ้าเห็น `shared read` จำนวนมากในหลายๆ ครั้งที่รัน (ไม่ใช่แค่ครั้งแรกหลัง restart) แปลว่าตารางนั้นใหญ่กว่า `shared_buffers` มาก หรือ working set ไม่ fit ใน memory — นี่คือสัญญาณให้พิจารณาเพิ่ม `shared_buffers`, เพิ่ม index ให้ query อ่าน page น้อยลง หรือพิจารณา partitioning (Part 076)

ตัวอย่างการดู buffer ระดับ session ทั้งหมดด้วย `pg_stat_statements` (ต้องเปิด extension ก่อน จะกล่าวถึงอีกครั้งใน Part 077):

```sql
-- ต้องมี shared_preload_libraries = 'pg_stat_statements' ใน postgresql.conf
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

SELECT
    query,
    calls,
    shared_blks_hit,
    shared_blks_read,
    round(shared_blks_hit::numeric
          / NULLIF(shared_blks_hit + shared_blks_read, 0) * 100, 2) AS hit_ratio_pct
FROM pg_stat_statements
ORDER BY shared_blks_read DESC
LIMIT 10;
```

`hit_ratio_pct` ต่ำกว่า 99% อย่างต่อเนื่องสำหรับ query ที่รันบ่อย มักเป็นสัญญาณว่าต้อง tuning เพิ่ม

---

## Step 732: วินิจฉัย Query ที่ Estimate ผิดพลาดมาก (Correlation & Extended Statistics)

Planner ของ PostgreSQL โดยปกติจะ**สมมติว่าแต่ละคอลัมน์เป็นอิสระต่อกัน (independence assumption)** เมื่อคำนวณ selectivity ของเงื่อนไขหลายตัวใน `WHERE` ที่ join ด้วย `AND` ถ้าในความเป็นจริงคอลัมน์เหล่านั้น**มีความสัมพันธ์กัน (correlated)** ตัวเลขที่ประมาณได้จะผิดพลาดหนัก และนำไปสู่การเลือกแผนที่แย่ (เช่น nested loop ที่คาด row น้อยแต่จริงมาก, หรือ hash join ที่จอง memory น้อยเกินไป)

### สร้างสถานการณ์ correlation จริง

สมมติสินค้าใน category 5 ส่วนใหญ่ "หมดสต๊อก" (`stock_quantity = 0`) เพราะเป็นสินค้าขายดี:

```sql
UPDATE products SET stock_quantity = 0
WHERE category_id = 5 AND random() < 0.85;

ANALYZE products;
```

รัน query กรองด้วยทั้งสองคอลัมน์นี้พร้อมกัน:

```sql
EXPLAIN ANALYZE
SELECT * FROM products
WHERE category_id = 5 AND stock_quantity = 0;
```

```
Seq Scan on products  (cost=0.00..118.00 rows=10 width=30) (actual time=0.368..0.386 rows=207 loops=1)
  Filter: ((category_id = 5) AND (stock_quantity = 0))
  Rows Removed by Filter: 4793
Planning Time: 0.077 ms
Execution Time: 0.399 ms
```

**Planner คาด 10 แถว แต่จริง 207 แถว — ผิดพลาดถึง ~20 เท่า** สาเหตุคือ planner คูณ selectivity ของ `category_id = 5` (1/20 = 5%) กับ selectivity ของ `stock_quantity = 0` (ประมาณ 1/500) แล้วได้ค่าที่ต่ำเกินจริงมาก เพราะไม่รู้ว่าทั้งสองคอลัมน์มีความสัมพันธ์กัน

ในตารางเล็กแบบนี้ ผลกระทบยังไม่รุนแรง (เป็นแค่ seq scan ที่ยังทำงานเร็ว) แต่ถ้า query แบบนี้ถูกฝังอยู่กลางแผนที่ซับซ้อน — เช่นเป็นผลลัพธ์ที่ต้อง join ต่อกับตารางอื่นด้วย nested loop — การประมาณ 10 แถวผิดพลาดเป็น 207 แถว อาจทำให้ planner เลือก nested loop ที่ loop ซ้ำ 207 ครั้งโดยไม่รู้ตัว กลายเป็น query ที่ช้าอย่างมากในโปรดักชัน

### แก้ด้วย CREATE STATISTICS (Extended Statistics)

```sql
CREATE STATISTICS products_cat_stock_stat (ndistinct, dependencies, mcv)
ON category_id, stock_quantity FROM products;

ANALYZE products;
```

รัน query เดิมอีกครั้ง:

```sql
EXPLAIN ANALYZE
SELECT * FROM products
WHERE category_id = 5 AND stock_quantity = 0;
```

```
Seq Scan on products  (cost=0.00..121.00 rows=207 width=30) (actual time=0.420..0.459 rows=207 loops=1)
  Filter: ((category_id = 5) AND (stock_quantity = 0))
  Rows Removed by Filter: 4793
Planning Time: 0.198 ms
Execution Time: 0.486 ms
```

**ตอนนี้ประมาณได้ 207 แถว ตรงกับความจริงพอดี** ปัญหาคือ ณ ตารางนี้ยังคงเป็น seq scan เหมือนเดิม (เพราะไม่มี index ที่เหมาะ) แต่ในสถานการณ์จริงที่ query นี้อยู่กลางแผนซับซ้อนกว่านี้ ตัวเลข estimate ที่แม่นยำจะเปลี่ยนการตัดสินใจของ planner ในขั้นตอนถัดไปทั้งหมด (join order, join method, memory allocation)

### สามประเภทของ Extended Statistics

| kind | ใช้เมื่อ |
|---|---|
| `dependencies` (functional dependencies) | คอลัมน์หนึ่ง "บอก" ค่าของอีกคอลัมน์ได้เกือบสมบูรณ์ เช่น `city` → `zip_code` (ใช้ดีที่สุดกับเงื่อนไข equality `=`) |
| `ndistinct` | แก้การประมาณ distinct value ของกลุ่มคอลัมน์ผิดพลาดตอนทำ `GROUP BY col1, col2` |
| `mcv` (most common values) | เก็บ "combination ของค่าที่พบบ่อย" ของหลายคอลัมน์ไว้ตรงๆ ช่วยแก้ estimate ของเงื่อนไข equality ร่วมกันได้แม่นยำที่สุด (เหมือนตัวอย่างข้างต้น) |

> **ข้อควรระวังสำคัญ:** ทดลองสร้าง statistics แบบ `dependencies` อย่างเดียวกับเงื่อนไขที่เป็น **range predicate** (`>`, `<`, `BETWEEN`) เช่น `status = 'pending' AND order_date > now() - interval '7 days'` แล้วพบว่า **ไม่ช่วยแก้ estimate เลย** เพราะ `dependencies` ถูกออกแบบมาสำหรับเงื่อนไข equality เป็นหลัก สำหรับ range predicate ที่ correlate กับคอลัมน์อื่น ปัจจุบัน PostgreSQL (16/17) ยังไม่มีกลไก extended statistics ที่แก้ได้ตรงๆ — ทางเลือกที่เหลือคือ ปรับ query ให้ใช้ index ที่เหมาะสม, denormalize เพิ่มคอลัมน์ที่คำนวณไว้ล่วงหน้า, หรือยอมรับความคลาดเคลื่อนและ monitor ด้วย `pg_stat_statements`

ตรวจสอบ extended statistics ที่มีอยู่ในระบบ:

```sql
SELECT stxname, stxkind, stxrelid::regclass AS table_name
FROM pg_statistic_ext;

-- ดูรายละเอียดค่าที่เก็บจริง (ต้อง join กับ pg_statistic_ext_data)
SELECT stxname, stxdndistinct, stxddependencies
FROM pg_statistic_ext
JOIN pg_statistic_ext_data ON stxoid = stxdinherit OR stxoid = pg_statistic_ext.oid
WHERE stxname = 'products_cat_stock_stat';
```

ลบ statistics ที่ไม่ต้องการ:

```sql
DROP STATISTICS IF EXISTS products_cat_stock_stat;
```

---

## Step 733: เทคนิคบังคับแผน Query (Query Hinting) เมื่อ Planner เลือกผิด

PostgreSQL **ไม่มี query hint syntax ในตัวแบบ MySQL หรือ Oracle** (`/*+ INDEX(...) */`) นี่เป็นการตัดสินใจเชิงปรัชญาของทีมพัฒนา: การให้ hint ตายตัวจะทำให้ query "ค้าง" กับแผนเดิมแม้ข้อมูลจะเปลี่ยนไปในอนาคต ซึ่งอาจกลายเป็นปัญหาที่แย่กว่าเดิม

### ทางออกที่ 1: แก้ต้นเหตุก่อนเสมอ (สาเหตุที่พบบ่อยที่สุดของแผนแย่คือ "ไม่มี index" ไม่ใช่ "planner โง่")

```sql
EXPLAIN ANALYZE
SELECT o.order_id, oi.quantity, p.product_name
FROM orders o
JOIN order_items oi ON oi.order_id = o.order_id
JOIN products p ON p.product_id = oi.product_id
WHERE o.order_id = 12345;
```

ก่อนมี index บน `order_items.order_id`:

```
Nested Loop  (cost=0.57..2306.55 rows=4 width=20) (actual time=1.783..11.657 rows=2 loops=1)
  ->  Nested Loop  (cost=0.29..2273.35 rows=4 width=12) (actual time=1.755..11.610 rows=2 loops=1)
        ->  Index Only Scan using orders_pkey on orders o  (cost=0.29..8.31 rows=1 width=4) (actual time=0.053..0.055 rows=1 loops=1)
              Index Cond: (order_id = 12345)
        ->  Seq Scan on order_items oi  (cost=0.00..2265.00 rows=4 width=12) (actual time=1.699..11.549 rows=2 loops=1)
              Filter: (order_id = 12345)
              Rows Removed by Filter: 119998
  ->  Index Scan using products_pkey on products p  (cost=0.28..8.30 rows=1 width=16) (actual time=0.019..0.019 rows=1 loops=2)
        Index Cond: (product_id = oi.product_id)
Execution Time: 11.765 ms
```

Query นี้ดูเหมือนจะ "ต้องการ hint" เพื่อบังคับให้ใช้ index scan บน `order_items` แต่ต้นเหตุจริงคือ**ไม่มี index บนคอลัมน์นั้นเลย** ทำให้ต้อง seq scan 120,000 แถวทุกครั้งแม้ต้องการแค่ 2 แถว:

```sql
CREATE INDEX idx_order_items_order_id ON order_items(order_id);
ANALYZE order_items;
```

```sql
EXPLAIN ANALYZE
SELECT o.order_id, oi.quantity, p.product_name
FROM orders o
JOIN order_items oi ON oi.order_id = o.order_id
JOIN products p ON p.product_id = oi.product_id
WHERE o.order_id = 12345;
```

```
Nested Loop  (cost=4.90..61.06 rows=4 width=20) (actual time=0.109..0.123 rows=2 loops=1)
  ->  Nested Loop  (cost=4.61..27.85 rows=4 width=12) (actual time=0.060..0.064 rows=2 loops=1)
        ->  Index Only Scan using orders_pkey on orders o  (cost=0.29..8.31 rows=1 width=4) (actual time=0.022..0.023 rows=1 loops=1)
              Index Cond: (order_id = 12345)
        ->  Bitmap Heap Scan on order_items oi  (cost=4.32..19.51 rows=4 width=12) (actual time=0.035..0.037 rows=2 loops=1)
              Recheck Cond: (order_id = 12345)
              Heap Blocks: exact=2
              ->  Bitmap Index Scan on idx_order_items_order_id  (cost=0.00..4.32 rows=4 width=0) (actual time=0.025..0.025 rows=2 loops=1)
                    Index Cond: (order_id = 12345)
  ->  Index Scan using products_pkey on products p  (cost=0.28..8.30 rows=1 width=16) (actual time=0.028..0.028 rows=1 loops=2)
        Index Cond: (product_id = oi.product_id)
Execution Time: 0.174 ms
```

**11.765ms → 0.174ms (เร็วขึ้น ~67 เท่า) โดยไม่ต้อง hint ใดๆ** — สอนบทเรียนสำคัญ: **90% ของกรณีที่คนคิดว่า "ต้อง hint" จริงๆ แล้วแก้ได้ด้วย index หรือ `ANALYZE` ให้ statistics ทันสมัย** นี่ควรเป็นขั้นตอนแรกเสมอก่อนคิดถึง hint

### ทางออกที่ 2: ปรับพฤติกรรม planner ด้วย GUC เฉพาะ session (ใช้เมื่อจำเป็นจริงๆ เท่านั้น)

เมื่อไม่มี index อื่นให้เลือก และคุณต้องการทดสอบว่า "ถ้าบังคับให้ planner ไม่ใช้วิธี X จะได้แผนแบบไหน" สามารถปิด planner method ชั่วคราวได้:

```sql
SET enable_seqscan = off;
SET enable_nestloop = off;
SET enable_hashjoin = off;
SET enable_mergejoin = off;
SET enable_bitmapscan = off;
SET enable_indexscan = off;

-- รัน EXPLAIN ANALYZE เพื่อดูว่า planner จะเลือกแผนสำรองแบบไหน
EXPLAIN ANALYZE SELECT ...;

RESET enable_seqscan;  -- อย่าลืม RESET ทุกตัวที่ SET ไว้!
RESET ALL;              -- หรือ reset ทุกอย่างในครั้งเดียว
```

ตัวอย่างจริง — ทดสอบ query ที่ default planner เลือก Seq Scan (เพราะ selectivity สูงถึง 60% ของตาราง จึงคุ้มกว่า):

```sql
EXPLAIN ANALYZE SELECT * FROM customers WHERE country = 'Thailand';
```

```
Seq Scan on customers  (cost=0.00..159.00 rows=4879 width=25) (actual time=0.008..1.026 rows=4879 loops=1)
  Filter: ((country)::text = 'Thailand'::text)
  Rows Removed by Filter: 3121
Execution Time: 1.256 ms
```

บังคับให้ใช้ index (`idx_customers_country`) แทน:

```sql
SET enable_seqscan = off;
EXPLAIN ANALYZE SELECT * FROM customers WHERE country = 'Thailand';
RESET enable_seqscan;
```

```
Bitmap Heap Scan on customers  (cost=62.09..182.08 rows=4879 width=25) (actual time=0.127..0.545 rows=4879 loops=1)
  Recheck Cond: ((country)::text = 'Thailand'::text)
  Heap Blocks: exact=59
  ->  Bitmap Index Scan on idx_customers_country  (cost=0.00..60.88 rows=4879 width=0) (actual time=0.111..0.111 rows=4879 loops=1)
        Index Cond: ((country)::text = 'Thailand'::text)
Execution Time: 0.683 ms
```

สังเกตว่า **cost ที่ planner ประมาณ** (159.00 สำหรับ seq scan เทียบกับ 182.08 สำหรับ bitmap scan) บอกว่า seq scan ควรถูกกว่า แต่ actual time ในการรันจริงกลับพบว่า bitmap scan เร็วกว่าเล็กน้อยในการทดสอบนี้ — เพราะข้อมูลทั้งหมด "warm" อยู่ใน memory cache หมดแล้ว (ตารางเล็ก) ในสภาพแวดล้อมจริงที่ตารางใหญ่กว่า RAM มาก การ random I/O ของ index scan อาจกลับมาช้ากว่า sequential scan ได้จริง — **นี่คือเหตุผลที่ห้ามเชื่อผลทดสอบบนเครื่อง dev เพียงอย่างเดียว ต้องทดสอบกับขนาดข้อมูลและ hardware ที่ใกล้เคียง production**

### ทางออกที่ 3: pg_hint_plan extension (third-party)

สำหรับกรณีที่ต้องการ hint ระดับ query เดียว (ไม่กระทบ session อื่น) โดยไม่ต้องปิด planner method ทั้ง session สามารถติดตั้ง extension ชื่อ `pg_hint_plan` (พัฒนาโดยทีม NTT/ossc-db) ซึ่ง **ไม่ได้เป็นส่วนหนึ่งของ core PostgreSQL** ต้อง compile และเพิ่มใน `shared_preload_libraries` ก่อน:

```ini
# postgresql.conf
shared_preload_libraries = 'pg_hint_plan'
```

```sql
CREATE EXTENSION pg_hint_plan;

-- ใส่ hint เป็น comment ก่อนคำสั่ง SQL
/*+
    IndexScan(o idx_orders_status)
    NestLoop(o oi)
    Leading((o oi p))
*/
SELECT o.order_id, oi.quantity, p.product_name
FROM orders o
JOIN order_items oi ON oi.order_id = o.order_id
JOIN products p ON p.product_id = oi.product_id
WHERE o.status = 'paid';
```

Hint ที่ใช้บ่อย:

| Hint | ความหมาย |
|---|---|
| `SeqScan(table)` / `IndexScan(table idx)` | บังคับวิธี scan ของตารางนั้น |
| `NestLoop(t1 t2)` / `HashJoin(t1 t2)` / `MergeJoin(t1 t2)` | บังคับวิธี join ระหว่างสองตาราง |
| `Leading((t1 t2 t3))` | บังคับลำดับ join |
| `Rows(t1 t2 #500)` | บังคับ row estimate ของผลลัพธ์การ join (ทางเลือกแทนการแก้ statistics) |
| `Set(work_mem '256MB')` | ปรับ GUC เฉพาะ query นั้น |

> **ข้อควรระวังอย่างยิ่ง (ต้องอ่านก่อนใช้จริง):**
> 1. **ใช้เป็นทางเลือกสุดท้ายเท่านั้น** — ลองแก้ index, statistics, และ query structure ให้หมดก่อน
> 2. Hint ที่ตายตัวจะ**ไม่ปรับตามข้อมูลที่เปลี่ยนไป** — วันนี้ table เล็ก nested loop เร็วที่สุด แต่ปีหน้า table โตขึ้น 100 เท่า nested loop อาจกลายเป็นตัวช้าที่สุด แต่ hint จะยังบังคับให้ใช้ nested loop ต่อไปเพราะไม่มีใครกลับมาแก้
> 3. เพิ่ม **dependency ต่อ extension ภายนอก** ที่ต้องติดตั้งทุก environment (dev/staging/prod) — เพิ่มความซับซ้อนในการ deploy และ upgrade PostgreSQL เวอร์ชันใหม่ (pg_hint_plan ต้อง release เวอร์ชันที่รองรับ PG version นั้นก่อน)
> 4. เวลามี hint ปนอยู่ใน query จำนวนมาก การ debug ว่า "ทำไม planner เลือกแผนนี้" จะซับซ้อนขึ้นอีกชั้น (ต้องดูทั้ง statistics และ hint พร้อมกัน)
> 5. ทางเลือกที่ยั่งยืนกว่ามักเป็น: แก้ statistics (Step 732), เพิ่ม index ที่เหมาะสม, เขียน query ใหม่ให้ planner ตัดสินใจถูกเองตามธรรมชาติ, หรือถ้าจำเป็นจริงๆ ให้ hint แบบ **scope แคบที่สุดเท่าที่ทำได้** (session-level `SET` ชั่วคราวรอบ query เดียว ดีกว่า hint ที่ฝังถาวรใน production code)

---

## Step 734: Parallel Query เจาะลึก

PostgreSQL จะพิจารณาใช้ **parallel query** โดยอัตโนมัติเมื่อครบเงื่อนไขต่อไปนี้พร้อมกัน:

1. `max_parallel_workers_per_gather > 0` (default = 2)
2. Query ไม่ใช่ `INSERT`/`UPDATE`/`DELETE` ที่แก้ไขข้อมูล (ยกเว้น `INSERT ... SELECT` บางรูปแบบใน PG 17+ ที่ SELECT ส่วนสามารถ parallel ได้)
3. ตารางที่ scan **มีขนาดใหญ่กว่า `min_parallel_table_scan_size`** (default 8MB) หรือ index ใหญ่กว่า `min_parallel_index_scan_size` (default 512kB)
4. Planner ประเมินว่า**ต้นทุนของงานที่ทำคุ้มกับ overhead การเปิด worker process** ซึ่งควบคุมด้วย `parallel_setup_cost` (default 1000) และ `parallel_tuple_cost` (default 0.1 ต่อแถวที่ส่งผ่าน worker)
5. ไม่มี operation ที่ parallel ไม่ได้ เช่น window function บางแบบ, cursor `WITH HOLD`, function ที่ประกาศ `PARALLEL UNSAFE`

### ตัวอย่างจริง: query ที่เล็กเกินไปจะไม่ parallel

```sql
SHOW max_parallel_workers_per_gather;  -- 2
SHOW min_parallel_table_scan_size;     -- 8MB

SELECT pg_size_pretty(pg_relation_size('order_items'));  -- 6120 kB (< 8MB threshold!)
```

เนื่องจาก `order_items` (6.1MB) ยังเล็กกว่า threshold 8MB แม้จะมี 120,000 แถว query ก็ยังไม่ parallel ตาม default นี่คือกับดักที่คนมักงงว่า "ทำไม query ไม่ parallel ทั้งที่ตารางก็ไม่เล็ก" — คำตอบคือ threshold คำนวณจาก**ขนาดหน่วยความจำ (byte) ไม่ใช่จำนวนแถว**

ลด threshold ลง (เฉพาะ session นี้ เพื่อสาธิต — ใน production ควรปรับที่ `postgresql.conf` หากต้องการถาวร):

```sql
SET min_parallel_table_scan_size = 0;
SET parallel_setup_cost = 0;
SET parallel_tuple_cost = 0;

EXPLAIN ANALYZE
SELECT status, count(*), sum(o.order_id)
FROM orders o
JOIN order_items oi ON oi.order_id = o.order_id
GROUP BY status;

RESET min_parallel_table_scan_size;
RESET parallel_setup_cost;
RESET parallel_tuple_cost;
```

```
Finalize GroupAggregate  (cost=2423.40..2423.64 rows=5 width=24) (actual time=27.893..30.792 rows=5 loops=1)
  Group Key: o.status
  ->  Gather Merge  (cost=2423.40..2423.51 rows=10 width=24) (actual time=27.886..30.782 rows=15 loops=1)
        Workers Planned: 2
        Workers Launched: 2
        ->  Sort  (cost=2423.37..2423.39 rows=5 width=24) (actual time=25.408..25.412 rows=5 loops=3)
              Sort Key: o.status
              Sort Method: quicksort  Memory: 25kB
              Worker 0:  Sort Method: quicksort  Memory: 25kB
              Worker 1:  Sort Method: quicksort  Memory: 25kB
              ->  Partial HashAggregate  (cost=2423.27..2423.32 rows=5 width=24) (actual time=25.384..25.387 rows=5 loops=3)
                    Group Key: o.status
                    ->  Parallel Hash Join  (cost=652.00..2048.27 rows=50000 width=12) (actual time=5.941..18.187 rows=40000 loops=3)
                          Hash Cond: (oi.order_id = o.order_id)
                          ->  Parallel Seq Scan on order_items oi  (cost=0.00..1265.00 rows=50000 width=4) (actual time=0.006..2.795 rows=40000 loops=3)
                          ->  Parallel Hash  (cost=443.67..443.67 rows=16667 width=12) (actual time=5.729..5.730 rows=13333 loops=3)
                                ->  Parallel Seq Scan on orders o  (cost=0.00..443.67 rows=16667 width=12) (actual time=0.008..2.009 rows=13333 loops=3)
Execution Time: 30.955 ms
```

เทียบกับแผนแบบ non-parallel (default settings) ของ query เดียวกัน ซึ่งใช้เวลา **57.612 ms** — parallel plan เร็วขึ้นเกือบ 2 เท่าด้วย worker 2 ตัว

### วิธีอ่านโครงสร้าง Gather / Gather Merge

```
Finalize GroupAggregate          <- รวมผลจาก worker ทุกตัว (ทำใน leader process)
  -> Gather Merge                <- รวบรวมผลลัพธ์ที่ "เรียงลำดับแล้ว" จาก worker (ใช้เมื่อต้องการ order)
       Workers Planned: 2        <- planner วางแผนใช้ 2 worker
       Workers Launched: 2       <- ต้องดูว่าตรงกับ Planned หรือไม่! ถ้าน้อยกว่าแปลว่า worker ไม่พอ
       -> Sort (ต่อ worker)
            -> Partial HashAggregate (ต่อ worker)  <- แต่ละ worker รวมผลของตัวเองก่อนส่งกลับ
                 -> Parallel Hash Join
                      -> Parallel Seq Scan          <- แต่ละ worker scan คนละส่วนของตาราง
```

จุดสำคัญที่ต้องดูเสมอ:

- **`Workers Planned` vs `Workers Launched`** — ถ้าต่างกัน (เช่น Planned 4 แต่ Launched 2) แปลว่าระบบมี worker process ไม่พอ (`max_worker_processes` เต็ม เพราะถูก query อื่นแย่งไปใช้) ทำให้ query ทำงานช้ากว่าที่ planner คาดไว้
- **`loops=3`** ใน node ที่อยู่ใต้ Gather หมายถึง node นั้นทำงาน 3 ครั้ง (leader + worker 2 ตัว) ตัวเลข `actual rows` ที่แสดงเป็น**ค่าเฉลี่ยต่อ loop** ต้องคูณด้วย loops คร่าวๆ เพื่อดูยอดรวมจริง
- **`Gather`** (ไม่มี Merge) ใช้เมื่อไม่ต้องรักษาลำดับ — เร็วกว่าเพราะไม่ต้อง merge-sort ผลลัพธ์
- **`Gather Merge`** ใช้เมื่อผลลัพธ์จาก worker แต่ละตัวเรียงลำดับมาแล้ว (worker ทำ partial sort ของตัวเอง) และต้องการรวมแบบรักษา order (เช่นก่อน aggregate ที่ใช้ `GROUP BY`)

ตัวอย่าง `Gather` แบบง่าย (ไม่มี merge):

```sql
SET min_parallel_table_scan_size = 0;
EXPLAIN ANALYZE
SELECT count(*) FROM order_items WHERE quantity >= 4;
RESET min_parallel_table_scan_size;
```

```
Finalize Aggregate  (cost=1439.76..1439.77 rows=1 width=8) (actual time=8.532..10.535 rows=1 loops=1)
  ->  Gather  (cost=1439.74..1439.75 rows=2 width=8) (actual time=8.521..10.527 rows=3 loops=1)
        Workers Planned: 2
        Workers Launched: 2
        ->  Partial Aggregate  (cost=1439.74..1439.75 rows=1 width=8) (actual time=5.128..5.129 rows=1 loops=3)
              ->  Parallel Seq Scan on order_items  (cost=0.00..1390.00 rows=19898 width=0) (actual time=0.010..4.556 rows=15979 loops=3)
                    Filter: (quantity >= 4)
                    Rows Removed by Filter: 24021
Execution Time: 10.630 ms
```

### เมื่อไหร่ parallel query "ไม่คุ้ม" และควรปิด

- Query ที่ return ผลลัพธ์จำนวนมาก (`parallel_tuple_cost` จะสูงตาม เพราะต้องส่งข้อมูลผ่าน IPC จาก worker กลับ leader)
- Query ที่ทำงานบนตารางเล็ก (overhead การ spawn process มากกว่าประโยชน์)
- ระบบที่มี concurrent connection จำนวนมากอยู่แล้ว — parallel query แต่ละตัวใช้ CPU core เพิ่ม ถ้าระบบมี query พร้อมกันมากอยู่แล้ว การให้แต่ละ query ใช้ 2-4 core อาจทำให้ CPU ทั้งระบบ contention กันเอง (ควรปรับ `max_parallel_workers` ทั้ง cluster ให้เหมาะสม ไม่ใช่แค่ `_per_gather`)

ปิด parallel เฉพาะ query:

```sql
SET LOCAL max_parallel_workers_per_gather = 0;
```

หรือปิดเฉพาะ table/function:

```sql
ALTER TABLE order_items SET (parallel_workers = 0);
```

---

## Step 735: CTE Materialization กับผลกระทบต่อ Performance

Part 025 แนะนำ CTE (`WITH ... AS`) เบื้องต้นไปแล้ว บทนี้จะดูผลกระทบจริงต่อ performance ในสถานการณ์ query ซับซ้อน

### พฤติกรรม default ใน PostgreSQL 12+

ตั้งแต่ PostgreSQL 12 เป็นต้นมา planner จะ**ตัดสินใจอัตโนมัติ**ว่าจะ materialize (คำนวณ CTE ล่วงหน้าแล้วเก็บผลไว้ชั่วคราว) หรือจะ **inline** (แทรก CTE เข้าไปในแผนหลักเหมือนเป็น subquery ธรรมดา) โดยกฎคร่าวๆ คือ:

- **CTE ที่ถูกอ้างอิงครั้งเดียว** → มักถูก inline อัตโนมัติ (เหมือนไม่มี CTE เลย planner มองทะลุเข้าไปได้)
- **CTE ที่ถูกอ้างอิงมากกว่าหนึ่งครั้ง** → มักถูก materialize อัตโนมัติ (คำนวณครั้งเดียว เก็บผลไว้ใช้ซ้ำ)

คุณสามารถบังคับพฤติกรรมได้ตรงๆ ด้วย `MATERIALIZED` / `NOT MATERIALIZED`

### กรณีที่ 1: CTE อ้างอิงครั้งเดียว — ถูก inline อัตโนมัติ

```sql
EXPLAIN ANALYZE
WITH high_value_orders AS (
    SELECT o.order_id, o.customer_id, sum(oi.quantity * oi.unit_price) AS order_total
    FROM orders o
    JOIN order_items oi ON oi.order_id = o.order_id
    GROUP BY o.order_id, o.customer_id
)
SELECT customer_id, order_total
FROM high_value_orders
WHERE order_total > 2000
ORDER BY order_total DESC
LIMIT 10;
```

```
Limit  (cost=10934.82..10934.85 rows=10 width=36) (actual time=147.563..147.567 rows=10 loops=1)
  ->  Sort  (cost=10934.82..10968.16 rows=13333 width=36) (actual time=147.562..147.564 rows=10 loops=1)
        Sort Key: high_value_orders.order_total DESC
        ->  Subquery Scan on high_value_orders  (cost=0.58..10646.70 rows=13333 width=36) (actual time=0.060..142.005 rows=29834 loops=1)
              ->  GroupAggregate  (cost=0.58..10513.37 rows=13333 width=40) (actual time=0.059..139.804 rows=29834 loops=1)
                    Group Key: o.order_id
                    Filter: (sum(((oi.quantity)::numeric * oi.unit_price)) > '2000'::numeric)
                    Rows Removed by Filter: 8132
                    ->  Merge Join  (cost=0.58..8713.37 rows=120000 width=18) (actual time=0.026..87.319 rows=120000 loops=1)
                          Merge Cond: (o.order_id = oi.order_id)
Execution Time: 147.654 ms
```

สังเกตว่า**ไม่มี node ชื่อ "CTE Scan" เลย** — planner แปลงเป็น `Subquery Scan` และ push เงื่อนไข `order_total > 2000` เข้าไปเป็น `Filter` ในขั้น `GroupAggregate` ได้เลย (แม้จะยังต้อง aggregate ครบทุกกลุ่มก่อน filter เพราะเป็น aggregate filter ไม่ใช่ filter บนคอลัมน์ธรรมดา) นี่คือพฤติกรรม inline อัตโนมัติ

### กรณีที่ 2: CTE อ้างอิงสองครั้ง — ถูก materialize อัตโนมัติ

```sql
EXPLAIN ANALYZE
WITH order_totals AS (
    SELECT o.order_id, o.customer_id, sum(oi.quantity * oi.unit_price) AS order_total
    FROM orders o
    JOIN order_items oi ON oi.order_id = o.order_id
    GROUP BY o.order_id, o.customer_id
)
SELECT
    (SELECT count(*) FROM order_totals WHERE order_total > 2000) AS high_value_count,
    (SELECT count(*) FROM order_totals WHERE order_total <= 2000) AS low_value_count;
```

```
Result  (cost=12280.06..12280.07 rows=1 width=16) (actual time=150.694..150.703 rows=1 loops=1)
  CTE order_totals
    ->  GroupAggregate  (cost=0.58..10413.37 rows=40000 width=40) (actual time=0.063..132.075 rows=37966 loops=1)
          ->  Merge Join  (cost=0.58..8713.37 rows=120000 width=18) (actual time=0.033..84.552 rows=120000 loops=1)
                Merge Cond: (o.order_id = oi.order_id)
  InitPlan 2 (returns $1)
    ->  Aggregate  (cost=933.33..933.34 rows=1 width=8) (actual time=146.710..146.715 rows=1 loops=1)
          ->  CTE Scan on order_totals  (cost=0.00..900.00 rows=13333 width=0) (actual time=0.073..144.737 rows=29834 loops=1)
                Filter: (order_total > '2000'::numeric)
  InitPlan 3 (returns $2)
    ->  Aggregate  (cost=933.33..933.34 rows=1 width=8) (actual time=3.976..3.977 rows=1 loops=1)
          ->  CTE Scan on order_totals order_totals_1  (cost=0.00..900.00 rows=13333 width=0) (actual time=0.003..3.664 rows=8132 loops=1)
                Filter: (order_total <= '2000'::numeric)
Execution Time: 151.166 ms
```

คราวนี้เห็น **`CTE order_totals`** คำนวณครั้งเดียว (132ms) แล้วมี **`CTE Scan on order_totals`** ปรากฏ 2 ครั้งข้างล่าง (ใช้ผลลัพธ์ที่เก็บไว้ซ้ำ) — นี่คือ materialization ทำงาน: ทำ join + aggregate ที่แพงที่สุดแค่ครั้งเดียว แล้วสแกนผลลัพธ์ที่เก็บไว้ (cheap) ซ้ำสองครั้ง

### ทดสอบบังคับ NOT MATERIALIZED เพื่อดูผลเสีย

```sql
EXPLAIN ANALYZE
WITH order_totals AS NOT MATERIALIZED (
    SELECT o.order_id, o.customer_id, sum(oi.quantity * oi.unit_price) AS order_total
    FROM orders o
    JOIN order_items oi ON oi.order_id = o.order_id
    GROUP BY o.order_id, o.customer_id
)
SELECT
    (SELECT count(*) FROM order_totals WHERE order_total > 2000) AS high_value_count,
    (SELECT count(*) FROM order_totals WHERE order_total <= 2000) AS low_value_count;
```

```
Result  (cost=21360.09..21360.10 rows=1 width=16) (actual time=254.457..254.462 rows=1 loops=1)
  InitPlan 1 (returns $0)
    ->  Aggregate  (cost=10680.04..10680.05 rows=1 width=8) (actual time=131.347..131.350 rows=1 loops=1)
          ->  GroupAggregate  (... เต็ม Merge Join + GroupAggregate ทั้งชุดอีกครั้ง ...)
  InitPlan 2 (returns $1)
    ->  Aggregate  (cost=10680.04..10680.05 rows=1 width=8) (actual time=123.102..123.103 rows=1 loops=1)
          ->  GroupAggregate  (... ทำ Merge Join + GroupAggregate ซ้ำเป็นรอบที่สอง ...)
Execution Time: 254.637 ms
```

**254.637 ms เทียบกับ 151.166 ms — ช้าลงเกือบ 1.7 เท่า** เพราะ `NOT MATERIALIZED` บังคับให้ inline CTE เข้าไปทุกจุดที่อ้างอิง ทำให้ Merge Join + GroupAggregate ที่แพง (~130ms) ถูกคำนวณซ้ำถึง 2 รอบแทนที่จะเป็นรอบเดียว

### สรุปกฎการตัดสินใจ

| สถานการณ์ | ควรใช้ |
|---|---|
| CTE คำนวณแพง (join/aggregate ใหญ่) และถูกอ้างอิงหลายครั้ง | `MATERIALIZED` (หรือปล่อย default ซึ่งมักเลือกแบบนี้ให้อยู่แล้ว) |
| CTE ง่าย/ถูก และมีเงื่อนไข filter ที่อยากให้ push ลงไปถึงตารางจริง (index pushdown) แต่ CTE นั้น "บัง" ไม่ให้ planner มองทะลุ | `NOT MATERIALIZED` (บังคับ inline เพื่อให้ optimizer มองเห็นและ optimize ข้ามขอบเขต CTE ได้) |
| ไม่แน่ใจ | ปล่อย default แล้ว**เช็ค EXPLAIN ว่าเห็น node ไหน** — ถ้าเห็น `CTE Scan` แปลว่า materialize แล้ว ถ้าเห็น query ถูก flatten เข้าแผนหลักแปลว่า inline แล้ว |

> เชื่อมโยง Part 025: เนื้อหาพื้นฐานอธิบายว่า CTE "optimization fence" หายไปตั้งแต่ PG12 บทนี้แสดงให้เห็นว่าแม้ planner จะฉลาดขึ้นและเลือกให้อัตโนมัติได้ดีในกรณีส่วนใหญ่ แต่ในกรณีขอบ (edge case) ที่ query ซับซ้อนมาก การรู้จักบังคับด้วยตัวเองและอ่าน `EXPLAIN` เพื่อยืนยันพฤติกรรมจริง ยังเป็นทักษะที่จำเป็นสำหรับ query ระดับ production

---

## Step 736: Optimize Batch Operation ขนาดใหญ่ — UNNEST Array Parameter Pattern

ปัญหาคลาสสิกของแอปพลิเคชัน: ต้อง insert ข้อมูล 5,000 แถวจากฝั่ง client แล้วเขียนโค้ดวนลูป `INSERT` ทีละแถว ซึ่งแต่ละแถวเป็น**การ round-trip ระหว่าง client กับ database แยกกัน** — ต้นทุนที่แท้จริงไม่ได้อยู่ที่ INSERT เอง แต่อยู่ที่ network latency, parse/plan overhead, และ (ถ้าไม่ได้ wrap ใน transaction เดียว) WAL fsync ต่อ statement

### เปรียบเทียบจริง: Naive loop vs UNNEST bulk pattern

**แบบ naive** — 5,000 statement แยกกัน (auto-commit ทีละคำสั่ง จำลองพฤติกรรมแอปทั่วไปที่ไม่ได้ wrap transaction):

```sql
-- ตัวอย่าง (จำลองจาก client): 5,000 statements แยกกัน ไม่มี transaction ครอบ
INSERT INTO bulk_test2 (val, note) VALUES (1, 'row 1');
INSERT INTO bulk_test2 (val, note) VALUES (2, 'row 2');
-- ... ซ้ำแบบนี้ 5,000 ครั้ง ...
```

ผลวัดจริง (รันผ่าน `psql` เป็น 5,000 statement แยก, local socket ไม่มี network latency จริงด้วยซ้ำ): **1.727 วินาที**

**แบบ UNNEST bulk insert** — statement เดียว ส่ง array parameter เข้าไป:

```sql
INSERT INTO bulk_test2 (val, note)
SELECT v, 'row ' || v
FROM unnest(array(SELECT generate_series(1, 5000))) AS v;
```

ผลวัดจริง: **0.064 วินาที**

**เร็วขึ้น ~27 เท่า แม้จะทดสอบบน local socket ที่แทบไม่มี network latency** — ในสภาพแวดล้อมจริงที่ client กับ database อยู่คนละเครื่อง (round-trip latency 1-5ms ต่อครั้ง) ความต่างจะยิ่งมหาศาลกว่านี้มาก (5,000 × 2ms = 10 วินาที เฉพาะ network overhead)

### Pattern ที่ใช้จริงในแอปพลิเคชัน (หลาย array parameter พร้อมกัน)

รูปแบบที่ driver ส่วนใหญ่ (เช่น `pgx`, `npgsql`, JDBC ผ่าน `unnest`) ใช้ในการ bind parameter เป็น array หลายชุดพร้อมกัน:

```sql
EXPLAIN ANALYZE
INSERT INTO bulk_test2 (val, note)
SELECT * FROM unnest(
    ARRAY[1,2,3,4,5]::int[],
    ARRAY['row 1','row 2','row 3','row 4','row 5']::text[]
) AS t(val, note);
```

```
Insert on bulk_test2  (cost=0.01..0.08 rows=0 width=0) (actual time=0.245..0.246 rows=0 loops=1)
  ->  Function Scan on t  (cost=0.01..0.08 rows=5 width=40) (actual time=0.099..0.103 rows=5 loops=1)
Execution Time: 0.269 ms
```

ในภาษาโปรแกรม (ตัวอย่างแนวคิดแบบ Python + psycopg):

```python
ids = [1, 2, 3, 4, 5]
notes = ["row 1", "row 2", "row 3", "row 4", "row 5"]

cur.execute("""
    INSERT INTO bulk_test2 (val, note)
    SELECT * FROM unnest(%s::int[], %s::text[]) AS t(val, note)
""", (ids, notes))
```

**ข้อดี**: ส่ง 1 round-trip แทน N round-trip, planner วาง plan แค่ครั้งเดียว, PostgreSQL จัดการ batch การเขียน WAL ให้มีประสิทธิภาพกว่า

### Bulk UPDATE ด้วย pattern เดียวกัน (join กับ temp table หรือ UNNEST โดยตรง)

สถานการณ์จริง: sync ราคาสินค้า 500 รายการจากระบบภายนอกเข้า `products` พร้อมกัน

```sql
-- วิธีที่ 1: ผ่าน temp table (เหมาะกับข้อมูลจำนวนมากมาก เพราะ planner ใช้ hash/merge join ได้)
CREATE TEMP TABLE price_updates (product_id int, new_price numeric);
INSERT INTO price_updates
SELECT product_id, round((random()*500+10)::numeric,2)
FROM products
ORDER BY random()
LIMIT 500;

EXPLAIN ANALYZE
UPDATE products p
SET unit_price = pu.new_price
FROM price_updates pu
WHERE p.product_id = pu.product_id;
```

```
Update on products p  (cost=155.50..184.71 rows=0 width=0) (actual time=3.034..3.036 rows=0 loops=1)
  ->  Hash Join  (cost=155.50..184.71 rows=1270 width=28) (actual time=1.014..1.226 rows=500 loops=1)
        Hash Cond: (pu.product_id = p.product_id)
        ->  Seq Scan on price_updates pu  (cost=0.00..22.70 rows=1270 width=42) (actual time=0.007..0.068 rows=500 loops=1)
        ->  Hash  (cost=93.00..93.00 rows=5000 width=10) (actual time=0.989..0.989 rows=5000 loops=1)
              ->  Seq Scan on products p  (cost=0.00..93.00 rows=5000 width=10) (actual time=0.004..0.445 rows=5000 loops=1)
Execution Time: 3.079 ms
```

```sql
-- วิธีที่ 2: UNNEST ตรงๆ ไม่ต้องสร้าง temp table (เหมาะกับ batch ขนาดกลาง ส่งจาก application ครั้งเดียว)
UPDATE products p
SET unit_price = v.new_price
FROM (
    SELECT * FROM unnest(
        ARRAY[1, 2, 3]::int[],
        ARRAY[199.00, 299.00, 399.00]::numeric[]
    ) AS t(product_id, new_price)
) v
WHERE p.product_id = v.product_id;
```

**500 แถวอัปเดตใน 3.079ms ด้วย statement เดียว** เทียบกับ 500 statement `UPDATE ... WHERE product_id = $1` แยกกันซึ่งต้องมี 500 round-trip — pattern นี้ควรเป็น **default choice** สำหรับ batch operation ทุกครั้งที่ทำได้ ไม่ใช่แค่ทางเลือกเมื่อเจอปัญหา performance

> **ข้อควรระวัง**: การ `UPDATE` แบบ batch ขนาดใหญ่มากในครั้งเดียว (เช่น 1 ล้านแถว) อาจสร้าง lock ยาวนานและ WAL จำนวนมหาศาลในครั้งเดียว ควรพิจารณาแบ่งเป็น chunk (เช่น ครั้งละ 10,000-50,000 แถว) ผ่าน loop ที่มี `COMMIT` คั่นระหว่างแต่ละ chunk เพื่อไม่ให้ transaction ค้างนานเกินไปและ block query อื่น

---

## Step 737: การวินิจฉัย Lock Contention ด้วย pg_locks และ pg_stat_activity

Query ที่รันมาปกติทุกวัน จู่ๆ "ค้าง" ไม่ตอบสนอง มักไม่ใช่ปัญหาที่ query plan เลย แต่เป็นเพราะ**ถูก block ด้วย lock จาก transaction อื่น**

### จำลองสถานการณ์จริง: Session A ถือ lock, Session B ต้องรอ

**Session A** (เริ่ม transaction แล้วยังไม่ commit):

```sql
BEGIN;
UPDATE products SET stock_quantity = stock_quantity - 1 WHERE product_id = 100;
-- ... ค้างอยู่ตรงนี้ ยังไม่ COMMIT ...
```

**Session B** (พยายามแก้ไข row เดียวกัน):

```sql
UPDATE products SET stock_quantity = stock_quantity - 1 WHERE product_id = 100;
-- Session นี้จะ "ค้าง" รอจนกว่า Session A จะ COMMIT หรือ ROLLBACK
```

### วิธีที่ 1: หา blocking session แบบรวดเร็วด้วย pg_blocking_pids()

```sql
SELECT
    pid,
    state,
    wait_event_type,
    wait_event,
    pg_blocking_pids(pid) AS blocked_by,
    left(query, 60) AS query
FROM pg_stat_activity
WHERE datname = current_database() AND pid != pg_backend_pid()
ORDER BY pid;
```

ผลลัพธ์จริงระหว่างที่ Session B ถูก block:

```
 pid  | state  | wait_event_type |  wait_event   | blocked_by |                            query
------+--------+------------------+---------------+------------+--------------------------------------------------------------
 6605 | active | Timeout          | PgSleep       | {}         | BEGIN; UPDATE products SET stock_quantity = stock_quantity -
 6609 | active | Lock             | transactionid | {6605}     | UPDATE products SET stock_quantity = stock_quantity - 1 WHER
(2 rows)
```

อ่านผลได้ทันที: **PID 6609 กำลังรอ PID 6605** (`wait_event_type = Lock`, `wait_event = transactionid` หมายถึงรอ row lock ของอีก transaction ปล่อย) — `pg_blocking_pids()` เป็นฟังก์ชันสำเร็จรูปที่สะดวกที่สุดสำหรับ diagnosis เบื้องต้น

### วิธีที่ 2: Query ละเอียดผ่าน pg_locks join pg_stat_activity (เห็น query ทั้งสองฝั่งชัดเจน)

```sql
SELECT
    blocked_locks.pid       AS blocked_pid,
    blocked_activity.query  AS blocked_query,
    blocking_locks.pid      AS blocking_pid,
    blocking_activity.query AS blocking_query,
    blocked_activity.wait_event_type,
    now() - blocked_activity.query_start AS blocked_duration
FROM pg_locks blocked_locks
JOIN pg_stat_activity blocked_activity
    ON blocked_activity.pid = blocked_locks.pid
JOIN pg_locks blocking_locks
    ON blocking_locks.locktype = blocked_locks.locktype
   AND blocking_locks.database IS NOT DISTINCT FROM blocked_locks.database
   AND blocking_locks.relation IS NOT DISTINCT FROM blocked_locks.relation
   AND blocking_locks.page IS NOT DISTINCT FROM blocked_locks.page
   AND blocking_locks.tuple IS NOT DISTINCT FROM blocked_locks.tuple
   AND blocking_locks.pid != blocked_locks.pid
JOIN pg_stat_activity blocking_activity
    ON blocking_activity.pid = blocking_locks.pid
WHERE NOT blocked_locks.granted;
```

ผลลัพธ์จริง:

```
 blocked_pid |                                  blocked_query                                  | blocking_pid |                                                   blocking_query                                                   | wait_event_type | blocked_duration
-------------+-----------------------------------------------------------------------------------+---------------+----------------------------------------------------------------------------------------------------------------------+------------------+-------------------
        6592 | UPDATE products SET stock_quantity = stock_quantity - 1 WHERE product_id = 100; |          6588 | BEGIN; UPDATE products SET stock_quantity = stock_quantity - 1 WHERE product_id = 100; SELECT pg_sleep(6); COMMIT; | Lock             | 00:00:01.997871
```

Query นี้ให้ข้อมูลครบกว่า: เห็น **SQL text ของทั้งฝั่ง blocking และ blocked** พร้อม `blocked_duration` ที่บอกว่ารอมานานแค่ไหนแล้ว — มีประโยชน์มากเมื่อต้องตัดสินใจว่าจะ kill session ไหน

### การแก้ปัญหาเมื่อพบ blocking session

```sql
-- ยกเลิก query ที่กำลังรันอยู่ (soft — เหมือนกด Ctrl+C)
SELECT pg_cancel_backend(6588);

-- ตัดการเชื่อมต่อทั้ง session (hard — ใช้เมื่อ cancel ไม่ได้ผล)
SELECT pg_terminate_backend(6588);
```

> **ข้อควรระวัง**: `pg_terminate_backend` จะ rollback transaction ของ session นั้นทันที ถ้าเป็น session ของแอปพลิเคชันที่ยังทำงานอยู่ ควรแจ้งทีมที่เกี่ยวข้องก่อน และควรใช้เป็นทางแก้ปัญหาเฉพาะหน้า ไม่ใช่ทางแก้ถาวร — ต้นเหตุจริงมักมาจาก transaction ที่เปิดค้างไว้นานเกินไป (เช่น แอปเปิด transaction แล้วรอ user input, หรือ ORM ที่ลืมปิด connection) ซึ่งควรแก้ที่ต้นทาง

### Monitoring เชิงป้องกัน: หา transaction ที่เปิดค้างนานผิดปกติ

```sql
SELECT
    pid,
    usename,
    state,
    now() - xact_start AS transaction_age,
    left(query, 80) AS query
FROM pg_stat_activity
WHERE state != 'idle'
  AND xact_start IS NOT NULL
  AND now() - xact_start > interval '5 minutes'
ORDER BY xact_start;
```

ควรตั้งเป็น monitoring query ที่รันเป็น cron/alerting job เพื่อจับ transaction ค้างก่อนที่จะกลายเป็นปัญหา lock contention ลูกโซ่

---

## Step 738: Query Timeout และ statement_timeout เพื่อป้องกัน Query ที่หลุดควบคุม

Query ที่เขียนผิดพลาด (เช่น join แบบ cartesian product โดยไม่ตั้งใจ) หรือ query ที่ปกติเร็วแต่เจอ lock ค้างเป็นชั่วโมง สามารถทำให้ connection pool เต็ม และทั้งระบบล่มตามได้ `statement_timeout` คือเครื่องมือป้องกันพื้นฐานที่สุด

### ทดสอบจริง

```sql
SET statement_timeout = '100ms';

SELECT count(*)
FROM orders o, order_items oi, order_items oi2
WHERE o.order_id = oi.order_id;

RESET statement_timeout;
```

ผลลัพธ์จริง:

```
SET
ERROR:  canceling statement due to statement timeout
RESET
```

Query ที่เขียนผิด (ลืมเงื่อนไข join กับ `oi2` ทำให้เกิด cartesian product มหาศาล) ถูกยกเลิกอัตโนมัติหลัง 100ms แทนที่จะรันค้างจนกิน CPU/memory ทั้งเครื่อง

### ระดับการตั้งค่า statement_timeout

```sql
-- 1) ระดับ session เดียว (ชั่วคราว จนกว่า session จะปิด)
SET statement_timeout = '30s';

-- 2) ระดับ transaction เดียว (กลับเป็นค่าเดิมทันทีที่ COMMIT/ROLLBACK)
BEGIN;
SET LOCAL statement_timeout = '5s';
-- ... query ...
COMMIT;

-- 3) ระดับ role (ถาวร ใช้ทุกครั้งที่ role นี้ login)
ALTER ROLE reporting_user SET statement_timeout = '60s';

-- 4) ระดับ database (ถาวร ใช้กับทุกคนที่เชื่อมต่อ database นี้)
ALTER DATABASE ecommerce SET statement_timeout = '30s';

-- 5) ระดับ cluster ทั้งหมด (postgresql.conf)
-- statement_timeout = '30s'   -- ค่า default คือ 0 (ไม่มี timeout)
```

### กลยุทธ์การตั้งค่าตามประเภทงาน

| ประเภท workload | statement_timeout แนะนำ |
|---|---|
| Web application (OLTP, query ต้องเร็ว) | 5-15 วินาที |
| Reporting / analytics user | 1-5 นาที (หรือแยก role ต่างหากพร้อม `work_mem` สูงกว่า) |
| Batch job / ETL | ปิด (`0`) ที่ระดับ role นั้น แต่ควบคุมด้วยกลไกอื่น เช่น application-level timeout |
| Migration script | ปิดชั่วคราวเฉพาะ session ที่รัน migration |

### GUC ที่เกี่ยวข้องอื่นๆ ที่ควรรู้จักคู่กัน

```sql
-- จำกัดเวลารอ lock (แยกจาก statement_timeout — ป้องกัน query ค้างรอ lock นานเกินไป)
SET lock_timeout = '3s';

-- จำกัดเวลารอทั้ง idle-in-transaction (connection ที่เปิด transaction แล้วไม่ทำอะไรต่อ)
SET idle_in_transaction_session_timeout = '10min';

-- (PostgreSQL 14+) จำกัดเวลา idle ของ session ทั้งหมด ไม่ว่าจะอยู่ใน transaction หรือไม่
SET idle_session_timeout = '30min';
```

`idle_in_transaction_session_timeout` สำคัญมากเป็นพิเศษ เพราะเป็นสาเหตุอันดับต้นๆ ของ lock contention ที่เจอใน Step 737 — แอปพลิเคชันที่เปิด transaction แล้ว "ลืม" commit (เช่น เปิด transaction รอ user กด submit form) จะถือ lock ค้างไว้เรื่อยๆ การตั้ง timeout นี้จะบังคับตัดการเชื่อมต่อและ rollback อัตโนมัติ ป้องกันปัญหาลูกโซ่

---

## Step 739: A/B Testing Query Performance ด้วย pgbench

การเปรียบเทียบ query สองแบบด้วยการรัน `EXPLAIN ANALYZE` ครั้งเดียวอาจให้ผลที่คลาดเคลื่อนจาก noise (cache state, CPU scheduling ชั่วขณะ) วิธีที่น่าเชื่อถือกว่าคือใช้ `pgbench` รันซ้ำหลายพันครั้งด้วยหลาย concurrent connection แล้วเทียบ throughput (TPS)

> บทนี้เกริ่นแนวคิดพื้นฐาน — เนื้อหาการทำ benchmark เชิงลึกแบบเต็มรูปแบบ (custom scripts, scale factor design, การวิเคราะห์ latency distribution) จะอยู่ใน **Part 098**

### สร้าง script เปรียบเทียบ 2 รูปแบบของ query เดียวกัน

**Variant A** — ใช้เงื่อนไขที่ index รองรับได้ตรงๆ:

```sql
-- variant_a.sql
SELECT count(*) FROM orders WHERE status = 'paid';
```

**Variant B** — ใช้ฟังก์ชันครอบคอลัมน์ (`lower()`) ซึ่งทำให้ index ธรรมดาบน `status` ใช้ไม่ได้ (ต้องมี expression index ถึงจะช่วย):

```sql
-- variant_b.sql
SELECT count(*) FROM orders WHERE lower(status) = 'paid';
```

### รันเปรียบเทียบด้วย pgbench

```bash
# Variant A: 4 concurrent clients, 2 threads, รันนาน 5 วินาที
pgbench -d ecommerce -f variant_a.sql -T 5 -c 4 -j 2 --no-vacuum

# Variant B: เงื่อนไขเดียวกันทุกอย่าง เปลี่ยนแค่ query
pgbench -d ecommerce -f variant_b.sql -T 5 -c 4 -j 2 --no-vacuum
```

ผลลัพธ์จริงที่วัดได้:

**Variant A** (ใช้ index `idx_orders_status`):

```
transaction type: variant_a.sql
scaling factor: 1
query mode: simple
number of clients: 4
number of threads: 2
duration: 5 s
number of transactions actually processed: 15755
latency average = 1.268 ms
tps = 3153.370704 (without initial connection time)
```

**Variant B** (`lower(status)` — บังคับ seq scan เพราะไม่มี expression index):

```
transaction type: variant_b.sql
scaling factor: 1
query mode: simple
number of clients: 4
number of threads: 2
duration: 5 s
number of transactions actually processed: 1896
latency average = 10.552 ms
tps = 379.074147 (without initial connection time)
```

**สรุปผล A/B test: Variant A เร็วกว่า Variant B ประมาณ 8.3 เท่า** (3153 TPS เทียบกับ 379 TPS) — ตัวเลขนี้น่าเชื่อถือกว่าการรัน `EXPLAIN ANALYZE` ครั้งเดียวมาก เพราะมาจากการรันซ้ำหลายพันครั้งภายใต้ concurrent load จริง ซึ่งสะท้อนสภาพ production ได้ดีกว่า

### แก้ปัญหา Variant B ด้วย expression index (bonus)

```sql
CREATE INDEX idx_orders_lower_status ON orders (lower(status));
```

หลังจากนี้ `lower(status) = 'paid'` จะสามารถใช้ index ได้ ทำให้ throughput กลับมาใกล้เคียง Variant A — เป็นตัวอย่างที่ดีว่า A/B testing ไม่ได้มีไว้แค่ "เลือกว่าแบบไหนดีกว่า" แต่ยังช่วย**พิสูจน์ว่าการแก้ไข (fix) ที่เราทำนั้นได้ผลจริงเชิงปริมาณ** ไม่ใช่แค่ความรู้สึก

### หลักปฏิบัติสำหรับ pgbench A/B testing

1. ใช้ `--no-vacuum` เมื่อต้องการวัด query performance ล้วนๆ โดยไม่ปน overhead ของ auto-vacuum
2. รัน warm-up ก่อนอย่างน้อย 1 รอบสั้นๆ เพื่อให้ cache อยู่ในสภาพเดียวกันก่อนวัดจริง (buffer cache state ที่ต่างกันทำให้ผลเทียบกันไม่ยุติธรรม)
3. รันทั้งสอง variant บนข้อมูลชุดเดียวกัน เครื่องเดียวกัน ในเวลาใกล้กัน (หลีกเลี่ยง noise จากระบบอื่นที่แย่ง resource)
4. ดูทั้ง `tps` และ `latency average` — บาง workload สนใจ throughput สูงสุด บาง workload สนใจ latency ต่ำสุดต่อ request (มักสำคัญกว่าสำหรับ user-facing query)
5. รันซ้ำอย่างน้อย 3 รอบต่อ variant เพื่อดูความแปรปรวน (variance) ไม่ใช่เชื่อผลจากรอบเดียว

---

## Step 740: แบบฝึกหัดรวม — วินิจฉัยและแก้ไข Production Incident จำลอง

**สถานการณ์**: ทีม on-call ได้รับแจ้งเตือนว่า dashboard รายงานยอดขายรายสินค้า (query ที่เคยใช้เวลาประมาณ 300-350ms) จู่ๆ ช้าลงเป็น 600ms+ ในช่วง 2-3 เดือนที่ผ่านมา โดยไม่มีการแก้โค้ดใดๆ เลย

Query ที่มีปัญหา:

```sql
SELECT oi.product_id, count(*), sum(oi.quantity * oi.unit_price) AS total
FROM order_items oi
JOIN orders o ON o.order_id = oi.order_id
GROUP BY oi.product_id;
```

### ขั้นตอนที่ 1: ยืนยันปัญหาด้วย EXPLAIN (ANALYZE, BUFFERS)

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT oi.product_id, count(*), sum(oi.quantity * oi.unit_price) AS total
FROM order_items oi
JOIN orders o ON o.order_id = oi.order_id
GROUP BY oi.product_id;
```

ผลลัพธ์จริง (หลังจากข้อมูล `order_items` เติบโตจาก 120,000 แถว เป็น 420,000 แถวในช่วงหลายเดือนที่ผ่านมา — ธุรกิจโตขึ้น มีออเดอร์เพิ่มขึ้นเรื่อยๆ ซึ่งเป็นเรื่องปกติ แต่ไม่มีใครกลับมาตรวจ performance อีกเลย):

```
HashAggregate  (cost=69254.11..77519.67 rows=4995 width=44) (actual time=436.770..635.727 rows=5000 loops=1)
  Group Key: oi.product_id
  Planned Partitions: 4  Batches: 85  Memory Usage: 181kB  Disk Usage: 15464kB
  ->  Hash Join  (cost=1334.00..14391.61 rows=420000 width=14) (actual time=7.655..265.970 rows=420000 loops=1)
        Hash Cond: (oi.order_id = o.order_id)
        ->  Seq Scan on order_items oi  (cost=0.00..6876.00 rows=420000 width=18) (actual time=0.008..43.108 rows=420000 loops=1)
        ->  Hash  (cost=677.00..677.00 rows=40000 width=4) (actual time=7.472..7.473 rows=40000 loops=1)
              Buckets: 4096  Batches: 32  Memory Usage: 77kB
              ->  Seq Scan on orders o  (cost=0.00..677.00 rows=40000 width=4) (actual time=0.004..3.233 rows=40000 loops=1)
Execution Time: 637.751 ms
```

### ขั้นตอนที่ 2: หา "จุดต้องสงสัย" ใน EXPLAIN output

ตัวชี้วัดสำคัญ 3 จุดที่โผล่มาให้เห็นทันที:

1. **`Batches: 85`** ที่ `HashAggregate` — ปกติ hash aggregate ที่ memory พอจะเป็น `Batches: 1` การที่ขึ้น 85 หมายถึง**hash table ใหญ่เกิน `work_mem` ต้อง spill ไปดิสก์**
2. **`Disk Usage: 15464kB`** — ยืนยันชัดเจนว่ามีการเขียนอ่านดิสก์ระหว่างการ aggregate จริง (ไม่ใช่แค่ทฤษฎี)
3. **`Buckets: 4096  Batches: 32`** ที่ node `Hash` (การ build hash table จาก `orders`) — ก็ spill เช่นกัน

### ขั้นตอนที่ 3: ตรวจสอบ work_mem ปัจจุบัน

```sql
SHOW work_mem;
```

```
 work_mem
----------
 4MB
```

`work_mem = 4MB` เป็นค่า default ที่เหมาะกับตอนที่ `order_items` มีแค่ 120,000 แถว (ตอนนั้น hash table ที่ต้องสร้างมีขนาดพอดีกับ 4MB) แต่เมื่อข้อมูลโตเป็น 420,000 แถว hash table ที่ต้องใช้จัดกลุ่ม `product_id` (ประมาณ 5,000 กลุ่ม) และ hash join กับ `orders` (40,000 แถว) มีขนาดใหญ่เกิน 4MB ไปแล้ว — **นี่คือสาเหตุที่แท้จริงของการช้าลง: ข้อมูลโตขึ้นตามธรรมชาติ แต่การตั้งค่า memory ไม่เคยถูกทบทวนตาม**

### ขั้นตอนที่ 4: ทดสอบแก้ไขด้วยการเพิ่ม work_mem (เฉพาะ session ก่อน เพื่อพิสูจน์สมมติฐาน)

```sql
SET work_mem = '16MB';

EXPLAIN (ANALYZE, BUFFERS)
SELECT oi.product_id, count(*), sum(oi.quantity * oi.unit_price) AS total
FROM order_items oi
JOIN orders o ON o.order_id = oi.order_id
GROUP BY oi.product_id;

RESET work_mem;
```

ผลลัพธ์จริงหลังเพิ่ม `work_mem` (ทดสอบด้วย `work_mem = '4MB'` ซึ่งเป็นค่าที่เพียงพอพอดีสำหรับข้อมูลปัจจุบัน):

```
HashAggregate  (cost=14405.61..14468.05 rows=4995 width=44) (actual time=339.944..341.292 rows=5000 loops=1)
  Group Key: oi.product_id
  Batches: 1  Memory Usage: 2257kB
  ->  Hash Join  (cost=1177.00..9155.61 rows=420000 width=14) (actual time=9.385..164.222 rows=420000 loops=1)
        Hash Cond: (oi.order_id = o.order_id)
        ->  Seq Scan on order_items oi  (cost=0.00..6876.00 rows=420000 width=18) (actual time=0.004..30.564 rows=420000 loops=1)
        ->  Hash  (cost=677.00..677.00 rows=40000 width=4) (actual time=9.137..9.139 rows=40000 loops=1)
              Buckets: 65536  Batches: 1  Memory Usage: 1919kB
              ->  Seq Scan on orders o  (cost=0.00..677.00 rows=40000 width=4) (actual time=0.004..3.712 rows=40000 loops=1)
Execution Time: 341.844 ms
```

**`Batches: 1` ทั้งสองจุด — ไม่มีการ spill ไปดิสก์อีกต่อไป Execution Time ลดจาก 637.751ms เหลือ 341.844ms (เร็วขึ้น ~1.87 เท่า)** ยืนยันสมมติฐานชัดเจน

### ขั้นตอนที่ 5: ตัดสินใจแก้ไขถาวรอย่างเหมาะสม

เนื่องจากนี่เป็น query ที่รันจากระบบ reporting/dashboard ไม่ใช่ query ทั่วไปของทุก connection การเพิ่ม `work_mem` ระดับ cluster ทั้งหมดจะสิ้นเปลือง memory โดยไม่จำเป็น (ทุก connection ที่ sort/hash จะได้ memory เพิ่มไปด้วย แม้ไม่ต้องการ) แนวทางที่เหมาะสมกว่า:

```sql
-- ทางเลือกที่ดีที่สุด: ตั้งค่าเฉพาะ role ที่ใช้รัน reporting query
ALTER ROLE reporting_user SET work_mem = '32MB';
```

```sql
-- หรือถ้า query นี้รันผ่าน connection pool เฉพาะทาง ตั้งเฉพาะ transaction ที่เรียกใช้
BEGIN;
SET LOCAL work_mem = '32MB';
SELECT oi.product_id, count(*), sum(oi.quantity * oi.unit_price) AS total
FROM order_items oi
JOIN orders o ON o.order_id = oi.order_id
GROUP BY oi.product_id;
COMMIT;
```

และตั้ง monitoring ป้องกันปัญหาซ้ำในอนาคต:

```sql
-- ติดตามการเติบโตของตารางสำคัญเป็นระยะ เพื่อทบทวนการตั้งค่า memory/index ให้ทัน
SELECT
    relname,
    n_live_tup,
    pg_size_pretty(pg_total_relation_size(relid)) AS total_size
FROM pg_stat_user_tables
WHERE relname IN ('orders', 'order_items', 'products', 'customers')
ORDER BY n_live_tup DESC;
```

### สรุปบทเรียนจาก Incident นี้

| จุดสังเกต | บทเรียน |
|---|---|
| `Batches > 1` ใน HashAggregate/Hash | สัญญาณแรกสุดของ `work_mem` ไม่พอ ต้องมองหาเสมอเมื่อ query ช้าลงโดยไม่มีการแก้โค้ด |
| ข้อมูลโตขึ้นตามธรรมชาติของธุรกิจ | เป็นสาเหตุการช้าลงที่พบบ่อยที่สุด แต่มักถูกมองข้าม เพราะ "ไม่มีใครแก้โค้ด" — การตั้งค่าที่เหมาะสมเมื่อ 6 เดือนก่อนอาจไม่เหมาะแล้ววันนี้ |
| อย่าเพิ่ม `work_mem` ระดับ cluster แบบไม่คิด | เพราะแต่ละ connection อาจเปิดหลาย sort/hash พร้อมกัน การเพิ่มแบบ global เสี่ยงทำให้ OOM ได้ในช่วง peak load — ตั้งเฉพาะ role/session ที่จำเป็นเท่านั้น |
| กระบวนการวินิจฉัยที่ถูกต้อง | (1) ยืนยันด้วย `EXPLAIN (ANALYZE, BUFFERS)` เทียบกับที่เคย baseline ไว้ (2) หา metric ผิดปกติ (Batches, Disk Usage, shared read สูง) (3) ตั้งสมมติฐาน (4) ทดสอบใน session เดียวก่อนแก้จริง (5) แก้ไขแบบ scope แคบที่สุดที่เพียงพอ (6) ตั้ง monitoring ป้องกันซ้ำ |

---

## สรุปท้ายบท

บทนี้พาไปไกลกว่าการอ่าน `EXPLAIN` พื้นฐานใน Part 045 สู่การวินิจฉัยระดับ production จริง:

- **`EXPLAIN (ANALYZE, BUFFERS)`** คือเครื่องมือที่ให้ข้อมูลจริงที่สุดเกี่ยวกับ I/O ของ query — `shared hit` vs `shared read` บอกว่า query กำลังใช้ cache หรือกำลังรอดิสก์
- **Extended statistics** (`CREATE STATISTICS`) แก้ปัญหาการประมาณ row ผิดพลาดที่เกิดจากคอลัมน์ที่มีความสัมพันธ์กัน แต่ต้องเลือก kind (`dependencies`/`ndistinct`/`mcv`) ให้ตรงกับลักษณะเงื่อนไข (equality vs range)
- **Query hint** ไม่ใช่ทางออกแรกที่ควรนึกถึง — ตรวจสอบ index และ statistics ก่อนเสมอ ถ้าจำเป็นจริงๆ ให้ใช้ scope แคบที่สุดที่ทำได้ (`SET LOCAL`) และเข้าใจความเสี่ยงของการ "ล็อก" แผนที่ไม่ปรับตามข้อมูลในอนาคต
- **Parallel query** เกิดอัตโนมัติตามเงื่อนไข cost/ขนาดตาราง — ต้องดู `Workers Planned` vs `Workers Launched` เพื่อรู้ว่าระบบมี worker พอหรือไม่
- **CTE materialization** ส่งผลต่อ performance ได้ชัดเจนในสถานการณ์จริง — CTE ที่อ้างอิงหลายครั้งควร materialize (default มักทำให้อยู่แล้ว), แต่บางครั้งการ inline (`NOT MATERIALIZED`) ช่วยให้ optimizer มองเห็นทะลุ CTE และ optimize ได้ดีกว่า
- **UNNEST pattern** สำหรับ bulk insert/update ลด round-trip จาก N ครั้งเหลือ 1 ครั้ง ให้ประสิทธิภาพต่างกันสิบถึงร้อยเท่า
- **`pg_locks` + `pg_stat_activity`** (โดยเฉพาะฟังก์ชันสำเร็จรูป `pg_blocking_pids()`) คือเครื่องมือหลักในการวินิจฉัย lock contention
- **`statement_timeout`** ควรตั้งเป็นมาตรฐานทุกระบบ production เพื่อป้องกัน query ที่หลุดควบคุมกินทรัพยากรทั้งเครื่อง
- **`pgbench`** ให้ผล A/B testing ที่น่าเชื่อถือกว่าการรัน `EXPLAIN ANALYZE` ครั้งเดียว เพราะวัดภายใต้ concurrent load จริง
- Incident จริงส่วนใหญ่ไม่ได้เกิดจาก "โค้ดผิด" แต่เกิดจาก**ข้อมูลที่โตขึ้นตามกาลเวลา** จนการตั้งค่าเดิม (เช่น `work_mem`) ไม่เพียงพออีกต่อไป — การ monitoring เชิงรุกสำคัญพอๆ กับการแก้ปัญหาเมื่อเกิดขึ้นแล้ว

บทถัดไปจะขยายมุมมองจากระดับ query เดี่ยว ไปสู่ระดับ **capacity planning** ทั้งระบบ — การประมาณ growth, การวางแผน hardware/resource ล่วงหน้า และการตั้ง threshold สำหรับ scale ก่อนที่ปัญหาแบบ Step 740 จะเกิดขึ้นจริง

**บทถัดไป**: [Part 075 — Capacity Planning](./part-075-capacity-planning.md)

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1

จากการรัน `EXPLAIN (ANALYZE, BUFFERS)` สองครั้งติดกันด้วย query เดียวกัน พบว่าครั้งแรกมี `Buffers: shared read=500` และครั้งที่สองมี `Buffers: shared hit=500` จงอธิบายว่าเกิดอะไรขึ้น และทำไม `cost` ในทั้งสองแผนถึงเท่ากันเป๊ะ แม้ `actual time` จะต่างกัน

<details>
<summary>เฉลย</summary>

ครั้งแรก page ทั้ง 500 page ที่ query ต้องการยังไม่อยู่ใน `shared_buffers` (PostgreSQL memory pool ของตัวเอง) จึงต้องขอจาก OS (`shared read`) ซึ่งอาจไปโดน OS page cache หรือ physical disk ก็ได้ พอรันครั้งที่สอง page เหล่านั้นถูกโหลดเข้า `shared_buffers` แล้วจากครั้งแรก จึงอ่านได้ทันที (`shared hit`) โดยไม่ต้องผ่าน system call เลย

ส่วน `cost` เท่ากันเป๊ะเพราะ `cost` เป็นตัวเลขที่ planner **คำนวณล่วงหน้าจาก statistics** (จำนวนแถว, ขนาดตาราง, การกระจายตัวของข้อมูล) ไม่ได้เกี่ยวกับสถานะ cache ของ buffer ขณะรันจริงเลย แผนที่ planner เลือกจึงเหมือนเดิมทุกครั้ง (เพราะ statistics ไม่เปลี่ยน) แต่ `actual time` วัดจากเวลาจริงที่ใช้ ซึ่งขึ้นกับว่าต้องรอ I/O หรือไม่
</details>

### แบบฝึกหัดที่ 2

Query `SELECT * FROM products WHERE category_id = 5 AND stock_quantity = 0;` มี estimate ผิดพลาดจาก 10 แถวเป็น 207 แถวจริง จงอธิบายว่าทำไม planner ถึงประมาณผิด และ `CREATE STATISTICS` kind ไหนที่แก้ปัญหานี้ได้ตรงจุดที่สุด

<details>
<summary>เฉลย</summary>

Planner สมมติว่า `category_id` และ `stock_quantity` เป็นอิสระต่อกัน จึงคำนวณ selectivity โดยคูณ P(category_id=5) กับ P(stock_quantity=0) เข้าด้วยกัน แต่ในความเป็นจริงสินค้าใน category 5 ส่วนใหญ่ (85%) มี `stock_quantity = 0` ซึ่งเป็นความสัมพันธ์ (correlation) ที่ planner ไม่รู้

kind ที่แก้ปัญหานี้ตรงจุดที่สุดคือ `mcv` (most common values) เพราะมันเก็บ **combination ของค่าที่พบบ่อยจากหลายคอลัมน์พร้อมกัน** ทำให้ planner รู้ตรงๆ ว่า `(category_id=5, stock_quantity=0)` เกิดขึ้นบ่อยแค่ไหนจริง โดยไม่ต้องอาศัยการคูณ selectivity แบบอิสระ (`dependencies` ก็ช่วยได้ในบางกรณีของ equality แต่ `mcv` ให้ผลแม่นยำที่สุดสำหรับกรณีนี้ตามที่พิสูจน์จากผลทดสอบจริง)
</details>

### แบบฝึกหัดที่ 3

ทำไม PostgreSQL ไม่มี syntax สำหรับ query hint ในตัว (เช่น `/*+ INDEX(...) */` แบบ MySQL) และอะไรคือ**ขั้นตอนที่ควรทำก่อน**การพิจารณาใช้ `pg_hint_plan` หรือปิด planner method ด้วย GUC

<details>
<summary>เฉลย</summary>

ทีมพัฒนา PostgreSQL มองว่า hint ที่ตายตัวจะ "ล็อก" query ไว้กับแผนหนึ่งแม้ข้อมูลจะเปลี่ยนแปลงในอนาคต ซึ่งอาจนำไปสู่ปัญหาที่แย่กว่าเดิม (แผนที่ดีวันนี้อาจแย่ในอีก 1 ปี แต่ hint จะยังบังคับใช้แผนเดิมต่อไปเพราะไม่มีใครแก้)

ก่อนพิจารณา hint ควรทำตามลำดับ: (1) ตรวจสอบว่ามี index ที่เหมาะสมหรือไม่ (สาเหตุอันดับหนึ่งของแผนแย่คือไม่มี index ไม่ใช่ planner โง่) (2) รัน `ANALYZE` ให้ statistics ทันสมัย (3) พิจารณา extended statistics ถ้ามีคอลัมน์ correlated (4) พิจารณาเขียน query ใหม่ให้ planner ตัดสินใจถูกเองตามธรรมชาติ — เมื่อทำครบทุกขั้นแล้วยังไม่ได้ผล จึงค่อยพิจารณา hint เป็นทางเลือกสุดท้าย
</details>

### แบบฝึกหัดที่ 4

`EXPLAIN` แสดง `Workers Planned: 4` แต่ `Workers Launched: 2` จงอธิบายสาเหตุที่เป็นไปได้ และผลกระทบต่อ performance ของ query นั้น

<details>
<summary>เฉลย</summary>

สาเหตุที่พบบ่อยที่สุดคือระบบมี worker process ไม่พอ ณ ขณะนั้น เพราะ `max_worker_processes` (หรือ `max_parallel_workers` ทั้ง cluster) ถูก query อื่นที่รันพร้อมกันแย่งใช้ไปแล้ว ทำให้ planner ที่วางแผนไว้ล่วงหน้าด้วยสมมติฐานว่าจะได้ 4 worker กลับได้จริงแค่ 2 worker

ผลกระทบคือ query จะทำงานช้ากว่าที่ planner ประมาณไว้ตอนวางแผน (cost ที่คำนวณจาก 4 worker ไม่ตรงกับความเป็นจริงที่มีแค่ 2 worker) ถ้าเกิดขึ้นบ่อยควรพิจารณาเพิ่ม `max_worker_processes`/`max_parallel_workers` หรือลด concurrent parallel query ในระบบ
</details>

### แบบฝึกหัดที่ 5

จงเขียน query ทดสอบว่า CTE ต่อไปนี้ (ที่ถูกอ้างอิง 2 ครั้ง) ถูก materialize โดย default หรือไม่ และอธิบายว่าจะดูจาก node ไหนใน `EXPLAIN` output

```sql
WITH recent_orders AS (
    SELECT * FROM orders WHERE order_date > now() - interval '7 days'
)
SELECT
    (SELECT count(*) FROM recent_orders WHERE status = 'paid'),
    (SELECT count(*) FROM recent_orders WHERE status = 'pending');
```

<details>
<summary>เฉลย</summary>

```sql
EXPLAIN ANALYZE
WITH recent_orders AS (
    SELECT * FROM orders WHERE order_date > now() - interval '7 days'
)
SELECT
    (SELECT count(*) FROM recent_orders WHERE status = 'paid'),
    (SELECT count(*) FROM recent_orders WHERE status = 'pending');
```

ให้ดูว่ามี node ชื่อ **`CTE recent_orders`** ปรากฏครั้งเดียว (การคำนวณจริง) ตามด้วย **`CTE Scan on recent_orders`** ปรากฏ 2 ครั้ง (การอ้างอิงซ้ำ) หรือไม่ ถ้าเห็นแบบนี้แปลว่า materialize แล้ว (คำนวณ `orders WHERE order_date > ...` แค่ครั้งเดียว แล้ว scan ผลลัพธ์ที่เก็บไว้ซ้ำสองครั้ง) ซึ่งเป็นพฤติกรรม default ของ PostgreSQL เมื่อ CTE ถูกอ้างอิงมากกว่า 1 ครั้ง
</details>

### แบบฝึกหัดที่ 6

แอปพลิเคชันต้อง insert ข้อมูล order_items 10,000 แถวจาก array ของ order_id, product_id, quantity ที่มีอยู่แล้วในโค้ด จงเขียน SQL statement เดียวที่มีประสิทธิภาพสูงสุดสำหรับงานนี้ โดยใช้ pattern ที่เรียนในบทนี้

<details>
<summary>เฉลย</summary>

```sql
INSERT INTO order_items (order_id, product_id, quantity, unit_price)
SELECT * FROM unnest(
    $1::int[],      -- array ของ order_id
    $2::int[],      -- array ของ product_id
    $3::int[],      -- array ของ quantity
    $4::numeric[]   -- array ของ unit_price
) AS t(order_id, product_id, quantity, unit_price);
```

การใช้ `unnest()` กับ array parameter หลายชุดทำให้ทั้ง 10,000 แถวถูก insert ด้วย statement เดียว (1 round-trip, 1 ครั้งของการ parse/plan) แทนที่จะเป็น 10,000 round-trip แยกกัน ซึ่งในสภาพแวดล้อมจริงที่มี network latency จะให้ผลต่างกันหลายสิบเท่า
</details>

### แบบฝึกหัดที่ 7

จงเขียน query สำหรับหา session ที่ถูก block อยู่ในขณะนี้ พร้อมทั้ง SQL text ของทั้งฝั่งที่ถูก block และฝั่งที่เป็นต้นเหตุ (blocking) โดยใช้ `pg_locks` และ `pg_stat_activity`

<details>
<summary>เฉลย</summary>

```sql
SELECT
    blocked_locks.pid       AS blocked_pid,
    blocked_activity.query  AS blocked_query,
    blocking_locks.pid      AS blocking_pid,
    blocking_activity.query AS blocking_query,
    now() - blocked_activity.query_start AS blocked_duration
FROM pg_locks blocked_locks
JOIN pg_stat_activity blocked_activity
    ON blocked_activity.pid = blocked_locks.pid
JOIN pg_locks blocking_locks
    ON blocking_locks.locktype = blocked_locks.locktype
   AND blocking_locks.database IS NOT DISTINCT FROM blocked_locks.database
   AND blocking_locks.relation IS NOT DISTINCT FROM blocked_locks.relation
   AND blocking_locks.pid != blocked_locks.pid
JOIN pg_stat_activity blocking_activity
    ON blocking_activity.pid = blocking_locks.pid
WHERE NOT blocked_locks.granted;
```

หรือใช้ฟังก์ชันสำเร็จรูปที่สั้นกว่าสำหรับดูภาพรวมเร็วๆ:

```sql
SELECT pid, wait_event_type, pg_blocking_pids(pid), query
FROM pg_stat_activity
WHERE pg_blocking_pids(pid) != '{}';
```
</details>

### แบบฝึกหัดที่ 8

Web application ต้องการ query timeout ที่ 10 วินาทีสำหรับทุก connection แต่ reporting tool (connection ผ่าน role `reporting_user`) ต้องการ timeout ที่ 5 นาที จงเขียนคำสั่งตั้งค่าทั้งสองแบบให้ถูกต้อง

<details>
<summary>เฉลย</summary>

```sql
-- ตั้งค่า default สำหรับทุก connection ที่ระดับ database (หรือ cluster ใน postgresql.conf)
ALTER DATABASE ecommerce SET statement_timeout = '10s';

-- override เฉพาะ role reporting_user ให้ยาวกว่า
ALTER ROLE reporting_user SET statement_timeout = '5min';
```

เมื่อตั้งแบบนี้ role อื่นๆ ที่เชื่อมต่อ database `ecommerce` จะใช้ค่า 10 วินาทีตาม default ของ database ส่วน `reporting_user` จะ override เป็น 5 นาทีเฉพาะตัวเอง (per-role setting มีความสำคัญเหนือกว่า per-database setting)
</details>

### แบบฝึกหัดที่ 9

จากผลการทดสอบ pgbench พบว่า Variant A ได้ tps = 3153 และ Variant B ได้ tps = 379 ภายใต้เงื่อนไขการทดสอบเดียวกันทุกอย่าง (4 clients, 5 วินาที) จงอธิบายว่าทำไมการรัน `EXPLAIN ANALYZE` เพียงครั้งเดียวอาจให้ข้อสรุปที่คลาดเคลื่อนกว่าการทำ A/B test ด้วย pgbench

<details>
<summary>เฉลย</summary>

`EXPLAIN ANALYZE` ครั้งเดียววัดผลจาก**การรันหนึ่งครั้งเท่านั้น** ซึ่งอาจได้รับผลกระทบจาก noise ชั่วขณะ เช่น buffer cache ยังไม่ warm, CPU ถูก process อื่นแย่งใช้ชั่วครู่, หรือ disk I/O ที่ผันผวน ทำให้ตัวเลขที่ได้ไม่สะท้อนพฤติกรรมทั่วไปของ query นั้น

`pgbench` รัน query ซ้ำหลายพันครั้งด้วย concurrent connection หลายตัว ทำให้ผลลัพธ์ (`tps`, `latency average`) เป็นค่าเฉลี่ยที่ผ่านการ smooth out ความผันผวนแล้ว และยังจำลองสภาพ concurrent load ใกล้เคียง production มากกว่าการรัน query เดี่ยวๆ หนึ่งครั้งบน connection เดียว — ทำให้ข้อสรุปว่า "variant ไหนดีกว่า" น่าเชื่อถือกว่ามาก
</details>

### แบบฝึกหัดที่ 10

Query สำหรับ dashboard ที่เคยใช้เวลา 300ms กลับมาใช้เวลา 600ms+ โดยไม่มีการแก้โค้ดใดๆ จง**เรียงลำดับขั้นตอน**การวินิจฉัยที่ถูกต้องตามที่เรียนในบทนี้ (Step 740) จากขั้นแรกจนถึงขั้นสุดท้าย

<details>
<summary>เฉลย</summary>

1. รัน `EXPLAIN (ANALYZE, BUFFERS)` กับ query จริง เพื่อยืนยันปัญหาและดูตัวเลขปัจจุบัน (เปรียบเทียบกับ baseline ถ้ามีเก็บไว้)
2. มองหา metric ผิดปกติในผลลัพธ์ เช่น `Batches > 1` (บ่งบอก work_mem ไม่พอ), `Disk Usage` (ยืนยันการ spill), `shared read` สูงผิดปกติ (บ่งบอก cache miss มาก), หรือ row estimate ที่คลาดเคลื่อนมาก (บ่งบอก statistics ล้าสมัยหรือ correlation)
3. ตรวจสอบค่า configuration ที่เกี่ยวข้อง (เช่น `SHOW work_mem;`) และเปรียบเทียบกับขนาดข้อมูลปัจจุบัน (เช่นดูจาก `pg_stat_user_tables` ว่าตารางโตขึ้นแค่ไหนจากเดิม)
4. ตั้งสมมติฐานถึงสาเหตุ (เช่น "ข้อมูลโตขึ้นจน work_mem เดิมไม่พอ")
5. ทดสอบแก้ไขสมมติฐานใน session เดียวก่อน (เช่น `SET work_mem = '16MB'` แล้วรัน `EXPLAIN ANALYZE` ซ้ำ) เพื่อพิสูจน์ว่าสมมติฐานถูกต้องและวัดผลเชิงปริมาณของการแก้ไข
6. เลือกวิธีแก้ไขถาวรที่มี scope แคบที่สุดที่เพียงพอ (เช่น ตั้งที่ระดับ role แทนที่จะตั้งทั้ง cluster) เพื่อไม่ให้กระทบ workload อื่นโดยไม่จำเป็น
7. ตั้ง monitoring เพื่อป้องกันปัญหาลักษณะเดียวกันเกิดซ้ำในอนาคต (เช่น ติดตามการเติบโตของตารางเป็นระยะ)
</details>

---

**บทถัดไป**: [Part 075 — Capacity Planning](./part-075-capacity-planning.md)
