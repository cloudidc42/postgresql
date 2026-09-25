# Part 046: PL/pgSQL — Stored Procedures และ Functions เบื้องต้น

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับสูง | Part 046

---

## เป้าหมายการเรียนรู้

เมื่อจบบทนี้ ผู้เรียนจะสามารถ:

- อธิบายได้ว่า PL/pgSQL คืออะไร และทำไม PostgreSQL ถึงต้องการ procedural language นอกเหนือจาก SQL ล้วน ๆ
- เขียน `CREATE FUNCTION` พื้นฐานได้ ทั้งแบบรับพารามิเตอร์ คืนค่าตัวเดียว และคืนหลายแถว
- ใช้ `DECLARE`, ตัวแปร, และ `%TYPE` เพื่อผูก type ของตัวแปรกับคอลัมน์จริงในตาราง ลดปัญหา type mismatch เมื่อ schema เปลี่ยน
- ใช้ `RETURN`, `RETURNS TABLE`, `RETURNS SETOF` ให้เหมาะกับสถานการณ์ที่ต่างกัน
- ใช้ `IF / ELSIF / ELSE` เขียน business logic แบบมีเงื่อนไขภายในฟังก์ชัน
- เข้าใจ function volatility (`IMMUTABLE`, `STABLE`, `VOLATILE`) และผลกระทบต่อ query planner และ caching
- แยกความแตกต่างระหว่าง `FUNCTION` กับ `PROCEDURE` และรู้ว่าเมื่อไรควรใช้อะไร
- เข้าใจ `SECURITY DEFINER` vs `SECURITY INVOKER` และความเสี่ยงด้านความปลอดภัยที่ต้องระวัง
- เขียนฟังก์ชันที่แก้ปัญหาธุรกิจจริง เช่น คำนวณส่วนลดตามระดับสมาชิก และดึงรายการสินค้าขายดี

---

## เตรียมข้อมูล

บทนี้ยังคงใช้ฐานข้อมูลร้านค้าออนไลน์ (e-commerce) ชุดเดิมที่ใช้ตลอดทั้งหลักสูตร ประกอบด้วย 6 ตาราง: `categories`, `suppliers`, `products`, `customers`, `orders`, `order_items`

```sql
-- ลบตารางเดิมถ้ามี เพื่อให้เริ่มต้นสะอาด (เรียงลำดับตาม dependency)
DROP TABLE IF EXISTS order_items CASCADE;
DROP TABLE IF EXISTS orders CASCADE;
DROP TABLE IF EXISTS products CASCADE;
DROP TABLE IF EXISTS customers CASCADE;
DROP TABLE IF EXISTS suppliers CASCADE;
DROP TABLE IF EXISTS categories CASCADE;

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
    product_id        SERIAL PRIMARY KEY,
    product_name      VARCHAR(150) NOT NULL,
    category_id       INTEGER REFERENCES categories(category_id),
    supplier_id       INTEGER REFERENCES suppliers(supplier_id),
    unit_price        NUMERIC(10,2) NOT NULL,
    stock_quantity    INTEGER NOT NULL DEFAULT 0,
    is_active         BOOLEAN NOT NULL DEFAULT true
);

CREATE TABLE customers (
    customer_id     SERIAL PRIMARY KEY,
    first_name      VARCHAR(60) NOT NULL,
    last_name       VARCHAR(60) NOT NULL,
    email           VARCHAR(150) UNIQUE,
    country         VARCHAR(60),
    signup_date     DATE NOT NULL DEFAULT CURRENT_DATE
);

CREATE TABLE orders (
    order_id        SERIAL PRIMARY KEY,
    customer_id     INTEGER REFERENCES customers(customer_id),
    order_date      TIMESTAMPTZ NOT NULL DEFAULT now(),
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',
    ship_country    VARCHAR(60)
);

CREATE TABLE order_items (
    order_item_id   SERIAL PRIMARY KEY,
    order_id        INTEGER REFERENCES orders(order_id),
    product_id      INTEGER REFERENCES products(product_id),
    quantity        INTEGER NOT NULL CHECK (quantity > 0),
    unit_price      NUMERIC(10,2) NOT NULL
);
```

### ข้อมูลตัวอย่าง

```sql
-- categories: มีลำดับชั้น (parent-child)
INSERT INTO categories (category_id, category_name, parent_category_id) VALUES
    (1,  'Electronics',        NULL),
    (2,  'Computers',          1),
    (3,  'Smartphones',        1),
    (4,  'Audio',              1),
    (5,  'Books',               NULL),
    (6,  'Fiction',            5),
    (7,  'Non-Fiction',        5),
    (8,  'Clothing',           NULL),
    (9,  'Men''s Clothing',    8),
    (10, 'Women''s Clothing',  8),
    (11, 'Home & Kitchen',     NULL),
    (12, 'Sports & Outdoors',  NULL);
SELECT setval('categories_category_id_seq', 12);

-- suppliers
INSERT INTO suppliers (supplier_id, supplier_name, country) VALUES
    (1,  'TechSource Inc.',            'USA'),
    (2,  'Global Gadgets Ltd.',        'China'),
    (3,  'BookWorld Publishing',       'UK'),
    (4,  'Fashion Forward Co.',        'Italy'),
    (5,  'HomeEssentials Trading',     'Thailand'),
    (6,  'SoundWave Audio',            'Japan'),
    (7,  'SportGear International',   'Germany'),
    (8,  'EcoStyle Textiles',          'Vietnam'),
    (9,  'Prime Electronics',          'South Korea'),
    (10, 'Nordic Living',              'Sweden');
SELECT setval('suppliers_supplier_id_seq', 10);

-- products
INSERT INTO products (product_id, product_name, category_id, supplier_id, unit_price, stock_quantity, is_active) VALUES
    (1,  'UltraBook Pro 15',           2,  1,  32900.00, 45,  true),
    (2,  'Galaxy Phone X200',          3,  2,  24900.00, 60,  true),
    (3,  'Wireless Earbuds Z1',        4,  6,  2590.00,  150, true),
    (4,  'Mechanical Keyboard K80',    2,  1,  3200.00,  80,  true),
    (5,  '4K Monitor 27"',             2,  9,  8900.00,  30,  true),
    (6,  'The Silent Ocean (Novel)',   6,  3,  350.00,   200, true),
    (7,  'Data Structures Explained',  7,  3,  590.00,   90,  true),
    (8,  'Men''s Denim Jacket',        9,  4,  1290.00,  70,  true),
    (9,  'Women''s Summer Dress',      10, 4,  990.00,   65,  true),
    (10, 'Non-Stick Frying Pan',       11, 5,  690.00,   120, true),
    (11, 'Yoga Mat Premium',           12, 7,  850.00,   100, true),
    (12, 'Bluetooth Speaker Mini',     4,  6,  1490.00,  110, true),
    (13, 'Smartphone Case Leather',    3,  2,  490.00,   300, true),
    (14, 'Cotton T-Shirt Basic',       9,  8,  350.00,   250, true),
    (15, 'Running Shoes Air',          12, 7,  2890.00,  55,  true),
    (16, 'Coffee Maker Deluxe',        11, 10, 2450.00,  40,  true),
    (17, 'Sci-Fi Anthology Vol.2',     6,  3,  420.00,   75,  true),
    (18, 'Gaming Mouse RGB',           2,  1,  1590.00,  130, true),
    (19, 'Winter Coat Wool',           10, 4,  3590.00,  25,  false),
    (20, 'Portable SSD 1TB',           2,  9,  3990.00,  60,  true);
SELECT setval('products_product_id_seq', 20);

-- customers
INSERT INTO customers (customer_id, first_name, last_name, email, country, signup_date) VALUES
    (1,  'Somchai',   'Jaidee',     'somchai.j@example.com',   'Thailand', '2023-01-15'),
    (2,  'Emily',     'Watson',     'emily.w@example.com',     'USA',      '2023-02-20'),
    (3,  'Yuki',      'Tanaka',     'yuki.t@example.com',      'Japan',    '2023-03-05'),
    (4,  'Nattaya',   'Suksawat',   'nattaya.s@example.com',   'Thailand', '2023-03-18'),
    (5,  'James',     'Miller',     'james.m@example.com',     'UK',       '2023-04-01'),
    (6,  'Chalermchai','Boon',      'chalermchai.b@example.com','Thailand','2023-04-22'),
    (7,  'Sophie',    'Dubois',     'sophie.d@example.com',    'France',   '2023-05-10'),
    (8,  'Wei',       'Zhang',      'wei.z@example.com',       'China',    '2023-05-29'),
    (9,  'Pimchanok', 'Rattana',    'pimchanok.r@example.com', 'Thailand', '2023-06-14'),
    (10, 'Carlos',    'Garcia',     'carlos.g@example.com',    'Spain',    '2023-07-02'),
    (11, 'Anong',     'Phetch',     'anong.p@example.com',     'Thailand', '2023-07-19'),
    (12, 'Liam',      'O''Brien',   'liam.o@example.com',      'Ireland',  '2023-08-08'),
    (13, 'Suda',      'Meechai',    'suda.m@example.com',      'Thailand', '2023-09-01'),
    (14, 'Hannah',    'Schmidt',    'hannah.s@example.com',    'Germany',  '2023-09-25'),
    (15, 'Kittipong',  'Wong',      'kittipong.w@example.com', 'Thailand', '2023-10-11');
SELECT setval('customers_customer_id_seq', 15);

-- orders
INSERT INTO orders (order_id, customer_id, order_date, status, ship_country) VALUES
    (1,  1,  '2024-01-05 09:12:00+07', 'delivered',  'Thailand'),
    (2,  2,  '2024-01-08 14:30:00+07', 'delivered',  'USA'),
    (3,  3,  '2024-01-15 10:05:00+07', 'delivered',  'Japan'),
    (4,  1,  '2024-01-20 16:45:00+07', 'delivered',  'Thailand'),
    (5,  4,  '2024-02-02 11:20:00+07', 'shipped',    'Thailand'),
    (6,  5,  '2024-02-10 08:55:00+07', 'delivered',  'UK'),
    (7,  6,  '2024-02-14 19:00:00+07', 'cancelled',  'Thailand'),
    (8,  7,  '2024-02-18 13:10:00+07', 'delivered',  'France'),
    (9,  8,  '2024-02-25 09:40:00+07', 'shipped',    'China'),
    (10, 9,  '2024-03-01 15:25:00+07', 'delivered',  'Thailand'),
    (11, 1,  '2024-03-05 10:00:00+07', 'processing', 'Thailand'),
    (12, 10, '2024-03-09 12:30:00+07', 'delivered',  'Spain'),
    (13, 11, '2024-03-12 17:15:00+07', 'delivered',  'Thailand'),
    (14, 4,  '2024-03-18 08:20:00+07', 'delivered',  'Thailand'),
    (15, 12, '2024-03-22 14:00:00+07', 'shipped',    'Ireland'),
    (16, 13, '2024-03-27 11:45:00+07', 'delivered',  'Thailand'),
    (17, 9,  '2024-04-02 09:30:00+07', 'delivered',  'Thailand'),
    (18, 14, '2024-04-06 16:10:00+07', 'pending',    'Germany'),
    (19, 15, '2024-04-10 10:50:00+07', 'delivered',  'Thailand'),
    (20, 6,  '2024-04-15 13:35:00+07', 'delivered',  'Thailand');
SELECT setval('orders_order_id_seq', 20);

-- order_items
INSERT INTO order_items (order_item_id, order_id, product_id, quantity, unit_price) VALUES
    (1,  1,  1,  1, 32900.00),
    (2,  1,  4,  1, 3200.00),
    (3,  2,  2,  1, 24900.00),
    (4,  2,  13, 2, 490.00),
    (5,  3,  6,  3, 350.00),
    (6,  3,  17, 2, 420.00),
    (7,  4,  3,  1, 2590.00),
    (8,  4,  12, 1, 1490.00),
    (9,  5,  5,  1, 8900.00),
    (10, 5,  18, 1, 1590.00),
    (11, 6,  7,  4, 590.00),
    (12, 7,  9,  1, 990.00),
    (13, 8,  8,  2, 1290.00),
    (14, 8,  14, 3, 350.00),
    (15, 9,  20, 1, 3990.00),
    (16, 10, 11, 2, 850.00),
    (17, 10, 10, 1, 690.00),
    (18, 11, 1,  1, 32900.00),
    (19, 12, 15, 1, 2890.00),
    (20, 12, 3,  2, 2590.00),
    (21, 13, 6,  5, 350.00),
    (22, 13, 16, 1, 2450.00),
    (23, 14, 2,  1, 24900.00),
    (24, 14, 13, 1, 490.00),
    (25, 15, 4,  2, 3200.00),
    (26, 16, 9,  1, 990.00),
    (27, 16, 11, 1, 850.00),
    (28, 17, 1,  1, 32900.00),
    (29, 17, 5,  1, 8900.00),
    (30, 17, 18, 2, 1590.00),
    (31, 18, 8,  1, 1290.00),
    (32, 19, 3,  3, 2590.00),
    (33, 19, 12, 2, 1490.00),
    (34, 20, 7,  2, 590.00),
    (35, 20, 17, 1, 420.00);
SELECT setval('order_items_order_item_id_seq', 35);
```

ตรวจสอบว่าข้อมูลเข้าครบถ้วน:

```sql
SELECT
    (SELECT count(*) FROM categories)  AS categories,
    (SELECT count(*) FROM suppliers)   AS suppliers,
    (SELECT count(*) FROM products)    AS products,
    (SELECT count(*) FROM customers)   AS customers,
    (SELECT count(*) FROM orders)      AS orders,
    (SELECT count(*) FROM order_items) AS order_items;
```

```
 categories | suppliers | products | customers | orders | order_items
------------+-----------+----------+-----------+--------+-------------
         12 |        10 |       20 |        15 |     20 |          35
```

---

## Step 451: PL/pgSQL คืออะไร

SQL เป็นภาษาที่ทรงพลังมากสำหรับการ "ถาม" คำถามกับข้อมูล (declarative language) แต่ SQL ล้วน ๆ มีข้อจำกัดเมื่อเราต้องการเขียน **ตรรกะแบบขั้นตอน** (procedural logic) เช่น:

- การวนลูป (loop) เพื่อประมวลผลทีละแถว
- เงื่อนไขซับซ้อนที่ต้องตัดสินใจหลายชั้น (`IF...ELSIF...ELSE`)
- การเก็บค่าตัวแปรชั่วคราวระหว่างขั้นตอนการคำนวณ
- การจัดการ error/exception อย่างละเอียด
- การรวมหลายคำสั่ง SQL เข้าด้วยกันเป็น "หน่วยตรรกะทางธุรกิจ" หนึ่งเดียว แล้วเรียกใช้ซ้ำได้

**PL/pgSQL** (Procedural Language/PostgreSQL) คือภาษาโปรแกรมเชิงกระบวนการ (procedural language) ที่ฝังอยู่ใน PostgreSQL โดยตรง ออกแบบมาให้ผสาน SQL เข้ากับโครงสร้างการเขียนโปรแกรมทั่วไป (ตัวแปร, loop, condition, exception handling) ทำให้เราสามารถเขียน business logic ที่ซับซ้อนให้ทำงาน **อยู่ภายในฐานข้อมูลเอง** แทนที่จะต้องดึงข้อมูลออกไปประมวลผลใน application แล้วส่งกลับมาหลายรอบ

### เหตุผลที่ควรใช้ PL/pgSQL

1. **ลด round-trip ระหว่าง application กับฐานข้อมูล** — แทนที่จะ query 5 ครั้งจาก backend แล้วประมวลผลใน Python/Node.js แล้วเขียนกลับ เราสามารถรวมตรรกะทั้งหมดไว้ในฟังก์ชันเดียว เรียกครั้งเดียวจบ
2. **บังคับ business rule ให้เป็นมาตรฐานเดียวกัน** — ไม่ว่าจะเรียกจาก application ไหน (web, mobile, batch job) กฎทางธุรกิจก็จะเหมือนกันเสมอ เพราะ logic อยู่ที่ฐานข้อมูล
3. **ประสิทธิภาพ** — คำนวณข้างใน database engine โดยตรง ไม่ต้องส่งข้อมูลจำนวนมากข้ามเครือข่าย
4. **Transaction control ที่แม่นยำ** — โดยเฉพาะกับ `PROCEDURE` ที่ควบคุม commit/rollback เองได้ (จะเรียนใน Step 458)
5. **ใช้ร่วมกับ trigger ได้** — logic ที่ซับซ้อนสำหรับ trigger (จะเรียนในบทถัดไป) ต้องเขียนด้วยภาษา procedural แบบนี้

### ตัวอย่าง: ปัญหาที่ SQL ล้วนทำได้ยาก

สมมติเราต้องการ "จัดระดับลูกค้า" ตามยอดซื้อสะสม โดยมีเงื่อนไขหลายชั้นซ้อนกัน — SQL ล้วนสามารถทำได้ด้วย `CASE WHEN` แต่เมื่อ logic ซับซ้อนขึ้นเรื่อย ๆ (เช่น ต้องเช็คหลายเงื่อนไขที่พึ่งพากัน มีการคำนวณหลายขั้นตอน) การเขียนด้วย PL/pgSQL จะอ่านง่ายและดูแลรักษาง่ายกว่ามาก

ลองดูตัวอย่าง **anonymous code block** (`DO`) ซึ่งเป็นวิธีทดลองรัน PL/pgSQL แบบไม่ต้องสร้างฟังก์ชันถาวรก่อน:

```sql
DO $$
DECLARE
    v_customer_id   INTEGER := 1;
    v_total_spent   NUMERIC(12,2);
    v_tier          TEXT;
BEGIN
    -- คำนวณยอดซื้อสะสมของลูกค้า
    SELECT COALESCE(SUM(oi.quantity * oi.unit_price), 0)
      INTO v_total_spent
      FROM orders o
      JOIN order_items oi ON oi.order_id = o.order_id
     WHERE o.customer_id = v_customer_id
       AND o.status <> 'cancelled';

    -- ตัดสินใจแบบหลายเงื่อนไข
    IF v_total_spent >= 50000 THEN
        v_tier := 'Platinum';
    ELSIF v_total_spent >= 20000 THEN
        v_tier := 'Gold';
    ELSIF v_total_spent >= 5000 THEN
        v_tier := 'Silver';
    ELSE
        v_tier := 'Bronze';
    END IF;

    RAISE NOTICE 'Customer % spent % -> tier %', v_customer_id, v_total_spent, v_tier;
END;
$$ LANGUAGE plpgsql;
```

ผลลัพธ์ (แสดงผ่าน `NOTICE`):

```
NOTICE:  Customer 1 spent 70510.00 -> tier Platinum
DO
```

สังเกตว่า `DO` block รันครั้งเดียวแล้วจบ ไม่สามารถเรียกซ้ำได้เหมือนฟังก์ชัน — มันเหมาะสำหรับทดสอบ logic หรือรัน script แบบ one-off เท่านั้น ในหัวข้อถัดไปเราจะเปลี่ยน logic นี้ให้เป็นฟังก์ชันถาวรที่เรียกใช้ซ้ำได้

> **หมายเหตุ:** PostgreSQL รองรับภาษา procedural อื่น ๆ ด้วย เช่น PL/Python, PL/Perl, PL/Tcl (ต้อง `CREATE EXTENSION` เพิ่ม) แต่ **PL/pgSQL เป็นค่าเริ่มต้นที่ติดตั้งมาพร้อม PostgreSQL เสมอ** และเป็นตัวเลือกที่แนะนำที่สุดสำหรับ logic ที่เกี่ยวข้องกับข้อมูลโดยตรง เพราะผสาน SQL ได้แนบเนียนที่สุด

---

## Step 452: CREATE FUNCTION พื้นฐาน syntax

โครงสร้างพื้นฐานของ `CREATE FUNCTION` มีดังนี้:

```sql
CREATE [OR REPLACE] FUNCTION function_name(param1 type1, param2 type2, ...)
RETURNS return_type
LANGUAGE plpgsql
AS $$
DECLARE
    -- ประกาศตัวแปร (ถ้ามี)
BEGIN
    -- โค้ด logic
    RETURN some_value;
END;
$$;
```

ส่วนประกอบสำคัญ:

- **`CREATE OR REPLACE FUNCTION`** — ใช้ `OR REPLACE` เสมอในระหว่างพัฒนา เพื่อแก้ไขฟังก์ชันเดิมได้โดยไม่ต้อง `DROP` ก่อน (ข้อควรระวัง: ถ้าเปลี่ยนชื่อ/ประเภทพารามิเตอร์ หรือ return type จะต้อง `DROP FUNCTION` ก่อนเสมอ)
- **พารามิเตอร์** — ระบุชื่อและ type ของค่าที่รับเข้า
- **`RETURNS`** — ประเภทข้อมูลที่ฟังก์ชันจะคืนกลับ
- **`LANGUAGE plpgsql`** — บอก PostgreSQL ว่าฟังก์ชันนี้เขียนด้วยภาษาอะไร
- **`$$ ... $$`** — เรียกว่า **dollar quoting** ใช้ล้อมโค้ดของฟังก์ชันแทนการใช้ single quote (`'...'`) เพราะโค้ดภายในอาจมี string literal ที่มี quote ซ้อนกันอยู่แล้ว การใช้ `$$` ช่วยไม่ให้ต้อง escape quote ซ้ำซ้อน (สามารถตั้งชื่อ tag ได้ เช่น `$body$ ... $body$` เมื่อโค้ดภายในมี `$$` ซ้อนกันเอง)

### ตัวอย่างฟังก์ชันแรก: get_product_price

```sql
CREATE OR REPLACE FUNCTION get_product_price(p_product_id INTEGER)
RETURNS NUMERIC
LANGUAGE plpgsql
AS $$
BEGIN
    RETURN (
        SELECT unit_price
          FROM products
         WHERE product_id = p_product_id
    );
END;
$$;
```

เรียกใช้งาน:

```sql
SELECT get_product_price(1);
```

```
 get_product_price
--------------------
           32900.00
(1 row)
```

หรือเรียกใช้ร่วมกับตารางอื่นได้เหมือนฟังก์ชันในตัว (built-in function):

```sql
SELECT product_id, product_name, get_product_price(product_id) AS price
  FROM products
 WHERE category_id = 4;
```

```
 product_id |      product_name      |  price
------------+-------------------------+----------
          3 | Wireless Earbuds Z1     |  2590.00
         12 | Bluetooth Speaker Mini  |  1490.00
(2 rows)
```

### ตัวอย่างฟังก์ชันที่รับหลายพารามิเตอร์

```sql
CREATE OR REPLACE FUNCTION calculate_line_total(p_quantity INTEGER, p_unit_price NUMERIC)
RETURNS NUMERIC
LANGUAGE plpgsql
AS $$
BEGIN
    RETURN p_quantity * p_unit_price;
END;
$$;
```

```sql
SELECT order_item_id, quantity, unit_price,
       calculate_line_total(quantity, unit_price) AS line_total
  FROM order_items
 WHERE order_id = 1;
```

```
 order_item_id | quantity | unit_price | line_total
---------------+----------+------------+------------
             1 |        1 |   32900.00 |   32900.00
             2 |        1 |    3200.00 |    3200.00
(2 rows)
```

### การลบฟังก์ชัน

```sql
DROP FUNCTION IF EXISTS calculate_line_total(INTEGER, NUMERIC);
```

ต้องระบุ signature (ประเภทพารามิเตอร์) ให้ตรงด้วย เพราะ PostgreSQL รองรับ **function overloading** — สามารถมีฟังก์ชันชื่อเดียวกันหลายตัวที่รับพารามิเตอร์ต่างกันได้ (จะกล่าวถึงรายละเอียดเพิ่มในบทถัดไป)

---

## Step 453: ตัวแปร (DECLARE), การกำหนดค่า, %TYPE

ภายใน `DECLARE` block เราสามารถประกาศตัวแปรพร้อมระบุ type ได้ และกำหนดค่าเริ่มต้นได้ด้วย `:=` หรือ `DEFAULT`

```sql
DECLARE
    v_count      INTEGER := 0;
    v_name       TEXT DEFAULT 'unknown';
    v_price      NUMERIC(10,2);
    v_is_valid   BOOLEAN := false;
```

### การกำหนดค่าให้ตัวแปร

มี 2 วิธีหลัก:

```sql
-- 1) กำหนดค่าตรง ๆ
v_count := v_count + 1;

-- 2) ดึงค่าจาก query ด้วย SELECT ... INTO
SELECT count(*) INTO v_count FROM orders;
```

### %TYPE — อ้างอิง type จากคอลัมน์จริง

ปัญหาที่พบบ่อยคือ ถ้าเราประกาศตัวแปรด้วย type ตายตัว (เช่น `NUMERIC(10,2)`) แล้ววันหนึ่ง column ในตารางถูกแก้เป็น `NUMERIC(12,4)` โค้ดในฟังก์ชันจะไม่ sync กับ schema ทันที (แม้จะยังทำงานได้ แต่เสี่ยงต่อการ mismatch ในระยะยาว)

`%TYPE` แก้ปัญหานี้โดยให้ PostgreSQL **ดึง type จากคอลัมน์จริงในตารางโดยอัตโนมัติ** ทุกครั้งที่ฟังก์ชันถูกคอมไพล์ใหม่ ทำให้ type ของตัวแปรตรงกับ schema เสมอ:

```sql
DECLARE
    v_price      products.unit_price%TYPE;   -- type เดียวกับคอลัมน์ unit_price
    v_customer_name   customers.first_name%TYPE;
```

### ตัวอย่างฟังก์ชันที่ใช้ตัวแปรและ %TYPE ครบถ้วน

```sql
CREATE OR REPLACE FUNCTION get_discounted_price(
    p_product_id  INTEGER,
    p_discount_pct NUMERIC DEFAULT 0
)
RETURNS NUMERIC
LANGUAGE plpgsql
AS $$
DECLARE
    v_original_price   products.unit_price%TYPE;   -- ดึง type จริงจากคอลัมน์
    v_final_price       NUMERIC(10,2);
BEGIN
    -- ดึงราคาต้นฉบับด้วย SELECT ... INTO
    SELECT unit_price
      INTO v_original_price
      FROM products
     WHERE product_id = p_product_id;

    -- ถ้าไม่พบสินค้า ให้คืนค่า NULL
    IF v_original_price IS NULL THEN
        RETURN NULL;
    END IF;

    -- คำนวณราคาหลังหักส่วนลด
    v_final_price := v_original_price * (1 - p_discount_pct / 100.0);

    RETURN ROUND(v_final_price, 2);
END;
$$;
```

ทดลองเรียกใช้:

```sql
SELECT get_discounted_price(1, 10);      -- ลด 10%
SELECT get_discounted_price(6, 25);      -- ลด 25%
SELECT get_discounted_price(999, 10);    -- ไม่มีสินค้านี้
```

```
 get_discounted_price
-----------------------
              29610.00
(1 row)

 get_discounted_price
-----------------------
                262.50
(1 row)

 get_discounted_price
-----------------------
                   NULL
(1 row)
```

> **แนวทางปฏิบัติที่ดี (Best Practice):** ควรใช้ `%TYPE` ทุกครั้งที่ตัวแปรมีความเกี่ยวข้องโดยตรงกับคอลัมน์ในตาราง โดยเฉพาะกับ `NUMERIC(p,s)`, `VARCHAR(n)` ที่มักถูกปรับแก้ตามความต้องการทางธุรกิจในอนาคต

---

## Step 454: RETURN — ฟังก์ชันคืนค่าเดียว (scalar) และการเรียกใช้ใน SELECT

ฟังก์ชันที่คืนค่าเดียว (scalar function) คือฟังก์ชันพื้นฐานที่สุด — `RETURN` จะหยุดการทำงานของฟังก์ชันทันทีและส่งค่ากลับ

### ตัวอย่าง: นับจำนวนออเดอร์ของลูกค้า

```sql
CREATE OR REPLACE FUNCTION count_customer_orders(p_customer_id INTEGER)
RETURNS INTEGER
LANGUAGE plpgsql
AS $$
DECLARE
    v_order_count INTEGER;
BEGIN
    SELECT count(*)
      INTO v_order_count
      FROM orders
     WHERE customer_id = p_customer_id;

    RETURN v_order_count;
END;
$$;
```

```sql
SELECT count_customer_orders(1);
```

```
 count_customer_orders
------------------------
                      3
(1 row)
```

### เรียกใช้ในบริบทต่าง ๆ ของ SELECT

ฟังก์ชันที่คืนค่าเดียวสามารถใช้ได้เหมือนคอลัมน์ปกติ ทั้งใน `SELECT list`, `WHERE`, `ORDER BY`:

```sql
-- ใช้ใน SELECT list ควบคู่กับ JOIN
SELECT customer_id, first_name, last_name,
       count_customer_orders(customer_id) AS total_orders
  FROM customers
 ORDER BY total_orders DESC
 LIMIT 5;
```

```
 customer_id | first_name | last_name | total_orders
-------------+------------+-----------+---------------
           1 | Somchai    | Jaidee    |             3
           4 | Nattaya    | Suksawat  |             2
           9 | Pimchanok  | Rattana   |             2
           6 | Chalermchai| Boon      |             2
           2 | Emily      | Watson    |             1
(5 rows)
```

```sql
-- ใช้ใน WHERE clause (กรองลูกค้าที่มีออเดอร์มากกว่า 1 ครั้ง)
SELECT customer_id, first_name, last_name
  FROM customers
 WHERE count_customer_orders(customer_id) > 1;
```

```
 customer_id | first_name |  last_name
-------------+------------+-------------
           1 | Somchai    | Jaidee
           4 | Nattaya    | Suksawat
           6 | Chalermchai| Boon
           9 | Pimchanok  | Rattana
(4 rows)
```

> **ข้อควรระวังด้านประสิทธิภาพ:** การเรียกฟังก์ชันใน `WHERE` แบบนี้จะทำให้ PostgreSQL ต้องรันฟังก์ชัน (ซึ่งข้างในมี query ย่อย) **สำหรับทุกแถว** ของตาราง (row-by-row) แทนที่จะ optimize เป็น JOIN/aggregate เดียว ในตารางขนาดใหญ่อาจช้ากว่าการเขียน SQL ตรง ๆ มาก เราจะพูดถึงเรื่องนี้ลึกขึ้นใน Step 457 (volatility) และเรื่อง query optimization ในบทถัดไป — สำหรับตารางขนาดเล็กแบบในบทนี้ยังไม่มีปัญหา

### RETURN แบบมีเงื่อนไขหลายจุด (early return)

ฟังก์ชันสามารถมี `RETURN` ได้หลายจุด โดยจุดแรกที่ถูก execute จะหยุดฟังก์ชันทันที:

```sql
CREATE OR REPLACE FUNCTION get_stock_status(p_product_id INTEGER)
RETURNS TEXT
LANGUAGE plpgsql
AS $$
DECLARE
    v_stock   products.stock_quantity%TYPE;
    v_active  products.is_active%TYPE;
BEGIN
    SELECT stock_quantity, is_active
      INTO v_stock, v_active
      FROM products
     WHERE product_id = p_product_id;

    IF NOT FOUND THEN
        RETURN 'ไม่พบสินค้า';
    END IF;

    IF NOT v_active THEN
        RETURN 'สินค้าถูกปิดการขาย';
    END IF;

    IF v_stock = 0 THEN
        RETURN 'สินค้าหมด';
    ELSIF v_stock < 30 THEN
        RETURN 'สินค้าเหลือน้อย';
    END IF;

    RETURN 'มีสินค้าเพียงพอ';
END;
$$;
```

```sql
SELECT product_id, product_name, get_stock_status(product_id) AS status
  FROM products
 WHERE product_id IN (1, 19, 5, 999);
```

```
 product_id |    product_name    |        status
------------+---------------------+------------------------
          1 | UltraBook Pro 15    | สินค้าเหลือน้อย
         19 | Winter Coat Wool    | สินค้าถูกปิดการขาย
          5 | 4K Monitor 27"      | สินค้าเหลือน้อย
(3 rows)
```

สังเกตว่า `product_id = 999` ไม่มีในผลลัพธ์เลย เพราะ `WHERE product_id IN (...)` กรองออกไปตั้งแต่ระดับตาราง (ฟังก์ชันไม่ได้ถูกเรียกด้วยซ้ำสำหรับแถวที่ไม่มีอยู่) — ตัวแปร `FOUND` เป็น special variable ของ PL/pgSQL ที่บอกว่าคำสั่ง `SELECT ... INTO` ก่อนหน้าเจอแถวหรือไม่ ใช้ตรวจสอบกรณีที่ query ไม่คืนผลลัพธ์เลย

---

## Step 455: RETURNS TABLE และ RETURNS SETOF — ฟังก์ชันที่คืนหลายแถว

บางครั้งเราต้องการฟังก์ชันที่คืนผลลัพธ์เป็น **ชุดข้อมูลหลายแถว** (เหมือนเป็น view หนึ่งที่รับพารามิเตอร์ได้) PostgreSQL มี 2 วิธีหลักในการทำสิ่งนี้

### วิธีที่ 1: RETURNS TABLE

กำหนดโครงสร้างคอลัมน์ของผลลัพธ์ไว้ล่วงหน้าใน `RETURNS TABLE(...)` แล้วใช้ `RETURN QUERY` เพื่อคืนผลลัพธ์จาก query:

```sql
CREATE OR REPLACE FUNCTION get_products_by_category(p_category_id INTEGER)
RETURNS TABLE (
    product_id     INTEGER,
    product_name   VARCHAR(150),
    unit_price     NUMERIC(10,2),
    stock_quantity INTEGER
)
LANGUAGE plpgsql
AS $$
BEGIN
    RETURN QUERY
    SELECT p.product_id, p.product_name, p.unit_price, p.stock_quantity
      FROM products p
     WHERE p.category_id = p_category_id
       AND p.is_active = true
     ORDER BY p.unit_price DESC;
END;
$$;
```

เรียกใช้งานเหมือน table function ทั่วไป (ใช้ใน `FROM`):

```sql
SELECT * FROM get_products_by_category(2);
```

```
 product_id |      product_name       | unit_price | stock_quantity
------------+---------------------------+------------+----------------
          1 | UltraBook Pro 15          |   32900.00 |             45
         20 | Portable SSD 1TB          |    3990.00 |             60
          5 | 4K Monitor 27"            |    8900.00 |             30
          4 | Mechanical Keyboard K80   |    3200.00 |             80
         18 | Gaming Mouse RGB          |    1590.00 |            130
(5 rows)
```

### วิธีที่ 2: RETURNS SETOF

`RETURNS SETOF` ใช้เมื่อโครงสร้างผลลัพธ์ตรงกับ **table type ที่มีอยู่แล้ว** เช่น `SETOF products` หมายถึง "คืนแถวหลายแถวที่มีโครงสร้างเหมือนตาราง `products` ทุกคอลัมน์"

```sql
CREATE OR REPLACE FUNCTION get_low_stock_products(p_threshold INTEGER DEFAULT 50)
RETURNS SETOF products
LANGUAGE plpgsql
AS $$
BEGIN
    RETURN QUERY
    SELECT *
      FROM products
     WHERE stock_quantity < p_threshold
       AND is_active = true
     ORDER BY stock_quantity ASC;
END;
$$;
```

```sql
SELECT product_id, product_name, stock_quantity
  FROM get_low_stock_products(50);
```

```
 product_id |      product_name       | stock_quantity
------------+---------------------------+-----------------
         19 | Winter Coat Wool          |              25  -- (ถูกกรองออกเพราะ is_active=false จริง ๆ)
          5 | 4K Monitor 27"            |              30
          1 | UltraBook Pro 15          |              45
(2 rows)
```

> หมายเหตุ: `Winter Coat Wool` มี `is_active = false` จึงถูกกรองออกจากผลลัพธ์จริง ผลลัพธ์ที่ถูกต้องคือ:

```
 product_id |    product_name    | stock_quantity
------------+---------------------+-----------------
          5 | 4K Monitor 27"      |              30
          1 | UltraBook Pro 15    |              45
(2 rows)
```

### ความแตกต่างระหว่าง RETURNS TABLE กับ RETURNS SETOF

| ประเด็น | `RETURNS TABLE` | `RETURNS SETOF <table>` |
|---|---|---|
| การกำหนดโครงสร้าง | กำหนดคอลัมน์เองอิสระ ไม่ต้องตรงกับ table จริง | ใช้โครงสร้างจาก table/composite type ที่มีอยู่แล้วทุกคอลัมน์ |
| ความยืดหยุ่น | เลือกได้ว่าจะคืนคอลัมน์ไหนบ้าง, ตั้งชื่อใหม่ได้ | ต้องคืนครบทุกคอลัมน์ตาม type ต้นทาง (หรือใช้ `SETOF record` ร่วมกับ `OUT` parameter) |
| กรณีใช้งาน | สร้างผลลัพธ์ custom ที่รวมข้อมูลจากหลายตาราง | คืนแถวจากตารางเดียวโดยตรง หรือใช้ scalar type เช่น `SETOF INTEGER` |

### RETURNS SETOF กับ scalar type

`SETOF` ยังใช้กับ scalar type ได้ด้วย เช่น คืนรายการ `customer_id` ที่เป็นลูกค้า VIP:

```sql
CREATE OR REPLACE FUNCTION get_vip_customer_ids(p_min_orders INTEGER DEFAULT 2)
RETURNS SETOF INTEGER
LANGUAGE plpgsql
AS $$
BEGIN
    RETURN QUERY
    SELECT customer_id
      FROM orders
     WHERE status <> 'cancelled'
     GROUP BY customer_id
    HAVING count(*) >= p_min_orders;
END;
$$;
```

```sql
SELECT * FROM get_vip_customer_ids(2);
```

```
 get_vip_customer_ids
-----------------------
                     1
                     4
                     9
                     6
(4 rows)
```

### RETURN QUERY กับหลาย statement (loop)

`RETURN QUERY` สามารถถูกเรียกได้หลายครั้งในฟังก์ชันเดียว ผลลัพธ์จะถูกสะสม (accumulate) เข้าด้วยกัน — มีประโยชน์เมื่อ logic การดึงข้อมูลซับซ้อนกว่า query เดียว:

```sql
CREATE OR REPLACE FUNCTION get_products_needing_attention()
RETURNS TABLE (product_id INTEGER, product_name VARCHAR(150), reason TEXT)
LANGUAGE plpgsql
AS $$
BEGIN
    -- กลุ่มที่ 1: สินค้าหมด
    RETURN QUERY
    SELECT p.product_id, p.product_name, 'สินค้าหมด'::TEXT
      FROM products p
     WHERE p.stock_quantity = 0 AND p.is_active = true;

    -- กลุ่มที่ 2: สินค้าที่ถูกปิดการขายแต่ยังมี stock ค้างอยู่
    RETURN QUERY
    SELECT p.product_id, p.product_name, 'ปิดการขายแต่ยังมี stock'::TEXT
      FROM products p
     WHERE p.is_active = false AND p.stock_quantity > 0;
END;
$$;
```

```sql
SELECT * FROM get_products_needing_attention();
```

```
 product_id |   product_name    |           reason
------------+---------------------+------------------------------
         19 | Winter Coat Wool    | ปิดการขายแต่ยังมี stock
(1 row)
```

---

## Step 456: IF/ELSIF/ELSE conditional logic ภายในฟังก์ชัน

PL/pgSQL รองรับ conditional statement ครบถ้วน ทั้งแบบ `IF`, `IF/ELSE`, และ `IF/ELSIF/ELSE`

### รูปแบบ syntax

```sql
IF condition THEN
    statements;
ELSIF another_condition THEN
    statements;
ELSE
    statements;
END IF;
```

**ข้อควรระวัง:** PL/pgSQL ใช้ `ELSIF` (ไม่มี "E" อย่าง `ELSEIF` แบบภาษาอื่น) และต้องปิดด้วย `END IF;` เสมอ

### ตัวอย่าง: จัดระดับ (tier) ลูกค้าตามยอดซื้อสะสม

```sql
CREATE OR REPLACE FUNCTION get_customer_tier(p_customer_id INTEGER)
RETURNS TEXT
LANGUAGE plpgsql
AS $$
DECLARE
    v_total_spent NUMERIC(12,2);
    v_tier        TEXT;
BEGIN
    SELECT COALESCE(SUM(oi.quantity * oi.unit_price), 0)
      INTO v_total_spent
      FROM orders o
      JOIN order_items oi ON oi.order_id = o.order_id
     WHERE o.customer_id = p_customer_id
       AND o.status <> 'cancelled';

    IF v_total_spent >= 50000 THEN
        v_tier := 'Platinum';
    ELSIF v_total_spent >= 20000 THEN
        v_tier := 'Gold';
    ELSIF v_total_spent >= 5000 THEN
        v_tier := 'Silver';
    ELSIF v_total_spent > 0 THEN
        v_tier := 'Bronze';
    ELSE
        v_tier := 'No Purchase';
    END IF;

    RETURN v_tier;
END;
$$;
```

```sql
SELECT customer_id, first_name, last_name,
       get_customer_tier(customer_id) AS tier
  FROM customers
 ORDER BY customer_id
 LIMIT 8;
```

```
 customer_id | first_name |  last_name  |    tier
-------------+------------+-------------+-------------
           1 | Somchai    | Jaidee      | Platinum
           2 | Emily      | Watson      | Gold
           3 | Yuki       | Tanaka      | Bronze
           4 | Nattaya    | Suksawat    | Platinum
           5 | James      | Miller      | Bronze
           6 | Chalermchai| Boon        | Silver
           7 | Sophie     | Dubois      | No Purchase
           8 | Wei        | Zhang       | Silver
(8 rows)
```

### เงื่อนไขซ้อนกัน (nested IF) และ CASE expression

บางครั้ง logic ซับซ้อนพอที่จะต้องมี `IF` ซ้อนกันหลายชั้น หรือใช้ `CASE` expression ผสมก็ได้ (PL/pgSQL รองรับทั้ง `CASE` แบบ statement และแบบ expression เหมือน SQL ทั่วไป):

```sql
CREATE OR REPLACE FUNCTION get_shipping_note(p_order_id INTEGER)
RETURNS TEXT
LANGUAGE plpgsql
AS $$
DECLARE
    v_status        orders.status%TYPE;
    v_ship_country  orders.ship_country%TYPE;
    v_note          TEXT;
BEGIN
    SELECT status, ship_country
      INTO v_status, v_ship_country
      FROM orders
     WHERE order_id = p_order_id;

    IF NOT FOUND THEN
        RETURN 'ไม่พบออเดอร์นี้';
    END IF;

    IF v_status = 'delivered' THEN
        v_note := 'จัดส่งสำเร็จแล้ว';
    ELSIF v_status = 'cancelled' THEN
        v_note := 'ออเดอร์ถูกยกเลิก';
    ELSIF v_status IN ('shipped', 'processing') THEN
        -- เงื่อนไขซ้อนภายใน branch นี้
        IF v_ship_country = 'Thailand' THEN
            v_note := 'กำลังจัดส่งในประเทศ (คาดว่าถึงใน 2-3 วัน)';
        ELSE
            v_note := 'กำลังจัดส่งระหว่างประเทศ (คาดว่าถึงใน 7-14 วัน)';
        END IF;
    ELSE
        v_note := 'รอดำเนินการ';
    END IF;

    RETURN v_note;
END;
$$;
```

```sql
SELECT order_id, status, ship_country, get_shipping_note(order_id) AS note
  FROM orders
 WHERE order_id IN (9, 15, 18, 7);
```

```
 order_id |   status   | ship_country |                          note
----------+------------+--------------+----------------------------------------------------------
        9 | shipped    | China        | กำลังจัดส่งระหว่างประเทศ (คาดว่าถึงใน 7-14 วัน)
       15 | shipped    | Ireland      | กำลังจัดส่งระหว่างประเทศ (คาดว่าถึงใน 7-14 วัน)
       18 | pending    | Germany      | รอดำเนินการ
        7 | cancelled  | Thailand     | ออเดอร์ถูกยกเลิก
(4 rows)
```

---

## Step 457: Function Volatility — IMMUTABLE, STABLE, VOLATILE

**Volatility category** คือการบอก query planner ว่าฟังก์ชันของเรา "เปลี่ยนแปลงผลลัพธ์ได้บ่อยแค่ไหน" เมื่อถูกเรียกด้วย input เดียวกัน ข้อมูลนี้สำคัญมากเพราะ planner ใช้ตัดสินใจว่าจะ **cache ผลลัพธ์** หรือ **optimize การเรียกฟังก์ชันซ้ำ** ได้หรือไม่

PostgreSQL มี 3 ระดับ:

| Volatility | ความหมาย | ตัวอย่าง |
|---|---|---|
| `IMMUTABLE` | ผลลัพธ์เหมือนเดิมเสมอสำหรับ input เดียวกัน ไม่ query ตาราง ไม่ขึ้นกับสภาพแวดล้อม | ฟังก์ชันคำนวณคณิตศาสตร์ล้วน ๆ เช่น `calculate_line_total` |
| `STABLE` | ผลลัพธ์เหมือนเดิมภายใน **query เดียวกัน/transaction เดียวกัน** แต่อาจเปลี่ยนได้ระหว่าง query ต่างครั้ง (เพราะข้อมูลในตารางเปลี่ยนได้) | ฟังก์ชันที่ `SELECT` อ่านค่าจากตาราง เช่น `get_product_price` |
| `VOLATILE` (ค่า default) | ผลลัพธ์อาจเปลี่ยนได้ทุกครั้งที่เรียก แม้ input เดิมและอยู่ใน query เดียวกัน | ฟังก์ชันที่ใช้ `random()`, `now()`, หรือมีการ `INSERT/UPDATE/DELETE` |

### ทำไม volatility ถึงสำคัญ

1. **Query optimization**: ถ้า planner รู้ว่าฟังก์ชันเป็น `IMMUTABLE`/`STABLE` มันสามารถ **เรียกฟังก์ชันแค่ครั้งเดียว** แล้วนำผลไปใช้ซ้ำ แทนที่จะเรียกทุกแถว (ถ้าเงื่อนไขเหมาะสม) ช่วยเพิ่มประสิทธิภาพได้มาก
2. **Index บน expression**: ฟังก์ชันที่จะใช้สร้าง **functional index** (`CREATE INDEX ON table (function(column))`) **ต้องเป็น `IMMUTABLE` เท่านั้น** เพราะ index ต้องมั่นใจได้ว่าค่าที่คำนวณไว้จะไม่เปลี่ยนแปลงในภายหลัง
3. **ความถูกต้องของผลลัพธ์**: ถ้าประกาศ volatility ผิด (เช่น บอกว่าฟังก์ชันที่ query ตารางเป็น `IMMUTABLE`) planner อาจ cache ผลลัพธ์ผิด ๆ ทำให้ query คืนค่าเก่าที่ไม่ตรงกับข้อมูลปัจจุบัน — **นี่คือบั๊กที่ตรวจจับยากมาก** เพราะทุกอย่างดูทำงานถูกในตอนทดสอบครั้งแรก

### ตัวอย่าง IMMUTABLE

```sql
CREATE OR REPLACE FUNCTION calculate_line_total(p_quantity INTEGER, p_unit_price NUMERIC)
RETURNS NUMERIC
LANGUAGE plpgsql
IMMUTABLE
AS $$
BEGIN
    RETURN p_quantity * p_unit_price;
END;
$$;
```

ฟังก์ชันนี้ไม่แตะตารางใด ๆ เลย คำนวณจาก input ล้วน ๆ เหมาะที่จะเป็น `IMMUTABLE` — สามารถใช้สร้าง functional index ได้ด้วย เช่น:

```sql
-- ตัวอย่าง (ไม่จำเป็นต้องรันจริงในบทนี้): index บนค่าที่คำนวณจากฟังก์ชัน immutable
-- CREATE INDEX idx_order_items_line_total
--     ON order_items (calculate_line_total(quantity, unit_price));
```

### ตัวอย่าง STABLE

```sql
CREATE OR REPLACE FUNCTION get_product_price(p_product_id INTEGER)
RETURNS NUMERIC
LANGUAGE plpgsql
STABLE
AS $$
BEGIN
    RETURN (
        SELECT unit_price
          FROM products
         WHERE product_id = p_product_id
    );
END;
$$;
```

ฟังก์ชันนี้อ่านค่าจากตาราง `products` — ถ้าเรียกซ้ำด้วย `product_id` เดียวกัน **ภายใน query เดียวกัน** ผลจะเหมือนเดิมเสมอ (เพราะ PostgreSQL ใช้ MVCC snapshot เดียวกันตลอด statement) จึงเหมาะเป็น `STABLE` — planner สามารถ optimize การเรียกซ้ำภายใน statement เดียวกันได้ แต่จะไม่ cache ข้าม statement หรือข้าม transaction

### ตัวอย่าง VOLATILE (ค่า default)

```sql
CREATE OR REPLACE FUNCTION log_stock_check(p_product_id INTEGER)
RETURNS TEXT
LANGUAGE plpgsql
VOLATILE   -- จริง ๆ ไม่จำเป็นต้องเขียนเพราะเป็นค่า default อยู่แล้ว
AS $$
DECLARE
    v_stock INTEGER;
BEGIN
    SELECT stock_quantity INTO v_stock
      FROM products WHERE product_id = p_product_id;

    -- ฟังก์ชันนี้มี side effect: บันทึกเวลาปัจจุบัน (สมมติว่ามีตาราง log)
    -- INSERT INTO stock_check_log (product_id, checked_at) VALUES (p_product_id, clock_timestamp());

    RETURN format('Product %s stock=%s checked at %s', p_product_id, v_stock, clock_timestamp());
END;
$$;
```

ฟังก์ชันนี้ใช้ `clock_timestamp()` (เวลาปัจจุบันแบบ real-time ที่เปลี่ยนทุกครั้งที่เรียก) และ/หรือมี side effect (`INSERT`) จึง**ต้อง**เป็น `VOLATILE` — ถ้าประกาศผิดเป็น `STABLE`/`IMMUTABLE` planner อาจไม่เรียกฟังก์ชันตามจำนวนครั้งที่ควรจะเป็นจริง ทำให้ log ขาดหายหรือเวลาไม่ถูกต้อง

### สรุปหลักการเลือก volatility

```
ฟังก์ชันมีการ INSERT/UPDATE/DELETE หรือใช้ now()/random()/clock_timestamp() ?
    ├── ใช่ → VOLATILE (ค่า default อยู่แล้ว ไม่ต้องระบุก็ได้)
    └── ไม่ใช่ → ฟังก์ชัน SELECT จากตาราง ?
                    ├── ใช่ → STABLE
                    └── ไม่ใช่ (คำนวณจาก input ล้วน ๆ) → IMMUTABLE
```

> **คำเตือนสำคัญ:** อย่าประกาศ volatility ผิด เพราะ PostgreSQL **ไม่ตรวจสอบให้** ว่าคุณประกาศตรงกับพฤติกรรมจริงหรือไม่ — เป็นหน้าที่ของผู้เขียนฟังก์ชันที่ต้องรับผิดชอบเอง การประกาศผิดอาจทำให้เกิดบั๊กที่ตรวจพบยากมากในภายหลัง

---

## Step 458: CREATE PROCEDURE เทียบกับ FUNCTION

ตั้งแต่ PostgreSQL 11 เป็นต้นมา มีคำสั่ง `CREATE PROCEDURE` แยกออกจาก `CREATE FUNCTION` โดยมีจุดประสงค์และพฤติกรรมต่างกันชัดเจน

### ความแตกต่างสำคัญ

| ประเด็น | `FUNCTION` | `PROCEDURE` |
|---|---|---|
| การเรียกใช้ | `SELECT function_name(...)` หรือใช้ใน expression | `CALL procedure_name(...)` เท่านั้น |
| ค่าที่คืน | ต้องมี `RETURNS` และ `RETURN` เสมอ (หรือ `RETURNS void`) | ไม่มี `RETURNS` — คืนค่าได้ผ่าน `INOUT` parameter เท่านั้น |
| ควบคุม transaction | **ทำไม่ได้** — ฟังก์ชันรันอยู่ภายใน transaction ของ statement ที่เรียกมันเสมอ ไม่สามารถ `COMMIT`/`ROLLBACK` เองได้ | **ทำได้** — สามารถเรียก `COMMIT`/`ROLLBACK` ภายใน procedure ได้โดยตรง (มีประโยชน์มากสำหรับ batch job ที่ประมวลผลทีละ chunk) |
| ใช้ใน SELECT | ใช้ได้ | ใช้ไม่ได้ |
| Trigger function | ใช้ได้ (`RETURNS TRIGGER`) | ใช้ไม่ได้ |

### ตัวอย่าง PROCEDURE: ประมวลผลออเดอร์ (ตัดสต๊อกสินค้า)

```sql
CREATE OR REPLACE PROCEDURE process_order_shipment(p_order_id INTEGER)
LANGUAGE plpgsql
AS $$
DECLARE
    v_item RECORD;
    v_current_status orders.status%TYPE;
BEGIN
    -- ตรวจสอบสถานะปัจจุบันของออเดอร์
    SELECT status INTO v_current_status
      FROM orders WHERE order_id = p_order_id;

    IF v_current_status IS NULL THEN
        RAISE EXCEPTION 'ไม่พบออเดอร์หมายเลข %', p_order_id;
    END IF;

    IF v_current_status <> 'processing' THEN
        RAISE EXCEPTION 'ออเดอร์นี้ไม่ได้อยู่ในสถานะ processing (ปัจจุบัน: %)', v_current_status;
    END IF;

    -- วนลูปตัดสต๊อกสินค้าแต่ละรายการในออเดอร์
    FOR v_item IN
        SELECT product_id, quantity
          FROM order_items
         WHERE order_id = p_order_id
    LOOP
        UPDATE products
           SET stock_quantity = stock_quantity - v_item.quantity
         WHERE product_id = v_item.product_id;

        RAISE NOTICE 'ตัดสต๊อกสินค้า % จำนวน % ชิ้น', v_item.product_id, v_item.quantity;
    END LOOP;

    -- เปลี่ยนสถานะออเดอร์
    UPDATE orders SET status = 'shipped' WHERE order_id = p_order_id;

    -- procedure สามารถ COMMIT เองได้ (ต่างจาก function)
    COMMIT;

    RAISE NOTICE 'ออเดอร์ % ถูกจัดส่งเรียบร้อยแล้ว', p_order_id;
END;
$$;
```

เรียกใช้งานด้วย `CALL`:

```sql
CALL process_order_shipment(11);
```

```
NOTICE:  ตัดสต๊อกสินค้า 1 จำนวน 1 ชิ้น
NOTICE:  ออเดอร์ 11 ถูกจัดส่งเรียบร้อยแล้ว
CALL
```

ตรวจสอบผล:

```sql
SELECT order_id, status FROM orders WHERE order_id = 11;
SELECT product_id, stock_quantity FROM products WHERE product_id = 1;
```

```
 order_id | status
----------+---------
       11 | shipped
(1 row)

 product_id | stock_quantity
------------+-----------------
          1 |              44
(1 row)
```

> **หมายเหตุเรื่อง COMMIT ใน PROCEDURE:** การเรียก `COMMIT`/`ROLLBACK` ภายใน procedure ทำได้ก็ต่อเมื่อ procedure นั้นถูกเรียกจาก **top-level** (เช่น `CALL` ตรง ๆ จาก client หรือจาก `DO` block) เท่านั้น — ถ้า procedure ถูกเรียกซ้อนอยู่ภายใน transaction block ที่ผู้ใช้เปิดเองด้วย `BEGIN` แล้วยังไม่ commit จะไม่สามารถ `COMMIT` ซ้อนภายในได้ ต้องระวังเรื่องนี้เวลาออกแบบ batch job ที่มี procedure เรียกกันเป็นชั้น ๆ

### ตัวอย่าง PROCEDURE ที่ใช้ INOUT parameter เพื่อ "คืนค่า"

```sql
CREATE OR REPLACE PROCEDURE apply_bulk_discount(
    p_category_id  INTEGER,
    p_discount_pct NUMERIC,
    INOUT p_affected_rows INTEGER DEFAULT 0
)
LANGUAGE plpgsql
AS $$
BEGIN
    UPDATE products
       SET unit_price = ROUND(unit_price * (1 - p_discount_pct / 100.0), 2)
     WHERE category_id = p_category_id
       AND is_active = true;

    GET DIAGNOSTICS p_affected_rows = ROW_COUNT;
END;
$$;
```

```sql
CALL apply_bulk_discount(6, 15, 0);
```

```
 p_affected_rows
------------------
                3
(1 row)
```

`GET DIAGNOSTICS ... = ROW_COUNT` เป็นคำสั่งที่ดึงจำนวนแถวที่ได้รับผลกระทบจากคำสั่ง SQL ก่อนหน้า มีประโยชน์มากสำหรับ logging และการรายงานผล

### เมื่อไรควรใช้ FUNCTION vs PROCEDURE

- ใช้ **FUNCTION** เมื่อ: ต้องการคำนวณและคืนค่าเพื่อใช้ต่อใน query (`SELECT`), ต้องใช้เป็น trigger function, หรือ logic ไม่จำเป็นต้องควบคุม transaction เอง
- ใช้ **PROCEDURE** เมื่อ: งานเป็นลักษณะ "ทำสิ่งหนึ่งให้เสร็จ" (perform an action) เช่น batch job, data migration, ETL step ที่ต้องการ `COMMIT` เป็นช่วง ๆ เพื่อไม่ให้ transaction ค้างนานเกินไป หรืองานที่ไม่มีผลลัพธ์ให้คืนแบบ scalar

---

## Step 459: Security — SECURITY DEFINER vs SECURITY INVOKER

เมื่อฟังก์ชัน/procedure ถูกเรียกใช้งาน มันจะรันด้วย "สิทธิ์" ของใครสักคน — PostgreSQL มี 2 โหมด:

| โหมด | ความหมาย |
|---|---|
| `SECURITY INVOKER` (ค่า **default**) | ฟังก์ชันรันด้วยสิทธิ์ของ **ผู้ที่เรียกใช้ฟังก์ชัน** (caller) |
| `SECURITY DEFINER` | ฟังก์ชันรันด้วยสิทธิ์ของ **ผู้ที่สร้างฟังก์ชัน** (owner/definer) ไม่ว่าใครจะเป็นคนเรียก |

### ทำไม SECURITY DEFINER ถึงมีประโยชน์

สมมติว่ามี role ชื่อ `app_readonly` ที่มีสิทธิ์แค่ `SELECT` บนตาราง `orders` และ `order_items` แต่ **ไม่มีสิทธิ์แก้ไข** ตาราง `products` เลย เราต้องการให้ role นี้สามารถ "ยืนยันการจัดส่ง" ได้ (ซึ่งต้องไปลดค่า `stock_quantity` ในตาราง `products`) โดยไม่ต้องให้สิทธิ์ `UPDATE products` แบบเต็ม ๆ — วิธีแก้คือสร้างฟังก์ชันแบบ `SECURITY DEFINER` ที่เจ้าของฟังก์ชันมีสิทธิ์เต็ม แล้วให้ role นั้นมีสิทธิ์แค่ `EXECUTE` ฟังก์ชันเท่านั้น

```sql
CREATE OR REPLACE FUNCTION confirm_delivery(p_order_id INTEGER)
RETURNS TEXT
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = public   -- best practice: ล็อก search_path เพื่อป้องกัน SQL injection ผ่าน schema
AS $$
DECLARE
    v_current_status orders.status%TYPE;
BEGIN
    SELECT status INTO v_current_status
      FROM orders WHERE order_id = p_order_id;

    IF v_current_status IS NULL THEN
        RETURN 'ไม่พบออเดอร์นี้';
    END IF;

    IF v_current_status NOT IN ('shipped', 'processing') THEN
        RETURN format('ไม่สามารถยืนยันจัดส่งได้ (สถานะปัจจุบัน: %s)', v_current_status);
    END IF;

    UPDATE orders SET status = 'delivered' WHERE order_id = p_order_id;

    RETURN 'ยืนยันการจัดส่งสำเร็จ';
END;
$$;

-- อนุญาตให้ role ที่มีสิทธิ์จำกัดเรียกใช้ฟังก์ชันนี้ได้ (โดยไม่ต้องให้สิทธิ์ UPDATE orders ตรง ๆ)
-- GRANT EXECUTE ON FUNCTION confirm_delivery(INTEGER) TO app_readonly;
```

```sql
SELECT confirm_delivery(5);
```

```
       confirm_delivery
--------------------------------
 ยืนยันการจัดส่งสำเร็จ
(1 row)
```

### ข้อควรระวังด้านความปลอดภัยของ SECURITY DEFINER

`SECURITY DEFINER` เป็นเครื่องมือที่ทรงพลังแต่**อันตรายถ้าใช้ไม่ระวัง** เพราะฟังก์ชันจะรันด้วยสิทธิ์เต็มของเจ้าของ (มักจะเป็น superuser หรือ role ที่มีสิทธิ์สูง) ข้อควรระวังหลัก ๆ:

1. **ตั้ง `SET search_path` เสมอ** — ถ้าไม่ล็อก `search_path` ผู้ไม่หวังดีอาจสร้าง object (เช่น function หรือ table) ชื่อซ้ำใน schema ที่ควบคุมได้ เพื่อ "แอบสวมรอย" object ที่ฟังก์ชันเรียกใช้ (เทคนิคนี้เรียกว่า **search_path hijacking**) — เป็นช่องโหว่ด้านความปลอดภัยที่มีการรายงานจริงมาแล้วหลายครั้งในระบบที่ใช้ PostgreSQL
2. **จำกัด logic ให้แคบและชัดเจน** — อย่าให้ฟังก์ชัน `SECURITY DEFINER` รับ SQL หรือชื่อ table/column จาก parameter แล้วนำไป `EXECUTE` แบบ dynamic (เสี่ยง SQL injection ที่จะรันด้วยสิทธิ์สูงสุด)
3. **ให้สิทธิ์ `EXECUTE` เฉพาะเท่าที่จำเป็น** — ใช้ `REVOKE EXECUTE ... FROM PUBLIC` แล้ว `GRANT` เฉพาะ role ที่ควรใช้งานได้จริง
4. **ตรวจสอบสิทธิ์ภายในฟังก์ชันเองถ้าจำเป็น** — เช่น ตรวจสอบว่า `current_user` มีสิทธิ์ทำสิ่งนี้จริงหรือไม่ ก่อนจะดำเนินการ

### ตัวอย่างเปรียบเทียบ: SECURITY INVOKER (ค่า default)

```sql
CREATE OR REPLACE FUNCTION get_my_order_count()
RETURNS INTEGER
LANGUAGE plpgsql
SECURITY INVOKER   -- ค่า default อยู่แล้ว เขียนไว้เพื่อความชัดเจน
AS $$
BEGIN
    RETURN (SELECT count(*) FROM orders);
END;
$$;
```

ฟังก์ชันนี้จะรันด้วยสิทธิ์ของ **ผู้เรียก** — ถ้าผู้เรียกไม่มีสิทธิ์ `SELECT` บนตาราง `orders` เลย ฟังก์ชันนี้ก็จะ error เช่นกัน ต่างจาก `SECURITY DEFINER` ที่จะใช้สิทธิ์ของเจ้าของฟังก์ชันเสมอไม่ว่าใครเรียก

> **กฎง่าย ๆ ในการเลือก:** ใช้ `SECURITY INVOKER` (ค่า default) เป็นหลักเสมอ เปลี่ยนเป็น `SECURITY DEFINER` **เฉพาะเมื่อมีเหตุผลชัดเจน** ว่าต้องการให้ฟังก์ชันทำสิ่งที่ผู้เรียกไม่มีสิทธิ์ทำโดยตรง (privilege escalation ที่ควบคุมได้) และต้องเขียนด้วยความระมัดระวังสูงสุดเสมอ

---

## Step 460: แบบฝึกหัดรวม — เขียนฟังก์ชันคำนวณธุรกิจจริง

มาผสานทุกสิ่งที่เรียนมาในบทนี้เข้าด้วยกัน เพื่อแก้ปัญหาทางธุรกิจจริง 2 เรื่อง

### โจทย์ที่ 1: ฟังก์ชันคำนวณส่วนลดตามระดับสมาชิก (loyalty discount)

ร้านค้าต้องการให้ส่วนลดตามระดับลูกค้า (ที่คำนวณจาก Step 456) โดยมีกฎเพิ่มเติมคือ ลูกค้าที่เป็นสมาชิกมานานกว่า 1 ปี จะได้ส่วนลดเพิ่มอีก 2%:

```sql
CREATE OR REPLACE FUNCTION calculate_member_discount(p_customer_id INTEGER)
RETURNS NUMERIC
LANGUAGE plpgsql
STABLE
AS $$
DECLARE
    v_tier            TEXT;
    v_base_discount   NUMERIC(5,2);
    v_signup_date     customers.signup_date%TYPE;
    v_loyalty_bonus   NUMERIC(5,2) := 0;
    v_total_discount  NUMERIC(5,2);
BEGIN
    -- ใช้ฟังก์ชันที่สร้างไว้ก่อนหน้า (function composition)
    v_tier := get_customer_tier(p_customer_id);

    -- กำหนดส่วนลดพื้นฐานตาม tier
    IF v_tier = 'Platinum' THEN
        v_base_discount := 15.00;
    ELSIF v_tier = 'Gold' THEN
        v_base_discount := 10.00;
    ELSIF v_tier = 'Silver' THEN
        v_base_discount := 5.00;
    ELSE
        v_base_discount := 0.00;
    END IF;

    -- ตรวจสอบระยะเวลาสมาชิก
    SELECT signup_date INTO v_signup_date
      FROM customers
     WHERE customer_id = p_customer_id;

    IF v_signup_date IS NOT NULL AND v_signup_date <= (CURRENT_DATE - INTERVAL '1 year') THEN
        v_loyalty_bonus := 2.00;
    END IF;

    v_total_discount := v_base_discount + v_loyalty_bonus;

    -- จำกัดส่วนลดสูงสุดไม่เกิน 20%
    IF v_total_discount > 20.00 THEN
        v_total_discount := 20.00;
    END IF;

    RETURN v_total_discount;
END;
$$;
```

ทดสอบ (สมมติวันปัจจุบันคือ 2026-09-25 ตาม context ของหลักสูตร ลูกค้าทุกคนสมัครในปี 2023 จึงเข้าเกณฑ์ loyalty bonus ทั้งหมด):

```sql
SELECT customer_id, first_name,
       get_customer_tier(customer_id)         AS tier,
       calculate_member_discount(customer_id) AS discount_pct
  FROM customers
 ORDER BY customer_id
 LIMIT 6;
```

```
 customer_id | first_name |    tier     | discount_pct
-------------+------------+-------------+---------------
           1 | Somchai    | Platinum    |         17.00
           2 | Emily      | Gold        |         12.00
           3 | Yuki       | Bronze      |          2.00
           4 | Nattaya    | Platinum    |         17.00
           5 | James      | Bronze      |          2.00
           6 | Chalermchai| Silver      |          7.00
(6 rows)
```

นำไปใช้คำนวณราคาสุทธิของสินค้าจริง โดยผสานกับฟังก์ชัน `get_discounted_price` จาก Step 453:

```sql
SELECT
    c.customer_id,
    c.first_name,
    p.product_name,
    p.unit_price,
    calculate_member_discount(c.customer_id) AS discount_pct,
    get_discounted_price(p.product_id, calculate_member_discount(c.customer_id)) AS final_price
FROM customers c
CROSS JOIN products p
WHERE c.customer_id = 1
  AND p.product_id IN (1, 6, 11);
```

```
 customer_id | first_name |    product_name     | unit_price | discount_pct | final_price
-------------+------------+-----------------------+------------+---------------+--------------
           1 | Somchai    | UltraBook Pro 15      |   32900.00 |         17.00 |    27307.00
           1 | Somchai    | The Silent Ocean      |     350.00 |         17.00 |      290.50
           1 | Somchai    | Yoga Mat Premium      |     850.00 |         17.00 |      705.50
(3 rows)
```

### โจทย์ที่ 2: ฟังก์ชันคืนรายการสินค้าขายดี (best-selling products)

ธุรกิจต้องการรายงานสินค้าขายดี โดยรับพารามิเตอร์ว่าต้องการกี่อันดับ (top N) และกรองเฉพาะออเดอร์ที่ไม่ถูกยกเลิก:

```sql
CREATE OR REPLACE FUNCTION get_best_selling_products(p_limit INTEGER DEFAULT 5)
RETURNS TABLE (
    rank              BIGINT,
    product_id        INTEGER,
    product_name      VARCHAR(150),
    category_name     VARCHAR(100),
    total_qty_sold    BIGINT,
    total_revenue     NUMERIC(14,2)
)
LANGUAGE plpgsql
STABLE
AS $$
BEGIN
    -- ตรวจสอบ input เบื้องต้นเพื่อความปลอดภัย
    IF p_limit IS NULL OR p_limit <= 0 THEN
        p_limit := 5;
    END IF;

    RETURN QUERY
    SELECT
        ROW_NUMBER() OVER (ORDER BY SUM(oi.quantity * oi.unit_price) DESC) AS rank,
        p.product_id,
        p.product_name,
        cat.category_name,
        SUM(oi.quantity)::BIGINT                 AS total_qty_sold,
        SUM(oi.quantity * oi.unit_price)::NUMERIC(14,2) AS total_revenue
    FROM order_items oi
    JOIN orders o      ON o.order_id = oi.order_id
    JOIN products p    ON p.product_id = oi.product_id
    LEFT JOIN categories cat ON cat.category_id = p.category_id
    WHERE o.status <> 'cancelled'
    GROUP BY p.product_id, p.product_name, cat.category_name
    ORDER BY total_revenue DESC
    LIMIT p_limit;
END;
$$;
```

ทดสอบ:

```sql
SELECT * FROM get_best_selling_products(5);
```

```
 rank | product_id |    product_name     | category_name |  total_qty_sold | total_revenue
------+------------+-----------------------+----------------+------------------+----------------
    1 |          1 | UltraBook Pro 15      | Computers      |                3 |       98700.00
    2 |          2 | Galaxy Phone X200      | Smartphones    |                2 |       49800.00
    3 |          5 | 4K Monitor 27"         | Computers      |                2 |       17800.00
    4 |          3 | Wireless Earbuds Z1    | Audio          |                6 |       15540.00
    5 |         20 | Portable SSD 1TB       | Computers      |                1 |        3990.00
(5 rows)
```

เรียกด้วยพารามิเตอร์ต่างกัน:

```sql
SELECT rank, product_name, total_qty_sold, total_revenue
  FROM get_best_selling_products(3);
```

```
 rank |    product_name    | total_qty_sold | total_revenue
------+----------------------+------------------+----------------
    1 | UltraBook Pro 15     |                3 |       98700.00
    2 | Galaxy Phone X200    |                2 |       49800.00
    3 | 4K Monitor 27"       |                2 |       17800.00
(3 rows)
```

ทั้งสองฟังก์ชันนี้แสดงให้เห็นภาพรวมของ PL/pgSQL ที่ประยุกต์ใช้ได้จริง — `calculate_member_discount` แสดงการต่อยอด (compose) ฟังก์ชันหลายตัวเข้าด้วยกัน ส่วน `get_best_selling_products` แสดงการรวม window function, aggregate, JOIN หลายตาราง และ parameter validation เข้าไว้ในฟังก์ชันเดียวที่พร้อมใช้งานจริงในระบบรายงาน

---

## สรุปท้ายบท

ในบทนี้เราได้เรียนรู้พื้นฐานของ PL/pgSQL ซึ่งเป็นรากฐานสำคัญสำหรับการเขียนโปรแกรมเชิงกระบวนการภายในฐานข้อมูล PostgreSQL:

- **PL/pgSQL** คือภาษา procedural ที่ผสาน SQL กับตรรกะแบบมีขั้นตอน (variable, condition, loop) ทำให้เขียน business logic ที่ซับซ้อนได้ภายในฐานข้อมูลโดยตรง ลด round-trip และรวม logic ให้เป็นมาตรฐานเดียวกัน
- **`CREATE FUNCTION`** มีโครงสร้างหลัก: ชื่อ, พารามิเตอร์, `RETURNS`, `LANGUAGE plpgsql`, และโค้ดใน `$$ ... $$`
- **`DECLARE`** ใช้ประกาศตัวแปร และ **`%TYPE`** ช่วยผูก type ของตัวแปรกับคอลัมน์จริง ป้องกันปัญหา type mismatch เมื่อ schema เปลี่ยน
- **`RETURN`** คืนค่าเดียว (scalar) ส่วน **`RETURNS TABLE`** และ **`RETURNS SETOF`** ใช้คืนผลลัพธ์หลายแถว โดย `RETURNS TABLE` ยืดหยุ่นกว่าเพราะกำหนดคอลัมน์เองได้
- **`IF/ELSIF/ELSE`** ใช้เขียนเงื่อนไขหลายชั้น เป็นหัวใจของ business rule ที่ซับซ้อน
- **Volatility** (`IMMUTABLE`/`STABLE`/`VOLATILE`) บอก query planner ว่าฟังก์ชัน cache ผลลัพธ์ได้แค่ไหน — ประกาศผิดอาจทำให้เกิดบั๊กที่ตรวจจับยาก และ `IMMUTABLE` เท่านั้นที่ใช้สร้าง functional index ได้
- **`CREATE PROCEDURE`** ต่างจาก `FUNCTION` ตรงที่เรียกด้วย `CALL`, ไม่มีค่า return แบบ scalar, แต่ควบคุม transaction (`COMMIT`/`ROLLBACK`) ได้เอง เหมาะกับงาน batch/ETL
- **`SECURITY DEFINER`** ให้ฟังก์ชันรันด้วยสิทธิ์ของเจ้าของแทนผู้เรียก มีประโยชน์สำหรับควบคุมสิทธิ์แบบละเอียด แต่ต้องระวังเรื่อง `search_path hijacking` และควรล็อก `SET search_path` เสมอ

บทถัดไป **Part 047: PL/pgSQL ขั้นสูง** จะพาไปลึกขึ้นในเรื่อง loop หลายรูปแบบ (`FOR`, `WHILE`, `LOOP`), exception handling (`EXCEPTION WHEN`), cursor, dynamic SQL (`EXECUTE`), และการเขียนฟังก์ชันที่ซับซ้อนระดับ production

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1
เขียนฟังก์ชัน `get_supplier_country(p_supplier_id INTEGER)` ที่คืนค่าประเทศของ supplier นั้น ถ้าไม่พบให้คืนค่า `'Unknown'`

<details>
<summary>เฉลย</summary>

```sql
CREATE OR REPLACE FUNCTION get_supplier_country(p_supplier_id INTEGER)
RETURNS TEXT
LANGUAGE plpgsql
STABLE
AS $$
DECLARE
    v_country suppliers.country%TYPE;
BEGIN
    SELECT country INTO v_country
      FROM suppliers
     WHERE supplier_id = p_supplier_id;

    IF NOT FOUND THEN
        RETURN 'Unknown';
    END IF;

    RETURN v_country;
END;
$$;

-- ทดสอบ
SELECT get_supplier_country(1);   -- USA
SELECT get_supplier_country(999); -- Unknown
```

</details>

---

### แบบฝึกหัดที่ 2
เขียนฟังก์ชัน `calculate_order_total(p_order_id INTEGER)` ที่คืนยอดรวมทั้งหมดของออเดอร์นั้น (ผลรวม `quantity * unit_price` ของทุก order_items ในออเดอร์) ใช้ `%TYPE` ในการประกาศตัวแปร

<details>
<summary>เฉลย</summary>

```sql
CREATE OR REPLACE FUNCTION calculate_order_total(p_order_id INTEGER)
RETURNS NUMERIC
LANGUAGE plpgsql
STABLE
AS $$
DECLARE
    v_total NUMERIC(12,2);
BEGIN
    SELECT COALESCE(SUM(oi.quantity * oi.unit_price), 0)
      INTO v_total
      FROM order_items oi
     WHERE oi.order_id = p_order_id;

    RETURN v_total;
END;
$$;

-- ทดสอบ
SELECT calculate_order_total(1);  -- 36100.00
```

</details>

---

### แบบฝึกหัดที่ 3
เขียนฟังก์ชัน `is_order_eligible_for_return(p_order_id INTEGER)` คืนค่า `BOOLEAN` โดยออเดอร์จะคืนสินค้าได้ถ้าสถานะเป็น `'delivered'` เท่านั้น (ใช้ `IF` ตัดสินใจ)

<details>
<summary>เฉลย</summary>

```sql
CREATE OR REPLACE FUNCTION is_order_eligible_for_return(p_order_id INTEGER)
RETURNS BOOLEAN
LANGUAGE plpgsql
STABLE
AS $$
DECLARE
    v_status orders.status%TYPE;
BEGIN
    SELECT status INTO v_status
      FROM orders
     WHERE order_id = p_order_id;

    IF v_status = 'delivered' THEN
        RETURN true;
    ELSE
        RETURN false;
    END IF;
END;
$$;

-- ทดสอบ
SELECT is_order_eligible_for_return(1);  -- t
SELECT is_order_eligible_for_return(18); -- f (pending)
```

</details>

---

### แบบฝึกหัดที่ 4
เขียนฟังก์ชัน `get_customers_by_country(p_country TEXT)` แบบ `RETURNS TABLE` ที่คืน `customer_id`, `first_name`, `last_name`, `email` ของลูกค้าในประเทศที่ระบุ เรียงตาม `signup_date`

<details>
<summary>เฉลย</summary>

```sql
CREATE OR REPLACE FUNCTION get_customers_by_country(p_country TEXT)
RETURNS TABLE (
    customer_id INTEGER,
    first_name  VARCHAR(60),
    last_name   VARCHAR(60),
    email       VARCHAR(150)
)
LANGUAGE plpgsql
STABLE
AS $$
BEGIN
    RETURN QUERY
    SELECT c.customer_id, c.first_name, c.last_name, c.email
      FROM customers c
     WHERE c.country = p_country
     ORDER BY c.signup_date;
END;
$$;

-- ทดสอบ
SELECT * FROM get_customers_by_country('Thailand');
```

</details>

---

### แบบฝึกหัดที่ 5
เขียนฟังก์ชัน `get_active_products_in_stock()` แบบ `RETURNS SETOF products` ที่คืนสินค้าที่ `is_active = true` และ `stock_quantity > 0` เท่านั้น

<details>
<summary>เฉลย</summary>

```sql
CREATE OR REPLACE FUNCTION get_active_products_in_stock()
RETURNS SETOF products
LANGUAGE plpgsql
STABLE
AS $$
BEGIN
    RETURN QUERY
    SELECT *
      FROM products
     WHERE is_active = true
       AND stock_quantity > 0
     ORDER BY product_id;
END;
$$;

-- ทดสอบ
SELECT product_id, product_name, stock_quantity
  FROM get_active_products_in_stock()
 LIMIT 5;
```

</details>

---

### แบบฝึกหัดที่ 6
เขียนฟังก์ชัน `classify_order_size(p_order_id INTEGER)` คืนค่า `TEXT` โดยใช้ `IF/ELSIF/ELSE` จัดกลุ่มออเดอร์ตามยอดรวม (ใช้ฟังก์ชัน `calculate_order_total` จากข้อ 2 ประกอบ): มากกว่า 20000 = `'Large'`, 5000–20000 = `'Medium'`, น้อยกว่า 5000 = `'Small'`

<details>
<summary>เฉลย</summary>

```sql
CREATE OR REPLACE FUNCTION classify_order_size(p_order_id INTEGER)
RETURNS TEXT
LANGUAGE plpgsql
STABLE
AS $$
DECLARE
    v_total NUMERIC(12,2);
BEGIN
    v_total := calculate_order_total(p_order_id);

    IF v_total > 20000 THEN
        RETURN 'Large';
    ELSIF v_total >= 5000 THEN
        RETURN 'Medium';
    ELSE
        RETURN 'Small';
    END IF;
END;
$$;

-- ทดสอบ
SELECT order_id, classify_order_size(order_id) AS size_class
  FROM orders
 ORDER BY order_id
 LIMIT 5;
```

</details>

---

### แบบฝึกหัดที่ 7
ฟังก์ชันต่อไปนี้ประกาศ volatility ผิด จงระบุว่าผิดตรงไหนและแก้ไขให้ถูกต้อง พร้อมอธิบายเหตุผล

```sql
CREATE OR REPLACE FUNCTION get_current_order_count()
RETURNS INTEGER
LANGUAGE plpgsql
IMMUTABLE
AS $$
BEGIN
    RETURN (SELECT count(*) FROM orders);
END;
$$;
```

<details>
<summary>เฉลย</summary>

**ปัญหา:** ฟังก์ชันนี้ query ตาราง `orders` ซึ่งข้อมูลเปลี่ยนแปลงได้ตลอดเวลา (มีการ insert/delete ออเดอร์ใหม่) การประกาศเป็น `IMMUTABLE` ผิดหลักการ เพราะ `IMMUTABLE` หมายถึง "ผลลัพธ์ต้องเหมือนเดิมทุกครั้งตลอดไปสำหรับ input เดียวกัน" — ซึ่งฟังก์ชันนี้ไม่มี input ด้วยซ้ำ แต่ผลลัพธ์เปลี่ยนได้ทุกครั้งที่มีการเพิ่ม/ลบออเดอร์ ถ้า planner นำไปสร้าง index หรือ cache ผลลัพธ์แบบถาวรจะทำให้ได้ค่าที่ผิดพลาด

**แก้ไข:** ควรเป็น `STABLE` เพราะฟังก์ชันอ่านค่าจากตาราง โดยผลลัพธ์จะเหมือนกันภายใน query/transaction เดียวกัน แต่เปลี่ยนแปลงได้ข้าม transaction:

```sql
CREATE OR REPLACE FUNCTION get_current_order_count()
RETURNS INTEGER
LANGUAGE plpgsql
STABLE
AS $$
BEGIN
    RETURN (SELECT count(*) FROM orders);
END;
$$;
```

</details>

---

### แบบฝึกหัดที่ 8
เขียน `PROCEDURE` ชื่อ `cancel_order(p_order_id INTEGER)` ที่เปลี่ยนสถานะออเดอร์เป็น `'cancelled'` แต่ถ้าสถานะปัจจุบันเป็น `'delivered'` อยู่แล้วให้ `RAISE EXCEPTION` แจ้งว่ายกเลิกไม่ได้

<details>
<summary>เฉลย</summary>

```sql
CREATE OR REPLACE PROCEDURE cancel_order(p_order_id INTEGER)
LANGUAGE plpgsql
AS $$
DECLARE
    v_status orders.status%TYPE;
BEGIN
    SELECT status INTO v_status
      FROM orders
     WHERE order_id = p_order_id;

    IF v_status IS NULL THEN
        RAISE EXCEPTION 'ไม่พบออเดอร์หมายเลข %', p_order_id;
    END IF;

    IF v_status = 'delivered' THEN
        RAISE EXCEPTION 'ออเดอร์ % ถูกจัดส่งแล้ว ไม่สามารถยกเลิกได้', p_order_id;
    END IF;

    UPDATE orders SET status = 'cancelled' WHERE order_id = p_order_id;
    COMMIT;

    RAISE NOTICE 'ยกเลิกออเดอร์ % สำเร็จ', p_order_id;
END;
$$;

-- ทดสอบ
CALL cancel_order(18);          -- สำเร็จ (สถานะ pending)
-- CALL cancel_order(1);        -- error: ออเดอร์ถูกจัดส่งแล้ว
```

</details>

---

### แบบฝึกหัดที่ 9
อธิบายว่าเพราะเหตุใด ฟังก์ชันที่มีคำสั่ง `INSERT INTO audit_log ...` อยู่ภายใน ไม่ควรถูกประกาศเป็น `STABLE` หรือ `IMMUTABLE` แม้ว่าฟังก์ชันนั้นจะคืนค่าตัวเลขเดิมเสมอสำหรับ input เดียวกันก็ตาม

<details>
<summary>เฉลย</summary>

แม้ว่าค่าที่ฟังก์ชัน **คืนกลับ** (return value) จะเหมือนเดิมทุกครั้ง แต่ `IMMUTABLE`/`STABLE` ไม่ได้พิจารณาแค่ค่าที่คืน — มันหมายถึง **ฟังก์ชันต้องไม่มี side effect ที่ผู้ใช้สังเกตเห็นได้ (no observable side effects)** ด้วย เพราะ planner อาจเลือก "ไม่เรียกฟังก์ชันซ้ำ" หรือ "เรียกน้อยกว่าที่ query เขียนไว้จริง" ถ้าคิดว่าฟังก์ชันนั้น stable/immutable

ถ้าฟังก์ชันมี `INSERT INTO audit_log` อยู่ข้างใน และถูกประกาศเป็น `STABLE`/`IMMUTABLE` ผิด ๆ planner อาจ optimize โดยเรียกฟังก์ชันแค่ครั้งเดียวแทนที่จะเรียกทุกแถวตามที่ query ต้องการจริง ทำให้ **audit log ขาดหายไปบางรายการ** ซึ่งเป็นบั๊กที่ตรวจจับยากมากเพราะผลลัพธ์หลักของ query (ค่าที่คืน) ยังคงถูกต้อง มีแต่ side effect เท่านั้นที่หายไป

**สรุป:** ฟังก์ชันที่มี `INSERT`/`UPDATE`/`DELETE` หรือ side effect ใด ๆ ต้องเป็น `VOLATILE` เสมอ ไม่ว่าค่าที่คืนจะดูเหมือนคงที่แค่ไหนก็ตาม

</details>

---

### แบบฝึกหัดที่ 10
เขียนฟังก์ชัน `get_category_revenue_report(p_min_revenue NUMERIC DEFAULT 0)` แบบ `RETURNS TABLE` ที่รายงานยอดขายรวม (`SUM(quantity * unit_price)`) แยกตามหมวดหมู่สินค้า (`category_name`) โดยกรองเฉพาะหมวดที่มียอดขายรวมมากกว่าหรือเท่ากับ `p_min_revenue` เรียงจากมากไปน้อย นับเฉพาะออเดอร์ที่ไม่ถูกยกเลิก

<details>
<summary>เฉลย</summary>

```sql
CREATE OR REPLACE FUNCTION get_category_revenue_report(p_min_revenue NUMERIC DEFAULT 0)
RETURNS TABLE (
    category_name  VARCHAR(100),
    total_revenue  NUMERIC(14,2),
    total_orders   BIGINT
)
LANGUAGE plpgsql
STABLE
AS $$
BEGIN
    IF p_min_revenue IS NULL THEN
        p_min_revenue := 0;
    END IF;

    RETURN QUERY
    SELECT
        cat.category_name,
        SUM(oi.quantity * oi.unit_price)::NUMERIC(14,2) AS total_revenue,
        COUNT(DISTINCT o.order_id)::BIGINT               AS total_orders
    FROM order_items oi
    JOIN orders o       ON o.order_id = oi.order_id
    JOIN products p     ON p.product_id = oi.product_id
    JOIN categories cat ON cat.category_id = p.category_id
    WHERE o.status <> 'cancelled'
    GROUP BY cat.category_name
    HAVING SUM(oi.quantity * oi.unit_price) >= p_min_revenue
    ORDER BY total_revenue DESC;
END;
$$;

-- ทดสอบ
SELECT * FROM get_category_revenue_report(5000);
```

ผลลัพธ์ตัวอย่าง:

```
 category_name |  total_revenue |  total_orders
----------------+-----------------+-----------------
 Computers      |       143870.00 |               6
 Smartphones    |        50290.00 |               2
 Audio          |        20510.00 |               4
 Fiction        |         6070.00 |               3
(4 rows)
```

</details>

---

## บทถัดไป

เนื้อหา PL/pgSQL ขั้นสูงกว่านี้ — loop หลากหลายรูปแบบ, exception handling, cursor, และ dynamic SQL — อยู่ในบทถัดไป: [Part 047: PL/pgSQL ขั้นสูง](./part-047-plpgsql-advanced.md)
