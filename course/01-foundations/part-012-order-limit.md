# Part 012: ORDER BY, LIMIT, OFFSET, FETCH — การเรียงลำดับและจำกัดผลลัพธ์

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับพื้นฐาน | Part 012

---

## เป้าหมายการเรียนรู้

หลังจากเรียนจบบทนี้ ผู้เรียนจะสามารถ:

- ใช้ `ORDER BY` เรียงลำดับผลลัพธ์แบบ ascending (`ASC`) และ descending (`DESC`) ได้ทั้งคอลัมน์เดียวและหลายคอลัมน์
- เรียงลำดับด้วย expression, alias หรือ column position ได้อย่างถูกต้องและเข้าใจข้อดี-ข้อเสียของแต่ละแบบ
- ควบคุมตำแหน่งของค่า `NULL` ในผลลัพธ์ด้วย `NULLS FIRST` / `NULLS LAST`
- ใช้ `LIMIT` เพื่อจำกัดจำนวนแถว และเขียน top-N query ได้อย่างมั่นใจ
- ใช้ `OFFSET` ร่วมกับ `LIMIT` เพื่อทำ pagination เบื้องต้น
- เข้าใจปัญหา performance ของ `OFFSET` เมื่อข้อมูลมีจำนวนมาก และรู้จักแนวคิด keyset pagination (cursor-based pagination) เป็นทางเลือก
- ใช้ syntax มาตรฐาน SQL อย่าง `FETCH FIRST ... ROWS ONLY` แทน `LIMIT` ได้
- ใช้ `FETCH FIRST ... WITH TIES` เพื่อจัดการกรณีมีค่าที่เท่ากันอยู่ที่ขอบเขตของผลลัพธ์
- ผสมผสาน `WHERE` + `ORDER BY` + `LIMIT` เพื่อแก้ปัญหาทางธุรกิจจริง เช่น หาสินค้าขายดี หาลูกค้าล่าสุด และทำระบบแบ่งหน้า (pagination) ของหน้าสินค้า

---

## เตรียมข้อมูล

บทนี้ยังคงใช้ฐานข้อมูล **ร้านกาแฟ (coffee shop)** ต่อเนื่องจากบทก่อนหน้า เพื่อให้ตัวอย่างทุกอันรันได้จริงแบบ self-contained เราจะสร้างตารางใหม่ทั้งหมด (DROP ก่อนถ้ามีอยู่แล้ว) แล้วใส่ข้อมูลตัวอย่างให้เพียงพอสำหรับสาธิตการเรียงลำดับและการจำกัดผลลัพธ์

```sql
-- ล้างตารางเดิม (ถ้ามี) เพื่อให้เริ่มต้นสะอาด
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
    product_id     SERIAL PRIMARY KEY,
    product_name   VARCHAR(100) NOT NULL,
    category_id    INTEGER REFERENCES categories(category_id),
    price          NUMERIC(10,2) NOT NULL,
    cost           NUMERIC(10,2),
    stock_quantity INTEGER NOT NULL DEFAULT 0,
    is_active      BOOLEAN NOT NULL DEFAULT TRUE,
    created_at     TIMESTAMP NOT NULL DEFAULT NOW()
);

-- ตารางลูกค้า
CREATE TABLE customers (
    customer_id   SERIAL PRIMARY KEY,
    first_name    VARCHAR(50) NOT NULL,
    last_name     VARCHAR(50) NOT NULL,
    email         VARCHAR(100) UNIQUE,
    phone         VARCHAR(20),
    member_since  DATE NOT NULL DEFAULT CURRENT_DATE,
    loyalty_points INTEGER NOT NULL DEFAULT 0
);

-- ตารางคำสั่งซื้อ
CREATE TABLE orders (
    order_id     SERIAL PRIMARY KEY,
    customer_id  INTEGER REFERENCES customers(customer_id),
    order_date   TIMESTAMP NOT NULL DEFAULT NOW(),
    status       VARCHAR(20) NOT NULL DEFAULT 'pending',
    total_amount NUMERIC(10,2) NOT NULL DEFAULT 0
);

-- ตารางรายการสินค้าในคำสั่งซื้อ
CREATE TABLE order_items (
    order_item_id SERIAL PRIMARY KEY,
    order_id      INTEGER REFERENCES orders(order_id),
    product_id    INTEGER REFERENCES products(product_id),
    quantity      INTEGER NOT NULL DEFAULT 1,
    unit_price    NUMERIC(10,2) NOT NULL
);
```

ต่อไปใส่ข้อมูลตัวอย่าง เริ่มจากหมวดหมู่สินค้า:

```sql
INSERT INTO categories (category_name, description) VALUES
    ('Coffee',   'เครื่องดื่มกาแฟทุกชนิด'),
    ('Tea',      'เครื่องดื่มชาทุกชนิด'),
    ('Bakery',   'ขนมอบและเบเกอรี'),
    ('Snack',    'ของว่างทานเล่น');
```

ใส่ข้อมูลสินค้า (มากกว่า 15 รายการ เพื่อให้เห็นผลลัพธ์การเรียงลำดับและ top-N ชัดเจน):

```sql
INSERT INTO products (product_name, category_id, price, cost, stock_quantity, is_active) VALUES
    ('Espresso',            1,  45.00, 15.00, 100, TRUE),
    ('Americano',           1,  50.00, 16.00, 120, TRUE),
    ('Latte',               1,  60.00, 20.00,  90, TRUE),
    ('Cappuccino',          1,  60.00, 20.00,  80, TRUE),
    ('Mocha',               1,  65.00, 22.00,  70, TRUE),
    ('Cold Brew',           1,  70.00, 25.00,  60, TRUE),
    ('Flat White',          1,  62.00, 21.00,  NULL, TRUE),
    ('Green Tea Latte',     2,  60.00, 20.00,  50, TRUE),
    ('Earl Grey',           2,  45.00, 14.00,  40, TRUE),
    ('Thai Milk Tea',       2,  55.00, 18.00,  65, TRUE),
    ('Jasmine Tea',         2,  40.00, 12.00,  30, TRUE),
    ('Croissant',           3,  45.00, 18.00,  25, TRUE),
    ('Chocolate Muffin',    3,  40.00, 15.00,  20, TRUE),
    ('Cheesecake',          3,  85.00, 35.00,  15, TRUE),
    ('Danish Pastry',       3,  50.00, 20.00,  18, TRUE),
    ('Potato Chips',        4,  25.00, 10.00, 200, TRUE),
    ('Mixed Nuts',          4,  55.00, 25.00,  70, TRUE),
    ('Old Recipe Cookie',   3,  35.00, 14.00,   0, FALSE);
```

ใส่ข้อมูลลูกค้า:

```sql
INSERT INTO customers (first_name, last_name, email, phone, member_since, loyalty_points) VALUES
    ('Somchai',  'Jaidee',    'somchai.j@example.com',  '0812345671', '2023-01-15', 320),
    ('Somsri',   'Rakthai',   'somsri.r@example.com',   '0812345672', '2023-02-20', 150),
    ('Anan',     'Suksawat',  'anan.s@example.com',     '0812345673', '2023-03-05', 480),
    ('Malee',    'Boonmee',   'malee.b@example.com',    '0812345674', '2023-04-11', 90),
    ('Prasert',  'Chaiyo',    'prasert.c@example.com',  '0812345675', '2023-05-02', 210),
    ('Nittaya',  'Wongsa',    'nittaya.w@example.com',  '0812345676', '2023-06-18', 60),
    ('Kittipong','Meechai',   'kittipong.m@example.com','0812345677', '2023-07-09', 340),
    ('Ratana',   'Srisuk',    'ratana.s@example.com',   '0812345678', '2023-08-25', 15),
    ('Wichai',   'Thongdee',  'wichai.t@example.com',   '0812345679', '2023-09-30', 275),
    ('Panida',   'Kaewkla',   'panida.k@example.com',   '0812345680', '2023-10-14', 5);
```

ใส่ข้อมูลคำสั่งซื้อ:

```sql
INSERT INTO orders (customer_id, order_date, status, total_amount) VALUES
    (1, '2024-01-05 08:30:00', 'completed', 165.00),
    (2, '2024-01-06 09:15:00', 'completed', 105.00),
    (3, '2024-01-06 10:00:00', 'completed', 245.00),
    (1, '2024-01-08 14:20:00', 'completed', 60.00),
    (4, '2024-01-10 11:45:00', 'cancelled', 0.00),
    (5, '2024-01-12 07:50:00', 'completed', 190.00),
    (2, '2024-01-15 16:10:00', 'completed', 85.00),
    (6, '2024-01-18 09:05:00', 'completed', 130.00),
    (3, '2024-01-20 13:30:00', 'completed', 310.00),
    (7, '2024-01-22 08:00:00', 'completed', 65.00),
    (8, '2024-01-25 15:45:00', 'pending',   120.00),
    (1, '2024-02-01 09:30:00', 'completed', 220.00),
    (9, '2024-02-03 10:20:00', 'completed', 175.00),
    (10,'2024-02-05 12:00:00', 'completed', 45.00),
    (5, '2024-02-08 08:10:00', 'completed', 90.00),
    (3, '2024-02-10 14:00:00', 'completed', 60.00);
```

ใส่ข้อมูลรายการสินค้าในแต่ละคำสั่งซื้อ:

```sql
INSERT INTO order_items (order_id, product_id, quantity, unit_price) VALUES
    (1, 3, 2, 60.00), (1, 12, 1, 45.00),
    (2, 2, 1, 50.00), (2, 13, 1, 40.00), (2, 16, 1, 25.00) - 10,
    (3, 14, 2, 85.00), (3, 5, 1, 65.00),
    (4, 4, 1, 60.00),
    (6, 6, 2, 70.00), (6, 9, 1, 45.00),
    (7, 10, 1, 55.00), (7, 15, 1, 50.00),
    (8, 1, 2, 45.00), (8, 17, 1, 55.00) - 15,
    (9, 14, 3, 85.00), (9, 3, 1, 60.00) - 5,
    (10, 8, 1, 60.00),
    (11, 2, 2, 50.00), (11, 12, 1, 45.00) - 25,
    (12, 6, 2, 70.00), (12, 5, 1, 65.00) - 10,
    (13, 7, 2, 62.00), (13, 9, 1, 45.00),
    (14, 11, 1, 40.00),
    (15, 2, 1, 50.00), (15, 10, 1, 55.00) - 15,
    (16, 3, 1, 60.00);
```

> **หมายเหตุ:** ในตัวอย่างข้างต้นมีการใช้ `- 10`, `- 15`, `- 5`, `- 25` ต่อท้ายค่าบางแถวเพื่อจำลองส่วนลด แต่ syntax แบบนี้ **ใช้ไม่ได้จริง** ใน `INSERT ... VALUES` เพราะแต่ละ tuple ต้องมีจำนวนค่าตรงกับจำนวนคอลัมน์เป๊ะ ๆ เท่านั้น หากต้องการใส่ส่วนลด ให้คำนวณค่าล่วงหน้าแล้วใส่เป็นตัวเลขตรง ๆ เช่น `(2, 16, 1, 15.00)` ด้านล่างนี้คือเวอร์ชันที่ถูกต้องและใช้รันได้จริงตลอดทั้งบท:

```sql
-- เวอร์ชันที่ถูกต้อง ใช้รันได้จริง (แทนที่บล็อกด้านบน)
TRUNCATE order_items RESTART IDENTITY;

INSERT INTO order_items (order_id, product_id, quantity, unit_price) VALUES
    (1, 3, 2, 60.00), (1, 12, 1, 45.00),
    (2, 2, 1, 50.00), (2, 13, 1, 40.00), (2, 16, 1, 15.00),
    (3, 14, 2, 85.00), (3, 5, 1, 65.00),
    (4, 4, 1, 60.00),
    (6, 6, 2, 70.00), (6, 9, 1, 45.00),
    (7, 10, 1, 55.00), (7, 15, 1, 50.00),
    (8, 1, 2, 45.00), (8, 17, 1, 40.00),
    (9, 14, 3, 85.00), (9, 3, 1, 55.00),
    (10, 8, 1, 60.00),
    (11, 2, 2, 50.00), (11, 12, 1, 20.00),
    (12, 6, 2, 70.00), (12, 5, 1, 55.00),
    (13, 7, 2, 62.00), (13, 9, 1, 45.00),
    (14, 11, 1, 40.00),
    (15, 2, 1, 50.00), (15, 10, 1, 40.00),
    (16, 3, 1, 60.00);
```

ตรวจสอบว่าข้อมูลถูกต้องด้วยการนับจำนวนแถวในแต่ละตาราง:

```sql
SELECT
    (SELECT COUNT(*) FROM categories)  AS categories_count,
    (SELECT COUNT(*) FROM products)    AS products_count,
    (SELECT COUNT(*) FROM customers)   AS customers_count,
    (SELECT COUNT(*) FROM orders)      AS orders_count,
    (SELECT COUNT(*) FROM order_items) AS order_items_count;
```

ผลลัพธ์ตัวอย่าง:

```
 categories_count | products_count | customers_count | orders_count | order_items_count
-------------------+----------------+------------------+--------------+--------------------
                 4 |             18 |               10 |           16 |                 26
```

ข้อมูลพร้อมแล้ว มาเริ่มเรียนรู้เรื่อง ORDER BY, LIMIT, OFFSET และ FETCH กันเลย

---

## Step 111: ORDER BY พื้นฐาน — ASC/DESC, เรียงตามหลายคอลัมน์

โดย default แล้ว PostgreSQL **ไม่รับประกัน** ลำดับของแถวที่ query คืนกลับมา ถ้าไม่ใส่ `ORDER BY` ลำดับอาจเปลี่ยนไปตาม physical storage, index ที่ query planner เลือกใช้ หรือแม้แต่การรัน query ซ้ำในเวลาต่างกัน ดังนั้นเมื่อไรก็ตามที่ลำดับของผลลัพธ์มีความสำคัญ **ต้องใส่ `ORDER BY` เสมอ**

Syntax พื้นฐาน:

```sql
SELECT column1, column2, ...
FROM table_name
ORDER BY column1 [ASC | DESC], column2 [ASC | DESC], ...;
```

- `ASC` (ascending) = เรียงจากน้อยไปมาก คือค่า default ถ้าไม่ระบุ
- `DESC` (descending) = เรียงจากมากไปน้อย

ตัวอย่างที่ 1: เรียงสินค้าตามราคาจากน้อยไปมาก

```sql
SELECT product_name, price
FROM products
ORDER BY price ASC;
```

ผลลัพธ์ (แสดงบางส่วน):

```
   product_name    | price
--------------------+-------
 Potato Chips       | 25.00
 Jasmine Tea        | 40.00
 Chocolate Muffin   | 40.00
 Old Recipe Cookie  | 35.00
 ...
```

ตัวอย่างที่ 2: เรียงสินค้าตามราคาจากมากไปน้อย (สินค้าราคาแพงสุดก่อน)

```sql
SELECT product_name, price
FROM products
ORDER BY price DESC;
```

ผลลัพธ์ (แสดงบางส่วน):

```
   product_name    | price
--------------------+-------
 Cheesecake         | 85.00
 Cold Brew          | 70.00
 Mocha              | 65.00
 Flat White         | 62.00
 ...
```

ตัวอย่างที่ 3: เรียงตามหลายคอลัมน์ — เรียงตามหมวดหมู่ก่อน แล้วภายในหมวดหมู่เดียวกันให้เรียงตามราคาจากแพงไปถูก

```sql
SELECT category_id, product_name, price
FROM products
ORDER BY category_id ASC, price DESC;
```

ผลลัพธ์ (แสดงบางส่วน):

```
 category_id |   product_name    | price
--------------+--------------------+-------
            1 | Cold Brew          | 70.00
            1 | Mocha              | 65.00
            1 | Flat White         | 62.00
            1 | Latte              | 60.00
            1 | Cappuccino         | 60.00
            1 | Americano          | 50.00
            1 | Espresso           | 45.00
            2 | Green Tea Latte    | 60.00
            2 | Thai Milk Tea      | 55.00
            2 | Earl Grey          | 45.00
            2 | Jasmine Tea        | 40.00
            ...
```

**ประเด็นสำคัญ:** เมื่อเรียงตามหลายคอลัมน์ PostgreSQL จะเรียงตามคอลัมน์แรกก่อน แล้วใช้คอลัมน์ถัดไปเป็นตัวตัดสิน "เมื่อค่าคอลัมน์แรกเท่ากัน" เท่านั้น (คล้ายการเรียงลำดับพจนานุกรม/lexicographic order) สังเกตว่า `Latte` และ `Cappuccino` มีราคาเท่ากัน (60.00) การเรียงระหว่างสองแถวนี้จึง**ไม่รับประกัน**ว่าใครจะมาก่อนใคร เพราะไม่มีคอลัมน์ที่สามมาช่วยตัดสิน — ถ้าต้องการผลลัพธ์ที่เสถียร (deterministic) ควรเพิ่มคอลัมน์ tie-breaker เช่น `product_id` เข้าไปด้วย เช่น `ORDER BY category_id, price DESC, product_id`

เราสามารถผสม `ASC` และ `DESC` ในแต่ละคอลัมน์ได้อย่างอิสระ:

```sql
SELECT customer_id, order_date, total_amount
FROM orders
ORDER BY customer_id ASC, order_date DESC;
```

query นี้จะจัดกลุ่มตามลูกค้า (customer_id น้อยไปมาก) และภายในลูกค้าคนเดียวกันจะแสดงคำสั่งซื้อล่าสุดก่อน — เป็น pattern ที่ใช้บ่อยมากในหน้ารายงาน "ประวัติการสั่งซื้อของลูกค้า"

---

## Step 112: ORDER BY ด้วย expression, alias, หรือ column position

`ORDER BY` ไม่จำเป็นต้องอ้างอิงเฉพาะชื่อคอลัมน์ตรง ๆ เท่านั้น เราสามารถใช้ได้ 3 รูปแบบ:

1. **Expression** — เรียงตามผลลัพธ์ของการคำนวณ
2. **Alias** — เรียงตามชื่อ alias ที่ตั้งไว้ใน `SELECT`
3. **Column position** — เรียงตามลำดับตำแหน่งของคอลัมน์ใน `SELECT` (1, 2, 3, ...)

### 1) เรียงด้วย expression

```sql
SELECT product_name, price, cost, (price - cost) AS profit
FROM products
WHERE cost IS NOT NULL
ORDER BY (price - cost) DESC;
```

ผลลัพธ์ (บางส่วน):

```
   product_name    | price | cost  | profit
--------------------+-------+-------+--------
 Cheesecake         | 85.00 | 35.00 |  50.00
 Cold Brew          | 70.00 | 25.00 |  45.00
 Mocha              | 65.00 | 22.00 |  43.00
 ...
```

เราสามารถเรียงตาม expression ที่ซับซ้อนกว่านั้นได้ เช่น เรียงตามความยาวของชื่อสินค้า:

```sql
SELECT product_name, LENGTH(product_name) AS name_length
FROM products
ORDER BY LENGTH(product_name) ASC;
```

### 2) เรียงด้วย alias

หาก `SELECT` มีการตั้งชื่อ alias ไว้ เราสามารถใช้ชื่อ alias นั้นใน `ORDER BY` ได้โดยตรง ทำให้ query อ่านง่ายขึ้นมาก:

```sql
SELECT product_name, price, cost, (price - cost) AS profit
FROM products
WHERE cost IS NOT NULL
ORDER BY profit DESC;
```

ผลลัพธ์จะเหมือนกับตัวอย่างก่อนหน้าทุกประการ เพราะ PostgreSQL รู้จัก alias `profit` และแทนที่ด้วย expression `(price - cost)` ให้อัตโนมัติ

> **ข้อควรรู้ (ลำดับการประมวลผลของ SQL):** ใน SQL แนวคิดของ logical query processing คือ `SELECT` (ซึ่งเป็นที่มาของ alias) จะถูกประมวลผล**หลัง** `WHERE` แต่**ก่อน** `ORDER BY` ดังนั้น alias ที่ตั้งใน `SELECT` จึงใช้ใน `ORDER BY` ได้ แต่ใช้ใน `WHERE` **ไม่ได้** (เพราะตอนที่ `WHERE` ทำงาน alias ยังไม่ถูกสร้างขึ้น) นี่คือเหตุผลที่ query แบบนี้จะ error:
>
> ```sql
> -- Error! ใช้ alias ใน WHERE ไม่ได้
> SELECT price - cost AS profit
> FROM products
> WHERE profit > 40;
> ```
>
> ข้อความ error ที่จะได้คือ `column "profit" does not exist`

### 3) เรียงด้วย column position

เราสามารถอ้างอิงคอลัมน์ใน `ORDER BY` ด้วยหมายเลขตำแหน่ง (นับจาก 1) ตามที่ปรากฏใน `SELECT` ได้:

```sql
SELECT product_name, price, category_id
FROM products
ORDER BY 3, 2 DESC;
```

query นี้เทียบเท่ากับ `ORDER BY category_id, price DESC` เพราะ `category_id` อยู่ตำแหน่งที่ 3 และ `price` อยู่ตำแหน่งที่ 2 ใน `SELECT`

**ข้อควรระวังของการใช้ column position:**

- **อ่านยาก** — คนอ่าน query ต้องนับตำแหน่งคอลัมน์เองว่าคือคอลัมน์ไหน ทำให้ maintain ยากขึ้น
- **เปราะบางต่อการเปลี่ยนแปลง** — ถ้ามีคนเพิ่ม/ลบ/สลับคอลัมน์ใน `SELECT` ในอนาคต ความหมายของ `ORDER BY 3` จะเปลี่ยนไปทันทีโดยไม่มี error เตือน (silent bug)
- เหมาะสำหรับใช้ใน query ชั่วคราวที่พิมพ์เร็ว ๆ บน `psql` เพื่อ debug เท่านั้น **ไม่แนะนำให้ใช้ในโค้ด production**

**คำแนะนำโดยรวม:** ใน production code ควรใช้ชื่อคอลัมน์หรือ alias ที่มีความหมายชัดเจนเสมอ หลีกเลี่ยง column position ยกเว้นกรณีที่เขียน query แบบ ad-hoc ชั่วคราว

---

## Step 113: การจัดการ NULL ใน ORDER BY — NULLS FIRST / NULLS LAST

ค่า `NULL` หมายถึง "ไม่มีค่า" หรือ "ไม่ทราบค่า" ซึ่งไม่สามารถเปรียบเทียบมากกว่าหรือน้อยกว่าตัวเลขปกติได้ตามหลักตรรกะ แต่เมื่อใช้ `ORDER BY` PostgreSQL ต้องมีกฎเกณฑ์ว่าจะวาง `NULL` ไว้ตรงไหนของผลลัพธ์

**พฤติกรรม default ของ PostgreSQL:**

- `ORDER BY column ASC` → `NULL` จะถูกจัดไว้**ท้ายสุด** (เหมือนเป็นค่ามากที่สุด) — ค่า default คือ `NULLS LAST`
- `ORDER BY column DESC` → `NULL` จะถูกจัดไว้**แรกสุด** — ค่า default คือ `NULLS FIRST`

พูดง่าย ๆ คือ default ของ PostgreSQL จะถือว่า `NULL` "ใหญ่กว่า" ค่าใด ๆ เสมอ

ลองดูตัวอย่าง สินค้าที่ `stock_quantity` เป็น `NULL` (คือ `Flat White` ที่เราใส่ไว้ตอนแรก):

```sql
SELECT product_name, stock_quantity
FROM products
ORDER BY stock_quantity ASC;
```

ผลลัพธ์:

```
   product_name    | stock_quantity
--------------------+-----------------
 Old Recipe Cookie  |               0
 Cheesecake         |              15
 Danish Pastry      |              18
 Chocolate Muffin   |              20
 Croissant          |              25
 Jasmine Tea        |              30
 ...
 Americano          |             120
 Potato Chips       |             200
 Flat White         |            NULL   <- NULL อยู่ท้ายสุด (default ของ ASC)
```

ถ้าต้องการให้ `NULL` มาก่อนแทน ใช้ `NULLS FIRST` อย่างชัดเจน:

```sql
SELECT product_name, stock_quantity
FROM products
ORDER BY stock_quantity ASC NULLS FIRST;
```

ผลลัพธ์:

```
   product_name    | stock_quantity
--------------------+-----------------
 Flat White         |            NULL   <- NULL อยู่แรกสุดตามที่สั่ง
 Old Recipe Cookie  |               0
 Cheesecake         |              15
 ...
```

ในทางกลับกัน ถ้าเรียง `DESC` แล้วต้องการให้ `NULL` ไปอยู่ท้ายสุด ก็ใช้ `NULLS LAST`:

```sql
SELECT product_name, stock_quantity
FROM products
ORDER BY stock_quantity DESC NULLS LAST;
```

ผลลัพธ์:

```
   product_name    | stock_quantity
--------------------+-----------------
 Potato Chips       |             200
 Americano          |             120
 ...
 Old Recipe Cookie  |               0
 Flat White         |            NULL   <- ถูกดันไปท้ายสุดแม้จะเรียง DESC
```

**กรณีใช้งานจริง:** สมมติหน้าเว็บร้านกาแฟต้องการแสดงสินค้าที่ "มีสต็อกมากที่สุดก่อน" เพื่อโปรโมทสินค้าที่มีของพร้อมขาย และไม่ต้องการให้สินค้าที่ยังไม่ระบุจำนวนสต็อก (`NULL`) ไปปนอยู่แถวบน ๆ ซึ่งจะทำให้ลูกค้าสับสน เราจึงเขียนได้ว่า:

```sql
SELECT product_name, stock_quantity
FROM products
WHERE is_active = TRUE
ORDER BY stock_quantity DESC NULLS LAST
LIMIT 5;
```

จะเห็นได้ว่า `NULLS FIRST`/`NULLS LAST` สามารถกำหนดแยกต่างหากในแต่ละคอลัมน์ของ `ORDER BY` ที่มีหลายคอลัมน์ได้ด้วย เช่น:

```sql
SELECT product_name, category_id, stock_quantity
FROM products
ORDER BY category_id ASC, stock_quantity DESC NULLS LAST;
```

---

## Step 114: LIMIT — จำกัดจำนวนแถวผลลัพธ์ พร้อมตัวอย่าง top-N query

`LIMIT` เป็น clause ของ PostgreSQL (ไม่ใช่ SQL standard แต่เป็นส่วนขยายที่ได้รับความนิยมมาก และมีอยู่ใน MySQL, SQLite ด้วย) ใช้เพื่อจำกัดจำนวนแถวสูงสุดที่ query จะคืนกลับมา

Syntax:

```sql
SELECT column1, column2, ...
FROM table_name
ORDER BY ...
LIMIT row_count;
```

**กฎเหล็กที่สำคัญที่สุด:** `LIMIT` ที่ใช้โดยไม่มี `ORDER BY` **ไม่มีความหมายที่แน่นอน** เพราะไม่มีการรับประกันว่าแถวไหนจะถูกเลือกมา 5 แถวแรก จึงควรใช้ `LIMIT` คู่กับ `ORDER BY` เสมอในทางปฏิบัติ

ตัวอย่างที่ 1: หาสินค้า 5 อันดับที่ราคาแพงที่สุด (top-N query)

```sql
SELECT product_name, price
FROM products
ORDER BY price DESC
LIMIT 5;
```

ผลลัพธ์:

```
  product_name   | price
------------------+-------
 Cheesecake       | 85.00
 Cold Brew        | 70.00
 Mocha            | 65.00
 Flat White       | 62.00
 Latte            | 60.00
```

ตัวอย่างที่ 2: หาลูกค้า 3 คนที่มีคะแนนสะสม (loyalty points) มากที่สุด

```sql
SELECT first_name, last_name, loyalty_points
FROM customers
ORDER BY loyalty_points DESC
LIMIT 3;
```

ผลลัพธ์:

```
 first_name | last_name | loyalty_points
------------+-----------+-----------------
 Anan       | Suksawat  |             480
 Kittipong  | Meechai   |             340
 Somchai    | Jaidee    |             320
```

ตัวอย่างที่ 3: `LIMIT 0` — ไม่คืนแถวใดเลย แต่ยังคง execute query จริง (มีประโยชน์ในการตรวจสอบ syntax หรือ structure ของผลลัพธ์โดยไม่ต้องดึงข้อมูลจริง)

```sql
SELECT * FROM products LIMIT 0;
```

ผลลัพธ์: ไม่มีแถวใด ๆ แต่ column header จะยังแสดงครบทุกคอลัมน์

ตัวอย่างที่ 4: การใช้ `LIMIT NULL` (เทียบเท่ากับไม่ใส่ `LIMIT` เลย — คืนทุกแถว) มักใช้ในโปรแกรมที่ generate query แบบ dynamic เพื่อให้ syntax สม่ำเสมอ:

```sql
SELECT product_name FROM products ORDER BY product_name LIMIT NULL;
```

**ข้อควรระวัง:** `LIMIT` ทำงาน**หลัง** `ORDER BY` เสมอ (ตามลำดับ logical query processing: `FROM` → `WHERE` → `GROUP BY` → `HAVING` → `SELECT` → `ORDER BY` → `LIMIT`/`OFFSET`) ดังนั้น `ORDER BY` จะถูกคำนวณกับ**ทุกแถว**ที่ผ่านเงื่อนไข `WHERE` ก่อน แล้วค่อยตัดเอาเฉพาะจำนวนแถวบนสุดตาม `LIMIT`

---

## Step 115: OFFSET — ข้ามแถว และการทำ pagination เบื้องต้นด้วย LIMIT + OFFSET

`OFFSET` ใช้เพื่อข้ามแถวจำนวนหนึ่งก่อนเริ่มคืนผลลัพธ์ ปกติใช้ร่วมกับ `LIMIT` เพื่อทำระบบแบ่งหน้า (pagination)

Syntax:

```sql
SELECT column1, column2, ...
FROM table_name
ORDER BY ...
LIMIT row_count OFFSET skip_count;
```

ตัวอย่างที่ 1: ข้าม 3 แถวแรก แล้วแสดง 5 แถวถัดไป (เรียงตามชื่อสินค้า)

```sql
SELECT product_name, price
FROM products
ORDER BY product_name ASC
LIMIT 5 OFFSET 3;
```

ผลลัพธ์: (สมมติเรียงตามตัวอักษร A-Z จะได้แถวที่ 4-8)

```
   product_name    | price
--------------------+-------
 Cold Brew          | 70.00
 Croissant          | 45.00
 Danish Pastry      | 50.00
 Earl Grey          | 45.00
 Espresso           | 45.00
```

### ทำ pagination ของหน้าสินค้า

สมมติเราต้องการแสดงสินค้า **หน้าละ 5 รายการ** เรียงตามชื่อ คำนวณค่า `OFFSET` ได้จากสูตร:

```
OFFSET = (page_number - 1) * page_size
```

**หน้าที่ 1** (แถวที่ 1-5):

```sql
SELECT product_id, product_name, price
FROM products
ORDER BY product_name ASC
LIMIT 5 OFFSET 0;   -- (1 - 1) * 5 = 0
```

**หน้าที่ 2** (แถวที่ 6-10):

```sql
SELECT product_id, product_name, price
FROM products
ORDER BY product_name ASC
LIMIT 5 OFFSET 5;   -- (2 - 1) * 5 = 5
```

**หน้าที่ 3** (แถวที่ 11-15):

```sql
SELECT product_id, product_name, price
FROM products
ORDER BY product_name ASC
LIMIT 5 OFFSET 10;  -- (3 - 1) * 5 = 10
```

ผลลัพธ์หน้าที่ 2 (ตัวอย่าง):

```
 product_id |   product_name    | price
------------+--------------------+-------
          8 | Green Tea Latte    | 60.00
         14 | Jasmine Tea        | 40.00
          3 | Latte              | 60.00
         17 | Mixed Nuts         | 55.00
          5 | Mocha              | 65.00
```

การคำนวณจำนวนหน้าทั้งหมด ต้องรู้จำนวนแถวทั้งหมดก่อน (มักคำนวณแยกด้วย `COUNT(*)`):

```sql
SELECT CEIL(COUNT(*)::NUMERIC / 5) AS total_pages
FROM products
WHERE is_active = TRUE;
```

ผลลัพธ์:

```
 total_pages
-------------
           4
```

> ในแอปพลิเคชันจริง มักเขียนเป็น query สองตัวคู่กันเสมอ: query แรกนับจำนวนทั้งหมด (`COUNT(*)`) สำหรับคำนวณเลขหน้า และ query ที่สองดึงข้อมูลของหน้านั้น ๆ ด้วย `LIMIT`/`OFFSET`

---

## Step 116: ปัญหาของ OFFSET กับข้อมูลจำนวนมาก และทางเลือก keyset pagination

แม้ `LIMIT` + `OFFSET` จะใช้งานง่ายและเข้าใจง่าย แต่มี**ปัญหาด้าน performance ร้ายแรง**เมื่อข้อมูลมีจำนวนมากและ `OFFSET` มีค่าสูง

### ปัญหาคืออะไร

เวลา PostgreSQL ประมวลผล `OFFSET N`, มันไม่ได้ "กระโดดข้าม" ไปที่แถวที่ N+1 ทันทีแบบมี index seek แต่มันต้อง**อ่านและนับ (scan) แถวทั้งหมดตั้งแต่ต้นจนถึงแถวที่ N ก่อน แล้วค่อยทิ้งแถวเหล่านั้น** จากนั้นจึงเริ่มคืนแถวตาม `LIMIT` ที่ระบุ

ลองพิจารณา:

```sql
-- หน้าที่ 1: เร็ว เพราะอ่านแค่ 20 แถวแรก
SELECT order_id, order_date, total_amount
FROM orders
ORDER BY order_date DESC
LIMIT 20 OFFSET 0;

-- หน้าที่ 5,000 (สมมติมีข้อมูลหลักล้านแถว): ช้ามาก!
SELECT order_id, order_date, total_amount
FROM orders
ORDER BY order_date DESC
LIMIT 20 OFFSET 99980;
```

query ที่สองต้อง sort และอ่านผ่านแถวเกือบ 100,000 แถวก่อนจะทิ้งไปแล้วคืนแค่ 20 แถวสุดท้าย ยิ่งเลขหน้ามากขึ้นเท่าไร query ก็ยิ่งช้าลงเรื่อย ๆ แบบ **linear (O(n))** เทียบกับตำแหน่งของ offset — นี่คือปัญหาคลาสสิกที่เรียกว่า **"deep pagination problem"**

ใช้ `EXPLAIN` เพื่อดูว่า planner ทำงานอย่างไร:

```sql
EXPLAIN SELECT order_id, order_date, total_amount
FROM orders
ORDER BY order_date DESC
LIMIT 20 OFFSET 10;
```

ผลลัพธ์ตัวอย่าง (concept — ตัวเลขจริงจะขึ้นกับข้อมูลและ index):

```
 Limit  (cost=1.32..1.67 rows=20 width=24)
   ->  Sort  (cost=1.27..1.31 rows=16 width=24)
         Sort Key: order_date DESC
         ->  Seq Scan on orders  (cost=0.00..1.16 rows=16 width=24)
```

จะสังเกตได้ว่า planner ยังคง sort ข้อมูลทั้งหมดก่อน (หรือ scan ผ่านทั้งหมดถ้าไม่มี index รองรับ) แล้วจึงข้าม (skip) ตาม `OFFSET` — ยิ่ง `OFFSET` มากเท่าไร cost ก็ยิ่งสูงขึ้นตามไปด้วย

### ทางเลือก: Keyset Pagination (Cursor-based Pagination)

แนวคิดของ **keyset pagination** คือแทนที่จะบอกว่า "ข้าม N แถว" เราจะจำ**ค่าของคอลัมน์ที่ใช้เรียงลำดับของแถวสุดท้ายที่แสดงไปแล้ว** (เรียกว่า cursor) แล้วใช้ `WHERE` เพื่อกรองเอาเฉพาะแถวที่ "มาหลังจาก" cursor นั้น แทนการ `OFFSET`

ตัวอย่าง: สมมติหน้าแรกเรียงตาม `order_id DESC` (คำสั่งซื้อล่าสุดก่อน) และแสดงหน้าละ 5 รายการ

**หน้าที่ 1:**

```sql
SELECT order_id, order_date, total_amount
FROM orders
ORDER BY order_id DESC
LIMIT 5;
```

ผลลัพธ์:

```
 order_id |     order_date      | total_amount
----------+----------------------+---------------
       16 | 2024-02-10 14:00:00  |         60.00
       15 | 2024-02-08 08:10:00  |         90.00
       14 | 2024-02-05 12:00:00  |         45.00
       13 | 2024-02-03 10:20:00  |        175.00
       12 | 2024-02-01 09:30:00  |        220.00
```

แถวสุดท้ายของหน้านี้มี `order_id = 12` เราจึงจำค่า `12` ไว้เป็น cursor สำหรับดึงหน้าถัดไป

**หน้าที่ 2** (แทนที่จะใช้ `OFFSET 5`, ใช้ `WHERE order_id < 12`):

```sql
SELECT order_id, order_date, total_amount
FROM orders
WHERE order_id < 12
ORDER BY order_id DESC
LIMIT 5;
```

ผลลัพธ์:

```
 order_id |     order_date      | total_amount
----------+----------------------+---------------
       11 | 2024-01-25 15:45:00  |        120.00
       10 | 2024-01-22 08:00:00  |         65.00
        9 | 2024-01-20 13:30:00  |        310.00
        8 | 2024-01-18 09:05:00  |        130.00
        7 | 2024-01-15 16:10:00  |         85.00
```

ถ้า `order_id` มี **index** (เช่น เป็น primary key ซึ่งมี index อยู่แล้วโดยอัตโนมัติ) query แบบ `WHERE order_id < 12 ORDER BY order_id DESC LIMIT 5` จะสามารถใช้ **index scan** ค้นหาตำแหน่งเริ่มต้นได้ทันทีโดยไม่ต้องอ่านแถวที่ถูกข้ามเลยแม้แต่แถวเดียว — เร็วในระดับ **O(log n)** แทนที่จะเป็น O(n) แบบ `OFFSET`

**ข้อดีของ keyset pagination:**
- Performance คงที่ (constant time) ไม่ว่าจะอยู่หน้าลึกแค่ไหน — ไม่ช้าลงเมื่อ "หน้า" เพิ่มขึ้น
- เหมาะกับ index range scan โดยตรง

**ข้อจำกัดของ keyset pagination:**
- ไม่สามารถกระโดดไปหน้าใด ๆ ได้โดยตรง (เช่น "ไปหน้า 50 เลย") ต้องไล่ทีละหน้าจากจุดที่รู้ cursor
- ต้องมีคอลัมน์ที่มีลำดับแน่นอนไม่ซ้ำกัน (unique, deterministic) เช่น primary key หรือ combination ของคอลัมน์ที่รับประกันความไม่ซ้ำ
- ถ้าเรียงตามคอลัมน์ที่ไม่ unique (เช่น `order_date` ที่อาจซ้ำกันได้) ต้องเพิ่มคอลัมน์ tie-breaker (เช่น `order_id`) เข้าไปในทั้ง `ORDER BY` และเงื่อนไข `WHERE` ด้วย เช่น `WHERE (order_date, order_id) < (cursor_date, cursor_id)`

**แนวทางปฏิบัติ:** สำหรับหน้าเว็บทั่วไปที่ต้องการปุ่ม "หน้า 1 2 3 ... 10" `LIMIT`/`OFFSET` ยังคงใช้ได้ดีถ้าจำนวนข้อมูลไม่มากนัก (หลักพัน-หมื่นแถว) แต่ถ้าเป็นระบบที่มีข้อมูลระดับล้านแถวขึ้นไป หรือเป็น API แบบ infinite scroll / "โหลดเพิ่ม" ควรพิจารณาใช้ keyset pagination แทน

---

## Step 117: FETCH FIRST ... ROWS ONLY (SQL standard syntax) เทียบกับ LIMIT

`LIMIT`/`OFFSET` เป็น extension เฉพาะของ PostgreSQL (และฐานข้อมูลบางตัว) แต่ **SQL standard** (ตั้งแต่ SQL:2008) กำหนด syntax ที่เป็นมาตรฐานเรียกว่า `FETCH FIRST ... ROWS ONLY` ซึ่ง PostgreSQL รองรับเช่นกัน (ตั้งแต่ PostgreSQL 8.4 เป็นต้นมา)

Syntax เต็ม:

```sql
SELECT column1, column2, ...
FROM table_name
ORDER BY ...
OFFSET skip_count { ROW | ROWS }
FETCH { FIRST | NEXT } row_count { ROW | ROWS } ONLY;
```

- `ROW`/`ROWS` และ `FIRST`/`NEXT` เป็นคำที่ใช้แทนกันได้ ไม่มีผลต่อความหมาย (มีไว้เพื่อให้ประโยคอ่านเป็นภาษาอังกฤษที่เป็นธรรมชาติ)
- ถ้าไม่ระบุ `row_count` เช่น `FETCH FIRST ROW ONLY` จะหมายถึง 1 แถว

ตัวอย่างที่ 1: เทียบเท่ากับ `LIMIT 5`

```sql
-- แบบ PostgreSQL extension
SELECT product_name, price
FROM products
ORDER BY price DESC
LIMIT 5;

-- แบบ SQL standard (เทียบเท่ากันทุกประการ)
SELECT product_name, price
FROM products
ORDER BY price DESC
FETCH FIRST 5 ROWS ONLY;
```

ทั้งสอง query ให้ผลลัพธ์เหมือนกันเป๊ะ:

```
  product_name   | price
------------------+-------
 Cheesecake       | 85.00
 Cold Brew        | 70.00
 Mocha            | 65.00
 Flat White       | 62.00
 Latte            | 60.00
```

ตัวอย่างที่ 2: เทียบเท่ากับ `LIMIT 5 OFFSET 10` (ทำ pagination หน้าที่ 3 ขนาดหน้าละ 5)

```sql
SELECT product_name, price
FROM products
ORDER BY product_name ASC
OFFSET 10 ROWS
FETCH NEXT 5 ROWS ONLY;
```

ผลลัพธ์เหมือนกับตัวอย่าง `LIMIT 5 OFFSET 10` ใน Step 115

ตัวอย่างที่ 3: `FETCH FIRST ROW ONLY` (1 แถว, ไม่ต้องระบุตัวเลข)

```sql
SELECT product_name, price
FROM products
ORDER BY price DESC
FETCH FIRST ROW ONLY;
```

ผลลัพธ์:

```
  product_name   | price
------------------+-------
 Cheesecake       | 85.00
```

### ควรใช้ syntax ไหน — LIMIT หรือ FETCH FIRST?

| ประเด็น | `LIMIT` / `OFFSET` | `FETCH FIRST ... ROWS ONLY` |
|---|---|---|
| มาตรฐาน SQL | ไม่ใช่ (PostgreSQL/MySQL extension) | ใช่ (SQL:2008 standard) |
| ความกระชับ | สั้นกว่า พิมพ์เร็วกว่า | ยาวกว่าเล็กน้อย |
| Portability ข้าม database | จำกัด (Oracle, SQL Server รุ่นเก่าไม่รองรับ) | รองรับกว้างขวางกว่า (Oracle 12c+, SQL Server 2012+, DB2 ก็รองรับ) |
| การใช้งานจริงในวงการ | นิยมมากในหมู่นักพัฒนา PostgreSQL/MySQL | นิยมในองค์กรที่ต้องการ SQL แบบ portable ข้ามหลาย database |
| Performance | เหมือนกันทุกประการ (PostgreSQL แปลงเป็น execution plan เดียวกัน) | เหมือนกันทุกประการ |

**สรุป:** ทั้งสองแบบทำงานเหมือนกันทุกประการในแง่ performance เพราะ PostgreSQL แปลง `LIMIT`/`OFFSET` และ `FETCH FIRST` ให้เป็น execution plan เดียวกันภายใน หากทีมของคุณทำงานเฉพาะบน PostgreSQL และคุ้นเคยกับ `LIMIT` อยู่แล้ว การใช้ `LIMIT` ต่อไปก็ไม่มีปัญหา แต่ถ้าต้องการเขียนโค้ดที่ portable ไปยัง database อื่น (เช่น Oracle, SQL Server) หรือทำงานในองค์กรที่มีมาตรฐานการเขียน SQL แบบ ANSI ควรพิจารณาใช้ `FETCH FIRST ... ROWS ONLY`

---

## Step 118: TIES — FETCH FIRST ... WITH TIES สำหรับกรณีค่าเท่ากัน

ปัญหาคลาสสิกของ top-N query คือ: ถ้าเราขอ "5 อันดับแรก" แต่มีหลายแถวที่ค่า**เท่ากันพอดี**ที่ขอบเขตของอันดับที่ 5 เราควรตัดแถวใดออกและเก็บแถวใดไว้?

ลองดูตัวอย่างปัญหานี้ สมมติเราต้องการหาสินค้า 3 อันดับแรกที่ราคาแพงที่สุด:

```sql
SELECT product_name, price
FROM products
ORDER BY price DESC
LIMIT 3;
```

ผลลัพธ์:

```
 product_name | price
---------------+-------
 Cheesecake    | 85.00
 Cold Brew     | 70.00
 Mocha         | 65.00
```

แต่ถ้าเราดูราคาทั้งหมด จะพบว่า `Flat White` ก็ราคา 62.00, ส่วน `Latte` และ `Cappuccino` ราคา 60.00 เท่ากันทั้งคู่ — คำถามคือถ้าเราต้องการ "3 อันดับแรก" แต่ตำแหน่งที่ 3 (คือ 60.00) มีสินค้าราคาเท่ากันหลายตัว เราควรจะตัดเอาแค่ตัวเดียว (ตามที่ `LIMIT` ทำ ซึ่งเลือกแบบไม่รับประกันว่าตัวไหน) หรือควรเก็บทุกตัวที่ค่าเท่ากันไว้?

`LIMIT` ธรรมดาจะเลือกแถวใดแถวหนึ่งแบบไม่แน่นอน (ขึ้นกับลำดับภายในที่ query planner ใช้) ถ้าค่าที่ใช้เรียงมีความเท่ากัน (tie) อยู่พอดีที่ขอบเขต

**`FETCH FIRST ... WITH TIES`** คือ syntax มาตรฐาน SQL ที่แก้ปัญหานี้โดยตรง: มันจะคืนแถวตามจำนวนที่ระบุ **บวกกับแถวอื่น ๆ ทั้งหมดที่มีค่าเรียงลำดับ (ORDER BY key) เท่ากับแถวสุดท้ายพอดี**

```sql
SELECT product_name, price
FROM products
ORDER BY price DESC
FETCH FIRST 3 ROWS WITH TIES;
```

สมมติราคาสินค้าเรียงจากมากไปน้อยคือ 85.00, 70.00, 65.00, 62.00, 60.00, 60.00, ... — ในกรณีนี้อันดับที่ 3 คือ 65.00 ซึ่งไม่มีใครเสมอด้วย ผลลัพธ์จึงเหมือน `LIMIT 3` ปกติ:

```
 product_name | price
---------------+-------
 Cheesecake    | 85.00
 Cold Brew     | 70.00
 Mocha         | 65.00
```

แต่ถ้าเราขอ 5 อันดับแรกแทน อันดับที่ 5 คือ 60.00 ซึ่งมี `Latte` และ `Cappuccino` เท่ากันพอดี:

```sql
SELECT product_name, price
FROM products
ORDER BY price DESC
FETCH FIRST 5 ROWS WITH TIES;
```

ผลลัพธ์:

```
 product_name | price
---------------+-------
 Cheesecake    | 85.00
 Cold Brew     | 70.00
 Mocha         | 65.00
 Flat White    | 62.00
 Latte         | 60.00
 Cappuccino    | 60.00
```

สังเกตว่าผลลัพธ์มี**6 แถว** ไม่ใช่ 5 แถว เพราะ `Latte` และ `Cappuccino` มีราคาเท่ากันที่ตำแหน่งขอบเขต (60.00) ทั้งคู่จึงถูกรวมเข้ามาด้วย นี่คือความหมายของ "WITH TIES" — คืนแถวที่ "เสมอกัน" (tie) กับแถวสุดท้ายด้วย

**ข้อกำหนดสำคัญ:** `WITH TIES` **ต้องใช้คู่กับ `ORDER BY` เสมอ** เพราะ PostgreSQL ต้องรู้ว่า "ค่าไหนคือค่าที่ใช้ตัดสินความเท่ากัน" ถ้าไม่มี `ORDER BY` จะเกิด error:

```sql
-- Error!
SELECT product_name, price
FROM products
FETCH FIRST 5 ROWS WITH TIES;
```

ข้อความ error: `WITH TIES cannot be specified without ORDER BY`

**ข้อควรรู้:** `LIMIT`/`OFFSET` แบบดั้งเดิมของ PostgreSQL **ไม่มี** option เทียบเท่า `WITH TIES` ต้องใช้ syntax `FETCH FIRST ... WITH TIES` เท่านั้น จึงเป็นอีกเหตุผลหนึ่งที่ควรรู้จัก syntax แบบ SQL standard นี้ไว้

**กรณีใช้งานจริง:** ระบบจัดอันดับ (leaderboard) ที่ต้องการแสดง "ผู้ชนะ 3 อันดับแรก" แต่ถ้ามีผู้เล่นคะแนนเท่ากันที่อันดับ 3 ก็ควรแสดงทุกคนที่ได้อันดับ 3 ไม่ใช่เลือกมาแค่คนเดียวแบบสุ่ม — `FETCH FIRST 3 ROWS WITH TIES` แก้ปัญหานี้ได้ตรงจุด

---

## Step 119: การรวม WHERE + ORDER BY + LIMIT ในสถานการณ์จริง

ในงานจริง เราแทบไม่เคยใช้ `ORDER BY`/`LIMIT` แบบโดดเดี่ยว แต่มักผสมกับ `WHERE`, `JOIN`, และบางครั้ง `GROUP BY` เพื่อตอบคำถามทางธุรกิจที่ซับซ้อนขึ้น

### กรณีที่ 1: หาสินค้าขายดี 5 อันดับแรก (Top-5 Best Sellers)

ต้อง `JOIN` ตาราง `order_items` กับ `products` แล้ว `GROUP BY` เพื่อรวมยอดขาย จากนั้นเรียงตามยอดขายรวมและจำกัด 5 อันดับ:

```sql
SELECT
    p.product_name,
    SUM(oi.quantity) AS total_quantity_sold,
    SUM(oi.quantity * oi.unit_price) AS total_revenue
FROM order_items oi
JOIN products p ON p.product_id = oi.product_id
JOIN orders o ON o.order_id = oi.order_id
WHERE o.status = 'completed'
GROUP BY p.product_id, p.product_name
ORDER BY total_quantity_sold DESC
LIMIT 5;
```

ผลลัพธ์ตัวอย่าง:

```
   product_name    | total_quantity_sold | total_revenue
--------------------+-----------------------+----------------
 Cheesecake         |                     5 |         425.00
 Latte              |                     4 |         240.00
 Espresso           |                     3 |          90.00
 Americano          |                     3 |         150.00
 Cold Brew          |                     2 |         140.00
```

### กรณีที่ 2: หาลูกค้าล่าสุดที่สมัครสมาชิก (Recent Customers)

ใช้กรณีในหน้า "ลูกค้าใหม่ล่าสุด" ของระบบหลังบ้าน:

```sql
SELECT
    customer_id,
    first_name || ' ' || last_name AS full_name,
    email,
    member_since
FROM customers
ORDER BY member_since DESC
LIMIT 5;
```

ผลลัพธ์:

```
 customer_id |    full_name     |           email           | member_since
-------------+-------------------+----------------------------+---------------
          10 | Panida Kaewkla    | panida.k@example.com      | 2023-10-14
           9 | Wichai Thongdee   | wichai.t@example.com      | 2023-09-30
           8 | Ratana Srisuk     | ratana.s@example.com      | 2023-08-25
           7 | Kittipong Meechai | kittipong.m@example.com   | 2023-07-09
           6 | Nittaya Wongsa    | nittaya.w@example.com     | 2023-06-18
```

### กรณีที่ 3: หาคำสั่งซื้อล่าสุดของลูกค้าคนหนึ่งโดยเฉพาะ (WHERE + ORDER BY + LIMIT ร่วมกัน)

```sql
SELECT order_id, order_date, status, total_amount
FROM orders
WHERE customer_id = 1
  AND status = 'completed'
ORDER BY order_date DESC
LIMIT 3;
```

ผลลัพธ์ (ลูกค้า Somchai Jaidee มี 3 คำสั่งซื้อที่ completed):

```
 order_id |     order_date      |  status   | total_amount
----------+----------------------+-----------+---------------
       12 | 2024-02-01 09:30:00  | completed |        220.00
        4 | 2024-01-08 14:20:00  | completed |         60.00
        1 | 2024-01-05 08:30:00  | completed |        165.00
```

### กรณีที่ 4: หาสินค้าที่ต้อง restock ด่วน (สต็อกน้อยที่สุด แต่ยัง active อยู่)

```sql
SELECT product_name, stock_quantity
FROM products
WHERE is_active = TRUE
ORDER BY stock_quantity ASC NULLS LAST
LIMIT 5;
```

ผลลัพธ์:

```
   product_name    | stock_quantity
--------------------+-----------------
 Cheesecake         |              15
 Danish Pastry      |              18
 Chocolate Muffin   |              20
 Croissant          |              25
 Jasmine Tea        |              30
```

สังเกตว่าเราใช้ `WHERE is_active = TRUE` เพื่อไม่นับสินค้าที่เลิกขายแล้ว (`Old Recipe Cookie` ที่ `is_active = FALSE` และ `stock_quantity = 0`) และใช้ `NULLS LAST` เพื่อไม่ให้สินค้าที่ยังไม่ได้ระบุสต็อก (`Flat White`) ไปปรากฏในลิสต์ "ต้อง restock ด่วน" อย่างผิดพลาด

**บทเรียนสำคัญจากทั้ง 4 กรณี:** ลำดับการเขียน clause ใน query (`SELECT` → `FROM` → `JOIN` → `WHERE` → `GROUP BY` → `ORDER BY` → `LIMIT`) ไม่ได้เป็นแค่ syntax ที่ต้องท่องจำ แต่สะท้อนลำดับการคิดเชิงตรรกะ: กรองข้อมูลที่ไม่เกี่ยวข้องออกก่อน (`WHERE`), รวมกลุ่มถ้าจำเป็น (`GROUP BY`), เรียงลำดับตามเกณฑ์ที่ต้องการ (`ORDER BY`), แล้วค่อยตัดเอาเฉพาะจำนวนที่ต้องการ (`LIMIT`)

---

## Step 120: แบบฝึกหัดรวม — สร้าง query สำหรับหน้า "สินค้าขายดี", "ลูกค้าล่าสุด", "การแบ่งหน้าสินค้า"

มาถึง step สุดท้าย เราจะรวบยอดทุกอย่างที่เรียนมาสร้าง query สำหรับ 3 หน้าจอที่พบบ่อยที่สุดในระบบร้านค้าออนไลน์จริง

### หน้าจอที่ 1: "สินค้าขายดี" (Best Sellers Widget) — แสดง 5 อันดับพร้อมสัดส่วนกำไร

```sql
SELECT
    p.product_id,
    p.product_name,
    c.category_name,
    SUM(oi.quantity)                       AS units_sold,
    SUM(oi.quantity * oi.unit_price)       AS revenue,
    SUM(oi.quantity * (oi.unit_price - p.cost)) AS estimated_profit
FROM order_items oi
JOIN orders o     ON o.order_id = oi.order_id
JOIN products p   ON p.product_id = oi.product_id
JOIN categories c ON c.category_id = p.category_id
WHERE o.status = 'completed'
GROUP BY p.product_id, p.product_name, c.category_name
ORDER BY units_sold DESC, revenue DESC
FETCH FIRST 5 ROWS ONLY;
```

ผลลัพธ์ตัวอย่าง:

```
 product_id |   product_name    | category_name | units_sold | revenue | estimated_profit
------------+--------------------+----------------+-------------+---------+--------------------
         14 | Cheesecake         | Bakery         |           5 |  425.00 |            175.00
          3 | Latte              | Coffee         |           4 |  240.00 |            160.00
          1 | Espresso           | Coffee         |           3 |   90.00 |             45.00
          2 | Americano          | Coffee         |           3 |  150.00 |            102.00
          6 | Cold Brew          | Coffee         |           2 |  140.00 |             90.00
```

### หน้าจอที่ 2: "ลูกค้าล่าสุด" (Recent Customers) — พร้อมยอดใช้จ่ายรวม เรียงตามวันที่สมัครล่าสุด

```sql
SELECT
    cu.customer_id,
    cu.first_name || ' ' || cu.last_name AS full_name,
    cu.member_since,
    COALESCE(SUM(o.total_amount) FILTER (WHERE o.status = 'completed'), 0) AS lifetime_spend
FROM customers cu
LEFT JOIN orders o ON o.customer_id = cu.customer_id
GROUP BY cu.customer_id, cu.first_name, cu.last_name, cu.member_since
ORDER BY cu.member_since DESC
LIMIT 5;
```

ผลลัพธ์ตัวอย่าง:

```
 customer_id |    full_name     | member_since | lifetime_spend
-------------+-------------------+---------------+------------------
          10 | Panida Kaewkla    | 2023-10-14   |           45.00
           9 | Wichai Thongdee   | 2023-09-30   |          175.00
           8 | Ratana Srisuk     | 2023-08-25   |            0.00
           7 | Kittipong Meechai | 2023-07-09   |           65.00
           6 | Nittaya Wongsa    | 2023-06-18   |          130.00
```

> หมายเหตุ: `FILTER (WHERE ...)` เป็น syntax ของ aggregate filter ที่จะกล่าวถึงรายละเอียดใน part ถัดไปเรื่อง aggregate functions — ในที่นี้ใช้เพื่อรวมยอดเฉพาะคำสั่งซื้อที่ `completed` เท่านั้น

### หน้าจอที่ 3: "การแบ่งหน้าสินค้า" (Product Listing with Pagination) — ทั้งแบบ OFFSET และแบบ Keyset

**แบบ OFFSET (เหมาะกับหน้าจอที่มีปุ่มเลขหน้า 1 2 3 ...):**

ฟังก์ชันสมมติ `get_products_page(page_number, page_size)`:

```sql
-- หน้าที่ 2 ขนาดหน้าละ 4 รายการ เรียงตามชื่อ
SELECT product_id, product_name, price, stock_quantity
FROM products
WHERE is_active = TRUE
ORDER BY product_name ASC
LIMIT 4 OFFSET 4;   -- (page 2 - 1) * page_size 4 = 4
```

ผลลัพธ์:

```
 product_id |   product_name    | price | stock_quantity
------------+--------------------+-------+-----------------
          6 | Cold Brew          | 70.00 |              60
         12 | Croissant          | 45.00 |              25
         15 | Danish Pastry      | 50.00 |              18
          9 | Earl Grey          | 45.00 |              40
```

**แบบ Keyset (เหมาะกับ infinite scroll / "โหลดเพิ่มเติม"):**

```sql
-- หน้าแรก
SELECT product_id, product_name, price
FROM products
WHERE is_active = TRUE
ORDER BY product_id ASC
LIMIT 4;
```

```sql
-- โหลดเพิ่ม: สมมติแถวสุดท้ายที่แสดงคือ product_id = 4
SELECT product_id, product_name, price
FROM products
WHERE is_active = TRUE
  AND product_id > 4          -- cursor จากหน้าก่อนหน้า
ORDER BY product_id ASC
LIMIT 4;
```

ผลลัพธ์ของการ "โหลดเพิ่ม":

```
 product_id |  product_name  | price
------------+-----------------+-------
          5 | Mocha           | 65.00
          6 | Cold Brew       | 70.00
          7 | Flat White      | 62.00
          8 | Green Tea Latte | 60.00
```

---

## สรุปท้ายบท

ในบทนี้เราได้เรียนรู้เครื่องมือสำคัญที่สุดสำหรับควบคุม "ลำดับ" และ "ปริมาณ" ของผลลัพธ์จาก query:

- **`ORDER BY`** ควบคุมลำดับผลลัพธ์ ใช้ `ASC`/`DESC` ได้ทั้งคอลัมน์เดียวและหลายคอลัมน์ โดยคอลัมน์แรกมีความสำคัญสูงสุดและคอลัมน์ถัดไปใช้ตัดสินเมื่อคอลัมน์ก่อนหน้าเท่ากัน
- `ORDER BY` รองรับทั้ง **expression, alias, และ column position** แต่ในทาง production ควรใช้ชื่อคอลัมน์หรือ alias ที่ชัดเจน หลีกเลี่ยง column position
- ค่า **`NULL`** ใน `ORDER BY` default จะไปอยู่ท้ายสุดเมื่อ `ASC` และอยู่แรกสุดเมื่อ `DESC` — ใช้ `NULLS FIRST`/`NULLS LAST` เพื่อควบคุมตำแหน่งได้ตามต้องการ
- **`LIMIT`** จำกัดจำนวนแถวสูงสุดที่คืนกลับ ต้องใช้คู่กับ `ORDER BY` เสมอเพื่อให้ผลลัพธ์แน่นอน (deterministic)
- **`OFFSET`** ข้ามแถวจำนวนหนึ่ง มักใช้คู่กับ `LIMIT` เพื่อทำ pagination แบบดั้งเดิม แต่มีปัญหา performance เมื่อ offset มีค่าสูงกับข้อมูลจำนวนมาก (deep pagination problem)
- **Keyset pagination** (cursor-based) คือทางเลือกที่มี performance คงที่ (constant time) ไม่ว่าจะ "ลึก" แค่ไหน โดยใช้ `WHERE` กรองจาก cursor แทนการ `OFFSET`
- **`FETCH FIRST ... ROWS ONLY`** คือ syntax มาตรฐาน SQL ที่เทียบเท่ากับ `LIMIT` มี performance เหมือนกันทุกประการ แต่ portable ข้าม database ได้ดีกว่า
- **`FETCH FIRST ... WITH TIES`** แก้ปัญหากรณีมีค่าเท่ากันที่ขอบเขตของผลลัพธ์ โดยคืนแถวทั้งหมดที่ "เสมอ" กับแถวสุดท้ายด้วย — เป็น feature ที่ `LIMIT` ธรรมดาไม่มี
- การผสม **`WHERE` + `ORDER BY` + `LIMIT`** (และบางครั้ง `JOIN`/`GROUP BY`) คือ pattern ที่ใช้บ่อยที่สุดในงานจริง ไม่ว่าจะเป็นหน้าสินค้าขายดี ลูกค้าล่าสุด หรือระบบแบ่งหน้า

ทักษะเหล่านี้เป็นพื้นฐานสำคัญที่จะถูกนำไปใช้ซ้ำ ๆ ตลอดทั้งหลักสูตร ทั้งในบทที่เกี่ยวกับ aggregate functions, window functions และการ optimize query ขนาดใหญ่ในระดับ world-class ต่อไป

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1
เขียน query แสดงสินค้าทั้งหมด เรียงตามราคาจากมากไปน้อย

<details>
<summary>เฉลย</summary>

```sql
SELECT product_name, price
FROM products
ORDER BY price DESC;
```

</details>

---

### แบบฝึกหัดที่ 2
เขียน query แสดงคำสั่งซื้อ (orders) เรียงตาม `customer_id` จากน้อยไปมาก และภายในลูกค้าคนเดียวกันให้เรียงตาม `order_date` จากล่าสุดไปเก่าสุด

<details>
<summary>เฉลย</summary>

```sql
SELECT customer_id, order_id, order_date, total_amount
FROM orders
ORDER BY customer_id ASC, order_date DESC;
```

</details>

---

### แบบฝึกหัดที่ 3
เขียน query แสดง `product_name`, `price`, `cost` และคอลัมน์กำไร (`price - cost`) โดยตั้งชื่อ alias ว่า `margin` แล้วเรียงตาม `margin` จากมากไปน้อย โดยใช้ alias ใน `ORDER BY` (ไม่ใช้ expression ซ้ำ)

<details>
<summary>เฉลย</summary>

```sql
SELECT product_name, price, cost, (price - cost) AS margin
FROM products
WHERE cost IS NOT NULL
ORDER BY margin DESC;
```

</details>

---

### แบบฝึกหัดที่ 4
ตาราง `products` มีสินค้าที่ `stock_quantity` เป็น `NULL` อยู่ 1 รายการ เขียน query เรียงสินค้าตาม `stock_quantity` จากน้อยไปมาก โดยให้ค่า `NULL` แสดง**อยู่แรกสุด**

<details>
<summary>เฉลย</summary>

```sql
SELECT product_name, stock_quantity
FROM products
ORDER BY stock_quantity ASC NULLS FIRST;
```

</details>

---

### แบบฝึกหัดที่ 5
เขียน query หาสินค้า 3 อันดับที่มีสต็อกเหลือน้อยที่สุด (เฉพาะสินค้าที่ `is_active = TRUE` และไม่นับสินค้าที่ `stock_quantity` เป็น `NULL`)

<details>
<summary>เฉลย</summary>

```sql
SELECT product_name, stock_quantity
FROM products
WHERE is_active = TRUE
  AND stock_quantity IS NOT NULL
ORDER BY stock_quantity ASC
LIMIT 3;
```

</details>

---

### แบบฝึกหัดที่ 6
สมมติหน้าเว็บแสดงลูกค้า หน้าละ 4 คน เรียงตาม `customer_id` จากน้อยไปมาก จงเขียน query สำหรับ **หน้าที่ 3**

<details>
<summary>เฉลย</summary>

```sql
-- page 3, page_size 4 -> OFFSET = (3-1) * 4 = 8
SELECT customer_id, first_name, last_name
FROM customers
ORDER BY customer_id ASC
LIMIT 4 OFFSET 8;
```

ผลลัพธ์ที่ได้ควรเป็นลูกค้าที่มี `customer_id` เท่ากับ 9 และ 10 (เพราะมีลูกค้าทั้งหมดเพียง 10 คน หน้าที่ 3 จึงมีแค่ 2 แถวแทนที่จะเป็น 4 แถวเต็ม)

</details>

---

### แบบฝึกหัดที่ 7
เขียน query เดียวกับแบบฝึกหัดที่ 6 แต่ใช้ syntax แบบ SQL standard (`FETCH FIRST ... ROWS ONLY`) แทน `LIMIT`/`OFFSET`

<details>
<summary>เฉลย</summary>

```sql
SELECT customer_id, first_name, last_name
FROM customers
ORDER BY customer_id ASC
OFFSET 8 ROWS
FETCH NEXT 4 ROWS ONLY;
```

</details>

---

### แบบฝึกหัดที่ 8
ตาราง `products` มีสินค้าราคา 45.00 อยู่หลายรายการ (Espresso, Croissant, Earl Grey) เขียน query หาสินค้า 5 อันดับที่ราคาถูกที่สุด โดยถ้ามีสินค้าราคาเท่ากันพอดีที่ขอบเขตอันดับที่ 5 ให้แสดงทุกรายการที่ราคาเท่ากันนั้นด้วย (ไม่ตัดทิ้งแบบสุ่ม)

<details>
<summary>เฉลย</summary>

```sql
SELECT product_name, price
FROM products
ORDER BY price ASC
FETCH FIRST 5 ROWS WITH TIES;
```

เนื่องจากมีสินค้าหลายรายการราคาเท่ากันที่ 45.00 ซึ่งอาจอยู่ที่ขอบเขตอันดับ 5 พอดี `WITH TIES` จะรวมทุกรายการที่ราคาเท่ากับแถวสุดท้ายเข้ามาด้วย ทำให้จำนวนแถวผลลัพธ์อาจมากกว่า 5 แถว

</details>

---

### แบบฝึกหัดที่ 9
เขียน query หาลูกค้า 3 คนที่มียอดใช้จ่ายรวม (เฉพาะคำสั่งซื้อที่ `status = 'completed'`) สูงที่สุด แสดงชื่อเต็มและยอดรวม

<details>
<summary>เฉลย</summary>

```sql
SELECT
    cu.first_name || ' ' || cu.last_name AS full_name,
    SUM(o.total_amount) AS total_spend
FROM customers cu
JOIN orders o ON o.customer_id = cu.customer_id
WHERE o.status = 'completed'
GROUP BY cu.customer_id, cu.first_name, cu.last_name
ORDER BY total_spend DESC
LIMIT 3;
```

</details>

---

### แบบฝึกหัดที่ 10
อธิบายด้วยคำพูดของตัวเอง (เขียนเป็น comment ใน SQL หรือข้อความ) ว่าทำไม query ที่ใช้ `OFFSET 500000` บนตารางที่มีข้อมูล 1 ล้านแถวถึงช้ากว่า query ที่ใช้ `OFFSET 10` มาก และจงเขียน query แบบ keyset pagination ที่แก้ปัญหานี้ สำหรับการดึงคำสั่งซื้อ (`orders`) หน้าถัดไป โดยสมมติว่าหน้าก่อนหน้าแสดงจบที่ `order_id = 9` (เรียงจากน้อยไปมาก) และต้องการหน้าละ 5 แถว

<details>
<summary>เฉลย</summary>

**คำอธิบาย:** `OFFSET` ไม่สามารถ "กระโดด" ไปยังตำแหน่งที่ต้องการได้โดยตรง มันต้องอ่านและนับผ่านแถวทั้งหมดตั้งแต่ต้นจนถึงตำแหน่งที่ระบุก่อน แล้วจึงทิ้งแถวเหล่านั้นไปและเริ่มคืนผลลัพธ์ ดังนั้น `OFFSET 500000` ต้องอ่านผ่านแถวถึง 500,000 แถวก่อนจะเริ่มคืนผลลัพธ์จริง ในขณะที่ `OFFSET 10` อ่านผ่านแค่ 10 แถวเท่านั้น ยิ่งค่า offset สูง เวลาที่ query ใช้ก็ยิ่งเพิ่มขึ้นแบบเป็นสัดส่วน (linear/O(n))

**Query แบบ keyset pagination:**

```sql
SELECT order_id, order_date, status, total_amount
FROM orders
WHERE order_id > 9          -- cursor จากหน้าก่อนหน้า (แถวสุดท้ายที่แสดงไปแล้ว)
ORDER BY order_id ASC
LIMIT 5;
```

query นี้ใช้ `WHERE order_id > 9` แทนการ `OFFSET` หากคอลัมน์ `order_id` มี index (ซึ่งมีอยู่แล้วเพราะเป็น primary key) PostgreSQL สามารถใช้ index scan เพื่อกระโดดไปยังตำแหน่งที่ `order_id > 9` ได้ทันทีโดยไม่ต้องอ่านแถวที่ถูกข้ามแม้แต่แถวเดียว ทำให้ performance คงที่ (constant time) ไม่ว่าจะอยู่ "หน้าลึก" แค่ไหนก็ตาม

</details>

---

## บทถัดไป

เรียนรู้ต่อได้ที่ [Part 013: UPDATE — การแก้ไขข้อมูล](./part-013-update.md)
