# Part 024: Subqueries — Scalar, Correlated, EXISTS/NOT EXISTS

หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับกลาง | Part 024

---

## เป้าหมายการเรียนรู้

หลังจากจบบทนี้ ผู้เรียนจะสามารถ:

- อธิบายได้ว่า subquery คืออะไร และมีกี่ประเภท (scalar, row, table)
- เขียน scalar subquery ใน SELECT list เพื่อเปรียบเทียบค่าของแต่ละแถวกับค่าสรุปภาพรวม
- ใช้ subquery ร่วมกับ operator เปรียบเทียบ (`=`, `>`, `<`) ใน `WHERE`
- ใช้ `IN` / `NOT IN` กับ subquery ได้อย่างถูกต้อง และรู้จักข้อควรระวังเรื่อง `NULL` ใน `NOT IN`
- เขียน derived table (subquery ใน `FROM`) พร้อม alias ที่ PostgreSQL บังคับให้ใส่
- แยกแยะ correlated subquery กับ non-correlated subquery และรู้ว่าตัวไหนถูกประมวลผลกี่ครั้ง
- ใช้ `EXISTS` / `NOT EXISTS` แทน `IN` / `NOT IN` เมื่อเหมาะสม และอธิบายได้ว่าทำไมมักเร็วกว่าและปลอดภัยกว่าเรื่อง `NULL`
- ใช้ `ANY` / `SOME` และ `ALL` ร่วมกับ subquery
- ตัดสินใจได้ว่าเมื่อไหร่ควรใช้ subquery เมื่อไหร่ควรใช้ `JOIN`
- แก้โจทย์วิเคราะห์ข้อมูล (analytics) จริงในระบบ e-commerce โดยผสมผสานเทคนิค subquery หลายแบบ

---

## เตรียมข้อมูล

บทนี้ใช้ชุดข้อมูลร้านค้าออนไลน์ (e-commerce) ชุดเดียวกับ Part 021–039 ทั้งหมด ประกอบด้วย 9 ตาราง: `categories`, `suppliers`, `products`, `customers`, `employees`, `orders`, `order_items`, `reviews`, `payments`

ถ้าเคยสร้างตารางชุดนี้ไว้แล้วจากบทก่อนหน้า สามารถข้ามไปยัง Step 231 ได้เลย แต่ถ้าเริ่มต้นใหม่ ให้รันสคริปต์ทั้งหมดด้านล่างตามลำดับ (ลำดับสำคัญ เพราะมี foreign key อ้างอิงกัน)

### 1. สร้างตาราง (schema)

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
```

### 2. เพิ่มข้อมูลตัวอย่าง (seed data)

```sql
-- categories: มีทั้งหมวดหลักและหมวดย่อย (self-referencing parent_category_id)
INSERT INTO categories (category_id, category_name, parent_category_id) VALUES
    (1,  'Electronics',          NULL),
    (2,  'Fashion',              NULL),
    (3,  'Home & Living',        NULL),
    (4,  'Books',                NULL),
    (5,  'Sports & Outdoor',     NULL),
    (6,  'Smartphones',          1),
    (7,  'Computers & Laptops',  1),
    (8,  'Men''s Clothing',      2),
    (9,  'Women''s Clothing',    2),
    (10, 'Kitchenware',          3),
    (11, 'Beauty & Health',      NULL),
    (12, 'Toys & Games',         NULL);
SELECT setval('categories_category_id_seq', 12);

-- suppliers
INSERT INTO suppliers (supplier_id, supplier_name, country) VALUES
    (1, 'Thai Gadget Co., Ltd.',       'Thailand'),
    (2, 'Shenzhen ElecTrade',          'China'),
    (3, 'Nordic Home Supplies',        'Sweden'),
    (4, 'Bangkok Textile Union',       'Thailand'),
    (5, 'Global Book Distributors',    'USA'),
    (6, 'Sports Direct Asia',          'Thailand'),
    (7, 'Seoul Beauty Export',         'South Korea'),
    (8, 'Kyoto Kitchenware Co.',       'Japan');
SELECT setval('suppliers_supplier_id_seq', 8);

-- products (สังเกต product_id = 20 คือสินค้าที่เลิกขายแล้ว is_active = false, stock = 0)
INSERT INTO products (product_id, product_name, category_id, supplier_id, unit_price, stock_quantity, is_active) VALUES
    (1,  'iPhone 15 Pro 128GB',            6,  2, 42900.00, 25,  true),
    (2,  'Samsung Galaxy S24',             6,  2, 32900.00, 18,  true),
    (3,  'Xiaomi Redmi Note 13',           6,  1,  6990.00, 60,  true),
    (4,  'MacBook Air M3 13"',             7,  2, 45900.00, 10,  true),
    (5,  'Dell XPS 13',                    7,  2, 39900.00,  8,  true),
    (6,  'Lenovo ThinkPad E14',            7,  1, 24900.00, 15,  true),
    (7,  'Logitech MX Master 3S Mouse',    7,  1,  3290.00, 40,  true),
    (8,  'Men''s Cotton Polo Shirt',       8,  4,   590.00, 120, true),
    (9,  'Men''s Slim Fit Jeans',          8,  4,   890.00, 75,  true),
    (10, 'Women''s Summer Dress',          9,  4,   790.00, 90,  true),
    (11, 'Women''s Silk Blouse',           9,  4,  1290.00, 45,  true),
    (12, 'Nonstick Frying Pan 28cm',       10, 8,   690.00, 55,  true),
    (13, 'Japanese Rice Cooker 1.8L',      10, 8,  2590.00, 30,  true),
    (14, 'Stainless Steel Knife Set',      10, 8,  1990.00, 22,  true),
    (15, 'Clean Code (Book)',              4,  5,   890.00, 33,  true),
    (16, 'Atomic Habits (Book)',           4,  5,   450.00, 70,  true),
    (17, 'Yoga Mat Premium',               5,  6,   690.00, 65,  true),
    (18, 'Adjustable Dumbbell Set',        5,  6,  3990.00, 12,  true),
    (19, 'Korean Vitamin C Serum',         11, 7,   590.00, 80,  true),
    (20, 'Wooden Building Blocks Set',     12, 3,   990.00,  0,  false);
SELECT setval('products_product_id_seq', 20);

-- customers
INSERT INTO customers (customer_id, first_name, last_name, email, country, signup_date) VALUES
    (1,  'Somchai',    'Jaidee',      'somchai.j@example.com',    'Thailand',  '2023-01-15'),
    (2,  'Malee',      'Suksan',      'malee.s@example.com',      'Thailand',  '2023-02-20'),
    (3,  'Anan',       'Wongsa',      'anan.w@example.com',       'Thailand',  '2023-03-05'),
    (4,  'Pranee',     'Chaiyo',      'pranee.c@example.com',     'Thailand',  '2023-03-18'),
    (5,  'Weerawat',   'Boonmee',     'weerawat.b@example.com',   'Thailand',  '2023-04-02'),
    (6,  'Siriporn',   'Intharat',    'siriporn.i@example.com',   'Thailand',  '2023-04-25'),
    (7,  'John',       'Anderson',    'john.anderson@example.com','USA',       '2023-05-10'),
    (8,  'Emily',      'Clark',       'emily.clark@example.com',  'UK',        '2023-05-22'),
    (9,  'Yuki',       'Tanaka',      'yuki.tanaka@example.com',  'Japan',     '2023-06-01'),
    (10, 'Li',         'Wei',         'li.wei@example.com',       'China',     '2023-06-14'),
    (11, 'Kittipong',  'Rattana',     'kittipong.r@example.com',  'Thailand',  '2023-07-03'),
    (12, 'Nutthida',   'Saelim',      'nutthida.s@example.com',   'Thailand',  '2023-07-19'),
    (13, 'David',      'Miller',      'david.miller@example.com', 'Australia', '2023-08-08'),
    (14, 'Chanya',     'Phromma',     'chanya.p@example.com',     'Thailand',  '2023-09-01'),
    (15, 'Piti',       'Suwannaphum', 'piti.s@example.com',       'Thailand',  '2023-09-20');
SELECT setval('customers_customer_id_seq', 15);

-- employees (มีลำดับชั้นผู้บังคับบัญชาผ่าน manager_id)
INSERT INTO employees (employee_id, first_name, last_name, hire_date, manager_id, department) VALUES
    (1, 'Kanya',      'Srisuk',    '2021-01-10', NULL, 'Sales'),
    (2, 'Thawatchai',  'Pongsak',   '2021-03-15', 1,    'Sales'),
    (3, 'Areeya',      'Nantawong', '2021-06-01', 1,    'Sales'),
    (4, 'Somsak',      'Uraiwan',   '2022-01-20', NULL, 'Support'),
    (5, 'Nattaya',     'Chumphon',  '2022-02-14', 4,    'Support'),
    (6, 'Ekachai',     'Boonrod',   '2022-05-05', 4,    'Support'),
    (7, 'Panida',      'Thepwong',  '2022-08-11', 1,    'Sales'),
    (8, 'Rungroj',     'Sombat',    '2023-01-09', 4,    'Support');
SELECT setval('employees_employee_id_seq', 8);

-- orders
INSERT INTO orders (order_id, customer_id, employee_id, order_date, status, ship_country) VALUES
    (1,  1,  2, '2024-01-05 10:15:00+07', 'delivered',  'Thailand'),
    (2,  2,  2, '2024-01-08 14:02:00+07', 'delivered',  'Thailand'),
    (3,  3,  3, '2024-01-12 09:40:00+07', 'delivered',  'Thailand'),
    (4,  1,  2, '2024-01-20 16:25:00+07', 'delivered',  'Thailand'),
    (5,  4,  7, '2024-02-02 11:00:00+07', 'delivered',  'Thailand'),
    (6,  5,  3, '2024-02-10 13:30:00+07', 'cancelled',  'Thailand'),
    (7,  7,  2, '2024-02-14 08:55:00+07', 'delivered',  'USA'),
    (8,  6,  7, '2024-02-18 17:10:00+07', 'delivered',  'Thailand'),
    (9,  8,  3, '2024-02-25 12:20:00+07', 'delivered',  'UK'),
    (10, 2,  2, '2024-03-01 10:05:00+07', 'delivered',  'Thailand'),
    (11, 9,  7, '2024-03-05 15:45:00+07', 'delivered',  'Japan'),
    (12, 10, 3, '2024-03-09 09:15:00+07', 'shipped',    'China'),
    (13, 3,  2, '2024-03-15 11:30:00+07', 'delivered',  'Thailand'),
    (14, 11, 7, '2024-03-20 14:50:00+07', 'delivered',  'Thailand'),
    (15, 12, 2, '2024-03-28 10:40:00+07', 'processing', 'Thailand'),
    (16, 1,  3, '2024-04-02 09:00:00+07', 'delivered',  'Thailand'),
    (17, 13, 7, '2024-04-10 16:15:00+07', 'delivered',  'Australia'),
    (18, 14, 2, '2024-04-15 13:00:00+07', 'pending',    'Thailand'),
    (19, 5,  3, '2024-04-22 10:20:00+07', 'delivered',  'Thailand'),
    (20, 15, 7, '2024-04-30 15:05:00+07', 'delivered',  'Thailand');
SELECT setval('orders_order_id_seq', 20);

-- order_items
-- หมายเหตุ: order_item_id = 27 มี product_id เป็น NULL โดยตั้งใจ (ค่าธรรมเนียมห่อของขวัญที่ไม่ผูกกับสินค้าใน catalog)
-- ใช้สาธิตข้อควรระวังเรื่อง NULL กับ NOT IN ใน Step 234 และ 237
INSERT INTO order_items (order_item_id, order_id, product_id, quantity, unit_price) VALUES
    (1,  1,  1,    1, 42900.00),
    (2,  1,  7,    1,  3290.00),
    (3,  2,  2,    1, 32900.00),
    (4,  3,  8,    2,   590.00),
    (5,  3,  9,    1,   890.00),
    (6,  4,  15,   1,   890.00),
    (7,  4,  16,   2,   450.00),
    (8,  5,  10,   1,   790.00),
    (9,  5,  11,   1,  1290.00),
    (10, 6,  4,    1, 45900.00),
    (11, 7,  3,    2,  6990.00),
    (12, 8,  12,   2,   690.00),
    (13, 8,  13,   1,  2590.00),
    (14, 9,  19,   3,   590.00),
    (15, 10, 1,    1, 42900.00),
    (16, 11, 17,   1,   690.00),
    (17, 11, 18,   1,  3990.00),
    (18, 12, 6,    1, 24900.00),
    (19, 13, 14,   1,  1990.00),
    (20, 14, 9,    2,   890.00),
    (21, 14, 8,    3,   590.00),
    (22, 15, 5,    1, 39900.00),
    (23, 16, 2,    1, 32900.00),
    (24, 17, 7,    2,  3290.00),
    (25, 18, 16,   1,   450.00),
    (26, 19, 3,    1,  6990.00),
    (27, 1,  NULL, 1,   100.00),
    (28, 19, 13,   1,  2590.00),
    (29, 20, 10,   2,   790.00);
SELECT setval('order_items_order_item_id_seq', 29);

-- reviews
INSERT INTO reviews (review_id, product_id, customer_id, rating, review_text, review_date) VALUES
    (1,  1,  1,  5, 'สินค้าดีมาก ส่งไว แพ็คดี',                       '2024-01-10'),
    (2,  2,  2,  4, 'ใช้งานดี แบตอึดกว่ารุ่นเก่า',                     '2024-01-15'),
    (3,  3,  7,  5, 'คุ้มราคามาก แนะนำเลย',                           '2024-02-20'),
    (4,  4,  5,  3, 'เครื่องดีแต่ราคาแพงไปหน่อยสำหรับสเปกนี้',           '2024-02-15'),
    (5,  8,  3,  4, 'ผ้านุ่มใส่สบาย ไซซ์ตรงปก',                        '2024-01-18'),
    (6,  9,  3,  5, 'ทรงสวย ใส่ได้พอดีตัว',                           '2024-01-19'),
    (7,  10, 4,  5, 'ชุดสวยมาก ผ้าดีเกินราคา',                        '2024-02-08'),
    (8,  12, 6,  4, 'ทอดไข่ไม่ติดกระทะเลย ทำความสะอาดง่าย',             '2024-02-22'),
    (9,  13, 6,  5, 'หุงข้าวอร่อย ใช้งานง่ายมาก',                      '2024-02-23'),
    (10, 15, 1,  5, 'หนังสือดีมากสำหรับโปรแกรมเมอร์ทุกระดับ',           '2024-01-25'),
    (11, 16, 1,  4, 'อ่านสนุก นำไปปรับใช้ในชีวิตได้จริง',              '2024-01-26'),
    (12, 17, 9,  5, 'เสื่อโยคะดีมาก หนาพอดี ไม่ลื่น',                  '2024-03-10'),
    (13, 19, 8,  3, 'เซรั่มโอเค แต่ราคาสูงไปหน่อยเมื่อเทียบกับปริมาณ',    '2024-03-02'),
    (14, 1,  11, 4, 'โดยรวมพอใจ กล้องถ่ายรูปสวย',                     '2024-03-25'),
    (15, 2,  14, 2, 'แบตหมดเร็วกว่าที่คิดไว้ ผิดหวังนิดหน่อย',           '2024-04-16'),
    (16, 7,  13, 5, 'เม้าส์ลื่นมาก คุ้มค่าคุ้มราคา',                    '2024-04-12'),
    (17, 20, 12, 1, 'สั่งไม่ได้ สินค้าหมดสต็อกแต่ยังโชว์ในเว็บ',          '2024-03-30');
SELECT setval('reviews_review_id_seq', 17);

-- payments (order 6 ถูกยกเลิก, order 15 กำลังดำเนินการ, order 18 ยัง pending จึงยังไม่มี payment)
INSERT INTO payments (payment_id, order_id, payment_date, amount, payment_method) VALUES
    (1,  1,  '2024-01-05 10:20:00+07', 46290.00, 'credit_card'),
    (2,  2,  '2024-01-08 14:05:00+07', 32900.00, 'credit_card'),
    (3,  3,  '2024-01-12 09:45:00+07',  2070.00, 'promptpay'),
    (4,  4,  '2024-01-20 16:30:00+07',  1790.00, 'credit_card'),
    (5,  5,  '2024-02-02 11:05:00+07',  2080.00, 'promptpay'),
    (6,  7,  '2024-02-14 09:00:00+07', 13980.00, 'paypal'),
    (7,  8,  '2024-02-18 17:15:00+07',  3970.00, 'credit_card'),
    (8,  9,  '2024-02-25 12:25:00+07',  1770.00, 'credit_card'),
    (9,  10, '2024-03-01 10:10:00+07', 42900.00, 'credit_card'),
    (10, 11, '2024-03-05 15:50:00+07',  4680.00, 'promptpay'),
    (11, 12, '2024-03-09 09:20:00+07', 24900.00, 'credit_card'),
    (12, 13, '2024-03-15 11:35:00+07',  1990.00, 'promptpay'),
    (13, 14, '2024-03-20 14:55:00+07',  3550.00, 'credit_card'),
    (14, 16, '2024-04-02 09:05:00+07', 32900.00, 'credit_card'),
    (15, 17, '2024-04-10 16:20:00+07',  6580.00, 'paypal'),
    (16, 19, '2024-04-22 10:25:00+07',  9580.00, 'credit_card'),
    (17, 20, '2024-04-30 15:10:00+07',  1580.00, 'promptpay');
SELECT setval('payments_payment_id_seq', 17);
```

ตรวจสอบว่าข้อมูลถูกโหลดครบด้วยคำสั่งง่าย ๆ:

```sql
SELECT 'categories' AS tbl, count(*) FROM categories
UNION ALL SELECT 'suppliers',   count(*) FROM suppliers
UNION ALL SELECT 'products',    count(*) FROM products
UNION ALL SELECT 'customers',   count(*) FROM customers
UNION ALL SELECT 'employees',   count(*) FROM employees
UNION ALL SELECT 'orders',      count(*) FROM orders
UNION ALL SELECT 'order_items', count(*) FROM order_items
UNION ALL SELECT 'reviews',     count(*) FROM reviews
UNION ALL SELECT 'payments',    count(*) FROM payments;
```

```
    tbl      | count
-------------+-------
 categories  |    12
 suppliers   |     8
 products    |    20
 customers   |    15
 employees   |     8
 orders      |    20
 order_items |    29
 reviews     |    17
 payments    |    17
(9 rows)
```

พร้อมแล้ว มาเริ่มเนื้อหากันเลย

---

## Step 231: Subquery คืออะไร

**Subquery** (บางครั้งเรียก inner query หรือ nested query) คือคำสั่ง `SELECT` ที่ถูกเขียนซ้อนอยู่ภายในคำสั่ง SQL อื่น (outer query) โดย subquery จะถูกประมวลผลก่อน แล้วผลลัพธ์ที่ได้จะถูกนำไปใช้ต่อใน outer query เสมอต้องอยู่ภายในวงเล็บ `( ... )`

จุดที่ subquery สามารถปรากฏได้ในคำสั่ง SQL มีหลักๆ 4 ตำแหน่ง:

| ตำแหน่ง | ตัวอย่างการใช้งาน |
|---|---|
| `SELECT` list | แสดงค่าสรุปภาพรวมข้าง ๆ แต่ละแถว |
| `FROM` clause | ใช้ผลลัพธ์ของ query หนึ่งเป็นตารางชั่วคราว (derived table) |
| `WHERE` / `HAVING` | กรองแถวโดยเทียบกับผลลัพธ์ของ query อื่น |
| หลัง keyword `IN`, `EXISTS`, `ANY`, `ALL` | ตรวจสอบสมาชิกหรือการมีอยู่ของข้อมูล |

Subquery แบ่งตามรูปร่างของผลลัพธ์ได้ 3 ประเภทหลัก:

1. **Scalar subquery** — คืนค่ากลับมา **1 แถว 1 คอลัมน์** (ค่าเดียว) ใช้ได้ทุกที่ที่ใช้ literal value ได้ เช่น ใน `SELECT`, `WHERE` ร่วมกับ `=`, `>`, `<`
2. **Row subquery** — คืนค่ากลับมา **1 แถว หลายคอลัมน์** ใช้เปรียบเทียบกับ row constructor เช่น `WHERE (col1, col2) = (SELECT c1, c2 FROM ...)`
3. **Table subquery** — คืนค่ากลับมา **หลายแถว หลายคอลัมน์** (หรือหลายแถว 1 คอลัมน์) ใช้กับ `IN`, `EXISTS`, `ANY`, `ALL` หรือใช้เป็นตารางใน `FROM`

ตัวอย่างง่าย ๆ ที่แสดงทั้ง 3 ประเภท:

```sql
-- 1) Scalar subquery: ราคาสินค้าที่แพงที่สุด (ค่าเดียว)
SELECT MAX(unit_price) FROM products;

-- 2) Row subquery: ข้อมูลของสินค้าที่แพงที่สุด (1 แถว หลายคอลัมน์)
SELECT product_name, unit_price
FROM products
WHERE (unit_price, product_id) = (
    SELECT unit_price, product_id
    FROM products
    ORDER BY unit_price DESC, product_id
    LIMIT 1
);

-- 3) Table subquery: รายชื่อสินค้าทุกชิ้นในหมวด Smartphones (หลายแถว)
SELECT product_id, product_name
FROM products
WHERE category_id IN (
    SELECT category_id FROM categories WHERE category_name = 'Smartphones'
);
```

```
 product_id |     product_name
------------+------------------------
          1 | iPhone 15 Pro 128GB
          2 | Samsung Galaxy S24
          3 | Xiaomi Redmi Note 13
(3 rows)
```

**ข้อสังเกตสำคัญของ subquery:**

- subquery ที่อยู่ใน `IN`, `EXISTS` ไม่จำเป็นต้องคืนค่าเพียงแถวเดียว แต่ scalar subquery (ที่ใช้กับ `=`, `>`, `<`) **ต้อง**คืนค่าไม่เกิน 1 แถว 1 คอลัมน์ ถ้าคืนมาหลายแถวจะเกิด error `more than one row returned by a subquery used as an expression`
- subquery สามารถซ้อนกันได้หลายชั้น (nested subquery) แต่ยิ่งซ้อนลึกยิ่งอ่านยาก — บทถัดไป (Part 025: CTE) จะแนะนำวิธีเขียนให้อ่านง่ายขึ้นด้วย `WITH`
- subquery แบ่งได้อีกมุมมองหนึ่งคือ **correlated** (อ้างอิงตาราง outer query) กับ **non-correlated** (ทำงานเป็นอิสระ ไม่พึ่งพา outer query) ซึ่งจะอธิบายละเอียดใน Step 236

---

## Step 232: Scalar subquery ใน SELECT list

การใส่ scalar subquery ไว้ใน `SELECT` list ทำให้เราสามารถแสดง**ค่าสรุปภาพรวม**ควบคู่ไปกับข้อมูลรายแถวได้ในผลลัพธ์เดียว โดยไม่ต้องรันคำสั่งแยกสองครั้ง

ตัวอย่าง: แสดงราคาสินค้าแต่ละชิ้น เทียบกับราคาเฉลี่ยของสินค้าทั้งหมด

```sql
SELECT
    product_name,
    unit_price,
    (SELECT ROUND(AVG(unit_price), 2) FROM products) AS avg_price_all,
    unit_price - (SELECT ROUND(AVG(unit_price), 2) FROM products) AS diff_from_avg
FROM products
ORDER BY diff_from_avg DESC
LIMIT 6;
```

```
      product_name       | unit_price | avg_price_all | diff_from_avg
--------------------------+------------+----------------+----------------
 MacBook Air M3 13"       |   45900.00 |       10574.50 |      35325.50
 iPhone 15 Pro 128GB      |   42900.00 |       10574.50 |      32325.50
 Dell XPS 13              |   39900.00 |       10574.50 |      29325.50
 Samsung Galaxy S24       |   32900.00 |       10574.50 |      22325.50
 Lenovo ThinkPad E14      |   24900.00 |       10574.50 |      14325.50
 Adjustable Dumbbell Set  |    3990.00 |       10574.50 |      -6584.50
(6 rows)
```

ในตัวอย่างนี้ subquery `(SELECT ROUND(AVG(unit_price), 2) FROM products)` เป็น **non-correlated scalar subquery** — คำนวณค่าเฉลี่ยจากทั้งตาราง `products` โดยไม่สนใจว่ากำลังพิจารณาแถวไหนของ outer query อยู่ ค่านี้จึงเท่ากันทุกแถว PostgreSQL ฉลาดพอที่จะคำนวณ subquery แบบนี้เพียงครั้งเดียวแล้วนำผลไปใช้ซ้ำ (ไม่ใช่รันใหม่ทุกแถว) จึงมีค่าใช้จ่ายด้าน performance ต่ำ

อีกตัวอย่างหนึ่ง: แสดงเปอร์เซ็นต์ของราคาสินค้าเทียบกับราคาสินค้าที่แพงที่สุดในระบบ

```sql
SELECT
    product_name,
    unit_price,
    ROUND(
        unit_price / (SELECT MAX(unit_price) FROM products) * 100,
        1
    ) AS pct_of_most_expensive
FROM products
WHERE is_active = true
ORDER BY pct_of_most_expensive DESC
LIMIT 5;
```

```
     product_name     | unit_price | pct_of_most_expensive
-----------------------+------------+------------------------
 MacBook Air M3 13"    |   45900.00 |                  100.0
 iPhone 15 Pro 128GB   |   42900.00 |                   93.5
 Dell XPS 13           |   39900.00 |                   86.9
 Samsung Galaxy S24    |   32900.00 |                   71.7
 Lenovo ThinkPad E14   |   24900.00 |                   54.2
(5 rows)
```

**กฎสำคัญ:** scalar subquery ต้องคืนค่ากลับมาพอดี **1 แถว 1 คอลัมน์** เท่านั้น ถ้าไม่มีแถวใดตรงเงื่อนไขเลย จะได้ผลลัพธ์เป็น `NULL` (ไม่ error) แต่ถ้าคืนมามากกว่า 1 แถว จะเกิด runtime error ทันที ลองดูตัวอย่างที่จะ error:

```sql
-- ผิด! subquery คืนมาหลายแถว เพราะไม่ได้จำกัดด้วย WHERE หรือ aggregate function
SELECT product_name, (SELECT unit_price FROM products) AS wrong_price
FROM products;
-- ERROR:  more than one row returned by a subquery used as an expression
```

จุดที่ต้องระวังคือ ต้องมั่นใจว่า subquery ใน `SELECT` list จะคืนค่าแถวเดียวเสมอ โดยทั่วไปทำได้ด้วยการใช้ aggregate function (`AVG`, `MAX`, `MIN`, `COUNT`, `SUM`) หรือใส่ `WHERE` ที่กรองให้เหลือแถวเดียวแน่นอน (เช่น กรองด้วย primary key)

---

## Step 233: Subquery ใน WHERE ด้วย =, >, < กับ scalar subquery

เมื่อ subquery คืนค่าเดียว (scalar) เราสามารถนำไปเปรียบเทียบใน `WHERE` ด้วย operator ปกติได้เลย เช่น `=`, `>`, `<`, `>=`, `<=`, `<>`

ตัวอย่างที่ 1: หาสินค้าที่ราคาสูงกว่าราคาเฉลี่ยของสินค้าทั้งหมด

```sql
SELECT product_name, unit_price
FROM products
WHERE unit_price > (SELECT AVG(unit_price) FROM products)
ORDER BY unit_price DESC;
```

```
      product_name       | unit_price
--------------------------+------------
 MacBook Air M3 13"       |   45900.00
 iPhone 15 Pro 128GB      |   42900.00
 Dell XPS 13              |   39900.00
 Samsung Galaxy S24       |   32900.00
 Lenovo ThinkPad E14      |   24900.00
(5 rows)
```

ตัวอย่างที่ 2: หาสินค้าที่ราคาแพงที่สุดโดยใช้ `=` กับ `MAX()` (แทนที่จะใช้ `ORDER BY ... LIMIT 1` ซึ่งบางกรณีอ่านเจตนาได้ชัดกว่า)

```sql
SELECT product_name, unit_price
FROM products
WHERE unit_price = (SELECT MAX(unit_price) FROM products);
```

```
     product_name     | unit_price
-----------------------+------------
 MacBook Air M3 13"    |   45900.00
(1 row)
```

ข้อดีของวิธีนี้เทียบกับ `ORDER BY ... LIMIT 1` คือถ้ามีสินค้าราคาสูงสุด**เท่ากันหลายชิ้น** วิธีนี้จะคืนมาครบทุกชิ้น ในขณะที่ `LIMIT 1` จะคืนมาแค่ชิ้นเดียว

ตัวอย่างที่ 3: ใช้ scalar subquery เปรียบเทียบวันที่ — หาพนักงานที่เข้าทำงานก่อน "Panida Thepwong"

```sql
SELECT first_name, last_name, hire_date, department
FROM employees
WHERE hire_date < (
    SELECT hire_date FROM employees
    WHERE first_name = 'Panida' AND last_name = 'Thepwong'
)
ORDER BY hire_date;
```

```
 first_name | last_name | hire_date  | department
------------+-----------+------------+------------
 Kanya      | Srisuk    | 2021-01-10 | Sales
 Thawatchai | Pongsak   | 2021-03-15 | Sales
 Areeya     | Nantawong | 2021-06-01 | Sales
 Somsak     | Uraiwan   | 2022-01-20 | Support
 Nattaya    | Chumphon  | 2022-02-14 | Support
 Ekachai    | Boonrod   | 2022-05-05 | Support
(6 rows)
```

ตัวอย่างที่ 4: หา order ที่มียอดชำระ (payment amount) มากกว่ายอดชำระเฉลี่ยของทุก order

```sql
SELECT o.order_id, c.first_name, c.last_name, p.amount
FROM orders o
JOIN customers c ON c.customer_id = o.customer_id
JOIN payments p ON p.order_id = o.order_id
WHERE p.amount > (SELECT AVG(amount) FROM payments)
ORDER BY p.amount DESC;
```

```
 order_id | first_name | last_name |  amount
----------+------------+-----------+----------
        1 | Somchai    | Jaidee    | 46290.00
       10 | Malee      | Suksan    | 42900.00
        7 | John       | Anderson  | 13980.00
       12 | Li         | Wei       | 24900.00
        2 | Malee      | Suksan    | 32900.00
       16 | Somchai    | Jaidee    | 32900.00
(6 rows)
```

**สิ่งที่ต้องระวัง:** ถ้าลืมใช้ aggregate function หรือเงื่อนไขกรองใน subquery แล้ว subquery ดันคืนค่ามาหลายแถว จะเกิด error `more than one row returned by a subquery used as an expression` ทันที เหมือนที่เห็นใน Step 232 — เป็นข้อผิดพลาดที่พบบ่อยมากสำหรับมือใหม่

---

## Step 234: Subquery ใน WHERE ด้วย IN / NOT IN

เมื่อ subquery คืนค่ามา**หลายแถว** (table subquery แบบ 1 คอลัมน์) เราใช้ `IN` แทนการเปรียบเทียบด้วย `=` ได้ ส่วน `NOT IN` ใช้ตรวจสอบว่าค่าที่พิจารณาไม่อยู่ในรายการที่ subquery คืนมา

### การใช้ IN

ตัวอย่างที่ 1: หาสินค้าทั้งหมดที่อยู่ในหมวดเสื้อผ้า (category name มีคำว่า "Clothing")

```sql
SELECT product_name, category_id, unit_price
FROM products
WHERE category_id IN (
    SELECT category_id FROM categories WHERE category_name LIKE '%Clothing%'
)
ORDER BY product_name;
```

```
       product_name        | category_id | unit_price
----------------------------+-------------+------------
 Men's Cotton Polo Shirt    |           8 |     590.00
 Men's Slim Fit Jeans       |           8 |     890.00
 Women's Silk Blouse        |           9 |    1290.00
 Women's Summer Dress       |           9 |     790.00
(4 rows)
```

ตัวอย่างที่ 2: หาลูกค้าที่เคยสั่งซื้อสินค้าอย่างน้อย 1 ครั้ง

```sql
SELECT customer_id, first_name, last_name
FROM customers
WHERE customer_id IN (SELECT customer_id FROM orders)
ORDER BY customer_id;
```

```
 customer_id | first_name |  last_name
-------------+------------+-------------
           1 | Somchai    | Jaidee
           2 | Malee      | Suksan
           3 | Anan       | Wongsa
           4 | Pranee     | Chaiyo
           5 | Weerawat   | Boonmee
           6 | Siriporn   | Intharat
           7 | John       | Anderson
           8 | Emily      | Clark
           9 | Yuki       | Tanaka
          10 | Li         | Wei
          11 | Kittipong  | Rattana
          12 | Nutthida   | Saelim
          13 | David      | Miller
          14 | Chanya     | Phromma
          15 | Piti       | Suwannaphum
(15 rows)
```

ในกรณีนี้ทุกคนเคยสั่งซื้ออย่างน้อย 1 ครั้ง จึงได้ครบ 15 แถว

### การใช้ NOT IN — และข้อควรระวังสำคัญเรื่อง NULL

`NOT IN` ใช้งานได้ตรงไปตรงมาเมื่อคอลัมน์ที่ subquery คืนมา**ไม่มีค่า NULL**เลย เช่น หาสินค้าที่ **ไม่เคย**ถูกสั่งซื้อ โดยลองกรองด้วย `product_id` ที่ไม่ใช่ NULL ก่อน:

```sql
SELECT product_id, product_name
FROM products
WHERE product_id NOT IN (
    SELECT product_id FROM order_items WHERE product_id IS NOT NULL
)
ORDER BY product_id;
```

```
 product_id |       product_name
------------+----------------------------
         20 | Wooden Building Blocks Set
(1 row)
```

ผลลัพธ์ถูกต้อง — `Wooden Building Blocks Set` เป็นสินค้าที่เลิกขายแล้ว (`is_active = false`) และไม่เคยมีใครสั่งซื้อเลย

แต่ถ้าเรา**ลืม**กรอง `WHERE product_id IS NOT NULL` ออกจาก subquery (ย้อนกลับไปดู Part 011 เรื่อง `IS NULL`) — จำได้ไหมว่าตาราง `order_items` ของเรามีแถวหนึ่ง (`order_item_id = 27`) ที่ `product_id` เป็น `NULL` (แทนค่าธรรมเนียมห่อของขวัญที่ไม่ผูกกับสินค้าใน catalog) ลองดูว่าเกิดอะไรขึ้น:

```sql
-- อันตราย! ลืมกรอง NULL ออกจาก subquery
SELECT product_id, product_name
FROM products
WHERE product_id NOT IN (
    SELECT product_id FROM order_items   -- ยังมี NULL ปนอยู่
)
ORDER BY product_id;
```

```
 product_id | product_name
------------+---------------
(0 rows)
```

**ได้ผลลัพธ์ว่างเปล่า!** ทั้งที่เรารู้ว่ามีสินค้าอย่างน้อย 1 ชิ้น (`product_id = 20`) ที่ไม่เคยถูกสั่งซื้อเลย นี่คือกับดักที่พบบ่อยและอันตรายที่สุดอันหนึ่งของ SQL

**เหตุผลเชิงตรรกะ:** `x NOT IN (a, b, NULL)` ในทาง logic เทียบเท่ากับ `NOT (x = a OR x = b OR x = NULL)` และเนื่องจาก `x = NULL` จะได้ผลเป็น `UNKNOWN` เสมอ (ไม่ใช่ `TRUE` หรือ `FALSE` — ตามที่อธิบายไว้ใน Part 011) การนำ `OR` ที่มีค่า `UNKNOWN` ปนอยู่มา `NOT` จึงทำให้ผลลัพธ์สุดท้ายเป็น `UNKNOWN` ไปด้วย ไม่ว่า `x` จะเป็นค่าอะไรก็ตาม และแถวที่มีเงื่อนไขเป็น `UNKNOWN` จะถูก `WHERE` กรองทิ้งเสมอ ผลคือ `NOT IN` ที่มี `NULL` ปนอยู่ใน subquery จะทำให้**ทั้ง query คืนค่าว่างเปล่าเสมอ** ไม่ว่า outer query จะมีกี่แถวก็ตาม

**วิธีป้องกัน 3 แบบ:**

1. กรอง `IS NOT NULL` ในคอลัมน์ที่ใช้กับ `IN`/`NOT IN` เสมอ (อย่างที่ทำไปแล้วด้านบน)
2. ใช้ `NOT EXISTS` แทน `NOT IN` (แนะนำที่สุด — ปลอดภัยจาก NULL โดยธรรมชาติ อธิบายละเอียดใน Step 237)
3. ใช้ `LEFT JOIN ... WHERE right_table.key IS NULL` แทน (อธิบายใน Step 239)

> **กฎทองที่ต้องจำ:** ทุกครั้งที่เขียน `NOT IN (subquery)` ให้ถามตัวเองเสมอว่า "คอลัมน์นี้มีโอกาส NULL ไหม" ถ้าไม่แน่ใจ ให้ใช้ `NOT EXISTS` ไปเลยจะปลอดภัยกว่า

---

## Step 235: Subquery ใน FROM (derived table / inline view)

เราสามารถใช้ผลลัพธ์ของ `SELECT` หนึ่งเป็นเหมือน "ตารางชั่วคราว" ใน `FROM` clause ของ query อื่นได้ เรียกว่า **derived table** หรือ **inline view** ข้อกำหนดสำคัญของ PostgreSQL คือ **derived table ทุกตัวต้องมี alias เสมอ** (ต่างจาก RDBMS บางตัวที่อนุญาตให้ไม่ใส่ alias ได้)

ตัวอย่างที่ 1: คำนวณยอดใช้จ่ายรวมของลูกค้าแต่ละคนใน derived table ก่อน แล้วนำไป join กับตาราง `customers` เพื่อเอาชื่อ

```sql
SELECT
    c.first_name,
    c.last_name,
    ot.total_spent
FROM customers c
JOIN (
    SELECT o.customer_id, SUM(oi.quantity * oi.unit_price) AS total_spent
    FROM orders o
    JOIN order_items oi ON oi.order_id = o.order_id
    WHERE o.status <> 'cancelled'
    GROUP BY o.customer_id
) ot ON ot.customer_id = c.customer_id
ORDER BY ot.total_spent DESC
LIMIT 5;
```

```
 first_name | last_name |  total_spent
------------+-----------+---------------
 Somchai    | Jaidee    |     122090.00
 Malee      | Suksan    |      65800.00
 Li         | Wei       |      24900.00
 Anan       | Wongsa    |      10450.00
 John       | Anderson  |      13980.00
(5 rows)
```

สังเกตว่า `ot` คือ alias ของ derived table (ต้องใส่ ไม่งั้น PostgreSQL จะ error: `subquery in FROM must have an alias`) ลองดูตัวอย่าง error:

```sql
-- ผิด! ไม่ใส่ alias ให้ derived table
SELECT * FROM (SELECT customer_id, COUNT(*) FROM orders GROUP BY customer_id);
-- ERROR:  subquery in FROM must have an alias
-- HINT:  For example, FROM (SELECT ...) [AS] foo.
```

ตัวอย่างที่ 2: หารายได้รวมต่อหมวดหมู่สินค้า แล้วกรองเฉพาะหมวดที่มีรายได้เกิน 50,000 บาท (คล้าย `HAVING` แต่แยกเป็นสอง query ผ่าน derived table เพื่อสาธิตแนวคิด — ในทางปฏิบัติกรณีนี้ใช้ `GROUP BY ... HAVING` ตรง ๆ ได้เลย ดู Part 028)

```sql
SELECT cat.category_name, rev.category_revenue
FROM (
    SELECT p.category_id, SUM(oi.quantity * oi.unit_price) AS category_revenue
    FROM order_items oi
    JOIN products p ON p.product_id = oi.product_id
    GROUP BY p.category_id
) rev
JOIN categories cat ON cat.category_id = rev.category_id
WHERE rev.category_revenue > 50000
ORDER BY rev.category_revenue DESC;
```

```
    category_name     | category_revenue
-----------------------+-------------------
 Smartphones           |         129680.00
 Computers & Laptops    |         110680.00
(2 rows)
```

**เมื่อไหร่ควรใช้ subquery ใน FROM:**

- เมื่อต้องการ aggregate ข้อมูลก่อน แล้วค่อยกรอง/join กับผลลัพธ์ที่ aggregate แล้ว (กรองหลัง `GROUP BY` ในระดับที่ `HAVING` ทำไม่สะดวก เช่น ต้อง join กับตารางอื่นก่อน)
- เมื่อต้องการสร้างชุดข้อมูลกลางที่ใช้ซ้ำหลายครั้งใน query เดียวกัน
- ข้อควรระวัง: ถ้า derived table ซับซ้อนและถูกใช้ซ้ำหลายครั้ง ควรพิจารณาใช้ **CTE** (`WITH`) แทน เพราะอ่านง่ายกว่าและ (ใน PostgreSQL สมัยใหม่) มักถูก optimize เทียบเท่ากัน — รายละเอียดเต็มอยู่ใน Part 025

---

## Step 236: Correlated subquery

**Correlated subquery** คือ subquery ที่**อ้างอิงคอลัมน์จากตาราง (หรือ query) ภายนอก** (outer query) ทำให้ subquery นั้นไม่สามารถรันแยกเดี่ยว ๆ ได้อีกต่อไป เพราะต้องพึ่งพาค่าจากแถวปัจจุบันของ outer query เสมอ

เปรียบเทียบให้เห็นความแตกต่าง:

| | Non-correlated subquery | Correlated subquery |
|---|---|---|
| การอ้างอิง | ไม่อ้างอิงตาราง outer query | อ้างอิงคอลัมน์จากตาราง outer query |
| ความเป็นอิสระ | รันแยกเดี่ยว ๆ ได้ | รันแยกเดี่ยว ๆ ไม่ได้ (error: column ไม่รู้จัก) |
| จำนวนครั้งที่ประมวลผล (แนวคิด) | ครั้งเดียว แล้วนำผลไปใช้ซ้ำ | ในทางแนวคิดคือครั้งละ 1 ครั้งต่อแถวของ outer query (ตัว optimizer จริงอาจ rewrite ให้มีประสิทธิภาพขึ้นได้) |
| ตัวอย่างใน Part นี้ | Step 232, 233, 234, 235 | Step 236 นี้, และมักพบร่วมกับ `EXISTS` ใน Step 237 |

ตัวอย่างที่ 1: หาสินค้าที่ราคาสูงกว่าราคาเฉลี่ย **ของหมวดหมู่ตัวเอง** (ต่างจาก Step 233 ที่เทียบกับค่าเฉลี่ยของสินค้า**ทั้งหมด**)

```sql
SELECT
    p1.product_name,
    p1.category_id,
    p1.unit_price,
    (SELECT ROUND(AVG(p2.unit_price), 2)
     FROM products p2
     WHERE p2.category_id = p1.category_id) AS avg_price_in_category
FROM products p1
WHERE p1.unit_price > (
    SELECT AVG(p2.unit_price)
    FROM products p2
    WHERE p2.category_id = p1.category_id   -- อ้างอิง p1 จาก outer query = correlated
)
ORDER BY p1.category_id, p1.unit_price DESC;
```

```
      product_name        | category_id | unit_price | avg_price_in_category
---------------------------+-------------+------------+-------------------------
 Lenovo ThinkPad E14       |           7 |   24900.00 |               18363.33
 Women's Silk Blouse       |           9 |    1290.00 |                1040.00
 Nonstick Frying Pan 28cm  |          10 |     690.00 |                1756.67
 Adjustable Dumbbell Set   |           5 |    3990.00 |                2340.00
 Clean Code (Book)         |           4 |     890.00 |                 670.00
(5 rows)
```

จะเห็นว่า subquery `SELECT AVG(p2.unit_price) FROM products p2 WHERE p2.category_id = p1.category_id` **ไม่สามารถรันแยกเดี่ยว ๆ ได้** เพราะ `p1.category_id` ไม่มีอยู่จริงถ้าไม่มี outer query — subquery นี้ต้องถูกประเมินผลใหม่สำหรับ**แต่ละหมวดหมู่**ที่ปรากฏใน outer query (ในทางแนวคิด คือหนึ่งครั้งต่อหนึ่งแถวของ `p1`)

ตัวอย่างที่ 2: หาลูกค้าที่ order ล่าสุดของพวกเขา (ไม่นับ cancelled) อยู่หลังวันที่ 15 มีนาคม 2024

```sql
SELECT c.first_name, c.last_name
FROM customers c
WHERE (
    SELECT MAX(o.order_date)
    FROM orders o
    WHERE o.customer_id = c.customer_id      -- correlated กับ c ภายนอก
      AND o.status <> 'cancelled'
) > '2024-03-15'
ORDER BY c.customer_id;
```

```
 first_name |  last_name
------------+-------------
 Anan       | Wongsa
 Kittipong  | Rattana
 Nutthida   | Saelim
 Somchai    | Jaidee
 Weerawat   | Boonmee
 David      | Miller
 Piti       | Suwannaphum
(7 rows)
```

**ข้อควรระวังด้าน performance:** correlated subquery ในเชิงแนวคิดต้องถูกประเมินผลใหม่สำหรับทุกแถวของ outer query ถ้าตารางมีข้อมูลจำนวนมาก (หลักล้านแถว) และไม่มี index รองรับ อาจทำให้ query ช้าลงอย่างมีนัยสำคัญ ในทางปฏิบัติ PostgreSQL query planner มักพยายาม rewrite correlated subquery บางรูปแบบให้กลายเป็น join ภายในโดยอัตโนมัติ (โดยเฉพาะกรณี `EXISTS`/`NOT EXISTS` ใน Step 237) แต่ scalar correlated subquery ใน `SELECT` list มักถูก optimize ได้ยากกว่า จึงควรตรวจสอบ execution plan ด้วย `EXPLAIN ANALYZE` เสมอเมื่อใช้กับข้อมูลขนาดใหญ่ (จะเรียนละเอียดเรื่อง `EXPLAIN` ในบทที่ว่าด้วย performance tuning ของหลักสูตรระดับสูง)

---

## Step 237: EXISTS และ NOT EXISTS

`EXISTS` เป็น operator ที่ตรวจสอบว่า subquery (โดยทั่วไปเป็น correlated subquery) **คืนค่ามาอย่างน้อย 1 แถวหรือไม่** โดยจะคืนค่าเป็น `TRUE`/`FALSE` เท่านั้น — `EXISTS` **ไม่สนใจว่า subquery คืนคอลัมน์อะไรมา** จึงเป็นธรรมเนียมปฏิบัติทั่วไปที่จะเขียน `SELECT 1` ภายใน subquery ของ `EXISTS` (เพราะค่าคอลัมน์ไม่มีผลต่อผลลัพธ์เลย)

ตัวอย่างที่ 1: หาลูกค้าที่เคยสั่งซื้อสินค้าอย่างน้อย 1 ครั้ง (เทียบกับ `IN` ใน Step 234)

```sql
-- แบบ IN (จาก Step 234)
SELECT customer_id, first_name, last_name
FROM customers c
WHERE customer_id IN (SELECT customer_id FROM orders);

-- แบบ EXISTS (correlated) — ให้ผลลัพธ์เหมือนกันทุกประการ
SELECT customer_id, first_name, last_name
FROM customers c
WHERE EXISTS (
    SELECT 1 FROM orders o WHERE o.customer_id = c.customer_id
)
ORDER BY customer_id;
```

ทั้งสองแบบให้ผลลัพธ์เหมือนกัน (15 แถว) เพราะในชุดข้อมูลนี้ทุกคนเคยสั่งซื้อ

ตัวอย่างที่ 2: `NOT EXISTS` — หาสินค้าที่**ไม่เคย**ถูกสั่งซื้อเลย (แก้ปัญหา NULL ที่เจอใน Step 234 ได้อย่างสมบูรณ์ โดยไม่ต้องกรอง `IS NOT NULL` เอง)

```sql
SELECT p.product_id, p.product_name
FROM products p
WHERE NOT EXISTS (
    SELECT 1 FROM order_items oi WHERE oi.product_id = p.product_id
)
ORDER BY p.product_id;
```

```
 product_id |       product_name
------------+----------------------------
         20 | Wooden Building Blocks Set
(1 row)
```

สังเกตว่า query นี้ให้ผลลัพธ์ถูกต้อง**ทันที** แม้ตาราง `order_items` จะมีแถวที่ `product_id IS NULL` ปนอยู่ก็ตาม (ต่างจาก `NOT IN` ที่ต้องกรอง `IS NOT NULL` ด้วยตัวเองใน Step 234 ไม่งั้นจะได้ผลลัพธ์ว่างเปล่า) เหตุผลคือ `NOT EXISTS` ตรวจสอบแค่ "มีแถวที่ `oi.product_id = p.product_id` เป็น `TRUE` จริง ๆ หรือไม่" — แถวที่ `oi.product_id` เป็น `NULL` จะทำให้เงื่อนไข `oi.product_id = p.product_id` เป็น `UNKNOWN` ซึ่งไม่นับว่า "found" แค่นั้นเอง ไม่ได้ทำให้ผลลัพธ์ทั้ง query เพี้ยนไปเหมือนกรณี `NOT IN`

**เปรียบเทียบ EXISTS vs IN — ทำไม EXISTS มักเร็วกว่าเมื่อข้อมูลใหญ่:**

1. **Semi-join กับการหยุดทันทีที่เจอ (short-circuit):** `EXISTS` แค่ต้องการรู้ว่า "มีอย่างน้อย 1 แถวหรือไม่" ดังนั้น query planner สามารถหยุดค้นหาทันทีที่เจอแถวแรกที่ match โดยไม่ต้องดึงข้อมูลทั้งหมดออกมาเหมือน `IN` ที่ (ในทางแนวคิด) ต้องเตรียมรายการทั้งหมดก่อนเปรียบเทียบ
2. **ปลอดภัยจาก NULL โดยธรรมชาติ:** ดังที่แสดงไปแล้ว `NOT EXISTS` ไม่มีปัญหากับ NULL เหมือน `NOT IN`
3. **PostgreSQL optimizer มักแปลง `IN`/`EXISTS` ให้กลายเป็น semi-join/anti-join ภายในเหมือนกัน** ในหลายกรณี performance จึงใกล้เคียงกัน — ข้อได้เปรียบที่ชัดเจนที่สุดของ `EXISTS`/`NOT EXISTS` จึงมักอยู่ที่**ความปลอดภัยเรื่อง NULL** มากกว่าความเร็วดิบ ๆ เสมอไป แต่สำหรับ correlated subquery ที่ซับซ้อน (เช่นมี aggregate หรือเงื่อนไขหลายชั้น) `EXISTS` มักถูก optimize ได้ดีกว่าและคาดเดาพฤติกรรมได้ง่ายกว่า

ตัวอย่างที่ 3: หาพนักงานที่**ไม่เคย**ดูแล order ใดเลย (พนักงานฝ่าย Support ทั้งหมดควรจะไม่มี order เพราะไม่ได้ทำหน้าที่ขาย)

```sql
SELECT e.employee_id, e.first_name, e.last_name, e.department
FROM employees e
WHERE NOT EXISTS (
    SELECT 1 FROM orders o WHERE o.employee_id = e.employee_id
)
ORDER BY e.employee_id;
```

```
 employee_id | first_name | last_name | department
-------------+------------+-----------+------------
           4 | Somsak     | Uraiwan   | Support
           5 | Nattaya    | Chumphon  | Support
           6 | Ekachai    | Boonrod   | Support
           8 | Rungroj    | Sombat    | Support
(4 rows)
```

ตรงตามที่คาดไว้ — พนักงานฝ่าย Support ทุกคน (ยกเว้นที่เป็นผู้จัดการที่อาจดูแล order ด้วย) ไม่มี order ที่ตัวเองดูแลเลย

---

## Step 238: ANY/SOME และ ALL operators ร่วมกับ subquery

`ANY` (คำพ้องความหมายคือ `SOME` — ใช้แทนกันได้ทุกที่) และ `ALL` เป็น operator ที่ใช้ร่วมกับ operator เปรียบเทียบ (`=`, `>`, `<`, `>=`, `<=`, `<>`) เพื่อเทียบค่าหนึ่งกับ**ค่าทุกตัว**หรือ**ค่าอย่างน้อยหนึ่งตัว**ที่ subquery คืนมา

| Operator | ความหมาย |
|---|---|
| `x > ANY (subquery)` | `x` มากกว่าค่า**อย่างน้อยหนึ่งตัว** ในผลลัพธ์ (เทียบเท่า `x > MIN(subquery)`) |
| `x > ALL (subquery)` | `x` มากกว่าค่า**ทุกตัว** ในผลลัพธ์ (เทียบเท่า `x > MAX(subquery)`) |
| `x < ANY (subquery)` | `x` น้อยกว่าค่าอย่างน้อยหนึ่งตัว (เทียบเท่า `x < MAX(subquery)`) |
| `x < ALL (subquery)` | `x` น้อยกว่าค่าทุกตัว (เทียบเท่า `x < MIN(subquery)`) |
| `x = ANY (subquery)` | เทียบเท่ากับ `x IN (subquery)` ทุกประการ |
| `x <> ALL (subquery)` | เทียบเท่ากับ `x NOT IN (subquery)` ทุกประการ (**มีปัญหา NULL แบบเดียวกัน** — ดู Step 234) |

ตัวอย่างที่ 1: หาสินค้าที่ราคาแพงกว่าสินค้า**ทุกชิ้น**ในหมวด Men's Clothing (category_id = 8) ด้วย `ALL`

```sql
SELECT product_name, category_id, unit_price
FROM products
WHERE unit_price > ALL (
    SELECT unit_price FROM products WHERE category_id = 8
)
ORDER BY unit_price DESC
LIMIT 6;
```

```
      product_name        | category_id | unit_price
---------------------------+-------------+------------
 MacBook Air M3 13"        |           7 |   45900.00
 iPhone 15 Pro 128GB       |           6 |   42900.00
 Dell XPS 13               |           7 |   39900.00
 Samsung Galaxy S24        |           6 |   32900.00
 Lenovo ThinkPad E14       |           7 |   24900.00
 Adjustable Dumbbell Set   |           5 |    3990.00
(6 rows)
```

Men's Clothing มีสินค้าราคาสูงสุดคือ 890.00 บาท (`Men's Slim Fit Jeans`) ดังนั้น `unit_price > ALL (...)` จึงเทียบเท่ากับ `unit_price > 890.00` นั่นเอง

ตัวอย่างที่ 2: หาสินค้าที่ราคาถูกกว่าสินค้า**อย่างน้อยหนึ่งชิ้น**ในหมวด Smartphones ด้วย `ANY`

```sql
SELECT product_name, category_id, unit_price
FROM products
WHERE unit_price < ANY (
    SELECT unit_price FROM products WHERE category_id = 6
)
ORDER BY unit_price;
```

```
        product_name         | category_id | unit_price
------------------------------+-------------+------------
 Men's Cotton Polo Shirt      |           8 |     590.00
 Korean Vitamin C Serum       |          11 |     590.00
 Yoga Mat Premium             |           5 |     690.00
 Nonstick Frying Pan 28cm     |          10 |     690.00
 Men's Slim Fit Jeans         |           8 |     890.00
 Clean Code (Book)            |           4 |     890.00
 Women's Summer Dress         |           9 |     790.00
 Women's Silk Blouse          |           9 |    1290.00
 Japanese Rice Cooker 1.8L    |          10 |    2590.00
 Xiaomi Redmi Note 13         |           6 |    6990.00
 Stainless Steel Knife Set    |          10 |    1990.00
 Adjustable Dumbbell Set      |           5 |    3990.00
 Atomic Habits (Book)         |           4 |     450.00
 Logitech MX Master 3S Mouse  |           7 |    3290.00
 Wooden Building Blocks Set   |          12 |     990.00
(15 rows)
```

Smartphones มีราคาสูงสุดถึง 42,900.00 บาท (`iPhone 15 Pro`) เพราะฉะนั้นสินค้าเกือบทั้งหมดในระบบ (ยกเว้นสมาร์ตโฟนและโน้ตบุ๊กที่แพงกว่า) จะ "ถูกกว่าสินค้าอย่างน้อยหนึ่งชิ้นในหมวด Smartphones"

ตัวอย่างที่ 3: `= ANY` เทียบเท่ากับ `IN` ทุกประการ

```sql
-- สองคำสั่งนี้ให้ผลลัพธ์เหมือนกันทุกประการ
SELECT product_name FROM products WHERE category_id = ANY (SELECT category_id FROM categories WHERE category_name LIKE '%Clothing%');
SELECT product_name FROM products WHERE category_id IN    (SELECT category_id FROM categories WHERE category_name LIKE '%Clothing%');
```

**ข้อควรระวัง:** `<> ALL` มีปัญหาเรื่อง `NULL` เหมือนกับ `NOT IN` ทุกประการ เพราะโดยนิยามแล้ว `x <> ALL (subquery)` ก็คือ `x NOT IN (subquery)` เพียงแค่เขียนคนละรูปแบบ ดังนั้นถ้า subquery คืนค่า `NULL` มาแม้แต่แถวเดียว ผลลัพธ์ทั้ง query จะว่างเปล่าเช่นเดียวกัน วิธีแก้ก็เหมือนกัน คือกรอง `IS NOT NULL` หรือเปลี่ยนไปใช้ `NOT EXISTS` แทน

---

## Step 239: เปรียบเทียบ Subquery กับ JOIN

โจทย์หลายข้อสามารถแก้ได้ทั้งด้วย subquery และด้วย `JOIN` คำถามคือ "เมื่อไหร่ควรใช้แบบไหน" และ "ผลลัพธ์เทียบเท่ากันเสมอไปหรือไม่"

### กรณีที่ผลลัพธ์ "เทียบเท่ากัน" แต่เขียนต่างวิธี

หาลูกค้าที่เคยสั่งซื้อสินค้า — เขียนได้ 3 แบบ ให้ผลลัพธ์ชุดแถว (ชื่อลูกค้า) เหมือนกัน:

```sql
-- แบบที่ 1: EXISTS (correlated subquery)
SELECT c.customer_id, c.first_name, c.last_name
FROM customers c
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.customer_id);

-- แบบที่ 2: IN (table subquery)
SELECT c.customer_id, c.first_name, c.last_name
FROM customers c
WHERE c.customer_id IN (SELECT customer_id FROM orders);

-- แบบที่ 3: JOIN + DISTINCT
SELECT DISTINCT c.customer_id, c.first_name, c.last_name
FROM customers c
JOIN orders o ON o.customer_id = c.customer_id;
```

ทั้ง 3 แบบให้ผลลัพธ์ชุดข้อมูลเหมือนกัน **แต่สังเกตว่าแบบที่ 3 ต้องใส่ `DISTINCT`** เพราะลูกค้า 1 คนอาจมีหลาย order — ถ้าลืมใส่ `DISTINCT` ลูกค้าที่มีหลาย order จะปรากฏซ้ำหลายแถว! นี่คือข้อแตกต่างเชิงพฤติกรรมที่สำคัญที่สุดระหว่าง subquery (`EXISTS`/`IN`) กับ `JOIN`:

> **หลักการสำคัญ:** `EXISTS`/`IN` ทำหน้าที่เป็น **semi-join** — กรองแถวของ outer query โดยไม่เพิ่มจำนวนแถว (แต่ละแถวของตารางหลักปรากฏได้อย่างมากแค่ 1 ครั้ง) ในขณะที่ `JOIN` แบบปกติจะคูณจำนวนแถวตามความสัมพันธ์ one-to-many เสมอ ถ้าไม่ต้องการดึงคอลัมน์จากตารางที่ join มาแสดงผล และแค่ต้องการ "เช็คว่ามีอยู่หรือไม่" การใช้ `EXISTS`/`IN` จะปลอดภัยและตรงเจตนากว่า `JOIN + DISTINCT` เสมอ

ลองดูตัวอย่างที่แสดงปัญหานี้ชัดเจน: หาลูกค้าที่มี order (ไม่ใส่ DISTINCT)

```sql
SELECT c.customer_id, c.first_name, count(*) AS row_count
FROM customers c
JOIN orders o ON o.customer_id = c.customer_id
GROUP BY c.customer_id, c.first_name
HAVING count(*) > 1
ORDER BY c.customer_id;
```

```
 customer_id | first_name | row_count
-------------+------------+-----------
           1 | Somchai    |         3
           2 | Malee      |         2
           3 | Anan       |         2
           5 | Weerawat   |         2
(4 rows)
```

ถ้าเราเขียน `SELECT c.customer_id, c.first_name FROM customers c JOIN orders o ...` โดยไม่ใส่ `GROUP BY`/`DISTINCT` ลูกค้าเหล่านี้จะปรากฏซ้ำตามจำนวน order — นี่คือสาเหตุที่การใช้ `EXISTS` จึงปลอดภัยกว่าเมื่อโจทย์คือ "เช็คการมีอยู่" ล้วน ๆ

### กรณีที่ subquery และ JOIN ให้ผลลัพธ์ต่างกัน

ตัวอย่างการหา "ลูกค้าที่ยังไม่เคยสั่งซื้อสินค้าในหมวด Books" — ถ้าใช้ `LEFT JOIN` แบบไม่ระวัง เงื่อนไขกรองที่ใส่ผิดตำแหน่ง (`WHERE` แทน `ON`) จะทำให้ `LEFT JOIN` กลายเป็น `INNER JOIN` โดยไม่ตั้งใจ (ทบทวนได้จาก Part 022) ในขณะที่ `NOT EXISTS`/`NOT IN` ไม่มีปัญหานี้เพราะเป็น subquery อิสระ:

```sql
-- ผิด! WHERE oi.product_id IS NOT NULL ทำให้ LEFT JOIN กลายเป็นทำงานเหมือน INNER JOIN
SELECT DISTINCT c.customer_id, c.first_name
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.customer_id
LEFT JOIN order_items oi ON oi.order_id = o.order_id
    AND oi.product_id IN (SELECT product_id FROM products WHERE category_id = 4)
WHERE oi.product_id IS NOT NULL;  -- นี่คือบั๊ก ตรงข้ามกับสิ่งที่ต้องการ
```

Query ข้างบนคือ query ที่**ตั้งใจผิด** (แสดงเป็นตัวอย่างเตือน) เพราะจะได้ลูกค้าที่**เคย**ซื้อหนังสือ ไม่ใช่ลูกค้าที่**ไม่เคย**ซื้อ วิธีที่ถูกต้องและอ่านง่ายที่สุดคือใช้ `NOT EXISTS`:

```sql
SELECT c.customer_id, c.first_name, c.last_name
FROM customers c
WHERE NOT EXISTS (
    SELECT 1
    FROM orders o
    JOIN order_items oi ON oi.order_id = o.order_id
    JOIN products p ON p.product_id = oi.product_id
    WHERE o.customer_id = c.customer_id
      AND p.category_id = 4   -- Books
)
ORDER BY c.customer_id;
```

```
 customer_id | first_name |  last_name
-------------+------------+-------------
           2 | Malee      | Suksan
           3 | Anan       | Wongsa
           5 | Weerawat   | Boonmee
           6 | Siriporn   | Intharat
           7 | John       | Anderson
           8 | Emily      | Clark
           9 | Yuki       | Tanaka
          10 | Li         | Wei
          12 | Nutthida   | Saelim
          13 | David      | Miller
          14 | Chanya     | Phromma
          15 | Piti       | Suwannaphum
(12 rows)
```

### ตารางสรุปแนวทางตัดสินใจ

| สถานการณ์ | แนะนำให้ใช้ |
|---|---|
| ต้องการคอลัมน์จากหลายตารางมาแสดงพร้อมกัน | `JOIN` |
| แค่ต้องการเช็คว่า "มี/ไม่มี" ความสัมพันธ์ ไม่ต้องการคอลัมน์จากตารางอื่น | `EXISTS` / `NOT EXISTS` |
| ต้องการเปรียบเทียบค่ากับผลสรุป (aggregate) ของอีกตาราง/หมวดหมู่ | scalar subquery (`WHERE x > (SELECT ...)`) |
| ต้องการ aggregate ก่อนแล้วค่อย join/กรองต่อ | derived table ใน `FROM` หรือ CTE (Part 025) |
| ความสัมพันธ์เป็น one-to-many และอยากได้ผลลัพธ์ไม่ซ้ำแถว | `EXISTS`/`IN` (ปลอดภัยกว่า `JOIN + DISTINCT`) |
| คอลัมน์ที่เทียบมีโอกาสเป็น `NULL` และต้องการ "not in" | `NOT EXISTS` เสมอ (หลีกเลี่ยง `NOT IN`) |

โดยทั่วไป PostgreSQL query planner ฉลาดพอที่จะ rewrite `IN`/`EXISTS` subquery จำนวนมากให้กลายเป็น join ภายใน (semi-join/anti-join) โดยอัตโนมัติอยู่แล้ว ดังนั้นเรื่อง performance มักไม่ใช่ปัจจัยตัดสินหลักในการเลือกระหว่าง subquery กับ JOIN — **ความถูกต้องของผลลัพธ์และความอ่านง่ายของโค้ด**ต่างหากที่ควรเป็นตัวตัดสินใจหลัก

---

## Step 240: แบบฝึกหัดรวม — แก้โจทย์ analytics จริงด้วย subquery หลายรูปแบบ

มาลองแก้โจทย์วิเคราะห์ข้อมูลจริงที่ต้องผสมผสานเทคนิคจากทุก step ที่ผ่านมาเข้าด้วยกัน

### โจทย์ที่ 1: หาสินค้าขายดีกว่าค่าเฉลี่ยของหมวดหมู่ตัวเอง (correlated subquery + derived table)

หาสินค้าที่มียอดขาย (จำนวนชิ้นที่ขายได้รวม) สูงกว่ายอดขายเฉลี่ยของสินค้าในหมวดหมู่เดียวกัน

```sql
SELECT
    p.product_name,
    cat.category_name,
    sold.total_qty
FROM products p
JOIN categories cat ON cat.category_id = p.category_id
JOIN (
    SELECT product_id, SUM(quantity) AS total_qty
    FROM order_items
    WHERE product_id IS NOT NULL
    GROUP BY product_id
) sold ON sold.product_id = p.product_id
WHERE sold.total_qty > (
    SELECT AVG(oi2.total_qty)
    FROM (
        SELECT p2.product_id, SUM(oi.quantity) AS total_qty
        FROM products p2
        JOIN order_items oi ON oi.product_id = p2.product_id
        WHERE p2.category_id = p.category_id       -- correlated กับ p ภายนอก
        GROUP BY p2.product_id
    ) oi2
)
ORDER BY cat.category_name, sold.total_qty DESC;
```

```
   category_name     |     product_name      | total_qty
----------------------+------------------------+-----------
 Beauty & Health      | Korean Vitamin C Serum |         3
 Books                | Atomic Habits (Book)   |         3
 Kitchenware          | Nonstick Frying Pan 28cm|         2
 Men's Clothing       | Men's Cotton Polo Shirt|         5
 Smartphones          | iPhone 15 Pro 128GB    |         2
 Sports & Outdoor     | Adjustable Dumbbell Set|         1
 Women's Clothing     | Women's Summer Dress   |         3
(7 rows)
```

### โจทย์ที่ 2: หาลูกค้าที่ไม่เคยเขียนรีวิวเลย แต่เคยสั่งซื้อสินค้าแล้ว (NOT EXISTS + EXISTS ผสมกัน)

```sql
SELECT c.customer_id, c.first_name, c.last_name
FROM customers c
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.customer_id)
  AND NOT EXISTS (SELECT 1 FROM reviews r WHERE r.customer_id = c.customer_id)
ORDER BY c.customer_id;
```

```
 customer_id | first_name |  last_name
-------------+------------+-------------
           4 | Pranee     | Chaiyo
           5 | Weerawat   | Boonmee
           7 | John       | Anderson
          10 | Li         | Wei
          11 | Kittipong  | Rattana
          15 | Piti       | Suwannaphum
(6 rows)
```

กลุ่มลูกค้ากลุ่มนี้เหมาะเป็นเป้าหมายของแคมเปญ "ช่วยรีวิวสินค้าแลกส่วนลด" เพราะซื้อสินค้าแล้วแต่ยังไม่เคยให้ feedback

### โจทย์ที่ 3: หาพนักงานขาย (Sales) ที่ทำยอดขายมากกว่าพนักงานขายทุกคนในแผนกเดียวกัน ด้วย ALL

```sql
SELECT
    e.first_name,
    e.last_name,
    sales.total_sales
FROM employees e
JOIN (
    SELECT o.employee_id, SUM(p.amount) AS total_sales
    FROM orders o
    JOIN payments p ON p.order_id = o.order_id
    GROUP BY o.employee_id
) sales ON sales.employee_id = e.employee_id
WHERE sales.total_sales > ALL (
    SELECT SUM(p2.amount)
    FROM orders o2
    JOIN payments p2 ON p2.order_id = o2.order_id
    WHERE o2.employee_id <> e.employee_id
      AND o2.employee_id IN (SELECT employee_id FROM employees WHERE department = 'Sales')
    GROUP BY o2.employee_id
);
```

```
 first_name | last_name | total_sales
------------+-----------+-------------
 Thawatchai | Pongsak   |   157940.00
(1 row)
```

`Thawatchai Pongsak` ทำยอดขายรวมได้มากกว่าพนักงานขาย (Sales) คนอื่น ๆ **ทุกคน** ในบริษัท

### โจทย์ที่ 4: หมวดหมู่สินค้าที่ไม่มีสินค้า active เหลือขายเลยสักชิ้น (NOT EXISTS สำหรับตรวจ business rule)

```sql
SELECT cat.category_id, cat.category_name
FROM categories cat
WHERE NOT EXISTS (
    SELECT 1 FROM products p
    WHERE p.category_id = cat.category_id AND p.is_active = true
)
ORDER BY cat.category_id;
```

```
 category_id | category_name
-------------+----------------
          12 | Toys & Games
(1 row)
```

`Toys & Games` มีสินค้าเดียวในหมวดคือ `Wooden Building Blocks Set` ซึ่งถูกปิดการขาย (`is_active = false`) ไปแล้ว ทำให้ทั้งหมวดหมู่ไม่มีสินค้าเหลือขายเลย — ข้อมูลนี้มีประโยชน์มากสำหรับทีม merchandising ที่ต้องเติมสินค้าเข้าหมวดหมู่นี้

โจทย์ทั้ง 4 ข้อนี้แสดงให้เห็นว่าในงาน analytics จริง เรามักต้อง**ผสมผสาน**เทคนิคหลายแบบเข้าด้วยกันในคำสั่งเดียว ไม่ว่าจะเป็น derived table + correlated subquery, EXISTS ซ้อนกันหลายชั้น, หรือ ALL ร่วมกับ subquery ที่มีเงื่อนไขซับซ้อน — ทักษะสำคัญคือการค่อย ๆ แตกโจทย์ใหญ่เป็นขั้นตอนย่อย เขียนและทดสอบทีละส่วนก่อนประกอบเป็น query สุดท้าย

---

## สรุปท้ายบท

- **Subquery** คือ query ที่ซ้อนอยู่ภายใน query อื่น แบ่งเป็น 3 ประเภทตามรูปร่างผลลัพธ์: **scalar** (1 แถว 1 คอลัมน์), **row** (1 แถว หลายคอลัมน์), และ **table** (หลายแถว)
- Subquery ปรากฏได้ 4 ตำแหน่งหลัก: `SELECT` list, `FROM` clause (derived table — **ต้องมี alias เสมอใน PostgreSQL**), `WHERE`/`HAVING`, และหลัง `IN`/`EXISTS`/`ANY`/`ALL`
- **Scalar subquery** ใช้กับ operator เปรียบเทียบตรง ๆ ได้ (`=`, `>`, `<`) แต่ต้องมั่นใจว่าคืนค่าไม่เกิน 1 แถวเสมอ ไม่งั้นจะเกิด runtime error
- **`IN`/`NOT IN`** ใช้กับ subquery ที่คืนหลายแถวได้ แต่ **`NOT IN` มีข้อควรระวังร้ายแรงเรื่อง NULL** — ถ้า subquery คืนค่า NULL แม้แต่แถวเดียว ผลลัพธ์ทั้ง query จะว่างเปล่าโดยไม่มี error เตือน ต้องกรอง `IS NOT NULL` เสมอ หรือใช้ `NOT EXISTS` แทน
- **Correlated subquery** อ้างอิงคอลัมน์จากตาราง outer query ทำให้ต้องถูกประเมินผลสัมพันธ์กับแต่ละแถวของ outer query ต่างจาก **non-correlated subquery** ที่เป็นอิสระและคำนวณครั้งเดียว
- **`EXISTS`/`NOT EXISTS`** ตรวจสอบการมีอยู่ของแถวโดยไม่สนใจคอลัมน์ที่ subquery คืนมา ปลอดภัยจากปัญหา NULL โดยธรรมชาติ และมักเป็นทางเลือกที่ปลอดภัยกว่า `IN`/`NOT IN` โดยเฉพาะกับ `NOT EXISTS`
- **`ANY`/`SOME`** เทียบค่ากับ**อย่างน้อยหนึ่งค่า**ในผลลัพธ์ ส่วน **`ALL`** เทียบกับ**ทุกค่า** — `= ANY` เทียบเท่า `IN` และ `<> ALL` เทียบเท่า `NOT IN` (พร้อมปัญหา NULL แบบเดียวกัน)
- **Subquery vs JOIN**: `JOIN` เหมาะเมื่อต้องการคอลัมน์จากหลายตารางพร้อมกัน แต่ต้องระวังแถวซ้ำจากความสัมพันธ์ one-to-many (ต้องใส่ `DISTINCT`) ส่วน `EXISTS`/`IN` เหมาะกับการ "เช็คการมีอยู่" เพราะทำงานเป็น semi-join ที่ไม่เพิ่มจำนวนแถวของ outer query
- เมื่อ query ซับซ้อนขึ้นและมี subquery ซ้อนหลายชั้น ควรพิจารณาใช้ **CTE (`WITH`)** เพื่อความอ่านง่าย ซึ่งเป็นหัวข้อของบทถัดไป

บทถัดไป **Part 025: Common Table Expressions (CTE)** จะสอนวิธีเขียน query ที่ซับซ้อนให้อ่านง่ายขึ้นด้วย `WITH`, การใช้ CTE หลายตัวในคำสั่งเดียว, และ CTE ที่ใช้ซ้ำได้หลายครั้งโดยไม่ต้องเขียน subquery เดิมซ้ำ ๆ

**บทถัดไป:** [Part 025: Common Table Expressions (CTE)](./part-025-cte.md)

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1 (ง่าย)

แสดงชื่อสินค้าและราคา สำหรับสินค้าที่มีราคา**ต่ำกว่า**ราคาเฉลี่ยของสินค้าทั้งหมด เรียงจากถูกไปแพง

<details>
<summary>เฉลย</summary>

```sql
SELECT product_name, unit_price
FROM products
WHERE unit_price < (SELECT AVG(unit_price) FROM products)
ORDER BY unit_price;
```

```
       product_name        | unit_price
----------------------------+------------
 Atomic Habits (Book)       |     450.00
 Men's Cotton Polo Shirt    |     590.00
 Korean Vitamin C Serum     |     590.00
 Nonstick Frying Pan 28cm   |     690.00
 Yoga Mat Premium           |     690.00
 Women's Summer Dress       |     790.00
 Men's Slim Fit Jeans       |     890.00
 Clean Code (Book)          |     890.00
 Wooden Building Blocks Set |     990.00
 Women's Silk Blouse        |    1290.00
 Stainless Steel Knife Set  |    1990.00
 Japanese Rice Cooker 1.8L  |    2590.00
 Logitech MX Master 3S Mouse|    3290.00
 Adjustable Dumbbell Set    |    3990.00
 Xiaomi Redmi Note 13       |    6990.00
(15 rows)
```

ราคาเฉลี่ยของสินค้าทั้งหมดคือ 10,574.50 บาท จึงเหลือสินค้าราคาต่ำกว่านี้ 15 ชิ้นจากทั้งหมด 20 ชิ้น
</details>

---

### แบบฝึกหัดที่ 2 (ง่าย)

แสดงชื่อลูกค้าที่ **ยังไม่เคย**สั่งซื้อสินค้าเลยแม้แต่ครั้งเดียว (ใช้ `NOT EXISTS`)

<details>
<summary>เฉลย</summary>

```sql
SELECT customer_id, first_name, last_name
FROM customers c
WHERE NOT EXISTS (
    SELECT 1 FROM orders o WHERE o.customer_id = c.customer_id
)
ORDER BY customer_id;
```

```
 customer_id | first_name | last_name
-------------+------------+-----------
(0 rows)
```

ในชุดข้อมูลตัวอย่างนี้ ลูกค้าทั้ง 15 คนเคยสั่งซื้อสินค้าอย่างน้อยหนึ่งครั้ง จึงได้ผลลัพธ์ว่างเปล่า — เป็นเรื่องปกติและถูกต้อง แสดงว่าไม่มีลูกค้ากลุ่ม "สมัครแล้วแต่ไม่เคยซื้อ" อยู่เลย
</details>

---

### แบบฝึกหัดที่ 3 (ปานกลาง)

แสดงชื่อสินค้าทุกชิ้น พร้อมราคาเฉลี่ยของหมวดหมู่ตัวเอง (ใช้ correlated scalar subquery ใน `SELECT` list) เรียงตามชื่อหมวดหมู่แล้วตามราคาสินค้าจากมากไปน้อย

<details>
<summary>เฉลย</summary>

```sql
SELECT
    cat.category_name,
    p.product_name,
    p.unit_price,
    (SELECT ROUND(AVG(p2.unit_price), 2)
     FROM products p2
     WHERE p2.category_id = p.category_id) AS avg_price_in_category
FROM products p
JOIN categories cat ON cat.category_id = p.category_id
ORDER BY cat.category_name, p.unit_price DESC;
```

ตัวอย่างผลลัพธ์บางส่วน:

```
   category_name    |     product_name      | unit_price | avg_price_in_category
---------------------+------------------------+------------+-------------------------
 Beauty & Health     | Korean Vitamin C Serum |     590.00 |                 590.00
 Books               | Clean Code (Book)      |     890.00 |                 670.00
 Books               | Atomic Habits (Book)   |     450.00 |                 670.00
 Computers & Laptops | MacBook Air M3 13"     |   45900.00 |               18363.33
 ...
```

subquery ในตัวอย่างนี้เป็น correlated subquery เพราะอ้างอิง `p.category_id` จาก outer query
</details>

---

### แบบฝึกหัดที่ 4 (ปานกลาง)

แสดงสินค้าที่มีราคาแพงกว่าสินค้า**ทุกชิ้น**ในหมวด `Books` โดยใช้ `ALL`

<details>
<summary>เฉลย</summary>

```sql
SELECT product_name, unit_price
FROM products
WHERE unit_price > ALL (
    SELECT unit_price
    FROM products p
    JOIN categories c ON c.category_id = p.category_id
    WHERE c.category_name = 'Books'
)
ORDER BY unit_price DESC;
```

```
       product_name        | unit_price
----------------------------+------------
 MacBook Air M3 13"         |   45900.00
 iPhone 15 Pro 128GB        |   42900.00
 Dell XPS 13                |   39900.00
 Samsung Galaxy S24         |   32900.00
 Lenovo ThinkPad E14        |   24900.00
 Xiaomi Redmi Note 13       |    6990.00
 Adjustable Dumbbell Set    |    3990.00
 Logitech MX Master 3S Mouse|    3290.00
 Japanese Rice Cooker 1.8L  |    2590.00
 Stainless Steel Knife Set  |    1990.00
 Women's Silk Blouse        |    1290.00
 Wooden Building Blocks Set |     990.00
(12 rows)
```

หนังสือที่แพงที่สุดในหมวด Books คือ `Clean Code` ราคา 890.00 บาท ดังนั้นสินค้าใดก็ตามที่ราคาสูงกว่า 890.00 บาท จะเข้าเงื่อนไขนี้ทั้งหมด
</details>

---

### แบบฝึกหัดที่ 5 (ปานกลาง)

แสดงชื่อลูกค้าที่เคยสั่งซื้อสินค้าในหมวด `Smartphones` (ใช้ `IN` ซ้อนกันสองชั้น: orders → order_items → products → categories)

<details>
<summary>เฉลย</summary>

```sql
SELECT DISTINCT c.customer_id, c.first_name, c.last_name
FROM customers c
WHERE c.customer_id IN (
    SELECT o.customer_id
    FROM orders o
    WHERE o.order_id IN (
        SELECT oi.order_id
        FROM order_items oi
        WHERE oi.product_id IN (
            SELECT p.product_id
            FROM products p
            WHERE p.category_id IN (
                SELECT category_id FROM categories WHERE category_name = 'Smartphones'
            )
        )
    )
)
ORDER BY c.customer_id;
```

```
 customer_id | first_name | last_name
-------------+------------+-----------
           1 | Somchai    | Jaidee
           2 | Malee      | Suksan
           7 | John       | Anderson
(3 rows)
```

query นี้จงใจซ้อน `IN` หลายชั้นเพื่อฝึกฝน — ในทางปฏิบัติเขียนให้กระชับขึ้นได้ด้วย `JOIN` หรือ `EXISTS` เช่น
```sql
SELECT DISTINCT c.customer_id, c.first_name, c.last_name
FROM customers c
JOIN orders o ON o.customer_id = c.customer_id
JOIN order_items oi ON oi.order_id = o.order_id
JOIN products p ON p.product_id = oi.product_id
JOIN categories cat ON cat.category_id = p.category_id
WHERE cat.category_name = 'Smartphones'
ORDER BY c.customer_id;
```
ซึ่งให้ผลลัพธ์เดียวกัน แต่อ่านง่ายกว่ามาก — เป็นตัวอย่างที่ดีของหลักการใน Step 239
</details>

---

### แบบฝึกหัดที่ 6 (ปานกลาง)

แสดง supplier ที่**ไม่มี**สินค้าใด ๆ ของตัวเองอยู่ในระบบเลย (ใช้ `NOT EXISTS`)

<details>
<summary>เฉลย</summary>

```sql
SELECT supplier_id, supplier_name, country
FROM suppliers s
WHERE NOT EXISTS (
    SELECT 1 FROM products p WHERE p.supplier_id = s.supplier_id
)
ORDER BY supplier_id;
```

```
 supplier_id | supplier_name | country
-------------+---------------+---------
(0 rows)
```

ในชุดข้อมูลนี้ supplier ทั้ง 8 รายมีสินค้าอย่างน้อย 1 ชิ้นในระบบ จึงได้ผลลัพธ์ว่างเปล่า ลองเปลี่ยนไปหา supplier ที่มีสินค้า `is_active = false` ทั้งหมด (ไม่มีสินค้า active เหลือเลย) เป็นแบบฝึกเพิ่มเติม:
```sql
SELECT supplier_id, supplier_name
FROM suppliers s
WHERE EXISTS (SELECT 1 FROM products p WHERE p.supplier_id = s.supplier_id)
  AND NOT EXISTS (SELECT 1 FROM products p WHERE p.supplier_id = s.supplier_id AND p.is_active = true);
```
ผลลัพธ์: `Nordic Home Supplies` (supplier_id = 3) เพราะสินค้าเดียวของ supplier นี้คือ `Wooden Building Blocks Set` ซึ่งถูกปิดการขายแล้ว
</details>

---

### แบบฝึกหัดที่ 7 (ยาก)

แสดงพนักงานฝ่ายขาย (department = 'Sales') ที่ทำยอดขายรวม (จาก `payments` ที่ผูกกับ order ที่ตัวเองดูแล) **มากกว่าค่าเฉลี่ย**ยอดขายของพนักงานขายทุกคน (ใช้ derived table ใน `FROM` ร่วมกับ scalar subquery)

<details>
<summary>เฉลย</summary>

```sql
SELECT e.first_name, e.last_name, sales.total_sales
FROM employees e
JOIN (
    SELECT o.employee_id, SUM(p.amount) AS total_sales
    FROM orders o
    JOIN payments p ON p.order_id = o.order_id
    GROUP BY o.employee_id
) sales ON sales.employee_id = e.employee_id
WHERE e.department = 'Sales'
  AND sales.total_sales > (
      SELECT AVG(sub.total_sales)
      FROM (
          SELECT o2.employee_id, SUM(p2.amount) AS total_sales
          FROM orders o2
          JOIN payments p2 ON p2.order_id = o2.order_id
          JOIN employees e2 ON e2.employee_id = o2.employee_id
          WHERE e2.department = 'Sales'
          GROUP BY o2.employee_id
      ) sub
  )
ORDER BY sales.total_sales DESC;
```

```
 first_name | last_name | total_sales
------------+-----------+-------------
 Thawatchai | Pongsak   |   157940.00
```

พนักงานขายที่ทำยอดขายรวมสูงกว่าค่าเฉลี่ยของพนักงานขายทุกคนมีเพียงคนเดียวคือ `Thawatchai Pongsak`
</details>

---

### แบบฝึกหัดที่ 8 (ยาก)

แสดงสินค้าที่**ไม่เคย**ถูกรีวิวเลย แต่**เคย**ถูกสั่งซื้อแล้วอย่างน้อย 1 ครั้ง (ผสม `EXISTS` กับ `NOT EXISTS`)

<details>
<summary>เฉลย</summary>

```sql
SELECT p.product_id, p.product_name
FROM products p
WHERE EXISTS (
    SELECT 1 FROM order_items oi WHERE oi.product_id = p.product_id
)
AND NOT EXISTS (
    SELECT 1 FROM reviews r WHERE r.product_id = p.product_id
)
ORDER BY p.product_id;
```

```
 product_id |       product_name
------------+----------------------------
          6 | Lenovo ThinkPad E14
         11 | Women's Silk Blouse
         14 | Stainless Steel Knife Set
         18 | Adjustable Dumbbell Set
(4 rows)
```

สินค้ากลุ่มนี้เหมาะเป็นเป้าหมายของอีเมล "ช่วยรีวิวสินค้าที่คุณเคยซื้อ" เพราะมีคนซื้อจริงแต่ยังไม่มีใครรีวิวเลย
</details>

---

### แบบฝึกหัดที่ 9 (ยาก)

หา order (ที่ไม่ถูกยกเลิก) ที่มียอดรวมจาก `order_items` (quantity × unit_price รวมทุกรายการในออเดอร์) **มากกว่า**ยอดเฉลี่ยของทุกออเดอร์ โดยคำนวณยอดต่อออเดอร์ด้วย derived table ใน `FROM` ก่อน แล้วเทียบกับ scalar subquery ของค่าเฉลี่ย

<details>
<summary>เฉลย</summary>

```sql
SELECT
    ord_total.order_id,
    c.first_name,
    c.last_name,
    ord_total.order_total
FROM (
    SELECT o.order_id, o.customer_id, SUM(oi.quantity * oi.unit_price) AS order_total
    FROM orders o
    JOIN order_items oi ON oi.order_id = o.order_id
    WHERE o.status <> 'cancelled'
    GROUP BY o.order_id, o.customer_id
) ord_total
JOIN customers c ON c.customer_id = ord_total.customer_id
WHERE ord_total.order_total > (
    SELECT AVG(t.order_total)
    FROM (
        SELECT o2.order_id, SUM(oi2.quantity * oi2.unit_price) AS order_total
        FROM orders o2
        JOIN order_items oi2 ON oi2.order_id = o2.order_id
        WHERE o2.status <> 'cancelled'
        GROUP BY o2.order_id
    ) t
)
ORDER BY ord_total.order_total DESC;
```

```
 order_id | first_name | last_name | order_total
----------+------------+-----------+--------------
        1 | Somchai    | Jaidee    |     46290.00
       10 | Malee      | Suksan    |     42900.00
       12 | Li         | Wei       |     24900.00
       15 | Nutthida   | Saelim    |     39900.00
        7 | John       | Anderson  |     13980.00
        2 | Malee      | Suksan    |     32900.00
       16 | Somchai    | Jaidee    |     32900.00
(7 rows)
```
</details>

---

### แบบฝึกหัดที่ 10 (ขั้นสูงมาก)

หาลูกค้าที่**ไม่เคย**สั่งซื้อสินค้าในหมวด `Electronics` (รวมหมวดย่อย `Smartphones` และ `Computers & Laptops` ด้วย) โดยเขียน query เดียวกัน **3 วิธี**: (a) `NOT IN` ที่ป้องกัน NULL อย่างถูกต้อง, (b) `NOT EXISTS`, (c) `LEFT JOIN ... WHERE ... IS NULL` แล้วอธิบายว่าทำไมวิธี (a) ต้อง filter NULL ก่อนเสมอ

<details>
<summary>เฉลย</summary>

ก่อนอื่นต้องหาหมวด Electronics และหมวดย่อยทั้งหมดก่อน (parent + children ผ่าน `parent_category_id`):

```sql
-- หมวดที่เกี่ยวข้อง: Electronics เอง (id=1) รวมถึงหมวดย่อย Smartphones(6), Computers & Laptops(7)
SELECT category_id FROM categories WHERE category_id = 1 OR parent_category_id = 1;
-- ได้ 1, 6, 7
```

**(a) วิธี `NOT IN` — ต้อง filter NULL ก่อนเสมอ**

```sql
SELECT c.customer_id, c.first_name, c.last_name
FROM customers c
WHERE c.customer_id NOT IN (
    SELECT o.customer_id
    FROM orders o
    JOIN order_items oi ON oi.order_id = o.order_id
    JOIN products p ON p.product_id = oi.product_id
    WHERE p.category_id IN (1, 6, 7)
      AND o.customer_id IS NOT NULL   -- ป้องกัน NULL ปนใน subquery (แม้ในเคสนี้ orders.customer_id ไม่ค่อยเป็น NULL แต่เขียนเผื่อไว้เป็น defensive practice)
)
ORDER BY c.customer_id;
```

**(b) วิธี `NOT EXISTS` — ปลอดภัยจาก NULL โดยธรรมชาติ ไม่ต้องกรองเพิ่ม**

```sql
SELECT c.customer_id, c.first_name, c.last_name
FROM customers c
WHERE NOT EXISTS (
    SELECT 1
    FROM orders o
    JOIN order_items oi ON oi.order_id = o.order_id
    JOIN products p ON p.product_id = oi.product_id
    WHERE o.customer_id = c.customer_id
      AND p.category_id IN (1, 6, 7)
)
ORDER BY c.customer_id;
```

**(c) วิธี `LEFT JOIN ... WHERE ... IS NULL`**

```sql
SELECT DISTINCT c.customer_id, c.first_name, c.last_name
FROM customers c
LEFT JOIN (
    SELECT DISTINCT o.customer_id
    FROM orders o
    JOIN order_items oi ON oi.order_id = o.order_id
    JOIN products p ON p.product_id = oi.product_id
    WHERE p.category_id IN (1, 6, 7)
) electronics_buyers ON electronics_buyers.customer_id = c.customer_id
WHERE electronics_buyers.customer_id IS NULL
ORDER BY c.customer_id;
```

ทั้ง 3 วิธีให้ผลลัพธ์เหมือนกัน:

```
 customer_id | first_name |  last_name
-------------+------------+-------------
           4 | Pranee     | Chaiyo
           6 | Siriporn   | Intharat
           8 | Emily      | Clark
           9 | Yuki       | Tanaka
          11 | Kittipong  | Rattana
          12 | Nutthida   | Saelim
          13 | David      | Miller
          15 | Piti       | Suwannaphum
(8 rows)
```

**คำอธิบายว่าทำไม `NOT IN` ต้อง filter NULL ก่อนเสมอ:** ในทางตรรกะ `NOT IN (subquery)` จะถูกแปลงภายในเป็น `NOT (x = v1 OR x = v2 OR ... OR x = vN)` ถ้าใน subquery มีค่า `vi` ตัวใดตัวหนึ่งเป็น `NULL` การเปรียบเทียบ `x = NULL` จะได้ผลเป็น `UNKNOWN` เสมอ (ไม่ใช่ `TRUE`/`FALSE`) และเมื่อ `OR` ที่มี `UNKNOWN` ปนอยู่ถูกนำไป `NOT` ผลลัพธ์สุดท้ายจะกลายเป็น `UNKNOWN` เสมอไม่ว่า `x` จะเป็นอะไร ทำให้แถวนั้นถูก `WHERE` กรองทิ้งไปโดยอัตโนมัติ — ผลลัพธ์คือ `NOT IN` ทั้ง query จะคืนค่าว่างเปล่าถ้า subquery มี `NULL` แม้แต่ตัวเดียว โดยไม่มี error ใด ๆ เตือนให้รู้ตัว นี่คือเหตุผลที่ `NOT EXISTS` (วิธี b) และ `LEFT JOIN ... IS NULL` (วิธี c) จึงเป็นทางเลือกที่ปลอดภัยกว่า `NOT IN` เสมอเมื่อทำงานกับข้อมูลจริงที่ไม่สามารถรับประกันได้ 100% ว่าจะไม่มี `NULL` ปนอยู่
</details>

---

**บทถัดไป:** [Part 025: Common Table Expressions (CTE)](./part-025-cte.md)
