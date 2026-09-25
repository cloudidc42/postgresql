# Common Table Expressions (WITH / CTE)

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับกลาง | Part 025

## เป้าหมายการเรียนรู้

หลังจบบทนี้ ผู้เรียนจะสามารถ:

- อธิบายได้ว่า CTE (Common Table Expression) คืออะไร และทำไมมันช่วยให้ query ที่ซับซ้อนอ่านและดูแลรักษาได้ง่ายขึ้น
- เขียน CTE ด้วย `WITH` clause แปลง nested subquery เดิมให้เป็น CTE ที่อ่านง่าย
- ใช้ CTE หลายตัวในคำสั่งเดียว และเขียน CTE ที่อ้างอิง CTE ตัวก่อนหน้าได้ (chained CTE)
- ใช้ CTE เพื่อลดการเขียน subquery ซ้ำซ้อน (reuse logic เดียวกันหลายจุดใน query เดียว)
- เข้าใจความแตกต่างระหว่าง `MATERIALIZED` กับ `NOT MATERIALIZED` CTE ใน PostgreSQL 12+ และผลกระทบต่อ performance
- ใช้ CTE ร่วมกับ `JOIN` และ aggregate function ในการวิเคราะห์ข้อมูลจริงของระบบ e-commerce
- เขียน Writable CTE (ใช้ `INSERT` / `UPDATE` / `DELETE ... RETURNING` ภายใน `WITH`) เพื่อสร้าง data pipeline สั้นๆ ในคำสั่งเดียว
- เขียน Writable CTE ขั้นสูงที่มีหลายขั้นตอนเชื่อมต่อกัน เช่น insert ตารางหนึ่งแล้วนำผลลัพธ์ไป insert ต่ออีกตาราง
- รู้ข้อจำกัดของ CTE และตัดสินใจได้ว่าเมื่อไหร่ควรใช้ temporary table แทน
- เขียนรายงานวิเคราะห์ยอดขายแบบหลายชั้น (multi-layer analytics report) ด้วย CTE สำหรับระบบ e-commerce จริง

---

## เตรียมข้อมูล

บทนี้และบทถัดๆ ไปในช่วง Part 021–039 จะใช้ฐานข้อมูลจำลองร้านค้าออนไลน์ (e-commerce) ชุดเดียวกันตลอดทั้งชุด เพื่อให้ผู้เรียนคุ้นเคยกับโครงสร้างข้อมูลและสามารถต่อยอดเทคนิคต่างๆ ได้อย่างต่อเนื่อง

### สร้างตาราง

```sql
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

### ใส่ข้อมูลตัวอย่าง

```sql
-- categories: มีทั้งหมวดหลักและหมวดย่อย (parent_category_id ชี้กลับไปยัง category_id)
INSERT INTO categories (category_name, parent_category_id) VALUES
    ('Electronics', NULL),          -- 1
    ('Computers', 1),               -- 2
    ('Smartphones', 1),             -- 3
    ('Home Appliances', NULL),      -- 4
    ('Kitchen', 4),                 -- 5
    ('Fashion', NULL),              -- 6
    ('Men''s Clothing', 6),         -- 7
    ('Women''s Clothing', 6),       -- 8
    ('Books', NULL),                -- 9
    ('Sports & Outdoors', NULL);    -- 10

-- suppliers
INSERT INTO suppliers (supplier_name, country) VALUES
    ('TechSource Co., Ltd.', 'Thailand'),        -- 1
    ('Global Gadgets Inc.', 'USA'),               -- 2
    ('Shenzhen Electronics Ltd.', 'China'),       -- 3
    ('Nordic Home AB', 'Sweden'),                 -- 4
    ('Kitchen Master Co.', 'Thailand'),           -- 5
    ('Fashion Hub Ltd.', 'Vietnam'),              -- 6
    ('Bangkok Textile Group', 'Thailand'),        -- 7
    ('BookWorld Publishing', 'UK'),               -- 8
    ('Outdoor Gear Co.', 'Thailand'),             -- 9
    ('Samsung Electronics', 'South Korea');       -- 10

-- products
INSERT INTO products (product_name, category_id, supplier_id, unit_price, stock_quantity, is_active) VALUES
    ('Laptop Pro 15"',            2, 1,  32900.00, 25,  true),  -- 1
    ('Wireless Mouse',            2, 3,    590.00, 150, true),  -- 2
    ('Mechanical Keyboard',       2, 3,   2490.00, 80,  true),  -- 3
    ('Smartphone X12',            3, 10, 24900.00, 40,  true),  -- 4
    ('Smartphone Lite',           3, 2,   8990.00, 60,  true),  -- 5
    ('Bluetooth Earbuds',         1, 2,   1990.00, 200, true),  -- 6
    ('4K Monitor 27"',            2, 1,   8900.00, 30,  true),  -- 7
    ('Air Fryer 5L',              5, 5,   2590.00, 45,  true),  -- 8
    ('Stand Mixer',               5, 4,   6990.00, 15,  true),  -- 9
    ('Robot Vacuum',              4, 4,  12900.00, 20,  true),  -- 10
    ('Men''s Denim Jacket',       7, 6,   1290.00, 70,  true),  -- 11
    ('Men''s Running Shoes',      7, 7,   2190.00, 90,  true),  -- 12
    ('Women''s Summer Dress',     8, 6,    990.00, 100, true),  -- 13
    ('Women''s Yoga Pants',       8, 7,    690.00, 120, true),  -- 14
    ('PostgreSQL Mastery Book',   9, 8,    890.00, 35,  true),  -- 15
    ('Data Engineering Handbook', 9, 8,   1290.00, 20,  true),  -- 16
    ('Camping Tent 4-person',     10,9,   4590.00, 12,  true),  -- 17
    ('Yoga Mat Premium',          10,9,    590.00, 150, true),  -- 18
    ('Old Model Tablet',          1, 2,   5990.00, 5,   false), -- 19
    ('Discontinued Smartwatch',   1, 10,  3990.00, 0,   false); -- 20

-- customers
INSERT INTO customers (first_name, last_name, email, country, signup_date) VALUES
    ('Somchai',  'Jaidee',   'somchai.j@example.com',  'Thailand',  '2024-06-01'),  -- 1
    ('Suda',     'Wongsa',   'suda.w@example.com',     'Thailand',  '2024-07-15'),  -- 2
    ('Anong',    'Phromma',  'anong.p@example.com',    'Thailand',  '2024-08-02'),  -- 3
    ('Wichai',   'Sombat',   'wichai.s@example.com',   'Thailand',  '2024-08-20'),  -- 4
    ('Nok',      'Srisai',   'nok.s@example.com',      'Thailand',  '2024-09-05'),  -- 5
    ('John',     'Smith',    'john.smith@example.com', 'USA',       '2024-09-10'),  -- 6
    ('Emma',     'Johnson',  'emma.j@example.com',     'USA',       '2024-10-01'),  -- 7
    ('Liu',      'Wei',      'liu.wei@example.com',    'China',     '2024-10-12'),  -- 8
    ('Tanaka',   'Yuki',     'tanaka.y@example.com',   'Japan',     '2024-11-01'),  -- 9
    ('Nguyen',   'Van A',    'nguyen.a@example.com',   'Vietnam',   '2024-11-18'),  -- 10
    ('Pim',      'Suwan',    'pim.s@example.com',      'Thailand',  '2024-12-01'),  -- 11
    ('Kritsada', 'Thong',    'kritsada.t@example.com', 'Thailand',  '2024-12-10'),  -- 12
    ('Sarah',    'Lee',      'sarah.lee@example.com',  'Singapore', '2025-01-02'),  -- 13
    ('Ahmad',    'Rahman',   'ahmad.r@example.com',    'Malaysia',  '2025-01-15'),  -- 14
    ('Malee',    'Boon',     'malee.b@example.com',    'Thailand',  '2025-02-01');  -- 15

-- employees (manager_id อ้างอิงตัวเอง)
INSERT INTO employees (first_name, last_name, hire_date, manager_id, department) VALUES
    ('Prasert',  'Kittipong', '2018-01-15', NULL, 'Management'), -- 1
    ('Siriwan',  'Chaiyo',    '2019-03-01', 1,    'Sales'),      -- 2
    ('Anucha',   'Petch',     '2019-06-10', 1,    'Sales'),      -- 3
    ('Nattaya',  'Ruam',      '2020-02-20', 2,    'Sales'),      -- 4
    ('Kamon',    'Srisuk',    '2020-05-15', 2,    'Sales'),      -- 5
    ('Piyada',   'Nakorn',    '2021-01-10', 3,    'Support'),    -- 6
    ('Somsak',   'Yai',       '2021-07-01', 3,    'Support'),    -- 7
    ('Waraporn', 'Sang',      '2022-01-05', 1,    'Marketing'),  -- 8
    ('Decha',    'Boonmee',   '2022-08-15', 8,    'Marketing'),  -- 9
    ('Ratchanee','Phon',      '2023-02-01', 2,    'Sales');      -- 10

-- orders
INSERT INTO orders (customer_id, employee_id, order_date, status, ship_country) VALUES
    (1,  2,  '2025-01-05 10:00+07', 'delivered',   'Thailand'),  -- 1
    (2,  2,  '2025-01-08 14:30+07', 'delivered',   'Thailand'),  -- 2
    (3,  3,  '2025-01-10 09:15+07', 'delivered',   'Thailand'),  -- 3
    (1,  2,  '2025-01-15 16:00+07', 'delivered',   'Thailand'),  -- 4
    (4,  4,  '2025-01-20 11:20+07', 'cancelled',   'Thailand'),  -- 5
    (6,  5,  '2025-02-02 08:45+07', 'delivered',   'USA'),       -- 6
    (7,  5,  '2025-02-05 13:10+07', 'delivered',   'USA'),       -- 7
    (2,  2,  '2025-02-10 10:00+07', 'shipped',     'Thailand'),  -- 8
    (5,  3,  '2025-02-14 15:30+07', 'delivered',   'Thailand'),  -- 9
    (8,  10, '2025-02-18 09:00+07', 'delivered',   'China'),     -- 10
    (9,  10, '2025-02-20 12:00+07', 'processing',  'Japan'),     -- 11
    (3,  3,  '2025-03-01 10:30+07', 'delivered',   'Thailand'),  -- 12
    (10, 5,  '2025-03-05 14:00+07', 'delivered',   'Vietnam'),   -- 13
    (1,  2,  '2025-03-08 09:45+07', 'delivered',   'Thailand'),  -- 14
    (11, 4,  '2025-03-12 11:00+07', 'delivered',   'Thailand'),  -- 15
    (12, 4,  '2025-03-15 16:20+07', 'pending',     'Thailand'),  -- 16
    (13, 10, '2025-03-20 10:10+07', 'delivered',   'Singapore'), -- 17
    (6,  5,  '2025-03-25 13:40+07', 'delivered',   'USA'),       -- 18
    (14, 10, '2025-04-01 09:30+07', 'cancelled',   'Malaysia'),  -- 19
    (15, 4,  '2025-04-05 15:00+07', 'delivered',   'Thailand');  -- 20

-- order_items
INSERT INTO order_items (order_id, product_id, quantity, unit_price) VALUES
    (1,  1,  1, 32900.00),
    (1,  2,  1,   590.00),
    (2,  4,  1, 24900.00),
    (3,  6,  2,  1990.00),
    (4,  3,  1,  2490.00),
    (5,  5,  1,  8990.00),
    (6,  7,  1,  8900.00),
    (7,  9,  1,  6990.00),
    (8,  11, 2,  1290.00),
    (9,  12, 1,  2190.00),
    (10, 4,  1, 24900.00),
    (11, 15, 2,   890.00),
    (12, 1,  1, 32900.00),
    (13, 13, 3,   990.00),
    (14, 6,  1,  1990.00),
    (15, 10, 1, 12900.00),
    (16, 17, 1,  4590.00),
    (17, 5,  1,  8990.00),
    (18, 9,  1,  6990.00),
    (19, 14, 2,   690.00),
    (20, 16, 1,  1290.00),
    (20, 15, 1,   890.00);

-- reviews
INSERT INTO reviews (product_id, customer_id, rating, review_text, review_date) VALUES
    (1,  1,  5, 'แรงมาก ทำงานลื่นสุดๆ',               '2025-01-10'),
    (1,  3,  4, 'ดีแต่ราคาสูงไปนิด',                   '2025-03-05'),
    (4,  2,  5, 'กล้องสวย จอสวย',                     '2025-01-12'),
    (4,  8,  4, 'ใช้งานดี แบตอยู่ได้ทั้งวัน',           '2025-02-20'),
    (6,  3,  3, 'เสียงกลางๆ ไม่ประทับใจมาก',           '2025-01-14'),
    (6,  1,  5, 'คุ้มราคามาก',                         '2025-03-10'),
    (8,  7,  5, 'ทอดอาหารอร่อย ทำความสะอาดง่าย',       '2025-02-08'),
    (9,  7,  4, 'แข็งแรงดี เสียงดังนิดหน่อย',           '2025-02-09'),
    (11, 2,  4, 'ใส่สบาย ทรงสวย',                      '2025-02-12'),
    (12, 5,  5, 'วิ่งสบาย ระบายอากาศดี',                '2025-02-16'),
    (13, 10, 5, 'ผ้าดี ใส่เย็นสบาย',                    '2025-03-07'),
    (15, 1,  5, 'อธิบายละเอียด เข้าใจง่าย',            '2025-01-20'),
    (15, 12, 4, 'หนังสือดี เหมาะกับมือใหม่',            '2025-03-18'),
    (10, 11, 3, 'ดูดฝุ่นได้ดีแต่เสียงดัง',              '2025-03-14'),
    (7,  6,  5, 'จอสวย สีสด คมชัด',                    '2025-02-03');

-- payments (เฉพาะออเดอร์ที่ชำระเงินแล้ว: delivered/shipped)
INSERT INTO payments (order_id, payment_date, amount, payment_method) VALUES
    (1,  '2025-01-05 10:05+07', 33490.00, 'credit_card'),
    (2,  '2025-01-08 14:35+07', 24900.00, 'promptpay'),
    (3,  '2025-01-10 09:20+07',  3980.00, 'credit_card'),
    (4,  '2025-01-15 16:05+07',  2490.00, 'bank_transfer'),
    (6,  '2025-02-02 08:50+07',  8900.00, 'credit_card'),
    (7,  '2025-02-05 13:15+07',  6990.00, 'credit_card'),
    (8,  '2025-02-10 10:05+07',  2580.00, 'promptpay'),
    (9,  '2025-02-14 15:35+07',  2190.00, 'cod'),
    (10, '2025-02-18 09:05+07', 24900.00, 'credit_card'),
    (12, '2025-03-01 10:35+07', 32900.00, 'bank_transfer'),
    (13, '2025-03-05 14:05+07',  2970.00, 'credit_card'),
    (14, '2025-03-08 09:50+07',  1990.00, 'promptpay'),
    (15, '2025-03-12 11:05+07', 12900.00, 'credit_card'),
    (17, '2025-03-20 10:15+07',  8990.00, 'credit_card'),
    (18, '2025-03-25 13:45+07',  6990.00, 'bank_transfer'),
    (20, '2025-04-05 15:05+07',  2180.00, 'promptpay');
```

> **หมายเหตุ:** ออเดอร์ที่มีสถานะ `pending`, `processing` หรือ `cancelled` (order_id 5, 11, 16, 19) จะยังไม่มีแถวใน `payments` เพราะยังไม่ได้ชำระเงินจริง ซึ่งเป็นสถานการณ์ปกติที่เจอในระบบจริง และเป็นจุดที่ CTE จะช่วยให้เราจัดการ logic แบบนี้ได้อย่างเป็นระเบียบ

---

## Step 241: CTE คืออะไร

**CTE (Common Table Expression)** คือผลลัพธ์ชั่วคราวที่เรา "ตั้งชื่อ" ไว้ล่วงหน้าด้วยคำสั่ง `WITH ... AS (...)` แล้วนำไปใช้ต่อใน query หลักราวกับว่ามันเป็นตารางจริงตารางหนึ่ง มันไม่ได้เก็บข้อมูลถาวรในดิสก์เหมือน view หรือ table — มันมีชีวิตอยู่แค่ในช่วงเวลาที่คำสั่ง SQL นั้นกำลังรันเท่านั้น

รูปแบบพื้นฐาน:

```sql
WITH cte_name AS (
    SELECT ...
)
SELECT ...
FROM cte_name
WHERE ...;
```

### ทำไมต้องใช้ CTE ถ้า subquery ก็ทำงานได้เหมือนกัน?

ลองดูตัวอย่าง: เราต้องการหา "ลูกค้าที่มียอดใช้จ่ายรวม (จาก payments) สูงกว่าค่าเฉลี่ยของลูกค้าทุกคน"

**แบบ nested subquery (แบบเดิม):**

```sql
SELECT
    c.customer_id,
    c.first_name || ' ' || c.last_name AS customer_name,
    COALESCE(SUM(p.amount), 0) AS total_spent
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.customer_id
LEFT JOIN payments p ON p.order_id = o.order_id
GROUP BY c.customer_id, c.first_name, c.last_name
HAVING COALESCE(SUM(p.amount), 0) > (
    SELECT AVG(customer_total)
    FROM (
        SELECT COALESCE(SUM(p2.amount), 0) AS customer_total
        FROM customers c2
        LEFT JOIN orders o2 ON o2.customer_id = c2.customer_id
        LEFT JOIN payments p2 ON p2.order_id = o2.order_id
        GROUP BY c2.customer_id
    ) AS sub
)
ORDER BY total_spent DESC;
```

สังเกตว่าเราต้องเขียน `LEFT JOIN orders ... LEFT JOIN payments ...` และ `GROUP BY` **ซ้ำสองครั้ง** — ครั้งแรกสำหรับผลลัพธ์หลัก ครั้งที่สองซ้อนอยู่ใน subquery เพื่อคำนวณค่าเฉลี่ย ยิ่ง query ซับซ้อนขึ้น subquery ที่ซ้อนกันแบบนี้ก็ยิ่งอ่านยากขึ้นเรื่อยๆ

**แบบ CTE (แบบใหม่ที่อ่านง่ายกว่า):**

```sql
WITH customer_totals AS (
    SELECT
        c.customer_id,
        c.first_name || ' ' || c.last_name AS customer_name,
        COALESCE(SUM(p.amount), 0) AS total_spent
    FROM customers c
    LEFT JOIN orders o ON o.customer_id = c.customer_id
    LEFT JOIN payments p ON p.order_id = o.order_id
    GROUP BY c.customer_id, c.first_name, c.last_name
)
SELECT
    customer_id,
    customer_name,
    total_spent
FROM customer_totals
WHERE total_spent > (SELECT AVG(total_spent) FROM customer_totals)
ORDER BY total_spent DESC;
```

**ผลลัพธ์ตัวอย่าง:**

```text
 customer_id | customer_name  | total_spent
-------------+----------------+-------------
           1 | Somchai Jaidee |    37970.00
           3 | Anong Phromma  |    36880.00
           2 | Suda Wongsa    |    27480.00
           8 | Liu Wei        |    24900.00
           6 | John Smith     |    15890.00
          11 | Pim Suwan      |    12900.00
(6 rows)
```

จะเห็นว่า logic การคำนวณ `total_spent` ถูกเขียนไว้ **ครั้งเดียว** ในชื่อ `customer_totals` แล้วเรานำมาใช้ได้ทั้งใน `SELECT` หลักและใน subquery ที่คำนวณค่าเฉลี่ย ข้อดีของ CTE เมื่อเทียบกับ nested subquery คือ:

1. **อ่านจากบนลงล่างเป็นขั้นตอน** เหมือนเขียนโปรแกรม — "ก่อนอื่นคำนวณอันนี้ก่อน (`customer_totals`) แล้วค่อยนำไปใช้ต่อ" แทนที่จะต้องไล่อ่านจากในสุดออกมานอกสุดแบบ subquery ที่ซ้อนกันหลายชั้น
2. **ตั้งชื่อที่สื่อความหมาย** ได้ (`customer_totals`, `paid_orders`, `top_products`) ทำให้ query อธิบายตัวเองได้ (self-documenting)
3. **ลดการเขียนโค้ดซ้ำ** เมื่อต้องใช้ผลลัพธ์เดียวกันหลายจุด (จะเห็นชัดเจนใน Step 244)
4. **debug ง่ายกว่า** เพราะสามารถรัน `SELECT * FROM customer_totals` ส่วนเดียวเพื่อตรวจสอบผลลัพธ์กลางได้ทันที โดยไม่ต้องรื้อทั้ง query

CTE ที่กล่าวถึงในบทนี้ทั้งหมดเป็น **non-recursive CTE** (CTE ที่ไม่อ้างอิงตัวเอง) ส่วน **recursive CTE** (`WITH RECURSIVE`) ซึ่งใช้สำหรับข้อมูลแบบลำดับชั้น (hierarchical data) เช่น โครงสร้างองค์กรหรือหมวดหมู่สินค้าแบบหลายชั้น จะอยู่ใน [Part 026: Recursive CTE](./part-026-recursive-cte.md)

---

## Step 242: CTE พื้นฐาน — Syntax และการแปลง Subquery

### Syntax เต็มรูปแบบ

```sql
WITH cte_name [(column_name [, ...])] AS [[NOT] MATERIALIZED] (
    subquery
)
SELECT ...;
```

- `cte_name` — ชื่อที่เราตั้งให้ ใช้อ้างอิงในส่วนที่เหลือของคำสั่ง
- `(column_name [, ...])` — ไม่บังคับ ใช้ตั้งชื่อคอลัมน์ใหม่ให้ CTE ถ้าไม่ระบุจะใช้ชื่อคอลัมน์จาก `SELECT` ภายใน
- `[NOT] MATERIALIZED` — ตัวเลือกใหม่ตั้งแต่ PostgreSQL 12 ซึ่งจะอธิบายละเอียดใน Step 245
- `subquery` — คำสั่ง `SELECT` (หรือ `INSERT`/`UPDATE`/`DELETE ... RETURNING` สำหรับ writable CTE ใน Step 247-248)

### ตัวอย่างที่ 1: ระบุชื่อคอลัมน์ใหม่ให้ CTE

```sql
WITH cat_stats (cat_name, product_count, avg_price) AS (
    SELECT
        c.category_name,
        COUNT(p.product_id),
        ROUND(AVG(p.unit_price), 2)
    FROM categories c
    JOIN products p ON p.category_id = c.category_id
    GROUP BY c.category_name
)
SELECT cat_name, product_count, avg_price
FROM cat_stats
ORDER BY avg_price DESC;
```

**ผลลัพธ์ตัวอย่าง:**

```text
     cat_name      | product_count | avg_price
--------------------+---------------+-----------
 Computers          |             4 |  11220.00
 Smartphones        |             2 |  16945.00
 Home Appliances    |             1 |  12900.00
 Sports & Outdoors  |             2 |   2590.00
 Kitchen            |             2 |   4790.00
 Electronics        |             3 |   3990.00
 Men's Clothing     |             2 |   1740.00
 Books              |             2 |   1090.00
 Women's Clothing   |             2 |    840.00
(9 rows)
```

### ตัวอย่างที่ 2: แปลง subquery เดิมให้เป็น CTE

โจทย์: หาสินค้าที่ราคาสูงกว่าค่าเฉลี่ยราคาสินค้า **ในหมวดหมู่เดียวกัน** (correlated subquery แบบเดิมมักอ่านยาก)

**แบบ correlated subquery เดิม:**

```sql
SELECT p.product_name, p.category_id, p.unit_price
FROM products p
WHERE p.unit_price > (
    SELECT AVG(p2.unit_price)
    FROM products p2
    WHERE p2.category_id = p.category_id
)
ORDER BY p.category_id, p.unit_price DESC;
```

**แบบ CTE (คำนวณค่าเฉลี่ยต่อหมวดหมู่ครั้งเดียว แล้ว JOIN กลับ):**

```sql
WITH category_avg AS (
    SELECT category_id, AVG(unit_price) AS avg_price
    FROM products
    GROUP BY category_id
)
SELECT
    p.product_name,
    p.category_id,
    p.unit_price,
    ca.avg_price
FROM products p
JOIN category_avg ca ON ca.category_id = p.category_id
WHERE p.unit_price > ca.avg_price
ORDER BY p.category_id, p.unit_price DESC;
```

**ผลลัพธ์ตัวอย่าง:**

```text
      product_name       | category_id | unit_price | avg_price
--------------------------+-------------+------------+-----------
 Discontinued Smartwatch  |           1 |    3990.00 |  3990.0000000000000000
 Laptop Pro 15"           |           2 |   32900.00 | 11220.0000000000000000
 Smartphone X12           |           3 |   24900.00 | 16945.0000000000000000
 Stand Mixer              |           5 |    6990.00 |  4790.0000000000000000
 Men's Running Shoes      |           7 |    2190.00 |  1740.0000000000000000
 Women's Summer Dress     |           8 |     990.00 |   840.0000000000000000
 Data Engineering Handbook|           9 |    1290.00 |  1090.0000000000000000
 Camping Tent 4-person    |          10 |    4590.00 |  2590.0000000000000000
(8 rows)
```

ทั้งสองแบบให้ผลลัพธ์เหมือนกัน แต่แบบ CTE คำนวณค่าเฉลี่ยเพียง **ครั้งเดียวต่อหมวดหมู่** (จำนวนหมวดหมู่) แล้วนำมา `JOIN` แทนที่จะคำนวณซ้ำสำหรับสินค้าทุกแถว (correlated subquery จะถูกรันซ้ำต่อแถวของ outer query ซึ่งอาจช้ากว่าเมื่อข้อมูลมีปริมาณมาก) แม้ optimizer ของ PostgreSQL จะฉลาดพอที่จะปรับ query ทั้งสองแบบให้เร็วใกล้เคียงกันในหลายกรณี แต่ความอ่านง่ายของ CTE ยังคงเป็นข้อได้เปรียบสำคัญ

---

## Step 243: หลาย CTE ในคำสั่งเดียว และ CTE ที่อ้างอิง CTE ก่อนหน้า

เราสามารถประกาศ CTE **หลายตัว** ในคำสั่งเดียวได้ โดยคั่นด้วยเครื่องหมายจุลภาค (comma) และ CTE ที่ประกาศทีหลังสามารถอ้างอิง CTE ที่ประกาศไว้ก่อนหน้าได้ (แต่ CTE ก่อนหน้าจะอ้างอิง CTE ที่ยังไม่ถูกประกาศไม่ได้ — ต้องเรียงตามลำดับจากบนลงล่าง ยกเว้นกรณี `WITH RECURSIVE` ที่ CTE อ้างอิงตัวเองได้ ซึ่งจะเรียนใน Part 026)

```sql
WITH first_cte AS (
    SELECT ...
),
second_cte AS (
    SELECT ... FROM first_cte ...   -- อ้างอิง first_cte ได้
),
third_cte AS (
    SELECT ... FROM second_cte ...  -- อ้างอิง second_cte (และ first_cte) ได้
)
SELECT ... FROM third_cte;
```

### ตัวอย่าง: chained CTE 3 ชั้นสำหรับวิเคราะห์รายได้ลูกค้า

โจทย์: หารายได้รวมต่อลูกค้า จากออเดอร์ที่ **ชำระเงินแล้วเท่านั้น** แล้วจัดอันดับว่าลูกค้าคนไหนอยู่ในกลุ่ม "top spender" (รายได้มากกว่า 20,000 บาท)

```sql
WITH paid_orders AS (
    -- ชั้นที่ 1: คัดเฉพาะออเดอร์ที่มีการชำระเงินแล้ว
    SELECT DISTINCT o.order_id, o.customer_id
    FROM orders o
    JOIN payments p ON p.order_id = o.order_id
),
order_revenue AS (
    -- ชั้นที่ 2: คำนวณยอดขายต่อออเดอร์ (อ้างอิง paid_orders)
    SELECT
        po.order_id,
        po.customer_id,
        SUM(oi.quantity * oi.unit_price) AS order_total
    FROM paid_orders po
    JOIN order_items oi ON oi.order_id = po.order_id
    GROUP BY po.order_id, po.customer_id
),
customer_revenue AS (
    -- ชั้นที่ 3: รวมยอดขายต่อลูกค้า (อ้างอิง order_revenue)
    SELECT
        customer_id,
        SUM(order_total) AS total_revenue,
        COUNT(*) AS paid_order_count
    FROM order_revenue
    GROUP BY customer_id
)
SELECT
    c.first_name || ' ' || c.last_name AS customer_name,
    cr.paid_order_count,
    cr.total_revenue,
    CASE WHEN cr.total_revenue > 20000 THEN 'Top Spender' ELSE 'Regular' END AS segment
FROM customer_revenue cr
JOIN customers c ON c.customer_id = cr.customer_id
ORDER BY cr.total_revenue DESC;
```

**ผลลัพธ์ตัวอย่าง:**

```text
 customer_name  | paid_order_count | total_revenue |   segment
-----------------+-------------------+----------------+-------------
 Somchai Jaidee  |                 3 |       37970.00 | Top Spender
 Anong Phromma   |                 2 |       36880.00 | Top Spender
 Suda Wongsa     |                 2 |       27480.00 | Top Spender
 Liu Wei         |                 1 |       24900.00 | Top Spender
 John Smith      |                 2 |       15890.00 | Regular
 Pim Suwan       |                 1 |       12900.00 | Regular
 Sarah Lee       |                 1 |        8990.00 | Regular
 Emma Johnson    |                 1 |        6990.00 | Regular
 Nguyen Van A    |                 1 |        2970.00 | Regular
 Nok Srisai      |                 1 |        2190.00 | Regular
 Malee Boon      |                 1 |        2180.00 | Regular
(11 rows)
```

จุดสำคัญของตัวอย่างนี้คือแต่ละ CTE ทำหน้าที่เพียง "ขั้นตอนเดียว" ของ pipeline: กรองข้อมูล → รวมต่อออเดอร์ → รวมต่อลูกค้า เมื่ออ่านจากบนลงล่างเราจะเข้าใจ logic ทั้งหมดได้เป็นลำดับขั้น ซึ่งต่างจากการพยายามยัดทุกอย่างไว้ใน query เดียวที่มี subquery ซ้อนกันหลายชั้น

> **ข้อควรระวัง:** CTE ที่ประกาศไว้แต่ไม่ได้ถูกเรียกใช้ใน query หลักเลย จะถือว่าเป็นส่วนเกิน PostgreSQL อาจจะยังคงรันมันอยู่ (ขึ้นกับว่าเป็น MATERIALIZED หรือไม่ ดู Step 245) ดังนั้นควรลบ CTE ที่ไม่ได้ใช้งานออกเพื่อความสะอาดของโค้ดและประสิทธิภาพ

---

## Step 244: CTE ที่ใช้ซ้ำหลายครั้งในคำสั่งเดียวกัน

ข้อดีอีกอย่างของ CTE คือเราสามารถ **อ้างอิงชื่อเดียวกันได้หลายครั้ง** ใน query หลัก โดยไม่ต้องเขียน subquery ซ้ำ ซึ่งช่วยลดโอกาสพิมพ์ logic ผิดพลาดไม่ตรงกันระหว่างจุดที่ใช้งาน

### ตัวอย่าง: หาสัดส่วนรายได้ของแต่ละหมวดหมู่เทียบกับรายได้รวมทั้งหมด

โจทย์นี้ต้องใช้ผลลัพธ์ "รายได้ต่อหมวดหมู่" สองครั้ง — ครั้งแรกเพื่อแสดงรายได้ของแต่ละหมวด ครั้งที่สองเพื่อคำนวณผลรวมทั้งหมด (grand total) สำหรับหารหาเปอร์เซ็นต์

```sql
WITH category_revenue AS (
    SELECT
        cat.category_id,
        cat.category_name,
        SUM(oi.quantity * oi.unit_price) AS revenue
    FROM order_items oi
    JOIN orders o     ON o.order_id = oi.order_id
    JOIN payments pay ON pay.order_id = o.order_id
    JOIN products p   ON p.product_id = oi.product_id
    JOIN categories cat ON cat.category_id = p.category_id
    GROUP BY cat.category_id, cat.category_name
)
SELECT
    cr.category_name,
    cr.revenue,
    -- ใช้ category_revenue ซ้ำเป็นครั้งที่สองเพื่อหาผลรวมทั้งหมด
    ROUND(cr.revenue * 100.0 / (SELECT SUM(revenue) FROM category_revenue), 2) AS pct_of_total
FROM category_revenue cr
ORDER BY cr.revenue DESC;
```

**ผลลัพธ์ตัวอย่าง:**

```text
    category_name    | revenue  | pct_of_total
-----------------------+----------+--------------
 Computers             | 77780.00 |        40.33
 Smartphones            | 58790.00 |        30.48
 Home Appliances        | 12900.00 |         6.69
 Kitchen                | 13980.00 |         7.25
 Electronics            |  5970.00 |         3.10
 Men's Clothing         |  4770.00 |         2.47
 Women's Clothing       |  2970.00 |         1.54
 Books                  |  2180.00 |         1.13
(8 rows)
```

โดยไม่ใช้ CTE เราจะต้องเขียน `JOIN order_items ... JOIN orders ... JOIN payments ... JOIN products ... JOIN categories ... GROUP BY` ซ้ำสองครั้งทั้งใน `SELECT` หลักและใน subquery ของ `SUM(revenue)` ซึ่งนอกจากจะยาวและอ่านยากแล้ว ยังเสี่ยงที่จะแก้ logic จุดหนึ่งแล้วลืมแก้อีกจุดหนึ่งให้ตรงกัน (เช่น ถ้าวันหนึ่งต้องเพิ่มเงื่อนไข `WHERE o.status = 'delivered'` แล้วลืมใส่ในอีกจุด ผลลัพธ์จะผิดทันที) การรวม logic ไว้ใน CTE เดียวแล้วอ้างอิงซ้ำจึงช่วยลดความเสี่ยงนี้ได้มาก

### ตัวอย่างที่สอง: ใช้ CTE ร่วมกับ self-join

```sql
WITH employee_sales AS (
    SELECT
        o.employee_id,
        SUM(pay.amount) AS total_sales
    FROM orders o
    JOIN payments pay ON pay.order_id = o.order_id
    GROUP BY o.employee_id
)
SELECT
    e.first_name || ' ' || e.last_name AS employee_name,
    es.total_sales,
    -- เทียบยอดขายของแต่ละคนกับยอดขายเฉลี่ยของทีม (อ้างอิง employee_sales ซ้ำ)
    ROUND(es.total_sales - (SELECT AVG(total_sales) FROM employee_sales), 2) AS diff_from_avg
FROM employee_sales es
JOIN employees e ON e.employee_id = es.employee_id
ORDER BY es.total_sales DESC;
```

**ผลลัพธ์ตัวอย่าง:**

```text
  employee_name  | total_sales | diff_from_avg
------------------+-------------+----------------
 Siriwan Chaiyo   |    65450.00 |       28206.00
 Anucha Petch     |    39070.00 |        1826.00
 Ratchanee Phon   |    33890.00 |       -3354.00
 Kamon Srisuk     |    25850.00 |      -11394.00
 Nattaya Ruam     |    15080.00 |      -22164.00
(5 rows)
```

---

## Step 245: Materialized vs Not Materialized CTE (PostgreSQL 12+)

นี่เป็นหัวข้อสำคัญที่มักสร้างความเข้าใจผิดให้ผู้ที่เคยใช้ PostgreSQL เวอร์ชันเก่า

### พฤติกรรมก่อน PostgreSQL 12: "Optimization Fence"

ก่อนเวอร์ชัน 12 ทุก CTE ใน PostgreSQL จะถูก **materialize เสมอ** หมายความว่า PostgreSQL จะรันคำสั่งภายใน CTE ให้เสร็จสมบูรณ์ก่อน แล้วเก็บผลลัพธ์ไว้ใน temporary storage ก่อนนำไปใช้ต่อใน query หลัก พฤติกรรมนี้เรียกว่า **"optimization fence"** (รั้วกั้นการ optimize) — planner จะไม่สามารถ "มองทะลุ" เข้าไปปรับแผนการทำงานร่วมกันระหว่าง CTE กับส่วนที่เหลือของ query ได้ เช่น ไม่สามารถ push เงื่อนไข `WHERE` จากภายนอกเข้าไปกรองข้อมูลตั้งแต่ในตัว CTE เพื่อลดจำนวนแถวที่ต้องประมวลผลตั้งแต่ต้น

ผลคือในบางกรณี CTE แบบเก่าจะ **ช้ากว่า** การเขียนเป็น subquery หรือ view ธรรมดา เพราะ optimizer ไม่สามารถปรับแผนให้เหมาะสมที่สุดได้

### พฤติกรรมตั้งแต่ PostgreSQL 12 เป็นต้นไป

ตั้งแต่ PostgreSQL 12 เป็นต้นมา ผู้เรียนสามารถควบคุมพฤติกรรมนี้ได้ด้วยคีย์เวิร์ด `MATERIALIZED` หรือ `NOT MATERIALIZED` และมี **ค่าเริ่มต้นอัตโนมัติ (auto)** ที่ optimizer จะเลือกให้เองตามกฎต่อไปนี้:

- ถ้า CTE ถูกอ้างอิง **เพียงครั้งเดียว** ใน query หลัก และไม่มีผลข้างเคียง (ไม่ใช่ writable CTE) → PostgreSQL จะปฏิบัติเหมือนเป็น `NOT MATERIALIZED` โดยอัตโนมัติ คือ "inline" เนื้อหาของ CTE เข้าไปในแผนการทำงานโดยตรง เหมือนเขียนเป็น subquery ธรรมดา — ทำให้ optimizer ปรับแผนร่วมกับส่วนอื่นได้เต็มที่
- ถ้า CTE ถูกอ้างอิง **มากกว่าหนึ่งครั้ง** → ค่าเริ่มต้นจะเป็น `MATERIALIZED` (รันครั้งเดียว เก็บผลลัพธ์ไว้ใช้ซ้ำ) เพื่อไม่ให้ต้องคำนวณซ้ำหลายรอบโดยไม่จำเป็น
- ถ้า CTE เป็น **writable CTE** (มี `INSERT`/`UPDATE`/`DELETE`) → จะเป็น `MATERIALIZED` เสมอ เพราะการ modify ข้อมูลต้องเกิดขึ้นแค่ครั้งเดียวอย่างแน่นอน (ดู Step 247-248)
- ถ้า CTE มีฟังก์ชันที่ไม่ deterministic เช่น `random()`, `now()`, หรือ volatile function อื่นๆ → มักจะถูก materialize เพื่อความสอดคล้องของผลลัพธ์ภายในคำสั่งเดียว

เราสามารถ **บังคับ** พฤติกรรมได้ด้วยตัวเองโดยใส่คีย์เวิร์ดตรงๆ:

```sql
WITH expensive_calc AS MATERIALIZED (
    SELECT product_id, unit_price, unit_price * 1.07 AS price_with_vat
    FROM products
)
SELECT * FROM expensive_calc WHERE price_with_vat > 10000;
```

```sql
WITH simple_filter AS NOT MATERIALIZED (
    SELECT * FROM products WHERE is_active = true
)
SELECT * FROM simple_filter WHERE category_id = 2;
```

### ตัวอย่างเปรียบเทียบผลกระทบต่อ performance

```sql
-- NOT MATERIALIZED: planner สามารถ push เงื่อนไข category_id = 2
-- เข้าไปกรองตั้งแต่ต้น ทำให้ scan สินค้าน้อยลง
EXPLAIN (COSTS OFF)
WITH active_products AS NOT MATERIALIZED (
    SELECT * FROM products WHERE is_active = true
)
SELECT * FROM active_products WHERE category_id = 2;
```

```text
                      QUERY PLAN
--------------------------------------------------------
 Seq Scan on products
   Filter: (is_active AND (category_id = 2))
(2 rows)
```

```sql
-- MATERIALIZED: planner ต้องรัน CTE ให้เสร็จสมบูรณ์
-- (scan สินค้า active ทั้งหมด) ก่อน แล้วค่อยกรอง category_id ทีหลัง
EXPLAIN (COSTS OFF)
WITH active_products AS MATERIALIZED (
    SELECT * FROM products WHERE is_active = true
)
SELECT * FROM active_products WHERE category_id = 2;
```

```text
                        QUERY PLAN
------------------------------------------------------------
 CTE Scan on active_products
   Filter: (category_id = 2)
   CTE active_products
     ->  Seq Scan on products
           Filter: is_active
(5 rows)
```

จะเห็นว่าแผนที่สองมีขั้นตอน "CTE Scan" เพิ่มเข้ามาและกรองเงื่อนไข `category_id = 2` **หลังจาก** materialize ข้อมูล active ทั้งหมดแล้ว ซึ่งในตารางเล็กๆ แบบนี้แทบไม่ต่างกัน แต่ถ้าเป็นตารางขนาดหลายล้านแถว การ push filter ลงไปตั้งแต่ต้น (แบบ NOT MATERIALIZED) จะช่วยลดจำนวนแถวที่ต้องอ่านได้มากอย่างมีนัยสำคัญ

### เมื่อไหร่ควรบังคับ MATERIALIZED เอง?

แม้ optimizer จะเลือกให้อัตโนมัติได้ดีในกรณีส่วนใหญ่ แต่มีบางสถานการณ์ที่ควรบังคับ `MATERIALIZED` ด้วยตัวเอง:

1. **CTE คำนวณหนักและถูกอ้างอิงหลายครั้งแบบมีเงื่อนไขต่างกัน** — ถ้าปล่อยให้ inline อาจทำให้ query ต้องคำนวณ logic เดิมซ้ำหลายรอบโดยไม่รู้ตัว
2. **ต้องการ "fence" การ optimize ไว้โดยตั้งใจ** เช่น เมื่อ query ซับซ้อนมากจน planner เลือกแผนที่ผิดพลาด การ materialize CTE บางตัวช่วยให้ควบคุมลำดับการทำงานได้ชัดเจนขึ้น
3. **CTE มีการเรียกฟังก์ชันที่ "แพง"** (expensive function) และเราต้องการให้มันถูกเรียกแค่ครั้งเดียวแน่นอน แม้ optimizer จะ auto-detect กรณีนี้ได้บ้าง แต่การระบุชัดเจนช่วยให้แน่ใจ 100%

---

## Step 246: CTE ร่วมกับ JOIN และ Aggregate Function ในสถานการณ์ Analytics จริง

ในงานวิเคราะห์ข้อมูลจริง เรามักต้องรวมหลายตารางเข้าด้วยกัน คำนวณค่าสถิติ แล้วกรองผลลัพธ์ระดับกลุ่ม — CTE ช่วยแบ่งงานเหล่านี้ออกเป็นขั้นตอนที่ชัดเจน

### ตัวอย่างที่ 1: สินค้าขายดี พร้อมคะแนนรีวิวเฉลี่ย

โจทย์: หา Top 5 สินค้าที่มียอดขาย (จาก order_items ทุกออเดอร์ ไม่จำกัดสถานะ) สูงสุด พร้อมแสดงคะแนนรีวิวเฉลี่ยประกบไปด้วย

```sql
WITH product_sales AS (
    SELECT
        oi.product_id,
        SUM(oi.quantity)               AS total_units_sold,
        SUM(oi.quantity * oi.unit_price) AS total_revenue
    FROM order_items oi
    GROUP BY oi.product_id
),
product_ratings AS (
    SELECT
        product_id,
        ROUND(AVG(rating), 2) AS avg_rating,
        COUNT(*)              AS review_count
    FROM reviews
    GROUP BY product_id
)
SELECT
    p.product_name,
    ps.total_units_sold,
    ps.total_revenue,
    COALESCE(pr.avg_rating, 0)   AS avg_rating,
    COALESCE(pr.review_count, 0) AS review_count
FROM product_sales ps
JOIN products p ON p.product_id = ps.product_id
LEFT JOIN product_ratings pr ON pr.product_id = ps.product_id
ORDER BY ps.total_revenue DESC
LIMIT 5;
```

**ผลลัพธ์ตัวอย่าง:**

```text
     product_name     | total_units_sold | total_revenue | avg_rating | review_count
------------------------+--------------------+------------------+--------------+---------------
 Laptop Pro 15"         |                  2 |        65800.00 |         4.50 |             2
 Smartphone X12         |                  2 |        49800.00 |         5.00 |             1
 Smartphone Lite        |                  2 |        17980.00 |         0.00 |             0
 Bluetooth Earbuds      |                  3 |         5970.00 |         4.00 |             2
 Robot Vacuum           |                  1 |        12900.00 |         3.00 |             1
(5 rows)
```

### ตัวอย่างที่ 2: วิเคราะห์ยอดขายรายเดือนตามประเทศปลายทาง

```sql
WITH monthly_country_sales AS (
    SELECT
        DATE_TRUNC('month', o.order_date)::date AS sales_month,
        o.ship_country,
        SUM(pay.amount) AS revenue,
        COUNT(DISTINCT o.order_id) AS order_count
    FROM orders o
    JOIN payments pay ON pay.order_id = o.order_id
    GROUP BY DATE_TRUNC('month', o.order_date), o.ship_country
),
country_totals AS (
    SELECT ship_country, SUM(revenue) AS country_revenue
    FROM monthly_country_sales
    GROUP BY ship_country
)
SELECT
    mcs.sales_month,
    mcs.ship_country,
    mcs.order_count,
    mcs.revenue,
    ct.country_revenue AS country_total_all_months
FROM monthly_country_sales mcs
JOIN country_totals ct ON ct.ship_country = mcs.ship_country
ORDER BY mcs.sales_month, mcs.revenue DESC;
```

**ผลลัพธ์ตัวอย่าง (บางส่วน):**

```text
 sales_month | ship_country | order_count |  revenue  | country_total_all_months
-------------+---------------+-------------+-----------+----------------------------
 2025-01-01  | Thailand      |           4 |  40950.00 |                   90980.00
 2025-02-01  | USA           |           1 |   8900.00 |                   15890.00
 2025-02-01  | Thailand      |           2 |   4770.00 |                   90980.00
 2025-02-01  | China         |           1 |  24900.00 |                   24900.00
 2025-03-01  | Thailand      |           2 |  45370.00 |                   90980.00
 2025-03-01  | Vietnam       |           1 |   2970.00 |                    2970.00
 2025-03-01  | Singapore     |           1 |   8990.00 |                    8990.00
 2025-03-01  | USA           |           1 |   6990.00 |                   15890.00
 2025-04-01  | Thailand      |           1 |   2180.00 |                   90980.00
(9 rows)
```

รูปแบบนี้ — CTE หนึ่งตัวคำนวณสถิติแบบละเอียด (breakdown) อีกตัวคำนวณผลรวมระดับที่กว้างกว่า แล้ว `JOIN` กลับเพื่อเทียบสัดส่วน — เป็นรูปแบบ analytics ที่พบบ่อยมากในการทำรายงานธุรกิจ

### ตัวอย่างที่ 3: ลูกค้าที่ไม่เคยสั่งซื้อเลย (Anti-join ผ่าน CTE)

```sql
WITH customers_with_orders AS (
    SELECT DISTINCT customer_id FROM orders
)
SELECT
    c.customer_id,
    c.first_name || ' ' || c.last_name AS customer_name,
    c.signup_date
FROM customers c
LEFT JOIN customers_with_orders cwo ON cwo.customer_id = c.customer_id
WHERE cwo.customer_id IS NULL
ORDER BY c.signup_date;
```

**ผลลัพธ์ตัวอย่าง:**

```text
 customer_id | customer_name  | signup_date
-------------+-----------------+-------------
(0 rows)
```

ในชุดข้อมูลนี้ลูกค้าทุกคนมีออเดอร์อย่างน้อยหนึ่งรายการ ผลลัพธ์จึงว่างเปล่า — ซึ่งเป็นเรื่องปกติและแสดงให้เห็นว่า query ทำงานถูกต้องตามตรรกะ

---

## Step 247: Writable CTE — INSERT / UPDATE / DELETE ... RETURNING ภายใน CTE

PostgreSQL อนุญาตให้ใส่คำสั่ง `INSERT`, `UPDATE`, หรือ `DELETE` (ที่มี `RETURNING`) ไว้ภายใน `WITH` clause ได้ เรียกว่า **Writable CTE** หรือ **Data-Modifying CTE** ทำให้เราสามารถ "แก้ไขข้อมูล" แล้ว "นำผลลัพธ์ที่ถูกแก้ไข" ไปใช้ต่อใน query เดียวกันได้ทันที — มีประโยชน์มากสำหรับสร้าง data pipeline สั้นๆ ที่ atomic (ทำสำเร็จหรือ rollback ทั้งหมดพร้อมกัน)

### รูปแบบพื้นฐาน

```sql
WITH modified_rows AS (
    INSERT INTO some_table (...) VALUES (...)
    RETURNING *
)
SELECT * FROM modified_rows;
```

### ตัวอย่างที่ 1: อัปเดตสต็อกสินค้าหลังขาย แล้วดูรายการที่ถูกแก้ไขทันที

โจทย์: ลดสต็อกสินค้าหมวด Kitchen ลง 5 ชิ้น (สมมติเป็นการปรับปรุงสต็อกหลัง stock take) แล้วแสดงรายการที่ถูกแก้ไขพร้อมสต็อกใหม่

```sql
WITH stock_adjustment AS (
    UPDATE products
    SET stock_quantity = stock_quantity - 5
    WHERE category_id = 5  -- Kitchen
      AND stock_quantity >= 5
    RETURNING product_id, product_name, stock_quantity AS new_stock
)
SELECT * FROM stock_adjustment ORDER BY product_id;
```

**ผลลัพธ์ตัวอย่าง:**

```text
 product_id | product_name  | new_stock
------------+----------------+-----------
          8 | Air Fryer 5L   |        40
          9 | Stand Mixer    |        10
(2 rows)
```

### ตัวอย่างที่ 2: INSERT พร้อม RETURNING เพื่อยืนยันแถวที่เพิ่งสร้าง

```sql
WITH new_customer AS (
    INSERT INTO customers (first_name, last_name, email, country)
    VALUES ('Piti', 'Maneerat', 'piti.m@example.com', 'Thailand')
    RETURNING customer_id, first_name, last_name, signup_date
)
SELECT
    customer_id,
    first_name || ' ' || last_name AS full_name,
    signup_date,
    'Welcome email queued' AS next_action
FROM new_customer;
```

**ผลลัพธ์ตัวอย่าง:**

```text
 customer_id | full_name    | signup_date |     next_action
-------------+---------------+-------------+-----------------------
          16 | Piti Maneerat | 2025-09-25  | Welcome email queued
(1 row)
```

### ตัวอย่างที่ 3: Data Pipeline — ย้ายสินค้าที่ไม่ active ไปเก็บในตาราง archive ในคำสั่งเดียว

นี่คือรูปแบบ pipeline ที่ใช้บ่อยในงานจริง: "ลบ" แถวออกจากตารางหลักพร้อมกับ "แทรก" มันเข้าตาราง archive ในคำสั่งเดียวแบบ atomic

```sql
CREATE TABLE IF NOT EXISTS discontinued_products (
    product_id      INTEGER PRIMARY KEY,
    product_name    VARCHAR(150),
    category_id     INTEGER,
    supplier_id     INTEGER,
    unit_price      NUMERIC(10,2),
    stock_quantity  INTEGER,
    archived_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

WITH archived AS (
    DELETE FROM products
    WHERE is_active = false
    RETURNING product_id, product_name, category_id, supplier_id, unit_price, stock_quantity
)
INSERT INTO discontinued_products
        (product_id, product_name, category_id, supplier_id, unit_price, stock_quantity)
SELECT product_id, product_name, category_id, supplier_id, unit_price, stock_quantity
FROM archived
RETURNING *;
```

**ผลลัพธ์ตัวอย่าง:**

```text
 product_id |       product_name       | category_id | supplier_id | unit_price | stock_quantity |         archived_at
------------+---------------------------+--------------+--------------+------------+------------------+-------------------------------
         19 | Old Model Tablet          |            1 |            2 |    5990.00 |                5 | 2025-09-25 12:00:00.123456+07
         20 | Discontinued Smartwatch  |            1 |           10 |    3990.00 |                0 | 2025-09-25 12:00:00.123456+07
(2 rows)
```

สังเกตโครงสร้างที่สำคัญ: คำสั่งนี้เริ่มด้วย `WITH archived AS (DELETE ... RETURNING ...)` แต่คำสั่งหลักของทั้งประโยคกลับเป็น `INSERT INTO ... SELECT ... FROM archived` (ไม่ใช่ `SELECT` ธรรมดา) นี่คือรูปแบบมาตรฐานของ writable CTE ที่ใช้ทำ data migration: **DELETE ในตารางต้นทาง → INSERT ในตารางปลายทาง** ทั้งหมดเกิดขึ้นภายใน transaction เดียว ถ้าขั้นตอนใดล้มเหลว ทั้งคำสั่งจะ rollback กลับหมด ข้อมูลจะไม่ตกหล่นอยู่ระหว่างกลาง (ไม่มีสถานะที่ลบไปแล้วแต่ insert ไม่สำเร็จ)

> **ข้อควรระวัง:** writable CTE **ไม่รองรับ** `WHERE CURRENT OF` และการ `INSERT`/`UPDATE`/`DELETE` หลายคำสั่งที่แก้ไข**ตารางเดียวกัน**ในลักษณะที่ผลลัพธ์ของกันและกันขึ้นต่อกันโดยตรงในบางกรณีจะมีพฤติกรรมที่ไม่ชัดเจน (undefined behavior) ซึ่งจะอธิบายเพิ่มเติมใน Step 249

---

## Step 248: Writable CTE ขั้นสูง — หลายขั้นตอนเชื่อมต่อกัน

Writable CTE ที่ทรงพลังที่สุดคือเมื่อเรามี CTE หลายตัวที่ **insert แล้วส่งผลลัพธ์ (RETURNING) ต่อให้ CTE ถัดไปใช้ insert อีกที** — ทำให้เราสร้างข้อมูลที่มีความสัมพันธ์แบบ foreign key กันได้ในคำสั่งเดียว โดยไม่ต้องแยกรันหลายคำสั่งและส่ง id กลับไปกลับมาด้วยโค้ดฝั่ง application

### ตัวอย่างที่ 1: สร้างหมวดหมู่ใหม่ แล้วสร้างสินค้าในหมวดนั้นทันที

โจทย์: เพิ่มหมวดหมู่ย่อย "Gaming" ภายใต้ "Electronics" แล้วเพิ่มสินค้าตัวแรกในหมวดนั้นทันที โดยต้องใช้ `category_id` ที่เพิ่งถูกสร้างขึ้น (มาจาก `RETURNING` ของ CTE แรก) เป็นค่าที่ insert เข้าตาราง `products`

```sql
WITH new_category AS (
    INSERT INTO categories (category_name, parent_category_id)
    VALUES ('Gaming', 1)  -- 1 = Electronics
    RETURNING category_id
),
new_product AS (
    INSERT INTO products (product_name, category_id, supplier_id, unit_price, stock_quantity, is_active)
    SELECT
        'Gaming Console Z',
        new_category.category_id,
        1,          -- supplier_id = TechSource Co., Ltd.
        15900.00,
        10,
        true
    FROM new_category
    RETURNING product_id, product_name, category_id, unit_price
)
SELECT
    nc.category_id,
    np.product_id,
    np.product_name,
    np.unit_price
FROM new_category nc
JOIN new_product np ON np.category_id = nc.category_id;
```

**ผลลัพธ์ตัวอย่าง:**

```text
 category_id | product_id |   product_name    | unit_price
-------------+------------+--------------------+-------------
          11 |         21 | Gaming Console Z   |   15900.00
(1 row)
```

จุดสำคัญของตัวอย่างนี้คือ `new_product` ใช้ `SELECT new_category.category_id FROM new_category` แทนการเขียน `category_id` เป็นค่าคงที่ตรงๆ เพราะเรายังไม่รู้ล่วงหน้าว่า `SERIAL` จะ generate ค่าอะไรให้ — การอ้างอิง CTE ก่อนหน้าแบบนี้ทำให้ทั้งสองคำสั่ง insert **ผูกกันด้วยค่าจริงที่เพิ่งถูกสร้าง** อย่างถูกต้องแน่นอน โดยไม่ต้องเขียนโค้ดฝั่งแอปพลิเคชันมาคอย query หา id กลับ

### ตัวอย่างที่ 2: ย้ายข้อมูลระหว่างตาราง 2 ขั้นตอน — เก็บ log การเปลี่ยนแปลงราคาไปพร้อมกัน

โจทย์: ปรับราคาสินค้าทุกชิ้นในหมวด "Computers" ขึ้น 5% (เนื่องจากต้นทุนวัตถุดิบเพิ่มขึ้น) และบันทึก log การเปลี่ยนแปลงราคาไว้ในตารางประวัติ ทั้งหมดในคำสั่งเดียว

```sql
CREATE TABLE IF NOT EXISTS price_change_log (
    log_id       SERIAL PRIMARY KEY,
    product_id   INTEGER NOT NULL,
    old_price    NUMERIC(10,2) NOT NULL,
    new_price    NUMERIC(10,2) NOT NULL,
    changed_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);

WITH price_updates AS (
    UPDATE products
    SET unit_price = ROUND(unit_price * 1.05, 2)
    WHERE category_id = 2  -- Computers
    RETURNING product_id, unit_price AS new_price
),
old_prices AS (
    -- ดึงราคาก่อนอัปเดตจาก order_items ล่าสุดของสินค้านั้น เป็นค่าอ้างอิงราคาที่เคยขาย
    -- (ในสถานการณ์จริงมักมีตาราง price_history แยกไว้อยู่แล้ว แต่ตัวอย่างนี้สาธิต
    --  การคำนวณราคาก่อนหน้าจาก new_price ย้อนกลับเพื่อความง่าย)
    SELECT
        pu.product_id,
        ROUND(pu.new_price / 1.05, 2) AS old_price,
        pu.new_price
    FROM price_updates pu
),
logged AS (
    INSERT INTO price_change_log (product_id, old_price, new_price)
    SELECT product_id, old_price, new_price
    FROM old_prices
    RETURNING product_id, old_price, new_price
)
SELECT
    p.product_name,
    l.old_price,
    l.new_price,
    ROUND(l.new_price - l.old_price, 2) AS increase
FROM logged l
JOIN products p ON p.product_id = l.product_id
ORDER BY l.product_id;
```

**ผลลัพธ์ตัวอย่าง:**

```text
     product_name     | old_price | new_price | increase
------------------------+-----------+-----------+----------
 Laptop Pro 15"         |  32900.00 |  34545.00 |  1645.00
 Wireless Mouse         |    590.00 |    619.50 |    29.50
 Mechanical Keyboard    |   2490.00 |   2614.50 |   124.50
 4K Monitor 27"         |   8900.00 |   9345.00 |   445.00
(4 rows)
```

รูปแบบนี้แสดงให้เห็น pipeline สามขั้นตอนเต็มรูปแบบ: **UPDATE (แก้ราคา) → คำนวณ/เตรียมข้อมูล log → INSERT (บันทึก log)** ทั้งหมดอยู่ใน transaction เดียว รับประกันว่าถ้าการบันทึก log ล้มเหลวด้วยเหตุผลใดก็ตาม (เช่น constraint violation) การอัปเดตราคาจะถูก rollback ไปด้วย ไม่มีทางที่ราคาจะถูกเปลี่ยนโดยไม่มี log บันทึกไว้

---

## Step 249: ข้อจำกัดของ CTE และเมื่อไหร่ไม่ควรใช้

แม้ CTE จะมีประโยชน์มาก แต่ก็มีข้อจำกัดที่ผู้เรียนควรรู้ เพื่อเลือกเครื่องมือให้เหมาะกับสถานการณ์

### ข้อจำกัดหลัก

1. **ไม่มี index บน CTE โดยตรง** — CTE ไม่ใช่ตารางจริง จึงสร้าง index บนมันไม่ได้ ถ้า query ต้องใช้ CTE เดิมซ้ำหลายครั้งพร้อม filter ต่างกันจำนวนมาก และข้อมูลมีปริมาณมาก การ scan ซ้ำๆ (แม้จะ materialize ไว้แล้ว) อาจช้ากว่าการมี index ช่วย

2. **ไม่มี statistics แยกสำหรับ CTE** — เมื่อ CTE ถูก `MATERIALIZED` planner จะไม่รู้ค่าสถิติ (เช่น การกระจายของข้อมูล, จำนวนแถวโดยประมาณที่แม่นยำ) ของผลลัพธ์ภายใน CTE เหมือนที่รู้จักตารางจริงที่เคยรัน `ANALYZE` ทำให้บางครั้ง planner เลือกแผนการ join ที่ไม่เหมาะสมที่สุด

3. **Writable CTE หลายตัวที่แก้ไขตารางเดียวกันมีพฤติกรรมไม่ชัดเจน** — เอกสารของ PostgreSQL ระบุชัดว่า ถ้ามี writable CTE มากกว่าหนึ่งตัวพยายามแก้ไขแถวเดียวกันในตารางเดียวกัน ผลลัพธ์จะไม่แน่นอน (unspecified) เพราะ CTE ทั้งหมดในคำสั่งเดียวเห็นภาพข้อมูล (snapshot) เดียวกัน ณ ตอนเริ่มคำสั่ง — ไม่เห็นผลของกันและกันในลักษณะ "ลำดับ" เหมือนการรันหลายคำสั่งแยกกัน ควรหลีกเลี่ยงการออกแบบให้ writable CTE สองตัวชนกันบนแถวเดียวกัน

4. **ใช้ครั้งเดียวต่อคำสั่ง SQL** — CTE มีชีวิตอยู่แค่ในคำสั่งเดียว (statement scope) ไม่สามารถนำไปใช้ต่อในคำสั่งถัดไปได้ ถ้าต้องการผลลัพธ์กลางที่ใช้ซ้ำได้หลายคำสั่ง ต้องใช้ temporary table หรือ view แทน

5. **ไม่เหมาะกับข้อมูลปริมาณมากที่ต้องประมวลผลซับซ้อนหลายรอบ** — เมื่อ CTE ถูก materialize มันจะถูกเก็บใน work_mem/temp storage ชั่วคราว ถ้าข้อมูลมีขนาดใหญ่มากและต้องผ่านการประมวลผลหลายขั้นตอนซับซ้อน (เช่น join ซ้ำไปมาหลายรอบ, ต้องการ index ระหว่างขั้นตอน) การใช้ **temporary table** จะเหมาะกว่า เพราะสามารถสร้าง index และรัน `ANALYZE` เพื่อให้ planner มีสถิติที่แม่นยำสำหรับขั้นตอนถัดไปได้

### เปรียบเทียบ CTE กับ Temporary Table

| ประเด็น | CTE | Temporary Table |
|---|---|---|
| อายุการใช้งาน | รันเดียวจบ (statement) | อยู่ตลอด session (หรือ transaction) |
| สร้าง index ได้ | ไม่ได้ | ได้ |
| มี statistics (ANALYZE) | ไม่มี (โดยเฉพาะเมื่อ MATERIALIZED) | มี ถ้ารัน `ANALYZE` |
| ใช้ซ้ำข้ามหลายคำสั่ง | ไม่ได้ | ได้ |
| เหมาะกับ | logic วิเคราะห์ที่อ่านง่าย, pipeline สั้นๆ, ข้อมูลขนาดกลาง | batch job ที่ซับซ้อนหลายขั้นตอน, ข้อมูลขนาดใหญ่, ETL |
| Syntax | กระชับ อยู่ใน query เดียว | ต้องเขียนหลายคำสั่งแยกกัน |

### ตัวอย่าง: เมื่อควรเปลี่ยนจาก CTE เป็น Temporary Table

ถ้าเรามี pipeline วิเคราะห์ข้อมูลที่ซับซ้อนมาก ต้องคำนวณ "ยอดขายต่อลูกค้า" แล้วนำไปทำ self-join ซับซ้อนหลายรอบ (เช่น หาคู่ลูกค้าที่ซื้อสินค้าคล้ายกัน) บนข้อมูลระดับล้านแถว การเขียนแบบ CTE (materialize ทุกครั้งที่รัน query) จะเสียเวลาคำนวณใหม่ทุกครั้ง แต่การใช้ temp table ทำให้คำนวณครั้งเดียว สร้าง index แล้วนำไปใช้ต่อได้หลายคำสั่ง:

```sql
-- แทนที่จะใช้ CTE ที่ต้อง materialize ใหม่ทุกครั้ง
CREATE TEMP TABLE tmp_customer_revenue AS
SELECT
    o.customer_id,
    SUM(pay.amount) AS total_revenue
FROM orders o
JOIN payments pay ON pay.order_id = o.order_id
GROUP BY o.customer_id;

-- สร้าง index เพื่อให้ query ถัดไปเร็วขึ้น (ทำไม่ได้กับ CTE)
CREATE INDEX ON tmp_customer_revenue (total_revenue);

-- รัน ANALYZE เพื่อให้ planner มีสถิติที่แม่นยำ (ทำไม่ได้กับ CTE)
ANALYZE tmp_customer_revenue;

-- นำไปใช้ต่อได้หลายคำสั่ง โดยไม่ต้องคำนวณซ้ำ
SELECT * FROM tmp_customer_revenue WHERE total_revenue > 20000;
SELECT AVG(total_revenue) FROM tmp_customer_revenue;
```

**กฎง่ายๆ ในการตัดสินใจ:** ถ้า query ของเราต้องรันเพียงครั้งเดียวและอ่านง่ายเป็นสิ่งสำคัญที่สุด → ใช้ CTE ถ้าต้องประมวลผลข้อมูลจำนวนมากหลายขั้นตอนซับซ้อน หรือต้องนำผลลัพธ์กลางไปใช้ในหลายคำสั่งแยกกัน → ใช้ temporary table

---

## Step 250: แบบฝึกหัดรวม — รายงานวิเคราะห์ยอดขาย E-commerce ด้วย CTE หลายชั้น

มาลองประกอบทุกเทคนิคที่เรียนมาในบทนี้เข้าด้วยกัน เพื่อสร้าง **รายงานวิเคราะห์ยอดขายฉบับสมบูรณ์** สำหรับผู้บริหารร้านค้าออนไลน์

**โจทย์:** สร้างรายงานที่แสดง สำหรับแต่ละหมวดหมู่สินค้า (เฉพาะหมวดที่มียอดขาย):
1. รายได้รวม (จากออเดอร์ที่ชำระเงินแล้วเท่านั้น)
2. จำนวนสินค้าที่ขายได้
3. คะแนนรีวิวเฉลี่ยของสินค้าในหมวดนั้น
4. สัดส่วนรายได้เทียบกับรายได้รวมทั้งร้าน (เปอร์เซ็นต์)
5. จัดอันดับหมวดหมู่ตามรายได้ พร้อมป้าย "Top Category" สำหรับ 3 อันดับแรก

```sql
WITH paid_order_items AS (
    -- ชั้นที่ 1: รวม order_items เข้ากับออเดอร์ที่ชำระเงินแล้ว
    SELECT
        oi.product_id,
        oi.quantity,
        oi.quantity * oi.unit_price AS line_revenue
    FROM order_items oi
    JOIN orders o     ON o.order_id = oi.order_id
    JOIN payments pay ON pay.order_id = o.order_id
),
category_sales AS (
    -- ชั้นที่ 2: รวมยอดขายต่อหมวดหมู่
    SELECT
        p.category_id,
        SUM(poi.quantity)     AS units_sold,
        SUM(poi.line_revenue) AS category_revenue
    FROM paid_order_items poi
    JOIN products p ON p.product_id = poi.product_id
    GROUP BY p.category_id
),
category_ratings AS (
    -- ชั้นที่ 3: คะแนนรีวิวเฉลี่ยต่อหมวดหมู่
    SELECT
        p.category_id,
        ROUND(AVG(r.rating), 2) AS avg_rating
    FROM reviews r
    JOIN products p ON p.product_id = r.product_id
    GROUP BY p.category_id
),
report_base AS (
    -- ชั้นที่ 4: รวมทุกอย่างเข้าด้วยกัน พร้อมคำนวณสัดส่วน (ใช้ category_sales ซ้ำเพื่อหา grand total)
    SELECT
        cat.category_name,
        cs.units_sold,
        cs.category_revenue,
        COALESCE(cr.avg_rating, 0) AS avg_rating,
        ROUND(cs.category_revenue * 100.0 / (SELECT SUM(category_revenue) FROM category_sales), 2) AS pct_of_total
    FROM category_sales cs
    JOIN categories cat ON cat.category_id = cs.category_id
    LEFT JOIN category_ratings cr ON cr.category_id = cs.category_id
)
SELECT
    category_name,
    units_sold,
    category_revenue,
    avg_rating,
    pct_of_total,
    CASE
        WHEN category_revenue >= (
            SELECT category_revenue FROM report_base ORDER BY category_revenue DESC OFFSET 2 LIMIT 1
        ) THEN 'Top Category'
        ELSE ''
    END AS ranking_flag
FROM report_base
ORDER BY category_revenue DESC;
```

**ผลลัพธ์ตัวอย่าง:**

```text
    category_name    | units_sold | category_revenue | avg_rating | pct_of_total | ranking_flag
-----------------------+------------+---------------------+--------------+----------------+---------------
 Computers             |          4 |            77780.00 |         4.50 |          40.33 | Top Category
 Smartphones            |          2 |            58790.00 |         5.00 |          30.48 | Top Category
 Kitchen                |          2 |            13980.00 |         4.50 |           7.25 | Top Category
 Home Appliances        |          1 |            12900.00 |         3.00 |           6.69 |
 Electronics            |          3 |             5970.00 |         4.00 |           3.10 |
 Men's Clothing         |          2 |             4770.00 |         4.00 |           2.47 |
 Women's Clothing       |          3 |             2970.00 |         4.00 |           1.54 |
 Books                  |          1 |             2180.00 |         4.50 |           1.13 |
(8 rows)
```

รายงานนี้แสดงให้เห็นพลังของการรวม CTE หลายชั้น (`paid_order_items` → `category_sales` → `category_ratings` → `report_base`) เข้ากับการ **ใช้ CTE ซ้ำ** (`category_sales` ถูกอ้างอิงทั้งใน `report_base` และใน subquery คำนวณ grand total และ ranking) — logic ทั้งหมดถูกเขียนเป็นขั้นตอนที่อ่านและตรวจสอบได้ทีละชั้น ตรงตามแนวคิดหลักของบทนี้

---

## สรุปท้ายบท

ในบทนี้เราได้เรียนรู้การใช้ Common Table Expression (CTE) ผ่าน `WITH` clause อย่างครบถ้วน:

- **CTE คืออะไร (Step 241-242):** ผลลัพธ์ชั่วคราวที่ตั้งชื่อไว้ล่วงหน้า ช่วยแปลง nested subquery ที่อ่านยากให้กลายเป็นขั้นตอนที่อ่านจากบนลงล่างได้เหมือนเขียนโปรแกรม
- **หลาย CTE ในคำสั่งเดียว (Step 243-244):** ประกาศ CTE หลายตัวคั่นด้วยจุลภาค อ้างอิงกันเป็นลำดับ (chained) และใช้ CTE เดียวกันซ้ำได้หลายจุดเพื่อลดโค้ดซ้ำซ้อนและความเสี่ยงต่อ logic ไม่ตรงกัน
- **Materialized vs Not Materialized (Step 245):** ตั้งแต่ PostgreSQL 12 CTE ที่ถูกอ้างอิงครั้งเดียวจะถูก inline โดยอัตโนมัติ (เหมือน subquery) ส่วน CTE ที่ถูกใช้หลายครั้งหรือเป็น writable CTE จะถูก materialize โดยดีฟอลต์ — และเราสามารถบังคับพฤติกรรมได้ด้วยคีย์เวิร์ด `MATERIALIZED`/`NOT MATERIALIZED`
- **CTE ในงาน Analytics จริง (Step 246):** ผสาน CTE กับ `JOIN` และ aggregate function เพื่อสร้างรายงานวิเคราะห์แบบหลายชั้น
- **Writable CTE (Step 247-248):** ใช้ `INSERT`/`UPDATE`/`DELETE ... RETURNING` ภายใน CTE เพื่อสร้าง data pipeline แบบ atomic ในคำสั่งเดียว รวมถึงรูปแบบขั้นสูงที่ insert ตารางหนึ่งแล้วส่งผลไป insert ต่ออีกตาราง
- **ข้อจำกัดของ CTE (Step 249):** ไม่มี index, ไม่มี statistics, ใช้ได้แค่ในคำสั่งเดียว — เมื่อข้อมูลใหญ่มากและซับซ้อนหลายขั้นตอน ควรพิจารณาใช้ temporary table แทน
- **แบบฝึกหัดรวม (Step 250):** ประกอบทุกเทคนิคเข้าด้วยกันเป็นรายงานวิเคราะห์ยอดขายฉบับสมบูรณ์

CTE เป็นเครื่องมือที่ทรงพลังที่สุดตัวหนึ่งสำหรับการเขียน SQL ให้อ่านง่ายและดูแลรักษาได้ในระยะยาว แต่สิ่งที่เรายังไม่ได้พูดถึงคือ CTE ที่ **อ้างอิงตัวเอง** ซึ่งจำเป็นสำหรับการไล่โครงสร้างข้อมูลแบบลำดับชั้น (hierarchy) เช่น หมวดหมู่สินค้าที่มีหมวดย่อยซ้อนกันหลายชั้น หรือโครงสร้างสายการบังคับบัญชาของพนักงาน — เนื้อหานี้จะอยู่ในบทถัดไป

---

## แบบฝึกหัด

<details>
<summary><strong>แบบฝึกหัดที่ 1:</strong> เขียน CTE ชื่อ <code>active_customers</code> ที่คัดเฉพาะลูกค้าที่มีออเดอร์อย่างน้อย 1 รายการ แล้วนำมา SELECT แสดงชื่อลูกค้าและประเทศ เรียงตามชื่อ</summary>

```sql
WITH active_customers AS (
    SELECT DISTINCT c.customer_id, c.first_name, c.last_name, c.country
    FROM customers c
    JOIN orders o ON o.customer_id = c.customer_id
)
SELECT
    first_name || ' ' || last_name AS customer_name,
    country
FROM active_customers
ORDER BY customer_name;
```

**คำอธิบาย:** ใช้ `DISTINCT` ใน CTE เพื่อไม่ให้ลูกค้าที่มีหลายออเดอร์ซ้ำหลายแถว จากนั้น `SELECT` จาก CTE ตามปกติ

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 2:</strong> ใช้ CTE 2 ตัวที่เชื่อมกัน (chained) เพื่อหา "สินค้าที่ไม่เคยถูกสั่งซื้อเลย" (ไม่มีแถวใน order_items)</summary>

```sql
WITH ordered_products AS (
    SELECT DISTINCT product_id FROM order_items
),
never_ordered AS (
    SELECT p.product_id, p.product_name
    FROM products p
    LEFT JOIN ordered_products op ON op.product_id = p.product_id
    WHERE op.product_id IS NULL
)
SELECT * FROM never_ordered ORDER BY product_id;
```

**ผลลัพธ์ตัวอย่าง:**

```text
 product_id |    product_name
------------+----------------------
          2 | Wireless Mouse
```

**คำอธิบาย:** ในชุดข้อมูลนี้ `product_id = 2` (Wireless Mouse) ถูกขายพ่วงไปกับ order 1 อยู่แล้ว ดังนั้นถ้ารันจริงอาจไม่มีแถวผลลัพธ์เลย (0 rows) ขึ้นกับข้อมูลที่ผู้เรียนแทรกเพิ่มเติมระหว่างฝึกฝน — จุดสำคัญคือรูปแบบ anti-join ผ่าน CTE ที่ใช้ `LEFT JOIN ... WHERE ... IS NULL`

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 3:</strong> แปลง query ที่มี correlated subquery ต่อไปนี้ให้เป็นรูปแบบ CTE: "หาพนักงานที่มียอดขาย (จาก payments) มากกว่าค่าเฉลี่ยยอดขายของพนักงานทุกคน"</summary>

```sql
WITH employee_sales AS (
    SELECT o.employee_id, SUM(pay.amount) AS total_sales
    FROM orders o
    JOIN payments pay ON pay.order_id = o.order_id
    GROUP BY o.employee_id
)
SELECT
    e.first_name || ' ' || e.last_name AS employee_name,
    es.total_sales
FROM employee_sales es
JOIN employees e ON e.employee_id = es.employee_id
WHERE es.total_sales > (SELECT AVG(total_sales) FROM employee_sales)
ORDER BY es.total_sales DESC;
```

**ผลลัพธ์ตัวอย่าง:**

```text
 employee_name  | total_sales
-----------------+-------------
 Siriwan Chaiyo  |    65450.00
 Anucha Petch    |    39070.00
```

**คำอธิบาย:** ค่าเฉลี่ยยอดขายของ 5 พนักงานคือ (65450+39070+33890+25850+15080)/5 = 35868 ดังนั้นมีเพียง Siriwan และ Anucha ที่สูงกว่าค่าเฉลี่ย

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 4:</strong> เขียน query ที่ใช้คีย์เวิร์ด <code>NOT MATERIALIZED</code> อย่างชัดเจน เพื่อกรองสินค้า active ในหมวด "Fashion" หรือหมวดย่อยของมัน (Men's Clothing, Women's Clothing) — ใช้ IN กับรายการ category_id ตรงๆ (ยังไม่ต้องใช้ recursive CTE)</summary>

```sql
WITH active_fashion AS NOT MATERIALIZED (
    SELECT * FROM products WHERE is_active = true
)
SELECT product_name, category_id, unit_price
FROM active_fashion
WHERE category_id IN (6, 7, 8)  -- Fashion, Men's Clothing, Women's Clothing
ORDER BY category_id, unit_price DESC;
```

**คำอธิบาย:** `NOT MATERIALIZED` บอกให้ planner "inline" CTE นี้เข้ากับ query หลัก ทำให้เงื่อนไข `category_id IN (6,7,8)` ถูก push ลงไปกรองพร้อมกับ `is_active = true` ตั้งแต่ scan ตาราง `products` ครั้งเดียว

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 5:</strong> ใช้ CTE เดียวกันซ้ำ 2 ครั้งเพื่อหา "ประเทศที่มียอดขายสูงกว่าค่าเฉลี่ยของทุกประเทศ" จากตาราง orders/payments</summary>

```sql
WITH country_revenue AS (
    SELECT o.ship_country, SUM(pay.amount) AS revenue
    FROM orders o
    JOIN payments pay ON pay.order_id = o.order_id
    GROUP BY o.ship_country
)
SELECT ship_country, revenue
FROM country_revenue
WHERE revenue > (SELECT AVG(revenue) FROM country_revenue)
ORDER BY revenue DESC;
```

**คำอธิบาย:** `country_revenue` ถูกอ้างอิง 2 ครั้ง (ใน `FROM` หลัก และใน subquery `AVG`) — PostgreSQL จะ materialize CTE นี้โดยอัตโนมัติเพราะถูกใช้มากกว่าหนึ่งครั้ง

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 6:</strong> เขียน Writable CTE ที่อัปเดตสถานะออเดอร์ที่ค้างสถานะ <code>pending</code> เกิน 7 วัน (จาก order_date) ให้เป็น <code>cancelled</code> พร้อมแสดงรายการที่ถูกเปลี่ยน</summary>

```sql
WITH cancelled_orders AS (
    UPDATE orders
    SET status = 'cancelled'
    WHERE status = 'pending'
      AND order_date < now() - INTERVAL '7 days'
    RETURNING order_id, customer_id, order_date, status
)
SELECT * FROM cancelled_orders ORDER BY order_id;
```

**คำอธิบาย:** เมื่อรันด้วยข้อมูลตัวอย่างที่ `order_date` เป็น 2025-03-15 (order_id 16) ถ้าวันปัจจุบันเกิน 2025-03-22 ไปแล้ว ออเดอร์นี้จะถูกเปลี่ยนเป็น `cancelled` และแสดงในผลลัพธ์

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 7:</strong> เขียน Writable CTE 2 ขั้นตอนที่ insert ลูกค้าใหม่ แล้ว insert ออเดอร์แรกให้ลูกค้าคนนั้นทันที โดยใช้ <code>customer_id</code> ที่ได้จาก RETURNING ของ CTE แรก</summary>

```sql
WITH new_cust AS (
    INSERT INTO customers (first_name, last_name, email, country)
    VALUES ('Araya', 'Chotiros', 'araya.c@example.com', 'Thailand')
    RETURNING customer_id
),
new_order AS (
    INSERT INTO orders (customer_id, employee_id, status, ship_country)
    SELECT customer_id, 2, 'pending', 'Thailand'
    FROM new_cust
    RETURNING order_id, customer_id, status
)
SELECT
    nc.customer_id,
    no.order_id,
    no.status
FROM new_cust nc
JOIN new_order no ON no.customer_id = nc.customer_id;
```

**คำอธิบาย:** รูปแบบนี้เหมือนตัวอย่างใน Step 248 — CTE ที่สอง (`new_order`) ใช้ `SELECT customer_id FROM new_cust` เพื่อดึงค่า `customer_id` ที่เพิ่ง insert สำเร็จมาใช้เป็น foreign key ในการ insert ออเดอร์ ทำให้มั่นใจได้ว่าข้อมูลสัมพันธ์กันถูกต้อง

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 8:</strong> อธิบายว่าทำไมคำสั่งต่อไปนี้อาจให้ผลลัพธ์ที่ไม่แน่นอน (unspecified behavior) และควรแก้ไขอย่างไร</summary>

```sql
-- ตัวอย่างโค้ดที่มีปัญหา (ห้ามใช้แบบนี้จริง)
WITH increase_stock AS (
    UPDATE products SET stock_quantity = stock_quantity + 10
    WHERE category_id = 2
    RETURNING product_id
),
decrease_stock AS (
    UPDATE products SET stock_quantity = stock_quantity - 3
    WHERE category_id = 2
    RETURNING product_id
)
SELECT * FROM increase_stock;
```

**คำตอบ:** ทั้งสอง CTE (`increase_stock` และ `decrease_stock`) พยายามแก้ไข**ตารางเดียวกัน** (`products`) บนแถวชุดเดียวกัน (`category_id = 2`) ในคำสั่งเดียว ซึ่งตามเอกสาร PostgreSQL ระบุว่าพฤติกรรมนี้ **unspecified** — เราไม่สามารถคาดเดาได้แน่นอนว่าผลลัพธ์สุดท้ายของ `stock_quantity` จะเป็นเท่าไหร่ (อาจจะเป็นผลจาก +10 อย่างเดียว, -3 อย่างเดียว, หรือค่าอื่นที่ไม่ตรงกับที่คาดหวัง) วิธีแก้คือรวม logic เป็น `UPDATE` เดียว หรือแยกรันเป็นคนละคำสั่ง (คนละ transaction statement) แทน:

```sql
WITH adjust_stock AS (
    UPDATE products
    SET stock_quantity = stock_quantity + 10 - 3
    WHERE category_id = 2
    RETURNING product_id, stock_quantity
)
SELECT * FROM adjust_stock;
```

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 9:</strong> เขียนรายงานด้วย CTE หลายชั้นที่แสดง "Top 3 ลูกค้าตามจำนวนรีวิวที่เขียน" พร้อมคะแนนเฉลี่ยที่ลูกค้าคนนั้นให้</summary>

```sql
WITH customer_review_stats AS (
    SELECT
        r.customer_id,
        COUNT(*) AS review_count,
        ROUND(AVG(r.rating), 2) AS avg_rating_given
    FROM reviews r
    GROUP BY r.customer_id
)
SELECT
    c.first_name || ' ' || c.last_name AS customer_name,
    crs.review_count,
    crs.avg_rating_given
FROM customer_review_stats crs
JOIN customers c ON c.customer_id = crs.customer_id
ORDER BY crs.review_count DESC, crs.avg_rating_given DESC
LIMIT 3;
```

**ผลลัพธ์ตัวอย่าง:**

```text
 customer_name  | review_count | avg_rating_given
-----------------+---------------+-------------------
 Somchai Jaidee  |             3 |              5.00
 Anong Phromma   |             2 |              3.50
 Emma Johnson    |             2 |              4.50
(3 rows)
```

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 10 (โจทย์รวม):</strong> เขียนรายงาน "Employee Performance Dashboard" ด้วย CTE หลายชั้น แสดง: ชื่อพนักงาน, แผนก, จำนวนออเดอร์ที่ดูแลทั้งหมด, จำนวนออเดอร์ที่ชำระเงินแล้ว, ยอดขายรวม, และเปรียบเทียบกับค่าเฉลี่ยของทั้งทีมว่าสูงกว่าหรือต่ำกว่า</summary>

```sql
WITH employee_orders AS (
    SELECT
        e.employee_id,
        COUNT(o.order_id) AS total_orders
    FROM employees e
    LEFT JOIN orders o ON o.employee_id = e.employee_id
    GROUP BY e.employee_id
),
employee_paid_sales AS (
    SELECT
        o.employee_id,
        COUNT(DISTINCT o.order_id) AS paid_orders,
        SUM(pay.amount) AS total_sales
    FROM orders o
    JOIN payments pay ON pay.order_id = o.order_id
    GROUP BY o.employee_id
),
team_avg AS (
    SELECT AVG(total_sales) AS avg_team_sales
    FROM employee_paid_sales
)
SELECT
    e.first_name || ' ' || e.last_name AS employee_name,
    e.department,
    eo.total_orders,
    COALESCE(eps.paid_orders, 0)  AS paid_orders,
    COALESCE(eps.total_sales, 0)  AS total_sales,
    CASE
        WHEN COALESCE(eps.total_sales, 0) > (SELECT avg_team_sales FROM team_avg)
            THEN 'Above Average'
        ELSE 'Below Average'
    END AS performance
FROM employee_orders eo
JOIN employees e ON e.employee_id = eo.employee_id
LEFT JOIN employee_paid_sales eps ON eps.employee_id = eo.employee_id
ORDER BY total_sales DESC NULLS LAST;
```

**ผลลัพธ์ตัวอย่าง:**

```text
  employee_name  | department | total_orders | paid_orders | total_sales |  performance
------------------+-------------+---------------+---------------+---------------+-----------------
 Siriwan Chaiyo   | Sales       |             5 |             5 |    65450.00 | Above Average
 Anucha Petch     | Sales       |             3 |             3 |    39070.00 | Above Average
 Ratchanee Phon   | Sales       |             4 |             2 |    33890.00 | Below Average
 Kamon Srisuk     | Sales       |             4 |             4 |    25850.00 | Below Average
 Nattaya Ruam     | Sales       |             4 |             2 |    15080.00 | Below Average
 Prasert Kittipong| Management  |             0 |             0 |        0.00 | Below Average
 Piyada Nakorn    | Support     |             0 |             0 |        0.00 | Below Average
 Somsak Yai       | Support     |             0 |             0 |        0.00 | Below Average
 Waraporn Sang    | Marketing   |             0 |             0 |        0.00 | Below Average
 Decha Boonmee    | Marketing   |             0 |             0 |        0.00 | Below Average
(10 rows)
```

**คำอธิบาย:** รายงานนี้รวม 3 CTE เข้าด้วยกัน: `employee_orders` (นับออเดอร์ทั้งหมดรวมที่ยังไม่ชำระเงิน), `employee_paid_sales` (ยอดขายจริงที่ชำระแล้ว), และ `team_avg` (ค่าเฉลี่ยของทีมสำหรับเปรียบเทียบ) — เป็นตัวอย่างสรุปการใช้ CTE หลายชั้นร่วมกับ `LEFT JOIN` และ `COALESCE` เพื่อจัดการพนักงานที่ไม่มีออเดอร์เลย (แผนก Support/Marketing/Management) ให้แสดงเป็น 0 แทนที่จะหายไปจากรายงาน

</details>

---

**บทถัดไป:** [Part 026: Recursive CTE](./part-026-recursive-cte.md) — เรียนรู้การใช้ `WITH RECURSIVE` เพื่อไล่โครงสร้างข้อมูลแบบลำดับชั้น เช่น หมวดหมู่สินค้าที่ซ้อนกันหลายชั้น และสายการบังคับบัญชาของพนักงาน
