# String Functions และ Pattern Matching (Regex)

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับกลาง | Part 031

## เป้าหมายการเรียนรู้

หลังจากเรียนจบบทนี้ ผู้เรียนจะสามารถ:

- ใช้ฟังก์ชันข้อความพื้นฐาน (`LENGTH`, `UPPER`, `LOWER`, `INITCAP`, `TRIM`) เพื่อทำความสะอาดและปรับรูปแบบข้อความ
- ตัด ค้นหา และดึงข้อความบางส่วนด้วย `SUBSTRING`, `LEFT`, `RIGHT`, `POSITION`
- ต่อข้อความหลายค่าด้วย `CONCAT`, `CONCAT_WS`, และ operator `||` พร้อมเข้าใจความแตกต่างในการจัดการค่า `NULL`
- แทนที่ ทำซ้ำ และกลับด้านข้อความด้วย `REPLACE`, `TRANSLATE`, `REPEAT`, `REVERSE`
- แยกข้อความออกเป็นส่วนย่อยด้วย `SPLIT_PART` และ `STRING_TO_ARRAY`
- จัดรูปแบบข้อความให้มีความยาวคงที่ด้วย `LPAD` และ `RPAD` เช่น การสร้างเลขที่ออเดอร์
- ใช้ `LIKE`, `ILIKE`, `SIMILAR TO` ในการจับคู่รูปแบบข้อความอย่างมืออาชีพ
- ใช้ Regular Expression functions (`REGEXP_MATCH`, `REGEXP_MATCHES`, `REGEXP_REPLACE`, `REGEXP_SPLIT_TO_TABLE/ARRAY`) เพื่องานที่ซับซ้อนขึ้น
- เขียน `CHECK constraint` ที่ใช้ regex เพื่อ validate รูปแบบข้อมูล เช่น อีเมลและเบอร์โทรศัพท์
- ประยุกต์ใช้ฟังก์ชันข้อความทั้งหมดร่วมกันเพื่อทำความสะอาดข้อมูล (data cleaning) ในสถานการณ์จริง

---

## เตรียมข้อมูล

บทนี้ใช้ชุดข้อมูลร้านค้าออนไลน์ (e-commerce) ชุดเดียวกับ Part 021-039 ทั้งหมด ให้รันสคริปต์ด้านล่างเพื่อสร้างตารางและข้อมูลตัวอย่างก่อนเริ่มฝึก

```sql
-- ล้างตารางเดิม (ถ้ามี) เพื่อให้เริ่มต้นสะอาด
DROP TABLE IF EXISTS payments, reviews, order_items, orders,
    employees, customers, products, suppliers, categories CASCADE;

-- โครงสร้างตาราง
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

-- ข้อมูลตัวอย่าง: categories
INSERT INTO categories (category_name, parent_category_id) VALUES
('อิเล็กทรอนิกส์', NULL),
('เครื่องใช้ไฟฟ้าในบ้าน', NULL),
('แฟชั่น', NULL),
('หนังสือ', NULL),
('มือถือและแท็บเล็ต', 1),
('คอมพิวเตอร์และแล็ปท็อป', 1),
('เสื้อผ้าผู้ชาย', 3),
('เสื้อผ้าผู้หญิง', 3),
('ของใช้ในครัว', 2),
('อุปกรณ์กีฬา', NULL);

-- ข้อมูลตัวอย่าง: suppliers
INSERT INTO suppliers (supplier_name, country) VALUES
('Thai Digital Supply Co., Ltd.', 'Thailand'),
('Shenzhen Electronics Ltd.', 'China'),
('Global Fashion House', 'Thailand'),
('Nordic Home Living', 'Sweden'),
('BookWorm Publishing', 'Thailand'),
('Sports World Inc.', 'USA'),
('Kitchen Master Co.', 'Thailand'),
('TechGear International', 'Singapore');

-- ข้อมูลตัวอย่าง: products
INSERT INTO products (product_name, category_id, supplier_id, unit_price, stock_quantity, is_active) VALUES
('iPhone 15 Pro Max 256GB', 5, 2, 42900.00, 25, true),
('Samsung Galaxy S24 Ultra', 5, 2, 39900.00, 18, true),
('Xiaomi Pad 6', 5, 2, 12900.00, 40, true),
('Dell XPS 13 Laptop', 6, 8, 45900.00, 12, true),
('MacBook Air M2', 6, 8, 39900.00, 15, true),
('Logitech MX Master 3S Mouse', 6, 8, 3290.00, 60, true),
('Panasonic Rice Cooker 1.8L', 9, 7, 1590.00, 35, true),
('Philips Air Fryer XXL', 9, 4, 5990.00, 22, true),
('IKEA Study Desk', 2, 4, 3990.00, 8, true),
('Men''s Slim Fit Denim Jacket', 7, 3, 1290.00, 50, true),
('Women''s Floral Summer Dress', 8, 3, 890.00, 45, true),
('Men''s Cotton Polo Shirt', 7, 3, 590.00, 70, true),
('Clean Code by Robert Martin', 4, 5, 690.00, 30, true),
('Atomic Habits (Thai Edition)', 4, 5, 350.00, 55, true),
('Nike Air Zoom Pegasus', 10, 6, 4290.00, 20, true),
('Yoga Mat Premium 6mm', 10, 6, 890.00, 40, true),
('Sony WH-1000XM5 Headphones', 1, 2, 12900.00, 16, true),
('Anker PowerBank 20000mAh', 1, 2, 1590.00, 65, true);

-- ข้อมูลตัวอย่าง: customers (มีข้อมูลไม่สะอาดปะปนโดยตั้งใจ เพื่อใช้ฝึกทำความสะอาดข้อมูลในบทนี้)
INSERT INTO customers (first_name, last_name, email, country, signup_date) VALUES
('สมชาย', 'ใจดี', 'somchai.jaidee@email.com', 'Thailand', '2022-01-10'),
('สุภาพร', 'แก้วขาว', '  Supaporn.K@GMAIL.com  ', 'Thailand', '2022-02-15'),
('John', 'Smith', 'john.smith@email.com', 'USA', '2022-03-01'),
('JOHN', 'DOE', 'JOHN.DOE@email.com', 'USA', '2022-03-10'),
('  Maria  ', 'Garcia', 'maria.garcia@email.com', 'Spain', '2022-04-05'),
('สมหญิง', 'รักเรียน', 'somying.r@email.com', 'Thailand', '2022-04-20'),
('David', 'lee', 'david.lee@email.com', 'Singapore', '2022-05-02'),
('Aiko', 'Tanaka', 'aiko.tanaka@email.co.jp', 'Japan', '2022-05-18'),
('ประยุทธ', 'มั่นคง', 'prayuth.m@email.com', 'Thailand', '2022-06-01'),
('Wei', 'Chen', 'wei.chen@email.cn', 'China', '2022-06-15'),
('kanya', 'RATTANA', 'kanya.r@email.com', 'Thailand', '2022-07-03'),
('Michael', 'O''Brien', 'michael.obrien@email.com', 'Ireland', '2022-07-20'),
('Nok', 'Srisawat', 'invalid-email-format', 'Thailand', '2022-08-01'),
('Praew', 'Suksri', 'praew.suksri@email.com', 'Thailand', '2022-08-15'),
('  Tom ', ' Anderson ', 'tom.anderson@email.com', 'UK', '2022-09-01'),
('Yuki', 'Yamamoto', 'yuki.yamamoto@email.co.jp', 'Japan', '2022-09-20');

-- ข้อมูลตัวอย่าง: employees
INSERT INTO employees (first_name, last_name, hire_date, manager_id, department) VALUES
('อนงค์', 'สุขสวัสดิ์', '2019-01-15', NULL, 'Management'),
('วิชัย', 'บุญมี', '2019-03-01', 1, 'Sales'),
('กันยา', 'รัตนากร', '2020-06-10', 1, 'Sales'),
('ปราโมทย์', 'ชัยยศิษฐ์', '2020-08-20', 1, 'Support'),
('ศิริพร', 'วงศ์สวัสดิ์', '2021-02-14', 2, 'Sales'),
('ธนกร', 'ศรีสุข', '2021-05-05', 4, 'Support'),
('ณภัทร', 'อินทรคำหาญ', '2022-01-10', 2, 'Sales'),
('พลอย', 'โชติรสรานี', '2022-09-01', 4, 'Support');

-- ข้อมูลตัวอย่าง: orders
INSERT INTO orders (customer_id, employee_id, order_date, status, ship_country) VALUES
(1, 2, '2024-01-15 10:23:00+07', 'delivered', 'Thailand'),
(2, 2, '2024-01-18 14:05:00+07', 'delivered', 'Thailand'),
(3, 3, '2024-01-20 09:12:00+07', 'delivered', 'USA'),
(1, 2, '2024-02-02 16:40:00+07', 'shipped', 'Thailand'),
(4, 5, '2024-02-05 11:30:00+07', 'delivered', 'USA'),
(5, 3, '2024-02-10 13:15:00+07', 'delivered', 'Spain'),
(6, 2, '2024-02-14 08:50:00+07', 'cancelled', 'Thailand'),
(7, 5, '2024-02-20 17:22:00+07', 'delivered', 'Singapore'),
(8, 7, '2024-03-01 10:05:00+07', 'delivered', 'Japan'),
(9, 2, '2024-03-05 12:45:00+07', 'shipped', 'Thailand'),
(10, 3, '2024-03-08 09:30:00+07', 'delivered', 'China'),
(11, 2, '2024-03-12 15:10:00+07', 'processing', 'Thailand'),
(12, 5, '2024-03-15 11:00:00+07', 'delivered', 'Ireland'),
(13, 2, '2024-03-18 14:20:00+07', 'pending', 'Thailand'),
(14, 7, '2024-03-22 10:15:00+07', 'delivered', 'Thailand'),
(15, 3, '2024-03-25 16:00:00+07', 'delivered', 'UK'),
(16, 5, '2024-03-28 09:40:00+07', 'shipped', 'Japan'),
(2, 2, '2024-04-01 13:30:00+07', 'delivered', 'Thailand');

-- ข้อมูลตัวอย่าง: order_items
INSERT INTO order_items (order_id, product_id, quantity, unit_price) VALUES
(1, 1, 1, 42900.00),
(1, 17, 1, 12900.00),
(2, 2, 1, 39900.00),
(3, 13, 2, 690.00),
(4, 6, 2, 3290.00),
(5, 9, 1, 3990.00),
(6, 11, 3, 890.00),
(7, 7, 1, 1590.00),
(8, 4, 1, 45900.00),
(9, 8, 1, 5990.00),
(9, 7, 1, 1590.00),
(10, 14, 4, 350.00),
(11, 3, 1, 12900.00),
(12, 10, 2, 1290.00),
(13, 5, 1, 39900.00),
(14, 18, 3, 1590.00),
(15, 15, 1, 4290.00),
(15, 16, 1, 890.00),
(16, 1, 1, 42900.00),
(17, 2, 1, 39900.00),
(18, 17, 2, 12900.00);

-- ข้อมูลตัวอย่าง: reviews (มีข้อความไม่สะอาดปะปนโดยตั้งใจ)
INSERT INTO reviews (product_id, customer_id, rating, review_text, review_date) VALUES
(1, 1, 5, 'สินค้าดีมากค่ะ!!! ส่งไวสุดๆ   แนะนำเลยค่ะ', '2024-01-20'),
(1, 3, 4, 'GREAT PHONE!!! battery life is amazing, but a bit pricey.   ', '2024-01-25'),
(2, 2, 5, '  ใช้งานดีมาก คุ้มค่ากับราคา ', '2024-01-23'),
(4, 3, 3, 'Good laptop overall, screen could be brighter. Contact me at john.smith@email.com if you have questions.', '2024-03-05'),
(7, 6, 5, 'หุงข้าวอร่อยมาก!!! ใช้ง่าย ทำความสะอาดง่าย', '2024-02-18'),
(8, 8, 2, 'ทอดไม่ค่อยกรอบเท่าที่ควร  ราคาแพงไปหน่อย', '2024-03-03'),
(13, 3, 5, 'Best programming book EVER!!! a MUST READ for every developer.', '2024-01-26'),
(14, 10, 4, 'หนังสือดี อ่านง่าย แต่ราคาแพงไปนิด', '2024-03-10'),
(9, 4, 1, 'DO NOT BUY!!! โต๊ะพังภายใน 2 สัปดาห์ ติดต่อร้านไม่ได้เลย โทร 081-234-5678 ก็ไม่มีคนรับสาย', '2024-02-10'),
(17, 7, 5, 'headphones  are   AMAZING, noise cancelling works perfectly!!!', '2024-02-25'),
(10, 9, 4, 'แจ็คเก็ตใส่สบาย ไซส์ตรงตามที่ระบุ', '2024-03-16'),
(15, 11, 5, 'Great running shoes, very comfortable for long distance running.', '2024-03-18'),
(3, 12, 3, 'Tablet ok but battery drains fast. Support email: support@shop.com', '2024-03-20'),
(18, 13, 2, 'POWER BANK ไม่เก็บไฟตามสเปค ใช้ได้ไม่ถึงครึ่งที่โฆษณา', '2024-03-22');

-- ข้อมูลตัวอย่าง: payments
INSERT INTO payments (order_id, payment_date, amount, payment_method) VALUES
(1, '2024-01-15 10:30:00+07', 55800.00, 'credit_card'),
(2, '2024-01-18 14:10:00+07', 39900.00, 'promptpay'),
(3, '2024-01-20 09:20:00+07', 1380.00, 'credit_card'),
(4, '2024-02-02 16:45:00+07', 6580.00, 'bank_transfer'),
(5, '2024-02-05 11:35:00+07', 3990.00, 'credit_card'),
(6, '2024-02-10 13:20:00+07', 2670.00, 'promptpay'),
(8, '2024-02-20 17:30:00+07', 45900.00, 'credit_card'),
(9, '2024-03-01 10:10:00+07', 7580.00, 'bank_transfer'),
(10, '2024-03-05 12:50:00+07', 1400.00, 'promptpay'),
(11, '2024-03-08 09:35:00+07', 12900.00, 'credit_card'),
(12, '2024-03-12 15:15:00+07', 2580.00, 'cod'),
(13, '2024-03-15 11:05:00+07', 39900.00, 'credit_card'),
(15, '2024-03-22 10:20:00+07', 5180.00, 'promptpay'),
(16, '2024-03-25 16:05:00+07', 42900.00, 'credit_card'),
(17, '2024-03-28 09:45:00+07', 39900.00, 'bank_transfer'),
(18, '2024-04-01 13:35:00+07', 25800.00, 'credit_card');
```

> **หมายเหตุ:** ตั้งใจใส่ข้อมูลที่ "ไม่สะอาด" ไว้ในตาราง `customers` และ `reviews` เช่น ช่องว่างเกิน ตัวพิมพ์ใหญ่-เล็กไม่สม่ำเสมอ อีเมลรูปแบบผิด และเบอร์โทรที่ปะปนอยู่ในข้อความรีวิว เพื่อให้เราได้ฝึกใช้ String Functions แก้ปัญหาจริงตลอดทั้งบท

---

## Step 301: ฟังก์ชันข้อความพื้นฐาน — LENGTH, UPPER, LOWER, INITCAP, TRIM/LTRIM/RTRIM

ฟังก์ชันกลุ่มนี้เป็นเครื่องมือพื้นฐานที่สุดในการตรวจสอบและปรับรูปแบบข้อความ ใช้บ่อยแทบทุกวันในงานจริง

### LENGTH / CHAR_LENGTH — วัดความยาวข้อความ

```sql
SELECT
    first_name,
    LENGTH(first_name)      AS char_length_thai_aware,
    OCTET_LENGTH(first_name) AS byte_length
FROM customers
WHERE customer_id IN (1, 3);
```

**ผลลัพธ์:**
```
 first_name | char_length_thai_aware | byte_length
------------+-------------------------+-------------
 สมชาย      |                       5 |          15
 John       |                       4 |           4
```

> **หมายเหตุสำคัญ:** `LENGTH()` ในฐานข้อมูลที่ตั้งค่า encoding เป็น UTF-8 จะนับ "ตัวอักษร" (character) ไม่ใช่ "ไบต์" ดังนั้นคำว่า "สมชาย" ที่ในไบต์จริงใช้ 15 ไบต์ (ภาษาไทยใช้ 3 ไบต์ต่อตัวอักษรใน UTF-8) จะแสดงความยาวเป็น 5 ตัวอักษรอย่างถูกต้อง ถ้าต้องการนับไบต์จริงให้ใช้ `OCTET_LENGTH()`

### UPPER / LOWER — แปลงตัวพิมพ์ใหญ่-เล็ก

```sql
SELECT
    email,
    UPPER(email) AS email_upper,
    LOWER(email) AS email_lower
FROM customers
WHERE customer_id = 4;
```

**ผลลัพธ์:**
```
        email         |      email_upper      |      email_lower
-----------------------+------------------------+------------------------
 JOHN.DOE@email.com    | JOHN.DOE@EMAIL.COM     | john.doe@email.com
```

`LOWER()` มีประโยชน์มากในการทำ **case-insensitive comparison** เช่น เปรียบเทียบอีเมลโดยไม่สนใจตัวพิมพ์:

```sql
SELECT customer_id, first_name, email
FROM customers
WHERE LOWER(email) = LOWER('john.doe@EMAIL.COM');
```

**ผลลัพธ์:**
```
 customer_id | first_name |        email
-------------+------------+------------------------
           4 | JOHN       | JOHN.DOE@email.com
```

> **ข้อสังเกตเรื่องภาษาไทย:** `UPPER()` และ `LOWER()` ใช้ไม่ได้ผลกับตัวอักษรไทยเพราะภาษาไทยไม่มีแนวคิดตัวพิมพ์ใหญ่-เล็ก ฟังก์ชันจะคืนค่าข้อความเดิมโดยไม่เปลี่ยนแปลง

### INITCAP — ทำให้ตัวอักษรแรกของแต่ละคำเป็นตัวพิมพ์ใหญ่

```sql
SELECT
    first_name,
    last_name,
    INITCAP(first_name) AS name_initcap,
    INITCAP(last_name)  AS lastname_initcap
FROM customers
WHERE customer_id IN (4, 7, 11);
```

**ผลลัพธ์:**
```
 first_name | last_name | name_initcap | lastname_initcap
------------+-----------+--------------+-------------------
 JOHN       | DOE       | John         | Doe
 David      | lee       | David        | Lee
 kanya      | RATTANA   | Kanya        | Rattana
```

`INITCAP` เหมาะกับการทำให้ชื่อลูกค้าที่กรอกมาไม่สม่ำเสมอ (บางคนพิมพ์ตัวใหญ่ทั้งหมด บางคนพิมพ์ตัวเล็กทั้งหมด) ให้อยู่ในรูปแบบมาตรฐานเดียวกัน

### TRIM / LTRIM / RTRIM — ตัดช่องว่าง (หรืออักขระอื่น) ออกจากข้อความ

```sql
SELECT
    '[' || first_name || ']'           AS original,
    '[' || TRIM(first_name) || ']'     AS trimmed_both,
    '[' || LTRIM(first_name) || ']'    AS trimmed_left,
    '[' || RTRIM(first_name) || ']'    AS trimmed_right
FROM customers
WHERE customer_id IN (5, 15);
```

**ผลลัพธ์:**
```
   original   | trimmed_both | trimmed_left | trimmed_right
--------------+--------------+--------------+----------------
 [  Maria  ]  | [Maria]      | [Maria  ]    | [  Maria]
 [  Tom ]     | [Tom]        | [Tom ]       | [  Tom]
```

`TRIM` ยังสามารถระบุอักขระที่ต้องการตัดออกได้ ไม่จำกัดแค่ช่องว่าง โดยใช้ syntax แบบเต็ม `TRIM([LEADING|TRAILING|BOTH] characters FROM string)`:

```sql
SELECT
    TRIM(BOTH '*' FROM '***สินค้าขายดี***')       AS trim_stars,
    TRIM(LEADING '0' FROM '000123')                AS trim_leading_zero,
    TRIM(TRAILING '.' FROM 'ราคาพิเศษ...')          AS trim_trailing_dots;
```

**ผลลัพธ์:**
```
    trim_stars    | trim_leading_zero | trim_trailing_dots
-------------------+--------------------+----------------------
 สินค้าขายดี       | 123                | ราคาพิเศษ
```

ตัวอย่างการใช้งานจริง: มาตรฐานอีเมลของลูกค้าทั้งหมดในตาราง (ตัดช่องว่างและแปลงเป็นตัวพิมพ์เล็ก) เพื่อเตรียมใช้ในขั้นตอนถัดไป:

```sql
SELECT
    customer_id,
    first_name,
    email AS email_original,
    LOWER(TRIM(email)) AS email_normalized
FROM customers
WHERE customer_id = 2;
```

**ผลลัพธ์:**
```
 customer_id | first_name |             email_original              |    email_normalized
-------------+------------+-------------------------------------------+---------------------------
           2 | สุภาพร     |   Supaporn.K@GMAIL.com                     | supaporn.k@gmail.com
```

---

## Step 302: SUBSTRING, LEFT, RIGHT, POSITION — การตัดและค้นหาข้อความ

### SUBSTRING — ดึงข้อความบางส่วน

PostgreSQL รองรับ `SUBSTRING` สองรูปแบบ syntax:

```sql
SELECT
    product_name,
    SUBSTRING(product_name FROM 1 FOR 10)   AS sub_sql_standard,
    SUBSTRING(product_name, 1, 10)           AS sub_function_style
FROM products
WHERE product_id = 1;
```

**ผลลัพธ์:**
```
      product_name       | sub_sql_standard | sub_function_style
--------------------------+-------------------+----------------------
 iPhone 15 Pro Max 256GB  | iPhone 15 | iPhone 15
```

> **หมายเหตุ:** ตำแหน่ง (position) ของข้อความใน PostgreSQL เริ่มนับที่ **1 เสมอ** (1-based indexing) ไม่ใช่ 0 เหมือนภาษาโปรแกรมมิ่งทั่วไป เช่น Python หรือ JavaScript

`SUBSTRING` ยังสามารถละพารามิเตอร์ `FOR` ได้ เพื่อดึงข้อความตั้งแต่ตำแหน่งที่ระบุไปจนจบ:

```sql
SELECT
    product_name,
    SUBSTRING(product_name FROM 8) AS from_position_8
FROM products
WHERE product_id = 1;
```

**ผลลัพธ์:**
```
      product_name       | from_position_8
--------------------------+-------------------
 iPhone 15 Pro Max 256GB  | 15 Pro Max 256GB
```

### LEFT / RIGHT — ดึงข้อความจากด้านซ้าย/ขวา

```sql
SELECT
    product_name,
    LEFT(product_name, 6)   AS first_6_chars,
    RIGHT(product_name, 4)  AS last_4_chars
FROM products
WHERE product_id IN (1, 17);
```

**ผลลัพธ์:**
```
       product_name        | first_6_chars | last_4_chars
----------------------------+----------------+---------------
 iPhone 15 Pro Max 256GB    | iPhone         | 56GB
 Sony WH-1000XM5 Headphones | Sony W         | ones
```

`LEFT` และ `RIGHT` รองรับค่าลบด้วย ซึ่งหมายถึง "ตัดออก" จำนวนตัวอักษรจากฝั่งตรงข้าม:

```sql
SELECT
    LEFT('Clean Code by Robert Martin', -13)  AS all_except_last_13,
    RIGHT('Clean Code by Robert Martin', -11) AS all_except_first_11;
```

**ผลลัพธ์:**
```
   all_except_last_13   |   all_except_first_11
--------------------------+---------------------------
 Clean Code by            | Robert Martin
```

### POSITION / STRPOS — ค้นหาตำแหน่งของข้อความย่อย

```sql
SELECT
    product_name,
    POSITION('Pro' IN product_name)  AS pos_using_position,
    STRPOS(product_name, 'Pro')      AS pos_using_strpos
FROM products
WHERE product_id = 1;
```

**ผลลัพธ์:**
```
      product_name       | pos_using_position | pos_using_strpos
--------------------------+---------------------+--------------------
 iPhone 15 Pro Max 256GB  |                  11 |                11
```

ถ้าหาไม่พบจะคืนค่า `0` (ไม่ใช่ `NULL`) — จุดนี้เป็นข้อควรระวังที่มือใหม่มักพลาด:

```sql
SELECT POSITION('xyz' IN 'iPhone 15 Pro Max') AS not_found;
```

**ผลลัพธ์:**
```
 not_found
------------
          0
```

### ตัวอย่างประยุกต์: ดึงโดเมนอีเมลของลูกค้าด้วย POSITION + SUBSTRING

```sql
SELECT
    email,
    POSITION('@' IN email) AS at_pos,
    SUBSTRING(email FROM POSITION('@' IN email) + 1) AS email_domain
FROM customers
WHERE customer_id IN (1, 8, 10);
```

**ผลลัพธ์:**
```
           email           | at_pos | email_domain
-----------------------------+---------+----------------
 somchai.jaidee@email.com    |     16 | email.com
 aiko.tanaka@email.co.jp     |     12 | email.co.jp
 wei.chen@email.cn           |      9 | email.cn
```

---

## Step 303: CONCAT, CONCAT_WS, || operator — การต่อข้อความและการจัดการ NULL ที่แตกต่างกัน

การต่อข้อความ (string concatenation) เป็นงานที่พบบ่อยมาก แต่ PostgreSQL มีสามวิธีหลักที่ **จัดการค่า `NULL` ต่างกันโดยสิ้นเชิง** ซึ่งเป็นจุดที่ทำให้เกิดบั๊กบ่อยที่สุดถ้าไม่เข้าใจ

### Operator `||` — เข้มงวดที่สุด: ถ้ามี NULL ผลลัพธ์คือ NULL ทันที

```sql
SELECT
    first_name || ' ' || last_name AS full_name_pipe
FROM customers
WHERE customer_id = 1;
```

**ผลลัพธ์:**
```
 full_name_pipe
-----------------
 สมชาย ใจดี
```

แต่ถ้ามีค่า `NULL` ปนอยู่ ผลลัพธ์ทั้งหมดจะกลายเป็น `NULL`:

```sql
SELECT
    'Order #' || NULL || ' confirmed' AS result_with_null;
```

**ผลลัพธ์:**
```
 result_with_null
-------------------
 (null)
```

### CONCAT() — ปลอดภัยกว่า: ข้าม (skip) ค่า NULL โดยอัตโนมัติ

```sql
SELECT
    CONCAT('Order #', NULL, ' confirmed') AS result_with_concat,
    CONCAT(first_name, ' ', last_name, ' <', email, '>') AS customer_label
FROM customers
WHERE customer_id = 1
LIMIT 1;
```

**ผลลัพธ์:**
```
   result_with_concat    |               customer_label
---------------------------+------------------------------------------------
 Order # confirmed         | สมชาย ใจดี <somchai.jaidee@email.com>
```

สังเกตว่า `CONCAT('Order #', NULL, ' confirmed')` ให้ผลลัพธ์ `'Order # confirmed'` โดยที่ `NULL` ถูกข้ามไปเฉยๆ ไม่ทำให้ทั้งประโยคเป็น `NULL`

### CONCAT_WS() — ต่อข้อความพร้อมตัวคั่น (Separator) และข้าม NULL

`CONCAT_WS` ย่อมาจาก **Concat With Separator** พารามิเตอร์ตัวแรกคือตัวคั่นที่จะแทรกระหว่างค่าที่ไม่ใช่ `NULL` เท่านั้น

```sql
SELECT
    CONCAT_WS(', ', first_name, last_name, country) AS full_info
FROM customers
WHERE customer_id IN (1, 3);
```

**ผลลัพธ์:**
```
        full_info
----------------------------
 สมชาย, ใจดี, Thailand
 John, Smith, USA
```

ข้อดีของ `CONCAT_WS` เห็นชัดเมื่อมีค่า `NULL` แทรกอยู่ตรงกลาง — จะไม่มีตัวคั่นซ้อนกันเกิดขึ้น:

```sql
SELECT
    CONCAT_WS(' - ', 'สาขากรุงเทพ', NULL, 'ชั้น 3') AS branch_label;
```

**ผลลัพธ์:**
```
      branch_label
--------------------------
 สาขากรุงเทพ - ชั้น 3
```

(ถ้าใช้ `||` ตรงๆ จะได้ `NULL` ทั้งหมด และถ้าต่อด้วยเครื่องหมายคั่นตรงๆ โดยไม่ใช้ `CONCAT_WS` จะได้ `'สาขากรุงเทพ -  - ชั้น 3'` ซึ่งมีตัวคั่นซ้อนกันน่าเกลียด)

### ตารางเปรียบเทียบพฤติกรรมเมื่อเจอ NULL

| วิธี | ตัวอย่าง | ผลลัพธ์เมื่อมี NULL |
|---|---|---|
| `\|\|` | `'A' \|\| NULL \|\| 'B'` | `NULL` (ทั้งประโยค) |
| `CONCAT()` | `CONCAT('A', NULL, 'B')` | `'AB'` (ข้าม NULL) |
| `CONCAT_WS()` | `CONCAT_WS('-', 'A', NULL, 'B')` | `'A-B'` (ข้าม NULL และไม่เว้นตัวคั่นซ้อน) |

### ตัวอย่างประยุกต์: สร้าง shipping label จาก orders + customers

```sql
SELECT
    o.order_id,
    CONCAT_WS(', ', c.first_name || ' ' || c.last_name, c.country, o.ship_country) AS shipping_label
FROM orders o
JOIN customers c ON c.customer_id = o.customer_id
WHERE o.order_id IN (1, 5, 8);
```

**ผลลัพธ์:**
```
 order_id |                shipping_label
----------+-----------------------------------------------
        1 | สมชาย ใจดี, Thailand, Thailand
        5 | Maria   Garcia, Spain, USA
        8 | David lee, Singapore, Singapore
```

> **เคล็ดลับ:** ฟังก์ชัน `FORMAT()` เป็นอีกทางเลือกที่มีประโยชน์เมื่อต้องต่อข้อความแบบมี placeholder คล้าย `printf` เช่น `FORMAT('ลูกค้า %s สั่งซื้อ %s บาท', c.first_name, o.amount)` ซึ่งจะกล่าวถึงในบทที่เกี่ยวกับ Dynamic SQL ต่อไป

---

## Step 304: REPLACE, TRANSLATE, REPEAT, REVERSE

### REPLACE() — แทนที่ข้อความย่อยทั้งหมดที่ตรงกัน

```sql
SELECT
    review_text,
    REPLACE(review_text, '!!!', '.') AS cleaned_punctuation
FROM reviews
WHERE review_id = 1;
```

**ผลลัพธ์:**
```
              review_text               |          cleaned_punctuation
------------------------------------------+------------------------------------------
 สินค้าดีมากค่ะ!!! ส่งไวสุดๆ   แนะนำเลยค่ะ | สินค้าดีมากค่ะ. ส่งไวสุดๆ   แนะนำเลยค่ะ
```

`REPLACE` แทนที่ **ทุก** ตำแหน่งที่พบ (ไม่ใช่แค่ตำแหน่งแรก) ตัวอย่างการลบ HTML tag แบบง่าย ๆ (สำหรับ tag ที่รู้รูปแบบตายตัว):

```sql
SELECT REPLACE(REPLACE('<b>ดีมาก</b>', '<b>', ''), '</b>', '') AS no_bold_tags;
```

**ผลลัพธ์:**
```
 no_bold_tags
---------------
 ดีมาก
```

### TRANSLATE() — แทนที่ทีละตัวอักษร (character-by-character)

`TRANSLATE` ต่างจาก `REPLACE` ตรงที่ทำงานแบบ **แมปตัวอักษรต่อตัวอักษร** ไม่ใช่แทนที่ทั้งคำ เหมาะกับงานเช่นการลบหรือแปลงอักขระพิเศษหลายตัวพร้อมกันในครั้งเดียว

```sql
SELECT TRANSLATE('081-234-5678', '-', '') AS phone_no_dashes;
```

**ผลลัพธ์:**
```
 phone_no_dashes
-------------------
 0812345678
```

ตัวอย่างที่แสดงความต่างชัดเจน: แปลงตัวอักษร `a,e,i,o,u` เป็น `*` ทีละตัวในคำเดียว:

```sql
SELECT
    TRANSLATE('customer service', 'aeiou', '*****') AS vowels_masked,
    REPLACE('customer service', 'aeiou', '*****')    AS vowels_not_found;
```

**ผลลัพธ์:**
```
      vowels_masked      |    vowels_not_found
---------------------------+---------------------------
 c*st*m*r s*rv*c*           | customer service
```

สังเกตว่า `REPLACE` หาคำว่า `'aeiou'` ทั้งคำแบบ exact match (ไม่พบ จึงคืนค่าเดิม) ในขณะที่ `TRANSLATE` แมปทีละตัวอักษร (`a`→`*`, `e`→`*`, `i`→`*` และอื่น ๆ) จึงได้ผลลัพธ์ที่ต่างกันโดยสิ้นเชิง

### REPEAT() — ทำซ้ำข้อความ

```sql
SELECT
    REPEAT('=', 20) AS divider_line,
    REPEAT('★', 4) || REPEAT('☆', 1) AS rating_stars_4_of_5;
```

**ผลลัพธ์:**
```
    divider_line      | rating_stars_4_of_5
------------------------+-----------------------
 ====================   | ★★★★☆
```

ตัวอย่างประยุกต์: แปลงคะแนนรีวิว (1-5) ให้เป็นรูปดาวข้อความ:

```sql
SELECT
    review_id,
    rating,
    REPEAT('★', rating) || REPEAT('☆', 5 - rating) AS stars
FROM reviews
WHERE review_id IN (1, 9, 12);
```

**ผลลัพธ์:**
```
 review_id | rating |  stars
-----------+--------+----------
         1 |      5 | ★★★★★
         9 |      1 | ★☆☆☆☆
        12 |      5 | ★★★★★
```

### REVERSE() — กลับลำดับข้อความ

```sql
SELECT
    product_name,
    REVERSE(product_name) AS reversed
FROM products
WHERE product_id = 13;
```

**ผลลัพธ์:**
```
        product_name         |          reversed
-------------------------------+-------------------------------
 Clean Code by Robert Martin   | nitraM treboR yb edoC naelC
```

`REVERSE` ใช้บ่อยในการตรวจสอบว่าข้อความเป็น **palindrome** หรือไม่ หรือใช้เป็นเทคนิคช่วยค้นหาข้อความที่ "ลงท้ายด้วย" รูปแบบบางอย่างโดยไม่ต้องใช้ regex (เช่น กลับข้อความแล้วเทียบกับ `LIKE 'pattern%'` แทนการหา `LIKE '%pattern'` เพื่อให้ใช้ index ได้ในบางกรณี)

---

## Step 305: SPLIT_PART และการแยกข้อความด้วย Delimiter, STRING_TO_ARRAY

### SPLIT_PART() — แยกข้อความตาม delimiter แล้วดึงส่วนที่ต้องการ

```sql
SELECT
    email,
    SPLIT_PART(email, '@', 1) AS username_part,
    SPLIT_PART(email, '@', 2) AS domain_part
FROM customers
WHERE customer_id IN (1, 8);
```

**ผลลัพธ์:**
```
           email           | username_part |  domain_part
-----------------------------+-----------------+----------------
 somchai.jaidee@email.com    | somchai.jaidee  | email.com
 aiko.tanaka@email.co.jp     | aiko.tanaka     | email.co.jp
```

สามารถซ้อน `SPLIT_PART` เพื่อแยกข้อมูลหลายชั้นได้ เช่น แยก username ที่คั่นด้วยจุด:

```sql
SELECT
    email,
    SPLIT_PART(SPLIT_PART(email, '@', 1), '.', 1) AS first_part_of_username,
    SPLIT_PART(SPLIT_PART(email, '@', 1), '.', 2) AS second_part_of_username
FROM customers
WHERE customer_id = 1;
```

**ผลลัพธ์:**
```
           email           | first_part_of_username | second_part_of_username
-----------------------------+---------------------------+----------------------------
 somchai.jaidee@email.com    | somchai                   | jaidee
```

> **ข้อควรระวัง:** ถ้าตำแหน่ง (index) ที่ระบุเกินจำนวนส่วนที่แยกได้จริง `SPLIT_PART` จะคืนค่าเป็น string ว่าง `''` (ไม่ใช่ `NULL`) เช่น `SPLIT_PART('a@b.com', '@', 3)` จะได้ `''`

### STRING_TO_ARRAY() — แยกข้อความทั้งหมดออกเป็น array

เมื่อจำนวนส่วนที่แยกได้ไม่คงที่ (ไม่รู้ล่วงหน้าว่ามีกี่ส่วน) `STRING_TO_ARRAY` เหมาะกว่า `SPLIT_PART` มาก เพราะคืนค่าทุกส่วนพร้อมกันในรูปแบบ array

```sql
SELECT
    product_name,
    STRING_TO_ARRAY(product_name, ' ') AS words_array,
    ARRAY_LENGTH(STRING_TO_ARRAY(product_name, ' '), 1) AS word_count
FROM products
WHERE product_id = 1;
```

**ผลลัพธ์:**
```
      product_name       |              words_array               | word_count
--------------------------+-------------------------------------------+-------------
 iPhone 15 Pro Max 256GB  | {iPhone,15,Pro,Max,256GB}                 |          5
```

การเข้าถึงสมาชิกใน array ทำได้ด้วย index (เริ่มที่ 1 เช่นเดียวกับ string):

```sql
SELECT
    product_name,
    (STRING_TO_ARRAY(product_name, ' '))[1] AS brand_or_first_word
FROM products
WHERE product_id IN (1, 2, 5);
```

**ผลลัพธ์:**
```
      product_name        | brand_or_first_word
---------------------------+-----------------------
 iPhone 15 Pro Max 256GB   | iPhone
 Samsung Galaxy S24 Ultra  | Samsung
 MacBook Air M2            | MacBook
```

### UNNEST() ร่วมกับ STRING_TO_ARRAY — แตกแต่ละส่วนเป็นแถว

```sql
SELECT
    product_id,
    product_name,
    UNNEST(STRING_TO_ARRAY(product_name, ' ')) AS word
FROM products
WHERE product_id = 15;
```

**ผลลัพธ์:**
```
 product_id |     product_name       |  word
------------+--------------------------+---------
         15 | Nike Air Zoom Pegasus    | Nike
         15 | Nike Air Zoom Pegasus    | Air
         15 | Nike Air Zoom Pegasus    | Zoom
         15 | Nike Air Zoom Pegasus    | Pegasus
```

### ARRAY_TO_STRING() — ทำสิ่งตรงกันข้าม รวม array กลับเป็นข้อความ

```sql
SELECT
    ARRAY_TO_STRING(ARRAY['สมชาย', 'สมหญิง', 'ประยุทธ'], ', ') AS combined_names;
```

**ผลลัพธ์:**
```
          combined_names
-------------------------------------
 สมชาย, สมหญิง, ประยุทธ
```

ตัวอย่างประยุกต์: รวมรายชื่อสินค้าทั้งหมดในแต่ละออเดอร์ให้อยู่ในบรรทัดเดียว (จะได้เรียนรู้ `STRING_AGG` ซึ่งเป็นฟังก์ชัน aggregate ที่ทำงานคล้ายกันในบทเรื่อง Aggregate Functions ขั้นสูงต่อไป) แต่แนวคิดการรวม array เป็น string แบบนี้ใช้บ่อยมากเมื่อทำงานกับผลลัพธ์ที่ผ่าน `ARRAY_AGG` มาก่อน:

```sql
SELECT
    ARRAY_TO_STRING(
        ARRAY_AGG(p.product_name ORDER BY p.product_name), ' + '
    ) AS products_in_order
FROM order_items oi
JOIN products p ON p.product_id = oi.product_id
WHERE oi.order_id = 15
GROUP BY oi.order_id;
```

**ผลลัพธ์:**
```
              products_in_order
-----------------------------------------------
 Nike Air Zoom Pegasus + Yoga Mat Premium 6mm
```

---

## Step 306: LPAD, RPAD — การจัดรูปแบบข้อความให้มีความยาวคงที่

`LPAD` (Left Pad) และ `RPAD` (Right Pad) ใช้เติมอักขระเพิ่มเข้าไปด้านซ้ายหรือขวาของข้อความ จนกว่าจะมีความยาวตามที่กำหนด — เหมาะมากสำหรับการสร้างรหัสที่มีความยาวคงที่ เช่น เลขที่ออเดอร์ รหัสพนักงาน หรือ invoice number

### Syntax พื้นฐาน

```sql
SELECT
    LPAD('7', 5, '0')   AS lpad_example,
    RPAD('7', 5, '0')   AS rpad_example,
    LPAD('ABC', 6, '*') AS lpad_with_star;
```

**ผลลัพธ์:**
```
 lpad_example | rpad_example | lpad_with_star
---------------+---------------+------------------
 00007         | 70000         | ***ABC
```

### ตัวอย่างประยุกต์: สร้างเลขที่ออเดอร์แบบมาตรฐาน (เช่น ORD-000001)

```sql
SELECT
    order_id,
    'ORD-' || LPAD(order_id::TEXT, 6, '0') AS order_code
FROM orders
WHERE order_id IN (1, 15, 18);
```

**ผลลัพธ์:**
```
 order_id |  order_code
----------+---------------
        1 | ORD-000001
       15 | ORD-000015
       18 | ORD-000018
```

### ตัวอย่างประยุกต์: สร้างรหัสพนักงานตามแผนก

```sql
SELECT
    employee_id,
    department,
    UPPER(LEFT(department, 3)) || '-' || LPAD(employee_id::TEXT, 4, '0') AS employee_code
FROM employees;
```

**ผลลัพธ์:**
```
 employee_id | department |  employee_code
-------------+------------+-------------------
           1 | Management | MAN-0001
           2 | Sales      | SAL-0002
           3 | Sales      | SAL-0003
           4 | Support    | SUP-0004
           5 | Sales      | SAL-0005
           6 | Support    | SUP-0006
           7 | Sales      | SAL-0007
           8 | Support    | SUP-0008
```

### ตัวอย่างประยุกต์: จัดคอลัมน์ให้ตรงกันเวลาแสดงผลแบบ fixed-width (เช่นทำรายงานแบบข้อความล้วน)

```sql
SELECT
    RPAD(product_name, 30) || RPAD(unit_price::TEXT, 12) || stock_quantity::TEXT AS report_line
FROM products
WHERE product_id IN (1, 13, 16)
ORDER BY product_id;
```

**ผลลัพธ์:**
```
                       report_line
-----------------------------------------------------------
 iPhone 15 Pro Max 256GB      42900.00    25
 Clean Code by Robert Martin  690.00      30
 Yoga Mat Premium 6mm         890.00      40
```

> **หมายเหตุ:** ถ้าข้อความต้นฉบับยาวกว่าความยาวที่กำหนดใน `LPAD`/`RPAD` อยู่แล้ว ฟังก์ชันจะ **ตัดข้อความให้สั้นลง** ให้พอดีกับความยาวที่ระบุ ไม่ใช่คงข้อความเดิมไว้ เช่น `LPAD('HelloWorld', 5)` จะได้ `'Hello'`

```sql
SELECT LPAD('HelloWorld', 5) AS truncated_result;
```

**ผลลัพธ์:**
```
 truncated_result
-------------------
 Hello
```

---

## Step 307: Pattern Matching ด้วย LIKE/ILIKE ทบทวนเชิงลึก, SIMILAR TO

### LIKE — จับคู่รูปแบบแบบพื้นฐาน (case-sensitive)

`LIKE` ใช้ไวลด์การ์ดสองตัว:
- `%` แทนอักขระ **กี่ตัวก็ได้** (รวมถึงศูนย์ตัว)
- `_` แทนอักขระ **หนึ่งตัวเท่านั้น**

```sql
SELECT product_name
FROM products
WHERE product_name LIKE 'M%';
```

**ผลลัพธ์:**
```
   product_name
--------------------
 MacBook Air M2
```

```sql
SELECT product_name
FROM products
WHERE product_name LIKE '%Air%';
```

**ผลลัพธ์:**
```
     product_name
------------------------
 MacBook Air M2
 Philips Air Fryer XXL
```

ใช้ `_` แทนอักขระเดี่ยว เช่น หารหัสสินค้าที่มีรูปแบบ 2 ตัวอักษรตามด้วยตัวเลข:

```sql
SELECT email
FROM customers
WHERE email LIKE '____@email.com';
```

**ผลลัพธ์:**
```
     email
----------------
 (ไม่มีแถวตรงกัน เพราะ username หน้า @ ของทุกคนยาวเกิน 4 ตัวอักษร)
```

### ILIKE — เหมือน LIKE แต่ไม่สนใจตัวพิมพ์ใหญ่-เล็ก (case-insensitive)

```sql
SELECT first_name, last_name, email
FROM customers
WHERE email ILIKE '%GMAIL%';
```

**ผลลัพธ์:**
```
 first_name |  last_name   |            email
------------+---------------+-------------------------------
 สุภาพร     | แก้วขาว       |   Supaporn.K@GMAIL.com
```

```sql
SELECT first_name, last_name
FROM customers
WHERE last_name ILIKE 'rattana';
```

**ผลลัพธ์:**
```
 first_name | last_name
------------+------------
 kanya      | RATTANA
```

### ESCAPE — จับคู่อักขระ % หรือ _ ตัวจริง ไม่ใช่ไวลด์การ์ด

ถ้าข้อมูลมีเครื่องหมาย `%` หรือ `_` ปนอยู่จริงและต้องการค้นหาตัวอักษรนั้นตรง ๆ ต้อง escape ด้วย backslash หรือกำหนดอักขระ escape เอง:

```sql
SELECT REPLACE('ลด 20% วันนี้', '%', '#pct#') AS demo_text
WHERE 'ลด 20% วันนี้' LIKE '%20\%%' ESCAPE '\';
```

**ผลลัพธ์:**
```
      demo_text
-----------------------
 ลด 20#pct# วันนี้
```

### SIMILAR TO — อยู่กึ่งกลางระหว่าง LIKE กับ Regular Expression เต็มรูปแบบ

`SIMILAR TO` รองรับ syntax คล้าย POSIX regex บางส่วน เช่น `|` (or), `*` (ซ้ำ 0 ครั้งขึ้นไป), `+` (ซ้ำ 1 ครั้งขึ้นไป), `{n,m}` (ซ้ำ n ถึง m ครั้ง) และ `()` (จัดกลุ่ม) ผสมกับไวลด์การ์ดสไตล์ `LIKE`

```sql
SELECT department
FROM employees
WHERE department SIMILAR TO 'Sales|Support';
```

**ผลลัพธ์:**
```
 department
------------
 Sales
 Sales
 Support
 Sales
 Support
 Sales
 Support
```

ตัวอย่างตรวจสอบว่าข้อความเป็นตัวเลขล้วนหรือไม่ (ทางเลือกแบบง่ายก่อนไปเรียน regex เต็มรูปแบบใน Step 308):

```sql
SELECT
    '0812345678' SIMILAR TO '[0-9]{10}'  AS is_10_digit_number,
    'ABC123'     SIMILAR TO '[0-9]{10}'  AS is_10_digit_number_2;
```

**ผลลัพธ์:**
```
 is_10_digit_number | is_10_digit_number_2
---------------------+------------------------
 t                   | f
```

### ตารางเปรียบเทียบ LIKE / ILIKE / SIMILAR TO

| เครื่องมือ | Case-sensitive | รองรับ regex เต็มรูปแบบ | ความเร็ว (ทั่วไป) |
|---|---|---|---|
| `LIKE` | ใช่ | ไม่ (เฉพาะ `%`, `_`) | เร็วที่สุด |
| `ILIKE` | ไม่ | ไม่ (เฉพาะ `%`, `_`) | เร็ว |
| `SIMILAR TO` | ใช่ | บางส่วน (POSIX subset) | ปานกลาง |
| `~` / `~*` (regex) | ใช่ / ไม่ | ใช่ เต็มรูปแบบ | ยืดหยุ่นสุดแต่ช้าสุด |

> **เคล็ดลับด้านประสิทธิภาพ:** `LIKE '%คำค้น%'` ที่มีไวลด์การ์ดนำหน้า (`%` อยู่ตำแหน่งแรก) **ไม่สามารถใช้ B-tree index มาตรฐานได้** เพราะ index แบบ B-tree เรียงลำดับจากตัวอักษรแรก ถ้าต้องค้นหาข้อความแบบ "มีคำนี้อยู่ตรงไหนก็ได้" บ่อยๆ บนตารางขนาดใหญ่ ควรพิจารณาใช้ extension `pg_trgm` (`CREATE EXTENSION pg_trgm;`) แล้วสร้าง GIN index แบบ `CREATE INDEX ON products USING GIN (product_name gin_trgm_ops);` ซึ่งจะเร่งความเร็วของ `LIKE '%...%'` และ `ILIKE` ได้อย่างมาก — หัวข้อนี้จะลงรายละเอียดในบท Indexing ขั้นสูง

---

## Step 308: Regular Expression Functions — REGEXP_MATCH, REGEXP_MATCHES, REGEXP_REPLACE, REGEXP_SPLIT_TO_TABLE/ARRAY

PostgreSQL รองรับ POSIX regular expression แบบเต็มรูปแบบ ทำให้จับคู่และดึงข้อมูลจากข้อความที่ซับซ้อนได้อย่างทรงพลัง

### Operator พื้นฐานสำหรับ regex: `~`, `~*`, `!~`, `!~*`

| Operator | ความหมาย |
|---|---|
| `~` | ตรงกับรูปแบบ (case-sensitive) |
| `~*` | ตรงกับรูปแบบ (case-insensitive) |
| `!~` | ไม่ตรงกับรูปแบบ (case-sensitive) |
| `!~*` | ไม่ตรงกับรูปแบบ (case-insensitive) |

```sql
SELECT product_name
FROM products
WHERE product_name ~ '^[A-Z]';
```

**ผลลัพธ์:** (สินค้าทุกตัวขึ้นต้นด้วยตัวพิมพ์ใหญ่ จึงได้ผลลัพธ์ทั้ง 18 แถว — แสดงเฉพาะบางส่วน)
```
      product_name
--------------------------
 iPhone 15 Pro Max 256GB
 Samsung Galaxy S24 Ultra
 ...
```

ตัวอย่างที่เห็นผลชัดกว่า: หารีวิวที่มีตัวเลขปนอยู่ในข้อความ (เช่นเบอร์โทรศัพท์):

```sql
SELECT review_id, review_text
FROM reviews
WHERE review_text ~ '[0-9]{3}-[0-9]{3}-[0-9]{4}';
```

**ผลลัพธ์:**
```
 review_id |                                         review_text
-----------+------------------------------------------------------------------------------------------------
         9 | DO NOT BUY!!! โต๊ะพังภายใน 2 สัปดาห์ ติดต่อร้านไม่ได้เลย โทร 081-234-5678 ก็ไม่มีคนรับสาย
```

### REGEXP_MATCH() — ดึงข้อความที่ตรงกับรูปแบบครั้งแรกที่พบ (คืนค่า text[] หนึ่งแถว หรือ NULL)

```sql
SELECT
    review_id,
    REGEXP_MATCH(review_text, '[0-9]{3}-[0-9]{3}-[0-9]{4}') AS phone_found
FROM reviews
WHERE review_id = 9;
```

**ผลลัพธ์:**
```
 review_id |   phone_found
-----------+------------------
         9 | {081-234-5678}
```

ถ้าต้องการดึงหลาย group พร้อมกัน ใช้วงเล็บ `()` เพื่อจับกลุ่ม (capturing group):

```sql
SELECT
    review_id,
    REGEXP_MATCH(review_text, '(\w+@\w+\.\w+)') AS email_found
FROM reviews
WHERE review_id IN (4, 13);
```

**ผลลัพธ์:**
```
 review_id |     email_found
-----------+-----------------------
         4 | {john.smith@email.com}
        13 | {support@shop.com}
```

### REGEXP_MATCHES() — คืนค่าได้หลายแถวเมื่อใช้ flag 'g' (global)

`REGEXP_MATCH` (เอกพจน์) คืนแค่ผลลัพธ์ที่ตรงกัน **ครั้งแรก** เท่านั้น ในขณะที่ `REGEXP_MATCHES` (พหูพจน์) เมื่อใช้ร่วมกับ flag `'g'` จะคืน **ทุกครั้ง** ที่ตรงกันในรูปแบบ set of rows:

```sql
SELECT
    review_id,
    REGEXP_MATCHES(review_text, '[A-Z]{2,}', 'g') AS shouting_words
FROM reviews
WHERE review_id = 9;
```

**ผลลัพธ์:**
```
 review_id | shouting_words
-----------+------------------
         9 | {DO}
         9 | {NOT}
         9 | {BUY}
```

จะเห็นว่ารีวิว #9 ที่มีคำว่า "DO NOT BUY!!!" ถูกดึงคำที่พิมพ์ตัวใหญ่ทั้งหมด (2 ตัวอักษรขึ้นไป) ออกมาทีละคำ เป็น 3 แถว

### REGEXP_REPLACE() — แทนที่ข้อความด้วยรูปแบบ regex

Syntax: `REGEXP_REPLACE(source, pattern, replacement [, flags])`

```sql
SELECT
    review_text,
    REGEXP_REPLACE(review_text, '\s+', ' ', 'g') AS single_spaced
FROM reviews
WHERE review_id = 6;
```

**ผลลัพธ์:**
```
                review_text                |            single_spaced
---------------------------------------------+----------------------------------------
 ทอดไม่ค่อยกรอบเท่าที่ควร  ราคาแพงไปหน่อย     | ทอดไม่ค่อยกรอบเท่าที่ควร ราคาแพงไปหน่อย
```

`\s+` หมายถึง "ช่องว่างหนึ่งตัวขึ้นไป (รวม tab, newline)" และ flag `'g'` หมายถึง "แทนที่ **ทุก** ตำแหน่งที่พบ" (ถ้าไม่ใส่ `'g'` จะแทนที่แค่ตำแหน่งแรก)

ตัวอย่างการปกปิดข้อมูลอ่อนไหว (masking) เช่น ซ่อนเบอร์โทรในรีวิว:

```sql
SELECT
    review_id,
    REGEXP_REPLACE(review_text, '[0-9]{3}-[0-9]{3}-[0-9]{4}', 'XXX-XXX-XXXX', 'g') AS masked_review
FROM reviews
WHERE review_id = 9;
```

**ผลลัพธ์:**
```
 review_id |                                          masked_review
-----------+---------------------------------------------------------------------------------------------------
         9 | DO NOT BUY!!! โต๊ะพังภายใน 2 สัปดาห์ ติดต่อร้านไม่ได้เลย โทร XXX-XXX-XXXX ก็ไม่มีคนรับสาย
```

ตัวอย่างลบ HTML tag ทั้งหมดด้วย regex (ยืดหยุ่นกว่า `REPLACE` ธรรมดาใน Step 304 มาก เพราะไม่ต้องรู้ tag ล่วงหน้า):

```sql
SELECT REGEXP_REPLACE('<b>ดีมาก</b> <i>ราคาคุ้ม</i>', '<[^>]+>', '', 'g') AS text_only;
```

**ผลลัพธ์:**
```
   text_only
-----------------
 ดีมาก ราคาคุ้ม
```

### REGEXP_SPLIT_TO_TABLE() / REGEXP_SPLIT_TO_ARRAY() — แยกข้อความด้วย regex pattern

ต่างจาก `STRING_TO_ARRAY` (Step 305) ตรงที่ delimiter เป็น **regular expression** ไม่ใช่ตัวอักษรตายตัว จึงยืดหยุ่นกว่าเมื่อ delimiter ไม่คงที่ (เช่น ช่องว่างจำนวนไม่เท่ากัน หรือมีทั้ง comma และ semicolon ปน)

```sql
SELECT REGEXP_SPLIT_TO_TABLE('สมชาย,   สมหญิง;ประยุทธ  ,Wei', '\s*[,;]\s*') AS name;
```

**ผลลัพธ์:**
```
   name
-----------
 สมชาย
 สมหญิง
 ประยุทธ
 Wei
```

`\s*[,;]\s*` หมายถึง "ช่องว่าง (ถ้ามี) ตามด้วยเครื่องหมาย comma หรือ semicolon ตามด้วยช่องว่าง (ถ้ามี)" — จัดการทั้งรูปแบบตัวคั่นและช่องว่างที่ไม่สม่ำเสมอได้ในครั้งเดียว

```sql
SELECT
    review_text,
    REGEXP_SPLIT_TO_ARRAY(TRIM(review_text), '\s+') AS words
FROM reviews
WHERE review_id = 10;
```

**ผลลัพธ์:**
```
                    review_text                      |                    words
-------------------------------------------------------+-----------------------------------------------
 headphones  are   AMAZING, noise cancelling works... | {headphones,are,AMAZING,,noise,cancelling,...}
```

### ตารางสรุป Regex Functions

| ฟังก์ชัน | คืนค่า | ใช้เมื่อ |
|---|---|---|
| `REGEXP_MATCH(str, pattern)` | `text[]` หรือ `NULL` (แถวเดียว) | ต้องการผลลัพธ์ที่ตรงกันครั้งแรกเท่านั้น |
| `REGEXP_MATCHES(str, pattern, 'g')` | หลายแถว (`text[]` ต่อแถว) | ต้องการทุกตำแหน่งที่ตรงกัน |
| `REGEXP_REPLACE(str, pattern, repl, 'g')` | `text` | แทนที่ข้อความตามรูปแบบ |
| `REGEXP_SPLIT_TO_TABLE(str, pattern)` | หลายแถว (`text` ต่อแถว) | แยกข้อความเป็นแถว |
| `REGEXP_SPLIT_TO_ARRAY(str, pattern)` | `text[]` | แยกข้อความเป็น array |
| `str ~ pattern` | `boolean` | ทดสอบว่าตรงรูปแบบหรือไม่ (case-sensitive) |
| `str ~* pattern` | `boolean` | ทดสอบว่าตรงรูปแบบหรือไม่ (case-insensitive) |

---

## Step 309: การ Validate รูปแบบข้อมูลด้วย Regex ใน CHECK Constraint

การใช้ `CHECK` constraint ร่วมกับ regex เป็นวิธีที่มีประสิทธิภาพมากในการบังคับให้ข้อมูลที่เข้าสู่ตารางมีรูปแบบถูกต้องตั้งแต่ระดับฐานข้อมูล (ไม่ต้องพึ่งการตรวจสอบที่ชั้น Application เพียงอย่างเดียว)

### ทดลอง Validate อีเมลของลูกค้า

ก่อนอื่น ลองเพิ่ม constraint ตรวจสอบอีเมลเข้าไปในตาราง `customers` ทันที:

```sql
ALTER TABLE customers
    ADD CONSTRAINT chk_customers_email_format
    CHECK (email ~* '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$');
```

**ผลลัพธ์:**
```
ERROR:  check constraint "chk_customers_email_format" of relation "customers" is violated by some row
```

คำสั่งล้มเหลว เพราะมีข้อมูลเก่าที่ไม่ผ่านเงื่อนไขอยู่แล้ว (ลูกค้า Nok Srisawat ที่มีอีเมล `'invalid-email-format'`) นี่คือสถานการณ์ที่พบบ่อยมากในงานจริง — ต้อง **หาและแก้ไขข้อมูลเก่าก่อน** จึงจะเพิ่ม constraint ได้สำเร็จ

### ขั้นที่ 1: หาแถวที่ไม่ผ่านเงื่อนไข

```sql
SELECT customer_id, first_name, last_name, email
FROM customers
WHERE email !~* '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$';
```

**ผลลัพธ์:**
```
 customer_id | first_name | last_name |         email
-------------+------------+-----------+------------------------
          13 | Nok        | Srisawat  | invalid-email-format
```

### ขั้นที่ 2: แก้ไขข้อมูล (สมมติติดต่อลูกค้าเพื่อขอที่อยู่อีเมลที่ถูกต้องแล้ว)

```sql
UPDATE customers
SET email = 'nok.srisawat@email.com'
WHERE customer_id = 13;
```

### ขั้นที่ 3: เพิ่ม constraint อีกครั้ง

```sql
ALTER TABLE customers
    ADD CONSTRAINT chk_customers_email_format
    CHECK (email ~* '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$');
```

**ผลลัพธ์:**
```
ALTER TABLE
```

คราวนี้สำเร็จ เพราะข้อมูลทุกแถวผ่านเงื่อนไขแล้ว ทดสอบว่า constraint ทำงานจริงโดยลอง insert ข้อมูลผิดรูปแบบ:

```sql
INSERT INTO customers (first_name, last_name, email, country)
VALUES ('Test', 'User', 'not-an-email', 'Thailand');
```

**ผลลัพธ์:**
```
ERROR:  new row for relation "customers" violates check constraint "chk_customers_email_format"
DETAIL:  Failing row contains (17, Test, User, not-an-email, Thailand, 2026-09-25).
```

### ตัวอย่างเพิ่มเติม: Validate เบอร์โทรศัพท์มือถือไทย

เบอร์โทรศัพท์มือถือไทยมีรูปแบบมาตรฐานคือ ขึ้นต้นด้วย `0` ตามด้วยตัวเลข `6`, `8`, หรือ `9` แล้วตามด้วยตัวเลขอีก 8 หลัก (รวม 10 หลัก) ลองเพิ่มคอลัมน์ทดลองเพื่อสาธิต:

```sql
ALTER TABLE customers ADD COLUMN phone VARCHAR(15);

ALTER TABLE customers
    ADD CONSTRAINT chk_customers_phone_format
    CHECK (phone IS NULL OR phone ~ '^0[689][0-9]{8}$');

-- ทดสอบด้วยเบอร์ที่ถูกต้อง
UPDATE customers SET phone = '0812345678' WHERE customer_id = 1;

-- ทดสอบด้วยเบอร์ที่มีรูปแบบผิด (มีขีดคั่น)
UPDATE customers SET phone = '081-234-5678' WHERE customer_id = 2;
```

**ผลลัพธ์ของคำสั่งที่สอง:**
```
ERROR:  new row for relation "customers" violates check constraint "chk_customers_phone_format"
DETAIL:  Failing row contains (2, สุภาพร, แก้วขาว, ..., 081-234-5678).
```

หากต้องการรองรับเบอร์ที่มีขีดคั่นด้วย สามารถใช้ `REGEXP_REPLACE` เพื่อลบขีดคั่นออกก่อนตรวจสอบภายใน constraint เดียวกัน:

```sql
ALTER TABLE customers DROP CONSTRAINT chk_customers_phone_format;

ALTER TABLE customers
    ADD CONSTRAINT chk_customers_phone_format
    CHECK (phone IS NULL OR REGEXP_REPLACE(phone, '[-\s]', '', 'g') ~ '^0[689][0-9]{8}$');

-- ตอนนี้เบอร์ที่มีขีดคั่นก็ผ่านได้แล้ว
UPDATE customers SET phone = '081-234-5678' WHERE customer_id = 2;
```

**ผลลัพธ์:**
```
UPDATE 1
```

ทำความสะอาดคอลัมน์ทดลองก่อนไปต่อ:

```sql
ALTER TABLE customers DROP CONSTRAINT chk_customers_phone_format;
ALTER TABLE customers DROP COLUMN phone;
```

> **เคล็ดลับระดับมืออาชีพ:** การตรวจสอบรูปแบบข้อมูลควรทำทั้งสองชั้น — ชั้น Application (เพื่อ UX ที่ดี แจ้งเตือนผู้ใช้ได้ทันที) และชั้นฐานข้อมูลด้วย `CHECK constraint` (เพื่อเป็นด่านสุดท้ายที่รับประกันความถูกต้องของข้อมูล ไม่ว่าข้อมูลจะถูกใส่เข้ามาทางไหนก็ตาม เช่น ผ่าน script, migration หรือระบบอื่นที่เชื่อมต่อฐานข้อมูลโดยตรง) นอกจากนี้ regex ที่ซับซ้อนมากอาจกระทบ performance ตอน insert/update จำนวนมาก ควรทดสอบ `EXPLAIN ANALYZE` เทียบก่อนนำไปใช้กับตารางขนาดใหญ่จริง

---

## Step 310: แบบฝึกหัดรวม — ทำความสะอาดและแปลงข้อมูลข้อความสกปรกในระบบ E-commerce (Data Cleaning)

ขั้นตอนนี้จะรวมฟังก์ชันทั้งหมดที่เรียนมาในบทนี้ เพื่อแก้ปัญหาข้อมูลสกปรกจริงในชุดข้อมูลของเรา

### ปัญหาที่ 1: ชื่อลูกค้าที่มีช่องว่างเกินและตัวพิมพ์ไม่สม่ำเสมอ

```sql
SELECT
    customer_id,
    '[' || first_name || ']' AS original_first_name,
    INITCAP(TRIM(first_name)) AS cleaned_first_name,
    '[' || last_name || ']'  AS original_last_name,
    INITCAP(TRIM(last_name)) AS cleaned_last_name
FROM customers
WHERE customer_id IN (2, 4, 5, 7, 11, 15);
```

**ผลลัพธ์:**
```
 customer_id | original_first_name | cleaned_first_name | original_last_name | cleaned_last_name
-------------+-----------------------+----------------------+----------------------+---------------------
           2 | [สุภาพร]              | สุภาพร               | [แก้วขาว]            | แก้วขาว
           4 | [JOHN]                | John                 | [DOE]                | Doe
           5 | [  Maria  ]           | Maria                | [Garcia]             | Garcia
           7 | [David]               | David                | [lee]                | Lee
          11 | [kanya]               | Kanya                | [RATTANA]            | Rattana
          15 | [  Tom ]              | Tom                  | [ Anderson ]         | Anderson
```

### ปัญหาที่ 2: อีเมลที่มีช่องว่างและตัวพิมพ์ใหญ่ปน (ทำให้เทียบข้อมูลซ้ำไม่เจอ)

```sql
SELECT
    customer_id,
    email AS original_email,
    LOWER(TRIM(email)) AS cleaned_email
FROM customers
WHERE customer_id = 2;
```

**ผลลัพธ์:**
```
 customer_id |              original_email              |    cleaned_email
-------------+--------------------------------------------+---------------------------
           2 |   Supaporn.K@GMAIL.com                      | supaporn.k@gmail.com
```

### ปัญหาที่ 3: ข้อความรีวิวที่มีช่องว่างซ้ำและช่องว่างหัว-ท้าย

```sql
SELECT
    review_id,
    '[' || review_text || ']' AS original,
    REGEXP_REPLACE(TRIM(review_text), '\s+', ' ', 'g') AS cleaned
FROM reviews
WHERE review_id IN (1, 2, 3, 10);
```

**ผลลัพธ์:**
```
 review_id |                          original                           |                         cleaned
-----------+---------------------------------------------------------------+---------------------------------------------
         1 | [สินค้าดีมากค่ะ!!! ส่งไวสุดๆ   แนะนำเลยค่ะ]                    | สินค้าดีมากค่ะ!!! ส่งไวสุดๆ แนะนำเลยค่ะ
         2 | [GREAT PHONE!!! battery life is amazing, but a bit pricey.   ]| GREAT PHONE!!! battery life is amazing...
         3 | [  ใช้งานดีมาก คุ้มค่ากับราคา ]                                | ใช้งานดีมาก คุ้มค่ากับราคา
        10 | [headphones  are   AMAZING, noise cancelling works perfectly!!!]| headphones are AMAZING, noise cancelling...
```

### ปัญหาที่ 4: ดึงข้อมูลติดต่อ (อีเมล/เบอร์โทร) ที่แอบปนอยู่ในข้อความรีวิว เพื่อตรวจสอบนโยบายความเป็นส่วนตัว

```sql
SELECT
    review_id,
    REGEXP_MATCH(review_text, '[\w.+-]+@[\w-]+\.[a-zA-Z]{2,}') AS email_in_review,
    REGEXP_MATCH(review_text, '[0-9]{3}-[0-9]{3}-[0-9]{4}')     AS phone_in_review
FROM reviews
WHERE review_text ~ '[\w.+-]+@[\w-]+\.[a-zA-Z]{2,}'
   OR review_text ~ '[0-9]{3}-[0-9]{3}-[0-9]{4}';
```

**ผลลัพธ์:**
```
 review_id |     email_in_review     |  phone_in_review
-----------+---------------------------+--------------------
         4 | {john.smith@email.com}   | (null)
         9 | (null)                    | {081-234-5678}
        13 | {support@shop.com}        | (null)
```

### ปัญหาที่ 5: สร้าง VIEW ข้อมูลลูกค้าที่สะอาดแล้ว พร้อมข้อความปกปิดข้อมูลติดต่อในรีวิว

```sql
CREATE OR REPLACE VIEW customers_clean AS
SELECT
    customer_id,
    INITCAP(TRIM(first_name)) AS first_name,
    INITCAP(TRIM(last_name))  AS last_name,
    LOWER(TRIM(email))        AS email,
    TRIM(country)             AS country,
    signup_date
FROM customers;

SELECT * FROM customers_clean ORDER BY customer_id LIMIT 5;
```

**ผลลัพธ์:**
```
 customer_id | first_name | last_name |           email            | country  | signup_date
-------------+-------------+------------+-----------------------------+----------+---------------
           1 | สมชาย       | ใจดี       | somchai.jaidee@email.com    | Thailand | 2022-01-10
           2 | สุภาพร      | แก้วขาว    | supaporn.k@gmail.com        | Thailand | 2022-02-15
           3 | John        | Smith      | john.smith@email.com        | USA      | 2022-03-01
           4 | John        | Doe        | john.doe@email.com          | USA      | 2022-03-10
           5 | Maria       | Garcia     | maria.garcia@email.com      | Spain    | 2022-04-05
```

```sql
CREATE OR REPLACE VIEW reviews_masked AS
SELECT
    review_id,
    product_id,
    customer_id,
    rating,
    REGEXP_REPLACE(
        REGEXP_REPLACE(
            REGEXP_REPLACE(TRIM(review_text), '\s+', ' ', 'g'),
            '[\w.+-]+@[\w-]+\.[a-zA-Z]{2,}', '[อีเมลถูกซ่อน]', 'g'
        ),
        '[0-9]{3}-[0-9]{3}-[0-9]{4}', '[เบอร์โทรถูกซ่อน]', 'g'
    ) AS review_text_safe,
    review_date
FROM reviews;

SELECT review_id, review_text_safe
FROM reviews_masked
WHERE review_id IN (4, 9, 13);
```

**ผลลัพธ์:**
```
 review_id |                                              review_text_safe
-----------+-----------------------------------------------------------------------------------------------------------------
         4 | Good laptop overall, screen could be brighter. Contact me at [อีเมลถูกซ่อน] if you have questions.
         9 | DO NOT BUY!!! โต๊ะพังภายใน 2 สัปดาห์ ติดต่อร้านไม่ได้เลย โทร [เบอร์โทรถูกซ่อน] ก็ไม่มีคนรับสาย
        13 | Tablet ok but battery drains fast. Support email: [อีเมลถูกซ่อน]
```

ตัวอย่างนี้แสดงให้เห็นพลังของการนำ String Functions หลายตัวมาผสมกัน (nested functions) เพื่อแก้ปัญหาข้อมูลสกปรกในสถานการณ์จริงได้อย่างครบวงจร ตั้งแต่การตัดช่องว่าง ปรับตัวพิมพ์ จนถึงการค้นหาและปกปิดข้อมูลอ่อนไหวด้วย regex

---

## สรุปท้ายบท

### ตารางสรุปฟังก์ชันข้อความทั้งหมดที่เรียนในบทนี้

| ฟังก์ชัน / Operator | หมวดหมู่ | หน้าที่ | ตัวอย่าง |
|---|---|---|---|
| `LENGTH(str)` | พื้นฐาน | นับความยาว (ตัวอักษร) | `LENGTH('สมชาย')` → `5` |
| `OCTET_LENGTH(str)` | พื้นฐาน | นับความยาว (ไบต์) | `OCTET_LENGTH('สมชาย')` → `15` |
| `UPPER(str)` | พื้นฐาน | แปลงเป็นตัวพิมพ์ใหญ่ | `UPPER('abc')` → `'ABC'` |
| `LOWER(str)` | พื้นฐาน | แปลงเป็นตัวพิมพ์เล็ก | `LOWER('ABC')` → `'abc'` |
| `INITCAP(str)` | พื้นฐาน | ตัวแรกของแต่ละคำเป็นตัวใหญ่ | `INITCAP('john doe')` → `'John Doe'` |
| `TRIM(str)` | พื้นฐาน | ตัดช่องว่างหัว-ท้าย | `TRIM('  a  ')` → `'a'` |
| `LTRIM(str)` / `RTRIM(str)` | พื้นฐาน | ตัดช่องว่างซ้าย/ขวาเท่านั้น | `LTRIM('  a')` → `'a'` |
| `SUBSTRING(str FROM n FOR m)` | ตัด/ค้นหา | ดึงข้อความบางส่วน | `SUBSTRING('hello' FROM 2 FOR 3)` → `'ell'` |
| `LEFT(str, n)` / `RIGHT(str, n)` | ตัด/ค้นหา | ดึงจากซ้าย/ขวา n ตัว | `LEFT('hello', 3)` → `'hel'` |
| `POSITION(sub IN str)` / `STRPOS(str, sub)` | ตัด/ค้นหา | หาตำแหน่งข้อความย่อย | `STRPOS('hello', 'l')` → `3` |
| `CONCAT(a, b, ...)` | ต่อข้อความ | ต่อข้อความ ข้าม NULL | `CONCAT('a', NULL, 'b')` → `'ab'` |
| `CONCAT_WS(sep, a, b, ...)` | ต่อข้อความ | ต่อข้อความพร้อมตัวคั่น ข้าม NULL | `CONCAT_WS('-', 'a', NULL, 'b')` → `'a-b'` |
| `\|\|` | ต่อข้อความ | ต่อข้อความ (NULL ทำให้ทั้งหมดเป็น NULL) | `'a' \|\| NULL` → `NULL` |
| `REPLACE(str, from, to)` | แทนที่ | แทนที่ข้อความย่อยทั้งหมด | `REPLACE('aXbXc','X','-')` → `'a-b-c'` |
| `TRANSLATE(str, from_chars, to_chars)` | แทนที่ | แทนที่ทีละตัวอักษร | `TRANSLATE('abc','ac','13')` → `'1b3'` |
| `REPEAT(str, n)` | แทนที่ | ทำซ้ำข้อความ | `REPEAT('ab', 3)` → `'ababab'` |
| `REVERSE(str)` | แทนที่ | กลับลำดับข้อความ | `REVERSE('abc')` → `'cba'` |
| `SPLIT_PART(str, delim, n)` | แยกข้อความ | ดึงส่วนที่ n หลังแยกด้วย delimiter | `SPLIT_PART('a@b','@',2)` → `'b'` |
| `STRING_TO_ARRAY(str, delim)` | แยกข้อความ | แยกข้อความทั้งหมดเป็น array | `STRING_TO_ARRAY('a,b,c',',')` → `{a,b,c}` |
| `ARRAY_TO_STRING(arr, sep)` | แยกข้อความ | รวม array เป็นข้อความ | `ARRAY_TO_STRING(ARRAY['a','b'],'-')` → `'a-b'` |
| `LPAD(str, n, pad)` | จัดรูปแบบ | เติมด้านซ้ายให้ครบ n ตัว | `LPAD('7',3,'0')` → `'007'` |
| `RPAD(str, n, pad)` | จัดรูปแบบ | เติมด้านขวาให้ครบ n ตัว | `RPAD('7',3,'0')` → `'700'` |
| `LIKE` / `ILIKE` | Pattern matching | จับคู่แบบ wildcard (`%`, `_`) | `'abc' LIKE 'a%'` → `true` |
| `SIMILAR TO` | Pattern matching | จับคู่แบบ wildcard + POSIX subset | `'abc' SIMILAR TO 'a(b\|x)c'` → `true` |
| `~` / `~*` / `!~` / `!~*` | Regex | ทดสอบว่าตรงกับ regex หรือไม่ | `'abc123' ~ '[0-9]+'` → `true` |
| `REGEXP_MATCH(str, pattern)` | Regex | ดึงผลลัพธ์ที่ตรงกันครั้งแรก | คืนค่า `text[]` |
| `REGEXP_MATCHES(str, pattern, 'g')` | Regex | ดึงผลลัพธ์ที่ตรงกันทุกครั้ง | คืนค่าหลายแถว |
| `REGEXP_REPLACE(str, pattern, repl, 'g')` | Regex | แทนที่ตามรูปแบบ regex | คืนค่า `text` |
| `REGEXP_SPLIT_TO_TABLE/ARRAY` | Regex | แยกข้อความด้วย regex pattern | คืนค่าแถว/array |

### สิ่งที่ควรจำขึ้นใจ

1. **ตำแหน่งใน string เริ่มนับที่ 1** ไม่ใช่ 0 (`SUBSTRING`, `POSITION`, array index)
2. **`\|\|` เข้มงวดกับ NULL แต่ `CONCAT`/`CONCAT_WS` ไม่เข้มงวด** — เลือกให้เหมาะกับสถานการณ์
3. **`POSITION`/`STRPOS` คืนค่า `0` เมื่อหาไม่พบ ไม่ใช่ `NULL`**
4. **`LIKE '%...'` ที่มีไวลด์การ์ดนำหน้าใช้ index ปกติไม่ได้** ควรพิจารณา `pg_trgm` สำหรับตารางขนาดใหญ่
5. **`CHECK constraint` + regex คือด่านสุดท้ายในการควบคุมคุณภาพข้อมูล** ควรใช้ร่วมกับการตรวจสอบที่ชั้น Application
6. **ผสมฟังก์ชันหลายตัวเข้าด้วยกัน (nested functions) คือกุญแจสำคัญของการทำ data cleaning ในงานจริง**

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1
เขียนคำสั่งแสดงชื่อเต็ม (first_name + last_name) ของลูกค้าทุกคน โดยใช้ `CONCAT_WS` และแปลงให้เป็นรูปแบบ Title Case (ตัวแรกของแต่ละคำเป็นตัวใหญ่) พร้อมตัดช่องว่างส่วนเกินออกก่อน

<details>
<summary>เฉลย</summary>

```sql
SELECT
    customer_id,
    CONCAT_WS(' ', INITCAP(TRIM(first_name)), INITCAP(TRIM(last_name))) AS full_name
FROM customers
ORDER BY customer_id;
```

ใช้ `TRIM` ตัดช่องว่างส่วนเกินก่อน จากนั้นใช้ `INITCAP` ปรับตัวพิมพ์ แล้วค่อยต่อด้วย `CONCAT_WS` เพื่อความปลอดภัยจาก NULL (เช่นกรณี `last_name` เป็น NULL ในอนาคต)
</details>

### แบบฝึกหัดที่ 2
หาสินค้าทั้งหมดที่ชื่อมีคำว่า "Book" อยู่ในชื่อ (ไม่สนใจตัวพิมพ์ใหญ่-เล็ก)

<details>
<summary>เฉลย</summary>

```sql
SELECT product_id, product_name
FROM products
WHERE product_name ILIKE '%book%';
```

**ผลลัพธ์:**
```
 product_id |  product_name
------------+------------------
          5 | MacBook Air M2
         13 | (ไม่ตรง เพราะไม่มีคำว่า book)
```

ผลลัพธ์จริงจะมีเฉพาะ `MacBook Air M2` (product_id 5) เพราะเป็นสินค้าเดียวที่มีคำว่า "book" ปรากฏอยู่ในชื่อ (ใน "Mac**Book**")
</details>

### แบบฝึกหัดที่ 3
สร้างคอลัมน์ `order_code` จากตาราง `orders` ในรูปแบบ `ORD-YYYYMMDD-000001` โดยใช้เลขที่ order พร้อม padding และรวมวันที่สั่งซื้อ (ใช้ `TO_CHAR` สำหรับแปลงวันที่ — ฟังก์ชันนี้จะสอนละเอียดในบท Date/Time ถัดไป แต่ใช้ตัวอย่างนี้ได้เลย)

<details>
<summary>เฉลย</summary>

```sql
SELECT
    order_id,
    'ORD-' || TO_CHAR(order_date, 'YYYYMMDD') || '-' || LPAD(order_id::TEXT, 6, '0') AS order_code
FROM orders
ORDER BY order_id
LIMIT 5;
```

**ผลลัพธ์:**
```
 order_id |      order_code
----------+------------------------
        1 | ORD-20240115-000001
        2 | ORD-20240118-000002
        3 | ORD-20240120-000003
        4 | ORD-20240202-000004
        5 | ORD-20240205-000005
```
</details>

### แบบฝึกหัดที่ 4
ดึงเฉพาะโดเมนอีเมล (ส่วนหลัง `@`) ของลูกค้าทุกคน แล้วนับจำนวนลูกค้าต่อโดเมน เรียงจากมากไปน้อย

<details>
<summary>เฉลย</summary>

```sql
SELECT
    LOWER(SPLIT_PART(TRIM(email), '@', 2)) AS email_domain,
    COUNT(*) AS customer_count
FROM customers
WHERE email LIKE '%@%'
GROUP BY LOWER(SPLIT_PART(TRIM(email), '@', 2))
ORDER BY customer_count DESC, email_domain;
```

**ผลลัพธ์ (บางส่วน):**
```
 email_domain  | customer_count
----------------+-----------------
 email.com      |              9
 gmail.com      |              1
 email.co.jp    |              2
 email.cn       |              1
```

ใช้ `WHERE email LIKE '%@%'` เพื่อกรองอีเมลที่ไม่มี `@` ออกก่อน (เช่นก่อนแก้ไขข้อมูลใน Step 309) ป้องกัน `SPLIT_PART` คืนค่า string ว่าง
</details>

### แบบฝึกหัดที่ 5
เขียนคำสั่งแทนที่คำหยาบหรือคำต้องห้าม (สมมติคือคำว่า "DO NOT BUY") ในข้อความรีวิวด้วยคำว่า "[ข้อความถูกลบ]" โดยไม่สนใจตัวพิมพ์ใหญ่-เล็ก

<details>
<summary>เฉลย</summary>

```sql
SELECT
    review_id,
    REGEXP_REPLACE(review_text, 'DO NOT BUY', '[ข้อความถูกลบ]', 'gi') AS filtered_text
FROM reviews
WHERE review_id = 9;
```

**ผลลัพธ์:**
```
 review_id |                                         filtered_text
-----------+---------------------------------------------------------------------------------------------------
         9 | [ข้อความถูกลบ]!!! โต๊ะพังภายใน 2 สัปดาห์ ติดต่อร้านไม่ได้เลย โทร 081-234-5678 ก็ไม่มีคนรับสาย
```

flag `'gi'` รวม `g` (global — แทนที่ทุกตำแหน่ง) และ `i` (case-insensitive) เข้าด้วยกัน
</details>

### แบบฝึกหัดที่ 6
หาพนักงานทุกคนที่นามสกุลมีความยาวมากกว่า 6 ตัวอักษร แล้วแสดงนามสกุลพร้อมความยาว เรียงจากยาวไปสั้น

<details>
<summary>เฉลย</summary>

```sql
SELECT
    first_name,
    last_name,
    LENGTH(last_name) AS name_length
FROM employees
WHERE LENGTH(last_name) > 6
ORDER BY name_length DESC;
```

**ผลลัพธ์:**
```
 first_name |  last_name   | name_length
------------+---------------+--------------
 พลอย       | โชติรสรานี   |           10
 ณภัทร      | อินทรคำหาญ  |            9
 อนงค์      | สุขสวัสดิ์    |            8
```
</details>

### แบบฝึกหัดที่ 7
ใช้ `STRING_TO_ARRAY` และ `UNNEST` เพื่อแสดงคำทุกคำที่ปรากฏในชื่อสินค้าทั้งหมด พร้อมนับความถี่ (frequency) ของแต่ละคำ เรียงจากมากไปน้อย แสดง 5 อันดับแรก

<details>
<summary>เฉลย</summary>

```sql
SELECT
    word,
    COUNT(*) AS frequency
FROM products,
     UNNEST(STRING_TO_ARRAY(product_name, ' ')) AS word
GROUP BY word
ORDER BY frequency DESC, word
LIMIT 5;
```

**ผลลัพธ์:**
```
   word    | frequency
------------+-----------
 Air        |         2
 Men's      |         2
 15         |         1
 17         |         1
 18         |         1
```
</details>

### แบบฝึกหัดที่ 8
เขียน query ตรวจสอบว่าอีเมลของลูกค้าคนใดบ้างที่ **ไม่ผ่าน** รูปแบบอีเมลมาตรฐาน (ใช้ regex `^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$`) โดยต้อง `TRIM` ช่องว่างออกก่อนตรวจสอบด้วย เพราะบางอีเมลมีช่องว่างแฝงอยู่

<details>
<summary>เฉลย</summary>

```sql
SELECT
    customer_id,
    email,
    TRIM(email) !~* '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$' AS is_invalid
FROM customers
WHERE TRIM(email) !~* '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$';
```

หมายเหตุ: ถ้าได้แก้ไขอีเมลของลูกค้า #13 ไปแล้วตาม Step 309 ผลลัพธ์จะเป็น 0 แถว (ไม่มีอีเมลที่ผิดรูปแบบเหลืออยู่) ซึ่งแสดงว่าการทำความสะอาดข้อมูลก่อนหน้านี้สำเร็จแล้ว
</details>

### แบบฝึกหัดที่ 9
สร้างรายงานสรุปยอดขายรูปแบบข้อความล้วน (text report) โดยใช้ `RPAD`/`LPAD` จัดคอลัมน์ให้ตรงกัน แสดง product_name (30 ตัวอักษร), unit_price (จัดชิดขวา 12 ตัวอักษร), stock_quantity (จัดชิดขวา 8 ตัวอักษร) สำหรับสินค้าที่มี `stock_quantity < 20`

<details>
<summary>เฉลย</summary>

```sql
SELECT
    RPAD(product_name, 30) ||
    LPAD(unit_price::TEXT, 12) ||
    LPAD(stock_quantity::TEXT, 8) AS report_line
FROM products
WHERE stock_quantity < 20
ORDER BY stock_quantity;
```

**ผลลัพธ์:**
```
                        report_line
-------------------------------------------------------------
 IKEA Study Desk                     3990.00       8
 Dell XPS 13 Laptop                 45900.00      12
 Sony WH-1000XM5 Headphones         12900.00      16
 MacBook Air M2                     39900.00      15
 Samsung Galaxy S24 Ultra           39900.00      18
```
</details>

### แบบฝึกหัดที่ 10 (โจทย์รวม)
เขียน query เดียวที่ทำสิ่งต่อไปนี้พร้อมกันสำหรับตาราง `reviews`: (1) ตัดช่องว่างส่วนเกินในข้อความรีวิว (2) แทนที่เครื่องหมาย `!!!` ด้วย `!` เดียว (3) ปกปิดอีเมลใด ๆ ที่พบด้วยคำว่า `[hidden email]` (4) แสดงเฉพาะรีวิวที่มี rating น้อยกว่าหรือเท่ากับ 2 เรียงตาม review_date

<details>
<summary>เฉลย</summary>

```sql
SELECT
    review_id,
    rating,
    REGEXP_REPLACE(
        REGEXP_REPLACE(
            REGEXP_REPLACE(TRIM(review_text), '\s+', ' ', 'g'),
            '!{2,}', '!', 'g'
        ),
        '[\w.+-]+@[\w-]+\.[a-zA-Z]{2,}', '[hidden email]', 'g'
    ) AS cleaned_review,
    review_date
FROM reviews
WHERE rating <= 2
ORDER BY review_date;
```

**ผลลัพธ์:**
```
 review_id | rating |                                        cleaned_review                                         | review_date
-----------+--------+---------------------------------------------------------------------------------------------------+---------------
         9 |      1 | DO NOT BUY! โต๊ะพังภายใน 2 สัปดาห์ ติดต่อร้านไม่ได้เลย โทร 081-234-5678 ก็ไม่มีคนรับสาย            | 2024-02-10
         8 |      2 | ทอดไม่ค่อยกรอบเท่าที่ควร ราคาแพงไปหน่อย                                                              | 2024-03-03
        14 |      2 | POWER BANK ไม่เก็บไฟตามสเปค ใช้ได้ไม่ถึงครึ่งที่โฆษณา                                                | 2024-03-22
```

โจทย์นี้แสดงให้เห็นการซ้อนฟังก์ชัน `REGEXP_REPLACE` หลายชั้นเข้าด้วยกัน — เทคนิคที่ใช้บ่อยมากในงาน data cleaning ระดับมืออาชีพ โดยแต่ละชั้นแก้ปัญหาทีละอย่าง: ตัดช่องว่าง → รวมเครื่องหมายอัศเจรีย์ซ้ำ → ปกปิดอีเมล
</details>

---

**บทถัดไป:** [Part 032 — Date/Time Functions](./part-032-datetime-functions.md)
