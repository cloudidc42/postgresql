# Part 017: Foreign Key และ Referential Integrity, ON DELETE/UPDATE

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับพื้นฐาน | Part 017

## เป้าหมายการเรียนรู้

เมื่อเรียนจบ Part นี้ ผู้เรียนจะสามารถ:

- อธิบายได้ว่า Foreign Key (FK) คืออะไร และทำไมฐานข้อมูลเชิงสัมพันธ์ (Relational Database) จึงต้องมี Referential Integrity
- กำหนด Foreign Key ได้ทั้งตอนสร้างตาราง (`CREATE TABLE ... REFERENCES`) และเพิ่มภายหลังด้วย `ALTER TABLE ... ADD CONSTRAINT`
- สร้าง Composite Foreign Key ที่อ้างอิงหลายคอลัมน์พร้อมกันได้
- อ่านและแก้ไข error ที่พบบ่อยที่สุดสองแบบของ FK คือ `insert or update violates foreign key constraint` และ `update or delete violates foreign key constraint`
- เลือกใช้ `ON DELETE` ได้อย่างเหมาะสมทั้ง 5 แบบ: `CASCADE`, `RESTRICT`, `SET NULL`, `SET DEFAULT`, `NO ACTION`
- เข้าใจ `ON UPDATE` และรู้ว่าเมื่อไหร่จำเป็นต้องใช้
- อธิบายความแตกต่างเชิงเทคนิคระหว่าง `NO ACTION` กับ `RESTRICT` โดยเฉพาะเรื่อง deferred constraint checking
- ออกแบบ Self-referencing Foreign Key สำหรับโครงสร้างแบบลำดับชั้น (hierarchy) เช่น employee-manager
- ปิด/เปิด FK ชั่วคราวสำหรับงาน bulk load ได้อย่างปลอดภัย และเข้าใจความเสี่ยงของการทำเช่นนั้น
- ออกแบบความสัมพันธ์ทั้งหมดของระบบร้านกาแฟ พร้อมเลือกนโยบาย `ON DELETE` ที่เหมาะสมในแต่ละจุดได้ด้วยเหตุผลทางธุรกิจ

---

## เตรียมข้อมูล

เพื่อความต่อเนื่องกับบทก่อนหน้า เราจะใช้สคีมา "ร้านกาแฟ" (coffee shop) ชุดเดิม ประกอบด้วยตาราง `categories`, `products`, `customers`, `orders`, และ `order_items` ที่มีความสัมพันธ์แบบ Foreign Key เชื่อมโยงกันอยู่แล้ว มาสร้างใหม่อีกครั้งเพื่อให้แน่ใจว่าทุกคนมีฐานข้อมูลชุดเดียวกันก่อนเริ่มบทเรียนนี้

```sql
-- ลบตารางเก่าทิ้งก่อน (ถ้ามี) เรียงจากตารางลูกไปตารางแม่
DROP TABLE IF EXISTS order_items;
DROP TABLE IF EXISTS orders;
DROP TABLE IF EXISTS products;
DROP TABLE IF EXISTS customers;
DROP TABLE IF EXISTS categories;

-- ตารางหมวดหมู่สินค้า
CREATE TABLE categories (
    category_id   SERIAL PRIMARY KEY,
    category_name VARCHAR(50) NOT NULL UNIQUE,
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ตารางสินค้า
CREATE TABLE products (
    product_id    SERIAL PRIMARY KEY,
    category_id   INTEGER NOT NULL REFERENCES categories(category_id),
    product_name  VARCHAR(100) NOT NULL,
    price         NUMERIC(10,2) NOT NULL CHECK (price >= 0),
    is_active     BOOLEAN NOT NULL DEFAULT true,
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ตารางลูกค้า
CREATE TABLE customers (
    customer_id   SERIAL PRIMARY KEY,
    full_name     VARCHAR(100) NOT NULL,
    email         VARCHAR(150) NOT NULL UNIQUE,
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

-- ตารางรายการสินค้าในคำสั่งซื้อ (junction table)
CREATE TABLE order_items (
    order_item_id SERIAL PRIMARY KEY,
    order_id      INTEGER NOT NULL REFERENCES orders(order_id),
    product_id    INTEGER NOT NULL REFERENCES products(product_id),
    quantity      INTEGER NOT NULL CHECK (quantity > 0),
    unit_price    NUMERIC(10,2) NOT NULL
);

-- เติมข้อมูลตัวอย่าง
INSERT INTO categories (category_name) VALUES
    ('กาแฟร้อน'), ('กาแฟเย็น'), ('เบเกอรี่'), ('เครื่องดื่มอื่นๆ');

INSERT INTO products (category_id, product_name, price) VALUES
    (1, 'เอสเพรสโซ่', 45.00),
    (1, 'อเมริกาโน่ร้อน', 50.00),
    (2, 'ลาเต้เย็น', 60.00),
    (2, 'คาปูชิโน่เย็น', 60.00),
    (3, 'ครัวซองต์', 55.00),
    (4, 'ชาไทย', 40.00);

INSERT INTO customers (full_name, email, phone) VALUES
    ('สมชาย ใจดี', 'somchai@example.com', '081-111-1111'),
    ('สมหญิง รักเรียน', 'somying@example.com', '082-222-2222'),
    ('วิชัย มั่นคง', 'wichai@example.com', '083-333-3333');

INSERT INTO orders (customer_id, status) VALUES
    (1, 'completed'),
    (2, 'completed'),
    (1, 'pending');

INSERT INTO order_items (order_id, product_id, quantity, unit_price) VALUES
    (1, 1, 2, 45.00),
    (1, 5, 1, 55.00),
    (2, 3, 1, 60.00),
    (3, 6, 2, 40.00);
```

ตรวจสอบว่าข้อมูลถูกต้อง:

```sql
SELECT o.order_id, c.full_name, p.product_name, oi.quantity, oi.unit_price
FROM order_items oi
JOIN orders o ON o.order_id = oi.order_id
JOIN customers c ON c.customer_id = o.customer_id
JOIN products p ON p.product_id = oi.product_id
ORDER BY o.order_id;
```

ผลลัพธ์:

```
 order_id |  full_name   |   product_name   | quantity | unit_price
----------+--------------+-------------------+----------+------------
        1 | สมชาย ใจดี   | เอสเพรสโซ่        |        2 |      45.00
        1 | สมชาย ใจดี   | ครัวซองต์         |        1 |      55.00
        2 | สมหญิง รักเรียน | ลาเต้เย็น      |        1 |      60.00
        3 | สมชาย ใจดี   | ชาไทย             |        2 |      40.00
(4 rows)
```

พร้อมแล้ว มาเริ่มเรียนเรื่อง Foreign Key กันครับ

---

## Step 161: Foreign Key คืออะไร — การรักษา Referential Integrity ระหว่างตาราง

### แนวคิดพื้นฐาน

**Foreign Key (FK)** คือคอลัมน์ (หรือกลุ่มคอลัมน์) ในตารางหนึ่งที่ "อ้างอิง" ไปยัง Primary Key หรือ Unique Key ของอีกตารางหนึ่ง จุดประสงค์หลักคือการรักษา **Referential Integrity** — หมายความว่า ข้อมูลที่อ้างอิงกันระหว่างตารางจะต้องสอดคล้องกันเสมอ ห้ามมีแถวไหนอ้างอิงไปยังข้อมูลที่ "ไม่มีอยู่จริง"

ตัวอย่างในสคีมาร้านกาแฟของเรา:

- `products.category_id` เป็น FK อ้างอิงไปที่ `categories.category_id`
- `orders.customer_id` เป็น FK อ้างอิงไปที่ `customers.customer_id`
- `order_items.order_id` เป็น FK อ้างอิงไปที่ `orders.order_id`
- `order_items.product_id` เป็น FK อ้างอิงไปที่ `products.product_id`

ความสัมพันธ์นี้ทำให้เราไม่มีทางเพิ่มสินค้า (`products`) ที่มี `category_id = 999` ได้ ถ้าหมวดหมู่ `999` ไม่มีอยู่จริงในตาราง `categories` — PostgreSQL จะปฏิเสธการ INSERT นั้นทันที

### ทำไมต้องมี Referential Integrity

ลองจินตนาการว่าไม่มี FK constraint เลย แล้วมีคนพลาด INSERT ข้อมูลแบบนี้:

```sql
-- ตัวอย่าง (สมมติ) หากไม่มี FK คุ้มครอง — ห้ามรันจริงกับตารางที่มี FK อยู่แล้ว
-- INSERT INTO order_items (order_id, product_id, quantity, unit_price)
-- VALUES (9999, 9999, 1, 100.00);
```

ถ้าไม่มี FK คุ้มครองอยู่ ข้อมูลแถวนี้จะกลายเป็น "ขยะ" (orphan record) ที่ชี้ไปยัง order และ product ที่ไม่มีอยู่จริง เมื่อทำรายงานยอดขายในภายหลัง ระบบอาจ error หรือแสดงผลผิดพลาดโดยไม่รู้สาเหตุ FK จึงเป็นเครื่องมือที่ฐานข้อมูลใช้ "การันตี" ว่าความสัมพันธ์ระหว่างข้อมูลจะถูกต้องเสมอ โดยไม่ต้องพึ่งพา application code ให้ตรวจสอบเอง

### พิสูจน์ด้วยตัวอย่างจริง

ลองสร้างคำสั่งซื้อที่อ้างอิงลูกค้าที่ไม่มีอยู่จริงดูครับ (`customer_id = 999`):

```sql
INSERT INTO orders (customer_id, status) VALUES (999, 'pending');
```

ผลลัพธ์ (error):

```
ERROR:  insert or update on table "orders" violates foreign key constraint "orders_customer_id_fkey"
DETAIL:  Key (customer_id)=(999) is not present in table "customers".
```

PostgreSQL ปฏิเสธคำสั่งนี้ทันที เพราะไม่มีลูกค้า `customer_id = 999` อยู่ในตาราง `customers` — นี่คือ Referential Integrity ทำงานอยู่เบื้องหลัง

### สรุปคุณสมบัติของ Foreign Key

| คุณสมบัติ | รายละเอียด |
|---|---|
| ตารางที่อ้างอิง (Referencing table) | ตารางที่มีคอลัมน์ FK เช่น `orders` |
| ตารางที่ถูกอ้างอิง (Referenced table) | ตารางที่มี PK/Unique key เช่น `customers` |
| ค่า NULL | FK column สามารถเป็น NULL ได้ (เว้นแต่กำหนด `NOT NULL` เพิ่ม) — หมายถึง "ไม่มีความสัมพันธ์" |
| การตรวจสอบ | PostgreSQL ตรวจสอบทุกครั้งที่ INSERT/UPDATE ตารางลูก และ UPDATE/DELETE ตารางแม่ |
| Index | PostgreSQL **ไม่** สร้าง index ให้ FK column โดยอัตโนมัติ (ต่างจาก PK) ควรสร้างเองเพื่อ performance |

> **หมายเหตุสำคัญ**: PostgreSQL จะสร้าง index ให้อัตโนมัติเฉพาะฝั่ง Primary/Unique Key เท่านั้น ส่วนฝั่ง FK column (เช่น `orders.customer_id`) จะไม่มี index ให้อัตโนมัติ ถ้าตารางมีข้อมูลเยอะและมีการ JOIN หรือ DELETE บ่อย ควรสร้าง index เพิ่มเอง เราจะพูดเรื่องนี้ละเอียดใน Part ที่ว่าด้วย Indexing

---

## Step 162: การกำหนด Foreign Key ตอนสร้างตาราง และการเพิ่มทีหลังด้วย ALTER TABLE

### วิธีที่ 1: กำหนดตอนสร้างตาราง (Inline REFERENCES)

รูปแบบสั้น ๆ ที่เราใช้ไปแล้วในส่วน "เตรียมข้อมูล":

```sql
CREATE TABLE products (
    product_id    SERIAL PRIMARY KEY,
    category_id   INTEGER NOT NULL REFERENCES categories(category_id),
    product_name  VARCHAR(100) NOT NULL,
    price         NUMERIC(10,2) NOT NULL CHECK (price >= 0)
);
```

รูปแบบนี้เรียกว่า **column constraint** — เขียนต่อท้ายคอลัมน์เลย เหมาะกับกรณีที่ FK อ้างอิงคอลัมน์เดียว

### วิธีที่ 2: กำหนดตอนสร้างตารางแบบ table constraint (ตั้งชื่อเอง)

การตั้งชื่อ constraint เองเป็นวิธีที่แนะนำในงานจริง เพราะทำให้ error message อ่านง่ายขึ้น และแก้ไข/ลบ constraint ได้สะดวก:

```sql
CREATE TABLE order_items_v2 (
    order_item_id SERIAL PRIMARY KEY,
    order_id      INTEGER NOT NULL,
    product_id    INTEGER NOT NULL,
    quantity      INTEGER NOT NULL CHECK (quantity > 0),
    unit_price    NUMERIC(10,2) NOT NULL,
    CONSTRAINT fk_order_items_v2_order
        FOREIGN KEY (order_id) REFERENCES orders(order_id),
    CONSTRAINT fk_order_items_v2_product
        FOREIGN KEY (product_id) REFERENCES products(product_id)
);

-- ลบตารางตัวอย่างทิ้ง ไม่ได้ใช้งานต่อ
DROP TABLE order_items_v2;
```

รูปแบบ `CONSTRAINT ชื่อ FOREIGN KEY (คอลัมน์) REFERENCES ตาราง(คอลัมน์)` เรียกว่า **table constraint** เขียนแยกจากนิยามคอลัมน์ ข้อดีคือใช้ได้ทั้งกรณีคอลัมน์เดียวและหลายคอลัมน์ (composite FK ที่จะพูดถึงใน Step 163)

### วิธีที่ 3: เพิ่ม Foreign Key ทีหลังด้วย ALTER TABLE

ในงานจริง บางครั้งตารางถูกสร้างไปแล้วโดยยังไม่มี FK (เช่น ตอน migrate ข้อมูลเก่า หรือออกแบบเพิ่มทีหลัง) เราสามารถเพิ่ม FK ทีหลังได้ด้วย `ALTER TABLE ... ADD CONSTRAINT`

ลองสร้างตารางสาขาร้าน (`branches`) และสมมติว่า `orders` ต้องมีคอลัมน์ `branch_id` เพิ่มเข้ามาทีหลัง:

```sql
-- สร้างตารางสาขา
CREATE TABLE branches (
    branch_id   SERIAL PRIMARY KEY,
    branch_name VARCHAR(100) NOT NULL
);

INSERT INTO branches (branch_name) VALUES ('สาขาสยาม'), ('สาขาอโศก');

-- เพิ่มคอลัมน์ branch_id ให้ orders (ยังไม่มี FK)
ALTER TABLE orders ADD COLUMN branch_id INTEGER;

-- อัปเดตข้อมูลเดิมให้มีสาขา
UPDATE orders SET branch_id = 1;

-- ตอนนี้ค่อยเพิ่ม FK constraint ทีหลัง
ALTER TABLE orders
    ADD CONSTRAINT fk_orders_branch
    FOREIGN KEY (branch_id) REFERENCES branches(branch_id);
```

ผลลัพธ์:

```
ALTER TABLE
```

เมื่อ ALTER TABLE เพิ่ม FK ทีหลัง PostgreSQL จะ **สแกนข้อมูลที่มีอยู่ทั้งหมด** ในตาราง `orders` เพื่อตรวจสอบว่าทุกแถวมีค่า `branch_id` ที่ถูกต้องหรือไม่ (หรือเป็น NULL) ถ้ามีข้อมูลที่ผิดพลาดอยู่ก่อนแล้ว คำสั่งนี้จะ error ทันที

ลองทดสอบกรณีข้อมูลผิดพลาดดูครับ:

```sql
ALTER TABLE orders ADD COLUMN test_branch_id INTEGER DEFAULT 999;

ALTER TABLE orders
    ADD CONSTRAINT fk_orders_test_branch
    FOREIGN KEY (test_branch_id) REFERENCES branches(branch_id);
```

ผลลัพธ์ (error):

```
ERROR:  insert or update on table "orders" violates foreign key constraint "fk_orders_test_branch"
DETAIL:  Key (test_branch_id)=(999) is not present in table "branches".
```

ล้างตารางทดสอบทิ้ง:

```sql
ALTER TABLE orders DROP COLUMN test_branch_id;
```

### เทคนิค: เพิ่ม FK แบบไม่ล็อกตารางนาน (NOT VALID + VALIDATE CONSTRAINT)

ในระบบ production ที่มีข้อมูลเยอะมาก การ ADD CONSTRAINT ปกติจะสแกนและล็อกตารางเป็นเวลานาน PostgreSQL มีเทคนิคแยกสองขั้นตอนเพื่อลด downtime:

```sql
-- ขั้นตอนที่ 1: เพิ่ม constraint โดยยังไม่ validate ข้อมูลเก่า (เร็ว, ล็อกสั้น)
ALTER TABLE orders
    ADD CONSTRAINT fk_orders_branch_v2
    FOREIGN KEY (branch_id) REFERENCES branches(branch_id)
    NOT VALID;

-- ขั้นตอนที่ 2: validate ข้อมูลภายหลัง (ใช้ lock ที่เบากว่า)
ALTER TABLE orders VALIDATE CONSTRAINT fk_orders_branch_v2;
```

ระหว่างขั้นตอนที่ 1 และ 2 constraint จะยังคุ้มครอง **ข้อมูลใหม่** ทันที (INSERT/UPDATE ใหม่ต้องผ่าน FK) แต่ยังไม่รับประกันว่าข้อมูลเก่าถูกต้องจนกว่าจะรัน `VALIDATE CONSTRAINT` เทคนิคนี้สำคัญมากสำหรับ DBA ระดับ production เพราะช่วยลด lock time บนตารางขนาดใหญ่

ล้าง constraint ทดสอบทิ้ง:

```sql
ALTER TABLE orders DROP CONSTRAINT fk_orders_branch_v2;
ALTER TABLE orders DROP CONSTRAINT fk_orders_branch;
ALTER TABLE orders DROP COLUMN branch_id;
DROP TABLE branches;
```

### ตรวจสอบ FK ที่มีอยู่ในตาราง

```sql
SELECT
    tc.constraint_name,
    tc.table_name,
    kcu.column_name,
    ccu.table_name  AS foreign_table_name,
    ccu.column_name AS foreign_column_name
FROM information_schema.table_constraints tc
JOIN information_schema.key_column_usage kcu
    ON tc.constraint_name = kcu.constraint_name
JOIN information_schema.constraint_column_usage ccu
    ON tc.constraint_name = ccu.constraint_name
WHERE tc.constraint_type = 'FOREIGN KEY'
  AND tc.table_schema = 'public';
```

หรือใช้คำสั่ง psql ที่สะดวกกว่า:

```sql
\d order_items
```

ผลลัพธ์ (บางส่วน):

```
Foreign-key constraints:
    "order_items_order_id_fkey" FOREIGN KEY (order_id) REFERENCES orders(order_id)
    "order_items_product_id_fkey" FOREIGN KEY (product_id) REFERENCES products(product_id)
```

---

## Step 163: Composite Foreign Key — FK ที่อ้างอิงหลายคอลัมน์

### เมื่อไหร่ที่ต้องใช้ Composite FK

บางครั้ง Primary Key ของตารางแม่ไม่ได้เป็นคอลัมน์เดียว แต่เป็น **Composite Primary Key** (หลายคอลัมน์รวมกัน) ในกรณีนี้ FK ที่อ้างอิงไปก็ต้องเป็น Composite FK ด้วยเช่นกัน — ใช้หลายคอลัมน์อ้างอิงพร้อมกัน

ตัวอย่าง: สมมติร้านกาแฟมีระบบ "ราคาสินค้าตามสาขา" ที่ต้องระบุทั้ง `branch_id` และ `product_id` คู่กันเป็น Primary Key:

```sql
-- สร้างตารางสาขาใหม่สำหรับตัวอย่างนี้
CREATE TABLE store_branches (
    branch_id   INTEGER NOT NULL,
    region_code VARCHAR(10) NOT NULL,
    branch_name VARCHAR(100) NOT NULL,
    PRIMARY KEY (branch_id, region_code)
);

INSERT INTO store_branches (branch_id, region_code, branch_name) VALUES
    (1, 'BKK', 'สาขาสยาม'),
    (2, 'BKK', 'สาขาอโศก'),
    (1, 'CNX', 'สาขานิมมาน');

-- ตารางราคาสินค้าตามสาขา ใช้ composite FK อ้างอิงไปที่ store_branches
CREATE TABLE branch_product_prices (
    id            SERIAL PRIMARY KEY,
    branch_id     INTEGER NOT NULL,
    region_code   VARCHAR(10) NOT NULL,
    product_id    INTEGER NOT NULL REFERENCES products(product_id),
    special_price NUMERIC(10,2) NOT NULL,
    CONSTRAINT fk_branch_product_prices_branch
        FOREIGN KEY (branch_id, region_code)
        REFERENCES store_branches (branch_id, region_code)
);
```

สังเกตว่า `FOREIGN KEY (branch_id, region_code) REFERENCES store_branches (branch_id, region_code)` ระบุคอลัมน์สองตัวพร้อมกัน คอลัมน์ต้องเรียงลำดับให้ตรงกับ Primary Key ของตารางแม่

### ทดสอบว่า Composite FK ทำงานถูกต้อง

```sql
-- ข้อมูลนี้ถูกต้อง เพราะ (1, 'BKK') มีอยู่ใน store_branches
INSERT INTO branch_product_prices (branch_id, region_code, product_id, special_price)
VALUES (1, 'BKK', 1, 40.00);
```

ผลลัพธ์:

```
INSERT 0 1
```

ลองใส่ข้อมูลที่คู่ `(branch_id, region_code)` ไม่ตรงกับตารางแม่:

```sql
-- (2, 'CNX') ไม่มีอยู่จริงใน store_branches (มีแค่ 1-CNX และ 2-BKK)
INSERT INTO branch_product_prices (branch_id, region_code, product_id, special_price)
VALUES (2, 'CNX', 1, 40.00);
```

ผลลัพธ์ (error):

```
ERROR:  insert or update on table "branch_product_prices" violates foreign key constraint "fk_branch_product_prices_branch"
DETAIL:  Key (branch_id, region_code)=(2, CNX) is not present in table "store_branches".
```

จุดสำคัญคือ แม้ `branch_id = 2` จะมีอยู่จริง (คู่กับ `BKK`) และ `region_code = 'CNX'` ก็มีอยู่จริง (คู่กับ `branch_id = 1`) แต่ **คู่ (2, CNX) ไม่มีอยู่จริง** — Composite FK ตรวจสอบทั้งคู่พร้อมกันเสมอ ไม่ใช่ตรวจทีละคอลัมน์แยกกัน

### ข้อกำหนดสำคัญของ Composite FK

ตารางแม่ต้องมี **UNIQUE constraint หรือ PRIMARY KEY ที่ครอบคลุมคอลัมน์ชุดเดียวกันพอดี** กับที่ FK อ้างอิง มิฉะนั้นจะสร้างไม่ได้:

```sql
-- ทดสอบ: ถ้าตารางแม่ไม่มี unique constraint ครอบคลุมคอลัมน์ที่จะอ้างอิง จะ error
CREATE TABLE bad_composite_test (
    x INTEGER,
    y INTEGER,
    FOREIGN KEY (x, y) REFERENCES store_branches (branch_id, branch_name)
);
```

ผลลัพธ์ (error):

```
ERROR:  there is no unique constraint matching given keys for referenced table "store_branches"
```

เพราะ `store_branches` มี Primary Key เป็น `(branch_id, region_code)` ไม่ใช่ `(branch_id, branch_name)` จึงไม่มี unique constraint ที่ตรงกับคู่คอลัมน์ที่พยายามอ้างอิง

ล้างตารางตัวอย่างทิ้ง:

```sql
DROP TABLE branch_product_prices;
DROP TABLE store_branches;
```

---

## Step 164: ข้อผิดพลาดที่พบบ่อยของ Foreign Key

FK error ที่เจอบ่อยที่สุดมีอยู่ 2 รูปแบบหลัก มาดูรายละเอียดและวิธีแก้ทีละแบบ

### รูปแบบที่ 1: `insert or update violates foreign key constraint`

เกิดขึ้นเมื่อ **ตารางลูก** (referencing table) พยายาม INSERT หรือ UPDATE ค่าที่ไม่มีอยู่ในตารางแม่

```sql
-- พยายามเพิ่มสินค้าในหมวดหมู่ที่ไม่มีอยู่จริง
INSERT INTO products (category_id, product_name, price)
VALUES (999, 'กาแฟมนุษย์ต่างดาว', 199.00);
```

ผลลัพธ์:

```
ERROR:  insert or update on table "products" violates foreign key constraint "products_category_id_fkey"
DETAIL:  Key (category_id)=(999) is not present in table "categories".
```

**วิธีแก้**: ตรวจสอบก่อนว่า `category_id` ที่ต้องการใช้มีอยู่จริงหรือไม่

```sql
SELECT * FROM categories WHERE category_id = 999;
```

```
 category_id | category_name | created_at
-------------+---------------+------------
(0 rows)
```

ไม่พบ → ต้องสร้างหมวดหมู่ใหม่ก่อน หรือใช้ `category_id` ที่มีอยู่จริง:

```sql
INSERT INTO categories (category_name) VALUES ('เครื่องดื่มพิเศษ')
RETURNING category_id;
```

```
 category_id
-------------
           5
(1 row)
```

```sql
INSERT INTO products (category_id, product_name, price)
VALUES (5, 'กาแฟมนุษย์ต่างดาว', 199.00);
```

```
INSERT 0 1
```

การ UPDATE ก็ให้ผลแบบเดียวกัน ลองดูตัวอย่าง:

```sql
UPDATE products SET category_id = 999 WHERE product_id = 1;
```

```
ERROR:  insert or update on table "products" violates foreign key constraint "products_category_id_fkey"
DETAIL:  Key (category_id)=(999) is not present in table "categories".
```

### รูปแบบที่ 2: `update or delete violates foreign key constraint`

เกิดขึ้นเมื่อ **ตารางแม่** (referenced table) ถูกพยายาม DELETE หรือ UPDATE ค่า key ที่ยังมีตารางลูกอ้างอิงอยู่ (นี่คือพฤติกรรม default ที่เรียกว่า `NO ACTION`)

```sql
-- ลองลบลูกค้า customer_id = 1 ที่ยังมี orders อ้างอิงอยู่
DELETE FROM customers WHERE customer_id = 1;
```

ผลลัพธ์:

```
ERROR:  update or delete on table "customers" violates foreign key constraint "orders_customer_id_fkey" on table "orders"
DETAIL:  Key (customer_id)=(1) is still referenced from table "orders".
```

สังเกตความแตกต่าง: error นี้บอกว่า `orders` (ตารางลูก) ยังอ้างอิง `customer_id = 1` อยู่ PostgreSQL จึงปฏิเสธการลบ เพราะถ้าลบไปจริงจะทำให้ `orders` มี `customer_id` ที่ไม่มีอยู่จริง (orphan record) — นี่คือหัวใจของ Referential Integrity

**วิธีแก้มีหลายทาง**:

1. ลบ/ย้ายข้อมูลตารางลูกก่อน แล้วค่อยลบตารางแม่
2. เปลี่ยนนโยบาย `ON DELETE` ของ FK ให้จัดการอัตโนมัติ (จะสอนใน Step 165)
3. ถ้าไม่ต้องการลบจริง ใช้ soft delete แทน (เพิ่มคอลัมน์ `is_deleted` หรือ `deleted_at`)

ตัวอย่างการลบตารางลูกก่อน:

```sql
-- ตรวจสอบก่อนว่าลูกค้าคนนี้มี orders อะไรบ้าง
SELECT order_id, status FROM orders WHERE customer_id = 1;
```

```
 order_id |  status
----------+-----------
        1 | completed
        3 | pending
(2 rows)
```

เนื่องจาก `order_items` ยังอ้างอิง `orders` อยู่ด้วย จึงต้องลบตามลำดับจากลูกไปหาแม่:

```sql
BEGIN;

DELETE FROM order_items WHERE order_id IN (SELECT order_id FROM orders WHERE customer_id = 1);
DELETE FROM orders WHERE customer_id = 1;
DELETE FROM customers WHERE customer_id = 1;

ROLLBACK;  -- ในตัวอย่างนี้เรา rollback ไว้ก่อน เพื่อเก็บข้อมูลไว้ใช้ในบทถัดไป
```

ผลลัพธ์:

```
DELETE 2
DELETE 2
DELETE 1
ROLLBACK
```

> เราใช้ `ROLLBACK` เพื่อยกเลิกการลบ เนื่องจากต้องการเก็บข้อมูลตัวอย่างไว้ใช้ในหัวข้อถัดไป ในการใช้งานจริง หากต้องการลบจริง ให้ใช้ `COMMIT` แทน

### ตารางสรุป error ทั้งสองแบบ

| Error | เกิดขึ้นเมื่อ | ตารางที่ถูกปฏิเสธคำสั่ง | วิธีแก้หลัก |
|---|---|---|---|
| `insert or update on table "X" violates foreign key constraint` | INSERT/UPDATE ตารางลูก ด้วยค่าที่ไม่มีในตารางแม่ | ตารางลูก (X) | ตรวจสอบค่าที่จะใส่ให้มีอยู่จริงในตารางแม่ก่อน |
| `update or delete on table "Y" violates foreign key constraint ... on table "Z"` | DELETE/UPDATE ตารางแม่ ที่ยังมีตารางลูกอ้างอิงอยู่ | ตารางแม่ (Y) | ลบ/ย้ายข้อมูลตารางลูกก่อน หรือกำหนด ON DELETE ให้เหมาะสม |

---

## Step 165: ON DELETE actions — CASCADE, RESTRICT, SET NULL, SET DEFAULT, NO ACTION

เมื่อเรากำหนด FK เราสามารถระบุ **นโยบาย** ว่าจะเกิดอะไรขึ้นกับตารางลูก เมื่อแถวในตารางแม่ที่ถูกอ้างอิงถูก DELETE — ผ่าน clause `ON DELETE` มีให้เลือก 5 แบบ มาดูทีละแบบพร้อมตัวอย่างข้อมูลก่อน-หลัง

### เตรียมตารางทดสอบสำหรับแต่ละแบบ

เพื่อให้เห็นพฤติกรรมชัดเจนแบบแยกส่วน เราจะสร้างตารางทดสอบคู่ขนาน 5 ชุด แทนการแก้ไขตารางจริงในสคีมาหลัก

```sql
-- ตารางแม่ร่วม (ใช้ซ้ำได้)
CREATE TABLE demo_categories (
    id   SERIAL PRIMARY KEY,
    name VARCHAR(50) NOT NULL
);

INSERT INTO demo_categories (name) VALUES ('เครื่องดื่ม'), ('ขนม'), ('อื่นๆ');
```

```
 id |    name
----+-------------
  1 | เครื่องดื่ม
  2 | ขนม
  3 | อื่นๆ
(3 rows)
```

### 1) ON DELETE CASCADE — ลบตามกันเป็นทอด ๆ

เมื่อแถวตารางแม่ถูกลบ แถวตารางลูกที่อ้างอิงจะถูก **ลบตามไปด้วยอัตโนมัติ**

```sql
CREATE TABLE demo_products_cascade (
    id          SERIAL PRIMARY KEY,
    category_id INTEGER REFERENCES demo_categories(id) ON DELETE CASCADE,
    name        VARCHAR(100)
);

INSERT INTO demo_products_cascade (category_id, name) VALUES
    (1, 'น้ำเปล่า'), (1, 'โค้ก'), (2, 'คุกกี้');
```

ข้อมูลก่อนลบ:

```sql
SELECT * FROM demo_products_cascade;
```

```
 id | category_id |   name
----+-------------+----------
  1 |           1 | น้ำเปล่า
  2 |           1 | โค้ก
  3 |           2 | คุกกี้
(3 rows)
```

ลบหมวดหมู่ `id = 1` (เครื่องดื่ม):

```sql
DELETE FROM demo_categories WHERE id = 1;
```

```
DELETE 1
```

ข้อมูลหลังลบ — สังเกตว่า `น้ำเปล่า` และ `โค้ก` หายไปด้วย โดยไม่ error เลย:

```sql
SELECT * FROM demo_products_cascade;
```

```
 id | category_id |  name
----+-------------+---------
  3 |           2 | คุกกี้
(1 row)
```

**ใช้เมื่อไหร่**: เหมาะกับความสัมพันธ์แบบ "ส่วนหนึ่งของ" (composition) ที่ข้อมูลลูกไม่มีความหมายถ้าไม่มีข้อมูลแม่ เช่น `order_items` ควรถูกลบตามเมื่อ `orders` ถูกลบ (เพราะรายการสินค้าในบิลไม่มีความหมายถ้าไม่มีบิล) แต่ต้องระวังมาก เพราะ CASCADE อาจลบข้อมูลจำนวนมากโดยไม่ตั้งใจ

### 2) ON DELETE RESTRICT — ห้ามลบเด็ดขาดถ้ายังมีลูกอ้างอิง

```sql
CREATE TABLE demo_products_restrict (
    id          SERIAL PRIMARY KEY,
    category_id INTEGER REFERENCES demo_categories(id) ON DELETE RESTRICT,
    name        VARCHAR(100)
);

INSERT INTO demo_products_restrict (category_id, name) VALUES (2, 'บราวนี่');
```

ลองลบหมวดหมู่ `id = 2` (ขนม) ที่ยังมีสินค้าอ้างอิงอยู่:

```sql
DELETE FROM demo_categories WHERE id = 2;
```

ผลลัพธ์:

```
ERROR:  update or delete on table "demo_categories" violates foreign key constraint "demo_products_restrict_category_id_fkey" on table "demo_products_restrict"
DETAIL:  Key (id)=(2) is still referenced from table "demo_products_restrict".
```

**ใช้เมื่อไหร่**: เหมาะกับข้อมูลสำคัญที่ไม่ต้องการให้ลบพลาดโดยไม่ตั้งใจ เช่น ไม่ควรให้ลบหมวดหมู่สินค้าที่ยังมีสินค้าขายอยู่ บังคับให้ผู้ใช้งานต้องจัดการสินค้าก่อนเสมอ

### 3) ON DELETE SET NULL — ตั้งค่า FK เป็น NULL แทนการลบ

```sql
CREATE TABLE demo_products_setnull (
    id          SERIAL PRIMARY KEY,
    category_id INTEGER REFERENCES demo_categories(id) ON DELETE SET NULL,
    name        VARCHAR(100)
);

INSERT INTO demo_products_setnull (category_id, name) VALUES (3, 'ของเล่นแมว');
```

ข้อมูลก่อนลบ:

```sql
SELECT * FROM demo_products_setnull;
```

```
 id | category_id |    name
----+-------------+-------------
  1 |           3 | ของเล่นแมว
(1 row)
```

ลบหมวดหมู่ `id = 3` (อื่นๆ):

```sql
DELETE FROM demo_categories WHERE id = 3;
```

```
DELETE 1
```

ข้อมูลหลังลบ — สินค้ายังอยู่ แต่ `category_id` กลายเป็น NULL:

```sql
SELECT * FROM demo_products_setnull;
```

```
 id | category_id |    name
----+-------------+-------------
  1 |             | ของเล่นแมว
(1 row)
```

> หมายเหตุ: ถ้าคอลัมน์ FK ถูกกำหนด `NOT NULL` ไว้ การใช้ `ON DELETE SET NULL` จะทำให้เกิด error ตอนลบตารางแม่ เพราะ PostgreSQL ไม่สามารถตั้งค่าเป็น NULL ในคอลัมน์ที่ห้ามเป็น NULL ได้

**ใช้เมื่อไหร่**: เหมาะกับกรณีที่ข้อมูลลูกยังมีความหมายอยู่แม้ไม่มีข้อมูลแม่แล้ว เช่น สินค้ายังขายได้แม้หมวดหมู่จะถูกลบไป (กลายเป็น "ไม่มีหมวดหมู่" แทนที่จะหายไปเลย)

### 4) ON DELETE SET DEFAULT — ตั้งค่า FK กลับเป็นค่า default

```sql
CREATE TABLE demo_products_setdefault (
    id          SERIAL PRIMARY KEY,
    category_id INTEGER REFERENCES demo_categories(id) ON DELETE SET DEFAULT DEFAULT 2,
    name        VARCHAR(100)
);

INSERT INTO demo_products_setdefault (category_id, name) VALUES (1, 'กาแฟกระป๋อง');
```

ข้อมูลก่อนลบ:

```sql
SELECT * FROM demo_products_setdefault;
```

```
 id | category_id |     name
----+-------------+---------------
  1 |           1 | กาแฟกระป๋อง
(1 row)
```

ลบหมวดหมู่ `id = 1` (เครื่องดื่ม):

```sql
DELETE FROM demo_categories WHERE id = 1;
```

```
DELETE 1
```

ข้อมูลหลังลบ — `category_id` เปลี่ยนเป็นค่า default คือ `2`:

```sql
SELECT * FROM demo_products_setdefault;
```

```
 id | category_id |     name
----+-------------+---------------
  1 |           2 | กาแฟกระป๋อง
(1 row)
```

> **ข้อควรระวังสำคัญ**: `SET DEFAULT` จะใช้งานได้จริงก็ต่อเมื่อค่า default นั้น "มีอยู่จริง" ในตารางแม่เสมอ (เช่น หมวดหมู่ `id = 2` ต้องไม่มีวันถูกลบ) ถ้าค่า default ถูกลบไปด้วย จะเกิด error ทันทีที่พยายาม DELETE แถวตารางแม่ตัวอื่น เพราะ PostgreSQL จะพยายามตั้งค่าเป็นค่าที่ไม่มีอยู่จริง ในทางปฏิบัติ `SET DEFAULT` มักใช้คู่กับหมวดหมู่พิเศษที่ป้องกันการลบด้วย `ON DELETE RESTRICT` ในตัวมันเอง (กรณีมีการอ้างอิงซ้อนกัน) หรือใช้ business rule ควบคุมไม่ให้ลบแถวนั้น

### 5) ON DELETE NO ACTION — ค่า default ของ PostgreSQL (ถ้าไม่ระบุอะไรเลย)

```sql
CREATE TABLE demo_products_noaction (
    id          SERIAL PRIMARY KEY,
    category_id INTEGER REFERENCES demo_categories(id) ON DELETE NO ACTION,
    name        VARCHAR(100)
);
```

พฤติกรรมของ `NO ACTION` **ในกรณีทั่วไปดูเหมือนกับ `RESTRICT`** คือปฏิเสธการลบถ้ายังมีลูกอ้างอิงอยู่:

```sql
INSERT INTO demo_categories (name) VALUES ('เครื่องเขียน') RETURNING id;
```

```
 id
----
  4
(1 row)
```

```sql
INSERT INTO demo_products_noaction (category_id, name) VALUES (4, 'ปากกา');

DELETE FROM demo_categories WHERE id = 4;
```

```
ERROR:  update or delete on table "demo_categories" violates foreign key constraint "demo_products_noaction_category_id_fkey" on table "demo_products_noaction"
DETAIL:  Key (id)=(4) is still referenced from table "demo_products_noaction".
```

ผลลัพธ์เหมือนกับ `RESTRICT` ทุกประการในตัวอย่างนี้ — แต่ความแตกต่างที่แท้จริงจะเห็นได้เมื่อพูดถึง **deferred constraint** ซึ่งจะอธิบายละเอียดใน Step 167

### ตารางสรุป ON DELETE ทั้ง 5 แบบ

| Action | พฤติกรรมเมื่อลบแถวแม่ | เหมาะกับ |
|---|---|---|
| `CASCADE` | ลบแถวลูกที่อ้างอิงทั้งหมดตามไปด้วย | ความสัมพันธ์แบบ composition เช่น orders → order_items |
| `RESTRICT` | ปฏิเสธการลบทันที ถ้ายังมีลูกอ้างอิง (ตรวจสอบทันที ไม่รอ deferred) | ข้อมูลอ้างอิงสำคัญที่ห้ามลบพลาด เช่น categories |
| `SET NULL` | ตั้งค่า FK ของลูกเป็น NULL | ลูกยังมีความหมายได้แม้ไม่มีแม่ เช่น product ไม่มีหมวดหมู่ |
| `SET DEFAULT` | ตั้งค่า FK ของลูกเป็นค่า default ที่กำหนดไว้ | ต้องการ fallback ไปยังค่ากลาง เช่น "หมวดหมู่ทั่วไป" |
| `NO ACTION` (default) | ปฏิเสธการลบ ถ้ายังมีลูกอ้างอิง (แต่รองรับ deferred check ได้) | พฤติกรรม default ที่ปลอดภัยที่สุด |

ล้างตารางทดสอบทั้งหมดทิ้ง:

```sql
DROP TABLE demo_products_cascade;
DROP TABLE demo_products_restrict;
DROP TABLE demo_products_setnull;
DROP TABLE demo_products_setdefault;
DROP TABLE demo_products_noaction;
DROP TABLE demo_categories;
```

---

## Step 166: ON UPDATE actions และเมื่อไหร่ควรใช้

### แนวคิด

`ON UPDATE` ทำงานเหมือนกับ `ON DELETE` ทุกประการ แต่เกิดขึ้นเมื่อ **ค่า Primary Key ของตารางแม่ถูก UPDATE** (เปลี่ยนค่า) แทนที่จะถูกลบ รองรับ action ชุดเดียวกันคือ `CASCADE`, `RESTRICT`, `SET NULL`, `SET DEFAULT`, `NO ACTION`

### ทำไมถึง "rare" (ไม่ค่อยได้ใช้)

ในทางปฏิบัติ Primary Key ที่ดีมักเป็นค่าที่ **ไม่เปลี่ยนแปลง** (immutable) เช่น `SERIAL`/`IDENTITY` หรือ UUID ที่สร้างขึ้นครั้งเดียวแล้วไม่แก้ไขอีก ด้วยเหตุนี้ `ON UPDATE CASCADE` จึงไม่ค่อยได้ใช้ในระบบที่ออกแบบมาอย่างดี เพราะ PK แทบไม่เคยถูก UPDATE เลย

แต่ก็มีบางกรณีที่ PK **เปลี่ยนแปลงได้** และ `ON UPDATE CASCADE` มีประโยชน์มาก เช่น:

- ใช้ **natural key** เป็น PK เช่น `product_code` (รหัสสินค้าที่เป็นตัวอักษร) ซึ่งอาจต้องเปลี่ยนแปลงตามนโยบายบริษัท
- ระบบ merge ข้อมูล (data migration) ที่ต้องเปลี่ยน ID เดิมเป็น ID ใหม่หลังรวมฐานข้อมูล

### ตัวอย่างการใช้งาน ON UPDATE CASCADE

สมมติว่าร้านกาแฟใช้ `category_code` (รหัสตัวอักษร) เป็น Primary Key ของหมวดหมู่แทนตัวเลข:

```sql
CREATE TABLE demo_category_natural (
    category_code VARCHAR(10) PRIMARY KEY,
    category_name VARCHAR(50) NOT NULL
);

INSERT INTO demo_category_natural VALUES ('HOT', 'กาแฟร้อน'), ('COLD', 'กาแฟเย็น');

CREATE TABLE demo_product_natural (
    id            SERIAL PRIMARY KEY,
    category_code VARCHAR(10) REFERENCES demo_category_natural(category_code)
        ON UPDATE CASCADE
        ON DELETE RESTRICT,
    product_name  VARCHAR(100)
);

INSERT INTO demo_product_natural (category_code, product_name) VALUES
    ('HOT', 'เอสเพรสโซ่'), ('HOT', 'อเมริกาโน่'), ('COLD', 'ลาเต้เย็น');
```

ข้อมูลก่อนแก้ไขรหัส:

```sql
SELECT * FROM demo_product_natural;
```

```
 id | category_code | product_name
----+----------------+---------------
  1 | HOT            | เอสเพรสโซ่
  2 | HOT            | อเมริกาโน่
  3 | COLD           | ลาเต้เย็น
(3 rows)
```

สมมติบริษัทต้องการเปลี่ยนรหัสหมวดหมู่จาก `HOT` เป็น `HOT_COFFEE` (ให้สื่อความหมายชัดเจนขึ้น):

```sql
UPDATE demo_category_natural SET category_code = 'HOT_COFFEE' WHERE category_code = 'HOT';
```

```
UPDATE 1
```

ข้อมูลหลังแก้ไข — สังเกตว่า `category_code` ในตารางลูกเปลี่ยนตามอัตโนมัติทั้งสองแถว โดยไม่ต้อง UPDATE ตารางลูกเอง:

```sql
SELECT * FROM demo_product_natural;
```

```
 id | category_code | product_name
----+----------------+---------------
  1 | HOT_COFFEE     | เอสเพรสโซ่
  2 | HOT_COFFEE     | อเมริกาโน่
  3 | COLD           | ลาเต้เย็น
(3 rows)
```

`ON UPDATE CASCADE` ช่วยให้เราไม่ต้องเขียนโค้ดเพิ่มเติมเพื่อ sync ค่าที่เปลี่ยนไปในทุกตารางลูกเอง — PostgreSQL จัดการให้อัตโนมัติภายใน transaction เดียว

### เปรียบเทียบ ON UPDATE แต่ละแบบโดยย่อ

| Action | พฤติกรรมเมื่อ UPDATE key แม่ |
|---|---|
| `CASCADE` | อัปเดตค่า FK ในตารางลูกตามไปด้วยอัตโนมัติ |
| `RESTRICT` | ปฏิเสธการ UPDATE ถ้ายังมีลูกอ้างอิงอยู่ |
| `SET NULL` | ตั้งค่า FK ของลูกเป็น NULL |
| `SET DEFAULT` | ตั้งค่า FK ของลูกเป็นค่า default |
| `NO ACTION` (default) | ปฏิเสธการ UPDATE เหมือน RESTRICT (ในกรณีทั่วไป) |

### คำแนะนำ

- ถ้า PK เป็น `SERIAL`/`IDENTITY`/UUID ที่ไม่เปลี่ยนแปลง ไม่จำเป็นต้องระบุ `ON UPDATE` เลย (ปล่อยเป็น default `NO ACTION` ก็เพียงพอ)
- ถ้าออกแบบระบบด้วย natural key ที่มีโอกาสเปลี่ยนแปลง ควรพิจารณาใช้ `ON UPDATE CASCADE` เพื่อความสะดวกและป้องกันข้อมูลไม่สอดคล้องกัน
- โดยรวมแนะนำให้ใช้ surrogate key (`SERIAL`/`IDENTITY`) เป็น PK เสมอเมื่อเป็นไปได้ เพื่อหลีกเลี่ยงปัญหานี้ตั้งแต่ต้น

ล้างตารางทดสอบทิ้ง:

```sql
DROP TABLE demo_product_natural;
DROP TABLE demo_category_natural;
```

---

## Step 167: NO ACTION vs RESTRICT — ความแตกต่างเชิงเทคนิค (Deferred Constraint Checking)

### ทำไมสองแบบนี้ดูเหมือนกันแต่ไม่เหมือนกัน

ใน Step 165 เราเห็นว่า `NO ACTION` กับ `RESTRICT` ให้ผลลัพธ์เหมือนกันทุกประการในตัวอย่างพื้นฐาน คำถามคือ แล้วทำไม PostgreSQL (และมาตรฐาน SQL) ถึงมีสองคำนี้แยกกัน?

**คำตอบ**: ความแตกต่างอยู่ที่ **เวลาที่ตรวจสอบ constraint**

- `RESTRICT` — ตรวจสอบ**ทันที** (immediately) เสมอ ไม่สามารถ defer (เลื่อน) การตรวจสอบไปตอนท้าย transaction ได้ ไม่ว่าจะตั้งค่าอะไรก็ตาม
- `NO ACTION` — ตรวจสอบทันทีโดย **default** เช่นกัน แต่**สามารถ defer** การตรวจสอบไปจนกว่า transaction จะ `COMMIT` ได้ ถ้า constraint ถูกประกาศเป็น `DEFERRABLE`

### Deferrable Constraint คืออะไร

โดย default constraint ทุกชนิดใน PostgreSQL เป็น `NOT DEFERRABLE INITIALLY IMMEDIATE` คือตรวจสอบทันทีหลังแต่ละคำสั่ง SQL แต่เราสามารถประกาศให้ FK เป็น `DEFERRABLE` ได้ ซึ่งจะเปิดโอกาสให้เลื่อนการตรวจสอบไปจนจบ transaction — ฟีเจอร์นี้ใช้ได้กับ `NO ACTION` เท่านั้น **ใช้กับ `RESTRICT` ไม่ได้**

### ตัวอย่างที่แสดงความแตกต่างชัดเจน

ลองจินตนาการสถานการณ์: มีตาราง A และ B ที่อ้างอิงกันแบบวนลูป (circular reference แบบง่าย) — เราต้องการ DELETE แล้ว INSERT แถวใหม่ใน transaction เดียว โดยที่ระหว่างทางข้อมูลจะ "ผิดกฎ FK ชั่วคราว" แต่ถูกต้องตอนจบ transaction

```sql
-- สร้างตารางทดสอบสำหรับ NO ACTION แบบ DEFERRABLE
CREATE TABLE demo_parent (
    id   INTEGER PRIMARY KEY,
    name VARCHAR(50)
);

CREATE TABLE demo_child_deferrable (
    id        INTEGER PRIMARY KEY,
    parent_id INTEGER REFERENCES demo_parent(id)
        ON DELETE NO ACTION
        DEFERRABLE INITIALLY DEFERRED,
    name      VARCHAR(50)
);

INSERT INTO demo_parent VALUES (1, 'แม่คนที่ 1');
INSERT INTO demo_child_deferrable VALUES (1, 1, 'ลูกคนที่ 1');
```

ทดสอบ: ลบแม่ก่อน แล้วค่อยเพิ่มแม่คนใหม่ที่มี id เดิมกลับมา ภายใน transaction เดียว:

```sql
BEGIN;

DELETE FROM demo_parent WHERE id = 1;
-- ปกติจะ error ทันทีถ้าเป็น RESTRICT หรือ NO ACTION แบบ immediate
-- แต่เพราะเราตั้งเป็น DEFERRABLE INITIALLY DEFERRED จึงยังไม่ตรวจสอบตอนนี้

INSERT INTO demo_parent VALUES (1, 'แม่คนที่ 1 (คนใหม่)');
-- ตอนนี้ id=1 กลับมามีอยู่จริงแล้ว ความสัมพันธ์กลับมาถูกต้อง

COMMIT;
-- PostgreSQL ตรวจสอบ constraint ตอน COMMIT พบว่าถูกต้อง จึงสำเร็จ
```

ผลลัพธ์:

```
BEGIN
DELETE 1
INSERT 0 1
COMMIT
```

Transaction สำเร็จ! เพราะ ณ เวลาที่ `COMMIT` ข้อมูลถูกต้องครบถ้วน (มี `demo_parent.id = 1` อยู่จริง) แม้ระหว่างทางในตอน DELETE จะดูเหมือนผิดกฎก็ตาม

ลองเปรียบเทียบกับกรณีที่ใช้ `RESTRICT` แทน:

```sql
CREATE TABLE demo_child_restrict (
    id        INTEGER PRIMARY KEY,
    parent_id INTEGER REFERENCES demo_parent(id) ON DELETE RESTRICT,
    name      VARCHAR(50)
);

INSERT INTO demo_child_restrict VALUES (1, 1, 'ลูกคนที่ 1 (restrict)');

BEGIN;

DELETE FROM demo_parent WHERE id = 1;
```

ผลลัพธ์ (error ทันที ไม่รอถึง COMMIT):

```
ERROR:  update or delete on table "demo_parent" violates foreign key constraint "demo_child_restrict_parent_id_fkey" on table "demo_child_restrict"
DETAIL:  Key (id)=(1) is still referenced from table "demo_child_restrict".
```

```sql
ROLLBACK;
```

`RESTRICT` ตรวจสอบทันทีเสมอ ไม่สนใจว่าจะมีการแก้ไขให้ถูกต้องอีกครั้งใน transaction เดียวกันหรือไม่ — นี่คือความแตกต่างที่แท้จริงระหว่างสองคำนี้

### สรุปตัวเลือกของ Deferrable

| การประกาศ | ความหมาย |
|---|---|
| `NOT DEFERRABLE` (default) | ตรวจสอบทันทีเสมอ ไม่สามารถเปลี่ยนแปลงได้ |
| `DEFERRABLE INITIALLY IMMEDIATE` | ค่าเริ่มต้นคือตรวจสอบทันที แต่สามารถสั่ง `SET CONSTRAINTS ... DEFERRED` เพื่อเลื่อนได้ในแต่ละ transaction |
| `DEFERRABLE INITIALLY DEFERRED` | เลื่อนการตรวจสอบไปจนถึง COMMIT เสมอ (เว้นแต่สั่ง `SET CONSTRAINTS ... IMMEDIATE`) |

ตัวอย่างการสั่งเปลี่ยนโหมดกลางทาง (สำหรับ constraint ที่เป็น `DEFERRABLE INITIALLY IMMEDIATE`):

```sql
BEGIN;
SET CONSTRAINTS ALL DEFERRED;
-- ... คำสั่งที่อาจผิดกฎชั่วคราว ...
COMMIT;
```

### เมื่อไหร่ควรใช้ Deferrable Constraint

- เมื่อต้องการ "สลับค่า" ระหว่างสองแถวที่มี FK อ้างอิงกันเอง (เช่น สลับตำแหน่งใน hierarchy)
- เมื่อ bulk load ข้อมูลที่มีความสัมพันธ์แบบวนลูปกัน ต้อง insert ข้อมูลก่อนแล้วค่อยอัปเดต FK ทีหลังภายใน transaction เดียว
- ในงานส่วนใหญ่ **ไม่จำเป็นต้องใช้** deferrable — ใช้ default (`NOT DEFERRABLE`, ตรวจสอบทันที) ก็เพียงพอและปลอดภัยกว่า เพราะจะจับข้อผิดพลาดได้เร็วที่สุด

ล้างตารางทดสอบทิ้ง:

```sql
DROP TABLE demo_child_restrict;
DROP TABLE demo_child_deferrable;
DROP TABLE demo_parent;
```

---

## Step 168: Self-referencing Foreign Key

### แนวคิด

**Self-referencing Foreign Key** คือ FK ที่อ้างอิงกลับไปยัง **ตารางเดียวกัน** ใช้สำหรับสร้างโครงสร้างแบบลำดับชั้น (hierarchy) เช่น โครงสร้างองค์กร (พนักงาน-หัวหน้า), หมวดหมู่ซ้อนหมวดหมู่ย่อย (category-subcategory), หรือความคิดเห็นตอบกลับความคิดเห็น (comment-reply)

### ตัวอย่าง: ตารางพนักงานกับหัวหน้างาน

```sql
CREATE TABLE employees (
    employee_id   SERIAL PRIMARY KEY,
    full_name     VARCHAR(100) NOT NULL,
    position      VARCHAR(50) NOT NULL,
    manager_id    INTEGER REFERENCES employees(employee_id) ON DELETE SET NULL
);
```

สังเกตว่า `manager_id` อ้างอิงไปยัง `employees.employee_id` ซึ่งเป็นตารางเดียวกันกับตัวมันเอง

เพิ่มข้อมูลตัวอย่าง (ผู้จัดการใหญ่ไม่มี `manager_id` จึงเป็น NULL):

```sql
-- ผู้บริหารสูงสุด (ไม่มีหัวหน้า)
INSERT INTO employees (full_name, position, manager_id) VALUES
    ('คุณประยุทธ์ ผู้จัดการใหญ่', 'CEO', NULL);

-- ผู้จัดการสาขา (หัวหน้าคือ CEO ซึ่งมี employee_id = 1)
INSERT INTO employees (full_name, position, manager_id) VALUES
    ('คุณมานี ผู้จัดการสาขา', 'Store Manager', 1);

-- พนักงานบาริสต้า (หัวหน้าคือผู้จัดการสาขา ซึ่งมี employee_id = 2)
INSERT INTO employees (full_name, position, manager_id) VALUES
    ('คุณสมศรี บาริสต้า', 'Barista', 2),
    ('คุณสมปอง บาริสต้า', 'Barista', 2);
```

ตรวจสอบโครงสร้าง:

```sql
SELECT employee_id, full_name, position, manager_id FROM employees ORDER BY employee_id;
```

```
 employee_id |         full_name          |   position    | manager_id
-------------+-----------------------------+----------------+------------
           1 | คุณประยุทธ์ ผู้จัดการใหญ่   | CEO            |
           2 | คุณมานี ผู้จัดการสาขา       | Store Manager  |          1
           3 | คุณสมศรี บาริสต้า           | Barista        |          2
           4 | คุณสมปอง บาริสต้า           | Barista        |          2
(4 rows)
```

### แสดงโครงสร้างองค์กรด้วย Self-Join

```sql
SELECT
    e.full_name  AS employee_name,
    e.position   AS employee_position,
    m.full_name  AS manager_name
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.employee_id
ORDER BY e.employee_id;
```

```
       employee_name        | employee_position |        manager_name
-----------------------------+--------------------+-----------------------------
 คุณประยุทธ์ ผู้จัดการใหญ่   | CEO                |
 คุณมานี ผู้จัดการสาขา       | Store Manager      | คุณประยุทธ์ ผู้จัดการใหญ่
 คุณสมศรี บาริสต้า           | Barista            | คุณมานี ผู้จัดการสาขา
 คุณสมปอง บาริสต้า           | Barista            | คุณมานี ผู้จัดการสาขา
(4 rows)
```

### ทดสอบพฤติกรรม ON DELETE SET NULL กับ Self-reference

ลองลบ "คุณมานี ผู้จัดการสาขา" (`employee_id = 2`) ที่มีลูกน้องอ้างอิงอยู่:

```sql
DELETE FROM employees WHERE employee_id = 2;
```

```
DELETE 1
```

ตรวจสอบผลลัพธ์ — ลูกน้องยังอยู่ แต่ `manager_id` กลายเป็น NULL (ไม่มีหัวหน้าชั่วคราว):

```sql
SELECT employee_id, full_name, position, manager_id FROM employees ORDER BY employee_id;
```

```
 employee_id |        full_name        |  position | manager_id
-------------+--------------------------+------------+------------
           1 | คุณประยุทธ์ ผู้จัดการใหญ่ | CEO        |
           3 | คุณสมศรี บาริสต้า        | Barista    |
           4 | คุณสมปอง บาริสต้า        | Barista    |
(3 rows)
```

### ข้อควรระวัง: ห้ามใช้ CASCADE กับ Self-reference แบบไม่ระวัง

ถ้าใช้ `ON DELETE CASCADE` กับ self-referencing FK แบบไม่ระวัง การลบโหนดระดับบนสุดจะทำให้ **ลบทั้งสายบังคับบัญชา** ทันที ซึ่งมักไม่ใช่พฤติกรรมที่ต้องการในระบบพนักงาน (แต่อาจเหมาะกับกรณีอื่น เช่น เธรดความคิดเห็นที่ลบ comment แม่แล้วอยากลบ reply ทั้งหมดตามไปด้วย)

ตัวอย่างกรณีที่ CASCADE เหมาะสม — ระบบความคิดเห็นตอบกลับ:

```sql
CREATE TABLE comments (
    comment_id  SERIAL PRIMARY KEY,
    parent_id   INTEGER REFERENCES comments(comment_id) ON DELETE CASCADE,
    content     TEXT NOT NULL
);

INSERT INTO comments (parent_id, content) VALUES (NULL, 'กาแฟร้านนี้อร่อยมาก');
INSERT INTO comments (parent_id, content) VALUES (1, 'เห็นด้วยเลยครับ');
INSERT INTO comments (parent_id, content) VALUES (1, 'ผมก็ชอบเหมือนกัน');
INSERT INTO comments (parent_id, content) VALUES (2, 'ลองแก้วไหนดีครับ');
```

ลบความคิดเห็นต้นทาง (`comment_id = 1`):

```sql
DELETE FROM comments WHERE comment_id = 1;
```

```
DELETE 1
```

ตรวจสอบผลลัพธ์ — comment ทั้งสายที่ตอบกลับ (รวมถึงการตอบกลับซ้อนกันหลายชั้น) ถูกลบตามไปด้วยทั้งหมด:

```sql
SELECT * FROM comments;
```

```
 comment_id | parent_id | content
------------+-----------+---------
(0 rows)
```

นี่คือพลังของ `CASCADE` กับ self-reference — ลบได้ลึกหลายชั้น (`comment_id = 4` ที่ตอบกลับ `comment_id = 2` ซึ่งตอบกลับ `comment_id = 1` ก็ถูกลบตามไปด้วย) เพราะ CASCADE ทำงานแบบ **recursive** ตามธรรมชาติของ FK chain

ล้างตารางทดสอบทิ้ง:

```sql
DROP TABLE comments;
DROP TABLE employees;
```

---

## Step 169: การปิด/เปิด FK ชั่วคราวสำหรับ Bulk Load

### ทำไมบางครั้งต้องปิด FK ชั่วคราว

เมื่อต้อง import ข้อมูลจำนวนมาก (bulk load) จากระบบเก่า หรือจากไฟล์ CSV ขนาดใหญ่ บางครั้งลำดับข้อมูลอาจไม่เรียงตามความสัมพันธ์ FK (เช่น ข้อมูล `order_items` มาก่อน `orders` ในไฟล์) การตรวจสอบ FK ทุกแถวระหว่าง INSERT จะทำให้ช้าลงมาก และอาจ error กลางทางถ้าลำดับข้อมูลไม่ตรง

PostgreSQL มีสองวิธีหลักในการจัดการสถานการณ์นี้:

### วิธีที่ 1: Deferred Constraint (ปลอดภัยกว่า แนะนำ)

ถ้า FK ถูกประกาศเป็น `DEFERRABLE` ไว้ตั้งแต่ตอนสร้าง เราสามารถสั่งเลื่อนการตรวจสอบไปจนจบ transaction ได้ตามที่อธิบายใน Step 167:

```sql
BEGIN;
SET CONSTRAINTS ALL DEFERRED;

-- โหลดข้อมูลได้โดยไม่สนลำดับ เพราะ constraint ยังไม่ถูกตรวจสอบ
-- ... bulk INSERT จำนวนมาก ...

COMMIT;  -- ตรวจสอบ FK ทั้งหมดตอนนี้ครั้งเดียว
```

วิธีนี้ปลอดภัยที่สุด เพราะ PostgreSQL ยังคงรับประกัน Referential Integrity ณ จุดสิ้นสุด transaction เสมอ ถ้าข้อมูลผิดพลาดจริง COMMIT จะ error และ **rollback ข้อมูลทั้งหมดโดยอัตโนมัติ**

### วิธีที่ 2: DISABLE/ENABLE TRIGGER ALL (อันตรายกว่า ต้องระวังมาก)

FK constraint ใน PostgreSQL ถูก implement ด้วยกลไก **internal trigger** เบื้องหลัง เราจึงสามารถ "ปิด" การตรวจสอบ FK ได้โดยการปิด trigger ทั้งหมดของตาราง:

```sql
-- สร้างตารางทดสอบเพื่อสาธิต (จำลองสถานการณ์ bulk load ที่ลำดับข้อมูลผิด)
CREATE TABLE demo_orders_bulk (
    order_id    INTEGER PRIMARY KEY,
    customer_id INTEGER REFERENCES customers(customer_id)
);

CREATE TABLE demo_items_bulk (
    item_id   SERIAL PRIMARY KEY,
    order_id  INTEGER REFERENCES demo_orders_bulk(order_id),
    product_name VARCHAR(100)
);
```

สมมติไฟล์ข้อมูลมี `demo_items_bulk` มาก่อน `demo_orders_bulk` (ลำดับผิดปกติ) — ถ้า insert ตรง ๆ จะ error:

```sql
INSERT INTO demo_items_bulk (order_id, product_name) VALUES (500, 'เอสเพรสโซ่');
```

```
ERROR:  insert or update on table "demo_items_bulk" violates foreign key constraint "demo_items_bulk_order_id_fkey"
DETAIL:  Key (order_id)=(500) is not present in table "demo_orders_bulk".
```

ปิด trigger (ปิด FK check) ชั่วคราวเพื่อ bulk load:

```sql
ALTER TABLE demo_items_bulk DISABLE TRIGGER ALL;
ALTER TABLE demo_orders_bulk DISABLE TRIGGER ALL;

-- ตอนนี้ insert ได้โดยไม่เรียงลำดับ แม้ order_id จะยังไม่มีอยู่จริงก็ตาม
INSERT INTO demo_items_bulk (order_id, product_name) VALUES (500, 'เอสเพรสโซ่');
INSERT INTO demo_items_bulk (order_id, product_name) VALUES (500, 'ลาเต้เย็น');
INSERT INTO demo_orders_bulk (order_id, customer_id) VALUES (500, 1);
```

ผลลัพธ์:

```
ALTER TABLE
ALTER TABLE
INSERT 0 1
INSERT 0 1
INSERT 0 1
```

เปิด trigger กลับคืนหลัง load เสร็จ:

```sql
ALTER TABLE demo_items_bulk ENABLE TRIGGER ALL;
ALTER TABLE demo_orders_bulk ENABLE TRIGGER ALL;
```

```
ALTER TABLE
ALTER TABLE
```

### อันตรายสำคัญของ DISABLE TRIGGER ALL

**คำเตือนที่ต้องรู้**: `ENABLE TRIGGER ALL` **ไม่ได้** ตรวจสอบข้อมูลที่มีอยู่แล้วย้อนหลังโดยอัตโนมัติ! มันแค่เปิดการตรวจสอบสำหรับ INSERT/UPDATE/DELETE **ครั้งถัดไป** เท่านั้น ข้อมูลที่ผิดพลาดซึ่งถูกใส่เข้าไประหว่างปิด trigger จะยังคงอยู่ในระบบโดยไม่มีใครรู้ตัว

ลองพิสูจน์ด้วยตัวอย่าง — ปิด trigger แล้วใส่ข้อมูลที่อ้างอิงถึง order ที่ไม่มีอยู่จริงเลย:

```sql
ALTER TABLE demo_items_bulk DISABLE TRIGGER ALL;

INSERT INTO demo_items_bulk (order_id, product_name) VALUES (99999, 'ข้อมูลผิดพลาด');

ALTER TABLE demo_items_bulk ENABLE TRIGGER ALL;

-- ตรวจสอบ: ข้อมูลผิดพลาดนี้ยังอยู่ในระบบ ไม่มีการแจ้งเตือนใดๆ
SELECT * FROM demo_items_bulk WHERE order_id = 99999;
```

ผลลัพธ์:

```
 item_id | order_id |    product_name
---------+----------+---------------------
       3 |    99999 | ข้อมูลผิดพลาด
(1 row)
```

ข้อมูล "ขยะ" (orphan record) นี้จะอยู่ในระบบต่อไปเงียบ ๆ จนกว่าจะมีใครมาสังเกตเห็นความผิดปกติ (เช่น ตอนทำรายงานแล้ว JOIN ไม่เจอ)

### วิธีตรวจสอบข้อมูลย้อนหลังหลังใช้ DISABLE TRIGGER

หากจำเป็นต้องใช้วิธี DISABLE TRIGGER ควร **ตรวจสอบข้อมูลด้วยตนเอง** ทันทีหลังโหลดเสร็จ:

```sql
-- หา orphan records ที่ order_id ไม่มีอยู่จริงในตารางแม่
SELECT dib.item_id, dib.order_id, dib.product_name
FROM demo_items_bulk dib
LEFT JOIN demo_orders_bulk dob ON dib.order_id = dob.order_id
WHERE dob.order_id IS NULL;
```

ผลลัพธ์:

```
 item_id | order_id |    product_name
---------+----------+---------------------
       3 |    99999 | ข้อมูลผิดพลาด
(1 row)
```

พบข้อมูลผิดพลาด 1 แถว ต้องลบหรือแก้ไขก่อนใช้งานระบบจริง:

```sql
DELETE FROM demo_items_bulk WHERE order_id = 99999;
```

```
DELETE 1
```

### เปรียบเทียบสองวิธี

| วิธี | ความปลอดภัย | การตรวจสอบข้อมูลตอนจบ | เหมาะกับ |
|---|---|---|---|
| Deferred Constraint (`SET CONSTRAINTS ALL DEFERRED`) | ปลอดภัย — PostgreSQL ตรวจสอบให้อัตโนมัติตอน COMMIT | อัตโนมัติ, ล้มเหลว = rollback ทั้งหมด | Bulk load ภายใน transaction เดียว ที่ FK ถูกประกาศ DEFERRABLE ไว้แล้ว |
| `DISABLE/ENABLE TRIGGER ALL` | อันตราย — ไม่มีการตรวจสอบอัตโนมัติเลย | ต้องเขียน query ตรวจสอบเอง | กรณีจำเป็นจริง ๆ เช่น restore ข้อมูลจาก backup ที่มั่นใจว่าถูกต้อง หรือ migration ขนาดใหญ่ที่มีการตรวจสอบคุณภาพข้อมูลเป็นขั้นตอนแยกอยู่แล้ว |

> **คำแนะนำจากมืออาชีพ**: ควรใช้ `DISABLE TRIGGER ALL` เป็นทางเลือกสุดท้ายเท่านั้น และต้องมีขั้นตอนตรวจสอบ orphan records เสมอหลังใช้งาน นอกจากนี้ `DISABLE TRIGGER ALL` ยังปิด trigger อื่น ๆ ทั้งหมดของตารางด้วย (ไม่ใช่แค่ FK trigger) รวมถึง trigger ที่ผู้ใช้สร้างเอง เช่น audit log trigger — ดังนั้นควรพิจารณาใช้ `DISABLE TRIGGER USER` แทนถ้าต้องการปิดเฉพาะ trigger ที่ผู้ใช้สร้าง (ไม่ปิด FK/PK internal trigger) หรือระบุชื่อ trigger เฉพาะเจาะจงแทนการใช้ `ALL`

ล้างตารางทดสอบทิ้ง:

```sql
DROP TABLE demo_items_bulk;
DROP TABLE demo_orders_bulk;
```

---

## สรุปท้ายบท

ใน Part นี้เราได้เรียนรู้เรื่อง Foreign Key และ Referential Integrity อย่างละเอียด ตั้งแต่แนวคิดพื้นฐานไปจนถึงเทคนิคระดับมืออาชีพ สรุปประเด็นสำคัญได้ดังนี้:

1. **Foreign Key** คือกลไกที่ PostgreSQL ใช้รักษา Referential Integrity ระหว่างตาราง ป้องกันไม่ให้เกิด orphan records
2. FK กำหนดได้ทั้งตอน `CREATE TABLE` (แบบ column constraint หรือ table constraint) และเพิ่มทีหลังด้วย `ALTER TABLE ... ADD CONSTRAINT` โดยมีเทคนิค `NOT VALID` + `VALIDATE CONSTRAINT` ช่วยลด lock time ในตารางขนาดใหญ่
3. **Composite FK** ใช้เมื่อ Primary Key ของตารางแม่เป็นหลายคอลัมน์รวมกัน ต้องอ้างอิงทุกคอลัมน์พร้อมกันเสมอ
4. Error สองแบบหลักคือ `insert or update violates foreign key constraint` (ตารางลูกอ้างอิงค่าที่ไม่มีจริง) และ `update or delete violates foreign key constraint` (ตารางแม่ถูกลบ/แก้ไขทั้งที่ยังมีลูกอ้างอิงอยู่)
5. **ON DELETE** มี 5 แบบ: `CASCADE` (ลบตามกัน), `RESTRICT` (ห้ามลบ), `SET NULL` (ตั้งเป็น NULL), `SET DEFAULT` (ตั้งเป็นค่า default), `NO ACTION` (ค่า default ของระบบ)
6. **ON UPDATE** ทำงานคล้าย ON DELETE แต่ทำงานเมื่อ Primary Key ถูกแก้ไข — ใช้น้อยเพราะ PK ควรไม่เปลี่ยนแปลง แต่มีประโยชน์กับ natural key
7. **NO ACTION vs RESTRICT** ต่างกันที่ deferred checking — `RESTRICT` ตรวจสอบทันทีเสมอ ส่วน `NO ACTION` สามารถเลื่อนตรวจสอบไปจนจบ transaction ได้ถ้าประกาศเป็น `DEFERRABLE`
8. **Self-referencing FK** ใช้สร้างโครงสร้างลำดับชั้น เช่น employee-manager หรือ comment-reply โดยอ้างอิงกลับมาที่ตารางตัวเอง
9. การปิด FK ชั่วคราวด้วย `DISABLE/ENABLE TRIGGER ALL` มีประโยชน์สำหรับ bulk load แต่อันตรายมากเพราะไม่ตรวจสอบข้อมูลย้อนหลังให้อัตโนมัติ ควรใช้ deferred constraint แทนเมื่อเป็นไปได้

Foreign Key เป็นหนึ่งในเครื่องมือสำคัญที่สุดของฐานข้อมูลเชิงสัมพันธ์ การเลือกนโยบาย `ON DELETE`/`ON UPDATE` ที่เหมาะสมสะท้อนความเข้าใจ business logic ของระบบอย่างแท้จริง ไม่ใช่แค่เรื่องทางเทคนิค — ผู้ออกแบบฐานข้อมูลที่ดีต้องคิดเสมอว่า "ถ้าข้อมูลนี้ถูกลบ ข้อมูลที่เกี่ยวข้องควรเกิดอะไรขึ้น" ก่อนเลือก action ที่จะใช้

ใน Part ถัดไปเราจะเรียนรู้เรื่อง CHECK constraint และ DEFAULT value เพื่อเสริมการควบคุมคุณภาพข้อมูลในระดับคอลัมน์ให้ครบถ้วนยิ่งขึ้น

**อ่านต่อ**: [Part 018: CHECK Constraint และ DEFAULT Value](./part-018-check-default.md)

---

## แบบฝึกหัด

<details>
<summary><b>แบบฝึกหัดที่ 1:</b> จงอธิบายความแตกต่างระหว่าง Foreign Key กับ Primary Key ว่าแตกต่างกันอย่างไร และตารางหนึ่งสามารถมี Foreign Key ได้กี่ตัว</summary>

**เฉลย:**

- **Primary Key (PK)**: คือคอลัมน์ (หรือกลุ่มคอลัมน์) ที่ใช้ระบุแถวข้อมูลแต่ละแถวในตารางนั้นให้ไม่ซ้ำกัน (unique) และห้ามเป็น NULL ตารางหนึ่งมี PK ได้เพียง **1 ตัวเท่านั้น**
- **Foreign Key (FK)**: คือคอลัมน์ที่อ้างอิงไปยัง PK หรือ Unique Key ของอีกตาราง (หรือตารางเดียวกันในกรณี self-reference) ใช้รักษา Referential Integrity ระหว่างตาราง FK สามารถเป็น NULL ได้ (ถ้าไม่ได้กำหนด NOT NULL)
- ตารางหนึ่งสามารถมี FK ได้**หลายตัว** เช่น ตาราง `order_items` มีทั้ง FK ไปที่ `orders` และ FK ไปที่ `products` พร้อมกัน

```sql
\d order_items
-- Foreign-key constraints:
--     "order_items_order_id_fkey" FOREIGN KEY (order_id) REFERENCES orders(order_id)
--     "order_items_product_id_fkey" FOREIGN KEY (product_id) REFERENCES products(product_id)
```

</details>

<details>
<summary><b>แบบฝึกหัดที่ 2:</b> จงเขียนคำสั่ง CREATE TABLE สำหรับตาราง <code>reviews</code> (รีวิวสินค้า) ที่มีคอลัมน์ <code>review_id</code>, <code>product_id</code> (FK ไปที่ <code>products</code>), <code>customer_id</code> (FK ไปที่ <code>customers</code>), <code>rating</code> (1-5), <code>comment</code> โดยตั้งชื่อ constraint เอง</summary>

**เฉลย:**

```sql
CREATE TABLE reviews (
    review_id   SERIAL PRIMARY KEY,
    product_id  INTEGER NOT NULL,
    customer_id INTEGER NOT NULL,
    rating      INTEGER NOT NULL CHECK (rating BETWEEN 1 AND 5),
    comment     TEXT,
    CONSTRAINT fk_reviews_product
        FOREIGN KEY (product_id) REFERENCES products(product_id),
    CONSTRAINT fk_reviews_customer
        FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
);
```

ทดสอบ:

```sql
INSERT INTO reviews (product_id, customer_id, rating, comment)
VALUES (1, 1, 5, 'อร่อยมากครับ');
-- INSERT 0 1

DROP TABLE reviews;
```

</details>

<details>
<summary><b>แบบฝึกหัดที่ 3:</b> ตาราง <code>orders</code> ที่มีอยู่แล้วยังไม่มี FK ไปที่ <code>customers</code> (สมมติสถานการณ์) จงเขียนคำสั่งเพิ่ม FK ทีหลังด้วย ALTER TABLE โดยตั้งชื่อ constraint ว่า <code>fk_orders_customer_v2</code></summary>

**เฉลย:**

```sql
ALTER TABLE orders
    ADD CONSTRAINT fk_orders_customer_v2
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id);
```

หมายเหตุ: ในสคีมาจริงของเรา `orders.customer_id` มี FK อยู่แล้วชื่อ `orders_customer_id_fkey` ตั้งแต่ตอนสร้างตาราง ถ้ารันคำสั่งข้างต้นจริงจะได้ FK สองตัวที่ทำหน้าที่เดียวกัน ในทางปฏิบัติควรตรวจสอบด้วย `\d orders` ก่อนเสมอว่ามี FK อยู่แล้วหรือไม่ เพื่อไม่ให้ซ้ำซ้อนโดยไม่จำเป็น

</details>

<details>
<summary><b>แบบฝึกหัดที่ 4:</b> จงเขียนคำสั่งสร้างตาราง <code>student_courses</code> ที่มี Composite FK อ้างอิงไปที่ตาราง <code>course_offerings</code> ซึ่งมี Composite Primary Key เป็น <code>(course_code, semester)</code></summary>

**เฉลย:**

```sql
CREATE TABLE course_offerings (
    course_code VARCHAR(10) NOT NULL,
    semester    VARCHAR(10) NOT NULL,
    instructor  VARCHAR(100),
    PRIMARY KEY (course_code, semester)
);

CREATE TABLE student_courses (
    id          SERIAL PRIMARY KEY,
    student_id  INTEGER NOT NULL,
    course_code VARCHAR(10) NOT NULL,
    semester    VARCHAR(10) NOT NULL,
    CONSTRAINT fk_student_courses_offering
        FOREIGN KEY (course_code, semester)
        REFERENCES course_offerings (course_code, semester)
);

DROP TABLE student_courses;
DROP TABLE course_offerings;
```

</details>

<details>
<summary><b>แบบฝึกหัดที่ 5:</b> พยายามรันคำสั่งนี้กับสคีมาร้านกาแฟของเรา แล้วอธิบายว่าทำไม error เกิดขึ้น พร้อมวิธีแก้:
<pre>INSERT INTO order_items (order_id, product_id, quantity, unit_price)
VALUES (500, 1, 1, 45.00);</pre></summary>

**เฉลย:**

Error ที่เกิดขึ้น:

```
ERROR:  insert or update on table "order_items" violates foreign key constraint "order_items_order_id_fkey"
DETAIL:  Key (order_id)=(500) is not present in table "orders".
```

สาเหตุ: `order_id = 500` ไม่มีอยู่จริงในตาราง `orders` เพราะสคีมาของเรามีแค่ order_id 1-3 เท่านั้น (`order_items.order_id` เป็น FK ที่ต้องอ้างอิงแถวที่มีอยู่จริง)

วิธีแก้: ตรวจสอบก่อนว่า order นั้นมีอยู่จริงหรือไม่ หรือสร้าง order ใหม่ก่อน insert:

```sql
INSERT INTO orders (customer_id, status) VALUES (2, 'pending') RETURNING order_id;
-- สมมติได้ order_id = 4

INSERT INTO order_items (order_id, product_id, quantity, unit_price)
VALUES (4, 1, 1, 45.00);
-- INSERT 0 1
```

</details>

<details>
<summary><b>แบบฝึกหัดที่ 6:</b> ในระบบร้านกาแฟ ถ้าต้องการให้ "เมื่อลบสินค้า (products) ตัวใดตัวหนึ่ง ห้ามลบถ้ายังมีประวัติการสั่งซื้อ (order_items) อยู่" ควรใช้ ON DELETE แบบไหน จงเขียนคำสั่งตัวอย่าง</summary>

**เฉลย:** ควรใช้ `ON DELETE RESTRICT` (หรือปล่อยเป็นค่า default `NO ACTION` ก็ได้ผลเหมือนกันในทางปฏิบัติทั่วไป) เพราะประวัติการสั่งซื้อเป็นข้อมูลสำคัญทางบัญชี/การเงินที่ห้ามสูญหายไปโดยไม่ตั้งใจ

```sql
ALTER TABLE order_items DROP CONSTRAINT order_items_product_id_fkey;
ALTER TABLE order_items
    ADD CONSTRAINT order_items_product_id_fkey
    FOREIGN KEY (product_id) REFERENCES products(product_id)
    ON DELETE RESTRICT;
```

ด้วยวิธีนี้ ถ้าพยายามลบสินค้าที่เคยถูกสั่งซื้อ ระบบจะปฏิเสธและแจ้ง error ทันที บังคับให้ผู้ใช้ตัดสินใจ (เช่น เปลี่ยนสถานะเป็น `is_active = false` แทนการลบจริง)

</details>

<details>
<summary><b>แบบฝึกหัดที่ 7:</b> จงอธิบายว่าทำไมการใช้ <code>ON DELETE CASCADE</code> กับ <code>customers → orders</code> อาจเป็นความคิดที่อันตราย และเสนอทางเลือกอื่นที่เหมาะสมกว่า</summary>

**เฉลย:**

ถ้าใช้ `ON DELETE CASCADE` กับ `orders.customer_id REFERENCES customers(customer_id)` การลบลูกค้าหนึ่งคนจะทำให้ **ประวัติคำสั่งซื้อทั้งหมด** ของลูกค้าคนนั้นถูกลบไปด้วยทันที ซึ่งอันตรายมากเพราะ:

1. ข้อมูลคำสั่งซื้อเป็นข้อมูลทางบัญชี/ภาษี ที่กฎหมายอาจกำหนดให้ต้องเก็บไว้ระยะเวลาหนึ่ง
2. รายงานยอดขายย้อนหลังจะผิดพลาด เพราะยอดขายของลูกค้าที่ถูกลบจะหายไปจากสถิติทั้งหมด
3. ถ้า `order_items` มี `ON DELETE CASCADE` ต่อจาก `orders` อีกชั้น การลบลูกค้า 1 คนอาจลบข้อมูลนับพันแถวโดยไม่ตั้งใจ (cascade เป็นทอด ๆ)

ทางเลือกที่เหมาะสมกว่า:

- ใช้ **soft delete**: เพิ่มคอลัมน์ `is_active` หรือ `deleted_at` ในตาราง `customers` แทนการลบจริง
- หรือใช้ `ON DELETE RESTRICT`/`NO ACTION` (default) เพื่อบังคับให้ตรวจสอบก่อนลบเสมอ
- ถ้าจำเป็นต้องลบลูกค้าจริง ๆ ตามกฎหมาย (เช่น GDPR) ควรใช้ `ON DELETE SET NULL` แล้วเก็บข้อมูลคำสั่งซื้อไว้ (แค่ไม่ทราบว่าเป็นของใคร) แทนการลบประวัติทั้งหมด

</details>

<details>
<summary><b>แบบฝึกหัดที่ 8:</b> จงสร้างตาราง <code>categories</code> เวอร์ชันใหม่ที่รองรับหมวดหมู่ย่อย (self-referencing) เช่น "เครื่องดื่ม" มีหมวดหมู่ย่อยคือ "กาแฟร้อน" และ "กาแฟเย็น"</summary>

**เฉลย:**

```sql
CREATE TABLE categories_v2 (
    category_id      SERIAL PRIMARY KEY,
    category_name    VARCHAR(50) NOT NULL,
    parent_category_id INTEGER REFERENCES categories_v2(category_id) ON DELETE CASCADE
);

INSERT INTO categories_v2 (category_name, parent_category_id) VALUES
    ('เครื่องดื่ม', NULL);
-- category_id = 1

INSERT INTO categories_v2 (category_name, parent_category_id) VALUES
    ('กาแฟร้อน', 1),
    ('กาแฟเย็น', 1);

SELECT c.category_name AS sub_category, p.category_name AS parent_category
FROM categories_v2 c
LEFT JOIN categories_v2 p ON c.parent_category_id = p.category_id;
```

ผลลัพธ์:

```
 sub_category  | parent_category
----------------+-----------------
 เครื่องดื่ม     |
 กาแฟร้อน        | เครื่องดื่ม
 กาแฟเย็น        | เครื่องดื่ม
(3 rows)
```

```sql
DROP TABLE categories_v2;
```

ใช้ `ON DELETE CASCADE` ในกรณีนี้เพราะสมเหตุสมผลที่ว่า ถ้าหมวดหมู่แม่ "เครื่องดื่ม" ถูกลบ หมวดหมู่ย่อยทั้งหมดที่อยู่ภายใต้มันก็ควรถูกลบตามไปด้วย

</details>

<details>
<summary><b>แบบฝึกหัดที่ 9:</b> จงอธิบายว่า <code>DISABLE TRIGGER ALL</code> ต่างจาก transaction ที่ใช้ <code>SET CONSTRAINTS ALL DEFERRED</code> อย่างไร และแบบไหนปลอดภัยกว่ากัน เพราะอะไร</summary>

**เฉลย:**

| ประเด็น | `DISABLE TRIGGER ALL` | `SET CONSTRAINTS ALL DEFERRED` |
|---|---|---|
| การตรวจสอบ FK | ปิดสนิท ไม่ตรวจสอบเลยจนกว่าจะ ENABLE กลับ | ยังตรวจสอบอยู่ แค่เลื่อนเวลาไปจนถึง COMMIT |
| ความปลอดภัยของข้อมูล | ต่ำ — ข้อมูลผิดพลาดอาจหลุดเข้าระบบถาวรโดยไม่รู้ตัว | สูง — ถ้าข้อมูลผิดพลาดจริง COMMIT จะ error และ rollback ทั้งหมดอัตโนมัติ |
| ขอบเขตผลกระทบ | ปิด trigger ทั้งหมดของตาราง (รวม trigger อื่นที่ไม่เกี่ยวกับ FK เช่น audit log) | จำกัดเฉพาะ constraint เท่านั้น ไม่กระทบ trigger อื่น |
| ต้องตรวจสอบย้อนหลังหรือไม่ | ต้อง เขียน query หา orphan records เอง | ไม่ต้อง PostgreSQL ตรวจสอบให้อัตโนมัติ |

**สรุป**: `SET CONSTRAINTS ALL DEFERRED` ปลอดภัยกว่ามาก เพราะ PostgreSQL ยังคงรับประกัน Referential Integrity ให้เสมอ (แค่เลื่อนเวลาตรวจสอบ) ในขณะที่ `DISABLE TRIGGER ALL` เป็นการปิดการป้องกันทั้งหมดชั่วคราว ควรใช้เป็นทางเลือกสุดท้ายเท่านั้น และต้องมีการตรวจสอบข้อมูลด้วยตนเองหลังใช้งานเสมอ

</details>

<details>
<summary><b>แบบฝึกหัดที่ 10 (แบบฝึกหัดรวม):</b> ออกแบบความสัมพันธ์ทั้งหมดของระบบร้านกาแฟ (categories, products, customers, orders, order_items) พร้อมเลือก <code>ON DELETE</code> ที่เหมาะสมในแต่ละจุด พร้อมให้เหตุผลทางธุรกิจประกอบทุกจุด</summary>

**เฉลย:**

ตารางสรุปการออกแบบ FK ทั้งหมดของระบบร้านกาแฟ พร้อมเหตุผล:

| FK | จาก → ไป | ON DELETE ที่เลือก | เหตุผลทางธุรกิจ |
|---|---|---|---|
| `products.category_id` | products → categories | `RESTRICT` | หมวดหมู่สินค้าเป็นข้อมูลอ้างอิงหลัก ไม่ควรลบได้ถ้ายังมีสินค้าอยู่ในหมวดนั้น บังคับให้ย้ายสินค้าไปหมวดอื่นหรือปิดการขายสินค้าก่อน (ใช้ `is_active = false`) แล้วค่อยลบหมวดหมู่ทีหลัง ป้องกันการลบหมวดหมู่โดยไม่ตั้งใจซึ่งจะทำให้สินค้าไม่มีหมวดหมู่ |
| `orders.customer_id` | orders → customers | `RESTRICT` (หรือ `SET NULL` ถ้าต้องรองรับ GDPR/สิทธิ์ลบข้อมูลส่วนบุคคล) | ประวัติคำสั่งซื้อเป็นข้อมูลบัญชี/ภาษีที่ต้องเก็บไว้ ไม่ควรลบลูกค้าที่มีประวัติสั่งซื้อโดยตรง แนะนำให้ใช้ soft delete (`is_active`) กับลูกค้าแทนการลบจริง ถ้าจำเป็นต้องลบตามกฎหมายจริง ๆ ค่อยพิจารณา `SET NULL` เพื่อรักษายอดขายในรายงานไว้แต่ไม่ระบุตัวตน |
| `order_items.order_id` | order_items → orders | `CASCADE` | รายการสินค้าในบิล (`order_items`) ไม่มีความหมายใด ๆ ถ้าไม่มีบิล (`orders`) แม่อยู่ — เป็นความสัมพันธ์แบบ composition แท้จริง ถ้าจำเป็นต้องลบบิลทั้งใบ (เช่น บิลที่สร้างผิดพลาดและยกเลิก) ควรลบรายการสินค้าที่เกี่ยวข้องตามไปด้วยอัตโนมัติ |
| `order_items.product_id` | order_items → products | `RESTRICT` | ประวัติการขายสินค้าต้องคงอยู่เพื่อการทำรายงานย้อนหลัง (เช่น สินค้าขายดี, ยอดขายรายเดือน) ห้ามลบสินค้าที่เคยมีประวัติการขายแล้ว ถ้าต้องการเลิกขายสินค้า ให้ใช้ `is_active = false` แทนการลบจริงเสมอ |

**คำสั่งสร้างสคีมาฉบับสมบูรณ์ตามการออกแบบนี้:**

```sql
CREATE TABLE categories (
    category_id   SERIAL PRIMARY KEY,
    category_name VARCHAR(50) NOT NULL UNIQUE
);

CREATE TABLE products (
    product_id    SERIAL PRIMARY KEY,
    category_id   INTEGER NOT NULL
        REFERENCES categories(category_id) ON DELETE RESTRICT,
    product_name  VARCHAR(100) NOT NULL,
    price         NUMERIC(10,2) NOT NULL CHECK (price >= 0),
    is_active     BOOLEAN NOT NULL DEFAULT true
);

CREATE TABLE customers (
    customer_id   SERIAL PRIMARY KEY,
    full_name     VARCHAR(100) NOT NULL,
    email         VARCHAR(150) NOT NULL UNIQUE,
    is_active     BOOLEAN NOT NULL DEFAULT true
);

CREATE TABLE orders (
    order_id      SERIAL PRIMARY KEY,
    customer_id   INTEGER
        REFERENCES customers(customer_id) ON DELETE RESTRICT,
    order_date    TIMESTAMPTZ NOT NULL DEFAULT now(),
    status        VARCHAR(20) NOT NULL DEFAULT 'pending'
);

CREATE TABLE order_items (
    order_item_id SERIAL PRIMARY KEY,
    order_id      INTEGER NOT NULL
        REFERENCES orders(order_id) ON DELETE CASCADE,
    product_id    INTEGER NOT NULL
        REFERENCES products(product_id) ON DELETE RESTRICT,
    quantity      INTEGER NOT NULL CHECK (quantity > 0),
    unit_price    NUMERIC(10,2) NOT NULL
);
```

**หลักการสำคัญที่สรุปได้จากแบบฝึกหัดนี้**: การเลือก `ON DELETE` ไม่ใช่เรื่องทางเทคนิคล้วน ๆ แต่สะท้อนกฎทางธุรกิจ (business rule) ของระบบ — ข้อมูลที่เป็น "หลักฐาน" หรือ "ประวัติ" ทางธุรกิจ (เช่น คำสั่งซื้อ, ยอดขาย) ควรป้องกันด้วย `RESTRICT` เสมอ ส่วนข้อมูลที่เป็น "ส่วนประกอบ" ของอีกข้อมูลหนึ่งอย่างแท้จริง (เช่น รายการในบิล) จึงเหมาะกับ `CASCADE` และควรใช้ soft delete (`is_active`, `deleted_at`) แทนการลบจริงสำหรับข้อมูลหลักอย่าง `products` และ `customers` เสมอในระบบธุรกิจจริง

</details>

---

**อ่านต่อ**: [Part 018: CHECK Constraint และ DEFAULT Value](./part-018-check-default.md)
