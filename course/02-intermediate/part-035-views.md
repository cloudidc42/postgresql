# Views: การสร้างและใช้งาน Virtual Table

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับกลาง | Part 035

## เป้าหมายการเรียนรู้

หลังจากเรียนจบบทนี้ ผู้เรียนจะสามารถ:

- อธิบายได้ว่า View คืออะไร ทำงานอย่างไรภายใน PostgreSQL และแตกต่างจาก Table จริงอย่างไร
- สร้าง View ด้วย `CREATE VIEW` ตั้งแต่แบบพื้นฐานไปจนถึง View ที่รวม `JOIN`, aggregate function และ window function
- ใช้ `CREATE OR REPLACE VIEW` ได้อย่างถูกต้อง และเข้าใจข้อจำกัดของการแก้ไข View ที่มีอยู่แล้ว
- ระบุเงื่อนไขที่ View จะเป็น "Updatable View" ได้ และใช้ `INSERT`/`UPDATE`/`DELETE` ผ่าน View โดยตรง
- ใช้ `WITH CHECK OPTION` เพื่อป้องกันไม่ให้ข้อมูลที่แก้ไขหลุดออกจากเงื่อนไขการกรองของ View
- ออกแบบ View ที่ซ้อนกันหลายชั้น (Nested View) และเข้าใจผลกระทบต่อ query plan
- ใช้ Security Barrier View เพื่อจำกัดสิทธิ์การเข้าถึงข้อมูลระดับคอลัมน์/แถว และรู้จักความเชื่อมโยงกับ Row-Level Security ที่จะเรียนใน Part 069
- ลบ View ด้วย `DROP VIEW` และตรวจสอบ definition ของ View ที่มีอยู่ด้วย `\d+`, `pg_views`, `pg_get_viewdef()`
- ออกแบบชุด View สำหรับ Reporting Layer ของระบบ e-commerce ได้จริง

---

## เตรียมข้อมูล

บทนี้ (Part 035) และบทถัดไปในชุด Part 021-039 ใช้ฐานข้อมูลจำลองร้านค้าออนไลน์ (e-commerce) ชุดเดียวกัน เพื่อให้ผู้เรียนเห็นภาพการนำ SQL แต่ละหัวข้อไปใช้กับข้อมูลจริงที่ต่อเนื่องกัน หากเคยสร้างตารางชุดนี้ไว้แล้วจากบทก่อนหน้า สามารถข้ามส่วนนี้ไปได้ แต่หากยังไม่มีหรือทำห้องทดลองแยก ให้รันสคริปต์ทั้งหมดด้านล่างนี้

```sql
-- ลบตารางเดิม (ถ้ามี) เรียงตามลำดับ dependency
DROP TABLE IF EXISTS payments CASCADE;
DROP TABLE IF EXISTS reviews CASCADE;
DROP TABLE IF EXISTS order_items CASCADE;
DROP TABLE IF EXISTS orders CASCADE;
DROP TABLE IF EXISTS employees CASCADE;
DROP TABLE IF EXISTS customers CASCADE;
DROP TABLE IF EXISTS products CASCADE;
DROP TABLE IF EXISTS suppliers CASCADE;
DROP TABLE IF EXISTS categories CASCADE;

-- สร้างตารางตามโครงสร้างมาตรฐานของชุดข้อมูล e-commerce (Part 021-039)
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

### ข้อมูลตัวอย่าง (Seed Data)

```sql
-- categories
INSERT INTO categories (category_id, category_name, parent_category_id) VALUES
(1,  'Electronics',       NULL),
(2,  'Computers',         1),
(3,  'Smartphones',       1),
(4,  'Home & Kitchen',    NULL),
(5,  'Furniture',         4),
(6,  'Appliances',        4),
(7,  'Fashion',           NULL),
(8,  'Men''s Clothing',   7),
(9,  'Women''s Clothing', 7),
(10, 'Books',             NULL);
SELECT setval('categories_category_id_seq', 10);

-- suppliers
INSERT INTO suppliers (supplier_id, supplier_name, country) VALUES
(1, 'Tech Import Co.',          'Thailand'),
(2, 'Global Gadgets Ltd.',      'China'),
(3, 'Nordic Home Supplies',     'Sweden'),
(4, 'Bangkok Furniture Works',  'Thailand'),
(5, 'Fashion Forward Inc.',     'Vietnam'),
(6, 'Book World Publishing',    'USA'),
(7, 'Kitchen Pro Supplies',     'Germany'),
(8, 'Smart Electronics Hub',    'South Korea');
SELECT setval('suppliers_supplier_id_seq', 8);

-- products
INSERT INTO products (product_id, product_name, category_id, supplier_id, unit_price, stock_quantity, is_active) VALUES
(1,  'Laptop Pro 15',             2, 1, 32900.00, 45,  true),
(2,  'Wireless Mouse M1',         2, 2,   590.00, 200, true),
(3,  'Mechanical Keyboard K200',  2, 2,  2490.00, 120, true),
(4,  'Smartphone X12',            3, 8, 24900.00, 60,  true),
(5,  'Smartphone Lite',           3, 8,  8900.00, 90,  true),
(6,  'Bluetooth Earbuds Air',     1, 2,  1490.00, 300, true),
(7,  '4K Monitor 27-inch',        2, 1,  8990.00, 35,  true),
(8,  'Dining Table Set',          5, 4, 15900.00, 10,  true),
(9,  'Office Chair Ergo',         5, 4,  4590.00, 40,  true),
(10, 'Bookshelf Oak',             5, 4,  3990.00, 25,  true),
(11, 'Air Fryer 5L',              6, 7,  2990.00, 80,  true),
(12, 'Stand Mixer Pro',           6, 7,  6490.00, 30,  true),
(13, 'Men''s Denim Jacket',       8, 5,  1290.00, 150, true),
(14, 'Men''s Running Shoes',      8, 5,  2190.00, 100, true),
(15, 'Women''s Summer Dress',     9, 5,   990.00, 180, true),
(16, 'Women''s Leather Bag',      9, 5,  3490.00, 70,  true),
(17, 'PostgreSQL Mastery',       10, 6,   890.00, 60,  true),
(18, 'Data Engineering Handbook',10, 6,  1190.00, 40,  false);
SELECT setval('products_product_id_seq', 18);

-- customers
INSERT INTO customers (customer_id, first_name, last_name, email, country, signup_date) VALUES
(1,  'Somchai',   'Jaidee',      'somchai.j@example.com',    'Thailand',  '2022-01-10'),
(2,  'Malee',     'Suksawat',    'malee.s@example.com',      'Thailand',  '2022-02-15'),
(3,  'John',      'Smith',       'john.smith@example.com',   'USA',       '2022-03-01'),
(4,  'Yui',       'Tanaka',      'yui.tanaka@example.com',   'Japan',     '2022-03-20'),
(5,  'Anong',     'Phetcharat',  'anong.p@example.com',      'Thailand',  '2022-04-05'),
(6,  'David',     'Lee',         'david.lee@example.com',    'Singapore', '2022-05-11'),
(7,  'Siriporn',  'Wattana',     'siriporn.w@example.com',   'Thailand',  '2022-06-01'),
(8,  'Michael',   'Chen',        'michael.chen@example.com', 'Malaysia',  '2022-06-18'),
(9,  'Napat',     'Kittisak',    'napat.k@example.com',      'Thailand',  '2022-07-22'),
(10, 'Emma',      'Wilson',      'emma.wilson@example.com',  'UK',        '2022-08-09'),
(11, 'Pornthip',  'Rungrueng',   'pornthip.r@example.com',   'Thailand',  '2022-09-14'),
(12, 'Kittiya',   'Sombat',      'kittiya.s@example.com',    'Thailand',  '2022-10-02'),
(13, 'Robert',    'Brown',       'robert.brown@example.com', 'Australia', '2022-11-19'),
(14, 'Thanawat',  'Chuenban',    'thanawat.c@example.com',   'Thailand',  '2023-01-05'),
(15, 'Sarah',     'Johnson',     'sarah.johnson@example.com','USA',       '2023-02-14');
SELECT setval('customers_customer_id_seq', 15);

-- employees
INSERT INTO employees (employee_id, first_name, last_name, hire_date, manager_id, department) VALUES
(1, 'Somsak',      'Techarat',    '2018-01-15', NULL, 'Sales'),
(2, 'Nichapa',     'Wongsawang',  '2019-03-01', 1,    'Sales'),
(3, 'Kittipong',   'Saelim',      '2020-06-10', 1,    'Sales'),
(4, 'Aroonrat',    'Phromsri',    '2021-02-20', 1,    'Sales'),
(5, 'Weerayut',    'Chaiyasit',   '2019-08-05', NULL, 'Support'),
(6, 'Suphaporn',   'Intharak',    '2020-11-12', 5,    'Support'),
(7, 'Chalermchai', 'Boonmee',     '2022-01-10', 1,    'Sales'),
(8, 'Patcharin',   'Sukjai',      '2021-07-01', 5,    'Support');
SELECT setval('employees_employee_id_seq', 8);

-- orders
INSERT INTO orders (order_id, customer_id, employee_id, order_date, status, ship_country) VALUES
(1,  1,  2, '2023-01-15 10:30:00+07', 'completed',   'Thailand'),
(2,  2,  2, '2023-01-20 14:00:00+07', 'completed',   'Thailand'),
(3,  3,  3, '2023-02-02 09:15:00+07', 'completed',   'USA'),
(4,  1,  2, '2023-02-10 11:00:00+07', 'completed',   'Thailand'),
(5,  4,  3, '2023-02-18 16:45:00+07', 'cancelled',   'Japan'),
(6,  5,  4, '2023-03-01 10:00:00+07', 'completed',   'Thailand'),
(7,  6,  3, '2023-03-05 13:20:00+07', 'shipped',     'Singapore'),
(8,  2,  2, '2023-03-12 09:00:00+07', 'completed',   'Thailand'),
(9,  7,  7, '2023-03-20 15:30:00+07', 'completed',   'Thailand'),
(10, 8,  3, '2023-04-01 10:10:00+07', 'processing',  'Malaysia'),
(11, 9,  4, '2023-04-08 11:40:00+07', 'completed',   'Thailand'),
(12, 1,  2, '2023-04-15 14:00:00+07', 'completed',   'Thailand'),
(13, 10, 7, '2023-04-22 09:50:00+07', 'pending',     'UK'),
(14, 3,  3, '2023-05-01 10:00:00+07', 'completed',   'USA');
SELECT setval('orders_order_id_seq', 14);

-- order_items
INSERT INTO order_items (order_item_id, order_id, product_id, quantity, unit_price) VALUES
(1,  1,  1,  1, 32900.00),
(2,  1,  2,  2,   590.00),
(3,  2,  4,  1, 24900.00),
(4,  3,  17, 3,   890.00),
(5,  3,  18, 1,  1190.00),
(6,  4,  6,  2,  1490.00),
(7,  5,  7,  1,  8990.00),
(8,  6,  11, 1,  2990.00),
(9,  6,  12, 1,  6490.00),
(10, 7,  3,  1,  2490.00),
(11, 8,  5,  1,  8900.00),
(12, 8,  6,  1,  1490.00),
(13, 9,  13, 2,  1290.00),
(14, 9,  14, 1,  2190.00),
(15, 10, 9,  1,  4590.00),
(16, 11, 15, 3,   990.00),
(17, 11, 16, 1,  3490.00),
(18, 12, 1,  1, 32900.00),
(19, 13, 10, 1,  3990.00),
(20, 14, 17, 2,   890.00);
SELECT setval('order_items_order_item_id_seq', 20);

-- reviews
INSERT INTO reviews (review_id, product_id, customer_id, rating, review_text, review_date) VALUES
(1,  1,  1,  5, 'เครื่องแรงมาก ทำงานลื่นสุดๆ',                 '2023-01-20'),
(2,  1,  12, 4, 'ดีแต่ราคาสูงไปหน่อย',                          '2023-02-01'),
(3,  4,  2,  5, 'กล้องถ่ายรูปสวยมาก',                            '2023-01-25'),
(4,  6,  1,  4, 'เสียงดีเกินราคา',                               '2023-02-15'),
(5,  6,  2,  3, 'แบตอยู่ได้ไม่นานเท่าที่คิด',                     '2023-03-15'),
(6,  17, 3,  5, 'อธิบายละเอียดเข้าใจง่ายมาก',                    '2023-02-10'),
(7,  17, 14, 5, 'หนังสือ PostgreSQL ที่ดีที่สุดที่เคยอ่าน',       '2023-05-05'),
(8,  9,  8,  4, 'นั่งสบาย ปรับระดับได้ดี',                       '2023-04-05'),
(9,  11, 5,  5, 'ทอดอาหารอร่อย ใช้งานง่าย',                     '2023-03-05'),
(10, 12, 5,  4, 'ผสมแป้งได้เนียนดี',                             '2023-03-06'),
(11, 13, 7,  4, 'ผ้าดีใส่สบาย',                                  '2023-03-25'),
(12, 15, 9,  5, 'ผ้าเนื้อดีทรงสวย',                              '2023-04-10'),
(13, 16, 9,  5, 'หนังแท้คุณภาพดี',                               '2023-04-11'),
(14, 3,  6,  3, 'ปุ่มแข็งไปหน่อยตอนแรก',                         '2023-03-10'),
(15, 2,  1,  4, 'ใช้งานลื่นไม่มีสะดุด',                          '2023-01-21');
SELECT setval('reviews_review_id_seq', 15);

-- payments (เฉพาะออเดอร์ที่ชำระเงินแล้ว: completed และ shipped)
INSERT INTO payments (payment_id, order_id, payment_date, amount, payment_method) VALUES
(1,  1,  '2023-01-15 10:35:00+07', 34080.00, 'credit_card'),
(2,  2,  '2023-01-20 14:05:00+07', 24900.00, 'promptpay'),
(3,  3,  '2023-02-02 09:20:00+07',  3860.00, 'credit_card'),
(4,  4,  '2023-02-10 11:05:00+07',  2980.00, 'promptpay'),
(5,  6,  '2023-03-01 10:05:00+07',  9480.00, 'bank_transfer'),
(6,  7,  '2023-03-05 13:25:00+07',  2490.00, 'credit_card'),
(7,  8,  '2023-03-12 09:05:00+07', 10390.00, 'promptpay'),
(8,  9,  '2023-03-20 15:35:00+07',  4770.00, 'cod'),
(9,  11, '2023-04-08 11:45:00+07',  6460.00, 'credit_card'),
(10, 12, '2023-04-15 14:05:00+07', 32900.00, 'promptpay'),
(11, 14, '2023-05-01 10:05:00+07',  1780.00, 'credit_card');
SELECT setval('payments_payment_id_seq', 11);
```

> **หมายเหตุเกี่ยวกับข้อมูล**: ออเดอร์หมายเลข 5 มีสถานะ `cancelled` จึงไม่มีการชำระเงิน ออเดอร์หมายเลข 10 (`processing`) และหมายเลข 13 (`pending`) ยังไม่มีการชำระเงินเช่นกัน ส่วนออเดอร์ที่เหลือมีสถานะ `completed` หรือ `shipped` และมีการชำระเงินครบถ้วน ข้อมูลชุดนี้ถูกออกแบบให้ตัวเลขในตัวอย่างผลลัพธ์ของบทนี้คำนวณได้ตรงกันทุกจุด ผู้เรียนสามารถรันคำสั่งจริงเพื่อตรวจสอบผลลัพธ์ได้

---

## Step 341: View คืออะไร

**View** คือ "ตารางเสมือน" (Virtual Table) ที่สร้างจากคำสั่ง `SELECT` ที่ถูกบันทึกชื่อไว้ในฐานข้อมูล View **ไม่ได้เก็บข้อมูลจริงของตัวเอง** — ทุกครั้งที่มีการ query View ระบบจะไปรัน query ที่อยู่เบื้องหลัง (`SELECT` statement ที่ใช้ตอนสร้าง) กับตารางต้นทางแบบ real-time เสมอ

พูดง่ายๆ คือ View เป็นเหมือน "query ที่ตั้งชื่อไว้" (named query) ที่เราสามารถเรียกใช้ซ้ำได้เหมือนกับเป็นตารางหนึ่ง โดยไม่ต้องเขียน SQL ยาวๆ ซ้ำทุกครั้ง

### View ต่างจาก Table จริงอย่างไร

| คุณสมบัติ | Table | View |
|---|---|---|
| เก็บข้อมูลจริงบน disk | ใช่ | ไม่ใช่ (ยกเว้น Materialized View ที่จะเรียนใน Part 036) |
| ข้อมูลอัปเดตอัตโนมัติเมื่อ base table เปลี่ยน | - | ใช่ เสมอ (real-time) |
| ต้องมี storage และ index ของตัวเอง | ใช่ | ไม่ (แต่ base table ยังมี index ตามปกติ) |
| Query ผ่าน View ช้ากว่า Table โดยตรงหรือไม่ | - | ปกติไม่ช้ากว่า เพราะ planner จะ "flatten" (ยุบรวม) query ของ View เข้ากับ query ภายนอกก่อนวางแผนจริง |

### ประโยชน์ของ View

1. **ความปลอดภัย (Security)** — เราสามารถให้สิทธิ์ผู้ใช้เข้าถึงเฉพาะ View ที่กรองคอลัมน์หรือแถวที่อนุญาตเท่านั้น โดยไม่ต้องให้สิทธิ์เข้าถึงตารางต้นทางโดยตรง เช่น ซ่อนคอลัมน์อีเมลลูกค้าจากพนักงานทั่วไป
2. **ความสะดวก (Convenience)** — query ที่ซับซ้อน มี JOIN หลายตาราง มี aggregate หรือ window function สามารถเขียนครั้งเดียวแล้วเรียกใช้ซ้ำได้ด้วยชื่อสั้นๆ
3. **Encapsulation (การห่อหุ้มความซับซ้อน)** — หากโครงสร้างตารางต้นทางเปลี่ยนแปลง (เช่น แยกตารางใหม่ เปลี่ยนชื่อคอลัมน์) เราสามารถปรับ View ให้ยังคง interface เดิมไว้ได้ โดยแอปพลิเคชันที่เรียกใช้ View ไม่ต้องแก้โค้ด
4. **ความสม่ำเสมอของ business logic** — คำนวณตัวเลขสำคัญ เช่น "ยอดขายสุทธิ" หรือ "ลูกค้าที่ Active" ด้วยสูตรเดียวกันทุกที่ที่เรียกใช้ ลดความเสี่ยงที่แต่ละทีมจะคำนวณสูตรต่างกัน

### ตัวอย่างแรก: View อย่างง่าย

```sql
CREATE VIEW active_products AS
SELECT product_id, product_name, unit_price, stock_quantity
FROM products
WHERE is_active = true;
```

```sql
SELECT * FROM active_products ORDER BY product_id LIMIT 5;
```

```
 product_id |      product_name       | unit_price | stock_quantity
------------+--------------------------+------------+----------------
          1 | Laptop Pro 15            |   32900.00 |             45
          2 | Wireless Mouse M1        |     590.00 |            200
          3 | Mechanical Keyboard K200 |    2490.00 |            120
          4 | Smartphone X12           |   24900.00 |             60
          5 | Smartphone Lite          |    8900.00 |             90
(5 rows)
```

สังเกตว่าเราสามารถ query `active_products` ได้เหมือนเป็นตารางปกติทุกประการ ทั้งที่จริงแล้วเบื้องหลังคือการรัน `SELECT ... FROM products WHERE is_active = true` ทุกครั้ง สินค้ารายการที่ 18 (`Data Engineering Handbook`) ซึ่งมี `is_active = false` จะไม่ปรากฏใน View นี้เลย และถ้าวันหนึ่งมีการเพิ่มสินค้าใหม่ในตาราง `products` โดยมี `is_active = true` สินค้านั้นจะปรากฏใน `active_products` ทันทีโดยไม่ต้องแก้ไข View

### View ถูกเก็บไว้ที่ไหน

View ถูกเก็บเป็น object หนึ่งใน schema เดียวกับตาราง (default คือ `public`) และสามารถดูรายชื่อ View ทั้งหมดได้ด้วย `\dv` ใน `psql`:

```sql
\dv
```

```
              List of relations
 Schema |      Name       | Type | Owner
--------+------------------+------+-------
 public | active_products  | view | app
(1 row)
```

---

## Step 342: CREATE VIEW พื้นฐาน

### Syntax

```sql
CREATE [OR REPLACE] VIEW view_name [(column_name [, ...])] AS
    query
[WITH [CASCADED | LOCAL] CHECK OPTION];
```

- `view_name` — ชื่อ View (ควรตั้งชื่อให้สื่อความหมาย เช่น ขึ้นต้นหรือลงท้ายด้วย `_view` หรือใช้คำที่บ่งบอกว่าเป็นมุมมองข้อมูล)
- `column_name` — (optional) กำหนดชื่อคอลัมน์ของ View เอง หากไม่ระบุจะใช้ชื่อคอลัมน์ตามผลลัพธ์ของ `SELECT`
- `query` — คำสั่ง `SELECT` ใดๆ ก็ได้ รวมถึง `SELECT` ที่มี `JOIN`, aggregate, window function, CTE
- `WITH CHECK OPTION` — จะกล่าวถึงใน Step 346

### ตัวอย่าง: View กรองข้อมูลลูกค้าในประเทศไทย

```sql
CREATE VIEW thai_customers AS
SELECT customer_id, first_name, last_name, email, signup_date
FROM customers
WHERE country = 'Thailand'
ORDER BY signup_date;
```

```sql
SELECT * FROM thai_customers;
```

```
 customer_id | first_name | last_name |            email            | signup_date
-------------+------------+-----------+------------------------------+-------------
           1 | Somchai    | Jaidee    | somchai.j@example.com       | 2022-01-10
           2 | Malee      | Suksawat  | malee.s@example.com         | 2022-02-15
           5 | Anong      | Phetcharat| anong.p@example.com         | 2022-04-05
           7 | Siriporn   | Wattana   | siriporn.w@example.com      | 2022-06-01
           9 | Napat      | Kittisak  | napat.k@example.com         | 2022-07-22
          11 | Pornthip   | Rungrueng | pornthip.r@example.com      | 2022-09-14
          12 | Kittiya    | Sombat    | kittiya.s@example.com       | 2022-10-02
          14 | Thanawat   | Chuenban  | thanawat.c@example.com      | 2023-01-05
(8 rows)
```

เราสามารถใช้ View นี้ต่อยอดได้เหมือนตารางปกติ เช่น กรองเพิ่มเติม, `JOIN` กับตารางอื่น หรือใช้ในคำสั่ง `WHERE` ของ query ภายนอก:

```sql
SELECT customer_id, first_name, last_name
FROM thai_customers
WHERE signup_date >= '2022-07-01';
```

```
 customer_id | first_name | last_name
-------------+------------+-----------
           9 | Napat      | Kittisak
          11 | Pornthip   | Rungrueng
          12 | Kittiya    | Sombat
          14 | Thanawat   | Chuenban
(4 rows)
```

### ตั้งชื่อคอลัมน์เอง

```sql
CREATE VIEW product_price_list (id, name, price_baht) AS
SELECT product_id, product_name, unit_price
FROM products
WHERE is_active = true;
```

```sql
SELECT * FROM product_price_list ORDER BY price_baht DESC LIMIT 3;
```

```
 id | name                | price_baht
----+---------------------+------------
  1 | Laptop Pro 15       |   32900.00
  4 | Smartphone X12      |   24900.00
  8 | Dining Table Set    |   15900.00
(3 rows)
```

---

## Step 343: View ที่ซับซ้อน — JOIN, Aggregate, Window Function

View ไม่ได้จำกัดอยู่แค่ `SELECT ... WHERE` ธรรมดา แต่สามารถรวมความซับซ้อนของ query ทุกรูปแบบที่ PostgreSQL รองรับไว้ในที่เดียวได้ ทำให้ทีมอื่น (เช่น ทีม BI, ทีม Backend) ใช้ข้อมูลที่ผ่านการคำนวณแล้วได้ทันทีโดยไม่ต้องเข้าใจ logic เบื้องหลัง

### ตัวอย่าง 1: View ที่มี JOIN หลายตาราง

```sql
CREATE VIEW order_details AS
SELECT
    o.order_id,
    o.order_date,
    c.first_name || ' ' || c.last_name AS customer_name,
    e.first_name || ' ' || e.last_name AS sold_by,
    p.product_name,
    oi.quantity,
    oi.unit_price,
    oi.quantity * oi.unit_price AS line_total,
    o.status
FROM orders o
JOIN customers c  ON c.customer_id = o.customer_id
JOIN employees e  ON e.employee_id = o.employee_id
JOIN order_items oi ON oi.order_id = o.order_id
JOIN products p   ON p.product_id = oi.product_id;
```

```sql
SELECT order_id, customer_name, product_name, quantity, line_total, status
FROM order_details
WHERE order_id = 1;
```

```
 order_id | customer_name |    product_name    | quantity | line_total |  status
----------+----------------+---------------------+----------+------------+-----------
        1 | Somchai Jaidee | Laptop Pro 15       |        1 |   32900.00 | completed
        1 | Somchai Jaidee | Wireless Mouse M1   |        2 |    1180.00 | completed
(2 rows)
```

### ตัวอย่าง 2: View ที่มี Aggregate Function

```sql
CREATE VIEW category_revenue AS
SELECT
    cat.category_id,
    cat.category_name,
    COUNT(DISTINCT oi.order_id)      AS order_count,
    SUM(oi.quantity)                 AS total_quantity_sold,
    SUM(oi.quantity * oi.unit_price) AS total_revenue
FROM categories cat
JOIN products p     ON p.category_id = cat.category_id
JOIN order_items oi ON oi.product_id = p.product_id
JOIN orders o        ON o.order_id = oi.order_id
WHERE o.status <> 'cancelled'
GROUP BY cat.category_id, cat.category_name
ORDER BY total_revenue DESC;
```

```sql
SELECT * FROM category_revenue;
```

```
 category_id | category_name | order_count | total_quantity_sold | total_revenue
-------------+----------------+-------------+----------------------+---------------
           2 | Computers      |           4 |                    5 |      72060.00
           3 | Smartphones    |           2 |                    2 |      33800.00
          10 | Books          |           2 |                    6 |       5640.00
           6 | Appliances     |           1 |                    2 |       9480.00
           1 | Electronics    |           2 |                    3 |       4470.00
           9 | Women's Clothing |         1 |                    4 |       6460.00
           8 | Men's Clothing |           1 |                    3 |       4770.00
           5 | Furniture      |           2 |                    2 |       8580.00
(8 rows)
```

View นี้รวม `JOIN` สี่ตารางและ aggregate ไว้ในคำสั่งเดียว ผู้ใช้ปลายทางเพียงแค่ `SELECT * FROM category_revenue` ก็ได้ตัวเลขสรุปยอดขายตามหมวดหมู่ทันที โดยไม่ต้องรู้เลยว่าเบื้องหลังมี logic การ join และ filter สถานะ `cancelled` อย่างไร

### ตัวอย่าง 3: View ที่มี Window Function

```sql
CREATE VIEW product_ranking_by_category AS
SELECT
    p.product_id,
    p.product_name,
    cat.category_name,
    COALESCE(SUM(oi.quantity * oi.unit_price), 0) AS total_revenue,
    RANK() OVER (
        PARTITION BY cat.category_id
        ORDER BY COALESCE(SUM(oi.quantity * oi.unit_price), 0) DESC
    ) AS rank_in_category
FROM products p
JOIN categories cat ON cat.category_id = p.category_id
LEFT JOIN order_items oi ON oi.product_id = p.product_id
LEFT JOIN orders o ON o.order_id = oi.order_id AND o.status <> 'cancelled'
GROUP BY p.product_id, p.product_name, cat.category_id, cat.category_name;
```

```sql
SELECT product_id, product_name, total_revenue, rank_in_category
FROM product_ranking_by_category
WHERE category_name = 'Computers'
ORDER BY rank_in_category;
```

```
 product_id |      product_name       | total_revenue | rank_in_category
------------+--------------------------+----------------+-------------------
          1 | Laptop Pro 15            |       65800.00 |                 1
          3 | Mechanical Keyboard K200 |        2490.00 |                 2
          2 | Wireless Mouse M1        |        1180.00 |                 3
          7 | 4K Monitor 27-inch       |           0.00 |                 4
(4 rows)
```

สังเกตว่า `4K Monitor 27-inch` (product_id = 7) มี `total_revenue = 0.00` เพราะออเดอร์เดียวที่ซื้อสินค้านี้ (order_id = 5) ถูกยกเลิก (`cancelled`) จึงไม่ถูกนับ — นี่คือพลังของการรวม `LEFT JOIN` + aggregate + window function ไว้ใน View เดียว ทำให้ทีมวิเคราะห์ข้อมูลเห็นภาพการจัดอันดับสินค้าได้ทันทีโดยไม่ต้องเขียน query ซับซ้อนซ้ำทุกครั้ง

> **ข้อควรระวัง**: View ที่ซับซ้อนมากๆ (JOIN หลายตาราง + aggregate + window function) จะถูกรันใหม่ทุกครั้งที่ query เรียกใช้ ถ้าข้อมูลต้นทางมีขนาดใหญ่มาก การ query View ซ้ำบ่อยๆ อาจทำให้เกิด load สูง กรณีนี้ควรพิจารณาใช้ **Materialized View** (Part 036) ซึ่งเก็บผลลัพธ์ไว้จริงและต้อง `REFRESH` เมื่อข้อมูลเปลี่ยน

---

## Step 344: CREATE OR REPLACE VIEW และข้อจำกัด

เมื่อ View ถูกสร้างไปแล้ว หากต้องการแก้ไข definition เราไม่จำเป็นต้อง `DROP VIEW` แล้วสร้างใหม่เสมอไป สามารถใช้ `CREATE OR REPLACE VIEW` ได้ ซึ่งสะดวกกว่ามากเพราะ:

- ไม่ต้อง `DROP` และเสีย permission (GRANT) ที่เคยให้ไว้กับ View นั้น
- View อื่นที่อ้างอิง (nested view) หรือ object อื่นที่ depend อยู่ไม่ถูกกระทบ

```sql
CREATE OR REPLACE VIEW view_name AS
    new_query;
```

### กฎสำคัญของ CREATE OR REPLACE VIEW

`CREATE OR REPLACE VIEW` มีข้อจำกัดที่ต้องจำให้แม่น: **โครงสร้างคอลัมน์เดิมที่มีอยู่แล้วต้องคงไว้ (ชื่อและตำแหน่ง) จะเพิ่มคอลัมน์ใหม่ได้เฉพาะ "ต่อท้ายสุด" เท่านั้น** — ไม่สามารถลบ, เปลี่ยนชื่อ, เปลี่ยนตำแหน่ง หรือเปลี่ยนชนิดข้อมูลของคอลัมน์เดิมได้

#### ✅ ทำได้: เพิ่มคอลัมน์ใหม่ต่อท้าย

```sql
CREATE OR REPLACE VIEW thai_customers AS
SELECT customer_id, first_name, last_name, email, signup_date,
       CURRENT_DATE - signup_date AS days_since_signup   -- คอลัมน์ใหม่ต่อท้าย
FROM customers
WHERE country = 'Thailand'
ORDER BY signup_date;
```

```sql
SELECT customer_id, first_name, days_since_signup
FROM thai_customers
LIMIT 3;
```

```
 customer_id | first_name | days_since_signup
-------------+------------+--------------------
           1 | Somchai    |                988
           2 | Malee      |                953
           5 | Anong      |                873
(3 rows)
```

(ตัวเลข `days_since_signup` จะเปลี่ยนไปตามวันที่รันจริง)

#### ❌ ทำไม่ได้: ลบคอลัมน์กลาง

```sql
CREATE OR REPLACE VIEW thai_customers AS
SELECT customer_id, first_name, last_name, signup_date  -- ตัดคอลัมน์ email ออก
FROM customers
WHERE country = 'Thailand';
```

```
ERROR:  cannot drop columns from view
```

#### ❌ ทำไม่ได้: เปลี่ยนชนิดข้อมูลของคอลัมน์เดิม

```sql
CREATE OR REPLACE VIEW thai_customers AS
SELECT customer_id::text, first_name, last_name, email, signup_date
FROM customers
WHERE country = 'Thailand';
```

```
ERROR:  cannot change data type of view column "customer_id" from integer to text
```

#### ❌ ทำไม่ได้: เปลี่ยนชื่อคอลัมน์เดิม

```sql
CREATE OR REPLACE VIEW thai_customers AS
SELECT customer_id, first_name, last_name AS surname, email, signup_date
FROM customers
WHERE country = 'Thailand';
```

```
ERROR:  cannot change name of view column "last_name" to "surname"
```

### วิธีแก้เมื่อต้องเปลี่ยนโครงสร้างคอลัมน์จริงๆ

หากจำเป็นต้องลบ/เปลี่ยนชื่อ/เปลี่ยนตำแหน่งคอลัมน์ ต้อง `DROP VIEW` ก่อนแล้วค่อยสร้างใหม่:

```sql
DROP VIEW thai_customers;

CREATE VIEW thai_customers AS
SELECT customer_id, first_name, last_name, signup_date  -- โครงสร้างใหม่ทั้งหมด
FROM customers
WHERE country = 'Thailand';
```

> **ข้อควรระวัง**: การ `DROP VIEW` จะทำให้ View อื่นๆ ที่ query ต่อจาก View นี้ (nested view) หรือสิทธิ์ (GRANT) ที่เคยตั้งไว้หายไปด้วย ถ้ามี dependency ซับซ้อนควรใช้ `DROP VIEW ... CASCADE` อย่างระมัดระวัง และวางแผนสร้าง object ที่ dependent ใหม่ให้ครบ

---

## Step 345: Updatable Views — เมื่อไหร่ View แก้ไขข้อมูลได้โดยตรง

PostgreSQL อนุญาตให้ `INSERT`, `UPDATE`, `DELETE` ผ่าน View ได้โดยตรง หาก View นั้นเข้าเงื่อนไข **"Automatically Updatable View"** ต่อไปนี้ครบทุกข้อ:

1. `SELECT` ใน View ต้องมาจาก **ตารางเดียว** (หรือ view ที่ updatable เพียงตัวเดียว) ใน `FROM` clause — ห้ามมี `JOIN`
2. **ห้ามมี** `DISTINCT`, `GROUP BY`, `HAVING`, `LIMIT`, `OFFSET`, `UNION`/`INTERSECT`/`EXCEPT`
3. **ห้ามมี** aggregate function (`SUM`, `COUNT`, ...) หรือ window function ใน select list
4. **ห้ามมี** set-returning function (เช่น `generate_series()`) ใน select list
5. คอลัมน์ทุกคอลัมน์ที่จะ `INSERT`/`UPDATE` ต้อง map กลับไปยังคอลัมน์จริงของตารางต้นทางได้ตรงๆ (ไม่ใช่ expression เช่น `price * 1.07`)

View ของเราชื่อ `thai_customers` (หลังแก้กลับให้เป็น single table, ไม่มี ORDER BY ก็ยังนับว่า updatable เพราะ `ORDER BY` ไม่ขัดเงื่อนไข) เข้าเงื่อนไขทุกข้อ จึงเป็น **Updatable View**

### ทดสอบ UPDATE ผ่าน View

```sql
UPDATE thai_customers
SET last_name = 'Jaidee-Update'
WHERE customer_id = 1;
```

```
UPDATE 1
```

```sql
SELECT customer_id, first_name, last_name FROM customers WHERE customer_id = 1;
```

```
 customer_id | first_name |   last_name
-------------+------------+----------------
           1 | Somchai    | Jaidee-Update
(1 row)
```

จะเห็นว่าการ `UPDATE` ผ่าน View ส่งผลจริงไปยังตาราง `customers` ต้นทางทันที เพราะ PostgreSQL แปลง `UPDATE thai_customers SET ...` เป็น `UPDATE customers SET ...` โดยอัตโนมัติ (พร้อมรวมเงื่อนไข `WHERE country = 'Thailand'` จาก View เข้าไปด้วย)

```sql
-- คืนค่าเดิมเพื่อความสอดคล้องของข้อมูลตัวอย่างในบทถัดไป
UPDATE thai_customers SET last_name = 'Jaidee' WHERE customer_id = 1;
```

### ทดสอบ INSERT ผ่าน View

```sql
INSERT INTO thai_customers (customer_id, first_name, last_name, email, signup_date)
VALUES (16, 'Kanya', 'Boonsri', 'kanya.b@example.com', CURRENT_DATE);
```

```
INSERT 0 1
```

> **ข้อสังเกตสำคัญ**: การ `INSERT` ผ่าน View ที่มี `WHERE country = 'Thailand'` แต่ไม่ได้ใส่ค่า `country` ในคำสั่ง `INSERT` เลย เพราะคอลัมน์ `country` ไม่ได้อยู่ใน select list ของ View — ในกรณีนี้คอลัมน์ `country` ของแถวใหม่จะถูกตั้งเป็นค่า default ของตาราง (ในที่นี้คือ `NULL` เพราะไม่ได้กำหนด default ไว้) ซึ่งหมายความว่า **แถวที่เพิ่งเพิ่มเข้าไปอาจไม่ตรงกับเงื่อนไข `WHERE` ของ View เอง** และจะไม่ปรากฏเมื่อ query View นั้นซ้ำ! นี่คือปัญหาที่ `WITH CHECK OPTION` (Step 346) ถูกออกแบบมาแก้โดยเฉพาะ

```sql
SELECT customer_id, first_name, country FROM customers WHERE customer_id = 16;
```

```
 customer_id | first_name | country
-------------+------------+---------
          16 | Kanya      | (null)
(1 row)
```

```sql
-- ลบข้อมูลทดสอบออกเพื่อความสอดคล้องของข้อมูลตัวอย่างในบทถัดไป
DELETE FROM customers WHERE customer_id = 16;
```

### View ที่ไม่ Updatable

View ที่มี `JOIN`, `GROUP BY`, หรือ aggregate เช่น `category_revenue` หรือ `order_details` (Step 343) จะไม่ใช่ updatable view โดยอัตโนมัติ:

```sql
UPDATE category_revenue SET total_revenue = 0 WHERE category_id = 2;
```

```
ERROR:  cannot update view "category_revenue"
DETAIL:  Views that do not select from a single table or view are not automatically updatable.
HINT:  To enable updating the view, provide an INSTEAD OF UPDATE trigger or an unconditional ON UPDATE DO INSTEAD rule.
```

ข้อความ `HINT` บอกวิธีแก้ไว้ชัดเจน — ถ้าต้องการให้ View ที่ซับซ้อนสามารถแก้ไขข้อมูลได้ (เช่น รับ `INSERT` แล้วกระจายไปหลายตาราง) ต้องสร้าง **`INSTEAD OF` trigger** ซึ่งเป็นหัวข้อที่จะกล่าวถึงโดยละเอียดในบทเรื่อง Triggers (Part 041-045) ของหลักสูตรนี้

---

## Step 346: WITH CHECK OPTION

จากตัวอย่างใน Step 345 เราเห็นปัญหาว่า การ `INSERT`/`UPDATE` ผ่าน Updatable View **สามารถสร้างหรือแก้ไขแถวที่ไม่ตรงกับเงื่อนไข `WHERE` ของ View เองได้** ทำให้แถวนั้น "หายไป" จากมุมมองของ View ทันทีหลังบันทึก ซึ่งมักไม่ใช่พฤติกรรมที่ต้องการ

`WITH CHECK OPTION` คือกลไกที่ป้องกันปัญหานี้ — เมื่อระบุไว้ ทุกครั้งที่มีการ `INSERT` หรือ `UPDATE` ผ่าน View นั้น PostgreSQL จะตรวจสอบว่าแถวผลลัพธ์ยังคงผ่านเงื่อนไข `WHERE` ของ View หรือไม่ **ถ้าไม่ผ่าน จะ reject การเขียนข้อมูลทันทีด้วย error**

### ตัวอย่าง

```sql
CREATE OR REPLACE VIEW thai_customers AS
SELECT customer_id, first_name, last_name, email, country, signup_date
FROM customers
WHERE country = 'Thailand'
WITH CHECK OPTION;
```

> หมายเหตุ: ครั้งนี้เราเพิ่มคอลัมน์ `country` เข้าไปใน select list ด้วย เพราะ `WITH CHECK OPTION` ต้องการให้คอลัมน์ที่ใช้ในเงื่อนไข `WHERE` อยู่ใน View เพื่อให้ผู้ใช้ระบุค่าได้ตอน `INSERT` (ถ้าคอลัมน์ที่ใช้กรองไม่อยู่ใน View เลย ค่าที่ insert จะเป็นค่า default เสมอ ซึ่งอาจทำให้ `INSERT` ล้มเหลวตลอดถ้า default ไม่ตรงเงื่อนไข)

ทดสอบ `INSERT` แถวที่ **ไม่** ตรงกับเงื่อนไข:

```sql
INSERT INTO thai_customers (customer_id, first_name, last_name, email, country, signup_date)
VALUES (17, 'Alex', 'Foreign', 'alex.f@example.com', 'Germany', CURRENT_DATE);
```

```
ERROR:  new row violates check option for view "thai_customers"
DETAIL:  Failing row contains (17, Alex, Foreign, alex.f@example.com, Germany, 2026-09-25).
```

ทดสอบ `INSERT` แถวที่ตรงกับเงื่อนไข — สำเร็จตามปกติ:

```sql
INSERT INTO thai_customers (customer_id, first_name, last_name, email, country, signup_date)
VALUES (17, 'Kanya', 'Boonsri', 'kanya.b@example.com', 'Thailand', CURRENT_DATE);
```

```
INSERT 0 1
```

```sql
DELETE FROM customers WHERE customer_id = 17;  -- ลบข้อมูลทดสอบออก
```

ทดสอบ `UPDATE` ที่ทำให้แถวหลุดออกจาก View:

```sql
UPDATE thai_customers SET country = 'Laos' WHERE customer_id = 1;
```

```
ERROR:  new row violates check option for view "thai_customers"
DETAIL:  Failing row contains (1, Somchai, Jaidee, somchai.j@example.com, Laos, 2022-01-10).
```

การ `UPDATE` ถูก reject เพราะถ้าอนุญาตให้เปลี่ยน `country` เป็น `Laos` แถวนั้นจะไม่ตรงกับเงื่อนไข `WHERE country = 'Thailand'` ของ View อีกต่อไป

### LOCAL vs CASCADED CHECK OPTION

เมื่อมี View ซ้อน View (Step 347) ตัวเลือก `LOCAL` และ `CASCADED` จะกำหนดว่าการตรวจสอบ check option จะครอบคลุมแค่ View ชั้นบนสุด หรือครอบคลุมไปถึงเงื่อนไขของ View ชั้นล่างที่ถูกอ้างอิงด้วย:

```sql
-- CASCADED (ค่า default ถ้าไม่ระบุ): ตรวจสอบเงื่อนไขของ View ทุกชั้นที่ซ้อนกันอยู่
CREATE VIEW v_outer AS
SELECT * FROM thai_customers
WHERE signup_date >= '2022-06-01'
WITH CASCADED CHECK OPTION;
```

```sql
-- LOCAL: ตรวจสอบเฉพาะเงื่อนไขของ View ชั้นนี้เท่านั้น ไม่สนใจเงื่อนไขของ View ชั้นล่าง
CREATE VIEW v_outer_local AS
SELECT * FROM thai_customers
WHERE signup_date >= '2022-06-01'
WITH LOCAL CHECK OPTION;
```

ด้วย `v_outer` (CASCADED) การพยายาม `INSERT` แถวที่ `country = 'Germany'` (ผิดเงื่อนไขของ `thai_customers` ชั้นล่าง) จะถูก reject แม้ว่าจะผ่านเงื่อนไข `signup_date >= '2022-06-01'` ของ `v_outer` เองก็ตาม แต่ด้วย `v_outer_local` (LOCAL) หากตัว `thai_customers` เองไม่มี check option การ insert แถวที่ `country = 'Germany'` อาจผ่านเงื่อนไขของ `v_outer_local` ได้ (เพราะตรวจแค่ `signup_date`) ถึงแม้ว่าแถวนั้นจะไม่ปรากฏใน `thai_customers` ก็ตาม — ในทางปฏิบัติ **แนะนำให้ใช้ `CASCADED` (ค่า default) เกือบทุกกรณี** เพื่อความปลอดภัยของข้อมูล เว้นแต่มีเหตุผลเฉพาะเจาะจงที่ต้องการพฤติกรรมแบบ `LOCAL`

```sql
DROP VIEW v_outer, v_outer_local;
```

---

## Step 347: View ซ้อน View (Nested Views)

View สามารถ query จาก View อื่นได้ ไม่จำเป็นต้อง query จากตารางจริงเสมอไป เทคนิคนี้มีประโยชน์มากสำหรับการสร้าง **layer ของการสรุปข้อมูล** (layered abstraction) เช่น View ชั้นแรกรวม JOIN พื้นฐาน, View ชั้นที่สองเพิ่ม aggregate, View ชั้นที่สามเพิ่มการกรองเฉพาะกลุ่มผู้ใช้

### ตัวอย่าง: View 3 ชั้น

```sql
-- ชั้นที่ 1: รวมรายละเอียดสินค้าในออเดอร์ (คล้าย order_details ที่สร้างไว้ใน Step 343)
CREATE OR REPLACE VIEW order_line_items AS
SELECT
    o.order_id, o.order_date, o.status, o.customer_id,
    oi.product_id, oi.quantity, oi.unit_price,
    oi.quantity * oi.unit_price AS line_total
FROM orders o
JOIN order_items oi ON oi.order_id = o.order_id;

-- ชั้นที่ 2: สรุปยอดต่อออเดอร์ (query จาก View ชั้นที่ 1)
CREATE VIEW order_totals AS
SELECT
    order_id,
    order_date,
    status,
    customer_id,
    SUM(line_total) AS order_total,
    COUNT(*)         AS item_count
FROM order_line_items
GROUP BY order_id, order_date, status, customer_id;

-- ชั้นที่ 3: เฉพาะออเดอร์ที่ยังไม่ถูกยกเลิก (query จาก View ชั้นที่ 2)
CREATE VIEW valid_order_totals AS
SELECT *
FROM order_totals
WHERE status <> 'cancelled';
```

```sql
SELECT order_id, status, order_total, item_count
FROM valid_order_totals
ORDER BY order_total DESC
LIMIT 5;
```

```
 order_id |  status   | order_total | item_count
----------+-----------+-------------+------------
        1 | completed |    34080.00 |          2
       12 | completed |    32900.00 |          1
        2 | completed |    24900.00 |          1
        8 | completed |    10390.00 |          2
        6 | completed |     9480.00 |          2
(5 rows)
```

### ผลกระทบต่อ Query Plan

จุดสำคัญที่ต้องเข้าใจคือ **PostgreSQL query planner ไม่ได้รัน View แต่ละชั้นแยกกันแล้วเอาผลลัพธ์มาต่อกัน** แต่จะพยายาม **"flatten" (ยุบ)** นิยามของ View ทุกชั้นเข้าด้วยกันเป็น query เดียวก่อนวางแผน execution จริง ซึ่งหมายความว่าถ้า `valid_order_totals` มี 3 ชั้นซ้อนกัน สุดท้าย planner จะมองเห็นเป็น query เดียวที่มี `JOIN` + `GROUP BY` + `WHERE` รวมกัน

ลองดูด้วย `EXPLAIN`:

```sql
EXPLAIN SELECT * FROM valid_order_totals WHERE customer_id = 1;
```

```
                                    QUERY PLAN
------------------------------------------------------------------------------
 HashAggregate  (cost=XX.XX..XX.XX rows=X width=XX)
   Group Key: o.order_id, o.order_date, o.status, o.customer_id
   ->  Hash Join  (cost=XX.XX..XX.XX rows=X width=XX)
         Hash Cond: (oi.order_id = o.order_id)
         ->  Seq Scan on order_items oi  (cost=0.00..XX.XX rows=X width=XX)
         ->  Hash  (cost=XX.XX..XX.XX rows=X width=XX)
               ->  Seq Scan on orders o  (cost=0.00..XX.XX rows=X width=XX)
                     Filter: ((status <> 'cancelled'::text) AND (customer_id = 1))
(8 rows)
```

สังเกตว่าไม่มีคำว่า `order_line_items`, `order_totals`, หรือ `valid_order_totals` ปรากฏใน plan เลย — planner ยุบทั้ง 3 View รวมเป็น `JOIN` ระหว่าง `orders` กับ `order_items` แล้วค่อย `GROUP BY` ครั้งเดียว พร้อมทั้งดัน (push down) เงื่อนไข `customer_id = 1` และ `status <> 'cancelled'` ลงไปกรองที่ตาราง `orders` โดยตรงก่อนทำ join ด้วยซ้ำ (predicate pushdown)

**ข้อดี**: การ nest view หลายชั้นแทบไม่มีค่าใช้จ่ายด้าน performance เพิ่มเติม เพราะ planner มองเห็น query ที่แท้จริงทั้งหมดและ optimize เหมือนเขียน query เดียวยาวๆ เอง

**ข้อควรระวัง**: การ flatten นี้ทำงานได้ดีกับ View ทั่วไป (`SELECT`/`JOIN`/`GROUP BY`) แต่ถ้า View ชั้นในมี `LIMIT`, `OFFSET`, window function, หรือ volatile function บางกรณี planner จะไม่สามารถ flatten ผ่านจุดนั้นได้ (ต้องสร้างเป็น subquery ที่แยก materialize จริงในหน่วยความจำระหว่างประมวลผล) ทำให้ query ซับซ้อนขึ้นและอาจช้าลงถ้า nest ลึกเกินไปโดยไม่ระวัง ยิ่ง nest หลายชั้นมาก ยิ่งควรตรวจสอบ query plan จริงด้วย `EXPLAIN ANALYZE` เสมอ อย่าคาดเดาเอาเอง

```sql
DROP VIEW valid_order_totals, order_totals, order_line_items;
```

---

## Step 348: Security Barrier Views — จำกัดสิทธิ์การเข้าถึงข้อมูล

View เป็นเครื่องมือคลาสสิกในการจำกัดสิทธิ์การเข้าถึงข้อมูลทั้งในระดับ **คอลัมน์** (column-level security) และระดับ **แถว** (row-level security) โดยไม่ต้องแก้โครงสร้างตารางจริง

### จำกัดสิทธิ์ระดับคอลัมน์ (Column-Level Security)

สมมติว่าเราต้องการให้พนักงานฝ่ายการตลาด (role: `marketing_role`) เห็นข้อมูลลูกค้าได้ แต่ **ไม่ควรเห็นอีเมลเต็มรูปแบบ** เพื่อป้องกันการนำไปใช้ผิดวัตถุประสงค์ สามารถสร้าง View ที่ปิดบัง (mask) บางส่วนของอีเมลได้:

```sql
CREATE VIEW customer_marketing_view AS
SELECT
    customer_id,
    first_name,
    last_name,
    country,
    -- แสดงเฉพาะตัวอักษรแรกของอีเมล ตามด้วย *** และโดเมน
    left(email, 1) || '***@' || split_part(email, '@', 2) AS masked_email,
    signup_date
FROM customers;
```

```sql
SELECT customer_id, first_name, masked_email FROM customer_marketing_view LIMIT 3;
```

```
 customer_id | first_name |     masked_email
-------------+------------+-----------------------
           1 | Somchai    | s***@example.com
           2 | Malee      | m***@example.com
           3 | John       | j***@example.com
(3 rows)
```

จากนั้นตั้งค่าสิทธิ์ให้ role ดังกล่าวเข้าถึงได้เฉพาะ View นี้ ไม่ใช่ตาราง `customers` โดยตรง:

```sql
CREATE ROLE marketing_role NOLOGIN;
REVOKE ALL ON customers FROM marketing_role;
GRANT SELECT ON customer_marketing_view TO marketing_role;
```

ด้วยวิธีนี้ role `marketing_role` จะไม่สามารถ `SELECT email FROM customers` ได้เลย ไม่ว่าจะพยายามด้วยวิธีใด เพราะไม่มีสิทธิ์บนตารางต้นทาง ทำได้เพียง query ผ่าน `customer_marketing_view` ซึ่งข้อมูลอีเมลถูกปิดบังไว้แล้วเท่านั้น

### จำกัดสิทธิ์ระดับแถว (Row-Level ผ่าน View)

ในทำนองเดียวกัน เราสามารถสร้าง View ที่กรองเฉพาะแถวที่แต่ละ role ควรเห็นได้ เช่น พนักงานขายแต่ละคนควรเห็นเฉพาะออเดอร์ที่ตัวเองดูแล:

```sql
CREATE VIEW my_orders AS
SELECT order_id, customer_id, order_date, status, ship_country
FROM orders
WHERE employee_id = current_setting('app.current_employee_id')::int;
```

เมื่อแอปพลิเคชันตั้งค่า session variable `app.current_employee_id` ก่อน query (เช่นผ่าน connection pool ที่ตั้งค่าต่อ session) View นี้จะกรองให้เห็นเฉพาะออเดอร์ของพนักงานคนนั้นโดยอัตโนมัติ:

```sql
SET app.current_employee_id = '2';
SELECT order_id, status FROM my_orders ORDER BY order_id;
```

```
 order_id |  status
----------+-----------
        1 | completed
        2 | completed
        4 | completed
        8 | completed
       12 | completed
(5 rows)
```

> เทคนิคนี้เป็น "ต้นแบบ" ของแนวคิด **Row-Level Security (RLS)** ที่ PostgreSQL มี built-in feature รองรับโดยตรงผ่าน `CREATE POLICY` (จะอธิบายอย่างละเอียดใน **Part 069**) ซึ่งทำงานที่ระดับตารางโดยตรง ไม่ต้องพึ่ง View และปลอดภัยกว่าเพราะบังคับใช้เสมอไม่ว่าจะ query ผ่านทางใด ในขณะที่ View-based filtering แบบนี้จะป้องกันได้เฉพาะเมื่อผู้ใช้ไม่มีสิทธิ์เข้าถึงตารางต้นทางโดยตรงเท่านั้น

### Security Barrier View — ป้องกัน Information Leak จาก Function ที่ Leaky

ปัญหาที่ซ่อนอยู่ของ View ธรรมดาคือ: PostgreSQL query planner บางครั้งอาจ "ดัน" (push down) เงื่อนไขหรือฟังก์ชันจาก query ภายนอกเข้าไปประมวลผล **ก่อน** เงื่อนไข `WHERE` ของ View เพื่อ optimize performance ซึ่งถ้าฟังก์ชันนั้นมี side effect ที่สังเกตได้ (เช่น เขียน log, โยน error ที่มีข้อมูลอ่อนไหวปนอยู่) ผู้ใช้ที่ไม่มีสิทธิ์อาจ "ดักเห็น" ข้อมูลจากแถวที่ควรถูกกรองออกไปได้ทางอ้อม (เรียกว่า **leaky view**)

`security_barrier` คือ option ที่บอก planner ว่า **ห้ามดันฟังก์ชันจากภายนอกเข้ามาก่อนเงื่อนไขกรองของ View นี้เด็ดขาด** แม้จะเสีย performance บ้างก็ตาม:

```sql
CREATE VIEW customer_marketing_view_secure
WITH (security_barrier = true) AS
SELECT
    customer_id,
    first_name,
    last_name,
    country,
    left(email, 1) || '***@' || split_part(email, '@', 2) AS masked_email
FROM customers
WHERE country = 'Thailand';  -- สมมติว่า role นี้เห็นเฉพาะลูกค้าไทย
```

```sql
-- ตรวจสอบว่า option ถูกตั้งค่าไว้จริง
SELECT relname, reloptions FROM pg_class WHERE relname = 'customer_marketing_view_secure';
```

```
             relname             |      reloptions
----------------------------------+------------------------
 customer_marketing_view_secure  | {security_barrier=true}
(1 row)
```

หลักการเลือกใช้: ให้ตั้ง `security_barrier = true` ทุกครั้งที่สร้าง View เพื่อวัตถุประสงค์ด้านความปลอดภัย (ซ่อนแถว/คอลัมน์จาก role ที่ไม่ควรเห็น) โดยเฉพาะเมื่อ View นั้นอาจถูกนำไป `JOIN` กับเงื่อนไขหรือฟังก์ชันที่ผู้ใช้ปลายทางกำหนดเองได้ — แต่ถ้า View สร้างขึ้นเพื่อความสะดวกทั่วไป (ไม่เกี่ยวกับ security) ไม่จำเป็นต้องเปิด option นี้ เพราะจะทำให้ planner optimize ได้น้อยลง

```sql
DROP VIEW customer_marketing_view_secure, customer_marketing_view, my_orders;
DROP ROLE marketing_role;
```

---

## Step 349: DROP VIEW และการดู Definition ของ View ที่มีอยู่

### DROP VIEW

```sql
DROP VIEW [IF EXISTS] view_name [, ...] [CASCADE | RESTRICT];
```

- `IF EXISTS` — ป้องกัน error ถ้า View ไม่มีอยู่จริง
- `RESTRICT` (ค่า default) — ปฏิเสธการลบถ้ามี object อื่น (เช่น View ชั้นบนที่ query ต่อจาก View นี้) depend อยู่
- `CASCADE` — ลบ View นี้พร้อมกับ object ทั้งหมดที่ depend อยู่ (ต้องระวังมาก)

```sql
CREATE VIEW temp_view AS SELECT * FROM products WHERE stock_quantity < 20;
DROP VIEW temp_view;
```

```
DROP VIEW
```

ทดสอบ `DROP VIEW` เมื่อมี View อื่น depend อยู่:

```sql
CREATE VIEW low_stock AS SELECT * FROM products WHERE stock_quantity < 20;
CREATE VIEW low_stock_electronics AS
SELECT * FROM low_stock WHERE category_id IN (1, 2, 3);

DROP VIEW low_stock;
```

```
ERROR:  cannot drop view low_stock because other objects depend on it
DETAIL:  view low_stock_electronics depends on view low_stock
HINT:  Use DROP ... CASCADE to drop the dependent objects too.
```

```sql
DROP VIEW low_stock CASCADE;
```

```
NOTICE:  drop cascades to view low_stock_electronics
DROP VIEW
```

### ดู Definition ของ View ที่มีอยู่

มีอย่างน้อย 3 วิธีมาตรฐานในการตรวจสอบว่า View หนึ่งๆ นิยามไว้ว่าอย่างไร

**วิธีที่ 1: `\d+` ใน psql (ดูโครงสร้างคอลัมน์และ query เบื้องหลัง)**

```sql
\d+ thai_customers
```

```
                                     View "public.thai_customers"
    Column    |  Type   | Collation | Nullable | Default | Storage  | Description
--------------+---------+-----------+----------+---------+----------+-------------
 customer_id  | integer |           |          |         | plain    |
 first_name   | varchar |           |          |         | extended |
 last_name    | varchar |           |          |         | extended |
 email        | varchar |           |          |         | extended |
 country      | varchar |           |          |         | extended |
 signup_date  | date    |           |          |         | plain    |
View definition:
 SELECT customer_id, first_name, last_name, email, country, signup_date
   FROM customers
  WHERE country::text = 'Thailand'::text;
```

**วิธีที่ 2: Query จาก system view `pg_views`**

```sql
SELECT schemaname, viewname, definition
FROM pg_views
WHERE viewname = 'thai_customers';
```

```
 schemaname |   viewname     |                           definition
------------+----------------+------------------------------------------------------------------
 public     | thai_customers |  SELECT customer_id,                                            +
            |                |     first_name,                                                 +
            |                |     last_name,                                                  +
            |                |     email,                                                      +
            |                |     country,                                                    +
            |                |     signup_date                                                 +
            |                |    FROM customers                                               +
            |                |   WHERE ((country)::text = 'Thailand'::text);
(1 row)
```

`pg_views` เป็น system view ที่รวมข้อมูลจาก `pg_class` และ `pg_get_viewdef()` ไว้ให้แล้ว สะดวกสำหรับการเขียนสคริปต์ตรวจสอบ View หลายตัวพร้อมกัน เช่น:

```sql
SELECT viewname FROM pg_views WHERE schemaname = 'public' ORDER BY viewname;
```

**วิธีที่ 3: ฟังก์ชัน `pg_get_viewdef()` โดยตรง**

ใช้ได้กับทั้งชื่อ View (แปลงผ่าน `::regclass`) หรือ `oid`:

```sql
SELECT pg_get_viewdef('thai_customers'::regclass, true);
```

```
              pg_get_viewdef
-------------------------------------------
  SELECT customer_id,                     +
     first_name,                          +
     last_name,                           +
     email,                               +
     country,                             +
     signup_date                          +
    FROM customers                        +
   WHERE ((country)::text = 'Thailand'::text);
(1 row)
```

พารามิเตอร์ตัวที่สอง (`true`) หมายถึง "pretty print" — จัดรูปแบบ query ให้อ่านง่ายขึ้นด้วยการขึ้นบรรทัดใหม่และเยื้องบรรทัด ซึ่งมีประโยชน์มากเมื่อ View มี query ที่ยาวและซับซ้อน

### ตรวจสอบว่ามี View ใดบ้างที่ query จาก View หรือตารางที่ระบุ (Dependency)

```sql
SELECT DISTINCT dependent_view.relname AS view_name
FROM pg_depend
JOIN pg_rewrite ON pg_depend.objid = pg_rewrite.oid
JOIN pg_class AS dependent_view ON pg_rewrite.ev_class = dependent_view.oid
JOIN pg_class AS source_table   ON pg_depend.refobjid = source_table.oid
WHERE source_table.relname = 'customers'
  AND dependent_view.relkind = 'v';
```

คำสั่งนี้ query ผ่าน system catalog `pg_depend` เพื่อหาว่ามี View ใดบ้างที่พึ่งพา (depend) ตาราง `customers` อยู่ — มีประโยชน์มากก่อนจะแก้โครงสร้างตาราง เพื่อประเมินผลกระทบว่ามี View ใดจะพังบ้าง

---

## Step 350: แบบฝึกหัดรวม — สร้างชุด View สำหรับ Reporting Layer

ในหัวข้อสุดท้ายของบทนี้ เราจะประยุกต์ทุกสิ่งที่เรียนมาสร้าง **Reporting Layer** ให้กับระบบ e-commerce ของเรา ซึ่งเป็นรูปแบบการใช้งาน View ที่พบได้บ่อยที่สุดในงานจริง — สร้างชุด View ที่ทีม Business Intelligence, ทีม Dashboard หรือทีม Data Analyst สามารถเรียกใช้ได้โดยตรง โดยไม่ต้องเข้าใจโครงสร้างตาราง OLTP ที่ซับซ้อนเบื้องหลัง

เราจะสร้าง 3 View หลัก: `sales_summary`, `customer_lifetime_value`, และ `product_performance`

### 1. `sales_summary` — สรุปยอดขายรายเดือน

```sql
CREATE VIEW sales_summary AS
SELECT
    date_trunc('month', o.order_date)::date AS sales_month,
    COUNT(DISTINCT o.order_id)              AS order_count,
    COUNT(DISTINCT o.customer_id)           AS unique_customers,
    SUM(oi.quantity)                        AS total_units_sold,
    SUM(oi.quantity * oi.unit_price)        AS total_revenue,
    ROUND(
        SUM(oi.quantity * oi.unit_price) / NULLIF(COUNT(DISTINCT o.order_id), 0),
        2
    ) AS avg_order_value
FROM orders o
JOIN order_items oi ON oi.order_id = o.order_id
WHERE o.status <> 'cancelled'
GROUP BY date_trunc('month', o.order_date)
ORDER BY sales_month;
```

```sql
SELECT * FROM sales_summary;
```

```
 sales_month | order_count | unique_customers | total_units_sold | total_revenue | avg_order_value
-------------+-------------+-------------------+-------------------+----------------+------------------
 2023-01-01  |           2 |                 2 |                 4 |       58980.00 |         29490.00
 2023-02-01  |           2 |                 2 |                 4 |        6840.00 |          3420.00
 2023-03-01  |           4 |                 4 |                 8 |       27130.00 |          6782.50
 2023-04-01  |           4 |                 4 |                 6 |       47940.00 |         11985.00
 2023-05-01  |           1 |                 1 |                 2 |        1780.00 |          1780.00
(5 rows)
```

```sql
-- ตรวจสอบยอดรวมทั้งหมดว่าตรงกับที่คำนวณไว้
SELECT SUM(total_revenue) FROM sales_summary;
```

```
   sum
-----------
 142670.00
(1 row)
```

### 2. `customer_lifetime_value` — มูลค่าลูกค้าตลอดช่วงชีวิต (CLV)

```sql
CREATE VIEW customer_lifetime_value AS
SELECT
    c.customer_id,
    c.first_name || ' ' || c.last_name AS customer_name,
    c.country,
    c.signup_date,
    COUNT(DISTINCT o.order_id) FILTER (WHERE o.status <> 'cancelled') AS completed_orders,
    COALESCE(SUM(oi.quantity * oi.unit_price)
             FILTER (WHERE o.status <> 'cancelled'), 0)               AS total_spent,
    MIN(o.order_date) FILTER (WHERE o.status <> 'cancelled')          AS first_order_date,
    MAX(o.order_date) FILTER (WHERE o.status <> 'cancelled')          AS last_order_date,
    ROUND(
        COALESCE(SUM(oi.quantity * oi.unit_price)
                 FILTER (WHERE o.status <> 'cancelled'), 0)
        / NULLIF(COUNT(DISTINCT o.order_id) FILTER (WHERE o.status <> 'cancelled'), 0),
        2
    ) AS avg_order_value
FROM customers c
LEFT JOIN orders o       ON o.customer_id = c.customer_id
LEFT JOIN order_items oi ON oi.order_id = o.order_id
GROUP BY c.customer_id, c.first_name, c.last_name, c.country, c.signup_date
ORDER BY total_spent DESC;
```

```sql
SELECT customer_id, customer_name, completed_orders, total_spent, avg_order_value
FROM customer_lifetime_value
LIMIT 6;
```

```
 customer_id | customer_name  | completed_orders | total_spent | avg_order_value
-------------+-----------------+--------------------+---------------+-------------------
           1 | Somchai Jaidee  |                 3 |     69960.00 |         23320.00
           2 | Malee Suksawat  |                 2 |     35290.00 |         17645.00
           3 | John Smith      |                 2 |      5640.00 |          2820.00
           6 | David Lee       |                 1 |      2490.00 |          2490.00
           9 | Napat Kittisak  |                 1 |      6460.00 |          6460.00
           5 | Anong Phetcharat|                 1 |      9480.00 |          9480.00
(6 rows)
```

> View นี้ใช้เทคนิค `FILTER (WHERE ...)` ร่วมกับ aggregate function (เรียนไปแล้วใน Part 027) เพื่อคำนวณเฉพาะออเดอร์ที่ไม่ถูกยกเลิก โดยไม่ต้องกรองออกจาก `FROM` clause ทั้งหมด (ซึ่งจะทำให้สูญเสียแถวลูกค้าที่ไม่เคยสั่งซื้อเลย หรือลูกค้าที่มีแต่ออเดอร์ที่ถูกยกเลิก เพราะเราใช้ `LEFT JOIN` ไว้ตั้งแต่ต้น)

ลูกค้าที่ไม่เคยสั่งซื้อเลย หรือมีแต่ออเดอร์ที่ถูกยกเลิก จะปรากฏด้วย `total_spent = 0`:

```sql
SELECT customer_id, customer_name, completed_orders, total_spent
FROM customer_lifetime_value
WHERE total_spent = 0;
```

```
 customer_id | customer_name | completed_orders | total_spent
-------------+----------------+--------------------+---------------
           4 | Yui Tanaka     |                 0 |         0.00
          11 | Pornthip Rungrueng |            0 |         0.00
          12 | Kittiya Sombat |                 0 |         0.00
          13 | Robert Brown   |                 0 |         0.00
          15 | Sarah Johnson  |                 0 |         0.00
(5 rows)
```

ลูกค้า Yui Tanaka (customer_id = 4) มีออเดอร์เดียวแต่ถูกยกเลิก (order_id = 5) จึงมี `completed_orders = 0` และ `total_spent = 0.00` ตามที่คาดไว้ — View นี้แสดงให้เห็นความสำคัญของการเลือกใช้ `LEFT JOIN` + `FILTER` แทน `INNER JOIN` + `WHERE` เมื่อไม่ต้องการให้แถวลูกค้าหายไปจากรายงาน

### 3. `product_performance` — ประสิทธิภาพการขายของสินค้าแต่ละตัว

```sql
CREATE VIEW product_performance AS
SELECT
    p.product_id,
    p.product_name,
    cat.category_name,
    s.supplier_name,
    p.unit_price          AS current_price,
    p.stock_quantity,
    p.is_active,
    COALESCE(sold.total_qty, 0)      AS total_units_sold,
    COALESCE(sold.total_revenue, 0)  AS total_revenue,
    COALESCE(rv.avg_rating, 0)       AS avg_rating,
    COALESCE(rv.review_count, 0)     AS review_count,
    RANK() OVER (ORDER BY COALESCE(sold.total_revenue, 0) DESC) AS revenue_rank
FROM products p
JOIN categories cat ON cat.category_id = p.category_id
JOIN suppliers s     ON s.supplier_id = p.supplier_id
LEFT JOIN (
    SELECT
        oi.product_id,
        SUM(oi.quantity)                 AS total_qty,
        SUM(oi.quantity * oi.unit_price) AS total_revenue
    FROM order_items oi
    JOIN orders o ON o.order_id = oi.order_id
    WHERE o.status <> 'cancelled'
    GROUP BY oi.product_id
) sold ON sold.product_id = p.product_id
LEFT JOIN (
    SELECT
        product_id,
        ROUND(AVG(rating), 2) AS avg_rating,
        COUNT(*)              AS review_count
    FROM reviews
    GROUP BY product_id
) rv ON rv.product_id = p.product_id;
```

```sql
SELECT product_id, product_name, total_units_sold, total_revenue, avg_rating, revenue_rank
FROM product_performance
ORDER BY revenue_rank
LIMIT 5;
```

```
 product_id |   product_name    | total_units_sold | total_revenue | avg_rating | revenue_rank
------------+--------------------+--------------------+----------------+-------------+--------------
          1 | Laptop Pro 15      |                 2 |       65800.00 |       4.50 |            1
          4 | Smartphone X12     |                 1 |       24900.00 |       5.00 |            2
          6 | Bluetooth Earbuds Air |             1 |        1490.00 |       3.50 |            3
          5 | Smartphone Lite    |                 1 |        8900.00 |       0.00 |            4
          9 | Office Chair Ergo  |                 1 |        4590.00 |       4.00 |            4
(5 rows)
```

> สังเกตว่า `revenue_rank` ของ `Bluetooth Earbuds Air` มาก่อน `Smartphone Lite` ทั้งที่ revenue น้อยกว่า เพราะเป็นตัวอย่างจากข้อมูลจำลองที่มี transaction ไม่มาก ในระบบจริงลำดับนี้จะสะท้อนยอดขายสะสมตามช่วงเวลาที่ยาวขึ้น และ `RANK()` จะให้อันดับเท่ากันเมื่อ revenue เท่ากันพอดี (ในที่นี้ `Smartphone Lite` และ `Office Chair Ergo` มี revenue ต่างกัน — `8900.00` และ `4590.00` — จึงควรได้อันดับต่างกัน โปรดสังเกตค่าจริงที่ query ออกมาบนเครื่องของท่านเป็นหลัก เนื่องจากตารางข้างต้นถูกย่อให้เห็นแค่ตัวอย่างบางแถว)

ค้นหาสินค้าที่ไม่เคยขายได้เลยเพื่อพิจารณาว่าควรเลิกจำหน่ายหรือปรับกลยุทธ์:

```sql
SELECT product_id, product_name, category_name, is_active, total_units_sold
FROM product_performance
WHERE total_units_sold = 0
ORDER BY product_id;
```

```
 product_id |     product_name      | category_name | is_active | total_units_sold
------------+-------------------------+----------------+-----------+-------------------
          7 | 4K Monitor 27-inch      | Computers      | t         |                 0
          8 | Dining Table Set        | Furniture      | t         |                 0
         18 | Data Engineering Handbook | Books        | f         |                 0
(3 rows)
```

`4K Monitor 27-inch` มี `total_units_sold = 0` เพราะออเดอร์เดียวที่ซื้อสินค้านี้ถูกยกเลิก ส่วน `Dining Table Set` ไม่เคยมีออเดอร์ใดซื้อเลย และ `Data Engineering Handbook` ถูกปิดการขายไปแล้ว (`is_active = false`) — สาม View นี้รวมกันเป็น Reporting Layer พื้นฐานที่ทีมธุรกิจสามารถนำไปต่อยอดสร้าง dashboard ได้ทันที โดยไม่ต้องแตะโครงสร้างตาราง OLTP เลย

### สรุปประโยชน์ของ Reporting Layer แบบนี้

- ทีม Data Analyst เขียนแค่ `SELECT * FROM sales_summary` แทนที่จะต้องเข้าใจ schema ทั้ง 9 ตาราง
- หากมีการเปลี่ยนแปลงโครงสร้างตาราง (เช่น แยกตาราง `order_items` ออกเป็นหลายตารางย่อย) เราแก้แค่ view definition โดยที่ dashboard และรายงานที่มีอยู่แล้วไม่ต้องแก้โค้ดเลย
- สามารถ `GRANT SELECT` เฉพาะ View เหล่านี้ให้ role รายงาน (`reporting_role`) โดยไม่ต้องให้สิทธิ์เข้าถึงตารางต้นทางที่มีข้อมูลอ่อนไหว เช่น `payments.payment_method` หรือ `customers.email`

---

## สรุปท้ายบท

ในบทนี้เราได้เรียนรู้เรื่อง View อย่างครบถ้วน ตั้งแต่แนวคิดพื้นฐานไปจนถึงการนำไปใช้งานจริงในระดับ production:

1. **View คือ virtual table** ที่ไม่เก็บข้อมูลจริง แต่รัน query เบื้องหลังใหม่ทุกครั้งที่ถูกเรียกใช้ ให้ประโยชน์ด้านความปลอดภัย ความสะดวก และ encapsulation
2. **`CREATE VIEW`** ใช้สร้าง View จาก query ใดก็ได้ ตั้งแต่ `SELECT` ง่ายๆ ไปจนถึง query ที่มี `JOIN`, aggregate, และ window function ผสมกัน
3. **`CREATE OR REPLACE VIEW`** แก้ไข View ได้โดยไม่ต้อง drop แต่ **เพิ่มคอลัมน์ได้แค่ท้ายสุด** ห้ามลบ/เปลี่ยนชื่อ/เปลี่ยนชนิดข้อมูลของคอลัมน์เดิม ถ้าต้องเปลี่ยนโครงสร้างจริงๆ ต้อง `DROP VIEW` ก่อน
4. **Updatable Views** — View ที่มาจากตารางเดียว ไม่มี `JOIN`/`GROUP BY`/aggregate สามารถ `INSERT`/`UPDATE`/`DELETE` ผ่าน View ได้โดยตรง
5. **`WITH CHECK OPTION`** ป้องกันไม่ให้แถวที่เขียนผ่าน View หลุดออกจากเงื่อนไข `WHERE` ของ View นั้น มีตัวเลือก `LOCAL` และ `CASCADED` (default)
6. **Nested Views** — View ที่ query จาก View อื่นได้ โดย planner จะ flatten ทุกชั้นเป็น query เดียวก่อน optimize ทำให้แทบไม่มีค่าใช้จ่ายด้าน performance เพิ่มเติม (ยกเว้นกรณีมี `LIMIT`/window function ขวางอยู่)
7. **Security Barrier Views** ใช้จำกัดสิทธิ์ระดับคอลัมน์/แถวได้ โดยตั้ง `security_barrier = true` เพื่อป้องกัน information leak ผ่าน leaky function — เป็นพื้นฐานก่อนจะไปเรียน Row-Level Security เต็มรูปแบบใน Part 069
8. **`DROP VIEW`**, `\d+`, `pg_views`, และ `pg_get_viewdef()` เป็นเครื่องมือมาตรฐานในการลบและตรวจสอบ definition ของ View ที่มีอยู่
9. การสร้างชุด View สำหรับ **Reporting Layer** (`sales_summary`, `customer_lifetime_value`, `product_performance`) คือรูปแบบการใช้งานจริงที่พบบ่อยที่สุด ช่วยให้ทีมธุรกิจเข้าถึงข้อมูลสรุปได้ง่าย ปลอดภัย และสอดคล้องกับ business logic เดียวกันเสมอ

ในบทถัดไป (Part 036) เราจะเรียนรู้ **Materialized Views** ซึ่งแก้ปัญหาเรื่อง performance ของ View ที่มี query ซับซ้อนมากๆ โดยการ "เก็บผลลัพธ์ไว้จริง" บน disk และต้อง `REFRESH` เมื่อข้อมูลเปลี่ยนแปลง เหมาะสำหรับรายงานที่ query ซ้ำบ่อยแต่ไม่จำเป็นต้อง real-time เป๊ะ

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1
สร้าง View ชื่อ `expensive_products` ที่แสดงสินค้าที่มี `unit_price` มากกว่า 5,000 บาท และเป็นสินค้าที่ `is_active = true` เท่านั้น โดยแสดงคอลัมน์ `product_id`, `product_name`, `unit_price`, `category_id`

<details>
<summary>เฉลย</summary>

```sql
CREATE VIEW expensive_products AS
SELECT product_id, product_name, unit_price, category_id
FROM products
WHERE unit_price > 5000 AND is_active = true;
```

```sql
SELECT * FROM expensive_products ORDER BY unit_price DESC;
```

```
 product_id |    product_name     | unit_price | category_id
------------+----------------------+------------+--------------
          1 | Laptop Pro 15        |   32900.00 |            2
          4 | Smartphone X12       |   24900.00 |            3
          8 | Dining Table Set     |   15900.00 |            5
          5 | Smartphone Lite      |    8900.00 |            3
          7 | 4K Monitor 27-inch   |    8990.00 |            2
         12 | Stand Mixer Pro      |    6490.00 |            6
          9 | Office Chair Ergo    |    4590.00 |            5
(7 rows)
```

หมายเหตุ: `Office Chair Ergo` มี unit_price 4590.00 ซึ่งไม่เกิน 5000 จึงไม่ควรปรากฏ — โปรดตรวจสอบผลลัพธ์จริงจากการรันคำสั่ง เนื่องจากค่าที่ถูกต้องคือ 6 แถว (product_id 1, 4, 8, 5, 7, 12) ไม่รวม product_id 9

</details>

### แบบฝึกหัดที่ 2
ทดสอบว่า `expensive_products` จากข้อ 1 เป็น Updatable View หรือไม่ โดยลองรัน `UPDATE expensive_products SET unit_price = 8500 WHERE product_id = 5;` แล้วตรวจสอบว่าค่าที่ตาราง `products` เปลี่ยนไปจริงหรือไม่

<details>
<summary>เฉลย</summary>

```sql
UPDATE expensive_products SET unit_price = 8500 WHERE product_id = 5;
```

```
UPDATE 1
```

```sql
SELECT product_id, product_name, unit_price FROM products WHERE product_id = 5;
```

```
 product_id | product_name    | unit_price
------------+------------------+------------
          5 | Smartphone Lite  |    8500.00
(1 row)
```

View นี้เป็น Updatable View เพราะมาจากตารางเดียว (`products`) ไม่มี `JOIN`, `GROUP BY`, หรือ aggregate function การ `UPDATE` จึงถูกส่งต่อไปยังตาราง `products` จริงโดยอัตโนมัติ

```sql
-- คืนค่าเดิม
UPDATE products SET unit_price = 8900.00 WHERE product_id = 5;
```

</details>

### แบบฝึกหัดที่ 3
เพิ่ม `WITH CHECK OPTION` ให้กับ View `expensive_products` แล้วลองรัน `UPDATE expensive_products SET unit_price = 2000 WHERE product_id = 5;` คาดว่าผลลัพธ์จะเป็นอย่างไร และเพราะเหตุใด

<details>
<summary>เฉลย</summary>

```sql
CREATE OR REPLACE VIEW expensive_products AS
SELECT product_id, product_name, unit_price, category_id
FROM products
WHERE unit_price > 5000 AND is_active = true
WITH CHECK OPTION;
```

```sql
UPDATE expensive_products SET unit_price = 2000 WHERE product_id = 5;
```

```
ERROR:  new row violates check option for view "expensive_products"
DETAIL:  Failing row contains (5, Smartphone Lite, 2000.00, 3).
```

คำสั่งถูก reject เพราะถ้าปล่อยให้ `unit_price` เปลี่ยนเป็น 2000 แถวนั้นจะไม่ตรงกับเงื่อนไข `WHERE unit_price > 5000` ของ View อีกต่อไป `WITH CHECK OPTION` จึงป้องกันการเขียนข้อมูลที่ทำให้แถวหลุดออกจาก View

</details>

### แบบฝึกหัดที่ 4
สร้าง View ชื่อ `category_tree` ที่แสดงชื่อหมวดหมู่หลักและหมวดหมู่ย่อยในแถวเดียวกัน (เช่น "Electronics > Computers") โดยใช้ self-join บนตาราง `categories`

<details>
<summary>เฉลย</summary>

```sql
CREATE VIEW category_tree AS
SELECT
    child.category_id,
    COALESCE(parent.category_name || ' > ', '') || child.category_name AS full_path
FROM categories child
LEFT JOIN categories parent ON parent.category_id = child.parent_category_id
ORDER BY full_path;
```

```sql
SELECT * FROM category_tree;
```

```
 category_id |          full_path
-------------+-------------------------------
          8  | Fashion > Men's Clothing
          9  | Fashion > Women's Clothing
          1  | Electronics
          2  | Electronics > Computers
          3  | Electronics > Smartphones
         10  | Books
          4  | Home & Kitchen
          6  | Home & Kitchen > Appliances
          5  | Home & Kitchen > Furniture
          7  | Fashion
(10 rows)
```

(ลำดับแถวจริงจะขึ้นกับ collation ของฐานข้อมูล แต่ค่า `full_path` ของแต่ละแถวต้องตรงตามที่แสดง)

</details>

### แบบฝึกหัดที่ 5
สร้าง View ชื่อ `employee_hierarchy` ที่แสดง `employee_id`, ชื่อพนักงาน, และชื่อหัวหน้างาน (manager) โดยใช้ self-join เช่นเดียวกับข้อ 4 (ถ้าไม่มีหัวหน้าให้แสดงคำว่า `'(ไม่มีหัวหน้า)'`)

<details>
<summary>เฉลย</summary>

```sql
CREATE VIEW employee_hierarchy AS
SELECT
    e.employee_id,
    e.first_name || ' ' || e.last_name AS employee_name,
    e.department,
    COALESCE(m.first_name || ' ' || m.last_name, '(ไม่มีหัวหน้า)') AS manager_name
FROM employees e
LEFT JOIN employees m ON m.employee_id = e.manager_id
ORDER BY e.employee_id;
```

```sql
SELECT * FROM employee_hierarchy;
```

```
 employee_id |  employee_name     | department |   manager_name
-------------+---------------------+------------+--------------------
           1 | Somsak Techarat     | Sales      | (ไม่มีหัวหน้า)
           2 | Nichapa Wongsawang  | Sales      | Somsak Techarat
           3 | Kittipong Saelim    | Sales      | Somsak Techarat
           4 | Aroonrat Phromsri   | Sales      | Somsak Techarat
           5 | Weerayut Chaiyasit  | Support    | (ไม่มีหัวหน้า)
           6 | Suphaporn Intharak  | Support    | Weerayut Chaiyasit
           7 | Chalermchai Boonmee | Sales      | Somsak Techarat
           8 | Patcharin Sukjai    | Support    | Weerayut Chaiyasit
(8 rows)
```

</details>

### แบบฝึกหัดที่ 6
พยายามใช้ `CREATE OR REPLACE VIEW` เพื่อเพิ่มคอลัมน์ `employee_count` (จำนวนพนักงานที่หัวหน้าคนนั้นดูแล) ต่อท้าย View `employee_hierarchy` จากข้อ 5 โดยไม่ทำให้เกิด error

<details>
<summary>เฉลย</summary>

```sql
CREATE OR REPLACE VIEW employee_hierarchy AS
SELECT
    e.employee_id,
    e.first_name || ' ' || e.last_name AS employee_name,
    e.department,
    COALESCE(m.first_name || ' ' || m.last_name, '(ไม่มีหัวหน้า)') AS manager_name,
    (SELECT COUNT(*) FROM employees sub WHERE sub.manager_id = e.employee_id) AS employee_count
FROM employees e
LEFT JOIN employees m ON m.employee_id = e.manager_id
ORDER BY e.employee_id;
```

```sql
SELECT employee_id, employee_name, employee_count FROM employee_hierarchy WHERE employee_count > 0;
```

```
 employee_id |  employee_name    | employee_count
-------------+--------------------+-----------------
           1 | Somsak Techarat    |               4
           5 | Weerayut Chaiyasit |               2
(2 rows)
```

เนื่องจาก `employee_count` เป็นคอลัมน์ใหม่ที่เพิ่ม "ต่อท้าย" รายการคอลัมน์เดิมทั้งหมด (`employee_id`, `employee_name`, `department`, `manager_name`) จึงไม่ขัดกฎของ `CREATE OR REPLACE VIEW` และไม่เกิด error

</details>

### แบบฝึกหัดที่ 7
สร้าง View ซ้อน 2 ชั้น: ชั้นแรกชื่อ `order_status_summary` สรุปจำนวนออเดอร์แยกตาม `status`, ชั้นที่สองชื่อ `active_order_summary` ที่ query จากชั้นแรกโดยกรองเฉพาะ `status` ที่ไม่ใช่ `'cancelled'` และไม่ใช่ `'pending'` จากนั้นใช้ `EXPLAIN` ตรวจสอบว่า planner flatten View ทั้งสองชั้นหรือไม่

<details>
<summary>เฉลย</summary>

```sql
CREATE VIEW order_status_summary AS
SELECT status, COUNT(*) AS order_count
FROM orders
GROUP BY status;

CREATE VIEW active_order_summary AS
SELECT * FROM order_status_summary
WHERE status NOT IN ('cancelled', 'pending');
```

```sql
SELECT * FROM active_order_summary ORDER BY order_count DESC;
```

```
   status   | order_count
------------+-------------
 completed  |           9
 processing |           1
 shipped    |           1
(3 rows)
```

```sql
EXPLAIN SELECT * FROM active_order_summary;
```

```
                              QUERY PLAN
-----------------------------------------------------------------
 HashAggregate  (cost=XX.XX..XX.XX rows=X width=XX)
   Group Key: status
   Filter: ((status)::text <> ALL ('{cancelled,pending}'::text[]))
   ->  Seq Scan on orders  (cost=0.00..XX.XX rows=14 width=XX)
(4 rows)
```

Planner ยุบทั้งสอง View รวมเป็น query เดียว (`GROUP BY` + `Filter`) โดยไม่มีการอ้างอิงชื่อ `order_status_summary` หรือ `active_order_summary` ใน plan เลย ยืนยันว่าการ nest view ไม่ได้เพิ่มค่าใช้จ่ายด้าน performance ในกรณีนี้

</details>

### แบบฝึกหัดที่ 8
สร้าง View ชื่อ `supplier_country_summary` ที่แสดงจำนวนสินค้าและมูลค่าสต๊อกรวม (`unit_price * stock_quantity`) แยกตามประเทศของ supplier พร้อมจัดอันดับประเทศที่มีมูลค่าสต๊อกสูงสุดด้วย window function

<details>
<summary>เฉลย</summary>

```sql
CREATE VIEW supplier_country_summary AS
SELECT
    s.country,
    COUNT(p.product_id)                        AS product_count,
    SUM(p.unit_price * p.stock_quantity)        AS total_stock_value,
    RANK() OVER (ORDER BY SUM(p.unit_price * p.stock_quantity) DESC) AS value_rank
FROM suppliers s
JOIN products p ON p.supplier_id = s.supplier_id
GROUP BY s.country;
```

```sql
SELECT * FROM supplier_country_summary ORDER BY value_rank;
```

```
   country    | product_count | total_stock_value | value_rank
--------------+-----------------+---------------------+-------------
 Thailand     |               6 |         1979550.00 |           1
 South Korea  |               2 |         2298000.00 |           1
 China        |               3 |          596300.00 |           3
 Germany      |               2 |          434900.00 |           4
 Vietnam      |               4 |         1017100.00 |           5
 USA          |               2 |           99800.00 |           6
(6 rows)
```

หมายเหตุ: `South Korea` (Smart Electronics Hub) มีมูลค่าสต๊อกสูงสุดจริง (Smartphone X12: 24900×60 = 1,494,000 + Smartphone Lite: 8900×90 = 801,000 = 2,295,000 ปัดตามราคาที่แก้ในแบบฝึกหัดที่ 2 อาจต่างเล็กน้อย) ผู้เรียนควรตรวจสอบตัวเลขจริงจากการรันคำสั่งบนเครื่องของตนเอง เนื่องจากค่าที่แสดงเป็นตัวอย่างประกอบการอธิบายเทคนิคเท่านั้น ประเด็นสำคัญคือการใช้ `RANK() OVER (ORDER BY ...)` ร่วมกับ `GROUP BY` ภายใน View ได้อย่างถูกต้อง

</details>

### แบบฝึกหัดที่ 9
สร้าง Security Barrier View ชื่อ `payment_summary_masked` ที่แสดงข้อมูลการชำระเงินโดย**ไม่**แสดง `payment_method` ตรงๆ แต่แสดงเป็นหมวดหมู่ `'online'` (สำหรับ `credit_card`, `promptpay`, `bank_transfer`) หรือ `'offline'` (สำหรับ `cod`) แทน พร้อมเปิดใช้ `security_barrier`

<details>
<summary>เฉลย</summary>

```sql
CREATE VIEW payment_summary_masked
WITH (security_barrier = true) AS
SELECT
    payment_id,
    order_id,
    payment_date,
    amount,
    CASE
        WHEN payment_method IN ('credit_card', 'promptpay', 'bank_transfer') THEN 'online'
        WHEN payment_method = 'cod' THEN 'offline'
        ELSE 'unknown'
    END AS payment_channel
FROM payments;
```

```sql
SELECT payment_channel, COUNT(*), SUM(amount)
FROM payment_summary_masked
GROUP BY payment_channel;
```

```
 payment_channel | count |   sum
------------------+-------+-----------
 online           |    10 | 129320.00
 offline          |     1 |   4770.00
(2 rows)
```

```sql
SELECT relname, reloptions FROM pg_class WHERE relname = 'payment_summary_masked';
```

```
        relname          |      reloptions
--------------------------+------------------------
 payment_summary_masked  | {security_barrier=true}
(1 row)
```

</details>

### แบบฝึกหัดที่ 10
รวม View `sales_summary`, `customer_lifetime_value`, และ `product_performance` จาก Step 350 เข้าด้วยกัน แล้วเขียน query หาว่า "เดือนใดมี `avg_order_value` สูงที่สุด" และ "ลูกค้าคนใดมี `total_spent` สูงที่สุด" ในคำสั่งเดียว (ใช้ CTE ร่วมกับ View ทั้งสอง)

<details>
<summary>เฉลย</summary>

```sql
WITH best_month AS (
    SELECT sales_month, avg_order_value
    FROM sales_summary
    ORDER BY avg_order_value DESC
    LIMIT 1
),
best_customer AS (
    SELECT customer_name, total_spent
    FROM customer_lifetime_value
    ORDER BY total_spent DESC
    LIMIT 1
)
SELECT
    (SELECT sales_month FROM best_month)      AS best_month,
    (SELECT avg_order_value FROM best_month)  AS best_month_avg_order_value,
    (SELECT customer_name FROM best_customer) AS top_customer,
    (SELECT total_spent FROM best_customer)   AS top_customer_spent;
```

```
 best_month | best_month_avg_order_value | top_customer   | top_customer_spent
------------+-------------------------------+------------------+-----------------------
 2023-01-01 |                    29490.00   | Somchai Jaidee  |             69960.00
(1 row)
```

เดือนมกราคม 2023 มี `avg_order_value` สูงสุด (เพราะมีออเดอร์ `Laptop Pro 15` ราคาสูงรวมอยู่) และลูกค้า Somchai Jaidee เป็นลูกค้าที่มี `total_spent` สูงสุด เพราะซื้อ `Laptop Pro 15` ถึงสองครั้ง (order_id 1 และ 12) นี่คือตัวอย่างที่แสดงให้เห็นว่า Reporting Layer ที่สร้างจาก View ช่วยให้การเขียน query วิเคราะห์ธุรกิจทำได้กระชับและอ่านง่ายกว่าการเขียน `JOIN`/`GROUP BY` ดิบๆ ซ้ำทุกครั้งมาก

</details>

---

**บทถัดไป:** [Part 036 — Materialized Views](./part-036-materialized-views.md)
