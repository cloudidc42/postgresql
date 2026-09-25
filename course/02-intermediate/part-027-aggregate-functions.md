# ฟังก์ชันรวมกลุ่ม (Aggregate Functions): COUNT, SUM, AVG, MIN, MAX

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับกลาง | Part 027

## เป้าหมายการเรียนรู้

เมื่อเรียนจบบทนี้ ผู้เรียนจะสามารถ:

- อธิบายได้ว่า aggregate function คืออะไร และแตกต่างจากฟังก์ชันทั่วไป (scalar function) อย่างไร
- ใช้ `COUNT(*)`, `COUNT(column)`, และ `COUNT(DISTINCT column)` ได้อย่างถูกต้อง และเข้าใจผลกระทบของค่า `NULL` ต่อผลลัพธ์
- ใช้ `SUM()` รวมค่าตัวเลข พร้อมเข้าใจพฤติกรรมของ `SUM` เมื่อเจอ `NULL` หรือไม่มีแถวเลย
- ใช้ `AVG()` หาค่าเฉลี่ย และเข้าใจว่าเหตุใดผลลัพธ์จึงเป็น `numeric` แม้ input จะเป็น `integer`
- ใช้ `MIN()`/`MAX()` กับข้อมูลได้หลายประเภท ทั้งตัวเลข ข้อความ และวันที่
- ใช้ `FILTER (WHERE ...)` ร่วมกับ aggregate function เพื่อคำนวณค่าแบบมีเงื่อนไขหลายชุดในคิวรีเดียว
- เขียนคิวรีที่รวม aggregate function หลายตัวพร้อมกันเพื่อสรุปข้อมูลเชิงธุรกิจ
- ใช้ `STRING_AGG()` และ `ARRAY_AGG()` เพื่อรวมค่าจากหลายแถวให้เป็นข้อความเดียวหรือ array
- ใช้ statistical aggregate เบื้องต้น เช่น `STDDEV`, `VARIANCE`, `PERCENTILE_CONT`, `PERCENTILE_DISC`
- ประยุกต์ทุกความรู้ในบทนี้สร้างคิวรี "dashboard" สรุปยอดขายของระบบ e-commerce

> **หมายเหตุ:** บทนี้ยังไม่ใช้ `GROUP BY` — เราจะรวมข้อมูล "ทั้งตาราง" หรือ "ทั้งผลลัพธ์ที่กรองด้วย WHERE/FILTER" ให้เหลือค่าเดียวก่อน เพื่อให้เข้าใจพฤติกรรมพื้นฐานของ aggregate function อย่างถ่องแท้ ส่วนการรวมข้อมูลแบบแบ่งกลุ่มย่อย (เช่น สรุปยอดขายแยกตามลูกค้าแต่ละคน) จะเรียนใน **Part 028: GROUP BY และ HAVING**

---

## เตรียมข้อมูล

บทนี้ใช้ชุดข้อมูลร้านค้าออนไลน์ (e-commerce) เดียวกันกับ Part 021–039 ทั้งหมด ประกอบด้วย 9 ตาราง: `categories`, `suppliers`, `products`, `customers`, `employees`, `orders`, `order_items`, `reviews`, `payments`

รันสคริปต์ทั้งหมดนี้ตามลำดับ (ลำดับสำคัญเพราะมี foreign key อ้างอิงกัน):

```sql
-- ลบตารางเก่า (ถ้ามี) เพื่อเริ่มต้นใหม่
DROP TABLE IF EXISTS payments, reviews, order_items, orders, employees,
    customers, products, suppliers, categories CASCADE;

-- ============================================
-- โครงสร้างตาราง
-- ============================================

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
    unit_price      NUMERIC(10,2) NOT NULL CHECK (unit_price >= 0),
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

CREATE TABLE employees (
    employee_id   SERIAL PRIMARY KEY,
    first_name    VARCHAR(60) NOT NULL,
    last_name     VARCHAR(60) NOT NULL,
    hire_date     DATE NOT NULL,
    manager_id    INTEGER REFERENCES employees(employee_id),
    department    VARCHAR(60)
);

CREATE TABLE orders (
    order_id      SERIAL PRIMARY KEY,
    customer_id   INTEGER REFERENCES customers(customer_id),
    employee_id   INTEGER REFERENCES employees(employee_id),
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

CREATE TABLE reviews (
    review_id     SERIAL PRIMARY KEY,
    product_id    INTEGER REFERENCES products(product_id),
    customer_id   INTEGER REFERENCES customers(customer_id),
    rating        INTEGER NOT NULL CHECK (rating BETWEEN 1 AND 5),
    review_text   TEXT,
    review_date   DATE NOT NULL DEFAULT CURRENT_DATE
);

CREATE TABLE payments (
    payment_id      SERIAL PRIMARY KEY,
    order_id        INTEGER REFERENCES orders(order_id),
    payment_date    TIMESTAMPTZ NOT NULL DEFAULT now(),
    amount          NUMERIC(10,2) NOT NULL,
    payment_method  VARCHAR(30)
);

-- ============================================
-- ข้อมูลตัวอย่าง (seed data)
-- ============================================

-- categories (10 แถว)
INSERT INTO categories (category_name, parent_category_id) VALUES
('Electronics', NULL),
('Computers', 1),
('Smartphones', 1),
('Accessories', 1),
('Home & Kitchen', NULL),
('Furniture', 5),
('Books', NULL),
('Toys', NULL),
('Sports & Outdoor', NULL),
('Fashion', NULL);

-- suppliers (10 แถว)
INSERT INTO suppliers (supplier_name, country) VALUES
('Siam Tech Supply', 'Thailand'),
('Shenzhen Electronics Co.', 'China'),
('Global Gadgets Inc.', 'USA'),
('Tokyo Denki', 'Japan'),
('Berlin Hardware GmbH', 'Germany'),
('Seoul Digital', 'South Korea'),
('Saigon Trading', 'Vietnam'),
('Singapore Distributors Pte', 'Singapore'),
('Taipei Components', 'Taiwan'),
('Mumbai Exports', 'India');

-- products (20 แถว)
INSERT INTO products (product_name, category_id, supplier_id, unit_price, stock_quantity, is_active) VALUES
('Wireless Mouse', 4, 2, 259.00, 150, true),
('Mechanical Keyboard', 4, 2, 1490.00, 80, true),
('27" 4K Monitor', 2, 4, 8990.00, 25, true),
('Laptop Stand', 4, 9, 590.00, 200, true),
('Smartphone X12', 3, 6, 15990.00, 40, true),
('Smartphone Lite', 3, 6, 6990.00, 60, true),
('Bluetooth Earbuds', 4, 2, 1290.00, 120, true),
('Gaming Laptop 15"', 2, 4, 32900.00, 15, true),
('Ultrabook 14"', 2, 4, 24900.00, 20, true),
('USB-C Hub', 4, 9, 690.00, 300, true),
('Office Chair', 6, 10, 3590.00, 35, true),
('Standing Desk', 6, 10, 6990.00, 18, true),
('Coffee Maker', 5, 5, 2190.00, 45, true),
('Air Fryer', 5, 5, 2590.00, 50, true),
('Novel: Time and Tide', 7, 1, 320.00, 100, true),
('Cookbook: Thai Kitchen', 7, 1, 450.00, 80, true),
('Yoga Mat', 9, 7, 590.00, 90, true),
('Running Shoes', 9, 7, 2290.00, 70, true),
('Kids Building Blocks', 8, 8, 890.00, 60, true),
('Denim Jacket', 10, 3, 1590.00, 40, false);

-- customers (15 แถว)
INSERT INTO customers (first_name, last_name, email, country, signup_date) VALUES
('Somchai', 'Jaidee', 'somchai.j@example.com', 'Thailand', '2023-01-15'),
('Suda', 'Meesuk', 'suda.m@example.com', 'Thailand', '2023-02-20'),
('Anan', 'Wong', 'anan.w@example.com', 'Thailand', '2023-03-05'),
('Nattaya', 'Srisawat', 'nattaya.s@example.com', 'Thailand', '2023-03-22'),
('Pichai', 'Boonmee', 'pichai.b@example.com', 'Thailand', '2023-04-10'),
('Kanya', 'Thongdee', 'kanya.t@example.com', 'Thailand', '2023-05-01'),
('Wichai', 'Saetang', 'wichai.s@example.com', 'Thailand', '2023-05-18'),
('Malee', 'Phromma', 'malee.p@example.com', 'Thailand', '2023-06-09'),
('John', 'Smith', 'john.smith@example.com', 'USA', '2023-06-25'),
('Emily', 'Chen', 'emily.chen@example.com', 'Singapore', '2023-07-14'),
('Yuki', 'Tanaka', 'yuki.tanaka@example.com', 'Japan', '2023-08-02'),
('Rattana', 'Chaisuk', 'rattana.c@example.com', 'Thailand', '2023-08-20'),
('Somsak', 'Intharak', 'somsak.i@example.com', 'Thailand', '2023-09-11'),
('Piyawan', 'Kulsri', 'piyawan.k@example.com', 'Thailand', '2023-10-01'),
('David', 'Lee', 'david.lee@example.com', 'South Korea', '2023-10-19');

-- employees (10 แถว, มี manager_id อ้างอิงตัวเอง)
INSERT INTO employees (first_name, last_name, hire_date, manager_id, department) VALUES
('Prasert', 'Wattana', '2020-01-10', NULL, 'Management'),
('Siriporn', 'Chaiyo', '2020-03-15', 1, 'Sales'),
('Thanakorn', 'Suksai', '2020-06-01', 1, 'Sales'),
('Napat', 'Rungrueang', '2021-01-20', 2, 'Sales'),
('Orawan', 'Petch', '2021-04-11', 2, 'Sales'),
('Kittipong', 'Sangthong', '2021-07-30', 1, 'Support'),
('Benjawan', 'Sirikul', '2022-02-14', 6, 'Support'),
('Chatchai', 'Amnuay', '2022-05-19', 1, 'Warehouse'),
('Sasithorn', 'Boonchu', '2022-09-01', 8, 'Warehouse'),
('Wirat', 'Kongkeaw', '2023-01-16', 6, 'Support');

-- orders (20 แถว)
INSERT INTO orders (customer_id, employee_id, order_date, status, ship_country) VALUES
(1,  2, '2024-01-05 10:20:00+07', 'delivered',  'Thailand'),
(2,  2, '2024-01-08 14:00:00+07', 'delivered',  'Thailand'),
(3,  3, '2024-01-12 09:15:00+07', 'delivered',  'Thailand'),
(1,  4, '2024-01-20 16:40:00+07', 'delivered',  'Thailand'),
(4,  4, '2024-02-02 11:05:00+07', 'cancelled',  'Thailand'),
(5,  5, '2024-02-10 13:30:00+07', 'delivered',  'Thailand'),
(6,  2, '2024-02-15 08:50:00+07', 'delivered',  'Thailand'),
(9,  3, '2024-02-18 19:22:00+07', 'shipped',    'USA'),
(7,  4, '2024-02-25 10:10:00+07', 'delivered',  'Thailand'),
(2,  5, '2024-03-01 15:45:00+07', 'delivered',  'Thailand'),
(10, 2, '2024-03-05 12:00:00+07', 'shipped',    'Singapore'),
(8,  3, '2024-03-09 09:30:00+07', 'delivered',  'Thailand'),
(1,  4, '2024-03-14 17:15:00+07', 'pending',    'Thailand'),
(11, 5, '2024-03-20 21:00:00+07', 'delivered',  'Japan'),
(3,  2, '2024-03-25 10:40:00+07', 'delivered',  'Thailand'),
(12, 3, '2024-04-02 14:20:00+07', 'delivered',  'Thailand'),
(13, 4, '2024-04-08 11:50:00+07', 'cancelled',  'Thailand'),
(14, 5, '2024-04-15 16:05:00+07', 'delivered',  'Thailand'),
(5,  2, '2024-04-22 09:00:00+07', 'processing', 'Thailand'),
(15, 3, '2024-04-28 20:30:00+07', 'delivered',  'South Korea');

-- order_items (32 แถว)
INSERT INTO order_items (order_id, product_id, quantity, unit_price) VALUES
(1,  1,  2,   259.00),
(1,  7,  1,  1290.00),
(2,  2,  1,  1490.00),
(2,  10, 2,   690.00),
(3,  5,  1, 15990.00),
(4,  15, 3,   320.00),
(4,  16, 1,   450.00),
(5,  3,  1,  8990.00),
(6,  11, 1,  3590.00),
(6,  12, 1,  6990.00),
(7,  17, 2,   590.00),
(7,  18, 1,  2290.00),
(8,  8,  1, 32900.00),
(9,  6,  1,  6990.00),
(9,  7,  1,  1290.00),
(10, 1,  3,   259.00),
(10, 4,  2,   590.00),
(11, 9,  1, 24900.00),
(12, 19, 2,   890.00),
(12, 20, 1,  1590.00),
(13, 2,  1,  1490.00),
(14, 5,  1, 15990.00),
(14, 7,  2,  1290.00),
(15, 13, 1,  2190.00),
(15, 14, 1,  2590.00),
(16, 17, 1,   590.00),
(16, 1,  1,   259.00),
(17, 12, 1,  6990.00),
(18, 3,  1,  8990.00),
(19, 6,  1,  6990.00),
(20, 9,  1, 24900.00),
(20, 10, 1,   690.00);

-- reviews (15 แถว)
INSERT INTO reviews (product_id, customer_id, rating, review_text, review_date) VALUES
(1,  1,  5, 'เมาส์ใช้งานลื่นมาก คุ้มราคา', '2024-01-10'),
(1,  2,  4, 'ดีไซน์สวย แต่แบตอยู่ได้ไม่นาน', '2024-01-15'),
(2,  2,  5, 'คีย์บอร์ดพิมพ์สนุกมาก เสียงคลิกชัดเจน', '2024-01-20'),
(3,  3,  5, 'จอสวยคมชัดมาก คุ้มค่าเงิน', '2024-01-25'),
(5,  3,  4, 'สเปคดี แต่ราคาแรงไปหน่อย', '2024-01-28'),
(5,  11, 5, 'กล้องถ่ายรูปสวยมาก', '2024-03-25'),
(7,  1,  3, 'เสียงโอเค แต่การเชื่อมต่อหลุดบ่อย', '2024-01-12'),
(7,  6,  5, 'หูฟังเสียงดีมาก ใส่สบาย', '2024-02-20'),
(8,  9,  5, 'โน้ตบุ๊คแรงมาก เล่นเกมลื่น', '2024-02-22'),
(9,  10, 4, 'บางเบา พกพาสะดวก', '2024-03-08'),
(11, 5,  4, 'เก้าอี้นั่งสบาย ปรับระดับได้ดี', '2024-02-14'),
(12, 5,  5, 'โต๊ะปรับระดับใช้งานดีมาก', '2024-02-16'),
(15, 1,  4, 'หนังสือสนุก อ่านเพลิน', '2024-01-22'),
(17, 6,  5, 'โยคะแมทหนาดี ไม่ลื่น', '2024-02-18'),
(19, 8,  3, 'ของเล่นดี แต่ชิ้นส่วนเล็กไปหน่อย', '2024-03-12');

-- payments (16 แถว — เฉพาะออเดอร์ที่ชำระเงินแล้ว)
INSERT INTO payments (order_id, payment_date, amount, payment_method) VALUES
(1,  '2024-01-05 10:25:00+07',  1808.00, 'promptpay'),
(2,  '2024-01-08 14:05:00+07',  2870.00, 'credit_card'),
(3,  '2024-01-12 09:20:00+07', 15990.00, 'credit_card'),
(4,  '2024-01-20 16:45:00+07',  1410.00, 'promptpay'),
(6,  '2024-02-10 13:35:00+07', 10580.00, 'credit_card'),
(7,  '2024-02-15 08:55:00+07',  3470.00, 'bank_transfer'),
(8,  '2024-02-18 19:30:00+07', 32900.00, 'credit_card'),
(9,  '2024-02-25 10:15:00+07',  8280.00, 'promptpay'),
(10, '2024-03-01 15:50:00+07',  1957.00, 'cod'),
(11, '2024-03-05 12:10:00+07', 24900.00, 'credit_card'),
(12, '2024-03-09 09:35:00+07',  3370.00, 'bank_transfer'),
(14, '2024-03-20 21:10:00+07', 18570.00, 'credit_card'),
(15, '2024-03-25 10:45:00+07',  4780.00, 'promptpay'),
(16, '2024-04-02 14:25:00+07',   849.00, 'cod'),
(18, '2024-04-15 16:10:00+07',  8990.00, 'credit_card'),
(20, '2024-04-28 20:35:00+07', 25590.00, 'credit_card');
```

สังเกตว่า:
- ออเดอร์ที่มีสถานะ `cancelled` (order 5, 17) และ `pending`/`processing` (order 13, 19) **ไม่มี** แถวใน `payments` เพราะยังไม่ได้ชำระเงินจริง — ดังนั้นตาราง `payments` จึงมี 16 แถว ไม่ใช่ 20 แถวเท่าจำนวนออเดอร์
- สินค้า `Denim Jacket` (product_id 20) มี `is_active = false` เพราะเลิกขายแล้ว แต่ยังปรากฏในประวัติการสั่งซื้อเก่า (order 12) ได้ตามปกติ
- `employees.manager_id` ของพนักงานคนแรก (Prasert Wattana) เป็น `NULL` เพราะเป็นผู้บริหารสูงสุด ไม่มีหัวหน้า — เราจะใช้ตรงนี้สาธิตความแตกต่างระหว่าง `COUNT(*)` กับ `COUNT(column)` ใน Step 262

ตรวจสอบว่านำเข้าข้อมูลครบถ้วน:

```sql
SELECT 'categories' AS table_name, COUNT(*) FROM categories
UNION ALL SELECT 'suppliers',   COUNT(*) FROM suppliers
UNION ALL SELECT 'products',    COUNT(*) FROM products
UNION ALL SELECT 'customers',   COUNT(*) FROM customers
UNION ALL SELECT 'employees',   COUNT(*) FROM employees
UNION ALL SELECT 'orders',      COUNT(*) FROM orders
UNION ALL SELECT 'order_items', COUNT(*) FROM order_items
UNION ALL SELECT 'reviews',     COUNT(*) FROM reviews
UNION ALL SELECT 'payments',    COUNT(*) FROM payments;
```

```
 table_name  | count
-------------+-------
 categories  |    10
 suppliers   |    10
 products    |    20
 customers   |    15
 employees   |    10
 orders      |    20
 order_items |    32
 reviews     |    15
 payments    |    16
(9 rows)
```

---

## Step 261: Aggregate function คืออะไร

**Aggregate function** (ฟังก์ชันรวมกลุ่ม) คือฟังก์ชันที่รับ**ค่าจากหลายแถว**เป็น input แล้วคำนวณออกมาเป็น**ค่าเดียว** (single value) ต่างจากฟังก์ชันทั่วไป (scalar function เช่น `UPPER()`, `ROUND()`, `NOW()`) ที่ทำงานทีละแถว แล้วคืนค่าหนึ่งค่าต่อหนึ่งแถวที่ input เข้าไป

ลองเปรียบเทียบให้เห็นภาพ:

```sql
-- Scalar function: 1 แถว input → 1 แถว output (ทำงานทีละแถว)
SELECT product_name, ROUND(unit_price) AS rounded_price
FROM products
LIMIT 3;
```

```
   product_name       | rounded_price
-----------------------+---------------
 Wireless Mouse        |           259
 Mechanical Keyboard   |          1490
 27" 4K Monitor        |          8990
(3 rows)
```

```sql
-- Aggregate function: N แถว input → 1 แถว output (สรุปทั้งหมดเป็นค่าเดียว)
SELECT COUNT(*) AS total_products
FROM products;
```

```
 total_products
-----------------
              20
(1 row)
```

ไม่ว่าตาราง `products` จะมีกี่แถว คิวรีที่ใช้ aggregate function ล้วน ๆ (ไม่มี `GROUP BY`) จะคืนผลลัพธ์ออกมา**เพียง 1 แถวเสมอ**

### Aggregate function ที่ใช้บ่อยที่สุด 5 ตัว

| ฟังก์ชัน | ความหมาย | ใช้กับชนิดข้อมูล |
|---|---|---|
| `COUNT()` | นับจำนวนแถว | ทุกชนิด |
| `SUM()` | รวมค่า | ตัวเลข (numeric, integer, ฯลฯ) |
| `AVG()` | ค่าเฉลี่ย | ตัวเลข |
| `MIN()` | ค่าน้อยที่สุด | ทุกชนิดที่เปรียบเทียบได้ |
| `MAX()` | ค่ามากที่สุด | ทุกชนิดที่เปรียบเทียบได้ |

นอกจาก 5 ตัวนี้ PostgreSQL ยังมี aggregate function อื่นอีกมาก เช่น `STRING_AGG`, `ARRAY_AGG`, `STDDEV`, `VARIANCE`, `PERCENTILE_CONT`, `BOOL_AND`, `BOOL_OR` ซึ่งเราจะได้เรียนในบทนี้เช่นกัน (Step 268-269)

### aggregate function ทำงานร่วมกับ WHERE ได้ตามปกติ

`WHERE` จะกรองแถวก่อนที่ aggregate function จะรับไปคำนวณ:

```sql
SELECT COUNT(*) AS active_products
FROM products
WHERE is_active = true;
```

```
 active_products
------------------
               19
(1 row)
```

```sql
-- หาช่วงเวลาที่มีออเดอร์แรกและออเดอร์ล่าสุด
SELECT MIN(order_date) AS first_order, MAX(order_date) AS last_order
FROM orders;
```

```
       first_order        |        last_order
---------------------------+---------------------------
 2024-01-05 10:20:00+07    | 2024-04-28 20:30:00+07
(1 row)
```

### กฎสำคัญ: ห้ามผสมคอลัมน์ธรรมดากับ aggregate function โดยไม่มี GROUP BY

```sql
-- ผิด! PostgreSQL จะ error
SELECT product_name, COUNT(*) FROM products;
```

```
ERROR:  column "products.product_name" must appear in the GROUP BY clause
        or be used in an aggregate function
```

เหตุผลคือ `COUNT(*)` สรุปทั้งตารางให้เหลือ 1 แถว แต่ `product_name` มีค่าต่างกันถึง 20 ค่า PostgreSQL จึงไม่รู้ว่าจะเอาค่าไหนมาแสดงคู่กับผลรวมนั้น กฎนี้จะคลายลงเมื่อเราใช้ `GROUP BY` ซึ่งเป็นเนื้อหา **Part 028** — บทนี้เราจะ `SELECT` เฉพาะ aggregate function (และค่าคงที่/subquery) เท่านั้น เพื่อไม่ชนกฎนี้

> **สรุป Step 261:** aggregate function ย่อรวมหลายแถวให้เหลือค่าเดียว ใช้ร่วมกับ `WHERE` ได้ปกติ แต่ห้ามปนกับคอลัมน์ที่ไม่ได้ aggregate โดยไม่มี `GROUP BY`

---

## Step 262: COUNT(*) vs COUNT(column) vs COUNT(DISTINCT column)

`COUNT` มี 3 รูปแบบที่ให้ผลลัพธ์ต่างกันเมื่อข้อมูลมีค่า `NULL` หรือมีค่าซ้ำ:

| รูปแบบ | นับอะไร |
|---|---|
| `COUNT(*)` | นับ**จำนวนแถวทั้งหมด** ไม่สนใจว่าคอลัมน์ใดจะเป็น `NULL` หรือไม่ |
| `COUNT(column)` | นับเฉพาะแถวที่คอลัมน์นั้น**ไม่ใช่ `NULL`** |
| `COUNT(DISTINCT column)` | นับจำนวน**ค่าที่ไม่ซ้ำกัน** ของคอลัมน์นั้น (และไม่นับ `NULL` ด้วย) |

### ตัวอย่างที่เห็นผลชัดเจน: employees.manager_id

ตาราง `employees` มี 10 แถว โดยพนักงานคนแรก (Prasert Wattana ซึ่งเป็นผู้บริหารสูงสุด) มี `manager_id = NULL` เพราะไม่มีใครเป็นหัวหน้า ส่วนที่เหลืออีก 9 คนมีหัวหน้า และหัวหน้าที่ปรากฏมีอยู่จริงเพียง 4 คน (employee_id 1, 2, 6, 8)

```sql
SELECT
    COUNT(*)                     AS total_employees,
    COUNT(manager_id)            AS employees_with_manager,
    COUNT(DISTINCT manager_id)   AS distinct_managers
FROM employees;
```

```
 total_employees | employees_with_manager | distinct_managers
------------------+-------------------------+--------------------
              10  |                       9 |                  4
(1 row)
```

อธิบายทีละคอลัมน์:
- `COUNT(*)` = 10 → นับทุกแถวในตาราง ไม่สนใจ `manager_id` เลย
- `COUNT(manager_id)` = 9 → นับเฉพาะแถวที่ `manager_id IS NOT NULL` (ตัด Prasert ที่เป็น `NULL` ออกไป 1 คน)
- `COUNT(DISTINCT manager_id)` = 4 → ในบรรดา 9 ค่าที่ไม่ใช่ `NULL` นั้น มีค่าไม่ซ้ำกันเพียง 4 ค่า คือ 1, 2, 6, 8 (เพราะหลายคนมีหัวหน้าคนเดียวกัน)

### ตัวอย่างที่สอง: หมวดหมู่สินค้าที่ถูกใช้งานจริง

ตาราง `categories` มีทั้งหมด 10 หมวดหมู่ แต่ตาราง `products` (20 แถว) อาจไม่ได้ใช้ทุกหมวดหมู่:

```sql
SELECT
    COUNT(*)                     AS total_products,
    COUNT(category_id)           AS products_with_category,
    COUNT(DISTINCT category_id)  AS distinct_categories_used
FROM products;
```

```
 total_products | products_with_category | distinct_categories_used
-----------------+-------------------------+---------------------------
              20 |                      20 |                         9
(1 row)
```

ในกรณีนี้ `category_id` ไม่มีค่า `NULL` เลย (ทุกสินค้าต้องมีหมวดหมู่) ดังนั้น `COUNT(*)` กับ `COUNT(category_id)` จึงเท่ากันคือ 20 แต่ `COUNT(DISTINCT category_id)` = 9 เพราะสินค้าทั้ง 20 ชิ้นใช้หมวดหมู่ย่อยซ้ำกันอยู่ (หมวด "Electronics" หลักไม่มีสินค้าผูกตรง ๆ เลยสักชิ้น มีแต่หมวดย่อยของมัน)

### COUNT(*) นับแถวว่าง (empty result) ได้เป็น 0 เสมอ — ต่างจาก SUM/AVG

```sql
SELECT COUNT(*) AS suspiciously_expensive_products
FROM products
WHERE unit_price > 100000;
```

```
 suspiciously_expensive_products
-----------------------------------
                                 0
(1 row)
```

`COUNT` ไม่เคยคืนค่า `NULL` เพราะมันนับ "จำนวน" เสมอ แม้จะไม่มีแถวใดตรงเงื่อนไขเลย ก็ยังนับได้ว่าเท่ากับ 0 — พฤติกรรมนี้**แตกต่างจาก `SUM`/`AVG`/`MIN`/`MAX` ที่จะคืนค่า `NULL` เมื่อไม่มีแถว** (รายละเอียดใน Step 263)

### COUNT(DISTINCT ...) กับหลายคอลัมน์พร้อมกัน

PostgreSQL อนุญาตให้ใส่หลายคอลัมน์ใน `COUNT(DISTINCT ...)` ได้ โดยจะนับ**คู่ค่าที่ไม่ซ้ำกัน** (row-wise distinct):

```sql
-- นับคู่ (customer_id, employee_id) ที่ไม่ซ้ำกันในตาราง orders
-- หมายถึง: มีกี่ "คู่ลูกค้า-พนักงาน" ที่เคยทำธุรกรรมร่วมกัน
SELECT COUNT(DISTINCT (customer_id, employee_id)) AS unique_customer_employee_pairs
FROM orders;
```

```
 unique_customer_employee_pairs
----------------------------------
                              19
(1 row)
```

(ออเดอร์มี 20 แถว แต่มีคู่ลูกค้า-พนักงานซ้ำกัน 1 คู่ คือ customer_id=1 กับ employee_id=4 ที่ปรากฏทั้งใน order 4 และ order 13)

> **สรุป Step 262:** ใช้ `COUNT(*)` เมื่อต้องการนับแถวทั้งหมด ใช้ `COUNT(column)` เมื่อต้องการนับเฉพาะที่มีค่า (ไม่ใช่ `NULL`) และใช้ `COUNT(DISTINCT column)` เมื่อต้องการนับค่าที่ไม่ซ้ำกัน

---

## Step 263: SUM — รวมค่า, การจัดการ NULL, SUM กับ FILTER clause

`SUM()` รวมค่าตัวเลขจากทุกแถวที่ผ่านการกรอง โดยจะ**ข้าม (ignore) ค่า `NULL` โดยอัตโนมัติ** เสมือนไม่มีแถวนั้นอยู่ในการคำนวณเลย

### ตัวอย่างพื้นฐาน

```sql
SELECT SUM(amount) AS total_revenue
FROM payments;
```

```
 total_revenue
----------------
      166314.00
(1 row)
```

```sql
SELECT SUM(quantity) AS total_units_sold
FROM order_items;
```

```
 total_units_sold
--------------------
                 42
```
```
(1 row)
```

### กับดักสำคัญ: SUM ของแถวว่าง (หรือ SUM ที่ทุกค่าเป็น NULL) จะได้ผลลัพธ์เป็น NULL ไม่ใช่ 0

```sql
SELECT SUM(amount) AS revenue_from_nonexistent_order
FROM payments
WHERE order_id = 999;
```

```
 revenue_from_nonexistent_order
-----------------------------------
                            [NULL]
(1 row)
```

นี่เป็นจุดที่ผู้เริ่มต้นพลาดบ่อยมาก เพราะสัญชาตญาณมักคิดว่า "ไม่มีอะไรให้บวก ผลรวมต้องเป็น 0" แต่ทางคณิตศาสตร์เชิงสัมพันธ์ (relational) แล้ว "ผลรวมของเซตว่าง" ถือว่า**ไม่นิยาม (undefined)** จึงคืนค่าเป็น `NULL` — ถ้าต้องการให้เป็น 0 แทน ต้องครอบด้วย `COALESCE`:

```sql
SELECT COALESCE(SUM(amount), 0) AS revenue_from_nonexistent_order
FROM payments
WHERE order_id = 999;
```

```
 revenue_from_nonexistent_order
-----------------------------------
                            0.00
(1 row)
```

**เปรียบเทียบให้เห็นชัด:**

```sql
SELECT
    COUNT(*)  AS row_count,      -- นับแถว → ได้ 0 เสมอถ้าไม่มีแถว
    SUM(amount) AS sum_amount    -- รวมค่า → ได้ NULL ถ้าไม่มีแถว
FROM payments
WHERE order_id = 999;
```

```
 row_count | sum_amount
------------+-------------
          0 |     [NULL]
(1 row)
```

### SUM ร่วมกับนิพจน์คำนวณ (expression)

`SUM` รับนิพจน์ใด ๆ ที่คำนวณเป็นตัวเลขได้ ไม่จำเป็นต้องเป็นแค่ชื่อคอลัมน์เดี่ยว ๆ:

```sql
-- คำนวณยอดขายรวมทั้งหมดจาก order_items โดยคูณ quantity * unit_price ก่อนรวม
SELECT SUM(quantity * unit_price) AS gross_revenue_all_orders
FROM order_items;
```

```
 gross_revenue_all_orders
-----------------------------
                 190774.00
(1 row)
```

สังเกตว่าตัวเลขนี้ (190,774.00) **มากกว่า** `SUM(amount) FROM payments` (166,314.00) เพราะ `order_items` รวมออเดอร์ที่ถูกยกเลิก (`cancelled`) และยังไม่ชำระเงิน (`pending`, `processing`) ด้วย ในขณะที่ `payments` มีเฉพาะออเดอร์ที่จ่ายเงินสำเร็จแล้ว — เป็นตัวอย่างที่ดีว่าทำไมต้องระวังว่าเรากำลัง `SUM` จากตารางไหน และควรมี `WHERE` กรองสถานะให้ถูกต้องเสมอ

### SUM กับ FILTER clause — รวมแบบมีเงื่อนไข "หลายชุด" ในคิวรีเดียว

`FILTER (WHERE ...)` คือ syntax พิเศษที่ใส่ต่อท้าย aggregate function เพื่อกำหนดว่า "ให้ aggregate ตัวนี้คำนวณเฉพาะแถวที่ตรงเงื่อนไขนี้เท่านั้น" — ต่างจาก `WHERE` ทั่วไปของคิวรีตรงที่ `WHERE` กรองทุกแถวก่อนส่งเข้า aggregate function **ทุกตัว** แต่ `FILTER` กรองเฉพาะ aggregate function ตัวนั้นตัวเดียว ทำให้เขียนอากรีเกตหลายเงื่อนไขในคิวรีเดียวกันได้ (รายละเอียดเชิงลึกอยู่ใน Step 266):

```sql
SELECT
    SUM(amount) FILTER (WHERE payment_method = 'credit_card')   AS credit_card_total,
    SUM(amount) FILTER (WHERE payment_method = 'promptpay')     AS promptpay_total,
    SUM(amount) FILTER (WHERE payment_method = 'bank_transfer') AS bank_transfer_total,
    SUM(amount) FILTER (WHERE payment_method = 'cod')           AS cod_total
FROM payments;
```

```
 credit_card_total | promptpay_total | bank_transfer_total | cod_total
---------------------+-------------------+------------------------+-----------
          140390.00 |          16278.00 |               6840.00 |   2806.00
(1 row)
```

ตรวจทานผลรวมย่อยทั้ง 4 ช่องทางการชำระเงิน: 140,390.00 + 16,278.00 + 6,840.00 + 2,806.00 = 166,314.00 ตรงกับ `SUM(amount)` รวมทั้งหมดพอดี

> **สรุป Step 263:** `SUM` รวมค่าตัวเลขและข้าม `NULL` โดยอัตโนมัติ แต่ถ้าไม่มีแถวให้รวมเลยจะได้ `NULL` (ไม่ใช่ 0) ใช้ `COALESCE(SUM(...), 0)` เมื่อต้องการค่าเริ่มต้นเป็น 0 และใช้ `FILTER (WHERE ...)` เมื่อต้องการรวมแบบมีเงื่อนไขหลายชุดในคิวรีเดียว

---

## Step 264: AVG — ค่าเฉลี่ย, ผลลัพธ์เป็น numeric แม้ input เป็น integer

`AVG()` คำนวณค่าเฉลี่ยเลขคณิต (arithmetic mean) โดย**ข้าม `NULL` ออกจากทั้งตัวเศษ (sum) และตัวส่วน (count)** — พูดง่าย ๆ คือ `AVG(x)` เทียบเท่ากับ `SUM(x) / COUNT(x)` ไม่ใช่ `SUM(x) / COUNT(*)`

### ตัวอย่างพื้นฐาน

```sql
SELECT AVG(unit_price) AS avg_product_price
FROM products;
```

```
 avg_product_price
---------------------
          5778.950000000000
(1 row)
```

### จุดสำคัญ: AVG(integer_column) ไม่ตัดทศนิยมทิ้งเหมือนหาร integer ธรรมดา

คอลัมน์ `order_items.quantity` เป็น `INTEGER` แต่ `AVG(quantity)` จะคืนผลลัพธ์เป็น `numeric` ที่มีทศนิยม ไม่ใช่ integer division ที่ปัดเศษทิ้ง:

```sql
SELECT AVG(quantity) AS avg_quantity_per_line
FROM order_items;
```

```
 avg_quantity_per_line
-------------------------
      1.31250000000000000
(1 row)
```

เปรียบเทียบกับการหารแบบ integer ธรรมดาซึ่ง**ผิด**ถ้าตั้งใจจะหาค่าเฉลี่ย:

```sql
-- ระวัง! นี่คือ integer division ไม่ใช่ AVG — ผลลัพธ์จะถูกปัดเศษทิ้งทันที
SELECT SUM(quantity) / COUNT(*) AS wrong_integer_division,
       AVG(quantity)            AS correct_average
FROM order_items;
```

```
 wrong_integer_division | correct_average
--------------------------+-------------------
                        1 | 1.31250000000000000
(1 row)
```

`SUM(quantity)` = 42 (เป็น `bigint`) หารด้วย `COUNT(*)` = 32 (เป็น `bigint`) ด้วยตัวดำเนินการหารปกติ (`/`) ระหว่างจำนวนเต็มสองตัว PostgreSQL จะทำ integer division ทิ้งเศษ ได้ 1 ซึ่งผิดจากค่าเฉลี่ยจริง (1.3125) — ถ้าจะหารเองต้อง cast เป็น `numeric` ก่อน เช่น `SUM(quantity)::numeric / COUNT(*)` แต่ทางที่ดีที่สุดคือใช้ `AVG()` ไปเลยเพราะ PostgreSQL จัดการเรื่อง type casting ให้ถูกต้องอยู่แล้ว

### ควบคุมจำนวนทศนิยมด้วย ROUND

ผลลัพธ์ของ `AVG` มักมีทศนิยมยาวเกินความจำเป็น ใช้ `ROUND()` ครอบเพื่อจัดรูปแบบให้อ่านง่าย:

```sql
SELECT ROUND(AVG(unit_price), 2) AS avg_price_2dp
FROM products
WHERE is_active = true;
```

```
 avg_price_2dp
-----------------
        5999.42
(1 row)
```

(หมายเหตุ: ค่าเฉลี่ยนี้คำนวณจากสินค้าที่ `is_active = true` เท่านั้น คือ 19 ชิ้นจากทั้งหมด 20 ชิ้น เพราะ Denim Jacket ถูกเลิกขายไปแล้ว)

### AVG กับแถวว่างก็คืน NULL เช่นเดียวกับ SUM

```sql
SELECT AVG(rating) AS avg_rating_for_nonexistent_product
FROM reviews
WHERE product_id = 999;
```

```
 avg_rating_for_nonexistent_product
---------------------------------------
                               [NULL]
(1 row)
```

> **สรุป Step 264:** `AVG` = `SUM(ที่ไม่นับ NULL) / COUNT(ที่ไม่นับ NULL)` เสมอ ผลลัพธ์เป็น `numeric` ที่มีทศนิยมแม่นยำแม้ input จะเป็น integer และเช่นเดียวกับ `SUM`, `AVG` ของเซตว่างคือ `NULL`

---

## Step 265: MIN/MAX — ใช้ได้กับตัวเลข ข้อความ วันที่ ทุกประเภทที่เปรียบเทียบได้

`MIN()` และ `MAX()` แตกต่างจาก `SUM`/`AVG` ตรงที่**ไม่ได้จำกัดเฉพาะข้อมูลตัวเลข** — ใช้ได้กับ**ทุกชนิดข้อมูลที่มี operator เปรียบเทียบ (`<`, `>`) นิยามไว้** ซึ่งครอบคลุมเกือบทุกชนิดข้อมูลพื้นฐานใน PostgreSQL

### MIN/MAX กับตัวเลข

```sql
SELECT
    MIN(unit_price) AS cheapest_price,
    MAX(unit_price) AS most_expensive_price
FROM products;
```

```
 cheapest_price | most_expensive_price
------------------+------------------------
          259.00 |              32900.00
(1 row)
```

### MIN/MAX กับวันที่และเวลา (date / timestamp)

```sql
SELECT
    MIN(order_date) AS earliest_order,
    MAX(order_date) AS latest_order,
    MIN(signup_date) AS earliest_signup
FROM orders, customers;  -- ตัวอย่างนี้จงใจใช้ cross join เพื่อความกระชับ ปกติควรแยกคิวรี
```

โดยทั่วไปควรเขียนแยกกันเพื่อความชัดเจนและถูกต้อง (ตัวอย่างข้างต้นมี cross join ซึ่งไม่จำเป็นและสิ้นเปลือง):

```sql
SELECT MIN(order_date) AS earliest_order, MAX(order_date) AS latest_order
FROM orders;

SELECT MIN(signup_date) AS earliest_signup, MAX(signup_date) AS latest_signup
FROM customers;
```

```
      earliest_order       |        latest_order
-----------------------------+-----------------------------
 2024-01-05 10:20:00+07     | 2024-04-28 20:30:00+07
(1 row)

 earliest_signup | latest_signup
-------------------+------------------
 2023-01-15       | 2023-10-19
(1 row)
```

### MIN/MAX กับข้อความ (text) — เรียงตาม collation

เมื่อใช้กับข้อความ `MIN`/`MAX` จะเทียบตามลำดับ collation ของฐานข้อมูล (โดยทั่วไปคือเรียงตามตัวอักษร คล้าย dictionary order):

```sql
SELECT
    MIN(product_name) AS first_alphabetically,
    MAX(product_name) AS last_alphabetically
FROM products;
```

```
    first_alphabetically   | last_alphabetically
------------------------------+-----------------------
 27" 4K Monitor              | Yoga Mat
(1 row)
```

`MIN(product_name)` ได้ `27" 4K Monitor` เพราะตัวเลข `'2'` มีลำดับ code point ต่ำกว่าตัวอักษรภาษาอังกฤษทุกตัวใน collation แบบมาตรฐาน (`C`/`en_US`) ส่วน `MAX` ได้ `Yoga Mat` เพราะตัว `Y` อยู่ท้าย ๆ ของตัวอักษรภาษาอังกฤษ และไม่มีชื่อสินค้าใดขึ้นต้นด้วยตัวอักษรที่มาหลัง `Y` เลย

> **ข้อควรระวัง:** ผลลัพธ์ของ `MIN`/`MAX` บนข้อความจะขึ้นกับ **collation** ที่ตั้งไว้ตอนสร้างฐานข้อมูล/คอลัมน์ ถ้าฐานข้อมูลตั้ง collation แบบ locale-aware (เช่น `th_TH`) ลำดับการเรียงตัวอักษรไทยหรือสัญลักษณ์พิเศษอาจแตกต่างจาก `C` collation ได้

### MIN/MAX ใช้ไม่ได้กับ boolean และ JSON โดยตรง

```sql
-- ผิด! boolean ไม่มี MIN/MAX aggregate ในตัว
SELECT MIN(is_active) FROM products;
```

```
ERROR:  function min(boolean) does not exist
```

สำหรับข้อมูล `BOOLEAN` ต้องใช้ `BOOL_AND()` (จริงทั้งหมดหรือไม่) หรือ `BOOL_OR()` (จริงอย่างน้อยหนึ่งค่าหรือไม่) แทน:

```sql
SELECT
    BOOL_AND(is_active) AS all_products_active,
    BOOL_OR(is_active)  AS any_product_active
FROM products;
```

```
 all_products_active | any_product_active
-----------------------+-----------------------
 f                    | t
(1 row)
```

`BOOL_AND` ได้ `f` (false) เพราะไม่ใช่สินค้าทุกตัวที่ active (Denim Jacket เป็น false) ส่วน `BOOL_OR` ได้ `t` (true) เพราะมีสินค้าอย่างน้อยหนึ่งตัวที่ active

### MIN/MAX ใช้หาแถวที่เกี่ยวข้องได้ด้วย subquery

`MIN`/`MAX` เองคืนแค่ "ค่า" ไม่ใช่ "ทั้งแถว" ถ้าต้องการรู้ว่าสินค้าไหนราคาแพงที่สุด ต้องใช้ subquery หรือ `ORDER BY ... LIMIT 1` (ซึ่งเป็นเทคนิคที่มักมีประสิทธิภาพดีกว่าเมื่อมี index):

```sql
SELECT product_name, unit_price
FROM products
WHERE unit_price = (SELECT MAX(unit_price) FROM products);
```

```
    product_name     | unit_price
-----------------------+-------------
 Gaming Laptop 15"    |   32900.00
(1 row)
```

> **สรุป Step 265:** `MIN`/`MAX` ใช้ได้กับตัวเลข วันที่/เวลา ข้อความ (ตาม collation) และชนิดข้อมูลอื่น ๆ ที่เปรียบเทียบได้ แต่ใช้กับ `BOOLEAN` ไม่ได้โดยตรง (ต้องใช้ `BOOL_AND`/`BOOL_OR` แทน) และถ้าต้องการทั้งแถวที่มีค่า MIN/MAX ต้องใช้ subquery ร่วมด้วย

---

## Step 266: Aggregate function ร่วมกับ FILTER (WHERE ...)

`FILTER (WHERE condition)` เป็น syntax ที่ต่อท้าย aggregate function เพื่อจำกัดว่า aggregate function ตัวนั้นจะคำนวณจากแถวใดบ้าง — เป็นฟีเจอร์มาตรฐาน SQL (SQL:2003) ที่ PostgreSQL รองรับมาตั้งแต่เวอร์ชัน 9.4

รูปแบบ:

```sql
aggregate_function(expression) FILTER (WHERE condition)
```

### ทำไมต้องมี FILTER ทั้งที่มี WHERE อยู่แล้ว?

`WHERE` ของคิวรีกรอง**ทุกแถวก่อนส่งเข้า aggregate function ทุกตัวในคิวรีเดียวกัน** ถ้าต้องการนับ/รวมด้วยเงื่อนไขที่ต่างกันหลายชุดในคิวรีเดียว (เช่น นับออเดอร์แยกตามสถานะ) `WHERE` เพียงอย่างเดียวทำไม่ได้ ต้องพึ่ง `FILTER` (หรือเทคนิคเก่าคือ `CASE WHEN` ซึ่งจะเปรียบเทียบให้ดูด้านล่าง)

### ตัวอย่าง: สรุปจำนวนออเดอร์แยกตามสถานะในคิวรีเดียว

```sql
SELECT
    COUNT(*) AS total_orders,
    COUNT(*) FILTER (WHERE status = 'delivered')  AS delivered_orders,
    COUNT(*) FILTER (WHERE status = 'shipped')    AS shipped_orders,
    COUNT(*) FILTER (WHERE status = 'pending')    AS pending_orders,
    COUNT(*) FILTER (WHERE status = 'processing') AS processing_orders,
    COUNT(*) FILTER (WHERE status = 'cancelled')  AS cancelled_orders
FROM orders;
```

```
 total_orders | delivered_orders | shipped_orders | pending_orders | processing_orders | cancelled_orders
----------------+---------------------+------------------+------------------+-----------------------+---------------------
            20 |                  14 |               2 |               1 |                    1 |                  2
(1 row)
```

ตรวจทาน: 14 + 2 + 1 + 1 + 2 = 20 ตรงกับ `total_orders` พอดี — คิวรีนี้อ่าน `orders` เพียงครั้งเดียว (single table scan) แล้วคำนวณทั้ง 6 ค่าไปพร้อมกัน แทนที่จะต้องรันคิวรีแยก 5 รอบ

### เปรียบเทียบกับเทคนิคเก่า: CASE WHEN ภายใน SUM

ก่อนที่ PostgreSQL จะรองรับ `FILTER` (และในฐานข้อมูลบางตัวที่ยังไม่รองรับ) นักพัฒนามักใช้ `CASE WHEN ... THEN 1 ELSE 0 END` ภายใน `SUM` แทน ซึ่งให้ผลลัพธ์เหมือนกันทุกประการแต่อ่านยากกว่า:

```sql
SELECT
    SUM(CASE WHEN status = 'delivered' THEN 1 ELSE 0 END) AS delivered_orders_old_style,
    COUNT(*) FILTER (WHERE status = 'delivered')          AS delivered_orders_filter_style
FROM orders;
```

```
 delivered_orders_old_style | delivered_orders_filter_style
-------------------------------+----------------------------------
                            14 |                              14
(1 row)
```

`FILTER` อ่านง่ายกว่า สื่อความหมายชัดเจนกว่า และในหลายกรณี query planner ของ PostgreSQL ก็ optimize ได้ดีกว่าเวอร์ชัน `CASE WHEN` ด้วย จึงแนะนำให้ใช้ `FILTER` เป็นค่าเริ่มต้นเมื่อเขียนโค้ดใหม่

### FILTER ใช้ได้กับ aggregate function ทุกตัว ไม่ใช่แค่ COUNT

```sql
SELECT
    SUM(amount) FILTER (WHERE payment_method = 'credit_card') AS credit_card_revenue,
    AVG(amount) FILTER (WHERE payment_method = 'credit_card') AS avg_credit_card_payment,
    MAX(amount) FILTER (WHERE payment_method = 'cod')         AS max_cod_payment
FROM payments;
```

```
 credit_card_revenue | avg_credit_card_payment | max_cod_payment
------------------------+----------------------------+---------------------
            140390.00 |            17548.75000000 |         1957.00
(1 row)
```

### FILTER กับเงื่อนไขซับซ้อน (หลายเงื่อนไข)

เงื่อนไขใน `FILTER (WHERE ...)` เขียนซับซ้อนแค่ไหนก็ได้เหมือน `WHERE` ปกติ:

```sql
SELECT
    COUNT(*) FILTER (WHERE rating >= 4) AS positive_reviews,
    COUNT(*) FILTER (WHERE rating <= 2) AS negative_reviews,
    COUNT(*) FILTER (WHERE rating = 3)  AS neutral_reviews,
    COUNT(*) AS total_reviews
FROM reviews;
```

```
 positive_reviews | negative_reviews | neutral_reviews | total_reviews
---------------------+---------------------+--------------------+------------------
                 13 |                   0 |                  2 |               15
(1 row)
```

### ข้อจำกัดของ FILTER

`FILTER` ใช้ได้เฉพาะกับ aggregate function และ ordered-set/hypothetical-set aggregate เท่านั้น ใช้กับ scalar function หรือ window function ทั่วไปไม่ได้ และเงื่อนไขภายใน `FILTER` ต้องอ้างอิงคอลัมน์จาก `FROM` ของคิวรีนั้น (ไม่สามารถอ้างอิง alias ของ `SELECT` เดียวกันได้)

> **สรุป Step 266:** `FILTER (WHERE condition)` ให้เราคำนวณ aggregate function แบบมีเงื่อนไขต่างกันหลายชุดในคิวรีเดียว โดยอ่านตารางเพียงครั้งเดียว อ่านง่ายและมักมีประสิทธิภาพดีกว่าเทคนิค `CASE WHEN` แบบเก่า

---

## Step 267: Aggregate function หลายตัวในคิวรีเดียว

เราสามารถใส่ aggregate function ได้หลายตัวใน `SELECT` เดียวกัน โดยแต่ละตัวจะคำนวณจากชุดแถวเดียวกัน (ที่ผ่านการกรองด้วย `WHERE` แล้ว) — เทคนิคนี้มีประโยชน์มากในการสร้างรายงานสรุป (summary report) แบบครั้งเดียวจบ

### ตัวอย่าง: รายงานสรุปยอดชำระเงิน

```sql
SELECT
    COUNT(*)               AS payment_count,
    SUM(amount)             AS total_revenue,
    ROUND(AVG(amount), 2)   AS avg_payment,
    MIN(amount)             AS smallest_payment,
    MAX(amount)             AS largest_payment
FROM payments;
```

```
 payment_count | total_revenue | avg_payment | smallest_payment | largest_payment
------------------+------------------+---------------+----------------------+--------------------
             16 |      166314.00 |     10394.63 |             849.00 |         32900.00
(1 row)
```

คิวรีเดียวนี้ให้ข้อมูลสรุปครบถ้วน: มี 16 การชำระเงิน รวมทั้งหมด 166,314 บาท เฉลี่ยครั้งละ 10,394.63 บาท ตั้งแต่รายการที่เล็กสุด (849 บาท) ไปจนถึงใหญ่สุด (32,900 บาท)

### ผสม aggregate หลายตัวกับ FILTER เพื่อเปรียบเทียบกลุ่มย่อย

```sql
SELECT
    ROUND(AVG(unit_price), 2)                                          AS avg_price_all,
    ROUND(AVG(unit_price) FILTER (WHERE is_active = true), 2)         AS avg_price_active,
    ROUND(AVG(unit_price) FILTER (WHERE is_active = false), 2)        AS avg_price_inactive,
    COUNT(*) FILTER (WHERE unit_price > 5000)                          AS premium_product_count
FROM products;
```

```
 avg_price_all | avg_price_active | avg_price_inactive | premium_product_count
------------------+---------------------+------------------------+-------------------------
       5778.95 |           5999.42 |             1590.00 |                       7
(1 row)
```

### aggregate function หลายตัวช่วยตรวจสอบคุณภาพข้อมูล (data quality check)

การรวม `COUNT(*)` กับ `COUNT(column)` หลาย ๆ คอลัมน์ในคิวรีเดียวเป็นเทคนิคยอดนิยมสำหรับเช็คว่าตารางมีค่า `NULL` ที่ไม่คาดคิดอยู่หรือไม่:

```sql
SELECT
    COUNT(*)              AS total_rows,
    COUNT(email)           AS non_null_email,
    COUNT(country)         AS non_null_country,
    COUNT(*) - COUNT(email)   AS missing_email,
    COUNT(*) - COUNT(country) AS missing_country
FROM customers;
```

```
 total_rows | non_null_email | non_null_country | missing_email | missing_country
--------------+-------------------+---------------------+------------------+--------------------
          15 |              15 |                15 |              0 |                0
(1 row)
```

ในกรณีนี้ข้อมูลลูกค้าสมบูรณ์ 100% ไม่มีค่าใดขาดหาย

### ข้อควรระวัง: aggregate function หลายตัวคำนวณจากแถว "เดียวกัน" เสมอ ไม่ใช่คนละชุด

จุดสำคัญที่มือใหม่มักเข้าใจผิดคือ เมื่อเขียน aggregate หลายตัวใน `SELECT` เดียวกัน (โดยไม่มี `FILTER` ต่างกัน) ทุกตัวจะคำนวณจาก**แถวชุดเดียวกันที่ผ่าน `WHERE`** เสมอ:

```sql
-- ทั้ง SUM และ COUNT คำนวณจากแถวเดียวกันที่ status = 'delivered' เท่านั้น
SELECT
    COUNT(*)   AS delivered_order_lines,
    SUM(oi.quantity * oi.unit_price) AS delivered_revenue
FROM orders o
JOIN order_items oi ON oi.order_id = o.order_id
WHERE o.status = 'delivered';
```

```
 delivered_order_lines | delivered_revenue
--------------------------+----------------------
                     26 |          142174.00
(1 row)
```

> **สรุป Step 267:** ใส่ aggregate function ได้หลายตัวในคิวรีเดียว ทุกตัวจะคำนวณจากแถวชุดเดียวกันที่ผ่าน `WHERE` (ยกเว้นตัวที่มี `FILTER` กำกับ ซึ่งจะกรองซ้ำเฉพาะตัวมันเอง) เทคนิคนี้เหมาะมากสำหรับสร้างรายงานสรุปและตรวจสอบคุณภาพข้อมูล

---

## Step 268: STRING_AGG และ ARRAY_AGG — รวมข้อความ/สร้าง array จากหลายแถว

`STRING_AGG` และ `ARRAY_AGG` เป็น aggregate function ที่ไม่ได้ "คำนวณตัวเลข" แต่ "รวบรวม" ค่าจากหลายแถวให้กลายเป็นข้อความเดียวหรือ array เดียว มีประโยชน์มากเวลาต้องการแสดงรายการในรูปแบบอ่านง่าย

### STRING_AGG(expression, delimiter) — รวมข้อความคั่นด้วยตัวคั่น

```sql
SELECT STRING_AGG(product_name, ', ') AS accessory_list
FROM products
WHERE category_id = 4;  -- category_id 4 = Accessories
```

```
                                        accessory_list
-------------------------------------------------------------------------------------------
 Wireless Mouse, Mechanical Keyboard, Laptop Stand, Bluetooth Earbuds, USB-C Hub
(1 row)
```

### เรียงลำดับภายใน STRING_AGG ด้วย ORDER BY

โดยปกติลำดับของค่าที่ `STRING_AGG`/`ARRAY_AGG` รวมมาจะ**ไม่รับประกันลำดับ** (ขึ้นกับลำดับที่ database engine อ่านแถวจริง ซึ่งอาจไม่ตรงกับลำดับที่ insert) ถ้าต้องการลำดับที่แน่นอน ต้องใส่ `ORDER BY` ไว้**ภายใน** วงเล็บของ aggregate function เอง (ไม่ใช่ `ORDER BY` ท้ายคิวรี เพราะคิวรีนี้มีแค่ 1 แถวผลลัพธ์):

```sql
SELECT STRING_AGG(product_name, ', ' ORDER BY unit_price) AS accessory_list_by_price
FROM products
WHERE category_id = 4;
```

```
                                             accessory_list_by_price
-------------------------------------------------------------------------------------------------------------
 Wireless Mouse, Laptop Stand, USB-C Hub, Bluetooth Earbuds, Mechanical Keyboard
(1 row)
```

ผลลัพธ์นี้เรียงจากสินค้าราคาถูกที่สุด (Wireless Mouse, 259.00) ไปแพงที่สุด (Mechanical Keyboard, 1490.00)

### ARRAY_AGG(expression) — รวมเป็น PostgreSQL array

```sql
SELECT ARRAY_AGG(product_name ORDER BY unit_price) AS accessory_array
FROM products
WHERE category_id = 4;
```

```
                                              accessory_array
------------------------------------------------------------------------------------------------------
 {"Wireless Mouse","Laptop Stand","USB-C Hub","Bluetooth Earbuds","Mechanical Keyboard"}
(1 row)
```

`ARRAY_AGG` ให้ผลลัพธ์เป็นชนิดข้อมูล array จริง ๆ (เช่น `text[]`) ซึ่งสามารถนำไปใช้ต่อกับฟังก์ชัน array อื่น ๆ ได้ (เช่น `array_length`, `unnest`) ต่างจาก `STRING_AGG` ที่ได้เป็น `text` ธรรมดา

### STRING_AGG/ARRAY_AGG ร่วมกับ DISTINCT

เหมือน `COUNT(DISTINCT ...)` เราสามารถใส่ `DISTINCT` ใน `STRING_AGG`/`ARRAY_AGG` เพื่อตัดค่าซ้ำออกก่อนรวมได้:

```sql
SELECT ARRAY_AGG(DISTINCT payment_method ORDER BY payment_method) AS payment_methods_used
FROM payments;
```

```
                  payment_methods_used
---------------------------------------------
 {bank_transfer,cod,credit_card,promptpay}
(1 row)
```

> **หมายเหตุ syntax:** เมื่อใช้ `DISTINCT` ร่วมกับ `ORDER BY` ภายใน aggregate function เดียวกัน นิพจน์ใน `ORDER BY` ต้องปรากฏใน argument list ของ aggregate function ด้วย (ในที่นี้คือ `payment_method` ทั้งคู่) มิฉะนั้น PostgreSQL จะ error

### รวมข้อความจากหลายคอลัมน์ก่อน aggregate

`STRING_AGG` รับ**นิพจน์** ไม่ใช่แค่ชื่อคอลัมน์เดี่ยว ๆ จึงต่อข้อความหลายคอลัมน์เข้าด้วยกันก่อน แล้วค่อย aggregate ได้:

```sql
SELECT STRING_AGG(first_name || ' ' || last_name, ', ' ORDER BY customer_id) AS thai_customers
FROM customers
WHERE country = 'Thailand';
```

```
                                                          thai_customers
-------------------------------------------------------------------------------------------------------------------------
 Somchai Jaidee, Suda Meesuk, Anan Wong, Nattaya Srisawat, Pichai Boonmee, Kanya Thongdee, Wichai Saetang, Malee Phromma,
 Rattana Chaisuk, Somsak Intharak, Piyawan Kulsri
(1 row)
```

### กรณีใช้งานจริง: รวมรีวิวของสินค้าหนึ่งชิ้นเป็นสรุปเดียว

```sql
SELECT
    COUNT(*)                                            AS review_count,
    ROUND(AVG(rating), 2)                                AS avg_rating,
    STRING_AGG(review_text, ' | ' ORDER BY review_date)  AS all_reviews
FROM reviews
WHERE product_id = 7;  -- Bluetooth Earbuds
```

```
 review_count | avg_rating |                                          all_reviews
-----------------+--------------+-----------------------------------------------------------------------------------
              2 |        4.0 | เสียงโอเค แต่การเชื่อมต่อหลุดบ่อย | หูฟังเสียงดีมาก ใส่สบาย
(1 row)
```

> **สรุป Step 268:** `STRING_AGG` รวมค่าเป็นข้อความเดียวคั่นด้วยตัวคั่นที่กำหนด `ARRAY_AGG` รวมค่าเป็น PostgreSQL array ทั้งคู่รองรับ `DISTINCT` และ `ORDER BY` ภายในวงเล็บของฟังก์ชันเพื่อควบคุมลำดับผลลัพธ์

---

## Step 269: Statistical aggregate เบื้องต้น: STDDEV, VARIANCE, PERCENTILE_CONT/DISC

นอกเหนือจาก aggregate function พื้นฐาน PostgreSQL ยังมี aggregate function เชิงสถิติในตัว ที่ใช้บ่อยในการวิเคราะห์ข้อมูลเชิงลึก (data analysis)

### STDDEV และ VARIANCE — วัดการกระจายตัวของข้อมูล

| ฟังก์ชัน | ความหมาย |
|---|---|
| `STDDEV(x)` / `STDDEV_SAMP(x)` | ส่วนเบี่ยงเบนมาตรฐานของ**กลุ่มตัวอย่าง** (sample standard deviation, หารด้วย n-1) |
| `STDDEV_POP(x)` | ส่วนเบี่ยงเบนมาตรฐานของ**ประชากรทั้งหมด** (population standard deviation, หารด้วย n) |
| `VARIANCE(x)` / `VAR_SAMP(x)` | ความแปรปรวนของกลุ่มตัวอย่าง |
| `VAR_POP(x)` | ความแปรปรวนของประชากรทั้งหมด |

`STDDEV()` และ `VARIANCE()` เป็นชื่อย่อของ `STDDEV_SAMP()` และ `VAR_SAMP()` ตามลำดับ (ใช้สูตรหารด้วย n-1 ซึ่งเหมาะกับกรณีที่ข้อมูลของเราเป็นเพียง "ตัวอย่าง" จากประชากรที่ใหญ่กว่า)

```sql
SELECT
    ROUND(AVG(unit_price), 2)         AS avg_price,
    ROUND(STDDEV(unit_price), 2)      AS stddev_sample,
    ROUND(STDDEV_POP(unit_price), 2)  AS stddev_population,
    ROUND(VARIANCE(unit_price), 2)    AS variance_sample
FROM products;
```

```
 avg_price | stddev_sample | stddev_population | variance_sample
-------------+------------------+-----------------------+--------------------
   5778.95 |         8906.24 |             8680.73 |      79321064.16
(1 row)
```

ตีความ: ราคาสินค้าเฉลี่ยอยู่ที่ 5,778.95 บาท แต่ส่วนเบี่ยงเบนมาตรฐาน (8,906.24) **มากกว่าค่าเฉลี่ยเสียอีก** ซึ่งบ่งชี้ว่าราคาสินค้ากระจายตัวกว้างมาก (มีทั้งสินค้าราคาหลักร้อยและหลักหมื่น) — ค่าเฉลี่ยเพียงอย่างเดียวจึงไม่สามารถสะท้อนภาพรวมของข้อมูลได้ครบถ้วน ต้องดู `STDDEV` ประกอบเสมอ

### เมื่อไรใช้ SAMP เมื่อไรใช้ POP?

- ใช้ `STDDEV_SAMP`/`VAR_SAMP` (หรือชื่อย่อ `STDDEV`/`VARIANCE`) เมื่อข้อมูลที่มีอยู่เป็นเพียง**ตัวอย่างส่วนหนึ่ง**ของประชากรทั้งหมด (กรณีทั่วไปในงานวิเคราะห์ธุรกิจ เช่น รีวิวที่มีอยู่เป็นแค่ส่วนหนึ่งของลูกค้าทั้งหมดที่อาจจะรีวิว)
- ใช้ `STDDEV_POP`/`VAR_POP` เมื่อข้อมูลที่มีอยู่คือ**ประชากรทั้งหมด**จริง ๆ (เช่น ราคาสินค้าทุกชิ้นที่ขายจริงในร้าน ไม่ใช่การสุ่มตัวอย่าง)

### PERCENTILE_CONT และ PERCENTILE_DISC — หา median และ percentile อื่น ๆ

`PERCENTILE_CONT`/`PERCENTILE_DISC` เป็น **ordered-set aggregate function** ที่มี syntax ต่างจาก aggregate ทั่วไปเล็กน้อย ต้องใช้ `WITHIN GROUP (ORDER BY ...)` แทน argument ปกติ:

```sql
aggregate_function(fraction) WITHIN GROUP (ORDER BY expression)
```

ความแตกต่างระหว่างสองฟังก์ชัน:
- `PERCENTILE_CONT(fraction)` — คำนวณแบบ**ต่อเนื่อง (continuous)** โดยใช้การประมาณค่าเชิงเส้น (linear interpolation) ระหว่างค่าจริงสองค่าที่ใกล้ตำแหน่งเปอร์เซ็นไทล์นั้น ผลลัพธ์อาจเป็นค่าที่**ไม่มีอยู่จริง**ในข้อมูล
- `PERCENTILE_DISC(fraction)` — คำนวณแบบ**ไม่ต่อเนื่อง (discrete)** โดยเลือกค่าจริง**ที่มีอยู่ในข้อมูล**เท่านั้น (ไม่ประมาณค่า)

```sql
SELECT
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY unit_price) AS median_cont,
    PERCENTILE_DISC(0.5) WITHIN GROUP (ORDER BY unit_price) AS median_disc,
    PERCENTILE_CONT(0.9) WITHIN GROUP (ORDER BY unit_price) AS p90_price
FROM products;
```

```
 median_cont | median_disc | p90_price
---------------+---------------+-------------
       1890.0 |     2190.00 |   16881.00
(1 row)
```

`median_cont` (1890.0) คือค่ามัธยฐานแบบประมาณเชิงเส้น ซึ่งอยู่ระหว่างค่าจริงสองค่าที่ตำแหน่งกลาง ๆ ของข้อมูล 20 ค่า (ระหว่าง 1590.00 กับ 2190.00) ในขณะที่ `median_disc` (2190.00) คือค่าจริงที่มีอยู่ในตารางที่ใกล้ตำแหน่งมัธยฐานที่สุด

### เปรียบเทียบ MEDIAN กับ AVG — ทำไมบางครั้ง median สื่อความหมายดีกว่า

```sql
SELECT
    ROUND(AVG(unit_price), 2)                                        AS avg_price,
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY unit_price)          AS median_price
FROM products;
```

```
 avg_price | median_price
-------------+----------------
   5778.95 |         1890.0
(1 row)
```

สังเกตว่า `avg_price` (5,778.95) สูงกว่า `median_price` (1,890.0) อย่างมาก เพราะมีสินค้าราคาแพงมาก ๆ ไม่กี่ชิ้น (เช่น Gaming Laptop 32,900 บาท) ที่ดึงค่าเฉลี่ยให้สูงขึ้นผิดปกติ (skewed distribution) ในสถานการณ์แบบนี้ **median สะท้อน "ราคาสินค้าทั่วไปในร้าน" ได้ตรงกว่าค่าเฉลี่ย**

### PERCENTILE_CONT รับ array ของหลาย fraction พร้อมกันได้

```sql
SELECT PERCENTILE_CONT(ARRAY[0.25, 0.5, 0.75]) WITHIN GROUP (ORDER BY amount) AS quartiles
FROM payments;
```

```
                  quartiles
---------------------------------------
 {2237.5,6530,15467.5}
(1 row)
```

ผลลัพธ์นี้คือ Q1 (25th percentile) = 2,237.5, Q2/median (50th percentile) = 6,530, และ Q3 (75th percentile) = 15,467.5 ของยอดชำระเงิน ในคิวรีเดียว

> **สรุป Step 269:** `STDDEV`/`VARIANCE` วัดการกระจายตัวของข้อมูล ส่วน `PERCENTILE_CONT`/`PERCENTILE_DISC` หาค่ามัธยฐานหรือเปอร์เซ็นไทล์อื่น ๆ ผ่าน syntax พิเศษ `WITHIN GROUP (ORDER BY ...)` — ใช้ median แทน average เมื่อข้อมูลมีค่าผิดปกติสูง (outlier) ที่อาจทำให้ค่าเฉลี่ยบิดเบือน

---

## Step 270: แบบฝึกหัดรวม — สร้าง Dashboard สรุปยอดขายของระบบ e-commerce

มาถึงจุดนี้เรามีเครื่องมือครบถ้วนสำหรับสร้าง "dashboard" สรุปข้อมูลเชิงธุรกิจด้วยคิวรีเดียว (หรือหลายคิวรีที่ประกอบกันเป็นแผงข้อมูล) ลองรวมทุกเทคนิคที่เรียนมาในบทนี้เข้าด้วยกัน

### Dashboard ส่วนที่ 1: ภาพรวมยอดขายทั้งระบบ

```sql
SELECT
    COUNT(DISTINCT o.order_id)      AS total_orders,
    COUNT(DISTINCT o.customer_id)   AS unique_customers,
    SUM(oi.quantity)                 AS total_units_sold,
    SUM(oi.quantity * oi.unit_price) AS gross_revenue,
    ROUND(AVG(oi.quantity * oi.unit_price), 2) AS avg_line_value,
    COUNT(*) FILTER (WHERE o.status = 'delivered')  AS delivered_lines,
    MIN(o.order_date)                AS first_order_date,
    MAX(o.order_date)                AS last_order_date
FROM orders o
JOIN order_items oi ON oi.order_id = o.order_id;
```

```
 total_orders | unique_customers | total_units_sold | gross_revenue | avg_line_value | delivered_lines |     first_order_date      |      last_order_date
----------------+---------------------+---------------------+------------------+-------------------+---------------------+-----------------------------+-----------------------------
            20 |                  15 |                42 |     190774.00 |         5961.69 |                  26 | 2024-01-05 10:20:00+07     | 2024-04-28 20:30:00+07
(1 row)
```

### Dashboard ส่วนที่ 2: สุขภาพของออเดอร์ (order health) และรายได้จริง

```sql
SELECT
    COUNT(*)                                        AS total_orders,
    COUNT(*) FILTER (WHERE status = 'delivered')    AS delivered,
    COUNT(*) FILTER (WHERE status = 'cancelled')    AS cancelled,
    COUNT(*) FILTER (WHERE status IN ('pending', 'processing')) AS in_progress,
    ROUND(
        100.0 * COUNT(*) FILTER (WHERE status = 'cancelled') / COUNT(*), 2
    ) AS cancellation_rate_pct
FROM orders;
```

```
 total_orders | delivered | cancelled | in_progress | cancellation_rate_pct
----------------+-------------+-------------+---------------+---------------------------
            20 |        14 |         2 |           2 |                     10.00
(1 row)
```

```sql
SELECT
    COUNT(*)                     AS successful_payments,
    SUM(amount)                   AS confirmed_revenue,
    ROUND(AVG(amount), 2)         AS avg_payment,
    ROUND(STDDEV(amount), 2)      AS payment_stddev,
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY amount) AS median_payment
FROM payments;
```

```
 successful_payments | confirmed_revenue | avg_payment | payment_stddev | median_payment
------------------------+----------------------+---------------+-------------------+------------------
                   16 |          166314.00 |     10394.63 |         10180.46 |          6530.0
(1 row)
```

### Dashboard ส่วนที่ 3: คลังสินค้าและความหลากหลาย

```sql
SELECT
    COUNT(*)                                    AS total_products,
    COUNT(*) FILTER (WHERE is_active = true)     AS active_products,
    COUNT(*) FILTER (WHERE is_active = false)    AS discontinued_products,
    COUNT(DISTINCT category_id)                   AS categories_in_use,
    SUM(stock_quantity)                            AS total_units_in_stock,
    ROUND(AVG(unit_price), 2)                      AS avg_price,
    MIN(unit_price)                                AS cheapest,
    MAX(unit_price)                                AS most_expensive
FROM products;
```

```
 total_products | active_products | discontinued_products | categories_in_use | total_units_in_stock | avg_price | cheapest | most_expensive
-------------------+---------------------+---------------------------+-----------------------+--------------------------+-------------+------------+-------------------
              20 |               19 |                      1 |                   9 |                  1553 |   5778.95 |   259.00 |        32900.00
(1 row)
```

### Dashboard ส่วนที่ 4: ความพึงพอใจของลูกค้า (customer satisfaction)

```sql
SELECT
    COUNT(*)                                      AS total_reviews,
    ROUND(AVG(rating), 2)                          AS avg_rating,
    ROUND(STDDEV(rating), 2)                       AS rating_stddev,
    COUNT(*) FILTER (WHERE rating >= 4)             AS positive_reviews,
    COUNT(*) FILTER (WHERE rating <= 2)             AS negative_reviews,
    ROUND(
        100.0 * COUNT(*) FILTER (WHERE rating >= 4) / COUNT(*), 2
    ) AS satisfaction_rate_pct,
    STRING_AGG(DISTINCT rating::text, ', ' ORDER BY rating::text) AS ratings_seen
FROM reviews;
```

```
 total_reviews | avg_rating | rating_stddev | positive_reviews | negative_reviews | satisfaction_rate_pct | ratings_seen
------------------+--------------+------------------+---------------------+---------------------+-----------------------------+----------------
             15 |        4.4 |           0.74 |                13 |                 0 |                      86.67 | 3, 4, 5
(1 row)
```

### Dashboard ส่วนที่ 5: ช่องทางการชำระเงินยอดนิยม

```sql
SELECT
    ARRAY_AGG(DISTINCT payment_method ORDER BY payment_method) AS methods_available,
    SUM(amount) FILTER (WHERE payment_method = 'credit_card')   AS credit_card_revenue,
    SUM(amount) FILTER (WHERE payment_method = 'promptpay')     AS promptpay_revenue,
    SUM(amount) FILTER (WHERE payment_method = 'bank_transfer') AS bank_transfer_revenue,
    SUM(amount) FILTER (WHERE payment_method = 'cod')           AS cod_revenue
FROM payments;
```

```
              methods_available            | credit_card_revenue | promptpay_revenue | bank_transfer_revenue | cod_revenue
---------------------------------------------+------------------------+----------------------+---------------------------+---------------
 {bank_transfer,cod,credit_card,promptpay} |            140390.00 |           16278.00 |                6840.00 |     2806.00
(1 row)
```

ทั้ง 5 ส่วนนี้รวมกันเป็น "dashboard" ที่ครอบคลุมมุมมองสำคัญของธุรกิจ e-commerce: ยอดขายโดยรวม สถานะออเดอร์ คลังสินค้า ความพึงพอใจลูกค้า และช่องทางการชำระเงิน — ทั้งหมดนี้ทำได้ด้วย `COUNT`, `SUM`, `AVG`, `MIN`, `MAX`, `FILTER`, `STRING_AGG`, `ARRAY_AGG`, `STDDEV` และ `PERCENTILE_CONT` ที่เรียนมาในบทนี้ล้วน ๆ โดยยังไม่ต้องพึ่ง `GROUP BY` เลยแม้แต่น้อย

> ในบทถัดไป (Part 028) เราจะเรียนรู้ `GROUP BY`/`HAVING` ซึ่งจะทำให้ dashboard เหล่านี้ละเอียดขึ้นไปอีกขั้น เช่น สรุปยอดขายแยกตามลูกค้าแต่ละคน แยกตามหมวดหมู่สินค้า หรือแยกตามเดือน แทนที่จะสรุปเป็นตัวเลขก้อนเดียวแบบในบทนี้

---

## สรุปท้ายบท

ในบทนี้เราได้เรียนรู้ aggregate function ซึ่งเป็นเครื่องมือพื้นฐานที่สำคัญที่สุดตัวหนึ่งในการวิเคราะห์ข้อมูลด้วย SQL:

1. **Aggregate function** ย่อค่าจากหลายแถวให้เหลือค่าเดียว และห้ามปนกับคอลัมน์ที่ไม่ได้ aggregate โดยไม่มี `GROUP BY`
2. **COUNT** มี 3 รูปแบบ: `COUNT(*)` นับทุกแถว, `COUNT(column)` นับเฉพาะที่ไม่ใช่ `NULL`, `COUNT(DISTINCT column)` นับค่าที่ไม่ซ้ำกัน
3. **SUM** และ **AVG** ข้าม `NULL` โดยอัตโนมัติ แต่คืนค่า `NULL` (ไม่ใช่ 0) เมื่อไม่มีแถวให้คำนวณเลย — ใช้ `COALESCE` เมื่อต้องการค่าเริ่มต้นเป็น 0
4. **AVG** คืนผลลัพธ์เป็น `numeric` ที่แม่นยำเสมอ ไม่เกิด integer division truncation เหมือนการหารตรง ๆ
5. **MIN/MAX** ใช้ได้กับทุกชนิดข้อมูลที่เปรียบเทียบได้ (ตัวเลข ข้อความ วันที่) ยกเว้น `BOOLEAN` ที่ต้องใช้ `BOOL_AND`/`BOOL_OR` แทน
6. **FILTER (WHERE ...)** ให้คำนวณ aggregate แบบมีเงื่อนไขต่างกันหลายชุดในคิวรีเดียว อ่านง่ายกว่าและมักเร็วกว่าเทคนิค `CASE WHEN` แบบเก่า
7. การใส่ **aggregate function หลายตัว**ในคิวรีเดียวช่วยสร้างรายงานสรุปได้ครบถ้วนในครั้งเดียว
8. **STRING_AGG** และ **ARRAY_AGG** รวมค่าจากหลายแถวเป็นข้อความหรือ array เดียว รองรับ `DISTINCT` และ `ORDER BY` ภายในวงเล็บ
9. **STDDEV/VARIANCE** วัดการกระจายตัวของข้อมูล ส่วน **PERCENTILE_CONT/PERCENTILE_DISC** หาค่ามัธยฐานและเปอร์เซ็นไทล์ ซึ่งบางครั้งสะท้อนภาพข้อมูลได้ดีกว่าค่าเฉลี่ยเมื่อมี outlier
10. การผสมผสานเทคนิคทั้งหมดนี้เข้าด้วยกันทำให้สร้าง **dashboard สรุปข้อมูลเชิงธุรกิจ** ได้อย่างมีประสิทธิภาพโดยไม่ต้องพึ่ง `GROUP BY`

บทถัดไปเราจะต่อยอดความรู้นี้ไปสู่ **`GROUP BY` และ `HAVING`** ซึ่งจะทำให้เราสรุปข้อมูลแบบแบ่งกลุ่มย่อยได้ (เช่น ยอดขายแยกตามลูกค้า แยกตามหมวดหมู่ แยกตามเดือน) — เป็นก้าวสำคัญที่จะปลดล็อกพลังที่แท้จริงของ aggregate function

---

## แบบฝึกหัด

พยายามเขียนคิวรีด้วยตัวเองก่อนเปิดดูเฉลย ใช้ชุดข้อมูลจากส่วน "เตรียมข้อมูล" ด้านบนในการฝึกฝนทุกข้อ

### แบบฝึกหัดที่ 1

เขียนคิวรีนับจำนวนลูกค้าทั้งหมด และนับจำนวนลูกค้าที่มาจากประเทศไทย (`country = 'Thailand'`) ในคิวรีเดียว โดยไม่ใช้ `GROUP BY`

<details>
<summary>เฉลยแบบฝึกหัดที่ 1</summary>

```sql
SELECT
    COUNT(*) AS total_customers,
    COUNT(*) FILTER (WHERE country = 'Thailand') AS thai_customers
FROM customers;
```

```
 total_customers | thai_customers
--------------------+-------------------
              15 |               11
(1 row)
```

ใช้ `FILTER (WHERE ...)` เพื่อคำนวณจำนวนลูกค้าไทยแยกจากจำนวนลูกค้าทั้งหมดในคิวรีเดียว โดยไม่ต้องรันคิวรีสองครั้ง

</details>

### แบบฝึกหัดที่ 2

หายอดรวม (`SUM`) ของยอดชำระเงินทั้งหมดจากตาราง `payments` พร้อมทั้งแสดงจำนวนรายการชำระเงินด้วย

<details>
<summary>เฉลยแบบฝึกหัดที่ 2</summary>

```sql
SELECT
    COUNT(*)     AS payment_count,
    SUM(amount)  AS total_revenue
FROM payments;
```

```
 payment_count | total_revenue
------------------+------------------
             16 |      166314.00
(1 row)
```

</details>

### แบบฝึกหัดที่ 3

หาค่าเฉลี่ยราคาสินค้า (`unit_price`) เฉพาะสินค้าที่ยังขายอยู่ (`is_active = true`) ปัดเศษทศนิยม 2 ตำแหน่ง

<details>
<summary>เฉลยแบบฝึกหัดที่ 3</summary>

```sql
SELECT ROUND(AVG(unit_price), 2) AS avg_active_price
FROM products
WHERE is_active = true;
```

```
 avg_active_price
---------------------
          5999.42
(1 row)
```

หมายเหตุ: ใช้ `WHERE` กรองก่อน ไม่ใช่ `FILTER` เพราะในคิวรีนี้มี aggregate function เพียงตัวเดียวและต้องการกรองทั้งคิวรี `WHERE` จึงเหมาะสมและมีประสิทธิภาพกว่า (`FILTER` เหมาะกับกรณีที่ต้องการเงื่อนไข**ต่างกัน**สำหรับ aggregate หลายตัวในคิวรีเดียว)

</details>

### แบบฝึกหัดที่ 4

หาสินค้าที่ราคาถูกที่สุดและแพงที่สุด พร้อมชื่อสินค้า (ไม่ใช่แค่ราคา)

<details>
<summary>เฉลยแบบฝึกหัดที่ 4</summary>

```sql
SELECT product_name, unit_price
FROM products
WHERE unit_price = (SELECT MIN(unit_price) FROM products)
   OR unit_price = (SELECT MAX(unit_price) FROM products)
ORDER BY unit_price;
```

```
    product_name     | unit_price
-----------------------+-------------
 Wireless Mouse       |     259.00
 Gaming Laptop 15"    |   32900.00
(2 rows)
```

`MIN`/`MAX` คืนแค่ "ค่า" ตัวเดียว ไม่ใช่ทั้งแถว จึงต้องใช้ subquery เพื่อนำค่านั้นไปหาแถวที่ตรงกันอีกครั้ง

</details>

### แบบฝึกหัดที่ 5

นับจำนวนออเดอร์ทั้งหมด แยกตามสถานะ (`delivered`, `shipped`, `pending`, `processing`, `cancelled`) ในคิวรีเดียว โดยไม่ใช้ `GROUP BY`

<details>
<summary>เฉลยแบบฝึกหัดที่ 5</summary>

```sql
SELECT
    COUNT(*) AS total_orders,
    COUNT(*) FILTER (WHERE status = 'delivered')  AS delivered,
    COUNT(*) FILTER (WHERE status = 'shipped')    AS shipped,
    COUNT(*) FILTER (WHERE status = 'pending')    AS pending,
    COUNT(*) FILTER (WHERE status = 'processing') AS processing,
    COUNT(*) FILTER (WHERE status = 'cancelled')  AS cancelled
FROM orders;
```

```
 total_orders | delivered | shipped | pending | processing | cancelled
----------------+-------------+-----------+-----------+--------------+-------------
            20 |        14 |       2 |       1 |          1 |         2
(1 row)
```

</details>

### แบบฝึกหัดที่ 6

หาค่าเฉลี่ยคะแนนรีวิว (`rating`) และส่วนเบี่ยงเบนมาตรฐาน (`STDDEV`) จากตาราง `reviews`

<details>
<summary>เฉลยแบบฝึกหัดที่ 6</summary>

```sql
SELECT
    ROUND(AVG(rating), 2)    AS avg_rating,
    ROUND(STDDEV(rating), 2) AS rating_stddev,
    COUNT(*)                  AS total_reviews
FROM reviews;
```

```
 avg_rating | rating_stddev | total_reviews
--------------+------------------+------------------
        4.4 |           0.74 |             15
(1 row)
```

ค่าเฉลี่ยรีวิว 4.4 จาก 5 พร้อมส่วนเบี่ยงเบนมาตรฐานต่ำ (0.74) บ่งบอกว่าลูกค้าส่วนใหญ่ให้คะแนนใกล้เคียงกันและเป็นบวก

</details>

### แบบฝึกหัดที่ 7

ใช้ `STRING_AGG` รวมชื่อ-นามสกุลลูกค้าทั้งหมดจากประเทศไทย เป็นข้อความเดียวคั่นด้วยจุลภาค เรียงตาม `customer_id`

<details>
<summary>เฉลยแบบฝึกหัดที่ 7</summary>

```sql
SELECT STRING_AGG(first_name || ' ' || last_name, ', ' ORDER BY customer_id) AS thai_customer_names
FROM customers
WHERE country = 'Thailand';
```

```
                                                       thai_customer_names
--------------------------------------------------------------------------------------------------------------------------
 Somchai Jaidee, Suda Meesuk, Anan Wong, Nattaya Srisawat, Pichai Boonmee, Kanya Thongdee, Wichai Saetang, Malee Phromma,
 Rattana Chaisuk, Somsak Intharak, Piyawan Kulsri
(1 row)
```

</details>

### แบบฝึกหัดที่ 8

ใช้ `ARRAY_AGG` เก็บรายการ `payment_method` ที่ไม่ซ้ำกัน (`DISTINCT`) เรียงตามตัวอักษร จากตาราง `payments`

<details>
<summary>เฉลยแบบฝึกหัดที่ 8</summary>

```sql
SELECT ARRAY_AGG(DISTINCT payment_method ORDER BY payment_method) AS distinct_methods
FROM payments;
```

```
              distinct_methods
---------------------------------------------
 {bank_transfer,cod,credit_card,promptpay}
(1 row)
```

</details>

### แบบฝึกหัดที่ 9

หาค่ามัธยฐาน (median) ของยอดชำระเงิน (`payments.amount`) ด้วย `PERCENTILE_CONT`

<details>
<summary>เฉลยแบบฝึกหัดที่ 9</summary>

```sql
SELECT PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY amount) AS median_payment
FROM payments;
```

```
 median_payment
------------------
          6530.0
(1 row)
```

ยอดชำระเงินมัธยฐานอยู่ที่ 6,530 บาท ซึ่งต่ำกว่าค่าเฉลี่ย (10,394.63 บาท จากแบบฝึกหัดที่ 2 หากคำนวณด้วย `AVG`) ค่อนข้างมาก บ่งชี้ว่ามีการชำระเงินก้อนใหญ่บางรายการ (เช่น 32,900 บาท) ที่ดึงค่าเฉลี่ยให้สูงกว่าค่ากลางทั่วไป

</details>

### แบบฝึกหัดที่ 10

สร้างคิวรี dashboard สรุปสถานะระบบ e-commerce ในคิวรีเดียว ประกอบด้วย: จำนวนคำสั่งซื้อทั้งหมด, ยอดขายรวมจาก `payments`, ค่าเฉลี่ยยอดชำระเงินต่อครั้ง, จำนวนคำสั่งซื้อที่ถูกยกเลิก, และจำนวนสินค้าที่ยัง active อยู่

<details>
<summary>เฉลยแบบฝึกหัดที่ 10</summary>

```sql
SELECT
    (SELECT COUNT(*) FROM orders)                                       AS total_orders,
    (SELECT SUM(amount) FROM payments)                                  AS total_revenue,
    (SELECT ROUND(AVG(amount), 2) FROM payments)                        AS avg_payment,
    (SELECT COUNT(*) FROM orders WHERE status = 'cancelled')            AS cancelled_orders,
    (SELECT COUNT(*) FROM products WHERE is_active = true)              AS active_products;
```

```
 total_orders | total_revenue | avg_payment | cancelled_orders | active_products
----------------+------------------+---------------+----------------------+-------------------
            20 |      166314.00 |     10394.63 |                 2 |               19
(1 row)
```

ในที่นี้ใช้ scalar subquery แยกแต่ละค่า เพราะแต่ละค่ามาจากตารางคนละตารางกัน (orders, payments, products) จึง `JOIN` รวมกันไม่ได้ตรง ๆ โดยไม่ทำให้ตัวเลขผิดเพี้ยนจาก fan-out (การคูณแถวซ้ำจาก join) — เป็นแนวทางที่ปลอดภัยเมื่อต้องรวมค่าสรุปจากหลายตารางที่ไม่มีความสัมพันธ์แบบ 1-to-1 ต่อกันโดยตรง

</details>

---

**บทถัดไป:** [Part 028: GROUP BY และ HAVING](./part-028-group-by-having.md)
