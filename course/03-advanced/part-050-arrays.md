# Part 050: Arrays — การสร้าง จัดเก็บ และ Query

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับสูง | Part 050

---

## เป้าหมายการเรียนรู้

หลังจากเรียนจบบทนี้ ผู้เรียนจะสามารถ:

- เข้าใจว่า Array type ใน PostgreSQL คืออะไร และทำไม PostgreSQL ถึงรองรับ array เป็น native type ในขณะที่ RDBMS อื่นส่วนใหญ่ไม่รองรับ (หรือรองรับแบบจำกัด)
- ประกาศคอลัมน์แบบ array และ insert ข้อมูลด้วยได้ทั้ง `ARRAY[...]` syntax และ `'{...}'` literal syntax
- เข้าถึง element ของ array ด้วย index แบบ 1-based (ข้อควรระวังสำคัญที่ต่างจากภาษาโปรแกรมทั่วไปที่ใช้ 0-based) และทำ array slicing
- ใช้ array operators `@>`, `<@`, `&&`, `=` เพื่อเปรียบเทียบและค้นหาข้อมูลใน array
- ใช้ `ANY` และ `ALL` เป็นทางเลือกแทน `IN` เมื่อต้องเทียบกับค่าหลายค่า หรือค่าที่เก็บอยู่ใน array
- ใช้ array functions ที่สำคัญ เช่น `array_length`, `array_append`, `array_remove`, `array_position`, `cardinality`
- ใช้ `UNNEST` แปลง array ให้กลายเป็นแถว (rows) เพื่อทำ query หรือ JOIN กับข้อมูล array ได้อย่างมีประสิทธิภาพ
- สร้างและใช้ GIN Index บนคอลัมน์ array เพื่อเร่งความเร็วการค้นหา
- ตัดสินใจได้อย่างมีเหตุผลว่าเมื่อไหร่ควรใช้ array และเมื่อไหร่ควรแยกเป็นตาราง (normalization) โดยพิจารณาจาก trade-off จริง ไม่ใช่แค่ความสะดวก
- ประยุกต์ใช้ array ในระบบจริง เช่น ระบบ tag สินค้า และ wishlist ของลูกค้า พร้อม query ค้นหาที่มีประสิทธิภาพ

---

## เตรียมข้อมูล

บทนี้ยังคงใช้ฐานข้อมูล e-commerce เดิมที่เราใช้ตลอดทั้งหลักสูตร แต่จะ**เพิ่มคอลัมน์ array สองคอลัมน์**เข้าไปเพื่อสาธิตการทำงานกับ array โดยเฉพาะ:

- `products.tags TEXT[]` — เก็บ tag ของสินค้า เช่น `'bestseller'`, `'new-arrival'`, `'eco-friendly'`
- `customers.favorite_product_ids INTEGER[]` — เก็บรายการสินค้าที่ลูกค้าถูกใจ/บันทึกไว้ (wishlist)

ถ้าผู้เรียนสร้างฐานข้อมูลใหม่ ให้รันสคริปต์เต็มด้านล่างนี้ทั้งหมด (ถ้ามีตารางจาก Part ก่อนหน้าอยู่แล้ว ให้ `DROP TABLE` ก่อน หรือสร้างในฐานข้อมูลใหม่)

### 1. สร้างตาราง

```sql
DROP TABLE IF EXISTS order_items CASCADE;
DROP TABLE IF EXISTS orders CASCADE;
DROP TABLE IF EXISTS products CASCADE;
DROP TABLE IF EXISTS customers CASCADE;
DROP TABLE IF EXISTS suppliers CASCADE;
DROP TABLE IF EXISTS categories CASCADE;

CREATE TABLE categories (
    category_id         SERIAL PRIMARY KEY,
    category_name        VARCHAR(100) NOT NULL,
    parent_category_id   INTEGER REFERENCES categories(category_id)
);

CREATE TABLE suppliers (
    supplier_id   SERIAL PRIMARY KEY,
    supplier_name VARCHAR(150) NOT NULL,
    country       VARCHAR(60)
);

CREATE TABLE products (
    product_id     SERIAL PRIMARY KEY,
    product_name   VARCHAR(150) NOT NULL,
    category_id    INTEGER REFERENCES categories(category_id),
    supplier_id    INTEGER REFERENCES suppliers(supplier_id),
    unit_price     NUMERIC(10,2) NOT NULL,
    stock_quantity INTEGER NOT NULL DEFAULT 0,
    is_active      BOOLEAN NOT NULL DEFAULT true,
    tags           TEXT[] DEFAULT '{}'
);

CREATE TABLE customers (
    customer_id           SERIAL PRIMARY KEY,
    first_name            VARCHAR(60) NOT NULL,
    last_name             VARCHAR(60) NOT NULL,
    email                 VARCHAR(150) UNIQUE,
    country               VARCHAR(60),
    signup_date           DATE NOT NULL DEFAULT CURRENT_DATE,
    favorite_product_ids  INTEGER[] DEFAULT '{}'
);

CREATE TABLE orders (
    order_id     SERIAL PRIMARY KEY,
    customer_id  INTEGER REFERENCES customers(customer_id),
    order_date   TIMESTAMPTZ NOT NULL DEFAULT now(),
    status       VARCHAR(20) NOT NULL DEFAULT 'pending',
    ship_country VARCHAR(60)
);

CREATE TABLE order_items (
    order_item_id SERIAL PRIMARY KEY,
    order_id      INTEGER REFERENCES orders(order_id),
    product_id    INTEGER REFERENCES products(product_id),
    quantity      INTEGER NOT NULL CHECK (quantity > 0),
    unit_price    NUMERIC(10,2) NOT NULL
);
```

### 2. เติมข้อมูลตัวอย่าง

```sql
-- categories
INSERT INTO categories (category_name, parent_category_id) VALUES
    ('Electronics', NULL),          -- 1
    ('Computers', 1),               -- 2
    ('Mobile Phones', 1),           -- 3
    ('Home & Living', NULL),        -- 4
    ('Kitchenware', 4),             -- 5
    ('Fashion', NULL),              -- 6
    ('Men Clothing', 6),            -- 7
    ('Women Clothing', 6),          -- 8
    ('Sports & Outdoor', NULL),     -- 9
    ('Beauty & Health', NULL);      -- 10

-- suppliers
INSERT INTO suppliers (supplier_name, country) VALUES
    ('Thai Tech Supply',        'Thailand'),   -- 1
    ('Global Gadget Co.',       'China'),      -- 2
    ('Nordic Home Living',      'Sweden'),     -- 3
    ('Kitchen Master',          'Germany'),    -- 4
    ('Bangkok Fashion House',   'Thailand'),   -- 5
    ('Green Earth Textiles',    'Vietnam'),    -- 6
    ('Active Gear Co.',         'USA'),        -- 7
    ('Pure Beauty Labs',        'South Korea'),-- 8
    ('Smart Living Imports',    'Japan'),      -- 9
    ('Everyday Essentials Ltd', 'Thailand');   -- 10

-- products (พร้อม tags)
INSERT INTO products (product_name, category_id, supplier_id, unit_price, stock_quantity, is_active, tags) VALUES
    ('Wireless Mouse M1',            2, 2,   350.00, 120, true,  ARRAY['bestseller','wireless']),
    ('Mechanical Keyboard K80',      2, 1,  2190.00,  45, true,  ARRAY['bestseller','gaming','new-arrival']),
    ('27-inch 4K Monitor',           2, 2,  9990.00,  20, true,  ARRAY['premium']),
    ('Smartphone X12 128GB',         3, 2, 12900.00,  60, true,  ARRAY['bestseller','new-arrival']),
    ('Smartphone X12 Case',          3, 2,   199.00, 200, true,  ARRAY['accessory','eco-friendly']),
    ('Bluetooth Earbuds Pro',        1, 9,  1590.00,  80, true,  ARRAY['bestseller','wireless','new-arrival']),
    ('Stainless Steel Water Bottle', 5, 4,   290.00, 150, true,  ARRAY['eco-friendly','bestseller']),
    ('Non-stick Frying Pan 28cm',    5, 4,   690.00,  70, true,  ARRAY['kitchen']),
    ('Bamboo Cutting Board',         5, 3,   250.00,  90, true,  ARRAY['eco-friendly']),
    ('Ceramic Coffee Mug Set',       5, 4,   450.00,  60, true,  ARRAY['new-arrival']),
    ('Men Slim Fit Shirt',           7, 5,   590.00, 100, true,  ARRAY['new-arrival']),
    ('Women Summer Dress',           8, 5,  890.00,   65, true,  ARRAY['new-arrival','bestseller']),
    ('Organic Cotton T-Shirt',       7, 6,   350.00, 140, true,  ARRAY['eco-friendly','bestseller']),
    ('Yoga Mat Premium',             9, 7,   790.00,  55, true,  ARRAY['eco-friendly','bestseller']),
    ('Running Shoes AirFlex',        9, 7,  2490.00,  40, true,  ARRAY['bestseller','sports']),
    ('Camping Tent 4-Person',        9, 7,  4990.00,  15, true,  ARRAY['new-arrival','outdoor']),
    ('Vitamin C Serum 30ml',        10, 8,   690.00,  75, true,  ARRAY['bestseller','skincare']),
    ('Herbal Shampoo Bar',          10, 8,   190.00, 130, true,  ARRAY['eco-friendly','new-arrival']),
    ('Smart LED Desk Lamp',          4, 9,   990.00,  50, true,  ARRAY['new-arrival','smart-home']),
    ('Discontinued USB Hub',         2, 1,   150.00,   0, false, ARRAY['clearance']);

-- customers (พร้อม favorite_product_ids)
INSERT INTO customers (first_name, last_name, email, country, signup_date, favorite_product_ids) VALUES
    ('Somchai',  'Jaidee',     'somchai.j@example.com',  'Thailand',  '2023-01-15', ARRAY[1,4,6]),
    ('Nalinee',  'Suksawat',   'nalinee.s@example.com',  'Thailand',  '2023-02-20', ARRAY[12,13,14]),
    ('John',     'Smith',      'john.smith@example.com', 'USA',       '2023-03-05', ARRAY[3,4]),
    ('Aiko',     'Tanaka',     'aiko.t@example.com',     'Japan',     '2023-03-18', ARRAY[19,10]),
    ('Wei',      'Zhang',      'wei.zhang@example.com',  'China',     '2023-04-02', ARRAY[]::INTEGER[]),
    ('Emma',     'Johnson',    'emma.j@example.com',     'USA',       '2023-04-25', ARRAY[15,16,17]),
    ('Kittipong','Meesuk',     'kittipong.m@example.com','Thailand',  '2023-05-10', ARRAY[7,9,18]),
    ('Sarah',    'Williams',   'sarah.w@example.com',    'UK',        '2023-05-30', ARRAY[2,4,6]),
    ('Minh',     'Nguyen',     'minh.n@example.com',     'Vietnam',   '2023-06-14', ARRAY[13,9]),
    ('Yuki',     'Sato',       'yuki.sato@example.com',  'Japan',     '2023-07-01', ARRAY[19,20]),
    ('Pornthip', 'Rattana',    'pornthip.r@example.com', 'Thailand',  '2023-07-22', ARRAY[1,2,3,4,5]),
    ('David',    'Brown',      'david.b@example.com',    'USA',       '2023-08-09', ARRAY[15]),
    ('Lin',      'Huang',      'lin.huang@example.com',  'China',     '2023-08-28', ARRAY[]::INTEGER[]),
    ('Anong',    'Srisuk',     'anong.s@example.com',    'Thailand',  '2023-09-11', ARRAY[17,18]),
    ('Michael',  'Davis',      'michael.d@example.com',  'USA',       '2023-09-30', ARRAY[2,16]);

-- orders
INSERT INTO orders (customer_id, order_date, status, ship_country) VALUES
    (1,  '2023-10-01 10:15:00+07', 'completed', 'Thailand'),
    (2,  '2023-10-02 14:22:00+07', 'completed', 'Thailand'),
    (3,  '2023-10-03 09:05:00-05', 'completed', 'USA'),
    (4,  '2023-10-04 18:40:00+09', 'completed', 'Japan'),
    (1,  '2023-10-06 11:00:00+07', 'completed', 'Thailand'),
    (6,  '2023-10-07 08:30:00-05', 'shipped',   'USA'),
    (7,  '2023-10-08 16:12:00+07', 'completed', 'Thailand'),
    (8,  '2023-10-09 12:50:00+00', 'cancelled', 'UK'),
    (9,  '2023-10-10 13:25:00+07', 'completed', 'Vietnam'),
    (10, '2023-10-11 19:00:00+09', 'shipped',   'Japan'),
    (11, '2023-10-12 10:10:00+07', 'completed', 'Thailand'),
    (12, '2023-10-13 15:35:00-05', 'pending',   'USA'),
    (2,  '2023-10-15 09:45:00+07', 'completed', 'Thailand'),
    (14, '2023-10-16 17:20:00+07', 'completed', 'Thailand'),
    (15, '2023-10-18 11:55:00-05', 'shipped',   'USA');

-- order_items
INSERT INTO order_items (order_id, product_id, quantity, unit_price) VALUES
    (1, 1, 1,   350.00),
    (1, 4, 1, 12900.00),
    (2, 12, 2,   890.00),
    (2, 13, 1,   350.00),
    (3, 3, 1,  9990.00),
    (4, 19, 1,   990.00),
    (5, 6, 2,  1590.00),
    (6, 15, 1,  2490.00),
    (6, 16, 1,  4990.00),
    (7, 7, 3,   290.00),
    (7, 9, 1,   250.00),
    (8, 2, 1,  2190.00),
    (9, 13, 2,   350.00),
    (9, 9, 1,   250.00),
    (10, 19, 1,   990.00),
    (10, 20, 1,   150.00),
    (11, 1, 2,   350.00),
    (11, 2, 1,  2190.00),
    (11, 4, 1, 12900.00),
    (12, 15, 1,  2490.00),
    (13, 2, 1,  2190.00),
    (13, 6, 1,  1590.00),
    (14, 17, 2,   690.00),
    (14, 18, 1,   190.00),
    (15, 15, 1,  2490.00),
    (15, 16, 1,  4990.00);
```

ตรวจสอบจำนวนแถวอย่างรวดเร็ว:

```sql
SELECT 'categories' AS tbl, count(*) FROM categories
UNION ALL SELECT 'suppliers', count(*) FROM suppliers
UNION ALL SELECT 'products', count(*) FROM products
UNION ALL SELECT 'customers', count(*) FROM customers
UNION ALL SELECT 'orders', count(*) FROM orders
UNION ALL SELECT 'order_items', count(*) FROM order_items;
```

```
    tbl      | count
-------------+-------
 categories  |    10
 suppliers   |    10
 products    |    20
 customers   |    15
 orders      |    15
 order_items |    26
(6 rows)
```

พร้อมแล้ว มาเริ่มทำความเข้าใจ Array ใน PostgreSQL กันทีละ Step

---

## Step 491: Array type ใน PostgreSQL คืออะไร

PostgreSQL เป็นหนึ่งในไม่กี่ RDBMS ที่รองรับ **array เป็น native data type** ของคอลัมน์ได้โดยตรง หมายความว่าเราสามารถเก็บ "รายการของค่า" หลายค่าไว้ใน**ช่องเดียว**ของแถวเดียวได้ โดยไม่ต้องแตกเป็นตารางลูก (child table) เสมอไป

ตัวอย่างที่เห็นได้ชัดคือคอลัมน์ `tags` ในตาราง `products` ของเรา — สินค้าหนึ่งชิ้นอาจมีหลาย tag พร้อมกัน เช่น `{bestseller, wireless, new-arrival}` เราสามารถเก็บ tag ทั้งหมดไว้ในคอลัมน์เดียวได้เลย

### ทำไม PostgreSQL ถึงมี Array type?

RDBMS ส่วนใหญ่ (เช่น MySQL แบบดั้งเดิม, SQL Server) ยึดหลัก **First Normal Form (1NF)** อย่างเคร่งครัด ซึ่งกำหนดว่าแต่ละช่องของตารางต้องเก็บ "atomic value" (ค่าเดียวที่แบ่งย่อยไม่ได้) เท่านั้น ถ้าต้องการเก็บหลายค่า จะต้องแยกเป็นตารางใหม่และ JOIN กัน

แต่ PostgreSQL เลือกที่จะ**ยืดหยุ่นกว่านั้น** เพราะในทางปฏิบัติ มีหลายกรณีที่การเก็บค่าแบบ array ในคอลัมน์เดียวนั้นเหมาะสมกว่าการสร้างตารางแยก เช่น:

1. **ข้อมูลที่เป็น "attribute ของ entity เดียว" อย่างชัดเจน** และไม่ต้องการ query ข้ามไปมาซับซ้อน เช่น tag ของบทความ, สีที่มีของสินค้า, หมายเลขโทรศัพท์สำรอง
2. **ข้อมูลที่มีลำดับ (ordered)** และต้องการรักษาลำดับนั้นไว้ เช่น ขั้นตอนของ workflow
3. **ลด JOIN ที่ไม่จำเป็น** เมื่อข้อมูลนั้นแทบไม่เคย query แยกจาก entity หลักเลย
4. รองรับการทำงานร่วมกับ ภาษาโปรแกรมสมัยใหม่ที่มี array/list type อยู่แล้ว (Python list, JS array) — การ map ข้อมูลจึงเป็นธรรมชาติ

Array ใน PostgreSQL สามารถเป็นได้แทบทุก data type: `INTEGER[]`, `TEXT[]`, `NUMERIC[]`, `BOOLEAN[]`, แม้แต่ array ของ array (multi-dimensional array) ก็ทำได้

```sql
-- ตัวอย่างชนิดข้อมูล array ต่าง ๆ ที่ประกาศได้
CREATE TABLE demo_array_types (
    int_array      INTEGER[],
    text_array     TEXT[],
    numeric_array  NUMERIC(10,2)[],
    bool_array     BOOLEAN[],
    -- เขียนแบบ ANSI SQL ก็ได้ ผลลัพธ์เหมือนกันทุกประการ
    int_array_ansi INTEGER ARRAY
);
```

> **ข้อควรระวัง:** แม้ PostgreSQL อนุญาตให้ระบุขนาดของ array ได้ เช่น `INTEGER[3]` แต่จริง ๆ แล้ว PostgreSQL **ไม่ได้บังคับขนาดนั้นจริง** — `INTEGER[3]` ยังยอมรับ array ที่มีสมาชิกมากกว่าหรือน้อยกว่า 3 ตัวได้ตามปกติ ตัวเลขในวงเล็บเป็นเพียง "คำอธิบายประกอบ" (documentation) เท่านั้น ไม่ใช่ constraint จริง ดังนั้นการเขียน `TEXT[]` เฉย ๆ (ไม่ระบุขนาด) จึงเป็นแนวปฏิบัติที่แนะนำ เพราะสื่อความหมายตรงกับพฤติกรรมจริง

### เมื่อไหร่ที่เราใช้ array ในสคีมาของเรา

ในสคีมา e-commerce ของเรา เราเลือกใช้ array สองจุด:

- `products.tags TEXT[]` — สินค้าหนึ่งตัวมีได้หลาย tag, tag ไม่มีข้อมูลอื่นประกอบ (ไม่มี "วันที่ติด tag" หรือ "ใครเป็นคนติด tag"), และมักถูก query ร่วมกับสินค้าเสมอ (ไม่แยก query tag เดี่ยว ๆ บ่อยนัก)
- `customers.favorite_product_ids INTEGER[]` — wishlist ของลูกค้า เป็นรายการ id สินค้าที่เรียงลำดับตามที่ลูกค้าเพิ่มเข้าไป

ทั้งสองกรณีนี้เป็นตัวอย่างคลาสสิกของการใช้ array อย่างเหมาะสม ซึ่งเราจะพิจารณา trade-off อย่างละเอียดใน Step 499

---

## Step 492: การประกาศ Array Column และการ INSERT ค่า Array

### การประกาศคอลัมน์

การประกาศคอลัมน์ array ทำได้ง่ายมาก เพียงเติม `[]` ต่อท้าย data type:

```sql
ALTER TABLE products ADD COLUMN sizes_available TEXT[];
-- ตรวจสอบ
\d products
```

(คอลัมน์นี้เป็นตัวอย่างเสริมเท่านั้น ในสคีมาหลักของเราไม่ได้ใช้ ให้ข้ามหรือลบทิ้งได้ด้วย `ALTER TABLE products DROP COLUMN sizes_available;`)

คอลัมน์ `tags` และ `favorite_product_ids` ที่เราสร้างไว้ตั้งแต่ต้นบทคือตัวอย่างจริงที่จะใช้ตลอดทั้งบทนี้

### การ INSERT ค่า Array: สอง syntax

PostgreSQL รองรับการเขียนค่า array สองแบบหลัก ๆ

**1) `ARRAY[...]` constructor syntax** — อ่านง่าย เหมือน array literal ในภาษาโปรแกรมทั่วไป แนะนำให้ใช้แบบนี้เป็นหลักเมื่อเขียน SQL ด้วยมือ

```sql
INSERT INTO products (product_name, category_id, supplier_id, unit_price, stock_quantity, tags)
VALUES ('Portable Power Bank 20000mAh', 1, 2, 890.00, 80,
        ARRAY['bestseller', 'new-arrival', 'travel']);
```

**2) `'{...}'` literal syntax (curly-brace string)** — เป็นรูปแบบที่ PostgreSQL ใช้แสดงผล array ออกมาเวลา `SELECT` และเป็นรูปแบบที่ `pg_dump` ใช้ในการ backup/restore ข้อมูล ต้องใส่ quote ครอบและ cast type ให้ถูกต้องเมื่อจำเป็น

```sql
INSERT INTO products (product_name, category_id, supplier_id, unit_price, stock_quantity, tags)
VALUES ('Foldable Umbrella', 4, 9, 199.00, 150, '{eco-friendly,new-arrival}');
```

ทั้งสองแบบให้ผลลัพธ์เหมือนกันทุกประการ เมื่อ query กลับมาดู:

```sql
SELECT product_name, tags
FROM products
WHERE product_name IN ('Portable Power Bank 20000mAh', 'Foldable Umbrella');
```

```
          product_name          |                tags
----------------------------------+-------------------------------------
 Portable Power Bank 20000mAh    | {bestseller,new-arrival,travel}
 Foldable Umbrella               | {eco-friendly,new-arrival}
(2 rows)
```

### ข้อควรระวังของ `'{...}'` literal

ถ้า element มีเครื่องหมายพิเศษ เช่น comma, curly brace, หรือช่องว่างนำหน้า/ตามหลัง ต้องครอบด้วย double quote ภายใน string:

```sql
-- ผิด: comma ใน element จะถูกตีความว่าเป็นตัวแบ่ง element
-- INSERT ... VALUES ('{New York, USA}');   -- จะกลายเป็น 2 element คือ "New York" กับ "USA"!

-- ถูกต้อง: ใช้ double quote ครอบ element ที่มี comma
INSERT INTO customers (first_name, last_name, email, country, favorite_product_ids)
VALUES ('Test', 'User', 'test.array@example.com', 'Thailand', '{1,2,3}');

SELECT favorite_product_ids FROM customers WHERE email = 'test.array@example.com';
```

```
 favorite_product_ids
-----------------------
 {1,2,3}
(1 row)
```

ด้วยเหตุนี้ เมื่อเขียนโค้ดด้วยมือหรือสร้างค่า array แบบ dynamic จาก application จึงแนะนำให้ใช้ `ARRAY[...]` มากกว่า เพราะปลอดภัยกว่าในการจัดการ string escaping

### array ว่าง และ NULL

array ที่ไม่มีสมาชิกเลย (`{}`) ไม่เหมือนกับ `NULL`:

```sql
SELECT
    ARRAY[]::INTEGER[]        AS empty_array,
    NULL::INTEGER[]           AS null_array,
    ARRAY[]::INTEGER[] IS NULL   AS empty_is_null,
    cardinality(ARRAY[]::INTEGER[]) AS empty_cardinality;
```

```
 empty_array | null_array | empty_is_null | empty_cardinality
-------------+------------+---------------+-------------------
 {}          |            | f             |                 0
(1 row)
```

นี่คือเหตุผลที่เราตั้ง `DEFAULT '{}'` ไว้กับทั้ง `tags` และ `favorite_product_ids` ตั้งแต่สร้างตาราง — เพื่อให้แถวใหม่เริ่มต้นด้วย array ว่างแทนที่จะเป็น `NULL` ซึ่งช่วยลดความยุ่งยากในการเขียนเงื่อนไขตรวจสอบภายหลัง (ไม่ต้องกังวลเรื่อง `NULL` propagation ในหลาย ๆ ฟังก์ชัน)

### UPDATE คอลัมน์ array ทั้งก้อน

```sql
UPDATE products
SET tags = ARRAY['bestseller', 'wireless', 'limited-edition']
WHERE product_name = 'Wireless Mouse M1';

SELECT product_name, tags FROM products WHERE product_name = 'Wireless Mouse M1';
```

```
    product_name    |               tags
---------------------+-----------------------------------
 Wireless Mouse M1   | {bestseller,wireless,limited-edition}
(1 row)
```

ลบข้อมูลทดสอบที่เพิ่มเข้ามาในหัวข้อนี้ก่อนไปต่อ (เพื่อให้ผลลัพธ์ใน step ถัด ๆ ไปตรงกับตัวอย่างในบท):

```sql
DELETE FROM products WHERE product_name IN ('Portable Power Bank 20000mAh', 'Foldable Umbrella');
DELETE FROM customers WHERE email = 'test.array@example.com';
UPDATE products SET tags = ARRAY['bestseller','wireless'] WHERE product_name = 'Wireless Mouse M1';
```

---

## Step 493: การเข้าถึง Element ด้วย Index และ Array Slicing

### Index เริ่มที่ 1 ไม่ใช่ 0 — ข้อควรระวังที่สำคัญที่สุด

นี่คือจุดที่โปรแกรมเมอร์แทบทุกคนพลาดเมื่อเริ่มใช้ array ใน PostgreSQL เป็นครั้งแรก เพราะภาษาโปรแกรมเกือบทั้งหมด (Python, JavaScript, Java, C, Go) ใช้ index แบบ **0-based** (element แรกอยู่ที่ index 0) แต่ **PostgreSQL array ใช้ index แบบ 1-based โดย default** (element แรกอยู่ที่ index 1)

```sql
SELECT
    tags,
    tags[1] AS first_tag,   -- element แรก
    tags[2] AS second_tag   -- element ที่สอง
FROM products
WHERE product_name = 'Mechanical Keyboard K80';
```

```
                    tags                     | first_tag | second_tag
----------------------------------------------+-----------+------------
 {bestseller,gaming,new-arrival}              | bestseller| gaming
(1 row)
```

ถ้าเผลอเขียน `tags[0]` จะไม่เกิด error แต่จะได้ผลลัพธ์เป็น `NULL` เสมอ เพราะ array ตาม default เริ่มที่ index 1 การเข้าถึง index 0 จึงเท่ากับ "อยู่นอกขอบเขตด้านล่าง":

```sql
SELECT tags[0] AS wrong_index, tags[1] AS correct_index
FROM products
WHERE product_name = 'Mechanical Keyboard K80';
```

```
 wrong_index | correct_index
-------------+---------------
             | bestseller
(1 row)
```

> **กับดักที่พบบ่อย:** เพราะ `tags[0]` คืนค่า `NULL` แทนที่จะ error โปรแกรมเมอร์ที่ใช้ index ผิดอาจไม่รู้ตัวว่าโค้ดของตัวเองพัง จนกว่าจะสังเกตว่าผลลัพธ์ผิดปกติ ควรทดสอบให้แน่ใจเสมอเมื่อทำงานกับ index ของ array ใน PostgreSQL

### การเข้าถึง element สุดท้าย

PostgreSQL ไม่มี syntax แบบ `array[-1]` เหมือนบางภาษา แต่ใช้ฟังก์ชัน `array_length` ร่วมกับ index ได้:

```sql
SELECT
    tags,
    tags[array_length(tags, 1)] AS last_tag
FROM products
WHERE product_name = 'Mechanical Keyboard K80';
```

```
                 tags               | last_tag
--------------------------------------+-------------
 {bestseller,gaming,new-arrival}     | new-arrival
(1 row)
```

### Array Slicing

ใช้ syntax `array[start:end]` เพื่อดึงช่วงของ element ออกมา (inclusive ทั้งสองด้าน):

```sql
SELECT
    tags,
    tags[1:2]  AS first_two_tags,
    tags[2:]   AS from_second_onward,
    tags[:2]   AS up_to_second
FROM products
WHERE product_name = 'Mechanical Keyboard K80';
```

```
                 tags               | first_two_tags     | from_second_onward | up_to_second
--------------------------------------+--------------------+--------------------+------------------
 {bestseller,gaming,new-arrival}     | {bestseller,gaming}| {gaming,new-arrival}| {bestseller,gaming}
(1 row)
```

ถ้า slicing เกินขอบเขตของ array จะไม่เกิด error แต่ PostgreSQL จะ clamp ให้อยู่ในขอบเขตที่มีจริงโดยอัตโนมัติ:

```sql
SELECT tags[1:100] AS slice_beyond_bound
FROM products
WHERE product_name = 'Mechanical Keyboard K80';
```

```
       slice_beyond_bound
----------------------------------
 {bestseller,gaming,new-arrival}
(1 row)
```

### เข้าถึง element ของ integer array (favorite_product_ids)

```sql
SELECT
    first_name,
    favorite_product_ids,
    favorite_product_ids[1] AS first_favorite
FROM customers
WHERE first_name = 'Pornthip';
```

```
 first_name | favorite_product_ids | first_favorite
-------------+-----------------------+-----------------
 Pornthip    | {1,2,3,4,5}           |               1
(1 row)
```

และเราสามารถ JOIN element นั้นกลับไปหาชื่อสินค้าได้ทันที:

```sql
SELECT c.first_name, p.product_name AS first_favorite_product
FROM customers c
JOIN products p ON p.product_id = c.favorite_product_ids[1]
WHERE c.first_name = 'Pornthip';
```

```
 first_name | first_favorite_product
-------------+-------------------------
 Pornthip    | Wireless Mouse M1
(1 row)
```

---

## Step 494: Array Operators — `@>`, `<@`, `&&`, `=`

PostgreSQL มี operator เฉพาะสำหรับเปรียบเทียบ array สองก้อนกัน ซึ่งเป็นหัวใจสำคัญของการ query ข้อมูล array อย่างมีประสิทธิภาพ (และเป็น operator ที่ GIN index รองรับ — ดู Step 498)

| Operator | ความหมาย | ตัวอย่าง |
|---|---|---|
| `@>` | contains — array ด้านซ้ายมี**ครบทุกสมาชิก**ของด้านขวาหรือไม่ | `tags @> ARRAY['bestseller']` |
| `<@` | contained by — array ด้านซ้ายเป็น**สับเซตของ**ด้านขวาหรือไม่ | `ARRAY['bestseller'] <@ tags` |
| `&&` | overlap — มีสมาชิก**ร่วมกันอย่างน้อยหนึ่งตัว**หรือไม่ | `tags && ARRAY['bestseller','new-arrival']` |
| `=` | เท่ากันทุกประการ (ลำดับและจำนวนต้องตรงกันหมด) | `tags = ARRAY['bestseller','wireless']` |

### `@>` — Contains: หาสินค้าที่มี tag ที่กำหนดครบทุกตัว

```sql
-- หาสินค้าที่มีทั้ง tag 'bestseller' และ 'new-arrival' (ต้องมีครบทั้งสองอย่าง)
SELECT product_name, tags
FROM products
WHERE tags @> ARRAY['bestseller', 'new-arrival'];
```

```
      product_name       |                    tags
---------------------------+---------------------------------------------
 Mechanical Keyboard K80  | {bestseller,gaming,new-arrival}
 Smartphone X12 128GB     | {bestseller,new-arrival}
 Bluetooth Earbuds Pro    | {bestseller,wireless,new-arrival}
 Women Summer Dress       | {new-arrival,bestseller}
(4 rows)
```

สังเกตว่า `@>` ไม่สนใจลำดับของ element — `{new-arrival,bestseller}` ก็ถือว่า contains `ARRAY['bestseller','new-arrival']` เช่นกัน

### `<@` — Contained By: ตรงข้ามกับ `@>`

```sql
-- หาสินค้าที่ tags ทั้งหมดของมันอยู่ภายในเซตนี้เท่านั้น (ไม่มี tag อื่นนอกเหนือจากนี้)
SELECT product_name, tags
FROM products
WHERE tags <@ ARRAY['bestseller', 'wireless', 'gaming', 'new-arrival'];
```

```
      product_name       |               tags
---------------------------+-----------------------------------
 Wireless Mouse M1        | {bestseller,wireless}
 Mechanical Keyboard K80  | {bestseller,gaming,new-arrival}
 Bluetooth Earbuds Pro    | {bestseller,wireless,new-arrival}
(3 rows)
```

Smartphone X12 128GB ไม่ปรากฏในผลลัพธ์ แม้จะมี `bestseller` และ `new-arrival` เพราะ tags ของมันเป็นสับเซตที่ครบตามที่กำหนดเช่นกัน แต่ในความเป็นจริงมันก็ปรากฏ — ให้สังเกตว่าเงื่อนไข `<@` ตรวจสอบว่า**ทุก**สมาชิกของ `tags` ต้องอยู่ในเซตด้านขวา ซึ่ง `{bestseller,new-arrival}` ก็เป็นสับเซตของเซตด้านขวาเช่นกัน (ผลลัพธ์จริงจะมี 4 แถวรวม Smartphone X12 128GB ด้วย)

### `&&` — Overlap: มี tag ใดตรงกันบ้าง (เหมาะกับ "หรือ" logic)

```sql
-- หาสินค้าที่มี tag อย่างน้อยหนึ่งใน 'eco-friendly' หรือ 'sports'
SELECT product_name, tags
FROM products
WHERE tags && ARRAY['eco-friendly', 'sports'];
```

```
         product_name           |                tags
------------------------------------+---------------------------------
 Smartphone X12 Case               | {accessory,eco-friendly}
 Stainless Steel Water Bottle      | {eco-friendly,bestseller}
 Bamboo Cutting Board              | {eco-friendly}
 Organic Cotton T-Shirt            | {eco-friendly,bestseller}
 Yoga Mat Premium                  | {eco-friendly,bestseller}
 Running Shoes AirFlex             | {bestseller,sports}
 Herbal Shampoo Bar                | {eco-friendly,new-arrival}
(7 rows)
```

`&&` คือ operator ที่ตอบโจทย์ "หาแถวที่ tag ตรงกับอย่างน้อยหนึ่งใน list ที่ผู้ใช้เลือก" ซึ่งพบบ่อยมากในหน้า filter สินค้าของเว็บ e-commerce จริง

### `=` — เท่ากันทุกประการ

```sql
SELECT product_name, tags
FROM products
WHERE tags = ARRAY['eco-friendly', 'bestseller'];
```

```
      product_name        |          tags
----------------------------+--------------------------
 Stainless Steel Water Bottle | {eco-friendly,bestseller}
 Organic Cotton T-Shirt       | {eco-friendly,bestseller}
 Yoga Mat Premium             | {eco-friendly,bestseller}
(3 rows)
```

`=` เข้มงวดมาก ต้องมีจำนวนสมาชิกเท่ากัน**และ**เรียงลำดับเดียวกันทุกประการ ถ้าสลับลำดับเป็น `ARRAY['bestseller','eco-friendly']` ผลลัพธ์จะไม่ตรงกับแถวเหล่านี้เลย จึงควรใช้ `=` เฉพาะกรณีที่มั่นใจเรื่องลำดับจริง ๆ (ในทางปฏิบัติ `@>` ร่วมกับ `<@` ทั้งสองทาง มักเป็นทางเลือกที่ปลอดภัยกว่าเมื่อต้องการเทียบว่า "เซตเดียวกัน" โดยไม่สนใจลำดับ)

### สรุปการเลือกใช้ operator

```sql
-- "เซตเดียวกัน" โดยไม่สนใจลำดับ = ใช้ @> ร่วมกับ <@ ทั้งสองทาง
SELECT product_name
FROM products
WHERE tags @> ARRAY['bestseller','eco-friendly']
  AND tags <@ ARRAY['bestseller','eco-friendly'];
```

```
      product_name
---------------------------
 Stainless Steel Water Bottle
 Organic Cotton T-Shirt
 Yoga Mat Premium
(3 rows)
```

---

## Step 495: ANY และ ALL กับ Array — ทางเลือกแทน IN

### `= ANY(array)` เทียบเท่ากับ `IN`

`ANY` เมื่อใช้กับ array จะเทียบค่าฝั่งซ้ายกับ**สมาชิกแต่ละตัว**ของ array และคืนค่า true ถ้าตรงกับตัวใดตัวหนึ่ง — พฤติกรรมเหมือน `IN (...)` ทุกประการ แต่ใช้ array แทน list ค่าคงที่

```sql
-- เทียบเท่ากับ WHERE category_id IN (2, 3, 9)
SELECT product_name, category_id
FROM products
WHERE category_id = ANY(ARRAY[2, 3, 9]);
```

```
       product_name        | category_id
------------------------------+-------------
 Wireless Mouse M1            |           2
 Mechanical Keyboard K80      |           2
 27-inch 4K Monitor           |           2
 Smartphone X12 128GB         |           3
 Smartphone X12 Case          |           3
 Yoga Mat Premium             |           9
 Running Shoes AirFlex        |           9
 Camping Tent 4-Person        |           9
 Discontinued USB Hub         |           2
(9 rows)
```

จุดที่ `ANY` มีประโยชน์มากกว่า `IN` คือเมื่อ**ค่ามาจากตัวแปรหรือคอลัมน์ array อยู่แล้ว** เช่น การเช็คว่า product_id นี้อยู่ใน wishlist ของลูกค้าหรือไม่ — ใช้ `ANY` กับคอลัมน์ array ได้โดยตรงโดยไม่ต้องแปลงรูปแบบ:

```sql
-- เช็คว่า product_id = 4 อยู่ใน favorite_product_ids ของลูกค้าคนไหนบ้าง
SELECT first_name, favorite_product_ids
FROM customers
WHERE 4 = ANY(favorite_product_ids);
```

```
 first_name | favorite_product_ids
-------------+-----------------------
 Somchai     | {1,4,6}
 John        | {3,4}
 Sarah       | {2,4,6}
 Pornthip    | {1,2,3,4,5}
(4 rows)
```

> เทคนิคนี้เทียบเท่ากับการใช้ `@>` แบบ scalar เดี่ยว (`favorite_product_ids @> ARRAY[4]`) แต่ `= ANY(...)` มักอ่านง่ายกว่าเวลาต้องการเช็คแค่ค่าเดียว ในขณะที่ `@>` เหมาะกับการเช็คหลายค่าพร้อมกัน

### `ALL` — ต้องตรงกับ**ทุกตัว**ในเงื่อนไข

`ALL` มักใช้ร่วมกับ operator เปรียบเทียบเชิงตัวเลข มากกว่าการเช็คความเท่ากันแบบ `ANY`

```sql
-- หาสินค้าที่ราคาสูงกว่าสินค้า "ทุกตัว" ในหมวด Kitchenware (category_id = 5)
SELECT product_name, unit_price, category_id
FROM products
WHERE unit_price > ALL(
    SELECT unit_price FROM products WHERE category_id = 5
)
ORDER BY unit_price;
```

```
       product_name       | unit_price | category_id
-----------------------------+------------+-------------
 Men Slim Fit Shirt          |     590.00 |           7
 Bluetooth Earbuds Pro       |    1590.00 |           1
 Women Summer Dress          |     890.00 |           8
 Mechanical Keyboard K80     |    2190.00 |           2
 Running Shoes AirFlex       |    2490.00 |           9
 Smart LED Desk Lamp         |     990.00 |           4
 Camping Tent 4-Person       |    4990.00 |           9
 27-inch 4K Monitor          |    9990.00 |           2
 Smartphone X12 128GB        |   12900.00 |           3
(9 rows)
```

`unit_price > ALL(...)` เทียบเท่ากับ `unit_price > MAX(...)` ทางตรรกะ (แต่สามารถใช้ operator อื่นได้ เช่น `<`, `>=`, `<>`) และมีประโยชน์เมื่อเขียนเป็น subquery ธรรมดาโดยไม่ต้องพึ่ง aggregate function โดยตรง

### เปรียบเทียบ `IN` กับ `= ANY(ARRAY[...])`

ทั้งสองแบบทำงานเหมือนกันทุกประการในแง่ผลลัพธ์และ query planner จะ optimize ให้เหมือนกัน แต่ `ANY` มีข้อได้เปรียบเมื่อ:

- ค่าถูกส่งมาเป็น array parameter จาก application (เช่น driver บางตัวส่ง array ได้ตรง ๆ โดยไม่ต้อง build string `IN (1,2,3,...)`) — ปลอดภัยกว่าเรื่อง SQL injection ด้วย
- ต้องการเทียบกับคอลัมน์ array ที่มีอยู่แล้วในตาราง (ทำแบบนี้กับ `IN` ไม่ได้โดยตรง)

```sql
-- ตัวอย่าง: ส่ง array parameter จาก application (แสดงด้วย literal แทน placeholder)
SELECT product_name
FROM products
WHERE product_id = ANY(ARRAY[1,4,6,15]);
```

```
      product_name
---------------------------
 Wireless Mouse M1
 Smartphone X12 128GB
 Bluetooth Earbuds Pro
 Running Shoes AirFlex
(4 rows)
```

---

## Step 496: Array Functions ที่สำคัญ

PostgreSQL มีฟังก์ชันมาตรฐานสำหรับจัดการ array มากมาย ต่อไปนี้คือฟังก์ชันที่ใช้บ่อยที่สุด

### `array_length(array, dimension)` — นับจำนวนสมาชิก

```sql
SELECT product_name, tags, array_length(tags, 1) AS tag_count
FROM products
ORDER BY tag_count DESC NULLS LAST
LIMIT 5;
```

```
        product_name         |                    tags                      | tag_count
--------------------------------+------------------------------------------------+-----------
 Mechanical Keyboard K80        | {bestseller,gaming,new-arrival}                |         3
 Bluetooth Earbuds Pro          | {bestseller,wireless,new-arrival}              |         3
 Smartphone X12 128GB           | {bestseller,new-arrival}                       |         2
 Women Summer Dress             | {new-arrival,bestseller}                       |         2
 Smartphone X12 Case            | {accessory,eco-friendly}                       |         2
(5 rows)
```

พารามิเตอร์ที่สองของ `array_length` คือ dimension (มิติของ array) — สำหรับ array แบบมิติเดียว (ซึ่งเป็นกรณีส่วนใหญ่ที่ใช้งาน) ให้ใส่ `1` เสมอ

> **ข้อควรระวัง:** `array_length(ARRAY[]::INTEGER[], 1)` คืนค่า `NULL` ไม่ใช่ `0` เพราะ array ว่างไม่มี "มิติที่ 1" ในทางเทคนิค ถ้าต้องการนับสมาชิกของ array ที่อาจว่างเปล่าอย่างปลอดภัย ควรใช้ `cardinality()` แทน (ดูด้านล่าง)

### `cardinality(array)` — นับจำนวนสมาชิกแบบปลอดภัยกว่า

```sql
SELECT
    array_length(ARRAY[]::INTEGER[], 1) AS length_of_empty,
    cardinality(ARRAY[]::INTEGER[])      AS cardinality_of_empty,
    cardinality(NULL::INTEGER[])         AS cardinality_of_null;
```

```
 length_of_empty | cardinality_of_empty | cardinality_of_null
------------------+-----------------------+----------------------
                  |                     0 |
(1 row)
```

สังเกตว่า `cardinality` คืน `0` สำหรับ array ว่าง (ไม่ใช่ `NULL`) แต่ยังคง `NULL` สำหรับ input ที่เป็น `NULL` จริง ๆ — นี่คือเหตุผลที่ `cardinality` เป็นตัวเลือกที่ปลอดภัยกว่า `array_length` เมื่อไม่แน่ใจว่า array จะว่างเปล่าหรือไม่

```sql
-- ใช้หาลูกค้าที่ยังไม่มีสินค้าถูกใจเลยสักตัว
SELECT first_name, favorite_product_ids
FROM customers
WHERE cardinality(favorite_product_ids) = 0;
```

```
 first_name | favorite_product_ids
-------------+-----------------------
 Wei         | {}
 Lin         | {}
(2 rows)
```

### `array_append(array, element)` — เพิ่มสมาชิกต่อท้าย

```sql
-- Somchai เพิ่มสินค้า id=7 เข้า wishlist
UPDATE customers
SET favorite_product_ids = array_append(favorite_product_ids, 7)
WHERE first_name = 'Somchai';

SELECT first_name, favorite_product_ids FROM customers WHERE first_name = 'Somchai';
```

```
 first_name | favorite_product_ids
-------------+-----------------------
 Somchai     | {1,4,6,7}
(1 row)
```

ทางเลือกอื่นที่เขียนสั้นกว่าและให้ผลเหมือนกันคือ operator `||` (concatenation):

```sql
UPDATE customers
SET favorite_product_ids = favorite_product_ids || ARRAY[9]
WHERE first_name = 'Somchai';

SELECT first_name, favorite_product_ids FROM customers WHERE first_name = 'Somchai';
```

```
 first_name | favorite_product_ids
-------------+-----------------------
 Somchai     | {1,4,6,7,9}
(1 row)
```

`array_prepend(element, array)` ทำงานตรงข้าม คือเพิ่มสมาชิกไว้ด้านหน้า

```sql
SELECT array_prepend(0, ARRAY[1,2,3]) AS result;
```

```
   result
-------------
 {0,1,2,3}
(1 row)
```

### `array_remove(array, element)` — ลบสมาชิกที่ตรงกับค่าที่กำหนด (ลบทุกตัวที่ตรงกัน)

```sql
-- Somchai เอาสินค้า id=7 ออกจาก wishlist (ยกเลิกใจ)
UPDATE customers
SET favorite_product_ids = array_remove(favorite_product_ids, 7)
WHERE first_name = 'Somchai';

SELECT first_name, favorite_product_ids FROM customers WHERE first_name = 'Somchai';
```

```
 first_name | favorite_product_ids
-------------+-----------------------
 Somchai     | {1,4,6,9}
(1 row)
```

```sql
-- คืนค่าทดสอบให้กลับสภาพเดิมก่อนไปต่อ
UPDATE customers
SET favorite_product_ids = ARRAY[1,4,6]
WHERE first_name = 'Somchai';
```

### `array_position(array, element)` — หาตำแหน่ง (index) ของสมาชิก

```sql
SELECT
    tags,
    array_position(tags, 'new-arrival') AS position_of_new_arrival
FROM products
WHERE product_name = 'Mechanical Keyboard K80';
```

```
                 tags               | position_of_new_arrival
--------------------------------------+---------------------------
 {bestseller,gaming,new-arrival}     |                         3
(1 row)
```

ถ้าไม่พบสมาชิกนั้นเลย จะคืนค่า `NULL`:

```sql
SELECT array_position(ARRAY['a','b','c'], 'z') AS not_found;
```

```
 not_found
-----------

(1 row)
```

### `array_cat(array1, array2)` — รวม array สองก้อนเข้าด้วยกัน

```sql
SELECT array_cat(ARRAY[1,2,3], ARRAY[4,5]) AS combined;
```

```
   combined
---------------
 {1,2,3,4,5}
(1 row)
```

เขียนแบบ `||` ก็ให้ผลเหมือนกัน: `ARRAY[1,2,3] || ARRAY[4,5]`

### `array_to_string(array, delimiter)` — แปลง array เป็น string (มีประโยชน์มากสำหรับแสดงผล)

```sql
SELECT
    product_name,
    array_to_string(tags, ', ') AS tags_display
FROM products
WHERE product_name = 'Bluetooth Earbuds Pro';
```

```
       product_name       |           tags_display
-----------------------------+-------------------------------------
 Bluetooth Earbuds Pro       | bestseller, wireless, new-arrival
(1 row)
```

### `string_to_array(string, delimiter)` — แปลง string กลับเป็น array (ตรงข้ามกับด้านบน)

```sql
SELECT string_to_array('bestseller, wireless, new-arrival', ', ') AS tags_from_string;
```

```
              tags_from_string
----------------------------------------------
 {bestseller,wireless,new-arrival}
(1 row)
```

### ตารางสรุปฟังก์ชัน array ที่สำคัญ

| ฟังก์ชัน | หน้าที่ |
|---|---|
| `array_length(arr, dim)` | นับจำนวนสมาชิก (คืน `NULL` ถ้า array ว่าง) |
| `cardinality(arr)` | นับจำนวนสมาชิก (คืน `0` ถ้า array ว่าง — ปลอดภัยกว่า) |
| `array_append(arr, el)` | เพิ่มสมาชิกท้าย array |
| `array_prepend(el, arr)` | เพิ่มสมาชิกหน้า array |
| `array_remove(arr, el)` | ลบสมาชิกที่ตรงกับค่าที่ระบุทั้งหมด |
| `array_position(arr, el)` | หาตำแหน่งแรกที่พบสมาชิก |
| `array_positions(arr, el)` | หาตำแหน่งทั้งหมดที่พบสมาชิก (คืนเป็น array) |
| `array_cat(a1, a2)` / `\|\|` | รวมสอง array |
| `array_to_string(arr, sep)` | แปลง array เป็น string |
| `string_to_array(str, sep)` | แปลง string เป็น array |
| `unnest(arr)` | แปลง array เป็นแถว (ดู Step 497) |

---

## Step 497: UNNEST — แปลง Array เป็นแถว

`UNNEST` คือฟังก์ชันที่**สำคัญที่สุด**ในบทนี้ เพราะเป็นสะพานเชื่อมระหว่างโลกของ array (ข้อมูลแนวนอนในช่องเดียว) กับโลกของ relational query (ข้อมูลแนวตั้งเป็นแถว) — ทำให้เราสามารถ `GROUP BY`, `JOIN`, หรือ aggregate ข้อมูลที่อยู่ใน array ได้เหมือนกับว่ามันเป็นตารางปกติ

### การใช้งานพื้นฐาน

```sql
SELECT unnest(ARRAY['bestseller', 'wireless', 'new-arrival']) AS tag;
```

```
    tag
-------------
 bestseller
 wireless
 new-arrival
(3 rows)
```

`UNNEST` แปลง array หนึ่งก้อนที่มี 3 สมาชิก ให้กลายเป็น 3 แถว

### UNNEST ร่วมกับตาราง (แบบ LATERAL implicit)

เมื่อใช้ `UNNEST` ในส่วน `SELECT` ของ query ที่มี `FROM` อยู่แล้ว มันจะทำงานเป็น set-returning function ที่ "ขยาย" จำนวนแถวออกมาโดยอัตโนมัติ — เทียบเท่ากับการทำ `CROSS JOIN LATERAL`

```sql
-- แตก tags ของทุกสินค้าออกมาเป็นแถว พร้อมชื่อสินค้า
SELECT product_name, unnest(tags) AS tag
FROM products
WHERE product_name IN ('Wireless Mouse M1', 'Mechanical Keyboard K80')
ORDER BY product_name;
```

```
       product_name       |     tag
-----------------------------+-------------
 Mechanical Keyboard K80     | bestseller
 Mechanical Keyboard K80     | gaming
 Mechanical Keyboard K80     | new-arrival
 Wireless Mouse M1           | bestseller
 Wireless Mouse M1           | wireless
(5 rows)
```

รูปแบบที่แนะนำและชัดเจนกว่าคือเขียนเป็น `JOIN LATERAL` หรือ `CROSS JOIN` แบบชัดเจน:

```sql
SELECT p.product_name, t.tag
FROM products p
CROSS JOIN unnest(p.tags) AS t(tag)
WHERE p.product_name IN ('Wireless Mouse M1', 'Mechanical Keyboard K80')
ORDER BY p.product_name, t.tag;
```

```
       product_name       |     tag
-----------------------------+-------------
 Mechanical Keyboard K80     | bestseller
 Mechanical Keyboard K80     | gaming
 Mechanical Keyboard K80     | new-arrival
 Wireless Mouse M1           | bestseller
 Wireless Mouse M1           | wireless
(5 rows)
```

การเขียนแบบ `CROSS JOIN unnest(...)` มีข้อดีคือตั้งชื่อคอลัมน์ผลลัพธ์ได้ชัดเจน (`t(tag)`) และอ่านง่ายกว่าเมื่อ query ซับซ้อนขึ้น — เป็นรูปแบบที่แนะนำในการเขียนโค้ดจริง

### กรณีการใช้งานที่ 1: นับความนิยมของแต่ละ tag

```sql
SELECT
    unnest(p.tags) AS tag,
    count(*) AS product_count
FROM products p
GROUP BY tag
ORDER BY product_count DESC, tag;
```

```
      tag       | product_count
------------------+---------------
 bestseller       |             9
 new-arrival      |             8
 eco-friendly     |             6
 wireless         |             2
 sports           |             1
 skincare         |             1
 smart-home       |             1
 outdoor          |             1
 accessory        |             1
 gaming           |             1
 kitchen          |             1
 premium          |             1
 clearance        |             1
(13 rows)
```

Query นี้จะเป็นไปไม่ได้เลยถ้าไม่มี `UNNEST` — เพราะ `GROUP BY tags` แบบตรง ๆ จะกลุ่มตาม array ทั้งก้อน (เช่น `{bestseller,wireless}` กับ `{bestseller,new-arrival}` จะถือเป็นคนละกลุ่ม) ไม่ใช่กลุ่มตาม tag แต่ละตัว

### กรณีการใช้งานที่ 2: JOIN wishlist กับตาราง products

```sql
-- หาลิสต์สินค้าที่ลูกค้าแต่ละคนถูกใจ พร้อมชื่อสินค้าจริง
SELECT
    c.first_name,
    c.last_name,
    p.product_name,
    p.unit_price
FROM customers c
CROSS JOIN unnest(c.favorite_product_ids) AS fav(product_id)
JOIN products p ON p.product_id = fav.product_id
WHERE c.first_name = 'Pornthip'
ORDER BY p.product_id;
```

```
 first_name | last_name |     product_name      | unit_price
-------------+-----------+--------------------------+------------
 Pornthip    | Rattana   | Wireless Mouse M1        |     350.00
 Pornthip    | Rattana   | Mechanical Keyboard K80  |    2190.00
 Pornthip    | Rattana   | 27-inch 4K Monitor       |    9990.00
 Pornthip    | Rattana   | Smartphone X12 128GB     |   12900.00
 Pornthip    | Rattana   | Smartphone X12 Case      |     199.00
(5 rows)
```

นี่คือรูปแบบ query ที่สำคัญมากในระบบจริง — การแตก array ของ foreign key ออกมาแล้ว JOIN กับตารางหลักเพื่อดึงข้อมูลเต็มรูปแบบ

### กรณีการใช้งานที่ 3: หาสินค้ายอดนิยมในหมู่ wishlist ของลูกค้าทุกคน

```sql
SELECT
    p.product_name,
    count(*) AS wishlist_count
FROM customers c
CROSS JOIN unnest(c.favorite_product_ids) AS fav(product_id)
JOIN products p ON p.product_id = fav.product_id
GROUP BY p.product_name
ORDER BY wishlist_count DESC, p.product_name
LIMIT 5;
```

```
       product_name        | wishlist_count
------------------------------+-----------------
 Smartphone X12 128GB        |               3
 Wireless Mouse M1           |               2
 Mechanical Keyboard K80     |               2
 Bluetooth Earbuds Pro       |               2
 27-inch 4K Monitor          |               2
(5 rows)
```

### UNNEST หลายคอลัมน์พร้อมกัน (parallel unnest)

ถ้ามี array สองก้อนที่มีความยาวสัมพันธ์กัน สามารถ `unnest` พร้อมกันในบรรทัดเดียว โดย PostgreSQL จะจับคู่ element ตามตำแหน่ง (index) ให้อัตโนมัติ:

```sql
SELECT * FROM unnest(
    ARRAY['bestseller', 'wireless', 'new-arrival'],
    ARRAY[10, 25, 5]
) AS t(tag_name, usage_count);
```

```
   tag_name   | usage_count
---------------+-------------
 bestseller    |          10
 wireless      |          25
 new-arrival   |           5
(3 rows)
```

### WITH ORDINALITY — เก็บลำดับตำแหน่งเดิมของ array

บางครั้งเราต้องการรู้ด้วยว่า element นั้นอยู่ตำแหน่งที่เท่าไหร่ใน array เดิม (เช่น ลำดับความสำคัญของ wishlist) ใช้ `WITH ORDINALITY` ได้:

```sql
SELECT
    c.first_name,
    fav.product_id,
    fav.priority_order
FROM customers c
CROSS JOIN unnest(c.favorite_product_ids) WITH ORDINALITY AS fav(product_id, priority_order)
WHERE c.first_name = 'Somchai';
```

```
 first_name | product_id | priority_order
-------------+------------+-----------------
 Somchai     |          1 |               1
 Somchai     |          4 |               2
 Somchai     |          6 |               3
(3 rows)
```

`priority_order` บอกว่า product_id = 1 ถูกเพิ่มเข้า wishlist เป็นตัวแรก, product_id = 4 เป็นตัวที่สอง และ product_id = 6 เป็นตัวที่สาม — มีประโยชน์มากถ้าต้องการรักษาลำดับความสำคัญของรายการ

---

## Step 498: GIN Index บน Array Column

เมื่อตาราง `products` มีข้อมูลนับล้านแถว การ `WHERE tags @> ARRAY['bestseller']` แบบไม่มี index จะต้องสแกนทุกแถว (sequential scan) ซึ่งช้ามาก — คำตอบคือสร้าง **GIN index** (Generalized Inverted Index) บนคอลัมน์ array

> เชื่อมโยงกับ **Part 042 Step 413** ที่เราแนะนำ GIN index ไปแล้วสำหรับ full-text search — หลักการเดียวกันนี้ใช้ได้กับ array เพราะทั้งสองกรณีเป็นข้อมูลแบบ "หลายค่าในหนึ่งแถว" (multi-valued column) ซึ่งเป็นสถานการณ์ที่ GIN index ถูกออกแบบมาให้รองรับโดยเฉพาะ ต่างจาก B-tree index ที่เหมาะกับค่าเดี่ยว (scalar) มากกว่า

### สร้าง GIN Index

```sql
CREATE INDEX idx_products_tags_gin ON products USING GIN (tags);
CREATE INDEX idx_customers_favorites_gin ON customers USING GIN (favorite_product_ids);
```

### GIN index รองรับ operator ไหนบ้าง

GIN index สำหรับ array type รองรับ operator ต่อไปนี้เป็นหลัก (ทั้งหมดคือ operator ที่เราเรียนใน Step 494):

- `@>` (contains)
- `<@` (contained by)
- `&&` (overlap)
- `=` (equal — แต่ในทางปฏิบัติ GIN index ช่วยเรื่อง `@>`/`&&` ได้ดีกว่า `=` มาก)

```sql
EXPLAIN ANALYZE
SELECT product_name, tags
FROM products
WHERE tags @> ARRAY['bestseller'];
```

```
                                                       QUERY PLAN
--------------------------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on products  (cost=12.14..18.39 rows=2 width=48) (actual time=0.045..0.052 rows=9 loops=1)
   Recheck Cond: (tags @> '{bestseller}'::text[])
   Heap Blocks: exact=1
   ->  Bitmap Index Scan on idx_products_tags_gin  (cost=0.00..12.14 rows=2 width=0) (actual time=0.032..0.032 rows=9 loops=1)
         Index Cond: (tags @> '{bestseller}'::text[])
 Planning Time: 0.180 ms
 Execution Time: 0.078 ms
(7 rows)
```

> **หมายเหตุเกี่ยวกับตัวอย่างข้างต้น:** ข้อมูลตัวอย่างของเรามีเพียง 20 แถว ดังนั้น PostgreSQL planner อาจเลือกทำ **Sequential Scan** แทน Bitmap Index Scan จริง เพราะตารางเล็กเกินกว่าที่การใช้ index จะคุ้มค่ากว่าการสแกนตรง ๆ (planner ประเมินจาก cost) — พฤติกรรมนี้เป็นเรื่องปกติและถูกต้องแล้ว ในระบบจริงที่มีข้อมูลหลักแสนถึงหลักล้านแถว planner จะเลือกใช้ GIN index โดยอัตโนมัติเมื่อคุ้มค่ากว่า ผู้เรียนสามารถทดสอบเปรียบเทียบได้ด้วยการปิด sequential scan ชั่วคราว:

```sql
SET enable_seqscan = off;

EXPLAIN ANALYZE
SELECT product_name, tags
FROM products
WHERE tags @> ARRAY['bestseller'];

RESET enable_seqscan;
```

```
                                                    QUERY PLAN
--------------------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on products  (cost=8.01..12.28 rows=1 width=48) (actual time=0.028..0.035 rows=9 loops=1)
   Recheck Cond: (tags @> '{bestseller}'::text[])
   ->  Bitmap Index Scan on idx_products_tags_gin  (cost=0.00..8.01 rows=1 width=0) (actual time=0.020..0.020 rows=9 loops=1)
         Index Cond: (tags @> '{bestseller}'::text[])
 Planning Time: 0.150 ms
 Execution Time: 0.055 ms
(6 rows)
```

การ `SET enable_seqscan = off` ทำให้เห็นชัดว่า index ถูกเลือกใช้จริงเมื่อไม่มีทางเลือกอื่น (คำสั่งนี้ใช้เพื่อการทดสอบ/สอนเท่านั้น ไม่ควรใช้ใน production)

### GIN index vs B-tree index สำหรับ array

| ลักษณะ | GIN Index | B-tree Index |
|---|---|---|
| เหมาะกับ operator | `@>`, `<@`, `&&` | `=` (ทั้งก้อนเป๊ะ ๆ), `<`, `>` |
| ขนาด index | ใหญ่กว่า, build ช้ากว่า | เล็กกว่า, build เร็วกว่า |
| ความเร็วในการค้นหาบางส่วน (partial match) | เร็วมาก | ทำไม่ได้เลย (ต้องเทียบทั้งก้อน) |
| Insert/Update overhead | สูงกว่า (ต้องอัปเดตหลาย entry ใน index) | ต่ำกว่า |

โดยทั่วไป ถ้าต้องการค้นหาว่า "array มี element X อยู่หรือไม่" (ซึ่งเป็นกรณีส่วนใหญ่ของการใช้งาน array เพื่อทำ tag/filter) ให้ใช้ **GIN index** เสมอ B-tree ใช้ไม่ได้ผลกับ operator เหล่านี้

### ทดสอบกับ dataset ขนาดใหญ่ขึ้น (แนวคิดเพื่อการทดลอง)

ถ้าต้องการเห็นผลต่างของ performance อย่างชัดเจน ผู้เรียนสามารถสร้างข้อมูลจำลองเพิ่มเติมได้ (ไม่บังคับ เป็นแบบฝึกหัดเสริม):

```sql
-- ตัวอย่างแนวคิด (ไม่รันจริงในบทนี้เพื่อไม่ให้ dataset หลักเพี้ยน)
-- INSERT INTO products (product_name, category_id, supplier_id, unit_price, stock_quantity, tags)
-- SELECT
--     'Test Product ' || i,
--     (i % 10) + 1,
--     (i % 10) + 1,
--     (random() * 1000)::numeric(10,2),
--     (random() * 200)::int,
--     ARRAY['tag' || (i % 50)]
-- FROM generate_series(1, 500000) AS i;
```

เมื่อ dataset มีขนาดหลักแสนแถวขึ้นไป ความแตกต่างของเวลาในการ query ระหว่างมี GIN index กับไม่มี จะเห็นได้ชัดเจนมาก (จากหลักร้อย milliseconds เหลือเพียงหลัก millisecond)

---

## Step 499: เมื่อไหร่ควรใช้ Array เทียบกับตารางแยก (Normalization)

นี่คือคำถามเชิงออกแบบที่สำคัญที่สุดของบทนี้ — Array เป็นเครื่องมือที่ทรงพลัง แต่ก็มี trade-off ที่ต้องเข้าใจอย่างถ่องแท้ก่อนตัดสินใจใช้

### ทางเลือกที่แข่งกัน: Array column vs. Junction table

สำหรับ wishlist ของลูกค้า เรามีสองทางเลือกหลัก:

**ทางเลือก A: ใช้ array (แบบที่เราทำในบทนี้)**

```sql
-- customers.favorite_product_ids INTEGER[]
SELECT favorite_product_ids FROM customers WHERE customer_id = 1;
```

**ทางเลือก B: สร้างตารางแยก (normalized)**

```sql
CREATE TABLE customer_favorites (
    customer_id  INTEGER REFERENCES customers(customer_id),
    product_id   INTEGER REFERENCES products(product_id),
    added_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (customer_id, product_id)
);
```

### เปรียบเทียบ trade-off อย่างตรงไปตรงมา

| ประเด็น | Array Column | Junction Table (แยกตาราง) |
|---|---|---|
| **Referential integrity** | PostgreSQL **ไม่มี** foreign key constraint บน element ของ array ได้โดยตรง — ถ้าลบ product ที่มี id อยู่ใน `favorite_product_ids` ของใครสักคน ค่านั้นจะกลายเป็น "ผี" (dangling reference) โดยไม่มี error ใด ๆ เตือน | `REFERENCES` ทำงานได้เต็มรูปแบบ, `ON DELETE CASCADE`/`SET NULL` ป้องกันข้อมูลเพี้ยนได้อัตโนมัติ |
| **ข้อมูลเสริม (metadata) ต่อความสัมพันธ์** | ทำไม่ได้ — เก็บได้แค่ id เปล่า ๆ ถ้าต้องการเก็บ "วันที่เพิ่มเข้า wishlist" ต้องมี array คู่ขนานแยกต่างหาก ซึ่งซับซ้อนและเสี่ยงข้อมูลไม่ sync กัน | เพิ่มคอลัมน์ เช่น `added_at`, `note` ได้ตามสบาย |
| **Query ประสิทธิภาพเมื่อข้อมูลใหญ่มาก** | ดีสำหรับ "ดึงทั้งหมดมาพร้อมแถวหลัก" (ไม่ต้อง JOIN) แต่การ query ข้ามจาก "มุมของ product" (เช่น "product นี้อยู่ใน wishlist กี่คน") ต้อง `UNNEST` ทุกครั้งซึ่งมี overhead | Query ได้ตรงไปตรงมาทั้งสองทิศทางด้วย index ปกติ (`WHERE product_id = ...` หรือ `WHERE customer_id = ...`) |
| **จำนวนสมาชิกที่เหมาะสม** | เหมาะกับจำนวนน้อยถึงปานกลาง (หลักสิบ) — ถ้ามีเป็นพันเป็นหมื่น element จะเทอะทะและช้า | รองรับจำนวนมากได้ดีกว่ามาก เพราะแต่ละแถวคือ record อิสระ |
| **การ update บางส่วน** | `array_append`/`array_remove` ทำงานได้ แต่ทุกครั้งที่ update จะ rewrite ทั้งคอลัมน์ array ใหม่ (PostgreSQL ใช้ MVCC — แถวใหม่ถูกสร้างทุกครั้งที่ update อยู่แล้ว แต่ array column ทำให้ payload การเขียนใหญ่ขึ้นถ้า array มีขนาดใหญ่) | `INSERT`/`DELETE` หนึ่งแถวในตารางลูกเป็น operation เล็กและเร็วเสมอ ไม่ว่าจำนวนความสัมพันธ์ทั้งหมดจะมากแค่ไหน |
| **ความง่ายในการเขียนโค้ด** | เขียนง่ายกว่า ไม่ต้อง JOIN บ่อย ๆ เหมาะกับ "อ่านพร้อมแถวหลักเสมอ" | ต้อง JOIN เพิ่มเสมอ แต่เป็นรูปแบบมาตรฐานที่ ORM ส่วนใหญ่รองรับดีอยู่แล้ว |
| **Index รองรับ** | GIN index เท่านั้นสำหรับ containment query | B-tree ปกติ ทำงานได้ดีทุกกรณี |

### กฎการตัดสินใจ (decision heuristic)

ใช้แนวทางนี้เพื่อตัดสินใจ:

1. **ใช้ array เมื่อ:**
   - ข้อมูลนั้นเป็น "attribute" ของ entity หลักอย่างแท้จริง ไม่ใช่ "ความสัมพันธ์" ที่มีความหมายในตัวเอง (เช่น `tags` เป็นคุณสมบัติของ product ไม่ใช่ entity ที่มีตัวตนเป็นของตัวเอง)
   - จำนวนสมาชิกน้อยและค่อนข้างคงที่ (ไม่เกินหลักสิบ)
   - ไม่ต้องการเก็บ metadata เพิ่มเติมต่อสมาชิกแต่ละตัว
   - อ่านพร้อมกับแถวหลักเกือบทุกครั้ง แทบไม่เคย query แยก
   - ไม่ต้องการ referential integrity ที่เข้มงวด (ยอมรับความเสี่ยงเรื่อง dangling reference ได้)

2. **ใช้ตารางแยก (normalized) เมื่อ:**
   - ความสัมพันธ์นั้นมีความหมายเป็นของตัวเอง หรือมี metadata ประกอบ (เช่น วันที่สั่งซื้อ, จำนวน, สถานะ) — เหมือนที่เราทำกับ `order_items` อยู่แล้ว (ความสัมพันธ์ many-to-many ระหว่าง orders กับ products ที่มี `quantity` และ `unit_price` เป็น metadata)
   - ต้องการ referential integrity ที่รับประกันได้ (`FOREIGN KEY` + `ON DELETE CASCADE`)
   - จำนวนสมาชิกอาจเติบโตมากในระยะยาว
   - ต้องการ query จากทั้งสองทิศทางอย่างมีประสิทธิภาพเท่ากัน

### ตัวอย่างเปรียบเทียบ: ทำไม `order_items` ถึงเป็นตารางแยก แต่ `tags` เป็น array

สังเกตว่าในสคีมาของเราเอง เราใช้ทั้งสองแนวทางพร้อมกัน และนั่นคือการออกแบบที่ถูกต้อง:

- **`order_items`** คือตารางแยก เพราะความสัมพันธ์ order↔product มี metadata สำคัญ (`quantity`, `unit_price` ณ เวลาที่สั่งซื้อ) และต้องรักษาความถูกต้องอย่างเข้มงวด (ประวัติการสั่งซื้อห้ามเพี้ยน)
- **`products.tags`** คือ array เพราะ tag ไม่มี metadata อะไรเพิ่มเติม เป็นเพียง label แบนราบที่ติดอยู่กับสินค้า และมักถูกอ่านพร้อมกับสินค้าเสมอ

### ตัวอย่างปัญหาจริงของ dangling reference ใน array

```sql
-- สมมติเราลบสินค้า id=20 ทิ้ง
DELETE FROM products WHERE product_id = 20;

-- ลูกค้าที่มี id=20 ใน favorite_product_ids จะยังมีค่านั้นค้างอยู่ (ไม่มี error เตือน!)
SELECT first_name, favorite_product_ids
FROM customers
WHERE 20 = ANY(favorite_product_ids);
```

ถ้าเรารัน query ด้านบนก่อนลบ เราจะเห็นว่า Yuki และ Michael มี `20` อยู่ใน `favorite_product_ids` — หลังจากลบสินค้าไปแล้ว ค่า `20` ยังคงอยู่ใน array นั้น แต่กลายเป็น "ชี้ไปยังสินค้าที่ไม่มีอยู่จริง" ซึ่งจะทำให้ query แบบ `JOIN products p ON p.product_id = fav.product_id` เพียงแค่ไม่คืนแถวนั้นออกมา (แทนที่จะ error) — เป็นบั๊กเงียบที่ตรวจจับได้ยากในระบบจริง

```sql
-- คืนค่าข้อมูลกลับ (rollback ตัวอย่างนี้)
INSERT INTO products (product_id, product_name, category_id, supplier_id, unit_price, stock_quantity, is_active, tags)
OVERRIDING SYSTEM VALUE
VALUES (20, 'Discontinued USB Hub', 2, 1, 150.00, 0, false, ARRAY['clearance']);

SELECT setval(pg_get_serial_sequence('products','product_id'), (SELECT max(product_id) FROM products));
```

นี่คือเหตุผลสำคัญว่าทำไมเมื่อใช้ array เก็บ foreign key (เช่น `favorite_product_ids`) ทีมพัฒนาต้อง**จัดการความสอดคล้องของข้อมูลด้วยตัวเองที่ชั้น application** หรือใช้ trigger เสริม ซึ่งเป็นภาระเพิ่มเติมที่ตารางแยกไม่ต้องมี (เพราะ database รับประกันให้อัตโนมัติผ่าน foreign key constraint)

---

## Step 500: แบบฝึกหัดรวม — ระบบ Tag สินค้าและ Wishlist ลูกค้าแบบสมบูรณ์

มาสรุปทุกอย่างที่เรียนมาในบทนี้ด้วยการ implement ระบบที่สมบูรณ์ พร้อม query ค้นหาที่มีประสิทธิภาพ

### 1. ตรวจสอบว่ามี Index ครบถ้วน

```sql
-- ตรวจสอบ index ที่มีอยู่บนตาราง products และ customers
SELECT indexname, indexdef
FROM pg_indexes
WHERE tablename IN ('products', 'customers')
ORDER BY tablename, indexname;
```

```
          indexname          |                                     indexdef
-------------------------------+-----------------------------------------------------------------------------------
 customers_email_key           | CREATE UNIQUE INDEX customers_email_key ON public.customers USING btree (email)
 customers_pkey                | CREATE UNIQUE INDEX customers_pkey ON public.customers USING btree (customer_id)
 idx_customers_favorites_gin   | CREATE INDEX idx_customers_favorites_gin ON public.customers USING gin (favorite_product_ids)
 idx_products_tags_gin         | CREATE INDEX idx_products_tags_gin ON public.products USING gin (tags)
 products_pkey                 | CREATE UNIQUE INDEX products_pkey ON public.products USING btree (product_id)
(5 rows)
```

### 2. Function สำหรับเพิ่ม/ลบ tag แบบปลอดภัย (ไม่ซ้ำ)

```sql
CREATE OR REPLACE FUNCTION add_product_tag(p_product_id INTEGER, p_tag TEXT)
RETURNS VOID AS $$
BEGIN
    UPDATE products
    SET tags = CASE
        WHEN tags @> ARRAY[p_tag] THEN tags          -- มีอยู่แล้ว ไม่ต้องเพิ่มซ้ำ
        ELSE array_append(tags, p_tag)
    END
    WHERE product_id = p_product_id;
END;
$$ LANGUAGE plpgsql;

CREATE OR REPLACE FUNCTION remove_product_tag(p_product_id INTEGER, p_tag TEXT)
RETURNS VOID AS $$
BEGIN
    UPDATE products
    SET tags = array_remove(tags, p_tag)
    WHERE product_id = p_product_id;
END;
$$ LANGUAGE plpgsql;
```

ทดสอบ:

```sql
SELECT add_product_tag(9, 'bestseller');  -- Bamboo Cutting Board: {eco-friendly} -> {eco-friendly,bestseller}
SELECT add_product_tag(9, 'bestseller');  -- เรียกซ้ำ ต้องไม่เพิ่มซ้ำ

SELECT product_name, tags FROM products WHERE product_id = 9;
```

```
       product_name       |          tags
----------------------------+--------------------------
 Bamboo Cutting Board       | {eco-friendly,bestseller}
(1 row)
```

### 3. Function สำหรับเพิ่ม/ลบสินค้าใน wishlist แบบปลอดภัย

```sql
CREATE OR REPLACE FUNCTION toggle_wishlist(p_customer_id INTEGER, p_product_id INTEGER)
RETURNS TEXT AS $$
DECLARE
    v_currently_in BOOLEAN;
BEGIN
    SELECT favorite_product_ids @> ARRAY[p_product_id]
    INTO v_currently_in
    FROM customers WHERE customer_id = p_customer_id;

    IF v_currently_in THEN
        UPDATE customers
        SET favorite_product_ids = array_remove(favorite_product_ids, p_product_id)
        WHERE customer_id = p_customer_id;
        RETURN 'removed';
    ELSE
        UPDATE customers
        SET favorite_product_ids = array_append(favorite_product_ids, p_product_id)
        WHERE customer_id = p_customer_id;
        RETURN 'added';
    END IF;
END;
$$ LANGUAGE plpgsql;
```

ทดสอบ:

```sql
SELECT toggle_wishlist(5, 3);   -- Wei เพิ่ม product_id=3 เข้า wishlist
SELECT toggle_wishlist(5, 3);   -- เรียกซ้ำ ต้องลบออก (toggle)

SELECT first_name, favorite_product_ids FROM customers WHERE customer_id = 5;
```

```
 first_name | favorite_product_ids
-------------+-----------------------
 Wei         | {}
(1 row)
```

### 4. View สรุป: สินค้าพร้อมจำนวนคนถูกใจ

```sql
CREATE OR REPLACE VIEW v_product_popularity AS
SELECT
    p.product_id,
    p.product_name,
    p.tags,
    p.unit_price,
    count(fav.customer_id) AS wishlist_count
FROM products p
LEFT JOIN (
    SELECT c.customer_id, unnest(c.favorite_product_ids) AS product_id
    FROM customers c
) fav ON fav.product_id = p.product_id
GROUP BY p.product_id, p.product_name, p.tags, p.unit_price
ORDER BY wishlist_count DESC, p.product_name;

SELECT * FROM v_product_popularity LIMIT 5;
```

```
 product_id |      product_name       |               tags               | unit_price | wishlist_count
-------------+--------------------------+-----------------------------------+------------+-----------------
           4 | Smartphone X12 128GB    | {bestseller,new-arrival}          |   12900.00 |               3
           1 | Wireless Mouse M1       | {bestseller,wireless}             |     350.00 |               2
           2 | Mechanical Keyboard K80 | {bestseller,gaming,new-arrival}   |    2190.00 |               2
           6 | Bluetooth Earbuds Pro   | {bestseller,wireless,new-arrival} |    1590.00 |               2
           3 | 27-inch 4K Monitor      | {bestseller,new-arrival}          |    9990.00 |               2
(5 rows)
```

### 5. Query ค้นหาสินค้าแบบ multi-tag filter (จำลองหน้า filter ของเว็บ e-commerce)

```sql
-- ผู้ใช้เลือก filter: ต้องการสินค้าที่เป็นทั้ง 'bestseller' และ 'eco-friendly'
-- และราคาต้องไม่เกิน 1000 บาท
EXPLAIN (COSTS OFF)
SELECT product_name, unit_price, tags
FROM products
WHERE tags @> ARRAY['bestseller', 'eco-friendly']
  AND unit_price <= 1000
  AND is_active = true
ORDER BY unit_price;
```

```sql
SELECT product_name, unit_price, tags
FROM products
WHERE tags @> ARRAY['bestseller', 'eco-friendly']
  AND unit_price <= 1000
  AND is_active = true
ORDER BY unit_price;
```

```
        product_name          | unit_price |          tags
----------------------------------+------------+---------------------------
 Bamboo Cutting Board             |     250.00 | {eco-friendly,bestseller}
 Stainless Steel Water Bottle     |     290.00 | {eco-friendly,bestseller}
 Organic Cotton T-Shirt           |     350.00 | {eco-friendly,bestseller}
 Yoga Mat Premium                 |     790.00 | {eco-friendly,bestseller}
(4 rows)
```

### 6. Query แนะนำสินค้า: "คนที่ถูกใจสินค้านี้ ก็มักถูกใจสินค้าอื่นด้วย" (collaborative filtering แบบง่าย)

```sql
-- หาสินค้าที่มักถูกใจร่วมกับ product_id = 4 (Smartphone X12 128GB)
WITH customers_who_like_4 AS (
    SELECT customer_id
    FROM customers
    WHERE favorite_product_ids @> ARRAY[4]
)
SELECT
    p.product_name,
    count(*) AS co_favorite_count
FROM customers_who_like_4 cw
JOIN customers c ON c.customer_id = cw.customer_id
CROSS JOIN unnest(c.favorite_product_ids) AS fav(product_id)
JOIN products p ON p.product_id = fav.product_id
WHERE fav.product_id != 4
GROUP BY p.product_name
ORDER BY co_favorite_count DESC;
```

```
       product_name       | co_favorite_count
-----------------------------+---------------------
 Wireless Mouse M1           |                   2
 27-inch 4K Monitor          |                   2
 Bluetooth Earbuds Pro       |                   1
 Mechanical Keyboard K80     |                   1
 Smartphone X12 Case         |                   1
```

Query นี้แสดงให้เห็นพลังของการรวม array operator (`@>`) เข้ากับ `UNNEST` และ `JOIN` เพื่อสร้างฟีเจอร์ recommendation อย่างง่ายโดยไม่ต้องพึ่งระบบภายนอกใด ๆ

ระบบที่เราสร้างขึ้นในหัวข้อนี้แสดงให้เห็นวงจรที่สมบูรณ์ของการใช้ array ใน production: การประกาศคอลัมน์, การ maintain ข้อมูลอย่างปลอดภัยผ่านฟังก์ชัน, การสร้าง index ให้เหมาะสม, และการเขียน query ที่ทั้งถูกต้องและมีประสิทธิภาพ

---

## สรุปท้ายบท

ในบทนี้เราได้เรียนรู้การทำงานกับ **Array type** ซึ่งเป็นความสามารถเฉพาะตัวที่ทำให้ PostgreSQL แตกต่างจาก RDBMS ทั่วไป:

- Array เก็บหลายค่าในคอลัมน์เดียวได้ เหมาะกับข้อมูลที่เป็น attribute แบนราบของ entity หลัก เช่น tags หรือ wishlist
- INSERT ค่า array ทำได้ทั้งด้วย `ARRAY[...]` (แนะนำ) และ `'{...}'` literal syntax
- **Index ของ array เริ่มที่ 1 ไม่ใช่ 0** — ความผิดพลาดที่พบบ่อยที่สุดสำหรับผู้เริ่มต้น และการเข้าถึง index นอกขอบเขตจะคืน `NULL` ไม่ error
- Array slicing (`arr[1:3]`) ปลอดภัยแม้ index จะเกินขอบเขตจริงของ array
- Operator `@>`, `<@`, `&&`, `=` คือหัวใจของการ query ข้อมูล array — โดยเฉพาะ `@>` (contains) และ `&&` (overlap) ที่ใช้บ่อยที่สุดในระบบ filter/tag จริง
- `ANY`/`ALL` เป็นทางเลือกที่ยืดหยุ่นกว่า `IN` โดยเฉพาะเมื่อค่ามาจาก array parameter หรือคอลัมน์ array อยู่แล้ว
- ฟังก์ชัน `array_length`, `cardinality`, `array_append`, `array_remove`, `array_position` ช่วยจัดการ array ได้ครบวงจร — จำไว้ว่า `cardinality` ปลอดภัยกว่า `array_length` เมื่อ array อาจว่างเปล่า
- `UNNEST` คือฟังก์ชันที่สำคัญที่สุดในการเชื่อมโลก array กับโลก relational — ทำให้ `GROUP BY`, `JOIN`, aggregate กับข้อมูลใน array ได้
- **GIN Index** จำเป็นสำหรับการค้นหา containment (`@>`, `<@`, `&&`) บน array ขนาดใหญ่ให้เร็วขึ้นอย่างมีนัยสำคัญ
- การตัดสินใจระหว่าง array กับตารางแยก (normalization) ต้องพิจารณา referential integrity, metadata ที่ต้องเก็บ, และขนาดของข้อมูลอย่างรอบคอบ — ไม่มีคำตอบเดียวที่ถูกเสมอไป

Array เป็นเครื่องมือที่ทรงพลังเมื่อใช้ถูกที่ถูกเวลา แต่ก็เป็นเครื่องมือที่ถูกใช้ผิดบ่อยเช่นกัน — โดยเฉพาะเมื่อใช้แทนที่ตารางแยกในกรณีที่ต้องการ referential integrity ที่เข้มงวด บทถัดไปเราจะเรียนรู้อีกหนึ่ง flexible data type ที่ทรงพลังยิ่งกว่า นั่นคือ **JSON/JSONB** ซึ่งเหมาะกับข้อมูลที่มีโครงสร้างซับซ้อนกว่า array ธรรมดา

---

## แบบฝึกหัด

<details>
<summary><b>แบบฝึกหัดที่ 1:</b> เขียน query หาสินค้าทั้งหมดที่มี tag 'new-arrival' โดยใช้ operator ที่เหมาะสมที่สุด</summary>

```sql
SELECT product_name, tags
FROM products
WHERE tags @> ARRAY['new-arrival']
ORDER BY product_name;
```

ใช้ `@>` เพราะเป็นการเช็คว่า array มีสมาชิกที่ระบุอยู่หรือไม่ (จะใช้ `= ANY(tags)` แบบกลับด้านก็ได้ผลเหมือนกัน: `WHERE 'new-arrival' = ANY(tags)`)
</details>

<details>
<summary><b>แบบฝึกหัดที่ 2:</b> เขียน query หาลูกค้าที่ยังไม่มีสินค้าใดเลยใน wishlist (favorite_product_ids ว่างเปล่า)</summary>

```sql
SELECT first_name, last_name, favorite_product_ids
FROM customers
WHERE cardinality(favorite_product_ids) = 0;
```

ใช้ `cardinality()` แทน `array_length()` เพราะ `array_length` จะคืน `NULL` สำหรับ array ว่าง ทำให้เงื่อนไข `= 0` ไม่ true (เนื่องจาก `NULL = 0` เป็น `NULL` ไม่ใช่ `false`) จึงพลาดแถวที่ต้องการไป
</details>

<details>
<summary><b>แบบฝึกหัดที่ 3:</b> เขียน query หา element ตัวที่ 2 ของ tags ของสินค้าทุกตัว (ถ้าสินค้ามี tag น้อยกว่า 2 ตัว ให้แสดง NULL)</summary>

```sql
SELECT product_name, tags, tags[2] AS second_tag
FROM products
ORDER BY product_name;
```

การเข้าถึง index ที่เกินจำนวนสมาชิกจริงของ array (เช่น สินค้าที่มี tag เดียว) จะคืนค่า `NULL` โดยอัตโนมัติ ไม่ error
</details>

<details>
<summary><b>แบบฝึกหัดที่ 4:</b> เขียน query หาสินค้าที่มี tag ตรงกับอย่างน้อยหนึ่งใน ('gaming', 'sports', 'outdoor') พร้อมเรียงตามราคาจากมากไปน้อย</summary>

```sql
SELECT product_name, unit_price, tags
FROM products
WHERE tags && ARRAY['gaming', 'sports', 'outdoor']
ORDER BY unit_price DESC;
```

```
       product_name        | unit_price |              tags
------------------------------+------------+---------------------------------
 Camping Tent 4-Person        |    4990.00 | {new-arrival,outdoor}
 Running Shoes AirFlex        |    2490.00 | {bestseller,sports}
 Mechanical Keyboard K80      |    2190.00 | {bestseller,gaming,new-arrival}
(3 rows)
```

ใช้ `&&` (overlap) เพราะต้องการ "มีอย่างน้อยหนึ่งตัวตรงกัน" ไม่ใช่ "ต้องมีครบทุกตัว" (ซึ่งจะใช้ `@>`)
</details>

<details>
<summary><b>แบบฝึกหัดที่ 5:</b> เขียน query แสดงจำนวนสินค้าใน wishlist ของลูกค้าแต่ละคน เรียงจากมากไปน้อย โดยใช้ cardinality</summary>

```sql
SELECT
    first_name,
    last_name,
    cardinality(favorite_product_ids) AS wishlist_size
FROM customers
ORDER BY wishlist_size DESC, first_name;
```

```
 first_name | last_name | wishlist_size
-------------+-----------+----------------
 Pornthip    | Rattana   |              5
 Somchai     | Jaidee    |              3
 Nalinee     | Suksawat  |              3
 Emma        | Johnson   |              3
 Kittipong   | Meesuk    |              3
 ...
```
</details>

<details>
<summary><b>แบบฝึกหัดที่ 6:</b> ใช้ UNNEST เขียน query แสดงคู่ (customer, product_name) ของทุกรายการใน wishlist ที่มีราคาสูงกว่า 5000 บาท</summary>

```sql
SELECT
    c.first_name,
    p.product_name,
    p.unit_price
FROM customers c
CROSS JOIN unnest(c.favorite_product_ids) AS fav(product_id)
JOIN products p ON p.product_id = fav.product_id
WHERE p.unit_price > 5000
ORDER BY p.unit_price DESC;
```

```
 first_name |     product_name      | unit_price
-------------+--------------------------+------------
 John        | Smartphone X12 128GB    |   12900.00
 Pornthip    | Smartphone X12 128GB    |   12900.00
 Somchai     | Smartphone X12 128GB    |   12900.00
 Pornthip    | 27-inch 4K Monitor      |    9990.00
 John        | 27-inch 4K Monitor      |    9990.00
```
</details>

<details>
<summary><b>แบบฝึกหัดที่ 7:</b> เขียนฟังก์ชัน SQL ที่รับ product_id และคืนค่าจำนวนลูกค้าที่มีสินค้านั้นใน wishlist โดยใช้ operator ที่มี GIN index รองรับ</summary>

```sql
CREATE OR REPLACE FUNCTION count_wishlist_fans(p_product_id INTEGER)
RETURNS INTEGER AS $$
    SELECT count(*)::INTEGER
    FROM customers
    WHERE favorite_product_ids @> ARRAY[p_product_id];
$$ LANGUAGE sql STABLE;

SELECT count_wishlist_fans(4);
```

```
 count_wishlist_fans
----------------------
                    3
(1 row)
```

ใช้ `@>` แทน `= ANY(...)` เพราะ `@>` เป็น operator ที่ GIN index รองรับโดยตรง ทำให้ performance ดีกว่าเมื่อข้อมูลมีขนาดใหญ่ (แม้ผลลัพธ์เชิงตรรกะจะเทียบเท่ากับ `p_product_id = ANY(favorite_product_ids)`)
</details>

<details>
<summary><b>แบบฝึกหัดที่ 8:</b> เขียน query หาสินค้าที่มี tag ครบทุกตัวตรงกับ ('bestseller', 'wireless') พอดี ไม่มากไม่น้อยกว่านั้น (เซตเดียวกันโดยไม่สนใจลำดับ)</summary>

```sql
SELECT product_name, tags
FROM products
WHERE tags @> ARRAY['bestseller', 'wireless']
  AND tags <@ ARRAY['bestseller', 'wireless'];
```

```
    product_name    |          tags
---------------------+-------------------------
 Wireless Mouse M1   | {bestseller,wireless}
(1 row)
```

ใช้ `@>` ร่วมกับ `<@` ทั้งสองทิศทางเพื่อยืนยันว่าเป็น "เซตเดียวกันทุกประการ" โดยไม่สนใจลำดับของ element (ต่างจาก `=` ที่สนใจลำดับด้วย)
</details>

<details>
<summary><b>แบบฝึกหัดที่ 9:</b> สร้าง GIN index บนคอลัมน์ tags ถ้ายังไม่มี แล้วเขียน query อธิบาย (EXPLAIN) เพื่อตรวจสอบว่า index ถูกนำมาใช้หรือไม่ พร้อมอธิบายว่าทำไม planner อาจไม่เลือกใช้ index</summary>

```sql
CREATE INDEX IF NOT EXISTS idx_products_tags_gin ON products USING GIN (tags);

EXPLAIN ANALYZE
SELECT product_name FROM products WHERE tags @> ARRAY['bestseller'];
```

หากตารางมีข้อมูลน้อย (เช่น 20 แถวในบทนี้) PostgreSQL planner มักจะเลือก **Sequential Scan** แทนที่จะใช้ GIN index แม้ index จะมีอยู่จริง เพราะ cost ของการอ่าน index แล้วย้อนกลับไปอ่านตาราง (bitmap heap scan) อาจสูงกว่าการสแกนตารางทั้งหมดตรง ๆ เมื่อข้อมูลมีจำนวนน้อย — planner ตัดสินใจจาก cost estimation ไม่ใช่จากการมีอยู่ของ index เพียงอย่างเดียว เมื่อข้อมูลมีจำนวนมากขึ้น (หลักหมื่นถึงหลักล้านแถว) planner จะเลือกใช้ index โดยอัตโนมัติเมื่อคุ้มค่ากว่า
</details>

<details>
<summary><b>แบบฝึกหัดที่ 10:</b> อภิปราย (เขียนคำตอบเป็นข้อความ) ว่าถ้าต้องเก็บ "วันที่ลูกค้าเพิ่มสินค้าแต่ละชิ้นเข้า wishlist" ควรออกแบบสคีมาอย่างไร ระหว่างใช้ array คู่ขนานสองคอลัมน์ กับสร้างตารางแยก และเพราะเหตุใด</summary>

**คำตอบที่แนะนำ:** ควรสร้าง**ตารางแยก** (`customer_favorites` ที่มี `customer_id`, `product_id`, `added_at`) แทนการใช้ array คู่ขนาน (เช่น `favorite_product_ids INTEGER[]` คู่กับ `favorite_added_dates TIMESTAMPTZ[]`)

เหตุผล:

1. **Array คู่ขนานเสี่ยงข้อมูลไม่ sync กัน** — ไม่มีอะไรบังคับว่าทั้งสอง array ต้องมีความยาวเท่ากันหรือเรียงตรงตำแหน่งกันเสมอ ถ้า code ที่ไหนสักแห่ง `array_append` เข้าคอลัมน์เดียวโดยลืมอีกคอลัมน์ ข้อมูลจะเพี้ยนทันทีโดยไม่มี constraint ใดจับได้
2. **Metadata ต่อความสัมพันธ์ (added_at)** คือสัญญาณชัดเจนว่าความสัมพันธ์นี้ "มีความหมายเป็นของตัวเอง" ไม่ใช่แค่ attribute แบนราบแล้ว — ตามหลักที่กล่าวไว้ใน Step 499 กรณีนี้ควรใช้ junction table
3. **Referential integrity** — ตารางแยกสามารถกำหนด `FOREIGN KEY ... ON DELETE CASCADE` ได้ ป้องกันไม่ให้เกิด dangling reference เมื่อสินค้าถูกลบ
4. **Query ที่ซับซ้อนขึ้น** เช่น "หาสินค้าที่ถูกเพิ่มเข้า wishlist ในช่วง 7 วันล่าสุด" เขียนด้วย SQL ปกติกับตารางแยกได้ง่ายและมีประสิทธิภาพกว่าการ `UNNEST` สอง array พร้อมกันแล้วจับคู่ตามตำแหน่งซึ่งเสี่ยงผิดพลาดสูง

บทเรียนสำคัญ: ทันทีที่ความสัมพันธ์ต้องการ "ข้อมูลเสริม" มากกว่าหนึ่งอย่าง สัญญาณนี้บ่งบอกว่าถึงเวลาเปลี่ยนจาก array ไปเป็นตารางแยกแล้ว
</details>

---

**บทถัดไป:** [Part 051: JSON และ JSONB พื้นฐาน](./part-051-json-jsonb-basics.md)
