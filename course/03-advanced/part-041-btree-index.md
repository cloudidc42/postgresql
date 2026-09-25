# Part 041: Index พื้นฐาน — B-Tree และหลักการทำงาน

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับสูง (Advanced) | Part 041

ยินดีต้อนรับสู่**ระดับสูง (Advanced)** ของหลักสูตรนี้ หลังจากที่ผ่าน 40 part แรกซึ่งปูพื้นตั้งแต่การติดตั้ง PostgreSQL, การออกแบบตาราง, SQL พื้นฐานถึงระดับกลาง, transaction, constraint และการเขียน query ที่ซับซ้อนขึ้นมาแล้ว ถึงเวลาที่เราจะเจาะลึกเรื่องที่ทำให้ PostgreSQL "เร็ว" หรือ "ช้า" จริง ๆ นั่นคือ **Index**

Part นี้เป็น Part แรกในชุด Index ทั้งหมด (Part 041–050) โดยจะเริ่มจากรากฐานที่สุด: **B-Tree Index** ซึ่งเป็น index ประเภทที่ใช้บ่อยที่สุดและเป็นค่าเริ่มต้นของ PostgreSQL เราจะไม่ท่องจำ syntax เฉย ๆ แต่จะ**รัน EXPLAIN ANALYZE จริง**เปรียบเทียบก่อน-หลังสร้าง index บนข้อมูลขนาดใหญ่พอที่จะเห็นความแตกต่างชัดเจน

## เป้าหมายการเรียนรู้

หลังจบ Part นี้ คุณจะสามารถ:

- อธิบายได้ว่า Index คืออะไร ทำงานอย่างไร และทำไมมันถึงทำให้ query เร็วขึ้น
- อ่านผลลัพธ์ `EXPLAIN ANALYZE` และแยกแยะ **Seq Scan** กับ **Index Scan** ได้
- เข้าใจโครงสร้างข้อมูลแบบ **B-Tree** ในระดับที่เพียงพอต่อการใช้งานจริง และรู้ว่าทำไมมันเหมาะกับ operator `=`, `<`, `>`, `BETWEEN` และ `ORDER BY`
- เขียน `CREATE INDEX` ได้อย่างถูกต้อง พร้อมตั้งชื่อ index ตามธรรมเนียมที่อ่านง่าย
- แยกความแตกต่างระหว่าง **Index Scan**, **Index Only Scan** และ **Bitmap Index Scan** พร้อมรู้ว่า planner เลือกใช้แบบไหนเมื่อไหร่
- เข้าใจว่า **Unique Index** สัมพันธ์กับ `PRIMARY KEY` และ `UNIQUE` constraint อย่างไร
- ประเมิน**ต้นทุน**ของการมี index ทั้งด้านความเร็วของ `INSERT/UPDATE/DELETE` และพื้นที่ดิสก์ที่เพิ่มขึ้น
- จัดการ index ด้วย `DROP INDEX`, `REINDEX` และตรวจสอบ index ที่มีอยู่ด้วย `\di` และ `pg_indexes`
- วิเคราะห์ query ที่ช้าในระบบจริงแล้วตัดสินใจได้ว่าควรสร้าง index ที่คอลัมน์ไหน

---

## เตรียมข้อมูล

เราจะยังใช้ schema ฐาน e-commerce เดิมจาก Part 021–040 แต่ครั้งนี้จะ**เพิ่มปริมาณข้อมูลใน `products` และ `orders` ให้มากขึ้นอย่างมีนัยสำคัญ** (`products` ~1,000 แถว, `orders` ~2,000 แถว, `order_items` ~5,000 แถว) เพราะบนตารางที่มีข้อมูลน้อยมาก (10–20 แถว) PostgreSQL planner มักจะเลือก Sequential Scan อยู่ดีแม้จะมี index ให้ใช้ เนื่องจากการอ่านทั้งตารางที่เล็กมากนั้น "ถูก" กว่าการเปิด index อยู่แล้ว — เราจึงต้องมีข้อมูลระดับหลักพันแถวขึ้นไปเพื่อให้เห็นความต่างของแผนการ query อย่างชัดเจนใน `EXPLAIN ANALYZE`

สร้างฐานข้อมูลใหม่สำหรับ Part นี้ (หรือใช้ฐานข้อมูลเดิมที่ตั้งไว้ตั้งแต่ต้นหลักสูตรก็ได้ แต่ต้อง `DROP` ตารางเก่าก่อน):

```sql
-- ตารางอ้างอิงขนาดเล็ก
CREATE TABLE categories (
    category_id SERIAL PRIMARY KEY,
    category_name VARCHAR(100) NOT NULL,
    parent_category_id INTEGER REFERENCES categories(category_id)
);

CREATE TABLE suppliers (
    supplier_id SERIAL PRIMARY KEY,
    supplier_name VARCHAR(150) NOT NULL,
    country VARCHAR(60)
);

-- ตารางหลักที่จะมีข้อมูลจำนวนมาก
CREATE TABLE products (
    product_id SERIAL PRIMARY KEY,
    product_name VARCHAR(150) NOT NULL,
    category_id INTEGER REFERENCES categories(category_id),
    supplier_id INTEGER REFERENCES suppliers(supplier_id),
    unit_price NUMERIC(10,2) NOT NULL CHECK (unit_price >= 0),
    stock_quantity INTEGER NOT NULL DEFAULT 0,
    is_active BOOLEAN NOT NULL DEFAULT true
);

CREATE TABLE customers (
    customer_id SERIAL PRIMARY KEY,
    first_name VARCHAR(60) NOT NULL,
    last_name VARCHAR(60) NOT NULL,
    email VARCHAR(150) UNIQUE,
    country VARCHAR(60),
    signup_date DATE NOT NULL DEFAULT CURRENT_DATE
);

CREATE TABLE orders (
    order_id SERIAL PRIMARY KEY,
    customer_id INTEGER REFERENCES customers(customer_id),
    order_date TIMESTAMPTZ NOT NULL DEFAULT now(),
    status VARCHAR(20) NOT NULL DEFAULT 'pending',
    ship_country VARCHAR(60)
);

CREATE TABLE order_items (
    order_item_id SERIAL PRIMARY KEY,
    order_id INTEGER REFERENCES orders(order_id),
    product_id INTEGER REFERENCES products(product_id),
    quantity INTEGER NOT NULL CHECK (quantity > 0),
    unit_price NUMERIC(10,2) NOT NULL
);
```

ข้อมูลอ้างอิงขนาดเล็ก (categories, suppliers, customers) ใส่ตรง ๆ เหมือนเดิม:

```sql
-- categories (~15 แถว)
INSERT INTO categories (category_name, parent_category_id) VALUES
('Electronics', NULL),
('Computers', 1),
('Mobile Phones', 1),
('Home Appliances', NULL),
('Kitchen', 4),
('Furniture', NULL),
('Office Furniture', 6),
('Fashion', NULL),
('Men Clothing', 8),
('Women Clothing', 8),
('Sports', NULL),
('Outdoor', 11),
('Books', NULL),
('Toys', NULL),
('Beauty', NULL);

-- suppliers (~12 แถว)
INSERT INTO suppliers (supplier_name, country) VALUES
('Bangkok Trading Co.', 'Thailand'),
('Siam Electronics Supply', 'Thailand'),
('Global Gadgets Ltd.', 'China'),
('Nordic Home Co.', 'Sweden'),
('Sakura Import', 'Japan'),
('Seoul Tech Partners', 'South Korea'),
('EuroStyle Furniture', 'Germany'),
('Pacific Sports Inc.', 'Vietnam'),
('Golden Textile', 'Thailand'),
('American Basics LLC', 'USA'),
('India Craft Traders', 'India'),
('Nusantara Goods', 'Indonesia');

-- customers (~20 แถว)
INSERT INTO customers (first_name, last_name, email, country, signup_date) VALUES
('Somchai', 'Jaidee', 'somchai.j@example.com', 'Thailand', '2023-01-15'),
('Suda', 'Kaewkla', 'suda.k@example.com', 'Thailand', '2023-02-20'),
('Anong', 'Saetang', 'anong.s@example.com', 'Thailand', '2023-03-05'),
('Wichai', 'Boonmee', 'wichai.b@example.com', 'Thailand', '2023-03-18'),
('Malee', 'Sukjai', 'malee.s@example.com', 'Thailand', '2023-04-02'),
('John', 'Smith', 'john.smith@example.com', 'USA', '2023-04-25'),
('Emily', 'Johnson', 'emily.j@example.com', 'USA', '2023-05-10'),
('Liu', 'Wei', 'liu.wei@example.com', 'China', '2023-05-28'),
('Tanaka', 'Hiro', 'tanaka.h@example.com', 'Japan', '2023-06-14'),
('Kim', 'Minsu', 'kim.m@example.com', 'South Korea', '2023-07-01'),
('Nguyen', 'Van', 'nguyen.v@example.com', 'Vietnam', '2023-07-19'),
('Somsak', 'Chaiyaporn', 'somsak.c@example.com', 'Thailand', '2023-08-08'),
('Piyada', 'Rungrueang', 'piyada.r@example.com', 'Thailand', '2023-08-30'),
('Michael', 'Brown', 'michael.b@example.com', 'USA', '2023-09-11'),
('Sophie', 'Davis', 'sophie.d@example.com', 'UK', '2023-10-02'),
('Ahmad', 'Rahman', 'ahmad.r@example.com', 'Indonesia', '2023-10-20'),
('Ravi', 'Kumar', 'ravi.k@example.com', 'India', '2023-11-05'),
('Nattapong', 'Srisuk', 'nattapong.s@example.com', 'Thailand', '2023-11-22'),
('Chalida', 'Wongsa', 'chalida.w@example.com', 'Thailand', '2023-12-10'),
('Peter', 'Müller', 'peter.m@example.com', 'Germany', '2024-01-05');
```

ทีนี้ถึงส่วนสำคัญ — ใช้ `generate_series` เพื่อสร้างข้อมูลจำนวนมากแบบสุ่มแต่สมจริง:

```sql
-- products (~1,000 แถว) สุ่มราคา, หมวดหมู่, ผู้จัดจำหน่าย, สต็อก
INSERT INTO products (product_name, category_id, supplier_id, unit_price, stock_quantity, is_active)
SELECT
    'Product #' || g.id,
    (1 + floor(random() * 15))::INT,      -- category_id สุ่มจาก 1-15
    (1 + floor(random() * 12))::INT,      -- supplier_id สุ่มจาก 1-12
    round((10 + random() * 4990)::NUMERIC, 2),  -- ราคาสุ่ม 10.00 - 5000.00
    floor(random() * 500)::INT,           -- สต็อกสุ่ม 0-499
    (random() < 0.9)                      -- 90% active
FROM generate_series(1, 1000) AS g(id);

-- orders (~2,000 แถว) กระจายในช่วง 2 ปีย้อนหลัง
INSERT INTO orders (customer_id, order_date, status, ship_country)
SELECT
    (1 + floor(random() * 20))::INT,      -- customer_id สุ่มจาก 1-20
    now() - (random() * 730) * interval '1 day',  -- กระจายใน 2 ปี
    (ARRAY['pending','processing','shipped','delivered','cancelled'])[1 + floor(random() * 5)],
    (ARRAY['Thailand','USA','China','Japan','South Korea','Vietnam','UK','Indonesia','India','Germany'])[1 + floor(random() * 10)]
FROM generate_series(1, 2000) AS g(id);

-- order_items (~5,000 แถว)
INSERT INTO order_items (order_id, product_id, quantity, unit_price)
SELECT
    (1 + floor(random() * 2000))::INT,    -- order_id สุ่มจาก 1-2000
    (1 + floor(random() * 1000))::INT,    -- product_id สุ่มจาก 1-1000
    (1 + floor(random() * 5))::INT,       -- จำนวนสุ่ม 1-5
    round((10 + random() * 4990)::NUMERIC, 2)
FROM generate_series(1, 5000) AS g(id);

-- อัปเดตสถิติให้ planner ตัดสินใจได้แม่นยำ (สำคัญมากหลัง bulk insert!)
ANALYZE;
```

> **หมายเหตุสำคัญ:** หลัง `INSERT` ข้อมูลจำนวนมาก ควรรัน `ANALYZE` เสมอ เพราะ PostgreSQL query planner ตัดสินใจว่าจะใช้ Seq Scan หรือ Index Scan จาก**สถิติ** (จำนวนแถว, การกระจายค่า ฯลฯ) ที่เก็บไว้ใน `pg_statistic` ถ้าไม่ `ANALYZE` หลัง insert ข้อมูลจำนวนมาก planner อาจใช้สถิติเก่าที่ผิดเพี้ยนและเลือกแผนที่ไม่เหมาะสม

ตรวจสอบจำนวนแถวที่ได้:

```sql
SELECT
    (SELECT count(*) FROM products)     AS products,
    (SELECT count(*) FROM orders)       AS orders,
    (SELECT count(*) FROM order_items)  AS order_items,
    (SELECT count(*) FROM customers)    AS customers,
    (SELECT count(*) FROM categories)   AS categories,
    (SELECT count(*) FROM suppliers)    AS suppliers;
```

ผลลัพธ์:

```
 products | orders | order_items | customers | categories | suppliers
----------+--------+-------------+-----------+------------+-----------
     1000 |   2000 |        5000 |        20 |         15 |        12
(1 row)
```

ข้อมูลชุดนี้จะถูกใช้ตลอด Part 041–050 พร้อมสำหรับการทดลองสร้าง/ลบ index และดู `EXPLAIN ANALYZE` จริงในทุก Step ต่อไปนี้

---

## Step 401: Index คืออะไร

ลองนึกภาพหนังสือเรียนหนา 1,000 หน้าที่ไม่มี**ดัชนีท้ายเล่ม (index)** ถ้าอยากหาว่าคำว่า "B-Tree" ถูกพูดถึงที่หน้าไหนบ้าง คุณต้องพลิกอ่านทีละหน้าตั้งแต่หน้า 1 จนจบเล่ม — นี่คือสิ่งที่ PostgreSQL ทำเมื่อไม่มี index เรียกว่า **Sequential Scan (Seq Scan)**: อ่านข้อมูลทุกแถวในตารางทีละแถวเพื่อตรวจสอบว่าตรงเงื่อนไข `WHERE` หรือไม่

แต่ถ้าหนังสือเล่มนั้นมีดัชนีท้ายเล่มที่เรียงตามตัวอักษรและบอกเลขหน้า คุณสามารถเปิดไปที่ตัว "B" ได้ทันทีโดยไม่ต้องอ่านทั้งเล่ม — นี่คือสิ่งที่ **Index** ทำให้ฐานข้อมูล มันเป็นโครงสร้างข้อมูลแยกต่างหากที่เก็บ "ค่าของคอลัมน์" คู่กับ "ตำแหน่งของแถวจริงในตาราง (heap)" และเรียงลำดับไว้ล่วงหน้า ทำให้ค้นหาได้เร็วโดยไม่ต้องไล่อ่านทั้งตาราง

ลองดูตัวอย่างจริงบนตาราง `products` ที่มี 1,000 แถว โดยยังไม่มี index บนคอลัมน์ `product_name` (มีแค่ index อัตโนมัติของ `PRIMARY KEY` บน `product_id`):

```sql
EXPLAIN ANALYZE
SELECT * FROM products WHERE product_name = 'Product #500';
```

ผลลัพธ์จริง:

```
                                             QUERY PLAN
----------------------------------------------------------------------------------------------------
 Seq Scan on products  (cost=0.00..22.50 rows=1 width=35) (actual time=0.046..0.084 rows=1 loops=1)
   Filter: ((product_name)::text = 'Product #500'::text)
   Rows Removed by Filter: 999
 Planning Time: 0.473 ms
 Execution Time: 0.126 ms
(5 rows)
```

สังเกตบรรทัด `Rows Removed by Filter: 999` — PostgreSQL ต้องอ่านครบทั้ง 1,000 แถวแล้ว**ทิ้ง 999 แถวที่ไม่ตรงเงื่อนไข**ไป เหลือแค่แถวเดียวที่ต้องการ นี่คือการทำงานแบบ Seq Scan: อ่านทุกแถวเสมอ ไม่ว่าจะหาแค่ 1 แถวหรือ 1,000 แถวก็ตาม บนตาราง 1,000 แถวนี้ยังเร็วมาก (0.126 ms) แต่ลองจินตนาการว่าถ้าตารางมี 10 ล้านแถว เวลาที่ใช้ก็จะเพิ่มขึ้นตามสัดส่วนแบบเชิงเส้น (linear) — นี่คือปัญหาที่ index มาช่วยแก้

สรุปหลักการ:

| แนวคิด | คำอธิบาย |
|---|---|
| **Heap (ตารางจริง)** | พื้นที่จัดเก็บแถวข้อมูลจริงบนดิสก์ ไม่ได้เรียงลำดับตามคอลัมน์ใด ๆ เป็นพิเศษ |
| **Index** | โครงสร้างข้อมูลแยกต่างหาก เก็บ (ค่าคอลัมน์, ตำแหน่งในตาราง) โดยเรียงลำดับไว้ |
| **Seq Scan** | อ่านทุกแถวใน heap ตามลำดับ เหมาะกับตารางเล็กหรือดึงข้อมูลสัดส่วนมาก |
| **Index Scan** | ค้นใน index ก่อนเพื่อหาตำแหน่งแถวที่ตรงเงื่อนไข แล้วค่อยไปอ่านแถวจริงจาก heap |

> Index ไม่ใช่ของฟรี — มันคือโครงสร้างข้อมูลเพิ่มเติมที่ PostgreSQL ต้องสร้าง เก็บ และดูแลรักษาให้ตรงกับข้อมูลในตารางตลอดเวลา รายละเอียดต้นทุนของ index จะอยู่ใน Step 408

---

## Step 402: Sequential Scan vs Index Scan — เปรียบเทียบก่อน/หลังสร้าง Index

มาดูตัวอย่างที่ชัดเจนกว่าเดิมบนตาราง `order_items` (5,000 แถว) ซึ่งยังไม่มี index บนคอลัมน์ `product_id` แม้ว่าคอลัมน์นี้จะเป็น Foreign Key ก็ตาม (ข้อควรรู้: **PostgreSQL ไม่สร้าง index ให้ Foreign Key โดยอัตโนมัติ** ต่างจาก Primary Key ที่สร้าง unique index ให้อัตโนมัติเสมอ)

**ก่อนสร้าง index:**

```sql
EXPLAIN ANALYZE
SELECT * FROM order_items WHERE product_id = 500;
```

```
                                              QUERY PLAN
-------------------------------------------------------------------------------------------------------
 Seq Scan on order_items  (cost=0.00..94.50 rows=5 width=22) (actual time=0.026..0.330 rows=1 loops=1)
   Filter: (product_id = 500)
   Rows Removed by Filter: 4999
 Planning Time: 0.296 ms
 Execution Time: 0.370 ms
(5 rows)
```

ต้องอ่านทั้ง 5,000 แถวเพื่อหาแค่ 1 แถวที่ตรงกัน (`Rows Removed by Filter: 4999`)

**สร้าง index:**

```sql
CREATE INDEX idx_order_items_product_id ON order_items(product_id);
```

**หลังสร้าง index:**

```sql
EXPLAIN ANALYZE
SELECT * FROM order_items WHERE product_id = 500;
```

```
                                                            QUERY PLAN
-----------------------------------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on order_items  (cost=4.32..18.45 rows=5 width=22) (actual time=0.031..0.031 rows=1 loops=1)
   Recheck Cond: (product_id = 500)
   Heap Blocks: exact=1
   ->  Bitmap Index Scan on idx_order_items_product_id  (cost=0.00..4.32 rows=5 width=0) (actual time=0.015..0.016 rows=1 loops=1)
         Index Cond: (product_id = 500)
 Planning Time: 0.385 ms
 Execution Time: 0.080 ms
(7 rows)
```

หมายเหตุ: บนตารางขนาดกลางแบบนี้ planner อาจเลือก **Bitmap Heap Scan** แทน Index Scan แบบตรง ๆ (เรื่องนี้จะอธิบายละเอียดใน Step 406) แต่ถ้าอยากบังคับให้เห็น Index Scan แบบพื้นฐานที่สุดเพื่อการเรียนรู้ ลองปิด bitmap scan ชั่วคราว:

```sql
SET enable_bitmapscan = off;

EXPLAIN ANALYZE
SELECT * FROM order_items WHERE product_id = 500;
```

```
                                                                QUERY PLAN
------------------------------------------------------------------------------------------------------------------------------------------
 Index Scan using idx_order_items_product_id on order_items  (cost=0.28..24.37 rows=5 width=22) (actual time=0.009..0.010 rows=1 loops=1)
   Index Cond: (product_id = 500)
 Planning Time: 0.362 ms
 Execution Time: 0.056 ms
(4 rows)
```

```sql
SET enable_bitmapscan = on;  -- อย่าลืมเปิดกลับคืน! (ค่านี้ set เฉพาะ session)
```

เปรียบเทียบทั้งสองแผน:

| | Seq Scan | Index Scan |
|---|---|---|
| จำนวนแถวที่อ่าน | ทุกแถว (5,000) | เฉพาะที่ index ชี้ไป (~1) |
| Execution Time | 0.370 ms | 0.056 ms |
| Cost (planner estimate) | 94.50 | 24.37 |
| เหมาะกับ | ดึงข้อมูลสัดส่วนมากของตาราง | ดึงข้อมูลสัดส่วนน้อย (selective) |

แม้ตัวเลขหน่วย millisecond จะดูน้อยมากในตัวอย่างนี้ (เพราะข้อมูลแค่หลักพันแถวและ PostgreSQL cache ข้อมูลไว้ใน memory ระหว่างทดสอบ) แต่ **cost ที่ planner ประเมิน (94.50 เทียบกับ 24.37)** ต่างกันชัดเจนกว่า 3 เท่า และเมื่อข้อมูลโตขึ้นเป็นล้านแถว ความต่างของเวลาจริงจะยิ่งเห็นชัดขึ้นแบบทวีคูณ เพราะ Seq Scan โตแบบ O(n) ในขณะที่ Index Scan โตแบบ O(log n) โดยประมาณ

> **วิธีอ่าน EXPLAIN ANALYZE เบื้องต้น:**
> - `cost=0.00..94.50` = ต้นทุนโดยประมาณ (startup cost..total cost) หน่วยเป็น "arbitrary unit" ไม่ใช่เวลาจริง ใช้เทียบสัดส่วนระหว่างแผนต่าง ๆ
> - `actual time=0.026..0.330` = เวลาจริงที่ใช้ (start..finish) หน่วย millisecond
> - `rows=1 loops=1` = จำนวนแถวจริงที่ได้ และจำนวนครั้งที่ node นี้ถูกเรียก
> - `Planning Time` = เวลาที่ planner ใช้วางแผน query (แยกจาก execution)
> - `Execution Time` = เวลาที่ใช้รัน query จริงหลังวางแผนเสร็จแล้ว

---

## Step 403: B-Tree คืออะไร — โครงสร้างข้อมูลแบบต้นไม้สมดุล

Index ประเภทเริ่มต้นของ PostgreSQL (และเป็นค่า default เมื่อคุณเขียน `CREATE INDEX` โดยไม่ระบุประเภท) คือ **B-Tree (Balanced Tree)**

### โครงสร้างคร่าว ๆ

B-Tree เป็นโครงสร้างข้อมูลแบบต้นไม้ที่มีคุณสมบัติสำคัญคือ **สมดุล (balanced)** — ทุก "ใบ" (leaf node) ของต้นไม้อยู่ห่างจาก "ราก" (root node) เป็นจำนวนขั้น (level) เท่ากันเสมอ ลองนึกภาพ:

```
                    [ Root Node ]
                    ค่ากลาง เช่น 500
                   /              \
          [ Node < 500 ]      [ Node >= 500 ]
             /      \             /      \
        [Leaf]   [Leaf]      [Leaf]    [Leaf]
       1..125   126..499    500..750  751..1000
```

แต่ละ node เก็บค่าคอลัมน์ที่เรียงลำดับไว้ พร้อม pointer ไปยัง node ลูกหรือ (ในกรณี leaf node) pointer ไปยังตำแหน่งแถวจริงใน heap (เรียกว่า **TID — Tuple ID**) เมื่อค้นหาค่าใดค่าหนึ่ง PostgreSQL จะเริ่มจาก root แล้วเปรียบเทียบค่าเพื่อ**ตัดครึ่ง**ไปเรื่อย ๆ จนถึง leaf node ที่มีคำตอบ — คล้ายการเล่นเกมทายตัวเลข 1-1000 ที่ทายทีละครั้งแล้วตัด "มากกว่า/น้อยกว่า" ออกครึ่งหนึ่งเสมอ ทำให้แม้ตารางจะมีเป็นล้านแถว การค้นหาก็ใช้แค่ไม่กี่ขั้น (logarithmic time: O(log n))

### ทำไม B-Tree ถึงเหมาะกับ `=`, `<`, `>`, `BETWEEN`, `ORDER BY`

เพราะข้อมูลใน B-Tree **ถูกเรียงลำดับไว้แล้ว** operator เชิงเปรียบเทียบ (comparison operators) ทุกตัวจึงใช้ประโยชน์จากลำดับนี้ได้โดยตรง:

- `=` → เดินจาก root ไปยัง leaf ที่ตรงค่าเป๊ะ ๆ
- `<`, `>`, `<=`, `>=` → เดินไปหาจุดเริ่มต้น แล้วไล่อ่านต่อเนื่องไปทางซ้ายหรือขวา (leaf nodes เชื่อมกันเป็น linked list)
- `BETWEEN` → หาจุดเริ่มต้นของช่วง แล้วไล่อ่านต่อเนื่องจนถึงจุดสิ้นสุด
- `ORDER BY` → เพราะข้อมูลเรียงอยู่แล้วใน index สามารถ**อ่านตามลำดับได้เลยโดยไม่ต้อง sort เพิ่ม**

มาดูตัวอย่างจริงที่แสดงพลังของ B-Tree กับ `BETWEEN`:

**ก่อนสร้าง index บน `order_date`:**

```sql
EXPLAIN ANALYZE
SELECT * FROM orders
WHERE order_date BETWEEN '2025-01-01' AND '2025-01-31';
```

```
                                                                      QUERY PLAN
-------------------------------------------------------------------------------------------------------------------------------------------------------
 Seq Scan on orders  (cost=0.00..46.00 rows=81 width=32) (actual time=0.009..0.187 rows=75 loops=1)
   Filter: ((order_date >= '2025-01-01 00:00:00+00'::timestamp with time zone) AND (order_date <= '2025-01-31 00:00:00+00'::timestamp with time zone))
   Rows Removed by Filter: 1925
 Planning Time: 0.340 ms
 Execution Time: 0.236 ms
(5 rows)
```

**สร้าง index แล้วลองใหม่:**

```sql
CREATE INDEX idx_orders_order_date ON orders(order_date);

EXPLAIN ANALYZE
SELECT * FROM orders
WHERE order_date BETWEEN '2025-01-01' AND '2025-01-31';
```

```
                                                                           QUERY PLAN
-----------------------------------------------------------------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on orders  (cost=5.11..22.32 rows=81 width=32) (actual time=0.037..0.088 rows=75 loops=1)
   Recheck Cond: ((order_date >= '2025-01-01 00:00:00+00'::timestamp with time zone) AND (order_date <= '2025-01-31 00:00:00+00'::timestamp with time zone))
   Heap Blocks: exact=16
   ->  Bitmap Index Scan on idx_orders_order_date  (cost=0.00..5.09 rows=81 width=0) (actual time=0.023..0.023 rows=75 loops=1)
         Index Cond: ((order_date >= '2025-01-01 00:00:00+00'::timestamp with time zone) AND (order_date <= '2025-01-31 00:00:00+00'::timestamp with time zone))
 Planning Time: 0.508 ms
 Execution Time: 0.148 ms
(7 rows)
```

cost ลดจาก 46.00 เหลือ 22.32 และไม่ต้องอ่านทั้ง 2,000 แถวอีกต่อไป

### B-Tree ช่วย `ORDER BY` ได้อย่างไร — ตัวอย่างที่ชัดที่สุด

ลอง query ที่มีทั้ง filter และ `ORDER BY ... LIMIT`:

```sql
EXPLAIN ANALYZE
SELECT order_id, order_date, status
FROM orders
WHERE ship_country = 'Thailand'
ORDER BY order_date DESC
LIMIT 10;
```

**ถ้าไม่มี index บน `order_date`** planner ต้องอ่านแถวที่ผ่าน filter แล้วนำมา **sort** เองในหน่วยความจำ:

```
                                                     QUERY PLAN
--------------------------------------------------------------------------------------------------------------------
 Limit  (cost=45.54..45.56 rows=10 width=21) (actual time=0.216..0.217 rows=10 loops=1)
   ->  Sort  (cost=45.54..46.06 rows=210 width=21) (actual time=0.215..0.215 rows=10 loops=1)
         Sort Key: order_date DESC
         Sort Method: top-N heapsort  Memory: 26kB
         ->  Seq Scan on orders o  (cost=0.00..41.00 rows=210 width=21) (actual time=0.010..0.159 rows=210 loops=1)
               Filter: ((ship_country)::text = 'Thailand'::text)
               Rows Removed by Filter: 1790
 Planning Time: 0.361 ms
 Execution Time: 0.241 ms
(9 rows)
```

สังเกตบรรทัด `Sort` ที่เพิ่มเข้ามา — เป็นขั้นตอนพิเศษที่ต้องทำหลังอ่านข้อมูลเสร็จ

**แต่เมื่อมี index บน `order_date` อยู่แล้ว** (จากที่สร้างไว้ข้างต้น) planner สามารถ**เดินไล่ index ย้อนกลับ (backward)** เพื่อให้ได้ลำดับที่ต้องการโดยตรง ไม่ต้อง sort เพิ่มเลย:

```
                                                                     QUERY PLAN
-----------------------------------------------------------------------------------------------------------------------------------------------------
 Limit  (cost=0.28..6.52 rows=10 width=21) (actual time=0.037..0.068 rows=10 loops=1)
   ->  Index Scan Backward using idx_orders_order_date on orders o  (cost=0.28..131.27 rows=210 width=21) (actual time=0.036..0.066 rows=10 loops=1)
         Filter: ((ship_country)::text = 'Thailand'::text)
         Rows Removed by Filter: 49
 Planning Time: 0.380 ms
 Execution Time: 0.089 ms
(6 rows)
```

ไม่มี `Sort` node อีกต่อไป! และเพราะมี `LIMIT 10` planner ยังฉลาดพอที่จะหยุดอ่านทันทีที่ได้ครบ 10 แถวที่ตรงเงื่อนไข (`Rows Removed by Filter: 49` น้อยกว่าการอ่านทั้ง 2,000 แถวมาก) — นี่คือเหตุผลที่ `ORDER BY ... LIMIT` กับ B-Tree index เป็นคู่หูที่ทรงพลังมากในการทำ pagination หรือ "ดึงรายการล่าสุด N รายการ"

> **ข้อจำกัดของ B-Tree:** B-Tree ไม่เหมาะกับการค้นหาแบบ "มีคำนี้อยู่ในข้อความหรือไม่" (`LIKE '%คำ%'`) หรือการค้นหาข้อมูลแบบ JSON/Array/ตำแหน่งภูมิศาสตร์ — กรณีเหล่านั้นต้องใช้ index ประเภทอื่น เช่น GIN, GiST, Hash ซึ่งจะเรียนใน Part 042 เป็นต้นไป

---

## Step 404: CREATE INDEX Syntax พื้นฐาน

Syntax พื้นฐานที่สุดของการสร้าง index:

```sql
CREATE INDEX index_name ON table_name (column_name);
```

ตัวอย่างที่เราสร้างไปแล้วในตัวอย่างก่อนหน้านี้:

```sql
CREATE INDEX idx_order_items_product_id ON order_items(product_id);
CREATE INDEX idx_orders_order_date ON orders(order_date);
```

### ตั้งชื่อ index อย่างไรดี

PostgreSQL ไม่บังคับรูปแบบชื่อ index แต่ธรรมเนียมที่นิยมใช้กันในวงการและทำให้ทีมอ่านโค้ดเข้าใจตรงกันคือ:

```
idx_<table_name>_<column_name>
```

เช่น `idx_orders_customer_id`, `idx_products_category_id` — ชื่อแบบนี้บอกทันทีว่า index อยู่บนตารางไหน คอลัมน์ไหน ทำให้ดูแลรักษาง่ายเมื่อระบบมี index หลายสิบตัว

หากไม่ระบุชื่อ ก็สามารถให้ PostgreSQL ตั้งชื่อให้อัตโนมัติได้ด้วย `CREATE INDEX ON table_name (column)` (ไม่ใส่ชื่อ) โดย PostgreSQL จะตั้งชื่อรูปแบบ `table_column_idx` ให้ แต่ **แนะนำให้ตั้งชื่อเองเสมอในโปรเจกต์จริง** เพื่อความชัดเจนและป้องกันชื่อชนกันโดยไม่ตั้งใจ

### ป้องกัน error ด้วย `IF NOT EXISTS`

```sql
CREATE INDEX IF NOT EXISTS idx_orders_customer_id ON orders(customer_id);
```

ถ้า index ชื่อนี้มีอยู่แล้ว คำสั่งจะไม่ error แต่แจ้งเตือนเฉย ๆ:

```
NOTICE:  relation "idx_orders_customer_id" already exists, skipping
CREATE INDEX
```

มีประโยชน์มากเวลาเขียน migration script ที่อาจต้องรันซ้ำได้โดยไม่พัง

### สร้าง index บนคอลัมน์เดียว — ตัวอย่างจริงเพิ่มเติม

ลองสร้าง index บน `orders.customer_id` (คอลัมน์ Foreign Key ที่ไม่มี index มาก่อน) และเปรียบเทียบ:

```sql
-- ก่อนสร้าง index
EXPLAIN ANALYZE SELECT * FROM orders WHERE customer_id = 5;
```

```
                                              QUERY PLAN
------------------------------------------------------------------------------------------------------
 Seq Scan on orders  (cost=0.00..41.00 rows=113 width=32) (actual time=0.007..0.125 rows=113 loops=1)
   Filter: (customer_id = 5)
   Rows Removed by Filter: 1887
 Planning Time: 0.416 ms
 Execution Time: 0.170 ms
(5 rows)
```

```sql
CREATE INDEX idx_orders_customer_id ON orders(customer_id);

EXPLAIN ANALYZE SELECT * FROM orders WHERE customer_id = 5;
```

```
                                                            QUERY PLAN
-----------------------------------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on orders  (cost=5.15..22.57 rows=113 width=32) (actual time=0.036..0.084 rows=113 loops=1)
   Recheck Cond: (customer_id = 5)
   Heap Blocks: exact=16
   ->  Bitmap Index Scan on idx_orders_customer_id  (cost=0.00..5.12 rows=113 width=0) (actual time=0.022..0.023 rows=113 loops=1)
         Index Cond: (customer_id = 5)
 Planning Time: 0.563 ms
 Execution Time: 0.138 ms
(7 rows)
```

cost ลดจาก 41.00 เหลือ 22.57 — ลูกค้ารายนี้มี 113 order จาก 2,000 order ทั้งหมด (~5.6%) ซึ่งยังถือว่าค่อนข้าง selective พอที่ index จะช่วยได้

> **จำไว้เสมอ:** `CREATE INDEX` เป็นคำสั่งที่**ล็อกตาราง**ระหว่างสร้าง (ในโหมดปกติจะล็อกแบบที่ยังให้ query อ่านได้แต่บล็อกการเขียน) บนตารางขนาดใหญ่ในระบบ production ควรใช้ `CREATE INDEX CONCURRENTLY` แทน (จะสอนละเอียดใน Part ถัดไปของชุด Index) เพื่อไม่ให้ระบบหยุดชะงักระหว่างสร้าง index

---

## Step 405: Index Scan vs Index Only Scan

ทั้งสองแบบนี้ต่างกันตรงที่ **Index Scan ต้องกลับไปอ่านตาราง (heap) เพิ่มเติม** ในขณะที่ **Index Only Scan ตอบคำถามได้จากข้อมูลใน index เพียงอย่างเดียว โดยไม่ต้องแตะ heap เลย**

### ทำไมต้องกลับไปอ่าน heap

index เก็บแค่คอลัมน์ที่ถูก index ไว้ (เช่น `product_id`) พร้อม pointer ไปยังตำแหน่งแถวจริง ถ้า query ขอ `SELECT *` หรือขอคอลัมน์อื่นที่ไม่ได้อยู่ใน index (เช่น `quantity`, `unit_price`) PostgreSQL ก็ต้องตามไปอ่านแถวจริงจาก heap เพื่อเอาข้อมูลคอลัมน์เหล่านั้นมา — ขั้นตอนนี้เรียกว่า **heap fetch**

แต่ถ้า query ขอ**เฉพาะคอลัมน์ที่มีอยู่ใน index แล้ว** PostgreSQL สามารถตอบคำถามจาก index อย่างเดียวได้เลย ไม่ต้องเสียเวลาไปอ่าน heap เพิ่ม — เร็วกว่ามาก โดยเฉพาะเมื่อ heap กระจายอยู่คนละ block บนดิสก์จำนวนมาก

### ทดลองจริง

```sql
EXPLAIN ANALYZE
SELECT product_id FROM order_items WHERE product_id = 500;
```

ผลลัพธ์แรก (ก่อน `VACUUM`):

```
                                                            QUERY PLAN
-----------------------------------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on order_items  (cost=4.32..18.45 rows=5 width=4) (actual time=0.021..0.022 rows=1 loops=1)
   Recheck Cond: (product_id = 500)
   Heap Blocks: exact=1
   ->  Bitmap Index Scan on idx_order_items_product_id  (cost=0.00..4.32 rows=5 width=0) (actual time=0.007..0.007 rows=1 loops=1)
         Index Cond: (product_id = 500)
 Planning Time: 0.516 ms
 Execution Time: 0.068 ms
(7 rows)
```

แม้จะ `SELECT` แค่คอลัมน์เดียวที่มีใน index แต่ PostgreSQL ยังคงต้องแวะไปที่ heap (`Bitmap Heap Scan`) — ทำไม? เพราะ PostgreSQL ใช้กลไก MVCC (Multi-Version Concurrency Control) ที่ต้องตรวจสอบว่าแถวนั้น "มองเห็นได้ (visible)" สำหรับ transaction ปัจจุบันหรือไม่ ข้อมูลนี้ปกติอยู่ใน heap เท่านั้น

PostgreSQL แก้ปัญหานี้ด้วย **Visibility Map (VM)** — บิตแมปพิเศษที่บันทึกว่า block ไหนของ heap มีแต่แถวที่ "ทุก transaction มองเห็นเหมือนกันหมด" (all-visible) แล้ว ถ้า block นั้นถูกทำเครื่องหมายไว้ ก็ **ไม่จำเป็นต้องแวะไปอ่าน heap เพื่อเช็ค visibility อีก** — Visibility Map จะถูกอัปเดตเมื่อรัน `VACUUM`:

```sql
VACUUM order_items;

EXPLAIN ANALYZE
SELECT product_id FROM order_items WHERE product_id = 500;
```

```
                                                                 QUERY PLAN
---------------------------------------------------------------------------------------------------------------------------------------------
 Index Only Scan using idx_order_items_product_id on order_items  (cost=0.28..4.37 rows=5 width=4) (actual time=0.018..0.019 rows=1 loops=1)
   Index Cond: (product_id = 500)
   Heap Fetches: 0
 Planning Time: 0.384 ms
 Execution Time: 0.057 ms
(5 rows)
```

ตอนนี้กลายเป็น **Index Only Scan** และบรรทัด `Heap Fetches: 0` ยืนยันว่าไม่ต้องแตะ heap เลยแม้แต่ครั้งเดียว! cost ลดลงจาก 18.45 เหลือแค่ 4.37

### เปรียบเทียบกับกรณีที่ต้องขอคอลัมน์นอก index

```sql
EXPLAIN ANALYZE
SELECT product_id, quantity FROM order_items WHERE product_id = 500;
```

เพราะ `quantity` ไม่ได้อยู่ใน index `idx_order_items_product_id` (ซึ่ง index แค่คอลัมน์ `product_id`) จึงกลับไปเป็น **Index Scan** ธรรมดาที่ต้องอ่าน heap (ทดสอบโดยปิด bitmap scan เพื่อดูรูปแบบที่ชัดเจน):

```
                                                               QUERY PLAN
-----------------------------------------------------------------------------------------------------------------------------------------
 Index Scan using idx_order_items_product_id on order_items  (cost=0.28..24.37 rows=5 width=8) (actual time=0.015..0.015 rows=1 loops=1)
   Index Cond: (product_id = 500)
 Planning Time: 0.458 ms
 Execution Time: 0.060 ms
(4 rows)
```

สรุปความแตกต่าง:

| | Index Scan | Index Only Scan |
|---|---|---|
| ต้องอ่าน heap ไหม | ต้อง (เพื่อเอาคอลัมน์อื่น + เช็ค visibility) | ไม่ต้อง (ถ้า block นั้น all-visible) |
| เงื่อนไข | คอลัมน์ที่ขอไม่ครบตาม index | คอลัมน์ที่ขอทั้งหมดอยู่ใน index และ VM บอกว่า all-visible |
| เร็วกว่าไหม | ช้ากว่า | เร็วกว่า เพราะลด I/O |
| ขึ้นกับ `VACUUM` ไหม | ไม่ขึ้น | ขึ้นมาก ถ้าไม่ค่อย vacuum จะได้ Index Scan ปกติแทน |

> **แนวคิด Covering Index เบื้องต้น:** จากตัวอย่างข้างต้น ถ้าเราอยากให้ query `SELECT product_id, quantity FROM order_items WHERE product_id = 500` เป็น Index Only Scan ได้ด้วย เราสามารถสร้าง index ที่ "ครอบคลุม (cover)" ทั้งคอลัมน์ที่ใช้กรองและคอลัมน์ที่ใช้ดึงข้อมูล เช่น `CREATE INDEX ON order_items(product_id, quantity)` หรือใช้ `INCLUDE` clause — รายละเอียดเชิงลึกเรื่อง covering index และ `INCLUDE` จะอยู่ใน Part ถัดไปของชุด Index

---

## Step 406: Bitmap Index Scan — เมื่อไหร่ Planner เลือกใช้

จากตัวอย่างหลาย ๆ ตัวข้างต้น คุณคงสังเกตเห็นว่า PostgreSQL มักเลือก **Bitmap Heap Scan** ร่วมกับ **Bitmap Index Scan** บ่อยกว่า Index Scan ธรรมดา นี่ไม่ใช่เรื่องบังเอิญ

### Bitmap Index Scan ทำงานอย่างไร

แทนที่จะเดินไป-กลับระหว่าง index กับ heap ทีละแถว (ซึ่งอาจกระโดดไปมาแบบสุ่มบนดิสก์ — random I/O ที่ช้า) Bitmap Index Scan ทำงานเป็น 2 ขั้นตอน:

1. **Bitmap Index Scan**: ค้นใน index เพื่อสร้าง "แผนที่บิต (bitmap)" ในหน่วยความจำ ระบุว่า block ไหนของ heap มีแถวที่ตรงเงื่อนไขบ้าง (ยังไม่ไปแตะ heap)
2. **Bitmap Heap Scan**: นำ bitmap นั้นมาไล่อ่าน heap **ตามลำดับ block** (sequential-ish I/O ที่เร็วกว่า) แทนที่จะกระโดดไปมาแบบสุ่ม แล้ว **Recheck Cond** เงื่อนไขอีกครั้งกับแถวจริง (จำเป็นเมื่อ bitmap เป็นแบบ "lossy" คือจำแค่ระดับ block ไม่ใช่ระดับแถว)

Planner จะเลือกวิธีนี้เมื่อคาดว่าจำนวนแถวที่ตรงเงื่อนไข "มากพอ" ที่การเดินสุ่มไปมาแบบ Index Scan ปกติจะไม่คุ้ม แต่ก็ "ไม่มากเกินไป" จนกระทั่ง Seq Scan อ่านทั้งตารางจะคุ้มกว่า — มันคือจุดกึ่งกลางระหว่าง Seq Scan กับ Index Scan

### จุดเด่น: รวมหลาย Index เข้าด้วยกันได้ (BitmapAnd / BitmapOr)

นี่คือจุดที่ Bitmap Index Scan ทรงพลังที่สุด — มันสามารถ**รวมผลลัพธ์จากหลาย index พร้อมกัน**ได้ในหน่วยความจำก่อนไปอ่าน heap แม้แต่ละคอลัมน์จะมี index แยกกันคนละตัวก็ตาม

ลองสร้าง index บนสองคอลัมน์ของ `products`:

```sql
CREATE INDEX idx_products_category_id ON products(category_id);
CREATE INDEX idx_products_supplier_id ON products(supplier_id);
```

**กรณี `OR`** — ต้องการแถวที่ตรงเงื่อนไขใดเงื่อนไขหนึ่ง มักจะเห็น `BitmapOr` รวมผลลัพธ์จากสอง index ชัดเจน:

```sql
EXPLAIN ANALYZE
SELECT * FROM products WHERE category_id = 3 OR supplier_id = 5;
```

```
                                                               QUERY PLAN
-----------------------------------------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on products  (cost=9.57..21.95 rows=153 width=35) (actual time=0.058..0.112 rows=151 loops=1)
   Recheck Cond: ((category_id = 3) OR (supplier_id = 5))
   Heap Blocks: exact=10
   ->  BitmapOr  (cost=9.57..9.57 rows=159 width=0) (actual time=0.039..0.040 rows=0 loops=1)
         ->  Bitmap Index Scan on idx_products_category_id  (cost=0.00..4.66 rows=68 width=0) (actual time=0.013..0.013 rows=68 loops=1)
               Index Cond: (category_id = 3)
         ->  Bitmap Index Scan on idx_products_supplier_id  (cost=0.00..4.83 rows=91 width=0) (actual time=0.025..0.025 rows=91 loops=1)
               Index Cond: (supplier_id = 5)
 Planning Time: 0.618 ms
 Execution Time: 0.232 ms
(10 rows)
```

จะเห็นว่า planner ใช้ **ทั้งสอง index พร้อมกัน** — สแกน `idx_products_category_id` ได้ 68 แถว, สแกน `idx_products_supplier_id` ได้ 91 แถว แล้วนำ bitmap ทั้งสองมา "รวม (OR)" กันเป็น bitmap เดียวก่อนไปอ่าน heap เพียงครั้งเดียว (`Heap Blocks: exact=10`) — ถ้าไม่มีกลไกนี้ PostgreSQL คงต้องอ่าน heap สองรอบแยกกันแล้วนำผลมารวมกันในภายหลัง ซึ่งจะแพงกว่า

**กรณี `AND`** — ต้องการแถวที่ตรงทั้งสองเงื่อนไข ลองดู:

```sql
EXPLAIN ANALYZE
SELECT * FROM products WHERE category_id = 3 AND supplier_id = 5;
```

```
                                                            QUERY PLAN
-----------------------------------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on products  (cost=4.66..15.68 rows=6 width=35) (actual time=0.025..0.049 rows=8 loops=1)
   Recheck Cond: (category_id = 3)
   Filter: (supplier_id = 5)
   Rows Removed by Filter: 60
   Heap Blocks: exact=10
   ->  Bitmap Index Scan on idx_products_category_id  (cost=0.00..4.66 rows=68 width=0) (actual time=0.011..0.011 rows=68 loops=1)
         Index Cond: (category_id = 3)
 Planning Time: 0.467 ms
 Execution Time: 0.113 ms
(9 rows)
```

น่าสนใจ — คราวนี้ planner **เลือกใช้แค่ index เดียว** (`idx_products_category_id`) แล้วกรองเงื่อนไข `supplier_id = 5` ด้วย `Filter` ธรรมดาหลังจากได้แถวมาแล้ว แทนที่จะทำ `BitmapAnd` รวมสอง index เข้าด้วยกัน นี่คือตัวอย่างจริงที่แสดงให้เห็นว่า **planner ไม่ได้ใช้ index ทุกตัวที่มีเสมอไป** — มันประเมินต้นทุนแล้วพบว่าบนข้อมูลขนาดนี้ (category_id = 3 มีแค่ 68 แถวจาก 1,000 แถว) การกรอง `supplier_id` ด้วย in-memory filter หลังจากได้ 68 แถวมาแล้วนั้น **ถูกกว่า**การเปิด index ตัวที่สองแล้วรวมผลแบบ `BitmapAnd`

ในระบบที่มีข้อมูลขนาดใหญ่กว่านี้มาก (เช่นหลักล้านแถว) และทั้งสองเงื่อนไขต่าง selective พอสมควร คุณจะเห็น `BitmapAnd` ปรากฏขึ้นจริง โดยมีรูปแบบคล้าย `BitmapOr` แต่รวมผลแบบ "และ" แทน — หลักการเดียวกันคือรวม bitmap ในหน่วยความจำก่อนไปแตะ heap

### สรุปเมื่อไหร่ Planner เลือก Bitmap Scan

| สถานการณ์ | แผนที่มักถูกเลือก |
|---|---|
| ตรงเงื่อนไขน้อยมาก (1-2 แถว) | Index Scan ธรรมดา |
| ตรงเงื่อนไขสัดส่วนปานกลาง (ไม่กี่% ถึงหลายสิบ%) | Bitmap Heap Scan |
| ตรงเงื่อนไขเกือบทั้งตาราง | Seq Scan |
| มีหลายเงื่อนไข AND/OR บนคอลัมน์ต่าง index กัน | อาจใช้ BitmapAnd/BitmapOr รวม index หลายตัว |

---

## Step 407: Unique Index — ความสัมพันธ์กับ PRIMARY KEY และ UNIQUE Constraint

ใน Part ก่อนหน้านี้ (ช่วงพื้นฐาน) คุณได้เรียนเรื่อง `PRIMARY KEY` และ `UNIQUE` constraint ไปแล้ว — สิ่งที่หลายคนไม่รู้คือ **เบื้องหลัง constraint ทั้งสองแบบนี้ PostgreSQL สร้าง Unique B-Tree Index ให้อัตโนมัติเสมอ** เพื่อใช้บังคับความไม่ซ้ำกันของค่า

ลองดูโครงสร้างตาราง `customers` ที่มีทั้ง `PRIMARY KEY` และ `UNIQUE`:

```sql
\d customers
```

```
                                           Table "public.customers"
   Column    |          Type          | Collation | Nullable |                    Default
-------------+------------------------+-----------+----------+------------------------------------------------
 customer_id | integer                |           | not null | nextval('customers_customer_id_seq'::regclass)
 first_name  | character varying(60)  |           | not null |
 last_name   | character varying(60)  |           | not null |
 email       | character varying(150) |           |          |
 country     | character varying(60)  |           |          |
 signup_date | date                   |           | not null | CURRENT_DATE
Indexes:
    "customers_pkey" PRIMARY KEY, btree (customer_id)
    "customers_email_key" UNIQUE CONSTRAINT, btree (email)
Referenced by:
    TABLE "orders" CONSTRAINT "orders_customer_id_fkey" FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
```

สังเกตส่วน `Indexes:` — ทั้ง `customers_pkey` (จาก `PRIMARY KEY`) และ `customers_email_key` (จาก `UNIQUE`) ถูกระบุว่าเป็น **btree** ทั้งคู่ นี่คือสิ่งที่เกิดขึ้นอัตโนมัติเมื่อคุณประกาศ:

```sql
CREATE TABLE customers (
    customer_id SERIAL PRIMARY KEY,   -- สร้าง unique btree index อัตโนมัติชื่อ customers_pkey
    ...
    email VARCHAR(150) UNIQUE,        -- สร้าง unique btree index อัตโนมัติชื่อ customers_email_key
    ...
);
```

### Unique Index ทำงานอย่างไรตอนบังคับความไม่ซ้ำ

เพราะ B-Tree เก็บข้อมูลเรียงลำดับ เวลา insert ค่าใหม่ PostgreSQL สามารถ**ค้นหาอย่างรวดเร็วว่ามีค่านี้อยู่แล้วหรือยัง**ก่อนตัดสินใจว่าจะ insert สำเร็จหรือ error — ลองทดสอบ:

```sql
INSERT INTO customers (first_name, last_name, email, country)
VALUES ('Dup', 'Test', 'tanaka.h@example.com', 'Thailand');
```

```
ERROR:  duplicate key value violates unique constraint "customers_email_key"
DETAIL:  Key (email)=(tanaka.h@example.com) already exists.
```

ข้อความ error บอกชัดเจนว่าเป็น unique constraint ตัวไหนที่ถูกละเมิด — ชื่อนี้ตรงกับชื่อ index ที่เห็นใน `\d customers`

### สร้าง Unique Index ได้เองโดยไม่ต้องผ่าน Constraint

คุณสามารถสร้าง unique index ตรง ๆ ได้เช่นกัน โดยไม่ต้องประกาศเป็น `UNIQUE` constraint บนตาราง:

```sql
CREATE UNIQUE INDEX idx_products_name_unique ON products(product_name);
```

```
CREATE INDEX
```

คำสั่งนี้สำเร็จเพราะข้อมูลตัวอย่างของเราตั้งชื่อ `'Product #' || g.id` ซึ่งไม่ซ้ำกันอยู่แล้ว (ตรวจสอบได้ด้วย `SELECT count(*), product_name FROM products GROUP BY product_name HAVING count(*) > 1` ซึ่งจะได้ 0 แถว) ถ้ามีค่าซ้ำอยู่ในตารางอยู่ก่อนแล้ว คำสั่ง `CREATE UNIQUE INDEX` จะ error ทันที

ลบทิ้งเพราะเป็นแค่ตัวอย่างสาธิต:

```sql
DROP INDEX idx_products_name_unique;
```

**ข้อแตกต่างระหว่างสองวิธี:**

| วิธี | ผลลัพธ์ |
|---|---|
| `ALTER TABLE ... ADD CONSTRAINT ... UNIQUE (col)` | ได้ทั้ง constraint (ปรากฏใน `\d` ส่วน constraint) และ unique index โดยอัตโนมัติ เหมาะเมื่อความไม่ซ้ำเป็น**กฎทางธุรกิจ**ที่ต้องบังคับเสมอ |
| `CREATE UNIQUE INDEX ... ON table(col)` | ได้แค่ unique index อย่างเดียว ไม่มี constraint object แยก ใช้ในกรณีพิเศษ เช่น unique index บนนิพจน์ (expression) หรือ partial unique index ที่ constraint ทำไม่ได้ |

> ทั้งสองวิธีบังคับความไม่ซ้ำได้เหมือนกันในทางปฏิบัติ เพราะ constraint ก็คือ unique index ภายใต้ฝากระโปรงอยู่ดี — ความต่างหลักคือ **metadata และความชัดเจนในการสื่อสารเจตนา (intent)**

### เกร็ดที่น่าสนใจ: index มีอยู่ แต่ planner ไม่เลือกใช้

ลองรัน:

```sql
EXPLAIN ANALYZE SELECT * FROM customers WHERE email = 'tanaka.h@example.com';
```

```
                                             QUERY PLAN
----------------------------------------------------------------------------------------------------
 Seq Scan on customers  (cost=0.00..1.25 rows=1 width=48) (actual time=0.009..0.010 rows=1 loops=1)
   Filter: ((email)::text = 'tanaka.h@example.com'::text)
   Rows Removed by Filter: 19
 Planning Time: 0.365 ms
 Execution Time: 0.047 ms
(5 rows)
```

แม้ `email` จะมี unique index (`customers_email_key`) อยู่แล้ว แต่ planner กลับเลือก **Seq Scan** เพราะตาราง `customers` มีแค่ 20 แถว — การอ่านทั้งตาราง 20 แถวนั้น "ถูกกว่า" การเปิด index อยู่ดี เรื่องนี้จะอธิบายเหตุผลเชิงลึกใน Step 408 ถัดไป: **การมี index ไม่ได้แปลว่า planner จะใช้มันเสมอ**

---

## Step 408: ต้นทุนของ Index

Index ไม่ใช่ของฟรี — ทุกครั้งที่คุณสร้าง index หนึ่งตัว คุณกำลังแลกกับต้นทุน 3 ด้านหลัก: **ความเร็วในการเขียนข้อมูลที่ลดลง**, **พื้นที่ดิสก์ที่เพิ่มขึ้น** และ **เวลาที่ planner ต้องใช้พิจารณาทางเลือกเพิ่มขึ้น**

### ต้นทุนที่ 1: INSERT/UPDATE/DELETE ช้าลง

ทุกครั้งที่มีการ `INSERT` แถวใหม่ PostgreSQL ต้อง**อัปเดตทุก index**ของตารางนั้นให้สอดคล้องกับข้อมูลใหม่ ไม่ใช่แค่เขียนลง heap เท่านั้น ยิ่งมี index มาก ยิ่งต้องเขียนมาก

มาพิสูจน์ด้วยการทดลองจริง: สร้างตารางเหมือนกันทุกประการสองตาราง ตารางหนึ่งไม่มี index เลย อีกตารางมี index 3 ตัว แล้ว bulk insert ข้อมูลจำนวนเท่ากัน:

```sql
CREATE TABLE oi_no_index (
    order_item_id SERIAL, order_id INT, product_id INT,
    quantity INT, unit_price NUMERIC(10,2)
);

CREATE TABLE oi_with_index (
    order_item_id SERIAL, order_id INT, product_id INT,
    quantity INT, unit_price NUMERIC(10,2)
);
CREATE INDEX idx_oiw_order_id   ON oi_with_index(order_id);
CREATE INDEX idx_oiw_product_id ON oi_with_index(product_id);
CREATE INDEX idx_oiw_unit_price ON oi_with_index(unit_price);
```

```sql
\timing on

INSERT INTO oi_no_index (order_id, product_id, quantity, unit_price)
SELECT (1+floor(random()*2000))::INT, (1+floor(random()*1000))::INT,
       (1+floor(random()*5))::INT, round((10+random()*4990)::NUMERIC,2)
FROM generate_series(1,50000);
```

```
INSERT 0 50000
Time: 111.115 ms
```

```sql
INSERT INTO oi_with_index (order_id, product_id, quantity, unit_price)
SELECT (1+floor(random()*2000))::INT, (1+floor(random()*1000))::INT,
       (1+floor(random()*5))::INT, round((10+random()*4990)::NUMERIC,2)
FROM generate_series(1,50000);
```

```
INSERT 0 50000
Time: 384.774 ms
```

**ผลลัพธ์จริง: การ insert 50,000 แถวเดียวกัน ใช้เวลา 111 ms บนตารางที่ไม่มี index เพิ่มเติม แต่ใช้เวลาถึง 384 ms (ช้ากว่าเกือบ 3.5 เท่า) บนตารางที่มี index 3 ตัว** นี่คือ 3 index บนตารางเดียว ลองจินตนาการระบบจริงที่มี index 8-10 ตัวบนตารางเดียว — ต้นทุนนี้จะยิ่งเห็นชัดขึ้นมาก โดยเฉพาะกับระบบที่ insert/update ถี่มาก (high write throughput)

### ต้นทุนที่ 2: พื้นที่ดิสก์เพิ่มขึ้น

Index แต่ละตัวคือโครงสร้างข้อมูลที่ต้องใช้พื้นที่ดิสก์แยกต่างหากจากตารางจริง มาดูขนาดจริง:

```sql
SELECT pg_size_pretty(pg_total_relation_size('oi_no_index'))   AS no_index_total,
       pg_size_pretty(pg_total_relation_size('oi_with_index')) AS with_index_total,
       pg_size_pretty(pg_relation_size('oi_no_index'))         AS no_index_table,
       pg_size_pretty(pg_relation_size('oi_with_index'))       AS with_index_table,
       pg_size_pretty(pg_indexes_size('oi_with_index'))        AS with_index_idx_size;
```

```
 no_index_total | with_index_total | no_index_table | with_index_table | with_index_idx_size
-----------------+------------------+-----------------+-------------------+----------------------
 2576 kB         | 4936 kB          | 2552 kB         | 2552 kB           | 2360 kB
```

ข้อมูลตารางเองมีขนาดเท่ากันทั้งคู่ (2552 kB) แต่ตารางที่มี index 3 ตัวมีขนาด**รวมเกือบเท่าตัว**เป็น 4936 kB เพราะ index ทั้ง 3 ตัวรวมกันกิน 2360 kB เพิ่มเข้ามา — **ในระบบจริงที่มีหลาย index บนตารางขนาดใหญ่ ขนาดของ index รวมกันอาจมากกว่าขนาดตารางจริงเสียอีก**

```sql
DROP TABLE oi_no_index;
DROP TABLE oi_with_index;
```

ลองดูตัวอย่างขนาด index จริงในตาราง `order_items` ของเราเอง (5,000 แถว, มี index 3 ตัว: primary key + 2 ตัวที่สร้างไว้):

```sql
SELECT pg_size_pretty(pg_relation_size('order_items'))       AS table_size,
       pg_size_pretty(pg_indexes_size('order_items'))        AS indexes_size,
       pg_size_pretty(pg_total_relation_size('order_items')) AS total_size;
```

```
 table_size | indexes_size | total_size
------------+--------------+------------
 256 kB     | 304 kB       | 592 kB
```

ตัวตารางเองแค่ 256 kB แต่ index ทั้งหมดรวมกันหนักกว่าตัวตารางเสียอีกที่ 304 kB!

### ต้นทุนที่ 3: เมื่อไหร่ไม่ควรสร้าง Index

จากทุกตัวอย่างที่ผ่านมา สรุปเป็นหลักเกณฑ์ได้ว่า**ไม่ควร**สร้าง index ในกรณีต่อไปนี้:

1. **ตารางมีข้อมูลน้อยมาก** (เช่น `customers` ที่มี 20 แถวในตัวอย่างของเรา) — Seq Scan อ่านทั้งตารางเร็วกว่าเปิด index อยู่แล้ว ดังที่เห็นใน Step 407
2. **คอลัมน์มีค่าซ้ำเยอะมาก (low cardinality / low selectivity)** เช่น `orders.status` ที่มีแค่ 5 ค่าที่เป็นไปได้ กระจายตัวใกล้เคียงกัน (~400 แถวต่อค่าจาก 2,000 แถว = 20% ต่อค่า):

```sql
SELECT status, count(*) FROM orders GROUP BY status ORDER BY status;
```

```
   status   | count
------------+-------
 cancelled  |   397
 delivered  |   419
 pending    |   382
 processing |   414
 shipped    |   388
```

```sql
EXPLAIN ANALYZE SELECT * FROM orders WHERE status = 'cancelled';
```

```
                                              QUERY PLAN
--------------------------------------------------------------------------------------------------------
 Seq Scan on orders  (cost=0.00..41.00 rows=397 width=32) (actual time=0.010..0.175 rows=397 loops=1)
   Filter: ((status)::text = 'cancelled'::text)
   Rows Removed by Filter: 1603
 Planning Time: 0.364 ms
 Execution Time: 0.224 ms
(5 rows)
```

แม้จะสร้าง index บน `status` ก็ตาม planner มักจะยังเลือก Seq Scan อยู่ดี เพราะการดึงข้อมูลถึง ~20% ของตารางออกมา ไม่คุ้มกับการเดินผ่าน index แล้ววกกลับไปอ่าน heap ทีละแถว — Seq Scan อ่านทั้งตารางรวดเดียวยังเร็วกว่า

3. **ตารางที่มีการเขียน (INSERT/UPDATE/DELETE) ถี่มากและอ่านน้อย** เช่น ตาราง log หรือ event queue ที่เขียนตลอดเวลาแต่แทบไม่มีการ query กลับ — ทุก index ที่มีคือต้นทุนที่จ่ายซ้ำทุกครั้งที่เขียนโดยไม่ได้ประโยชน์คืนมา
4. **คอลัมน์ที่แทบไม่เคยถูกใช้ใน `WHERE`, `JOIN`, หรือ `ORDER BY`** — สร้างไปก็เสียพื้นที่และเวลาเขียนเปล่า ๆ โดยไม่มีใครได้ใช้ประโยชน์จากมันเลย

> **หลักคิดสำคัญ:** ก่อนสร้าง index ทุกครั้ง ควรถามตัวเองว่า "คอลัมน์นี้ถูก query บ่อยแค่ไหน", "ตารางมีข้อมูลกี่แถว", "ค่าคอลัมน์นี้กระจายตัวแบบไหน (selective หรือไม่)" และ "ระบบนี้เน้นอ่านหรือเน้นเขียน" — index ที่ดีคือ index ที่**ถูกใช้งานจริงบ่อย ๆ** ไม่ใช่ index ที่สร้างไว้ "เผื่อ" โดยไม่มีเหตุผลรองรับ

---

## Step 409: DROP INDEX, REINDEX และการดูรายการ Index ที่มีอยู่

### DROP INDEX — ลบ Index

```sql
DROP INDEX index_name;
```

หรือแบบปลอดภัยกว่าเมื่อไม่แน่ใจว่า index มีอยู่จริงหรือไม่:

```sql
DROP INDEX IF EXISTS idx_products_supplier_id;
```

```
DROP INDEX
```

ทดสอบสร้างกลับคืน:

```sql
CREATE INDEX idx_products_supplier_id ON products(supplier_id);
```

> ในระบบ production ที่มีการใช้งานตลอดเวลา ควรใช้ `DROP INDEX CONCURRENTLY` เพื่อไม่ให้ query อื่นถูกบล็อกระหว่างลบ (รายละเอียดเชิงลึกอยู่ใน Part ถัดไปของชุด Index)

### REINDEX — สร้าง Index ใหม่ทับของเดิม

เมื่อเวลาผ่านไป index อาจเกิดปัญหา **bloat** (มีพื้นที่ว่างสะสมจากการ update/delete บ่อย ๆ) หรือเสียหายจากสาเหตุอื่น คำสั่ง `REINDEX` จะสร้าง index ตัวใหม่ทดแทนตัวเดิมทั้งหมด:

```sql
-- reindex เฉพาะ index ตัวเดียว
REINDEX INDEX idx_order_items_product_id;
```

```
REINDEX
```

```sql
-- reindex ทุก index ของตารางเดียว
REINDEX TABLE products;
```

```
REINDEX
```

คำสั่ง `REINDEX` ยังใช้ระดับอื่นได้ด้วย เช่น `REINDEX DATABASE dbname` (ทุก index ในฐานข้อมูล — ใช้ด้วยความระมัดระวังมาก เพราะกินเวลานานและล็อกตารางจำนวนมาก) หรือ `REINDEX SCHEMA schema_name`

> เช่นเดียวกับ `CREATE INDEX`, คำสั่ง `REINDEX` ปกติจะล็อกตาราง/index ระหว่างทำงาน ในระบบ production ควรใช้ `REINDEX ... CONCURRENTLY` (มีตั้งแต่ PostgreSQL 12 เป็นต้นไป) เพื่อลดผลกระทบต่อระบบที่กำลังใช้งานอยู่

### ดูรายการ Index ที่มีอยู่

**วิธีที่ 1: ใช้ `\di` ใน psql**

```sql
\di
```

```
                          List of relations
 Schema |            Name            | Type  |  Owner   |    Table
--------+----------------------------+-------+----------+-------------
 public | categories_pkey            | index | postgres | categories
 public | customers_email_key        | index | postgres | customers
 public | customers_pkey             | index | postgres | customers
 public | idx_order_items_order_id   | index | postgres | order_items
 public | idx_order_items_product_id | index | postgres | order_items
 public | idx_orders_customer_id     | index | postgres | orders
 public | idx_orders_order_date      | index | postgres | orders
 public | idx_products_category_id   | index | postgres | products
 public | idx_products_supplier_id   | index | postgres | products
 public | order_items_pkey           | index | postgres | order_items
 public | orders_pkey                | index | postgres | orders
 public | products_pkey              | index | postgres | products
 public | suppliers_pkey             | index | postgres | suppliers
(13 rows)
```

ถ้าอยากเห็นขนาดของแต่ละ index ด้วย ใช้ `\di+`:

```sql
\di+
```

จะได้คอลัมน์ `Size` เพิ่มมาบอกขนาดของแต่ละ index

**วิธีที่ 2: query ผ่าน view `pg_indexes`** (ใช้ได้แม้ไม่ได้อยู่ใน psql เช่นเรียกจาก application หรือ script)

```sql
SELECT indexname, indexdef
FROM pg_indexes
WHERE tablename = 'products';
```

```
        indexname         |                                      indexdef
---------------------------+--------------------------------------------------------------------------------------
 products_pkey             | CREATE UNIQUE INDEX products_pkey ON public.products USING btree (product_id)
 idx_products_category_id  | CREATE INDEX idx_products_category_id ON public.products USING btree (category_id)
 idx_products_supplier_id  | CREATE INDEX idx_products_supplier_id ON public.products USING btree (supplier_id)
(3 rows)
```

คอลัมน์ `indexdef` มีประโยชน์มาก เพราะให้คำสั่ง `CREATE INDEX` ฉบับเต็มที่ใช้ copy ไปสร้างใหม่บนฐานข้อมูลอื่นได้ทันที และบอกด้วยว่าเป็น index ประเภทไหน (`USING btree`)

**วิธีที่ 3: ดูรวมกับขนาดของแต่ละ index**

```sql
SELECT indexname,
       pg_size_pretty(pg_relation_size(indexname::regclass)) AS size
FROM pg_indexes
WHERE schemaname = 'public'
ORDER BY indexname;
```

**วิธีที่ 4: ดูรายละเอียด index ทั้งหมดของตารางหนึ่งพร้อมกับ constraint** — ใช้ `\d table_name` ตามที่เห็นใน Step 407

สรุปคำสั่งจัดการ index ทั้งหมดใน Step นี้:

| คำสั่ง | ใช้ทำอะไร |
|---|---|
| `DROP INDEX name;` | ลบ index |
| `DROP INDEX IF EXISTS name;` | ลบ index แบบไม่ error ถ้าไม่มีอยู่ |
| `DROP INDEX CONCURRENTLY name;` | ลบ index โดยไม่บล็อกการเขียน (production-safe) |
| `REINDEX INDEX name;` | สร้าง index ตัวเดียวใหม่ทับของเดิม |
| `REINDEX TABLE name;` | สร้างทุก index ของตารางใหม่ |
| `\di` / `\di+` | ดูรายการ index ผ่าน psql (แบบมี/ไม่มีขนาด) |
| `SELECT * FROM pg_indexes WHERE ...` | ดูรายการ index ผ่าน SQL (ใช้ได้ทุก client) |

---

## Step 410: แบบฝึกหัดรวม — วิเคราะห์ Query ที่ช้าและตัดสินใจสร้าง Index

มาลองประยุกต์สิ่งที่เรียนมาทั้งหมดกับสถานการณ์จริง: ทีม engineer รายงานว่าหน้า **"ประวัติคำสั่งซื้อของลูกค้า"** ในระบบ e-commerce ของเราช้าลงเรื่อย ๆ เมื่อข้อมูลโตขึ้น query ที่ใช้อยู่คือ:

```sql
EXPLAIN ANALYZE
SELECT o.order_id, o.order_date, o.status, o.ship_country
FROM orders o
WHERE o.customer_id = 12
ORDER BY o.order_date DESC;
```

### ขั้นตอนการวิเคราะห์

**ขั้นที่ 1: รัน `EXPLAIN ANALYZE` เพื่อดูว่า planner ทำอะไรอยู่จริง ๆ** — อย่าเดาเอง ให้ข้อมูลจริงเป็นตัวชี้นำเสมอ ดูว่ามี `Seq Scan` หรือ `Sort` node ที่ไม่จำเป็นปรากฏอยู่หรือไม่

**ขั้นที่ 2: เช็คว่ามี index บนคอลัมน์ที่ใช้ใน `WHERE` และ `ORDER BY` อยู่แล้วหรือยัง**

```sql
SELECT indexname, indexdef FROM pg_indexes WHERE tablename = 'orders';
```

**ขั้นที่ 3: ประเมิน selectivity ของคอลัมน์ที่จะสร้าง index** — ถามว่า "ถ้าสร้าง index บนคอลัมน์นี้ จะช่วยตัดจำนวนแถวที่ต้องอ่านได้มากแค่ไหน"

```sql
SELECT count(*) FROM orders WHERE customer_id = 12;
SELECT count(*) FROM orders;
```

ถ้า `customer_id = 12` มีแค่ไม่กี่สิบแถวจาก 2,000 แถวทั้งหมด (selective มาก) → index จะช่วยได้มาก

**ขั้นที่ 4: ตัดสินใจ** — จาก Step 404 เราได้สร้าง `idx_orders_customer_id` ไว้แล้ว และจาก Step 403 เราก็มี `idx_orders_order_date` ด้วย ลองดูว่า query ข้างต้นใช้ประโยชน์จากทั้งสอง index ได้ดีแค่ไหน:

```sql
EXPLAIN ANALYZE
SELECT o.order_id, o.order_date, o.status, o.ship_country
FROM orders o
WHERE o.customer_id = 12
ORDER BY o.order_date DESC;
```

Planner จะเลือกใช้ index ที่เหมาะกับสถานการณ์ที่สุด (อาจเป็น `idx_orders_customer_id` เพราะ filter ก่อนแล้วค่อย sort ผลลัพธ์ที่ได้ไม่กี่แถวในหน่วยความจำ ซึ่งเร็วกว่าเดิน `idx_orders_order_date` ทั้งตารางแล้วมา filter ทีหลัง) — นี่คือเหตุผลที่ **"มี index" ไม่ได้แปลว่า "index ถูกใช้เสมอ"** planner จะเลือกให้เองตามสถิติจริงของข้อมูล และในบาง scenario ก็อาจต้องพิจารณาสร้าง **composite index** บนหลายคอลัมน์พร้อมกัน (เช่น `(customer_id, order_date)`) เพื่อให้ query แบบนี้เร็วที่สุด — เนื้อหาเรื่อง composite/multi-column index จะอยู่ใน Part ถัดไปของชุด Index

### กรอบการตัดสินใจ (Decision Framework) ที่ใช้ได้กับทุกสถานการณ์

เมื่อเจอ query ที่ช้าในระบบจริง ให้ไล่ตามลำดับนี้:

1. **รัน `EXPLAIN ANALYZE`** ดูว่า bottleneck จริง ๆ อยู่ตรงไหน (Seq Scan บนตารางใหญ่? Sort ที่ไม่จำเป็น? Filter ที่ทิ้งแถวจำนวนมาก?)
2. **เช็คขนาดตาราง** — ถ้าเล็กมาก (หลักสิบ/ร้อยแถว) อาจไม่ต้องทำอะไรเลย เพราะ Seq Scan เร็วพออยู่แล้ว
3. **เช็ค selectivity ของคอลัมน์ที่จะ index** — ยิ่งค่ากระจายตัวมาก (unique มาก) ยิ่งเหมาะกับ index; ค่าที่ซ้ำเยอะ (เช่น boolean, status ไม่กี่ค่า) มักไม่คุ้ม
4. **เช็คว่าคอลัมน์นั้นถูกใช้บ่อยแค่ไหนใน `WHERE`/`JOIN`/`ORDER BY`** — สร้าง index เฉพาะที่คุ้มค่าจริง
5. **ชั่งน้ำหนักกับความถี่ในการเขียน** — ตารางที่ insert/update ถี่มากต้องระวังไม่ให้มี index เกินความจำเป็น
6. **สร้าง index แล้ววัดผลซ้ำด้วย `EXPLAIN ANALYZE`** เพื่อยืนยันว่าช่วยได้จริงตามที่คาด ไม่ใช่แค่เดา

---

## สรุปท้ายบท

ใน Part นี้เราได้ปูพื้นฐานที่สำคัญที่สุดเรื่องหนึ่งของ PostgreSQL — **Index แบบ B-Tree**:

- **Index** คือโครงสร้างข้อมูลแยกต่างหากที่เก็บค่าคอลัมน์แบบเรียงลำดับพร้อม pointer ไปยังแถวจริง ช่วยให้ค้นหาข้อมูลได้เร็วโดยไม่ต้องอ่านทั้งตาราง เปรียบเหมือนดัชนีท้ายเล่มหนังสือ
- **Seq Scan** อ่านทุกแถวเสมอ (O(n)) เหมาะกับตารางเล็กหรือดึงข้อมูลสัดส่วนมาก ส่วน **Index Scan** ค้นผ่านโครงสร้าง index ก่อน (ประมาณ O(log n)) เหมาะกับการดึงข้อมูลสัดส่วนน้อย
- **B-Tree** เป็นต้นไม้สมดุลที่เก็บข้อมูลเรียงลำดับ เหมาะกับ `=`, `<`, `>`, `BETWEEN`, `ORDER BY` เพราะใช้ประโยชน์จากลำดับที่เรียงไว้แล้วได้โดยตรง รวมถึงช่วยให้ `ORDER BY ... LIMIT` เร็วขึ้นมากโดยไม่ต้อง sort เพิ่ม
- `CREATE INDEX index_name ON table (column);` คือ syntax พื้นฐาน ควรตั้งชื่อตามธรรมเนียม `idx_<table>_<column>` และใช้ `IF NOT EXISTS` เมื่อจำเป็น
- **Index Scan** ต้องกลับไปอ่าน heap เพิ่ม ส่วน **Index Only Scan** ตอบได้จาก index อย่างเดียวเมื่อคอลัมน์ที่ขอครบและ Visibility Map ยืนยันว่า block นั้น all-visible (ต้องพึ่ง `VACUUM`)
- **Bitmap Index Scan** คือกลยุทธ์กึ่งกลางระหว่าง Seq Scan กับ Index Scan รวมผลจากหลาย index ได้ด้วย `BitmapAnd`/`BitmapOr` แต่ planner ก็ไม่ได้เลือกใช้ทุก index ที่มีเสมอไป — ขึ้นกับการประเมินต้นทุน
- `PRIMARY KEY` และ `UNIQUE` constraint สร้าง **Unique B-Tree Index** ให้อัตโนมัติเสมอ และสามารถสร้าง `CREATE UNIQUE INDEX` เองได้โดยตรงเช่นกัน
- Index มี**ต้นทุน**เสมอ: ทำให้ `INSERT/UPDATE/DELETE` ช้าลง (พิสูจน์แล้วว่าช้ากว่าได้หลายเท่า) และกินพื้นที่ดิสก์เพิ่ม (บางครั้งมากกว่าตารางจริงเสียอีก) — ไม่ควรสร้าง index พร่ำเพรื่อ
- `DROP INDEX`, `REINDEX`, `\di`/`\di+` และ `pg_indexes` คือเครื่องมือหลักในการจัดการและตรวจสอบ index ที่มีอยู่
- การตัดสินใจสร้าง index ที่ดีต้องอาศัย**ข้อมูลจริงจาก `EXPLAIN ANALYZE`** ไม่ใช่การเดา — พิจารณาขนาดตาราง, selectivity ของคอลัมน์, ความถี่ในการใช้งาน และสัดส่วนการอ่าน/เขียนของระบบประกอบกัน

Part ถัดไป (**Part 042**) เราจะไปทำความรู้จักกับ **Index ประเภทอื่น ๆ** นอกเหนือจาก B-Tree เช่น Hash, GIN, GiST, BRIN และ SP-GiST — แต่ละแบบเหมาะกับข้อมูลและรูปแบบการ query ที่ต่างกันไป

**บทถัดไป:** [Part 042 — Index ประเภทอื่น ๆ](./part-042-other-index-types.md)

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1

อธิบายด้วยคำพูดของตัวเองว่าทำไม Sequential Scan บนตารางที่มี 10 ล้านแถวถึงช้ากว่า Index Scan อย่างมาก ในขณะที่บนตารางที่มีแค่ 20 แถว ทั้งสองวิธีแทบไม่ต่างกัน

<details>
<summary>เฉลย</summary>

Sequential Scan ต้องอ่านทุกแถวในตารางเสมอไม่ว่าจะหาข้อมูลกี่แถวก็ตาม เวลาที่ใช้จึงเพิ่มขึ้นเป็นสัดส่วนโดยตรงกับจำนวนแถว (O(n)) — ถ้าตารางมี 10 ล้านแถว ก็ต้องอ่านทั้ง 10 ล้านแถว ในขณะที่ Index Scan ใช้โครงสร้าง B-Tree ที่ตัดครึ่งข้อมูลในแต่ละขั้น ทำให้จำนวนขั้นที่ต้องเดินเพิ่มขึ้นแบบ logarithm (O(log n)) เท่านั้น เช่น 10 ล้านแถวอาจใช้แค่ ~23 ขั้น เทียบกับ 20 แถวที่ใช้แค่ ~5 ขั้น

แต่บนตารางที่มีแค่ 20 แถว การอ่านทั้ง 20 แถวใช้เวลาน้อยมากอยู่แล้ว (block ข้อมูลเล็กมาก มักถูก cache ไว้ใน memory ทั้งหมด) ในขณะที่การเปิด index ก็มี overhead ของตัวมันเอง (ต้องเดินผ่าน node ของ index ก่อน แล้วยังต้องวกกลับไปอ่าน heap อีก) ทำให้บนข้อมูลขนาดเล็กมาก ๆ ต้นทุนของการเปิด index อาจจะสูงกว่าการอ่านทั้งตารางตรง ๆ เสียด้วยซ้ำ — นี่คือเหตุผลที่ planner มักเลือก Seq Scan บนตารางเล็กแม้จะมี index อยู่ก็ตาม

</details>

### แบบฝึกหัดที่ 2

เขียนคำสั่ง SQL สร้าง index บนคอลัมน์ `products.is_active` โดยตั้งชื่อตามธรรมเนียมที่เรียนมาในบทนี้ แล้วอธิบายว่าทำไม index นี้อาจไม่มีประโยชน์มากนัก

<details>
<summary>เฉลย</summary>

```sql
CREATE INDEX idx_products_is_active ON products(is_active);
```

Index นี้อาจไม่มีประโยชน์มากนัก เพราะ `is_active` เป็นคอลัมน์ `BOOLEAN` ที่มีแค่ 2 ค่าที่เป็นไปได้ (`true`/`false`) และจากข้อมูลตัวอย่างของเรา ~90% ของแถวเป็น `true` — เป็นคอลัมน์ที่มี **cardinality ต่ำมาก (low selectivity)** การ query `WHERE is_active = true` จะได้ผลลัพธ์เกือบทั้งตาราง ทำให้ planner มักเลือก Seq Scan อยู่ดีเพราะถูกกว่าการเปิด index แล้ววกไปอ่าน heap ทีละแถวสำหรับข้อมูลสัดส่วนมากขนาดนั้น

</details>

### แบบฝึกหัดที่ 3

รันคำสั่งต่อไปนี้บนตาราง `order_items` ก่อนและหลังสร้าง index บน `order_id` แล้วอธิบายความแตกต่างของผลลัพธ์ `EXPLAIN ANALYZE` ที่ได้:

```sql
SELECT * FROM order_items WHERE order_id = 250;
```

<details>
<summary>เฉลย</summary>

ก่อนสร้าง index จะเห็น `Seq Scan on order_items` พร้อม `Filter: (order_id = 250)` และ `Rows Removed by Filter` ที่มีค่าสูงใกล้เคียงกับจำนวนแถวทั้งหมดของตาราง (~5,000) แสดงว่าต้องอ่านทุกแถวแล้วทิ้งเกือบหมดเพื่อหาแถวที่ตรงกัน

```sql
CREATE INDEX idx_order_items_order_id ON order_items(order_id);
```

หลังสร้าง index จะเห็นแผนเปลี่ยนเป็น `Bitmap Heap Scan on order_items` ที่มี `Bitmap Index Scan on idx_order_items_order_id` เป็น child node พร้อม `Index Cond: (order_id = 250)` — ตอนนี้ PostgreSQL ใช้ index เพื่อหาตำแหน่งแถวที่ตรงเงื่อนไขโดยตรงแทนที่จะอ่านทั้งตาราง ทำให้ `cost` และ `Execution Time` ลดลงอย่างเห็นได้ชัด

</details>

### แบบฝึกหัดที่ 4

อธิบายว่า Index Only Scan ต่างจาก Index Scan อย่างไร และเพราะเหตุใดการรัน `VACUUM` จึงมีผลต่อการที่ query จะได้ Index Only Scan หรือไม่

<details>
<summary>เฉลย</summary>

Index Scan ต้องเดินผ่าน index เพื่อหาตำแหน่งแถว แล้ว**วกกลับไปอ่านแถวจริงจาก heap** เพื่อ (1) เอาข้อมูลคอลัมน์ที่ไม่ได้อยู่ใน index และ (2) ตรวจสอบว่าแถวนั้น "มองเห็นได้ (visible)" สำหรับ transaction ปัจจุบันตามกลไก MVCC หรือไม่

Index Only Scan สามารถตอบคำถามได้จากข้อมูลใน index เพียงอย่างเดียว โดยไม่ต้องแตะ heap เลย แต่ทำได้ก็ต่อเมื่อคอลัมน์ที่ query ขอทั้งหมดมีอยู่ใน index **และ** PostgreSQL มั่นใจได้ว่าแถวนั้น all-visible โดยไม่ต้องเช็คกับ heap — ข้อมูลความ "all-visible" นี้เก็บอยู่ใน **Visibility Map** ซึ่งจะถูกอัปเดตให้ทันสมัยก็ต่อเมื่อรัน `VACUUM` เท่านั้น ถ้าตารางไม่ได้ vacuum มานาน (เช่นเพิ่ง insert/update ข้อมูลจำนวนมาก) Visibility Map จะไม่ทันสมัย ทำให้ PostgreSQL ต้องกลับไปเช็ค heap อยู่ดี (จะกลายเป็น Index Scan ธรรมดาแทน แม้คอลัมน์ที่ขอจะอยู่ใน index ครบก็ตาม)

</details>

### แบบฝึกหัดที่ 5

ระหว่าง `CREATE INDEX` และ `CREATE UNIQUE INDEX` ต่างกันอย่างไร และสัมพันธ์กับ `PRIMARY KEY`/`UNIQUE` constraint ที่เรียนมาก่อนหน้านี้อย่างไร

<details>
<summary>เฉลย</summary>

`CREATE INDEX` สร้าง index ธรรมดาที่ไม่บังคับความไม่ซ้ำของค่า ส่วน `CREATE UNIQUE INDEX` สร้าง index ที่**บังคับว่าทุกค่าในคอลัมน์ (หรือกลุ่มคอลัมน์) ที่ index ต้องไม่ซ้ำกัน** — ถ้า insert ค่าที่ซ้ำจะเกิด error ทันที

เรื่องนี้สัมพันธ์กับ `PRIMARY KEY` และ `UNIQUE` constraint โดยตรง เพราะเบื้องหลัง PostgreSQL สร้าง **Unique B-Tree Index ให้อัตโนมัติเสมอ**เมื่อประกาศ constraint ทั้งสองแบบนี้ (เช่น `customer_id SERIAL PRIMARY KEY` สร้าง `customers_pkey` และ `email VARCHAR(150) UNIQUE` สร้าง `customers_email_key` ซึ่งทั้งคู่เป็น unique btree index) การสร้าง `CREATE UNIQUE INDEX` ตรง ๆ เองก็ได้ผลบังคับความไม่ซ้ำเหมือนกัน ต่างกันที่ไม่มี constraint object แยกต่างหากปรากฏใน metadata และมักใช้ในกรณีพิเศษ เช่น unique index บนนิพจน์หรือ partial unique index ที่ constraint แบบปกติทำไม่ได้

</details>

### แบบฝึกหัดที่ 6

ทีม data engineer เสนอให้สร้าง index บนทุกคอลัมน์ของตาราง `order_items` "เผื่อไว้" ในอนาคตจะมี query ใช้งาน คุณจะให้คำแนะนำอย่างไร โดยอ้างอิงแนวคิดต้นทุนของ index ที่เรียนมา

<details>
<summary>เฉลย</summary>

ไม่ควรทำเช่นนั้น เพราะ index แต่ละตัวมีต้นทุนจริงสองด้านหลัก: (1) ทำให้ `INSERT/UPDATE/DELETE` ช้าลง เพราะทุกครั้งที่เขียนข้อมูล PostgreSQL ต้องอัปเดต**ทุก index** ให้สอดคล้องกับข้อมูลใหม่ (จากตัวอย่างในบทนี้ การ insert 50,000 แถวช้าลงจาก 111 ms เป็น 384 ms เมื่อมี index เพิ่ม 3 ตัว) และ (2) กินพื้นที่ดิสก์เพิ่ม ซึ่งบางครั้งรวมกันมากกว่าขนาดตารางจริงเสียอีก

การสร้าง index "เผื่อไว้" โดยไม่มี query จริงมารองรับคือการจ่ายต้นทุนทั้งสองด้านนี้ตลอดไปโดยไม่ได้ประโยชน์คืนมาเลย คำแนะนำที่ถูกต้องคือ**สร้าง index เฉพาะคอลัมน์ที่มีการใช้งานจริงบ่อย ๆ ใน `WHERE`, `JOIN`, หรือ `ORDER BY`** โดยพิจารณาจาก query ที่เกิดขึ้นจริงในระบบ (เช่นดูจาก slow query log หรือ `pg_stat_statements`) ไม่ใช่คาดเดาล่วงหน้า และควรทบทวน index ที่ไม่ได้ถูกใช้เป็นระยะเพื่อพิจารณาลบทิ้ง

</details>

### แบบฝึกหัดที่ 7

ตาราง `orders` มี index อยู่บน `order_date` แล้ว ลองเขียน query ที่ใช้ทั้ง `WHERE ship_country = 'Thailand'` และ `ORDER BY order_date DESC LIMIT 5` แล้วอธิบายว่าทำไม EXPLAIN ถึงไม่มี node ชื่อ `Sort`

<details>
<summary>เฉลย</summary>

```sql
EXPLAIN ANALYZE
SELECT order_id, order_date, status
FROM orders
WHERE ship_country = 'Thailand'
ORDER BY order_date DESC
LIMIT 5;
```

จะไม่เห็น node ชื่อ `Sort` เพราะข้อมูลใน B-Tree index `idx_orders_order_date` **ถูกเรียงลำดับไว้อยู่แล้ว** PostgreSQL จึงสามารถเดินไล่อ่าน index ตามลำดับที่ต้องการได้โดยตรง (ในที่นี้คือ `Index Scan Backward` เพราะต้องการ `DESC`) แทนที่จะต้องอ่านข้อมูลทั้งหมดมาก่อนแล้วค่อย sort เพิ่มในหน่วยความจำ (ซึ่งจะปรากฏเป็น node `Sort` แยกต่างหาก) การใช้ index เพื่อหลีกเลี่ยงขั้นตอน sort แบบนี้เป็นหนึ่งในประโยชน์สำคัญของ B-Tree โดยเฉพาะเมื่อรวมกับ `LIMIT` เพราะ planner จะหยุดอ่านทันทีที่ได้ครบจำนวนแถวที่ต้องการ ไม่ต้องอ่านข้อมูลทั้งตาราง

</details>

### แบบฝึกหัดที่ 8

คำสั่งใดใช้ตรวจสอบขนาด (เป็นไบต์หรือหน่วยอ่านง่าย) ของ index ทุกตัวในตาราง `products` พร้อมกัน โดยไม่ต้องเปิด `\di+`

<details>
<summary>เฉลย</summary>

```sql
SELECT indexname,
       pg_size_pretty(pg_relation_size(indexname::regclass)) AS size
FROM pg_indexes
WHERE tablename = 'products'
ORDER BY indexname;
```

ใช้ view `pg_indexes` ร่วมกับฟังก์ชัน `pg_relation_size()` (คืนค่าขนาดเป็นไบต์ของ object ที่ระบุ) และ `pg_size_pretty()` (แปลงไบต์ให้อ่านง่ายเป็น kB/MB/GB) วิธีนี้ใช้ได้จากทุก client ที่เชื่อมต่อ SQL ได้ ไม่จำเป็นต้องอยู่ใน psql เท่านั้นเหมือน `\di+`

</details>

### แบบฝึกหัดที่ 9

ระบบ e-commerce ของเรามี query นี้ทำงานทุกครั้งที่ผู้ดูแลระบบเปิดหน้า dashboard สรุปยอดขายรายวัน:

```sql
SELECT date_trunc('day', order_date) AS day, count(*) AS order_count
FROM orders
WHERE order_date >= now() - interval '30 days'
GROUP BY day
ORDER BY day;
```

query นี้ควรได้ประโยชน์จาก index ที่มีอยู่แล้วหรือไม่ (`idx_orders_order_date`) จงอธิบายเหตุผล

<details>
<summary>เฉลย</summary>

ควรได้ประโยชน์บางส่วน — เงื่อนไข `WHERE order_date >= now() - interval '30 days'` เป็นการเปรียบเทียบแบบช่วง (`>=`) บนคอลัมน์ `order_date` ตรง ๆ ซึ่ง B-Tree index `idx_orders_order_date` รองรับได้ดี PostgreSQL จะใช้ index (มักผ่าน Bitmap Index Scan หรือ Index Scan) เพื่อกรองแถวที่อยู่ในช่วง 30 วันล่าสุดได้อย่างรวดเร็วโดยไม่ต้องอ่านทั้งตาราง โดยเฉพาะถ้าช่วง 30 วันคิดเป็นสัดส่วนน้อยของข้อมูลทั้งหมด (เช่นถ้าข้อมูลมีย้อนหลัง 2 ปี 30 วันคิดเป็นแค่ ~4% เท่านั้น)

อย่างไรก็ตาม ส่วน `GROUP BY date_trunc('day', order_date)` และ `count(*)` เป็นการประมวลผลเพิ่มเติมที่เกิด**หลังจาก**กรองแถวด้วย index แล้ว — ขั้นตอนนี้ index จะไม่ได้ช่วยอะไรเพิ่ม (เว้นแต่จะสร้าง index แบบ expression บน `date_trunc('day', order_date)` โดยเฉพาะ ซึ่งเป็นเทคนิคขั้นสูงกว่าที่จะพูดถึงใน Part ต่อ ๆ ไปของชุด Index) โดยสรุป index ช่วยได้ที่ขั้นตอนกรองข้อมูล แต่ไม่ได้ช่วยขั้นตอน aggregation/group by

</details>

### แบบฝึกหัดที่ 10

จากสถานการณ์ Step 410: มี query ค้นหาคำสั่งซื้อของลูกค้ารายหนึ่งที่ช้า และคุณพบว่าตาราง `orders` มี index อยู่แล้วทั้งบน `customer_id` และ `order_date` แยกกันคนละตัว แต่ query ที่ทั้ง filter `customer_id` และ `ORDER BY order_date` พร้อมกันยังคงมีส่วนที่ประมวลผลช้าอยู่บ้าง คุณคิดว่าสาเหตุที่เป็นไปได้คืออะไร และควรตรวจสอบอะไรต่อไป (โดยไม่ต้องลงรายละเอียดเชิงลึกเกินเนื้อหาบทนี้)

<details>
<summary>เฉลย</summary>

สาเหตุที่เป็นไปได้คือ planner เลือกใช้ได้แค่ index เดียวสำหรับ node การกรองข้อมูล (เช่นใช้ `idx_orders_customer_id` กรองก่อน) แล้วค่อยนำผลลัพธ์ (ซึ่งอาจมีหลายสิบ/หลายร้อยแถว) มาทำ `Sort` เพิ่มเองในหน่วยความจำเพื่อให้ตรงกับ `ORDER BY order_date` เพราะ index บน `customer_id` เพียงอย่างเดียวไม่ได้เรียงข้อมูลตาม `order_date` ให้ — ทำให้ยังเห็น node `Sort` ปรากฏใน `EXPLAIN ANALYZE` แม้จะมี index ทั้งสองตัวอยู่แล้วก็ตาม

สิ่งที่ควรตรวจสอบต่อไปคือ:
1. รัน `EXPLAIN ANALYZE` จริงเพื่อดูว่ามี node `Sort` ปรากฏอยู่หรือไม่ และมันกินเวลาสัดส่วนเท่าไหร่ของ query ทั้งหมด
2. เช็คว่าจำนวนแถวที่ลูกค้ารายนั้นมี order เยอะแค่ไหน — ถ้ามีน้อย (หลักสิบแถว) การ sort ในหน่วยความจำก็เร็วมากอยู่แล้วและอาจไม่ใช่ปัญหาจริง
3. ถ้าพบว่าเป็นปัญหาจริงบนข้อมูลขนาดใหญ่ (ลูกค้าที่มี order เยอะมาก) แนวทางแก้คือพิจารณาสร้าง **composite index** บนหลายคอลัมน์พร้อมกัน เช่น `(customer_id, order_date)` เพื่อให้ index เดียวรองรับทั้งการกรองและการเรียงลำดับพร้อมกันโดยไม่ต้อง sort เพิ่ม — ซึ่งเป็นเนื้อหาเชิงลึกที่จะเรียนใน Part ถัดไปของชุด Index

ข้อคิดสำคัญคือ index แยกทีละคอลัมน์ไม่ได้แปลว่าจะครอบคลุมทุก pattern การ query เสมอไป ต้องดูจากรูปแบบ query จริงที่ใช้บ่อยประกอบการออกแบบ index

</details>

---

**บทถัดไป:** [Part 042 — Index ประเภทอื่น ๆ](./part-042-other-index-types.md)
