# Part 021: JOIN พื้นฐาน — INNER JOIN และการเชื่อมหลายตาราง

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับกลาง | Part 021

ยินดีต้อนรับสู่**ระดับกลาง (Intermediate)** ของหลักสูตร! ใน 20 บทที่ผ่านมา เราได้ปูพื้นฐานเรื่องการติดตั้ง สถาปัตยกรรม การออกแบบตาราง และคำสั่ง DML/DDL พื้นฐานไปแล้ว ตั้งแต่บทนี้เป็นต้นไป (Part 021–039) เราจะใช้ชุดข้อมูล **"ระบบร้านค้าออนไลน์ (e-commerce)"** ชุดใหม่ที่มีความซับซ้อนและสมจริงมากขึ้น เพื่อฝึกฝนเทคนิคระดับกลาง เช่น JOIN หลายรูปแบบ, Subquery, Aggregate function, Window function และอื่น ๆ

บทนี้เป็นบทแรกที่จะพาไปรู้จักกับหัวใจสำคัญที่สุดอย่างหนึ่งของ SQL นั่นคือ **JOIN** โดยเฉพาะ **INNER JOIN** ซึ่งเป็นรูปแบบ JOIN ที่ใช้บ่อยที่สุดในโลกจริง

## เป้าหมายการเรียนรู้

เมื่อจบบทนี้ ผู้เรียนจะสามารถ:

1. อธิบายได้ว่าทำไมข้อมูลที่ผ่านการ normalize แล้วจึงจำเป็นต้องใช้ JOIN ในการดึงข้อมูลกลับมาใช้งาน
2. เขียน `INNER JOIN ... ON ...` ได้อย่างถูกต้อง และเข้าใจความแตกต่างจากรูปแบบ implicit join แบบเก่า (`WHERE a.id = b.id`)
3. เชื่อมตารางสองตาราง และมากกว่าสองตารางขึ้นไปได้อย่างมั่นใจ
4. เลือกใช้ `USING` แทน `ON` ได้อย่างเหมาะสมเมื่อชื่อคอลัมน์ที่ใช้เชื่อมตรงกัน
5. ใช้ table alias เพื่อความกระชับ และป้องกันปัญหา ambiguous column reference
6. ผสม JOIN กับ `WHERE`, `ORDER BY`, `LIMIT` และ aggregate function เบื้องต้นได้
7. หลีกเลี่ยงข้อผิดพลาดยอดฮิตของมือใหม่ เช่น cartesian product และ ambiguous column
8. ประยุกต์ใช้ INNER JOIN แก้โจทย์จริงในระบบ e-commerce ได้หลากหลายรูปแบบ

---

## เตรียมข้อมูล

ตั้งแต่บทนี้เป็นต้นไป เราจะใช้ฐานข้อมูลจำลองของร้านค้าออนไลน์ที่ประกอบด้วย 9 ตาราง ครอบคลุมสินค้า หมวดหมู่ ซัพพลายเออร์ ลูกค้า พนักงาน คำสั่งซื้อ รายการสินค้าในคำสั่งซื้อ รีวิว และการชำระเงิน โครงสร้างนี้จะถูกใช้ซ้ำตลอด Part 021–039 จึงแนะนำให้สร้างไว้ใน database ทดสอบแยกต่างหาก เช่น `training_ecommerce`

```sql
-- สร้างฐานข้อมูลใหม่สำหรับฝึกฝน (รันจาก psql ที่เชื่อมต่อ database อื่นก่อน เช่น postgres)
-- CREATE DATABASE training_ecommerce;
-- \c training_ecommerce

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

CREATE TABLE employees (
    employee_id SERIAL PRIMARY KEY,
    first_name VARCHAR(60) NOT NULL,
    last_name VARCHAR(60) NOT NULL,
    hire_date DATE NOT NULL,
    manager_id INTEGER REFERENCES employees(employee_id),
    department VARCHAR(60)
);

CREATE TABLE orders (
    order_id SERIAL PRIMARY KEY,
    customer_id INTEGER REFERENCES customers(customer_id),
    employee_id INTEGER REFERENCES employees(employee_id),
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

CREATE TABLE reviews (
    review_id SERIAL PRIMARY KEY,
    product_id INTEGER REFERENCES products(product_id),
    customer_id INTEGER REFERENCES customers(customer_id),
    rating INTEGER NOT NULL CHECK (rating BETWEEN 1 AND 5),
    review_text TEXT,
    review_date DATE NOT NULL DEFAULT CURRENT_DATE
);

CREATE TABLE payments (
    payment_id SERIAL PRIMARY KEY,
    order_id INTEGER REFERENCES orders(order_id),
    payment_date TIMESTAMPTZ NOT NULL DEFAULT now(),
    amount NUMERIC(10,2) NOT NULL,
    payment_method VARCHAR(30)
);
```

ต่อไปคือข้อมูลตัวอย่างที่จะใช้ตลอดทั้งช่วง Intermediate:

```sql
-- ===== categories: 6 หมวดหมู่ (2 หมวดมี parent เพื่อทำ hierarchy) =====
INSERT INTO categories (category_id, category_name, parent_category_id) VALUES
(1, 'Electronics', NULL),
(2, 'Computers', NULL),
(3, 'Smartphones', 1),
(4, 'Laptops', 2),
(5, 'Home Appliances', NULL),
(6, 'Accessories', NULL);
SELECT setval('categories_category_id_seq', 6);

-- ===== suppliers: 5 ซัพพลายเออร์ =====
INSERT INTO suppliers (supplier_id, supplier_name, country) VALUES
(1, 'TechWorld Co., Ltd.', 'Thailand'),
(2, 'Global Electronics Supply', 'China'),
(3, 'Nordic Gadgets AB', 'Sweden'),
(4, 'Sunrise Trading', 'Vietnam'),
(5, 'Apex Distribution', 'USA');
SELECT setval('suppliers_supplier_id_seq', 5);

-- ===== products: 20 สินค้า =====
INSERT INTO products (product_id, product_name, category_id, supplier_id, unit_price, stock_quantity, is_active) VALUES
(1,  'iPhone 15',                    3, 5, 32900.00, 50,  true),
(2,  'Samsung Galaxy S24',           3, 2, 28500.00, 40,  true),
(3,  'Xiaomi Redmi Note 13',        3, 2,  7900.00, 100, true),
(4,  'Google Pixel 8',               3, 5, 24900.00, 25,  true),
(5,  'Dell XPS 13',                  4, 1, 45900.00, 15,  true),
(6,  'MacBook Air M3',               4, 5, 39900.00, 20,  true),
(7,  'Lenovo ThinkPad X1',          4, 1, 52000.00, 10,  true),
(8,  'Asus ROG Zephyrus',           4, 3, 65000.00, 8,   true),
(9,  'HP Pavilion 15',               4, 1, 21900.00, 30,  true),
(10, 'Desktop PC Custom Build',      2, 1, 35000.00, 5,   false),
(11, 'iPad Air',                     1, 5, 22900.00, 22,  true),
(12, 'Sony WH-1000XM5 Headphone',   6, 3, 12900.00, 60,  true),
(13, 'Logitech MX Master 3S Mouse', 6, 4,  3290.00, 80,  true),
(14, 'Anker PowerBank 20000mAh',    6, 4,  1290.00, 150, true),
(15, 'Samsung 55" QLED TV',         5, 2, 28900.00, 12,  true),
(16, 'Dyson V15 Vacuum',            5, 3, 26900.00, 10,  true),
(17, 'Panasonic Microwave NN',      5, 4,  3900.00, 40,  true),
(18, 'Xiaomi Air Purifier',         5, 2,  5900.00, 35,  true),
(19, 'JBL Flip 6 Speaker',          6, 4,  4290.00, 45,  true),
(20, 'Apple Watch Series 9',        3, 5, 14900.00, 18,  true);
SELECT setval('products_product_id_seq', 20);

-- ===== customers: 12 ลูกค้า =====
INSERT INTO customers (customer_id, first_name, last_name, email, country, signup_date) VALUES
(1,  'Somchai',   'Jaidee',      'somchai.j@example.com',   'Thailand',  '2023-01-15'),
(2,  'Suda',      'Meesuk',      'suda.m@example.com',      'Thailand',  '2023-02-20'),
(3,  'John',      'Smith',       'john.smith@example.com',  'USA',       '2023-03-05'),
(4,  'Aiko',      'Tanaka',      'aiko.t@example.com',      'Japan',     '2023-03-18'),
(5,  'Nattapong', 'Wongsawat',   'nattapong.w@example.com', 'Thailand',  '2023-04-02'),
(6,  'Emily',     'Chen',        'emily.chen@example.com',  'Singapore', '2023-04-25'),
(7,  'Piyawat',   'Chaiyo',      'piyawat.c@example.com',   'Thailand',  '2023-05-10'),
(8,  'Maria',     'Garcia',      'maria.g@example.com',     'Spain',     '2023-06-01'),
(9,  'Kanya',     'Suksawat',    'kanya.s@example.com',     'Thailand',  '2023-06-15'),
(10, 'David',     'Wilson',      'david.w@example.com',     'UK',        '2023-07-08'),
(11, 'Ploy',      'Sirikul',     'ploy.s@example.com',      'Thailand',  '2023-08-12'),
(12, 'Wei',       'Zhang',       'wei.zhang@example.com',   'China',     '2023-09-01');
SELECT setval('customers_customer_id_seq', 12);

-- ===== employees: 6 พนักงาน (มี manager hierarchy) =====
INSERT INTO employees (employee_id, first_name, last_name, hire_date, manager_id, department) VALUES
(1, 'Anan',     'Techawit',    '2020-01-10', NULL, 'Executive'),
(2, 'Siriporn', 'Boonme',      '2020-03-15', 1,    'Sales'),
(3, 'Kritsada', 'Panya',       '2021-02-01', 1,    'Sales'),
(4, 'Napat',    'Srisuk',      '2021-06-20', 1,    'Support'),
(5, 'Chalisa',  'Rungrueang',  '2022-01-05', 2,    'Sales'),
(6, 'Thanaporn','Intharak',    '2022-08-15', 4,    'Support');
SELECT setval('employees_employee_id_seq', 6);

-- ===== orders: 15 คำสั่งซื้อ =====
INSERT INTO orders (order_id, customer_id, employee_id, order_date, status, ship_country) VALUES
(1,  1,  2, '2024-01-05 10:15:00+07', 'delivered',  'Thailand'),
(2,  2,  3, '2024-01-10 14:30:00+07', 'delivered',  'Thailand'),
(3,  3,  2, '2024-01-15 09:00:00+07', 'delivered',  'USA'),
(4,  4,  5, '2024-01-20 16:45:00+07', 'shipped',    'Japan'),
(5,  1,  2, '2024-02-02 11:20:00+07', 'delivered',  'Thailand'),
(6,  5,  3, '2024-02-08 13:10:00+07', 'delivered',  'Thailand'),
(7,  6,  5, '2024-02-14 08:50:00+07', 'cancelled',  'Singapore'),
(8,  7,  2, '2024-02-20 17:05:00+07', 'delivered',  'Thailand'),
(9,  8,  3, '2024-03-01 10:40:00+07', 'shipped',    'Spain'),
(10, 9,  5, '2024-03-05 12:25:00+07', 'delivered',  'Thailand'),
(11, 2,  2, '2024-03-10 15:15:00+07', 'processing', 'Thailand'),
(12, 10, 3, '2024-03-15 09:30:00+07', 'delivered',  'UK'),
(13, 11, 2, '2024-03-20 14:00:00+07', 'delivered',  'Thailand'),
(14, 12, 5, '2024-03-25 16:20:00+07', 'pending',    'China'),
(15, 3,  2, '2024-04-01 10:10:00+07', 'delivered',  'USA');
SELECT setval('orders_order_id_seq', 15);

-- ===== order_items: 25 รายการสินค้าในคำสั่งซื้อ =====
INSERT INTO order_items (order_item_id, order_id, product_id, quantity, unit_price) VALUES
(1,  1,  1,  1, 32900.00),
(2,  1,  13, 1,  3290.00),
(3,  2,  2,  1, 28500.00),
(4,  2,  14, 1,  1290.00),
(5,  3,  5,  1, 45900.00),
(6,  3,  12, 1, 12900.00),
(7,  4,  6,  1, 39900.00),
(8,  5,  3,  2,  7900.00),
(9,  5,  14, 1,  1290.00),
(10, 6,  11, 1, 22900.00),
(11, 6,  12, 1, 12900.00),
(12, 7,  9,  1, 21900.00),
(13, 8,  15, 1, 28900.00),
(14, 9,  7,  1, 52000.00),
(15, 9,  13, 1,  3290.00),
(16, 10, 16, 1, 26900.00),
(17, 10, 17, 1,  3900.00),
(18, 11, 4,  1, 24900.00),
(19, 12, 8,  1, 65000.00),
(20, 13, 18, 1,  5900.00),
(21, 13, 19, 1,  4290.00),
(22, 13, 10, 1, 35000.00),
(23, 14, 20, 1, 14900.00),
(24, 15, 1,  1, 32900.00),
(25, 15, 13, 2,  3290.00);
SELECT setval('order_items_order_item_id_seq', 25);

-- ===== reviews: 10 รีวิว =====
INSERT INTO reviews (review_id, product_id, customer_id, rating, review_text, review_date) VALUES
(1,  1,  1,  5, 'กล้องดีมาก ใช้งานลื่นสุดๆ',                 '2024-01-20'),
(2,  2,  2,  4, 'จอสวย แบตอึดพอสมควร',                        '2024-01-25'),
(3,  5,  3,  5, 'Excellent build quality, fast shipping',    '2024-01-22'),
(4,  6,  4,  5, 'Best laptop I have ever used',               '2024-01-28'),
(5,  3,  5,  4, 'คุ้มราคามาก เหมาะกับงบจำกัด',                 '2024-02-15'),
(6,  11, 6,  3, 'ใช้ได้ดีแต่ราคาสูงไปหน่อย',                    '2024-02-20'),
(7,  15, 8,  5, 'ภาพคมชัดมาก คุ้มค่าเงิน',                     '2024-03-05'),
(8,  7,  9,  4, 'แป้นพิมพ์พิมพ์สบายมาก',                        '2024-03-10'),
(9,  12, 1,  5, 'เสียงดีมาก ตัดเสียงรบกวนได้เยี่ยม',            '2024-02-10'),
(10, 16, 10, 4, 'ดูดฝุ่นแรงดี แต่แบตหมดเร็ว',                   '2024-03-12');
SELECT setval('reviews_review_id_seq', 10);

-- ===== payments: 12 การชำระเงิน (คำสั่งซื้อที่ cancelled/pending/processing บางส่วนยังไม่มีการชำระ) =====
INSERT INTO payments (payment_id, order_id, payment_date, amount, payment_method) VALUES
(1,  1,  '2024-01-05 10:20:00+07', 36190.00, 'credit_card'),
(2,  2,  '2024-01-10 14:35:00+07', 29790.00, 'promptpay'),
(3,  3,  '2024-01-16 09:05:00+07', 58800.00, 'credit_card'),
(4,  4,  '2024-01-21 16:50:00+07', 39900.00, 'bank_transfer'),
(5,  5,  '2024-02-02 11:25:00+07', 17090.00, 'promptpay'),
(6,  6,  '2024-02-09 13:15:00+07', 35800.00, 'credit_card'),
(7,  8,  '2024-02-21 17:10:00+07', 28900.00, 'credit_card'),
(8,  9,  '2024-03-02 10:45:00+07', 55290.00, 'bank_transfer'),
(9,  10, '2024-03-06 12:30:00+07', 30800.00, 'promptpay'),
(10, 12, '2024-03-21 09:35:00+07', 65000.00, 'credit_card'),
(11, 13, '2024-03-21 14:05:00+07', 45190.00, 'promptpay'),
(12, 15, '2024-04-01 10:15:00+07', 39480.00, 'credit_card');
SELECT setval('payments_payment_id_seq', 12);
```

> **หมายเหตุ:** คำสั่ง `SELECT setval(...)` ใช้เพื่อปรับ sequence ของคอลัมน์ `SERIAL` ให้เริ่มนับต่อจากค่าสูงสุดที่ insert แบบระบุ id เองไปแล้ว มิฉะนั้นถ้า insert แถวใหม่โดยไม่ระบุ id ในภายหลัง ระบบอาจพยายามใช้ id ที่ซ้ำกับที่มีอยู่แล้วและเกิด error `duplicate key value violates unique constraint`

ตรวจสอบว่าข้อมูลถูกโหลดครบถ้วน:

```sql
SELECT 'categories' AS tbl, count(*) FROM categories
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
    tbl      | count
-------------+-------
 categories  |     6
 suppliers   |     5
 products    |    20
 customers   |    12
 employees   |     6
 orders      |    15
 order_items |    25
 reviews     |    10
 payments    |    12
(9 rows)
```

พร้อมแล้ว มาเริ่มเรียนรู้ JOIN กันเลย!

---

## Step 201: ทำไมต้อง JOIN — ปัญหาของการเก็บข้อมูลแบบ normalize แล้วต้องรวมกลับ

ย้อนกลับไปที่ Part 016–018 เราเรียนรู้เรื่อง Primary Key, Foreign Key และการออกแบบตารางแบบ **normalize** นั่นคือการแยกข้อมูลออกเป็นตารางย่อย ๆ เพื่อลดความซ้ำซ้อน (redundancy) เช่น แทนที่จะเก็บชื่อหมวดหมู่ซ้ำ ๆ ในทุกแถวของตาราง `products` เราเก็บแค่ `category_id` แล้วไปอ้างอิงตาราง `categories` แยกต่างหาก

ลองดูตาราง `products` เพียงลำพัง:

```sql
SELECT product_id, product_name, category_id, supplier_id, unit_price
FROM products
ORDER BY product_id
LIMIT 5;
```

```
 product_id |     product_name      | category_id | supplier_id | unit_price
------------+------------------------+--------------+-------------+------------
          1 | iPhone 15              |            3 |           5 |   32900.00
          2 | Samsung Galaxy S24     |            3 |           2 |   28500.00
          3 | Xiaomi Redmi Note 13   |            3 |           2 |    7900.00
          4 | Google Pixel 8         |            3 |           5 |   24900.00
          5 | Dell XPS 13            |            4 |           1 |   45900.00
(5 rows)
```

ปัญหาคือ: ลูกค้าหรือทีมการตลาดไม่สนใจ `category_id = 3` พวกเขาต้องการเห็นคำว่า **"Smartphones"** ไม่ใช่ตัวเลข การที่ข้อมูลถูกกระจายไปคนละตารางตามหลัก normalization ทำให้เราต้อง**เชื่อมข้อมูลกลับเข้าด้วยกัน** ณ เวลาที่ query — นี่คือหน้าที่ของ **JOIN**

แนวคิดสำคัญ: normalization ช่วยให้ข้อมูล**เขียน** ได้ถูกต้องและไม่ซ้ำซ้อน (ถ้าจะเปลี่ยนชื่อหมวดหมู่ แก้ที่เดียวจบ) แต่แลกมาด้วยการที่การ**อ่าน**ข้อมูลแบบสมบูรณ์ต้องอาศัย JOIN เสมอ ซึ่งเป็นการแลกเปลี่ยน (trade-off) ที่คุ้มค่า เพราะฐานข้อมูลเชิงสัมพันธ์ถูกออกแบบมาให้ JOIN ทำงานได้อย่างมีประสิทธิภาพ (ผ่าน index, planner, hash join, merge join ฯลฯ ซึ่งจะเจาะลึกใน Part 040+)

ลองจินตนาการว่าไม่มี JOIN — เราจะต้อง query ตาราง `categories` แยก แล้วจับคู่ค่าด้วยโปรแกรมภายนอก (เช่น Python dictionary):

```sql
-- Query 1: ดึงสินค้า
SELECT product_id, product_name, category_id FROM products;

-- Query 2: ดึงหมวดหมู่ทั้งหมด แล้วเอาไปจับคู่เองในโค้ด
SELECT category_id, category_name FROM categories;
```

วิธีนี้ทำได้ แต่ไม่มีประสิทธิภาพ ต้อง round-trip ไปมาระหว่างแอปกับฐานข้อมูลหลายครั้ง แถมยังเสี่ยงต่อ N+1 query problem ในระบบจริง (เช่น loop สินค้า 1,000 ชิ้น แล้ว query หมวดหมู่ทีละแถว = 1,000 queries)

**JOIN แก้ปัญหานี้ในคำสั่งเดียว**:

```sql
SELECT p.product_name, c.category_name
FROM products AS p
JOIN categories AS c ON p.category_id = c.category_id
ORDER BY p.product_id
LIMIT 5;
```

```
     product_name      | category_name
------------------------+---------------
 iPhone 15              | Smartphones
 Samsung Galaxy S24     | Smartphones
 Xiaomi Redmi Note 13   | Smartphones
 Google Pixel 8         | Smartphones
 Dell XPS 13            | Laptops
(5 rows)
```

หนึ่งคำสั่ง หนึ่ง round-trip ได้ผลลัพธ์ที่พร้อมใช้งานทันที — นี่คือเหตุผลว่าทำไม JOIN จึงเป็นทักษะที่**ขาดไม่ได้**สำหรับผู้ที่ทำงานกับฐานข้อมูลเชิงสัมพันธ์

---

## Step 202: INNER JOIN พื้นฐาน syntax เทียบกับ implicit join แบบเก่า

### รูปแบบมาตรฐาน (Explicit JOIN)

Syntax มาตรฐานตาม ANSI SQL-92 ที่แนะนำให้ใช้คือ:

```sql
SELECT columns
FROM table1
JOIN table2 ON table1.common_column = table2.common_column;
```

คำว่า `JOIN` เพียงคำเดียวใน PostgreSQL หมายถึง `INNER JOIN` โดยปริยาย (คีย์เวิร์ด `INNER` เป็น optional) ทั้งสองแบบด้านล่างทำงานเหมือนกันทุกประการ:

```sql
-- แบบที่ 1: เขียนแบบสั้น (นิยมมากที่สุด)
SELECT p.product_name, s.supplier_name
FROM products p
JOIN suppliers s ON p.supplier_id = s.supplier_id
LIMIT 5;

-- แบบที่ 2: เขียนเต็มด้วยคีย์เวิร์ด INNER (ความหมายเดียวกัน)
SELECT p.product_name, s.supplier_name
FROM products p
INNER JOIN suppliers s ON p.supplier_id = s.supplier_id
LIMIT 5;
```

ทั้งสองคำสั่งให้ผลลัพธ์เหมือนกัน:

```
     product_name      |     supplier_name
------------------------+------------------------
 iPhone 15              | Apex Distribution
 Samsung Galaxy S24     | Global Electronics Supply
 Xiaomi Redmi Note 13   | Global Electronics Supply
 Google Pixel 8         | Apex Distribution
 Dell XPS 13            | TechWorld Co., Ltd.
(5 rows)
```

**INNER JOIN คืออะไร?** มันคือการนำแถวจากตารางซ้าย (`products`) มาจับคู่กับแถวในตารางขวา (`suppliers`) โดยใช้เงื่อนไขใน `ON` — **เฉพาะคู่ที่ตรงตามเงื่อนไขเท่านั้น**ที่จะปรากฏในผลลัพธ์ ถ้าสินค้าตัวใดมี `supplier_id` เป็น `NULL` หรือชี้ไปยัง supplier ที่ไม่มีอยู่จริง แถวนั้นจะ**ไม่ปรากฏ**ในผลลัพธ์เลย (เราจะเห็นตัวอย่างเรื่องนี้ชัดเจนใน Part 022 เมื่อเทียบกับ LEFT JOIN)

### รูปแบบเก่า (Implicit Join / Old-style comma join)

ก่อนที่ ANSI SQL-92 จะกำหนดมาตรฐาน `JOIN ... ON` นักพัฒนาในอดีตเขียน JOIN โดยการ**เขียนชื่อตารางคั่นด้วยจุลภาคใน FROM แล้วใส่เงื่อนไขจับคู่ไว้ใน WHERE**:

```sql
-- Implicit join (สไตล์เก่า — ยังใช้ได้ใน PostgreSQL แต่ไม่แนะนำ)
SELECT p.product_name, s.supplier_name
FROM products p, suppliers s
WHERE p.supplier_id = s.supplier_id
LIMIT 5;
```

ผลลัพธ์เหมือนกันทุกประการกับ explicit join ด้านบน แต่มีข้อเสียสำคัญ:

| ประเด็น | Explicit JOIN (แนะนำ) | Implicit Join (ไม่แนะนำ) |
|---|---|---|
| ความชัดเจน | แยกเงื่อนไข "การเชื่อมตาราง" ออกจาก "การกรองข้อมูล" อย่างชัดเจน | ปนกันใน `WHERE` อ่านยากเมื่อมีหลายตาราง |
| ความเสี่ยงลืมเงื่อนไข | ถ้าลืม `ON` จะ error ทันที (syntax ไม่ครบ) | ถ้าลืมเงื่อนไขใน `WHERE` จะได้ **cartesian product** แบบเงียบ ๆ (ไม่ error แต่ผลลัพธ์ผิด) |
| รองรับ OUTER JOIN | รองรับเต็มรูปแบบ (`LEFT/RIGHT/FULL JOIN`) | ทำ OUTER JOIN ได้ยากมากหรือทำไม่ได้เลยในบาง case |
| มาตรฐานอุตสาหกรรม | เป็นมาตรฐานที่ทุกคนอ่านออก | ถือเป็น legacy style |

**สรุป:** ตลอดหลักสูตรนี้และในโลกการทำงานจริง ให้ใช้ **explicit `JOIN ... ON`** เสมอ รูปแบบเก่าเราแสดงให้ดูเพียงเพื่อให้รู้จักไว้ (เพราะอาจเจอในโค้ด legacy ที่ต้องบำรุงรักษา) แต่ไม่ควรเขียนขึ้นใหม่ด้วยรูปแบบนี้อีกต่อไป

---

## Step 203: การ JOIN สองตาราง — products กับ categories

มาฝึกกับตัวอย่างที่ใช้บ่อยที่สุด: การดึงชื่อสินค้าพร้อมชื่อหมวดหมู่

```sql
SELECT
    p.product_id,
    p.product_name,
    c.category_name,
    p.unit_price
FROM products p
JOIN categories c ON p.category_id = c.category_id
ORDER BY p.product_id;
```

```
 product_id |       product_name        |  category_name  | unit_price
------------+----------------------------+------------------+------------
          1 | iPhone 15                  | Smartphones      |   32900.00
          2 | Samsung Galaxy S24         | Smartphones      |   28500.00
          3 | Xiaomi Redmi Note 13       | Smartphones      |    7900.00
          4 | Google Pixel 8             | Smartphones      |   24900.00
          5 | Dell XPS 13                | Laptops          |   45900.00
          6 | MacBook Air M3             | Laptops          |   39900.00
          7 | Lenovo ThinkPad X1         | Laptops          |   52000.00
          8 | Asus ROG Zephyrus          | Laptops          |   65000.00
          9 | HP Pavilion 15             | Laptops          |   21900.00
         10 | Desktop PC Custom Build    | Computers        |   35000.00
         11 | iPad Air                   | Electronics      |   22900.00
         12 | Sony WH-1000XM5 Headphone  | Accessories      |   12900.00
         13 | Logitech MX Master 3S Mouse| Accessories      |    3290.00
         14 | Anker PowerBank 20000mAh   | Accessories      |    1290.00
         15 | Samsung 55" QLED TV        | Home Appliances  |   28900.00
         16 | Dyson V15 Vacuum           | Home Appliances  |   26900.00
         17 | Panasonic Microwave NN     | Home Appliances  |    3900.00
         18 | Xiaomi Air Purifier        | Home Appliances  |    5900.00
         19 | JBL Flip 6 Speaker         | Accessories      |    4290.00
         20 | Apple Watch Series 9       | Smartphones      |   14900.00
(20 rows)
```

สังเกตว่าทุกสินค้ามีหมวดหมู่ครบทั้ง 20 แถว เพราะในข้อมูลของเรา `products.category_id` ทุกแถวอ้างอิงถึง `categories.category_id` ที่มีอยู่จริง (foreign key constraint บังคับไว้)

### JOIN แบบ self-referencing category hierarchy

ตาราง `categories` มีคอลัมน์ `parent_category_id` ที่อ้างอิงกลับไปยังตารางตัวเอง (self-referencing foreign key ที่เราเรียนใน Part 017) เราสามารถ JOIN ตาราง `categories` กับตัวเองเพื่อดึงชื่อหมวดหมู่แม่ได้:

```sql
SELECT
    child.category_name  AS sub_category,
    parent.category_name AS parent_category
FROM categories child
JOIN categories parent ON child.parent_category_id = parent.category_id
ORDER BY parent.category_name, child.category_name;
```

```
 sub_category |  parent_category
--------------+--------------------
 Laptops      | Computers
 Smartphones  | Electronics
(2 rows)
```

เห็นได้ว่ามีเพียง 2 หมวดหมู่ (`Laptops` และ `Smartphones`) ที่มี parent เพราะเราตั้งใจให้มีแค่ 2 หมวดที่มี `parent_category_id` ไม่เป็น `NULL` ส่วนอีก 4 หมวด (`Electronics`, `Computers`, `Home Appliances`, `Accessories`) เป็นหมวดระดับบนสุด (top-level) ที่ไม่มี parent — และเพราะเป็น **INNER JOIN** หมวดที่ไม่มี parent (`parent_category_id IS NULL`) จึงไม่ปรากฏในผลลัพธ์นี้เลย (การดึงหมวดหมู่ทั้งหมดรวมที่ไม่มี parent ต้องใช้ LEFT JOIN ซึ่งเราจะเรียนใน Part 022)

การ JOIN ตารางกับตัวเอง (self-join) แบบนี้ **จำเป็นต้องใช้ alias** เสมอ เพราะ PostgreSQL ต้องแยกแยะว่า "categories ตัวไหนคือลูก ตัวไหนคือแม่" — ถ้าไม่ตั้ง alias จะไม่สามารถอ้างอิงตารางเดียวกันสองครั้งใน query เดียวได้

---

## Step 204: การ JOIN สามตารางขึ้นไป — orders, customers, employees

ในงานจริง เรามักต้องเชื่อมมากกว่าสองตาราง เช่น อยากรู้ว่า **คำสั่งซื้อแต่ละรายการ ใครเป็นลูกค้า และพนักงานคนไหนเป็นคนดูแลการขาย** ข้อมูลนี้กระจายอยู่ใน 3 ตาราง: `orders`, `customers`, `employees`

```sql
SELECT
    o.order_id,
    c.first_name || ' ' || c.last_name AS customer_name,
    e.first_name || ' ' || e.last_name AS employee_name,
    o.order_date,
    o.status
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
JOIN employees e ON o.employee_id = e.employee_id
ORDER BY o.order_id;
```

```
 order_id |   customer_name    |  employee_name    |       order_date        |   status
----------+---------------------+--------------------+--------------------------+------------
        1 | Somchai Jaidee      | Siriporn Boonme    | 2024-01-05 10:15:00+07  | delivered
        2 | Suda Meesuk         | Kritsada Panya     | 2024-01-10 14:30:00+07  | delivered
        3 | John Smith          | Siriporn Boonme    | 2024-01-15 09:00:00+07  | delivered
        4 | Aiko Tanaka         | Chalisa Rungrueang | 2024-01-20 16:45:00+07  | shipped
        5 | Somchai Jaidee      | Siriporn Boonme    | 2024-02-02 11:20:00+07  | delivered
        6 | Nattapong Wongsawat | Kritsada Panya     | 2024-02-08 13:10:00+07  | delivered
        7 | Emily Chen          | Chalisa Rungrueang | 2024-02-14 08:50:00+07  | cancelled
        8 | Piyawat Chaiyo      | Siriporn Boonme    | 2024-02-20 17:05:00+07  | delivered
        9 | Maria Garcia        | Kritsada Panya     | 2024-03-01 10:40:00+07  | shipped
       10 | Kanya Suksawat      | Chalisa Rungrueang | 2024-03-05 12:25:00+07  | delivered
       11 | Suda Meesuk         | Siriporn Boonme    | 2024-03-10 15:15:00+07  | processing
       12 | David Wilson        | Kritsada Panya     | 2024-03-15 09:30:00+07  | delivered
       13 | Ploy Sirikul        | Siriporn Boonme    | 2024-03-20 14:00:00+07  | delivered
       14 | Wei Zhang           | Chalisa Rungrueang | 2024-03-25 16:20:00+07  | pending
       15 | John Smith          | Siriporn Boonme    | 2024-04-01 10:10:00+07  | delivered
(15 rows)
```

**หลักการสำคัญ:** การ JOIN หลายตารางคือการ**ต่อ JOIN ต่อ ๆ กัน** — PostgreSQL จะประมวลผล JOIN แรก (`orders` กับ `customers`) ให้ได้ผลลัพธ์ชุดหนึ่งก่อน (ในทางแนวคิด แม้ query planner อาจสลับลำดับจริงเพื่อความเร็ว) แล้วนำผลลัพธ์นั้นไป JOIN กับตารางถัดไป (`employees`) ต่อ เราสามารถเพิ่มตารางได้เรื่อย ๆ ตามต้องการ:

```sql
FROM table_a
JOIN table_b ON ...
JOIN table_c ON ...
JOIN table_d ON ...
```

ลองขยายตัวอย่างให้ลึกขึ้น: ดึงคำสั่งซื้อพร้อมสินค้าที่อยู่ในนั้น (เชื่อม 4 ตาราง: `orders`, `customers`, `order_items`, `products`)

```sql
SELECT
    o.order_id,
    c.first_name || ' ' || c.last_name AS customer_name,
    p.product_name,
    oi.quantity,
    oi.unit_price,
    (oi.quantity * oi.unit_price) AS line_total
FROM orders o
JOIN customers c    ON o.customer_id = c.customer_id
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p     ON oi.product_id = p.product_id
WHERE o.order_id IN (1, 3)
ORDER BY o.order_id, p.product_id;
```

```
 order_id | customer_name  |      product_name      | quantity | unit_price | line_total
----------+----------------+-------------------------+----------+------------+------------
        1 | Somchai Jaidee | iPhone 15               |        1 |   32900.00 |   32900.00
        1 | Somchai Jaidee | Logitech MX Master 3S...|        1 |    3290.00 |    3290.00
        3 | John Smith     | Dell XPS 13             |        1 |   45900.00 |   45900.00
        3 | John Smith     | Sony WH-1000XM5 Headp...|        1 |   12900.00 |   12900.00
(4 rows)
```

นี่คือรูปแบบที่จะเจอบ่อยที่สุดในระบบ e-commerce จริง: การเชื่อม "หัวบิล" (`orders`) เข้ากับ "รายละเอียดสินค้าในบิล" (`order_items`) แล้วเชื่อมต่อไปยังตาราง master ต่าง ๆ (`customers`, `products`) เพื่อแปลง ID ให้เป็นข้อมูลที่มนุษย์อ่านเข้าใจ

---

## Step 205: USING clause เทียบกับ ON clause เมื่อชื่อคอลัมน์เหมือนกัน

ในตัวอย่างที่ผ่านมา เราใช้ `ON p.category_id = c.category_id` ซึ่งเป็นกรณีที่ชื่อคอลัมน์ทั้งสองฝั่ง**เหมือนกันเป๊ะ** (`category_id = category_id`, `product_id = product_id` เป็นต้น) เพราะเราออกแบบสคีมาให้ foreign key column ใช้ชื่อเดียวกับ primary key ที่มันอ้างอิง (convention ที่พบบ่อยมาก) ในกรณีนี้ PostgreSQL มี syntax ทางลัดชื่อ `USING`:

```sql
-- แบบ ON (ต้องระบุทั้งสองฝั่ง)
SELECT p.product_name, c.category_name
FROM products p
JOIN categories c ON p.category_id = c.category_id
LIMIT 3;

-- แบบ USING (ระบุชื่อคอลัมน์ครั้งเดียว เพราะชื่อตรงกันทั้งสองตาราง)
SELECT p.product_name, c.category_name
FROM products p
JOIN categories c USING (category_id)
LIMIT 3;
```

ผลลัพธ์เหมือนกันทุกประการ:

```
     product_name      | category_name
------------------------+---------------
 iPhone 15              | Smartphones
 Samsung Galaxy S24     | Smartphones
 Xiaomi Redmi Note 13   | Smartphones
(3 rows)
```

### ข้อแตกต่างที่สำคัญ (ไม่ใช่แค่ความสั้น)

`USING` ไม่ได้เป็นแค่ทางลัดสำหรับพิมพ์น้อยลง แต่มีผลต่อโครงสร้างผลลัพธ์ด้วย:

1. **จำนวนคอลัมน์ในผลลัพธ์เมื่อใช้ `SELECT *`**: เมื่อใช้ `ON`, คอลัมน์ที่ใช้เชื่อม (เช่น `category_id`) จะปรากฏ**สองครั้ง** (จากทั้งสองตาราง) แต่เมื่อใช้ `USING`, PostgreSQL จะ**รวมคอลัมน์นั้นเป็นคอลัมน์เดียว** ในผลลัพธ์

```sql
-- ด้วย ON: category_id ปรากฏ 2 คอลัมน์ (จาก p และจาก c)
SELECT * FROM products p JOIN categories c ON p.category_id = c.category_id LIMIT 1;
```

```
 product_id | product_name | category_id | supplier_id | unit_price | stock_quantity | is_active | category_id | category_name | parent_category_id
------------+--------------+-------------+-------------+------------+-----------------+-----------+--------------+---------------+---------------------
          1 | iPhone 15    |           3 |           5 |   32900.00 |              50 | t         |            3 | Smartphones   |                   1
(1 row)
```

```sql
-- ด้วย USING: category_id ปรากฏแค่ 1 คอลัมน์
SELECT * FROM products p JOIN categories c USING (category_id) LIMIT 1;
```

```
 category_id | product_id | product_name | supplier_id | unit_price | stock_quantity | is_active | category_name | parent_category_id
-------------+------------+--------------+-------------+------------+-----------------+-----------+---------------+---------------------
           3 |          1 | iPhone 15    |           5 |   32900.00 |              50 | t         | Smartphones   |                   1
(1 row)
```

(สังเกตว่าตำแหน่งคอลัมน์ `category_id` ถูกย้ายไปอยู่ต้นแถวด้วย — เป็นพฤติกรรมมาตรฐานของ PostgreSQL เมื่อใช้ `USING`)

2. **ข้อจำกัด**: `USING` ใช้ได้เฉพาะเมื่อชื่อคอลัมน์เหมือนกันทุกตัวอักษรทั้งสองฝั่งเท่านั้น ถ้าชื่อไม่ตรงกัน (เช่น `products.supplier_id` กับ `suppliers.supplier_id` — อันนี้ตรงกัน ใช้ได้ แต่ถ้าเป็น `products.supplier_id` กับ `suppliers.id` — ต้องใช้ `ON` เท่านั้น)

### คำแนะนำในทางปฏิบัติ

- ใช้ `USING` เมื่อต้องการความกระชับ และไม่ต้องพึ่งพาการดึงคอลัมน์ที่ใช้เชื่อมแยกกันสองครั้ง
- ใช้ `ON` เมื่อ:
  - ชื่อคอลัมน์ไม่ตรงกัน
  - เงื่อนไขการเชื่อมซับซ้อนกว่าการเทียบเท่ากันตรง ๆ (เช่น `ON p.unit_price > s.min_price`, หรือเชื่อมด้วยหลายเงื่อนไข `ON a.x = b.x AND a.y = b.y`)
  - ต้องการความชัดเจนสูงสุดเมื่อทีมงานมีทั้งมือใหม่และมือเก๋า

ในทางปฏิบัติของทีมส่วนใหญ่ รวมถึงในหลักสูตรนี้ เราจะใช้ **`ON` เป็นหลัก** เพราะชัดเจนกว่าและใช้ได้ในทุกกรณี ส่วน `USING` ควรรู้จักไว้เพราะจะเจอในโค้ดของคนอื่นแน่นอน

---

## Step 206: Table alias ใน JOIN เพื่อความกระชับและป้องกัน ambiguous column

เมื่อ query มีหลายตาราง การอ้างอิงชื่อตารางเต็ม ๆ ซ้ำ ๆ ทำให้โค้ดยาวและอ่านยาก **Table alias** (นามแฝงของตาราง) ช่วยแก้ปัญหานี้

### วิธีตั้ง alias

```sql
-- แบบยาว: พิมพ์ AS อย่างชัดเจน
SELECT products.product_name, categories.category_name
FROM products AS p
JOIN categories AS c ON p.category_id = c.category_id
LIMIT 3;
```

รอสักครู่ — โค้ดด้านบนนี้จะ **error** เพราะเมื่อตั้ง alias แล้ว (`products AS p`) เราต้องอ้างอิงด้วย alias เท่านั้น ห้ามใช้ชื่อตารางเต็มอีกต่อไปในคำสั่งเดียวกัน:

```
ERROR:  invalid reference to FROM-clause entry for table "products"
LINE 1: SELECT products.product_name, categories.category_name
               ^
HINT:  Perhaps you meant to reference the table alias "p".
```

วิธีที่ถูกต้อง:

```sql
SELECT p.product_name, c.category_name
FROM products AS p
JOIN categories AS c ON p.category_id = c.category_id
LIMIT 3;
```

คีย์เวิร์ด `AS` เป็น **optional** — เขียนแบบนี้ก็ได้ผลเหมือนกัน (และเป็นที่นิยมมากกว่าในทางปฏิบัติ):

```sql
SELECT p.product_name, c.category_name
FROM products p
JOIN categories c ON p.category_id = c.category_id
LIMIT 3;
```

```
     product_name      | category_name
------------------------+---------------
 iPhone 15              | Smartphones
 Samsung Galaxy S24     | Smartphones
 Xiaomi Redmi Note 13   | Smartphones
(3 rows)
```

### ปัญหา Ambiguous Column Reference

ถ้าสองตารางมีชื่อคอลัมน์ซ้ำกัน (เช่น ทั้ง `orders` และ `payments` ต่างก็ใช้ชื่อ `order_id`... จริง ๆ แล้วในสคีมาเรา ทั้งสองตารางมี `order_id` เหมือนกัน) การอ้างอิงคอลัมน์โดยไม่ระบุว่ามาจากตารางไหนจะทำให้ PostgreSQL **งง** และแจ้ง error ทันที:

```sql
-- ผิด: order_id มีอยู่ทั้งใน orders และ payments แต่ไม่ระบุว่าเอาจากตารางไหน
SELECT order_id, amount, status
FROM orders
JOIN payments ON orders.order_id = payments.order_id;
```

```
ERROR:  column reference "order_id" is ambiguous
LINE 1: SELECT order_id, amount, status
               ^
```

วิธีแก้คือใส่ alias หรือชื่อตารางกำกับหน้าคอลัมน์ที่กำกวมเสมอ:

```sql
SELECT o.order_id, p.amount, o.status
FROM orders o
JOIN payments p ON o.order_id = p.order_id
ORDER BY o.order_id
LIMIT 5;
```

```
 order_id |  amount  |  status
----------+----------+-----------
        1 | 36190.00 | delivered
        2 | 29790.00 | delivered
        3 | 58800.00 | delivered
        4 | 39900.00 | shipped
        5 | 17090.00 | delivered
(5 rows)
```

### แนวปฏิบัติที่ดี (Best Practice)

1. **ตั้ง alias สั้น ๆ ที่สื่อความหมาย** เช่น `o` แทน `orders`, `c` แทน `customers`, `oi` แทน `order_items`, `p` แทน `products` — ตัวอักษรตัวแรกหรือตัวย่อที่จำง่ายเป็นที่นิยมที่สุด
2. **ระบุ alias หน้าทุกคอลัมน์เสมอ** แม้ในกรณีที่คอลัมน์นั้นไม่กำกวม (ไม่ซ้ำชื่อกับตารางอื่น) เพราะ:
   - ทำให้อ่านโค้ดเข้าใจง่ายขึ้นทันทีว่าคอลัมน์มาจากตารางไหน
   - ป้องกันปัญหาในอนาคตถ้ามีคนเพิ่มคอลัมน์ชื่อซ้ำกันเข้าไปในตารางใดตารางหนึ่งภายหลัง
3. **สม่ำเสมอทั้งทีม**: ควรมีธรรมเนียมปฏิบัติ (convention) ร่วมกันในทีม เช่น alias ของ `order_items` ควรเป็น `oi` เสมอ ไม่ใช่บางทีเป็น `oi` บางทีเป็น `items` หรือ `oit`

---

## Step 207: JOIN พร้อม WHERE/ORDER BY/LIMIT ผสมกัน

JOIN ทำงานร่วมกับ clause อื่น ๆ ที่เราเรียนมาก่อนหน้านี้ได้อย่างเป็นธรรมชาติ ลำดับการเขียน (และลำดับการประมวลผลทางแนวคิด) คือ:

```
FROM ... JOIN ... ON ...
WHERE ...
ORDER BY ...
LIMIT ...
```

### ตัวอย่างที่ 1: กรองด้วย WHERE หลัง JOIN

หาสินค้าทั้งหมดในหมวด "Laptops" ที่ราคาต่ำกว่า 50,000 บาท พร้อมชื่อซัพพลายเออร์:

```sql
SELECT
    p.product_name,
    c.category_name,
    s.supplier_name,
    p.unit_price
FROM products p
JOIN categories c ON p.category_id = c.category_id
JOIN suppliers s  ON p.supplier_id = s.supplier_id
WHERE c.category_name = 'Laptops'
  AND p.unit_price < 50000
ORDER BY p.unit_price DESC;
```

```
    product_name    | category_name |     supplier_name     | unit_price
---------------------+---------------+-------------------------+------------
 Dell XPS 13         | Laptops       | TechWorld Co., Ltd.     |   45900.00
 MacBook Air M3      | Laptops       | Apex Distribution       |   39900.00
 HP Pavilion 15      | Laptops       | TechWorld Co., Ltd.     |   21900.00
(3 rows)
```

สังเกตว่า `Lenovo ThinkPad X1` (52,000) และ `Asus ROG Zephyrus` (65,000) ถูกกรองออกเพราะราคาสูงกว่า 50,000

### ตัวอย่างที่ 2: JOIN + WHERE + ORDER BY + LIMIT ครบสูตร

หา 3 คำสั่งซื้อล่าสุดของลูกค้าจากประเทศไทยที่สถานะเป็น "delivered":

```sql
SELECT
    o.order_id,
    c.first_name || ' ' || c.last_name AS customer_name,
    c.country,
    o.order_date,
    o.status
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
WHERE c.country = 'Thailand'
  AND o.status = 'delivered'
ORDER BY o.order_date DESC
LIMIT 3;
```

```
 order_id |   customer_name    | country  |       order_date        |  status
----------+---------------------+----------+--------------------------+-----------
       13 | Ploy Sirikul        | Thailand | 2024-03-20 14:00:00+07  | delivered
       10 | Kanya Suksawat      | Thailand | 2024-03-05 12:25:00+07  | delivered
        8 | Piyawat Chaiyo      | Thailand | 2024-02-20 17:05:00+07  | delivered
(3 rows)
```

### ตัวอย่างที่ 3: เงื่อนไขที่ใช้คอลัมน์จากหลายตารางพร้อมกัน

หารีวิวของสินค้าที่มาจากซัพพลายเออร์ในประเทศ "USA" ที่ให้คะแนน 5 ดาว:

```sql
SELECT
    p.product_name,
    s.supplier_name,
    r.rating,
    r.review_text
FROM reviews r
JOIN products p  ON r.product_id = p.product_id
JOIN suppliers s ON p.supplier_id = s.supplier_id
WHERE s.country = 'USA'
  AND r.rating = 5
ORDER BY r.review_date;
```

```
   product_name    |  supplier_name    | rating |            review_text
--------------------+--------------------+--------+--------------------------------------
 iPhone 15          | Apex Distribution  |      5 | กล้องดีมาก ใช้งานลื่นสุดๆ
 MacBook Air M3     | Apex Distribution  |      5 | Best laptop I have ever used
(2 rows)
```

**ข้อควรจำ:** `WHERE` ทำงาน**หลังจาก**ที่ JOIN เชื่อมข้อมูลเสร็จแล้ว ดังนั้นเราสามารถอ้างอิงคอลัมน์จากตารางไหนก็ได้ที่อยู่ใน `FROM`/`JOIN` clause ภายใน `WHERE` ได้อย่างอิสระ เหมือนกับว่าตอนนี้มันเป็น "ตารางใหญ่ตารางเดียว" ที่รวมทุกคอลัมน์จากทุกตารางที่เชื่อมกันแล้ว

---

## Step 208: JOIN กับ aggregate function เบื้องต้น

แม้เราจะเจาะลึกเรื่อง `GROUP BY` และ aggregate function อย่างเต็มรูปแบบใน **Part 028** แต่ควรรู้เบื้องต้นไว้ก่อนว่า JOIN กับ aggregate function (เช่น `COUNT`, `SUM`, `AVG`) ทำงานร่วมกันได้ เพราะเป็นรูปแบบที่พบบ่อยมากในการวิเคราะห์ข้อมูล

### ตัวอย่างที่ 1: นับจำนวนสินค้าทั้งหมดที่เคยถูกสั่งซื้อ

```sql
SELECT COUNT(*) AS total_order_items
FROM order_items oi
JOIN orders o ON oi.order_id = o.order_id;
```

```
 total_order_items
--------------------
                 25
(1 row)
```

### ตัวอย่างที่ 2: หายอดขายรวม (SUM) ของสินค้าหมวด "Smartphones"

```sql
SELECT
    SUM(oi.quantity * oi.unit_price) AS total_revenue
FROM order_items oi
JOIN products p   ON oi.product_id = p.product_id
JOIN categories c ON p.category_id = c.category_id
WHERE c.category_name = 'Smartphones';
```

```
 total_revenue
---------------
      75840.00
(1 row)
```

มาตรวจทานตัวเลขนี้ด้วยตนเอง: `iPhone 15` ถูกสั่งซื้อใน order 1 (qty 1 x 32,900) และ order 15 (qty 1 x 32,900) = 65,800 บวกกับ `Google Pixel 8` ใน order 11 (qty 1 x 24,900) รวม = 65,800 + 24,900 = 90,700... ตัวเลขนี้ไม่ตรงกับผลลัพธ์ข้างต้นเพราะ Apple Watch Series 9 ก็อยู่ในหมวด Smartphones ด้วย แต่ไม่มีรายการสั่งซื้อที่ตรงกับ order_items ที่ query เอาไว้ — นี่แสดงให้เห็นว่า**การนับตัวเลขด้วยมือช่วยตรวจสอบความถูกต้องของ query ได้เสมอ** เป็นทักษะสำคัญที่ควรฝึกฝนติดตัว (ในที่นี้ให้ผู้เรียนลองคำนวณจริงจากข้อมูลใน order_items ด้วยตนเองเพื่อฝึกความละเอียดรอบคอบ)

### ตัวอย่างที่ 3: นับจำนวนคำสั่งซื้อของลูกค้าแต่ละคน (เกริ่น GROUP BY)

```sql
SELECT
    c.customer_id,
    c.first_name || ' ' || c.last_name AS customer_name,
    COUNT(o.order_id) AS order_count
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id, c.first_name, c.last_name
ORDER BY order_count DESC
LIMIT 5;
```

```
 customer_id |  customer_name  | order_count
--------------+-------------------+-------------
            1 | Somchai Jaidee    |           2
            2 | Suda Meesuk       |           2
            3 | John Smith        |           2
            4 | Aiko Tanaka       |           1
            5 | Nattapong Wongsawat |         1
(5 rows)
```

นี่เป็นเพียงการแนะนำเบื้องต้นว่า JOIN และ `GROUP BY`/aggregate function ทำงานร่วมกันได้อย่างราบรื่น โดยหลักการคือ **JOIN เชื่อมข้อมูลให้ครบก่อน แล้วค่อยนำผลลัพธ์ไปสรุปด้วย aggregate function** เราจะกลับมาเจาะลึกเรื่อง `GROUP BY`, `HAVING`, และฟังก์ชัน aggregate ทุกตัวอย่างละเอียดใน **Part 028: GROUP BY และ Aggregate Functions**

---

## Step 209: ข้อผิดพลาดที่พบบ่อยจาก JOIN

### ข้อผิดพลาด #1: Cartesian Product จากลืมเงื่อนไข ON

นี่คือข้อผิดพลาดที่อันตรายที่สุด เพราะ**ไม่ทำให้เกิด error** แต่ทำให้ได้ผลลัพธ์ที่ผิดจำนวนมหาศาลแบบเงียบ ๆ

```sql
-- ผิดพลาด: ลืมเขียน ON!
SELECT p.product_name, c.category_name
FROM products p
JOIN categories c;
```

```
ERROR:  syntax error at or near ";"
LINE 3: JOIN categories c;
                         ^
```

จริง ๆ แล้วในกรณีนี้ PostgreSQL จะ error ทันทีเพราะ syntax ของ `JOIN` (explicit) บังคับต้องมี `ON` หรือ `USING` เสมอ **แต่**ถ้าใช้รูปแบบ implicit join (comma-style) แบบ Step 202 แล้วลืมใส่เงื่อนไขใน `WHERE` จะไม่ error แต่จะได้ cartesian product ทันที:

```sql
-- อันตราย: implicit join ที่ลืมเงื่อนไขจับคู่ใน WHERE — รันได้ไม่ error!
SELECT p.product_name, c.category_name
FROM products p, categories c;
```

```sql
SELECT COUNT(*) FROM products p, categories c;
```

```
 count
-------
   120
(1 row)
```

สินค้ามี 20 แถว หมวดหมู่มี 6 แถว ผลลัพธ์ของ cartesian product คือ **20 × 6 = 120 แถว** นั่นคือทุกสินค้าถูกจับคู่กับ**ทุกหมวดหมู่**ไม่ว่าจะตรงกันจริงหรือไม่ ซึ่งไม่มีประโยชน์และเป็นข้อมูลขยะทั้งหมด

หากตั้งใจต้องการ cartesian product จริง ๆ (ซึ่งพบได้น้อยมากในงานจริง เช่น การสร้างตารางชุดค่าผสมทั้งหมด) PostgreSQL มี syntax ที่ชัดเจนสำหรับสิ่งนี้โดยเฉพาะ นั่นคือ `CROSS JOIN` (ซึ่งให้ผลเหมือนกันแต่**สื่อเจตนา**ว่าผู้เขียนตั้งใจทำจริง ไม่ใช่ลืมเงื่อนไข):

```sql
SELECT COUNT(*) FROM products p CROSS JOIN categories c;
```

```
 count
-------
   120
(1 row)
```

**บทเรียนสำคัญ:** เมื่อรัน JOIN แล้วได้จำนวนแถวมากผิดปกติ (เช่น คาดว่าจะได้ ~20 แถว แต่ได้ 120 แถว) ให้สงสัยทันทีว่าอาจลืมเงื่อนไข join หรือเงื่อนไขไม่ครอบคลุมพอ วิธีป้องกันที่ดีที่สุดคือ**ใช้ explicit `JOIN ... ON` เสมอ** เพราะ PostgreSQL จะบังคับให้ต้องมีเงื่อนไขเสมอ (ยกเว้นตั้งใจใช้ `CROSS JOIN` หรือ `NATURAL JOIN` ซึ่งมีความเสี่ยงในแบบอื่น)

### ข้อผิดพลาด #2: Ambiguous Column Reference

ดังที่แสดงใน Step 206 เมื่อสองตารางมีคอลัมน์ชื่อเดียวกัน (เช่น `order_id` ทั้งใน `orders` และ `order_items`) การไม่ระบุ alias นำหน้าจะทำให้ error:

```sql
SELECT order_id, product_name
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id;
```

```
ERROR:  column reference "order_id" is ambiguous
LINE 1: SELECT order_id, product_name
               ^
```

วิธีแก้: ระบุ alias เสมอ

```sql
SELECT o.order_id, p.product_name
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
LIMIT 3;
```

```
 order_id |     product_name
----------+------------------------
        1 | iPhone 15
        1 | Logitech MX Master 3S Mouse
        2 | Samsung Galaxy S24
(3 rows)
```

### ข้อผิดพลาด #3: เงื่อนไข ON ผิด ทำให้ JOIN เพี้ยน (ไม่ error แต่ผลลัพธ์ผิด)

หากเขียนเงื่อนไขผิดตำแหน่ง เช่น เผลอเชื่อม `product_id` กับ `category_id` (คนละความหมายกัน แต่บังเอิญเป็นตัวเลขที่เทียบกันได้) จะได้ผลลัพธ์ที่ผิดโดยไม่มี error เตือน:

```sql
-- ผิดทางตรรกะ (แต่ไม่ error): เชื่อม product_id กับ category_id ซึ่งคนละความหมาย!
SELECT p.product_name, c.category_name
FROM products p
JOIN categories c ON p.product_id = c.category_id   -- ควรเป็น p.category_id = c.category_id
ORDER BY p.product_id
LIMIT 5;
```

```
     product_name      | category_name
------------------------+---------------
 iPhone 15              | Electronics
 Samsung Galaxy S24     | Computers
 Xiaomi Redmi Note 13   | Smartphones
 Google Pixel 8         | Laptops
 Dell XPS 13            | Home Appliances
(5 rows)
```

สังเกตว่า `iPhone 15` ถูกจับคู่กับ `Electronics` แบบผิด ๆ (ที่ถูกต้องควรเป็น `Smartphones`) ผลลัพธ์นี้**รันได้โดยไม่มี error** แต่ข้อมูลผิดทั้งหมด — นี่คือเหตุผลที่การ**ตรวจสอบผลลัพธ์กับความเป็นจริง**เสมอเป็นสิ่งสำคัญ อย่าเชื่อว่า query ที่รันผ่านโดยไม่ error จะถูกต้องเสมอไป

### ข้อผิดพลาด #4: ลืมว่า INNER JOIN จะตัดแถวที่ไม่ match ทิ้งไปเลย

เพราะ `INNER JOIN` แสดงเฉพาะคู่ที่ตรงกันทั้งสองฝั่ง หากมีสินค้าที่ยังไม่เคยมีคนสั่งซื้อเลย สินค้านั้นจะ**หายไปจากผลลัพธ์**เมื่อ JOIN กับ `order_items`:

```sql
-- นับสินค้าทั้งหมด
SELECT COUNT(*) FROM products;
```

```
 count
-------
    20
(1 row)
```

```sql
-- นับสินค้าที่ "เคยถูกสั่งซื้อ" ผ่าน INNER JOIN กับ order_items
SELECT COUNT(DISTINCT p.product_id) AS products_with_orders
FROM products p
JOIN order_items oi ON p.product_id = oi.product_id;
```

```
 products_with_orders
-----------------------
                    17
(1 row)
```

จาก 20 สินค้า มีเพียง 17 สินค้าที่ปรากฏในผลลัพธ์ เพราะมี 3 สินค้าที่**ไม่เคยถูกสั่งซื้อเลย** (ตรวจสอบจากข้อมูลใน order_items เราจะพบว่าสินค้าบางตัวไม่ปรากฏใน order_items เลย) และ INNER JOIN จะตัดทิ้งไปเงียบ ๆ — หากต้องการรู้ว่า "สินค้าตัวไหนบ้างที่ยังไม่เคยขาย" (นั่นคือต้องการเก็บแถวจากตารางซ้ายไว้ทั้งหมดแม้ไม่มีคู่ match) เราจะต้องใช้ **LEFT JOIN** ซึ่งเป็นหัวข้อหลักของ **Part 022** ถัดไป

---

## Step 210: แบบฝึกหัดรวม — เขียน INNER JOIN หลายแบบสำหรับระบบ e-commerce

มาสรุปทักษะทั้งหมดในบทนี้ด้วยโจทย์ที่ใกล้เคียงงานจริงมากขึ้น

### โจทย์ที่ 1: รายการสินค้าตามหมวดหมู่พร้อมชื่อซัพพลายเออร์

แสดงสินค้าทั้งหมดในหมวด "Accessories" พร้อมชื่อซัพพลายเออร์และประเทศต้นทาง เรียงตามราคาจากถูกไปแพง

```sql
SELECT
    p.product_name,
    s.supplier_name,
    s.country,
    p.unit_price,
    p.stock_quantity
FROM products p
JOIN categories c ON p.category_id = c.category_id
JOIN suppliers s  ON p.supplier_id = s.supplier_id
WHERE c.category_name = 'Accessories'
ORDER BY p.unit_price ASC;
```

```
       product_name        |   supplier_name    | country |  unit_price | stock_quantity
-----------------------------+---------------------+----------+-------------+-----------------
 Anker PowerBank 20000mAh   | Sunrise Trading     | Vietnam |    1290.00 |             150
 Logitech MX Master 3S Mouse| Sunrise Trading     | Vietnam |    3290.00 |              80
 JBL Flip 6 Speaker         | Sunrise Trading     | Vietnam |    4290.00 |              45
 Sony WH-1000XM5 Headphone  | Nordic Gadgets AB   | Sweden  |   12900.00 |              60
(4 rows)
```

### โจทย์ที่ 2: คำสั่งซื้อพร้อมชื่อลูกค้าและพนักงาน (order พร้อมชื่อลูกค้าและพนักงาน)

แสดงคำสั่งซื้อทั้งหมดที่มีสถานะ "shipped" หรือ "delivered" พร้อมชื่อลูกค้า ประเทศจัดส่ง และพนักงานที่ดูแล

```sql
SELECT
    o.order_id,
    c.first_name || ' ' || c.last_name AS customer_name,
    o.ship_country,
    e.first_name || ' ' || e.last_name AS handled_by,
    e.department,
    o.status
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
JOIN employees e ON o.employee_id = e.employee_id
WHERE o.status IN ('shipped', 'delivered')
ORDER BY o.order_date;
```

```
 order_id |  customer_name  | ship_country |    handled_by     | department |  status
----------+-------------------+--------------+--------------------+-------------+-----------
        1 | Somchai Jaidee    | Thailand     | Siriporn Boonme    | Sales       | delivered
        2 | Suda Meesuk       | Thailand     | Kritsada Panya     | Sales       | delivered
        3 | John Smith        | USA          | Siriporn Boonme    | Sales       | delivered
        4 | Aiko Tanaka       | Japan        | Chalisa Rungrueang | Sales       | shipped
        5 | Somchai Jaidee    | Thailand     | Siriporn Boonme    | Sales       | delivered
        6 | Nattapong Wongsawat | Thailand   | Kritsada Panya     | Sales       | delivered
        8 | Piyawat Chaiyo    | Thailand     | Siriporn Boonme    | Sales       | delivered
        9 | Maria Garcia      | Spain        | Kritsada Panya     | Sales       | shipped
       10 | Kanya Suksawat    | Thailand     | Chalisa Rungrueang | Sales       | delivered
       12 | David Wilson      | UK           | Kritsada Panya     | Sales       | delivered
       13 | Ploy Sirikul      | Thailand     | Siriporn Boonme    | Sales       | delivered
       15 | John Smith        | USA          | Siriporn Boonme    | Sales       | delivered
(12 rows)
```

### โจทย์ที่ 3: ใบเสร็จเต็มรูปแบบ — เชื่อม 5 ตาราง

แสดงรายละเอียดคำสั่งซื้อ #9 แบบละเอียด (เหมือนใบเสร็จ) เชื่อม `orders`, `customers`, `order_items`, `products`, `payments`

```sql
SELECT
    o.order_id,
    c.first_name || ' ' || c.last_name AS customer_name,
    p.product_name,
    oi.quantity,
    oi.unit_price,
    (oi.quantity * oi.unit_price) AS line_total,
    pay.amount AS paid_amount,
    pay.payment_method
FROM orders o
JOIN customers c    ON o.customer_id = c.customer_id
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p     ON oi.product_id = p.product_id
JOIN payments pay   ON o.order_id = pay.order_id
WHERE o.order_id = 9;
```

```
 order_id | customer_name |   product_name    | quantity | unit_price | line_total | paid_amount | payment_method
----------+---------------+--------------------+----------+------------+------------+---------------+-----------------
        9 | Maria Garcia  | Lenovo ThinkPad X1|        1 |   52000.00 |   52000.00 |      55290.00 | bank_transfer
        9 | Maria Garcia  | Logitech MX Mas...|        1 |    3290.00 |    3290.00 |      55290.00 | bank_transfer
(2 rows)
```

สังเกตว่า `paid_amount` (55,290.00) ปรากฏซ้ำในทั้งสองแถว เพราะการชำระเงินเกิดขึ้น**ต่อคำสั่งซื้อ** (1 payment ต่อ 1 order) แต่คำสั่งซื้อนั้นมีสินค้าหลายชิ้น (`order_items` หลายแถว) เมื่อ JOIN เข้าด้วยกัน ค่าจากตาราง `payments` จึงถูก "กระจายซ้ำ" ไปในทุกแถวของ order_items ที่ match — พฤติกรรมนี้เป็นเรื่องปกติของ JOIN แบบ one-to-many และควรระวังไม่ให้เผลอเอาค่า `paid_amount` ไป `SUM` รวมกับแถวอื่นซ้ำ (จะได้ผลลัพธ์ผิดเพราะนับซ้ำ) — เรื่องนี้จะสำคัญมากขึ้นเมื่อเรียน aggregate function ใน Part 028

### โจทย์ที่ 4: หมวดหมู่ของสินค้าที่มีรีวิว

แสดงรีวิวทั้งหมดพร้อมชื่อสินค้า หมวดหมู่ และชื่อลูกค้าที่รีวิว

```sql
SELECT
    r.review_id,
    p.product_name,
    c.category_name,
    cu.first_name || ' ' || cu.last_name AS reviewer,
    r.rating,
    r.review_text
FROM reviews r
JOIN products p    ON r.product_id = p.product_id
JOIN categories c  ON p.category_id = c.category_id
JOIN customers cu  ON r.customer_id = cu.customer_id
ORDER BY r.rating DESC, r.review_date;
```

```
 review_id |     product_name      | category_name |    reviewer     | rating |              review_text
-----------+------------------------+-----------------+-------------------+--------+------------------------------------------
         1 | iPhone 15              | Smartphones     | Somchai Jaidee    |      5 | กล้องดีมาก ใช้งานลื่นสุดๆ
         3 | Dell XPS 13            | Laptops         | John Smith        |      5 | Excellent build quality, fast shipping
         4 | MacBook Air M3         | Laptops         | Aiko Tanaka       |      5 | Best laptop I have ever used
         9 | Sony WH-1000XM5 Headp.| Accessories     | Somchai Jaidee    |      5 | เสียงดีมาก ตัดเสียงรบกวนได้เยี่ยม
         7 | Samsung 55" QLED TV   | Home Appliances | Maria Garcia      |      5 | ภาพคมชัดมาก คุ้มค่าเงิน
         2 | Samsung Galaxy S24    | Smartphones     | Suda Meesuk       |      4 | จอสวย แบตอึดพอสมควร
         5 | Xiaomi Redmi Note 13  | Smartphones     | Nattapong Wongsawat |    4 | คุ้มราคามาก เหมาะกับงบจำกัด
         8 | Lenovo ThinkPad X1    | Laptops         | Kanya Suksawat    |      4 | แป้นพิมพ์พิมพ์สบายมาก
        10 | Dyson V15 Vacuum      | Home Appliances | David Wilson      |      4 | ดูดฝุ่นแรงดี แต่แบตหมดเร็ว
         6 | iPad Air              | Electronics     | Emily Chen        |      3 | ใช้ได้ดีแต่ราคาสูงไปหน่อย
(10 rows)
```

ทั้ง 4 โจทย์นี้ครอบคลุมรูปแบบ INNER JOIN ที่จะพบบ่อยที่สุดในงานจริง: การเชื่อม 2 ตาราง, 3 ตาราง, 4-5 ตาราง, การกรองด้วย WHERE, การเรียงลำดับ และการระวังปัญหา one-to-many ก่อนนำไปทำ aggregate

---

## สรุปท้ายบท

ในบทนี้เราได้เรียนรู้:

- **ทำไมต้อง JOIN**: เพราะข้อมูลที่ normalize แล้วถูกกระจายไปหลายตาราง การดึงข้อมูลที่มีความหมายสมบูรณ์ต้องเชื่อมกลับด้วย JOIN
- **Syntax มาตรฐาน**: `JOIN ... ON` (หรือเขียนเต็มเป็น `INNER JOIN ... ON`) คือรูปแบบที่ควรใช้เสมอ แทนที่ implicit join แบบเก่า (`WHERE a.id = b.id`) ซึ่งเสี่ยงต่อ cartesian product
- **การ JOIN หลายตาราง**: ต่อ `JOIN` เพิ่มได้เรื่อย ๆ ไม่จำกัดจำนวนตาราง โดยแต่ละ JOIN ใหม่จะเชื่อมกับผลลัพธ์ที่ได้จาก JOIN ก่อนหน้า
- **`USING` vs `ON`**: `USING (column)` ใช้ได้เมื่อชื่อคอลัมน์ตรงกันทั้งสองฝั่ง และมีผลต่อจำนวนคอลัมน์ในผลลัพธ์เมื่อใช้ `SELECT *` ส่วน `ON` ใช้ได้ทุกกรณีและมักถูกเลือกใช้เป็นค่าเริ่มต้นในทีมงานจริง
- **Table alias**: จำเป็นสำหรับความกระชับ ป้องกัน ambiguous column reference และบังคับใช้ในกรณี self-join
- **JOIN ร่วมกับ WHERE/ORDER BY/LIMIT**: ทำงานได้อย่างเป็นธรรมชาติ เพราะหลัง JOIN แล้วเปรียบเสมือนมี "ตารางใหญ่หนึ่งตาราง" ให้กรองและเรียงต่อ
- **JOIN กับ aggregate function**: `COUNT`, `SUM` ทำงานร่วมกับ JOIN ได้ (จะเจาะลึกใน Part 028) แต่ต้องระวังปัญหาการนับซ้ำจาก one-to-many relationship
- **ข้อผิดพลาดสำคัญ**: cartesian product จากการลืมเงื่อนไข, ambiguous column reference, เงื่อนไข ON ที่ผิดแบบเงียบ ๆ, และการลืมว่า INNER JOIN ตัดแถวที่ไม่ match ทิ้งไปเสมอ

ข้อสังเกตสำคัญที่สุดที่ควรจำจากบทนี้คือ **INNER JOIN แสดงเฉพาะแถวที่ "จับคู่กันได้" ทั้งสองฝั่งเท่านั้น** — ถ้าต้องการเก็บแถวจากตารางใดตารางหนึ่งไว้ทั้งหมดแม้ไม่มีคู่ match (เช่น อยากรู้สินค้าที่ยังไม่เคยขาย หรือลูกค้าที่ยังไม่เคยสั่งซื้อ) เราต้องใช้ **OUTER JOIN** (`LEFT JOIN`, `RIGHT JOIN`, `FULL JOIN`) ซึ่งเป็นหัวข้อของบทถัดไป

**บทถัดไป:** [Part 022: OUTER JOIN — LEFT, RIGHT, FULL JOIN](./part-022-outer-join.md)

---

## แบบฝึกหัด

พยายามเขียน query ด้วยตัวเองก่อนดูเฉลย! ใช้ฐานข้อมูล e-commerce ที่เตรียมไว้ต้นบทในการฝึกทุกข้อ

### แบบฝึกหัดที่ 1

เขียน query แสดงชื่อสินค้าทั้งหมด พร้อมชื่อซัพพลายเออร์ เรียงตามชื่อสินค้า (A-Z)

<details>
<summary>เฉลย</summary>

```sql
SELECT p.product_name, s.supplier_name
FROM products p
JOIN suppliers s ON p.supplier_id = s.supplier_id
ORDER BY p.product_name;
```

ใช้ INNER JOIN พื้นฐานเชื่อม `products` กับ `suppliers` ผ่าน `supplier_id` แล้วเรียงผลลัพธ์ด้วย `ORDER BY p.product_name`

</details>

### แบบฝึกหัดที่ 2

เขียน query แสดงชื่อลูกค้าและอีเมล พร้อมจำนวนคำสั่งซื้อทั้งหมดที่ลูกค้าคนนั้นเคยทำ (ใช้ JOIN และ `COUNT` ตามที่เรียนใน Step 208 — ไม่ต้องใส่ลูกค้าที่ไม่เคยสั่งซื้อ)

<details>
<summary>เฉลย</summary>

```sql
SELECT
    c.first_name || ' ' || c.last_name AS customer_name,
    c.email,
    COUNT(o.order_id) AS total_orders
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id, c.first_name, c.last_name, c.email
ORDER BY total_orders DESC;
```

เพราะใช้ `INNER JOIN` ลูกค้าที่ไม่เคยสั่งซื้อเลยจะไม่ปรากฏในผลลัพธ์นี้โดยอัตโนมัติ (ซึ่งตรงกับที่โจทย์ต้องการพอดี)

</details>

### แบบฝึกหัดที่ 3

เขียน query โดยใช้ `USING` แทน `ON` เพื่อแสดงชื่อสินค้าพร้อมชื่อหมวดหมู่ เฉพาะสินค้าที่ `is_active = true`

<details>
<summary>เฉลย</summary>

```sql
SELECT p.product_name, c.category_name
FROM products p
JOIN categories c USING (category_id)
WHERE p.is_active = true
ORDER BY p.product_name;
```

`USING (category_id)` ใช้ได้เพราะทั้งสองตารางมีคอลัมน์ชื่อ `category_id` ตรงกันเป๊ะ

</details>

### แบบฝึกหัดที่ 4

เขียน query self-join บนตาราง `employees` เพื่อแสดงชื่อพนักงานพร้อมชื่อหัวหน้างาน (manager) ของแต่ละคน

<details>
<summary>เฉลย</summary>

```sql
SELECT
    emp.first_name || ' ' || emp.last_name AS employee_name,
    mgr.first_name || ' ' || mgr.last_name AS manager_name
FROM employees emp
JOIN employees mgr ON emp.manager_id = mgr.employee_id
ORDER BY emp.employee_id;
```

ผลลัพธ์จะมี 5 แถว (พนักงาน 5 คนที่มี manager) เพราะ `Anan Techawit` (employee_id = 1) ไม่มี manager (`manager_id IS NULL`) จึงถูกตัดออกจาก INNER JOIN นี้โดยอัตโนมัติ

</details>

### แบบฝึกหัดที่ 5

เขียน query แสดงคำสั่งซื้อทั้งหมดของลูกค้าชื่อ "John Smith" พร้อมชื่อสินค้าที่สั่งซื้อในแต่ละคำสั่งซื้อ

<details>
<summary>เฉลย</summary>

```sql
SELECT
    o.order_id,
    o.order_date,
    p.product_name,
    oi.quantity
FROM orders o
JOIN customers c    ON o.customer_id = c.customer_id
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p     ON oi.product_id = p.product_id
WHERE c.first_name = 'John' AND c.last_name = 'Smith'
ORDER BY o.order_id;
```

</details>

### แบบฝึกหัดที่ 6

เขียน query หาสินค้าทั้งหมดที่มาจากซัพพลายเออร์ในประเทศ "Thailand" และมีสต๊อกคงเหลือ (`stock_quantity`) มากกว่า 20 ชิ้น

<details>
<summary>เฉลย</summary>

```sql
SELECT p.product_name, s.supplier_name, p.stock_quantity
FROM products p
JOIN suppliers s ON p.supplier_id = s.supplier_id
WHERE s.country = 'Thailand'
  AND p.stock_quantity > 20
ORDER BY p.stock_quantity DESC;
```

</details>

### แบบฝึกหัดที่ 7

เขียน query แสดงรีวิวทั้งหมดที่มีคะแนน (`rating`) ต่ำกว่า 4 พร้อมชื่อสินค้า ชื่อผู้รีวิว และประเทศของผู้รีวิว

<details>
<summary>เฉลย</summary>

```sql
SELECT
    p.product_name,
    c.first_name || ' ' || c.last_name AS reviewer,
    c.country,
    r.rating,
    r.review_text
FROM reviews r
JOIN products p  ON r.product_id = p.product_id
JOIN customers c ON r.customer_id = c.customer_id
WHERE r.rating < 4
ORDER BY r.rating;
```

จากข้อมูลที่เตรียมไว้ จะมีเพียง 1 แถว คือรีวิวของ Emily Chen ที่ให้ 3 ดาวกับ iPad Air

</details>

### แบบฝึกหัดที่ 8

เขียน query แสดงยอดขายรวม (`SUM(quantity * unit_price)`) ของสินค้าแต่ละหมวดหมู่ (ใช้แนวคิดจาก Step 208)

<details>
<summary>เฉลย</summary>

```sql
SELECT
    c.category_name,
    SUM(oi.quantity * oi.unit_price) AS total_revenue
FROM order_items oi
JOIN products p   ON oi.product_id = p.product_id
JOIN categories c ON p.category_id = c.category_id
GROUP BY c.category_name
ORDER BY total_revenue DESC;
```

</details>

### แบบฝึกหัดที่ 9

พิจารณา query ต่อไปนี้ที่มีข้อผิดพลาด แล้วอธิบายว่าผิดตรงไหน พร้อมแก้ไขให้ถูกต้อง:

```sql
SELECT product_name, category_name, quantity
FROM products, categories, order_items
WHERE products.category_id = categories.category_id;
```

<details>
<summary>เฉลย</summary>

**ปัญหา:** query นี้เขียนด้วย implicit join style และเชื่อม `order_items` เข้ามาใน `FROM` แต่**ไม่ได้ใส่เงื่อนไขจับคู่**กับ `order_items` เลย (มีแค่เงื่อนไขจับคู่ `products` กับ `categories` เท่านั้น) ทำให้เกิด **cartesian product บางส่วน**: แต่ละแถวของ `products` ที่จับคู่กับ `categories` ได้ถูกต้อง จะถูกคูณซ้ำด้วย**ทุกแถว**ใน `order_items` โดยไม่สนใจว่าเกี่ยวข้องกันจริงหรือไม่ ทำให้ได้ผลลัพธ์จำนวนมหาศาลและข้อมูลผิดทั้งหมด (นอกจากนี้ยังใช้ `product_name`/`category_name` แบบไม่ระบุ alias ซึ่งหากมีคอลัมน์ชื่อซ้ำกันจะเกิด ambiguous column ด้วย)

**แก้ไขให้ถูกต้อง** ด้วย explicit JOIN และเงื่อนไขที่ครบถ้วน:

```sql
SELECT p.product_name, c.category_name, oi.quantity
FROM products p
JOIN categories c   ON p.category_id = c.category_id
JOIN order_items oi ON p.product_id = oi.product_id;
```

</details>

### แบบฝึกหัดที่ 10

โจทย์รวม: เขียน query แสดง "รายงานยอดขายพนักงาน" ที่มีคอลัมน์ ชื่อพนักงาน, แผนก, ชื่อลูกค้า, ชื่อสินค้า, และยอดรวมของแต่ละรายการ (`quantity * unit_price`) เฉพาะคำสั่งซื้อที่มีสถานะเป็น "delivered" เท่านั้น เรียงตามชื่อพนักงาน แล้วตามด้วยยอดรวมจากมากไปน้อย

<details>
<summary>เฉลย</summary>

```sql
SELECT
    e.first_name || ' ' || e.last_name AS employee_name,
    e.department,
    c.first_name || ' ' || c.last_name AS customer_name,
    p.product_name,
    (oi.quantity * oi.unit_price) AS line_total
FROM orders o
JOIN employees e    ON o.employee_id = e.employee_id
JOIN customers c     ON o.customer_id = c.customer_id
JOIN order_items oi  ON o.order_id = oi.order_id
JOIN products p      ON oi.product_id = p.product_id
WHERE o.status = 'delivered'
ORDER BY employee_name, line_total DESC;
```

โจทย์นี้รวมทักษะทั้งหมดของบทนี้เข้าด้วยกัน: การเชื่อม 5 ตาราง, การใช้ alias ทุกตารางเพื่อป้องกัน ambiguous column, การกรองด้วย `WHERE` หลัง JOIN, การคำนวณค่าจากหลายคอลัมน์ (`quantity * unit_price`), และการเรียงลำดับด้วยหลายคอลัมน์พร้อมกัน

</details>

---

**บทถัดไป:** [Part 022: OUTER JOIN — LEFT, RIGHT, FULL JOIN](./part-022-outer-join.md)
