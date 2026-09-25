# Triggers และ Trigger Functions

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับสูง | Part 048

## เป้าหมายการเรียนรู้

เมื่อเรียนจบบทนี้ ผู้เรียนจะสามารถ:

- อธิบายได้ว่า Trigger คืออะไร ทำงานอย่างไร และควรใช้เมื่อไหร่เทียบกับ constraint หรือโค้ดฝั่ง application
- เขียน Trigger Function ด้วย PL/pgSQL ที่คืนค่าชนิด `TRIGGER` และใช้ตัวแปรพิเศษ `NEW`, `OLD`, `TG_OP`, `TG_TABLE_NAME` ได้อย่างถูกต้อง
- เข้าใจความแตกต่างระหว่าง `BEFORE`, `AFTER`, `INSTEAD OF` และระหว่าง `FOR EACH ROW` กับ `FOR EACH STATEMENT`
- สร้าง Trigger สำหรับอัปเดตคอลัมน์ `updated_at` อัตโนมัติ
- สร้างระบบ Audit Trail ที่บันทึกการเปลี่ยนแปลงข้อมูลลงตาราง `audit_log` โดยอัตโนมัติ
- ใช้ Trigger ทำ Cross-table Validation ที่ constraint ธรรมดาทำไม่ได้
- เข้าใจลำดับการทำงานเมื่อมี Trigger หลายตัวบนตารางเดียวกัน
- เปิด/ปิด Trigger ชั่วคราว และรู้ข้อควรระวังเรื่อง performance ของ Trigger
- ออกแบบและสร้างระบบรักษาความถูกต้องของสต๊อกสินค้า (stock integrity) ด้วย Trigger แบบมืออาชีพ

---

## เตรียมข้อมูล

บทนี้ใช้ฐานข้อมูล e-commerce ชุดเดิมที่ใช้ตลอดทั้งหลักสูตร พร้อมเพิ่มตาราง `audit_log` สำหรับเก็บประวัติการเปลี่ยนแปลงข้อมูล และคอลัมน์ `updated_at` ในตาราง `products` สำหรับสาธิต Trigger ที่อัปเดตเวลาการแก้ไขอัตโนมัติ

```sql
-- ล้างของเก่า (ถ้ามี) เพื่อให้รันซ้ำได้
DROP TABLE IF EXISTS audit_log CASCADE;
DROP TABLE IF EXISTS order_items CASCADE;
DROP TABLE IF EXISTS orders CASCADE;
DROP TABLE IF EXISTS customers CASCADE;
DROP TABLE IF EXISTS products CASCADE;
DROP TABLE IF EXISTS suppliers CASCADE;
DROP TABLE IF EXISTS categories CASCADE;

-- ตารางหมวดหมู่สินค้า (รองรับหมวดหมู่ย่อยแบบ self-reference)
CREATE TABLE categories (
    category_id         SERIAL PRIMARY KEY,
    category_name       VARCHAR(100) NOT NULL,
    parent_category_id  INTEGER REFERENCES categories(category_id)
);

-- ตารางซัพพลายเออร์
CREATE TABLE suppliers (
    supplier_id    SERIAL PRIMARY KEY,
    supplier_name  VARCHAR(150) NOT NULL,
    country        VARCHAR(60)
);

-- ตารางสินค้า (มี updated_at สำหรับสาธิต trigger)
CREATE TABLE products (
    product_id      SERIAL PRIMARY KEY,
    product_name    VARCHAR(150) NOT NULL,
    category_id     INTEGER REFERENCES categories(category_id),
    supplier_id     INTEGER REFERENCES suppliers(supplier_id),
    unit_price      NUMERIC(10,2) NOT NULL,
    stock_quantity  INTEGER NOT NULL DEFAULT 0,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ตารางลูกค้า
CREATE TABLE customers (
    customer_id   SERIAL PRIMARY KEY,
    first_name    VARCHAR(60) NOT NULL,
    last_name     VARCHAR(60) NOT NULL,
    email         VARCHAR(150) UNIQUE,
    country       VARCHAR(60),
    signup_date   DATE NOT NULL DEFAULT CURRENT_DATE
);

-- ตารางคำสั่งซื้อ
CREATE TABLE orders (
    order_id      SERIAL PRIMARY KEY,
    customer_id   INTEGER REFERENCES customers(customer_id),
    order_date    TIMESTAMPTZ NOT NULL DEFAULT now(),
    status        VARCHAR(20) NOT NULL DEFAULT 'pending',
    ship_country  VARCHAR(60)
);

-- ตารางรายการสินค้าในคำสั่งซื้อ
CREATE TABLE order_items (
    order_item_id  SERIAL PRIMARY KEY,
    order_id       INTEGER REFERENCES orders(order_id),
    product_id     INTEGER REFERENCES products(product_id),
    quantity       INTEGER NOT NULL CHECK (quantity > 0),
    unit_price     NUMERIC(10,2) NOT NULL
);

-- ตาราง audit log สำหรับเก็บประวัติการเปลี่ยนแปลงข้อมูล (ใช้กับ Trigger ตลอดทั้งบท)
CREATE TABLE audit_log (
    audit_id     SERIAL PRIMARY KEY,
    table_name   VARCHAR(60),
    operation    VARCHAR(10),
    row_id       INTEGER,
    changed_at   TIMESTAMPTZ DEFAULT now(),
    changed_by   TEXT DEFAULT current_user,
    old_data     JSONB,
    new_data     JSONB
);
```

### ข้อมูลตัวอย่าง (Seed Data)

```sql
-- categories: 10 หมวดหมู่ (มีหมวดหมู่ย่อย)
INSERT INTO categories (category_name, parent_category_id) VALUES
('อิเล็กทรอนิกส์',        NULL),
('คอมพิวเตอร์และแล็ปท็อป', 1),
('โทรศัพท์มือถือ',         1),
('เครื่องใช้ไฟฟ้าในบ้าน',   NULL),
('เครื่องครัว',             4),
('เสื้อผ้าแฟชั่น',          NULL),
('เสื้อผ้าผู้ชาย',          6),
('เสื้อผ้าผู้หญิง',         6),
('หนังสือ',                NULL),
('ของเล่นและงานอดิเรก',    NULL);

-- suppliers: 8 ซัพพลายเออร์
INSERT INTO suppliers (supplier_name, country) VALUES
('Bangkok Tech Distribution', 'Thailand'),
('Global Electronics Co.',    'China'),
('Nordic Home Living',        'Sweden'),
('Sakura Appliances',         'Japan'),
('Everest Textile Group',     'Vietnam'),
('Siam Book House',           'Thailand'),
('PlayWorld Toys Ltd.',       'Malaysia'),
('EuroFashion Trading',       'Italy');

-- products: 20 สินค้า
INSERT INTO products (product_name, category_id, supplier_id, unit_price, stock_quantity, is_active) VALUES
('โน้ตบุ๊ก UltraBook 14"',        2, 1, 24900.00, 15, true),
('โน้ตบุ๊ก Gamer X15',            2, 2, 39900.00, 8,  true),
('เมาส์ไร้สาย ErgoClick',         2, 1, 590.00,   120, true),
('คีย์บอร์ดกลไก TypeMaster',      2, 2, 1590.00,  60,  true),
('สมาร์ตโฟน Nova 12',             3, 2, 18900.00, 25,  true),
('สมาร์ตโฟน Nova 12 Lite',        3, 2, 11900.00, 40,  true),
('เคสกันกระแทก Nova 12',          3, 1, 290.00,   200, true),
('หูฟังไร้สาย SoundBeat Pro',     3, 2, 2490.00,  75,  true),
('หม้อทอดไร้น้ำมัน CrispAir 5L',  5, 4, 2990.00,  30,  true),
('เครื่องปั่นน้ำผลไม้ BlendGo',   5, 4, 1290.00,  45,  true),
('หม้อหุงข้าว SmartCook 1.8L',    5, 4, 1890.00,  50,  true),
('พัดลมตั้งพื้น CoolBreeze',      4, 3, 1190.00,  35,  true),
('เสื้อยืดผ้าคอตตอน Basic',       7, 5, 259.00,   300, true),
('กางเกงยีนส์ Slim Fit',          7, 5, 890.00,   150, true),
('เดรสลายดอกไม้ Summer',          8, 8, 1290.00,  60,  true),
('เสื้อคาร์ดิแกนไหมพรม',          8, 8, 1590.00,  40,  true),
('หนังสือ "เริ่มต้นเขียนโค้ด"',   9, 6, 350.00,   80,  true),
('หนังสือนิยาย "แสงจันทร์สีคราม"', 9, 6, 285.00,   65,  true),
('ตัวต่อเลโก้ SpaceBuilder',      10, 7, 1990.00,  20,  true),
('โดรนบังคับ SkyMini',            10, 7, 3290.00,  12,  false);

-- customers: 15 ลูกค้า
INSERT INTO customers (first_name, last_name, email, country, signup_date) VALUES
('สมชาย',   'ใจดี',       'somchai.j@example.com',  'Thailand',  '2023-01-15'),
('สุภาพร',  'แสงทอง',     'supaporn.s@example.com', 'Thailand',  '2023-02-20'),
('ธนกร',    'วงศ์สุวรรณ', 'thanakorn.w@example.com','Thailand',  '2023-03-05'),
('Kenji',   'Tanaka',     'kenji.t@example.com',    'Japan',     '2023-03-18'),
('Mei',     'Chen',       'mei.chen@example.com',   'China',     '2023-04-02'),
('Somying', 'Rattana',    'somying.r@example.com',  'Thailand',  '2023-04-22'),
('John',    'Smith',      'john.smith@example.com', 'USA',       '2023-05-10'),
('Anna',    'Kowalski',   'anna.k@example.com',     'Poland',    '2023-05-29'),
('ปิยะพงษ์', 'ศรีสุข',     'piyapong.s@example.com', 'Thailand',  '2023-06-14'),
('Linh',    'Nguyen',     'linh.nguyen@example.com','Vietnam',   '2023-07-01'),
('Sara',    'Johansson',  'sara.j@example.com',     'Sweden',    '2023-07-19'),
('วราภรณ์',  'ทองแท้',     'waraporn.t@example.com', 'Thailand',  '2023-08-08'),
('Marco',   'Rossi',      'marco.rossi@example.com','Italy',     '2023-08-25'),
('Fatima',  'Ali',        'fatima.ali@example.com', 'UAE',       '2023-09-11'),
('เอกชัย',   'บุญมา',      'ekachai.b@example.com',  'Thailand',  '2023-09-30');

-- orders: 15 คำสั่งซื้อ
INSERT INTO orders (customer_id, order_date, status, ship_country) VALUES
(1,  '2024-01-05 10:15:00+07', 'delivered',  'Thailand'),
(2,  '2024-01-08 14:30:00+07', 'delivered',  'Thailand'),
(3,  '2024-01-12 09:00:00+07', 'shipped',    'Thailand'),
(4,  '2024-01-15 11:45:00+09', 'delivered',  'Japan'),
(5,  '2024-01-20 16:20:00+08', 'delivered',  'China'),
(6,  '2024-02-02 13:10:00+07', 'pending',    'Thailand'),
(7,  '2024-02-05 08:30:00-05', 'delivered',  'USA'),
(8,  '2024-02-10 19:00:00+01', 'cancelled',  'Poland'),
(9,  '2024-02-14 10:00:00+07', 'shipped',    'Thailand'),
(10, '2024-02-18 15:40:00+07', 'delivered',  'Vietnam'),
(1,  '2024-03-01 12:00:00+07', 'pending',    'Thailand'),
(11, '2024-03-04 09:25:00+01', 'delivered',  'Sweden'),
(12, '2024-03-09 17:15:00+07', 'shipped',    'Thailand'),
(13, '2024-03-12 10:50:00+01', 'delivered',  'Italy'),
(3,  '2024-03-20 14:00:00+07', 'pending',    'Thailand');

-- order_items: รายการสินค้าในแต่ละคำสั่งซื้อ
INSERT INTO order_items (order_id, product_id, quantity, unit_price) VALUES
(1, 1, 1, 24900.00),
(1, 3, 2, 590.00),
(2, 5, 1, 18900.00),
(2, 7, 1, 290.00),
(3, 2, 1, 39900.00),
(4, 8, 2, 2490.00),
(5, 6, 1, 11900.00),
(5, 7, 2, 290.00),
(6, 13, 3, 259.00),
(7, 9, 1, 2990.00),
(7, 10, 1, 1290.00),
(8, 15, 1, 1290.00),
(9, 4, 1, 1590.00),
(9, 3, 3, 590.00),
(10, 17, 2, 350.00),
(11, 19, 1, 1990.00),
(12, 11, 1, 1890.00),
(13, 14, 2, 890.00),
(14, 16, 1, 1590.00),
(15, 18, 2, 285.00);
```

> ตาราง `audit_log` เริ่มต้นเป็นตารางว่าง เราจะเห็นแถวถูกเติมเข้ามาอัตโนมัติเมื่อสร้าง Trigger ในหัวข้อถัดไป

---

## Step 471: Trigger คืออะไร

**Trigger** คือกลไกของฐานข้อมูลที่ทำให้ "โค้ดบางอย่าง" (Trigger Function) ทำงาน **โดยอัตโนมัติ** เมื่อมีเหตุการณ์ (event) เกิดขึ้นกับตารางหรือ view เช่น `INSERT`, `UPDATE`, `DELETE` หรือ `TRUNCATE` โดยที่แอปพลิเคชันฝั่งไคลเอนต์ไม่ต้องเรียกใช้เอง

พูดง่าย ๆ คือ Trigger คือ "ผู้เฝ้าดูอยู่เบื้องหลัง" ของตาราง ทุกครั้งที่มีการเปลี่ยนแปลงข้อมูลตรงกับเงื่อนไขที่กำหนด ฐานข้อมูลจะรันฟังก์ชันที่ผูกไว้ให้เองโดยอัตโนมัติ ไม่ว่าการเปลี่ยนแปลงนั้นจะมาจาก `psql`, แอปพลิเคชัน, สคริปต์ ETL หรือแม้แต่ Trigger ตัวอื่นก็ตาม

### ทำไมต้องใช้ Trigger

ลองนึกภาพสถานการณ์ต่อไปนี้:

1. ทุกครั้งที่มีการแก้ไขราคาสินค้า อยากให้บันทึก timestamp ของการแก้ไขล่าสุดไว้อัตโนมัติ (`updated_at`)
2. ทุกครั้งที่มีการเพิ่ม/แก้ไข/ลบข้อมูลสินค้า อยากเก็บ "ประวัติ" ไว้ตรวจสอบย้อนหลังว่าใครแก้อะไรเมื่อไหร่ (audit trail)
3. ก่อนจะเพิ่มรายการสั่งซื้อ อยากตรวจสอบว่าสินค้านั้นยัง `is_active = true` และมีสต๊อกเพียงพอหรือไม่ — logic นี้ต้องเช็คข้าม 2 ตาราง ซึ่ง `CHECK constraint` ธรรมดาทำไม่ได้ (CHECK constraint มองเห็นแค่แถวตัวเอง ไม่สามารถ query ตารางอื่นได้)
4. เมื่อมีการสั่งซื้อสินค้า อยากให้สต๊อกสินค้าลดลงอัตโนมัติ โดยไม่ต้องพึ่งพาแอปพลิเคชันทุกตัวที่เขียนโค้ดแยกกัน

ทั้งหมดนี้คือหน้าที่ของ Trigger

### Trigger ต่างจาก Constraint อย่างไร

| คุณสมบัติ | Constraint (CHECK, FK, UNIQUE) | Trigger |
|---|---|---|
| ความซับซ้อนของ logic | ตรวจสอบแถวเดียว/นิพจน์ง่าย ๆ | เขียน logic ซับซ้อนได้เต็มที่ (query ตารางอื่น, loop, exception) |
| Cross-table validation | ทำไม่ได้โดยตรง (ยกเว้น FK) | ทำได้ |
| Side effect (เช่น เขียนลง audit_log) | ทำไม่ได้ | ทำได้ |
| Performance | เร็วกว่า เพราะ optimizer เข้าใจและใช้ประโยชน์ได้ | ช้ากว่าเล็กน้อย เพราะต้องรันฟังก์ชันทุกครั้ง |
| ความชัดเจน | อ่านง่าย เห็นในโครงสร้างตารางทันที | ต้องไปดู `pg_trigger` แยกต่างหาก อาจ "ซ่อน" logic ไว้ |

**หลักการเลือกใช้**: ถ้า constraint ธรรมดา (`CHECK`, `NOT NULL`, `UNIQUE`, `FOREIGN KEY`) ตอบโจทย์ได้ ให้ใช้ constraint ก่อนเสมอ เพราะเร็วกว่าและ optimizer เข้าใจได้ดีกว่า ใช้ Trigger เมื่อ logic ซับซ้อนเกินกว่าที่ constraint จะทำได้ เช่น ต้องเช็คข้ามตาราง ต้องเขียนผลข้างเคียง (side effect) หรือต้องคำนวณค่าที่ผันแปรตามเวลา

### ประเภทของ Trigger ใน PostgreSQL

PostgreSQL รองรับ Trigger บน:

- **Table** — ตารางปกติ รองรับทุก event (`INSERT`, `UPDATE`, `DELETE`, `TRUNCATE`)
- **View** — ใช้ `INSTEAD OF` trigger เพื่อทำให้ view ที่แก้ไขไม่ได้โดยตรง (non-updatable view) สามารถรองรับ `INSERT`/`UPDATE`/`DELETE` ได้
- **Foreign table** — ตารางจาก foreign data wrapper

เนื้อหาในบทนี้จะเน้นที่ Trigger บนตารางปกติ ซึ่งเป็นรูปแบบที่ใช้งานมากที่สุดในการพัฒนาระบบจริง

---

## Step 472: Trigger Function — ฟังก์ชันพิเศษที่คืนค่า TRIGGER

Trigger จะทำงานไม่ได้ถ้าไม่มี **Trigger Function** ผูกอยู่ Trigger Function คือฟังก์ชันที่เขียนด้วยภาษา procedural (ปกติคือ PL/pgSQL) ที่มีคุณสมบัติพิเศษดังนี้:

1. **ต้องไม่รับพารามิเตอร์แบบปกติ** — ฟังก์ชันประกาศเป็น `()` เสมอ (ถึงแม้จะส่ง argument ผ่าน `CREATE TRIGGER ... EXECUTE FUNCTION fn(arg1, arg2)` ได้ ก็จะไปอยู่ใน `TG_ARGV` ไม่ใช่พารามิเตอร์ปกติ)
2. **ต้องคืนค่าชนิด `TRIGGER`** — ไม่ใช่ `INTEGER`, `VOID` หรือชนิดข้อมูลทั่วไป
3. **เข้าถึงตัวแปรพิเศษ (special variables)** ที่ PostgreSQL เตรียมไว้ให้อัตโนมัติภายใน trigger context เท่านั้น

### ตัวแปรพิเศษที่สำคัญ

| ตัวแปร | ชนิด | ความหมาย |
|---|---|---|
| `NEW` | RECORD | แถวข้อมูล **ใหม่** หลังการเปลี่ยนแปลง ใช้ได้กับ `INSERT` และ `UPDATE` (ไม่มีใน `DELETE`) |
| `OLD` | RECORD | แถวข้อมูล **เดิม** ก่อนการเปลี่ยนแปลง ใช้ได้กับ `UPDATE` และ `DELETE` (ไม่มีใน `INSERT`) |
| `TG_OP` | TEXT | ชนิดของ operation: `'INSERT'`, `'UPDATE'`, `'DELETE'`, หรือ `'TRUNCATE'` |
| `TG_TABLE_NAME` | NAME | ชื่อตารางที่ trigger ผูกอยู่ |
| `TG_TABLE_SCHEMA` | NAME | ชื่อ schema ของตารางนั้น |
| `TG_WHEN` | TEXT | `'BEFORE'`, `'AFTER'`, หรือ `'INSTEAD OF'` |
| `TG_LEVEL` | TEXT | `'ROW'` หรือ `'STATEMENT'` |
| `TG_NAME` | NAME | ชื่อของ trigger เอง |
| `TG_NARGS` | INTEGER | จำนวน argument ที่ส่งมาตอน `CREATE TRIGGER` |
| `TG_ARGV[]` | TEXT[] | argument ที่ส่งมาตอน `CREATE TRIGGER` (index เริ่มที่ 0) |
| `TG_RELID` | OID | OID ของตารางที่ trigger ผูกอยู่ |

### โครงสร้างพื้นฐานของ Trigger Function

```sql
CREATE OR REPLACE FUNCTION trg_demo_function()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    RAISE NOTICE 'Trigger "%" fired: TG_OP=%, TG_TABLE_NAME=%, TG_WHEN=%, TG_LEVEL=%',
        TG_NAME, TG_OP, TG_TABLE_NAME, TG_WHEN, TG_LEVEL;

    IF TG_OP = 'INSERT' THEN
        RAISE NOTICE 'NEW row: %', NEW;
    ELSIF TG_OP = 'UPDATE' THEN
        RAISE NOTICE 'OLD row: % -> NEW row: %', OLD, NEW;
    ELSIF TG_OP = 'DELETE' THEN
        RAISE NOTICE 'OLD row: %', OLD;
    END IF;

    -- สำหรับ BEFORE trigger ระดับ ROW ต้อง RETURN ค่าที่จะเขียนจริง
    IF TG_OP = 'DELETE' THEN
        RETURN OLD;
    ELSE
        RETURN NEW;
    END IF;
END;
$$;
```

### ความหมายของค่าที่ `RETURN`

ค่าที่ trigger function คืนกลับมามีผลต่างกันตาม timing:

- **`BEFORE` trigger ระดับ ROW**: ค่าที่ `RETURN` จะถูกใช้เป็นแถวจริงที่จะถูกเขียนลงตาราง
  - `RETURN NEW;` (หรือแก้ไขค่าใน `NEW` ก่อน return) — ดำเนินการต่อด้วยค่าที่ (อาจถูกแก้ไขแล้ว)
  - `RETURN NULL;` — **ยกเลิก** operation สำหรับแถวนั้นทันที (แถวจะไม่ถูก insert/update/delete)
- **`AFTER` trigger ระดับ ROW**: ค่าที่ `RETURN` **ไม่มีผลใด ๆ** ต่อข้อมูล (เพราะข้อมูลถูกเขียนไปแล้ว) แต่ตามธรรมเนียมนิยมคืนค่า `NEW` หรือ `OLD` หรือ `NULL` ก็ได้
- **`STATEMENT`-level trigger**: ไม่มี `NEW`/`OLD` ให้ใช้ (ยกเว้นผ่าน transition table) จึง `RETURN NULL;` เสมอ
- **`INSTEAD OF` trigger บน view**: `RETURN NULL;` หมายถึงไม่ให้ดำเนินการใด ๆ, ค่าอื่นถือว่าดำเนินการสำเร็จ

> ข้อควรจำ: `NEW` และ `OLD` เป็นตัวแปรชนิด `RECORD` — เข้าถึงคอลัมน์ด้วย dot notation เช่น `NEW.product_name`, `OLD.unit_price` และสามารถแก้ไขค่าใน `NEW` ได้ก่อน `RETURN` (มีผลเฉพาะใน `BEFORE` trigger)

---

## Step 473: CREATE TRIGGER Syntax

รูปแบบคำสั่งพื้นฐาน (แบบย่อของ syntax เต็มใน PostgreSQL 16/17):

```sql
CREATE [ OR REPLACE ] TRIGGER trigger_name
    { BEFORE | AFTER | INSTEAD OF } { event [ OR ... ] }
    ON table_name
    [ REFERENCING { { OLD | NEW } TABLE [ AS ] transition_relation_name } [ ... ] ]
    [ FOR [ EACH ] { ROW | STATEMENT } ]
    [ WHEN ( condition ) ]
    EXECUTE { FUNCTION | PROCEDURE } function_name ( arguments )
```

โดย `event` คือ `INSERT`, `UPDATE [ OF column_name [, ...] ]`, `DELETE`, หรือ `TRUNCATE` (รวมกันด้วย `OR` ได้ เช่น `INSERT OR UPDATE OR DELETE`)

> หมายเหตุ: `CREATE OR REPLACE TRIGGER` รองรับตั้งแต่ PostgreSQL 14 เป็นต้นไป ส่วน `EXECUTE PROCEDURE` เป็นชื่อเก่าที่ใช้แทน `EXECUTE FUNCTION` ได้เหมือนกัน (เก็บไว้เพื่อความเข้ากันได้ย้อนหลัง) ในโค้ดใหม่ควรใช้ `EXECUTE FUNCTION`

### BEFORE vs AFTER vs INSTEAD OF

**`BEFORE`** — ทำงาน **ก่อน** ที่การเปลี่ยนแปลงจะถูกเขียนลงตารางจริง เหมาะกับ:
- แก้ไขค่าที่จะถูกบันทึก (เช่น normalize ข้อมูล, set `updated_at`)
- validate ข้อมูลก่อนบันทึก แล้ว `RAISE EXCEPTION` เพื่อยกเลิกถ้าไม่ผ่าน
- ยกเลิก operation แบบเงียบ ๆ ด้วย `RETURN NULL;` (โดยไม่ error)

**`AFTER`** — ทำงาน **หลัง** จากการเปลี่ยนแปลงถูกเขียนลงตารางแล้ว (แต่ยังอยู่ใน transaction เดียวกัน ยังไม่ commit) เหมาะกับ:
- เขียน audit log (เพราะต้องการบันทึกว่า "เกิดอะไรขึ้นแล้ว")
- ทำ side effect กับตารางอื่น เช่น อัปเดตสต๊อกสินค้า, ส่ง notification
- แก้ไข `NEW`/`OLD` ใน `AFTER` trigger จะ**ไม่มีผล**ต่อข้อมูลที่บันทึกแล้ว เพราะเขียนไปแล้ว

**`INSTEAD OF`** — ใช้ได้เฉพาะกับ **view** เท่านั้น (และต้องเป็นระดับ `ROW`) ทำงาน **แทนที่** operation เดิมทั้งหมด เหมาะกับ:
- ทำให้ view ที่ query จากหลายตาราง (ซึ่งปกติแก้ไขไม่ได้โดยตรง) สามารถรองรับ `INSERT`/`UPDATE`/`DELETE` ได้ โดย trigger จะเขียน logic กระจายการเปลี่ยนแปลงไปยังตารางจริงเบื้องหลังเอง

### ตัวอย่าง: Trigger พื้นฐานที่สุด

```sql
-- 1) สร้าง trigger function อย่างง่าย
CREATE OR REPLACE FUNCTION trg_log_product_change()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    RAISE NOTICE '[%] operation=% on product_id=%',
        TG_WHEN, TG_OP, COALESCE(NEW.product_id, OLD.product_id);
    RETURN NEW;
END;
$$;

-- 2) ผูก trigger function เข้ากับตาราง products
CREATE TRIGGER trg_products_before_insert
    BEFORE INSERT ON products
    FOR EACH ROW
    EXECUTE FUNCTION trg_log_product_change();

-- 3) ทดสอบ
INSERT INTO products (product_name, category_id, supplier_id, unit_price, stock_quantity)
VALUES ('เม้าส์แพดยางกันลื่น', 2, 1, 190.00, 100);
-- NOTICE: [BEFORE] operation=INSERT on product_id=<NULL เพราะยังไม่ insert>
```

> สังเกตว่าใน `BEFORE INSERT` ค่า `NEW.product_id` ยังเป็น `NULL` เพราะ `SERIAL`/`sequence` ยังไม่ได้ถูกกำหนดค่าจนกว่าจะถึงขั้นตอนเขียนจริง (ยกเว้นกรณีใช้ `IDENTITY`/`DEFAULT` ที่คำนวณค่าไว้ล่วงหน้าก่อนเข้า trigger — พฤติกรรมจริงคือ `DEFAULT` ของคอลัมน์จะถูกคำนวณ**ก่อน** `BEFORE` trigger ทำงาน ดังนั้น `NEW.product_id` มักจะมีค่าแล้วในทางปฏิบัติ แต่ไม่ควรพึ่งพาลำดับนี้กับ generated column ที่ซับซ้อน)

ลบ trigger ทดลองนี้ทิ้งก่อนไปหัวข้อถัดไป เพื่อไม่ให้ NOTICE รบกวนตัวอย่างอื่น:

```sql
DROP TRIGGER trg_products_before_insert ON products;
```

### การดูรายการ Trigger ที่มีอยู่

```sql
-- ผ่าน information_schema (มาตรฐาน ANSI SQL)
SELECT trigger_name, event_manipulation, event_object_table, action_timing
FROM information_schema.triggers
WHERE event_object_table = 'products'
ORDER BY trigger_name;

-- ผ่าน catalog เฉพาะของ PostgreSQL (ละเอียดกว่า)
SELECT tgname, tgrelid::regclass AS table_name, tgenabled,
       pg_get_triggerdef(oid) AS definition
FROM pg_trigger
WHERE tgrelid = 'products'::regclass
  AND NOT tgisinternal   -- กรอง trigger ภายในที่ระบบสร้างเอง (เช่นจาก FK)
ORDER BY tgname;
```

---

## Step 474: Trigger ระดับ ROW เทียบกับ STATEMENT

`FOR EACH ROW` และ `FOR EACH STATEMENT` เป็นตัวกำหนด **ความถี่** ในการทำงานของ trigger:

- **`FOR EACH ROW`** — ทำงาน **หนึ่งครั้งต่อหนึ่งแถว** ที่ถูกกระทบ ถ้า `UPDATE` กระทบ 1,000 แถว trigger จะทำงาน 1,000 ครั้ง มี `NEW`/`OLD` ให้ใช้งานเสมอ (ยกเว้น `TRUNCATE` ที่ไม่รองรับ `FOR EACH ROW`)
- **`FOR EACH STATEMENT`** (ค่า default ถ้าไม่ระบุ) — ทำงาน **หนึ่งครั้งต่อหนึ่งคำสั่ง SQL** ไม่ว่าคำสั่งนั้นจะกระทบกี่แถวก็ตาม (แม้กระทบ 0 แถว ก็ยังทำงาน 1 ครั้ง) ไม่มี `NEW`/`OLD` ให้ใช้โดยตรง (ต้องใช้ transition table แทน)

### ตัวอย่างเปรียบเทียบ

```sql
-- Trigger function สำหรับสาธิตระดับ ROW
CREATE OR REPLACE FUNCTION trg_row_level_demo()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    RAISE NOTICE '[ROW-LEVEL] % on product_id=%', TG_OP, COALESCE(NEW.product_id, OLD.product_id);
    RETURN NULL; -- ค่า return ไม่มีผลใน AFTER trigger
END;
$$;

CREATE TRIGGER trg_products_row_demo
    AFTER UPDATE ON products
    FOR EACH ROW
    EXECUTE FUNCTION trg_row_level_demo();

-- Trigger function สำหรับสาธิตระดับ STATEMENT
CREATE OR REPLACE FUNCTION trg_statement_level_demo()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    RAISE NOTICE '[STATEMENT-LEVEL] % fired once for the whole statement', TG_OP;
    RETURN NULL;
END;
$$;

CREATE TRIGGER trg_products_statement_demo
    AFTER UPDATE ON products
    FOR EACH STATEMENT
    EXECUTE FUNCTION trg_statement_level_demo();

-- ทดสอบ: UPDATE ที่กระทบหลายแถวพร้อมกัน
UPDATE products SET stock_quantity = stock_quantity - 1
WHERE category_id = 2;   -- กระทบ 4 แถว (คอมพิวเตอร์และแล็ปท็อป)

/*
ผลลัพธ์ NOTICE ที่ได้ (ลำดับโดยประมาณ):
[ROW-LEVEL] UPDATE on product_id=1
[ROW-LEVEL] UPDATE on product_id=2
[ROW-LEVEL] UPDATE on product_id=3
[ROW-LEVEL] UPDATE on product_id=4
[STATEMENT-LEVEL] UPDATE fired once for the whole statement
*/
```

จะเห็นว่า `FOR EACH ROW` ทำงาน 4 ครั้ง (ตามจำนวนแถวที่ถูกอัปเดต) แต่ `FOR EACH STATEMENT` ทำงานเพียง **1 ครั้ง** เท่านั้น ไม่ว่าจะกระทบกี่แถวก็ตาม (ลำดับจริงคือ `STATEMENT`-level trigger ที่เป็น `AFTER` จะทำงานหลังจาก `ROW`-level ทุกแถวทำงานเสร็จแล้ว)

### Transition Table — ใช้ NEW/OLD ระดับ STATEMENT (PostgreSQL 10+)

ถ้าอยากรู้ "ภาพรวม" ของแถวทั้งหมดที่ถูกกระทบในระดับ STATEMENT (เช่น สรุปยอดรวมของการเปลี่ยนแปลง) ใช้ `REFERENCING ... TABLE` เพื่อสร้าง transition table ชั่วคราวที่รวบรวมทุกแถวที่เปลี่ยน:

```sql
CREATE OR REPLACE FUNCTION trg_summarize_price_update()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
DECLARE
    v_row_count   INTEGER;
    v_total_delta NUMERIC;
BEGIN
    SELECT COUNT(*), SUM(n.unit_price - o.unit_price)
    INTO v_row_count, v_total_delta
    FROM old_rows o
    JOIN new_rows n ON n.product_id = o.product_id;

    RAISE NOTICE 'สรุป: อัปเดตราคา % แถว รวมส่วนต่างราคา = %', v_row_count, v_total_delta;
    RETURN NULL;
END;
$$;

CREATE TRIGGER trg_products_price_summary
    AFTER UPDATE ON products
    REFERENCING OLD TABLE AS old_rows NEW TABLE AS new_rows
    FOR EACH STATEMENT
    WHEN (pg_trigger_depth() = 0)  -- ป้องกันการนับซ้ำถ้ามี trigger อื่น update ซ้อน
    EXECUTE FUNCTION trg_summarize_price_update();

-- ทดสอบ: ปรับราคาสินค้าทุกชิ้นในหมวด "เครื่องครัว" ขึ้น 5%
UPDATE products
SET unit_price = ROUND(unit_price * 1.05, 2)
WHERE category_id = 5;
-- NOTICE: สรุป: อัปเดตราคา 3 แถว รวมส่วนต่างราคา = 258.50 (ตัวเลขตัวอย่าง)
```

Transition table (`old_rows`, `new_rows`) ใช้ได้เฉพาะกับ `FOR EACH STATEMENT` และเป็นเทคนิคสำคัญเวลาต้องการทำ bulk summary โดยไม่เสีย performance จากการ loop ทีละแถว

ลบ trigger สาธิตทั้งหมดก่อนไปต่อ:

```sql
DROP TRIGGER trg_products_row_demo        ON products;
DROP TRIGGER trg_products_statement_demo  ON products;
DROP TRIGGER trg_products_price_summary   ON products;
```

---

## Step 475: ตัวอย่างจริง — BEFORE UPDATE Trigger สำหรับอัปเดต updated_at อัตโนมัติ

นี่คือหนึ่งใน pattern ที่ใช้บ่อยที่สุดในระบบจริง: ทุกครั้งที่มีการ `UPDATE` แถวในตาราง อยากให้คอลัมน์ `updated_at` ถูกตั้งเป็นเวลาปัจจุบันโดยอัตโนมัติ โดยไม่ต้องพึ่งพาให้แอปพลิเคชันจำ set เอง

### สร้าง Trigger Function ที่ใช้ซ้ำได้กับทุกตาราง

```sql
CREATE OR REPLACE FUNCTION set_updated_at()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    -- ป้องกันการเขียนซ้ำโดยไม่จำเป็น ถ้าไม่มีคอลัมน์ใดเปลี่ยนแปลงจริง
    IF NEW IS NOT DISTINCT FROM OLD THEN
        RETURN NEW;
    END IF;

    NEW.updated_at := now();
    RETURN NEW;
END;
$$;

COMMENT ON FUNCTION set_updated_at() IS
    'ฟังก์ชันสำหรับ BEFORE UPDATE trigger: อัปเดตคอลัมน์ updated_at เป็นเวลาปัจจุบันอัตโนมัติ';
```

### ผูก Trigger เข้ากับตาราง products

```sql
CREATE TRIGGER trg_products_set_updated_at
    BEFORE UPDATE ON products
    FOR EACH ROW
    EXECUTE FUNCTION set_updated_at();
```

### สาธิตก่อน/หลัง

```sql
-- ดูค่า updated_at ก่อนแก้ไข
SELECT product_id, product_name, unit_price, updated_at
FROM products
WHERE product_id = 1;
/*
 product_id |      product_name       | unit_price |          updated_at
------------+--------------------------+------------+-------------------------------
          1 | โน้ตบุ๊ก UltraBook 14"   |   24900.00 | 2026-09-25 10:00:00.123456+07
*/

-- รอสักครู่แล้วแก้ไขราคา
UPDATE products
SET unit_price = 23900.00
WHERE product_id = 1;

-- ตรวจสอบอีกครั้ง
SELECT product_id, product_name, unit_price, updated_at
FROM products
WHERE product_id = 1;
/*
 product_id |      product_name       | unit_price |          updated_at
------------+--------------------------+------------+-------------------------------
          1 | โน้ตบุ๊ก UltraBook 14"   |   23900.00 | 2026-09-25 10:05:42.987654+07
                                                       ^^^^ เปลี่ยนเป็นเวลาปัจจุบันอัตโนมัติ
*/
```

จะเห็นว่าเราไม่ต้องเขียน `SET updated_at = now()` ในคำสั่ง `UPDATE` เอง — trigger จัดการให้อัตโนมัติ และป้องกันกรณี "ลืม set" ซึ่งเป็นบั๊กที่พบได้บ่อยในระบบที่พึ่งพา application code ล้วน ๆ

### ทำไมต้องเช็ค `IS NOT DISTINCT FROM`

ถ้า `UPDATE` แถวโดยไม่ได้เปลี่ยนค่าอะไรเลย (เช่น `UPDATE products SET unit_price = unit_price WHERE product_id = 1;`) การเช็คนี้จะช่วยไม่ให้ `updated_at` เปลี่ยนไปโดยไม่จำเป็น ซึ่งสำคัญมากถ้ามีระบบอื่น (เช่น cache invalidation, sync job) ที่อาศัย `updated_at` เป็นตัวบอกว่า "ข้อมูลเปลี่ยนจริงหรือไม่"

> `IS NOT DISTINCT FROM` ต่างจาก `<>` ตรงที่มันจัดการค่า `NULL` ได้ถูกต้อง (`NULL <> NULL` ให้ผลเป็น `NULL` ไม่ใช่ `true`/`false` แต่ `NULL IS NOT DISTINCT FROM NULL` ให้ผลเป็น `false` อย่างถูกต้อง)

### รูปแบบทั่วไปสำหรับใช้กับหลายตาราง

เนื่องจาก `set_updated_at()` ไม่ได้ผูกกับตารางใดตารางหนึ่ง จึงนำไปใช้ซ้ำกับตารางอื่นที่มีคอลัมน์ `updated_at` ได้ทันที เพียงเพิ่มคอลัมน์และสร้าง trigger ใหม่ เช่น ถ้าจะเพิ่ม `updated_at` ให้ `orders`:

```sql
ALTER TABLE orders ADD COLUMN updated_at TIMESTAMPTZ NOT NULL DEFAULT now();

CREATE TRIGGER trg_orders_set_updated_at
    BEFORE UPDATE ON orders
    FOR EACH ROW
    EXECUTE FUNCTION set_updated_at();
```

นี่คือข้อดีของการแยก trigger function ออกจากตารางใดตารางหนึ่งโดยเฉพาะ — เขียนครั้งเดียว ใช้ได้ทุกที่

---

## Step 476: ตัวอย่างจริง — AFTER INSERT/UPDATE/DELETE Trigger สำหรับบันทึก audit_log อัตโนมัติ

ต่อไปนี้คือการสร้างระบบ Audit Trail แบบครบวงจร: ทุกครั้งที่มีการ `INSERT`, `UPDATE`, หรือ `DELETE` บนตาราง `products` จะมีการบันทึกลง `audit_log` โดยอัตโนมัติ เก็บทั้งค่าก่อนและหลังการเปลี่ยนแปลงในรูปแบบ `JSONB`

### สร้าง Generic Audit Trigger Function

```sql
CREATE OR REPLACE FUNCTION trg_audit_row_changes()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
DECLARE
    v_row_id INTEGER;
BEGIN
    -- ดึง primary key column แรกของแถวมาเก็บเป็น row_id
    -- (สมมติว่า primary key column ชื่อ <table>_id ตามธรรมเนียมของฐานข้อมูลนี้)
    IF TG_OP = 'DELETE' THEN
        v_row_id := (to_jsonb(OLD) ->> (TG_TABLE_NAME || '_id'))::INTEGER;
    ELSE
        v_row_id := (to_jsonb(NEW) ->> (TG_TABLE_NAME || '_id'))::INTEGER;
    END IF;

    INSERT INTO audit_log (table_name, operation, row_id, old_data, new_data)
    VALUES (
        TG_TABLE_NAME,
        TG_OP,
        v_row_id,
        CASE WHEN TG_OP IN ('UPDATE', 'DELETE') THEN to_jsonb(OLD) ELSE NULL END,
        CASE WHEN TG_OP IN ('INSERT', 'UPDATE') THEN to_jsonb(NEW) ELSE NULL END
    );

    -- AFTER trigger: ค่า return ไม่มีผลต่อข้อมูล แต่ต้อง return ให้ครบ
    IF TG_OP = 'DELETE' THEN
        RETURN OLD;
    ELSE
        RETURN NEW;
    END IF;
END;
$$;

COMMENT ON FUNCTION trg_audit_row_changes() IS
    'Generic AFTER trigger function: บันทึกการเปลี่ยนแปลงทุกแถวลง audit_log โดยอัตโนมัติ';
```

### ผูก Trigger เข้ากับตาราง products

```sql
CREATE TRIGGER trg_products_audit
    AFTER INSERT OR UPDATE OR DELETE ON products
    FOR EACH ROW
    EXECUTE FUNCTION trg_audit_row_changes();
```

### สาธิต: INSERT

```sql
INSERT INTO products (product_name, category_id, supplier_id, unit_price, stock_quantity)
VALUES ('แท่นชาร์จไร้สาย MagCharge', 3, 2, 890.00, 150);

SELECT audit_id, table_name, operation, row_id, new_data
FROM audit_log
ORDER BY audit_id DESC
LIMIT 1;
/*
 audit_id | table_name | operation | row_id |                         new_data
----------+------------+-----------+--------+-----------------------------------------------------------
        1 | products   | INSERT    |     21 | {"product_id": 21, "product_name": "แท่นชาร์จไร้สาย MagCharge", ...}
*/
```

### สาธิต: UPDATE (เห็นทั้งค่าเก่าและใหม่)

```sql
UPDATE products
SET stock_quantity = 140, unit_price = 850.00
WHERE product_id = 21;

SELECT audit_id, operation, row_id,
       old_data ->> 'unit_price'      AS old_price,
       new_data ->> 'unit_price'      AS new_price,
       old_data ->> 'stock_quantity'  AS old_stock,
       new_data ->> 'stock_quantity'  AS new_stock
FROM audit_log
WHERE table_name = 'products' AND row_id = 21
ORDER BY audit_id;
/*
 audit_id | operation | row_id | old_price | new_price | old_stock | new_stock
----------+-----------+--------+-----------+-----------+-----------+-----------
        1 | INSERT    |     21 |           |    890.00 |           |       150
        2 | UPDATE    |     21 |    890.00 |    850.00 |       150 |       140
*/
```

### สาธิต: DELETE

```sql
DELETE FROM products WHERE product_id = 21;

SELECT audit_id, operation, row_id, old_data ->> 'product_name' AS deleted_product
FROM audit_log
WHERE table_name = 'products' AND row_id = 21
ORDER BY audit_id;
/*
 audit_id | operation | row_id |     deleted_product
----------+-----------+--------+--------------------------
        1 | INSERT    |     21 |
        2 | UPDATE    |     21 |
        3 | DELETE    |     21 | แท่นชาร์จไร้สาย MagCharge
*/
```

ระบบ audit trail นี้ทำงานโดยอิสระจากแอปพลิเคชันโดยสิ้นเชิง ไม่ว่าจะแก้ไขข้อมูลผ่านช่องทางใด (`psql`, ORM, สคริปต์ batch) ก็จะถูกบันทึกลง `audit_log` เสมอ — นี่คือข้อดีสำคัญของการทำ audit ที่ระดับฐานข้อมูลแทนที่จะทำที่ระดับแอปพลิเคชัน

### ทำไมใช้ `AFTER` ไม่ใช่ `BEFORE` สำหรับ Audit Log

เพราะเราต้องการบันทึกว่า "การเปลี่ยนแปลงเกิดขึ้นสำเร็จแล้ว" ถ้าใช้ `BEFORE` แล้วมี trigger หรือ constraint อื่นทำให้ operation ล้มเหลวภายหลัง (เช่น constraint violation) เราจะได้ audit log ของเหตุการณ์ที่ไม่เคยเกิดขึ้นจริง — ซึ่งผิดหลักการของ audit trail ที่ควรสะท้อนความเป็นจริงเท่านั้น (และในทางเทคนิค ถ้า transaction ถูก rollback ทั้ง audit_log entry และการเปลี่ยนแปลงจริงก็จะถูก rollback ไปด้วยกันอยู่แล้ว เพราะอยู่ใน transaction เดียวกัน)

---

## Step 477: ตัวอย่างจริง — BEFORE INSERT Trigger สำหรับ Cross-table Validation

`CHECK` constraint มองเห็นแค่คอลัมน์ในแถวตัวเอง ไม่สามารถ query ตารางอื่นได้ ดังนั้นถ้าต้องการ validate เงื่อนไขที่ต้องอ้างอิงข้อมูลจากตารางอื่น (cross-table validation) ต้องใช้ Trigger

### โจทย์: ก่อนเพิ่มรายการสั่งซื้อ (order_items) ต้องตรวจสอบว่า

1. สินค้านั้นต้อง `is_active = true` (ห้ามสั่งซื้อสินค้าที่ปิดการขายแล้ว)
2. สต๊อกสินค้าต้องเพียงพอ (`stock_quantity >= quantity` ที่จะสั่ง)

```sql
CREATE OR REPLACE FUNCTION trg_validate_order_item()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
DECLARE
    v_is_active      BOOLEAN;
    v_stock_quantity INTEGER;
    v_product_name   VARCHAR(150);
BEGIN
    SELECT is_active, stock_quantity, product_name
    INTO v_is_active, v_stock_quantity, v_product_name
    FROM products
    WHERE product_id = NEW.product_id
    FOR UPDATE;  -- ล็อคแถวสินค้าไว้ ป้องกัน race condition ระหว่างตรวจสต๊อก

    IF NOT FOUND THEN
        RAISE EXCEPTION 'ไม่พบสินค้า product_id = %', NEW.product_id
            USING ERRCODE = 'foreign_key_violation';
    END IF;

    IF NOT v_is_active THEN
        RAISE EXCEPTION 'ไม่สามารถสั่งซื้อสินค้า "%" ได้ เนื่องจากสินค้านี้ปิดการขายแล้ว (is_active = false)',
            v_product_name
            USING ERRCODE = 'check_violation',
                  HINT = 'ตรวจสอบสถานะสินค้าในตาราง products ก่อนสั่งซื้อ';
    END IF;

    IF v_stock_quantity < NEW.quantity THEN
        RAISE EXCEPTION 'สต๊อกสินค้า "%" ไม่เพียงพอ: มีอยู่ % ชิ้น แต่พยายามสั่งซื้อ % ชิ้น',
            v_product_name, v_stock_quantity, NEW.quantity
            USING ERRCODE = 'check_violation',
                  HINT = 'ลดจำนวนที่สั่งซื้อ หรือรอเติมสต๊อก';
    END IF;

    RETURN NEW;
END;
$$;

CREATE TRIGGER trg_order_items_validate
    BEFORE INSERT ON order_items
    FOR EACH ROW
    EXECUTE FUNCTION trg_validate_order_item();
```

### สาธิต: กรณีล้มเหลว — สินค้าปิดการขาย

```sql
-- product_id = 20 คือ "โดรนบังคับ SkyMini" ซึ่ง is_active = false
INSERT INTO order_items (order_id, product_id, quantity, unit_price)
VALUES (15, 20, 1, 3290.00);

/*
ERROR:  ไม่สามารถสั่งซื้อสินค้า "โดรนบังคับ SkyMini" ได้ เนื่องจากสินค้านี้ปิดการขายแล้ว (is_active = false)
HINT:  ตรวจสอบสถานะสินค้าในตาราง products ก่อนสั่งซื้อ
*/
```

### สาธิต: กรณีล้มเหลว — สต๊อกไม่พอ

```sql
-- product_id = 20 มี stock_quantity = 12 เท่านั้น
UPDATE products SET is_active = true WHERE product_id = 20; -- เปิดขายชั่วคราวเพื่อทดสอบ

INSERT INTO order_items (order_id, product_id, quantity, unit_price)
VALUES (15, 20, 999, 3290.00);

/*
ERROR:  สต๊อกสินค้า "โดรนบังคับ SkyMini" ไม่เพียงพอ: มีอยู่ 12 ชิ้น แต่พยายามสั่งซื้อ 999 ชิ้น
HINT:  ลดจำนวนที่สั่งซื้อ หรือรอเติมสต๊อก
*/
```

### สาธิต: กรณีสำเร็จ

```sql
INSERT INTO order_items (order_id, product_id, quantity, unit_price)
VALUES (15, 20, 2, 3290.00);
-- INSERT 0 1  (สำเร็จ เพราะ 2 <= 12 และสินค้า active)
```

### ทำไมต้อง `FOR UPDATE` ในคำสั่ง `SELECT`

การเพิ่ม `FOR UPDATE` ทำให้แถวสินค้านั้นถูกล็อคระหว่าง transaction ป้องกันปัญหา **race condition** ที่สอง transaction พร้อมกันอาจอ่านค่า `stock_quantity` เดียวกันก่อนที่ฝั่งใดฝั่งหนึ่งจะเขียนค่าลดสต๊อกจริง (คล้ายกับปัญหา lost update) การ lock แถวไว้ก่อนจะทำให้ transaction ที่สองต้องรอ transaction แรก commit หรือ rollback ก่อน จึงจะอ่านค่าที่เป็นปัจจุบันจริง ๆ

### ข้อสังเกตสำคัญ: RAISE EXCEPTION จะ Rollback ทั้ง Statement

เมื่อ trigger function เรียก `RAISE EXCEPTION` ทั้งคำสั่ง `INSERT`/`UPDATE`/`DELETE` ที่กำลังดำเนินการจะถูกยกเลิกทันที (ถ้าอยู่ใน explicit transaction ที่มีคำสั่งอื่นมาก่อนหน้า คำสั่งอื่นเหล่านั้นจะไม่ได้รับผลกระทบ เพราะ PostgreSQL จะ rollback แค่ถึงจุด savepoint ล่าสุดถ้าไคลเอนต์จัดการ error เอง แต่ถ้าไม่มีการจัดการ error ทั้ง transaction จะถูก abort)

---

## Step 478: Trigger หลายตัวบนตารางเดียวกัน — ลำดับการทำงาน

ตารางหนึ่งสามารถมี Trigger ได้หลายตัว แม้กระทั่งหลายตัวที่ event/timing เดียวกัน คำถามคือ **ลำดับการทำงาน** เป็นอย่างไร

**กฎของ PostgreSQL**: เมื่อมี Trigger หลายตัวที่ timing และ level เดียวกัน (เช่น `BEFORE INSERT FOR EACH ROW` ทั้งคู่) จะถูกเรียงลำดับการทำงานตาม **ชื่อ trigger เรียงตามตัวอักษร (alphabetical order)** ไม่ใช่ตามลำดับที่สร้าง

### ตัวอย่างสาธิต

```sql
CREATE OR REPLACE FUNCTION trg_announce(msg TEXT)
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    RAISE NOTICE 'Trigger "%" (TG_ARGV[0]=%) กำลังทำงาน', TG_NAME, TG_ARGV[0];
    RETURN NEW;
END;
$$;

-- สร้าง 3 trigger บน BEFORE INSERT ตัวเดียวกัน แต่ตั้งชื่อไม่เรียงตามลำดับที่สร้าง
CREATE TRIGGER trg_c_third
    BEFORE INSERT ON order_items
    FOR EACH ROW
    EXECUTE FUNCTION trg_announce('C');

CREATE TRIGGER trg_a_first
    BEFORE INSERT ON order_items
    FOR EACH ROW
    EXECUTE FUNCTION trg_announce('A');

CREATE TRIGGER trg_b_second
    BEFORE INSERT ON order_items
    FOR EACH ROW
    EXECUTE FUNCTION trg_announce('B');

-- ทดสอบ
INSERT INTO order_items (order_id, product_id, quantity, unit_price)
VALUES (15, 3, 1, 590.00);

/*
NOTICE:  Trigger "trg_a_first" (TG_ARGV[0]=A) กำลังทำงาน
NOTICE:  Trigger "trg_b_second" (TG_ARGV[0]=B) กำลังทำงาน
NOTICE:  Trigger "trg_c_third" (TG_ARGV[0]=C) กำลังทำงาน
*/
```

แม้เราจะสร้าง `trg_c_third` ก่อนตัวอื่น แต่มันทำงาน **ลำดับสุดท้าย** เพราะ PostgreSQL เรียงตามชื่อ trigger (`trg_a_first` < `trg_b_second` < `trg_c_third` ตามลำดับตัวอักษร)

```sql
-- ล้าง trigger ทดลอง
DROP TRIGGER trg_c_third ON order_items;
DROP TRIGGER trg_a_first ON order_items;
DROP TRIGGER trg_b_second ON order_items;
DROP FUNCTION trg_announce(TEXT);
```

### เทคนิคการควบคุมลำดับด้วยชื่อ

เนื่องจากลำดับขึ้นกับชื่อ trigger ล้วน ๆ นักพัฒนามักใช้ **prefix ตัวเลข** ในชื่อ trigger เพื่อควบคุมลำดับให้ชัดเจนและคาดเดาได้ เช่น:

```sql
CREATE TRIGGER trg_10_validate_stock
    BEFORE INSERT ON order_items
    FOR EACH ROW
    EXECUTE FUNCTION trg_validate_order_item();

CREATE TRIGGER trg_20_normalize_data
    BEFORE INSERT ON order_items
    FOR EACH ROW
    EXECUTE FUNCTION trg_normalize_order_item();

CREATE TRIGGER trg_30_log_creation
    BEFORE INSERT ON order_items
    FOR EACH ROW
    EXECUTE FUNCTION trg_log_order_item_creation();
```

การตั้งชื่อแบบนี้ทำให้อ่านโค้ดแล้วรู้ทันทีว่าอะไรทำงานก่อนหลัง โดยไม่ต้องไปเปิด `pg_trigger` มาไล่ดู เป็น pattern ที่แนะนำอย่างยิ่งเมื่อระบบมี trigger หลายตัวซ้อนกันบนตารางเดียว

### ลำดับข้าม timing ที่ต่างกัน

ถ้ามี timing ต่างกัน ลำดับใหญ่จะเป็นดังนี้เสมอ ไม่ว่าจะตั้งชื่ออย่างไร:

1. `BEFORE` trigger ระดับ ROW (เรียงตามชื่อ ถ้ามีหลายตัว)
2. Constraint checks (เช่น `CHECK`, `FOREIGN KEY`) และการเขียนแถวจริงลง table/index
3. `AFTER` trigger ระดับ ROW (เรียงตามชื่อ)
4. `AFTER` trigger ระดับ STATEMENT (เรียงตามชื่อ) — ทำงานหลังสุด เมื่อทุกแถวใน statement เสร็จหมดแล้ว

และ `BEFORE STATEMENT` trigger จะทำงานก่อน `BEFORE ROW` เสมอ เพราะเป็นจุดเริ่มต้นของ statement ทั้งหมด

---

## Step 479: การปิด/เปิด Trigger ชั่วคราว และข้อควรระวังเรื่อง Performance

### ปิด/เปิด Trigger ด้วย ALTER TABLE

บางสถานการณ์ (เช่น การ import ข้อมูลจำนวนมาก, การ migrate ข้อมูลเก่า) เราต้องการปิด trigger ชั่วคราวเพื่อความเร็ว หรือเพื่อไม่ให้ audit log บวมด้วยข้อมูลที่ไม่จำเป็น

```sql
-- ปิด trigger ตัวเดียว
ALTER TABLE products DISABLE TRIGGER trg_products_audit;

-- ปิด trigger ทั้งหมดของตาราง (รวมถึงที่มาจาก FOREIGN KEY ด้วย! ต้องระวัง)
ALTER TABLE products DISABLE TRIGGER ALL;

-- เปิดกลับ
ALTER TABLE products ENABLE TRIGGER trg_products_audit;
ALTER TABLE products ENABLE TRIGGER ALL;

-- เปิดเฉพาะ "USER" trigger (ไม่แตะ trigger ระบบที่มาจาก FK constraint)
ALTER TABLE products ENABLE TRIGGER USER;
```

> **คำเตือนสำคัญ**: `DISABLE TRIGGER ALL` จะปิด trigger ที่ระบบสร้างขึ้นเองสำหรับ `FOREIGN KEY` constraint ด้วย! ถ้าปิดแล้ว insert/update/delete ข้อมูลที่ละเมิด referential integrity โดยไม่รู้ตัว ข้อมูลจะเสียหายแบบเงียบ ๆ ควรใช้ `ALTER TABLE ... DISABLE TRIGGER USER` แทนถ้าต้องการปิดเฉพาะ trigger ที่ผู้ใช้สร้างเอง หรือระบุชื่อ trigger เจาะจงแทนการใช้ `ALL`

### ตัวอย่างการใช้งานจริง: bulk import โดยปิด audit trigger ชั่วคราว

```sql
BEGIN;

ALTER TABLE products DISABLE TRIGGER trg_products_audit;

-- import ข้อมูลจำนวนมากโดยไม่บันทึก audit log รายแถว (เพราะรู้อยู่แล้วว่าเป็นการ import ครั้งแรก)
COPY products (product_name, category_id, supplier_id, unit_price, stock_quantity)
FROM '/data/legacy_products.csv' WITH (FORMAT csv, HEADER true);

ALTER TABLE products ENABLE TRIGGER trg_products_audit;

-- บันทึก audit entry เดียวสรุปว่ามีการ import เกิดขึ้น (แทนการ log ทีละแถว)
INSERT INTO audit_log (table_name, operation, row_id, new_data)
VALUES ('products', 'BULK_IMPORT', NULL,
        jsonb_build_object('note', 'legacy_products.csv imported', 'imported_at', now()));

COMMIT;
```

### วิธีอื่นในการข้าม Trigger: session_replication_role

อีกวิธีหนึ่ง (มักใช้ในเครื่องมือ replication/migration เช่น `pg_dump`/`pg_restore`) คือการตั้งค่า session variable `session_replication_role = replica` ซึ่งจะทำให้ trigger ที่มี attribute เป็น `ORIGIN` (ค่า default) ไม่ทำงาน แต่ trigger ที่ตั้งเป็น `ALWAYS` จะยังทำงานอยู่:

```sql
SET session_replication_role = replica;
-- ...operations ที่ต้องการข้าม trigger ปกติ...
SET session_replication_role = origin;  -- กลับสู่ปกติ
```

วิธีนี้ต้องมีสิทธิ์ superuser (หรือ role ที่มีสิทธิ์เพียงพอ) และมักสงวนไว้สำหรับเครื่องมือระดับระบบ ไม่แนะนำให้ใช้ในโค้ดแอปพลิเคชันทั่วไป

### ข้อควรระวังเรื่อง Performance ของ Trigger

1. **Trigger ระดับ ROW เพิ่ม overhead ต่อแถว** — ถ้าตารางมี write throughput สูงมาก (เช่น หลักพัน/วินาที) การมี `BEFORE`/`AFTER` trigger ที่ซับซ้อนจะทำให้ throughput ลดลงอย่างมีนัยสำคัญ ควรทำ benchmark ก่อนนำ trigger ที่ซับซ้อนไปใช้กับตารางที่มี traffic สูง

2. **หลีกเลี่ยง query ที่หนักภายใน trigger function** — โดยเฉพาะ query ที่ไม่มี index รองรับ หรือ query ที่ scan ตารางใหญ่ เพราะจะทำงานซ้ำทุกครั้งที่ trigger fire (ต่อแถว ถ้าเป็น ROW-level)

3. **ระวัง Trigger Cascade / Recursive Trigger** — ถ้า trigger บนตาราง A ไป `UPDATE` ตาราง B ที่มี trigger ไป `UPDATE` กลับมาที่ตาราง A อีกที อาจเกิด infinite loop หรือ deadlock ได้ ควรใช้ `pg_trigger_depth()` เพื่อตรวจสอบและป้องกัน recursion ที่ไม่ตั้งใจ:

```sql
CREATE OR REPLACE FUNCTION trg_safe_example()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    IF pg_trigger_depth() > 1 THEN
        RETURN NEW;  -- กำลังถูกเรียกซ้อนจาก trigger อื่น ข้าม logic เพื่อป้องกัน infinite loop
    END IF;
    -- ... logic จริง ...
    RETURN NEW;
END;
$$;
```

4. **ใช้ `WHEN` clause เพื่อกรองตั้งแต่ต้น** — แทนที่จะเช็คเงื่อนไขข้างในฟังก์ชันแล้ว `RETURN NEW;` เฉย ๆ ถ้าไม่เข้าเงื่อนไข ให้ใช้ `WHEN` clause ใน `CREATE TRIGGER` แทน เพราะ PostgreSQL จะข้ามการเรียกฟังก์ชันไปเลยถ้าเงื่อนไขไม่ผ่าน (เร็วกว่าเพราะไม่ต้องเข้า/ออกฟังก์ชัน):

```sql
-- แทนที่จะเช็คเงื่อนไขในฟังก์ชัน ให้กรองด้วย WHEN
CREATE TRIGGER trg_products_price_change_only
    AFTER UPDATE ON products
    FOR EACH ROW
    WHEN (OLD.unit_price IS DISTINCT FROM NEW.unit_price)  -- fire เฉพาะเมื่อราคาเปลี่ยนจริง
    EXECUTE FUNCTION trg_audit_row_changes();
```

5. **พิจารณาใช้ STATEMENT-level trigger กับ transition table แทน ROW-level เมื่อทำได้** — สำหรับงานสรุปผล (aggregation) การทำงานครั้งเดียวต่อ statement (โดยใช้ transition table วนลูปข้อมูลใน SQL ปกติ) มักเร็วกว่าการ fire ฟังก์ชัน PL/pgSQL ทีละแถวมาก โดยเฉพาะเมื่อ statement นั้นกระทบหลายพันแถว

6. **Index ตาราง audit_log ให้เหมาะสม** — เมื่อ audit_log โตขึ้นเรื่อย ๆ ควรมี index บน `(table_name, row_id)` และ `changed_at` เพื่อให้ query ย้อนหลังเร็ว และพิจารณาทำ partitioning ตาม `changed_at` (range partitioning รายเดือน/รายปี) เมื่อข้อมูลมีปริมาณมากในระยะยาว

```sql
CREATE INDEX idx_audit_log_table_row ON audit_log (table_name, row_id);
CREATE INDEX idx_audit_log_changed_at ON audit_log (changed_at);
```

7. **วัดผลกระทบจริงด้วย `EXPLAIN ANALYZE`** — คำสั่ง `EXPLAIN ANALYZE` จะแสดงเวลาที่ trigger ใช้ไปด้วย (ในบรรทัด `Trigger <name>: time=... calls=...`) ทำให้วัด overhead ของแต่ละ trigger ได้อย่างเป็นรูปธรรม:

```sql
EXPLAIN ANALYZE
UPDATE products SET stock_quantity = stock_quantity - 1 WHERE product_id = 1;
/*
...
Trigger trg_products_set_updated_at: time=0.045 calls=1
Trigger trg_products_audit: time=0.128 calls=1
Execution Time: 0.312 ms
*/
```

---

## Step 480: แบบฝึกหัดรวม — ระบบ Audit Trail แบบเต็มรูปแบบ + Trigger รักษาความถูกต้องของสต๊อกสินค้า

มาประกอบทุกความรู้ในบทนี้เข้าด้วยกัน สร้างระบบที่สมบูรณ์ 2 ส่วน:

**ส่วนที่ 1**: ระบบ Audit Trail ครอบคลุมตารางหลัก (`products`, `customers`, `orders`) ด้วยฟังก์ชันเดียว
**ส่วนที่ 2**: Trigger ที่ทำให้ `products.stock_quantity` ถูกต้องเสมอโดยอัตโนมัติ ไม่ว่าจะมีการสั่งซื้อ (ลดสต๊อก) แก้ไขจำนวนสั่งซื้อ (ปรับส่วนต่าง) หรือยกเลิกรายการ (คืนสต๊อก)

### ส่วนที่ 1: ขยาย Audit Trail ไปยังตารางอื่น

เราใช้ `trg_audit_row_changes()` ที่สร้างไว้แล้วใน Step 476 ผูกเพิ่มกับ `customers` และ `orders`:

```sql
CREATE TRIGGER trg_customers_audit
    AFTER INSERT OR UPDATE OR DELETE ON customers
    FOR EACH ROW
    EXECUTE FUNCTION trg_audit_row_changes();

CREATE TRIGGER trg_orders_audit
    AFTER INSERT OR UPDATE OR DELETE ON orders
    FOR EACH ROW
    EXECUTE FUNCTION trg_audit_row_changes();
```

ทดสอบ:

```sql
UPDATE customers SET country = 'Thailand' WHERE customer_id = 7;  -- John Smith ย้ายมาไทย
UPDATE orders SET status = 'delivered' WHERE order_id = 15;

SELECT table_name, operation, row_id, changed_at
FROM audit_log
WHERE table_name IN ('customers', 'orders')
ORDER BY audit_id DESC
LIMIT 5;
```

### ส่วนที่ 2: Trigger รักษาความถูกต้องของสต๊อกสินค้า

โจทย์: เมื่อมีการเปลี่ยนแปลง `order_items` ต้องการให้ `products.stock_quantity` ถูกปรับให้สอดคล้องกันเสมอ โดยไม่ต้องพึ่งพา application code:

- `INSERT` order_item ใหม่ → ลดสต๊อกสินค้านั้นตาม `quantity`
- `UPDATE` order_item ที่เปลี่ยน `quantity` หรือ `product_id` → ปรับส่วนต่างให้ถูกต้อง (คืนสต๊อกเดิม แล้วหักสต๊อกใหม่)
- `DELETE` order_item → คืนสต๊อกกลับ

```sql
CREATE OR REPLACE FUNCTION trg_maintain_stock_quantity()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        UPDATE products
        SET stock_quantity = stock_quantity - NEW.quantity
        WHERE product_id = NEW.product_id;

    ELSIF TG_OP = 'DELETE' THEN
        UPDATE products
        SET stock_quantity = stock_quantity + OLD.quantity
        WHERE product_id = OLD.product_id;

    ELSIF TG_OP = 'UPDATE' THEN
        IF NEW.product_id = OLD.product_id THEN
            -- สินค้าเดิม แต่จำนวนเปลี่ยน: ปรับเฉพาะส่วนต่าง
            UPDATE products
            SET stock_quantity = stock_quantity - (NEW.quantity - OLD.quantity)
            WHERE product_id = NEW.product_id;
        ELSE
            -- เปลี่ยนสินค้าไปคนละตัว: คืนสต๊อกสินค้าเดิมทั้งหมด แล้วหักสต๊อกสินค้าใหม่ทั้งหมด
            UPDATE products
            SET stock_quantity = stock_quantity + OLD.quantity
            WHERE product_id = OLD.product_id;

            UPDATE products
            SET stock_quantity = stock_quantity - NEW.quantity
            WHERE product_id = NEW.product_id;
        END IF;
    END IF;

    IF TG_OP = 'DELETE' THEN
        RETURN OLD;
    ELSE
        RETURN NEW;
    END IF;
END;
$$;

CREATE TRIGGER trg_20_order_items_maintain_stock
    AFTER INSERT OR UPDATE OR DELETE ON order_items
    FOR EACH ROW
    EXECUTE FUNCTION trg_maintain_stock_quantity();
```

> สังเกตว่าเราตั้งชื่อ `trg_20_...` เพื่อให้แน่ใจว่า trigger ตรวจสอบความถูกต้อง (`trg_10_validate_stock` จาก Step 477 ถ้าเปลี่ยนชื่อให้เข้าชุดนี้) ทำงาน**ก่อน** trigger ที่ปรับสต๊อกจริง ตามหลักการจัดลำดับใน Step 478 — ในทางปฏิบัติควรรวม validate + maintain ไว้ในฟังก์ชันเดียวหรือคุมลำดับด้วย prefix ตัวเลขเสมอ เพื่อไม่ให้เกิดกรณีที่สต๊อกถูกหักไปแล้วแต่ validation ล้มเหลวทีหลัง (ในที่นี้ validation เป็น `BEFORE` และ maintain เป็น `AFTER` จึงมั่นใจได้ว่า validate ทำงานก่อนเสมอไม่ว่าจะตั้งชื่ออย่างไร เพราะคนละ timing กัน)

### สาธิตการทำงานครบวงจร

```sql
-- ดูสต๊อกก่อนเริ่ม (product_id = 3: เมาส์ไร้สาย ErgoClick)
SELECT product_id, product_name, stock_quantity FROM products WHERE product_id = 3;
-- stock_quantity = 120 (ลบไปแล้ว 3 จาก Step 478 ตัวอย่าง insert 1 ชิ้น -> เหลือ 119 ถ้ารันตามลำดับบทนี้)

-- 1) สั่งซื้อเพิ่ม 5 ชิ้น
INSERT INTO order_items (order_id, product_id, quantity, unit_price)
VALUES (15, 3, 5, 590.00);

SELECT stock_quantity FROM products WHERE product_id = 3;  -- ลดลง 5

-- 2) แก้ไขจำนวนจาก 5 เป็น 8 (เพิ่มขึ้น 3)
UPDATE order_items
SET quantity = 8
WHERE order_id = 15 AND product_id = 3;

SELECT stock_quantity FROM products WHERE product_id = 3;  -- ลดลงอีก 3 (รวมลดลง 8 จากค่าเริ่มต้น)

-- 3) ยกเลิกรายการนี้ (ลบทิ้ง)
DELETE FROM order_items
WHERE order_id = 15 AND product_id = 3;

SELECT stock_quantity FROM products WHERE product_id = 3;  -- คืนกลับสู่ค่าเดิมครบ 8
```

ตรวจสอบผ่าน audit_log ว่าทุกการเปลี่ยนแปลงของ `products.stock_quantity` (ที่เกิดจาก trigger เอง ไม่ใช่จากแอปพลิเคชันโดยตรง) ก็ถูกบันทึกไว้ด้วย เพราะ `trg_products_audit` (จาก Step 476) ยัง `AFTER UPDATE` อยู่บน `products` เสมอ ไม่ว่าการ `UPDATE` นั้นจะมาจากผู้ใช้โดยตรงหรือมาจาก trigger อีกตัวหนึ่งก็ตาม:

```sql
SELECT audit_id, table_name, operation, row_id,
       old_data ->> 'stock_quantity' AS old_stock,
       new_data ->> 'stock_quantity' AS new_stock
FROM audit_log
WHERE table_name = 'products' AND row_id = 3
ORDER BY audit_id DESC
LIMIT 5;
```

นี่คือพลังของ Trigger: เราไม่ต้องเขียน logic "ลดสต๊อก" หรือ "บันทึก audit" ซ้ำในทุกจุดของแอปพลิเคชันที่แตะต้องข้อมูล — ฐานข้อมูลรับประกันความถูกต้องของข้อมูลให้เองในทุกช่องทางที่เข้าถึง

### ข้อควรระวังของ Pattern นี้ในโลกจริง

- ถ้าระบบมี concurrent transaction จำนวนมากพร้อมกัน การ `UPDATE products SET stock_quantity = stock_quantity - x` ใน trigger จะสร้าง lock contention บนแถวสินค้ายอดนิยม (hot row) ซึ่งเป็นข้อจำกัดโดยธรรมชาติของการรักษา counter ที่แม่นยำแบบ real-time — ทางแก้ในระบบที่มี throughput สูงมากคือใช้ตาราง `stock_movements` (append-only, ไม่มี lock contention) แล้วคำนวณสต๊อกคงเหลือแบบ derived/materialized แทนการ update counter ตรง ๆ
- ควรพิจารณาเพิ่ม `CHECK (stock_quantity >= 0)` เป็น constraint คู่กับ trigger นี้ เพื่อป้องกันสต๊อกติดลบจาก bug หรือ race condition ที่หลุดรอด

```sql
ALTER TABLE products ADD CONSTRAINT chk_products_stock_non_negative
    CHECK (stock_quantity >= 0);
```

---

## สรุปท้ายบท

- **Trigger** คือกลไกที่ทำให้โค้ด (Trigger Function) ทำงานอัตโนมัติเมื่อมี `INSERT`/`UPDATE`/`DELETE`/`TRUNCATE` เกิดขึ้นกับตารางหรือ view โดยไม่ต้องพึ่งพาแอปพลิเคชันเรียกเอง
- **Trigger Function** ต้องคืนค่าชนิด `TRIGGER` และเข้าถึงตัวแปรพิเศษอย่าง `NEW`, `OLD`, `TG_OP`, `TG_TABLE_NAME`, `TG_WHEN`, `TG_LEVEL` ได้
- **`BEFORE`** ใช้แก้ไขข้อมูลก่อนบันทึกหรือยกเลิก operation, **`AFTER`** ใช้ทำ side effect หลังบันทึกสำเร็จแล้ว (เช่น audit log), **`INSTEAD OF`** ใช้กับ view เพื่อทำให้แก้ไขได้
- **`FOR EACH ROW`** ทำงานต่อแถว ส่วน **`FOR EACH STATEMENT`** ทำงานครั้งเดียวต่อคำสั่ง ใช้ transition table (`REFERENCING ... TABLE`) เมื่อต้องการเข้าถึงชุดแถวทั้งหมดในระดับ STATEMENT
- Pattern ที่ใช้บ่อยที่สุด: **`updated_at` auto-update** (BEFORE UPDATE) และ **audit trail** (AFTER INSERT/UPDATE/DELETE) — ทั้งสองแบบควรเขียนเป็นฟังก์ชัน generic ที่ใช้ซ้ำได้กับหลายตาราง
- Trigger เหมาะกับ **cross-table validation** ที่ `CHECK` constraint ทำไม่ได้ แต่ควรใช้ `CHECK`/`FOREIGN KEY`/`UNIQUE` ก่อนเสมอถ้าเพียงพอ เพราะเร็วกว่าและชัดเจนกว่า
- Trigger หลายตัวบน timing/level เดียวกันจะทำงานตาม **ลำดับตัวอักษรของชื่อ trigger** — ใช้ prefix ตัวเลข (`trg_10_...`, `trg_20_...`) เพื่อควบคุมลำดับให้ชัดเจน
- ปิด/เปิด Trigger ด้วย `ALTER TABLE ... DISABLE/ENABLE TRIGGER` — ระวังการใช้ `ALL` เพราะจะปิด trigger ของ Foreign Key ด้วย ใช้ `USER` แทนถ้าต้องการปิดเฉพาะ trigger ที่ผู้ใช้สร้าง
- Trigger มี performance overhead ต่อแถวเสมอ ควรใช้ `WHEN` clause กรองแต่เนิ่น ๆ ระวัง recursive trigger ด้วย `pg_trigger_depth()` และวัดผลกระทบจริงด้วย `EXPLAIN ANALYZE`
- ระบบ audit trail และ stock-integrity ที่สร้างในบทนี้แสดงให้เห็นว่า Trigger ทำให้ฐานข้อมูล**รับประกันความถูกต้องของข้อมูลได้เองโดยไม่ต้องพึ่งพา application code** ไม่ว่าจะเข้าถึงข้อมูลผ่านช่องทางใดก็ตาม

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1
เพิ่มคอลัมน์ `updated_at TIMESTAMPTZ NOT NULL DEFAULT now()` ให้ตาราง `customers` แล้วสร้าง Trigger ให้อัปเดตค่านี้อัตโนมัติทุกครั้งที่มีการแก้ไขข้อมูลลูกค้า โดยใช้ฟังก์ชัน `set_updated_at()` ที่มีอยู่แล้ว

<details>
<summary>เฉลย</summary>

```sql
ALTER TABLE customers ADD COLUMN updated_at TIMESTAMPTZ NOT NULL DEFAULT now();

CREATE TRIGGER trg_customers_set_updated_at
    BEFORE UPDATE ON customers
    FOR EACH ROW
    EXECUTE FUNCTION set_updated_at();

-- ทดสอบ
UPDATE customers SET country = 'Thailand' WHERE customer_id = 8;
SELECT customer_id, country, updated_at FROM customers WHERE customer_id = 8;
```

เนื่องจากฟังก์ชัน `set_updated_at()` เป็น generic function (ไม่อ้างอิงชื่อตารางเฉพาะเจาะจง) จึงนำมาใช้ซ้ำกับตารางใดก็ได้ที่มีคอลัมน์ชื่อ `updated_at`
</details>

---

### แบบฝึกหัดที่ 2
สร้าง Trigger แบบ `AFTER DELETE` บนตาราง `customers` ที่บันทึกลง `audit_log` เมื่อมีการลบลูกค้า (ใช้ฟังก์ชัน `trg_audit_row_changes()` ที่มีอยู่แล้วได้เลย หรือเขียนฟังก์ชันใหม่เฉพาะ DELETE ก็ได้)

<details>
<summary>เฉลย</summary>

วิธีที่ง่ายที่สุดคือผูก `trg_audit_row_changes()` ที่มีอยู่แล้วเข้ากับ event `DELETE` (ถ้ายังไม่ได้ผูกครบทุก event จาก Step 480):

```sql
-- ถ้ายังไม่มี trigger ครอบคลุม DELETE บน customers ให้สร้างเพิ่ม
CREATE TRIGGER trg_customers_audit_delete
    AFTER DELETE ON customers
    FOR EACH ROW
    EXECUTE FUNCTION trg_audit_row_changes();

-- ทดสอบ (ใช้ transaction แล้ว rollback เพื่อไม่ลบข้อมูลจริง)
BEGIN;
DELETE FROM customers WHERE customer_id = 14;

SELECT table_name, operation, row_id, old_data ->> 'email' AS deleted_email
FROM audit_log
WHERE table_name = 'customers' AND operation = 'DELETE'
ORDER BY audit_id DESC LIMIT 1;

ROLLBACK;  -- ยกเลิกการลบ เพื่อรักษาข้อมูลตัวอย่างไว้
```

หมายเหตุ: ถ้า Step 480 สร้าง `trg_customers_audit` ที่ครอบคลุม `INSERT OR UPDATE OR DELETE` ไว้แล้ว ก็ไม่จำเป็นต้องสร้าง trigger ใหม่ซ้ำอีก
</details>

---

### แบบฝึกหัดที่ 3
สร้าง `BEFORE INSERT` Trigger บนตาราง `products` เพื่อป้องกันไม่ให้เพิ่มสินค้าใหม่ที่มี `category_id` ชี้ไปยังหมวดหมู่ที่มีสถานะ "เป็นหมวดหมู่แม่ที่ไม่ควรใส่สินค้าโดยตรง" (สมมติว่าหมวดหมู่ที่มีหมวดหมู่ลูก คือ `parent_category_id` ของหมวดหมู่อื่นชี้มาหา ถือว่าเป็นหมวดหมู่แม่ ห้ามผูกสินค้ากับหมวดหมู่แม่โดยตรง ต้องผูกกับหมวดหมู่ลูกเท่านั้น)

<details>
<summary>เฉลย</summary>

```sql
CREATE OR REPLACE FUNCTION trg_validate_product_category()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
DECLARE
    v_is_parent BOOLEAN;
    v_category_name VARCHAR(100);
BEGIN
    SELECT EXISTS (
        SELECT 1 FROM categories child WHERE child.parent_category_id = NEW.category_id
    ) INTO v_is_parent;

    IF v_is_parent THEN
        SELECT category_name INTO v_category_name
        FROM categories WHERE category_id = NEW.category_id;

        RAISE EXCEPTION 'ไม่สามารถผูกสินค้ากับหมวดหมู่แม่ "%" ได้ กรุณาเลือกหมวดหมู่ย่อย',
            v_category_name
            USING ERRCODE = 'check_violation';
    END IF;

    RETURN NEW;
END;
$$;

CREATE TRIGGER trg_products_validate_category
    BEFORE INSERT OR UPDATE OF category_id ON products
    FOR EACH ROW
    EXECUTE FUNCTION trg_validate_product_category();

-- ทดสอบ: category_id = 1 ("อิเล็กทรอนิกส์") เป็นหมวดหมู่แม่ของ category_id 2 และ 3
INSERT INTO products (product_name, category_id, supplier_id, unit_price, stock_quantity)
VALUES ('ทดสอบ', 1, 1, 100.00, 10);
-- ERROR: ไม่สามารถผูกสินค้ากับหมวดหมู่แม่ "อิเล็กทรอนิกส์" ได้ กรุณาเลือกหมวดหมู่ย่อย
```
</details>

---

### แบบฝึกหัดที่ 4
สร้าง `BEFORE DELETE` Trigger บนตาราง `categories` ที่ป้องกันการลบหมวดหมู่ที่ยังมีสินค้า `is_active = true` ผูกอยู่ (cross-table validation)

<details>
<summary>เฉลย</summary>

```sql
CREATE OR REPLACE FUNCTION trg_prevent_delete_category_with_products()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
DECLARE
    v_active_count INTEGER;
BEGIN
    SELECT COUNT(*) INTO v_active_count
    FROM products
    WHERE category_id = OLD.category_id AND is_active = true;

    IF v_active_count > 0 THEN
        RAISE EXCEPTION 'ไม่สามารถลบหมวดหมู่ "%" ได้ เนื่องจากยังมีสินค้าที่เปิดขายอยู่ % รายการ',
            OLD.category_name, v_active_count
            USING ERRCODE = 'foreign_key_violation';
    END IF;

    RETURN OLD;
END;
$$;

CREATE TRIGGER trg_categories_prevent_delete
    BEFORE DELETE ON categories
    FOR EACH ROW
    EXECUTE FUNCTION trg_prevent_delete_category_with_products();

-- ทดสอบ
DELETE FROM categories WHERE category_id = 2;  -- คอมพิวเตอร์และแล็ปท็อป (มีสินค้า active อยู่)
-- ERROR: ไม่สามารถลบหมวดหมู่ "คอมพิวเตอร์และแล็ปท็อป" ได้ เนื่องจากยังมีสินค้าที่เปิดขายอยู่ 4 รายการ
```
</details>

---

### แบบฝึกหัดที่ 5
สร้าง `AFTER INSERT` Trigger ระดับ `STATEMENT` (ใช้ transition table) บนตาราง `order_items` ที่พิมพ์ (`RAISE NOTICE`) จำนวนแถวรวมและยอดรวม (`quantity * unit_price`) ของ order_items ที่ถูก insert ในคำสั่งเดียวกัน

<details>
<summary>เฉลย</summary>

```sql
CREATE OR REPLACE FUNCTION trg_summarize_order_items_insert()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
DECLARE
    v_row_count INTEGER;
    v_total_amount NUMERIC;
BEGIN
    SELECT COUNT(*), SUM(quantity * unit_price)
    INTO v_row_count, v_total_amount
    FROM inserted_rows;

    RAISE NOTICE 'INSERT order_items: % แถว รวมมูลค่า % บาท', v_row_count, v_total_amount;
    RETURN NULL;
END;
$$;

CREATE TRIGGER trg_order_items_summary
    AFTER INSERT ON order_items
    REFERENCING NEW TABLE AS inserted_rows
    FOR EACH STATEMENT
    EXECUTE FUNCTION trg_summarize_order_items_insert();

-- ทดสอบ: insert หลายแถวในคำสั่งเดียว
INSERT INTO order_items (order_id, product_id, quantity, unit_price)
VALUES (11, 12, 2, 1190.00), (11, 9, 1, 2990.00);
-- NOTICE: INSERT order_items: 2 แถว รวมมูลค่า 5370.00 บาท
```
</details>

---

### แบบฝึกหัดที่ 6
เขียนคำสั่ง query เพื่อแสดงรายการ Trigger ทั้งหมดในฐานข้อมูล พร้อมชื่อฟังก์ชันที่ผูกอยู่ เรียงตามชื่อตารางแล้วตามลำดับการทำงานจริง (timing, alphabetical)

<details>
<summary>เฉลย</summary>

```sql
SELECT
    c.relname                          AS table_name,
    t.tgname                           AS trigger_name,
    CASE WHEN t.tgtype & 2  = 2  THEN 'BEFORE'
         WHEN t.tgtype & 64 = 64 THEN 'INSTEAD OF'
         ELSE 'AFTER' END              AS timing,
    CASE WHEN t.tgtype & 1 = 1 THEN 'ROW' ELSE 'STATEMENT' END AS level,
    p.proname                          AS function_name,
    CASE t.tgenabled
        WHEN 'O' THEN 'enabled'
        WHEN 'D' THEN 'disabled'
        ELSE 'other'
    END                                 AS status
FROM pg_trigger t
JOIN pg_class c ON c.oid = t.tgrelid
JOIN pg_proc p ON p.oid = t.tgfoid
WHERE NOT t.tgisinternal
  AND c.relnamespace = 'public'::regnamespace
ORDER BY c.relname, timing, t.tgname;
```

หรือใช้มุมมองมาตรฐาน `information_schema.triggers` ที่อ่านง่ายกว่าแต่รายละเอียดน้อยกว่า:

```sql
SELECT event_object_table, trigger_name, action_timing, event_manipulation, action_statement
FROM information_schema.triggers
ORDER BY event_object_table, action_timing, trigger_name;
```
</details>

---

### แบบฝึกหัดที่ 7
สาธิตการปิด Trigger ชั่วคราวเพื่อทำ bulk update บนตาราง `products` (ลดราคาสินค้าทุกชิ้น 10% ในหมวดหมู่ "เสื้อผ้าผู้ชาย") โดยไม่ต้องการให้เกิด audit log รายแถว แต่ยังต้องการให้ `updated_at` เปลี่ยนตามปกติ จากนั้นเปิด trigger audit กลับคืน และตรวจสอบว่า trigger ที่ถูกปิดไม่ได้ทำงานจริง

<details>
<summary>เฉลย</summary>

```sql
-- ตรวจสอบสถานะ trigger ก่อนเริ่ม
SELECT tgname, tgenabled FROM pg_trigger
WHERE tgrelid = 'products'::regclass AND NOT tgisinternal;

BEGIN;

ALTER TABLE products DISABLE TRIGGER trg_products_audit;

UPDATE products
SET unit_price = ROUND(unit_price * 0.9, 2)
WHERE category_id = 7;  -- เสื้อผ้าผู้ชาย

ALTER TABLE products ENABLE TRIGGER trg_products_audit;

COMMIT;

-- ตรวจสอบว่า updated_at เปลี่ยน (trigger set_updated_at ยังทำงานปกติ เพราะไม่ได้ถูกปิด)
SELECT product_id, product_name, unit_price, updated_at
FROM products WHERE category_id = 7;

-- ตรวจสอบว่าไม่มี audit_log entry ใหม่สำหรับการเปลี่ยนแปลงนี้ (เพราะ trigger audit ถูกปิดระหว่าง update)
SELECT COUNT(*) FROM audit_log
WHERE table_name = 'products'
  AND changed_at >= now() - INTERVAL '1 minute';
-- ควรเป็น 0 หรือไม่รวมแถวจากคำสั่ง UPDATE ข้างต้น
```

จุดสำคัญคือ `DISABLE TRIGGER` ระบุชื่อ trigger เจาะจง (`trg_products_audit`) แทนการใช้ `ALL` เพื่อไม่ให้กระทบ trigger อื่น (เช่น `set_updated_at`) หรือ trigger ของ Foreign Key
</details>

---

### แบบฝึกหัดที่ 8
โค้ดต่อไปนี้มีบั๊ก (recursive trigger ที่จะเกิด infinite loop หรืออย่างน้อยก็ overhead ที่ไม่จำเป็น) จงหาปัญหาและแก้ไข:

```sql
CREATE OR REPLACE FUNCTION trg_buggy_price_rounder()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    UPDATE products SET unit_price = ROUND(unit_price, 0) WHERE product_id = NEW.product_id;
    RETURN NEW;
END;
$$;

CREATE TRIGGER trg_products_round_price
    AFTER UPDATE ON products
    FOR EACH ROW
    EXECUTE FUNCTION trg_buggy_price_rounder();
```

<details>
<summary>เฉลย</summary>

**ปัญหา**: Trigger นี้เป็น `AFTER UPDATE` บนตาราง `products` แต่ภายในฟังก์ชันกลับ `UPDATE products` **ซ้ำ** อีกครั้ง ซึ่งจะทำให้ trigger `trg_products_round_price` ถูกเรียกตัวเองซ้ำไปเรื่อย ๆ (recursive trigger) — แม้ว่าค่า `ROUND(unit_price, 0)` ในรอบที่สองอาจจะเท่าเดิมแล้ว (ทำให้ recursion หยุดเพราะไม่มี infinite เปลี่ยนแปลงค่า) แต่ก็ยังเป็นการ fire trigger ซ้ำโดยไม่จำเป็น เปลือง performance เสมอทุกครั้งที่มีการอัปเดตราคา และถ้าค่าที่ปัดเปลี่ยนไปเรื่อย ๆ ในบางกรณี (เช่น floating point edge case) ก็เสี่ยงเกิด infinite loop จริง ๆ ได้

**วิธีแก้ที่ถูกต้อง**: ใช้ `BEFORE UPDATE` แล้วแก้ไขค่าใน `NEW` โดยตรง แทนที่จะยิง `UPDATE` ซ้อนกลับเข้าตารางเดิม:

```sql
CREATE OR REPLACE FUNCTION trg_fixed_price_rounder()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    NEW.unit_price := ROUND(NEW.unit_price, 0);
    RETURN NEW;
END;
$$;

DROP TRIGGER trg_products_round_price ON products;
DROP FUNCTION trg_buggy_price_rounder();

CREATE TRIGGER trg_products_round_price
    BEFORE UPDATE ON products
    FOR EACH ROW
    WHEN (ROUND(NEW.unit_price, 0) IS DISTINCT FROM NEW.unit_price)  -- fire เฉพาะเมื่อจำเป็นต้องปัด
    EXECUTE FUNCTION trg_fixed_price_rounder();
```

วิธีนี้แก้ไขค่าก่อนที่จะเขียนลงตารางจริง (ภายใน transaction เดียวกัน ไม่มีการ `UPDATE` ซ้อนกลับเข้าตารางเดิม) จึงไม่มีความเสี่ยงเรื่อง recursion เลย และเร็วกว่าเดิมมาก เพราะไม่ต้องยิง `UPDATE` เพิ่มอีกรอบ
</details>

---

### แบบฝึกหัดที่ 9
สร้าง Trigger ที่ป้องกันการ `UPDATE` คอลัมน์ `order_id` หรือ `product_id` ในตาราง `order_items` (กล่าวคือ อนุญาตให้แก้ไขได้เฉพาะ `quantity` และ `unit_price` เท่านั้น เพราะการเปลี่ยนคำสั่งซื้อ/สินค้าของรายการที่มีอยู่แล้วถือว่าผิดปกติทางธุรกิจ ควรลบแล้วสร้างใหม่แทน)

<details>
<summary>เฉลย</summary>

```sql
CREATE OR REPLACE FUNCTION trg_prevent_order_item_key_change()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    IF NEW.order_id IS DISTINCT FROM OLD.order_id THEN
        RAISE EXCEPTION 'ห้ามแก้ไข order_id ของ order_items (order_item_id=%) กรุณาลบแล้วสร้างรายการใหม่แทน',
            OLD.order_item_id
            USING ERRCODE = 'restrict_violation';
    END IF;

    IF NEW.product_id IS DISTINCT FROM OLD.product_id THEN
        RAISE EXCEPTION 'ห้ามแก้ไข product_id ของ order_items (order_item_id=%) กรุณาลบแล้วสร้างรายการใหม่แทน',
            OLD.order_item_id
            USING ERRCODE = 'restrict_violation';
    END IF;

    RETURN NEW;
END;
$$;

CREATE TRIGGER trg_10_order_items_immutable_keys
    BEFORE UPDATE ON order_items
    FOR EACH ROW
    EXECUTE FUNCTION trg_prevent_order_item_key_change();

-- ทดสอบ
UPDATE order_items SET product_id = 5 WHERE order_item_id = 1;
-- ERROR: ห้ามแก้ไข product_id ของ order_items (order_item_id=1) กรุณาลบแล้วสร้างรายการใหม่แทน

UPDATE order_items SET quantity = 3 WHERE order_item_id = 1;  -- ทำงานปกติ ไม่ error
```

ชื่อ trigger ตั้งเป็น `trg_10_...` เพื่อให้ทำงาน**ก่อน** trigger อื่นที่อาจผูกกับ `BEFORE UPDATE` บนตารางเดียวกัน (เช่น trigger ปรับสต๊อกในกรณีที่ทำเป็น BEFORE) ตามหลักการจัดลำดับที่เรียนไปใน Step 478
</details>

---

### แบบฝึกหัดที่ 10
ออกแบบและสร้างระบบครบวงจร: Trigger บนตาราง `orders` ที่เมื่อ `status` เปลี่ยนเป็น `'cancelled'` (จากสถานะอื่นที่ไม่ใช่ `'cancelled'` มาก่อน) ให้คืนสต๊อกสินค้าทั้งหมดในคำสั่งซื้อนั้นกลับเข้า `products.stock_quantity` โดยอัตโนมัติ (ใช้ตรรกะคล้ายกับการ `DELETE` order_items แต่ในกรณีนี้ order_items ยังอยู่ครบ เพียงแต่คำสั่งซื้อถูกยกเลิก)

<details>
<summary>เฉลย</summary>

```sql
CREATE OR REPLACE FUNCTION trg_restock_on_order_cancel()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    -- ทำงานเฉพาะกรณีเปลี่ยนสถานะ "เป็น" cancelled จากสถานะอื่นที่ไม่ใช่ cancelled มาก่อน
    IF NEW.status = 'cancelled' AND OLD.status IS DISTINCT FROM 'cancelled' THEN
        UPDATE products p
        SET stock_quantity = p.stock_quantity + oi.quantity
        FROM order_items oi
        WHERE oi.order_id = NEW.order_id
          AND oi.product_id = p.product_id;

        RAISE NOTICE 'คำสั่งซื้อ #% ถูกยกเลิก: คืนสต๊อกสินค้าทั้งหมดในคำสั่งซื้อนี้แล้ว', NEW.order_id;

        INSERT INTO audit_log (table_name, operation, row_id, old_data, new_data)
        VALUES ('orders', 'CANCEL_RESTOCK', NEW.order_id,
                jsonb_build_object('status', OLD.status),
                jsonb_build_object('status', NEW.status, 'restocked_at', now()));
    END IF;

    RETURN NEW;
END;
$$;

CREATE TRIGGER trg_orders_restock_on_cancel
    AFTER UPDATE OF status ON orders
    FOR EACH ROW
    WHEN (NEW.status = 'cancelled' AND OLD.status IS DISTINCT FROM 'cancelled')
    EXECUTE FUNCTION trg_restock_on_order_cancel();

-- ทดสอบ
SELECT p.product_id, p.stock_quantity
FROM products p
JOIN order_items oi ON oi.product_id = p.product_id
WHERE oi.order_id = 9;   -- order_id = 9 มีสินค้า product_id 4 และ 3

UPDATE orders SET status = 'cancelled' WHERE order_id = 9;
-- NOTICE: คำสั่งซื้อ #9 ถูกยกเลิก: คืนสต๊อกสินค้าทั้งหมดในคำสั่งซื้อนี้แล้ว

SELECT p.product_id, p.stock_quantity
FROM products p
JOIN order_items oi ON oi.product_id = p.product_id
WHERE oi.order_id = 9;   -- stock_quantity เพิ่มขึ้นตาม quantity ในแต่ละ order_item

-- ทดสอบว่าถ้ายกเลิกซ้ำ (status เป็น cancelled อยู่แล้ว) จะไม่คืนสต๊อกซ้ำ
UPDATE orders SET status = 'cancelled' WHERE order_id = 9;  -- ไม่มี NOTICE เพราะ WHEN ไม่ผ่าน (OLD.status = 'cancelled' แล้ว)
```

จุดสำคัญของเฉลยนี้:
1. ใช้ `WHEN` clause กรองตั้งแต่ระดับ trigger definition (ประสิทธิภาพดีกว่าเช็คใน body ของฟังก์ชัน)
2. ป้องกันการคืนสต๊อกซ้ำด้วยเงื่อนไข `OLD.status IS DISTINCT FROM 'cancelled'` — สำคัญมาก เพราะถ้าไม่เช็ค การ `UPDATE` สถานะเป็น `'cancelled'` ซ้ำ ๆ จะทำให้สต๊อกถูกคืนซ้ำหลายรอบ ซึ่งเป็นบั๊กร้ายแรง
3. บันทึก audit log พิเศษ (`CANCEL_RESTOCK`) แยกจาก audit ปกติ เพื่อให้ตรวจสอบย้อนหลังได้ชัดเจนว่าทำไมสต๊อกถึงเปลี่ยน
4. ใช้ `UPDATE ... FROM` เพื่ออัปเดตหลายแถวของ `products` พร้อมกันในคำสั่งเดียว แทนการ loop ทีละแถวด้วย PL/pgSQL ซึ่งเร็วกว่า
</details>

---

**บทถัดไป**: [Part 049 — Custom Types และ Domains](./part-049-custom-types-domains.md)
