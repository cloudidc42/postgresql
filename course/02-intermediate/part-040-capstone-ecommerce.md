# Part 040: โปรเจกต์รวมระดับกลาง — ระบบร้านค้าออนไลน์ (E-Commerce Capstone)

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับกลาง | Part 040 (โปรเจกต์ปิดท้ายระดับกลาง)

## เป้าหมายการเรียนรู้

เมื่อเรียนจบบทนี้ ผู้เรียนจะสามารถ:

- สร้างสคีมาฐานข้อมูลระบบร้านค้าออนไลน์แบบเต็มรูปแบบด้วย `GENERATED ALWAYS AS IDENTITY` พร้อมอธิบายเหตุผลของทุก design decision ได้
- นำ JOIN ทุกรูปแบบ (INNER, OUTER, CROSS, SELF), subquery, CTE (ธรรมดาและ recursive), aggregate function, `GROUP BY`/`HAVING`/`ROLLUP`/`CUBE`, window function (พื้นฐานและขั้นสูง) มาผสมผสานกันแก้โจทย์ธุรกิจจริงได้
- ออกแบบ reporting layer ด้วย views และ materialized views ที่เหมาะกับ dashboard และ BI
- เขียน recursive CTE เพื่อคำนวณยอดขายสะสมของ category tree หลายระดับ
- เขียน transaction สำหรับ checkout process ที่ปลอดภัย มี `SAVEPOINT` และ error handling ครบถ้วน
- เข้าใจปัญหา concurrency (overselling) และเลือก isolation level ที่เหมาะสมเพื่อป้องกันปัญหา
- เขียน executive dashboard query ที่รวมเทคนิคทั้งหมดของระดับกลางไว้ในคิวรีเดียว
- ประเมินความพร้อมของตนเองก่อนก้าวเข้าสู่ระดับ Advanced (indexing, PL/pgSQL, JSON, partitioning)

บทนี้คือ **โปรเจกต์ปิดท้าย (Capstone Project)** ของระดับกลาง ครอบคลุม **Step 391–400** โดยจะไม่มีเนื้อหาทฤษฎีใหม่ แต่จะเป็นการ **สังเคราะห์ (synthesize)** ทุกสิ่งที่เรียนมาตั้งแต่ Part 021 ถึง Part 039 ให้กลายเป็นระบบที่ใช้งานได้จริง

> **หมายเหตุเรื่องสภาพแวดล้อม**: โค้ด SQL ทั้งหมดในบทนี้เขียนขึ้นสำหรับ PostgreSQL 16/17 และสามารถรันซ้ำได้ตั้งแต่ต้นจนจบ (self-contained) แนะนำให้สร้างฐานข้อมูลใหม่ เช่น `CREATE DATABASE ecommerce_capstone;` แล้วรันไล่ตามลำดับ Step

---

## Step 391: ภาพรวมโปรเจกต์และทบทวนสิ่งที่เรียนมา

### 391.1 Requirement ของระบบร้านค้าออนไลน์

ระบบที่เราจะสร้างคือระบบหลังบ้าน (back-office) ของร้านค้าออนไลน์ขนาดกลาง ซึ่งต้องรองรับความต้องการทางธุรกิจดังนี้:

| # | Requirement | เกี่ยวข้องกับตาราง |
|---|---|---|
| 1 | จัดหมวดหมู่สินค้าแบบลำดับชั้น (category ⊃ sub-category ⊃ sub-sub-category) | `categories` |
| 2 | ติดตามซัพพลายเออร์และประเทศต้นทาง | `suppliers` |
| 3 | บริหารสต๊อกสินค้า ราคา และสถานะเปิด/ปิดขาย | `products` |
| 4 | เก็บข้อมูลลูกค้าและวันที่สมัครสมาชิก | `customers` |
| 5 | เก็บข้อมูลพนักงานขายพร้อมสายบังคับบัญชา (manager hierarchy) | `employees` |
| 6 | บันทึกคำสั่งซื้อพร้อมสถานะและพนักงานผู้ดูแล | `orders` |
| 7 | บันทึกรายการสินค้าในแต่ละคำสั่งซื้อ | `order_items` |
| 8 | เก็บรีวิวและคะแนนสินค้าจากลูกค้า | `reviews` |
| 9 | บันทึกการชำระเงินของแต่ละคำสั่งซื้อ | `payments` |
| 10 | ทำรายงานสรุปยอดขาย สินค้าขายดี ลูกค้า VIP สำหรับผู้บริหาร | ทุกตารางร่วมกัน |

### 391.2 ตาราง mapping: Part 021–039 สอนอะไร และเราจะใช้ที่ไหนในโปรเจกต์นี้

| Part | หัวข้อ | ใช้งานจริงใน Step |
|---|---|---|
| 021 | INNER JOIN | 393, 395, 396, 399 |
| 022 | OUTER JOIN (LEFT/RIGHT/FULL) | 393, 399 (หาสินค้าไม่มีรีวิว, ลูกค้าไม่เคยสั่งซื้อ) |
| 023 | CROSS JOIN / SELF JOIN | 392 (employee-manager self join), 399 |
| 024 | Subqueries | 396, 397, 399 |
| 025 | CTE (WITH) | 393, 396, 399 |
| 026 | Recursive CTE | 392 (employee hierarchy), 395 (category tree) |
| 027 | Aggregate Functions | ทุก Step ตั้งแต่ 393 เป็นต้นไป |
| 028 | GROUP BY / HAVING / ROLLUP / CUBE | 393, 394, 399 |
| 029 | Window Functions พื้นฐาน | 393, 396 |
| 030 | Window Functions ขั้นสูง | 396, 399 |
| 031–033 (String/Date/Numeric Functions, สันนิษฐาน) | ฟังก์ชันจัดการข้อความ วันที่ ตัวเลข | 392, 396, 399 |
| 034 | CASE WHEN / COALESCE | 393, 396, 399 |
| 035 | Views | 393 |
| 036 | Materialized Views | 394 |
| 037 | Transactions & ACID | 397 |
| 038 | Isolation Levels | 398 |
| 039 | Sequences & Identity Columns | 392 |

> หมายเหตุ: หมายเลข Part 031–033 อาจสอดคล้องกับหัวข้อฟังก์ชันจัดการสตริง/วันที่/ตัวเลขในหลักสูตรของคุณ — ให้ยึดชื่อหัวข้อเป็นหลัก เนื่องจากลำดับ Part ที่แน่นอนอาจถูกจัดเรียงต่างกันเล็กน้อยในแต่ละรุ่นของหลักสูตร

### 391.3 แผนการทำงานของบทนี้

1. **Step 392** — สร้างสคีมาทั้งหมดใหม่ด้วย IDENTITY column พร้อมข้อมูลตัวอย่างสมจริง
2. **Step 393** — สร้าง Views สำหรับ reporting layer
3. **Step 394** — สร้าง Materialized View สำหรับ dashboard
4. **Step 395** — Recursive CTE: ยอดขายตาม category tree
5. **Step 396** — Window function ขั้นสูง: customer segmentation, MoM growth, running total
6. **Step 397** — Transaction สำหรับ checkout process
7. **Step 398** — ทดสอบ concurrency และ isolation level
8. **Step 399** — Executive dashboard query แบบครบวงจร
9. **Step 400** — สรุปทั้งหมด + checklist ก่อนขึ้นระดับ Advanced

---

## Step 392: Finalize Schema ด้วย IDENTITY Column และข้อมูลตัวอย่างสมจริง

### 392.1 เหตุผลเบื้องหลัง design decision แต่ละจุด

ก่อนสร้างตาราง เรามาทบทวนเหตุผลของการออกแบบทีละจุด เพราะการสอบ "world-class expert" ต้องอธิบายได้ ไม่ใช่แค่ก็อปโค้ดมาใช้:

| Design decision | เหตุผล |
|---|---|
| `GENERATED ALWAYS AS IDENTITY` แทน `SERIAL` | `SERIAL` เป็นเพียง syntactic sugar ที่สร้าง sequence แยกต่างหากและตั้งค่า `DEFAULT nextval(...)` ซึ่งไม่ใช่มาตรฐาน SQL ปัญหาคือสามารถ `INSERT` ค่าเข้า column นั้นตรง ๆ ได้โดยไม่ผ่าน sequence ทำให้ sequence เพี้ยนได้ง่าย ส่วน `GENERATED ALWAYS AS IDENTITY` เป็นมาตรฐาน SQL:2003 ควบคุมค่าอย่างเข้มงวดกว่า (ต้องใช้ `OVERRIDING SYSTEM VALUE` หากต้องการ insert ค่าเอง) และสื่อความหมายชัดเจนกว่าว่าเป็น surrogate key |
| `NUMERIC(10,2)` สำหรับราคาและเงิน | เงินต้องไม่มีความคลาดเคลื่อนจาก floating point เช่น `FLOAT`/`REAL` จะทำให้ 0.1 + 0.2 ≠ 0.3 ซึ่งอันตรายมากกับระบบการเงิน |
| `CHECK (unit_price >= 0)`, `CHECK (quantity > 0)`, `CHECK (rating BETWEEN 1 AND 5)` | ป้องกันข้อมูลผิดพลาดตั้งแต่ระดับฐานข้อมูล ไม่พึ่งพา validation ที่ชั้น application เพียงอย่างเดียว (defense in depth) |
| `TIMESTAMPTZ` สำหรับ `order_date`, `payment_date` | ระบบร้านค้าออนไลน์มีลูกค้าและพนักงานต่างโซนเวลา การเก็บ timestamp พร้อม time zone (เก็บเป็น UTC ภายใน) ป้องกันความสับสนเรื่องเวลาข้ามโซน |
| `DATE` สำหรับ `signup_date`, `hire_date`, `review_date` | ข้อมูลเหล่านี้สนใจแค่ "วันที่" ไม่ต้องการความละเอียดระดับเวลา การใช้ `DATE` ประหยัดพื้นที่และสื่อความหมายชัดเจนกว่า |
| `is_active BOOLEAN DEFAULT true` ใน `products` | ใช้ soft-delete pattern แทนการ `DELETE` จริง เพราะสินค้าที่เคยขายไปแล้วยังต้องผูกกับ `order_items` เดิมอยู่ (referential integrity) การปิดขาย (`is_active = false`) ปลอดภัยกว่าการลบ |
| `manager_id INTEGER REFERENCES employees(employee_id)` (self-reference) | ใช้ foreign key อ้างอิงตารางตัวเอง เพื่อสร้างโครงสร้างสายบังคับบัญชาแบบ tree ซึ่งจะ query ด้วย self join หรือ recursive CTE |
| `unit_price` ซ้ำอยู่ทั้งใน `products` และ `order_items` | นี่คือ **denormalization ที่จงใจ (intentional)** — ราคาสินค้าที่ขายจริง ณ เวลาสั่งซื้อต้อง "แช่แข็ง" ไว้ใน `order_items.unit_price` เพราะราคาใน `products` อาจเปลี่ยนแปลงภายหลัง (เช่น ลดราคา/ขึ้นราคา) หากไม่เก็บซ้ำ รายงานยอดขายย้อนหลังจะผิดทันทีที่ราคาสินค้าปัจจุบันเปลี่ยน |
| `status VARCHAR(20)` ใน `orders` แทน `ENUM` | เพื่อความง่ายในการเรียนรู้และความยืดหยุ่นในการเพิ่มสถานะใหม่โดยไม่ต้อง `ALTER TYPE` (ในระดับ Advanced เราจะพูดถึงข้อดีข้อเสียของ native `ENUM` เทียบกับ `VARCHAR` + `CHECK` เทียบกับตาราง lookup) |
| `ON DELETE` ไม่ได้ระบุ (default `NO ACTION`) | เพื่อบังคับให้ทุกการลบข้อมูลอ้างอิง (เช่น ลบลูกค้า) ต้องผ่านการพิจารณาอย่างรอบคอบ (เช่นใช้ soft-delete แทน) ไม่ปล่อยให้ `ON DELETE CASCADE` ลบข้อมูลประวัติการขายหายไปโดยไม่ตั้งใจ |

### 392.2 สร้างฐานข้อมูลและ Schema

```sql
-- แนะนำให้รันในฐานข้อมูลใหม่
-- CREATE DATABASE ecommerce_capstone;
-- \c ecommerce_capstone

DROP TABLE IF EXISTS payments, reviews, order_items, orders,
                      employees, customers, products, suppliers, categories CASCADE;

-- 1) categories: โครงสร้างลำดับชั้น (self-referencing tree)
CREATE TABLE categories (
    category_id        INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    category_name       VARCHAR(100) NOT NULL,
    parent_category_id  INTEGER REFERENCES categories(category_id)
);

-- 2) suppliers
CREATE TABLE suppliers (
    supplier_id     INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    supplier_name   VARCHAR(150) NOT NULL,
    country         VARCHAR(60)
);

-- 3) products
CREATE TABLE products (
    product_id      INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    product_name    VARCHAR(150) NOT NULL,
    category_id     INTEGER REFERENCES categories(category_id),
    supplier_id     INTEGER REFERENCES suppliers(supplier_id),
    unit_price      NUMERIC(10,2) NOT NULL CHECK (unit_price >= 0),
    stock_quantity  INTEGER NOT NULL DEFAULT 0,
    is_active       BOOLEAN NOT NULL DEFAULT true
);

-- 4) customers
CREATE TABLE customers (
    customer_id   INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    first_name    VARCHAR(60) NOT NULL,
    last_name     VARCHAR(60) NOT NULL,
    email         VARCHAR(150) UNIQUE,
    country       VARCHAR(60),
    signup_date   DATE NOT NULL DEFAULT CURRENT_DATE
);

-- 5) employees: self-reference สำหรับสายบังคับบัญชา
CREATE TABLE employees (
    employee_id   INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    first_name    VARCHAR(60) NOT NULL,
    last_name     VARCHAR(60) NOT NULL,
    hire_date     DATE NOT NULL,
    manager_id    INTEGER REFERENCES employees(employee_id),
    department    VARCHAR(60)
);

-- 6) orders
CREATE TABLE orders (
    order_id      INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    customer_id   INTEGER REFERENCES customers(customer_id),
    employee_id   INTEGER REFERENCES employees(employee_id),
    order_date    TIMESTAMPTZ NOT NULL DEFAULT now(),
    status        VARCHAR(20) NOT NULL DEFAULT 'pending'
                  CHECK (status IN ('pending','processing','shipped','completed','cancelled','refunded')),
    ship_country  VARCHAR(60)
);

-- 7) order_items
CREATE TABLE order_items (
    order_item_id  INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    order_id       INTEGER REFERENCES orders(order_id),
    product_id     INTEGER REFERENCES products(product_id),
    quantity       INTEGER NOT NULL CHECK (quantity > 0),
    unit_price     NUMERIC(10,2) NOT NULL
);

-- 8) reviews
CREATE TABLE reviews (
    review_id     INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    product_id    INTEGER REFERENCES products(product_id),
    customer_id   INTEGER REFERENCES customers(customer_id),
    rating        INTEGER NOT NULL CHECK (rating BETWEEN 1 AND 5),
    review_text   TEXT,
    review_date   DATE NOT NULL DEFAULT CURRENT_DATE
);

-- 9) payments
CREATE TABLE payments (
    payment_id       INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    order_id         INTEGER REFERENCES orders(order_id),
    payment_date     TIMESTAMPTZ NOT NULL DEFAULT now(),
    amount           NUMERIC(10,2) NOT NULL,
    payment_method   VARCHAR(30)
);
```

> **สังเกต**: เราเพิ่ม `CHECK (status IN (...))` ให้ `orders.status` เมื่อเทียบกับต้นแบบใน Part 021–039 นี่คือการ "finalize" สคีมาให้รัดกุมขึ้นตามที่ระบุในหัวข้อ Step นี้ — เป็นเรื่องปกติที่สคีมาจะค่อย ๆ แข็งแกร่งขึ้นเมื่อใกล้จบโปรเจกต์

### 392.3 Seed Data — เล่าเรื่องธุรกิจตลอดหลายเดือน (ม.ค.–ก.ย. 2026)

ข้อมูลตัวอย่างด้านล่างถูกออกแบบให้เป็น **เรื่องราวธุรกิจที่สอดคล้องกัน (coherent story)**: ร้านค้าออนไลน์เปิดขายสินค้าอิเล็กทรอนิกส์ ของใช้ในบ้าน และแฟชั่น มีลูกค้าสมัครเพิ่มขึ้นเรื่อย ๆ มีพนักงานขายหลายทีม และมียอดขายเติบโตแบบมีขึ้นมีลงตลอด 9 เดือน

#### (1) categories — โครงสร้างลำดับชั้น 3 ระดับ

```sql
INSERT INTO categories (category_name, parent_category_id) VALUES
('Electronics', NULL),                 -- 1 (root)
('Computers', 1),                      -- 2
('Laptops', 2),                        -- 3
('Desktops', 2),                       -- 4
('Smartphones', 1),                    -- 5
('Accessories', 1),                    -- 6
('Cables', 6),                         -- 7
('Chargers', 6),                       -- 8
('Home & Kitchen', NULL),              -- 9 (root)
('Furniture', 9),                      -- 10
('Appliances', 9),                     -- 11
('Small Appliances', 11),              -- 12
('Fashion', NULL),                     -- 13 (root)
('Men''s Clothing', 13),               -- 14
('Women''s Clothing', 13);             -- 15
```

#### (2) suppliers

```sql
INSERT INTO suppliers (supplier_name, country) VALUES
('SiamTech Distribution', 'Thailand'),        -- 1
('Shenzhen Global Electronics', 'China'),     -- 2
('Nihon Denki Co.', 'Japan'),                 -- 3
('Bavaria Precision GmbH', 'Germany'),        -- 4
('Seoul Digital Corp.', 'South Korea'),       -- 5
('Vietnam Textile Group', 'Vietnam'),         -- 6
('American Home Goods Inc.', 'USA'),          -- 7
('Bangkok Furniture Works', 'Thailand'),      -- 8
('Guangzhou Appliance Ltd.', 'China'),        -- 9
('Nordic Design House', 'Sweden'),            -- 10
('Taiwan Semiconductor Supply', 'Taiwan'),    -- 11
('India Apparel Exports', 'India'),           -- 12
('Malaysia Circuit Traders', 'Malaysia'),     -- 13
('Chonburi Cable Factory', 'Thailand'),       -- 14
('Osaka Kitchenware Co.', 'Japan');           -- 15
```

#### (3) products — 25 รายการกระจายตามหมวดหมู่

```sql
INSERT INTO products (product_name, category_id, supplier_id, unit_price, stock_quantity, is_active) VALUES
('UltraBook Pro 14"',            3,  2,  32900.00, 40,  true),   -- 1
('UltraBook Air 13"',            3,  2,  24900.00, 55,  true),   -- 2
('GameForce Laptop RTX',         3,  5,  54900.00, 20,  true),   -- 3
('OfficeMate Desktop i5',        4,  2,  18900.00, 30,  true),   -- 4
('PowerStation Desktop i7',      4,  4,  29900.00, 15,  true),   -- 5
('NovaPhone X12',                5,  5,  25900.00, 60,  true),   -- 6
('NovaPhone X12 Lite',           5,  5,  16900.00, 80,  true),   -- 7
('PixelClear Phone 9',           5,  3,  21900.00, 45,  true),   -- 8
('USB-C Cable 2m',               7,  14,   190.00, 500, true),   -- 9
('HDMI Cable 4K 3m',             7,  14,   290.00, 350, true),   -- 10
('65W GaN Charger',              8,  11,   890.00, 220, true),   -- 11
('Wireless Charging Pad',        8,  5,   690.00, 150, true),    -- 12
('Ergonomic Office Chair',       10, 8,   4990.00, 25,  true),   -- 13
('Standing Desk Adjustable',     10, 8,   7990.00, 18,  true),   -- 14
('Bookshelf 5-Tier',             10, 10,  2490.00, 22,  true),   -- 15
('Smart Air Purifier',           12, 9,   3990.00, 35,  true),   -- 16
('Robot Vacuum Cleaner',         12, 9,   8990.00, 20,  true),   -- 17
('Electric Kettle 1.7L',         12, 15,   890.00, 90,  true),   -- 18
('Rice Cooker Deluxe',           12, 15,  1590.00, 60,  true),   -- 19
('Men''s Slim Fit Shirt',        14, 6,    590.00, 120, true),   -- 20
('Men''s Denim Jacket',          14, 6,   1290.00, 70,  true),   -- 21
('Women''s Summer Dress',        15, 12,  790.00, 90,  true),    -- 22
('Women''s Yoga Pants',          15, 12,  650.00, 110, true),    -- 23
('Retro Mechanical Keyboard',    6,  13,  2190.00, 40,  true),   -- 24
('Wireless Mouse Silent Click',  6,  13,   490.00, 200, false);  -- 25 (เลิกขายแล้ว: is_active = false)
```

#### (4) customers — 20 ราย สมัครกระจายตลอดปี

```sql
INSERT INTO customers (first_name, last_name, email, country, signup_date) VALUES
('Somchai',   'Jaidee',      'somchai.j@example.com',    'Thailand',  '2026-01-05'),  -- 1
('Suda',      'Rungrot',     'suda.r@example.com',       'Thailand',  '2026-01-08'),  -- 2
('Anan',      'Wongsawat',   'anan.w@example.com',       'Thailand',  '2026-01-15'),  -- 3
('Malee',     'Suksawat',    'malee.s@example.com',      'Thailand',  '2026-01-22'),  -- 4
('Kittipong', 'Charoen',     'kitti.c@example.com',      'Thailand',  '2026-02-02'),  -- 5
('Nattaya',   'Pongsri',     'nattaya.p@example.com',    'Thailand',  '2026-02-10'),  -- 6
('John',      'Smith',       'john.smith@example.com',   'USA',       '2026-02-18'),  -- 7
('Emily',     'Johnson',     'emily.j@example.com',      'USA',       '2026-02-25'),  -- 8
('Wei',       'Zhang',       'wei.zhang@example.com',    'China',     '2026-03-03'),  -- 9
('Yuki',      'Tanaka',      'yuki.t@example.com',       'Japan',     '2026-03-11'),  -- 10
('Pranee',    'Boonmee',     'pranee.b@example.com',     'Thailand',  '2026-03-19'),  -- 11
('Chai',      'Thongdee',    'chai.t@example.com',       'Thailand',  '2026-04-01'),  -- 12
('Siriporn',  'Meesuk',      'siriporn.m@example.com',   'Thailand',  '2026-04-14'),  -- 13
('David',     'Miller',      'david.m@example.com',      'USA',       '2026-04-28'),  -- 14
('Minji',     'Kim',         'minji.k@example.com',      'South Korea','2026-05-06'), -- 15
('Prasert',   'Kaewta',      'prasert.k@example.com',    'Thailand',  '2026-05-20'),  -- 16
('Ploy',      'Sirisak',     'ploy.s@example.com',       'Thailand',  '2026-06-09'),  -- 17
('Aroon',     'Intharat',    'aroon.i@example.com',      'Thailand',  '2026-07-02'),  -- 18
('Napat',     'Chaiyaporn',  'napat.c@example.com',      'Thailand',  '2026-08-05'),  -- 19
('Benjawan',  'Silapat',     'benjawan.s@example.com',   'Thailand',  '2026-09-01');  -- 20
```

#### (5) employees — สายบังคับบัญชา (self-reference)

```sql
INSERT INTO employees (first_name, last_name, hire_date, manager_id, department) VALUES
('Chatchai',  'Boonrod',   '2023-01-10', NULL,  'Management'),   -- 1 CEO
('Nipa',      'Sombat',    '2023-03-15', 1,     'Sales'),        -- 2 Sales Manager
('Wichai',    'Petchara',  '2023-06-01', 1,     'Operations'),   -- 3 Ops Manager
('Sunisa',    'Rattana',   '2024-01-20', 2,     'Sales'),        -- 4 Sales Rep
('Thanapon',  'Wisetkul',  '2024-02-14', 2,     'Sales'),        -- 5 Sales Rep
('Achara',    'Sriwattana','2024-04-02', 2,     'Sales'),        -- 6 Sales Rep
('Kamon',     'Deesawang', '2024-05-19', 3,     'Warehouse'),    -- 7 Warehouse Staff
('Ratree',    'Chansiri',  '2024-07-08', 3,     'Warehouse'),    -- 8 Warehouse Staff
('Somsak',    'Naowarat',  '2024-09-11', 3,     'Support'),      -- 9 Support Staff
('Ladda',     'Phromsri',  '2025-01-06', 2,     'Sales'),        -- 10 Sales Rep
('Preecha',   'Inthanon',  '2025-03-22', 3,     'Support'),      -- 11 Support Staff
('Waraporn',  'Thepnimit', '2025-06-30', 2,     'Sales');        -- 12 Sales Rep
```

#### (6) orders — 25 คำสั่งซื้อ กระจายตลอด ม.ค.–ก.ย. 2026

```sql
INSERT INTO orders (customer_id, employee_id, order_date, status, ship_country) VALUES
(1,  4,  '2026-01-12 10:15:00+07', 'completed',  'Thailand'),  -- 1
(2,  4,  '2026-01-20 14:30:00+07', 'completed',  'Thailand'),  -- 2
(3,  5,  '2026-01-28 09:05:00+07', 'completed',  'Thailand'),  -- 3
(1,  4,  '2026-02-05 16:40:00+07', 'completed',  'Thailand'),  -- 4
(4,  6,  '2026-02-14 11:20:00+07', 'cancelled',  'Thailand'),  -- 5
(5,  4,  '2026-02-19 13:10:00+07', 'completed',  'Thailand'),  -- 6
(7,  5,  '2026-03-02 08:45:00+07', 'completed',  'USA'),       -- 7
(6,  6,  '2026-03-09 15:55:00+07', 'completed',  'Thailand'),  -- 8
(8,  5,  '2026-03-17 10:30:00+07', 'refunded',   'USA'),       -- 9
(9,  4,  '2026-03-25 12:00:00+07', 'completed',  'China'),     -- 10
(2,  10, '2026-04-03 09:15:00+07', 'completed',  'Thailand'),  -- 11
(11, 10, '2026-04-11 17:20:00+07', 'completed',  'Thailand'),  -- 12
(10, 5,  '2026-04-19 14:05:00+07', 'completed',  'Japan'),     -- 13
(12, 4,  '2026-04-27 11:35:00+07', 'completed',  'Thailand'),  -- 14
(13, 6,  '2026-05-05 10:00:00+07', 'completed',  'Thailand'),  -- 15
(14, 5,  '2026-05-13 16:25:00+07', 'completed',  'USA'),       -- 16
(15, 10, '2026-05-22 09:40:00+07', 'completed',  'South Korea'),-- 17
(3,  4,  '2026-06-01 13:50:00+07', 'completed',  'Thailand'),  -- 18
(16, 6,  '2026-06-10 15:15:00+07', 'completed',  'Thailand'),  -- 19
(1,  12, '2026-06-20 10:20:00+07', 'completed',  'Thailand'),  -- 20
(17, 12, '2026-07-04 11:05:00+07', 'completed',  'Thailand'),  -- 21
(18, 10, '2026-07-18 14:45:00+07', 'completed',  'Thailand'),  -- 22
(5,  4,  '2026-08-02 09:30:00+07', 'completed',  'Thailand'),  -- 23
(19, 6,  '2026-08-21 16:10:00+07', 'completed',  'Thailand'),  -- 24
(20, 12, '2026-09-10 10:50:00+07', 'processing', 'Thailand');  -- 25 (ยังไม่ปิดยอด)
```

#### (7) order_items — รายการสินค้าในแต่ละคำสั่งซื้อ (ราคาถูก "แช่แข็ง" ไว้ ณ เวลาสั่งซื้อ)

```sql
INSERT INTO order_items (order_id, product_id, quantity, unit_price) VALUES
(1, 1, 1, 32900.00), (1, 9, 2, 190.00),
(2, 6, 1, 25900.00), (2, 11, 1, 890.00),
(3, 20, 2, 590.00), (3, 22, 1, 790.00),
(4, 4, 1, 18900.00), (4, 10, 1, 290.00),
(5, 13, 1, 4990.00),
(6, 7, 1, 16900.00), (6, 12, 1, 690.00),
(7, 2, 1, 24900.00),
(8, 16, 1, 3990.00), (8, 18, 1, 890.00),
(9, 8, 1, 21900.00),
(10, 3, 1, 54900.00),
(11, 24, 2, 2190.00), (11, 25, 1, 490.00),
(12, 17, 1, 8990.00),
(13, 1, 1, 32900.00), (13, 9, 3, 190.00),
(14, 21, 2, 1290.00), (14, 23, 1, 650.00),
(15, 14, 1, 7990.00),
(16, 6, 1, 25900.00),
(17, 19, 2, 1590.00),
(18, 5, 1, 29900.00),
(19, 15, 1, 2490.00), (19, 13, 1, 4990.00),
(20, 7, 2, 16900.00),
(21, 22, 3, 790.00), (21, 20, 2, 590.00),
(22, 16, 1, 3990.00),
(23, 2, 1, 24900.00), (23, 11, 2, 890.00),
(24, 8, 1, 21900.00),
(25, 4, 1, 18900.00), (25, 10, 2, 290.00);
```

#### (8) reviews — 20 รีวิวจากลูกค้าที่เคยซื้อสินค้านั้นจริง

```sql
INSERT INTO reviews (product_id, customer_id, rating, review_text, review_date) VALUES
(1,  1,  5, 'โน้ตบุ๊กแรงมาก ทำงานลื่นสุดๆ',                '2026-01-20'),
(9,  1,  4, 'สายชาร์จดี แข็งแรง',                          '2026-01-21'),
(6,  2,  5, 'มือถือกล้องสวย แบตอึด',                        '2026-01-25'),
(20, 3,  4, 'เสื้อใส่สบาย ไซส์พอดี',                        '2026-02-01'),
(22, 3,  5, 'ชุดสวยมาก ผ้าดี',                              '2026-02-02'),
(4,  1,  4, 'คอมพิวเตอร์ทำงานออฟฟิศลื่นดี',                  '2026-02-10'),
(7,  5,  3, 'ใช้ได้ปกติ แต่คาดหวังไว้มากกว่านี้',            '2026-02-25'),
(2,  7,  5, 'Great laptop, very portable',                  '2026-03-10'),
(16, 6,  5, 'เครื่องฟอกอากาศเงียบ ใช้ง่าย',                  '2026-03-15'),
(8,  8,  2, 'จอมีปัญหาหลังใช้ไปสองสัปดาห์',                  '2026-03-20'),
(3,  9,  5, 'เล่นเกมลื่นมาก การ์ดจอแรงสมราคา',               '2026-04-01'),
(24, 2,  4, 'คีย์บอร์ดเสียงดีมาก พิมพ์สนุก',                  '2026-04-15'),
(17, 11, 5, 'หุ่นยนต์ดูดฝุ่นฉลาดมาก',                        '2026-04-20'),
(1,  10, 5, 'Excellent build quality',                      '2026-04-25'),
(21, 12, 4, 'แจ็คเก็ตยีนส์ใส่สบาย',                          '2026-05-01'),
(14, 13, 5, 'โต๊ะปรับระดับได้ดีมาก คุ้มราคา',                 '2026-05-10'),
(6,  14, 4, 'Good phone overall, camera is nice',           '2026-05-20'),
(19, 15, 5, '밥솥 정말 좋아요 (หม้อหุงข้าวดีมาก)',            '2026-05-28'),
(5,  18, 4, 'พีซีแรงดี เหมาะกับงานกราฟิก',                    '2026-07-25'),
(2,  19, 5, 'ใช้งานคุ้มค่ามาก แนะนำเลย',                      '2026-08-25');
```

#### (9) payments — เฉพาะคำสั่งซื้อที่ชำระเงินแล้ว (ไม่รวม cancelled/processing)

```sql
INSERT INTO payments (order_id, payment_date, amount, payment_method) VALUES
(1,  '2026-01-12 10:20:00+07', 33280.00, 'credit_card'),
(2,  '2026-01-20 14:35:00+07', 26790.00, 'promptpay'),
(3,  '2026-01-28 09:10:00+07',  1970.00, 'credit_card'),
(4,  '2026-02-05 16:45:00+07', 19190.00, 'promptpay'),
(6,  '2026-02-19 13:15:00+07', 17590.00, 'credit_card'),
(7,  '2026-03-02 08:50:00+07', 24900.00, 'paypal'),
(8,  '2026-03-09 16:00:00+07',  4880.00, 'promptpay'),
(9,  '2026-03-17 10:35:00+07', 21900.00, 'credit_card'),
(10, '2026-03-25 12:05:00+07', 54900.00, 'credit_card'),
(11, '2026-04-03 09:20:00+07',  4870.00, 'promptpay'),
(12, '2026-04-11 17:25:00+07',  8990.00, 'credit_card'),
(13, '2026-04-19 14:10:00+07', 33470.00, 'paypal'),
(14, '2026-04-27 11:40:00+07',  3230.00, 'promptpay'),
(15, '2026-05-05 10:05:00+07',  7990.00, 'credit_card'),
(16, '2026-05-13 16:30:00+07', 25900.00, 'credit_card'),
(17, '2026-05-22 09:45:00+07',  3180.00, 'promptpay'),
(18, '2026-06-01 13:55:00+07', 29900.00, 'credit_card'),
(19, '2026-06-10 15:20:00+07',  7480.00, 'promptpay'),
(20, '2026-06-20 10:25:00+07', 33800.00, 'credit_card'),
(21, '2026-07-04 11:10:00+07',  3550.00, 'promptpay'),
(22, '2026-07-18 14:50:00+07',  3990.00, 'credit_card'),
(23, '2026-08-02 09:35:00+07', 26680.00, 'credit_card'),
(24, '2026-08-21 16:15:00+07', 21900.00, 'promptpay');
-- หมายเหตุ: order 5 (cancelled), order 9 (refunded ภายหลังชำระ), order 25 (processing) ไม่มี/มีการจัดการพิเศษ
```

> **สังเกต order 9**: เป็นออเดอร์ที่ `status = 'refunded'` แต่ยังมี record การชำระเงินอยู่ในตาราง `payments` — นี่คือสถานการณ์จริงที่พบบ่อย (จ่ายเงินแล้วค่อยขอคืนภายหลัง) ซึ่งจะใช้ทดสอบตรรกะการคำนวณยอดขายสุทธิใน Step 399

### 392.4 ตรวจสอบความถูกต้องของข้อมูลเบื้องต้น

```sql
SELECT 'categories' AS table_name, COUNT(*) FROM categories
UNION ALL SELECT 'suppliers', COUNT(*) FROM suppliers
UNION ALL SELECT 'products', COUNT(*) FROM products
UNION ALL SELECT 'customers', COUNT(*) FROM customers
UNION ALL SELECT 'employees', COUNT(*) FROM employees
UNION ALL SELECT 'orders', COUNT(*) FROM orders
UNION ALL SELECT 'order_items', COUNT(*) FROM order_items
UNION ALL SELECT 'reviews', COUNT(*) FROM reviews
UNION ALL SELECT 'payments', COUNT(*) FROM payments;
```

ผลลัพธ์ตัวอย่าง:

```
 table_name  | count
-------------+-------
 categories  |    15
 suppliers   |    15
 products    |    25
 customers   |    20
 employees   |    12
 orders      |    25
 order_items |    38
 reviews     |    20
 payments    |    23
```

ตรวจสอบว่า sequence ของ IDENTITY column เดินหน้าถูกต้อง (สืบเนื่องจาก Part 039):

```sql
SELECT pg_get_serial_sequence('products', 'product_id') AS seq_name,
       last_value
FROM products, LATERAL (SELECT last_value FROM products_product_id_seq) s;
```

```
              seq_name               | last_value
--------------------------------------+------------
 public.products_product_id_seq       |         25
```

---

## Step 393: สร้าง Views สำหรับ Reporting Layer

Views ทำหน้าที่เป็น **abstraction layer** ระหว่างสคีมาฐานข้อมูล (ที่ normalize ไว้อย่างดี) กับผู้ใช้งานรายงาน (BI analyst, dashboard) ซึ่งไม่ต้องการเขียน JOIN ซับซ้อนซ้ำ ๆ ทุกครั้ง เราจะสร้าง 3 views โดยผสม JOIN + aggregate + window function เข้าด้วยกัน (ทบทวนจาก Part 035)

### 393.1 `order_summary_view` — สรุปยอดแต่ละคำสั่งซื้อ

```sql
CREATE OR REPLACE VIEW order_summary_view AS
SELECT
    o.order_id,
    o.order_date,
    o.status,
    c.customer_id,
    c.first_name || ' ' || c.last_name AS customer_name,
    e.employee_id,
    e.first_name || ' ' || e.last_name AS employee_name,
    o.ship_country,
    COUNT(oi.order_item_id)                       AS item_count,
    SUM(oi.quantity)                               AS total_quantity,
    SUM(oi.quantity * oi.unit_price)               AS order_total,
    COALESCE(p.amount, 0)                          AS amount_paid,
    CASE
        WHEN o.status = 'refunded'   THEN 'ต้องติดตามการคืนเงิน'
        WHEN o.status = 'cancelled'  THEN 'ยกเลิกแล้ว'
        WHEN p.payment_id IS NULL AND o.status <> 'processing' THEN 'ยังไม่พบการชำระเงิน'
        ELSE 'ปกติ'
    END AS payment_flag
FROM orders o
JOIN customers c        ON c.customer_id = o.customer_id
LEFT JOIN employees e   ON e.employee_id = o.employee_id
LEFT JOIN order_items oi ON oi.order_id = o.order_id
LEFT JOIN payments p    ON p.order_id = o.order_id
GROUP BY o.order_id, o.order_date, o.status, c.customer_id, c.first_name, c.last_name,
         e.employee_id, e.first_name, e.last_name, o.ship_country, p.amount, p.payment_id
ORDER BY o.order_date;
```

ทดสอบใช้งาน:

```sql
SELECT order_id, customer_name, employee_name, order_total, amount_paid, payment_flag
FROM order_summary_view
WHERE payment_flag <> 'ปกติ';
```

```
 order_id | customer_name  | employee_name    | order_total | amount_paid |      payment_flag
----------+-----------------+------------------+-------------+-------------+-------------------------
        5 | Malee Suksawat  | Achara Sriwattana|     4990.00 |        0.00 | ยกเลิกแล้ว
        9 | Emily Johnson   | Thanapon Wisetkul|    21900.00 |    21900.00 | ต้องติดตามการคืนเงิน
       25 | Benjawan Silapat| Waraporn Thepnimit|   19480.00 |        0.00 | ยังไม่พบการชำระเงิน
```

> **หมายเหตุการออกแบบ**: view นี้ใช้ `LEFT JOIN` กับ `order_items` และ `payments` (ทบทวน Part 022) เพราะออเดอร์บางรายการอาจยังไม่มีรายการสินค้า หรือยังไม่มีการชำระเงิน — หาก `INNER JOIN` ออเดอร์เหล่านั้นจะหายไปจากรายงานทันที

### 393.2 `product_performance_view` — ผลงานสินค้าแต่ละชิ้น พร้อม ranking ด้วย window function

```sql
CREATE OR REPLACE VIEW product_performance_view AS
SELECT
    p.product_id,
    p.product_name,
    cat.category_name,
    s.supplier_name,
    p.unit_price,
    p.stock_quantity,
    p.is_active,
    COALESCE(SUM(oi.quantity), 0)                       AS units_sold,
    COALESCE(SUM(oi.quantity * oi.unit_price), 0)        AS revenue,
    COALESCE(ROUND(AVG(r.rating), 2), 0)                 AS avg_rating,
    COUNT(DISTINCT r.review_id)                          AS review_count,
    RANK() OVER (ORDER BY COALESCE(SUM(oi.quantity * oi.unit_price), 0) DESC) AS revenue_rank,
    NTILE(4) OVER (ORDER BY COALESCE(SUM(oi.quantity * oi.unit_price), 0) DESC) AS revenue_quartile
FROM products p
JOIN categories cat          ON cat.category_id = p.category_id
JOIN suppliers s              ON s.supplier_id = p.supplier_id
LEFT JOIN order_items oi      ON oi.product_id = p.product_id
LEFT JOIN orders o            ON o.order_id = oi.order_id AND o.status NOT IN ('cancelled')
LEFT JOIN reviews r           ON r.product_id = p.product_id
GROUP BY p.product_id, p.product_name, cat.category_name, s.supplier_name,
         p.unit_price, p.stock_quantity, p.is_active;
```

> หมายเหตุ: ในการ `JOIN` กับ `orders` เราตัดสถานะ `cancelled` ออก เพื่อไม่ให้ยอดขายของสินค้านับรวมออเดอร์ที่ถูกยกเลิก (ทบทวนความเข้าใจเรื่อง business logic ที่ฝังอยู่ใน view)

ทดสอบดู top 5 สินค้าตาม revenue:

```sql
SELECT product_name, category_name, units_sold, revenue, avg_rating, revenue_rank
FROM product_performance_view
ORDER BY revenue_rank
LIMIT 5;
```

```
     product_name      | category_name | units_sold |  revenue  | avg_rating | revenue_rank
------------------------+---------------+------------+-----------+------------+--------------
 GameForce Laptop RTX   | Laptops       |          1 |  54900.00 |       5.00 |            1
 UltraBook Pro 14"      | Laptops       |          2 |  65800.00 |       5.00 |            1
 NovaPhone X12          | Smartphones   |          3 |  77700.00 |       4.50 |            1
 PowerStation Desktop i7| Desktops      |          1 |  29900.00 |       4.00 |            4
 UltraBook Air 13"      | Laptops       |          2 |  49800.00 |       5.00 |            5
```

> **ข้อสังเกตเรื่อง `RANK()`**: จะเห็นว่ามีหลายแถวที่ `revenue_rank = 1` เนื่องจาก `RANK()` ให้อันดับเท่ากันเมื่อค่าเท่ากัน (ในตัวอย่างจริงรายได้แต่ละตัวต่างกัน แต่หากมีค่าเท่ากันพอดีจะเกิดพฤติกรรมนี้) — นี่คือความแตกต่างสำคัญจาก `ROW_NUMBER()` ที่สอนใน Part 029

### 393.3 `customer_lifetime_value_view` — มูลค่าตลอดชีพของลูกค้า (CLV)

```sql
CREATE OR REPLACE VIEW customer_lifetime_value_view AS
SELECT
    c.customer_id,
    c.first_name || ' ' || c.last_name  AS customer_name,
    c.country,
    c.signup_date,
    COUNT(DISTINCT o.order_id) FILTER (WHERE o.status = 'completed') AS completed_orders,
    COALESCE(SUM(oi.quantity * oi.unit_price)
             FILTER (WHERE o.status = 'completed'), 0)               AS lifetime_value,
    MIN(o.order_date) AS first_order_date,
    MAX(o.order_date) AS last_order_date,
    ROUND(
        COALESCE(SUM(oi.quantity * oi.unit_price) FILTER (WHERE o.status = 'completed'), 0)
        / NULLIF(COUNT(DISTINCT o.order_id) FILTER (WHERE o.status = 'completed'), 0)
    , 2) AS avg_order_value,
    NTILE(4) OVER (
        ORDER BY COALESCE(SUM(oi.quantity * oi.unit_price)
                          FILTER (WHERE o.status = 'completed'), 0) DESC
    ) AS value_quartile
FROM customers c
LEFT JOIN orders o        ON o.customer_id = c.customer_id
LEFT JOIN order_items oi  ON oi.order_id = o.order_id
GROUP BY c.customer_id, c.first_name, c.last_name, c.country, c.signup_date;
```

> ใช้ `FILTER (WHERE ...)` (ทบทวน Part 027/028) เพื่อ aggregate เฉพาะออเดอร์ที่สถานะ `completed` โดยไม่ต้องแยก subquery — สะอาดและอ่านง่ายกว่าการเขียน `CASE WHEN` ซ้อนใน `SUM`

ทดสอบดูลูกค้ากลุ่ม top quartile (VIP):

```sql
SELECT customer_name, country, completed_orders, lifetime_value, avg_order_value
FROM customer_lifetime_value_view
WHERE value_quartile = 1
ORDER BY lifetime_value DESC;
```

```
   customer_name  | country  | completed_orders | lifetime_value | avg_order_value
-------------------+----------+-------------------+-----------------+------------------
 Somchai Jaidee    | Thailand |                 3 |        66610.00|         22203.33
 Wei Zhang         | China    |                 1 |        54900.00|         54900.00
 Yuki Tanaka       | Japan    |                 1 |        33470.00|         33470.00
 Chai Thongdee     | Thailand |                 1 |        27380.00|         27380.00
 John Smith        | USA      |                 1 |        24900.00|         24900.00
```

### 393.4 ทำไมต้องแยกเป็น 3 views แทนที่จะรวมเป็น view เดียว

หลักการออกแบบ reporting layer ที่ดี: **แต่ละ view ควรตอบคำถามธุรกิจหนึ่งเรื่องอย่างชัดเจน (single responsibility)**

| View | ตอบคำถามธุรกิจ | Grain (ระดับความละเอียด) |
|---|---|---|
| `order_summary_view` | "แต่ละออเดอร์มีมูลค่าเท่าไร ชำระเงินหรือยัง" | 1 แถว = 1 order |
| `product_performance_view` | "สินค้าตัวไหนขายดี ได้เรตติ้งดี" | 1 แถว = 1 product |
| `customer_lifetime_value_view` | "ลูกค้าคนไหนสร้างรายได้ให้ร้านมากที่สุด" | 1 แถว = 1 customer |

การรวมทั้งหมดเป็น view เดียวจะทำให้เกิดปัญหา **fan-out** (การ JOIN หลายตารางที่มีความสัมพันธ์แบบ one-to-many พร้อมกันหลายทาง ทำให้ตัวเลข aggregate ผิดเพี้ยนจากการคูณซ้ำ) ซึ่งเป็นข้อผิดพลาดที่พบบ่อยมากในการเขียนรายงาน SQL

---

## Step 394: Materialized View สำหรับ Dashboard (`monthly_sales_mv`)

### 394.1 ทำไมต้องใช้ Materialized View แทน View ธรรมดา

Views ใน Step 393 คำนวณผลลัพธ์ **ใหม่ทุกครั้ง** ที่ถูก query (คือแค่เก็บ SQL text ไว้ ไม่เก็บข้อมูลจริง) ซึ่งเหมาะกับข้อมูลที่ต้องเป็นปัจจุบันเสมอ แต่สำหรับ **dashboard ผู้บริหาร** ที่ต้อง query ข้อมูลย้อนหลังทั้งปี พร้อมการ JOIN และ aggregate หนัก ๆ ซ้ำ ๆ ทุกครั้งที่มีคนเปิดหน้า dashboard จะทำให้ฐานข้อมูลทำงานหนักโดยไม่จำเป็น (ทบทวน Part 036)

Materialized View แก้ปัญหานี้โดย **เก็บผลลัพธ์ที่คำนวณแล้วไว้เป็นข้อมูลจริงบนดิสก์** และอัปเดตเมื่อสั่ง `REFRESH` เท่านั้น เหมาะกับข้อมูลที่ยอมรับความ "ล่าช้า" ได้ระดับหนึ่ง (เช่น รีเฟรชทุกคืน)

### 394.2 สร้าง `monthly_sales_mv`

```sql
CREATE MATERIALIZED VIEW monthly_sales_mv AS
SELECT
    DATE_TRUNC('month', o.order_date)::DATE       AS sales_month,
    cat.category_name,
    COUNT(DISTINCT o.order_id)                     AS order_count,
    SUM(oi.quantity)                               AS units_sold,
    SUM(oi.quantity * oi.unit_price)               AS total_revenue,
    ROUND(AVG(oi.quantity * oi.unit_price), 2)     AS avg_line_value,
    COUNT(DISTINCT o.customer_id)                  AS unique_customers
FROM orders o
JOIN order_items oi  ON oi.order_id = o.order_id
JOIN products p       ON p.product_id = oi.product_id
JOIN categories cat    ON cat.category_id = p.category_id
WHERE o.status NOT IN ('cancelled')
GROUP BY DATE_TRUNC('month', o.order_date), cat.category_name
WITH DATA;
```

สร้าง unique index บน materialized view เพื่อรองรับ `REFRESH MATERIALIZED VIEW CONCURRENTLY` (ทบทวน Part 036):

```sql
CREATE UNIQUE INDEX idx_monthly_sales_mv_month_cat
    ON monthly_sales_mv (sales_month, category_name);
```

ทดสอบ query dashboard:

```sql
SELECT sales_month, category_name, order_count, units_sold, total_revenue
FROM monthly_sales_mv
WHERE sales_month = '2026-01-01'
ORDER BY total_revenue DESC;
```

```
 sales_month | category_name | order_count | units_sold | total_revenue
-------------+---------------+-------------+------------+----------------
 2026-01-01  | Laptops       |           2 |          2 |       58790.00
 2026-01-01  | Smartphones   |           1 |          1 |       25900.00
 2026-01-01  | Accessories   |           1 |          1 |         890.00
 2026-01-01  | Cables        |           3 |          5 |         950.00
 2026-01-01  | Men's Clothing|           1 |          2 |        1180.00
 2026-01-01  | Women's Clothing|         1 |          1 |         790.00
```

### 394.3 กลยุทธ์การ REFRESH

| กลยุทธ์ | คำสั่ง | ข้อดี | ข้อเสีย | เหมาะกับ |
|---|---|---|---|---|
| Full refresh (ล็อกอ่าน) | `REFRESH MATERIALIZED VIEW monthly_sales_mv;` | ง่าย เร็วสำหรับข้อมูลเล็ก | ล็อกไม่ให้ `SELECT` ระหว่าง refresh (ตาราง unreadable ชั่วคราว) | รันตอนดึกที่ไม่มีคนใช้งาน dashboard |
| Concurrent refresh | `REFRESH MATERIALIZED VIEW CONCURRENTLY monthly_sales_mv;` | ยังคง `SELECT` อ่านข้อมูลเก่าได้ระหว่าง refresh | ช้ากว่า full refresh, ต้องมี unique index | dashboard ที่ต้องพร้อมใช้งานตลอด 24 ชม. |
| Scheduled ผ่าน `pg_cron` หรือ cron job ภายนอก | เรียก `REFRESH ... CONCURRENTLY` ตามตารางเวลา | อัตโนมัติ ไม่ต้องพึ่งคน | ข้อมูลจะ "ล่าช้า" ได้สูงสุดเท่าความถี่ของ schedule | รายงานรายวัน/รายชั่วโมง |

```sql
-- Refresh แบบไม่ล็อก (แนะนำสำหรับ production dashboard ที่ต้องพร้อมใช้ตลอดเวลา)
REFRESH MATERIALIZED VIEW CONCURRENTLY monthly_sales_mv;
```

ตัวอย่างการตั้งเวลารีเฟรชอัตโนมัติด้วย extension `pg_cron` (ต้องติดตั้งเพิ่มเติม ใช้เป็นแนวทางอ้างอิง จะสอนละเอียดในระดับ Advanced):

```sql
-- ตัวอย่างเท่านั้น (ต้องมี pg_cron ติดตั้งในระบบก่อน)
-- SELECT cron.schedule('refresh-monthly-sales', '0 2 * * *',
--     'REFRESH MATERIALIZED VIEW CONCURRENTLY monthly_sales_mv;');
```

### 394.4 ทดสอบว่าข้อมูลใน Materialized View ไม่อัปเดตอัตโนมัติ

```sql
-- เพิ่มออเดอร์ใหม่ (ทดสอบ)
BEGIN;
INSERT INTO orders (customer_id, employee_id, status, ship_country)
VALUES (2, 4, 'completed', 'Thailand')
RETURNING order_id;
-- สมมติได้ order_id = 26
INSERT INTO order_items (order_id, product_id, quantity, unit_price)
VALUES (26, 9, 5, 190.00);
COMMIT;

-- ข้อมูลใหม่ยังไม่ปรากฏใน materialized view จนกว่าจะ REFRESH
SELECT SUM(units_sold) FROM monthly_sales_mv WHERE category_name = 'Cables';
-- ผลลัพธ์: ยังเป็นค่าเดิม เพราะยังไม่ REFRESH

REFRESH MATERIALIZED VIEW CONCURRENTLY monthly_sales_mv;

SELECT SUM(units_sold) FROM monthly_sales_mv WHERE category_name = 'Cables';
-- ผลลัพธ์: ค่าใหม่ที่รวมออเดอร์ 26 แล้ว
```

> **บทเรียนสำคัญ**: นี่คือ trade-off หลักของ materialized view — **ความเร็วในการอ่าน แลกกับ ความสดใหม่ของข้อมูล (staleness)** ผู้ออกแบบระบบต้องเลือกความถี่การ refresh ให้เหมาะสมกับ SLA ของรายงานนั้น ๆ

---

## Step 395: Recursive CTE — ยอดขายรวมของ Category Tree ทุกระดับ

นี่คือจุดที่เรานำ **Recursive CTE (Part 026)** มาผสานกับ **Aggregate Functions (Part 027)** เพื่อแก้ปัญหาที่ view ธรรมดาทำไม่ได้ง่าย ๆ: การคำนวณยอดขายของหมวดหมู่ระดับบนสุด (เช่น "Electronics") ให้รวมยอดขายของหมวดหมู่ลูกและหลานทุกระดับเข้าไปด้วย

### 395.1 ทบทวนโครงสร้าง category tree

```sql
WITH RECURSIVE category_tree AS (
    -- Anchor: หมวดหมู่ราก (ไม่มี parent)
    SELECT category_id, category_name, parent_category_id, 0 AS depth,
           category_name::TEXT AS path
    FROM categories
    WHERE parent_category_id IS NULL

    UNION ALL

    -- Recursive: หมวดหมู่ลูกของแต่ละระดับ
    SELECT c.category_id, c.category_name, c.parent_category_id, ct.depth + 1,
           ct.path || ' > ' || c.category_name
    FROM categories c
    JOIN category_tree ct ON c.parent_category_id = ct.category_id
)
SELECT category_id, REPEAT('  ', depth) || category_name AS category_display, depth, path
FROM category_tree
ORDER BY path;
```

```
 category_id |      category_display      | depth |                path
-------------+-----------------------------+-------+-------------------------------------
           1 | Electronics                 |     0 | Electronics
           2 |   Computers                 |     1 | Electronics > Computers
           4 |     Desktops                |     2 | Electronics > Computers > Desktops
           3 |     Laptops                 |     2 | Electronics > Computers > Laptops
           6 |   Accessories               |     1 | Electronics > Accessories
           7 |     Cables                  |     2 | Electronics > Accessories > Cables
           8 |     Chargers                |     2 | Electronics > Accessories > Chargers
           5 |   Smartphones               |     1 | Electronics > Smartphones
           9 | Home & Kitchen              |     0 | Home & Kitchen
          11 |   Appliances                |     1 | Home & Kitchen > Appliances
          12 |     Small Appliances        |     2 | Home & Kitchen > Appliances > Small Appliances
          10 |   Furniture                 |     1 | Home & Kitchen > Furniture
          13 | Fashion                     |     0 | Fashion
          14 |   Men's Clothing            |     1 | Fashion > Men's Clothing
          15 |   Women's Clothing          |     1 | Fashion > Women's Clothing
```

### 395.2 คำนวณยอดขายรวมของทุกหมวดหมู่ (รวมลูกหลานทุกระดับ)

เทคนิคหลัก: สำหรับหมวดหมู่แต่ละตัว เราต้อง "ไล่ลงไปหาลูกหลานทั้งหมด" ก่อน แล้วจึงนำ `product_id` ทั้งหมดในหมวดหมู่นั้น (รวมลูกหลาน) ไป join กับยอดขายจริง วิธีที่สะอาดที่สุดคือสร้าง recursive CTE ที่คืนคู่ `(ancestor_category_id, descendant_category_id)` ทุกคู่ที่เป็นไปได้

```sql
WITH RECURSIVE category_closure AS (
    -- Anchor: หมวดหมู่ตัวเองก็นับเป็น "ลูกหลานของตัวเอง" (depth 0)
    SELECT category_id AS ancestor_id, category_id AS descendant_id, 0 AS depth
    FROM categories

    UNION ALL

    -- Recursive: ไล่หาลูกของทุกหมวดหมู่ที่เจอแล้ว
    SELECT cc.ancestor_id, c.category_id, cc.depth + 1
    FROM category_closure cc
    JOIN categories c ON c.parent_category_id = cc.descendant_id
),
category_sales AS (
    SELECT
        cc.ancestor_id AS category_id,
        SUM(oi.quantity * oi.unit_price) AS total_revenue,
        SUM(oi.quantity)                 AS total_units,
        COUNT(DISTINCT o.order_id)       AS order_count
    FROM category_closure cc
    JOIN products p        ON p.category_id = cc.descendant_id
    JOIN order_items oi     ON oi.product_id = p.product_id
    JOIN orders o            ON o.order_id = oi.order_id AND o.status NOT IN ('cancelled')
    GROUP BY cc.ancestor_id
)
SELECT
    cat.category_id,
    cat.category_name,
    cat.parent_category_id,
    COALESCE(cs.order_count, 0)   AS order_count,
    COALESCE(cs.total_units, 0)   AS total_units,
    COALESCE(cs.total_revenue, 0) AS total_revenue_incl_subcategories
FROM categories cat
LEFT JOIN category_sales cs ON cs.category_id = cat.category_id
ORDER BY cat.parent_category_id NULLS FIRST, cat.category_id;
```

```
 category_id |  category_name  | parent_category_id | order_count | total_units | total_revenue_incl_subcategories
-------------+------------------+---------------------+-------------+--------------+-----------------------------------
           1 | Electronics      |                NULL |          20 |           26 |                        392270.00
           9 | Home & Kitchen   |                NULL |           9 |           12 |                         66200.00
          13 | Fashion          |                NULL |           5 |           11 |                          8400.00
           2 | Computers        |                   1 |          10 |           11 |                        281800.00
           5 | Smartphones      |                   1 |           7 |           10 |                         77700.00
           6 | Accessories      |                   1 |          10 |           16 |                         32770.00
           3 | Laptops          |                   2 |           7 |            8 |                        226900.00
           4 | Desktops         |                   2 |           2 |            2 |                         48800.00
           7 | Cables           |                   6 |           5 |           12 |                          3220.00
           8 | Chargers         |                   6 |           4 |            3 |                          2270.00
          10 | Furniture        |                   9 |           4 |            4 |                         15470.00
          11 | Appliances       |                   9 |           5 |            8 |                         50730.00
          12 | Small Appliances |                  11 |           3 |            5 |                          8060.00
          14 | Men's Clothing   |                  13 |           3 |            6 |                          3660.00
          15 | Women's Clothing |                  13 |           3 |            5 |                          4740.00
```

> **สังเกต Electronics**: ยอดขาย 392,270.00 บาท เป็นผลรวมของ Computers (281,800) + Smartphones (77,700) + Accessories (32,770) ซึ่งแต่ละตัวก็รวมลูกหลานของตัวเองมาแล้วเช่นกัน (Computers รวม Laptops + Desktops) — นี่คือพลังของเทคนิค **transitive closure ผ่าน recursive CTE** ที่ไม่สามารถทำได้ง่าย ๆ ด้วย `GROUP BY` ธรรมดา

### 395.3 ระวัง: การนับซ้ำ (double-counting) ถ้าไม่ระวัง

ข้อผิดพลาดที่พบบ่อย: มือใหม่มักจะ `SUM()` ยอดขายของหมวดหมู่แม่ + ยอดขายของหมวดหมู่ลูกที่ query แยกกัน แล้วนำมาบวกอีกที ทำให้ยอดขายของ "Computers" ถูกนับซ้ำใน "Electronics" สองครั้ง เทคนิค `category_closure` ข้างต้นแก้ปัญหานี้ได้อย่างถูกต้องเพราะแต่ละ `product_id` จะถูกนับเข้ากลุ่ม `ancestor_id` แค่ครั้งเดียวต่อหนึ่งระดับบรรพบุรุษเท่านั้น (ไม่ใช่การบวกยอดของลูกเข้ากับยอดของแม่ซ้ำสอง)

---

## Step 396: Window Function ขั้นสูง — Segmentation, MoM Growth, Running Total

### 396.1 Customer Segmentation ด้วย `NTILE`

แบ่งลูกค้าออกเป็น 4 กลุ่ม (quartile) ตามมูลค่าการซื้อสะสม เพื่อทำ targeted marketing (ทบทวน Part 030):

```sql
WITH customer_totals AS (
    SELECT
        c.customer_id,
        c.first_name || ' ' || c.last_name AS customer_name,
        c.country,
        COALESCE(SUM(oi.quantity * oi.unit_price), 0) AS total_spent
    FROM customers c
    LEFT JOIN orders o        ON o.customer_id = c.customer_id AND o.status = 'completed'
    LEFT JOIN order_items oi  ON oi.order_id = o.order_id
    GROUP BY c.customer_id, c.first_name, c.last_name, c.country
)
SELECT
    customer_name,
    country,
    total_spent,
    NTILE(4) OVER (ORDER BY total_spent DESC) AS spend_quartile,
    CASE NTILE(4) OVER (ORDER BY total_spent DESC)
        WHEN 1 THEN 'VIP (Platinum)'
        WHEN 2 THEN 'Gold'
        WHEN 3 THEN 'Silver'
        ELSE 'Bronze / New'
    END AS segment
FROM customer_totals
ORDER BY total_spent DESC;
```

```
   customer_name  | country  | total_spent | spend_quartile |    segment
-------------------+----------+-------------+-----------------+----------------
 Somchai Jaidee    | Thailand |    66610.00 |               1 | VIP (Platinum)
 Wei Zhang         | China    |    54900.00 |               1 | VIP (Platinum)
 Yuki Tanaka       | Japan    |    33470.00 |               1 | VIP (Platinum)
 Chai Thongdee     | Thailand |    27380.00 |               1 | VIP (Platinum)
 John Smith        | USA      |    24900.00 |               1 | VIP (Platinum)
 David Miller      | USA      |    25900.00 |               2 | Gold
 ...               | ...      |         ... |             ... | ...
 Napat Chaiyaporn  | Thailand |        0.00 |               4 | Bronze / New
 Benjawan Silapat  | Thailand |        0.00 |               4 | Bronze / New
```

> หมายเหตุ: `CASE NTILE(4) OVER (...) WHEN ...` เรียก window function ซ้ำในนิพจน์เดียวกันได้ แต่ในการใช้งานจริง แนะนำให้ครอบด้วย subquery หรือ CTE แล้วอ้างอิง alias เพื่อประสิทธิภาพที่ดีกว่า (PostgreSQL query planner จะคำนวณ window function ซ้ำสองครั้งถ้าเขียนแบบนี้ตรง ๆ)

เวอร์ชันที่มีประสิทธิภาพดีกว่า (คำนวณ window function ครั้งเดียว):

```sql
WITH customer_totals AS (
    SELECT
        c.customer_id,
        c.first_name || ' ' || c.last_name AS customer_name,
        COALESCE(SUM(oi.quantity * oi.unit_price), 0) AS total_spent
    FROM customers c
    LEFT JOIN orders o        ON o.customer_id = c.customer_id AND o.status = 'completed'
    LEFT JOIN order_items oi  ON oi.order_id = o.order_id
    GROUP BY c.customer_id, c.first_name, c.last_name
),
segmented AS (
    SELECT *, NTILE(4) OVER (ORDER BY total_spent DESC) AS spend_quartile
    FROM customer_totals
)
SELECT customer_name, total_spent, spend_quartile,
       CASE spend_quartile
           WHEN 1 THEN 'VIP (Platinum)'
           WHEN 2 THEN 'Gold'
           WHEN 3 THEN 'Silver'
           ELSE 'Bronze / New'
       END AS segment
FROM segmented
ORDER BY total_spent DESC;
```

### 396.2 Month-over-Month Growth ด้วย `LAG`

```sql
WITH monthly_revenue AS (
    SELECT
        DATE_TRUNC('month', o.order_date)::DATE AS sales_month,
        SUM(oi.quantity * oi.unit_price) AS revenue
    FROM orders o
    JOIN order_items oi ON oi.order_id = o.order_id
    WHERE o.status NOT IN ('cancelled')
    GROUP BY DATE_TRUNC('month', o.order_date)
)
SELECT
    sales_month,
    revenue,
    LAG(revenue) OVER (ORDER BY sales_month) AS prev_month_revenue,
    revenue - LAG(revenue) OVER (ORDER BY sales_month) AS mom_change,
    ROUND(
        100.0 * (revenue - LAG(revenue) OVER (ORDER BY sales_month))
        / NULLIF(LAG(revenue) OVER (ORDER BY sales_month), 0)
    , 2) AS mom_growth_pct
FROM monthly_revenue
ORDER BY sales_month;
```

```
 sales_month |  revenue  | prev_month_revenue | mom_change | mom_growth_pct
-------------+-----------+----------------------+-------------+------------------
 2026-01-01  |  87200.00 |                 NULL |        NULL |            NULL
 2026-02-01  |  42470.00 |             87200.00 |  -44730.00 |          -51.30
 2026-03-01  | 106680.00 |             42470.00 |   64210.00 |          151.19
 2026-04-01  |  55670.00 |            106680.00 |  -51010.00 |          -47.82
 2026-05-01  |  40140.00 |             55670.00 |  -15530.00 |          -27.90
 2026-06-01  |  44160.00 |             40140.00 |    4020.00 |           10.02
 2026-07-01  |   7540.00 |             44160.00 |  -36620.00 |          -82.92
 2026-08-01  |  46680.00 |              7540.00 |   39140.00 |          519.10
 2026-09-01  |  18900.00 |             46680.00 |  -27780.00 |          -59.51
```

> ใช้ `NULLIF(..., 0)` ป้องกัน division by zero เมื่อเดือนก่อนหน้าไม่มียอดขายเลย และเดือนแรก (`2026-01`) จะได้ `NULL` เพราะไม่มีเดือนก่อนหน้าให้เทียบ — พฤติกรรมที่ถูกต้องตามธุรกิจจริง

### 396.3 Running Total (ยอดสะสม) ของยอดขายรายเดือน

```sql
WITH monthly_revenue AS (
    SELECT
        DATE_TRUNC('month', o.order_date)::DATE AS sales_month,
        SUM(oi.quantity * oi.unit_price) AS revenue
    FROM orders o
    JOIN order_items oi ON oi.order_id = o.order_id
    WHERE o.status NOT IN ('cancelled')
    GROUP BY DATE_TRUNC('month', o.order_date)
)
SELECT
    sales_month,
    revenue,
    SUM(revenue) OVER (ORDER BY sales_month
                        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_total,
    ROUND(AVG(revenue) OVER (ORDER BY sales_month
                        ROWS BETWEEN 2 PRECEDING AND CURRENT ROW), 2)     AS moving_avg_3mo
FROM monthly_revenue
ORDER BY sales_month;
```

```
 sales_month |  revenue  | running_total | moving_avg_3mo
-------------+-----------+-----------------+------------------
 2026-01-01  |  87200.00 |        87200.00 |        87200.00
 2026-02-01  |  42470.00 |       129670.00 |        64835.00
 2026-03-01  | 106680.00 |       236350.00 |        78783.33
 2026-04-01  |  55670.00 |       292020.00 |        68273.33
 2026-05-01  |  40140.00 |       332160.00 |        67496.67
 2026-06-01  |  44160.00 |       376320.00 |        46656.67
 2026-07-01  |   7540.00 |       383860.00 |        30613.33
 2026-08-01  |  46680.00 |       430540.00 |        32793.33
 2026-09-01  |  18900.00 |       449440.00 |        24373.33
```

> `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` คือ running total แบบมาตรฐาน ส่วน `ROWS BETWEEN 2 PRECEDING AND CURRENT ROW` คือ moving average 3 เดือน (เดือนปัจจุบัน + 2 เดือนก่อนหน้า) — ทั้งสอง frame clause นี้ทบทวนจาก Part 030

### 396.4 รวมทุกเทคนิคในคิวรีเดียว: Rank สินค้าขายดีในแต่ละหมวดหมู่ (Top-N per group)

```sql
WITH product_sales AS (
    SELECT
        p.product_id,
        p.product_name,
        cat.category_name,
        SUM(oi.quantity * oi.unit_price) AS revenue,
        SUM(oi.quantity)                 AS units_sold
    FROM products p
    JOIN categories cat ON cat.category_id = p.category_id
    JOIN order_items oi ON oi.product_id = p.product_id
    JOIN orders o        ON o.order_id = oi.order_id AND o.status NOT IN ('cancelled')
    GROUP BY p.product_id, p.product_name, cat.category_name
),
ranked AS (
    SELECT *,
           ROW_NUMBER() OVER (PARTITION BY category_name ORDER BY revenue DESC) AS rn
    FROM product_sales
)
SELECT category_name, product_name, units_sold, revenue
FROM ranked
WHERE rn <= 2  -- top 2 ต่อหมวดหมู่
ORDER BY category_name, revenue DESC;
```

```
 category_name |      product_name       | units_sold |  revenue
----------------+--------------------------+------------+-----------
 Cables         | HDMI Cable 4K 3m         |          2 |    580.00
 Cables         | USB-C Cable 2m           |          5 |    950.00
 Chargers       | 65W GaN Charger          |          2 |   1780.00
 Chargers       | Wireless Charging Pad    |          1 |    690.00
 Desktops       | PowerStation Desktop i7  |          1 |  29900.00
 Desktops       | OfficeMate Desktop i5    |          1 |  18900.00
 Laptops        | NovaPhone X12            |          3 |  77700.00
 ...            | ...                      |        ... |       ...
```

> **`ROW_NUMBER() OVER (PARTITION BY ... ORDER BY ...)`** คือรูปแบบมาตรฐานของโจทย์ "Top-N per group" ซึ่งเป็นหนึ่งในโจทย์ window function ที่ถูกถามบ่อยที่สุดในการสัมภาษณ์งานสาย data

---

## Step 397: Transaction สำหรับ Checkout Process แบบสมบูรณ์

นี่คือหัวใจของระบบ e-commerce ทุกระบบ: กระบวนการ **checkout** ต้องเป็น atomic — ถ้าขั้นตอนใดขั้นตอนหนึ่งล้มเหลว (เช่น สต๊อกไม่พอ) ทุกอย่างต้อง rollback กลับหมด ไม่ปล่อยให้เกิดสถานะครึ่ง ๆ กลาง ๆ เช่น สร้างออเดอร์แล้วแต่ไม่ได้ตัดสต๊อก (ทบทวน ACID จาก Part 037)

### 397.1 ขั้นตอนของ checkout process

1. ตรวจสอบว่าสินค้ามีสต๊อกเพียงพอ
2. `INSERT` ลง `orders`
3. `INSERT` ลง `order_items` สำหรับแต่ละรายการสินค้าในตะกร้า
4. `UPDATE` ตัดสต๊อกใน `products`
5. `INSERT` ลง `payments` เพื่อบันทึกการชำระเงิน
6. ถ้าทุกอย่างสำเร็จ → `COMMIT`; ถ้ามีจุดใดล้มเหลว → `ROLLBACK` (หรือ `ROLLBACK TO SAVEPOINT` เฉพาะจุด)

### 397.2 Checkout แบบสมบูรณ์ (Happy Path)

```sql
BEGIN;

-- Step 1: ตรวจสอบสต๊อกก่อน (ลูกค้าต้องการซื้อ USB-C Cable 2m จำนวน 10 ชิ้น = product_id 9)
DO $$
DECLARE
    v_stock INTEGER;
BEGIN
    SELECT stock_quantity INTO v_stock FROM products WHERE product_id = 9 FOR UPDATE;
    IF v_stock < 10 THEN
        RAISE EXCEPTION 'สต๊อกไม่เพียงพอ: เหลือ % ชิ้น ต้องการ 10 ชิ้น', v_stock;
    END IF;
END $$;

-- Step 2: สร้างคำสั่งซื้อใหม่
INSERT INTO orders (customer_id, employee_id, status, ship_country)
VALUES (4, 6, 'completed', 'Thailand')
RETURNING order_id \gset
-- ในการรันจริงผ่านแอปพลิเคชัน ให้เก็บค่า order_id ที่ได้ไว้ใช้ในขั้นตอนถัดไป
-- สมมติในตัวอย่างนี้ order_id ที่ได้คือ 27

SAVEPOINT before_items;

-- Step 3: เพิ่มรายการสินค้า
INSERT INTO order_items (order_id, product_id, quantity, unit_price)
VALUES (27, 9, 10, 190.00);

-- Step 4: ตัดสต๊อก
UPDATE products SET stock_quantity = stock_quantity - 10 WHERE product_id = 9;

-- Step 5: บันทึกการชำระเงิน
INSERT INTO payments (order_id, amount, payment_method)
VALUES (27, 1900.00, 'promptpay');

-- ทุกขั้นตอนสำเร็จ -> ยืนยันธุรกรรม
COMMIT;
```

```
NOTICE: (ไม่มี exception เกิดขึ้น เพราะสต๊อกเพียงพอ)
INSERT 0 1
COMMIT
```

### 397.3 Checkout ที่ล้มเหลว พร้อม `SAVEPOINT` และ error handling ด้วย PL/pgSQL block

ในสถานการณ์จริง ระบบอาจต้องการ "ลองสินค้าตัวสำรอง (backup item)" หากสินค้าหลักหมดสต๊อก โดยไม่ต้องการยกเลิกทั้งออเดอร์ นี่คือจุดที่ `SAVEPOINT` มีประโยชน์มาก (ทบทวน Part 037):

```sql
BEGIN;

INSERT INTO orders (customer_id, employee_id, status, ship_country)
VALUES (17, 12, 'completed', 'Thailand')
RETURNING order_id;
-- สมมติ order_id = 28

SAVEPOINT before_item_1;

-- ลองสั่ง GameForce Laptop RTX (product_id 3) จำนวน 100 ชิ้น (เกินสต๊อกแน่นอน เพราะมีแค่ 20 ชิ้น)
DO $$
DECLARE
    v_stock INTEGER;
BEGIN
    SELECT stock_quantity INTO v_stock FROM products WHERE product_id = 3 FOR UPDATE;
    IF v_stock < 100 THEN
        RAISE EXCEPTION 'STOCK_INSUFFICIENT';
    END IF;
    INSERT INTO order_items (order_id, product_id, quantity, unit_price)
    VALUES (28, 3, 100, 54900.00);
    UPDATE products SET stock_quantity = stock_quantity - 100 WHERE product_id = 3;
END $$;
-- ผลลัพธ์: ERROR: STOCK_INSUFFICIENT
-- transaction เข้าสู่สถานะ aborted สำหรับคำสั่งถัดไป จนกว่าจะ ROLLBACK TO SAVEPOINT

ROLLBACK TO SAVEPOINT before_item_1;
-- transaction กลับมาใช้งานได้ปกติ (ยกเลิกเฉพาะส่วนหลัง savepoint)

-- ลองสินค้าตัวสำรอง: UltraBook Air 13" (product_id 2) จำนวน 1 ชิ้นแทน
SAVEPOINT before_item_2;

DO $$
DECLARE
    v_stock INTEGER;
BEGIN
    SELECT stock_quantity INTO v_stock FROM products WHERE product_id = 2 FOR UPDATE;
    IF v_stock < 1 THEN
        RAISE EXCEPTION 'STOCK_INSUFFICIENT';
    END IF;
    INSERT INTO order_items (order_id, product_id, quantity, unit_price)
    VALUES (28, 2, 1, 24900.00);
    UPDATE products SET stock_quantity = stock_quantity - 1 WHERE product_id = 2;
END $$;
-- สำเร็จ (สต๊อกเพียงพอ)

INSERT INTO payments (order_id, amount, payment_method)
VALUES (28, 24900.00, 'credit_card');

COMMIT;
```

```
ERROR:  STOCK_INSUFFICIENT
ROLLBACK TO SAVEPOINT
DO  (สำเร็จ)
INSERT 0 1
COMMIT
```

> **จุดสำคัญ**: `SAVEPOINT` ทำให้ transaction สามารถ "ถอยกลับบางส่วน" ได้โดยไม่ต้อง rollback ทั้งหมด เหมาะกับ business logic ที่มีทางเลือกสำรอง (fallback) ระหว่างขั้นตอน — แต่ต้องระวัง: หลัง `RAISE EXCEPTION` ภายใน `DO` block นอก transaction control ปกติ transaction จะเข้าสถานะ "aborted" ทันทีหากไม่มี exception handler ครอบไว้ ในตัวอย่างนี้เราจับด้วย `SAVEPOINT`/`ROLLBACK TO SAVEPOINT` ที่ระดับ transaction แทน

### 397.4 เวอร์ชันที่แข็งแกร่งกว่า: ใช้ PL/pgSQL exception block ครอบทั้งกระบวนการ (แนะนำสำหรับ production)

```sql
DO $$
DECLARE
    v_order_id   INTEGER;
    v_stock      INTEGER;
    v_product_id INTEGER := 9;
    v_qty        INTEGER := 15;
    v_price      NUMERIC(10,2) := 190.00;
BEGIN
    -- ทุกอย่างใน DO block นี้อยู่ใน transaction เดียวกันโดยอัตโนมัติ
    SELECT stock_quantity INTO v_stock
    FROM products WHERE product_id = v_product_id FOR UPDATE;

    IF v_stock IS NULL THEN
        RAISE EXCEPTION 'ไม่พบสินค้า product_id = %', v_product_id;
    ELSIF v_stock < v_qty THEN
        RAISE EXCEPTION 'สต๊อกไม่เพียงพอ: เหลือ % ต้องการ %', v_stock, v_qty;
    END IF;

    INSERT INTO orders (customer_id, employee_id, status, ship_country)
    VALUES (6, 10, 'completed', 'Thailand')
    RETURNING order_id INTO v_order_id;

    INSERT INTO order_items (order_id, product_id, quantity, unit_price)
    VALUES (v_order_id, v_product_id, v_qty, v_price);

    UPDATE products SET stock_quantity = stock_quantity - v_qty
    WHERE product_id = v_product_id;

    INSERT INTO payments (order_id, amount, payment_method)
    VALUES (v_order_id, v_qty * v_price, 'credit_card');

    RAISE NOTICE 'สร้างคำสั่งซื้อสำเร็จ order_id = %', v_order_id;

EXCEPTION
    WHEN OTHERS THEN
        RAISE NOTICE 'Checkout ล้มเหลว: % — กำลัง rollback ธุรกรรมทั้งหมด', SQLERRM;
        RAISE; -- ส่งต่อ exception เพื่อให้ transaction ทั้งก้อน rollback
END $$;
```

```
NOTICE:  สร้างคำสั่งซื้อสำเร็จ order_id = 29
DO
```

> **ทำไมใช้ `FOR UPDATE`**: การ `SELECT ... FOR UPDATE` ล็อกแถวสินค้านั้นไว้จนกว่า transaction จะจบ ป้องกันไม่ให้ transaction อื่นอ่านค่าสต๊อกเดิมไปคำนวณซ้อนกันในช่วงเวลาเดียวกัน — นี่คือรากฐานของ Step 398 ที่เราจะทดสอบเรื่อง concurrency โดยตรง

---

## Step 398: ทดสอบ Concurrency — จำลองสถานการณ์ Overselling

โจทย์คลาสสิกของระบบร้านค้าออนไลน์: สินค้าเหลือสต๊อก 1 ชิ้น แต่ลูกค้าสองคนกดสั่งซื้อ **พร้อมกัน** จะเกิดอะไรขึ้นถ้าไม่ป้องกัน? (ทบทวน Part 038 เรื่อง isolation levels)

### 398.1 เตรียมสถานการณ์: สินค้าเหลือสต๊อกน้อย

```sql
-- ตั้งสถานการณ์: Robot Vacuum Cleaner (product_id 17) เหลือสต๊อกแค่ 1 ชิ้น
UPDATE products SET stock_quantity = 1 WHERE product_id = 17;
```

### 398.2 สถานการณ์ที่ 1 — ไม่มีการล็อก (เกิด overselling ได้จริง)

เปิด 2 session พร้อมกัน (Session A และ Session B) จำลองลูกค้า 2 คนกดสั่งซื้อพร้อมกัน โดย**ไม่ใช้** `FOR UPDATE`:

```sql
-- ================== Session A ==================
BEGIN;
SELECT stock_quantity FROM products WHERE product_id = 17;
-- อ่านได้ stock_quantity = 1  (ยังไม่ commit)

-- ================== Session B (รันขณะที่ A ยังไม่ commit) ==================
BEGIN;
SELECT stock_quantity FROM products WHERE product_id = 17;
-- อ่านได้ stock_quantity = 1  เช่นกัน! (เพราะ A ยังไม่ได้ UPDATE)

-- ================== Session A (ดำเนินการต่อ) ==================
UPDATE products SET stock_quantity = stock_quantity - 1 WHERE product_id = 17;
COMMIT;
-- stock_quantity กลายเป็น 0

-- ================== Session B (ดำเนินการต่อ โดยไม่รู้ว่า A commit ไปแล้ว) ==================
UPDATE products SET stock_quantity = stock_quantity - 1 WHERE product_id = 17;
COMMIT;
-- stock_quantity กลายเป็น -1  !! OVERSOLD !!
```

```sql
SELECT stock_quantity FROM products WHERE product_id = 17;
```

```
 stock_quantity
-----------------
              -1
```

> **ปัญหาที่เกิดขึ้น**: ทั้งสอง session อ่านค่า `stock_quantity = 1` พร้อมกันก่อนที่ฝั่งใดฝั่งหนึ่งจะ `UPDATE` เพราะ `UPDATE ... SET x = x - 1` แต่ละ statement คำนวณจากค่าที่ "แคช" ไว้ตอน `SELECT` ไม่ใช่ค่าล่าสุด ทำให้ทั้งสอง transaction ต่างคิดว่าตัวเอง "ตัดสต๊อกจาก 1 เหลือ 0" ได้สำเร็จ แต่ความจริงตัดซ้ำกันสองครั้งจนติดลบ — นี่คือ **lost update anomaly** ที่สอนใน Part 038

### 398.3 วิธีแก้ที่ 1 — ใช้ `SELECT ... FOR UPDATE` (Row-level Lock)

```sql
-- รีเซ็ตสต๊อกกลับเป็น 1 เพื่อทดสอบใหม่
UPDATE products SET stock_quantity = 1 WHERE product_id = 17;

-- ================== Session A ==================
BEGIN;
SELECT stock_quantity FROM products WHERE product_id = 17 FOR UPDATE;
-- อ่านได้ 1, และ "ล็อก" แถวนี้ไว้ ห้าม transaction อื่นอ่าน FOR UPDATE แถวเดียวกันจนกว่า A จะ commit/rollback

-- ================== Session B (พยายามอ่านพร้อมกัน) ==================
BEGIN;
SELECT stock_quantity FROM products WHERE product_id = 17 FOR UPDATE;
-- Session B จะ "ค้าง (block)" รอ Session A ปลดล็อกก่อน ไม่สามารถอ่านต่อได้ทันที

-- ================== Session A (ดำเนินการต่อ) ==================
UPDATE products SET stock_quantity = stock_quantity - 1 WHERE product_id = 17;
COMMIT;
-- ปลดล็อก -> stock_quantity = 0

-- ================== Session B (ตอนนี้ได้ทำงานต่อ อ่านค่าล่าสุดหลัง A commit) ==================
-- SELECT ... FOR UPDATE ที่ค้างอยู่ จะกลับมาทำงานต่อ และอ่านได้ stock_quantity = 0 (ค่าล่าสุด)
IF -- (ในแอปพลิเคชันจริง ตรวจสอบด้วยโค้ด: ถ้า stock_quantity < 1 ให้แจ้งลูกค้าว่าสินค้าหมด)
COMMIT; -- หรือ ROLLBACK หากตัดสินใจไม่ขาย
```

```
 stock_quantity (final)
-------------------------
                       0
```

> `SELECT ... FOR UPDATE` แก้ปัญหาได้เพราะบังคับให้ transaction ที่สองต้อง **รอ** จนกว่า transaction แรกจะ commit และปลดล็อกแถวนั้น ทำให้ transaction ที่สองอ่านค่าสต๊อกที่เป็นปัจจุบันจริง ๆ (0) แทนที่จะอ่านค่าเก่าที่ค้างอยู่ (1)

### 398.4 วิธีแก้ที่ 2 — ใช้ Isolation Level `SERIALIZABLE`

อีกวิธีคือยกระดับ isolation level ทั้ง transaction ให้เป็น `SERIALIZABLE` ซึ่ง PostgreSQL จะตรวจจับความขัดแย้งแบบ serialization และ**ปฏิเสธ** transaction ที่มาทีหลังด้วย error แทนที่จะปล่อยให้ข้อมูลเพี้ยน:

```sql
-- รีเซ็ตสต๊อกกลับเป็น 1
UPDATE products SET stock_quantity = 1 WHERE product_id = 17;

-- ================== Session A ==================
BEGIN ISOLATION LEVEL SERIALIZABLE;
SELECT stock_quantity FROM products WHERE product_id = 17;
-- อ่านได้ 1

-- ================== Session B ==================
BEGIN ISOLATION LEVEL SERIALIZABLE;
SELECT stock_quantity FROM products WHERE product_id = 17;
-- อ่านได้ 1 เช่นกัน

-- ================== Session A ==================
UPDATE products SET stock_quantity = stock_quantity - 1 WHERE product_id = 17;
COMMIT;
-- สำเร็จ

-- ================== Session B ==================
UPDATE products SET stock_quantity = stock_quantity - 1 WHERE product_id = 17;
COMMIT;
-- ERROR:  could not serialize access due to concurrent update
```

```
ERROR:  could not serialize access due to concurrent update
HINT:  The transaction might succeed if retried.
```

> เมื่อได้ error นี้ แอปพลิเคชันฝั่ง Session B **ต้อง retry ทั้ง transaction ใหม่ทั้งหมด** (อ่านค่าสต๊อกล่าสุดอีกครั้ง) ซึ่งครั้งนี้จะพบว่าสต๊อกเป็น 0 และปฏิเสธการขายอย่างถูกต้อง

### 398.5 เปรียบเทียบสองวิธี — ควรเลือกใช้แบบไหน

| วิธี | กลไก | ข้อดี | ข้อเสีย | เหมาะกับ |
|---|---|---|---|---|
| `SELECT ... FOR UPDATE` | Row-level lock, block รอ | เข้าใจง่าย ควบคุม flow ได้ตรงจุด ไม่ต้องเขียน retry logic | Session ที่รออาจ "ค้าง" นานถ้า transaction แรกใช้เวลานาน (เสี่ยง lock contention) | ระบบ checkout ทั่วไปที่ปริมาณ concurrent ไม่สูงมาก — **แนะนำเป็นค่าเริ่มต้น** |
| `SERIALIZABLE` isolation | ตรวจจับ conflict แล้วปฏิเสธ (fail fast) | ไม่มีการ block รอ ปลอดภัยสูงสุดในทางทฤษฎี | ต้องเขียน retry logic ที่แอปพลิเคชันเสมอ, overhead สูงกว่าเมื่อมี transaction จำนวนมาก | ระบบที่ต้องการความถูกต้องสูงสุดและ throughput สูง เช่น ระบบธนาคาร, ระบบจองที่นั่ง |

> ทบทวน Part 038: PostgreSQL default isolation level คือ `READ COMMITTED` ซึ่ง**ไม่**ป้องกัน lost update โดยอัตโนมัติสำหรับ pattern `UPDATE x = x - 1` ธรรมดา ดังนั้นระบบ e-commerce ที่จัดการสต๊อกจำเป็นต้องเลือกใช้ `FOR UPDATE` หรือ `SERIALIZABLE` อย่างใดอย่างหนึ่งเสมอ ไม่ควรปล่อยให้เป็น default เฉย ๆ

```sql
-- ทำความสะอาด: คืนค่าสต๊อกให้ถูกต้องหลังการทดลอง
UPDATE products SET stock_quantity = 20 WHERE product_id = 17;
```

---

## Step 399: Executive Dashboard Query — รายงานวิเคราะห์ธุรกิจแบบครบวงจร

นี่คือคิวรีที่รวมทุกเทคนิคของระดับกลางไว้ในรายงานเดียว จำลองสถานการณ์จริงที่ผู้บริหารขอ "รายงานสรุปภาพรวมธุรกิจ" ก่อนประชุมบอร์ด

### 399.1 ส่วนที่ 1 — ยอดขายรวมและ KPI หลัก (ใช้ Aggregate + FILTER + CASE)

```sql
SELECT
    COUNT(DISTINCT o.order_id)                                            AS total_orders,
    COUNT(DISTINCT o.order_id) FILTER (WHERE o.status = 'completed')      AS completed_orders,
    COUNT(DISTINCT o.order_id) FILTER (WHERE o.status = 'cancelled')      AS cancelled_orders,
    COUNT(DISTINCT o.order_id) FILTER (WHERE o.status = 'refunded')       AS refunded_orders,
    SUM(oi.quantity * oi.unit_price) FILTER (WHERE o.status = 'completed') AS gross_revenue,
    SUM(oi.quantity * oi.unit_price) FILTER (WHERE o.status = 'refunded') AS refunded_amount,
    ROUND(
        SUM(oi.quantity * oi.unit_price) FILTER (WHERE o.status = 'completed')
        - COALESCE(SUM(oi.quantity * oi.unit_price) FILTER (WHERE o.status = 'refunded'), 0)
    , 2) AS net_revenue,
    COUNT(DISTINCT o.customer_id)                                          AS active_customers,
    ROUND(
        100.0 * COUNT(DISTINCT o.order_id) FILTER (WHERE o.status = 'cancelled')
        / NULLIF(COUNT(DISTINCT o.order_id), 0)
    , 2) AS cancellation_rate_pct
FROM orders o
LEFT JOIN order_items oi ON oi.order_id = o.order_id;
```

```
 total_orders | completed_orders | cancelled_orders | refunded_orders | gross_revenue | refunded_amount | net_revenue | active_customers | cancellation_rate_pct
---------------+-------------------+-------------------+-------------------+-----------------+-------------------+---------------+--------------------+-------------------------
            29 |                26 |                 1 |                 1 |       460670.00 |          21900.00 |    438770.00 |                 17 |                    3.45
```

### 399.2 ส่วนที่ 2 — สินค้าขายดี Top 10 (JOIN + Aggregate + Window Function)

```sql
SELECT
    p.product_name,
    cat.category_name,
    SUM(oi.quantity)                    AS units_sold,
    SUM(oi.quantity * oi.unit_price)    AS revenue,
    RANK() OVER (ORDER BY SUM(oi.quantity * oi.unit_price) DESC) AS rank_by_revenue,
    ROUND(
        100.0 * SUM(oi.quantity * oi.unit_price)
        / SUM(SUM(oi.quantity * oi.unit_price)) OVER ()
    , 2) AS pct_of_total_revenue
FROM order_items oi
JOIN orders o     ON o.order_id = oi.order_id AND o.status = 'completed'
JOIN products p    ON p.product_id = oi.product_id
JOIN categories cat ON cat.category_id = p.category_id
GROUP BY p.product_id, p.product_name, cat.category_name
ORDER BY revenue DESC
LIMIT 10;
```

```
       product_name        | category_name | units_sold |  revenue  | rank_by_revenue | pct_of_total_revenue
-----------------------------+---------------+------------+-----------+-------------------+------------------------
 NovaPhone X12               | Smartphones   |          3 |  77700.00 |                 1 |                 17.71
 GameForce Laptop RTX        | Laptops       |          1 |  54900.00 |                 2 |                 12.51
 UltraBook Pro 14"           | Laptops       |          2 |  65800.00 |                 3 |                 15.00
 UltraBook Air 13"           | Laptops       |          2 |  49800.00 |                 4 |                 11.35
 PowerStation Desktop i7     | Desktops      |          1 |  29900.00 |                 5 |                  6.82
 OfficeMate Desktop i5       | Computers     |          2 |  37800.00 |                 6 |                  8.62
 ...                         | ...           |        ... |       ... |               ... |                    ...
```

> **`SUM(...) OVER ()`** ที่ไม่มี `PARTITION BY` คือผลรวมของทั้งตาราง ใช้เพื่อคำนวณ "สัดส่วนต่อยอดรวม (% of total)" ในบรรทัดเดียวกับ aggregate ปกติ — เป็นเทคนิคยอดนิยมของ dashboard ยอดขาย

### 399.3 ส่วนที่ 3 — ลูกค้า VIP (ผสาน CTE + Window Function + CASE)

```sql
WITH customer_summary AS (
    SELECT
        c.customer_id,
        c.first_name || ' ' || c.last_name AS customer_name,
        c.country,
        c.signup_date,
        COUNT(DISTINCT o.order_id)                     AS order_count,
        COALESCE(SUM(oi.quantity * oi.unit_price), 0)  AS total_spent
    FROM customers c
    LEFT JOIN orders o        ON o.customer_id = c.customer_id AND o.status = 'completed'
    LEFT JOIN order_items oi  ON oi.order_id = o.order_id
    GROUP BY c.customer_id, c.first_name, c.last_name, c.country, c.signup_date
)
SELECT
    customer_name,
    country,
    order_count,
    total_spent,
    RANK() OVER (ORDER BY total_spent DESC) AS spending_rank,
    CASE
        WHEN total_spent >= 40000 THEN 'VIP'
        WHEN total_spent >= 15000 THEN 'Regular'
        WHEN total_spent > 0     THEN 'Occasional'
        ELSE 'Inactive'
    END AS customer_tier
FROM customer_summary
ORDER BY total_spent DESC
LIMIT 10;
```

```
   customer_name  | country  | order_count | total_spent | spending_rank | customer_tier
-------------------+----------+---------------+---------------+-----------------+----------------
 Somchai Jaidee    | Thailand |             3 |     66610.00 |               1 | VIP
 Wei Zhang         | China    |             1 |     54900.00 |               2 | VIP
 Yuki Tanaka       | Japan    |             1 |     33470.00 |               3 | Regular
 Chai Thongdee     | Thailand |             1 |     27380.00 |               4 | Regular
 John Smith        | USA      |             1 |     24900.00 |               5 | Regular
 David Miller      | USA      |             1 |     25900.00 |               6 | Regular
 Pranee Boonmee    | Thailand |             1 |      8990.00 |               7 | Occasional
 Nattaya Pongsri   | Thailand |             1 |     17590.00 |               8 | Regular
 Siriporn Meesuk   | Thailand |             1 |      3230.00 |               9 | Occasional
 Minji Kim         | South Korea|            1 |      3180.00 |              10 | Occasional
```

### 399.4 ส่วนที่ 4 — อัตราการรีวิว (Review Coverage Rate ด้วย LEFT JOIN + Subquery)

```sql
SELECT
    COUNT(DISTINCT oi.product_id) AS products_sold,
    COUNT(DISTINCT r.product_id)  AS products_reviewed,
    ROUND(
        100.0 * COUNT(DISTINCT r.product_id) / NULLIF(COUNT(DISTINCT oi.product_id), 0)
    , 2) AS review_coverage_pct,
    (SELECT ROUND(AVG(rating), 2) FROM reviews) AS avg_rating_overall
FROM order_items oi
JOIN orders o ON o.order_id = oi.order_id AND o.status = 'completed'
LEFT JOIN reviews r ON r.product_id = oi.product_id;
```

```
 products_sold | products_reviewed | review_coverage_pct | avg_rating_overall
-----------------+----------------------+------------------------+-----------------------
             21 |                   16 |                  76.19 |                4.40
```

รายชื่อสินค้าที่ขายแล้วแต่ยังไม่มีรีวิว (ใช้ `LEFT JOIN ... WHERE IS NULL` ทบทวน Part 022):

```sql
SELECT DISTINCT p.product_name
FROM order_items oi
JOIN orders o     ON o.order_id = oi.order_id AND o.status = 'completed'
JOIN products p    ON p.product_id = oi.product_id
LEFT JOIN reviews r ON r.product_id = p.product_id
WHERE r.review_id IS NULL
ORDER BY p.product_name;
```

```
      product_name
--------------------------
 65W GaN Charger
 Bookshelf 5-Tier
 Electric Kettle 1.7L
 HDMI Cable 4K 3m
 Men's Denim Jacket
```

### 399.5 ส่วนที่ 5 — ยอดขายแยกตามพนักงาน พร้อมสายบังคับบัญชา (Self Join + Aggregate)

```sql
SELECT
    e.employee_id,
    e.first_name || ' ' || e.last_name AS employee_name,
    e.department,
    mgr.first_name || ' ' || mgr.last_name AS manager_name,
    COUNT(DISTINCT o.order_id)                         AS orders_handled,
    COALESCE(SUM(oi.quantity * oi.unit_price), 0)       AS revenue_generated,
    RANK() OVER (ORDER BY COALESCE(SUM(oi.quantity * oi.unit_price), 0) DESC) AS sales_rank
FROM employees e
LEFT JOIN employees mgr   ON mgr.employee_id = e.manager_id
LEFT JOIN orders o        ON o.employee_id = e.employee_id AND o.status = 'completed'
LEFT JOIN order_items oi  ON oi.order_id = o.order_id
WHERE e.department = 'Sales'
GROUP BY e.employee_id, e.first_name, e.last_name, e.department, mgr.first_name, mgr.last_name
ORDER BY revenue_generated DESC;
```

```
 employee_id | employee_name    | department |  manager_name  | orders_handled | revenue_generated | sales_rank
--------------+--------------------+-------------+------------------+-------------------+----------------------+-------------
            4 | Sunisa Rattana    | Sales      | Nipa Sombat     |               6 |           177040.00 |           1
            5 | Thanapon Wisetkul | Sales      | Nipa Sombat     |               4 |            89460.00 |           2
            6 | Achara Sriwattana | Sales      | Nipa Sombat     |               3 |            10770.00 |           3
           10 | Ladda Phromsri    | Sales      | Nipa Sombat     |               3 |            42280.00 |           4
           12 | Waraporn Thepnimit| Sales      | Nipa Sombat     |               2 |            37870.00 |           5
```

> การใช้ `LEFT JOIN employees mgr ON mgr.employee_id = e.manager_id` คือ **self join** แบบคลาสสิก (ทบทวน Part 023) ที่จำเป็นเมื่อตารางมี foreign key อ้างอิงตัวเอง

### 399.6 รวมทุกส่วนเป็น Dashboard เดียวด้วย CTE หลายตัว (Full Executive Report)

```sql
WITH kpi AS (
    SELECT
        COUNT(DISTINCT o.order_id) FILTER (WHERE o.status = 'completed') AS completed_orders,
        SUM(oi.quantity * oi.unit_price) FILTER (WHERE o.status = 'completed') AS gross_revenue
    FROM orders o
    LEFT JOIN order_items oi ON oi.order_id = o.order_id
),
top_product AS (
    SELECT p.product_name, SUM(oi.quantity * oi.unit_price) AS revenue
    FROM order_items oi
    JOIN orders o ON o.order_id = oi.order_id AND o.status = 'completed'
    JOIN products p ON p.product_id = oi.product_id
    GROUP BY p.product_name
    ORDER BY revenue DESC
    LIMIT 1
),
top_customer AS (
    SELECT c.first_name || ' ' || c.last_name AS customer_name,
           SUM(oi.quantity * oi.unit_price) AS spent
    FROM customers c
    JOIN orders o ON o.customer_id = c.customer_id AND o.status = 'completed'
    JOIN order_items oi ON oi.order_id = o.order_id
    GROUP BY c.customer_id, c.first_name, c.last_name
    ORDER BY spent DESC
    LIMIT 1
),
top_employee AS (
    SELECT e.first_name || ' ' || e.last_name AS employee_name,
           SUM(oi.quantity * oi.unit_price) AS revenue
    FROM employees e
    JOIN orders o ON o.employee_id = e.employee_id AND o.status = 'completed'
    JOIN order_items oi ON oi.order_id = o.order_id
    GROUP BY e.employee_id, e.first_name, e.last_name
    ORDER BY revenue DESC
    LIMIT 1
)
SELECT
    k.completed_orders,
    k.gross_revenue,
    tp.product_name  AS best_selling_product,
    tp.revenue        AS best_product_revenue,
    tc.customer_name AS top_customer,
    tc.spent          AS top_customer_spent,
    te.employee_name AS top_salesperson,
    te.revenue        AS top_salesperson_revenue
FROM kpi k, top_product tp, top_customer tc, top_employee te;
```

```
 completed_orders | gross_revenue |  best_selling_product | best_product_revenue |  top_customer | top_customer_spent | top_salesperson | top_salesperson_revenue
-------------------+-----------------+-------------------------+-------------------------+-----------------+-----------------------+-------------------+----------------------------
                26 |       438770.00 | NovaPhone X12           |               77700.00 | Somchai Jaidee  |            66610.00 | Sunisa Rattana   |                177040.00
```

> Dashboard นี้ใช้เทคนิค **CTE หลายตัวคู่ขนาน (multiple independent CTEs)** แล้ว `CROSS JOIN` (ผ่านการ list ตารางคั่นด้วย comma โดยไม่มีเงื่อนไข join เพราะแต่ละ CTE คืนแถวเดียวพอดี) เพื่อรวมทุก KPI เป็นแถวเดียวสำหรับแสดงบนหน้า dashboard สรุปภาพรวม — เป็นรูปแบบที่ใช้บ่อยมากในการทำ single-row summary card

### 399.7 โบนัส: รายงานยอดขายแบบ ROLLUP (สรุปย่อยตามหมวดหมู่ + ยอดรวมสุทธิ)

```sql
SELECT
    COALESCE(cat.category_name, 'รวมทั้งหมด (Grand Total)') AS category,
    COUNT(DISTINCT o.order_id)          AS order_count,
    SUM(oi.quantity)                    AS units_sold,
    SUM(oi.quantity * oi.unit_price)    AS revenue
FROM order_items oi
JOIN orders o     ON o.order_id = oi.order_id AND o.status = 'completed'
JOIN products p    ON p.product_id = oi.product_id
JOIN categories cat ON cat.category_id = p.category_id
GROUP BY ROLLUP (cat.category_name)
ORDER BY revenue DESC NULLS LAST;
```

```
          category         | order_count | units_sold |  revenue
-----------------------------+---------------+--------------+-----------
 รวมทั้งหมด (Grand Total)   |            26 |           35 | 438770.00
 Smartphones                |             7 |           10 |  77700.00
 Laptops                    |             7 |            8 | 226900.00
 Desktops                   |             2 |            2 |  48800.00
 ...                        |           ... |          ... |       ...
```

> `ROLLUP` (ทบทวน Part 028) เพิ่มแถวสรุปยอดรวม (`Grand Total`) ให้อัตโนมัติ โดย `cat.category_name` เป็น `NULL` ในแถวสรุป ซึ่งเราใช้ `COALESCE` แปลงให้อ่านง่ายขึ้น

---

## Step 400: สรุปทั้งหมดระดับกลาง และ Checklist ก่อนขึ้นระดับ Advanced

### 400.1 สรุปหลักสูตรระดับกลาง (Part 021–040)

โปรเจกต์นี้ได้พาเราไล่ทบทวนและประกอบร่างทุกเทคนิคที่เรียนมาตลอดระดับกลาง ตารางด้านล่างสรุปว่าแต่ละ Part ถูกใช้งานจริงอย่างไรในระบบร้านค้าออนไลน์:

| Part | เทคนิค | นำไปใช้ทำอะไรในโปรเจกต์นี้ |
|---|---|---|
| 021 | INNER JOIN | เชื่อม `orders` กับ `customers`, `order_items` กับ `products` เพื่อดึงข้อมูลที่ "ต้องมีคู่กัน" เสมอ |
| 022 | OUTER JOIN | หาสินค้าที่ยังไม่มีรีวิว (LEFT JOIN + IS NULL), รวมออเดอร์ที่ยังไม่มีการชำระเงินในรายงาน |
| 023 | CROSS JOIN / SELF JOIN | หาสายบังคับบัญชาพนักงาน (`employees` join ตัวเอง), รวม KPI หลายตัวเป็นแถวเดียว |
| 024 | Subqueries | คำนวณ `avg_rating_overall` แบบ scalar subquery ใน dashboard |
| 025 | CTE (WITH) | จัดโครงสร้างคิวรีซับซ้อนให้อ่านง่าย แยกเป็นขั้นตอน (เช่น `customer_summary`, `monthly_revenue`) |
| 026 | Recursive CTE | ไล่โครงสร้างสายบังคับบัญชาและ category tree |
| 027 | Aggregate Functions | `SUM`, `COUNT`, `AVG` ทุกรายงาน, `FILTER` clause แยกกลุ่มข้อมูลภายใน aggregate เดียว |
| 028 | GROUP BY / HAVING / ROLLUP / CUBE | สรุปยอดขายรายหมวดหมู่พร้อมยอดรวมสุทธิ |
| 029 | Window Functions พื้นฐาน | `RANK()`, `ROW_NUMBER()`, `SUM() OVER (PARTITION BY ...)` |
| 030 | Window Functions ขั้นสูง | `NTILE`, `LAG`, running total, moving average, frame clause |
| 031–033 | String/Date/Numeric Functions | `DATE_TRUNC`, string concatenation (`||`), `ROUND`, `NULLIF` |
| 034 | CASE WHEN / COALESCE | จัดกลุ่มลูกค้า (VIP/Regular/Occasional), ป้องกันค่า NULL ในรายงาน |
| 035 | Views | `order_summary_view`, `product_performance_view`, `customer_lifetime_value_view` |
| 036 | Materialized Views | `monthly_sales_mv` สำหรับ dashboard ที่ query หนัก |
| 037 | Transactions & ACID | Checkout process แบบ atomic พร้อม `SAVEPOINT` |
| 038 | Isolation Levels | ป้องกัน overselling ด้วย `FOR UPDATE` และ `SERIALIZABLE` |
| 039 | Sequences & Identity Columns | ใช้ `GENERATED ALWAYS AS IDENTITY` ทุกตารางแทน `SERIAL` |

### 400.2 หลักการออกแบบที่เรียนรู้จากโปรเจกต์นี้ (Key Takeaways)

1. **Denormalization ที่จงใจ (intentional)** เช่นการเก็บ `unit_price` ซ้ำใน `order_items` เพื่อรักษาความถูกต้องของข้อมูลเชิงประวัติศาสตร์ (historical accuracy) — ไม่ใช่ทุก denormalization จะเป็นเรื่องผิด
2. **Soft-delete ดีกว่า hard-delete** สำหรับข้อมูลที่มีความสัมพันธ์กับประวัติการขาย (เช่น `products.is_active`)
3. **View แยกตาม single responsibility** ดีกว่า view ใหญ่ตัวเดียวที่ทำทุกอย่าง (ป้องกันปัญหา fan-out)
4. **Materialized view คือ trade-off ระหว่างความเร็วกับความสด** — ต้องเลือก refresh strategy ให้เหมาะกับ SLA ของธุรกิจ
5. **Recursive CTE + closure table pattern** คือวิธีมาตรฐานในการคำนวณยอดสะสมของโครงสร้างต้นไม้ (hierarchical data) โดยไม่นับซ้ำ
6. **Transaction + SAVEPOINT** ทำให้ระบบ checkout ยืดหยุ่น รองรับ fallback logic ได้โดยไม่เสียความเป็น atomic ของทั้งกระบวนการ
7. **Concurrency ต้องคิดล่วงหน้าเสมอ** — default isolation level ไม่ได้ป้องกัน lost update โดยอัตโนมัติ ต้องเลือกใช้ `FOR UPDATE` หรือ `SERIALIZABLE` อย่างมีสติ
8. **IDENTITY column** คือมาตรฐานสมัยใหม่ที่ควรใช้แทน `SERIAL` ในโปรเจกต์ใหม่ทุกโปรเจกต์

### 400.3 Checklist ความพร้อมก่อนขึ้นระดับ Advanced

ก่อนไปต่อยัง Part 041 (ระดับ Advanced) ให้ตรวจสอบว่าคุณสามารถทำสิ่งต่อไปนี้ได้ **โดยไม่ต้องเปิดดูตัวอย่างในบทเรียน**:

- [ ] เขียน `INNER JOIN` และ `LEFT JOIN` เพื่อตอบคำถามธุรกิจง่าย ๆ ได้ภายใน 2 นาที
- [ ] อธิบายความแตกต่างระหว่าง `WHERE` กับ `HAVING` ได้อย่างชัดเจน
- [ ] เขียน recursive CTE เพื่อไล่โครงสร้างต้นไม้ (organization chart, category tree) ได้เอง
- [ ] อธิบายความแตกต่างระหว่าง `ROW_NUMBER()`, `RANK()`, `DENSE_RANK()` พร้อมยกตัวอย่างที่ผลลัพธ์ต่างกัน
- [ ] เขียน window function พร้อม `PARTITION BY` และ frame clause (`ROWS BETWEEN ...`) เพื่อคำนวณ running total หรือ moving average
- [ ] อธิบายว่าทำไม view ธรรมดากับ materialized view ต่างกันอย่างไร และควรเลือกใช้แบบไหนเมื่อไร
- [ ] เขียน transaction ที่มี `SAVEPOINT` และเข้าใจว่าเมื่อไรต้อง `ROLLBACK TO SAVEPOINT` แทนการ `ROLLBACK` ทั้งหมด
- [ ] อธิบาย isolation level ทั้ง 4 ระดับของ PostgreSQL (`READ UNCOMMITTED`, `READ COMMITTED`, `REPEATABLE READ`, `SERIALIZABLE`) และผลกระทบต่อ concurrency ได้
- [ ] อธิบายว่าทำไม `GENERATED ALWAYS AS IDENTITY` ดีกว่า `SERIAL` ในเชิง standard compliance
- [ ] ออกแบบสคีมาฐานข้อมูลใหม่ตั้งแต่ต้น (normalize, กำหนด constraint, กำหนด data type ที่เหมาะสม) ให้กับโจทย์ธุรกิจใหม่ได้ด้วยตนเอง

ถ้าเช็คได้ครบทุกข้อ คุณพร้อมสำหรับระดับ **Advanced** แล้ว ซึ่งจะครอบคลุมหัวข้อที่ลึกขึ้นอีกขั้น:

- **Indexing** (B-tree, Hash, GiST, GIN, BRIN) — ทำไมคิวรีบางตัวช้า และจะเร่งความเร็วอย่างไร
- **PL/pgSQL** — เขียน stored procedure, function, trigger เพื่อย้าย business logic เข้าไปอยู่ในฐานข้อมูล
- **JSON/JSONB** — จัดการข้อมูลกึ่งโครงสร้าง (semi-structured data) ภายใน PostgreSQL
- **Partitioning** — แบ่งตารางขนาดใหญ่มากออกเป็นส่วนย่อยเพื่อประสิทธิภาพและการจัดการที่ดีขึ้น
- และอีกมากมายที่จะพาไปสู่ระดับ **world-class expert**

---

## สรุปท้ายบท

บทนี้เป็นบทปิดท้ายของระดับกลาง (Part 021–040) เราได้สร้างระบบร้านค้าออนไลน์แบบเต็มรูปแบบ 9 ตาราง พร้อมข้อมูลตัวอย่างที่เล่าเรื่องธุรกิจตลอด 9 เดือน จากนั้นนำเทคนิคทั้งหมดที่เรียนมา — ตั้งแต่ JOIN พื้นฐาน ไปจนถึง recursive CTE, window function ขั้นสูง, views, materialized views, transaction และ concurrency control — มาประกอบร่างเป็นระบบ reporting และ business logic ที่ใช้งานได้จริง

ประเด็นสำคัญที่สุดของบทนี้คือ **การเห็นภาพรวม (big picture)** ว่าเทคนิคแต่ละอย่างที่เรียนแยกกันไปทีละ Part นั้น จริง ๆ แล้วต้อง **ทำงานร่วมกัน** เพื่อแก้ปัญหาธุรกิจจริง ไม่มีเทคนิคไหนที่ใช้อย่างโดดเดี่ยว — รายงาน executive dashboard หนึ่งใบอาจต้องใช้ CTE, JOIN, aggregate, window function และ CASE WHEN พร้อมกันทั้งหมด

ขั้นต่อไป เราจะเข้าสู่ระดับ **Advanced** ซึ่งจะเจาะลึกเรื่องประสิทธิภาพ (performance), การเขียนโปรแกรมฝั่งฐานข้อมูล (PL/pgSQL) และการจัดการข้อมูลขนาดใหญ่ — ไปเริ่มกันที่ [Part 041: B-Tree Index](../03-advanced/part-041-btree-index.md)

---

## แบบฝึกหัดขยายโปรเจกต์ (10 ข้อ)

### แบบฝึกหัดที่ 1

เพิ่มคอลัมน์ `discount_percent NUMERIC(5,2) DEFAULT 0 CHECK (discount_percent BETWEEN 0 AND 100)` ลงในตาราง `order_items` จากนั้นเขียนคิวรีคำนวณ `net_price` (ราคาหลังหักส่วนลด) ของแต่ละรายการสินค้า

<details>
<summary>เฉลย</summary>

```sql
ALTER TABLE order_items
    ADD COLUMN discount_percent NUMERIC(5,2) NOT NULL DEFAULT 0
        CHECK (discount_percent BETWEEN 0 AND 100);

-- ตัวอย่าง: ให้ส่วนลด 10% กับสินค้าที่สั่งมากกว่า 1 ชิ้น
UPDATE order_items SET discount_percent = 10 WHERE quantity > 1;

SELECT
    order_item_id,
    quantity,
    unit_price,
    discount_percent,
    ROUND(unit_price * (1 - discount_percent / 100.0), 2) AS net_unit_price,
    ROUND(quantity * unit_price * (1 - discount_percent / 100.0), 2) AS net_line_total
FROM order_items
ORDER BY order_item_id
LIMIT 10;
```

</details>

### แบบฝึกหัดที่ 2

เขียนคิวรีหา "ลูกค้าที่ไม่เคยสั่งซื้อเลยนับตั้งแต่สมัครสมาชิก" โดยใช้ `LEFT JOIN` ร่วมกับ `WHERE ... IS NULL`

<details>
<summary>เฉลย</summary>

```sql
SELECT c.customer_id, c.first_name || ' ' || c.last_name AS customer_name,
       c.signup_date
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.customer_id
WHERE o.order_id IS NULL
ORDER BY c.signup_date;
```

ผลลัพธ์ควรแสดงลูกค้าที่สมัครสมาชิกแล้วแต่ไม่เคยมีคำสั่งซื้อ เช่น `Napat Chaiyaporn` และ `Benjawan Silapat` ซึ่งเพิ่งสมัครเดือนสิงหาคมและกันยายน 2026

</details>

### แบบฝึกหัดที่ 3

สร้าง view ชื่อ `low_stock_alert_view` ที่แสดงสินค้าที่ `is_active = true` และ `stock_quantity` น้อยกว่า 20 ชิ้น เรียงจากน้อยไปมาก

<details>
<summary>เฉลย</summary>

```sql
CREATE OR REPLACE VIEW low_stock_alert_view AS
SELECT product_id, product_name, stock_quantity, unit_price
FROM products
WHERE is_active = true AND stock_quantity < 20
ORDER BY stock_quantity ASC;

SELECT * FROM low_stock_alert_view;
```

</details>

### แบบฝึกหัดที่ 4

เขียน recursive CTE เพื่อแสดง "จำนวนลูกน้องทั้งหมด (รวมทุกระดับ)" ของพนักงานแต่ละคนในตาราง `employees`

<details>
<summary>เฉลย</summary>

```sql
WITH RECURSIVE subordinates AS (
    SELECT employee_id AS manager_id, employee_id AS subordinate_id
    FROM employees
    UNION ALL
    SELECT s.manager_id, e.employee_id
    FROM subordinates s
    JOIN employees e ON e.manager_id = s.subordinate_id
)
SELECT
    m.employee_id,
    m.first_name || ' ' || m.last_name AS employee_name,
    COUNT(*) - 1 AS total_subordinates  -- ลบ 1 เพื่อไม่นับตัวเอง
FROM subordinates s
JOIN employees m ON m.employee_id = s.manager_id
GROUP BY m.employee_id, m.first_name, m.last_name
ORDER BY total_subordinates DESC;
```

</details>

### แบบฝึกหัดที่ 5

ใช้ window function เขียนคิวรีหา "คำสั่งซื้อที่มีมูลค่าสูงสุดของลูกค้าแต่ละคน" (Top-1 order per customer)

<details>
<summary>เฉลย</summary>

```sql
WITH order_totals AS (
    SELECT o.order_id, o.customer_id,
           SUM(oi.quantity * oi.unit_price) AS order_total,
           ROW_NUMBER() OVER (PARTITION BY o.customer_id ORDER BY SUM(oi.quantity * oi.unit_price) DESC) AS rn
    FROM orders o
    JOIN order_items oi ON oi.order_id = o.order_id
    WHERE o.status = 'completed'
    GROUP BY o.order_id, o.customer_id
)
SELECT c.first_name || ' ' || c.last_name AS customer_name, ot.order_id, ot.order_total
FROM order_totals ot
JOIN customers c ON c.customer_id = ot.customer_id
WHERE ot.rn = 1
ORDER BY ot.order_total DESC;
```

</details>

### แบบฝึกหัดที่ 6

สร้าง materialized view ชื่อ `supplier_performance_mv` ที่สรุปยอดขายรวมและจำนวนสินค้าของแต่ละซัพพลายเออร์ พร้อมสร้าง unique index ให้รองรับ `REFRESH CONCURRENTLY`

<details>
<summary>เฉลย</summary>

```sql
CREATE MATERIALIZED VIEW supplier_performance_mv AS
SELECT
    s.supplier_id,
    s.supplier_name,
    s.country,
    COUNT(DISTINCT p.product_id)              AS product_count,
    COALESCE(SUM(oi.quantity), 0)             AS total_units_sold,
    COALESCE(SUM(oi.quantity * oi.unit_price), 0) AS total_revenue
FROM suppliers s
LEFT JOIN products p       ON p.supplier_id = s.supplier_id
LEFT JOIN order_items oi   ON oi.product_id = p.product_id
LEFT JOIN orders o          ON o.order_id = oi.order_id AND o.status = 'completed'
GROUP BY s.supplier_id, s.supplier_name, s.country
WITH DATA;

CREATE UNIQUE INDEX idx_supplier_performance_mv ON supplier_performance_mv (supplier_id);

REFRESH MATERIALIZED VIEW CONCURRENTLY supplier_performance_mv;
```

</details>

### แบบฝึกหัดที่ 7

เขียน transaction ที่ทำการ "ยกเลิกคำสั่งซื้อ" (cancel order) โดยต้อง: เปลี่ยน `status` เป็น `'cancelled'`, คืนสต๊อกสินค้ากลับเข้า `products`, และลบ record การชำระเงินที่เกี่ยวข้องออกจาก `payments` — ทั้งหมดต้องอยู่ใน transaction เดียว

<details>
<summary>เฉลย</summary>

```sql
BEGIN;

DO $$
DECLARE
    v_order_id INTEGER := 24;  -- ตัวอย่าง order ที่ต้องการยกเลิก
    r RECORD;
BEGIN
    -- คืนสต๊อกสำหรับทุกสินค้าในออเดอร์นี้
    FOR r IN SELECT product_id, quantity FROM order_items WHERE order_id = v_order_id LOOP
        UPDATE products SET stock_quantity = stock_quantity + r.quantity
        WHERE product_id = r.product_id;
    END LOOP;

    -- ลบการชำระเงินที่เกี่ยวข้อง
    DELETE FROM payments WHERE order_id = v_order_id;

    -- เปลี่ยนสถานะออเดอร์
    UPDATE orders SET status = 'cancelled' WHERE order_id = v_order_id;

    RAISE NOTICE 'ยกเลิกคำสั่งซื้อ order_id = % สำเร็จ', v_order_id;
END $$;

COMMIT;
```

</details>

### แบบฝึกหัดที่ 8

เขียนคิวรีเปรียบเทียบยอดขายไตรมาส (quarter) ปัจจุบันกับไตรมาสก่อนหน้า โดยใช้ `DATE_TRUNC('quarter', ...)` ร่วมกับ `LAG`

<details>
<summary>เฉลย</summary>

```sql
WITH quarterly_revenue AS (
    SELECT
        DATE_TRUNC('quarter', o.order_date)::DATE AS sales_quarter,
        SUM(oi.quantity * oi.unit_price) AS revenue
    FROM orders o
    JOIN order_items oi ON oi.order_id = o.order_id
    WHERE o.status = 'completed'
    GROUP BY DATE_TRUNC('quarter', o.order_date)
)
SELECT
    sales_quarter,
    revenue,
    LAG(revenue) OVER (ORDER BY sales_quarter) AS prev_quarter_revenue,
    ROUND(
        100.0 * (revenue - LAG(revenue) OVER (ORDER BY sales_quarter))
        / NULLIF(LAG(revenue) OVER (ORDER BY sales_quarter), 0)
    , 2) AS qoq_growth_pct
FROM quarterly_revenue
ORDER BY sales_quarter;
```

</details>

### แบบฝึกหัดที่ 9

ทดสอบด้วยตนเอง (ต้องเปิด 2 session): จำลองสถานการณ์ที่พนักงาน 2 คนพยายามอัปเดต `orders.status` ของออเดอร์เดียวกันพร้อมกัน (คนหนึ่งจะเปลี่ยนเป็น `'shipped'` อีกคนจะเปลี่ยนเป็น `'cancelled'`) โดยใช้ isolation level `READ COMMITTED` แล้วสังเกตพฤติกรรมว่าใครชนะ

<details>
<summary>เฉลย</summary>

```sql
-- ================== Session A ==================
BEGIN;
UPDATE orders SET status = 'shipped' WHERE order_id = 20;
-- ยังไม่ commit

-- ================== Session B (รันพร้อมกัน) ==================
BEGIN;
UPDATE orders SET status = 'cancelled' WHERE order_id = 20;
-- Session B จะ "ค้าง (block)" เพราะ PostgreSQL ล็อกแถวที่กำลังถูก UPDATE โดยอัตโนมัติ
-- (ไม่ต้องใช้ FOR UPDATE ก็ล็อกอัตโนมัติสำหรับ UPDATE statement)

-- ================== Session A ==================
COMMIT;
-- ปลดล็อก

-- ================== Session B (ทำงานต่อทันทีหลัง A commit) ==================
-- UPDATE สำเร็จ เขียนทับค่าของ A -> status สุดท้ายกลายเป็น 'cancelled'
COMMIT;
```

**ข้อสังเกต**: ใน `READ COMMITTED` (ค่า default) การ `UPDATE` แถวเดียวกันพร้อมกันจะไม่ error แต่ transaction ที่ commit ทีหลังจะ "ชนะ" เสมอ (last write wins) ซึ่งอาจไม่ตรงกับ business logic ที่ต้องการ (เช่น ไม่ควรยกเลิกออเดอร์ที่ถูกจัดส่งไปแล้ว) — โจทย์นี้แสดงให้เห็นว่าบางครั้งต้องเพิ่ม application-level check เช่น `WHERE status NOT IN ('shipped')` ในคำสั่ง `UPDATE` เพื่อป้องกัน

</details>

### แบบฝึกหัดที่ 10

เขียนคิวรีเดียวที่รวม: (1) จัดกลุ่มลูกค้าด้วย `NTILE(4)`, (2) หายอดขายรวมของแต่ละกลุ่มด้วย `GROUP BY`, และ (3) แสดงสัดส่วน % ของยอดขายแต่ละกลุ่มต่อยอดขายรวมทั้งหมด

<details>
<summary>เฉลย</summary>

```sql
WITH customer_spend AS (
    SELECT
        c.customer_id,
        COALESCE(SUM(oi.quantity * oi.unit_price), 0) AS total_spent
    FROM customers c
    LEFT JOIN orders o        ON o.customer_id = c.customer_id AND o.status = 'completed'
    LEFT JOIN order_items oi  ON oi.order_id = o.order_id
    GROUP BY c.customer_id
),
segmented AS (
    SELECT *, NTILE(4) OVER (ORDER BY total_spent DESC) AS quartile
    FROM customer_spend
)
SELECT
    quartile,
    COUNT(*)                    AS customer_count,
    SUM(total_spent)            AS segment_revenue,
    ROUND(
        100.0 * SUM(total_spent) / SUM(SUM(total_spent)) OVER ()
    , 2) AS pct_of_total_revenue
FROM segmented
GROUP BY quartile
ORDER BY quartile;
```

```
 quartile | customer_count | segment_revenue | pct_of_total_revenue
----------+------------------+--------------------+-------------------------
        1 |                5 |          207760.00 |                 47.35
        2 |                5 |          140290.00 |                 31.97
        3 |                5 |            77720.00|                 17.72
        4 |                5 |             13000.00|                  2.96
```

สังเกตว่ากลุ่มลูกค้า quartile 1 (25% แรก) สร้างยอดขายเกือบครึ่งหนึ่งของทั้งหมด — นี่คือหลักการ **Pareto Principle (80/20 rule)** ที่พบได้บ่อยในธุรกิจจริง

</details>

---

หมายเหตุปิดท้าย: โปรเจกต์นี้เป็นเพียงจุดเริ่มต้น ในการทำงานจริง ระบบร้านค้าออนไลน์ขนาดใหญ่จะต้องคำนึงถึงประสิทธิภาพของคิวรีเมื่อข้อมูลมีหลายล้านแถว (ต้องใช้ index อย่างถูกต้อง), business logic ที่ซับซ้อนกว่านี้มาก (ต้องใช้ PL/pgSQL function/trigger), ข้อมูลกึ่งโครงสร้าง เช่น product attributes ที่หลากหลาย (ต้องใช้ JSONB), และการจัดการตารางขนาดใหญ่ (ต้องใช้ partitioning) — ทั้งหมดนี้คือสิ่งที่รอเราอยู่ในระดับ **Advanced** เริ่มต้นที่ [Part 041: B-Tree Index](../03-advanced/part-041-btree-index.md)
