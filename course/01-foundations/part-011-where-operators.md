# Part 011: WHERE Clause และ Operators

**หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับพื้นฐาน | Part 011**

---

## เป้าหมายการเรียนรู้

หลังจากเรียนจบบทนี้ ผู้เรียนจะสามารถ:

- ใช้ `WHERE` clause กรองแถวข้อมูลด้วย comparison operators (`=`, `<>`/`!=`, `<`, `>`, `<=`, `>=`)
- รวมเงื่อนไขหลายตัวด้วย `AND`, `OR`, `NOT` และเข้าใจลำดับความสำคัญ (operator precedence) เพื่อไม่ให้ผลลัพธ์ผิดพลาด
- ใช้ `BETWEEN...AND` กรองช่วงตัวเลขและวันที่ พร้อมรู้ทันข้อผิดพลาดที่พบบ่อยเรื่อง timestamp
- ใช้ `IN` / `NOT IN` แทนการเขียน `OR` ซ้ำๆ และเข้าใจกับดักของ `NOT IN` เมื่อเจอ `NULL`
- ใช้ `LIKE` และ `ILIKE` ทำ pattern matching ด้วย `%` และ `_`
- ใช้ regular expression matching เบื้องต้น (`~`, `~*`, `!~`, `!~*`) และรู้ว่าควรเลือกใช้ระหว่าง `LIKE` กับ regex เมื่อไร
- ใช้ `IS NULL` / `IS NOT NULL` อย่างถูกต้อง และเข้าใจว่าทำไม `= NULL` ใช้ไม่ได้
- เขียน `WHERE` clause ที่รวม operator หลายตัวสำหรับสถานการณ์จริง
- แยกแยะเบื้องต้นว่า `WHERE` clause แบบไหน "sargable" (ใช้ index ได้) และแบบไหนไม่ใช่

---

## เตรียมข้อมูล

บทนี้ใช้ schema "ร้านกาแฟ" (coffee shop) เดิมจาก Part ก่อนหน้า ถ้าคุณยังไม่มีข้อมูลนี้ในฐานข้อมูล หรือต้องการรีเซ็ตข้อมูลให้ตรงกับตัวอย่างในบทนี้ ให้รันสคริปต์ด้านล่างทั้งหมด

```sql
-- ล้างตารางเดิม (ถ้ามี) เพื่อให้ผลลัพธ์ตรงกับตัวอย่างในบทนี้
DROP TABLE IF EXISTS order_items, orders, customers, products, categories CASCADE;

-- ตารางหมวดหมู่สินค้า
CREATE TABLE categories (
    category_id   SERIAL PRIMARY KEY,
    category_name VARCHAR(50) NOT NULL,
    description   TEXT
);

-- ตารางสินค้า
CREATE TABLE products (
    product_id     SERIAL PRIMARY KEY,
    product_name   VARCHAR(100) NOT NULL,
    category_id    INT REFERENCES categories(category_id),
    price          NUMERIC(10,2) NOT NULL,
    cost           NUMERIC(10,2),
    stock_quantity INT DEFAULT 0,
    is_active      BOOLEAN DEFAULT TRUE,
    created_at     DATE DEFAULT CURRENT_DATE
);

-- ตารางลูกค้า
CREATE TABLE customers (
    customer_id   SERIAL PRIMARY KEY,
    first_name    VARCHAR(50),
    last_name     VARCHAR(50),
    email         VARCHAR(100) UNIQUE,
    phone         VARCHAR(20),
    member_tier   VARCHAR(20),
    birth_date    DATE,
    created_at    DATE DEFAULT CURRENT_DATE
);

-- ตารางคำสั่งซื้อ
CREATE TABLE orders (
    order_id       SERIAL PRIMARY KEY,
    customer_id    INT REFERENCES customers(customer_id),
    order_date     TIMESTAMP NOT NULL,
    order_status   VARCHAR(20) NOT NULL,
    total_amount   NUMERIC(10,2) NOT NULL,
    payment_method VARCHAR(20)
);

-- ตารางรายการสินค้าในคำสั่งซื้อ
CREATE TABLE order_items (
    order_item_id SERIAL PRIMARY KEY,
    order_id      INT REFERENCES orders(order_id),
    product_id    INT REFERENCES products(product_id),
    quantity      INT NOT NULL,
    unit_price    NUMERIC(10,2) NOT NULL,
    subtotal      NUMERIC(10,2) NOT NULL
);
```

```sql
-- ข้อมูลหมวดหมู่
INSERT INTO categories (category_id, category_name, description) VALUES
(1, 'Hot Coffee',  'เครื่องดื่มกาแฟร้อน'),
(2, 'Cold Coffee', 'เครื่องดื่มกาแฟเย็น'),
(3, 'Tea',         'เครื่องดื่มชา'),
(4, 'Bakery',      'เบเกอรี่'),
(5, 'Snack',       'ขนมทานเล่น');

-- ข้อมูลสินค้า (สังเกต product_id = 15 ไม่มีหมวดหมู่ และ product_id = 14 หยุดขายแล้ว)
INSERT INTO products (product_id, product_name, category_id, price, cost, stock_quantity, is_active, created_at) VALUES
(1,  'Espresso',         1, 55.00, 20.00, 100, TRUE,  '2024-01-10'),
(2,  'Americano',        1, 60.00, 22.00, 120, TRUE,  '2024-01-10'),
(3,  'Cappuccino',       1, 70.00, 28.00,  80, TRUE,  '2024-01-12'),
(4,  'Latte',            1, 70.00, 28.00,  90, TRUE,  '2024-01-12'),
(5,  'Iced Americano',   2, 65.00, 22.00, 150, TRUE,  '2024-02-01'),
(6,  'Iced Latte',       2, 75.00, 30.00, 100, TRUE,  '2024-02-01'),
(7,  'Cold Brew',        2, 85.00, 35.00,  60, TRUE,  '2024-02-15'),
(8,  'Thai Tea',         3, 55.00, 18.00,  70, TRUE,  '2024-01-20'),
(9,  'Green Tea Latte',  3, 75.00, 30.00,  50, TRUE,  '2024-01-20'),
(10, 'Earl Grey',        3, 60.00, 20.00,  40, TRUE,  '2024-03-01'),
(11, 'Croissant',        4, 45.00, 18.00,  30, TRUE,  '2024-01-05'),
(12, 'Chocolate Muffin', 4, 40.00, 15.00,  25, TRUE,  '2024-01-05'),
(13, 'Cheese Cake',      4, 95.00, 40.00,  15, TRUE,  '2024-04-01'),
(14, 'Potato Chips',     5, 35.00, 15.00,   0, FALSE, '2024-01-05'),
(15, 'Cookies',       NULL, 30.00, 10.00,  40, TRUE,  '2024-05-01');

SELECT setval('products_product_id_seq', 15);

-- ข้อมูลลูกค้า (สังเกต customer_id = 3 ไม่มีเบอร์โทร, customer_id = 6 ไม่มีวันเกิด)
INSERT INTO customers (customer_id, first_name, last_name, email, phone, member_tier, birth_date, created_at) VALUES
(1,  'สมชาย',    'ใจดี',     'somchai@example.com',       '0812345678', 'gold',     '1990-05-15', '2023-01-10'),
(2,  'สมหญิง',   'รักเรียน', 'somying@example.com',       '0823456789', 'silver',   '1995-08-20', '2023-02-15'),
(3,  'วิชัย',     'มั่งมี',    'wichai@example.com',        NULL,         'gold',     '1988-12-01', '2023-01-20'),
(4,  'มานี',     'มีนา',      'manee@example.com',         '0834567890', 'bronze',   '2000-03-10', '2023-05-01'),
(5,  'ประเสริฐ', 'ดีเลิศ',    'prasert@example.com',       '0845678901', 'silver',   '1992-07-07', '2023-06-15'),
(6,  'สุดา',      'ใจงาม',    'suda@example.com',          '0856789012', 'bronze',    NULL,        '2023-07-01'),
(7,  'อนันต์',    'สุขสันต์',  'anan@gmail.com',            '0867890123', 'gold',     '1985-11-11', '2022-12-01'),
(8,  'กมลชนก',   'แสงทอง',   'kamolchanok@gmail.com',     '0878901234', 'platinum', '1998-02-28', '2023-03-01'),
(9,  'ธนกร',      'รุ่งเรือง', 'thanakorn@example.com',     '0889012345', 'bronze',   '2001-09-09', '2024-01-01'),
(10, 'พิมพ์ใจ',    'น่ารัก',    'pimjai@example.com',        '0890123456', 'silver',   '1993-04-04', '2023-09-01'),
(11, 'วราภรณ์',   'ศรีสุข',    'waraporn@yahoo.com',        '0801112222', 'bronze',   '1999-06-18', '2024-02-01');

SELECT setval('customers_customer_id_seq', 11);

-- ข้อมูลคำสั่งซื้อ (order_id = 13 เป็น guest order ไม่มี customer_id)
INSERT INTO orders (order_id, customer_id, order_date, order_status, total_amount, payment_method) VALUES
(1,  1,    '2024-06-01 08:15:00', 'completed', 155.00, 'cash'),
(2,  2,    '2024-06-01 09:30:00', 'completed',  70.00, 'credit_card'),
(3,  1,    '2024-06-02 08:00:00', 'completed', 195.00, 'promptpay'),
(4,  3,    '2024-06-02 14:20:00', 'cancelled',  60.00, 'cash'),
(5,  4,    '2024-06-03 10:10:00', 'completed',  45.00, 'credit_card'),
(6,  5,    '2024-06-03 16:45:00', 'completed', 210.00, 'promptpay'),
(7,  6,    '2024-06-04 07:50:00', 'completed',  55.00, 'cash'),
(8,  2,    '2024-06-05 11:00:00', 'completed', 150.00, 'credit_card'),
(9,  7,    '2024-06-05 13:30:00', 'pending',    95.00, 'promptpay'),
(10, 8,    '2024-06-06 09:00:00', 'completed', 310.00, 'credit_card'),
(11, 1,    '2024-06-07 08:20:00', 'completed',  65.00, 'cash'),
(12, 9,    '2024-06-08 15:15:00', 'completed',  40.00, 'cash'),
(13, NULL, '2024-06-09 12:00:00', 'completed',  75.00, 'cash'),
(14, 10,   '2024-06-10 10:30:00', 'cancelled', 115.00, 'promptpay');

SELECT setval('orders_order_id_seq', 14);

-- ข้อมูลรายการสินค้าในคำสั่งซื้อ
INSERT INTO order_items (order_item_id, order_id, product_id, quantity, unit_price, subtotal) VALUES
(1,  1,  1,  2, 55.00, 110.00),
(2,  1,  11, 1, 45.00,  45.00),
(3,  2,  3,  1, 70.00,  70.00),
(4,  3,  4,  2, 70.00, 140.00),
(5,  3,  8,  1, 55.00,  55.00),
(6,  4,  2,  1, 60.00,  60.00),
(7,  5,  11, 1, 45.00,  45.00),
(8,  6,  7,  2, 85.00, 170.00),
(9,  6,  12, 1, 40.00,  40.00),
(10, 7,  8,  1, 55.00,  55.00),
(11, 8,  6,  2, 75.00, 150.00),
(12, 9,  13, 1, 95.00,  95.00),
(13, 10, 7,  2, 85.00, 170.00),
(14, 10, 13, 1, 95.00,  95.00),
(15, 10, 11, 1, 45.00,  45.00),
(16, 11, 5,  1, 65.00,  65.00),
(17, 12, 12, 1, 40.00,  40.00),
(18, 13, 6,  1, 75.00,  75.00),
(19, 14, 6,  1, 75.00,  75.00),
(20, 14, 12, 1, 40.00,  40.00);

SELECT setval('order_items_order_item_id_seq', 20);
```

> **หมายเหตุ**: ในตัวอย่างของบทนี้ ผลลัพธ์จะแสดงตามลำดับ `product_id` / `customer_id` / `order_id` จากน้อยไปมาก เพื่อให้อ่านง่าย แต่ในทางปฏิบัติ PostgreSQL **ไม่การันตีลำดับผลลัพธ์** ถ้าไม่ได้ระบุ `ORDER BY` (จะเรียนละเอียดใน Part 012)

---

## Step 101: WHERE clause พื้นฐาน และ Comparison Operators

`WHERE` คือ clause ที่ใช้ "กรอง" แถวข้อมูลจากตารางให้เหลือเฉพาะแถวที่ตรงกับเงื่อนไขที่กำหนด เป็นหนึ่งใน clause ที่ใช้บ่อยที่สุดใน SQL

รูปแบบทั่วไป:

```sql
SELECT column1, column2, ...
FROM table_name
WHERE condition;
```

PostgreSQL รองรับ comparison operators มาตรฐานดังนี้:

| Operator | ความหมาย | ตัวอย่าง |
|----------|----------|----------|
| `=`      | เท่ากับ | `price = 70` |
| `<>` หรือ `!=` | ไม่เท่ากับ (ทั้งสองแบบใช้แทนกันได้ แต่ `<>` เป็นมาตรฐาน SQL) | `price <> 70` |
| `<`      | น้อยกว่า | `price < 70` |
| `>`      | มากกว่า | `price > 70` |
| `<=`     | น้อยกว่าหรือเท่ากับ | `price <= 70` |
| `>=`     | มากกว่าหรือเท่ากับ | `price >= 70` |

### ตัวอย่างที่ 1: กรองด้วย `=`

```sql
SELECT product_name, price
FROM products
WHERE price = 70.00;
```

```
 product_name | price
--------------+-------
 Cappuccino   | 70.00
 Latte        | 70.00
(2 rows)
```

### ตัวอย่างที่ 2: กรองด้วย `<>` (ไม่เท่ากับ)

```sql
SELECT product_name, price
FROM products
WHERE price <> 70.00;
```

ผลลัพธ์จะเป็นสินค้าทั้งหมด 13 แถวที่ราคาไม่ใช่ 70.00 (ทุกแถวยกเว้น Cappuccino และ Latte) ใช้ `!=` แทน `<>` ได้ผลลัพธ์เหมือนกันทุกประการ:

```sql
SELECT product_name, price
FROM products
WHERE price != 70.00;
```

### ตัวอย่างที่ 3: กรองด้วย `<` และการรวมกับคอลัมน์อื่น

```sql
SELECT product_name, stock_quantity
FROM products
WHERE stock_quantity < 30;
```

```
   product_name   | stock_quantity
-------------------+----------------
 Chocolate Muffin  |             25
 Cheese Cake       |             15
 Potato Chips      |              0
(3 rows)
```

### ตัวอย่างที่ 4: กรองด้วย `>=`

```sql
SELECT product_name, price
FROM products
WHERE price >= 80;
```

```
 product_name | price
--------------+-------
 Cold Brew    | 85.00
 Cheese Cake  | 95.00
(2 rows)
```

### ตัวอย่างที่ 5: ใช้กับข้อความ (text)

Comparison operators ใช้ได้กับ text เช่นกัน โดยเทียบตามลำดับ collation (โดยทั่วไปคือเรียงตามตัวอักษร):

```sql
SELECT first_name, last_name, member_tier
FROM customers
WHERE member_tier = 'gold';
```

```
 first_name | last_name | member_tier
------------+-----------+-------------
 สมชาย      | ใจดี      | gold
 วิชัย       | มั่งมี     | gold
 อนันต์      | สุขสันต์   | gold
(3 rows)
```

> **ข้อควรระวัง**: `=` กับข้อความใน PostgreSQL เป็น **case-sensitive** เสมอ (ตัวพิมพ์ใหญ่-เล็กมีผล) เช่น `'Gold' = 'gold'` จะได้ `false` ถ้าต้องการเทียบแบบไม่สนตัวพิมพ์ใหญ่เล็ก จะเรียนใน Step 105 (`ILIKE`) หรือใช้ฟังก์ชัน `LOWER()`/`UPPER()`

---

## Step 102: Logical Operators (AND, OR, NOT) และลำดับความสำคัญ

เมื่อต้องการกรองด้วยหลายเงื่อนไขพร้อมกัน ใช้ logical operators สามตัวนี้ร่วมกับ `WHERE`:

- `AND` — เงื่อนไขทุกตัวต้องเป็นจริง
- `OR` — เงื่อนไขอย่างน้อยหนึ่งตัวต้องเป็นจริง
- `NOT` — กลับค่าความจริง (negation)

### ตัวอย่างที่ 1: AND

```sql
SELECT product_name, category_id, price
FROM products
WHERE category_id = 1 AND price > 60;
```

```
 product_name | category_id | price
--------------+-------------+-------
 Cappuccino   |           1 | 70.00
 Latte        |           1 | 70.00
(2 rows)
```

### ตัวอย่างที่ 2: OR

```sql
SELECT product_name, category_id
FROM products
WHERE category_id = 1 OR category_id = 3;
```

```
   product_name   | category_id
-------------------+-------------
 Espresso          |           1
 Americano         |           1
 Cappuccino        |           1
 Latte             |           1
 Thai Tea          |           3
 Green Tea Latte   |           3
 Earl Grey         |           3
(7 rows)
```

### ตัวอย่างที่ 3: NOT

```sql
SELECT product_name, is_active
FROM products
WHERE NOT is_active;
```

```
 product_name  | is_active
---------------+-----------
 Potato Chips  | f
(1 row)
```

ผลลัพธ์เดียวกันเขียนได้อีกแบบว่า `WHERE is_active = FALSE`

### ลำดับความสำคัญ (Operator Precedence) — จุดที่พลาดกันบ่อยที่สุด

เมื่อผสม `AND` กับ `OR` ในเงื่อนไขเดียวกัน PostgreSQL จะประมวลผล **`AND` ก่อน `OR` เสมอ** (เหมือนคณิตศาสตร์ที่คูณมาก่อนบวก) ลำดับความสำคัญจากสูงไปต่ำคือ:

| ลำดับ | Operator |
|-------|----------|
| 1 (สูงสุด) | `NOT` |
| 2 | `AND` |
| 3 (ต่ำสุด) | `OR` |

ลองดูตัวอย่างที่แสดงให้เห็นความแตกต่างชัดเจน:

```sql
-- แบบไม่มีวงเล็บ: AND ทำงานก่อน OR
-- ความหมายจริงคือ: category_id = 1 OR (category_id = 2 AND price > 70)
SELECT product_name, category_id, price
FROM products
WHERE category_id = 1 OR category_id = 2 AND price > 70;
```

```
   product_name   | category_id | price
-------------------+-------------+-------
 Espresso          |           1 | 55.00
 Americano         |           1 | 60.00
 Cappuccino        |           1 | 70.00
 Latte             |           1 | 70.00
 Iced Latte        |           2 | 75.00
 Cold Brew         |           2 | 85.00
(6 rows)
```

สังเกตว่าสินค้าทุกตัวใน category 1 ติดมาหมด (แม้ราคาจะไม่เกิน 70) เพราะเงื่อนไข `category_id = 2 AND price > 70` ถูกจัดกลุ่มไว้ด้วยกันก่อน แล้วค่อยเอาไป `OR` กับ `category_id = 1`

ถ้าต้องการ "เฉพาะสินค้า category 1 หรือ 2 **ที่ราคามากกว่า 70 เท่านั้น**" ต้องใส่วงเล็บเพื่อบังคับลำดับ:

```sql
-- มีวงเล็บ: บังคับให้ OR ทำงานก่อน
SELECT product_name, category_id, price
FROM products
WHERE (category_id = 1 OR category_id = 2) AND price > 70;
```

```
 product_name | category_id | price
--------------+-------------+-------
 Iced Latte   |           2 | 75.00
 Cold Brew    |           2 | 85.00
(2 rows)
```

ผลลัพธ์ต่างกันโดยสิ้นเชิง (6 แถว vs 2 แถว) จากเงื่อนไขที่ดูคล้ายกันมาก นี่คือเหตุผลที่ **ควรใส่วงเล็บทุกครั้งที่ผสม `AND` กับ `OR`** แม้จะมั่นใจเรื่องลำดับความสำคัญก็ตาม เพราะวงเล็บช่วยให้โค้ดอ่านง่ายขึ้นและลดโอกาสเขียนผิดของทั้งตัวเราเองและเพื่อนร่วมทีม

> **แนวทางปฏิบัติที่ดี (best practice)**: ใส่วงเล็บเสมอเมื่อผสม `AND`/`OR` มากกว่าหนึ่งชนิดในเงื่อนไขเดียว อย่าอาศัยความจำเรื่อง precedence — โค้ดที่ต้องตีความยากคือโค้ดที่มีบั๊กได้ง่าย

---

## Step 103: BETWEEN...AND สำหรับช่วงตัวเลขและวันที่

`BETWEEN...AND` เป็น syntax ย่อสำหรับเงื่อนไข "อยู่ในช่วง" ซึ่งเทียบเท่ากับการใช้ `>=` และ `<=` ร่วมกัน โดย **รวมค่าขอบทั้งสองด้าน (inclusive)**

```sql
column BETWEEN low AND high
-- เทียบเท่ากับ
column >= low AND column <= high
```

### ตัวอย่างที่ 1: BETWEEN กับตัวเลข

```sql
SELECT product_name, price
FROM products
WHERE price BETWEEN 50 AND 70;
```

```
 product_name   | price
-----------------+-------
 Espresso        | 55.00
 Americano       | 60.00
 Cappuccino      | 70.00
 Latte           | 70.00
 Iced Americano  | 65.00
 Thai Tea        | 55.00
 Earl Grey       | 60.00
(7 rows)
```

### ตัวอย่างที่ 2: NOT BETWEEN

```sql
SELECT product_name, price
FROM products
WHERE price NOT BETWEEN 50 AND 70;
```

```
   product_name    | price
--------------------+-------
 Iced Latte         | 75.00
 Cold Brew          | 85.00
 Green Tea Latte    | 75.00
 Croissant          | 45.00
 Chocolate Muffin   | 40.00
 Cheese Cake        | 95.00
 Potato Chips       | 35.00
 Cookies            | 30.00
(8 rows)
```

### ตัวอย่างที่ 3: BETWEEN กับวันที่ — กับดักที่ต้องระวัง!

`BETWEEN` กับคอลัมน์ชนิด `DATE` ใช้งานตรงไปตรงมา แต่กับคอลัมน์ชนิด `TIMESTAMP` (ที่มีเวลาแฝงอยู่) เป็นจุดที่มือใหม่พลาดบ่อยที่สุด

ลองดูตัวอย่างนี้: ต้องการดึงคำสั่งซื้อ "ตั้งแต่วันที่ 1 ถึงวันที่ 2 มิถุนายน" (สองวัน)

```sql
SELECT order_id, order_date, total_amount
FROM orders
WHERE order_date BETWEEN '2024-06-01' AND '2024-06-02';
```

```
 order_id |      order_date      | total_amount
----------+-----------------------+--------------
        1 | 2024-06-01 08:15:00  |       155.00
        2 | 2024-06-01 09:30:00  |        70.00
(2 rows)
```

**สังเกตว่า order_id 3 และ 4 ซึ่งเกิดขึ้นวันที่ 2 มิถุนายน (08:00 น. และ 14:20 น.) หายไป!** เหตุผลคือเมื่อเขียน `'2024-06-02'` โดยไม่ระบุเวลา PostgreSQL จะตีความเป็น `'2024-06-02 00:00:00'` (เที่ยงคืนพอดี) ดังนั้นเงื่อนไขจริงๆ คือ

```
order_date >= '2024-06-01 00:00:00' AND order_date <= '2024-06-02 00:00:00'
```

ซึ่งครอบคลุมแค่ "ถึงเที่ยงคืนของวันที่ 2" เท่านั้น ไม่ใช่ทั้งวันที่ 2!

**วิธีแก้ที่ถูกต้อง** คือใช้ `<` กับวันถัดไป แทน `BETWEEN`:

```sql
SELECT order_id, order_date, total_amount
FROM orders
WHERE order_date >= '2024-06-01'
  AND order_date <  '2024-06-03';  -- ใช้ < วันถัดจากวันสุดท้ายที่ต้องการ
```

```
 order_id |      order_date      | total_amount
----------+-----------------------+--------------
        1 | 2024-06-01 08:15:00  |       155.00
        2 | 2024-06-01 09:30:00  |        70.00
        3 | 2024-06-02 08:00:00  |       195.00
        4 | 2024-06-02 14:20:00  |        60.00
(4 rows)
```

ครบทั้ง 4 แถวตามที่ต้องการ

> **กฎจำง่าย**: เมื่อกรองช่วงวันที่บนคอลัมน์ `TIMESTAMP`/`TIMESTAMPTZ` ให้ใช้ `>= วันเริ่มต้น AND < วันถัดจากวันสุดท้าย` เสมอ แทนการใช้ `BETWEEN` ตรงๆ กับวันที่แบบไม่มีเวลา เพื่อไม่ให้ข้อมูลของ "วันสุดท้าย" ตกหล่นไป `BETWEEN` ยังใช้ได้ดีถ้าคอลัมน์เป็น `DATE` ล้วนๆ (ไม่มีเวลา) เพราะไม่มีปัญหาเรื่องเวลาแฝง

---

## Step 104: IN และ NOT IN

`IN` ใช้ตรวจสอบว่าค่าคอลัมน์ตรงกับค่าใดค่าหนึ่งในลิสต์ที่กำหนด เป็นทางลัดที่สะอาดกว่าการเขียน `OR` ซ้ำๆ หลายตัว

```sql
column IN (value1, value2, value3, ...)
-- เทียบเท่ากับ
column = value1 OR column = value2 OR column = value3 OR ...
```

### ตัวอย่างที่ 1: IN แทน OR หลายตัว

```sql
-- แบบเดิมด้วย OR
SELECT product_name, category_id
FROM products
WHERE category_id = 1 OR category_id = 3;

-- แบบใหม่ด้วย IN (ผลลัพธ์เหมือนกันทุกประการ อ่านง่ายกว่า)
SELECT product_name, category_id
FROM products
WHERE category_id IN (1, 3);
```

ทั้งสองคำสั่งให้ผลลัพธ์ 7 แถวเหมือนกัน (Espresso, Americano, Cappuccino, Latte, Thai Tea, Green Tea Latte, Earl Grey) แต่ `IN` อ่านง่ายกว่ามากเมื่อมีค่าที่ต้องเทียบเยอะๆ ลองนึกภาพถ้าต้องเขียน `OR` 10 ตัว เทียบกับ `IN (v1, v2, ..., v10)`

### ตัวอย่างที่ 2: NOT IN

```sql
SELECT product_name, category_id
FROM products
WHERE category_id NOT IN (1, 2, 3);
```

```
   product_name    | category_id
--------------------+-------------
 Croissant          |           4
 Chocolate Muffin   |           4
 Cheese Cake        |           4
 Potato Chips       |           5
(4 rows)
```

สังเกตว่า **Cookies (product_id 15) ไม่ปรากฏในผลลัพธ์** ทั้งที่ `category_id` เป็น `NULL` ซึ่งไม่ได้อยู่ใน `(1, 2, 3)` เลย นี่เป็นเพราะ `NULL NOT IN (...)` จะได้ผลเป็น `UNKNOWN` ไม่ใช่ `TRUE` (รายละเอียดเรื่อง `NULL` อยู่ใน Step 107)

### ตัวอย่างที่ 3: IN กับ subquery

จุดแข็งอีกอย่างของ `IN` คือใช้ร่วมกับ subquery ได้ทันที ทำให้กรองข้อมูลจากอีกตารางหนึ่งได้โดยไม่ต้อง join:

```sql
-- หาชื่อสินค้าที่เคยถูกสั่งซื้อใน order_id = 10
SELECT product_name, price
FROM products
WHERE product_id IN (
    SELECT product_id FROM order_items WHERE order_id = 10
);
```

```
 product_name  | price
----------------+-------
 Cold Brew      | 85.00
 Croissant      | 45.00
 Cheese Cake    | 95.00
(3 rows)
```

(เรื่อง subquery แบบละเอียดจะเรียนในบทถัดๆ ไป ตอนนี้ขอให้เข้าใจแค่ว่า `IN` รับผลลัพธ์จาก subquery เป็นลิสต์ได้)

### กับดักสำคัญ: NOT IN กับ subquery ที่มี NULL

นี่คือหนึ่งในกับดักที่อันตรายที่สุดของ PostgreSQL (และ SQL ทั่วไป) ลองดูตัวอย่างนี้:

```sql
-- ตั้งใจจะหา "ลูกค้าที่ customer_id ไม่ปรากฏในตาราง orders เลย"
SELECT customer_id, first_name
FROM customers
WHERE customer_id NOT IN (SELECT customer_id FROM orders);
```

```
(0 rows)
```

**ได้ 0 แถว ทั้งที่ควรจะมีลูกค้าบางคนที่ไม่เคยสั่งซื้อเลย!** สาเหตุคือตาราง `orders` มี order_id = 13 ที่ `customer_id` เป็น `NULL` (guest order) ทำให้ subquery คืนค่ารายการที่มี `NULL` ปนอยู่ เมื่อ `NULL` ปรากฏใน list ของ `NOT IN` ผลลัพธ์ของเงื่อนไขทั้งหมดจะกลายเป็น `UNKNOWN` สำหรับทุกแถว ทำให้ `WHERE` กรองทุกแถวทิ้งหมด

**วิธีแก้**: กรอง `NULL` ออกจาก subquery ก่อน หรือใช้ `NOT EXISTS` แทน (ปลอดภัยกว่าเสมอ):

```sql
-- วิธีที่ 1: กรอง NULL ออกจาก subquery
SELECT customer_id, first_name
FROM customers
WHERE customer_id NOT IN (
    SELECT customer_id FROM orders WHERE customer_id IS NOT NULL
);

-- วิธีที่ 2 (แนะนำ): ใช้ NOT EXISTS แทน ปลอดภัยจาก NULL โดยธรรมชาติ
SELECT c.customer_id, c.first_name
FROM customers c
WHERE NOT EXISTS (
    SELECT 1 FROM orders o WHERE o.customer_id = c.customer_id
);
```

```
 customer_id | first_name
-------------+------------
          11 | วราภรณ์
(1 row)
```

> **กฎจำง่าย**: ถ้า subquery ที่ใช้กับ `NOT IN` มีโอกาสคืนค่า `NULL` ได้ ให้เปลี่ยนไปใช้ `NOT EXISTS` เสมอ หรืออย่างน้อยต้องใส่ `IS NOT NULL` กรองใน subquery ก่อน มิเช่นนั้นอาจได้ผลลัพธ์ว่างเปล่าโดยไม่รู้ตัว

---

## Step 105: LIKE และ ILIKE — Pattern Matching

`LIKE` ใช้ค้นหาข้อความที่ตรงกับ "แพทเทิร์น" (pattern) โดยใช้ wildcard สองตัว:

| Wildcard | ความหมาย | ตัวอย่าง |
|----------|----------|----------|
| `%`      | ตัวอักษรกี่ตัวก็ได้ (รวมถึงไม่มีเลย) | `'lat%'` ตรงกับ `latte`, `latency`, `lat` |
| `_`      | ตัวอักษรตัวเดียวเท่านั้น | `'c_t'` ตรงกับ `cat`, `cot`, `cut` |

`LIKE` เป็น **case-sensitive** ส่วน `ILIKE` (PostgreSQL extension นอกมาตรฐาน SQL) ทำงานเหมือนกันทุกอย่างแต่ **case-insensitive**

### ตัวอย่างที่ 1: `%` — ค้นหาแบบ "ลงท้ายด้วย"

```sql
SELECT first_name, last_name, email
FROM customers
WHERE email LIKE '%@gmail.com';
```

```
 first_name | last_name | email
------------+-----------+------------------------
 อนันต์      | สุขสันต์   | anan@gmail.com
 กมลชนก     | แสงทอง    | kamolchanok@gmail.com
(2 rows)
```

### ตัวอย่างที่ 2: `%` — ค้นหาแบบ "ขึ้นต้นด้วย"

```sql
SELECT first_name, last_name
FROM customers
WHERE first_name LIKE 'ส%';
```

```
 first_name | last_name
------------+-----------
 สมชาย      | ใจดี
 สมหญิง     | รักเรียน
 สุดา        | ใจงาม
(3 rows)
```

### ตัวอย่างที่ 3: `%` — ค้นหาแบบ "มีคำนี้อยู่ตรงไหนก็ได้"

```sql
SELECT product_name
FROM products
WHERE product_name LIKE '%Latte%';
```

```
   product_name
-------------------
 Latte
 Iced Latte
 Green Tea Latte
(3 rows)
```

### ตัวอย่างที่ 4: ILIKE — ไม่สนตัวพิมพ์ใหญ่เล็ก

```sql
-- LIKE (case-sensitive) หาคำว่า 'latte' ตัวพิมพ์เล็กล้วน จะไม่เจออะไรเลย
SELECT product_name FROM products WHERE product_name LIKE '%latte%';
```

```
(0 rows)
```

```sql
-- ILIKE (case-insensitive) เจอทุกตัวที่มีคำว่า latte ไม่ว่าตัวพิมพ์จะเป็นแบบไหน
SELECT product_name FROM products WHERE product_name ILIKE '%latte%';
```

```
   product_name
-------------------
 Latte
 Iced Latte
 Green Tea Latte
(3 rows)
```

### ตัวอย่างที่ 5: `_` — ตัวแทนตัวอักษรตัวเดียว

```sql
SELECT product_name
FROM products
WHERE product_name LIKE '_atte';
```

```
 product_name
---------------
 Latte
(1 row)
```

`Iced Latte` และ `Green Tea Latte` ไม่ตรงกับแพทเทิร์นนี้ เพราะแพทเทิร์น `_atte` มีความยาวคงที่ 5 ตัวอักษรเท่านั้น (ตัวแทน 1 ตัว + `atte` 4 ตัว) ส่วน `Latte` มีความยาวพอดี 5 ตัวอักษร

> **หมายเหตุ**: ถ้าต้องการค้นหาตัวอักษร `%` หรือ `_` ตามตัวอักษรจริงๆ (ไม่ใช่ wildcard) ต้องใช้ escape character เช่น `LIKE '50\%' ESCAPE '\'` หรือใช้ฟังก์ชัน `position()`/`strpos()` แทน

---

## Step 106: Regular Expression Matching เบื้องต้น (~, ~*, !~, !~*)

เมื่อ `LIKE`/`ILIKE` ไม่ยืดหยุ่นพอ (เช่น ต้องการระบุช่วงตัวอักษร, ทางเลือกหลายแบบ, หรือ anchor ตำแหน่งซับซ้อน) PostgreSQL มี operator สำหรับ POSIX regular expression ให้ใช้โดยตรง:

| Operator | ความหมาย |
|----------|----------|
| `~`      | ตรงกับ pattern (case-sensitive) |
| `~*`     | ตรงกับ pattern (case-insensitive) |
| `!~`     | ไม่ตรงกับ pattern (case-sensitive) |
| `!~*`    | ไม่ตรงกับ pattern (case-insensitive) |

### ตัวอย่างที่ 1: ตรวจรูปแบบเบอร์โทรศัพท์

หาลูกค้าที่เบอร์โทรขึ้นต้นด้วย `080`, `081`, `082`, หรือ `083` (ค่าย AIS ยุคเก่า) โดยใช้ character class `[0-3]`:

```sql
SELECT first_name, phone
FROM customers
WHERE phone ~ '^08[0-3]';
```

```
 first_name | phone
------------+-------------
 สมชาย      | 0812345678
 สมหญิง     | 0823456789
 มานี       | 0834567890
 วราภรณ์    | 0801112222
(4 rows)
```

สังเกตว่า `^` หมายถึง "ขึ้นต้นด้วย" (anchor) และวิชัย (customer_id 3) ไม่ปรากฏเพราะ `phone` เป็น `NULL`

### ตัวอย่างที่ 2: case-sensitive vs case-insensitive

```sql
-- ~ (case-sensitive): ค้นหา 'tea' ตัวพิมพ์เล็กล้วน ไม่เจอเพราะข้อมูลจริงเป็น 'Tea' (T ใหญ่)
SELECT product_name FROM products WHERE product_name ~ 'tea';
```

```
(0 rows)
```

```sql
-- ~* (case-insensitive): เจอทั้งสองตัวที่มีคำว่า Tea อยู่
SELECT product_name FROM products WHERE product_name ~* 'tea';
```

```
   product_name
-------------------
 Thai Tea
 Green Tea Latte
(2 rows)
```

### ตัวอย่างที่ 3: `!~*` — ไม่ตรงกับ pattern (ไม่สนตัวพิมพ์)

```sql
SELECT product_name
FROM products
WHERE product_name !~* 'tea';
```

ผลลัพธ์คือสินค้าทั้งหมด 13 รายการที่ชื่อไม่มีคำว่า "tea" ปนอยู่เลย (ไม่ว่าตัวพิมพ์ใหญ่เล็ก)

### เลือกใช้ LIKE หรือ Regex เมื่อไร?

| สถานการณ์ | แนะนำให้ใช้ |
|-----------|-------------|
| แค่ต้องการ "ขึ้นต้นด้วย", "ลงท้ายด้วย", "มีคำนี้อยู่" | `LIKE` / `ILIKE` — อ่านง่ายกว่า เร็วกว่าเล็กน้อย |
| ต้องการระบุช่วงตัวอักษร เช่น ตัวเลขเท่านั้น, ตัวพิมพ์ใหญ่เท่านั้น | Regex (`~`) ด้วย character class `[0-9]`, `[A-Z]` |
| ต้องการ "ทางเลือกหลายแบบ" เช่น ขึ้นต้นด้วย A หรือ B หรือ C | Regex ด้วย `^(A|B|C)` |
| ต้องการ validate รูปแบบซับซ้อน เช่น อีเมล, เบอร์โทร | Regex |

> **หมายเหตุเรื่อง performance**: ทั้ง `LIKE '%xxx'` (wildcard นำหน้า) และ regex ทั่วไปมักใช้ index แบบ B-tree มาตรฐานไม่ได้ (ต้อง scan ทั้งตาราง) รายละเอียดเรื่องการทำ index ให้ pattern matching เร็วขึ้น (เช่น `pg_trgm`, GIN index) จะอยู่ใน Part 041-045

---

## Step 107: IS NULL และ IS NOT NULL

`NULL` ใน SQL หมายถึง "ไม่มีค่า" หรือ "ไม่ทราบค่า" (unknown) ซึ่ง **ไม่เท่ากับ** ค่าว่าง (`''`), ค่าศูนย์ (`0`), หรือค่าอะไรทั้งสิ้น — แม้แต่ `NULL` ด้วยกันเองก็ยังไม่ถือว่า "เท่ากัน"

### ทำไม `= NULL` ใช้ไม่ได้

PostgreSQL ใช้ตรรกะสามค่า (three-valued logic): `TRUE`, `FALSE`, และ `UNKNOWN` การเปรียบเทียบใดๆ กับ `NULL` ด้วย `=` จะได้ผลเป็น `UNKNOWN` เสมอ ไม่ใช่ `TRUE` หรือ `FALSE` และ `WHERE` จะเก็บไว้เฉพาะแถวที่เงื่อนไขเป็น `TRUE` เท่านั้น แถวที่ได้ `UNKNOWN` จะถูกตัดทิ้งเสมอ

ลองพิสูจน์ด้วยตัวอย่าง:

```sql
-- ผิด: พยายามหาลูกค้าที่ไม่มีเบอร์โทรด้วย = NULL
SELECT first_name, phone
FROM customers
WHERE phone = NULL;
```

```
(0 rows)
```

ได้ 0 แถวเสมอ ไม่ว่าข้อมูลจริงจะมี `NULL` กี่แถวก็ตาม เพราะ `phone = NULL` ประเมินเป็น `UNKNOWN` ทุกครั้ง

```sql
-- ถูกต้อง: ใช้ IS NULL
SELECT first_name, phone
FROM customers
WHERE phone IS NULL;
```

```
 first_name | phone
------------+-------
 วิชัย       |
(1 row)
```

### ตัวอย่างที่ 2: IS NOT NULL

```sql
SELECT first_name, birth_date
FROM customers
WHERE birth_date IS NOT NULL;
```

ผลลัพธ์คือลูกค้าทั้งหมด 10 คน ยกเว้นสุดา (customer_id 6) ที่ `birth_date` เป็น `NULL`

### ตัวอย่างที่ 3: กรณีใช้งานจริง — Guest Order

```sql
-- หาคำสั่งซื้อที่ไม่ผูกกับบัญชีลูกค้า (guest checkout)
SELECT order_id, order_date, total_amount
FROM orders
WHERE customer_id IS NULL;
```

```
 order_id |      order_date      | total_amount
----------+-----------------------+--------------
       13 | 2024-06-09 12:00:00  |        75.00
(1 row)
```

### ตัวอย่างที่ 4: products ที่ยังไม่ได้จัดหมวดหมู่

```sql
SELECT product_name, category_id
FROM products
WHERE category_id IS NULL;
```

```
 product_name | category_id
--------------+-------------
 Cookies      |
(1 row)
```

> **สรุปกฎเหล็ก**: ใช้ `IS NULL` / `IS NOT NULL` เท่านั้นในการตรวจสอบค่า `NULL` ห้ามใช้ `=` หรือ `<>` กับ `NULL` เด็ดขาด เพราะจะได้ผลลัพธ์ผิดเสมอ (0 แถว หรือแถวที่ไม่ครบ) โดยไม่มี error ใดๆ แจ้งเตือน — เป็นบั๊กที่ตรวจจับยากที่สุดแบบหนึ่งใน SQL

---

## Step 108: การรวม Operators หลายตัวใน Query ซับซ้อน

ในงานจริง แทบไม่มี query ไหนใช้ operator แค่ตัวเดียว ลองดูตัวอย่าง filter ที่สมจริงสำหรับระบบร้านกาแฟ

### ตัวอย่างที่ 1: สินค้าที่พร้อมขายในหมวดกาแฟ ราคากลางๆ

โจทย์: "หาสินค้าในหมวด Hot Coffee หรือ Cold Coffee ที่ราคาอยู่ระหว่าง 60-80 บาท และยังเปิดขายอยู่ และมีของในสต็อก"

```sql
SELECT product_name, category_id, price, stock_quantity
FROM products
WHERE category_id IN (1, 2)
  AND price BETWEEN 60 AND 80
  AND is_active = TRUE
  AND stock_quantity > 0;
```

```
   product_name    | category_id | price | stock_quantity
--------------------+-------------+-------+----------------
 Americano          |           1 | 60.00 |            120
 Cappuccino         |           1 | 70.00 |             80
 Latte              |           1 | 70.00 |             90
 Iced Americano     |           2 | 65.00 | 150
 Iced Latte         |           2 | 75.00 | 100
(5 rows)
```

### ตัวอย่างที่ 2: ลูกค้าระดับพรีเมียมที่สมัครปี 2023 และไม่ใช้อีเมล gmail

โจทย์: "หาลูกค้าระดับ gold หรือ platinum ที่สมัครสมาชิกในปี 2023 และอีเมลไม่ใช่ gmail (เพื่อส่ง direct mail แบบจดหมายองค์กร)"

```sql
SELECT first_name, last_name, member_tier, email, created_at
FROM customers
WHERE member_tier IN ('gold', 'platinum')
  AND created_at BETWEEN '2023-01-01' AND '2023-12-31'
  AND email NOT LIKE '%gmail.com';
```

```
 first_name | last_name | member_tier |        email         | created_at
------------+-----------+-------------+-----------------------+------------
 สมชาย      | ใจดี      | gold        | somchai@example.com   | 2023-01-10
 วิชัย       | มั่งมี     | gold        | wichai@example.com    | 2023-01-20
(2 rows)
```

สังเกตว่าอนันต์ (gold แต่สมัคร 2022-12-01) และกมลชนก (platinum แต่ใช้ gmail) ถูกกรองออกไปตามเงื่อนไข

> เนื่องจาก `created_at` ในตัวอย่างนี้เป็นชนิด `DATE` ล้วน (ไม่มีเวลา) การใช้ `BETWEEN` จึงปลอดภัย ไม่มีปัญหาแบบ Step 103

### ตัวอย่างที่ 3: คำสั่งซื้อมูลค่าสูงที่ชำระผ่านช่องทางดิจิทัล

โจทย์: "หาคำสั่งซื้อที่สำเร็จแล้ว มูลค่ามากกว่า 100 บาท และจ่ายผ่านบัตรเครดิตหรือ PromptPay เท่านั้น"

```sql
SELECT order_id, order_date, total_amount, payment_method
FROM orders
WHERE order_status = 'completed'
  AND total_amount > 100
  AND payment_method IN ('credit_card', 'promptpay');
```

```
 order_id |      order_date      | total_amount | payment_method
----------+-----------------------+---------------+----------------
        3 | 2024-06-02 08:00:00  |        195.00 | promptpay
        6 | 2024-06-03 16:45:00  |        210.00 | promptpay
        8 | 2024-06-05 11:00:00  |        150.00 | credit_card
       10 | 2024-06-06 09:00:00  |        310.00 | credit_card
(4 rows)
```

order_id = 1 มูลค่า 155 บาทเกิน 100 เหมือนกัน แต่จ่ายด้วย `cash` จึงไม่เข้าเงื่อนไข

จะเห็นว่าเมื่อเงื่อนไขซับซ้อนขึ้น การจัดวางแต่ละเงื่อนไขคนละบรรทัด (ตามตัวอย่างข้างต้น) ช่วยให้อ่านและแก้ไข query ได้ง่ายกว่าเขียนรวมกันเป็นบรรทัดเดียวมาก

---

## Step 109: Sargable vs Non-Sargable WHERE Clauses (เกริ่นนำ)

คำว่า **sargable** ย่อมาจาก "Search ARGument ABLE" หมายถึงเงื่อนไขใน `WHERE` ที่ PostgreSQL (หรือฐานข้อมูลใดๆ) **สามารถใช้ index ช่วยค้นหาได้โดยตรง** โดยไม่ต้องอ่านข้อมูลทุกแถวในตาราง (sequential scan)

ในบทนี้ขอแค่แนะนำแนวคิดคร่าวๆ ก่อน — รายละเอียดเชิงลึกเรื่อง index, `EXPLAIN`, และการอ่าน query plan จะอยู่ใน **Part 041-045**

### ตัวอย่างเงื่อนไขที่ไม่ sargable (หลีกเลี่ยงถ้าเป็นไปได้)

```sql
-- ไม่ sargable: ใช้ฟังก์ชัน/นิพจน์คำนวณกับคอลัมน์ฝั่งซ้าย
-- แม้ price จะมี index อยู่ ก็ใช้ index ไม่ได้ เพราะต้องคำนวณ price * 1.07 ทุกแถวก่อนเปรียบเทียบ
SELECT product_name FROM products WHERE price * 1.07 > 100;

-- ไม่ sargable: ใช้ฟังก์ชันครอบคอลัมน์
SELECT * FROM customers WHERE LOWER(email) = 'somchai@example.com';

-- ไม่ sargable: LIKE ที่มี wildcard นำหน้า
SELECT * FROM products WHERE product_name LIKE '%latte';
```

### ตัวอย่างเงื่อนไขที่ sargable (ใช้ index ได้ ถ้ามี index อยู่)

```sql
-- sargable: ย้ายการคำนวณไปไว้ฝั่งขวา (ค่าคงที่) แทน
SELECT product_name FROM products WHERE price > 100 / 1.07;

-- sargable: เทียบตรงๆ โดยไม่ครอบฟังก์ชันที่คอลัมน์
SELECT * FROM customers WHERE email = 'somchai@example.com';

-- sargable: LIKE ที่ wildcard อยู่ท้ายเท่านั้น (prefix match)
SELECT * FROM products WHERE product_name LIKE 'Iced%';
```

หลักการง่ายๆ ที่จำไว้ก่อนได้เลยคือ: **"อย่าเอาคอลัมน์ไปครอบด้วยฟังก์ชันหรือคำนวณฝั่งซ้ายของ operator"** ถ้าจำเป็นต้องคำนวณ ให้ย้ายไปคำนวณฝั่งค่าคงที่ (ฝั่งขวา) แทน เพราะ query planner ของ PostgreSQL จะพยายามจับคู่รูปแบบ `column operator constant` กับ index ที่มีอยู่ ถ้าคอลัมน์ถูกครอบด้วยฟังก์ชัน มันจะไม่รู้ว่าจะ "เดา" ค่าที่ตรงกันในดัชนีได้อย่างไร จึงต้องไล่อ่านสแกนทุกแถวแทน

> เราจะเรียนวิธีสร้าง index, การอ่านผลลัพธ์จาก `EXPLAIN ANALYZE`, และวิธีสร้าง functional index/expression index เพื่อทำให้ query ประเภทที่ 1 กลายเป็น sargable ได้ ในหมวด **Part 041-045: Indexing** โดยละเอียด ตอนนี้เพียงแค่ขอให้เริ่มมีสัญชาตญาณว่า "การครอบฟังก์ชันที่คอลัมน์ใน WHERE" เป็นสิ่งที่ควรระวัง

---

## Step 110: แบบฝึกหัดรวม — Filter ระบบร้านกาแฟ

มาลองประกอบทุกสิ่งที่เรียนมาเข้าด้วยกันในสถานการณ์ที่ใกล้เคียงงานจริงมากขึ้น

### สถานการณ์ที่ 1: Dashboard สำหรับผู้จัดการร้าน — สินค้าที่ต้องเติมสต็อกด่วน

```sql
SELECT product_name, category_id, stock_quantity, is_active
FROM products
WHERE stock_quantity <= 20
  AND is_active = TRUE;
```

```
 product_name | category_id | stock_quantity | is_active
--------------+-------------+-----------------+-----------
 Cheese Cake  |           4 |              15 | t
(1 row)
```

(Potato Chips มี stock_quantity = 0 เช่นกัน แต่ `is_active = FALSE` แสดงว่าเลิกขายแล้ว จึงไม่ต้องเติมสต็อก)

### สถานการณ์ที่ 2: แคมเปญการตลาด — ลูกค้าที่ยังไม่มีข้อมูลครบ

```sql
-- ลูกค้าที่ต้องขอข้อมูลเพิ่มเติม (เบอร์โทรหรือวันเกิดยังไม่ครบ)
SELECT first_name, last_name, phone, birth_date
FROM customers
WHERE phone IS NULL OR birth_date IS NULL;
```

```
 first_name | last_name | phone        | birth_date
------------+-----------+--------------+------------
 วิชัย       | มั่งมี     |              | 1988-12-01
 สุดา        | ใจงาม     | 0856789012   |
(2 rows)
```

### สถานการณ์ที่ 3: ค้นหาสินค้าแบบ "auto-complete" จากคำที่พิมพ์บางส่วน

```sql
-- ผู้ใช้พิมพ์ "ic" ในช่องค้นหา ต้องการเจอสินค้าที่มีคำนี้อยู่ตรงไหนก็ได้ ไม่สนตัวพิมพ์ใหญ่เล็ก
SELECT product_name, price
FROM products
WHERE product_name ILIKE '%ic%';
```

```
 product_name    | price
------------------+-------
 Iced Americano   | 65.00
 Iced Latte       | 75.00
(2 rows)
```

### สถานการณ์ที่ 4: รายงานยอดขายที่ไม่นับคำสั่งซื้อที่ถูกยกเลิกหรือรอดำเนินการ

```sql
SELECT order_id, order_date, total_amount, order_status
FROM orders
WHERE order_status NOT IN ('cancelled', 'pending')
  AND order_date >= '2024-06-05'
  AND order_date <  '2024-06-09';
```

```
 order_id |      order_date      | total_amount | order_status
----------+-----------------------+---------------+--------------
        8 | 2024-06-05 11:00:00  |        150.00 | completed
       10 | 2024-06-06 09:00:00  |        310.00 | completed
       11 | 2024-06-07 08:20:00  |         65.00 | completed
       12 | 2024-06-08 15:15:00  |         40.00 | completed
(4 rows)
```

สังเกตว่า order_id = 9 (pending) และ order_id = 14 (cancelled, วันที่ 06-10 อยู่นอกช่วงอยู่แล้ว) ถูกตัดออก และเราใช้เทคนิคจาก Step 103 (`>=` กับ `<`) แทน `BETWEEN` เพราะ `order_date` เป็น timestamp

ทั้งสี่สถานการณ์นี้แสดงให้เห็นว่าในงานจริง เราแทบไม่เคยใช้ operator ตัวเดียวโดดๆ แต่จะผสมผสาน comparison, logical, `IN`, `LIKE`/`ILIKE`, `IS NULL`, และ `BETWEEN`/ช่วงวันที่เข้าด้วยกันเสมอ เพื่อตอบโจทย์ทางธุรกิจที่ซับซ้อน

---

## สรุปท้ายบท

ในบทนี้เราได้เรียนรู้แกนหลักของการกรองข้อมูลด้วย `WHERE` clause ซึ่งเป็นทักษะที่ใช้ในแทบทุก query ที่เขียนจริง:

- **Comparison operators** (`=`, `<>`/`!=`, `<`, `>`, `<=`, `>=`) เป็นพื้นฐานของการเปรียบเทียบค่า
- **Logical operators** (`AND`, `OR`, `NOT`) ใช้รวมหลายเงื่อนไขเข้าด้วยกัน โดย `AND` มีความสำคัญเหนือ `OR` เสมอ — **ใส่วงเล็บทุกครั้ง** เมื่อผสมกันเพื่อความชัดเจน
- **`BETWEEN...AND`** สะดวกสำหรับช่วงตัวเลขและวันที่ แต่ต้องระวังเรื่อง timestamp ที่มีเวลาแฝงอยู่ — ใช้ `>=` กับ `<` แทนเมื่อจำเป็น
- **`IN`/`NOT IN`** เป็นทางลัดของ `OR` หลายตัว ใช้กับ subquery ได้ด้วย แต่ต้องระวัง `NOT IN` เมื่อ list มี `NULL` ปน — ใช้ `NOT EXISTS` แทนถ้าไม่แน่ใจ
- **`LIKE`/`ILIKE`** ทำ pattern matching ด้วย `%` และ `_` โดย `ILIKE` ไม่สนตัวพิมพ์ใหญ่เล็ก
- **Regex (`~`, `~*`, `!~`, `!~*`)** ให้พลังในการ match pattern ที่ซับซ้อนกว่า `LIKE` ทำได้
- **`IS NULL`/`IS NOT NULL`** คือวิธีเดียวที่ถูกต้องในการตรวจสอบ `NULL` — ห้ามใช้ `= NULL` เด็ดขาด
- เงื่อนไขใน `WHERE` ควรเขียนให้ **sargable** เท่าที่ทำได้ (หลีกเลี่ยงการครอบฟังก์ชันที่คอลัมน์) เพื่อให้ใช้ประโยชน์จาก index ได้เต็มที่ — รายละเอียดเชิงลึกรออยู่ใน Part 041-045

ทักษะเหล่านี้จะถูกนำไปใช้ซ้ำแล้วซ้ำเล่าตลอดทั้งหลักสูตร ตั้งแต่การเขียน query ง่ายๆ ไปจนถึงการ optimize query ที่ซับซ้อนระดับ production

ใน **Part 012** เราจะเรียนรู้วิธีจัดเรียงผลลัพธ์ด้วย `ORDER BY` และจำกัดจำนวนแถวด้วย `LIMIT`/`OFFSET` ซึ่งมักใช้คู่กับ `WHERE` ที่เรียนในบทนี้เสมอ

---

## แบบฝึกหัด

พยายามเขียน query ด้วยตัวเองก่อนเปิดดูเฉลย ข้อมูลที่ใช้คือ schema ร้านกาแฟจากส่วน "เตรียมข้อมูล" ด้านบน

**ข้อ 1.** เขียน query หาสินค้าทั้งหมดที่ราคาต่ำกว่า 50 บาท

<details>
<summary>เฉลยข้อ 1</summary>

```sql
SELECT product_name, price
FROM products
WHERE price < 50;
```

```
   product_name    | price
--------------------+-------
 Croissant          | 45.00
 Chocolate Muffin   | 40.00
 Potato Chips       | 35.00
 Cookies            | 30.00
(4 rows)
```

</details>

---

**ข้อ 2.** เขียน query หาลูกค้าที่ระดับสมาชิกเป็น `'silver'` **และ** สมัครก่อนวันที่ 1 มิถุนายน 2023

<details>
<summary>เฉลยข้อ 2</summary>

```sql
SELECT first_name, last_name, member_tier, created_at
FROM customers
WHERE member_tier = 'silver'
  AND created_at < '2023-06-01';
```

```
 first_name | last_name | member_tier | created_at
------------+-----------+-------------+------------
 สมหญิง     | รักเรียน  | silver      | 2023-02-15
(1 row)
```

ประเสริฐ (silver) สมัคร 2023-06-15 และพิมพ์ใจ (silver) สมัคร 2023-09-01 ต่างเกินเงื่อนไขวันที่จึงไม่ติดผลลัพธ์

</details>

---

**ข้อ 3.** เขียน query หาสินค้าในหมวด Bakery (`category_id = 4`) โดยใช้เงื่อนไข **OR** กับหมวด Snack (`category_id = 5`) ที่ราคา**มากกว่า** 40 บาท จงพิจารณาว่าต้องใส่วงเล็บหรือไม่ เพื่อให้ได้ "สินค้าในหมวด Bakery หรือ Snack ที่ราคามากกว่า 40 บาททั้งคู่"

<details>
<summary>เฉลยข้อ 3</summary>

ต้องใส่วงเล็บ เพราะถ้าไม่ใส่ `AND` จะทำงานก่อน `OR` ทำให้เงื่อนไข `category_id = 5 AND price > 40` ถูกจับคู่กันก่อน แล้วค่อย `OR` กับ `category_id = 4` (ทำให้สินค้า Bakery ทุกราคาติดมาหมด ไม่ถูกกรองด้วยราคา)

```sql
SELECT product_name, category_id, price
FROM products
WHERE (category_id = 4 OR category_id = 5)
  AND price > 40;
```

```
 product_name | category_id | price
--------------+-------------+-------
 Croissant    |           4 | 45.00
 Cheese Cake  |           4 | 95.00
(2 rows)
```

Chocolate Muffin (40.00) ไม่ผ่านเพราะราคาไม่ได้ "มากกว่า" 40 พอดี และ Potato Chips (35.00, หมวด 5) ก็ราคาต่ำกว่าเกณฑ์เช่นกัน

</details>

---

**ข้อ 4.** เขียน query หาคำสั่งซื้อทั้งหมดที่เกิดขึ้นในวันที่ 3 มิถุนายน 2024 (ทั้งวัน) โดยไม่ใช้ `BETWEEN` ตรงๆ กับวันที่แบบไม่มีเวลา (เพื่อหลีกเลี่ยงกับดักจาก Step 103)

<details>
<summary>เฉลยข้อ 4</summary>

```sql
SELECT order_id, order_date, total_amount
FROM orders
WHERE order_date >= '2024-06-03'
  AND order_date <  '2024-06-04';
```

```
 order_id |      order_date      | total_amount
----------+-----------------------+--------------
        5 | 2024-06-03 10:10:00  |        45.00
        6 | 2024-06-03 16:45:00  |       210.00
(2 rows)
```

ถ้าใช้ `BETWEEN '2024-06-03' AND '2024-06-03'` จะได้ผลลัพธ์ผิด (0 แถว) เพราะ `'2024-06-03'` ถูกตีความเป็นเที่ยงคืน ทำให้ไม่มีคำสั่งซื้อใดตรงกับเงื่อนไขเลย (เนื่องจากทั้งสอง order เกิดหลังเที่ยงคืน)

</details>

---

**ข้อ 5.** เขียน query หาสินค้าที่อยู่ในหมวด Hot Coffee, Cold Coffee, หรือ Tea (`category_id` 1, 2, 3) โดยใช้ `IN` แทนการเขียน `OR` สามตัว

<details>
<summary>เฉลยข้อ 5</summary>

```sql
SELECT product_name, category_id
FROM products
WHERE category_id IN (1, 2, 3);
```

```
   product_name    | category_id
--------------------+-------------
 Espresso           |           1
 Americano          |           1
 Cappuccino         |           1
 Latte              |           1
 Iced Americano     |           2
 Iced Latte         |           2
 Cold Brew          |           2
 Thai Tea           |           3
 Green Tea Latte    |           3
 Earl Grey          |           3
(10 rows)
```

</details>

---

**ข้อ 6.** อธิบายว่าทำไม query ต่อไปนี้อาจให้ผลลัพธ์ที่ไม่คาดคิด (คืน 0 แถว) แล้วเขียนใหม่ให้ถูกต้อง

```sql
SELECT product_name FROM products
WHERE product_id NOT IN (SELECT product_id FROM order_items WHERE order_id = 9);
```

<details>
<summary>เฉลยข้อ 6</summary>

query ข้างต้น**ไม่ได้**มีปัญหาเรื่อง `NULL` เพราะ `order_items.product_id` เป็นคอลัมน์ `NOT NULL` (มี `FOREIGN KEY` และไม่มีการ insert ค่า `NULL`) และ order_id = 9 มีแค่ 1 รายการ (product_id = 13) ดังนั้น query นี้จะทำงานถูกต้องและคืนสินค้าทั้งหมด 14 รายการที่ไม่ใช่ Cheese Cake

แต่โดยทั่วไป หากไม่แน่ใจว่าคอลัมน์ใน subquery ของ `NOT IN` มีโอกาสเป็น `NULL` หรือไม่ **แนวทางที่ปลอดภัยกว่าเสมอ** คือเขียนด้วย `NOT EXISTS`:

```sql
SELECT product_name
FROM products p
WHERE NOT EXISTS (
    SELECT 1 FROM order_items oi
    WHERE oi.product_id = p.product_id AND oi.order_id = 9
);
```

ผลลัพธ์จะเหมือนกัน (14 แถว) แต่ `NOT EXISTS` จะไม่มีวันพังแม้ subquery จะมี `NULL` ปนอยู่ก็ตาม จึงเป็นนิสัยการเขียนโค้ดที่ควรฝึกไว้ตั้งแต่ต้น

</details>

---

**ข้อ 7.** เขียน query หาลูกค้าที่ชื่อ (`first_name`) มี 4 ตัวอักษรพอดี และขึ้นต้นด้วย `'มา'`

<details>
<summary>เฉลยข้อ 7</summary>

```sql
SELECT first_name, last_name
FROM customers
WHERE first_name LIKE 'มา__';
```

```
 first_name | last_name
------------+-----------
 มานี       | มีนา
(1 row)
```

ใช้ `_` สองตัวแทนอักขระอีก 2 ตัวหลัง `มา` เพื่อบังคับความยาวรวมเป็น 4 ตัวอักษรพอดี ("มานี" = ม-า-น-ี รวม 4 ตัวอักษร)

</details>

---

**ข้อ 8.** เขียน query โดยใช้ regular expression หาลูกค้าที่อีเมลเป็นโดเมน `example.com` **หรือ** `yahoo.com` เท่านั้น (ไม่รวม gmail.com)

<details>
<summary>เฉลยข้อ 8</summary>

```sql
SELECT first_name, email
FROM customers
WHERE email ~ '@(example|yahoo)\.com$';
```

```
 first_name | email
------------+------------------------
 สมชาย      | somchai@example.com
 สมหญิง     | somying@example.com
 วิชัย       | wichai@example.com
 มานี       | manee@example.com
 ประเสริฐ    | prasert@example.com
 สุดา        | suda@example.com
 ธนกร        | thanakorn@example.com
 พิมพ์ใจ      | pimjai@example.com
 วราภรณ์     | waraporn@yahoo.com
(9 rows)
```

`(example|yahoo)` คือการระบุ "ทางเลือก" (alternation) ในวงเล็บ ส่วน `\.` คือการ escape จุด (`.`) เพื่อให้หมายถึงจุดจริงๆ ไม่ใช่ wildcard ของ regex และ `$` คือ anchor บอกว่า "ต้องจบด้วยรูปแบบนี้พอดี" วิธีนี้เขียนให้กระชับกว่าการใช้ `LIKE '%example.com' OR LIKE '%yahoo.com'` สองเงื่อนไขแยกกัน

</details>

---

**ข้อ 9.** เขียน query หาสินค้าที่**ไม่มี**หมวดหมู่ (`category_id IS NULL`) **หรือ** หยุดขายแล้ว (`is_active = FALSE`)

<details>
<summary>เฉลยข้อ 9</summary>

```sql
SELECT product_name, category_id, is_active
FROM products
WHERE category_id IS NULL OR is_active = FALSE;
```

```
 product_name | category_id | is_active
--------------+-------------+-----------
 Potato Chips |           5 | f
 Cookies      |             | t
(2 rows)
```

</details>

---

**ข้อ 10.** (ข้อรวม) เขียน query เดียวที่ตอบโจทย์ทั้งหมดนี้พร้อมกัน: หา "คำสั่งซื้อที่สำเร็จ (`completed`) ของลูกค้าที่มีบัญชีจริง (ไม่ใช่ guest) ที่มีมูลค่าอยู่ระหว่าง 50-200 บาท และไม่ได้จ่ายด้วยเงินสด" เรียงตามเงื่อนไขให้ครบและใช้วงเล็บให้เหมาะสม

<details>
<summary>เฉลยข้อ 10</summary>

```sql
SELECT order_id, customer_id, order_date, total_amount, payment_method
FROM orders
WHERE order_status = 'completed'
  AND customer_id IS NOT NULL
  AND total_amount BETWEEN 50 AND 200
  AND payment_method <> 'cash';
```

```
 order_id | customer_id |      order_date      | total_amount | payment_method
----------+-------------+-----------------------+---------------+----------------
        2 |           2 | 2024-06-01 09:30:00  |         70.00 | credit_card
        3 |           1 | 2024-06-02 08:00:00  |        195.00 | promptpay
        8 |           2 | 2024-06-05 11:00:00  |        150.00 | credit_card
(3 rows)
```

คำอธิบายทีละเงื่อนไข: `order_status = 'completed'` ตัดคำสั่งซื้อที่ยกเลิกหรือ pending ออก, `customer_id IS NOT NULL` ตัด guest order (order_id 13) ออก, `total_amount BETWEEN 50 AND 200` ปลอดภัยเพราะเป็นตัวเลขไม่ใช่ timestamp จึงไม่มีปัญหาแบบ Step 103, และ `payment_method <> 'cash'` ตัดรายการที่จ่ายเงินสดออก ในที่นี้ทุกเงื่อนไขเชื่อมด้วย `AND` ล้วน (ไม่มี `OR` ปน) จึงไม่จำเป็นต้องใส่วงเล็บเพิ่ม แต่การจัดเงื่อนไขคนละบรรทัดก็ช่วยให้อ่านง่ายไม่แพ้กัน

</details>

---

**บทถัดไป**: [Part 012: ORDER BY และ LIMIT/OFFSET](./part-012-order-limit.md)
