# Part 043: Partial Index, Expression Index และ Multi-column Index

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับสูง | Part 043

---

## เป้าหมายการเรียนรู้

เมื่อจบบทนี้ ผู้เรียนจะสามารถ:

1. อธิบายได้ว่า **Partial Index** คืออะไร และรู้ว่าเมื่อไหร่ควรใช้เพื่อลดขนาด index และเพิ่มความเร็วในการค้นหาข้อมูลกลุ่มย่อย
2. ออกแบบ Partial Index สำหรับสถานการณ์จริง เช่น การค้นหา order ที่ยัง pending อยู่ ในตารางที่ส่วนใหญ่เป็น delivered แล้ว
3. ใช้ Partial Unique Index เพื่อสร้าง constraint แบบมีเงื่อนไข (conditional uniqueness) เช่น อีเมลต้องไม่ซ้ำเฉพาะผู้ใช้ที่ยัง active
4. อธิบายและสร้าง **Expression Index (Functional Index)** เพื่อเร่งความเร็ว query ที่ใช้ function ครอบคอลัมน์ เช่น `LOWER(email)` หรือ `DATE(order_date)`
5. เข้าใจข้อจำกัดเรื่อง IMMUTABLE function ในการสร้าง expression index และวิธีแก้ปัญหาที่พบบ่อย
6. ออกแบบ **Multi-column (Composite) Index** อย่างถูกต้องตามหลัก leftmost prefix rule
7. เลือกลำดับคอลัมน์ใน composite index ให้เหมาะสมกับรูปแบบ query จริง โดยอ้างอิงจาก EXPLAIN ANALYZE
8. ใช้ **Covering Index** ด้วย `INCLUDE` clause เพื่อให้เกิด Index Only Scan และลดการเข้าถึง heap
9. วิเคราะห์ index ที่ไม่ถูกใช้งานด้วย `pg_stat_user_indexes` และตัดสินใจลบ index ที่ไม่จำเป็นอย่างปลอดภัย
10. ออกแบบชุด index ทั้งหมด (partial + expression + composite + covering) ให้เหมาะกับ query workload จริงของระบบ e-commerce

---

## เตรียมข้อมูล

บทนี้ใช้ schema ฐานเดียวกับ Part 041 (ระบบ e-commerce จำลอง) โดยจะสร้างข้อมูลจำนวนมากพอสมควร และจงใจทำให้การกระจายตัวของ `orders.status` **เบ้ (skewed)** ไปทาง `'delivered'` มาก ๆ เพื่อให้เห็นประโยชน์ของ Partial Index ได้ชัดเจน

```sql
-- ลบตารางเดิมถ้ามี (ระวังในสภาพแวดล้อมจริง)
DROP TABLE IF EXISTS order_items, orders, customers, products, suppliers, categories CASCADE;

CREATE TABLE categories (
    category_id         SERIAL PRIMARY KEY,
    category_name       VARCHAR(100) NOT NULL,
    parent_category_id  INTEGER REFERENCES categories(category_id)
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
```

### สร้างข้อมูล categories และ suppliers

```sql
INSERT INTO categories (category_name, parent_category_id) VALUES
('อิเล็กทรอนิกส์', NULL),
('คอมพิวเตอร์และแล็ปท็อป', 1),
('มือถือและแท็บเล็ต', 1),
('เสื้อผ้าแฟชั่น', NULL),
('บ้านและสวน', NULL),
('หนังสือ', NULL),
('ของเล่นและงานอดิเรก', NULL),
('กีฬาและกิจกรรมกลางแจ้ง', NULL),
('ความงามและสุขภาพ', NULL),
('ของใช้สำนักงาน', NULL);

INSERT INTO suppliers (supplier_name, country)
SELECT
    'Supplier ' || gs,
    (ARRAY['Thailand','China','Japan','USA','Germany','Vietnam','South Korea','India'])
        [1 + floor(random() * 8)::int]
FROM generate_series(1, 30) AS gs;
```

### สร้างข้อมูล products (~800 แถว)

```sql
INSERT INTO products (product_name, category_id, supplier_id, unit_price, stock_quantity, is_active)
SELECT
    'Product ' || gs,
    1 + floor(random() * 10)::int,
    1 + floor(random() * 30)::int,
    round((random() * 4950 + 50)::numeric, 2),
    floor(random() * 500)::int,
    (random() > 0.08)   -- ประมาณ 92% active, 8% ถูกเลิกขาย
FROM generate_series(1, 800) AS gs;
```

### สร้างข้อมูล customers (~2,000 แถว)

```sql
INSERT INTO customers (first_name, last_name, email, country, signup_date)
SELECT
    'FirstName' || gs,
    'LastName' || gs,
    'customer' || gs || '@example.com',
    (ARRAY['Thailand','Singapore','Malaysia','Vietnam','USA','UK','Australia','Japan'])
        [1 + floor(random() * 8)::int],
    CURRENT_DATE - floor(random() * 1500)::int
FROM generate_series(1, 2000) AS gs;
```

### สร้างข้อมูล orders (~8,000 แถว) — จงใจเบ้ไปทาง 'delivered'

นี่คือจุดสำคัญของบทนี้: ในระบบ e-commerce จริง ออเดอร์ส่วนใหญ่จะถูกจัดส่งจนสำเร็จ (`delivered`) ไปแล้ว มีเพียงส่วนน้อยที่ยังอยู่ในสถานะ `pending` (รอดำเนินการ) ซึ่งเป็นสถานะที่ทีมปฏิบัติการต้อง query บ่อยที่สุดเพื่อเร่งจัดส่ง

```sql
INSERT INTO orders (customer_id, order_date, status, ship_country)
SELECT
    1 + floor(random() * 2000)::int,
    now() - (floor(random() * 400) || ' days')::interval
            - (floor(random() * 86400) || ' seconds')::interval,
    CASE
        WHEN r < 0.75 THEN 'delivered'   -- ~75%
        WHEN r < 0.90 THEN 'shipped'     -- ~15%
        WHEN r < 0.97 THEN 'cancelled'   -- ~7%
        ELSE 'pending'                   -- ~3%
    END,
    (ARRAY['Thailand','Singapore','Malaysia','Vietnam','USA','UK','Australia','Japan'])
        [1 + floor(random() * 8)::int]
FROM (
    SELECT gs, random() AS r
    FROM generate_series(1, 8000) AS gs
) t;
```

### สร้างข้อมูล order_items (~16,000 แถว)

```sql
INSERT INTO order_items (order_id, product_id, quantity, unit_price)
SELECT
    o.order_id,
    1 + floor(random() * 800)::int,
    1 + floor(random() * 5)::int,
    round((random() * 4950 + 50)::numeric, 2)
FROM orders o
CROSS JOIN LATERAL generate_series(1, 1 + floor(random() * 3)::int) AS item_no;
```

### ตรวจสอบการกระจายตัวของสถานะ order และอัปเดตสถิติ

```sql
ANALYZE categories, suppliers, products, customers, orders, order_items;

SELECT status, count(*) AS jumlah,
       round(100.0 * count(*) / sum(count(*)) OVER (), 2) AS percent
FROM orders
GROUP BY status
ORDER BY jumlah DESC;
```

ผลลัพธ์ตัวอย่าง (ตัวเลขจริงจะต่างกันเล็กน้อยเพราะใช้ `random()`):

```
  status   | jumlah | percent
-----------+--------+---------
 delivered |   5998 |   74.98
 shipped   |   1204 |   15.05
 cancelled |    560 |    7.00
 pending   |    238 |    2.98
```

จะเห็นว่า `pending` มีสัดส่วนแค่ประมาณ 3% ของทั้งตาราง — นี่คือสถานการณ์คลาสสิกที่ Partial Index เหมาะสมที่สุด

---

## Step 421: Partial Index คืออะไร

**Partial Index** คือ index ที่ไม่ครอบคลุมทุกแถวของตาราง แต่ครอบคลุมเฉพาะแถวที่ตรงตามเงื่อนไขใน `WHERE` clause ตอนสร้าง index เท่านั้น

รูปแบบทั่วไป:

```sql
CREATE INDEX index_name
    ON table_name (column_list)
    WHERE condition;
```

### ทำไมต้องใช้ Partial Index

ในตารางขนาดใหญ่ที่มีค่าบางค่ากระจุกตัวอยู่มาก (skewed distribution) การสร้าง index แบบปกติ (full index) บนคอลัมน์นั้นจะทำให้:

1. **index มีขนาดใหญ่โดยไม่จำเป็น** — เพราะต้องเก็บ entry ของทุกแถว แม้ว่า query ส่วนใหญ่จะสนใจแค่แถวส่วนน้อย
2. **VACUUM และ UPDATE ช้าลง** — เพราะทุกครั้งที่มีการเปลี่ยนแปลงข้อมูลในคอลัมน์ที่ทำ index ต้องอัปเดต index ทั้งหมด แม้ว่าแถวส่วนใหญ่จะไม่เคยถูก query ผ่าน index นั้นเลย
3. **planner scan ข้อมูลที่ไม่เกี่ยวข้องโดยเปล่าประโยชน์** — หากค่าที่ query หาพบได้น้อยมาก (selective) planner ก็ยังต้องเดินผ่าน index entry ของค่าที่ไม่เกี่ยวข้องเป็นจำนวนมาก (ในกรณี full index ที่ไม่ selective)

Partial Index แก้ปัญหานี้โดยสร้าง index เฉพาะ "ส่วนที่มีประโยชน์จริง" เท่านั้น ทำให้:

- ขนาด index เล็กลงมาก (เล็กลงตามสัดส่วนของแถวที่เข้าเงื่อนไข)
- อยู่ใน memory cache (shared_buffers) ได้ง่ายกว่า เพราะเล็กกว่า
- เขียน/อัปเดตเร็วขึ้น เพราะแถวที่ไม่เข้าเงื่อนไขไม่ต้องแตะ index เลย
- Planner เลือกใช้ index นี้ได้อย่างมั่นใจเมื่อ query มีเงื่อนไขที่ **ครอบคลุม (implies)** เงื่อนไขของ index

### ตัวอย่างเบื้องต้น

```sql
-- Full index: ครอบคลุมทุกแถวของ orders (8,000 แถว)
CREATE INDEX idx_orders_status_full ON orders (status);

-- Partial index: ครอบคลุมเฉพาะแถวที่ status = 'pending' (~240 แถว)
CREATE INDEX idx_orders_status_pending ON orders (status)
    WHERE status = 'pending';
```

ทั้งสอง index เก็บข้อมูลจากคอลัมน์เดียวกัน แต่ partial index มีขนาดเล็กกว่ามาก เราจะเปรียบเทียบขนาดจริงและ query plan ใน Step ถัดไป

> **ข้อสังเกตสำคัญ:** เงื่อนไขใน `WHERE` ของ Partial Index ไม่จำเป็นต้องเป็นคอลัมน์เดียวกับที่ index — สามารถใช้คอลัมน์อื่นเป็นเงื่อนไข (predicate) ในขณะที่ index จริง ๆ ทำบนคอลัมน์อื่นก็ได้ เช่น `CREATE INDEX ON orders (order_date) WHERE status = 'pending';`

---

## Step 422: ตัวอย่าง Partial Index จริง — Index เฉพาะ Order ที่ status = 'pending'

มาดูของจริงกันด้วยตาราง `orders` ที่เราเพิ่งสร้างข้อมูลไป ซึ่งมี `pending` เพียง ~3% เท่านั้น

### ลบ index ทดลองก่อนหน้า และเริ่มใหม่ให้สะอาด

```sql
DROP INDEX IF EXISTS idx_orders_status_full;
DROP INDEX IF EXISTS idx_orders_status_pending;
```

### สร้าง full index และวัดขนาด

```sql
CREATE INDEX idx_orders_status_full ON orders (status);

SELECT pg_size_pretty(pg_relation_size('idx_orders_status_full')) AS full_index_size;
```

ผลลัพธ์ตัวอย่าง:

```
 full_index_size
------------------
 224 kB
```

### สร้าง partial index และวัดขนาด

```sql
CREATE INDEX idx_orders_status_pending ON orders (status)
    WHERE status = 'pending';

SELECT pg_size_pretty(pg_relation_size('idx_orders_status_pending')) AS partial_index_size;
```

ผลลัพธ์ตัวอย่าง:

```
 partial_index_size
---------------------
 16 kB
```

จะเห็นว่า partial index มีขนาดเล็กกว่า full index หลายเท่า (ในตัวอย่างนี้เล็กลงประมาณ 14 เท่า) เพราะเก็บ entry แค่ ~240 แถว แทนที่จะเป็น 8,000 แถว

### เปรียบเทียบขนาดแบบตาราง

```sql
SELECT
    indexrelname,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
WHERE relname = 'orders'
  AND indexrelname LIKE 'idx_orders_status%';
```

```
       indexrelname        | index_size
----------------------------+------------
 idx_orders_status_full     | 224 kB
 idx_orders_status_pending  | 16 kB
```

### ทดสอบ query ที่ทีมปฏิบัติการใช้บ่อย: หา order ที่ยัง pending

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT order_id, customer_id, order_date, ship_country
FROM orders
WHERE status = 'pending'
ORDER BY order_date;
```

ผลลัพธ์ตัวอย่าง (planner เลือก partial index เพราะเงื่อนไข `status = 'pending'` ตรงกับ predicate ของ index พอดี และ index มีขนาดเล็กมาก):

```
 Sort  (cost=18.45..19.05 rows=238 width=24) (actual time=0.412..0.428 rows=238 loops=1)
   Sort Key: order_date
   Sort Method: quicksort  Memory: 40kB
   Buffers: shared hit=6
   ->  Index Scan using idx_orders_status_pending on orders
         (cost=0.14..9.21 rows=238 width=24) (actual time=0.021..0.180 rows=238 loops=1)
         Buffers: shared hit=4
 Planning Time: 0.156 ms
 Execution Time: 0.462 ms
```

สังเกตว่า:
- Planner ใช้ `idx_orders_status_pending` โดยตรง ไม่ต้อง filter เพิ่มเติม เพราะเงื่อนไข query กับเงื่อนไข index ตรงกันเป๊ะ
- จำนวน `Buffers: shared hit` ต่ำมาก (แค่ไม่กี่ block) เพราะ index เล็กพอที่จะอยู่ใน cache ได้ทั้งหมด

### ลอง DROP full index แล้วดูว่ายังใช้งานได้ปกติหรือไม่

ในสถานการณ์จริง ถ้าไม่มี query ใดต้องการค้นหา status อื่น ๆ นอกจาก `pending` เราอาจไม่จำเป็นต้องมี full index เลยด้วยซ้ำ:

```sql
DROP INDEX idx_orders_status_full;

-- query สำหรับ pending ยังทำงานได้เร็วเหมือนเดิม เพราะใช้ partial index
EXPLAIN ANALYZE
SELECT order_id FROM orders WHERE status = 'pending';
```

แต่ถ้ามี query ที่ค้นหา status อื่น เช่น `WHERE status = 'shipped'` query นั้นจะ **ไม่สามารถใช้ partial index นี้ได้** (เพราะเงื่อนไข `status = 'shipped'` ไม่ implied จาก predicate `status = 'pending'`) และจะกลับไปใช้ Seq Scan แทน:

```sql
EXPLAIN ANALYZE
SELECT order_id FROM orders WHERE status = 'shipped';
```

```
 Seq Scan on orders  (cost=0.00..170.00 rows=1204 width=4)
                      (actual time=0.015..2.340 rows=1204 loops=1)
   Filter: ((status)::text = 'shipped'::text)
   Rows Removed by Filter: 6796
 Planning Time: 0.098 ms
 Execution Time: 2.512 ms
```

> **บทเรียนสำคัญ:** Partial Index ไม่ใช่ index อเนกประสงค์ มันถูกออกแบบมาเพื่อ query ที่มีเงื่อนไขตรงกับ (หรือแคบกว่า) predicate ของ index เท่านั้น ถ้าต้องรองรับหลาย status ให้พิจารณา full index (หรือ composite index) แทน หรือสร้าง partial index หลายตัวสำหรับแต่ละ status ที่มีความสำคัญ (เช่น `pending` และ `cancelled` ที่ต้องติดตามบ่อย)

สร้าง full index กลับคืนไว้สำหรับใช้ในบทถัดไป:

```sql
CREATE INDEX idx_orders_status_full ON orders (status);
```

---

## Step 423: Partial Index สำหรับ Unique Constraint แบบมีเงื่อนไข

Partial Index ไม่ได้มีประโยชน์แค่เรื่องขนาดหรือความเร็วเท่านั้น แต่ยังใช้สร้าง **conditional unique constraint** ได้ด้วย — เป็นสิ่งที่ constraint ปกติของ PostgreSQL ทำไม่ได้โดยตรง

### สถานการณ์: ต้องการให้อีเมลไม่ซ้ำ แต่เฉพาะผู้ใช้ที่ยัง active

สมมติระบบอนุญาตให้ปิดการใช้งานบัญชี (deactivate) และต้องการให้ **อีเมลของบัญชีที่ปิดไปแล้วสามารถถูกนำไปสมัครใหม่ได้** (เช่น ลูกค้าลบบัญชีแล้วสมัครใหม่ด้วยอีเมลเดิม) แต่อีเมลของบัญชีที่ยัง active ต้องไม่ซ้ำกันเด็ดขาด

ก่อนอื่น เราต้องเพิ่มคอลัมน์ `is_active` ให้ตาราง `customers` (ตาราง `products` มีอยู่แล้ว แต่ `customers` ยังไม่มี):

```sql
ALTER TABLE customers ADD COLUMN is_active BOOLEAN NOT NULL DEFAULT true;

-- จำลองว่ามีลูกค้าประมาณ 10% ที่ถูกปิดการใช้งานไปแล้ว
UPDATE customers
SET is_active = false
WHERE customer_id IN (
    SELECT customer_id FROM customers
    ORDER BY random()
    LIMIT (SELECT count(*) / 10 FROM customers)
);
```

`customers.email` เดิมมี `UNIQUE` constraint อยู่แล้ว (ซึ่งสร้าง unique index แบบ full โดยอัตโนมัติ) แต่ constraint นี้ **บังคับความไม่ซ้ำกันในทุกแถว** ไม่ว่า active หรือไม่ ถ้าต้องการให้อีเมลของบัญชี inactive ใช้ซ้ำกันได้ ต้องถอด constraint เดิมออกก่อน แล้วแทนที่ด้วย partial unique index:

```sql
-- ถอด unique constraint เดิมออก (มันสร้าง index ชื่อ customers_email_key โดยอัตโนมัติ)
ALTER TABLE customers DROP CONSTRAINT customers_email_key;

-- สร้าง partial unique index: อีเมลต้องไม่ซ้ำ เฉพาะแถวที่ is_active = true
CREATE UNIQUE INDEX idx_customers_email_active_uniq
    ON customers (email)
    WHERE is_active = true;
```

### ทดสอบพฤติกรรม

```sql
-- 1) ลองเพิ่มลูกค้าใหม่ที่ active ด้วยอีเมลซ้ำกับคนที่ active อยู่แล้ว -> ต้อง error
INSERT INTO customers (first_name, last_name, email, country, is_active)
VALUES ('Test', 'Duplicate', 'customer1@example.com', 'Thailand', true);
```

```
ERROR:  duplicate key value violates unique constraint "idx_customers_email_active_uniq"
DETAIL:  Key (email)=(customer1@example.com) already exists.
```

```sql
-- 2) ลองเพิ่มลูกค้าใหม่ด้วยอีเมลที่ "เคยมีอยู่แล้วแต่บัญชีนั้น inactive" -> ต้องผ่าน
-- สมมติ customer_id = 5 ถูกปิดใช้งานไปแล้ว (is_active = false)
SELECT customer_id, email, is_active FROM customers WHERE customer_id = 5;
```

```
 customer_id |          email          | is_active
-------------+--------------------------+-----------
           5 | customer5@example.com   | f
```

```sql
INSERT INTO customers (first_name, last_name, email, country, is_active)
VALUES ('Reused', 'Email', 'customer5@example.com', 'Thailand', true);
-- สำเร็จ! เพราะ constraint ครอบคลุมเฉพาะแถวที่ is_active = true
```

### ทำไมวิธีนี้ถึงใช้งานได้

เมื่อ PostgreSQL ตรวจสอบ unique index มันจะพิจารณาเฉพาะแถวที่ **เข้าเงื่อนไข predicate ของ index** เท่านั้น การ insert แถวใหม่ที่ `is_active = true` และ email ที่มีอยู่แล้วเฉพาะในแถวที่ `is_active = false` จะไม่ชนกัน เพราะแถวเดิมไม่ได้ถูกนับรวมใน index นี้ตั้งแต่แรก

### ข้อควรระวัง

- แถวที่ `is_active = false` **ไม่ได้ถูกป้องกัน** โดย index นี้เลย ดังนั้นอาจมี email ซ้ำกันได้หลายแถวในกลุ่ม inactive ซึ่งอาจเป็นสิ่งที่ยอมรับได้หรือไม่ก็ได้ ขึ้นอยู่กับ business rule
- หากต้องการ query อีเมล (ไม่ว่า active หรือไม่) ให้เร็วด้วย ควรพิจารณาสร้าง full index บน `email` แยกต่างหาก (ไม่ unique) เพื่อรองรับ query ทั่วไป
- Partial unique index แบบนี้ **ใช้แทน CHECK constraint ข้ามแถว (cross-row) ไม่ได้ทั้งหมด** — มันใช้ได้ดีเฉพาะกรณี "ไม่ซ้ำภายใต้เงื่อนไข" เท่านั้น

---

## Step 424: Expression Index (Functional Index) คืออะไร

**Expression Index** (บางครั้งเรียก Functional Index) คือ index ที่สร้างจาก **ผลลัพธ์ของ expression หรือ function** ที่กระทำกับคอลัมน์ แทนที่จะสร้างจากค่าคอลัมน์ตรง ๆ

รูปแบบทั่วไป:

```sql
CREATE INDEX index_name
    ON table_name (expression);
```

โดยที่ `expression` ต้องอยู่ในวงเล็บถ้าเป็น expression ที่ซับซ้อนกว่าการเรียก function ตัวเดียว เช่น `((a + b) * c)`

### ทำไมต้องใช้ Expression Index

ปัญหาคลาสสิกคือ: ถ้าเรามี index ปกติบนคอลัมน์ `email` แต่ query ค้นหาด้วย `WHERE LOWER(email) = 'customer1@example.com'` planner **จะไม่สามารถใช้ index บน `email` ได้เลย** เพราะค่าที่เก็บใน index คือค่า `email` ดิบ ๆ ไม่ใช่ผลลัพธ์ของ `LOWER(email)`

ลองดูตัวอย่าง:

```sql
-- มี index ปกติอยู่แล้วจาก UNIQUE constraint เดิม (ถ้ายังไม่ถอดออก)
-- แต่ในที่นี้เราถอด constraint เดิมออกไปแล้วใน Step 423
-- ลองสร้าง index ปกติกลับมาเพื่อสาธิต
CREATE INDEX idx_customers_email_plain ON customers (email);

EXPLAIN ANALYZE
SELECT customer_id, email
FROM customers
WHERE LOWER(email) = 'customer1500@example.com';
```

ผลลัพธ์ตัวอย่าง (planner **ไม่สามารถใช้** `idx_customers_email_plain` ได้):

```
 Seq Scan on customers  (cost=0.00..46.50 rows=10 width=26)
                         (actual time=0.312..0.580 rows=1 loops=1)
   Filter: (lower((email)::text) = 'customer1500@example.com'::text)
   Rows Removed by Filter: 1999
 Planning Time: 0.084 ms
 Execution Time: 0.601 ms
```

ตารางนี้มีแค่ 2,000 แถว จึงยังเร็วอยู่ แต่ถ้าเป็นตารางหลักล้านแถว การ Seq Scan ทุกครั้งที่มีการ login (case-insensitive) จะกลายเป็นคอขวดสำคัญทันที

### หลักการทำงานของ Expression Index

เมื่อสร้าง `CREATE INDEX ... ON table (LOWER(column))` PostgreSQL จะ:

1. คำนวณค่า `LOWER(column)` ของทุกแถว ณ ตอนสร้าง index
2. เก็บค่าที่คำนวณได้นั้นไว้ใน B-tree (หรือ index type อื่น) แทนค่าคอลัมน์ดิบ
3. เมื่อมี query ที่ WHERE clause ตรงกับ expression เป๊ะ (`LOWER(email) = ...`) planner จะจับคู่และใช้ index นี้ได้

### ข้อกำหนดสำคัญ: Function ต้องเป็น IMMUTABLE

PostgreSQL อนุญาตให้ใช้เฉพาะ function ที่ถูก mark เป็น **IMMUTABLE** ในการสร้าง expression index เท่านั้น เพราะ index เก็บค่าที่คำนวณไว้ล่วงหน้า ถ้า function ให้ผลลัพธ์ต่างกันในแต่ละครั้งที่เรียก (เช่นขึ้นกับ timezone, locale, หรือเวลาปัจจุบัน) ค่าที่เก็บใน index จะกลายเป็นข้อมูลที่ไม่ถูกต้องได้

```sql
-- LOWER() เป็น IMMUTABLE -> ใช้ได้
CREATE INDEX idx_customers_email_lower ON customers (LOWER(email));
```

เราจะเจาะลึกเรื่อง IMMUTABLE function กับปัญหาที่พบบ่อย (เช่น `DATE(timestamptz)`) ใน Step ถัดไป

---

## Step 425: ตัวอย่าง Expression Index — LOWER(email) และ DATE(order_date)

### 425.1 Case-insensitive email search ด้วย LOWER(email)

```sql
-- ลบ index ปกติที่ไม่ช่วยอะไรทิ้งไป (ถ้าไม่ต้องการ exact-match ปกติ)
DROP INDEX IF EXISTS idx_customers_email_plain;

CREATE INDEX idx_customers_email_lower ON customers (LOWER(email));
```

ตรวจสอบขนาด:

```sql
SELECT pg_size_pretty(pg_relation_size('idx_customers_email_lower'));
```

```
 pg_size_pretty
-----------------
 88 kB
```

ทดสอบ query login ที่ใช้ case-insensitive comparison (รูปแบบที่พบบ่อยมากในระบบ authentication):

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT customer_id, email, is_active
FROM customers
WHERE LOWER(email) = LOWER('Customer1500@Example.com');
```

ผลลัพธ์ตัวอย่าง:

```
 Index Scan using idx_customers_email_lower on customers
     (cost=0.28..8.30 rows=1 width=27) (actual time=0.028..0.031 rows=1 loops=1)
   Index Cond: (lower((email)::text) = 'customer1500@example.com'::text)
   Buffers: shared hit=3
 Planning Time: 0.145 ms
 Execution Time: 0.052 ms
```

จาก Seq Scan (0.6 ms) เหลือ Index Scan (0.05 ms) — เร็วขึ้นกว่า 10 เท่าแม้ในตารางเล็ก ๆ และช่องว่างนี้จะยิ่งถ่างมากขึ้นเรื่อย ๆ เมื่อข้อมูลโตขึ้น

> **หมายเหตุ:** ต้องเขียน WHERE clause ให้ตรงกับ expression ของ index เป๊ะ ๆ (`LOWER(email) = ...`) ถ้าเขียนแค่ `WHERE email = 'Customer1500@Example.com'` โดยไม่ครอบ `LOWER()` planner จะไม่มองเห็นความสัมพันธ์กับ expression index นี้เลย

### 425.2 Filter ตามวันที่จาก TIMESTAMPTZ ด้วย DATE()

สถานการณ์: ทีมรายงานยอดขายต้องการ query "ออเดอร์ทั้งหมดของวันที่ X" จากคอลัมน์ `order_date` ที่เป็น `TIMESTAMPTZ` (มีทั้งวันและเวลา)

ลองสร้าง expression index แบบตรงไปตรงมาก่อน:

```sql
CREATE INDEX idx_orders_order_date_day ON orders (DATE(order_date));
```

ผลลัพธ์:

```
ERROR:  functions in index expression must be marked IMMUTABLE
```

### ทำไมถึง error

ฟังก์ชัน `date(timestamptz)` (การแปลง `TIMESTAMPTZ` เป็น `DATE`) มีผลลัพธ์ที่ **ขึ้นอยู่กับค่า `timezone` ของ session** ขณะนั้น เวลาเดียวกันในหน่วย UTC อาจตกคนละวันได้เมื่อดูผ่าน timezone ที่ต่างกัน (เช่น `2026-01-01 00:30:00+00` เป็นวันที่ 1 มกราคม ตาม UTC แต่เป็นวันที่ 31 ธันวาคมปีก่อนหน้าตาม `America/New_York`) PostgreSQL จึง mark function นี้ว่า **STABLE** (ไม่ใช่ IMMUTABLE) และปฏิเสธไม่ให้ใช้สร้าง index โดยตรง

### วิธีแก้: บังคับ timezone ให้คงที่ก่อนแปลงเป็น DATE

```sql
CREATE INDEX idx_orders_order_date_day
    ON orders (((order_date AT TIME ZONE 'UTC')::date));
```

เมื่อเราบังคับ timezone เป็นค่าคงที่ (`'UTC'`) เสียก่อน ผลลัพธ์ของ expression ทั้งหมด `(order_date AT TIME ZONE 'UTC')::date` จะไม่ขึ้นกับ session timezone อีกต่อไป และกลายเป็น IMMUTABLE ในทางปฏิบัติ ทำให้สร้าง index ได้สำเร็จ

ตรวจสอบขนาด:

```sql
SELECT pg_size_pretty(pg_relation_size('idx_orders_order_date_day'));
```

```
 pg_size_pretty
-----------------
 184 kB
```

ทดสอบ query รายงานยอดขายรายวัน — **ต้องเขียน WHERE clause ให้ตรงกับ expression เป๊ะ**:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT order_id, customer_id, status
FROM orders
WHERE (order_date AT TIME ZONE 'UTC')::date = '2026-06-15';
```

ผลลัพธ์ตัวอย่าง:

```
 Index Scan using idx_orders_order_date_day on orders
     (cost=0.29..8.61 rows=20 width=13) (actual time=0.033..0.058 rows=21 loops=1)
   Index Cond: (((order_date AT TIME ZONE 'UTC'::text))::date = '2026-06-15'::date)
   Buffers: shared hit=5
 Planning Time: 0.121 ms
 Execution Time: 0.079 ms
```

เทียบกับ query แบบ range scan บน `order_date` ตรง ๆ (ไม่ใช้ expression index):

```sql
EXPLAIN ANALYZE
SELECT order_id, customer_id, status
FROM orders
WHERE order_date >= '2026-06-15'::date
  AND order_date < '2026-06-16'::date;
```

วิธีนี้ก็ใช้ index แบบ range ได้เหมือนกัน (ถ้ามี index บน `order_date` ตรง ๆ) และ **ไม่ต้องพึ่ง expression index เลย** — ในทางปฏิบัติ วิธี range scan (`>=` และ `<`) มักจะยืดหยุ่นกว่าและเป็นที่นิยมมากกว่า expression index สำหรับ date filtering เพราะ range scan รองรับทั้ง exact date และ date range ได้ในตัว ในขณะที่ expression index เหมาะกับกรณีที่ query pattern ตายตัวและต้องการความเรียบง่ายของ SQL (`WHERE order_date::date = ...`) มากกว่า

> **ข้อคิดสำคัญ:** ก่อนสร้าง expression index บน date-truncation ให้พิจารณา range query (`>= start AND < end`) ก่อนเสมอ เพราะมักจะทำงานได้ดีเท่ากันหรือดีกว่า โดยไม่ต้องกังวลเรื่อง IMMUTABLE function เลย ใช้ expression index เมื่อ business logic บังคับให้ query ต้องเขียนเป็นรูปแบบ expression จริง ๆ (เช่น ORM บางตัว generate SQL แบบตายตัว)

### ตัวอย่าง Expression Index อื่นที่พบบ่อย

```sql
-- ค้นหาชื่อสินค้าแบบไม่สนตัวพิมพ์เล็ก-ใหญ่
CREATE INDEX idx_products_name_lower ON products (LOWER(product_name));

-- คำนวณราคารวมหลังหักส่วนลด (สมมติมี column discount_percent)
-- CREATE INDEX idx_products_net_price ON products ((unit_price * 0.9));

-- แยก domain จากอีเมลเพื่อ query ตาม domain
CREATE INDEX idx_customers_email_domain
    ON customers (split_part(email, '@', 2));
```

---

## Step 426: Multi-column (Composite) Index — Leftmost Prefix Rule

**Composite Index** (หรือ Multi-column Index) คือ index ที่สร้างจากหลายคอลัมน์รวมกัน โดยลำดับของคอลัมน์มีผลอย่างมากต่อประสิทธิภาพและความสามารถในการใช้งานของ index

```sql
CREATE INDEX index_name ON table_name (column_a, column_b, column_c);
```

### Leftmost Prefix Rule คืออะไร

PostgreSQL B-tree composite index จัดเรียงข้อมูลโดย **เรียงตามคอลัมน์แรกก่อน แล้วค่อยเรียงตามคอลัมน์ที่สองภายในกลุ่มที่คอลัมน์แรกเท่ากัน** (คล้ายการเรียงเบอร์โทรศัพท์: เรียงรหัสพื้นที่ก่อน แล้วค่อยเรียงเลขหมายภายในรหัสพื้นที่เดียวกัน)

ผลที่ตามมาคือ index จะถูกใช้งานได้อย่างมีประสิทธิภาพเมื่อ query filter **เริ่มจากคอลัมน์ซ้ายสุด (leftmost) ไปเรื่อย ๆ ตามลำดับ** — เรียกว่า "leftmost prefix rule"

### ตัวอย่าง: composite index บน (customer_id, status)

```sql
CREATE INDEX idx_orders_customer_status ON orders (customer_id, status);
```

| Query WHERE clause | ใช้ index นี้ได้หรือไม่ | เหตุผล |
|---|---|---|
| `WHERE customer_id = 100` | ได้ (เต็มประสิทธิภาพ) | ตรงกับ prefix ซ้ายสุด |
| `WHERE customer_id = 100 AND status = 'pending'` | ได้ (เต็มประสิทธิภาพ) | ตรงกับทั้งสองคอลัมน์ตามลำดับ |
| `WHERE status = 'pending'` | **ไม่ได้เต็มประสิทธิภาพ** | ข้าม leftmost column (`customer_id`) ไป — ต้องสแกนทั้ง index (Full Index Scan) ไม่ใช่ Index Range Scan |
| `WHERE customer_id > 100` | ได้ (range scan) | ยังคงเริ่มจาก leftmost column |
| `WHERE customer_id > 100 AND status = 'pending'` | ใช้ได้บางส่วน | ใช้ `customer_id` เป็น range ได้ แต่ `status` จะถูกใช้เป็น filter เพิ่มเติม ไม่ใช่ index condition โดยตรง (เพราะหลัง range condition แล้ว การเรียงของ status ภายในแต่ละ customer_id ไม่ต่อเนื่องกันอีกต่อไป) |

### ทดสอบจริงด้วย EXPLAIN

```sql
EXPLAIN ANALYZE
SELECT order_id, order_date FROM orders WHERE customer_id = 500;
```

```
 Index Scan using idx_orders_customer_status on orders
     (cost=0.29..8.45 rows=4 width=12) (actual time=0.024..0.028 rows=4 loops=1)
   Index Cond: (customer_id = 500)
 Planning Time: 0.089 ms
 Execution Time: 0.045 ms
```

```sql
EXPLAIN ANALYZE
SELECT order_id, order_date FROM orders WHERE status = 'pending';
```

```
 Bitmap Heap Scan on orders  (cost=15.20..95.40 rows=238 width=12)
                              (actual time=0.612..1.240 rows=238 loops=1)
   Recheck Cond: ((status)::text = 'pending'::text)
   Heap Blocks: exact=180
   ->  Bitmap Index Scan on idx_orders_status_pending
             (cost=0.00..15.14 rows=238 width=0) (actual time=0.201..0.201 rows=238 loops=1)
         Index Cond: ((status)::text = 'pending'::text)
 Planning Time: 0.102 ms
 Execution Time: 1.312 ms
```

สังเกตว่า query ที่สอง (`WHERE status = 'pending'`) planner **เลือกใช้ partial index `idx_orders_status_pending` ที่เราสร้างไว้ก่อนหน้า** แทนที่จะใช้ `idx_orders_customer_status` เลย เพราะ partial index ตรงกับเงื่อนไขและมีขนาดเล็กกว่ามาก นี่คือตัวอย่างที่ดีว่า planner จะเลือก index ที่ "คุ้มค่าที่สุด" เสมอเมื่อมีหลายตัวเลือก

หากลอง drop partial index ชั่วคราวเพื่อดูพฤติกรรมของ composite index ล้วน ๆ:

```sql
BEGIN;
DROP INDEX idx_orders_status_pending;

EXPLAIN ANALYZE
SELECT order_id, order_date FROM orders WHERE status = 'pending';
```

```
 Seq Scan on orders  (cost=0.00..170.00 rows=238 width=12)
                      (actual time=0.018..2.450 rows=238 loops=1)
   Filter: ((status)::text = 'pending'::text)
   Rows Removed by Filter: 7762
 Planning Time: 0.075 ms
 Execution Time: 2.601 ms
```

```sql
ROLLBACK;  -- ยกเลิกการ DROP index เพื่อคืนสถานะเดิม
```

จะเห็นว่าเมื่อไม่มี partial index อยู่ planner เลือก **Seq Scan แทนที่จะพยายามใช้ `idx_orders_customer_status`** เลย เพราะการค้นหาผ่าน composite index ที่ `status` ไม่ใช่ leftmost column จะต้องสแกนทุก entry ในทุกค่า `customer_id` เพื่อหาแถวที่ `status = 'pending'` ซึ่งไม่ต่างอะไรกับการสแกนทั้งตาราง (planner จึงมองว่า Seq Scan ถูกกว่า)

---

## Step 427: การเลือกลำดับคอลัมน์ใน Composite Index

การเลือกลำดับคอลัมน์ใน composite index มีหลักที่ควรพิจารณา 3 ข้อหลัก:

1. **Equality ก่อน Range** — คอลัมน์ที่ query ใช้เงื่อนไข `=` (equality) ควรอยู่ซ้ายกว่าคอลัมน์ที่ใช้เงื่อนไข range (`>`, `<`, `BETWEEN`) เสมอ เพราะหลังจากเจอ range condition แล้ว การเรียงลำดับของคอลัมน์ถัดไปจะไม่ต่อเนื่องอีกต่อไป
2. **Selectivity สูงไปต่ำ (เมื่อ query ใช้ equality ทั้งคู่)** — ถ้าทุกคอลัมน์ถูกใช้แบบ equality เหมือนกันหมด โดยทั่วไปควรใส่คอลัมน์ที่ **selective มากกว่า** (มีค่าไม่ซ้ำเยอะกว่า / กรองแถวออกได้มากกว่า) ไว้ก่อน แม้ว่าในทางทฤษฎี B-tree แบบ equality-equality จะให้ผลลัพธ์เหมือนกันไม่ว่าลำดับใด แต่การเรียง column ที่ selective กว่าไว้ก่อนช่วยให้ query ที่ใช้แค่ prefix บางส่วนมีประโยชน์มากกว่า
3. **ดูจาก query pattern จริงที่ใช้บ่อยที่สุด** — ให้ยึด "รูปแบบ query ที่เกิดขึ้นบ่อยในระบบจริง" เป็นหลักเสมอ ไม่ใช่ออกแบบตามทฤษฎีล้วน ๆ

### ทดลองเปรียบเทียบ: (customer_id, status) เทียบกับ (status, customer_id)

```sql
CREATE INDEX idx_orders_status_customer ON orders (status, customer_id);
```

ตอนนี้เรามี 2 composite index:
- `idx_orders_customer_status` → `(customer_id, status)`
- `idx_orders_status_customer` → `(status, customer_id)`

#### กรณีที่ 1: query "ประวัติการสั่งซื้อของลูกค้าคนหนึ่งที่ยัง pending"

```sql
EXPLAIN ANALYZE
SELECT order_id, order_date
FROM orders
WHERE customer_id = 500 AND status = 'pending';
```

```
 Index Scan using idx_orders_customer_status on orders
     (cost=0.29..8.46 rows=1 width=12) (actual time=0.026..0.028 rows=1 loops=1)
   Index Cond: ((customer_id = 500) AND ((status)::text = 'pending'::text))
 Planning Time: 0.187 ms
 Execution Time: 0.048 ms
```

planner เลือก `idx_orders_customer_status` เพราะ `customer_id = 500` selective มาก (แต่ละลูกค้ามีออเดอร์เฉลี่ยแค่ 4 รายการจาก 8,000 แถว) — เมื่อกรองด้วย `customer_id` ก่อน จำนวนแถวที่เหลือให้ตรวจ `status` ต่อมีน้อยมากอยู่แล้ว

#### กรณีที่ 2: query "ออเดอร์ pending ทั้งหมด เรียงตามลูกค้า" (dashboard ของทีมปฏิบัติการ)

```sql
EXPLAIN ANALYZE
SELECT order_id, customer_id, order_date
FROM orders
WHERE status = 'pending'
ORDER BY customer_id;
```

```
 Sort  (cost=15.90..16.50 rows=238 width=16) (actual time=0.520..0.545 rows=238 loops=1)
   Sort Key: customer_id
   Sort Method: quicksort  Memory: 40kB
   ->  Index Scan using idx_orders_status_customer on orders
             (cost=0.29..9.31 rows=238 width=16) (actual time=0.030..0.310 rows=238 loops=1)
         Index Cond: ((status)::text = 'pending'::text)
 Planning Time: 0.145 ms
 Execution Time: 0.580 ms
```

ในกรณีนี้ planner เลือก `idx_orders_status_customer` เพราะ `status = 'pending'` เป็น leftmost column ของ index นี้พอดี ทำให้ scan ได้ตรงกลุ่มทันทีโดยไม่ต้อง filter เพิ่ม (แม้ยังมี partial index ที่เล็กกว่าอยู่ — ในที่นี้สมมติว่า query นี้ยัง select เฉพาะ column ที่ partial index ไม่ครอบคลุมครบ จึงต้องพิจารณา cost รวม)

### สรุปการตัดสินใจ

| Composite Index | เหมาะกับ query pattern |
|---|---|
| `(customer_id, status)` | "ประวัติของลูกค้าคนหนึ่ง กรองด้วย status" — customer_id selective มาก ใส่ก่อน |
| `(status, customer_id)` | "รายการทั้งหมดของ status หนึ่ง ๆ เรียง/กรองต่อด้วยลูกค้า" — status เป็นเงื่อนไขหลักของ query |

**ข้อสรุปเชิงปฏิบัติ:** ถ้า query ส่วนใหญ่ของระบบเป็น "ดูประวัติของลูกค้า" ให้ใช้ `(customer_id, status)` แต่ถ้า query ส่วนใหญ่เป็น "รายงาน/dashboard ตามสถานะ" ให้ใช้ `(status, customer_id)` — ในทางปฏิบัติไม่จำเป็นต้องมีทั้งสอง index พร้อมกันเสมอไป เพราะแต่ละ index มีต้นทุนด้าน storage และ write overhead ให้เลือกตามรูปแบบ query ที่เกิดขึ้นบ่อยที่สุดจริง ๆ จาก `pg_stat_statements`

ลบ index ที่ไม่ได้ใช้ต่อออกเพื่อความสะอาด (จะอธิบายวิธีตรวจสอบอย่างเป็นระบบใน Step 429):

```sql
DROP INDEX idx_orders_status_customer;
```

---

## Step 428: Covering Index ด้วย INCLUDE Clause

### ปัญหา: Index Scan ยังต้องไปอ่าน Heap เพิ่ม

โดยปกติ แม้ query จะใช้ Index Scan ได้เต็มประสิทธิภาพ (index condition ตรงกับ WHERE clause ทั้งหมด) หาก query นั้น `SELECT` คอลัมน์ที่ **ไม่ได้อยู่ใน index** ด้วย PostgreSQL ยังต้องกลับไปอ่านข้อมูลจาก heap (ตารางจริง) เพิ่มเติมสำหรับทุกแถวที่พบ — ขั้นตอนนี้เรียกว่า "heap fetch"

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT customer_id, status
FROM orders
WHERE customer_id = 500;
```

```
 Index Scan using idx_orders_customer_status on orders
     (cost=0.29..8.46 rows=4 width=9) (actual time=0.022..0.026 rows=4 loops=1)
   Index Cond: (customer_id = 500)
   Buffers: shared hit=3
 Planning Time: 0.081 ms
 Execution Time: 0.042 ms
```

ในตัวอย่างนี้ query เลือกแค่ `customer_id` และ `status` ซึ่งทั้งคู่อยู่ใน index `idx_orders_customer_status` (คอลัมน์ที่ 1 และ 2) อยู่แล้ว จึงยัง**ไม่ต้อง** heap fetch — แต่ถ้าเพิ่ม `order_date` เข้าไปใน SELECT ด้วยล่ะ?

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT customer_id, status, order_date
FROM orders
WHERE customer_id = 500;
```

```
 Index Scan using idx_orders_customer_status on orders
     (cost=0.29..8.51 rows=4 width=17) (actual time=0.028..0.035 rows=4 loops=1)
   Index Cond: (customer_id = 500)
   Buffers: shared hit=6
 Planning Time: 0.085 ms
 Execution Time: 0.051 ms
```

Plan ยังเป็น "Index Scan" (ไม่ใช่ "Index Only Scan") เพราะ `order_date` ไม่ได้อยู่ใน index นี้ ทำให้ต้องเข้าไปอ่าน heap เพิ่มทุกแถว (`Buffers: shared hit` เพิ่มขึ้นจาก 3 เป็น 6) — ในตารางเล็ก ๆ ความต่างนี้ไม่มาก แต่ในตารางขนาดใหญ่และ query ที่ทำงานหนัก (high QPS) heap fetch ที่เพิ่มขึ้นนี้จะกลายเป็นต้นทุนที่มีนัยสำคัญ

### ทางแก้แบบเดิม: เพิ่มคอลัมน์เข้า index key โดยตรง

```sql
CREATE INDEX idx_orders_customer_status_date_v1
    ON orders (customer_id, status, order_date);
```

วิธีนี้ใช้งานได้ แต่มีข้อเสีย: คอลัมน์ `order_date` จะกลายเป็นส่วนหนึ่งของ **key ที่ใช้ในการเรียงลำดับและเปรียบเทียบ** ของ B-tree ด้วย ทำให้ index มีขนาดใหญ่ขึ้นและซับซ้อนขึ้นโดยไม่จำเป็น เพราะจริง ๆ แล้วเราแค่ต้องการ "เก็บค่า order_date ไว้เผื่ออ่าน" ไม่ได้ต้องการใช้มันเป็นเงื่อนไขค้นหาหรือเรียงลำดับเลย

### ทางแก้ที่ถูกต้อง: ใช้ INCLUDE clause

```sql
DROP INDEX IF EXISTS idx_orders_customer_status_date_v1;

CREATE INDEX idx_orders_customer_status_covering
    ON orders (customer_id, status)
    INCLUDE (order_date);
```

`INCLUDE` เพิ่มคอลัมน์เข้าไปเก็บใน index **เฉพาะที่ leaf node** โดยไม่นำไปใช้ในการเรียงลำดับ (sort order) หรือเป็นส่วนหนึ่งของ key เปรียบเทียบ ทำให้:

- ยังคง filter/search ได้เฉพาะ `customer_id` และ `status` เท่านั้น (ตามเดิม)
- แต่เพิ่มความสามารถให้ query ที่ `SELECT` คอลัมน์ `order_date` ด้วย **ไม่ต้อง heap fetch** เลย

ทดสอบอีกครั้ง:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT customer_id, status, order_date
FROM orders
WHERE customer_id = 500;
```

```
 Index Only Scan using idx_orders_customer_status_covering on orders
     (cost=0.29..4.55 rows=4 width=17) (actual time=0.019..0.022 rows=4 loops=1)
   Index Cond: (customer_id = 500)
   Heap Fetches: 0
   Buffers: shared hit=2
 Planning Time: 0.079 ms
 Execution Time: 0.036 ms
```

สังเกตความแตกต่างสำคัญ:

| | Index Scan (ไม่มี INCLUDE) | Index Only Scan (มี INCLUDE) |
|---|---|---|
| Plan node | `Index Scan` | `Index Only Scan` |
| Heap Fetches | ต้อง fetch ทุกแถว | `Heap Fetches: 0` |
| Buffers | 6 | 2 |
| Execution Time | 0.051 ms | 0.036 ms |

ในตัวอย่างเล็ก ๆ นี้ความต่างดูไม่มาก แต่ในระบบจริงที่มี traffic สูงและตารางขนาดหลักสิบล้านแถว การลด heap fetch ลงเป็นศูนย์ช่วยลด I/O และ CPU ได้อย่างมีนัยสำคัญ

### เงื่อนไขสำคัญของ Index Only Scan: Visibility Map

Index Only Scan จะทำงานได้เต็มประสิทธิภาพ (ไม่ต้อง heap fetch) ก็ต่อเมื่อ **หน้า heap block นั้นถูก mark ว่า "all visible"** ใน visibility map เท่านั้น ถ้าตารางถูก UPDATE/DELETE บ่อยและยังไม่ได้รับการ VACUUM ตามรอบ visibility map จะไม่ up-to-date ทำให้ PostgreSQL ยังต้องกลับไปตรวจสอบที่ heap อยู่ดี (`Heap Fetches` จะมีค่ามากกว่า 0)

```sql
-- รัน VACUUM เพื่ออัปเดต visibility map ให้เป็นปัจจุบัน
VACUUM orders;
```

> **แนวทางปฏิบัติ:** ตารางที่มี Index Only Scan เป็นหัวใจของ performance ควรตั้ง `autovacuum` ให้ทำงานถี่พอ (ปรับ `autovacuum_vacuum_scale_factor` ให้ต่ำลงสำหรับตารางนั้นโดยเฉพาะถ้าจำเป็น) เพื่อให้ visibility map ทันสมัยอยู่เสมอ

### สรุปแนวทางเลือกใช้ INCLUDE

ใช้ `INCLUDE` เมื่อ:
- คอลัมน์นั้นถูก `SELECT` บ่อย แต่ **ไม่เคยถูกใช้ใน WHERE, JOIN, หรือ ORDER BY** ของ query ที่ใช้ index นี้
- ต้องการลด heap fetch เพื่อทำ Index Only Scan โดยไม่อยากขยาย key columns ที่ซับซ้อนขึ้น

ใช้ key column ปกติ (ไม่ใช่ INCLUDE) เมื่อ:
- คอลัมน์นั้นถูกใช้ใน WHERE เป็นเงื่อนไข equality หรือ range
- คอลัมน์นั้นถูกใช้ใน ORDER BY และต้องการให้ index ให้ผลลัพธ์ตามลำดับนั้นโดยไม่ต้อง sort เพิ่ม

---

## Step 429: วิเคราะห์ Index ที่ไม่ถูกใช้งาน (Unused Index) ด้วย pg_stat_user_indexes

เมื่อสร้าง index มากขึ้นเรื่อย ๆ ระบบจะมีค่าใช้จ่ายแฝงตามมา: ทุก `INSERT`, `UPDATE`, `DELETE` ต้องอัปเดต index ทุกตัวที่เกี่ยวข้อง — ถ้ามี index ที่ไม่เคยถูกใช้ query เลย มันจะเป็นภาระล้วน ๆ โดยไม่มีประโยชน์ตอบแทน จึงต้องหมั่นตรวจสอบและลบทิ้งเป็นระยะ

### View pg_stat_user_indexes

PostgreSQL เก็บสถิติการใช้งาน index แต่ละตัวไว้ใน view `pg_stat_user_indexes`:

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
WHERE schemaname = 'public'
ORDER BY idx_scan ASC, pg_relation_size(indexrelid) DESC;
```

คอลัมน์สำคัญ:

| คอลัมน์ | ความหมาย |
|---|---|
| `idx_scan` | จำนวนครั้งที่ index นี้ถูกใช้ในการ scan (นับตั้งแต่ stat ถูก reset ล่าสุด) |
| `idx_tup_read` | จำนวน index entry ที่ถูกอ่านผ่าน index scan |
| `idx_tup_fetch` | จำนวนแถวที่ถูกดึงจาก heap หลังจากผ่าน index (สำหรับ Index Only Scan ค่านี้จะต่ำหรือเป็น 0) |

ผลลัพธ์ตัวอย่าง:

```
 schemaname | table_name |          index_name            | idx_scan | idx_tup_read | idx_tup_fetch | index_size
------------+------------+---------------------------------+----------+--------------+----------------+------------
 public     | orders     | idx_orders_status_customer      |        0 |            0 |              0 | 176 kB
 public     | products   | idx_products_name_lower         |        0 |            0 |              0 | 64 kB
 public     | orders     | idx_orders_status_pending       |       12 |         2856 |           2856 | 16 kB
 public     | customers  | idx_customers_email_lower       |       48 |           48 |             48 | 88 kB
 public     | orders     | idx_orders_customer_status_covering |    35 |          140 |              0 | 240 kB
```

### หา Index ที่ไม่เคยถูกใช้เลย (idx_scan = 0)

```sql
SELECT
    relname AS table_name,
    indexrelname AS index_name,
    pg_size_pretty(pg_relation_size(indexrelid)) AS wasted_size
FROM pg_stat_user_indexes
WHERE schemaname = 'public'
  AND idx_scan = 0
  AND indexrelname NOT LIKE '%_pkey'   -- ระวังไม่ตัด primary key ทิ้ง
ORDER BY pg_relation_size(indexrelid) DESC;
```

### ข้อควรระวังก่อนลบ index จาก idx_scan = 0

1. **สถิติถูก reset ได้** — เมื่อรัน `pg_stat_reset()`, database restart, หรือในบางกรณีหลัง `REINDEX` ค่า `idx_scan` จะกลับเป็น 0 ทันที ทำให้ตีความผิดว่า index ไม่ถูกใช้ทั้งที่จริง ๆ ใช้บ่อย ควรตรวจสอบ `stats_reset` เวลาที่นานพอ (เช่น อย่างน้อย 1 รอบ business cycle เต็ม — สัปดาห์ เดือน หรือไตรมาส แล้วแต่ลักษณะงาน) ก่อนสรุป
2. **ตรวจสอบเวลาที่สถิติเริ่มนับจาก `pg_stat_database`:**

```sql
SELECT stats_reset FROM pg_stat_database WHERE datname = current_database();
```

3. **Unique/Primary Key/Foreign Key constraint index** อาจมี `idx_scan = 0` แต่ยังจำเป็นต้องมีเพื่อ **บังคับ constraint** แม้จะไม่เคยถูกใช้ query โดยตรงเลยก็ตาม ห้ามลบ index เหล่านี้เพียงเพราะ `idx_scan = 0`
4. **ตรวจสอบ replica แยกต่างหาก** — สถิติของ primary และ replica (standby) แยกกันคนละชุด ถ้า query ส่วนใหญ่วิ่งไปที่ replica (read replica) การดู `idx_scan` แค่ที่ primary อาจทำให้เข้าใจผิด ต้องรวมสถิติจากทุก node ที่รับ query จริง

### หา Index ซ้ำซ้อน (Duplicate/Redundant Index)

นอกจาก index ที่ไม่ถูกใช้เลย อีกปัญหาที่พบบ่อยคือ index ที่ "ซ้ำซ้อนกันเอง" เช่น มีทั้ง `(customer_id)` และ `(customer_id, status)` — index ตัวแรกมักไม่จำเป็นเพราะ leftmost prefix ของตัวที่สองครอบคลุมการค้นหาด้วย `customer_id` เพียงอย่างเดียวได้อยู่แล้ว

```sql
SELECT
    indrelid::regclass AS table_name,
    array_agg(indexrelid::regclass) AS overlapping_indexes,
    array_agg(indkey::text) AS column_positions
FROM pg_index
WHERE indrelid = 'orders'::regclass
GROUP BY indrelid;
```

หรือใช้ query ที่เจาะจงกว่าเพื่อหา index ที่มี column ชุดแรกซ้ำกัน:

```sql
SELECT
    a.indexrelid::regclass AS index_a,
    b.indexrelid::regclass AS index_b
FROM pg_index a
JOIN pg_index b
  ON a.indrelid = b.indrelid
 AND a.indexrelid < b.indexrelid
 AND a.indkey[0] = b.indkey[0]           -- คอลัมน์แรกเหมือนกัน
WHERE a.indrelid = 'orders'::regclass;
```

### ลบ Index ที่ไม่จำเป็นอย่างปลอดภัย

```sql
-- สมมติยืนยันแล้วว่า idx_orders_status_customer ไม่เคยถูกใช้เลยตลอด 3 เดือนที่ผ่านมา
-- และไม่ใช่ constraint index
DROP INDEX CONCURRENTLY IF EXISTS idx_orders_status_customer;
```

การใช้ `CONCURRENTLY` ช่วยให้การลบ index ไม่ต้องล็อกตารางแบบ exclusive lock ซึ่งสำคัญมากบน production database ที่ต้องรองรับ traffic ตลอดเวลา (ข้อแลกเปลี่ยนคือใช้เวลานานกว่าปกติเล็กน้อย และไม่สามารถรันอยู่ภายใน transaction block ได้)

> **แนวทางปฏิบัติที่ดี:** ก่อนลบ index จริง ให้พิจารณา "renamed แล้วปิดใช้งานชั่วคราว" ด้วยการ disable ผ่าน monitoring ก่อน หรือ backup คำสั่ง `CREATE INDEX` ไว้เสมอ เพื่อให้สามารถสร้างกลับคืนได้ทันทีหากพบว่า query สำคัญบางตัว (ที่ไม่ได้รันบ่อยแต่สำคัญ เช่น รายงานสิ้นปี) ยังต้องพึ่ง index นั้นอยู่

---

## Step 430: แบบฝึกหัดรวม — ออกแบบชุด Index ที่เหมาะสมที่สุดสำหรับ Query Workload จริง

สมมติทีม data engineering รวบรวม query pattern ที่เกิดขึ้นบ่อยที่สุดในระบบ e-commerce ของเราจาก `pg_stat_statements` ได้ดังนี้ (เรียงตามความถี่):

1. **Login ด้วยอีเมล (case-insensitive)** — รันหลายพันครั้งต่อนาที
   ```sql
   SELECT customer_id, first_name, is_active FROM customers WHERE LOWER(email) = LOWER($1);
   ```
2. **Dashboard ทีมปฏิบัติการ: ดูออเดอร์ที่ยัง pending เรียงตามวันที่เก่าสุดก่อน** — รันทุก 10 วินาที
   ```sql
   SELECT order_id, customer_id, order_date FROM orders WHERE status = 'pending' ORDER BY order_date;
   ```
3. **ประวัติการสั่งซื้อของลูกค้าคนหนึ่ง (หน้า "คำสั่งซื้อของฉัน")** — รันบ่อยมาก
   ```sql
   SELECT order_id, order_date, status FROM orders WHERE customer_id = $1 ORDER BY order_date DESC;
   ```
4. **ค้นหาสินค้าที่ active อยู่ในหมวดหมู่หนึ่ง** — รันบ่อยมาก (หน้า category listing)
   ```sql
   SELECT product_id, product_name, unit_price FROM products
   WHERE category_id = $1 AND is_active = true
   ORDER BY unit_price;
   ```
5. **รายงานยอดขายรายวัน** — รันวันละ 1 ครั้ง (batch job กลางคืน)
   ```sql
   SELECT count(*), sum(oi.quantity * oi.unit_price)
   FROM orders o JOIN order_items oi ON oi.order_id = o.order_id
   WHERE o.order_date >= $1 AND o.order_date < $2 AND o.status = 'delivered';
   ```
6. **Unique constraint แบบมีเงื่อนไข** — email ต้องไม่ซ้ำเฉพาะบัญชี active (จาก Step 423)

### ขั้นตอนการออกแบบทีละ query

**Query 1 (Login):** ต้องการ exact match บน `LOWER(email)` และดึง `is_active` ด้วย → ใช้ **expression index** พร้อม **INCLUDE** เพื่อทำ Index Only Scan:

```sql
CREATE INDEX idx_customers_email_lower_covering
    ON customers (LOWER(email))
    INCLUDE (first_name, is_active);
```

**Query 2 (Dashboard pending):** ต้องการเฉพาะแถว `status = 'pending'` เรียงตาม `order_date` → ใช้ **partial index** ที่รวม `order_date` เป็น key column (เพื่อให้ index คืนผลลัพธ์แบบเรียงลำดับพร้อมใช้ ไม่ต้อง sort เพิ่ม):

```sql
CREATE INDEX idx_orders_pending_by_date
    ON orders (order_date)
    WHERE status = 'pending';
```

> สังเกตว่า index นี้แทนที่ `idx_orders_status_pending` เดิมได้ดีกว่า เพราะนอกจากกรองด้วย `status = 'pending'` แล้ว ยังให้ผลลัพธ์เรียงตาม `order_date` มาในตัว ไม่ต้อง sort เพิ่ม — ลบตัวเก่าทิ้งได้:
> ```sql
> DROP INDEX IF EXISTS idx_orders_status_pending;
> ```

**Query 3 (ประวัติลูกค้า):** ต้องการ `customer_id = X` แล้วเรียงตาม `order_date DESC` → **composite index** โดยใส่ `order_date` ต่อจาก `customer_id` และระบุทิศทางการเรียงให้ตรงกับ query:

```sql
CREATE INDEX idx_orders_customer_date_desc
    ON orders (customer_id, order_date DESC)
    INCLUDE (status);
```

**Query 4 (ค้นหาสินค้าตามหมวดหมู่ที่ active):** ต้องการ equality บน `category_id` และ `is_active = true` แล้วเรียงตาม `unit_price` — เนื่องจาก `is_active = true` เป็นเงื่อนไขคงที่เสมอ (ไม่มี query ไหนค้นหา `is_active = false` ในหน้าลูกค้า) จึงใช้ **partial + composite index** ร่วมกัน:

```sql
CREATE INDEX idx_products_active_category_price
    ON products (category_id, unit_price)
    WHERE is_active = true;
```

index นี้เล็กลง ~8% จากที่ตัดสินค้าที่เลิกขายออกไป (ตามสัดส่วนที่เราสุ่มไว้ตอนสร้างข้อมูล) และตรงกับ query pattern แบบเป๊ะ ๆ

**Query 5 (รายงานยอดขายรายวัน — batch job):** เป็น query ที่รันแค่วันละครั้งและ scan ข้อมูลจำนวนมากอยู่แล้ว (ทั้งช่วงวันที่) จึง **ไม่คุ้มที่จะสร้าง index พิเศษ** สำหรับ query นี้โดยเฉพาะ — ใช้ index ที่มีอยู่แล้วบน `order_date` (จาก full table scan แบบ range) และ `order_id` บน `order_items` (ซึ่งควรมี index อยู่แล้วจาก foreign key) ก็เพียงพอ:

```sql
-- ตรวจสอบว่ามี index รองรับ join แล้วหรือยัง
CREATE INDEX IF NOT EXISTS idx_order_items_order_id ON order_items (order_id);
CREATE INDEX IF NOT EXISTS idx_orders_order_date ON orders (order_date);
```

> **บทเรียน:** ไม่ใช่ทุก query ที่ต้องมี index เฉพาะทาง โดยเฉพาะ batch job ที่รันไม่บ่อยและต้องอ่านข้อมูลปริมาณมากอยู่แล้ว (planner มักเลือก Seq Scan + Hash Join ซึ่งเหมาะสมกว่า Index Scan อยู่แล้วในกรณีนี้) การสร้าง index เพิ่มโดยไม่จำเป็นมีแต่จะเพิ่มภาระ write โดยไม่ได้ประโยชน์คุ้มค่า

**Query 6 (partial unique):** ใช้ index จาก Step 423 ที่สร้างไว้แล้ว:

```sql
-- idx_customers_email_active_uniq ON customers (email) WHERE is_active = true
```

### ตรวจสอบผลลัพธ์สุดท้ายด้วย EXPLAIN ANALYZE ทุก query

```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT customer_id, first_name, is_active
FROM customers WHERE LOWER(email) = LOWER('customer1234@example.com');

EXPLAIN (ANALYZE, BUFFERS) SELECT order_id, customer_id, order_date
FROM orders WHERE status = 'pending' ORDER BY order_date;

EXPLAIN (ANALYZE, BUFFERS) SELECT order_id, order_date, status
FROM orders WHERE customer_id = 500 ORDER BY order_date DESC;

EXPLAIN (ANALYZE, BUFFERS) SELECT product_id, product_name, unit_price
FROM products WHERE category_id = 2 AND is_active = true ORDER BY unit_price;
```

ทุก query ควรได้ `Index Scan` หรือ `Index Only Scan` ที่ใช้ index ที่เราออกแบบไว้เฉพาะเจาะจง โดยไม่มี `Seq Scan` หรือ `Sort` node ที่ไม่จำเป็นปรากฏใน plan

### สรุปชุด index สุดท้ายสำหรับระบบนี้

```sql
-- customers
CREATE UNIQUE INDEX idx_customers_email_active_uniq ON customers (email) WHERE is_active = true;
CREATE INDEX idx_customers_email_lower_covering ON customers (LOWER(email)) INCLUDE (first_name, is_active);

-- orders
CREATE INDEX idx_orders_pending_by_date ON orders (order_date) WHERE status = 'pending';
CREATE INDEX idx_orders_customer_date_desc ON orders (customer_id, order_date DESC) INCLUDE (status);
CREATE INDEX idx_orders_order_date ON orders (order_date);
CREATE INDEX idx_orders_status_full ON orders (status);  -- รองรับ query ทั่วไปตาม status อื่นที่ไม่ใช่ pending

-- products
CREATE INDEX idx_products_active_category_price ON products (category_id, unit_price) WHERE is_active = true;

-- order_items
CREATE INDEX idx_order_items_order_id ON order_items (order_id);
```

ชุด index นี้ครอบคลุมทั้ง 4 เทคนิคที่เรียนในบทนี้: **partial index** (`idx_orders_pending_by_date`, `idx_products_active_category_price`), **partial unique index** (`idx_customers_email_active_uniq`), **expression index** (`idx_customers_email_lower_covering`), **composite index** (`idx_orders_customer_date_desc`, `idx_products_active_category_price`) และ **covering index ด้วย INCLUDE** (`idx_orders_customer_date_desc`, `idx_customers_email_lower_covering`)

---

## สรุปท้ายบท

ในบทนี้เราได้เรียนรู้เทคนิคการออกแบบ index ขั้นสูงของ PostgreSQL ที่ช่วยให้ index มีขนาดเล็กลง เร็วขึ้น และตรงกับ query pattern จริงมากขึ้น:

- **Partial Index** (Step 421-422): สร้าง index เฉพาะแถวที่ตรงตามเงื่อนไข `WHERE` ลดขนาด index ได้มากเมื่อข้อมูลกระจุกตัว (skewed) เช่น การเก็บเฉพาะออเดอร์ `pending` จากทั้งหมดที่ส่วนใหญ่ `delivered` แล้ว
- **Partial Unique Index** (Step 423): ใช้สร้าง conditional uniqueness เช่น "อีเมลไม่ซ้ำเฉพาะบัญชี active" ซึ่ง constraint ปกติทำไม่ได้
- **Expression Index / Functional Index** (Step 424-425): สร้าง index จากผลลัพธ์ของ function เช่น `LOWER(email)` เพื่อรองรับ case-insensitive search — ต้องระวังเรื่อง function ต้องเป็น **IMMUTABLE** โดยเฉพาะกรณี `DATE(timestamptz)` ที่ต้องบังคับ timezone ให้คงที่ก่อน
- **Composite (Multi-column) Index** (Step 426-427): ลำดับคอลัมน์สำคัญตาม **leftmost prefix rule** — วางคอลัมน์ equality ก่อน range, และวางตามลำดับที่ query pattern ใช้บ่อยที่สุด
- **Covering Index ด้วย INCLUDE** (Step 428): เพิ่มคอลัมน์เข้า index โดยไม่ใช้เป็น search key เพื่อให้เกิด **Index Only Scan** ลด heap fetch — ต้องพึ่งพา visibility map ที่ทันสมัยจาก VACUUM ด้วย
- **การวิเคราะห์ Unused Index** (Step 429): ใช้ `pg_stat_user_indexes` ตรวจสอบ `idx_scan` เพื่อหา index ที่ไม่เคยถูกใช้ แล้วลบทิ้งด้วย `DROP INDEX CONCURRENTLY` อย่างระมัดระวัง (เช็ค stats_reset, constraint index, และ replica ก่อนเสมอ)
- **การออกแบบชุด index แบบองค์รวม** (Step 430): ต้องมองจาก query workload จริงเป็นหลัก ไม่ใช่สร้าง index ตามทฤษฎีล้วน ๆ — บาง query (เช่น batch job รายวัน) อาจไม่ต้องมี index เฉพาะทางเลยด้วยซ้ำ

หลักคิดสำคัญที่สุดของบทนี้คือ **"index ที่ดีที่สุด ไม่ใช่ index ที่ครอบคลุมมากที่สุด แต่คือ index ที่ตรงกับ query pattern จริงมากที่สุด โดยมีต้นทุนด้าน storage และ write overhead น้อยที่สุด"** การใช้ partial, expression, composite และ covering index ร่วมกันอย่างมีชั้นเชิง คือทักษะที่แยก DBA มือใหม่ออกจาก DBA ระดับ world-class

---

## แบบฝึกหัด

<details>
<summary><strong>แบบฝึกหัดที่ 1:</strong> จงสร้าง partial index บนตาราง <code>orders</code> ที่ครอบคลุมเฉพาะออเดอร์ที่ <code>status = 'cancelled'</code> โดยรวมคอลัมน์ <code>order_date</code> ไว้ใน key ด้วย เพื่อให้ทีมบัญชีตรวจสอบยอดออเดอร์ที่ถูกยกเลิกในแต่ละช่วงเวลาได้เร็ว</summary>

```sql
CREATE INDEX idx_orders_cancelled_by_date
    ON orders (order_date)
    WHERE status = 'cancelled';
```

ทดสอบ:

```sql
EXPLAIN ANALYZE
SELECT order_id, order_date
FROM orders
WHERE status = 'cancelled'
  AND order_date >= '2026-01-01'
ORDER BY order_date;
```

Planner ควรเลือกใช้ `idx_orders_cancelled_by_date` เพราะเงื่อนไข `status = 'cancelled'` ตรงกับ predicate เป๊ะ และ `order_date` เป็นทั้ง filter เพิ่มเติมและใช้เรียงลำดับได้ในตัว
</details>

<details>
<summary><strong>แบบฝึกหัดที่ 2:</strong> ทำไมคำสั่งต่อไปนี้จึงไม่ error แม้จะดูเหมือนสร้าง index ซ้ำกัน?

```sql
CREATE INDEX idx_a ON orders (status) WHERE status = 'pending';
CREATE INDEX idx_b ON orders (status) WHERE status = 'shipped';
```
</summary>

ไม่ error เพราะ partial index ทั้งสองมี **predicate ต่างกัน** (`status = 'pending'` กับ `status = 'shipped'`) จึงครอบคลุมแถวคนละกลุ่มกัน ไม่ได้ซ้ำซ้อนกันจริง ๆ แม้จะ index บนคอลัมน์เดียวกันก็ตาม แต่ละ index จะถูกใช้เมื่อ query มีเงื่อนไขตรงกับ predicate ของมันเท่านั้น — `idx_a` ใช้ตอบ `WHERE status = 'pending'` และ `idx_b` ใช้ตอบ `WHERE status = 'shipped'` โดยไม่ก้าวก่ายกัน
</details>

<details>
<summary><strong>แบบฝึกหัดที่ 3:</strong> จงเขียน partial unique index เพื่อบังคับว่าสินค้าที่ <code>is_active = true</code> ต้องมี <code>product_name</code> ไม่ซ้ำกัน (สินค้าที่เลิกขายไปแล้วสามารถมีชื่อซ้ำกับสินค้าที่ active ได้)</summary>

```sql
CREATE UNIQUE INDEX idx_products_name_active_uniq
    ON products (product_name)
    WHERE is_active = true;
```

ทดสอบ:

```sql
-- ถ้ามีสินค้า active ชื่อ 'Product 1' อยู่แล้ว การ insert สินค้าใหม่ที่ active ชื่อเดียวกันจะ error
INSERT INTO products (product_name, category_id, supplier_id, unit_price, is_active)
VALUES ('Product 1', 1, 1, 100.00, true);
-- ERROR: duplicate key value violates unique constraint "idx_products_name_active_uniq"
```
</details>

<details>
<summary><strong>แบบฝึกหัดที่ 4:</strong> Query ต่อไปนี้จะใช้ index บนคอลัมน์ <code>email</code> ปกติ (ไม่ใช่ expression index) ได้หรือไม่ เพราะเหตุใด?

```sql
SELECT * FROM customers WHERE email || '' = 'customer1@example.com';
```
</summary>

**ไม่ได้** เพราะ `email || ''` เป็น expression ที่ครอบคอลัมน์ `email` ไว้ (string concatenation) ทำให้ค่าที่ต้องเปรียบเทียบกลายเป็นผลลัพธ์ของ expression ไม่ใช่ค่า `email` ดิบ ๆ index ปกติบน `email` จึงใช้ไม่ได้ ต้องสร้าง expression index บน `(email || '')` โดยตรง (แม้ในทางปฏิบัติ query แบบนี้ไม่มีประโยชน์อะไรและควรเขียนเป็น `WHERE email = 'customer1@example.com'` แทนเพื่อใช้ index ปกติได้เลย) — บทเรียนคือ: **การครอบคอลัมน์ด้วย expression ใด ๆ ก็ตาม (function, operator, cast) จะทำให้ index ปกติบนคอลัมน์นั้นใช้ไม่ได้ทันที เว้นแต่จะมี expression index ที่ตรงกับ expression นั้นเป๊ะ**
</details>

<details>
<summary><strong>แบบฝึกหัดที่ 5:</strong> ทำไมคำสั่งนี้จึง error และจะแก้ได้อย่างไร?

```sql
CREATE INDEX idx_orders_year ON orders (EXTRACT(YEAR FROM order_date));
```
</summary>

`EXTRACT(YEAR FROM order_date)` เมื่อ `order_date` เป็น `TIMESTAMPTZ` จะมีผลลัพธ์ขึ้นกับ session timezone เช่นเดียวกับปัญหา `DATE(timestamptz)` ใน Step 425 ทำให้ function นี้ถูก mark เป็น STABLE ไม่ใช่ IMMUTABLE และสร้าง index ไม่ได้ วิธีแก้คือบังคับ timezone ให้คงที่ก่อน:

```sql
CREATE INDEX idx_orders_year
    ON orders (EXTRACT(YEAR FROM (order_date AT TIME ZONE 'UTC')));
```
</details>

<details>
<summary><strong>แบบฝึกหัดที่ 6:</strong> มี composite index <code>ON products (supplier_id, category_id, unit_price)</code> จงบอกว่า query ต่อไปนี้แต่ละข้อใช้ index นี้ได้ "เต็มประสิทธิภาพ", "ใช้ได้บางส่วน", หรือ "ใช้ไม่ได้เลย"

1. `WHERE supplier_id = 5`
2. `WHERE category_id = 3`
3. `WHERE supplier_id = 5 AND category_id = 3`
4. `WHERE supplier_id = 5 AND unit_price > 100`
5. `WHERE supplier_id = 5 AND category_id = 3 AND unit_price > 100`
</summary>

1. **เต็มประสิทธิภาพ** — ตรงกับ leftmost column
2. **ใช้ไม่ได้เลย** (ในแง่ index condition โดยตรง) — ข้าม leftmost column (`supplier_id`) ไป planner จะเลือก Seq Scan หรือใช้ index อื่นแทนถ้ามี
3. **เต็มประสิทธิภาพ** — ตรงกับสองคอลัมน์แรกตามลำดับ (ทั้งคู่เป็น equality)
4. **ใช้ได้บางส่วน** — `supplier_id` ใช้เป็น index condition ได้เต็มที่ ส่วน `unit_price > 100` ใช้เป็น filter condition ได้เช่นกันแม้จะข้าม `category_id` ไป เพราะ `unit_price` เป็น key column ถัดจาก `category_id` — PostgreSQL จะ scan ทุก entry ที่ `supplier_id = 5` (ไม่ว่า category ใด) แล้วค่อย filter `unit_price > 100` เพิ่ม (ไม่ใช่ index range condition ที่ skip ข้าม category ได้ทันที)
5. **เต็มประสิทธิภาพ** — `supplier_id` และ `category_id` เป็น equality (ตรงกับ prefix) และ `unit_price > 100` เป็น range condition ที่ตามหลัง equality columns พอดี ตามหลัก "equality ก่อน range"
</details>

<details>
<summary><strong>แบบฝึกหัดที่ 7:</strong> จงสร้าง covering index ที่รองรับ query นี้ให้เป็น Index Only Scan:

```sql
SELECT product_id, unit_price, stock_quantity
FROM products
WHERE category_id = 5 AND is_active = true;
```
</summary>

```sql
CREATE INDEX idx_products_category_active_covering
    ON products (category_id)
    INCLUDE (unit_price, stock_quantity)
    WHERE is_active = true;
```

ที่นี้รวมทั้ง partial index (`WHERE is_active = true`) และ covering index (`INCLUDE`) เข้าด้วยกัน เพราะ `is_active` เป็นเงื่อนไข equality คงที่และ `unit_price`, `stock_quantity` ถูก select แต่ไม่ได้ใช้กรอง/เรียงลำดับ ทดสอบด้วย:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT product_id, unit_price, stock_quantity
FROM products
WHERE category_id = 5 AND is_active = true;
```

ควรเห็น `Index Only Scan` และ `Heap Fetches: 0`
</details>

<details>
<summary><strong>แบบฝึกหัดที่ 8:</strong> คำสั่งต่อไปนี้จะทำงานถูกต้องหรือไม่ และมันต่างจากการใส่ <code>order_date</code> เป็น key column ปกติอย่างไร?

```sql
CREATE INDEX idx_orders_customer_covering
    ON orders (customer_id)
    INCLUDE (order_date)
    WHERE status = 'pending'
    ORDER BY order_date;
```
</summary>

คำสั่งนี้ **error** เพราะ syntax ของ `CREATE INDEX` ไม่รองรับ `ORDER BY` clause ท้ายคำสั่ง (B-tree index เรียงลำดับตาม key column อยู่แล้วโดยอัตโนมัติ ไม่มี syntax ให้กำหนด sort order แยกต่างหากแบบนี้) syntax ที่ถูกต้องคือ:

```sql
CREATE INDEX idx_orders_customer_covering
    ON orders (customer_id, order_date)
    WHERE status = 'pending';
```

ความแตกต่างสำคัญระหว่างการใส่ `order_date` เป็น **key column** (ตามตัวอย่างที่แก้แล้ว) กับใส่เป็น **INCLUDE column**:
- ถ้าเป็น key column: index จะเรียงลำดับตาม `order_date` ภายในแต่ละ `customer_id` ด้วย ทำให้ query ที่มี `ORDER BY order_date` หรือ `WHERE order_date > ...` ร่วมกับ `customer_id = ...` ใช้ index ได้เต็มประสิทธิภาพ
- ถ้าเป็น INCLUDE column: `order_date` จะไม่ถูกเรียงลำดับหรือใช้เป็นเงื่อนไขค้นหาได้เลย เก็บไว้แค่เพื่อให้ query ที่ select คอลัมน์นี้ทำ Index Only Scan ได้เท่านั้น

เลือกใช้แบบไหนขึ้นกับว่า query จริงต้องการ filter/sort ด้วย `order_date` หรือแค่ต้องการอ่านค่าออกมาเฉย ๆ
</details>

<details>
<summary><strong>แบบฝึกหัดที่ 9:</strong> จากผลลัพธ์ <code>pg_stat_user_indexes</code> ต่อไปนี้ index ตัวไหนที่ควรพิจารณาลบทิ้ง และตัวไหนที่ต้องระวังแม้ <code>idx_scan = 0</code>

```
      index_name            | idx_scan | index_size | ลักษณะ
------------------------------+----------+------------+------------------------------
 idx_orders_ship_country      |        0 |    210 kB  | index ปกติ ไม่ใช่ constraint
 customers_pkey                |        0 |     88 kB  | primary key
 idx_customers_email_uniq_old  |        0 |     96 kB  | UNIQUE constraint index
 idx_products_discontinued     |        3 |    150 kB  | index ปกติ ใช้น้อยมาก
```
</summary>

- **`idx_orders_ship_country`** (idx_scan = 0, ไม่ใช่ constraint) — **ผู้สมัครที่ดีที่สุดสำหรับการลบ** เพราะไม่มีข้อผูกมัดอื่นใดนอกจากประสิทธิภาพ query แต่ยังต้องตรวจสอบ `stats_reset` ก่อนว่าผ่านมานานพอ (อย่างน้อยครอบคลุม business cycle เต็มรอบ) ก่อนตัดสินใจลบจริง
- **`customers_pkey`** — **ห้ามลบ** แม้ `idx_scan = 0` เพราะเป็น primary key ที่บังคับ uniqueness และมักถูกใช้เป็น target ของ foreign key จากตารางอื่น (`orders.customer_id REFERENCES customers`) การลบจะทำให้ referential integrity พัง
- **`idx_customers_email_uniq_old`** — **ต้องระวังมาก** แม้ `idx_scan = 0` เพราะเป็น index ที่รองรับ UNIQUE constraint หากลบจะทำให้ไม่มีการบังคับความไม่ซ้ำของอีเมลอีกต่อไป ต้องตรวจสอบก่อนว่ามี constraint อื่นมาแทนที่แล้วหรือยัง (เช่น partial unique index ใน Step 423) ก่อนจะลบตัวเดิมทิ้ง
- **`idx_products_discontinued`** — idx_scan = 3 ถือว่าต่ำมาก ควรตรวจสอบเพิ่มเติมว่า query ที่ใช้ index นี้สำคัญแค่ไหน (อาจเป็น query รายงานที่รันนาน ๆ ครั้งแต่สำคัญ เช่น รายงานประจำปี) ก่อนตัดสินใจ ไม่ควรด่วนลบเพียงเพราะตัวเลขน้อย
</details>

<details>
<summary><strong>แบบฝึกหัดที่ 10 (โจทย์รวม):</strong> ทีม frontend แจ้งว่ามี query ใหม่ที่ใช้บ่อยมากในหน้า "ค้นหาลูกค้าจากประเทศ พร้อมดูว่าสมัครสมาชิกช่วงไหน":

```sql
SELECT customer_id, first_name, last_name, signup_date
FROM customers
WHERE country = 'Thailand'
  AND is_active = true
  AND signup_date >= '2026-01-01'
ORDER BY signup_date DESC;
```

จงออกแบบ index ที่เหมาะสมที่สุดสำหรับ query นี้ พร้อมอธิบายเหตุผลการเลือกลำดับคอลัมน์และการใช้ partial/covering index</summary>

วิเคราะห์ query:
- `country = 'Thailand'` → equality condition
- `is_active = true` → equality condition คงที่ (query หน้านี้ค้นหาเฉพาะลูกค้า active เท่านั้นเสมอ) → เหมาะเป็น partial index predicate
- `signup_date >= '2026-01-01'` → range condition → ต้องอยู่หลัง equality columns ตามหลัก "equality ก่อน range"
- `ORDER BY signup_date DESC` → ควรให้ index จัดเรียงมาให้พร้อมเลย
- `SELECT` ต้องการ `first_name`, `last_name` เพิ่มเติม ซึ่งไม่ได้ใช้กรองหรือเรียงลำดับ → เหมาะเป็น INCLUDE column

Index ที่เหมาะสม:

```sql
CREATE INDEX idx_customers_country_active_signup
    ON customers (country, signup_date DESC)
    INCLUDE (first_name, last_name)
    WHERE is_active = true;
```

เหตุผลการออกแบบ:
1. **`country` เป็น key column แรก** เพราะเป็น equality condition ที่ selective พอสมควร (มีหลายประเทศ)
2. **`is_active = true` เป็น partial predicate ไม่ใช่ key column** เพราะเป็นเงื่อนไขคงที่เสมอในทุก query ของหน้านี้ — การทำเป็น partial index ช่วยลดขนาด index ลง (ตัดลูกค้าที่ inactive ออกไปทั้งหมด) และไม่ต้องเสีย key column slot ไปกับคอลัมน์ที่ค่าคงที่
3. **`signup_date DESC` เป็น key column ที่สอง** เพราะเป็นทั้ง range filter และ sort order ที่ query ต้องการ (`ORDER BY signup_date DESC`) — การระบุ `DESC` ใน index ทำให้ B-tree จัดเก็บในทิศทางที่ query ต้องการพอดี ไม่ต้อง sort เพิ่มใน plan
4. **`first_name`, `last_name` เป็น INCLUDE column** เพราะถูก select แต่ไม่เกี่ยวข้องกับการกรองหรือเรียงลำดับเลย การใส่เป็น INCLUDE ช่วยให้เกิด Index Only Scan โดยไม่ทำให้ B-tree key ซับซ้อนขึ้นโดยไม่จำเป็น

ทดสอบ:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT customer_id, first_name, last_name, signup_date
FROM customers
WHERE country = 'Thailand'
  AND is_active = true
  AND signup_date >= '2026-01-01'
ORDER BY signup_date DESC;
```

ผลลัพธ์ที่คาดหวัง: `Index Only Scan using idx_customers_country_active_signup` โดยไม่มี `Sort` node แยกต่างหาก และ `Heap Fetches: 0`
</details>

---

**บทถัดไป:** [Part 044 — Query Planner และการอ่าน EXPLAIN อย่างลึกซึ้ง](./part-044-query-planner-explain.md)
