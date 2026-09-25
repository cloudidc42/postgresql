# Part 026: Recursive CTE และการใช้งานกับข้อมูลแบบ Hierarchy

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับกลาง | Part 026

---

## เป้าหมายการเรียนรู้

หลังจากเรียนจบบทนี้ ผู้เรียนจะสามารถ:

- เข้าใจโครงสร้างและกลไกการทำงานของ `WITH RECURSIVE` ตั้งแต่ anchor member, recursive member ไปจนถึง UNION ALL
- เขียน recursive CTE พื้นฐานเพื่อสร้างลำดับตัวเลข ก่อนนำไปประยุกต์กับข้อมูลจริง
- ไล่ลูกหมวดหมู่ทั้งหมดจาก category หนึ่งตัวแบบ top-down (หา descendants)
- หา path ย้อนกลับขึ้นไปหา root category จาก category ใดๆ แบบ bottom-up (หา ancestors)
- เพิ่มคอลัมน์ level/depth และสร้าง path string เพื่อแสดงลำดับชั้นแบบอ่านง่าย (breadcrumb)
- สร้าง org chart ของพนักงานทั้งหมดที่อยู่ใต้ผู้จัดการคนหนึ่งด้วย recursive CTE
- ป้องกัน infinite loop ที่เกิดจากข้อมูลเป็นวงจร (cycle) ด้วยการเก็บ path เป็น array และด้วย native `CYCLE` clause
- เข้าใจความแตกต่างระหว่าง `UNION ALL` กับ `UNION` ใน recursive CTE และผลกระทบต่อการหยุด recursion
- เข้าใจข้อจำกัดด้าน performance ของ recursive CTE กับข้อมูลจำนวนมาก และรู้จักทางเลือกอื่น เช่น ltree extension
- สร้างรายงาน org chart แบบเต็มรูปแบบ และ breadcrumb หมวดหมู่สินค้าแบบใช้งานจริง

---

## เตรียมข้อมูล

บทนี้ใช้ชุดข้อมูลอีคอมเมิร์ซเดียวกับ Part 021-039 ทั้งหมด แต่เนื่องจากบทนี้เจาะจงเรื่อง recursive CTE บนข้อมูลแบบลำดับชั้น (hierarchy) เราจึงออกแบบข้อมูลใน `categories` ให้มีลำดับชั้นลึกอย่างน้อย 4 ระดับ และข้อมูลใน `employees` ให้มีสายบังคับบัญชาลึกอย่างน้อย 5 ระดับ เพื่อให้เห็นภาพการ recursion ได้ชัดเจน

รันสคริปต์ทั้งหมดนี้ในฐานข้อมูลว่างเปล่า (หรือ schema ใหม่) ก่อนเริ่มทำตามตัวอย่างในบทนี้

```sql
-- ==============================================
-- โครงสร้างตาราง (Schema) — ใช้ร่วมกันทุกบทใน Part 021-039
-- ==============================================

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

### ข้อมูลตัวอย่าง — categories (โครงสร้างหลายระดับ)

ลำดับการ INSERT สำคัญมาก เพราะ parent ต้องถูกสร้างก่อน child เสมอ (foreign key อ้างถึงตัวเอง)

```sql
-- ระดับ 1: หมวดหมู่ราก (root category, parent_category_id = NULL)
INSERT INTO categories (category_name, parent_category_id) VALUES ('Electronics', NULL);       -- id = 1
-- ระดับ 2: ลูกของ Electronics
INSERT INTO categories (category_name, parent_category_id) VALUES ('Computers', 1);             -- id = 2
-- ระดับ 3: ลูกของ Computers
INSERT INTO categories (category_name, parent_category_id) VALUES ('Laptops', 2);               -- id = 3
-- ระดับ 4: ลูกของ Laptops
INSERT INTO categories (category_name, parent_category_id) VALUES ('Gaming Laptops', 3);        -- id = 4
INSERT INTO categories (category_name, parent_category_id) VALUES ('Ultrabooks', 3);             -- id = 5
-- ระดับ 3 อีกสาขา: ลูกของ Computers
INSERT INTO categories (category_name, parent_category_id) VALUES ('Desktops', 2);              -- id = 6
-- ระดับ 2 อีกสาขา: ลูกของ Electronics
INSERT INTO categories (category_name, parent_category_id) VALUES ('Smartphones', 1);           -- id = 7
INSERT INTO categories (category_name, parent_category_id) VALUES ('Accessories', 1);           -- id = 8
-- ระดับ 3: ลูกของ Accessories
INSERT INTO categories (category_name, parent_category_id) VALUES ('Phone Cases', 8);           -- id = 9
INSERT INTO categories (category_name, parent_category_id) VALUES ('Chargers & Cables', 8);      -- id = 10

-- ระดับ 1: หมวดหมู่รากอีกต้นหนึ่ง
INSERT INTO categories (category_name, parent_category_id) VALUES ('Home & Kitchen', NULL);     -- id = 11
INSERT INTO categories (category_name, parent_category_id) VALUES ('Furniture', 11);            -- id = 12
INSERT INTO categories (category_name, parent_category_id) VALUES ('Office Furniture', 12);     -- id = 13

-- ระดับ 1: หมวดหมู่รากที่สาม
INSERT INTO categories (category_name, parent_category_id) VALUES ('Books', NULL);              -- id = 14
INSERT INTO categories (category_name, parent_category_id) VALUES ('Fiction', 14);              -- id = 15
INSERT INTO categories (category_name, parent_category_id) VALUES ('Non-Fiction', 14);           -- id = 16
```

โครงสร้าง `categories` ที่ได้จะเป็นต้นไม้ (tree) ดังนี้:

```text
Electronics (1)
├── Computers (2)
│   ├── Laptops (3)
│   │   ├── Gaming Laptops (4)     <- ลึก 4 ระดับ
│   │   └── Ultrabooks (5)         <- ลึก 4 ระดับ
│   └── Desktops (6)
├── Smartphones (7)
└── Accessories (8)
    ├── Phone Cases (9)
    └── Chargers & Cables (10)

Home & Kitchen (11)
└── Furniture (12)
    └── Office Furniture (13)

Books (14)
├── Fiction (15)
└── Non-Fiction (16)
```

### ข้อมูลตัวอย่าง — suppliers, products

```sql
INSERT INTO suppliers (supplier_name, country) VALUES
    ('TechSource Co., Ltd.', 'Thailand'),
    ('Global Gadgets Inc.', 'USA'),
    ('OfficeWorld Supply', 'Thailand'),
    ('Nordic Devices AB', 'Sweden'),
    ('AccessPlus Trading', 'China');

INSERT INTO products (product_name, category_id, supplier_id, unit_price, stock_quantity) VALUES
    ('ThinkPro Gaming X15', 4, 1, 45900.00, 12),
    ('ThinkPro Gaming X17', 4, 1, 59900.00, 8),
    ('AirBook Ultra 14', 5, 2, 42500.00, 20),
    ('AirBook Ultra 13', 5, 2, 38900.00, 15),
    ('PowerTower Desktop i7', 6, 3, 32000.00, 10),
    ('PowerTower Desktop i9', 6, 3, 45000.00, 6),
    ('SmartX Phone 12', 7, 4, 21900.00, 30),
    ('SmartX Phone 12 Pro', 7, 4, 32900.00, 18),
    ('Leather Case SmartX12', 9, 5, 590.00, 100),
    ('Silicone Case Universal', 9, 5, 290.00, 200),
    ('USB-C Fast Charger 65W', 10, 5, 890.00, 150),
    ('USB-C to USB-C Cable 2m', 10, 5, 350.00, 300),
    ('Ergonomic Office Chair', 13, 3, 6900.00, 25),
    ('Standing Desk 140cm', 13, 3, 12500.00, 15),
    ('The Silent Forest (Novel)', 15, 2, 350.00, 60),
    ('Time and Memory (Novel)', 15, 2, 420.00, 40),
    ('PostgreSQL for Professionals', 16, 1, 890.00, 35),
    ('Data Structures Explained', 16, 1, 750.00, 25);
```

### ข้อมูลตัวอย่าง — customers

```sql
INSERT INTO customers (first_name, last_name, email, country, signup_date) VALUES
    ('Araya', 'Suksawat', 'araya.s@email.com', 'Thailand', '2023-01-15'),
    ('Ben', 'Anderson', 'ben.a@email.com', 'USA', '2023-02-20'),
    ('Chalida', 'Wong', 'chalida.w@email.com', 'Thailand', '2023-03-05'),
    ('David', 'Kim', 'david.kim@email.com', 'South Korea', '2023-04-10'),
    ('Emma', 'Wilson', 'emma.w@email.com', 'UK', '2023-05-01'),
    ('Farid', 'Rahman', 'farid.r@email.com', 'Malaysia', '2023-06-18'),
    ('Gita', 'Suryani', 'gita.s@email.com', 'Indonesia', '2023-07-22'),
    ('Hiroshi', 'Tanaka', 'hiroshi.t@email.com', 'Japan', '2023-08-30'),
    ('Isara', 'Boonchu', 'isara.b@email.com', 'Thailand', '2023-09-14'),
    ('Julia', 'Martinez', 'julia.m@email.com', 'Spain', '2023-10-02');
```

### ข้อมูลตัวอย่าง — employees (สายบังคับบัญชาหลายระดับ)

เช่นเดียวกับ `categories` ตาราง `employees` ก็เป็น self-referencing table ต้อง INSERT ผู้บังคับบัญชาก่อนลูกน้องเสมอ

```sql
-- ระดับ 1: CEO (manager_id = NULL คือรากของสายบังคับบัญชา)
INSERT INTO employees (first_name, last_name, hire_date, manager_id, department) VALUES
    ('Somchai', 'Rattanakul', '2015-01-10', NULL, 'Executive');                       -- id = 1

-- ระดับ 2: รายงานตรงต่อ CEO
INSERT INTO employees (first_name, last_name, hire_date, manager_id, department) VALUES
    ('Suda', 'Charoensuk', '2016-03-01', 1, 'Sales'),                                 -- id = 2, VP Sales
    ('Anan', 'Phongsathorn', '2016-05-15', 1, 'Engineering');                         -- id = 3, VP Engineering

-- ระดับ 3: รายงานตรงต่อ VP
INSERT INTO employees (first_name, last_name, hire_date, manager_id, department) VALUES
    ('Piti', 'Wongsawang', '2017-02-01', 2, 'Sales'),                                 -- id = 4, Sales Manager (North)
    ('Kanokwan', 'Srisuwan', '2017-06-10', 2, 'Sales'),                               -- id = 5, Sales Manager (South)
    ('Malee', 'Boonmee', '2017-04-20', 3, 'Engineering'),                             -- id = 6, Engineering Manager (Backend)
    ('Wichai', 'Thanakit', '2017-08-05', 3, 'Engineering');                           -- id = 7, Engineering Manager (Frontend)

-- ระดับ 4: รายงานตรงต่อ Manager
INSERT INTO employees (first_name, last_name, hire_date, manager_id, department) VALUES
    ('Kanya', 'Sukjai', '2018-01-15', 4, 'Sales'),                                    -- id = 8
    ('Niran', 'Petcharat', '2018-03-20', 4, 'Sales'),                                 -- id = 9
    ('Somsak', 'Jaidee', '2018-05-01', 6, 'Engineering'),                             -- id = 10
    ('Ploy', 'Napasorn', '2018-07-11', 6, 'Engineering'),                             -- id = 11
    ('Apinya', 'Meesuk', '2018-09-01', 7, 'Engineering');                             -- id = 12

-- ระดับ 5: รายงานตรงต่อพนักงานระดับ 4 (ลึกที่สุดในองค์กร)
INSERT INTO employees (first_name, last_name, hire_date, manager_id, department) VALUES
    ('Chatchai', 'Ruangroj', '2019-02-14', 12, 'Engineering'),                        -- id = 13
    ('Siriporn', 'Kaewkla', '2019-04-01', 5, 'Sales');                                -- id = 14
```

สายบังคับบัญชาที่ได้:

```text
Somchai Rattanakul (CEO) (1)
├── Suda Charoensuk (VP Sales) (2)
│   ├── Piti Wongsawang (Sales Manager North) (4)
│   │   ├── Kanya Sukjai (8)
│   │   └── Niran Petcharat (9)
│   └── Kanokwan Srisuwan (Sales Manager South) (5)
│       └── Siriporn Kaewkla (14)
└── Anan Phongsathorn (VP Engineering) (3)
    ├── Malee Boonmee (Engineering Manager Backend) (6)
    │   ├── Somsak Jaidee (10)
    │   └── Ploy Napasorn (11)
    └── Wichai Thanakit (Engineering Manager Frontend) (7)
        └── Apinya Meesuk (12)
            └── Chatchai Ruangroj (13)     <- ลึก 5 ระดับ
```

### ข้อมูลตัวอย่าง — orders, order_items, payments, reviews

```sql
INSERT INTO orders (customer_id, employee_id, order_date, status, ship_country) VALUES
    (1, 8,  '2024-01-05 10:15:00+07', 'completed', 'Thailand'),      -- order_id = 1
    (2, 9,  '2024-01-10 14:30:00+07', 'completed', 'USA'),           -- order_id = 2
    (3, 8,  '2024-01-15 09:00:00+07', 'completed', 'Thailand'),      -- order_id = 3
    (4, 14, '2024-01-20 11:45:00+07', 'completed', 'South Korea'),   -- order_id = 4
    (5, 9,  '2024-02-01 16:20:00+07', 'shipped',   'UK'),            -- order_id = 5
    (6, 8,  '2024-02-05 08:10:00+07', 'completed', 'Malaysia'),      -- order_id = 6
    (7, 14, '2024-02-10 13:00:00+07', 'pending',   'Indonesia'),     -- order_id = 7
    (8, 9,  '2024-02-15 15:40:00+07', 'completed', 'Japan'),         -- order_id = 8
    (9, 8,  '2024-02-20 10:05:00+07', 'completed', 'Thailand'),      -- order_id = 9
    (10, 14,'2024-03-01 12:00:00+07', 'cancelled', 'Spain'),         -- order_id = 10
    (1, 9,  '2024-03-05 09:30:00+07', 'completed', 'Thailand'),      -- order_id = 11
    (3, 8,  '2024-03-10 17:00:00+07', 'shipped',   'Thailand');      -- order_id = 12

INSERT INTO order_items (order_id, product_id, quantity, unit_price) VALUES
    (1, 1, 1, 45900.00),
    (2, 7, 2, 21900.00),
    (3, 9, 3, 590.00), (3, 11, 1, 890.00),
    (4, 3, 1, 42500.00),
    (5, 13, 2, 6900.00),
    (6, 10, 5, 290.00),
    (7, 15, 4, 350.00),
    (8, 8, 1, 32900.00), (8, 9, 1, 590.00),
    (9, 17, 2, 890.00),
    (10, 14, 1, 12500.00),
    (11, 5, 1, 32000.00),
    (12, 2, 1, 59900.00);

INSERT INTO payments (order_id, amount, payment_method) VALUES
    (1, 45900.00, 'credit_card'),
    (2, 43800.00, 'paypal'),
    (3, 2660.00, 'credit_card'),
    (4, 42500.00, 'bank_transfer'),
    (5, 13800.00, 'credit_card'),
    (6, 1450.00, 'paypal'),
    (8, 33490.00, 'credit_card'),
    (9, 1780.00, 'promptpay'),
    (11, 32000.00, 'credit_card'),
    (12, 59900.00, 'bank_transfer');

INSERT INTO reviews (product_id, customer_id, rating, review_text) VALUES
    (1, 1, 5, 'สินค้าดีมาก ทำงานลื่นสุดๆ สำหรับเล่นเกม'),
    (7, 2, 4, 'Good phone, camera could be better'),
    (9, 3, 5, 'เคสสวย ใส่พอดีเป๊ะ'),
    (3, 4, 5, 'Excellent build quality'),
    (13, 5, 4, 'Comfortable chair, worth the price'),
    (17, 9, 5, 'หนังสือ PostgreSQL อ่านเข้าใจง่ายมาก');
```

ตอนนี้เรามีข้อมูลครบพร้อมแล้ว ทั้ง `categories` ที่ลึก 4 ระดับ และ `employees` ที่ลึก 5 ระดับ — เหมาะสำหรับฝึก recursive CTE ในทุกรูปแบบ มาเริ่มกันเลย

---

## Step 251: Recursive CTE คืออะไร

Recursive CTE (Common Table Expression แบบเวียนซ้ำ) คือ CTE พิเศษที่สามารถ**อ้างอิงถึงตัวเองได้**ภายใน query เดียวกัน เขียนด้วยคำสั่ง `WITH RECURSIVE` ใช้สำหรับ query ข้อมูลที่มีโครงสร้างแบบลำดับชั้น (hierarchy) หรือแบบกราฟ (graph) ที่ไม่รู้ความลึกล่วงหน้า เช่น:

- โครงสร้างหมวดหมู่สินค้าที่ซ้อนกันได้ไม่จำกัดชั้น
- สายบังคับบัญชาในองค์กร (org chart)
- เส้นทางการเดินทางระหว่างเมือง (graph traversal)
- Bill of Materials (BOM) — ชิ้นส่วนประกอบของชิ้นส่วนอีกที

### โครงสร้างของ WITH RECURSIVE

```sql
WITH RECURSIVE cte_name (column1, column2, ...) AS (
    -- 1) Anchor member (non-recursive term)
    --    ทำหน้าที่เป็นจุดเริ่มต้น รันแค่ครั้งเดียว
    SELECT ...
    FROM some_table
    WHERE ...

    UNION ALL   -- ใช้ UNION ALL เกือบทุกครั้ง (ดูรายละเอียดใน Step 258)

    -- 2) Recursive member (recursive term)
    --    อ้างอิงถึง cte_name (ตัวเอง) — จะวนซ้ำจนกว่าจะไม่มีแถวใหม่เกิดขึ้น
    SELECT ...
    FROM some_table AS t
    JOIN cte_name ON t.parent_column = cte_name.child_column
)
SELECT * FROM cte_name;
```

### กลไกการทำงาน (execution model)

PostgreSQL ประมวลผล recursive CTE เหมือนมี "working table" ชั่วคราวอยู่เบื้องหลัง ทำงานเป็นขั้นตอนดังนี้:

1. รัน **anchor member** หนึ่งครั้ง ผลลัพธ์ทั้งหมดถูกใส่ทั้งใน **working table** และ **result set**
2. รัน **recursive member** โดยใช้ข้อมูลใน working table ปัจจุบันเป็นตัวแทนของ `cte_name`
3. ผลลัพธ์ใหม่ที่ได้จาก recursive member จะแทนที่ working table เดิม และถูกเพิ่มเข้า result set
4. ทำซ้ำขั้นตอน 2-3 ไปเรื่อยๆ **จนกว่า recursive member จะไม่คืนแถวใดๆ เลย** (working table ว่าง) จึงหยุด
5. ผลลัพธ์สุดท้ายคือทุกแถวที่สะสมมาจากทุกรอบ

ข้อสำคัญที่ต้องจำ: recursive member **ห้าม**ใช้ aggregate function (เช่น `SUM`, `COUNT` บน CTE เอง), `DISTINCT`, `ORDER BY`, `LIMIT` หรือ `GROUP BY` ที่อ้างอิงตัว CTE เอง และในหนึ่ง query สามารถมี recursive member ได้แค่ตัวเดียวเท่านั้น (ห้ามมี `UNION ALL` หลายชั้นที่ recursive term อ้างตัวเองซ้ำซ้อน)

> **หมายเหตุ:** คำว่า `RECURSIVE` ใน `WITH RECURSIVE` จริงๆ แล้วมีผลกับ CTE **ทุกตัว**ใน `WITH` clause นั้น ไม่ใช่แค่ตัวที่ recursive เท่านั้น — ถ้ามีหลาย CTE ใน `WITH` เดียวกันและมีอย่างน้อยหนึ่งตัวเป็น recursive ต้องใส่คำว่า `RECURSIVE` ไว้ครั้งเดียวหน้าสุด

---

## Step 252: ตัวอย่างพื้นฐานที่สุด — นับเลข 1 ถึง N

ก่อนเข้าโจทย์จริงเรื่อง hierarchy มาดูตัวอย่างที่เรียบง่ายที่สุดก่อน เพื่อให้เห็นกลไกการวนซ้ำชัดๆ โดยไม่มีความซับซ้อนของ JOIN มาบดบัง

```sql
WITH RECURSIVE counter AS (
    -- Anchor member: เริ่มต้นที่ 1
    SELECT 1 AS n

    UNION ALL

    -- Recursive member: บวกเพิ่มทีละ 1 จนกว่า n จะถึง 10
    SELECT n + 1
    FROM counter
    WHERE n < 10
)
SELECT n FROM counter;
```

ผลลัพธ์:

```text
 n
----
  1
  2
  3
  4
  5
  6
  7
  8
  9
 10
(10 rows)
```

### ไล่ทีละรอบให้เห็นภาพ

| รอบ | working table (input) | recursive member คำนวณ | working table ใหม่ (output รอบนี้) |
|-----|------------------------|--------------------------|--------------------------------------|
| เริ่มต้น (anchor) | — | `SELECT 1` | `{1}` |
| รอบ 1 | `{1}` | `1+1=2` (เพราะ 1 < 10) | `{2}` |
| รอบ 2 | `{2}` | `2+1=3` (เพราะ 2 < 10) | `{3}` |
| ... | ... | ... | ... |
| รอบ 9 | `{9}` | `9+1=10` (เพราะ 9 < 10) | `{10}` |
| รอบ 10 | `{10}` | ไม่มีแถว (เพราะ 10 ไม่ < 10) | `{}` (ว่าง → หยุด) |

รวมทุกรอบ (`{1} + {2} + {3} + ... + {10}`) ได้ผลลัพธ์ 1 ถึง 10 ตามที่เห็น

**ข้อควรระวังสำคัญที่สุด:** เงื่อนไข `WHERE n < 10` ใน recursive member คือ "ตัวหยุด recursion" ถ้าลืมใส่เงื่อนไขนี้ (หรือเงื่อนไขไม่มีทางเป็นเท็จ) query จะวนซ้ำไม่มีที่สิ้นสุด (infinite loop) จนกว่าจะชนขีดจำกัด `work_mem` หรือถูก cancel ควรทดสอบด้วยตัวเลขน้อยๆ ก่อนเสมอ หรือใส่ `LIMIT` ครอบ query ด้านนอกไว้กันเหนียวระหว่างพัฒนา:

```sql
-- เทคนิคกันเหนียวระหว่างพัฒนา: ครอบด้วย LIMIT ที่ query ชั้นนอก
WITH RECURSIVE counter AS (
    SELECT 1 AS n
    UNION ALL
    SELECT n + 1 FROM counter   -- สมมุติลืมใส่ WHERE (อันตราย!)
)
SELECT n FROM counter LIMIT 20;   -- LIMIT ช่วยตัดตอนไม่ให้วนไม่รู้จบ
```

การนับเลขแบบนี้ปกติเราใช้ `generate_series(1, 10)` ซึ่งเร็วกว่ามาก แต่ตัวอย่างนี้มีไว้เพื่อ**ให้เห็นกลไก**ของ recursive CTE ล้วนๆ ก่อนนำไปใช้กับข้อมูลจริงที่ไม่มีฟังก์ชันสำเร็จรูปมาช่วยได้

---

## Step 253: Recursive CTE บน categories — ไล่ลูกหมวดหมู่ทั้งหมด (Top-Down)

โจทย์: เริ่มจาก `Electronics` (category_id = 1) ต้องการหาหมวดหมู่ลูกทั้งหมด **ทุกระดับ** ไม่ใช่แค่ลูกชั้นแรก

```sql
WITH RECURSIVE sub_categories AS (
    -- Anchor: เริ่มจาก category ที่เราสนใจ
    SELECT category_id, category_name, parent_category_id
    FROM categories
    WHERE category_id = 1   -- Electronics

    UNION ALL

    -- Recursive: หา category ที่มี parent_category_id ตรงกับ category
    -- ที่เพิ่งถูกดึงเข้ามาในรอบก่อนหน้า
    SELECT c.category_id, c.category_name, c.parent_category_id
    FROM categories c
    JOIN sub_categories sc ON c.parent_category_id = sc.category_id
)
SELECT category_id, category_name, parent_category_id
FROM sub_categories
ORDER BY category_id;
```

ผลลัพธ์:

```text
 category_id |   category_name   | parent_category_id
-------------+--------------------+---------------------
           1 | Electronics        |
           2 | Computers          |                   1
           3 | Laptops            |                   2
           4 | Gaming Laptops     |                   3
           5 | Ultrabooks         |                   3
           6 | Desktops           |                   2
           7 | Smartphones        |                   1
           8 | Accessories        |                   1
           9 | Phone Cases        |                   8
          10 | Chargers & Cables  |                   8
(10 rows)
```

สังเกตว่าเราได้หมวดหมู่ครบ**ทั้ง 4 ระดับ** (Electronics → Computers → Laptops → Gaming Laptops/Ultrabooks) ทั้งที่ query ไม่รู้ล่วงหน้าเลยว่าโครงสร้างลึกกี่ชั้น — นี่คือจุดแข็งของ recursive CTE เมื่อเทียบกับการ `JOIN` ตัวเองซ้ำๆ ด้วยมือ (self-join หลายชั้นซึ่งต้องรู้ความลึกตายตัวล่วงหน้า)

ถ้าต้องการเฉพาะ**ลูกหมวดหมู่จริงๆ** โดยไม่รวมตัวเอง (Electronics) ให้ตัดแถว anchor ออกที่ query ชั้นนอก:

```sql
WITH RECURSIVE sub_categories AS (
    SELECT category_id, category_name, parent_category_id
    FROM categories
    WHERE category_id = 1
    UNION ALL
    SELECT c.category_id, c.category_name, c.parent_category_id
    FROM categories c
    JOIN sub_categories sc ON c.parent_category_id = sc.category_id
)
SELECT category_id, category_name, parent_category_id
FROM sub_categories
WHERE category_id <> 1
ORDER BY category_id;
```

การประยุกต์ใช้จริงที่พบบ่อยที่สุดคือ "หาสินค้าทั้งหมดในหมวดหมู่นี้ รวมหมวดหมู่ย่อยทุกชั้น" เช่น ต้องการดูสินค้าทั้งหมดใน Electronics แม้สินค้านั้นจะถูกจัดอยู่ใน Gaming Laptops (ชั้นลึกสุด) ก็ตาม:

```sql
WITH RECURSIVE sub_categories AS (
    SELECT category_id FROM categories WHERE category_id = 1
    UNION ALL
    SELECT c.category_id
    FROM categories c
    JOIN sub_categories sc ON c.parent_category_id = sc.category_id
)
SELECT p.product_id, p.product_name, p.unit_price, c.category_name
FROM products p
JOIN categories c ON p.category_id = c.category_id
WHERE p.category_id IN (SELECT category_id FROM sub_categories)
ORDER BY p.product_id;
```

ผลลัพธ์:

```text
 product_id |       product_name        | unit_price |  category_name
------------+----------------------------+------------+------------------
          1 | ThinkPro Gaming X15        |   45900.00 | Gaming Laptops
          2 | ThinkPro Gaming X17        |   59900.00 | Gaming Laptops
          3 | AirBook Ultra 14           |   42500.00 | Ultrabooks
          4 | AirBook Ultra 13           |   38900.00 | Ultrabooks
          5 | PowerTower Desktop i7      |   32000.00 | Desktops
          6 | PowerTower Desktop i9      |   45000.00 | Desktops
          7 | SmartX Phone 12            |   21900.00 | Smartphones
          8 | SmartX Phone 12 Pro        |   32900.00 | Smartphones
          9 | Leather Case SmartX12      |     590.00 | Phone Cases
         10 | Silicone Case Universal    |     290.00 | Phone Cases
         11 | USB-C Fast Charger 65W     |     890.00 | Chargers & Cables
         12 | USB-C to USB-C Cable 2m    |     350.00 | Chargers & Cables
(12 rows)
```

สินค้าทุกชิ้นภายใต้ Electronics ถูกดึงมาครบ ไม่ว่าจะอยู่ลึกกี่ชั้นก็ตาม — เขียนแบบนี้เพียงครั้งเดียว ใช้ได้กับหมวดหมู่ไหนก็ได้ ไม่ต้องแก้ query แม้จะมีการเพิ่มชั้นย่อยเข้ามาในอนาคต

---

## Step 254: Recursive CTE แบบ Bottom-Up — หา Path ขึ้นไปหา Root Category

โจทย์ตรงข้ามกับ Step 253: เมื่อรู้ category_id ของสินค้าชิ้นหนึ่ง (เช่น `Gaming Laptops` id = 4) ต้องการไล่**ขึ้น**ไปหา root เพื่อรู้ว่า "สินค้านี้อยู่ในสายหมวดหมู่อะไรบ้าง" — สลับทิศทางของ JOIN condition จาก Step 253

```sql
WITH RECURSIVE category_path AS (
    -- Anchor: เริ่มจาก category ที่เราสนใจ
    SELECT category_id, category_name, parent_category_id
    FROM categories
    WHERE category_id = 4   -- Gaming Laptops

    UNION ALL

    -- Recursive: หา parent ของ category ที่เพิ่งถูกดึงเข้ามา
    -- (สังเกตว่า JOIN สลับด้านกับ Step 253)
    SELECT c.category_id, c.category_name, c.parent_category_id
    FROM categories c
    JOIN category_path cp ON c.category_id = cp.parent_category_id
)
SELECT category_id, category_name, parent_category_id
FROM category_path;
```

ผลลัพธ์:

```text
 category_id |  category_name  | parent_category_id
-------------+------------------+---------------------
           4 | Gaming Laptops   |                   3
           3 | Laptops          |                   2
           2 | Computers        |                   1
           1 | Electronics      |
(4 rows)
```

จะเห็นว่าผลลัพธ์เรียงจาก**ตัวเองไปหา root** ตามลำดับ: Gaming Laptops → Laptops → Computers → Electronics

ความแตกต่างของ JOIN condition ระหว่าง Top-Down กับ Bottom-Up คือหัวใจของบทนี้ สรุปเปรียบเทียบ:

| ทิศทาง | JOIN condition | ความหมาย |
|--------|-----------------|----------|
| Top-Down (Step 253) | `c.parent_category_id = sc.category_id` | หา "ลูก" ของแถวที่เพิ่งได้มา |
| Bottom-Up (Step 254) | `c.category_id = cp.parent_category_id` | หา "พ่อแม่" ของแถวที่เพิ่งได้มา |

เขียนเป็นฟังก์ชันใช้ซ้ำได้ (function) เพื่อเรียกหา root ของ category ใดๆ ก็ได้:

```sql
CREATE OR REPLACE FUNCTION get_root_category(p_category_id INTEGER)
RETURNS TABLE(root_id INTEGER, root_name VARCHAR) AS $$
    WITH RECURSIVE category_path AS (
        SELECT category_id, category_name, parent_category_id
        FROM categories
        WHERE category_id = p_category_id
        UNION ALL
        SELECT c.category_id, c.category_name, c.parent_category_id
        FROM categories c
        JOIN category_path cp ON c.category_id = cp.parent_category_id
    )
    SELECT category_id, category_name
    FROM category_path
    WHERE parent_category_id IS NULL;
$$ LANGUAGE sql STABLE;
```

```sql
SELECT * FROM get_root_category(4);   -- Gaming Laptops
```

```text
 root_id | root_name
---------+-------------
       1 | Electronics
(1 row)
```

---

## Step 255: เพิ่มคอลัมน์ Level/Depth และ Path String

เพื่อให้แสดงผลลำดับชั้นแบบอ่านง่าย เราสามารถเพิ่มคอลัมน์นับระดับ (`level`) และคอลัมน์ที่เก็บ path แบบข้อความ (breadcrumb) ไว้ในตัว CTE เองได้เลย โดยส่งค่าต่อกันไปเรื่อยๆ ในแต่ละรอบของการ recursion

```sql
WITH RECURSIVE category_tree AS (
    -- Anchor: เริ่มจากทุก root category (parent_category_id IS NULL)
    SELECT
        category_id,
        category_name,
        parent_category_id,
        1 AS level,
        category_name::TEXT AS path
    FROM categories
    WHERE parent_category_id IS NULL

    UNION ALL

    -- Recursive: level เพิ่มขึ้นทีละ 1, path ต่อท้ายด้วย ' > '
    SELECT
        c.category_id,
        c.category_name,
        c.parent_category_id,
        ct.level + 1,
        ct.path || ' > ' || c.category_name
    FROM categories c
    JOIN category_tree ct ON c.parent_category_id = ct.category_id
)
SELECT
    category_id,
    repeat('  ', level - 1) || category_name AS tree_display,
    level,
    path
FROM category_tree
ORDER BY path;
```

ผลลัพธ์ (แสดงเป็นทรีที่เยื้องตามระดับ — indented tree):

```text
 category_id |          tree_display          | level |                       path
-------------+---------------------------------+-------+----------------------------------------------------
          14 | Books                           |     1 | Books
          15 |   Fiction                       |     2 | Books > Fiction
          16 |   Non-Fiction                   |     2 | Books > Non-Fiction
           1 | Electronics                     |     1 | Electronics
           8 |   Accessories                   |     2 | Electronics > Accessories
          10 |     Chargers & Cables           |     3 | Electronics > Accessories > Chargers & Cables
           9 |     Phone Cases                 |     3 | Electronics > Accessories > Phone Cases
           2 |   Computers                     |     2 | Electronics > Computers
           6 |     Desktops                    |     3 | Electronics > Computers > Desktops
           3 |     Laptops                     |     3 | Electronics > Computers > Laptops
           4 |       Gaming Laptops            |     4 | Electronics > Computers > Laptops > Gaming Laptops
           5 |       Ultrabooks                |     4 | Electronics > Computers > Laptops > Ultrabooks
           7 |   Smartphones                   |     2 | Electronics > Smartphones
          11 | Home & Kitchen                  |     1 | Home & Kitchen
          12 |   Furniture                     |     2 | Home & Kitchen > Furniture
          13 |     Office Furniture            |     3 | Home & Kitchen > Furniture > Office Furniture
(16 rows)
```

การ `ORDER BY path` ทำให้แถวในสาขาเดียวกันเรียงอยู่ติดกันโดยธรรมชาติ (เพราะ path เป็น prefix ของ path ลูกเสมอ) ทำให้ได้ผลลัพธ์ที่หน้าตาเหมือนต้นไม้จริงๆ โดยไม่ต้องเขียน logic การจัดเรียงเพิ่มเติม

> **เทคนิคขั้นสูง:** ตั้งแต่ PostgreSQL 14 เป็นต้นมา มี clause `SEARCH DEPTH FIRST BY ... SET ...` ที่ช่วยสร้างคอลัมน์ลำดับการเดิน (ordering column) ให้อัตโนมัติโดยไม่ต้องพึ่ง path string เช่น:
>
> ```sql
> WITH RECURSIVE category_tree AS (
>     SELECT category_id, category_name, parent_category_id, 1 AS level
>     FROM categories
>     WHERE parent_category_id IS NULL
>     UNION ALL
>     SELECT c.category_id, c.category_name, c.parent_category_id, ct.level + 1
>     FROM categories c
>     JOIN category_tree ct ON c.parent_category_id = ct.category_id
> )
> SEARCH DEPTH FIRST BY category_id SET ordercol
> SELECT repeat('  ', level - 1) || category_name AS tree_display, level
> FROM category_tree
> ORDER BY ordercol;
> ```
>
> วิธีนี้ให้ผลลัพธ์เป็น depth-first tree เหมือนกัน แต่ไม่ต้องคำนวณ string ต่อกันเอง เหมาะกับกรณีที่ไม่ต้องการ path string แสดงผล เพียงต้องการลำดับการแสดงผลที่ถูกต้อง

การมีคอลัมน์ `path` แบบนี้ยังมีประโยชน์อีกอย่างคือใช้ทำ **breadcrumb navigation** บนหน้าเว็บอีคอมเมิร์ซได้ทันที เช่น "Electronics > Computers > Laptops > Gaming Laptops" ที่มักเห็นด้านบนของหน้าสินค้า

---

## Step 256: Recursive CTE บน employees — Org Chart ทั้งหมดใต้ผู้จัดการคนหนึ่ง

หลักการเดียวกับ Step 253-255 แต่เปลี่ยนจาก `categories` เป็น `employees` โจทย์: ต้องการดูพนักงานทั้งหมดที่อยู่ใต้ `Anan Phongsathorn` (VP Engineering, employee_id = 3) ไม่ว่าจะอยู่กี่ระดับก็ตาม

```sql
WITH RECURSIVE org_chart AS (
    -- Anchor: เริ่มจากผู้จัดการที่สนใจ
    SELECT
        employee_id,
        first_name,
        last_name,
        manager_id,
        department,
        1 AS level
    FROM employees
    WHERE employee_id = 3   -- Anan Phongsathorn, VP Engineering

    UNION ALL

    -- Recursive: หาพนักงานที่ manager_id ตรงกับ employee_id ที่เพิ่งดึงมา
    SELECT
        e.employee_id,
        e.first_name,
        e.last_name,
        e.manager_id,
        e.department,
        oc.level + 1
    FROM employees e
    JOIN org_chart oc ON e.manager_id = oc.employee_id
)
SELECT
    employee_id,
    repeat('    ', level - 1) || first_name || ' ' || last_name AS org_tree,
    department,
    level
FROM org_chart
ORDER BY level, employee_id;
```

ผลลัพธ์:

```text
 employee_id |             org_tree              | department  | level
-------------+------------------------------------+-------------+-------
           3 | Anan Phongsathorn                  | Engineering |     1
           6 |     Malee Boonmee                  | Engineering |     2
           7 |     Wichai Thanakit                | Engineering |     2
          10 |         Somsak Jaidee              | Engineering |     3
          11 |         Ploy Napasorn              | Engineering |     3
          12 |         Apinya Meesuk              | Engineering |     3
          13 |             Chatchai Ruangroj      | Engineering |     4
(7 rows)
```

จะเห็นว่า `Chatchai Ruangroj` ซึ่งอยู่ลึกถึงระดับ 5 ในองค์กร (นับจาก CEO) ก็ถูกดึงมาด้วย เพราะเขาคือลูกน้องของ `Apinya Meesuk` ซึ่งเป็นลูกน้องของ `Wichai Thanakit` ซึ่งอยู่ใต้ `Anan` — recursive CTE ไล่ตามสายจนสุดโดยอัตโนมัติ

**การใช้งานจริง:** นับจำนวนพนักงานทั้งหมด (รวมทางอ้อม) ที่อยู่ใต้แต่ละผู้จัดการ:

```sql
WITH RECURSIVE org_chart AS (
    SELECT employee_id, manager_id, employee_id AS root_manager
    FROM employees
    WHERE manager_id IS NULL

    UNION ALL

    SELECT e.employee_id, e.manager_id, oc.root_manager
    FROM employees e
    JOIN org_chart oc ON e.manager_id = oc.employee_id
)
-- ตัวอย่างนี้หา root_manager ของทุกคนใต้ CEO คนเดียว
-- สำหรับนับใต้ผู้จัดการทุกคนพร้อมกัน ดูตัวอย่างแบบเต็มในแบบฝึกหัดข้อ 6/9
SELECT COUNT(*) AS total_under_ceo
FROM org_chart
WHERE root_manager = 1 AND employee_id <> 1;
```

```text
 total_under_ceo
------------------
               13
```

ถูกต้อง เพราะพนักงานทั้งหมดมี 14 คน ลบ CEO เอง 1 คน เหลือ 13 คนที่อยู่ใต้สายบังคับบัญชาของ CEO ทั้งหมด (โดยตรงหรือโดยอ้อม)

---

## Step 257: การป้องกัน Infinite Loop กรณีข้อมูลเป็นวงจร (Cycle)

ในทางทฤษฎี ตาราง `employees` และ `categories` ที่เราออกแบบไว้ไม่ควรมี cycle เพราะโครงสร้างเป็น tree (ทุกแถวมี parent ได้แค่หนึ่งตัว) แต่ **ในทางปฏิบัติ ข้อมูลจริงอาจมี bug** เช่น พนักงาน A ถูกตั้งให้เป็น manager ของ B และมีคนไปแก้ B ให้กลายเป็น manager ของ A กลับ (data entry ผิดพลาด) กรณีแบบนี้ recursive CTE ธรรมดาจะ**วนซ้ำไม่รู้จบ** เพราะ A → B → A → B → ... ไม่มีวันถึงเงื่อนไขหยุด

### จำลองสถานการณ์ cycle

```sql
-- สมมุติมี bug: ตั้งให้ CEO (id=1) มี manager เป็น Chatchai (id=13)
-- ซึ่งจริงๆ แล้ว Chatchai อยู่ใต้สายของ CEO อยู่แล้ว (1 -> 3 -> 7 -> 12 -> 13)
-- ถ้าอัปเดตแบบนี้จะเกิด cycle: 1 -> 13 -> 12 -> 7 -> 3 -> 1 -> 13 -> ...
-- (ตัวอย่างนี้เพื่อสาธิตเท่านั้น ไม่ต้องรันจริงถ้าไม่ต้องการทำลายข้อมูล anchor)
-- UPDATE employees SET manager_id = 13 WHERE employee_id = 1;
```

ถ้ารัน recursive CTE ปกติกับข้อมูลที่มี cycle แบบนี้ query จะไม่มีวันหยุดเอง

### วิธีที่ 1: เก็บ path เป็น array แล้วเช็กก่อนวนซ้ำ

เทคนิคมาตรฐานคือเก็บรายการ `employee_id` ทั้งหมดที่เดินผ่านมาแล้วไว้ใน array แล้วในแต่ละรอบเช็กว่า "แถวถัดไปเคยเดินผ่านมาแล้วหรือยัง" ถ้าเคยแล้วให้หยุดไม่ไปต่อ (ตัด branch นั้นทิ้ง)

```sql
WITH RECURSIVE org_chart AS (
    SELECT
        employee_id,
        first_name,
        last_name,
        manager_id,
        1 AS level,
        ARRAY[employee_id] AS visited_path,
        false AS is_cycle
    FROM employees
    WHERE employee_id = 1

    UNION ALL

    SELECT
        e.employee_id,
        e.first_name,
        e.last_name,
        e.manager_id,
        oc.level + 1,
        oc.visited_path || e.employee_id,
        e.employee_id = ANY(oc.visited_path)   -- ถ้าเคยเดินผ่าน id นี้แล้ว = cycle
    FROM employees e
    JOIN org_chart oc ON e.manager_id = oc.employee_id
    WHERE NOT (e.employee_id = ANY(oc.visited_path))   -- เงื่อนไขหยุดไม่ให้วนซ้ำ
)
SELECT employee_id, first_name, last_name, level, visited_path
FROM org_chart
ORDER BY level, employee_id;
```

เงื่อนไข `WHERE NOT (e.employee_id = ANY(oc.visited_path))` คือหัวใจของเทคนิคนี้ — มันคือตัวป้องกัน ถ้า `employee_id` ถัดไปเคยอยู่ใน path ที่เดินผ่านมาแล้ว จะไม่ join แถวนั้นเข้ามาอีก ทำให้ recursion หยุดได้เสมอไม่ว่าข้อมูลจะมี cycle หรือไม่

### วิธีที่ 2: native CYCLE clause (PostgreSQL 14 ขึ้นไป)

ตั้งแต่ PostgreSQL 14 มี syntax เฉพาะสำหรับจัดการ cycle โดยไม่ต้องเขียน array logic เอง ทำให้ query สั้นและอ่านง่ายขึ้นมาก:

```sql
WITH RECURSIVE org_chart AS (
    SELECT employee_id, first_name, last_name, manager_id, 1 AS level
    FROM employees
    WHERE employee_id = 1

    UNION ALL

    SELECT e.employee_id, e.first_name, e.last_name, e.manager_id, oc.level + 1
    FROM employees e
    JOIN org_chart oc ON e.manager_id = oc.employee_id
)
CYCLE employee_id SET is_cycle USING visited_path
SELECT employee_id, first_name, last_name, level, is_cycle
FROM org_chart
ORDER BY level, employee_id;
```

- `CYCLE employee_id` บอกว่าให้ตรวจ cycle จากคอลัมน์ `employee_id`
- `SET is_cycle` คือคอลัมน์ boolean ใหม่ที่ระบบสร้างให้อัตโนมัติ เป็น `true` เมื่อพบว่าแถวนี้ทำให้เกิด cycle
- `USING visited_path` คือคอลัมน์ array ที่ระบบสร้างให้อัตโนมัติเพื่อเก็บ path (คล้ายกับ `visited_path` ที่เราทำมือใน วิธีที่ 1)

เมื่อ PostgreSQL ตรวจพบว่าแถวถัดไปจะทำให้เกิด cycle มันจะ**หยุดขยาย branch นั้นโดยอัตโนมัติ** (เหมือนมี `WHERE NOT (... = ANY(...))` แฝงอยู่ในตัว) ทำให้ query ไม่มีวันวนไม่รู้จบ พร้อมทั้งให้ข้อมูลกลับมาด้วยว่าแถวไหนคือจุดที่เกิด cycle ผ่านคอลัมน์ `is_cycle`

**สรุปเปรียบเทียบสองวิธี:**

| ประเด็น | Array + WHERE ด้วยมือ | Native CYCLE clause |
|---------|------------------------|------------------------|
| ความยาวโค้ด | ยาวกว่า ต้องเขียน column เพิ่มเอง | สั้นกว่า ระบบจัดการให้ |
| ความยืดหยุ่น | ปรับ logic เองได้เต็มที่ (เช่น เช็กจากหลายคอลัมน์) | ตรวจจาก column เดียวหรือชุดคอลัมน์ที่ระบุ |
| เวอร์ชันที่รองรับ | ทุกเวอร์ชันที่มี `WITH RECURSIVE` | PostgreSQL 14 ขึ้นไปเท่านั้น |
| ความชัดเจนของ intent | ต้องอ่านโค้ดถึงเข้าใจว่าป้องกัน cycle | ชัดเจนในตัว syntax เลยว่ากำลังป้องกัน cycle |

สำหรับโปรเจกต์ใหม่บน PostgreSQL 14+ (รวมถึง 16/17 ที่ใช้ในคอร์สนี้) แนะนำให้ใช้ **native `CYCLE` clause** เป็นค่าเริ่มต้น เพราะอ่านง่ายกว่าและมีโอกาสเขียนผิดน้อยกว่า ส่วนวิธี array เหมาะกับกรณีที่ต้องรองรับ PostgreSQL เวอร์ชันเก่า หรือ logic การตรวจ cycle ซับซ้อนกว่าปกติ

> **แนวคิดสำคัญ:** แม้ข้อมูลของเราตอนนี้จะไม่มี cycle จริง แต่การใส่กลไกป้องกันไว้ตั้งแต่ต้นเป็น **defensive programming ที่ดี** โดยเฉพาะ query ที่จะถูกใช้ซ้ำในระบบ production ที่ข้อมูลอาจถูกแก้ไขผิดพลาดได้ทุกเมื่อ

---

## Step 258: UNION ALL vs UNION ใน Recursive CTE

ปกติเราจะเห็น recursive CTE ใช้ `UNION ALL` เกือบทุกครั้ง แต่จริงๆ แล้วสามารถใช้ `UNION` (ที่ตัดข้อมูลซ้ำออก) ได้เช่นกัน ความแตกต่างส่งผลโดยตรงต่อพฤติกรรมการหยุด recursion

### UNION ALL — ค่าเริ่มต้นที่แนะนำ

```sql
WITH RECURSIVE category_tree AS (
    SELECT category_id, category_name, parent_category_id, 1 AS level
    FROM categories
    WHERE category_id = 1

    UNION ALL   -- ไม่ตัดข้อมูลซ้ำ เร็วกว่า

    SELECT c.category_id, c.category_name, c.parent_category_id, ct.level + 1
    FROM categories c
    JOIN category_tree ct ON c.parent_category_id = ct.category_id
)
SELECT * FROM category_tree ORDER BY level, category_id;
```

`UNION ALL` **ไม่เปรียบเทียบแถวใหม่กับแถวเก่า** เพียงแค่เติมแถวใหม่เข้าไปเรื่อยๆ เท่านั้น การหยุด recursion ต้องพึ่งเงื่อนไข `JOIN`/`WHERE` ในตัว query เอง (เช่นในตัวอย่างนี้คือธรรมชาติของข้อมูลที่ parent_category_id จะไม่มีวันวนกลับมาหาตัวเอง) ข้อดีคือ**เร็วกว่ามาก** เพราะไม่ต้องเสียเวลาเปรียบเทียบ/deduplicate ทุกรอบ

### UNION — ตัดข้อมูลซ้ำอัตโนมัติ แต่ช้ากว่า

```sql
WITH RECURSIVE category_tree AS (
    SELECT category_id, category_name, parent_category_id, 1 AS level
    FROM categories
    WHERE category_id = 1

    UNION   -- ตัดแถวที่ซ้ำกันทุกคอลัมน์ทิ้งโดยอัตโนมัติ

    SELECT c.category_id, c.category_name, c.parent_category_id, ct.level + 1
    FROM categories c
    JOIN category_tree ct ON c.parent_category_id = ct.category_id
)
SELECT * FROM category_tree ORDER BY level, category_id;
```

`UNION` จะเปรียบเทียบแถวใหม่ที่ recursive member สร้างขึ้นกับ**ทุกแถวที่เคยเกิดขึ้นมาแล้วในทุกรอบ** (ไม่ใช่แค่รอบก่อนหน้า) ถ้าแถวใหม่ซ้ำกับแถวเก่าทุกคอลัมน์ ระบบจะไม่เพิ่มแถวนั้นเข้า working table ของรอบถัดไป ผลคือ**ถ้าข้อมูลวนกลับมาเจอแถวเดิมเป๊ะ (ทุกคอลัมน์เหมือนกัน) recursion จะหยุดเองได้** แม้ผู้เขียนจะไม่ได้ใส่เงื่อนไขป้องกัน cycle ไว้เลย

### เปรียบเทียบพฤติกรรมเมื่อมี cycle

สมมุติมี cycle ในข้อมูล (เช่นตัวอย่าง Step 257) และคอลัมน์ที่ select ออกมา **ไม่มี `level`** (คอลัมน์ที่เปลี่ยนค่าทุกรอบ):

| กรณี | UNION ALL | UNION |
|------|-----------|-------|
| ไม่มีคอลัมน์ที่เปลี่ยนค่าทุกรอบ (เช่น `level`) และมี cycle | วนไม่รู้จบ (infinite loop) | หยุดได้เอง เพราะรอบถัดไปจะได้แถวที่ซ้ำกับรอบก่อนหน้าทุกคอลัมน์ |
| มีคอลัมน์ที่เปลี่ยนค่าทุกรอบ (เช่น `level`, `path`) และมี cycle | วนไม่รู้จบ | **ก็ยังวนไม่รู้จบเช่นกัน** เพราะ `level` เปลี่ยนค่าทุกรอบ ทำให้แถวไม่มีวันซ้ำกันแบบทุกคอลัมน์เป๊ะ |

นี่คือจุดสำคัญที่มักเข้าใจผิด: **`UNION` ไม่ใช่เครื่องมือป้องกัน cycle ที่เชื่อถือได้** เพราะในโลกจริงเรามักจะมีคอลัมน์อย่าง `level` หรือ `path` ติดอยู่เสมอ (เพื่อแสดงผลลำดับชั้น) ซึ่งทำให้แถวไม่ซ้ำกันแบบทุกคอลัมน์แม้ข้อมูลจะวนซ้ำที่ `employee_id`/`category_id` เดิมก็ตาม ดังนั้นการป้องกัน cycle ที่เชื่อถือได้จริงคือ **array + WHERE (Step 257 วิธีที่ 1)** หรือ **native `CYCLE` clause (Step 257 วิธีที่ 2)** เท่านั้น ไม่ใช่การเปลี่ยนจาก `UNION ALL` เป็น `UNION`

**ข้อสรุปเชิงปฏิบัติ:**

- ใช้ `UNION ALL` เป็นค่าเริ่มต้นเสมอ เพราะเร็วกว่า และเราจะจัดการเรื่อง cycle ด้วยเทคนิคเฉพาะทาง (Step 257) อยู่แล้ว
- ใช้ `UNION` เฉพาะกรณีที่ต้องการดึงข้อมูลแบบ "ไม่สนใจ path ที่มาถึง สนใจแค่ปลายทางที่ไม่ซ้ำกัน" เช่น graph traversal ที่ query แค่ node id เดียว ไม่มี level/path ติดมาด้วย และต้องการให้ node ที่ถูกเจอซ้ำถูกตัดออกอัตโนมัติ (ใช้บ่อยใน graph algorithm ทั่วไป เช่น หา "โหนดทั้งหมดที่เข้าถึงได้จากจุดเริ่มต้น" โดยไม่สนใจเส้นทาง)

---

## Step 259: ข้อจำกัดด้าน Performance และทางเลือกอื่น

### ข้อจำกัดของ recursive CTE

Recursive CTE เป็นเครื่องมือที่ทรงพลังและอ่านง่าย แต่มีข้อจำกัดด้าน performance ที่ควรรู้เมื่อข้อมูลมีขนาดใหญ่หรือลึกมาก:

1. **ประมวลผลทีละแถว/ทีละรอบ (iterative, row-at-a-time)** — recursive CTE ไม่สามารถใช้ประโยชน์จาก set-based optimization ได้เต็มที่เหมือน query ปกติ แต่ละรอบต้อง JOIN กับ working table ของรอบก่อนหน้า ทำให้จำนวนรอบเท่ากับความลึกของ tree เสมอ ถ้า tree ลึกมาก (เช่น หลักพันชั้น) จะมีรอบการ scan เยอะตามไปด้วย

2. **Query planner มองเห็น recursive term ได้จำกัด** — planner ไม่สามารถประเมิน cost ของทั้ง query ล่วงหน้าได้แม่นยำเหมือน query ทั่วไป เพราะไม่รู้ว่าจะวนกี่รอบ ทำให้บางครั้งเลือก execution plan ที่ไม่เหมาะสมที่สุด

3. **ไม่รองรับ parallel query** — recursive CTE (ณ PostgreSQL 16/17) **ไม่สามารถรันแบบขนาน (parallel)** ได้ ต่างจาก query ธรรมดาที่ PostgreSQL อาจแบ่งงานให้ worker หลายตัวช่วยกันทำ

4. **ต้องมี index บน foreign key ที่ใช้ JOIN เสมอ** — ถ้า `parent_category_id` หรือ `manager_id` ไม่มี index recursive CTE จะทำ sequential scan ซ้ำๆ ในทุกรอบ ยิ่ง tree ลึก ยิ่งช้าแบบทวีคูณ

```sql
-- index ที่ควรมีเสมอสำหรับตารางที่ใช้ recursive CTE บ่อย
CREATE INDEX idx_categories_parent ON categories(parent_category_id);
CREATE INDEX idx_employees_manager ON employees(manager_id);
```

5. **การใช้ `EXPLAIN ANALYZE` เพื่อดู performance:**

```sql
EXPLAIN ANALYZE
WITH RECURSIVE org_chart AS (
    SELECT employee_id, manager_id, 1 AS level
    FROM employees WHERE employee_id = 1
    UNION ALL
    SELECT e.employee_id, e.manager_id, oc.level + 1
    FROM employees e
    JOIN org_chart oc ON e.manager_id = oc.employee_id
)
SELECT * FROM org_chart;
```

จะเห็น node ชื่อ `CTE Scan on org_chart` และ `WorkTable Scan on org_chart` ในแผนการทำงาน ซึ่งบ่งบอกว่า PostgreSQL กำลังวน scan working table ซ้ำๆ ตามจำนวนรอบของ recursion — สำหรับข้อมูลขนาดเล็กแบบในคอร์สนี้ไม่มีปัญหา แต่ถ้า tree มีเป็นแสนแถวและลึกหลายสิบชั้น เวลาที่ใช้จะเพิ่มขึ้นอย่างมีนัยสำคัญ

### ทางเลือกอื่นสำหรับข้อมูล hierarchy ขนาดใหญ่

เมื่อ recursive CTE เริ่มช้าเกินไป มีทางเลือกอื่นให้พิจารณา:

**1. Materialized path column** — เก็บ path เต็มไว้เป็นคอลัมน์ text เลย (เช่น `'1.2.3.4'`) อัปเดตเมื่อมีการย้าย node แลกกับการ query เร็วขึ้นมาก (ใช้ `LIKE 'prefix%'` แทนการ recursion) แต่ต้องเขียน trigger คอยรักษาความถูกต้องของ path เวลาข้อมูลเปลี่ยน

**2. Closure table (Transitive closure table)** — สร้างตารางแยกเก็บความสัมพันธ์ ancestor-descendant ทุกคู่ที่เป็นไปได้ล่วงหน้า (เช่น `category_closure(ancestor_id, descendant_id, depth)`) ทำให้ query หาลูกหลานทั้งหมดกลายเป็น `SELECT` ธรรมดาไม่ต้อง recursive เลย แลกกับพื้นที่เก็บข้อมูลที่มากขึ้นและความซับซ้อนตอน insert/update/delete

**3. ltree extension** — PostgreSQL มี extension ชื่อ `ltree` ที่ออกแบบมาสำหรับเก็บข้อมูล tree/hierarchy โดยเฉพาะ รองรับการค้นหาแบบ ancestor/descendant ได้เร็วมากด้วย GiST/GIN index เพราะเก็บ path เป็น data type พิเศษที่ query ได้โดยตรง ไม่ต้อง recursive CTE เลย

```sql
-- ตัวอย่างแนวคิดคร่าวๆ (ไม่ใช่ schema หลักของคอร์สนี้)
CREATE EXTENSION IF NOT EXISTS ltree;

CREATE TABLE category_ltree (
    category_id INTEGER PRIMARY KEY,
    category_name VARCHAR(100),
    path LTREE   -- เช่น 'electronics.computers.laptops.gaming_laptops'
);

CREATE INDEX idx_category_path ON category_ltree USING GIST (path);

-- หาลูกหลานทั้งหมดของ electronics.computers ด้วย operator ของ ltree โดยตรง
-- SELECT * FROM category_ltree WHERE path <@ 'electronics.computers';
```

`ltree` เหมาะมากเมื่อ tree มีขนาดใหญ่มาก (หลายแสน-ล้านแถว) และ query แบบ ancestor/descendant เป็น workload หลักของระบบ แต่ต้องแลกกับความซับซ้อนในการดูแลรักษา path string ให้ตรงกับโครงสร้างจริงเสมอ (มักทำผ่าน trigger หรือ application logic)

**สรุปแนวทางเลือกใช้เครื่องมือ:**

| สถานการณ์ | แนะนำใช้ |
|-----------|----------|
| Tree ขนาดเล็ก-กลาง (ไม่กี่พันแถว), ต้องการความยืดหยุ่น, query หลากหลายรูปแบบ | Recursive CTE (adjacency list) |
| Tree ขนาดใหญ่, อ่านบ่อยกว่าเขียนมาก, ต้องการ query เร็วสุด | Materialized path หรือ ltree |
| ต้องการ query ความสัมพันธ์ ancestor-descendant บ่อยมาก และมีพื้นที่เก็บข้อมูลเหลือเฟือ | Closure table |
| งาน one-off/รายงานที่ไม่ได้รันบ่อย | Recursive CTE เพียงพอเสมอ ไม่ต้องเพิ่มความซับซ้อน |

สำหรับคอร์สนี้และงานส่วนใหญ่ในระดับองค์กรทั่วไป (หมวดหมู่สินค้าไม่กี่ร้อยหมวด, org chart ไม่กี่พันคน) **recursive CTE เพียงพอและเหมาะสมที่สุดแล้ว** — ทางเลือกอื่นควรพิจารณาเมื่อวัด performance จริงแล้วพบว่าเป็นคอขวดเท่านั้น (อย่า optimize ก่อนที่จะรู้ว่ามีปัญหาจริง)

---

## Step 260: แบบฝึกหัดรวม — Org Chart แบบเต็มรูปแบบ และ Breadcrumb หมวดหมู่สินค้า

มาประกอบทุกเทคนิคที่เรียนมาในบทนี้เข้าด้วยกัน เพื่อสร้างรายงานที่ใช้งานได้จริง 2 รายงาน

### รายงานที่ 1: Full Org Chart พร้อมจำนวนลูกน้องทั้งหมด (รวมทางอ้อม)

```sql
WITH RECURSIVE org_chart AS (
    SELECT
        employee_id,
        first_name,
        last_name,
        manager_id,
        department,
        hire_date,
        1 AS level,
        first_name || ' ' || last_name AS breadcrumb,
        ARRAY[employee_id] AS visited_path
    FROM employees
    WHERE manager_id IS NULL   -- เริ่มจาก CEO (root ของทั้งองค์กร)

    UNION ALL

    SELECT
        e.employee_id,
        e.first_name,
        e.last_name,
        e.manager_id,
        e.department,
        e.hire_date,
        oc.level + 1,
        oc.breadcrumb || ' > ' || e.first_name || ' ' || e.last_name,
        oc.visited_path || e.employee_id
    FROM employees e
    JOIN org_chart oc ON e.manager_id = oc.employee_id
    WHERE NOT (e.employee_id = ANY(oc.visited_path))
),
subordinate_counts AS (
    -- นับจำนวนลูกน้องทั้งหมด (ทุกระดับ) ของแต่ละคน
    SELECT oc_top.employee_id, COUNT(oc_sub.employee_id) AS total_subordinates
    FROM org_chart oc_top
    LEFT JOIN org_chart oc_sub
        ON oc_sub.breadcrumb LIKE oc_top.breadcrumb || ' > %'
    GROUP BY oc_top.employee_id
)
SELECT
    oc.employee_id,
    repeat('    ', oc.level - 1) || oc.first_name || ' ' || oc.last_name AS org_tree,
    oc.department,
    oc.level,
    sc.total_subordinates,
    oc.hire_date
FROM org_chart oc
JOIN subordinate_counts sc ON sc.employee_id = oc.employee_id
ORDER BY oc.breadcrumb;
```

ผลลัพธ์:

```text
 employee_id |              org_tree               | department  | level | total_subordinates | hire_date
-------------+--------------------------------------+-------------+-------+---------------------+------------
           1 | Somchai Rattanakul                   | Executive   |     1 |                  13 | 2015-01-10
           3 |     Anan Phongsathorn                | Engineering |     2 |                   6 | 2016-05-15
           6 |         Malee Boonmee                | Engineering |     3 |                   2 | 2017-04-20
          10 |             Somsak Jaidee            | Engineering |     4 |                   0 | 2018-05-01
          11 |             Ploy Napasorn            | Engineering |     4 |                   0 | 2018-07-11
           7 |         Wichai Thanakit              | Engineering |     3 |                   2 | 2017-08-05
          12 |             Apinya Meesuk            | Engineering |     4 |                   1 | 2018-09-01
          13 |                 Chatchai Ruangroj    | Engineering |     5 |                   0 | 2019-02-14
           2 |     Suda Charoensuk                  | Sales       |     2 |                   6 | 2016-03-01
           4 |         Piti Wongsawang              | Sales       |     3 |                   2 | 2017-02-01
           8 |             Kanya Sukjai             | Sales       |     4 |                   0 | 2018-01-15
           9 |             Niran Petcharat          | Sales       |     4 |                   0 | 2018-03-20
           5 |         Kanokwan Srisuwan            | Sales       |     3 |                   1 | 2017-06-10
          14 |             Siriporn Kaewkla         | Sales       |     4 |                   0 | 2019-04-01
(14 rows)
```

รายงานนี้ให้ทั้งโครงสร้างองค์กรแบบต้นไม้ (`org_tree`) และจำนวนลูกน้องทั้งหมดของแต่ละคน (`total_subordinates`) ในคำสั่งเดียว — เห็นชัดว่า CEO มีคนอยู่ใต้บังคับบัญชา 13 คน, VP ทั้งสองคนมีคนละ 6 คน สอดคล้องกับโครงสร้างที่เราออกแบบไว้

### รายงานที่ 2: Breadcrumb หมวดหมู่สินค้าเต็มรูปแบบ พร้อมข้อมูลสินค้า

```sql
WITH RECURSIVE category_tree AS (
    SELECT
        category_id,
        category_name,
        parent_category_id,
        1 AS level,
        category_name::TEXT AS breadcrumb
    FROM categories
    WHERE parent_category_id IS NULL

    UNION ALL

    SELECT
        c.category_id,
        c.category_name,
        c.parent_category_id,
        ct.level + 1,
        ct.breadcrumb || ' > ' || c.category_name
    FROM categories c
    JOIN category_tree ct ON c.parent_category_id = ct.category_id
)
SELECT
    p.product_id,
    p.product_name,
    ct.breadcrumb AS category_breadcrumb,
    ct.level AS category_depth,
    p.unit_price,
    p.stock_quantity
FROM products p
JOIN category_tree ct ON p.category_id = ct.category_id
ORDER BY ct.breadcrumb, p.product_name;
```

ผลลัพธ์:

```text
 product_id |       product_name        |               category_breadcrumb                | category_depth | unit_price | stock_quantity
------------+----------------------------+----------------------------------------------------+-----------------+------------+-----------------
         17 | PostgreSQL for Professionals | Books > Non-Fiction                              |               2 |     890.00 |              35
         18 | Data Structures Explained | Books > Non-Fiction                                 |               2 |     750.00 |              25
         15 | The Silent Forest (Novel) | Books > Fiction                                     |               2 |     350.00 |              60
         16 | Time and Memory (Novel)   | Books > Fiction                                     |               2 |     420.00 |              40
         11 | USB-C Fast Charger 65W   | Electronics > Accessories > Chargers & Cables        |               3 |     890.00 |             150
         12 | USB-C to USB-C Cable 2m  | Electronics > Accessories > Chargers & Cables        |               3 |     350.00 |             300
          9 | Leather Case SmartX12    | Electronics > Accessories > Phone Cases              |               3 |     590.00 |             100
         10 | Silicone Case Universal  | Electronics > Accessories > Phone Cases              |               3 |     290.00 |             200
          5 | PowerTower Desktop i7    | Electronics > Computers > Desktops                   |               3 |   32000.00 |              10
          6 | PowerTower Desktop i9    | Electronics > Computers > Desktops                   |               3 |   45000.00 |               6
          1 | ThinkPro Gaming X15      | Electronics > Computers > Laptops > Gaming Laptops   |               4 |   45900.00 |              12
          2 | ThinkPro Gaming X17      | Electronics > Computers > Laptops > Gaming Laptops   |               4 |   59900.00 |               8
          3 | AirBook Ultra 14         | Electronics > Computers > Laptops > Ultrabooks       |               4 |   42500.00 |              20
          4 | AirBook Ultra 13         | Electronics > Computers > Laptops > Ultrabooks       |               4 |   38900.00 |              15
          7 | SmartX Phone 12          | Electronics > Smartphones                            |               2 |   21900.00 |              30
          8 | SmartX Phone 12 Pro      | Electronics > Smartphones                            |               2 |   32900.00 |              18
         13 | Ergonomic Office Chair   | Home & Kitchen > Furniture > Office Furniture         |               3 |    6900.00 |              25
         14 | Standing Desk 140cm      | Home & Kitchen > Furniture > Office Furniture         |               3 |   12500.00 |              15
(18 rows)
```

รายงานนี้แสดง breadcrumb เต็มรูปแบบของทุกสินค้า ไม่ว่าสินค้าจะอยู่ลึกกี่ชั้นก็ตาม — พร้อมใช้งานจริงบนหน้าเว็บอีคอมเมิร์ซ หรือใน export รายงานสำหรับทีมจัดซื้อ/สต๊อกได้ทันที

---

## สรุปท้ายบท

- **Recursive CTE** เขียนด้วย `WITH RECURSIVE` ประกอบด้วย **anchor member** (จุดเริ่มต้น รันครั้งเดียว) และ **recursive member** (อ้างอิงตัวเอง วนซ้ำจนไม่มีแถวใหม่) เชื่อมกันด้วย `UNION ALL` เป็นหลัก
- ตัวอย่างการนับเลข 1-10 ช่วยให้เห็นกลไก working table ที่ถูกแทนที่ทุกรอบและสะสมผลลัพธ์ไปเรื่อยๆ จนกว่าเงื่อนไขหยุดจะเป็นจริง
- **Top-down** (หา descendants) ใช้ JOIN condition `child.parent_id = cte.id` ส่วน **bottom-up** (หา ancestors) ใช้ `child.id = cte.parent_id` — สลับทิศทางเดียวกันแต่ตอบคำถามคนละแบบ
- เพิ่มคอลัมน์ `level` และ `path` (breadcrumb) เข้าไปใน recursive member ได้โดยตรง ทำให้ได้รายงานที่แสดงลำดับชั้นแบบอ่านง่ายในคำสั่งเดียว และ `ORDER BY path` ทำให้ผลลัพธ์เรียงเป็นต้นไม้อัตโนมัติ
- ใช้หลักการเดียวกันนี้ได้ทั้งกับ `categories` (หมวดหมู่สินค้า) และ `employees` (org chart) เพราะทั้งคู่เป็น self-referencing table ที่มีโครงสร้างแบบ adjacency list เหมือนกัน
- ข้อมูลที่เป็นวงจร (cycle) ทำให้ recursive CTE วนไม่รู้จบได้ ป้องกันได้ด้วยการเก็บ path เป็น array แล้วเช็กก่อนวนซ้ำ หรือใช้ native `CYCLE ... SET ... USING ...` clause (PostgreSQL 14+) ที่กระชับกว่า
- `UNION ALL` เร็วกว่าและเป็นค่าเริ่มต้นที่ควรใช้เสมอ ส่วน `UNION` ไม่ใช่เครื่องมือป้องกัน cycle ที่เชื่อถือได้เมื่อมีคอลัมน์อย่าง `level`/`path` ติดอยู่ในผลลัพธ์
- Recursive CTE มีข้อจำกัดด้าน performance กับข้อมูลขนาดใหญ่มาก (ไม่รองรับ parallel query, ต้องมี index บน foreign key เสมอ) ทางเลือกอื่นได้แก่ materialized path, closure table และ `ltree` extension สำหรับกรณีที่ต้องการความเร็วสูงสุดกับ tree ขนาดใหญ่
- แบบฝึกหัดรวมแสดงให้เห็นว่าเทคนิคทั้งหมดในบทนี้ประกอบกันเป็นรายงานที่ใช้งานได้จริง ทั้ง org chart พร้อมจำนวนลูกน้อง และ breadcrumb หมวดหมู่สินค้าแบบเต็มรูปแบบ

Recursive CTE คือเครื่องมือที่ทรงพลังที่สุดตัวหนึ่งของ SQL สำหรับข้อมูลแบบลำดับชั้น เมื่อเข้าใจกลไก anchor/recursive member และรู้จักป้องกัน cycle อย่างถูกต้องแล้ว จะสามารถนำไปประยุกต์ใช้กับข้อมูล hierarchy รูปแบบใดก็ได้ ไม่ว่าจะเป็นหมวดหมู่สินค้า องค์กร Bill of Materials หรือแม้แต่กราฟเส้นทางที่ซับซ้อน

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1

เขียน recursive CTE เพื่อนับเลขถอยหลังจาก 10 ถึง 1 (ผลลัพธ์คือ 10, 9, 8, ..., 1)

<details>
<summary>เฉลย</summary>

```sql
WITH RECURSIVE countdown AS (
    SELECT 10 AS n
    UNION ALL
    SELECT n - 1
    FROM countdown
    WHERE n > 1
)
SELECT n FROM countdown;
```

ผลลัพธ์:

```text
 n
----
 10
  9
  8
  7
  6
  5
  4
  3
  2
  1
(10 rows)
```

</details>

### แบบฝึกหัดที่ 2

หาหมวดหมู่ลูกทั้งหมด (ทุกระดับ) ของ `Computers` (category_id = 2) พร้อมคอลัมน์ level

<details>
<summary>เฉลย</summary>

```sql
WITH RECURSIVE sub_categories AS (
    SELECT category_id, category_name, parent_category_id, 1 AS level
    FROM categories
    WHERE category_id = 2

    UNION ALL

    SELECT c.category_id, c.category_name, c.parent_category_id, sc.level + 1
    FROM categories c
    JOIN sub_categories sc ON c.parent_category_id = sc.category_id
)
SELECT category_id, category_name, level
FROM sub_categories
ORDER BY level, category_id;
```

ผลลัพธ์:

```text
 category_id |  category_name  | level
-------------+------------------+-------
           2 | Computers        |     1
           3 | Laptops          |     2
           6 | Desktops         |     2
           4 | Gaming Laptops   |     3
           5 | Ultrabooks       |     3
(5 rows)
```

</details>

### แบบฝึกหัดที่ 3

หา root category ของสินค้าแต่ละชิ้นในตาราง `products` (join สินค้ากับ recursive path ที่ไล่ขึ้นไปจนถึง root)

<details>
<summary>เฉลย</summary>

```sql
WITH RECURSIVE category_ancestors AS (
    SELECT category_id, category_name, parent_category_id, category_id AS start_id
    FROM categories

    UNION ALL

    SELECT c.category_id, c.category_name, c.parent_category_id, ca.start_id
    FROM categories c
    JOIN category_ancestors ca ON c.category_id = ca.parent_category_id
)
SELECT
    p.product_id,
    p.product_name,
    ca.category_name AS root_category
FROM products p
JOIN category_ancestors ca
    ON ca.start_id = p.category_id AND ca.parent_category_id IS NULL
ORDER BY p.product_id;
```

ผลลัพธ์ (บางส่วน):

```text
 product_id |    product_name       | root_category
------------+------------------------+----------------
          1 | ThinkPro Gaming X15    | Electronics
          2 | ThinkPro Gaming X17    | Electronics
         13 | Ergonomic Office Chair | Home & Kitchen
         15 | The Silent Forest (Novel) | Books
...
(18 rows)
```

หมายเหตุ: เทคนิคนี้เริ่ม anchor จาก**ทุกแถว**ใน `categories` (ไม่ระบุ WHERE) แล้วใช้คอลัมน์ `start_id` เป็นตัวจำว่าแต่ละ path เริ่มจาก category ไหน เป็นรูปแบบที่มีประโยชน์เมื่อต้องการคำนวณ ancestor ของทุกโหนดพร้อมกันในคำสั่งเดียว

</details>

### แบบฝึกหัดที่ 4

หาพนักงานทั้งหมดที่อยู่ใต้ `Suda Charoensuk` (VP Sales, employee_id = 2) รวมทุกระดับ

<details>
<summary>เฉลย</summary>

```sql
WITH RECURSIVE org_chart AS (
    SELECT employee_id, first_name, last_name, manager_id, 1 AS level
    FROM employees
    WHERE employee_id = 2

    UNION ALL

    SELECT e.employee_id, e.first_name, e.last_name, e.manager_id, oc.level + 1
    FROM employees e
    JOIN org_chart oc ON e.manager_id = oc.employee_id
)
SELECT employee_id, first_name, last_name, level
FROM org_chart
WHERE employee_id <> 2
ORDER BY level, employee_id;
```

ผลลัพธ์:

```text
 employee_id | first_name |  last_name  | level
-------------+-------------+-------------+-------
           4 | Piti        | Wongsawang  |     2
           5 | Kanokwan    | Srisuwan    |     2
           8 | Kanya       | Sukjai      |     3
           9 | Niran       | Petcharat   |     3
          14 | Siriporn    | Kaewkla     |     3
(5 rows)
```

</details>

### แบบฝึกหัดที่ 5

สร้าง breadcrumb string (path) สำหรับทุกหมวดหมู่ในตาราง `categories` โดยใช้ ' / ' เป็นตัวคั่นแทน ' > '

<details>
<summary>เฉลย</summary>

```sql
WITH RECURSIVE category_tree AS (
    SELECT category_id, category_name, parent_category_id,
           category_name::TEXT AS breadcrumb
    FROM categories
    WHERE parent_category_id IS NULL

    UNION ALL

    SELECT c.category_id, c.category_name, c.parent_category_id,
           ct.breadcrumb || ' / ' || c.category_name
    FROM categories c
    JOIN category_tree ct ON c.parent_category_id = ct.category_id
)
SELECT category_id, breadcrumb
FROM category_tree
ORDER BY breadcrumb;
```

ผลลัพธ์ (บางส่วน):

```text
 category_id |                     breadcrumb
-------------+------------------------------------------------------
          14 | Books
          15 | Books / Fiction
          16 | Books / Non-Fiction
           1 | Electronics
           4 | Electronics / Computers / Laptops / Gaming Laptops
...
(16 rows)
```

</details>

### แบบฝึกหัดที่ 6

นับจำนวนพนักงานใต้บังคับบัญชา (รวมทางอ้อม) ของ**ทุกผู้จัดการ**พร้อมกันในคำสั่งเดียว (ไม่ใช่แค่คนใดคนหนึ่ง)

<details>
<summary>เฉลย</summary>

```sql
WITH RECURSIVE org_pairs AS (
    -- anchor: ทุกคนคือ "ลูกน้องของตัวเอง" ที่ level 0 (ใช้เป็นจุดเริ่มไล่ขึ้น)
    SELECT employee_id AS subordinate_id, employee_id AS manager_ancestor_id
    FROM employees

    UNION ALL

    -- recursive: ไล่ขึ้นไปหา manager ของ manager_ancestor_id ปัจจุบันเรื่อยๆ
    SELECT op.subordinate_id, e.manager_id
    FROM org_pairs op
    JOIN employees e ON e.employee_id = op.manager_ancestor_id
    WHERE e.manager_id IS NOT NULL
)
SELECT
    m.employee_id,
    m.first_name || ' ' || m.last_name AS manager_name,
    COUNT(*) AS total_subordinates
FROM org_pairs op
JOIN employees m ON m.employee_id = op.manager_ancestor_id
WHERE op.subordinate_id <> op.manager_ancestor_id   -- ไม่นับตัวเอง
GROUP BY m.employee_id, m.first_name, m.last_name
ORDER BY total_subordinates DESC, m.employee_id;
```

ผลลัพธ์:

```text
 employee_id |    manager_name     | total_subordinates
-------------+----------------------+---------------------
           1 | Somchai Rattanakul   |                  13
           2 | Suda Charoensuk      |                   5
           3 | Anan Phongsathorn    |                   6
           4 | Piti Wongsawang      |                   2
           5 | Kanokwan Srisuwan    |                   1
           6 | Malee Boonmee        |                   2
           7 | Wichai Thanakit      |                   2
          12 | Apinya Meesuk        |                   1
(8 rows)
```

</details>

### แบบฝึกหัดที่ 7

หา level ที่ลึกที่สุด (max depth) ของ category tree ทั้งหมด

<details>
<summary>เฉลย</summary>

```sql
WITH RECURSIVE category_tree AS (
    SELECT category_id, parent_category_id, 1 AS level
    FROM categories
    WHERE parent_category_id IS NULL

    UNION ALL

    SELECT c.category_id, c.parent_category_id, ct.level + 1
    FROM categories c
    JOIN category_tree ct ON c.parent_category_id = ct.category_id
)
SELECT MAX(level) AS max_depth
FROM category_tree;
```

ผลลัพธ์:

```text
 max_depth
-----------
         4
(1 row)
```

สาขา `Electronics > Computers > Laptops > Gaming Laptops` (หรือ `Ultrabooks`) คือสาขาที่ลึกที่สุด อยู่ที่ 4 ระดับ

</details>

### แบบฝึกหัดที่ 8

จำลองสถานการณ์ที่ข้อมูล `employees` เป็นวงจร (cycle) แล้วเขียน query ที่ป้องกันไม่ให้เกิด infinite loop ด้วยเทคนิค array path (สมมุติว่า employee_id = 10 ถูกตั้งให้มี manager_id = 13 ซึ่งทำให้เกิด cycle: 6 → 10 → 13 → 12 → 7 → 3 → ... ไม่เกี่ยวกับ 6 โดยตรง แต่ตัวอย่างนี้ให้สมมุติ cycle แบบง่ายคือ 12 → 13 → 12)

<details>
<summary>เฉลย</summary>

```sql
-- สมมุติ (เพื่อสาธิตเท่านั้น ไม่ต้องรันจริงกับข้อมูลจริง):
-- UPDATE employees SET manager_id = 13 WHERE employee_id = 12;
-- ผลคือ 12 -> 13 -> 12 -> 13 -> ... เกิด cycle

WITH RECURSIVE org_chart AS (
    SELECT
        employee_id, first_name, last_name, manager_id,
        1 AS level,
        ARRAY[employee_id] AS visited_path
    FROM employees
    WHERE employee_id = 7   -- เริ่มจาก Wichai Thanakit

    UNION ALL

    SELECT
        e.employee_id, e.first_name, e.last_name, e.manager_id,
        oc.level + 1,
        oc.visited_path || e.employee_id
    FROM employees e
    JOIN org_chart oc ON e.manager_id = oc.employee_id
    WHERE NOT (e.employee_id = ANY(oc.visited_path))   -- ป้องกัน cycle
)
SELECT employee_id, first_name, last_name, level, visited_path
FROM org_chart
ORDER BY level;
```

เงื่อนไข `WHERE NOT (e.employee_id = ANY(oc.visited_path))` ทำให้แม้ข้อมูลจะถูกแก้ให้เกิด cycle จริง query ก็จะไม่วนซ้ำไม่รู้จบ เพราะเมื่อเจอ `employee_id` ที่เคยอยู่ใน path มาก่อน จะไม่ join แถวนั้นซ้ำอีก

หรือใช้ native clause แบบสั้นกว่า:

```sql
WITH RECURSIVE org_chart AS (
    SELECT employee_id, first_name, last_name, manager_id, 1 AS level
    FROM employees WHERE employee_id = 7
    UNION ALL
    SELECT e.employee_id, e.first_name, e.last_name, e.manager_id, oc.level + 1
    FROM employees e
    JOIN org_chart oc ON e.manager_id = oc.employee_id
)
CYCLE employee_id SET is_cycle USING visited_path
SELECT employee_id, first_name, last_name, level, is_cycle
FROM org_chart
ORDER BY level;
```

</details>

### แบบฝึกหัดที่ 9

หาผู้จัดการที่มีลูกน้อง**โดยตรง** (direct report เท่านั้น ไม่นับทางอ้อม) มากที่สุด และแสดงจำนวนลูกน้องทางอ้อมทั้งหมดของเขาประกอบด้วย (ใช้ recursive CTE สำหรับส่วนทางอ้อม)

<details>
<summary>เฉลย</summary>

```sql
WITH RECURSIVE org_chart AS (
    SELECT employee_id AS subordinate_id, employee_id AS manager_ancestor_id
    FROM employees
    UNION ALL
    SELECT op.subordinate_id, e.manager_id
    FROM org_chart op
    JOIN employees e ON e.employee_id = op.manager_ancestor_id
    WHERE e.manager_id IS NOT NULL
),
direct_reports AS (
    SELECT manager_id, COUNT(*) AS direct_count
    FROM employees
    WHERE manager_id IS NOT NULL
    GROUP BY manager_id
),
total_reports AS (
    SELECT manager_ancestor_id AS manager_id, COUNT(*) AS total_count
    FROM org_chart
    WHERE subordinate_id <> manager_ancestor_id
    GROUP BY manager_ancestor_id
)
SELECT
    m.employee_id,
    m.first_name || ' ' || m.last_name AS manager_name,
    dr.direct_count,
    tr.total_count AS total_subordinates_all_levels
FROM direct_reports dr
JOIN employees m ON m.employee_id = dr.manager_id
JOIN total_reports tr ON tr.manager_id = dr.manager_id
ORDER BY dr.direct_count DESC, tr.total_count DESC
LIMIT 1;
```

ผลลัพธ์:

```text
 employee_id |   manager_name     | direct_count | total_subordinates_all_levels
-------------+---------------------+---------------+--------------------------------
           1 | Somchai Rattanakul  |             2 |                             13
(1 row)
```

(CEO มีลูกน้องโดยตรงแค่ 2 คน (VP ทั้งสอง) แต่มีลูกน้องรวมทุกระดับ 13 คน — ถ้าต้องการดูผู้จัดการระดับกลางที่มีลูกน้องโดยตรงเยอะที่สุด ให้ลอง `WHERE m.employee_id <> 1` เพิ่มเข้าไป จะได้ Piti/Malee ที่มีลูกน้องโดยตรง 2 คนเท่ากัน)

</details>

### แบบฝึกหัดที่ 10

สร้างรายงานรวม: สำหรับแต่ละ order ที่ `status = 'completed'` ให้แสดงชื่อพนักงานที่ขาย, breadcrumb สายบังคับบัญชาของพนักงานคนนั้นจนถึง CEO, และ breadcrumb หมวดหมู่สินค้าที่ขายในออเดอร์นั้น

<details>
<summary>เฉลย</summary>

```sql
WITH RECURSIVE employee_chain AS (
    SELECT employee_id, first_name, last_name, manager_id,
           (first_name || ' ' || last_name)::TEXT AS reporting_chain
    FROM employees
    WHERE manager_id IS NULL

    UNION ALL

    SELECT e.employee_id, e.first_name, e.last_name, e.manager_id,
           ec.reporting_chain || ' > ' || e.first_name || ' ' || e.last_name
    FROM employees e
    JOIN employee_chain ec ON e.manager_id = ec.employee_id
),
category_tree AS (
    SELECT category_id, category_name, parent_category_id,
           category_name::TEXT AS category_breadcrumb
    FROM categories
    WHERE parent_category_id IS NULL

    UNION ALL

    SELECT c.category_id, c.category_name, c.parent_category_id,
           ct.category_breadcrumb || ' > ' || c.category_name
    FROM categories c
    JOIN category_tree ct ON c.parent_category_id = ct.category_id
)
SELECT DISTINCT
    o.order_id,
    ec.reporting_chain AS employee_reporting_chain,
    ct.category_breadcrumb AS product_category_breadcrumb
FROM orders o
JOIN employee_chain ec ON ec.employee_id = o.employee_id
JOIN order_items oi ON oi.order_id = o.order_id
JOIN products p ON p.product_id = oi.product_id
JOIN category_tree ct ON ct.category_id = p.category_id
WHERE o.status = 'completed'
ORDER BY o.order_id, ct.category_breadcrumb;
```

ผลลัพธ์ (บางส่วน):

```text
 order_id |     employee_reporting_chain      |              product_category_breadcrumb
----------+------------------------------------+---------------------------------------------------
        1 | Somchai Rattanakul > Suda Charoensuk > Piti Wongsawang > Kanya Sukjai | Electronics > Computers > Laptops > Gaming Laptops
        2 | Somchai Rattanakul > Suda Charoensuk > Kanokwan Srisuwan > Siriporn Kaewkla | Electronics > Smartphones
        3 | Somchai Rattanakul > Suda Charoensuk > Piti Wongsawang > Kanya Sukjai | Electronics > Accessories > Chargers & Cables
        3 | Somchai Rattanakul > Suda Charoensuk > Piti Wongsawang > Kanya Sukjai | Electronics > Accessories > Phone Cases
...
```

รายงานนี้รวมสอง recursive CTE เข้าด้วยกันในคำสั่งเดียว (`employee_chain` และ `category_tree`) แสดงให้เห็นว่าเทคนิคนี้สามารถประยุกต์ใช้กับหลายมิติของข้อมูลพร้อมกันได้อย่างเป็นธรรมชาติ

</details>

---

**บทถัดไป:** [Part 027: Aggregate Functions](./part-027-aggregate-functions.md)
