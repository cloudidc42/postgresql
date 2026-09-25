# Part 060: VACUUM, AUTOVACUUM และการจัดการ Bloat — พร้อมโปรเจกต์ Analytics Dashboard

> **หลักสูตร PostgreSQL ฉบับสมบูรณ์** | ระดับสูง (Advanced) | Part 060 (โปรเจกต์ปิดท้ายระดับสูง)

บทนี้คือ **บทปิดท้ายของระดับ Advanced (Part 041-060)** เราจะเจาะลึกกลไกที่สำคัญที่สุดกลไกหนึ่งของ PostgreSQL ซึ่งเป็นผลพวงโดยตรงจาก MVCC ที่เรียนไปใน Part 058 นั่นคือ **VACUUM** และกระบวนการอัตโนมัติของมันคือ **AUTOVACUUM** จากนั้นจะปิดท้ายด้วยโปรเจกต์ใหญ่ที่รวบรวมทุกเทคนิคของระดับ Advanced เข้าด้วยกันเป็นระบบ **Analytics Dashboard** ที่ใช้งานได้จริง

---

## เป้าหมายการเรียนรู้

เมื่อจบบทนี้ ผู้เรียนจะสามารถ:

1. อธิบายได้ว่าทำไม PostgreSQL ถึงต้องมี VACUUM และมันเกี่ยวข้องกับ MVCC อย่างไร
2. ใช้คำสั่ง `VACUUM` และ `VACUUM FULL` ได้อย่างถูกต้อง พร้อมเข้าใจความแตกต่างเรื่อง lock และการคืนพื้นที่
3. ตรวจจับและวัดปริมาณ Table Bloat ด้วย `pgstattuple` และ query ประเมินค่า
4. เข้าใจกลไกการทำงานของ Autovacuum และพารามิเตอร์หลักที่ควบคุมมัน
5. ปรับแต่ง Autovacuum เฉพาะตาราง (per-table) สำหรับตารางที่มีการ update/delete ถี่มาก
6. เข้าใจบทบาทของ `ANALYZE` ต่อ query planner และรู้ว่าสถิติที่ล้าสมัยทำให้เกิดปัญหาอย่างไร
7. เข้าใจปัญหา Transaction ID Wraparound และวิธีป้องกันด้วย freezing
8. ตรวจสอบ (monitor) สถานะ VACUUM ผ่าน `pg_stat_user_tables`, `pg_stat_progress_vacuum` และ log
9. สรุป Best Practices ในการดูแลรักษาฐานข้อมูลระยะยาวด้วย VACUUM/Autovacuum
10. ออกแบบและสร้างระบบ **Analytics Dashboard** แบบเต็มรูปแบบ ที่รวม index, JSONB, full text search, partitioning-ready design, PL/pgSQL function, trigger สำหรับ audit และแผนการ vacuum/monitoring เข้าด้วยกัน

---

## เตรียมข้อมูล

เราจะใช้สคีมา e-commerce เดิมที่คุ้นเคยจากบทก่อนหน้า แต่คราวนี้จะ**สร้างข้อมูลจำนวนมาก** (หลักพันถึงหลักหมื่นแถว) ด้วย `generate_series` เพื่อให้เห็นผลของ bloat และ VACUUM ได้จริง

### 1. สร้างตาราง

```sql
-- ลบของเก่าถ้ามี เพื่อเริ่มต้นสะอาด
DROP TABLE IF EXISTS reviews, order_items, orders, customers, products, suppliers, categories CASCADE;

CREATE TABLE categories (
    category_id   SERIAL PRIMARY KEY,
    category_name VARCHAR(100) NOT NULL
);

CREATE TABLE suppliers (
    supplier_id   SERIAL PRIMARY KEY,
    supplier_name VARCHAR(150) NOT NULL,
    country       VARCHAR(60)
);

CREATE TABLE products (
    product_id     SERIAL PRIMARY KEY,
    product_name   VARCHAR(150) NOT NULL,
    category_id    INTEGER REFERENCES categories(category_id),
    supplier_id    INTEGER REFERENCES suppliers(supplier_id),
    unit_price     NUMERIC(10,2) NOT NULL,
    stock_quantity INTEGER NOT NULL DEFAULT 0,
    attributes     JSONB
);

CREATE TABLE customers (
    customer_id SERIAL PRIMARY KEY,
    first_name  VARCHAR(60) NOT NULL,
    last_name   VARCHAR(60) NOT NULL,
    email       VARCHAR(150) UNIQUE,
    country     VARCHAR(60)
);

CREATE TABLE orders (
    order_id    SERIAL PRIMARY KEY,
    customer_id INTEGER REFERENCES customers(customer_id),
    order_date  TIMESTAMPTZ NOT NULL DEFAULT now(),
    status      VARCHAR(20) NOT NULL DEFAULT 'pending'
);

CREATE TABLE order_items (
    order_item_id SERIAL PRIMARY KEY,
    order_id      INTEGER REFERENCES orders(order_id),
    product_id    INTEGER REFERENCES products(product_id),
    quantity      INTEGER NOT NULL,
    unit_price    NUMERIC(10,2) NOT NULL
);

CREATE TABLE reviews (
    review_id   SERIAL PRIMARY KEY,
    product_id  INTEGER REFERENCES products(product_id),
    rating      INTEGER CHECK (rating BETWEEN 1 AND 5),
    review_text TEXT
);
```

### 2. Bulk-load ข้อมูลด้วย generate_series

```sql
-- 20 หมวดหมู่สินค้า
INSERT INTO categories (category_name)
SELECT 'Category ' || g
FROM generate_series(1, 20) AS g;

-- 50 ซัพพลายเออร์
INSERT INTO suppliers (supplier_name, country)
SELECT
    'Supplier ' || g,
    (ARRAY['Thailand','China','Japan','Vietnam','USA','Germany','Korea'])[1 + floor(random() * 7)::int]
FROM generate_series(1, 50) AS g;

-- 5,000 สินค้า พร้อม JSONB attributes ที่หลากหลาย
INSERT INTO products (product_name, category_id, supplier_id, unit_price, stock_quantity, attributes)
SELECT
    'Product ' || g,
    1 + floor(random() * 20)::int,
    1 + floor(random() * 50)::int,
    round((random() * 5000 + 10)::numeric, 2),
    floor(random() * 500)::int,
    jsonb_build_object(
        'color', (ARRAY['red','blue','green','black','white','yellow'])[1 + floor(random() * 6)::int],
        'weight_kg', round((random() * 20)::numeric, 2),
        'warranty_months', (ARRAY[6,12,24,36])[1 + floor(random() * 4)::int],
        'tags', to_jsonb(ARRAY[
            (ARRAY['sale','new','bestseller','clearance','premium'])[1 + floor(random() * 5)::int],
            (ARRAY['eco','imported','local','limited'])[1 + floor(random() * 4)::int]
        ])
    )
FROM generate_series(1, 5000) AS g;

-- 2,000 ลูกค้า
INSERT INTO customers (first_name, last_name, email, country)
SELECT
    'FirstName' || g,
    'LastName' || g,
    'customer' || g || '@example.com',
    (ARRAY['Thailand','Singapore','Malaysia','Vietnam','Indonesia','Philippines'])[1 + floor(random() * 6)::int]
FROM generate_series(1, 2000) AS g;

-- 20,000 ออเดอร์ กระจายในช่วง 2 ปีที่ผ่านมา
INSERT INTO orders (customer_id, order_date, status)
SELECT
    1 + floor(random() * 2000)::int,
    now() - (random() * interval '730 days'),
    (ARRAY['pending','paid','shipped','delivered','cancelled'])[1 + floor(random() * 5)::int]
FROM generate_series(1, 20000) AS g;

-- 55,000 รายการสินค้าในออเดอร์ (1-3 รายการต่อออเดอร์โดยเฉลี่ย)
INSERT INTO order_items (order_id, product_id, quantity, unit_price)
SELECT
    1 + floor(random() * 20000)::int,
    p.product_id,
    1 + floor(random() * 5)::int,
    p.unit_price
FROM generate_series(1, 55000) AS g
JOIN LATERAL (
    SELECT product_id, unit_price FROM products
    OFFSET floor(random() * 5000)::int LIMIT 1
) p ON true;

-- 10,000 รีวิว พร้อมข้อความสำหรับทดสอบ Full Text Search
INSERT INTO reviews (product_id, rating, review_text)
SELECT
    1 + floor(random() * 5000)::int,
    1 + floor(random() * 5)::int,
    (ARRAY[
        'สินค้าคุณภาพดีมาก จัดส่งรวดเร็ว บรรจุภัณฑ์แน่นหนา',
        'ราคาแพงไปหน่อยแต่คุณภาพคุ้มค่า จะกลับมาซื้อซ้ำแน่นอน',
        'สินค้าไม่ตรงตามที่โฆษณา ผิดหวังมาก ต้องการคืนเงิน',
        'ใช้งานได้ดีตามที่คาดหวัง วัสดุแข็งแรงทนทาน เหมาะกับราคา',
        'จัดส่งช้ามาก รอเป็นสัปดาห์ แต่สินค้าโอเคใช้ได้',
        'ประทับใจในบริการหลังการขาย ทีมงานตอบเร็วและสุภาพ',
        'คุณภาพต่ำกว่าที่คิด สีไม่ตรงตามรูป ไม่แนะนำ',
        'สุดยอดมาก คุ้มค่าเงินทุกบาท จะบอกต่อเพื่อนๆ แน่นอน'
    ])[1 + floor(random() * 8)::int]
FROM generate_series(1, 10000) AS g;
```

### 3. ตรวจสอบขนาดข้อมูลเริ่มต้น

```sql
SELECT relname AS table_name,
       n_live_tup,
       pg_size_pretty(pg_total_relation_size(relid)) AS total_size
FROM pg_stat_user_tables
WHERE schemaname = 'public'
ORDER BY n_live_tup DESC;
```

```
    table_name  | n_live_tup | total_size
----------------+------------+------------
 order_items    |      55000 | 6608 kB
 reviews        |      10000 | 1832 kB
 orders         |      20000 | 2416 kB
 products       |       5000 | 1200 kB
 customers      |       2000 | 344 kB
 suppliers      |         50 | 40 kB
 categories     |         20 | 24 kB
```

> **หมายเหตุ**: ตัวเลขขนาดจริงจะแตกต่างกันไปตามเวอร์ชัน PostgreSQL, การตั้งค่า `fillfactor` และสถาปัตยกรรมเครื่อง แต่แนวโน้ม (เช่น `order_items` ใหญ่ที่สุด) จะสอดคล้องกันเสมอ

### 4. สร้าง Churn (UPDATE/DELETE จำนวนมาก) เพื่อจำลอง Bloat จริง

ในระบบจริง ตารางอย่าง `orders` (สถานะเปลี่ยนบ่อย) และ `products` (สต็อกเปลี่ยนทุกครั้งที่มีการขาย) จะถูก UPDATE ซ้ำๆ ตลอดเวลา เราจะจำลองสถานการณ์นี้:

```sql
-- ปิด autovacuum ชั่วคราวสำหรับ products เพื่อให้เห็น bloat ชัดเจนในบทถัดไป
-- (จะเปิดกลับคืนใน Step 595)
ALTER TABLE products SET (autovacuum_enabled = false);

-- จำลองการอัปเดตสต็อกสินค้าซ้ำๆ หลายรอบ (เช่น ระบบขายของทุกวินาที)
DO $$
BEGIN
    FOR i IN 1..20 LOOP
        UPDATE products
        SET stock_quantity = stock_quantity - 1
        WHERE stock_quantity > 0
          AND product_id % 10 = (i % 10);
    END LOOP;
END $$;

-- จำลองการเปลี่ยนสถานะออเดอร์ (pending -> paid -> shipped -> delivered)
UPDATE orders SET status = 'paid'      WHERE status = 'pending'  AND random() < 0.6;
UPDATE orders SET status = 'shipped'   WHERE status = 'paid'     AND random() < 0.5;
UPDATE orders SET status = 'delivered' WHERE status = 'shipped'  AND random() < 0.5;

-- ลบออเดอร์ที่ถูกยกเลิกและ order_items ที่เกี่ยวข้อง (สร้าง dead tuple จำนวนมาก)
DELETE FROM order_items
WHERE order_id IN (SELECT order_id FROM orders WHERE status = 'cancelled');

DELETE FROM orders WHERE status = 'cancelled' AND random() < 0.3;
```

หลังจากรันคำสั่งข้างต้น ตาราง `products`, `orders`, `order_items` จะมี **dead tuple** สะสมอยู่จำนวนมาก เราจะใช้ข้อมูลชุดนี้เพื่อสาธิตทุก Step ในบทนี้

```sql
SELECT relname, n_live_tup, n_dead_tup,
       round(100.0 * n_dead_tup / GREATEST(n_live_tup + n_dead_tup, 1), 2) AS dead_pct
FROM pg_stat_user_tables
WHERE relname IN ('products','orders','order_items')
ORDER BY dead_pct DESC;
```

```
   relname   | n_live_tup | n_dead_tup | dead_pct
-------------+------------+------------+----------
 products    |       5000 |       9000 |    64.29
 orders      |      20000 |       6000 |    23.08
 order_items |      54200 |        800 |     1.46
```

> ตัวเลขจริงจะขึ้นกับค่า random ในเครื่องของคุณ แต่ประเด็นสำคัญคือ **`products` มี dead tuple มากกว่า live tuple เสียอีก** เพราะเราปิด autovacuum ไว้ นี่คือจุดเริ่มต้นของปัญหา bloat ที่เราจะแก้ไขในบทนี้

---

## Step 591: ทำไมต้องมี VACUUM — เชื่อมโยงกับ MVCC

ใน **Part 058** เราเรียนรู้ว่า PostgreSQL ใช้ **MVCC (Multi-Version Concurrency Control)** เพื่อให้หลาย transaction อ่าน-เขียนพร้อมกันได้โดยไม่ต้อง lock กันเอง หัวใจของ MVCC คือ: **ทุกแถว (tuple) มี `xmin` และ `xmax`** กำกับว่าถูกสร้างโดย transaction ไหน และถูก "ปิดการมองเห็น" โดย transaction ไหน

เมื่อคุณรัน `UPDATE`:

```sql
UPDATE products SET stock_quantity = stock_quantity - 1 WHERE product_id = 1;
```

PostgreSQL **ไม่ได้แก้ไขแถวเดิมในที่เดิม (in-place)** แต่จะ:

1. สร้างแถวใหม่ (new tuple version) พร้อมข้อมูลที่อัปเดตแล้ว
2. ตั้งค่า `xmax` ของแถวเก่าให้เท่ากับ transaction ID ปัจจุบัน (ทำให้แถวเก่า "ตาย" สำหรับ transaction ในอนาคต)
3. แถวเก่ายังคงอยู่ในไฟล์ตารางจนกว่าจะไม่มี transaction ใดมองเห็นมันได้อีกต่อไป

ในทำนองเดียวกัน `DELETE` ก็แค่ตั้งค่า `xmax` โดยไม่ได้ลบข้อมูลออกจากดิสก์ทันที

```sql
-- ดู tuple version จริงด้วย system column
SELECT product_id, xmin, xmax, ctid
FROM products
WHERE product_id = 1;
```

```
 product_id |  xmin   | xmax | ctid
------------+---------+------+---------
          1 | 5182341 |    0 | (312,7)
```

### แล้วแถวเก่าหายไปไหน?

แถวเก่าที่ไม่มีใครมองเห็นอีกแล้วเรียกว่า **dead tuple** — มันยังคง "กินพื้นที่" อยู่ในไฟล์ heap ของตาราง (และ index ที่เกี่ยวข้อง) เพราะ PostgreSQL **ไม่ลบมันออกทันที** ด้วยเหตุผลด้านประสิทธิภาพ (การลบทันทีจะทำให้ทุก UPDATE ช้าลงมาก และเสี่ยงต่อการชนกับ transaction อื่นที่ยังอ่าน snapshot เก่าอยู่)

นี่คือเหตุผลที่ PostgreSQL ต้องมีกระบวนการแยกต่างหากที่เรียกว่า **VACUUM** เพื่อ:

1. **สแกนหา dead tuple** ที่ไม่มี transaction ใดมองเห็นได้อีกแล้ว (ต่ำกว่า oldest running transaction's snapshot)
2. **ทำเครื่องหมายพื้นที่นั้นว่าใช้ซ้ำได้** ใน Free Space Map (FSM) เพื่อให้ INSERT/UPDATE ครั้งต่อไปนำพื้นที่กลับมาใช้
3. **อัปเดต visibility map** เพื่อให้ index-only scan ทำงานได้เร็วขึ้น
4. **ป้องกัน Transaction ID Wraparound** ด้วยการ freeze tuple เก่า (จะอธิบายละเอียดใน Step 597)

```sql
-- ดูจำนวน dead tuple สะสมที่ยังไม่ถูกเก็บกวาด
SELECT relname, n_dead_tup, last_autovacuum, last_vacuum
FROM pg_stat_user_tables
WHERE relname = 'products';
```

```
 relname  | n_dead_tup | last_autovacuum | last_vacuum
----------+------------+------------------+-------------
 products |       9000 | (null)           | (null)
```

เนื่องจากเราปิด `autovacuum_enabled` ไว้ที่ตาราง `products` ตั้งแต่ตอนเตรียมข้อมูล จะเห็นว่า `last_autovacuum` และ `last_vacuum` เป็น `NULL` — dead tuple 9,000 แถวยังคงค้างอยู่ในไฟล์ตารางโดยไม่มีใครมาเก็บกวาด นี่คือจุดเริ่มต้นของ **Table Bloat** ที่เราจะพูดถึงใน Step 593

**สรุป**: VACUUM ไม่ใช่ฟีเจอร์เสริม แต่เป็น**กลไกที่จำเป็นต่อการทำงานของ MVCC** — ถ้าไม่มี VACUUM ฐานข้อมูลจะโตขึ้นเรื่อยๆ โดยไม่มีที่สิ้นสุด แม้ข้อมูลจริงจะไม่ได้เพิ่มขึ้นเลยก็ตาม

---

## Step 592: VACUUM พื้นฐาน — ความแตกต่างระหว่าง VACUUM และ VACUUM FULL

### VACUUM (ธรรมดา)

```sql
VACUUM products;
```

หรือแบบละเอียด:

```sql
VACUUM (VERBOSE) products;
```

```
INFO:  vacuuming "public.products"
INFO:  finished vacuuming "public.products": index scans: 1
pages: 0 removed, 152 remain, 0 skipped due to pins, 0 skipped frozen
tuples: 9000 removed, 5000 remain, 0 are dead but not yet removable
removable cutoff: 5182401, which was 12 XIDs old when operation ended
index scan needed: 152 pages from table (100.00% of total) had 9000 dead item identifiers removed
index "products_pkey": pages: 41 in total, 0 newly deleted, 0 currently deleted, 0 reusable
avg read rate: 12.481 MB/s, avg write rate: 8.320 MB/s
buffer usage: 350 hits, 45 misses, 30 dirtied
WAL usage: 197 records, 30 full page images, 245678 bytes
system usage: CPU: user: 0.02 s, system: 0.00 s, elapsed: 0.03 s
VACUUM
```

จุดสำคัญของ `VACUUM` แบบธรรมดา:

- **สแกนตารางและลบ dead tuple ออกจากหน้า (page)** แต่ **ไม่คืนพื้นที่ให้ระบบปฏิบัติการ (OS)** — พื้นที่ที่ว่างจะถูกทำเครื่องหมายไว้ใน FSM เพื่อให้ PostgreSQL นำกลับมาใช้กับ INSERT/UPDATE ครั้งต่อไปเท่านั้น
- ขนาดไฟล์ตารางบนดิสก์ **จะไม่เล็กลง** (ยกเว้นกรณีพิเศษที่หน้าว่างอยู่ท้ายไฟล์พอดี PostgreSQL จะ truncate ไฟล์ได้)
- ใช้ lock ระดับ `ShareUpdateExclusiveLock` เท่านั้น — **อนุญาตให้ SELECT, INSERT, UPDATE, DELETE ทำงานพร้อมกันได้ตามปกติ** (แต่จะบล็อกคำสั่งที่ต้องการ schema lock เช่น `ALTER TABLE`, และบล็อก VACUUM อีกตัวที่รันพร้อมกันบนตารางเดียวกัน)
- เหมาะสำหรับรันเป็นประจำ (ซึ่งปกติ autovacuum จะทำให้อัตโนมัติอยู่แล้ว)

```sql
-- ตรวจสอบขนาดตารางก่อน-หลัง VACUUM ธรรมดา
SELECT pg_size_pretty(pg_relation_size('products')) AS before_vacuum;
VACUUM products;
SELECT pg_size_pretty(pg_relation_size('products')) AS after_vacuum;
```

```
 before_vacuum
---------------
 1360 kB

 after_vacuum
--------------
 1360 kB
```

จะเห็นว่า**ขนาดไฟล์เท่าเดิม** แม้ dead tuple 9,000 แถวจะถูกลบออกจากหน้าแล้ว — เพราะพื้นที่ถูกเก็บไว้ใช้ซ้ำ ไม่ได้คืนกลับ OS

### VACUUM FULL

```sql
VACUUM FULL products;
```

`VACUUM FULL` ทำงานต่างออกไปโดยสิ้นเชิง:

- **เขียนตารางใหม่ทั้งหมด** ลงไฟล์ใหม่ (คล้าย `CREATE TABLE AS SELECT` ภายใน) โดยคัดลอกเฉพาะ live tuple แล้วสลับไฟล์เก่ากับไฟล์ใหม่
- **คืนพื้นที่ดิสก์ให้ OS จริง** — ขนาดไฟล์จะเล็กลงตามจริง
- ต้องใช้ lock ระดับ **`AccessExclusiveLock`** ตลอดกระบวนการ — **บล็อกทุกการอ่าน-เขียนบนตารางนั้น** จนกว่าจะเสร็จ
- ต้องการพื้นที่ดิสก์ชั่วคราวเพิ่มขึ้นถึง**เกือบ 2 เท่า**ของขนาดตารางเดิม (เพราะต้องเก็บทั้งไฟล์เก่าและไฟล์ใหม่พร้อมกันระหว่างดำเนินการ)
- Index ทั้งหมดจะถูกสร้างใหม่ (rebuild) ด้วย

```sql
SELECT pg_size_pretty(pg_relation_size('products')) AS before_full;
VACUUM FULL products;
SELECT pg_size_pretty(pg_relation_size('products')) AS after_full;
```

```
 before_full
-------------
 1360 kB

 after_full
------------
 424 kB
```

คราวนี้ขนาดตารางเล็กลงจริง (จาก 1360 kB เหลือ 424 kB) เพราะ dead tuple ทั้งหมดถูกกำจัดและพื้นที่คืนสู่ OS

### เมื่อไหร่ควรใช้อันไหน?

| สถานการณ์ | คำแนะนำ |
|---|---|
| ต้องการเก็บกวาด dead tuple เป็นประจำ | `VACUUM` ธรรมดา (หรือปล่อยให้ autovacuum ทำ) |
| ตารางมี bloat มหาศาล (เช่น เคยมีข้อมูล 10 GB แต่ DELETE ไปเหลือ 100 MB) และต้องการพื้นที่ดิสก์คืน | `VACUUM FULL` — **แต่ต้องทำใน maintenance window** เท่านั้น |
| ต้องการหลีกเลี่ยง downtime แต่ยังอยากคืนพื้นที่ | พิจารณาใช้ **`pg_repack`** extension ซึ่งทำงานคล้าย VACUUM FULL แต่ใช้ lock สั้นกว่ามาก (rebuild ตารางใหม่แบบ online แล้วสลับในขั้นตอนสุดท้ายเท่านั้น) |
| ระบบ production ที่มี traffic ตลอดเวลา | **หลีกเลี่ยง `VACUUM FULL`** โดยเด็ดขาดในเวลาทำการ — ให้ปรับ autovacuum ให้ทำงานถี่พอที่จะไม่ต้องพึ่ง VACUUM FULL เลย |

> **คำเตือนสำคัญ**: `VACUUM FULL` บนตารางขนาดใหญ่ในระบบ production ที่ใช้งานจริง อาจทำให้แอปพลิเคชัน **หยุดทำงานหลายนาทีถึงหลายชั่วโมง** เพราะ lock แบบ Exclusive จะทำให้ query ทุกตัวที่แตะตารางนั้นต้องรอคิว ควรใช้เมื่อจำเป็นจริงๆ เท่านั้น และทำใน maintenance window

---

## Step 593: Table Bloat คืออะไร — การสังเกตและวัดผล

### Bloat คืออะไร

**Table Bloat** คือสถานการณ์ที่ตารางหรือ index มีขนาดไฟล์บนดิสก์ **ใหญ่กว่าที่ควรจะเป็นมาก** เมื่อเทียบกับปริมาณข้อมูลจริง (live rows) เพราะมี dead tuple และพื้นที่ว่างสะสมอยู่มากเกินไป โดยไม่ถูกเก็บกวาดหรือนำกลับมาใช้ซ้ำทันเวลา

**สาเหตุหลักของ Bloat**:

1. Autovacuum ทำงานไม่ทันกับอัตราการ UPDATE/DELETE (เช่น ตารางถูกตั้งค่าปิด autovacuum อย่างที่เราทำกับ `products`)
2. มี long-running transaction ค้างอยู่ ทำให้ VACUUM ไม่สามารถลบ dead tuple ที่ transaction นั้นยังอาจมองเห็นได้ (จะอธิบายเพิ่มใน Step 599)
3. อัตรา UPDATE/DELETE สูงมากจนแซงหน้าความเร็วของ autovacuum (เช่น ตาราง queue หรือ session table ที่หมุนเวียนข้อมูลตลอดเวลา)
4. Replication slot ที่ตายหรือ idle เก็บ WAL และ snapshot เก่าไว้ ทำให้ VACUUM ไม่กล้าลบ tuple

### อาการที่สังเกตได้

- ขนาดตาราง (`pg_total_relation_size`) ใหญ่ผิดปกติเมื่อเทียบกับจำนวนแถว
- Sequential scan หรือ Index scan ช้าลงเรื่อยๆ ทั้งที่จำนวนแถวจริงไม่ได้เพิ่ม
- Disk usage โตเร็วกว่าที่ควร
- `n_dead_tup` ใน `pg_stat_user_tables` สูงเมื่อเทียบกับ `n_live_tup`

### วัด Bloat ด้วย pgstattuple (แม่นยำที่สุด)

```sql
CREATE EXTENSION IF NOT EXISTS pgstattuple;

SELECT *
FROM pgstattuple('products');
```

```
 table_len | tuple_count | tuple_len | tuple_percent | dead_tuple_count | dead_tuple_len | dead_tuple_percent | free_space | free_percent
-----------+-------------+-----------+----------------+-------------------+-----------------+---------------------+------------+---------------
   1392640 |        5000 |    380000 |          27.29 |              9000 |          684000 |               49.11 |      98640 |          7.08
```

การตีความ:

- `tuple_percent` = 27.29% — พื้นที่ที่ใช้เก็บข้อมูลจริง (live) มีแค่ประมาณ 1 ใน 4 ของไฟล์ทั้งหมด
- `dead_tuple_percent` = 49.11% — เกือบครึ่งของไฟล์เป็น dead tuple ที่รอการเก็บกวาด!
- `free_percent` = 7.08% — พื้นที่ว่างที่ยังไม่ได้ใช้

`pgstattuple` ให้ผลลัพธ์**แม่นยำที่สุด** เพราะมันสแกนตารางจริง แต่ข้อเสียคือ**ใช้เวลานานและกิน I/O มาก**สำหรับตารางขนาดใหญ่ (คล้าย full table scan) จึงไม่เหมาะกับการรันบ่อยๆ บนตารางใหญ่ในเวลาทำการ

มีฟังก์ชันแบบเบา (approximate) ให้ใช้ด้วย:

```sql
SELECT * FROM pgstattuple_approx('products');
```

ซึ่งใช้ visibility map ประเมินผลแบบเร็วกว่ามาก โดยแลกกับความแม่นยำที่ลดลงเล็กน้อย

### วัด Bloat แบบประเมิน (ไม่ต้องติดตั้ง extension)

เมื่อไม่สามารถติดตั้ง extension ได้ (เช่น managed database บางเจ้าที่จำกัดสิทธิ์) เราสามารถประเมินคร่าวๆ จาก `pg_stat_user_tables` ได้:

```sql
SELECT
    relname,
    n_live_tup,
    n_dead_tup,
    round(100.0 * n_dead_tup / GREATEST(n_live_tup + n_dead_tup, 1), 2) AS dead_pct,
    pg_size_pretty(pg_total_relation_size(relid)) AS total_size
FROM pg_stat_user_tables
WHERE schemaname = 'public'
ORDER BY n_dead_tup DESC;
```

```
   relname   | n_live_tup | n_dead_tup | dead_pct | total_size
-------------+------------+------------+----------+------------
 products    |       5000 |       9000 |    64.29 | 1360 kB
 orders      |      20000 |       6000 |    23.08 | 3128 kB
 order_items |      54200 |        800 |     1.46 | 6608 kB
```

> **ข้อควรระวัง**: `n_live_tup`/`n_dead_tup` เป็นค่าที่ PostgreSQL ประเมินจากการนับตอน VACUUM/ANALYZE ครั้งล่าสุด **ไม่ใช่ค่าที่นับสดใหม่ทุกครั้ง** จึงอาจคลาดเคลื่อนได้หากไม่มีการ vacuum/analyze มานาน วิธีนี้เหมาะกับการ**ตรวจสอบแบบคร่าวๆ เพื่อหาตารางที่น่าสงสัย** ก่อนไปตรวจสอบละเอียดด้วย `pgstattuple`

**กฎง่ายๆ ที่ใช้ได้จริง**: ถ้า `dead_pct` เกิน **~20%** อย่างต่อเนื่อง ควรตรวจสอบว่า autovacuum ทำงานทันหรือไม่ (Step 594-595)

---

## Step 594: Autovacuum — การทำงานอัตโนมัติเบื้องหลัง

`autovacuum` คือ background process ที่ PostgreSQL รันอัตโนมัติเพื่อ vacuum และ analyze ตารางให้เราโดยไม่ต้องสั่งเอง เปิดใช้งานเป็นค่าเริ่มต้น (default: `on`) และ **ควรเปิดไว้เสมอในระบบ production**

### สถาปัตยกรรมคร่าวๆ

- **Autovacuum Launcher**: process หลักที่คอยปลุก worker ตามรอบเวลา (`autovacuum_naptime`, ค่าเริ่มต้น 1 นาที)
- **Autovacuum Workers**: process ย่อยที่รันจริง (จำนวนพร้อมกันสูงสุดกำหนดด้วย `autovacuum_max_workers`, ค่าเริ่มต้น 3)
- แต่ละ worker จะสแกนทุกตารางในฐานข้อมูล และตัดสินใจว่าตารางไหน "ถึงเกณฑ์" ที่ต้อง vacuum หรือ analyze

### สูตรคำนวณว่าเมื่อไหร่ต้อง Vacuum

```
vacuum threshold = autovacuum_vacuum_threshold + autovacuum_vacuum_scale_factor * pg_class.reltuples
```

ค่าเริ่มต้น:

| พารามิเตอร์ | ค่า default | ความหมาย |
|---|---|---|
| `autovacuum_vacuum_threshold` | 50 | จำนวน dead tuple ขั้นต่ำก่อนพิจารณา |
| `autovacuum_vacuum_scale_factor` | 0.2 (20%) | สัดส่วนของจำนวนแถวทั้งหมด |
| `autovacuum_analyze_threshold` | 50 | จำนวนแถวที่เปลี่ยนขั้นต่ำก่อนพิจารณา analyze |
| `autovacuum_analyze_scale_factor` | 0.1 (10%) | สัดส่วนของจำนวนแถวทั้งหมดสำหรับ analyze |

ตัวอย่าง: ตาราง `products` มี 5,000 แถว (`reltuples ≈ 5000`)

```
vacuum threshold = 50 + 0.2 * 5000 = 1,050 dead tuples
```

หมายความว่า autovacuum จะ vacuum ตาราง `products` ก็ต่อเมื่อมี dead tuple สะสมเกิน 1,050 แถว — ซึ่งเราสร้างไปแล้ว 9,000 แถว! แต่เพราะเราปิด `autovacuum_enabled = false` ไว้ที่ตารางนี้โดยเฉพาะ มันจึงยังไม่ถูก vacuum

```sql
SHOW autovacuum_vacuum_threshold;
SHOW autovacuum_vacuum_scale_factor;
SHOW autovacuum_max_workers;
SHOW autovacuum_naptime;
```

```
 autovacuum_vacuum_threshold
------------------------------
 50

 autovacuum_vacuum_scale_factor
---------------------------------
 0.2

 autovacuum_max_workers
-------------------------
 3

 autovacuum_naptime
---------------------
 1min
```

### Cost-based Throttling

Autovacuum ถูกออกแบบให้ **ไม่แย่ง I/O จนกระทบ query ปกติมากเกินไป** โดยใช้ระบบ "cost limit":

| พารามิเตอร์ | ค่า default (PG 16/17) | ความหมาย |
|---|---|---|
| `autovacuum_vacuum_cost_delay` | 2ms | เวลาพักหลังจากใช้ cost ครบโควตา |
| `autovacuum_vacuum_cost_limit` | 200 (หรือ -1 = ใช้ `vacuum_cost_limit`) | โควตา cost ต่อรอบก่อนพัก |

พูดง่ายๆ คือ autovacuum จะทำงานเป็นช่วงสั้นๆ แล้วพักเป็นจังหวะ เพื่อไม่ให้กิน I/O bandwidth ทั้งหมดของเซิร์ฟเวอร์ — ข้อดีคือไม่รบกวน production มาก แต่ข้อเสียคือถ้าตารางใหญ่มากและ churn สูง autovacuum อาจ**ตามไม่ทัน**

### ตรวจสอบว่า autovacuum กำลังทำงานอยู่หรือไม่

```sql
SELECT pid, datname, relid::regclass AS table_name, phase,
       heap_blks_total, heap_blks_scanned,
       round(100.0 * heap_blks_scanned / GREATEST(heap_blks_total,1), 1) AS pct_done
FROM pg_stat_progress_vacuum;
```

```
  pid  | datname  | table_name | phase               | heap_blks_total | heap_blks_scanned | pct_done
-------+----------+------------+----------------------+------------------+---------------------+----------
 18422 | ecommerce| orders     | scanning heap        |              352 |                 210 |     59.7
```

ตอนนี้ให้เปิด autovacuum กลับมาที่ `products` แล้วรอดู:

```sql
ALTER TABLE products SET (autovacuum_enabled = true);
```

ภายในไม่กี่นาที (ตาม `autovacuum_naptime`) worker จะเข้ามาเก็บกวาด dead tuple ที่สะสมไว้โดยอัตโนมัติ

---

## Step 595: การปรับแต่ง Autovacuum ต่อตาราง (Per-table Tuning)

ค่า default ของ autovacuum (`scale_factor = 0.2`) เหมาะกับตารางขนาดเล็ก-กลางทั่วไป แต่ **ไม่เหมาะกับตารางขนาดใหญ่มากหรือตารางที่มี churn สูง**

### ปัญหาของ scale_factor แบบ default บนตารางใหญ่

ตาราง `order_items` มี 55,000 แถว → threshold = 50 + 0.2 × 55,000 = **11,050 dead tuples** ก่อนจะถูก vacuum — ถ้าตารางนี้มี 50 ล้านแถว threshold จะกลายเป็น **10 ล้าน dead tuples** ซึ่งกว่าจะถึงเกณฑ์นั้น ตารางอาจบวมจนกิน disk และ cache มหาศาลไปแล้ว

**แนวทางแก้**: ลด `scale_factor` และใช้ `threshold` แบบตายตัวแทน สำหรับตารางใหญ่หรือตารางที่อัปเดตถี่

```sql
-- order_items: ตารางใหญ่ อัปเดต/ลบบ่อย ต้องการ vacuum ถี่ขึ้นมาก
ALTER TABLE order_items SET (
    autovacuum_vacuum_scale_factor = 0.02,   -- 2% แทนที่จะเป็น 20%
    autovacuum_vacuum_threshold    = 500,
    autovacuum_analyze_scale_factor = 0.02,
    autovacuum_analyze_threshold    = 500
);

-- orders: สถานะเปลี่ยนบ่อยมาก (pending -> paid -> shipped -> delivered)
ALTER TABLE orders SET (
    autovacuum_vacuum_scale_factor = 0.05,
    autovacuum_vacuum_threshold    = 200,
    autovacuum_vacuum_cost_delay   = 0        -- ให้ vacuum ทำงานเต็มสปีดสำหรับตารางสำคัญนี้
);

-- products: ปรับให้ vacuum ไวขึ้นเช่นกัน หลังจากที่เราเพิ่งเปิดกลับมาใน Step 594
ALTER TABLE products SET (
    autovacuum_vacuum_scale_factor = 0.05,
    autovacuum_vacuum_threshold    = 100
);
```

ตรวจสอบค่าที่ตั้งไว้:

```sql
SELECT relname, reloptions
FROM pg_class
WHERE relname IN ('order_items','orders','products');
```

```
   relname   |                                reloptions
-------------+---------------------------------------------------------------------------
 order_items | {autovacuum_vacuum_scale_factor=0.02,autovacuum_vacuum_threshold=500,...}
 orders      | {autovacuum_vacuum_scale_factor=0.05,autovacuum_vacuum_threshold=200,...}
 products    | {autovacuum_vacuum_scale_factor=0.05,autovacuum_vacuum_threshold=100}
```

### กรณีพิเศษ: ปิด autovacuum ชั่วคราวระหว่าง bulk load

เมื่อจะโหลดข้อมูลจำนวนมหาศาลเข้าตาราง (เช่น ETL, migration) การให้ autovacuum เข้ามาทำงานระหว่างโหลดอาจทำให้ I/O แย่งกันโดยไม่จำเป็น แนวทางที่นิยม:

```sql
-- ก่อนเริ่ม bulk load ขนาดใหญ่
ALTER TABLE staging_table SET (autovacuum_enabled = false);

-- ... โหลดข้อมูลจำนวนมาก ...

-- โหลดเสร็จแล้ว vacuum + analyze เอง แล้วค่อยเปิด autovacuum กลับ
VACUUM ANALYZE staging_table;
ALTER TABLE staging_table SET (autovacuum_enabled = true);
```

> **ข้อควรระวัง**: อย่าลืมเปิด autovacuum กลับคืนเสมอ มิฉะนั้นตารางนั้นจะไม่ถูกดูแลอีกเลย ซึ่งเป็นสาเหตุอันดับต้นๆ ของปัญหา bloat และ wraparound ในระบบจริง (กรณีของเราตอนต้นบทที่ลืมเปิด `products` กลับมาคือตัวอย่างที่ดี!)

### พารามิเตอร์ per-table อื่นๆ ที่มีประโยชน์

```sql
ALTER TABLE order_items SET (
    fillfactor = 90,                     -- เผื่อพื้นที่ 10% ในแต่ละ page สำหรับ HOT update
    autovacuum_vacuum_cost_limit = 1000   -- เพิ่มโควตา I/O ให้ vacuum ทำงานเร็วขึ้นเฉพาะตารางนี้
);
```

`fillfactor` ต่ำกว่า 100 จะเว้นพื้นที่ว่างในแต่ละหน้าข้อมูลไว้ล่วงหน้า ทำให้ UPDATE ที่ไม่แก้ไขคอลัมน์ที่มี index (Heap-Only Tuple / HOT update) สามารถเขียนแถวใหม่ในหน้าเดิมได้โดยไม่ต้องขยายไฟล์หรือแก้ index ทุกตัว — ช่วยลด bloat ได้มากสำหรับตารางที่ UPDATE บ่อย

---

## Step 596: ANALYZE และ VACUUM ANALYZE — อัปเดตสถิติสำหรับ Query Planner

### ANALYZE คืออะไร

`ANALYZE` คือคำสั่งที่สุ่มตัวอย่างข้อมูลในตาราง แล้วเก็บสถิติ (histogram, most common values, null fraction, correlation ฯลฯ) ไว้ใน system catalog `pg_statistic` (ดูผ่าน view `pg_stats`) — **สถิติเหล่านี้คือสิ่งที่ query planner ใช้ตัดสินใจเลือกแผนการ query** (เช่น จะใช้ index scan หรือ sequential scan, จะ join ด้วย nested loop หรือ hash join)

```sql
ANALYZE products;

SELECT attname, n_distinct, correlation, null_frac
FROM pg_stats
WHERE tablename = 'products' AND attname IN ('category_id','unit_price');
```

```
   attname    | n_distinct | correlation | null_frac
--------------+------------+-------------+-----------
 category_id  |         20 |        0.12 |         0
 unit_price   |       4230 |       -0.03 |         0
```

### ทำไมสถิติล้าสมัยถึงทำให้ query planner เลือกแผนผิด

มาดูตัวอย่างจริง สมมติเราเพิ่งเพิ่มสินค้าหมวดใหม่จำนวนมากแบบ bulk insert โดยยังไม่ได้ analyze:

```sql
-- ปิด autovacuum ชั่วคราวเพื่อสาธิต (ในสถานการณ์จริงอาจเกิดจาก bulk insert ที่เร็วกว่า autovacuum ตาม)
BEGIN;

INSERT INTO products (product_name, category_id, supplier_id, unit_price, stock_quantity)
SELECT 'FlashSale ' || g, 21, 1, 99.00, 1000
FROM generate_series(1, 3000) AS g;
-- หมายเหตุ: category_id = 21 ยังไม่มีอยู่จริงใน categories แต่เพื่อสาธิตสถิติเราข้าม FK check ในตัวอย่างนี้

COMMIT;

-- ก่อน ANALYZE: planner ยังใช้สถิติเก่าที่ไม่รู้จักข้อมูลใหม่ 3,000 แถวนี้เลย
EXPLAIN SELECT * FROM products WHERE category_id = 21;
```

```
 Seq Scan on products  (cost=0.00..104.50 rows=1 width=120)
   Filter: (category_id = 21)
```

Planner **ประเมินว่าจะเจอแค่ 1 แถว** (เพราะสถิติเก่าไม่รู้จัก category_id = 21 เลย) จึงเลือก Sequential Scan อย่างมั่นใจ ทั้งที่จริงมีถึง 3,000 แถว — ถ้า query นี้ถูกใช้เป็นส่วนหนึ่งของ nested loop join กับตารางอื่น การประเมินผิดพลาดขนาดนี้อาจทำให้ planner เลือก nested loop (ซึ่งเหมาะกับผลลัพธ์น้อยๆ) ทั้งที่ควรใช้ hash join — ส่งผลให้ query ช้าลงมหาศาล

```sql
ANALYZE products;

EXPLAIN SELECT * FROM products WHERE category_id = 21;
```

```
 Seq Scan on products  (cost=0.00..118.00 rows=3000 width=120)
   Filter: (category_id = 21)
```

หลัง ANALYZE, planner ประเมิน **rows=3000** ได้ถูกต้อง (ในกรณีนี้ยังเลือก Seq Scan เพราะสัดส่วนสูง ซึ่งถูกต้องแล้ว — แต่ถ้าสัดส่วนต่ำ มันจะเปลี่ยนไปเลือก Index Scan แทนโดยอัตโนมัติ)

### VACUUM ANALYZE — ทำสองอย่างพร้อมกันในรอบเดียว

```sql
VACUUM ANALYZE order_items;
```

การรัน `VACUUM` และ `ANALYZE` พร้อมกันมีประโยชน์เพราะ**ทั้งสองต้องสแกนตารางอยู่แล้ว** การรวมเป็นคำสั่งเดียวจึงประหยัด I/O กว่าการรันแยกสองรอบ — นี่คือสิ่งที่ **autovacuum worker ทำเป็นค่าเริ่มต้นอยู่แล้ว** (autovacuum จะ vacuum และ analyze ในรอบเดียวกันเมื่อทั้งสองเกณฑ์ถึงพร้อมกัน)

### ปรับความละเอียดของสถิติ

```sql
-- เพิ่มความละเอียดสถิติเฉพาะคอลัมน์ที่ query ใช้กรองบ่อยและมีการกระจายตัวซับซ้อน
ALTER TABLE products ALTER COLUMN unit_price SET STATISTICS 500;
ANALYZE products;

SHOW default_statistics_target;
```

```
 default_statistics_target
----------------------------
 100
```

ค่า default คือ 100 (แถวตัวอย่างและ histogram bucket) — การเพิ่มเป็น 500 สำหรับคอลัมน์ที่มีการกระจายตัวซับซ้อน (เช่น long-tail distribution) ช่วยให้ planner ประเมิน selectivity แม่นยำขึ้น แลกกับเวลา ANALYZE ที่นานขึ้นเล็กน้อยและพื้นที่เก็บสถิติที่มากขึ้น

---

## Step 597: Transaction ID Wraparound — ปัญหาระยะยาวที่ร้ายแรง

### ทำไม Wraparound ถึงเกิดขึ้น

PostgreSQL ใช้ **Transaction ID (XID) แบบ 32-bit** ในการกำกับลำดับ transaction (`xmin`/`xmax` ที่เราเห็นก่อนหน้านี้) ซึ่งมีค่าได้ประมาณ **4,000 ล้านค่า (2^32)** และถูกนำมาใช้แบบ**วงกลม (circular)** — เมื่อ XID วิ่งครบรอบและย้อนกลับมาทับค่าเดิม ระบบจะสับสนว่าแถวไหน "เก่ากว่า" หรือ "ใหม่กว่า" กัน

ถ้าปล่อยให้เกิด wraparound จริง **แถวข้อมูลเก่าจะดูเหมือนอยู่ใน "อนาคต"** และกลายเป็นมองไม่เห็น (invisible) ทันที — เท่ากับ**ข้อมูลหายไปทั้งที่ยังอยู่ในดิสก์** ซึ่งเป็นหายนะระดับร้ายแรงที่สุดอย่างหนึ่งที่ PostgreSQL อาจเจอได้

### Freezing คือทางแก้

เพื่อป้องกันปัญหานี้ VACUUM จะทำ **freezing**: เปลี่ยนค่า `xmin` ของ tuple ที่เก่าพอ (ไม่มีใครต้องดู transaction เก่าขนาดนั้นแล้ว) ให้กลายเป็นค่าพิเศษ `FrozenTransactionId` ซึ่งแปลว่า **"tuple นี้มองเห็นได้เสมอ ไม่ว่า XID ปัจจุบันจะเป็นเท่าไหร่"** — ทำให้ tuple นั้นไม่ต้องสนใจปัญหาการวนรอบของ XID อีกต่อไป

```sql
SELECT relname,
       age(relfrozenxid) AS xid_age,
       relfrozenxid
FROM pg_class
WHERE relname IN ('products','orders','order_items')
ORDER BY xid_age DESC;
```

```
   relname   | xid_age | relfrozenxid
-------------+---------+---------------
 orders      |   18432 |       5164210
 order_items |   12100 |       5170541
 products    |    9820 |       5172863
```

`age(relfrozenxid)` บอกว่า "XID ปัจจุบันห่างจาก relfrozenxid ของตารางนี้กี่ transaction แล้ว" — ยิ่งค่านี้สูง ยิ่งใกล้ wraparound มากขึ้น

### พารามิเตอร์ควบคุม

| พารามิเตอร์ | ค่า default | ความหมาย |
|---|---|---|
| `autovacuum_freeze_max_age` | 200,000,000 | อายุ XID สูงสุดก่อนที่ตารางจะถูกบังคับ vacuum แบบ "wraparound protection" |
| `vacuum_freeze_min_age` | 50,000,000 | อายุ XID ขั้นต่ำก่อนที่ tuple จะถูก freeze ระหว่าง vacuum ปกติ |
| `vacuum_freeze_table_age` | 150,000,000 | อายุที่ทำให้ vacuum สแกนทั้งตาราง (aggressive) แทนที่จะข้ามหน้าที่ freeze แล้ว |
| `autovacuum_multixact_freeze_max_age` | 400,000,000 | เหมือน freeze_max_age แต่สำหรับ Multixact ID (ใช้ตอน row-level lock หลาย transaction) |

### Anti-wraparound VACUUM (บังคับ ไม่มีใครหยุดได้)

เมื่อ `age(relfrozenxid)` ของตารางใดใกล้แตะ `autovacuum_freeze_max_age` (200 ล้าน) PostgreSQL จะสั่ง **"anti-wraparound autovacuum"** โดยอัตโนมัติ ซึ่งมีคุณสมบัติพิเศษ:

- **ไม่สนใจ cost-based throttling** — ทำงานเต็มสปีดจนกว่าจะเสร็จ
- **ไม่สามารถถูกยกเลิกได้ด้วยคำสั่งง่ายๆ** (การพยายาม cancel จะถูกเพิกเฉยในสถานการณ์วิกฤต)
- ทำงานแม้ตาราง config `autovacuum_enabled = false` ก็ตาม (เพราะเป็นเรื่องความปลอดภัยของข้อมูล ไม่ใช่เรื่อง performance)

### เมื่อสถานการณ์เลวร้ายที่สุด

ถ้า `age(relfrozenxid)` วิ่งไปถึงประมาณ **2,000 ล้าน XID (1 พันล้านก่อนจุดวนรอบจริง)** PostgreSQL จะ**ปฏิเสธการรับ transaction ใหม่ทั้งหมด** และแสดง error:

```
ERROR:  database is not accepting commands to avoid wraparound data loss in database "ecommerce"
HINT:  Stop the postmaster and vacuum that database in single-user mode.
```

ในจุดนี้ฐานข้อมูล**หยุดทำงานสำหรับการเขียนทุกชนิด** จนกว่าผู้ดูแลระบบจะเข้าไปรัน VACUUM แบบ single-user mode ด้วยตนเอง — เป็นสถานการณ์ downtime เต็มรูปแบบที่**ป้องกันได้ง่ายมาก** เพียงแค่ไม่ปิด autovacuum และคอยตรวจสอบ `age(relfrozenxid)` เป็นระยะ

### ตรวจสอบเชิงป้องกัน

```sql
SELECT relname,
       age(relfrozenxid) AS xid_age,
       round(100.0 * age(relfrozenxid) /
             current_setting('autovacuum_freeze_max_age')::bigint, 2) AS pct_toward_limit
FROM pg_class
WHERE relkind = 'r'
ORDER BY xid_age DESC
LIMIT 10;
```

```
   relname   | xid_age | pct_toward_limit
-------------+---------+-------------------
 orders      |   18432 |              0.01
 order_items |   12100 |              0.01
 products    |    9820 |              0.00
```

> **แนวทางปฏิบัติ**: ตั้ง alert เมื่อ `pct_toward_limit` เกิน **50-70%** ในระบบ production เพื่อให้มีเวลาตรวจสอบและแก้ไขก่อนถึงจุดวิกฤต

---

## Step 598: Monitoring VACUUM

### pg_stat_user_tables — ภาพรวมสถานะแต่ละตาราง

```sql
SELECT relname,
       n_live_tup,
       n_dead_tup,
       last_vacuum,
       last_autovacuum,
       last_analyze,
       last_autoanalyze,
       vacuum_count,
       autovacuum_count
FROM pg_stat_user_tables
WHERE relname IN ('products','orders','order_items')
ORDER BY relname;
```

```
   relname   | n_live_tup | n_dead_tup |    last_vacuum      |   last_autovacuum    | vacuum_count | autovacuum_count
-------------+------------+------------+----------------------+-----------------------+---------------+-------------------
 order_items |      54200 |         42 | (null)                | 2026-09-25 10:14:02   |             0 |                 3
 orders      |      19850 |         88 | 2026-09-25 09:02:11   | 2026-09-25 10:20:44   |             1 |                 5
 products    |       8000 |         15 | 2026-09-25 10:05:33   | 2026-09-25 10:31:19   |             2 |                 4
```

คอลัมน์สำคัญ:

- `n_live_tup` / `n_dead_tup`: จำนวนแถวที่ "มีชีวิต" และ "ตายแล้ว" โดยประมาณ
- `last_vacuum` / `last_autovacuum`: เวลาที่ manual VACUUM หรือ autovacuum ทำงานล่าสุด
- `last_analyze` / `last_autoanalyze`: เวลาที่ ANALYZE ทำงานล่าสุด
- `vacuum_count` / `autovacuum_count`: จำนวนครั้งสะสมที่แต่ละกลไกทำงาน

### pg_stat_progress_vacuum — ดูความคืบหน้าแบบ real-time

```sql
SELECT p.pid,
       p.relid::regclass AS table_name,
       p.phase,
       p.heap_blks_total,
       p.heap_blks_scanned,
       p.heap_blks_vacuumed,
       p.index_vacuum_count,
       p.max_dead_tuple_bytes,
       p.num_dead_item_ids
FROM pg_stat_progress_vacuum p;
```

ค่า `phase` ที่อาจพบ: `initializing`, `scanning heap`, `vacuuming indexes`, `vacuuming heap`, `cleaning up indexes`, `truncating heap`, `performing final cleanup` — มีประโยชน์มากเมื่อ VACUUM ตารางใหญ่ใช้เวลานานผิดปกติ และต้องการรู้ว่ามันค้างอยู่ที่ phase ไหน

### เปิด log สำหรับ autovacuum ที่ทำงานนาน

```sql
-- ตั้งค่าใน postgresql.conf หรือผ่าน ALTER SYSTEM
ALTER SYSTEM SET log_autovacuum_min_duration = '250ms';
SELECT pg_reload_conf();
```

หลังจากตั้งค่านี้ autovacuum ตัวไหนที่ทำงานนานกว่า 250ms จะถูกบันทึกลง server log พร้อมรายละเอียด:

```
LOG:  automatic vacuum of table "ecommerce.public.order_items": index scans: 1
        pages: 0 removed, 6800 remain, 0 skipped due to pins, 120 skipped frozen
        tuples: 4200 removed, 54200 remain, 0 are dead but not yet removable
        buffer usage: 8102 hits, 340 misses, 210 dirtied
        avg read rate: 9.421 MB/s, avg write rate: 5.812 MB/s
        system usage: CPU: user: 0.18 s, system: 0.02 s, elapsed: 1.34 s
```

การตั้งค่านี้ให้ค่าต่ำ (เช่น 0 = log ทุกครั้ง) ในช่วง troubleshooting แล้วปรับกลับให้สูงขึ้นในสภาวะปกติ (เช่น 1000ms) เพื่อไม่ให้ log รกเกินไป

### Query สรุปภาพรวมทั้งฐานข้อมูล (Health Check)

```sql
SELECT
    schemaname,
    relname,
    n_live_tup,
    n_dead_tup,
    round(100.0 * n_dead_tup / GREATEST(n_live_tup + n_dead_tup, 1), 2) AS dead_pct,
    coalesce(last_autovacuum, last_vacuum) AS last_vacuumed,
    coalesce(last_autoanalyze, last_analyze) AS last_analyzed,
    pg_size_pretty(pg_total_relation_size(relid)) AS total_size
FROM pg_stat_user_tables
WHERE n_live_tup + n_dead_tup > 0
ORDER BY dead_pct DESC
LIMIT 20;
```

Query นี้เหมาะสำหรับใส่ใน dashboard ตรวจสุขภาพประจำวัน — เราจะนำแนวคิดนี้ไปใช้ในโปรเจกต์ Analytics Dashboard ที่ Step 600

---

## Step 599: Best Practices — สรุปรวมการดูแล VACUUM/Autovacuum

### เมื่อไหร่ควร Manual VACUUM

1. **หลัง bulk load/bulk delete ขนาดใหญ่** — เพื่อให้สถิติและ FSM อัปเดตทันที ไม่ต้องรอรอบ autovacuum ถัดไป
2. **ก่อนการทำ maintenance ที่ critical** เช่นก่อน major version upgrade หรือก่อน backup ขนาดใหญ่
3. **เมื่อพบว่า autovacuum ตามไม่ทัน** (`n_dead_tup` โตต่อเนื่องแม้ autovacuum รันไปแล้วหลายรอบ) — ให้รัน manual VACUUM ควบคู่กับการปรับจูน scale_factor

### เมื่อไหร่ควรปรับแต่ง Autovacuum

- ตารางมี**แถวจำนวนมาก** (หลักล้านขึ้นไป) — ลด `scale_factor`, ใช้ `threshold` ตายตัวแทน (Step 595)
- ตารางมี**อัตรา UPDATE/DELETE สูงมาก** (เช่น queue, session, cache table) — ลด `scale_factor` ให้ต่ำมาก และพิจารณา `autovacuum_vacuum_cost_delay = 0`
- ตารางที่ **insert-only แบบ append** (เช่น log, event table) — ยังต้อง analyze บ่อยเพื่อสถิติที่แม่นยำ แต่ vacuum อาจไม่จำเป็นบ่อยเท่า (ไม่ค่อยมี dead tuple) ยกเว้นเพื่อ freeze ป้องกัน wraparound

### สัญญาณเตือนที่ต้องระวัง (Red Flags)

| สัญญาณ | ความเสี่ยง | วิธีตรวจสอบ |
|---|---|---|
| `n_dead_tup` สูงต่อเนื่องและโตขึ้นเรื่อยๆ | Bloat สะสม, query ช้าลง | `pg_stat_user_tables` |
| `last_autovacuum` เป็น NULL หรือเก่ามาก | ตารางไม่ถูกดูแลเลย | `pg_stat_user_tables` |
| `age(relfrozenxid)` ใกล้ `autovacuum_freeze_max_age` | เสี่ยง wraparound | Query ใน Step 597 |
| มี long-running transaction (`idle in transaction` นานๆ) | บล็อกไม่ให้ VACUUM ลบ dead tuple ได้ | `pg_stat_activity` |
| Replication slot ที่ inactive นาน | เก็บ WAL/snapshot เก่าไว้ ทำให้ vacuum ลบไม่ได้ | `pg_replication_slots` |
| ขนาดตารางโตเร็วกว่าจำนวนแถวที่เพิ่มขึ้นจริง | Bloat ชัดเจน | เทียบ `pg_total_relation_size` กับ `n_live_tup` ตามช่วงเวลา |

### ตรวจ Long-running Transaction ที่บล็อก VACUUM

```sql
SELECT pid, usename, state, now() - xact_start AS duration, query
FROM pg_stat_activity
WHERE state != 'idle'
  AND xact_start IS NOT NULL
ORDER BY duration DESC
LIMIT 10;
```

```
  pid  | usename  |        state        | duration  |                query
-------+----------+----------------------+-----------+--------------------------------------
 20144 | app_user | idle in transaction  | 02:14:33  | SELECT * FROM orders WHERE ...
```

Transaction ที่ค้างนานขนาดนี้ (`idle in transaction` 2+ ชั่วโมง) จะทำให้ VACUUM ไม่สามารถลบ dead tuple ที่เกิดขึ้น**หลัง**จุดเริ่ม transaction นี้ได้เลย เพราะ transaction นั้นยัง "อาจ" ต้องมองเห็นข้อมูล ณ ช่วงเวลานั้นอยู่ — ควรตั้ง `idle_in_transaction_session_timeout` เพื่อป้องกันปัญหานี้ในระบบ production:

```sql
ALTER SYSTEM SET idle_in_transaction_session_timeout = '5min';
SELECT pg_reload_conf();
```

### Checklist สรุป

- [ ] `autovacuum = on` เสมอในระบบ production (ไม่มีเหตุผลที่ดีพอจะปิดถาวร)
- [ ] ตารางใหญ่/churn สูง ปรับ `scale_factor` ให้ต่ำกว่า default
- [ ] เพิ่ม `maintenance_work_mem` ให้เพียงพอ (VACUUM ใช้หน่วยความจำนี้เก็บรายการ dead tuple — ยิ่งมากยิ่งลด index scan รอบ)
- [ ] Monitor `n_dead_tup`, `age(relfrozenxid)`, `last_autovacuum` เป็นประจำ (ทำ dashboard ดังใน Step 600)
- [ ] เปิด `log_autovacuum_min_duration` เพื่อเก็บ log ประวัติ
- [ ] หลีกเลี่ยง `VACUUM FULL` ในเวลาทำการ ใช้เฉพาะ maintenance window หรือใช้ `pg_repack`
- [ ] จำกัดเวลา idle-in-transaction ด้วย `idle_in_transaction_session_timeout`
- [ ] ตรวจสอบ replication slot ที่ไม่ได้ใช้งานแล้วให้ลบทิ้ง

---

## Step 600: โปรเจกต์ปิดท้ายระดับสูง — Analytics Dashboard

นี่คือโปรเจกต์สุดท้ายที่จะ**สังเคราะห์ทุกเทคนิคของระดับ Advanced** เข้าด้วยกัน เราจะสร้างระบบ **Analytics Dashboard** สำหรับร้านค้าออนไลน์ ที่ประกอบด้วย:

1. Index ที่เหมาะสม (B-Tree, GIN สำหรับ JSONB, GIN สำหรับ Full Text Search, Partial Index, BRIN)
2. Summary table สไตล์ materialized view สำหรับ dashboard ที่ต้องการความเร็วสูง
3. Full Text Search สำหรับค้นหารีวิวลูกค้า
4. การออกแบบพร้อมสำหรับ Partitioning (partition-ready design)
5. PL/pgSQL function สำหรับสร้างรายงาน
6. Trigger สำหรับ audit การเปลี่ยนแปลงราคาสินค้า
7. แผน VACUUM/Monitoring แบบครบวงจร

### 600.1 Index ที่เหมาะสม

```sql
-- B-Tree มาตรฐานสำหรับ foreign key และการกรอง/เรียงข้อมูลที่ใช้บ่อย
CREATE INDEX idx_orders_customer_id ON orders (customer_id);
CREATE INDEX idx_orders_order_date  ON orders (order_date DESC);
CREATE INDEX idx_order_items_order_id   ON order_items (order_id);
CREATE INDEX idx_order_items_product_id ON order_items (product_id);
CREATE INDEX idx_reviews_product_id ON reviews (product_id);

-- Composite index สำหรับ query แดชบอร์ดที่กรองตามลูกค้า+ช่วงเวลาพร้อมกัน
CREATE INDEX idx_orders_customer_date ON orders (customer_id, order_date DESC);

-- Partial index: เฉพาะออเดอร์ที่ยัง "ทำงานอยู่" (ใช้บ่อยในหน้า operations dashboard)
CREATE INDEX idx_orders_active
    ON orders (order_date)
    WHERE status IN ('pending','paid','shipped');

-- GIN index สำหรับค้นหาใน JSONB attributes (เช่น ค้นหาสินค้าสีแดง หรือมี tag 'bestseller')
CREATE INDEX idx_products_attributes_gin ON products USING GIN (attributes jsonb_path_ops);

-- BRIN index บนคอลัมน์ที่เรียงตามเวลาธรรมชาติและตารางมีขนาดใหญ่มาก (append-heavy)
-- เหมาะกับ order_date เพราะข้อมูลใหม่จะถูก insert ท้ายตารางเรื่อยๆ ตามเวลาจริง
CREATE INDEX idx_orders_order_date_brin ON orders USING BRIN (order_date);
```

ทดสอบการใช้งาน GIN index กับ JSONB:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT product_id, product_name, attributes
FROM products
WHERE attributes @> '{"color": "red"}'
LIMIT 20;
```

```
 Bitmap Heap Scan on products  (cost=12.45..156.30 rows=250 width=180) (actual time=0.412..1.203 rows=20 loops=1)
   Recheck Cond: (attributes @> '{"color": "red"}'::jsonb)
   Heap Blocks: exact=18
   ->  Bitmap Index Scan on idx_products_attributes_gin  (cost=0.00..12.38 rows=250 width=0) (actual time=0.201..0.201 rows=250 loops=1)
         Index Cond: (attributes @> '{"color": "red"}'::jsonb)
 Planning Time: 0.180 ms
 Execution Time: 1.245 ms
```

### 600.2 Full Text Search สำหรับรีวิวสินค้า

เพิ่มคอลัมน์ `tsvector` แบบ generated column (คำนวณอัตโนมัติ, PostgreSQL 12+) และสร้าง GIN index:

```sql
ALTER TABLE reviews
    ADD COLUMN review_tsv tsvector
    GENERATED ALWAYS AS (to_tsvector('simple', coalesce(review_text, ''))) STORED;

CREATE INDEX idx_reviews_tsv ON reviews USING GIN (review_tsv);
```

> **หมายเหตุ**: เราใช้ text search configuration `'simple'` เพราะ PostgreSQL ไม่มี dictionary ภาษาไทยในตัว ในระบบจริงที่ต้องการรองรับภาษาไทยแบบเต็มรูปแบบ ควรพิจารณาติดตั้ง extension เช่น `pg_search`/`zhparser` หรือทำ tokenization ภาษาไทยที่ระดับแอปพลิเคชันก่อนแล้วค่อย normalize เข้ามาเก็บ

ฟังก์ชันค้นหารีวิว:

```sql
CREATE OR REPLACE FUNCTION search_reviews(search_term TEXT, limit_count INTEGER DEFAULT 20)
RETURNS TABLE (
    review_id   INTEGER,
    product_id  INTEGER,
    product_name VARCHAR,
    rating      INTEGER,
    review_text TEXT,
    rank        REAL
) AS $$
BEGIN
    RETURN QUERY
    SELECT r.review_id, r.product_id, p.product_name, r.rating, r.review_text,
           ts_rank(r.review_tsv, query) AS rank
    FROM reviews r
    JOIN products p ON p.product_id = r.product_id,
         plainto_tsquery('simple', search_term) query
    WHERE r.review_tsv @@ query
    ORDER BY rank DESC
    LIMIT limit_count;
END;
$$ LANGUAGE plpgsql STABLE;

SELECT * FROM search_reviews('จัดส่งรวดเร็ว คุณภาพดี');
```

```
 review_id | product_id |  product_name   | rating |                        review_text                        |   rank
-----------+------------+-------------------+--------+------------------------------------------------------------+-----------
      4821 |       1203 | Product 1203      |      5 | สินค้าคุณภาพดีมาก จัดส่งรวดเร็ว บรรจุภัณฑ์แน่นหนา           | 0.1517143
      2093 |        341 | Product 341       |      5 | สินค้าคุณภาพดีมาก จัดส่งรวดเร็ว บรรจุภัณฑ์แน่นหนา           | 0.1517143
```

### 600.3 Summary Table สไตล์ Materialized View

สำหรับ dashboard ที่ query บ่อยและต้องการความเร็วสูง เราจะสร้างทั้ง**ตาราง summary แบบ pre-aggregate** และ**Materialized View จริง** เพื่อเปรียบเทียบแนวทาง

**แนวทางที่ 1: Materialized View มาตรฐาน**

```sql
CREATE MATERIALIZED VIEW mv_daily_sales_summary AS
SELECT
    date_trunc('day', o.order_date)::date AS sale_date,
    p.category_id,
    c.category_name,
    count(DISTINCT o.order_id)  AS order_count,
    sum(oi.quantity)            AS units_sold,
    sum(oi.quantity * oi.unit_price) AS revenue
FROM orders o
JOIN order_items oi ON oi.order_id = o.order_id
JOIN products p      ON p.product_id = oi.product_id
JOIN categories c    ON c.category_id = p.category_id
WHERE o.status IN ('paid','shipped','delivered')
GROUP BY 1, 2, 3
WITH DATA;

-- ต้องมี unique index ก่อนถึงจะใช้ REFRESH ... CONCURRENTLY ได้ (ไม่ล็อก read ระหว่าง refresh)
CREATE UNIQUE INDEX idx_mv_daily_sales_pk
    ON mv_daily_sales_summary (sale_date, category_id);

CREATE INDEX idx_mv_daily_sales_date ON mv_daily_sales_summary (sale_date);
```

```sql
-- Query dashboard เร็วมาก เพราะข้อมูลถูก pre-aggregate ไว้แล้ว
SELECT sale_date, category_name, order_count, units_sold, revenue
FROM mv_daily_sales_summary
WHERE sale_date >= current_date - 30
ORDER BY sale_date DESC, revenue DESC
LIMIT 10;
```

การ refresh เมื่อข้อมูลเปลี่ยน:

```sql
REFRESH MATERIALIZED VIEW CONCURRENTLY mv_daily_sales_summary;
```

`CONCURRENTLY` ทำให้ query ยังอ่าน view เก่าได้ระหว่าง refresh (ไม่ต้องรอ) แต่ต้องการ unique index ดังที่สร้างไว้ข้างต้น และใช้เวลานานกว่า refresh แบบธรรมดาเล็กน้อยเพราะต้องเปรียบเทียบ row เก่า-ใหม่

**แนวทางที่ 2: Summary table แบบ manual (ยืดหยุ่นกว่า สำหรับ incremental update)**

```sql
CREATE TABLE dashboard_summary (
    summary_date   DATE NOT NULL,
    category_id    INTEGER NOT NULL REFERENCES categories(category_id),
    order_count    INTEGER NOT NULL DEFAULT 0,
    units_sold     INTEGER NOT NULL DEFAULT 0,
    revenue        NUMERIC(14,2) NOT NULL DEFAULT 0,
    avg_rating     NUMERIC(3,2),
    updated_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (summary_date, category_id)
);

CREATE INDEX idx_dashboard_summary_date ON dashboard_summary (summary_date DESC);
```

ฟังก์ชันสำหรับ refresh summary table แบบเจาะจงช่วงวันที่ (ใช้ `INSERT ... ON CONFLICT` เพื่อ upsert):

```sql
CREATE OR REPLACE FUNCTION refresh_dashboard_summary(p_start_date DATE, p_end_date DATE)
RETURNS INTEGER AS $$
DECLARE
    v_rows_affected INTEGER;
BEGIN
    INSERT INTO dashboard_summary (summary_date, category_id, order_count, units_sold, revenue, avg_rating, updated_at)
    SELECT
        date_trunc('day', o.order_date)::date,
        p.category_id,
        count(DISTINCT o.order_id),
        sum(oi.quantity),
        sum(oi.quantity * oi.unit_price),
        (SELECT round(avg(r.rating), 2) FROM reviews r WHERE r.product_id = p.product_id),
        now()
    FROM orders o
    JOIN order_items oi ON oi.order_id = o.order_id
    JOIN products p      ON p.product_id = oi.product_id
    WHERE o.status IN ('paid','shipped','delivered')
      AND o.order_date::date BETWEEN p_start_date AND p_end_date
    GROUP BY 1, 2
    ON CONFLICT (summary_date, category_id)
    DO UPDATE SET
        order_count = EXCLUDED.order_count,
        units_sold  = EXCLUDED.units_sold,
        revenue     = EXCLUDED.revenue,
        avg_rating  = EXCLUDED.avg_rating,
        updated_at  = now();

    GET DIAGNOSTICS v_rows_affected = ROW_COUNT;
    RETURN v_rows_affected;
END;
$$ LANGUAGE plpgsql;

-- รัน refresh สำหรับ 90 วันล่าสุด
SELECT refresh_dashboard_summary(current_date - 90, current_date);
```

```
 refresh_dashboard_summary
----------------------------
                       1840
```

> แนวทางนี้เหมาะกับกรณีที่ต้องการ**อัปเดตเฉพาะบางช่วงวันที่** (เช่น รันทุกคืนสำหรับข้อมูลของวันนั้น) โดยไม่ต้อง refresh ข้อมูลทั้งหมดเหมือน Materialized View — ประหยัดเวลาและ I/O มากในระบบที่มีข้อมูลสะสมหลายปี

### 600.4 Partition-ready Design

ตาราง `orders` ในระบบจริงจะโตเร็วมาก (append-heavy ตามเวลา) — เป็นตัวเลือกที่ดีสำหรับ **Range Partitioning ตามเดือน** เราจะออกแบบโครงสร้างที่ partition-ready ไว้ล่วงหน้า

```sql
-- สร้างตารางเวอร์ชัน partitioned เพื่อสาธิตแนวทางการ migrate ในอนาคต
-- (แนวทางจริง: สร้างตารางใหม่ -> ย้ายข้อมูล -> สลับชื่อ ในช่วง maintenance window)
CREATE TABLE orders_partitioned (
    order_id    BIGINT NOT NULL,
    customer_id INTEGER REFERENCES customers(customer_id),
    order_date  TIMESTAMPTZ NOT NULL DEFAULT now(),
    status      VARCHAR(20) NOT NULL DEFAULT 'pending',
    PRIMARY KEY (order_id, order_date)
) PARTITION BY RANGE (order_date);

-- สร้างพาร์ทิชันรายเดือนสำหรับปีปัจจุบันและปีก่อนหน้า
DO $$
DECLARE
    v_month DATE;
    v_partition_name TEXT;
BEGIN
    FOR v_month IN
        SELECT generate_series(date_trunc('month', now() - interval '12 months'),
                                date_trunc('month', now() + interval '1 month'),
                                interval '1 month')::date
    LOOP
        v_partition_name := 'orders_y' || to_char(v_month, 'YYYY') || 'm' || to_char(v_month, 'MM');
        EXECUTE format(
            'CREATE TABLE IF NOT EXISTS %I PARTITION OF orders_partitioned
             FOR VALUES FROM (%L) TO (%L)',
            v_partition_name, v_month, v_month + interval '1 month'
        );
    END LOOP;
END $$;

-- พาร์ทิชันสำรองสำหรับข้อมูลที่หลุดช่วง (กันข้อผิดพลาด)
CREATE TABLE orders_default PARTITION OF orders_partitioned DEFAULT;

-- ตรวจสอบพาร์ทิชันที่สร้าง
SELECT inhrelid::regclass AS partition_name,
       pg_get_expr(relpartbound, inhrelid) AS partition_range
FROM pg_inherits
JOIN pg_class ON pg_class.oid = inhrelid
WHERE inhparent = 'orders_partitioned'::regclass
ORDER BY partition_name;
```

```
     partition_name      |                          partition_range
--------------------------+---------------------------------------------------------------------
 orders_default           | DEFAULT
 orders_y2025m10           | FOR VALUES FROM ('2025-10-01 00:00:00+00') TO ('2025-11-01 00:00:00+00')
 orders_y2025m11           | FOR VALUES FROM ('2025-11-01 00:00:00+00') TO ('2025-12-01 00:00:00+00')
 ...
```

**ประโยชน์ของ partition-ready design ในบริบทของบทนี้**:

- Query ที่กรองด้วย `order_date` จะใช้ **partition pruning** เข้าถึงเฉพาะพาร์ทิชันที่เกี่ยวข้อง ลดปริมาณข้อมูลที่ต้องสแกนและ vacuum ต่อครั้ง
- **VACUUM ทำงานต่อพาร์ทิชัน** แยกกัน — พาร์ทิชันเดือนเก่าที่ไม่มีการเขียนแล้วแทบไม่ต้อง vacuum ซ้ำเลย ในขณะที่พาร์ทิชันเดือนปัจจุบันซึ่งมี churn สูงจะถูก vacuum บ่อย — ช่วยกระจายภาระของ autovacuum ได้อย่างมีประสิทธิภาพกว่าตารางก้อนใหญ่ก้อนเดียว
- การลบข้อมูลเก่า (data retention) ทำได้เร็วมากด้วย `DROP TABLE orders_y2024m01` แทนการ `DELETE` ที่สร้าง dead tuple มหาศาล

```sql
-- ตั้งค่า autovacuum ให้ต่างกันตามพาร์ทิชัน: เดือนปัจจุบัน churn สูง ต้อง vacuum ถี่
ALTER TABLE orders_y2026m09 SET (
    autovacuum_vacuum_scale_factor = 0.02,
    autovacuum_vacuum_cost_delay = 0
);
```

### 600.5 PL/pgSQL Function สำหรับรายงาน

```sql
CREATE OR REPLACE FUNCTION get_sales_report(
    p_start_date DATE,
    p_end_date   DATE
)
RETURNS TABLE (
    category_name   VARCHAR,
    total_orders    BIGINT,
    total_units     BIGINT,
    total_revenue   NUMERIC,
    avg_order_value NUMERIC,
    avg_rating      NUMERIC
) AS $$
BEGIN
    IF p_start_date > p_end_date THEN
        RAISE EXCEPTION 'start_date (%) ต้องไม่มากกว่า end_date (%)', p_start_date, p_end_date;
    END IF;

    RETURN QUERY
    SELECT
        c.category_name,
        count(DISTINCT o.order_id)                          AS total_orders,
        sum(oi.quantity)::BIGINT                             AS total_units,
        sum(oi.quantity * oi.unit_price)                     AS total_revenue,
        round(sum(oi.quantity * oi.unit_price) /
              NULLIF(count(DISTINCT o.order_id), 0), 2)      AS avg_order_value,
        round(avg(r.rating), 2)                              AS avg_rating
    FROM orders o
    JOIN order_items oi ON oi.order_id = o.order_id
    JOIN products p      ON p.product_id = oi.product_id
    JOIN categories c    ON c.category_id = p.category_id
    LEFT JOIN reviews r  ON r.product_id = p.product_id
    WHERE o.order_date::date BETWEEN p_start_date AND p_end_date
      AND o.status IN ('paid','shipped','delivered')
    GROUP BY c.category_name
    ORDER BY total_revenue DESC;
END;
$$ LANGUAGE plpgsql STABLE;

SELECT * FROM get_sales_report(current_date - 30, current_date);
```

```
 category_name | total_orders | total_units | total_revenue | avg_order_value | avg_rating
----------------+---------------+-------------+-----------------+-------------------+-------------
 Category 3     |           420 |        1150 |       210500.00 |             501.19 |        3.42
 Category 11    |           388 |        1020 |       198320.50 |             511.13 |        3.51
 ...
```

ฟังก์ชันสำหรับ Top Products (ใช้ window function ที่เรียนมาในระดับ Intermediate ผสมกับเทคนิค Advanced):

```sql
CREATE OR REPLACE FUNCTION get_top_products(p_limit INTEGER DEFAULT 10)
RETURNS TABLE (
    product_id    INTEGER,
    product_name  VARCHAR,
    total_revenue NUMERIC,
    revenue_rank  BIGINT
) AS $$
    SELECT
        p.product_id,
        p.product_name,
        sum(oi.quantity * oi.unit_price) AS total_revenue,
        rank() OVER (ORDER BY sum(oi.quantity * oi.unit_price) DESC) AS revenue_rank
    FROM products p
    JOIN order_items oi ON oi.product_id = p.product_id
    JOIN orders o        ON o.order_id = oi.order_id
    WHERE o.status IN ('paid','shipped','delivered')
    GROUP BY p.product_id, p.product_name
    ORDER BY total_revenue DESC
    LIMIT p_limit;
$$ LANGUAGE sql STABLE;

SELECT * FROM get_top_products(5);
```

### 600.6 Trigger สำหรับ Audit

บันทึกทุกครั้งที่ราคาสินค้าเปลี่ยนแปลง เพื่อการตรวจสอบย้อนหลัง:

```sql
CREATE TABLE product_price_audit (
    audit_id     BIGSERIAL PRIMARY KEY,
    product_id   INTEGER NOT NULL,
    old_price    NUMERIC(10,2),
    new_price    NUMERIC(10,2),
    changed_by   TEXT NOT NULL DEFAULT current_user,
    changed_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_price_audit_product_id ON product_price_audit (product_id);

CREATE OR REPLACE FUNCTION fn_audit_product_price()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'UPDATE' AND OLD.unit_price IS DISTINCT FROM NEW.unit_price THEN
        INSERT INTO product_price_audit (product_id, old_price, new_price, changed_by)
        VALUES (NEW.product_id, OLD.unit_price, NEW.unit_price, current_user);
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_audit_product_price
    AFTER UPDATE ON products
    FOR EACH ROW
    EXECUTE FUNCTION fn_audit_product_price();

-- ทดสอบ
UPDATE products SET unit_price = unit_price * 1.10 WHERE product_id = 1;

SELECT * FROM product_price_audit WHERE product_id = 1;
```

```
 audit_id | product_id | old_price | new_price | changed_by |          changed_at
----------+------------+-----------+-----------+------------+-------------------------------
        1 |          1 |    899.00 |    988.90 | app_user   | 2026-09-25 11:02:14.221+00
```

> **ข้อควรระวังเรื่อง VACUUM**: ตาราง audit แบบนี้จะ**โตขึ้นเรื่อยๆ แบบ insert-only** ไม่มี UPDATE/DELETE จึงแทบไม่มี dead tuple — แต่ยังคงต้อง **ANALYZE** เป็นระยะเพื่อให้สถิติทันสมัย (สำหรับ query ที่ filter ตาม `changed_at`) และยังต้อง **VACUUM เพื่อ freeze** ป้องกัน wraparound (Step 597) แม้จะไม่มี dead tuple ให้ลบก็ตาม

### 600.7 แผน VACUUM/Monitoring แบบครบวงจรสำหรับระบบนี้

สรุปการตั้งค่า per-table ทั้งหมดของระบบ Analytics Dashboard:

```sql
-- ตารางที่ churn สูง: vacuum ถี่ ไม่ delay
ALTER TABLE orders SET (
    autovacuum_vacuum_scale_factor = 0.05,
    autovacuum_vacuum_threshold    = 200,
    autovacuum_vacuum_cost_delay   = 0
);

ALTER TABLE order_items SET (
    autovacuum_vacuum_scale_factor = 0.02,
    autovacuum_vacuum_threshold    = 500
);

ALTER TABLE products SET (
    autovacuum_vacuum_scale_factor = 0.05,
    autovacuum_vacuum_threshold    = 100,
    fillfactor                     = 90
);

-- ตาราง summary: อัปเดตเป็นรอบ (batch) ผ่านฟังก์ชัน ไม่ใช่ต่อแถว จึงตั้งค่าปกติได้
ALTER TABLE dashboard_summary SET (
    autovacuum_vacuum_scale_factor = 0.1
);

-- ตาราง audit แบบ append-only: เน้น analyze สม่ำเสมอ vacuum ไม่บ่อยนักแต่ต้องมี
ALTER TABLE product_price_audit SET (
    autovacuum_vacuum_scale_factor  = 0.2,
    autovacuum_analyze_scale_factor = 0.05
);
```

Dashboard query สำหรับทีม DBA ตรวจสุขภาพระบบทั้งหมดในหน้าเดียว:

```sql
CREATE OR REPLACE VIEW v_vacuum_health_dashboard AS
SELECT
    st.schemaname,
    st.relname,
    st.n_live_tup,
    st.n_dead_tup,
    round(100.0 * st.n_dead_tup / GREATEST(st.n_live_tup + st.n_dead_tup, 1), 2) AS dead_pct,
    coalesce(st.last_autovacuum, st.last_vacuum)   AS last_vacuumed,
    coalesce(st.last_autoanalyze, st.last_analyze) AS last_analyzed,
    age(c.relfrozenxid)                            AS xid_age,
    round(100.0 * age(c.relfrozenxid) /
          current_setting('autovacuum_freeze_max_age')::bigint, 2) AS xid_pct_toward_wraparound,
    pg_size_pretty(pg_total_relation_size(st.relid)) AS total_size
FROM pg_stat_user_tables st
JOIN pg_class c ON c.oid = st.relid
ORDER BY dead_pct DESC;

SELECT * FROM v_vacuum_health_dashboard LIMIT 10;
```

```
 schemaname |     relname       | n_live_tup | n_dead_tup | dead_pct |    last_vacuumed     | xid_age | xid_pct_toward_wraparound | total_size
------------+--------------------+------------+------------+----------+------------------------+---------+-----------------------------+------------
 public     | products           |       8000 |         42 |     0.52 | 2026-09-25 11:15:00   |    9945 |                        0.00 | 1408 kB
 public     | orders             |      19850 |         88 |     0.44 | 2026-09-25 11:14:02   |   18560 |                        0.01 | 3160 kB
 public     | order_items        |      54200 |         12 |     0.02 | 2026-09-25 11:16:10   |   12240 |                        0.01 | 6712 kB
```

โปรเจกต์นี้แสดงให้เห็นว่า **VACUUM/Autovacuum ไม่ใช่หัวข้อแยกต่างหาก** แต่เป็น**ส่วนหนึ่งของการออกแบบระบบทุกจุด** — ตั้งแต่การเลือก index (ยิ่ง index เยอะ ยิ่งต้อง vacuum index มากขึ้นด้วย), การออกแบบ partition (ช่วยกระจายภาระ vacuum), ไปจนถึงการออกแบบตาราง audit/summary (แต่ละแบบมีรูปแบบ churn ต่างกัน จึงต้องปรับ autovacuum ต่างกัน)

---

## สรุปหลักสูตรระดับสูง — ทบทวน Part 041-059

ก่อนจะข้ามไประดับ Professional เรามาทบทวนภาพรวมทั้งหมดของระดับ Advanced กัน:

| ช่วง Part | หัวข้อหลัก | นำมาใช้ในโปรเจกต์ Step 600 อย่างไร |
|---|---|---|
| **041-043** | B-Tree Index, Other Index Types (GIN/GiST/BRIN/Hash), Partial & Expression Index | ใช้ B-Tree สำหรับ FK/sort, GIN สำหรับ JSONB และ Full Text Search, BRIN สำหรับ `order_date`, Partial Index สำหรับออเดอร์ active |
| **044-045** | Query Planner, EXPLAIN, Query Optimization | ใช้ `EXPLAIN (ANALYZE, BUFFERS)` ตรวจสอบ index usage ตลอดบท และเข้าใจผลกระทบของสถิติล้าสมัยต่อแผน query |
| **046-047** | PL/pgSQL พื้นฐานและขั้นสูง | สร้างฟังก์ชัน `get_sales_report`, `refresh_dashboard_summary`, `search_reviews` |
| **048** | Triggers | สร้าง `trg_audit_product_price` สำหรับ audit การเปลี่ยนราคา |
| **049** | Custom Types & Domains | (แนวคิดที่นำไปประยุกต์ต่อได้ เช่นกำหนด domain สำหรับ `unit_price` ที่ต้อง > 0) |
| **050** | Arrays | ใช้ใน JSONB attributes (`tags` array) และสามารถขยายเป็น native array column ได้ |
| **051-054** (สันนิษฐาน) | JSONB และ Semi-structured Data | ออกแบบคอลัมน์ `attributes JSONB`, ใช้ `jsonb_build_object`, GIN index `jsonb_path_ops` |
| **055-057** (สันนิษฐาน) | Full Text Search | สร้าง `tsvector` generated column, GIN index, `ts_rank`, `plainto_tsquery` |
| **058** | MVCC และ Transaction Isolation | รากฐานของทั้งบทนี้ — dead tuple, xmin/xmax, snapshot visibility |
| **059** (สันนิษฐาน) | Table Partitioning | สร้าง `orders_partitioned` แบบ RANGE partition ตามเดือน พร้อม partition pruning |
| **060 (บทนี้)** | VACUUM, Autovacuum, Bloat Management | เชื่อมทุกอย่างเข้าด้วยกันเป็นระบบ Analytics Dashboard ที่ดูแลตัวเองได้ระยะยาว |

### สิ่งที่ผู้เรียนควรทำได้แล้วในตอนนี้

- อ่านและตีความผลลัพธ์ `EXPLAIN ANALYZE` เพื่อวินิจฉัยปัญหา performance
- เลือกประเภท index ที่เหมาะกับรูปแบบข้อมูลและ query pattern
- เขียน PL/pgSQL function และ trigger เพื่อ encapsulate business logic ในฐานข้อมูล
- ออกแบบสคีมาที่รองรับข้อมูลกึ่งโครงสร้าง (JSONB) และการค้นหาข้อความ (Full Text Search)
- ออกแบบตารางขนาดใหญ่ให้พร้อมสำหรับการ partition
- เข้าใจกลไก MVCC อย่างลึกซึ้งและผลกระทบต่อ dead tuple/bloat
- ดูแลรักษาฐานข้อมูลระยะยาวด้วย VACUUM/Autovacuum อย่างเป็นระบบ

---

## สรุปท้ายบท

VACUUM คือกลไกที่มองไม่เห็นแต่**สำคัญที่สุด**อย่างหนึ่งในการรัน PostgreSQL ระยะยาว มันคือผลพวงโดยตรงจากการออกแบบ MVCC ที่ทำให้ PostgreSQL รองรับ concurrency ได้ดีเยี่ยม แต่ก็แลกมาด้วยภาระในการเก็บกวาด dead tuple

สิ่งที่ควรจำจากบทนี้:

1. **VACUUM ไม่ใช่ตัวเลือก แต่เป็นความจำเป็น** — ไม่มี VACUUM เท่ากับฐานข้อมูลจะบวมไม่มีที่สิ้นสุดและสุดท้ายจะเจอ wraparound
2. **VACUUM ธรรมดา** เก็บพื้นที่ไว้ใช้ซ้ำ ไม่บล็อกการใช้งาน — ควรปล่อยให้ autovacuum จัดการเป็นหลัก
3. **VACUUM FULL** คืนพื้นที่จริงแต่ล็อกตารางเต็มรูปแบบ — ใช้เฉพาะ maintenance window
4. **Autovacuum default settings** เหมาะกับตารางทั่วไป แต่ตารางใหญ่/churn สูงต้องปรับ `scale_factor` ต่อตารางเสมอ
5. **สถิติที่ล้าสมัย** (ไม่ ANALYZE) ทำให้ query planner ตัดสินใจผิด แม้ index จะถูกต้องก็ตาม
6. **Transaction ID Wraparound** คือความเสี่ยงร้ายแรงที่สุดถ้า autovacuum ถูกปิดหรือตามไม่ทัน — ต้อง monitor `age(relfrozenxid)` เป็นประจำ
7. **Monitoring อย่างสม่ำเสมอ** ผ่าน `pg_stat_user_tables`, `pg_stat_progress_vacuum` และ log คือกุญแจสำคัญในการจับปัญหาก่อนที่มันจะกลายเป็นวิกฤต
8. โปรเจกต์ Analytics Dashboard แสดงให้เห็นว่าการดูแล VACUUM ต้องคิดตั้งแต่**ขั้นตอนออกแบบสคีมา** ไม่ใช่แค่เรื่องที่มาแก้ทีหลัง

ระดับ **Advanced (Part 041-060)** ของหลักสูตรนี้จบลงแล้ว — ผู้เรียนได้ผ่านการฝึกฝนตั้งแต่ index ขั้นสูง, การปรับแต่ง query, PL/pgSQL, JSONB, Full Text Search, Partitioning, MVCC ไปจนถึงการดูแลรักษาฐานข้อมูลระยะยาวด้วย VACUUM ครบทุกมิติที่จำเป็นสำหรับการเป็น PostgreSQL Developer ระดับสูง

ขั้นต่อไปคือระดับ **Professional** ซึ่งจะพาไปสู่หัวข้อระดับ DBA/Infrastructure เช่น Backup & Recovery, Replication, High Availability และการดูแลระบบในสภาพแวดล้อม production จริง

---

## แบบฝึกหัด

<details>
<summary><strong>แบบฝึกหัดที่ 1</strong>: อธิบายว่าทำไม `UPDATE` หนึ่งครั้งบนแถวที่มี column ถูก index อยู่ 3 index จึงอาจทำให้เกิด "dead tuple" ใน index ทั้ง 3 ตัวด้วย ไม่ใช่แค่ใน heap table</summary>

**เฉลย**:

เมื่อ `UPDATE` แถวที่ไม่ใช่ HOT update (เช่น มีการแก้ไขคอลัมน์ที่ถูก index อย่างน้อย 1 ตัว หรือไม่มีพื้นที่ว่างเหลือใน page เดิมสำหรับ HOT) PostgreSQL จะสร้าง **tuple version ใหม่** ในตำแหน่งใหม่ (ctid ใหม่) ซึ่งหมายความว่า **ทุก index ที่มีอยู่บนตารางนั้นต้องมี entry ใหม่ชี้ไปยัง ctid ใหม่นี้ด้วย** ในขณะที่ entry เก่าใน index ที่ชี้ไปยัง ctid เดิม (ที่ตอนนี้กลายเป็น dead tuple ใน heap) ก็ยังคงค้างอยู่จนกว่า VACUUM จะมาลบออก

ดังนั้น UPDATE 1 ครั้งที่ไม่ใช่ HOT update บนตารางที่มี 3 index จะสร้าง "ขยะ" ทั้งใน heap (1 dead tuple) และใน index ทั้ง 3 ตัว (index entry เก่าที่ชี้ไปยัง dead tuple) รวมเป็น 4 จุดที่ต้องถูก vacuum — นี่คือเหตุผลที่ตารางที่มี index เยอะจะมี **vacuum overhead สูงกว่า** ตารางที่มี index น้อย และเป็นเหตุผลหนึ่งที่ไม่ควรสร้าง index เกินความจำเป็น
</details>

<details>
<summary><strong>แบบฝึกหัดที่ 2</strong>: เขียน query เพื่อหา 5 ตารางที่มี `dead_pct` สูงที่สุดในฐานข้อมูล พร้อมแสดงว่าตารางไหนที่ `autovacuum_enabled` ถูกปิดไว้อย่างชัดเจน (ผ่าน reloptions)</summary>

**เฉลย**:

```sql
SELECT
    st.relname,
    st.n_live_tup,
    st.n_dead_tup,
    round(100.0 * st.n_dead_tup / GREATEST(st.n_live_tup + st.n_dead_tup, 1), 2) AS dead_pct,
    c.reloptions,
    CASE
        WHEN c.reloptions::text LIKE '%autovacuum_enabled=false%' THEN 'ปิด autovacuum'
        ELSE 'เปิด (หรือใช้ default)'
    END AS autovacuum_status
FROM pg_stat_user_tables st
JOIN pg_class c ON c.oid = st.relid
ORDER BY dead_pct DESC
LIMIT 5;
```

Query นี้ join `pg_stat_user_tables` กับ `pg_class` เพื่อดึงทั้งสถิติ dead tuple และการตั้งค่า `reloptions` มาพร้อมกัน ช่วยให้เห็นได้ทันทีว่าตารางที่ bloat สูงนั้นเป็นเพราะถูกปิด autovacuum ไว้โดยตั้งใจ (เหมือนที่เราทำกับ `products` ในตอนต้นบท) หรือเป็นเพราะ churn สูงเกินกว่า default settings จะตามทัน
</details>

<details>
<summary><strong>แบบฝึกหัดที่ 3</strong>: จำลองสถานการณ์ที่สถิติ (statistics) ล้าสมัยทำให้ query planner เลือกแผนผิด โดยใช้ตาราง `order_items` — ให้เพิ่มข้อมูลจำนวนมากโดยไม่ ANALYZE แล้วเปรียบเทียบ EXPLAIN ก่อน-หลัง ANALYZE</summary>

**เฉลย**:

```sql
-- ขั้นที่ 1: เพิ่มออเดอร์และ order_items จำนวนมากสำหรับสินค้าชนิดเดียว (product_id = 1)
BEGIN;
INSERT INTO order_items (order_id, product_id, quantity, unit_price)
SELECT (1 + floor(random() * 20000))::int, 1, 1, 500.00
FROM generate_series(1, 8000);
COMMIT;

-- ขั้นที่ 2: ดูแผนก่อน ANALYZE (planner ยังใช้สถิติเก่า อาจประเมิน rows ต่ำเกินจริง)
EXPLAIN SELECT * FROM order_items WHERE product_id = 1;

-- ขั้นที่ 3: ANALYZE แล้วเปรียบเทียบ
ANALYZE order_items;
EXPLAIN SELECT * FROM order_items WHERE product_id = 1;
```

ก่อน ANALYZE planner จะประเมิน `rows` ต่ำกว่าความเป็นจริงมาก (เพราะสถิติเดิมคิดว่า product_id กระจายตัวสม่ำเสมอ ไม่รู้ว่าตอนนี้มี 8,000 แถวกระจุกอยู่ที่ product_id=1) และอาจเลือก **Index Scan** ทั้งที่จริงควรเป็น **Sequential Scan หรือ Bitmap Scan** เพราะสัดส่วนของแถวที่ตรงเงื่อนไขสูงเกินจุดคุ้มทุนของ index scan แล้ว หลัง ANALYZE, planner จะเห็นค่า `n_distinct` และ histogram ที่อัปเดตแล้ว และเลือกแผนที่เหมาะสมกว่า
</details>

<details>
<summary><strong>แบบฝึกหัดที่ 4</strong>: เขียนฟังก์ชัน PL/pgSQL ชื่อ `check_bloat_alert()` ที่คืนค่าตารางทั้งหมดที่ `dead_pct > 20` พร้อมข้อความแนะนำ (เช่น "ควรตรวจสอบ autovacuum settings")</summary>

**เฉลย**:

```sql
CREATE OR REPLACE FUNCTION check_bloat_alert(p_threshold NUMERIC DEFAULT 20.0)
RETURNS TABLE (
    table_name TEXT,
    dead_pct   NUMERIC,
    n_dead_tup BIGINT,
    recommendation TEXT
) AS $$
BEGIN
    RETURN QUERY
    SELECT
        st.relname::TEXT,
        round(100.0 * st.n_dead_tup / GREATEST(st.n_live_tup + st.n_dead_tup, 1), 2),
        st.n_dead_tup,
        CASE
            WHEN st.last_autovacuum IS NULL AND st.last_vacuum IS NULL
                THEN 'ตารางนี้ไม่เคยถูก vacuum เลย ตรวจสอบ autovacuum_enabled'
            WHEN now() - coalesce(st.last_autovacuum, st.last_vacuum) > interval '7 days'
                THEN 'ไม่ถูก vacuum มานานกว่า 7 วัน ควรตรวจสอบ scale_factor'
            ELSE 'ควร vacuum ด้วยตนเองทันทีและพิจารณาลด scale_factor'
        END
    FROM pg_stat_user_tables st
    WHERE (st.n_live_tup + st.n_dead_tup) > 0
      AND 100.0 * st.n_dead_tup / GREATEST(st.n_live_tup + st.n_dead_tup, 1) > p_threshold
    ORDER BY dead_pct DESC;
END;
$$ LANGUAGE plpgsql STABLE;

SELECT * FROM check_bloat_alert(15.0);
```

ฟังก์ชันนี้สามารถนำไปเรียกจาก cron job หรือ monitoring system ภายนอกเพื่อแจ้งเตือนทีม DBA โดยอัตโนมัติเมื่อพบตารางที่มีปัญหา bloat เกินเกณฑ์ที่กำหนด
</details>

<details>
<summary><strong>แบบฝึกหัดที่ 5</strong>: ตาราง `dashboard_summary` ที่สร้างในโปรเจกต์ ควรตั้งค่า autovacuum อย่างไรถ้าฟังก์ชัน `refresh_dashboard_summary()` ถูกเรียกทุกชั่วโมงและทับข้อมูล (upsert) ประมาณ 500 แถวต่อครั้ง จากทั้งหมด ~5,000 แถวในตาราง</summary>

**เฉลย**:

การ upsert 500 แถวจาก 5,000 แถว ทุกชั่วโมง หมายความว่าใน 1 วันจะมี dead tuple สะสมประมาณ 500 × 24 = 12,000 แถว ซึ่งมากกว่าจำนวนแถวทั้งหมดในตารางถึง 2 เท่า! ถ้าใช้ default `scale_factor = 0.2` (threshold = 50 + 0.2×5000 = 1,050) autovacuum จะถูกกระตุ้นทุกๆ ~2 ชั่วโมง (1,050/500 ≈ 2.1 รอบ) ซึ่งพอรับได้ แต่ยังปล่อยให้ bloat สะสมได้พอสมควรก่อนจะถูกเก็บกวาด

แนวทางที่ดีกว่า:

```sql
ALTER TABLE dashboard_summary SET (
    autovacuum_vacuum_scale_factor = 0.05,   -- ให้ vacuum ทุกๆ ~250 แถว dead tuple
    autovacuum_vacuum_threshold    = 100,
    fillfactor                      = 85     -- เผื่อพื้นที่สำหรับ HOT update เพราะ upsert ซ้ำ key เดิมบ่อย
);
```

การลด `scale_factor` ลงและใช้ `fillfactor` ต่ำกว่า default ช่วยให้การ upsert ที่เกิดขึ้นทุกชั่วโมงส่วนใหญ่กลายเป็น HOT update (ถ้าไม่ได้แก้ไขคอลัมน์ที่มี index) ซึ่งลดภาระ index vacuum ลงได้มาก
</details>

<details>
<summary><strong>แบบฝึกหัดที่ 6</strong>: อธิบายว่าทำไมตาราง `product_price_audit` (append-only) ยังคงต้องถูก VACUUM แม้จะไม่มี UPDATE/DELETE เลย</summary>

**เฉลย**:

แม้ตาราง append-only จะไม่มี dead tuple จาก UPDATE/DELETE แต่ทุกแถวที่ถูก INSERT เข้ามาจะมี `xmin` เป็น transaction ID ของตอนที่ insert — เมื่อเวลาผ่านไป XID เหล่านี้จะ "เก่า" ขึ้นเรื่อยๆ เทียบกับ XID ปัจจุบันของระบบ

หาก **ไม่มีการ VACUUM เลย** แถวเหล่านี้จะไม่เคยถูก **freeze** (เปลี่ยน xmin เป็น `FrozenTransactionId`) และเมื่อ `age(relfrozenxid)` ของตารางนี้วิ่งเข้าใกล้ `autovacuum_freeze_max_age` (200 ล้าน) ตารางนี้ก็จะเสี่ยงต่อ **Transaction ID Wraparound** เหมือนตารางอื่นๆ ทุกประการ (ตามที่อธิบายใน Step 597) — ดังนั้นแม้ตารางจะไม่มี dead tuple ให้ "ลบ" เลย แต่ VACUUM ก็ยังต้องเข้ามาทำหน้าที่ **freeze tuple เก่า** เพื่อป้องกัน wraparound อยู่ดี ซึ่งเป็นเหตุผลที่ autovacuum (หรือ anti-wraparound autovacuum) จะยังคงทำงานกับตารางประเภทนี้เป็นระยะ แม้จะไม่มี dead tuple สะสมเลยก็ตาม
</details>

<details>
<summary><strong>แบบฝึกหัดที่ 7</strong>: เขียน query เปรียบเทียบขนาด (`pg_relation_size`) ของ index `idx_products_attributes_gin` ก่อนและหลังการทำ `REINDEX` เพื่อดูว่า index ก็สามารถ bloat ได้เช่นกัน</summary>

**เฉลย**:

```sql
-- สร้าง churn จำนวนมากบนคอลัมน์ attributes เพื่อจำลอง index bloat
DO $$
BEGIN
    FOR i IN 1..10 LOOP
        UPDATE products
        SET attributes = attributes || jsonb_build_object('last_touched', now()::text)
        WHERE product_id % 7 = (i % 7);
    END LOOP;
END $$;

-- ขนาด index ก่อน reindex
SELECT pg_size_pretty(pg_relation_size('idx_products_attributes_gin')) AS before_reindex;

-- Reindex (สร้าง index ใหม่ทั้งหมด กำจัด bloat ใน index)
REINDEX INDEX CONCURRENTLY idx_products_attributes_gin;

SELECT pg_size_pretty(pg_relation_size('idx_products_attributes_gin')) AS after_reindex;
```

Index ก็เกิด bloat ได้เช่นเดียวกับ heap table เพราะทุกครั้งที่แถวถูก UPDATE (แบบไม่ใช่ HOT) entry ใน index ที่ชี้ไปยัง tuple เก่าจะกลายเป็นขยะเช่นกัน `VACUUM` ปกติจะช่วยลบ index entry ที่ตายแล้วออกไปในระดับหนึ่ง แต่หากมี churn สูงต่อเนื่องเป็นเวลานาน โครงสร้างภายในของ B-Tree/GIN อาจเสีย "ความหนาแน่น" ที่ดีไป การใช้ `REINDEX CONCURRENTLY` (ไม่ล็อกตาราง) เป็นครั้งคราวช่วยให้ index กลับมามีขนาดกระชับและมีประสิทธิภาพเหมือนใหม่
</details>

<details>
<summary><strong>แบบฝึกหัดที่ 8</strong>: จากตาราง `orders_partitioned` ที่สร้างในโปรเจกต์ อธิบายว่าทำไมการลบข้อมูลเก่าด้วย `DROP TABLE orders_y2025m10` จึงดีกว่า `DELETE FROM orders_partitioned WHERE order_date < '2025-11-01'` ในแง่ของ VACUUM</summary>

**เฉลย**:

`DELETE FROM orders_partitioned WHERE order_date < '2025-11-01'` จะทำให้แถวทุกแถวที่ตรงเงื่อนไขกลายเป็น **dead tuple** ในพาร์ทิชันนั้น (เช่นอาจเป็นหลักหมื่นหรือหลักแสนแถว) ซึ่งต้องรอ autovacuum เข้ามาเก็บกวาดภายหลัง — ระหว่างนั้นพาร์ทิชันจะบวมขึ้นชั่วคราวและ autovacuum ต้องทำงานหนักเพื่อลบ dead tuple จำนวนมหาศาลนี้ (ซึ่งอาจใช้เวลานานและกิน I/O เยอะ)

ในทางกลับกัน `DROP TABLE orders_y2025m10` คือการ**ลบไฟล์ทั้งไฟล์ในระดับ catalog** — เป็นการดำเนินการที่รวดเร็วมาก (เพียงลบ metadata และ unlink ไฟล์บนดิสก์) **ไม่สร้าง dead tuple แม้แต่แถวเดียว** เพราะไม่มีการ "ลบทีละแถว" เกิดขึ้นเลย จึงไม่มีภาระใดๆ ตกไปที่ VACUUM

นี่คือเหตุผลสำคัญที่การออกแบบตารางให้ partition-ready ตั้งแต่ต้น (Step 600.4) ช่วยให้กลยุทธ์ data retention (การลบข้อมูลเก่าตามนโยบาย) ทำได้อย่างมีประสิทธิภาพและไม่กระทบ VACUUM/performance ของระบบเลย
</details>

<details>
<summary><strong>แบบฝึกหัดที่ 9</strong>: เขียน query ตรวจสอบว่ามี transaction ใดกำลัง "ถือ" ไม่ให้ VACUUM ลบ dead tuple ได้ (เช่น long-running transaction หรือ replication slot ค้าง) พร้อมอธิบายวิธีแก้ปัญหาแต่ละกรณี</summary>

**เฉลย**:

```sql
-- 1) ตรวจสอบ long-running / idle-in-transaction
SELECT pid, usename, state, now() - xact_start AS duration, query
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
  AND now() - xact_start > interval '10 minutes'
ORDER BY duration DESC;

-- 2) ตรวจสอบ replication slot ที่ inactive และเก็บ WAL ค้างไว้นาน
SELECT slot_name, slot_type, active,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS retained_wal
FROM pg_replication_slots
ORDER BY retained_wal DESC;

-- 3) ตรวจสอบ prepared transaction ที่ค้างอยู่ (two-phase commit)
SELECT gid, prepared, owner, database
FROM pg_prepared_xacts;
```

**วิธีแก้แต่ละกรณี**:

- **Long-running transaction**: ยกเลิกด้วย `SELECT pg_terminate_backend(pid);` (ระวังผลกระทบต่อ transaction นั้น) หรือป้องกันล่วงหน้าด้วย `statement_timeout` และ `idle_in_transaction_session_timeout`
- **Replication slot ค้าง**: ถ้า slot นั้นไม่ได้ใช้งานแล้วจริงๆ ให้ลบด้วย `SELECT pg_drop_replication_slot('slot_name');` — แต่ต้องตรวจสอบให้แน่ใจก่อนว่าไม่มี consumer ที่ยังต้องการ WAL เหล่านั้นอยู่
- **Prepared transaction ค้าง**: เกิดจาก two-phase commit ที่ไม่ได้ commit/rollback ให้เสร็จสมบูรณ์ — ต้อง `COMMIT PREPARED 'gid'` หรือ `ROLLBACK PREPARED 'gid'` เพื่อปลดปล่อย
</details>

<details>
<summary><strong>แบบฝึกหัดที่ 10</strong>: ออกแบบและเขียน SQL สำหรับ "extension" ของโปรเจกต์ Analytics Dashboard — เพิ่มตาราง `customer_ltv_summary` ที่เก็บมูลค่าซื้อสะสมตลอดชีพ (Lifetime Value) ของลูกค้าแต่ละคน พร้อม trigger ที่อัปเดตค่านี้ทุกครั้งที่มี order_items ใหม่เกิดขึ้น และอธิบายความเสี่ยงเรื่อง bloat ที่อาจเกิดจากการออกแบบนี้</summary>

**เฉลย**:

```sql
CREATE TABLE customer_ltv_summary (
    customer_id    INTEGER PRIMARY KEY REFERENCES customers(customer_id),
    total_orders   INTEGER NOT NULL DEFAULT 0,
    total_spent    NUMERIC(14,2) NOT NULL DEFAULT 0,
    last_order_at  TIMESTAMPTZ,
    updated_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE OR REPLACE FUNCTION fn_update_customer_ltv()
RETURNS TRIGGER AS $$
DECLARE
    v_customer_id INTEGER;
    v_line_total  NUMERIC;
BEGIN
    SELECT o.customer_id INTO v_customer_id
    FROM orders o WHERE o.order_id = NEW.order_id;

    v_line_total := NEW.quantity * NEW.unit_price;

    INSERT INTO customer_ltv_summary (customer_id, total_orders, total_spent, last_order_at, updated_at)
    VALUES (v_customer_id, 1, v_line_total, now(), now())
    ON CONFLICT (customer_id) DO UPDATE SET
        total_spent  = customer_ltv_summary.total_spent + v_line_total,
        last_order_at = now(),
        updated_at    = now();

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_update_customer_ltv
    AFTER INSERT ON order_items
    FOR EACH ROW
    EXECUTE FUNCTION fn_update_customer_ltv();
```

**ความเสี่ยงเรื่อง bloat**:

1. **`customer_ltv_summary` จะถูก UPDATE บ่อยมาก** — ทุกครั้งที่มี `order_items` ใหม่เกิดขึ้นสำหรับลูกค้าคนเดิม แถวของลูกค้านั้นใน `customer_ltv_summary` จะถูก UPDATE ซ้ำ ถ้าลูกค้า VIP สั่งซื้อหลายร้อยครั้งต่อวัน แถวเดียวนั้นจะถูก UPDATE หลายร้อยครั้ง สร้าง dead tuple สะสมเฉพาะที่ (hot row) จำนวนมาก
2. **Row-level lock contention**: การ UPDATE แถวเดียวกันพร้อมกันจากหลาย transaction (กรณีระบบมี concurrent order สูง) อาจทำให้เกิดการรอคิว (lock wait) ซึ่งไม่ใช่ปัญหา VACUUM โดยตรง แต่เป็นปัญหาข้างเคียงที่ควรพิจารณา
3. **ทางแก้**: ตั้งค่า `fillfactor` ต่ำ (เช่น 70) เพื่อเปิดโอกาสให้เกิด HOT update บ่อยขึ้น, ลด `autovacuum_vacuum_scale_factor` ให้ต่ำมากสำหรับตารางนี้ (เพราะจำนวนแถวทั้งหมดน้อยแต่ churn ต่อแถวสูงมาก), หรือพิจารณาเปลี่ยนแนวทางเป็นการคำนวณ LTV แบบ batch ผ่าน `dashboard_summary` แทนการอัปเดตแบบ real-time ต่อทุก order_item หากความสดใหม่แบบวินาทีต่อวินาทีไม่ใช่ความจำเป็นทางธุรกิจจริงๆ

```sql
ALTER TABLE customer_ltv_summary SET (
    fillfactor = 70,
    autovacuum_vacuum_scale_factor = 0.01,
    autovacuum_vacuum_threshold = 50,
    autovacuum_vacuum_cost_delay = 0
);
```

แบบฝึกหัดนี้สะท้อนให้เห็นหลักการสำคัญที่สุดของบทนี้: **ทุกการออกแบบสคีมาต้องคิดถึงผลกระทบด้าน VACUUM ควบคู่ไปด้วยเสมอ** ไม่ใช่คิดแยกกันทีหลัง
</details>

---

## ก้าวต่อไป

ยินดีด้วย คุณได้จบระดับ **Advanced** ของหลักสูตร PostgreSQL ฉบับสมบูรณ์แล้ว! ตอนนี้คุณมีพื้นฐานที่แข็งแกร่งพอที่จะออกแบบ, ปรับแต่ง, และดูแลรักษาฐานข้อมูล PostgreSQL ระดับ production ได้อย่างมั่นใจ

ระดับถัดไปคือ **Professional** ซึ่งจะเจาะลึกในมุมของ Infrastructure และ Operations ระดับสูง เริ่มต้นที่:

**[Part 061: Backup Strategies](../04-professional/part-061-backup-strategies.md)** — กลยุทธ์การสำรองข้อมูลของ PostgreSQL ตั้งแต่ `pg_dump`, `pg_basebackup`, Point-in-Time Recovery (PITR) ไปจนถึงการวางแผน Disaster Recovery สำหรับระบบ production จริง
