# Part 015: NULL คืออะไร และการจัดการ NULL อย่างถูกต้อง

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับพื้นฐาน | Part 015

## เป้าหมายการเรียนรู้

เมื่อเรียนจบบทนี้ ผู้เรียนจะสามารถ:

- อธิบายแนวคิดของ `NULL` ในฐานะ "unknown value" ได้อย่างถูกต้อง แยกแยะจาก `0`, `''` (empty string) และ `false`
- เข้าใจ three-valued logic (`TRUE` / `FALSE` / `UNKNOWN`) และผลกระทบต่อเงื่อนไขใน `WHERE`, `AND`, `OR`, `NOT`
- อธิบายได้ว่าทำไม `NULL = NULL` จึงให้ผลเป็น `NULL` ไม่ใช่ `TRUE` และเขียนเงื่อนไขด้วย `IS NULL` / `IS NOT NULL` ได้ถูกต้อง
- เข้าใจพฤติกรรมของ aggregate function (`COUNT`, `SUM`, `AVG`, `MIN`, `MAX`) เมื่อพบ `NULL` และแยกความแตกต่างระหว่าง `COUNT(*)` กับ `COUNT(column)`
- ใช้ `COALESCE` เพื่อกำหนดค่า default แทน `NULL`
- ใช้ `NULLIF` เพื่อแปลงค่าที่ไม่ต้องการให้กลายเป็น `NULL` เช่น การป้องกัน division by zero
- เข้าใจผลกระทบของ `NULL` ต่อ `JOIN` และ `ORDER BY`
- อธิบายได้ว่าทำไม `UNIQUE` constraint ใน PostgreSQL จึงยอมให้มีหลายแถวที่เป็น `NULL` ได้
- ใช้ `IS DISTINCT FROM` / `IS NOT DISTINCT FROM` เพื่อเปรียบเทียบค่าที่ปลอดภัยต่อ `NULL`
- ตัดสินใจได้ว่าเมื่อไหร่ควรออกแบบคอลัมน์ให้อนุญาต `NULL` และเมื่อไหร่ควรใช้ `NOT NULL` ร่วมกับ `DEFAULT` แทน

---

## เตรียมข้อมูล

บทนี้ยังคงใช้ schema ร้านกาแฟ (coffee shop) ที่เราใช้มาตลอดทั้งหลักสูตร แต่จะเพิ่มข้อมูลที่ "ขาดหาย" (เช่น ลูกค้าที่ไม่มีเบอร์โทร หรือไม่มีอีเมล) เพื่อให้เห็นพฤติกรรมของ `NULL` ชัดเจนขึ้น

เปิด `psql` แล้วรันคำสั่งต่อไปนี้เพื่อสร้างฐานข้อมูลตัวอย่างใหม่ (หรือ `DROP` ตารางเดิมก่อนถ้ามีอยู่แล้ว):

```sql
-- ล้างตารางเดิมถ้ามี (เรียงตามลำดับ dependency)
DROP TABLE IF EXISTS order_items;
DROP TABLE IF EXISTS orders;
DROP TABLE IF EXISTS products;
DROP TABLE IF EXISTS customers;
DROP TABLE IF EXISTS categories;

-- ตารางหมวดหมู่สินค้า
CREATE TABLE categories (
    category_id   SERIAL PRIMARY KEY,
    category_name VARCHAR(50) NOT NULL,
    description   TEXT
);

-- ตารางสินค้า
CREATE TABLE products (
    product_id    SERIAL PRIMARY KEY,
    category_id   INTEGER REFERENCES categories(category_id),
    product_name  VARCHAR(100) NOT NULL,
    price         NUMERIC(10, 2) NOT NULL,
    cost          NUMERIC(10, 2),          -- ต้นทุน: บางสินค้ายังไม่มีข้อมูลต้นทุน
    stock_qty     INTEGER DEFAULT 0,
    discontinued_at DATE                   -- วันที่เลิกขาย: ยังขายอยู่ = NULL
);

-- ตารางลูกค้า
CREATE TABLE customers (
    customer_id   SERIAL PRIMARY KEY,
    full_name     VARCHAR(100) NOT NULL,
    email         VARCHAR(100) UNIQUE,     -- ลูกค้าบางคนไม่ให้อีเมล
    phone         VARCHAR(20),             -- ลูกค้าบางคนไม่ให้เบอร์โทร
    member_since  DATE DEFAULT CURRENT_DATE,
    referred_by   INTEGER REFERENCES customers(customer_id)  -- ใครแนะนำมา (ถ้ามี)
);

-- ตารางคำสั่งซื้อ
CREATE TABLE orders (
    order_id      SERIAL PRIMARY KEY,
    customer_id   INTEGER REFERENCES customers(customer_id),
    order_date    TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    shipped_at    TIMESTAMP,               -- ยังไม่จัดส่ง = NULL
    notes         TEXT,
    discount_pct  NUMERIC(5, 2)            -- ส่วนลด: ไม่มีส่วนลด = NULL (ไม่ใช่ 0)
);

-- ตารางรายการสินค้าในคำสั่งซื้อ
CREATE TABLE order_items (
    order_item_id SERIAL PRIMARY KEY,
    order_id      INTEGER NOT NULL REFERENCES orders(order_id),
    product_id    INTEGER NOT NULL REFERENCES products(product_id),
    quantity      INTEGER NOT NULL CHECK (quantity > 0),
    unit_price    NUMERIC(10, 2) NOT NULL
);
```

จากนั้นใส่ข้อมูลตัวอย่าง โดยตั้งใจใส่ `NULL` ไว้หลายจุดเพื่อสาธิตปัญหา:

```sql
-- หมวดหมู่สินค้า
INSERT INTO categories (category_name, description) VALUES
    ('กาแฟร้อน', 'เครื่องดื่มกาแฟเสิร์ฟร้อน'),
    ('กาแฟเย็น', 'เครื่องดื่มกาแฟเสิร์ฟเย็น/ปั่น'),
    ('เบเกอรี่', NULL),                      -- ยังไม่ได้เขียนคำอธิบาย
    ('ชา', 'เครื่องดื่มชาแบบต่าง ๆ');

-- สินค้า: บางตัวไม่มีข้อมูล cost, บางตัวเลิกขายแล้ว
INSERT INTO products (category_id, product_name, price, cost, stock_qty, discontinued_at) VALUES
    (1, 'เอสเพรสโซ่',       45.00, 12.00, 100, NULL),
    (1, 'อเมริกาโน่ร้อน',    55.00, 15.00, 80,  NULL),
    (1, 'ลาเต้ร้อน',        60.00, NULL,  60,  NULL),   -- ยังไม่คำนวณต้นทุน
    (2, 'อเมริกาโน่เย็น',    60.00, 15.00, 90,  NULL),
    (2, 'ลาเต้เย็น',        65.00, 18.00, 70,  NULL),
    (2, 'โมค่าเย็น',        70.00, NULL,  40,  NULL),   -- ยังไม่คำนวณต้นทุน
    (3, 'ครัวซองต์',        55.00, 22.00, 20,  '2025-01-15'),  -- เลิกขายแล้ว
    (3, 'บราวนี่',          45.00, 18.00, 30,  NULL),
    (4, 'ชาเขียว',          50.00, 10.00, 50,  NULL),
    (4, 'ชามะนาว',          45.00, NULL,  0,   NULL);  -- หมดสต็อก และไม่มีต้นทุน

-- ลูกค้า: บางคนไม่มีอีเมล บางคนไม่มีเบอร์โทร
INSERT INTO customers (full_name, email, phone, member_since, referred_by) VALUES
    ('สมชาย ใจดี',       'somchai@example.com', '0812345678', '2024-01-10', NULL),
    ('สมหญิง รักเรียน',   'somying@example.com', NULL,          '2024-02-15', 1),
    ('วิชัย มั่งมี',       NULL,                   '0898765432', '2024-03-20', NULL),
    ('มานี ดีใจ',         'manee@example.com',    '0855555555', '2024-04-01', 1),
    ('ประยุทธ ขยันทำงาน', NULL,                   NULL,          '2024-05-12', NULL);  -- ไม่มีทั้งอีเมลและเบอร์โทร

-- คำสั่งซื้อ: บางออเดอร์ยังไม่จัดส่ง บางออเดอร์ไม่มีส่วนลด (NULL ไม่ใช่ 0)
INSERT INTO orders (customer_id, order_date, shipped_at, notes, discount_pct) VALUES
    (1, '2025-06-01 09:15:00', '2025-06-01 09:30:00', NULL,              10.00),
    (2, '2025-06-01 10:00:00', NULL,                   'รอลูกค้ามารับเอง', NULL),
    (3, '2025-06-02 08:45:00', '2025-06-02 09:00:00', NULL,              NULL),
    (1, '2025-06-03 14:20:00', NULL,                   'ลูกค้าประจำ',     5.00),
    (4, '2025-06-03 16:10:00', '2025-06-03 16:25:00', NULL,              NULL);

-- รายการสินค้าในแต่ละคำสั่งซื้อ
INSERT INTO order_items (order_id, product_id, quantity, unit_price) VALUES
    (1, 1, 2, 45.00),
    (1, 8, 1, 45.00),
    (2, 5, 1, 65.00),
    (3, 3, 3, 60.00),
    (4, 2, 1, 55.00),
    (4, 9, 2, 50.00),
    (5, 4, 2, 60.00);
```

ตรวจสอบว่าข้อมูลถูกใส่ครบ:

```sql
SELECT * FROM customers;
```

```
 customer_id |     full_name      |        email          |   phone     | member_since | referred_by
-------------+---------------------+------------------------+-------------+--------------+-------------
           1 | สมชาย ใจดี         | somchai@example.com   | 0812345678  | 2024-01-10   |
           2 | สมหญิง รักเรียน    | somying@example.com   |             | 2024-02-15   |           1
           3 | วิชัย มั่งมี        |                        | 0898765432  | 2024-03-20   |
           4 | มานี ดีใจ          | manee@example.com     | 0855555555  | 2024-04-01   |           1
           5 | ประยุทธ ขยันทำงาน  |                        |             | 2024-05-12   |
(5 rows)
```

สังเกตว่าช่องว่างในผลลัพธ์ที่เห็น "ไม่มีอะไรเลย" นั้นคือ `NULL` — ไม่ใช่ string ว่าง `''` และไม่ใช่ตัวเลข `0` เราจะมาดูกันว่าความแตกต่างนี้สำคัญแค่ไหน

---

## Step 141: NULL คืออะไร — แนวคิด "unknown value" ไม่ใช่ 0 หรือ empty string

หลายคนที่เริ่มเรียน SQL มักเข้าใจผิดว่า `NULL` คือ "ค่าว่าง" (empty) หรือ "ศูนย์" (zero) แต่ในความเป็นจริง **`NULL` หมายถึง "ไม่ทราบค่า" (unknown value)** หรือ "ไม่มีค่านี้อยู่" (absence of a value) ซึ่งเป็นแนวคิดที่ต่างจากค่าว่างหรือศูนย์โดยสิ้นเชิง

ลองดูตัวอย่างที่ทำให้ความแตกต่างชัดเจน:

- `phone = NULL` หมายถึง "เราไม่รู้ว่าเบอร์โทรของลูกค้าคนนี้คืออะไร" (อาจจะมีเบอร์ แต่เราไม่ได้บันทึกไว้)
- `phone = ''` หมายถึง "เรารู้ว่าค่านี้คือ string ว่าง" — เป็นค่าที่ชัดเจน เพียงแต่ไม่มีตัวอักษรใด ๆ
- `stock_qty = NULL` หมายถึง "เราไม่ทราบจำนวนสต็อก"
- `stock_qty = 0` หมายถึง "เรารู้แน่ชัดว่าสต็อกเหลือ 0 ชิ้น"

ทั้งสองแบบนี้มีความหมายต่างกันโดยสิ้นเชิงในเชิงธุรกิจ และ PostgreSQL ก็ปฏิบัติต่อมันต่างกันด้วย มาดูตัวอย่างจริง:

```sql
-- เปรียบเทียบ NULL, empty string และ 0
SELECT
    full_name,
    email,
    email IS NULL       AS email_is_null,
    email = ''           AS email_is_empty_string,
    length(email)        AS email_length
FROM customers
ORDER BY customer_id;
```

```
     full_name      |        email          | email_is_null | email_is_empty_string | email_length
---------------------+------------------------+----------------+------------------------+--------------
 สมชาย ใจดี         | somchai@example.com   | f              |                        |           20
 สมหญิง รักเรียน    | somying@example.com   | f              |                        |           20
 วิชัย มั่งมี        |                        | t              |                        |
 มานี ดีใจ          | manee@example.com     | f              |                        |           15
 ประยุทธ ขยันทำงาน  |                        | t              |                        |
(5 rows)
```

สังเกตสองอย่างสำคัญ:

1. คอลัมน์ `email_is_empty_string` (`email = ''`) ให้ผลเป็น `NULL` (แสดงเป็นช่องว่าง) สำหรับแถวที่ `email IS NULL` เพราะการเปรียบเทียบใด ๆ กับ `NULL` จะได้ผลเป็น `NULL` เสมอ (เราจะอธิบายละเอียดใน Step 143)
2. `length(email)` ก็ให้ผลเป็น `NULL` เช่นกัน เพราะฟังก์ชันเกือบทั้งหมดที่รับ `NULL` เป็น input จะคืนค่า `NULL` ออกมา — ไม่ใช่ `0`

ลองเปรียบเทียบกับกรณีที่ใส่ string ว่างจริง ๆ:

```sql
-- สร้างตารางทดสอบเล็ก ๆ เพื่อเทียบ NULL กับ empty string
CREATE TEMP TABLE test_values (label TEXT, val TEXT);
INSERT INTO test_values VALUES
    ('unknown (NULL)', NULL),
    ('empty string',   '');

SELECT
    label,
    val,
    val IS NULL   AS is_null,
    length(val)   AS len,
    val = ''      AS equals_empty
FROM test_values;
```

```
      label      | val | is_null | len | equals_empty
------------------+-----+---------+-----+--------------
 unknown (NULL)   |     | t       |     |
 empty string     |     | f       |   0 | t
(2 rows)
```

จะเห็นว่า:
- แถว `unknown (NULL)`: `is_null = t`, `length = NULL`, `equals_empty = NULL` (ไม่ทราบ เพราะเราไม่รู้ค่าจริง)
- แถว `empty string`: `is_null = f`, `length = 0`, `equals_empty = t` (เรารู้ว่ามันคือ string ว่างแน่นอน)

**บทสรุปสำคัญ:** เวลาออกแบบฐานข้อมูล ต้องแยกให้ชัดว่า "ไม่มีข้อมูล" (`NULL`) กับ "มีข้อมูลแต่เป็นค่าว่าง/ศูนย์" (`''` หรือ `0`) คือคนละความหมายกัน การใช้ผิดจะทำให้การ query และ business logic ผิดเพี้ยนตามไปด้วย

---

## Step 142: Three-valued logic (TRUE/FALSE/UNKNOWN) และผลกระทบต่อ WHERE, AND/OR/NOT

SQL ไม่ได้ใช้ตรรกะแบบ boolean ธรรมดา (TRUE/FALSE) เหมือนภาษาโปรแกรมทั่วไป แต่ใช้ **three-valued logic (3VL)** ที่มีสามสถานะ:

- `TRUE`
- `FALSE`
- `UNKNOWN` (แทนด้วย `NULL` เวลาผลลัพธ์เป็นเช่นนี้)

เมื่อไหร่ก็ตามที่มีการเปรียบเทียบที่เกี่ยวข้องกับ `NULL` ผลลัพธ์จะเป็น `UNKNOWN` เสมอ และ `WHERE` clause จะคัดเลือกเฉพาะแถวที่เงื่อนไขเป็น `TRUE` เท่านั้น — แถวที่ได้ `UNKNOWN` หรือ `FALSE` จะถูกตัดทิ้งทั้งคู่

ตารางความจริงของ `AND`:

| A \ B | TRUE | FALSE | UNKNOWN |
|---|---|---|---|
| **TRUE** | TRUE | FALSE | UNKNOWN |
| **FALSE** | FALSE | FALSE | FALSE |
| **UNKNOWN** | UNKNOWN | FALSE | UNKNOWN |

ตารางความจริงของ `OR`:

| A \ B | TRUE | FALSE | UNKNOWN |
|---|---|---|---|
| **TRUE** | TRUE | TRUE | TRUE |
| **FALSE** | TRUE | FALSE | UNKNOWN |
| **UNKNOWN** | TRUE | UNKNOWN | UNKNOWN |

ตารางความจริงของ `NOT`:

| A | NOT A |
|---|---|
| TRUE | FALSE |
| FALSE | TRUE |
| UNKNOWN | UNKNOWN |

มาดูตัวอย่างจริงที่แสดงให้เห็นว่าทำไมเรื่องนี้ถึงสำคัญมาก:

```sql
-- ต้องการหาลูกค้าที่ "ไม่มีเบอร์โทร หรือ เบอร์โทรขึ้นต้นด้วย 08"
SELECT full_name, phone
FROM customers
WHERE phone = '' OR phone LIKE '08%';
```

```
 full_name | phone
-----------+-------
(0 rows)
```

ผลลัพธ์ว่างเปล่า! ทั้งที่เรามีลูกค้าที่เบอร์ขึ้นต้นด้วย 08 อยู่ถึง 2 คน ปัญหาคือ `phone = ''` เทียบกับ `NULL` (ของ วิชัย, ประยุทธ) จะได้ `UNKNOWN` ไม่ใช่ `TRUE` — แต่ปัญหาจริง ๆ คือคำสั่งนี้ผิดตั้งแต่ต้น เพราะควรใช้ `IS NULL` ไม่ใช่ `= ''` มาดูว่าทำไมถึงได้ผลว่างทั้งที่มีข้อมูลตรงเงื่อนไข:

```sql
-- ตรวจสอบทีละเงื่อนไข
SELECT
    full_name,
    phone,
    (phone = '')       AS cond1,
    (phone LIKE '08%') AS cond2,
    (phone = '' OR phone LIKE '08%') AS combined
FROM customers;
```

```
     full_name      |    phone    | cond1 | cond2 | combined
---------------------+-------------+-------+-------+----------
 สมชาย ใจดี         | 0812345678  | f     | t     | t
 สมหญิง รักเรียน    |             |       |       |
 วิชัย มั่งมี        | 0898765432  | f     | t     | t
 มานี ดีใจ          | 0855555555  | f     | t     | t
 ประยุทธ ขยันทำงาน  |             |       |       |
(5 rows)
```

เอ๊ะ! รอบนี้กลับมีผลลัพธ์ 3 แถวที่ `combined = t` ถูกต้อง — แสดงว่าคำสั่งก่อนหน้าที่ได้ 0 rows นั้นผิดพลาดในการอธิบาย ลองรันคำสั่ง `WHERE` เดิมอีกครั้งเพื่อตรวจสอบ:

```sql
SELECT full_name, phone
FROM customers
WHERE phone = '' OR phone LIKE '08%';
```

```
     full_name      |    phone
---------------------+-------------
 สมชาย ใจดี         | 0812345678
 วิชัย มั่งมี        | 0898765432
 มานี ดีใจ          | 0855555555
(3 rows)
```

ที่จริงคำสั่งนี้ได้ผลลัพธ์ 3 แถวถูกต้อง (เพราะ `phone LIKE '08%'` เป็น `TRUE` สำหรับ 3 คนนั้น และ `OR` กับ `UNKNOWN` ของอีกเงื่อนไขไม่กระทบ) แต่สิ่งที่ **น่ากลัวกว่า** คือกรณีตรงข้าม — การหาลูกค้าที่ "**ไม่มี**" เบอร์โทร:

```sql
-- ความตั้งใจ: หาลูกค้าที่ไม่มีเบอร์โทร (ผิด! ใช้ = แทน IS)
SELECT full_name, phone
FROM customers
WHERE phone = NULL;
```

```
 full_name | phone
-----------+-------
(0 rows)
```

**ได้ 0 แถวเสมอ ไม่ว่าฐานข้อมูลจะมีข้อมูลกี่แถวก็ตาม** เพราะ `phone = NULL` จะให้ผลเป็น `UNKNOWN` สำหรับทุกแถว (แม้แถวที่ `phone` เป็น `NULL` จริง ๆ ก็ตาม) และ `WHERE` จะไม่เลือกแถวที่เป็น `UNKNOWN` เลย เรื่องนี้จะอธิบายลึกลงไปใน Step 143

ตัวอย่างผลกระทบของ `NOT` ร่วมกับ `UNKNOWN`:

```sql
-- ต้องการหาลูกค้าที่ "ไม่ใช่" คนที่มีคนแนะนำเป็น customer_id = 1
SELECT full_name, referred_by
FROM customers
WHERE NOT (referred_by = 1);
```

```
 full_name | referred_by
-----------+-------------
(0 rows)
```

ผลลัพธ์ว่างเปล่าอีกแล้ว! เพราะแถวที่ `referred_by = 1` ตรงจริงจะถูก `NOT` กลับเป็น `FALSE` และถูกตัดออก ส่วนแถวที่ `referred_by` เป็น `NULL` จะได้ `referred_by = 1` เป็น `UNKNOWN` และ `NOT UNKNOWN` ก็ยังเป็น `UNKNOWN` อยู่ดี จึงไม่ถูกเลือกเช่นกัน แถวที่ `referred_by` เป็นค่าอื่น (ไม่ใช่ 1 และไม่ใช่ NULL) ก็ไม่มีในตัวอย่างนี้ ทำให้ผลลัพธ์เป็น 0 แถวทั้งที่ควรจะได้ลูกค้าที่ `referred_by IS NULL` ด้วย

วิธีเขียนที่ถูกต้องคือต้องจัดการ `NULL` แยกออกมาอย่างชัดเจน:

```sql
SELECT full_name, referred_by
FROM customers
WHERE referred_by IS DISTINCT FROM 1;
```

```
     full_name      | referred_by
---------------------+-------------
 สมชาย ใจดี         |
 วิชัย มั่งมี        |
 ประยุทธ ขยันทำงาน  |
(3 rows)
```

(เราจะเรียนเรื่อง `IS DISTINCT FROM` โดยละเอียดใน Step 149)

**สรุป Step 142:** สาม-สถานะของ SQL logic ทำให้เงื่อนไขที่ดู "ตรงไปตรงมา" ในภาษาโปรแกรมทั่วไป อาจให้ผลลัพธ์ผิดคาดเมื่อมี `NULL` เข้ามาเกี่ยวข้อง ต้องคิดถึง `UNKNOWN` เป็นสถานะที่สามเสมอเมื่อออกแบบเงื่อนไขใน `WHERE`

---

## Step 143: ทำไม NULL = NULL ให้ผลเป็น NULL (ไม่ใช่ TRUE) และการใช้ IS NULL/IS NOT NULL ที่ถูกต้อง

หัวใจของปัญหาทั้งหมดในบทนี้อยู่ที่กฎข้อนี้: **การเปรียบเทียบใด ๆ กับ `NULL` โดยใช้ operator เปรียบเทียบทั่วไป (`=`, `<>`, `<`, `>`, `<=`, `>=`) จะให้ผลเป็น `NULL` เสมอ ไม่ว่าจะเทียบกับอะไรก็ตาม**

เหตุผลเชิงแนวคิด: `NULL` แปลว่า "ไม่ทราบค่า" ดังนั้นคำถาม "ค่าที่ไม่ทราบ เท่ากับ ค่าที่ไม่ทราบอีกตัวหนึ่งหรือไม่?" คำตอบที่ถูกต้องคือ "ไม่ทราบ" (`UNKNOWN`) — เพราะเราไม่มีทางรู้ได้ว่าค่าที่ไม่ทราบสองค่านั้นเหมือนกันจริงหรือไม่

```sql
-- ทดสอบการเปรียบเทียบกับ NULL โดยตรง
SELECT
    NULL = NULL       AS null_eq_null,
    NULL <> NULL       AS null_neq_null,
    NULL = 5           AS null_eq_5,
    5 <> NULL           AS five_neq_null,
    NULL IS NULL        AS null_is_null,
    NULL IS NOT NULL    AS null_is_not_null;
```

```
 null_eq_null | null_neq_null | null_eq_5 | five_neq_null | null_is_null | null_is_not_null
--------------+---------------+-----------+---------------+--------------+-------------------
              |               |           |               | t            | f
(1 row)
```

สังเกตว่า 4 คอลัมน์แรก (ที่ใช้ `=` หรือ `<>`) ทั้งหมดให้ผลเป็น `NULL` (ช่องว่าง) ในขณะที่ 2 คอลัมน์หลัง (ที่ใช้ `IS NULL` / `IS NOT NULL`) ให้ผลเป็น boolean จริง (`t`/`f`) — นี่คือเหตุผลว่าทำไม PostgreSQL (และ SQL มาตรฐาน) จึงต้องมี operator พิเศษ `IS NULL` และ `IS NOT NULL` ที่ **ไม่ใช่** การเปรียบเทียบค่าแบบธรรมดา แต่เป็นการ "ตรวจสอบสถานะ" ของค่านั้นโดยตรง ซึ่งให้ผลเป็น `TRUE`/`FALSE` เสมอ ไม่มีทาง `UNKNOWN`

ลองเปรียบเทียบวิธีที่ผิดกับวิธีที่ถูกในการหาลูกค้าที่ไม่มีอีเมล:

```sql
-- ❌ ผิด: ใช้ = NULL จะไม่เจอผลลัพธ์อะไรเลย
SELECT full_name FROM customers WHERE email = NULL;
```

```
 full_name
-----------
(0 rows)
```

```sql
-- ✅ ถูก: ใช้ IS NULL
SELECT full_name FROM customers WHERE email IS NULL;
```

```
     full_name
---------------------
 วิชัย มั่งมี
 ประยุทธ ขยันทำงาน
(2 rows)
```

และการหาลูกค้าที่ "มี" อีเมล:

```sql
-- ❌ ผิด: <> NULL ก็ไม่เจอผลลัพธ์เช่นกัน
SELECT full_name FROM customers WHERE email <> NULL;
```

```
 full_name
-----------
(0 rows)
```

```sql
-- ✅ ถูก: ใช้ IS NOT NULL
SELECT full_name FROM customers WHERE email IS NOT NULL;
```

```
     full_name
---------------------
 สมชาย ใจดี
 สมหญิง รักเรียน
 มานี ดีใจ
(3 rows)
```

PostgreSQL ยังเตือนเรื่องนี้ให้เราด้วย ถ้าเปิด warning ที่เกี่ยวข้อง (แม้ default จะไม่ error แต่ `= NULL` มักถูกมองว่าเป็นสัญญาณของ bug) — เครื่องมือ linter สำหรับ SQL หลายตัว (เช่น sqlfluff) จะ flag `= NULL` เป็นข้อผิดพลาดโดยอัตโนมัติ

**ข้อควรระวังเพิ่มเติม:** operator `IS [NOT] NULL` ใช้ได้กับทุก data type รวมถึงใช้ตรวจสอบทั้งแถวได้ด้วย (row-wise NULL check) แต่ในระดับเบื้องต้นให้จำหลักง่าย ๆ ไว้ก่อนว่า:

> เมื่อไหร่ก็ตามที่ต้องการเช็คว่าคอลัมน์มีค่าหรือไม่มีค่า **ห้ามใช้ `=` หรือ `<>` กับ `NULL` เด็ดขาด ให้ใช้ `IS NULL` / `IS NOT NULL` เสมอ**

ลองดูตัวอย่างที่ซับซ้อนขึ้นอีกนิด — การกรองสินค้าที่ "ยังไม่เลิกขาย":

```sql
-- สินค้าที่ยังขายอยู่ (discontinued_at เป็น NULL)
SELECT product_name, discontinued_at
FROM products
WHERE discontinued_at IS NULL
ORDER BY product_id;
```

```
   product_name    | discontinued_at
--------------------+-----------------
 เอสเพรสโซ่        |
 อเมริกาโน่ร้อน     |
 ลาเต้ร้อน          |
 อเมริกาโน่เย็น     |
 ลาเต้เย็น          |
 โมค่าเย็น          |
 บราวนี่            |
 ชาเขียว            |
 ชามะนาว            |
(9 rows)
```

ครัวซองต์ (ที่เลิกขายไปแล้วเมื่อ `2025-01-15`) ไม่ปรากฏในผลลัพธ์ ซึ่งถูกต้องตามที่ตั้งใจ

---

## Step 144: NULL ใน aggregate functions (COUNT, SUM, AVG ข้าม NULL อย่างไร) และ COUNT(*) vs COUNT(column)

Aggregate functions ใน PostgreSQL (`COUNT`, `SUM`, `AVG`, `MIN`, `MAX` ฯลฯ) มีกฎสำคัญข้อหนึ่งคือ **จะข้าม (ignore) ค่า `NULL` โดยอัตโนมัติ** ยกเว้น `COUNT(*)` ที่นับทุกแถวไม่สนว่าจะมี `NULL` หรือไม่

มาดูตัวอย่างจากตาราง `products` ที่มีคอลัมน์ `cost` เป็น `NULL` อยู่ 3 แถว (ลาเต้ร้อน, โมค่าเย็น, ชามะนาว):

```sql
SELECT
    COUNT(*)      AS total_rows,
    COUNT(cost)    AS rows_with_cost,
    SUM(cost)       AS sum_cost,
    AVG(cost)       AS avg_cost,
    MIN(cost)       AS min_cost,
    MAX(cost)       AS max_cost
FROM products;
```

```
 total_rows | rows_with_cost | sum_cost | avg_cost           | min_cost | max_cost
------------+-----------------+----------+---------------------+----------+----------
         10 |               7 |   110.00 | 15.7142857142857143 |    10.00 |    22.00
(1 row)
```

สังเกตว่า:

- `COUNT(*) = 10` — นับทุกแถวในตาราง ไม่สนใจว่ามี `NULL` หรือไม่
- `COUNT(cost) = 7` — นับเฉพาะแถวที่ `cost IS NOT NULL` (10 แถว - 3 แถวที่เป็น NULL = 7)
- `SUM(cost)`, `AVG(cost)`, `MIN(cost)`, `MAX(cost)` — คำนวณจากเฉพาะ 7 แถวที่มีค่า ไม่ได้เอา `NULL` มาคิดเป็น 0

นี่คือจุดที่คนมักเข้าใจผิดบ่อยมาก เพราะคิดว่า `AVG` จะเอา `NULL` มาคิดเป็น `0` ด้วย ทำให้ค่าเฉลี่ยต่ำลง แต่ในความเป็นจริง **`NULL` จะถูกข้ามไปเลย ไม่ถูกนำมาคิดในตัวหารด้วยซ้ำ**

ลองเปรียบเทียบให้เห็นชัดกว่านี้ด้วยการคำนวณค่าเฉลี่ยแบบ manual:

```sql
-- ถ้า AVG นับ NULL เป็น 0 ค่าเฉลี่ยจะเท่ากับ 110.00 / 10 = 11.00
-- แต่ AVG จริง ๆ ข้าม NULL ออกจากทั้งตัวเศษและตัวหาร: 110.00 / 7 = 15.71...
SELECT
    SUM(cost) / COUNT(*)     AS if_null_counted_as_zero,   -- ผิด แนวคิดสมมติ
    SUM(cost) / COUNT(cost)  AS correct_average_manual,     -- เทียบเท่ากับ AVG(cost) จริง
    AVG(cost)                 AS avg_cost_function
FROM products;
```

```
 if_null_counted_as_zero | correct_average_manual | avg_cost_function
--------------------------+-------------------------+---------------------
                    11.00 |    15.7142857142857143 | 15.7142857142857143
(1 row)
```

ชัดเจนว่า `AVG(cost)` เท่ากับการหาร `SUM(cost) / COUNT(cost)` (เอา `NULL` ออกทั้งเศษและส่วน) ไม่ใช่ `SUM(cost) / COUNT(*)`

### กรณี COUNT(*) vs COUNT(column) ในบริบทจริง

ลองดูตัวอย่างที่ใช้ประโยชน์จากพฤติกรรมนี้ — นับจำนวนลูกค้าทั้งหมด เทียบกับจำนวนลูกค้าที่มีอีเมล:

```sql
SELECT
    COUNT(*)     AS total_customers,
    COUNT(email)  AS customers_with_email,
    COUNT(phone)  AS customers_with_phone,
    COUNT(*) - COUNT(email) AS customers_without_email
FROM customers;
```

```
 total_customers | customers_with_email | customers_with_phone | customers_without_email
------------------+------------------------+------------------------+---------------------------
                5 |                      3 |                      3 |                        2
(1 row)
```

ค่านี้มีประโยชน์มากในการทำ data quality report — สามารถหาเปอร์เซ็นต์ของข้อมูลที่ "ขาดหาย" ได้ทันที:

```sql
SELECT
    COUNT(*) AS total,
    COUNT(email) AS with_email,
    ROUND(100.0 * COUNT(email) / COUNT(*), 1) AS pct_with_email
FROM customers;
```

```
 total | with_email | pct_with_email
-------+-------------+-----------------
     5 |           3 |            60.0
(1 row)
```

### ข้อควรระวัง: GROUP BY ก็ปฏิบัติต่อ NULL เป็น "กลุ่มเดียวกัน"

แม้ `NULL = NULL` จะเป็น `UNKNOWN` แต่ `GROUP BY` และ `DISTINCT` จะถือว่าแถวที่มี `NULL` ทั้งหมดอยู่ใน "กลุ่มเดียวกัน" (นี่เป็นกฎพิเศษที่ SQL standard กำหนดไว้ ต่างจาก `WHERE ... = NULL` โดยเจตนา):

```sql
-- จัดกลุ่มคำสั่งซื้อตาม discount_pct (มีทั้งค่าจริงและ NULL)
SELECT
    discount_pct,
    COUNT(*) AS order_count
FROM orders
GROUP BY discount_pct
ORDER BY discount_pct NULLS FIRST;
```

```
 discount_pct | order_count
--------------+-------------
              |           3
        5.00 |           1
       10.00 |           1
(3 rows)
```

`NULL` ทั้ง 3 แถวถูกจัดเป็นกลุ่มเดียวกัน (แสดงเป็นแถวเดียวในผลลัพธ์) ซึ่งเป็นพฤติกรรมที่ตั้งใจไว้เพื่อให้ `GROUP BY` ใช้งานได้จริงในทางปฏิบัติ

**สรุป Step 144:** อย่าลืมว่า aggregate function ข้าม `NULL` เสมอ ยกเว้น `COUNT(*)` การรู้พฤติกรรมนี้ช่วยให้เขียน query สำหรับวิเคราะห์ data quality และคำนวณสถิติได้อย่างถูกต้อง

---

## Step 145: COALESCE — การกำหนดค่า default แทน NULL

`COALESCE(val1, val2, ..., valN)` เป็นฟังก์ชันที่คืนค่า **แรกที่ไม่ใช่ `NULL`** จาก argument ที่ส่งเข้าไป ไล่จากซ้ายไปขวา ถ้าทุกค่าเป็น `NULL` ทั้งหมด จะคืน `NULL`

Syntax:

```sql
COALESCE(expression1, expression2, ..., expressionN)
```

ตัวอย่างพื้นฐาน:

```sql
SELECT
    COALESCE(NULL, NULL, 'ค่าเริ่มต้น') AS result1,
    COALESCE(NULL, 42, 99)               AS result2,
    COALESCE('ค่าจริง', 'ค่าสำรอง')     AS result3,
    COALESCE(NULL, NULL)                  AS result4;
```

```
   result1    | result2 |  result3  | result4
---------------+---------+-----------+----------
 ค่าเริ่มต้น  |      42 | ค่าจริง  |
(1 row)
```

### ใช้แทนค่า NULL ในการแสดงผล

กรณีใช้งานที่พบบ่อยที่สุดคือการแสดงข้อความ default แทน `NULL` ตอนแสดงผลรายงาน:

```sql
SELECT
    full_name,
    COALESCE(email, '(ไม่มีอีเมล)') AS email_display,
    COALESCE(phone, '(ไม่มีเบอร์โทร)') AS phone_display
FROM customers
ORDER BY customer_id;
```

```
     full_name      |     email_display      |   phone_display
---------------------+--------------------------+---------------------
 สมชาย ใจดี         | somchai@example.com     | 0812345678
 สมหญิง รักเรียน    | somying@example.com     | (ไม่มีเบอร์โทร)
 วิชัย มั่งมี        | (ไม่มีอีเมล)           | 0898765432
 มานี ดีใจ          | manee@example.com       | 0855555555
 ประยุทธ ขยันทำงาน  | (ไม่มีอีเมล)           | (ไม่มีเบอร์โทร)
(5 rows)
```

### ใช้ในการคำนวณเพื่อป้องกัน NULL propagation

ปัญหาสำคัญของ `NULL` คือมัน "แพร่กระจาย" (propagate) ผ่านการคำนวณทางคณิตศาสตร์ — ถ้าค่าใดค่าหนึ่งใน expression เป็น `NULL` ผลลัพธ์ทั้ง expression จะเป็น `NULL` ไปด้วย:

```sql
-- ทดสอบ: ราคาสินค้า - ต้นทุน = กำไร แต่ถ้า cost เป็น NULL ผลลัพธ์จะเป็น NULL ทั้งหมด
SELECT
    product_name,
    price,
    cost,
    price - cost AS profit_wrong   -- จะเป็น NULL ถ้า cost เป็น NULL
FROM products
ORDER BY product_id;
```

```
   product_name    | price  |  cost | profit_wrong
--------------------+--------+-------+---------------
 เอสเพรสโซ่        |  45.00 | 12.00 |         33.00
 อเมริกาโน่ร้อน     |  55.00 | 15.00 |         40.00
 ลาเต้ร้อน          |  60.00 |        |
 อเมริกาโน่เย็น     |  60.00 | 15.00 |         45.00
 ลาเต้เย็น          |  65.00 | 18.00 |         47.00
 โมค่าเย็น          |  70.00 |        |
 ครัวซองต์          |  55.00 | 22.00 |         33.00
 บราวนี่            |  45.00 | 18.00 |         27.00
 ชาเขียว            |  50.00 | 10.00 |         40.00
 ชามะนาว            |  45.00 |        |
(10 rows)
```

สินค้า 3 ตัวที่ไม่มีข้อมูล `cost` ทำให้ `profit_wrong` กลายเป็น `NULL` ทั้งที่เราอยากได้อย่างน้อยว่า "กำไรขั้นต่ำ = ราคาขาย" (สมมติต้นทุนยังไม่ทราบให้ถือว่าเป็น 0 ชั่วคราว) แก้ไขด้วย `COALESCE`:

```sql
SELECT
    product_name,
    price,
    cost,
    price - COALESCE(cost, 0) AS profit_estimated
FROM products
ORDER BY product_id;
```

```
   product_name    | price  |  cost | profit_estimated
--------------------+--------+-------+--------------------
 เอสเพรสโซ่        |  45.00 | 12.00 |             33.00
 อเมริกาโน่ร้อน     |  55.00 | 15.00 |             40.00
 ลาเต้ร้อน          |  60.00 |        |             60.00
 อเมริกาโน่เย็น     |  60.00 | 15.00 |             45.00
 ลาเต้เย็น          |  65.00 | 18.00 |             47.00
 โมค่าเย็น          |  70.00 |        |             70.00
 ครัวซองต์          |  55.00 | 22.00 |             33.00
 บราวนี่            |  45.00 | 18.00 |             27.00
 ชาเขียว            |  50.00 | 10.00 |             40.00
 ชามะนาว            |  45.00 |        |             45.00
(10 rows)
```

### ใช้กับหลายคอลัมน์เพื่อหา "ค่าที่ดีที่สุดที่มี" (fallback chain)

`COALESCE` รับ argument ได้มากกว่า 2 ตัว ทำให้สร้าง fallback chain ได้ เช่น ต้องการช่องทางติดต่อลูกค้า โดยเรียงลำดับความสำคัญ email > phone > "ติดต่อไม่ได้":

```sql
SELECT
    full_name,
    COALESCE(email, phone, 'ติดต่อไม่ได้') AS contact_method
FROM customers
ORDER BY customer_id;
```

```
     full_name      |     contact_method
---------------------+--------------------------
 สมชาย ใจดี         | somchai@example.com
 สมหญิง รักเรียน    | somying@example.com
 วิชัย มั่งมี        | 0898765432
 มานี ดีใจ          | manee@example.com
 ประยุทธ ขยันทำงาน  | ติดต่อไม่ได้
(5 rows)
```

### ใช้แทนส่วนลดที่เป็น NULL ในการคำนวณยอดขาย

กลับไปดูตาราง `orders` ที่คอลัมน์ `discount_pct` เป็น `NULL` เมื่อไม่มีส่วนลด (ไม่ใช่ 0.00 — นี่เป็นการออกแบบที่ตั้งใจ เพื่อแยก "ไม่มีส่วนลด" ออกจาก "มีส่วนลดแต่เป็น 0%"):

```sql
SELECT
    order_id,
    discount_pct,
    COALESCE(discount_pct, 0) AS discount_pct_safe
FROM orders
ORDER BY order_id;
```

```
 order_id | discount_pct | discount_pct_safe
----------+---------------+---------------------
        1 |        10.00 |              10.00
        2 |               |               0.00
        3 |               |               0.00
        4 |         5.00 |               5.00
        5 |               |               0.00
(5 rows)
```

**สรุป Step 145:** `COALESCE` เป็นเครื่องมือที่ใช้บ่อยที่สุดในการจัดการ `NULL` — ใช้กำหนดค่า default ทั้งเพื่อการแสดงผลและการคำนวณ ป้องกันปัญหา `NULL` แพร่กระจายผ่าน expression

---

## Step 146: NULLIF — การแปลงค่าที่ไม่ต้องการให้เป็น NULL (ป้องกัน division by zero)

`NULLIF(value1, value2)` ทำงานตรงข้ามกับ `COALESCE` — มันเปรียบเทียบ `value1` กับ `value2` ถ้า**เท่ากัน**จะคืนค่า `NULL` แต่ถ้า**ไม่เท่ากัน**จะคืนค่า `value1` เดิม

Syntax:

```sql
NULLIF(value1, value2)
```

เทียบเท่ากับ:

```sql
CASE WHEN value1 = value2 THEN NULL ELSE value1 END
```

### กรณีใช้งานคลาสสิก: ป้องกัน division by zero

ปัญหาที่พบบ่อยมากในการคำนวณคือ division by zero ซึ่งใน PostgreSQL จะทำให้เกิด error ทันที:

```sql
-- ลองหา profit margin เมื่อ price อาจเป็น 0 (ในทางทฤษฎี)
SELECT 100 / 0;
```

```
ERROR:  division by zero
```

สมมติเราต้องการคำนวณ "cost ratio" (สัดส่วนต้นทุนต่อราคาขาย) แต่บางสินค้าหมดสต็อก (`stock_qty = 0`) และเราต้องการหาค่าเฉลี่ยกำไรต่อหน่วยสต็อก การหารด้วย `stock_qty` ที่อาจเป็น 0 จะทำให้เกิด error:

```sql
-- ชามะนาว มี stock_qty = 0 จะทำให้เกิด division by zero
SELECT
    product_name,
    price,
    stock_qty,
    price / stock_qty AS price_per_stock_unit
FROM products
WHERE product_name = 'ชามะนาว';
```

```
ERROR:  division by zero
```

แก้ไขด้วย `NULLIF` เพื่อแปลง `0` ให้เป็น `NULL` ก่อนหาร (การหารด้วย `NULL` จะได้ `NULL` ซึ่งไม่ error):

```sql
SELECT
    product_name,
    price,
    stock_qty,
    price / NULLIF(stock_qty, 0) AS price_per_stock_unit
FROM products
ORDER BY product_id;
```

```
   product_name    | price  | stock_qty | price_per_stock_unit
--------------------+--------+-----------+------------------------
 เอสเพรสโซ่        |  45.00 |       100 |             0.4500000000000000
 อเมริกาโน่ร้อน     |  55.00 |        80 |             0.6875000000000000
 ลาเต้ร้อน          |  60.00 |        60 |             1.0000000000000000
 อเมริกาโน่เย็น     |  60.00 |        90 |             0.6666666666666667
 ลาเต้เย็น          |  65.00 |        70 |             0.9285714285714286
 โมค่าเย็น          |  70.00 |        40 |             1.7500000000000000
 ครัวซองต์          |  55.00 |        20 |             2.7500000000000000
 บราวนี่            |  45.00 |        30 |             1.5000000000000000
 ชาเขียว            |  50.00 |        50 |             1.0000000000000000
 ชามะนาว            |  45.00 |         0 |
(10 rows)
```

แถวชามะนาว (`stock_qty = 0`) ได้ผลลัพธ์เป็น `NULL` แทนที่จะ error ทั้ง query — นี่คือรูปแบบการใช้งานที่พบบ่อยที่สุดของ `NULLIF`

### ใช้ NULLIF เพื่อกรอง "ค่า placeholder" ที่ไม่มีความหมาย

บางครั้งข้อมูลเก่าอาจใช้ค่าบางอย่าง (เช่น `''`, `'N/A'`, `-1`, `'UNKNOWN'`) แทน `NULL` เพราะข้อจำกัดของระบบเดิม เราสามารถใช้ `NULLIF` แปลงค่าเหล่านั้นให้กลายเป็น `NULL` จริง ๆ เพื่อให้ aggregate function และ `COALESCE` ทำงานถูกต้อง:

```sql
-- สมมติมี column notes ที่บางทีเก็บ '' แทนการไม่มีโน้ต
SELECT
    order_id,
    notes,
    NULLIF(notes, '') AS notes_normalized
FROM orders
ORDER BY order_id;
```

```
 order_id | notes                | notes_normalized
----------+------------------------+--------------------
        1 |                        |
        2 | รอลูกค้ามารับเอง      | รอลูกค้ามารับเอง
        3 |                        |
        4 | ลูกค้าประจำ            | ลูกค้าประจำ
        5 |                        |
(5 rows)
```

ในตัวอย่างนี้ `notes` ที่เป็น `NULL` อยู่แล้วก็ยังเป็น `NULL` ต่อไป (เพราะ `NULLIF(NULL, '')` จะให้ผลเป็น `NULL` เสมอ เนื่องจาก `NULL = ''` ให้ผล `UNKNOWN` ไม่ใช่ `TRUE` — ดังนั้นไม่เข้าเงื่อนไข "เท่ากัน" และ `NULLIF` จะคืนค่า `value1` เดิมคือ `NULL`)

### ผสม COALESCE และ NULLIF เข้าด้วยกัน

ตัวอย่างการคำนวณ discount amount โดยป้องกันทั้งกรณี `NULL` และ `0`:

```sql
SELECT
    order_id,
    discount_pct,
    -- ถ้า discount_pct เป็น NULL หรือ 0 ให้แสดงว่า "ไม่มีส่วนลด"
    COALESCE(NULLIF(discount_pct, 0)::TEXT, 'ไม่มีส่วนลด') AS discount_display
FROM orders
ORDER BY order_id;
```

```
 order_id | discount_pct |  discount_display
----------+---------------+---------------------
        1 |        10.00 | 10.00
        2 |               | ไม่มีส่วนลด
        3 |               | ไม่มีส่วนลด
        4 |         5.00 | 5.00
        5 |               | ไม่มีส่วนลด
(5 rows)
```

**สรุป Step 146:** `NULLIF` มีประโยชน์มากที่สุดในการป้องกัน division by zero และการ normalize ค่า placeholder ให้กลายเป็น `NULL` จริง ๆ ทำงานเป็นคู่ตรงข้ามกับ `COALESCE` ได้อย่างลงตัว

---

## Step 147: NULL ใน JOIN (เกริ่นสำหรับ Part 022 LEFT JOIN) และ NULL ใน ORDER BY (ทบทวนจาก Part 012)

### NULL ใน JOIN

เมื่อเราทำ `JOIN` ระหว่างสองตาราง เงื่อนไขการ join (มักเป็น `ON a.id = b.id`) ก็เป็นการเปรียบเทียบแบบเดียวกับที่เราเรียนใน Step 143 — ถ้าค่าใดค่าหนึ่งเป็น `NULL` ผลลัพธ์การเปรียบเทียบจะเป็น `UNKNOWN` และแถวนั้นจะ**ไม่ถูก match** ใน `INNER JOIN`

ลองดูตัวอย่าง: ลูกค้า `สมชาย ใจดี` (customer_id = 1) และ `มานี ดีใจ` (customer_id = 4) เป็นผู้แนะนำลูกค้าคนอื่น ๆ แต่ลูกค้าอีก 3 คนมี `referred_by IS NULL`

```sql
-- INNER JOIN ระหว่างลูกค้ากับ "ผู้แนะนำ" ของเขา
SELECT
    c.full_name AS customer_name,
    r.full_name AS referred_by_name
FROM customers c
INNER JOIN customers r ON c.referred_by = r.customer_id
ORDER BY c.customer_id;
```

```
   customer_name    | referred_by_name
---------------------+--------------------
 สมหญิง รักเรียน    | สมชาย ใจดี
 มานี ดีใจ          | สมชาย ใจดี
(2 rows)
```

สังเกตว่ามีเพียง 2 แถวในผลลัพธ์ — ลูกค้าที่ `referred_by IS NULL` (สมชาย, วิชัย, ประยุทธ) หายไปทั้งหมด เพราะ `c.referred_by = r.customer_id` เมื่อ `c.referred_by` เป็น `NULL` จะได้ `UNKNOWN` เสมอ ไม่ว่า `r.customer_id` จะเป็นค่าอะไรก็ตาม ทำให้ `INNER JOIN` ไม่ดึงแถวเหล่านั้นออกมา

นี่คือพฤติกรรมที่ถูกต้องตามหลัก 3-valued logic แต่ถ้าเราต้องการ "เก็บลูกค้าทุกคนไว้ แม้จะไม่มีผู้แนะนำ" เราต้องใช้ `LEFT JOIN` แทน (รายละเอียดเต็มจะอยู่ใน **Part 022**):

```sql
-- LEFT JOIN: เก็บลูกค้าทุกคน แม้ไม่มีผู้แนะนำ (แสดง NULL แทน)
SELECT
    c.full_name AS customer_name,
    r.full_name AS referred_by_name
FROM customers c
LEFT JOIN customers r ON c.referred_by = r.customer_id
ORDER BY c.customer_id;
```

```
   customer_name    | referred_by_name
---------------------+--------------------
 สมชาย ใจดี         |
 สมหญิง รักเรียน    | สมชาย ใจดี
 วิชัย มั่งมี        |
 มานี ดีใจ          | สมชาย ใจดี
 ประยุทธ ขยันทำงาน  |
(5 rows)
```

ด้วย `LEFT JOIN` ลูกค้าที่ไม่มีผู้แนะนำจะยังคงปรากฏในผลลัพธ์ โดยคอลัมน์ `referred_by_name` เป็น `NULL` เราจะเจาะลึกเรื่อง `LEFT JOIN`, `RIGHT JOIN`, `FULL JOIN` และการใช้ `NULL` เพื่อตรวจจับแถวที่ "ไม่ match" อย่างละเอียดใน **Part 022**

### NULL ใน ORDER BY

จาก Part 012 เราได้เรียนเรื่อง `ORDER BY` มาแล้ว ในที่นี้จะมาทบทวนพฤติกรรมเฉพาะของ `NULL`: ตามมาตรฐาน SQL, PostgreSQL จะถือว่า `NULL` เป็นค่าที่ **"มากที่สุด"** (largest) โดย default เมื่อเรียงจากน้อยไปมาก (`ASC`) `NULL` จะอยู่ท้ายสุด และเมื่อเรียงจากมากไปน้อย (`DESC`) `NULL` จะอยู่หัวสุด

```sql
-- ORDER BY discount_pct ASC (default) — NULL จะอยู่ท้ายสุด
SELECT order_id, discount_pct
FROM orders
ORDER BY discount_pct ASC;
```

```
 order_id | discount_pct
----------+---------------
        4 |         5.00
        1 |        10.00
        2 |
        3 |
        5 |
(5 rows)
```

```sql
-- ORDER BY discount_pct DESC — NULL จะอยู่หัวสุด
SELECT order_id, discount_pct
FROM orders
ORDER BY discount_pct DESC;
```

```
 order_id | discount_pct
----------+---------------
        2 |
        3 |
        5 |
        1 |        10.00
        4 |         5.00
(5 rows)
```

หากต้องการควบคุมตำแหน่งของ `NULL` เอง โดยไม่ผูกกับทิศทางการเรียง สามารถใช้ `NULLS FIRST` หรือ `NULLS LAST` ได้อย่างชัดเจน:

```sql
-- บังคับให้ NULL อยู่ท้ายสุดเสมอ ไม่ว่าจะ ASC หรือ DESC
SELECT order_id, discount_pct
FROM orders
ORDER BY discount_pct DESC NULLS LAST;
```

```
 order_id | discount_pct
----------+---------------
        1 |        10.00
        4 |         5.00
        2 |
        3 |
        5 |
(5 rows)
```

```sql
-- บังคับให้ NULL อยู่หัวสุดเสมอ แม้เรียง ASC
SELECT order_id, discount_pct
FROM orders
ORDER BY discount_pct ASC NULLS FIRST;
```

```
 order_id | discount_pct
----------+---------------
        2 |
        3 |
        5 |
        4 |         5.00
        1 |        10.00
(5 rows)
```

รูปแบบ `NULLS FIRST` / `NULLS LAST` มีประโยชน์มากในรายงานที่ต้องการ "เน้นแถวที่ขาดข้อมูล" ให้ผู้ใช้เห็นก่อน เช่น รายงานคำสั่งซื้อที่ยังไม่มีส่วนลดกำหนด ให้ทีมขายตรวจสอบก่อน

**สรุป Step 147:** `NULL` ในเงื่อนไข `JOIN` ทำให้แถวไม่ match ใน `INNER JOIN` (ต้องใช้ `LEFT JOIN` ถ้าต้องการเก็บแถวไว้ — รายละเอียดเต็มใน Part 022) ส่วนใน `ORDER BY` ค่า `NULL` จะถูกจัดให้อยู่ท้ายสุดเมื่อ `ASC` และหัวสุดเมื่อ `DESC` โดย default แต่สามารถควบคุมได้ด้วย `NULLS FIRST` / `NULLS LAST`

---

## Step 148: NULL ใน UNIQUE constraint — ทำไม PostgreSQL ยอมให้มีหลาย NULL ได้

จากที่เราสร้างตาราง `customers` ไว้ คอลัมน์ `email` มี constraint `UNIQUE` แต่เราสามารถใส่ลูกค้าที่ `email IS NULL` ได้ถึง 2 คน (วิชัย และ ประยุทธ) โดยไม่เกิด error ใด ๆ ทำไมถึงเป็นเช่นนั้น?

**เหตุผล:** ตามมาตรฐาน SQL, constraint `UNIQUE` จะตรวจสอบว่าไม่มีค่าที่ "เท่ากัน" ซ้ำกันสองแถว แต่เนื่องจาก `NULL = NULL` ให้ผลเป็น `UNKNOWN` (ไม่ใช่ `TRUE`) ตามที่เราเรียนใน Step 143 — PostgreSQL (และฐานข้อมูลส่วนใหญ่ตามมาตรฐาน SQL) จึงถือว่า `NULL` แต่ละตัว **ไม่เท่ากับ** `NULL` ตัวอื่นเสมอ ดังนั้นจึงไม่ถือว่าเป็นการซ้ำกัน (duplicate) และยอมให้มีได้หลายแถว

มาทดสอบให้เห็นจริง:

```sql
-- ลองเพิ่มลูกค้าอีกคนที่ email เป็น NULL (ควรจะสำเร็จ)
INSERT INTO customers (full_name, email, phone) VALUES ('ทดสอบ ระบบ', NULL, '0899999999');

SELECT customer_id, full_name, email FROM customers WHERE email IS NULL;
```

```
 customer_id |    full_name      | email
--------------+---------------------+-------
            3 | วิชัย มั่งมี        |
            5 | ประยุทธ ขยันทำงาน  |
            6 | ทดสอบ ระบบ         |
(3 rows)
```

INSERT สำเร็จโดยไม่มี error แม้ตอนนี้จะมี 3 แถวที่ `email IS NULL` — ลองเปรียบเทียบกับกรณีที่ใส่อีเมลซ้ำจริง ๆ:

```sql
-- ลองเพิ่มลูกค้าที่ email ซ้ำกับที่มีอยู่แล้ว (ควรจะล้มเหลว)
INSERT INTO customers (full_name, email, phone) VALUES ('ซ้ำ อีเมล', 'somchai@example.com', NULL);
```

```
ERROR:  duplicate key value violates unique constraint "customers_email_key"
DETAIL:  Key (email)=(somchai@example.com) already exists.
```

เห็นความแตกต่างชัดเจน: ค่าที่ไม่ใช่ `NULL` ซ้ำกันจะ error ทันที แต่ `NULL` ซ้ำกันได้หลายตัว

ล้างข้อมูลทดสอบออก:

```sql
DELETE FROM customers WHERE full_name = 'ทดสอบ ระบบ';
```

### ผลกระทบเชิงออกแบบ

พฤติกรรมนี้มีทั้งข้อดีและข้อควรระวัง:

**ข้อดี:** เหมาะกับกรณีที่คอลัมน์ควร unique **เฉพาะเมื่อมีค่า** เช่น `email` — ลูกค้าที่ให้อีเมลไว้ต้องไม่ซ้ำกับคนอื่น แต่ลูกค้าที่ยังไม่ได้ให้อีเมล (`NULL`) ก็ไม่ควรถูกบังคับให้ "unique กับความไม่มีอะไร"

**ข้อควรระวัง:** ถ้าธุรกิจต้องการอนุญาตให้มี `NULL` ได้แค่ **หนึ่งแถว** เท่านั้น (เช่น ต้องการมีแถว "default configuration" ได้แค่แถวเดียวที่ระบุ `NULL` ในบางคอลัมน์) `UNIQUE` constraint ธรรมดาจะไม่ช่วย ต้องใช้เทคนิคอื่น เช่น partial unique index:

```sql
-- ตัวอย่าง: อนุญาตให้มีสินค้าที่ discontinued_at IS NULL ได้ไม่จำกัด
-- แต่ถ้าต้องการจำกัดว่า category ใด category หนึ่งมี "สินค้าเด่น" (is_featured) ได้แค่ 1 ชิ้นที่ไม่ถูก discontinue
-- ใช้ partial unique index แทน UNIQUE ธรรมดา (ตัวอย่างเชิงแนวคิด จะเรียนละเอียดใน Part เรื่อง Indexing)
CREATE UNIQUE INDEX idx_one_active_promo_per_category
    ON products (category_id)
    WHERE discontinued_at IS NULL AND product_name = 'สินค้าโปรโมชั่น';
```

*(หมายเหตุ: ตัวอย่าง index ด้านบนเป็นเพียงการสาธิตแนวคิด partial unique index เชิงโครงสร้าง รายละเอียดการสร้าง index และ partial index อย่างเต็มรูปแบบจะอยู่ในบทที่ว่าด้วยการทำ Indexing โดยเฉพาะ)*

ทดสอบพฤติกรรม `NULL` กับ multi-column unique constraint เพิ่มเติม — ถ้ามี `UNIQUE(col_a, col_b)` และ `col_b` เป็น `NULL` ในหลายแถว แต่ `col_a` เหมือนกัน ก็ยังถือว่าไม่ซ้ำ เพราะแค่คอลัมน์เดียวที่เป็น `NULL` ก็ทำให้การเปรียบเทียบทั้งแถวเป็น "ไม่เท่ากัน" แล้ว:

```sql
CREATE TEMP TABLE test_multi_unique (
    a INTEGER,
    b INTEGER,
    UNIQUE (a, b)
);

INSERT INTO test_multi_unique VALUES (1, NULL);
INSERT INTO test_multi_unique VALUES (1, NULL);  -- สำเร็จ! แม้ a เหมือนกันและ b เป็น NULL ทั้งคู่
INSERT INTO test_multi_unique VALUES (1, 5);
INSERT INTO test_multi_unique VALUES (1, 5);      -- ล้มเหลว: (1, 5) ซ้ำจริง

SELECT * FROM test_multi_unique;
```

```
ERROR:  duplicate key value violates unique constraint "test_multi_unique_a_b_key"
DETAIL:  Key (a, b)=(1, 5) already exists.
```

```sql
-- หลัง INSERT 3 แถวแรกสำเร็จ (แถวที่ 4 ล้มเหลว) ผลลัพธ์คือ:
SELECT * FROM test_multi_unique;
```

```
 a | b
---+---
 1 |
 1 |
 1 | 5
(3 rows)
```

**สรุป Step 148:** PostgreSQL ยอมให้มีหลายแถวที่คอลัมน์ `UNIQUE` เป็น `NULL` ได้ เพราะ `NULL` ไม่ถือว่า "เท่ากับ" `NULL` อีกตัวตามหลัก 3-valued logic การออกแบบเช่นนี้เหมาะกับข้อมูลที่ "อาจไม่มีค่า" แต่ถ้ามีค่าต้องไม่ซ้ำ (เช่น อีเมล, เลขบัตรประชาชน) หากต้องการจำกัด `NULL` ให้มีได้แค่แถวเดียว ต้องใช้ partial unique index

---

## Step 149: IS DISTINCT FROM / IS NOT DISTINCT FROM — การเทียบค่าที่ปลอดภัยกับ NULL

เราได้เห็นปัญหาของ `=` และ `<>` กับ `NULL` มาหลายครั้งแล้วในบทนี้ — ผลลัพธ์เป็น `UNKNOWN` เสมอเมื่อมี `NULL` เข้ามาเกี่ยวข้อง PostgreSQL มี operator พิเศษที่ช่วยแก้ปัญหานี้โดยตรง คือ `IS DISTINCT FROM` และ `IS NOT DISTINCT FROM`

**หลักการ:** operator เหล่านี้เปรียบเทียบค่าสองค่าโดย **ถือว่า `NULL` เป็นค่าที่เปรียบเทียบได้จริง (ไม่ใช่ unknown)** — กล่าวคือ `NULL IS NOT DISTINCT FROM NULL` จะให้ผลเป็น `TRUE` (สองค่าเหมือนกัน เพราะทั้งคู่เป็น NULL) และผลลัพธ์จะเป็น boolean เสมอ ไม่มีทางเป็น `UNKNOWN`

เปรียบเทียบให้เห็นชัด:

```sql
SELECT
    NULL = NULL                    AS eq_operator,          -- NULL (unknown)
    NULL IS NOT DISTINCT FROM NULL AS is_not_distinct,       -- TRUE
    NULL IS DISTINCT FROM NULL     AS is_distinct,           -- FALSE
    5 = 5                            AS eq_5_5,
    5 IS NOT DISTINCT FROM 5        AS is_not_distinct_5_5,
    5 IS DISTINCT FROM NULL          AS is_distinct_5_null,   -- TRUE (5 กับ NULL ต่างกันจริง)
    5 = NULL                         AS eq_5_null;             -- NULL (unknown)
```

```
 eq_operator | is_not_distinct | is_distinct | eq_5_5 | is_not_distinct_5_5 | is_distinct_5_null | eq_5_null
--------------+-------------------+--------------+--------+------------------------+-----------------------+------------
              | t                 | f            | t      | t                      | t                     |
(1 row)
```

### กรณีใช้งาน: เปรียบเทียบค่าเก่ากับค่าใหม่ (เช่น audit / change detection)

การใช้งานที่พบบ่อยที่สุดของ `IS DISTINCT FROM` คือการตรวจจับว่าค่าคอลัมน์ "เปลี่ยนแปลงจริงหรือไม่" เมื่อ `UPDATE` ข้อมูล — โดยเฉพาะเมื่อคอลัมน์นั้นอาจเป็น `NULL` ได้ทั้งค่าเก่าและค่าใหม่

สมมติเราต้องการเขียน trigger หรือ query เพื่อหาลูกค้าที่ `phone` มีการเปลี่ยนแปลงจากค่าก่อนหน้า (ตัวอย่างเชิงแนวคิดโดยใช้ตารางเปรียบเทียบ):

```sql
CREATE TEMP TABLE customer_phone_history (
    customer_id INTEGER,
    old_phone   VARCHAR(20),
    new_phone   VARCHAR(20)
);

INSERT INTO customer_phone_history VALUES
    (1, '0812345678', '0812345678'),  -- ไม่เปลี่ยน
    (2, NULL,          '0899999999'),  -- เพิ่มเบอร์ใหม่ (เปลี่ยนจริง)
    (3, '0898765432', NULL),           -- ลบเบอร์ออก (เปลี่ยนจริง)
    (4, NULL,          NULL);          -- ยังไม่มีเบอร์เหมือนเดิม (ไม่เปลี่ยน)

-- ❌ ใช้ <> ธรรมดา: แถวที่มี NULL จะไม่ถูกตรวจจับว่า "เปลี่ยน" เพราะได้ UNKNOWN
SELECT customer_id, old_phone, new_phone
FROM customer_phone_history
WHERE old_phone <> new_phone;
```

```
 customer_id | old_phone | new_phone
--------------+-----------+------------
(0 rows)
```

ผลลัพธ์ว่างเปล่า ทั้งที่ลูกค้า customer_id 2 และ 3 มีการเปลี่ยนแปลงเบอร์โทรจริง ๆ! ปัญหาคือ `NULL <> '0899999999'` ให้ผล `UNKNOWN` ไม่ถูกเลือกใน `WHERE` แก้ไขด้วย `IS DISTINCT FROM`:

```sql
-- ✅ ใช้ IS DISTINCT FROM: ตรวจจับการเปลี่ยนแปลงได้ถูกต้องแม้มี NULL
SELECT customer_id, old_phone, new_phone
FROM customer_phone_history
WHERE old_phone IS DISTINCT FROM new_phone;
```

```
 customer_id | old_phone   | new_phone
--------------+--------------+------------
            2 |               | 0899999999
            3 | 0898765432  |
(2 rows)
```

ตอนนี้ได้ผลลัพธ์ถูกต้อง — เฉพาะแถวที่มีการเปลี่ยนแปลงค่าจริง (customer_id 2 และ 3) ถูกเลือก ส่วน customer_id 1 (ค่าเดิม) และ customer_id 4 (`NULL` เหมือนกันทั้งคู่) ถูกตัดออกอย่างถูกต้อง

### เปรียบเทียบกับ IS NOT DISTINCT FROM (หาแถวที่ "ไม่เปลี่ยน")

```sql
SELECT customer_id, old_phone, new_phone
FROM customer_phone_history
WHERE old_phone IS NOT DISTINCT FROM new_phone;
```

```
 customer_id | old_phone    | new_phone
--------------+---------------+------------
            1 | 0812345678  | 0812345678
            4 |               |
(2 rows)
```

`customer_id = 4` ที่ทั้งสองค่าเป็น `NULL` ถูกจัดว่า "ไม่เปลี่ยนแปลง" ได้อย่างถูกต้อง ซึ่งเป็นสิ่งที่ `old_phone = new_phone` ธรรมดาไม่สามารถทำได้ (จะได้ `UNKNOWN` เสมอเมื่อทั้งคู่เป็น `NULL`)

### ใช้ IS DISTINCT FROM ใน WHERE เพื่อหาแถวที่ "ไม่ตรงกับค่าที่กำหนด" รวมถึงกรณี NULL

กลับมาที่ตัวอย่างจาก Step 142 ที่เราพบปัญหา `NOT (referred_by = 1)` ให้ผล 0 แถว ทั้งที่ควรจะได้ลูกค้าที่ไม่ได้ถูกแนะนำโดย customer_id 1:

```sql
-- แก้ปัญหาด้วย IS DISTINCT FROM แทน NOT (... = ...)
SELECT full_name, referred_by
FROM customers
WHERE referred_by IS DISTINCT FROM 1
ORDER BY customer_id;
```

```
     full_name      | referred_by
---------------------+-------------
 สมชาย ใจดี         |
 วิชัย มั่งมี        |
 ประยุทธ ขยันทำงาน  |
(3 rows)
```

ผลลัพธ์นี้ถูกต้องตามที่ตั้งใจ — ได้ลูกค้าทั้งหมดที่ `referred_by` ไม่ใช่ `1` รวมถึงคนที่ `referred_by IS NULL` ด้วย

**ข้อควรระวังด้านประสิทธิภาพ:** `IS DISTINCT FROM` มักจะไม่สามารถใช้ index scan ได้อย่างมีประสิทธิภาพเท่ากับ `=` ธรรมดา เพราะ planner ไม่สามารถใช้ B-tree index range scan ได้ตรง ๆ ในหลายกรณี ดังนั้นควรใช้เมื่อจำเป็นจริง ๆ (เช่น การเปรียบเทียบที่ต้องการความถูกต้องเชิง logic กับ `NULL`) ไม่ใช่ใช้แทน `=` ทุกกรณีโดยไม่จำเป็น

**สรุป Step 149:** `IS DISTINCT FROM` / `IS NOT DISTINCT FROM` คือทางออกที่ปลอดภัยที่สุดเมื่อต้องการเปรียบเทียบค่าที่อาจเป็น `NULL` และต้องการผลลัพธ์เป็น `TRUE`/`FALSE` เสมอ ไม่ใช่ `UNKNOWN` — เหมาะมากสำหรับการตรวจจับการเปลี่ยนแปลงข้อมูล (change detection) และเงื่อนไขที่ต้องรวม `NULL` เข้าไปในตรรกะ "ไม่เท่ากับ"

---

## Step 150: Best practices — เมื่อไหร่ควรอนุญาต NULL, เมื่อไหร่ควรใช้ NOT NULL + DEFAULT แทน

หลังจากเรียนรู้พฤติกรรมทั้งหมดของ `NULL` มาแล้ว มาสรุปเป็นแนวทางปฏิบัติ (best practices) สำหรับการออกแบบ schema ว่าเมื่อไหร่ควรอนุญาตให้คอลัมน์เป็น `NULL` ได้ และเมื่อไหร่ควรบังคับ `NOT NULL` พร้อมกำหนด `DEFAULT`

### หลักการตัดสินใจ

**ควรอนุญาต NULL เมื่อ:**

1. **ค่านั้น "ยังไม่ทราบ" หรือ "ยังไม่เกิดขึ้น" ในเชิงธุรกิจจริง ๆ** เช่น `shipped_at` (วันที่จัดส่ง) — ถ้ายังไม่ได้จัดส่ง ก็ไม่มีวันที่ให้ใส่ การบังคับใส่ค่า default เช่น `'1970-01-01'` จะยิ่งทำให้สับสนกว่า
2. **ข้อมูลเป็น optional โดยธรรมชาติ** เช่น `phone`, `email` ของลูกค้าบางราย, `description` ของหมวดหมู่สินค้า
3. **ต้องการแยกความหมายระหว่าง "ไม่มีค่า" กับ "ค่าเริ่มต้น"** เช่น `discount_pct` — `NULL` หมายถึง "ไม่มีส่วนลดที่กำหนดไว้" ในขณะที่ `0` หมายถึง "มีส่วนลด 0%" (ซึ่งอาจมีความหมายทางบัญชีต่างกัน)
4. **Self-referencing foreign key** ที่ราก (root) ของโครงสร้างไม่มี parent เช่น `referred_by` (ลูกค้าที่ไม่มีใครแนะนำ) หรือ `manager_id` ในตารางพนักงาน (CEO ไม่มีหัวหน้า)

**ควรใช้ NOT NULL + DEFAULT เมื่อ:**

1. **ค่านั้นควรมีอยู่เสมอในเชิง business rule** เช่น `order_date` (วันที่สั่งซื้อ) — ทุกคำสั่งซื้อต้องมีวันที่ ใช้ `NOT NULL DEFAULT CURRENT_TIMESTAMP`
2. **ค่าที่มีความหมายเป็น "ปริมาณ" ที่ควรเริ่มต้นจากศูนย์** เช่น `stock_qty INTEGER NOT NULL DEFAULT 0` — สต็อกสินค้าใหม่เริ่มต้นที่ 0 ไม่ใช่ "ไม่ทราบ"
3. **คอลัมน์ที่เป็น flag หรือ status ที่ต้องมีค่าเสมอ** เช่น `is_active BOOLEAN NOT NULL DEFAULT TRUE`
4. **ต้องการป้องกันปัญหา 3-valued logic ในเงื่อนไขที่ query บ่อย** — ถ้าคอลัมน์ที่ query อยู่เป็นประจำ (เช่นใน `WHERE`, `JOIN`) เป็น `NULL` ได้ จะทำให้ query ซับซ้อนขึ้นเรื่อย ๆ (ต้องคอย `COALESCE` หรือใช้ `IS DISTINCT FROM` ทุกครั้ง) — ถ้าตัดปัญหานี้ได้ตั้งแต่การออกแบบ จะช่วยลด bug ในระยะยาว

### ตัวอย่างการปรับปรุง schema ตาม best practice

ทบทวนตาราง `products` ที่เราสร้างไว้ตอนต้นบท — คอลัมน์ `stock_qty` มี `DEFAULT 0` อยู่แล้วแต่ยังไม่ได้บังคับ `NOT NULL` ทำให้ยังสามารถใส่ `NULL` เข้าไปได้ (ซึ่งไม่สมเหตุสมผล เพราะสต็อกควรมีตัวเลขเสมอ):

```sql
-- ตรวจสอบว่าคอลัมน์ stock_qty อนุญาต NULL หรือไม่
SELECT column_name, is_nullable, column_default
FROM information_schema.columns
WHERE table_name = 'products' AND column_name = 'stock_qty';
```

```
 column_name | is_nullable | column_default
--------------+--------------+-----------------
 stock_qty    | YES          | 0
(1 row)
```

`is_nullable = YES` แปลว่าตอนนี้ `stock_qty` ยังสามารถเป็น `NULL` ได้ (แม้จะมี `DEFAULT 0`) — `DEFAULT` มีผลเฉพาะตอนที่ `INSERT` โดยไม่ระบุค่าเท่านั้น แต่ถ้า `INSERT` โดยระบุ `NULL` ตรง ๆ ก็ยังทำได้:

```sql
-- แม้มี DEFAULT 0 แต่ก็ยังใส่ NULL ตรง ๆ ได้ ถ้าไม่มี NOT NULL กำกับ
INSERT INTO products (category_id, product_name, price, stock_qty)
VALUES (1, 'ทดสอบสินค้า', 50.00, NULL);

SELECT product_name, stock_qty FROM products WHERE product_name = 'ทดสอบสินค้า';
```

```
   product_name    | stock_qty
--------------------+-----------
 ทดสอบสินค้า        |
(1 row)
```

เพื่อป้องกันปัญหานี้ ควรเพิ่ม `NOT NULL` เข้าไปด้วย ไม่ใช่พึ่งพา `DEFAULT` อย่างเดียว:

```sql
-- ล้างข้อมูลทดสอบก่อน
DELETE FROM products WHERE product_name = 'ทดสอบสินค้า';

-- ปรับปรุงคอลัมน์ stock_qty ให้บังคับ NOT NULL (ต้องแน่ใจว่าไม่มี NULL อยู่ก่อน)
ALTER TABLE products ALTER COLUMN stock_qty SET NOT NULL;

-- ตรวจสอบผลลัพธ์
SELECT column_name, is_nullable, column_default
FROM information_schema.columns
WHERE table_name = 'products' AND column_name = 'stock_qty';
```

```
 column_name | is_nullable | column_default
--------------+--------------+-----------------
 stock_qty    | NO           | 0
(1 row)
```

ตอนนี้ถ้าลองใส่ `NULL` เข้าไปตรง ๆ จะเกิด error ทันที:

```sql
INSERT INTO products (category_id, product_name, price, stock_qty)
VALUES (1, 'ทดสอบสินค้า2', 50.00, NULL);
```

```
ERROR:  null value in column "stock_qty" of relation "products" violates not-null constraint
DETAIL:  Failing row contains (11, 1, ทดสอบสินค้า2, 50.00, null, null, null).
```

นี่คือหลักการสำคัญ: **`DEFAULT` และ `NOT NULL` ควรใช้คู่กันเสมอ** เมื่อคอลัมน์นั้นควรมีค่าที่ชัดเจนตลอดเวลา `DEFAULT` เพียงอย่างเดียวช่วยแค่ตอนไม่ระบุค่าใน `INSERT` แต่ไม่ได้ป้องกันการใส่ `NULL` ตรง ๆ หรือการ `UPDATE` ให้เป็น `NULL` ในภายหลัง

### ตารางสรุปแนวทางตัดสินใจ

| สถานการณ์ | คำแนะนำ | ตัวอย่าง |
|---|---|---|
| ข้อมูลที่ต้องมีเสมอ ณ เวลาสร้างแถว | `NOT NULL` (ไม่ต้องมี default ถ้าผู้ใช้ต้องระบุเอง) | `product_name`, `price` |
| ข้อมูลที่มีค่าเริ่มต้นสมเหตุสมผล และควรมีค่าเสมอ | `NOT NULL DEFAULT ...` | `stock_qty INTEGER NOT NULL DEFAULT 0`, `member_since DATE NOT NULL DEFAULT CURRENT_DATE` |
| เหตุการณ์ที่ "ยังไม่เกิดขึ้น" ได้ | อนุญาต `NULL` | `shipped_at`, `discontinued_at` |
| ข้อมูล optional ตามธรรมชาติ | อนุญาต `NULL` | `phone`, `email`, `description` |
| Self-referencing FK ที่ราก (root) ไม่มี parent | อนุญาต `NULL` | `referred_by`, `manager_id` |
| ค่าที่ "ไม่มี" ต่างความหมายจาก "มีค่าเป็นศูนย์" | อนุญาต `NULL` แยกจาก `0` | `discount_pct`, `cost` (ยังไม่คำนวณ) |

### แนวทางเพิ่มเติมระดับ production

- ใช้ `CHECK` constraint ร่วมกับ `NULL` เมื่อ logic ซับซ้อนกว่าปกติ เช่น "ถ้า `discontinued_at IS NOT NULL` แล้ว `stock_qty` ต้องเป็น 0":

```sql
ALTER TABLE products
    ADD CONSTRAINT chk_discontinued_no_stock
    CHECK (discontinued_at IS NULL OR stock_qty = 0);
```

- ใน production ที่มีข้อมูลจำนวนมาก การเพิ่ม `NOT NULL` ทีหลังอาจต้อง scan ทั้งตาราง ควรวางแผน migration ให้ดี (ตรวจสอบข้อมูลเดิมก่อนว่าไม่มี `NULL` หลงเหลือ ด้วย `SELECT COUNT(*) FROM table WHERE col IS NULL` ก่อนรัน `ALTER TABLE ... SET NOT NULL`)
- เอกสารการออกแบบ (data dictionary) ควรระบุชัดเจนว่าคอลัมน์ไหนอนุญาต `NULL` ได้ และ **ความหมาย** ของ `NULL` ในคอลัมน์นั้นคืออะไร เพื่อไม่ให้ทีมพัฒนาตีความผิดในภายหลัง

**สรุป Step 150:** การตัดสินใจว่าคอลัมน์ควรอนุญาต `NULL` หรือไม่ ไม่ใช่แค่เรื่อง syntax แต่เป็นการตัดสินใจเชิงออกแบบที่ส่งผลต่อความถูกต้องของ query, ประสิทธิภาพ, และความเข้าใจของทีมในระยะยาว ควรอนุญาต `NULL` เฉพาะเมื่อมันสื่อความหมาย "ไม่ทราบ/ยังไม่มี" ได้จริง และใช้ `NOT NULL` + `DEFAULT` เมื่อคอลัมน์นั้นควรมีค่าที่ชัดเจนเสมอ

---

## สรุปท้ายบท

ในบทนี้เราได้เรียนรู้เรื่อง `NULL` อย่างละเอียด ซึ่งเป็นหนึ่งในแนวคิดที่สำคัญและมักถูกเข้าใจผิดที่สุดใน SQL:

1. **NULL คือ "unknown value"** ไม่ใช่ `0` หรือ `''` — เป็นสถานะที่บอกว่า "ไม่ทราบ" หรือ "ไม่มีอยู่" ไม่ใช่ค่าที่มีความหมายเชิงตัวเลขหรือ string
2. **Three-valued logic** — SQL มีสามสถานะ (`TRUE`/`FALSE`/`UNKNOWN`) ไม่ใช่สองสถานะแบบภาษาโปรแกรมทั่วไป และ `WHERE` จะเลือกเฉพาะแถวที่เป็น `TRUE` เท่านั้น
3. **`NULL = NULL` ให้ผลเป็น `NULL`** เสมอ ต้องใช้ `IS NULL` / `IS NOT NULL` ในการตรวจสอบ ไม่ใช่ `=` หรือ `<>`
4. **Aggregate functions ข้าม NULL โดยอัตโนมัติ** ยกเว้น `COUNT(*)` ที่นับทุกแถว
5. **`COALESCE`** ใช้กำหนดค่า default แทน `NULL`
6. **`NULLIF`** ใช้แปลงค่าที่ไม่ต้องการให้เป็น `NULL` เช่น ป้องกัน division by zero
7. **NULL ใน JOIN** ทำให้แถวไม่ match ใน `INNER JOIN` (ต้องใช้ `LEFT JOIN` — รายละเอียดเต็มใน Part 022) และ **NULL ใน ORDER BY** จะถูกจัดท้ายสุดเมื่อ `ASC` เว้นแต่กำหนด `NULLS FIRST`/`NULLS LAST`
8. **UNIQUE constraint ยอมให้มีหลาย NULL ได้** เพราะ `NULL` ไม่ถือว่าเท่ากับ `NULL` อีกตัว
9. **`IS DISTINCT FROM` / `IS NOT DISTINCT FROM`** คือการเปรียบเทียบที่ปลอดภัยกับ `NULL` ให้ผลเป็น boolean เสมอ
10. **Best practice**: อนุญาต `NULL` เมื่อมันสื่อความหมายจริง ๆ ใช้ `NOT NULL DEFAULT` เมื่อคอลัมน์ควรมีค่าเสมอ

ความเข้าใจเรื่อง `NULL` อย่างลึกซึ้งเป็นพื้นฐานสำคัญที่จะติดตัวเราไปตลอดหลักสูตร ไม่ว่าจะเป็นเรื่อง `JOIN` (Part 022), aggregate/`GROUP BY` ขั้นสูง, window functions, หรือแม้แต่การออกแบบ schema ระดับ production — ทุกเรื่องล้วนต้องคำนึงถึงพฤติกรรมของ `NULL` เสมอ

---

## แบบฝึกหัด

**ข้อ 1:** เขียน query เพื่อหาลูกค้าทั้งหมดที่ **ไม่มี** ทั้งอีเมลและเบอร์โทร (ทั้งสองคอลัมน์เป็น `NULL`)

<details>
<summary>เฉลยข้อ 1</summary>

```sql
SELECT full_name
FROM customers
WHERE email IS NULL AND phone IS NULL;
```

```
     full_name
---------------------
 ประยุทธ ขยันทำงาน
(1 row)
```

ต้องใช้ `IS NULL` ทั้งสองเงื่อนไข ห้ามใช้ `= NULL` เพราะจะได้ `UNKNOWN` เสมอ ไม่มีทางได้แถวใดเลย

</details>

---

**ข้อ 2:** จงอธิบายว่าทำไม query ต่อไปนี้จึงให้ผลลัพธ์ 0 แถว ทั้งที่ตาราง `orders` มีคำสั่งซื้อที่ `shipped_at` เป็น `NULL` อยู่:

```sql
SELECT order_id FROM orders WHERE shipped_at = NULL;
```

<details>
<summary>เฉลยข้อ 2</summary>

เพราะ operator `=` เมื่อเทียบกับ `NULL` จะให้ผลเป็น `UNKNOWN` เสมอ ไม่ว่าค่าฝั่งซ้ายจะเป็นอะไรก็ตาม (รวมถึงกรณีที่ฝั่งซ้ายเป็น `NULL` จริง ๆ ด้วย) และ `WHERE` clause จะเลือกเฉพาะแถวที่ผลลัพธ์เป็น `TRUE` เท่านั้น แถวที่ได้ `UNKNOWN` จะถูกตัดทิ้งเสมอ ทำให้ query นี้ได้ 0 แถวไม่ว่าข้อมูลในตารางจะเป็นอย่างไร วิธีที่ถูกต้องคือใช้ `WHERE shipped_at IS NULL`

</details>

---

**ข้อ 3:** เขียน query หาจำนวนคำสั่งซื้อทั้งหมด และจำนวนคำสั่งซื้อที่ "จัดส่งแล้ว" (มีค่า `shipped_at`) ในคำสั่งเดียว

<details>
<summary>เฉลยข้อ 3</summary>

```sql
SELECT
    COUNT(*)          AS total_orders,
    COUNT(shipped_at)  AS shipped_orders,
    COUNT(*) - COUNT(shipped_at) AS pending_orders
FROM orders;
```

```
 total_orders | shipped_orders | pending_orders
---------------+------------------+-----------------
             5 |                3 |               2
(1 row)
```

`COUNT(*)` นับทุกแถว ส่วน `COUNT(shipped_at)` นับเฉพาะแถวที่คอลัมน์นั้นไม่ใช่ `NULL` ผลต่างของสองค่าคือจำนวนคำสั่งซื้อที่ยังไม่จัดส่ง

</details>

---

**ข้อ 4:** ใช้ `COALESCE` เขียน query แสดงรายชื่อสินค้าพร้อม `cost` โดยถ้า `cost` เป็น `NULL` ให้แสดงคำว่า `'ยังไม่ระบุต้นทุน'` แทน (ผลลัพธ์คอลัมน์นี้ควรเป็น TEXT)

<details>
<summary>เฉลยข้อ 4</summary>

```sql
SELECT
    product_name,
    COALESCE(cost::TEXT, 'ยังไม่ระบุต้นทุน') AS cost_display
FROM products
ORDER BY product_id;
```

```
   product_name    |   cost_display
--------------------+---------------------
 เอสเพรสโซ่        | 12.00
 อเมริกาโน่ร้อน     | 15.00
 ลาเต้ร้อน          | ยังไม่ระบุต้นทุน
 อเมริกาโน่เย็น     | 15.00
 ลาเต้เย็น          | 18.00
 โมค่าเย็น          | ยังไม่ระบุต้นทุน
 ครัวซองต์          | 22.00
 บราวนี่            | 18.00
 ชาเขียว            | 10.00
 ชามะนาว            | ยังไม่ระบุต้นทุน
(10 rows)
```

ต้อง cast `cost` เป็น `::TEXT` ก่อน เพราะ `COALESCE` ต้องการให้ทุก argument เป็น data type ที่เข้ากันได้ (`NUMERIC` กับ `TEXT` ตรง ๆ ไม่เข้ากัน)

</details>

---

**ข้อ 5:** เขียน query คำนวณ `price / stock_qty` สำหรับสินค้าทุกตัว โดยป้องกัน division by zero ด้วย `NULLIF` (ถ้า `stock_qty = 0` ให้ผลลัพธ์เป็น `NULL` แทนที่จะ error)

<details>
<summary>เฉลยข้อ 5</summary>

```sql
SELECT
    product_name,
    stock_qty,
    price / NULLIF(stock_qty, 0) AS price_per_unit_stock
FROM products
ORDER BY product_id;
```

สินค้าที่ `stock_qty = 0` (ชามะนาว) จะได้ผลลัพธ์เป็น `NULL` ในคอลัมน์ `price_per_unit_stock` แทนที่จะทำให้ query ทั้งหมด error ด้วย `division by zero`

</details>

---

**ข้อ 6:** จากตาราง `orders` เขียน query จัดกลุ่มคำสั่งซื้อตาม `discount_pct` แล้วนับจำนวนคำสั่งซื้อในแต่ละกลุ่ม พร้อมอธิบายว่า `NULL` ถูกจัดการอย่างไรใน `GROUP BY`

<details>
<summary>เฉลยข้อ 6</summary>

```sql
SELECT discount_pct, COUNT(*) AS order_count
FROM orders
GROUP BY discount_pct
ORDER BY discount_pct NULLS FIRST;
```

```
 discount_pct | order_count
--------------+-------------
              |           3
        5.00 |           1
       10.00 |           1
(3 rows)
```

แม้ `NULL = NULL` จะให้ผล `UNKNOWN` ตามหลัก 3-valued logic แต่ `GROUP BY` มีกฎพิเศษที่ถือว่าแถวทุกแถวที่มี `NULL` ในคอลัมน์ที่ใช้ group อยู่ใน "กลุ่มเดียวกัน" เสมอ (ซึ่งต่างจากพฤติกรรมของ `WHERE ... = NULL`) ทำให้คำสั่งซื้อ 3 รายการที่ไม่มีส่วนลดถูกนับรวมเป็นกลุ่มเดียว

</details>

---

**ข้อ 7:** อธิบายว่าทำไมคอลัมน์ `email` ในตาราง `customers` ที่มี constraint `UNIQUE` จึงยอมให้มีลูกค้าหลายคนที่ `email IS NULL` ได้ พร้อมยกตัวอย่างการทดสอบด้วย SQL

<details>
<summary>เฉลยข้อ 7</summary>

เพราะ `UNIQUE` constraint ตรวจสอบว่าไม่มีค่าที่ "เท่ากัน" ซ้ำกัน แต่ตามหลัก 3-valued logic `NULL = NULL` ให้ผลเป็น `UNKNOWN` ไม่ใช่ `TRUE` PostgreSQL (ตามมาตรฐาน SQL) จึงถือว่า `NULL` แต่ละตัวไม่เท่ากับ `NULL` ตัวอื่น ทำให้ไม่ถือว่าเป็นการซ้ำกัน

```sql
INSERT INTO customers (full_name, email) VALUES ('ทดสอบ A', NULL);
INSERT INTO customers (full_name, email) VALUES ('ทดสอบ B', NULL);
-- ทั้งสองคำสั่งสำเร็จ แม้ email เป็น NULL เหมือนกันทั้งคู่

SELECT full_name FROM customers WHERE email IS NULL;
```

หากต้องการจำกัดให้มีได้แค่แถวเดียวที่ `email IS NULL` ต้องใช้ partial unique index แทน `UNIQUE` ธรรมดา

</details>

---

**ข้อ 8:** เขียน query โดยใช้ `IS DISTINCT FROM` เพื่อหาลูกค้าที่ `referred_by` **ไม่ใช่** `customer_id = 1` (รวมถึงลูกค้าที่ `referred_by IS NULL` ด้วย) แล้วเปรียบเทียบกับการเขียนด้วย `NOT (referred_by = 1)` ว่าให้ผลต่างกันอย่างไร

<details>
<summary>เฉลยข้อ 8</summary>

```sql
-- ใช้ IS DISTINCT FROM (ถูกต้อง)
SELECT full_name, referred_by
FROM customers
WHERE referred_by IS DISTINCT FROM 1
ORDER BY customer_id;
```

```
     full_name      | referred_by
---------------------+-------------
 สมชาย ใจดี         |
 วิชัย มั่งมี        |
 ประยุทธ ขยันทำงาน  |
(3 rows)
```

```sql
-- ใช้ NOT (referred_by = 1) (ผิด — จะพลาดแถวที่ referred_by IS NULL)
SELECT full_name, referred_by
FROM customers
WHERE NOT (referred_by = 1)
ORDER BY customer_id;
```

```
 full_name | referred_by
-----------+-------------
(0 rows)
```

`NOT (referred_by = 1)` ให้ผล 0 แถว เพราะแถวที่ `referred_by = 1` ตรงจริงจะถูก `NOT` เป็น `FALSE` (ถูกตัดออก) ส่วนแถวที่ `referred_by IS NULL` จะได้ `referred_by = 1` เป็น `UNKNOWN` และ `NOT UNKNOWN` ยังเป็น `UNKNOWN` อยู่ (ก็ถูกตัดออกเช่นกัน) ในขณะที่ `IS DISTINCT FROM` ให้ผลเป็น boolean เสมอ จึงจับแถวที่ `referred_by IS NULL` ได้ถูกต้องด้วย

</details>

---

**ข้อ 9:** จงยกตัวอย่างคอลัมน์ 2 คอลัมน์ในตาราง `orders` หรือ `products` ที่ **ควรอนุญาต NULL** พร้อมเหตุผล และคอลัมน์ 2 คอลัมน์ที่ **ควรบังคับ NOT NULL** (พร้อมหรือไม่พร้อม DEFAULT) พร้อมเหตุผล

<details>
<summary>เฉลยข้อ 9</summary>

**ควรอนุญาต NULL:**

- `shipped_at` (orders) — เพราะคำสั่งซื้อที่ยังไม่จัดส่งไม่มีวันที่ให้ใส่จริง ๆ การบังคับใส่ค่า default ปลอม ๆ จะทำให้ตีความข้อมูลผิด
- `discontinued_at` (products) — สินค้าที่ยังขายอยู่ไม่มีวันที่เลิกขาย `NULL` สื่อความหมาย "ยังไม่เลิกขาย" ได้ชัดเจนกว่าการใส่ค่าอื่นใด

**ควรบังคับ NOT NULL:**

- `order_date` (orders) พร้อม `DEFAULT CURRENT_TIMESTAMP` — ทุกคำสั่งซื้อต้องมีวันที่สั่งซื้อเสมอ ไม่มีคำสั่งซื้อใดที่ "ไม่ทราบวันที่สั่ง"
- `product_name` (products) โดยไม่ต้องมี default — สินค้าทุกตัวต้องมีชื่อ ไม่สมเหตุสมผลที่จะมีสินค้าที่ "ไม่ทราบชื่อ" และไม่มีค่า default ที่เหมาะสมสำหรับชื่อสินค้า (ต้องให้ผู้ใช้ระบุเองเสมอ)

</details>

---

**ข้อ 10:** เขียน query แสดงยอดขายรวมของแต่ละคำสั่งซื้อ (จาก `order_items`) พร้อมคำนวณส่วนลดที่ต้องหักออก โดยใช้ `COALESCE` เพื่อจัดการกับ `discount_pct` ที่อาจเป็น `NULL` (สูตร: `ยอดขายหลังหักส่วนลด = ยอดขายรวม * (1 - discount_pct/100)`)

<details>
<summary>เฉลยข้อ 10</summary>

```sql
SELECT
    o.order_id,
    SUM(oi.quantity * oi.unit_price) AS subtotal,
    o.discount_pct,
    ROUND(
        SUM(oi.quantity * oi.unit_price) * (1 - COALESCE(o.discount_pct, 0) / 100),
        2
    ) AS total_after_discount
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
GROUP BY o.order_id, o.discount_pct
ORDER BY o.order_id;
```

```
 order_id | subtotal | discount_pct | total_after_discount
----------+-----------+---------------+------------------------
        1 |    135.00 |        10.00 |                121.50
        2 |     65.00 |               |                 65.00
        3 |    180.00 |               |                180.00
        4 |    155.00 |         5.00 |                147.25
        5 |    120.00 |               |                120.00
(5 rows)
```

`COALESCE(o.discount_pct, 0)` ทำให้คำสั่งซื้อที่ไม่มีส่วนลด (`NULL`) ถูกคำนวณเสมือนส่วนลด 0% แทนที่จะทำให้ผลคูณทั้งหมดกลายเป็น `NULL` จาก NULL propagation

</details>

---

บทถัดไปเราจะเรียนรู้เรื่อง Primary Key และ Unique Constraint อย่างละเอียด ซึ่งต่อยอดจากความเข้าใจเรื่อง `NULL` ในบทนี้โดยตรง โดยเฉพาะพฤติกรรมของ `UNIQUE` ที่เราเพิ่งเรียนใน Step 148

**บทถัดไป:** [Part 016: Primary Key และ Unique Constraint](./part-016-primary-key-unique.md)
