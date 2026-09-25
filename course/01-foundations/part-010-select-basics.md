# Part 010: SELECT พื้นฐาน — เลือกคอลัมน์, Alias, DISTINCT

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับพื้นฐาน | Part 010

คำสั่ง `SELECT` คือหัวใจของ SQL และเป็นคำสั่งที่คุณจะใช้บ่อยที่สุดตลอดการทำงานกับฐานข้อมูล ใน Part นี้เราจะเริ่มต้นจากพื้นฐานที่สุดของ `SELECT` — การเลือกคอลัมน์ การตั้งชื่อ alias การคำนวณค่าใน SELECT list ไปจนถึงฟีเจอร์เด่นเฉพาะของ PostgreSQL อย่าง `DISTINCT ON` ซึ่งเป็นเครื่องมือที่ทรงพลังมากและหาไม่ได้ในฐานข้อมูลอื่น

## เป้าหมายการเรียนรู้

เมื่อจบ Part นี้ คุณจะสามารถ:

1. เขียนคำสั่ง `SELECT` พื้นฐานได้อย่างถูกต้อง และเข้าใจว่าทำไมไม่ควรใช้ `SELECT *` ใน production code
2. ตั้งชื่อ alias ให้คอลัมน์และตารางด้วย `AS` (และรู้วิธีเขียนแบบไม่ใช้ `AS`)
3. ใช้ expression ทางคณิตศาสตร์และการต่อสตริง (`||`) ภายใน SELECT list
4. ใช้ `DISTINCT` เพื่อกรองแถวที่ซ้ำกัน ทั้งแบบคอลัมน์เดียวและหลายคอลัมน์
5. ใช้ `DISTINCT ON` ซึ่งเป็นฟีเจอร์เฉพาะของ PostgreSQL เพื่อแก้ปัญหาจริง เช่น หา order ล่าสุดของลูกค้าแต่ละคน
6. ใส่ literal values (ตัวเลข ข้อความ NULL) ลงใน SELECT list
7. ตั้ง alias ให้ตารางเพื่อเตรียมพร้อมสำหรับการทำ JOIN ใน Part ถัดไป
8. เข้าใจแนวคิด schema-qualified name และเขียน query ข้าม schema ได้
9. เขียน comment ใน SQL และจัดรูปแบบ query ให้อ่านง่ายตามมาตรฐานสากล

---

## เตรียมข้อมูล

Part นี้ยังคงใช้ schema "ร้านกาแฟ" ที่เราออกแบบไว้ใน Part 008–009 ต่อเนื่องกัน เพื่อให้ตัวอย่างทุกอันรันได้จริงในฐานข้อมูลทดสอบของคุณ ให้รันสคริปต์ด้านล่างนี้ก่อน (สคริปต์นี้ลบตารางเดิมทิ้งแล้วสร้างใหม่พร้อมข้อมูลตัวอย่าง เพื่อให้ผลลัพธ์ของทุก query ในบทนี้ตรงกับที่แสดงไว้)

```sql
-- ลบตารางเดิม (ถ้ามี) เพื่อให้สคริปต์รันซ้ำได้
DROP TABLE IF EXISTS order_items CASCADE;
DROP TABLE IF EXISTS orders CASCADE;
DROP TABLE IF EXISTS products CASCADE;
DROP TABLE IF EXISTS customers CASCADE;
DROP TABLE IF EXISTS categories CASCADE;

-- ตารางหมวดหมู่สินค้า
CREATE TABLE categories (
    category_id   SERIAL PRIMARY KEY,
    category_name VARCHAR(50) NOT NULL,
    description   TEXT
);

-- ตารางสินค้า
CREATE TABLE products (
    product_id    SERIAL PRIMARY KEY,
    product_name  VARCHAR(100) NOT NULL,
    category_id   INT REFERENCES categories(category_id),
    price         NUMERIC(10, 2) NOT NULL,
    cost          NUMERIC(10, 2),
    is_active     BOOLEAN DEFAULT true,
    created_at    TIMESTAMP DEFAULT now()
);

-- ตารางลูกค้า
CREATE TABLE customers (
    customer_id   SERIAL PRIMARY KEY,
    first_name    VARCHAR(50) NOT NULL,
    last_name     VARCHAR(50) NOT NULL,
    email         VARCHAR(100) UNIQUE,
    phone         VARCHAR(20),
    city          VARCHAR(50),
    member_since  DATE DEFAULT CURRENT_DATE
);

-- ตารางคำสั่งซื้อ
CREATE TABLE orders (
    order_id      SERIAL PRIMARY KEY,
    customer_id   INT REFERENCES customers(customer_id),
    order_date    TIMESTAMP DEFAULT now(),
    status        VARCHAR(20) DEFAULT 'pending',
    total_amount  NUMERIC(10, 2)
);

-- ตารางรายการสินค้าในแต่ละคำสั่งซื้อ
CREATE TABLE order_items (
    order_item_id SERIAL PRIMARY KEY,
    order_id      INT REFERENCES orders(order_id),
    product_id    INT REFERENCES products(product_id),
    quantity      INT NOT NULL,
    unit_price    NUMERIC(10, 2) NOT NULL
);
```

จากนั้นเติมข้อมูลตัวอย่าง:

```sql
-- หมวดหมู่สินค้า
INSERT INTO categories (category_name, description) VALUES
    ('Hot Coffee',   'เครื่องดื่มกาแฟร้อน'),
    ('Cold Coffee',  'เครื่องดื่มกาแฟเย็นและปั่น'),
    ('Tea',          'เครื่องดื่มชา'),
    ('Bakery',       'เบเกอรี่และของทานเล่น'),
    ('Merchandise',  'สินค้าที่ระลึก');

-- สินค้า 10 รายการ
INSERT INTO products (product_name, category_id, price, cost, is_active) VALUES
    ('Espresso',         1, 55.00, 18.00, true),
    ('Americano',        1, 60.00, 20.00, true),
    ('Cappuccino',       1, 70.00, 25.00, true),
    ('Latte',            1, 70.00, 25.00, true),
    ('Iced Latte',       2, 75.00, 27.00, true),
    ('Iced Americano',   2, 65.00, 22.00, true),
    ('Frappe Caramel',   2, 95.00, 35.00, true),
    ('Thai Milk Tea',    3, 65.00, 20.00, true),
    ('Butter Croissant', 4, 45.00, 15.00, true),
    ('Blueberry Muffin', 4, 50.00, 18.00, false);

-- ลูกค้า 9 คน
INSERT INTO customers (first_name, last_name, email, phone, city, member_since) VALUES
    ('สมชาย',   'ใจดี',     'somchai.j@example.com', '081-111-1111', 'กรุงเทพฯ',      '2023-01-15'),
    ('สมหญิง',  'รักเรียน', 'somying.r@example.com', '081-222-2222', 'เชียงใหม่',     '2023-02-20'),
    ('วิชัย',    'มั่งมี',    'wichai.m@example.com',  '081-333-3333', 'กรุงเทพฯ',      '2023-03-10'),
    ('มานี',     'มีนา',     'manee.m@example.com',   NULL,           'ขอนแก่น',       '2023-04-05'),
    ('ประยุทธ', 'ยงยุทธ',   'prayuth.y@example.com', '081-555-5555', 'กรุงเทพฯ',      '2023-05-18'),
    ('สุดา',     'สุขใจ',    'suda.s@example.com',    '081-666-6666', 'ภูเก็ต',        '2023-06-22'),
    ('อนันต์',   'อนุกูล',   NULL,                    '081-777-7777', 'เชียงใหม่',     '2023-07-30'),
    ('พิมพ์ใจ', 'พิมพ์ดี',  'pimjai.p@example.com',  '081-888-8888', 'กรุงเทพฯ',      '2023-08-14'),
    ('ธนาคาร',  'ธนกิจ',    'tanakan.t@example.com', '081-999-9999', 'นครราชสีมา',    '2023-09-01');

-- คำสั่งซื้อ 11 รายการ
INSERT INTO orders (customer_id, order_date, status, total_amount) VALUES
    (1, '2024-01-05 08:30:00', 'completed', 130.00),
    (1, '2024-01-12 09:15:00', 'completed',  70.00),
    (1, '2024-02-01 10:00:00', 'completed', 195.00),
    (2, '2024-01-08 14:20:00', 'completed',  65.00),
    (2, '2024-01-20 16:45:00', 'cancelled',  75.00),
    (3, '2024-01-10 07:50:00', 'completed', 240.00),
    (4, '2024-01-15 11:30:00', 'completed',  65.00),
    (5, '2024-01-18 13:10:00', 'completed', 115.00),
    (5, '2024-02-02 09:40:00', 'completed',  70.00),
    (6, '2024-01-25 15:00:00', 'completed',  95.00),
    (1, '2024-02-10 08:00:00', 'pending',    60.00);

-- รายการสินค้าในคำสั่งซื้อ (บางส่วน สำหรับใช้อ้างอิงใน Part ถัดไป)
INSERT INTO order_items (order_id, product_id, quantity, unit_price) VALUES
    (1,  3, 1, 70.00),
    (1,  9, 1, 45.00),
    (1,  1, 1, 55.00),
    (2,  4, 1, 70.00),
    (3,  5, 1, 75.00),
    (3,  7, 1, 95.00),
    (3,  9, 1, 45.00),
    (4,  8, 1, 65.00),
    (5,  6, 1, 65.00),
    (6,  7, 2, 95.00),
    (7,  8, 1, 65.00),
    (8,  4, 1, 70.00),
    (8,  1, 1, 55.00),
    (9,  6, 1, 65.00),
    (10, 5, 1, 75.00);
```

ตรวจสอบว่าข้อมูลเข้าครบถ้วน:

```sql
SELECT
    (SELECT count(*) FROM categories)  AS categories_count,
    (SELECT count(*) FROM products)    AS products_count,
    (SELECT count(*) FROM customers)   AS customers_count,
    (SELECT count(*) FROM orders)      AS orders_count,
    (SELECT count(*) FROM order_items) AS order_items_count;
```

```
 categories_count | products_count | customers_count | orders_count | order_items_count
-------------------+----------------+------------------+--------------+--------------------
                 5 |             10 |                9 |           11 |                 15
(1 row)
```

พร้อมแล้ว มาเริ่มเรียนรู้ `SELECT` กันเลย

---

## Step 91: SELECT syntax พื้นฐาน — SELECT * vs ระบุคอลัมน์

รูปแบบพื้นฐานที่สุดของคำสั่ง `SELECT` คือ:

```sql
SELECT column1, column2, ...
FROM table_name;
```

### SELECT * — เลือกทุกคอลัมน์

เครื่องหมาย `*` หมายถึง "ทุกคอลัมน์ในตาราง" เรียงตามลำดับที่นิยามไว้ตอนสร้างตาราง

```sql
SELECT * FROM categories;
```

```
 category_id | category_name |         description
-------------+----------------+-------------------------------
           1 | Hot Coffee     | เครื่องดื่มกาแฟร้อน
           2 | Cold Coffee    | เครื่องดื่มกาแฟเย็นและปั่น
           3 | Tea            | เครื่องดื่มชา
           4 | Bakery         | เบเกอรี่และของทานเล่น
           5 | Merchandise    | สินค้าที่ระลึก
(5 rows)
```

### ระบุคอลัมน์เฉพาะที่ต้องการ

ในทางปฏิบัติ เราแทบไม่เคยต้องการทุกคอลัมน์เสมอไป การระบุคอลัมน์ที่ต้องการอย่างชัดเจนเป็นแนวทางที่ถูกต้องกว่า:

```sql
SELECT product_name, price
FROM products;
```

```
   product_name    | price
--------------------+-------
 Espresso           | 55.00
 Americano          | 60.00
 Cappuccino         | 70.00
 Latte              | 70.00
 Iced Latte         | 75.00
 Iced Americano     | 65.00
 Frappe Caramel     | 95.00
 Thai Milk Tea      | 65.00
 Butter Croissant   | 45.00
 Blueberry Muffin   | 50.00
(10 rows)
```

### ทำไมควรหลีกเลี่ยง SELECT * ใน production code

`SELECT *` สะดวกมากตอนสำรวจข้อมูลเร็ว ๆ ใน `psql` แต่ในโค้ด application หรือ view ที่จะใช้งานจริง ควรหลีกเลี่ยงด้วยเหตุผลต่อไปนี้:

1. **Performance** — คุณอาจดึงคอลัมน์ที่ไม่ได้ใช้ เช่น คอลัมน์ `TEXT` ขนาดใหญ่ หรือ `BYTEA` ที่เก็บไฟล์ ทำให้ query ช้าลงและใช้ network bandwidth โดยไม่จำเป็น
2. **Index-only scan เสียโอกาส** — PostgreSQL สามารถตอบ query ได้จาก index อย่างเดียวโดยไม่ต้องแตะตารางหลัก (heap) ถ้า query ต้องการแค่คอลัมน์ที่อยู่ใน index นั้น แต่ `SELECT *` บังคับให้ต้องอ่านทุกคอลัมน์จากตารางเสมอ
3. **Schema เปลี่ยนแล้วพัง** — ถ้ามีคนเพิ่มคอลัมน์ใหม่ในตารางภายหลัง แอปพลิเคชันที่พึ่ง `SELECT *` และ map ผลลัพธ์ตามตำแหน่งคอลัมน์ (ordinal position) อาจพังทันทีโดยไม่มีสัญญาณเตือนล่วงหน้า
4. **อ่านโค้ดยาก** — คนอ่าน query ไม่รู้ว่า query นี้ต้องการข้อมูลอะไรจริง ๆ ต้องไปเปิดดู schema เพิ่ม
5. **ความปลอดภัย** — อาจดึงคอลัมน์ที่มีข้อมูลอ่อนไหว (เช่น password hash, เลขบัตรประชาชน) ออกมาโดยไม่ตั้งใจ ทั้งที่ผู้ใช้หน้าบ้านไม่ควรเห็น
6. **View และ `SELECT * FROM view` ที่สร้างจาก `SELECT *`** — จะไม่รับรู้คอลัมน์ที่เพิ่มใหม่ในตารางต้นทางโดยอัตโนมัติ (ต้อง `CREATE OR REPLACE VIEW` ใหม่)

**แนวทางที่แนะนำ:** ใช้ `SELECT *` เฉพาะตอน explore ข้อมูลใน `psql` แบบ ad-hoc เท่านั้น ส่วนใน production code, view, หรือ query ที่จะถูกเรียกซ้ำ ๆ ให้ระบุคอลัมน์เสมอ

---

## Step 92: Column Alias ด้วย AS

Alias คือการตั้งชื่อใหม่ให้คอลัมน์ (หรือ expression) ในผลลัพธ์ ใช้คำสั่ง `AS`

```sql
SELECT
    product_name AS name,
    price AS unit_price
FROM products
WHERE product_id <= 3;
```

```
    name    | unit_price
------------+-------------
 Espresso   |       55.00
 Americano  |       60.00
 Cappuccino |       70.00
(3 rows)
```

### ไม่ใช้ AS ก็ได้

PostgreSQL ยอมให้ละคำว่า `AS` ได้ (เป็น optional keyword) โดยเขียนชื่อ alias ต่อจากคอลัมน์ได้เลย:

```sql
SELECT product_name name, price unit_price
FROM products
WHERE product_id <= 3;
```

ผลลัพธ์จะเหมือนกับด้านบนทุกประการ อย่างไรก็ตาม **แนะนำให้เขียน `AS` เสมอ** เพราะทำให้อ่านง่ายขึ้นชัดเจนว่านี่คือการตั้งชื่อ ไม่ใช่การพิมพ์ผิดหรือลืม comma คั่นคอลัมน์

### Alias ที่มีช่องว่าง (space) ต้องใช้ double quotes

ชื่อ alias ปกติต้องเป็นไปตามกฎการตั้งชื่อ identifier ของ PostgreSQL (ขึ้นต้นด้วยตัวอักษรหรือ underscore ตามด้วยตัวอักษร ตัวเลข หรือ underscore) ถ้าต้องการใช้ space หรืออักขระพิเศษ หรือต้องการรักษาตัวพิมพ์เล็ก-ใหญ่ (case) ให้ครอบด้วย double quotes `"..."`:

```sql
SELECT
    product_name AS "ชื่อสินค้า",
    price AS "ราคา (บาท)"
FROM products
WHERE product_id <= 3;
```

```
    ชื่อสินค้า    | ราคา (บาท)
------------------+-------------
 Espresso         |       55.00
 Americano        |       60.00
 Cappuccino       |       70.00
(3 rows)
```

> **หมายเหตุสำคัญ:** ถ้าไม่ใช้ double quotes PostgreSQL จะแปลง identifier เป็นตัวพิมพ์เล็กโดยอัตโนมัติเสมอ (unless quoted) เช่น `SELECT price AS TotalPrice` จะได้ผลลัพธ์คอลัมน์ชื่อ `totalprice` ไม่ใช่ `TotalPrice` ถ้าต้องการรักษาตัวพิมพ์ใหญ่-เล็กตามที่เขียน ต้องเขียน `AS "TotalPrice"` เท่านั้น
>
> ข้อควรระวังอีกอย่างคือ ห้ามตั้งชื่อ alias ให้ตรงกับ SQL reserved word (เช่น `order`, `group`, `select`) โดยไม่ครอบ double quotes เพราะจะทำให้เกิด syntax error

---

## Step 93: Expressions และการคำนวณใน SELECT List

SELECT list ไม่จำเป็นต้องเป็นชื่อคอลัมน์ตรง ๆ เท่านั้น แต่สามารถเป็น **expression** ใด ๆ ที่ประเมินผลได้ เช่น การคำนวณทางคณิตศาสตร์ หรือการต่อสตริง

### Arithmetic expressions

```sql
SELECT
    product_name,
    price,
    cost,
    price - cost AS profit,
    ROUND(price * 1.07, 2) AS price_incl_vat
FROM products
ORDER BY product_id;
```

```
   product_name    | price | cost  | profit | price_incl_vat
--------------------+-------+-------+--------+-----------------
 Espresso           | 55.00 | 18.00 |  37.00 |           58.85
 Americano          | 60.00 | 20.00 |  40.00 |           64.20
 Cappuccino         | 70.00 | 25.00 |  45.00 |           74.90
 Latte              | 70.00 | 25.00 |  45.00 |           74.90
 Iced Latte         | 75.00 | 27.00 |  48.00 |           80.25
 Iced Americano     | 65.00 | 22.00 |  43.00 |           69.55
 Frappe Caramel     | 95.00 | 35.00 |  60.00 |          101.65
 Thai Milk Tea      | 65.00 | 20.00 |  45.00 |           69.55
 Butter Croissant   | 45.00 | 15.00 |  30.00 |           48.15
 Blueberry Muffin   | 50.00 | 18.00 |  32.00 |           53.50
(10 rows)
```

เราแอบใช้ `ORDER BY` ในตัวอย่างข้างบนเพื่อให้ผลลัพธ์เรียงตามลำดับที่คาดเดาได้ (รายละเอียดของ `ORDER BY` จะอธิบายอย่างละเอียดในบทถัดไป ตอนนี้ขอให้รู้แค่ว่ามันเรียงลำดับผลลัพธ์)

ตัวดำเนินการทางคณิตศาสตร์ที่ใช้ได้ใน PostgreSQL ได้แก่ `+` `-` `*` `/` (หารแบบ integer division ถ้าทั้งสองฝั่งเป็น integer) `%` (mod) และ `^` (ยกกำลัง)

### String concatenation ด้วย ||

PostgreSQL ใช้ตัวดำเนินการ `||` สำหรับต่อสตริง (ตามมาตรฐาน SQL) เทียบเท่ากับฟังก์ชัน `CONCAT()`

```sql
SELECT
    customer_id,
    first_name || ' ' || last_name AS full_name,
    city
FROM customers
ORDER BY customer_id;
```

```
 customer_id |     full_name      |      city
-------------+---------------------+-----------------
           1 | สมชาย ใจดี          | กรุงเทพฯ
           2 | สมหญิง รักเรียน     | เชียงใหม่
           3 | วิชัย มั่งมี         | กรุงเทพฯ
           4 | มานี มีนา            | ขอนแก่น
           5 | ประยุทธ ยงยุทธ      | กรุงเทพฯ
           6 | สุดา สุขใจ           | ภูเก็ต
           7 | อนันต์ อนุกูล        | เชียงใหม่
           8 | พิมพ์ใจ พิมพ์ดี     | กรุงเทพฯ
           9 | ธนาคาร ธนกิจ         | นครราชสีมา
(9 rows)
```

> **ข้อควรระวังเรื่อง NULL:** ถ้าคอลัมน์ใดคอลัมน์หนึ่งที่นำมา `||` เป็นค่า `NULL` ผลลัพธ์ของการต่อสตริงทั้งหมดจะกลายเป็น `NULL` ทันที (NULL propagation) เช่น ถ้าลูกค้า `อนันต์` ไม่มี `email` แล้วเราเขียน `email || ' (verified)'` ผลลัพธ์จะเป็น `NULL` ไม่ใช่ `'(verified)'` เฉย ๆ — เรื่องนี้จะอธิบายลึกขึ้นเมื่อพูดถึง `COALESCE()` ใน Part เกี่ยวกับ NULL handling

ทางเลือกอีกแบบคือฟังก์ชัน `CONCAT()` ซึ่ง**ไม่**ทำให้ผลลัพธ์กลายเป็น `NULL` แม้มี argument เป็น `NULL` (มันจะข้ามค่า `NULL` ไปเฉย ๆ):

```sql
SELECT
    customer_id,
    CONCAT(first_name, ' ', last_name, ' <', email, '>') AS contact
FROM customers
WHERE customer_id IN (1, 7);
```

```
 customer_id |             contact
-------------+-----------------------------------
           1 | สมชาย ใจดี <somchai.j@example.com>
           7 | อนันต์ อนุกูล <>
(2 rows)
```

สังเกตว่าแถวของ `อนันต์` (ที่ `email` เป็น `NULL`) ยังคงแสดงชื่อและวงเล็บ `<>` ปกติ ต่างจากการใช้ `||` ที่จะได้ `NULL` ทั้งบรรทัด

---

## Step 94: DISTINCT — การกรองแถวซ้ำ

`DISTINCT` ใช้กรองแถวที่มีค่าเหมือนกันทุกคอลัมน์ที่ระบุใน SELECT list ให้เหลือเพียงแถวเดียว

### DISTINCT บนคอลัมน์เดียว

```sql
SELECT DISTINCT city
FROM customers
ORDER BY city;
```

```
      city
-----------------
 กรุงเทพฯ
 ขอนแก่น
 นครราชสีมา
 ภูเก็ต
 เชียงใหม่
(5 rows)
```

ถึงแม้ตาราง `customers` จะมี 9 แถว แต่มี `city` ที่ไม่ซ้ำกันแค่ 5 ค่า (`กรุงเทพฯ` ปรากฏถึง 4 ครั้ง, `เชียงใหม่` ปรากฏ 2 ครั้ง) คำสั่ง `DISTINCT` จึงคืนแค่ 5 แถว

เปรียบเทียบกับไม่ใช้ `DISTINCT`:

```sql
SELECT city FROM customers ORDER BY city;
-- จะได้ 9 แถว โดยมี 'กรุงเทพฯ' ซ้ำ 4 ครั้ง และ 'เชียงใหม่' ซ้ำ 2 ครั้ง
```

### DISTINCT บนหลายคอลัมน์

เมื่อระบุหลายคอลัมน์ `DISTINCT` จะพิจารณา**ค่าผสมกัน (combination)** ของทุกคอลัมน์ที่ระบุ ไม่ใช่แยกกันทีละคอลัมน์:

```sql
SELECT DISTINCT category_id, is_active
FROM products
ORDER BY category_id, is_active;
```

```
 category_id | is_active
-------------+-----------
           1 | t
           2 | t
           3 | t
           4 | f
           4 | t
(5 rows)
```

สังเกตว่า `category_id = 4` (Bakery) ปรากฏ 2 ครั้ง เพราะมีทั้งสินค้าที่ `is_active = true` (Butter Croissant) และ `is_active = false` (Blueberry Muffin) — คู่ค่าทั้งสองนี้ถือว่า "ไม่ซ้ำกัน" เพราะค่าผสมต่างกัน แม้ `category_id` จะเหมือนกันก็ตาม

### DISTINCT ทำงานอย่างไรเบื้องหลัง และข้อควรระวังเรื่อง performance

PostgreSQL มักจะ implement `DISTINCT` ด้วยการ sort ข้อมูลก่อนแล้วไล่ตัดแถวที่ซ้ำติดกัน (Unique หลัง Sort) หรือใช้ HashAggregate ก็ได้ขึ้นกับ query planner ทั้งสองวิธีมีต้นทุน — ยิ่งข้อมูลมาก ยิ่งใช้ CPU และหน่วยความจำมากขึ้น ลองดู execution plan:

```sql
EXPLAIN SELECT DISTINCT city FROM customers;
```

```
                        QUERY PLAN
------------------------------------------------------------
 HashAggregate  (cost=1.11..1.16 rows=5 width=13)
   Group Key: city
   ->  Seq Scan on customers  (cost=0.00..1.09 rows=9 width=13)
(3 rows)
```

ในตารางเล็ก ๆ แบบนี้ไม่มีผลกระทบอะไร แต่ในตารางระดับล้านแถว `DISTINCT` ที่ไม่จำเป็นอาจทำให้ query ช้าลงมาก ควรใช้เมื่อจำเป็นจริง ๆ เท่านั้น และถ้าเป้าหมายคือการนับหรือจัดกลุ่ม ควรพิจารณาใช้ `GROUP BY` แทน ซึ่งจะอธิบายรายละเอียดใน Part ว่าด้วย Aggregate Functions

---

## Step 95: DISTINCT ON — ฟีเจอร์เฉพาะของ PostgreSQL

`DISTINCT ON` เป็น extension ของ PostgreSQL ที่ไม่มีในมาตรฐาน SQL (และไม่มีใน MySQL, SQL Server) แต่เป็นหนึ่งในฟีเจอร์ที่มีประโยชน์มากที่สุดของ PostgreSQL มันช่วยให้เราเลือก "หนึ่งแถวตัวแทน" จากแต่ละกลุ่มได้ในคำสั่งเดียว โดยไม่ต้องใช้ subquery หรือ window function ที่ซับซ้อน

### Syntax

```sql
SELECT DISTINCT ON (expression [, ...]) select_list
FROM table_name
ORDER BY expression [, ...], other_column [ASC | DESC] ...;
```

**กฎสำคัญ:** คอลัมน์ (หรือ expression) ที่ระบุใน `DISTINCT ON (...)` **ต้องปรากฏเป็นส่วนซ้ายสุดของ `ORDER BY`** เสมอ เพราะ PostgreSQL จะ sort ข้อมูลตาม `ORDER BY` ก่อน แล้วเก็บเฉพาะแถว**แรก**ของแต่ละกลุ่มที่มีค่า `DISTINCT ON` เหมือนกัน

### ตัวอย่างที่ 1: หา order ล่าสุดของลูกค้าแต่ละคน

โจทย์: เราต้องการรู้ว่าลูกค้าแต่ละคน (ที่เคยสั่งซื้อ) มี order ล่าสุดเมื่อไหร่ สถานะอะไร ยอดเท่าไหร่

```sql
SELECT DISTINCT ON (customer_id)
    customer_id,
    order_id,
    order_date,
    status,
    total_amount
FROM orders
ORDER BY customer_id, order_date DESC;
```

```
 customer_id | order_id |     order_date      |  status   | total_amount
-------------+----------+----------------------+-----------+---------------
           1 |       11 | 2024-02-10 08:00:00  | pending   |         60.00
           2 |        5 | 2024-01-20 16:45:00  | cancelled |         75.00
           3 |        6 | 2024-01-10 07:50:00  | completed |        240.00
           4 |        7 | 2024-01-15 11:30:00  | completed |         65.00
           5 |        9 | 2024-02-02 09:40:00  | completed |         70.00
           6 |       10 | 2024-01-25 15:00:00  | completed |         95.00
(6 rows)
```

อธิบายทีละขั้น:
1. `ORDER BY customer_id, order_date DESC` จัดเรียงแถวทั้งหมดตาม `customer_id` ก่อน แล้วภายในแต่ละ `customer_id` เรียงตาม `order_date` จากใหม่ไปเก่า
2. `DISTINCT ON (customer_id)` บอกให้เก็บเฉพาะแถว**แรก**ของแต่ละ `customer_id` — เนื่องจากข้อมูลถูกเรียงให้ `order_date` ใหม่สุดอยู่บนสุดของแต่ละกลุ่มแล้ว แถวแรกจึงเป็น order ล่าสุดพอดี
3. ลูกค้าหมายเลข 7, 8, 9 ไม่ปรากฏในผลลัพธ์เพราะยังไม่เคยมี order เลย (เราค้นจากตาราง `orders` เท่านั้น ยังไม่ได้ join กับ `customers`)

### ตัวอย่างที่ 2: หาสินค้าราคาถูกที่สุดในแต่ละหมวดหมู่

```sql
SELECT DISTINCT ON (category_id)
    category_id,
    product_name,
    price
FROM products
ORDER BY category_id, price ASC;
```

```
 category_id |   product_name    | price
-------------+---------------------+-------
           1 | Espresso            | 55.00
           2 | Iced Americano      | 65.00
           3 | Thai Milk Tea       | 65.00
           4 | Butter Croissant    | 45.00
(4 rows)
```

`category_id = 5` (Merchandise) ไม่มีสินค้าเลยจึงไม่ปรากฏ

> **ข้อควรระวัง:** ถ้ามีค่าที่เรียงเป็นอันดับหนึ่งเท่ากันหลายแถว (tie) เช่น ราคาสูงสุดเท่ากันพอดี PostgreSQL จะเลือกแถวใดแถวหนึ่งแบบไม่รับประกันว่าจะเป็นแถวไหน (ไม่ deterministic) เว้นแต่คุณจะเพิ่มคอลัมน์ tie-breaker เข้าไปใน `ORDER BY` ต่อท้าย เช่น `ORDER BY category_id, price ASC, product_id ASC` เพื่อให้ผลลัพธ์แน่นอนทุกครั้งที่รัน

### เปรียบเทียบ DISTINCT ON กับวิธีอื่น

ก่อนมี `DISTINCT ON` การแก้โจทย์แบบ "หาแถวล่าสุด/มากสุด/น้อยสุดของแต่ละกลุ่ม" ต้องเขียนด้วย subquery ที่ซับซ้อนกว่ามาก หรือใช้ window function (`ROW_NUMBER() OVER (PARTITION BY ...)`) ซึ่งจะอธิบายใน Part ที่ว่าด้วย Window Functions ในระดับ advanced — `DISTINCT ON` เป็นทางลัดที่กระชับและมักเร็วกว่าในหลายกรณี แต่มีข้อจำกัดคือใช้ได้เฉพาะ "หนึ่งแถวต่อกลุ่ม" เท่านั้น ถ้าต้องการ top-N ต่อกลุ่ม (เช่น 3 แถวล่าสุดต่อลูกค้า) ต้องใช้ window function แทน

---

## Step 96: Literal Values ใน SELECT

นอกจากชื่อคอลัมน์และ expression ที่อ้างอิงคอลัมน์แล้ว เรายังสามารถใส่ **literal value** (ค่าคงที่) ลงใน SELECT list ได้โดยตรง

### ตัวเลข ข้อความ และ NULL

```sql
SELECT 1 + 1 AS answer;
```

```
 answer
--------
      2
(1 row)
```

```sql
SELECT 'สวัสดี PostgreSQL' AS greeting;
```

```
      greeting
----------------------
 สวัสดี PostgreSQL
(1 row)
```

```sql
SELECT NULL AS empty_value;
```

```
 empty_value
--------------

(1 row)
```

Query ทั้งสามข้างบนไม่มี `FROM` clause เลย เพราะ PostgreSQL อนุญาตให้ `SELECT` โดยไม่ต้องอ้างอิงตารางใด ๆ ก็ได้ ถือเป็นการประเมินผล expression ตรง ๆ (มีประโยชน์มากตอนทดสอบฟังก์ชันหรือ expression แบบเร็ว ๆ โดยไม่ต้องพึ่งตารางจริง)

### การสร้างคอลัมน์คงที่ประกบกับข้อมูลจากตาราง

Literal value มีประโยชน์มากเวลาต้องการเพิ่มคอลัมน์ที่มีค่าคงที่เดียวกันทุกแถว เช่น ทำ tag แหล่งที่มาของข้อมูล (มีประโยชน์มากเมื่อใช้ร่วมกับ `UNION` ใน Part ถัดไป ๆ):

```sql
SELECT
    product_name,
    price,
    'THB' AS currency,
    'coffee_shop_demo' AS source_system
FROM products
WHERE product_id <= 3;
```

```
 product_name  | price | currency |   source_system
----------------+-------+----------+--------------------
 Espresso       | 55.00 | THB      | coffee_shop_demo
 Americano      | 60.00 | THB      | coffee_shop_demo
 Cappuccino     | 70.00 | THB      | coffee_shop_demo
(3 rows)
```

### ฟังก์ชันค่าปัจจุบันของระบบก็นับเป็น literal-like expression

```sql
SELECT current_date AS today, now() AS current_timestamp_value;
```

```
    today    |    current_timestamp_value
-------------+---------------------------------
 2026-09-25  | 2026-09-25 10:15:32.481203+07
(1 row)
```

(ค่าที่ได้จะเปลี่ยนไปตามเวลาจริงที่คุณรัน query — รายละเอียดฟังก์ชันวันเวลาเชิงลึกจะอยู่ใน Part เกี่ยวกับ Date/Time Functions)

---

## Step 97: Table Alias ด้วย AS

เช่นเดียวกับคอลัมน์ ตารางก็สามารถตั้ง alias ได้เช่นกัน

```sql
SELECT p.product_name, p.price
FROM products AS p
WHERE p.category_id = 1;
```

```
  product_name  | price
-----------------+-------
 Espresso        | 55.00
 Americano       | 60.00
 Cappuccino      | 70.00
 Latte           | 70.00
(4 rows)
```

### ไม่ใช้ AS ก็ได้ (เช่นเดียวกับ column alias)

```sql
SELECT p.product_name, p.price
FROM products p
WHERE p.category_id = 1;
```

ผลลัพธ์เหมือนกันทุกประการ ในทางปฏิบัติ นักพัฒนา PostgreSQL ส่วนใหญ่จะละ `AS` ตอนตั้ง alias ให้ตาราง (ต่างจาก column alias ที่มักเขียน `AS` เพื่อความชัดเจน) แต่ทั้งสองแบบถูกต้องตามหลักไวยากรณ์

### ทำไม table alias ถึงสำคัญ

1. **ย่อชื่อตารางยาว ๆ ให้กระชับ** — โดยเฉพาะเมื่อต้อง query หลายตารางพร้อมกัน
2. **แก้ความกำกวมเมื่อหลายตารางมีชื่อคอลัมน์ซ้ำกัน** — เช่นทั้ง `customers` และ `orders` ต่างก็มีคอลัมน์ `customer_id` ถ้าไม่ระบุว่ามาจากตารางไหน PostgreSQL จะไม่รู้ว่าคุณหมายถึงคอลัมน์ของตารางใด (จะเกิด error "column reference is ambiguous")
3. **จำเป็นสำหรับ self-join** — การ join ตารางกับตัวมันเอง (เช่น หาพนักงานที่มีหัวหน้าอยู่ในตารางเดียวกัน) ต้องใช้ alias คนละชื่อเพื่อแยกแยะว่าอันไหนคือ "แถวลูก" อันไหนคือ "แถวแม่"
4. **เตรียมความพร้อมสำหรับ JOIN** — ใน Part 021 เราจะเริ่มเรียนเรื่อง `JOIN` ซึ่งแทบทุก query จะต้องอ้างอิงหลายตารางพร้อมกัน table alias จะกลายเป็นสิ่งที่คุณใช้ทุกวัน

ตัวอย่างการตั้งชื่อ alias สั้น ๆ ตามอักษรตัวแรกของตาราง เป็นธรรมเนียมที่นิยมใช้กันอย่างแพร่หลาย:

```sql
-- แบบที่ยังไม่ join จริง แต่แสดงให้เห็นแนวคิดของการอ้างอิงคอลัมน์ผ่าน alias
SELECT
    c.first_name,
    c.last_name,
    c.city
FROM customers AS c
WHERE c.city = 'กรุงเทพฯ';
```

```
 first_name |  last_name  |   city
-------------+-------------+-----------
 สมชาย       | ใจดี         | กรุงเทพฯ
 วิชัย       | มั่งมี        | กรุงเทพฯ
 ประยุทธ    | ยงยุทธ       | กรุงเทพฯ
 พิมพ์ใจ    | พิมพ์ดี      | กรุงเทพฯ
(4 rows)
```

ใน Part 021 (JOIN) คุณจะเห็น pattern แบบนี้ทันที: `FROM orders o JOIN customers c ON o.customer_id = c.customer_id` — การเข้าใจ table alias ตั้งแต่ตอนนี้จะทำให้บทเรื่อง JOIN เข้าใจง่ายขึ้นมาก

---

## Step 98: Query ข้าม Schema ด้วย Schema-Qualified Names

PostgreSQL จัดกลุ่มตารางไว้ภายใต้ **schema** (namespace ภายในฐานข้อมูลเดียวกัน) โดย default ทุกตารางที่เราสร้างจะอยู่ใน schema ชื่อ `public` เว้นแต่จะระบุเป็นอย่างอื่น

### การอ้างอิงตารางแบบเต็ม (fully schema-qualified)

รูปแบบเต็มของชื่อตารางคือ `schema_name.table_name`:

```sql
SELECT * FROM public.categories;
```

query นี้ให้ผลลัพธ์เหมือนกับ `SELECT * FROM categories;` ทุกประการ เพราะ `public` คือ schema ที่อยู่ใน `search_path` โดย default อยู่แล้ว

ตรวจสอบ `search_path` ปัจจุบันได้ด้วย:

```sql
SHOW search_path;
```

```
   search_path
------------------
 "$user", public
(1 row)
```

`search_path` คือลำดับ schema ที่ PostgreSQL จะค้นหาตารางให้อัตโนมัติเมื่อเราไม่ได้ระบุ schema ไว้ชัดเจน

### สร้างและ query ข้าม schema

ลองสร้าง schema แยกสำหรับเก็บข้อมูล archive (ข้อมูลเก่าที่ไม่ได้ใช้งานบ่อย):

```sql
CREATE SCHEMA IF NOT EXISTS archive;

CREATE TABLE archive.orders_2023 (
    order_id      INT PRIMARY KEY,
    customer_id   INT,
    order_date    TIMESTAMP,
    status        VARCHAR(20),
    total_amount  NUMERIC(10, 2)
);

INSERT INTO archive.orders_2023 (order_id, customer_id, order_date, status, total_amount) VALUES
    (9001, 1, '2023-11-02 09:00:00', 'completed', 150.00),
    (9002, 3, '2023-12-15 10:30:00', 'completed', 200.00);
```

เนื่องจาก `archive` ไม่ได้อยู่ใน `search_path` ค่า default การ query ตารางนี้**ต้องระบุ schema เสมอ**:

```sql
SELECT * FROM archive.orders_2023;
```

```
 order_id | customer_id |     order_date      |   status   | total_amount
----------+-------------+----------------------+-------------+---------------
     9001 |           1 | 2023-11-02 09:00:00  | completed  |        150.00
     9002 |           3 | 2023-12-15 10:30:00  | completed  |        200.00
(2 rows)
```

ถ้าลองรัน `SELECT * FROM orders_2023;` โดยไม่ใส่ schema จะได้ error:

```
ERROR:  relation "orders_2023" does not exist
```

เพราะ PostgreSQL ค้นหาใน schema ตาม `search_path` (`"$user", public`) เท่านั้น ไม่พบตารางชื่อนี้ใน `public`

### ทำไมการจัดกลุ่มด้วย schema จึงมีประโยชน์

- **จัดระเบียบระบบใหญ่** — แยกตารางตามโมดูล เช่น `sales.*`, `inventory.*`, `hr.*` ภายในฐานข้อมูลเดียว
- **Multi-tenant pattern** — บาง application ใช้หนึ่ง schema ต่อหนึ่งลูกค้า (tenant) เพื่อแยกข้อมูลออกจากกันชัดเจนโดยยังอยู่ในฐานข้อมูลเดียว
- **สิทธิ์การเข้าถึง (permissions)** — ให้สิทธิ์ระดับ schema ได้ง่ายกว่าไล่ตั้งสิทธิ์ทีละตาราง
- **แยกข้อมูล archive/staging ออกจากข้อมูล production หลัก** โดยไม่ต้องสร้างฐานข้อมูลใหม่แยกกัน

> **ข้อควรทราบ:** PostgreSQL ไม่รองรับการ query ข้ามฐานข้อมูล (cross-database query) ในคำสั่ง SQL ปกติ ชื่อเต็มของตารางมีแค่ 2 ระดับคือ `schema.table` เท่านั้น (ไม่ใช่ `database.schema.table` แบบ SQL Server) หากต้องการเชื่อมข้อมูลข้ามฐานข้อมูลจริง ๆ ต้องใช้ส่วนขยายอย่าง `postgres_fdw` หรือ `dblink` ซึ่งจะกล่าวถึงใน Part ระดับ advanced

---

## Step 99: Comments ใน SQL และการจัดรูปแบบให้อ่านง่าย

### Comment แบบบรรทัดเดียว (--)

```sql
-- ดึงรายชื่อสินค้าที่ยังขายอยู่ (is_active = true)
SELECT product_name, price
FROM products; -- ยังไม่ได้กรองด้วย WHERE เพราะยังไม่ได้สอน (Part 011)
```

ทุกอย่างหลัง `--` จนจบบรรทัดจะถูกมองข้ามโดย parser ไม่มีผลต่อการทำงานของ query

### Comment แบบหลายบรรทัด (/* ... */)

```sql
/*
  Query: รายชื่อสินค้าพร้อมราคารวม VAT
  Author: ทีม Data Analytics
  Last updated: 2026-09-25
*/
SELECT
    product_name,
    price,
    ROUND(price * 1.07, 2) AS price_incl_vat
FROM products;
```

comment แบบ `/* */` สามารถซ้อนกันได้ใน PostgreSQL (nested comments) ซึ่งต่างจากภาษาโปรแกรมหลายภาษา:

```sql
/* comment ชั้นนอก
   /* comment ชั้นใน */
   ยังอยู่ใน comment ชั้นนอกต่อ
*/
SELECT 1;
```

### Style guide สำหรับเขียน SQL ให้อ่านง่าย

เมื่อ query เริ่มซับซ้อนขึ้น การจัดรูปแบบที่ดีจะช่วยให้ทั้งคุณเองและเพื่อนร่วมทีมอ่านและแก้ไขได้ง่ายในระยะยาว แนวทางที่นิยมใช้กันอย่างแพร่หลาย (และจะใช้ตลอดหลักสูตรนี้):

1. **เขียน keyword หลักด้วยตัวพิมพ์ใหญ่เสมอ** — `SELECT`, `FROM`, `WHERE`, `ORDER BY`, `AS` เพื่อแยกจาก identifier (ชื่อตาราง/คอลัมน์) ที่มักเขียนตัวพิมพ์เล็ก
2. **เขียนแต่ละ clause หลักขึ้นบรรทัดใหม่** — `SELECT`, `FROM`, `WHERE`, `GROUP BY`, `ORDER BY` ควรแยกบรรทัดชัดเจน
3. **หนึ่งคอลัมน์ต่อหนึ่งบรรทัดเมื่อ SELECT list ยาว** — ทำให้ diff ใน git อ่านง่าย และเพิ่ม/ลบคอลัมน์ทีหลังได้สะดวก
4. **ย่อหน้า (indent) ให้สม่ำเสมอ** — นิยมใช้ 4 spaces (หรือ 2 spaces ตามธรรมเนียมทีม แต่ให้สอดคล้องกันทั้งโปรเจกต์)
5. **ตั้งชื่อ table alias ให้สื่อความหมาย** — หลีกเลี่ยงชื่อสั้นเกินไปจนงงเมื่อมีหลายตาราง (เช่น `o` แทน `orders`, `c` แทน `customers` พอเข้าใจได้ แต่ถ้ามีตารางเยอะมากควรใช้ 2-3 ตัวอักษรที่สื่อความหมายชัดกว่า)
6. **ใช้ comment อธิบาย "ทำไม" มากกว่า "อะไร"** — โค้ดที่ดีอธิบายตัวเองว่ากำลังทำอะไรอยู่แล้ว comment ควรเสริมเหตุผลเชิงธุรกิจที่โค้ดเพียงอย่างเดียวบอกไม่ได้
7. **วาง comma ไว้หน้าคอลัมน์ถัดไป (leading comma) หรือท้ายคอลัมน์ (trailing comma) ก็ได้ แต่เลือกแบบเดียวแล้วใช้ให้สม่ำเสมอทั้งทีม**

ตัวอย่าง query ที่จัดรูปแบบตาม style guide ข้างต้น:

```sql
-- สรุปสินค้าพร้อมราคารวม VAT และกำไรต่อหน่วย
-- ใช้สำหรับรายงานราคาประจำเดือนของทีมจัดซื้อ
SELECT
    p.product_name              AS name,
    p.price                     AS selling_price,
    p.cost                      AS cost_price,
    p.price - p.cost            AS profit_per_unit,
    ROUND(p.price * 1.07, 2)    AS price_incl_vat
FROM
    products AS p
ORDER BY
    p.category_id,
    p.product_name;
```

การจัดคอลัมน์ `AS` ให้ตรงกันเป็นแนวตั้ง (alignment) แบบนี้ทำได้สวยงามในบางเครื่องมือ แต่ไม่จำเป็นเสมอไป — เลือกใช้ตามความสะดวกของทีมและเครื่องมือ formatter ที่ใช้งานอยู่ (เช่น `pgFormatter`, `sqlfluff`) สิ่งสำคัญที่สุดคือ**ความสม่ำเสมอ**ทั่วทั้งโปรเจกต์มากกว่ารูปแบบใดรูปแบบหนึ่งที่ "ถูกต้องที่สุด"

---

## Step 100: แบบฝึกหัดรวม — สำรวจข้อมูลร้านกาแฟหลายแบบ

มาลองรวมทุกสิ่งที่เรียนมาใน Part นี้เข้าด้วยกัน ด้วยชุด query สำรวจข้อมูลร้านกาแฟที่ผสมทั้งการเลือกคอลัมน์ alias expression DISTINCT และ DISTINCT ON

### สำรวจที่ 1: รายชื่อสินค้าทั้งหมดพร้อมข้อมูลราคาแบบอ่านง่าย

```sql
SELECT
    product_id                    AS id,
    product_name                  AS "ชื่อสินค้า",
    price                         AS "ราคาขาย",
    ROUND(price * 1.07, 2)        AS "ราคารวม VAT",
    CASE WHEN is_active THEN 'ขายอยู่' ELSE 'เลิกขายแล้ว' END AS "สถานะ"
FROM products
ORDER BY id;
```

(หมายเหตุ: `CASE WHEN` จะอธิบายละเอียดใน Part ถัดไปเช่นกัน ตอนนี้แค่ให้เห็นภาพว่า SELECT list ยืดหยุ่นแค่ไหน)

```
 id |    ชื่อสินค้า     | ราคาขาย | ราคารวม VAT |    สถานะ
----+--------------------+----------+--------------+---------------
  1 | Espresso           |    55.00 |        58.85 | ขายอยู่
  2 | Americano          |    60.00 |        64.20 | ขายอยู่
  3 | Cappuccino         |    70.00 |        74.90 | ขายอยู่
  4 | Latte              |    70.00 |        74.90 | ขายอยู่
  5 | Iced Latte         |    75.00 |        80.25 | ขายอยู่
  6 | Iced Americano     |    65.00 |        69.55 | ขายอยู่
  7 | Frappe Caramel     |    95.00 |       101.65 | ขายอยู่
  8 | Thai Milk Tea      |    65.00 |        69.55 | ขายอยู่
  9 | Butter Croissant   |    45.00 |        48.15 | ขายอยู่
 10 | Blueberry Muffin   |    50.00 |        53.50 | เลิกขายแล้ว
(10 rows)
```

### สำรวจที่ 2: เมืองที่มีลูกค้าอยู่ (ไม่ซ้ำ)

```sql
SELECT DISTINCT city AS "จังหวัด/เมือง"
FROM customers
ORDER BY 1;
```

```
 จังหวัด/เมือง
-----------------
 กรุงเทพฯ
 ขอนแก่น
 นครราชสีมา
 ภูเก็ต
 เชียงใหม่
(5 rows)
```

(สังเกต `ORDER BY 1` — เราสามารถอ้างอิงคอลัมน์ใน `ORDER BY` ด้วยลำดับตำแหน่งได้ด้วย ไม่จำเป็นต้องพิมพ์ชื่อคอลัมน์ซ้ำ)

### สำรวจที่ 3: order ล่าสุดของลูกค้าแต่ละคน พร้อมชื่อเต็มลูกค้า (เตรียมใจสำหรับ JOIN)

query นี้ยังไม่ใช้ JOIN จริง (จะเรียนใน Part 021) แต่แสดง `DISTINCT ON` ผสมกับ table alias:

```sql
SELECT DISTINCT ON (o.customer_id)
    o.customer_id,
    o.order_date,
    o.status,
    o.total_amount
FROM orders AS o
ORDER BY o.customer_id, o.order_date DESC;
```

```
 customer_id |     order_date      |  status   | total_amount
-------------+----------------------+-----------+---------------
           1 | 2024-02-10 08:00:00  | pending   |         60.00
           2 | 2024-01-20 16:45:00  | cancelled |         75.00
           3 | 2024-01-10 07:50:00  | completed |        240.00
           4 | 2024-01-15 11:30:00  | completed |         65.00
           5 | 2024-02-02 09:40:00  | completed |         70.00
           6 | 2024-01-25 15:00:00  | completed |         95.00
(6 rows)
```

---

## สรุปท้ายบท

ใน Part นี้เราครอบคลุมพื้นฐานของ `SELECT` ที่จะใช้ในทุก ๆ query ต่อจากนี้:

- **SELECT syntax** — `SELECT column1, column2 FROM table;` และเหตุผลว่าทำไมควรหลีกเลี่ยง `SELECT *` ใน production (performance, index-only scan, schema เปลี่ยนแล้วพัง, ความปลอดภัย)
- **Column alias** — ใช้ `AS` ตั้งชื่อใหม่ให้คอลัมน์หรือ expression, `AS` เป็น optional, ชื่อที่มี space หรือต้องการรักษา case ต้องใช้ double quotes `"..."`
- **Expressions** — เขียนการคำนวณทางคณิตศาสตร์และต่อสตริงด้วย `||` ลงใน SELECT list ได้โดยตรง พร้อมระวังเรื่อง `NULL` propagation ใน `||`
- **DISTINCT** — กรองแถวซ้ำโดยพิจารณาค่าผสมของทุกคอลัมน์ที่ระบุ ใช้ระวังเรื่อง performance ในตารางใหญ่
- **DISTINCT ON** — ฟีเจอร์เฉพาะของ PostgreSQL สำหรับเลือกหนึ่งแถวตัวแทนต่อกลุ่ม ต้องใช้คู่กับ `ORDER BY` ที่ขึ้นต้นด้วยคอลัมน์เดียวกับที่ระบุใน `DISTINCT ON`
- **Literal values** — ใส่ตัวเลข ข้อความ หรือ `NULL` ลงใน SELECT list ได้โดยตรง มีประโยชน์เมื่อต้องการเพิ่มคอลัมน์ค่าคงที่ หรือทดสอบ expression
- **Table alias** — ย่อชื่อตาราง แก้ความกำกวมเมื่อมีคอลัมน์ชื่อซ้ำ และเป็นพื้นฐานสำคัญก่อนเข้าสู่ `JOIN`
- **Schema-qualified names** — เข้าใจ `schema.table`, `search_path`, และประโยชน์ของการจัดกลุ่มตารางด้วย schema
- **Comments และ style guide** — `--` และ `/* */` พร้อมแนวทางจัดรูปแบบ SQL ให้อ่านง่ายและดูแลรักษาได้ในระยะยาว

Part ถัดไปเราจะไปต่อกับการ**กรองข้อมูล** ด้วย `WHERE` clause และตัวดำเนินการเปรียบเทียบต่าง ๆ ซึ่งจะทำให้ query ของเรามีประโยชน์มากขึ้นอย่างก้าวกระโดด

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1

เขียน query เลือกเฉพาะคอลัมน์ `product_name` และ `price` จากตาราง `products`

<details>
<summary>เฉลย</summary>

```sql
SELECT product_name, price
FROM products;
```

</details>

### แบบฝึกหัดที่ 2

เขียน query แบบเดียวกับข้อ 1 แต่ตั้ง alias ให้คอลัมน์ `price` เป็น `"ราคาขาย"` (มี space จึงต้องใช้ double quotes)

<details>
<summary>เฉลย</summary>

```sql
SELECT
    product_name,
    price AS "ราคาขาย"
FROM products;
```

</details>

### แบบฝึกหัดที่ 3

เขียน query แสดง `product_name` และราคารวม VAT 7% (ปัดเศษ 2 ตำแหน่ง) โดยตั้ง alias คอลัมน์ราคารวม VAT ว่า `price_incl_vat`

<details>
<summary>เฉลย</summary>

```sql
SELECT
    product_name,
    ROUND(price * 1.07, 2) AS price_incl_vat
FROM products;
```

</details>

### แบบฝึกหัดที่ 4

เขียน query รวมชื่อและนามสกุลของลูกค้าทุกคนในตาราง `customers` เป็นคอลัมน์เดียวชื่อ `full_name` (คั่นด้วยช่องว่าง)

<details>
<summary>เฉลย</summary>

```sql
SELECT
    customer_id,
    first_name || ' ' || last_name AS full_name
FROM customers
ORDER BY customer_id;
```

</details>

### แบบฝึกหัดที่ 5

เขียน query หา `city` ที่ไม่ซ้ำกันของลูกค้าทั้งหมด เรียงตามตัวอักษร

<details>
<summary>เฉลย</summary>

```sql
SELECT DISTINCT city
FROM customers
ORDER BY city;
```

</details>

### แบบฝึกหัดที่ 6

เขียน query หาคู่ค่า `(category_id, is_active)` ที่ไม่ซ้ำกันในตาราง `products` เรียงตาม `category_id`

<details>
<summary>เฉลย</summary>

```sql
SELECT DISTINCT category_id, is_active
FROM products
ORDER BY category_id, is_active;
```

ผลลัพธ์ที่ได้ควรมี 5 แถว โดย `category_id = 4` ปรากฏสองครั้ง (ทั้ง `true` และ `false`) เพราะในหมวดนั้นมีทั้งสินค้าที่ยังขายอยู่และเลิกขายแล้ว

</details>

### แบบฝึกหัดที่ 7

ใช้ `DISTINCT ON` เขียน query หา order **ล่าสุด** ของลูกค้าแต่ละคน (แสดง `customer_id`, `order_date`, `status`, `total_amount`)

<details>
<summary>เฉลย</summary>

```sql
SELECT DISTINCT ON (customer_id)
    customer_id,
    order_date,
    status,
    total_amount
FROM orders
ORDER BY customer_id, order_date DESC;
```

</details>

### แบบฝึกหัดที่ 8

ใช้ `DISTINCT ON` เขียน query หาสินค้าที่ **แพงที่สุด** ในแต่ละหมวดหมู่ (แสดง `category_id`, `product_name`, `price`)

<details>
<summary>เฉลย</summary>

```sql
SELECT DISTINCT ON (category_id)
    category_id,
    product_name,
    price
FROM products
ORDER BY category_id, price DESC, product_id ASC;
```

หมายเหตุ: เพิ่ม `product_id ASC` เป็น tie-breaker ในกรณีที่มีสินค้าราคาเท่ากันในหมวดเดียวกัน (เช่น Cappuccino และ Latte ในหมวด Hot Coffee ที่ราคา 70.00 บาทเท่ากัน) เพื่อให้ผลลัพธ์คงที่ (deterministic) ทุกครั้งที่รัน

</details>

### แบบฝึกหัดที่ 9

เขียน query แสดง `product_name`, `price` จากตาราง `products` พร้อมเพิ่มคอลัมน์ literal คงที่ชื่อ `store_name` ที่มีค่าเป็น `'Coffee Shop Demo'` ทุกแถว

<details>
<summary>เฉลย</summary>

```sql
SELECT
    product_name,
    price,
    'Coffee Shop Demo' AS store_name
FROM products;
```

</details>

### แบบฝึกหัดที่ 10

เขียน query ที่ใช้ table alias (`p` สำหรับ `products`) และ schema-qualified name (`public.products`) พร้อมใส่ comment อธิบาย query สั้น ๆ ไว้ด้านบน

<details>
<summary>เฉลย</summary>

```sql
-- แสดงชื่อและราคาสินค้าทั้งหมด อ้างอิงตารางแบบ schema-qualified
-- และใช้ table alias เตรียมความพร้อมก่อนเรียนเรื่อง JOIN
SELECT
    p.product_name,
    p.price
FROM public.products AS p
ORDER BY p.product_id;
```

</details>

---

**บทถัดไป:** [Part 011 — WHERE และตัวดำเนินการเปรียบเทียบ](./part-011-where-operators.md)
