# DELETE และความแตกต่างจาก TRUNCATE

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับพื้นฐาน | Part 014

---

## เป้าหมายการเรียนรู้

เมื่อจบบทนี้ ผู้เรียนจะสามารถ:

1. เขียนคำสั่ง `DELETE FROM ... WHERE` เพื่อลบข้อมูลได้อย่างถูกต้องและปลอดภัย
2. เข้าใจอันตรายของการลืมเขียน `WHERE` ใน `DELETE` และรู้วิธีป้องกันความเสียหายด้วยการทดสอบใน transaction ก่อนลงมือจริง
3. ใช้ `DELETE ... USING` เพื่อลบข้อมูลโดยอ้างอิงเงื่อนไขจากตารางอื่น
4. ใช้ `DELETE ... RETURNING` เพื่อดึงข้อมูลแถวที่ถูกลบกลับมาแสดงผล
5. วิเคราะห์และแก้ไข error ที่เกิดจาก Foreign Key constraint เมื่อพยายามลบข้อมูลที่ถูกอ้างอิงอยู่
6. เข้าใจ `TRUNCATE` ทั้ง syntax, ความเร็ว, และเหตุผลเชิงเทคนิคว่าทำไมมันเร็วกว่า `DELETE` ทั้งตาราง
7. ใช้ `TRUNCATE ... CASCADE` และ `RESTART IDENTITY` ได้อย่างเหมาะสม
8. เปรียบเทียบ `DELETE` vs `TRUNCATE` vs `DROP TABLE` ได้อย่างละเอียดและเลือกใช้ให้ถูกสถานการณ์
9. ออกแบบ soft delete pattern ด้วยคอลัมน์ `deleted_at` พร้อมชั่งน้ำหนักข้อดีข้อเสีย
10. ออกแบบและทดสอบสถานการณ์ลบข้อมูลของระบบร้านกาแฟได้อย่างปลอดภัยแบบมืออาชีพ

---

## เตรียมข้อมูล

ในบทนี้เราจะยังคงใช้ schema ของ "ร้านกาแฟ" (coffee shop) ต่อเนื่องจากบทก่อนหน้า เพื่อให้ทุกตัวอย่างรันได้จริงแบบ self-contained เราจะสร้างตารางและใส่ข้อมูลตัวอย่างใหม่ทั้งหมดในบทนี้

```sql
-- ล้างของเก่า (ถ้ามี) เพื่อเริ่มต้นสะอาด
DROP TABLE IF EXISTS order_items CASCADE;
DROP TABLE IF EXISTS orders CASCADE;
DROP TABLE IF EXISTS products CASCADE;
DROP TABLE IF EXISTS categories CASCADE;
DROP TABLE IF EXISTS customers CASCADE;

-- ตารางหมวดหมู่สินค้า
CREATE TABLE categories (
    category_id   SERIAL PRIMARY KEY,
    category_name VARCHAR(50) NOT NULL UNIQUE,
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ตารางสินค้า
CREATE TABLE products (
    product_id    SERIAL PRIMARY KEY,
    category_id   INTEGER REFERENCES categories(category_id),
    product_name  VARCHAR(100) NOT NULL,
    price         NUMERIC(10,2) NOT NULL CHECK (price >= 0),
    is_active     BOOLEAN NOT NULL DEFAULT true,
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ตารางลูกค้า
CREATE TABLE customers (
    customer_id   SERIAL PRIMARY KEY,
    full_name     VARCHAR(100) NOT NULL,
    email         VARCHAR(100) UNIQUE,
    phone         VARCHAR(20),
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ตารางคำสั่งซื้อ
CREATE TABLE orders (
    order_id      SERIAL PRIMARY KEY,
    customer_id   INTEGER REFERENCES customers(customer_id),
    order_date    TIMESTAMPTZ NOT NULL DEFAULT now(),
    status        VARCHAR(20) NOT NULL DEFAULT 'pending'
);

-- ตารางรายการสินค้าในคำสั่งซื้อ
CREATE TABLE order_items (
    order_item_id SERIAL PRIMARY KEY,
    order_id      INTEGER NOT NULL REFERENCES orders(order_id),
    product_id    INTEGER NOT NULL REFERENCES products(product_id),
    quantity      INTEGER NOT NULL CHECK (quantity > 0),
    unit_price    NUMERIC(10,2) NOT NULL
);

-- ใส่ข้อมูลหมวดหมู่
INSERT INTO categories (category_name) VALUES
    ('กาแฟร้อน'),
    ('กาแฟเย็น'),
    ('เบเกอรี่'),
    ('เครื่องดื่มอื่นๆ');

-- ใส่ข้อมูลสินค้า
INSERT INTO products (category_id, product_name, price, is_active) VALUES
    (1, 'เอสเปรสโซ่',           45.00, true),
    (1, 'อเมริกาโน่ร้อน',        50.00, true),
    (2, 'ลาเต้เย็น',             60.00, true),
    (2, 'คาปูชิโน่เย็น',          60.00, true),
    (2, 'โมคค่าเย็น',            65.00, true),
    (3, 'ครัวซองต์',             55.00, true),
    (3, 'บราวนี่',               45.00, true),
    (4, 'ชาไทย',                 45.00, true),
    (4, 'น้ำส้มคั้น',             40.00, false); -- เลิกขายแล้ว

-- ใส่ข้อมูลลูกค้า
INSERT INTO customers (full_name, email, phone) VALUES
    ('สมชาย ใจดี',     'somchai@example.com',   '0811111111'),
    ('สมหญิง รักเรียน', 'somying@example.com',   '0822222222'),
    ('วิชัย มั่นคง',     'wichai@example.com',    '0833333333'),
    ('มาลี สวยงาม',     'malee@example.com',     '0844444444');

-- ใส่ข้อมูลคำสั่งซื้อ
INSERT INTO orders (customer_id, status) VALUES
    (1, 'completed'),
    (2, 'completed'),
    (3, 'pending'),
    (1, 'cancelled'),
    (4, 'completed');

-- ใส่รายการสินค้าในแต่ละคำสั่งซื้อ
INSERT INTO order_items (order_id, product_id, quantity, unit_price) VALUES
    (1, 1, 2, 45.00),
    (1, 6, 1, 55.00),
    (2, 3, 1, 60.00),
    (2, 7, 2, 45.00),
    (3, 4, 3, 60.00),
    (4, 5, 1, 65.00),
    (5, 2, 2, 50.00),
    (5, 8, 1, 45.00);
```

ตรวจสอบว่าข้อมูลถูกใส่ครบถ้วน:

```sql
SELECT
    (SELECT COUNT(*) FROM categories)  AS categories_count,
    (SELECT COUNT(*) FROM products)    AS products_count,
    (SELECT COUNT(*) FROM customers)   AS customers_count,
    (SELECT COUNT(*) FROM orders)      AS orders_count,
    (SELECT COUNT(*) FROM order_items) AS order_items_count;
```

```
 categories_count | products_count | customers_count | orders_count | order_items_count
------------------+----------------+------------------+---------------+--------------------
                4 |              9 |                4 |             5 |                  8
(1 row)
```

พร้อมแล้ว เรามาเริ่มเรียนรู้เรื่องการลบข้อมูลกันครับ

---

## Step 131: DELETE พื้นฐาน syntax — DELETE FROM ... WHERE

คำสั่ง `DELETE` ใช้สำหรับลบ **แถวข้อมูล (rows)** ออกจากตาราง โดยมี syntax พื้นฐานดังนี้

```sql
DELETE FROM table_name
WHERE condition;
```

ส่วนประกอบสำคัญ:

- `DELETE FROM table_name` — ระบุว่าจะลบข้อมูลจากตารางไหน
- `WHERE condition` — เงื่อนไขที่ระบุว่าแถวไหนจะถูกลบ **(สำคัญมาก — ถ้าไม่มี WHERE จะลบทั้งตาราง!)**

### ตัวอย่างที่ 1: ลบสินค้าที่เลิกขายแล้ว

สมมติเราต้องการลบสินค้า "น้ำส้มคั้น" ที่มีสถานะ `is_active = false` ออกจากระบบ

```sql
DELETE FROM products
WHERE product_name = 'น้ำส้มคั้น' AND is_active = false;
```

```
DELETE 1
```

ข้อความ `DELETE 1` บอกว่ามีแถวถูกลบไป 1 แถว

### ตัวอย่างที่ 2: ลบด้วยเงื่อนไขหลายเงื่อนไข

```sql
DELETE FROM orders
WHERE status = 'cancelled'
  AND order_date < now() - INTERVAL '0 days';
```

```
DELETE 1
```

### ตัวอย่างที่ 3: ลบด้วย subquery

ลบ order_items ที่อ้างอิงสินค้าที่ราคาต่ำกว่า 40 บาท (สมมติ):

```sql
DELETE FROM order_items
WHERE product_id IN (
    SELECT product_id FROM products WHERE price < 40
);
```

```
DELETE 0
```

(ในกรณีนี้ไม่มีสินค้าราคาต่ำกว่า 40 บาท จึงไม่มีแถวถูกลบ)

### ตรวจสอบผลลัพธ์หลัง DELETE

```sql
SELECT product_id, product_name, is_active
FROM products
WHERE product_name = 'น้ำส้มคั้น';
```

```
 product_id | product_name | is_active
------------+--------------+-----------
(0 rows)
```

แถวที่ถูกลบจะหายไปจริง ไม่มี trace หลงเหลือในตาราง (ต่างจาก soft delete ที่จะพูดถึงใน Step 139)

> **หมายเหตุ:** `DELETE FROM` และ `DELETE` (ไม่มี `FROM`) มีความหมายเดียวกันใน PostgreSQL เพราะ `FROM` เป็น keyword บังคับตาม SQL standard แต่ PostgreSQL อนุญาตให้เขียนได้ทั้งสองแบบในบาง context — แต่ทางที่ดีควรเขียน `DELETE FROM table_name` เสมอเพื่อความชัดเจนและ portability

---

## Step 132: อันตรายของการลืม WHERE ใน DELETE และวิธีป้องกันด้วย transaction ทดสอบก่อน

นี่คือหนึ่งในเรื่องที่ "อันตรายที่สุด" ในการเขียน SQL ทั้งหมด หากคุณรันคำสั่งนี้:

```sql
DELETE FROM products;
```

**คำสั่งนี้จะลบข้อมูลสินค้าทุกแถวในตาราง `products` ทันที** โดยไม่มีการถามยืนยันใดๆ ทั้งสิ้น! ต่างจากโปรแกรมทั่วไปที่มักมี dialog "คุณแน่ใจหรือไม่" — PostgreSQL จะทำตามคำสั่งทันทีเมื่อ transaction ถูก commit

### ทำไมเรื่องนี้ถึงอันตรายในชีวิตจริง

สถานการณ์ที่เกิดขึ้นบ่อยในหน้างานจริง:

```sql
-- ตั้งใจจะลบแค่สินค้า id = 5
DELETE FROM products
-- WHERE product_id = 5   <- คอมเมนต์บรรทัดนี้ทิ้งไว้โดยไม่ได้ตั้งใจ ก่อน copy ไปรันจริง
;
```

หรือกรณีคลาสสิกที่สุด: เขียนเงื่อนไข WHERE ทีหลัง แต่ select และ run คำสั่งไปก่อนที่จะพิมพ์เงื่อนไขเสร็จ (โดยเฉพาะเวลาใช้ SQL client ที่ auto-execute เมื่อกด Enter)

### วิธีป้องกันที่ 1: ใช้ Transaction ทดสอบก่อนเสมอ

**นี่คือแนวปฏิบัติที่มืออาชีพทุกคนควรทำเป็นนิสัย** — ห่อคำสั่ง DELETE ด้วย `BEGIN` และตรวจสอบผลก่อน `COMMIT`

```sql
BEGIN;

DELETE FROM products
WHERE product_id = 999;  -- id ที่ไม่มีอยู่จริง เพื่อสาธิต

-- ตรวจสอบผลลัพธ์ก่อนว่าลบไปกี่แถว ถูกต้องตามที่คาดหวังหรือไม่
```

```
BEGIN
DELETE 0
```

หากผลลัพธ์ไม่ตรงกับที่คาดไว้ (เช่น คาดว่าจะลบ 1 แถว แต่ระบบบอก `DELETE 350`) ให้ `ROLLBACK` ทันที:

```sql
ROLLBACK;
```

```
ROLLBACK
```

แต่ถ้าผลลัพธ์ถูกต้องตามคาด จึงค่อย `COMMIT`:

```sql
BEGIN;

DELETE FROM orders
WHERE status = 'cancelled';

-- สมมติผลลัพธ์คือ DELETE 1 ซึ่งตรงกับที่คาดไว้ (มี cancelled order แค่ 1 รายการ)

COMMIT;
```

```
BEGIN
DELETE 1
COMMIT
```

### วิธีป้องกันที่ 2: เปลี่ยน DELETE เป็น SELECT ก่อนเสมอ

ก่อนรัน `DELETE FROM t WHERE condition`, ให้รัน `SELECT * FROM t WHERE condition` ก่อนเสมอ เพื่อดูว่าเงื่อนไขนี้เลือกแถวที่ถูกต้องจริงหรือไม่

```sql
-- ขั้นตอนที่ 1: ตรวจสอบก่อนด้วย SELECT
SELECT * FROM orders WHERE status = 'cancelled';
```

```
 order_id | customer_id |          order_date           |  status
----------+-------------+--------------------------------+-----------
        4 |           1 | 2026-09-25 10:00:00.000000+07  | cancelled
(1 row)
```

```sql
-- ขั้นตอนที่ 2: เมื่อมั่นใจแล้วว่า WHERE clause ถูกต้อง จึงเปลี่ยนเป็น DELETE
DELETE FROM orders WHERE status = 'cancelled';
```

### วิธีป้องกันที่ 3: ปิด autocommit ใน psql

ใน `psql` สามารถตั้งค่าให้ทุกคำสั่งต้องมีการ commit ด้วยตัวเองผ่าน `\set AUTOCOMMIT off` ได้ แต่วิธีที่แนะนำและใช้กันทั่วไปมากกว่าคือ **การฝึกนิสัยเขียน `BEGIN` ก่อนคำสั่งที่มีความเสี่ยงเสมอ**

```sql
\set AUTOCOMMIT off
```

เมื่อปิด autocommit แล้ว ทุกคำสั่งจะอยู่ใน transaction โดยอัตโนมัติ และต้องพิมพ์ `COMMIT` หรือ `ROLLBACK` เพื่อจบ transaction เสมอ

### วิธีป้องกันที่ 4: ใช้ WHERE ที่ชัดเจนเสมอ และระวัง NULL

```sql
-- อันตราย: ถ้า customer_id เป็น NULL คำสั่งนี้จะไม่ error แต่ก็ไม่ลบแถวที่มี customer_id IS NULL ด้วย
-- (นี่เป็นพฤติกรรมปกติของ SQL แต่ก็เป็นจุดที่คนสับสนได้บ่อย)
DELETE FROM orders WHERE customer_id = NULL;  -- ผิดหลักการ ควรใช้ IS NULL
```

```
DELETE 0
```

```sql
-- ถูกต้อง
DELETE FROM orders WHERE customer_id IS NULL;
```

### สรุปกฎเหล็ก 3 ข้อในการทำงานกับ DELETE

1. **เขียน `BEGIN` เสมอ** ก่อนรัน DELETE ที่มีผลกระทบมากกว่า 1 แถว หรือทำงานกับข้อมูล production
2. **SELECT ก่อน DELETE เสมอ** ด้วยเงื่อนไขเดียวกัน เพื่อเห็นแถวที่จะถูกลบล่วงหน้า
3. **ตรวจสอบจำนวนแถวที่ระบบรายงาน** (`DELETE n`) ว่าตรงกับที่คาดไว้หรือไม่ ก่อน `COMMIT`

---

## Step 133: DELETE ... USING — ลบโดยอ้างอิงตารางอื่น

บางครั้งเงื่อนไขการลบข้อมูลในตารางหนึ่ง ต้องอ้างอิงข้อมูลจากอีกตารางหนึ่ง PostgreSQL มี extension พิเศษคือ `DELETE ... USING` ซึ่งช่วยให้เขียน JOIN ในคำสั่ง DELETE ได้สะดวกกว่าการใช้ subquery

### Syntax

```sql
DELETE FROM table_name
USING other_table
WHERE table_name.column = other_table.column
  AND other_condition;
```

### ตัวอย่างที่ 1: ลบ order_items ของคำสั่งซื้อที่ถูกยกเลิก

สมมติเราต้องการลบ `order_items` ทั้งหมดที่เชื่อมโยงกับ `orders` ที่มีสถานะ `cancelled`

```sql
BEGIN;

DELETE FROM order_items
USING orders
WHERE order_items.order_id = orders.order_id
  AND orders.status = 'cancelled';
```

```
BEGIN
DELETE 0
```

(ในข้อมูลตัวอย่างของเรา order ที่ cancelled คือ order_id = 4 ซึ่งไม่มี order_items เชื่อมอยู่ จึงลบได้ 0 แถว — ROLLBACK ไว้ก่อน)

```sql
ROLLBACK;
```

### ตัวอย่างที่ 2: ลบสินค้าที่อยู่ในหมวดหมู่ที่ถูก deprecate

สมมติมีหมวดหมู่ "เครื่องดื่มอื่นๆ" ที่จะถูกยกเลิก และต้องการลบสินค้าทั้งหมดในหมวดนั้นที่ยังไม่เคยถูกสั่งซื้อ (ยังไม่มีใน order_items)

```sql
BEGIN;

DELETE FROM products
USING categories
WHERE products.category_id = categories.category_id
  AND categories.category_name = 'เครื่องดื่มอื่นๆ'
  AND products.product_id NOT IN (SELECT product_id FROM order_items);

ROLLBACK;  -- แค่สาธิต ไม่ commit จริง
```

```
BEGIN
DELETE 0
ROLLBACK
```

(สินค้าในหมวดนี้คือ 'ชาไทย' (id 8) ที่มีอยู่ใน order_items แล้ว และ 'น้ำส้มคั้น' ที่ถูกลบไปแล้วใน Step 131 จึงลบได้ 0 แถว)

### ตัวอย่างที่ 3: DELETE ... USING กับหลายตาราง

```sql
BEGIN;

DELETE FROM order_items
USING orders, customers
WHERE order_items.order_id = orders.order_id
  AND orders.customer_id = customers.customer_id
  AND customers.email = 'somchai@example.com'
  AND orders.status = 'pending';

ROLLBACK;
```

```
BEGIN
DELETE 0
ROLLBACK
```

### เปรียบเทียบ DELETE ... USING กับ subquery

ทั้งสองวิธีให้ผลลัพธ์เหมือนกัน แต่มีข้อแตกต่างเชิง readability และ performance:

```sql
-- วิธีที่ 1: DELETE ... USING (อ่านง่ายเหมือน JOIN)
DELETE FROM order_items
USING orders
WHERE order_items.order_id = orders.order_id
  AND orders.status = 'cancelled';

-- วิธีที่ 2: DELETE ... WHERE ... IN (subquery)
DELETE FROM order_items
WHERE order_id IN (
    SELECT order_id FROM orders WHERE status = 'cancelled'
);
```

| ประเด็น | DELETE ... USING | Subquery (IN) |
|---|---|---|
| Syntax | เหมือน JOIN อ่านง่ายเมื่อ join หลายเงื่อนไข | เหมาะกับเงื่อนไขเดียวที่ไม่ซับซ้อน |
| Performance | Planner มักปรับให้เท่ากัน (เหมือนกัน) | เท่ากันในกรณีส่วนใหญ่ |
| ความเสี่ยง | อาจลบซ้ำถ้า JOIN ทำให้เกิดหลาย match ต่อแถว (ต้องระวัง) | ปลอดภัยกว่าเพราะ IN คืนค่า unique list |
| Portability | เป็น PostgreSQL extension (ไม่ใช่ ANSI SQL standard) | เป็นมาตรฐาน SQL ทั่วไป ใช้ได้กับทุก DBMS |

> **ข้อควรระวัง:** เมื่อใช้ `DELETE ... USING`, หาก join เงื่อนไขทำให้ 1 แถวใน `table_name` match กับหลายแถวใน `other_table` ตัว PostgreSQL จะยังคงลบแถวนั้นเพียงครั้งเดียว (ไม่ error ไม่ duplicate) แต่เงื่อนไข WHERE ที่ join ควรถูกออกแบบให้ชัดเจนเพื่อไม่ให้เกิดความสับสนในการอ่านโค้ด

---

## Step 134: DELETE ... RETURNING — ดูข้อมูลที่ถูกลบ

ปกติเมื่อรัน `DELETE`, PostgreSQL จะบอกแค่จำนวนแถวที่ถูกลบ (`DELETE n`) แต่ไม่ได้แสดงว่า "แถวไหนบ้าง" ที่ถูกลบไป คำสั่ง `RETURNING` ช่วยแก้ปัญหานี้ได้ โดยจะคืนค่าข้อมูลของแถวที่ถูกลบกลับมาเหมือนกับผลลัพธ์ของ `SELECT`

### Syntax

```sql
DELETE FROM table_name
WHERE condition
RETURNING column1, column2, ...;
```

### ตัวอย่างที่ 1: ดูข้อมูลสินค้าที่ถูกลบ

```sql
BEGIN;

DELETE FROM products
WHERE product_id = 9
RETURNING product_id, product_name, price;

ROLLBACK; -- สาธิตเท่านั้น (สินค้า id 9 ถูกลบไปแล้วใน Step 131 จึงไม่มีจริง แต่โครงสร้างคำสั่งถูกต้อง)
```

สมมติว่ายังมีแถวนี้อยู่ ผลลัพธ์จะเป็น:

```
BEGIN
 product_id | product_name | price
------------+--------------+-------
          9 | น้ำส้มคั้น    | 40.00
DELETE 1
ROLLBACK
```

### ตัวอย่างที่ 2: ใช้ RETURNING * เพื่อดูทุกคอลัมน์

```sql
BEGIN;

DELETE FROM orders
WHERE status = 'cancelled'
RETURNING *;
```

```
BEGIN
 order_id | customer_id |          order_date           |  status
----------+-------------+--------------------------------+-----------
        4 |           1 | 2026-09-25 10:15:32.482910+07  | cancelled
DELETE 1
```

```sql
ROLLBACK;
```

### ตัวอย่างที่ 3: ใช้ RETURNING เพื่อเก็บ log การลบ (audit trail)

นี่เป็นเทคนิคที่มีประโยชน์มากในระบบจริง — เก็บข้อมูลที่ถูกลบไปไว้ในตาราง audit log แบบ atomic ในคำสั่งเดียว

```sql
-- สร้างตาราง audit log สำหรับเก็บประวัติการลบ
CREATE TABLE deleted_orders_log (
    log_id        SERIAL PRIMARY KEY,
    order_id      INTEGER,
    customer_id   INTEGER,
    order_date    TIMESTAMPTZ,
    status        VARCHAR(20),
    deleted_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_by    TEXT NOT NULL DEFAULT current_user
);

-- ลบและเก็บ log ในขั้นตอนเดียว โดยใช้ WITH ... AS ร่วมกับ RETURNING
WITH deleted AS (
    DELETE FROM orders
    WHERE status = 'cancelled'
    RETURNING order_id, customer_id, order_date, status
)
INSERT INTO deleted_orders_log (order_id, customer_id, order_date, status)
SELECT order_id, customer_id, order_date, status FROM deleted;
```

```
CREATE TABLE
INSERT 0 1
```

ตรวจสอบ log ที่บันทึกไว้:

```sql
SELECT * FROM deleted_orders_log;
```

```
 log_id | order_id | customer_id |          order_date           |  status   |           deleted_at           | deleted_by
--------+----------+-------------+--------------------------------+-----------+---------------------------------+------------
      1 |        4 |           1 | 2026-09-25 10:00:00.000000+07  | cancelled | 2026-09-25 10:20:11.334521+07  | app_user
(1 row)
```

> **เทคนิคระดับมืออาชีพ:** การใช้ `WITH ... AS (DELETE ... RETURNING ...) INSERT INTO ...` เรียกว่า **Data-Modifying CTE** ซึ่งทำให้การลบและการเก็บ log เป็น atomic operation เดียวกันในหนึ่ง transaction implicit — ถ้าขั้นตอนใดล้มเหลว ทั้งหมดจะ rollback อัตโนมัติ ไม่มีทางที่จะลบข้อมูลไปแล้วแต่ log ไม่ถูกบันทึกได้

---

## Step 135: DELETE ที่ติด Foreign Key constraint — error ที่พบและวิธีแก้

เมื่อตารางมีความสัมพันธ์แบบ Foreign Key การพยายามลบแถวที่ "ถูกอ้างอิง" อยู่โดยตารางลูกจะทำให้เกิด error ป้องกัน referential integrity

### ตัวอย่าง error ที่พบบ่อย

ลองลบสินค้าที่ถูกใช้ใน `order_items` อยู่แล้ว:

```sql
DELETE FROM products WHERE product_id = 1;
```

```
ERROR:  update or delete on table "products" violates foreign key constraint
        "order_items_product_id_fkey" on table "order_items"
DETAIL:  Key (product_id)=(1) is still referenced from table "order_items".
```

ลองลบลูกค้าที่มีคำสั่งซื้ออยู่:

```sql
DELETE FROM customers WHERE customer_id = 1;
```

```
ERROR:  update or delete on table "customers" violates foreign key constraint
        "orders_customer_id_fkey" on table "orders"
DETAIL:  Key (customer_id)=(1) is still referenced from table "orders".
```

### วิธีแก้ที่ 1: ลบตามลำดับ (Bottom-up) — ลบตารางลูกก่อนตารางแม่

หากต้องการลบข้อมูลลูกค้า id = 1 จริงๆ (พร้อมคำสั่งซื้อทั้งหมด) ต้องลบตามลำดับจากตารางที่อยู่ล่างสุดของความสัมพันธ์ขึ้นมา

```sql
BEGIN;

-- 1. ลบ order_items ที่เชื่อมกับ orders ของ customer_id = 1 ก่อน
DELETE FROM order_items
USING orders
WHERE order_items.order_id = orders.order_id
  AND orders.customer_id = 1;

-- 2. ลบ orders ของ customer_id = 1
DELETE FROM orders WHERE customer_id = 1;

-- 3. สุดท้ายค่อยลบ customer
DELETE FROM customers WHERE customer_id = 1;

COMMIT;
```

```
BEGIN
DELETE 2
DELETE 2
DELETE 1
COMMIT
```

### วิธีแก้ที่ 2: ใช้ ON DELETE CASCADE ตอนออกแบบ Foreign Key

หากต้องการให้ PostgreSQL ลบข้อมูลลูกให้อัตโนมัติเมื่อลบข้อมูลแม่ ให้กำหนด `ON DELETE CASCADE` ตอนสร้าง (หรือแก้ไข) constraint

```sql
-- ตัวอย่าง: แก้ constraint ของ order_items ให้ CASCADE เมื่อ orders ถูกลบ
ALTER TABLE order_items
    DROP CONSTRAINT order_items_order_id_fkey,
    ADD CONSTRAINT order_items_order_id_fkey
        FOREIGN KEY (order_id) REFERENCES orders(order_id)
        ON DELETE CASCADE;
```

```
ALTER TABLE
```

ทดสอบผล: ลบ order จะลบ order_items ที่เกี่ยวข้องให้อัตโนมัติ

```sql
BEGIN;

DELETE FROM orders WHERE order_id = 2;

SELECT * FROM order_items WHERE order_id = 2;

COMMIT;
```

```
BEGIN
DELETE 1
 order_item_id | order_id | product_id | quantity | unit_price
----------------+----------+------------+----------+------------
(0 rows)
COMMIT
```

order_items ของ order_id = 2 ถูกลบไปโดยอัตโนมัติพร้อมกับ orders

### วิธีแก้ที่ 3: ON DELETE SET NULL (สำหรับกรณีที่ไม่อยากลบข้อมูลลูก แค่ตัดความสัมพันธ์)

```sql
ALTER TABLE orders
    DROP CONSTRAINT orders_customer_id_fkey,
    ADD CONSTRAINT orders_customer_id_fkey
        FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
        ON DELETE SET NULL;
```

```
ALTER TABLE
```

เมื่อลบ customer แล้ว, `customer_id` ใน orders ที่เกี่ยวข้องจะถูกตั้งเป็น `NULL` แทนที่จะลบ order ทิ้งไปด้วย — เหมาะสำหรับกรณีที่ orders เป็นข้อมูลสำคัญทางบัญชีที่ต้องเก็บไว้แม้ลูกค้าจะถูกลบออกจากระบบแล้ว

### ตาราง ON DELETE options ทั้งหมด

| Option | พฤติกรรม |
|---|---|
| `NO ACTION` (default) | ไม่อนุญาตให้ลบถ้ามี child row อ้างอิงอยู่ — เกิด error (ตรวจสอบตอนจบ statement หรือจบ transaction ถ้า deferred) |
| `RESTRICT` | เหมือน `NO ACTION` แต่ตรวจสอบทันที ไม่รอ deferred |
| `CASCADE` | ลบ child rows ที่อ้างอิงไปด้วยอัตโนมัติ |
| `SET NULL` | ตั้งค่า FK column ของ child rows เป็น NULL |
| `SET DEFAULT` | ตั้งค่า FK column ของ child rows กลับเป็นค่า DEFAULT |

> **คำเตือนสำคัญ:** `ON DELETE CASCADE` เป็นเครื่องมือที่ทรงพลังแต่อันตราย เพราะการลบแถวเดียวในตารางแม่อาจลบข้อมูลจำนวนมหาศาลใน chain ของตารางลูกโดยไม่รู้ตัว ควรใช้อย่างระมัดระวังและมีการวางแผน — โดยเฉพาะเมื่อมี CASCADE หลายชั้น (แม่ → ลูก → หลาน) ควรทดสอบด้วย transaction ก่อนเสมอตามที่เรียนใน Step 132

ก่อนไป Step ถัดไป ให้เรารีเซ็ต constraint กลับเป็นค่าเริ่มต้น (NO ACTION) เพื่อความชัดเจนในตัวอย่างถัดๆ ไป:

```sql
ALTER TABLE order_items
    DROP CONSTRAINT order_items_order_id_fkey,
    ADD CONSTRAINT order_items_order_id_fkey
        FOREIGN KEY (order_id) REFERENCES orders(order_id);

ALTER TABLE orders
    DROP CONSTRAINT orders_customer_id_fkey,
    ADD CONSTRAINT orders_customer_id_fkey
        FOREIGN KEY (customer_id) REFERENCES customers(customer_id);
```

```
ALTER TABLE
ALTER TABLE
```

---

## Step 136: TRUNCATE — syntax, ความเร็วเทียบกับ DELETE ทั้งตาราง

เมื่อต้องการ "ลบข้อมูลทั้งหมดในตาราง" คำสั่ง `TRUNCATE` เป็นทางเลือกที่เร็วกว่า `DELETE` มาก

### Syntax

```sql
TRUNCATE [TABLE] table_name;
```

### ตัวอย่างพื้นฐาน

สมมติเราต้องการล้างข้อมูลทดสอบทั้งหมดในตาราง `deleted_orders_log`:

```sql
TRUNCATE TABLE deleted_orders_log;
```

```
TRUNCATE TABLE
```

ตรวจสอบ:

```sql
SELECT COUNT(*) FROM deleted_orders_log;
```

```
 count
-------
     0
(1 row)
```

### ความแตกต่างของความเร็ว: DELETE vs TRUNCATE

ลองเปรียบเทียบด้วยตารางทดสอบขนาดใหญ่:

```sql
-- สร้างตารางทดสอบพร้อมข้อมูล 1 ล้านแถว
CREATE TABLE test_large (id SERIAL PRIMARY KEY, val TEXT);

INSERT INTO test_large (val)
SELECT 'sample-' || g
FROM generate_series(1, 1000000) AS g;
```

```
CREATE TABLE
INSERT 0 1000000
```

**ทดสอบ DELETE ทั้งตาราง (วัดเวลาด้วย \timing ใน psql):**

```sql
\timing on

DELETE FROM test_large;
```

```
DELETE 1000000
Time: 2841.203 ms
```

**เติมข้อมูลใหม่แล้วทดสอบ TRUNCATE:**

```sql
INSERT INTO test_large (val)
SELECT 'sample-' || g
FROM generate_series(1, 1000000) AS g;

TRUNCATE TABLE test_large;
```

```
INSERT 0 1000000
Time: 1523.887 ms
TRUNCATE TABLE
Time: 12.451 ms
```

จากผลการทดสอบจะเห็นชัดเจนว่า `TRUNCATE` (12.4 ms) เร็วกว่า `DELETE` (2841 ms) แบบต่างระดับ — เร็วขึ้นกว่า **200 เท่า** ในกรณีนี้ (ตัวเลขจริงจะแตกต่างกันไปตามขนาดข้อมูล ฮาร์ดแวร์ และการตั้งค่า)

```sql
\timing off
```

### เหตุผลเชิงเทคนิคที่ TRUNCATE เร็วกว่า

**1. TRUNCATE ไม่สร้าง row-level log (WAL entries ต่อแถว)**

`DELETE` เป็น DML statement ที่ต้อง:
- สแกนหาแต่ละแถวที่ตรงเงื่อนไข (แม้จะลบทั้งตารางก็ต้อง scan ทุกแถว)
- ทำ MVCC — มาร์คแต่ละแถวว่า "ถูกลบแล้ว" (ตั้งค่า `xmax` ของแต่ละ tuple) แต่ยังไม่ลบพื้นที่จริงทันที (รอ `VACUUM` มาเก็บกวาดทีหลัง)
- เขียน WAL (Write-Ahead Log) record แยกสำหรับการเปลี่ยนแปลงของ **แต่ละแถว**
- ถ้ามี trigger แบบ `ROW-level` จะถูกเรียกทำงานสำหรับทุกแถว

`TRUNCATE` ในทางตรงข้าม:
- ไม่สแกนแถวทีละแถวเลย
- **deallocate ทั้ง data file (heap) ของตารางในระดับ disk block ทันที** — คล้ายกับการลบไฟล์ทิ้งทั้งไฟล์ แทนที่จะลบเนื้อหาทีละบรรทัดในไฟล์
- เขียน WAL แค่ record เดียวสำหรับการ truncate ทั้ง relation ไม่ใช่ต่อแถว
- ไม่ทำงานกับ row-level trigger (แต่จะ fire `TRUNCATE`-level trigger ถ้ามีการกำหนดไว้)

**2. TRUNCATE ไม่ต้องพึ่ง VACUUM ทำความสะอาดทีหลัง**

หลังจาก `DELETE` ทั้งตาราง พื้นที่ disk ที่แถวเดิมใช้อยู่จะยังไม่ถูกคืนกลับให้ OS ทันที ต้องรอ `VACUUM` (autovacuum หรือรันเอง) มาเก็บกวาด (reclaim) พื้นที่ในภายหลัง ส่วน `TRUNCATE` จะคืนพื้นที่ disk ให้ทันทีโดยไม่ต้องรอ VACUUM

**3. TRUNCATE ใช้ AccessExclusiveLock**

`TRUNCATE` จะล็อกตารางแบบ `ACCESS EXCLUSIVE` ซึ่งบล็อกการอ่าน/เขียนทุกชนิดจากทุก session อื่น ระหว่างที่ทำงาน (แต่เพราะทำงานเร็วมาก จึงมักไม่เป็นปัญหาในทางปฏิบัติ) ในขณะที่ `DELETE` ใช้ `ROW EXCLUSIVE` lock ซึ่งอนุญาตให้ query อื่นยังคง `SELECT` ข้อมูลได้ระหว่างที่ DELETE กำลังทำงาน

ทำความสะอาดตารางทดสอบ:

```sql
DROP TABLE test_large;
```

```
DROP TABLE
```

---

## Step 137: TRUNCATE ... CASCADE และ RESTART IDENTITY

### ปัญหา: TRUNCATE ติด Foreign Key เหมือนกับ DELETE หรือไม่?

ลองสั่ง TRUNCATE ตารางที่ถูกอ้างอิงโดยตารางอื่น:

```sql
TRUNCATE TABLE products;
```

```
ERROR:  cannot truncate a table referenced in a foreign key constraint
DETAIL:  Table "order_items" references "products".
HINT:  Truncate table "order_items" too, or use TRUNCATE ... CASCADE.
```

PostgreSQL ให้คำแนะนำมาตรงๆ ในข้อความ error เลยว่าให้ใช้ `CASCADE` หรือ truncate ตารางที่เกี่ยวข้องไปพร้อมกัน

### วิธีที่ 1: TRUNCATE หลายตารางพร้อมกันในคำสั่งเดียว

```sql
TRUNCATE TABLE order_items, products;
```

```
TRUNCATE TABLE
```

วิธีนี้จะสำเร็จเพราะ `order_items` (ตารางลูก) ถูก truncate พร้อมกับ `products` (ตารางแม่) ในคำสั่งเดียวกัน

### วิธีที่ 2: TRUNCATE ... CASCADE

หากไม่อยากระบุตารางลูกทั้งหมดเอง สามารถใช้ `CASCADE` ให้ PostgreSQL หาตารางที่เกี่ยวข้องและ truncate ให้อัตโนมัติ

```sql
TRUNCATE TABLE categories CASCADE;
```

```
NOTICE:  truncate cascades to table "products"
NOTICE:  truncate cascades to table "order_items"
TRUNCATE TABLE
```

สังเกตว่า PostgreSQL แสดง `NOTICE` บอกให้ทราบว่ามีการ cascade ไปที่ตารางไหนบ้าง — เป็นเรื่องสำคัญมากที่ต้องอ่าน NOTICE เหล่านี้ก่อนตัดสินใจว่าจะ commit หรือไม่ (โดยเฉพาะเมื่อทำใน transaction)

> **คำเตือน:** `TRUNCATE ... CASCADE` มีความหมายต่างจาก `ON DELETE CASCADE` ที่เราเรียนใน Step 135 — `TRUNCATE CASCADE` จะ truncate ตารางลูก **ทั้งตาราง** (ลบทุกแถวไม่ว่าจะเกี่ยวข้องกับแถวที่ถูกลบใน parent หรือไม่) ในขณะที่ `DELETE` ที่มี `ON DELETE CASCADE` จะลบเฉพาะแถวลูกที่เกี่ยวข้องกับแถวแม่ที่ถูกลบเท่านั้น ต้องแยกสองแนวคิดนี้ให้ชัดเจน

### RESTART IDENTITY — รีเซ็ต SERIAL/IDENTITY sequence

เมื่อตารางใช้ `SERIAL` หรือ `GENERATED ... AS IDENTITY` เป็น primary key การ `TRUNCATE` แบบปกติจะ**ไม่**รีเซ็ตค่า sequence ให้กลับไปเริ่มที่ 1

```sql
-- เติมข้อมูลกลับเข้าไปใหม่หลังโดน cascade ลบไป
INSERT INTO categories (category_name) VALUES ('กาแฟร้อน'), ('กาแฟเย็น');

SELECT * FROM categories;
```

```
 category_id | category_name |          created_at
-------------+----------------+--------------------------------
           5 | กาแฟร้อน       | 2026-09-25 10:35:02.112345+07
           6 | กาแฟเย็น       | 2026-09-25 10:35:02.112345+07
(2 rows)
```

สังเกตว่า `category_id` เริ่มจาก 5 ไม่ใช่ 1 (เพราะ sequence เดิมยังจำค่าที่เคยใช้ไปแล้วตั้งแต่ก่อน TRUNCATE)

ใช้ `RESTART IDENTITY` เพื่อรีเซ็ต sequence กลับไปเริ่มต้นใหม่:

```sql
TRUNCATE TABLE categories RESTART IDENTITY CASCADE;

INSERT INTO categories (category_name) VALUES ('กาแฟร้อน'), ('กาแฟเย็น'), ('เบเกอรี่'), ('เครื่องดื่มอื่นๆ');

SELECT * FROM categories;
```

```
NOTICE:  truncate cascades to table "products"
NOTICE:  truncate cascades to table "order_items"
TRUNCATE TABLE
INSERT 0 4
 category_id | category_name |          created_at
-------------+----------------+--------------------------------
           1 | กาแฟร้อน       | 2026-09-25 10:36:44.881023+07
           2 | กาแฟเย็น       | 2026-09-25 10:36:44.881023+07
           3 | เบเกอรี่       | 2026-09-25 10:36:44.881023+07
           4 | เครื่องดื่มอื่นๆ | 2026-09-25 10:36:44.881023+07
(4 rows)
```

`category_id` กลับมาเริ่มที่ 1 ใหม่แล้ว เพราะ `RESTART IDENTITY` สั่งให้รีเซ็ต sequence ทุกตัวที่เชื่อมอยู่กับตารางที่ถูก truncate

### CONTINUE IDENTITY (default)

หากไม่ระบุอะไร ค่า default คือ `CONTINUE IDENTITY` ซึ่งหมายถึง "ไม่ต้องรีเซ็ต sequence" — เขียนแบบเต็มได้ดังนี้ (แต่ไม่จำเป็นเพราะเป็น default อยู่แล้ว):

```sql
TRUNCATE TABLE categories CONTINUE IDENTITY;
```

### สรุป syntax ทั้งหมดของ TRUNCATE

```sql
TRUNCATE [TABLE] [ONLY] table_name [, ...]
    [RESTART IDENTITY | CONTINUE IDENTITY]
    [CASCADE | RESTRICT];
```

- `ONLY` — ไม่ truncate ตารางลูกที่สืบทอดมาจาก table inheritance (partition/inheritance)
- `RESTART IDENTITY` — รีเซ็ต sequence ของคอลัมน์ที่เป็น identity/serial กลับไปค่าเริ่มต้น
- `CONTINUE IDENTITY` (default) — ไม่แตะ sequence
- `CASCADE` — truncate ตารางที่มี FK อ้างอิงมาด้วยอัตโนมัติ
- `RESTRICT` (default) — ห้าม truncate ถ้ามีตารางอื่นอ้างอิงอยู่ (จะ error แบบที่เราเห็นด้านบน)

เติมข้อมูลกลับเข้าตารางทั้งหมดให้ครบเพื่อความต่อเนื่องของบทเรียน:

```sql
INSERT INTO products (category_id, product_name, price, is_active) VALUES
    (1, 'เอสเปรสโซ่',    45.00, true),
    (1, 'อเมริกาโน่ร้อน', 50.00, true),
    (2, 'ลาเต้เย็น',      60.00, true),
    (2, 'คาปูชิโน่เย็น',   60.00, true),
    (2, 'โมคค่าเย็น',     65.00, true),
    (3, 'ครัวซองต์',      55.00, true),
    (3, 'บราวนี่',        45.00, true),
    (4, 'ชาไทย',          45.00, true);
```

```
INSERT 0 8
```

---

## Step 138: เปรียบเทียบ DELETE vs TRUNCATE vs DROP TABLE อย่างละเอียด

ทั้งสามคำสั่งนี้มักถูกสับสนกันบ่อยเพราะดู "ทำให้ข้อมูลหายไป" เหมือนกัน แต่ในความเป็นจริงมีความแตกต่างเชิงโครงสร้างและผลกระทบที่ต่างกันมาก

### ตารางเปรียบเทียบละเอียด

| หัวข้อ | `DELETE` | `TRUNCATE` | `DROP TABLE` |
|---|---|---|---|
| **สิ่งที่ถูกลบ** | แถวข้อมูล (rows) ตามเงื่อนไข WHERE | ทุกแถวในตาราง | โครงสร้างตารางทั้งหมด (schema + data) |
| **สามารถใช้ WHERE ได้หรือไม่** | ได้ (ลบบางส่วน) | ไม่ได้ (ลบทั้งตารางเท่านั้น) | ไม่เกี่ยวข้อง (ไม่มีข้อมูลเหลืออยู่แล้ว) |
| **ตารางยังคงอยู่หรือไม่** | อยู่ (โครงสร้างไม่เปลี่ยน) | อยู่ (โครงสร้างไม่เปลี่ยน) | หายไปทั้งหมด (ต้อง CREATE TABLE ใหม่) |
| **ความเร็ว (ตารางขนาดใหญ่)** | ช้า — สแกนและลบทีละแถว | เร็วมาก — deallocate ทั้ง data file | เร็วมาก — ลบทั้ง object |
| **สร้าง row-level WAL log** | ใช่ — ต่อแถว | ไม่ — log ระดับ relation | ไม่ — log ระดับ object |
| **สามารถ ROLLBACK ได้หรือไม่ (ภายใน transaction)** | ได้ | ได้ (PostgreSQL รองรับ TRUNCATE แบบ transactional ซึ่งต่างจาก DBMS อื่นหลายตัว) | ได้ |
| **Trigger ที่ทำงาน** | `ROW`-level และ `STATEMENT`-level triggers | เฉพาะ `STATEMENT`-level `TRUNCATE` triggers | ไม่มี trigger ทำงาน (object ถูกลบไปเลย) |
| **ผลต่อ Sequence (SERIAL/IDENTITY)** | ไม่กระทบ sequence เลย | กระทบเมื่อใช้ `RESTART IDENTITY` เท่านั้น | Sequence ที่ผูกกับ SERIAL จะถูกลบไปด้วย (ถ้าสร้างจาก SERIAL) |
| **ต้อง VACUUM ทีหลังหรือไม่** | ควรทำ (โดยเฉพาะถ้าลบข้อมูลจำนวนมาก) | ไม่จำเป็น (คืนพื้นที่ disk ทันที) | ไม่เกี่ยวข้อง |
| **Foreign Key constraint** | Error ถ้ามี child rows อ้างอิง (เว้นแต่มี CASCADE) | Error ถ้ามีตารางอื่นอ้างอิง (เว้นแต่ใช้ TRUNCATE CASCADE) | Error ถ้ามีตารางอื่นอ้างอิงอยู่ (เว้นแต่ใช้ DROP ... CASCADE) |
| **Permission ที่ต้องมี** | `DELETE` privilege บนตาราง | `TRUNCATE` privilege บนตาราง | `DROP` privilege หรือเป็นเจ้าของตาราง (owner) |
| **RETURNING รองรับหรือไม่** | รองรับ | ไม่รองรับ | ไม่เกี่ยวข้อง |
| **ใช้กับ WHERE บางส่วนได้ไหม** | ได้ | ไม่ได้ | ไม่ได้ |
| **เหมาะกับสถานการณ์** | ลบข้อมูลบางส่วนตามเงื่อนไขธุรกิจ | ล้างข้อมูลทั้งตารางแบบเร็ว (เช่น staging table, test data) | เลิกใช้ตารางนี้ไปเลยถาวร (schema migration, cleanup) |

### ตัวอย่างสาธิตความแตกต่างแบบเห็นภาพ

```sql
-- สร้างตารางทดสอบเล็กๆ 3 ชุดที่มีโครงสร้างเดียวกัน
CREATE TABLE demo_delete   (id SERIAL PRIMARY KEY, val TEXT);
CREATE TABLE demo_truncate (id SERIAL PRIMARY KEY, val TEXT);
CREATE TABLE demo_drop     (id SERIAL PRIMARY KEY, val TEXT);

INSERT INTO demo_delete   (val) VALUES ('a'), ('b'), ('c');
INSERT INTO demo_truncate (val) VALUES ('a'), ('b'), ('c');
INSERT INTO demo_drop     (val) VALUES ('a'), ('b'), ('c');
```

```
CREATE TABLE
CREATE TABLE
CREATE TABLE
INSERT 0 3
INSERT 0 3
INSERT 0 3
```

```sql
DELETE FROM demo_delete;      -- ตารางยังอยู่ ว่างเปล่า
TRUNCATE TABLE demo_truncate; -- ตารางยังอยู่ ว่างเปล่า
DROP TABLE demo_drop;         -- ตารางหายไปทั้งหมด
```

```
DELETE 3
TRUNCATE TABLE
DROP TABLE
```

ตรวจสอบว่าตารางไหนยังอยู่บ้าง:

```sql
SELECT table_name
FROM information_schema.tables
WHERE table_schema = 'public'
  AND table_name IN ('demo_delete', 'demo_truncate', 'demo_drop');
```

```
   table_name
-----------------
 demo_delete
 demo_truncate
(2 rows)
```

`demo_drop` หายไปจาก schema เลย ในขณะที่อีกสองตารางยังคงอยู่ (แต่ว่างเปล่า)

```sql
-- ทำความสะอาด
DROP TABLE demo_delete, demo_truncate;
```

```
DROP TABLE
```

### แนวทางการเลือกใช้ในโลกจริง

```
ต้องการลบข้อมูลบางส่วนตามเงื่อนไข? ──────► ใช้ DELETE (พร้อม WHERE + transaction ทดสอบ)
                                              │
ต้องการล้างข้อมูลทั้งตารางแบบเร็ว           │
และตารางยังต้องใช้งานต่อ? ──────────────────► ใช้ TRUNCATE (ระวัง FK และ RESTART IDENTITY)
                                              │
ต้องการเลิกใช้ตารางนี้ถาวร                  │
ไม่ต้องการ schema นี้อีกแล้ว? ───────────────► ใช้ DROP TABLE
```

---

## Step 139: Soft delete pattern — การใช้คอลัมน์ deleted_at แทนการลบจริง

ในระบบ production จริงจำนวนมาก (โดยเฉพาะระบบที่ต้องมี audit trail, กฎหมายคุ้มครองข้อมูล, หรือฟีเจอร์ "กู้คืนข้อมูล") มักไม่ลบข้อมูลจริงๆ ด้วย `DELETE` แต่ใช้เทคนิคที่เรียกว่า **Soft Delete** แทน

### แนวคิดของ Soft Delete

แทนที่จะลบแถวออกจากตารางจริง เราเพิ่มคอลัมน์ `deleted_at TIMESTAMPTZ` (ค่าเริ่มต้นเป็น `NULL`) เมื่อ "ลบ" ข้อมูล เราแค่ `UPDATE` ให้ `deleted_at = now()` แทน

### การ implement Soft Delete

```sql
-- เพิ่มคอลัมน์ deleted_at ให้กับตาราง customers
ALTER TABLE customers ADD COLUMN deleted_at TIMESTAMPTZ;
```

```
ALTER TABLE
```

**"ลบ" ลูกค้าแบบ soft delete:**

```sql
UPDATE customers
SET deleted_at = now()
WHERE customer_id = 3;
```

```
UPDATE 1
```

**Query ข้อมูลที่ยัง "ไม่ถูกลบ" (active records):**

```sql
SELECT customer_id, full_name, email, deleted_at
FROM customers
WHERE deleted_at IS NULL;
```

```
 customer_id |   full_name    |          email          | deleted_at
-------------+-----------------+--------------------------+------------
           1 | สมชาย ใจดี      | somchai@example.com     |
           2 | สมหญิง รักเรียน | somying@example.com     |
           4 | มาลี สวยงาม     | malee@example.com       |
(3 rows)
```

**กู้คืนข้อมูล (restore) ได้ง่ายมาก — แค่ตั้ง deleted_at กลับเป็น NULL:**

```sql
UPDATE customers
SET deleted_at = NULL
WHERE customer_id = 3;
```

```
UPDATE 1
```

### เทคนิคที่ทำให้ Soft Delete สะดวกใช้งานมากขึ้น

**1. สร้าง View สำหรับข้อมูลที่ active เท่านั้น**

```sql
CREATE VIEW active_customers AS
SELECT customer_id, full_name, email, phone, created_at
FROM customers
WHERE deleted_at IS NULL;
```

```
CREATE VIEW
```

```sql
SELECT * FROM active_customers;
```

```
 customer_id |   full_name    |          email          |   phone    |           created_at
-------------+-----------------+--------------------------+------------+---------------------------------
           1 | สมชาย ใจดี      | somchai@example.com     | 0811111111 | 2026-09-25 09:00:00.000000+07
           2 | สมหญิง รักเรียน | somying@example.com     | 0822222222 | 2026-09-25 09:00:00.000000+07
           3 | วิชัย มั่นคง    | wichai@example.com      | 0833333333 | 2026-09-25 09:00:00.000000+07
           4 | มาลี สวยงาม     | malee@example.com       | 0844444444 | 2026-09-25 09:00:00.000000+07
(4 rows)
```

**2. Partial Unique Index เพื่ออนุญาตให้ email ซ้ำได้เฉพาะกับ record ที่ถูก soft delete ไปแล้ว**

ปัญหาที่พบบ่อย: ถ้า `email` มี `UNIQUE constraint` ตรงๆ เมื่อ soft delete ลูกค้าคนหนึ่งไปแล้ว แต่มีลูกค้าใหม่มาสมัครด้วย email เดียวกัน จะ insert ไม่ได้เพราะ email ซ้ำกับ record ที่ถูก soft delete (ทั้งที่ในทางธุรกิจควรอนุญาต)

```sql
-- ลบ UNIQUE constraint เดิม
ALTER TABLE customers DROP CONSTRAINT customers_email_key;

-- สร้าง partial unique index แทน — unique เฉพาะ record ที่ยัง active เท่านั้น
CREATE UNIQUE INDEX customers_email_active_uidx
    ON customers (email)
    WHERE deleted_at IS NULL;
```

```
ALTER TABLE
CREATE INDEX
```

**3. ใช้ Rule หรือ Trigger เพื่อบังคับให้ DELETE กลายเป็น soft delete โดยอัตโนมัติ (ขั้นสูง)**

```sql
CREATE OR REPLACE FUNCTION soft_delete_customer()
RETURNS TRIGGER AS $$
BEGIN
    UPDATE customers SET deleted_at = now() WHERE customer_id = OLD.customer_id;
    RETURN NULL;  -- ป้องกันไม่ให้เกิด DELETE จริง
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_soft_delete_customer
    BEFORE DELETE ON customers
    FOR EACH ROW
    EXECUTE FUNCTION soft_delete_customer();
```

```
CREATE FUNCTION
CREATE TRIGGER
```

ทดสอบ: แม้จะเรียก `DELETE` ตรงๆ ก็จะกลายเป็น soft delete แทน

```sql
DELETE FROM customers WHERE customer_id = 4;

SELECT customer_id, full_name, deleted_at FROM customers WHERE customer_id = 4;
```

```
DELETE 0
 customer_id |  full_name  |          deleted_at
-------------+--------------+--------------------------------
           4 | มาลี สวยงาม  | 2026-09-25 10:52:18.774521+07
(1 row)
```

สังเกตว่า `DELETE 0` (ไม่มีแถวถูกลบจริงเพราะ trigger คืนค่า `NULL`) แต่ `deleted_at` ถูกตั้งค่าแล้ว — ข้อมูลยังอยู่ในตารางแต่ถูกมาร์คว่า "soft deleted"

### ข้อดีของ Soft Delete

1. **กู้คืนข้อมูลได้** — เผลอลบไปสามารถ restore กลับมาได้ทันที ไม่ต้องพึ่ง backup
2. **รักษา referential integrity ได้ง่าย** — ข้อมูลที่เกี่ยวข้อง (เช่น orders ของลูกค้าที่ถูกลบ) ยังคง valid และอ้างอิงได้ ไม่ error FK constraint
3. **Audit trail โดยธรรมชาติ** — รู้ได้ว่าใครถูกลบเมื่อไหร่ (โดยเฉพาะถ้าเพิ่ม `deleted_by` ด้วย)
4. **รองรับ requirement ทางกฎหมาย** — บางระบบ (เช่น ธุรกรรมทางการเงิน) กฎหมายบังคับให้เก็บข้อมูลไว้ระยะหนึ่งแม้ผู้ใช้จะขอลบ

### ข้อเสียของ Soft Delete

1. **ทุก query ต้องมี `WHERE deleted_at IS NULL`** — เสี่ยงต่อการลืมเงื่อนไขนี้และแสดงข้อมูลที่ "ถูกลบ" ปนออกมาโดยไม่ตั้งใจ (แก้ได้ด้วย View แต่ต้องมีวินัยในการใช้)
2. **ตารางโตขึ้นเรื่อยๆ ไม่มีวันเล็กลง** — ข้อมูลไม่เคยถูกลบจริง ต้องมี process แยกมา archive หรือ hard-delete ข้อมูลเก่าเป็นระยะ
3. **Index และ Unique constraint ซับซ้อนขึ้น** — ต้องใช้ partial index แทน unique constraint ปกติ (ตามตัวอย่างข้างต้น)
4. **Performance ลดลงเล็กน้อย** — ทุก query ต้องกรองแถว soft-deleted ออกเพิ่มเติม แม้จะทำ index ช่วยได้ก็ตาม
5. **ข้อมูลที่ละเอียดอ่อน (PII) ยังคงอยู่ในระบบ** — อาจขัดกับกฎหมาย เช่น GDPR ที่กำหนดสิทธิ "right to be forgotten" ต้องมี process แยกมาลบข้อมูลจริง (hard delete) เมื่อครบกำหนด

### ตารางสรุปเปรียบเทียบ Soft Delete vs Hard Delete

| ประเด็น | Soft Delete (`deleted_at`) | Hard Delete (`DELETE`) |
|---|---|---|
| ข้อมูลหายจริงหรือไม่ | ไม่ ยังอยู่ในตาราง | หายจริง |
| กู้คืนได้หรือไม่ | ได้ทันที | ต้องพึ่ง backup/point-in-time recovery |
| ขนาดตาราง | โตขึ้นเรื่อยๆ | คงที่หรือเล็กลง |
| ความซับซ้อนของ query | สูงกว่า (ต้องกรอง deleted_at) | ต่ำกว่า |
| เหมาะกับ | ข้อมูลธุรกิจสำคัญ (orders, customers) | ข้อมูล temporary, log, cache, staging |

ลบ trigger และ function ที่สร้างไว้เพื่อสาธิต เพื่อไม่ให้กระทบตัวอย่างในแบบฝึกหัดถัดไป:

```sql
DROP TRIGGER trg_soft_delete_customer ON customers;
DROP FUNCTION soft_delete_customer();
DROP VIEW active_customers;
ALTER TABLE customers DROP COLUMN deleted_at;
ALTER TABLE customers ADD CONSTRAINT customers_email_key UNIQUE (email);
DROP INDEX IF EXISTS customers_email_active_uidx;
```

```
DROP TRIGGER
DROP FUNCTION
DROP VIEW
ALTER TABLE
ALTER TABLE
DROP INDEX
```

> **หมายเหตุ:** ในระบบจริงระดับ production มักจะไม่ใช้ trigger บังคับ soft delete แบบเงียบๆ (silent) เพราะทำให้ debug ยากและสร้างความประหลาดใจให้ทีม ทางที่ดีกว่าคือให้ application layer เรียก `UPDATE ... SET deleted_at = now()` ตรงๆ อย่างชัดเจน หรือใช้ ORM ที่รองรับ soft delete pattern อยู่แล้ว (เช่น Django's `SoftDeleteModel`, Rails' `paranoia` gem เป็นต้น)

---

## สรุปท้ายบท

ในบทนี้เราได้เรียนรู้เรื่องการลบข้อมูลใน PostgreSQL อย่างละเอียด ครอบคลุมทั้งเชิงทฤษฎีและเชิงปฏิบัติ:

1. **`DELETE FROM ... WHERE`** คือคำสั่งพื้นฐานสำหรับลบแถวข้อมูลตามเงื่อนไข
2. **การลืม WHERE คือหายนะ** — ต้องฝึกนิสัย `BEGIN` → ทดสอบ → `COMMIT`/`ROLLBACK` และ `SELECT` ก่อน `DELETE` เสมอ
3. **`DELETE ... USING`** ช่วยให้ลบข้อมูลโดยอ้างอิงเงื่อนไขจากตารางอื่นได้สะดวกแบบ JOIN
4. **`DELETE ... RETURNING`** ทำให้เห็นข้อมูลที่ถูกลบ และสามารถใช้ร่วมกับ Data-Modifying CTE เพื่อทำ audit log แบบ atomic
5. **Foreign Key constraint** ป้องกันการลบข้อมูลที่ถูกอ้างอิงอยู่ — ต้องลบตามลำดับ (ลูกก่อนแม่) หรือกำหนด `ON DELETE CASCADE`/`SET NULL` ตอนออกแบบ
6. **`TRUNCATE`** เร็วกว่า `DELETE` ทั้งตารางมาก เพราะไม่สร้าง row-level WAL log และ deallocate ทั้ง data file ในครั้งเดียว
7. **`TRUNCATE ... CASCADE`** และ **`RESTART IDENTITY`** ช่วยจัดการ FK constraint และรีเซ็ต sequence ตามลำดับ
8. **DELETE vs TRUNCATE vs DROP TABLE** มีความแตกต่างกันชัดเจนทั้งด้านความเร็ว ผลกระทบต่อโครงสร้างตาราง และ permission ที่ต้องใช้
9. **Soft Delete pattern** (คอลัมน์ `deleted_at`) เป็นทางเลือกที่ปลอดภัยกว่าสำหรับข้อมูลธุรกิจสำคัญ แลกกับความซับซ้อนของ query ที่เพิ่มขึ้น

การลบข้อมูลเป็นหนึ่งใน operation ที่ "ย้อนกลับไม่ได้" มากที่สุดในฐานข้อมูล การเข้าใจเครื่องมือแต่ละตัวอย่างลึกซึ้ง พร้อมฝึกวินัยความปลอดภัย (transaction testing, RETURNING, soft delete) จะช่วยป้องกันความเสียหายร้ายแรงในระบบ production ได้

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1

เขียนคำสั่ง `DELETE` เพื่อลบสินค้า (`products`) ที่มีราคา (`price`) น้อยกว่า 40 บาท

<details>
<summary>เฉลย</summary>

```sql
BEGIN;

-- ตรวจสอบก่อนด้วย SELECT
SELECT * FROM products WHERE price < 40;

-- ถ้าผลลัพธ์ถูกต้อง ค่อยลบ
DELETE FROM products WHERE price < 40;

COMMIT;
```

ในข้อมูลตัวอย่างของเรา ไม่มีสินค้าราคาต่ำกว่า 40 บาทอยู่แล้ว (หลังจากลบน้ำส้มคั้นไปใน Step 131) ดังนั้นผลลัพธ์คือ `DELETE 0`

</details>

---

### แบบฝึกหัดที่ 2

อธิบายว่าทำไมคำสั่งต่อไปนี้ถึงอันตราย และควรแก้ไขอย่างไรให้ปลอดภัย

```sql
DELETE FROM orders;
```

<details>
<summary>เฉลย</summary>

คำสั่งนี้อันตรายเพราะ **ไม่มี WHERE clause** จึงจะลบข้อมูลทุกแถวในตาราง `orders` ทั้งหมดทันที โดยไม่มีการยืนยันใดๆ

วิธีแก้ไขให้ปลอดภัย:

```sql
BEGIN;

-- เพิ่มเงื่อนไข WHERE ที่ระบุแถวที่ต้องการลบจริงๆ
DELETE FROM orders WHERE order_id = 3;  -- ตัวอย่าง

-- ตรวจสอบผลลัพธ์ (DELETE n) ว่าตรงกับที่คาดหวังหรือไม่
-- ถ้าถูกต้อง
COMMIT;
-- ถ้าผิดพลาด
-- ROLLBACK;
```

หลักการสำคัญ: ทุกครั้งที่เขียน `DELETE` ให้ถามตัวเองก่อนว่า "มี WHERE clause หรือยัง" และทดสอบด้วย `SELECT` เดียวกันก่อนเสมอ

</details>

---

### แบบฝึกหัดที่ 3

เขียนคำสั่ง `DELETE ... USING` เพื่อลบ `order_items` ทั้งหมดที่เชื่อมโยงกับคำสั่งซื้อ (`orders`) ของลูกค้าที่ชื่อ "สมชาย ใจดี"

<details>
<summary>เฉลย</summary>

```sql
BEGIN;

DELETE FROM order_items
USING orders, customers
WHERE order_items.order_id = orders.order_id
  AND orders.customer_id = customers.customer_id
  AND customers.full_name = 'สมชาย ใจดี';

ROLLBACK; -- หรือ COMMIT ถ้าต้องการลบจริง
```

</details>

---

### แบบฝึกหัดที่ 4

เขียนคำสั่ง `DELETE ... RETURNING` เพื่อลบสินค้าที่ `is_active = false` และแสดงชื่อสินค้ากับราคาของแถวที่ถูกลบ

<details>
<summary>เฉลย</summary>

```sql
BEGIN;

DELETE FROM products
WHERE is_active = false
RETURNING product_name, price;

ROLLBACK; -- หรือ COMMIT ถ้าต้องการลบจริง
```

(ในข้อมูลตัวอย่างปัจจุบันไม่มีสินค้าที่ `is_active = false` แล้ว เพราะถูกลบไปใน Step 131 ดังนั้นผลลัพธ์คือ `DELETE 0`)

</details>

---

### แบบฝึกหัดที่ 5

ลองรันคำสั่งนี้และอธิบายว่าทำไมจึงเกิด error พร้อมเสนอวิธีแก้ 2 วิธี

```sql
DELETE FROM categories WHERE category_name = 'กาแฟร้อน';
```

<details>
<summary>เฉลย</summary>

จะเกิด error เพราะ `categories` ถูกอ้างอิงโดย `products` ผ่าน Foreign Key (`products.category_id REFERENCES categories.category_id`) และมีสินค้าในหมวด "กาแฟร้อน" อยู่:

```
ERROR:  update or delete on table "categories" violates foreign key constraint
        "products_category_id_fkey" on table "products"
DETAIL:  Key (category_id)=(1) is still referenced from table "products".
```

**วิธีแก้ที่ 1: ลบตารางลูกก่อน (bottom-up)**

```sql
BEGIN;
DELETE FROM order_items
USING products
WHERE order_items.product_id = products.product_id
  AND products.category_id = (SELECT category_id FROM categories WHERE category_name = 'กาแฟร้อน');

DELETE FROM products
WHERE category_id = (SELECT category_id FROM categories WHERE category_name = 'กาแฟร้อน');

DELETE FROM categories WHERE category_name = 'กาแฟร้อน';
COMMIT;
```

**วิธีแก้ที่ 2: กำหนด ON DELETE CASCADE ที่ FK ของ products**

```sql
ALTER TABLE products
    DROP CONSTRAINT products_category_id_fkey,
    ADD CONSTRAINT products_category_id_fkey
        FOREIGN KEY (category_id) REFERENCES categories(category_id)
        ON DELETE CASCADE;
```

จากนั้นการลบ category จะลบ products ที่เกี่ยวข้องให้อัตโนมัติ (แต่ต้องระวัง เพราะจะลบ order_items ต่อไม่ได้ถ้ายังไม่มี CASCADE ที่ order_items ด้วย)

</details>

---

### แบบฝึกหัดที่ 6

อธิบายว่าทำไม `TRUNCATE` ถึงเร็วกว่า `DELETE` ทั้งตารางในเชิงเทคนิค (อย่างน้อย 2 เหตุผล)

<details>
<summary>เฉลย</summary>

1. **ไม่สร้าง row-level WAL log**: `DELETE` ต้องสแกนและมาร์คแต่ละแถวว่าถูกลบ (ตั้งค่า `xmax`) และเขียน WAL record แยกต่อแถว ในขณะที่ `TRUNCATE` เขียน WAL แค่ record เดียวสำหรับทั้ง relation

2. **Deallocate ทั้ง data file ทันที**: `TRUNCATE` ลบพื้นที่ disk block ของตารางทั้งหมดในทีเดียว เหมือนลบไฟล์ทิ้งทั้งไฟล์ ในขณะที่ `DELETE` แค่มาร์คแถวว่าถูกลบ (ยังไม่คืนพื้นที่ disk จริง) ต้องรอ VACUUM มาเก็บกวาดทีหลัง

3. **ไม่ทำงานกับ row-level trigger**: `TRUNCATE` ไม่ fire trigger ระดับแถว (มีแค่ statement-level TRUNCATE trigger) ทำให้ไม่มี overhead จากการเรียก trigger function ต่อแถวเหมือน DELETE

</details>

---

### แบบฝึกหัดที่ 7

เขียนคำสั่ง `TRUNCATE` ที่ใช้กับตาราง `order_items` ให้ CASCADE ไปยังตารางที่เกี่ยวข้อง และรีเซ็ต sequence ของ `order_item_id` กลับไปเริ่มที่ 1

<details>
<summary>เฉลย</summary>

```sql
TRUNCATE TABLE order_items RESTART IDENTITY CASCADE;
```

หมายเหตุ: ในกรณีนี้ `order_items` เป็นตารางลูกสุดในโครงสร้างความสัมพันธ์ (ไม่มีตารางใดอ้างอิงกลับมาที่มัน) ดังนั้น `CASCADE` อาจไม่ทำอะไรเพิ่มเติม แต่การใส่ไว้ก็ไม่ก่อให้เกิดปัญหา และเป็นการเขียนโค้ดที่ปลอดภัยไว้ก่อนในกรณีที่โครงสร้างเปลี่ยนแปลงในอนาคต

</details>

---

### แบบฝึกหัดที่ 8

ตารางต่อไปนี้ถูกต้องหรือไม่ ถ้าไม่ถูกต้องให้แก้ไข

| ประเด็น | DELETE | TRUNCATE |
|---|---|---|
| ใช้ WHERE ได้ | ได้ | ได้ |
| เร็วกว่าเมื่อลบทั้งตาราง | เร็วกว่า | ช้ากว่า |
| ROLLBACK ได้ | ได้ | ไม่ได้ |

<details>
<summary>เฉลย</summary>

ตารางนี้**ผิดทั้ง 3 แถว** ตารางที่ถูกต้องคือ:

| ประเด็น | DELETE | TRUNCATE |
|---|---|---|
| ใช้ WHERE ได้ | ได้ | **ไม่ได้** (TRUNCATE ลบทั้งตารางเท่านั้น ไม่รองรับ WHERE) |
| เร็วกว่าเมื่อลบทั้งตาราง | **ช้ากว่า** | **เร็วกว่า** (TRUNCATE เร็วกว่ามากเพราะไม่ทำงานระดับแถว) |
| ROLLBACK ได้ | ได้ | **ได้เช่นกัน** (PostgreSQL รองรับ TRUNCATE แบบ transactional ต่างจาก DBMS หลายตัว) |

</details>

---

### แบบฝึกหัดที่ 9

ออกแบบ schema เพิ่มเติมสำหรับตาราง `orders` ให้รองรับ soft delete และเขียนคำสั่งสำหรับ:
1. เพิ่มคอลัมน์ที่จำเป็น
2. "ลบ" order_id = 5 แบบ soft delete
3. Query orders ที่ active เท่านั้น (ไม่รวมที่ถูก soft delete)

<details>
<summary>เฉลย</summary>

```sql
-- 1. เพิ่มคอลัมน์
ALTER TABLE orders ADD COLUMN deleted_at TIMESTAMPTZ;

-- 2. Soft delete order_id = 5
UPDATE orders SET deleted_at = now() WHERE order_id = 5;

-- 3. Query เฉพาะ orders ที่ active
SELECT * FROM orders WHERE deleted_at IS NULL;
```

อาจเสริมด้วยการสร้าง view เพื่อความสะดวก:

```sql
CREATE VIEW active_orders AS
SELECT * FROM orders WHERE deleted_at IS NULL;
```

</details>

---

### แบบฝึกหัดที่ 10 (แบบฝึกหัดรวม)

ร้านกาแฟต้องการ "เลิกขาย" ลูกค้าคนหนึ่ง (customer_id = 2) ออกจากระบบอย่างสมบูรณ์ พร้อมข้อมูลคำสั่งซื้อและรายการสินค้าที่เกี่ยวข้องทั้งหมด แต่ต้องการเก็บ log ไว้ว่าลบอะไรไปบ้างเพื่อการตรวจสอบภายหลัง (audit)

ให้ออกแบบและเขียนคำสั่ง SQL ที่ปลอดภัย ครบถ้วน ตั้งแต่การทดสอบไปจนถึงการลบจริง โดยใช้เทคนิคที่เรียนมาในบทนี้ทั้งหมด (transaction ทดสอบ, RETURNING, การจัดการ FK, audit log)

<details>
<summary>เฉลย</summary>

```sql
-- ขั้นตอนที่ 1: สร้างตาราง audit log (ถ้ายังไม่มี)
CREATE TABLE IF NOT EXISTS deletion_audit_log (
    log_id       SERIAL PRIMARY KEY,
    table_name   TEXT NOT NULL,
    record_data  JSONB NOT NULL,
    deleted_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_by   TEXT NOT NULL DEFAULT current_user
);

-- ขั้นตอนที่ 2: เริ่ม transaction เพื่อทดสอบก่อนเสมอ
BEGIN;

-- ขั้นตอนที่ 3: ตรวจสอบข้อมูลที่จะได้รับผลกระทบด้วย SELECT ก่อน
SELECT o.order_id, oi.order_item_id
FROM orders o
JOIN order_items oi ON oi.order_id = o.order_id
WHERE o.customer_id = 2;

-- ขั้นตอนที่ 4: ลบ order_items ที่เกี่ยวข้อง พร้อมเก็บ audit log ด้วย Data-Modifying CTE
WITH deleted_items AS (
    DELETE FROM order_items
    USING orders
    WHERE order_items.order_id = orders.order_id
      AND orders.customer_id = 2
    RETURNING order_items.*
)
INSERT INTO deletion_audit_log (table_name, record_data)
SELECT 'order_items', to_jsonb(deleted_items)
FROM deleted_items;

-- ขั้นตอนที่ 5: ลบ orders ของลูกค้ารายนี้ พร้อมเก็บ audit log
WITH deleted_orders AS (
    DELETE FROM orders
    WHERE customer_id = 2
    RETURNING *
)
INSERT INTO deletion_audit_log (table_name, record_data)
SELECT 'orders', to_jsonb(deleted_orders)
FROM deleted_orders;

-- ขั้นตอนที่ 6: ลบข้อมูลลูกค้า พร้อมเก็บ audit log
WITH deleted_customer AS (
    DELETE FROM customers
    WHERE customer_id = 2
    RETURNING *
)
INSERT INTO deletion_audit_log (table_name, record_data)
SELECT 'customers', to_jsonb(deleted_customer)
FROM deleted_customer;

-- ขั้นตอนที่ 7: ตรวจสอบผลลัพธ์ทั้งหมดก่อนตัดสินใจ
SELECT * FROM deletion_audit_log ORDER BY log_id DESC LIMIT 10;

-- ขั้นตอนที่ 8: หากทุกอย่างถูกต้องตามที่คาดไว้ จึง COMMIT
COMMIT;

-- ถ้าพบว่าผลลัพธ์ผิดพลาดในขั้นตอนใดขั้นตอนหนึ่ง ให้ ROLLBACK แทน:
-- ROLLBACK;
```

**แนวทางออกแบบทางเลือก (Soft Delete แทน Hard Delete):**

หากร้านกาแฟต้องการเก็บข้อมูลไว้เผื่อกู้คืนในอนาคต (เช่น ลูกค้าติดต่อขอกลับมาใช้งานใหม่) อาจเลือกใช้ soft delete แทนตั้งแต่ต้น:

```sql
ALTER TABLE customers ADD COLUMN IF NOT EXISTS deleted_at TIMESTAMPTZ;

BEGIN;

UPDATE customers
SET deleted_at = now()
WHERE customer_id = 2
RETURNING customer_id, full_name, deleted_at;

COMMIT;
```

วิธีนี้ปลอดภัยกว่า ไม่ต้องกังวลเรื่อง FK constraint เลย เพราะไม่มีการลบข้อมูลจริง แต่ต้องแลกกับการที่ทุก query ในระบบต้องเพิ่มเงื่อนไข `WHERE deleted_at IS NULL` เสมอ

**บทเรียนสำคัญจากแบบฝึกหัดนี้:** การลบข้อมูลจริงในระบบ production ควรผ่านกระบวนการคิดหลายชั้น — ตั้งแต่การเลือกว่าจะ hard delete หรือ soft delete, การทดสอบด้วย transaction, การเก็บ audit trail, ไปจนถึงการจัดการ FK อย่างเป็นระบบ ไม่ใช่แค่การรันคำสั่ง `DELETE` เพียงบรรทัดเดียว

</details>

---

## บทถัดไป

เรียนรู้เรื่องการจัดการค่า NULL อย่างละเอียดใน PostgreSQL ต่อได้ที่: [Part 015 — NULL Handling](./part-015-null-handling.md)
