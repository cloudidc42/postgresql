# Custom Data Types, Domains, ENUM

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับสูง | Part 049

## เป้าหมายการเรียนรู้

เมื่อจบบทนี้ ผู้เรียนจะสามารถ:

- เข้าใจว่าทำไม built-in type ของ PostgreSQL (VARCHAR, INTEGER, NUMERIC ฯลฯ) บางครั้งไม่เพียงพอต่อการบังคับ business rule
- สร้างและใช้งาน **ENUM type** เพื่อจำกัดค่าที่เป็นไปได้ของคอลัมน์ให้ชัดเจนและปลอดภัยกว่า VARCHAR
- วิเคราะห์ข้อดี-ข้อเสียของ ENUM เทียบกับ VARCHAR+CHECK และตาราง lookup แยก เพื่อเลือกใช้ให้เหมาะกับสถานการณ์
- เพิ่มค่าใหม่ใน ENUM ภายหลังด้วย `ALTER TYPE ... ADD VALUE` พร้อมเข้าใจข้อจำกัดเรื่อง transaction และการลบค่า
- สร้าง **Composite Type** เพื่อรวมหลายฟิลด์เป็นชนิดข้อมูลเดียว และนำไปใช้เป็น column type, parameter, และ return type ของฟังก์ชัน
- เข้าใจแนวคิดของ **Domain** ในฐานะ type ใหม่ที่สร้างจาก type เดิมพร้อม constraint ในตัว
- สร้าง Domain ที่ใช้งานจริงได้ เช่น `email_address`, `positive_numeric` พร้อม CHECK/regex
- เปรียบเทียบ Domain กับ CHECK constraint ตรงๆ บนคอลัมน์ และตัดสินใจได้ว่าเมื่อไหร่ควรใช้แบบไหน
- ปรับปรุง schema ของระบบ e-commerce ให้ปลอดภัยและอ่านง่ายขึ้นด้วยการผสมผสาน ENUM, Domain และ Composite Type เข้าด้วยกัน

---

## เตรียมข้อมูล

บทนี้ยังคงใช้ฐานข้อมูล e-commerce ชุดเดิมที่ใช้ตลอดทั้งหลักสูตร ประกอบด้วย 6 ตาราง: `categories`, `suppliers`, `products`, `customers`, `orders`, `order_items` หากยังไม่มีตารางเหล่านี้ในฐานข้อมูลทดลอง ให้รันสคริปต์ด้านล่างนี้ก่อน

```sql
-- ล้างของเก่า (ถ้ามี) เพื่อเริ่มต้นสะอาด
DROP TABLE IF EXISTS order_items CASCADE;
DROP TABLE IF EXISTS orders CASCADE;
DROP TABLE IF EXISTS products CASCADE;
DROP TABLE IF EXISTS customers CASCADE;
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
    customer_id  SERIAL PRIMARY KEY,
    first_name   VARCHAR(60) NOT NULL,
    last_name    VARCHAR(60) NOT NULL,
    email        VARCHAR(150) UNIQUE,
    country      VARCHAR(60),
    signup_date  DATE NOT NULL DEFAULT CURRENT_DATE
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

ต่อไปคือข้อมูลตัวอย่าง (seed data) ที่จำลองร้านค้าออนไลน์จริง ครอบคลุมหมวดหมู่สินค้า ผู้จัดจำหน่าย สินค้า ลูกค้า คำสั่งซื้อ และรายการสินค้าในคำสั่งซื้อ

```sql
-- categories: มีทั้ง top-level และ sub-category (self-reference)
INSERT INTO categories (category_name, parent_category_id) VALUES
    ('Electronics', NULL),          -- 1
    ('Computers', 1),               -- 2
    ('Smartphones', 1),             -- 3
    ('Home Appliances', NULL),      -- 4
    ('Furniture', NULL),            -- 5
    ('Books', NULL),                -- 6
    ('Clothing', NULL),             -- 7
    ('Sports & Outdoors', NULL),    -- 8
    ('Toys', NULL),                 -- 9
    ('Beauty & Health', NULL),      -- 10
    ('Groceries', NULL),            -- 11
    ('Automotive', NULL);           -- 12

-- suppliers
INSERT INTO suppliers (supplier_name, country) VALUES
    ('Bangkok Tech Distribution', 'Thailand'),      -- 1
    ('Shenzhen Electronics Co.', 'China'),          -- 2
    ('Nordic Home Living', 'Sweden'),               -- 3
    ('Osaka Precision Ltd.', 'Japan'),               -- 4
    ('Seoul Digital Corp.', 'South Korea'),         -- 5
    ('Berlin Werkzeug GmbH', 'Germany'),            -- 6
    ('Hanoi Textile Group', 'Vietnam'),             -- 7
    ('Taipei Components Inc.', 'Taiwan'),           -- 8
    ('Singapore Trading House', 'Singapore'),       -- 9
    ('Mumbai Organic Foods', 'India'),              -- 10
    ('California Fitness Supply', 'USA'),           -- 11
    ('Chiang Mai Craft Furniture', 'Thailand');     -- 12

-- products
INSERT INTO products (product_name, category_id, supplier_id, unit_price, stock_quantity, is_active) VALUES
    ('Wireless Mechanical Keyboard', 2, 8, 1890.00, 120, true),
    ('27-inch 4K Monitor', 2, 2, 8990.00, 45, true),
    ('Smartphone Z12 Pro', 3, 5, 24900.00, 60, true),
    ('Smartphone Z12 Lite', 3, 5, 12900.00, 80, true),
    ('Bluetooth Earbuds Pro', 1, 2, 1590.00, 200, true),
    ('Robot Vacuum Cleaner X200', 4, 4, 6990.00, 30, true),
    ('Air Purifier Compact', 4, 4, 3490.00, 55, true),
    ('Ergonomic Office Chair', 5, 12, 4590.00, 25, true),
    ('Oak Wood Dining Table', 5, 12, 12990.00, 10, true),
    ('The Art of PostgreSQL (Book)', 6, 9, 990.00, 150, true),
    ('SQL Performance Explained', 6, 9, 850.00, 90, false),
    ('Men''s Running Jacket', 7, 7, 1290.00, 110, true),
    ('Women''s Yoga Pants', 7, 7, 690.00, 140, true),
    ('Yoga Mat Premium', 8, 11, 590.00, 200, true),
    ('Adjustable Dumbbell Set', 8, 11, 3990.00, 40, true),
    ('Building Blocks Set 500pcs', 9, 9, 790.00, 75, true),
    ('Remote Control Drone Mini', 9, 2, 2190.00, 35, true),
    ('Organic Green Tea 100g', 11, 10, 220.00, 300, true),
    ('Vitamin C Serum 30ml', 10, 9, 450.00, 160, true),
    ('Car Dash Camera HD', 12, 6, 2590.00, 50, true);

-- customers
INSERT INTO customers (first_name, last_name, email, country, signup_date) VALUES
    ('Somchai', 'Jaidee', 'somchai.j@example.com', 'Thailand', '2023-01-15'),
    ('Pranee', 'Suksawat', 'pranee.s@example.com', 'Thailand', '2023-02-20'),
    ('Kittipong', 'Wongchai', 'kittipong.w@example.com', 'Thailand', '2023-03-05'),
    ('Nutcha', 'Rattanakorn', 'nutcha.r@example.com', 'Thailand', '2023-03-18'),
    ('John', 'Smith', 'john.smith@example.com', 'USA', '2023-04-02'),
    ('Emily', 'Johnson', 'emily.j@example.com', 'USA', '2023-04-22'),
    ('Yuki', 'Tanaka', 'yuki.tanaka@example.com', 'Japan', '2023-05-10'),
    ('Hiroshi', 'Sato', 'hiroshi.sato@example.com', 'Japan', '2023-05-30'),
    ('Min-jun', 'Kim', 'minjun.kim@example.com', 'South Korea', '2023-06-14'),
    ('Seo-yeon', 'Park', 'seoyeon.park@example.com', 'South Korea', '2023-07-01'),
    ('Anna', 'Mueller', 'anna.mueller@example.com', 'Germany', '2023-07-19'),
    ('Lars', 'Andersson', 'lars.a@example.com', 'Sweden', '2023-08-08'),
    ('Priya', 'Sharma', 'priya.sharma@example.com', 'India', '2023-08-25'),
    ('Wei', 'Chen', 'wei.chen@example.com', 'China', '2023-09-12'),
    ('Le', 'Thi Hoa', 'lethihoa@example.com', 'Vietnam', '2023-09-30'),
    ('Siriwan', 'Boonmee', 'siriwan.b@example.com', 'Thailand', '2023-10-11'),
    ('Apisit', 'Chaiyaporn', 'apisit.c@example.com', 'Thailand', '2023-11-02'),
    ('Michael', 'Brown', 'michael.brown@example.com', 'USA', '2023-11-20');

-- orders (มีหลายสถานะ เพื่อใช้สาธิตเรื่อง ENUM ในบทนี้)
INSERT INTO orders (customer_id, order_date, status, ship_country) VALUES
    (1, '2024-01-05 10:15:00+07', 'delivered', 'Thailand'),
    (2, '2024-01-08 14:30:00+07', 'delivered', 'Thailand'),
    (3, '2024-01-12 09:00:00+07', 'shipped', 'Thailand'),
    (4, '2024-01-15 16:45:00+07', 'processing', 'Thailand'),
    (5, '2024-01-18 11:20:00-05', 'delivered', 'USA'),
    (6, '2024-01-20 08:10:00-05', 'cancelled', 'USA'),
    (7, '2024-01-22 19:00:00+09', 'delivered', 'Japan'),
    (8, '2024-01-25 13:25:00+09', 'shipped', 'Japan'),
    (9, '2024-01-28 10:40:00+09', 'pending', 'South Korea'),
    (10, '2024-02-01 17:15:00+09', 'processing', 'South Korea'),
    (11, '2024-02-03 12:00:00+01', 'delivered', 'Germany'),
    (12, '2024-02-05 15:30:00+01', 'refunded', 'Sweden'),
    (1, '2024-02-08 09:45:00+07', 'delivered', 'Thailand'),
    (13, '2024-02-10 20:10:00+05:30', 'pending', 'India'),
    (14, '2024-02-12 11:55:00+08', 'shipped', 'China'),
    (3, '2024-02-15 14:00:00+07', 'delivered', 'Thailand'),
    (16, '2024-02-18 10:20:00+07', 'processing', 'Thailand'),
    (17, '2024-02-20 16:00:00+07', 'cancelled', 'Thailand');

-- order_items
INSERT INTO order_items (order_id, product_id, quantity, unit_price) VALUES
    (1, 1, 1, 1890.00),
    (1, 5, 2, 1590.00),
    (2, 3, 1, 24900.00),
    (3, 10, 3, 990.00),
    (3, 14, 1, 590.00),
    (4, 6, 1, 6990.00),
    (5, 2, 1, 8990.00),
    (5, 5, 1, 1590.00),
    (6, 13, 2, 690.00),
    (7, 4, 1, 12900.00),
    (7, 19, 2, 450.00),
    (8, 8, 1, 4590.00),
    (9, 16, 3, 790.00),
    (10, 20, 1, 2590.00),
    (11, 9, 1, 12990.00),
    (12, 15, 1, 3990.00),
    (13, 1, 2, 1890.00),
    (13, 18, 5, 220.00),
    (14, 12, 1, 1290.00),
    (14, 13, 1, 690.00),
    (15, 7, 1, 3490.00),
    (16, 3, 1, 24900.00),
    (16, 5, 1, 1590.00),
    (17, 17, 1, 2190.00),
    (18, 11, 1, 850.00);
```

ตรวจสอบว่าโหลดข้อมูลครบถ้วน:

```sql
SELECT
    (SELECT COUNT(*) FROM categories)  AS categories_count,
    (SELECT COUNT(*) FROM suppliers)   AS suppliers_count,
    (SELECT COUNT(*) FROM products)    AS products_count,
    (SELECT COUNT(*) FROM customers)   AS customers_count,
    (SELECT COUNT(*) FROM orders)      AS orders_count,
    (SELECT COUNT(*) FROM order_items) AS order_items_count;
```

```
 categories_count | suppliers_count | products_count | customers_count | orders_count | order_items_count
-------------------+------------------+-----------------+------------------+---------------+--------------------
                12 |               12 |              20 |               18 |            18 |                 25
```

---

## Step 481: ทำไมต้องมี custom type — เมื่อ built-in type ไม่พอสำหรับ business rule

PostgreSQL มี built-in type ให้ใช้เยอะมาก ทั้ง `INTEGER`, `NUMERIC`, `VARCHAR`, `TIMESTAMPTZ`, `BOOLEAN` ฯลฯ แต่ในโลกความเป็นจริง ข้อมูลทางธุรกิจมักมี "กฎ" ที่ type พื้นฐานเหล่านี้ไม่สามารถบังคับได้ด้วยตัวเอง

ลองดูตัวอย่างคอลัมน์ `orders.status` ในตารางที่เราเพิ่งสร้าง มันเป็น `VARCHAR(20)` ธรรมดา ซึ่งหมายความว่าเราสามารถใส่ค่าอะไรก็ได้ที่ไม่เกิน 20 ตัวอักษร:

```sql
-- สิ่งนี้ "ผ่าน" แม้ว่าจะเป็นข้อมูลที่ไม่ถูกต้องตาม business rule
INSERT INTO orders (customer_id, status, ship_country)
VALUES (1, 'compelted', 'Thailand');   -- พิมพ์ผิด! ควรเป็น 'completed'

INSERT INTO orders (customer_id, status, ship_country)
VALUES (2, 'PENDING', 'Thailand');     -- ตัวพิมพ์ใหญ่ ไม่ตรงกับที่อื่นที่ใช้ 'pending'

SELECT DISTINCT status FROM orders ORDER BY status;
```

```
   status
-------------
 PENDING
 cancelled
 compelted
 delivered
 pending
 processing
 refunded
 shipped
(8 rows)
```

จะเห็นว่าตอนนี้เรามีทั้ง `'pending'`, `'PENDING'` และ `'compelted'` (พิมพ์ผิดจาก `'completed'`) ปะปนกันอยู่ในคอลัมน์เดียวกัน ซึ่งเป็นปัญหาที่พบบ่อยมากเมื่อใช้ `VARCHAR` แบบไม่มีการควบคุม ผลกระทบที่ตามมาคือ:

1. **Query ที่กรองด้วย `WHERE status = 'pending'` จะพลาดแถวที่เป็น `'PENDING'`** เพราะ string comparison เป็น case-sensitive
2. **รายงานสรุปยอดตามสถานะจะแตกกลุ่มผิด** เพราะ `GROUP BY status` มองว่า `'pending'` กับ `'PENDING'` เป็นคนละกลุ่ม
3. **ไม่มีที่ไหนบอกเราล่วงหน้าว่าค่าที่ถูกต้องมีอะไรบ้าง** ต้องไปเดาจากข้อมูลจริงหรือเอกสารแยกต่างหาก

ลบข้อมูลทดสอบที่ผิดออกก่อน:

```sql
DELETE FROM orders WHERE status IN ('compelted', 'PENDING');
```

แน่นอนว่าเราสามารถแก้ปัญหานี้ได้บางส่วนด้วย `CHECK` constraint:

```sql
ALTER TABLE orders
    ADD CONSTRAINT orders_status_check
    CHECK (status IN ('pending', 'processing', 'shipped', 'delivered', 'cancelled', 'refunded'));
```

วิธีนี้ช่วยป้องกันค่าที่ไม่ถูกต้องได้ก็จริง แต่ก็ยังมีข้อจำกัด:

- ถ้ามีคอลัมน์ "สถานะ" แบบเดียวกันในหลายตาราง (เช่น `payments.status`, `shipments.status`) ต้องเขียน CHECK ซ้ำทุกที่ และถ้าจะแก้ค่าที่อนุญาต ต้องไปแก้ทุกตารางเอง
- CHECK แบบนี้ไม่ได้บอก "ลำดับ" ของสถานะ (เช่น `pending` มาก่อน `shipped`) ให้ database engine รู้จัก
- การ debug schema ต้องเปิดดู constraint definition ถึงจะรู้ค่าที่อนุญาต ไม่ได้เห็นชัดเจนจาก type ของคอลัมน์โดยตรง

นี่คือจุดเริ่มต้นที่ PostgreSQL เสนอเครื่องมือที่ทรงพลังกว่านั้น นั่นคือ **custom type** ซึ่งมีอยู่ 3 รูปแบบหลักที่เราจะเรียนในบทนี้:

| ชนิด custom type | ใช้เมื่อไหร่ |
|---|---|
| **ENUM** | ต้องการจำกัดค่าให้เป็นหนึ่งใน "รายการคงที่" เช่น สถานะคำสั่งซื้อ, ระดับสมาชิก |
| **Composite type** | ต้องการรวมหลายฟิลด์ที่สัมพันธ์กันให้เป็น "หน่วยเดียว" เช่น ที่อยู่ (ถนน, เมือง, รหัสไปรษณีย์) |
| **Domain** | ต้องการสร้าง type ใหม่จาก type เดิม พร้อมกฎ (constraint) ที่ใช้ซ้ำได้หลายคอลัมน์/หลายตาราง เช่น อีเมล, เบอร์โทร |

ในบทนี้เราจะเรียนรู้ทั้ง 3 แบบ พร้อมนำไปประยุกต์ใช้กับ schema ของระบบ e-commerce จริง

---

## Step 482: ENUM type — CREATE TYPE ... AS ENUM

**ENUM (enumerated type)** คือ type ที่กำหนดชุดค่าคงที่ (labels) ไว้ล่วงหน้า คอลัมน์ที่เป็น ENUM จะรับได้เฉพาะค่าที่อยู่ในชุดนั้นเท่านั้น

### สร้าง ENUM type

```sql
CREATE TYPE order_status_enum AS ENUM (
    'pending',
    'processing',
    'shipped',
    'delivered',
    'cancelled',
    'refunded'
);
```

ตรวจสอบ type ที่สร้างขึ้น:

```sql
SELECT typname, typtype
FROM pg_type
WHERE typname = 'order_status_enum';
```

```
      typname       | typtype
---------------------+---------
 order_status_enum   | e
```

`typtype = 'e'` หมายถึง enum type

ดูรายการค่าทั้งหมดของ ENUM พร้อมลำดับ (ลำดับมีความหมาย จะอธิบายต่อ):

```sql
SELECT enumlabel, enumsortorder
FROM pg_enum
WHERE enumtypid = 'order_status_enum'::regtype
ORDER BY enumsortorder;
```

```
 enumlabel  | enumsortorder
------------+----------------
 pending    |              1
 processing |              2
 shipped    |              3
 delivered  |              4
 cancelled  |              5
 refunded   |              6
(6 rows)
```

### แปลงคอลัมน์ orders.status จาก VARCHAR เป็น ENUM

ก่อนแปลง ต้องลบ CHECK constraint เดิมที่สร้างไว้ใน Step 481 ออกก่อน (ไม่จำเป็นแล้วเพราะ ENUM จะทำหน้าที่แทน):

```sql
ALTER TABLE orders DROP CONSTRAINT IF EXISTS orders_status_check;
```

จากนั้นใช้ `ALTER TABLE ... ALTER COLUMN ... TYPE` พร้อม `USING` clause เพื่อแปลงข้อมูลที่มีอยู่แล้ว:

```sql
ALTER TABLE orders
    ALTER COLUMN status TYPE order_status_enum
    USING status::order_status_enum;
```

```
ALTER TABLE
```

ถ้าตอนนี้ยังมีข้อมูลเก่าที่ไม่ตรงกับค่าใน ENUM (เช่น `'compelted'` ที่เราลืมลบ) PostgreSQL จะปฏิเสธการแปลงทันที:

```
ERROR:  invalid input value for enum order_status_enum: "compelted"
```

นี่คือจุดแข็งสำคัญ — การแปลง type จะบังคับให้เราต้อง "ทำความสะอาดข้อมูล" ให้ตรงตามกฎก่อนเสมอ

ตั้ง default ใหม่ให้คอลัมน์ (default เดิมที่เป็น string จะหายไปหลัง ALTER TYPE):

```sql
ALTER TABLE orders ALTER COLUMN status SET DEFAULT 'pending'::order_status_enum;
```

ตรวจสอบโครงสร้างคอลัมน์:

```sql
\d orders
```

```
                                       Table "public.orders"
    Column    |           Type            | Collation | Nullable |               Default
--------------+----------------------------+-----------+----------+---------------------------------------
 order_id     | integer                    |           | not null | nextval('orders_order_id_seq'::regclass)
 customer_id  | integer                    |           |          |
 order_date   | timestamp with time zone   |           | not null | now()
 status       | order_status_enum          |           | not null | 'pending'::order_status_enum
 ship_country  | character varying(60)      |           |          |
```

### ทดสอบว่า ENUM ป้องกันค่าที่ไม่ถูกต้องจริง

```sql
INSERT INTO orders (customer_id, status, ship_country)
VALUES (1, 'compelted', 'Thailand');
```

```
ERROR:  invalid input value for enum order_status_enum: "compelted"
LINE 2: VALUES (1, 'compelted', 'Thailand');
                   ^
```

```sql
INSERT INTO orders (customer_id, status, ship_country)
VALUES (1, 'pending', 'Thailand');   -- ค่าที่ถูกต้อง ผ่านได้ปกติ
```

```
INSERT 0 1
```

ลบแถวทดสอบทิ้ง:

```sql
DELETE FROM orders WHERE customer_id = 1 AND ship_country = 'Thailand' AND status = 'pending'
    AND order_id = (SELECT MAX(order_id) FROM orders);
```

### คุณสมบัติพิเศษ: ENUM มีลำดับในตัว (ordering)

จุดเด่นอีกอย่างของ ENUM คือค่าต่างๆ จะเรียงลำดับตาม **ลำดับที่ประกาศตอนสร้าง type** ไม่ใช่ตามตัวอักษร (alphabetical) ทำให้เราใช้ตัวดำเนินการเปรียบเทียบ `<`, `>`, `ORDER BY` ได้อย่างมีความหมายทางธุรกิจ:

```sql
-- เรียงตามลำดับ "ความคืบหน้า" ของสถานะ ไม่ใช่ตามตัวอักษร
SELECT order_id, status
FROM orders
ORDER BY status
LIMIT 8;
```

```
 order_id |   status
----------+-------------
        9 | pending
       14 | pending
        4 | processing
       10 | processing
       17 | processing
        3 | shipped
        8 | shipped
       15 | shipped
```

สังเกตว่า `pending` มาก่อน `processing` มาก่อน `shipped` ซึ่งตรงกับลำดับความคืบหน้าจริง ต่างจาก VARCHAR ที่จะเรียงตามตัวอักษร (`cancelled` จะมาก่อน `delivered` ตามตัวอักษร A-Z)

ใช้เปรียบเทียบหาคำสั่งซื้อที่ "ยังไม่ถึงขั้นจัดส่ง" ได้ง่ายๆ:

```sql
SELECT order_id, status
FROM orders
WHERE status < 'shipped'::order_status_enum
ORDER BY status;
```

```
 order_id |   status
----------+------------
        9 | pending
       14 | pending
        4 | processing
       10 | processing
       17 | processing
(5 rows)
```

> **หมายเหตุ:** query ด้านบนใช้ `<` เปรียบเทียบตามลำดับที่ประกาศ ซึ่งจะ "พัง" ถ้าค่า `cancelled` หรือ `refunded` ถูกจัดลำดับไว้ในตำแหน่งที่ทำให้ความหมายไม่ตรงกับที่ตั้งใจ ดังนั้นการออกแบบลำดับตอนสร้าง ENUM จึงสำคัญมาก ควรวางแผนลำดับให้สอดคล้องกับ business flow จริงตั้งแต่แรก

---

## Step 483: ข้อดีข้อเสียของ ENUM เทียบกับ VARCHAR+CHECK และตาราง lookup แยก

การเลือกว่าจะใช้ ENUM, VARCHAR+CHECK, หรือ lookup table (reference table) เป็นการตัดสินใจด้าน schema design ที่สำคัญ แต่ละวิธีมีข้อดีข้อเสียต่างกัน

### วิธีที่ 1: ENUM type

```sql
CREATE TYPE membership_tier_enum AS ENUM ('bronze', 'silver', 'gold', 'platinum');
```

### วิธีที่ 2: VARCHAR + CHECK constraint

```sql
-- สมมติเป็นอีกทางเลือกสำหรับคอลัมน์เดียวกัน
membership_tier VARCHAR(20) NOT NULL
    CHECK (membership_tier IN ('bronze', 'silver', 'gold', 'platinum'))
```

### วิธีที่ 3: Lookup table (reference table)

```sql
CREATE TABLE membership_tiers (
    tier_code   VARCHAR(20) PRIMARY KEY,
    tier_name   VARCHAR(50) NOT NULL,
    sort_order  INTEGER NOT NULL,
    min_points  INTEGER NOT NULL DEFAULT 0
);

INSERT INTO membership_tiers (tier_code, tier_name, sort_order, min_points) VALUES
    ('bronze', 'Bronze Member', 1, 0),
    ('silver', 'Silver Member', 2, 1000),
    ('gold', 'Gold Member', 3, 5000),
    ('platinum', 'Platinum Member', 4, 20000);

-- ตารางลูกค้าจะอ้างอิงผ่าน foreign key
-- membership_tier VARCHAR(20) REFERENCES membership_tiers(tier_code)
```

### ตารางเปรียบเทียบ

| คุณสมบัติ | ENUM | VARCHAR + CHECK | Lookup table |
|---|---|---|---|
| **ประสิทธิภาพ storage** | ดีมาก (เก็บเป็นเลข 4 byte ภายใน) | ปานกลาง (เก็บเป็น string เต็ม) | ต้อง JOIN เพื่อดึงชื่อเต็ม แต่ FK column เก็บเป็นเลข/string สั้นได้ |
| **ความเร็วในการเปรียบเทียบ/เรียงลำดับ** | เร็วมาก และเรียงตามลำดับธุรกิจได้เลย | ต้องอาศัย `CASE WHEN` เพื่อเรียงตามลำดับธุรกิจ | เร็ว ถ้ามี index บน `sort_order` แต่ต้อง JOIN ก่อน |
| **เพิ่มค่าใหม่** | `ALTER TYPE ... ADD VALUE` (ง่าย แต่มีข้อจำกัด — ดู Step 484) | แก้ CHECK constraint (`DROP` แล้ว `ADD` ใหม่) | `INSERT` แถวใหม่เข้า lookup table (ง่ายที่สุด ไม่ต้องแตะ schema) |
| **ลบ/ปิดใช้งานค่าเดิม** | ทำไม่ได้ตรงๆ ต้องใช้ workaround | แก้ CHECK constraint ใหม่ | `UPDATE` หรือเพิ่มคอลัมน์ `is_active` ในตาราง — ยืดหยุ่นที่สุด |
| **เก็บ metadata เพิ่มเติม** (เช่น คำอธิบาย, สี, ไอคอน, ลำดับ) | ทำไม่ได้ ENUM เก็บได้แค่ label | ทำไม่ได้ | ทำได้เต็มที่ เพราะเป็นตารางปกติ |
| **แชร์ระหว่างหลายคอลัมน์/ตาราง** | ใช้ type เดียวกันได้กับหลายคอลัมน์ | ต้องเขียน CHECK ซ้ำทุกที่ (ซ้ำซ้อน) | หลายตารางทำ FK มาที่ตารางเดียวกันได้ |
| **การเปลี่ยนแปลงกระทบ schema** | ต้องใช้ DDL (`ALTER TYPE`) | ต้องใช้ DDL (`ALTER TABLE ... DROP/ADD CONSTRAINT`) | เปลี่ยนแค่ข้อมูล (DML) ไม่ต้องแตะ DDL เลย |
| **ความชัดเจนเวลาดู schema** | ดูชนิดคอลัมน์ก็รู้ค่าที่เป็นไปได้ทันที (`\d` แสดง type name) | ต้องดู constraint definition แยก | ต้องไปดูตาราง lookup แยก |
| **รองรับ i18n/หลายภาษา** | ไม่รองรับในตัว | ไม่รองรับในตัว | รองรับง่าย เพิ่มคอลัมน์ `tier_name_th`, `tier_name_en` ได้ |
| **ความเร็วในการ query ORM/BI tools** | บาง ORM/BI tool รองรับ ENUM ได้ไม่ดีนัก (ต้อง map เอง) | รองรับดีเพราะเป็น string ธรรมดา | รองรับดีเพราะเป็นตารางปกติ |

### แนวทางการเลือกใช้

```sql
-- แนวทางตัดสินใจอย่างง่าย:

-- ใช้ ENUM เมื่อ:
--   1. ค่าที่เป็นไปได้ "แทบไม่เปลี่ยนแปลง" เลย (เช่น ทิศทาง N/S/E/W, เพศตามเอกสารราชการ)
--   2. ต้องการ ordering ที่มีความหมายทางธุรกิจ (pending < shipped < delivered)
--   3. ต้องการ storage/performance ที่ดีที่สุด และคอลัมน์นี้ถูกใช้ query บ่อยมาก
--   4. ไม่ต้องการ metadata เพิ่มเติมนอกจากตัว label เอง

-- ใช้ VARCHAR + CHECK เมื่อ:
--   1. ทีมพัฒนาไม่คุ้นเคยกับ ENUM หรือ ORM ที่ใช้ยังรองรับได้ไม่ดี
--   2. ค่าที่เป็นไปได้มีการเปลี่ยนแปลงบ้าง (2-3 ครั้งต่อปี) แต่ไม่บ่อยมาก
--   3. ต้องการความง่ายในการ debug (อ่าน CHECK ได้ตรงไปตรงมา)

-- ใช้ Lookup table เมื่อ:
--   1. ค่าที่เป็นไปได้ "เปลี่ยนบ่อย" หรือถูกจัดการโดย admin ผ่านหน้าเว็บ (ไม่ต้องแก้โค้ด/deploy)
--   2. ต้องการเก็บ metadata เพิ่มเติม เช่น คำอธิบาย, สี, ไอคอน, ลำดับการแสดงผล, หลายภาษา
--   3. ต้องการยืดหยุ่นสูงสุด และค่ามีจำนวนมาก (เช่น รายชื่อประเทศ, สกุลเงิน, หมวดหมู่สินค้า)
```

สำหรับกรณี `orders.status` ในหลักสูตรนี้ ENUM เหมาะสมมากเพราะสถานะคำสั่งซื้อมีชุดค่าที่ค่อนข้างตายตัว เปลี่ยนแปลงไม่บ่อย และต้องการ ordering ที่ตรงกับ business flow อย่างชัดเจน

---

## Step 484: การเพิ่มค่าใหม่ใน ENUM ภายหลัง (ALTER TYPE ... ADD VALUE)

ธุรกิจเปลี่ยนแปลงตลอดเวลา สมมติว่าทีมปฏิบัติการต้องการเพิ่มสถานะใหม่ `'on_hold'` (คำสั่งซื้อที่ถูกพักไว้ชั่วคราว เช่น รอตรวจสอบการชำระเงิน) ซึ่งควรอยู่ระหว่าง `'processing'` กับ `'shipped'`

### เพิ่มค่าใหม่ด้วย ALTER TYPE ... ADD VALUE

```sql
ALTER TYPE order_status_enum ADD VALUE 'on_hold' AFTER 'processing';
```

```
ALTER TYPE
```

ตรวจสอบลำดับใหม่:

```sql
SELECT enumlabel, enumsortorder
FROM pg_enum
WHERE enumtypid = 'order_status_enum'::regtype
ORDER BY enumsortorder;
```

```
 enumlabel  | enumsortorder
------------+----------------
 pending    |              1
 processing |              2
 on_hold    |            2.5
 shipped    |              3
 delivered  |              4
 cancelled  |              5
 refunded   |              6
(7 rows)
```

สังเกตว่า `enumsortorder` ของ `on_hold` เป็น `2.5` — PostgreSQL ใช้ตัวเลขทศนิยมแทรกระหว่างค่าเดิม แทนที่จะ renumber ทั้งหมด ทำให้การแทรกค่าใหม่ทำได้เร็วโดยไม่กระทบค่าที่มีอยู่

สามารถระบุตำแหน่งได้ทั้ง `BEFORE` และ `AFTER`:

```sql
-- ตัวอย่างเพิ่มค่าไว้ก่อนค่าใดค่าหนึ่ง
ALTER TYPE order_status_enum ADD VALUE 'awaiting_payment' BEFORE 'pending';

-- ถ้าไม่ระบุตำแหน่ง จะถูกเพิ่มไว้ท้ายสุดของ enum เสมอ
ALTER TYPE order_status_enum ADD VALUE 'return_requested';
```

ทดสอบใช้งานค่าใหม่:

```sql
UPDATE orders SET status = 'on_hold' WHERE order_id = 9;

SELECT order_id, status FROM orders WHERE order_id = 9;
```

```
 order_id | status
----------+---------
        9 | on_hold
```

### ข้อจำกัดสำคัญที่ 1: ห้ามใช้ค่าใหม่ในทรานแซคชันเดียวกับที่เพิ่งเพิ่มค่า

ตั้งแต่ PostgreSQL 12 เป็นต้นไป `ALTER TYPE ... ADD VALUE` สามารถรันอยู่ภายใน transaction block ได้ (ก่อนหน้านั้นทำไม่ได้เลย) **แต่ยังมีข้อจำกัดว่า ห้ามใช้ค่าที่เพิ่งเพิ่มใหม่ภายในทรานแซคชันเดียวกัน**:

```sql
BEGIN;

ALTER TYPE order_status_enum ADD VALUE 'awaiting_customs';

-- พยายามใช้ค่าที่เพิ่งเพิ่มทันทีในทรานแซคชันเดียวกัน
UPDATE orders SET status = 'awaiting_customs' WHERE order_id = 10;

COMMIT;
```

```
ERROR:  unsafe use of new value "awaiting_customs" of enum type order_status_enum
HINT:  New enum values must be committed before they can be used.
```

วิธีแก้คือแยกเป็นสองทรานแซคชัน (หรือปล่อยให้ `ALTER TYPE` เป็น auto-commit statement เดี่ยวๆ แล้วค่อยรัน UPDATE แยกต่างหาก):

```sql
-- ทรานแซคชันที่ 1: เพิ่มค่าใหม่ และ commit ก่อน
ALTER TYPE order_status_enum ADD VALUE 'awaiting_customs';
-- (อัตโนมัติ commit ทันทีถ้ารันนอก BEGIN/COMMIT)

-- ทรานแซคชันที่ 2 (แยกต่างหาก): ใช้ค่าใหม่ได้แล้ว
UPDATE orders SET status = 'awaiting_customs' WHERE order_id = 10;
```

```
UPDATE 1
```

> **ข้อควรระวังในการ deploy จริง:** เครื่องมือ migration หลายตัว (เช่น Flyway, migrate, Django migrations) จะห่อ migration ทั้งไฟล์ไว้ใน transaction เดียว ถ้าไฟล์ migration มีทั้ง `ALTER TYPE ... ADD VALUE` และ statement ที่ใช้ค่าใหม่ในไฟล์เดียวกัน จะทำให้ deploy ล้มเหลว ต้องแยกเป็นสอง migration file หรือปิด wrap-in-transaction สำหรับ migration นั้นโดยเฉพาะ

### ข้อจำกัดสำคัญที่ 2: ลบค่าออกจาก ENUM ไม่ได้โดยตรง

PostgreSQL **ไม่มีคำสั่ง** `ALTER TYPE ... DROP VALUE` ให้ใช้ เพราะการลบค่าอาจทำให้ข้อมูลที่มีอยู่แล้วอ้างถึงค่าที่ไม่มีอยู่จริง (invalid reference) ซึ่งเป็นเรื่องอันตรายมาก

```sql
ALTER TYPE order_status_enum DROP VALUE 'refunded';
```

```
ERROR:  syntax error at or near "DROP"
LINE 1: ALTER TYPE order_status_enum DROP VALUE 'refunded';
                                      ^
```

คำสั่งนี้ไม่มีอยู่จริงใน PostgreSQL ถ้าต้องการ "ลบ" ค่าออกจริงๆ ต้องทำ workaround ด้วยการสร้าง type ใหม่แล้วย้ายข้อมูล:

```sql
-- Workaround: สร้าง type ใหม่ที่ไม่มีค่าที่ต้องการลบ แล้วย้ายคอลัมน์ไปใช้ type ใหม่

-- ขั้นที่ 1: สร้าง type ใหม่ (สมมติต้องการลบ 'return_requested' ที่ไม่ได้ใช้งานจริง)
CREATE TYPE order_status_enum_v2 AS ENUM (
    'awaiting_payment', 'pending', 'processing', 'on_hold',
    'awaiting_customs', 'shipped', 'delivered', 'cancelled', 'refunded'
);

-- ขั้นที่ 2: แปลงคอลัมน์ไปใช้ type ใหม่
-- (ถ้ามีแถวที่ใช้ค่าที่กำลังจะถูกลบอยู่ ต้อง UPDATE ให้เป็นค่าอื่นก่อน ไม่เช่นนั้นจะ error)
ALTER TABLE orders
    ALTER COLUMN status TYPE order_status_enum_v2
    USING status::text::order_status_enum_v2;

-- ขั้นที่ 3: ลบ type เก่า แล้ว rename type ใหม่ให้ใช้ชื่อเดิม
DROP TYPE order_status_enum;
ALTER TYPE order_status_enum_v2 RENAME TO order_status_enum;
```

```
ALTER TABLE
DROP TYPE
ALTER TYPE
```

ตรวจสอบผลลัพธ์:

```sql
SELECT enumlabel FROM pg_enum
WHERE enumtypid = 'order_status_enum'::regtype
ORDER BY enumsortorder;
```

```
    enumlabel
-------------------
 awaiting_payment
 pending
 processing
 on_hold
 awaiting_customs
 shipped
 delivered
 cancelled
 refunded
(9 rows)
```

> **สรุปข้อจำกัดของ ENUM:** เพิ่มค่าใหม่ทำได้ง่ายด้วย `ALTER TYPE ... ADD VALUE` แต่ต้องระวังเรื่อง transaction และค่าใหม่ต้อง commit ก่อนใช้งาน ส่วนการลบค่าต้องทำผ่าน workaround ที่ค่อนข้างยุ่งยาก นี่คือเหตุผลสำคัญที่ทำให้บางทีมเลือกใช้ lookup table แทน ถ้าคาดว่าค่าที่เป็นไปได้จะเปลี่ยนแปลงบ่อยหรือถูกลบออกในอนาคต

---

## Step 485: Composite Type — CREATE TYPE ... AS (...)

**Composite type** คือ type ที่รวมหลายฟิลด์ (แต่ละฟิลด์มี type ของตัวเอง) เข้าเป็นชนิดข้อมูลเดียว คล้ายกับ `struct` ในภาษาโปรแกรมมิ่งทั่วไป หรือคล้ายกับแถวหนึ่งแถวของตาราง (อันที่จริง ทุกตารางใน PostgreSQL จะมี composite type ของตัวเองโดยอัตโนมัติอยู่แล้ว)

### สร้าง Composite type

```sql
CREATE TYPE address_type AS (
    street       VARCHAR(150),
    city         VARCHAR(60),
    postal_code  VARCHAR(10),
    country      VARCHAR(60)
);
```

ตรวจสอบ type ที่สร้าง:

```sql
SELECT typname, typtype FROM pg_type WHERE typname = 'address_type';
```

```
   typname     | typtype
----------------+---------
 address_type   | c
```

`typtype = 'c'` หมายถึง composite type

ดูรายละเอียดฟิลด์ทั้งหมด:

```sql
\d address_type
```

```
Composite type "public.address_type"
    Column    |         Type          | Collation | Nullable | Default
--------------+------------------------+-----------+----------+---------
 street       | character varying(150) |           |          |
 city         | character varying(60)  |           |          |
 postal_code  | character varying(10)  |           |          |
 country      | character varying(60)  |           |          |
```

### ใช้ composite type เป็นชนิดข้อมูลของคอลัมน์

ลองเพิ่มคอลัมน์ที่อยู่จัดส่งให้ตาราง `orders` โดยใช้ composite type แทนที่จะแยกเป็นหลายคอลัมน์:

```sql
ALTER TABLE orders ADD COLUMN shipping_address address_type;
```

การ insert ค่าให้ composite column ใช้ syntax `ROW(...)` หรือวงเล็บ `(...)`:

```sql
UPDATE orders
SET shipping_address = ROW('123 Sukhumvit Road', 'Bangkok', '10110', 'Thailand')
WHERE order_id = 1;

UPDATE orders
SET shipping_address = ('456 5th Avenue', 'New York', '10001', 'USA')
WHERE order_id = 5;
```

```
UPDATE 1
UPDATE 1
```

### เข้าถึงฟิลด์ย่อยของ composite column

การอ้างถึงฟิลด์ย่อยต้องใส่วงเล็บรอบชื่อคอลัมน์ก่อนใช้ `.` (dot notation) เพราะไม่เช่นนั้น PostgreSQL parser จะสับสนกับการอ้างถึง `table.column`:

```sql
SELECT
    order_id,
    (shipping_address).street,
    (shipping_address).city,
    (shipping_address).country
FROM orders
WHERE shipping_address IS NOT NULL;
```

```
 order_id |        street         |   city   | country
----------+-------------------------+----------+----------
        1 | 123 Sukhumvit Road      | Bangkok  | Thailand
        5 | 456 5th Avenue          | New York | USA
(2 rows)
```

ถ้าลืมใส่วงเล็บจะเกิด error:

```sql
SELECT shipping_address.street FROM orders WHERE order_id = 1;
```

```
ERROR:  missing FROM-clause entry for table "shipping_address"
LINE 1: SELECT shipping_address.street FROM orders WHERE order_id ...
               ^
```

แสดงค่า composite ทั้งก้อนในรูปแบบข้อความ:

```sql
SELECT order_id, shipping_address FROM orders WHERE order_id IN (1, 5);
```

```
 order_id |                    shipping_address
----------+----------------------------------------------------------
        1 | ("123 Sukhumvit Road",Bangkok,10110,Thailand)
        5 | ("456 5th Avenue","New York",10001,USA)
(2 rows)
```

### filter/update ฟิลด์ย่อยของ composite type

```sql
-- หาคำสั่งซื้อที่จัดส่งไปยังประเทศไทยผ่าน composite field
SELECT order_id, (shipping_address).city
FROM orders
WHERE (shipping_address).country = 'Thailand';
```

```
 order_id |  city
----------+---------
        1 | Bangkok
(1 row)
```

```sql
-- อัปเดตเฉพาะฟิลด์ย่อยตัวเดียว โดยคงฟิลด์อื่นไว้
UPDATE orders
SET shipping_address.postal_code = '10330'
WHERE order_id = 1;

SELECT order_id, shipping_address FROM orders WHERE order_id = 1;
```

```
 order_id |                    shipping_address
----------+----------------------------------------------------------
        1 | ("123 Sukhumvit Road",Bangkok,10330,Thailand)
(1 row)
```

> **หมายเหตุ:** syntax `UPDATE ... SET column.field = value` สำหรับอัปเดตฟิลด์ย่อยของ composite type โดยตรงนี้รองรับตั้งแต่ PostgreSQL 14 เป็นต้นไป

### composite type ที่ generated จากตารางโดยอัตโนมัติ

เกร็ดความรู้ที่น่าสนใจ: ทุกตารางที่สร้างขึ้นใน PostgreSQL จะมี composite type ที่มีชื่อเดียวกับตารางเกิดขึ้นโดยอัตโนมัติ (ใช้แทนแถวหนึ่งแถวของตารางนั้น) เราใช้ประโยชน์จากสิ่งนี้ได้ในฟังก์ชัน:

```sql
-- ตัวแปรที่มี type เป็น "customers" หมายถึง หนึ่งแถวของตาราง customers ทั้งแถว
DO $$
DECLARE
    one_customer customers;
BEGIN
    SELECT * INTO one_customer FROM customers WHERE customer_id = 1;
    RAISE NOTICE 'Customer: % %, email: %', one_customer.first_name, one_customer.last_name, one_customer.email;
END;
$$;
```

```
NOTICE:  Customer: Somchai Jaidee, email: somchai.j@example.com
DO
```

---

## Step 486: ใช้ Composite Type เป็น parameter/return type ของฟังก์ชัน

จุดแข็งที่แท้จริงของ composite type ปรากฏชัดเมื่อนำไปใช้กับฟังก์ชัน — เราสามารถส่ง "ที่อยู่ทั้งก้อน" เป็น parameter เดียว หรือคืนค่าเป็น "หลายฟิลด์รวมกัน" ได้อย่างเป็นธรรมชาติ

### ฟังก์ชันที่รับ composite type เป็น parameter

```sql
CREATE OR REPLACE FUNCTION format_address(addr address_type)
RETURNS TEXT
LANGUAGE plpgsql
IMMUTABLE
AS $$
BEGIN
    RETURN COALESCE(addr.street, '') || ', ' ||
           COALESCE(addr.city, '') || ' ' ||
           COALESCE(addr.postal_code, '') || ', ' ||
           COALESCE(addr.country, '');
END;
$$;
```

ทดสอบเรียกใช้ด้วยค่า literal:

```sql
SELECT format_address(ROW('99 Rama IX Road', 'Bangkok', '10320', 'Thailand')::address_type);
```

```
                    format_address
--------------------------------------------------------
 99 Rama IX Road, Bangkok 10320, Thailand
(1 row)
```

ใช้กับคอลัมน์ composite ในตารางโดยตรง:

```sql
SELECT order_id, format_address(shipping_address) AS formatted_address
FROM orders
WHERE shipping_address IS NOT NULL;
```

```
 order_id |            formatted_address
----------+-------------------------------------------
        1 | 99 Rama IX Road, Bangkok 10330, Thailand
        5 | 456 5th Avenue, New York 10001, USA
(2 rows)
```

### ฟังก์ชันที่ return composite type

สร้างฟังก์ชันที่รับ input แยกส่วน แล้วประกอบเป็น composite type ส่งกลับ — มีประโยชน์เวลาต้องการ validate/normalize ข้อมูลที่อยู่ก่อนบันทึก:

```sql
CREATE OR REPLACE FUNCTION build_thai_address(
    p_street       VARCHAR,
    p_city         VARCHAR,
    p_postal_code  VARCHAR
)
RETURNS address_type
LANGUAGE plpgsql
IMMUTABLE
AS $$
DECLARE
    result address_type;
BEGIN
    IF p_postal_code !~ '^\d{5}$' THEN
        RAISE EXCEPTION 'Invalid Thai postal code: %', p_postal_code;
    END IF;

    result.street      := trim(p_street);
    result.city        := trim(p_city);
    result.postal_code := p_postal_code;
    result.country     := 'Thailand';

    RETURN result;
END;
$$;
```

ทดสอบใช้งาน:

```sql
SELECT build_thai_address('  200 Silom Road  ', 'Bangkok', '10500');
```

```
                build_thai_address
----------------------------------------------------
 ("200 Silom Road",Bangkok,10500,Thailand)
(1 row)
```

ทดสอบ error handling เมื่อรหัสไปรษณีย์ผิดรูปแบบ:

```sql
SELECT build_thai_address('200 Silom Road', 'Bangkok', '105');
```

```
ERROR:  Invalid Thai postal code: 105
```

นำผลลัพธ์ไปใช้บันทึกลงตารางได้โดยตรง:

```sql
UPDATE orders
SET shipping_address = build_thai_address('88 Ratchadamri Road', 'Bangkok', '10330')
WHERE order_id = 13;

SELECT order_id, format_address(shipping_address) FROM orders WHERE order_id = 13;
```

```
 order_id |               format_address
----------+---------------------------------------------
       13 | 88 Ratchadamri Road, Bangkok 10330, Thailand
```

### ฟังก์ชันที่ return หลายค่าแบบ composite (โดยไม่ต้องสร้าง type แยก)

บางครั้งเราต้องการคืนค่าหลายฟิลด์จากฟังก์ชันโดยไม่อยากสร้าง named type ต่างหาก สามารถประกาศ `RETURNS TABLE(...)` หรือใช้ `OUT` parameters ได้ ซึ่งภายใน PostgreSQL ก็สร้าง composite type แบบไม่มีชื่อ (anonymous) ให้อัตโนมัติ:

```sql
CREATE OR REPLACE FUNCTION order_summary(p_order_id INTEGER)
RETURNS TABLE(
    order_id       INTEGER,
    customer_name  TEXT,
    total_items    BIGINT,
    total_amount   NUMERIC
)
LANGUAGE sql
STABLE
AS $$
    SELECT
        o.order_id,
        c.first_name || ' ' || c.last_name,
        COUNT(oi.order_item_id),
        SUM(oi.quantity * oi.unit_price)
    FROM orders o
    JOIN customers c ON c.customer_id = o.customer_id
    LEFT JOIN order_items oi ON oi.order_id = o.order_id
    WHERE o.order_id = p_order_id
    GROUP BY o.order_id, c.first_name, c.last_name;
$$;

SELECT * FROM order_summary(1);
```

```
 order_id | customer_name  | total_items | total_amount
----------+-----------------+--------------+---------------
        1 | Somchai Jaidee  |            2 |       5070.00
(1 row)
```

Composite type จึงเป็นเครื่องมือสำคัญที่เชื่อมระหว่าง "โครงสร้างข้อมูลแบบ struct" กับระบบ relational ของ PostgreSQL ทำให้เขียนฟังก์ชันที่ทำงานกับข้อมูลหลายฟิลด์พร้อมกันได้อย่างเป็นธรรมชาติและอ่านง่าย

---

## Step 487: Domain คืออะไร — สร้าง type ใหม่จาก type เดิมพร้อม constraint

**Domain** คือ type ใหม่ที่สร้างขึ้นจาก type ที่มีอยู่แล้ว (base type) โดยสามารถแนบ constraint, default value และ NOT NULL เข้าไปได้ แนวคิดคือ "ห่อ" กฎทางธุรกิจไว้ในตัว type เอง แทนที่จะเขียน constraint ซ้ำๆ ในทุกคอลัมน์ที่ต้องการกฎเดียวกัน

### syntax พื้นฐานของ CREATE DOMAIN

```sql
CREATE DOMAIN domain_name AS base_type
    [ DEFAULT default_expr ]
    [ NOT NULL | NULL ]
    [ CONSTRAINT constraint_name ] CHECK (expression) [...];
```

### ตัวอย่างแรก: positive_numeric domain

สมมติว่าเรามีหลายคอลัมน์ในระบบที่ต้องเป็น "ตัวเลขบวกเท่านั้น" เช่น `unit_price`, `stock_quantity`, `quantity` — แทนที่จะเขียน `CHECK (unit_price > 0)` ซ้ำในทุกตาราง เราสร้าง domain เดียวแล้วใช้ซ้ำได้:

```sql
CREATE DOMAIN positive_numeric AS NUMERIC(10,2)
    NOT NULL
    CHECK (VALUE > 0);
```

ตรวจสอบ domain ที่สร้าง:

```sql
SELECT typname, typtype, typbasetype::regtype AS base_type, typnotnull
FROM pg_type
WHERE typname = 'positive_numeric';
```

```
      typname      | typtype | base_type |     typnotnull
--------------------+---------+-----------+---------------------
 positive_numeric   | d       | numeric   | t
```

`typtype = 'd'` หมายถึง domain type สังเกตว่า `base_type` คือ `numeric` และ `typnotnull` เป็น `t` (true)

### ใช้ domain กับคอลัมน์ในตาราง

```sql
-- สร้างตารางตัวอย่างเพื่อสาธิต (ไม่กระทบตารางหลักของ schema)
CREATE TABLE product_price_history (
    history_id   SERIAL PRIMARY KEY,
    product_id   INTEGER REFERENCES products(product_id),
    price        positive_numeric,
    changed_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

ทดสอบใส่ค่าที่ถูกต้อง:

```sql
INSERT INTO product_price_history (product_id, price) VALUES (1, 1990.00);
```

```
INSERT 0 1
```

ทดสอบใส่ค่าที่ผิดกฎ (ราคาติดลบหรือเป็นศูนย์):

```sql
INSERT INTO product_price_history (product_id, price) VALUES (1, -50.00);
```

```
ERROR:  value for domain positive_numeric violates check constraint "positive_numeric_check"
```

```sql
INSERT INTO product_price_history (product_id, price) VALUES (1, 0.00);
```

```
ERROR:  value for domain positive_numeric violates check constraint "positive_numeric_check"
```

ทดสอบใส่ค่า NULL (ต้องผิดเพราะกำหนด NOT NULL ไว้ใน domain):

```sql
INSERT INTO product_price_history (product_id, price) VALUES (2, NULL);
```

```
ERROR:  domain positive_numeric does not allow null values
```

### domain ยังคงรองรับ operator/function ของ base type ได้ทุกอย่าง

เพราะ domain เป็นแค่ "ชั้นห่อ" (wrapper) รอบ base type การคำนวณ, การเปรียบเทียบ, และฟังก์ชันต่างๆ ที่ใช้ได้กับ `NUMERIC` จึงใช้ได้กับ `positive_numeric` เช่นกันโดยไม่ต้อง cast:

```sql
SELECT price, price * 1.07 AS price_with_vat, price::TEXT
FROM product_price_history;
```

```
 price   | price_with_vat | price
---------+------------------+---------
 1990.00 |         2129.30 | 1990.00
```

### ปรับปรุง schema ให้ใช้ domain กับคอลัมน์ที่มีอยู่จริง

ลองแปลง `products.unit_price` และ `products.stock_quantity` ให้ใช้ domain (สร้าง domain สำหรับจำนวนที่ไม่ติดลบแยกต่างหาก เพราะ stock อนุญาตให้เป็น 0 ได้):

```sql
CREATE DOMAIN non_negative_int AS INTEGER
    NOT NULL
    CHECK (VALUE >= 0);

ALTER TABLE products
    ALTER COLUMN unit_price TYPE positive_numeric,
    ALTER COLUMN stock_quantity TYPE non_negative_int;
```

```
ALTER TABLE
```

ทดสอบว่ากฎยังทำงานถูกต้องผ่านคอลัมน์จริง:

```sql
UPDATE products SET stock_quantity = -5 WHERE product_id = 1;
```

```
ERROR:  value for domain non_negative_int violates check constraint "non_negative_int_check"
```

```sql
UPDATE products SET unit_price = 0 WHERE product_id = 1;
```

```
ERROR:  value for domain positive_numeric violates check constraint "positive_numeric_check"
```

จะเห็นว่าตอนนี้กฎ "ต้องเป็นบวก" และ "ห้ามติดลบ" ถูกบังคับใช้โดย type ของคอลัมน์เอง ไม่ต้องพึ่ง CHECK constraint แยกในแต่ละตารางอีกต่อไป

---

## Step 488: ตัวอย่าง Domain จริง — email_address domain และ positive_numeric domain

มาดูตัวอย่างที่ใช้งานได้จริงในระบบ e-commerce กันต่อ โดยเน้นที่ domain สำหรับ **อีเมล** ซึ่งเป็นข้อมูลที่ต้องตรวจสอบรูปแบบ (format validation) ด้วย regular expression

### สร้าง email_address domain

```sql
CREATE DOMAIN email_address AS VARCHAR(150)
    CHECK (VALUE ~* '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$');
```

อธิบาย regex: `~*` คือ operator สำหรับ case-insensitive regex match ใน PostgreSQL, pattern นี้ตรวจสอบว่ามีรูปแบบ `local-part@domain.tld` ที่สมเหตุสมผล (ไม่ครอบคลุมทุกกรณีตามมาตรฐาน RFC 5322 เต็มรูปแบบ แต่เพียงพอสำหรับดักข้อมูลผิดพลาดทั่วไปในทางปฏิบัติ)

ทดสอบ domain โดยตรงก่อนนำไปใช้กับตาราง:

```sql
SELECT 'somchai.j@example.com'::email_address;      -- ผ่าน
```

```
     email_address
------------------------
 somchai.j@example.com
```

```sql
SELECT 'not-an-email'::email_address;                -- ไม่ผ่าน
```

```
ERROR:  value for domain email_address violates check constraint "email_address_check"
```

```sql
SELECT 'missing@domain'::email_address;              -- ไม่ผ่าน (ไม่มี .tld)
```

```
ERROR:  value for domain email_address violates check constraint "email_address_check"
```

### นำ email_address domain ไปใช้กับคอลัมน์ customers.email

```sql
ALTER TABLE customers
    ALTER COLUMN email TYPE email_address;
```

```
ALTER TABLE
```

ทดสอบว่าตอนนี้ตาราง `customers` ปฏิเสธอีเมลรูปแบบผิดโดยอัตโนมัติ:

```sql
INSERT INTO customers (first_name, last_name, email, country)
VALUES ('Test', 'User', 'invalid-email-format', 'Thailand');
```

```
ERROR:  value for domain email_address violates check constraint "email_address_check"
LINE 2: VALUES ('Test', 'User', 'invalid-email-format', 'Thailand'...
                                 ^
```

```sql
INSERT INTO customers (first_name, last_name, email, country)
VALUES ('Test', 'User', 'test.user@example.co.th', 'Thailand');
```

```
INSERT 0 1
```

ลบแถวทดสอบทิ้ง:

```sql
DELETE FROM customers WHERE first_name = 'Test' AND last_name = 'User';
```

### สร้าง domain เพิ่มเติมที่มีประโยชน์สำหรับระบบ e-commerce

```sql
-- domain สำหรับรหัสไปรษณีย์ไทย (5 หลักพอดี)
CREATE DOMAIN thai_postal_code AS CHAR(5)
    CHECK (VALUE ~ '^\d{5}$');

-- domain สำหรับ percentage (ใช้กับส่วนลด, VAT rate ฯลฯ) อยู่ในช่วง 0-100
CREATE DOMAIN percentage AS NUMERIC(5,2)
    CHECK (VALUE >= 0 AND VALUE <= 100);

-- domain สำหรับ SKU code (ตัวอักษรพิมพ์ใหญ่ ตัวเลข และขีดกลางเท่านั้น ความยาว 4-20 ตัวอักษร)
CREATE DOMAIN sku_code AS VARCHAR(20)
    CHECK (VALUE ~ '^[A-Z0-9]{2,8}(-[A-Z0-9]{2,8}){1,3}$');
```

ทดสอบ `percentage` domain:

```sql
SELECT 15.5::percentage;    -- ผ่าน
```

```
 percentage
-------------
      15.50
```

```sql
SELECT 150::percentage;     -- ไม่ผ่าน เกิน 100
```

```
ERROR:  value for domain percentage violates check constraint "percentage_check"
```

ทดสอบ `sku_code` domain:

```sql
SELECT 'ELEC-KB-001'::sku_code;    -- ผ่าน
```

```
  sku_code
-------------
 ELEC-KB-001
```

```sql
SELECT 'kb001'::sku_code;          -- ไม่ผ่าน ตัวพิมพ์เล็กและไม่มีขีดกลาง
```

```
ERROR:  value for domain sku_code violates check constraint "sku_code_check"
```

นำ `sku_code` ไปใช้เพิ่มคอลัมน์ใหม่ในตาราง `products`:

```sql
ALTER TABLE products ADD COLUMN sku sku_code;

UPDATE products SET sku = 'ELEC-KB-' || LPAD(product_id::TEXT, 3, '0')
WHERE product_id = 1;

SELECT product_id, product_name, sku FROM products WHERE product_id = 1;
```

```
 product_id |        product_name          |    sku
------------+-------------------------------+-------------
          1 | Wireless Mechanical Keyboard  | ELEC-KB-001
```

### แก้ไข constraint ของ domain ที่สร้างไว้แล้วด้วย ALTER DOMAIN

จุดเด่นสำคัญของ domain คือสามารถแก้กฎได้จากจุดเดียว แล้วมีผลกับทุกคอลัมน์ที่ใช้ domain นั้นทันที:

```sql
-- เพิ่ม constraint ใหม่ให้กับ domain ที่มีอยู่แล้ว (ต้อง validate ข้อมูลเดิมทั้งหมดด้วย)
ALTER DOMAIN email_address
    ADD CONSTRAINT email_max_local_part_length
    CHECK (length(split_part(VALUE, '@', 1)) <= 64);
```

```
ALTER DOMAIN
```

ถ้าข้อมูลที่มีอยู่แล้วไม่ผ่านกฎใหม่ PostgreSQL จะแจ้ง error ทันทีและไม่ยอมให้เพิ่ม constraint:

```
ERROR:  column "email" of table "customers" contains values that violate the new constraint
```

ลบ constraint ที่เพิ่งเพิ่มออก (สาธิตวิธี `DROP CONSTRAINT` บน domain):

```sql
ALTER DOMAIN email_address DROP CONSTRAINT email_max_local_part_length;
```

```
ALTER DOMAIN
```

---

## Step 489: Domain เทียบกับ CHECK constraint ตรงๆ บนคอลัมน์

ทั้ง Domain และ CHECK constraint ตรงบนคอลัมน์ต่างก็ทำหน้าที่ "ตรวจสอบความถูกต้องของข้อมูล" ได้เหมือนกัน คำถามคือเมื่อไหร่ควรเลือกใช้แบบไหน

### เปรียบเทียบทั้งสองวิธีด้วยตัวอย่างเดียวกัน

**แบบที่ 1: CHECK constraint ตรงบนคอลัมน์**

```sql
CREATE TABLE suppliers_v2 (
    supplier_id    SERIAL PRIMARY KEY,
    supplier_name  VARCHAR(150) NOT NULL,
    contact_email  VARCHAR(150) CHECK (contact_email ~* '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$'),
    country        VARCHAR(60)
);
```

**แบบที่ 2: ใช้ domain ที่มีอยู่แล้ว**

```sql
CREATE TABLE suppliers_v3 (
    supplier_id    SERIAL PRIMARY KEY,
    supplier_name  VARCHAR(150) NOT NULL,
    contact_email  email_address,   -- ใช้ domain ที่สร้างไว้แล้วใน Step 488
    country        VARCHAR(60)
);
```

ทั้งสองแบบให้ผลลัพธ์การตรวจสอบเหมือนกันทุกประการ แต่ต่างกันตรงที่ **แบบที่ 2 ใช้ซ้ำได้** ไม่ต้องเขียน regex ซ้ำทุกตารางที่มีคอลัมน์อีเมล

### ตารางเปรียบเทียบ

| ประเด็น | CHECK constraint บนคอลัมน์ | Domain |
|---|---|---|
| **Reusability (ใช้ซ้ำ)** | ต้อง copy-paste กฎไปทุกตาราง/คอลัมน์ที่ต้องการ | เขียนครั้งเดียว ใช้ได้กับทุกคอลัมน์ที่ประกาศเป็น domain นั้น |
| **จุดศูนย์กลางในการแก้ไขกฎ** | ต้องไล่แก้ทุกตารางที่มี constraint ซ้ำกัน | แก้ที่ `ALTER DOMAIN` จุดเดียว มีผลทุกที่ที่ใช้ domain |
| **ความชัดเจนของ schema** | ต้องเปิดดู constraint definition ของแต่ละคอลัมน์ | เห็นชื่อ type ก็รู้ทันทีว่าคอลัมน์นี้มีความหมาย/กฎอะไร (self-documenting) |
| **ผูกกับคอลัมน์เดียวหรือหลายคอลัมน์** | เหมาะกับกฎที่เกี่ยวข้องกับ "หลายคอลัมน์ในแถวเดียวกัน" เช่น `CHECK (start_date < end_date)` | เหมาะกับกฎที่ตรวจสอบ "ค่าเดียว" อย่างอิสระ ไม่เหมาะกับกฎที่ต้องเทียบกับคอลัมน์อื่น |
| **NOT NULL / DEFAULT ในตัว** | ต้องประกาศแยกที่ระดับคอลัมน์ | สามารถฝัง NOT NULL และ DEFAULT ไว้ใน domain ได้เลย |
| **การเปลี่ยนแปลง type จากภายนอก** | เปลี่ยน type ของคอลัมน์ตรงๆ ด้วย `ALTER TABLE ... TYPE` | เปลี่ยนผ่าน `ALTER DOMAIN` แต่การ validate ข้อมูลเดิมทั้งหมดอาจใช้เวลานานถ้าตารางใหญ่ |
| **ความเข้ากันได้กับเครื่องมือ/ORM** | เข้ากันได้ดีมาก เพราะเป็นแค่ constraint ปกติ | บาง ORM/schema-introspection tool อาจมองข้าม domain แล้วเห็นแค่ base type |
| **Cast และ type compatibility** | ไม่มีปัญหา เพราะ type ของคอลัมน์เป็น base type ตรงๆ | บางครั้งต้อง cast ชัดเจน (`::base_type`) เมื่อส่งผ่านฟังก์ชันที่ overload หลาย type |

### กฎง่ายๆ ในการตัดสินใจ

```sql
-- ใช้ Domain เมื่อ:
--   1. กฎเดียวกันถูกใช้ซ้ำในหลายคอลัมน์ หรือหลายตาราง (เช่น email, phone, positive amount)
--   2. ต้องการให้ schema อ่านง่าย เห็น type ก็เข้าใจความหมายทันที
--   3. กฎตรวจสอบเฉพาะ "ค่าในคอลัมน์นั้นเพียงค่าเดียว" ไม่เกี่ยวกับคอลัมน์อื่นในแถวเดียวกัน

-- ใช้ CHECK constraint ตรงบนคอลัมน์/ตารางเมื่อ:
--   1. กฎเป็นเรื่องเฉพาะของตารางนั้นตารางเดียว ไม่ได้ใช้ซ้ำที่ไหนอีก
--   2. กฎต้องเปรียบเทียบระหว่างหลายคอลัมน์ในแถวเดียวกัน
--      เช่น CHECK (ship_date >= order_date), CHECK (discount_price < unit_price)
--   3. ต้องการความเรียบง่ายสูงสุด ไม่อยากเพิ่ม object ใหม่ในระดับ database schema
```

### ตัวอย่างกฎที่ "ต้องเป็น" CHECK constraint บนตาราง (ทำเป็น Domain ไม่ได้)

```sql
-- กฎนี้เปรียบเทียบสองคอลัมน์ในแถวเดียวกัน → domain ทำไม่ได้ เพราะ domain รู้จักแค่ "หนึ่งค่า"
ALTER TABLE order_items
    ADD CONSTRAINT check_reasonable_quantity
    CHECK (quantity <= 1000);   -- ป้องกัน order ผิดปกติ เช่น สั่ง 1 ล้านชิ้น

-- ตัวอย่างกฎข้ามคอลัมน์ที่ domain ไม่สามารถทำได้เลย (ต้องใช้ table-level CHECK เท่านั้น)
-- CHECK (actual_delivery_date >= order_date)
```

Domain กับ CHECK constraint จึงไม่ใช่คู่แข่งที่ต้องเลือกอย่างใดอย่างหนึ่งเสมอไป ในทางปฏิบัติ ทีมงานระดับมืออาชีพมักใช้ **ทั้งสองแบบร่วมกัน**: ใช้ Domain สำหรับกฎระดับ "ค่าเดี่ยว" ที่ใช้ซ้ำบ่อย และใช้ CHECK constraint ระดับตารางสำหรับกฎที่เกี่ยวข้องกับความสัมพันธ์ระหว่างคอลัมน์

---

## Step 490: แบบฝึกหัดรวม — ปรับปรุง schema ระบบ e-commerce ด้วย ENUM, Domain และ Composite Type

มาถึงขั้นตอนสุดท้าย เราจะรวบยอดทุกอย่างที่เรียนมาในบทนี้ เพื่อปรับปรุง schema ของระบบ e-commerce ให้ปลอดภัยและอ่านง่ายขึ้นอย่างเป็นระบบ

### สรุปแผนการปรับปรุง

| ส่วนที่จะปรับปรุง | เครื่องมือที่ใช้ | เหตุผล |
|---|---|---|
| `orders.status` | ENUM (`order_status_enum`) | ค่าคงที่ ต้องการ ordering ทำไปแล้วใน Step 482 |
| `customers.email` | Domain (`email_address`) | ตรวจสอบ format ใช้ซ้ำได้หลายตาราง ทำไปแล้วใน Step 488 |
| `products.unit_price`, `stock_quantity` | Domain (`positive_numeric`, `non_negative_int`) | ทำไปแล้วใน Step 487 |
| `orders.shipping_address` | Composite type (`address_type`) | รวมที่อยู่หลายฟิลด์เป็นหน่วยเดียว ทำไปแล้วใน Step 485-486 |
| `customers.country`, `suppliers.country`, `orders.ship_country` | Domain (`country_code`) ใหม่ | ยังไม่เคยทำ — จะสร้างในส่วนนี้ |
| `order_items.quantity` | Domain (`positive_int`) ใหม่ | ยังไม่เคยทำ — จะสร้างในส่วนนี้ |
| ระดับสมาชิกลูกค้า (เพิ่มใหม่) | ENUM (`customer_tier_enum`) | สาธิตการเพิ่มฟีเจอร์ใหม่ที่ใช้ ENUM ตั้งแต่แรก |

### ขั้นที่ 1: สร้าง domain สำหรับรหัสประเทศและจำนวนเต็มบวก

```sql
-- domain สำหรับรหัสประเทศ ISO 3166-1 alpha-2 (2 ตัวอักษรพิมพ์ใหญ่)
-- (ในระบบจริงเราจะเก็บชื่อประเทศเต็มไว้ใน seed data แล้ว
--  ตัวอย่างนี้สาธิตด้วยการสร้าง domain ใหม่ชื่อ non_empty_text แทน
--  เพื่อไม่ต้องเปลี่ยนข้อมูล seed ที่มีอยู่)
CREATE DOMAIN non_empty_text AS VARCHAR(60)
    CHECK (length(trim(VALUE)) > 0);

-- domain สำหรับจำนวนเต็มบวก (ใช้กับ order_items.quantity)
CREATE DOMAIN positive_int AS INTEGER
    CHECK (VALUE > 0);
```

### ขั้นที่ 2: ปรับปรุงคอลัมน์ quantity และ country ให้ใช้ domain

```sql
ALTER TABLE order_items
    ALTER COLUMN quantity TYPE positive_int;

ALTER TABLE customers
    ALTER COLUMN country TYPE non_empty_text;

ALTER TABLE suppliers
    ALTER COLUMN country TYPE non_empty_text;
```

```
ALTER TABLE
ALTER TABLE
ALTER TABLE
```

### ขั้นที่ 3: เพิ่มฟีเจอร์ระดับสมาชิกลูกค้าด้วย ENUM

```sql
CREATE TYPE customer_tier_enum AS ENUM ('standard', 'silver', 'gold', 'platinum');

ALTER TABLE customers
    ADD COLUMN membership_tier customer_tier_enum NOT NULL DEFAULT 'standard';

-- กำหนดระดับสมาชิกให้ลูกค้าบางรายตามยอดสั่งซื้อสะสม (ตัวอย่าง business logic)
UPDATE customers c
SET membership_tier = CASE
    WHEN total_spent >= 20000 THEN 'platinum'
    WHEN total_spent >= 10000 THEN 'gold'
    WHEN total_spent >= 3000  THEN 'silver'
    ELSE 'standard'
END::customer_tier_enum
FROM (
    SELECT o.customer_id, SUM(oi.quantity * oi.unit_price) AS total_spent
    FROM orders o
    JOIN order_items oi ON oi.order_id = o.order_id
    GROUP BY o.customer_id
) spend
WHERE spend.customer_id = c.customer_id;
```

```
ALTER TYPE
ALTER TABLE
UPDATE 12
```

ตรวจสอบผลลัพธ์:

```sql
SELECT customer_id, first_name, last_name, membership_tier
FROM customers
ORDER BY membership_tier DESC, customer_id
LIMIT 8;
```

```
 customer_id | first_name |  last_name   | membership_tier
-------------+------------+--------------+------------------
           7 | Yuki       | Tanaka       | gold
           2 | Pranee     | Suksawat     | silver
           3 | Kittipong  | Wongchai     | silver
           5 | John       | Smith        | silver
          11 | Anna       | Mueller      | silver
           1 | Somchai    | Jaidee       | silver
          12 | Lars       | Andersson    | standard
           4 | Nutcha     | Rattanakorn  | standard
(8 rows)
```

### ขั้นที่ 4: สร้าง view สรุปที่แสดงผลจากทั้ง ENUM, Domain และ Composite Type ร่วมกัน

```sql
CREATE OR REPLACE VIEW v_order_full_details AS
SELECT
    o.order_id,
    o.status,                                    -- ENUM
    o.order_date,
    c.first_name || ' ' || c.last_name AS customer_name,
    c.email,                                      -- Domain (email_address)
    c.membership_tier,                             -- ENUM
    format_address(o.shipping_address) AS shipping_address_formatted,  -- Composite type + function
    COUNT(oi.order_item_id)  AS total_line_items,
    SUM(oi.quantity * oi.unit_price) AS order_total
FROM orders o
JOIN customers c ON c.customer_id = o.customer_id
LEFT JOIN order_items oi ON oi.order_id = o.order_id
GROUP BY o.order_id, o.status, o.order_date, c.first_name, c.last_name,
         c.email, c.membership_tier, o.shipping_address;
```

```sql
SELECT order_id, status, customer_name, membership_tier, shipping_address_formatted, order_total
FROM v_order_full_details
WHERE shipping_address_formatted IS NOT NULL
ORDER BY order_id;
```

```
 order_id |  status  | customer_name  | membership_tier |          shipping_address_formatted           | order_total
----------+----------+-----------------+-------------------+-------------------------------------------------+--------------
        1 | delivered | Somchai Jaidee  | silver            | 99 Rama IX Road, Bangkok 10330, Thailand         |     5070.00
        5 | delivered | John Smith      | silver            | 456 5th Avenue, New York 10001, USA              |    10580.00
       13 | pending   | Priya Sharma    | standard          | 88 Ratchadamri Road, Bangkok 10330, Thailand     |     5000.00
(3 rows)
```

จาก schema เดิมที่ใช้ `VARCHAR` แทบทุกคอลัมน์ ตอนนี้ database ของเรากลายเป็น schema ที่ **บังคับกฎทางธุรกิจในระดับ type ของคอลัมน์เอง** ทั้งสถานะคำสั่งซื้อที่ถูกจำกัดด้วย ENUM, อีเมลที่ต้องผ่าน regex, ราคาที่ต้องเป็นบวก, จำนวนสินค้าที่ต้องมากกว่าศูนย์ และที่อยู่จัดส่งที่รวมเป็นหน่วยเดียวกันอย่างเป็นระเบียบ — ทั้งหมดนี้ทำให้ข้อมูลผิดพลาดถูกดักจับได้ตั้งแต่ระดับ database ไม่ต้องพึ่งการตรวจสอบจากฝั่ง application เพียงอย่างเดียว

---

## สรุปท้ายบท

ในบทนี้เราได้เรียนรู้เครื่องมือสำคัญ 3 อย่างที่ PostgreSQL มอบให้สำหรับสร้าง **custom data type** เพื่อบังคับ business rule ในระดับฐานข้อมูลโดยตรง:

1. **ENUM type** (`CREATE TYPE ... AS ENUM`) เหมาะสำหรับคอลัมน์ที่มีค่าคงที่จำนวนจำกัด เช่น สถานะคำสั่งซื้อ ให้ประโยชน์ด้าน storage ที่กะทัดรัด และ ordering ที่มีความหมายทางธุรกิจ แต่มีข้อจำกัดเรื่องการเพิ่ม/ลบค่าภายหลัง — เพิ่มค่าทำได้ด้วย `ALTER TYPE ... ADD VALUE` (ต้อง commit ก่อนใช้ค่าใหม่) แต่ลบค่าทำไม่ได้โดยตรง ต้อง workaround ด้วยการสร้าง type ใหม่

2. **Composite type** (`CREATE TYPE ... AS (...)`) เหมาะสำหรับรวมหลายฟิลด์ที่สัมพันธ์กันให้เป็นหน่วยเดียว เช่น ที่อยู่ ใช้เป็นชนิดข้อมูลของคอลัมน์ หรือเป็น parameter/return type ของฟังก์ชันได้ ทำให้โค้ดที่เกี่ยวกับข้อมูลหลายฟิลด์อ่านง่ายและเป็นธรรมชาติมากขึ้น

3. **Domain** (`CREATE DOMAIN ... AS base_type CHECK (...)`) เหมาะสำหรับสร้างกฎ (constraint) ที่ใช้ซ้ำได้หลายคอลัมน์/หลายตาราง เช่น อีเมล, ตัวเลขบวก, รหัสไปรษณีย์ ทำให้ schema สื่อความหมายชัดเจนขึ้น และแก้กฎจากจุดศูนย์กลางเดียวได้ผ่าน `ALTER DOMAIN`

การเลือกใช้เครื่องมือใดขึ้นอยู่กับสถานการณ์: ENUM เทียบกับ VARCHAR+CHECK เทียบกับ lookup table ต้องพิจารณาความถี่ในการเปลี่ยนแปลงค่าและความต้องการ metadata เพิ่มเติม ส่วน Domain เทียบกับ CHECK constraint ตรงบนคอลัมน์ ต้องพิจารณาว่ากฎนั้นใช้ซ้ำหรือไม่ และเป็นกฎระดับ "ค่าเดียว" หรือ "ข้ามคอลัมน์"

การผสมผสานเครื่องมือทั้งสามอย่างเข้าด้วยกัน อย่างที่เราทำใน Step 490 ทำให้ schema ของระบบ e-commerce มีความปลอดภัยของข้อมูล (data integrity) สูงขึ้นอย่างมาก โดยไม่ต้องพึ่งพา validation logic จากฝั่ง application แต่เพียงอย่างเดียว — นี่คือหลักการสำคัญของการออกแบบฐานข้อมูลระดับมืออาชีพ: "ให้ database เป็นผู้พิทักษ์กฎข้อมูลชั้นสุดท้าย"

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1

สร้าง ENUM type ชื่อ `payment_method_enum` ที่มีค่า `'credit_card'`, `'bank_transfer'`, `'promptpay'`, `'cash_on_delivery'`

<details>
<summary>เฉลย</summary>

```sql
CREATE TYPE payment_method_enum AS ENUM (
    'credit_card',
    'bank_transfer',
    'promptpay',
    'cash_on_delivery'
);
```

</details>

### แบบฝึกหัดที่ 2

เพิ่มคอลัมน์ `payment_method` ให้ตาราง `orders` โดยใช้ type จากแบบฝึกหัดที่ 1 กำหนดค่า default เป็น `'credit_card'` และห้ามเป็น NULL

<details>
<summary>เฉลย</summary>

```sql
ALTER TABLE orders
    ADD COLUMN payment_method payment_method_enum NOT NULL DEFAULT 'credit_card';
```

</details>

### แบบฝึกหัดที่ 3

เพิ่มค่าใหม่ `'wallet'` เข้าไปใน `payment_method_enum` โดยให้อยู่หลัง `'promptpay'`

<details>
<summary>เฉลย</summary>

```sql
ALTER TYPE payment_method_enum ADD VALUE 'wallet' AFTER 'promptpay';
```

ตรวจสอบผลลัพธ์:

```sql
SELECT enumlabel, enumsortorder
FROM pg_enum
WHERE enumtypid = 'payment_method_enum'::regtype
ORDER BY enumsortorder;
```

</details>

### แบบฝึกหัดที่ 4

อธิบายว่าทำไมคำสั่งต่อไปนี้จะเกิด error ถ้ารันในทรานแซคชันเดียวกัน และควรแก้ไขอย่างไร

```sql
BEGIN;
ALTER TYPE payment_method_enum ADD VALUE 'crypto';
UPDATE orders SET payment_method = 'crypto' WHERE order_id = 1;
COMMIT;
```

<details>
<summary>เฉลย</summary>

**สาเหตุ:** PostgreSQL ไม่อนุญาตให้ใช้ค่าที่เพิ่งเพิ่มเข้า ENUM ใหม่ภายในทรานแซคชันเดียวกับที่เพิ่มค่านั้น เพราะค่าใหม่ยังไม่ได้ถูก commit อย่างสมบูรณ์ ระบบจึงไม่สามารถรับประกันความสอดคล้องของข้อมูลได้หากทรานแซคชันถูก rollback

**วิธีแก้:** แยกเป็นสองทรานแซคชัน (หรือสอง statement แยกกันนอก BEGIN/COMMIT เดียวกัน)

```sql
-- statement แรก รันแยก (auto-commit)
ALTER TYPE payment_method_enum ADD VALUE 'crypto';

-- statement ที่สอง รันหลังจากนั้น (แยกทรานแซคชัน)
UPDATE orders SET payment_method = 'crypto' WHERE order_id = 1;
```

</details>

### แบบฝึกหัดที่ 5

สร้าง composite type ชื่อ `contact_info_type` ที่มีฟิลด์ `phone_number VARCHAR(20)` และ `line_id VARCHAR(50)` แล้วเพิ่มเป็นคอลัมน์ใหม่ชื่อ `contact_info` ในตาราง `customers`

<details>
<summary>เฉลย</summary>

```sql
CREATE TYPE contact_info_type AS (
    phone_number  VARCHAR(20),
    line_id       VARCHAR(50)
);

ALTER TABLE customers ADD COLUMN contact_info contact_info_type;

-- ตัวอย่างการใส่ข้อมูล
UPDATE customers
SET contact_info = ROW('081-234-5678', '@somchai_j')
WHERE customer_id = 1;

-- ตัวอย่างการอ่านฟิลด์ย่อย
SELECT customer_id, (contact_info).phone_number, (contact_info).line_id
FROM customers
WHERE contact_info IS NOT NULL;
```

</details>

### แบบฝึกหัดที่ 6

สร้าง domain ชื่อ `phone_number_th` จาก `VARCHAR(15)` ที่ตรวจสอบว่าค่าต้องเป็นเบอร์โทรไทยรูปแบบ `0XXXXXXXXX` (ขึ้นต้นด้วย 0 ตามด้วยตัวเลข 9 หลัก รวม 10 หลัก)

<details>
<summary>เฉลย</summary>

```sql
CREATE DOMAIN phone_number_th AS VARCHAR(15)
    CHECK (VALUE ~ '^0\d{9}$');

-- ทดสอบ
SELECT '0812345678'::phone_number_th;   -- ผ่าน
SELECT '812345678'::phone_number_th;    -- ไม่ผ่าน (ไม่ขึ้นต้นด้วย 0)
SELECT '081-234-5678'::phone_number_th; -- ไม่ผ่าน (มีขีดกลาง)
```

</details>

### แบบฝึกหัดที่ 7

อธิบายความแตกต่างระหว่างการใช้ Domain กับการใช้ CHECK constraint ตรงบนคอลัมน์ ในแง่ของ "การใช้ซ้ำ" (reusability) พร้อมยกตัวอย่างสถานการณ์ที่ควรใช้แต่ละแบบ

<details>
<summary>เฉลย</summary>

**Domain** เหมาะกับกฎที่ต้องใช้ซ้ำในหลายคอลัมน์หรือหลายตาราง เพราะเขียนกฎเพียงครั้งเดียวที่ `CREATE DOMAIN` แล้วนำไปใช้ได้ทุกที่ที่ต้องการ และถ้าต้องแก้กฎภายหลัง สามารถแก้ที่จุดเดียวด้วย `ALTER DOMAIN` มีผลกับทุกคอลัมน์ที่ใช้ domain นั้นทันที เช่น กฎการตรวจสอบรูปแบบอีเมลที่ใช้ทั้งใน `customers.email`, `suppliers.contact_email`, `employees.email`

**CHECK constraint ตรงบนคอลัมน์** เหมาะกับกฎที่ใช้เฉพาะตารางเดียว หรือกฎที่ต้องเปรียบเทียบค่าระหว่างหลายคอลัมน์ในแถวเดียวกัน (ซึ่ง domain ทำไม่ได้ เพราะ domain ตรวจสอบได้แค่ "ค่าเดียว" ที่ส่งเข้ามา) เช่น `CHECK (ship_date >= order_date)` หรือ `CHECK (discount_price < unit_price)`

</details>

### แบบฝึกหัดที่ 8

ตารางเปรียบเทียบข้อดีข้อเสียของ ENUM, VARCHAR+CHECK และ Lookup table ข้อใดที่เป็นข้อได้เปรียบสำคัญที่สุดของ **Lookup table** เหนือ ENUM เมื่อค่าที่เป็นไปได้เปลี่ยนแปลงบ่อย

<details>
<summary>เฉลย</summary>

ข้อได้เปรียบสำคัญที่สุดคือ **การเปลี่ยนแปลงค่าที่เป็นไปได้ทำผ่าน DML (INSERT/UPDATE/DELETE) แทนที่จะต้องทำผ่าน DDL (ALTER TYPE)**

การเพิ่ม/ลด/ปิดใช้งานค่าใน lookup table ทำได้ด้วยการ `INSERT` หรือ `UPDATE is_active = false` แถวในตาราง ซึ่งไม่ต้อง deploy schema migration ใหม่ และสามารถให้ admin จัดการผ่านหน้าเว็บได้โดยตรง ในขณะที่ ENUM ต้องใช้ `ALTER TYPE ... ADD VALUE` ซึ่งเป็นคำสั่ง DDL ที่ต้องมีสิทธิ์ระดับ schema owner และมีข้อจำกัดเรื่อง transaction นอกจากนี้ lookup table ยังลบค่าออกได้ตรงไปตรงมา (หรือ soft-delete ด้วย `is_active`) ในขณะที่ ENUM ไม่มีคำสั่งลบค่าโดยตรงเลย

</details>

### แบบฝึกหัดที่ 9

สร้าง domain ชื่อ `discount_rate` จาก `NUMERIC(4,2)` ที่ต้องมีค่าระหว่าง 0 ถึง 1 (แทนอัตราส่วนลด เช่น 0.15 หมายถึงลด 15%) และห้ามเป็น NULL พร้อม default เป็น 0

<details>
<summary>เฉลย</summary>

```sql
CREATE DOMAIN discount_rate AS NUMERIC(4,2)
    NOT NULL
    DEFAULT 0
    CHECK (VALUE >= 0 AND VALUE <= 1);

-- ทดสอบ
SELECT 0.15::discount_rate;   -- ผ่าน
SELECT 1.5::discount_rate;    -- ไม่ผ่าน เกิน 1

-- ตัวอย่างการนำไปใช้
ALTER TABLE order_items ADD COLUMN discount discount_rate;
```

</details>

### แบบฝึกหัดที่ 10

เขียนคำสั่งสร้าง composite type ชื่อ `money_with_currency` ที่มีฟิลด์ `amount NUMERIC(12,2)` และ `currency_code CHAR(3)` จากนั้นเขียนฟังก์ชัน `format_money(m money_with_currency)` ที่คืนค่าเป็นข้อความรูปแบบ `"1,234.50 THB"` (ใช้ `to_char` เพื่อใส่ comma คั่นหลักพัน)

<details>
<summary>เฉลย</summary>

```sql
CREATE TYPE money_with_currency AS (
    amount         NUMERIC(12,2),
    currency_code  CHAR(3)
);

CREATE OR REPLACE FUNCTION format_money(m money_with_currency)
RETURNS TEXT
LANGUAGE plpgsql
IMMUTABLE
AS $$
BEGIN
    RETURN to_char(m.amount, 'FM999,999,990.00') || ' ' || m.currency_code;
END;
$$;

-- ทดสอบ
SELECT format_money(ROW(1234.50, 'THB')::money_with_currency);
-- ผลลัพธ์: 1,234.50 THB

SELECT format_money(ROW(25990.00, 'USD')::money_with_currency);
-- ผลลัพธ์: 25,990.00 USD
```

</details>

---

**บทถัดไป:** [Part 050 — Array Types และการทำงานกับ Array](./part-050-arrays.md)
