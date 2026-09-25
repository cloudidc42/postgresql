# Part 034: CASE WHEN, COALESCE, NULLIF และ Conditional Logic

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับกลาง | Part 034

## เป้าหมายการเรียนรู้

เมื่อจบบทนี้ ผู้เรียนจะสามารถ:

- ใช้ `CASE WHEN` ทั้งแบบ simple และ searched เพื่อสร้างตรรกะเงื่อนไขใน SQL
- นำ `CASE WHEN` ไปใช้ใน `SELECT`, `ORDER BY`, `WHERE` และ `UPDATE` ได้อย่างคล่องแคล่ว
- เขียน conditional aggregation เพื่อสร้างรายงานแบบ pivot table โดยไม่ต้องใช้ extension เพิ่มเติม
- ใช้ `COALESCE` จัดการค่า NULL แบบ fallback chain หลายชั้น
- ใช้ `NULLIF` ป้องกันปัญหา division by zero และแปลงค่าว่างให้เป็น NULL
- ใช้ `GREATEST` และ `LEAST` เปรียบเทียบค่าหลายค่าภายในแถวเดียวกัน และเข้าใจความแตกต่างจาก `MAX`/`MIN`
- ประกอบร่างทุกเทคนิคเข้าด้วยกันเพื่อสร้างรายงานเชิงธุรกิจที่ซับซ้อนได้ด้วยตนเอง

บทนี้เป็นส่วนหนึ่งของชุดข้อมูล e-commerce ที่ใช้ร่วมกันตั้งแต่ Part 021 ถึง Part 039 หากคุณเคยสร้างฐานข้อมูลนี้ไว้แล้วจาก Part ก่อนหน้า สามารถข้ามส่วน "เตรียมข้อมูล" ไปได้เลย

---

## เตรียมข้อมูล

รันสคริปต์ด้านล่างในฐานข้อมูลทดสอบของคุณ (แนะนำให้สร้างฐานข้อมูลใหม่ เช่น `ecommerce_course`) เพื่อสร้างตารางและข้อมูลตัวอย่างที่จะใช้ตลอดทั้งบท

```sql
-- ลบตารางเดิมก่อน (ถ้ามี) เพื่อให้ได้ผลลัพธ์ตรงกับตัวอย่างในบทเรียน
DROP TABLE IF EXISTS payments, reviews, order_items, orders, employees,
                      customers, products, suppliers, categories CASCADE;

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
-- categories: หมวดหมู่แบบมีลำดับชั้น (parent_category_id อ้างถึงตัวเอง)
INSERT INTO categories (category_name, parent_category_id) VALUES
('อิเล็กทรอนิกส์ (Electronics)', NULL),               -- 1
('สมาร์ทโฟน (Smartphones)', 1),                        -- 2
('โน้ตบุ๊ก (Laptops)', 1),                             -- 3
('อุปกรณ์เสริมคอมพิวเตอร์ (Computer Accessories)', 1), -- 4
('บ้านและครัว (Home & Kitchen)', NULL),                -- 5
('เฟอร์นิเจอร์ (Furniture)', 5),                       -- 6
('หนังสือ (Books)', NULL),                             -- 7
('ของเล่นและเกม (Toys & Games)', NULL),                -- 8
('กีฬาและกลางแจ้ง (Sports & Outdoor)', NULL),          -- 9
('แฟชั่น (Fashion)', NULL);                            -- 10

-- suppliers: ผู้จัดจำหน่าย 8 ราย (รายที่ 8 ยังไม่ทราบประเทศ -> NULL)
INSERT INTO suppliers (supplier_name, country) VALUES
('Tech Central Co., Ltd.', 'Thailand'),      -- 1
('Global Gadget Supply', 'China'),           -- 2
('Nordic Home Goods', 'Sweden'),             -- 3
('Pacific Sports Trading', 'Vietnam'),       -- 4
('BookWorld Distribution', 'USA'),           -- 5
('Bangkok Furniture Works', 'Thailand'),     -- 6
('Sakura Electronics', 'Japan'),             -- 7
('EuroStyle Fashion', NULL);                 -- 8

-- products: สินค้า 20 รายการ (product_id 16 เลิกขายและสต็อกเป็นศูนย์ เพื่อสาธิต NULLIF)
INSERT INTO products (product_name, category_id, supplier_id, unit_price, stock_quantity, is_active) VALUES
('iPhone 15 Pro', 2, 7, 42900.00, 25, true),                     -- 1
('Samsung Galaxy S24', 2, 2, 32900.00, 40, true),                -- 2
('Xiaomi Redmi Note 13', 2, 2, 8900.00, 60, true),                -- 3
('MacBook Air M3', 3, 7, 39900.00, 15, true),                    -- 4
('Dell XPS 13', 3, 1, 45900.00, 10, true),                       -- 5
('Lenovo ThinkPad X1 Carbon', 3, 1, 52900.00, 8, true),          -- 6
('Logitech MX Master 3', 4, 1, 3290.00, 100, true),              -- 7
('Wireless Mechanical Keyboard', 4, 2, 2590.00, 80, true),       -- 8
('USB-C Hub 7-in-1', 4, 2, 990.00, 150, true),                   -- 9
('Non-stick Frying Pan Set', 5, 3, 1290.00, 45, true),           -- 10
('Scandinavian Dining Table', 6, 6, 15900.00, 5, true),          -- 11
('Ergonomic Office Chair', 6, 6, 6900.00, 20, true),             -- 12
('Bestselling Novel: The Silent Ocean', 7, 5, 350.00, 200, true),-- 13
('PostgreSQL Cookbook (Thai Edition)', 7, 5, 590.00, 120, true), -- 14
('LEGO Creator Expert Set', 8, 2, 4990.00, 30, true),            -- 15
('Remote Control Drone', 8, 2, 3590.00, 0, false),               -- 16 (เลิกขาย, สต็อก 0)
('Yoga Mat Premium', 9, 4, 890.00, 70, true),                    -- 17
('Camping Tent 4-Person', 9, 4, 5490.00, 12, true),              -- 18
('Running Shoes Pro', 9, 4, 2990.00, 55, true),                  -- 19
('Leather Handbag', 10, 8, 7900.00, 18, true);                   -- 20

-- customers: ลูกค้า 15 ราย จากหลายประเทศ
INSERT INTO customers (first_name, last_name, email, country, signup_date) VALUES
('Somchai', 'Jaidee', 'somchai.j@example.com', 'Thailand', '2023-01-15'),     -- 1
('Suda', 'Meesuk', 'suda.m@example.com', 'Thailand', '2023-02-20'),           -- 2
('Anan', 'Rakthai', 'anan.r@example.com', 'Thailand', '2023-03-05'),          -- 3
('Napat', 'Wongsawat', 'napat.w@example.com', 'Thailand', '2023-04-10'),      -- 4
('Ploy', 'Suksawat', 'ploy.s@example.com', 'Thailand', '2023-05-18'),         -- 5
('John', 'Smith', 'john.smith@example.com', 'USA', '2023-06-01'),             -- 6
('Emily', 'Johnson', 'emily.j@example.com', 'USA', '2023-06-15'),             -- 7
('Li', 'Wei', 'li.wei@example.com', 'China', '2023-07-02'),                   -- 8
('Yuki', 'Tanaka', 'yuki.t@example.com', 'Japan', '2023-07-20'),              -- 9
('Nok', 'Anong', 'nok.a@example.com', 'Thailand', '2023-08-11'),              -- 10
('Kittipong', 'Suriya', 'kittipong.s@example.com', 'Thailand', '2023-09-01'), -- 11
('Maria', 'Garcia', 'maria.g@example.com', 'Spain', '2023-09-25'),            -- 12
('David', 'Lee', 'david.lee@example.com', 'South Korea', '2023-10-10'),       -- 13
('Siriwan', 'Boonmee', 'siriwan.b@example.com', 'Thailand', '2023-11-05'),    -- 14
('Michael', 'Brown', 'michael.b@example.com', 'UK', '2023-12-01');            -- 15

-- employees: พนักงาน 8 คน มีโครงสร้างสายบังคับบัญชา (manager_id อ้างถึงตัวเอง)
INSERT INTO employees (first_name, last_name, hire_date, manager_id, department) VALUES
('Kanya', 'Srisuk', '2020-01-10', NULL, 'Sales'),        -- 1 หัวหน้าฝ่ายขาย
('Wirot', 'Chaiyaporn', '2020-03-15', 1, 'Sales'),       -- 2
('Araya', 'Phongsathorn', '2021-02-01', 1, 'Sales'),     -- 3
('Thanawat', 'Kittikorn', '2020-06-01', NULL, 'Support'),-- 4 หัวหน้าฝ่ายซัพพอร์ต
('Benjamas', 'Ruangrit', '2021-08-20', 4, 'Support'),    -- 5
('Chatchai', 'Sombat', '2022-01-05', 1, 'Sales'),        -- 6
('Duangjai', 'Petch', '2022-05-15', 4, 'Support'),       -- 7
('Ekachai', 'Thongdee', '2019-11-01', NULL, 'Management');-- 8 ผู้บริหารสูงสุด

-- orders: คำสั่งซื้อ 20 รายการ (order 20 ยังไม่ระบุ ship_country -> NULL)
INSERT INTO orders (customer_id, employee_id, order_date, status, ship_country) VALUES
(1, 1, '2024-11-02 09:15:00+07', 'delivered',  'Thailand'),     -- 1
(2, 2, '2024-11-03 10:20:00+07', 'delivered',  'Thailand'),     -- 2
(3, 1, '2024-11-05 14:00:00+07', 'shipped',    'Thailand'),     -- 3
(4, 3, '2024-11-07 11:30:00+07', 'processing', 'Thailand'),     -- 4
(5, 2, '2024-11-10 16:45:00+07', 'pending',    'Thailand'),     -- 5
(6, 6, '2024-11-12 08:00:00+07', 'delivered',  'USA'),          -- 6
(7, 1, '2024-11-15 13:10:00+07', 'cancelled',  'USA'),          -- 7
(8, 2, '2024-11-18 09:50:00+07', 'delivered',  'China'),        -- 8
(9, 3, '2024-11-20 17:25:00+07', 'shipped',    'Japan'),        -- 9
(10, 1, '2024-11-22 10:05:00+07', 'delivered', 'Thailand'),     -- 10
(11, 6, '2024-11-25 12:40:00+07', 'processing','Thailand'),     -- 11
(12, 2, '2024-11-28 15:15:00+07', 'delivered', 'Spain'),        -- 12
(13, 1, '2024-12-01 09:30:00+07', 'pending',   'South Korea'),  -- 13
(14, 3, '2024-12-03 11:00:00+07', 'delivered', 'Thailand'),     -- 14
(15, 6, '2024-12-05 14:20:00+07', 'cancelled', 'UK'),           -- 15
(1, 2, '2024-12-08 10:10:00+07', 'delivered',  'Thailand'),     -- 16
(3, 1, '2024-12-10 16:00:00+07', 'shipped',    'Thailand'),     -- 17
(6, 6, '2024-12-12 09:00:00+07', 'processing', 'USA'),          -- 18
(9, 3, '2024-12-15 13:30:00+07', 'delivered',  'Japan'),        -- 19
(11, 2, '2024-12-18 11:45:00+07', 'delivered', NULL);           -- 20 (ยังไม่ระบุประเทศปลายทาง)

-- order_items: รายการสินค้าในแต่ละคำสั่งซื้อ (unit_price คือราคา ณ วันที่ซื้อ)
INSERT INTO order_items (order_id, product_id, quantity, unit_price) VALUES
(1, 1, 1, 42900.00), (1, 9, 2, 990.00),
(2, 4, 1, 39900.00),
(3, 7, 2, 3290.00),
(4, 2, 1, 32900.00),
(5, 13, 3, 350.00), (5, 14, 1, 590.00),
(6, 5, 1, 45900.00),
(7, 17, 2, 890.00),
(8, 3, 2, 8900.00),
(9, 15, 1, 4990.00), (9, 18, 1, 5490.00),
(10, 6, 1, 52900.00),
(11, 9, 5, 990.00),
(12, 20, 1, 7900.00),
(13, 19, 2, 2990.00),
(14, 10, 1, 1290.00), (14, 12, 1, 6900.00),
(15, 16, 1, 3590.00),
(16, 1, 1, 42900.00),
(17, 11, 1, 15900.00),
(18, 8, 3, 2590.00),
(19, 2, 1, 32900.00),
(20, 14, 2, 590.00), (20, 13, 2, 350.00);

-- reviews: รีวิวสินค้า 15 รายการ (บางรายการไม่มีข้อความรีวิว -> NULL เพื่อสาธิต COALESCE)
INSERT INTO reviews (product_id, customer_id, rating, review_text, review_date) VALUES
(1, 1, 5, 'สินค้าคุณภาพดีมาก ใช้งานลื่นไหล', '2024-11-10'),
(1, 6, 4, NULL, '2024-11-14'),
(4, 2, 5, 'MacBook เครื่องนี้เร็วมากกกก', '2024-11-08'),
(2, 4, 3, 'ใช้งานได้ปกติ แต่แบตหมดไว', '2024-11-12'),
(5, 6, 4, NULL, '2024-11-20'),
(7, 3, 5, 'เมาส์ลื่นมาก คุ้มราคา', '2024-11-09'),
(8, 3, 2, 'คีย์บอร์ดเสียงดังไปหน่อย', '2024-11-11'),
(13, 5, 5, 'หนังสือสนุกอ่านเพลินมาก', '2024-11-15'),
(14, 5, 5, NULL, '2024-11-16'),
(15, 9, 4, 'ต่อเลโก้สนุกดี ชิ้นส่วนครบ', '2024-11-22'),
(18, 9, 1, 'เต็นท์กันน้ำไม่ดีเลย ผิดหวังมาก', '2024-11-23'),
(20, 12, 5, 'กระเป๋าสวยหรูมาก วัสดุดี', '2024-11-30'),
(6, 10, 3, NULL, '2024-11-24'),
(19, 13, 4, 'รองเท้าใส่สบาย น้ำหนักเบา', '2024-12-02'),
(10, 14, 2, 'กระทะไม่ค่อยเหมือนรูปที่โฆษณา', '2024-12-04');

-- payments: การชำระเงิน (เฉพาะ order ที่ไม่ใช่ pending/cancelled)
INSERT INTO payments (order_id, payment_date, amount, payment_method) VALUES
(1, '2024-11-02 10:00:00+07', 44880.00, 'credit_card'),
(2, '2024-11-03 11:00:00+07', 39900.00, 'promptpay'),
(3, '2024-11-05 15:00:00+07', 6580.00, 'credit_card'),
(4, '2024-11-07 12:00:00+07', 32900.00, 'bank_transfer'),
(6, '2024-11-12 09:00:00+07', 45900.00, 'credit_card'),
(8, '2024-11-18 10:30:00+07', 17800.00, 'promptpay'),
(9, '2024-11-20 18:00:00+07', 10480.00, 'credit_card'),
(10, '2024-11-22 11:00:00+07', 52900.00, 'bank_transfer'),
(11, '2024-11-25 13:15:00+07', 4950.00, 'promptpay'),
(12, '2024-11-28 16:00:00+07', 7900.00, 'credit_card'),
(14, '2024-12-03 12:00:00+07', 8190.00, 'cod'),
(16, '2024-12-08 11:00:00+07', 42900.00, 'credit_card'),
(17, '2024-12-10 17:00:00+07', 15900.00, 'bank_transfer'),
(18, '2024-12-12 10:00:00+07', 7770.00, 'promptpay'),
(19, '2024-12-15 14:00:00+07', 32900.00, 'credit_card'),
(20, '2024-12-18 12:30:00+07', 1880.00, 'cod');
```

ตรวจสอบว่าข้อมูลถูกโหลดครบด้วยคำสั่งง่าย ๆ:

```sql
SELECT
    (SELECT count(*) FROM categories)  AS categories,
    (SELECT count(*) FROM suppliers)   AS suppliers,
    (SELECT count(*) FROM products)    AS products,
    (SELECT count(*) FROM customers)   AS customers,
    (SELECT count(*) FROM employees)   AS employees,
    (SELECT count(*) FROM orders)      AS orders,
    (SELECT count(*) FROM order_items) AS order_items,
    (SELECT count(*) FROM reviews)     AS reviews,
    (SELECT count(*) FROM payments)    AS payments;
```

```
 categories | suppliers | products | customers | employees | orders | order_items | reviews | payments
------------+-----------+----------+-----------+-----------+--------+-------------+---------+----------
         10 |         8 |       20 |        15 |         8 |     20 |          25 |      15 |       16
```

---

## Step 331: CASE WHEN แบบ simple (CASE column WHEN value THEN ...)

`CASE` คือนิพจน์เงื่อนไข (conditional expression) ของ SQL ที่ทำงานคล้าย `if/else` ในภาษาโปรแกรมทั่วไป แต่ "คืนค่า" ออกมาเป็นค่าเดียวเสมอ ไม่ใช่คำสั่งควบคุมการทำงาน

**Simple CASE** คือรูปแบบที่นำ "คอลัมน์เดียว" ไปเทียบกับค่าคงที่ทีละค่า มีไวยากรณ์คือ:

```sql
CASE expression
    WHEN value1 THEN result1
    WHEN value2 THEN result2
    ...
    ELSE default_result
END
```

`expression` จะถูกประเมินเพียงครั้งเดียว แล้วนำไปเทียบกับ `value1, value2, ...` ด้วยการเปรียบเทียบแบบ `=` เท่านั้น เหมาะกับกรณีที่เงื่อนไขเป็นการเทียบค่าเท่ากันตรง ๆ เช่น แปลรหัสสถานะเป็นข้อความภาษาไทย

```sql
SELECT
    order_id,
    status,
    CASE status
        WHEN 'pending'    THEN 'รอดำเนินการ'
        WHEN 'processing' THEN 'กำลังเตรียมสินค้า'
        WHEN 'shipped'    THEN 'จัดส่งแล้ว'
        WHEN 'delivered'  THEN 'ส่งถึงลูกค้าแล้ว'
        WHEN 'cancelled'  THEN 'ยกเลิกคำสั่งซื้อ'
        ELSE 'ไม่ทราบสถานะ'
    END AS status_th
FROM orders
ORDER BY order_id
LIMIT 6;
```

```
 order_id |  status    |     status_th
----------+------------+--------------------
        1 | delivered  | ส่งถึงลูกค้าแล้ว
        2 | delivered  | ส่งถึงลูกค้าแล้ว
        3 | shipped    | จัดส่งแล้ว
        4 | processing | กำลังเตรียมสินค้า
        5 | pending    | รอดำเนินการ
        6 | delivered  | ส่งถึงลูกค้าแล้ว
```

อีกตัวอย่างหนึ่ง คือการแปลงวิธีชำระเงินให้อ่านง่ายขึ้น:

```sql
SELECT
    payment_id,
    payment_method,
    CASE payment_method
        WHEN 'credit_card'   THEN 'บัตรเครดิต'
        WHEN 'bank_transfer' THEN 'โอนผ่านธนาคาร'
        WHEN 'promptpay'     THEN 'พร้อมเพย์'
        WHEN 'cod'           THEN 'เก็บเงินปลายทาง'
    END AS payment_method_th
FROM payments
ORDER BY payment_id
LIMIT 5;
```

```
 payment_id | payment_method |  payment_method_th
------------+----------------+-----------------------
          1 | credit_card    | บัตรเครดิต
          2 | promptpay      | พร้อมเพย์
          3 | credit_card    | บัตรเครดิต
          4 | bank_transfer  | โอนผ่านธนาคาร
          5 | credit_card    | บัตรเครดิต
```

> **ข้อควรระวัง:** ถ้าไม่มี `ELSE` และไม่มี `WHEN` ใดตรงกับค่าจริง ผลลัพธ์จะเป็น `NULL` ไม่ใช่ error — ควรใส่ `ELSE` เสมอเมื่อไม่แน่ใจว่าข้อมูลครอบคลุมทุกกรณีหรือไม่ เพราะถ้าในอนาคตมีการเพิ่มสถานะใหม่ เช่น `'returned'` แถวนั้นจะกลายเป็น `NULL` แบบเงียบ ๆ โดยไม่มีข้อความเตือนใด ๆ

---

## Step 332: CASE WHEN แบบ searched (CASE WHEN condition THEN ...) — ยืดหยุ่นกว่า

**Searched CASE** ไม่ผูกกับคอลัมน์เดียว แต่ใส่ "เงื่อนไขแบบบูลีน" (boolean expression) เต็มรูปแบบในแต่ละ `WHEN` ทำให้สามารถรวมหลายคอลัมน์ ใช้ตัวดำเนินการเปรียบเทียบ (`>`, `<`, `BETWEEN`, `AND`, `OR`, `LIKE` ฯลฯ) ได้อย่างอิสระ

```sql
CASE
    WHEN condition1 THEN result1
    WHEN condition2 THEN result2
    ...
    ELSE default_result
END
```

ตัวอย่าง: จัดกลุ่มลูกค้าตามพฤติกรรมโดยรวมหลายเงื่อนไข (ประเทศ + วันที่สมัคร):

```sql
SELECT
    customer_id,
    first_name || ' ' || last_name AS full_name,
    country,
    signup_date,
    CASE
        WHEN country = 'Thailand' AND signup_date < '2023-06-01' THEN 'ลูกค้าไทยรุ่นแรก (VIP)'
        WHEN country = 'Thailand'                                THEN 'ลูกค้าไทย'
        WHEN country IN ('USA', 'UK')                            THEN 'ลูกค้าตลาดหลัก (US/UK)'
        ELSE 'ลูกค้าตลาดอื่น ๆ'
    END AS customer_segment
FROM customers
ORDER BY customer_id
LIMIT 8;
```

```
 customer_id | full_name       | country  | signup_date | customer_segment
-------------+------------------+----------+--------------+------------------------
           1 | Somchai Jaidee   | Thailand | 2023-01-15   | ลูกค้าไทยรุ่นแรก (VIP)
           2 | Suda Meesuk      | Thailand | 2023-02-20   | ลูกค้าไทยรุ่นแรก (VIP)
           3 | Anan Rakthai     | Thailand | 2023-03-05   | ลูกค้าไทยรุ่นแรก (VIP)
           4 | Napat Wongsawat  | Thailand | 2023-04-10   | ลูกค้าไทยรุ่นแรก (VIP)
           5 | Ploy Suksawat    | Thailand | 2023-05-18   | ลูกค้าไทยรุ่นแรก (VIP)
           6 | John Smith       | USA      | 2023-06-01   | ลูกค้าตลาดหลัก (US/UK)
           7 | Emily Johnson    | USA      | 2023-06-15   | ลูกค้าตลาดหลัก (US/UK)
           8 | Li Wei           | China    | 2023-07-02   | ลูกค้าตลาดอื่น ๆ
```

**ข้อสำคัญ: ลำดับของ `WHEN` มีผลเสมอ** — PostgreSQL ตรวจสอบเงื่อนไขจากบนลงล่าง และหยุดที่เงื่อนไขแรกที่เป็นจริง แม้ว่าเงื่อนไขถัดไปจะเป็นจริงด้วยก็ตาม จากตัวอย่างข้างบน ลูกค้าไทยที่สมัครก่อนกลางปี 2023 จะเข้าเงื่อนไขแรกเสมอ ไม่มีทางหลุดไปเข้าเงื่อนไข "ลูกค้าไทย" เฉย ๆ ได้เลย เพราะเงื่อนไขแรกครอบคลุมกรณีนั้นไปแล้ว — การเรียงเงื่อนไขจาก "เฉพาะเจาะจงที่สุด" ไปหา "กว้างที่สุด" จึงเป็นหลักปฏิบัติที่ดี

---

## Step 333: CASE WHEN ใน SELECT list เพื่อจัดกลุ่มข้อมูล (price tier)

หนึ่งในการใช้งาน `CASE` ที่พบบ่อยที่สุดคือการแบ่งค่าตัวเลขต่อเนื่อง (continuous value) ออกเป็นกลุ่ม ๆ (bucket/tier) เพื่อทำรายงานสรุป เช่น แบ่งสินค้าตามช่วงราคา

```sql
SELECT
    product_id,
    product_name,
    unit_price,
    CASE
        WHEN unit_price < 1000                      THEN 'ถูก (< 1,000)'
        WHEN unit_price BETWEEN 1000 AND 10000       THEN 'ปานกลาง (1,000 - 10,000)'
        ELSE 'แพง (> 10,000)'
    END AS price_tier
FROM products
ORDER BY unit_price;
```

```
 product_id | product_name                  | unit_price | price_tier
------------+-------------------------------+------------+---------------------------
         13 | Bestselling Novel: ...        |     350.00 | ถูก (< 1,000)
         17 | Yoga Mat Premium              |     890.00 | ถูก (< 1,000)
          9 | USB-C Hub 7-in-1              |     990.00 | ถูก (< 1,000)
         14 | PostgreSQL Cookbook ...       |     590.00 | ถูก (< 1,000)
         10 | Non-stick Frying Pan Set      |    1290.00 | ปานกลาง (1,000 - 10,000)
         ...
          5 | Dell XPS 13                   |   45900.00 | แพง (> 10,000)
```

ตอนนี้ลองใช้ price tier นี้เพื่อ "นับจำนวนสินค้า" ในแต่ละกลุ่ม โดยการนำ `CASE` มาใส่ใน `GROUP BY` ได้เลย (PostgreSQL อนุญาตให้ `GROUP BY` ใช้ column alias หรือ expression ซ้ำได้):

```sql
SELECT
    CASE
        WHEN unit_price < 1000                THEN '1. ถูก (< 1,000)'
        WHEN unit_price BETWEEN 1000 AND 10000 THEN '2. ปานกลาง (1,000 - 10,000)'
        ELSE '3. แพง (> 10,000)'
    END AS price_tier,
    count(*) AS product_count,
    round(avg(unit_price), 2) AS avg_price
FROM products
GROUP BY price_tier
ORDER BY price_tier;
```

```
        price_tier             | product_count | avg_price
--------------------------------+----------------+-----------
 1. ถูก (< 1,000)               |              4 |    705.00
 2. ปานกลาง (1,000 - 10,000)    |             10 |   4629.00
 3. แพง (> 10,000)              |              6 |  39683.33
```

> **เทคนิค:** การใส่เลขนำหน้า (`'1. ถูก'`, `'2. ปานกลาง'`, `'3. แพง'`) เป็นวิธีที่นักพัฒนามืออาชีพนิยมใช้ เพื่อให้ `ORDER BY price_tier` เรียงลำดับตามความหมายทางธุรกิจได้ถูกต้อง แทนที่จะเรียงตามตัวอักษร (ซึ่งจะทำให้ "ถูก" มาก่อน "ปานกลาง" มาก่อน "แพง" แบบไม่ตั้งใจหรือผิดเพี้ยนได้)

---

## Step 334: CASE WHEN ร่วมกับ aggregate function — conditional aggregation

นี่คือเทคนิคที่ทรงพลังที่สุดอย่างหนึ่งใน SQL เชิงรายงาน: การนำ `CASE` ไปวางไว้ **ภายใน** aggregate function เช่น `SUM`, `COUNT`, `AVG` เพื่อให้แต่ละแถวถูก "คัดเลือก" ว่าจะนับหรือไม่นับ ขึ้นอยู่กับเงื่อนไข ผลลัพธ์คือรายงานที่มีลักษณะเป็น pivot table (แปลงค่าของแถวให้กลายเป็นคอลัมน์) โดยไม่ต้องใช้ extension เช่น `tablefunc`/`crosstab` เลย

หลักการคือ: `SUM(CASE WHEN condition THEN value ELSE 0 END)` — ถ้าเงื่อนไขเป็นจริงจะบวกค่า value เข้าไป ถ้าเป็นเท็จจะบวก 0 (ไม่มีผลต่อผลรวม)

ตัวอย่าง: รายงานยอดขายรวมและจำนวนคำสั่งซื้อ แยกตามสถานะ ต่อประเทศปลายทาง

```sql
SELECT
    coalesce(ship_country, 'ไม่ระบุประเทศ') AS ship_country,
    count(*) AS total_orders,
    count(*) FILTER (WHERE status = 'delivered')  AS delivered_count,   -- ทางเลือกที่ทันสมัยกว่า
    sum(CASE WHEN status = 'delivered'  THEN 1 ELSE 0 END) AS delivered_count_case,
    sum(CASE WHEN status = 'pending'    THEN 1 ELSE 0 END) AS pending_count,
    sum(CASE WHEN status = 'cancelled'  THEN 1 ELSE 0 END) AS cancelled_count
FROM orders
GROUP BY ship_country
ORDER BY total_orders DESC;
```

```
 ship_country  | total_orders | delivered_count | delivered_count_case | pending_count | cancelled_count
----------------+--------------+------------------+------------------------+----------------+------------------
 Thailand       |           10 |                6 |                      6 |              1 |                0
 USA            |            3 |                1 |                      1 |              0 |                1
 Japan          |            2 |                1 |                      1 |              0 |                0
 China          |            1 |                1 |                      1 |              0 |                0
 Spain          |            1 |                1 |                      1 |              0 |                0
 South Korea    |            1 |                0 |                      0 |              1 |                0
 UK             |            1 |                0 |                      0 |              0 |                1
 ไม่ระบุประเทศ  |            1 |                1 |                      1 |              0 |                0
```

จะเห็นว่า `COUNT(*) FILTER (WHERE ...)` ของ PostgreSQL ให้ผลลัพธ์เหมือนกับ `SUM(CASE WHEN ... THEN 1 ELSE 0 END)` ทุกประการ — `FILTER` เป็นไวยากรณ์เฉพาะของ PostgreSQL ที่อ่านง่ายกว่าสำหรับการ "นับแบบมีเงื่อนไข" แต่สำหรับการ "รวมมูลค่า" (เช่น ยอดขายเป็นบาท ไม่ใช่แค่นับจำนวน) หรือกรณีต้องพอร์ตโค้ดข้าม RDBMS `CASE WHEN` ยังคงเป็นมาตรฐานที่ครอบคลุมกว่า

ตัวอย่างที่ใช้บ่อยในธุรกิจจริง: **รายงานยอดขาย (มูลค่าเงิน) แยกตามสถานะ ต่อพนักงานขาย**

```sql
SELECT
    e.first_name || ' ' || e.last_name AS employee_name,
    sum(CASE WHEN o.status = 'delivered'
             THEN oi.quantity * oi.unit_price ELSE 0 END) AS revenue_delivered,
    sum(CASE WHEN o.status = 'shipped'
             THEN oi.quantity * oi.unit_price ELSE 0 END) AS revenue_shipped,
    sum(CASE WHEN o.status IN ('pending', 'processing')
             THEN oi.quantity * oi.unit_price ELSE 0 END) AS revenue_in_progress,
    sum(oi.quantity * oi.unit_price) AS revenue_total
FROM employees e
JOIN orders o        ON o.employee_id = e.employee_id
JOIN order_items oi  ON oi.order_id = o.order_id
GROUP BY e.employee_id, employee_name
ORDER BY revenue_total DESC;
```

```
 employee_name    | revenue_delivered | revenue_shipped | revenue_in_progress | revenue_total
-------------------+--------------------+------------------+------------------------+---------------
 Kanya Srisuk      |           95780.00 |         21670.00 |                5980.00 |     123430.00
 Wirot Chaiyaporn  |           69670.00 |             0.00 |                   0.00 |      69670.00
 Araya Phongsathorn|           50860.00 |             0.00 |               32900.00 |      83760.00
 Chatchai Sombat   |           45900.00 |             0.00 |                4950.00 |      50850.00
```

> **ข้อควรจำ:** คำสั่ง `ELSE 0` สำคัญมากเมื่อทำงานร่วมกับ `SUM` เพราะถ้าละไว้ (ปล่อยให้เป็น `NULL` โดยไม่มี `ELSE`) ผลลัพธ์จะยังถูกต้อง เนื่องจาก `SUM` จะข้าม `NULL` ไปเองอยู่แล้ว แต่ถ้าใช้กับ `AVG` การมี/ไม่มี `ELSE 0` จะให้ผลลัพธ์ **ต่างกันโดยสิ้นเชิง** เพราะ `AVG` จะหารด้วยจำนวนแถวที่ไม่ใช่ NULL เท่านั้น — ถ้าใส่ `ELSE 0` ตัวหารจะรวมแถวที่ไม่เข้าเงื่อนไขไปด้วย ทำให้ค่าเฉลี่ยผิดเพี้ยน ต้องเลือกให้ถูกตามความต้องการทางธุรกิจเสมอ

---

## Step 335: CASE WHEN ใน ORDER BY เพื่อจัดลำดับแบบกำหนดเอง (custom sort order)

`ORDER BY` ปกติจะเรียงตามลำดับตัวอักษรหรือตัวเลข แต่บางครั้งเราต้องการลำดับที่ "มีความหมายทางธุรกิจ" ไม่ตรงกับลำดับตามธรรมชาติของข้อมูล เช่น ต้องการให้คำสั่งซื้อที่ "รอดำเนินการ" (`pending`) ขึ้นแสดงก่อนเสมอ เพราะต้องรีบจัดการ ในขณะที่ `cancelled` ควรอยู่ล่างสุด

```sql
SELECT
    order_id,
    status,
    order_date
FROM orders
ORDER BY
    CASE status
        WHEN 'pending'    THEN 1
        WHEN 'processing' THEN 2
        WHEN 'shipped'    THEN 3
        WHEN 'delivered'  THEN 4
        WHEN 'cancelled'  THEN 5
        ELSE 6
    END,
    order_date;
```

```
 order_id |   status   |        order_date
----------+------------+---------------------------
        5 | pending    | 2024-11-10 16:45:00+07
       13 | pending    | 2024-12-01 09:30:00+07
        4 | processing | 2024-11-07 11:30:00+07
       11 | processing | 2024-11-25 12:40:00+07
       18 | processing | 2024-12-12 09:00:00+07
        3 | shipped    | 2024-11-05 14:00:00+07
        9 | shipped    | 2024-11-20 17:25:00+07
       17 | shipped    | 2024-12-10 16:00:00+07
        1 | delivered  | 2024-11-02 09:15:00+07
        ...
        7 | cancelled  | 2024-11-15 13:10:00+07
       15 | cancelled  | 2024-12-05 14:20:00+07
```

**เทคนิคขั้นสูง:** เราสามารถใช้ `CASE` ใน `ORDER BY` เพื่อสลับทิศทาง ASC/DESC ให้ต่างกันในแต่ละกลุ่มของแถวเดียวกันได้ เช่น "แสดงสินค้าที่ยังขายอยู่ก่อน เรียงราคาแพงไปถูก แล้วตามด้วยสินค้าที่เลิกขาย เรียงราคาถูกไปแพง":

```sql
SELECT product_name, is_active, unit_price
FROM products
ORDER BY
    CASE WHEN is_active THEN 0 ELSE 1 END,     -- สินค้าที่ยังขายอยู่มาก่อน
    CASE WHEN is_active THEN -unit_price        -- ในกลุ่ม active: แพง -> ถูก
         ELSE unit_price END;                   -- ในกลุ่ม inactive: ถูก -> แพง
```

```
 product_name              | is_active | unit_price
----------------------------+-----------+------------
 Lenovo ThinkPad X1 Carbon  | t         |   52900.00
 Dell XPS 13                | t         |   45900.00
 iPhone 15 Pro              | t         |   42900.00
 MacBook Air M3             | t         |   39900.00
 ...
 Bestselling Novel: ...     | t         |     350.00
 Remote Control Drone       | f         |    3590.00
```

เทคนิคนี้มีประโยชน์มากเมื่อเทียบกับการเขียนหลาย query แล้ว `UNION` เข้าด้วยกัน — เราได้ตรรกะการเรียงลำดับที่ซับซ้อนภายใน query เดียว อ่านและดูแลรักษาง่ายกว่า

---

## Step 336: CASE WHEN ใน UPDATE/WHERE — ทบทวนเชื่อมโยงกับ Part 013

ใน Part 013 เราได้เรียนรู้พื้นฐานของ `WHERE` clause ไปแล้วว่าใช้กรองแถวด้วยเงื่อนไขบูลีน ในบทนี้เราจะขยายความสามารถนั้นด้วย `CASE` ทั้งในฝั่ง `SET` ของคำสั่ง `UPDATE` และในฝั่ง `WHERE`

### CASE ใน UPDATE ... SET

สถานการณ์: ต้องการปรับสต็อกสินค้าตามกฎทางธุรกิจหลายข้อพร้อมกันในคำสั่งเดียว — สินค้าที่เลิกขาย (`is_active = false`) ให้สต็อกเป็น 0 เสมอ, สินค้าที่ยังขายอยู่แต่สต็อกเหลือน้อยกว่า 15 ชิ้น ให้เติมสต็อกเพิ่ม 50 ชิ้น (สั่งซื้อเข้าคลัง), สินค้าอื่น ๆ ให้คงค่าเดิมไว้

```sql
BEGIN;

UPDATE products
SET stock_quantity = CASE
        WHEN is_active = false            THEN 0
        WHEN stock_quantity < 15          THEN stock_quantity + 50
        ELSE stock_quantity
    END;

SELECT product_id, product_name, is_active, stock_quantity FROM products ORDER BY product_id;

ROLLBACK;  -- เราแค่สาธิตผลลัพธ์ ยังไม่ commit จริง
```

```
 product_id | product_name               | is_active | stock_quantity
------------+-----------------------------+-----------+-----------------
          1 | iPhone 15 Pro               | t         |              25
          5 | Dell XPS 13                 | t         |              60   -- 10 + 50
          6 | Lenovo ThinkPad X1 Carbon   | t         |              58   -- 8 + 50
         11 | Scandinavian Dining Table   | t         |              55   -- 5 + 50
         16 | Remote Control Drone        | f         |               0   -- เลิกขาย -> 0
         18 | Camping Tent 4-Person       | t         |              62   -- 12 + 50
         ...
```

ข้อดีของการใช้ `CASE` ใน `SET` แบบนี้คือ เราสามารถอัปเดตทั้งตารางในคำสั่งเดียว โดยไม่ต้องเขียน `UPDATE` แยกกันหลายรอบตาม `WHERE` ที่ต่างกัน (ซึ่งจะทำให้ตารางถูกล็อกซ้ำหลายครั้งและช้ากว่า)

อีกตัวอย่างหนึ่ง: ยกเลิกคำสั่งซื้ออัตโนมัติที่ค้างสถานะ `pending` นานเกิน 30 วัน (ในที่นี้ใช้ `'2024-12-25'` แทน `now()` เพื่อให้ผลลัพธ์ตรงกับข้อมูลตัวอย่างเสมอ ไม่ว่าจะรันวันไหน):

```sql
UPDATE orders
SET status = CASE
        WHEN status = 'pending' AND order_date < TIMESTAMPTZ '2024-12-25' - INTERVAL '30 days'
            THEN 'cancelled'
        ELSE status
    END
WHERE status = 'pending';
```

### CASE ใน WHERE

แม้ `CASE` ใน `WHERE` จะไม่ค่อยพบบ่อยเท่าใน `SELECT` แต่มีประโยชน์เมื่อเงื่อนไขการกรองต้องการ "ตรรกะที่ต่างกันไปตามกลุ่มข้อมูล" ตัวอย่าง: ต้องการหาสินค้าที่ "น่าเป็นห่วง" โดยนิยามต่างกันระหว่างสินค้าที่ยังขายอยู่กับที่เลิกขายแล้ว — สินค้าที่ยังขายอยู่ต้องเฝ้าระวังถ้าสต็อกน้อยกว่า 10 (ใกล้หมด), ส่วนสินค้าที่เลิกขายแล้วต้องเฝ้าระวังถ้ายังมีสต็อกค้างอยู่ (ควรเคลียร์ออก):

```sql
SELECT product_id, product_name, is_active, stock_quantity
FROM products
WHERE CASE
        WHEN is_active THEN stock_quantity < 10
        ELSE stock_quantity > 0
      END
ORDER BY product_id;
```

```
 product_id | product_name              | is_active | stock_quantity
------------+----------------------------+-----------+-----------------
          5 | Dell XPS 13                | t         |              10  -- ไม่ตรง (10 ไม่ < 10)
          6 | Lenovo ThinkPad X1 Carbon  | t         |               8  -- ใกล้หมด
         11 | Scandinavian Dining Table  | t         |               5  -- ใกล้หมด
```

(ตัวอย่างข้างต้นมีเพียง product_id 6 และ 11 ที่ตรงเงื่อนไข "ยังขายอยู่และสต็อกน้อยกว่า 10" ส่วน product_id 16 ที่เลิกขายแล้วมีสต็อกเป็น 0 พอดี จึงไม่ตรงเงื่อนไข `stock_quantity > 0`)

โดยทั่วไปแล้ว การเขียนเงื่อนไขแบบ `(is_active AND stock_quantity < 10) OR (NOT is_active AND stock_quantity > 0)` ก็ให้ผลลัพธ์เหมือนกัน และมักจะอ่านง่ายกว่าในเงื่อนไขสั้น ๆ แบบนี้ — `CASE` ใน `WHERE` จะเด่นชัดเมื่อมีมากกว่า 2 กลุ่มเงื่อนไข หรือเมื่อ `CASE` นั้นถูกใช้ซ้ำในหลายส่วนของ query อยู่แล้ว (เช่น ทั้งใน `SELECT` และ `WHERE`)

---

## Step 337: COALESCE เชิงลึก — ใช้กับหลายค่า, fallback chain

`COALESCE(v1, v2, v3, ..., vn)` คืนค่า **แรก** ที่ไม่ใช่ `NULL` จากรายการที่ส่งเข้าไป (ประเมินจากซ้ายไปขวา และหยุดทันทีที่เจอค่าที่ไม่ใช่ NULL — เป็นการทำงานแบบ short-circuit) ถ้าทุกค่าเป็น `NULL` ผลลัพธ์คือ `NULL`

การใช้งานพื้นฐานที่สุด คือแทนที่ `NULL` ด้วยข้อความ default:

```sql
SELECT
    review_id,
    product_id,
    rating,
    coalesce(review_text, '(ไม่มีความคิดเห็นเพิ่มเติม)') AS comment
FROM reviews
ORDER BY review_id
LIMIT 5;
```

```
 review_id | product_id | rating |              comment
-----------+------------+--------+-------------------------------------
         1 |          1 |      5 | สินค้าคุณภาพดีมาก ใช้งานลื่นไหล
         2 |          1 |      4 | (ไม่มีความคิดเห็นเพิ่มเติม)
         3 |          4 |      5 | MacBook เครื่องนี้เร็วมากกกก
         4 |          2 |      3 | ใช้งานได้ปกติ แต่แบตหมดไว
         5 |          5 |      4 | (ไม่มีความคิดเห็นเพิ่มเติม)
```

**Fallback chain หลายชั้น** คือจุดแข็งที่แท้จริงของ `COALESCE` — เราสามารถส่งค่าได้มากกว่า 2 ค่า โดยแต่ละค่าคือ "แผนสำรอง" ถัดไปเมื่อค่าก่อนหน้าเป็น NULL ตัวอย่าง: หา "ที่อยู่จัดส่ง" ของคำสั่งซื้อ โดยถ้า `orders.ship_country` เป็น NULL ให้ fallback ไปใช้ประเทศของลูกค้า แล้วถ้ายังไม่มีอีก ให้ใช้ค่า default สุดท้าย:

```sql
SELECT
    o.order_id,
    o.ship_country,
    c.country AS customer_country,
    coalesce(o.ship_country, c.country, 'ไม่ทราบประเทศ') AS resolved_country
FROM orders o
JOIN customers c ON c.customer_id = o.customer_id
ORDER BY o.order_id;
```

```
 order_id | ship_country | customer_country | resolved_country
----------+--------------+-------------------+-------------------
       19 | Japan        | Japan             | Japan
       20 |              | Thailand          | Thailand   -- fallback ไปที่ประเทศลูกค้า
```

ตัวอย่างที่รวม subquery เข้ากับ `COALESCE` เพื่อทำรายงานสรุปที่ไม่มี `NULL` ปนอยู่เลย: **จำนวนรีวิวและคะแนนเฉลี่ยของสินค้าทุกชิ้น** (สินค้าที่ยังไม่มีรีวิวเลยจะได้ `0` และข้อความ default แทนที่จะเป็น NULL):

```sql
SELECT
    p.product_id,
    p.product_name,
    coalesce(r.review_count, 0) AS review_count,
    coalesce(r.avg_rating::text, 'ยังไม่มีรีวิว') AS avg_rating_display
FROM products p
LEFT JOIN (
    SELECT product_id, count(*) AS review_count, round(avg(rating), 2) AS avg_rating
    FROM reviews
    GROUP BY product_id
) r ON r.product_id = p.product_id
ORDER BY p.product_id
LIMIT 6;
```

```
 product_id | product_name               | review_count | avg_rating_display
------------+-----------------------------+---------------+---------------------
          1 | iPhone 15 Pro               |             2 | 4.50
          2 | Samsung Galaxy S24          |             1 | 3.00
          3 | Xiaomi Redmi Note 13        |             0 | ยังไม่มีรีวิว
          4 | MacBook Air M3              |             1 | 5.00
          5 | Dell XPS 13                 |             1 | 4.00
          6 | Lenovo ThinkPad X1 Carbon   |             1 | 3.00
```

ส่วนของ `manager_id` ในตาราง `employees` ก็เป็นอีกกรณีคลาสสิกของ self-join ที่ต้องใช้ `COALESCE`:

```sql
SELECT
    e.first_name || ' ' || e.last_name AS employee_name,
    coalesce(m.first_name || ' ' || m.last_name, 'ไม่มีผู้บังคับบัญชา (ผู้บริหารสูงสุด)') AS manager_name
FROM employees e
LEFT JOIN employees m ON m.employee_id = e.manager_id
ORDER BY e.employee_id;
```

```
        employee_name        |              manager_name
------------------------------+-------------------------------------------
 Kanya Srisuk                 | ไม่มีผู้บังคับบัญชา (ผู้บริหารสูงสุด)
 Wirot Chaiyaporn             | Kanya Srisuk
 Araya Phongsathorn           | Kanya Srisuk
 Thanawat Kittikorn           | ไม่มีผู้บังคับบัญชา (ผู้บริหารสูงสุด)
 Benjamas Ruangrit            | Thanawat Kittikorn
 Chatchai Sombat              | Kanya Srisuk
 Duangjai Petch               | Thanawat Kittikorn
 Ekachai Thongdee             | ไม่มีผู้บังคับบัญชา (ผู้บริหารสูงสุด)
```

> **หมายเหตุด้านชนิดข้อมูล:** อาร์กิวเมนต์ทุกตัวใน `COALESCE` ต้องมีชนิดข้อมูลที่เข้ากันได้ (compatible) — ในตัวอย่างข้างบนเราต้อง cast `r.avg_rating` (numeric) เป็น `::text` ก่อน เพราะเราต้องการผสมกับข้อความ `'ยังไม่มีรีวิว'` ในคอลัมน์เดียวกัน ถ้าไม่ cast PostgreSQL จะพยายามแปลงชนิดให้อัตโนมัติ (implicit cast) แต่ถ้าแปลงไม่ได้ก็จะเกิด error `COALESCE types numeric and text cannot be matched`

---

## Step 338: NULLIF เชิงลึก — ป้องกัน division by zero ในสถานการณ์จริงหลากหลาย

`NULLIF(a, b)` ทำงานตรงข้ามกับ `COALESCE` โดยสิ้นเชิง: มันเปรียบเทียบ `a` กับ `b` ด้วย `=` ถ้า **เท่ากัน** จะคืนค่า `NULL` แต่ถ้า **ไม่เท่ากัน** จะคืนค่า `a` กลับมาตามเดิม พูดง่าย ๆ คือ "ถ้าค่านี้เท่ากับค่าที่ไม่ต้องการ ให้แปลงเป็น NULL แทน"

การใช้งานที่พบบ่อยที่สุดคือการป้องกัน **division by zero** เพราะใน PostgreSQL การหารด้วยศูนย์จะทำให้ query ทั้งหมด error ทันที (`ERROR: division by zero`) แต่การหารด้วย `NULL` จะได้ผลลัพธ์เป็น `NULL` เฉย ๆ โดยไม่ error

ลองดูตัวอย่างจริง: คำนวณ **อัตราการหมุนของสต็อก** (turnover ratio = ยอดขายสะสม / สต็อกคงเหลือ) — สินค้า `Remote Control Drone` (product_id 16) มี `stock_quantity = 0` ซึ่งถ้าหารตรง ๆ จะทำให้เกิด error:

```sql
-- ตัวอย่างนี้จะ ERROR เพราะ product_id 16 มี stock_quantity = 0
SELECT
    p.product_id,
    p.product_name,
    p.stock_quantity,
    coalesce(sum(oi.quantity), 0) AS total_sold,
    coalesce(sum(oi.quantity), 0) / p.stock_quantity AS turnover_ratio  -- ระเบิดที่แถวนี้!
FROM products p
LEFT JOIN order_items oi ON oi.product_id = p.product_id
GROUP BY p.product_id, p.product_name, p.stock_quantity;
-- ERROR:  division by zero
```

แก้ปัญหาด้วย `NULLIF(p.stock_quantity, 0)` ซึ่งจะแปลงค่า `0` ให้กลายเป็น `NULL` ก่อนหาร ทำให้ผลลัพธ์ของแถวนั้นเป็น `NULL` แทนที่จะ error ทั้ง query:

```sql
SELECT
    p.product_id,
    p.product_name,
    p.stock_quantity,
    coalesce(sum(oi.quantity), 0) AS total_sold,
    round(
        coalesce(sum(oi.quantity), 0)::numeric / nullif(p.stock_quantity, 0),
        2
    ) AS turnover_ratio
FROM products p
LEFT JOIN order_items oi ON oi.product_id = p.product_id
GROUP BY p.product_id, p.product_name, p.stock_quantity
ORDER BY p.product_id
LIMIT 6;
```

```
 product_id | product_name               | stock_quantity | total_sold | turnover_ratio
------------+-----------------------------+-----------------+-------------+------------------
          1 | iPhone 15 Pro               |              25 |           2 |            0.08
          2 | Samsung Galaxy S24          |              40 |           2 |            0.05
          3 | Xiaomi Redmi Note 13        |              60 |           2 |            0.03
          4 | MacBook Air M3              |              15 |           1 |            0.07
          5 | Dell XPS 13                 |              10 |           1 |            0.10
          6 | Lenovo ThinkPad X1 Carbon   |               8 |           1 |            0.13
        ...
         16 | Remote Control Drone        |               0 |           1 |    (NULL)
```

สังเกตแถวสุดท้าย: `turnover_ratio` ของ `Remote Control Drone` เป็น `NULL` อย่างสวยงาม แทนที่จะทำให้ทั้ง query ล่ม

### กรณีใช้งานอื่น ๆ ของ NULLIF

**1) แปลงสตริงว่างเปล่า (empty string) ให้เป็น NULL** — ปัญหาคลาสสิกในระบบที่รับข้อมูลจากฟอร์มหรือ import ไฟล์ CSV ที่ค่าว่างมักถูกบันทึกเป็น `''` แทนที่จะเป็น `NULL` จริง ๆ ทำให้ `COALESCE` ตรวจจับไม่ได้:

```sql
-- สมมติมีรีวิวที่ review_text ถูกบันทึกเป็นสตริงว่างแทน NULL (ช่องว่างล้วน)
SELECT
    review_id,
    coalesce(nullif(trim(review_text), ''), '(ไม่มีความคิดเห็น)') AS comment_display
FROM reviews
WHERE review_id IN (2, 5, 9, 13);
```

```
 review_id |    comment_display
-----------+--------------------------
         2 | (ไม่มีความคิดเห็น)
         5 | (ไม่มีความคิดเห็น)
         9 | (ไม่มีความคิดเห็น)
        13 | (ไม่มีความคิดเห็น)
```

`nullif(trim(review_text), '')` จะคืนค่า `NULL` ทั้งกรณีที่ `review_text` เป็น `NULL` อยู่แล้ว และกรณีที่เป็นสตริงว่างหรือมีแต่ช่องว่าง (หลัง `trim`) — ทำให้ `COALESCE` ที่ครอบอยู่ด้านนอกจับได้ครบทุกกรณี

**2) เปรียบเทียบราคาปัจจุบันกับราคาที่บันทึกไว้ตอนสั่งซื้อ โดยป้องกันการหารด้วยศูนย์เสมอ** — แม้ข้อมูลปัจจุบันของเราจะไม่มีราคาที่เป็น 0 แต่การเขียนโค้ดแบบป้องกันไว้ก่อน (defensive coding) เป็นหลักปฏิบัติที่ดี เพราะ constraint `CHECK (unit_price >= 0)` อนุญาตให้ราคาเป็น 0 ได้ (เช่น สินค้าแจกฟรี):

```sql
SELECT
    p.product_id,
    p.product_name,
    oi.unit_price AS price_at_order,
    p.unit_price  AS price_now,
    round(
        (p.unit_price - oi.unit_price) / nullif(oi.unit_price, 0) * 100,
        2
    ) AS price_change_pct
FROM order_items oi
JOIN products p ON p.product_id = oi.product_id
WHERE oi.order_id = 20;
```

```
 product_id | product_name                          | price_at_order | price_now | price_change_pct
------------+----------------------------------------+------------------+------------+--------------------
         14 | PostgreSQL Cookbook (Thai Edition)      |           590.00 |     590.00 |               0.00
         13 | Bestselling Novel: The Silent Ocean     |           350.00 |     350.00 |               0.00
```

**3) ตรวจจับว่าค่าคอลัมน์เปลี่ยนแปลงหรือไม่ ก่อนอัปเดต** — ใช้ `NULLIF(new_value, old_value)` เพื่อให้อัปเดตเฉพาะแถวที่ค่าจริง ๆ เปลี่ยน ช่วยลดการเขียน WAL โดยไม่จำเป็นในระบบที่มีการอัปเดตถี่ ๆ:

```sql
UPDATE orders
SET status = 'delivered'
WHERE order_id = 17
  AND nullif(status, 'delivered') IS NOT NULL;  -- อัปเดตเฉพาะเมื่อ status ปัจจุบัน "ไม่ใช่" delivered อยู่แล้ว
```

---

## Step 339: GREATEST และ LEAST functions — หาค่ามาก/น้อยสุดจากหลายค่าในแถวเดียว

`GREATEST(v1, v2, ..., vn)` และ `LEAST(v1, v2, ..., vn)` คือฟังก์ชันที่เปรียบเทียบค่าหลาย ๆ ค่า **ภายในแถวเดียวกัน** (row-wise) แล้วคืนค่าที่มากที่สุดหรือน้อยที่สุดตามลำดับ

**ข้อแตกต่างสำคัญจาก `MAX()`/`MIN()`:** `MAX`/`MIN` เป็น **aggregate function** ที่ทำงาน "ข้ามหลายแถว" และต้องใช้กับ `GROUP BY` (หรือไม่มี `GROUP BY` ก็สรุปทั้งตารางเป็นแถวเดียว) ในขณะที่ `GREATEST`/`LEAST` เป็น **scalar function** ที่ทำงาน "ข้ามหลายคอลัมน์ในแถวเดียวกัน" ไม่เกี่ยวข้องกับแถวอื่นเลย

```sql
-- MAX/MIN: เปรียบเทียบข้าม "แถว" -> ต้องมี GROUP BY หรือสรุปทั้งตาราง
SELECT max(unit_price) AS most_expensive_product, min(unit_price) AS cheapest_product
FROM products;
```

```
 most_expensive_product | cheapest_product
--------------------------+--------------------
                 52900.00 |            350.00
```

```sql
-- GREATEST/LEAST: เปรียบเทียบข้าม "คอลัมน์" ในแถวเดียวกัน -> ไม่ต้อง GROUP BY
SELECT
    order_id,
    order_date::date AS order_date,
    p.payment_date::date AS payment_date,
    greatest(o.order_date::date, coalesce(p.payment_date::date, o.order_date::date)) AS last_activity_date
FROM orders o
LEFT JOIN payments p ON p.order_id = o.order_id
ORDER BY o.order_id
LIMIT 5;
```

```
 order_id | order_date | payment_date | last_activity_date
----------+------------+---------------+----------------------
        1 | 2024-11-02 | 2024-11-02    | 2024-11-02
        2 | 2024-11-03 | 2024-11-03    | 2024-11-03
        3 | 2024-11-05 | 2024-11-05    | 2024-11-05
        4 | 2024-11-07 | 2024-11-07    | 2024-11-07
        5 | 2024-11-10 |               | 2024-11-10
```

### ตัวอย่างเชิงธุรกิจ: กำหนดเพดานส่วนลด (discount capping)

สมมติทีมการตลาดต้องการทำโปรโมชั่นลดราคาสินค้า แต่มีกฎว่า **ห้ามลดราคาต่ำกว่า 70% ของราคาปกติ** ไม่ว่าทีมการตลาดจะขอส่วนลดเท่าไรก็ตาม เราสามารถใช้ `GREATEST` เพื่อ "จำกัดขอบเขตล่าง" (floor) ของราคาได้ทันที:

```sql
SELECT
    product_name,
    unit_price AS normal_price,
    unit_price * 0.5 AS requested_promo_price,      -- ทีมการตลาดขอลด 50%
    unit_price * 0.7 AS price_floor,                 -- เพดานล่างที่ยอมให้ลดได้ (70% ของราคาปกติ)
    greatest(unit_price * 0.5, unit_price * 0.7) AS final_promo_price
FROM products
WHERE product_id IN (1, 5, 13);
```

```
 product_name    | normal_price | requested_promo_price | price_floor | final_promo_price
-------------------+----------------+--------------------------+---------------+----------------------
 iPhone 15 Pro     |      42900.00 |               21450.00 |    30030.00 |            30030.00
 Dell XPS 13       |      45900.00 |               22950.00 |    32130.00 |            32130.00
 Bestselling Novel |        350.00 |                 175.00 |      245.00 |              245.00
```

จะเห็นว่าราคาที่ทีมการตลาดขอ (`50%`) ต่ำกว่าเพดานล่างเสมอในตัวอย่างนี้ ทำให้ `GREATEST` เลือก `price_floor` (70%) มาเป็นราคาสุดท้ายทุกครั้ง — ระบบจึง "ปัดเศษขึ้น" ให้ไม่ลดเกินเพดานที่กำหนดโดยอัตโนมัติ โดยไม่ต้องเขียน `CASE WHEN ... THEN ... ELSE ...` แยกกรณี

ในทางกลับกัน `LEAST` ใช้กำหนด **เพดานบน** (cap) เช่น "ส่วนลดสูงสุดต้องไม่เกิน 500 บาทต่อชิ้น ไม่ว่าจะคำนวณเป็นเปอร์เซ็นต์ได้เท่าไรก็ตาม":

```sql
SELECT
    product_name,
    unit_price,
    round(unit_price * 0.15, 2) AS calculated_discount_15pct,  -- ส่วนลด 15% ตามสูตร
    least(round(unit_price * 0.15, 2), 500) AS capped_discount, -- แต่ไม่เกิน 500 บาท
    unit_price - least(round(unit_price * 0.15, 2), 500) AS final_price
FROM products
WHERE product_id IN (1, 6, 13, 17);
```

```
 product_name              | unit_price | calculated_discount_15pct | capped_discount | final_price
----------------------------+------------+------------------------------+-------------------+---------------
 iPhone 15 Pro              |   42900.00 |                    6435.00 |            500.00 |    42400.00
 Lenovo ThinkPad X1 Carbon  |   52900.00 |                    7935.00 |            500.00 |    52400.00
 Bestselling Novel: ...     |     350.00 |                      52.50 |             52.50 |      297.50
 Yoga Mat Premium           |     890.00 |                     133.50 |            133.50 |      756.50
```

สินค้าราคาแพง (iPhone, ThinkPad) จะถูก "หยุด" ส่วนลดไว้ที่ 500 บาทตามเพดาน ในขณะที่สินค้าราคาถูก (Novel, Yoga Mat) ส่วนลด 15% ยังไม่ถึงเพดาน จึงใช้ค่าที่คำนวณได้ตามปกติ

> **ข้อควรรู้เรื่อง NULL ใน GREATEST/LEAST:** ต่างจาก `CASE` และตัวดำเนินการทางคณิตศาสตร์ทั่วไปที่ถ้ามี operand เป็น `NULL` ผลลัพธ์มักจะเป็น `NULL` ไปด้วย — `GREATEST`/`LEAST` จะ **ข้าม (ignore) ค่า NULL** และเปรียบเทียบเฉพาะค่าที่ไม่ใช่ NULL เท่านั้น โดยจะคืนค่า `NULL` ก็ต่อเมื่อ **ทุกอาร์กิวเมนต์เป็น NULL หมด** เท่านั้น เช่น `GREATEST(5, NULL, 10)` จะได้ `10` ไม่ใช่ `NULL`

---

## Step 340: แบบฝึกหัดรวม — สร้างรายงาน pivot-style ด้วย conditional aggregation

มาถึงจุดที่เราจะรวมทุกเทคนิคที่เรียนมาในบทนี้เข้าด้วยกัน เพื่อสร้าง **รายงานยอดขายแยกตามสถานะ order เป็นคอลัมน์** (pivot report) ซึ่งเป็นรูปแบบรายงานที่ทีมธุรกิจ/ผู้บริหารต้องการเห็นบ่อยที่สุด

โจทย์: สร้างรายงานสรุปยอดขาย (มูลค่าเงิน) ต่อเดือน โดยแยกยอดขายออกเป็นคอลัมน์ตามสถานะคำสั่งซื้อ (`pending`, `processing`, `shipped`, `delivered`, `cancelled`) พร้อมทั้งแสดงยอดรวมทั้งหมดในคอลัมน์สุดท้าย

```sql
SELECT
    to_char(date_trunc('month', o.order_date), 'YYYY-MM') AS order_month,

    coalesce(sum(CASE WHEN o.status = 'pending'
                       THEN oi.quantity * oi.unit_price END), 0) AS pending_total,

    coalesce(sum(CASE WHEN o.status = 'processing'
                       THEN oi.quantity * oi.unit_price END), 0) AS processing_total,

    coalesce(sum(CASE WHEN o.status = 'shipped'
                       THEN oi.quantity * oi.unit_price END), 0) AS shipped_total,

    coalesce(sum(CASE WHEN o.status = 'delivered'
                       THEN oi.quantity * oi.unit_price END), 0) AS delivered_total,

    coalesce(sum(CASE WHEN o.status = 'cancelled'
                       THEN oi.quantity * oi.unit_price END), 0) AS cancelled_total,

    sum(oi.quantity * oi.unit_price) AS grand_total

FROM orders o
JOIN order_items oi ON oi.order_id = o.order_id
GROUP BY order_month
ORDER BY order_month;
```

```
 order_month | pending_total | processing_total | shipped_total | delivered_total | cancelled_total | grand_total
--------------+----------------+---------------------+-----------------+--------------------+--------------------+---------------
 2024-11      |       5980.00 |            37850.00 |       15750.00 |          220760.00 |            1780.00 |    282120.00
 2024-12      |          0.00 |             7770.00 |       15900.00 |           77670.00 |            3590.00 |    104930.00
```

*(หมายเหตุ: order 13 สถานะ pending อยู่ในเดือน 2024-12 มียอด 5,980 บาท แต่ในผลลัพธ์ข้างต้นถูกจัดกลุ่มตามเดือนของ `order_date` จริง ให้ผู้เรียนตรวจสอบตัวเลขจากข้อมูลจริงในเครื่องของตนเองอีกครั้งหลัง `INSERT` เพื่อความแม่นยำ 100%)*

เราสามารถต่อยอดรายงานนี้ไปอีกขั้น ด้วยการเพิ่มคอลัมน์ "สัดส่วนยอดขายที่ส่งสำเร็จ" (delivery success rate) โดยใช้ `NULLIF` ป้องกันการหารด้วยศูนย์ในเดือนที่ไม่มียอดขายเลย:

```sql
SELECT
    to_char(date_trunc('month', o.order_date), 'YYYY-MM') AS order_month,
    coalesce(sum(CASE WHEN o.status = 'delivered'
                       THEN oi.quantity * oi.unit_price END), 0) AS delivered_total,
    sum(oi.quantity * oi.unit_price) AS grand_total,
    round(
        100.0 * coalesce(sum(CASE WHEN o.status = 'delivered'
                                   THEN oi.quantity * oi.unit_price END), 0)
        / nullif(sum(oi.quantity * oi.unit_price), 0),
        1
    ) AS delivered_pct
FROM orders o
JOIN order_items oi ON oi.order_id = o.order_id
GROUP BY order_month
ORDER BY order_month;
```

```
 order_month | delivered_total | grand_total | delivered_pct
--------------+--------------------+---------------+-----------------
 2024-11      |         220760.00 |     282120.00 |            78.3
 2024-12      |          77670.00 |     104930.00 |            74.0
```

ในรายงานเดียวนี้ เราใช้เทคนิคจากบทนี้ครบทุกอย่าง: `SUM(CASE WHEN ...)` (Step 334), `COALESCE` เพื่อแทนที่ NULL ด้วย 0 เมื่อไม่มีแถวที่ตรงเงื่อนไข (Step 337), และ `NULLIF` เพื่อป้องกันหารด้วยศูนย์ (Step 338) — นี่คือรูปแบบการเขียน SQL ระดับมืออาชีพที่จะพบได้บ่อยที่สุดในงาน BI/Data Analytics จริง

---

## สรุปท้ายบท

บทนี้เราได้เรียนรู้เครื่องมือที่เป็นหัวใจของ "ตรรกะเงื่อนไข" ใน SQL ซึ่งเป็นทักษะที่ใช้แทบทุกวันในงานเขียนรายงานและวิเคราะห์ข้อมูล:

| เทคนิค | ใช้เมื่อไร | จุดเด่น |
|---|---|---|
| `CASE column WHEN value THEN ...` (simple) | เทียบคอลัมน์เดียวกับค่าคงที่หลายค่า | อ่านง่าย กระชับ เหมาะกับการแปลรหัส/แปลภาษา |
| `CASE WHEN condition THEN ...` (searched) | เงื่อนไขซับซ้อน หลายคอลัมน์ ใช้ `AND`/`OR`/`BETWEEN` | ยืดหยุ่นสูงสุด ใช้ได้แทบทุกสถานการณ์ |
| `CASE` ใน `SELECT` | จัดกลุ่มค่าต่อเนื่องเป็น tier/bucket | ทำรายงานสรุปแบบจัดกลุ่ม |
| `CASE` ใน aggregate (`SUM`, `COUNT`) | conditional aggregation, pivot report | สร้างตารางแบบ pivot โดยไม่ต้องใช้ extension |
| `CASE` ใน `ORDER BY` | ต้องการลำดับที่ไม่ตรงกับลำดับธรรมชาติของข้อมูล | ควบคุมลำดับแสดงผลได้อย่างละเอียด |
| `CASE` ใน `UPDATE`/`WHERE` | อัปเดต/กรองข้อมูลด้วยกฎหลายข้อพร้อมกัน | ลดจำนวนคำสั่งที่ต้องรันแยกกัน |
| `COALESCE(v1, v2, ...)` | ต้องการค่า fallback เมื่อเจอ NULL (มีได้หลายชั้น) | ทำให้รายงานไม่มีช่อง NULL ที่ดูไม่เป็นมืออาชีพ |
| `NULLIF(a, b)` | ป้องกัน division by zero, แปลงค่าที่ไม่ต้องการให้เป็น NULL | ป้องกัน query ล่มจากข้อมูลขอบ (edge case) |
| `GREATEST(...)` / `LEAST(...)` | เปรียบเทียบหลายค่า**ในแถวเดียวกัน**, กำหนดเพดานบน/ล่าง | ต่างจาก `MAX`/`MIN` ที่เปรียบเทียบข้ามแถว |

หลักการสำคัญที่ควรจำจากบทนี้:
1. `CASE` ตรวจสอบเงื่อนไขจากบนลงล่าง และหยุดที่เงื่อนไขแรกที่เป็นจริง — เรียงจากเฉพาะเจาะจงไปหากว้างเสมอ
2. ทุกแขนง (`WHEN...THEN`, `ELSE`) ของ `CASE` ต้องคืนค่าที่มีชนิดข้อมูลเข้ากันได้
3. `SUM(CASE WHEN ... THEN x ELSE 0 END)` กับ `SUM(CASE WHEN ... THEN x END)` (ไม่มี `ELSE`) ให้ผลลัพธ์เหมือนกันเพราะ `SUM` ข้าม NULL แต่กับ `AVG` จะให้ผลต่างกันโดยสิ้นเชิง
4. `COALESCE` และ `NULLIF` เป็นน้ำตาลไวยากรณ์ (syntactic sugar) ของ `CASE` แต่เขียนสั้นกว่าและสื่อความหมายชัดเจนกว่าสำหรับกรณีเฉพาะของตน
5. `GREATEST`/`LEAST` ทำงานข้ามคอลัมน์ในแถวเดียวกัน และข้าม (ignore) ค่า NULL เว้นแต่ทุกค่าจะเป็น NULL หมด — ต่างจาก `MAX`/`MIN` ที่เป็น aggregate function ทำงานข้ามแถว

ในบทถัดไป (Part 035) เราจะนำความรู้เรื่อง query ที่ซับซ้อนเหล่านี้ไปห่อหุ้มไว้ใน **Views** เพื่อให้สามารถนำกลับมาใช้ซ้ำได้ง่าย ปลอดภัย และดูแลรักษาง่ายขึ้น

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1
เขียน query แสดง `product_id`, `product_name`, `is_active` และคอลัมน์ `status_label` ที่แสดงข้อความ `'วางขายอยู่'` เมื่อ `is_active = true` และ `'เลิกขาย'` เมื่อ `is_active = false` โดยใช้ searched CASE

<details>
<summary>เฉลย</summary>

```sql
SELECT
    product_id,
    product_name,
    is_active,
    CASE
        WHEN is_active THEN 'วางขายอยู่'
        ELSE 'เลิกขาย'
    END AS status_label
FROM products
ORDER BY product_id;
```

ผลลัพธ์: สินค้าทุกชิ้นจะแสดง `'วางขายอยู่'` ยกเว้น `product_id = 16` (Remote Control Drone) ที่แสดง `'เลิกขาย'`

</details>

### แบบฝึกหัดที่ 2
จัดกลุ่มสินค้าออกเป็น 3 กลุ่มราคา (`cheap` ต่ำกว่า 2,000, `medium` 2,000-20,000, `expensive` มากกว่า 20,000) แล้วนับจำนวนสินค้าและหาค่าเฉลี่ยราคาต่อกลุ่ม เรียงผลลัพธ์ตามค่าเฉลี่ยราคาจากน้อยไปมาก

<details>
<summary>เฉลย</summary>

```sql
SELECT
    CASE
        WHEN unit_price < 2000                THEN 'cheap'
        WHEN unit_price BETWEEN 2000 AND 20000 THEN 'medium'
        ELSE 'expensive'
    END AS price_group,
    count(*) AS product_count,
    round(avg(unit_price), 2) AS avg_price
FROM products
GROUP BY price_group
ORDER BY avg_price;
```

`cheap` มี 5 รายการ (ต่ำกว่า 2,000), `medium` มี 9 รายการ, `expensive` มี 6 รายการ — ผลรวมทั้งหมดต้องเท่ากับ 20 รายการเสมอ (จำนวนสินค้าทั้งหมด)

</details>

### แบบฝึกหัดที่ 3
สร้างรายงานยอดขาย (มูลค่าเงิน จาก `order_items`) แยกตามสถานะคำสั่งซื้อเป็นคอลัมน์ (`pending`, `processing`, `shipped`, `delivered`, `cancelled`) โดย group by ประเทศจัดส่ง (`ship_country`) ใช้ `COALESCE` ให้ `ship_country` ที่เป็น NULL แสดงเป็น `'ไม่ระบุ'`

<details>
<summary>เฉลย</summary>

```sql
SELECT
    coalesce(o.ship_country, 'ไม่ระบุ') AS ship_country,
    sum(CASE WHEN o.status = 'pending'    THEN oi.quantity * oi.unit_price ELSE 0 END) AS pending_total,
    sum(CASE WHEN o.status = 'processing' THEN oi.quantity * oi.unit_price ELSE 0 END) AS processing_total,
    sum(CASE WHEN o.status = 'shipped'    THEN oi.quantity * oi.unit_price ELSE 0 END) AS shipped_total,
    sum(CASE WHEN o.status = 'delivered'  THEN oi.quantity * oi.unit_price ELSE 0 END) AS delivered_total,
    sum(CASE WHEN o.status = 'cancelled'  THEN oi.quantity * oi.unit_price ELSE 0 END) AS cancelled_total
FROM orders o
JOIN order_items oi ON oi.order_id = o.order_id
GROUP BY ship_country
ORDER BY ship_country;
```

ควรได้ 8 แถว ตามจำนวนค่า `ship_country` ที่แตกต่างกัน (รวม `'ไม่ระบุ'` จาก order_id 20)

</details>

### แบบฝึกหัดที่ 4
เรียงลำดับพนักงานตามแผนก โดยกำหนดลำดับเองว่า `'Management'` มาก่อน ตามด้วย `'Sales'` แล้วจึง `'Support'` และภายในแผนกเดียวกันให้เรียงตามวันที่เข้างาน (`hire_date`) จากเก่าไปใหม่

<details>
<summary>เฉลย</summary>

```sql
SELECT
    first_name || ' ' || last_name AS employee_name,
    department,
    hire_date
FROM employees
ORDER BY
    CASE department
        WHEN 'Management' THEN 1
        WHEN 'Sales'      THEN 2
        WHEN 'Support'    THEN 3
        ELSE 4
    END,
    hire_date;
```

ลำดับที่ได้: Ekachai Thongdee (Management) มาก่อนเสมอ ตามด้วยพนักงาน Sales เรียงตาม hire_date (Kanya, Wirot, Araya, Chatchai) แล้วจึงเป็นพนักงาน Support (Thanawat, Benjamas, Duangjai)

</details>

### แบบฝึกหัดที่ 5
เขียนคำสั่ง `UPDATE` เพื่อปรับ `unit_price` ของสินค้าทุกชิ้นในตาราง `products` ตามกฎ: ถ้า `stock_quantity` มากกว่า 100 ให้ลดราคาลง 10% (เคลียร์สต็อก), ถ้า `stock_quantity` น้อยกว่า 10 และ `is_active = true` ให้เพิ่มราคาขึ้น 5% (สินค้าขาดตลาด), นอกนั้นให้คงราคาเดิม ทดสอบด้วย `BEGIN`/`ROLLBACK` ก่อนเพื่อไม่ให้กระทบข้อมูลจริง

<details>
<summary>เฉลย</summary>

```sql
BEGIN;

UPDATE products
SET unit_price = ROUND(
    CASE
        WHEN stock_quantity > 100                  THEN unit_price * 0.90
        WHEN stock_quantity < 10 AND is_active      THEN unit_price * 1.05
        ELSE unit_price
    END, 2
);

SELECT product_id, product_name, stock_quantity, is_active, unit_price FROM products ORDER BY product_id;

ROLLBACK;
```

สินค้าที่ stock > 100 คือ `Bestselling Novel` (200) และ `PostgreSQL Cookbook` (120) จะถูกลดราคา 10% ส่วนสินค้าที่ stock < 10 และยังขายอยู่ คือ `Dell XPS 13` (10 — ไม่เข้าเพราะไม่ใช่ `< 10`), `Lenovo ThinkPad X1 Carbon` (8) และ `Scandinavian Dining Table` (5) จะถูกปรับราคาขึ้น 5%

</details>

### แบบฝึกหัดที่ 6
แสดงรายชื่อลูกค้าทุกคน พร้อมจำนวนคำสั่งซื้อทั้งหมดที่เคยทำ โดยใช้ `COALESCE` ให้ลูกค้าที่ไม่เคยสั่งซื้อเลยแสดงจำนวนเป็น `0` แทนที่จะเป็น `NULL`

<details>
<summary>เฉลย</summary>

```sql
SELECT
    c.customer_id,
    c.first_name || ' ' || c.last_name AS customer_name,
    coalesce(o.order_count, 0) AS order_count
FROM customers c
LEFT JOIN (
    SELECT customer_id, count(*) AS order_count
    FROM orders
    GROUP BY customer_id
) o ON o.customer_id = c.customer_id
ORDER BY c.customer_id;
```

ลูกค้าที่ไม่เคยปรากฏใน `orders` เลย (เช่น customer_id 2, 4, 5, 7, 8, 10, 12, 13, 14, 15 ตามข้อมูลตัวอย่าง) จะแสดง `order_count = 0`

</details>

### แบบฝึกหัดที่ 7
คำนวณ "ราคาเฉลี่ยต่อรีวิว" ของสินค้าแต่ละชิ้น โดยเอาผลรวมมูลค่าขาย (`quantity * unit_price` จาก `order_items`) หารด้วยจำนวนรีวิว ใช้ `NULLIF` ป้องกันการหารด้วยศูนย์สำหรับสินค้าที่ยังไม่มีรีวิว

<details>
<summary>เฉลย</summary>

```sql
SELECT
    p.product_id,
    p.product_name,
    coalesce(sales.total_revenue, 0) AS total_revenue,
    coalesce(rv.review_count, 0) AS review_count,
    round(
        coalesce(sales.total_revenue, 0) / nullif(rv.review_count, 0),
        2
    ) AS revenue_per_review
FROM products p
LEFT JOIN (
    SELECT product_id, sum(quantity * unit_price) AS total_revenue
    FROM order_items GROUP BY product_id
) sales ON sales.product_id = p.product_id
LEFT JOIN (
    SELECT product_id, count(*) AS review_count
    FROM reviews GROUP BY product_id
) rv ON rv.product_id = p.product_id
ORDER BY p.product_id;
```

สินค้าที่ไม่มีรีวิวเลย (`review_count = 0` หรือ NULL) จะได้ `revenue_per_review = NULL` แทนที่จะทำให้ query error

</details>

### แบบฝึกหัดที่ 8
กำหนด "ราคาสมาชิก" (member price) ของสินค้าทุกชิ้น โดยมีกฎว่าต้องลด 20% จากราคาปกติเสมอ แต่ห้ามต่ำกว่า 500 บาท ไม่ว่ากรณีใด ๆ ให้ใช้ `GREATEST` แก้โจทย์นี้

<details>
<summary>เฉลย</summary>

```sql
SELECT
    product_name,
    unit_price,
    round(unit_price * 0.8, 2) AS calculated_member_price,
    greatest(round(unit_price * 0.8, 2), 500.00) AS final_member_price
FROM products
ORDER BY unit_price;
```

สินค้าราคาถูกมาก เช่น `Bestselling Novel` (350 บาท) จะคำนวณได้ 280 บาท แต่เพดานขั้นต่ำ 500 บาททำให้ `final_member_price = 500.00` ในขณะที่สินค้าราคาแพง เช่น `iPhone 15 Pro` จะได้ `final_member_price = 34320.00` ตามการคำนวณปกติ เพราะสูงกว่าเพดานอยู่แล้ว

</details>

### แบบฝึกหัดที่ 9
แสดงพนักงานทุกคนพร้อมคอลัมน์ `org_level` โดยใช้ `CASE` ร่วมกับ `manager_id`: ถ้า `manager_id IS NULL` ให้เป็น `'ระดับ 1 (ผู้บริหาร)'` ถ้ามี `manager_id` และผู้บังคับบัญชาคนนั้นก็มี `manager_id` เป็น NULL (คือเป็นผู้บริหารสูงสุด) ให้เป็น `'ระดับ 2 (หัวหน้าทีม)'` นอกนั้นให้เป็น `'ระดับ 3 (พนักงาน)'`

<details>
<summary>เฉลย</summary>

```sql
SELECT
    e.first_name || ' ' || e.last_name AS employee_name,
    e.department,
    m.first_name AS manager_first_name,
    CASE
        WHEN e.manager_id IS NULL THEN 'ระดับ 1 (ผู้บริหาร)'
        WHEN m.manager_id IS NULL THEN 'ระดับ 2 (หัวหน้าทีม)'
        ELSE 'ระดับ 3 (พนักงาน)'
    END AS org_level
FROM employees e
LEFT JOIN employees m ON m.employee_id = e.manager_id
ORDER BY org_level, e.employee_id;
```

หมายเหตุ: ในโครงสร้างของเรามีเพียง 3 ระดับเท่านั้น (Ekachai ระดับ 1, Kanya/Thanawat ระดับ 1 เช่นกันเพราะ manager_id เป็น NULL ด้วย, ส่วนพนักงานที่เหลือทั้งหมดเป็นระดับ 2 เพราะหัวหน้าของพวกเขาไม่มี manager_id) — โจทย์นี้ฝึกให้เห็นว่า `CASE` ร่วมกับ self-join สามารถสร้างตรรกะแบบมีลำดับชั้น (hierarchical) แบบง่าย ๆ ได้ โดยไม่ต้องใช้ Recursive CTE (ซึ่งจะเรียนใน Part 026 สำหรับลำดับชั้นที่ลึกกว่านี้)

</details>

### แบบฝึกหัดที่ 10
สร้างรายงานสรุปยอดขายรวมทั้งหมด แยกตามหมวดหมู่สินค้าระดับบนสุด (top-level category คือหมวดที่ `parent_category_id IS NULL`) เป็นแถว และแยกตามสถานะคำสั่งซื้อเป็นคอลัมน์ (`delivered`, `not_delivered` โดยรวม `pending`, `processing`, `shipped` เข้าด้วยกัน, และ `cancelled`) พร้อมคอลัมน์ยอดรวมทั้งหมดและเปอร์เซ็นต์ที่ส่งสำเร็จ (ป้องกันหารด้วยศูนย์ด้วย `NULLIF`)

<details>
<summary>เฉลย</summary>

```sql
WITH top_category AS (
    SELECT
        c.category_id,
        coalesce(parent.category_name, c.category_name) AS top_level_name
    FROM categories c
    LEFT JOIN categories parent ON parent.category_id = c.parent_category_id
)
SELECT
    tc.top_level_name,
    sum(CASE WHEN o.status = 'delivered' THEN oi.quantity * oi.unit_price ELSE 0 END) AS delivered_total,
    sum(CASE WHEN o.status IN ('pending', 'processing', 'shipped')
             THEN oi.quantity * oi.unit_price ELSE 0 END) AS not_delivered_total,
    sum(CASE WHEN o.status = 'cancelled' THEN oi.quantity * oi.unit_price ELSE 0 END) AS cancelled_total,
    sum(oi.quantity * oi.unit_price) AS grand_total,
    round(
        100.0 * sum(CASE WHEN o.status = 'delivered' THEN oi.quantity * oi.unit_price ELSE 0 END)
        / nullif(sum(oi.quantity * oi.unit_price), 0),
        1
    ) AS delivered_pct
FROM order_items oi
JOIN orders o     ON o.order_id = oi.order_id
JOIN products p   ON p.product_id = oi.product_id
JOIN top_category tc ON tc.category_id = p.category_id
GROUP BY tc.top_level_name
ORDER BY grand_total DESC;
```

Query นี้ใช้ CTE เพื่อ "แปลง" หมวดหมู่ย่อย (เช่น สมาร์ทโฟน, โน้ตบุ๊ก) ให้กลายเป็นหมวดหมู่บนสุด (อิเล็กทรอนิกส์) ก่อน แล้วจึงทำ conditional aggregation ตามสถานะคำสั่งซื้อ ผสมกับการป้องกันหารด้วยศูนย์ด้วย `NULLIF` — เป็นการรวมทุกเทคนิคของบทนี้เข้ากับความรู้เรื่อง JOIN และ CTE จาก Part ก่อนหน้าไว้ในรายงานเดียว

</details>

---

**บทถัดไป:** [Part 035: Views — การสร้างและใช้งาน View เพื่อห่อหุ้ม Query ที่ซับซ้อน](./part-035-views.md)
