# Primary Key และ Unique Constraint

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับพื้นฐาน | Part 016

ใน Part ที่แล้วเราได้รู้จักชนิดข้อมูล (Data Types) ต่าง ๆ ของ PostgreSQL ไปแล้ว มาถึง Part นี้เราจะเริ่มเข้าสู่เรื่อง **Constraints** ซึ่งเป็นกลไกสำคัญที่ทำให้ฐานข้อมูลของเรา "เชื่อถือได้" (data integrity) โดยจะเริ่มจากสอง constraint ที่ใช้บ่อยที่สุดคือ **Primary Key** และ **Unique Constraint**

Constraints คือกฎที่เรากำหนดให้กับคอลัมน์หรือตาราง เพื่อบังคับว่าข้อมูลที่จะถูกเขียนลงไปต้องเป็นไปตามเงื่อนไขที่ตั้งไว้เสมอ ถ้าข้อมูลใดไม่ผ่านเงื่อนไข PostgreSQL จะปฏิเสธคำสั่งนั้นทันทีพร้อมแสดง error — นี่คือหัวใจของการออกแบบฐานข้อมูลเชิงสัมพันธ์ (Relational Database) ที่ทำให้ข้อมูลไม่ผิดเพี้ยนแม้จะมีแอปพลิเคชันหลายตัวเขียนเข้ามาพร้อมกัน

## เป้าหมายการเรียนรู้

เมื่อจบ Part นี้ ผู้เรียนจะสามารถ:

- อธิบายได้ว่า Primary Key คืออะไร และเหตุใดทุกตารางควรมี Primary Key
- กำหนด Primary Key ได้ทั้งแบบ column-level และ table-level syntax
- สร้างและใช้งาน Composite Primary Key สำหรับตารางเชื่อมความสัมพันธ์ (junction table)
- เพิ่มและลบ Primary Key ด้วย `ALTER TABLE` บนตารางที่มีอยู่แล้ว
- เข้าใจความแตกต่างระหว่าง Primary Key และ Unique Constraint
- ใช้ Composite Unique Constraint และ `UNIQUE NULLS NOT DISTINCT` (PostgreSQL 15+)
- เข้าใจว่า PostgreSQL สร้าง Index อัตโนมัติให้กับ Primary Key/Unique Constraint อย่างไร
- วินิจฉัยและแก้ไข error `duplicate key value violates unique constraint` ได้
- เปรียบเทียบข้อดีข้อเสียของการใช้ integer/bigint identity กับ UUID เป็น Primary Key
- ออกแบบ constraint ที่เหมาะสมให้กับตารางจริงในระบบร้านกาแฟและร้านค้าออนไลน์

## เตรียมข้อมูล

เราจะยังคงใช้ฐานข้อมูลร้านกาแฟ (coffee shop) ต่อเนื่องจาก Part ก่อนหน้า หากยังไม่มีตารางเหล่านี้ในฐานข้อมูลของท่าน ให้รันคำสั่งต่อไปนี้เพื่อสร้างใหม่ทั้งหมด (สคริปต์นี้ลบตารางเดิมทิ้งก่อนสร้าง เพื่อให้ทุกคนเริ่มต้นจากจุดเดียวกัน):

```sql
-- ลบตารางเดิม (เรียงลำดับตามความสัมพันธ์ เพื่อไม่ให้ติด foreign key)
DROP TABLE IF EXISTS order_items;
DROP TABLE IF EXISTS orders;
DROP TABLE IF EXISTS products;
DROP TABLE IF EXISTS customers;
DROP TABLE IF EXISTS categories;

-- ตารางหมวดหมู่สินค้า
CREATE TABLE categories (
    category_id   INTEGER,
    category_name VARCHAR(50) NOT NULL,
    description   TEXT
);

-- ตารางสินค้า
CREATE TABLE products (
    product_id    INTEGER,
    product_name  VARCHAR(100) NOT NULL,
    category_id   INTEGER,
    price         NUMERIC(10, 2) NOT NULL,
    is_active     BOOLEAN DEFAULT true
);

-- ตารางลูกค้า
CREATE TABLE customers (
    customer_id   INTEGER,
    full_name     VARCHAR(100) NOT NULL,
    email         VARCHAR(255),
    phone         VARCHAR(20),
    member_code   VARCHAR(20),
    created_at    TIMESTAMP DEFAULT now()
);

-- ตารางคำสั่งซื้อ
CREATE TABLE orders (
    order_id      INTEGER,
    customer_id   INTEGER,
    order_date    TIMESTAMP DEFAULT now(),
    status        VARCHAR(20) DEFAULT 'pending'
);

-- ตารางรายการสินค้าในคำสั่งซื้อ (junction table)
CREATE TABLE order_items (
    order_id      INTEGER,
    product_id    INTEGER,
    quantity      INTEGER NOT NULL,
    unit_price    NUMERIC(10, 2) NOT NULL
);

-- ข้อมูลตัวอย่าง
INSERT INTO categories (category_id, category_name, description) VALUES
    (1, 'Coffee',   'เครื่องดื่มกาแฟทุกชนิด'),
    (2, 'Non-Coffee', 'เครื่องดื่มที่ไม่ใช่กาแฟ'),
    (3, 'Bakery',   'ขนมและเบเกอรี่');

INSERT INTO products (product_id, product_name, category_id, price) VALUES
    (1, 'Espresso',       1, 55.00),
    (2, 'Latte',          1, 65.00),
    (3, 'Cappuccino',     1, 65.00),
    (4, 'Matcha Latte',   2, 70.00),
    (5, 'Croissant',      3, 45.00);

INSERT INTO customers (customer_id, full_name, email, phone, member_code) VALUES
    (1, 'สมชาย ใจดี',   'somchai@example.com', '0811111111', 'MEM001'),
    (2, 'สมหญิง รักเรียน', 'somying@example.com', '0822222222', 'MEM002');
```

สังเกตว่าโครงสร้างข้างต้น **ยังไม่มี** Primary Key หรือ Unique Constraint ใด ๆ เลย — นี่คือจุดเริ่มต้นที่เราจะค่อย ๆ เติมเข้าไปทีละ Step เพื่อให้เห็นภาพชัดเจนว่าทำไมเราต้องใช้ constraint เหล่านี้

---

## Step 151: Primary Key คืออะไร

**Primary Key (PK)** คือคอลัมน์ (หรือกลุ่มคอลัมน์) ที่ใช้เป็น **เอกลักษณ์เฉพาะของแต่ละแถว** ในตาราง กล่าวคือทุกแถวในตารางต้องมีค่า Primary Key ที่ไม่ซ้ำกัน และไม่สามารถเป็นค่าว่าง (`NULL`) ได้

Primary Key มีคุณสมบัติหลัก 3 ประการ:

1. **Uniqueness (ไม่ซ้ำกัน)** — ไม่มีสองแถวใดในตารางที่มีค่า Primary Key เหมือนกัน
2. **Not Null (ห้ามเป็นค่าว่าง)** — ทุกแถวต้องมีค่า Primary Key เสมอ ไม่สามารถเว้นว่างได้
3. **Immutability (ควรคงที่)** — แม้ PostgreSQL จะไม่ได้บังคับห้ามแก้ไขค่า PK แต่ตามหลักการออกแบบที่ดี ค่า Primary Key ไม่ควรเปลี่ยนแปลงตลอดอายุของแถวนั้น เพราะมีตารางอื่นอ้างอิงถึงมันด้วย Foreign Key (จะเรียนใน Part 017)

ลองดูตัวอย่างปัญหาที่จะเกิดขึ้นถ้า **ไม่มี** Primary Key:

```sql
-- ไม่มี Primary Key จึงแทรกข้อมูลซ้ำได้โดยไม่มีการเตือน
INSERT INTO customers (customer_id, full_name, email, phone, member_code)
VALUES (1, 'สมชาย ใจดี', 'somchai2@example.com', '0899999999', 'MEM099');

SELECT * FROM customers WHERE customer_id = 1;
```

ผลลัพธ์:

```
 customer_id |   full_name    |        email         |    phone    | member_code |         created_at
-------------+----------------+-----------------------+-------------+-------------+----------------------------
           1 | สมชาย ใจดี      | somchai@example.com   | 0811111111  | MEM001      | 2026-09-25 10:00:00.123456
           1 | สมชาย ใจดี      | somchai2@example.com  | 0899999999  | MEM099      | 2026-09-25 10:05:12.654321
(2 rows)
```

เกิดปัญหาทันที — มีลูกค้ารหัส `1` สองคน! ถ้าระบบอ้างอิง `customer_id = 1` ในตาราง `orders` เราจะไม่มีทางรู้เลยว่าคำสั่งซื้อนั้นเป็นของลูกค้าคนไหนกันแน่ นี่คือเหตุผลว่าทำไม **แทบทุกตารางในฐานข้อมูลเชิงสัมพันธ์ควรมี Primary Key**

ก่อนไปต่อ ให้เราลบแถวซ้ำที่เพิ่งสร้างออกก่อน:

```sql
DELETE FROM customers WHERE email = 'somchai2@example.com';
```

> **หมายเหตุ:** ในทางทฤษฎี ตารางที่ไม่มี Primary Key ยังคง "ใช้งานได้" ตามกฎของ SQL แต่ในทางปฏิบัติแทบไม่มีเหตุผลที่ดีเลยที่จะออกแบบตารางแบบนั้น ยกเว้นตาราง log บางประเภทที่ไม่สนใจความซ้ำ ซึ่งก็ยังแนะนำให้มีคอลัมน์ identity เป็น PK อยู่ดี

---

## Step 152: การกำหนด Primary Key ตอนสร้างตาราง

การกำหนด Primary Key ทำได้ 2 รูปแบบ คือ **column-level constraint** (เขียนต่อท้ายคอลัมน์โดยตรง) และ **table-level constraint** (เขียนแยกเป็นบรรทัดท้ายตาราง)

### 152.1 Column-level syntax

ใช้ได้เมื่อ Primary Key มีเพียงคอลัมน์เดียว เขียนคำว่า `PRIMARY KEY` ต่อท้ายชนิดข้อมูลของคอลัมน์นั้นได้เลย:

```sql
DROP TABLE IF EXISTS categories CASCADE;

CREATE TABLE categories (
    category_id   INTEGER PRIMARY KEY,
    category_name VARCHAR(50) NOT NULL,
    description   TEXT
);
```

### 152.2 Table-level syntax

เขียนแยกเป็นบรรทัดต่างหากหลังประกาศคอลัมน์ทั้งหมด โดยใช้ `CONSTRAINT ชื่อ PRIMARY KEY (คอลัมน์)` — รูปแบบนี้ **แนะนำมากกว่า** เพราะเราตั้งชื่อ constraint เองได้ ทำให้อ่าน error message และจัดการภายหลังง่ายขึ้น:

```sql
DROP TABLE IF EXISTS categories CASCADE;

CREATE TABLE categories (
    category_id   INTEGER,
    category_name VARCHAR(50) NOT NULL,
    description   TEXT,
    CONSTRAINT pk_categories PRIMARY KEY (category_id)
);
```

ทั้งสองรูปแบบให้ผลลัพธ์เหมือนกันทุกประการ — ถ้าไม่ตั้งชื่อเอง PostgreSQL จะตั้งชื่อ constraint ให้อัตโนมัติตามรูปแบบ `ชื่อตาราง_pkey` เช่น `categories_pkey`

ลองสร้างตารางอื่น ๆ ให้ครบด้วย Primary Key แบบ column-level กันก่อน (ตาราง `order_items` จะทำใน Step ถัดไปเพราะต้องใช้ composite key):

```sql
DROP TABLE IF EXISTS products CASCADE;
DROP TABLE IF EXISTS customers CASCADE;
DROP TABLE IF EXISTS orders CASCADE;

CREATE TABLE products (
    product_id    INTEGER PRIMARY KEY,
    product_name  VARCHAR(100) NOT NULL,
    category_id   INTEGER,
    price         NUMERIC(10, 2) NOT NULL,
    is_active     BOOLEAN DEFAULT true
);

CREATE TABLE customers (
    customer_id   INTEGER PRIMARY KEY,
    full_name     VARCHAR(100) NOT NULL,
    email         VARCHAR(255),
    phone         VARCHAR(20),
    member_code   VARCHAR(20),
    created_at    TIMESTAMP DEFAULT now()
);

CREATE TABLE orders (
    order_id      INTEGER PRIMARY KEY,
    customer_id   INTEGER,
    order_date    TIMESTAMP DEFAULT now(),
    status        VARCHAR(20) DEFAULT 'pending'
);

-- เติมข้อมูลกลับเข้าไปใหม่
INSERT INTO products (product_id, product_name, category_id, price) VALUES
    (1, 'Espresso',       1, 55.00),
    (2, 'Latte',          1, 65.00),
    (3, 'Cappuccino',     1, 65.00),
    (4, 'Matcha Latte',   2, 70.00),
    (5, 'Croissant',      3, 45.00);

INSERT INTO customers (customer_id, full_name, email, phone, member_code) VALUES
    (1, 'สมชาย ใจดี',   'somchai@example.com', '0811111111', 'MEM001'),
    (2, 'สมหญิง รักเรียน', 'somying@example.com', '0822222222', 'MEM002');

INSERT INTO categories (category_id, category_name, description) VALUES
    (1, 'Coffee',   'เครื่องดื่มกาแฟทุกชนิด'),
    (2, 'Non-Coffee', 'เครื่องดื่มที่ไม่ใช่กาแฟ'),
    (3, 'Bakery',   'ขนมและเบเกอรี่');
```

ตอนนี้ลองทดสอบว่า constraint ทำงานจริงหรือไม่:

```sql
-- ทดสอบ 1: แทรกค่า category_id ซ้ำ
INSERT INTO categories (category_id, category_name) VALUES (1, 'Snack');
```

```
ERROR:  duplicate key value violates unique constraint "categories_pkey"
DETAIL:  Key (category_id)=(1) already exists.
```

```sql
-- ทดสอบ 2: แทรกค่า category_id เป็น NULL
INSERT INTO categories (category_id, category_name) VALUES (NULL, 'Snack');
```

```
ERROR:  null value in column "category_id" of relation "categories" violates not-null constraint
DETAIL:  Failing row contains (null, Snack, null).
```

จะเห็นว่า Primary Key บังคับทั้งสองเงื่อนไขให้เราอัตโนมัติ โดยเบื้องหลัง PostgreSQL จะเพิ่ม `NOT NULL` ให้กับคอลัมน์ที่เป็น PK โดยอัตโนมัติเสมอ ไม่ว่าเราจะเขียน `NOT NULL` เองหรือไม่ก็ตาม

---

## Step 153: Composite Primary Key — Primary Key ที่ประกอบด้วยหลายคอลัมน์

บางตารางไม่มีคอลัมน์เดียวที่สามารถระบุความเป็นเอกลักษณ์ของแถวได้ จำเป็นต้องใช้ **หลายคอลัมน์รวมกัน** เป็น Primary Key เรียกว่า **Composite Primary Key** (หรือ Composite Key)

ตัวอย่างคลาสสิกคือตาราง `order_items` ซึ่งเก็บว่า "คำสั่งซื้อไหน มีสินค้าอะไรบ้าง" — คอลัมน์ `order_id` เพียงอย่างเดียวไม่สามารถเป็น PK ได้ เพราะคำสั่งซื้อเดียวมีสินค้าหลายรายการ และคอลัมน์ `product_id` เพียงอย่างเดียวก็ไม่สามารถเป็น PK ได้เช่นกัน เพราะสินค้าชนิดเดียวกันถูกซื้อในหลายคำสั่งซื้อได้ แต่ **คู่ของ (order_id, product_id)** นั้นไม่ควรซ้ำกัน (สมมติว่าสินค้าชนิดเดียวกันจะถูกรวมเป็นแถวเดียวโดยเพิ่ม quantity แทนการแยกแถว)

Composite Primary Key เขียนได้เฉพาะแบบ **table-level** เท่านั้น (เพราะเกี่ยวข้องกับหลายคอลัมน์พร้อมกัน จะเขียนแบบ column-level ต่อท้ายคอลัมน์เดียวไม่ได้):

```sql
DROP TABLE IF EXISTS order_items CASCADE;

CREATE TABLE order_items (
    order_id      INTEGER,
    product_id    INTEGER,
    quantity      INTEGER NOT NULL CHECK (quantity > 0),
    unit_price    NUMERIC(10, 2) NOT NULL,
    CONSTRAINT pk_order_items PRIMARY KEY (order_id, product_id)
);
```

เติมข้อมูลตัวอย่าง:

```sql
INSERT INTO orders (order_id, customer_id, status) VALUES
    (1001, 1, 'completed'),
    (1002, 2, 'pending');

INSERT INTO order_items (order_id, product_id, quantity, unit_price) VALUES
    (1001, 1, 2, 55.00),   -- คำสั่งซื้อ 1001: Espresso 2 แก้ว
    (1001, 2, 1, 65.00),   -- คำสั่งซื้อ 1001: Latte 1 แก้ว
    (1002, 3, 3, 65.00);   -- คำสั่งซื้อ 1002: Cappuccino 3 แก้ว
```

ทดสอบว่า composite key ทำงานอย่างไร:

```sql
-- ใช้ order_id ซ้ำได้ ถ้า product_id ไม่ซ้ำ (เพราะ PK คือคู่ order_id+product_id)
INSERT INTO order_items (order_id, product_id, quantity, unit_price)
VALUES (1001, 4, 1, 70.00);   -- สำเร็จ: order_id=1001 ซ้ำ แต่ product_id=4 ไม่ซ้ำ

-- แต่ถ้าคู่ (order_id, product_id) ซ้ำเดิมทุกตัว จะ error ทันที
INSERT INTO order_items (order_id, product_id, quantity, unit_price)
VALUES (1001, 1, 5, 55.00);
```

```
ERROR:  duplicate key value violates unique constraint "pk_order_items"
DETAIL:  Key (order_id, product_id)=(1001, 1) already exists.
```

สังเกตว่า error message แสดงคู่ค่าทั้งสองคอลัมน์รวมกัน `(order_id, product_id)=(1001, 1)` นี่คือหลักการสำคัญของ Composite Key: **การตรวจสอบความซ้ำจะพิจารณาค่าทุกคอลัมน์ที่ประกอบกันเป็นชุด** ไม่ใช่พิจารณาทีละคอลัมน์แยกกัน

ตรวจสอบ Primary Key ของตารางผ่าน catalog view `information_schema` ได้ดังนี้:

```sql
SELECT tc.constraint_name, tc.table_name, kcu.column_name, kcu.ordinal_position
FROM information_schema.table_constraints tc
JOIN information_schema.key_column_usage kcu
    ON tc.constraint_name = kcu.constraint_name
   AND tc.table_schema = kcu.table_schema
WHERE tc.constraint_type = 'PRIMARY KEY'
  AND tc.table_name = 'order_items';
```

```
 constraint_name |  table_name  | column_name | ordinal_position
------------------+--------------+-------------+-------------------
 pk_order_items   | order_items  | order_id    |                 1
 pk_order_items   | order_items  | product_id  |                 2
(2 rows)
```

คอลัมน์ `ordinal_position` บอกลำดับของคอลัมน์ในชุด composite key ซึ่งมีผลต่อประสิทธิภาพของ index ที่ถูกสร้างขึ้นมาด้วย (จะกล่าวถึงใน Step 157 และลงลึกใน Part 041)

> **แนวคิดสำคัญ:** ตารางที่ทำหน้าที่เชื่อมความสัมพันธ์แบบ many-to-many ระหว่างสองตาราง (เรียกว่า **junction table** หรือ **associative table**) มักใช้ composite primary key ที่ประกอบด้วย foreign key ของทั้งสองฝั่งเสมอ เช่น `order_items` เชื่อม `orders` กับ `products`

---

## Step 154: การเพิ่ม/ลบ Primary Key ด้วย ALTER TABLE ในตารางที่มีอยู่แล้ว

ในชีวิตจริง เรามักต้องเพิ่มหรือลบ Primary Key จากตารางที่มีข้อมูลอยู่แล้ว โดยไม่ต้องการ `DROP TABLE` แล้วสร้างใหม่ ซึ่งทำได้ด้วยคำสั่ง `ALTER TABLE`

### 154.1 เพิ่ม Primary Key ให้ตารางที่ยังไม่มี

สมมติว่ามีตาราง `promotions` ที่สร้างไว้โดยยังไม่มี PK:

```sql
CREATE TABLE promotions (
    promo_code  VARCHAR(20),
    description TEXT,
    discount_percent NUMERIC(5, 2)
);

INSERT INTO promotions (promo_code, description, discount_percent) VALUES
    ('WELCOME10', 'ส่วนลดสมาชิกใหม่', 10.00),
    ('SUMMER20',  'โปรโมชั่นหน้าร้อน', 20.00);
```

เพิ่ม Primary Key ภายหลังด้วย `ALTER TABLE ... ADD CONSTRAINT`:

```sql
ALTER TABLE promotions
    ADD CONSTRAINT pk_promotions PRIMARY KEY (promo_code);
```

```
ALTER TABLE
```

PostgreSQL จะตรวจสอบข้อมูลที่มีอยู่เดิมทั้งหมดก่อนว่าผ่านเงื่อนไข (ไม่ซ้ำ, ไม่เป็น NULL) หรือไม่ ถ้าข้อมูลเดิมมีปัญหา คำสั่งจะถูกปฏิเสธ:

```sql
CREATE TABLE bad_example (
    code VARCHAR(10),
    name TEXT
);

INSERT INTO bad_example (code, name) VALUES ('A1', 'Item A'), ('A1', 'Item B');

ALTER TABLE bad_example ADD CONSTRAINT pk_bad_example PRIMARY KEY (code);
```

```
ERROR:  could not create unique index "pk_bad_example"
DETAIL:  Key (code)=(A1) is duplicated.
```

ในกรณีนี้ต้องแก้ไขข้อมูลให้ไม่ซ้ำก่อน (เช่น `UPDATE` หรือ `DELETE` แถวที่ซ้ำ) จึงจะเพิ่ม PK สำเร็จ

```sql
DROP TABLE bad_example;
```

### 154.2 ลบ Primary Key

ใช้คำสั่ง `ALTER TABLE ... DROP CONSTRAINT` โดยระบุชื่อ constraint:

```sql
ALTER TABLE promotions DROP CONSTRAINT pk_promotions;
```

```
ALTER TABLE
```

หากจำชื่อ constraint ไม่ได้ ให้ค้นหาด้วยคำสั่ง `\d` ใน psql หรือ query จาก `information_schema`:

```sql
\d promotions
```

```
                     Table "public.promotions"
      Column      |     Type      | Collation | Nullable | Default
-------------------+---------------+-----------+----------+---------
 promo_code        | character varying(20) |           |          |
 description       | text          |           |          |
 discount_percent  | numeric(5,2)  |           |          |
```

หรือ query จาก catalog:

```sql
SELECT conname, contype
FROM pg_constraint
WHERE conrelid = 'promotions'::regclass;
```

### 154.3 เปลี่ยนคอลัมน์ของ Primary Key

PostgreSQL ไม่มีคำสั่ง "แก้ไข" Primary Key โดยตรง หากต้องการเปลี่ยนคอลัมน์ที่เป็น PK ต้อง `DROP` แล้ว `ADD` ใหม่:

```sql
ALTER TABLE promotions ADD CONSTRAINT pk_promotions PRIMARY KEY (promo_code);

-- สมมติภายหลังต้องการเปลี่ยนไปใช้คอลัมน์ใหม่เป็น PK
ALTER TABLE promotions ADD COLUMN promo_id INTEGER;
UPDATE promotions SET promo_id = 1 WHERE promo_code = 'WELCOME10';
UPDATE promotions SET promo_id = 2 WHERE promo_code = 'SUMMER20';

ALTER TABLE promotions DROP CONSTRAINT pk_promotions;
ALTER TABLE promotions ALTER COLUMN promo_id SET NOT NULL;
ALTER TABLE promotions ADD CONSTRAINT pk_promotions PRIMARY KEY (promo_id);
```

> **ข้อควรระวัง:** ในตารางที่มีข้อมูลจำนวนมากในระบบ production การ `DROP` แล้ว `ADD CONSTRAINT PRIMARY KEY` ใหม่จะ **ล็อกตาราง (table lock)** และสร้าง index ใหม่ทั้งหมด ซึ่งอาจใช้เวลานานและกระทบผู้ใช้งานจริง ควรวางแผนทำในช่วงเวลาที่มีผู้ใช้น้อย หรือศึกษาเทคนิคการสร้าง unique index แบบ `CONCURRENTLY` ก่อนแล้วค่อยแนบเป็น constraint ภายหลัง (จะกล่าวถึงรายละเอียดใน Part 041 เรื่อง Index)

---

## Step 155: Unique Constraint — ความแตกต่างจาก Primary Key

**Unique Constraint** ทำหน้าที่คล้าย Primary Key ตรงที่บังคับว่าค่าของคอลัมน์ (หรือกลุ่มคอลัมน์) ต้อง **ไม่ซ้ำกัน** แต่มีข้อแตกต่างสำคัญ:

| คุณสมบัติ | Primary Key | Unique Constraint |
|---|---|---|
| จำนวนต่อตาราง | มีได้ **เพียง 1 ตัว** เท่านั้น | มีได้ **หลายตัว** |
| อนุญาตค่า NULL | **ไม่ได้** (บังคับ NOT NULL เสมอ) | **ได้** (ค่าเดิมมาตรฐานถือว่า NULL แต่ละตัวไม่ซ้ำกัน — ดู Step 156) |
| ความหมายเชิงออกแบบ | ระบุเอกลักษณ์หลักของแถว มักถูกอ้างอิงจาก Foreign Key | ระบุว่าคอลัมน์นี้ต้อง "ไม่ซ้ำกัน" แต่ไม่ใช่ตัวระบุหลักของแถว |
| Index ที่สร้างอัตโนมัติ | Unique B-tree index | Unique B-tree index |

ตัวอย่างการใช้งาน: ตาราง `customers` ของเรามี `customer_id` เป็น Primary Key อยู่แล้ว แต่ในความเป็นจริง คอลัมน์ `email` และ `member_code` ก็ควรไม่ซ้ำกันเช่นกัน (ลูกค้าแต่ละคนควรมีอีเมลและรหัสสมาชิกที่ไม่ซ้ำใคร) — นี่คือกรณีที่เหมาะกับ Unique Constraint

```sql
ALTER TABLE customers
    ADD CONSTRAINT uq_customers_email ADD CONSTRAINT uq_customers_email UNIQUE (email);
```

> เขียนผิดโดยตั้งใจด้านบนเพื่อแสดง error ที่พบบ่อย — ลองรันดูจะได้ syntax error สอนให้สังเกต ไวยากรณ์ที่ถูกต้องคือ:

```sql
ALTER TABLE customers
    ADD CONSTRAINT uq_customers_email UNIQUE (email);

ALTER TABLE customers
    ADD CONSTRAINT uq_customers_member_code UNIQUE (member_code);
```

```
ALTER TABLE
ALTER TABLE
```

หรือกำหนดตอนสร้างตารางเลยก็ได้ ทั้งแบบ column-level และ table-level เหมือน Primary Key:

```sql
-- ตัวอย่าง column-level (ถ้าสร้างตารางใหม่)
CREATE TABLE example_customers (
    customer_id INTEGER PRIMARY KEY,
    email       VARCHAR(255) UNIQUE,
    member_code VARCHAR(20) UNIQUE
);

DROP TABLE example_customers;
```

ทดสอบว่า Unique Constraint ยอมให้ค่า NULL ได้จริง ในขณะที่ยังคงห้ามค่าซ้ำ:

```sql
-- ลูกค้าใหม่ที่ยังไม่ได้ให้อีเมล
INSERT INTO customers (customer_id, full_name, email, phone, member_code)
VALUES (3, 'วิชัย มั่งมี', NULL, '0833333333', 'MEM003');

-- ลูกค้าอีกคนที่ยังไม่ได้ให้อีเมลเช่นกัน — สำเร็จ! เพราะ NULL ไม่ถือว่าซ้ำกับ NULL
INSERT INTO customers (customer_id, full_name, email, phone, member_code)
VALUES (4, 'มานี ดีใจ', NULL, '0844444444', 'MEM004');

SELECT customer_id, full_name, email FROM customers WHERE email IS NULL;
```

```
 customer_id |  full_name  | email
-------------+-------------+--------
           3 | วิชัย มั่งมี  |
           4 | มานี ดีใจ    |
(2 rows)
```

แต่ถ้าใส่อีเมลซ้ำกับที่มีอยู่แล้ว จะถูกปฏิเสธทันที:

```sql
INSERT INTO customers (customer_id, full_name, email, phone, member_code)
VALUES (5, 'สมปอง มีทรัพย์', 'somchai@example.com', '0855555555', 'MEM005');
```

```
ERROR:  duplicate key value violates unique constraint "uq_customers_email"
DETAIL:  Key (email)=(somchai@example.com) already exists.
```

พฤติกรรมนี้เป็นไปตามมาตรฐาน SQL ที่ถือว่า `NULL` แต่ละตัว **ไม่เท่ากับ NULL ตัวอื่น** (`NULL <> NULL` ในทางตรรกะเป็น unknown เสมอ) ดังนั้น Unique Constraint แบบมาตรฐานจึงมองว่าค่า NULL หลายตัวไม่ถือว่า "ซ้ำกัน"

---

## Step 156: Composite Unique Constraint และ UNIQUE NULLS NOT DISTINCT (PostgreSQL 15+)

### 156.1 Composite Unique Constraint

เช่นเดียวกับ Composite Primary Key เราสามารถกำหนด Unique Constraint ให้ครอบคลุมหลายคอลัมน์พร้อมกันได้ โดยเงื่อนไขความซ้ำจะพิจารณา **ทุกคอลัมน์รวมกันเป็นชุด**

ตัวอย่าง: สมมติร้านค้าออนไลน์ต้องการให้สินค้าแต่ละชิ้นมี SKU ที่ไม่ซ้ำกัน **ภายในแต่ละหมวดหมู่** (แต่สินค้าต่างหมวดหมู่มี SKU ซ้ำกันได้ เพราะระบบเก่าเคยตั้งรหัสแบบนี้ไว้) เราสามารถเพิ่มคอลัมน์และกำหนด composite unique ได้ดังนี้:

```sql
ALTER TABLE products ADD COLUMN sku VARCHAR(20);

UPDATE products SET sku = 'SKU-001' WHERE product_id = 1;
UPDATE products SET sku = 'SKU-002' WHERE product_id = 2;
UPDATE products SET sku = 'SKU-003' WHERE product_id = 3;
UPDATE products SET sku = 'SKU-001' WHERE product_id = 4;  -- ซ้ำกับ product_id=1 แต่คนละหมวดหมู่
UPDATE products SET sku = 'SKU-004' WHERE product_id = 5;

ALTER TABLE products
    ADD CONSTRAINT uq_products_category_sku UNIQUE (category_id, sku);
```

```
ALTER TABLE
```

ทดสอบ: แทรก SKU ซ้ำในหมวดหมู่เดียวกันจะถูกปฏิเสธ แต่ซ้ำกันข้ามหมวดหมู่ทำได้:

```sql
-- ซ้ำ (category_id=1, sku='SKU-001') กับ product_id=1 -> ผิดพลาด
INSERT INTO products (product_id, product_name, category_id, price, sku)
VALUES (6, 'Americano', 1, 55.00, 'SKU-001');
```

```
ERROR:  duplicate key value violates unique constraint "uq_products_category_sku"
DETAIL:  Key (category_id, sku)=(1, SKU-001) already exists.
```

```sql
-- category_id=2, sku='SKU-001' -> ไม่ซ้ำกับใคร (คนละ category) จึงสำเร็จ
INSERT INTO products (product_id, product_name, category_id, price, sku)
VALUES (7, 'Chocolate', 2, 75.00, 'SKU-001');
```

```
INSERT 0 1
```

### 156.2 ปัญหาของ NULL หลายตัวใน Composite Unique

ปัญหาที่พบบ่อยในทางปฏิบัติคือ เมื่อคอลัมน์ที่เป็นส่วนหนึ่งของ Unique Constraint มีค่า `NULL` ได้ ระบบจะอนุญาตให้มีแถวที่ค่าอื่นเหมือนกันทุกประการ ตราบใดที่มีอย่างน้อยหนึ่งคอลัมน์เป็น `NULL` เพราะ `NULL` ไม่ถือว่าเท่ากับ `NULL`

```sql
-- ทั้งสองแถวมี category_id=3 เหมือนกัน แต่ sku เป็น NULL ทั้งคู่ -> ไม่ถือว่าซ้ำ
INSERT INTO products (product_id, product_name, category_id, price, sku)
VALUES (8, 'Muffin', 3, 40.00, NULL);

INSERT INTO products (product_id, product_name, category_id, price, sku)
VALUES (9, 'Bagel', 3, 42.00, NULL);

SELECT product_id, product_name, category_id, sku FROM products WHERE sku IS NULL;
```

```
 product_id | product_name | category_id | sku
------------+--------------+-------------+------
          8 | Muffin       |           3 |
          9 | Bagel        |           3 |
(2 rows)
```

ในบางกรณีธุรกิจ พฤติกรรมนี้ **ไม่ใช่สิ่งที่ต้องการ** เช่น ถ้าเราต้องการให้ "รหัสประจำตัวประชาชนของลูกค้า" ไม่ซ้ำกัน แต่ยอมให้เป็น NULL ได้สำหรับลูกค้าต่างชาติที่ยังไม่ได้กรอกข้อมูล — แต่ก็ยังอยากให้มี "ลูกค้าที่ไม่มีเลขบัตรได้เพียงคนเดียวต่อชุดข้อมูลอื่น ๆ ที่ระบุ" เป็นต้น

### 156.3 UNIQUE NULLS NOT DISTINCT (PostgreSQL 15+)

ตั้งแต่ **PostgreSQL 15** เป็นต้นไป เราสามารถระบุ `NULLS NOT DISTINCT` เพื่อบอกว่า "ให้ถือว่า NULL แต่ละตัวเหมือนกัน (ซ้ำกัน)" ซึ่งจะพลิกพฤติกรรมมาตรฐานข้างต้น:

```sql
ALTER TABLE products DROP CONSTRAINT uq_products_category_sku;

ALTER TABLE products
    ADD CONSTRAINT uq_products_category_sku
    UNIQUE NULLS NOT DISTINCT (category_id, sku);
```

```
ALTER TABLE
ALTER TABLE
```

ลองแทรกแถวที่มี `sku IS NULL` ซ้ำ `category_id` เดิมอีกครั้ง:

```sql
-- ลบข้อมูลเดิมที่ค้างอยู่ก่อน เพื่อทดสอบ constraint ใหม่
DELETE FROM products WHERE product_id IN (8, 9);

INSERT INTO products (product_id, product_name, category_id, price, sku)
VALUES (8, 'Muffin', 3, 40.00, NULL);

-- คราวนี้จะถูกปฏิเสธ เพราะ NULLS NOT DISTINCT ถือว่า NULL ซ้ำกับ NULL
INSERT INTO products (product_id, product_name, category_id, price, sku)
VALUES (9, 'Bagel', 3, 42.00, NULL);
```

```
ERROR:  duplicate key value violates unique constraint "uq_products_category_sku"
DETAIL:  Key (category_id, sku)=(3, null) already exists.
```

> **ตารางเปรียบเทียบ:**
>
> | รูปแบบ | พฤติกรรมเมื่อมี NULL หลายตัว |
> |---|---|
> | `UNIQUE (col)` (ค่าเริ่มต้น) | NULL หลายตัวอยู่ร่วมกันได้ (ไม่ถือว่าซ้ำ) |
> | `UNIQUE NULLS DISTINCT (col)` | เหมือนค่าเริ่มต้น (เขียนชัดเจนเพื่อความเข้าใจ) |
> | `UNIQUE NULLS NOT DISTINCT (col)` | NULL ถือว่าซ้ำกัน อนุญาตให้มีแถวที่ col เป็น NULL ได้เพียงแถวเดียว |

> **หมายเหตุเรื่องเวอร์ชัน:** คีย์เวิร์ด `NULLS NOT DISTINCT` ใช้ได้เฉพาะ PostgreSQL **15 ขึ้นไป** เท่านั้น หากท่านใช้เวอร์ชันเก่ากว่านี้ ต้องจำลองพฤติกรรมด้วยวิธีอื่น เช่น สร้าง **partial unique index** (`CREATE UNIQUE INDEX ... WHERE col IS NOT NULL` ร่วมกับการควบคุมด้วย trigger หรือ unique expression index ที่แปลง NULL เป็นค่าคงที่) ซึ่งเป็นเทคนิคขั้นสูงที่จะกล่าวถึงใน Part 041-042 เรื่อง Index

---

## Step 157: Index ที่เกิดขึ้นอัตโนมัติจาก Primary Key/Unique Constraint

เมื่อเราสร้าง Primary Key หรือ Unique Constraint ให้กับตาราง PostgreSQL จะ **สร้าง Unique B-tree Index ให้อัตโนมัติ** เสมอ เพื่อใช้ตรวจสอบความซ้ำของข้อมูลอย่างรวดเร็ว (ถ้าไม่มี index การตรวจสอบความซ้ำทุกครั้งที่ INSERT จะต้อง scan ทั้งตาราง ซึ่งช้ามากเมื่อข้อมูลเยอะ)

ลองดู index ที่ถูกสร้างขึ้นโดยอัตโนมัติในตารางที่เราทำมาทั้งหมด:

```sql
SELECT indexname, indexdef
FROM pg_indexes
WHERE tablename = 'customers';
```

```
          indexname         |                                     indexdef
-----------------------------+-------------------------------------------------------------------------------------
 customers_pkey              | CREATE UNIQUE INDEX customers_pkey ON public.customers USING btree (customer_id)
 uq_customers_email          | CREATE UNIQUE INDEX uq_customers_email ON public.customers USING btree (email)
 uq_customers_member_code    | CREATE UNIQUE INDEX uq_customers_member_code ON public.customers USING btree (member_code)
(3 rows)
```

สังเกตว่า:

- ชื่อ index ของ Primary Key จะตรงกับชื่อ constraint (`customers_pkey` เพราะเราไม่ได้ตั้งชื่อ constraint เอง จึงใช้ชื่อ default)
- ชื่อ index ของ Unique Constraint ก็ตรงกับชื่อ constraint ที่เราตั้งเอง (`uq_customers_email`, `uq_customers_member_code`)
- ทุก index เป็นชนิด `UNIQUE` และใช้โครงสร้าง `btree` (B-tree) ซึ่งเป็น index ชนิดมาตรฐานที่เหมาะกับการค้นหาแบบเท่ากับ (`=`) และการเรียงลำดับ

ตรวจสอบผ่าน `\d` ใน psql ก็จะเห็นข้อมูลชุดเดียวกันในรูปแบบที่อ่านง่ายกว่า:

```sql
\d customers
```

```
                                     Table "public.customers"
    Column     |            Type             | Collation | Nullable |      Default
----------------+------------------------------+-----------+----------+--------------------
 customer_id    | integer                      |           | not null |
 full_name      | character varying(100)       |           | not null |
 email          | character varying(255)       |           |          |
 phone          | character varying(20)        |           |          |
 member_code    | character varying(20)        |           |          |
 created_at     | timestamp without time zone  |           |          | now()
Indexes:
    "customers_pkey" PRIMARY KEY, btree (customer_id)
    "uq_customers_email" UNIQUE CONSTRAINT, btree (email)
    "uq_customers_member_code" UNIQUE CONSTRAINT, btree (member_code)
```

ประโยชน์เชิงประสิทธิภาพของ index ที่เกิดขึ้นอัตโนมัตินี้ไม่ได้มีแค่การป้องกันข้อมูลซ้ำเท่านั้น แต่ยังทำให้การค้นหาข้อมูลด้วยเงื่อนไข `WHERE customer_id = ...` หรือ `WHERE email = ...` **เร็วขึ้นมาก** เพราะ PostgreSQL ใช้ index scan แทนการ sequential scan ทั้งตาราง:

```sql
EXPLAIN SELECT * FROM customers WHERE customer_id = 1;
```

```
                                    QUERY PLAN
------------------------------------------------------------------------------------
 Index Scan using customers_pkey on customers  (cost=0.15..8.17 rows=1 width=...)
   Index Cond: (customer_id = 1)
```

> **ความเชื่อมโยงกับ Part 041:** เนื้อหาเรื่อง Index อย่างละเอียด (B-tree, Hash, GIN, GiST, partial index, expression index, multi-column index ordering ฯลฯ) จะถูกอธิบายลึกใน **Part 041 — Index พื้นฐาน** และต่อเนื่องไปจนถึง Part 042-043 สำหรับตอนนี้ให้จำหลักสำคัญไว้ก่อนว่า:
>
> 1. Primary Key และ Unique Constraint แต่ละตัว **สร้าง index แยกกันคนละตัว** เสมอ (ไม่ได้ใช้ index ร่วมกัน)
> 2. การมี index มากเกินจำเป็นจะทำให้ `INSERT`/`UPDATE`/`DELETE` ช้าลง เพราะทุก index ต้องถูกอัปเดตพร้อมกัน — จึงไม่ควรใส่ Unique Constraint พร่ำเพรื่อโดยไม่มีเหตุผลทางธุรกิจรองรับ
> 3. ห้าม `DROP` index ของ Primary Key/Unique Constraint โดยตรงด้วย `DROP INDEX` เพราะ index เหล่านี้ผูกกับ constraint — ต้อง `DROP CONSTRAINT` แทน (ลอง `DROP INDEX customers_pkey;` จะได้ error แจ้งให้ไปลบที่ constraint)

---

## Step 158: ข้อผิดพลาดที่พบบ่อย — duplicate key value violates unique constraint

Error ที่พบบ่อยที่สุดเมื่อทำงานกับ Primary Key/Unique Constraint คือ:

```
ERROR:  duplicate key value violates unique constraint "constraint_name"
DETAIL:  Key (column)=(value) already exists.
```

มาดูสาเหตุที่พบบ่อยและวิธีวินิจฉัยทีละกรณี

### 158.1 สาเหตุที่ 1 — พยายามแทรกค่าที่มีอยู่แล้วจริง ๆ

```sql
INSERT INTO categories (category_id, category_name) VALUES (1, 'Snack Bar');
```

```
ERROR:  duplicate key value violates unique constraint "categories_pkey"
DETAIL:  Key (category_id)=(1) already exists.
```

**วิธีวินิจฉัย:** อ่านชื่อ constraint จาก error (`categories_pkey`) แล้วตรวจสอบว่ามีแถวที่มีค่านั้นอยู่แล้วหรือไม่:

```sql
SELECT * FROM categories WHERE category_id = 1;
```

**วิธีแก้:** ถ้าต้องการ "แทรกถ้ายังไม่มี หรืออัปเดตถ้ามีอยู่แล้ว" ให้ใช้ `INSERT ... ON CONFLICT` (จะเรียนละเอียดใน Part เรื่อง Upsert ภายหลัง) เช่น:

```sql
INSERT INTO categories (category_id, category_name, description)
VALUES (1, 'Coffee (Updated)', 'ปรับปรุงคำอธิบาย')
ON CONFLICT (category_id)
DO UPDATE SET category_name = EXCLUDED.category_name,
              description   = EXCLUDED.description;
```

```
INSERT 0 1
```

### 158.2 สาเหตุที่ 2 — Sequence/Identity ไม่ sync กับข้อมูลที่มีอยู่

กรณีนี้พบบ่อยมากเมื่อมีการ `INSERT` ข้อมูลโดยระบุค่า Primary Key ตรง ๆ (เช่น import ข้อมูลจากระบบเก่า) ปนกับการใช้ auto-increment (`GENERATED ALWAYS AS IDENTITY`) ทำให้ sequence ภายในไม่รู้ว่าค่าสูงสุดถูกใช้ไปแล้วเท่าไร:

```sql
CREATE TABLE tickets (
    ticket_id  INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    subject    TEXT NOT NULL
);

-- import ข้อมูลเก่าโดยระบุ ticket_id ตรง ๆ
INSERT INTO tickets OVERRIDING SYSTEM VALUE VALUES (1, 'ปัญหาเครื่องชงกาแฟ');
INSERT INTO tickets OVERRIDING SYSTEM VALUE VALUES (2, 'สอบถามโปรโมชั่น');

-- ต่อมาแทรกข้อมูลใหม่แบบปกติ โดยให้ระบบ gen เลขให้เอง
INSERT INTO tickets (subject) VALUES ('ขอใบเสร็จ');
```

```
ERROR:  duplicate key value violates unique constraint "tickets_pkey"
DETAIL:  Key (ticket_id)=(1) already exists.
```

**สาเหตุ:** sequence ภายในของ `ticket_id` ยังนับจาก 1 อยู่ (ไม่รู้ว่าเรา insert ค่า 1, 2 ไปแล้วด้วยมือผ่าน `OVERRIDING SYSTEM VALUE`) จึงพยายามสร้างค่า 1 ซ้ำ

**วิธีแก้:** ปรับ sequence ให้ตรงกับค่าสูงสุดที่มีอยู่จริงด้วย `setval()`:

```sql
SELECT setval(
    pg_get_serial_sequence('tickets', 'ticket_id'),
    (SELECT MAX(ticket_id) FROM tickets)
);

INSERT INTO tickets (subject) VALUES ('ขอใบเสร็จ');
```

```
INSERT 0 1
```

### 158.3 สาเหตุที่ 3 — Race Condition จากการทำงานพร้อมกันหลาย connection

ในระบบจริงที่มีผู้ใช้พร้อมกันจำนวนมาก อาจเกิดกรณีที่สอง transaction ตรวจสอบว่า "ค่านี้ยังไม่มี" พร้อมกัน แล้วทั้งคู่พยายาม `INSERT` ค่าเดียวกันในเวลาไล่เลี่ยกัน — ฝั่งที่ commit ก่อนจะสำเร็จ ส่วนอีกฝั่งจะได้รับ error `duplicate key value` แม้ตอนตรวจสอบตอนแรกจะไม่พบข้อมูลซ้ำก็ตาม

**วิธีแก้ที่ถูกต้อง:** อย่าพึ่งพาการ `SELECT` ตรวจสอบก่อนแล้วค่อย `INSERT` (check-then-act) เพราะมีช่องว่างของเวลาให้เกิด race condition เสมอ ให้ใช้ constraint เป็นตัวป้องกันจริง แล้วจัดการ error ที่แอปพลิเคชัน หรือใช้ `INSERT ... ON CONFLICT DO NOTHING`:

```sql
INSERT INTO categories (category_id, category_name)
VALUES (99, 'Seasonal')
ON CONFLICT (category_id) DO NOTHING;
```

```
INSERT 0 0
```

ผลลัพธ์ `INSERT 0 0` หมายถึงไม่มีแถวถูกเพิ่ม (เพราะชนกับ conflict) แต่คำสั่งไม่ error — เหมาะสำหรับกรณีที่เราต้องการแค่ "ให้แน่ใจว่ามีข้อมูลนี้อยู่" โดยไม่สนใจว่าจะเป็นการเพิ่มใหม่หรือของเดิม

> **สรุปการวินิจฉัย:** เมื่อเจอ error `duplicate key value violates unique constraint` ให้ทำตามลำดับ (1) อ่านชื่อ constraint และคอลัมน์ที่ระบุใน `DETAIL` (2) `SELECT` ตรวจสอบว่ามีข้อมูลนั้นอยู่จริงหรือไม่ (3) ถ้ามีอยู่จริงและตั้งใจจะแทรกใหม่ ให้พิจารณาใช้ `ON CONFLICT` (4) ถ้าไม่มีข้อมูลนั้นอยู่เลยแต่ error ยังขึ้น ให้สงสัยปัญหา sequence ไม่ sync ตามข้อ 158.2

---

## Step 159: การเลือก Primary Key ที่เหมาะสม — Integer/Bigint Identity vs UUID

หนึ่งในการตัดสินใจสำคัญที่สุดตอนออกแบบตาราง คือจะใช้ชนิดข้อมูลใดเป็น Primary Key ตัวเลือกหลักที่นิยมใช้กันคือ **Integer/Bigint แบบ Auto-increment (Identity)** และ **UUID**

### 159.1 Integer/Bigint Identity

```sql
CREATE TABLE reviews (
    review_id  INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    product_id INTEGER NOT NULL,
    rating     SMALLINT NOT NULL CHECK (rating BETWEEN 1 AND 5),
    comment    TEXT
);

INSERT INTO reviews (product_id, rating, comment) VALUES
    (1, 5, 'กาแฟหอมมาก'),
    (2, 4, 'รสชาติดี แต่หวานไปนิด');

SELECT * FROM reviews;
```

```
 review_id | product_id | rating |        comment
-----------+------------+--------+-------------------------
         1 |          1 |      5 | กาแฟหอมมาก
         2 |          2 |      4 | รสชาติดี แต่หวานไปนิด
(2 rows)
```

**ข้อดี:**

- ขนาดเล็ก (`INTEGER` 4 bytes, `BIGINT` 8 bytes) ประหยัดพื้นที่จัดเก็บและหน่วยความจำสำหรับ index
- เรียงลำดับตามลำดับการสร้าง (sequential) ทำให้ B-tree index เขียนข้อมูลใหม่ต่อท้ายได้อย่างมีประสิทธิภาพ ลดปัญหา index fragmentation
- อ่านง่าย จำง่าย เหมาะกับการ debug หรือใช้ใน URL (เช่น `/products/42`)
- เปรียบเทียบและ join เร็วกว่า UUID เพราะเป็นตัวเลขล้วน

**ข้อเสีย:**

- ค่าที่ต่อเนื่องกันทำให้ **เดาได้ง่าย** (enumerable) — หากเป็น public API อาจถูกเดา ID เพื่อ scrape ข้อมูลของผู้อื่นได้ เช่น เดาว่ามี `/orders/1001`, `/orders/1002` ต่อกันไปเรื่อย ๆ
- หากต้องรวมข้อมูลจากหลายฐานข้อมูล (เช่น สาขาต่าง ๆ ที่แต่ละสาขามีฐานข้อมูลแยกกัน) ค่า ID จะชนกันได้ ต้องมีกลไกจัดการเพิ่มเติม
- `INTEGER` มีขีดจำกัดที่ 2,147,483,647 — ตารางที่มีการเติบโตสูงมากควรพิจารณาใช้ `BIGINT` ตั้งแต่แรก เพื่อเลี่ยงปัญหาการ migrate ภายหลัง

### 159.2 UUID

```sql
CREATE TABLE sessions (
    session_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id INTEGER NOT NULL,
    started_at  TIMESTAMP DEFAULT now()
);

INSERT INTO sessions (customer_id) VALUES (1), (2);

SELECT * FROM sessions;
```

```
               session_id              | customer_id |         started_at
--------------------------------------+-------------+----------------------------
 3f2a9c1e-6b7d-4a12-9e3f-1d2c8a5b7e90  |           1 | 2026-09-25 10:20:00.111111
 8a71b0d4-52c9-4e88-bc4a-9f0e6d1a2b33  |           2 | 2026-09-25 10:20:00.222222
(2 rows)
```

ฟังก์ชัน `gen_random_uuid()` เป็นฟังก์ชันในตัวของ PostgreSQL (ตั้งแต่เวอร์ชัน 13 ขึ้นไป ไม่ต้องติดตั้ง extension เพิ่มเติม) สร้างค่า UUID เวอร์ชัน 4 (สุ่มล้วน) ให้อัตโนมัติทุกครั้งที่ไม่ระบุค่าเอง

**ข้อดี:**

- **เดาไม่ได้** เหมาะกับ public-facing ID เช่น session token, API resource ID ที่ไม่ต้องการให้ผู้ใช้เดาค่าอื่นได้
- สร้างค่าได้จากฝั่งไคลเอนต์หรือหลายระบบพร้อมกันโดย **ไม่ชนกัน** แม้ไม่ได้เชื่อมต่อฐานข้อมูลส่วนกลาง เหมาะกับสถาปัตยกรรมแบบ distributed หรือระบบที่ต้อง merge ข้อมูลจากหลายแหล่ง
- ค่าคงที่ในเชิงความหมายไม่ขึ้นกับลำดับการสร้าง จึงไม่เปิดเผยข้อมูลเชิงธุรกิจ (เช่น จำนวนคำสั่งซื้อทั้งหมดของร้าน)

**ข้อเสีย:**

- ขนาดใหญ่กว่ามาก (16 bytes เทียบกับ 4-8 bytes ของ integer/bigint) ทำให้ index ใหญ่ขึ้น ใช้ I/O และหน่วยความจำมากขึ้น โดยเฉพาะเมื่อเป็น Foreign Key ที่ถูกอ้างอิงจากหลายตาราง
- UUID v4 เป็นค่าสุ่มล้วน **ไม่เรียงลำดับ** ทำให้การแทรกค่าใหม่กระจายไปทั่ว B-tree index (ไม่ใช่ต่อท้ายเหมือน integer) ส่งผลให้ index fragmentation สูงขึ้นและประสิทธิภาพการเขียนลดลงเมื่อข้อมูลมีจำนวนมาก
- อ่านยาก จำยาก ไม่เหมาะกับการ debug ด้วยตาเปล่า

> **ทางเลือกผสมผสาน — UUID v7:** ตั้งแต่มาตรฐาน UUID เวอร์ชัน 7 ถูกกำหนดขึ้น (ประกอบด้วยส่วน timestamp นำหน้า) ทำให้ UUID เรียงลำดับได้ตามเวลาสร้าง (time-ordered) ซึ่งแก้ปัญหา index fragmentation ของ UUID v4 ได้เป็นอย่างดี ในเวอร์ชัน PostgreSQL ที่รองรับ ฟังก์ชัน `uuidv7()` (มีให้ใช้งานเป็น built-in ตั้งแต่ PostgreSQL 18) จะเป็นทางเลือกที่คุ้มค่าในการพิจารณา หากใช้ PostgreSQL 16/17 ยังต้องพึ่งพา extension ภายนอกหรือ library ระดับแอปพลิเคชันเพื่อสร้าง UUID v7 เอง

### 159.3 ตารางสรุปการเลือกใช้

| สถานการณ์ | คำแนะนำ |
|---|---|
| ตารางภายใน ไม่ expose ID ให้ผู้ใช้เห็นโดยตรง ต้องการประสิทธิภาพสูงสุด | `BIGINT GENERATED ALWAYS AS IDENTITY` |
| API/Resource ID ที่ผู้ใช้ภายนอกเห็นได้ ไม่ต้องการให้เดาค่าได้ | `UUID` |
| ระบบ distributed ที่หลายโหนดต้องสร้าง ID เองโดยไม่ต้องประสานงานกัน | `UUID` (หรือ `UUID v7` ถ้ารองรับ) |
| ตารางเชื่อมความสัมพันธ์ (junction table) | มักใช้ **Composite Primary Key** จาก Foreign Key ของสองฝั่งอยู่แล้ว ไม่จำเป็นต้องมี ID เดี่ยวเพิ่ม |
| ต้องการทั้งความปลอดภัย (เดาไม่ได้) และประสิทธิภาพการ join ภายใน | ใช้ `BIGINT identity` เป็น PK ภายใน + เพิ่มคอลัมน์ `UUID UNIQUE` แยกต่างหากสำหรับ expose ออก public |

แนวทางสุดท้ายในตารางด้านบนเป็นที่นิยมมากในระบบระดับ production เพราะได้ทั้งสองข้อดี: ใช้ `BIGINT` เป็น PK สำหรับ join ภายในฐานข้อมูลที่รวดเร็ว และใช้ `UUID` แยกต่างหากเป็น public identifier ที่เดาไม่ได้ ตัวอย่าง:

```sql
CREATE TABLE api_resources (
    id          BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    public_id   UUID NOT NULL DEFAULT gen_random_uuid() UNIQUE,
    name        TEXT NOT NULL
);
```

---

## Step 160: แบบฝึกหัดรวม — ออกแบบ Constraint ที่เหมาะสม

มาถึงขั้นตอนสุดท้ายของ Part นี้ ให้เราลองออกแบบ constraint ให้ครบถ้วนสำหรับระบบจริงสองระบบ คือ **ร้านกาแฟ** (ที่ใช้อยู่ตลอด Part นี้) และ **ร้านค้าออนไลน์** (e-commerce) ขนาดเล็ก เพื่อฝึกการตัดสินใจแบบองค์รวม

### 160.1 ออกแบบให้ระบบร้านกาแฟสมบูรณ์

รวบรวม constraint ทั้งหมดที่เราสร้างไว้ตลอด Part นี้ให้อยู่ในภาพเดียว (DDL สรุปสุดท้าย):

```sql
-- categories: PK เดี่ยว
--   category_id INTEGER PRIMARY KEY

-- products: PK เดี่ยว + composite unique (category_id, sku)
--   product_id INTEGER PRIMARY KEY
--   UNIQUE NULLS NOT DISTINCT (category_id, sku)

-- customers: PK เดี่ยว + unique email + unique member_code
--   customer_id INTEGER PRIMARY KEY
--   UNIQUE (email)
--   UNIQUE (member_code)

-- orders: PK เดี่ยว
--   order_id INTEGER PRIMARY KEY

-- order_items: composite PK (order_id, product_id)
--   PRIMARY KEY (order_id, product_id)
```

ตรวจสอบภาพรวม constraint ทั้งหมดในฐานข้อมูลด้วย query เดียว:

```sql
SELECT
    tc.table_name,
    tc.constraint_name,
    tc.constraint_type,
    string_agg(kcu.column_name, ', ' ORDER BY kcu.ordinal_position) AS columns
FROM information_schema.table_constraints tc
JOIN information_schema.key_column_usage kcu
    ON tc.constraint_name = kcu.constraint_name
   AND tc.table_schema = kcu.table_schema
WHERE tc.constraint_type IN ('PRIMARY KEY', 'UNIQUE')
  AND tc.table_schema = 'public'
GROUP BY tc.table_name, tc.constraint_name, tc.constraint_type
ORDER BY tc.table_name, tc.constraint_type;
```

```
 table_name  |       constraint_name       | constraint_type |     columns
--------------+------------------------------+------------------+-------------------
 categories   | categories_pkey             | PRIMARY KEY      | category_id
 customers    | customers_pkey              | PRIMARY KEY      | customer_id
 customers    | uq_customers_email          | UNIQUE           | email
 customers    | uq_customers_member_code    | UNIQUE           | member_code
 order_items  | pk_order_items              | PRIMARY KEY      | order_id, product_id
 orders       | orders_pkey                 | PRIMARY KEY      | order_id
 products     | products_pkey               | PRIMARY KEY      | product_id
 products     | uq_products_category_sku    | UNIQUE           | category_id, sku
(8 rows)
```

### 160.2 ออกแบบระบบร้านค้าออนไลน์ใหม่

ลองประยุกต์หลักการเดียวกันกับระบบร้านค้าออนไลน์ที่มีความซับซ้อนขึ้น ซึ่งต้องมี: ผู้ใช้ (users), ที่อยู่จัดส่ง (addresses), สินค้า (items), ตะกร้าสินค้า (cart_items), และ คูปองส่วนลด (coupons)

```sql
-- ผู้ใช้: ใช้ BIGINT identity เป็น PK ภายใน + UUID เป็น public identifier
CREATE TABLE users (
    user_id     BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    public_id   UUID NOT NULL DEFAULT gen_random_uuid(),
    username    VARCHAR(50) NOT NULL,
    email       VARCHAR(255) NOT NULL,
    national_id VARCHAR(13),
    CONSTRAINT uq_users_public_id UNIQUE (public_id),
    CONSTRAINT uq_users_username  UNIQUE (username),
    CONSTRAINT uq_users_email     UNIQUE (email),
    -- เลขบัตรประชาชนไม่บังคับกรอก แต่ถ้ามีต้องไม่ซ้ำ และถือว่า "ไม่มี" ทุกคนเหมือนกันได้ (NULL ซ้ำกันได้)
    CONSTRAINT uq_users_national_id UNIQUE NULLS DISTINCT (national_id)
);

-- ที่อยู่จัดส่ง: PK เดี่ยว, ผู้ใช้หนึ่งคนมีได้หลายที่อยู่ แต่ label ต้องไม่ซ้ำต่อผู้ใช้ (เช่น "บ้าน", "ที่ทำงาน")
CREATE TABLE addresses (
    address_id  BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    user_id     BIGINT NOT NULL,
    label       VARCHAR(30) NOT NULL,
    full_address TEXT NOT NULL,
    CONSTRAINT uq_addresses_user_label UNIQUE (user_id, label)
);

-- สินค้า: PK เดี่ยว + SKU ต้องไม่ซ้ำทั้งระบบ (ต่างจากร้านกาแฟที่ SKU ซ้ำข้ามหมวดได้)
CREATE TABLE items (
    item_id     BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    sku         VARCHAR(30) NOT NULL,
    item_name   VARCHAR(200) NOT NULL,
    price       NUMERIC(12, 2) NOT NULL,
    CONSTRAINT uq_items_sku UNIQUE (sku)
);

-- ตะกร้าสินค้า: composite PK (user_id, item_id) เพราะผู้ใช้แต่ละคนมีสินค้าแต่ละชิ้นในตะกร้าได้แค่ 1 แถว
CREATE TABLE cart_items (
    user_id     BIGINT NOT NULL,
    item_id     BIGINT NOT NULL,
    quantity    INTEGER NOT NULL CHECK (quantity > 0),
    CONSTRAINT pk_cart_items PRIMARY KEY (user_id, item_id)
);

-- คูปองส่วนลด: รหัสคูปองเป็น PK ตรง ๆ ได้ เพราะเป็น natural key ที่มีความหมายและไม่เปลี่ยนแปลง
CREATE TABLE coupons (
    coupon_code VARCHAR(20) PRIMARY KEY,
    discount_percent NUMERIC(5, 2) NOT NULL CHECK (discount_percent BETWEEN 0 AND 100),
    expires_at  TIMESTAMP
);
```

```
CREATE TABLE
CREATE TABLE
CREATE TABLE
CREATE TABLE
CREATE TABLE
```

**เหตุผลของการตัดสินใจในตัวอย่างนี้:**

1. `users.user_id` ใช้ `BIGINT identity` เพราะเป็น PK ภายในที่ถูก join บ่อยมาก (กับ addresses, cart_items ฯลฯ) ต้องการประสิทธิภาพสูงสุด ส่วน `public_id` เป็น UUID แยกไว้สำหรับ expose ใน API เพื่อไม่ให้เดาจำนวนผู้ใช้ทั้งหมดได้
2. `national_id` ใช้ `UNIQUE NULLS DISTINCT` (พฤติกรรมมาตรฐาน) อย่างชัดเจน เพราะยอมให้ผู้ใช้หลายคนยังไม่กรอกเลขบัตรประชาชนพร้อมกันได้
3. `addresses.address_id` มี PK เดี่ยวของตัวเอง (ไม่ใช้ composite) เพราะที่อยู่แต่ละรายการมีสิทธิ์ถูกอ้างอิงเดี่ยว ๆ จากที่อื่น (เช่น ผูกกับคำสั่งซื้อว่าจัดส่งไปที่อยู่ไหน) — composite unique `(user_id, label)` ทำหน้าที่ป้องกันความซ้ำของ label เท่านั้น
4. `cart_items` ใช้ composite PK ตรงตามธรรมชาติของข้อมูล: ผู้ใช้คนหนึ่งมีสินค้าชนิดหนึ่งอยู่ในตะกร้าได้เพียงแถวเดียวเท่านั้น (ถ้าเพิ่มซ้ำ ควรอัปเดต quantity แทน)
5. `coupons.coupon_code` ใช้ **natural key** (รหัสที่มีความหมายทางธุรกิจ เช่น `SUMMER20`) เป็น PK ตรง ๆ แทนที่จะสร้างคอลัมน์ id แยกต่างหาก เพราะรหัสคูปองมีความหมายชัดเจน ไม่เปลี่ยนแปลงบ่อย และมักถูกใช้อ้างอิงโดยตรงจากผู้ใช้งาน (กรอกรหัสคูปอง) จึงไม่จำเป็นต้องมี surrogate key เพิ่ม

---

## สรุปท้ายบท

ใน Part นี้เราได้เรียนรู้แก่นสำคัญของ Data Integrity ในฐานข้อมูลเชิงสัมพันธ์ ได้แก่:

- **Primary Key** คือเอกลักษณ์เฉพาะของแถว ต้องไม่ซ้ำและห้ามเป็น NULL มีได้เพียง 1 ตัวต่อตาราง
- กำหนด Primary Key ได้ทั้งแบบ column-level (`PRIMARY KEY` ต่อท้ายคอลัมน์) และ table-level (`CONSTRAINT ชื่อ PRIMARY KEY (คอลัมน์)`) — แนะนำใช้ table-level พร้อมตั้งชื่อเองเสมอ
- **Composite Primary Key** ใช้เมื่อไม่มีคอลัมน์เดียวที่ระบุเอกลักษณ์ของแถวได้ พบมากในตารางเชื่อมความสัมพันธ์แบบ many-to-many
- เพิ่ม/ลบ Primary Key ในตารางที่มีอยู่แล้วด้วย `ALTER TABLE ... ADD/DROP CONSTRAINT`
- **Unique Constraint** ต่างจาก Primary Key ตรงที่มีได้หลายตัวต่อตาราง และยอมให้มีค่า NULL ได้ (โดยค่าเริ่มต้น NULL แต่ละตัวไม่ถือว่าซ้ำกัน)
- ตั้งแต่ PostgreSQL 15 สามารถใช้ `UNIQUE NULLS NOT DISTINCT` เพื่อให้ NULL ถือว่าซ้ำกันได้ เมื่อ business logic ต้องการเช่นนั้น
- Primary Key และ Unique Constraint แต่ละตัวสร้าง **Unique B-tree Index อัตโนมัติ** เสมอ ซึ่งช่วยทั้งเรื่องการตรวจสอบความซ้ำและความเร็วในการค้นหา
- error `duplicate key value violates unique constraint` วินิจฉัยได้จากชื่อ constraint และคอลัมน์ใน `DETAIL` และมักแก้ด้วย `ON CONFLICT` หรือการปรับ sequence
- การเลือกชนิดข้อมูลของ Primary Key (integer/bigint identity vs UUID) ต้องพิจารณาทั้งประสิทธิภาพ ความปลอดภัย และบริบทของระบบ

Constraint ทั้งสองชนิดนี้เป็นรากฐานสำคัญก่อนที่เราจะไปเรียนรู้ **Foreign Key** ใน Part ถัดไป ซึ่งจะเป็นกลไกที่เชื่อมโยงตารางต่าง ๆ เข้าด้วยกันโดยอ้างอิง Primary Key/Unique Constraint ที่เราเพิ่งเรียนรู้ไปนี้เอง

---

## แบบฝึกหัดท้ายบท

**ข้อ 1.** จงอธิบายความแตกต่างระหว่าง Primary Key และ Unique Constraint อย่างน้อย 3 ข้อ

<details>
<summary>เฉลยข้อ 1</summary>

1. Primary Key มีได้เพียง 1 ตัวต่อตาราง ส่วน Unique Constraint มีได้หลายตัว
2. Primary Key ห้ามเป็น NULL เสมอ ส่วน Unique Constraint อนุญาตให้เป็น NULL ได้ (ตามค่าเริ่มต้น)
3. Primary Key มักใช้เป็นตัวที่ Foreign Key จากตารางอื่นมาอ้างอิง ส่วน Unique Constraint ใช้เพื่อบังคับความไม่ซ้ำของคอลัมน์ที่ไม่ใช่ตัวระบุหลักของแถว (เช่น email, username)
4. ทั้งสองสร้าง Unique B-tree Index ให้อัตโนมัติเหมือนกัน

</details>

**ข้อ 2.** จงเขียนคำสั่ง `CREATE TABLE` สร้างตาราง `employees` ที่มี `employee_id` เป็น Primary Key แบบ table-level พร้อมตั้งชื่อ constraint เองว่า `pk_employees`

<details>
<summary>เฉลยข้อ 2</summary>

```sql
CREATE TABLE employees (
    employee_id INTEGER,
    full_name   VARCHAR(100) NOT NULL,
    CONSTRAINT pk_employees PRIMARY KEY (employee_id)
);
```

</details>

**ข้อ 3.** ตาราง `enrollments` เก็บว่านักเรียนคนใดลงทะเบียนวิชาใดบ้าง (many-to-many ระหว่าง students และ courses) จงออกแบบ Primary Key ที่เหมาะสมให้ตารางนี้

<details>
<summary>เฉลยข้อ 3</summary>

ควรใช้ Composite Primary Key จากคู่ (student_id, course_id) เพราะนักเรียนหนึ่งคนลงทะเบียนได้หลายวิชา และวิชาหนึ่งวิชามีนักเรียนลงทะเบียนได้หลายคน แต่คู่ (นักเรียน, วิชา) ไม่ควรซ้ำกัน:

```sql
CREATE TABLE enrollments (
    student_id INTEGER,
    course_id  INTEGER,
    enrolled_at TIMESTAMP DEFAULT now(),
    CONSTRAINT pk_enrollments PRIMARY KEY (student_id, course_id)
);
```

</details>

**ข้อ 4.** จงเขียนคำสั่งเพิ่ม Unique Constraint ชื่อ `uq_employees_email` ให้กับคอลัมน์ `email` ในตาราง `employees` ที่มีอยู่แล้ว

<details>
<summary>เฉลยข้อ 4</summary>

```sql
ALTER TABLE employees
    ADD CONSTRAINT uq_employees_email UNIQUE (email);
```

(หมายเหตุ: ต้องเพิ่มคอลัมน์ `email` เข้าไปในตารางก่อน ถ้ายังไม่มี ด้วย `ALTER TABLE employees ADD COLUMN email VARCHAR(255);`)

</details>

**ข้อ 5.** เมื่อรันคำสั่งต่อไปนี้แล้วเกิด error จงอธิบายว่าเกิดจากอะไร และแก้ไขอย่างไร

```sql
INSERT INTO customers (customer_id, full_name, email, phone, member_code)
VALUES (1, 'ทดสอบ', 'test@example.com', '0800000000', 'MEM999');
```

```
ERROR:  duplicate key value violates unique constraint "customers_pkey"
DETAIL:  Key (customer_id)=(1) already exists.
```

<details>
<summary>เฉลยข้อ 5</summary>

เกิดจากมีแถวที่ `customer_id = 1` อยู่แล้วในตาราง `customers` (คือ "สมชาย ใจดี") การ INSERT ครั้งใหม่พยายามใช้ค่า `customer_id` เดิมซ้ำ ซึ่งละเมิด Primary Key Constraint

วิธีแก้: ตรวจสอบด้วย `SELECT * FROM customers WHERE customer_id = 1;` ก่อนเสมอ แล้วเลือก `customer_id` ค่าใหม่ที่ยังไม่ถูกใช้ หรือถ้าตั้งใจจะอัปเดตข้อมูลเดิม ให้ใช้ `UPDATE` แทน `INSERT` หรือใช้ `INSERT ... ON CONFLICT (customer_id) DO UPDATE ...`

</details>

**ข้อ 6.** จงอธิบายว่าเหตุใด Unique Constraint แบบมาตรฐานจึงยอมให้มีค่า NULL ซ้ำกันได้หลายแถว และจะแก้ไขพฤติกรรมนี้อย่างไรใน PostgreSQL 15 ขึ้นไป

<details>
<summary>เฉลยข้อ 6</summary>

เพราะตามมาตรฐาน SQL ค่า `NULL` หมายถึง "ไม่ทราบค่า" (unknown) การเปรียบเทียบ `NULL = NULL` จึงได้ผลลัพธ์เป็น unknown เสมอ ไม่ใช่ true ดังนั้นระบบจึงไม่ถือว่า NULL สองตัวเท่ากัน (ซ้ำกัน) จึงยอมให้มีหลายแถวที่คอลัมน์นั้นเป็น NULL ได้พร้อมกัน

แก้ไขได้ด้วยการระบุ `UNIQUE NULLS NOT DISTINCT (column)` แทน `UNIQUE (column)` ตอนสร้าง constraint ซึ่งจะทำให้ PostgreSQL ถือว่าค่า NULL ทุกตัวซ้ำกัน อนุญาตให้มีแถวที่คอลัมน์นั้นเป็น NULL ได้เพียงแถวเดียวเท่านั้น (ใช้ได้ตั้งแต่ PostgreSQL 15)

</details>

**ข้อ 7.** จงเขียนคำสั่งตรวจสอบว่าตาราง `products` มี Index อะไรบ้าง โดยใช้ catalog view `pg_indexes`

<details>
<summary>เฉลยข้อ 7</summary>

```sql
SELECT indexname, indexdef
FROM pg_indexes
WHERE tablename = 'products';
```

หรือใช้คำสั่ง `\d products` ใน psql ก็แสดงข้อมูล index ได้เช่นกัน

</details>

**ข้อ 8.** ระหว่าง `BIGINT GENERATED ALWAYS AS IDENTITY` กับ `UUID` แบบใดเหมาะกับการเป็น Primary Key ของตาราง `payment_transactions` ที่ต้อง expose transaction ID ให้ลูกค้าเห็นในใบเสร็จ และไม่ต้องการให้ลูกค้าเดาจำนวนธุรกรรมทั้งหมดของร้านได้ จงอธิบายเหตุผล

<details>
<summary>เฉลยข้อ 8</summary>

ควรใช้ `UUID` เป็นตัวที่ expose ให้ลูกค้าเห็น เพราะ UUID เป็นค่าสุ่ม เดาไม่ได้ ทำให้ลูกค้าไม่สามารถเดาหรือคำนวณจำนวนธุรกรรมทั้งหมดของร้านจากรูปแบบเลขที่ต่อเนื่องกันได้ (ต่างจาก BIGINT identity ที่เรียงลำดับ ทำให้เดาปริมาณธุรกรรมได้ง่าย)

ในทางปฏิบัติ อาจออกแบบแบบผสมผสาน: ใช้ `BIGINT identity` เป็น Primary Key ภายในสำหรับ join ที่รวดเร็ว และมีคอลัมน์ `UUID UNIQUE` แยกต่างหากสำหรับแสดงในใบเสร็จหรือ public-facing reference

</details>

**ข้อ 9.** จงออกแบบตาราง `product_tags` ที่เก็บว่าสินค้าแต่ละชิ้นมีแท็กอะไรบ้าง (สินค้าหนึ่งชิ้นมีได้หลายแท็ก และแท็กหนึ่งใช้กับสินค้าได้หลายชิ้น) พร้อมกำหนด constraint ที่เหมาะสม

<details>
<summary>เฉลยข้อ 9</summary>

```sql
CREATE TABLE product_tags (
    product_id INTEGER NOT NULL,
    tag_name   VARCHAR(30) NOT NULL,
    CONSTRAINT pk_product_tags PRIMARY KEY (product_id, tag_name)
);
```

ใช้ Composite Primary Key จากคู่ (product_id, tag_name) เพราะเป็นความสัมพันธ์แบบ many-to-many แบบเดียวกับ order_items และ enrollments — คู่ของสินค้ากับแท็กไม่ควรซ้ำกัน แต่สินค้าเดียวมีหลายแท็กได้ และแท็กเดียวใช้กับหลายสินค้าได้

</details>

**ข้อ 10.** จงเขียนคำสั่งลบ Primary Key ของตาราง `enrollments` (จากข้อ 3) แล้วสร้างใหม่โดยเปลี่ยนชื่อ constraint เป็น `pk_enrollments_v2`

<details>
<summary>เฉลยข้อ 10</summary>

```sql
ALTER TABLE enrollments DROP CONSTRAINT pk_enrollments;

ALTER TABLE enrollments
    ADD CONSTRAINT pk_enrollments_v2 PRIMARY KEY (student_id, course_id);
```

</details>

---

**Part ถัดไป:** [Part 017 — Foreign Key และความสัมพันธ์ระหว่างตาราง](./part-017-foreign-key.md)
