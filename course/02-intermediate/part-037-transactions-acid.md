# Transactions และหลักการ ACID

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับกลาง | Part 037

## เป้าหมายการเรียนรู้

เมื่อจบบทนี้ ผู้เรียนจะสามารถ:

- อธิบายได้ว่า transaction คืออะไร และทำไมระบบฐานข้อมูลเชิงสัมพันธ์ (relational database) ถึงต้องมีแนวคิดนี้
- ใช้คำสั่ง `BEGIN`, `COMMIT`, `ROLLBACK` เพื่อควบคุมกลุ่มคำสั่ง SQL ให้ทำงานเป็นหน่วยเดียว (unit of work)
- อธิบายหลักการ ACID (Atomicity, Consistency, Isolation, Durability) พร้อมยกตัวอย่างที่จับต้องได้จากฐานข้อมูล e-commerce
- เข้าใจว่า Atomicity ป้องกันปัญหาข้อมูล "ทำไปครึ่งทาง" ได้อย่างไร ผ่านตัวอย่างการหักสต๊อกสินค้า
- เข้าใจว่า Consistency ทำงานร่วมกับ constraint (CHECK, FOREIGN KEY, UNIQUE) อย่างไรภายใน transaction
- เข้าใจภาพรวมของ Isolation และเหตุผลที่ PostgreSQL มีหลายระดับ (isolation level) ก่อนเจาะลึกใน Part 038
- เข้าใจแนวคิด Durability และบทบาทของ WAL (Write-Ahead Log) แบบคร่าว ๆ ก่อนเจาะลึกใน Part 083
- ใช้ `SAVEPOINT` และ `ROLLBACK TO SAVEPOINT` เพื่อย้อนกลับเฉพาะบางส่วนของ transaction โดยไม่ต้องยกเลิกทั้งหมด
- แยกความแตกต่างระหว่าง implicit transaction (autocommit) กับ explicit transaction block
- จัดการกับ transaction ที่เข้าสถานะ aborted เมื่อเกิด error กลางทาง
- ออกแบบ transaction สำหรับกระบวนการทางธุรกิจจริง เช่น checkout process ที่ต้องตัดสต๊อก สร้างคำสั่งซื้อ และบันทึกการชำระเงินพร้อมกัน

---

## เตรียมข้อมูล

บทนี้ใช้ชุดข้อมูลร้านค้าออนไลน์ (e-commerce) เดียวกับ Part 021–039 ทั้งหมด หากคุณเคยสร้างฐานข้อมูลนี้ไว้แล้วจากบทก่อนหน้า สามารถข้ามไปยัง Step 361 ได้เลย แต่ถ้าต้องการฐานข้อมูลใหม่สำหรับฝึกฝนบทนี้โดยเฉพาะ ให้รันสคริปต์ด้านล่างทั้งหมด

```sql
-- ล้างตารางเดิม (ถ้ามี) เพื่อเริ่มต้นใหม่
DROP TABLE IF EXISTS product_stock_log CASCADE;
DROP TABLE IF EXISTS payments CASCADE;
DROP TABLE IF EXISTS reviews CASCADE;
DROP TABLE IF EXISTS order_items CASCADE;
DROP TABLE IF EXISTS orders CASCADE;
DROP TABLE IF EXISTS employees CASCADE;
DROP TABLE IF EXISTS customers CASCADE;
DROP TABLE IF EXISTS products CASCADE;
DROP TABLE IF EXISTS suppliers CASCADE;
DROP TABLE IF EXISTS categories CASCADE;

-- โครงสร้างตารางหลัก (schema กลางที่ใช้ตลอด Part 021-039)
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
    customer_id  SERIAL PRIMARY KEY,
    first_name   VARCHAR(60) NOT NULL,
    last_name    VARCHAR(60) NOT NULL,
    email        VARCHAR(150) UNIQUE,
    country      VARCHAR(60),
    signup_date  DATE NOT NULL DEFAULT CURRENT_DATE
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

-- ตารางเสริมเฉพาะบทนี้: ใช้บันทึกการเคลื่อนไหวของสต๊อกสินค้า (inventory movement log)
-- ตารางนี้ช่วยให้เราเห็นภาพ "หลายขั้นตอนที่ต้องสำเร็จไปด้วยกัน" ได้ชัดเจนขึ้น
-- เมื่อสต๊อกเปลี่ยนแปลง (ไม่ว่าจากคำสั่งซื้อ การยกเลิก หรือการปรับปรุงคลัง)
-- ระบบควรบันทึกลง product_stock_log พร้อมกันในธุรกรรมเดียวกับการ UPDATE stock_quantity เสมอ
CREATE TABLE product_stock_log (
    log_id      SERIAL PRIMARY KEY,
    product_id  INTEGER REFERENCES products(product_id),
    change_qty  INTEGER NOT NULL,          -- ค่าบวก = สต๊อกเพิ่ม, ค่าลบ = สต๊อกลด
    reason      VARCHAR(50) NOT NULL,      -- เช่น 'order_placed', 'order_cancelled', 'restock'
    order_id    INTEGER REFERENCES orders(order_id),
    logged_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

ต่อไปเป็นข้อมูลตัวอย่าง (seed data) กรอกด้วย `ID` ที่ระบุตายตัว เพื่อให้ตัวอย่างในบทเรียนอ้างอิงแถวข้อมูลได้ตรงกันทุกครั้งที่รันสคริปต์ใหม่

```sql
-- categories
INSERT INTO categories (category_id, category_name, parent_category_id) VALUES
(1,  'Electronics',        NULL),
(2,  'Computers',          1),
(3,  'Mobile Phones',      1),
(4,  'Home Appliances',    NULL),
(5,  'Kitchen',            4),
(6,  'Fashion',            NULL),
(7,  'Men''s Clothing',    6),
(8,  'Women''s Clothing',  6),
(9,  'Books',              NULL),
(10, 'Sports & Outdoors',  NULL);
SELECT setval('categories_category_id_seq', 10);

-- suppliers
INSERT INTO suppliers (supplier_id, supplier_name, country) VALUES
(1,  'TechSource Co., Ltd.',         'Thailand'),
(2,  'Global Gadgets Inc.',          'USA'),
(3,  'Shenzhen Electronics Ltd.',    'China'),
(4,  'Nordic Home Living',          'Sweden'),
(5,  'Bangkok Textile Group',       'Thailand'),
(6,  'Osaka Appliance Corp',        'Japan'),
(7,  'Seoul Digital Co.',           'South Korea'),
(8,  'EuroBooks Publishing',        'Germany'),
(9,  'Active Gear Manufacturing',   'Vietnam'),
(10, 'Premium Leather Works',       'Italy');
SELECT setval('suppliers_supplier_id_seq', 10);

-- products
INSERT INTO products (product_id, product_name, category_id, supplier_id, unit_price, stock_quantity, is_active) VALUES
(1,  'Laptop Pro 15"',              2, 1,  32900.00, 25,  true),
(2,  'Wireless Mouse',              2, 3,    590.00, 150, true),
(3,  'Mechanical Keyboard',         2, 3,   2490.00, 80,  true),
(4,  'Smartphone X12',              3, 7,  21900.00, 40,  true),
(5,  'Smartphone Lite',             3, 7,   8900.00, 60,  true),
(6,  'Bluetooth Earbuds',           3, 2,   1990.00, 200, true),
(7,  'Air Fryer 5L',                5, 6,   2590.00, 45,  true),
(8,  'Stand Mixer',                 5, 4,   6900.00, 20,  true),
(9,  'Coffee Maker',                5, 6,   1890.00, 55,  true),
(10, 'Men''s Cotton T-Shirt',       7, 5,    350.00, 300, true),
(11, 'Men''s Denim Jacket',         7, 5,   1590.00, 70,  true),
(12, 'Women''s Summer Dress',       8, 5,    990.00, 90,  true),
(13, 'Women''s Leather Handbag',    8, 10,  3900.00, 35,  true),
(14, 'PostgreSQL Internals Book',   9, 8,   1290.00, 40,  true),
(15, 'The Art of SQL',              9, 8,    890.00, 60,  true),
(16, 'Yoga Mat Premium',           10, 9,    690.00, 120, true),
(17, 'Running Shoes Pro',          10, 9,   2990.00, 65,  true),
(18, 'Camping Tent 4-Person',      10, 9,   4590.00, 15,  true);
SELECT setval('products_product_id_seq', 18);

-- customers
INSERT INTO customers (customer_id, first_name, last_name, email, country, signup_date) VALUES
(1,  'Somchai',   'Jaidee',      'somchai.j@example.com',   'Thailand',       '2024-11-02'),
(2,  'Nattapong', 'Suksawat',    'nattapong.s@example.com', 'Thailand',       '2024-11-15'),
(3,  'Ploy',      'Wongchai',    'ploy.w@example.com',      'Thailand',       '2024-12-01'),
(4,  'Napat',     'Chaiyaporn',  'napat.c@example.com',     'Thailand',       '2024-12-10'),
(5,  'Michael',   'Johnson',     'michael.j@example.com',   'USA',            '2024-10-20'),
(6,  'Emma',      'Wilson',      'emma.w@example.com',      'United Kingdom', '2024-10-25'),
(7,  'Yuki',      'Tanaka',      'yuki.t@example.com',      'Japan',          '2024-11-05'),
(8,  'Hana',      'Kim',         'hana.k@example.com',      'South Korea',    '2024-11-18'),
(9,  'Anna',      'Schmidt',     'anna.s@example.com',      'Germany',        '2024-12-05'),
(10, 'Kittisak',  'Boonmee',     'kittisak.b@example.com',  'Thailand',       '2025-01-03'),
(11, 'Suda',      'Meechai',     'suda.m@example.com',      'Thailand',       '2025-01-08'),
(12, 'David',     'Lee',         'david.l@example.com',     'Singapore',      '2025-01-10'),
(13, 'Wipa',      'Srisuk',      'wipa.s@example.com',      'Thailand',       '2025-01-20'),
(14, 'Thomas',    'Müller',      'thomas.m@example.com',    'Germany',        '2025-02-01');
SELECT setval('customers_customer_id_seq', 14);

-- employees (manager_id อ้างอิงตัวเอง จึงต้องแทรกตามลำดับสายบังคับบัญชา)
INSERT INTO employees (employee_id, first_name, last_name, hire_date, manager_id, department) VALUES
(1, 'Prasert',    'Tantiwong',   '2018-01-15', NULL, 'Executive'),
(2, 'Siriporn',   'Kanjana',     '2018-03-01', 1,    'Sales'),
(3, 'Anucha',     'Petchara',    '2019-02-10', 1,    'Warehouse'),
(4, 'Malee',      'Sombat',      '2019-06-20', 2,    'Sales'),
(5, 'Wichai',     'Ruangrit',    '2020-01-10', 2,    'Sales'),
(6, 'Kanya',      'Uthaiwan',    '2020-05-15', 3,    'Warehouse'),
(7, 'Thanawat',   'Chuenjai',    '2021-03-01', 3,    'Warehouse'),
(8, 'Orawan',     'Pipatpong',   '2021-08-11', 2,    'Customer Service'),
(9, 'Sasiwimol',  'Detsri',      '2022-04-01', 1,    'Finance');
SELECT setval('employees_employee_id_seq', 9);

-- orders
INSERT INTO orders (order_id, customer_id, employee_id, order_date, status, ship_country) VALUES
(1,  1,  2, '2025-01-05 10:15:00+07', 'delivered',  'Thailand'),
(2,  2,  4, '2025-01-08 14:30:00+07', 'delivered',  'Thailand'),
(3,  3,  2, '2025-01-10 09:00:00+07', 'delivered',  'Thailand'),
(4,  5,  5, '2025-01-12 16:45:00+07', 'delivered',  'USA'),
(5,  1,  4, '2025-01-15 11:20:00+07', 'shipped',    'Thailand'),
(6,  6,  2, '2025-01-18 13:10:00+07', 'delivered',  'United Kingdom'),
(7,  4,  5, '2025-01-20 08:50:00+07', 'cancelled',  'Thailand'),
(8,  7,  4, '2025-01-22 15:30:00+07', 'delivered',  'Japan'),
(9,  8,  2, '2025-01-25 10:05:00+07', 'shipped',    'South Korea'),
(10, 2,  5, '2025-01-28 12:40:00+07', 'processing', 'Thailand'),
(11, 9,  4, '2025-02-01 09:15:00+07', 'delivered',  'Germany'),
(12, 10, 2, '2025-02-03 14:00:00+07', 'delivered',  'Thailand'),
(13, 3,  5, '2025-02-05 16:20:00+07', 'pending',    'Thailand'),
(14, 11, 4, '2025-02-08 10:30:00+07', 'delivered',  'Thailand'),
(15, 12, 2, '2025-02-10 11:45:00+07', 'shipped',    'Singapore'),
(16, 13, 5, '2025-02-12 13:55:00+07', 'delivered',  'Thailand'),
(17, 14, 4, '2025-02-15 09:40:00+07', 'processing', 'Germany'),
(18, 1,  2, '2025-02-18 15:10:00+07', 'pending',    'Thailand');
SELECT setval('orders_order_id_seq', 18);

-- order_items
INSERT INTO order_items (order_item_id, order_id, product_id, quantity, unit_price) VALUES
(1,  1,  1,  1, 32900.00),
(2,  1,  2,  2,   590.00),
(3,  2,  4,  1, 21900.00),
(4,  3,  7,  1,  2590.00),
(5,  3,  9,  1,  1890.00),
(6,  4,  1,  1, 32900.00),
(7,  5,  10, 3,   350.00),
(8,  6,  13, 1,  3900.00),
(9,  7,  6,  2,  1990.00),
(10, 8,  14, 1,  1290.00),
(11, 8,  15, 1,   890.00),
(12, 9,  5,  1,  8900.00),
(13, 10, 16, 2,   690.00),
(14, 11, 17, 1,  2990.00),
(15, 12, 3,  1,  2490.00),
(16, 13, 8,  1,  6900.00),
(17, 14, 11, 1,  1590.00),
(18, 14, 12, 2,   990.00),
(19, 15, 18, 1,  4590.00),
(20, 16, 2,  3,   590.00),
(21, 17, 6,  1,  1990.00),
(22, 18, 4,  1, 21900.00),
(23, 18, 9,  1,  1890.00);
SELECT setval('order_items_order_item_id_seq', 23);

-- reviews
INSERT INTO reviews (review_id, product_id, customer_id, rating, review_text, review_date) VALUES
(1,  1,  1,  5, 'Laptop เร็วมาก คุ้มราคา',                          '2025-01-20'),
(2,  1,  4,  4, 'จอสวยแต่แบตอยู่ได้ไม่นานเท่าที่คิด',                  '2025-01-25'),
(3,  4,  2,  5, 'กล้องถ่ายรูปสวยมาก',                                '2025-01-15'),
(4,  6,  7,  3, 'เสียงดีแต่ใส่นานแล้วไม่ค่อยสบายหู',                   '2025-01-30'),
(5,  7,  3,  5, 'ทอดกรอบอร่อยเหมือนทอดน้ำมัน',                        '2025-01-18'),
(6,  10, 1,  4, 'ผ้านุ่มใส่สบาย',                                    '2025-01-22'),
(7,  13, 6,  5, 'หนังแท้คุณภาพดีมาก',                                '2025-01-28'),
(8,  14, 8,  5, 'อธิบายเรื่อง internals ได้ละเอียดมาก',               '2025-02-02'),
(9,  16, 10, 4, 'หนาพอดีไม่ลื่น',                                    '2025-02-05'),
(10, 17, 11, 5, 'วิ่งสบายเท้ามาก',                                   '2025-02-09'),
(11, 2,  13, 3, 'ใช้งานทั่วไปได้ดีแต่ปุ่มคลิกดังไปหน่อย',              '2025-02-11'),
(12, 9,  3,  4, 'ชงกาแฟได้รสชาติดี',                                 '2025-01-19');
SELECT setval('reviews_review_id_seq', 12);

-- payments (มีเฉพาะออเดอร์ที่ไม่ถูกยกเลิกและไม่ค้างชำระ)
INSERT INTO payments (payment_id, order_id, payment_date, amount, payment_method) VALUES
(1,  1,  '2025-01-05 10:20:00+07', 34080.00, 'credit_card'),
(2,  2,  '2025-01-08 14:35:00+07', 21900.00, 'promptpay'),
(3,  3,  '2025-01-10 09:05:00+07',  4480.00, 'bank_transfer'),
(4,  4,  '2025-01-12 16:50:00+07', 32900.00, 'credit_card'),
(5,  5,  '2025-01-15 11:25:00+07',  1050.00, 'promptpay'),
(6,  6,  '2025-01-18 13:15:00+07',  3900.00, 'credit_card'),
(7,  8,  '2025-01-22 15:35:00+07',  2180.00, 'cod'),
(8,  9,  '2025-01-25 10:10:00+07',  8900.00, 'credit_card'),
(9,  10, '2025-01-28 12:45:00+07',  1380.00, 'promptpay'),
(10, 11, '2025-02-01 09:20:00+07',  2990.00, 'bank_transfer'),
(11, 12, '2025-02-03 14:05:00+07',  2490.00, 'credit_card'),
(12, 14, '2025-02-08 10:35:00+07',  3570.00, 'promptpay'),
(13, 15, '2025-02-10 11:50:00+07',  4590.00, 'credit_card'),
(14, 16, '2025-02-12 14:00:00+07',  1770.00, 'bank_transfer'),
(15, 17, '2025-02-15 09:45:00+07',  1990.00, 'cod');
SELECT setval('payments_payment_id_seq', 15);

-- product_stock_log เริ่มต้นเป็นตารางว่าง เราจะเห็นข้อมูลถูกเติมเข้าไปในตัวอย่าง transaction ของบทนี้
```

ตรวจสอบว่าข้อมูลครบถ้วน:

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
------------+-----------+----------+-----------+-----------+--------+--------------+---------+----------
         10 |        10 |       18 |        14 |         9 |     18 |           23 |      12 |       15
(1 row)
```

---

## Step 361: Transaction คืออะไร

**Transaction** คือกลุ่มของคำสั่ง SQL หนึ่งชุดหรือมากกว่า ที่ระบบฐานข้อมูลปฏิบัติต่อมันเสมือนเป็น "งานเดียว" (single unit of work) หลักการสำคัญที่สุดคือ **all-or-nothing**: ไม่ว่ากลุ่มคำสั่งนั้นจะมีกี่คำสั่งก็ตาม ผลลัพธ์จะออกมาได้แค่สองแบบเท่านั้น

1. **ทุกคำสั่งสำเร็จทั้งหมด** และผลการเปลี่ยนแปลงถูกบันทึกถาวร (COMMIT)
2. **ไม่มีคำสั่งใดมีผลเลยแม้แต่คำสั่งเดียว** ราวกับว่าไม่เคยรันอะไรเลย (ROLLBACK)

สิ่งที่ **ไม่สามารถเกิดขึ้นได้** คือสถานะ "ทำไปครึ่งทาง" เช่น คำสั่งแรกสำเร็จแต่คำสั่งที่สองล้มเหลว แล้วปล่อยให้ข้อมูลอยู่ในสภาพที่ไม่สอดคล้องกัน (inconsistent)

### ทำไมต้องมี transaction

ลองนึกภาพกระบวนการ "checkout" ของร้านค้าออนไลน์ในฐานข้อมูลของเรา เมื่อลูกค้ากดสั่งซื้อหนึ่งครั้ง ระบบต้องทำงานหลายขั้นตอนพร้อมกัน:

1. ตรวจสอบและหักสต๊อกสินค้าในตาราง `products`
2. บันทึกการเคลื่อนไหวของสต๊อกลงตาราง `product_stock_log`
3. สร้างแถวใหม่ในตาราง `orders`
4. สร้างแถวใน `order_items` สำหรับสินค้าแต่ละรายการ
5. บันทึกการชำระเงินลงตาราง `payments`

ถ้าขั้นตอนที่ 1-2 สำเร็จ แต่ขั้นตอนที่ 3 ล้มเหลว (เช่น เซิร์ฟเวอร์แอปพลิเคชันแครชกลางทาง หรือ `employee_id` ที่ส่งมาไม่มีอยู่จริง) ผลลัพธ์คือ **สต๊อกถูกหักไปแล้ว แต่ไม่มีคำสั่งซื้อเกิดขึ้นจริง** สินค้าจะ "หายไปในอากาศ" โดยไม่มีใครซื้อ นี่คือปัญหาคลาสสิกที่ transaction ถูกออกแบบมาเพื่อป้องกัน

### หลักการ ACID โดยสรุป

PostgreSQL รับประกันคุณสมบัติ 4 ข้อของ transaction ที่เรียกย่อว่า **ACID**:

| ตัวอักษร | ชื่อเต็ม | ความหมายโดยย่อ |
|---|---|---|
| **A** | Atomicity | ทุกคำสั่งใน transaction สำเร็จหรือล้มเหลวไปด้วยกันทั้งหมด |
| **C** | Consistency | ข้อมูลจะย้ายจากสถานะที่ถูกต้องหนึ่ง ไปสู่อีกสถานะที่ถูกต้องเสมอ (ไม่ละเมิด constraint) |
| **I** | Isolation | transaction ที่รันพร้อมกันจะไม่เห็นผลกระทบที่ยังไม่ COMMIT ของกันและกัน (ขึ้นกับ isolation level) |
| **D** | Durability | เมื่อ COMMIT สำเร็จแล้ว ข้อมูลจะไม่สูญหายแม้ระบบล่มทันทีหลังจากนั้น |

บทนี้จะพาไปดูแต่ละหลักการทีละข้อ พร้อมตัวอย่างที่รันได้จริงบนฐานข้อมูล e-commerce ของเรา ส่วน Isolation และ Durability จะเกริ่นนำในบทนี้ก่อน แล้วไปเจาะลึกแบบเต็มรูปแบบใน Part 038 (Isolation Levels) และ Part 083 (WAL) ตามลำดับ

---

## Step 362: BEGIN, COMMIT, ROLLBACK — syntax พื้นฐาน

โดยค่าเริ่มต้น PostgreSQL ทำงานในโหมด **autocommit** นั่นคือทุกคำสั่ง SQL เดี่ยว ๆ ที่คุณรัน จะถูกห่อด้วย transaction ของตัวเองโดยอัตโนมัติ และ COMMIT ทันทีที่คำสั่งเสร็จ ถ้าต้องการรวมหลายคำสั่งเข้าด้วยกันเป็น transaction เดียว ต้องเปิด **explicit transaction block** ด้วยคำสั่ง `BEGIN`

### syntax พื้นฐาน

```sql
BEGIN;
    -- คำสั่ง SQL หนึ่งหรือหลายคำสั่ง
COMMIT;      -- ยืนยันการเปลี่ยนแปลงทั้งหมดอย่างถาวร
```

หรือถ้าต้องการยกเลิกทุกอย่างที่ทำไปใน transaction นั้น:

```sql
BEGIN;
    -- คำสั่ง SQL หนึ่งหรือหลายคำสั่ง
ROLLBACK;    -- ยกเลิกการเปลี่ยนแปลงทั้งหมด ราวกับไม่เคยเกิดขึ้น
```

> หมายเหตุ: `BEGIN` ใน PostgreSQL เทียบเท่ากับ `START TRANSACTION` ตามมาตรฐาน SQL ทั้งสองคำสั่งใช้แทนกันได้ทุกที่

### ตัวอย่าง: โอนสต๊อกสินค้าระหว่างคำสั่งซื้อ

สถานการณ์: คำสั่งซื้อ #7 (Bluetooth Earbuds จำนวน 2 ชิ้น) ถูกยกเลิกไปแล้ว (`status = 'cancelled'`) แต่ระบบเก่ายังไม่เคยคืนสต๊อกกลับเข้าคลัง ในขณะเดียวกันคำสั่งซื้อ #10 ที่กำลังอยู่ในสถานะ `processing` ต้องการ Bluetooth Earbuds เพิ่มอีก 2 ชิ้นเพื่อรวมกับของเดิม

เราจะเขียน transaction เดียวที่ทำสองอย่างพร้อมกัน: **คืนสต๊อกจากออเดอร์ที่ถูกยกเลิก แล้วหักสต๊อกให้ออเดอร์ใหม่** — ถ้าขั้นตอนใดขั้นตอนหนึ่งล้มเหลว (เช่น สต๊อกไม่พอหลังคืนแล้ว) การเปลี่ยนแปลงทั้งหมดต้องไม่เกิดขึ้นเลย

ตรวจสอบสต๊อกปัจจุบันของ Bluetooth Earbuds (`product_id = 6`) ก่อน:

```sql
SELECT product_id, product_name, stock_quantity FROM products WHERE product_id = 6;
```

```
 product_id |   product_name    | stock_quantity
------------+--------------------+----------------
          6 | Bluetooth Earbuds  |            200
(1 row)
```

เริ่ม transaction แล้วทำขั้นตอนที่ 1 (คืนสต๊อก 2 ชิ้นจากออเดอร์ #7 ที่ถูกยกเลิก พร้อมบันทึก log):

```sql
BEGIN;
UPDATE products
SET stock_quantity = stock_quantity + 2
WHERE product_id = 6;
INSERT INTO product_stock_log (product_id, change_qty, reason, order_id)
VALUES (6, 2, 'order_cancelled', 7);
```

```
BEGIN
UPDATE 1
INSERT 0 1
```

ขั้นตอนที่ 2: หักสต๊อก 2 ชิ้นให้ออเดอร์ #10 และบันทึก log

```sql
UPDATE products
SET stock_quantity = stock_quantity - 2
WHERE product_id = 6;
INSERT INTO product_stock_log (product_id, change_qty, reason, order_id)
VALUES (6, -2, 'order_placed', 10);
```

```
UPDATE 1
INSERT 0 1
```

ก่อน COMMIT เราตรวจสอบผลลัพธ์ระหว่างทาง (ยังอยู่ใน transaction เดียวกัน):

```sql
SELECT product_id, product_name, stock_quantity FROM products WHERE product_id = 6;
```

```
 product_id |   product_name    | stock_quantity
------------+--------------------+----------------
          6 | Bluetooth Earbuds  |            200
(1 row)
```

สต๊อกกลับมาเป็น 200 พอดี (คืน 2 แล้วหัก 2 หักล้างกัน) เมื่อพอใจกับผลลัพธ์แล้ว ยืนยันด้วย COMMIT:

```sql
COMMIT;
```

```
COMMIT
```

ตรวจสอบ log ที่บันทึกไว้:

```sql
SELECT * FROM product_stock_log ORDER BY log_id;
```

```
 log_id | product_id | change_qty |      reason      | order_id |          logged_at
--------+------------+------------+-------------------+----------+-------------------------------
      1 |          6 |          2 | order_cancelled   |        7 | 2025-09-25 10:30:01.123456+07
      2 |          6 |         -2 | order_placed      |       10 | 2025-09-25 10:30:01.456789+07
(2 rows)
```

### ตัวอย่าง: ROLLBACK เมื่อเปลี่ยนใจกลางทาง

สมมติว่าระหว่างทำ transaction ข้างต้น เราพบว่ากรอกจำนวนผิด (ตั้งใจจะคืน 3 ชิ้น ไม่ใช่ 2 ชิ้น) เราสามารถยกเลิกทั้งหมดด้วย `ROLLBACK` แล้วเริ่มใหม่:

```sql
BEGIN;
UPDATE products SET stock_quantity = stock_quantity + 2 WHERE product_id = 6;
-- อ๊ะ กรอกผิด! ตั้งใจจะคืน 3 ไม่ใช่ 2 ยกเลิกทั้งหมดแล้วเริ่มใหม่ดีกว่า
ROLLBACK;
```

```
BEGIN
UPDATE 1
ROLLBACK
```

```sql
SELECT product_id, product_name, stock_quantity FROM products WHERE product_id = 6;
```

```
 product_id |   product_name    | stock_quantity
------------+--------------------+----------------
          6 | Bluetooth Earbuds  |            200
(1 row)
```

สต๊อกยังคงเป็น 200 เหมือนเดิม เพราะ `UPDATE` ที่รันไปก่อน `ROLLBACK` ไม่เคยถูก COMMIT จึงไม่มีผลใด ๆ หลงเหลืออยู่เลย — นี่คือหัวใจของ Atomicity ที่เราจะเจาะลึกใน Step ถัดไป

---

## Step 363: Atomicity เชิงลึก

**Atomicity** (ความเป็นเอกภาพ) คือการรับประกันว่าคำสั่งทั้งหมดใน transaction จะถูกมองเป็น "หน่วยเดียวที่แบ่งแยกไม่ได้" (atomic = แบ่งแยกไม่ได้) ถ้าคำสั่งใดคำสั่งหนึ่งล้มเหลว ทุกคำสั่งก่อนหน้าใน transaction เดียวกันจะถูกย้อนกลับทั้งหมดโดยอัตโนมัติ

### สาธิตปัญหาที่เกิดขึ้นถ้า "ไม่ใช้" transaction

เพื่อให้เห็นภาพชัดเจน เราจะจำลองสถานการณ์ที่ไม่มี transaction ครอบ โดยรันแต่ละคำสั่งแยกกันในโหมด autocommit (ค่าเริ่มต้นของ psql) แล้วดูว่าเกิดอะไรขึ้นเมื่อคำสั่งกลางทางล้มเหลว

สถานการณ์: ลูกค้า #4 (Napat) ต้องการสั่งซื้อ Air Fryer 5L (`product_id = 7`) จำนวน 1 ชิ้น แอปพลิเคชันของเราเขียนโค้ดผิดพลาด โดยส่ง `employee_id = 999` ซึ่งไม่มีอยู่จริงในตาราง `employees`

ตรวจสอบสต๊อกก่อน:

```sql
SELECT product_id, product_name, stock_quantity FROM products WHERE product_id = 7;
```

```
 product_id |  product_name  | stock_quantity
------------+-----------------+----------------
          7 | Air Fryer 5L    |             45
(1 row)
```

**คำสั่งที่ 1 (autocommit, ไม่มี BEGIN):** หักสต๊อก — สำเร็จและ COMMIT ทันที

```sql
UPDATE products SET stock_quantity = stock_quantity - 1 WHERE product_id = 7;
SELECT product_id, product_name, stock_quantity FROM products WHERE product_id = 7;
```

```
UPDATE 1
 product_id |  product_name  | stock_quantity
------------+-----------------+----------------
          7 | Air Fryer 5L    |             44
(1 row)
```

**คำสั่งที่ 2 (autocommit, ไม่มี BEGIN):** สร้างคำสั่งซื้อใหม่ — ล้มเหลวเพราะ `employee_id = 999` ไม่มีจริง

```sql
INSERT INTO orders (customer_id, employee_id, status, ship_country)
VALUES (4, 999, 'pending', 'Thailand');
```

```
ERROR:  insert or update on table "orders" violates foreign key constraint "orders_employee_id_fkey"
DETAIL:  Key (employee_id)=(999) is not present in table "employees".
```

**ผลลัพธ์ที่เกิดขึ้น:** เพราะคำสั่งที่ 1 รันในโหมด autocommit และ COMMIT ไปแล้วก่อนคำสั่งที่ 2 จะล้มเหลว สต๊อกของ Air Fryer 5L จึงถูกหักไปเรียบร้อยแล้ว (จาก 45 เหลือ 44) **แต่ไม่มีคำสั่งซื้อเกิดขึ้นจริงเลย** สินค้าสูญหายไป 1 ชิ้นโดยไม่มีใบสั่งซื้อรองรับ — ข้อมูลอยู่ในสภาพไม่สอดคล้องกัน (data inconsistency) ซึ่งเป็นปัญหาที่ตรวจจับได้ยากมากในระบบจริง

แก้ไขข้อมูลให้กลับสู่สภาพเดิมก่อนไปตัวอย่างถัดไป:

```sql
UPDATE products SET stock_quantity = 45 WHERE product_id = 7;
```

```
UPDATE 1
```

### แก้ปัญหาด้วย explicit transaction

คราวนี้เราห่อทั้งสองคำสั่งด้วย `BEGIN` ... `COMMIT` แล้วจงใจทำผิดพลาดแบบเดียวกันอีกครั้ง เพื่อดูว่า Atomicity ปกป้องเราอย่างไร

```sql
BEGIN;
UPDATE products SET stock_quantity = stock_quantity - 1 WHERE product_id = 7;
INSERT INTO orders (customer_id, employee_id, status, ship_country)
VALUES (4, 999, 'pending', 'Thailand');
```

```
BEGIN
UPDATE 1
ERROR:  insert or update on table "orders" violates foreign key constraint "orders_employee_id_fkey"
DETAIL:  Key (employee_id)=(999) is not present in table "employees".
```

เมื่อเกิด error ขึ้นระหว่าง transaction block, PostgreSQL จะเปลี่ยนสถานะของ transaction ทั้งหมดเป็น **aborted** ทันที (รายละเอียดเรื่องนี้จะอธิบายเต็ม ๆ ใน Step 369) เราต้องสั่ง `ROLLBACK` เพื่อออกจากสถานะนี้:

```sql
ROLLBACK;
```

```
ROLLBACK
```

ตรวจสอบสต๊อกอีกครั้ง:

```sql
SELECT product_id, product_name, stock_quantity FROM products WHERE product_id = 7;
```

```
 product_id |  product_name  | stock_quantity
------------+-----------------+----------------
          7 | Air Fryer 5L    |             45
(1 row)
```

สต๊อกยังคงเป็น 45 เหมือนเดิมทุกประการ! เพราะการ `UPDATE` ที่ทำไปก่อนหน้า error ไม่เคยถูก COMMIT จึงถูกยกเลิกไปพร้อมกับทั้ง transaction โดยอัตโนมัติ นี่คือคุณค่าของ Atomicity — ไม่ว่าจะมีกี่คำสั่งใน transaction, ระบบจะไม่ปล่อยให้ข้อมูลอยู่ในสภาพ "ทำไปครึ่งทาง" เด็ดขาด

---

## Step 364: Consistency เชิงลึก

**Consistency** (ความสอดคล้อง) หมายถึงการรับประกันว่า transaction จะพาฐานข้อมูลจาก **สถานะที่ถูกต้องตาม constraint หนึ่ง ไปสู่อีกสถานะที่ถูกต้องตาม constraint** เสมอ ไม่ว่าจะเป็น `CHECK`, `FOREIGN KEY`, `UNIQUE`, `NOT NULL` หรือ `PRIMARY KEY` — PostgreSQL จะตรวจสอบ constraint ทุกตัวที่เกี่ยวข้องก่อนอนุญาตให้ COMMIT สำเร็จ ถ้ามี constraint ใดถูกละเมิด ทั้ง transaction จะเข้าสถานะ aborted และต้อง ROLLBACK

ข้อควรเข้าใจ: Consistency ในความหมายของ ACID **ไม่ได้** แปลว่าฐานข้อมูล "ถูกต้องตามตรรกะธุรกิจ" เสมอไป มันหมายถึงข้อมูลจะไม่ละเมิด constraint ที่เราประกาศไว้เท่านั้น ส่วนตรรกะทางธุรกิจที่ซับซ้อนกว่านั้น (เช่น "ยอดในตาราง payments ต้องเท่ากับผลรวมของ order_items") เป็นหน้าที่ของแอปพลิเคชันหรือ trigger ที่เราจะเรียนในบทต่อ ๆ ไป

ตารางในฐานข้อมูลของเรามี constraint สำคัญดังนี้:

```sql
SELECT conname, conrelid::regclass AS table_name, pg_get_constraintdef(oid) AS definition
FROM pg_constraint
WHERE contype = 'c'
ORDER BY conrelid::regclass::text;
```

```
              conname               |  table_name  |              definition
-------------------------------------+---------------+----------------------------------------
 order_items_quantity_check          | order_items   | CHECK ((quantity > 0))
 products_unit_price_check           | products      | CHECK ((unit_price >= (0)::numeric))
 reviews_rating_check                | reviews       | CHECK ((rating >= 1) AND (rating <= 5))
(3 rows)
```

### ตัวอย่าง: CHECK constraint ทำงานร่วมกับ transaction

ลองสร้างคำสั่งซื้อใหม่ที่มีสินค้าสองรายการ โดยรายการที่สองกรอกจำนวนผิดเป็นค่าลบ (บั๊กในแอปพลิเคชันที่คำนวณจำนวนคืนสินค้าแล้วลบผิด)

```sql
BEGIN;
INSERT INTO orders (customer_id, employee_id, status, ship_country)
VALUES (3, 2, 'pending', 'Thailand')
RETURNING order_id;
```

```
BEGIN
 order_id
----------
       19
(1 row)

INSERT 0 1
```

```sql
INSERT INTO order_items (order_id, product_id, quantity, unit_price)
VALUES (19, 15, 2, 890.00);

-- บั๊ก: quantity ติดลบเพราะคำนวณผิดพลาด
INSERT INTO order_items (order_id, product_id, quantity, unit_price)
VALUES (19, 16, -1, 690.00);
```

```
INSERT 0 1
ERROR:  new row for relation "order_items" violates check constraint "order_items_quantity_check"
DETAIL:  Failing row contains (24, 19, 16, -1, 690.00).
```

```sql
ROLLBACK;
SELECT * FROM orders WHERE order_id = 19;
```

```
ROLLBACK
 order_id | customer_id | employee_id | order_date | status | ship_country
----------+-------------+-------------+------------+--------+--------------
(0 rows)
```

สังเกตว่าแม้คำสั่ง `INSERT INTO orders` และ `INSERT INTO order_items` รายการแรกจะสำเร็จไปแล้ว (Postgres ตอบ `INSERT 0 1` ทั้งคู่) แต่เพราะ CHECK constraint ของรายการที่สองล้มเหลว ทำให้ transaction ทั้งหมดต้อง ROLLBACK คำสั่งซื้อ #19 จึงไม่ปรากฏอยู่ในระบบเลย — ฐานข้อมูลยังคง "สอดคล้อง" อยู่เสมอ ไม่มีคำสั่งซื้อที่ไม่มี order_items ที่ถูกต้องหลงเหลืออยู่

### ตัวอย่าง: FOREIGN KEY constraint ตรวจสอบตอน COMMIT (deferrable)

โดยปกติ FOREIGN KEY constraint ใน PostgreSQL จะถูกตรวจสอบทันทีหลังแต่ละคำสั่ง (`NOT DEFERRABLE` เป็นค่าเริ่มต้น) แต่สามารถประกาศให้เลื่อนการตรวจสอบไปจนกว่าจะ COMMIT ได้ด้วย `DEFERRABLE INITIALLY DEFERRED` ซึ่งมีประโยชน์เมื่อต้อง insert ข้อมูลที่อ้างอิงกันไปมาในลำดับที่ยังไม่ผ่าน constraint ระหว่างทาง เราจะกล่าวถึงเทคนิคนี้อย่างละเอียดใน Part 041 (Constraints ขั้นสูง) แต่ตัวอย่างสั้น ๆ ต่อไปนี้แสดงให้เห็นว่า constraint ทำงานร่วมกับขอบเขตของ transaction อย่างไร:

```sql
SELECT conname, condeferrable, condeferred
FROM pg_constraint
WHERE conrelid = 'order_items'::regclass AND contype = 'f';
```

```
        conname          | condeferrable | condeferred
--------------------------+---------------+-------------
 order_items_order_id_fkey    | f             | f
 order_items_product_id_fkey  | f             | f
(2 rows)
```

ในกรณีของเรา FOREIGN KEY เป็นแบบ `NOT DEFERRABLE` (ค่าเริ่มต้น) จึงถูกตรวจสอบทันทีหลังแต่ละคำสั่ง ไม่ใช่รอตอน COMMIT — นี่คือเหตุผลที่ตัวอย่างใน Step 363 เห็น error ทันทีที่รันคำสั่ง `INSERT` ที่ละเมิด FK ไม่ต้องรอถึง COMMIT

**ประเด็นสำคัญ:** ไม่ว่า constraint จะถูกตรวจสอบทันทีหรือเลื่อนไปตรวจตอน COMMIT ผลลัพธ์สุดท้ายเหมือนกันคือ **transaction จะไม่มีวันจบลงด้วยข้อมูลที่ละเมิด constraint** — ถ้าตรวจพบการละเมิดเมื่อใด ทั้ง transaction จะไม่ถูก COMMIT

---

## Step 365: Isolation เกริ่นนำ

**Isolation** (การแยกส่วน) คือการรับประกันว่า transaction หลายตัวที่รันพร้อมกัน (concurrent transactions) จะไม่เห็นผลกระทบที่ยังไม่ COMMIT ของกันและกัน — อย่างน้อยก็ในระดับหนึ่ง ขึ้นอยู่กับ **isolation level** ที่เลือกใช้

หัวข้อนี้เป็นเรื่องที่ซับซ้อนที่สุดในบรรดา ACID ทั้ง 4 ข้อ เพราะมาตรฐาน SQL กำหนด isolation level ไว้ถึง 4 ระดับ (Read Uncommitted, Read Committed, Repeatable Read, Serializable) แต่ละระดับแลกความถูกต้อง (correctness) กับประสิทธิภาพ (performance) ต่างกัน — เราจะเจาะลึกเรื่องนี้แบบเต็มรูปแบบใน **Part 038: Isolation Levels** ส่วนบทนี้จะแนะนำแนวคิดพื้นฐานเพื่อให้เข้าใจภาพรวมก่อน

### ทำไมต้องมีหลายระดับ

ลองจินตนาการสถานการณ์: พนักงานสองคนกำลังอัปเดตสต๊อกสินค้าตัวเดียวกันพร้อมกันในสอง transaction คนละ session

- ถ้า transaction ของคนแรกยังไม่ COMMIT แต่คนที่สองมองเห็นค่าที่ยังไม่ COMMIT ได้ — เรียกว่า **dirty read** ซึ่งอันตรายมาก เพราะถ้าคนแรก ROLLBACK ข้อมูลที่คนที่สองใช้ไปแล้วจะกลายเป็นข้อมูลที่ไม่เคยมีอยู่จริง
- ถ้า transaction คนที่สองอ่านค่าเดิมซ้ำสองครั้งในธุรกรรมเดียวกัน แล้วได้ค่าไม่ตรงกันเพราะมีคนอื่นมา COMMIT แทรกกลาง — เรียกว่า **non-repeatable read**
- ถ้าธุรกรรมหนึ่ง query ช่วงข้อมูลแล้วมีแถวใหม่โผล่ขึ้นมาตอน query ซ้ำ เพราะมีธุรกรรมอื่นแทรกข้อมูลเข้ามา — เรียกว่า **phantom read**

PostgreSQL ป้องกัน dirty read ได้ในทุก isolation level (ต่างจากฐานข้อมูลบางระบบ) แต่ปัญหาอื่น ๆ ขึ้นอยู่กับระดับที่เลือกใช้ ค่าเริ่มต้นของ PostgreSQL คือ **Read Committed**

### ตัวอย่างสาธิตสั้น ๆ ด้วยสอง session

เปิด `psql` สองหน้าต่างพร้อมกัน (เรียกว่า Session A และ Session B) เพื่อดูว่า Session B มองไม่เห็นการเปลี่ยนแปลงของ Session A จนกว่า Session A จะ COMMIT

**Session A:**

```sql
BEGIN;
UPDATE products SET stock_quantity = stock_quantity - 5 WHERE product_id = 2;
```

```
BEGIN
UPDATE 1
```

ยังไม่ COMMIT — ค้างไว้ตรงนี้ก่อน

**Session B (เปิดหน้าต่างใหม่ ในขณะที่ Session A ยังไม่ COMMIT):**

```sql
SELECT product_id, product_name, stock_quantity FROM products WHERE product_id = 2;
```

```
 product_id |  product_name  | stock_quantity
------------+-----------------+----------------
          2 | Wireless Mouse  |            150
(1 row)
```

Session B เห็นค่า `150` ซึ่งเป็นค่าก่อนที่ Session A จะแก้ไข — **ไม่เห็น** ค่าที่ยังไม่ COMMIT ของ Session A เลย นี่คือการป้องกัน dirty read ที่ PostgreSQL รับประกันในทุก isolation level

**กลับไปที่ Session A:**

```sql
COMMIT;
```

```
COMMIT
```

**กลับไปที่ Session B (query ใหม่หลังจาก Session A commit แล้ว):**

```sql
SELECT product_id, product_name, stock_quantity FROM products WHERE product_id = 2;
```

```
 product_id |  product_name  | stock_quantity
------------+-----------------+----------------
          2 | Wireless Mouse  |            145
(1 row)
```

ตอนนี้ Session B เห็นค่าใหม่ `145` แล้ว เพราะ Session A COMMIT สำเร็จไปแล้ว

ตรวจสอบ isolation level เริ่มต้นของ session ปัจจุบัน:

```sql
SHOW transaction_isolation;
```

```
 transaction_isolation
------------------------
 read committed
(1 row)
```

พฤติกรรมที่เราเพิ่งเห็นคือของระดับ **Read Committed** ซึ่งเป็นค่าเริ่มต้น — คำสั่ง `SELECT` แต่ละคำสั่งภายใน transaction จะเห็นข้อมูล ณ เวลาที่คำสั่งนั้นเริ่มทำงาน (ไม่ใช่ ณ เวลาที่ transaction เริ่มทำงาน) ซึ่งอาจนำไปสู่ปัญหา non-repeatable read ได้ในบางกรณี — รายละเอียดเชิงลึกของแต่ละ isolation level, phenomena ต่าง ๆ, และวิธีเลือกใช้ระดับที่เหมาะสมกับแต่ละ use case จะอยู่ใน **Part 038: Isolation Levels** ครบถ้วน

---

## Step 366: Durability

**Durability** (ความคงทน) คือการรับประกันว่า เมื่อ transaction ได้รับคำตอบ `COMMIT` กลับมาแล้ว **ข้อมูลจะไม่สูญหาย** ไม่ว่าจะเกิดอะไรขึ้นหลังจากนั้น แม้แต่ไฟดับกะทันหัน เครื่องเซิร์ฟเวอร์แครช หรือระบบปฏิบัติการค้าง

### PostgreSQL ทำได้อย่างไร: แนวคิดเรื่อง WAL

PostgreSQL ใช้กลไกที่เรียกว่า **Write-Ahead Log (WAL)** เป็นหัวใจสำคัญของ Durability หลักการคร่าว ๆ คือ:

1. ก่อนที่ PostgreSQL จะแก้ไขข้อมูลจริงในไฟล์ตาราง (heap file) มันจะเขียนบันทึกการเปลี่ยนแปลงนั้นลงใน **WAL** ก่อนเสมอ (จึงเรียกว่า "write-ahead")
2. บันทึก WAL จะถูก `fsync` (บังคับเขียนลงดิสก์จริง ไม่ใช่แค่ค้างอยู่ใน OS cache) ก่อนที่ PostgreSQL จะตอบว่า `COMMIT` สำเร็จ
3. ถ้าระบบล่มก่อนที่การเปลี่ยนแปลงจริงในไฟล์ตารางจะถูกเขียนลงดิสก์ครบถ้วน PostgreSQL จะใช้ WAL ในการ "เล่นซ้ำ" (replay) การเปลี่ยนแปลงเหล่านั้นตอนเริ่มระบบใหม่ (เรียกว่า crash recovery)

พูดง่าย ๆ คือ: **ตราบใดที่ WAL ถูกเขียนลงดิสก์แล้ว ข้อมูลปลอดภัย** แม้ตัวไฟล์ตารางจริงจะยังไม่ทันอัปเดตในดิสก์ก็ตาม เพราะ WAL สามารถนำมาเล่นซ้ำได้เสมอ

ตรวจสอบการตั้งค่าที่เกี่ยวข้องกับ WAL และ Durability ในระบบ:

```sql
SHOW wal_level;
SHOW synchronous_commit;
```

```
 wal_level
-----------
 replica
(1 row)

 synchronous_commit
---------------------
 on
(1 row)
```

`synchronous_commit = on` หมายความว่า คำสั่ง COMMIT จะรอจนกว่า WAL record ของ transaction นั้นถูกเขียนและ `fsync` ลงดิสก์เรียบร้อยก่อน แล้วจึงส่งคำตอบกลับไปยังไคลเอนต์ — นี่คือสิ่งที่ทำให้เรามั่นใจได้ว่า "COMMIT สำเร็จ = ข้อมูลปลอดภัยแล้วจริง ๆ"

### ทำไมเรื่องนี้สำคัญกับนักพัฒนา

ในทางปฏิบัติ นักพัฒนาแอปพลิเคชันไม่จำเป็นต้องเข้าใจกลไกภายในของ WAL แบบละเอียด แต่สิ่งสำคัญที่ต้องจำไว้คือ:

- **หลัง COMMIT สำเร็จ ให้เชื่อได้เลยว่าข้อมูลถูกบันทึกถาวรแล้ว** ไม่ต้องกังวลว่าจะหายไปเพราะไฟดับ
- ถ้าแอปพลิเคชันของคุณต้องการความเร็วสูงกว่าความปลอดภัยสัมบูรณ์ (เช่น ระบบ log ที่ยอมเสียข้อมูลไม่กี่วินาทีสุดท้ายได้) สามารถปรับ `synchronous_commit = off` เป็นรายเซสชันได้ แต่ต้องเข้าใจ trade-off ที่แลกมา
- WAL ยังเป็นรากฐานของฟีเจอร์สำคัญอื่น ๆ เช่น point-in-time recovery (PITR), streaming replication, และ logical replication

รายละเอียดเชิงลึกของ WAL — โครงสร้างไฟล์, checkpoint, WAL segment, การตั้งค่า `wal_level` แต่ละแบบ, และการทำ crash recovery จริง — จะอยู่ใน **Part 083: Write-Ahead Logging (WAL)** ซึ่งเป็นส่วนหนึ่งของเนื้อหาระดับ world-class expert ในหลักสูตรนี้ สำหรับตอนนี้ ขอให้จำไว้เพียงว่า: **Durability คือเหตุผลที่คุณไว้ใจ COMMIT ได้ในทุกสถานการณ์**

---

## Step 367: SAVEPOINT — จุดย้อนกลับบางส่วนภายใน transaction เดียว

บางครั้ง transaction ของเรามีหลายขั้นตอน และถ้าขั้นตอนใดขั้นตอนหนึ่งล้มเหลว เราอาจไม่อยากยกเลิกทั้ง transaction ทั้งหมด แต่อยากแค่ "ย้อนกลับบางส่วน" แล้วลองทำขั้นตอนนั้นใหม่ด้วยวิธีอื่น — นี่คือหน้าที่ของ **SAVEPOINT**

### syntax

```sql
BEGIN;
    -- คำสั่งชุดที่ 1
    SAVEPOINT sp1;
    -- คำสั่งชุดที่ 2 (อาจล้มเหลวได้)
    -- ถ้าล้มเหลว: ROLLBACK TO SAVEPOINT sp1;
    RELEASE SAVEPOINT sp1;   -- ยืนยันว่าไม่ต้องการ savepoint นี้แล้ว (ไม่บังคับ)
COMMIT;
```

`SAVEPOINT` สามารถสร้างซ้อนกันได้หลายจุดภายใน transaction เดียว และ `ROLLBACK TO SAVEPOINT` จะย้อนกลับเฉพาะคำสั่งที่รันหลังจากจุดนั้น โดยไม่กระทบคำสั่งก่อนหน้าที่รันไปแล้ว

### ตัวอย่าง: ประมวลผลคำสั่งซื้อหลายสินค้า โดยข้ามสินค้าที่สต๊อกไม่พอ

สถานการณ์: ลูกค้า #12 (David) ต้องการสั่งซื้อ 3 รายการพร้อมกันในออเดอร์เดียว ได้แก่ Camping Tent (เหลือ 15 ชิ้น สั่ง 1 ชิ้น — พอ), Stand Mixer (เหลือ 20 ชิ้น สั่ง 1 ชิ้น — พอ) และ Camping Tent อีกครั้งแบบสั่งเกินจำนวนที่มี (สั่ง 999 ชิ้น — ไม่พอแน่นอน เราจะจำลองด้วย CHECK constraint ที่เพิ่มขึ้นมาชั่วคราวเพื่อสาธิต)

ก่อนอื่น เพิ่ม CHECK constraint ชั่วคราวเพื่อไม่ให้สต๊อกติดลบ (เป็นตัวอย่างการใช้ constraint ป้องกัน overselling ซึ่งเราจะพูดถึงอย่างเป็นทางการใน Part 041):

```sql
ALTER TABLE products ADD CONSTRAINT products_stock_nonnegative CHECK (stock_quantity >= 0);
```

```
ALTER TABLE
```

ตอนนี้เริ่ม transaction สำหรับสร้างออเดอร์ใหม่:

```sql
BEGIN;
INSERT INTO orders (customer_id, employee_id, status, ship_country)
VALUES (12, 2, 'pending', 'Singapore')
RETURNING order_id;
```

```
BEGIN
 order_id
----------
       20
(1 row)

INSERT 0 1
```

**รายการที่ 1: Camping Tent จำนวน 1 ชิ้น**

```sql
SAVEPOINT item_1;
UPDATE products SET stock_quantity = stock_quantity - 1 WHERE product_id = 18;
INSERT INTO order_items (order_id, product_id, quantity, unit_price)
VALUES (20, 18, 1, 4590.00);
RELEASE SAVEPOINT item_1;
```

```
SAVEPOINT
UPDATE 1
INSERT 0 1
RELEASE
```

**รายการที่ 2: Stand Mixer จำนวน 1 ชิ้น**

```sql
SAVEPOINT item_2;
UPDATE products SET stock_quantity = stock_quantity - 1 WHERE product_id = 8;
INSERT INTO order_items (order_id, product_id, quantity, unit_price)
VALUES (20, 8, 1, 6900.00);
RELEASE SAVEPOINT item_2;
```

```
SAVEPOINT
UPDATE 1
INSERT 0 1
RELEASE
```

**รายการที่ 3: Camping Tent จำนวน 999 ชิ้น (เกินสต๊อกแน่นอน)**

```sql
SAVEPOINT item_3;
UPDATE products SET stock_quantity = stock_quantity - 999 WHERE product_id = 18;
```

```
SAVEPOINT
ERROR:  new row for relation "products" violates check constraint "products_stock_nonnegative"
DETAIL:  Failing row contains (18, Camping Tent 4-Person, 10, 9, 4590.00, -985, t).
```

เกิด error! ถ้าเราไม่มี SAVEPOINT ทั้ง transaction จะเข้าสถานะ aborted ทันที และต้อง ROLLBACK ทิ้งทุกอย่างรวมถึงรายการที่ 1 และ 2 ที่สำเร็จไปแล้ว แต่เพราะเรามี SAVEPOINT เราสามารถย้อนกลับเฉพาะรายการที่ 3 ได้:

```sql
ROLLBACK TO SAVEPOINT item_3;
SELECT order_item_id, order_id, product_id, quantity, unit_price
FROM order_items
WHERE order_id = 20;
```

```
ROLLBACK
 order_item_id | order_id | product_id | quantity | unit_price
----------------+----------+------------+----------+------------
             24 |       20 |         18 |        1 |    4590.00
             25 |       20 |          8 |        1 |    6900.00
(2 rows)
```

รายการที่ 1 (Camping Tent x1) และรายการที่ 2 (Stand Mixer x1) ยังอยู่ครบ แม้รายการที่ 3 จะถูกยกเลิกไปแล้ว! นี่คือพลังของ `SAVEPOINT` — เราสามารถ COMMIT สิ่งที่สำเร็จไปแล้ว โดยไม่ต้องเสียงานที่ทำไปทั้งหมด:

```sql
COMMIT;
```

```
COMMIT
```

ตรวจสอบผลลัพธ์สุดท้าย:

```sql
SELECT product_id, product_name, stock_quantity
FROM products
WHERE product_id IN (18, 8);
```

```
 product_id |     product_name       | stock_quantity
------------+--------------------------+----------------
          8 | Stand Mixer              |             19
         18 | Camping Tent 4-Person    |             14
(2 rows)
```

สต๊อกถูกหักถูกต้องเฉพาะรายการที่สำเร็จจริงเท่านั้น (Stand Mixer จาก 20 เหลือ 19, Camping Tent จาก 15 เหลือ 14) ส่วนความพยายามหักสต๊อก 999 ชิ้นถูกย้อนกลับไปโดยไม่กระทบส่วนอื่นเลย

### ข้อควรรู้เกี่ยวกับ SAVEPOINT

- `SAVEPOINT` ใช้ได้เฉพาะภายใน explicit transaction block เท่านั้น (ต้องมี `BEGIN` ก่อน)
- สามารถสร้าง SAVEPOINT ซ้อนกันได้หลายชั้น และ `ROLLBACK TO SAVEPOINT` จะย้อนกลับทุกอย่างที่อยู่ "หลัง" savepoint นั้น รวมถึง savepoint ย่อยที่ซ้อนอยู่ข้างในด้วย
- `RELEASE SAVEPOINT` ไม่ใช่การ COMMIT — มันเพียงแค่ลบ savepoint นั้นออกจากรายการ (savepoint ที่ยังไม่ถูก release จะถูกลบไปเองเมื่อ transaction จบลงไม่ว่าจะ COMMIT หรือ ROLLBACK)
- ถ้าตั้งชื่อ SAVEPOINT ซ้ำกัน savepoint เดิมจะถูกซ่อนไว้ (ไม่ error) และ `ROLLBACK TO SAVEPOINT` จะอ้างถึง savepoint ล่าสุดที่ใช้ชื่อนั้น
- ในไลบรารีเชื่อมต่อฐานข้อมูลหลายตัว (เช่น psycopg, SQLAlchemy) แนวคิด "nested transaction" มักถูกจำลองขึ้นด้วย SAVEPOINT เบื้องหลัง

ลบ constraint ชั่วคราวที่เพิ่มไว้สาธิต (เพื่อไม่ให้กระทบตัวอย่างในบทถัดไป):

```sql
ALTER TABLE products DROP CONSTRAINT products_stock_nonnegative;
```

```
ALTER TABLE
```

---

## Step 368: Implicit Transaction (Autocommit) เทียบกับ Explicit Transaction Block

เราได้เห็นตัวอย่างทั้งสองแบบมาแล้วในบทนี้ มาสรุปความแตกต่างให้ชัดเจนกัน

### Implicit transaction (autocommit)

เมื่อคุณรันคำสั่ง SQL เดี่ยว ๆ โดยไม่มี `BEGIN` ครอบ PostgreSQL จะสร้าง transaction ให้เองโดยอัตโนมัติสำหรับคำสั่งนั้นเพียงคำสั่งเดียว แล้ว COMMIT ทันทีถ้าสำเร็จ (หรือ ROLLBACK ทันทีถ้าล้มเหลว) — นี่คือค่าเริ่มต้นของ psql และไคลเอนต์ส่วนใหญ่

```sql
-- แต่ละคำสั่งด้านล่างคือ transaction ของตัวเอง เสร็จแล้ว commit ทันที
UPDATE products SET is_active = true WHERE product_id = 5;
UPDATE products SET is_active = true WHERE product_id = 6;
```

```
UPDATE 1
UPDATE 1
```

ทั้งสองคำสั่งข้างต้นเป็นคนละ transaction กันโดยสิ้นเชิง ถ้าคำสั่งที่สองล้มเหลว คำสั่งแรกจะยังคง COMMIT ไปแล้วเรียบร้อย ไม่ถูกยกเลิกตามไปด้วย

### Explicit transaction block

เมื่อคุณเปิดด้วย `BEGIN` ทุกคำสั่งที่ตามมาจะถูกจัดเป็น transaction เดียวกัน จนกว่าจะเจอ `COMMIT` หรือ `ROLLBACK`

```sql
BEGIN;
UPDATE products SET is_active = true WHERE product_id = 5;
UPDATE products SET is_active = true WHERE product_id = 6;
COMMIT;
```

```
BEGIN
UPDATE 1
UPDATE 1
COMMIT
```

ทั้งสองคำสั่งนี้ถูกมองเป็นหน่วยเดียว — ถ้าคำสั่งที่สองล้มเหลว คำสั่งแรกจะถูกยกเลิกไปด้วยเมื่อ ROLLBACK

### ตารางเปรียบเทียบ

| คุณสมบัติ | Implicit (Autocommit) | Explicit (BEGIN...COMMIT) |
|---|---|---|
| ขอบเขต transaction | หนึ่งคำสั่งต่อหนึ่ง transaction | หลายคำสั่งรวมเป็นหนึ่ง transaction |
| ต้องเขียน BEGIN | ไม่ต้อง | ต้อง |
| เหมาะกับ | คำสั่งเดี่ยวที่ไม่พึ่งพาคำสั่งอื่น | กระบวนการหลายขั้นตอนที่ต้องสำเร็จร่วมกัน |
| ผลเมื่อ error กลางทาง | เฉพาะคำสั่งนั้นล้มเหลว คำสั่งก่อนหน้ายังอยู่ | ทั้ง transaction เข้าสถานะ aborted ต้อง ROLLBACK ทั้งหมด |

### การตรวจสอบว่ากำลังอยู่ใน transaction หรือไม่

ใช้ฟังก์ชัน `pg_current_xact_id_if_assigned()` หรือดูค่าจาก `txid_current()` (จะ assign transaction ID ให้ทันทีที่เรียก) หรือวิธีที่ง่ายที่สุดคือดูพรอมต์ของ psql เอง — เมื่ออยู่ใน explicit transaction psql จะแสดงเป็น `mydb=*#` (มีเครื่องหมาย `*`) แทนที่จะเป็น `mydb=#` ปกติ

```sql
SELECT current_setting('transaction_isolation') AS iso_level;
```

```
    iso_level
------------------
 read committed
(1 row)
```

```sql
BEGIN;
```

```
BEGIN
```

ในบรรทัดพรอมต์ psql ตอนนี้จะเปลี่ยนจาก `postgres=#` เป็น `postgres=*#` เพื่อบอกว่าเรากำลังอยู่ในธุรกรรมที่ยังไม่ COMMIT

```sql
SELECT 1;
COMMIT;
```

```
 ?column?
----------
        1
(1 row)

COMMIT
```

พรอมต์กลับมาเป็น `postgres=#` ตามปกติ (ไม่มี `*`)

> **เกร็ดความรู้:** คำสั่ง Data Definition Language (DDL) เช่น `CREATE TABLE`, `ALTER TABLE`, `DROP TABLE` ใน PostgreSQL ก็อยู่ภายใต้ transaction ได้เช่นกัน! ซึ่งแตกต่างจากฐานข้อมูลอื่นบางระบบที่ DDL commit ทันทีโดยอัตโนมัติ ไม่สามารถ rollback ได้ ความสามารถนี้เรียกว่า **transactional DDL** และเป็นจุดเด่นสำคัญของ PostgreSQL — คุณสามารถ `BEGIN; CREATE TABLE ...; ALTER TABLE ...; ROLLBACK;` แล้วทุกอย่างจะถูกยกเลิกราวกับไม่เคยเกิดขึ้น เราจะใช้ประโยชน์จากคุณสมบัตินี้เมื่อพูดถึง migration scripts ใน Part ที่ว่าด้วยการจัดการ schema changes

---

## Step 369: Error Handling ภายใน Transaction — สถานะ Aborted

หนึ่งในพฤติกรรมที่ทำให้ผู้เริ่มต้นสับสนบ่อยที่สุดคือ: เมื่อเกิด error ขึ้นระหว่าง explicit transaction block, PostgreSQL จะไม่อนุญาตให้รันคำสั่งใด ๆ ต่อในธุรกรรมนั้นอีก **แม้แต่คำสั่งที่ถูกต้องสมบูรณ์แบบก็ตาม** จนกว่าจะสั่ง `ROLLBACK` (หรือ `ROLLBACK TO SAVEPOINT` ถ้ามี savepoint ก่อนหน้า error)

### สาธิตสถานะ aborted

```sql
BEGIN;
UPDATE customers SET country = 'Thailand' WHERE customer_id = 5;
-- พิมพ์ชื่อคอลัมน์ผิด (ไม่มี column ชื่อ customer_country ในตาราง)
SELECT customer_country FROM customers WHERE customer_id = 5;
```

```
BEGIN
UPDATE 1
ERROR:  column "customer_country" does not exist
LINE 1: SELECT customer_country FROM customers WHERE customer_id =...
               ^
HINT:  Perhaps you meant to reference the column "customers.country".
```

ทันทีที่เกิด error, transaction เข้าสู่สถานะ **aborted** ลองรันคำสั่งที่ถูกต้อง 100% ดู:

```sql
SELECT * FROM customers WHERE customer_id = 5;
```

```
ERROR:  current transaction is aborted, commands ignored until end of transaction block
```

แม้คำสั่งนี้จะไม่มีอะไรผิดเลย แต่ PostgreSQL ปฏิเสธที่จะรันมัน เพราะ transaction ทั้งก้อนถูก "แช่แข็ง" ไว้ในสถานะ error แล้ว วิธีเดียวที่จะหลุดออกจากสถานะนี้คือ:

```sql
ROLLBACK;
SELECT country FROM customers WHERE customer_id = 5;
```

```
ROLLBACK
 country
---------
 USA
(1 row)
```

สังเกตว่า `UPDATE customers SET country = 'Thailand'` ที่สำเร็จไปก่อนหน้า error ก็ถูกยกเลิกไปด้วยเช่นกัน (ยังคงเป็น `USA` เหมือนเดิม) เพราะ ROLLBACK ยกเลิกทั้ง transaction ไม่ใช่แค่คำสั่งที่ error

### ทำไม PostgreSQL ถึงออกแบบมาแบบนี้

เหตุผลเชิงวิศวกรรมคือ: เมื่อคำสั่งหนึ่ง error กลางทาง PostgreSQL ไม่มีทางรู้ได้แน่ชัดว่าแอปพลิเคชันฝั่งไคลเอนต์ "ตั้งใจ" ให้ transaction ดำเนินต่อไปโดยไม่สนใจ error นั้นหรือไม่ การบังคับให้ ROLLBACK ทั้งหมดเป็นการป้องกันไม่ให้เกิดข้อมูลที่ไม่สอดคล้องกันโดยไม่ได้ตั้งใจ — เป็นการยึดหลัก **fail-safe** (ล้มเหลวอย่างปลอดภัย) มากกว่าจะเดาใจผู้ใช้

### ใช้ SAVEPOINT เพื่อกันไม่ให้ทั้ง transaction ต้อง abort

ถ้าคุณคาดการณ์ล่วงหน้าว่าคำสั่งใดคำสั่งหนึ่งอาจล้มเหลว (เช่น ข้อมูลจากผู้ใช้ที่ไม่น่าเชื่อถือ) ควรวาง `SAVEPOINT` ไว้ก่อนคำสั่งนั้นเสมอ เพื่อให้สามารถกู้สถานะกลับมาได้โดยไม่เสียงานส่วนอื่นทั้งหมด — ดังที่แสดงไว้แล้วใน Step 367

```sql
BEGIN;
UPDATE customers SET country = 'Thailand' WHERE customer_id = 5;
SAVEPOINT before_risky_query;
SELECT customer_country FROM customers WHERE customer_id = 5;
```

```
BEGIN
UPDATE 1
SAVEPOINT
ERROR:  column "customer_country" does not exist
LINE 1: SELECT customer_country FROM customers WHERE customer_id =...
               ^
HINT:  Perhaps you meant to reference the column "customers.country".
```

```sql
ROLLBACK TO SAVEPOINT before_risky_query;
-- ตอนนี้ transaction กลับมาใช้งานได้ปกติ ไม่ต้อง ROLLBACK ทั้งก้อน
SELECT country FROM customers WHERE customer_id = 5;
COMMIT;
```

```
ROLLBACK
 country
---------
 Thailand
(1 row)

COMMIT
```

การ `UPDATE` ที่ทำไปก่อน SAVEPOINT ยังคงอยู่และถูก COMMIT สำเร็จ ส่วนคำสั่งที่ error หลัง SAVEPOINT ถูกย้อนกลับไปโดยไม่กระทบส่วนอื่น นี่คือเหตุผลที่ SAVEPOINT มีประโยชน์มากในการเขียนโปรแกรมที่ต้องจัดการ error อย่างละเอียด

### เกี่ยวกับ error handling ในฝั่งแอปพลิเคชันและ PL/pgSQL

ในโค้ดแอปพลิเคชัน (เช่น Python, Node.js, Java) ไลบรารีเชื่อมต่อฐานข้อมูลส่วนใหญ่จะจัดการเรื่องนี้ให้อัตโนมัติผ่าน try/except หรือ try/catch — เมื่อคำสั่งใดล้มเหลว ไลบรารีจะโยน exception กลับมาให้แอปพลิเคชันตัดสินใจว่าจะ ROLLBACK หรือจัดการต่ออย่างไร

ส่วนภายในฟังก์ชันหรือ stored procedure ที่เขียนด้วยภาษา PL/pgSQL นั้น สามารถใช้บล็อก `BEGIN ... EXCEPTION WHEN ... END` เพื่อดักจับ error และตัดสินใจได้ในระดับที่ละเอียดกว่านี้อีก โดยเบื้องหลังจะทำงานคล้ายกับการสร้าง SAVEPOINT อัตโนมัติรอบแต่ละ exception block — เนื้อหาส่วนนี้จะอธิบายอย่างละเอียดเมื่อเราเข้าสู่บทว่าด้วย PL/pgSQL และ Stored Procedures ในระดับถัดไปของหลักสูตร สำหรับตอนนี้ ขอให้จำหลักการสำคัญไว้ว่า: **เมื่อ error เกิดขึ้นกลางทาง transaction ต้อง ROLLBACK เท่านั้น (หรือ ROLLBACK TO SAVEPOINT ถ้าเตรียมไว้ล่วงหน้า) ไม่มีทางเลือกอื่น**

---

## Step 370: แบบฝึกหัดรวม — ออกแบบ Transaction สำหรับ Checkout Process จริง

ถึงเวลานำทุกอย่างที่เรียนมารวมกัน มาออกแบบ transaction สำหรับกระบวนการ **checkout** ที่สมบูรณ์แบบ ซึ่งประกอบด้วย 4 ขั้นตอนหลัก:

1. ตรวจสอบและหักสต๊อกสินค้าแต่ละรายการ พร้อมบันทึก stock log
2. สร้างคำสั่งซื้อใหม่ในตาราง `orders`
3. สร้างรายการสินค้าในตาราง `order_items`
4. บันทึกการชำระเงินในตาราง `payments`

ถ้าขั้นตอนใดขั้นตอนหนึ่งล้มเหลว (เช่น สต๊อกไม่พอ) ทั้งกระบวนการต้องไม่มีร่องรอยหลงเหลือในฐานข้อมูลเลย

### สถานการณ์ที่ 1: Checkout สำเร็จทั้งหมด

ลูกค้า #9 (Anna) ต้องการสั่งซื้อ The Art of SQL จำนวน 2 เล่ม และ Yoga Mat Premium จำนวน 1 ชิ้น ชำระเงินผ่าน `credit_card`

**ขั้นตอน 1: สร้างคำสั่งซื้อ**

```sql
BEGIN;
INSERT INTO orders (customer_id, employee_id, status, ship_country)
VALUES (9, 4, 'processing', 'Germany')
RETURNING order_id;
```

```
BEGIN
 order_id
----------
       21
(1 row)

INSERT 0 1
```

**ขั้นตอน 2: ตรวจสอบสต๊อกและหักสต๊อกทีละรายการ ด้วย SAVEPOINT ป้องกันความผิดพลาด**

รายการที่ 1: The Art of SQL (`product_id = 15`) จำนวน 2 เล่ม — เปิด savepoint, ตรวจสอบสต๊อกด้วย `DO` block ที่ `SELECT ... FOR UPDATE` (ล็อกแถวป้องกันคนอื่นแก้พร้อมกัน, รายละเอียดเต็มใน Part 038) แล้วจึงหักสต๊อกจริง:

```sql
SAVEPOINT stock_item_1;

DO $$
DECLARE
    v_current_stock INTEGER;
BEGIN
    SELECT stock_quantity INTO v_current_stock
    FROM products WHERE product_id = 15 FOR UPDATE;

    IF v_current_stock < 2 THEN
        RAISE EXCEPTION 'สต๊อกไม่เพียงพอสำหรับสินค้า product_id=15 (มี % ต้องการ 2)', v_current_stock;
    END IF;
END $$;

UPDATE products SET stock_quantity = stock_quantity - 2 WHERE product_id = 15;
INSERT INTO product_stock_log (product_id, change_qty, reason, order_id)
VALUES (15, -2, 'order_placed', 21);
INSERT INTO order_items (order_id, product_id, quantity, unit_price)
VALUES (21, 15, 2, 890.00);
RELEASE SAVEPOINT stock_item_1;
```

```
SAVEPOINT
DO
UPDATE 1
INSERT 0 1
INSERT 0 1
RELEASE
```

รายการที่ 2: Yoga Mat Premium (`product_id = 16`) จำนวน 1 ชิ้น ทำซ้ำแบบแผนเดียวกัน (SAVEPOINT → ตรวจสอบสต๊อกด้วย `DO` block → UPDATE → INSERT log → INSERT order_items → RELEASE):

```sql
SAVEPOINT stock_item_2;
DO $$
DECLARE
    v_current_stock INTEGER;
BEGIN
    SELECT stock_quantity INTO v_current_stock
    FROM products WHERE product_id = 16 FOR UPDATE;

    IF v_current_stock < 1 THEN
        RAISE EXCEPTION 'สต๊อกไม่เพียงพอสำหรับสินค้า product_id=16 (มี % ต้องการ 1)', v_current_stock;
    END IF;
END $$;
UPDATE products SET stock_quantity = stock_quantity - 1 WHERE product_id = 16;
INSERT INTO product_stock_log (product_id, change_qty, reason, order_id)
VALUES (16, -1, 'order_placed', 21);
INSERT INTO order_items (order_id, product_id, quantity, unit_price)
VALUES (21, 16, 1, 690.00);
RELEASE SAVEPOINT stock_item_2;
```

```
SAVEPOINT
DO
UPDATE 1
INSERT 0 1
INSERT 0 1
RELEASE
```

**ขั้นตอน 3: บันทึกการชำระเงิน** (ยอดรวม = 890.00 × 2 + 690.00 × 1 = 2470.00)

```sql
INSERT INTO payments (order_id, amount, payment_method)
VALUES (21, 2470.00, 'credit_card');
```

```
INSERT 0 1
```

**ขั้นตอน 4: อัปเดตสถานะคำสั่งซื้อเป็น confirmed (ในระบบจริงมักเปลี่ยนหลังชำระเงินสำเร็จ)**

```sql
UPDATE orders SET status = 'processing' WHERE order_id = 21;
```

```
UPDATE 1
```

ตรวจสอบภาพรวมก่อน COMMIT:

```sql
SELECT o.order_id, o.status, oi.product_id, p.product_name, oi.quantity, oi.unit_price,
       pay.amount, pay.payment_method
FROM orders o
JOIN order_items oi ON oi.order_id = o.order_id
JOIN products p ON p.product_id = oi.product_id
JOIN payments pay ON pay.order_id = o.order_id
WHERE o.order_id = 21;
```

```
 order_id |   status   | product_id |   product_name   | quantity | unit_price | amount  | payment_method
----------+-------------+------------+--------------------+----------+-------------+---------+-----------------
       21 | processing |         15 | The Art of SQL     |        2 |      890.00 | 2470.00 | credit_card
       21 | processing |         16 | Yoga Mat Premium   |        1 |      690.00 | 2470.00 | credit_card
(2 rows)
```

ทุกอย่างถูกต้อง — ยืนยันด้วย COMMIT:

```sql
COMMIT;
```

```
COMMIT
```

### สถานการณ์ที่ 2: Checkout ล้มเหลวเพราะสต๊อกไม่พอ

ลูกค้า #14 (Thomas) ต้องการสั่ง Camping Tent 4-Person จำนวน 100 ชิ้น (เกินสต๊อกที่มีอยู่แน่นอน หลังจากสถานการณ์ก่อนหน้านี้เหลืออยู่ 14 ชิ้น) เราจะเห็นว่าระบบปฏิเสธคำสั่งซื้อทั้งหมดโดยไม่มีร่องรอยตกค้าง

```sql
BEGIN;
INSERT INTO orders (customer_id, employee_id, status, ship_country)
VALUES (14, 4, 'processing', 'Germany')
RETURNING order_id;

SAVEPOINT stock_check;
DO $$
DECLARE
    v_current_stock INTEGER;
BEGIN
    SELECT stock_quantity INTO v_current_stock
    FROM products WHERE product_id = 18 FOR UPDATE;

    IF v_current_stock < 100 THEN
        RAISE EXCEPTION 'สต๊อกไม่เพียงพอสำหรับสินค้า product_id=18 (มี % ต้องการ 100)', v_current_stock;
    END IF;
END $$;
```

```
BEGIN
 order_id
----------
       22
(1 row)

INSERT 0 1
SAVEPOINT
ERROR:  สต๊อกไม่เพียงพอสำหรับสินค้า product_id=18 (มี 14 ต้องการ 100)
CONTEXT:  PL/pgSQL function inline_code_block line 6 at RAISE
```

Exception ที่ถูก `RAISE` ขึ้นมาจาก `DO` block ทำให้ transaction เข้าสถานะ aborted เหมือนกับ error ปกติ เนื่องจากเราไม่มีทางกู้สถานการณ์นี้ได้ (สต๊อกไม่พอจริง ๆ ไม่ใช่แค่ query ผิด) วิธีที่ถูกต้องคือยกเลิกคำสั่งซื้อทั้งหมด:

```sql
ROLLBACK;
```

```
ROLLBACK
```

ตรวจสอบว่าไม่มีคำสั่งซื้อ #22 หลงเหลืออยู่เลย:

```sql
SELECT * FROM orders WHERE order_id = 22;
SELECT product_id, product_name, stock_quantity FROM products WHERE product_id = 18;
```

```
 order_id | customer_id | employee_id | order_date | status | ship_country
----------+-------------+-------------+------------+--------+--------------
(0 rows)

 product_id |     product_name      | stock_quantity
------------+-------------------------+----------------
         18 | Camping Tent 4-Person   |             14
(1 row)
```

สต๊อกยังคงเป็น 14 ไม่ถูกกระทบเลย และไม่มีคำสั่งซื้อที่ไม่สมบูรณ์ตกค้างอยู่ในระบบ — นี่คือผลลัพธ์ที่ถูกต้องของการออกแบบ transaction สำหรับ checkout process: **ไม่ว่าจะสำเร็จหรือล้มเหลว ฐานข้อมูลจะไม่มีวันอยู่ในสภาพครึ่ง ๆ กลาง ๆ**

### แม่แบบ (template) สำหรับ checkout transaction ที่นำไปใช้จริงได้

สรุปเป็นแม่แบบ pseudo-transaction ที่ทีมพัฒนาสามารถนำไปปรับใช้ในโค้ดแอปพลิเคชันได้:

```sql
BEGIN;

-- 1. สร้างคำสั่งซื้อ (status เริ่มต้นเป็น pending หรือ processing)
INSERT INTO orders (customer_id, employee_id, status, ship_country)
VALUES (:customer_id, :employee_id, 'processing', :ship_country)
RETURNING order_id;   -- เก็บค่านี้ไว้ใช้ในขั้นตอนถัดไป

-- 2. สำหรับสินค้าแต่ละรายการในตะกร้า ทำซ้ำ:
--    a) SAVEPOINT ก่อนแตะต้องสต๊อก
--    b) ตรวจสอบสต๊อกด้วย SELECT ... FOR UPDATE (ป้องกัน race condition — รายละเอียดใน Part 038)
--    c) ถ้าไม่พอ: RAISE EXCEPTION แล้วปล่อยให้แอปพลิเคชันจับ error และ ROLLBACK ทั้งหมด
--    d) ถ้าพอ: UPDATE stock_quantity, INSERT product_stock_log, INSERT order_items
--    e) RELEASE SAVEPOINT

-- 3. คำนวณยอดรวมและบันทึกการชำระเงิน
INSERT INTO payments (order_id, amount, payment_method)
VALUES (:order_id, :total_amount, :payment_method);

-- 4. ถ้าทุกอย่างผ่าน
COMMIT;

-- ถ้าขั้นตอนใดล้มเหลว (จับได้จากฝั่งแอปพลิเคชัน หรือ error กลางทาง):
-- ROLLBACK;
```

หลักการสำคัญที่ควรจำจากแบบฝึกหัดนี้: **transaction ที่ดีสำหรับ business process ไม่ใช่แค่ห่อคำสั่งทั้งหมดด้วย BEGIN...COMMIT เท่านั้น** แต่ต้องคิดล่วงหน้าด้วยว่าจุดไหนอาจล้มเหลว (สต๊อกไม่พอ, constraint ถูกละเมิด, ข้อมูล input ผิดพลาด) แล้ววาง SAVEPOINT หรือการตรวจสอบเงื่อนไขไว้ล่วงหน้า เพื่อให้ระบบตอบสนองต่อความล้มเหลวได้อย่างสง่างาม (graceful failure) โดยไม่ทิ้งข้อมูลที่ไม่สอดคล้องกันไว้เบื้องหลัง

---

## สรุปท้ายบท

ในบทนี้เราได้เรียนรู้แนวคิดที่เป็นรากฐานสำคัญที่สุดอย่างหนึ่งของฐานข้อมูลเชิงสัมพันธ์ นั่นคือ **transaction** และหลักการ **ACID** ที่ PostgreSQL ใช้รับประกันความถูกต้องของข้อมูล:

- **Transaction** คือกลุ่มคำสั่ง SQL ที่ถูกมองเป็นหน่วยเดียว ทำงานแบบ all-or-nothing ใช้คำสั่ง `BEGIN`, `COMMIT`, `ROLLBACK` ในการควบคุม
- **Atomicity** รับประกันว่าไม่มีวันเกิดสถานะ "ทำไปครึ่งทาง" — เราเห็นตัวอย่างที่ชัดเจนจากการหักสต๊อกสินค้ากับการสร้างคำสั่งซื้อที่ต้องสำเร็จไปด้วยกัน
- **Consistency** รับประกันว่าข้อมูลจะไม่ละเมิด constraint ที่ประกาศไว้ ไม่ว่าจะเป็น CHECK, FOREIGN KEY หรือ UNIQUE เมื่อใดที่ constraint ถูกละเมิด ทั้ง transaction จะไม่ถูก COMMIT
- **Isolation** ควบคุมว่า transaction ที่รันพร้อมกันจะมองเห็นการเปลี่ยนแปลงของกันและกันแค่ไหน PostgreSQL ป้องกัน dirty read ได้เสมอ ส่วนรายละเอียดของแต่ละ isolation level จะอยู่ใน Part 038
- **Durability** รับประกันว่าเมื่อ COMMIT สำเร็จแล้ว ข้อมูลจะไม่สูญหาย โดยอาศัยกลไก Write-Ahead Log (WAL) ที่จะเจาะลึกใน Part 083
- **SAVEPOINT** ช่วยให้ย้อนกลับเฉพาะบางส่วนของ transaction ได้ โดยไม่ต้องเสียงานทั้งหมดที่ทำไปแล้ว เป็นเครื่องมือสำคัญสำหรับการจัดการ error อย่างละเอียด
- คำสั่งเดี่ยวที่รันโดยไม่มี `BEGIN` จะทำงานในโหมด **autocommit** (implicit transaction) ซึ่งต่างจาก **explicit transaction block** ที่รวมหลายคำสั่งเข้าด้วยกัน
- เมื่อเกิด error กลางทาง explicit transaction, ธุรกรรมจะเข้าสถานะ **aborted** และปฏิเสธคำสั่งทุกคำสั่งจนกว่าจะ `ROLLBACK` (หรือ `ROLLBACK TO SAVEPOINT` ถ้าเตรียมไว้)
- กระบวนการทางธุรกิจจริงอย่าง **checkout process** ต้องออกแบบ transaction ให้ครอบคลุมทุกขั้นตอนที่เกี่ยวข้องกัน พร้อมวางแผนรับมือกับความล้มเหลวที่อาจเกิดขึ้นล่วงหน้า

บทถัดไป **Part 038: Isolation Levels** จะพาไปเจาะลึกเรื่อง Isolation อย่างละเอียด — Read Uncommitted, Read Committed, Repeatable Read, Serializable, phenomena ต่าง ๆ อย่าง dirty read, non-repeatable read, phantom read, serialization anomaly รวมถึงการใช้ `SELECT ... FOR UPDATE` และการจัดการ race condition ในสถานการณ์ที่มีผู้ใช้จำนวนมากเข้าถึงข้อมูลเดียวกันพร้อมกัน

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1

จงอธิบายความหมายของคำว่า "all-or-nothing" ในบริบทของ transaction พร้อมยกตัวอย่างสถานการณ์จากฐานข้อมูล e-commerce ของเราที่ไม่ควรปล่อยให้เกิดผลลัพธ์แบบ "ทำไปครึ่งทาง"

<details>
<summary>เฉลย</summary>

"All-or-nothing" หมายถึงกลุ่มคำสั่งใน transaction จะมีผลลัพธ์ได้เพียงสองแบบเท่านั้น คือ **ทุกคำสั่งสำเร็จทั้งหมดและถูก COMMIT** หรือ **ไม่มีคำสั่งใดมีผลเลยเหมือนไม่เคยรัน** จะไม่มีสถานะกึ่งกลางที่คำสั่งบางส่วนสำเร็จแต่บางส่วนล้มเหลวหลงเหลืออยู่ในฐานข้อมูล

ตัวอย่างจากฐานข้อมูลของเรา: กระบวนการ checkout ที่ต้องหักสต๊อกสินค้าในตาราง `products` พร้อมกับสร้างแถวใหม่ในตาราง `orders` และ `order_items` ถ้าหักสต๊อกสำเร็จแต่สร้างคำสั่งซื้อไม่สำเร็จ (เช่น เพราะ `employee_id` ไม่ถูกต้อง) สินค้าจะถูกหักออกจากคลังไปแล้วโดยไม่มีคำสั่งซื้อรองรับ ทำให้ยอดสต๊อกในระบบไม่ตรงกับความเป็นจริง และไม่สามารถตรวจสอบย้อนหลังได้ว่าสินค้าที่หายไปนั้นถูกขายให้ใคร

</details>

### แบบฝึกหัดที่ 2

เขียน transaction ที่เพิ่มสินค้าใหม่ชื่อ `"Wireless Charger"` เข้าตาราง `products` (category_id = 2, supplier_id = 3, unit_price = 890.00, stock_quantity = 100, is_active = true) จากนั้นเพิ่ม log การเติมสต๊อกเริ่มต้นลงตาราง `product_stock_log` (reason = `'initial_stock'`) โดยต้องทำทั้งสองคำสั่งนี้ให้สำเร็จหรือล้มเหลวไปด้วยกัน

<details>
<summary>เฉลย</summary>

```sql
BEGIN;

INSERT INTO products (product_name, category_id, supplier_id, unit_price, stock_quantity, is_active)
VALUES ('Wireless Charger', 2, 3, 890.00, 100, true)
RETURNING product_id;
-- สมมติได้ product_id = 19

INSERT INTO product_stock_log (product_id, change_qty, reason, order_id)
VALUES (19, 100, 'initial_stock', NULL);

COMMIT;
```

การห่อทั้งสองคำสั่งด้วย `BEGIN...COMMIT` รับประกันว่าถ้าการ INSERT ลง `product_stock_log` ล้มเหลว (เช่น `product_id` อ้างอิงผิดพลาด) การเพิ่มสินค้าใหม่ในตาราง `products` ก็จะถูกยกเลิกไปด้วย ป้องกันไม่ให้เกิดสินค้าที่ไม่มี log การเติมสต๊อกเริ่มต้นรองรับ

</details>

### แบบฝึกหัดที่ 3

จากโค้ดต่อไปนี้ จงบอกว่าหลังรันเสร็จ ค่า `stock_quantity` ของ `product_id = 3` (Mechanical Keyboard, เริ่มต้น 80) จะเป็นเท่าใด

```sql
BEGIN;
UPDATE products SET stock_quantity = stock_quantity - 10 WHERE product_id = 3;
SAVEPOINT sp1;
UPDATE products SET stock_quantity = stock_quantity - 5 WHERE product_id = 3;
ROLLBACK TO SAVEPOINT sp1;
UPDATE products SET stock_quantity = stock_quantity - 2 WHERE product_id = 3;
COMMIT;
```

<details>
<summary>เฉลย</summary>

ค่าสุดท้ายคือ **68**

ลำดับการทำงาน:
1. เริ่มที่ 80
2. `UPDATE ... - 10` → 70 (ก่อน SAVEPOINT)
3. `SAVEPOINT sp1` บันทึกจุดนี้ไว้ (สต๊อก = 70)
4. `UPDATE ... - 5` → 65 (หลัง SAVEPOINT)
5. `ROLLBACK TO SAVEPOINT sp1` → ย้อนกลับเฉพาะขั้นตอนหลัง savepoint กลับไปเป็น 70
6. `UPDATE ... - 2` → 68
7. `COMMIT` → ยืนยันค่า 68 อย่างถาวร

การ `UPDATE ... - 10` ในขั้นตอนที่ 1 ไม่ถูกยกเลิก เพราะมันเกิดขึ้น**ก่อน** SAVEPOINT ส่วนการ `UPDATE ... - 5` ถูกยกเลิกเพราะเกิดขึ้น**หลัง** SAVEPOINT และมี ROLLBACK TO SAVEPOINT ตามมา

</details>

### แบบฝึกหัดที่ 4

พิจารณาโค้ดต่อไปนี้ เมื่อรันแล้วจะเกิดอะไรขึ้น และจะแก้ไขปัญหาอย่างไร

```sql
BEGIN;
UPDATE orders SET status = 'shipped' WHERE order_id = 5;
INSERT INTO reviews (product_id, customer_id, rating, review_text)
VALUES (1, 1, 7, 'สินค้าดีมาก');
SELECT * FROM orders WHERE order_id = 5;
```

<details>
<summary>เฉลย</summary>

คำสั่ง `INSERT INTO reviews` จะล้มเหลวทันที เพราะ `rating = 7` ละเมิด CHECK constraint `reviews_rating_check` (`CHECK (rating BETWEEN 1 AND 5)`) และ error นี้จะทำให้ transaction ทั้งหมดเข้าสู่สถานะ **aborted**

เมื่อรัน `SELECT * FROM orders WHERE order_id = 5;` ต่อไป จะได้ error:

```
ERROR:  current transaction is aborted, commands ignored until end of transaction block
```

แม้คำสั่ง SELECT นี้จะไม่มีข้อผิดพลาดใด ๆ ในตัวมันเองก็ตาม เพราะทั้ง transaction ถูกแช่แข็งไปแล้วตั้งแต่ error ก่อนหน้า

วิธีแก้ไข: ต้องรัน `ROLLBACK;` ก่อน เพื่อออกจากสถานะ aborted (ซึ่งจะทำให้ `UPDATE orders` ที่สำเร็จไปก่อนหน้าถูกยกเลิกไปด้วย) จากนั้นแก้ไข `rating` ให้อยู่ในช่วง 1-5 แล้วเริ่ม transaction ใหม่ หรือถ้าต้องการป้องกันปัญหานี้ล่วงหน้า ควรวาง `SAVEPOINT` ไว้ก่อนคำสั่ง INSERT ที่มีความเสี่ยง

</details>

### แบบฝึกหัดที่ 5

จงอธิบายว่าทำไมการรันคำสั่ง `UPDATE` สองคำสั่งแยกกันแบบ autocommit (ไม่มี `BEGIN`) ถึงมีความเสี่ยงมากกว่าการห่อด้วย `BEGIN...COMMIT` ในกรณีที่ทั้งสองคำสั่งต้องสอดคล้องกันทางตรรกะธุรกิจ ยกตัวอย่างจากตาราง `orders` และ `payments`

<details>
<summary>เฉลย</summary>

ในโหมด autocommit แต่ละคำสั่งจะถูก COMMIT ทันทีที่สำเร็จ โดยไม่รอคำสั่งถัดไป ถ้าเรารัน:

```sql
UPDATE orders SET status = 'delivered' WHERE order_id = 9;
-- (สมมติเซิร์ฟเวอร์ล่มตรงนี้พอดี ก่อนรันคำสั่งถัดไป)
INSERT INTO payments (order_id, amount, payment_method) VALUES (9, 8900.00, 'credit_card');
```

ถ้าเซิร์ฟเวอร์แอปพลิเคชันล่มหรือเกิด error ระหว่างสองคำสั่งนี้ (เช่น การเชื่อมต่อฐานข้อมูลหลุด) คำสั่งแรกจะถูก COMMIT ไปแล้วเรียบร้อย ทำให้ order #9 ถูกทำเครื่องหมายว่า `delivered` **ทั้งที่ยังไม่มีการบันทึกการชำระเงินเลย** ซึ่งขัดกับตรรกะธุรกิจที่ควรจะบันทึกการชำระเงินก่อนหรือพร้อมกับการยืนยันสถานะ

ถ้าห่อด้วย `BEGIN...COMMIT` แทน หากเกิดปัญหาระหว่างสองคำสั่ง (ก่อนถึง COMMIT) ทั้งสอง statement จะไม่มีผลใด ๆ เลย เพราะยังไม่เคย COMMIT — เมื่อระบบกลับมาทำงานใหม่ สามารถเริ่ม transaction นี้ใหม่ได้อย่างปลอดภัยโดยไม่ต้องกังวลว่าข้อมูลจะไม่สอดคล้องกัน

</details>

### แบบฝึกหัดที่ 6

เปิด `psql` สอง session (A และ B) แล้วทำตามขั้นตอนต่อไปนี้ พร้อมทำนายผลลัพธ์ที่ Session B จะเห็นในแต่ละจุด

Session A: `BEGIN;` แล้ว `UPDATE products SET unit_price = 999.99 WHERE product_id = 2;` (ยังไม่ COMMIT)

Session B: `SELECT unit_price FROM products WHERE product_id = 2;`

จากนั้น Session A: `ROLLBACK;`

Session B: `SELECT unit_price FROM products WHERE product_id = 2;` อีกครั้ง

<details>
<summary>เฉลย</summary>

ในขั้นตอนแรก Session B จะเห็นค่า `unit_price` **เดิม** (590.00) ไม่ใช่ 999.99 เพราะ PostgreSQL ป้องกัน dirty read ในทุก isolation level — session อื่นจะไม่มีวันเห็นข้อมูลที่ยังไม่ถูก COMMIT

หลังจาก Session A สั่ง `ROLLBACK` การเปลี่ยนแปลงเป็น 999.99 จะไม่มีผลใด ๆ เลย ดังนั้นเมื่อ Session B query ซ้ำอีกครั้ง ก็ยังคงเห็นค่าเดิม 590.00 เหมือนเดิมทั้งสองครั้ง

ข้อสังเกตสำคัญ: ในกรณีนี้ Session B เห็นค่า 590.00 เหมือนกันทั้งสองครั้ง แต่นั่นเป็นเพราะ Session A ไม่เคย COMMIT เลย ถ้า Session A สั่ง COMMIT แทน ROLLBACK ค่าที่ Session B เห็นในการ query ครั้งที่สองจะเปลี่ยนเป็น 999.99 ทันที ซึ่งเป็นตัวอย่างของพฤติกรรม Read Committed ที่จะอธิบายอย่างละเอียดใน Part 038

</details>

### แบบฝึกหัดที่ 7

จงเขียน transaction ที่จำลองการยกเลิกคำสั่งซื้อ #10 (`status = 'processing'`) อย่างสมบูรณ์ กล่าวคือ ต้อง: (1) เปลี่ยน status เป็น `'cancelled'`, (2) คืนสต๊อกสินค้าทุกรายการที่อยู่ใน order_items ของออเดอร์นี้กลับเข้าคลัง และ (3) บันทึก log การคืนสต๊อกแต่ละรายการ ทั้งหมดต้องอยู่ใน transaction เดียวกัน

<details>
<summary>เฉลย</summary>

```sql
BEGIN;

-- 1. เปลี่ยนสถานะคำสั่งซื้อ
UPDATE orders SET status = 'cancelled' WHERE order_id = 10;

-- 2. คืนสต๊อกทุกรายการของออเดอร์นี้ (ใช้ join กับ order_items)
UPDATE products p
SET stock_quantity = p.stock_quantity + oi.quantity
FROM order_items oi
WHERE oi.order_id = 10
  AND oi.product_id = p.product_id;

-- 3. บันทึก log การคืนสต๊อกแต่ละรายการ
INSERT INTO product_stock_log (product_id, change_qty, reason, order_id)
SELECT oi.product_id, oi.quantity, 'order_cancelled', 10
FROM order_items oi
WHERE oi.order_id = 10;

COMMIT;
```

จุดสำคัญ: การใช้ `UPDATE ... FROM` ช่วยให้อัปเดตสต๊อกของสินค้าหลายรายการในคำสั่งเดียวได้ ไม่ต้องวนลูปทีละแถว และเพราะทุกคำสั่งอยู่ใน transaction เดียวกัน ถ้าขั้นตอนใดล้มเหลว (เช่น `order_id = 10` ไม่มีอยู่จริง) ทุกอย่างจะถูกยกเลิกร่วมกันโดยอัตโนมัติ ไม่ทิ้งสถานะ order เป็น cancelled ทั้งที่สต๊อกยังไม่ถูกคืน

</details>

### แบบฝึกหัดที่ 8

`SAVEPOINT` กับ `ROLLBACK` (แบบเต็ม ไม่ระบุ savepoint) แตกต่างกันอย่างไร จงอธิบายพร้อมยกตัวอย่างสถานการณ์ที่ควรเลือกใช้แต่ละแบบ

<details>
<summary>เฉลย</summary>

`ROLLBACK` (แบบเต็ม) จะยกเลิกการเปลี่ยนแปลง**ทั้งหมด**ใน transaction และปิด transaction block นั้นทันที ต้องเปิด `BEGIN` ใหม่ถ้าต้องการทำงานต่อ

`ROLLBACK TO SAVEPOINT <ชื่อ>` จะยกเลิกเฉพาะคำสั่งที่รันหลังจากจุด savepoint นั้นเท่านั้น โดย transaction ยังคงเปิดอยู่ และคำสั่งที่รันก่อนหน้า savepoint ยังคงมีผลอยู่ สามารถรันคำสั่งใหม่ต่อไปได้ภายใน transaction เดิม

**ควรใช้ `ROLLBACK` แบบเต็ม** เมื่อพบว่างานทั้งหมดที่ทำมาผิดพลาดตั้งแต่ต้น หรือเมื่อไม่มีความจำเป็นต้องรักษาคำสั่งก่อนหน้าไว้เลย เช่น สถานการณ์ในแบบฝึกหัดที่ 4 ที่เราตัดสินใจยกเลิกออเดอร์ทั้งใบไปเลย

**ควรใช้ `SAVEPOINT` + `ROLLBACK TO SAVEPOINT`** เมื่อ transaction มีหลายขั้นตอนที่เป็นอิสระต่อกันในระดับหนึ่ง และต้องการรักษาผลลัพธ์ของขั้นตอนที่สำเร็จไปแล้วไว้ เช่น สถานการณ์ในการประมวลผลคำสั่งซื้อหลายรายการใน Step 367 ที่รายการซึ่งสต๊อกไม่พอถูกข้ามไป แต่รายการอื่นที่สำเร็จยังคง COMMIT ได้ตามปกติ

</details>

### แบบฝึกหัดที่ 9

จงอธิบายว่าทำไมการที่ COMMIT "รอ" ให้ WAL ถูกเขียนลงดิสก์ก่อนตอบกลับ (`synchronous_commit = on`) ถึงเป็นสิ่งจำเป็นสำหรับ Durability และจะเกิดอะไรขึ้นถ้าปิดการตั้งค่านี้

<details>
<summary>เฉลย</summary>

Durability รับประกันว่าเมื่อไคลเอนต์ได้รับคำตอบ `COMMIT` กลับมาแล้ว ข้อมูลจะไม่สูญหายแม้ระบบจะล่มทันทีหลังจากนั้น การรับประกันนี้ทำได้ก็ต่อเมื่อ **ข้อมูลถูกเขียนลงสื่อบันทึกถาวร (ดิสก์) จริง ๆ ก่อนที่จะตอบว่าสำเร็จ** — ถ้า COMMIT ตอบกลับไปแล้วแต่ข้อมูลยังค้างอยู่ใน memory หรือ OS cache เท่านั้น เมื่อไฟดับหรือเครื่องแครชกะทันหัน ข้อมูลที่ "COMMIT สำเร็จ" นั้นอาจหายไปได้จริง ซึ่งขัดกับคำสัญญาของ Durability

การตั้งค่า `synchronous_commit = on` (ค่าเริ่มต้น) ทำให้ PostgreSQL รอจนกว่า WAL record จะถูก `fsync` ลงดิสก์จริงก่อนตอบ COMMIT กลับไป จึงมั่นใจได้ว่าถึงแม้ระบบล่มทันทีหลังจากนั้น ข้อมูลก็ยังกู้คืนได้จาก WAL

ถ้าตั้ง `synchronous_commit = off` การตอบกลับ COMMIT จะเร็วขึ้น (เพราะไม่ต้องรอ fsync) แต่แลกมาด้วยความเสี่ยงที่ transaction ไม่กี่รายการล่าสุดอาจสูญหายได้หากระบบล่มในช่วงเวลาสั้น ๆ ก่อนที่ WAL จะถูกเขียนลงดิสก์จริง แม้ไคลเอนต์จะได้รับคำตอบว่า COMMIT สำเร็จไปแล้วก็ตาม — เหมาะสำหรับงานที่ยอมรับความเสี่ยงนี้ได้เพื่อแลกกับประสิทธิภาพที่สูงขึ้น (รายละเอียดเชิงลึกอยู่ใน Part 083)

</details>

### แบบฝึกหัดที่ 10

ออกแบบ transaction สำหรับสถานการณ์ต่อไปนี้: ลูกค้า #6 (Emma) ต้องการเปลี่ยนสินค้าในคำสั่งซื้อที่ยังไม่ชำระเงิน จาก Women's Leather Handbag (`product_id = 13`, ที่มีอยู่ใน order #6 อยู่แล้วจำนวน 1 ชิ้น) เป็น Women's Summer Dress (`product_id = 12`) จำนวน 2 ชิ้นแทน โดยต้อง: (1) คืนสต๊อก handbag กลับเข้าคลัง 1 ชิ้น, (2) หักสต๊อก dress ออก 2 ชิ้น, (3) ลบแถวเดิมใน order_items แล้วสร้างแถวใหม่แทน, (4) บันทึก log การเปลี่ยนแปลงสต๊อกทั้งสองรายการ — ถ้าสต๊อก dress ไม่พอ ต้องไม่มีการเปลี่ยนแปลงใด ๆ เกิดขึ้นเลย

<details>
<summary>เฉลย</summary>

```sql
BEGIN;

-- 1. คืนสต๊อก handbag กลับเข้าคลัง
UPDATE products SET stock_quantity = stock_quantity + 1 WHERE product_id = 13;

INSERT INTO product_stock_log (product_id, change_qty, reason, order_id)
VALUES (13, 1, 'item_swapped_out', 6);

-- 2. ตรวจสอบและหักสต๊อก dress ด้วย SAVEPOINT ป้องกันปัญหาสต๊อกไม่พอ
SAVEPOINT swap_in;

DO $$
DECLARE
    v_current_stock INTEGER;
BEGIN
    SELECT stock_quantity INTO v_current_stock
    FROM products WHERE product_id = 12 FOR UPDATE;

    IF v_current_stock < 2 THEN
        RAISE EXCEPTION 'สต๊อกไม่เพียงพอสำหรับสินค้า product_id=12 (มี % ต้องการ 2)', v_current_stock;
    END IF;
END $$;

UPDATE products SET stock_quantity = stock_quantity - 2 WHERE product_id = 12;

INSERT INTO product_stock_log (product_id, change_qty, reason, order_id)
VALUES (12, -2, 'item_swapped_in', 6);

-- 3. ลบรายการเดิมและสร้างรายการใหม่ใน order_items
DELETE FROM order_items WHERE order_id = 6 AND product_id = 13;

INSERT INTO order_items (order_id, product_id, quantity, unit_price)
VALUES (6, 12, 2, 990.00);

RELEASE SAVEPOINT swap_in;

COMMIT;
```

ถ้าขั้นตอนตรวจสอบสต๊อกใน `DO` block พบว่า dress มีไม่พอ คำสั่ง `RAISE EXCEPTION` จะทำให้ transaction ตั้งแต่จุด `SAVEPOINT swap_in` เข้าสถานะที่ต้องจัดการ — ในกรณีนี้เนื่องจากเราไม่มี `ROLLBACK TO SAVEPOINT swap_in` ตามหลัง exception (เพราะไม่มีทางแก้ไขให้สำเร็จได้ในสถานการณ์นี้) แอปพลิเคชันควรจับ error นี้แล้วสั่ง `ROLLBACK;` แบบเต็ม เพื่อยกเลิกทั้งการคืนสต๊อก handbag และทุกอย่างที่ทำไปด้วย เพื่อให้แน่ใจว่าจะไม่มีสถานการณ์ที่ handbag ถูกคืนสต๊อกไปแล้ว แต่ dress ยังไม่ถูกหัก และรายการใน order_items ยังเป็นของเดิม (ซึ่งจะทำให้ข้อมูลไม่สอดคล้องกัน)

</details>

---

**บทถัดไป:** [Part 038: Isolation Levels](./part-038-isolation-levels.md) — เจาะลึกระดับ Isolation ทั้งหมดของ PostgreSQL, phenomena ต่าง ๆ ที่อาจเกิดขึ้นเมื่อ transaction รันพร้อมกัน, และเทคนิคการจัดการ concurrency ในสถานการณ์จริง
