# Part 009: INSERT — เพิ่มข้อมูลเดี่ยว, หลายแถว, INSERT...SELECT

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับพื้นฐาน | Part 009

---

## เป้าหมายการเรียนรู้

เมื่อจบบทนี้ คุณจะสามารถ:

- เขียนคำสั่ง `INSERT INTO ... VALUES` ได้ทั้งแบบระบุคอลัมน์และไม่ระบุคอลัมน์ พร้อมเข้าใจว่าทำไมควรระบุคอลัมน์เสมอในโค้ดจริง
- เพิ่มข้อมูลหลายแถวพร้อมกันในคำสั่งเดียว (multi-row `INSERT`) และอธิบายได้ว่าทำไมวิธีนี้เร็วกว่าการ `INSERT` ทีละแถว
- ใช้ `DEFAULT VALUES` และคีย์เวิร์ด `DEFAULT` สำหรับคอลัมน์ที่มีค่าเริ่มต้นหรือ auto-generate
- ใช้ `RETURNING` เพื่อดึงค่าที่เพิ่งถูก insert กลับมาโดยไม่ต้อง `SELECT` ซ้ำ
- คัดลอกข้อมูลจากตารางหนึ่งไปอีกตารางหนึ่งด้วย `INSERT ... SELECT`
- ทำ UPSERT ด้วย `ON CONFLICT DO NOTHING` และ `ON CONFLICT DO UPDATE`
- ระบุ conflict target แบบละเอียดด้วย `ON CONFLICT (คอลัมน์)` และ `ON CONFLICT ON CONSTRAINT`
- เลือกวิธี bulk insert ที่เหมาะสมระหว่าง `INSERT` หลายแถวกับ `COPY` และเข้าใจผลกระทบด้าน performance
- อ่านและแก้ error message ที่พบบ่อยตอน insert เช่น constraint violation, type mismatch, NOT NULL violation
- ปฏิบัติตาม best practices ในการ insert ข้อมูลจริง เช่นการห่อ transaction, การกำหนด batch size, และการ validate ข้อมูลก่อน insert

---

## เตรียมข้อมูล

บทนี้ต่อยอดจาก Part 008 ที่เราออกแบบสคีมาร้านกาแฟ (coffee shop) ไว้แล้ว เพื่อให้ตัวอย่างในบทนี้รันได้ทันทีโดยไม่ต้องย้อนกลับไปอ่าน Part 008 เราจะสร้างสคีมาเดิมขึ้นมาใหม่แบบย่อ (schema เดียวกัน ชื่อตาราง/คอลัมน์เดียวกัน) ไว้ในฐานข้อมูลทดสอบของคุณ

```sql
-- ลบตารางเก่าถ้ามี (เผื่อรันซ้ำ) เรียงตามลำดับ dependency
DROP TABLE IF EXISTS order_items CASCADE;
DROP TABLE IF EXISTS orders CASCADE;
DROP TABLE IF EXISTS products CASCADE;
DROP TABLE IF EXISTS customers CASCADE;
DROP TABLE IF EXISTS categories CASCADE;

-- ตารางหมวดหมู่สินค้า
CREATE TABLE categories (
    category_id   SERIAL PRIMARY KEY,
    category_name VARCHAR(50) NOT NULL UNIQUE,
    description   TEXT
);

-- ตารางสินค้า (เมนูเครื่องดื่ม/ขนม)
CREATE TABLE products (
    product_id    SERIAL PRIMARY KEY,
    category_id   INTEGER NOT NULL REFERENCES categories(category_id),
    product_name  VARCHAR(100) NOT NULL,
    sku           VARCHAR(20) NOT NULL UNIQUE,
    price         NUMERIC(10, 2) NOT NULL CHECK (price >= 0),
    is_active     BOOLEAN NOT NULL DEFAULT TRUE,
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ตารางลูกค้า
CREATE TABLE customers (
    customer_id   SERIAL PRIMARY KEY,
    full_name     VARCHAR(100) NOT NULL,
    email         VARCHAR(100) NOT NULL UNIQUE,
    phone         VARCHAR(20),
    loyalty_point INTEGER NOT NULL DEFAULT 0,
    joined_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ตารางออเดอร์
CREATE TABLE orders (
    order_id     SERIAL PRIMARY KEY,
    customer_id  INTEGER REFERENCES customers(customer_id),
    order_status VARCHAR(20) NOT NULL DEFAULT 'pending',
    order_date   TIMESTAMPTZ NOT NULL DEFAULT now(),
    notes        TEXT
);

-- ตารางรายการสินค้าในแต่ละออเดอร์
CREATE TABLE order_items (
    order_item_id SERIAL PRIMARY KEY,
    order_id      INTEGER NOT NULL REFERENCES orders(order_id) ON DELETE CASCADE,
    product_id    INTEGER NOT NULL REFERENCES products(product_id),
    quantity      INTEGER NOT NULL CHECK (quantity > 0),
    unit_price    NUMERIC(10, 2) NOT NULL CHECK (unit_price >= 0),
    UNIQUE (order_id, product_id)
);
```

ตรวจสอบว่าตารางถูกสร้างครบ:

```sql
\dt
```

```
              List of relations
 Schema |     Name     | Type  |  Owner
--------+--------------+-------+----------
 public | categories   | table | postgres
 public | customers    | table | postgres
 public | order_items  | table | postgres
 public | orders       | table | postgres
 public | products     | table | postgres
(5 rows)
```

ทุกตารางยังว่างเปล่า — บทนี้เราจะเติมข้อมูลลงไปทีละขั้นด้วยคำสั่ง `INSERT` ในหลากหลายรูปแบบ

---

## Step 81: INSERT พื้นฐาน — syntax, INSERT INTO ... VALUES แบบระบุคอลัมน์และไม่ระบุ

### รูปแบบ syntax ทั่วไป

```sql
INSERT INTO table_name (column1, column2, column3, ...)
VALUES (value1, value2, value3, ...);
```

องค์ประกอบหลัก:

- `INSERT INTO table_name` — บอกว่าจะเพิ่มข้อมูลลงตารางไหน
- `(column1, column2, ...)` — รายชื่อคอลัมน์ที่จะใส่ค่า (ส่วนนี้เลือกใส่หรือไม่ใส่ก็ได้ แต่ **แนะนำให้ใส่เสมอ**)
- `VALUES (value1, value2, ...)` — ค่าที่จะใส่ โดยต้องเรียงลำดับให้ตรงกับรายชื่อคอลัมน์ด้านบน

### ตัวอย่างที่ 1: INSERT แบบระบุคอลัมน์ (แนะนำ)

เริ่มเพิ่มหมวดหมู่สินค้าในร้านกาแฟ:

```sql
INSERT INTO categories (category_name, description)
VALUES ('เครื่องดื่มร้อน', 'กาแฟ ชา และเครื่องดื่มร้อนอื่น ๆ');
```

```
INSERT 0 1
```

ผลลัพธ์ `INSERT 0 1` อ่านว่า:
- `0` คือ OID ของแถว (ในยุคปัจจุบันมักเป็น 0 เสมอ เพราะ PostgreSQL ไม่ได้ใช้ OID เป็นค่าเริ่มต้นแล้วตั้งแต่เวอร์ชันใหม่ ๆ)
- `1` คือจำนวนแถวที่ถูกเพิ่มสำเร็จ

สังเกตว่าเราไม่ได้ระบุค่าให้ `category_id` เพราะเป็น `SERIAL` (auto-increment) ระบบจะสร้างค่าให้เองโดยอัตโนมัติ

### ตัวอย่างที่ 2: INSERT แบบไม่ระบุคอลัมน์

PostgreSQL อนุญาตให้ละเว้นรายชื่อคอลัมน์ได้ โดยจะยึดตามลำดับคอลัมน์ในตาราง (ตามที่ `CREATE TABLE` นิยามไว้) ทุกคอลัมน์:

```sql
INSERT INTO categories
VALUES (DEFAULT, 'เครื่องดื่มเย็น', 'กาแฟเย็น ชาเย็น สมูทตี้');
```

```
INSERT 0 1
```

ในที่นี้เราต้องใส่ค่าให้ครบทุกคอลัมน์ตามลำดับ (`category_id`, `category_name`, `description`) และใช้คีย์เวิร์ด `DEFAULT` แทนค่า `category_id` เพื่อบอกให้ PostgreSQL ใช้ค่าเริ่มต้น (auto-increment) แทน

> **ทำไมไม่ควรใช้ INSERT แบบไม่ระบุคอลัมน์ในโค้ดจริง**
>
> 1. **เปราะบางต่อการเปลี่ยนแปลงโครงสร้างตาราง** — ถ้าวันหนึ่งมีคนเพิ่มคอลัมน์ใหม่ตรงกลางตาราง (เช่นใช้ `ALTER TABLE ... ADD COLUMN` แล้ว migration เปลี่ยนลำดับ) โค้ดที่ไม่ระบุคอลัมน์จะพังทันทีหรือแย่กว่านั้นคือใส่ค่าผิดคอลัมน์แบบเงียบ ๆ
> 2. **อ่านยาก** — คนอ่านโค้ดต้องไปเปิดดู schema ตารางเพื่อรู้ว่าค่าตัวที่ 3 คือคอลัมน์อะไร
> 3. **เสี่ยงต่อ type mismatch ที่ตรวจจับยาก** — ถ้าลำดับคอลัมน์สลับกัน ค่าที่ควรเป็น string อาจไปตกในคอลัมน์ตัวเลข ทำให้เกิด error หรือแย่กว่านั้นคือแปลงชนิดข้อมูลแบบผิดเพี้ยนโดยไม่รู้ตัว

ดังนั้นในบทนี้และบทต่อ ๆ ไป เราจะระบุชื่อคอลัมน์ทุกครั้ง

### ตัวอย่างที่ 3: เพิ่มสินค้าชิ้นแรก

```sql
INSERT INTO products (category_id, product_name, sku, price)
VALUES (1, 'Espresso', 'BEV-ESP-001', 55.00);
```

```
INSERT 0 1
```

ที่นี่เราไม่ได้ใส่ค่าให้ `is_active` และ `created_at` เพราะทั้งสองคอลัมน์มี `DEFAULT` กำหนดไว้ตั้งแต่ตอน `CREATE TABLE` (`DEFAULT TRUE` และ `DEFAULT now()` ตามลำดับ) — PostgreSQL จะเติมค่าเริ่มต้นให้อัตโนมัติเมื่อเราไม่ระบุคอลัมน์นั้นในคำสั่ง `INSERT`

ตรวจสอบผลลัพธ์:

```sql
SELECT product_id, product_name, price, is_active, created_at FROM products;
```

```
 product_id | product_name | price | is_active |          created_at
------------+--------------+-------+-----------+-------------------------------
          1 | Espresso     | 55.00 | t         | 2026-09-25 10:15:32.442112+00
(1 row)
```

### ตัวอย่างที่ 4: ค่าที่เป็น NULL อย่างชัดเจน

```sql
INSERT INTO customers (full_name, email, phone)
VALUES ('สมชาย ใจดี', 'somchai.jaidee@example.com', NULL);
```

```
INSERT 0 1
```

ใส่ `NULL` ตรง ๆ ได้เมื่อคอลัมน์นั้นอนุญาตให้เป็นค่าว่าง (ในที่นี้ `phone` ไม่มี `NOT NULL` constraint) ต่างจากการ "ไม่ระบุคอลัมน์เลย" ตรงที่ `NULL` คือการบอกอย่างชัดเจนว่า "ค่านี้ไม่มี" ในขณะที่การไม่ระบุคอลัมน์คือการปล่อยให้ระบบใช้ `DEFAULT`

---

## Step 82: INSERT หลายแถวพร้อมกัน (Multi-row VALUES)

### Syntax

PostgreSQL (และ SQL มาตรฐาน) รองรับการใส่หลายชุดค่าใน `VALUES` เดียว โดยคั่นแต่ละแถวด้วยเครื่องหมายจุลภาค:

```sql
INSERT INTO table_name (column1, column2)
VALUES
    (value1a, value2a),
    (value1b, value2b),
    (value1c, value2c);
```

### ตัวอย่าง: เพิ่มหมวดหมู่สินค้าหลายรายการพร้อมกัน

```sql
INSERT INTO categories (category_name, description)
VALUES
    ('เบเกอรี่', 'ขนมปังและเบเกอรี่สดใหม่ทุกวัน'),
    ('ขนมหวาน', 'เค้ก คุกกี้ และของหวานอื่น ๆ'),
    ('อาหารเช้า', 'เซ็ตอาหารเช้าคู่กาแฟ');
```

```
INSERT 0 3
```

ตัวเลข `3` บอกว่าเพิ่มสำเร็จ 3 แถวในคำสั่งเดียว

### ตัวอย่าง: เพิ่มสินค้าหลายรายการพร้อมกัน

```sql
INSERT INTO products (category_id, product_name, sku, price)
VALUES
    (1, 'Americano',     'BEV-AME-001', 50.00),
    (1, 'Cappuccino',    'BEV-CAP-001', 60.00),
    (1, 'Latte',         'BEV-LAT-001', 60.00),
    (2, 'Iced Americano','BEV-IAM-001', 55.00),
    (2, 'Iced Latte',    'BEV-ILA-001', 65.00),
    (4, 'Croissant',     'BAK-CRO-001', 45.00),
    (4, 'Danish',        'BAK-DAN-001', 50.00),
    (5, 'Chocolate Cake','SWT-CHC-001', 85.00);
```

```
INSERT 0 8
```

### ทำไม multi-row INSERT เร็วกว่า INSERT ทีละแถว

ลองเปรียบเทียบสองแนวทางนี้ในทางความคิด:

**แนวทาง A: INSERT ทีละแถว (ช้า)**

```sql
INSERT INTO customers (full_name, email) VALUES ('ลูกค้า A', 'a@example.com');
INSERT INTO customers (full_name, email) VALUES ('ลูกค้า B', 'b@example.com');
INSERT INTO customers (full_name, email) VALUES ('ลูกค้า C', 'c@example.com');
-- ... สมมติมี 10,000 คำสั่งแบบนี้
```

**แนวทาง B: Multi-row INSERT (เร็วกว่ามาก)**

```sql
INSERT INTO customers (full_name, email)
VALUES
    ('ลูกค้า A', 'a@example.com'),
    ('ลูกค้า B', 'b@example.com'),
    ('ลูกค้า C', 'c@example.com');
    -- ... รวมหลายพันแถวในคำสั่งเดียว
```

เหตุผลที่แนวทาง B เร็วกว่ามีหลายชั้น:

1. **ลด network round-trip** — ทุกครั้งที่ client ส่งคำสั่งไปหา PostgreSQL server จะมีค่าใช้จ่ายด้าน network latency (ยิ่งถ้า client กับ server อยู่คนละเครื่อง/คนละ region ยิ่งชัดเจน) การรวมหลายแถวในคำสั่งเดียวทำให้ round-trip ลดจากหลักพัน/หมื่นครั้งเหลือครั้งเดียว
2. **ลดค่าใช้จ่ายในการ parse/plan** — PostgreSQL ต้อง parse SQL, สร้าง execution plan, และเตรียม transaction context ทุกครั้งที่ได้รับคำสั่งใหม่ การส่งเป็นคำสั่งเดียวที่มีหลายแถวทำให้ค่าใช้จ่ายส่วนนี้เกิดขึ้นเพียงครั้งเดียว
3. **WAL (Write-Ahead Log) และ fsync ที่มีประสิทธิภาพกว่า** — ถ้าแต่ละ `INSERT` เป็น transaction เดี่ยว (auto-commit) PostgreSQL ต้อง flush WAL ลงดิสก์ (fsync) ทุกครั้งที่ commit ซึ่งเป็นการ I/O ที่ค่อนข้างแพง การรวมหลายแถวในคำสั่งเดียว (หรือห่อด้วย transaction เดียวตามที่จะกล่าวใน Step 90) ทำให้ fsync เกิดขึ้นน้อยครั้งลงมาก
4. **Index maintenance ที่มีประสิทธิภาพกว่า** — การอัปเดต index หลายแถวพร้อมกันมักทำได้ efficient กว่าการอัปเดตทีละแถวซ้ำ ๆ

โดยประมาณ (ตัวเลขจะแตกต่างกันตาม hardware, network, จำนวน index ฯลฯ) การ insert ข้อมูลหลักหมื่นแถวด้วย multi-row VALUES อาจเร็วกว่าการ insert ทีละแถวแบบ auto-commit ได้หลายสิบเท่าถึงหลักร้อยเท่า

> **ข้อควรระวัง**: อย่ารวมหลายแถวมากเกินไปในคำสั่งเดียว (เช่นเป็นล้านแถวในคำสั่งเดียว) เพราะจะกิน memory มากและทำให้ debug ยากหากมีข้อผิดพลาดตรงกลาง ในทางปฏิบัติมักแบ่งเป็น batch ขนาดพอเหมาะ (ดูรายละเอียดใน Step 90)

### ตรวจสอบข้อมูลที่ insert ไป

```sql
SELECT customer_id, full_name, email FROM customers ORDER BY customer_id;
```

```
 customer_id | full_name  |     email
-------------+------------+----------------
           1 | สมชาย ใจดี  | somchai.jaidee@example.com
(1 row)
```

(ตัวอย่าง multi-row customers ด้านบนเป็นตัวอย่างสมมติเพื่ออธิบายแนวคิด ยังไม่ได้รันจริงในสคีมาของเรา — เราจะ insert ลูกค้าเพิ่มเติมจริง ๆ ใน Step ถัดไป)

---

## Step 83: INSERT ... DEFAULT VALUES และการใช้ DEFAULT keyword

### DEFAULT VALUES — เพิ่มแถวใหม่โดยใช้ค่าเริ่มต้นทั้งหมด

ในบางสถานการณ์เราต้องการสร้างแถวใหม่โดยใช้ค่า default ของทุกคอลัมน์ (เช่นสร้าง placeholder หรือสร้าง order ใหม่แบบว่าง ๆ ที่ยังไม่รู้รายละเอียด) PostgreSQL มี syntax พิเศษสำหรับกรณีนี้:

```sql
INSERT INTO table_name DEFAULT VALUES;
```

ตัวอย่าง: สร้างออเดอร์ใหม่ที่ยังไม่ผูกกับลูกค้า (walk-in guest ที่ยังไม่ลงทะเบียน) โดยใช้ค่า default ทั้งหมด:

```sql
INSERT INTO orders DEFAULT VALUES
RETURNING order_id, order_status, order_date;
```

```
 order_id | order_status |          order_date
----------+--------------+-------------------------------
        1 | pending      | 2026-09-25 10:22:05.117832+00
(1 row)

INSERT 0 1
```

ทุกคอลัมน์ในตาราง `orders` ที่มี `DEFAULT` หรืออนุญาต `NULL` จะถูกเติมค่าให้อัตโนมัติ:
- `order_id` → ใช้ `SERIAL` sequence
- `customer_id` → เป็น `NULL` (เพราะไม่มี `NOT NULL` และไม่มี `DEFAULT` กำหนดไว้)
- `order_status` → ใช้ค่า `'pending'` ตาม `DEFAULT`
- `order_date` → ใช้ `now()` ตาม `DEFAULT`
- `notes` → เป็น `NULL`

> `DEFAULT VALUES` ใช้ได้เฉพาะเมื่อทุกคอลัมน์ในตารางมี `DEFAULT` หรืออนุญาตให้เป็น `NULL` ถ้ามีคอลัมน์ใดเป็น `NOT NULL` โดยไม่มี `DEFAULT` คำสั่งนี้จะ error ทันที

### คีย์เวิร์ด DEFAULT สำหรับบางคอลัมน์

เราสามารถใช้คีย์เวิร์ด `DEFAULT` แทนค่าตัวใดตัวหนึ่งใน `VALUES` ได้ โดยไม่ต้องปล่อยให้ค่า default ครอบคลุมทั้งแถว:

```sql
INSERT INTO products (category_id, product_name, sku, price, is_active)
VALUES (3, 'Chocolate Chip Cookie', 'SWT-CCC-001', 35.00, DEFAULT)
RETURNING product_id, product_name, is_active;
```

```
 product_id |     product_name      | is_active
------------+------------------------+-----------
          9 | Chocolate Chip Cookie  | t
(1 row)

INSERT 0 1
```

ในที่นี้ `is_active` จะได้ค่า `TRUE` ซึ่งเป็นค่า default ที่กำหนดไว้ตอน `CREATE TABLE` (`DEFAULT TRUE`) การเขียน `DEFAULT` ตรง ๆ แบบนี้มีประโยชน์เมื่อ:

- ต้องการให้โค้ดชัดเจนว่า "ค่านี้ตั้งใจใช้ default" ไม่ใช่ลืมใส่
- เขียนโปรแกรมที่ generate คำสั่ง SQL แบบ dynamic แล้วต้องการคง column list ให้ครบทุกคอลัมน์เสมอ เพื่อความสม่ำเสมอของโค้ด

### DEFAULT ผสมกับ multi-row INSERT

```sql
INSERT INTO customers (full_name, email, phone, loyalty_point)
VALUES
    ('สุดา ยินดี', 'suda.yindee@example.com', '081-234-5678', DEFAULT),
    ('วิชัย มั่งมี', 'wichai.mangmee@example.com', DEFAULT, 100);
```

```
INSERT 0 2
```

- แถวแรก `loyalty_point` จะได้ `0` (ค่า default)
- แถวที่สอง `phone` จะได้ `NULL` (ค่า default เมื่อไม่มี `DEFAULT` clause กำหนดไว้ชัดเจนสำหรับ `phone` แต่คอลัมน์อนุญาต `NULL`) และ `loyalty_point` ได้ `100` ตามที่ระบุ

ตรวจสอบผลลัพธ์:

```sql
SELECT full_name, phone, loyalty_point FROM customers ORDER BY customer_id;
```

```
   full_name   |    phone     | loyalty_point
----------------+--------------+---------------
 สมชาย ใจดี      |              |             0
 สุดา ยินดี      | 081-234-5678 |             0
 วิชัย มั่งมี      |              |           100
(3 rows)
```

---

## Step 84: RETURNING clause — ดึงค่าที่เพิ่ง insert กลับมา

### ปัญหาที่ RETURNING แก้ไข

เวลาเราสร้างแถวใหม่ที่มีค่า auto-generate เช่น `SERIAL`/`IDENTITY` หรือ `DEFAULT now()` แอปพลิเคชันมักต้องรู้ค่าที่ระบบสร้างขึ้นทันที (เช่น `order_id` ที่เพิ่งสร้าง เพื่อเอาไปสร้าง `order_items` ต่อ) วิธีเดิมที่หลายคนคุ้นเคยจากฐานข้อมูลอื่นคือ insert แล้วค่อย `SELECT` แยกเพื่อดึงค่ากลับมา แต่วิธีนี้มีปัญหา:

- เสีย round-trip เพิ่มอีกครั้ง (ประสิทธิภาพต่ำกว่า)
- อาจเกิด race condition ถ้ามีหลาย connection ทำงานพร้อมกัน และเราพยายามหา "แถวล่าสุดที่เพิ่ง insert" ด้วยวิธีที่ไม่ปลอดภัย เช่น `SELECT MAX(id)`

PostgreSQL มี `RETURNING` clause ที่ให้เราดึงค่าที่เพิ่ง insert กลับมาได้ทันทีในคำสั่งเดียว โดยไม่ต้อง query แยก

### Syntax

```sql
INSERT INTO table_name (column1, column2)
VALUES (value1, value2)
RETURNING column1, column2, ...;
```

### ตัวอย่างที่ 1: ดึง id ที่ auto-generate

```sql
INSERT INTO products (category_id, product_name, sku, price)
VALUES (6, 'Ham & Cheese Set', 'BRK-HAC-001', 95.00)
RETURNING product_id;
```

```
 product_id
------------
         10
(1 row)

INSERT 0 1
```

### ตัวอย่างที่ 2: RETURNING หลายคอลัมน์

```sql
INSERT INTO customers (full_name, email)
VALUES ('มานี รักเรียน', 'manee.rakrian@example.com')
RETURNING customer_id, full_name, joined_at;
```

```
 customer_id |   full_name    |          joined_at
-------------+----------------+-------------------------------
           4 | มานี รักเรียน   | 2026-09-25 10:31:44.883921+00
(1 row)

INSERT 0 1
```

### ตัวอย่างที่ 3: RETURNING ทุกคอลัมน์ด้วย *

```sql
INSERT INTO orders (customer_id, order_status, notes)
VALUES (4, 'pending', 'ลูกค้าสั่งผ่านแอป')
RETURNING *;
```

```
 order_id | customer_id | order_status |          order_date           |       notes
----------+-------------+--------------+--------------------------------+-------------------
        2 |           4 | pending      | 2026-09-25 10:32:10.204555+00 | ลูกค้าสั่งผ่านแอป
(1 row)

INSERT 0 1
```

### ตัวอย่างที่ 4: RETURNING กับ expression

`RETURNING` ไม่จำกัดแค่ชื่อคอลัมน์ตรง ๆ แต่รองรับ expression ได้ด้วย เช่นคำนวณ `unit_price * quantity` ทันทีตอน insert:

```sql
INSERT INTO order_items (order_id, product_id, quantity, unit_price)
VALUES (2, 1, 2, 55.00)
RETURNING order_item_id, quantity, unit_price, (quantity * unit_price) AS line_total;
```

```
 order_item_id | quantity | unit_price | line_total
----------------+----------+------------+------------
              1 |        2 |      55.00 |     110.00
(1 row)

INSERT 0 1
```

### ตัวอย่างที่ 5: RETURNING กับ multi-row INSERT

`RETURNING` ทำงานร่วมกับ multi-row `INSERT` ได้ โดยจะคืนค่าทุกแถวที่ insert สำเร็จ:

```sql
INSERT INTO order_items (order_id, product_id, quantity, unit_price)
VALUES
    (2, 3, 1, 60.00),
    (2, 6, 1, 45.00)
RETURNING order_item_id, product_id, quantity;
```

```
 order_item_id | product_id | quantity
----------------+------------+----------
              2 |          3 |        1
              3 |          6 |        1
(2 rows)

INSERT 0 2
```

### ทำไม RETURNING สำคัญในงาน production

ในแอปพลิเคชันจริง เรามักเขียนโค้ดแบบนี้ (ตัวอย่างแนวคิดในภาษาโปรแกรมทั่วไป ไม่ใช่ SQL):

```sql
-- แทนที่จะต้องทำ 2 คำสั่ง:
-- 1) INSERT INTO orders ...
-- 2) SELECT order_id FROM orders WHERE ... (เสี่ยง race condition)

-- ใช้คำสั่งเดียวจบ:
INSERT INTO orders (customer_id, order_status)
VALUES (4, 'pending')
RETURNING order_id;
```

แล้วนำ `order_id` ที่ได้ไปใช้สร้าง `order_items` ต่อได้ทันทีในฝั่งแอปพลิเคชัน โดยไม่ต้องกังวลเรื่อง concurrency ว่าจะได้ id ผิดแถว

---

## Step 85: INSERT ... SELECT — copy ข้อมูลจากตารางอื่นหรือผลลัพธ์ query

### แนวคิด

แทนที่จะใช้ `VALUES` ที่มีค่าคงที่ เราสามารถใช้ผลลัพธ์จากคำสั่ง `SELECT` เป็นแหล่งข้อมูลให้ `INSERT` ได้โดยตรง ซึ่งมีประโยชน์มากเมื่อต้องการ:

- คัดลอกข้อมูลจากตารางหนึ่งไปอีกตารางหนึ่ง (เช่น archive ข้อมูลเก่า)
- สร้างข้อมูลสรุป/aggregate ลงตารางใหม่
- ย้ายข้อมูลระหว่างสภาพแวดล้อม (staging → production)
- คัดลอกโครงสร้างข้อมูลเดิมมาปรับปรุงบางส่วน

### Syntax

```sql
INSERT INTO target_table (column1, column2, ...)
SELECT expr1, expr2, ...
FROM source_table
WHERE condition;
```

จำนวนและชนิดข้อมูล (type) ของคอลัมน์ใน `SELECT` ต้องตรงกับที่ระบุใน `INSERT INTO target_table (...)` (หรือแปลงชนิดได้โดย implicit/explicit cast)

### ตัวอย่างที่ 1: สร้างตาราง archive แล้ว copy ข้อมูล

สมมติเราต้องการเก็บออเดอร์ที่ "เสร็จสมบูรณ์" (completed) ไว้ในตาราง archive แยกต่างหาก เพื่อให้ตารางหลักเบาลง:

```sql
CREATE TABLE orders_archive (
    order_id     INTEGER PRIMARY KEY,
    customer_id  INTEGER,
    order_status VARCHAR(20),
    order_date   TIMESTAMPTZ,
    notes        TEXT,
    archived_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

```
CREATE TABLE
```

ก่อนอื่นอัปเดตสถานะออเดอร์ตัวอย่างให้เป็น completed (เพื่อให้มีข้อมูลไว้ทดสอบ — คำสั่ง `UPDATE` จะสอนละเอียดในบทถัดไป):

```sql
UPDATE orders SET order_status = 'completed' WHERE order_id = 1;
```

```
UPDATE 1
```

จากนั้นคัดลอกเฉพาะออเดอร์ที่ completed ไปยังตาราง archive:

```sql
INSERT INTO orders_archive (order_id, customer_id, order_status, order_date, notes)
SELECT order_id, customer_id, order_status, order_date, notes
FROM orders
WHERE order_status = 'completed';
```

```
INSERT 0 1
```

ตรวจสอบผลลัพธ์:

```sql
SELECT order_id, order_status, archived_at FROM orders_archive;
```

```
 order_id | order_status |          archived_at
----------+--------------+-------------------------------
        1 | completed    | 2026-09-25 10:41:07.552901+00
(1 row)
```

สังเกตว่าเราไม่ต้องใส่ค่าให้ `archived_at` เพราะเป็นคอลัมน์ที่มีเฉพาะในตารางปลายทางและมี `DEFAULT now()` — คอลัมน์นี้ไม่ได้อยู่ใน column list ของ `INSERT` เลย ดังนั้นจะใช้ default โดยอัตโนมัติ

### ตัวอย่างที่ 2: INSERT ... SELECT พร้อม JOIN

เราสามารถใช้ query ที่ซับซ้อนกว่านั้น เช่น `JOIN` หลายตาราง เป็นแหล่งข้อมูลให้ `INSERT` ได้เช่นกัน สมมติต้องการสร้างตารางสรุปยอดขายรายสินค้า:

```sql
CREATE TABLE product_sales_summary (
    product_id    INTEGER PRIMARY KEY REFERENCES products(product_id),
    product_name  VARCHAR(100),
    total_quantity INTEGER,
    total_revenue  NUMERIC(12, 2)
);
```

```
CREATE TABLE
```

```sql
INSERT INTO product_sales_summary (product_id, product_name, total_quantity, total_revenue)
SELECT
    p.product_id,
    p.product_name,
    SUM(oi.quantity)              AS total_quantity,
    SUM(oi.quantity * oi.unit_price) AS total_revenue
FROM products p
JOIN order_items oi ON oi.product_id = p.product_id
GROUP BY p.product_id, p.product_name;
```

```
INSERT 0 3
```

```sql
SELECT * FROM product_sales_summary ORDER BY total_revenue DESC;
```

```
 product_id | product_name | total_quantity | total_revenue
------------+---------------+-----------------+----------------
          1 | Espresso      |               2 |         110.00
          3 | Latte         |               1 |          60.00
          6 | Croissant     |               1 |          45.00
(3 rows)
```

หมายเหตุ: `GROUP BY`, `SUM()`, และ `JOIN` จะสอนละเอียดในบทถัดไปเกี่ยวกับ `SELECT` และ aggregate functions ในที่นี้เพียงต้องการแสดงว่า `INSERT ... SELECT` รองรับ query ที่ซับซ้อนได้เต็มรูปแบบ

### ตัวอย่างที่ 3: INSERT ... SELECT ในตารางเดียวกัน (สร้างข้อมูลตัวอย่างจากข้อมูลเดิม)

บางครั้งเราต้องการ duplicate แถวบางแถวในตารางเดียวกันเพื่อจุดประสงค์บางอย่าง เช่นสร้างสินค้าตัวใหม่ที่คล้ายของเดิมแต่เปลี่ยนราคา:

```sql
INSERT INTO products (category_id, product_name, sku, price)
SELECT category_id, product_name || ' (Large)', sku || '-L', price + 10.00
FROM products
WHERE sku = 'BEV-LAT-001';
```

```
INSERT 0 1
```

```sql
SELECT product_name, sku, price FROM products WHERE sku LIKE 'BEV-LAT%';
```

```
 product_name  |    sku     | price
----------------+------------+-------
 Latte          | BEV-LAT-001| 60.00
 Latte (Large)  | BEV-LAT-001-L| 70.00
(2 rows)
```

### INSERT ... SELECT ร่วมกับ RETURNING

```sql
INSERT INTO orders_archive (order_id, customer_id, order_status, order_date, notes)
SELECT order_id, customer_id, order_status, order_date, notes
FROM orders
WHERE order_status = 'completed' AND order_id <> 1  -- เลี่ยง unique violation จากตัวอย่างก่อนหน้า
RETURNING order_id;
```

```
 order_id
----------
(0 rows)

INSERT 0 0
```

(ในตัวอย่างนี้ไม่มีแถวใหม่ที่ตรงเงื่อนไข เพราะเรา archive order_id = 1 ไปแล้วก่อนหน้านี้ — แสดงให้เห็นว่า `RETURNING` คืนค่าว่างได้เมื่อไม่มีแถวถูก insert)

---

## Step 86: ON CONFLICT (UPSERT) — DO NOTHING และ DO UPDATE

### ปัญหาที่ UPSERT แก้ไข

บ่อยครั้งเราต้องการ "insert ถ้ายังไม่มี, update ถ้ามีอยู่แล้ว" — พฤติกรรมนี้เรียกว่า **UPSERT** (UPDATE + INSERT) ตัวอย่างเช่น การ sync ข้อมูลสินค้าจากระบบภายนอก ถ้า SKU มีอยู่แล้วให้อัปเดตราคา ถ้ายังไม่มีให้สร้างใหม่

วิธีเดิมที่หลายคนเคยทำ (เช่น `SELECT` เช็คก่อนแล้วค่อย `INSERT` หรือ `UPDATE`) มีปัญหาเรื่อง **race condition**: ถ้ามีสอง connection พยายาม insert ค่าเดียวกันพร้อมกัน อาจเกิด unique violation error ได้ แม้จะเช็คก่อนแล้วก็ตาม เพราะช่วงเวลาระหว่างเช็คกับ insert อาจมี connection อื่นแทรกเข้ามาก่อน

PostgreSQL แก้ปัญหานี้ด้วย `INSERT ... ON CONFLICT` ซึ่งเป็น atomic operation ระดับเดียว ไม่มีช่องว่างให้เกิด race condition

### Syntax พื้นฐาน

```sql
INSERT INTO table_name (columns...)
VALUES (values...)
ON CONFLICT (conflict_column) DO NOTHING;

-- หรือ

INSERT INTO table_name (columns...)
VALUES (values...)
ON CONFLICT (conflict_column) DO UPDATE
SET column1 = value1, ...;
```

### ON CONFLICT DO NOTHING

`DO NOTHING` หมายถึง "ถ้าชนกับข้อมูลเดิม (violate unique/primary key constraint) ให้ข้ามไปเฉย ๆ ไม่ error ไม่เปลี่ยนแปลงอะไร"

ตัวอย่าง: พยายาม insert หมวดหมู่ที่มีชื่อซ้ำกับที่มีอยู่แล้ว (คอลัมน์ `category_name` มี `UNIQUE` constraint):

```sql
INSERT INTO categories (category_name, description)
VALUES ('เครื่องดื่มร้อน', 'คำอธิบายใหม่ที่จะถูกละเว้น')
ON CONFLICT (category_name) DO NOTHING;
```

```
INSERT 0 0
```

สังเกตผลลัพธ์ `INSERT 0 0` — ไม่มีแถวใดถูก insert เพราะ `'เครื่องดื่มร้อน'` มีอยู่แล้ว และคำสั่งไม่ error เลย (ถ้าไม่มี `ON CONFLICT` คำสั่งนี้จะ error ด้วย unique violation ทันที)

ลองเทียบกับกรณีที่ไม่ชน:

```sql
INSERT INTO categories (category_name, description)
VALUES ('ของทานเล่น', 'มันฝรั่งทอด แซนด์วิช')
ON CONFLICT (category_name) DO NOTHING;
```

```
INSERT 0 1
```

### ON CONFLICT DO UPDATE (UPSERT เต็มรูปแบบ)

`DO UPDATE` หมายถึง "ถ้าชนกับข้อมูลเดิม ให้อัปเดตแถวนั้นแทน" ภายใน `SET` เราสามารถอ้างอิงค่าที่พยายาม insert ผ่านคีย์เวิร์ดพิเศษ `EXCLUDED` (หมายถึงแถวที่ "ถูกกันออก" เพราะชนกับของเดิม)

ตัวอย่าง: sync ราคาสินค้าจากระบบภายนอก โดยอ้างอิง `sku` เป็น unique key — ถ้า SKU มีอยู่แล้วให้อัปเดตราคาและชื่อ ถ้ายังไม่มีให้สร้างสินค้าใหม่:

```sql
INSERT INTO products (category_id, product_name, sku, price)
VALUES (1, 'Espresso', 'BEV-ESP-001', 58.00)
ON CONFLICT (sku) DO UPDATE
SET price = EXCLUDED.price,
    product_name = EXCLUDED.product_name
RETURNING product_id, product_name, price;
```

```
 product_id | product_name | price
------------+---------------+-------
          1 | Espresso      | 58.00
(1 row)

INSERT 0 1
```

สังเกตว่า `product_id = 1` (ตัวเดิมที่เคย insert ไว้ใน Step 81) ถูกอัปเดตราคาจาก `55.00` เป็น `58.00` แทนที่จะสร้างแถวใหม่ และ `EXCLUDED.price` หมายถึงค่า `58.00` ที่เราพยายาม insert เข้ามา (ไม่ใช่ค่าเดิมในตาราง)

### UPSERT พร้อม multi-row VALUES

`ON CONFLICT` ทำงานร่วมกับ multi-row `INSERT` ได้ โดยแต่ละแถวจะถูกตรวจสอบและจัดการ conflict แยกกัน:

```sql
INSERT INTO products (category_id, product_name, sku, price)
VALUES
    (1, 'Espresso',   'BEV-ESP-001', 60.00),  -- มีอยู่แล้ว จะถูก update
    (2, 'Mocha',       'BEV-MOC-001', 65.00)   -- ยังไม่มี จะถูก insert ใหม่
ON CONFLICT (sku) DO UPDATE
SET price = EXCLUDED.price
RETURNING product_id, sku, price;
```

```
 product_id |    sku      | price
------------+-------------+-------
          1 | BEV-ESP-001 | 60.00
         11 | BEV-MOC-001 | 65.00
(2 rows)

INSERT 0 2
```

### เงื่อนไขเพิ่มเติมใน DO UPDATE ด้วย WHERE

เราสามารถเพิ่ม `WHERE` เพื่อควบคุมว่าจะ update เมื่อไหร่เท่านั้น เช่น อัปเดตเฉพาะเมื่อราคาที่มาใหม่สูงกว่าราคาเดิม:

```sql
INSERT INTO products (category_id, product_name, sku, price)
VALUES (1, 'Espresso', 'BEV-ESP-001', 50.00)
ON CONFLICT (sku) DO UPDATE
SET price = EXCLUDED.price
WHERE EXCLUDED.price > products.price
RETURNING product_id, sku, price;
```

```
 product_id | sku | price
------------+-----+-------
(0 rows)

INSERT 0 0
```

ในที่นี้ราคาใหม่ (`50.00`) ต่ำกว่าราคาเดิม (`60.00`) เงื่อนไข `WHERE` จึงไม่เป็นจริง ทำให้ไม่มีการ update และ `RETURNING` คืนค่าว่าง (แต่ก็ไม่ error เช่นกัน — ถือว่า "ชนกันแต่ไม่ต้องทำอะไร")

---

## Step 87: ON CONFLICT ON CONSTRAINT และการระบุ conflict target แบบละเอียด

### สองวิธีในการระบุ conflict target

PostgreSQL รองรับการระบุว่า "ให้ตรวจ conflict จากอะไร" ได้ 2 รูปแบบหลัก:

1. **ระบุคอลัมน์ (หรือกลุ่มคอลัมน์)**: `ON CONFLICT (column1, column2)` — ต้องตรงกับ unique index หรือ unique/primary key constraint ที่มีอยู่จริง
2. **ระบุชื่อ constraint โดยตรง**: `ON CONFLICT ON CONSTRAINT constraint_name` — ใช้เมื่อต้องการอ้างอิง constraint ที่มีชื่อเฉพาะ โดยเฉพาะ composite unique constraint ที่ตั้งชื่อไว้แล้ว

### ตัวอย่าง: composite unique constraint

ตาราง `order_items` ของเรามี `UNIQUE (order_id, product_id)` — หมายความว่าในออเดอร์เดียวกัน จะมีสินค้าชนิดเดียวกันซ้ำไม่ได้ (ต้องรวม quantity เข้าด้วยกันแทน)

ก่อนอื่นมาดูชื่อ constraint ที่ PostgreSQL ตั้งให้อัตโนมัติ:

```sql
SELECT conname, contype, pg_get_constraintdef(oid) AS definition
FROM pg_constraint
WHERE conrelid = 'order_items'::regclass;
```

```
        conname          | contype |                     definition
--------------------------+---------+------------------------------------------------------
 order_items_pkey         | p       | PRIMARY KEY (order_item_id)
 order_items_order_id_fkey| f       | FOREIGN KEY (order_id) REFERENCES orders(order_id) ...
 order_items_product_id_fkey | f    | FOREIGN KEY (product_id) REFERENCES products(product_id)
 order_items_order_id_product_id_key | u | UNIQUE (order_id, product_id)
(4 rows)
```

PostgreSQL ตั้งชื่อ constraint ให้อัตโนมัติเป็น `order_items_order_id_product_id_key` เราสามารถใช้ชื่อนี้ใน `ON CONFLICT ON CONSTRAINT` ได้:

```sql
INSERT INTO order_items (order_id, product_id, quantity, unit_price)
VALUES (2, 1, 3, 55.00)
ON CONFLICT ON CONSTRAINT order_items_order_id_product_id_key
DO UPDATE SET quantity = order_items.quantity + EXCLUDED.quantity
RETURNING order_item_id, order_id, product_id, quantity;
```

```
 order_item_id | order_id | product_id | quantity
----------------+----------+------------+----------
              1 |        2 |          1 |        5
(1 row)

INSERT 0 1
```

สังเกตว่า `order_id = 2, product_id = 1` มีอยู่แล้ว (จาก Step 84 ที่ quantity = 2) เมื่อพยายาม insert quantity = 3 เพิ่ม ระบบจะรวมยอดเป็น `2 + 3 = 5` แทนการสร้างแถวใหม่หรือ error — นี่คือรูปแบบ UPSERT ที่ใช้บ่อยมากสำหรับ "สะสมจำนวน" (เช่น ตะกร้าสินค้า, inventory)

เทียบกับการเขียนแบบระบุคอลัมน์ตรง ๆ (ได้ผลลัพธ์เหมือนกัน เพราะ PostgreSQL จะหา unique constraint ที่ตรงกับคอลัมน์ที่ระบุให้อัตโนมัติ):

```sql
INSERT INTO order_items (order_id, product_id, quantity, unit_price)
VALUES (2, 1, 1, 55.00)
ON CONFLICT (order_id, product_id)
DO UPDATE SET quantity = order_items.quantity + EXCLUDED.quantity
RETURNING order_item_id, quantity;
```

```
 order_item_id | quantity
----------------+----------
              1 |        6
(1 row)

INSERT 0 1
```

### เมื่อไหร่ควรใช้ ON CONSTRAINT แทนการระบุคอลัมน์

| สถานการณ์ | แนะนำใช้ |
|---|---|
| Unique constraint ธรรมดา ที่รู้ชื่อคอลัมน์ชัดเจน | `ON CONFLICT (column)` — อ่านง่าย ไม่ต้องจำชื่อ constraint |
| Partial unique index (มี `WHERE` clause) | ต้องระบุ index ให้ตรงกับ predicate — บางกรณีจำเป็นต้องใช้ชื่อ constraint/index |
| Exclusion constraint (`EXCLUDE USING gist ...`) | ต้องใช้ `ON CONFLICT ON CONSTRAINT constraint_name` เท่านั้น เพราะ exclusion constraint ไม่ใช่ unique constraint ธรรมดา |
| ต้องการให้โค้ด robust ต่อการเปลี่ยนชื่อคอลัมน์ในอนาคต | ใช้ชื่อ constraint ที่ตั้งเองอย่างมีความหมาย เช่น `uq_order_items_order_product` |

> **ทางปฏิบัติที่ดี**: เมื่อออกแบบตารางที่จะใช้ `ON CONFLICT` บ่อย ควรตั้งชื่อ constraint เองให้สื่อความหมาย แทนที่จะปล่อยให้ PostgreSQL ตั้งชื่ออัตโนมัติ เช่น:
>
> ```sql
> ALTER TABLE order_items
>     DROP CONSTRAINT order_items_order_id_product_id_key,
>     ADD CONSTRAINT uq_order_items_order_product UNIQUE (order_id, product_id);
> ```
>
> จากนั้นใช้ `ON CONFLICT ON CONSTRAINT uq_order_items_order_product DO UPDATE ...` ได้อย่างชัดเจนและอ่านง่ายกว่า

### ON CONFLICT DO NOTHING แบบไม่ระบุ target

หากไม่ระบุคอลัมน์หรือ constraint เลย (`ON CONFLICT DO NOTHING` เฉย ๆ) PostgreSQL จะข้ามการ insert เมื่อชนกับ **unique/exclusion constraint ใดก็ได้** ในตาราง ซึ่งสะดวกแต่ควบคุมได้น้อยกว่า:

```sql
INSERT INTO categories (category_name, description)
VALUES ('เบเกอรี่', 'ชื่อซ้ำจะถูกข้าม')
ON CONFLICT DO NOTHING;
```

```
INSERT 0 0
```

> ข้อควรระวัง: `ON CONFLICT DO NOTHING` แบบไม่ระบุ target **ใช้ร่วมกับ `DO UPDATE` ไม่ได้** — ถ้าจะ `DO UPDATE` ต้องระบุ conflict target เสมอ เพราะ PostgreSQL ต้องรู้ว่าจะ update โดยอ้างอิงจาก constraint ไหน

---

## Step 88: Bulk Insert ด้วย COPY เทียบกับ INSERT หลายคำสั่ง

### เมื่อข้อมูลมีจำนวนมาก (หลักหมื่น–ล้านแถว)

เมื่อต้อง insert ข้อมูลจำนวนมาก เช่น import ข้อมูลลูกค้าจากไฟล์ CSV เป็นแสนแถว `INSERT` (แม้จะเป็น multi-row) ก็ยังไม่ใช่ตัวเลือกที่เร็วที่สุด PostgreSQL มีคำสั่ง `COPY` ที่ออกแบบมาเฉพาะสำหรับ bulk data transfer

### COPY คืออะไร

`COPY` เป็นคำสั่งที่ถ่ายโอนข้อมูลระหว่างตารางกับไฟล์ (หรือ stream) โดยตรง ในรูปแบบข้อความ (text/CSV) หรือ binary โดยไม่ต้อง parse SQL statement ทีละแถวเหมือน `INSERT`

```sql
COPY table_name (column1, column2, ...)
FROM '/path/to/file.csv'
WITH (FORMAT csv, HEADER true);
```

ตัวอย่าง: สมมติมีไฟล์ `customers_import.csv` หน้าตาแบบนี้:

```
full_name,email,phone
กิตติ สุขใจ,kitti.sukjai@example.com,082-111-2222
นภา แสงทอง,napa.saengthong@example.com,083-222-3333
ประยุทธ คงมั่น,prayuth.kongman@example.com,084-333-4444
```

```sql
COPY customers (full_name, email, phone)
FROM '/tmp/customers_import.csv'
WITH (FORMAT csv, HEADER true);
```

```
COPY 3
```

ผลลัพธ์บอกว่า copy สำเร็จ 3 แถว

### psql meta-command \copy

ถ้าไฟล์อยู่ที่เครื่อง client (ไม่ใช่เครื่อง server ที่รัน PostgreSQL) ให้ใช้ `\copy` ของ `psql` แทน (มี backslash นำหน้า ไม่ใช่ SQL command แต่เป็น client-side command ที่ psql จัดการให้):

```sql
\copy customers (full_name, email, phone) FROM '/local/path/customers_import.csv' WITH (FORMAT csv, HEADER true)
```

```
COPY 3
```

`\copy` จะอ่านไฟล์จากฝั่ง client แล้วส่งข้อมูลผ่าน connection ไปให้ server — มีประโยชน์มากเมื่อคุณเชื่อมต่อ server จากเครื่องอื่นและไม่มีสิทธิ์เข้าถึงไฟล์ระบบบน server โดยตรง

### เปรียบเทียบ Performance: COPY vs multi-row INSERT vs INSERT ทีละแถว

| วิธี | ความเร็วโดยประมาณ (สัมพัทธ์) | เหมาะกับ |
|---|---|---|
| `INSERT` ทีละแถว (auto-commit) | ช้าที่สุด (1x) | ข้อมูลน้อยมาก (สิบ–ร้อยแถว), งาน interactive |
| `INSERT` ทีละแถวใน transaction เดียว | เร็วขึ้น (~3-5x) | กรณีต้อง insert ทีละแถวแต่ต้องการลด fsync overhead |
| Multi-row `INSERT` (batch ละ 100-1000 แถว) | เร็วขึ้นมาก (~10-30x) | ข้อมูลระดับพัน–หมื่นแถว, มาจากแอปพลิเคชันที่ generate ข้อมูลเป็น batch |
| `COPY` | เร็วที่สุด (~30-50x หรือมากกว่า) | Bulk import ข้อมูลระดับหมื่น–ล้านแถว จากไฟล์หรือ ETL pipeline |

เหตุผลที่ `COPY` เร็วกว่า `INSERT` แม้จะเป็น multi-row:

1. **Protocol overhead ต่ำกว่า** — `COPY` ใช้ wire protocol แบบ streaming เฉพาะทาง ไม่ต้อง parse SQL statement syntax แบบ `INSERT` ทำให้ parsing overhead ต่อแถวต่ำกว่ามาก
2. **ไม่ต้องสร้าง execution plan ซ้ำ** — `INSERT` แต่ละคำสั่ง (แม้จะเป็น prepared statement) ยังมีค่าใช้จ่ายด้านการ bind parameter ต่อแถว ในขณะที่ `COPY` อ่านข้อมูลเป็น stream ต่อเนื่อง
3. **Batching ที่ optimize มาให้ในระดับ internal** — PostgreSQL จัดการ WAL และ buffer สำหรับ `COPY` อย่างมีประสิทธิภาพเป็นพิเศษ

### ข้อจำกัดของ COPY

`COPY` ไม่ใช่ตัวเลือกที่ดีที่สุดเสมอไป มีข้อจำกัดที่ควรรู้:

- **ไม่รองรับ `ON CONFLICT`** — `COPY` แบบมาตรฐานจะ error ทันทีถ้าชน constraint (ไม่มี UPSERT ในตัว) ถ้าต้องการ UPSERT ระหว่าง bulk load ต้องใช้เทคนิคเสริม เช่น `COPY` เข้า staging table ก่อน แล้วค่อย `INSERT ... SELECT ... ON CONFLICT` จาก staging table ไปตารางจริง
- **ไม่รองรับ `RETURNING`** — ไม่สามารถดึงค่าที่ auto-generate กลับมาได้โดยตรงจาก `COPY`
- **รูปแบบข้อมูลต้องตรง** — ไฟล์ต้องอยู่ในรูปแบบที่ `COPY` เข้าใจ (CSV, text, binary) ถ้าข้อมูลต้อง transform ซับซ้อนก่อน insert อาจต้องประมวลผลก่อนแล้วค่อย `COPY`

### รูปแบบผสม: COPY เข้า staging table แล้ว UPSERT

เทคนิคที่ใช้บ่อยในงาน ETL คือ:

```sql
-- 1) สร้าง staging table ชั่วคราว
CREATE TEMP TABLE products_staging (
    sku          VARCHAR(20),
    product_name VARCHAR(100),
    price        NUMERIC(10, 2)
);

-- 2) COPY ข้อมูลดิบเข้า staging table (เร็วมาก ไม่มี constraint checking ที่ซับซ้อน)
COPY products_staging (sku, product_name, price)
FROM '/tmp/products_import.csv'
WITH (FORMAT csv, HEADER true);

-- 3) UPSERT จาก staging table ไปตารางจริง โดยใช้ INSERT ... SELECT ... ON CONFLICT
INSERT INTO products (category_id, product_name, sku, price)
SELECT 1, s.product_name, s.sku, s.price
FROM products_staging s
ON CONFLICT (sku) DO UPDATE
SET price = EXCLUDED.price,
    product_name = EXCLUDED.product_name;

-- 4) ลบ staging table (TEMP table จะหายอัตโนมัติเมื่อ session จบ แต่ลบทันทีก็ได้)
DROP TABLE products_staging;
```

วิธีนี้ได้ทั้งความเร็วของ `COPY` และความสามารถ UPSERT ของ `INSERT ... ON CONFLICT`

---

## Step 89: ข้อผิดพลาดที่พบบ่อยตอน INSERT และวิธีอ่าน Error Message

การเข้าใจ error message ของ PostgreSQL เป็นทักษะสำคัญที่ช่วยประหยัดเวลา debug ได้มาก มาดู error ที่พบบ่อยที่สุด 4 แบบ

### 1. NOT NULL violation

เกิดเมื่อพยายาม insert `NULL` (หรือไม่ระบุค่า) ให้คอลัมน์ที่มี `NOT NULL` constraint:

```sql
INSERT INTO products (category_id, product_name, sku)
VALUES (1, 'Test Product', 'TEST-001');
```

```
ERROR:  null value in column "price" of relation "products" violates not-null constraint
DETAIL:  Failing row contains (12, 1, Test Product, TEST-001, null, t, 2026-09-25 10:55:12+00).
```

**วิธีอ่าน**:
- `column "price"` → บอกชัดเจนว่าคอลัมน์ไหนเป็นปัญหา
- `DETAIL` → แสดงค่าทั้งแถวที่พยายาม insert ทำให้เห็นบริบทว่าคอลัมน์อื่น ๆ มีค่าอะไรบ้าง

**วิธีแก้**: ใส่ค่าให้คอลัมน์ `price` เสมอ (เพราะไม่มี `DEFAULT` กำหนดไว้และเป็น `NOT NULL`)

```sql
INSERT INTO products (category_id, product_name, sku, price)
VALUES (1, 'Test Product', 'TEST-001', 30.00);
```

```
INSERT 0 1
```

### 2. Unique constraint violation

เกิดเมื่อพยายาม insert ค่าที่ซ้ำกับค่าที่มีอยู่แล้วในคอลัมน์ที่มี `UNIQUE` หรือ `PRIMARY KEY` constraint:

```sql
INSERT INTO products (category_id, product_name, sku, price)
VALUES (1, 'Duplicate SKU Test', 'TEST-001', 40.00);
```

```
ERROR:  duplicate key value violates unique constraint "products_sku_key"
DETAIL:  Key (sku)="TEST-001" already exists.
```

**วิธีอ่าน**:
- `unique constraint "products_sku_key"` → ชื่อ constraint ที่ถูกละเมิด (ชื่อนี้ตั้งอัตโนมัติ ปกติจะเป็น `<table>_<column>_key`)
- `DETAIL: Key (sku)="TEST-001" already exists` → บอกตรง ๆ ว่าคอลัมน์ไหน ค่าอะไรที่ชนกัน

**วิธีแก้**: เปลี่ยนค่าให้ไม่ซ้ำ หรือถ้าต้องการ UPSERT ให้ใช้ `ON CONFLICT` (Step 86-87)

```sql
INSERT INTO products (category_id, product_name, sku, price)
VALUES (1, 'Unique SKU Test', 'TEST-002', 40.00);
```

```
INSERT 0 1
```

### 3. Foreign key violation

เกิดเมื่อพยายาม insert ค่าที่อ้างอิงไปยังแถวที่ไม่มีอยู่จริงในตารางแม่ (parent table):

```sql
INSERT INTO products (category_id, product_name, sku, price)
VALUES (999, 'Ghost Category Product', 'TEST-003', 50.00);
```

```
ERROR:  insert or update on table "products" violates foreign key constraint "products_category_id_fkey"
DETAIL:  Key (category_id)=(999) already exists.
```

รอสักครู่ — ในความเป็นจริง PostgreSQL จะแสดง detail ที่แม่นยำกว่านี้สำหรับกรณี foreign key violation ฝั่ง insert คือ "ไม่มีอยู่" ไม่ใช่ "มีอยู่แล้ว" ข้อความจริงจะเป็น:

```
ERROR:  insert or update on table "products" violates foreign key constraint "products_category_id_fkey"
DETAIL:  Key (category_id)=(999) is not present in table "categories".
```

**วิธีอ่าน**:
- `foreign key constraint "products_category_id_fkey"` → ชื่อ FK constraint
- `Key (category_id)=(999) is not present in table "categories"` → บอกชัดว่า `category_id = 999` ไม่มีอยู่จริงในตาราง `categories`

**วิธีแก้**: ตรวจสอบก่อนว่า `category_id` ที่จะใช้มีอยู่จริง หรือ insert ตารางแม่ก่อน

```sql
SELECT category_id, category_name FROM categories WHERE category_id = 999;
```

```
 category_id | category_name
-------------+----------------
(0 rows)
```

```sql
INSERT INTO products (category_id, product_name, sku, price)
VALUES (1, 'Valid Category Product', 'TEST-003', 50.00);
```

```
INSERT 0 1
```

### 4. Type mismatch / invalid input syntax

เกิดเมื่อค่าที่ใส่ไม่สามารถแปลงเป็นชนิดข้อมูลของคอลัมน์ได้:

```sql
INSERT INTO products (category_id, product_name, sku, price)
VALUES (1, 'Bad Price Test', 'TEST-004', 'ห้าสิบบาท');
```

```
ERROR:  invalid input syntax for type numeric: "ห้าสิบบาท"
LINE 2: VALUES (1, 'Bad Price Test', 'TEST-004', 'ห้าสิบบาท');
                                                   ^
```

**วิธีอ่าน**:
- `invalid input syntax for type numeric` → บอกชนิดข้อมูลปลายทางที่คาดหวัง (`numeric` เพราะ `price` เป็น `NUMERIC(10,2)`)
- `"ห้าสิบบาท"` → ค่าที่พยายามแปลงแต่ล้มเหลว
- ลูกศร `^` ใต้ `LINE` → ชี้ตำแหน่งที่มีปัญหาใน SQL statement (มีประโยชน์มากเมื่อ query ยาวและซับซ้อน)

**วิธีแก้**: ใส่ค่าที่เป็นตัวเลขจริง ๆ

```sql
INSERT INTO products (category_id, product_name, sku, price)
VALUES (1, 'Correct Price Test', 'TEST-004', 50.00);
```

```
INSERT 0 1
```

### 5. CHECK constraint violation

เกิดเมื่อค่าไม่ผ่านเงื่อนไขที่กำหนดใน `CHECK` constraint (ในสคีมาของเรา `products.price` ต้อง `>= 0` และ `order_items.quantity` ต้อง `> 0`):

```sql
INSERT INTO products (category_id, product_name, sku, price)
VALUES (1, 'Negative Price Test', 'TEST-005', -10.00);
```

```
ERROR:  new row for relation "products" violates check constraint "products_price_check"
DETAIL:  Failing row contains (17, 1, Negative Price Test, TEST-005, -10.00, t, 2026-09-25 11:02:44+00).
```

**วิธีอ่าน**: `check constraint "products_price_check"` บอกชื่อ constraint และ `DETAIL` แสดงแถวทั้งหมดที่ล้มเหลว ทำให้เห็นว่า `price = -10.00` คือค่าที่ผิดเงื่อนไข `price >= 0`

### เทคนิคการอ่าน error message อย่างเป็นระบบ

เมื่อเจอ error จาก `INSERT` ให้อ่านตามลำดับนี้เสมอ:

1. **บรรทัด `ERROR:`** — บอกประเภทปัญหาโดยรวม (constraint อะไร, type อะไร)
2. **บรรทัด `DETAIL:`** (ถ้ามี) — ให้ข้อมูลเจาะจงมากขึ้น เช่นค่าไหนที่มีปัญหา
3. **บรรทัด `HINT:`** (ถ้ามี) — PostgreSQL บางครั้งจะแนะนำวิธีแก้ให้ตรง ๆ
4. **บรรทัด `LINE:` และลูกศร `^`** — ช่วยหาตำแหน่งที่แม่นยำในคำสั่ง SQL โดยเฉพาะเมื่อ query ยาว

ตัวอย่างการใช้ `HINT`:

```sql
INSERT INTO products (category_id, product_nam, sku, price)
VALUES (1, 'Typo Column Test', 'TEST-006', 30.00);
```

```
ERROR:  column "product_nam" of relation "products" does not exist
LINE 1: INSERT INTO products (category_id, product_nam, sku, price...
                                            ^
HINT:  Perhaps you meant to reference the column "products.product_name".
```

`HINT` บอกตรง ๆ ว่าน่าจะพิมพ์ผิด และคาดว่าตั้งใจใช้คอลัมน์ไหน — เป็นตัวช่วย debug ที่มีประโยชน์มาก

---

## Step 90: Best Practices — Transaction Wrapping, Batch Size, และ Validate ข้อมูลก่อน INSERT

### 1. ห่อ INSERT หลายคำสั่งด้วย Transaction

เมื่อต้อง insert ข้อมูลที่สัมพันธ์กันหลายตาราง (เช่นสร้างออเดอร์พร้อมรายการสินค้า) ควรห่อด้วย `BEGIN ... COMMIT` เพื่อรับประกันว่า **ทุกคำสั่งสำเร็จพร้อมกัน หรือไม่สำเร็จเลยสักคำสั่ง** (atomicity)

```sql
BEGIN;

INSERT INTO orders (customer_id, order_status)
VALUES (2, 'pending')
RETURNING order_id;
-- สมมติได้ order_id = 5

INSERT INTO order_items (order_id, product_id, quantity, unit_price)
VALUES
    (5, 1, 2, 58.00),
    (5, 4, 1, 55.00);

COMMIT;
```

```
BEGIN
INSERT 0 1
 order_id
----------
        5
(1 row)

INSERT 0 2
COMMIT
```

ถ้าคำสั่งที่สองล้มเหลว (เช่น `product_id` ไม่มีอยู่จริง) เราสามารถ `ROLLBACK` เพื่อยกเลิกทุกอย่างรวมถึง order ที่เพิ่งสร้าง ไม่ปล่อยให้เกิด "ออเดอร์กำพร้า" (order ที่ไม่มีรายการสินค้า) ค้างอยู่ในระบบ:

```sql
BEGIN;

INSERT INTO orders (customer_id, order_status)
VALUES (2, 'pending')
RETURNING order_id;
-- ได้ order_id = 6

INSERT INTO order_items (order_id, product_id, quantity, unit_price)
VALUES (6, 999, 1, 100.00);  -- product_id = 999 ไม่มีอยู่จริง -> FK violation

ROLLBACK;
```

```
BEGIN
INSERT 0 1
 order_id
----------
        6
(1 row)

ERROR:  insert or update on table "order_items" violates foreign key constraint "order_items_product_id_fkey"
DETAIL:  Key (product_id)=(999) is not present in table "products".
ROLLBACK
```

หลังจาก `ROLLBACK` ตรวจสอบได้ว่า order_id = 6 ไม่ถูกสร้างจริง เพราะทั้ง transaction ถูกยกเลิก:

```sql
SELECT * FROM orders WHERE order_id = 6;
```

```
 order_id | customer_id | order_status | order_date | notes
----------+-------------+--------------+------------+-------
(0 rows)
```

> **หลักการสำคัญ**: การกระทำที่ "ทางตรรกะควรสำเร็จพร้อมกัน" (เช่น order + order_items, หรือ debit บัญชีหนึ่ง + credit อีกบัญชีหนึ่ง) ต้องอยู่ใน transaction เดียวกันเสมอ ไม่เช่นนั้นข้อมูลจะไม่สอดคล้องกัน (data inconsistency) เมื่อเกิด error กลางทาง

### 2. กำหนด Batch Size ที่เหมาะสมสำหรับ Bulk Insert

เมื่อ insert ข้อมูลจำนวนมากจากแอปพลิเคชัน (ไม่ใช่จากไฟล์ที่ใช้ `COPY` ได้) ควรแบ่งเป็น batch แทนที่จะ:
- insert ทีละแถว (ช้าเกินไป ดังที่กล่าวใน Step 82)
- insert ทั้งหมดในคำสั่งเดียว (ถ้ามีเป็นล้านแถว อาจกิน memory มากเกินไปและยากต่อการจัดการ error)

แนวทางที่นิยม: แบ่งเป็น batch ละ **500-2,000 แถว** ต่อคำสั่ง `INSERT` (ตัวเลขจริงควรปรับตามขนาดข้อมูลต่อแถวและ hardware) แล้วห่อแต่ละ batch (หรือหลาย batch) ด้วย transaction เดียว:

```sql
BEGIN;

INSERT INTO customers (full_name, email, phone)
VALUES
    ('ลูกค้า 1', 'customer1@example.com', '080-000-0001'),
    ('ลูกค้า 2', 'customer2@example.com', '080-000-0002');
    -- ... ต่อเนื่องไปจนครบ batch (เช่น 1,000 แถว)

COMMIT;

-- แล้วเริ่ม batch ถัดไป
BEGIN;
INSERT INTO customers (full_name, email, phone)
VALUES
    ('ลูกค้า 1001', 'customer1001@example.com', '080-001-0001');
    -- ...
COMMIT;
```

**เหตุผลที่ไม่ควรใช้ batch ใหญ่เกินไปหรือเล็กเกินไป**:

| Batch size | ข้อดี | ข้อเสีย |
|---|---|---|
| เล็กมาก (1-10 แถว/คำสั่ง) | error ตรวจจับง่าย, memory ต่ำ | round-trip เยอะ, ช้า |
| กลาง (500-2,000 แถว/คำสั่ง) | สมดุลระหว่างความเร็วกับความปลอดภัย | ต้อง implement retry logic เมื่อ batch ใดล้มเหลว |
| ใหญ่มาก (แสน-ล้านแถว/คำสั่ง) | round-trip น้อยที่สุด | กิน memory มาก, ถ้า error ต้องเริ่มใหม่ทั้ง batch, transaction ค้างนานเสี่ยง lock contention |

สำหรับข้อมูลระดับแสน-ล้านแถว แนะนำให้ใช้ `COPY` (Step 88) แทน multi-row `INSERT` เพราะออกแบบมาเพื่องานนี้โดยเฉพาะ

### 3. Validate ข้อมูลก่อน INSERT

การป้องกันข้อผิดพลาดที่ดีที่สุดคือ **ตรวจสอบข้อมูลก่อนส่งเข้าฐานข้อมูล** แทนที่จะปล่อยให้ database constraint เป็นด่านสุดท้าย (แม้ constraint ก็ยังจำเป็นเป็น safety net เสมอ)

**แนวทางที่แนะนำ**:

1. **Validate ที่ชั้นแอปพลิเคชันก่อนเสมอ** — เช็ครูปแบบ email, ค่าตัวเลขต้องไม่ติดลบ, ความยาว string ไม่เกินขีดจำกัด ก่อนส่ง `INSERT` เพื่อลดจำนวน round-trip ที่จะ error โดยไม่จำเป็น
2. **ใช้ CHECK constraint เป็นด่านสุดท้ายในฐานข้อมูล** — เพราะแอปพลิเคชันอาจมีหลายตัว (web, mobile, batch job) การพึ่งพา validation ที่แอปอย่างเดียวเสี่ยงต่อการมี bug ในบางจุดที่ลืม validate ฐานข้อมูลจึงต้องเป็นผู้พิทักษ์ความถูกต้องขั้นสุดท้ายเสมอ (defense in depth)
3. **ตรวจสอบ foreign key ที่จะอ้างอิงล่วงหน้าเมื่อเป็นไปได้** — โดยเฉพาะเมื่อ import ข้อมูลจากแหล่งภายนอกที่ไม่น่าเชื่อถือเต็มร้อย
4. **ใช้ transaction + savepoint สำหรับ batch ที่ยอมรับความล้มเหลวบางส่วนได้** — ถ้าต้องการ insert ทีละแถวใน batch แต่ยอมให้บางแถว fail โดยไม่กระทบแถวอื่น สามารถใช้ `SAVEPOINT` ได้:

```sql
BEGIN;

SAVEPOINT before_row_1;
INSERT INTO customers (full_name, email) VALUES ('ลูกค้า X', 'x@example.com');
-- ถ้าสำเร็จ ไม่ต้องทำอะไรเพิ่ม

SAVEPOINT before_row_2;
INSERT INTO customers (full_name, email) VALUES ('ลูกค้า Y', 'x@example.com');
-- ถ้า error (เช่น email ซ้ำ) ให้ ROLLBACK TO SAVEPOINT เพื่อยกเลิกเฉพาะแถวนี้
ROLLBACK TO SAVEPOINT before_row_2;

COMMIT;  -- ลูกค้า X ยังถูก insert สำเร็จ ส่วนลูกค้า Y ถูกยกเลิกไป
```

```
BEGIN
SAVEPOINT
INSERT 0 1
SAVEPOINT
ERROR:  duplicate key value violates unique constraint "customers_email_key"
DETAIL:  Key (email)=(x@example.com) already exists.
ROLLBACK
COMMIT
```

> `SAVEPOINT` มีประโยชน์มากสำหรับ batch job ที่ต้องการความยืดหยุ่น แต่ก็มีค่าใช้จ่ายเพิ่มขึ้น (overhead) เมื่อเทียบกับ transaction ธรรมดา จึงควรใช้เฉพาะเมื่อจำเป็นจริง ๆ

### 5. เช็คลิสต์ Best Practices สรุป

- [ ] ระบุชื่อคอลัมน์ใน `INSERT INTO table (...)` เสมอ ไม่พึ่งพาลำดับ default ของตาราง
- [ ] ใช้ multi-row `VALUES` แทนการ `INSERT` ทีละแถว เมื่อมีหลายแถวที่รู้ล่วงหน้า
- [ ] ใช้ `RETURNING` แทนการ `SELECT` แยกเพื่อดึงค่า auto-generate
- [ ] ใช้ `ON CONFLICT` แทนการ "เช็คก่อน insert" ด้วยตัวเอง เพื่อเลี่ยง race condition
- [ ] ใช้ `COPY` สำหรับ bulk import ข้อมูลระดับหมื่นแถวขึ้นไป
- [ ] ห่อคำสั่งที่สัมพันธ์กันด้วย transaction (`BEGIN` / `COMMIT` / `ROLLBACK`) เสมอ
- [ ] แบ่ง batch ขนาดพอเหมาะ (ไม่เล็ก/ใหญ่เกินไป) สำหรับ bulk insert จากแอปพลิเคชัน
- [ ] Validate ข้อมูลทั้งที่ชั้นแอปพลิเคชันและพึ่งพา constraint ในฐานข้อมูลเป็นด่านสุดท้าย
- [ ] อ่าน `DETAIL` และ `HINT` ใน error message เสมอ ก่อนเดาสาเหตุ
- [ ] ตั้งชื่อ constraint เองให้สื่อความหมาย เพื่อให้ใช้ `ON CONFLICT ON CONSTRAINT` ได้ชัดเจน

---

## สรุปท้ายบท

ในบทนี้เราครอบคลุมการ `INSERT` ข้อมูลใน PostgreSQL อย่างละเอียด ตั้งแต่ระดับพื้นฐานจนถึงเทคนิคระดับ production:

- **INSERT พื้นฐาน** (`INSERT INTO ... VALUES`) ควรระบุคอลัมน์เสมอเพื่อความปลอดภัยและอ่านง่าย
- **Multi-row INSERT** ลด round-trip และเพิ่มประสิทธิภาพอย่างมากเมื่อเทียบกับการ insert ทีละแถว
- **DEFAULT VALUES / DEFAULT keyword** ช่วยให้ใช้ค่าเริ่มต้นของตารางได้อย่างชัดเจนและยืดหยุ่น
- **RETURNING** ดึงค่าที่เพิ่ง insert (เช่น auto-generated id) กลับมาได้ในคำสั่งเดียว ไม่ต้อง query ซ้ำ
- **INSERT ... SELECT** ใช้คัดลอก/ย้ายข้อมูลระหว่างตาราง หรือสร้างข้อมูลสรุปจาก query ที่ซับซ้อนได้
- **ON CONFLICT (UPSERT)** แก้ปัญหา race condition ในการ "insert ถ้าไม่มี, update ถ้ามี" ได้อย่าง atomic ทั้งแบบ `DO NOTHING` และ `DO UPDATE`
- **ON CONFLICT ON CONSTRAINT** ให้ความละเอียดในการระบุ conflict target โดยเฉพาะกับ composite unique constraint หรือ exclusion constraint
- **COPY** คือตัวเลือกที่เร็วที่สุดสำหรับ bulk insert ข้อมูลจำนวนมาก เหนือกว่า multi-row `INSERT` อย่างชัดเจน
- **Error message** ของ PostgreSQL ให้ข้อมูลที่ชัดเจนมาก (ERROR, DETAIL, HINT, LINE) การอ่านให้เป็นจะช่วยลดเวลา debug ได้มหาศาล
- **Best practices** เช่นการห่อ transaction, กำหนด batch size ที่เหมาะสม, และ validate ข้อมูลก่อน insert เป็นสิ่งที่แยก "โค้ดที่ใช้งานได้ในตอน demo" ออกจาก "โค้ดที่พร้อมใช้งานจริงใน production"

ในบทถัดไป (Part 010) เราจะเริ่มเจาะลึกคำสั่ง `SELECT` ซึ่งเป็นหัวใจของการดึงข้อมูลจาก PostgreSQL ตั้งแต่ระดับพื้นฐานไปจนถึงการกรอง เรียงลำดับ และจำกัดผลลัพธ์

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1

เขียนคำสั่ง `INSERT` เพื่อเพิ่มหมวดหมู่สินค้าใหม่ชื่อ `'เครื่องดื่มพิเศษ'` พร้อมคำอธิบาย `'เมนูตามฤดูกาล'` โดยระบุชื่อคอลัมน์ให้ครบถ้วน

<details>
<summary>เฉลย</summary>

```sql
INSERT INTO categories (category_name, description)
VALUES ('เครื่องดื่มพิเศษ', 'เมนูตามฤดูกาล');
```

ผลลัพธ์ที่คาดหวัง: `INSERT 0 1`

</details>

---

### แบบฝึกหัดที่ 2

เขียนคำสั่ง `INSERT` เดียวเพื่อเพิ่มลูกค้าใหม่ 3 คนพร้อมกัน (multi-row VALUES) ด้วยข้อมูลชื่อและอีเมลตามต้องการ

<details>
<summary>เฉลย</summary>

```sql
INSERT INTO customers (full_name, email)
VALUES
    ('อรุณ สว่างใจ', 'arun.sawangjai@example.com'),
    ('พิมพ์ใจ ดีงาม', 'pimjai.deengam@example.com'),
    ('ชาติชาย รุ่งเรือง', 'chatchai.rungrueang@example.com');
```

ผลลัพธ์ที่คาดหวัง: `INSERT 0 3`

</details>

---

### แบบฝึกหัดที่ 3

เขียนคำสั่ง `INSERT` เพื่อสร้างออเดอร์ใหม่โดยใช้ `DEFAULT VALUES` ทั้งหมด แล้วใช้ `RETURNING` เพื่อดึง `order_id` และ `order_status` กลับมา

<details>
<summary>เฉลย</summary>

```sql
INSERT INTO orders DEFAULT VALUES
RETURNING order_id, order_status;
```

ผลลัพธ์ตัวอย่าง (ค่า `order_id` จะแตกต่างกันตามข้อมูลที่มีอยู่จริงในตาราง):

```
 order_id | order_status
----------+--------------
        7 | pending
(1 row)
```

</details>

---

### แบบฝึกหัดที่ 4

เขียนคำสั่ง `INSERT` เพื่อเพิ่มสินค้าใหม่ชื่อ `'Matcha Latte'` ในหมวดหมู่ `category_id = 2` ราคา `70.00` และให้ `is_active` ใช้ค่า default โดยใช้คีย์เวิร์ด `DEFAULT` อย่างชัดเจน จากนั้นใช้ `RETURNING` เพื่อดึงคอลัมน์ทั้งหมดกลับมา

<details>
<summary>เฉลย</summary>

```sql
INSERT INTO products (category_id, product_name, sku, price, is_active)
VALUES (2, 'Matcha Latte', 'BEV-MAT-001', 70.00, DEFAULT)
RETURNING *;
```

`is_active` จะได้ค่า `TRUE` ซึ่งเป็นค่า default ของคอลัมน์

</details>

---

### แบบฝึกหัดที่ 5

เขียนคำสั่ง `INSERT ... SELECT` เพื่อคัดลอกสินค้าทั้งหมดที่มีราคาสูงกว่า `60.00` จากตาราง `products` ไปยังตาราง `product_sales_summary` โดยใส่ `total_quantity = 0` และ `total_revenue = 0` (สมมติว่ายังไม่มีการขายเลย ใช้ `ON CONFLICT (product_id) DO NOTHING` เพื่อเลี่ยง error หากมีอยู่แล้ว)

<details>
<summary>เฉลย</summary>

```sql
INSERT INTO product_sales_summary (product_id, product_name, total_quantity, total_revenue)
SELECT product_id, product_name, 0, 0
FROM products
WHERE price > 60.00
ON CONFLICT (product_id) DO NOTHING;
```

</details>

---

### แบบฝึกหัดที่ 6

ตาราง `products` มี unique constraint บนคอลัมน์ `sku` เขียนคำสั่ง UPSERT ที่ insert สินค้าด้วย SKU `'BEV-ESP-001'` ราคา `65.00` ถ้ามี SKU นี้อยู่แล้วให้อัปเดตเฉพาะราคา (ไม่แก้ชื่อสินค้า) พร้อม `RETURNING` ราคาใหม่

<details>
<summary>เฉลย</summary>

```sql
INSERT INTO products (category_id, product_name, sku, price)
VALUES (1, 'Espresso', 'BEV-ESP-001', 65.00)
ON CONFLICT (sku) DO UPDATE
SET price = EXCLUDED.price
RETURNING product_id, sku, price;
```

</details>

---

### แบบฝึกหัดที่ 7

จงอธิบายความแตกต่างระหว่าง `ON CONFLICT (column) DO NOTHING` กับ `ON CONFLICT DO NOTHING` (ไม่ระบุคอลัมน์) และบอกว่ากรณีไหนที่ `ON CONFLICT DO NOTHING` แบบไม่ระบุ target **ใช้ไม่ได้เลย**

<details>
<summary>เฉลย</summary>

- `ON CONFLICT (column) DO NOTHING` — ตรวจ conflict เฉพาะจาก unique/exclusion constraint ที่ตรงกับคอลัมน์ที่ระบุเท่านั้น ถ้าตารางมีหลาย unique constraint คำสั่งนี้จะมองข้ามการชนที่มาจาก constraint อื่น
- `ON CONFLICT DO NOTHING` (ไม่ระบุ) — ข้ามการ insert เมื่อชนกับ unique/exclusion constraint **ใดก็ได้** ในตาราง ควบคุมได้น้อยกว่าแต่เขียนง่ายกว่า

กรณีที่ใช้ไม่ได้เลย: เมื่อต้องการทำ `DO UPDATE` (ไม่ใช่ `DO NOTHING`) — `DO UPDATE` **ต้อง**ระบุ conflict target เสมอ (ไม่ว่าจะเป็น `ON CONFLICT (column)` หรือ `ON CONFLICT ON CONSTRAINT name`) เพราะ PostgreSQL ต้องรู้ว่าจะอ้างอิงแถวที่ชนกันจาก constraint ไหนเพื่อนำไป `UPDATE`

</details>

---

### แบบฝึกหัดที่ 8

ระหว่าง `INSERT` แบบ multi-row (batch ละ 1,000 แถว) กับ `COPY` จากไฟล์ CSV จำนวน 500,000 แถว ควรเลือกวิธีไหน และเพราะเหตุใด

<details>
<summary>เฉลย</summary>

ควรเลือก `COPY` เพราะ:

1. `COPY` ใช้ wire protocol แบบ streaming เฉพาะทางที่มี overhead ต่ำกว่าการ parse SQL statement ของ `INSERT` มาก
2. ลด parsing/planning overhead ต่อแถวเมื่อเทียบกับการส่งคำสั่ง `INSERT` หลายพันคำสั่ง (แม้จะเป็น multi-row ก็ตาม)
3. สำหรับข้อมูลระดับ 500,000 แถว ความแตกต่างด้านความเร็วระหว่าง `COPY` กับ multi-row `INSERT` มักมีนัยสำคัญมาก (`COPY` อาจเร็วกว่าหลายเท่า)

ข้อควรระวัง: ถ้าต้องการ UPSERT ระหว่าง bulk load ให้ `COPY` เข้า staging table ก่อน แล้วค่อยใช้ `INSERT ... SELECT ... ON CONFLICT` จาก staging table ไปตารางจริง เพราะ `COPY` แบบมาตรฐานไม่รองรับ `ON CONFLICT`

</details>

---

### แบบฝึกหัดที่ 9

คำสั่งต่อไปนี้จะเกิด error อะไร และเพราะเหตุใด (สมมติว่า `category_id = 5` ไม่มีอยู่จริงในตาราง `categories`)

```sql
INSERT INTO products (category_id, product_name, sku, price)
VALUES (5, 'Test', 'TEST-999', 30.00);
```

<details>
<summary>เฉลย</summary>

จะเกิด **foreign key violation** เพราะคอลัมน์ `category_id` ในตาราง `products` มี `REFERENCES categories(category_id)` และค่า `5` ไม่มีอยู่จริงในตาราง `categories`

```
ERROR:  insert or update on table "products" violates foreign key constraint "products_category_id_fkey"
DETAIL:  Key (category_id)=(5) is not present in table "categories".
```

วิธีแก้: ตรวจสอบก่อนว่ามี `category_id = 5` อยู่จริงหรือไม่ (`SELECT ... FROM categories WHERE category_id = 5`) หรือใช้ `category_id` ที่มีอยู่จริงแทน

</details>

---

### แบบฝึกหัดที่ 10

จงเขียน transaction ที่สร้างออเดอร์ใหม่สำหรับลูกค้า `customer_id = 1` แล้วเพิ่มรายการสินค้า 2 รายการ (`product_id = 1` จำนวน 1 ชิ้น ราคา `58.00` และ `product_id = 3` จำนวน 2 ชิ้น ราคา `60.00`) โดยห่อทั้งหมดด้วย transaction และใช้ `RETURNING` เพื่อดึง `order_id` ที่สร้างใหม่ อธิบายด้วยว่าทำไมต้องห่อด้วย transaction

<details>
<summary>เฉลย</summary>

```sql
BEGIN;

INSERT INTO orders (customer_id, order_status)
VALUES (1, 'pending')
RETURNING order_id;
-- สมมติได้ order_id = 8

INSERT INTO order_items (order_id, product_id, quantity, unit_price)
VALUES
    (8, 1, 1, 58.00),
    (8, 3, 2, 60.00);

COMMIT;
```

**เหตุผลที่ต้องห่อด้วย transaction**: การสร้าง order และ order_items เป็นการกระทำที่สัมพันธ์กันทางตรรกะ — order ที่ไม่มีรายการสินค้าเลยถือเป็นข้อมูลที่ไม่สมบูรณ์ (inconsistent) หากไม่ห่อด้วย transaction และคำสั่งที่สอง (insert order_items) ล้มเหลว (เช่น product_id ไม่มีอยู่จริง หรือเชื่อมต่อขาดกลางทาง) จะเกิด "ออเดอร์กำพร้า" (orphan order) ที่ไม่มี items ค้างอยู่ในระบบ การห่อด้วย `BEGIN...COMMIT` รับประกันว่าทั้งสองคำสั่งจะสำเร็จพร้อมกันหรือไม่สำเร็จเลยทั้งคู่ (atomicity) — ถ้าคำสั่งใดคำสั่งหนึ่งล้มเหลว สามารถ `ROLLBACK` เพื่อยกเลิกทุกอย่างรวมถึง order ที่เพิ่งสร้างได้

</details>

---

## บทถัดไป

เมื่อคุณสามารถเพิ่มข้อมูลเข้าตารางได้อย่างมั่นใจแล้ว ขั้นตอนถัดไปคือการดึงข้อมูลกลับออกมาใช้งาน ไปต่อกันที่ **[Part 010: SELECT พื้นฐาน](./part-010-select-basics.md)** ซึ่งจะสอนการ query ข้อมูล การกรองด้วย `WHERE` การเรียงลำดับด้วย `ORDER BY` และการจำกัดผลลัพธ์ด้วย `LIMIT`/`OFFSET`
