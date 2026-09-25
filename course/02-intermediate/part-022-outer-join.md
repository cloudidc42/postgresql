# OUTER JOIN: LEFT, RIGHT, FULL — เก็บข้อมูลที่ไม่มีคู่ให้ครบ

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับกลาง | Part 022

---

## เป้าหมายการเรียนรู้

เมื่อเรียนจบบทนี้ ผู้เรียนจะสามารถ:

1. เข้าใจความแตกต่างระหว่าง `INNER JOIN` กับ `OUTER JOIN` ทั้งในแง่แนวคิดและผลลัพธ์จริง
2. เขียน `LEFT OUTER JOIN` เพื่อดึงข้อมูล "ทุกแถว" จากตารางหลัก แม้จะไม่มีข้อมูลคู่กันในอีกตารางหนึ่ง
3. ใช้ pattern `LEFT JOIN ... WHERE ... IS NULL` เพื่อหาแถวที่ "ไม่มีคู่" (anti-join) เช่น ลูกค้าที่ไม่เคยสั่งซื้อ
4. เข้าใจ `RIGHT OUTER JOIN` และเหตุผลที่ทีมพัฒนามืออาชีพส่วนใหญ่นิยมเขียนเป็น `LEFT JOIN` แทนเพื่อความสม่ำเสมอของโค้ด
5. ใช้ `FULL OUTER JOIN` เพื่อรวมข้อมูลทั้งหมดจากทั้งสองตาราง ไม่ว่าฝั่งไหนจะมีคู่หรือไม่
6. ผสม `INNER JOIN` และ `OUTER JOIN` ในคิวรีเดียวกันได้อย่างถูกต้อง
7. หลีกเลี่ยงข้อผิดพลาดที่พบบ่อยที่สุด — การวาง condition ใน `WHERE` ผิดตำแหน่งจนทำให้ `OUTER JOIN` กลายเป็น `INNER JOIN` โดยไม่ตั้งใจ
8. ใช้ `COALESCE()` ร่วมกับ `OUTER JOIN` เพื่อแสดงค่า default แทนที่จะเป็น `NULL`
9. เข้าใจภาพรวมว่า query planner จัดการ `OUTER JOIN` ต่างจาก `INNER JOIN` อย่างไร และเหตุใดจึงมีข้อจำกัดเรื่องการจัดลำดับ join
10. เขียนรายงานเชิงธุรกิจจริงที่ต้องพึ่งพา `OUTER JOIN` เช่น ลูกค้าที่ไม่เคยซื้อสินค้า, สินค้าที่ไม่เคยถูกรีวิว, หมวดหมู่ที่ไม่มีสินค้าเลย

---

## เตรียมข้อมูล

บทนี้อยู่ในชุด Part 021–039 ของระดับกลาง ซึ่งใช้ฐานข้อมูลร้านค้าออนไลน์ (e-commerce) ชุดเดียวกันตลอดทั้งชุด เพื่อให้ผู้เรียนคุ้นเคยกับโครงสร้างข้อมูลและไม่ต้องเรียนรู้สคีมาใหม่ทุกบท

**สิ่งสำคัญสำหรับบทนี้โดยเฉพาะ**: เนื่องจากบทนี้ว่าด้วยเรื่อง `OUTER JOIN` ข้อมูลตัวอย่างจึงถูกออกแบบให้มี "แถวกำพร้า" (orphan rows) ที่ไม่มีคู่ในตารางที่เกี่ยวข้องโดยตั้งใจ เช่น ลูกค้าที่ไม่เคยสั่งซื้อ สินค้าที่ไม่เคยถูกรีวิว หมวดหมู่ที่ไม่มีสินค้า ฯลฯ เพื่อให้เห็นพฤติกรรมการเติม `NULL` ของ `OUTER JOIN` ได้อย่างชัดเจน

หากยังไม่มีฐานข้อมูลนี้ในเครื่อง ให้รันสคริปต์ทั้งหมดด้านล่างนี้ก่อน (แนะนำให้สร้างฐานข้อมูลใหม่ชื่อ `ecommerce_course` แล้วรันภายในนั้น):

```sql
-- ล้างตารางเดิม (ถ้ามี) เพื่อให้รันซ้ำได้โดยไม่ error
DROP TABLE IF EXISTS payments CASCADE;
DROP TABLE IF EXISTS reviews CASCADE;
DROP TABLE IF EXISTS order_items CASCADE;
DROP TABLE IF EXISTS orders CASCADE;
DROP TABLE IF EXISTS employees CASCADE;
DROP TABLE IF EXISTS customers CASCADE;
DROP TABLE IF EXISTS products CASCADE;
DROP TABLE IF EXISTS suppliers CASCADE;
DROP TABLE IF EXISTS categories CASCADE;

-- โครงสร้างตาราง (เหมือนกันทุกบทใน Part 021-039)
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
    employee_id  SERIAL PRIMARY KEY,
    first_name   VARCHAR(60) NOT NULL,
    last_name    VARCHAR(60) NOT NULL,
    hire_date    DATE NOT NULL,
    manager_id   INTEGER REFERENCES employees(employee_id),
    department   VARCHAR(60)
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
    review_id    SERIAL PRIMARY KEY,
    product_id   INTEGER REFERENCES products(product_id),
    customer_id  INTEGER REFERENCES customers(customer_id),
    rating       INTEGER NOT NULL CHECK (rating BETWEEN 1 AND 5),
    review_text  TEXT,
    review_date  DATE NOT NULL DEFAULT CURRENT_DATE
);

CREATE TABLE payments (
    payment_id      SERIAL PRIMARY KEY,
    order_id        INTEGER REFERENCES orders(order_id),
    payment_date    TIMESTAMPTZ NOT NULL DEFAULT now(),
    amount          NUMERIC(10,2) NOT NULL,
    payment_method  VARCHAR(30)
);
```

ต่อไปนี้คือข้อมูลตัวอย่าง สังเกตคอมเมนต์ `-- (กำพร้า)` ที่กำกับแถวซึ่งตั้งใจไม่ให้มีคู่ในตารางที่เกี่ยวข้อง:

```sql
-- ===== categories =====
-- category_id 9 "Toys" ไม่มีสินค้าใดอ้างอิงถึงเลย (กำพร้า - ใช้สาธิต OUTER JOIN)
INSERT INTO categories (category_name, parent_category_id) VALUES
('Electronics', NULL),        -- 1
('Computers', 1),             -- 2
('Smartphones', 1),           -- 3
('Home Appliances', NULL),    -- 4
('Kitchen', 4),                -- 5
('Books', NULL),               -- 6  (สินค้าจริงถูกจัดใน subcategory 7,8 ไม่ใช่ตัวนี้ตรงๆ)
('Fiction', 6),                -- 7
('Non-Fiction', 6),            -- 8
('Toys', NULL),                -- 9  -- (กำพร้า: ไม่มีสินค้าเลย)
('Sports', NULL);              -- 10

-- ===== suppliers =====
-- supplier_id 7 "FitLife Supplies" ไม่มีสินค้าใดอ้างอิงถึงเลย (กำพร้า)
INSERT INTO suppliers (supplier_name, country) VALUES
('TechWorld Co., Ltd.', 'Thailand'),      -- 1
('Global Gadgets Inc.', 'USA'),           -- 2
('Sakura Electronics', 'Japan'),          -- 3
('EuroHome Supplies', 'Germany'),         -- 4
('Pacific Traders', 'China'),             -- 5
('BookHouse Publishing', 'Thailand'),     -- 6
('FitLife Supplies', 'Thailand'),         -- 7  -- (กำพร้า: ไม่มีสินค้าจากซัพพลายเออร์นี้)
('Nordic Design', 'Sweden');              -- 8

-- ===== products =====
-- product_id 9 (Blender Max) ไม่ active, ไม่เคยถูกสั่งซื้อ, ไม่เคยถูกรีวิว
-- product_id 15 (Yoga Mat Premium) ไม่เคยถูกรีวิวเลย (กำพร้าฝั่ง reviews)
-- product_id 16 (Mystery Grab Box) category_id เป็น NULL (สินค้ายังไม่ถูกจัดหมวดหมู่ - กำพร้าอีกแบบ)
INSERT INTO products (product_name, category_id, supplier_id, unit_price, stock_quantity, is_active) VALUES
('Laptop Pro 15', 2, 1, 32900.00, 15, true),                    -- 1
('Wireless Mouse M1', 2, 1, 590.00, 120, true),                  -- 2
('Mechanical Keyboard K2', 2, 2, 2490.00, 45, true),             -- 3
('Smartphone X12', 3, 3, 21900.00, 30, true),                    -- 4
('Smartphone Lite S5', 3, 5, 8990.00, 60, true),                 -- 5
('Bluetooth Earbuds Air', 1, 2, 1290.00, 200, true),             -- 6
('Rice Cooker Deluxe', 5, 4, 1590.00, 40, true),                 -- 7
('Air Fryer 5L', 5, 4, 2990.00, 25, true),                       -- 8
('Blender Max', 5, 5, 1190.00, 0, false),                        -- 9  -- (กำพร้า: ไม่เคยขาย/ไม่เคยรีวิว)
('Vacuum Cleaner Robot', 4, 8, 8900.00, 12, true),               -- 10
('Thriller Novel: Midnight Run', 7, 6, 350.00, 80, true),        -- 11
('Sci-Fi Saga: Star Drift', 7, 6, 420.00, 55, true),             -- 12
('Cookbook: Thai Kitchen', 8, 6, 480.00, 30, true),              -- 13
('History of Siam', 8, 6, 520.00, 18, true),                     -- 14
('Yoga Mat Premium', 10, 8, 890.00, 70, true),                   -- 15 -- (กำพร้า: ไม่เคยถูกรีวิว)
('Mystery Grab Box', NULL, 1, 199.00, 500, true);                -- 16 -- (กำพร้า: ยังไม่มี category)

-- ===== customers =====
-- customer_id 9 (Nattapong Srisuk) สมัครสมาชิกแล้วแต่ไม่เคยสั่งซื้อเลย (กำพร้า - ใช้สาธิต LEFT JOIN บ่อยที่สุดในบทนี้)
INSERT INTO customers (first_name, last_name, email, country, signup_date) VALUES
('Somchai', 'Jaidee', 'somchai.j@example.com', 'Thailand', '2023-01-15'),      -- 1
('Suda', 'Sriwan', 'suda.s@example.com', 'Thailand', '2023-02-20'),            -- 2
('Anong', 'Phutjirakul', 'anong.p@example.com', 'Thailand', '2023-03-05'),     -- 3
('John', 'Smith', 'john.smith@example.com', 'USA', '2023-01-10'),              -- 4
('Emily', 'Chen', 'emily.chen@example.com', 'Singapore', '2023-04-18'),        -- 5
('Kenji', 'Tanaka', 'kenji.t@example.com', 'Japan', '2023-05-22'),             -- 6
('Maria', 'Garcia', 'maria.g@example.com', 'Spain', '2023-06-30'),             -- 7
('Wichai', 'Boonmee', 'wichai.b@example.com', 'Thailand', '2023-02-14'),       -- 8
('Nattapong', 'Srisuk', 'nattapong.s@example.com', 'Thailand', '2024-08-01'),  -- 9  -- (กำพร้า: ไม่เคยสั่งซื้อ)
('Lisa', 'Wong', 'lisa.wong@example.com', 'Malaysia', '2023-07-11'),           -- 10
('David', 'Miller', 'david.miller@example.com', 'UK', '2023-08-25'),           -- 11
('Piyada', 'Kongkaew', 'piyada.k@example.com', 'Thailand', '2023-09-09');      -- 12

-- ===== employees =====
-- employee_id 4,5 (แผนก Support) ไม่เคยดูแล order เลยเพราะไม่ใช่ฝ่ายขาย (กำพร้า)
-- employee_id 6 (พนักงานใหม่) ยังไม่เคยดูแล order เลย (กำพร้า)
INSERT INTO employees (first_name, last_name, hire_date, manager_id, department) VALUES
('Prasert', 'Wongsawang', '2020-01-10', NULL, 'Sales'),    -- 1 (ผู้จัดการฝ่ายขาย)
('Kanya', 'Thepsuriya', '2020-06-15', 1, 'Sales'),          -- 2
('Anucha', 'Meesuk', '2021-03-01', 1, 'Sales'),             -- 3
('Ratana', 'Chaiyaporn', '2019-11-20', NULL, 'Support'),    -- 4 -- (กำพร้า: ไม่เคยดูแล order)
('Somsak', 'Phromchan', '2021-09-05', 4, 'Support'),        -- 5 -- (กำพร้า: ไม่เคยดูแล order)
('Napat', 'Wattana', '2025-01-06', 1, 'Sales');             -- 6 -- (กำพร้า: พนักงานใหม่ ยังไม่มี order)

-- ===== orders =====
-- customer_id 9 ไม่ปรากฏใน orders เลย, employee_id 4,5,6 ไม่ปรากฏใน orders เลย
INSERT INTO orders (customer_id, employee_id, order_date, status, ship_country) VALUES
(1,  2, '2024-01-05 09:12:00+07', 'completed', 'Thailand'),  -- 1
(1,  2, '2024-03-12 14:30:00+07', 'completed', 'Thailand'),  -- 2
(2,  3, '2024-01-20 10:05:00+07', 'completed', 'Thailand'),  -- 3
(3,  2, '2024-02-02 16:45:00+07', 'completed', 'Thailand'),  -- 4
(4,  1, '2024-02-10 08:20:00-05', 'completed', 'USA'),       -- 5
(5,  3, '2024-02-15 11:00:00+08', 'cancelled', 'Singapore'), -- 6 -- (ยกเลิก ไม่มี payment)
(6,  1, '2024-03-01 13:15:00+09', 'completed', 'Japan'),     -- 7
(7,  2, '2024-03-18 09:40:00+01', 'completed', 'Spain'),     -- 8
(8,  3, '2024-04-02 15:25:00+07', 'completed', 'Thailand'),  -- 9
(10, 1, '2024-04-20 12:00:00+08', 'completed', 'Malaysia'),  -- 10
(11, 2, '2024-05-05 17:30:00+01', 'pending', 'UK'),          -- 11 -- (pending ไม่มี payment)
(12, 3, '2024-05-15 10:10:00+07', 'completed', 'Thailand'),  -- 12
(2,  1, '2024-06-01 09:00:00+07', 'completed', 'Thailand'),  -- 13
(4,  2, '2024-06-10 18:20:00-05', 'shipped', 'USA'),         -- 14
(1,  3, '2024-07-01 10:45:00+07', 'completed', 'Thailand'),  -- 15
(8,  1, '2024-07-15 14:00:00+07', 'pending', 'Thailand');    -- 16 -- (pending ไม่มี payment)

-- ===== order_items =====
INSERT INTO order_items (order_id, product_id, quantity, unit_price) VALUES
(1,  1,  1, 32900.00),
(1,  2,  1, 590.00),
(2,  6,  2, 1290.00),
(3,  4,  1, 21900.00),
(4,  7,  1, 1590.00),
(5,  3,  1, 2490.00),
(6,  5,  1, 8990.00),
(7,  11, 2, 350.00),
(8,  13, 1, 480.00),
(9,  8,  1, 2990.00),
(10, 2,  3, 590.00),
(11, 1,  1, 32900.00),
(12, 14, 1, 520.00),
(13, 6,  1, 1290.00),
(13, 12, 2, 420.00),
(14, 10, 1, 8900.00),
(15, 2,  2, 590.00),
(15, 3,  1, 2490.00),
(16, 11, 1, 350.00),
(16, 13, 1, 480.00);

-- ===== reviews =====
-- product_id 9, 10, 14, 15, 16 ไม่มีรีวิวใดๆ เลย (กำพร้าหลายตัวโดยตั้งใจ เพื่อใช้สาธิตในหลาย Step)
INSERT INTO reviews (product_id, customer_id, rating, review_text, review_date) VALUES
(1,  1,  5, 'เร็วมาก คุ้มราคา ทำงานลื่นไหลดี', '2024-01-10'),
(1,  3,  4, 'ดีมาก แต่จอสว่างไปนิดตอนเปิดเครื่อง', '2024-02-05'),
(2,  2,  5, 'ใช้งานลื่นไหล จับถนัดมือ', '2024-01-25'),
(4,  3,  4, 'กล้องคมชัด แบตอยู่ได้ทั้งวัน', '2024-02-08'),
(6,  1,  5, 'เสียงดีมาก ตัดเสียงรบกวนได้ดี', '2024-01-15'),
(7,  4,  3, 'หุงข้าวได้ดีแต่เสียงดังตอนทำงาน', '2024-02-15'),
(8,  8,  5, 'ทอดกรอบอร่อย ทำความสะอาดง่าย', '2024-04-10'),
(11, 6,  5, 'อ่านเพลินมาก วางไม่ลงเลย', '2024-03-10'),
(12, 2,  4, 'พล็อตดี แต่ตอนจบยังงงๆ', '2024-06-05'),
(13, 7,  5, 'สูตรอาหารเยี่ยม ทำตามได้ง่าย', '2024-03-25'),
(3,  5,  4, 'พิมพ์สบายมือ แป้นตอบสนองดี', '2024-02-20'),
(2,  12, 4, 'ดีใช้ได้ ราคาสมเหตุสมผล', '2024-06-20');

-- ===== payments =====
-- order_id 6 (cancelled), 11, 16 (pending) ไม่มี payment เลย (กำพร้า - ใช้สาธิตใน Step 218)
INSERT INTO payments (order_id, payment_date, amount, payment_method) VALUES
(1,  '2024-01-05 09:20:00+07', 33490.00, 'credit_card'),
(2,  '2024-03-12 14:40:00+07', 2580.00,  'credit_card'),
(3,  '2024-01-20 10:15:00+07', 21900.00, 'promptpay'),
(4,  '2024-02-02 17:00:00+07', 1590.00,  'credit_card'),
(5,  '2024-02-10 08:30:00-05', 2490.00,  'paypal'),
(7,  '2024-03-01 13:25:00+09', 700.00,   'credit_card'),
(8,  '2024-03-18 09:50:00+01', 480.00,   'promptpay'),
(9,  '2024-04-02 15:35:00+07', 2990.00,  'credit_card'),
(10, '2024-04-20 12:10:00+08', 1770.00,  'paypal'),
(12, '2024-05-15 10:20:00+07', 520.00,   'credit_card'),
(13, '2024-06-01 09:10:00+07', 2130.00,  'promptpay'),
(14, '2024-06-10 18:30:00-05', 8900.00,  'credit_card'),
(15, '2024-07-01 10:55:00+07', 3670.00,  'credit_card');
```

**สรุปแถวกำพร้าที่ตั้งใจสร้างไว้ในชุดข้อมูลนี้** (จะใช้สาธิตตลอดทั้งบท):

| ตาราง | แถวกำพร้า | ไม่มีคู่ในตาราง |
|---|---|---|
| `categories` | Toys (id 9) | `products` |
| `categories` | Books (id 6, สินค้าจริงอยู่ใน subcategory) | `products` (ตรงๆ) |
| `suppliers` | FitLife Supplies (id 7) | `products` |
| `products` | Blender Max (id 9) | `order_items`, `reviews` |
| `products` | Yoga Mat Premium (id 15) | `reviews` |
| `products` | Mystery Grab Box (id 16) | `categories` (category_id เป็น NULL) |
| `customers` | Nattapong Srisuk (id 9) | `orders`, `reviews` |
| `employees` | Ratana, Somsak, Napat (id 4, 5, 6) | `orders` |
| `orders` | id 6, 11, 16 | `payments` |

---

## Step 211: LEFT OUTER JOIN (LEFT JOIN) — แนวคิดและ syntax

### แนวคิด

`INNER JOIN` ที่เรียนไปในบทก่อนหน้าจะคืนเฉพาะแถวที่ **มีคู่ตรงกันทั้งสองฝั่ง** เท่านั้น ถ้าแถวไหนในตารางใดตารางหนึ่งไม่มีคู่ตรงกันเลย แถวนั้นจะถูกตัดทิ้งไปจากผลลัพธ์ทันที

`LEFT OUTER JOIN` (นิยมเขียนสั้นๆ ว่า `LEFT JOIN` ซึ่งเป็นคำเดียวกัน คำว่า `OUTER` เป็น optional ใน PostgreSQL) แก้ปัญหานี้โดยยืนยันว่า **ทุกแถวจากตารางฝั่งซ้าย (ตารางที่เขียนก่อน `LEFT JOIN`) จะถูกเก็บไว้ในผลลัพธ์เสมอ** ไม่ว่าจะมีคู่ตรงกันในตารางฝั่งขวาหรือไม่ก็ตาม ถ้าแถวฝั่งซ้ายไม่มีคู่ตรงกัน คอลัมน์ทั้งหมดที่มาจากตารางฝั่งขวาจะถูกเติมด้วยค่า `NULL`

```
LEFT JOIN คือ:  ทุกแถวจากตารางซ้าย + (แถวคู่จากตารางขวา หรือ NULL ถ้าไม่มีคู่)
```

### Syntax

```sql
SELECT columns
FROM ตารางซ้าย
LEFT JOIN ตารางขวา ON ตารางซ้าย.คีย์ = ตารางขวา.คีย์;

-- เขียนแบบเต็มก็ได้ ความหมายเหมือนกันทุกประการ
SELECT columns
FROM ตารางซ้าย
LEFT OUTER JOIN ตารางขวา ON ตารางซ้าย.คีย์ = ตารางขวา.คีย์;
```

### ตัวอย่างแรก: categories LEFT JOIN products

ลองดูหมวดหมู่สินค้าทั้งหมด พร้อมสินค้าที่อยู่ในหมวดนั้น (ถ้ามี):

```sql
SELECT
    c.category_id,
    c.category_name,
    p.product_name,
    p.unit_price
FROM categories c
LEFT JOIN products p ON p.category_id = c.category_id
ORDER BY c.category_id, p.product_id;
```

ผลลัพธ์ (17 แถว จาก 10 หมวดหมู่ + สินค้า 15 ชิ้นที่มี category_id ไม่เป็น NULL):

| category_id | category_name | product_name | unit_price |
|---|---|---|---|
| 1 | Electronics | Bluetooth Earbuds Air | 1290.00 |
| 2 | Computers | Laptop Pro 15 | 32900.00 |
| 2 | Computers | Wireless Mouse M1 | 590.00 |
| 2 | Computers | Mechanical Keyboard K2 | 2490.00 |
| 3 | Smartphones | Smartphone X12 | 21900.00 |
| 3 | Smartphones | Smartphone Lite S5 | 8990.00 |
| 4 | Home Appliances | Vacuum Cleaner Robot | 8900.00 |
| 5 | Kitchen | Rice Cooker Deluxe | 1590.00 |
| 5 | Kitchen | Air Fryer 5L | 2990.00 |
| 5 | Kitchen | Blender Max | 1190.00 |
| 6 | Books | **NULL** | **NULL** |
| 7 | Fiction | Thriller Novel: Midnight Run | 350.00 |
| 7 | Fiction | Sci-Fi Saga: Star Drift | 420.00 |
| 8 | Non-Fiction | Cookbook: Thai Kitchen | 480.00 |
| 8 | Non-Fiction | History of Siam | 520.00 |
| 9 | Toys | **NULL** | **NULL** |
| 10 | Sports | Yoga Mat Premium | 890.00 |

สังเกตแถว `category_id = 6` (Books) และ `category_id = 9` (Toys) — ทั้งสองไม่มีสินค้าที่อ้างอิง `category_id` มาตรงๆ (สินค้าหนังสือจริงถูกจัดในหมวดย่อย Fiction/Non-Fiction ส่วน Toys ไม่มีสินค้าเลย) คอลัมน์ `product_name` และ `unit_price` จึงถูกเติมด้วย `NULL` แทนที่จะหายไปจากผลลัพธ์ นี่คือหัวใจของ `LEFT JOIN`

> **เทียบกับ INNER JOIN**: ถ้าใช้ `INNER JOIN` แทน `LEFT JOIN` ในคิวรีเดียวกัน ผลลัพธ์จะเหลือแค่ 15 แถว (เท่ากับจำนวนสินค้าที่มี `category_id`) เพราะหมวด Books และ Toys จะถูกตัดทิ้งไปเลย นี่คือความแตกต่างที่สำคัญที่สุดที่ต้องจำให้ขึ้นใจ

### ทำไมต้องใส่ ON แยกจาก WHERE

`LEFT JOIN` จะพิจารณาเงื่อนไขใน `ON` เพื่อ "จับคู่" ก่อน แล้วค่อยเก็บแถวฝั่งซ้ายทั้งหมดไว้ (ไม่ว่าจับคู่ได้หรือไม่) — เรื่องนี้สำคัญมากและจะอธิบายอย่างละเอียดใน Step 217 เพราะเป็นจุดที่มือใหม่พลาดบ่อยที่สุด

---

## Step 212: ตัวอย่าง LEFT JOIN จริง — หาลูกค้าทุกคนพร้อม order (รวมคนที่ไม่เคยสั่งซื้อ)

นี่คือ use case ที่พบบ่อยที่สุดของ `LEFT JOIN` ในระบบ e-commerce: อยากเห็น "ลูกค้าทุกคน" พร้อมประวัติการสั่งซื้อ แม้ว่าลูกค้าบางคนจะยังไม่เคยสั่งซื้อเลยก็ตาม

```sql
SELECT
    c.customer_id,
    c.first_name,
    c.last_name,
    o.order_id,
    o.order_date,
    o.status
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.customer_id
ORDER BY c.customer_id, o.order_id;
```

ผลลัพธ์บางส่วน (แสดงเฉพาะช่วงที่น่าสนใจ จากทั้งหมด 16 แถว):

| customer_id | first_name | last_name | order_id | order_date | status |
|---|---|---|---|---|---|
| 1 | Somchai | Jaidee | 1 | 2024-01-05 09:12:00+07 | completed |
| 1 | Somchai | Jaidee | 2 | 2024-03-12 14:30:00+07 | completed |
| 1 | Somchai | Jaidee | 15 | 2024-07-01 10:45:00+07 | completed |
| 2 | Suda | Sriwan | 3 | 2024-01-20 10:05:00+07 | completed |
| 2 | Suda | Sriwan | 13 | 2024-06-01 09:00:00+07 | completed |
| ... | ... | ... | ... | ... | ... |
| 8 | Wichai | Boonmee | 9 | 2024-04-02 15:25:00+07 | completed |
| 8 | Wichai | Boonmee | 16 | 2024-07-15 14:00:00+07 | pending |
| **9** | **Nattapong** | **Srisuk** | **NULL** | **NULL** | **NULL** |
| 10 | Lisa | Wong | 10 | 2024-04-20 12:00:00+08 | completed |
| 11 | David | Miller | 11 | 2024-05-05 17:30:00+01 | pending |
| 12 | Piyada | Kongkaew | 12 | 2024-05-15 10:10:00+07 | completed |

**Nattapong Srisuk (customer_id 9)** ปรากฏในผลลัพธ์เพียงแถวเดียว โดยคอลัมน์จากตาราง `orders` ทั้งหมดเป็น `NULL` เพราะเขาไม่เคยสั่งซื้อเลย — ถ้าใช้ `INNER JOIN` เขาจะหายไปจากรายงานทั้งหมด ทั้งที่จริงๆ แล้วเขาเป็นสมาชิกที่มีตัวตนในระบบ

### ใช้ร่วมกับ aggregate: นับจำนวนออเดอร์ต่อลูกค้า

```sql
SELECT
    c.customer_id,
    c.first_name || ' ' || c.last_name AS full_name,
    COUNT(o.order_id) AS total_orders
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.customer_id
GROUP BY c.customer_id, c.first_name, c.last_name
ORDER BY total_orders DESC, c.customer_id;
```

ผลลัพธ์ (12 แถว ครบทุกคน):

| customer_id | full_name | total_orders |
|---|---|---|
| 1 | Somchai Jaidee | 3 |
| 2 | Suda Sriwan | 2 |
| 4 | John Smith | 2 |
| 8 | Wichai Boonmee | 2 |
| 3 | Anong Phutjirakul | 1 |
| 5 | Emily Chen | 1 |
| 6 | Kenji Tanaka | 1 |
| 7 | Maria Garcia | 1 |
| 10 | Lisa Wong | 1 |
| 11 | David Miller | 1 |
| 12 | Piyada Kongkaew | 1 |
| **9** | **Nattapong Srisuk** | **0** |

จุดสำคัญ: `COUNT(o.order_id)` นับเฉพาะ `order_id` ที่ไม่เป็น `NULL` ดังนั้น Nattapong จึงได้ค่า `0` ไม่ใช่ `NULL` — แต่ถ้าเผลอเขียน `COUNT(*)` แทน จะได้ `1` เพราะ `COUNT(*)` นับ "จำนวนแถว" ไม่ใช่ "จำนวนค่าที่ไม่ใช่ NULL" นี่คือข้อผิดพลาดเล็กๆ ที่พบบ่อยมากเวลาใช้ `LEFT JOIN` ร่วมกับ aggregate function — **กฎทอง: เวลานับแถวที่มาจากฝั่งขวาของ LEFT JOIN ให้ใช้ `COUNT(คอลัมน์ที่มาจากตารางขวา)` เสมอ ไม่ใช่ `COUNT(*)`**

---

## Step 213: การใช้ IS NULL ร่วมกับ LEFT JOIN เพื่อหาแถวที่ไม่มีคู่ (Anti-Join Pattern)

จาก Step 212 เราเห็นแล้วว่า `LEFT JOIN` เติม `NULL` ให้แถวที่ไม่มีคู่ ดังนั้นถ้าเราอยากหา "เฉพาะ" แถวที่ไม่มีคู่ (เช่น ลูกค้าที่ไม่เคยสั่งซื้อ) เราก็แค่กรองด้วย `WHERE ... IS NULL` บนคอลัมน์ที่มาจากตารางฝั่งขวา (ควรใช้คอลัมน์ที่เป็น primary key หรือ NOT NULL เพื่อความชัดเจน)

Pattern นี้เรียกว่า **anti-join** — เทคนิคที่ใช้บ่อยมากในการทำรายงานเชิงธุรกิจ

```sql
SELECT
    c.customer_id,
    c.first_name,
    c.last_name,
    c.email,
    c.signup_date
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.customer_id
WHERE o.order_id IS NULL
ORDER BY c.customer_id;
```

ผลลัพธ์:

| customer_id | first_name | last_name | email | signup_date |
|---|---|---|---|---|
| 9 | Nattapong | Srisuk | nattapong.s@example.com | 2024-08-01 |

ได้ลูกค้าที่ไม่เคยสั่งซื้อเพียงคนเดียว ตรงกับที่เราตั้งใจสร้างไว้ในข้อมูล — รายงานแบบนี้มีประโยชน์มากในทางธุรกิจ เช่น เอาไปส่งอีเมลโปรโมชั่นกระตุ้นการสั่งซื้อครั้งแรก

### ตัวอย่างที่สอง: สินค้าที่ไม่เคยถูกสั่งซื้อเลย

```sql
SELECT
    p.product_id,
    p.product_name,
    p.stock_quantity,
    p.is_active
FROM products p
LEFT JOIN order_items oi ON oi.product_id = p.product_id
WHERE oi.order_item_id IS NULL
ORDER BY p.product_id;
```

ผลลัพธ์:

| product_id | product_name | stock_quantity | is_active |
|---|---|---|---|
| 9 | Blender Max | 0 | false |
| 15 | Yoga Mat Premium | 70 | true |
| 16 | Mystery Grab Box | 500 | true |

สินค้าเหล่านี้ยังไม่เคยถูกสั่งซื้อเลย — `Blender Max` เข้าใจได้เพราะถูกปิดการขายไปแล้ว (`is_active = false`) แต่ `Yoga Mat Premium` และ `Mystery Grab Box` ยังขายอยู่ (`is_active = true`) แต่มีสต็อกเหลือเยอะและยังไม่มีออเดอร์เข้ามาเลย — เป็นสัญญาณที่ฝ่ายการตลาดควรให้ความสนใจ

> **ข้อควรระวัง**: อย่าใช้ `WHERE oi.product_id IS NULL` ในตัวอย่างนี้ เพราะถ้าคอลัมน์ `product_id` ในตาราง `order_items` เกิดมีค่า `NULL` ขึ้นมาจริงๆ (ซึ่งเป็นไปได้เพราะ schema ไม่ได้บังคับ `NOT NULL`) จะทำให้ตีความผิดได้ ควรเช็คด้วย primary key ของตารางขวาเสมอ เช่น `oi.order_item_id IS NULL` เพื่อความปลอดภัยและชัดเจนที่สุด

---

## Step 214: RIGHT OUTER JOIN (RIGHT JOIN) และทำไมมักเขียนเป็น LEFT JOIN แทน

### แนวคิด

`RIGHT OUTER JOIN` (หรือ `RIGHT JOIN`) ทำงานตรงข้ามกับ `LEFT JOIN` — คือ **เก็บทุกแถวจากตารางฝั่งขวา** (ตารางที่เขียนหลัง `RIGHT JOIN`) แม้จะไม่มีคู่ตรงกันในตารางฝั่งซ้ายก็ตาม

```
RIGHT JOIN คือ:  (แถวคู่จากตารางซ้าย หรือ NULL ถ้าไม่มีคู่) + ทุกแถวจากตารางขวา
```

### ตัวอย่าง: หาพนักงานทุกคนพร้อมออเดอร์ที่ดูแล (รวมคนที่ไม่เคยดูแลออเดอร์เลย)

```sql
SELECT
    e.employee_id,
    e.first_name,
    e.last_name,
    e.department,
    o.order_id,
    o.order_date
FROM orders o
RIGHT JOIN employees e ON o.employee_id = e.employee_id
ORDER BY e.employee_id, o.order_id;
```

ผลลัพธ์บางส่วน (จากทั้งหมด 19 แถว: 16 ออเดอร์ + 3 พนักงานที่ไม่มีออเดอร์):

| employee_id | first_name | last_name | department | order_id | order_date |
|---|---|---|---|---|---|
| 1 | Prasert | Wongsawang | Sales | 5 | 2024-02-10 08:20:00-05 |
| 1 | Prasert | Wongsawang | Sales | 7 | 2024-03-01 13:15:00+09 |
| 1 | Prasert | Wongsawang | Sales | 10 | 2024-04-20 12:00:00+08 |
| 1 | Prasert | Wongsawang | Sales | 13 | 2024-06-01 09:00:00+07 |
| 1 | Prasert | Wongsawang | Sales | 16 | 2024-07-15 14:00:00+07 |
| ... | ... | ... | ... | ... | ... |
| **4** | **Ratana** | **Chaiyaporn** | **Support** | **NULL** | **NULL** |
| **5** | **Somsak** | **Phromchan** | **Support** | **NULL** | **NULL** |
| **6** | **Napat** | **Wattana** | **Sales** | **NULL** | **NULL** |

จะเห็นว่าพนักงานฝ่าย Support (Ratana, Somsak) และพนักงานใหม่ (Napat) ไม่มีออเดอร์ที่ดูแลเลย แต่ยังปรากฏในรายงาน เพราะ `employees` เป็นตารางฝั่งขวาของ `RIGHT JOIN`

### ทำไมทีมมืออาชีพมักหลีกเลี่ยง RIGHT JOIN

คิวรีข้างบนสามารถเขียนใหม่ให้ผลลัพธ์เหมือนกันทุกประการโดยใช้ `LEFT JOIN` เพียงแค่สลับลำดับตาราง:

```sql
-- ผลลัพธ์เหมือนกับ RIGHT JOIN ด้านบนทุกประการ เพียงสลับตำแหน่งตาราง
SELECT
    e.employee_id,
    e.first_name,
    e.last_name,
    e.department,
    o.order_id,
    o.order_date
FROM employees e
LEFT JOIN orders o ON o.employee_id = e.employee_id
ORDER BY e.employee_id, o.order_id;
```

เหตุผลที่ทีมพัฒนาส่วนใหญ่เลือกใช้ `LEFT JOIN` แทน `RIGHT JOIN` เสมอ มีดังนี้:

1. **อ่านง่ายกว่าเมื่อ join หลายตาราง**: เวลาเขียน query ที่ join ตั้งแต่ 3-4 ตารางขึ้นไป การไล่จากซ้ายไปขวาด้วย `LEFT JOIN` ทุกตัวจะอ่านเป็นลำดับตรรกะเดียวกันตลอดทั้งคิวรี ในขณะที่ถ้าสลับ `LEFT`/`RIGHT` ไปมาจะทำให้สับสนว่าตารางไหนคือ "ตารางหลัก" ที่ต้องการเก็บทุกแถว
2. **เป็นธรรมเนียมปฏิบัติ (convention) ที่ทีมส่วนใหญ่ยึดถือ**: เมื่อทุกคนในทีมเขียน `LEFT JOIN` เป็นมาตรฐานเดียวกัน การรีวิวโค้ดและแก้ไขบั๊กจะง่ายขึ้นมาก เพราะไม่ต้องตีความทิศทางสลับไปมา
3. **`RIGHT JOIN` ไม่มีความสามารถอะไรที่ `LEFT JOIN` ทำไม่ได้**: ทุก `RIGHT JOIN` สามารถแปลงเป็น `LEFT JOIN` ที่ให้ผลลัพธ์เหมือนกันทุกประการได้เสมอ เพียงแค่สลับลำดับตารางสองฝั่ง จึงไม่มีเหตุผลทางเทคนิคที่ต้องใช้ `RIGHT JOIN`

> **หมายเหตุ**: `RIGHT JOIN` ไม่ใช่คำสั่งที่ผิดหรือล้าสมัย และ query planner ของ PostgreSQL จัดการมันได้อย่างมีประสิทธิภาพเท่ากับ `LEFT JOIN` ทุกประการ (ภายในจะถูกแปลง plan ให้เหมือนกัน) เพียงแต่เป็นเรื่องของ**สไตล์การเขียนโค้ดที่สม่ำเสมอ**ในทีมเท่านั้น หากไปเจอโค้ดเก่าที่ใช้ `RIGHT JOIN` ก็ควรอ่านให้เข้าใจได้เช่นกัน

---

## Step 215: FULL OUTER JOIN — รวมทุกแถวจากทั้งสองตาราง

### แนวคิด

`FULL OUTER JOIN` (หรือ `FULL JOIN`) คือการรวมพฤติกรรมของทั้ง `LEFT JOIN` และ `RIGHT JOIN` เข้าด้วยกัน — **เก็บทุกแถวจากทั้งสองตาราง** ไม่ว่าฝั่งไหนจะมีคู่ตรงกันหรือไม่ก็ตาม ถ้าฝั่งไหนไม่มีคู่ คอลัมน์จากฝั่งนั้นจะเป็น `NULL`

```
FULL OUTER JOIN คือ:
  แถวที่จับคู่กันได้ทั้งสองฝั่ง
  + แถวฝั่งซ้ายที่ไม่มีคู่ (ฝั่งขวาเป็น NULL)
  + แถวฝั่งขวาที่ไม่มีคู่ (ฝั่งซ้ายเป็น NULL)
```

### ตัวอย่างที่เห็นผลชัดเจน: categories กับ products

ข้อมูลของเราถูกออกแบบมาให้เห็นผลทั้งสองทิศทางในคิวรีเดียว: หมวดหมู่ `Toys` ไม่มีสินค้า (ฝั่งซ้ายไม่มีคู่) และสินค้า `Mystery Grab Box` ยังไม่ถูกจัดหมวดหมู่ (ฝั่งขวาไม่มีคู่)

```sql
SELECT
    c.category_id,
    c.category_name,
    p.product_id,
    p.product_name
FROM categories c
FULL OUTER JOIN products p ON p.category_id = c.category_id
ORDER BY c.category_id NULLS LAST, p.product_id;
```

ผลลัพธ์ (18 แถว):

| category_id | category_name | product_id | product_name |
|---|---|---|---|
| 1 | Electronics | 6 | Bluetooth Earbuds Air |
| 2 | Computers | 1 | Laptop Pro 15 |
| 2 | Computers | 2 | Wireless Mouse M1 |
| 2 | Computers | 3 | Mechanical Keyboard K2 |
| 3 | Smartphones | 4 | Smartphone X12 |
| 3 | Smartphones | 5 | Smartphone Lite S5 |
| 4 | Home Appliances | 10 | Vacuum Cleaner Robot |
| 5 | Kitchen | 7 | Rice Cooker Deluxe |
| 5 | Kitchen | 8 | Air Fryer 5L |
| 5 | Kitchen | 9 | Blender Max |
| 6 | Books | **NULL** | **NULL** |
| 7 | Fiction | 11 | Thriller Novel: Midnight Run |
| 7 | Fiction | 12 | Sci-Fi Saga: Star Drift |
| 8 | Non-Fiction | 13 | Cookbook: Thai Kitchen |
| 8 | Non-Fiction | 14 | History of Siam |
| **9** | **Toys** | **NULL** | **NULL** |
| 10 | Sports | 15 | Yoga Mat Premium |
| **NULL** | **NULL** | **16** | **Mystery Grab Box** |

สังเกตแถวสุดท้าย: `category_id` และ `category_name` เป็น `NULL` เพราะไม่มีหมวดหมู่ใดที่ `Mystery Grab Box` (product_id 16) จับคู่ได้ (เนื่องจากมันมี `category_id = NULL`) — นี่คือแถวที่ `LEFT JOIN` (ในทิศทาง categories → products) จะไม่มีทางแสดงให้เห็นเลย เพราะ `LEFT JOIN` รับประกันเฉพาะแถวฝั่งซ้าย (`categories`) เท่านั้น ส่วน `RIGHT JOIN` เพียงอย่างเดียวก็จะไม่แสดงแถว `Toys` และ `Books` ที่ไม่มีสินค้า

**`FULL OUTER JOIN` คือทางเดียวที่จะเห็นทั้งสองด้านของปัญหาในคิวรีเดียว**

### เมื่อไหร่ควรใช้ FULL OUTER JOIN

- การกระทบยอด (reconciliation) ข้อมูลระหว่างสองระบบ เช่น เทียบรายการสั่งซื้อในระบบ ERP กับระบบชำระเงิน เพื่อหาทั้ง "ออเดอร์ที่ยังไม่ได้จ่ายเงิน" และ "การจ่ายเงินที่ไม่มีออเดอร์อ้างอิง" (อาจเป็นข้อมูลผิดพลาดหรือทุจริต)
- การตรวจสอบความสมบูรณ์ของข้อมูล (data quality audit) ก่อน migrate ระบบ
- รายงานเปรียบเทียบสองช่วงเวลา เช่น สินค้าที่ขายในเดือนนี้เทียบกับเดือนที่แล้ว (หาสินค้าที่หายไปและสินค้าใหม่ในคิวรีเดียว)

> **หมายเหตุด้านเทคนิค**: PostgreSQL ตั้งแต่เวอร์ชัน 13 เป็นต้นมา รองรับ hash join algorithm สำหรับ `FULL OUTER JOIN` และ `RIGHT OUTER JOIN` ทำให้ประสิทธิภาพดีขึ้นมากเมื่อเทียบกับเวอร์ชันเก่าที่ต้องพึ่ง merge join หรือ nested loop เพียงอย่างเดียว ใน PostgreSQL 16/17 ที่เราใช้ในหลักสูตรนี้ query planner จึงมีทางเลือก (strategy) ที่หลากหลายและมีประสิทธิภาพดีในการจัดการ `FULL OUTER JOIN`

---

## Step 216: การ JOIN หลายตารางแบบผสม INNER และ OUTER JOIN ในคำสั่งเดียว

ในงานจริง แทบไม่มีคิวรีไหนที่ใช้ join ประเภทเดียวตลอดทั้งคำสั่ง ส่วนใหญ่จะต้องผสมกันตามความหมายทางธุรกิจของแต่ละความสัมพันธ์:

- ความสัมพันธ์ที่ "ต้องมี" คู่กันเสมอ (เช่น `order_items` ต้องอ้างอิง `orders` และ `products` ที่มีอยู่จริง) → ใช้ `INNER JOIN`
- ความสัมพันธ์ที่ "อาจจะไม่มี" คู่กัน (เช่น สินค้าอาจจะยังไม่มีรีวิว) → ใช้ `LEFT JOIN`

### ตัวอย่าง: รายงานยอดขายสินค้าพร้อมคะแนนรีวิวเฉลี่ย (ถ้ามี)

โจทย์: อยากรู้ว่าสินค้าที่**เคยขายแล้ว**แต่ละชิ้น ขายไปทั้งหมดกี่ชิ้น และมีคะแนนรีวิวเฉลี่ยเท่าไหร่ (ถ้ายังไม่มีใครรีวิว ให้แสดง `NULL`)

```sql
SELECT
    p.product_id,
    p.product_name,
    SUM(oi.quantity) AS total_sold,
    ROUND(AVG(r.rating), 2) AS avg_rating
FROM products p
JOIN order_items oi ON oi.product_id = p.product_id   -- INNER: สนใจเฉพาะสินค้าที่เคยขายแล้วจริงๆ
LEFT JOIN reviews r ON r.product_id = p.product_id     -- LEFT: รีวิวจะมีหรือไม่มีก็ได้
GROUP BY p.product_id, p.product_name
ORDER BY total_sold DESC, p.product_id;
```

ผลลัพธ์ (13 แถว — เฉพาะสินค้าที่เคยขาย จากทั้งหมด 16 ชิ้น เพราะ id 9, 15, 16 ไม่เคยขายจึงถูกตัดออกด้วย `INNER JOIN` กับ `order_items`):

| product_id | product_name | total_sold | avg_rating |
|---|---|---|---|
| 2 | Wireless Mouse M1 | 6 | 4.50 |
| 6 | Bluetooth Earbuds Air | 3 | 5.00 |
| 11 | Thriller Novel: Midnight Run | 3 | 5.00 |
| 1 | Laptop Pro 15 | 2 | 4.50 |
| 3 | Mechanical Keyboard K2 | 2 | 4.00 |
| 12 | Sci-Fi Saga: Star Drift | 2 | 4.00 |
| 13 | Cookbook: Thai Kitchen | 2 | 5.00 |
| 4 | Smartphone X12 | 1 | 4.00 |
| 5 | Smartphone Lite S5 | 1 | **NULL** |
| 7 | Rice Cooker Deluxe | 1 | 3.00 |
| 8 | Air Fryer 5L | 1 | 5.00 |
| 10 | Vacuum Cleaner Robot | 1 | **NULL** |
| 14 | History of Siam | 1 | **NULL** |

สังเกตว่า:
- สินค้า `Blender Max` (id 9), `Yoga Mat Premium` (id 15), `Mystery Grab Box` (id 16) **หายไปทั้งหมด** เพราะไม่เคยถูกขาย และเราใช้ `INNER JOIN` กับ `order_items` — นี่คือพฤติกรรมที่ตั้งใจ เพราะโจทย์บอกว่า "อยากรู้เฉพาะสินค้าที่เคยขายแล้ว"
- สินค้า `Smartphone Lite S5`, `Vacuum Cleaner Robot`, `History of Siam` **ยังอยู่ในผลลัพธ์** แม้จะไม่มีรีวิวเลย (`avg_rating = NULL`) เพราะ `reviews` ใช้ `LEFT JOIN` — ตรงตามโจทย์ที่ว่า "รีวิวจะมีหรือไม่ก็ได้"

นี่คือพลังของการผสม `INNER JOIN` กับ `LEFT JOIN` — เราควบคุมได้อย่างละเอียดว่าความสัมพันธ์ไหน "บังคับต้องมี" และความสัมพันธ์ไหน "มีก็ได้ไม่มีก็ได้"

### ตัวอย่างที่ซับซ้อนขึ้น: join 4 ตารางผสมกัน

```sql
SELECT
    o.order_id,
    c.first_name || ' ' || c.last_name AS customer_name,
    o.status,
    pay.amount AS paid_amount,
    pay.payment_method
FROM orders o
JOIN customers c ON c.customer_id = o.customer_id     -- INNER: order ต้องมีลูกค้าเสมอ (NOT NULL ทางธุรกิจ)
LEFT JOIN payments pay ON pay.order_id = o.order_id     -- LEFT: บาง order ยังไม่จ่ายเงิน
ORDER BY o.order_id;
```

ผลลัพธ์บางส่วน (16 แถว รวมออเดอร์ทั้งหมด):

| order_id | customer_name | status | paid_amount | payment_method |
|---|---|---|---|---|
| 5 | John Smith | completed | 2490.00 | paypal |
| **6** | **Emily Chen** | **cancelled** | **NULL** | **NULL** |
| 7 | Kenji Tanaka | completed | 700.00 | credit_card |
| ... | ... | ... | ... | ... |
| **11** | **David Miller** | **pending** | **NULL** | **NULL** |
| ... | ... | ... | ... | ... |
| **16** | **Wichai Boonmee** | **pending** | **NULL** | **NULL** |

ออเดอร์ที่ถูกยกเลิก (id 6) หรือยังอยู่ในสถานะ `pending` (id 11, 16) ยังไม่มี payment จึงแสดงเป็น `NULL` แต่ยังคงปรากฏในรายงานครบถ้วน — เหมาะมากสำหรับทีมการเงินที่ต้องการเห็นภาพรวมทุกออเดอร์ ไม่ใช่แค่ออเดอร์ที่จ่ายเงินแล้ว

---

## Step 217: ผลกระทบของ WHERE clause ที่วางผิดตำแหน่งกับ OUTER JOIN

นี่คือ**จุดที่มือใหม่พลาดบ่อยที่สุด**เวลาใช้ `OUTER JOIN` — และเป็นสาเหตุของบั๊กที่ตรวจจับยากมาก เพราะคิวรีจะรันผ่านได้ปกติไม่มี error ใดๆ เพียงแต่ผลลัพธ์ผิดโดยไม่รู้ตัว

### ปัญหา: WHERE บนตารางขวาทำให้ LEFT JOIN กลายเป็น INNER JOIN

สมมติเราต้องการ "ลูกค้าทุกคน พร้อมออเดอร์ที่สถานะ `completed` เท่านั้น (ถ้าไม่มีให้เป็น NULL)"

**เขียนแบบผิด** (ข้อผิดพลาดที่พบบ่อยมาก):

```sql
-- ❌ ผิด: WHERE ทำงานหลังจาก JOIN เสร็จสมบูรณ์แล้ว
SELECT
    c.customer_id,
    c.first_name,
    o.order_id,
    o.status
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.customer_id
WHERE o.status = 'completed'
ORDER BY c.customer_id;
```

ผลลัพธ์ที่ได้จริง (เหลือเพียง 11 แถว จาก 9 ลูกค้า):

| customer_id | first_name | order_id | status |
|---|---|---|---|
| 1 | Somchai | 1 | completed |
| 1 | Somchai | 2 | completed |
| 1 | Somchai | 15 | completed |
| 2 | Suda | 3 | completed |
| 2 | Suda | 13 | completed |
| 3 | Anong | 4 | completed |
| 4 | John | 5 | completed |
| 6 | Kenji | 7 | completed |
| 7 | Maria | 8 | completed |
| 8 | Wichai | 9 | completed |
| 10 | Lisa | 10 | completed |
| 12 | Piyada | 12 | completed |

**ลูกค้าที่หายไปทั้งหมด**: Emily Chen (id 5, มีแต่ออเดอร์ที่ถูกยกเลิก), David Miller (id 11, มีแต่ออเดอร์ที่ pending) และที่ร้ายแรงที่สุดคือ **Nattapong Srisuk (id 9) ที่ไม่เคยสั่งซื้อเลย ก็หายไปด้วย!**

**เกิดอะไรขึ้น**: ขั้นตอนการทำงานจริงของ SQL คือ (1) `LEFT JOIN` ทำงานก่อน สร้างผลลัพธ์ 16 แถวพร้อม NULL สำหรับ Nattapong (2) จากนั้น `WHERE o.status = 'completed'` จึงค่อยกรอง — แต่แถวของ Nattapong มี `o.status = NULL` และ `NULL = 'completed'` ให้ผลเป็น `UNKNOWN` (ไม่ใช่ `TRUE`) จึงถูกกรองทิ้งไปเหมือนแถวธรรมดาที่ไม่ผ่านเงื่อนไข — ผลลัพธ์สุดท้ายจึงเหมือนกับใช้ `INNER JOIN` ทุกประการ ทั้งที่ตั้งใจเขียน `LEFT JOIN`

### วิธีแก้: ย้ายเงื่อนไขเข้าไปใน ON แทน

```sql
-- ✅ ถูกต้อง: เงื่อนไขกรองอยู่ใน ON ซึ่งเป็นส่วนหนึ่งของการจับคู่ ไม่ใช่การกรองหลัง join
SELECT
    c.customer_id,
    c.first_name,
    o.order_id,
    o.status
FROM customers c
LEFT JOIN orders o
    ON o.customer_id = c.customer_id
    AND o.status = 'completed'
ORDER BY c.customer_id;
```

ผลลัพธ์ที่ถูกต้อง (16 แถว ครบทุกลูกค้า):

| customer_id | first_name | order_id | status |
|---|---|---|---|
| 1 | Somchai | 1 | completed |
| 1 | Somchai | 2 | completed |
| 1 | Somchai | 15 | completed |
| 2 | Suda | 3 | completed |
| 2 | Suda | 13 | completed |
| 3 | Anong | 4 | completed |
| 4 | John | 5 | completed |
| **5** | **Emily** | **NULL** | **NULL** |
| 6 | Kenji | 7 | completed |
| 7 | Maria | 8 | completed |
| 8 | Wichai | 9 | completed |
| **9** | **Nattapong** | **NULL** | **NULL** |
| 10 | Lisa | 10 | completed |
| **11** | **David** | **NULL** | **NULL** |
| 12 | Piyada | 12 | completed |

คราวนี้ Emily Chen, Nattapong Srisuk และ David Miller ยังคงปรากฏในรายงาน (เพราะเป็นลูกค้าจริงที่มีตัวตน) เพียงแต่ไม่มีออเดอร์ที่ตรงเงื่อนไข `completed` ให้แสดง จึงเป็น `NULL` — **ตรงตามความหมายทางธุรกิจที่ต้องการทุกประการ**

### กฎการจำง่ายๆ

| ตำแหน่งเงื่อนไข | ผลกระทบต่อ OUTER JOIN |
|---|---|
| เงื่อนไขอยู่ใน `ON` | เป็นส่วนหนึ่งของการ "จับคู่" — แถวฝั่งซ้าย (ของ LEFT JOIN) ยังคงอยู่ครบเสมอ ถ้าจับคู่ไม่ได้ตามเงื่อนไขก็แค่เติม NULL |
| เงื่อนไขอยู่ใน `WHERE` **และเช็คคอลัมน์จากตารางขวา** | ทำงานหลัง join เสร็จแล้ว ถ้าค่าเป็น NULL (เพราะไม่มีคู่) เงื่อนไขจะเป็น UNKNOWN และแถวนั้นถูกตัดทิ้ง — ทำให้ LEFT JOIN แปรสภาพเป็น INNER JOIN โดยไม่ตั้งใจ |
| เงื่อนไขอยู่ใน `WHERE` **และเช็คคอลัมน์จากตารางซ้าย** | ไม่มีปัญหา เพราะตารางซ้ายไม่มี NULL จาก join อยู่แล้ว (นอกจากข้อมูลเดิมจะมี NULL) |

> **ข้อยกเว้นที่ควรรู้**: ถ้าตั้งใจจะให้ `LEFT JOIN` กลายเป็น `INNER JOIN` จริงๆ (เช่น ต้องการกรองเฉพาะลูกค้าที่มีออเดอร์ completed เท่านั้น ไม่สนใจคนที่ไม่มี) การใส่เงื่อนไขใน `WHERE` ก็คือพฤติกรรมที่ถูกต้องแล้ว — ประเด็นสำคัญคือ**ต้องตั้งใจ**ไม่ใช่พลาดโดยไม่รู้ตัว ก่อนเขียน `WHERE` บนคอลัมน์จากตารางขวาของ `OUTER JOIN` ทุกครั้ง ให้ถามตัวเองว่า "ฉันต้องการตัดแถวที่ไม่มีคู่ทิ้งไปด้วยหรือไม่?"

---

## Step 218: COALESCE ร่วมกับ OUTER JOIN เพื่อแสดงค่า default แทน NULL

การแสดง `NULL` ตรงๆ ในรายงานมักไม่เป็นมิตรกับผู้ใช้งานปลายทาง (เช่น Dashboard, Excel export) ฟังก์ชัน `COALESCE(value1, value2, ...)` จะคืนค่าแรกที่ไม่ใช่ `NULL` จาก list ที่ส่งเข้าไป จึงเหมาะมากที่จะใช้แทน `NULL` ที่เกิดจาก `OUTER JOIN` ด้วยค่า default ที่มีความหมาย

### ตัวอย่าง 1: แสดงจำนวนรีวิวและคะแนนเฉลี่ยของสินค้า

```sql
SELECT
    p.product_id,
    p.product_name,
    COUNT(r.review_id) AS review_count,
    COALESCE(ROUND(AVG(r.rating), 2), 0) AS avg_rating,
    COALESCE(ROUND(AVG(r.rating), 2)::TEXT, 'ยังไม่มีรีวิว') AS avg_rating_display
FROM products p
LEFT JOIN reviews r ON r.product_id = p.product_id
GROUP BY p.product_id, p.product_name
ORDER BY p.product_id;
```

ผลลัพธ์บางส่วน:

| product_id | product_name | review_count | avg_rating | avg_rating_display |
|---|---|---|---|---|
| 1 | Laptop Pro 15 | 2 | 4.50 | 4.50 |
| 9 | Blender Max | 0 | 0 | ยังไม่มีรีวิว |
| 10 | Vacuum Cleaner Robot | 0 | 0 | ยังไม่มีรีวิว |
| 14 | History of Siam | 0 | 0 | ยังไม่มีรีวิว |
| 15 | Yoga Mat Premium | 0 | 0 | ยังไม่มีรีวิว |
| 16 | Mystery Grab Box | 0 | 0 | ยังไม่มีรีวิว |

คอลัมน์ `avg_rating_display` แสดงข้อความที่เป็นมิตรกับผู้อ่านมากกว่าตัวเลข `0` เฉยๆ ซึ่งอาจทำให้เข้าใจผิดว่าสินค้าได้คะแนนแย่ ทั้งที่จริงแล้วมันแค่ยังไม่มีใครรีวิว

### ตัวอย่าง 2: แสดงประเทศปลายทางเริ่มต้นเมื่อไม่มีออเดอร์

```sql
SELECT
    c.customer_id,
    c.first_name,
    c.last_name,
    COALESCE(o.ship_country, c.country, 'ไม่ทราบประเทศ') AS display_country,
    COALESCE(o.order_id::TEXT, 'ยังไม่เคยสั่งซื้อ') AS order_status
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.customer_id AND o.order_id = (
    SELECT MAX(o2.order_id) FROM orders o2 WHERE o2.customer_id = c.customer_id
)
ORDER BY c.customer_id;
```

ผลลัพธ์บางส่วน (แสดงเฉพาะ Nattapong ที่ไม่เคยสั่งซื้อ):

| customer_id | first_name | last_name | display_country | order_status |
|---|---|---|---|---|
| 9 | Nattapong | Srisuk | Thailand | ยังไม่เคยสั่งซื้อ |

สังเกตว่า `COALESCE(o.ship_country, c.country, 'ไม่ทราบประเทศ')` ยัง fallback ไปใช้ `c.country` (ประเทศที่ลงทะเบียนตอนสมัครสมาชิก) ได้ แม้จะไม่มีออเดอร์เลยก็ตาม — นี่คือความยืดหยุ่นของ `COALESCE` ที่รับ argument ได้หลายตัว โดยจะคืนค่าตัวแรกที่ไม่ใช่ `NULL`

### ตัวอย่าง 3: ยอดขายรวมต่อหมวดหมู่ (แสดง 0 แทน NULL)

```sql
SELECT
    cat.category_name,
    COALESCE(SUM(oi.quantity * oi.unit_price), 0) AS total_revenue
FROM categories cat
LEFT JOIN products p ON p.category_id = cat.category_id
LEFT JOIN order_items oi ON oi.product_id = p.product_id
GROUP BY cat.category_id, cat.category_name
ORDER BY total_revenue DESC;
```

ผลลัพธ์บางส่วน:

| category_name | total_revenue |
|---|---|
| Computers | 39390.00 |
| Smartphones | 21900.00 |
| ... | ... |
| **Books** | **0** |
| **Toys** | **0** |

หมวด `Books` และ `Toys` แสดงยอดขาย `0` อย่างชัดเจนแทนที่จะเป็น `NULL` ซึ่งสื่อความหมายตรงประเด็นกว่ามากสำหรับรายงานทางธุรกิจ (ยอด `0` บอกว่า "ยังไม่มียอดขาย" ในขณะที่ `NULL` อาจถูกตีความผิดว่า "ไม่มีข้อมูล/ข้อมูลผิดพลาด")

> **เทคนิคเสริม**: ฟังก์ชันที่คล้ายกันคือ `NULLIF(a, b)` ซึ่งทำตรงข้ามกัน — คืนค่า `NULL` ถ้า `a = b` มักใช้คู่กับ `COALESCE` เพื่อป้องกันการหารด้วยศูนย์ เช่น `COALESCE(total / NULLIF(count, 0), 0)`

---

## Step 219: Performance เบื้องต้นของ OUTER JOIN เทียบกับ INNER JOIN

### แนวคิดสำคัญ: OUTER JOIN ไม่ commutative เหมือน INNER JOIN

`INNER JOIN` มีคุณสมบัติทางคณิตศาสตร์ที่เรียกว่า **commutative** (สลับที่ได้) และ **associative** (จัดกลุ่มใหม่ได้) นั่นคือ `A JOIN B JOIN C` query planner สามารถจัดลำดับการ join ใหม่ได้อย่างอิสระ (เช่น join C กับ A ก่อน แล้วค่อย join กับ B) โดยผลลัพธ์จะเหมือนเดิมเสมอ — สิ่งนี้ทำให้ query planner มีอิสระในการเลือก join order ที่มีต้นทุนต่ำที่สุด

แต่ `LEFT JOIN`, `RIGHT JOIN`, `FULL OUTER JOIN` **ไม่ commutative** — ลำดับของตารางมีผลต่อผลลัพธ์โดยตรง (`A LEFT JOIN B` ไม่เท่ากับ `B LEFT JOIN A`) ดังนั้น query planner จึง**มีอิสระในการจัดลำดับ join น้อยกว่า** เมื่อมี `OUTER JOIN` ปะปนอยู่ในคิวรี โดยเฉพาะเมื่อมีหลาย `OUTER JOIN` ต่อกันเป็นชุด (เช่น `A LEFT JOIN B LEFT JOIN C`) planner จะต้องคงลำดับการประมวลผลตามที่เขียนไว้ในหลายกรณี ซึ่งอาจทำให้พลาดแผนการ query ที่มีประสิทธิภาพสูงกว่าไปได้

### ดูแผนการทำงานด้วย EXPLAIN

```sql
EXPLAIN ANALYZE
SELECT c.customer_id, c.first_name, o.order_id
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.customer_id;
```

ตัวอย่างผลลัพธ์ (ตัวเลขจริงจะแตกต่างกันไปตามขนาดข้อมูล สถิติของตาราง และ index ที่มี — ในชุดข้อมูลเล็กๆ แบบนี้ planner มักเลือก Seq Scan และ Hash Join เพราะข้อมูลน้อยเกินกว่าจะคุ้มค่ากับการใช้ index):

```
Hash Right Join  (cost=1.18..1.55 rows=16 width=44) (actual time=0.020..0.028 rows=16 loops=1)
  Hash Cond: (o.customer_id = c.customer_id)
  ->  Seq Scan on orders o  (cost=0.00..1.16 rows=16 width=12) (actual time=0.005..0.007 rows=16 loops=1)
  ->  Hash  (cost=1.12..1.12 rows=12 width=36) (actual time=0.009..0.009 rows=12 loops=1)
        Buckets: 1024  Batches: 1  Memory Usage: 9kB
        ->  Seq Scan on customers c  (cost=0.00..1.12 rows=12 width=36) (actual time=0.002..0.003 rows=12 loops=1)
Planning Time: 0.180 ms
Execution Time: 0.052 ms
```

สังเกตว่า planner เลือกใช้ **Hash Right Join** ภายใน แม้เราจะเขียนคิวรีเป็น `LEFT JOIN` — เพราะ PostgreSQL สามารถสลับทิศทางการประมวลผลภายใน (build/probe side ของ hash table) ได้ตราบใดที่ผลลัพธ์ทางตรรกะยังถูกต้อง นี่คือหลักฐานว่า `LEFT JOIN` และ `RIGHT JOIN` มีประสิทธิภาพเท่ากันในทางปฏิบัติ — planner จัดการให้เอง

### สิ่งที่ควรรู้เกี่ยวกับ performance ของ OUTER JOIN

1. **Index บนคอลัมน์ foreign key ยังสำคัญเหมือนเดิม**: ไม่ว่าจะเป็น `INNER` หรือ `OUTER JOIN` การมี index บนคอลัมน์ที่ใช้ใน `ON` (เช่น `orders.customer_id`) ช่วยให้ planner เลือก Nested Loop with Index Scan ได้เมื่อข้อมูลมีขนาดใหญ่ ซึ่งเร็วกว่า Hash Join ในบางสถานการณ์ (เช่น เมื่อฝั่งหนึ่งมีจำนวนแถวน้อยมากเทียบกับอีกฝั่ง)
2. **OUTER JOIN ต้องส่งแถว NULL-padded เพิ่มเข้าไปในผลลัพธ์**: ทำให้จำนวนแถวที่ต้องประมวลผลในขั้นตอนถัดไป (เช่น `GROUP BY`, `ORDER BY`, join ตัวถัดไป) อาจมากกว่าการใช้ `INNER JOIN` เสมอ ถ้าข้อมูลมีแถวกำพร้าเยอะ ควรพิจารณาว่าจำเป็นต้องใช้ `OUTER JOIN` จริงหรือไม่ในแต่ละจุดของคิวรี
3. **การ join OUTER หลายตัวต่อกันจำกัดอิสระของ planner**: ยิ่งมี `LEFT JOIN` ต่อกันหลายชั้น planner ยิ่งมีตัวเลือกในการจัดลำดับ join น้อยลง สำหรับคิวรีที่ join ตารางจำนวนมาก (5+ ตาราง) ควรพิจารณาโครงสร้างคิวรีให้ดี เช่น ใช้ CTE หรือ subquery แยกส่วนที่ควรเป็น `INNER JOIN` ออกจากส่วนที่ต้องเป็น `OUTER JOIN` ก่อน
4. **`WHERE` ที่ทำ `OUTER JOIN` กลายเป็น `INNER JOIN` โดยไม่ตั้งใจ (Step 217) ก็มีผลด้าน performance เช่นกัน**: บางครั้ง planner ฉลาดพอที่จะสังเกตว่า `WHERE o.status = 'completed'` บังคับให้ `o` ต้องไม่เป็น NULL จึงปรับ (optimize) `LEFT JOIN` ให้เป็น `INNER JOIN` ภายในโดยอัตโนมัติเพื่อให้ query plan มีตัวเลือกมากขึ้น (เพราะ INNER JOIN จัดลำดับใหม่ได้อิสระกว่า) — เป็นอีกเหตุผลหนึ่งที่ควรเข้าใจ semantic ของ `OUTER JOIN` ให้แม่นยำ ไม่ใช่แค่เพื่อความถูกต้อง แต่เพื่อประสิทธิภาพด้วย

> **สรุปสั้นๆ**: ไม่ต้องกังวลว่า `OUTER JOIN` จะ "ช้ากว่า" `INNER JOIN` เสมอไป — ในหลายกรณี PostgreSQL query planner จัดการได้อย่างมีประสิทธิภาพไม่ต่างกันมาก สิ่งที่ควรให้ความสำคัญมากกว่าคือ **เลือกประเภท join ให้ตรงกับความหมายทางธุรกิจ** และ **มี index รองรับคอลัมน์ join เสมอ** ส่วนเรื่อง query planner และการอ่าน `EXPLAIN` อย่างลึกซึ้งจะกลับมาอธิบายอย่างละเอียดอีกครั้งในบทที่ว่าด้วย Query Optimization โดยเฉพาะในระดับ Advanced ของหลักสูตรนี้

---

## Step 220: แบบฝึกหัดรวม — รายงานที่ต้องใช้ OUTER JOIN จริงในระบบ e-commerce

มาลองประกอบร่างทุกสิ่งที่เรียนมาในบทนี้ เพื่อสร้างรายงานเชิงธุรกิจ 3 แบบที่พบบ่อยที่สุดในระบบ e-commerce จริง

### รายงานที่ 1: ลูกค้าที่ไม่เคยสั่งซื้อ (Never-Purchased Customers)

ใช้สำหรับแคมเปญกระตุ้นยอดขาย เช่น ส่งคูปองส่วนลดสำหรับการสั่งซื้อครั้งแรก

```sql
SELECT
    c.customer_id,
    c.first_name || ' ' || c.last_name AS customer_name,
    c.email,
    c.country,
    c.signup_date,
    CURRENT_DATE - c.signup_date AS days_since_signup
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.customer_id
WHERE o.order_id IS NULL
ORDER BY c.signup_date;
```

ผลลัพธ์:

| customer_id | customer_name | email | country | signup_date | days_since_signup |
|---|---|---|---|---|---|
| 9 | Nattapong Srisuk | nattapong.s@example.com | Thailand | 2024-08-01 | (ขึ้นกับวันปัจจุบัน) |

### รายงานที่ 2: สินค้าที่ไม่เคยถูกรีวิว (Never-Reviewed Products)

ใช้สำหรับทีมการตลาดพิจารณาว่าสินค้าตัวไหนควรกระตุ้นให้ลูกค้าเขียนรีวิวเพิ่ม เพื่อเพิ่มความน่าเชื่อถือ

```sql
SELECT
    p.product_id,
    p.product_name,
    cat.category_name,
    p.unit_price,
    p.stock_quantity,
    p.is_active
FROM products p
LEFT JOIN categories cat ON cat.category_id = p.category_id
LEFT JOIN reviews r ON r.product_id = p.product_id
WHERE r.review_id IS NULL
ORDER BY p.is_active DESC, p.product_id;
```

ผลลัพธ์:

| product_id | product_name | category_name | unit_price | stock_quantity | is_active |
|---|---|---|---|---|---|
| 10 | Vacuum Cleaner Robot | Home Appliances | 8900.00 | 12 | true |
| 14 | History of Siam | Non-Fiction | 520.00 | 18 | true |
| 15 | Yoga Mat Premium | Sports | 890.00 | 70 | true |
| 16 | Mystery Grab Box | **NULL** | 199.00 | 500 | true |
| 9 | Blender Max | Kitchen | 1190.00 | 0 | false |

สังเกตว่า `Mystery Grab Box` (id 16) ใช้ `LEFT JOIN` สองชั้นในคิวรีเดียว — ทั้งไม่มีหมวดหมู่ (`category_name = NULL`) และไม่มีรีวิว แต่ยังคงปรากฏในรายงานได้อย่างสมบูรณ์เพราะทุก join เป็น `LEFT JOIN` ทั้งหมด

### รายงานที่ 3: หมวดหมู่ที่ไม่มีสินค้าเลย (Empty Categories)

ใช้สำหรับทีมจัดหมวดหมู่พิจารณาว่าควรลบหมวดหมู่ที่ไม่ได้ใช้งาน หรือเพิ่มสินค้าเข้าไปในหมวดที่ยังว่างอยู่

```sql
SELECT
    c.category_id,
    c.category_name,
    parent.category_name AS parent_category,
    COUNT(p.product_id) AS product_count
FROM categories c
LEFT JOIN categories parent ON parent.category_id = c.parent_category_id
LEFT JOIN products p ON p.category_id = c.category_id
GROUP BY c.category_id, c.category_name, parent.category_name
HAVING COUNT(p.product_id) = 0
ORDER BY c.category_id;
```

ผลลัพธ์:

| category_id | category_name | parent_category | product_count |
|---|---|---|---|
| 6 | Books | **NULL** | 0 |
| 9 | Toys | **NULL** | 0 |

รายงานนี้ยังสาธิตการทำ **self-join แบบ LEFT JOIN** (join ตาราง `categories` กับตัวมันเองเพื่อดึงชื่อหมวดหมู่แม่) ซึ่งเป็นเทคนิคที่จะเรียนอย่างละเอียดใน Part 023 — ที่นี่ใช้ `LEFT JOIN` เพราะหมวดหมู่ระดับบนสุด (เช่น Electronics, Home Appliances) ไม่มี parent เลย (`parent_category_id IS NULL`)

### สรุปทั้ง 3 รายงาน: มุมมองที่ INNER JOIN ทำไม่ได้

ทั้งสามรายงานข้างต้นมีจุดร่วมเดียวกัน: **สิ่งที่เราต้องการค้นหาคือ "การไม่มีอยู่" ของความสัมพันธ์** (absence of a relationship) ซึ่งเป็นสิ่งที่ `INNER JOIN` ไม่สามารถแสดงให้เห็นได้เลยไม่ว่าในกรณีใด เพราะธรรมชาติของ `INNER JOIN` คือแสดงเฉพาะสิ่งที่ "มี" คู่กันเท่านั้น การเข้าใจ `OUTER JOIN` อย่างลึกซึ้งจึงเป็นทักษะที่แยกระหว่างการเขียน SQL แบบพื้นฐานกับการเขียนรายงานเชิงธุรกิจที่ตอบโจทย์การตัดสินใจได้จริง

---

## สรุปท้ายบท

- `LEFT JOIN` (หรือ `LEFT OUTER JOIN`) เก็บทุกแถวจากตารางฝั่งซ้ายเสมอ เติม `NULL` ให้คอลัมน์ฝั่งขวาเมื่อไม่มีคู่ — เป็น join type ที่ใช้บ่อยที่สุดในงานจริง
- Pattern `LEFT JOIN ... WHERE <คอลัมน์จากตารางขวา> IS NULL` คือ anti-join ใช้หาแถวที่ "ไม่มีคู่" เช่น ลูกค้าที่ไม่เคยสั่งซื้อ สินค้าที่ไม่เคยถูกรีวิว
- `RIGHT JOIN` ทำงานตรงข้ามกับ `LEFT JOIN` แต่แนะนำให้เขียนเป็น `LEFT JOIN` แทนเสมอ (สลับลำดับตาราง) เพื่อความสม่ำเสมอของโค้ดในทีม
- `FULL OUTER JOIN` รวมทั้ง `LEFT` และ `RIGHT` เข้าด้วยกัน เก็บทุกแถวจากทั้งสองตาราง เหมาะกับงาน reconciliation และ data quality audit
- สามารถผสม `INNER JOIN` และ `OUTER JOIN` ในคิวรีเดียวกันได้ตามความหมายทางธุรกิจของแต่ละความสัมพันธ์
- **ข้อผิดพลาดที่พบบ่อยที่สุด**: การใส่เงื่อนไขกรองบนคอลัมน์จากตารางขวาไว้ใน `WHERE` แทนที่จะเป็น `ON` ทำให้ `OUTER JOIN` กลายเป็น `INNER JOIN` โดยไม่ตั้งใจ — ต้องตรวจสอบทุกครั้งก่อนใช้งาน
- `COALESCE()` ช่วยแปลง `NULL` ที่เกิดจาก `OUTER JOIN` ให้เป็นค่า default ที่มีความหมายและเป็นมิตรกับผู้อ่านรายงานมากขึ้น
- `OUTER JOIN` ไม่ commutative เหมือน `INNER JOIN` ทำให้ query planner มีอิสระในการจัดลำดับ join น้อยกว่า แต่ PostgreSQL ยังคงจัดการได้อย่างมีประสิทธิภาพผ่าน Hash Join และ index ที่เหมาะสม
- OUTER JOIN คือเครื่องมือหลักในการเขียนรายงานที่ต้องการค้นหา "สิ่งที่ไม่มีอยู่" ซึ่งเป็นคำถามเชิงธุรกิจที่พบบ่อยมาก และ `INNER JOIN` ไม่สามารถตอบได้เลย

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1
เขียนคิวรีแสดงซัพพลายเออร์ทุกราย พร้อมชื่อสินค้าที่จัดหา (ถ้าซัพพลายเออร์รายใดไม่เคยจัดหาสินค้าเลย ให้แสดง NULL)

<details>
<summary>เฉลย</summary>

```sql
SELECT
    s.supplier_id,
    s.supplier_name,
    s.country,
    p.product_name
FROM suppliers s
LEFT JOIN products p ON p.supplier_id = s.supplier_id
ORDER BY s.supplier_id, p.product_id;
```

`FitLife Supplies` (supplier_id 7) จะปรากฏหนึ่งแถวโดยมี `product_name` เป็น `NULL` เพราะไม่มีสินค้าใดอ้างอิงถึงเลย
</details>

### แบบฝึกหัดที่ 2
เขียนคิวรีหาซัพพลายเออร์ที่ไม่เคยจัดหาสินค้าเลยแม้แต่ชิ้นเดียว (ใช้ anti-join pattern)

<details>
<summary>เฉลย</summary>

```sql
SELECT s.supplier_id, s.supplier_name, s.country
FROM suppliers s
LEFT JOIN products p ON p.supplier_id = s.supplier_id
WHERE p.product_id IS NULL;
```

ผลลัพธ์: `FitLife Supplies` (supplier_id 7) เท่านั้น
</details>

### แบบฝึกหัดที่ 3
เขียนคิวรีแสดงพนักงานทุกคน พร้อมจำนวนออเดอร์ที่ดูแล (ใช้ COUNT ร่วมกับ LEFT JOIN ให้พนักงานที่ไม่เคยดูแลออเดอร์แสดงเป็น 0 ไม่ใช่ NULL)

<details>
<summary>เฉลย</summary>

```sql
SELECT
    e.employee_id,
    e.first_name || ' ' || e.last_name AS employee_name,
    e.department,
    COUNT(o.order_id) AS total_orders_handled
FROM employees e
LEFT JOIN orders o ON o.employee_id = e.employee_id
GROUP BY e.employee_id, e.first_name, e.last_name, e.department
ORDER BY total_orders_handled DESC, e.employee_id;
```

พนักงาน Ratana, Somsak และ Napat (employee_id 4, 5, 6) จะได้ `total_orders_handled = 0` เพราะ `COUNT(o.order_id)` นับเฉพาะค่าที่ไม่เป็น NULL — ถ้าใช้ `COUNT(*)` แทนจะได้ผลผิดเป็น 1 สำหรับทุกคน
</details>

### แบบฝึกหัดที่ 4
เขียนคิวรีเดียวกับแบบฝึกหัดที่ 3 แต่เปลี่ยนมาใช้ `RIGHT JOIN` แทน โดยให้ได้ผลลัพธ์เหมือนกันทุกประการ

<details>
<summary>เฉลย</summary>

```sql
SELECT
    e.employee_id,
    e.first_name || ' ' || e.last_name AS employee_name,
    e.department,
    COUNT(o.order_id) AS total_orders_handled
FROM orders o
RIGHT JOIN employees e ON o.employee_id = e.employee_id
GROUP BY e.employee_id, e.first_name, e.last_name, e.department
ORDER BY total_orders_handled DESC, e.employee_id;
```

หัวใจสำคัญคือต้องสลับให้ `employees` (ตารางที่ต้องการเก็บทุกแถว) อยู่ฝั่งขวาของ `RIGHT JOIN` — ผลลัพธ์จะเหมือนกับแบบฝึกหัดที่ 3 ทุกประการ แสดงให้เห็นว่า `RIGHT JOIN` และ `LEFT JOIN` ที่สลับตารางกันให้ผลลัพธ์เดียวกันเสมอ
</details>

### แบบฝึกหัดที่ 5
เขียนคิวรีที่ใช้ `FULL OUTER JOIN` ระหว่าง `categories` และ `products` เพื่อหา **เฉพาะ** แถวที่ไม่มีคู่ในฝั่งใดฝั่งหนึ่ง (หมวดหมู่ที่ไม่มีสินค้า หรือ สินค้าที่ไม่มีหมวดหมู่)

<details>
<summary>เฉลย</summary>

```sql
SELECT
    c.category_id,
    c.category_name,
    p.product_id,
    p.product_name
FROM categories c
FULL OUTER JOIN products p ON p.category_id = c.category_id
WHERE c.category_id IS NULL OR p.product_id IS NULL
ORDER BY c.category_id NULLS LAST, p.product_id;
```

ผลลัพธ์จะได้ 3 แถว: หมวด `Books` (ไม่มีสินค้าตรงๆ), หมวด `Toys` (ไม่มีสินค้าเลย), และสินค้า `Mystery Grab Box` (ไม่มีหมวดหมู่) — สังเกตว่าเงื่อนไข `WHERE` ในที่นี้ปลอดภัย เพราะเราเช็ค `IS NULL` ไม่ใช่เช็คค่าจริง จึงไม่ทำให้ FULL OUTER JOIN แปรสภาพเป็น INNER JOIN
</details>

### แบบฝึกหัดที่ 6
ต่อไปนี้คือคิวรีที่มีบั๊ก ให้หาจุดผิดและแก้ไข: โจทย์คือ "แสดงลูกค้าทุกคน พร้อมยอดชำระเงินรวมของออเดอร์ที่จ่ายด้วย `credit_card` เท่านั้น (ถ้าไม่มีให้เป็น 0)"

```sql
SELECT
    c.customer_id,
    c.first_name,
    COALESCE(SUM(pay.amount), 0) AS total_credit_card_paid
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.customer_id
LEFT JOIN payments pay ON pay.order_id = o.order_id
WHERE pay.payment_method = 'credit_card'
GROUP BY c.customer_id, c.first_name
ORDER BY c.customer_id;
```

<details>
<summary>เฉลย</summary>

**จุดผิด**: `WHERE pay.payment_method = 'credit_card'` ทำงานหลัง `LEFT JOIN` เสร็จแล้ว ลูกค้าที่ไม่มี payment เลย (เช่น Nattapong) จะมี `pay.payment_method = NULL` ซึ่งไม่เท่ากับ `'credit_card'` และถูกกรองทิ้งไปทั้งแถว ทำให้ `LEFT JOIN` กลายเป็น `INNER JOIN` โดยไม่ตั้งใจ — ลูกค้าที่ไม่เคยจ่ายด้วยบัตรเครดิตเลยจะหายไปจากรายงานทั้งหมด แทนที่จะแสดงเป็น 0

**คิวรีที่แก้ไขแล้ว**:

```sql
SELECT
    c.customer_id,
    c.first_name,
    COALESCE(SUM(pay.amount), 0) AS total_credit_card_paid
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.customer_id
LEFT JOIN payments pay
    ON pay.order_id = o.order_id
    AND pay.payment_method = 'credit_card'
GROUP BY c.customer_id, c.first_name
ORDER BY c.customer_id;
```

ย้ายเงื่อนไข `payment_method = 'credit_card'` เข้าไปเป็นส่วนหนึ่งของ `ON` ในการ join ตาราง `payments` ทำให้ลูกค้าทุกคนยังคงปรากฏในผลลัพธ์ครบถ้วน โดยผู้ที่ไม่เคยจ่ายด้วยบัตรเครดิตจะได้ `total_credit_card_paid = 0`
</details>

### แบบฝึกหัดที่ 7
เขียนคิวรีแสดงออเดอร์ทั้งหมดที่ยังไม่ได้รับการชำระเงิน (ไม่มีแถวใน `payments` เลย) พร้อมชื่อลูกค้าและสถานะออเดอร์

<details>
<summary>เฉลย</summary>

```sql
SELECT
    o.order_id,
    c.first_name || ' ' || c.last_name AS customer_name,
    o.order_date,
    o.status
FROM orders o
JOIN customers c ON c.customer_id = o.customer_id
LEFT JOIN payments pay ON pay.order_id = o.order_id
WHERE pay.payment_id IS NULL
ORDER BY o.order_id;
```

ผลลัพธ์จะได้ 3 ออเดอร์: order_id 6 (cancelled), 11 (pending), 16 (pending) — ทั้งหมดสมเหตุสมผลเพราะยังไม่มีการจ่ายเงินเกิดขึ้นจริง ในที่นี้ `customers` ใช้ `INNER JOIN` เพราะทุกออเดอร์ต้องมีลูกค้าเสมอ (ความสัมพันธ์บังคับ) ส่วน `payments` ใช้ `LEFT JOIN` เพราะอาจมีหรือไม่มีก็ได้
</details>

### แบบฝึกหัดที่ 8
เขียนคิวรีแสดงหมวดหมู่สินค้าทุกหมวด พร้อมจำนวนสินค้าทั้งหมด และยอดขายรวม (quantity × unit_price จาก order_items) โดยหมวดที่ไม่มีสินค้าหรือไม่มียอดขายให้แสดง 0

<details>
<summary>เฉลย</summary>

```sql
SELECT
    cat.category_id,
    cat.category_name,
    COUNT(DISTINCT p.product_id) AS product_count,
    COALESCE(SUM(oi.quantity * oi.unit_price), 0) AS total_revenue
FROM categories cat
LEFT JOIN products p ON p.category_id = cat.category_id
LEFT JOIN order_items oi ON oi.product_id = p.product_id
GROUP BY cat.category_id, cat.category_name
ORDER BY total_revenue DESC, cat.category_id;
```

ใช้ `COUNT(DISTINCT p.product_id)` แทน `COUNT(p.product_id)` ธรรมดา เพื่อป้องกันการนับซ้ำ เพราะ `LEFT JOIN` กับ `order_items` อาจทำให้สินค้าชิ้นเดียวปรากฏหลายแถว (ถ้ามีหลาย order_items) การนับแบบ `COUNT(p.product_id)` ตรงๆ จะได้ตัวเลขจำนวนสินค้าที่ผิดเพี้ยนไป
</details>

### แบบฝึกหัดที่ 9
เขียนคิวรีหาลูกค้าที่เคยรีวิวสินค้า แต่**ไม่เคย**สั่งซื้อสินค้าชิ้นนั้นเลย (สมมติเหตุการณ์นี้เกิดขึ้นได้ในระบบจริง เช่น ได้รับสินค้าจากเพื่อน)

<details>
<summary>เฉลย</summary>

```sql
SELECT DISTINCT
    r.review_id,
    c.first_name || ' ' || c.last_name AS customer_name,
    p.product_name,
    r.rating
FROM reviews r
JOIN customers c ON c.customer_id = r.customer_id
JOIN products p ON p.product_id = r.product_id
LEFT JOIN order_items oi
    ON oi.product_id = r.product_id
LEFT JOIN orders o
    ON o.order_id = oi.order_id
    AND o.customer_id = r.customer_id
WHERE o.order_id IS NULL
ORDER BY r.review_id;
```

คิวรีนี้ตรวจสอบว่า "ลูกค้าคนเดียวกับที่รีวิว" เคยมีออเดอร์ที่มีสินค้าชิ้นนั้นหรือไม่ ถ้าไม่มี (`o.order_id IS NULL`) แสดงว่าเป็นการรีวิวที่ไม่ได้มาจากการซื้อในระบบ — ในชุดข้อมูลตัวอย่างของเราทุกรีวิวมาจากลูกค้าที่เคยซื้อสินค้านั้นจริง จึงคาดว่าผลลัพธ์จะว่างเปล่า (0 แถว) ซึ่งเป็นสัญญาณที่ดีว่าข้อมูลรีวิวในระบบสอดคล้องกับประวัติการซื้อ
</details>

### แบบฝึกหัดที่ 10
เขียนคิวรีสรุปภาพรวม "สุขภาพของแคตตาล็อกสินค้า" ในคิวรีเดียว แสดง: จำนวนหมวดหมู่ทั้งหมด, จำนวนหมวดหมู่ที่ไม่มีสินค้า, จำนวนสินค้าทั้งหมด, จำนวนสินค้าที่ไม่เคยถูกรีวิว, จำนวนสินค้าที่ไม่เคยถูกสั่งซื้อ

<details>
<summary>เฉลย</summary>

```sql
SELECT
    (SELECT COUNT(*) FROM categories) AS total_categories,
    (SELECT COUNT(*)
     FROM categories cat
     LEFT JOIN products p ON p.category_id = cat.category_id
     WHERE p.product_id IS NULL) AS categories_with_no_products,
    (SELECT COUNT(*) FROM products) AS total_products,
    (SELECT COUNT(*)
     FROM products p
     LEFT JOIN reviews r ON r.product_id = p.product_id
     WHERE r.review_id IS NULL) AS products_never_reviewed,
    (SELECT COUNT(*)
     FROM products p
     LEFT JOIN order_items oi ON oi.product_id = p.product_id
     WHERE oi.order_item_id IS NULL) AS products_never_ordered;
```

ผลลัพธ์ที่ควรได้จากชุดข้อมูลของบทนี้:

| total_categories | categories_with_no_products | total_products | products_never_reviewed | products_never_ordered |
|---|---|---|---|---|
| 10 | 2 | 16 | 5 | 3 |

รูปแบบนี้ (ใช้ scalar subquery หลายตัวใน `SELECT` list แทนการ join ทั้งหมดในคิวรีเดียว) เหมาะกับการสรุปตัวเลขจากหลายเงื่อนไขที่ไม่เกี่ยวข้องกันโดยตรง เพราะแต่ละ subquery ทำงานอิสระกัน อ่านและดูแลรักษาง่ายกว่าการพยายามยัดทุกอย่างลงใน join เดียว
</details>

---

**บทถัดไป**: [Part 023 — CROSS JOIN และ SELF JOIN](./part-023-cross-self-join.md)
