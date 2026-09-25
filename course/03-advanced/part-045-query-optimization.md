# Part 045: เทคนิค Query Optimization

**หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับสูง | Part 045**

---

## เป้าหมายการเรียนรู้

เมื่อเรียนจบบทนี้ ผู้เรียนจะสามารถ:

1. เข้าใจหลักการพื้นฐานของการ optimize query — "วัดก่อนแก้เสมอ" (measure, don't guess) และรู้จักใช้ `EXPLAIN (ANALYZE, BUFFERS)` เป็นเครื่องมือหลัก
2. เขียน `WHERE` clause ให้เป็น sargable (Search ARGument ABLE) เพื่อให้ query planner สามารถใช้ index ได้จริง
3. เข้าใจผลกระทบของ `SELECT *` เทียบกับการเลือกเฉพาะคอลัมน์ที่ต้องใช้ ทั้งในแง่ I/O และโอกาสเกิด Index Only Scan
4. ออกแบบ index ให้รองรับการทำงานร่วมกันของ `ORDER BY` และ `LIMIT` เพื่อหลีกเลี่ยงการ sort ข้อมูลทั้งตาราง
5. Optimize การทำ `JOIN` หลายตาราง — เข้าใจผลของลำดับ join, index บนคอลัมน์ที่ join, และความสำคัญของสถิติ (statistics) ที่แม่นยำ
6. แปลง correlated subquery ให้เป็น `JOIN` เมื่อเหมาะสม และทบทวนความแตกต่างระหว่าง `IN` กับ `EXISTS` ในเชิง performance
7. เข้าใจผลของค่า `work_mem` ต่อการทำ sort และ hash operation พร้อมรู้วิธีปรับค่าในระดับ session เทียบกับระดับ global
8. รู้จักปัญหา N+1 query ที่มักเกิดจากการใช้ ORM และรู้วิธีแก้ด้วยการรวม query เป็น batch หรือ JOIN
9. ใช้ `pg_stat_statements` เพื่อค้นหา query ที่ช้าที่สุดในระบบจริงเบื้องต้น (รายละเอียดเชิงลึกจะอยู่ใน Part 057 และ Part 072)
10. ลงมือ optimize query ที่ซับซ้อนแบบครบวงจร ทีละขั้นตอน พร้อมวัดผลก่อน-หลังด้วย `EXPLAIN ANALYZE` อย่างเป็นระบบ

---

## เตรียมข้อมูล

บทนี้ใช้ schema แบบ e-commerce เดียวกับ Part 041 เพื่อให้ผู้เรียนที่ผ่านบทก่อนหน้ามาแล้วคุ้นเคย และสร้างข้อมูลจำนวนมากพอสมควร (หลักพันแถว) ด้วย `generate_series` เพื่อให้เห็นผลต่างของ performance ได้ชัดเจนเมื่อรัน `EXPLAIN ANALYZE`

```sql
-- ลบตารางเดิมถ้ามี เพื่อให้เริ่มต้นสะอาด
DROP TABLE IF EXISTS order_items CASCADE;
DROP TABLE IF EXISTS orders CASCADE;
DROP TABLE IF EXISTS customers CASCADE;
DROP TABLE IF EXISTS products CASCADE;
DROP TABLE IF EXISTS suppliers CASCADE;
DROP TABLE IF EXISTS categories CASCADE;

CREATE TABLE categories (
    category_id         SERIAL PRIMARY KEY,
    category_name       VARCHAR(100) NOT NULL,
    parent_category_id  INTEGER REFERENCES categories(category_id)
);

CREATE TABLE suppliers (
    supplier_id    SERIAL PRIMARY KEY,
    supplier_name  VARCHAR(150) NOT NULL,
    country        VARCHAR(60)
);

CREATE TABLE products (
    product_id      SERIAL PRIMARY KEY,
    product_name    VARCHAR(150) NOT NULL,
    category_id     INTEGER REFERENCES categories(category_id),
    supplier_id     INTEGER REFERENCES suppliers(supplier_id),
    unit_price      NUMERIC(10,2) NOT NULL,
    stock_quantity  INTEGER NOT NULL DEFAULT 0,
    is_active       BOOLEAN NOT NULL DEFAULT true
);

CREATE TABLE customers (
    customer_id   SERIAL PRIMARY KEY,
    first_name    VARCHAR(60) NOT NULL,
    last_name     VARCHAR(60) NOT NULL,
    email         VARCHAR(150) UNIQUE,
    country       VARCHAR(60),
    signup_date   DATE NOT NULL DEFAULT CURRENT_DATE
);

CREATE TABLE orders (
    order_id      SERIAL PRIMARY KEY,
    customer_id   INTEGER REFERENCES customers(customer_id),
    order_date    TIMESTAMPTZ NOT NULL DEFAULT now(),
    status        VARCHAR(20) NOT NULL DEFAULT 'pending',
    ship_country  VARCHAR(60)
);

CREATE TABLE order_items (
    order_item_id  SERIAL PRIMARY KEY,
    order_id       INTEGER REFERENCES orders(order_id),
    product_id     INTEGER REFERENCES products(product_id),
    quantity       INTEGER NOT NULL CHECK (quantity > 0),
    unit_price     NUMERIC(10,2) NOT NULL
);
```

### สร้างข้อมูลจำนวนมากด้วย generate_series

```sql
-- หมวดหมู่สินค้า (ประมาณ 30 หมวดหลัก + หมวดย่อย)
INSERT INTO categories (category_name, parent_category_id)
SELECT 'หมวดหลัก ' || g, NULL
FROM generate_series(1, 20) AS g;

INSERT INTO categories (category_name, parent_category_id)
SELECT 'หมวดย่อย ' || g, (g % 20) + 1
FROM generate_series(1, 40) AS g;

-- ผู้จัดจำหน่าย 200 ราย
INSERT INTO suppliers (supplier_name, country)
SELECT
    'Supplier ' || g,
    (ARRAY['Thailand','China','Japan','Vietnam','USA','Germany','Korea'])[1 + (g % 7)]
FROM generate_series(1, 200) AS g;

-- สินค้า 5,000 รายการ
INSERT INTO products (product_name, category_id, supplier_id, unit_price, stock_quantity, is_active)
SELECT
    'Product ' || g,
    1 + (g % 60),
    1 + (g % 200),
    round((random() * 5000 + 10)::numeric, 2),
    (random() * 500)::int,
    (random() > 0.05)
FROM generate_series(1, 5000) AS g;

-- ลูกค้า 10,000 ราย
INSERT INTO customers (first_name, last_name, email, country, signup_date)
SELECT
    'FirstName' || g,
    'LastName' || g,
    'customer' || g || '@example.com',
    (ARRAY['Thailand','Singapore','Malaysia','Vietnam','Indonesia','Philippines'])[1 + (g % 6)],
    CURRENT_DATE - ((random() * 1500)::int || ' days')::interval
FROM generate_series(1, 10000) AS g;

-- คำสั่งซื้อ 50,000 รายการ
INSERT INTO orders (customer_id, order_date, status, ship_country)
SELECT
    1 + (random() * 9999)::int,
    now() - ((random() * 900)::int || ' days')::interval,
    (ARRAY['pending','processing','shipped','delivered','cancelled'])[1 + (random() * 4)::int],
    (ARRAY['Thailand','Singapore','Malaysia','Vietnam','Indonesia','Philippines'])[1 + (random() * 5)::int]
FROM generate_series(1, 50000) AS g;

-- รายการสินค้าในคำสั่งซื้อ ~150,000 รายการ (เฉลี่ยออเดอร์ละ 3 รายการ)
INSERT INTO order_items (order_id, product_id, quantity, unit_price)
SELECT
    1 + (random() * 49999)::int,
    1 + (random() * 4999)::int,
    1 + (random() * 5)::int,
    round((random() * 5000 + 10)::numeric, 2)
FROM generate_series(1, 150000) AS g;

-- อัปเดตสถิติให้ planner รู้จักข้อมูลล่าสุด
ANALYZE categories;
ANALYZE suppliers;
ANALYZE products;
ANALYZE customers;
ANALYZE orders;
ANALYZE order_items;
```

> **หมายเหตุ:** จำนวนแถวที่ generate (products 5,000, customers 10,000, orders 50,000, order_items 150,000) ถูกเลือกให้มากพอที่ planner จะเลือกใช้กลยุทธ์ที่ต่างกันตามว่ามี index หรือไม่ ทำให้เห็นความแตกต่างของเวลาใน `EXPLAIN ANALYZE` ได้ชัดเจน ตัวเลข timing ที่แสดงในบทนี้เป็นตัวอย่างจากเครื่องทดสอบ ซึ่งอาจแตกต่างกันไปตาม hardware ของแต่ละคน แต่ **สัดส่วนความเร็วที่ดีขึ้น** ควรอยู่ในทิศทางเดียวกัน

---

## Step 441: หลักการทั่วไปในการ optimize query — วัดก่อนแก้เสมอ (Measure, Don't Guess)

หลักการที่สำคัญที่สุดของการ optimize query คือ **อย่าเดา** ผู้พัฒนามือใหม่มักจะ "รู้สึก" ว่า query ช้าเพราะเหตุผลบางอย่าง แล้วรีบแก้ไขโดยไม่ตรวจสอบก่อน ซึ่งอาจทำให้เสียเวลาไปกับจุดที่ไม่ใช่ปัญหาจริง หรือแย่กว่านั้นคือทำให้ query ช้าลงกว่าเดิม

### ขั้นตอนมาตรฐานในการ optimize query

1. **วัดผลปัจจุบัน** ด้วย `EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)`
2. **หา bottleneck** จาก execution plan — ดูว่า node ไหนใช้เวลานานที่สุด, มี Seq Scan บนตารางใหญ่หรือไม่, estimated rows กับ actual rows ต่างกันมากหรือไม่
3. **ตั้งสมมติฐาน** ว่าจะแก้ปัญหาอย่างไร (เพิ่ม index, เขียน query ใหม่, ปรับ config)
4. **ทำการแก้ไขทีละอย่าง** ไม่ควรแก้หลายอย่างพร้อมกันเพราะจะไม่รู้ว่าอะไรคือสาเหตุที่แท้จริงที่ทำให้ดีขึ้น
5. **วัดผลอีกครั้ง** เปรียบเทียบกับค่าเดิม
6. **ทำซ้ำ** จนกว่าจะได้ performance ที่ยอมรับได้

### เครื่องมือพื้นฐานที่ต้องรู้จัก

```sql
-- แบบพื้นฐาน: ดู execution plan โดยไม่รันจริง
EXPLAIN
SELECT * FROM orders WHERE customer_id = 500;

-- แบบที่แนะนำเสมอ: รันจริงและวัดเวลา + การใช้ buffer/cache
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders WHERE customer_id = 500;
```

ตัวอย่างผลลัพธ์:

```
Seq Scan on orders  (cost=0.00..1041.00 rows=5 width=50)
                     (actual time=0.020..8.912 rows=5 loops=1)
  Filter: (customer_id = 500)
  Rows Removed by Filter: 49995
  Buffers: shared hit=541
Planning Time: 0.089 ms
Execution Time: 8.941 ms
```

**สิ่งที่ต้องอ่านให้เป็น:**

| ส่วนของ plan | ความหมาย |
|---|---|
| `cost=0.00..1041.00` | ต้นทุนที่ planner ประเมิน (startup..total) หน่วยเป็น "arbitrary unit" ไม่ใช่มิลลิวินาที |
| `rows=5` (ใน cost) | จำนวนแถวที่ planner **คาดการณ์** ไว้ล่วงหน้า |
| `actual time=0.020..8.912` | เวลาจริงที่ใช้ (startup..total) หน่วยเป็นมิลลิวินาที |
| `rows=5` (ใน actual) | จำนวนแถวที่ได้จริงเมื่อรัน — ถ้าต่างจาก estimated มาก แปลว่าสถิติไม่แม่นยำ |
| `Rows Removed by Filter` | จำนวนแถวที่ถูกอ่านขึ้นมาแล้วทิ้งไป เป็นสัญญาณว่าอาจต้องมี index |
| `Buffers: shared hit=541` | จำนวน page ที่อ่านจาก shared buffer cache (ถ้ามี `read=` แปลว่าต้องอ่านจาก disk) |

> **ข้อควรระวัง:** `EXPLAIN ANALYZE` จะ**รัน query จริง** ดังนั้นถ้าเป็น `INSERT`, `UPDATE`, `DELETE` ข้อมูลจะถูกเปลี่ยนแปลงจริง ควรใช้ภายใน transaction ที่ `ROLLBACK` เมื่อทดสอบ:

```sql
BEGIN;
EXPLAIN (ANALYZE, BUFFERS)
UPDATE products SET stock_quantity = stock_quantity - 1 WHERE product_id = 100;
ROLLBACK;
```

### สร้าง baseline function ช่วยเปรียบเทียบ

เพื่อความสะดวกในการเปรียบเทียบ "ก่อน" กับ "หลัง" ตลอดทั้งบทนี้ เราจะใช้รูปแบบเดียวกันเสมอ คือรัน `EXPLAIN (ANALYZE, BUFFERS)` ก่อนแก้ไข บันทึก Execution Time ไว้ แล้วรันอีกครั้งหลังแก้ไขเพื่อเทียบกัน

```sql
-- เคลียร์ cache ของ OS/PostgreSQL ระหว่างทดสอบทำได้ยากในเครื่อง production
-- แต่สำหรับ training environment เราสามารถรัน query ซ้ำ 2 ครั้งเพื่อให้ cache "warm" เท่ากัน
-- แล้วเปรียบเทียบเฉพาะรอบที่สองเป็นต้นไป เพื่อไม่ให้ disk I/O รอบแรกมากวนผล
```

---

## Step 442: การเขียน WHERE clause ให้ sargable

**Sargable** (SARGable = Search ARGument ABLE) หมายถึงเงื่อนไขใน `WHERE` ที่ planner สามารถใช้ index มาช่วย seek ข้อมูลได้โดยตรง โดยไม่ต้องอ่านและคำนวณค่าจากทุกแถวก่อน

ปัญหาที่พบบ่อยที่สุดคือการ **ครอบฟังก์ชัน (wrap function) บนคอลัมน์ที่มี index** ซึ่งทำให้ index ธรรมดาใช้งานไม่ได้

### ตัวอย่างปัญหา: ใช้ฟังก์ชันครอบคอลัมน์ที่มี index

```sql
-- สร้าง index ปกติบน email
CREATE INDEX idx_customers_email ON customers (email);

-- BEFORE: ค้นหาแบบไม่สนตัวพิมพ์เล็ก-ใหญ่ ด้วย LOWER()
EXPLAIN (ANALYZE, BUFFERS)
SELECT customer_id, first_name, last_name, email
FROM customers
WHERE LOWER(email) = 'customer4321@example.com';
```

ผลลัพธ์ (BEFORE):

```
Seq Scan on customers  (cost=0.00..245.00 rows=50 width=45)
                        (actual time=0.015..3.201 rows=1 loops=1)
  Filter: (lower((email)::text) = 'customer4321@example.com'::text)
  Rows Removed by Filter: 9999
  Buffers: shared hit=145
Planning Time: 0.112 ms
Execution Time: 3.224 ms
```

จะเห็นว่า planner **ไม่ใช้** `idx_customers_email` เลย เพราะ index ถูกสร้างจากค่า `email` ดิบๆ ไม่ใช่ค่าที่ผ่าน `LOWER()` แล้ว จึงต้องทำ Seq Scan อ่านทุกแถวแล้วค่อยคำนวณ `LOWER(email)` ทีละแถว

### วิธีแก้: สร้าง Expression Index

```sql
CREATE INDEX idx_customers_email_lower ON customers (LOWER(email));

ANALYZE customers;

-- AFTER: query เดิมทุกตัวอักษร แต่ตอนนี้มี expression index รองรับ
EXPLAIN (ANALYZE, BUFFERS)
SELECT customer_id, first_name, last_name, email
FROM customers
WHERE LOWER(email) = 'customer4321@example.com';
```

ผลลัพธ์ (AFTER):

```
Index Scan using idx_customers_email_lower on customers
                        (cost=0.29..8.31 rows=1 width=45)
                        (actual time=0.028..0.031 rows=1 loops=1)
  Index Cond: (lower((email)::text) = 'customer4321@example.com'::text)
  Buffers: shared hit=4
Planning Time: 0.098 ms
Execution Time: 0.051 ms
```

**สรุปผล:** จาก 3.224 ms เหลือ 0.051 ms (เร็วขึ้นประมาณ 60 เท่า) และ buffer ที่อ่านลดจาก 145 เหลือ 4 page เท่านั้น

### รูปแบบ non-sargable ที่พบบ่อยอื่นๆ

```sql
-- ❌ Non-sargable: บวกเลขในคอลัมน์
-- WHERE unit_price + 10 > 100

-- ✅ Sargable: ย้ายค่าคงที่ไปอีกฝั่ง
-- WHERE unit_price > 90

-- ❌ Non-sargable: แปลง type คอลัมน์ (implicit cast)
-- WHERE customer_id::text = '500'

-- ✅ Sargable: เทียบชนิดข้อมูลตรงกันตั้งแต่ต้น
-- WHERE customer_id = 500

-- ❌ Non-sargable: LIKE ที่ขึ้นต้นด้วย wildcard
-- WHERE product_name LIKE '%Product 99%'

-- ✅ Sargable (ถ้าค้นแบบ prefix): LIKE ที่ wildcard อยู่ท้าย เท่านั้นที่ btree index ช่วยได้
-- WHERE product_name LIKE 'Product 99%'

-- ❌ Non-sargable: ฟังก์ชันวันที่ครอบคอลัมน์
-- WHERE DATE(order_date) = '2026-01-15'

-- ✅ Sargable: ใช้ range แทน
-- WHERE order_date >= '2026-01-15' AND order_date < '2026-01-16'
```

ตัวอย่างการแก้ปัญหาเรื่องวันที่ให้เห็นภาพจริง:

```sql
CREATE INDEX idx_orders_order_date ON orders (order_date);
ANALYZE orders;

-- BEFORE: ใช้ DATE() ครอบคอลัมน์
EXPLAIN (ANALYZE, BUFFERS)
SELECT order_id, customer_id, order_date
FROM orders
WHERE DATE(order_date) = '2026-01-15';
```

```
Seq Scan on orders  (cost=0.00..1166.00 rows=137 width=24)
                     (actual time=0.018..9.874 rows=61 loops=1)
  Filter: (date(order_date) = '2026-01-15'::date)
  Rows Removed by Filter: 49939
  Buffers: shared hit=616
Execution Time: 9.911 ms
```

```sql
-- AFTER: เขียนเป็น range เพื่อให้ btree index ใช้งานได้
EXPLAIN (ANALYZE, BUFFERS)
SELECT order_id, customer_id, order_date
FROM orders
WHERE order_date >= '2026-01-15' AND order_date < '2026-01-16';
```

```
Index Scan using idx_orders_order_date on orders
                     (cost=0.29..14.12 rows=137 width=24)
                     (actual time=0.024..0.198 rows=61 loops=1)
  Index Cond: ((order_date >= '2026-01-15 00:00:00+00')
           AND (order_date < '2026-01-16 00:00:00+00'))
  Buffers: shared hit=9
Execution Time: 0.221 ms
```

เร็วขึ้นจาก 9.911 ms เหลือ 0.221 ms — ราว 45 เท่า

---

## Step 443: การหลีกเลี่ยง SELECT * และการเลือกเฉพาะคอลัมน์ที่ต้องใช้

การใช้ `SELECT *` เป็นนิสัยที่สะดวกตอนเขียนโค้ด แต่ส่งผลเสียต่อ performance หลายด้าน:

1. **I/O เพิ่มขึ้น** — ต้องอ่านข้อมูลทุกคอลัมน์จาก heap แม้จะใช้แค่บางคอลัมน์
2. **Network transfer เพิ่มขึ้น** — ส่งข้อมูลที่ไม่จำเป็นผ่าน network ระหว่าง database กับ application
3. **เสีย โอกาสใช้ Index Only Scan** — ถ้า query ต้องการแค่คอลัมน์ที่มีอยู่ใน index ครบถ้วน (covering index) planner สามารถอ่านจาก index อย่างเดียวโดยไม่ต้องแตะ heap เลย แต่ถ้าใช้ `SELECT *` จะบังคับให้ต้องไปอ่าน heap เสมอ

### ตัวอย่าง: Index Only Scan vs Heap Access

```sql
-- สร้าง composite index สำหรับ query ที่ใช้บ่อย
CREATE INDEX idx_orders_customer_status ON orders (customer_id, status);
ANALYZE orders;

-- BEFORE: SELECT * ทำให้ต้องอ่าน heap เพิ่ม แม้จะมี index ที่ตรงกับเงื่อนไข
EXPLAIN (ANALYZE, BUFFERS)
SELECT *
FROM orders
WHERE customer_id = 4321 AND status = 'delivered';
```

```
Index Scan using idx_orders_customer_status on orders
                     (cost=0.29..8.32 rows=1 width=50)
                     (actual time=0.021..0.024 rows=1 loops=1)
  Index Cond: ((customer_id = 4321) AND (status = 'delivered'::text))
  Buffers: shared hit=4
Execution Time: 0.041 ms
```

```sql
-- AFTER: เลือกเฉพาะคอลัมน์ที่อยู่ใน index (order_id ต้องเพิ่มเข้า index ด้วย INCLUDE)
CREATE INDEX idx_orders_customer_status_covering
    ON orders (customer_id, status) INCLUDE (order_id, order_date);
ANALYZE orders;

EXPLAIN (ANALYZE, BUFFERS)
SELECT order_id, order_date
FROM orders
WHERE customer_id = 4321 AND status = 'delivered';
```

```
Index Only Scan using idx_orders_customer_status_covering on orders
                     (cost=0.29..4.31 rows=1 width=12)
                     (actual time=0.015..0.017 rows=1 loops=1)
  Index Cond: ((customer_id = 4321) AND (status = 'delivered'::text))
  Heap Fetches: 0
  Buffers: shared hit=3
Execution Time: 0.029 ms
```

สังเกตบรรทัด `Heap Fetches: 0` — หมายความว่า planner **ไม่ต้อง** ไปอ่านตาราง heap เลย เพราะข้อมูลทั้งหมดที่ query ต้องการอยู่ใน index อยู่แล้ว (ต้องแน่ใจว่า table ผ่าน `VACUUM` เป็นประจำ เพื่อให้ visibility map แม่นยำและได้ประโยชน์เต็มที่จาก Index Only Scan)

### เปรียบเทียบผลกระทบต่อ network เมื่อ query จำนวนมากรวมกัน

```sql
-- ตัวอย่างที่เห็นภาพชัดกว่าคือ query จำนวนมากพร้อมกัน (เช่น listing หน้าแรกของเว็บ)
-- BEFORE: ดึงทุกคอลัมน์ของ products 200 แถว
EXPLAIN (ANALYZE, BUFFERS)
SELECT *
FROM products
WHERE is_active = true
ORDER BY product_id
LIMIT 200;
```

```
Limit  (cost=0.29..108.45 rows=200 width=58)
       (actual time=0.019..1.842 rows=200 loops=1)
  ->  Index Scan using products_pkey on products
       (cost=0.29..2571.79 rows=4750 width=58)
       (actual time=0.018..1.798 rows=200 loops=1)
       Filter: is_active
Buffers: shared hit=210
Execution Time: 1.879 ms
```

```sql
-- AFTER: เลือกเฉพาะคอลัมน์ที่หน้า listing ใช้จริง (สมมติหน้าเว็บใช้แค่ 4 คอลัมน์)
EXPLAIN (ANALYZE, BUFFERS)
SELECT product_id, product_name, unit_price, stock_quantity
FROM products
WHERE is_active = true
ORDER BY product_id
LIMIT 200;
```

```
Limit  (cost=0.29..95.10 rows=200 width=26)
       (actual time=0.014..1.203 rows=200 loops=1)
  ->  Index Scan using products_pkey on products
       (cost=0.29..2258.60 rows=4750 width=26)
       (actual time=0.013..1.171 rows=200 loops=1)
       Filter: is_active
Buffers: shared hit=210
Execution Time: 1.231 ms
```

แม้ในตัวอย่างนี้ `width` (ขนาดแถวโดยประมาณ) จะลดจาก 58 เหลือ 26 ไบต์ต่อแถว ความต่างของเวลาดูไม่มากในระดับ database เดียว แต่เมื่อคูณด้วยจำนวน request ต่อวินาทีในระบบจริง (เช่น หลักพัน request/วินาที) ผลต่างของ network payload และ memory ที่ต้อง serialize จะมีนัยสำคัญมาก

**หลักปฏิบัติ:** เลือกเฉพาะคอลัมน์ที่ใช้จริงเสมอ ทั้งเพื่อประสิทธิภาพและเพื่อความชัดเจนของโค้ด (คนอ่านรู้ทันทีว่า query นี้ต้องการอะไร) และเมื่อมีการเปลี่ยนแปลง schema (เช่น เพิ่มคอลัมน์ใหม่) จะไม่กระทบ query เดิมโดยไม่ได้ตั้งใจ

---

## Step 444: LIMIT กับ ORDER BY — ทำไมควรมี index รองรับการเรียงเสมอ

เมื่อใช้ `ORDER BY` คู่กับ `LIMIT` จุดประสงค์มักเป็นการดึง "N อันดับแรก" เช่น "คำสั่งซื้อล่าสุด 10 รายการ" ถ้าไม่มี index ที่รองรับลำดับการเรียง PostgreSQL จะต้อง**เรียงข้อมูลทั้งหมดก่อน**แล้วค่อยตัดเอาแค่ N แถวแรก ซึ่งสิ้นเปลืองมากเมื่อตารางมีข้อมูลจำนวนมาก

### ตัวอย่างปัญหา

```sql
-- BEFORE: ไม่มี index รองรับ order_date DESC
EXPLAIN (ANALYZE, BUFFERS)
SELECT order_id, customer_id, order_date, status
FROM orders
ORDER BY order_date DESC
LIMIT 10;
```

```
Limit  (cost=2456.78..2456.81 rows=10 width=24)
       (actual time=28.912..28.918 rows=10 loops=1)
  ->  Sort  (cost=2456.78..2581.78 rows=50000 width=24)
            (actual time=28.910..28.914 rows=10 loops=1)
       Sort Key: order_date DESC
       Sort Method: top-N heapsort  Memory: 26kB
       ->  Seq Scan on orders
            (cost=0.00..1041.00 rows=50000 width=24)
            (actual time=0.009..15.201 rows=50000 loops=1)
Buffers: shared hit=541
Execution Time: 28.941 ms
```

จะเห็นว่า PostgreSQL ต้อง Seq Scan อ่านทั้งตาราง (50,000 แถว) แล้วนำมา sort ทั้งหมด (แม้จะใช้ top-N heapsort ที่ฉลาดกว่า full sort ธรรมดา แต่ก็ยังต้องอ่านข้อมูลทุกแถวอยู่ดี)

### วิธีแก้: สร้าง index ตามลำดับการเรียงที่ query ใช้

```sql
CREATE INDEX idx_orders_order_date_desc ON orders (order_date DESC);
ANALYZE orders;

-- AFTER: query เดิมทุกตัวอักษร
EXPLAIN (ANALYZE, BUFFERS)
SELECT order_id, customer_id, order_date, status
FROM orders
ORDER BY order_date DESC
LIMIT 10;
```

```
Limit  (cost=0.29..0.85 rows=10 width=24)
       (actual time=0.019..0.035 rows=10 loops=1)
  ->  Index Scan using idx_orders_order_date_desc on orders
       (cost=0.29..2800.29 rows=50000 width=24)
       (actual time=0.018..0.032 rows=10 loops=1)
Buffers: shared hit=4
Execution Time: 0.052 ms
```

**สรุปผล:** จาก 28.941 ms เหลือ 0.052 ms — เร็วขึ้นกว่า **550 เท่า** เพราะ planner สามารถเดิน index จากค่ามากสุด (เนื่องจาก index เรียง DESC) แล้วหยุดทันทีที่ได้ 10 แถวโดยไม่ต้องแตะแถวอื่นเลย (`Buffers` ลดจาก 541 เหลือ 4)

### ข้อควรระวังเมื่อมี WHERE ร่วมกับ ORDER BY + LIMIT

ถ้า query มีทั้ง filter และ sort ร่วมกัน index ที่ดีที่สุดควรออกแบบให้ **คอลัมน์ที่ใช้ equality filter มาก่อน แล้วตามด้วยคอลัมน์ที่ใช้ sort**

```sql
-- ตัวอย่าง: ดึงคำสั่งซื้อล่าสุดของลูกค้าคนหนึ่ง
-- BEFORE: มีแค่ index เดี่ยวบน order_date
EXPLAIN (ANALYZE, BUFFERS)
SELECT order_id, order_date, status
FROM orders
WHERE customer_id = 4321
ORDER BY order_date DESC
LIMIT 5;
```

```
Limit  (cost=248.35..248.36 rows=5 width=20)
       (actual time=0.412..0.415 rows=5 loops=1)
  ->  Sort  (cost=248.35..248.60 rows=100 width=20)
            (actual time=0.411..0.413 rows=5 loops=1)
       Sort Key: order_date DESC
       Sort Method: top-N heapsort  Memory: 25kB
       ->  Bitmap Heap Scan on orders
            (cost=4.60..246.10 rows=100 width=20)
            (actual time=0.098..0.372 rows=100 loops=1)
            Recheck Cond: (customer_id = 4321)
            ->  Bitmap Index Scan on idx_orders_customer_status
                 (cost=0.00..4.58 rows=100 width=0)
                 (actual time=0.061..0.062 rows=100 loops=1)
                 Index Cond: (customer_id = 4321)
Buffers: shared hit=12
Execution Time: 0.451 ms
```

```sql
-- AFTER: สร้าง composite index ที่ครอบทั้ง filter และ sort
CREATE INDEX idx_orders_customer_date ON orders (customer_id, order_date DESC);
ANALYZE orders;

EXPLAIN (ANALYZE, BUFFERS)
SELECT order_id, order_date, status
FROM orders
WHERE customer_id = 4321
ORDER BY order_date DESC
LIMIT 5;
```

```
Limit  (cost=0.29..1.68 rows=5 width=20)
       (actual time=0.017..0.021 rows=5 loops=1)
  ->  Index Scan using idx_orders_customer_date on orders
       (cost=0.29..27.77 rows=100 width=20)
       (actual time=0.016..0.019 rows=5 loops=1)
       Index Cond: (customer_id = 4321)
Buffers: shared hit=3
Execution Time: 0.038 ms
```

จาก 0.451 ms เหลือ 0.038 ms (เร็วขึ้นราว 12 เท่า) และไม่ต้องมี `Sort` node แยกต่างหากอีกต่อไป เพราะข้อมูลใน index เรียงมาถูกต้องอยู่แล้ว

---

## Step 445: การ optimize JOIN — ลำดับ JOIN, index บน column ที่ join, สถิติที่แม่นยำ

### 1. Index บนคอลัมน์ที่ใช้ join เสมอ

Foreign key ใน PostgreSQL **ไม่ได้สร้าง index ให้อัตโนมัติ** (ต่างจากบาง database) ผู้พัฒนาต้องสร้างเองบนคอลัมน์ที่ใช้ join บ่อย

```sql
-- ตรวจสอบว่ามี index บน foreign key column หรือยัง
SELECT
    conrelid::regclass AS table_name,
    a.attname AS column_name
FROM pg_constraint c
JOIN pg_attribute a
    ON a.attrelid = c.conrelid AND a.attnum = ANY(c.conkey)
WHERE c.contype = 'f'
  AND NOT EXISTS (
      SELECT 1 FROM pg_index i
      WHERE i.indrelid = c.conrelid
        AND a.attnum = ANY(i.indkey)
  );
```

```sql
-- BEFORE: join order_items กับ orders โดยไม่มี index บน order_items.order_id
EXPLAIN (ANALYZE, BUFFERS)
SELECT o.order_id, o.order_date, oi.product_id, oi.quantity
FROM orders o
JOIN order_items oi ON oi.order_id = o.order_id
WHERE o.customer_id = 4321;
```

```
Hash Join  (cost=250.10..4102.60 rows=300 width=20)
           (actual time=1.203..38.451 rows=298 loops=1)
  Hash Cond: (oi.order_id = o.order_id)
  ->  Seq Scan on order_items oi
       (cost=0.00..2750.00 rows=150000 width=12)
       (actual time=0.008..18.902 rows=150000 loops=1)
  ->  Hash  (cost=248.35..248.35 rows=100 width=12)
            (actual time=0.512..0.514 rows=100 loops=1)
       ->  Bitmap Heap Scan on orders o
            (cost=4.60..248.35 rows=100 width=12)
            (actual time=0.098..0.451 rows=100 loops=1)
            Recheck Cond: (customer_id = 4321)
            ->  Bitmap Index Scan on idx_orders_customer_status
                 (cost=0.00..4.58 rows=100 width=0)
Buffers: shared hit=2891
Execution Time: 38.512 ms
```

```sql
-- AFTER: สร้าง index บน order_items.order_id
CREATE INDEX idx_order_items_order_id ON order_items (order_id);
ANALYZE order_items;

EXPLAIN (ANALYZE, BUFFERS)
SELECT o.order_id, o.order_date, oi.product_id, oi.quantity
FROM orders o
JOIN order_items oi ON oi.order_id = o.order_id
WHERE o.customer_id = 4321;
```

```
Nested Loop  (cost=4.89..302.15 rows=300 width=20)
             (actual time=0.045..0.521 rows=298 loops=1)
  ->  Bitmap Heap Scan on orders o
       (cost=4.60..248.35 rows=100 width=12)
       (actual time=0.028..0.198 rows=100 loops=1)
       Recheck Cond: (customer_id = 4321)
       ->  Bitmap Index Scan on idx_orders_customer_status
            (cost=0.00..4.58 rows=100 width=0)
  ->  Index Scan using idx_order_items_order_id on order_items oi
       (cost=0.29..0.53 rows=3 width=12)
       (actual time=0.002..0.003 rows=3 loops=100)
       Index Cond: (order_id = o.order_id)
Buffers: shared hit=421
Execution Time: 0.578 ms
```

**สรุปผล:** จาก 38.512 ms เหลือ 0.578 ms — เร็วขึ้นราว **66 เท่า** planner เปลี่ยนจาก Hash Join (ที่ต้อง Seq Scan ตาราง order_items ทั้งหมด 150,000 แถว) มาเป็น Nested Loop ที่ใช้ index seek สำหรับแต่ละคำสั่งซื้อ ซึ่งเหมาะสมกว่ามากเมื่อจำนวนแถวที่ match ฝั่ง orders มีไม่มาก (100 แถว)

### 2. สถิติที่แม่นยำสำคัญต่อการเลือก join strategy

Planner ตัดสินใจว่าจะใช้ Nested Loop, Hash Join หรือ Merge Join โดยอ้างอิงจาก **estimated row count** ถ้าสถิติล้าสมัย (ไม่ได้ `ANALYZE` หลังข้อมูลเปลี่ยนแปลงมาก) planner อาจเลือกกลยุทธ์ที่ผิดพลาด

```sql
-- จำลองสถานการณ์สถิติล้าสมัย: เพิ่มข้อมูลจำนวนมากโดยไม่ ANALYZE
INSERT INTO order_items (order_id, product_id, quantity, unit_price)
SELECT
    1 + (random() * 49999)::int,
    1 + (random() * 4999)::int,
    1,
    9.99
FROM generate_series(1, 100000) AS g;

-- ตรวจสอบว่าสถิติปัจจุบันคิดว่ามีกี่แถว (อาจยังเป็นตัวเลขเก่า)
SELECT relname, n_live_tup, last_analyze
FROM pg_stat_user_tables
WHERE relname = 'order_items';
```

```
   relname    | n_live_tup |          last_analyze
---------------+------------+---------------------------------
 order_items   |     150000 | 2026-09-25 10:15:02.123456+00
```

```sql
-- ANALYZE เพื่ออัปเดตสถิติให้ตรงกับความจริง (250,000 แถวหลัง insert)
ANALYZE order_items;

SELECT relname, n_live_tup, last_analyze
FROM pg_stat_user_tables
WHERE relname = 'order_items';
```

```
   relname    | n_live_tup | last_analyze
---------------+------------+---------------------------------
 order_items   |     250000 | 2026-09-25 10:22:47.987654+00
```

**หลักปฏิบัติ:** หลังจาก bulk insert/update/delete จำนวนมาก ควรรัน `ANALYZE` เสมอ (autovacuum จะทำให้อัตโนมัติในที่สุด แต่ในสถานการณ์ที่ต้องการผลทันที ควรสั่งเองแบบ manual)

### 3. ลำดับ JOIN ที่เขียนใน SQL ไม่ได้บังคับลำดับจริง

ข้อควรเข้าใจคือ PostgreSQL query planner เป็น **cost-based optimizer** ลำดับของ `JOIN` ที่เขียนใน SQL (เมื่อใช้ implicit join หรือ `JOIN` ปกติ) ไม่ได้บังคับว่า database จะ execute ตามลำดับนั้น planner จะพิจารณาทุกลำดับที่เป็นไปได้ (ภายใต้ `join_collapse_limit`) แล้วเลือกลำดับที่ cost ต่ำที่สุด ผู้พัฒนาจึงควรมุ่งเน้นที่การให้ **สถิติแม่นยำ + index ครบถ้วน** มากกว่าพยายามเรียง `JOIN` ในโค้ดให้ "ดูเหมือน" เร็ว

```sql
-- ตรวจสอบค่า config ที่เกี่ยวกับการพิจารณาลำดับ join
SHOW join_collapse_limit;
-- ค่า default คือ 8 หมายความว่าถ้า join เกิน 8 ตาราง planner
-- จะเริ่มใช้ heuristic แทนการพิจารณาทุกลำดับ (เพื่อจำกัดเวลา planning)
```

---

## Step 446: การ optimize subquery — Correlated Subquery vs JOIN, IN vs EXISTS ทบทวน

### 1. Correlated Subquery ที่ทำงานช้าเพราะรันซ้ำทุกแถว

Correlated subquery คือ subquery ที่อ้างอิงค่าจาก outer query ทำให้ต้องถูก execute ซ้ำสำหรับทุกแถวของ outer query ถ้าไม่มี index รองรับ จะกลายเป็นปัญหา performance รุนแรง

```sql
-- BEFORE: correlated subquery หา "ยอดขายรวมของลูกค้าแต่ละคน" แบบไม่มี index รองรับ subquery
EXPLAIN (ANALYZE, BUFFERS)
SELECT
    c.customer_id,
    c.first_name,
    c.last_name,
    (
        SELECT COALESCE(SUM(oi.quantity * oi.unit_price), 0)
        FROM orders o
        JOIN order_items oi ON oi.order_id = o.order_id
        WHERE o.customer_id = c.customer_id
    ) AS total_spent
FROM customers c
WHERE c.country = 'Thailand'
LIMIT 100;
```

```
Limit  (cost=0.71..892.15 rows=100 width=48)
       (actual time=0.512..145.203 rows=100 loops=1)
  ->  Index Scan using idx_customers_country on customers c
       (cost=0.71..14812.45 rows=1667 width=24)
       (actual time=0.501..145.189 rows=100 loops=1)
       SubPlan 1
         ->  Aggregate
              (cost=8.89..8.90 rows=1 width=32)
              (actual time=1.451..1.452 rows=1 loops=100)
              ->  Nested Loop
                   (cost=0.58..8.87 rows=3 width=12)
                   (actual time=0.021..1.398 rows=6 loops=100)
                   ->  Index Scan using idx_orders_customer_status on orders o
                        (cost=0.29..4.32 rows=5 width=4)
                   ->  Index Scan using idx_order_items_order_id on order_items oi
                        (cost=0.29..0.87 rows=3 width=12)
Buffers: shared hit=1842
Execution Time: 145.241 ms
```

แม้จะมี index รองรับ subquery แล้ว แต่ subquery ก็ยังถูกรันซ้ำ 100 ครั้ง (เท่ากับจำนวนแถวของ outer query) ทำให้ overhead สะสม

### 2. วิธีแก้: แปลงเป็น JOIN + GROUP BY

```sql
-- AFTER: รวม subquery เป็น JOIN แล้ว aggregate ครั้งเดียว
EXPLAIN (ANALYZE, BUFFERS)
SELECT
    c.customer_id,
    c.first_name,
    c.last_name,
    COALESCE(SUM(oi.quantity * oi.unit_price), 0) AS total_spent
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.customer_id
LEFT JOIN order_items oi ON oi.order_id = o.order_id
WHERE c.country = 'Thailand'
GROUP BY c.customer_id, c.first_name, c.last_name
LIMIT 100;
```

```
Limit  (cost=1250.31..1289.77 rows=100 width=48)
       (actual time=12.021..14.895 rows=100 loops=1)
  ->  GroupAggregate
       (cost=1250.31..1908.45 rows=1667 width=48)
       (actual time=12.019..14.881 rows=100 loops=1)
       Group Key: c.customer_id
       ->  Sort
            (cost=1250.31..1287.65 rows=14934 width=20)
            (actual time=12.001..13.102 rows=15021 loops=1)
            Sort Key: c.customer_id
            ->  Hash Right Join
                 (cost=205.20..980.45 rows=14934 width=20)
                 (actual time=1.201..9.845 rows=15021 loops=1)
                 Hash Cond: (o.customer_id = c.customer_id)
                 ->  Hash Join
                      (cost=90.10..820.30 rows=14934 width=12)
                      ->  Seq Scan on order_items oi
                      ->  Hash
                           ->  Seq Scan on orders o
                 ->  Hash
                      ->  Index Scan using idx_customers_country on customers c
Buffers: shared hit=2104
Execution Time: 14.932 ms
```

**สรุปผล:** จาก 145.241 ms เหลือ 14.932 ms — เร็วขึ้นราว **10 เท่า** เพราะแทนที่จะรัน aggregate query ย่อยซ้ำ 100 ครั้ง PostgreSQL รวมข้อมูลด้วยการ join ครั้งเดียวแล้ว group แบบ set-based ซึ่งมีประสิทธิภาพสูงกว่ามากเมื่อข้อมูลมีขนาดใหญ่

> **ข้อสังเกต:** ไม่ใช่ correlated subquery ทุกตัวจะช้ากว่า JOIN เสมอไป ในกรณีที่ outer query มีแถวน้อยมาก (เช่น หลักสิบแถว) และ subquery มี index รองรับดี correlated subquery อาจเร็วพอๆ กันหรือเร็วกว่าด้วยซ้ำ เพราะไม่ต้อง join ข้อมูลก้อนใหญ่ทั้งหมดก่อนแล้วค่อย filter — **จึงต้องวัดผลจริงเสมอ** ไม่ใช่ยึดกฎตายตัว

### 3. ทบทวน IN vs EXISTS

ในเวอร์ชันปัจจุบันของ PostgreSQL (16/17) planner ฉลาดพอที่จะแปลง `IN` และ `EXISTS` ให้กลายเป็นแผนการทำงานที่เหมือนกันในกรณีส่วนใหญ่ (ทั้งคู่มักถูกแปลงเป็น semi-join ภายใน) แต่ยังมีบางกรณีที่ต่างกัน

```sql
-- ค้นหาลูกค้าที่เคยสั่งซื้อสินค้าในหมวดหมู่ที่ระบุ

-- แบบ IN (subquery)
EXPLAIN (ANALYZE, BUFFERS)
SELECT customer_id, first_name, last_name
FROM customers c
WHERE c.customer_id IN (
    SELECT o.customer_id
    FROM orders o
    JOIN order_items oi ON oi.order_id = o.order_id
    JOIN products p ON p.product_id = oi.product_id
    WHERE p.category_id = 5
);
```

```
Hash Semi Join  (cost=3021.45..3890.12 rows=850 width=24)
                (actual time=25.102..48.301 rows=812 loops=1)
  Hash Cond: (c.customer_id = o.customer_id)
  ->  Seq Scan on customers c
  ->  Hash
       ->  Hash Join (customer_id from orders joined with filtered order_items/products)
Buffers: shared hit=3204
Execution Time: 48.352 ms
```

```sql
-- แบบ EXISTS (correlated)
EXPLAIN (ANALYZE, BUFFERS)
SELECT customer_id, first_name, last_name
FROM customers c
WHERE EXISTS (
    SELECT 1
    FROM orders o
    JOIN order_items oi ON oi.order_id = o.order_id
    JOIN products p ON p.product_id = oi.product_id
    WHERE p.category_id = 5
      AND o.customer_id = c.customer_id
);
```

```
Hash Semi Join  (cost=3021.45..3890.12 rows=850 width=24)
                (actual time=24.987..47.982 rows=812 loops=1)
  Hash Cond: (o.customer_id = c.customer_id)
  ->  Hash Join (customer_id from orders joined with filtered order_items/products)
  ->  Hash
       ->  Seq Scan on customers c
Buffers: shared hit=3198
Execution Time: 48.021 ms
```

**สรุป:** ทั้งสองแบบให้ execution plan และเวลาที่ใกล้เคียงกันมาก (planner แปลงเป็น Semi Join เหมือนกัน) — ในกรณีนี้เลือกใช้แบบไหนก็ได้ตามความอ่านง่าย แต่มีข้อแตกต่างสำคัญที่ต้องจำ:

| สถานการณ์ | คำแนะนำ |
|---|---|
| ต้องการเช็คว่า "มีอยู่หรือไม่" อย่างเดียว | `EXISTS` อ่านง่ายกว่าและสื่อความหมายชัดเจนกว่า |
| subquery return ค่า `NULL` ปนอยู่ใน list ที่ใช้กับ `NOT IN` | **อันตราย!** `NOT IN` กับ list ที่มี `NULL` จะไม่ return แถวใดเลย ควรใช้ `NOT EXISTS` แทนเสมอ |
| ต้องการ column เพิ่มเติมจาก subquery | ต้องใช้ `JOIN` ไม่ใช่ `IN`/`EXISTS` เพราะทั้งสองคืนแค่ true/false |

```sql
-- ตัวอย่างอันตรายของ NOT IN กับ NULL
-- สมมติว่า order_items.product_id มีบาง row เป็น NULL (สมมติสถานการณ์)
-- SELECT * FROM products WHERE product_id NOT IN (SELECT product_id FROM order_items);
-- ถ้า order_items.product_id มี NULL แม้แต่ 1 แถว query ข้างต้นจะไม่ return อะไรเลย!

-- วิธีที่ปลอดภัยกว่าเสมอ:
-- SELECT * FROM products p
-- WHERE NOT EXISTS (SELECT 1 FROM order_items oi WHERE oi.product_id = p.product_id);
```

---

## Step 447: work_mem และผลกระทบต่อ Sort/Hash Operation

`work_mem` คือหน่วยความจำสูงสุดที่แต่ละ operation (sort, hash join, hash aggregate) สามารถใช้ได้ **ต่อหนึ่ง operation ต่อหนึ่ง query** (ไม่ใช่ต่อ query ทั้งหมด — query ที่ซับซ้อนมีหลาย sort/hash พร้อมกันได้ ก็จะใช้ work_mem คูณตามจำนวน operation)

ถ้า operation ต้องการ memory มากกว่า `work_mem` ที่กำหนด PostgreSQL จะ**เขียนข้อมูลลง disk ชั่วคราว** (spill to disk) ซึ่งช้ากว่าการทำใน memory มาก

### ดูค่าปัจจุบัน

```sql
SHOW work_mem;
-- ค่า default มักอยู่ที่ 4MB
```

### ตัวอย่าง: Sort ที่ spill ลง disk เพราะ work_mem น้อยเกินไป

```sql
-- ตั้งค่า work_mem ให้น้อยมากเพื่อจำลองปัญหา (session-level เท่านั้น)
SET work_mem = '64kB';

EXPLAIN (ANALYZE, BUFFERS)
SELECT order_id, customer_id, order_date, status, ship_country
FROM orders
ORDER BY ship_country, order_date DESC;
```

```
Sort  (cost=6892.34..7017.34 rows=50000 width=48)
      (actual time=185.201..212.845 rows=50000 loops=1)
  Sort Key: ship_country, order_date DESC
  Sort Method: external merge  Disk: 3120kB
  ->  Seq Scan on orders
       (cost=0.00..1041.00 rows=50000 width=48)
       (actual time=0.010..8.912 rows=50000 loops=1)
Buffers: shared hit=541, temp read=390 written=392
Execution Time: 215.102 ms
```

สังเกตบรรทัดสำคัญ: `Sort Method: external merge  Disk: 3120kB` และ `temp read=390 written=392` — แปลว่า sort operation ต้องเขียนและอ่านข้อมูลชั่วคราวลง disk เพราะ memory ไม่พอ

```sql
-- เพิ่ม work_mem ในระดับ session ให้เพียงพอ
SET work_mem = '16MB';

EXPLAIN (ANALYZE, BUFFERS)
SELECT order_id, customer_id, order_date, status, ship_country
FROM orders
ORDER BY ship_country, order_date DESC;
```

```
Sort  (cost=6892.34..7017.34 rows=50000 width=48)
      (actual time=18.201..21.845 rows=50000 loops=1)
  Sort Key: ship_country, order_date DESC
  Sort Method: quicksort  Memory: 4823kB
  ->  Seq Scan on orders
       (cost=0.00..1041.00 rows=50000 width=48)
       (actual time=0.009..8.734 rows=50000 loops=1)
Buffers: shared hit=541
Execution Time: 22.012 ms
```

**สรุปผล:** จาก 215.102 ms เหลือ 22.012 ms — เร็วขึ้นราว **10 เท่า** เพราะ sort ทำใน memory ล้วนๆ (`Sort Method: quicksort  Memory: 4823kB`) ไม่ต้องแตะ disk เลย

```sql
-- คืนค่า work_mem กลับสู่ค่าเริ่มต้นของ session
RESET work_mem;
```

### session-level vs global (postgresql.conf)

| ระดับ | วิธีตั้งค่า | ขอบเขตผลกระทบ | เหมาะกับ |
|---|---|---|---|
| **Session** | `SET work_mem = '64MB';` | เฉพาะ connection ปัจจุบันเท่านั้น หายไปเมื่อ disconnect | Query เฉพาะทางที่หนักเป็นครั้งคราว เช่น reporting query, batch job |
| **Transaction** | `SET LOCAL work_mem = '64MB';` (ต้องอยู่ใน transaction) | เฉพาะ transaction ปัจจุบัน กลับเป็นค่าเดิมอัตโนมัติเมื่อ commit/rollback | Query ภายใน stored procedure หรือ batch script ที่ไม่อยากกระทบ session อื่น |
| **Global** | แก้ใน `postgresql.conf` แล้ว `pg_ctl reload` หรือ `ALTER SYSTEM SET work_mem = '32MB';` แล้ว `SELECT pg_reload_conf();` | ทุก connection ใหม่หลังจากนี้ | ระบบที่ query ส่วนใหญ่ต้องการ work_mem สูงกว่าปกติ แต่ต้องระวังเรื่อง memory รวมทั้งระบบ |
| **Role-level** | `ALTER ROLE reporting_user SET work_mem = '128MB';` | เฉพาะ role นั้น ทุกครั้งที่ login | แยก workload เช่น reporting user ต้องการ memory เยอะกว่า transactional user |

```sql
-- ตัวอย่างตั้งค่าเฉพาะ role สำหรับงาน reporting
ALTER ROLE reporting_user SET work_mem = '128MB';

-- ตัวอย่างตั้งค่าเฉพาะ transaction เดียว
BEGIN;
SET LOCAL work_mem = '256MB';
-- รัน query หนักๆ ที่นี่
COMMIT;
```

> **คำเตือนสำคัญ:** อย่าตั้ง `work_mem` แบบ global สูงเกินไปโดยไม่คำนวณ เพราะแต่ละ connection สามารถเปิดหลาย operation พร้อมกันได้ (เช่น query ที่มีหลาย sort/hash join) การตั้งค่าสูงเกินไปในระดับ global เมื่อคูณด้วยจำนวน concurrent connection อาจทำให้ระบบ**ใช้ memory เกินและเกิด OOM (Out Of Memory)** ได้ สูตรคร่าวๆ ที่ใช้ประเมิน: `max_connections × work_mem × average_operations_per_query` ไม่ควรเกิน memory ที่จัดสรรไว้สำหรับงานนี้

---

## Step 448: N+1 Query Problem — ปัญหาจาก ORM และวิธีแก้

**N+1 Query Problem** คือรูปแบบปัญหาที่พบบ่อยมากในแอปพลิเคชันที่ใช้ ORM (เช่น Django ORM, SQLAlchemy, Hibernate, Prisma, TypeORM) โดยเกิดจาก:

1. Query แรก (1 query) ดึง list ของ record หลัก เช่น "orders 100 รายการ"
2. จากนั้น loop ในโค้ด application เพื่อดึงข้อมูลที่เกี่ยวข้องของแต่ละ record **ทีละตัว** (N query เพิ่มเติม)

รวมเป็น 1 + N queries ซึ่งเมื่อ N มีค่าสูง (เช่น หลักร้อยหรือหลักพัน) จะทำให้เกิด overhead จากการ round-trip ไปมาระหว่าง application กับ database จำนวนมาก แม้แต่ละ query ย่อยจะเร็วมากก็ตาม

### ตัวอย่างจำลองปัญหา N+1 (เขียนเป็น SQL หลายคำสั่งเพื่อแสดงพฤติกรรม)

```sql
-- Query แรก: ดึง orders 100 รายการล่าสุด (คล้ายกับที่ ORM ทำตอนเรียก Order.objects.all()[:100])
SELECT order_id, customer_id, order_date, status
FROM orders
ORDER BY order_date DESC
LIMIT 100;
```

```sql
-- จากนั้น ORM ทั่วไปมักจะ loop เพื่อดึง order_items ของแต่ละ order แยกกัน
-- (นี่คือรูปแบบที่เกิดขึ้นจริงเมื่อเขียนโค้ดแบบ for order in orders: order.items.all())
-- ตัวอย่างสมมติว่า 100 คำสั่งซื้อ จะมี query แบบนี้เกิดขึ้น 100 ครั้ง!
SELECT * FROM order_items WHERE order_id = 1;
SELECT * FROM order_items WHERE order_id = 2;
SELECT * FROM order_items WHERE order_id = 3;
-- ... (ซ้ำไปเรื่อยๆ จนครบ 100 ครั้ง)
SELECT * FROM order_items WHERE order_id = 100;
```

รวมแล้วต้องยิง **101 queries** เพื่อดึงข้อมูล orders พร้อม items ทั้งที่ทำได้ในคำสั่งเดียว แต่ละ query อาจใช้เวลาแค่ 0.05-0.1 ms แต่เมื่อรวม network round-trip latency (สมมติ 0.5-1 ms ต่อครั้งในสภาพแวดล้อมจริงที่ database อยู่คนละเครื่องกับ application) จะรวมเป็นเวลานับร้อยมิลลิวินาทีถึงหลักวินาที

### วิธีแก้ที่ 1: ใช้ JOIN รวมเป็น Query เดียว

```sql
-- แก้ปัญหาด้วย JOIN: ดึงข้อมูลทั้งหมดในคำสั่งเดียว
EXPLAIN (ANALYZE, BUFFERS)
SELECT
    o.order_id,
    o.customer_id,
    o.order_date,
    o.status,
    oi.order_item_id,
    oi.product_id,
    oi.quantity,
    oi.unit_price
FROM (
    SELECT order_id, customer_id, order_date, status
    FROM orders
    ORDER BY order_date DESC
    LIMIT 100
) o
JOIN order_items oi ON oi.order_id = o.order_id
ORDER BY o.order_date DESC, oi.order_item_id;
```

```
Sort  (cost=458.12..459.87 rows=700 width=48)
      (actual time=1.201..1.245 rows=612 loops=1)
  Sort Key: o.order_date DESC, oi.order_item_id
  ->  Nested Loop
       (cost=0.71..421.34 rows=700 width=48)
       (actual time=0.045..0.998 rows=612 loops=1)
       ->  Limit
            (cost=0.29..1.68 rows=100 width=24)
            (actual time=0.019..0.078 rows=100 loops=1)
            ->  Index Scan using idx_orders_order_date_desc on orders
       ->  Index Scan using idx_order_items_order_id on order_items oi
            (cost=0.29..3.97 rows=7 width=12)
            (actual time=0.002..0.008 rows=6 loops=100)
            Index Cond: (order_id = o.order_id)
Buffers: shared hit=312
Execution Time: 1.312 ms
```

**สรุปผล:** จาก "101 queries" (แต่ละ query มี network round-trip overhead) เหลือเพียง **1 query ใช้เวลา 1.312 ms** ในระดับ database เอง — ถ้ารวม network overhead ที่ลดลงจริงในระบบ production ผลต่างจะยิ่งเห็นชัดกว่านี้มาก (มักลดเวลารวมได้ 90-99% ขึ้นอยู่กับ latency ของ network)

### วิธีแก้ที่ 2: Batch Query ด้วย ANY/IN แทนการ loop

ในกรณีที่ไม่สามารถปรับโครงสร้าง JOIN ได้ง่าย (เช่น ORM บางตัวจัดการผลลัพธ์แบบ nested object ยาก) ทางเลือกคือรวบรวม ID ทั้งหมดก่อน แล้วยิง query เดียวด้วย `= ANY(...)` หรือ `IN (...)`

```sql
-- แทนที่จะ loop ยิง 100 queries แยกกัน
-- ให้รวบรวม order_id ทั้งหมดก่อน แล้วยิงครั้งเดียว
EXPLAIN (ANALYZE, BUFFERS)
SELECT order_item_id, order_id, product_id, quantity, unit_price
FROM order_items
WHERE order_id = ANY(ARRAY[1,2,3,4,5,6,7,8,9,10,
                            11,12,13,14,15,16,17,18,19,20
                            /* ... สมมติมี id ครบ 100 ตัว ... */]);
```

```
Index Scan using idx_order_items_order_id on order_items
                     (cost=0.29..45.12 rows=140 width=12)
                     (actual time=0.021..0.198 rows=132 loops=1)
  Index Cond: (order_id = ANY ('{1,2,3,...,20}'::integer[]))
Buffers: shared hit=45
Execution Time: 0.215 ms
```

แนวทางนี้เหมาะกับสถานการณ์ที่ application layer ต้องการ mapping แบบ `order_id -> [items]` โดยตรง (จับกลุ่มผลลัพธ์เองในโค้ด) ซึ่งเป็นวิธีที่ ORM สมัยใหม่จำนวนมาก (Django's `prefetch_related`, SQLAlchemy's `selectinload`, Prisma's `include`) ใช้ภายในเพื่อแก้ปัญหา N+1 โดยอัตโนมัติ

### หลักการสังเกตว่ากำลังเจอ N+1

- เปิด query log (`log_statement = 'all'` หรือใช้ `pg_stat_statements`) แล้วสังเกตว่ามี query pattern ซ้ำๆ กันจำนวนมากในช่วงเวลาสั้นๆ ที่ต่างกันแค่ค่า parameter
- ตรวจสอบโค้ด ORM ว่ามีการ loop แล้วเรียก relation attribute ข้างใน loop หรือไม่ (เช่น `for order in orders: print(order.customer.name)` ที่ไม่ได้ทำ `select_related`/`join` ไว้ล่วงหน้า)
- ใช้ฟีเจอร์ eager loading ของ ORM เสมอเมื่อรู้ล่วงหน้าว่าจะต้องใช้ relation นั้น (`select_related`, `prefetch_related`, `joinedload`, `include`, `with`)

---

## Step 449: การใช้ pg_stat_statements เพื่อหา Query ที่ช้าที่สุดในระบบจริง

`pg_stat_statements` เป็น extension มาตรฐานของ PostgreSQL ที่เก็บสถิติการทำงานของทุก query ที่เคยรันในระบบ (รวม execution count, total/mean time, rows, และการใช้ buffer) ทำให้สามารถหา query ที่เป็นปัญหาจริงในระบบ production ได้อย่างเป็นระบบ แทนที่จะเดาว่า query ไหน "น่าจะ" ช้า

> บทนี้จะแนะนำการใช้งานเบื้องต้นเท่านั้น รายละเอียดเชิงลึก (การตีความคอลัมน์ทั้งหมด, การตั้งค่า sampling, การ reset สถิติแบบมีกลยุทธ์, การผูกกับ monitoring dashboard) จะอยู่ใน **Part 057 (Monitoring และ Performance Diagnostics)** และ **Part 072 (Production Troubleshooting ขั้นสูง)**

### การเปิดใช้งาน

```sql
-- ต้องเพิ่มใน postgresql.conf ก่อน แล้ว restart database:
-- shared_preload_libraries = 'pg_stat_statements'

-- จากนั้นสร้าง extension ใน database ที่ต้องการ
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
```

### ตัวอย่างการค้นหา query ที่ช้าที่สุด

```sql
-- Top 10 query ที่ใช้ total time สะสมมากที่สุด (มีผลกระทบต่อระบบโดยรวมมากที่สุด)
SELECT
    substring(query, 1, 80) AS query_snippet,
    calls,
    round(total_exec_time::numeric, 2) AS total_ms,
    round(mean_exec_time::numeric, 2) AS mean_ms,
    round(stddev_exec_time::numeric, 2) AS stddev_ms,
    rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;
```

ผลลัพธ์ตัวอย่าง:

```
              query_snippet               | calls | total_ms | mean_ms | stddev_ms |  rows
--------------------------------------------+-------+----------+---------+-----------+--------
 SELECT * FROM order_items WHERE order_id = |  4821 | 15234.50 |    3.16 |      1.02 |  14463
 SELECT o.*, c.* FROM orders o JOIN custom  |   102 |  8912.34 |   87.37 |     45.21 |  10200
 SELECT DATE(order_date), COUNT(*) FROM or  |    45 |  4102.88 |   91.17 |     12.05 |   1350
```

จาก query แรกในตัวอย่างนี้ (`calls = 4821`) เป็นสัญญาณชัดเจนของ **N+1 problem** ที่กล่าวถึงใน Step 448 — query ที่เรียบง่ายมากถูกเรียกซ้ำหลายพันครั้ง แม้แต่ละครั้งจะเร็ว (mean 3.16 ms) แต่ total time สะสมกลับสูงถึง 15 วินาที ซึ่งมากกว่า query ที่ซับซ้อนกว่าด้วยซ้ำ

### หา query ที่ mean time สูงที่สุด (query เดี่ยวที่ช้าที่สุดต่อครั้ง)

```sql
SELECT
    substring(query, 1, 80) AS query_snippet,
    calls,
    round(mean_exec_time::numeric, 2) AS mean_ms,
    round(max_exec_time::numeric, 2) AS max_ms
FROM pg_stat_statements
WHERE calls > 5  -- กรอง query ที่รันน้อยเกินไปจนสถิติไม่น่าเชื่อถือ
ORDER BY mean_exec_time DESC
LIMIT 10;
```

### หา query ที่ใช้ I/O (buffer) มากที่สุด

```sql
SELECT
    substring(query, 1, 80) AS query_snippet,
    calls,
    shared_blks_hit,
    shared_blks_read,
    round(
        shared_blks_hit::numeric /
        NULLIF(shared_blks_hit + shared_blks_read, 0) * 100, 2
    ) AS cache_hit_ratio_pct
FROM pg_stat_statements
ORDER BY shared_blks_read DESC
LIMIT 10;
```

`cache_hit_ratio_pct` ที่ต่ำกว่า ~99% สำหรับ workload แบบ OLTP ทั่วไปมักเป็นสัญญาณว่า query นั้นอ่าน disk บ่อยเกินไป ควรพิจารณาเพิ่ม index หรือปรับ `shared_buffers`

### รีเซ็ตสถิติเมื่อเริ่มการทดสอบรอบใหม่

```sql
-- รีเซ็ตทุกสถิติ (ใช้เมื่อต้องการเริ่มวัดผลใหม่ เช่น หลัง deploy การแก้ไข)
SELECT pg_stat_statements_reset();
```

> **ข้อควรระวัง:** `pg_stat_statements_reset()` จะลบประวัติสถิติทั้งหมดถาวร ควรใช้อย่างระมัดระวังในระบบ production และอาจพิจารณาบันทึกข้อมูลปัจจุบันไว้ก่อน (เช่น copy ไปตารางเก็บประวัติ) หากต้องการเก็บ trend ในระยะยาว

---

## Step 450: แบบฝึกหัดรวม — Optimize Query ช้าแบบครบวงจร

มาถึงขั้นตอนสุดท้าย เราจะนำ query ที่ "ช้า" ซึ่งรวมปัญหาหลายอย่างที่เรียนมาในบทนี้เข้าด้วยกัน แล้ว optimize ทีละขั้นตอนอย่างเป็นระบบ พร้อมวัดผลก่อน-หลังทุกขั้น

### โจทย์: รายงานสรุปยอดขายรายลูกค้าในแต่ละประเทศ พร้อมคำสั่งซื้อล่าสุด

สมมติว่ามี requirement ดังนี้: "แสดงลูกค้าจากประเทศไทยที่มียอดใช้จ่ายรวมมากกว่า 10,000 บาท เรียงตามยอดใช้จ่ายมากไปน้อย พร้อมวันที่สั่งซื้อล่าสุด และจำนวนสินค้าที่เคยซื้อทั้งหมด แสดง 20 อันดับแรก"

### Query ตั้งต้น (มีปัญหาหลายจุด)

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT *
FROM (
    SELECT
        c.customer_id,
        c.first_name,
        c.last_name,
        c.email,
        (
            SELECT SUM(oi.quantity * oi.unit_price)
            FROM orders o
            JOIN order_items oi ON oi.order_id = o.order_id
            WHERE o.customer_id = c.customer_id
        ) AS total_spent,
        (
            SELECT MAX(o.order_date)
            FROM orders o
            WHERE o.customer_id = c.customer_id
        ) AS last_order_date,
        (
            SELECT COUNT(*)
            FROM orders o
            JOIN order_items oi ON oi.order_id = o.order_id
            WHERE o.customer_id = c.customer_id
        ) AS total_items_purchased
    FROM customers c
    WHERE LOWER(c.country) = 'thailand'
) sub
WHERE total_spent > 10000
ORDER BY total_spent DESC
LIMIT 20;
```

**ปัญหาที่มีอยู่ในนี้ (ลองหาก่อนอ่านเฉลย):**

<details>
<summary>คลิกเพื่อดูรายการปัญหาที่พบใน query นี้</summary>

1. `SELECT *` ครอบ subquery — ดึงคอลัมน์เกินความจำเป็น
2. `LOWER(c.country) = 'thailand'` — non-sargable ถ้าไม่มี expression index, ทั้งที่ประเทศใน seed data สะกดตรงกับ `'Thailand'` อยู่แล้วจึงไม่จำเป็นต้องใช้ `LOWER()` เลย
3. มี correlated subquery ถึง **3 ตัว** ที่ต่างก็ query `orders`/`order_items` ซ้ำๆ กันสำหรับลูกค้าแต่ละคน — ควรรวมเป็น JOIN + aggregate ครั้งเดียว
4. `WHERE total_spent > 10000` อยู่นอก subquery ทำให้ database คำนวณ subquery ให้ **ทุกแถว** ของลูกค้าไทยก่อน ค่อยกรองทีหลัง (ควรใช้ `HAVING` ใน query เดียวแทน)
5. ไม่มี index รองรับการค้นหา `country`

</details>

### ขั้นตอนที่ 1: วัดผล baseline

```sql
-- รัน query ตั้งต้นก่อนแก้ไขอะไรเลย เพื่อเป็นค่าอ้างอิง
```

ผลลัพธ์ตัวอย่าง (BEFORE — baseline):

```
Limit  (cost=125890.45..125890.50 rows=20 width=80)
       (actual time=892.451..892.478 rows=20 loops=1)
  ->  Sort
       (cost=125890.45..125894.62 rows=1667 width=80)
       (actual time=892.449..892.462 rows=20 loops=1)
       Sort Key: sub.total_spent DESC
       Sort Method: top-N heapsort  Memory: 28kB
       ->  Subquery Scan on sub
            (cost=0.29..125812.20 rows=1667 width=80)
            (actual time=1.203..889.912 rows=1543 loops=1)
            Filter: (sub.total_spent > '10000'::numeric)
            Rows Removed by Filter: 124
            ->  Seq Scan on customers c
                 (cost=0.29..125562.20 rows=1667 width=48)
                 (actual time=1.198..888.456 rows=1667 loops=1)
                 Filter: (lower((country)::text) = 'thailand'::text)
                 Rows Removed by Filter: 8333
                 SubPlan 1
                 SubPlan 2
                 SubPlan 3
Buffers: shared hit=41892
Execution Time: 892.512 ms
```

**สรุป baseline:** 892.512 ms — ช้ามาก และมี `Buffers: shared hit=41892` (อ่านข้อมูลจำนวนมหาศาล) เพราะ subquery ทั้ง 3 ตัวถูกรันซ้ำสำหรับลูกค้า 1,667 คน (รวมเป็นการ query orders/order_items หลายพันครั้ง)

### ขั้นตอนที่ 2: แก้ปัญหาที่ 1 — เพิ่ม index บน country และตัด LOWER() ที่ไม่จำเป็น

```sql
CREATE INDEX idx_customers_country ON customers (country);
ANALYZE customers;
```

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT *
FROM (
    SELECT
        c.customer_id,
        c.first_name,
        c.last_name,
        c.email,
        (
            SELECT SUM(oi.quantity * oi.unit_price)
            FROM orders o
            JOIN order_items oi ON oi.order_id = o.order_id
            WHERE o.customer_id = c.customer_id
        ) AS total_spent,
        (
            SELECT MAX(o.order_date)
            FROM orders o
            WHERE o.customer_id = c.customer_id
        ) AS last_order_date,
        (
            SELECT COUNT(*)
            FROM orders o
            JOIN order_items oi ON oi.order_id = o.order_id
            WHERE o.customer_id = c.customer_id
        ) AS total_items_purchased
    FROM customers c
    WHERE c.country = 'Thailand'   -- ตัด LOWER() ออก เทียบตรงๆ
) sub
WHERE total_spent > 10000
ORDER BY total_spent DESC
LIMIT 20;
```

```
Limit  (cost=118234.10..118234.15 rows=20 width=80)
       (actual time=845.203..845.231 rows=20 loops=1)
  ->  Sort ...
       ->  Subquery Scan on sub
            ->  Index Scan using idx_customers_country on customers c
                 (actual time=0.045..842.198 rows=1667 loops=1)
                 Index Cond: (country = 'Thailand'::text)
                 SubPlan 1 / SubPlan 2 / SubPlan 3
Buffers: shared hit=39104
Execution Time: 845.267 ms
```

**ผลหลังขั้นตอนที่ 2:** 892.512 ms → 845.267 ms (ดีขึ้นเล็กน้อย ~5%) — แสดงให้เห็นว่าการ scan หา `customers` ไม่ใช่ bottleneck หลัก ปัญหาที่แท้จริงอยู่ที่ subquery ทั้ง 3 ตัว ซึ่งเราต้องแก้ในขั้นถัดไป

### ขั้นตอนที่ 3: แก้ปัญหาหลัก — รวม 3 correlated subquery เป็น JOIN + aggregate เดียว พร้อมใช้ HAVING

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT
    c.customer_id,
    c.first_name,
    c.last_name,
    c.email,
    SUM(oi.quantity * oi.unit_price) AS total_spent,
    MAX(o.order_date) AS last_order_date,
    COUNT(oi.order_item_id) AS total_items_purchased
FROM customers c
JOIN orders o ON o.customer_id = c.customer_id
JOIN order_items oi ON oi.order_id = o.order_id
WHERE c.country = 'Thailand'
GROUP BY c.customer_id, c.first_name, c.last_name, c.email
HAVING SUM(oi.quantity * oi.unit_price) > 10000
ORDER BY total_spent DESC
LIMIT 20;
```

```
Limit  (cost=8921.34..8921.39 rows=20 width=80)
       (actual time=48.102..48.128 rows=20 loops=1)
  ->  Sort
       (cost=8921.34..8934.67 rows=533 width=80)
       (actual time=48.100..48.112 rows=20 loops=1)
       Sort Key: (sum((oi.quantity * oi.unit_price))) DESC
       Sort Method: top-N heapsort  Memory: 30kB
       ->  GroupAggregate
            (cost=8654.21..8901.45 rows=533 width=80)
            (actual time=42.301..47.892 rows=1543 loops=1)
            Group Key: c.customer_id
            Filter: (sum((oi.quantity * oi.unit_price)) > '10000'::numeric)
            Rows Removed by Filter: 124
            ->  Sort
                 (cost=8654.21..8687.10 rows=13157 width=20)
                 (actual time=42.045..44.201 rows=13890 loops=1)
                 Sort Key: c.customer_id
                 ->  Nested Loop
                      (cost=8.90..7789.34 rows=13157 width=20)
                      (actual time=0.089..38.451 rows=13890 loops=1)
                      ->  Nested Loop
                           (cost=8.61..2891.20 rows=1667 width=12)
                           (actual time=0.045..8.201 rows=1667 loops=1)
                           ->  Index Scan using idx_customers_country on customers c
                                Index Cond: (country = 'Thailand'::text)
                           ->  Index Scan using idx_orders_customer_status on orders o
                                Index Cond: (customer_id = c.customer_id)
                      ->  Index Scan using idx_order_items_order_id on order_items oi
                           Index Cond: (order_id = o.order_id)
Buffers: shared hit=8934
Execution Time: 48.201 ms
```

**ผลหลังขั้นตอนที่ 3:** 845.267 ms → 48.201 ms — เร็วขึ้นราว **17.5 เท่า** จากการยุบ 3 subquery ที่รันซ้ำ เหลือเพียง JOIN + GroupAggregate ครั้งเดียว

### ขั้นตอนที่ 4: เลือกเฉพาะคอลัมน์ที่จำเป็น (ตัด email ออกถ้าไม่ได้ใช้แสดงผลจริง — สมมติ requirement ต้องการ email ด้วย จึงคงไว้ แต่ตรวจสอบว่าไม่มี `SELECT *` หลงเหลือ)

จาก query ในขั้นตอนที่ 3 เราเลือกคอลัมน์ที่ต้องใช้ตรงตาม requirement อยู่แล้ว (ไม่มี `SELECT *`) จึงไม่มีอะไรต้องแก้เพิ่มในจุดนี้ — นี่คือตัวอย่างที่ดีว่าการ optimize ไม่จำเป็นต้องทำครบทุกเทคนิคเสมอไป ต้องพิจารณาตามบริบทจริงของแต่ละ query

### ขั้นตอนที่ 5: ตรวจสอบว่า work_mem เพียงพอสำหรับ Sort/GroupAggregate หรือไม่

```sql
SHOW work_mem;
-- ตรวจสอบใน EXPLAIN ANALYZE ว่ามี "Sort Method: external merge" หรือไม่
-- จากผลลัพธ์ขั้นตอนที่ 3 เราเห็นว่าเป็น top-N heapsort ในหน่วยความจำล้วนๆ (ไม่มี Disk)
-- แปลว่า work_mem ปัจจุบันเพียงพอแล้ว ไม่จำเป็นต้องปรับเพิ่ม
```

### สรุปผลการ optimize ทั้งหมด

| ขั้นตอน | การแก้ไข | Execution Time | อัตราเร่ง (เทียบ baseline) |
|---|---|---:|---:|
| 0 (Baseline) | Query ตั้งต้น | 892.512 ms | 1x |
| 1 | เพิ่ม index บน `country` + ตัด `LOWER()` ที่ไม่จำเป็น | 845.267 ms | 1.06x |
| 2 | รวม correlated subquery 3 ตัวเป็น JOIN + `HAVING` | 48.201 ms | **18.5x** |
| 3 | ตรวจสอบคอลัมน์ที่เลือกและ work_mem | 48.201 ms (ไม่เปลี่ยน — อยู่ในระดับที่ดีแล้ว) | 18.5x |

**บทเรียนสำคัญ:** ไม่ใช่ทุกเทคนิคการ optimize จะให้ผลเท่ากัน — ในกรณีนี้ index บน `country` ให้ผลเพียงเล็กน้อย (5-6%) เพราะไม่ใช่ bottleneck หลัก ในขณะที่การแก้ correlated subquery ให้ผลลัพธ์ที่เปลี่ยนเกมอย่างสิ้นเชิง (18.5 เท่า) นี่คือเหตุผลที่ **Step 441** เน้นย้ำเรื่อง "วัดก่อนแก้เสมอ" — การเดาว่าอะไรคือปัญหาโดยไม่วัดผล อาจทำให้เสียเวลาไปกับจุดที่ไม่ใช่ bottleneck ที่แท้จริง

---

## สรุปท้ายบท

บทนี้ครอบคลุมเทคนิคการ optimize query ที่ใช้งานได้จริงในระบบ production ตั้งแต่หลักการพื้นฐานไปจนถึงการวิเคราะห์ query ที่ซับซ้อนแบบครบวงจร

### Checklist การ optimize query

เมื่อเจอ query ที่ทำงานช้า ให้ไล่ตรวจสอบตามลำดับต่อไปนี้:

- [ ] **วัดผลก่อนเสมอ** — รัน `EXPLAIN (ANALYZE, BUFFERS)` เพื่อดู execution plan จริง อย่าเดาว่าอะไรคือปัญหา
- [ ] **ตรวจสอบ estimated rows vs actual rows** — ถ้าต่างกันมาก ให้รัน `ANALYZE` เพื่ออัปเดตสถิติ
- [ ] **ตรวจสอบ `WHERE` clause ว่า sargable หรือไม่** — หลีกเลี่ยงการครอบฟังก์ชันบนคอลัมน์ที่มี index (`LOWER()`, `DATE()`, การบวกเลข, การแปลง type) ถ้าจำเป็นต้องใช้ฟังก์ชัน ให้สร้าง expression index
- [ ] **หลีกเลี่ยง `SELECT *`** — เลือกเฉพาะคอลัมน์ที่ใช้จริง เพื่อลด I/O, network transfer และเพิ่มโอกาสใช้ Index Only Scan
- [ ] **มี index รองรับ `ORDER BY` + `LIMIT`** — โดยเฉพาะเมื่อดึง "N รายการล่าสุด/แรกสุด" ควรมี index ที่เรียงลำดับตรงกับ query
- [ ] **มี index บนคอลัมน์ที่ใช้ `JOIN`** — โดยเฉพาะ foreign key columns ซึ่ง PostgreSQL ไม่สร้าง index ให้อัตโนมัติ
- [ ] **สถิติของตารางเป็นปัจจุบัน** — รัน `ANALYZE` หลัง bulk insert/update/delete และตรวจสอบว่า autovacuum ทำงานปกติ
- [ ] **พิจารณาแปลง correlated subquery เป็น JOIN** เมื่อ subquery ถูกเรียกซ้ำจำนวนมาก (แต่ต้องวัดผลจริง เพราะบางกรณี subquery ก็เร็วกว่า)
- [ ] **ใช้ `EXISTS`/`NOT EXISTS` แทน `IN`/`NOT IN`** เมื่อ subquery อาจมีค่า `NULL` ปน เพื่อหลีกเลี่ยง bug ที่ query ไม่ return ผลลัพธ์เลย
- [ ] **ตรวจสอบ `work_mem`** — ถ้าเห็น `Sort Method: external merge` หรือ `Disk:` ใน execution plan แปลว่า sort/hash กำลัง spill ลง disk ควรพิจารณาเพิ่ม `work_mem` (ระดับ session/transaction/role ตามความเหมาะสม)
- [ ] **ตรวจหาปัญหา N+1** — สังเกต query pattern ที่ถูกเรียกซ้ำจำนวนมากด้วย parameter ต่างกัน แก้ด้วย JOIN หรือ batch query (`= ANY(...)`) หรือใช้ eager loading ของ ORM
- [ ] **ใช้ `pg_stat_statements`** เพื่อหา query ที่ส่งผลกระทบต่อระบบโดยรวมมากที่สุด (ดูทั้ง `total_exec_time` และ `mean_exec_time`) แทนที่จะไล่ดูทีละ query ในโค้ด
- [ ] **แก้ทีละจุด วัดผลทุกครั้ง** — อย่าแก้หลายอย่างพร้อมกัน เพราะจะไม่รู้ว่าอะไรคือสิ่งที่ทำให้ดีขึ้นจริง และบันทึกผลเปรียบเทียบก่อน-หลังไว้เสมอ

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1

จงอธิบายว่าทำไม query ต่อไปนี้ถึงไม่ใช้ index แม้จะมี `CREATE INDEX idx_products_price ON products (unit_price);` อยู่แล้ว

```sql
SELECT product_id, product_name, unit_price
FROM products
WHERE unit_price * 1.07 > 1000;  -- คำนวณราคารวม VAT
```

<details>
<summary>เฉลยแบบฝึกหัดที่ 1</summary>

เพราะ `unit_price * 1.07` เป็นการคำนวณครอบคอลัมน์ที่มี index ทำให้ query ไม่ sargable — planner ไม่สามารถใช้ btree index ที่สร้างจากค่า `unit_price` ดิบๆ มาช่วยหาค่าที่ผ่านการคูณแล้วได้ ต้องอ่านทุกแถวมาคำนวณก่อนเปรียบเทียบ

วิธีแก้: ย้ายค่าคงที่ไปอีกฝั่งของเงื่อนไข เพื่อให้คอลัมน์อยู่เดี่ยวๆ

```sql
SELECT product_id, product_name, unit_price
FROM products
WHERE unit_price > 1000 / 1.07;
```

ตอนนี้ `unit_price` ไม่ถูกครอบด้วยฟังก์ชันหรือการคำนวณใดๆ แล้ว planner จึงสามารถใช้ `idx_products_price` ได้ตามปกติ

</details>

---

### แบบฝึกหัดที่ 2

Query ด้านล่างดึงข้อมูลสินค้า 10 รายการที่ stock เหลือน้อยที่สุด แต่ query นี้ช้าเพราะไม่มี index รองรับ จงเขียนคำสั่ง `CREATE INDEX` ที่เหมาะสมที่สุด

```sql
SELECT product_id, product_name, stock_quantity
FROM products
WHERE is_active = true
ORDER BY stock_quantity ASC
LIMIT 10;
```

<details>
<summary>เฉลยแบบฝึกหัดที่ 2</summary>

ควรสร้าง composite index ที่มีคอลัมน์ filter (`is_active`) ก่อน ตามด้วยคอลัมน์ sort (`stock_quantity`):

```sql
CREATE INDEX idx_products_active_stock
    ON products (is_active, stock_quantity ASC);

ANALYZE products;
```

ด้วย index นี้ planner จะสามารถ seek ไปยังกลุ่ม `is_active = true` แล้วอ่านตามลำดับ `stock_quantity` ที่เรียงมาให้แล้วโดยตรง ไม่ต้อง sort เพิ่มและไม่ต้องอ่านแถวที่ `is_active = false` เลย

</details>

---

### แบบฝึกหัดที่ 3

จงอธิบายความแตกต่างระหว่าง `Index Scan` กับ `Index Only Scan` ในผลลัพธ์ของ `EXPLAIN ANALYZE` และบอกเงื่อนไขที่ต้องมีเพื่อให้เกิด `Index Only Scan`

<details>
<summary>เฉลยแบบฝึกหัดที่ 3</summary>

- **Index Scan**: planner ใช้ index เพื่อหาตำแหน่งของแถวที่ตรงเงื่อนไข แต่ยังต้องกลับไปอ่านข้อมูลจากตาราง heap เพิ่มเติม (เพราะ query ต้องการคอลัมน์ที่ไม่ได้อยู่ใน index)
- **Index Only Scan**: planner สามารถตอบ query ได้ครบถ้วนจากข้อมูลใน index อย่างเดียว โดยไม่ต้องแตะ heap เลย (สังเกตจาก `Heap Fetches: 0` ใน execution plan)

เงื่อนไขที่ต้องมี:
1. ทุกคอลัมน์ที่ query เลือก (`SELECT`) และใช้ใน `WHERE` ต้องอยู่ใน index (อาจใช้ `INCLUDE` เพื่อเพิ่มคอลัมน์เข้า index โดยไม่ต้องอยู่ในส่วน key ก็ได้)
2. Table ต้องผ่านการ `VACUUM` เพื่อให้ visibility map เป็นปัจจุบัน — ถ้า page มีแถวที่ถูกแก้ไขล่าสุดยังไม่ถูก vacuum planner อาจยังต้องเช็ค visibility จาก heap อยู่ดี (ทำให้เห็น `Heap Fetches` มากกว่า 0)

</details>

---

### แบบฝึกหัดที่ 4

Query ด้านล่างพยายามหาสินค้าที่ **ไม่เคย** ถูกสั่งซื้อเลย แต่มี bug ที่ทำให้บางครั้ง return ผลลัพธ์ว่างเปล่าผิดปกติ จงหา bug และแก้ไข

```sql
SELECT product_id, product_name
FROM products
WHERE product_id NOT IN (SELECT product_id FROM order_items);
```

<details>
<summary>เฉลยแบบฝึกหัดที่ 4</summary>

Bug: ถ้าคอลัมน์ `order_items.product_id` มีค่า `NULL` แม้แต่แถวเดียว (เช่น ข้อมูลไม่สมบูรณ์ หรือถูกลบ FK constraint ออกชั่วคราว) คำสั่ง `NOT IN` จะ return ผลลัพธ์เป็น**ว่างเปล่าทั้งหมด** เพราะการเปรียบเทียบกับ `NULL` ด้วย `<>` ให้ผลเป็น `UNKNOWN` เสมอ ทำให้เงื่อนไขทั้งหมดกลายเป็นเท็จ

วิธีแก้: ใช้ `NOT EXISTS` แทน ซึ่งไม่มีปัญหานี้

```sql
SELECT p.product_id, p.product_name
FROM products p
WHERE NOT EXISTS (
    SELECT 1
    FROM order_items oi
    WHERE oi.product_id = p.product_id
);
```

หรืออีกทางเลือกคือกรอง `NULL` ออกจาก subquery ก่อน (แต่ `NOT EXISTS` เป็นทางที่ปลอดภัยและอ่านง่ายกว่าเสมอ):

```sql
SELECT product_id, product_name
FROM products
WHERE product_id NOT IN (
    SELECT product_id FROM order_items WHERE product_id IS NOT NULL
);
```

</details>

---

### แบบฝึกหัดที่ 5

ในการรัน `EXPLAIN ANALYZE` ของ query หนึ่ง พบว่ามีบรรทัด `Sort Method: external merge  Disk: 15234kB` จงอธิบายว่าเกิดอะไรขึ้น และมีวิธีแก้ปัญหากี่แบบ

<details>
<summary>เฉลยแบบฝึกหัดที่ 5</summary>

`Sort Method: external merge  Disk: 15234kB` หมายความว่าข้อมูลที่ต้อง sort มีขนาดใหญ่เกินกว่า `work_mem` ที่กำหนดไว้ PostgreSQL จึงต้องเขียนข้อมูลบางส่วนลง disk ชั่วคราว (temp file) แล้วทำ merge sort จาก disk ซึ่งช้ากว่าการ sort ใน memory ล้วนๆ (`quicksort`) มาก

วิธีแก้มีหลายทาง:

1. **เพิ่ม `work_mem`** ในระดับที่เหมาะสม (session, transaction, หรือ role) ให้เพียงพอกับขนาดข้อมูลที่ต้อง sort เช่น `SET work_mem = '32MB';`
2. **ลดจำนวนแถวที่ต้อง sort** ตั้งแต่ต้น เช่น กรองข้อมูลด้วย `WHERE` ให้เหลือน้อยลงก่อน sort หรือใช้ index ที่เรียงลำดับไว้แล้วเพื่อไม่ต้อง sort เลย (ดู Step 444)
3. **ลดจำนวนคอลัมน์ที่ sort พา** เช่น sort เฉพาะ primary key แล้วค่อย join กลับมาดึงคอลัมน์อื่นทีหลัง (ช่วยลดขนาดข้อมูลต่อแถวที่ต้องเก็บระหว่าง sort)

</details>

---

### แบบฝึกหัดที่ 6

Query ต่อไปนี้เขียนโดยนักพัฒนาที่ใช้ ORM แล้วพบว่า log แสดง query ที่เหมือนกันเกือบทุกอย่าง (ต่างแค่ `customer_id`) ถูกยิงซ้ำหลายร้อยครั้งติดกันในเวลาไม่กี่วินาที จงระบุว่านี่คือปัญหาอะไร และเสนอวิธีแก้ 2 แบบ

```sql
SELECT * FROM customers WHERE customer_id = 1;
SELECT * FROM customers WHERE customer_id = 2;
SELECT * FROM customers WHERE customer_id = 3;
-- ... ซ้ำไปเรื่อยๆ หลายร้อยครั้ง
```

<details>
<summary>เฉลยแบบฝึกหัดที่ 6</summary>

นี่คือปัญหา **N+1 Query Problem** ที่มักเกิดจากการ loop ในโค้ด application เพื่อดึงข้อมูลทีละแถว แทนที่จะดึงมาทีเดียวทั้งหมด (เช่น เกิดจากโค้ด `for order in orders: order.customer` ที่ ORM ไม่ได้ทำ eager loading ไว้ล่วงหน้า)

วิธีแก้ที่ 1 — ใช้ `= ANY(...)` รวบรวม ID ทั้งหมดแล้วยิงครั้งเดียว:

```sql
SELECT * FROM customers
WHERE customer_id = ANY(ARRAY[1,2,3, /* ... */ ]);
```

วิธีแก้ที่ 2 — ถ้าข้อมูลต้นทางมาจากตารางอื่น (เช่น orders) ให้ใช้ `JOIN` รวมเป็น query เดียวตั้งแต่ต้น แทนที่จะดึง orders มาก่อนแล้วค่อยวน loop ดึง customer แยก:

```sql
SELECT o.order_id, o.order_date, c.customer_id, c.first_name, c.last_name
FROM orders o
JOIN customers c ON c.customer_id = o.customer_id
WHERE o.order_date >= CURRENT_DATE - INTERVAL '7 days';
```

ในระดับ ORM ควรใช้ฟีเจอร์ eager loading ที่ ORM มีให้ (เช่น `select_related`, `prefetch_related`, `joinedload`, `include`) เพื่อป้องกันปัญหานี้ตั้งแต่ต้น

</details>

---

### แบบฝึกหัดที่ 7

จงเขียนคำสั่ง SQL เพื่อค้นหา 5 query ที่มี `calls` (จำนวนครั้งที่ถูกเรียก) มากที่สุดในระบบ โดยใช้ `pg_stat_statements` และอธิบายว่าทำไมค่านี้ถึงช่วยตรวจจับปัญหา N+1 ได้

<details>
<summary>เฉลยแบบฝึกหัดที่ 7</summary>

```sql
SELECT
    substring(query, 1, 100) AS query_snippet,
    calls,
    round(mean_exec_time::numeric, 3) AS mean_ms,
    round(total_exec_time::numeric, 2) AS total_ms
FROM pg_stat_statements
ORDER BY calls DESC
LIMIT 5;
```

ค่า `calls` ที่สูงผิดปกติ (เช่น หลักพันหรือหลักหมื่นครั้ง) โดยเฉพาะเมื่อ query มีรูปแบบง่ายๆ (เช่น `SELECT * FROM table WHERE id = $1`) เป็นสัญญาณบ่งชี้ที่ชัดเจนของปัญหา N+1 เพราะในระบบที่ออกแบบมาอย่างเหมาะสม query ที่ค้นหาด้วย primary key ควรถูกเรียกในจำนวนที่สอดคล้องกับ traffic จริง ไม่ใช่ถูกเรียกซ้ำๆ เป็นชุดใหญ่สำหรับแต่ละ request เดียว การดูค่า `calls` ควบคู่กับ `total_exec_time` ช่วยยืนยันว่า query ที่ถูกเรียกบ่อยเหล่านี้ส่งผลกระทบสะสมต่อระบบมากน้อยเพียงใด

</details>

---

### แบบฝึกหัดที่ 8

Query ต่อไปนี้ใช้ correlated subquery หายอดขายรวมของลูกค้าแต่ละคน จงแปลงเป็น query ที่ใช้ `JOIN` แทน โดยให้ผลลัพธ์เหมือนเดิมทุกประการ (รวมถึงลูกค้าที่ยังไม่เคยสั่งซื้อเลยต้องแสดง `total_spent = 0`)

```sql
SELECT
    customer_id,
    first_name,
    (SELECT COALESCE(SUM(oi.quantity * oi.unit_price), 0)
     FROM orders o
     JOIN order_items oi ON oi.order_id = o.order_id
     WHERE o.customer_id = c.customer_id) AS total_spent
FROM customers c;
```

<details>
<summary>เฉลยแบบฝึกหัดที่ 8</summary>

จุดสำคัญคือต้องใช้ `LEFT JOIN` (ไม่ใช่ `JOIN` ธรรมดา) เพื่อให้ลูกค้าที่ไม่มีคำสั่งซื้อยังคงปรากฏในผลลัพธ์ พร้อมค่า `total_spent = 0` แทนที่จะถูกตัดออกไป

```sql
SELECT
    c.customer_id,
    c.first_name,
    COALESCE(SUM(oi.quantity * oi.unit_price), 0) AS total_spent
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.customer_id
LEFT JOIN order_items oi ON oi.order_id = o.order_id
GROUP BY c.customer_id, c.first_name;
```

ข้อสังเกต: ต้องใช้ `LEFT JOIN` ทั้งสองจุด (customers→orders และ orders→order_items) เพราะแม้ลูกค้าจะมี order แต่ order นั้นอาจไม่มี order_items ก็ได้ (แม้ในทางธุรกิจจะไม่ควรเกิดขึ้น แต่ในเชิงโครงสร้างข้อมูลต้องรองรับกรณีนี้ไว้ด้วยเพื่อความถูกต้อง) และ `COALESCE` ยังจำเป็นอยู่เพราะ `SUM()` ของแถวที่ไม่มีข้อมูลจะได้ `NULL` ไม่ใช่ `0`

</details>

---

### แบบฝึกหัดที่ 9

จงอธิบายความแตกต่างระหว่างการตั้งค่า `SET work_mem = '64MB';` กับ `SET LOCAL work_mem = '64MB';` และบอกว่าแบบไหนเหมาะกับการใช้งานภายใน function หรือ stored procedure มากกว่า พร้อมเหตุผล

<details>
<summary>เฉลยแบบฝึกหัดที่ 9</summary>

- `SET work_mem = '64MB';` — เปลี่ยนค่าใน session ปัจจุบันไปตลอดจนกว่าจะมีการ `RESET` หรือ disconnect ถ้าเรียกใน transaction ที่ rollback ค่านี้จะ**ไม่ถูก rollback** กลับ (การตั้งค่าแบบนี้ไม่ได้อยู่ภายใต้ transaction control)
- `SET LOCAL work_mem = '64MB';` — เปลี่ยนค่าเฉพาะภายใน transaction ปัจจุบันเท่านั้น เมื่อ `COMMIT` หรือ `ROLLBACK` ค่าจะกลับเป็นค่าเดิมโดยอัตโนมัติ ต้องใช้ภายใน transaction เท่านั้น (นอก transaction จะมีผลแค่ statement เดียว)

สำหรับการใช้งานภายใน function หรือ stored procedure ควรใช้ `SET LOCAL` เพราะ:
1. ป้องกันไม่ให้การตั้งค่าไปกระทบ query อื่นๆ ที่รันต่อจากนั้นใน connection เดียวกัน (โดยเฉพาะเมื่อใช้ connection pooling ที่ connection เดิมอาจถูกนำไปใช้กับ request อื่นต่อ)
2. คืนค่ากลับอัตโนมัติเมื่อจบ transaction ไม่ต้องเขียนโค้ดมา `RESET` เอง ลดโอกาสเกิด bug ที่ค่าตั้งค้างอยู่โดยไม่ตั้งใจ

</details>

---

### แบบฝึกหัดที่ 10

จากตาราง `orders` และ `order_items` ที่เตรียมไว้ในบทนี้ จงเขียน query เพื่อหา "5 สินค้าที่ทำยอดขายรวม (quantity × unit_price) สูงสุด เฉพาะคำสั่งซื้อที่มีสถานะ `delivered` เท่านั้น" แล้ววิเคราะห์ execution plan ว่าควรมี index อะไรเพิ่มเพื่อให้ query นี้ทำงานได้เร็วที่สุด

<details>
<summary>เฉลยแบบฝึกหัดที่ 10</summary>

Query ที่ตอบโจทย์:

```sql
SELECT
    p.product_id,
    p.product_name,
    SUM(oi.quantity * oi.unit_price) AS total_revenue
FROM order_items oi
JOIN orders o ON o.order_id = oi.order_id
JOIN products p ON p.product_id = oi.product_id
WHERE o.status = 'delivered'
GROUP BY p.product_id, p.product_name
ORDER BY total_revenue DESC
LIMIT 5;
```

Index ที่ควรมีเพื่อประสิทธิภาพสูงสุด:

```sql
-- 1. Index บน orders.status เพื่อกรองเฉพาะคำสั่งซื้อที่ delivered ได้เร็ว
CREATE INDEX idx_orders_status ON orders (status);

-- 2. Index บน order_items.order_id (อาจมีอยู่แล้วจากบทก่อนหน้า)
--    ใช้สำหรับ join กลับไปหา order ของแต่ละ order_item
CREATE INDEX IF NOT EXISTS idx_order_items_order_id
    ON order_items (order_id);

-- 3. Index บน order_items.product_id เพื่อช่วย join กับ products
--    และช่วยให้ GROUP BY p.product_id ทำงานได้เร็วขึ้นเมื่อ aggregate
CREATE INDEX idx_order_items_product_id ON order_items (product_id);

ANALYZE orders;
ANALYZE order_items;
```

หลังจากรัน `ANALYZE` แล้วตรวจสอบด้วย `EXPLAIN (ANALYZE, BUFFERS)` ควรเห็น planner เปลี่ยนจาก Seq Scan บน `orders` มาเป็น Index Scan/Bitmap Index Scan บน `idx_orders_status` และใช้ Index Scan บน `idx_order_items_order_id` สำหรับการ join กับ order_items แทนการ Seq Scan ทั้งตารางซึ่งมีข้อมูลจำนวนมาก (150,000+ แถว) ทำให้เวลาการทำงานลดลงอย่างมีนัยสำคัญ โดยเฉพาะเมื่อสัดส่วนของ order ที่มีสถานะ `delivered` เป็นเพียงส่วนหนึ่งของข้อมูลทั้งหมด (ประมาณ 1 ใน 5 จากการสุ่มสถานะทั้ง 5 แบบในขั้นตอนเตรียมข้อมูล)

</details>

---

## บทถัดไป

เมื่อเข้าใจเทคนิคการ optimize query ในระดับ SQL แล้ว บทถัดไปจะพาไปสู่การเขียนโปรแกรมภายในฐานข้อมูลด้วย PL/pgSQL ซึ่งเป็นพื้นฐานสำคัญสำหรับการสร้าง function, trigger และ stored procedure ที่ซับซ้อนมากขึ้น

**บทถัดไป:** [Part 046: PL/pgSQL พื้นฐาน](./part-046-plpgsql-basics.md)
