# Part 044: Query Planner และการอ่าน EXPLAIN / EXPLAIN ANALYZE

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับสูง | Part 044

---

## เป้าหมายการเรียนรู้

เมื่อเรียนจบ Part นี้ ผู้เรียนจะสามารถ:

- อธิบายได้ว่า **Query Planner** (หรือ Query Optimizer) ของ PostgreSQL ทำงานอย่างไร และเหตุใด SQL ที่เขียนแบบ declarative จึงต้องผ่านการ "แปลง" เป็นแผนการทำงาน (execution plan) ก่อนรันจริง
- อ่านผลลัพธ์จาก `EXPLAIN` แบบพื้นฐานได้ เข้าใจความหมายของ `cost`, `rows`, `width`
- ใช้ `EXPLAIN ANALYZE` เพื่อดูเวลาทำงานจริง และรู้ข้อควรระวังเมื่อใช้กับคำสั่งที่มี side effect เช่น `UPDATE`/`DELETE`
- จำแนก Node type หลักที่พบบ่อยได้: `Seq Scan`, `Index Scan`, `Index Only Scan`, `Bitmap Heap/Index Scan`
- เข้าใจ Join algorithm ทั้งสามแบบ (`Nested Loop`, `Hash Join`, `Merge Join`) และรู้ว่า Planner เลือกใช้แบบไหนเมื่อไหร่
- อ่าน Sort และ Aggregate node ได้ และรู้จักปัญหา external sort (`Disk` vs `Memory`) เมื่อ `work_mem` ไม่พอ
- เปรียบเทียบ **planned rows** กับ **actual rows** เพื่อวินิจฉัยปัญหา statistics ที่ล้าสมัย และผลกระทบต่อแผนที่ Planner เลือก
- ใช้คำสั่ง `ANALYZE`, เข้าใจ system view `pg_stats`, และค่า `default_statistics_target`
- ใช้ `EXPLAIN (FORMAT JSON/YAML)` และเข้าใจแนวคิดของเครื่องมือ visualize เช่น explain.dalibo.com (PEV2)
- วินิจฉัยแผนการทำงานของ query ที่ซับซ้อนในระบบ e-commerce จริงได้ด้วยตนเอง

---

## เตรียมข้อมูล

Part นี้ใช้ schema ฐาน e-commerce เดียวกับ Part 041 แต่จะเพิ่มปริมาณข้อมูลด้วย `generate_series` ให้มากพอ (หลักพันแถว) เพื่อให้ Query Planner ต้องตัดสินใจเลือกแผนที่ "คุ้มค่า" จริง ๆ ไม่ใช่แค่ตารางว่าง ๆ ที่ Seq Scan ก็เร็วอยู่แล้ว

```sql
-- ==========================================================
-- 1) สร้างตาราง (schema เดียวกับ Part 041)
-- ==========================================================
DROP TABLE IF EXISTS order_items, orders, customers, products, suppliers, categories CASCADE;

CREATE TABLE categories (
    category_id         SERIAL PRIMARY KEY,
    category_name        VARCHAR(100) NOT NULL,
    parent_category_id   INTEGER REFERENCES categories(category_id)
);

CREATE TABLE suppliers (
    supplier_id     SERIAL PRIMARY KEY,
    supplier_name   VARCHAR(150) NOT NULL,
    country         VARCHAR(60)
);

CREATE TABLE products (
    product_id       SERIAL PRIMARY KEY,
    product_name     VARCHAR(150) NOT NULL,
    category_id      INTEGER REFERENCES categories(category_id),
    supplier_id      INTEGER REFERENCES suppliers(supplier_id),
    unit_price       NUMERIC(10,2) NOT NULL,
    stock_quantity   INTEGER NOT NULL DEFAULT 0,
    is_active        BOOLEAN NOT NULL DEFAULT true
);

CREATE TABLE customers (
    customer_id    SERIAL PRIMARY KEY,
    first_name     VARCHAR(60) NOT NULL,
    last_name      VARCHAR(60) NOT NULL,
    email          VARCHAR(150) UNIQUE,
    country        VARCHAR(60),
    signup_date    DATE NOT NULL DEFAULT CURRENT_DATE
);

CREATE TABLE orders (
    order_id       SERIAL PRIMARY KEY,
    customer_id    INTEGER REFERENCES customers(customer_id),
    order_date     TIMESTAMPTZ NOT NULL DEFAULT now(),
    status         VARCHAR(20) NOT NULL DEFAULT 'pending',
    ship_country   VARCHAR(60)
);

CREATE TABLE order_items (
    order_item_id   SERIAL PRIMARY KEY,
    order_id        INTEGER REFERENCES orders(order_id),
    product_id      INTEGER REFERENCES products(product_id),
    quantity        INTEGER NOT NULL CHECK (quantity > 0),
    unit_price      NUMERIC(10,2) NOT NULL
);

-- ==========================================================
-- 2) ข้อมูลอ้างอิงขนาดเล็ก: categories, suppliers
-- ==========================================================
INSERT INTO categories (category_name, parent_category_id) VALUES
    ('Electronics', NULL),
    ('Computers', 1),
    ('Mobile Phones', 1),
    ('Home Appliances', NULL),
    ('Kitchen', 4),
    ('Fashion', NULL),
    ('Men Clothing', 6),
    ('Women Clothing', 6),
    ('Books', NULL),
    ('Sports', NULL);

INSERT INTO suppliers (supplier_name, country)
SELECT 'Supplier ' || g, (ARRAY['Thailand','China','Japan','USA','Germany','Vietnam'])[1 + (g % 6)]
FROM generate_series(1, 40) AS g;

-- ==========================================================
-- 3) products: 1,500 แถว (สุ่ม category / supplier / ราคา)
-- ==========================================================
INSERT INTO products (product_name, category_id, supplier_id, unit_price, stock_quantity, is_active)
SELECT
    'Product ' || g,
    1 + (g % 10),                                  -- category_id 1-10
    1 + (g % 40),                                   -- supplier_id 1-40
    ROUND((10 + random() * 990)::numeric, 2),        -- ราคา 10 - 1000
    (random() * 500)::int,                           -- stock 0-500
    (random() < 0.9)                                 -- 90% active
FROM generate_series(1, 1500) AS g;

-- ==========================================================
-- 4) customers: 2,000 แถว
-- ==========================================================
INSERT INTO customers (first_name, last_name, email, country, signup_date)
SELECT
    'FirstName' || g,
    'LastName' || g,
    'customer' || g || '@example.com',
    (ARRAY['Thailand','Singapore','Malaysia','Vietnam','Philippines','Indonesia'])[1 + (g % 6)],
    CURRENT_DATE - ((random() * 1000)::int)
FROM generate_series(1, 2000) AS g;

-- ==========================================================
-- 5) orders: 8,000 แถว (กระจายช่วงเวลา 2 ปี, สถานะหลายแบบ)
-- ==========================================================
INSERT INTO orders (customer_id, order_date, status, ship_country)
SELECT
    1 + (random() * 1999)::int,
    now() - (random() * 730) * INTERVAL '1 day',
    (ARRAY['pending','paid','shipped','delivered','cancelled'])[1 + (g % 5)],
    (ARRAY['Thailand','Singapore','Malaysia','Vietnam','Philippines','Indonesia'])[1 + (g % 6)]
FROM generate_series(1, 8000) AS g;

-- ==========================================================
-- 6) order_items: ~20,000 แถว (แต่ละ order มี 1-4 รายการ)
-- ==========================================================
INSERT INTO order_items (order_id, product_id, quantity, unit_price)
SELECT
    o.order_id,
    1 + (random() * 1499)::int,
    1 + (random() * 4)::int,
    ROUND((10 + random() * 990)::numeric, 2)
FROM orders o
CROSS JOIN LATERAL generate_series(1, 1 + (random() * 3)::int) AS item_no
;

-- ==========================================================
-- 7) สร้าง index พื้นฐาน + เก็บ statistics
-- ==========================================================
CREATE INDEX idx_products_category ON products(category_id);
CREATE INDEX idx_products_supplier ON products(supplier_id);
CREATE INDEX idx_orders_customer   ON orders(customer_id);
CREATE INDEX idx_orders_date       ON orders(order_date);
CREATE INDEX idx_order_items_order   ON order_items(order_id);
CREATE INDEX idx_order_items_product ON order_items(product_id);

ANALYZE categories;
ANALYZE suppliers;
ANALYZE products;
ANALYZE customers;
ANALYZE orders;
ANALYZE order_items;
```

ตรวจสอบจำนวนแถวที่ได้:

```sql
SELECT 'categories' AS tbl, count(*) FROM categories
UNION ALL SELECT 'suppliers', count(*) FROM suppliers
UNION ALL SELECT 'products', count(*) FROM products
UNION ALL SELECT 'customers', count(*) FROM customers
UNION ALL SELECT 'orders', count(*) FROM orders
UNION ALL SELECT 'order_items', count(*) FROM order_items;
```

```
    tbl      | count
-------------+-------
 categories  |    10
 suppliers   |    40
 products    |  1500
 customers   |  2000
 orders      |  8000
 order_items | 19987
(6 rows)
```

> **หมายเหตุ:** ค่า `order_items` อาจแตกต่างเล็กน้อยในแต่ละครั้งที่รัน เพราะใช้ `random()` ในการสุ่มจำนวนรายการต่อออเดอร์ — ไม่เป็นปัญหา เพราะ Part นี้เน้นการอ่านแผนการทำงาน ไม่ใช่ผลลัพธ์ตัวเลขที่ตายตัว ตัวเลข cost/rows/time ที่แสดงในตัวอย่างทั้งหมดของบทนี้เป็นค่าประมาณเพื่อประกอบการอธิบาย รันจริงในเครื่องของท่านอาจได้ตัวเลขต่างไปได้ แต่ **รูปแบบของ plan และแนวคิดการอ่านจะเหมือนกัน**

---

## Step 431: Query Planner คืออะไร

SQL เป็นภาษาแบบ **declarative** — เราบอก PostgreSQL ว่า "อยากได้อะไร" (เช่น "ขอรายชื่อลูกค้าที่มียอดสั่งซื้อรวมมากกว่า 10,000 บาท") แต่เราไม่ได้บอกว่า "ต้องทำอย่างไรทีละขั้นตอน" (เช่น จะอ่านตารางไหนก่อน จะ join ด้วยวิธีไหน จะ sort ตอนไหน) หน้าที่ในการแปลงคำขอแบบ declarative ให้กลายเป็นชุดคำสั่งเชิงกระบวนการ (procedural) ที่รันได้จริง เป็นของ **Query Planner** (บางครั้งเรียก Query Optimizer)

กระบวนการคร่าว ๆ เมื่อ PostgreSQL ได้รับ SQL statement มีดังนี้:

```
SQL Text
   │
   ▼
Parser          → ตรวจ syntax, สร้าง Parse Tree
   │
   ▼
Rewriter        → ขยาย view, apply rule (เช่น RLS)
   │
   ▼
Planner/Optimizer → พิจารณาแผนการทำงานที่เป็นไปได้หลายแบบ
                    ประเมิน "cost" ของแต่ละแบบ แล้วเลือกแผนที่ cost ต่ำสุด
   │
   ▼
Executor        → รันแผนที่เลือกจริง ดึงข้อมูล คืนผลลัพธ์
```

ประเด็นสำคัญคือ Planner **ไม่ได้ลองรันจริงทุกแผน** (นั่นจะช้าเกินไป) แต่ใช้ **cost-based optimization**: มันจะประเมิน "ต้นทุน" ของแต่ละแผนที่เป็นไปได้โดยอาศัย

1. **สถิติของข้อมูล** (statistics) เช่น จำนวนแถวทั้งหมดในตาราง, การกระจายของค่าต่าง ๆ ในแต่ละคอลัมน์, ค่าที่พบบ่อยที่สุด (most common values) — เก็บไว้ใน catalog ผ่านคำสั่ง `ANALYZE`
2. **โครงสร้างที่มีอยู่** เช่น index ที่สร้างไว้, การเรียงลำดับทางกายภาพของข้อมูลในตาราง (correlation)
3. **ค่าพารามิเตอร์ cost ต่าง ๆ** ที่บอกว่าการอ่าน disk 1 หน้า แพงแค่ไหนเทียบกับการประมวลผล 1 แถวใน CPU

ตัวอย่างพารามิเตอร์ cost ที่สำคัญ (ดูค่าได้ด้วย `SHOW`):

```sql
SHOW seq_page_cost;        -- ต้นทุนอ่าน 1 page แบบ sequential  (default 1.0)
SHOW random_page_cost;     -- ต้นทุนอ่าน 1 page แบบ random      (default 4.0, ลดเหลือ ~1.1 ถ้าใช้ SSD)
SHOW cpu_tuple_cost;       -- ต้นทุนประมวลผล 1 แถว               (default 0.01)
SHOW cpu_index_tuple_cost; -- ต้นทุนประมวลผล 1 index entry        (default 0.005)
SHOW cpu_operator_cost;    -- ต้นทุนประมวลผล 1 operator/function  (default 0.0025)
SHOW work_mem;             -- หน่วยความจำสำหรับ sort/hash ต่อ operation
```

```
 seq_page_cost
----------------
 1
(1 row)
```

แนวคิดสำคัญที่ต้องจำไว้ตลอดบทนี้: **Planner ไม่ได้ตัดสินใจแบบ "ถูกหรือผิด" แต่ตัดสินใจแบบ "ประมาณการต้นทุนต่ำสุด"** ถ้าค่าประมาณ (estimate) ผิดพลาด (เช่น statistics เก่า) แผนที่ Planner เลือกก็อาจไม่เหมาะสมกับข้อมูลจริง — นี่คือสาเหตุหลักของปัญหา query ช้าที่พบบ่อยที่สุดในโลกจริง และเป็นเหตุผลที่เราต้องเรียนรู้การอ่าน `EXPLAIN` ให้เป็น

**เปรียบเทียบง่าย ๆ:** เหมือนแอปนำทาง (Google Maps) — เราบอกแค่จุดเริ่มต้นกับปลายทาง (declarative) แอปจะคำนวณเส้นทางที่ "คาดว่า" เร็วที่สุดจากข้อมูลการจราจรที่มันมี (statistics) แต่ถ้าข้อมูลจราจรไม่อัปเดต (statistics ล้าสมัย) เส้นทางที่แนะนำก็อาจไม่ใช่เส้นทางที่เร็วที่สุดจริง ๆ

---

## Step 432: EXPLAIN พื้นฐาน — อ่าน cost, rows, width

คำสั่ง `EXPLAIN` จะแสดง **แผนการทำงาน (execution plan)** ที่ Planner เลือก โดย**ไม่รัน query จริง** จึงปลอดภัยที่จะใช้กับคำสั่งใด ๆ รวมถึง `UPDATE`/`DELETE` ที่มี side effect ด้วย เพราะ EXPLAIN เฉย ๆ ไม่ execute จริง

```sql
EXPLAIN
SELECT *
FROM products
WHERE category_id = 3;
```

ผลลัพธ์ตัวอย่าง:

```
                                QUERY PLAN
--------------------------------------------------------------------------
 Bitmap Heap Scan on products  (cost=4.42..38.15 rows=150 width=41)
   Recheck Cond: (category_id = 3)
   ->  Bitmap Index Scan on idx_products_category  (cost=0.00..4.38 rows=150 width=0)
         Index Cond: (category_id = 3)
(4 rows)
```

มาแยกอ่านทีละส่วน:

### 1. โครงสร้างแบบ tree

Plan จะอยู่ในรูปแบบ **tree** อ่านจาก**ในสุด/ล่างสุดก่อน แล้วไล่ขึ้นบนออกมา** — Node ที่อยู่ลึกที่สุด (มี indent มากสุด) จะทำงานก่อน แล้วส่งผลลัพธ์ขึ้นไปให้ node แม่ (parent) ประมวลผลต่อ ในตัวอย่างนี้ `Bitmap Index Scan` (ลูก) ทำงานก่อน แล้วส่งต่อให้ `Bitmap Heap Scan` (แม่)

### 2. cost=startup..total

```
cost=4.42..38.15
```

- **startup cost (4.42):** ต้นทุนโดยประมาณ "ก่อนที่จะเริ่มคืนแถวแรก" เช่น เวลาที่ใช้ sort ข้อมูลก่อนจะได้แถวแรกออกมา (สำหรับ Seq Scan startup cost มักเป็น 0 เพราะเริ่มคืนแถวได้ทันที)
- **total cost (38.15):** ต้นทุนโดยประมาณ "ถ้าต้องดึงข้อมูลทั้งหมดจน node นี้จบ"
- หน่วยของ cost **ไม่ใช่วินาที** แต่เป็น**หน่วยนามธรรม** ที่คำนวณจากพารามิเตอร์ cost ต่าง ๆ (ดู Step 431) ใช้เปรียบเทียบระหว่างแผนที่เป็นไปได้เท่านั้น ไม่ควรตีความเป็นเวลาจริงโดยตรง

### 3. rows

```
rows=150
```

คือ**จำนวนแถวที่ Planner คาดว่า node นี้จะคืนออกมา** (estimated rows) ไม่ใช่จำนวนจริง — มาจากสถิติที่เก็บไว้ตอน `ANALYZE` ค่านี้สำคัญมาก เพราะ Planner ใช้มันประเมิน cost ของ node ที่อยู่เหนือขึ้นไป (เช่น join) ถ้าค่านี้ผิดเพี้ยนไปมาก แผนทั้งหมดจะเพี้ยนตาม (ดู Step 437)

### 4. width

```
width=41
```

คือ**ความกว้างเฉลี่ยโดยประมาณของแต่ละแถว (เป็น byte)** ที่ node นี้จะคืน ใช้ประกอบการคำนวณ cost ของการ sort/hash/ส่งข้อมูล — ยิ่งแถวกว้าง (เช่น `SELECT *` ที่มีคอลัมน์ text ยาว ๆ) cost การจัดการก็ยิ่งสูงขึ้น

### ตัวอย่างเปรียบเทียบ: EXPLAIN แบบไม่มี filter (Seq Scan)

```sql
EXPLAIN
SELECT * FROM products;
```

```
                          QUERY PLAN
---------------------------------------------------------------
 Seq Scan on products  (cost=0.00..48.50 rows=1500 width=41)
(1 row)
```

สังเกตว่า `Seq Scan` มี startup cost = 0.00 (เริ่มอ่านแถวแรกได้ทันที เพราะอ่านตารางไล่ตั้งแต่ต้น) และ `rows=1500` คือประมาณจำนวนแถวทั้งหมดในตาราง `products`

### EXPLAIN (ANALYZE false, COSTS true, ...) — ตัวเลือกเสริม

```sql
EXPLAIN (COSTS true, VERBOSE true)
SELECT product_name, unit_price
FROM products
WHERE category_id = 3 AND is_active = true;
```

```
                                     QUERY PLAN
--------------------------------------------------------------------------------------
 Bitmap Heap Scan on public.products  (cost=4.79..37.06 rows=134 width=21)
   Output: product_name, unit_price
   Recheck Cond: (products.category_id = 3)
   Filter: products.is_active
   ->  Bitmap Index Scan on idx_products_category  (cost=0.00..4.75 rows=150 width=0)
         Index Cond: (products.category_id = 3)
(6 rows)
```

- `VERBOSE` เพิ่มบรรทัด `Output:` แสดงคอลัมน์ที่แต่ละ node จะส่งออกไป (มีประโยชน์เวลา debug ว่าคอลัมน์ไหนถูกส่งผ่านไปจริง ๆ)
- สังเกตความแตกต่างระหว่าง `Recheck Cond` (เงื่อนไขจาก index ที่ heap scan ต้องตรวจซ้ำ) กับ `Filter` (เงื่อนไขเพิ่มเติมที่ไม่ได้ใช้ index เลย ต้องมาเช็คทีละแถวหลังอ่าน heap แล้ว)

---

## Step 433: EXPLAIN ANALYZE — รันจริงและวัดเวลาจริง

`EXPLAIN` เฉย ๆ บอกแค่ "แผนที่จะใช้" และ "ค่าประมาณ" แต่ `EXPLAIN ANALYZE` จะ **รัน query จริง** แล้ววัดเวลาจริงของแต่ละ node พร้อมนับจำนวนแถวจริงที่เกิดขึ้นจริง (actual rows)

```sql
EXPLAIN ANALYZE
SELECT *
FROM products
WHERE category_id = 3;
```

```
                                                        QUERY PLAN
----------------------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on products  (cost=4.42..38.15 rows=150 width=41) (actual time=0.045..0.128 rows=142 loops=1)
   Recheck Cond: (category_id = 3)
   Heap Blocks: exact=38
   ->  Bitmap Index Scan on idx_products_category  (cost=0.00..4.38 rows=150 width=0) (actual time=0.026..0.026 rows=142 loops=1)
         Index Cond: (category_id = 3)
 Planning Time: 0.187 ms
 Execution Time: 0.163 ms
(7 rows)
```

### ส่วนที่เพิ่มเข้ามาจาก EXPLAIN ธรรมดา

```
(actual time=0.045..0.128 rows=142 loops=1)
```

- **actual time=startup..total (มิลลิวินาที):** เวลาจริงที่ node นี้ใช้ — startup คือเวลาก่อนคืนแถวแรก, total คือเวลารวมจนคืนแถวสุดท้าย **(หมายเหตุ: ค่านี้คือเวลาเฉลี่ยต่อ 1 loop ถ้า `loops > 1` ต้องคูณกลับเพื่อดูเวลารวมจริงของ node นั้น)**
- **rows=142:** จำนวนแถว**จริง**ที่ node นี้คืนออกมา (actual) — เทียบกับ `rows=150` ที่เป็นค่าประมาณด้านบน ต่างกันไม่มาก แปลว่าสถิติแม่นยำ
- **loops=1:** จำนวนครั้งที่ node นี้ถูกเรียกทำงาน — สำหรับ node ที่อยู่ใต้ Nested Loop join อาจถูกเรียกหลายครั้ง (loops > 1) เพราะถูกเรียกซ้ำสำหรับแต่ละแถวของฝั่งซ้าย
- **Planning Time:** เวลาที่ใช้ในการวางแผน (parse + plan) แยกจากเวลารัน
- **Execution Time:** เวลารวมที่ใช้ execute แผนทั้งหมดจริง (เวลานี้รวม overhead ของ EXPLAIN ANALYZE เองเล็กน้อยด้วย เพราะต้องมีการวัดเวลาของทุก node)

### ⚠️ ข้อควรระวังสำคัญ: EXPLAIN ANALYZE กับคำสั่งที่มี side effect

**`EXPLAIN ANALYZE` รัน query จริง** ซึ่งหมายความว่าถ้าใช้กับ `INSERT`, `UPDATE`, `DELETE` หรือ `CREATE TABLE AS` — **การเปลี่ยนแปลงข้อมูลจะเกิดขึ้นจริง ไม่ใช่แค่จำลอง!**

```sql
-- ❌ อันตราย: นี่คือ UPDATE จริง ข้อมูลจะถูกแก้ไขจริงในตาราง!
EXPLAIN ANALYZE
UPDATE products
SET stock_quantity = stock_quantity - 1
WHERE product_id = 100;
```

```
                                              QUERY PLAN
--------------------------------------------------------------------------------------------------
 Update on products  (cost=0.29..8.31 rows=0 width=0) (actual time=0.089..0.090 rows=0 loops=1)
   ->  Index Scan using products_pkey on products  (cost=0.29..8.31 rows=1 width=10) (actual time=0.021..0.023 rows=1 loops=1)
         Index Cond: (product_id = 100)
 Planning Time: 0.112 ms
 Execution Time: 0.145 ms
(5 rows)
```

รันคำสั่งด้านบนหนึ่งครั้ง `stock_quantity` ของ `product_id = 100` จะถูกลบไปจริง 1 หน่วย!

**วิธีป้องกันที่ปลอดภัย** — ครอบด้วย transaction แล้ว `ROLLBACK`:

```sql
BEGIN;

EXPLAIN ANALYZE
UPDATE products
SET stock_quantity = stock_quantity - 1
WHERE product_id = 100;

ROLLBACK;  -- ยกเลิกการเปลี่ยนแปลง แต่ยังได้เห็นแผน + เวลาจริง
```

วิธีนี้ปลอดภัยเพราะ EXPLAIN ANALYZE ต้อง execute คำสั่งจริงเพื่อวัดผล (รวมถึง trigger, constraint check ต่าง ๆ ที่เกี่ยวข้องด้วย) แต่เมื่อ `ROLLBACK` การเปลี่ยนแปลงข้อมูลทั้งหมดจะถูกยกเลิก

### EXPLAIN (ANALYZE, BUFFERS) — ดู I/O จริง

ตัวเลือกที่มีประโยชน์มากอีกตัวคือ `BUFFERS` ซึ่งจะแสดงว่า node นั้นอ่านข้อมูลจาก **shared buffer cache (hit)** หรือต้องไปอ่านจาก **disk/OS cache (read)**:

```sql
EXPLAIN (ANALYZE, BUFFERS, TIMING true)
SELECT *
FROM order_items
WHERE order_id = 500;
```

```
                                                     QUERY PLAN
---------------------------------------------------------------------------------------------------------------------
 Index Scan using idx_order_items_order on order_items  (cost=0.29..8.53 rows=3 width=20)
                                                          (actual time=0.019..0.023 rows=2 loops=1)
   Index Cond: (order_id = 500)
   Buffers: shared hit=4
 Planning Time: 0.078 ms
 Execution Time: 0.041 ms
(5 rows)
```

- `shared hit=4` แปลว่าอ่านได้ครบ 4 page จาก shared buffer cache ทั้งหมด — **ไม่มีการอ่าน disk จริง** (ถ้าข้อมูลไม่อยู่ใน cache จะเห็น `shared read=N` เพิ่มเข้ามา ซึ่งช้ากว่ามาก)
- ตัวเลือกนี้มีประโยชน์มากในการวินิจฉัยว่า query ช้าเพราะ **I/O จริง** (ต้องอ่าน disk) หรือช้าเพราะ **CPU/algorithm** (ข้อมูลอยู่ใน cache หมดแล้วแต่ยังช้า)

> **แนวทางปฏิบัติที่แนะนำ:** เวลา debug performance จริง ใช้ `EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)` เป็นค่าเริ่มต้น และครอบด้วย `BEGIN ... ROLLBACK` เสมอถ้า query เป็น `UPDATE`/`DELETE`/`INSERT`

---

## Step 434: Node type ที่พบบ่อย — วิธีเข้าถึงข้อมูล (Scan)

PostgreSQL มีวิธี "สแกน" ตารางหลายแบบ Planner จะเลือกแบบที่ cost ต่ำสุดตามเงื่อนไขและ selectivity ของ query

### 1. Seq Scan (Sequential Scan)

อ่านทุกแถวในตารางเรียงตามลำดับทางกายภาพ ไม่ใช้ index เลย เหมาะเมื่อ:
- ต้องอ่านข้อมูล**สัดส่วนใหญ่**ของตาราง (เช่น เกิน ~10-20% ของแถวทั้งหมด)
- ตารางมีขนาดเล็กมาก จน Seq Scan เร็วกว่าการเปิด index
- ไม่มี index ที่ใช้ได้กับเงื่อนไขนั้น

```sql
EXPLAIN ANALYZE
SELECT * FROM orders WHERE status = 'pending';
```

```
                                             QUERY PLAN
------------------------------------------------------------------------------------------------
 Seq Scan on orders  (cost=0.00..170.00 rows=1600 width=32) (actual time=0.012..1.234 rows=1598 loops=1)
   Filter: ((status)::text = 'pending'::text)
   Rows Removed by Filter: 6402
 Planning Time: 0.095 ms
 Execution Time: 1.312 ms
(5 rows)
```

`status` มีเพียง 5 ค่าเท่า ๆ กัน (20% ต่อค่า) — คิดเป็นสัดส่วนใหญ่ของตาราง Planner จึงเลือก Seq Scan แม้จะมี index อื่นอยู่ก็ตาม (เพราะไม่มี index บน `status`) สังเกต `Rows Removed by Filter: 6402` บอกว่าอ่านมา 8000 แถว แต่ทิ้งไป 6402 แถวเพราะไม่ตรงเงื่อนไข — เป็นสัญญาณว่า filter ไม่ selective พอที่จะคุ้มกับการสร้าง index เพิ่ม

### 2. Index Scan

ใช้ index เพื่อค้นหาตำแหน่งของแถวที่ตรงเงื่อนไข แล้ว**กลับไปอ่าน heap (ตารางจริง)** เพื่อดึงคอลัมน์ที่ index ไม่มี เหมาะเมื่อเงื่อนไข selective มาก (คืนแถวจำนวนน้อยเทียบกับทั้งตาราง)

```sql
EXPLAIN ANALYZE
SELECT * FROM customers WHERE customer_id = 555;
```

```
                                                    QUERY PLAN
-------------------------------------------------------------------------------------------------------------
 Index Scan using customers_pkey on customers  (cost=0.29..8.31 rows=1 width=48) (actual time=0.019..0.021 rows=1 loops=1)
   Index Cond: (customer_id = 555)
 Planning Time: 0.068 ms
 Execution Time: 0.038 ms
(4 rows)
```

`customers_pkey` คือ index บน primary key — ใช้ค้นหาแถวเดียวได้เร็วมาก (cost ต่ำมากเทียบกับ Seq Scan ทั้งตาราง 2000 แถว)

### 3. Index Only Scan

เหมือน Index Scan แต่**ไม่ต้องกลับไปอ่าน heap เลย** เพราะคอลัมน์ทั้งหมดที่ query ต้องการมีอยู่ครบใน index แล้ว (covering index) — เร็วกว่า Index Scan ปกติเพราะลด I/O ไปหนึ่งขั้น

```sql
EXPLAIN ANALYZE
SELECT customer_id FROM customers WHERE customer_id BETWEEN 100 AND 200;
```

```
                                                        QUERY PLAN
--------------------------------------------------------------------------------------------------------------------------
 Index Only Scan using customers_pkey on customers  (cost=0.29..8.52 rows=101 width=4)
                                                     (actual time=0.021..0.089 rows=101 loops=1)
   Index Cond: ((customer_id >= 100) AND (customer_id <= 200))
   Heap Fetches: 0
 Planning Time: 0.089 ms
 Execution Time: 0.121 ms
(5 rows)
```

- `Heap Fetches: 0` คือตัวชี้วัดสำคัญ — หมายความว่า**ไม่ต้องแตะ heap เลย** ข้อมูลทุกอย่างที่ต้องการ (แค่ `customer_id`) มีอยู่ครบใน index แล้ว
- ถ้า `Heap Fetches` มีค่ามาก (> 0 เยอะ ๆ) แปลว่า visibility map ยังไม่ up-to-date (ต้อง `VACUUM` เพื่อให้ PostgreSQL รู้ว่าแถวไหน "มองเห็นได้แน่นอน" โดยไม่ต้องเช็ค heap)

### 4. Bitmap Heap Scan / Bitmap Index Scan

ใช้เมื่อคาดว่าจะได้แถวจำนวน**ปานกลาง** (ไม่น้อยพอจะใช้ Index Scan ตรง ๆ แต่ก็ไม่มากพอจะ Seq Scan) กระบวนการคือ:

1. **Bitmap Index Scan**: สแกน index เพื่อสร้าง "bitmap" ของ**ตำแหน่ง page** (ไม่ใช่ตำแหน่งแถว) ที่มีข้อมูลตรงเงื่อนไข
2. **Bitmap Heap Scan**: ไล่อ่าน heap ตาม page ที่ bitmap ระบุ (เรียงตามลำดับ physical page แทนที่จะกระโดดไปมาแบบสุ่มเหมือน Index Scan) ทำให้ I/O มีประสิทธิภาพมากขึ้นเมื่อจำนวนแถวเยอะ

```sql
EXPLAIN ANALYZE
SELECT * FROM products WHERE category_id IN (2, 3);
```

```
                                                        QUERY PLAN
--------------------------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on products  (cost=6.85..64.20 rows=284 width=41) (actual time=0.078..0.312 rows=291 loops=1)
   Recheck Cond: (category_id = ANY ('{2,3}'::integer[]))
   Heap Blocks: exact=62
   ->  BitmapOr  (cost=6.85..6.85 rows=284 width=0) (actual time=0.052..0.053 rows=0 loops=1)
         ->  Bitmap Index Scan on idx_products_category  (cost=0.00..3.40 rows=142 width=0)
                                                          (actual time=0.028..0.028 rows=134 loops=1)
               Index Cond: (category_id = 2)
         ->  Bitmap Index Scan on idx_products_category  (cost=0.00..3.40 rows=142 width=0)
                                                          (actual time=0.023..0.023 rows=157 loops=1)
               Index Cond: (category_id = 3)
 Planning Time: 0.145 ms
 Execution Time: 0.398 ms
(9 rows)
```

สังเกต `BitmapOr` — เมื่อมีหลายเงื่อนไขที่ใช้ index คนละครั้ง (เช่น `IN (2, 3)`) PostgreSQL จะสแกน index แยกแต่ละค่าแล้วนำ bitmap มา **OR** รวมกัน ก่อนไปอ่าน heap เพียงครั้งเดียว (ต่างจากการทำ Index Scan สองรอบแยกกันที่จะอ่าน heap page ซ้ำซ้อน)

### สรุปเปรียบเทียบ Scan node

| Node | ใช้เมื่อ | I/O pattern | ต้องอ่าน heap ไหม |
|---|---|---|---|
| Seq Scan | เงื่อนไขไม่ selective / ไม่มี index | Sequential ทั้งตาราง | ต้อง |
| Index Scan | เงื่อนไข selective มาก (แถวน้อย) | Random (ตามตำแหน่ง index) | ต้อง |
| Index Only Scan | เงื่อนไข selective + ทุกคอลัมน์อยู่ใน index | Random แต่เบากว่า | ไม่ต้อง (ถ้า visibility map พร้อม) |
| Bitmap Heap/Index Scan | เงื่อนไขคืนแถวปานกลาง | สร้าง bitmap แล้วอ่าน heap แบบเรียง page | ต้อง (แต่มีประสิทธิภาพกว่า Index Scan ตรง ๆ) |

---

## Step 435: Join node types — Nested Loop, Hash Join, Merge Join

เมื่อ query ต้อง join สองตารางขึ้นไป Planner ต้องเลือกทั้ง**ลำดับการ join** (join order) และ**อัลกอริทึมการ join** (join algorithm) — ในบทนี้เน้นที่อัลกอริทึม 3 แบบหลัก

### 1. Nested Loop Join

สำหรับแต่ละแถวใน**ตารางนอก (outer/driving table)** จะวนไปค้นหาแถวที่ match ใน**ตารางใน (inner table)** — เหมาะเมื่อตารางนอกมีแถวน้อย และตารางในมี index ที่ใช้ค้นหาได้เร็ว (เพราะจะถูกเรียกซ้ำหลายครั้ง)

```sql
EXPLAIN ANALYZE
SELECT c.first_name, c.last_name, o.order_id, o.order_date
FROM customers c
JOIN orders o ON o.customer_id = c.customer_id
WHERE c.customer_id = 42;
```

```
                                                      QUERY PLAN
-----------------------------------------------------------------------------------------------------------------------
 Nested Loop  (cost=0.58..16.90 rows=4 width=44) (actual time=0.028..0.052 rows=4 loops=1)
   ->  Index Scan using customers_pkey on customers c  (cost=0.29..8.31 rows=1 width=17) (actual time=0.015..0.016 rows=1 loops=1)
         Index Cond: (customer_id = 42)
   ->  Index Scan using idx_orders_customer on orders o  (cost=0.29..8.55 rows=4 width=16) (actual time=0.010..0.030 rows=4 loops=1)
         Index Cond: (customer_id = 42)
 Planning Time: 0.203 ms
 Execution Time: 0.086 ms
(7 rows)
```

Flow: อ่าน `customers` มา 1 แถว (ตรงกับ `customer_id = 42`) แล้วสำหรับแถวนั้น (loop=1 ครั้ง) ไปค้นหาใน `orders` ผ่าน index `idx_orders_customer` — เพราะฝั่งนอกมีแค่ 1 แถว จึงเหมาะกับ Nested Loop มาก (ต้นทุนต่ำที่สุด)

### 2. Hash Join

สร้าง **hash table** จากตารางที่เล็กกว่า (ปกติ build จากฝั่งที่ประมาณว่าแถวน้อยกว่า) ในหน่วยความจำ (`work_mem`) แล้วสแกนอีกตารางหนึ่งครั้งเดียว ไล่ hash-lookup เทียบกับ hash table เหมาะเมื่อ**ทั้งสองฝั่งมีแถวจำนวนมาก** และไม่มีเงื่อนไข equality ที่ selective พอจะใช้ index ได้ดี

```sql
EXPLAIN ANALYZE
SELECT p.product_name, count(*) AS times_ordered
FROM order_items oi
JOIN products p ON p.product_id = oi.product_id
GROUP BY p.product_name;
```

```
                                                          QUERY PLAN
--------------------------------------------------------------------------------------------------------------------------------
 HashAggregate  (cost=712.34..727.34 rows=1500 width=23) (actual time=48.123..48.987 rows=1487 loops=1)
   Group Key: p.product_name
   Batches: 1  Memory Usage: 273kB
   ->  Hash Join  (cost=42.50..612.80 rows=19987 width=15) (actual time=0.487..38.221 rows=19987 loops=1)
         Hash Cond: (oi.product_id = p.product_id)
         ->  Seq Scan on order_items oi  (cost=0.00..339.87 rows=19987 width=4) (actual time=0.008..8.102 rows=19987 loops=1)
         ->  Hash  (cost=23.00..23.00 rows=1500 width=15) (actual time=0.462..0.463 rows=1500 loops=1)
               Buckets: 2048  Batches: 1  Memory Usage: 94kB
               ->  Seq Scan on products p  (cost=0.00..23.00 rows=1500 width=15) (actual time=0.005..0.198 rows=1500 loops=1)
 Planning Time: 0.312 ms
 Execution Time: 49.201 ms
(11 rows)
```

Flow: PostgreSQL สร้าง Hash table จาก `products` (ตารางเล็กกว่า 1500 แถว, `Hash` node) แล้วสแกน `order_items` (19987 แถว) เพียงรอบเดียว ไล่ lookup แต่ละแถวใน hash table — มีประสิทธิภาพกว่า Nested Loop มากเมื่อทั้งสองฝั่งมีขนาดใหญ่ เพราะไม่ต้อง loop ซ้ำ ๆ

สังเกต `Buckets: 2048  Batches: 1` — ถ้า hash table ใหญ่เกินกว่า `work_mem` จะเห็น `Batches` มากกว่า 1 (ต้องแบ่งเป็นหลาย batch เขียนลง disk ชั่วคราว) ซึ่งจะทำให้ช้าลงมาก

### 3. Merge Join

ทั้งสองฝั่งต้อง**เรียงลำดับ (sorted)** ตามคีย์ join ก่อน (ไม่ว่าจะเรียงมาแล้วจาก index หรือต้อง sort เพิ่ม) แล้ว "merge" สองสตรีมที่เรียงแล้วเข้าด้วยกันแบบเดินหน้าเพียงรอบเดียว (คล้ายการ merge สองกองไพ่ที่เรียงแล้ว) เหมาะเมื่อข้อมูลทั้งสองฝั่งเรียงลำดับอยู่แล้ว (เช่นมาจาก Index Scan) หรือเมื่อผลลัพธ์ต้องการ ORDER BY ตามคีย์เดียวกันอยู่แล้ว

```sql
EXPLAIN ANALYZE
SELECT o.order_id, o.order_date, oi.product_id, oi.quantity
FROM orders o
JOIN order_items oi ON oi.order_id = o.order_id
ORDER BY o.order_id
LIMIT 2000;
```

```
                                                             QUERY PLAN
-------------------------------------------------------------------------------------------------------------------------------------
 Limit  (cost=0.85..245.30 rows=2000 width=24) (actual time=0.041..12.887 rows=2000 loops=1)
   ->  Merge Join  (cost=0.85..2445.12 rows=19987 width=24) (actual time=0.040..12.612 rows=2000 loops=1)
         Merge Cond: (o.order_id = oi.order_id)
         ->  Index Scan using orders_pkey on orders o  (cost=0.29..410.29 rows=8000 width=16) (actual time=0.012..2.345 rows=1180 loops=1)
         ->  Index Scan using idx_order_items_order on order_items oi  (cost=0.29..1600.12 rows=19987 width=12) (actual time=0.010..5.221 rows=2000 loops=1)
 Planning Time: 0.198 ms
 Execution Time: 13.045 ms
(7 rows)
```

Flow: ทั้ง `orders` (ผ่าน `orders_pkey`) และ `order_items` (ผ่าน `idx_order_items_order`) ถูกอ่านแบบเรียงตาม `order_id` อยู่แล้วจาก index ทำให้ Merge Join สามารถเดินหน้าเทียบคีย์ทีละคู่ได้โดยไม่ต้อง sort เพิ่ม — เหมาะมากกับ query ที่มี `ORDER BY` ตามคีย์ join พร้อม `LIMIT`

### สรุปเปรียบเทียบ Join algorithm

| Algorithm | เหมาะเมื่อ | ต้องการ | ข้อเสีย |
|---|---|---|---|
| **Nested Loop** | ฝั่งนอกมีแถวน้อย + ฝั่งในมี index ดี | Index บน join key ฝั่งใน | ช้ามากถ้าฝั่งนอกมีแถวเยอะ (loop ซ้ำเยอะ) |
| **Hash Join** | ทั้งสองฝั่งแถวเยอะ, equality join | `work_mem` พอสร้าง hash table | ต้องใช้หน่วยความจำ, ใช้ได้กับ `=` เท่านั้น |
| **Merge Join** | ข้อมูลเรียงอยู่แล้ว (จาก index) หรือ join แล้วต้อง sort ต่อ | ข้อมูลถูก sort มาก่อน (หรือ sort เพิ่ม) | ถ้าต้อง sort ทั้งสองฝั่งเพิ่มเอง อาจแพงกว่า Hash Join |

---

## Step 436: Sort และ Aggregate node — external sort เมื่อ work_mem ไม่พอ

### Sort node

เมื่อ query มี `ORDER BY` ที่ไม่สามารถใช้ index ตอบได้โดยตรง (หรือใช้ในบริบทอื่นเช่นก่อน Merge Join) PostgreSQL จะแทรก `Sort` node

```sql
EXPLAIN ANALYZE
SELECT product_name, unit_price
FROM products
ORDER BY unit_price DESC
LIMIT 20;
```

```
                                                          QUERY PLAN
--------------------------------------------------------------------------------------------------------------------------------
 Limit  (cost=52.14..52.19 rows=20 width=21) (actual time=1.234..1.241 rows=20 loops=1)
   ->  Sort  (cost=52.14..55.89 rows=1500 width=21) (actual time=1.232..1.235 rows=20 loops=1)
         Sort Key: unit_price DESC
         Sort Method: top-N heapsort  Memory: 27kB
         ->  Seq Scan on products  (cost=0.00..23.00 rows=1500 width=21) (actual time=0.006..0.412 rows=1500 loops=1)
 Planning Time: 0.089 ms
 Execution Time: 1.278 ms
(7 rows)
```

ประเด็นสำคัญคือบรรทัด **`Sort Method`** และ **`Memory`**:

- `Sort Method: top-N heapsort  Memory: 27kB` — เมื่อมี `LIMIT` ประกอบกับ `ORDER BY` PostgreSQL ฉลาดพอที่จะใช้ **heap-based top-N algorithm** ซึ่งใช้หน่วยความจำน้อยมาก (เก็บแค่ N แถวที่ดีที่สุด ณ ขณะนั้น) ไม่ต้อง sort ทั้งตาราง

เปรียบเทียบกับกรณีที่ต้อง sort ทั้งหมดโดยไม่มี LIMIT:

```sql
EXPLAIN ANALYZE
SELECT product_name, unit_price
FROM products
ORDER BY unit_price DESC;
```

```
                                                    QUERY PLAN
--------------------------------------------------------------------------------------------------------------------
 Sort  (cost=131.66..135.41 rows=1500 width=21) (actual time=2.145..2.298 rows=1500 loops=1)
   Sort Key: unit_price DESC
   Sort Method: quicksort  Memory: 158kB
   ->  Seq Scan on products  (cost=0.00..23.00 rows=1500 width=21) (actual time=0.006..0.398 rows=1500 loops=1)
 Planning Time: 0.078 ms
 Execution Time: 2.401 ms
(6 rows)
```

`Sort Method: quicksort  Memory: 158kB` — sort ข้อมูลทั้งหมดในหน่วยความจำ (in-memory) เพราะ 158kB ยังน้อยกว่า `work_mem` (default 4MB) มาก

### ⚠️ External sort — เมื่อ work_mem ไม่พอ

ถ้าข้อมูลที่ต้อง sort **ใหญ่กว่า `work_mem`** PostgreSQL จะต้องใช้ **disk-based merge sort** (เขียนไฟล์ชั่วคราวลง disk แล้ว merge กลับ) ซึ่งช้ากว่า in-memory sort มาก มาจำลองสถานการณ์นี้โดยลด `work_mem` ให้ต่ำมาก ๆ ชั่วคราว:

```sql
SET work_mem = '64kB';  -- ลดชั่วคราวเพื่อสาธิต (ปกติค่า default คือ 4MB)

EXPLAIN ANALYZE
SELECT *
FROM order_items
ORDER BY unit_price DESC;
```

```
                                                       QUERY PLAN
---------------------------------------------------------------------------------------------------------------------
 Sort  (cost=2847.51..2897.48 rows=19987 width=20) (actual time=45.201..52.887 rows=19987 loops=1)
   Sort Key: unit_price DESC
   Sort Method: external merge  Disk: 528kB
   ->  Seq Scan on order_items  (cost=0.00..339.87 rows=19987 width=20) (actual time=0.008..3.102 rows=19987 loops=1)
 Planning Time: 0.067 ms
 Execution Time: 55.412 ms
(6 rows)
```

```sql
RESET work_mem;  -- คืนค่าเดิมทันทีหลังทดสอบ
```

สังเกตความแตกต่าง:

```
Sort Method: external merge  Disk: 528kB      ← ใช้ disk! ช้ากว่ามาก
Sort Method: quicksort       Memory: 158kB    ← ใช้ memory ล้วน เร็วกว่า
```

**`Sort Method: external merge  Disk: ...`** คือสัญญาณเตือนสำคัญ — แปลว่า PostgreSQL ต้องเขียนข้อมูลชั่วคราวลง disk (temp file) ระหว่าง sort ซึ่งช้ากว่าการ sort ในหน่วยความจำล้วนหลายเท่า ถ้าเจอในระบบจริง วิธีแก้คือ:

1. เพิ่มค่า `work_mem` (ระดับ session หรือ global) ให้เพียงพอ
2. ลดจำนวนแถว/คอลัมน์ที่ต้อง sort (เช่นกรองก่อน sort, เลือกเฉพาะคอลัมน์ที่ใช้จริง)
3. พิจารณาสร้าง index ที่ช่วยให้ query อ่านข้อมูลแบบเรียงอยู่แล้ว (หลีกเลี่ยงการ sort ทั้งหมด)

### Aggregate node

มี 2 แบบหลักที่พบบ่อย:

**HashAggregate** — ใช้ hash table เก็บผลรวมของแต่ละกลุ่ม (ไม่ต้องเรียงข้อมูลก่อน) เหมาะเมื่อจำนวนกลุ่ม (distinct groups) ไม่มากเกินไปจนเกิน `work_mem`

```sql
EXPLAIN ANALYZE
SELECT category_id, count(*), avg(unit_price)
FROM products
GROUP BY category_id;
```

```
                                                    QUERY PLAN
---------------------------------------------------------------------------------------------------------------
 HashAggregate  (cost=27.75..27.85 rows=10 width=44) (actual time=0.512..0.516 rows=10 loops=1)
   Group Key: category_id
   Batches: 1  Memory Usage: 24kB
   ->  Seq Scan on products  (cost=0.00..23.00 rows=1500 width=12) (actual time=0.006..0.198 rows=1500 loops=1)
 Planning Time: 0.089 ms
 Execution Time: 0.551 ms
(6 rows)
```

**GroupAggregate** — ต้องการข้อมูลที่**เรียงลำดับตาม GROUP BY key อยู่แล้ว** (จาก Sort node หรือ Index Scan) แล้วไล่รวมกลุ่มไปทีละกลุ่มตามลำดับ เหมาะเมื่อมีจำนวนกลุ่มมากเกินกว่าจะเก็บใน hash table ได้ หรือเมื่อข้อมูลเรียงมาแล้วโดยไม่ต้อง sort เพิ่ม

```sql
EXPLAIN ANALYZE
SELECT customer_id, count(*) AS order_count
FROM orders
GROUP BY customer_id
ORDER BY customer_id;
```

```
                                                       QUERY PLAN
---------------------------------------------------------------------------------------------------------------------
 GroupAggregate  (cost=0.29..800.29 rows=2000 width=12) (actual time=0.025..8.912 rows=1847 loops=1)
   Group Key: customer_id
   ->  Index Scan using idx_orders_customer on orders  (cost=0.29..700.29 rows=8000 width=4) (actual time=0.018..5.201 rows=8000 loops=1)
 Planning Time: 0.098 ms
 Execution Time: 9.045 ms
(5 rows)
```

เพราะข้อมูลถูกอ่านผ่าน `idx_orders_customer` ซึ่งเรียงตาม `customer_id` อยู่แล้ว จึงใช้ GroupAggregate ได้โดยไม่ต้อง sort เพิ่มเติม (ประหยัด cost เมื่อเทียบกับ HashAggregate ที่ต้องสร้าง hash table สำหรับ 2000 กลุ่ม)

---

## Step 437: planned rows vs actual rows — เมื่อ estimate ผิดพลาด

นี่คือทักษะที่**สำคัญที่สุด**ในการอ่าน `EXPLAIN ANALYZE` — การเปรียบเทียบ **ค่าประมาณ (planned/estimated rows)** กับ **ค่าจริง (actual rows)** ในแต่ละ node

```
Bitmap Heap Scan on products  (cost=4.42..38.15 rows=150 width=41) (actual time=0.045..0.128 rows=142 loops=1)
                                                        ▲                                            ▲
                                                  planned rows                                  actual rows
```

ถ้าตัวเลขทั้งสองใกล้เคียงกัน (เช่น 150 vs 142) แปลว่า statistics แม่นยำ Planner มีข้อมูลที่ดีพอในการตัดสินใจ แต่ถ้าตัวเลข**ต่างกันมาก** (เช่น เป็นสิบ/ร้อย/พันเท่า) นั่นคือสัญญาณว่า **statistics ล้าสมัยหรือไม่เพียงพอ** ซึ่งอาจทำให้ Planner เลือกแผนที่ผิดพลาด

### ตัวอย่าง: จำลองสถิติล้าสมัย

ลองเพิ่มข้อมูลจำนวนมากเข้าตารางโดยไม่ `ANALYZE` ใหม่:

```sql
-- เพิ่มออเดอร์สถานะ 'cancelled' เข้าไปอีก 5000 แถว แบบ bulk (ไม่ ANALYZE ทันที)
INSERT INTO orders (customer_id, order_date, status, ship_country)
SELECT
    1 + (random() * 1999)::int,
    now() - (random() * 30) * INTERVAL '1 day',
    'cancelled',
    'Thailand'
FROM generate_series(1, 5000);

-- ยังไม่ ANALYZE — สถิติยังเป็นค่าเก่าก่อนเพิ่มข้อมูล
EXPLAIN ANALYZE
SELECT * FROM orders WHERE status = 'cancelled';
```

```
                                             QUERY PLAN
------------------------------------------------------------------------------------------------
 Seq Scan on orders  (cost=0.00..212.00 rows=1600 width=32) (actual time=0.010..2.145 rows=6600 loops=1)
   Filter: ((status)::text = 'cancelled'::text)
   Rows Removed by Filter: 6400
 Planning Time: 0.078 ms
 Execution Time: 2.301 ms
(5 rows)
```

สังเกต: **`rows=1600` (ประมาณการ) แต่ `actual ... rows=6600` (จริง)** — ต่างกันกว่า 4 เท่า! เพราะสถิติยังนับสัดส่วนของ `status = 'cancelled'` จากก่อนที่จะ insert 5000 แถวใหม่เข้าไป (ครั้งก่อน 20% ของ 8000 = 1600 แถว แต่ตอนนี้มี 6600 แถวจริงจาก 13000 แถวทั้งหมด)

ในตัวอย่างนี้ Planner ยังเลือก Seq Scan ถูกอยู่ดี (เพราะสัดส่วนยังสูงทั้งคู่) แต่ถ้าเป็น query ที่ join หลายตาราง **ความคลาดเคลื่อนแบบนี้จะถูกขยายทวีคูณ (compounding error)** เมื่อผ่าน join หลายชั้น จนอาจทำให้ Planner เลือก Nested Loop ในจุดที่ควรใช้ Hash Join (หรือกลับกัน) — เกิดปัญหา query ที่ "ปกติเร็ว แต่จู่ ๆ ช้าลงมาก" หลังจากมีการเพิ่มข้อมูลจำนวนมาก

แก้ปัญหาด้วยการ `ANALYZE` ตารางที่เพิ่งเปลี่ยนแปลงข้อมูลมาก:

```sql
ANALYZE orders;

EXPLAIN ANALYZE
SELECT * FROM orders WHERE status = 'cancelled';
```

```
                                             QUERY PLAN
------------------------------------------------------------------------------------------------
 Seq Scan on orders  (cost=0.00..274.75 rows=6624 width=32) (actual time=0.011..2.398 rows=6600 loops=1)
   Filter: ((status)::text = 'cancelled'::text)
   Rows Removed by Filter: 6400
 Planning Time: 0.089 ms
 Execution Time: 2.501 ms
(5 rows)
```

หลัง `ANALYZE` ค่า `rows=6624` ใกล้เคียงกับ `actual rows=6600` มาก — สถิติแม่นยำขึ้นทันที

### เมื่อไหร่ที่ estimate ผิดพลาดสูง มักเกิดจาก

1. **ไม่ได้ `ANALYZE` หลัง bulk insert/update/delete จำนวนมาก** (autovacuum ปกติจะ analyze ให้อัตโนมัติ แต่มี threshold และ delay)
2. **คอลัมน์ที่มีความสัมพันธ์กัน (correlated columns)** — เช่น `city` กับ `zip_code` ที่ Planner มองแยกกันทีละคอลัมน์ (คำนวณ selectivity แบบเป็นอิสระต่อกัน) ทั้งที่จริงมีความสัมพันธ์กันสูง — แก้ได้ด้วย extended statistics (`CREATE STATISTICS`)
3. **Expression หรือ function ที่ซับซ้อนใน WHERE** (เช่น `WHERE lower(email) = ...`) ที่ Planner ประเมิน selectivity แบบเดา (default guess) เพราะไม่มีสถิติเฉพาะของผลลัพธ์ function นั้น
4. **การกระจายข้อมูลไม่สม่ำเสมอ (skewed distribution)** ที่ histogram แบบ default (100 buckets) ยังไม่ละเอียดพอ

---

## Step 438: ANALYZE command, pg_stats, default_statistics_target

### คำสั่ง ANALYZE

`ANALYZE` คือคำสั่งที่สแกนตาราง (แบบสุ่มตัวอย่าง ไม่ใช่อ่านทั้งหมด) เพื่ออัปเดตสถิติที่ Planner ใช้ในการตัดสินใจ เก็บไว้ใน system catalog `pg_statistic` (และแสดงผลอ่านง่ายผ่าน view `pg_stats`)

```sql
-- ANALYZE ตารางเดียว
ANALYZE products;

-- ANALYZE เฉพาะบางคอลัมน์
ANALYZE products (unit_price, stock_quantity);

-- ANALYZE ทั้งฐานข้อมูล
ANALYZE;

-- VACUUM + ANALYZE พร้อมกัน (แนะนำหลัง bulk load ข้อมูลจำนวนมาก)
VACUUM ANALYZE products;
```

```
ANALYZE
```

โดยปกติ **autovacuum** จะรัน `ANALYZE` ให้อัตโนมัติเมื่อจำนวนแถวที่เปลี่ยนแปลง (insert/update/delete) เกิน threshold ที่กำหนด (ค่า default ประมาณ 10% ของตาราง + offset คงที่) — แต่หลัง bulk load ข้อมูลจำนวนมากในคราวเดียว **แนะนำให้ `ANALYZE` ด้วยตนเองทันที** แทนที่จะรอ autovacuum เพราะช่วงเวลาที่สถิติยัง "เก่า" อาจทำให้ query สำคัญ ๆ ทำงานช้าโดยไม่จำเป็น

### pg_stats — ดูสถิติที่เก็บไว้

```sql
SELECT
    attname AS column_name,
    n_distinct,
    most_common_vals,
    most_common_freqs,
    correlation
FROM pg_stats
WHERE tablename = 'orders' AND attname = 'status';
```

```
 column_name | n_distinct |              most_common_vals               |               most_common_freqs                | correlation
-------------+------------+----------------------------------------------+--------------------------------------------------+-------------
 status      |          5 | {cancelled,delivered,paid,pending,shipped}   | {0.508,0.123,0.123,0.123,0.123}                   |        0.18
(1 row)
```

คอลัมน์สำคัญใน `pg_stats`:

| คอลัมน์ | ความหมาย |
|---|---|
| `n_distinct` | จำนวนค่าที่ไม่ซ้ำกันโดยประมาณ (ค่าบวก = จำนวนจริง, ค่าลบ = สัดส่วนต่อขนาดตาราง เช่น -0.5 แปลว่าประมาณครึ่งหนึ่งของแถวมีค่าไม่ซ้ำกัน) |
| `most_common_vals` | รายการค่าที่พบบ่อยที่สุด (MCV) — ใช้คำนวณ selectivity แม่นยำสำหรับค่าเหล่านี้โดยตรง |
| `most_common_freqs` | ความถี่ (สัดส่วน) ของแต่ละค่าใน `most_common_vals` |
| `histogram_bounds` | ขอบเขต histogram สำหรับค่าที่ไม่ติด MCV ใช้ประมาณ selectivity ของช่วง (range) |
| `correlation` | ความสัมพันธ์ระหว่างลำดับค่าในคอลัมน์กับลำดับทางกายภาพในตาราง (ใกล้ 1 หรือ -1 = เรียงตามลำดับ physical, ใกล้ 0 = กระจายสุ่ม) ใช้ประเมิน cost ของ Index Scan |

### default_statistics_target

ควบคุม**ความละเอียด**ของสถิติ (จำนวน MCV + จำนวน histogram bucket) ที่ `ANALYZE` จะเก็บต่อคอลัมน์

```sql
SHOW default_statistics_target;
```

```
 default_statistics_target
----------------------------
 100
(1 row)
```

ค่า default คือ **100** (หมายถึงเก็บ MCV สูงสุด 100 ค่า และ histogram 100 buckets) — ยิ่งค่าสูง สถิติยิ่งละเอียด แต่ `ANALYZE` จะใช้เวลานานขึ้นและกิน catalog space มากขึ้น เหมาะปรับเพิ่มเฉพาะคอลัมน์ที่มีการกระจายข้อมูลซับซ้อน (skewed) และถูกใช้ใน WHERE บ่อย:

```sql
-- เพิ่มความละเอียดสถิติเฉพาะคอลัมน์ unit_price (การกระจายราคาซับซ้อน)
ALTER TABLE products ALTER COLUMN unit_price SET STATISTICS 500;

ANALYZE products;  -- ต้อง ANALYZE ใหม่เพื่อให้ค่ามีผล
```

```
ALTER TABLE
ANALYZE
```

ปรับ level ทั้งฐานข้อมูล (ต้อง restart หรือ reload):

```sql
ALTER SYSTEM SET default_statistics_target = 200;
SELECT pg_reload_conf();
```

> **แนวทางปฏิบัติ:** ไม่ควรปรับ `default_statistics_target` ให้สูงทั่วทั้งระบบโดยไม่จำเป็น เพราะทำให้ `ANALYZE` ช้าลงทุกตาราง ควรปรับเฉพาะคอลัมน์ที่มีปัญหาจริง (ผ่าน `ALTER TABLE ... SET STATISTICS`) ดีกว่า

---

## Step 439: EXPLAIN (FORMAT JSON/YAML) และเครื่องมือ visualize

นอกจาก text format ที่ใช้มาตลอดบทนี้ `EXPLAIN` ยังรองรับ format อื่น ๆ ที่เหมาะกับการประมวลผลด้วยโปรแกรม หรือส่งต่อให้เครื่องมือ visualize

### EXPLAIN (FORMAT JSON)

```sql
EXPLAIN (ANALYZE, FORMAT JSON, BUFFERS)
SELECT c.country, count(*) AS order_count, sum(oi.quantity * oi.unit_price) AS total_sales
FROM customers c
JOIN orders o ON o.customer_id = c.customer_id
JOIN order_items oi ON oi.order_id = o.order_id
WHERE o.status = 'delivered'
GROUP BY c.country
ORDER BY total_sales DESC;
```

```json
[
  {
    "Plan": {
      "Node Type": "Sort",
      "Startup Cost": 1245.67,
      "Total Cost": 1248.12,
      "Plan Rows": 6,
      "Plan Width": 40,
      "Actual Startup Time": 42.315,
      "Actual Total Time": 42.331,
      "Actual Rows": 6,
      "Actual Loops": 1,
      "Sort Key": ["(sum((oi.quantity * oi.unit_price))) DESC"],
      "Sort Method": "quicksort",
      "Sort Space Used": 25,
      "Sort Space Type": "Memory",
      "Plans": [
        {
          "Node Type": "Aggregate",
          "Strategy": "Hashed",
          "Parent Relationship": "Outer",
          "Startup Cost": 1245.29,
          "Total Cost": 1245.50,
          "Plan Rows": 6,
          "Plan Width": 40,
          "Actual Startup Time": 42.267,
          "Actual Total Time": 42.281,
          "Actual Rows": 6,
          "Actual Loops": 1,
          "Group Key": ["c.country"],
          "Plans": [
            {
              "Node Type": "Hash Join",
              "Parent Relationship": "Outer",
              "Startup Cost": 92.50,
              "Total Cost": 998.34,
              "Plan Rows": 3980,
              "Plan Width": 16,
              "Actual Startup Time": 3.201,
              "Actual Total Time": 32.487,
              "Actual Rows": 3987,
              "Actual Loops": 1,
              "Hash Cond": "(oi.order_id = o.order_id)"
            }
          ]
        }
      ]
    }
  }
]
```

ประโยชน์ของ `FORMAT JSON`:

- **แปลงเป็น object/array ได้ตรง ๆ** ในภาษาโปรแกรม (Python, Node.js, Go ฯลฯ) เหมาะกับการเขียนสคริปต์ตรวจสอบ plan อัตโนมัติ (เช่น CI pipeline ที่เช็คว่า query สำคัญยังใช้ index scan อยู่ ไม่ตกไปเป็น seq scan)
- เก็บค่าตัวเลขเป็น**ตัวเลขจริง** (ไม่ใช่ string ที่ต้อง parse) ทำให้เปรียบเทียบ/คำนวณ cost, time ได้ง่ายกว่า
- เป็น format ที่เครื่องมือ visualize ภายนอกส่วนใหญ่รองรับ (นำเข้าได้โดยตรง)

### EXPLAIN (FORMAT YAML)

```sql
EXPLAIN (FORMAT YAML)
SELECT * FROM products WHERE category_id = 3;
```

```yaml
- Plan:
    Node Type: "Bitmap Heap Scan"
    Relation Name: "products"
    Alias: "products"
    Startup Cost: 4.42
    Total Cost: 38.15
    Plan Rows: 150
    Plan Width: 41
    Recheck Cond: "(category_id = 3)"
    Plans:
      - Node Type: "Bitmap Index Scan"
        Parent Relationship: "Outer"
        Index Name: "idx_products_category"
        Startup Cost: 0.00
        Total Cost: 4.38
        Plan Rows: 150
        Plan Width: 0
        Index Cond: "(category_id = 3)"
```

YAML อ่านง่ายกว่า JSON เล็กน้อยสำหรับมนุษย์ (ไม่มี comma/bracket รกตา) แต่ในทางปฏิบัติ JSON ถูกใช้แพร่หลายกว่าเพราะรองรับโดย library แทบทุกภาษา

### เครื่องมือ Visualize: explain.dalibo.com (PEV2)

เมื่อ plan ซับซ้อนมาก (มีหลายสิบ node ซ้อนกันหลายชั้น) การอ่าน text/JSON ตรง ๆ อาจเข้าใจยาก จึงมีเครื่องมือ **visualize** ที่ช่วยแสดงผลเป็นกราฟิก เช่น:

- **explain.dalibo.com (PEV2 - PostgreSQL Explain Visualizer 2)** — เว็บฟรีที่รับ `EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON)` แล้ววาดเป็น tree diagram พร้อม**ไฮไลต์สี**ตาม cost/เวลาที่ใช้ (node ที่กิน resource เยอะสุดจะเด่นชัดด้วยสีและขนาด) ทำให้มองเห็นได้ทันทีว่า "คอขวด" ของ query อยู่ที่ node ไหน โดยไม่ต้องไล่อ่านตัวเลขทีละบรรทัด
- **pgAdmin's Graphical Explain** — เครื่องมือในตัว pgAdmin ที่วาด plan เป็น flowchart พร้อม tooltip แสดงรายละเอียดเมื่อ hover
- **auto_explain** (extension) — บันทึก plan ของ query ที่ช้าเกิน threshold ที่กำหนดลง log อัตโนมัติ โดยไม่ต้องรัน EXPLAIN เอง เหมาะกับการ debug query ที่ช้าเป็นครั้งคราวใน production

**แนวคิดของ visualizer เหล่านี้** (ไม่ต้อง fetch จริงเพื่อเข้าใจ concept):

1. รับ plan ในรูป `FORMAT JSON` (เพราะโครงสร้างข้อมูลชัดเจนที่สุดสำหรับ parse)
2. วาดแต่ละ node เป็นกล่อง เชื่อมด้วยเส้นตาม parent-child relationship (เหมือน tree ที่เราวาดด้วยมือในบทนี้)
3. ใช้ **สี** หรือ**ความหนาของเส้น** แทนสัดส่วนของ**เวลาที่ node นั้นใช้เทียบกับเวลารวมทั้งหมด** (exclusive time) — ช่วยให้เห็น "จุดที่กิน 80% ของเวลา" ได้ในพริบตา แทนที่จะต้องคำนวณเองจาก `actual time` ของแต่ละ node
4. แสดง**คำเตือนอัตโนมัติ** เช่น "estimate ผิดพลาดสูง" (เทียบ planned vs actual rows), "external sort ใช้ disk", "loops สูงผิดปกติ" ให้เห็นชัดโดยไม่ต้องไล่หาเอง

ตัวอย่างวิธีใช้งานจริง (ขั้นตอน ไม่ใช่การ fetch):

```sql
-- 1) รัน EXPLAIN แบบ ANALYZE + BUFFERS + FORMAT JSON
EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON)
SELECT ...;

-- 2) copy ผลลัพธ์ JSON ทั้งหมด
-- 3) วางลงในหน้าเว็บ explain.dalibo.com/plans/new
-- 4) ดูภาพ tree diagram ที่ไฮไลต์ node ที่ช้าที่สุดโดยอัตโนมัติ
```

> **ข้อควรระวัง:** เวลาส่ง plan ไปยังเครื่องมือ third-party ควรตรวจสอบว่า plan นั้นไม่มีข้อมูลที่อ่อนไหว (เช่น literal value ใน `Filter`/`Index Cond` ที่อาจเป็นข้อมูลส่วนบุคคล) รั่วไหลออกไปโดยไม่ตั้งใจ

---

## สรุปท้ายบท: Cheat Sheet การอ่าน EXPLAIN

### ตารางสรุป field สำคัญใน EXPLAIN output

| Field | ความหมาย | ดูใน EXPLAIN เฉย ๆ | ดูใน EXPLAIN ANALYZE |
|---|---|:---:|:---:|
| `cost=startup..total` | ต้นทุนโดยประมาณ (หน่วยนามธรรม เปรียบเทียบได้ ไม่ใช่วินาที) | ✅ | ✅ |
| `rows` | จำนวนแถวที่**ประมาณ**ว่าจะได้จาก node นี้ | ✅ | ✅ |
| `width` | ความกว้างเฉลี่ยของแถว (byte) | ✅ | ✅ |
| `actual time=startup..total` | เวลาจริง (ms) ที่ node ใช้ (เฉลี่ยต่อ 1 loop) | ❌ | ✅ |
| `actual rows` | จำนวนแถว**จริง**ที่ได้จาก node นี้ | ❌ | ✅ |
| `loops` | จำนวนครั้งที่ node ถูกเรียก | ❌ | ✅ |
| `Planning Time` | เวลาที่ใช้วางแผน query | ❌ | ✅ |
| `Execution Time` | เวลารวมที่ใช้ execute จริง | ❌ | ✅ |
| `Buffers: shared hit/read` | จำนวน page ที่อ่านจาก cache (hit) หรือ disk (read) | ❌ | ✅ (ต้องเปิด `BUFFERS`) |
| `Rows Removed by Filter` | จำนวนแถวที่อ่านมาแล้วแต่ถูกทิ้งเพราะไม่ผ่านเงื่อนไข | ❌ | ✅ |
| `Sort Method` | `quicksort`/`top-N heapsort` (memory) หรือ `external merge` (disk) | ❌ | ✅ |
| `Heap Fetches` | จำนวนครั้งที่ Index Only Scan ต้องกลับไปอ่าน heap (ควรเป็น 0) | ❌ | ✅ |

### ตารางสรุป Node type

| Node | ใช้เมื่อ | สัญญาณเตือนที่ควรระวัง |
|---|---|---|
| **Seq Scan** | อ่านสัดส่วนใหญ่ของตาราง หรือไม่มี index | ถ้า `rows=` น้อยแต่ยังเลือก Seq Scan → เช็คว่ามี index ไหม |
| **Index Scan** | เงื่อนไข selective มาก | ถ้า loops สูงมาก (อยู่ใต้ Nested Loop) → อาจช้าสะสม |
| **Index Only Scan** | ทุกคอลัมน์ที่ใช้อยู่ใน index | `Heap Fetches` สูง → ต้อง `VACUUM` |
| **Bitmap Heap/Index Scan** | คืนแถวปานกลาง | `Heap Blocks: lossy` → `work_mem` ไม่พอสร้าง bitmap แม่นยำ |
| **Nested Loop** | ฝั่งนอกแถวน้อย + ฝั่งในมี index | ถ้าฝั่งนอกแถวเยอะโดยไม่คาดคิด → ช้ามาก (estimate ผิด) |
| **Hash Join** | ทั้งสองฝั่งแถวเยอะ, equality join | `Batches > 1` → hash table ล้น `work_mem` |
| **Merge Join** | ข้อมูลเรียงอยู่แล้ว หรือ join แล้วต้อง sort ต่อ | มี extra `Sort` node ก่อน merge → เสีย cost เพิ่ม |
| **Sort** | `ORDER BY`, ใช้ก่อน Merge Join | `Sort Method: external merge Disk:` → `work_mem` ไม่พอ |
| **HashAggregate** | `GROUP BY` ที่จำนวนกลุ่มไม่เยอะเกินไป | `Batches > 1` → กลุ่มเยอะเกิน `work_mem` |
| **GroupAggregate** | `GROUP BY` บนข้อมูลที่เรียงอยู่แล้ว | ถ้ามี `Sort` node ตามมาด้วย → เสีย cost ที่ควรเลี่ยงได้ |

### กระบวนการวินิจฉัยที่แนะนำ (Diagnostic Checklist)

```
1. รัน EXPLAIN (ANALYZE, BUFFERS) ครอบด้วย BEGIN...ROLLBACK ถ้าเป็น DML
2. ไล่หา node ที่มี "actual total time" มากที่สุด (มักเป็นตัวการหลักที่ query ช้า)
3. เทียบ planned rows กับ actual rows ในทุก node — ถ้าต่างกันมาก → ANALYZE ตาราง
4. เช็คว่ามี "Seq Scan" บนตารางใหญ่ที่ไม่ควรมีไหม → พิจารณาสร้าง index
5. เช็คว่ามี "Sort Method: external merge Disk:" ไหม → เพิ่ม work_mem
6. เช็คว่ามี "Heap Fetches" สูงใน Index Only Scan ไหม → รัน VACUUM
7. เช็คว่า Hash Join/Aggregate มี "Batches > 1" ไหม → เพิ่ม work_mem
8. เช็ค Buffers: shared read สูงผิดปกติไหม → ปัญหา I/O / cache ไม่พอ
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1

รัน `EXPLAIN` (ไม่ต้อง ANALYZE) กับ query ต่อไปนี้ แล้วอธิบายว่า Planner เลือกใช้ Node type อะไร เพราะเหตุใด:

```sql
EXPLAIN
SELECT * FROM customers WHERE email = 'customer555@example.com';
```

<details>
<summary>เฉลย</summary>

เนื่องจาก `customers.email` มี `UNIQUE` constraint ซึ่ง PostgreSQL สร้าง unique index ให้อัตโนมัติ และเงื่อนไข `=` บนคอลัมน์ unique จะคืนแถวเดียวเสมอ (selectivity สูงสุด) Planner จึงเลือก **Index Scan** ผ่าน unique index ที่สร้างจาก constraint (ชื่อ index มักเป็น `customers_email_key`):

```
                                                     QUERY PLAN
---------------------------------------------------------------------------------------------------------------
 Index Scan using customers_email_key on customers  (cost=0.29..8.31 rows=1 width=48)
   Index Cond: ((email)::text = 'customer555@example.com'::text)
(2 rows)
```

`rows=1` เพราะ Planner รู้จาก unique constraint ว่าค่า email ไม่ซ้ำกัน จึงมั่นใจว่าจะได้แถวเดียวเสมอ ไม่จำเป็นต้องพึ่ง statistics ในการประมาณค่านี้เลย

</details>

---

### แบบฝึกหัดที่ 2

รัน `EXPLAIN ANALYZE` กับ query ที่กรองบนคอลัมน์ `is_active` (boolean ที่ไม่มี index) แล้วสังเกตค่า `Rows Removed by Filter`:

```sql
EXPLAIN ANALYZE
SELECT * FROM products WHERE is_active = false;
```

อธิบายว่าทำไม Planner ไม่เลือกสร้างและใช้ index ในกรณีนี้ (สมมติว่าไม่มี index บน `is_active`)

<details>
<summary>เฉลย</summary>

ผลลัพธ์ตัวอย่าง:

```
                                          QUERY PLAN
-----------------------------------------------------------------------------------------------
 Seq Scan on products  (cost=0.00..27.75 rows=150 width=41) (actual time=0.008..0.312 rows=148 loops=1)
   Filter: (NOT is_active)
   Rows Removed by Filter: 1352
 Planning Time: 0.067 ms
 Execution Time: 0.356 ms
(5 rows)
```

เหตุผลที่ Planner ใช้ Seq Scan (ไม่ใช่เพราะไม่มี index — คำถามถามว่าถ้ามี index จะช่วยไหม): แม้จะสร้าง index บน `is_active` ก็ตาม โดยทั่วไป boolean column มีแค่ 2 ค่าที่เป็นไปได้ (true/false) ทำให้แต่ละค่ามีสัดส่วนสูงมาก (~10% สำหรับ false ในตัวอย่างนี้ เพราะ generate ข้อมูลด้วย `random() < 0.9` ให้ active) การใช้ Index Scan สำหรับสัดส่วนที่สูงขนาดนี้มักจะ**แพงกว่า** Seq Scan เพราะต้องกระโดดอ่าน heap แบบ random หลายร้อยครั้ง เทียบกับ Seq Scan ที่อ่านแบบ sequential รวดเดียว — โดยทั่วไป index จะคุ้มค่าเมื่อ selectivity ต่ำกว่าประมาณ 5-10% ของตาราง ไม่ใช่ 10% ขึ้นไป

</details>

---

### แบบฝึกหัดที่ 3

Query ต่อไปนี้ join `orders` กับ `customers` โดยกรองด้วยประเทศ ให้รัน `EXPLAIN ANALYZE` แล้วระบุว่า Join algorithm ที่ใช้คืออะไร และอธิบายเหตุผล:

```sql
EXPLAIN ANALYZE
SELECT o.order_id, o.order_date, c.first_name, c.last_name
FROM orders o
JOIN customers c ON c.customer_id = o.customer_id
WHERE o.ship_country = 'Thailand';
```

<details>
<summary>เฉลย</summary>

ผลลัพธ์ตัวอย่าง:

```
                                                         QUERY PLAN
----------------------------------------------------------------------------------------------------------------------------
 Hash Join  (cost=52.00..478.34 rows=2167 width=28) (actual time=0.612..8.201 rows=2178 loops=1)
   Hash Cond: (o.customer_id = c.customer_id)
   ->  Seq Scan on orders o  (cost=0.00..410.00 rows=2167 width=12) (actual time=0.010..3.102 rows=2178 loops=1)
         Filter: ((ship_country)::text = 'Thailand'::text)
         Rows Removed by Filter: 10822
   ->  Hash  (cost=27.00..27.00 rows=2000 width=24) (actual time=0.582..0.583 rows=2000 loops=1)
         Buckets: 2048  Batches: 1  Memory Usage: 141kB
         ->  Seq Scan on customers c  (cost=0.00..27.00 rows=2000 width=24) (actual time=0.006..0.221 rows=2000 loops=1)
 Planning Time: 0.156 ms
 Execution Time: 8.512 ms
(9 rows)
```

Planner เลือก **Hash Join** เพราะ:
1. `orders` ที่กรองด้วย `ship_country = 'Thailand'` ยังคืนแถวจำนวนค่อนข้างมาก (~2167 แถว จาก ~13000 แถว, ไม่ selective พอจะใช้ Nested Loop + Index Scan อย่างมีประสิทธิภาพ)
2. `customers` ทั้งตารางมี 2000 แถว ไม่มีเงื่อนไข filter — ขนาดพอเหมาะที่จะสร้าง hash table เก็บในหน่วยความจำได้ (`Buckets: 2048 Batches: 1` แสดงว่าพอดีไม่ต้อง spill ลง disk)
3. เป็น equality join (`=`) ซึ่ง Hash Join รองรับได้ดี และเมื่อทั้งสองฝั่งมีขนาดปานกลาง-ใหญ่ Hash Join จะเร็วกว่า Nested Loop มาก เพราะสแกนแต่ละตารางเพียงครั้งเดียว

</details>

---

### แบบฝึกหัดที่ 4

ให้ปรับค่า `work_mem` เป็น `'128kB'` ชั่วคราว แล้วรัน query ที่ sort ตาราง `order_items` ทั้งหมดตาม `unit_price` สังเกตว่า `Sort Method` เปลี่ยนไปเป็นอะไร แล้วอธิบายวิธีแก้ปัญหานี้ในระบบจริง

<details>
<summary>เฉลย</summary>

```sql
SET work_mem = '128kB';

EXPLAIN ANALYZE
SELECT * FROM order_items ORDER BY unit_price;

RESET work_mem;
```

ผลลัพธ์ตัวอย่าง (จะขึ้นกับขนาดข้อมูลจริง แต่แนวโน้มเดียวกัน):

```
                                                       QUERY PLAN
---------------------------------------------------------------------------------------------------------------------
 Sort  (cost=2847.51..2897.48 rows=19987 width=20) (actual time=48.201..58.887 rows=19987 loops=1)
   Sort Key: unit_price
   Sort Method: external merge  Disk: 552kB
   ->  Seq Scan on order_items  (cost=0.00..339.87 rows=19987 width=20) (actual time=0.008..3.102 rows=19987 loops=1)
 Planning Time: 0.067 ms
 Execution Time: 61.412 ms
(6 rows)
```

`Sort Method: external merge  Disk: 552kB` แปลว่าข้อมูลที่ต้อง sort (19987 แถว) ใหญ่เกินกว่า `work_mem` ที่ตั้งไว้ (128kB) ทำให้ต้องเขียนไฟล์ชั่วคราวลง disk แล้ว merge กลับ — ช้ากว่า in-memory sort มาก

**วิธีแก้ในระบบจริง:**
1. เพิ่ม `work_mem` ให้เหมาะสมกับขนาดข้อมูลที่ต้อง sort โดยทั่วไป (แต่ต้องระวัง เพราะ `work_mem` ถูกจองแยกต่อ operation ต่อ connection พร้อมกันหลาย session อาจใช้ RAM รวมสูงมาก)
2. ปรับ `work_mem` เฉพาะ session/query ที่ต้องการ sort ข้อมูลใหญ่ (`SET work_mem = '64MB'` ก่อนรัน query นั้น) แทนที่จะปรับ global
3. ลดจำนวนแถวที่ต้อง sort ด้วยการกรองก่อน (`WHERE`) หรือใช้ `LIMIT` ร่วมกับ `ORDER BY` เพื่อให้ PostgreSQL ใช้ top-N heapsort แทน

</details>

---

### แบบฝึกหัดที่ 5

รัน query ต่อไปนี้แล้วอธิบายความหมายของ `Heap Fetches` ที่เห็นในผลลัพธ์:

```sql
EXPLAIN ANALYZE
SELECT product_id FROM products WHERE product_id BETWEEN 1 AND 500;
```

ถ้า `Heap Fetches` มีค่ามากกว่า 0 มาก ควรทำอย่างไร?

<details>
<summary>เฉลย</summary>

ผลลัพธ์ตัวอย่าง (กรณี visibility map พร้อม):

```
                                                       QUERY PLAN
--------------------------------------------------------------------------------------------------------------------
 Index Only Scan using products_pkey on products  (cost=0.29..14.02 rows=500 width=4)
                                                   (actual time=0.021..0.198 rows=500 loops=1)
   Index Cond: ((product_id >= 1) AND (product_id <= 500))
   Heap Fetches: 0
 Planning Time: 0.078 ms
 Execution Time: 0.245 ms
(5 rows)
```

`Heap Fetches: 0` หมายความว่า query ตอบได้จาก index อย่างเดียวทั้งหมด โดยไม่ต้องกลับไปอ่าน heap เลย — เกิดขึ้นได้เพราะ PostgreSQL ใช้ **visibility map** เพื่อจำว่า page ไหน "มีแต่แถวที่มองเห็นได้แน่นอนสำหรับทุก transaction" (all-visible) ถ้า page นั้น all-visible ก็ไม่จำเป็นต้องเช็ค MVCC visibility จาก heap อีก

ถ้า `Heap Fetches` มีค่ามาก (ใกล้เคียงหรือเท่ากับ `actual rows`) แปลว่า visibility map ของตารางยัง**ไม่ update** (หลาย page ยังไม่ถูกทำเครื่องหมาย all-visible) ซึ่งมักเกิดจากตารางมีการ update/delete บ่อยแต่ยังไม่ได้ `VACUUM` — วิธีแก้คือรัน:

```sql
VACUUM products;
```

`VACUUM` จะอัปเดต visibility map ให้ page ที่ไม่มี dead tuple ถูกทำเครื่องหมาย all-visible ทำให้ Index Only Scan ครั้งถัดไปมี `Heap Fetches` ลดลง (หรือเป็น 0)

</details>

---

### แบบฝึกหัดที่ 6

Query ต่อไปนี้ join ตาราง 3 ชั้น (`customers` → `orders` → `order_items`) โดยกรอง `customer_id` เดี่ยว ๆ ให้รัน `EXPLAIN ANALYZE` แล้วอธิบายว่าทำไม node ระดับบนสุดถึงเป็น `Nested Loop` ไม่ใช่ `Hash Join`:

```sql
EXPLAIN ANALYZE
SELECT c.first_name, o.order_id, oi.product_id, oi.quantity
FROM customers c
JOIN orders o ON o.customer_id = c.customer_id
JOIN order_items oi ON oi.order_id = o.order_id
WHERE c.customer_id = 77;
```

<details>
<summary>เฉลย</summary>

ผลลัพธ์ตัวอย่าง:

```
                                                        QUERY PLAN
----------------------------------------------------------------------------------------------------------------------------
 Nested Loop  (cost=0.87..25.80 rows=8 width=24) (actual time=0.035..0.098 rows=6 loops=1)
   ->  Nested Loop  (cost=0.58..16.98 rows=4 width=20) (actual time=0.024..0.052 rows=3 loops=1)
         ->  Index Scan using customers_pkey on customers c  (cost=0.29..8.31 rows=1 width=17)
                                                              (actual time=0.011..0.012 rows=1 loops=1)
               Index Cond: (customer_id = 77)
         ->  Index Scan using idx_orders_customer on orders o  (cost=0.29..8.63 rows=4 width=8)
                                                                (actual time=0.009..0.032 rows=3 loops=1)
               Index Cond: (customer_id = 77)
   ->  Index Scan using idx_order_items_order on order_items oi  (cost=0.29..2.20 rows=2 width=12)
                                                                  (actual time=0.010..0.014 rows=2 loops=3)
         Index Cond: (order_id = o.order_id)
 Planning Time: 0.245 ms
 Execution Time: 0.134 ms
(9 rows)
```

เหตุผลที่ทุกชั้นเป็น **Nested Loop**: เงื่อนไข `WHERE c.customer_id = 77` ทำให้ตาราง `customers` คืนแค่ **1 แถว** เมื่อ join ต่อกับ `orders` ผ่าน `idx_orders_customer` ก็ได้แถวจำนวนน้อยมาก (~3-4 แถว) และเมื่อ join ต่อกับ `order_items` ก็ยังคงเป็นจำนวนแถวน้อย (สังเกต `loops=3` ใน Index Scan สุดท้าย — เพราะถูกเรียกซ้ำ 3 ครั้งตามจำนวนแถวของ orders ที่ match)

เมื่อจำนวนแถวในทุกขั้นน้อยมาก (single-digit) ต้นทุนของ Nested Loop (ที่ loop ซ้ำผ่าน index) จะต่ำกว่าการสร้าง Hash table (ที่มี overhead คงที่ในการสร้าง hash table แม้จะมีข้อมูลแค่ไม่กี่แถว) มาก — Nested Loop เหมาะที่สุดเมื่อ**ทุกฝั่งของ join มีแถวน้อยและมี index รองรับ** ซึ่งเป็นกรณีทั่วไปของการ query แบบ "ดูรายละเอียดของลูกค้า/ออเดอร์รายตัว" (OLTP pattern)

</details>

---

### แบบฝึกหัดที่ 7

พิจารณา query ต่อไปนี้ที่หา top 5 สินค้าขายดีที่สุด รัน `EXPLAIN ANALYZE` แล้วระบุ node ที่ใช้เวลามากที่สุด (bottleneck) พร้อมเสนอแนวทางปรับปรุง:

```sql
EXPLAIN ANALYZE
SELECT p.product_name, sum(oi.quantity) AS total_sold
FROM order_items oi
JOIN products p ON p.product_id = oi.product_id
JOIN orders o ON o.order_id = oi.order_id
WHERE o.status = 'delivered'
GROUP BY p.product_name
ORDER BY total_sold DESC
LIMIT 5;
```

<details>
<summary>เฉลย</summary>

ผลลัพธ์ตัวอย่าง:

```
                                                              QUERY PLAN
----------------------------------------------------------------------------------------------------------------------------------
 Limit  (cost=987.45..987.46 rows=5 width=27) (actual time=58.201..58.204 rows=5 loops=1)
   ->  Sort  (cost=987.20..991.20 rows=1487 width=27) (actual time=58.198..58.201 rows=5 loops=1)
         Sort Key: (sum(oi.quantity)) DESC
         Sort Method: top-N heapsort  Memory: 26kB
         ->  HashAggregate  (cost=920.34..945.34 rows=1487 width=27) (actual time=55.201..57.887 rows=1478 loops=1)
               Group Key: p.product_name
               Batches: 1  Memory Usage: 273kB
               ->  Hash Join  (cost=52.00..812.45 rows=6400 width=19) (actual time=2.201..48.334 rows=6398 loops=1)
                     Hash Cond: (oi.order_id = o.order_id)
                     ->  Hash Join  (cost=42.50..712.80 rows=19987 width=19) (actual time=0.487..38.221 rows=19987 loops=1)
                           Hash Cond: (oi.product_id = p.product_id)
                           ->  Seq Scan on order_items oi  (cost=0.00..339.87 rows=19987 width=8)
                                                            (actual time=0.008..8.102 rows=19987 loops=1)
                           ->  Hash  (cost=23.00..23.00 rows=1500 width=19) (actual time=0.462..0.463 rows=1500 loops=1)
                                 ->  Seq Scan on products p  (cost=0.00..23.00 rows=1500 width=19)
                                                              (actual time=0.005..0.198 rows=1500 loops=1)
                     ->  Hash  (cost=210.00..210.00 rows=2600 width=4) (actual time=1.598..1.599 rows=2600 loops=1)
                           ->  Seq Scan on orders o  (cost=0.00..210.00 rows=2600 width=4) (actual time=0.008..1.201 rows=1602 loops=1)
                                 Filter: ((status)::text = 'delivered'::text)
                                 Rows Removed by Filter: 11398
 Planning Time: 0.412 ms
 Execution Time: 58.501 ms
(19 rows)
```

**Bottleneck:** node ที่กิน "exclusive time" มากที่สุดคือ `HashAggregate` (actual time 55.201..57.887 = ใช้เวลาเองประมาณ 57.887 - 48.334 ≈ 9.5ms) และ `Hash Join` ชั้นบนสุด (actual time 2.201..48.334 = ประมวลผลเองประมาณ 48.334 - 38.221 ≈ 10ms หลังหักเวลาของ node ลูก) รวมกันแล้ว **Hash Join + HashAggregate คือส่วนที่ใช้เวลารวมมากที่สุด** เพราะต้อง join ข้อมูล ~20,000 แถวของ `order_items` เข้ากับทั้ง `products` และ `orders` ก่อนจะ group

**แนวทางปรับปรุง:**
1. สร้าง **composite index** บน `orders(status, order_id)` เพื่อให้กรอง `status = 'delivered'` ผ่าน index ได้เร็วขึ้น แทนที่จะ Seq Scan ทั้งตาราง (สังเกต `Rows Removed by Filter: 11398` — ทิ้งแถวไปเยอะมาก)
2. ถ้า query แบบนี้ถูกเรียกบ่อย พิจารณาสร้าง **materialized view** สรุปยอดขายต่อสินค้าล่วงหน้า แล้ว refresh เป็นระยะ แทนที่จะคำนวณสด (aggregate) ทุกครั้ง
3. พิจารณาว่ามีการกรอง `order_date` ร่วมด้วยหรือไม่ (เช่น "ยอดขายเดือนนี้") ถ้ามี ควรใช้ index ที่ครอบคลุมทั้ง `order_date` และ `status` เพื่อลดจำนวนแถวตั้งแต่ต้น

(รายละเอียดเทคนิคการ optimize เชิงลึกจะอยู่ใน Part 045 — Query Optimization)

</details>

---

### แบบฝึกหัดที่ 8

รัน `EXPLAIN (FORMAT JSON)` กับ query ง่าย ๆ หนึ่งคำสั่ง แล้วอธิบายว่าทำไม field ตัวเลขใน JSON output (เช่น `"Total Cost"`) จึงมีประโยชน์มากกว่า text format เมื่อต้องเขียนสคริปต์ตรวจสอบ plan อัตโนมัติ

<details>
<summary>เฉลย</summary>

```sql
EXPLAIN (FORMAT JSON)
SELECT * FROM products WHERE category_id = 3;
```

```json
[
  {
    "Plan": {
      "Node Type": "Bitmap Heap Scan",
      "Relation Name": "products",
      "Alias": "products",
      "Startup Cost": 4.42,
      "Total Cost": 38.15,
      "Plan Rows": 150,
      "Plan Width": 41,
      "Recheck Cond": "(category_id = 3)",
      "Plans": [
        {
          "Node Type": "Bitmap Index Scan",
          "Index Name": "idx_products_category",
          "Startup Cost": 0.00,
          "Total Cost": 4.38,
          "Plan Rows": 150,
          "Plan Width": 0,
          "Index Cond": "(category_id = 3)"
        }
      ]
    }
  }
]
```

ใน text format ค่าต่าง ๆ ถูกฝังรวมกันในสตริงเดียว เช่น `(cost=4.42..38.15 rows=150 width=41)` ต้องใช้ regular expression มา parse แยกตัวเลขออกมา ซึ่งเสี่ยง error ถ้ารูปแบบเปลี่ยนแปลงเล็กน้อย (เช่น version ใหม่เพิ่ม field)

ใน JSON format แต่ละค่าถูกแยกเป็น **key-value ที่มี type ชัดเจน** (`"Total Cost": 38.15` เป็นตัวเลข ไม่ใช่ string) ทำให้:
1. โปรแกรมสามารถอ่านค่าได้ตรง ๆ ผ่าน JSON parser มาตรฐานของทุกภาษา โดยไม่ต้องเขียน regex เอง
2. เปรียบเทียบ/คำนวณตัวเลขได้ทันที เช่น เขียนสคริปต์เช็คว่า `Total Cost` เกิน threshold ที่กำหนดหรือไม่ (ใช้ตรวจจับ query ที่แผนแย่ลงก่อน deploy — "plan regression testing")
3. โครงสร้าง `Plans` เป็น array ซ้อนกัน (nested) ทำให้ traverse tree ด้วยโค้ด recursive ได้ตรงไปตรงมา เหมาะกับการสร้างเครื่องมือ visualize หรือระบบ monitoring ที่ดึง metric จาก plan โดยอัตโนมัติ

</details>

---

### แบบฝึกหัดที่ 9

พิจารณา query ต่อไปนี้ที่หายอดขายรวมของลูกค้าแต่ละประเทศในช่วง 30 วันล่าสุด ให้รัน `EXPLAIN ANALYZE` แล้วตรวจสอบว่า **planned rows** กับ **actual rows** ของ node ที่กรอง `order_date` ต่างกันมากหรือไม่ พร้อมอธิบายว่าเหตุใดจึงเป็นเช่นนั้น:

```sql
EXPLAIN ANALYZE
SELECT c.country, sum(oi.quantity * oi.unit_price) AS revenue
FROM orders o
JOIN customers c ON c.customer_id = o.customer_id
JOIN order_items oi ON oi.order_id = o.order_id
WHERE o.order_date >= now() - INTERVAL '30 days'
GROUP BY c.country
ORDER BY revenue DESC;
```

<details>
<summary>เฉลย</summary>

ผลลัพธ์ตัวอย่าง (สมมติข้อมูลกระจายสม่ำเสมอในรอบ 2 ปี ดังนั้น 30 วัน ≈ 4% ของข้อมูล):

```
                                                              QUERY PLAN
--------------------------------------------------------------------------------------------------------------------------------------
 Sort  (cost=1123.45..1123.46 rows=6 width=40) (actual time=15.201..15.203 rows=6 loops=1)
   Sort Key: (sum((oi.quantity * oi.unit_price))) DESC
   Sort Method: quicksort  Memory: 25kB
   ->  HashAggregate  (cost=1123.10..1123.28 rows=6 width=40) (actual time=15.187..15.192 rows=6 loops=1)
         Group Key: c.country
         ->  Hash Join  (cost=142.50..1080.45 rows=853 width=19) (actual time=3.201..14.201 rows=812 loops=1)
               Hash Cond: (oi.order_id = o.order_id)
               ->  Seq Scan on order_items oi  (cost=0.00..339.87 rows=19987 width=12) (actual time=0.008..4.102 rows=19987 loops=1)
               ->  Hash  (cost=132.60..132.60 rows=356 width=15) (actual time=2.201..2.202 rows=331 loops=1)
                     ->  Hash Join  (cost=27.00..132.60 rows=356 width=15) (actual time=0.512..2.087 rows=331 loops=1)
                           Hash Cond: (o.customer_id = c.customer_id)
                           ->  Index Scan using idx_orders_date on orders o
                                     (cost=0.29..102.35 rows=356 width=8) (actual time=0.021..1.201 rows=331 loops=1)
                                 Index Cond: (order_date >= (now() - '30 days'::interval))
                           ->  Hash  (cost=23.00..23.00 rows=2000 width=15) (actual time=0.478..0.479 rows=2000 loops=1)
                                 ->  Seq Scan on customers c (cost=0.00..23.00 rows=2000 width=15) (actual time=0.005..0.201 rows=2000 loops=1)
 Planning Time: 0.389 ms
 Execution Time: 15.301 ms
(15 rows)
```

ที่ node `Index Scan using idx_orders_date`: **`rows=356` (planned) vs `actual ... rows=331` (จริง)** — ใกล้เคียงกันมาก (ต่างกันไม่ถึง 10%) แปลว่าสถิติของคอลัมน์ `order_date` **ยังแม่นยำดี** เพราะ:

1. `order_date` เป็นคอลัมน์ที่มีการกระจายแบบต่อเนื่อง (continuous distribution) ซึ่ง histogram ของ PostgreSQL (100 buckets โดย default) จับความหนาแน่นของช่วงเวลาได้ดีพอสมควร ตราบใดที่ข้อมูลกระจายสม่ำเสมอ
2. เงื่อนไข `order_date >= now() - INTERVAL '30 days'` เป็น expression ที่คำนวณค่าคงที่ตอน plan time (`now()` ถูก evaluate ครั้งเดียวตอนเริ่ม planning) ทำให้ Planner เทียบกับ histogram ได้ตรงไปตรงมาเหมือนเทียบกับค่าคงที่ทั่วไป
3. ตราบใดที่ `ANALYZE` ตาราง `orders` เป็นประจำ (ไม่ล้าสมัยเกินไป) การกระจายของ timestamp แบบนี้มักจะประมาณได้แม่นยำ

ถ้าค่าต่างกันมาก (เช่น planned 356 แต่ actual 3500) จะเป็นสัญญาณว่าข้อมูลเพิ่งถูก insert เข้ามาใหม่จำนวนมากในช่วง 30 วันล่าสุด (เช่น campaign พิเศษ) โดยที่ยังไม่ได้ `ANALYZE` ตารางใหม่ — ต้องรัน `ANALYZE orders;` เพื่อแก้ไข

</details>

---

### แบบฝึกหัดที่ 10 (แบบฝึกหัดรวม — วินิจฉัย query ซับซ้อน)

โจทย์: ทีมงานรายงานว่า dashboard "รายงานยอดขายตามหมวดหมู่สินค้า" ที่ใช้ query ต่อไปนี้เริ่มทำงานช้าลงเรื่อย ๆ ให้รัน `EXPLAIN (ANALYZE, BUFFERS)` แล้ววินิจฉัยปัญหาทั้งหมดที่พบ พร้อมเสนอแนวทางแก้ไขอย่างเป็นระบบ:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT
    cat.category_name,
    count(DISTINCT o.order_id) AS order_count,
    sum(oi.quantity * oi.unit_price) AS revenue,
    avg(oi.unit_price) AS avg_item_price
FROM order_items oi
JOIN products p ON p.product_id = oi.product_id
JOIN categories cat ON cat.category_id = p.category_id
JOIN orders o ON o.order_id = oi.order_id
WHERE o.status IN ('paid', 'shipped', 'delivered')
  AND o.order_date >= now() - INTERVAL '90 days'
GROUP BY cat.category_name
ORDER BY revenue DESC;
```

<details>
<summary>เฉลย</summary>

ผลลัพธ์ตัวอย่าง (จำลองสถานการณ์ที่มีทั้งปัญหา statistics และ join order ที่ไม่เหมาะสม):

```
                                                                 QUERY PLAN
----------------------------------------------------------------------------------------------------------------------------------------------
 Sort  (cost=1876.23..1876.25 rows=10 width=52) (actual time=145.201..145.204 rows=10 loops=1)
   Sort Key: (sum((oi.quantity * oi.unit_price))) DESC
   Sort Method: quicksort  Memory: 26kB
   ->  HashAggregate  (cost=1875.60..1875.90 rows=10 width=52) (actual time=145.156..145.187 rows=10 loops=1)
         Group Key: cat.category_name
         Batches: 1  Memory Usage: 24kB
         ->  Hash Join  (cost=175.50..1750.34 rows=4200 width=27) (actual time=8.201..135.887 rows=4587 loops=1)
               Hash Cond: (p.category_id = cat.category_id)
               ->  Hash Join  (cost=152.00..1698.45 rows=4200 width=15) (actual time=8.021..130.221 rows=4587 loops=1)
                     Hash Cond: (oi.order_id = o.order_id)
                     ->  Seq Scan on order_items oi  (cost=0.00..339.87 rows=19987 width=12) (actual time=0.008..8.102 rows=19987 loops=1)
                     ->  Hash  (cost=142.50..142.50 rows=760 width=8) (actual time=7.845..7.846 rows=1846 loops=1)
                           Buckets: 1024  Batches: 1  Memory Usage: 89kB
                           ->  Seq Scan on orders o  (cost=0.00..142.50 rows=760 width=8) (actual time=0.012..7.201 rows=1846 loops=1)
                                 Filter: (((status)::text = ANY ('{paid,shipped,delivered}'::text[]))
                                          AND (order_date >= (now() - '90 days'::interval)))
                                 Rows Removed by Filter: 11154
               ->  Hash  (cost=23.00..23.00 rows=1500 width=8) (actual time=0.145..0.146 rows=1500 loops=1)
                     ->  Seq Scan on products p  (cost=0.00..23.00 rows=1500 width=8) (actual time=0.005..0.198 rows=1500 loops=1)
         (join กับ categories ผ่าน Hash เพิ่มอีกชั้น เชื่อม cat.category_id)
 Planning Time: 0.512 ms
 Execution Time: 145.601 ms
 Buffers: shared hit=482 read=128
(20 rows)
```

**การวินิจฉัย (checklist ตามที่สรุปไว้ท้ายบท):**

1. **Estimate ผิดพลาดที่ node กรอง orders:** `rows=760 (planned)` แต่ `actual rows=1846` — ต่างกันเกือบ 2.5 เท่า สาเหตุน่าจะมาจาก:
   - เงื่อนไข `status IN (...)` รวมกับ `order_date >= ...` เป็นสองเงื่อนไขที่ Planner ประมาณ selectivity **แยกอิสระจากกัน** แล้วคูณกัน (independence assumption) ทั้งที่ในความเป็นจริงอาจมีความสัมพันธ์กัน (เช่น ออเดอร์ใหม่ ๆ มักจะยังไม่ถึงสถานะ cancelled) ทำให้ค่าประมาณต่ำกว่าความจริง
   - อาจเกิดจาก statistics ล้าสมัยหลังมีการเพิ่ม/เปลี่ยนสถานะออเดอร์จำนวนมากโดยไม่ได้ `ANALYZE`

2. **ผลกระทบต่อ join ที่อยู่เหนือขึ้นไป (compounding error):** เพราะ `orders` ประมาณผิดไป 2.5 เท่า ทำให้ `Hash` node ที่สร้างจาก orders (ใช้เป็น build side) มีขนาดใหญ่กว่าที่วางแผนไว้ (`Buckets: 1024` อาจน้อยเกินไปสำหรับ 1846 แถวจริง ทำให้เกิด hash collision มากกว่าที่ควร) และตัวเลข `rows=4200` ของ Hash Join ชั้นบนก็ถูกประมาณต่ำกว่าจริง (`actual rows=4587`) ตามไปด้วย

3. **ไม่มี index รองรับเงื่อนไข orders โดยตรง:** แม้จะมี `idx_orders_date` แต่เพราะ query กรองทั้ง `status` และ `order_date` ร่วมกัน (AND) การมี index แยกคอลัมน์เดียวอาจไม่ช่วยมากเท่า **composite index** ที่ครอบทั้งสองเงื่อนไข สังเกตว่า Planner เลือก Seq Scan บน orders ทั้งที่มี `idx_orders_date` อยู่ — เป็นเพราะ optimizer ประเมินว่าเงื่อนไข status filter จะลดจำนวนแถวจาก index scan ไม่คุ้มกับ overhead ของการ random I/O เมื่อเทียบกับ Seq Scan (โดยเฉพาะเมื่อ estimate ยังคลาดเคลื่อน)

4. **Buffers:** `shared hit=482 read=128` — มีการอ่านจาก disk จริง (`read=128`) ไม่ใช่ทั้งหมดจาก cache ซึ่งในระบบที่มี concurrent load สูงอาจทำให้ query ช้าลงเพิ่มเติมจากการแย่ง I/O

**แนวทางแก้ไขอย่างเป็นระบบ:**

```sql
-- 1) แก้ปัญหา statistics ล้าสมัยก่อนเป็นอันดับแรก (ทำง่ายสุด ผลกระทบเยอะสุด)
ANALYZE orders;
ANALYZE order_items;

-- 2) สร้าง composite index ให้ครอบทั้ง status และ order_date
--    (เรียง order_date ไว้หลัง เพราะมักใช้ range query ตามหลัง equality/IN)
CREATE INDEX idx_orders_status_date ON orders(status, order_date);

-- 3) ถ้า dashboard นี้ถูกเรียกบ่อยและ real-time ไม่จำเป็นต้องเป๊ะวินาที
--    ให้พิจารณาสร้าง materialized view สรุปยอดขายรายหมวดหมู่ + refresh เป็นรอบ
--    (จะกล่าวถึงรายละเอียดเพิ่มเติมใน Part ถัดไปเรื่อง Query Optimization)

-- 4) หลังแก้ไข ให้รัน EXPLAIN ANALYZE ซ้ำอีกครั้งเพื่อเทียบผล
--    ตรวจสอบว่า planned rows ใกล้เคียง actual rows มากขึ้น
--    และ Execution Time ลดลงจริง
```

**บทเรียนสำคัญจากแบบฝึกหัดนี้:** ปัญหา performance ของ query ที่ซับซ้อนมักไม่ได้เกิดจากสาเหตุเดียว แต่เป็น**การสะสมของความคลาดเคลื่อน** (compounding errors) ที่เริ่มจาก estimate ผิดในจุดเล็ก ๆ (node ด้านล่างสุดของ tree) แล้วขยายผลไปเรื่อย ๆ ตาม join ที่อยู่เหนือขึ้นไป การวินิจฉัยที่ดีจึงต้อง**เริ่มจากล่างขึ้นบน** (bottom-up) เสมอ — แก้ node ที่ estimate ผิดมากที่สุดก่อน แล้วรัน EXPLAIN ANALYZE ซ้ำเพื่อดูว่า plan ทั้งหมดดีขึ้นหรือไม่ ก่อนจะไล่แก้ node ถัดไป

</details>

---

## สรุปและก้าวต่อไป

ใน Part นี้เราได้เรียนรู้พื้นฐานที่สำคัญที่สุดอย่างหนึ่งของการทำงานกับ PostgreSQL ในระดับมืออาชีพ — การอ่านและตีความ `EXPLAIN`/`EXPLAIN ANALYZE` เพื่อเข้าใจว่า Query Planner ตัดสินใจอย่างไร และทำไมบาง query ถึงช้าโดยไม่คาดคิด ทักษะนี้คือรากฐานที่จำเป็นก่อนจะไปเรียนรู้เทคนิคการ**ปรับปรุงประสิทธิภาพ (query optimization)** เชิงลึกต่อไป เช่น การเลือกชนิด index ที่เหมาะสม, การเขียน query ใหม่ให้ Planner เลือกแผนที่ดีขึ้น, การจัดการ partition, และการปรับแต่งค่า configuration ต่าง ๆ ให้เหมาะกับ workload จริง

**บทถัดไป:** [Part 045: Query Optimization](./part-045-query-optimization.md)
