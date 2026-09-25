# Part 018: Check Constraint, Default Value, Not Null

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับพื้นฐาน | Part 018

---

## เป้าหมายการเรียนรู้

เมื่อเรียนจบบทนี้ ผู้เรียนจะสามารถ:

- ใช้ `NOT NULL` เพื่อบังคับให้คอลัมน์ต้องมีค่าเสมอ และเข้าใจว่าทำไมควรใช้ `NOT NULL` ให้มากที่สุดเท่าที่ตรรกะทางธุรกิจอนุญาต
- กำหนด `DEFAULT` ให้คอลัมน์ ทั้งค่าคงที่และค่าที่มาจากฟังก์ชัน เช่น `now()`, `gen_random_uuid()`
- เขียน `CHECK` constraint ทั้งระดับคอลัมน์ (column-level) และระดับตาราง (table-level)
- เข้าใจพฤติกรรมของ `CHECK` เมื่อผลลัพธ์เป็น `NULL`/`UNKNOWN` ว่าทำไมจึง "ผ่าน" การตรวจสอบ
- ตั้งชื่อ constraint ด้วย `CONSTRAINT` keyword เพื่อให้ error message อ่านง่ายและดูแลรักษาง่าย
- เพิ่ม ลบ และตรวจสอบ constraint บนตารางที่มีข้อมูลอยู่แล้วด้วย `ALTER TABLE` รวมถึงเทคนิค `NOT VALID` + `VALIDATE CONSTRAINT` เพื่อลด downtime
- เข้าใจแนวคิดพื้นฐานของ `EXCLUSION constraint` สำหรับป้องกันข้อมูลที่ทับซ้อนกัน (overlapping) เช่น การจองห้อง/โต๊ะ
- อ่านและตีความ error message ที่ซับซ้อนเมื่อ constraint หลายตัวทำงานร่วมกัน
- ออกแบบชุด constraint ที่เหมาะสมให้กับระบบร้านกาแฟทั้งระบบ

---

## เตรียมข้อมูล

บทนี้ต่อเนื่องจากสคีมา "ร้านกาแฟ" (coffee shop) ที่เราใช้มาตลอดหลักสูตร ก่อนเริ่ม ให้สร้างฐานข้อมูลและตารางพื้นฐานขึ้นมาใหม่ (หรือใช้ฐานข้อมูลเดิมที่มีอยู่แล้วก็ได้) เพื่อให้ทุกตัวอย่างในบทนี้รันได้จริงตั้งแต่ต้นจนจบ

```sql
-- สร้างฐานข้อมูล (รันครั้งเดียว หากยังไม่มี)
-- CREATE DATABASE coffee_shop;
-- \c coffee_shop

-- ล้างตารางเดิม (ถ้ามี) เพื่อเริ่มต้นใหม่ให้สะอาด
DROP TABLE IF EXISTS order_items CASCADE;
DROP TABLE IF EXISTS orders CASCADE;
DROP TABLE IF EXISTS products CASCADE;
DROP TABLE IF EXISTS categories CASCADE;
DROP TABLE IF EXISTS employees CASCADE;
DROP TABLE IF EXISTS customers CASCADE;

-- หมวดหมู่สินค้า เช่น กาแฟ, ชา, เบเกอรี่
CREATE TABLE categories (
    category_id   SERIAL PRIMARY KEY,
    category_name VARCHAR(50) NOT NULL UNIQUE
);

-- เมนูสินค้า
CREATE TABLE products (
    product_id    SERIAL PRIMARY KEY,
    product_name  VARCHAR(100) NOT NULL,
    category_id   INTEGER REFERENCES categories(category_id),
    price         NUMERIC(10, 2) NOT NULL,
    is_active     BOOLEAN NOT NULL DEFAULT true
);

-- ลูกค้า
CREATE TABLE customers (
    customer_id   SERIAL PRIMARY KEY,
    customer_name VARCHAR(100) NOT NULL,
    email         VARCHAR(150),
    phone         VARCHAR(20),
    member_since  DATE
);

-- พนักงาน
CREATE TABLE employees (
    employee_id   SERIAL PRIMARY KEY,
    employee_name VARCHAR(100) NOT NULL,
    position      VARCHAR(50),
    hourly_wage   NUMERIC(10, 2),
    hire_date     DATE
);

-- คำสั่งซื้อ (order)
CREATE TABLE orders (
    order_id      SERIAL PRIMARY KEY,
    customer_id   INTEGER REFERENCES customers(customer_id),
    employee_id   INTEGER REFERENCES employees(employee_id),
    order_date    TIMESTAMP,
    status        VARCHAR(20),
    total_amount  NUMERIC(10, 2)
);

-- รายการสินค้าในแต่ละคำสั่งซื้อ
CREATE TABLE order_items (
    order_item_id SERIAL PRIMARY KEY,
    order_id      INTEGER REFERENCES orders(order_id),
    product_id    INTEGER REFERENCES products(product_id),
    quantity      INTEGER,
    unit_price    NUMERIC(10, 2)
);

-- ข้อมูลตัวอย่างเบื้องต้น
INSERT INTO categories (category_name) VALUES
    ('กาแฟ'), ('ชา'), ('เบเกอรี่'), ('เครื่องดื่มปั่น');

INSERT INTO products (product_name, category_id, price) VALUES
    ('Espresso', 1, 55.00),
    ('Latte', 1, 65.00),
    ('Thai Tea', 2, 45.00),
    ('Croissant', 3, 60.00);

INSERT INTO customers (customer_name, email, phone, member_since) VALUES
    ('สมชาย ใจดี', 'somchai@example.com', '0812345678', '2024-01-15'),
    ('มานี รักเรียน', 'manee@example.com', '0898765432', '2024-03-20');

INSERT INTO employees (employee_name, position, hourly_wage, hire_date) VALUES
    ('บาริสต้า เอ', 'Barista', 45.00, '2023-06-01'),
    ('บาริสต้า บี', 'Barista', 45.00, '2024-02-10');
```

สังเกตว่าสคีมานี้ยัง "หลวม" อยู่มาก เช่น `price` ยังไม่ได้บังคับว่าต้องเป็นบวก, `status` ยังเป็น `VARCHAR` ที่ใส่อะไรก็ได้, `email`/`phone` ยังไม่บังคับว่าต้องมีค่า นี่คือจุดที่เนื้อหาบทนี้จะเข้ามาช่วยทำให้สคีมา "แน่นหนา" (robust) ขึ้นทีละขั้น

---

## Step 171: NOT NULL constraint

### ทำไมต้องมี NOT NULL

ค่า `NULL` ใน PostgreSQL แปลว่า "ไม่ทราบค่า" (unknown) ไม่ใช่ "ค่าว่าง" หรือ "ศูนย์" การปล่อยให้คอลัมน์เป็น `NULL` ได้อย่างอิสระมักนำไปสู่ปัญหาที่ตามแก้ยากในภายหลัง เช่น การคำนวณผลรวมที่ขาดหาย การ JOIN ที่ไม่แมตช์ตามที่คาด หรือรายงานที่ต้องคอยเขียน `COALESCE`/`IS NULL` ทุกจุดที่ใช้งาน

หลักการที่ดีคือ **ใช้ `NOT NULL` ให้มากที่สุดเท่าที่ตรรกะทางธุรกิจจะอนุญาต** แล้วค่อยเปิดให้เป็น `NULL` ได้เฉพาะกรณีที่ "ไม่มีค่าจริง ๆ" เป็นความหมายทางธุรกิจที่ถูกต้อง (เช่น `phone` ของลูกค้าบางคนอาจไม่ให้ไว้จริง ๆ)

### Syntax พื้นฐาน

```sql
-- กำหนดตอนสร้างตาราง (column-level)
CREATE TABLE example_table (
    id   SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL
);

-- เพิ่มทีหลังด้วย ALTER TABLE
ALTER TABLE example_table
    ALTER COLUMN name SET NOT NULL;

-- ยกเลิก NOT NULL
ALTER TABLE example_table
    ALTER COLUMN name DROP NOT NULL;
```

### นำไปใช้กับสคีมาร้านกาแฟ

พิจารณาตาราง `customers` — `customer_name` ควรบังคับว่าต้องมีค่าเสมอ (สร้างไว้แล้วตั้งแต่ต้น) แต่ `order_date` ในตาราง `orders` ก็ควรบังคับเช่นกัน เพราะออร์เดอร์ทุกใบต้องมีเวลาที่สั่ง:

```sql
ALTER TABLE orders
    ALTER COLUMN order_date SET NOT NULL;
```

ทดลอง insert แถวที่ไม่มี `order_date`:

```sql
INSERT INTO orders (customer_id, employee_id, status, total_amount)
VALUES (1, 1, 'pending', 120.00);
```

ผลลัพธ์:

```text
ERROR:  null value in column "order_date" of relation "orders" violates not-null constraint
DETAIL:  Failing row contains (1, 1, 1, null, pending, 120.00).
```

ข้อควรระวัง: การเพิ่ม `NOT NULL` บนตารางที่มีข้อมูลอยู่แล้วและมี `NULL` ค้างอยู่ในคอลัมน์นั้น จะทำให้คำสั่ง `ALTER TABLE` ล้มเหลวทันที ต้องอัปเดตข้อมูลให้ครบก่อน:

```sql
-- ตรวจสอบก่อนว่ามีแถวไหนเป็น NULL หรือไม่
SELECT order_id FROM orders WHERE order_date IS NULL;

-- อัปเดตให้มีค่าก่อน แล้วค่อยเพิ่ม NOT NULL
UPDATE orders SET order_date = now() WHERE order_date IS NULL;
```

### ตรวจสอบว่าคอลัมน์ใดเป็น NOT NULL บ้าง

```sql
SELECT column_name, is_nullable, data_type
FROM information_schema.columns
WHERE table_name = 'orders'
ORDER BY ordinal_position;
```

ผลลัพธ์ตัวอย่าง:

```text
 column_name  | is_nullable | data_type
--------------+-------------+-----------
 order_id     | NO          | integer
 customer_id  | YES         | integer
 employee_id  | YES         | integer
 order_date   | NO          | timestamp without time zone
 status       | YES         | character varying
 total_amount | YES         | numeric
```

หรือใช้คำสั่ง `\d orders` ใน `psql` ซึ่งจะแสดงคอลัมน์ที่มี `not null` กำกับไว้ในบรรทัดนั้นโดยตรง — สะดวกกว่าเมื่อทำงานหน้าจอ terminal

---

## Step 172: DEFAULT value

### ค่าคงที่ (constant default)

`DEFAULT` คือค่าที่ PostgreSQL จะใส่ให้อัตโนมัติเมื่อคำสั่ง `INSERT` ไม่ได้ระบุค่าสำหรับคอลัมน์นั้น

```sql
ALTER TABLE products
    ALTER COLUMN is_active SET DEFAULT true;
```

จริง ๆ แล้วเราตั้ง `DEFAULT true` ไว้ตั้งแต่ตอนสร้างตารางแล้ว ลองเพิ่ม default ให้ `orders.status` ด้วย:

```sql
ALTER TABLE orders
    ALTER COLUMN status SET DEFAULT 'pending';
```

ทดสอบ:

```sql
INSERT INTO orders (customer_id, employee_id, order_date, total_amount)
VALUES (1, 1, now(), 65.00)
RETURNING order_id, status;
```

ผลลัพธ์:

```text
 order_id | status
----------+---------
        3 | pending
(1 row)
```

### ค่าจากฟังก์ชัน (function default)

`DEFAULT` ไม่จำเป็นต้องเป็นค่าคงที่เท่านั้น สามารถเป็นผลลัพธ์จากฟังก์ชันได้ เช่น เวลาปัจจุบัน หรือ UUID แบบสุ่ม

**`now()` — เวลาปัจจุบัน**

```sql
ALTER TABLE orders
    ALTER COLUMN order_date SET DEFAULT now();
```

ตอนนี้ถ้า insert โดยไม่ระบุ `order_date` ระบบจะใส่เวลาปัจจุบันให้อัตโนมัติ:

```sql
INSERT INTO orders (customer_id, employee_id, total_amount)
VALUES (2, 2, 45.00)
RETURNING order_id, order_date, status;
```

ผลลัพธ์:

```text
 order_id |         order_date         | status
----------+-----------------------------+---------
        4 | 2026-09-25 09:12:03.481022 | pending
(1 row)
```

**`gen_random_uuid()` — สร้างค่า UUID แบบสุ่ม**

ตั้งแต่ PostgreSQL 13 เป็นต้นไป ฟังก์ชัน `gen_random_uuid()` ถูกรวมเข้ามาใน core แล้ว (ไม่ต้อง `CREATE EXTENSION pgcrypto` เหมือนเวอร์ชันเก่า) จึงใช้งานได้ทันทีใน PostgreSQL 16/17

ตัวอย่าง: สมมติต้องการเพิ่มคอลัมน์ `order_uuid` เป็นรหัสอ้างอิงสาธารณะของออร์เดอร์ (ไม่อยากเปิดเผย `order_id` ตัวเลขที่เดาลำดับได้ง่าย):

```sql
ALTER TABLE orders
    ADD COLUMN order_uuid UUID NOT NULL DEFAULT gen_random_uuid();

SELECT order_id, order_uuid FROM orders LIMIT 3;
```

ผลลัพธ์ตัวอย่าง:

```text
 order_id |              order_uuid
----------+--------------------------------------
        1 | 3f29a1d2-9b4e-4c11-8f0a-1122334455aa
        2 | 7c6e0b3a-2211-4a90-bd77-99aabbccddee
        3 | 0a1b2c3d-4e5f-4071-9a8b-7c6d5e4f3a21
(3 rows)
```

> หมายเหตุ: การเพิ่มคอลัมน์ที่มีทั้ง `NOT NULL` และ `DEFAULT` พร้อมกันแบบนี้ใน PostgreSQL 11+ จะไม่ rewrite ทั้งตารางถ้า default เป็นค่าคงที่ (constant) แต่ถ้า default เป็นฟังก์ชันที่ให้ค่าไม่คงที่ (volatile function เช่น `gen_random_uuid()`, `now()`) PostgreSQL จะต้อง rewrite ตารางทั้งหมดเพื่อคำนวณค่าให้ทุกแถวที่มีอยู่เดิม ซึ่งอาจใช้เวลานานและล็อกตารางถ้าตารางมีข้อมูลจำนวนมาก — เรื่องนี้จะกล่าวถึงรายละเอียดเพิ่มเติมใน Part 019

### DEFAULT กับ expression ทั่วไป

`DEFAULT` รองรับ expression ได้หลากหลาย ไม่จำกัดแค่ค่าคงที่หรือฟังก์ชันเดี่ยว ๆ เช่น:

```sql
-- ตัวอย่าง: สร้างตาราง promotions ที่มีวันหมดอายุ default เป็น 30 วันจากวันนี้
CREATE TABLE promotions (
    promotion_id   SERIAL PRIMARY KEY,
    promotion_name VARCHAR(100) NOT NULL,
    start_date     DATE NOT NULL DEFAULT CURRENT_DATE,
    end_date       DATE NOT NULL DEFAULT CURRENT_DATE + INTERVAL '30 days'
);

INSERT INTO promotions (promotion_name) VALUES ('ลด 10% เครื่องดื่มร้อน')
RETURNING *;
```

ผลลัพธ์ตัวอย่าง (รันวันที่ 25 กันยายน 2026):

```text
 promotion_id |     promotion_name      | start_date |          end_date
--------------+--------------------------+------------+-----------------------------
            1 | ลด 10% เครื่องดื่มร้อน  | 2026-09-25 | 2026-10-25 00:00:00
(1 row)
```

> สังเกตว่า `end_date` ควรเป็น `DATE` แต่ผลลัพธ์จาก `CURRENT_DATE + INTERVAL '30 days'` เป็น `timestamp` PostgreSQL จะแปลงให้อัตโนมัติ (cast) ให้ตรงกับชนิดข้อมูลของคอลัมน์ปลายทางเสมอ

---

## Step 173: CHECK constraint พื้นฐาน (column-level)

`CHECK` constraint คือเงื่อนไขบูลีน (boolean expression) ที่ PostgreSQL จะตรวจสอบทุกครั้งที่มีการ `INSERT` หรือ `UPDATE` แถวนั้น ถ้าผลลัพธ์เป็น `FALSE` คำสั่งจะถูกปฏิเสธทันที

### Syntax

```sql
-- ระดับคอลัมน์: เขียนต่อท้ายนิยามคอลัมน์
column_name data_type CHECK (condition)
```

### ตัวอย่าง: ราคาสินค้าต้องเป็นบวก

ตาราง `products` ตอนนี้ยังยอมให้ `price` เป็น 0 หรือติดลบได้ ซึ่งไม่สมเหตุสมผลทางธุรกิจ เพิ่ม `CHECK` เข้าไป:

```sql
ALTER TABLE products
    ADD CHECK (price > 0);
```

ทดสอบ:

```sql
INSERT INTO products (product_name, category_id, price)
VALUES ('Mystery Drink', 1, -20.00);
```

ผลลัพธ์:

```text
ERROR:  new row for relation "products" violates check constraint "products_price_check"
DETAIL:  Failing row contains (5, Mystery Drink, 1, -20.00, t).
```

สังเกตว่า PostgreSQL ตั้งชื่อ constraint ให้อัตโนมัติเป็น `products_price_check` (รูปแบบ `<table>_<column>_check`) — เรื่องการตั้งชื่อเองจะพูดถึงใน Step 176

### ตัวอย่างเพิ่มเติม: จำกัดค่าใน column ระดับคอลัมน์ตอนสร้างตาราง

```sql
CREATE TABLE order_items_v2 (
    order_item_id SERIAL PRIMARY KEY,
    order_id      INTEGER NOT NULL REFERENCES orders(order_id),
    product_id    INTEGER NOT NULL REFERENCES products(product_id),
    quantity      INTEGER NOT NULL CHECK (quantity > 0),
    unit_price    NUMERIC(10, 2) NOT NULL CHECK (unit_price >= 0)
);
```

`CHECK` สามารถอ้างอิงได้เฉพาะคอลัมน์ **ภายในแถวเดียวกัน** เท่านั้น ไม่สามารถอ้างอิงแถวอื่น หรือทำ subquery ข้ามตารางได้ (ถ้าต้องการเงื่อนไขแบบนั้น ต้องใช้ trigger แทน ซึ่งจะกล่าวถึงในบทถัดไปเกี่ยวกับ Advanced Constraints)

---

## Step 174: CHECK constraint ระดับตาราง (table-level)

เมื่อเงื่อนไขเกี่ยวข้องกับ **มากกว่าหนึ่งคอลัมน์** ในแถวเดียวกัน เราจะเขียน `CHECK` แบบ table-level คือแยกออกมาเป็นบรรทัดของตัวเองในนิยามตาราง (ไม่ผูกติดกับคอลัมน์ใดคอลัมน์หนึ่ง)

### Syntax

```sql
CREATE TABLE example (
    col1 ...,
    col2 ...,
    CHECK (col1 < col2)
);
```

### ตัวอย่าง: promotions — start_date ต้องมาก่อน end_date เสมอ

```sql
ALTER TABLE promotions
    ADD CONSTRAINT promotions_date_range_check
    CHECK (start_date < end_date);
```

ทดสอบ:

```sql
INSERT INTO promotions (promotion_name, start_date, end_date)
VALUES ('โปรโมชันผิดพลาด', '2026-11-01', '2026-10-01');
```

ผลลัพธ์:

```text
ERROR:  new row for relation "promotions" violates check constraint "promotions_date_range_check"
DETAIL:  Failing row contains (2, โปรโมชันผิดพลาด, 2026-11-01, 2026-10-01).
```

### ตัวอย่าง: order_items — ราคาที่บันทึกไว้ต้องสอดคล้องกับส่วนลดสูงสุดที่กำหนด

สมมติเพิ่มคอลัมน์ `discount_amount` และต้องการบังคับว่าส่วนลดต้องไม่เกินราคาต่อหน่วย:

```sql
ALTER TABLE order_items
    ADD COLUMN discount_amount NUMERIC(10, 2) NOT NULL DEFAULT 0;

ALTER TABLE order_items
    ADD CONSTRAINT order_items_discount_check
    CHECK (discount_amount >= 0 AND discount_amount <= unit_price);
```

`CHECK` แบบ table-level เขียนเงื่อนไขที่ซับซ้อนได้อย่างอิสระ รวมถึงใช้ `AND` / `OR` / ฟังก์ชัน / operator ต่าง ๆ ได้ตราบเท่าที่ผลลัพธ์สุดท้ายเป็นบูลีนและอ้างอิงเฉพาะคอลัมน์ในแถวเดียวกัน

---

## Step 175: CHECK constraint กับ NULL

นี่คือจุดที่มือใหม่มักเข้าใจผิดบ่อยที่สุด: **`CHECK` constraint จะ "ผ่าน" (ไม่ error) เมื่อผลลัพธ์ของเงื่อนไขเป็น `NULL`/`UNKNOWN`** ไม่ใช่แค่ตอนที่ผลลัพธ์เป็น `TRUE` เท่านั้น

PostgreSQL ปฏิเสธแถวก็ต่อเมื่อผลลัพธ์ของ `CHECK` เป็น `FALSE` เท่านั้น ถ้าผลลัพธ์เป็น `TRUE` หรือ `NULL` แถวนั้นจะถูกยอมรับทั้งคู่

### ตัวอย่างสาธิต

```sql
CREATE TABLE demo_check_null (
    id    INTEGER PRIMARY KEY,
    score INTEGER CHECK (score > 0)
);

-- กรณีที่ 1: score = 10 -> (10 > 0) = TRUE -> ผ่าน
INSERT INTO demo_check_null VALUES (1, 10);

-- กรณีที่ 2: score = -5 -> (-5 > 0) = FALSE -> ถูกปฏิเสธ
INSERT INTO demo_check_null VALUES (2, -5);

-- กรณีที่ 3: score = NULL -> (NULL > 0) = NULL (UNKNOWN) -> ผ่าน!
INSERT INTO demo_check_null VALUES (3, NULL);
```

ผลลัพธ์:

```text
INSERT 0 1
ERROR:  new row for relation "demo_check_null" violates check constraint "demo_check_null_score_check"
DETAIL:  Failing row contains (2, -5).
INSERT 0 1
```

แถวที่ 3 (`score = NULL`) ถูก insert สำเร็จ แม้จะดูเหมือนว่า "ไม่มีค่าที่มากกว่า 0" ก็ตาม เพราะ `NULL > 0` ไม่ได้ประเมินว่าเป็น `FALSE` แต่เป็น `UNKNOWN` ซึ่งตามกฎ SQL มาตรฐาน ระบบจะถือว่า **ยังไม่มีหลักฐานว่าเงื่อนไขนี้ล้มเหลว** จึงอนุญาตให้ผ่าน

```sql
SELECT * FROM demo_check_null;
```

```text
 id | score
----+-------
  1 |    10
  3 |
(2 rows)
```

### ข้อแนะนำในทางปฏิบัติ

ถ้าต้องการบังคับว่าคอลัมน์ต้อง "มีค่าเสมอ และต้องเป็นค่าที่ถูกต้อง" ต้องใช้ `NOT NULL` **ควบคู่กับ** `CHECK` เสมอ อย่าพึ่งพา `CHECK` เพียงอย่างเดียวเพื่อป้องกัน `NULL`:

```sql
DROP TABLE demo_check_null;

CREATE TABLE demo_check_null (
    id    INTEGER PRIMARY KEY,
    score INTEGER NOT NULL CHECK (score > 0)
);

INSERT INTO demo_check_null VALUES (1, NULL);
```

ผลลัพธ์:

```text
ERROR:  null value in column "score" of relation "demo_check_null" violates not-null constraint
DETAIL:  Failing row contains (1, null).
```

คราวนี้ `NOT NULL` จะดักจับก่อนที่ `CHECK` จะมีโอกาสตัดสินใจด้วยซ้ำ — นี่คือเหตุผลสำคัญอีกข้อที่สนับสนุนหลักการใน Step 171: **ใช้ `NOT NULL` ให้มากที่สุดเท่าที่เป็นไปได้** เพราะ `CHECK` เพียงอย่างเดียวไม่สามารถป้องกัน `NULL` ได้

```sql
DROP TABLE demo_check_null;
```

---

## Step 176: การตั้งชื่อ constraint ด้วย CONSTRAINT keyword

ถ้าไม่ตั้งชื่อ constraint เอง PostgreSQL จะสร้างชื่ออัตโนมัติตามรูปแบบ `<table>_<column>_<type>` เช่น `products_price_check` หรือ `orders_customer_id_fkey` ซึ่งใช้งานได้ในระบบเล็ก ๆ แต่เมื่อระบบขยายใหญ่ขึ้น การตั้งชื่อเองจะมีประโยชน์มาก:

1. **Error message อ่านง่ายขึ้น** — ชื่อที่สื่อความหมาย เช่น `chk_price_positive` ดีกว่า `products_price_check1` (กรณีมี CHECK มากกว่าหนึ่งตัวในคอลัมน์เดียวกัน ชื่ออัตโนมัติจะชนกันและ PostgreSQL ต้องเติมตัวเลขต่อท้าย)
2. **จัดการ (DROP/ALTER) ได้ง่าย** — ไม่ต้องเดาชื่อจาก system catalog
3. **สื่อสารกับทีมได้ชัดเจน** — ชื่อ constraint กลายเป็นเอกสารในตัวมันเอง (self-documenting)

### Syntax

```sql
column_name data_type CONSTRAINT constraint_name CHECK (condition)

-- หรือระดับตาราง
CONSTRAINT constraint_name CHECK (condition)
```

### ตัวอย่าง: ตั้งชื่อ constraint ใหม่ทั้งหมดให้สื่อความหมาย

ลบ constraint อัตโนมัติที่ตั้งไว้ก่อนหน้า แล้วเพิ่มใหม่พร้อมชื่อที่ตั้งใจ:

```sql
-- ดูชื่อ constraint ปัจจุบันของ products
SELECT conname, pg_get_constraintdef(oid)
FROM pg_constraint
WHERE conrelid = 'products'::regclass;
```

ผลลัพธ์ตัวอย่าง:

```text
       conname        |         pg_get_constraintdef
-----------------------+----------------------------------------
 products_pkey         | PRIMARY KEY (product_id)
 products_category_id_fkey | FOREIGN KEY (category_id) REFERENCES categories(category_id)
 products_price_check  | CHECK (price > 0)
```

เปลี่ยนชื่อให้สื่อความหมายมากขึ้น:

```sql
ALTER TABLE products
    DROP CONSTRAINT products_price_check;

ALTER TABLE products
    ADD CONSTRAINT chk_products_price_positive CHECK (price > 0);
```

ทดสอบ error message ใหม่:

```sql
INSERT INTO products (product_name, category_id, price)
VALUES ('Free Sample', 1, 0);
```

ผลลัพธ์:

```text
ERROR:  new row for relation "products" violates check constraint "chk_products_price_positive"
DETAIL:  Failing row contains (6, Free Sample, 1, 0.00, t).
```

ชื่อ `chk_products_price_positive` สื่อความหมายชัดเจนกว่า `products_price_check1` มาก โดยเฉพาะเมื่อมี log จำนวนมากที่ต้องไล่ตรวจสอบ

### ข้อตกลงการตั้งชื่อ (naming convention) ที่นิยมใช้

หลายทีมใช้ prefix เพื่อแยกประเภท constraint ให้ดูออกทันทีจากชื่อ:

| Prefix | ประเภท |
|---|---|
| `pk_` | Primary Key |
| `fk_` | Foreign Key |
| `uq_` หรือ `uk_` | Unique |
| `chk_` | Check |
| `ex_` | Exclusion |

ตัวอย่างการตั้งชื่อครบชุดสำหรับตาราง `promotions`:

```sql
ALTER TABLE promotions
    DROP CONSTRAINT promotions_date_range_check;

ALTER TABLE promotions
    ADD CONSTRAINT chk_promotions_date_range CHECK (start_date < end_date);
```

---

## Step 177: การเพิ่ม/ลบ/ปิดใช้งาน constraint ด้วย ALTER TABLE

### เพิ่ม constraint (ADD CONSTRAINT)

```sql
ALTER TABLE table_name
    ADD CONSTRAINT constraint_name CHECK (condition);
```

เมื่อรันคำสั่งนี้บนตารางที่มีข้อมูลอยู่แล้ว **PostgreSQL จะสแกนทั้งตารางทันทีเพื่อตรวจสอบว่าทุกแถวผ่านเงื่อนไขหรือไม่** และจะล็อกตาราง (`ACCESS EXCLUSIVE LOCK` ในช่วงสั้น ๆ ขณะตรวจสอบ) ซึ่งถ้าตารางมีข้อมูลจำนวนมากและระบบใช้งานจริง (production) อยู่ อาจทำให้แอปพลิเคชันหยุดชะงักได้

### ลบ constraint (DROP CONSTRAINT)

```sql
ALTER TABLE table_name
    DROP CONSTRAINT constraint_name;

-- ถ้าไม่แน่ใจว่ามี constraint นี้อยู่หรือไม่
ALTER TABLE table_name
    DROP CONSTRAINT IF EXISTS constraint_name;
```

### NOT VALID + VALIDATE CONSTRAINT: เทคนิคลด downtime

PostgreSQL ไม่มีคำสั่ง "DISABLE constraint" แบบฐานข้อมูลบางค่าย (เช่น Oracle) โดยตรงสำหรับ `CHECK` แต่มีเทคนิคที่ทรงพลังกว่าคือ `NOT VALID` ซึ่งช่วยแบ่งงานหนักออกเป็นสองขั้นตอน:

1. **เพิ่ม constraint ด้วย `NOT VALID`** — PostgreSQL จะ**ไม่**สแกนแถวเดิมเพื่อตรวจสอบ (เร็วมาก ใช้ล็อกช่วงสั้น) แต่จะเริ่มบังคับใช้กับแถวใหม่ที่ insert/update ตั้งแต่วินาทีนั้นเป็นต้นไปทันที
2. **ตรวจสอบย้อนหลังด้วย `VALIDATE CONSTRAINT`** — สแกนแถวเดิมทั้งหมดในภายหลัง (ช่วงเวลาที่สะดวก เช่น traffic ต่ำ) โดยใช้แค่ `SHARE UPDATE EXCLUSIVE LOCK` ซึ่งไม่บล็อก read/write ปกติ

```sql
-- ขั้นตอนที่ 1: เพิ่ม constraint แบบ NOT VALID (เร็ว ไม่บล็อกนาน)
ALTER TABLE order_items
    ADD CONSTRAINT chk_order_items_quantity_positive
    CHECK (quantity > 0) NOT VALID;
```

ผลลัพธ์:

```text
ALTER TABLE
```

ตรวจสอบสถานะว่า constraint ยัง "not valid" อยู่:

```sql
SELECT conname, convalidated
FROM pg_constraint
WHERE conrelid = 'order_items'::regclass
  AND conname = 'chk_order_items_quantity_positive';
```

```text
              conname               | convalidated
-------------------------------------+--------------
 chk_order_items_quantity_positive  | f
(1 row)
```

```sql
-- ขั้นตอนที่ 2: ตรวจสอบย้อนหลัง (ทำตอนที่ระบบว่างหรือ off-peak)
ALTER TABLE order_items
    VALIDATE CONSTRAINT chk_order_items_quantity_positive;
```

หลังจากรัน `VALIDATE CONSTRAINT` สำเร็จ `convalidated` จะกลายเป็น `t` (true) และระบบยืนยันว่าทุกแถวในตารางผ่านเงื่อนไขนี้แล้วจริง ๆ

> ข้อควรรู้: ในช่วงเวลาที่ constraint ยังเป็น `NOT VALID` (ก่อน validate) PostgreSQL **ยังคงบังคับใช้กับแถวใหม่ทุกแถว** อยู่แล้ว เพียงแต่ยังไม่รับประกันว่าแถวเก่าทั้งหมดผ่านเงื่อนไข ดังนั้นเทคนิคนี้จึงปลอดภัยสำหรับการ deploy บนระบบ production ที่ต้องการ zero-downtime

### ตรวจสอบรายการ constraint ทั้งหมดของตาราง

```sql
\d order_items
```

หรือใช้ query ตรง ๆ:

```sql
SELECT
    conname AS constraint_name,
    contype AS constraint_type,
    pg_get_constraintdef(oid) AS definition,
    convalidated
FROM pg_constraint
WHERE conrelid = 'order_items'::regclass
ORDER BY contype;
```

ผลลัพธ์ตัวอย่าง (คอลัมน์ `contype`: `p` = primary key, `f` = foreign key, `c` = check, `u` = unique, `x` = exclusion):

```text
          constraint_name           | constraint_type |                    definition                     | convalidated
-------------------------------------+-----------------+----------------------------------------------------+--------------
 order_items_pkey                   | p               | PRIMARY KEY (order_item_id)                        | t
 order_items_order_id_fkey          | f               | FOREIGN KEY (order_id) REFERENCES orders(order_id) | t
 order_items_product_id_fkey        | f               | FOREIGN KEY (product_id) REFERENCES products(product_id) | t
 chk_order_items_quantity_positive  | c               | CHECK (quantity > 0)                               | t
 order_items_discount_check         | c               | CHECK (discount_amount >= 0 AND discount_amount <= unit_price) | t
```

---

## Step 178: EXCLUSION constraint เบื้องต้น

### แนวคิด: ป้องกันข้อมูลที่ "ทับซ้อนกัน"

`UNIQUE` constraint ป้องกันค่าที่ "เท่ากันเป๊ะ" ซ้ำกัน แต่ในโลกจริงมีปัญหาอีกแบบที่พบบ่อยมาก คือการป้องกัน **ช่วง (range) ที่ทับซ้อนกัน** เช่น:

- โต๊ะเดียวกัน ห้ามมีการจอง (booking) สองรายการที่เวลาทับซ้อนกัน
- พนักงานคนเดียวกัน ห้ามมีกะทำงาน (shift) สองกะที่เวลาทับซ้อนกัน
- ห้องประชุมเดียวกัน ห้ามมีการจองซ้อนกัน

เงื่อนไขแบบนี้ `CHECK` และ `UNIQUE` ทำไม่ได้ เพราะต้องเปรียบเทียบ "แถวนี้กับแถวอื่น ๆ ทั้งหมดในตาราง" ไม่ใช่แค่ภายในแถวเดียว — นี่คือหน้าที่ของ **`EXCLUSION` constraint**

### หลักการทำงาน

`EXCLUSION` constraint ใช้ index แบบ **GiST** (Generalized Search Tree) เพื่อตรวจสอบว่า "ไม่มีแถวสองแถวใดที่ตรงตามเงื่อนไข operator ที่กำหนดพร้อมกันทุกคอลัมน์" พูดง่าย ๆ คือเป็น `UNIQUE` ที่ทรงพลังกว่า เพราะแทนที่จะเทียบด้วย `=` อย่างเดียว สามารถเทียบด้วย operator อื่นได้ด้วย เช่น `&&` (overlaps) สำหรับชนิดข้อมูล range

### ตัวอย่าง: ระบบจองโต๊ะร้านกาแฟ (table bookings)

สมมติร้านกาแฟเปิดให้จองโต๊ะล่วงหน้าได้ ต้องการป้องกันไม่ให้โต๊ะเดียวกันถูกจองซ้อนเวลากัน:

```sql
-- ต้อง enable extension btree_gist ก่อน เพื่อให้ column ชนิด integer
-- ใช้ equality operator (=) ร่วมกับ GiST index ได้
CREATE EXTENSION IF NOT EXISTS btree_gist;

CREATE TABLE table_bookings (
    booking_id   SERIAL PRIMARY KEY,
    table_number INTEGER NOT NULL CHECK (table_number > 0),
    customer_id  INTEGER REFERENCES customers(customer_id),
    during       TSRANGE NOT NULL,
    EXCLUDE USING gist (
        table_number WITH =,
        during WITH &&
    )
);
```

อธิบาย syntax:

- `table_number WITH =` — ให้ตรวจสอบเฉพาะแถวที่ `table_number` **เท่ากัน**
- `during WITH &&` — และช่วงเวลา `during` **ทับซ้อนกัน** (`&&` คือ range overlap operator)
- ถ้าทั้งสองเงื่อนไขเป็นจริงพร้อมกันในสองแถวใด ๆ constraint จะปฏิเสธการ insert/update

ทดสอบ:

```sql
INSERT INTO table_bookings (table_number, customer_id, during) VALUES
    (5, 1, '[2026-10-01 18:00, 2026-10-01 20:00)');

-- พยายามจองโต๊ะ 5 ซ้อนเวลาเดิม (ทับซ้อนบางส่วน)
INSERT INTO table_bookings (table_number, customer_id, during) VALUES
    (5, 2, '[2026-10-01 19:00, 2026-10-01 21:00)');
```

ผลลัพธ์:

```text
INSERT 0 1
ERROR:  conflicting key value violates exclusion constraint "table_bookings_table_number_during_excl"
DETAIL:  Key (table_number, during)=(5, ["2026-10-01 19:00:00","2026-10-01 21:00:00")) conflicts with existing key (table_number, during)=(5, ["2026-10-01 18:00:00","2026-10-01 20:00:00")).
```

แต่การจองโต๊ะ**คนละหมายเลข**ในเวลาเดียวกัน หรือจองโต๊ะเดียวกันใน**เวลาที่ไม่ทับซ้อน** ยังทำได้ตามปกติ:

```sql
-- คนละโต๊ะ เวลาเดียวกัน -> ผ่าน
INSERT INTO table_bookings (table_number, customer_id, during) VALUES
    (6, 2, '[2026-10-01 19:00, 2026-10-01 21:00)');

-- โต๊ะเดียวกัน แต่เวลาไม่ทับซ้อน -> ผ่าน
INSERT INTO table_bookings (table_number, customer_id, during) VALUES
    (5, 2, '[2026-10-01 20:00, 2026-10-01 22:00)');
```

```text
INSERT 0 1
INSERT 0 1
```

> เรื่อง `EXCLUSION` constraint ในเชิงลึก (การใช้ร่วมกับ range types แบบกำหนดเอง, `btree_gist` กับชนิดข้อมูลอื่น ๆ, performance ของ GiST index) จะถูกอธิบายอย่างละเอียดในบทที่ว่าด้วย Advanced Constraints & Indexing ในระดับ Intermediate ของหลักสูตรนี้ บทนี้เพียงแค่เกริ่นแนวคิดให้เห็นภาพว่ามันมีอยู่และใช้แก้ปัญหาอะไร

---

## Step 179: ลำดับการตรวจสอบ constraint และการอ่าน error message ที่ซับซ้อน

### ลำดับการตรวจสอบเมื่อ INSERT/UPDATE

เมื่อ PostgreSQL ประมวลผลคำสั่ง `INSERT` หรือ `UPDATE` หนึ่งแถว ลำดับการทำงานคร่าว ๆ คือ:

1. **คำนวณค่า `DEFAULT`** สำหรับคอลัมน์ที่ไม่ได้ระบุค่ามา
2. **ตรวจสอบ `NOT NULL`** ของทุกคอลัมน์ที่เกี่ยวข้อง
3. **ตรวจสอบ `CHECK` constraint** ทั้งระดับคอลัมน์และระดับตาราง (PostgreSQL ไม่รับประกันลำดับการตรวจสอบระหว่าง `CHECK` หลายตัว หากมีมากกว่าหนึ่งตัว — ตัวไหนพบว่าเป็น `FALSE` ก่อนจะรายงาน error ก่อน)
4. **ตรวจสอบ `UNIQUE`/`PRIMARY KEY`** (ผ่าน index)
5. **ตรวจสอบ `EXCLUSION`** constraint (ผ่าน GiST index เช่นกัน)
6. **ตรวจสอบ `FOREIGN KEY`** — โดยทั่วไปจะถูกตรวจสอบ**หลังสุด** และมักถูก defer ไปตรวจตอนจบ statement หรือจบ transaction ถ้า constraint นั้นถูกประกาศเป็น `DEFERRABLE`

สิ่งสำคัญที่ต้องจำ: **ห้ามพึ่งพาลำดับการตรวจสอบระหว่าง constraint ประเภทเดียวกัน** (เช่น `CHECK` สองตัว) เพราะ PostgreSQL ไม่รับประกันว่าจะตรวจตัวไหนก่อน แต่โดยรวมแล้ว `NOT NULL` มักจะถูกตรวจพบก่อน `CHECK` เสมอ เพราะเป็นการตรวจสอบที่เบากว่าและทำก่อนในขั้นตอนภายใน

### ตัวอย่าง: หลาย constraint ทำงานพร้อมกัน

```sql
CREATE TABLE demo_order_check (
    id       SERIAL PRIMARY KEY,
    quantity INTEGER NOT NULL CHECK (quantity > 0),
    price    NUMERIC NOT NULL CHECK (price > 0),
    total    NUMERIC NOT NULL,
    CONSTRAINT chk_total_matches CHECK (total = quantity * price)
);

-- ผิดพลาดหลายจุดพร้อมกัน: quantity ติดลบ และ total ไม่ตรงสูตร
INSERT INTO demo_order_check (quantity, price, total)
VALUES (-2, 50, 100);
```

ผลลัพธ์:

```text
ERROR:  new row for relation "demo_order_check" violates check constraint "demo_order_check_quantity_check"
DETAIL:  Failing row contains (1, -2, 50, 100).
```

PostgreSQL หยุดที่ constraint แรกที่ล้มเหลว (ในตัวอย่างนี้คือ `quantity_check`) และไม่รายงาน constraint อื่นที่ล้มเหลวด้วย (เช่น `chk_total_matches` ก็ล้มเหลวเช่นกันเพราะ `100 != -2 * 50`) — ดังนั้นเมื่อแก้ error หนึ่งจุดแล้ว รันซ้ำ อาจเจอ error จุดถัดไปอีก:

```sql
INSERT INTO demo_order_check (quantity, price, total)
VALUES (2, 50, 100);
```

รอบนี้ `quantity > 0` ผ่าน, `price > 0` ผ่าน, และ `100 = 2 * 50` ก็เป็นจริง จึง insert สำเร็จ:

```text
INSERT 0 1
```

```sql
DROP TABLE demo_order_check;
```

### วิธีอ่าน error message ให้ได้ข้อมูลครบ

Error message ของ PostgreSQL มีโครงสร้างที่มีประโยชน์มากกว่าที่หลายคนสังเกต ลองดูตัวอย่างจริงจากระบบร้านกาแฟ:

```sql
INSERT INTO order_items (order_id, product_id, quantity, unit_price)
VALUES (1, 999, 3, 65.00);
```

ผลลัพธ์:

```text
ERROR:  insert or update on table "order_items" violates foreign key constraint "order_items_product_id_fkey"
DETAIL:  Key (product_id)=(999) is not present in table "products".
```

แยกส่วนประกอบของ error:

| ส่วน | ความหมาย |
|---|---|
| `ERROR:` | ระดับความรุนแรง (severity) — statement นี้ถูกยกเลิกทั้งหมด |
| `insert or update on table "order_items"` | บอกว่าปัญหาเกิดตอนทำ operation อะไร บนตารางไหน |
| `violates foreign key constraint "order_items_product_id_fkey"` | บอกชื่อ constraint ที่ถูกละเมิด — นี่คือเหตุผลว่าทำไมการตั้งชื่อ constraint ให้สื่อความหมาย (Step 176) จึงสำคัญมาก เพราะชื่อนี้จะปรากฏใน log ทุกครั้ง |
| `DETAIL:` | ให้ข้อมูลเจาะลึกเพิ่มเติม บอกค่าจริงที่ทำให้ผิดพลาด และเหตุผลที่ชัดเจน |

เมื่อ error message มาจาก `CHECK` constraint ที่ซับซ้อน อาจจำเป็นต้อง query นิยามของ constraint นั้นเพิ่มเติมเพื่อเข้าใจเงื่อนไขทั้งหมด:

```sql
SELECT conname, pg_get_constraintdef(oid)
FROM pg_constraint
WHERE conname = 'order_items_discount_check';
```

```text
          conname           |                          pg_get_constraintdef
-----------------------------+-----------------------------------------------------------------------
 order_items_discount_check | CHECK (((discount_amount >= (0)::numeric) AND (discount_amount <= unit_price)))
```

การเปิดดูนิยาม constraint แบบนี้ ช่วยให้เข้าใจได้ทันทีว่าทำไมแถวที่พยายาม insert ถึงถูกปฏิเสธ โดยไม่ต้องไปงมหาในโค้ดแอปพลิเคชัน

---

## Step 180: แบบฝึกหัดรวม — เพิ่ม constraint ที่เหมาะสมให้ระบบร้านกาแฟทั้งระบบ

ตอนนี้มาถึงเวลารวบรวมทุกสิ่งที่เรียนมาในบทนี้ เพื่อทำให้สคีมาร้านกาแฟ "แน่นหนา" อย่างแท้จริง ลองไล่ทีละตารางตามหลักการ:

1. คอลัมน์ที่ทางธุรกิจต้องมีค่าเสมอ ให้ใส่ `NOT NULL`
2. ค่าที่ควรมีค่าตั้งต้นตามธรรมชาติ ให้ใส่ `DEFAULT`
3. ค่าตัวเลขที่ต้องเป็นบวกหรืออยู่ในช่วงที่กำหนด ให้ใส่ `CHECK`
4. สถานะ (status) ที่ควรอยู่ในชุดค่าที่กำหนดตายตัว ให้ใส่ `CHECK ... IN (...)`
5. ตั้งชื่อทุก constraint ให้สื่อความหมาย

```sql
-- ================================
-- 1) categories
-- ================================
-- category_name เป็น NOT NULL UNIQUE อยู่แล้วตั้งแต่ต้น ถือว่าเพียงพอ

-- ================================
-- 2) products
-- ================================
ALTER TABLE products
    ALTER COLUMN category_id SET NOT NULL;

ALTER TABLE products
    ADD CONSTRAINT chk_products_price_not_absurd
    CHECK (price < 100000);  -- กันการกรอกราคาผิดพลาดแบบสุดโต่ง

-- ================================
-- 3) customers
-- ================================
ALTER TABLE customers
    ADD CONSTRAINT chk_customers_email_format
    CHECK (email IS NULL OR email LIKE '%_@_%._%');

ALTER TABLE customers
    ADD CONSTRAINT chk_customers_member_since_not_future
    CHECK (member_since IS NULL OR member_since <= CURRENT_DATE);

-- ================================
-- 4) employees
-- ================================
ALTER TABLE employees
    ALTER COLUMN position SET NOT NULL;

ALTER TABLE employees
    ADD CONSTRAINT chk_employees_hourly_wage_positive
    CHECK (hourly_wage > 0);

ALTER TABLE employees
    ADD CONSTRAINT chk_employees_position_valid
    CHECK (position IN ('Barista', 'Cashier', 'Manager', 'Kitchen Staff'));

-- ================================
-- 5) orders
-- ================================
ALTER TABLE orders
    ALTER COLUMN order_date SET DEFAULT now();

ALTER TABLE orders
    ALTER COLUMN order_date SET NOT NULL;

ALTER TABLE orders
    ALTER COLUMN status SET DEFAULT 'pending';

ALTER TABLE orders
    ALTER COLUMN status SET NOT NULL;

ALTER TABLE orders
    ADD CONSTRAINT chk_orders_status_valid
    CHECK (status IN ('pending', 'preparing', 'ready', 'completed', 'cancelled'));

ALTER TABLE orders
    ADD CONSTRAINT chk_orders_total_amount_non_negative
    CHECK (total_amount IS NULL OR total_amount >= 0);

-- ================================
-- 6) order_items
-- ================================
ALTER TABLE order_items
    ALTER COLUMN quantity SET NOT NULL;

ALTER TABLE order_items
    ALTER COLUMN unit_price SET NOT NULL;

ALTER TABLE order_items
    ADD CONSTRAINT chk_order_items_unit_price_positive
    CHECK (unit_price > 0);
```

ทดสอบชุดใหญ่ท้ายสุด — ลองทำสิ่งผิดกฎหลาย ๆ อย่างเพื่อยืนยันว่า constraint ทำงานครบ:

```sql
-- (1) พนักงานตำแหน่งที่ไม่มีในลิสต์
INSERT INTO employees (employee_name, position, hourly_wage, hire_date)
VALUES ('พนักงานลึกลับ', 'Astronaut', 50, '2026-01-01');
```

```text
ERROR:  new row for relation "employees" violates check constraint "chk_employees_position_valid"
DETAIL:  Failing row contains (3, พนักงานลึกลับ, Astronaut, 50.00, 2026-01-01).
```

```sql
-- (2) สถานะออร์เดอร์ที่ไม่มีอยู่จริง
INSERT INTO orders (customer_id, employee_id, status, total_amount)
VALUES (1, 1, 'shipped', 100.00);
```

```text
ERROR:  new row for relation "orders" violates check constraint "chk_orders_status_valid"
DETAIL:  Failing row contains (5, 1, 1, 2026-09-25 09:30:11.223, shipped, 100.00, ...).
```

```sql
-- (3) อีเมลรูปแบบผิด
INSERT INTO customers (customer_name, email)
VALUES ('ทดสอบ อีเมล', 'not-an-email');
```

```text
ERROR:  new row for relation "customers" violates check constraint "chk_customers_email_format"
DETAIL:  Failing row contains (3, ทดสอบ อีเมล, not-an-email, null, null).
```

```sql
-- (4) กรณีถูกต้องทั้งหมด -> ผ่านทุก constraint
INSERT INTO orders (customer_id, employee_id, status, total_amount)
VALUES (2, 2, 'preparing', 240.00)
RETURNING *;
```

```text
 order_id | customer_id | employee_id |         order_date         |  status   | total_amount |             order_uuid
----------+-------------+-------------+-----------------------------+------------+---------------+--------------------------------------
        6 |           2 |           2 | 2026-09-25 09:31:02.881044 | preparing |        240.00 | 4d21e3aa-7f01-4a2b-9c33-11223344abcd
(1 row)
```

ตรวจสอบภาพรวม constraint ทั้งหมดที่มีอยู่ในสคีมาตอนนี้:

```sql
SELECT
    rel.relname AS table_name,
    con.conname AS constraint_name,
    CASE con.contype
        WHEN 'p' THEN 'PRIMARY KEY'
        WHEN 'f' THEN 'FOREIGN KEY'
        WHEN 'u' THEN 'UNIQUE'
        WHEN 'c' THEN 'CHECK'
        WHEN 'x' THEN 'EXCLUSION'
    END AS constraint_type
FROM pg_constraint con
JOIN pg_class rel ON rel.oid = con.conrelid
WHERE rel.relnamespace = 'public'::regnamespace
  AND rel.relname IN ('categories', 'products', 'customers', 'employees', 'orders', 'order_items')
ORDER BY rel.relname, con.contype;
```

ผลลัพธ์ตัวอย่าง (ย่อ):

```text
 table_name  |          constraint_name           | constraint_type
--------------+-------------------------------------+------------------
 customers    | customers_pkey                     | PRIMARY KEY
 customers    | chk_customers_email_format         | CHECK
 customers    | chk_customers_member_since_not_future | CHECK
 employees    | employees_pkey                     | PRIMARY KEY
 employees    | chk_employees_hourly_wage_positive | CHECK
 employees    | chk_employees_position_valid       | CHECK
 order_items  | order_items_pkey                   | PRIMARY KEY
 order_items  | order_items_order_id_fkey          | FOREIGN KEY
 order_items  | order_items_product_id_fkey        | FOREIGN KEY
 order_items  | chk_order_items_unit_price_positive | CHECK
 orders       | orders_pkey                        | PRIMARY KEY
 orders       | orders_customer_id_fkey            | FOREIGN KEY
 orders       | orders_employee_id_fkey            | FOREIGN KEY
 orders       | chk_orders_status_valid            | CHECK
 orders       | chk_orders_total_amount_non_negative | CHECK
 products     | products_pkey                      | PRIMARY KEY
 products     | products_category_id_fkey          | FOREIGN KEY
 products     | chk_products_price_positive         | CHECK
 products     | chk_products_price_not_absurd       | CHECK
```

สคีมาร้านกาแฟตอนนี้ปลอดภัยขึ้นมากแล้ว: ราคาห้ามติดลบ สถานะห้ามเพี้ยน อีเมลต้องมีรูปแบบสมเหตุสมผล และวันที่ต้องไม่ขัดแย้งกันเอง — ทั้งหมดนี้ถูกบังคับใช้ที่**ระดับฐานข้อมูล** ไม่ใช่แค่ที่ฝั่งแอปพลิเคชัน ซึ่งหมายความว่าไม่ว่าจะมีกี่แอปพลิเคชัน กี่ทีม หรือกี่สคริปต์ที่เขียนข้อมูลเข้าฐานข้อมูลนี้ กฎเหล่านี้จะถูกบังคับใช้เสมอโดยไม่มีทางหลีกเลี่ยง

---

## สรุปท้ายบท

ในบทนี้เราได้เรียนรู้เครื่องมือสำคัญสามชนิดที่ทำให้ฐานข้อมูล "ปกป้องตัวเอง" จากข้อมูลที่ไม่ถูกต้อง:

- **`NOT NULL`** — บังคับให้คอลัมน์ต้องมีค่าเสมอ ควรใช้ให้มากที่สุดเท่าที่ตรรกะธุรกิจอนุญาต เพราะ `CHECK` เพียงลำพังไม่สามารถกัน `NULL` ได้
- **`DEFAULT`** — กำหนดค่าตั้งต้นให้คอลัมน์ ทั้งค่าคงที่และค่าจากฟังก์ชัน เช่น `now()`, `gen_random_uuid()`, `CURRENT_DATE` โดยต้องระวังผลกระทบเรื่อง table rewrite เมื่อใช้ volatile function บนตารางขนาดใหญ่
- **`CHECK`** — ทั้งระดับคอลัมน์และระดับตาราง ใช้บังคับเงื่อนไขทางธุรกิจที่ซับซ้อนได้ แต่ต้องระวังพฤติกรรมกับ `NULL` ที่ทำให้เงื่อนไข "ผ่าน" ได้เมื่อผลลัพธ์เป็น `UNKNOWN`
- การ **ตั้งชื่อ constraint** ด้วย `CONSTRAINT` keyword ช่วยให้ error message และการดูแลรักษาระบบง่ายขึ้นมาก
- การใช้ `NOT VALID` + `VALIDATE CONSTRAINT` เป็นเทคนิคสำคัญสำหรับเพิ่ม constraint บนตารางใหญ่ในระบบ production โดยไม่ทำให้ระบบหยุดชะงัก
- **`EXCLUSION` constraint** เป็นเครื่องมือขั้นสูงที่ใช้ป้องกันข้อมูลทับซ้อนกัน (overlapping) เช่น การจองที่ทับเวลากัน โดยอาศัย GiST index
- การเข้าใจ**ลำดับการตรวจสอบ constraint** และวิธีอ่าน error message ช่วยให้ debug ปัญหาข้อมูลได้เร็วขึ้นมาก

หลักการสำคัญที่สุดที่ควรจำจากบทนี้คือ: **สิ่งใดที่สามารถบังคับด้วย constraint ระดับฐานข้อมูลได้ ควรบังคับที่นั่น** อย่าพึ่งพาการตรวจสอบฝั่งแอปพลิเคชันเพียงอย่างเดียว เพราะแอปพลิเคชันอาจมีบั๊ก มีหลายเวอร์ชัน หรือมีสคริปต์ ad-hoc ที่เขียนข้อมูลตรงเข้าฐานข้อมูลโดยข้ามชั้นแอปพลิเคชันไปเลยก็ได้ — ฐานข้อมูลคือ "ด่านสุดท้าย" ที่ควรไว้ใจได้เสมอ

บทถัดไปจะพาไปลงลึกเรื่อง `ALTER TABLE` แบบครบวงจร ทั้งการเปลี่ยนชนิดข้อมูล การเพิ่ม/ลบคอลัมน์บนตารางขนาดใหญ่โดยไม่ให้ระบบหยุดทำงาน และเทคนิคการทำ schema migration อย่างปลอดภัย

**บทถัดไป:** [Part 019: ALTER TABLE](./part-019-alter-table.md)

---

## แบบฝึกหัด

<details>
<summary><strong>ข้อ 1:</strong> เพิ่ม `NOT NULL` ให้คอลัมน์ `customers.customer_name` (สมมติว่ายังไม่มี) โดยต้องตรวจสอบก่อนว่ามีแถวที่เป็น `NULL` อยู่หรือไม่ ก่อนรันคำสั่งจริง</summary>

```sql
-- ขั้นแรก ตรวจสอบว่ามีแถวที่เป็น NULL อยู่หรือไม่
SELECT customer_id FROM customers WHERE customer_name IS NULL;

-- ถ้าไม่มีแถวใดเป็น NULL แล้ว จึงเพิ่ม NOT NULL ได้อย่างปลอดภัย
ALTER TABLE customers
    ALTER COLUMN customer_name SET NOT NULL;
```

ถ้ามีแถวที่เป็น `NULL` อยู่ ต้อง `UPDATE` ให้มีค่าก่อน มิเช่นนั้นคำสั่ง `ALTER TABLE` จะล้มเหลวด้วย error ทำนอง `column "customer_name" contains null values`
</details>

<details>
<summary><strong>ข้อ 2:</strong> เพิ่มคอลัมน์ `created_at TIMESTAMP` ให้ตาราง `products` โดยให้มีค่า default เป็นเวลาปัจจุบันโดยอัตโนมัติทุกครั้งที่มีการเพิ่มสินค้าใหม่</summary>

```sql
ALTER TABLE products
    ADD COLUMN created_at TIMESTAMP NOT NULL DEFAULT now();
```

ทดสอบ:

```sql
INSERT INTO products (product_name, category_id, price)
VALUES ('Cold Brew', 1, 70.00)
RETURNING product_name, created_at;
```
</details>

<details>
<summary><strong>ข้อ 3:</strong> เพิ่ม `CHECK` constraint ให้ `employees.hourly_wage` ต้องมากกว่าค่าแรงขั้นต่ำ 40 บาทต่อชั่วโมง พร้อมตั้งชื่อ constraint ให้สื่อความหมาย</summary>

```sql
ALTER TABLE employees
    ADD CONSTRAINT chk_employees_wage_minimum
    CHECK (hourly_wage >= 40);
```

ทดสอบว่า error message แสดงชื่อ constraint ที่ตั้งไว้:

```sql
INSERT INTO employees (employee_name, position, hourly_wage, hire_date)
VALUES ('พนักงานฝึกงาน', 'Barista', 30, '2026-01-01');
-- ERROR: ... violates check constraint "chk_employees_wage_minimum"
```
</details>

<details>
<summary><strong>ข้อ 4:</strong> อธิบายว่าทำไมคำสั่งต่อไปนี้ insert สำเร็จ ทั้งที่ดูเหมือนควรถูกปฏิเสธ

```sql
CREATE TABLE t1 (x INTEGER CHECK (x BETWEEN 1 AND 10));
INSERT INTO t1 VALUES (NULL);
```
</summary>

เพราะ `NULL BETWEEN 1 AND 10` ไม่ได้ประเมินผลเป็น `FALSE` แต่เป็น `NULL`/`UNKNOWN` ตามกฎ three-valued logic ของ SQL และ `CHECK` constraint จะปฏิเสธแถวก็ต่อเมื่อผลลัพธ์เป็น `FALSE` เท่านั้น เมื่อผลลัพธ์เป็น `NULL` หรือ `TRUE` แถวนั้นจะผ่านการตรวจสอบทั้งคู่

ถ้าต้องการป้องกัน `NULL` ด้วย ต้องเพิ่ม `NOT NULL` เข้าไปด้วย:

```sql
CREATE TABLE t1_fixed (x INTEGER NOT NULL CHECK (x BETWEEN 1 AND 10));
```
</details>

<details>
<summary><strong>ข้อ 5:</strong> เขียน table-level `CHECK` constraint สำหรับตาราง `employees` ที่ต้องการเพิ่ม เพื่อบังคับว่า ถ้า `position = 'Manager'` แล้ว `hourly_wage` ต้องมากกว่าหรือเท่ากับ 80 บาท (ตำแหน่งอื่นไม่ถูกจำกัดด้วยกฎนี้)</summary>

```sql
ALTER TABLE employees
    ADD CONSTRAINT chk_employees_manager_wage
    CHECK (position <> 'Manager' OR hourly_wage >= 80);
```

หลักการ: เขียนเป็น logical implication (`ถ้า A แล้ว B`) ในรูปแบบ SQL คือ `NOT A OR B` ซึ่งในที่นี้คือ `position <> 'Manager' OR hourly_wage >= 80` — ถ้าไม่ใช่ Manager เงื่อนไขเป็นจริงเสมอ (ผ่านฝั่งซ้าย) แต่ถ้าเป็น Manager ต้องพึ่งเงื่อนไขฝั่งขวา
</details>

<details>
<summary><strong>ข้อ 6:</strong> บนตาราง `orders` ที่มีข้อมูลอยู่แล้วหลายล้านแถวในระบบ production ต้องการเพิ่ม `CHECK (total_amount >= 0)` โดยให้กระทบ downtime น้อยที่สุด จงเขียนขั้นตอนทั้งหมด</summary>

```sql
-- ขั้นตอนที่ 1: เพิ่มแบบ NOT VALID ก่อน (เร็ว ล็อกสั้น)
ALTER TABLE orders
    ADD CONSTRAINT chk_orders_total_non_negative
    CHECK (total_amount >= 0) NOT VALID;

-- ขั้นตอนที่ 2: validate ภายหลัง ในช่วง traffic ต่ำ
-- (ใช้ SHARE UPDATE EXCLUSIVE LOCK เท่านั้น ไม่บล็อก read/write ปกติ)
ALTER TABLE orders
    VALIDATE CONSTRAINT chk_orders_total_non_negative;
```

ข้อดี: ตั้งแต่ขั้นตอนที่ 1 เสร็จ ข้อมูลใหม่ทุกแถวจะถูกบังคับตามกฎทันที ส่วนขั้นตอนที่ 2 ใช้ตรวจสอบข้อมูลเก่าแบบไม่บล็อกการใช้งานปกติ
</details>

<details>
<summary><strong>ข้อ 7:</strong> ออกแบบตาราง `employee_shifts` สำหรับบันทึกกะทำงานของพนักงาน โดยต้องป้องกันไม่ให้พนักงานคนเดียวกันมีกะทำงานที่เวลาทับซ้อนกัน (ใช้ EXCLUSION constraint)</summary>

```sql
CREATE EXTENSION IF NOT EXISTS btree_gist;

CREATE TABLE employee_shifts (
    shift_id    SERIAL PRIMARY KEY,
    employee_id INTEGER NOT NULL REFERENCES employees(employee_id),
    shift_time  TSRANGE NOT NULL,
    EXCLUDE USING gist (
        employee_id WITH =,
        shift_time WITH &&
    )
);

-- ทดสอบ
INSERT INTO employee_shifts (employee_id, shift_time)
VALUES (1, '[2026-09-26 08:00, 2026-09-26 16:00)');

-- ซ้อนเวลากับกะเดิมของพนักงานคนเดียวกัน -> ถูกปฏิเสธ
INSERT INTO employee_shifts (employee_id, shift_time)
VALUES (1, '[2026-09-26 14:00, 2026-09-26 22:00)');
```
</details>

<details>
<summary><strong>ข้อ 8:</strong> ตาราง `order_items` ปัจจุบันมี constraint ชื่ออัตโนมัติหลายตัวที่อ่านยาก จงเขียน query เพื่อดึงรายชื่อ constraint ทั้งหมดของตารางนี้พร้อมนิยาม แล้วอธิบายว่าจะใช้ข้อมูลนี้ปรับปรุงชื่อ constraint อย่างไร</summary>

```sql
SELECT conname, pg_get_constraintdef(oid)
FROM pg_constraint
WHERE conrelid = 'order_items'::regclass;
```

จากผลลัพธ์ ให้ไล่ดูทีละตัวที่ชื่อยังเป็นรูปแบบอัตโนมัติ (เช่น `order_items_quantity_check`) แล้วใช้ `DROP CONSTRAINT` ตามด้วย `ADD CONSTRAINT` ชื่อใหม่ที่สื่อความหมายตาม convention เช่น `chk_` สำหรับ CHECK, `fk_` สำหรับ FOREIGN KEY:

```sql
ALTER TABLE order_items DROP CONSTRAINT order_items_quantity_check;
ALTER TABLE order_items ADD CONSTRAINT chk_order_items_quantity_positive CHECK (quantity > 0);
```
</details>

<details>
<summary><strong>ข้อ 9:</strong> วิเคราะห์ error message ต่อไปนี้ แล้วบอกว่าเกิดจากอะไร และควรแก้ไขอย่างไร

```text
ERROR:  new row for relation "orders" violates check constraint "chk_orders_status_valid"
DETAIL:  Failing row contains (10, 3, 2, 2026-09-25 10:00:00, on-hold, 150.00).
```
</summary>

สาเหตุ: พยายาม insert หรือ update แถวในตาราง `orders` โดยให้ค่า `status = 'on-hold'` ซึ่งไม่อยู่ในรายการค่าที่ `chk_orders_status_valid` อนุญาต (`'pending', 'preparing', 'ready', 'completed', 'cancelled'`)

วิธีแก้ไข: เลือกใช้ค่า status ที่อยู่ในลิสต์ที่กำหนดไว้แทน หรือถ้าธุรกิจต้องการสถานะ `'on-hold'` จริง ๆ ต้องแก้ไขนิยาม constraint ให้รวมค่านี้เข้าไปด้วย:

```sql
ALTER TABLE orders DROP CONSTRAINT chk_orders_status_valid;
ALTER TABLE orders ADD CONSTRAINT chk_orders_status_valid
    CHECK (status IN ('pending', 'preparing', 'ready', 'completed', 'cancelled', 'on-hold'));
```
</details>

<details>
<summary><strong>ข้อ 10:</strong> เขียนคำสั่งเดียวเพื่อสร้างตาราง `discount_codes` ที่มีคุณสมบัติครบตามนี้: `code` เป็นตัวอักษรไม่ซ้ำและห้ามว่าง, `discount_percent` ต้องอยู่ระหว่าง 1-100, `valid_from` ต้องมาก่อน `valid_until` เสมอ, และ `created_at` ตั้งค่าเริ่มต้นเป็นเวลาปัจจุบัน</summary>

```sql
CREATE TABLE discount_codes (
    discount_code_id SERIAL PRIMARY KEY,
    code              VARCHAR(20) NOT NULL UNIQUE,
    discount_percent  NUMERIC(5, 2) NOT NULL
        CONSTRAINT chk_discount_percent_range CHECK (discount_percent BETWEEN 1 AND 100),
    valid_from        TIMESTAMP NOT NULL DEFAULT now(),
    valid_until       TIMESTAMP NOT NULL,
    created_at        TIMESTAMP NOT NULL DEFAULT now(),
    CONSTRAINT chk_discount_date_range CHECK (valid_from < valid_until)
);
```

ทดสอบ:

```sql
INSERT INTO discount_codes (code, discount_percent, valid_until)
VALUES ('WELCOME10', 10, now() + INTERVAL '7 days')
RETURNING *;
```
</details>

---

**บทถัดไป:** [Part 019: ALTER TABLE](./part-019-alter-table.md)
