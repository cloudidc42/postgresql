# Part 013: UPDATE — แก้ไขข้อมูลแบบมีเงื่อนไขและหลายคอลัมน์

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับพื้นฐาน | Part 013

---

## เป้าหมายการเรียนรู้

เมื่อจบบทนี้ ผู้เรียนจะสามารถ:

- เขียนคำสั่ง `UPDATE` เพื่อแก้ไขข้อมูลในตารางได้อย่างถูกต้องและปลอดภัย
- อัปเดตหลายคอลัมน์พร้อมกันในคำสั่งเดียว
- เข้าใจอันตรายของการลืมเขียน `WHERE` clause และรู้วิธีป้องกันความเสียหายก่อนที่มันจะเกิดขึ้นจริง
- อัปเดตค่าคอลัมน์โดยอ้างอิงจากค่าคอลัมน์เดิม เช่น การคำนวณส่วนลดหรือส่วนเพิ่มราคา
- ใช้ `UPDATE ... FROM` เพื่ออัปเดตข้อมูลโดยอ้างอิงจากตารางอื่น
- ใช้ `RETURNING` เพื่อดูผลลัพธ์ของการอัปเดตทันทีโดยไม่ต้อง `SELECT` ซ้ำ
- ใช้ subquery ทั้งใน `SET` และ `WHERE` ของคำสั่ง `UPDATE`
- ใช้ `CASE WHEN` ภายใน `UPDATE` เพื่อสร้างเงื่อนไขที่ซับซ้อนได้ในคำสั่งเดียว
- เข้าใจผลกระทบของการอัปเดตข้อมูลจำนวนมาก (bulk update) ต่อประสิทธิภาพระบบ และวิธีแบ่ง batch เพื่อลดผลกระทบ
- ประยุกต์ความรู้ทั้งหมดกับสถานการณ์จริงของระบบร้านกาแฟ เช่น ปรับราคาสินค้า เปลี่ยนสถานะออเดอร์ และอัปเดตสต็อก

---

## เตรียมข้อมูล

บทนี้ยังคงใช้สคีมา "ร้านกาแฟ" (coffee shop) ต่อเนื่องจากบทก่อนหน้า เพื่อให้ตัวอย่างทั้งหมดรันได้ครบในตัวเอง เราจะสร้างตารางใหม่ (`DROP ... IF EXISTS` ก่อนเพื่อความสะอาด) แล้วใส่ข้อมูลตัวอย่างให้พร้อมสำหรับฝึก `UPDATE`

```sql
-- ล้างตารางเดิม (ถ้ามี) เพื่อให้ตัวอย่างรันซ้ำได้
DROP TABLE IF EXISTS order_items CASCADE;
DROP TABLE IF EXISTS orders CASCADE;
DROP TABLE IF EXISTS products CASCADE;
DROP TABLE IF EXISTS customers CASCADE;
DROP TABLE IF EXISTS categories CASCADE;

-- ตารางหมวดหมู่สินค้า
CREATE TABLE categories (
    category_id   SERIAL PRIMARY KEY,
    category_name VARCHAR(50) NOT NULL UNIQUE
);

-- ตารางสินค้า
CREATE TABLE products (
    product_id    SERIAL PRIMARY KEY,
    product_name  VARCHAR(100) NOT NULL,
    category_id   INTEGER REFERENCES categories(category_id),
    price         NUMERIC(10,2) NOT NULL CHECK (price >= 0),
    cost          NUMERIC(10,2) NOT NULL DEFAULT 0,
    stock_qty     INTEGER NOT NULL DEFAULT 0 CHECK (stock_qty >= 0),
    is_active     BOOLEAN NOT NULL DEFAULT TRUE,
    updated_at    TIMESTAMP NOT NULL DEFAULT NOW()
);

-- ตารางลูกค้า
CREATE TABLE customers (
    customer_id   SERIAL PRIMARY KEY,
    full_name     VARCHAR(100) NOT NULL,
    email         VARCHAR(100) UNIQUE,
    member_level  VARCHAR(20) NOT NULL DEFAULT 'bronze',
    total_spent   NUMERIC(12,2) NOT NULL DEFAULT 0,
    joined_at     DATE NOT NULL DEFAULT CURRENT_DATE
);

-- ตารางออเดอร์
CREATE TABLE orders (
    order_id      SERIAL PRIMARY KEY,
    customer_id   INTEGER REFERENCES customers(customer_id),
    order_date    TIMESTAMP NOT NULL DEFAULT NOW(),
    status        VARCHAR(20) NOT NULL DEFAULT 'pending',
    total_amount  NUMERIC(12,2) NOT NULL DEFAULT 0
);

-- ตารางรายการสินค้าในแต่ละออเดอร์
CREATE TABLE order_items (
    order_item_id SERIAL PRIMARY KEY,
    order_id      INTEGER REFERENCES orders(order_id),
    product_id    INTEGER REFERENCES products(product_id),
    quantity      INTEGER NOT NULL CHECK (quantity > 0),
    unit_price    NUMERIC(10,2) NOT NULL
);

-- ข้อมูลหมวดหมู่
INSERT INTO categories (category_name) VALUES
    ('กาแฟร้อน'),
    ('กาแฟเย็น'),
    ('ชา'),
    ('เบเกอรี่'),
    ('ของว่าง');

-- ข้อมูลสินค้า
INSERT INTO products (product_name, category_id, price, cost, stock_qty, is_active) VALUES
    ('เอสเปรสโซ่',        1, 55.00, 20.00, 100, TRUE),
    ('อเมริกาโน่ร้อน',     1, 60.00, 22.00, 100, TRUE),
    ('ลาเต้ร้อน',          1, 65.00, 25.00, 80,  TRUE),
    ('อเมริกาโน่เย็น',     2, 65.00, 22.00, 120, TRUE),
    ('ลาเต้เย็น',          2, 70.00, 25.00, 120, TRUE),
    ('คาปูชิโน่เย็น',       2, 70.00, 26.00, 90,  TRUE),
    ('มอคค่าเย็น',         2, 75.00, 28.00, 60,  TRUE),
    ('ชาเขียวเย็น',        3, 60.00, 20.00, 70,  TRUE),
    ('ชาไทยเย็น',          3, 55.00, 18.00, 70,  TRUE),
    ('ครัวซองต์',          4, 45.00, 18.00, 30,  TRUE),
    ('บราวนี่',            4, 50.00, 20.00, 25,  TRUE),
    ('คุกกี้ช็อกโกแลต',     5, 35.00, 12.00, 40,  TRUE),
    ('เค้กกล้วยหอม',       4, 55.00, 22.00, 0,   TRUE),
    ('เฟรนช์ฟรายส์',       5, 45.00, 15.00, 15,  FALSE);

-- ข้อมูลลูกค้า
INSERT INTO customers (full_name, email, member_level, total_spent) VALUES
    ('สมชาย ใจดี',      'somchai@example.com',   'bronze', 850.00),
    ('สมหญิง รักเรียน',  'somying@example.com',   'silver', 3200.00),
    ('วิชัย มั่งมี',       'wichai@example.com',    'gold',   8700.00),
    ('มาลี สวยงาม',      'malee@example.com',     'bronze', 450.00),
    ('ประยุทธ ขยันทำงาน', 'prayut@example.com',    'silver', 2900.00);

-- ข้อมูลออเดอร์
INSERT INTO orders (customer_id, order_date, status, total_amount) VALUES
    (1, NOW() - INTERVAL '5 days', 'completed', 130.00),
    (2, NOW() - INTERVAL '4 days', 'completed', 205.00),
    (3, NOW() - INTERVAL '3 days', 'completed', 340.00),
    (1, NOW() - INTERVAL '2 days', 'pending',   65.00),
    (4, NOW() - INTERVAL '1 days', 'pending',   90.00),
    (5, NOW(),                     'pending',   150.00),
    (2, NOW(),                     'cancelled', 70.00);

-- ข้อมูลรายการสินค้าต่อออเดอร์ (ตัวอย่างบางส่วน)
INSERT INTO order_items (order_id, product_id, quantity, unit_price) VALUES
    (1, 1, 1, 55.00),
    (1, 3, 1, 65.00),
    (1, 12, 1, 35.00) -- ยอดจริงต่างจาก total_amount เดิมเล็กน้อย ใช้เพื่อฝึก recalculation
;
```

> หมายเหตุ: ตัวเลข `order_id`, `product_id` ที่ได้จาก `SERIAL` อาจไม่ตรงกับตัวอย่างเป๊ะ ๆ หากคุณรันสคริปต์นี้ในฐานข้อมูลที่เคยมีข้อมูลมาก่อน แนะนำให้รันในฐานข้อมูลทดสอบใหม่ หรือปรับ `WHERE` clause ในตัวอย่างให้ตรงกับข้อมูลจริงของคุณ

ตรวจสอบข้อมูลก่อนเริ่ม:

```sql
SELECT product_id, product_name, price, stock_qty FROM products ORDER BY product_id;
```

ผลลัพธ์ตัวอย่าง (ย่อบางแถว):

```
 product_id |   product_name   | price | stock_qty
------------+-------------------+-------+-----------
          1 | เอสเปรสโซ่         | 55.00 |       100
          2 | อเมริกาโน่ร้อน      | 60.00 |       100
          3 | ลาเต้ร้อน           | 65.00 |        80
          4 | อเมริกาโน่เย็น      | 65.00 |       120
          5 | ลาเต้เย็น           | 70.00 |       120
          6 | คาปูชิโน่เย็น        | 70.00 |        90
          7 | มอคค่าเย็น          | 75.00 |        60
          8 | ชาเขียวเย็น         | 60.00 |        70
          9 | ชาไทยเย็น           | 55.00 |        70
         10 | ครัวซองต์           | 45.00 |        30
         11 | บราวนี่             | 50.00 |        25
         12 | คุกกี้ช็อกโกแลต       | 35.00 |        40
         13 | เค้กกล้วยหอม        | 55.00 |         0
         14 | เฟรนช์ฟรายส์         | 45.00 |        15
(14 rows)
```

---

## Step 121: UPDATE พื้นฐาน syntax — SET column = value WHERE condition

คำสั่ง `UPDATE` ใช้แก้ไขค่าของแถวที่มีอยู่แล้วในตาราง โครงสร้างพื้นฐานคือ:

```sql
UPDATE table_name
SET column_name = new_value
WHERE condition;
```

หลักการสำคัญที่ต้องจำให้ขึ้นใจ: **`WHERE` clause คือตัวกำหนดว่าจะแก้ไข "แถวไหน"** ถ้าไม่มี `WHERE` คำสั่งจะแก้ไขข้อมูล**ทุกแถวในตาราง** (รายละเอียดเรื่องนี้อยู่ใน Step 123)

ตัวอย่าง: ปรับสต็อกของ "เค้กกล้วยหอม" (product_id = 13) ที่หมดสต็อกอยู่ ให้มีของเข้าใหม่ 20 ชิ้น

```sql
UPDATE products
SET stock_qty = 20
WHERE product_id = 13;
```

ผลลัพธ์ที่ psql แสดง:

```
UPDATE 1
```

ข้อความ `UPDATE 1` หมายความว่ามีแถวที่ถูกแก้ไขจำนวน 1 แถว ซึ่งเป็นข้อมูลสำคัญที่ควรสังเกตทุกครั้ง — ถ้าตัวเลขไม่ตรงกับที่คาดไว้ (เช่น คาดว่าจะแก้ 1 แถวแต่ขึ้น `UPDATE 5`) แปลว่า `WHERE` clause อาจเขียนผิด

ตรวจสอบผลลัพธ์:

```sql
SELECT product_id, product_name, stock_qty FROM products WHERE product_id = 13;
```

```
 product_id |  product_name  | stock_qty
------------+----------------+-----------
         13 | เค้กกล้วยหอม     |        20
(1 row)
```

### การอัปเดตด้วยเงื่อนไขที่ครอบคลุมหลายแถว

`WHERE` ไม่จำเป็นต้องระบุ primary key เสมอไป สามารถใช้เงื่อนไขใด ๆ ที่ `SELECT` ใช้ได้ เช่น เปิดขายสินค้าที่เคยปิด (is_active = FALSE) กลับมาขายอีกครั้ง:

```sql
UPDATE products
SET is_active = TRUE
WHERE product_id = 14;
```

```
UPDATE 1
```

### การอัปเดตด้วยค่าจากคอลัมน์อื่นในตารางเดียวกัน

`SET` รับค่าคงที่ (literal) หรือ expression ก็ได้ เช่น กำหนดให้ `updated_at` เป็นเวลาปัจจุบันทุกครั้งที่แก้ไข:

```sql
UPDATE products
SET price = 58.00,
    updated_at = NOW()
WHERE product_id = 1;
```

```
UPDATE 1
```

> **เกร็ดความรู้**: PostgreSQL ไม่ได้อัปเดต `updated_at` ให้อัตโนมัติเหมือนบาง framework — ถ้าต้องการให้อัปเดตอัตโนมัติทุกครั้งที่มีการแก้ไขแถว ต้องเขียน `updated_at = NOW()` ไว้ใน `SET` เอง หรือสร้าง trigger (จะกล่าวถึงในบท Advanced ต่อไป)

---

## Step 122: UPDATE หลายคอลัมน์พร้อมกัน

การอัปเดตหลายคอลัมน์ในคำสั่งเดียวทำได้โดยคั่นแต่ละคู่ `column = value` ด้วยเครื่องหมายจุลภาค (`,`) ภายใน `SET` เดียว — **ไม่ต้องเขียน `SET` ซ้ำหลายครั้ง**

```sql
UPDATE products
SET price      = 68.00,
    cost        = 26.00,
    stock_qty   = 150,
    updated_at  = NOW()
WHERE product_id = 3;
```

```
UPDATE 1
```

ตรวจสอบก่อน/หลัง:

```sql
SELECT product_id, product_name, price, cost, stock_qty, updated_at
FROM products
WHERE product_id = 3;
```

**ก่อนอัปเดต:**

```
 product_id | product_name | price | cost  | stock_qty |      updated_at
------------+--------------+-------+-------+-----------+------------------------
          3 | ลาเต้ร้อน      | 65.00 | 25.00 |        80 | 2026-09-20 09:12:03.11
```

**หลังอัปเดต:**

```
 product_id | product_name | price | cost  | stock_qty |      updated_at
------------+--------------+-------+-------+-----------+------------------------
          3 | ลาเต้ร้อน      | 68.00 | 26.00 |       150 | 2026-09-25 14:03:21.55
```

### ข้อควรระวัง: syntax ที่ผิด

มือใหม่มักเขียนผิดโดยใส่ `SET` ซ้ำในแต่ละคอลัมน์ ซึ่ง **ไม่ใช่ syntax ที่ถูกต้อง** ใน PostgreSQL:

```sql
-- ❌ ผิด! ห้ามใช้ SET ซ้ำแบบนี้
UPDATE products
SET price = 68.00
SET cost = 26.00
WHERE product_id = 3;
```

```
ERROR:  syntax error at or near "SET"
LINE 3: SET cost = 26.00
        ^
```

วิธีที่ถูกต้องคือใช้ `SET` เพียงครั้งเดียว แล้วคั่นด้วยจุลภาคตามตัวอย่างด้านบน

### การอัปเดตหลายคอลัมน์แบบ row-value syntax

PostgreSQL ยังรองรับ syntax แบบ "row value" ที่กระชับขึ้น โดยเฉพาะเมื่อค่าที่ set มาจาก subquery เดียวกันหลายคอลัมน์:

```sql
UPDATE products
SET (price, cost) = (72.00, 28.00)
WHERE product_id = 7;
```

```
UPDATE 1
```

รูปแบบนี้มีประโยชน์มากเมื่อใช้ร่วมกับ subquery ที่คืนค่าหลายคอลัมน์พร้อมกัน (จะสาธิตเพิ่มเติมใน Step 127)

---

## Step 123: อันตรายของการลืม WHERE clause

นี่คือหนึ่งในความผิดพลาดที่**อันตรายที่สุด**ในการเขียน SQL — และเป็นเรื่องที่เกิดขึ้นจริงกับมือใหม่และมือเก๋าได้เท่า ๆ กัน

### ตัวอย่างหายนะ: UPDATE ที่ลืม WHERE

สมมติว่าตั้งใจจะปรับราคาสินค้าตัวเดียว (product_id = 5) แต่พิมพ์ตกหล่นบรรทัด `WHERE` ไป:

```sql
-- ⚠️ อันตราย! ไม่มี WHERE clause
UPDATE products
SET price = 70.00;
```

```
UPDATE 14
```

สังเกตข้อความ `UPDATE 14` — นั่นหมายความว่า **สินค้าทุกตัวในตาราง (14 แถว) ถูกเปลี่ยนราคาเป็น 70.00 บาทหมด** ไม่ว่าจะเป็นเอสเปรสโซ่ ลาเต้ ครัวซองต์ หรือคุกกี้ ราคากลายเป็น 70 บาทเท่ากันหมด ซึ่งเป็นความเสียหายทางธุรกิจที่ร้ายแรงมาก

```sql
SELECT product_id, product_name, price FROM products ORDER BY product_id LIMIT 5;
```

```
 product_id |   product_name   | price
------------+-------------------+-------
          1 | เอสเปรสโซ่         | 70.00
          2 | อเมริกาโน่ร้อน      | 70.00
          3 | ลาเต้ร้อน           | 70.00
          4 | อเมริกาโน่เย็น      | 70.00
          5 | ลาเต้เย็น           | 70.00
```

ราคาสินค้าทุกชนิดกลายเป็น 70.00 เหมือนกันหมด — ข้อมูลเดิมสูญหายและไม่สามารถย้อนกลับได้อีก **ถ้าคำสั่งนี้ถูก commit ไปแล้ว**

### เหตุใดเรื่องนี้จึงเกิดขึ้นบ่อย

1. พิมพ์ query ยาว ๆ แล้วลืมบรรทัดสุดท้าย
2. Comment out เงื่อนไข `WHERE` เพื่อทดสอบ แล้วลืม uncomment กลับ
3. Copy-paste query จากที่อื่นโดยตัดส่วน `WHERE` ทิ้งโดยไม่ตั้งใจ
4. กด Execute ทั้งไฟล์ (all statements) ทั้งที่ตั้งใจจะรันแค่บางบรรทัด

### วิธีป้องกัน 1: ใช้ Transaction (BEGIN...ROLLBACK) ทดสอบก่อนเสมอ

วิธีที่ปลอดภัยที่สุดคือ **ห่อคำสั่ง UPDATE ด้วย transaction** แล้วตรวจสอบผลลัพธ์ก่อนจะ `COMMIT` จริง:

```sql
BEGIN;

UPDATE products
SET price = 70.00;
```

```
BEGIN
UPDATE 14
```

ตอนนี้ยังไม่มีอะไรถูก commit — เราสามารถตรวจสอบผลลัพธ์ก่อนได้:

```sql
SELECT product_id, product_name, price FROM products ORDER BY product_id LIMIT 5;
```

```
 product_id |   product_name   | price
------------+-------------------+-------
          1 | เอสเปรสโซ่         | 70.00
          2 | อเมริกาโน่ร้อน      | 70.00
          3 | ลาเต้ร้อน           | 70.00
          4 | อเมริกาโน่เย็น      | 70.00
          5 | ลาเต้เย็น           | 70.00
```

เมื่อเห็นว่าผิดพลาด (ราคาสินค้าทุกตัวเท่ากันหมดทั้งที่ไม่ควรเป็นแบบนั้น) ให้สั่ง `ROLLBACK` ทันทีเพื่อยกเลิกการเปลี่ยนแปลงทั้งหมด:

```sql
ROLLBACK;
```

```
ROLLBACK
```

ตรวจสอบอีกครั้งว่าข้อมูลกลับมาเป็นปกติ:

```sql
SELECT product_id, product_name, price FROM products ORDER BY product_id LIMIT 5;
```

```
 product_id |   product_name   | price
------------+-------------------+-------
          1 | เอสเปรสโซ่         | 58.00
          2 | อเมริกาโน่ร้อน      | 60.00
          3 | ลาเต้ร้อน           | 68.00
          4 | อเมริกาโน่เย็น      | 65.00
          5 | ลาเต้เย็น           | 70.00
```

ข้อมูลกลับมาเหมือนก่อนรัน `UPDATE` ทุกประการ — นี่คือพลังของ transaction

### วิธีป้องกัน 2: SELECT ก่อนเสมอ เพื่อยืนยันจำนวนแถวที่จะถูกแก้ไข

ก่อนจะรัน `UPDATE` จริง ให้เขียน `SELECT` ด้วยเงื่อนไข `WHERE` เดียวกันก่อน เพื่อดูว่าจะกระทบกี่แถว:

```sql
-- ขั้นที่ 1: ตรวจสอบก่อนว่าจะกระทบกี่แถว และเป็นแถวที่ถูกต้องหรือไม่
SELECT product_id, product_name, price
FROM products
WHERE category_id = 2;
```

```
 product_id |   product_name   | price
------------+-------------------+-------
          4 | อเมริกาโน่เย็น      | 65.00
          5 | ลาเต้เย็น           | 70.00
          6 | คาปูชิโน่เย็น        | 70.00
          7 | มอคค่าเย็น          | 72.00
(4 rows)
```

```sql
-- ขั้นที่ 2: เมื่อมั่นใจแล้วว่าเงื่อนไขถูกต้อง ค่อยเปลี่ยน SELECT เป็น UPDATE
UPDATE products
SET price = price + 2.00
WHERE category_id = 2;
```

```
UPDATE 4
```

จำนวนแถวที่ `UPDATE 4` ตรงกับจำนวนแถวที่ `SELECT` แสดงไว้ก่อนหน้า — เป็นสัญญาณยืนยันว่าคำสั่งทำงานถูกต้องตามที่คาดไว้

### วิธีป้องกัน 3: ใช้ Primary Key หรือเงื่อนไขที่เจาะจงที่สุดเท่าที่จะทำได้

หลีกเลี่ยงการเขียน `WHERE` ที่กว้างเกินไปโดยไม่จำเป็น และควรใช้ `product_id = 5` แทน `product_name LIKE '%เย็น%'` เมื่อรู้ ID ที่แน่นอน เพราะเงื่อนไขที่เจาะจงจะลดความเสี่ยงจากการจับคู่แถวเกินความตั้งใจ

### วิธีป้องกัน 4: ปิด autocommit เมื่อทำงานกับข้อมูลสำคัญ (psql)

ใน `psql` สามารถตั้งค่าให้ทุกคำสั่งอยู่ใน transaction เสมอ โดยไม่ auto-commit จนกว่าจะสั่งเอง:

```sql
\set AUTOCOMMIT off
```

เมื่อปิด autocommit แล้ว ทุกคำสั่งจะต้องปิดท้ายด้วย `COMMIT` หรือ `ROLLBACK` เสมอ ทำให้มีจังหวะได้ตรวจสอบก่อนยืนยันทุกครั้ง

> **กฎเหล็กที่ควรยึดถือ**: ทุกครั้งที่เขียน `UPDATE` (หรือ `DELETE`) กับข้อมูลจริงในระบบ production ให้ **เขียน `BEGIN` ก่อนเสมอ**, รันคำสั่ง, `SELECT` ตรวจสอบผลลัพธ์, แล้วค่อย `COMMIT` — ฝึกเป็นนิสัยตั้งแต่วันนี้จะช่วยป้องกันหายนะในอนาคตได้มาก

---

## Step 124: UPDATE ด้วยค่าที่คำนวณจากคอลัมน์เดิม

หนึ่งในความสามารถที่ทรงพลังของ `UPDATE` คือการใช้ **ค่าคอลัมน์เดิมของแถวนั้นเอง** มาคำนวณเป็นค่าใหม่ ผ่าน expression ฝั่งขวาของเครื่องหมาย `=`

### ตัวอย่าง: เพิ่มราคาสินค้าหมวดกาแฟร้อนขึ้น 10%

```sql
BEGIN;

SELECT product_id, product_name, price
FROM products
WHERE category_id = 1;
```

```
 product_id |  product_name  | price
------------+-----------------+-------
          1 | เอสเปรสโซ่       | 58.00
          2 | อเมริกาโน่ร้อน    | 60.00
          3 | ลาเต้ร้อน         | 68.00
(3 rows)
```

```sql
UPDATE products
SET price = ROUND(price * 1.10, 2)
WHERE category_id = 1;
```

```
UPDATE 3
```

```sql
SELECT product_id, product_name, price
FROM products
WHERE category_id = 1;
```

```
 product_id |  product_name  | price
------------+-----------------+-------
          1 | เอสเปรสโซ่       | 63.80
          2 | อเมริกาโน่ร้อน    | 66.00
          3 | ลาเต้ร้อน         | 74.80
(3 rows)
```

```sql
COMMIT;
```

`ROUND(price * 1.10, 2)` คำนวณราคาใหม่จากราคาเดิมคูณ 1.10 (เพิ่ม 10%) แล้วปัดเศษทศนิยมให้เหลือ 2 ตำแหน่งเพื่อความเหมาะสมกับหน่วยเงินบาท

### ตัวอย่าง: ลดสต็อกหลังขายสินค้า

ระบบขายของมักต้องลดจำนวนสต็อกลงเมื่อมีการขาย เช่น ขายลาเต้เย็น (product_id = 5) ไป 3 แก้ว:

```sql
UPDATE products
SET stock_qty = stock_qty - 3
WHERE product_id = 5;
```

```
UPDATE 1
```

### การป้องกันค่าติดลบด้วย GREATEST() หรือ CHECK constraint

เนื่องจากตาราง `products` มี `CHECK (stock_qty >= 0)` การลดสต็อกเกินกว่าที่มีจะทำให้เกิด error โดยอัตโนมัติ:

```sql
UPDATE products
SET stock_qty = stock_qty - 1000
WHERE product_id = 5;
```

```
ERROR:  new row for relation "products" violates check constraint "products_stock_qty_check"
DETAIL:  Failing row contains (5, ลาเต้เย็น, 2, 70.00, 25.00, -883, t, 2026-09-25 14:20:11.02).
```

นี่คือตัวอย่างที่ดีว่าทำไม CHECK constraint (จาก Part ก่อนหน้า) จึงสำคัญ — มันช่วยป้องกันข้อมูลผิดพลาดแม้ query จะเขียนพลาดไปก็ตาม หากต้องการป้องกันไม่ให้ error โดยจำกัดค่าต่ำสุดไว้ที่ 0 แทน สามารถใช้ `GREATEST()`:

```sql
UPDATE products
SET stock_qty = GREATEST(stock_qty - 1000, 0)
WHERE product_id = 5;
```

```
UPDATE 1
```

```sql
SELECT product_id, product_name, stock_qty FROM products WHERE product_id = 5;
```

```
 product_id | product_name | stock_qty
------------+--------------+-----------
          5 | ลาเต้เย็น      |         0
(1 row)
```

### ตัวอย่าง: อัปเดตส่วนต่างกำไร (margin) จากคอลัมน์ price และ cost

```sql
UPDATE products
SET price = cost * 2.5
WHERE category_id = 4;  -- หมวดเบเกอรี่ ปรับราคาให้ได้กำไร 150% จากต้นทุน
```

```
UPDATE 3
```

```sql
SELECT product_id, product_name, cost, price
FROM products
WHERE category_id = 4;
```

```
 product_id | product_name  | cost  | price
------------+----------------+-------+--------
         10 | ครัวซองต์       | 18.00 |  45.00
         11 | บราวนี่         | 20.00 |  50.00
         13 | เค้กกล้วยหอม    | 22.00 |  55.00
(3 rows)
```

---

## Step 125: UPDATE ... FROM — อัปเดตโดยอ้างอิงข้อมูลจากตารางอื่น

บ่อยครั้งที่ค่าที่ต้องการอัปเดตไม่ได้อยู่ในตารางเดียวกัน แต่ต้องคำนวณหรืออ้างอิงจากตารางอื่น PostgreSQL มี syntax พิเศษคือ `UPDATE ... FROM` ที่ทำให้เขียนได้กระชับกว่าการใช้ subquery ซ้ำหลายจุด

โครงสร้าง:

```sql
UPDATE target_table
SET column = source_table.column
FROM source_table
WHERE target_table.key = source_table.key;
```

### ตัวอย่าง: อัปเดตยอดใช้จ่ายรวม (total_spent) ของลูกค้าจากออเดอร์ที่เสร็จสมบูรณ์จริง

สมมติว่าค่า `total_spent` ในตาราง `customers` เก่าไม่ตรงกับข้อมูลจริงในตาราง `orders` แล้ว ต้องการคำนวณใหม่จากผลรวมของออเดอร์ที่ `status = 'completed'`

ก่อนอื่นดูข้อมูลปัจจุบัน:

```sql
SELECT customer_id, full_name, total_spent FROM customers ORDER BY customer_id;
```

```
 customer_id |    full_name     | total_spent
-------------+-------------------+-------------
           1 | สมชาย ใจดี         |      850.00
           2 | สมหญิง รักเรียน     |     3200.00
           3 | วิชัย มั่งมี         |     8700.00
           4 | มาลี สวยงาม         |      450.00
           5 | ประยุทธ ขยันทำงาน   |     2900.00
(5 rows)
```

```sql
UPDATE customers
SET total_spent = order_summary.sum_total
FROM (
    SELECT customer_id, SUM(total_amount) AS sum_total
    FROM orders
    WHERE status = 'completed'
    GROUP BY customer_id
) AS order_summary
WHERE customers.customer_id = order_summary.customer_id;
```

```
UPDATE 3
```

สังเกตว่า `UPDATE 3` เพราะมีลูกค้าเพียง 3 คนที่มีออเดอร์สถานะ `completed` (ลูกค้าที่ไม่มีออเดอร์ completed เลยจะไม่ถูกแก้ไข ค่าเดิมยังคงอยู่)

```sql
SELECT customer_id, full_name, total_spent FROM customers ORDER BY customer_id;
```

```
 customer_id |    full_name     | total_spent
-------------+-------------------+-------------
           1 | สมชาย ใจดี         |      130.00
           2 | สมหญิง รักเรียน     |      205.00
           3 | วิชัย มั่งมี         |      340.00
           4 | มาลี สวยงาม         |      450.00
           5 | ประยุทธ ขยันทำงาน   |     2900.00
(5 rows)
```

ลูกค้า `customer_id = 4` และ `5` ไม่มีออเดอร์ที่ `status = 'completed'` เลย จึงยังคงค่าเดิมไว้ (ไม่ถูกแตะต้อง) นี่คือพฤติกรรมสำคัญที่ต้องเข้าใจ: `UPDATE ... FROM` จะแก้ไขเฉพาะแถวที่ `WHERE` เจอคู่ (match) เท่านั้น ถ้าต้องการให้ลูกค้าที่ไม่มีออเดอร์ completed มีค่าเป็น 0 ต้องเขียนแยกหรือใช้ `COALESCE` ร่วมกับ subquery แบบ correlated

### ตัวอย่าง: อัปเดตราคาต่อหน่วยใน order_items ให้ตรงกับราคาปัจจุบันของสินค้า

```sql
UPDATE order_items oi
SET unit_price = p.price
FROM products p
WHERE oi.product_id = p.product_id
  AND oi.order_id = 1;
```

```
UPDATE 3
```

การใช้ **table alias** (`oi`, `p`) ช่วยให้เขียน query สั้นและอ่านง่ายขึ้นมาก โดยเฉพาะเมื่อชื่อคอลัมน์ซ้ำกันระหว่างตาราง (เช่น `product_id` มีทั้งสองตาราง)

```sql
SELECT oi.order_item_id, oi.product_id, p.product_name, oi.unit_price
FROM order_items oi
JOIN products p ON oi.product_id = p.product_id
WHERE oi.order_id = 1;
```

```
 order_item_id | product_id |  product_name  | unit_price
----------------+------------+-----------------+------------
              1 |          1 | เอสเปรสโซ่       |      63.80
              2 |          3 | ลาเต้ร้อน         |      74.80
              3 |         12 | คุกกี้ช็อกโกแลต    |      35.00
(3 rows)
```

### ข้อควรระวัง: UPDATE ... FROM กับตารางที่ join แล้วได้หลายแถว

ถ้า `FROM` join แล้วได้มากกว่า 1 แถวต่อ 1 แถวเป้าหมาย ผลลัพธ์จะไม่แน่นอนว่าจะใช้แถวไหน (PostgreSQL จะเลือกแถวใดแถวหนึ่งแบบไม่รับประกันลำดับ) ดังนั้นควรมั่นใจว่าความสัมพันธ์เป็นแบบ 1 ต่อ 1 หรือใช้ `GROUP BY`/`DISTINCT ON` ใน subquery ก่อนเสมอ เหมือนตัวอย่าง `order_summary` ด้านบนที่ใช้ `GROUP BY customer_id` เพื่อรับประกันว่าแต่ละ `customer_id` มีเพียงแถวเดียว

---

## Step 126: UPDATE ... RETURNING — ดูค่าที่เปลี่ยนแปลง

ปกติแล้ว `UPDATE` จะคืนแค่ข้อความ `UPDATE n` โดยไม่แสดงข้อมูลที่ถูกแก้ไข ถ้าต้องการดูค่าจริงหลังแก้ไข (หรือค่าก่อนแก้ไข) ในทันทีโดยไม่ต้อง `SELECT` แยก สามารถใช้ `RETURNING` ต่อท้ายคำสั่งได้

### ตัวอย่างพื้นฐาน: RETURNING คืนค่าคอลัมน์ที่ระบุ

```sql
UPDATE products
SET price = ROUND(price * 1.05, 2)
WHERE category_id = 2
RETURNING product_id, product_name, price;
```

```
 product_id |   product_name   | price
------------+-------------------+-------
          4 | อเมริกาโน่เย็น      | 70.35
          5 | ลาเต้เย็น           | 73.50
          6 | คาปูชิโน่เย็น        | 73.50
          7 | มอคค่าเย็น          | 75.60
UPDATE 4
```

สังเกตว่าผลลัพธ์แสดงทั้งตารางค่าที่อัปเดตแล้ว **และ** ข้อความ `UPDATE 4` ต่อท้าย — `RETURNING` ทำให้เราเห็นผลลัพธ์ในคำสั่งเดียว ไม่ต้องรัน `SELECT` ซ้ำ

### RETURNING ทุกคอลัมน์ด้วย *

```sql
UPDATE products
SET stock_qty = stock_qty + 50
WHERE product_id = 13
RETURNING *;
```

```
 product_id | product_name  | category_id | price | cost  | stock_qty | is_active |        updated_at
------------+----------------+-------------+-------+-------+-----------+-----------+---------------------------
         13 | เค้กกล้วยหอม    |           4 | 55.00 | 22.00 |        70 | t         | 2026-09-25 14:35:02.881
(1 row)
UPDATE 1
```

### RETURNING เพื่อเปรียบเทียบค่าก่อน-หลัง (ใช้ expression)

เราสามารถใช้ expression ใน `RETURNING` เพื่อคำนวณส่วนต่างได้ทันที เช่น แสดงราคาที่เปลี่ยนไป:

```sql
UPDATE products
SET price = price - 3.00
WHERE category_id = 3
RETURNING
    product_id,
    product_name,
    price + 3.00 AS old_price,
    price          AS new_price,
    price - (price + 3.00) AS price_change;
```

```
 product_id |  product_name  | old_price | new_price | price_change
------------+-----------------+-----------+-----------+---------------
          8 | ชาเขียวเย็น      |     60.00 |     57.00 |         -3.00
          9 | ชาไทยเย็น        |     55.00 |     52.00 |         -3.00
(2 rows)
UPDATE 2
```

### การใช้ RETURNING ร่วมกับ CTE เพื่อบันทึก log การเปลี่ยนแปลง

`RETURNING` มีประโยชน์มากเมื่อใช้ร่วมกับ Common Table Expression (CTE) เพื่อนำผลลัพธ์การอัปเดตไปใช้ต่อในคำสั่งเดียว เช่น การเก็บ log ราคาที่เปลี่ยนแปลง (สมมติมีตาราง `price_change_log`):

```sql
CREATE TABLE IF NOT EXISTS price_change_log (
    log_id       SERIAL PRIMARY KEY,
    product_id   INTEGER,
    old_price    NUMERIC(10,2),
    new_price    NUMERIC(10,2),
    changed_at   TIMESTAMP DEFAULT NOW()
);

WITH updated AS (
    UPDATE products
    SET price = ROUND(price * 1.08, 2)
    WHERE category_id = 1
    RETURNING product_id, price AS new_price,
              ROUND(price / 1.08, 2) AS old_price
)
INSERT INTO price_change_log (product_id, old_price, new_price)
SELECT product_id, old_price, new_price FROM updated;
```

```
INSERT 0 3
```

```sql
SELECT * FROM price_change_log;
```

```
 log_id | product_id | old_price | new_price |         changed_at
--------+------------+-----------+-----------+----------------------------
      1 |          1 |     63.80 |     68.90 | 2026-09-25 14:41:09.223
      2 |          2 |     66.00 |     71.28 | 2026-09-25 14:41:09.223
      3 |          3 |     74.80 |     80.78 | 2026-09-25 14:41:09.223
(3 rows)
```

รูปแบบนี้ (data-modifying CTE) เป็นเทคนิคขั้นสูงที่ช่วยให้ "อัปเดต" และ "บันทึกประวัติ" ทำเป็น atomic operation เดียวได้ในคำสั่งเดียว จะกล่าวถึงลึกขึ้นในบทเรื่อง CTE โดยเฉพาะ

---

## Step 127: Subquery ใน UPDATE (ทั้งใน SET และ WHERE)

Subquery คือคำสั่ง `SELECT` ที่ซ้อนอยู่ภายในคำสั่งอื่น ใน `UPDATE` เราสามารถใช้ subquery ได้ทั้งในส่วน `SET` (เพื่อคำนวณค่าที่จะ set) และส่วน `WHERE` (เพื่อกำหนดว่าจะแก้ไขแถวไหน)

### Subquery ใน WHERE

ตัวอย่าง: เพิ่มระดับสมาชิก (member_level) เป็น `'gold'` ให้ลูกค้าที่มียอดสั่งซื้อรวมจากออเดอร์ completed เกิน 300 บาท

```sql
UPDATE customers
SET member_level = 'gold'
WHERE customer_id IN (
    SELECT customer_id
    FROM orders
    WHERE status = 'completed'
    GROUP BY customer_id
    HAVING SUM(total_amount) > 300
);
```

```
UPDATE 1
```

```sql
SELECT customer_id, full_name, member_level FROM customers ORDER BY customer_id;
```

```
 customer_id |    full_name     | member_level
-------------+-------------------+---------------
           1 | สมชาย ใจดี         | bronze
           2 | สมหญิง รักเรียน     | silver
           3 | วิชัย มั่งมี         | gold
           4 | มาลี สวยงาม         | bronze
           5 | ประยุทธ ขยันทำงาน   | silver
(5 rows)
```

มีเพียงลูกค้า `วิชัย มั่งมี` (customer_id = 3) เท่านั้นที่มียอด completed order รวมเกิน 300 บาท (340.00) จึงถูกอัปเกรดเป็น gold

### Subquery ใน WHERE ด้วย NOT IN / NOT EXISTS

ตัวอย่าง: ปิดการขาย (is_active = FALSE) สินค้าที่ไม่เคยถูกสั่งซื้อเลยแม้แต่ครั้งเดียว

```sql
UPDATE products
SET is_active = FALSE
WHERE product_id NOT IN (
    SELECT DISTINCT product_id
    FROM order_items
    WHERE product_id IS NOT NULL
);
```

```
UPDATE 11
```

> **ข้อควรระวังเรื่อง NOT IN กับ NULL**: หาก subquery ของ `NOT IN` มีโอกาสคืนค่า `NULL` ปะปนมาด้วย ผลลัพธ์ทั้งหมดของ `NOT IN` จะกลายเป็น unknown (ไม่มีแถวไหนตรงเงื่อนไขเลย) ซึ่งเป็นกับดักที่พบบ่อยมาก วิธีที่ปลอดภัยกว่าคือใช้ `NOT EXISTS` แทน:

```sql
UPDATE products p
SET is_active = FALSE
WHERE NOT EXISTS (
    SELECT 1
    FROM order_items oi
    WHERE oi.product_id = p.product_id
);
```

```
UPDATE 11
```

`NOT EXISTS` ไม่มีปัญหาเรื่อง `NULL` แบบ `NOT IN` และมักมีประสิทธิภาพดีกว่าเมื่อข้อมูลมีขนาดใหญ่ จึงเป็นรูปแบบที่แนะนำให้ใช้เป็นค่าเริ่มต้น

### Subquery ใน SET (scalar subquery)

Subquery ใน `SET` ต้องคืนค่าเพียง**หนึ่งแถวหนึ่งคอลัมน์** (scalar subquery) มิฉะนั้นจะเกิด error

ตัวอย่าง: ตั้งราคาสินค้าหมวด "เบเกอรี่" ให้เท่ากับราคาเฉลี่ยของสินค้าทั้งหมวด (ปรับให้ราคาเท่ากันหมดในหมวดนั้น เพื่อจัดโปรโมชัน "เบเกอรี่ราคาเดียว"):

```sql
UPDATE products
SET price = (
    SELECT ROUND(AVG(price), 2)
    FROM products
    WHERE category_id = 4
)
WHERE category_id = 4;
```

```
UPDATE 3
```

```sql
SELECT product_id, product_name, price FROM products WHERE category_id = 4;
```

```
 product_id | product_name  | price
------------+----------------+-------
         10 | ครัวซองต์       | 50.00
         11 | บราวนี่         | 50.00
         13 | เค้กกล้วยหอม    | 50.00
(3 rows)
```

### ตัวอย่าง scalar subquery ที่คืนค่ามากกว่า 1 แถว (error)

หากลืมใส่เงื่อนไขให้ subquery คืนค่าแถวเดียว จะเกิด error ทันที:

```sql
UPDATE products
SET price = (SELECT price FROM products WHERE category_id = 2)
WHERE product_id = 10;
```

```
ERROR:  more than one row returned by a subquery used as an expression
```

นี่เป็น error ที่พบบ่อยมากเมื่อเขียน subquery ใน `SET` — ต้องมั่นใจเสมอว่า subquery จะคืนค่าแค่แถวเดียว โดยใช้ `LIMIT 1`, aggregate function (`AVG`, `MAX`, `SUM`), หรือเงื่อนไขที่เจาะจงพอ

### Correlated subquery ใน SET

Subquery ที่อ้างอิงถึงคอลัมน์ของแถวปัจจุบันในตารางหลัก เรียกว่า correlated subquery ตัวอย่าง: อัปเดต `total_amount` ในตาราง `orders` ให้คำนวณจากผลรวมจริงของ `order_items`:

```sql
UPDATE orders o
SET total_amount = (
    SELECT COALESCE(SUM(oi.quantity * oi.unit_price), 0)
    FROM order_items oi
    WHERE oi.order_id = o.order_id
)
WHERE o.order_id = 1;
```

```
UPDATE 1
```

```sql
SELECT order_id, total_amount FROM orders WHERE order_id = 1;
```

```
 order_id | total_amount
----------+---------------
        1 |        173.60
(1 row)
```

ในตัวอย่างนี้ `o.order_id` ภายใน subquery อ้างอิงกลับไปที่แถวของตารางหลัก (`o`) ทำให้ subquery ถูกคำนวณใหม่สำหรับทุกแถวที่ตรงเงื่อนไข `WHERE` ของ `UPDATE` ด้านนอก — เทียบเท่ากับ `UPDATE ... FROM` ที่ใช้ `GROUP BY` แต่เขียนในสไตล์ correlated subquery แทน

---

## Step 128: CASE WHEN ภายใน UPDATE สำหรับ logic ที่ซับซ้อน

เมื่อค่าที่ต้องการ set ขึ้นอยู่กับเงื่อนไขหลายแบบพร้อมกัน (เช่น "ถ้าเงื่อนไข A ให้ค่า X, ถ้าเงื่อนไข B ให้ค่า Y, นอกนั้นให้ค่า Z") การเขียน `UPDATE` แยกหลายคำสั่งจะยุ่งยากและเสี่ยงต่อ race condition หากมีการอัปเดตพร้อมกัน วิธีที่ดีกว่าคือใช้ `CASE WHEN` ภายใน `SET` เพื่อรวมทุกเงื่อนไขไว้ในคำสั่งเดียว

### รูปแบบ CASE WHEN

```sql
CASE
    WHEN condition1 THEN result1
    WHEN condition2 THEN result2
    ...
    ELSE default_result
END
```

### ตัวอย่าง: ปรับระดับสมาชิก (member_level) ตามยอดใช้จ่ายรวมในคำสั่งเดียว

```sql
BEGIN;

SELECT customer_id, full_name, total_spent, member_level FROM customers ORDER BY customer_id;
```

```
 customer_id |    full_name     | total_spent | member_level
-------------+-------------------+-------------+---------------
           1 | สมชาย ใจดี         |      130.00 | bronze
           2 | สมหญิง รักเรียน     |      205.00 | silver
           3 | วิชัย มั่งมี         |      340.00 | gold
           4 | มาลี สวยงาม         |      450.00 | bronze
           5 | ประยุทธ ขยันทำงาน   |     2900.00 | silver
(5 rows)
```

```sql
UPDATE customers
SET member_level = CASE
    WHEN total_spent >= 5000 THEN 'platinum'
    WHEN total_spent >= 2000 THEN 'gold'
    WHEN total_spent >= 500  THEN 'silver'
    ELSE 'bronze'
END;
```

```
UPDATE 5
```

```sql
SELECT customer_id, full_name, total_spent, member_level FROM customers ORDER BY customer_id;
```

```
 customer_id |    full_name     | total_spent | member_level
-------------+-------------------+-------------+---------------
           1 | สมชาย ใจดี         |      130.00 | bronze
           2 | สมหญิง รักเรียน     |      205.00 | bronze
           3 | วิชัย มั่งมี         |      340.00 | bronze
           4 | มาลี สวยงาม         |      450.00 | bronze
           5 | ประยุทธ ขยันทำงาน   |     2900.00 | gold
(5 rows)
```

```sql
COMMIT;
```

สังเกตว่าคำสั่งเดียวสามารถกำหนดระดับสมาชิกให้ลูกค้าทั้ง 5 คนได้ถูกต้องตามเงื่อนไขที่แตกต่างกัน โดยไม่ต้องเขียน `UPDATE` แยกทีละเงื่อนไข — ลด I/O และรับประกันว่าข้อมูลจะสอดคล้องกันเสมอ เพราะทุกแถวถูกประเมินจากค่าที่อ่าน ณ เวลาเดียวกัน

### ตัวอย่าง: ปรับสถานะออเดอร์ตามอายุของออเดอร์ (order aging)

ธุรกิจจริงมักมีกฎว่า "ถ้าออเดอร์ pending ค้างนานเกิน 3 วัน ให้ยกเลิกอัตโนมัติ" ตัวอย่างนี้แสดงการใช้ `CASE WHEN` ร่วมกับฟังก์ชันวันที่:

```sql
UPDATE orders
SET status = CASE
    WHEN status = 'pending' AND order_date < NOW() - INTERVAL '3 days' THEN 'cancelled'
    WHEN status = 'pending' AND order_date >= NOW() - INTERVAL '3 days' THEN 'pending'
    ELSE status
END
WHERE status = 'pending';
```

```
UPDATE 3
```

การเขียน `ELSE status` (คงค่าเดิมไว้) เป็นเทคนิคสำคัญ — มันทำให้แถวที่ไม่เข้าเงื่อนไขใดเลยไม่ถูกเปลี่ยนแปลงค่า แม้จะอยู่ใน `WHERE` ที่ query เข้าถึงก็ตาม

### ตัวอย่าง: ปรับราคาแบบหลายเงื่อนไขซ้อนกัน (multi-condition pricing)

```sql
UPDATE products
SET price = CASE
    WHEN category_id = 1 AND stock_qty > 50 THEN ROUND(price * 0.95, 2)  -- กาแฟร้อนสต็อกเยอะ ลด 5%
    WHEN category_id = 2 AND stock_qty < 30 THEN ROUND(price * 1.10, 2)  -- กาแฟเย็นสต็อกน้อย ขึ้นราคา 10%
    WHEN is_active = FALSE                  THEN ROUND(price * 0.50, 2) -- สินค้าปิดการขาย เคลียร์ 50%
    ELSE price
END;
```

```
UPDATE 14
```

ตัวอย่างนี้แสดงให้เห็นว่า `CASE WHEN` สามารถรวมตรรกะทางธุรกิจที่ซับซ้อนหลายชั้นไว้ในคำสั่งเดียวได้อย่างมีประสิทธิภาพ แทนที่จะต้องเขียน `UPDATE` แยก 3-4 คำสั่ง ซึ่งแต่ละคำสั่งจะต้อง scan ตารางซ้ำและอาจเกิดผลลัพธ์ที่ไม่สอดคล้องกันหากมีการอ่าน-เขียนพร้อมกันจากหลาย connection

### CASE WHEN แบบ searched กับแบบ simple

นอกจาก simple `CASE WHEN condition THEN ...` แล้ว ยังมีรูปแบบ "simple case" ที่เทียบค่าคอลัมน์กับหลายค่า:

```sql
UPDATE customers
SET member_level = CASE member_level
    WHEN 'bronze' THEN 'silver'
    WHEN 'silver' THEN 'gold'
    WHEN 'gold'   THEN 'platinum'
    ELSE member_level
END
WHERE customer_id = 1;  -- โปรโมทลูกค้าคนนี้ขึ้นระดับถัดไป
```

```
UPDATE 1
```

รูปแบบนี้เหมาะกับกรณีที่เทียบค่าคอลัมน์เดียวกับหลายค่าคงที่ ทำให้ query อ่านง่ายกว่าการเขียน `WHEN column = 'x' THEN ...` ซ้ำ ๆ

---

## Step 129: UPDATE จำนวนมาก (bulk update) — ผลกระทบต่อ performance และการแบ่ง batch

เมื่อข้อมูลมีขนาดใหญ่ระดับล้านแถว การรัน `UPDATE` ที่กระทบทั้งตารางในคำสั่งเดียวอาจสร้างปัญหาต่อระบบได้หลายด้าน จำเป็นต้องเข้าใจกลไกเบื้องหลังก่อนจะออกแบบวิธีอัปเดตข้อมูลจำนวนมากอย่างปลอดภัย

### ทำไม bulk UPDATE ถึงส่งผลต่อ performance

1. **MVCC และ dead tuples**: PostgreSQL ใช้ระบบ MVCC (Multi-Version Concurrency Control) ซึ่งหมายความว่า `UPDATE` ไม่ได้แก้ไขแถวเดิมแบบ in-place แต่จะสร้างแถวใหม่ (new tuple version) แล้วทำเครื่องหมายแถวเก่าว่าเป็น "dead tuple" การอัปเดตหลายล้านแถวในคำสั่งเดียวจะสร้าง dead tuples จำนวนมหาศาล ทำให้ตารางบวม (table bloat) และต้องรอ `VACUUM` มาเก็บกวาดทีหลัง
2. **Lock contention**: `UPDATE` จะล็อกแถวที่ถูกแก้ไข (row-level lock) ตลอดช่วงเวลาที่ transaction ยังไม่ commit ถ้า transaction ใหญ่ใช้เวลานาน แถวเหล่านั้นจะถูกล็อกค้างไว้นาน ทำให้ transaction อื่นที่ต้องการแก้ไขแถวเดียวกันต้องรอ (blocking)
3. **WAL (Write-Ahead Log) volume**: การอัปเดตจำนวนมากสร้าง WAL record จำนวนมาก ซึ่งอาจกระทบ disk I/O, replication lag (ถ้ามี replica), และพื้นที่ดิสก์
4. **Transaction ขนาดใหญ่เสี่ยงต่อการ rollback ที่แพง**: ถ้า transaction ขนาดใหญ่ล้มเหลวกลางทาง (เช่น connection หลุด, ติด constraint) ระบบต้อง rollback ทั้งหมด ซึ่งใช้เวลาและทรัพยากรพอสมควร
5. **Index maintenance**: ทุกครั้งที่แถวถูกอัปเดต (โดยเฉพาะถ้าคอลัมน์ที่แก้ไขมี index) index ที่เกี่ยวข้องต้องถูกอัปเดตตามไปด้วย ยิ่งมี index เยอะ ยิ่งช้า

### ตัวอย่าง: การรัน bulk UPDATE แบบทีเดียวทั้งหมด (ไม่แนะนำสำหรับตารางใหญ่)

```sql
-- สมมติว่า order_items มีหลายล้านแถว การรันแบบนี้ในคำสั่งเดียวอาจล็อกตารางนานเกินไป
UPDATE order_items
SET unit_price = unit_price * 1.02
WHERE order_id IN (SELECT order_id FROM orders WHERE status = 'completed');
```

ในตารางเล็ก (หลักพัน-หมื่นแถว) คำสั่งนี้ไม่มีปัญหา แต่ในตารางที่มีหลายล้านแถว คำสั่งเดียวนี้อาจใช้เวลานานหลายนาทีถึงหลายชั่วโมง และล็อกแถวจำนวนมากพร้อมกัน ซึ่งอาจกระทบระบบที่ต้องให้บริการ (production) แบบ real-time

### เทคนิค: แบ่ง batch update ด้วย LIMIT + primary key range

แนวทางที่นิยมใช้คือแบ่งการอัปเดตออกเป็นชุดเล็ก ๆ (batch) แล้วรันทีละชุด โดยพักระหว่าง batch เพื่อลดผลกระทบต่อระบบ

**วิธีที่ 1: ใช้ subquery + LIMIT เพื่อจำกัดจำนวนแถวต่อรอบ**

```sql
UPDATE order_items
SET unit_price = ROUND(unit_price * 1.02, 2)
WHERE order_item_id IN (
    SELECT order_item_id
    FROM order_items
    WHERE unit_price = unit_price  -- เงื่อนไขจริง เช่น ยังไม่เคยถูกอัปเดตรอบนี้
    ORDER BY order_item_id
    LIMIT 1000
);
```

```
UPDATE 3
```

(ในตัวอย่างข้อมูลจริงของเรามีแค่ 3 แถว จึงอัปเดตครบในรอบเดียว แต่แนวคิดนี้ใช้กับตารางขนาดใหญ่ที่มีนับล้านแถวได้โดยรันคำสั่งนี้ซ้ำหลายรอบจนกว่า `UPDATE 0`)

**วิธีที่ 2: ใช้ flag คอลัมน์เพื่อติดตามว่าแถวไหน "อัปเดตแล้ว"**

เทคนิคที่ทำงานได้แน่นอนกว่าคือเพิ่มคอลัมน์ (หรือใช้คอลัมน์ที่มีอยู่ เช่น `updated_at`) เพื่อกันไม่ให้อัปเดตแถวเดิมซ้ำในรอบถัดไป:

```sql
-- รันคำสั่งนี้ซ้ำ ๆ เป็นรอบ ๆ (เช่นจาก script หรือ cron job) จนกว่าจะได้ UPDATE 0
UPDATE order_items
SET unit_price = ROUND(unit_price * 1.02, 2)
WHERE order_item_id IN (
    SELECT order_item_id
    FROM order_items
    WHERE unit_price < 100   -- เงื่อนไขที่ยังไม่ถึง "เป้าหมาย" หมายถึงยังไม่ได้อัปเดต
    ORDER BY order_item_id
    LIMIT 500
)
RETURNING order_item_id;
```

**วิธีที่ 3: แบ่งตามช่วง primary key (range-based batching)**

สำหรับตารางขนาดใหญ่มาก การแบ่งตามช่วง ID จะมีประสิทธิภาพดีกว่าการใช้ subquery + LIMIT เพราะสามารถใช้ index range scan ได้โดยตรง:

```sql
-- รอบที่ 1
UPDATE order_items
SET unit_price = ROUND(unit_price * 1.02, 2)
WHERE order_item_id BETWEEN 1 AND 100000;

-- รอบที่ 2 (รันหลังรอบแรก commit แล้ว)
UPDATE order_items
SET unit_price = ROUND(unit_price * 1.02, 2)
WHERE order_item_id BETWEEN 100001 AND 200000;

-- ทำซ้ำไปเรื่อย ๆ จนครบทุกช่วง
```

แต่ละรอบควรอยู่ใน transaction ของตัวเอง (`COMMIT` หลังจบแต่ละ batch) เพื่อไม่ให้ transaction เดียวใหญ่เกินไป และควรมีการ `sleep` เล็กน้อยระหว่างรอบ (เช่นในระดับ application หรือ script ภายนอก) เพื่อให้ระบบอื่น (เช่น replication, autovacuum) มีโอกาสได้ทำงานแทรก

### แนวทางปฏิบัติที่แนะนำสำหรับ bulk update ในระบบ production

| แนวทาง | เหตุผล |
|---|---|
| แบ่งเป็น batch ขนาดเล็ก (เช่น 1,000-10,000 แถวต่อรอบ) | ลดเวลา lock ต่อรอบ ลดผลกระทบต่อ transaction อื่น |
| Commit หลังจบแต่ละ batch | ลดขนาด transaction ลด risk การ rollback ที่แพง |
| รันในช่วง traffic ต่ำ (off-peak hours) | ลดการแย่งทรัพยากรกับ query ของผู้ใช้จริง |
| ตรวจสอบ `pg_stat_activity` ระหว่างรัน | เฝ้าดู lock wait และ query ที่ block กัน |
| พิจารณาใช้ `VACUUM ANALYZE` หลังอัปเดตจำนวนมาก | เก็บกวาด dead tuples และอัปเดต statistics ให้ query planner ทำงานแม่นยำ |
| หลีกเลี่ยงการอัปเดตคอลัมน์ที่มี index โดยไม่จำเป็น | ลด overhead ในการ maintain index ระหว่างอัปเดต |

```sql
-- ตรวจสอบว่ามี query ไหนถูก block อยู่หรือไม่ ระหว่างรัน bulk update
SELECT pid, state, wait_event_type, wait_event, query
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY query_start;
```

```sql
-- หลังอัปเดตจำนวนมากเสร็จแล้ว ควรรัน VACUUM ANALYZE เพื่อเก็บกวาด dead tuples
VACUUM ANALYZE order_items;
```

```
VACUUM
```

> **สรุปหลักคิด**: สำหรับตารางขนาดเล็ก-กลาง (ไม่กี่แสนแถว) การรัน `UPDATE` คำสั่งเดียวมักไม่มีปัญหา แต่เมื่อข้อมูลโตขึ้นถึงระดับล้านแถวในระบบที่ต้องให้บริการต่อเนื่อง (24/7) การแบ่ง batch คือแนวทางที่ปลอดภัยกว่าเสมอ แม้จะใช้เวลารวมมากกว่าเล็กน้อย แต่แลกกับความเสถียรของระบบที่คุ้มค่ากว่ามาก

---

## สรุปท้ายบท

ในบทนี้เราได้เรียนรู้คำสั่ง `UPDATE` อย่างครบถ้วนตั้งแต่ syntax พื้นฐานไปจนถึงเทคนิคระดับที่ใช้งานจริงในระบบ production:

- **Syntax พื้นฐาน**: `UPDATE table SET column = value WHERE condition` — `WHERE` คือหัวใจสำคัญที่กำหนดขอบเขตของการแก้ไข
- **หลายคอลัมน์พร้อมกัน**: คั่นด้วยจุลภาคภายใน `SET` เดียว หรือใช้ row-value syntax `(col1, col2) = (val1, val2)`
- **อันตรายของการลืม WHERE**: อาจทำให้ข้อมูลทั้งตารางถูกเขียนทับโดยไม่ตั้งใจ วิธีป้องกันที่สำคัญที่สุดคือฝึกใช้ `BEGIN` ก่อนเสมอ ตรวจสอบด้วย `SELECT` แล้วค่อย `COMMIT`
- **คำนวณจากค่าคอลัมน์เดิม**: ใช้ expression เช่น `price * 1.10`, `stock_qty - 3`, `GREATEST(...)` เพื่ออัปเดตแบบสัมพัทธ์
- **UPDATE ... FROM**: อัปเดตโดยอ้างอิงข้อมูลจากตารางอื่น เหมาะกับการซิงค์ข้อมูลสรุประหว่างตาราง
- **RETURNING**: ดูผลลัพธ์การเปลี่ยนแปลงได้ทันทีในคำสั่งเดียว ไม่ต้อง `SELECT` ซ้ำ และใช้ร่วมกับ CTE เพื่อบันทึก log ได้
- **Subquery ใน UPDATE**: ใช้ได้ทั้งใน `SET` (ต้องเป็น scalar subquery) และ `WHERE` (ระวังกับดัก `NOT IN` + `NULL`, แนะนำใช้ `NOT EXISTS` แทน)
- **CASE WHEN**: รวมตรรกะทางธุรกิจหลายเงื่อนไขไว้ในคำสั่ง `UPDATE` เดียว ลดจำนวนรอบการ scan ตาราง
- **Bulk update**: ตระหนักถึงผลกระทบด้าน MVCC, lock, WAL และ index maintenance เมื่อข้อมูลมีขนาดใหญ่ ควรแบ่งเป็น batch และ commit เป็นช่วง ๆ พร้อมเฝ้าระวังด้วย `pg_stat_activity` และเก็บกวาดด้วย `VACUUM ANALYZE` หลังอัปเดตเสร็จ

ทักษะเหล่านี้คือพื้นฐานสำคัญที่จะนำไปต่อยอดในบทถัดไปเรื่อง `DELETE` และ `TRUNCATE` ซึ่งมีหลักการเรื่องความปลอดภัยของข้อมูลที่คล้ายกัน — โดยเฉพาะอันตรายของการลืม `WHERE` clause ที่ยิ่งร้ายแรงกว่าเดิม เพราะ `DELETE` ที่ไม่มีเงื่อนไขจะลบข้อมูลทั้งตารางทิ้งไปเลย

---

## แบบฝึกหัด

จากสคีมาร้านกาแฟที่เตรียมไว้ในบทนี้ ให้เขียนคำสั่ง SQL ตอบโจทย์ต่อไปนี้

### แบบฝึกหัดที่ 1

เขียนคำสั่ง `UPDATE` เพื่อเปลี่ยนราคาของ "เอสเปรสโซ่" (product_id = 1) เป็น 60.00 บาท

<details>
<summary>เฉลย</summary>

```sql
UPDATE products
SET price = 60.00
WHERE product_id = 1;
```

ควรตรวจสอบก่อนว่ามีแถวไหนที่ `product_id = 1` จริง ด้วย `SELECT` ก่อนรัน `UPDATE` เสมอในสถานการณ์จริง

</details>

---

### แบบฝึกหัดที่ 2

เขียนคำสั่ง `UPDATE` เพื่อเปลี่ยนทั้งราคาและต้นทุน (cost) ของสินค้าที่ชื่อ "บราวนี่" ในคำสั่งเดียว โดยตั้งราคาเป็น 55.00 บาท และต้นทุนเป็น 22.00 บาท

<details>
<summary>เฉลย</summary>

```sql
UPDATE products
SET price = 55.00,
    cost  = 22.00
WHERE product_name = 'บราวนี่';
```

การอัปเดตหลายคอลัมน์ในคำสั่งเดียวใช้เครื่องหมายจุลภาคคั่นภายใน `SET` เดียว ไม่ต้องเขียน `SET` ซ้ำ

</details>

---

### แบบฝึกหัดที่ 3

ทดสอบคำสั่ง `UPDATE products SET stock_qty = 0;` (ไม่มี WHERE) โดยใช้ `BEGIN` และ `ROLLBACK` เพื่อดูผลลัพธ์อย่างปลอดภัยโดยไม่ทำให้ข้อมูลจริงเสียหาย

<details>
<summary>เฉลย</summary>

```sql
BEGIN;

UPDATE products SET stock_qty = 0;
-- ตรวจสอบผลลัพธ์
SELECT product_id, product_name, stock_qty FROM products ORDER BY product_id;

-- เมื่อเห็นว่าสต็อกทุกตัวกลายเป็น 0 หมด (ซึ่งไม่ใช่สิ่งที่ต้องการ) ให้ยกเลิก
ROLLBACK;
```

หลัง `ROLLBACK` ข้อมูล `stock_qty` จะกลับมาเป็นค่าเดิมทั้งหมด เพราะการเปลี่ยนแปลงยังไม่ถูก commit

</details>

---

### แบบฝึกหัดที่ 4

เขียนคำสั่ง `UPDATE` เพื่อเพิ่มสต็อก (stock_qty) ของสินค้าทุกตัวในหมวด "ชา" (category_id ที่ตรงกับ 'ชา') ขึ้นอีก 25 ชิ้นจากค่าปัจจุบัน

<details>
<summary>เฉลย</summary>

```sql
UPDATE products
SET stock_qty = stock_qty + 25
WHERE category_id = (
    SELECT category_id FROM categories WHERE category_name = 'ชา'
);
```

ตัวอย่างนี้ใช้ scalar subquery ใน `WHERE` เพื่อหา `category_id` จากชื่อหมวดหมู่ แทนที่จะ hardcode ตัวเลข ทำให้ query ยืดหยุ่นและอ่านง่ายขึ้น

</details>

---

### แบบฝึกหัดที่ 5

เขียนคำสั่ง `UPDATE ... RETURNING` เพื่อลดราคาสินค้าทุกตัวที่ `is_active = FALSE` ลง 20% แล้วให้แสดงผลลัพธ์ชื่อสินค้าและราคาที่เปลี่ยนแล้วออกมาทันที

<details>
<summary>เฉลย</summary>

```sql
UPDATE products
SET price = ROUND(price * 0.80, 2)
WHERE is_active = FALSE
RETURNING product_id, product_name, price;
```

`RETURNING` ทำให้เห็นผลลัพธ์ของการอัปเดตทันทีโดยไม่ต้องรัน `SELECT` แยกอีกคำสั่ง

</details>

---

### แบบฝึกหัดที่ 6

ใช้ `UPDATE ... FROM` เพื่ออัปเดตคอลัมน์ `total_amount` ของทุกแถวในตาราง `orders` ให้เท่ากับผลรวม (`quantity * unit_price`) ของรายการสินค้าที่เกี่ยวข้องในตาราง `order_items`

<details>
<summary>เฉลย</summary>

```sql
UPDATE orders o
SET total_amount = summary.calculated_total
FROM (
    SELECT order_id, SUM(quantity * unit_price) AS calculated_total
    FROM order_items
    GROUP BY order_id
) AS summary
WHERE o.order_id = summary.order_id;
```

ออเดอร์ที่ไม่มีรายการใน `order_items` เลยจะไม่ถูกแก้ไข (เพราะ subquery ไม่มีแถวที่ match) ซึ่งเป็นพฤติกรรมปกติของ `UPDATE ... FROM`

</details>

---

### แบบฝึกหัดที่ 7

เขียนคำสั่ง `UPDATE` โดยใช้ `CASE WHEN` เพื่อเปลี่ยนสถานะออเดอร์ (`status`) ตามเงื่อนไขต่อไปนี้ในคำสั่งเดียว: ถ้า `status = 'pending'` และมีอายุมากกว่า 2 วัน ให้เปลี่ยนเป็น `'expired'`, ถ้า `status = 'cancelled'` ให้คงเดิม, นอกนั้นให้คงค่าเดิมไว้เช่นกัน

<details>
<summary>เฉลย</summary>

```sql
UPDATE orders
SET status = CASE
    WHEN status = 'pending' AND order_date < NOW() - INTERVAL '2 days' THEN 'expired'
    ELSE status
END;
```

เนื่องจากเงื่อนไข "cancelled ให้คงเดิม" และ "นอกนั้นให้คงเดิม" มีผลลัพธ์เหมือนกันคือไม่เปลี่ยนแปลงค่า จึงรวมไว้ใน `ELSE status` ได้เลยโดยไม่ต้องเขียน `WHEN` แยก

</details>

---

### แบบฝึกหัดที่ 8

เขียนคำสั่งเพื่อค้นหาว่าสินค้าตัวใดที่ราคาต่ำกว่าต้นทุน (price < cost) ก่อน แล้วจึงเขียนคำสั่ง `UPDATE` เพื่อปรับราคาสินค้าที่ราคาต่ำกว่าต้นทุนให้เท่ากับ `cost * 1.3` (ให้มีกำไรขั้นต่ำ 30%)

<details>
<summary>เฉลย</summary>

```sql
-- ขั้นที่ 1: ตรวจสอบก่อน
SELECT product_id, product_name, price, cost
FROM products
WHERE price < cost;

-- ขั้นที่ 2: อัปเดตด้วยเงื่อนไขเดียวกัน
UPDATE products
SET price = ROUND(cost * 1.3, 2)
WHERE price < cost
RETURNING product_id, product_name, price;
```

การ `SELECT` ก่อนด้วยเงื่อนไขเดียวกับที่จะใช้ใน `UPDATE` ช่วยให้มั่นใจได้ว่าจะแก้ไขแถวที่ถูกต้องก่อนลงมือจริง

</details>

---

### แบบฝึกหัดที่ 9

สมมติว่าตาราง `order_items` มีข้อมูลจำนวนมาก (หลายล้านแถว) และต้องการปรับ `unit_price` เพิ่มขึ้น 3% ทุกแถว จงอธิบายแนวทาง (ไม่ต้องรันจริง) ว่าจะออกแบบการอัปเดตอย่างไรเพื่อไม่ให้กระทบระบบ production พร้อมยกตัวอย่างคำสั่ง batch หนึ่งรอบ

<details>
<summary>เฉลย</summary>

แนวทาง:
1. แบ่งการอัปเดตเป็น batch ย่อย ๆ (เช่น 5,000-10,000 แถวต่อรอบ) แทนการรันทีเดียวทั้งตาราง
2. ใช้ช่วง primary key (`order_item_id BETWEEN ... AND ...`) หรือ `LIMIT` ร่วมกับเงื่อนไขติดตามว่าแถวไหนอัปเดตแล้ว
3. `COMMIT` หลังจบแต่ละ batch เพื่อลดขนาด transaction และปลดล็อกแถวให้ transaction อื่นทำงานได้
4. รันในช่วง traffic ต่ำ และเฝ้าดู `pg_stat_activity` ระหว่างรัน
5. รัน `VACUUM ANALYZE order_items;` หลังอัปเดตครบทุก batch เพื่อเก็บกวาด dead tuples

ตัวอย่างคำสั่งหนึ่งรอบ (batch แรก):

```sql
BEGIN;

UPDATE order_items
SET unit_price = ROUND(unit_price * 1.03, 2)
WHERE order_item_id BETWEEN 1 AND 5000;

COMMIT;
-- รอสักครู่ แล้วรันรอบถัดไปด้วยช่วง ID ถัดไป (5001 ถึง 10000) จนครบทุกแถว
```

</details>

---

### แบบฝึกหัดที่ 10

เขียนคำสั่ง `UPDATE` เดียวที่ทำสามอย่างพร้อมกันสำหรับสถานการณ์ร้านกาแฟ: (1) ปรับราคาสินค้าในหมวด "กาแฟเย็น" ขึ้น 5% (2) ปัดเศษราคาให้เหลือ 2 ตำแหน่งทศนิยม (3) ใช้ `RETURNING` เพื่อแสดงชื่อสินค้า ราคาเดิม และราคาใหม่

<details>
<summary>เฉลย</summary>

```sql
UPDATE products
SET price = ROUND(price * 1.05, 2)
WHERE category_id = (
    SELECT category_id FROM categories WHERE category_name = 'กาแฟเย็น'
)
RETURNING
    product_name,
    ROUND(price / 1.05, 2) AS old_price,
    price AS new_price;
```

คำสั่งนี้รวมทุกเทคนิคที่เรียนมาในบทนี้ไว้ด้วยกัน: subquery ใน `WHERE`, expression คำนวณจากค่าคอลัมน์เดิมใน `SET`, และ `RETURNING` เพื่อดูผลลัพธ์ก่อน-หลังในคำสั่งเดียว

</details>

---

**บทถัดไป**: [Part 014: DELETE และ TRUNCATE — ลบข้อมูลอย่างปลอดภัย](./part-014-delete-truncate.md)
