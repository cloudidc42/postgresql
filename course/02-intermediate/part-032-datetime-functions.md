# Part 032: Date/Time Functions และการคำนวณช่วงเวลา

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับกลาง | Part 032

## เป้าหมายการเรียนรู้

Part 007 ในระดับพื้นฐานได้ปูพื้นเรื่อง data type สำหรับวันที่และเวลา (`DATE`, `TIME`, `TIMESTAMP`, `TIMESTAMPTZ`, `INTERVAL`) ไปแล้ว บทนี้เราจะก้าวข้ามไปสู่ "ฟังก์ชัน" ที่ใช้ประมวลผลค่าวันที่-เวลาในงานธุรกิจจริง ซึ่งเป็นสิ่งที่นักวิเคราะห์ข้อมูลและนักพัฒนาต้องใช้แทบทุกวันเมื่อทำรายงานยอดขาย รายงานลูกค้า หรือ dashboard ต่างๆ

เมื่อจบบทนี้ ผู้เรียนจะสามารถ:

- ใช้ `CURRENT_DATE`, `CURRENT_TIMESTAMP`, `NOW()` ได้อย่างถูกต้อง และเข้าใจว่าทำไมค่าเหล่านี้ "คงที่" ตลอด transaction
- ใช้ `DATE_TRUNC` ปัดเศษวันที่ลงตามหน่วยต่างๆ เพื่อจัดกลุ่มข้อมูลรายวัน/สัปดาห์/เดือน/ไตรมาส/ปี
- ใช้ `EXTRACT` และ `DATE_PART` ดึงส่วนประกอบของวันที่ไปวิเคราะห์ เช่น เปรียบเทียบยอดขายวันธรรมดากับวันหยุด
- ใช้ `AGE()` คำนวณอายุหรือระยะเวลาแบบอ่านง่าย (ปี-เดือน-วัน)
- คำนวณจำนวนวันทำงาน (business days) ระหว่างสองวันที่ โดยไม่นับเสาร์-อาทิตย์
- ใช้ `TO_CHAR` จัดรูปแบบวันที่สำหรับรายงานธุรกิจ ทั้งภาษาไทยและอังกฤษ
- ใช้ `GENERATE_SERIES` สร้างปฏิทินวันที่ต่อเนื่อง เพื่ออุดช่องว่าง (fill gaps) ในรายงานที่บางช่วงไม่มีข้อมูล
- ทำ cohort analysis เบื้องต้นโดยจัดกลุ่มลูกค้าตามเดือนที่สมัครสมาชิก
- ผสาน window function กับวันที่เพื่อเปรียบเทียบยอดขายรายเดือน/รายปี และคำนวณ year-over-year growth
- สร้างรายงานยอดขายรายเดือนแบบเต็มรูปแบบที่ครอบคลุมทุกเดือน แม้เดือนที่ไม่มียอดขายเลยก็ตาม

---

## เตรียมข้อมูล

บทนี้และ Part 021-039 ทั้งหมดใช้ฐานข้อมูลร้านค้าออนไลน์ (e-commerce) ชุดเดียวกัน หากคุณเคยสร้างตารางเหล่านี้ไปแล้วในบทก่อนหน้า สามารถข้ามไปยัง Step 311 ได้เลย มิฉะนั้นให้รันสคริปต์ด้านล่างเพื่อสร้างตารางและข้อมูลตัวอย่าง

```sql
-- ล้างตารางเดิม (ถ้ามี) เพื่อให้รันซ้ำได้อย่างปลอดภัย
DROP TABLE IF EXISTS payments, reviews, order_items, orders, employees, customers, products, suppliers, categories CASCADE;

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

### ข้อมูลตัวอย่าง

```sql
-- categories
INSERT INTO categories (category_id, category_name, parent_category_id) VALUES
(1, 'Electronics', NULL),
(2, 'Computers', 1),
(3, 'Mobile Phones', 1),
(4, 'Clothing', NULL),
(5, 'Men''s Clothing', 4),
(6, 'Women''s Clothing', 4),
(7, 'Books', NULL),
(8, 'Home & Kitchen', NULL);
SELECT setval('categories_category_id_seq', 8);

-- suppliers
INSERT INTO suppliers (supplier_id, supplier_name, country) VALUES
(1, 'Tech Import Co.', 'Thailand'),
(2, 'Global Electronics Ltd.', 'China'),
(3, 'Fashion House', 'Thailand'),
(4, 'Book World', 'USA'),
(5, 'Home Living Co.', 'Thailand'),
(6, 'Siam Gadget', 'Thailand'),
(7, 'Euro Textiles', 'Italy'),
(8, 'Kitchen Pro', 'Germany');
SELECT setval('suppliers_supplier_id_seq', 8);

-- products
INSERT INTO products (product_id, product_name, category_id, supplier_id, unit_price, stock_quantity, is_active) VALUES
(1, 'Laptop Pro 15', 2, 1, 32900.00, 25, true),
(2, 'Wireless Mouse', 2, 2, 590.00, 150, true),
(3, 'Mechanical Keyboard', 2, 2, 2490.00, 80, true),
(4, 'Smartphone X12', 3, 1, 21900.00, 40, true),
(5, 'Smartphone Lite', 3, 6, 8990.00, 60, true),
(6, 'Bluetooth Earbuds', 1, 2, 1290.00, 200, true),
(7, 'Men''s Cotton Shirt', 5, 3, 590.00, 100, true),
(8, 'Men''s Jeans', 5, 7, 890.00, 90, true),
(9, 'Women''s Dress', 6, 3, 1290.00, 70, true),
(10, 'Women''s Blouse', 6, 7, 690.00, 85, true),
(11, 'PostgreSQL Handbook', 7, 4, 890.00, 40, true),
(12, 'SQL for Beginners', 7, 4, 590.00, 55, true),
(13, 'Non-stick Pan Set', 8, 8, 1990.00, 35, true),
(14, 'Coffee Maker', 8, 5, 2590.00, 28, true),
(15, 'Air Fryer', 8, 8, 3290.00, 32, true),
(16, 'Desk Lamp', 8, 5, 490.00, 60, true);
SELECT setval('products_product_id_seq', 16);

-- customers (signup_date กระจายตั้งแต่มีนาคม 2025 ถึงเมษายน 2026 เพื่อใช้ทำ cohort analysis)
INSERT INTO customers (customer_id, first_name, last_name, email, country, signup_date) VALUES
(1, 'Somchai', 'Jaidee', 'somchai.j@email.com', 'Thailand', '2025-03-10'),
(2, 'Suda', 'Meesuk', 'suda.m@email.com', 'Thailand', '2025-04-18'),
(3, 'Wichai', 'Thongdee', 'wichai.t@email.com', 'Thailand', '2025-06-02'),
(4, 'Nattaya', 'Boonmee', 'nattaya.b@email.com', 'Thailand', '2025-07-09'),
(5, 'Anan', 'Srisuk', 'anan.s@email.com', 'Thailand', '2025-08-14'),
(6, 'Pranee', 'Chaiyo', 'pranee.c@email.com', 'Thailand', '2025-09-01'),
(7, 'Kittipong', 'Rungrueang', 'kittipong.r@email.com', 'Thailand', '2026-01-15'),
(8, 'Malee', 'Wongsa', 'malee.w@email.com', 'Thailand', '2026-01-22'),
(9, 'Prasert', 'Kaewta', 'prasert.k@email.com', 'Thailand', '2026-02-03'),
(10, 'Siriporn', 'Intharak', 'siriporn.i@email.com', 'Thailand', '2026-02-18'),
(11, 'Thanawat', 'Phromma', 'thanawat.p@email.com', 'Thailand', '2026-02-28'),
(12, 'Ratchanee', 'Suksawat', 'ratchanee.s@email.com', 'Thailand', '2026-03-10'),
(13, 'Boonchu', 'Panya', 'boonchu.p@email.com', 'Thailand', '2026-03-25'),
(14, 'Kanya', 'Thepsuwan', 'kanya.t@email.com', 'Thailand', '2026-04-05'),
(15, 'Sombat', 'Naowarat', 'sombat.n@email.com', 'Thailand', '2026-04-20');
SELECT setval('customers_customer_id_seq', 15);

-- employees
INSERT INTO employees (employee_id, first_name, last_name, hire_date, manager_id, department) VALUES
(1, 'Piti', 'Kunakorn', '2022-01-10', NULL, 'Sales'),
(2, 'Orawan', 'Suthep', '2022-03-15', 1, 'Sales'),
(3, 'Decha', 'Wattana', '2023-02-01', 1, 'Sales'),
(4, 'Napa', 'Sirisak', '2023-06-20', 1, 'Customer Service'),
(5, 'Somsak', 'Buri', '2024-01-05', 2, 'Sales'),
(6, 'Ploy', 'Aksorn', '2024-05-12', 2, 'Sales');
SELECT setval('employees_employee_id_seq', 6);

-- orders (2026 = ข้อมูลหลัก 6 เดือนล่าสุด, 2025 = ข้อมูลปีก่อนหน้าสำหรับเปรียบเทียบ YoY)
INSERT INTO orders (order_id, customer_id, employee_id, order_date, status, ship_country) VALUES
(1, 1, 1, '2026-04-03 10:15:00+07', 'completed', 'Thailand'),
(2, 2, 2, '2026-04-10 14:30:00+07', 'completed', 'Thailand'),
(3, 3, 1, '2026-04-22 09:00:00+07', 'completed', 'Thailand'),
(4, 1, 3, '2026-05-02 11:20:00+07', 'completed', 'Thailand'),
(5, 4, 2, '2026-05-09 16:45:00+07', 'completed', 'Thailand'),
(6, 5, 1, '2026-05-15 08:30:00+07', 'cancelled', 'Thailand'),
(7, 6, 4, '2026-05-28 13:10:00+07', 'completed', 'Thailand'),
(8, 2, 3, '2026-06-04 10:00:00+07', 'completed', 'Thailand'),
(9, 7, 2, '2026-06-11 15:25:00+07', 'completed', 'Thailand'),
(10, 8, 1, '2026-06-19 09:50:00+07', 'completed', 'Thailand'),
(11, 3, 5, '2026-06-27 12:00:00+07', 'completed', 'Thailand'),
(12, 9, 4, '2026-07-05 17:15:00+07', 'completed', 'Thailand'),
(13, 10, 2, '2026-07-13 10:40:00+07', 'completed', 'Thailand'),
(14, 1, 1, '2026-07-20 14:05:00+07', 'completed', 'Thailand'),
(15, 11, 6, '2026-07-30 09:20:00+07', 'pending', 'Thailand'),
(16, 4, 3, '2026-08-02 11:55:00+07', 'completed', 'Thailand'),
(17, 12, 2, '2026-08-09 16:30:00+07', 'completed', 'Thailand'),
(18, 6, 5, '2026-08-16 08:45:00+07', 'completed', 'Thailand'),
(19, 13, 1, '2026-08-24 13:35:00+07', 'completed', 'Thailand'),
(20, 2, 4, '2026-08-29 10:10:00+07', 'completed', 'Thailand'),
(21, 14, 6, '2026-09-05 15:00:00+07', 'completed', 'Thailand'),
(22, 7, 2, '2026-09-12 09:30:00+07', 'completed', 'Thailand'),
(23, 15, 3, '2026-09-18 14:20:00+07', 'completed', 'Thailand'),
(24, 5, 1, '2026-09-23 11:00:00+07', 'pending', 'Thailand'),
(25, 1, 1, '2025-04-08 10:00:00+07', 'completed', 'Thailand'),
(26, 2, 2, '2025-05-14 11:00:00+07', 'completed', 'Thailand'),
(27, 3, 1, '2025-06-20 09:30:00+07', 'completed', 'Thailand'),
(28, 4, 3, '2025-07-11 14:00:00+07', 'completed', 'Thailand'),
(29, 5, 2, '2025-08-20 16:20:00+07', 'completed', 'Thailand'),
(30, 6, 1, '2025-09-17 10:45:00+07', 'completed', 'Thailand');
SELECT setval('orders_order_id_seq', 30);

-- order_items
INSERT INTO order_items (order_item_id, order_id, product_id, quantity, unit_price) VALUES
(1, 1, 1, 1, 32900.00),
(2, 1, 2, 2, 590.00),
(3, 2, 4, 1, 21900.00),
(4, 3, 6, 3, 1290.00),
(5, 4, 3, 1, 2490.00),
(6, 4, 2, 1, 590.00),
(7, 5, 9, 2, 1290.00),
(8, 6, 5, 1, 8990.00),
(9, 7, 11, 2, 890.00),
(10, 8, 7, 3, 590.00),
(11, 9, 1, 1, 32900.00),
(12, 10, 13, 1, 1990.00),
(13, 10, 16, 2, 490.00),
(14, 11, 4, 1, 21900.00),
(15, 12, 8, 2, 890.00),
(16, 13, 6, 5, 1290.00),
(17, 14, 14, 1, 2590.00),
(18, 15, 2, 4, 590.00),
(19, 16, 9, 1, 1290.00),
(20, 16, 10, 2, 690.00),
(21, 17, 1, 1, 32900.00),
(22, 18, 12, 3, 590.00),
(23, 19, 15, 1, 3290.00),
(24, 20, 3, 2, 2490.00),
(25, 21, 5, 1, 8990.00),
(26, 22, 7, 2, 590.00),
(27, 22, 8, 1, 890.00),
(28, 23, 6, 2, 1290.00),
(29, 24, 2, 3, 590.00),
(30, 25, 1, 1, 32900.00),
(31, 26, 4, 1, 21900.00),
(32, 27, 6, 2, 1290.00),
(33, 28, 9, 1, 1290.00),
(34, 29, 11, 1, 890.00),
(35, 30, 7, 2, 590.00);
SELECT setval('order_items_order_item_id_seq', 35);

-- reviews
INSERT INTO reviews (review_id, product_id, customer_id, rating, review_text, review_date) VALUES
(1, 1, 1, 5, 'สินค้าดีมากค่ะ ทำงานลื่น', '2026-04-10'),
(2, 4, 2, 4, 'ใช้งานดี แบตอึด', '2026-04-15'),
(3, 6, 3, 5, 'เสียงดีมาก คุ้มราคา', '2026-04-28'),
(4, 3, 1, 4, 'พิมพ์สบายมือ', '2026-05-05'),
(5, 9, 4, 5, 'ผ้าดีใส่สบาย', '2026-05-12'),
(6, 11, 6, 5, 'อธิบายเข้าใจง่ายมาก', '2026-06-01'),
(7, 7, 2, 3, 'ไซส์เล็กไปนิดนึง', '2026-06-08'),
(8, 1, 7, 5, 'คุ้มค่ามาก แนะนำเลย', '2026-06-15'),
(9, 13, 8, 4, 'กระทะดีไม่ติดจริง', '2026-06-22'),
(10, 6, 3, 4, 'เสียงดีแต่แบตหมดเร็ว', '2026-07-10'),
(11, 14, 1, 5, 'ชงกาแฟอร่อยทุกเช้า', '2026-07-25'),
(12, 1, 12, 4, 'จอสวยทำงานลื่น', '2026-08-12'),
(13, 15, 13, 5, 'ทอดกรอบไม่ต้องใช้น้ำมัน', '2026-08-28'),
(14, 5, 14, 4, 'ราคาคุ้มค่าดี', '2026-09-08');
SELECT setval('reviews_review_id_seq', 14);

-- payments (เฉพาะออเดอร์ที่ status = 'completed')
INSERT INTO payments (payment_id, order_id, payment_date, amount, payment_method) VALUES
(1, 1, '2026-04-03 10:20:00+07', 34080.00, 'credit_card'),
(2, 2, '2026-04-10 14:35:00+07', 21900.00, 'bank_transfer'),
(3, 3, '2026-04-22 09:05:00+07', 3870.00, 'promptpay'),
(4, 4, '2026-05-02 11:25:00+07', 3080.00, 'credit_card'),
(5, 5, '2026-05-09 16:50:00+07', 2580.00, 'promptpay'),
(6, 7, '2026-05-28 13:15:00+07', 1780.00, 'cod'),
(7, 8, '2026-06-04 10:05:00+07', 1770.00, 'credit_card'),
(8, 9, '2026-06-11 15:30:00+07', 32900.00, 'bank_transfer'),
(9, 10, '2026-06-19 09:55:00+07', 2970.00, 'promptpay'),
(10, 11, '2026-06-27 12:05:00+07', 21900.00, 'credit_card'),
(11, 12, '2026-07-05 17:20:00+07', 1780.00, 'cod'),
(12, 13, '2026-07-13 10:45:00+07', 6450.00, 'credit_card'),
(13, 14, '2026-07-20 14:10:00+07', 2590.00, 'promptpay'),
(14, 16, '2026-08-02 12:00:00+07', 2670.00, 'credit_card'),
(15, 17, '2026-08-09 16:35:00+07', 32900.00, 'bank_transfer'),
(16, 18, '2026-08-16 08:50:00+07', 1770.00, 'promptpay'),
(17, 19, '2026-08-24 13:40:00+07', 3290.00, 'credit_card'),
(18, 20, '2026-08-29 10:15:00+07', 4980.00, 'cod'),
(19, 21, '2026-09-05 15:05:00+07', 8990.00, 'credit_card'),
(20, 22, '2026-09-12 09:35:00+07', 2070.00, 'promptpay'),
(21, 23, '2026-09-18 14:25:00+07', 2580.00, 'credit_card'),
(22, 25, '2025-04-08 10:05:00+07', 32900.00, 'credit_card'),
(23, 26, '2025-05-14 11:05:00+07', 21900.00, 'bank_transfer'),
(24, 27, '2025-06-20 09:35:00+07', 2580.00, 'promptpay'),
(25, 28, '2025-07-11 14:05:00+07', 1290.00, 'credit_card'),
(26, 29, '2025-08-20 16:25:00+07', 890.00, 'cod'),
(27, 30, '2025-09-17 10:50:00+07', 1180.00, 'promptpay');
SELECT setval('payments_payment_id_seq', 27);
```

หมายเหตุ: คอลัมน์ `amount` ของ `payments` ในตัวอย่างนี้เท่ากับยอดรวมของ `order_items` ในออเดอร์นั้นๆ พอดี (คำนวณจาก `SUM(quantity * unit_price)`) เพื่อให้ตัวอย่างการคำนวณในบทนี้ตรวจสอบได้ง่าย ออเดอร์ที่มี status เป็น `pending` หรือ `cancelled` (order_id 6, 15, 24) จะไม่มีแถวใน `payments`

---

## Step 311: ทบทวน CURRENT_DATE / CURRENT_TIMESTAMP / NOW() และ transaction time

ก่อนเข้าสู่ฟังก์ชันขั้นสูง เรามาทบทวนฟังก์ชันพื้นฐานที่ใช้บ่อยที่สุดกันก่อน เพราะมีรายละเอียดสำคัญที่มักทำให้เกิด bug ในรายงาน

```sql
SELECT
    CURRENT_DATE                 AS today_date,
    CURRENT_TIME                 AS current_time_val,
    CURRENT_TIMESTAMP            AS current_ts,
    NOW()                        AS now_val,
    LOCALTIMESTAMP                AS local_ts;
```

**ตัวอย่างผลลัพธ์** (สมมติรันวันที่ 2026-09-25 เวลา 14:30 ตามเขตเวลา Asia/Bangkok):

| today_date | current_time_val  | current_ts                   | now_val                       | local_ts            |
|------------|--------------------|-------------------------------|--------------------------------|----------------------|
| 2026-09-25 | 14:30:00.123456+07 | 2026-09-25 14:30:00.123456+07 | 2026-09-25 14:30:00.123456+07 | 2026-09-25 14:30:00 |

ประเด็นสำคัญที่ต้องเข้าใจ:

1. **`CURRENT_TIMESTAMP` และ `NOW()` เป็นฟังก์ชันเดียวกัน** — `NOW()` คือ syntax แบบ PostgreSQL ส่วน `CURRENT_TIMESTAMP` คือ syntax ตามมาตรฐาน SQL ทั้งคู่คืนค่า `timestamptz`
2. **ค่าคงที่ตลอด transaction (transaction time)** — ทั้ง `NOW()`, `CURRENT_TIMESTAMP`, `CURRENT_DATE` จะคืนค่า "เวลาที่ transaction เริ่มต้น" ไม่ใช่เวลาที่บรรทัดนั้นถูกรันจริง ลองดูตัวอย่างนี้:

```sql
BEGIN;
SELECT NOW();          -- สมมติได้ 14:30:00.000
SELECT pg_sleep(3);    -- หน่วงเวลา 3 วินาที
SELECT NOW();          -- ยังคงได้ 14:30:00.000 เท่าเดิม!
COMMIT;
```

หากต้องการเวลา ณ ขณะที่บรรทัดนั้นถูกประมวลผลจริง (statement time / clock time) ให้ใช้ `CLOCK_TIMESTAMP()` แทน:

```sql
BEGIN;
SELECT CLOCK_TIMESTAMP();  -- 14:30:00.000
SELECT pg_sleep(3);
SELECT CLOCK_TIMESTAMP();  -- 14:30:03.001 (เดินหน้าไปจริง)
COMMIT;
```

3. **`STATEMENT_TIMESTAMP()`** อยู่ตรงกลาง คือเวลาเริ่มต้นของ "statement ปัจจุบัน" (จะเปลี่ยนทุกครั้งที่รันคำสั่งใหม่ แต่ไม่เปลี่ยนระหว่างที่ statement เดียวกันกำลังทำงานอยู่)

| ฟังก์ชัน | ความคงที่ | เหมาะกับ |
|---|---|---|
| `NOW()` / `CURRENT_TIMESTAMP` | คงที่ตลอด transaction | บันทึก created_at ที่ต้องสอดคล้องกันทั้ง transaction |
| `STATEMENT_TIMESTAMP()` | เปลี่ยนทุก statement | log การทำงานทีละคำสั่ง |
| `CLOCK_TIMESTAMP()` | เปลี่ยนตลอดเวลาแบบ real-time | วัด performance, benchmark |

ในงาน insert ข้อมูล เช่น `orders.order_date DEFAULT now()` การใช้ `now()` จึงปลอดภัย เพราะทุกแถวที่ insert ภายใน transaction เดียวกันจะได้ timestamp เดียวกัน ทำให้ข้อมูลสอดคล้องกัน (consistent) ไม่เกิดปัญหาแถวหนึ่งช้ากว่าอีกแถวเสี้ยววินาที

```sql
-- ตัวอย่างการใช้ CURRENT_DATE หาออเดอร์ของ "วันนี้"
SELECT order_id, order_date, status
FROM orders
WHERE order_date::date = CURRENT_DATE;
```

> เคล็ดลับ: ในระบบ production ที่ต้อง reproducible testing เช่น การเขียน unit test ควรหลีกเลี่ยงการ hardcode ผลลัพธ์ที่อิงกับ `NOW()` โดยตรง เพราะผลลัพธ์จะเปลี่ยนทุกวันที่รัน

---

## Step 312: DATE_TRUNC — ปัดเศษวันที่ลงตามหน่วย

`DATE_TRUNC(field, source)` เป็นฟังก์ชันที่ใช้บ่อยที่สุดในการทำรายงานสรุปตามช่วงเวลา เพราะมันจะ "ปัดวันที่ลง" (truncate ไม่ใช่ round) ไปยังจุดเริ่มต้นของหน่วยที่ระบุ

```sql
SELECT
    order_date,
    DATE_TRUNC('day', order_date)     AS trunc_day,
    DATE_TRUNC('week', order_date)    AS trunc_week,
    DATE_TRUNC('month', order_date)   AS trunc_month,
    DATE_TRUNC('quarter', order_date) AS trunc_quarter,
    DATE_TRUNC('year', order_date)    AS trunc_year
FROM orders
WHERE order_id = 13;
```

**ผลลัพธ์:**

| order_date                     | trunc_day               | trunc_week               | trunc_month              | trunc_quarter             | trunc_year                |
|----------------------------------|--------------------------|----------------------------|----------------------------|------------------------------|------------------------------|
| 2026-07-13 10:40:00+07          | 2026-07-13 00:00:00+07  | 2026-07-13 00:00:00+07    | 2026-07-01 00:00:00+07    | 2026-07-01 00:00:00+07      | 2026-01-01 00:00:00+07      |

ข้อสังเกต: `DATE_TRUNC('week', ...)` ในกรณีนี้ตรงกับวันเดียวกัน เพราะ 13 กรกฎาคม 2026 เป็นวันจันทร์พอดี — PostgreSQL ถือว่าสัปดาห์เริ่มต้นวัน**จันทร์** ตามมาตรฐาน ISO 8601 (ต่างจาก `EXTRACT(DOW ...)` ที่ถือว่าอาทิตย์เป็นวันที่ 0)

### ใช้งานจริง: ยอดขายรายเดือน

```sql
SELECT
    DATE_TRUNC('month', o.order_date)::date AS sales_month,
    COUNT(DISTINCT o.order_id)              AS order_count,
    SUM(oi.quantity * oi.unit_price)        AS total_revenue
FROM orders o
JOIN order_items oi ON oi.order_id = o.order_id
WHERE o.status = 'completed'
  AND o.order_date >= '2026-01-01'
GROUP BY DATE_TRUNC('month', o.order_date)
ORDER BY sales_month;
```

**ผลลัพธ์:**

| sales_month | order_count | total_revenue |
|-------------|-------------|-----------------|
| 2026-04-01  | 3           | 59850.00        |
| 2026-05-01  | 3           | 7440.00         |
| 2026-06-01  | 4           | 59540.00        |
| 2026-07-01  | 3           | 10820.00        |
| 2026-08-01  | 5           | 45610.00        |
| 2026-09-01  | 3           | 13640.00        |

สังเกตว่าเดือนมกราคม-มีนาคม 2026 ไม่ปรากฏในผลลัพธ์เลย เพราะไม่มีออเดอร์ในช่วงนั้น — นี่คือปัญหาคลาสสิกที่เราจะแก้ด้วย `GENERATE_SERIES` ใน Step 317

`DATE_TRUNC` รองรับหน่วยอื่นๆ อีกมาก เช่น `'decade'`, `'century'`, `'hour'`, `'minute'`, `'microseconds'` — เหมาะกับการทำรายงานแบบ real-time dashboard ที่ต้อง group ตามชั่วโมงก็ได้เช่นกัน:

```sql
SELECT DATE_TRUNC('hour', order_date) AS order_hour, COUNT(*) AS orders_in_hour
FROM orders
GROUP BY 1
ORDER BY 1;
```

ยังมี syntax แบบใหม่ตั้งแต่ PostgreSQL 14 ที่รับ `INTERVAL` เป็นตัวปัดหน่วยที่กำหนดเองได้ (เช่น ปัดทุก 15 นาที) ผ่าน `date_bin()`:

```sql
-- ปัดเวลาลงทุกๆ 15 นาที นับจาก origin ที่กำหนด
SELECT
    order_date,
    date_bin('15 minutes', order_date, TIMESTAMPTZ '2026-01-01 00:00:00+07') AS bucket_15min
FROM orders
ORDER BY order_date
LIMIT 5;
```

`date_bin` มีประโยชน์มากเมื่อต้องการ bucket ที่ไม่ตรงกับหน่วยมาตรฐาน เช่น ทุก 5 นาที ทุก 10 วัน เป็นต้น ซึ่ง `DATE_TRUNC` ทำไม่ได้

---

## Step 313: DATE_PART และ EXTRACT เชิงลึก

`EXTRACT(field FROM source)` เป็น syntax ตามมาตรฐาน SQL ส่วน `DATE_PART('field', source)` เป็น syntax แบบฟังก์ชันของ PostgreSQL ทั้งสองทำงานเหมือนกันทุกประการ ต่างกันแค่รูปแบบการเขียนเท่านั้น

```sql
SELECT
    order_date,
    EXTRACT(YEAR FROM order_date)    AS yr,
    EXTRACT(MONTH FROM order_date)   AS mth,
    EXTRACT(DAY FROM order_date)     AS d,
    EXTRACT(DOW FROM order_date)     AS day_of_week,   -- 0=อาทิตย์ ... 6=เสาร์
    EXTRACT(ISODOW FROM order_date)  AS iso_day_of_week, -- 1=จันทร์ ... 7=อาทิตย์
    EXTRACT(DOY FROM order_date)     AS day_of_year,
    EXTRACT(QUARTER FROM order_date) AS qtr,
    DATE_PART('hour', order_date)    AS hr
FROM orders
WHERE order_id = 1;
```

**ผลลัพธ์:**

| order_date | yr | mth | d | day_of_week | iso_day_of_week | day_of_year | qtr | hr |
|---|---|---|---|---|---|---|---|---|
| 2026-04-03 10:15:00+07 | 2026 | 4 | 3 | 5 | 5 | 93 | 2 | 10 |

`EXTRACT(DOW ...)` คืนค่า 0-6 โดย 0 = วันอาทิตย์ ส่วน `EXTRACT(ISODOW ...)` คืนค่า 1-7 โดย 1 = วันจันทร์ (ตามมาตรฐาน ISO) — ควรเลือกใช้ตัวที่ตรงกับ logic ที่ต้องการ เพื่อไม่ให้สับสนเวลาเขียนเงื่อนไข

### Use case: เปรียบเทียบยอดขายวันธรรมดา vs วันหยุดสุดสัปดาห์

```sql
SELECT
    CASE
        WHEN EXTRACT(ISODOW FROM o.order_date) IN (6, 7) THEN 'weekend'
        ELSE 'weekday'
    END AS day_type,
    COUNT(DISTINCT o.order_id)         AS order_count,
    SUM(oi.quantity * oi.unit_price)   AS total_revenue,
    ROUND(AVG(order_totals.order_amt), 2) AS avg_order_value
FROM orders o
JOIN order_items oi ON oi.order_id = o.order_id
JOIN (
    SELECT order_id, SUM(quantity * unit_price) AS order_amt
    FROM order_items
    GROUP BY order_id
) order_totals ON order_totals.order_id = o.order_id
WHERE o.status = 'completed'
GROUP BY day_type
ORDER BY day_type;
```

**ผลลัพธ์:**

| day_type | order_count | total_revenue | avg_order_value |
|----------|-------------|-----------------|-------------------|
| weekday  | 11          | 114180.00        | 10380.00          |
| weekend  | 10          | 82720.00         | 8272.00           |

จากข้อมูลตัวอย่าง แม้จำนวนออเดอร์วันธรรมดาจะมากกว่าเล็กน้อย แต่ **ยอดเฉลี่ยต่อออเดอร์ในวันธรรมดาสูงกว่าวันหยุด** อย่างชัดเจน (10,380 บาท เทียบกับ 8,272 บาท) เพราะสินค้าราคาสูงอย่างโน้ตบุ๊กถูกสั่งซื้อในวันธรรมดาหลายครั้ง — นี่คือตัวอย่างของ insight ที่ business analyst มักใช้ `EXTRACT`/`DATE_PART` ค้นหา

อีกตัวอย่างหนึ่งที่ใช้บ่อยคือการหาช่วงเวลาที่มีคนสั่งซื้อมากที่สุดในแต่ละวัน (peak hour):

```sql
SELECT
    EXTRACT(HOUR FROM order_date) AS order_hour,
    COUNT(*) AS order_count
FROM orders
GROUP BY order_hour
ORDER BY order_count DESC, order_hour
LIMIT 5;
```

> หมายเหตุ: `EXTRACT(EPOCH FROM ts)` คืนค่าจำนวนวินาทีตั้งแต่ 1970-01-01 UTC มีประโยชน์มากเมื่อต้องการแปลง timestamp ไปเป็นตัวเลขเพื่อคำนวณ หรือส่งให้ระบบภายนอกที่ใช้ Unix timestamp

```sql
SELECT order_id, order_date, EXTRACT(EPOCH FROM order_date) AS unix_ts
FROM orders
WHERE order_id = 1;
```

---

## Step 314: AGE() — คำนวณอายุแบบ human-readable

`AGE(timestamp, timestamp)` คืนค่าเป็น `INTERVAL` ในรูปแบบปี-เดือน-วัน-ชั่วโมง ที่มนุษย์อ่านเข้าใจง่าย ต่างจากการลบ timestamp ตรงๆ ที่จะได้ผลลัพธ์เป็นจำนวนวันล้วนๆ

```sql
SELECT
    employee_id,
    first_name || ' ' || last_name AS employee_name,
    hire_date,
    AGE(CURRENT_DATE, hire_date) AS tenure
FROM employees
ORDER BY hire_date;
```

**ผลลัพธ์** (สมมติวันนี้คือ 2026-09-25):

| employee_id | employee_name    | hire_date  | tenure                          |
|--------------|--------------------|------------|-----------------------------------|
| 1            | Piti Kunakorn      | 2022-01-10 | 4 years 8 mons 15 days           |
| 2            | Orawan Suthep      | 2022-03-15 | 4 years 6 mons 10 days           |
| 3            | Decha Wattana      | 2023-02-01 | 3 years 7 mons 24 days           |
| 4            | Napa Sirisak       | 2023-06-20 | 3 years 3 mons 5 days            |
| 5            | Somsak Buri        | 2024-01-05 | 2 years 8 mons 20 days           |
| 6            | Ploy Aksorn        | 2024-05-12 | 2 years 4 mons 13 days           |

`AGE(timestamp)` แบบ argument เดียว จะเทียบกับ `CURRENT_DATE` โดยอัตโนมัติ (เทียบเท่ากับ `AGE(CURRENT_DATE, timestamp)`):

```sql
SELECT customer_id, first_name, signup_date, AGE(signup_date) AS customer_since
FROM customers
ORDER BY signup_date
LIMIT 3;
```

**ผลลัพธ์:**

| customer_id | first_name | signup_date | customer_since          |
|--------------|-------------|---------------|----------------------------|
| 1            | Somchai     | 2025-03-10   | 1 year 6 mons 15 days     |
| 2            | Suda        | 2025-04-18   | 1 year 5 mons 7 days      |
| 3            | Wichai      | 2025-06-02   | 1 year 3 mons 23 days     |

### ดึงส่วนประกอบจาก AGE() ออกมาใช้แยกกัน

เมื่อได้ผลลัพธ์เป็น `INTERVAL` แล้ว เราสามารถใช้ `EXTRACT` ดึงเฉพาะปี/เดือน/วันออกมาแสดงในรายงานแบบข้อความได้:

```sql
SELECT
    customer_id,
    first_name,
    signup_date,
    EXTRACT(YEAR  FROM AGE(signup_date)) AS years_as_customer,
    EXTRACT(MONTH FROM AGE(signup_date)) AS months_as_customer,
    CONCAT(
        EXTRACT(YEAR FROM AGE(signup_date))::int, ' ปี ',
        EXTRACT(MONTH FROM AGE(signup_date))::int, ' เดือน'
    ) AS membership_thai
FROM customers
ORDER BY signup_date
LIMIT 3;
```

**ผลลัพธ์:**

| customer_id | first_name | years_as_customer | months_as_customer | membership_thai |
|---|---|---|---|---|
| 1 | Somchai | 1 | 6 | 1 ปี 6 เดือน |
| 2 | Suda    | 1 | 5 | 1 ปี 5 เดือน |
| 3 | Wichai  | 1 | 3 | 1 ปี 3 เดือน |

> ข้อควรระวัง: `AGE()` คำนึงถึงจำนวนวันในแต่ละเดือนที่ไม่เท่ากัน (28/29/30/31 วัน) ให้อัตโนมัติ ทำให้ผลลัพธ์แม่นยำกว่าการหารจำนวนวันทั้งหมดด้วย 30 หรือ 365 แบบหยาบๆ ซึ่งเป็นวิธีที่มักเห็นในโค้ดของมือใหม่และให้ผลคลาดเคลื่อน

---

## Step 315: คำนวณวันทำงาน (Business Days) ระหว่างสองวันที่

PostgreSQL ไม่มีฟังก์ชันสำเร็จรูปสำหรับนับ "วันทำงาน" (ไม่นับเสาร์-อาทิตย์) โดยตรง แต่เราสามารถสร้างได้ง่ายๆ ด้วย `GENERATE_SERIES` ร่วมกับ `EXTRACT(ISODOW ...)`

```sql
SELECT COUNT(*) AS business_days
FROM GENERATE_SERIES('2026-04-01'::date, '2026-04-30'::date, '1 day') AS d
WHERE EXTRACT(ISODOW FROM d) NOT IN (6, 7);  -- ตัดเสาร์(6) และอาทิตย์(7) ออก
```

**ผลลัพธ์:**

| business_days |
|-----------------|
| 22              |

เดือนเมษายน 2026 มี 30 วัน แต่มีวันทำงานเพียง 22 วัน (8 วันตรงกับเสาร์-อาทิตย์)

### สร้างเป็นฟังก์ชันใช้ซ้ำได้

ในงานจริง เรามักห่อ logic นี้เป็น SQL function เพื่อเรียกใช้ซ้ำได้สะดวก:

```sql
CREATE OR REPLACE FUNCTION count_business_days(start_date DATE, end_date DATE)
RETURNS INTEGER
LANGUAGE sql
IMMUTABLE
AS $$
    SELECT COUNT(*)::integer
    FROM GENERATE_SERIES(start_date, end_date, '1 day') AS d
    WHERE EXTRACT(ISODOW FROM d) NOT IN (6, 7);
$$;
```

```sql
SELECT count_business_days('2026-04-01', '2026-04-30') AS april_business_days;
```

**ผลลัพธ์:**

| april_business_days |
|------------------------|
| 22                      |

### Use case: คำนวณวันครบกำหนดจัดส่ง (SLA) แบบข้าม weekend

สมมติร้านค้ามีนโยบายจัดส่งสินค้าภายใน 3 วันทำการหลังจากได้รับออเดอร์ เราต้องการหาว่าวันครบกำหนดจัดส่งคือวันที่เท่าไร โดยข้ามวันเสาร์-อาทิตย์:

```sql
WITH RECURSIVE ship_deadline AS (
    SELECT
        order_id,
        order_date::date AS current_day,
        0 AS business_days_counted
    FROM orders
    WHERE order_id = 15   -- ออเดอร์ที่ยังเป็น pending, สั่งวันที่ 2026-07-30
    UNION ALL
    SELECT
        sd.order_id,
        sd.current_day + 1,
        sd.business_days_counted +
            CASE WHEN EXTRACT(ISODOW FROM sd.current_day + 1) NOT IN (6, 7) THEN 1 ELSE 0 END
    FROM ship_deadline sd
    WHERE sd.business_days_counted < 3
)
SELECT order_id, MAX(current_day) AS ship_deadline_date
FROM ship_deadline
GROUP BY order_id;
```

**ผลลัพธ์:**

| order_id | ship_deadline_date |
|----------|-----------------------|
| 15       | 2026-08-04            |

ออเดอร์ 15 สั่งวันพฤหัสบดีที่ 30 กรกฎาคม 2026 เมื่อนับ 3 วันทำการโดยข้ามเสาร์-อาทิตย์ (31 ก.ค. ศุกร์ = วันที่ 1, 1-2 ส.ค. เสาร์-อาทิตย์ ข้าม, 3 ส.ค. จันทร์ = วันที่ 2, 4 ส.ค. อังคาร = วันที่ 3) จะได้กำหนดส่งคือวันที่ **4 สิงหาคม 2026**

> สำหรับระบบที่ต้องคำนึงถึงวันหยุดนักขัตฤกษ์ด้วย (ไม่ใช่แค่เสาร์-อาทิตย์) แนวทางที่ scale ได้ดีกว่าคือสร้างตาราง `public_holidays (holiday_date DATE PRIMARY KEY)` แล้วเพิ่มเงื่อนไข `AND d NOT IN (SELECT holiday_date FROM public_holidays)` เข้าไปในนิพจน์ `GENERATE_SERIES` ข้างต้น

---

## Step 316: TO_CHAR สำหรับ Reporting

`TO_CHAR(value, format_pattern)` คือฟังก์ชันที่ขาดไม่ได้เมื่อต้องแสดงวันที่ในรายงานให้อ่านง่าย เพราะควบคุมรูปแบบได้ละเอียดมาก

```sql
SELECT
    order_date,
    TO_CHAR(order_date, 'DD/MM/YYYY')            AS thai_short,
    TO_CHAR(order_date, 'DD Month YYYY')          AS full_date_en,
    TO_CHAR(order_date, 'FMDD Month YYYY')        AS full_date_en_trim,  -- FM ตัดช่องว่างส่วนเกิน
    TO_CHAR(order_date, 'Day')                    AS day_name,
    TO_CHAR(order_date, 'HH24:MI:SS')             AS time_24h,
    TO_CHAR(order_date, '"Q"Q YYYY')              AS quarter_label,
    TO_CHAR(order_date, 'YYYY-MM')                AS year_month
FROM orders
WHERE order_id = 13;
```

**ผลลัพธ์:**

| order_date | thai_short | full_date_en | full_date_en_trim | day_name | time_24h | quarter_label | year_month |
|---|---|---|---|---|---|---|---|
| 2026-07-13 10:40:00+07 | 13/07/2026 | 13 July      2026 | 13 July 2026 | Monday    | 10:40:00 | Q3 2026 | 2026-07 |

### รูปแบบ pattern ที่ใช้บ่อยในรายงานธุรกิจ

| Pattern | ความหมาย | ตัวอย่างผลลัพธ์ |
|---|---|---|
| `YYYY` | ปี ค.ศ. 4 หลัก | 2026 |
| `MM` | เดือนตัวเลข 2 หลัก | 07 |
| `Month` / `FMMonth` | ชื่อเดือนเต็ม (มีช่องว่างขวา / ไม่มี) | July / July |
| `Mon` | ชื่อเดือนย่อ | Jul |
| `DD` | วันที่ 2 หลัก | 13 |
| `Day` / `Dy` | ชื่อวันเต็ม / ย่อ | Monday / Mon |
| `Q` | ไตรมาส (1-4) | 3 |
| `HH24:MI:SS` | เวลาแบบ 24 ชั่วโมง | 10:40:00 |
| `HH12:MI AM` | เวลาแบบ 12 ชั่วโมง | 10:40 AM |

### เดือนภาษาไทย

PostgreSQL ไม่มี locale ภาษาไทยติดตั้งมาให้ในเซิร์ฟเวอร์ส่วนใหญ่ (ต่างจาก `TO_CHAR(..., 'TMMonth')` ที่ใช้ locale ของระบบปฏิบัติการ ซึ่งมักไม่มีภาษาไทย) วิธีที่นิยมและพกพาได้ (portable) มากที่สุดคือสร้าง mapping เอง:

```sql
SELECT
    order_date,
    CASE EXTRACT(MONTH FROM order_date)
        WHEN 1  THEN 'มกราคม'    WHEN 2  THEN 'กุมภาพันธ์'
        WHEN 3  THEN 'มีนาคม'    WHEN 4  THEN 'เมษายน'
        WHEN 5  THEN 'พฤษภาคม'   WHEN 6  THEN 'มิถุนายน'
        WHEN 7  THEN 'กรกฎาคม'   WHEN 8  THEN 'สิงหาคม'
        WHEN 9  THEN 'กันยายน'   WHEN 10 THEN 'ตุลาคม'
        WHEN 11 THEN 'พฤศจิกายน' WHEN 12 THEN 'ธันวาคม'
    END || ' ' || EXTRACT(YEAR FROM order_date)::text AS thai_month_year
FROM orders
WHERE order_id = 13;
```

**ผลลัพธ์:**

| order_date | thai_month_year |
|---|---|
| 2026-07-13 10:40:00+07 | กรกฎาคม 2026 |

เทคนิคนี้นิยมทำเป็น array lookup แบบ compact กว่านี้ก็ได้ (เร็วกว่าเล็กน้อยเมื่อข้อมูลมาก):

```sql
SELECT
    order_date,
    (ARRAY['มกราคม','กุมภาพันธ์','มีนาคม','เมษายน','พฤษภาคม','มิถุนายน',
           'กรกฎาคม','สิงหาคม','กันยายน','ตุลาคม','พฤศจิกายน','ธันวาคม']
    )[EXTRACT(MONTH FROM order_date)::int] AS thai_month
FROM orders
WHERE order_id = 13;
```

**ผลลัพธ์:**

| order_date | thai_month |
|---|---|
| 2026-07-13 10:40:00+07 | กรกฎาคม |

> หากต้องการ**ปีพุทธศักราช (พ.ศ.)** ให้บวกเพิ่ม 543 เข้ากับปี ค.ศ.: `EXTRACT(YEAR FROM order_date)::int + 543` — เป็นรูปแบบที่รายงานธุรกิจในไทยจำนวนมากต้องการ

```sql
SELECT
    order_date,
    TO_CHAR(order_date, 'DD') || '/' ||
    TO_CHAR(order_date, 'MM') || '/' ||
    (EXTRACT(YEAR FROM order_date)::int + 543)::text AS thai_buddhist_date
FROM orders
WHERE order_id = 13;
```

**ผลลัพธ์:**

| order_date | thai_buddhist_date |
|---|---|
| 2026-07-13 10:40:00+07 | 13/07/2569 |

---

## Step 317: GENERATE_SERIES กับวันที่ — อุดช่องว่าง (Fill Gaps)

ปัญหาคลาสสิกของรายงานสรุปตามช่วงเวลาคือ `GROUP BY` จะแสดงเฉพาะช่วงเวลาที่ **มีข้อมูลอยู่จริง** เท่านั้น หากเดือนไหนไม่มียอดขายเลย เดือนนั้นจะหายไปจากรายงาน ซึ่งอาจทำให้กราฟเส้นดูเหมือนกระโดดข้ามเดือน หรือระบบสรุปผลเข้าใจผิดว่าไม่มีเดือนนั้นอยู่

`GENERATE_SERIES(start, stop, step)` แก้ปัญหานี้ได้โดยสร้าง "ปฏิทิน" ของวันที่ต่อเนื่องขึ้นมาก่อน แล้วค่อย `LEFT JOIN` กับข้อมูลจริง

```sql
-- สร้างรายการวันที่ทุกวันในเดือนกันยายน 2026
SELECT GENERATE_SERIES('2026-09-01'::date, '2026-09-30'::date, '1 day')::date AS calendar_day;
```

**ผลลัพธ์ (ตัดมาบางส่วน):**

| calendar_day |
|-----------------|
| 2026-09-01      |
| 2026-09-02      |
| ...             |
| 2026-09-30      |

`GENERATE_SERIES` ยังใช้ step อื่นได้ เช่น `'1 month'`, `'1 week'`, `'1 hour'`:

```sql
SELECT GENERATE_SERIES('2026-01-01'::date, '2026-09-01'::date, '1 month')::date AS month_start;
```

**ผลลัพธ์:**

| month_start |
|---------------|
| 2026-01-01    |
| 2026-02-01    |
| 2026-03-01    |
| 2026-04-01    |
| 2026-05-01    |
| 2026-06-01    |
| 2026-07-01    |
| 2026-08-01    |
| 2026-09-01    |

### นำมาใช้อุดช่องว่างในรายงานยอดขายรายเดือน

```sql
WITH months AS (
    SELECT GENERATE_SERIES('2026-01-01'::date, '2026-09-01'::date, '1 month') AS month_start
),
monthly_sales AS (
    SELECT
        DATE_TRUNC('month', o.order_date)::date AS month_start,
        SUM(oi.quantity * oi.unit_price)        AS total_revenue,
        COUNT(DISTINCT o.order_id)              AS order_count
    FROM orders o
    JOIN order_items oi ON oi.order_id = o.order_id
    WHERE o.status = 'completed'
    GROUP BY 1
)
SELECT
    m.month_start,
    COALESCE(ms.order_count, 0)                   AS order_count,
    COALESCE(ms.total_revenue, 0)                 AS total_revenue
FROM months m
LEFT JOIN monthly_sales ms ON ms.month_start = m.month_start
ORDER BY m.month_start;
```

**ผลลัพธ์:**

| month_start | order_count | total_revenue |
|-------------|-------------|-----------------|
| 2026-01-01  | 0           | 0.00             |
| 2026-02-01  | 0           | 0.00             |
| 2026-03-01  | 0           | 0.00             |
| 2026-04-01  | 3           | 59850.00         |
| 2026-05-01  | 3           | 7440.00          |
| 2026-06-01  | 4           | 59540.00         |
| 2026-07-01  | 3           | 10820.00         |
| 2026-08-01  | 5           | 45610.00         |
| 2026-09-01  | 3           | 13640.00         |

ตอนนี้เราจะเห็นชัดเจนว่า มกราคม-มีนาคม 2026 ไม่มียอดขายเลย (0.00) แทนที่จะหายไปเงียบๆ จากรายงาน ซึ่งสำคัญมากเมื่อนำข้อมูลนี้ไปวาดกราฟเส้น (line chart) เพราะถ้าไม่มีแถวเหล่านี้ กราฟจะลากเส้นข้ามเดือนที่ไม่มีข้อมูลไปโดยไม่บอกผู้อ่านว่าจริงๆ แล้วยอดขายเป็นศูนย์

> เทคนิค `LEFT JOIN` จากตาราง "ปฏิทิน" (calendar/spine table) ไปยังตารางข้อมูลจริง คือ pattern มาตรฐานที่นักวิเคราะห์ข้อมูลใช้เสมอ ไม่ว่าจะเป็นรายวัน รายสัปดาห์ หรือรายชั่วโมง

---

## Step 318: Signup Cohort Analysis เบื้องต้น

Cohort analysis คือการจัดกลุ่มลูกค้าตาม "เหตุการณ์เริ่มต้น" ที่เกิดขึ้นในช่วงเวลาเดียวกัน (ในที่นี้คือเดือนที่สมัครสมาชิก) แล้ววิเคราะห์พฤติกรรมของแต่ละกลุ่มแยกกัน เป็นเทคนิคพื้นฐานที่ทีม growth/marketing ใช้ประเมินคุณภาพของลูกค้าที่ได้มาในแต่ละช่วงเวลา

### ขั้นที่ 1: จัดกลุ่มลูกค้าตามเดือนที่สมัคร (cohort)

```sql
SELECT
    DATE_TRUNC('month', signup_date)::date AS cohort_month,
    COUNT(*)                                AS customers_count
FROM customers
GROUP BY 1
ORDER BY 1;
```

**ผลลัพธ์:**

| cohort_month | customers_count |
|---------------|--------------------|
| 2025-03-01    | 1                   |
| 2025-04-01    | 1                   |
| 2025-06-01    | 1                   |
| 2025-07-01    | 1                   |
| 2025-08-01    | 1                   |
| 2025-09-01    | 1                   |
| 2026-01-01    | 2                   |
| 2026-02-01    | 3                   |
| 2026-03-01    | 2                   |
| 2026-04-01    | 2                   |

### ขั้นที่ 2: ดูยอดใช้จ่ายรวมของแต่ละ cohort

การจัดกลุ่มตาม cohort จะยิ่งมีประโยชน์เมื่อเราผสานกับข้อมูลออเดอร์เข้าไปด้วย เพื่อดูว่าลูกค้าแต่ละกลุ่มที่สมัครเข้ามาสร้างรายได้เท่าไรในภาพรวม:

```sql
SELECT
    DATE_TRUNC('month', c.signup_date)::date AS cohort_month,
    COUNT(DISTINCT c.customer_id)             AS customers_in_cohort,
    COUNT(DISTINCT o.order_id)                AS total_orders,
    COALESCE(SUM(oi.quantity * oi.unit_price), 0) AS total_revenue,
    ROUND(
        COALESCE(SUM(oi.quantity * oi.unit_price), 0)
        / COUNT(DISTINCT c.customer_id), 2
    ) AS revenue_per_customer
FROM customers c
LEFT JOIN orders o
    ON o.customer_id = c.customer_id AND o.status = 'completed'
LEFT JOIN order_items oi
    ON oi.order_id = o.order_id
GROUP BY 1
ORDER BY 1;
```

**ผลลัพธ์:**

| cohort_month | customers_in_cohort | total_orders | total_revenue | revenue_per_customer |
|---------------|------------------------|----------------|------------------|-------------------------|
| 2025-03-01    | 1                        | 4               | 72650.00          | 72650.00                 |
| 2025-04-01    | 1                        | 4               | 50550.00          | 50550.00                 |
| 2025-06-01    | 1                        | 3               | 28350.00          | 28350.00                 |
| 2025-07-01    | 1                        | 3               | 6540.00           | 6540.00                  |
| 2025-08-01    | 1                        | 1               | 890.00            | 890.00                   |
| 2025-09-01    | 1                        | 3               | 4730.00           | 4730.00                  |
| 2026-01-01    | 2                        | 3               | 37940.00          | 18970.00                 |
| 2026-02-01    | 3                        | 2               | 8230.00           | 2743.33                  |
| 2026-03-01    | 2                        | 2               | 36190.00          | 18095.00                 |
| 2026-04-01    | 2                        | 2               | 11570.00          | 5785.00                  |

จากตารางนี้ ลูกค้ากลุ่ม cohort เดือนมีนาคม 2025 (ลูกค้าคนแรกสุดของร้าน) มียอดใช้จ่ายเฉลี่ยต่อคนสูงที่สุด (72,650 บาท) ซึ่งสมเหตุสมผล เพราะเป็นลูกค้าที่อยู่กับร้านมานานที่สุดและมีเวลาสั่งซื้อสะสมมากกว่ากลุ่มอื่น — เป็นตัวอย่างว่าทำไม cohort analysis จึงต้องระวังเรื่อง "อายุของ cohort" (cohort age) ไม่ใช่เปรียบเทียบยอดรวมตรงๆ โดยไม่ปรับตามระยะเวลา

### ขั้นที่ 3: ใช้ GENERATE_SERIES เติมเดือนที่ไม่มี cohort (ไม่มีลูกค้าสมัครเลย)

```sql
WITH cohort_months AS (
    SELECT GENERATE_SERIES(
        DATE_TRUNC('month', (SELECT MIN(signup_date) FROM customers)),
        DATE_TRUNC('month', (SELECT MAX(signup_date) FROM customers)),
        '1 month'
    )::date AS cohort_month
),
actual_cohorts AS (
    SELECT DATE_TRUNC('month', signup_date)::date AS cohort_month, COUNT(*) AS customers_count
    FROM customers
    GROUP BY 1
)
SELECT cm.cohort_month, COALESCE(ac.customers_count, 0) AS customers_count
FROM cohort_months cm
LEFT JOIN actual_cohorts ac ON ac.cohort_month = cm.cohort_month
ORDER BY cm.cohort_month;
```

**ผลลัพธ์ (ตัดมาบางส่วน):**

| cohort_month | customers_count |
|---------------|--------------------|
| 2025-03-01    | 1                   |
| 2025-04-01    | 1                   |
| 2025-05-01    | 0                   |
| 2025-06-01    | 1                   |
| 2025-07-01    | 1                   |
| 2025-08-01    | 1                   |
| 2025-09-01    | 1                   |
| 2025-10-01    | 0                   |
| 2025-11-01    | 0                   |
| 2025-12-01    | 0                   |
| 2026-01-01    | 2                   |
| 2026-02-01    | 3                   |
| 2026-03-01    | 2                   |
| 2026-04-01    | 2                   |

ตอนนี้เดือนที่ไม่มีลูกค้าสมัครใหม่เลย (พ.ค. 2025, ต.ค.-ธ.ค. 2025) จะแสดงเป็น 0 แทนที่จะหายไปจากรายงาน — สำคัญมากเมื่อทีมการตลาดต้องวิเคราะห์ว่าช่วงไหนที่ campaign หาลูกค้าใหม่ไม่ได้ผลเลย

---

## Step 319: Window Function ร่วมกับวันที่ — Year-over-Year Growth

เมื่อนำ window function มาผสมกับการจัดกลุ่มตามวันที่ เราจะสามารถเปรียบเทียบค่าระหว่างช่วงเวลาได้อย่างทรงพลัง โดยไม่ต้อง self-join ตารางให้ยุ่งยาก

### เปรียบเทียบเดือนปัจจุบันกับเดือนก่อนหน้า (Month-over-Month)

```sql
WITH monthly_sales AS (
    SELECT
        DATE_TRUNC('month', o.order_date)::date AS sales_month,
        SUM(oi.quantity * oi.unit_price)        AS total_revenue
    FROM orders o
    JOIN order_items oi ON oi.order_id = o.order_id
    WHERE o.status = 'completed'
    GROUP BY 1
)
SELECT
    sales_month,
    total_revenue,
    LAG(total_revenue) OVER (ORDER BY sales_month)                    AS prev_month_revenue,
    total_revenue - LAG(total_revenue) OVER (ORDER BY sales_month)    AS mom_change,
    ROUND(
        100.0 * (total_revenue - LAG(total_revenue) OVER (ORDER BY sales_month))
        / NULLIF(LAG(total_revenue) OVER (ORDER BY sales_month), 0), 2
    ) AS mom_growth_pct
FROM monthly_sales
ORDER BY sales_month;
```

**ผลลัพธ์:**

| sales_month | total_revenue | prev_month_revenue | mom_change | mom_growth_pct |
|-------------|-----------------|------------------------|--------------|-------------------|
| 2026-04-01  | 59850.00         | NULL                    | NULL         | NULL               |
| 2026-05-01  | 7440.00          | 59850.00                | -52410.00    | -87.57             |
| 2026-06-01  | 59540.00         | 7440.00                 | 52100.00     | 700.27             |
| 2026-07-01  | 10820.00         | 59540.00                | -48720.00    | -81.83             |
| 2026-08-01  | 45610.00         | 10820.00                | 34790.00     | 321.53             |
| 2026-09-01  | 13640.00         | 45610.00                | -31970.00    | -70.10             |

การใช้ `NULLIF(..., 0)` ในตัวหารเป็นเทคนิคสำคัญเพื่อป้องกัน error `division by zero` เมื่อเดือนก่อนหน้ามียอดขายเท่ากับศูนย์พอดี

### เปรียบเทียบ Year-over-Year (YoY) ด้วย LAG แบบ offset 12

การเปรียบเทียบแบบปีต่อปีต้องมองย้อนกลับไป 12 แถว (12 เดือน) แทนที่จะเป็น 1 แถว โดยต้องแน่ใจว่าตารางรายเดือนของเรา "ต่อเนื่องไม่มีช่องว่าง" ก่อน (ใช้เทคนิคจาก Step 317) ไม่เช่นนั้น `LAG(x, 12)` จะเทียบผิดเดือน

```sql
WITH months AS (
    SELECT GENERATE_SERIES('2025-01-01'::date, '2026-09-01'::date, '1 month') AS sales_month
),
monthly_sales AS (
    SELECT
        DATE_TRUNC('month', o.order_date)::date AS sales_month,
        SUM(oi.quantity * oi.unit_price)        AS total_revenue
    FROM orders o
    JOIN order_items oi ON oi.order_id = o.order_id
    WHERE o.status = 'completed'
    GROUP BY 1
),
full_series AS (
    SELECT m.sales_month, COALESCE(ms.total_revenue, 0) AS total_revenue
    FROM months m
    LEFT JOIN monthly_sales ms ON ms.sales_month = m.sales_month
)
SELECT
    sales_month,
    total_revenue,
    LAG(total_revenue, 12) OVER (ORDER BY sales_month) AS revenue_same_month_last_year,
    ROUND(
        100.0 * (total_revenue - LAG(total_revenue, 12) OVER (ORDER BY sales_month))
        / NULLIF(LAG(total_revenue, 12) OVER (ORDER BY sales_month), 0), 2
    ) AS yoy_growth_pct
FROM full_series
WHERE sales_month >= '2026-04-01'   -- แสดงเฉพาะช่วงที่มีข้อมูลปีก่อนให้เทียบ
ORDER BY sales_month;
```

**ผลลัพธ์:**

| sales_month | total_revenue | revenue_same_month_last_year | yoy_growth_pct |
|-------------|-----------------|---------------------------------|-------------------|
| 2026-04-01  | 59850.00         | 32900.00                         | 81.91              |
| 2026-05-01  | 7440.00          | 21900.00                         | -66.03             |
| 2026-06-01  | 59540.00         | 2580.00                          | 2207.75            |
| 2026-07-01  | 10820.00         | 1290.00                          | 738.76             |
| 2026-08-01  | 45610.00         | 890.00                           | 5024.72             |
| 2026-09-01  | 13640.00         | 1180.00                          | 1055.93             |

> **ข้อควรระวังในการตีความ**: ข้อมูลตัวอย่างในบทนี้มีขนาดเล็กมาก (มีเพียงไม่กี่ออเดอร์ต่อเดือน) ทำให้ตัวเลข YoY growth ดูสุดโต่งเกินจริง (เช่น +5024%) นี่คือ "base effect" — เมื่อฐานปีก่อนหน้ามีค่าน้อยมาก แม้เพิ่มขึ้นเพียงเล็กน้อยก็ทำให้ % การเติบโตพุ่งสูงผิดปกติ ในข้อมูลจริงระดับองค์กรที่มีธุรกรรมหลักพัน-หมื่นรายการต่อเดือน ตัวเลข % จะมีความหมายและเสถียรกว่านี้มาก นักวิเคราะห์ข้อมูลที่ดีต้องระวังจุดนี้เสมอเมื่อนำเสนอ % growth ให้ผู้บริหาร

### จัดอันดับเดือนที่ขายดีที่สุดในแต่ละปีด้วย RANK()

```sql
SELECT
    EXTRACT(YEAR FROM sales_month)::int  AS sales_year,
    sales_month,
    total_revenue,
    RANK() OVER (PARTITION BY EXTRACT(YEAR FROM sales_month) ORDER BY total_revenue DESC) AS rank_in_year
FROM (
    SELECT
        DATE_TRUNC('month', o.order_date)::date AS sales_month,
        SUM(oi.quantity * oi.unit_price)        AS total_revenue
    FROM orders o
    JOIN order_items oi ON oi.order_id = o.order_id
    WHERE o.status = 'completed'
    GROUP BY 1
) monthly
ORDER BY sales_year, rank_in_year;
```

**ผลลัพธ์ (ตัดมาเฉพาะปี 2026):**

| sales_year | sales_month | total_revenue | rank_in_year |
|------------|-------------|-----------------|-----------------|
| 2026       | 2026-04-01  | 59850.00         | 1                |
| 2026       | 2026-06-01  | 59540.00         | 2                |
| 2026       | 2026-08-01  | 45610.00         | 3                |
| 2026       | 2026-09-01  | 13640.00         | 4                |
| 2026       | 2026-07-01  | 10820.00         | 5                |
| 2026       | 2026-05-01  | 7440.00          | 6                |

---

## Step 320: แบบฝึกหัดรวม — สร้างรายงานยอดขายรายเดือนแบบเต็มรูปแบบ

ตอนนี้เรามาผสานทุกเทคนิคที่เรียนมาในบทนี้ (`DATE_TRUNC`, `GENERATE_SERIES`, `TO_CHAR`, window function) เพื่อสร้างรายงานยอดขายรายเดือนที่พร้อมส่งให้ผู้บริหารดูได้จริง โดยต้องครอบคลุมทุกเดือนตั้งแต่มกราคมถึงกันยายน 2026 แม้เดือนที่ไม่มียอดขายก็ต้องแสดง

```sql
WITH months AS (
    SELECT GENERATE_SERIES('2026-01-01'::date, '2026-09-01'::date, '1 month') AS sales_month
),
monthly_sales AS (
    SELECT
        DATE_TRUNC('month', o.order_date)::date AS sales_month,
        COUNT(DISTINCT o.order_id)              AS order_count,
        SUM(oi.quantity * oi.unit_price)        AS total_revenue
    FROM orders o
    JOIN order_items oi ON oi.order_id = o.order_id
    WHERE o.status = 'completed'
    GROUP BY 1
),
full_report AS (
    SELECT
        m.sales_month,
        COALESCE(ms.order_count, 0)   AS order_count,
        COALESCE(ms.total_revenue, 0) AS total_revenue
    FROM months m
    LEFT JOIN monthly_sales ms ON ms.sales_month = m.sales_month
)
SELECT
    (ARRAY['มกราคม','กุมภาพันธ์','มีนาคม','เมษายน','พฤษภาคม','มิถุนายน',
           'กรกฎาคม','สิงหาคม','กันยายน','ตุลาคม','พฤศจิกายน','ธันวาคม']
    )[EXTRACT(MONTH FROM sales_month)::int] || ' ' ||
    (EXTRACT(YEAR FROM sales_month)::int + 543)::text          AS month_label_th,
    TO_CHAR(sales_month, 'YYYY-MM')                             AS month_key,
    order_count,
    total_revenue,
    CASE WHEN order_count > 0
         THEN ROUND(total_revenue / order_count, 2)
         ELSE 0
    END                                                          AS avg_order_value,
    SUM(total_revenue) OVER (ORDER BY sales_month)               AS running_total_revenue,
    ROUND(
        100.0 * (total_revenue - LAG(total_revenue) OVER (ORDER BY sales_month))
        / NULLIF(LAG(total_revenue) OVER (ORDER BY sales_month), 0), 2
    )                                                              AS mom_growth_pct
FROM full_report
ORDER BY sales_month;
```

**ผลลัพธ์:**

| month_label_th | month_key | order_count | total_revenue | avg_order_value | running_total_revenue | mom_growth_pct |
|------------------|-----------|-------------|-----------------|--------------------|--------------------------|--------------------|
| มกราคม 2569       | 2026-01   | 0           | 0.00             | 0.00                | 0.00                      | NULL                |
| กุมภาพันธ์ 2569     | 2026-02   | 0           | 0.00             | 0.00                | 0.00                      | NULL                |
| มีนาคม 2569        | 2026-03   | 0           | 0.00             | 0.00                | 0.00                      | NULL                |
| เมษายน 2569        | 2026-04   | 3           | 59850.00         | 19950.00            | 59850.00                  | NULL                |
| พฤษภาคม 2569       | 2026-05   | 3           | 7440.00          | 2480.00             | 67290.00                  | -87.57              |
| มิถุนายน 2569       | 2026-06   | 4           | 59540.00         | 14885.00            | 126830.00                 | 700.27              |
| กรกฎาคม 2569       | 2026-07   | 3           | 10820.00         | 3606.67             | 137650.00                 | -81.83              |
| สิงหาคม 2569        | 2026-08   | 5           | 45610.00         | 9122.00             | 183260.00                 | 321.53              |
| กันยายน 2569        | 2026-09   | 3           | 13640.00         | 4546.67             | 196900.00                 | -70.10              |

รายงานนี้แสดงให้เห็นครบทุกมิติที่ทีมบริหารต้องการในหน้าเดียว: ชื่อเดือนภาษาไทยพร้อมปี พ.ศ., จำนวนออเดอร์, ยอดขายรวม, ยอดเฉลี่ยต่อออเดอร์, ยอดสะสม (running total) และอัตราการเติบโตเทียบเดือนก่อนหน้า — ครบทุกเทคนิคที่เราเรียนมาตั้งแต่ Step 311 ถึง Step 319 ในตัวอย่างเดียว

---

## สรุปท้ายบท

ในบทนี้เราได้เจาะลึกฟังก์ชันวันที่-เวลาที่ใช้บ่อยที่สุดในงาน reporting และ analytics ของ PostgreSQL:

- **`NOW()` / `CURRENT_TIMESTAMP`** คงที่ตลอด transaction ส่วน **`CLOCK_TIMESTAMP()`** เปลี่ยนแปลงตามเวลาจริงเสมอ — ต้องเลือกใช้ให้ถูกกับบริบท
- **`DATE_TRUNC`** ปัดวันที่ลงไปยังจุดเริ่มต้นของหน่วยที่ต้องการ (day/week/month/quarter/year) เป็นหัวใจของการ `GROUP BY` ตามช่วงเวลา
- **`EXTRACT` / `DATE_PART`** ดึงส่วนประกอบของวันที่ออกมาวิเคราะห์ เช่น วันในสัปดาห์ (`DOW`/`ISODOW`) เพื่อเปรียบเทียบพฤติกรรมวันธรรมดากับวันหยุด
- **`AGE()`** คำนวณระยะเวลาแบบปี-เดือน-วันที่มนุษย์อ่านเข้าใจได้ทันที แม่นยำกว่าการหารจำนวนวันแบบหยาบๆ
- การนับ **วันทำงาน (business days)** ทำได้ด้วยการผสาน `GENERATE_SERIES` กับ `EXTRACT(ISODOW ...)` เพื่อกรองเสาร์-อาทิตย์ออก
- **`TO_CHAR`** คือเครื่องมือหลักในการจัดรูปแบบวันที่สำหรับรายงาน รวมถึงการสร้างชื่อเดือนภาษาไทยและปี พ.ศ. ด้วย lookup array
- **`GENERATE_SERIES`** กับวันที่คือเทคนิคสำคัญที่สุดในการ "อุดช่องว่าง" ของรายงาน ทำให้ช่วงเวลาที่ไม่มีข้อมูลแสดงเป็น 0 แทนที่จะหายไปเงียบๆ
- **Cohort analysis** เบื้องต้นทำได้ด้วยการจัดกลุ่มลูกค้าตามเดือนที่สมัคร (`DATE_TRUNC('month', signup_date)`) แล้วผสานกับข้อมูลออเดอร์
- **Window function** อย่าง `LAG()` ร่วมกับตารางเดือนที่ต่อเนื่องไม่มีช่องว่าง ช่วยให้คำนวณ month-over-month และ year-over-year growth ได้อย่างแม่นยำ

เทคนิคทั้งหมดในบทนี้เป็นพื้นฐานสำคัญของงาน business intelligence และ data analytics แทบทุกประเภท ไม่ว่าจะเป็นการสร้าง dashboard, รายงานผู้บริหาร, หรือการวิเคราะห์พฤติกรรมลูกค้าเชิงลึก ในบทถัดไปเราจะไปดูฟังก์ชันทางคณิตศาสตร์และตัวเลข (numeric functions) ที่ใช้คู่กับฟังก์ชันวันที่เหล่านี้ในการทำรายงานสรุปต่างๆ

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1
เขียนคำสั่ง SQL หายอดขายรวม (จาก `order_items` ของออเดอร์ที่ `status = 'completed'`) โดยจัดกลุ่มตาม**ไตรมาส** ของปี 2026 (ใช้ `DATE_TRUNC('quarter', ...)`)

<details>
<summary>เฉลย</summary>

```sql
SELECT
    DATE_TRUNC('quarter', o.order_date)::date AS quarter_start,
    SUM(oi.quantity * oi.unit_price) AS total_revenue
FROM orders o
JOIN order_items oi ON oi.order_id = o.order_id
WHERE o.status = 'completed'
  AND o.order_date >= '2026-01-01' AND o.order_date < '2027-01-01'
GROUP BY 1
ORDER BY 1;
```

ผลลัพธ์ที่ได้: Q2 2026 (2026-04-01) รวม 76,770.00 บาท (เม.ย.+พ.ค.+มิ.ย. = 59850+7440+59540) และ Q3 2026 (2026-07-01) รวม 70,070.00 บาท (ก.ค.+ส.ค.+ก.ย. = 10820+45610+13640)

</details>

### แบบฝึกหัดที่ 2
ใช้ `EXTRACT` หาว่าพนักงาน (employees) คนใดทำงานมานานกว่า 3 ปีแล้ว (นับถึง `CURRENT_DATE`) โดยแสดงชื่อและจำนวนปีที่ทำงาน (ปัดเศษเป็นจำนวนเต็ม)

<details>
<summary>เฉลย</summary>

```sql
SELECT
    first_name || ' ' || last_name AS employee_name,
    hire_date,
    EXTRACT(YEAR FROM AGE(CURRENT_DATE, hire_date))::int AS years_employed
FROM employees
WHERE EXTRACT(YEAR FROM AGE(CURRENT_DATE, hire_date)) >= 3
ORDER BY hire_date;
```

ผลลัพธ์ที่ได้ (อิงวันที่ 2026-09-25): Piti Kunakorn (4 ปี), Orawan Suthep (4 ปี), Decha Wattana (3 ปี), Napa Sirisak (3 ปี)

</details>

### แบบฝึกหัดที่ 3
คำนวณจำนวนวันทำงาน (ไม่นับเสาร์-อาทิตย์) ระหว่างวันที่ 1 มิถุนายน 2026 ถึง 30 มิถุนายน 2026

<details>
<summary>เฉลย</summary>

```sql
SELECT COUNT(*) AS business_days
FROM GENERATE_SERIES('2026-06-01'::date, '2026-06-30'::date, '1 day') AS d
WHERE EXTRACT(ISODOW FROM d) NOT IN (6, 7);
```

เดือนมิถุนายน 2026 มี 30 วัน เริ่มต้นวันจันทร์ (1 มิ.ย. เป็นวันจันทร์) ผลลัพธ์คือ 22 วันทำงาน

</details>

### แบบฝึกหัดที่ 4
ใช้ `TO_CHAR` แสดงวันที่รีวิวสินค้า (`reviews.review_date`) ในรูปแบบ `"วันที่ 13 กรกฎาคม 2026"` (ใช้ lookup array สำหรับชื่อเดือนภาษาไทย)

<details>
<summary>เฉลย</summary>

```sql
SELECT
    review_id,
    review_date,
    'วันที่ ' || EXTRACT(DAY FROM review_date)::text || ' ' ||
    (ARRAY['มกราคม','กุมภาพันธ์','มีนาคม','เมษายน','พฤษภาคม','มิถุนายน',
           'กรกฎาคม','สิงหาคม','กันยายน','ตุลาคม','พฤศจิกายน','ธันวาคม']
    )[EXTRACT(MONTH FROM review_date)::int] || ' ' ||
    EXTRACT(YEAR FROM review_date)::text AS thai_full_date
FROM reviews
ORDER BY review_date;
```

</details>

### แบบฝึกหัดที่ 5
เขียน query สร้างปฏิทินวันที่ต่อเนื่องตั้งแต่ 1-30 กันยายน 2026 แล้ว `LEFT JOIN` กับ `orders` เพื่อแสดงจำนวนออเดอร์ในแต่ละวัน (รวมวันที่ไม่มีออเดอร์เลยด้วย โดยแสดงเป็น 0)

<details>
<summary>เฉลย</summary>

```sql
WITH days AS (
    SELECT GENERATE_SERIES('2026-09-01'::date, '2026-09-30'::date, '1 day') AS the_day
)
SELECT
    d.the_day,
    COUNT(o.order_id) AS order_count
FROM days d
LEFT JOIN orders o ON o.order_date::date = d.the_day
GROUP BY d.the_day
ORDER BY d.the_day;
```

</details>

### แบบฝึกหัดที่ 6
จัดกลุ่มลูกค้าตาม cohort เดือนที่สมัคร (`DATE_TRUNC('month', signup_date)`) แล้วหาว่า cohort ใดมีจำนวนลูกค้ามากที่สุด

<details>
<summary>เฉลย</summary>

```sql
SELECT
    DATE_TRUNC('month', signup_date)::date AS cohort_month,
    COUNT(*) AS customers_count
FROM customers
GROUP BY 1
ORDER BY customers_count DESC, cohort_month
LIMIT 1;
```

ผลลัพธ์: cohort เดือนกุมภาพันธ์ 2026 (2026-02-01) มีลูกค้ามากที่สุด 3 คน (Prasert, Siriporn, Thanawat)

</details>

### แบบฝึกหัดที่ 7
ใช้ window function หาว่าเดือนใดใน 2026 มียอดขาย (จากออเดอร์ที่ completed) เพิ่มขึ้นจากเดือนก่อนหน้ามากที่สุด (เป็นจำนวนเงิน ไม่ใช่ %)

<details>
<summary>เฉลย</summary>

```sql
WITH monthly_sales AS (
    SELECT
        DATE_TRUNC('month', o.order_date)::date AS sales_month,
        SUM(oi.quantity * oi.unit_price) AS total_revenue
    FROM orders o
    JOIN order_items oi ON oi.order_id = o.order_id
    WHERE o.status = 'completed'
    GROUP BY 1
),
with_diff AS (
    SELECT
        sales_month,
        total_revenue,
        total_revenue - LAG(total_revenue) OVER (ORDER BY sales_month) AS mom_change
    FROM monthly_sales
)
SELECT sales_month, total_revenue, mom_change
FROM with_diff
WHERE mom_change IS NOT NULL
ORDER BY mom_change DESC
LIMIT 1;
```

ผลลัพธ์: เดือนมิถุนายน 2026 เพิ่มขึ้นจากพฤษภาคม 2026 มากที่สุด (+52,100.00 บาท)

</details>

### แบบฝึกหัดที่ 8
เขียน query หาว่าการชำระเงิน (`payments`) ที่เกิดขึ้นในวันหยุดสุดสัปดาห์ (เสาร์-อาทิตย์) มีสัดส่วนเป็นกี่เปอร์เซ็นต์ของจำนวนการชำระเงินทั้งหมด

<details>
<summary>เฉลย</summary>

```sql
SELECT
    ROUND(
        100.0 * COUNT(*) FILTER (WHERE EXTRACT(ISODOW FROM payment_date) IN (6, 7))
        / COUNT(*), 2
    ) AS weekend_payment_pct
FROM payments;
```

ใช้ `FILTER` clause (ที่เรียนไปแล้วใน Part 027) ร่วมกับ `EXTRACT(ISODOW ...)` เพื่อนับเฉพาะแถวที่ตรงเงื่อนไขโดยไม่ต้อง subquery แยก

</details>

### แบบฝึกหัดที่ 9
สร้างรายงานเปรียบเทียบยอดขายเดือนสิงหาคม 2026 กับเดือนสิงหาคม 2025 (year-over-year) แสดงยอดขายทั้งสองปีและ % การเติบโต

<details>
<summary>เฉลย</summary>

```sql
WITH monthly_sales AS (
    SELECT
        DATE_TRUNC('month', o.order_date)::date AS sales_month,
        SUM(oi.quantity * oi.unit_price) AS total_revenue
    FROM orders o
    JOIN order_items oi ON oi.order_id = o.order_id
    WHERE o.status = 'completed'
    GROUP BY 1
)
SELECT
    (SELECT total_revenue FROM monthly_sales WHERE sales_month = '2025-08-01') AS aug_2025_revenue,
    (SELECT total_revenue FROM monthly_sales WHERE sales_month = '2026-08-01') AS aug_2026_revenue,
    ROUND(
        100.0 * (
            (SELECT total_revenue FROM monthly_sales WHERE sales_month = '2026-08-01') -
            (SELECT total_revenue FROM monthly_sales WHERE sales_month = '2025-08-01')
        ) / (SELECT total_revenue FROM monthly_sales WHERE sales_month = '2025-08-01'), 2
    ) AS yoy_growth_pct;
```

ผลลัพธ์: สิงหาคม 2025 = 890.00 บาท, สิงหาคม 2026 = 45,610.00 บาท, เติบโต +5024.72% (ตัวเลขสูงมากเพราะฐานปีก่อนหน้ามีขนาดเล็ก ดังที่อธิบายไว้ใน Step 319)

</details>

### แบบฝึกหัดที่ 10
สร้างรายงานยอดขายรายเดือนแบบเต็มรูปแบบตั้งแต่มกราคมถึงธันวาคม 2026 (ครอบคลุมทั้งปี รวมเดือนในอนาคตที่ยังไม่มีข้อมูล) แสดงชื่อเดือนภาษาไทย จำนวนออเดอร์ และยอดขายรวม โดยเดือนที่ไม่มีข้อมูลให้แสดงเป็น 0

<details>
<summary>เฉลย</summary>

```sql
WITH months AS (
    SELECT GENERATE_SERIES('2026-01-01'::date, '2026-12-01'::date, '1 month') AS sales_month
),
monthly_sales AS (
    SELECT
        DATE_TRUNC('month', o.order_date)::date AS sales_month,
        COUNT(DISTINCT o.order_id) AS order_count,
        SUM(oi.quantity * oi.unit_price) AS total_revenue
    FROM orders o
    JOIN order_items oi ON oi.order_id = o.order_id
    WHERE o.status = 'completed'
    GROUP BY 1
)
SELECT
    (ARRAY['มกราคม','กุมภาพันธ์','มีนาคม','เมษายน','พฤษภาคม','มิถุนายน',
           'กรกฎาคม','สิงหาคม','กันยายน','ตุลาคม','พฤศจิกายน','ธันวาคม']
    )[EXTRACT(MONTH FROM m.sales_month)::int] || ' ' ||
    (EXTRACT(YEAR FROM m.sales_month)::int + 543)::text AS month_label_th,
    COALESCE(ms.order_count, 0) AS order_count,
    COALESCE(ms.total_revenue, 0) AS total_revenue
FROM months m
LEFT JOIN monthly_sales ms ON ms.sales_month = m.sales_month
ORDER BY m.sales_month;
```

เดือนตุลาคม-ธันวาคม 2026 (ที่ยังไม่ถึงเวลาจริงในข้อมูลตัวอย่าง) จะแสดงเป็น 0 เช่นเดียวกับมกราคม-มีนาคม เพราะยังไม่มีออเดอร์ใน `orders` เลย เทคนิคนี้มีประโยชน์มากสำหรับการทำ template รายงานที่ต้องเตรียมโครงสร้างไว้ล่วงหน้าทั้งปี

</details>

---

**บทถัดไป**: [Part 033 — Numeric Functions และการคำนวณเชิงตัวเลข](./part-033-numeric-functions.md)
