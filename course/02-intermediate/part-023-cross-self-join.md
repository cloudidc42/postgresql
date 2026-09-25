# CROSS JOIN, SELF JOIN และ NATURAL JOIN

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับกลาง | Part 023

---

## เป้าหมายการเรียนรู้

เมื่อเรียนจบบทนี้ ผู้เรียนจะสามารถ:

- อธิบายแนวคิด **cartesian product** และเขียน `CROSS JOIN` ได้อย่างถูกต้อง
- ใช้ `CROSS JOIN` แก้ปัญหาจริง เช่น สร้างตารางสินค้าแบบ size × color หรือสร้างปฏิทินร่วมกับ `generate_series`
- รู้เท่าทัน "กับดัก" ของ `CROSS JOIN` ที่เกิดจากการลืมเงื่อนไข JOIN โดยไม่ตั้งใจ
- เข้าใจแนวคิด **SELF JOIN** และเหตุผลที่ต้องใช้ table alias เสมอ
- เขียน SELF JOIN เพื่อหาความสัมพันธ์แบบลำดับชั้น (hierarchy) เช่น พนักงานกับหัวหน้า
- เขียน SELF JOIN เพื่อเปรียบเทียบแถวภายในตารางเดียวกัน เช่น เปรียบเทียบราคาสินค้าในหมวดหมู่เดียวกัน หรือจับคู่ลูกค้าประเทศเดียวกัน
- เข้าใจการทำงานของ `NATURAL JOIN` และเหตุผลที่มืออาชีพส่วนใหญ่หลีกเลี่ยงการใช้งานจริง
- สรุปเปรียบเทียบ JOIN ทุกประเภทที่เรียนมาได้ในตารางเดียว และเลือกใช้ JOIN ที่เหมาะสมกับสถานการณ์ต่าง ๆ

---

## เตรียมข้อมูล

บทนี้ใช้ชุดข้อมูลร้านค้าออนไลน์ (e-commerce) ชุดเดียวกับ Part 021-039 ทั้งหมด เพื่อให้ตัวอย่างต่อเนื่องกันไปตลอดทั้งหมวด ให้รันสคริปต์ด้านล่างนี้ในฐานข้อมูลทดลองของท่านก่อนเริ่มเรียน

```sql
-- ลบตารางเดิม (ถ้ามี) เพื่อเริ่มต้นใหม่แบบสะอาด
DROP TABLE IF EXISTS payments CASCADE;
DROP TABLE IF EXISTS reviews CASCADE;
DROP TABLE IF EXISTS order_items CASCADE;
DROP TABLE IF EXISTS orders CASCADE;
DROP TABLE IF EXISTS employees CASCADE;
DROP TABLE IF EXISTS customers CASCADE;
DROP TABLE IF EXISTS products CASCADE;
DROP TABLE IF EXISTS suppliers CASCADE;
DROP TABLE IF EXISTS categories CASCADE;

-- โครงสร้างตาราง
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
```

### ข้อมูลตัวอย่าง (seed data)

```sql
-- categories: มีโครงสร้างลำดับชั้น (parent_category_id) สำหรับหมวดหมู่ย่อย
INSERT INTO categories (category_id, category_name, parent_category_id) VALUES
(1,  'Electronics',        NULL),
(2,  'Computers',          1),
(3,  'Smartphones',        1),
(4,  'Clothing',           NULL),
(5,  'Men''s Clothing',    4),
(6,  'Women''s Clothing',  4),
(7,  'Home & Kitchen',     NULL),
(8,  'Furniture',          7),
(9,  'Books',               NULL),
(10, 'Sports & Outdoors',  NULL);
SELECT setval('categories_category_id_seq', 10);

-- suppliers
INSERT INTO suppliers (supplier_id, supplier_name, country) VALUES
(1,  'Global Tech Supply',          'Thailand'),
(2,  'Bangkok Electronics Co.',     'Thailand'),
(3,  'Shenzhen Gadgets Ltd.',       'China'),
(4,  'Fashion Forward Inc.',        'Vietnam'),
(5,  'Home Comfort Manufacturing',  'Thailand'),
(6,  'Nordic Furniture House',      'Sweden'),
(7,  'Pacific Book Distributors',   'Thailand'),
(8,  'Sports Gear International',  'USA'),
(9,  'Siam Textile Group',         'Thailand'),
(10, 'EuroTech Components',        'Germany');
SELECT setval('suppliers_supplier_id_seq', 10);

-- products
INSERT INTO products (product_id, product_name, category_id, supplier_id, unit_price, stock_quantity, is_active) VALUES
(1,  'Wireless Mouse',           2,  1,  350.00,  120, true),
(2,  'Mechanical Keyboard',      2,  1,  1890.00, 45,  true),
(3,  '27-inch Monitor',          2,  10, 6500.00, 20,  true),
(4,  'Laptop Stand',             2,  3,  590.00,  80,  true),
(5,  'Smartphone X12',           3,  2,  15900.00, 30, true),
(6,  'Smartphone Lite',          3,  3,  6900.00, 60,  true),
(7,  'Wireless Earbuds',         1,  2,  1290.00, 100, true),
(8,  'Men''s Cotton T-Shirt',    5,  9,  290.00,  200, true),
(9,  'Men''s Denim Jeans',       5,  4,  890.00,  90,  true),
(10, 'Women''s Summer Dress',    6,  4,  1290.00, 70,  true),
(11, 'Women''s Blouse',          6,  9,  650.00,  110, true),
(12, 'Office Chair',             8,  6,  3900.00, 25,  true),
(13, 'Dining Table Set',         8,  6,  12500.00, 8,  true),
(14, 'Non-stick Frying Pan',     7,  5,  450.00,  150, true),
(15, 'Electric Kettle',          7,  5,  690.00,  130, true),
(16, 'PostgreSQL Mastery Book',  9,  7,  590.00,  60,  true),
(17, 'Thai Cooking Book',        9,  7,  390.00,  75,  true),
(18, 'Yoga Mat',                 10, 8,  590.00,  140, true),
(19, 'Running Shoes',            10, 8,  2490.00, 55,  true),
(20, 'Camping Tent',             10, 8,  3200.00, 15,  false);
SELECT setval('products_product_id_seq', 20);

-- customers
INSERT INTO customers (customer_id, first_name, last_name, email, country, signup_date) VALUES
(1,  'Somchai',  'Jaidee',     'somchai.j@email.com',   'Thailand', '2023-01-15'),
(2,  'Suda',     'Thongdee',   'suda.t@email.com',      'Thailand', '2023-02-20'),
(3,  'Anong',    'Srisuk',     'anong.s@email.com',     'Thailand', '2023-03-05'),
(4,  'John',     'Smith',      'john.smith@email.com',  'USA',      '2023-01-28'),
(5,  'Emily',    'Johnson',    'emily.j@email.com',     'USA',      '2023-04-12'),
(6,  'Wei',      'Chen',       'wei.chen@email.com',    'China',    '2023-02-14'),
(7,  'Li',       'Na',         'li.na@email.com',       'China',    '2023-05-01'),
(8,  'Nguyen',   'Van A',      'nguyen.a@email.com',    'Vietnam',  '2023-03-19'),
(9,  'Somsri',   'Kaewta',     'somsri.k@email.com',    'Thailand', '2023-06-10'),
(10, 'Piti',     'Boonmee',    'piti.b@email.com',      'Thailand', '2023-01-08'),
(11, 'Sarah',    'Williams',   'sarah.w@email.com',     'USA',      '2023-07-22'),
(12, 'Michael',  'Brown',      'michael.b@email.com',   'USA',      '2023-08-15'),
(13, 'Yui',      'Tanaka',     'yui.t@email.com',       'Japan',    '2023-09-01'),
(14, 'Kenji',    'Sato',       'kenji.s@email.com',     'Japan',    '2023-09-15'),
(15, 'Malee',    'Rakthai',    'malee.r@email.com',     'Thailand', '2023-10-02');
SELECT setval('customers_customer_id_seq', 15);

-- employees: มีโครงสร้างลำดับชั้น (manager_id) สำหรับใช้ทำ SELF JOIN
INSERT INTO employees (employee_id, first_name, last_name, hire_date, manager_id, department) VALUES
(1,  'Prasert',   'Wongsawat',   '2018-01-10', NULL, 'Executive'),
(2,  'Kanya',     'Suksawat',    '2018-03-15', 1,    'Sales'),
(3,  'Anan',      'Phetchara',   '2018-06-01', 1,    'Operations'),
(4,  'Nattapong', 'Srisawat',    '2019-02-20', 2,    'Sales'),
(5,  'Ratana',    'Chaiyaporn',  '2019-05-11', 2,    'Sales'),
(6,  'Siriporn',  'Boonyarit',   '2019-08-30', 3,    'Operations'),
(7,  'Wichai',    'Kittisak',    '2020-01-15', 4,    'Sales'),
(8,  'Duangjai',  'Meesuk',      '2020-04-22', 4,    'Sales'),
(9,  'Somkiat',   'Thaweesak',   '2020-07-09', 5,    'Sales'),
(10, 'Panida',    'Jansuk',      '2020-11-03', 6,    'Operations'),
(11, 'Chatchai',  'Rungrueang',  '2021-02-18', 6,    'Operations'),
(12, 'Napat',     'Suriyan',     '2021-06-25', 3,    'Operations');
SELECT setval('employees_employee_id_seq', 12);

-- orders
INSERT INTO orders (order_id, customer_id, employee_id, order_date, status, ship_country) VALUES
(1,  1,  7, '2024-01-05 09:15:00+07', 'completed', 'Thailand'),
(2,  2,  8, '2024-01-10 11:30:00+07', 'completed', 'Thailand'),
(3,  3,  7, '2024-01-15 14:00:00+07', 'shipped',   'Thailand'),
(4,  4,  9, '2024-01-20 08:45:00-05', 'completed', 'USA'),
(5,  5,  9, '2024-02-02 10:00:00-05', 'cancelled', 'USA'),
(6,  6,  4, '2024-02-05 16:20:00+08', 'completed', 'China'),
(7,  7,  4, '2024-02-10 13:10:00+08', 'shipped',   'China'),
(8,  8,  5, '2024-02-14 09:05:00+07', 'completed', 'Vietnam'),
(9,  9,  7, '2024-02-20 17:40:00+07', 'pending',   'Thailand'),
(10, 10, 8, '2024-03-01 12:00:00+07', 'completed', 'Thailand'),
(11, 11, 9, '2024-03-05 15:25:00-05', 'completed', 'USA'),
(12, 12, 9, '2024-03-10 18:00:00-05', 'shipped',   'USA'),
(13, 1,  7, '2024-03-15 09:30:00+07', 'completed', 'Thailand'),
(14, 13, 5, '2024-03-20 10:10:00+09', 'completed', 'Japan'),
(15, 14, 5, '2024-03-25 11:50:00+09', 'pending',   'Japan'),
(16, 15, 8, '2024-04-01 14:35:00+07', 'completed', 'Thailand');
SELECT setval('orders_order_id_seq', 16);

-- order_items
INSERT INTO order_items (order_item_id, order_id, product_id, quantity, unit_price) VALUES
(1,  1,  1,  2, 350.00),
(2,  1,  7,  1, 1290.00),
(3,  2,  8,  3, 290.00),
(4,  2,  14, 1, 450.00),
(5,  3,  16, 1, 590.00),
(6,  3,  17, 2, 390.00),
(7,  4,  5,  1, 15900.00),
(8,  5,  6,  1, 6900.00),
(9,  6,  2,  1, 1890.00),
(10, 6,  4,  1, 590.00),
(11, 7,  3,  1, 6500.00),
(12, 8,  10, 2, 1290.00),
(13, 8,  11, 1, 650.00),
(14, 9,  18, 2, 590.00),
(15, 10, 19, 1, 2490.00),
(16, 10, 9,  1, 890.00),
(17, 11, 12, 1, 3900.00),
(18, 12, 13, 1, 12500.00),
(19, 13, 15, 1, 690.00),
(20, 13, 1,  1, 350.00),
(21, 14, 7,  2, 1290.00),
(22, 15, 20, 1, 3200.00),
(23, 16, 16, 1, 590.00),
(24, 16, 18, 1, 590.00);
SELECT setval('order_items_order_item_id_seq', 24);

-- reviews
INSERT INTO reviews (review_id, product_id, customer_id, rating, review_text, review_date) VALUES
(1,  1,  1,  5, 'เมาส์ใช้งานลื่นมาก คุ้มราคา', '2024-01-10'),
(2,  7,  1,  4, 'เสียงดีแต่แบตอยู่ได้ไม่นานเท่าที่คิด', '2024-01-12'),
(3,  5,  4,  5, 'สมาร์ทโฟนเรือธงคุ้มค่ามาก กล้องดีสุดๆ', '2024-01-25'),
(4,  6,  5,  2, 'แบตหมดเร็ว ผิดหวังเล็กน้อย', '2024-02-08'),
(5,  2,  6,  5, 'คีย์บอร์ดพิมพ์สนุก เสียงคลิกชัดเจน', '2024-02-09'),
(6,  3,  7,  4, 'จอสวย สีสันคมชัด แต่ขาตั้งโยกเล็กน้อย', '2024-02-15'),
(7,  10, 8,  5, 'ชุดสวยมาก ผ้าดีใส่สบาย', '2024-02-18'),
(8,  19, 10, 3, 'รองเท้าดีแต่ไซส์เล็กไปนิดหน่อย', '2024-03-05'),
(9,  12, 11, 4, 'เก้าอี้นั่งสบาย ปรับระดับได้ดี', '2024-03-08'),
(10, 13, 12, 5, 'โต๊ะแข็งแรงดูดีมาก', '2024-03-14'),
(11, 16, 3,  5, 'หนังสือสอน PostgreSQL เข้าใจง่ายมาก', '2024-01-18'),
(12, 18, 9,  4, 'เสื่อโยคะหนาดีไม่ลื่น', '2024-02-22');
SELECT setval('reviews_review_id_seq', 12);

-- payments (เฉพาะออเดอร์ที่ไม่ใช่ pending/cancelled)
INSERT INTO payments (payment_id, order_id, payment_date, amount, payment_method) VALUES
(1,  1,  '2024-01-05 09:20:00+07', 1990.00,  'credit_card'),
(2,  2,  '2024-01-10 11:35:00+07', 1320.00,  'promptpay'),
(3,  3,  '2024-01-15 14:05:00+07', 1370.00,  'bank_transfer'),
(4,  4,  '2024-01-20 08:50:00-05', 15900.00, 'credit_card'),
(5,  6,  '2024-02-05 16:25:00+08', 2480.00,  'promptpay'),
(6,  7,  '2024-02-10 13:15:00+08', 6500.00,  'credit_card'),
(7,  8,  '2024-02-14 09:10:00+07', 3230.00,  'bank_transfer'),
(8,  10, '2024-03-01 12:05:00+07', 3380.00,  'promptpay'),
(9,  11, '2024-03-05 15:30:00-05', 3900.00,  'credit_card'),
(10, 12, '2024-03-10 18:05:00-05', 12500.00, 'credit_card'),
(11, 13, '2024-03-15 09:35:00+07', 1040.00,  'promptpay'),
(12, 14, '2024-03-20 10:15:00+09', 2580.00,  'credit_card'),
(13, 16, '2024-04-01 14:40:00+07', 1180.00,  'cod');
SELECT setval('payments_payment_id_seq', 13);
```

ตรวจสอบว่าข้อมูลเข้าครบทุกตารางด้วยคำสั่งนี้:

```sql
SELECT 'categories' AS table_name, count(*) FROM categories
UNION ALL SELECT 'suppliers', count(*) FROM suppliers
UNION ALL SELECT 'products', count(*) FROM products
UNION ALL SELECT 'customers', count(*) FROM customers
UNION ALL SELECT 'employees', count(*) FROM employees
UNION ALL SELECT 'orders', count(*) FROM orders
UNION ALL SELECT 'order_items', count(*) FROM order_items
UNION ALL SELECT 'reviews', count(*) FROM reviews
UNION ALL SELECT 'payments', count(*) FROM payments;
```

```
 table_name  | count
-------------+-------
 categories  |    10
 suppliers   |    10
 products    |    20
 customers   |    15
 employees   |    12
 orders      |    16
 order_items |    24
 reviews     |    12
 payments    |    13
(9 rows)
```

> โครงสร้างสำคัญของบทนี้: ตาราง `employees` มีคอลัมน์ `manager_id` ที่อ้างอิงกลับไปยัง `employee_id` ของตารางเดียวกัน ทำให้เกิดความสัมพันธ์แบบ **self-referencing** — พนักงานคนหนึ่งมี "หัวหน้า" ซึ่งก็คือพนักงานอีกแถวหนึ่งในตารางเดียวกันนั่นเอง โครงสร้างนี้คือหัวใจของ SELF JOIN ที่จะเรียนในบทนี้

---

## Step 221: CROSS JOIN — แนวคิด Cartesian Product และ Syntax

จนถึงตอนนี้ JOIN ทุกแบบที่เราเรียนมา (`INNER JOIN`, `LEFT JOIN`, `RIGHT JOIN`, `FULL JOIN`) ล้วนมีเงื่อนไข `ON` เพื่อ "จับคู่" แถวจากสองตารางเข้าด้วยกัน แต่ **`CROSS JOIN`** นั้นแตกต่างออกไปโดยสิ้นเชิง — มันคือการจับคู่ **ทุกแถว** ของตารางซ้าย กับ **ทุกแถว** ของตารางขวา โดยไม่มีเงื่อนไขใด ๆ เลย

ผลลัพธ์ที่ได้เรียกว่า **cartesian product** (ผลคูณคาร์ทีเซียน) — ถ้าตาราง A มี `n` แถว และตาราง B มี `m` แถว ผลลัพธ์ของ `A CROSS JOIN B` จะมี `n × m` แถวเสมอ

### Syntax

```sql
SELECT columns
FROM table_a
CROSS JOIN table_b;
```

หรือเขียนแบบเก่า (implicit cross join) โดยใช้ comma คั่นระหว่างตารางใน `FROM` โดยไม่มี `WHERE` เชื่อมเงื่อนไข:

```sql
SELECT columns
FROM table_a, table_b;
```

ทั้งสองรูปแบบให้ผลลัพธ์เหมือนกันทุกประการ แต่รูปแบบ `CROSS JOIN` ชัดเจนกว่าและเป็นที่นิยมในโค้ดยุคใหม่ เพราะทำให้ผู้อ่านรู้ทันทีว่า "นี่คือ cartesian product ที่ตั้งใจเขียน" ไม่ใช่การลืมใส่เงื่อนไข

### ตัวอย่าง: นับจำนวนแถวที่เกิดขึ้น

```sql
SELECT count(*) AS total_combinations
FROM categories
CROSS JOIN suppliers;
```

```
 total_combinations
---------------------
                 100
(1 row)
```

เพราะ `categories` มี 10 แถว และ `suppliers` มี 10 แถว จึงได้ 10 × 10 = 100 แถว — ทุก "หมวดหมู่" จับคู่กับทุก "ซัพพลายเออร์" แม้ในความเป็นจริงหมวดหมู่นั้นจะไม่เคยสั่งของจากซัพพลายเออร์รายนั้นเลยก็ตาม

### ตัวอย่างที่มองเห็นข้อมูลจริง

```sql
SELECT c.category_name, s.supplier_name
FROM categories c
CROSS JOIN suppliers s
WHERE c.category_id <= 2 AND s.supplier_id <= 3
ORDER BY c.category_name, s.supplier_name;
```

```
 category_name |       supplier_name
---------------+----------------------------
 Computers     | Bangkok Electronics Co.
 Computers     | Global Tech Supply
 Computers     | Shenzhen Gadgets Ltd.
 Electronics   | Bangkok Electronics Co.
 Electronics   | Global Tech Supply
 Electronics   | Shenzhen Gadgets Ltd.
(6 rows)
```

สังเกตว่า 2 หมวดหมู่ × 3 ซัพพลายเออร์ = 6 แถว ครบทุกคู่ที่เป็นไปได้ ไม่ว่าซัพพลายเออร์นั้นจะเคยผลิตสินค้าในหมวดหมู่นั้นจริงหรือไม่

> **ข้อสังเกต:** `CROSS JOIN` ไม่รองรับ clause `ON` หรือ `USING` เพราะไม่มีเงื่อนไขการจับคู่ใด ๆ ถ้าเขียน `CROSS JOIN ... ON ...` PostgreSQL จะรายงาน syntax error ทันที

---

## Step 222: ตัวอย่างการใช้ CROSS JOIN จริง

แม้ `CROSS JOIN` จะดูเหมือนสร้างข้อมูล "ไม่มีความหมาย" แต่ในทางปฏิบัติมันเป็นเครื่องมือสำคัญสำหรับสร้าง **ทุกความเป็นไปได้** (all possible combinations) ซึ่งมีประโยชน์มากในหลายสถานการณ์

### กรณีที่ 1: สร้างตาราง Size × Color สำหรับสินค้า (product variant matrix)

สมมติร้านค้าต้องการออกแบบตัวแปรสินค้า (variant) ของเสื้อผ้าแต่ละไซส์และสี ก่อนที่จะเริ่มผลิตจริง เราสามารถใช้ `CROSS JOIN` ร่วมกับ `VALUES` เพื่อสร้างทุกชุดความเป็นไปได้:

```sql
SELECT
    p.product_name,
    sz.size_label,
    col.color_name
FROM products p
CROSS JOIN (VALUES ('S'), ('M'), ('L'), ('XL')) AS sz(size_label)
CROSS JOIN (VALUES ('Black'), ('White'), ('Navy')) AS col(color_name)
WHERE p.category_id IN (5, 6)   -- เฉพาะหมวดเสื้อผ้า
ORDER BY p.product_name, sz.size_label, col.color_name;
```

```
      product_name      | size_label | color_name
-------------------------+------------+------------
 Men's Cotton T-Shirt    | L          | Black
 Men's Cotton T-Shirt    | L          | Navy
 Men's Cotton T-Shirt    | L          | White
 Men's Cotton T-Shirt    | M          | Black
 Men's Cotton T-Shirt    | M          | Navy
 Men's Cotton T-Shirt    | M          | White
 ...
(48 rows)
```

จากสินค้าเสื้อผ้า 4 รายการ (category_id 5,6) × ไซส์ 4 แบบ × สี 3 แบบ = 48 แถว — ครบทุกตัวแปรที่ทีมสินค้าอาจต้องพิจารณาผลิต โดยไม่ต้องเขียน `INSERT` เองทีละแถว

### กรณีที่ 2: สร้างปฏิทิน (calendar) ด้วย generate_series + CROSS JOIN

`generate_series()` เป็นฟังก์ชันที่คืนค่าเป็น "ตาราง" ของลำดับตัวเลขหรือวันที่ เมื่อใช้ร่วมกับ `CROSS JOIN` เราสามารถสร้างรายงานแบบ "ทุกพนักงาน × ทุกวัน" หรือ "ทุกสินค้า × ทุกเดือน" ได้ทันที ซึ่งมีประโยชน์มากสำหรับการทำรายงานที่ต้องการเห็น "ช่องว่าง" (เช่น วันที่ไม่มีออเดอร์เลย)

```sql
SELECT
    d::date AS report_date,
    e.first_name || ' ' || e.last_name AS employee_name
FROM generate_series('2024-01-01'::date, '2024-01-05'::date, interval '1 day') AS d
CROSS JOIN employees e
WHERE e.department = 'Sales'
ORDER BY d, employee_name;
```

```
 report_date |   employee_name
-------------+---------------------
 2024-01-01  | Duangjai Meesuk
 2024-01-01  | Kanya Suksawat
 2024-01-01  | Nattapong Srisawat
 2024-01-01  | Ratana Chaiyaporn
 2024-01-01  | Somkiat Thaweesak
 2024-01-01  | Wichai Kittisak
 2024-01-02  | Duangjai Meesuk
 ...
(30 rows)
```

ผลลัพธ์นี้คือ "โครงกระดูก" ของรายงานการเข้างานหรือยอดขายรายวัน — 5 วัน × 6 พนักงานฝ่ายขาย = 30 แถว ครบทุกวันสำหรับทุกคน แม้บางวันพนักงานคนนั้นจะไม่มีออเดอร์เลยก็ตาม ขั้นต่อไปมักจะ `LEFT JOIN` ตารางนี้เข้ากับ `orders` เพื่อดูว่าวันไหน "ว่างงาน" บ้าง:

```sql
SELECT
    d::date AS report_date,
    e.first_name || ' ' || e.last_name AS employee_name,
    count(o.order_id) AS order_count
FROM generate_series('2024-01-01'::date, '2024-01-20'::date, interval '1 day') AS d
CROSS JOIN employees e
LEFT JOIN orders o
    ON o.employee_id = e.employee_id
    AND o.order_date::date = d
WHERE e.department = 'Sales'
GROUP BY d, e.employee_id, e.first_name, e.last_name
ORDER BY d, employee_name
LIMIT 8;
```

```
 report_date |   employee_name    | order_count
-------------+---------------------+-------------
 2024-01-01  | Duangjai Meesuk     |           0
 2024-01-01  | Kanya Suksawat      |           0
 2024-01-01  | Nattapong Srisawat  |           0
 2024-01-01  | Ratana Chaiyaporn   |           0
 2024-01-01  | Somkiat Thaweesak   |           0
 2024-01-01  | Wichai Kittisak     |           0
 2024-01-02  | Duangjai Meesuk     |           0
 2024-01-02  | Kanya Suksawat      |           0
(8 rows)
```

เทคนิคนี้เป็นวิธีมาตรฐานในการทำรายงานที่ต้อง "เติมช่องว่าง" (fill gaps) เพื่อไม่ให้วันที่ไม่มีข้อมูลหายไปจากรายงาน ซึ่งเป็นปัญหาที่พบบ่อยมากเมื่อใช้ `GROUP BY` ตรง ๆ กับข้อมูลที่ไม่ครบทุกวัน

### กรณีที่ 3: จับคู่หมวดหมู่กับช่วงราคาเพื่อวางแผนโปรโมชัน

```sql
SELECT
    c.category_name,
    pr.price_range
FROM categories c
CROSS JOIN (VALUES ('0-500'), ('501-2000'), ('2001+')) AS pr(price_range)
WHERE c.parent_category_id IS NULL
ORDER BY c.category_name, pr.price_range;
```

```
   category_name   | price_range
--------------------+-------------
 Books              | 0-500
 Books              | 2001+
 Books              | 501-2000
 Clothing           | 0-500
 Clothing           | 2001+
 Clothing           | 501-2000
 Electronics        | 0-500
 Electronics        | 2001+
 Electronics        | 501-2000
 Home & Kitchen      | 0-500
 Home & Kitchen      | 2001+
 Home & Kitchen      | 501-2000
 Sports & Outdoors   | 0-500
 Sports & Outdoors   | 2001+
 Sports & Outdoors   | 501-2000
(15 rows)
```

ผลลัพธ์นี้ใช้เป็นแม่แบบ (template) สำหรับ dashboard วางแผนโปรโมชันแบบ matrix — ทีมการตลาดจะเห็นทุกช่อง "หมวดหมู่ × ช่วงราคา" แม้บางช่องจะยังไม่มีสินค้าจริงอยู่ก็ตาม ซึ่งช่วยให้มองเห็น "ช่องว่างของสินค้า" (product gap) ได้ง่าย

---

## Step 223: อันตรายของ CROSS JOIN โดยไม่ตั้งใจ

ปัญหาที่พบบ่อยที่สุดของมือใหม่ (และบางครั้งมือเก๋า) คือการเขียน JOIN โดย **ลืมใส่เงื่อนไข `ON`** หรือเขียนเงื่อนไขผิดจนกลายเป็นจริงเสมอ (`ON true` โดยไม่ตั้งใจ) ทำให้ `INNER JOIN` ที่ตั้งใจไว้กลายเป็น `CROSS JOIN` แบบไม่รู้ตัว

### ตัวอย่างข้อผิดพลาดที่พบบ่อย

สมมติต้องการดูรายการสินค้าที่เคยถูกสั่งซื้อ พร้อมชื่อลูกค้า แต่พิมพ์ query ผิดโดยลืม `JOIN` ตัวกลาง (`order_items`) และเขียนแบบ implicit join ผสมกัน:

```sql
-- ผิด! ลืมเงื่อนไขเชื่อม orders กับ order_items
SELECT
    c.first_name,
    p.product_name
FROM customers c, orders o, order_items oi, products p
WHERE c.customer_id = o.customer_id
  AND oi.product_id = p.product_id;
  -- ลืมบรรทัด: AND oi.order_id = o.order_id
```

```sql
SELECT count(*) FROM (
    SELECT c.first_name, p.product_name
    FROM customers c, orders o, order_items oi, products p
    WHERE c.customer_id = o.customer_id
      AND oi.product_id = p.product_id
) AS mistake;
```

```
 count
-------
   384
(1 row)
```

เทียบกับ query ที่ถูกต้อง:

```sql
SELECT count(*) FROM (
    SELECT c.first_name, p.product_name
    FROM customers c
    JOIN orders o      ON c.customer_id = o.customer_id
    JOIN order_items oi ON o.order_id = oi.order_id
    JOIN products p     ON oi.product_id = p.product_id
) AS correct;
```

```
 count
-------
    24
(1 row)
```

ผลลัพธ์ต่างกันถึง 16 เท่า (384 เทียบกับ 24) — เพราะ query ที่ผิดขาดเงื่อนไข `oi.order_id = o.order_id` ทำให้ `orders` กับ `order_items` จับคู่กันแบบ cartesian product (16 orders × 24 order_items บางส่วน) ซึ่งเป็นข้อมูลที่ "ผิดโดยสิ้นเชิง" แต่ query กลับรันผ่านโดยไม่มี error ใด ๆ เลย เพราะทาง syntax มันถูกต้องสมบูรณ์แบบ — นี่คือความอันตรายที่แท้จริงของ CROSS JOIN โดยไม่ตั้งใจ

### วิธีป้องกัน

1. **หลีกเลี่ยง implicit join** (comma-separated FROM) ให้ใช้ `JOIN ... ON ...` แบบ explicit เสมอ เพราะบังคับให้ต้องเขียนเงื่อนไข
2. **ตรวจสอบจำนวนแถวที่คาดหวัง** ก่อนและหลังเขียน query เสมอ เช่น ถ้ารู้ว่ามี order_items 24 แถว ผลลัพธ์ที่ join กับ orders/products ไม่ควรเกิน 24 แถว (เว้นแต่จะตั้งใจขยายด้วยเหตุผลอื่น)
3. **ใช้ EXPLAIN** ดูจำนวนแถวโดยประมาณ (`rows=...`) ก่อนรัน query จริงกับข้อมูลขนาดใหญ่ ถ้าตัวเลขสูงผิดปกติ (เช่น หลักล้านทั้งที่ตารางมีหลักพัน) ให้สงสัยว่าอาจเกิด cross join โดยไม่ตั้งใจ
4. **เปิด linter หรือ code review** ที่เตือนเมื่อพบ `JOIN` โดยไม่มี `ON`/`USING` (ยกเว้นตั้งใจเขียน `CROSS JOIN` อย่างชัดเจน)

```sql
EXPLAIN SELECT c.first_name, p.product_name
FROM customers c, orders o, order_items oi, products p
WHERE c.customer_id = o.customer_id
  AND oi.product_id = p.product_id;
```

```
                                QUERY PLAN
--------------------------------------------------------------------------
 Nested Loop  (cost=... rows=384 width=...)
   ...
(varies)
```

สังเกตคำว่า `rows=384` ใน plan — ถ้าเราคุ้นเคยกับขนาดข้อมูลจริง (order_items มีแค่ 24 แถว) ตัวเลข 384 ควรทำให้เรา "สะดุด" ทันทีว่ามีบางอย่างผิดปกติ

> **หลักการสำคัญ:** ทุกครั้งที่เขียน `JOIN` ระหว่างสองตาราง ให้ถามตัวเองเสมอว่า "คอลัมน์ไหนคือกุญแจที่เชื่อมสองตารางนี้เข้าด้วยกัน" ถ้าตอบไม่ได้ชัดเจน หรือคิดว่าไม่ต้องมีเงื่อนไข ให้สงสัยไว้ก่อนว่าอาจกำลังเขียน CROSS JOIN โดยไม่ได้ตั้งใจ

---

## Step 224: SELF JOIN — แนวคิดการ JOIN ตารางกับตัวเอง และทำไมต้องใช้ Alias

**SELF JOIN** ไม่ใช่ JOIN ประเภทใหม่ทาง syntax แต่เป็น **เทคนิคการใช้งาน** — คือการ JOIN ตารางกับ **ตัวมันเอง** โดยอาจใช้ `INNER JOIN`, `LEFT JOIN` หรือ JOIN แบบใดก็ได้ตามปกติ เพียงแต่ตารางทั้งสองฝั่งของ `ON` คือตารางเดียวกัน

เหตุผลที่ต้องทำ SELF JOIN มักเกิดจากการที่ตารางมี **ความสัมพันธ์กับตัวเอง** (self-referencing relationship) เช่น:

- พนักงานคนหนึ่งมี "หัวหน้า" ซึ่งก็คือพนักงานอีกคนในตารางเดียวกัน (`employees.manager_id → employees.employee_id`)
- หมวดหมู่สินค้ามี "หมวดหมู่แม่" ซึ่งก็คือหมวดหมู่อีกแถวในตารางเดียวกัน (`categories.parent_category_id → categories.category_id`)
- หรือแม้แต่การเปรียบเทียบแถวสองแถวในตารางเดียวกันที่ไม่มีความสัมพันธ์ foreign key ชัดเจน เช่น หาสินค้าสองรายการที่อยู่หมวดหมู่เดียวกัน

### ทำไมต้องใช้ Alias เสมอ

เมื่อ query อ้างถึงตารางเดียวกันสองครั้งใน `FROM`/`JOIN` PostgreSQL **จำเป็นต้อง** แยกแยะว่าคอลัมน์ที่เขียนถึงนั้นมาจาก "ตัวตนไหน" ของตาราง เพราะถ้าเขียน `employees.employee_id` โดยไม่ระบุ alias ระบบจะไม่รู้ว่าหมายถึง employees ตัวไหนจากสองตัวที่ถูกอ้างถึง — จึงต้อง **ตั้งชื่อ alias ที่แตกต่างกัน** ให้กับตารางทั้งสองฝั่งเสมอ

```sql
-- ผิด! ไม่ระบุ alias จะเกิด error หรือ ambiguous
SELECT employee_id, manager_id
FROM employees
JOIN employees ON employees.manager_id = employees.employee_id;
```

รันคำสั่งนี้ PostgreSQL จะแจ้ง:

```
ERROR:  table name "employees" specified more than once
```

วิธีที่ถูกต้องคือตั้ง alias ให้ตารางทั้งสองฝั่งมีชื่อไม่ซ้ำกัน เช่น `e` (สำหรับพนักงาน) และ `m` (สำหรับหัวหน้า):

```sql
SELECT
    e.employee_id,
    e.first_name AS employee_first_name,
    m.employee_id AS manager_id,
    m.first_name AS manager_first_name
FROM employees e
JOIN employees m ON e.manager_id = m.employee_id
ORDER BY e.employee_id
LIMIT 5;
```

```
 employee_id | employee_first_name | manager_id | manager_first_name
-------------+----------------------+------------+---------------------
           2 | Kanya                |          1 | Prasert
           3 | Anan                 |          1 | Prasert
           4 | Nattapong            |          2 | Kanya
           5 | Ratana               |          2 | Kanya
           6 | Siriporn             |          3 | Anan
(5 rows)
```

สังเกตว่า `employees e` คือ "ตัวตนแรก" ที่เราถือว่าเป็นพนักงาน และ `employees m` คือ "ตัวตนที่สอง" ที่เราถือว่าเป็นหัวหน้า — แม้ในความเป็นจริงทั้งคู่คือแถวจากตาราง `employees` ตารางเดียวกันทั้งหมด เพียงแต่เรา "สวมบทบาท" ให้มันสองแบบต่างกันในมุมมองของ query นี้

> **แนวคิดสำคัญ:** ให้จินตนาการว่า SELF JOIN คือการ "ถ่ายเอกสารตาราง" ออกมาสองชุด แล้วนำมาวางเทียบกัน — ชุดหนึ่งมองในมุม "พนักงาน" อีกชุดหนึ่งมองในมุม "หัวหน้า" ทั้งที่ข้อมูลจริงมาจากตารางเดียวกัน

---

## Step 225: ตัวอย่าง SELF JOIN — หาพนักงานกับหัวหน้าของตัวเอง

มาดูตัวอย่างแบบเต็มรูปแบบของการหาพนักงานทุกคนพร้อมชื่อหัวหน้า

### INNER JOIN: แสดงเฉพาะพนักงานที่มีหัวหน้า

```sql
SELECT
    e.first_name || ' ' || e.last_name AS employee_name,
    e.department,
    m.first_name || ' ' || m.last_name AS manager_name
FROM employees e
JOIN employees m ON e.manager_id = m.employee_id
ORDER BY e.employee_id;
```

```
     employee_name      | department  |    manager_name
-------------------------+-------------+---------------------
 Kanya Suksawat          | Sales       | Prasert Wongsawat
 Anan Phetchara          | Operations  | Prasert Wongsawat
 Nattapong Srisawat      | Sales       | Kanya Suksawat
 Ratana Chaiyaporn       | Sales       | Kanya Suksawat
 Siriporn Boonyarit      | Operations  | Anan Phetchara
 Wichai Kittisak         | Sales       | Nattapong Srisawat
 Duangjai Meesuk         | Sales       | Nattapong Srisawat
 Somkiat Thaweesak       | Sales       | Ratana Chaiyaporn
 Panida Jansuk           | Operations  | Siriporn Boonyarit
 Chatchai Rungrueang     | Operations  | Siriporn Boonyarit
 Napat Suriyan           | Operations  | Anan Phetchara
(11 rows)
```

สังเกตว่ามีเพียง 11 แถว ไม่ใช่ 12 แถว — เพราะ `Prasert Wongsawat` (CEO) มี `manager_id IS NULL` ไม่มีหัวหน้า จึงไม่ถูกจับคู่ได้ใน `INNER JOIN`

### LEFT JOIN: แสดงพนักงานทุกคน แม้ไม่มีหัวหน้า (เช่น CEO)

ถ้าต้องการเห็นพนักงาน**ทุกคน**รวมถึงคนที่ไม่มีหัวหน้าด้วย ต้องเปลี่ยนเป็น `LEFT JOIN`:

```sql
SELECT
    e.first_name || ' ' || e.last_name AS employee_name,
    e.department,
    COALESCE(m.first_name || ' ' || m.last_name, '(ไม่มีหัวหน้า — ตำแหน่งสูงสุด)') AS manager_name
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.employee_id
ORDER BY e.employee_id;
```

```
     employee_name      | department  |            manager_name
-------------------------+-------------+---------------------------------------
 Prasert Wongsawat       | Executive   | (ไม่มีหัวหน้า — ตำแหน่งสูงสุด)
 Kanya Suksawat          | Sales       | Prasert Wongsawat
 Anan Phetchara          | Operations  | Prasert Wongsawat
 Nattapong Srisawat      | Sales       | Kanya Suksawat
 Ratana Chaiyaporn       | Sales       | Kanya Suksawat
 Siriporn Boonyarit      | Operations  | Anan Phetchara
 Wichai Kittisak         | Sales       | Nattapong Srisawat
 Duangjai Meesuk         | Sales       | Nattapong Srisawat
 Somkiat Thaweesak       | Sales       | Ratana Chaiyaporn
 Panida Jansuk           | Operations  | Siriporn Boonyarit
 Chatchai Rungrueang     | Operations  | Siriporn Boonyarit
 Napat Suriyan           | Operations  | Anan Phetchara
(12 rows)
```

ตอนนี้ได้ครบ 12 แถวทุกคน — นี่คือความแตกต่างสำคัญระหว่าง `INNER JOIN` และ `LEFT JOIN` เมื่อใช้กับ SELF JOIN: `LEFT JOIN` รับประกันว่าตาราง "ต้นทาง" (e) จะปรากฏครบทุกแถว แม้จะไม่มีคู่ในฝั่ง "หัวหน้า" (m) ก็ตาม

### หาว่าใครเป็นหัวหน้าของใครบ้าง (reverse: หัวหน้า → ลูกทีม)

```sql
SELECT
    m.first_name || ' ' || m.last_name AS manager_name,
    count(e.employee_id) AS team_size,
    string_agg(e.first_name, ', ' ORDER BY e.first_name) AS team_members
FROM employees m
JOIN employees e ON e.manager_id = m.employee_id
GROUP BY m.employee_id, m.first_name, m.last_name
ORDER BY team_size DESC;
```

```
    manager_name     | team_size |           team_members
----------------------+-----------+------------------------------------
 Anan Phetchara       |         2 | Napat, Siriporn
 Kanya Suksawat       |         2 | Nattapong, Ratana
 Nattapong Srisawat   |         2 | Duangjai, Wichai
 Prasert Wongsawat    |         2 | Anan, Kanya
 Siriporn Boonyarit   |         2 | Chatchai, Panida
 Ratana Chaiyaporn    |         1 | Somkiat
(6 rows)
```

query นี้แสดงให้เห็นว่า SELF JOIN สามารถใช้ร่วมกับ `GROUP BY` และ aggregate function ได้ตามปกติทุกประการ เหมือน JOIN ระหว่างสองตารางที่แตกต่างกัน

### หาสายบังคับบัญชา 2 ระดับ (grandparent-style self join)

เราสามารถ SELF JOIN ตารางเดียวกัน**มากกว่าสองครั้ง** เพื่อไล่ลำดับชั้นได้ลึกขึ้น เช่น หา "หัวหน้าของหัวหน้า" (ระดับ 2):

```sql
SELECT
    e.first_name AS employee,
    m1.first_name AS direct_manager,
    m2.first_name AS skip_level_manager
FROM employees e
JOIN employees m1 ON e.manager_id = m1.employee_id
LEFT JOIN employees m2 ON m1.manager_id = m2.employee_id
ORDER BY e.employee_id;
```

```
  employee   | direct_manager | skip_level_manager
-------------+-----------------+---------------------
 Kanya       | Prasert         |
 Anan        | Prasert         |
 Nattapong   | Kanya           | Prasert
 Ratana      | Kanya           | Prasert
 Siriporn    | Anan            | Prasert
 Wichai      | Nattapong       | Kanya
 Duangjai    | Nattapong       | Kanya
 Somkiat     | Ratana          | Kanya
 Panida      | Siriporn        | Anan
 Chatchai    | Siriporn        | Anan
 Napat       | Anan            | Prasert
(11 rows)
```

ที่นี่เราใช้ตาราง `employees` ถึง 3 ครั้งในคำสั่งเดียว (`e`, `m1`, `m2`) แต่ละ alias เป็น "มุมมอง" ที่แตกต่างกันของข้อมูลชุดเดียวกัน — เทคนิคนี้ยังจำกัดจำนวนระดับที่ต้องไล่ล่วงหน้า สำหรับลำดับชั้นที่ลึกไม่จำกัดจำนวนระดับ จะต้องใช้ **recursive CTE** (`WITH RECURSIVE`) ซึ่งจะเรียนในบทถัดไปของหลักสูตร

---

## Step 226: ตัวอย่าง SELF JOIN เพิ่มเติม

SELF JOIN ไม่ได้ใช้ได้แค่กับความสัมพันธ์แบบ foreign key ที่ชัดเจนอย่าง manager_id เท่านั้น แต่ยังใช้ **เปรียบเทียบแถวสองแถวในตารางเดียวกัน** ตามเงื่อนไขทางธุรกิจใด ๆ ก็ได้

### กรณีที่ 1: เปรียบเทียบราคาสินค้าในหมวดหมู่เดียวกัน

สมมติต้องการหา "สินค้าทางเลือกที่ถูกกว่า" ในหมวดหมู่เดียวกัน เพื่อแนะนำลูกค้าที่กำลังดูสินค้าราคาแพง:

```sql
SELECT
    p1.product_name AS expensive_product,
    p1.unit_price AS expensive_price,
    p2.product_name AS cheaper_alternative,
    p2.unit_price AS cheaper_price,
    p1.unit_price - p2.unit_price AS price_difference
FROM products p1
JOIN products p2
    ON p1.category_id = p2.category_id
    AND p1.product_id <> p2.product_id
    AND p2.unit_price < p1.unit_price
WHERE p1.category_id = 2   -- Computers
ORDER BY p1.product_name, price_difference;
```

```
 expensive_product  | expensive_price | cheaper_alternative | cheaper_price | price_difference
---------------------+------------------+----------------------+----------------+-------------------
 27-inch Monitor     |          6500.00 | Laptop Stand         |         590.00 |           5910.00
 27-inch Monitor     |          6500.00 | Wireless Mouse       |         350.00 |           6150.00
 27-inch Monitor     |          6500.00 | Mechanical Keyboard  |        1890.00 |           4610.00
 Mechanical Keyboard |          1890.00 | Laptop Stand         |         590.00 |           1300.00
 Mechanical Keyboard |          1890.00 | Wireless Mouse       |         350.00 |           1540.00
 Laptop Stand        |           590.00 | Wireless Mouse       |         350.00 |            240.00
(6 rows)
```

เงื่อนไข `p1.product_id <> p2.product_id` **สำคัญมาก** — ถ้าลืมใส่ สินค้าจะถูกจับคู่กับตัวเอง (price_difference = 0) ซึ่งไม่มีประโยชน์และทำให้ผลลัพธ์รก และเงื่อนไข `p2.unit_price < p1.unit_price` ทำให้เราไม่เห็นคู่ซ้ำ (A แพงกว่า B และ B แพงกว่า A พร้อมกัน)

### กรณีที่ 2: หาลูกค้าที่อยู่ประเทศเดียวกัน (จับคู่ไม่ซ้ำ)

ต้องการหาคู่ลูกค้าที่อยู่ประเทศเดียวกัน เพื่อส่งแคมเปญ "แนะนำเพื่อน" ในประเทศเดียวกัน:

```sql
SELECT
    c1.first_name || ' ' || c1.last_name AS customer_a,
    c2.first_name || ' ' || c2.last_name AS customer_b,
    c1.country
FROM customers c1
JOIN customers c2
    ON c1.country = c2.country
    AND c1.customer_id < c2.customer_id   -- ป้องกันคู่ซ้ำและจับคู่กับตัวเอง
ORDER BY c1.country, customer_a, customer_b
LIMIT 10;
```

```
      customer_a       |      customer_b       | country
------------------------+------------------------+----------
 Wei Chen               | Li Na                  | China
 John Smith             | Emily Johnson          | USA
 John Smith             | Sarah Williams         | USA
 John Smith             | Michael Brown          | USA
 Emily Johnson          | Sarah Williams         | USA
 Emily Johnson          | Michael Brown          | USA
 Sarah Williams         | Michael Brown          | USA
 Nguyen Van A            | (no match)            |
 Yui Tanaka             | Kenji Sato             | Japan
 Somchai Jaidee         | Suda Thongdee          | Thailand
(10 rows)
```

> **หมายเหตุเรื่องเทคนิค `customer_id < customer_id`:** นี่คือลูกเล่นมาตรฐานเวลาต้องการ "จับคู่ไม่ซ้ำแบบไม่มีทิศทาง" (unordered pair) จากตารางเดียวกัน — ถ้าใช้ `<>` (ไม่เท่ากัน) เฉย ๆ จะได้คู่ซ้ำสองทาง (A-B และ B-A) ซึ่งความหมายเหมือนกันแต่นับซ้ำ การใช้ `<` แทนทำให้ได้แค่ทิศทางเดียว ลดผลลัพธ์ลงครึ่งหนึ่งพอดี

### กรณีที่ 3: หาสินค้าที่มีซัพพลายเออร์เดียวกันแต่คนละหมวดหมู่

```sql
SELECT
    p1.product_name AS product_a,
    p2.product_name AS product_b,
    s.supplier_name
FROM products p1
JOIN products p2
    ON p1.supplier_id = p2.supplier_id
    AND p1.product_id < p2.product_id
    AND p1.category_id <> p2.category_id
JOIN suppliers s ON s.supplier_id = p1.supplier_id
ORDER BY s.supplier_name;
```

```
       product_a       |       product_b        |     supplier_name
------------------------+-------------------------+-------------------------
 Wireless Earbuds       | Smartphone X12          | Bangkok Electronics Co.
 Office Chair           | Dining Table Set        | Nordic Furniture House
 Men's Denim Jeans      | Women's Summer Dress    | Fashion Forward Inc.
(3 rows)
```

ตัวอย่างนี้ผสม SELF JOIN (`products p1` กับ `products p2`) เข้ากับ JOIN ปกติ (`suppliers s`) ในคำสั่งเดียว — แสดงให้เห็นว่า SELF JOIN สามารถอยู่ร่วมกับ JOIN ประเภทอื่นได้อย่างอิสระ ไม่มีข้อจำกัดพิเศษใด ๆ

### กรณีที่ 4: หาพนักงานที่อยู่แผนกเดียวกันและเข้าทำงานในปีเดียวกัน

```sql
SELECT
    e1.first_name AS employee_a,
    e2.first_name AS employee_b,
    e1.department,
    EXTRACT(YEAR FROM e1.hire_date) AS hire_year
FROM employees e1
JOIN employees e2
    ON e1.department = e2.department
    AND e1.employee_id < e2.employee_id
    AND EXTRACT(YEAR FROM e1.hire_date) = EXTRACT(YEAR FROM e2.hire_date)
ORDER BY hire_year, e1.department;
```

```
 employee_a | employee_b | department  | hire_year
-------------+-------------+-------------+-----------
 Wichai      | Duangjai    | Sales       |      2020
(1 row)
```

query นี้แสดงให้เห็นว่าเงื่อนไข SELF JOIN สามารถผสมหลายเงื่อนไขพร้อมกันได้ (แผนกเดียวกัน **และ** ปีเข้าทำงานเดียวกัน) เหมือนการ JOIN ปกติทุกประการ

---

## Step 227: NATURAL JOIN — Syntax และการทำงานอัตโนมัติ

**`NATURAL JOIN`** เป็น JOIN ที่ PostgreSQL จะ **จับคู่คอลัมน์ให้อัตโนมัติ** โดยดูจาก **ชื่อคอลัมน์ที่ตรงกันทุกตัว** ระหว่างสองตาราง โดยไม่ต้องเขียน `ON` หรือ `USING` เลย

### Syntax

```sql
SELECT columns
FROM table_a
NATURAL JOIN table_b;
```

PostgreSQL จะทำงานดังนี้:
1. หาคอลัมน์ที่มี **ชื่อเดียวกัน** ปรากฏอยู่ในทั้งสองตาราง
2. ใช้คอลัมน์เหล่านั้นทั้งหมดเป็นเงื่อนไข JOIN โดยอัตโนมัติ (เทียบเท่ากับ `USING (col1, col2, ...)`)
3. ถ้าไม่มีคอลัมน์ชื่อตรงกันเลย ผลลัพธ์จะกลายเป็น `CROSS JOIN` โดยปริยาย (ไม่มี error แต่ผลลัพธ์อาจไม่ใช่สิ่งที่ต้องการ)

### ตัวอย่าง

พิจารณาตาราง `orders` และ `customers` — ทั้งสองตารางมีคอลัมน์ชื่อ `customer_id` ตรงกัน:

```sql
SELECT
    order_id,
    first_name,
    last_name,
    order_date,
    status
FROM orders
NATURAL JOIN customers
ORDER BY order_id
LIMIT 5;
```

```
 order_id | first_name | last_name |      order_date       |  status
----------+-------------+------------+------------------------+-----------
        1 | Somchai     | Jaidee     | 2024-01-05 09:15:00+07 | completed
        2 | Suda        | Thongdee   | 2024-01-10 11:35:00+07 | completed
        3 | Anong       | Srisuk     | 2024-01-15 14:00:00+07 | shipped
        4 | John        | Smith      | 2024-01-20 20:50:00+07 | completed
        5 | Emily       | Johnson    | 2024-02-02 23:00:00+07 | cancelled
(5 rows)
```

`NATURAL JOIN` ที่ใช้ที่นี่เทียบเท่ากับ:

```sql
SELECT
    order_id,
    first_name,
    last_name,
    order_date,
    status
FROM orders o
JOIN customers c USING (customer_id)
ORDER BY order_id
LIMIT 5;
```

ให้ผลลัพธ์เหมือนกันทุกประการ เพราะคอลัมน์ที่ชื่อตรงกันระหว่าง `orders` และ `customers` มีเพียง `customer_id` เท่านั้น (ส่วน `orders.ship_country` กับ `customers.country` ชื่อไม่ตรงกัน จึงไม่ถูกนำมาใช้เป็นเงื่อนไข)

### ตรวจสอบว่าคอลัมน์ไหนบ้างที่ตรงกัน

```sql
SELECT column_name
FROM information_schema.columns
WHERE table_name = 'orders'
INTERSECT
SELECT column_name
FROM information_schema.columns
WHERE table_name = 'customers';
```

```
 column_name
--------------
 customer_id
(1 row)
```

query นี้ยืนยันว่ามีคอลัมน์ชื่อตรงกันเพียงตัวเดียวคือ `customer_id` — นี่คือสิ่งที่ `NATURAL JOIN` จะใช้เป็นเงื่อนไข JOIN โดยอัตโนมัติ เป็นเทคนิคที่มีประโยชน์เวลาต้องการตรวจสอบพฤติกรรมของ `NATURAL JOIN` ก่อนใช้งานจริง (ถ้าจำเป็นต้องใช้เลย)

---

## Step 228: อันตรายของ NATURAL JOIN ในทางปฏิบัติ

แม้ `NATURAL JOIN` จะดูสะดวกและพิมพ์สั้น แต่ในวงการมืออาชีพ **แทบไม่มีใครใช้ในโค้ด production เลย** เหตุผลหลักคือมันขึ้นอยู่กับ "ชื่อคอลัมน์" ซึ่งเป็นสิ่งที่เปลี่ยนแปลงได้ตลอดเวลา และเมื่อเปลี่ยน พฤติกรรมของ query จะเปลี่ยนไปโดย **ไม่มีการแจ้งเตือนใด ๆ**

### สาธิต: เปลี่ยนชื่อคอลัมน์แล้วพฤติกรรมเปลี่ยนโดยไม่รู้ตัว

สมมติทีมพัฒนาตัดสินใจเปลี่ยนชื่อคอลัมน์ `orders.ship_country` เป็น `country` เพื่อให้ตรงกับชื่อในตารางอื่น ๆ (ดูเหมือนเป็นการ refactor ที่ไม่มีพิษภัย):

```sql
ALTER TABLE orders RENAME COLUMN ship_country TO country;
```

ตอนนี้ลองรัน `NATURAL JOIN` แบบเดิมอีกครั้ง:

```sql
SELECT
    order_id,
    first_name,
    last_name,
    order_date,
    status
FROM orders
NATURAL JOIN customers
ORDER BY order_id;
```

```
 order_id | first_name | last_name |      order_date       |  status
----------+-------------+------------+------------------------+-----------
        1 | Somchai     | Jaidee     | 2024-01-05 09:15:00+07 | completed
        2 | Suda        | Thongdee   | 2024-01-10 11:35:00+07 | completed
        3 | Anong       | Srisuk     | 2024-01-15 14:00:00+07 | shipped
       10 | Piti        | Boonmee    | 2024-03-01 12:00:00+07 | completed
       16 | Malee       | Rakthai    | 2024-04-01 14:35:00+07 | completed
(5 rows)
```

**สังเกตความแตกต่าง!** ผลลัพธ์เหลือแค่ 5 แถวจากเดิม 16 แถว (แสดงเฉพาะบางส่วน) เพราะตอนนี้ `NATURAL JOIN` มองเห็นคอลัมน์ชื่อตรงกัน **สองตัว**: `customer_id` **และ** `country` — มันจึงเปลี่ยนเงื่อนไข JOIN เป็น `orders.customer_id = customers.customer_id AND orders.country = customers.country` โดยอัตโนมัติ ซึ่งเป็นตรรกะที่ **ผิดทางธุรกิจโดยสิ้นเชิง** เพราะ `orders.country` (ตอนนี้) หมายถึง "ประเทศที่จัดส่งสินค้า" ในขณะที่ `customers.country` หมายถึง "ประเทศที่ลูกค้าสมัครสมาชิก" — สองความหมายที่แตกต่างกันโดยสิ้นเชิง แต่ถูกบังคับให้ต้องเท่ากันโดยไม่มีใครตั้งใจ ออเดอร์ใดที่ลูกค้าสั่งของไปส่งต่างประเทศ (เช่น ลูกค้าอยู่ USA แต่ส่งของไปญี่ปุ่น) จะหายไปจากผลลัพธ์ทันที **โดยไม่มี error หรือคำเตือนใด ๆ เลย**

เรามา rename กลับเพื่อคืนสภาพ schema ให้ถูกต้องตามเดิม:

```sql
ALTER TABLE orders RENAME COLUMN country TO ship_country;
```

### สรุปเหตุผลที่มืออาชีพหลีกเลี่ยง NATURAL JOIN

| ปัญหา | รายละเอียด |
|---|---|
| **Silent behavior change** | เพียงแค่เปลี่ยนชื่อคอลัมน์ในตารางใดตารางหนึ่ง (แม้จะไม่เกี่ยวข้องกับ query นี้โดยตรง) ก็เปลี่ยนผลลัพธ์ของ query ได้ทันที โดยไม่มี error |
| **ไม่ชัดเจนสำหรับผู้อ่านโค้ด** | ผู้อ่าน query ต้องไปเปิดดู schema ของทั้งสองตารางเพื่อรู้ว่า JOIN เกิดขึ้นบนคอลัมน์ใดบ้าง ต่างจาก `ON`/`USING` ที่บอกชัดเจนในตัว query เอง |
| **เพิ่มคอลัมน์ใหม่แล้วพัง** | ถ้ามีคนเพิ่มคอลัมน์ใหม่ในตารางหนึ่ง แล้วบังเอิญชื่อไปตรงกับคอลัมน์ในอีกตาราง (เช่น `created_at`, `updated_at`, `notes` ที่หลายตารางมักมีชื่อซ้ำกัน) เงื่อนไข JOIN จะเปลี่ยนไปทันทีโดยไม่มีใครตั้งใจ |
| **Migration เสี่ยงสูง** | ทีมที่ทำ database migration หรือ refactor ต้องตรวจสอบทุก query ที่ใช้ `NATURAL JOIN` ทั่วทั้งระบบก่อนเปลี่ยนชื่อคอลัมน์ใด ๆ ซึ่งเป็นภาระที่ไม่จำเป็น |
| **ทดสอบยาก** | เพราะพฤติกรรมขึ้นกับ schema ทั้งหมด ไม่ใช่แค่ query เดียว การเขียน unit test ให้ครอบคลุมทุกกรณีทำได้ยากกว่า JOIN แบบระบุเงื่อนไขชัดเจน |

### แนวปฏิบัติที่แนะนำ

ใช้ `JOIN ... ON ...` หรือ `JOIN ... USING (...)` แทนเสมอ เพราะ:
- `ON` และ `USING` ระบุเงื่อนไข JOIN **อย่างชัดแจ้ง** (explicit) ในตัว query เอง ไม่ขึ้นกับ schema ภายนอกที่มองไม่เห็น
- แม้จะมีคนเปลี่ยนชื่อคอลัมน์ในตารางอื่นภายหลัง query ที่เขียนด้วย `ON`/`USING` ก็จะยัง error ทันที (ถ้าคอลัมน์หายไป) แทนที่จะรันผ่านแบบเงียบ ๆ ด้วยผลลัพธ์ที่ผิด — การ "พังแบบส่งเสียงดัง" (fail loudly) ย่อมดีกว่า "พังแบบเงียบ" (fail silently) เสมอในงาน production

```sql
-- แนะนำ: ระบุเงื่อนไข JOIN อย่างชัดเจนเสมอ
SELECT o.order_id, c.first_name, c.last_name
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id;
```

> **กฎทองข้อนี้ใช้ได้แทบทุกสถานการณ์:** เขียนโค้ดที่ "ตั้งใจ" ชัดเจนเสมอดีกว่าโค้ดที่ "สะดวก" แต่พึ่งพาพฤติกรรมอัตโนมัติที่มองไม่เห็นด้วยตา `NATURAL JOIN` จึงเหมาะกับการทดลองเร็ว ๆ ใน query ชั่วคราวมากกว่าโค้ดที่ต้องอยู่ในระบบระยะยาว

---

## Step 229: สรุปเปรียบเทียบ JOIN ทุกประเภทที่เรียนมา

ตารางด้านล่างนี้สรุป JOIN ทุกประเภทที่เรียนมาในหลักสูตรจนถึงบทนี้ ทั้งพฤติกรรม การใช้งานจริง และข้อควรระวัง

| ประเภท JOIN | พฤติกรรม | จำนวนแถวผลลัพธ์ (โดยทั่วไป) | ใช้เมื่อไหร่ | ข้อควรระวัง |
|---|---|---|---|---|
| **INNER JOIN** | คืนเฉพาะแถวที่จับคู่ได้ทั้งสองฝั่งตามเงื่อนไข `ON` | ≤ min(n, m) | ต้องการเฉพาะข้อมูลที่มีความสัมพันธ์ครบทั้งสองฝั่ง เช่น ออเดอร์ที่มีลูกค้าจริง | ข้อมูลฝั่งใดฝั่งหนึ่งที่ไม่มีคู่จะหายไปจากผลลัพธ์ทั้งหมด |
| **LEFT JOIN** (LEFT OUTER JOIN) | คืนทุกแถวจากตารางซ้าย + คอลัมน์จากตารางขวา (เป็น `NULL` ถ้าไม่พบคู่) | ≥ n (จำนวนแถวตารางซ้าย) | ต้องการข้อมูลทั้งหมดจากตารางหลัก แม้จะไม่มีข้อมูลที่เกี่ยวข้องในอีกตาราง เช่น ลูกค้าทุกคนแม้ไม่เคยสั่งซื้อ | ต้องใช้ `COALESCE`/`IS NULL` จัดการค่า NULL ที่เกิดขึ้นอย่างระมัดระวัง |
| **RIGHT JOIN** (RIGHT OUTER JOIN) | คืนทุกแถวจากตารางขวา + คอลัมน์จากตารางซ้าย (เป็น `NULL` ถ้าไม่พบคู่) | ≥ m (จำนวนแถวตารางขวา) | เหมือน LEFT JOIN แต่กลับด้าน — ในทางปฏิบัติมักเขียนเป็น LEFT JOIN สลับตำแหน่งตารางแทนเพื่อความสม่ำเสมอของโค้ด | ใช้น้อยกว่า LEFT JOIN มากในทางปฏิบัติ อ่านสลับทิศทางอาจสับสน |
| **FULL JOIN** (FULL OUTER JOIN) | คืนทุกแถวจากทั้งสองตาราง จับคู่เมื่อทำได้ เติม `NULL` เมื่อไม่พบคู่ฝั่งใดฝั่งหนึ่ง | ≥ max(n, m) | ต้องการเห็นข้อมูลทั้งหมดจากทั้งสองตาราง รวมถึงส่วนที่ไม่ตรงกันทั้งสองด้าน เช่น รายงาน reconciliation | ผลลัพธ์อาจมี NULL ทั้งสองฝั่งพร้อมกันในบางแถว ต้องเขียน logic แยกแยะให้ดี |
| **CROSS JOIN** | จับคู่ทุกแถวของตารางซ้ายกับทุกแถวของตารางขวา (cartesian product) ไม่มีเงื่อนไข | n × m เสมอ | สร้างชุดความเป็นไปได้ทั้งหมด เช่น size × color, วันที่ × พนักงาน | ถ้าใช้โดยไม่ตั้งใจ (ลืม `ON`) จะทำให้ข้อมูลระเบิดจำนวนแถวผิดพลาดมหาศาล |
| **SELF JOIN** | JOIN ตารางกับตัวเอง โดยใช้ alias แยกสองบทบาท ใช้ JOIN ประเภทใดก็ได้ (INNER/LEFT/...) | ขึ้นกับเงื่อนไขและประเภท JOIN ที่เลือก | หาความสัมพันธ์แบบลำดับชั้น (hierarchy) หรือเปรียบเทียบแถวภายในตารางเดียวกัน เช่น พนักงาน-หัวหน้า, เปรียบเทียบราคาสินค้า | ต้องตั้ง alias ให้ชัดเจนเสมอ และระวังเงื่อนไข `<>`/`<` เพื่อไม่ให้จับคู่กับตัวเองหรือได้คู่ซ้ำ |
| **NATURAL JOIN** | จับคู่อัตโนมัติจากคอลัมน์ที่ชื่อตรงกันทุกตัวระหว่างสองตาราง ไม่ต้องเขียน `ON`/`USING` | เหมือน INNER JOIN แต่เงื่อนไขถูกกำหนดโดย schema ไม่ใช่ query | ใช้ในงานสำรวจ/ทดลองชั่วคราวที่รู้ schema ดี ไม่แนะนำในโค้ด production | พฤติกรรมเปลี่ยนได้เองเมื่อ schema เปลี่ยน (rename/เพิ่มคอลัมน์) โดยไม่มีคำเตือน |

### แผนภาพความสัมพันธ์โดยสรุป (แบบข้อความ)

```
INNER JOIN   :  A ∩ B                       (เฉพาะที่ตรงกัน)
LEFT JOIN    :  A ∩ B  +  (A - B)           (A ทั้งหมด)
RIGHT JOIN   :  A ∩ B  +  (B - A)           (B ทั้งหมด)
FULL JOIN    :  A ∩ B  +  (A - B) + (B - A) (ทั้งสองฝั่งทั้งหมด)
CROSS JOIN   :  A × B                       (ทุกคู่ที่เป็นไปได้ ไม่สนใจความสัมพันธ์)
SELF JOIN    :  A JOIN A (บทบาทต่างกันผ่าน alias — ใช้ได้กับ JOIN ทุกประเภทข้างต้น)
NATURAL JOIN :  A INNER JOIN B USING (คอลัมน์ชื่อตรงกันทั้งหมด) — กำหนดโดย schema อัตโนมัติ
```

---

## Step 230: แบบฝึกหัดรวม — เลือกประเภท JOIN ที่เหมาะสม

โจทย์ต่อไปนี้เป็นสถานการณ์ทางธุรกิจ ให้พิจารณาว่าควรเลือกใช้ JOIN ประเภทใด ก่อนเปิดดูเฉลย

### สถานการณ์ที่ 1

**โจทย์:** ต้องการรายงานยอดขายของสินค้าทุกรายการในร้าน แม้สินค้าบางรายการจะยังไม่เคยถูกสั่งซื้อเลยก็ตาม (ยอดขายจะแสดงเป็น 0)

<details>
<summary>เฉลย</summary>

ควรใช้ **LEFT JOIN** จาก `products` ไปยัง `order_items` เพราะต้องการให้สินค้า**ทุกรายการ**ปรากฏในผลลัพธ์ แม้จะไม่มีการสั่งซื้อเลยก็ตาม

```sql
SELECT
    p.product_name,
    COALESCE(SUM(oi.quantity), 0) AS total_sold
FROM products p
LEFT JOIN order_items oi ON p.product_id = oi.product_id
GROUP BY p.product_id, p.product_name
ORDER BY total_sold DESC
LIMIT 5;
```

```
     product_name      | total_sold
------------------------+------------
 Men's Cotton T-Shirt   |          3
 Wireless Earbuds       |          3
 Women's Summer Dress   |          2
 Yoga Mat               |          3
 Thai Cooking Book      |          2
(5 rows)
```
</details>

### สถานการณ์ที่ 2

**โจทย์:** ต้องการสร้างตารางทดสอบสำหรับทีม QA โดยจับคู่ผลิตภัณฑ์ทุกรายการในหมวด "Sports & Outdoors" กับทุกวิธีการชำระเงินที่ระบบรองรับ (credit_card, promptpay, bank_transfer, cod) เพื่อทดสอบ flow การชำระเงินให้ครบทุกกรณี

<details>
<summary>เฉลย</summary>

ควรใช้ **CROSS JOIN** เพราะต้องการทุกความเป็นไปได้ระหว่างสินค้ากับวิธีชำระเงิน ไม่มีความสัมพันธ์ทางข้อมูลที่แท้จริงระหว่างสองสิ่งนี้

```sql
SELECT
    p.product_name,
    pm.payment_method
FROM products p
CROSS JOIN (VALUES ('credit_card'), ('promptpay'), ('bank_transfer'), ('cod')) AS pm(payment_method)
WHERE p.category_id = 10
ORDER BY p.product_name, pm.payment_method;
```

```
 product_name  | payment_method
----------------+----------------
 Camping Tent   | bank_transfer
 Camping Tent   | cod
 Camping Tent   | credit_card
 Camping Tent   | promptpay
 Running Shoes  | bank_transfer
 ...
(12 rows)
```
</details>

### สถานการณ์ที่ 3

**โจทย์:** ต้องการหาคู่พนักงานที่อยู่แผนกเดียวกัน เพื่อจัดกิจกรรม team building แบบจับคู่

<details>
<summary>เฉลย</summary>

ควรใช้ **SELF JOIN** บนตาราง `employees` โดยจับคู่ผ่าน `department` และใช้ `employee_id <` เพื่อป้องกันคู่ซ้ำและจับคู่กับตัวเอง

```sql
SELECT
    e1.first_name AS employee_a,
    e2.first_name AS employee_b,
    e1.department
FROM employees e1
JOIN employees e2
    ON e1.department = e2.department
    AND e1.employee_id < e2.employee_id
ORDER BY e1.department, employee_a;
```
</details>

### สถานการณ์ที่ 4

**โจทย์:** ต้องการรายงาน reconciliation ระหว่างออเดอร์กับการชำระเงิน เพื่อหาว่า (ก) ออเดอร์ไหนยังไม่ได้ชำระเงิน และ (ข) การชำระเงินไหนที่ไม่มีออเดอร์อ้างอิง (กรณีข้อมูลผิดพลาด) ในรายงานเดียว

<details>
<summary>เฉลย</summary>

ควรใช้ **FULL JOIN** เพราะต้องการเห็นทั้งสองด้านที่ไม่ตรงกัน ทั้งออเดอร์ที่ไม่มีการชำระเงิน และการชำระเงินที่ไม่มีออเดอร์อ้างอิง

```sql
SELECT
    o.order_id,
    o.status,
    pay.payment_id,
    pay.amount
FROM orders o
FULL JOIN payments pay ON o.order_id = pay.order_id
WHERE o.order_id IS NULL OR pay.payment_id IS NULL;
```

```
 order_id |  status   | payment_id | amount
----------+-----------+------------+---------
        5 | cancelled |            |
        9 | pending   |            |
       15 | pending   |            |
(3 rows)
```
</details>

### สถานการณ์ที่ 5

**โจทย์:** ต้องการดึงชื่อลูกค้าและออเดอร์คู่กันแบบเร็ว ๆ สำหรับ query ชั่วคราวที่จะลบทิ้งหลังใช้เสร็จ ไม่ได้จะนำไปใช้ในระบบจริง

<details>
<summary>เฉลย</summary>

กรณีนี้ **อาจ** ใช้ `NATURAL JOIN` ได้ เนื่องจากเป็น query ชั่วคราว (ad-hoc) ไม่ได้อยู่ในโค้ด production แต่ในทางปฏิบัติที่ดีที่สุด แนะนำให้ใช้ `JOIN ... ON` ให้เป็นนิสัยเสมอ เพื่อไม่ให้ติดพฤติกรรมที่เสี่ยงในระยะยาว

```sql
-- พอยอมรับได้สำหรับ query ทดลองชั่วคราวเท่านั้น
SELECT * FROM orders NATURAL JOIN customers LIMIT 5;

-- แนะนำมากกว่าเสมอ แม้จะเป็น query ชั่วคราว
SELECT o.*, c.first_name, c.last_name
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
LIMIT 5;
```
</details>

---

## สรุปท้ายบท

ในบทนี้เราได้เรียนรู้ JOIN สามประเภทสุดท้ายที่เติมเต็มความรู้เรื่อง JOIN ทั้งหมดในหลักสูตร:

1. **CROSS JOIN** สร้าง cartesian product — ทุกแถวของตารางหนึ่งจับคู่กับทุกแถวของอีกตารางหนึ่ง มีประโยชน์มากในการสร้างชุดความเป็นไปได้ทั้งหมด (variant matrix, ปฏิทินร่วมกับ `generate_series`) แต่ก็เป็นอันตรายมากเช่นกันเมื่อเกิดขึ้นโดยไม่ตั้งใจจากการลืมเงื่อนไข `ON`
2. **SELF JOIN** คือเทคนิคการ JOIN ตารางกับตัวเอง โดยใช้ alias แยกบทบาทให้ชัดเจน ใช้ได้กับ JOIN ทุกประเภท (INNER, LEFT, ฯลฯ) เหมาะสำหรับข้อมูลที่มีความสัมพันธ์แบบลำดับชั้น (เช่น พนักงาน-หัวหน้า) หรือการเปรียบเทียบแถวภายในตารางเดียวกัน
3. **NATURAL JOIN** จับคู่คอลัมน์อัตโนมัติจากชื่อที่ตรงกัน สะดวกแต่อันตราย เพราะพฤติกรรมขึ้นกับ schema ที่เปลี่ยนแปลงได้ตลอดเวลาโดยไม่มีการแจ้งเตือน จึงไม่แนะนำให้ใช้ในโค้ด production

### ตารางเปรียบเทียบ JOIN ทุกประเภท (ทบทวน)

| ประเภท JOIN | คืนแถวจาก | เงื่อนไข JOIN | ระดับความเสี่ยง | แนะนำใช้ใน production? |
|---|---|---|---|---|
| INNER JOIN | เฉพาะที่ตรงกันทั้งสองฝั่ง | ระบุด้วย `ON`/`USING` | ต่ำ | ใช่ |
| LEFT JOIN | ตารางซ้ายทั้งหมด | ระบุด้วย `ON`/`USING` | ต่ำ | ใช่ |
| RIGHT JOIN | ตารางขวาทั้งหมด | ระบุด้วย `ON`/`USING` | ต่ำ | ใช่ (แต่นิยมเขียนเป็น LEFT JOIN แทน) |
| FULL JOIN | ทั้งสองตารางทั้งหมด | ระบุด้วย `ON`/`USING` | ต่ำ-กลาง | ใช่ |
| CROSS JOIN | ทุกคู่ที่เป็นไปได้ | ไม่มีเงื่อนไข | สูงถ้าไม่ตั้งใจ | ใช่ ถ้าตั้งใจเขียนชัดเจน |
| SELF JOIN | ตารางเดียวกัน สองบทบาท | ระบุด้วย `ON` เสมอ | กลาง (ต้องระวัง alias และคู่ซ้ำ) | ใช่ |
| NATURAL JOIN | อัตโนมัติตามชื่อคอลัมน์ | กำหนดโดย schema | สูง | ไม่แนะนำ |

### กฎทองของบทนี้

> เขียน JOIN ให้ **ชัดเจนและตั้งใจเสมอ** — ระบุเงื่อนไขด้วย `ON` หรือ `USING` อย่างชัดแจ้งทุกครั้ง แม้จะรู้สึกว่ายาวกว่า `NATURAL JOIN` เพราะความชัดเจนวันนี้ คือการป้องกัน bug ที่ตรวจจับยากในวันข้างหน้า

---

## แบบฝึกหัด

พยายามเขียนคำตอบเองก่อนเปิดดูเฉลยใน `<details>`

**1.** เขียน query หาจำนวนแถวทั้งหมดที่จะเกิดขึ้นจาก `CROSS JOIN` ระหว่างตาราง `products` (20 แถว) กับตาราง `suppliers` (10 แถว) โดยไม่ต้องรัน query จริง ให้คำนวณด้วยปากเปล่า แล้วเขียน query ยืนยันคำตอบ

<details>
<summary>เฉลย</summary>

คำตอบทางทฤษฎี: 20 × 10 = 200 แถว

```sql
SELECT count(*) AS total FROM products CROSS JOIN suppliers;
```

```
 total
-------
   200
(1 row)
```
</details>

**2.** ใช้ `CROSS JOIN` ร่วมกับ `generate_series` สร้างรายงาน "ทุกหมวดหมู่หลัก (parent_category_id IS NULL) × ทุกไตรมาสของปี 2024" (Q1-Q4) สำหรับใช้เป็นแม่แบบวางแผนงบประมาณ

<details>
<summary>เฉลย</summary>

```sql
SELECT
    c.category_name,
    'Q' || q AS quarter
FROM categories c
CROSS JOIN generate_series(1, 4) AS q
WHERE c.parent_category_id IS NULL
ORDER BY c.category_name, quarter;
```

```
   category_name    | quarter
---------------------+---------
 Books               | Q1
 Books               | Q2
 Books               | Q3
 Books               | Q4
 Clothing            | Q1
 ...
(20 rows)
```
</details>

**3.** ทีมพัฒนาเขียน query นี้เพื่อหารายชื่อลูกค้าที่มีออเดอร์พร้อมรายการสินค้า แต่ query มีบั๊กที่ทำให้เกิด CROSS JOIN โดยไม่ตั้งใจ หาจุดผิดและแก้ไข

```sql
SELECT c.first_name, oi.product_id, oi.quantity
FROM customers c, orders o, order_items oi
WHERE c.customer_id = o.customer_id;
```

<details>
<summary>เฉลย</summary>

จุดผิดคือ query ขาดเงื่อนไขเชื่อม `orders` กับ `order_items` (`o.order_id = oi.order_id`) ทำให้ `order_items` ทุกแถวจับคู่กับทุกแถวของผลลัพธ์ `customers JOIN orders` แบบ cartesian product

```sql
-- แก้ไขแล้ว
SELECT c.first_name, oi.product_id, oi.quantity
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
JOIN order_items oi ON o.order_id = oi.order_id;
```
</details>

**4.** เขียน SELF JOIN บนตาราง `employees` เพื่อแสดงพนักงานทุกคนพร้อมชื่อหัวหน้า โดยพนักงานที่ไม่มีหัวหน้า (เช่น CEO) ให้แสดงคำว่า `'No Manager'` แทน

<details>
<summary>เฉลย</summary>

```sql
SELECT
    e.first_name || ' ' || e.last_name AS employee,
    COALESCE(m.first_name || ' ' || m.last_name, 'No Manager') AS manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.employee_id
ORDER BY e.employee_id;
```
</details>

**5.** เขียน SELF JOIN บนตาราง `employees` เพื่อหา "เพื่อนร่วมหัวหน้า" คือพนักงานสองคนที่มี `manager_id` เดียวกัน (ไม่นับจับคู่กับตัวเอง และไม่นับคู่ซ้ำ)

<details>
<summary>เฉลย</summary>

```sql
SELECT
    e1.first_name AS employee_a,
    e2.first_name AS employee_b,
    m.first_name AS shared_manager
FROM employees e1
JOIN employees e2
    ON e1.manager_id = e2.manager_id
    AND e1.employee_id < e2.employee_id
JOIN employees m ON e1.manager_id = m.employee_id
ORDER BY shared_manager, employee_a;
```

```
 employee_a | employee_b | shared_manager
-------------+-------------+-----------------
 Napat       | Siriporn    | Anan
 Kanya       | Anan        | Prasert
 Nattapong   | Ratana      | Kanya
 Duangjai    | Wichai      | Nattapong
 Chatchai    | Panida      | Siriporn
(5 rows)
```
</details>

**6.** เขียน query หาสินค้าสองรายการในหมวดหมู่เดียวกันที่ราคาต่างกันไม่เกิน 100 บาท (เพื่อแนะนำเป็นสินค้าทดแทนกันได้)

<details>
<summary>เฉลย</summary>

```sql
SELECT
    p1.product_name AS product_a,
    p2.product_name AS product_b,
    ABS(p1.unit_price - p2.unit_price) AS price_gap
FROM products p1
JOIN products p2
    ON p1.category_id = p2.category_id
    AND p1.product_id < p2.product_id
    AND ABS(p1.unit_price - p2.unit_price) <= 100
ORDER BY price_gap;
```

```
   product_a          |    product_b    | price_gap
------------------------+------------------+-----------
 Yoga Mat               | PostgreSQL Mastery Book |      0.00
 Women's Blouse         | Men's Cotton T-Shirt    |     ...
(varies based on category grouping)
```

*(หมายเหตุ: ตรวจสอบผลลัพธ์จริงกับข้อมูลของท่าน เนื่องจาก category_id ของ Yoga Mat และ PostgreSQL Mastery Book ต่างกัน ผลลัพธ์ที่ถูกต้องจะขึ้นกับ category_id ที่ตรงกันจริงเท่านั้น เช่น เทียบใน category_id = 9 (Books): Thai Cooking Book 390.00 กับ PostgreSQL Mastery Book 590.00 → price_gap 200.00 ไม่เข้าเงื่อนไข ≤100 จึงไม่ปรากฏ ส่วนคู่ที่เข้าเงื่อนไขจริงคือ category_id = 7: Non-stick Frying Pan (450.00) และ Electric Kettle (690.00) → 240.00 ไม่เข้าเงื่อนไขเช่นกัน ในชุดข้อมูลนี้อาจไม่มีคู่ที่ตรงเงื่อนไข ≤100 พอดี ให้ลองปรับเงื่อนไขเป็น ≤300 เพื่อดูผลลัพธ์)*
</details>

**7.** อธิบายด้วยคำพูดของตัวเองว่าทำไม `NATURAL JOIN` ระหว่าง `products` และ `order_items` อาจให้ผลลัพธ์ที่ผิดพลาดได้ ถ้าทั้งสองตารางมีคอลัมน์ `unit_price` ชื่อเดียวกัน

<details>
<summary>เฉลย</summary>

เพราะ `products.unit_price` หมายถึง "ราคาปัจจุบันของสินค้าในระบบ" ในขณะที่ `order_items.unit_price` หมายถึง "ราคา ณ เวลาที่สั่งซื้อ" (ซึ่งอาจแตกต่างจากราคาปัจจุบันถ้าสินค้ามีการปรับราคาไปแล้ว) `NATURAL JOIN` จะเห็นคอลัมน์ชื่อ `unit_price` ตรงกันทั้งสองตาราง และบังคับให้ใช้เป็นเงื่อนไข JOIN เพิ่มเติมโดยอัตโนมัติ (นอกเหนือจาก `product_id`) ทำให้ order_items ที่ราคาสั่งซื้อไม่ตรงกับราคาปัจจุบันของสินค้า (เช่น สินค้าที่เคยลดราคาหรือปรับราคาขึ้นภายหลัง) จะถูกตัดออกจากผลลัพธ์ไปอย่างผิดพลาด ทั้งที่ควรจะจับคู่กันได้ตามปกติผ่าน `product_id` เพียงอย่างเดียว
</details>

**8.** เขียน query ใช้ `CROSS JOIN` สร้างรายการ "สินค้าที่ยังไม่มีรีวิว" จับคู่กับ "ลูกค้าที่เคยสั่งซื้อสินค้านั้น" เพื่อส่งอีเมลขอให้รีวิว (ใบ้: อาจต้องใช้ `NOT EXISTS` หรือ `LEFT JOIN ... WHERE IS NULL` ร่วมด้วย ไม่ใช่ CROSS JOIN ล้วน ๆ — ลองพิจารณาว่าทำไม CROSS JOIN เดี่ยว ๆ ไม่เพียงพอสำหรับโจทย์นี้)

<details>
<summary>เฉลย</summary>

โจทย์นี้จงใจให้เห็นว่า `CROSS JOIN` เดี่ยว ๆ **ไม่เหมาะสม** เพราะเราต้องการ "ลูกค้าที่เคยสั่งซื้อสินค้านั้นจริง" ไม่ใช่ "ลูกค้าทุกคนจับคู่กับทุกสินค้า" การใช้ `CROSS JOIN` ตรงนี้จะทำให้เกิดคู่ที่ไม่เกี่ยวข้องกันทางธุรกิจ (ลูกค้าที่ไม่เคยซื้อสินค้านั้นเลยก็ถูกจับคู่ด้วย) คำตอบที่ถูกต้องควรใช้ `JOIN` ปกติผ่าน `order_items` แทน:

```sql
SELECT DISTINCT
    p.product_name,
    c.email
FROM products p
JOIN order_items oi ON p.product_id = oi.product_id
JOIN orders o ON oi.order_id = o.order_id
JOIN customers c ON o.customer_id = c.customer_id
WHERE NOT EXISTS (
    SELECT 1 FROM reviews r
    WHERE r.product_id = p.product_id AND r.customer_id = c.customer_id
)
ORDER BY p.product_name;
```

บทเรียนสำคัญ: `CROSS JOIN` เหมาะกับการสร้าง "ทุกความเป็นไปได้ที่ไม่มีความสัมพันธ์อยู่แล้วในข้อมูล" เท่านั้น ถ้าความสัมพันธ์มีอยู่จริงในข้อมูล (เช่น "ลูกค้าเคยซื้อสินค้านี้") ต้องใช้ JOIN ที่มีเงื่อนไขตามความสัมพันธ์จริงเสมอ
</details>

**9.** ตารางใดในชุดข้อมูลของเรา (categories, suppliers, products, customers, employees, orders, order_items, reviews, payments) มีศักยภาพในการทำ SELF JOIN ได้ตามธรรมชาติ (มีคอลัมน์ที่อ้างอิงกลับไปยังตัวเอง) และคอลัมน์นั้นคือคอลัมน์อะไร

<details>
<summary>เฉลย</summary>

มีสองตาราง:

1. **`employees`** — คอลัมน์ `manager_id INTEGER REFERENCES employees(employee_id)` อ้างอิงกลับไปยังตัวเอง ใช้ SELF JOIN หาความสัมพันธ์พนักงาน-หัวหน้าได้ตามที่เรียนไปในบทนี้
2. **`categories`** — คอลัมน์ `parent_category_id INTEGER REFERENCES categories(category_id)` อ้างอิงกลับไปยังตัวเอง ใช้ SELF JOIN หาความสัมพันธ์หมวดหมู่แม่-ลูกได้ เช่น:

```sql
SELECT
    child.category_name AS subcategory,
    parent.category_name AS parent_category
FROM categories child
JOIN categories parent ON child.parent_category_id = parent.category_id
ORDER BY parent_category, subcategory;
```

```
 subcategory       | parent_category
--------------------+------------------
 Furniture          | Home & Kitchen
 Computers          | Electronics
 Smartphones        | Electronics
 Men's Clothing     | Clothing
 Women's Clothing   | Clothing
(5 rows)
```
</details>

**10.** ให้เขียน query เดียวที่ผสมทั้ง `CROSS JOIN` และ `SELF JOIN` ในคำสั่งเดียวกัน: สร้างรายงาน "ทุกคู่พนักงานฝ่ายขายที่อยู่คนละแผนกกับทุกเดือนของไตรมาสที่ 1 ปี 2024" (Q1 = มกราคม-มีนาคม) สำหรับวางแผนตารางสลับเวรร่วมกันข้ามแผนก

<details>
<summary>เฉลย</summary>

```sql
SELECT
    e1.first_name AS employee_a,
    e2.first_name AS employee_b,
    to_char(m, 'Month') AS schedule_month
FROM employees e1
JOIN employees e2
    ON e1.department <> e2.department
    AND e1.employee_id < e2.employee_id
CROSS JOIN generate_series('2024-01-01'::date, '2024-03-01'::date, interval '1 month') AS m
ORDER BY m, employee_a, employee_b
LIMIT 10;
```

```
 employee_a | employee_b | schedule_month
-------------+-------------+-----------------
 Prasert     | Anan        | January
 Prasert     | Siriporn    | January
 Prasert     | Panida      | January
 ...
(varies)
```

query นี้แสดงให้เห็นว่า SELF JOIN (`e1`/`e2` เปรียบเทียบแผนกกัน) และ CROSS JOIN (จับคู่กับทุกเดือน) สามารถอยู่ร่วมกันในคำสั่งเดียวได้อย่างเป็นธรรมชาติ ไม่มีข้อจำกัดพิเศษใด ๆ ระหว่างสองเทคนิคนี้
</details>

---

## บทถัดไป

เมื่อเข้าใจ JOIN ทุกประเภทอย่างครบถ้วนแล้ว บทถัดไปเราจะก้าวไปสู่เรื่อง **Subqueries** — การเขียน query ซ้อน query เพื่อแก้ปัญหาที่ JOIN เพียงอย่างเดียวไม่สามารถทำได้อย่างสะดวก

**บทถัดไป:** [Part 024 — Subqueries](./part-024-subqueries.md)
