# Isolation Levels และปัญหา Concurrency (Dirty Read, Phantom Read)

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับกลาง | Part 038

---

## เป้าหมายการเรียนรู้

เมื่อจบบทนี้ ผู้เรียนจะสามารถ:

- อธิบายได้ว่าทำไม transaction หลายตัวที่ทำงาน "พร้อมกัน" (concurrent) ถึงเกิดปัญหา และปัญหาคลาสสิก 3 แบบคืออะไร (dirty read, non-repeatable read, phantom read)
- อธิบายความแตกต่างระหว่าง isolation level ทั้ง 4 ระดับตามมาตรฐาน SQL (READ UNCOMMITTED, READ COMMITTED, REPEATABLE READ, SERIALIZABLE)
- เข้าใจว่า PostgreSQL implement isolation level เหล่านี้จริง ๆ อย่างไร (โดยเฉพาะว่า READ UNCOMMITTED ใน PostgreSQL ไม่มี dirty read)
- จำลองปัญหา concurrency จริงด้วยการเปิด 2 psql session พร้อมกัน แล้วสลับรันคำสั่งทีละขั้น
- เข้าใจกลไก MVCC (Multi-Version Concurrency Control) แบบ snapshot ที่ REPEATABLE READ ใช้
- เข้าใจว่า SERIALIZABLE ป้องกัน serialization anomaly ได้อย่างไร และทำไมมันอาจโยน error `serialization_failure` (SQLSTATE 40001)
- แก้ปัญหา "overselling" (ขายสินค้าเกินสต๊อกที่มี) ด้วย isolation level ที่เหมาะสม พร้อมเขียน retry logic เมื่อเจอ serialization failure
- เลือก isolation level ให้เหมาะกับงานแต่ละประเภท โดยชั่งน้ำหนักระหว่างความถูกต้อง (correctness) กับประสิทธิภาพ (performance)

---

## เตรียมข้อมูล

บทนี้ใช้ชุดข้อมูลร้านค้าออนไลน์ (e-commerce) ชุดเดียวกับ Part 021–039 ทั้งหมด ถ้าคุณสร้างตารางเหล่านี้ไปแล้วจากบทก่อนหน้า สามารถข้ามไปยัง Step 371 ได้เลย แต่ถ้าต้องการสร้างใหม่ทั้งหมดสำหรับบทนี้โดยเฉพาะ (เช่น สร้าง schema แยกด้วย `CREATE SCHEMA`) ให้รันสคริปต์ด้านล่างนี้

```sql
-- ล้างตารางเก่า (ถ้ามี) เรียงตามลำดับ dependency
DROP TABLE IF EXISTS payments CASCADE;
DROP TABLE IF EXISTS reviews CASCADE;
DROP TABLE IF EXISTS order_items CASCADE;
DROP TABLE IF EXISTS orders CASCADE;
DROP TABLE IF EXISTS employees CASCADE;
DROP TABLE IF EXISTS customers CASCADE;
DROP TABLE IF EXISTS products CASCADE;
DROP TABLE IF EXISTS suppliers CASCADE;
DROP TABLE IF EXISTS categories CASCADE;

-- ตารางหมวดหมู่สินค้า (รองรับหมวดหมู่ย่อยผ่าน self-reference)
CREATE TABLE categories (
    category_id         SERIAL PRIMARY KEY,
    category_name       VARCHAR(100) NOT NULL,
    parent_category_id  INTEGER REFERENCES categories(category_id)
);

-- ตารางผู้จัดหาสินค้า
CREATE TABLE suppliers (
    supplier_id    SERIAL PRIMARY KEY,
    supplier_name  VARCHAR(150) NOT NULL,
    country        VARCHAR(60)
);

-- ตารางสินค้า
CREATE TABLE products (
    product_id      SERIAL PRIMARY KEY,
    product_name    VARCHAR(150) NOT NULL,
    category_id     INTEGER REFERENCES categories(category_id),
    supplier_id     INTEGER REFERENCES suppliers(supplier_id),
    unit_price      NUMERIC(10,2) NOT NULL CHECK (unit_price >= 0),
    stock_quantity  INTEGER NOT NULL DEFAULT 0,
    is_active       BOOLEAN NOT NULL DEFAULT true
);

-- ตารางลูกค้า
CREATE TABLE customers (
    customer_id   SERIAL PRIMARY KEY,
    first_name    VARCHAR(60) NOT NULL,
    last_name     VARCHAR(60) NOT NULL,
    email         VARCHAR(150) UNIQUE,
    country       VARCHAR(60),
    signup_date   DATE NOT NULL DEFAULT CURRENT_DATE
);

-- ตารางพนักงาน (มีโครงสร้าง manager แบบ self-reference)
CREATE TABLE employees (
    employee_id   SERIAL PRIMARY KEY,
    first_name    VARCHAR(60) NOT NULL,
    last_name     VARCHAR(60) NOT NULL,
    hire_date     DATE NOT NULL,
    manager_id    INTEGER REFERENCES employees(employee_id),
    department    VARCHAR(60)
);

-- ตารางคำสั่งซื้อ
CREATE TABLE orders (
    order_id      SERIAL PRIMARY KEY,
    customer_id   INTEGER REFERENCES customers(customer_id),
    employee_id   INTEGER REFERENCES employees(employee_id),
    order_date    TIMESTAMPTZ NOT NULL DEFAULT now(),
    status        VARCHAR(20) NOT NULL DEFAULT 'pending',
    ship_country  VARCHAR(60)
);

-- ตารางรายการสินค้าในคำสั่งซื้อ
CREATE TABLE order_items (
    order_item_id  SERIAL PRIMARY KEY,
    order_id       INTEGER REFERENCES orders(order_id),
    product_id     INTEGER REFERENCES products(product_id),
    quantity       INTEGER NOT NULL CHECK (quantity > 0),
    unit_price     NUMERIC(10,2) NOT NULL
);

-- ตารางรีวิวสินค้า
CREATE TABLE reviews (
    review_id     SERIAL PRIMARY KEY,
    product_id    INTEGER REFERENCES products(product_id),
    customer_id   INTEGER REFERENCES customers(customer_id),
    rating        INTEGER NOT NULL CHECK (rating BETWEEN 1 AND 5),
    review_text   TEXT,
    review_date   DATE NOT NULL DEFAULT CURRENT_DATE
);

-- ตารางการชำระเงิน
CREATE TABLE payments (
    payment_id      SERIAL PRIMARY KEY,
    order_id        INTEGER REFERENCES orders(order_id),
    payment_date    TIMESTAMPTZ NOT NULL DEFAULT now(),
    amount          NUMERIC(10,2) NOT NULL,
    payment_method  VARCHAR(30)
);
```

### ข้อมูลตัวอย่าง (seed data)

```sql
-- categories
INSERT INTO categories (category_id, category_name, parent_category_id) VALUES
 (1, 'Electronics',              NULL),
 (2, 'Computers & Laptops',      1),
 (3, 'Smartphones',              1),
 (4, 'Fashion',                  NULL),
 (5, 'Men''s Clothing',          4),
 (6, 'Women''s Clothing',        4),
 (7, 'Home & Kitchen',           NULL),
 (8, 'Books',                    NULL),
 (9, 'Sports & Outdoor',         NULL),
 (10, 'Beauty & Personal Care',  NULL);
SELECT setval('categories_category_id_seq', 10);

-- suppliers
INSERT INTO suppliers (supplier_id, supplier_name, country) VALUES
 (1, 'TechSource Co., Ltd.',        'Thailand'),
 (2, 'Global Gadgets Inc.',         'China'),
 (3, 'Nordic Home Supplies',        'Sweden'),
 (4, 'Pacific Apparel Group',       'Vietnam'),
 (5, 'BookWorld Distribution',      'USA'),
 (6, 'UrbanFit Sports',             'Thailand'),
 (7, 'Sakura Electronics',          'Japan'),
 (8, 'EuroStyle Fashion',           'Italy'),
 (9, 'GreenLeaf Beauty',            'South Korea'),
 (10, 'Summit Outdoor Gear',        'Thailand');
SELECT setval('suppliers_supplier_id_seq', 10);

-- products (สังเกต product_id 3, 15, 18 มี stock_quantity น้อย -- ใช้จำลอง overselling ในบทนี้)
INSERT INTO products (product_id, product_name, category_id, supplier_id, unit_price, stock_quantity, is_active) VALUES
 (1,  'Wireless Mouse Pro',          2, 1,  590.00,  150, true),
 (2,  'Mechanical Keyboard RGB',     2, 2,  2490.00, 80,  true),
 (3,  'UltraBook 14" Laptop',        2, 1,  28900.00, 3,  true),
 (4,  'Smartphone Nova X5',          3, 7,  15900.00, 45, true),
 (5,  'Smartphone Case Clear',       3, 2,  190.00,  500, true),
 (6,  'Men''s Denim Jacket',         5, 4,  1290.00, 60,  true),
 (7,  'Men''s Running Shoes',        5, 6,  2190.00, 40,  true),
 (8,  'Women''s Summer Dress',       6, 8,  990.00,  70,  true),
 (9,  'Women''s Leather Handbag',    6, 8,  3490.00, 25,  true),
 (10, 'Non-stick Frying Pan Set',    7, 3,  1590.00, 55,  true),
 (11, 'Electric Kettle 1.7L',        7, 3,  890.00,  90,  true),
 (12, 'PostgreSQL for Beginners',    8, 5,  450.00,  120, true),
 (13, 'Data Engineering Handbook',   8, 5,  690.00,  65,  true),
 (14, 'Yoga Mat Premium',            9, 10, 590.00,  100, true),
 (15, 'Camping Tent 4-Person',       9, 10, 4590.00, 12,  true),
 (16, 'Facial Serum Vitamin C',      10, 9, 690.00,  200, true),
 (17, 'Organic Shampoo Bar',         10, 9, 250.00,  300, true),
 (18, 'Bluetooth Earbuds Air',       1, 7,  1990.00, 5,   true);
SELECT setval('products_product_id_seq', 18);

-- customers
INSERT INTO customers (customer_id, first_name, last_name, email, country, signup_date) VALUES
 (1,  'Somchai',   'Charoen',   'somchai.c@example.com',   'Thailand', '2024-01-15'),
 (2,  'Nattaya',   'Suksawat',  'nattaya.s@example.com',   'Thailand', '2024-01-20'),
 (3,  'Kittipong', 'Wongsa',    'kittipong.w@example.com', 'Thailand', '2024-02-03'),
 (4,  'Maria',     'Garcia',    'maria.g@example.com',     'Spain',    '2024-02-10'),
 (5,  'John',      'Smith',     'john.smith@example.com',  'USA',      '2024-02-18'),
 (6,  'Ploy',      'Sirisak',   'ploy.s@example.com',      'Thailand', '2024-03-01'),
 (7,  'Wei',       'Zhang',     'wei.zhang@example.com',   'China',    '2024-03-08'),
 (8,  'Anong',     'Petch',     'anong.p@example.com',     'Thailand', '2024-03-15'),
 (9,  'Yuki',      'Tanaka',    'yuki.t@example.com',      'Japan',    '2024-04-02'),
 (10, 'Siriporn',  'Kaewta',    'siriporn.k@example.com',  'Thailand', '2024-04-10'),
 (11, 'David',     'Miller',    'david.m@example.com',     'USA',      '2024-04-22'),
 (12, 'Pimchanok',  'Rattana',  'pimchanok.r@example.com', 'Thailand', '2024-05-05'),
 (13, 'Hanna',      'Muller',   'hanna.m@example.com',     'Germany',  '2024-05-19'),
 (14, 'Thanawat',   'Boon',     'thanawat.b@example.com',  'Thailand', '2024-06-01');
SELECT setval('customers_customer_id_seq', 14);

-- employees (มีโครงสร้างสายบังคับบัญชา)
INSERT INTO employees (employee_id, first_name, last_name, hire_date, manager_id, department) VALUES
 (1, 'Suchart',  'Anantasin', '2020-01-10', NULL, 'Management'),
 (2, 'Kanya',    'Thongdee',  '2020-06-15', 1,    'Sales'),
 (3, 'Prasert',  'Leelawat',  '2021-02-01', 1,    'Warehouse'),
 (4, 'Areeya',   'Chan',      '2021-08-20', 2,    'Sales'),
 (5, 'Montri',   'Sanguan',   '2022-01-05', 2,    'Sales'),
 (6, 'Wipada',   'Ruangsri',  '2022-05-17', 3,    'Warehouse'),
 (7, 'Chaiwat',  'Ittipong',  '2023-01-09', 3,    'Warehouse'),
 (8, 'Napat',    'Sirichai',  '2023-09-11', 2,    'Sales');
SELECT setval('employees_employee_id_seq', 8);

-- orders
INSERT INTO orders (order_id, customer_id, employee_id, order_date, status, ship_country) VALUES
 (1,  1,  4, '2024-06-01 10:15:00+07', 'delivered',  'Thailand'),
 (2,  2,  4, '2024-06-03 14:20:00+07', 'delivered',  'Thailand'),
 (3,  3,  5, '2024-06-05 09:05:00+07', 'shipped',    'Thailand'),
 (4,  4,  5, '2024-06-07 16:40:00+07', 'delivered',  'Spain'),
 (5,  5,  8, '2024-06-10 11:25:00+07', 'delivered',  'USA'),
 (6,  6,  4, '2024-06-12 13:00:00+07', 'processing', 'Thailand'),
 (7,  7,  8, '2024-06-15 08:50:00+07', 'delivered',  'China'),
 (8,  8,  5, '2024-06-18 17:10:00+07', 'cancelled',  'Thailand'),
 (9,  9,  4, '2024-06-20 12:30:00+07', 'delivered',  'Japan'),
 (10, 10, 8, '2024-06-22 10:00:00+07', 'shipped',    'Thailand'),
 (11, 11, 5, '2024-06-25 15:45:00+07', 'delivered',  'USA'),
 (12, 12, 4, '2024-06-28 09:20:00+07', 'processing', 'Thailand'),
 (13, 1,  8, '2024-07-01 14:10:00+07', 'delivered',  'Thailand'),
 (14, 13, 5, '2024-07-03 11:35:00+07', 'delivered',  'Germany'),
 (15, 6,  4, '2024-07-05 16:00:00+07', 'pending',    'Thailand'),
 (16, 14, 8, '2024-07-08 10:50:00+07', 'delivered',  'Thailand'),
 (17, 2,  5, '2024-07-10 13:25:00+07', 'shipped',    'Thailand'),
 (18, 9,  4, '2024-07-12 09:40:00+07', 'processing', 'Japan');
SELECT setval('orders_order_id_seq', 18);

-- order_items
INSERT INTO order_items (order_item_id, order_id, product_id, quantity, unit_price) VALUES
 (1,  1,  1,  2, 590.00),
 (2,  1,  12, 1, 450.00),
 (3,  2,  4,  1, 15900.00),
 (4,  2,  5,  1, 190.00),
 (5,  3,  3,  1, 28900.00),
 (6,  4,  8,  2, 990.00),
 (7,  5,  9,  1, 3490.00),
 (8,  6,  14, 3, 590.00),
 (9,  7,  2,  1, 2490.00),
 (10, 8,  6,  1, 1290.00),
 (11, 9,  18, 1, 1990.00),
 (12, 9,  5,  2, 190.00),
 (13, 10, 10, 1, 1590.00),
 (14, 11, 7,  2, 2190.00),
 (15, 12, 13, 2, 690.00),
 (16, 13, 1,  1, 590.00),
 (17, 13, 11, 1, 890.00),
 (18, 14, 15, 1, 4590.00),
 (19, 15, 3,  1, 28900.00),
 (20, 16, 16, 3, 690.00),
 (21, 16, 17, 2, 250.00),
 (22, 17, 4,  1, 15900.00),
 (23, 18, 18, 1, 1990.00),
 (24, 18, 9,  1, 3490.00),
 (25, 6,  1,  1, 590.00);
SELECT setval('order_items_order_item_id_seq', 25);

-- reviews
INSERT INTO reviews (review_id, product_id, customer_id, rating, review_text, review_date) VALUES
 (1,  1,  1,  5, 'เมาส์ลื่นมาก ใช้งานสบาย',        '2024-06-05'),
 (2,  3,  3,  4, 'สเปกดี แต่ราคาสูงไปนิด',          '2024-06-10'),
 (3,  4,  2,  5, 'กล้องถ่ายชัดมาก คุ้มราคา',        '2024-06-08'),
 (4,  9,  5,  4, 'หนังแท้สวย ทนทาน',                '2024-06-15'),
 (5,  14, 6,  5, 'เสื่อโยคะหนาดี ไม่ลื่น',          '2024-06-18'),
 (6,  2,  7,  3, 'ปุ่มแข็งไปหน่อยตอนแรก ใช้ไปสักพักดีขึ้น', '2024-06-20'),
 (7,  6,  8,  2, 'ไซส์เล็กกว่าที่คิด',              '2024-06-22'),
 (8,  18, 9,  5, 'เสียงดีเกินราคา แบตอึด',          '2024-06-25'),
 (9,  10, 10, 4, 'กระทะเกาะติดดี',                  '2024-06-27'),
 (10, 13, 12, 5, 'หนังสืออ่านง่าย เนื้อหาแน่น',      '2024-07-01'),
 (11, 15, 13, 4, 'เต็นท์กันน้ำได้ดี ประกอบง่าย',     '2024-07-05'),
 (12, 16, 14, 5, 'เซรั่มซึมไว ผิวกระจ่างขึ้น',       '2024-07-08');
SELECT setval('reviews_review_id_seq', 12);

-- payments
INSERT INTO payments (payment_id, order_id, payment_date, amount, payment_method) VALUES
 (1,  1,  '2024-06-01 10:20:00+07', 1630.00,  'credit_card'),
 (2,  2,  '2024-06-03 14:25:00+07', 16090.00, 'promptpay'),
 (3,  3,  '2024-06-05 09:10:00+07', 28900.00, 'credit_card'),
 (4,  4,  '2024-06-07 16:45:00+07', 1980.00,  'bank_transfer'),
 (5,  5,  '2024-06-10 11:30:00+07', 3490.00,  'credit_card'),
 (6,  7,  '2024-06-15 08:55:00+07', 2490.00,  'promptpay'),
 (7,  9,  '2024-06-20 12:35:00+07', 2370.00,  'credit_card'),
 (8,  10, '2024-06-22 10:05:00+07', 1590.00,  'cod'),
 (9,  11, '2024-06-25 15:50:00+07', 4380.00,  'credit_card'),
 (10, 13, '2024-07-01 14:15:00+07', 1480.00,  'promptpay'),
 (11, 14, '2024-07-03 11:40:00+07', 4590.00,  'bank_transfer'),
 (12, 16, '2024-07-08 10:55:00+07', 2570.00,  'credit_card'),
 (13, 17, '2024-07-10 13:30:00+07', 15900.00, 'promptpay'),
 (14, 4,  '2024-06-07 16:46:00+07', 0.00,     'bank_transfer'),
 (15, 6,  '2024-06-12 13:05:00+07', 1770.00,  'cod');
SELECT setval('payments_payment_id_seq', 15);
```

> **หมายเหตุ:** ในบทนี้เราจะให้ความสำคัญเป็นพิเศษกับตาราง `products` โดยเฉพาะคอลัมน์ `stock_quantity` เพราะปัญหา concurrency ที่พบบ่อยที่สุดในระบบ e-commerce คือ "สต๊อกสินค้าติดลบ" หรือ "ขายเกินจำนวนที่มี" (overselling) — สินค้า `product_id = 3` (UltraBook 14" Laptop, stock = 3) และ `product_id = 18` (Bluetooth Earbuds Air, stock = 5) จะถูกใช้เป็นตัวอย่างหลักตลอดบทนี้

---

## Step 371: ปัญหา Concurrency คืออะไร

ในระบบฐานข้อมูลจริง ไม่มีทางที่จะมี "ผู้ใช้คนเดียว" รันคำสั่งทีละคำสั่งอย่างเป็นระเบียบ ในทางปฏิบัติจะมี**หลาย connection** (หลาย session, หลาย transaction) พยายามอ่านและเขียนข้อมูล**พร้อมกัน** — เช่น ลูกค้า 50 คนกด "สั่งซื้อ" สินค้าตัวเดียวกันในเวลาไล่เลี่ยกัน หรือพนักงานหลายคนอัปเดตออร์เดอร์เดียวกันพร้อมกัน

**Concurrency** (การทำงานพร้อมกัน) คือสถานการณ์ที่ transaction หลายตัวเข้าถึงข้อมูลชุดเดียวกันในช่วงเวลาที่ทับซ้อนกัน ถ้าฐานข้อมูลไม่มีกลไกควบคุมที่ดีพอ อาจเกิดผลลัพธ์ที่**ผิดพลาดทางตรรกะ** (logical inconsistency) แม้ว่าแต่ละ transaction เมื่อดูแยกกันจะถูกต้องทุกขั้นตอนก็ตาม

### ทำไมถึงเป็นปัญหา

ลองจินตนาการสถานการณ์จริง: สินค้า `UltraBook 14" Laptop` (product_id = 3) เหลือสต๊อก **3 ชิ้น**

1. ลูกค้า A เปิดหน้าเว็บ เห็นว่ามีสินค้าเหลือ 3 ชิ้น กดสั่งซื้อ 2 ชิ้น
2. ในเวลาไล่เลี่ยกัน ลูกค้า B ก็เปิดหน้าเว็บ เห็นว่ามีสินค้าเหลือ 3 ชิ้น (เพราะ A ยังไม่ commit) กดสั่งซื้อ 2 ชิ้นเช่นกัน
3. ถ้าระบบตรวจสอบสต๊อก "ก่อน" แล้วค่อยตัดสต๊อก "ทีหลัง" โดยไม่มีกลไกป้องกันที่ดีพอ ทั้ง A และ B อาจสั่งซื้อสำเร็จทั้งคู่ รวมเป็น 4 ชิ้น ทั้งที่มีของจริงแค่ 3 ชิ้น

นี่คือปัญหา **overselling** ซึ่งเป็นผลลัพธ์ของ **race condition** — ผลลัพธ์สุดท้ายขึ้นอยู่กับ "จังหวะเวลา" (timing) ของการรัน ไม่ใช่ตรรกะทางธุรกิจที่ถูกต้อง

### ปัญหา concurrency คลาสสิก 3 แบบ

มาตรฐาน SQL (ANSI SQL) นิยามปัญหา concurrency หลัก ๆ ไว้ 3 แบบ (และเพิ่ม lost update / serialization anomaly ในการวิเคราะห์เชิงลึก):

| ปัญหา | คำอธิบายสั้น ๆ |
|---|---|
| **Dirty Read** | อ่านข้อมูลที่ transaction อื่นยังไม่ commit (อาจถูก rollback ทีหลัง) |
| **Non-repeatable Read** | อ่านแถวเดิมสองครั้งในทรานแซคชันเดียว ได้ค่าต่างกัน เพราะ transaction อื่น update และ commit แทรกเข้ามา |
| **Phantom Read** | query เงื่อนไขเดิมสองครั้ง ได้จำนวนแถวต่างกัน เพราะ transaction อื่น insert/delete แถวใหม่ที่ตรงเงื่อนไขแล้ว commit แทรกเข้ามา |
| **Lost Update** | transaction สองตัวอ่านค่าเดิม แล้วต่างคน update ทับกัน ทำให้การเปลี่ยนแปลงของตัวหนึ่ง "หาย" ไป |
| **Serialization Anomaly** | ผลลัพธ์สุดท้ายของ transaction ที่รันพร้อมกัน ไม่ตรงกับผลลัพธ์ที่ควรได้ถ้ารันทีละตัวเรียงลำดับ (serial order) ไม่ว่าจะเรียงลำดับไหนก็ตาม |

Isolation level ที่เราจะเรียนในบทนี้ คือกลไกที่ฐานข้อมูลใช้ "กั้น" ไม่ให้ transaction หนึ่งเห็นผลกระทบบางอย่างจาก transaction อื่นที่ยังทำงานอยู่ (หรือทำงานพร้อมกัน) — ยิ่ง isolation level สูง ยิ่งป้องกันปัญหาได้มากขึ้น แต่ก็มัก "แลก" มาด้วยการที่ transaction อาจถูกบล็อกหรือถูกยกเลิก (abort) บ่อยขึ้น

### วิธีทดสอบในบทนี้: สอง session

ตลอดบทนี้เราจะใช้วิธี**เปิด psql สองหน้าต่าง (terminal) พร้อมกัน** แล้วเรียกว่า **Session A** และ **Session B** จากนั้นสลับกันรันคำสั่งทีละขั้นตามลำดับเวลา (T1, T2, T3, ...) เพื่อจำลองสถานการณ์ concurrency จริง

```sql
-- เปิด terminal ที่ 1
psql -U postgres -d ecommerce_course   -- นี่คือ Session A

-- เปิด terminal ที่ 2 (อีกหน้าต่างหนึ่ง)
psql -U postgres -d ecommerce_course   -- นี่คือ Session B
```

ทุกตัวอย่างในบทนี้จะแสดงคำสั่งของ Session A และ Session B แยกกันตามลำดับเวลา พร้อมระบุ "เวลา" กำกับไว้ชัดเจน เพื่อให้เห็นว่าถ้าเอาไปรันจริงในสอง terminal จะต้องรันคำสั่งไหนก่อนหลัง

---

## Step 372: Dirty Read

**Dirty Read** คือการที่ transaction หนึ่งอ่านข้อมูลที่ transaction อีกตัวหนึ่ง **แก้ไขแล้วแต่ยังไม่ commit** — ถ้า transaction ที่แก้ไขนั้นถูก `ROLLBACK` ทีหลัง ข้อมูลที่อ่านไปก่อนหน้านี้ก็จะกลายเป็นข้อมูลที่ "ไม่เคยมีอยู่จริง" (dirty)

### สถานการณ์จำลอง

สมมติว่าฐานข้อมูลรองรับ dirty read (ซึ่ง PostgreSQL **ไม่รองรับ** แต่เราจะสมมติเพื่อให้เข้าใจแนวคิดก่อน แล้วค่อยพิสูจน์ในขั้นถัดไปว่า PostgreSQL ป้องกันได้จริง):

**เวลา T1 — Session A:**
```sql
BEGIN;
UPDATE products
SET stock_quantity = stock_quantity - 100
WHERE product_id = 12;   -- 'PostgreSQL for Beginners' เดิมมี 120

-- UPDATE 1  (ยังไม่ COMMIT)
```

**เวลา T2 — Session B (ถ้าเป็นระบบที่มี dirty read):**
```sql
SELECT stock_quantity FROM products WHERE product_id = 12;

-- ถ้าเป็นระบบที่มี dirty read จะเห็น: stock_quantity = 20
-- (ค่าที่ A ยัง "ไม่ commit" เลย!)
```

**เวลา T3 — Session A:**
```sql
ROLLBACK;   -- A เปลี่ยนใจ ยกเลิกการเปลี่ยนแปลง

-- ค่าจริงในตารางกลับไปเป็น stock_quantity = 120 เหมือนเดิม
```

ถ้า Session B อ่านค่าไปแล้วตอน T2 (เห็นว่าเหลือ 20) แล้วนำค่านั้นไปตัดสินใจอะไรบางอย่าง (เช่น แจ้งเตือนลูกค้าว่า "สินค้าใกล้หมด") ข้อมูลนั้นก็จะเป็นข้อมูลเท็จ เพราะความจริงคือสต๊อกไม่เคยลดลงเลย — นี่คือปัญหาของ dirty read

### พิสูจน์ว่า PostgreSQL ไม่มี dirty read

มาลองพิสูจน์จริงด้วยสอง session:

**เวลา T1 — Session A:**
```sql
BEGIN;
UPDATE products
SET stock_quantity = stock_quantity - 100
WHERE product_id = 12;
-- UPDATE 1  (ยังไม่ commit ค้างไว้)
```

**เวลา T2 — Session B:**
```sql
-- ลองอ่านค่าแบบ default isolation level ของ PostgreSQL (READ COMMITTED)
SELECT product_name, stock_quantity FROM products WHERE product_id = 12;
```

ผลลัพธ์ที่ได้จริง:

```
      product_name       | stock_quantity
--------------------------+----------------
 PostgreSQL for Beginners |            120
(1 row)
```

Session B **ยังเห็นค่าเดิม (120)** ทั้งที่ Session A ได้ `UPDATE` ไปแล้ว (แค่ยังไม่ commit) — นี่คือหลักฐานว่า PostgreSQL ป้องกัน dirty read ได้ในทุก isolation level แม้จะตั้งเป็น `READ UNCOMMITTED` ก็ตาม (รายละเอียดใน Step 375)

**เวลา T3 — Session A:**
```sql
ROLLBACK;
```

### ทำไม dirty read เป็นปัญหาที่อันตรายที่สุด

- ข้อมูลที่อ่านไปอาจ**ไม่เคยมีอยู่จริง** ถ้า transaction ต้นทางถูก rollback
- ถ้า Session B นำค่า "ผี" นี้ไปคำนวณต่อ แล้วเขียนกลับลงฐานข้อมูล จะทำให้ข้อมูลเสียหายแบบถาวร (data corruption) ที่ตรวจสอบย้อนหลังได้ยากมาก
- ฐานข้อมูลสมัยใหม่แทบทุกตัว (รวมถึง PostgreSQL, Oracle, SQL Server ในโหมดปกติ) จึงออกแบบมาไม่ให้เกิด dirty read เลย ไม่ว่าจะตั้ง isolation level ใดก็ตาม

---

## Step 373: Non-repeatable Read

**Non-repeatable Read** คือการที่ transaction หนึ่ง `SELECT` แถวเดิมสองครั้งในทรานแซคชันเดียวกัน แต่ได้**ค่าต่างกัน** เพราะระหว่างสองครั้งนั้น มี transaction อื่น `UPDATE` แถวนั้นแล้ว `COMMIT` สำเร็จไปแล้ว

ต่างจาก dirty read ตรงที่ non-repeatable read เกิดจากการอ่านข้อมูลที่ **commit แล้วจริง ๆ** — ไม่ใช่ข้อมูลผี แต่ปัญหาคือ transaction ของเราเอง**เห็นข้อมูลเปลี่ยนกลางทาง** ซึ่งอาจทำให้ตรรกะภายใน transaction ขัดแย้งกันเอง

### สถานการณ์จำลอง: READ COMMITTED (default ของ PostgreSQL)

**เวลา T1 — Session A:**
```sql
BEGIN;
-- ใช้ isolation level default (READ COMMITTED)
SELECT unit_price FROM products WHERE product_id = 4;   -- Smartphone Nova X5
```

ผลลัพธ์:
```
 unit_price
------------
   15900.00
(1 row)
```

**เวลา T2 — Session B:**
```sql
BEGIN;
UPDATE products SET unit_price = 14900.00 WHERE product_id = 4;  -- ลดราคาโปรโมชัน
COMMIT;
```

**เวลา T3 — Session A (ยังอยู่ใน transaction เดิม อ่านซ้ำแถวเดียวกัน):**
```sql
SELECT unit_price FROM products WHERE product_id = 4;
```

ผลลัพธ์:
```
 unit_price
------------
   14900.00
(1 row)
```

```sql
COMMIT;
```

สังเกตว่า **ภายใน transaction เดียวกันของ Session A** การ `SELECT` แถวเดียวกันสองครั้ง ได้ผลลัพธ์ต่างกัน (15900.00 → 14900.00) นี่คือ non-repeatable read — เกิดขึ้นได้ใน isolation level `READ COMMITTED` ซึ่งเป็นค่า default ของ PostgreSQL

### ทำไมเป็นปัญหา

ลองนึกภาพระบบที่คำนวณราคารวมของตะกร้าสินค้าโดยอ่านราคาสินค้าแต่ละชิ้นหลายครั้งในทรานแซคชันเดียว (เช่น อ่านครั้งแรกเพื่อแสดงผล และอ่านอีกครั้งเพื่อคำนวณยอดสุดท้ายก่อนตัดเงิน) ถ้าราคาเปลี่ยนกลางทาง ลูกค้าอาจเห็นราคาหนึ่ง แต่ถูกตัดเงินอีกราคาหนึ่ง — เกิดความไม่สอดคล้องภายใน business logic เดียวกัน

### แก้ปัญหาด้วย REPEATABLE READ (preview)

ถ้าเปลี่ยน isolation level ของ Session A เป็น `REPEATABLE READ` การอ่านซ้ำแถวเดียวกันจะได้ค่าเดิมเสมอตลอด transaction (รายละเอียดเต็มใน Step 376) — เราจะสาธิตให้เห็นตรงนั้น

---

## Step 374: Phantom Read

**Phantom Read** คล้ายกับ non-repeatable read แต่ต่างตรงที่เป็นเรื่องของ**จำนวนแถวที่ตรงเงื่อนไข** ไม่ใช่ค่าของแถวเดิม — คือการ query ด้วยเงื่อนไข (`WHERE`) เดิมสองครั้งในทรานแซคชันเดียว แต่ได้**จำนวนแถวต่างกัน** เพราะมี transaction อื่น `INSERT` หรือ `DELETE` แถวที่ตรงเงื่อนไขนั้น แล้ว commit แทรกเข้ามา

### สถานการณ์จำลอง

**เวลา T1 — Session A:**
```sql
BEGIN;
-- นับจำนวนสินค้าในหมวด Sports & Outdoor (category_id = 9) ที่ราคาต่ำกว่า 1000 บาท
SELECT product_id, product_name, unit_price
FROM products
WHERE category_id = 9 AND unit_price < 1000;
```

ผลลัพธ์:
```
 product_id |   product_name   | unit_price
------------+-------------------+------------
         14 | Yoga Mat Premium  |     590.00
(1 row)
```

**เวลา T2 — Session B:**
```sql
BEGIN;
INSERT INTO products (product_name, category_id, supplier_id, unit_price, stock_quantity, is_active)
VALUES ('Resistance Band Set', 9, 10, 350.00, 200, true);
COMMIT;
```

**เวลา T3 — Session A (query เงื่อนไขเดิมซ้ำในทรานแซคชันเดียวกัน):**
```sql
SELECT product_id, product_name, unit_price
FROM products
WHERE category_id = 9 AND unit_price < 1000;
```

ผลลัพธ์ (ภายใต้ READ COMMITTED):
```
 product_id |     product_name     | unit_price
------------+-----------------------+------------
         14 | Yoga Mat Premium      |     590.00
         19 | Resistance Band Set   |     350.00
(2 rows)
```

```sql
COMMIT;
```

จะเห็นว่าจากที่ query แรกเจอ 1 แถว query ที่สองในทรานแซคชันเดียวกันกลับเจอ 2 แถว — แถวที่สอง (`Resistance Band Set`) คือ **phantom row** ที่ "โผล่ขึ้นมา" ระหว่างทาง

### Phantom Read vs Non-repeatable Read

| | Non-repeatable Read | Phantom Read |
|---|---|---|
| เกิดกับ | แถวเดิมที่มีอยู่แล้ว (ค่าเปลี่ยน) | จำนวนแถวที่ตรงเงื่อนไข (แถวใหม่โผล่/หาย) |
| สาเหตุ | transaction อื่น `UPDATE` | transaction อื่น `INSERT` / `DELETE` |
| ป้องกันได้ตั้งแต่ | REPEATABLE READ (ในมาตรฐาน SQL ทั่วไป) | SERIALIZABLE (ในมาตรฐาน SQL ทั่วไป) |
| ป้องกันได้ใน PostgreSQL ตั้งแต่ | REPEATABLE READ | **REPEATABLE READ** (PostgreSQL ป้องกันได้มากกว่ามาตรฐานกำหนด!) |

ข้อสังเกตสำคัญ: ตามมาตรฐาน SQL, `REPEATABLE READ` ป้องกันแค่ non-repeatable read แต่ **ไม่รับประกัน**ว่าป้องกัน phantom read ได้ — แต่ **PostgreSQL implement REPEATABLE READ ด้วยกลไก snapshot ที่ป้องกัน phantom read ได้ด้วย** (เข้มงวดกว่ามาตรฐานขั้นต่ำที่กำหนด) เราจะพิสูจน์เรื่องนี้ใน Step 376

---

## Step 375: READ UNCOMMITTED และ READ COMMITTED

มาตรฐาน SQL นิยาม isolation level ไว้ 4 ระดับ เรียงจากหลวมสุดไปเข้มงวดสุด:

| Level | Dirty Read | Non-repeatable Read | Phantom Read |
|---|---|---|---|
| READ UNCOMMITTED | อาจเกิด | อาจเกิด | อาจเกิด |
| READ COMMITTED | ป้องกันได้ | อาจเกิด | อาจเกิด |
| REPEATABLE READ | ป้องกันได้ | ป้องกันได้ | อาจเกิด (ตามมาตรฐาน) |
| SERIALIZABLE | ป้องกันได้ | ป้องกันได้ | ป้องกันได้ |

นี่คือตารางมาตรฐาน SQL ที่หนังสือเรียนทั่วไปสอน — แต่ **PostgreSQL ไม่ได้ทำตามตารางนี้เป๊ะ ๆ**

### PostgreSQL รองรับ isolation level อะไรบ้างจริง ๆ

PostgreSQL รับ syntax ทั้ง 4 ระดับตามมาตรฐาน แต่**internal implementation มีแค่ 3 แบบ**:

```sql
-- ตรวจสอบ isolation level ปัจจุบันของ session
SHOW transaction_isolation;
```

```
 transaction_isolation
------------------------
 read committed
(1 row)
```

```sql
-- ตั้งค่า isolation level สำหรับ transaction ถัดไป
BEGIN;
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
SHOW transaction_isolation;
```

```
 transaction_isolation
------------------------
 read uncommitted
(1 row)
```

`SHOW` บอกว่าเป็น `read uncommitted` จริง แต่ **เบื้องหลัง PostgreSQL จะทำงานเหมือน READ COMMITTED ทุกประการ** — เอกสารทางการของ PostgreSQL ระบุไว้ชัดเจนว่า:

> "PostgreSQL's Read Uncommitted mode behaves like Read Committed" — เพราะ PostgreSQL ใช้กลไก MVCC (Multi-Version Concurrency Control) ซึ่งทำให้ transaction ไม่มีทางเห็นข้อมูลที่ยังไม่ commit ได้เลย ไม่ว่าจะตั้ง isolation level ใดก็ตาม

```sql
ROLLBACK;
```

### สรุป: PostgreSQL ไม่มี dirty read ในทุกกรณี

นี่คือจุดสำคัญที่ต้องจำ: **ไม่ว่าจะตั้ง isolation level เป็นอะไร PostgreSQL จะไม่มี dirty read เกิดขึ้นเลย** — เพราะสถาปัตยกรรม MVCC ของ PostgreSQL ทำงานโดยให้แต่ละ transaction เห็น "snapshot" ของข้อมูลที่ commit แล้วเท่านั้น ไม่มีทางเข้าไปอ่าน uncommitted row ของ transaction อื่นได้ในทางเทคนิค

### READ COMMITTED — ค่า default ของ PostgreSQL

`READ COMMITTED` คือ isolation level เริ่มต้นของ PostgreSQL (และของฐานข้อมูลส่วนใหญ่) พฤติกรรมสำคัญ:

- แต่ละคำสั่ง (statement) ภายใน transaction จะเห็น snapshot ของข้อมูล ณ **จุดเริ่มต้นของคำสั่งนั้น** (ไม่ใช่จุดเริ่มต้นของ transaction)
- หมายความว่า คำสั่ง `SELECT` สองครั้งในทรานแซคชันเดียวกัน อาจเห็นข้อมูลต่างกัน ถ้ามี transaction อื่น commit แทรกเข้ามาระหว่างสองคำสั่งนั้น (นี่คือที่มาของ non-repeatable read และ phantom read ที่เราเห็นใน Step 373-374)

ทดสอบดูค่า default:

```sql
BEGIN;
SHOW transaction_isolation;
```

```
 transaction_isolation
------------------------
 read committed
(1 row)
```

```sql
COMMIT;
```

### วิธีตั้ง isolation level

มี 2 วิธีหลัก:

```sql
-- วิธีที่ 1: ตั้งตอนเริ่ม transaction
BEGIN;
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
-- ... คำสั่งต่าง ๆ ...
COMMIT;

-- วิธีที่ 2: ใช้ BEGIN ... ISOLATION LEVEL ... โดยตรง
BEGIN ISOLATION LEVEL REPEATABLE READ;
-- ... คำสั่งต่าง ๆ ...
COMMIT;

-- วิธีที่ 3: ตั้งเป็นค่า default ของทั้ง session (ไม่แนะนำสำหรับ production ทั่วไป)
SET default_transaction_isolation = 'repeatable read';
```

> **ข้อควรระวัง:** `SET TRANSACTION ISOLATION LEVEL` ต้องเรียกก่อนคำสั่งแรกที่เข้าถึงข้อมูลใน transaction นั้น ถ้ามีการ query ไปแล้วค่อยเปลี่ยน isolation level จะเกิด error

```sql
BEGIN;
SELECT 1;  -- มี query ไปแล้ว
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
```

```
ERROR:  SET TRANSACTION ISOLATION LEVEL must be called before any query
```

```sql
ROLLBACK;
```

---

## Step 376: REPEATABLE READ

`REPEATABLE READ` คือ isolation level ที่รับประกันว่า **ทุกคำสั่ง `SELECT` ภายใน transaction เดียวกัน จะเห็นข้อมูลชุดเดียวกันเป๊ะ ๆ** ไม่ว่าจะ query กี่ครั้งก็ตาม — เปรียบเสมือนการ "ถ่ายภาพนิ่ง" (snapshot) ของฐานข้อมูล ณ จุดเริ่มต้น transaction แล้วมองผ่านภาพนิ่งนั้นไปตลอดทั้ง transaction

### กลไก snapshot ของ PostgreSQL

ต่างจาก `READ COMMITTED` ที่ snapshot ถูกสร้างใหม่ทุกครั้งที่มีคำสั่งใหม่ `REPEATABLE READ` จะสร้าง snapshot **แค่ครั้งเดียว** ตอนคำสั่งแรกที่เข้าถึงข้อมูลใน transaction แล้วใช้ snapshot เดิมนั้นตลอดทั้ง transaction

```sql
-- ภาพรวมกลไก:
-- READ COMMITTED:   snapshot ใหม่ทุก statement
-- REPEATABLE READ:  snapshot เดียว ตลอดทั้ง transaction
-- SERIALIZABLE:      snapshot เดียว + ตรวจสอบ conflict เพิ่มเติม
```

### ทดสอบแก้ปัญหา Non-repeatable Read

ทำซ้ำสถานการณ์จาก Step 373 แต่เปลี่ยน Session A ให้ใช้ `REPEATABLE READ`:

**เวลา T1 — Session A:**
```sql
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT unit_price FROM products WHERE product_id = 4;   -- Smartphone Nova X5
```

ผลลัพธ์:
```
 unit_price
------------
   14900.00
(1 row)
```
*(สมมติว่าราคาปัจจุบันคือ 14900.00 หลังจากการทดสอบใน Step 373)*

**เวลา T2 — Session B:**
```sql
BEGIN;
UPDATE products SET unit_price = 13900.00 WHERE product_id = 4;
COMMIT;
```

**เวลา T3 — Session A (query เดิมซ้ำ):**
```sql
SELECT unit_price FROM products WHERE product_id = 4;
```

ผลลัพธ์:
```
 unit_price
------------
   14900.00
(1 row)
```

ค่ายังคงเป็น **14900.00 เหมือนเดิม** แม้ Session B จะ commit การเปลี่ยนแปลงไปแล้ว! เพราะ Session A มองผ่าน snapshot ที่ถ่ายไว้ตอนเริ่ม transaction ซึ่งยังไม่เห็นการเปลี่ยนแปลงของ B

```sql
COMMIT;

-- หลังจาก commit แล้ว ถ้า SELECT ใหม่ (transaction ใหม่) จะเห็นค่าล่าสุด
SELECT unit_price FROM products WHERE product_id = 4;
```

```
 unit_price
------------
   13900.00
(1 row)
```

### ทดสอบแก้ปัญหา Phantom Read (PostgreSQL เข้มงวดกว่ามาตรฐาน)

ทำซ้ำสถานการณ์จาก Step 374:

**เวลา T1 — Session A:**
```sql
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT product_id, product_name, unit_price
FROM products
WHERE category_id = 9 AND unit_price < 1000;
```

ผลลัพธ์:
```
 product_id |     product_name     | unit_price
------------+-----------------------+------------
         14 | Yoga Mat Premium      |     590.00
         19 | Resistance Band Set   |     350.00
(2 rows)
```

**เวลา T2 — Session B:**
```sql
BEGIN;
INSERT INTO products (product_name, category_id, supplier_id, unit_price, stock_quantity, is_active)
VALUES ('Jump Rope Speed', 9, 10, 190.00, 150, true);
COMMIT;
```

**เวลา T3 — Session A (query เดิมซ้ำ):**
```sql
SELECT product_id, product_name, unit_price
FROM products
WHERE category_id = 9 AND unit_price < 1000;
```

ผลลัพธ์:
```
 product_id |     product_name     | unit_price
------------+-----------------------+------------
         14 | Yoga Mat Premium      |     590.00
         19 | Resistance Band Set   |     350.00
(2 rows)
```

```sql
COMMIT;
```

ยังคงเห็น **2 แถวเท่าเดิม** ไม่มี `Jump Rope Speed` โผล่มาเลย แม้ Session B จะ insert และ commit ไปแล้วก็ตาม — พิสูจน์ว่า **`REPEATABLE READ` ใน PostgreSQL ป้องกัน phantom read ได้ด้วย** ซึ่งเข้มงวดกว่าที่มาตรฐาน SQL กำหนดขั้นต่ำไว้

### ข้อจำกัดของ REPEATABLE READ: ปัญหา Lost Update ยังเกิดได้

แม้ `REPEATABLE READ` จะป้องกัน non-repeatable read และ phantom read ได้ แต่ยังมีปัญหา **lost update** และ **write skew** ที่หลุดรอดได้ในบางกรณี โดยเฉพาะเมื่อ transaction สองตัวต่างอ่านค่าจาก snapshot เดิม แล้วต่างคน `UPDATE` ทับกันโดยอิงจากค่าที่อ่านไป

**เวลา T1 — Session A:**
```sql
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT stock_quantity FROM products WHERE product_id = 18;  -- Bluetooth Earbuds Air
```
```
 stock_quantity
----------------
              5
(1 row)
```

**เวลา T2 — Session B:**
```sql
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT stock_quantity FROM products WHERE product_id = 18;
```
```
 stock_quantity
----------------
              5
(1 row)
```

**เวลา T3 — Session A (ตัดสต๊อก 3 ชิ้นโดยคำนวณเอง แล้ว UPDATE ด้วยค่าที่คำนวณ):**
```sql
UPDATE products SET stock_quantity = 5 - 3 WHERE product_id = 18;  -- ตั้งใจให้เหลือ 2
COMMIT;
```

**เวลา T4 — Session B (ตัดสต๊อก 4 ชิ้นโดยอิงจากค่าที่อ่านไปตอน T2 ซึ่งยังเป็น 5):**
```sql
UPDATE products SET stock_quantity = 5 - 4 WHERE product_id = 18;  -- ตั้งใจให้เหลือ 1
COMMIT;
```

ผลลัพธ์สุดท้าย:
```sql
SELECT stock_quantity FROM products WHERE product_id = 18;
```
```
 stock_quantity
----------------
              1
(1 row)
```

ในเคสนี้ทั้งสอง `UPDATE` สำเร็จ (เพราะเขียนคนละเวลา ไม่ conflict กันในระดับ row lock) แต่ผลลัพธ์สุดท้ายคือ 1 ทั้งที่ควรจะเป็น 5 - 3 - 4 = -2 (ถ้าตัดสต๊อกถูกต้องแบบ sequential) หรืออย่างน้อยควรถูกปฏิเสธไปตัวหนึ่งเพราะสต๊อกไม่พอ — นี่คือ **write-based lost update / anomaly** ที่เกิดจากการเขียนโค้ดแบบ "อ่านค่ามาแล้วคำนวณเองที่ฝั่ง application" (read-then-write) ซึ่ง `REPEATABLE READ` เพียงอย่างเดียวป้องกันไม่ได้ครบ ต้องอาศัย `SERIALIZABLE` หรือใช้ atomic update (`stock_quantity = stock_quantity - 3` ในคำสั่งเดียว) ร่วมด้วย — รายละเอียดเต็มใน Step 378

---

## Step 377: SERIALIZABLE

`SERIALIZABLE` คือ isolation level ที่**เข้มงวดที่สุด** ในมาตรฐาน SQL รับประกันว่าผลลัพธ์ของ transaction ที่รันพร้อมกันจะ**เทียบเท่ากับ**การรัน transaction เหล่านั้นทีละตัวเรียงลำดับกัน (serial order) เสมอ ไม่ว่าจะเรียงลำดับไหนก็ตาม — นี่คือการรับประกันสูงสุดที่ป้องกันปัญหา concurrency ได้ **ทุกแบบ** รวมถึง dirty read, non-repeatable read, phantom read, lost update และ write skew

### Serialization Anomaly คืออะไร

**Serialization anomaly** คือสถานการณ์ที่ transaction แต่ละตัวดูถูกต้องเมื่อพิจารณาแยกกัน แต่เมื่อรันพร้อมกันแล้วได้ผลลัพธ์ที่ **ไม่สามารถเกิดขึ้นได้เลยถ้ารันทีละตัว** ไม่ว่าจะจัดลำดับการรันแบบไหนก็ตาม ตัวอย่างคลาสสิกคือ "write skew" ที่เราเห็นใน Step 376 (ผลลัพธ์ 1 ที่ไม่ตรงกับทั้งกรณี A-ก่อน-B และ B-ก่อน-A)

### กลไกของ PostgreSQL: Serializable Snapshot Isolation (SSI)

PostgreSQL implement `SERIALIZABLE` ด้วยเทคนิคที่เรียกว่า **Serializable Snapshot Isolation (SSI)** ซึ่งทำงานบนพื้นฐานของ `REPEATABLE READ` (snapshot เดียวตลอด transaction) **บวกกับ**การติดตาม (monitor) รูปแบบการอ่าน-เขียนที่อาจก่อให้เกิด serialization anomaly แบบ real-time ถ้าตรวจพบว่ารูปแบบการเข้าถึงข้อมูลของ transaction คู่หนึ่งเสี่ยงจะทำให้เกิดผลลัพธ์ที่ไม่ serializable ระบบจะ**ยกเลิก transaction ตัวใดตัวหนึ่ง**โดยโยน error

### ทดสอบด้วยสถานการณ์เดิมจาก Step 376

**เวลา T1 — Session A:**
```sql
BEGIN ISOLATION LEVEL SERIALIZABLE;
SELECT stock_quantity FROM products WHERE product_id = 18;  -- Bluetooth Earbuds Air
```
```
 stock_quantity
----------------
              1
(1 row)
```
*(ต่อเนื่องจากผลลัพธ์ Step 376 ที่เหลือ 1)*

**เวลา T2 — Session B:**
```sql
BEGIN ISOLATION LEVEL SERIALIZABLE;
SELECT stock_quantity FROM products WHERE product_id = 18;
```
```
 stock_quantity
----------------
              1
(1 row)
```

**เวลา T3 — Session A:**
```sql
UPDATE products SET stock_quantity = stock_quantity - 1 WHERE product_id = 18;
COMMIT;
```
```
UPDATE 1
COMMIT
```

**เวลา T4 — Session B (พยายาม update ตามหลัง โดยอิงจาก snapshot เดิมที่เห็นค่า 1):**
```sql
UPDATE products SET stock_quantity = stock_quantity - 1 WHERE product_id = 18;
```

ที่จุดนี้ PostgreSQL จะปล่อยให้ `UPDATE` รันผ่านไปก่อน (เพราะ row-level lock ไม่ได้ชนกันตรง ๆ ในทันที) แต่เมื่อ Session B พยายาม `COMMIT`:

```sql
COMMIT;
```

```
ERROR:  could not serialize access due to read/write dependencies among transactions
DETAIL:  Reason code: Canceled on identification as a pivot, during commit attempt.
HINT:  The transaction might succeed if retried.
```

PostgreSQL ตรวจพบว่าถ้าปล่อยให้ transaction ทั้งสองสำเร็จพร้อมกัน จะเกิดผลลัพธ์ที่ไม่ตรงกับการรันแบบ serial (เพราะ B อ่านค่าก่อน A commit แต่ B พยายามเขียนทับหลังจาก A commit ไปแล้วโดยอิงจากค่าเก่า) ระบบจึงเลือก**ยกเลิก Session B** ด้วย SQLSTATE `40001` (`serialization_failure`) — Session B ต้อง `ROLLBACK` แล้วลองใหม่

```sql
ROLLBACK;
```

### ตรวจสอบ error code ด้วยตัวเอง

```sql
-- ดู SQLSTATE ของ serialization failure
SELECT '40001' AS serialization_failure_sqlstate;
```

```
 serialization_failure_sqlstate
---------------------------------
 40001
(1 row)
```

ในโค้ด application (เช่น Python, Node.js, Java) เราสามารถดักจับ error code `40001` เพื่อสั่ง retry transaction ได้อัตโนมัติ (รายละเอียดเต็มใน Step 379)

### ข้อควรพิจารณาก่อนใช้ SERIALIZABLE

| ข้อดี | ข้อควรระวัง |
|---|---|
| ป้องกันปัญหา concurrency ได้ครบทุกแบบ | ต้องเขียนโค้ด retry เสมอ เพราะ transaction อาจถูกยกเลิกโดยไม่มีการ deadlock จริง |
| ไม่ต้องคิด locking strategy เอง (`SELECT ... FOR UPDATE` ฯลฯ) | มี overhead ในการติดตาม read/write dependency (ใช้ CPU/memory เพิ่มขึ้น) |
| โค้ด business logic เขียนง่ายขึ้น (ไม่ต้องกังวล race condition) | Throughput อาจลดลงเมื่อมี transaction จำนวนมากแข่งกันเข้าถึงข้อมูลชุดเดียวกัน (high contention) |

---

## Step 378: จำลองปัญหา Overselling ด้วยสอง psql Session

มาถึงส่วนสำคัญที่สุดของบทนี้ — การจำลองปัญหา **overselling** (ขายสินค้าเกินสต๊อกที่มีจริง) ให้เห็นภาพชัดเจนที่สุด โดยใช้สินค้า `UltraBook 14" Laptop` (product_id = 3) ที่เหลือสต๊อกแค่ **3 ชิ้น**

### โจทย์: ลูกค้า 2 คนสั่งซื้อพร้อมกัน สต๊อกเหลือ 3 ชิ้น คนละ 2 ชิ้น

ระบบ e-commerce ทั่วไปมักเขียน logic แบบนี้ (ผิดพลาด):

```
1. SELECT stock_quantity ... (ตรวจสอบว่ามีของพอไหม)
2. ถ้ามีพอ (ที่ฝั่ง application) → UPDATE stock_quantity ... (ตัดสต๊อก)
3. INSERT ลง order_items
```

### ทดสอบด้วย READ COMMITTED (default) — เกิด overselling!

**เวลา T1 — Session A (ลูกค้า Somchai สั่งซื้อ 2 ชิ้น):**
```sql
BEGIN;  -- ใช้ READ COMMITTED (default)
SELECT stock_quantity FROM products WHERE product_id = 3 FOR KEY SHARE;
```
```
 stock_quantity
----------------
              3
(1 row)
```
*(แอปฝั่ง A เห็นว่ามี 3 ชิ้น มากกว่า 2 ที่ต้องการ → ตัดสินใจว่า "สั่งได้")*

**เวลา T2 — Session B (ลูกค้า Nattaya สั่งซื้อ 2 ชิ้นเช่นกัน ในเวลาไล่เลี่ยกัน):**
```sql
BEGIN;  -- ใช้ READ COMMITTED (default)
SELECT stock_quantity FROM products WHERE product_id = 3 FOR KEY SHARE;
```
```
 stock_quantity
----------------
              3
(1 row)
```
*(แอปฝั่ง B ก็เห็นว่ามี 3 ชิ้น เช่นกัน เพราะ A ยังไม่ update → ตัดสินใจว่า "สั่งได้" เหมือนกัน)*

**เวลา T3 — Session A (ตัดสต๊อกตามที่ตัดสินใจไว้):**
```sql
UPDATE products SET stock_quantity = stock_quantity - 2 WHERE product_id = 3;
INSERT INTO order_items (order_id, product_id, quantity, unit_price) VALUES (15, 3, 2, 28900.00);
COMMIT;
```
```
UPDATE 1
INSERT 0 1
COMMIT
```

**เวลา T4 — Session B (ตัดสต๊อกตามที่ตัดสินใจไว้ โดยไม่รู้ว่า A ตัดไปแล้ว):**
```sql
UPDATE products SET stock_quantity = stock_quantity - 2 WHERE product_id = 3;
INSERT INTO order_items (order_id, product_id, quantity, unit_price) VALUES (12, 3, 2, 28900.00);
COMMIT;
```
```
UPDATE 1
INSERT 0 1
COMMIT
```

ตรวจสอบผลลัพธ์:

```sql
SELECT stock_quantity FROM products WHERE product_id = 3;
```

```
 stock_quantity
----------------
             -1
(1 row)
```

**สต๊อกติดลบ!** ทั้งที่มีของจริงแค่ 3 ชิ้น แต่ขายไปแล้ว 4 ชิ้น (2+2) นี่คือปัญหา overselling ที่เกิดขึ้นจริงในระบบจำนวนมากที่ไม่ได้ป้องกัน race condition อย่างถูกต้อง — สาเหตุคือ `UPDATE ... SET stock_quantity = stock_quantity - 2` แม้จะเป็น atomic operation ในระดับ row (ไม่มีใคร "แซง" ระหว่างอ่าน-เขียนในบรรทัดเดียวกันนี้ได้) แต่ **การตัดสินใจ "มีของพอไหม" เกิดขึ้นที่ฝั่ง application ก่อนหน้านั้น** โดยอิงจากค่าที่อ่านมาซึ่งอาจไม่ทันสมัยแล้ว (`SELECT ... FOR KEY SHARE` ในตัวอย่างนี้ไม่ได้ช่วยป้องกัน เพราะ shared lock ยอมให้อีก transaction อ่านพร้อมกันได้)

### แก้ไขวิธีที่ 1: ใช้ `SELECT ... FOR UPDATE` (row-level lock)

`FOR UPDATE` จะล็อกแถวที่อ่านแบบ exclusive lock ทำให้ transaction อื่นที่พยายาม `SELECT ... FOR UPDATE` แถวเดียวกันต้อง**รอ** จนกว่า transaction แรกจะ commit หรือ rollback

**เวลา T1 — Session A:**
```sql
BEGIN;
SELECT stock_quantity FROM products WHERE product_id = 3 FOR UPDATE;
```
```
 stock_quantity
----------------
              3
(1 row)
```

**เวลา T2 — Session B (พยายามอ่านแถวเดียวกันด้วย FOR UPDATE):**
```sql
BEGIN;
SELECT stock_quantity FROM products WHERE product_id = 3 FOR UPDATE;
```

Session B **ถูกบล็อกทันที** (query ค้างอยู่ ไม่คืนผลลัพธ์) เพราะ Session A ถือ lock อยู่

**เวลา T3 — Session A (ตรวจสอบสต๊อกพอ ตัดสต๊อก และ commit):**
```sql
-- แอปตรวจสอบ: stock_quantity (3) >= quantity ที่ต้องการ (2) → ดำเนินการต่อ
UPDATE products SET stock_quantity = stock_quantity - 2 WHERE product_id = 3;
INSERT INTO order_items (order_id, product_id, quantity, unit_price) VALUES (15, 3, 2, 28900.00);
COMMIT;
```
```
UPDATE 1
INSERT 0 1
COMMIT
```

**เวลา T4 — Session B (ทันทีที่ A commit, B ที่ถูกบล็อกไว้จะได้ lock และเห็นค่าล่าสุด):**
```sql
-- query ที่ค้างไว้ตั้งแต่ T2 จะทำงานต่อทันทีหลัง A commit
```
```
 stock_quantity
----------------
              1
(1 row)
```

Session B เห็นค่าล่าสุด (เหลือ 1 ชิ้น) ไม่ใช่ค่าเก่า (3) เพราะ `FOR UPDATE` บังคับให้รอจน lock ว่าง แล้วอ่านค่าล่าสุดหลัง commit

```sql
-- แอปตรวจสอบ: stock_quantity (1) < quantity ที่ต้องการ (2) → ปฏิเสธคำสั่งซื้อ
ROLLBACK;
```

```sql
SELECT stock_quantity FROM products WHERE product_id = 3;
```
```
 stock_quantity
----------------
              1
(1 row)
```

สต๊อกถูกต้อง ไม่ติดลบ! เพราะ Session B ถูกบังคับให้อ่านค่าล่าสุดก่อนตัดสินใจ

### แก้ไขวิธีที่ 2: ใช้ atomic `UPDATE ... WHERE ... RETURNING` โดยตรง (แนะนำที่สุด)

วิธีที่ดีที่สุดและมีประสิทธิภาพสูงสุด คือให้ database เป็นผู้ตรวจสอบเงื่อนไขและตัดสต๊อกในคำสั่งเดียว โดยไม่ต้องแยก `SELECT` แล้วค่อย `UPDATE`:

```sql
BEGIN;

UPDATE products
SET stock_quantity = stock_quantity - 2
WHERE product_id = 3
  AND stock_quantity >= 2
RETURNING stock_quantity;
```

ถ้ามีของพอ จะได้ผลลัพธ์:
```
 stock_quantity
----------------
              1
(1 row)

UPDATE 1
```

ถ้าของไม่พอ (เช่นอีก session ตัดไปก่อนแล้ว) จะได้:
```
(0 rows)

UPDATE 0
```

โค้ดฝั่ง application ตรวจสอบง่าย ๆ ว่า `UPDATE 0` แปลว่าของไม่พอ ให้ `ROLLBACK` และแจ้งลูกค้า — วิธีนี้**ไม่ต้องใช้ explicit lock เลย** เพราะ `WHERE stock_quantity >= 2` ผนวกกับ atomic UPDATE ของ PostgreSQL (ที่ใช้ row-level lock ภายในโดยอัตโนมัติระหว่างประมวลผลคำสั่ง) รับประกันความถูกต้องอยู่แล้ว และมีประสิทธิภาพดีกว่าการ lock ด้วย `FOR UPDATE` แล้วรอ

```sql
COMMIT;
```

### สรุปเปรียบเทียบ

| วิธี | ข้อดี | ข้อเสีย |
|---|---|---|
| `SELECT` ธรรมดา + `UPDATE` (ไม่ล็อก) | เขียนง่าย | เกิด overselling ได้ (ตามที่พิสูจน์แล้ว) |
| `SELECT ... FOR UPDATE` + ตรวจสอบเอง + `UPDATE` | ป้องกันได้ ควบคุม logic ได้ละเอียด | ต้อง lock รอคิว อาจช้าเมื่อมีการแข่งขันสูง |
| `UPDATE ... WHERE stock_quantity >= N RETURNING` | ป้องกันได้ ประสิทธิภาพดีที่สุด โค้ดสั้น | ต้องออกแบบเงื่อนไขให้ atomic ในคำสั่งเดียว ไม่เหมาะกับ logic ที่ซับซ้อนมาก |
| `SERIALIZABLE` isolation | ป้องกันได้ครบทุกกรณี รวมถึง logic ซับซ้อน | ต้องมี retry logic เสมอ overhead สูงกว่า |

---

## Step 379: เลือก Isolation Level ที่เหมาะสม และ Retry Logic

การเลือก isolation level ไม่ใช่แค่ "ยิ่งสูงยิ่งดี" เสมอไป เพราะยิ่งเข้มงวดมาก ยิ่งมีโอกาสที่ transaction จะถูกบล็อกนานขึ้น หรือถูกยกเลิกบ่อยขึ้น (โดยเฉพาะ `SERIALIZABLE`) — การเลือกที่ดีต้องพิจารณาจากลักษณะงานจริง

### แนวทางการเลือก

| สถานการณ์ | Isolation Level ที่แนะนำ | เหตุผล |
|---|---|---|
| อ่านรายงาน, dashboard ทั่วไป | READ COMMITTED (default) | ไม่ critical มาก รับ non-repeatable read ได้ ประสิทธิภาพดีสุด |
| แสดงข้อมูลตะกร้าสินค้า, สรุปยอด ณ ช่วงเวลาหนึ่ง | REPEATABLE READ | ต้องการความสอดคล้องของข้อมูลตลอด transaction (snapshot คงที่) |
| ตัดสต๊อกสินค้า, โอนเงิน, ธุรกรรมการเงิน | `UPDATE ... WHERE ... RETURNING` (atomic) หรือ `SELECT ... FOR UPDATE` ร่วมกับ READ COMMITTED | เร็ว ป้องกัน race condition เฉพาะจุดได้แม่นยำ ไม่ต้อง retry บ่อย |
| Business logic ซับซ้อนหลายตารางที่ต้องรับประกันความถูกต้องแบบ 100% (เช่น การจองที่นั่ง, ระบบบัญชีที่มีกฎซับซ้อน) | SERIALIZABLE | ยอมแลก throughput เพื่อความถูกต้องสูงสุด และให้ database จัดการ conflict แทนการเขียน lock เอง |
| Batch job / ETL ที่รันตัวเดียวไม่มีใครแข่ง | READ COMMITTED | ไม่มี concurrency ให้กังวล ไม่จำเป็นต้องเข้มงวด |

### กฎง่าย ๆ

1. เริ่มจาก `READ COMMITTED` (default) เสมอ เว้นแต่มีเหตุผลชัดเจนที่ต้องการมากกว่านั้น
2. ถ้าต้องการความสอดคล้องของข้อมูลตลอด transaction (อ่านหลายครั้งต้องได้ค่าเดิม) ให้ใช้ `REPEATABLE READ`
3. ถ้า business logic ซับซ้อนจนไม่สามารถ "ล็อกแถวที่ถูกต้องได้ครบ" ด้วยมือ (เช่นต้องพิจารณาหลายตารางที่สัมพันธ์กันแบบซับซ้อน) ให้ใช้ `SERIALIZABLE` แล้วปล่อยให้ database ตรวจจับ conflict แทน
4. ไม่ว่าจะใช้ `REPEATABLE READ` หรือ `SERIALIZABLE` **ต้องเขียน retry logic เสมอ** เพราะทั้งสอง level มีโอกาสโยน error `serialization_failure` (40001) ได้

### เขียน Retry Logic ด้วย PL/pgSQL

ตัวอย่างฟังก์ชันที่ตัดสต๊อกสินค้าด้วย `SERIALIZABLE` พร้อม retry logic ในตัว:

```sql
CREATE OR REPLACE FUNCTION purchase_product(
    p_product_id  INTEGER,
    p_quantity    INTEGER,
    p_order_id    INTEGER
) RETURNS BOOLEAN
LANGUAGE plpgsql
AS $$
DECLARE
    v_retry_count   INTEGER := 0;
    v_max_retries   INTEGER := 5;
    v_success       BOOLEAN := false;
    v_updated_rows  INTEGER;
BEGIN
    WHILE v_retry_count < v_max_retries AND NOT v_success LOOP
        BEGIN
            -- ตัดสต๊อกแบบ atomic พร้อมตรวจสอบเงื่อนไขในคำสั่งเดียว
            UPDATE products
            SET stock_quantity = stock_quantity - p_quantity
            WHERE product_id = p_product_id
              AND stock_quantity >= p_quantity;

            GET DIAGNOSTICS v_updated_rows = ROW_COUNT;

            IF v_updated_rows = 0 THEN
                RAISE EXCEPTION 'สต๊อกไม่พอสำหรับ product_id = %', p_product_id
                    USING ERRCODE = 'P0001';
            END IF;

            INSERT INTO order_items (order_id, product_id, quantity, unit_price)
            SELECT p_order_id, p_product_id, p_quantity, unit_price
            FROM products WHERE product_id = p_product_id;

            v_success := true;

        EXCEPTION
            WHEN sqlstate '40001' THEN
                -- serialization_failure: ลองใหม่หลังหน่วงเวลาสั้น ๆ (exponential backoff)
                v_retry_count := v_retry_count + 1;
                RAISE NOTICE 'พบ serialization failure ครั้งที่ % กำลังลองใหม่...', v_retry_count;
                PERFORM pg_sleep(0.05 * v_retry_count);
            WHEN sqlstate 'P0001' THEN
                -- สต๊อกไม่พอจริง ไม่ต้อง retry
                RAISE NOTICE 'สต๊อกไม่พอ ยกเลิกคำสั่งซื้อ';
                RETURN false;
        END;
    END LOOP;

    IF NOT v_success THEN
        RAISE EXCEPTION 'ล้มเหลวหลังจากลองซ้ำ % ครั้ง เนื่องจาก serialization conflict สูงเกินไป', v_max_retries;
    END IF;

    RETURN v_success;
END;
$$;
```

ทดสอบใช้งาน:

```sql
SELECT purchase_product(p_product_id => 3, p_quantity => 1, p_order_id => 15);
```

```
 purchase_product
-------------------
 t
(1 row)
```

> **หมายเหตุสำคัญ:** ในตัวอย่างนี้ `EXCEPTION` block ภายใน PL/pgSQL function จะสร้าง subtransaction (savepoint) โดยอัตโนมัติ ซึ่งมี overhead เล็กน้อย สำหรับ transaction ที่ concurrency สูงมาก บางทีมเลือกเขียน retry logic ไว้ที่**ฝั่ง application** แทน (เช่น ใน Python ด้วย `try/except` ดัก exception ที่มี `pgcode == '40001'` แล้ว retry ทั้ง transaction ใหม่ตั้งแต่ `BEGIN`) เพราะยืดหยุ่นกว่าและไม่ผูก retry logic ไว้กับ subtransaction เดียว

### รูปแบบ retry logic ฝั่ง application (แนวคิด)

แม้บทนี้เน้น SQL แต่ควรเข้าใจแนวคิดว่าฝั่ง application มักเขียนแบบนี้ (pseudo-code):

```
for attempt in 1..max_retries:
    try:
        BEGIN transaction with isolation level SERIALIZABLE
        ... ทำงานตาม business logic ...
        COMMIT
        break  -- สำเร็จ ออกจาก loop
    except SerializationFailure (SQLSTATE 40001):
        ROLLBACK
        wait (exponential backoff + jitter)
        continue  -- ลองใหม่
    except อื่น ๆ:
        ROLLBACK
        raise  -- error จริง ไม่ retry
```

หลักการสำคัญคือ: **ห่อทั้ง transaction ตั้งแต่ `BEGIN` ถึง `COMMIT` ไว้ใน retry loop เดียว** ไม่ใช่แค่คำสั่งเดียว เพราะ serialization failure อาจเกิดตอน `COMMIT` (ไม่ใช่ตอนรันคำสั่งใดคำสั่งหนึ่ง) ดังที่เราเห็นใน Step 377

---

## Step 380: แบบฝึกหัดรวม — ทดสอบ Isolation Level กับสถานการณ์ Overselling

ก่อนไปถึงแบบฝึกหัดท้ายบท มาสรุปขั้นตอนการทดสอบ isolation level อย่างเป็นระบบ เพื่อให้ผู้เรียนนำไปประยุกต์ใช้ได้เองกับสถานการณ์จริงในงาน

### Checklist สำหรับทดสอบ concurrency

เมื่อต้องออกแบบระบบที่มีความเสี่ยงต่อ race condition (เช่น ตัดสต๊อก, โอนเงิน, จองคิว) ให้ทำตามขั้นตอนนี้:

1. **ระบุจุดเสี่ยง**: หาคำสั่งที่มีรูปแบบ "อ่านค่า → ตัดสินใจที่แอป → เขียนค่ากลับ" (read-then-write) เพราะเป็นจุดที่เกิด race condition ได้ง่ายที่สุด
2. **จำลองด้วยสอง session**: เปิด psql สอง terminal รันคำสั่งสลับกันตามลำดับเวลาที่คาดว่าจะเกิดในสถานการณ์จริง (เช่น worst case ที่สอง transaction ทับซ้อนกันพอดี)
3. **ทดสอบด้วย isolation level ต่าง ๆ**: ลองทั้ง READ COMMITTED, REPEATABLE READ, SERIALIZABLE ดูว่าแบบไหนป้องกันปัญหาที่พบได้
4. **เลือกวิธีแก้ที่เหมาะสม**: ระหว่าง atomic UPDATE, `FOR UPDATE` lock, หรือ SERIALIZABLE + retry ตามที่สรุปไว้ใน Step 379
5. **เขียน automated test**: จำลอง concurrency ด้วยสคริปต์ (เช่น เปิดหลาย connection พร้อมกันด้วย library อย่าง `pgbench` หรือเขียน integration test ที่เปิดสอง connection แล้วสลับ synchronize จุดตัดสินใจ) เพื่อไม่ต้องทดสอบมือทุกครั้ง

### ตัวอย่างการใช้ `pgbench` จำลอง concurrency จริง (เสริม)

สำหรับการทดสอบโหลดจริงที่มีหลาย transaction แข่งกันพร้อมกันจำนวนมาก สามารถใช้เครื่องมือ `pgbench` ที่มากับ PostgreSQL:

```sql
-- สร้างไฟล์ทดสอบ purchase_test.sql
-- \set product_id 3
-- BEGIN;
-- UPDATE products SET stock_quantity = stock_quantity - 1
--   WHERE product_id = :product_id AND stock_quantity >= 1;
-- COMMIT;
```

```bash
# รันจำลอง 10 client พร้อมกัน ยิง transaction รวม 100 ครั้ง
pgbench -c 10 -t 10 -f purchase_test.sql -d ecommerce_course
```

หลังรันเสร็จ ตรวจสอบว่า `stock_quantity` ไม่ติดลบ:

```sql
SELECT product_id, stock_quantity FROM products WHERE product_id = 3;
```

ถ้าคำสั่ง `UPDATE` เขียนแบบ atomic (มี `WHERE stock_quantity >= 1` ในคำสั่งเดียว) ผลลัพธ์จะไม่มีทางติดลบ ไม่ว่าจะรันแข่งกันกี่ client ก็ตาม — นี่คือวิธีตรวจสอบเชิงประจักษ์ (empirical validation) ว่าโค้ดที่เขียนปลอดภัยจาก race condition จริง ไม่ใช่แค่ "เดา" จากการอ่านโค้ด

### สรุปภาพรวมทั้งบท

ตลอดบทนี้เราได้เห็นว่า:

- ปัญหา concurrency เกิดจาก transaction หลายตัวเข้าถึงข้อมูลชุดเดียวกันพร้อมกัน
- PostgreSQL ป้องกัน **dirty read** ได้เสมอ ไม่ว่า isolation level ใด (ด้วยกลไก MVCC)
- **READ COMMITTED** (default) ยังเสี่ยงต่อ non-repeatable read และ phantom read
- **REPEATABLE READ** ป้องกันทั้ง non-repeatable read และ phantom read ได้ (เข้มงวดกว่ามาตรฐาน SQL) แต่ยังเสี่ยงต่อ write skew / lost update ในบาง pattern
- **SERIALIZABLE** ป้องกันได้ครบทุกปัญหา แต่ต้องมี retry logic เสมอ
- การแก้ปัญหา overselling ที่ efficient และ practical ที่สุดในงานส่วนใหญ่ คือการเขียน atomic `UPDATE ... WHERE condition RETURNING` แทนการแยก `SELECT` แล้วค่อย `UPDATE`

---

## สรุปท้ายบท

### ตารางสรุป Isolation Level กับปัญหาที่ป้องกันได้ (พฤติกรรมจริงใน PostgreSQL)

| Isolation Level | Dirty Read | Non-repeatable Read | Phantom Read | Lost Update / Write Skew | Serialization Anomaly |
|---|---|---|---|---|---|
| READ UNCOMMITTED | ป้องกันได้ (เหมือน READ COMMITTED) | อาจเกิด | อาจเกิด | อาจเกิด | อาจเกิด |
| READ COMMITTED (default) | ป้องกันได้ | อาจเกิด | อาจเกิด | อาจเกิด | อาจเกิด |
| REPEATABLE READ | ป้องกันได้ | ป้องกันได้ | **ป้องกันได้** (เข้มงวดกว่ามาตรฐาน) | อาจเกิด (เช่น write skew) | อาจเกิด |
| SERIALIZABLE | ป้องกันได้ | ป้องกันได้ | ป้องกันได้ | ป้องกันได้ | ป้องกันได้ |

### คำสั่งสำคัญที่ต้องจำ

```sql
-- ตรวจสอบ isolation level ปัจจุบัน
SHOW transaction_isolation;

-- ตั้ง isolation level แบบที่ 1
BEGIN;
SET TRANSACTION ISOLATION LEVEL {READ COMMITTED | REPEATABLE READ | SERIALIZABLE};

-- ตั้ง isolation level แบบที่ 2
BEGIN ISOLATION LEVEL SERIALIZABLE;

-- ล็อกแถวแบบ exclusive เพื่อป้องกัน race condition
SELECT ... FROM table_name WHERE ... FOR UPDATE;

-- atomic update พร้อมตรวจสอบเงื่อนไข (วิธีที่แนะนำที่สุดสำหรับงานทั่วไป)
UPDATE table_name SET col = col - N WHERE id = ? AND col >= N RETURNING col;

-- ดัก serialization failure (SQLSTATE 40001) ใน PL/pgSQL
EXCEPTION WHEN sqlstate '40001' THEN ...
```

### สิ่งที่ต้องจำให้ขึ้นใจ

1. **PostgreSQL ไม่มี dirty read เลย** ไม่ว่าจะตั้ง isolation level เป็นอะไร (แม้แต่ READ UNCOMMITTED)
2. **READ COMMITTED คือค่า default** และเหมาะกับงานส่วนใหญ่ที่ไม่ critical มาก
3. **REPEATABLE READ ของ PostgreSQL เข้มงวดกว่ามาตรฐาน SQL** เพราะป้องกัน phantom read ได้ด้วย
4. **SERIALIZABLE ป้องกันได้ครบทุกปัญหา แต่ต้องมี retry logic เสมอ** เพราะอาจโยน `serialization_failure` (40001) ได้ตลอดเวลา แม้กระทั่งตอน `COMMIT`
5. สำหรับปัญหา overselling ทั่วไป วิธีที่ **เร็วและปลอดภัยที่สุด** คือ atomic `UPDATE ... WHERE condition` ไม่ใช่การเพิ่ม isolation level ให้สูงขึ้นเสมอไป — isolation level สูงมีไว้แก้ปัญหาที่ atomic UPDATE เดี่ยว ๆ แก้ไม่ได้ (เช่น business logic ที่ซับซ้อนข้ามหลายตาราง)

---

## แบบฝึกหัด

<details>
<summary><strong>แบบฝึกหัดที่ 1:</strong> ตรวจสอบ isolation level ปัจจุบันของ session และเปลี่ยนเป็น REPEATABLE READ แล้วยืนยันด้วย SHOW</summary>

```sql
BEGIN;
SHOW transaction_isolation;
-- read committed

SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SHOW transaction_isolation;
-- repeatable read

COMMIT;
```

**อธิบาย:** ค่า default ของ PostgreSQL คือ `read committed` เสมอ เว้นแต่จะมีการตั้งค่า `default_transaction_isolation` ไว้ล่วงหน้า การใช้ `SET TRANSACTION ISOLATION LEVEL` ต้องอยู่ในช่วงต้นของ transaction ก่อนมี query ใด ๆ
</details>

<details>
<summary><strong>แบบฝึกหัดที่ 2:</strong> จำลอง dirty read scenario ระหว่าง 2 session แล้วพิสูจน์ว่า PostgreSQL ป้องกันได้ในทุก isolation level รวมถึง READ UNCOMMITTED</summary>

**Session A:**
```sql
BEGIN;
UPDATE customers SET country = 'TEST_DIRTY' WHERE customer_id = 1;
-- ยังไม่ COMMIT
```

**Session B:**
```sql
BEGIN;
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
SELECT country FROM customers WHERE customer_id = 1;
-- ผลลัพธ์: Thailand (ค่าเดิม ไม่ใช่ TEST_DIRTY)
COMMIT;
```

**Session A:**
```sql
ROLLBACK;  -- ยกเลิกการเปลี่ยนแปลง
```

**อธิบาย:** แม้ Session B จะตั้ง isolation level เป็น READ UNCOMMITTED อย่างชัดเจน ก็ยังไม่เห็นค่าที่ A ยังไม่ commit เพราะ PostgreSQL implement READ UNCOMMITTED ให้ทำงานเหมือน READ COMMITTED เสมอ (ป้องกัน dirty read ได้ 100% ในทุกกรณี)
</details>

<details>
<summary><strong>แบบฝึกหัดที่ 3:</strong> เขียนสถานการณ์ non-repeatable read กับตาราง orders (คอลัมน์ status) แล้วแก้ด้วย REPEATABLE READ</summary>

**Session A (READ COMMITTED, default):**
```sql
BEGIN;
SELECT status FROM orders WHERE order_id = 6;
-- processing
```

**Session B:**
```sql
BEGIN;
UPDATE orders SET status = 'shipped' WHERE order_id = 6;
COMMIT;
```

**Session A (query ซ้ำในทรานแซคชันเดิม):**
```sql
SELECT status FROM orders WHERE order_id = 6;
-- shipped  (ค่าเปลี่ยน = non-repeatable read)
COMMIT;
```

**แก้ไขด้วย REPEATABLE READ — Session A:**
```sql
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT status FROM orders WHERE order_id = 6;
-- shipped (สมมติค่าปัจจุบันหลังการทดสอบก่อนหน้า)
```

**Session B:**
```sql
BEGIN;
UPDATE orders SET status = 'delivered' WHERE order_id = 6;
COMMIT;
```

**Session A (query ซ้ำ):**
```sql
SELECT status FROM orders WHERE order_id = 6;
-- shipped (ค่าเดิม ไม่เปลี่ยน เพราะ snapshot คงที่)
COMMIT;
```
</details>

<details>
<summary><strong>แบบฝึกหัดที่ 4:</strong> จำลอง phantom read กับตาราง reviews (นับจำนวนรีวิวที่ rating >= 4 ของสินค้าหนึ่ง) แล้วพิสูจน์ว่า REPEATABLE READ ป้องกันได้</summary>

**Session A:**
```sql
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT COUNT(*) FROM reviews WHERE product_id = 18 AND rating >= 4;
-- count = 1 (มีรีวิว review_id = 8 ที่ rating 5)
```

**Session B:**
```sql
BEGIN;
INSERT INTO reviews (product_id, customer_id, rating, review_text)
VALUES (18, 4, 5, 'สินค้าคุณภาพดีมาก');
COMMIT;
```

**Session A (นับซ้ำในทรานแซคชันเดิม):**
```sql
SELECT COUNT(*) FROM reviews WHERE product_id = 18 AND rating >= 4;
-- count = 1 (ยังเป็น 1 เหมือนเดิม ไม่เห็น phantom row)
COMMIT;

-- transaction ใหม่จะเห็นค่าล่าสุด
SELECT COUNT(*) FROM reviews WHERE product_id = 18 AND rating >= 4;
-- count = 2
```
</details>

<details>
<summary><strong>แบบฝึกหัดที่ 5:</strong> เขียน UPDATE แบบ atomic เพื่อตัดสต๊อกสินค้า product_id = 18 (Bluetooth Earbuds Air) จำนวน 2 ชิ้น พร้อมตรวจสอบว่ามีของพอหรือไม่ในคำสั่งเดียว</summary>

```sql
BEGIN;

UPDATE products
SET stock_quantity = stock_quantity - 2
WHERE product_id = 18
  AND stock_quantity >= 2
RETURNING stock_quantity;

-- ถ้าได้ 1 แถวกลับมา แปลว่าตัดสำเร็จ ให้ INSERT order_items ต่อ แล้ว COMMIT
-- ถ้าได้ 0 แถว แปลว่าของไม่พอ ให้ ROLLBACK

COMMIT;
```

**อธิบาย:** วิธีนี้ปลอดภัยจาก race condition โดยไม่ต้องใช้ `FOR UPDATE` หรือ isolation level สูง ๆ เพราะเงื่อนไข `stock_quantity >= 2` ถูกตรวจสอบพร้อมกับการเขียนในคำสั่งเดียว ซึ่ง PostgreSQL รับประกันความเป็น atomic operation ของคำสั่ง `UPDATE` เดี่ยว ๆ อยู่แล้ว
</details>

<details>
<summary><strong>แบบฝึกหัดที่ 6:</strong> จำลองสถานการณ์ overselling ด้วย 2 session บนสินค้า product_id = 15 (Camping Tent, stock = 12) ที่ลูกค้า 2 คนสั่งครั้งละ 7 ชิ้นพร้อมกัน โดยใช้ READ COMMITTED แบบ read-then-write (ไม่ป้องกัน) แล้วสังเกตผลลัพธ์</summary>

**Session A:**
```sql
BEGIN;
SELECT stock_quantity FROM products WHERE product_id = 15;
-- 12 (มากกว่า 7 ที่ต้องการ → ตัดสินใจว่าสั่งได้)
```

**Session B:**
```sql
BEGIN;
SELECT stock_quantity FROM products WHERE product_id = 15;
-- 12 (มากกว่า 7 ที่ต้องการ → ตัดสินใจว่าสั่งได้เช่นกัน)
```

**Session A:**
```sql
UPDATE products SET stock_quantity = stock_quantity - 7 WHERE product_id = 15;
COMMIT;
```

**Session B:**
```sql
UPDATE products SET stock_quantity = stock_quantity - 7 WHERE product_id = 15;
COMMIT;
```

**ตรวจสอบผลลัพธ์:**
```sql
SELECT stock_quantity FROM products WHERE product_id = 15;
-- -2  (ติดลบ! ขายไป 14 ชิ้น ทั้งที่มีแค่ 12 ชิ้น)
```

**อธิบาย:** นี่คือ overselling ที่เกิดจากรูปแบบ read-then-write โดยไม่มีการป้องกัน race condition ใด ๆ — แก้ไขได้ด้วยวิธีจากแบบฝึกหัดที่ 5 หรือ `SELECT ... FOR UPDATE`

*(หมายเหตุ: ก่อนรันแบบฝึกหัดนี้ ควร reset ค่า `stock_quantity` ของ product_id = 15 กลับเป็น 12 ก่อน ด้วย `UPDATE products SET stock_quantity = 12 WHERE product_id = 15;`)*
</details>

<details>
<summary><strong>แบบฝึกหัดที่ 7:</strong> แก้ไขแบบฝึกหัดที่ 6 ด้วย SELECT ... FOR UPDATE แล้วพิสูจน์ว่าไม่เกิด overselling อีก</summary>

*(reset ค่าก่อน: `UPDATE products SET stock_quantity = 12 WHERE product_id = 15;`)*

**Session A:**
```sql
BEGIN;
SELECT stock_quantity FROM products WHERE product_id = 15 FOR UPDATE;
-- 12
```

**Session B:**
```sql
BEGIN;
SELECT stock_quantity FROM products WHERE product_id = 15 FOR UPDATE;
-- ถูกบล็อก รอ Session A commit หรือ rollback ก่อน
```

**Session A:**
```sql
-- ตรวจสอบ 12 >= 7 → พอ ตัดสต๊อก
UPDATE products SET stock_quantity = stock_quantity - 7 WHERE product_id = 15;
COMMIT;
```

**Session B (query ที่ค้างไว้ทำงานต่อทันที):**
```sql
-- ได้ค่าล่าสุด = 5
-- ตรวจสอบ 5 >= 7 → ไม่พอ!
ROLLBACK;
```

**ตรวจสอบผลลัพธ์:**
```sql
SELECT stock_quantity FROM products WHERE product_id = 15;
-- 5 (ถูกต้อง ไม่ติดลบ)
```
</details>

<details>
<summary><strong>แบบฝึกหัดที่ 8:</strong> ทดสอบ SERIALIZABLE ระหว่าง 2 session ที่ทั้งคู่อ่านและเขียนแถวเดียวกัน (product_id = 3) แล้วสังเกต error ที่เกิดขึ้นตอน COMMIT</summary>

*(reset ค่าก่อน: `UPDATE products SET stock_quantity = 3 WHERE product_id = 3;`)*

**Session A:**
```sql
BEGIN ISOLATION LEVEL SERIALIZABLE;
SELECT stock_quantity FROM products WHERE product_id = 3;
-- 3
```

**Session B:**
```sql
BEGIN ISOLATION LEVEL SERIALIZABLE;
SELECT stock_quantity FROM products WHERE product_id = 3;
-- 3
```

**Session A:**
```sql
UPDATE products SET stock_quantity = stock_quantity - 1 WHERE product_id = 3;
COMMIT;
-- สำเร็จ
```

**Session B:**
```sql
UPDATE products SET stock_quantity = stock_quantity - 1 WHERE product_id = 3;
COMMIT;
```

```
ERROR:  could not serialize access due to read/write dependencies among transactions
```

**อธิบาย:** Session B ต้อง `ROLLBACK` แล้วเริ่ม transaction ใหม่ (retry) เพื่ออ่านค่าล่าสุดและตัดสินใจใหม่
</details>

<details>
<summary><strong>แบบฝึกหัดที่ 9:</strong> เขียนฟังก์ชัน PL/pgSQL ที่รับ order_id และยกเลิกคำสั่งซื้อ (เปลี่ยน status เป็น 'cancelled') พร้อมคืนสต๊อกสินค้าทุกชิ้นในออร์เดอร์นั้น โดยป้องกัน race condition ด้วย SERIALIZABLE และ retry logic</summary>

```sql
CREATE OR REPLACE FUNCTION cancel_order(p_order_id INTEGER)
RETURNS BOOLEAN
LANGUAGE plpgsql
AS $$
DECLARE
    v_retry_count  INTEGER := 0;
    v_max_retries  INTEGER := 5;
    v_success      BOOLEAN := false;
BEGIN
    WHILE v_retry_count < v_max_retries AND NOT v_success LOOP
        BEGIN
            -- คืนสต๊อกสินค้าทุกชิ้นในออร์เดอร์นี้
            UPDATE products p
            SET stock_quantity = p.stock_quantity + oi.quantity
            FROM order_items oi
            WHERE oi.order_id = p_order_id
              AND oi.product_id = p.product_id;

            -- เปลี่ยนสถานะออร์เดอร์
            UPDATE orders
            SET status = 'cancelled'
            WHERE order_id = p_order_id
              AND status <> 'cancelled';

            v_success := true;
        EXCEPTION
            WHEN sqlstate '40001' THEN
                v_retry_count := v_retry_count + 1;
                PERFORM pg_sleep(0.05 * v_retry_count);
        END;
    END LOOP;

    IF NOT v_success THEN
        RAISE EXCEPTION 'ยกเลิกออร์เดอร์ % ไม่สำเร็จหลังจากลองซ้ำ % ครั้ง', p_order_id, v_max_retries;
    END IF;

    RETURN v_success;
END;
$$;

-- ทดสอบใช้งาน (ต้องรันใน transaction ที่มี isolation level เป็น SERIALIZABLE
-- หรือฟังก์ชันสามารถตั้ง isolation level ภายในเองผ่าน SET TRANSACTION ก็ได้
-- ขึ้นกับว่าต้องการให้ transaction ทั้งหมดที่เรียกฟังก์ชันนี้ serializable หรือไม่)
SELECT cancel_order(6);
```
</details>

<details>
<summary><strong>แบบฝึกหัดที่ 10:</strong> เขียนสรุปเปรียบเทียบ (ในรูปแบบ comment SQL หรือคำอธิบาย) ว่าถ้าคุณต้องออกแบบระบบจองที่นั่งคอนเสิร์ต (ที่นั่งจำกัด ห้ามจองซ้ำ) ควรเลือก isolation level และเทคนิคใด พร้อมให้เหตุผล</summary>

```sql
-- แนวทางที่แนะนำสำหรับระบบจองที่นั่งคอนเสิร์ต:
--
-- 1. ใช้ atomic UPDATE พร้อมเงื่อนไขตรวจสอบสถานะที่นั่งในคำสั่งเดียว
--    เช่น: UPDATE seats SET status = 'booked', customer_id = :cid
--          WHERE seat_id = :sid AND status = 'available'
--          RETURNING seat_id;
--    ถ้า RETURNING ไม่มีแถว แปลว่าที่นั่งถูกจองไปแล้ว
--
-- 2. isolation level: ใช้ READ COMMITTED (default) ร่วมกับ atomic UPDATE ข้างต้น
--    ก็เพียงพอสำหรับการจองที่นั่งเดี่ยว ๆ เพราะ UPDATE เดียวเป็น atomic อยู่แล้ว
--
-- 3. ถ้าต้องจองหลายที่นั่งพร้อมกันเป็น "แพ็กเกจ" (all-or-nothing)
--    เช่น ต้องได้ที่นั่งติดกัน 4 ที่ทั้งหมด หรือไม่ได้เลย
--    ควรใช้ SERIALIZABLE เพราะ logic ซับซ้อนข้ามหลายแถว
--    ยากที่จะ lock ให้ครบถ้วนถูกต้องด้วยมือ (ล็อกผิดแถว/ผิดลำดับอาจเกิด deadlock)
--    แล้วปล่อยให้ database ตรวจจับ conflict พร้อมเขียน retry logic รองรับ
--
-- 4. ควรมี unique constraint เสริมเป็นด่านสุดท้าย
--    เช่น UNIQUE (seat_id) WHERE status = 'booked' (partial unique index)
--    เพื่อป้องกันข้อผิดพลาดจาก logic ระดับ application ที่อาจมี bug หลุดมา
--
-- สรุป: ระบบจองที่นั่งเดี่ยวใช้ atomic UPDATE + READ COMMITTED พอ
--       ระบบจองแบบแพ็กเกจ/หลายเงื่อนไขซับซ้อนใช้ SERIALIZABLE + retry logic
--       ทั้งสองกรณีควรมี unique constraint เป็นเกราะป้องกันชั้นสุดท้ายเสมอ
```
</details>

---

**บทถัดไป:** [Part 039 — Sequences และ Identity Columns](./part-039-sequences-identity.md)
