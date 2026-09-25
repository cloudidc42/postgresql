# Part 019: ALTER TABLE — เพิ่ม/ลบ/แก้ไขคอลัมน์และ Constraint

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับพื้นฐาน | Part 019

---

## เป้าหมายการเรียนรู้

เมื่อจบบทนี้ ผู้เรียนจะสามารถ:

1. เพิ่มคอลัมน์ใหม่ (`ADD COLUMN`) ให้ตารางที่มีข้อมูลอยู่แล้ว และเข้าใจว่าเมื่อไหร่ PostgreSQL ทำแบบ metadata-only (เร็วมาก) เมื่อไหร่ต้อง rewrite ตารางทั้งใบ (ช้า)
2. ลบคอลัมน์ (`DROP COLUMN`) อย่างปลอดภัย และเข้าใจผลกระทบต่อ view, index, constraint ที่พึ่งพาคอลัมน์นั้น
3. เปลี่ยนชนิดข้อมูลของคอลัมน์ (`ALTER COLUMN ... TYPE`) พร้อมใช้ `USING` clause เมื่อจำเป็น
4. ตั้ง/ยกเลิกค่า default และ `NOT NULL` ของคอลัมน์
5. เปลี่ยนชื่อคอลัมน์และชื่อตาราง (`RENAME COLUMN`, `RENAME TO`)
6. ย้ายตารางข้าม schema ด้วย `SET SCHEMA`
7. เพิ่ม/ลบ constraint ทุกประเภท (PK, FK, UNIQUE, CHECK) ผ่าน `ALTER TABLE`
8. เข้าใจผลกระทบด้าน performance ของ `ALTER TABLE` บนตารางขนาดใหญ่ — lock ประเภทไหนถูกใช้ เมื่อไหร่ต้อง rewrite ตารางทั้งใบ
9. เข้าใจแนวคิดพื้นฐานของ zero-downtime schema migration (จะเจาะลึกใน Part 079)
10. ฝึกวิวัฒนาการ schema ของระบบร้านกาแฟผ่านหลาย migration steps ต่อเนื่องกัน

---

## เตรียมข้อมูล

บทนี้ต่อเนื่องจากสคีมา "ร้านกาแฟ" (coffee shop) ที่เราใช้มาตลอดทั้งหลักสูตร ก่อนเริ่ม ให้สร้างฐานข้อมูลและตารางชุดนี้ขึ้นมาใหม่ (หากมีอยู่แล้วจากบทก่อนหน้า ข้ามส่วนนี้ไปได้เลย)

```sql
-- ล้างของเก่า (เผื่อรันซ้ำ) แล้วสร้างใหม่ทั้งหมด
DROP TABLE IF EXISTS order_items, orders, products, categories, customers, employees CASCADE;

-- หมวดหมู่สินค้า
CREATE TABLE categories (
    category_id   SERIAL PRIMARY KEY,
    category_name VARCHAR(50) NOT NULL UNIQUE,
    description   TEXT
);

-- สินค้า (เมนูเครื่องดื่ม/ขนม)
CREATE TABLE products (
    product_id   SERIAL PRIMARY KEY,
    product_name VARCHAR(100) NOT NULL,
    category_id  INTEGER REFERENCES categories(category_id),
    price        NUMERIC(10,2) NOT NULL CHECK (price >= 0),
    in_stock     BOOLEAN NOT NULL DEFAULT true
);

-- ลูกค้า
CREATE TABLE customers (
    customer_id  SERIAL PRIMARY KEY,
    full_name    VARCHAR(100) NOT NULL,
    email        VARCHAR(150) UNIQUE,
    phone        VARCHAR(10),
    created_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- พนักงาน
CREATE TABLE employees (
    employee_id  SERIAL PRIMARY KEY,
    full_name    VARCHAR(100) NOT NULL,
    position     VARCHAR(50) NOT NULL,
    hourly_wage  NUMERIC(10,2),
    hire_date    DATE NOT NULL DEFAULT CURRENT_DATE
);

-- ออเดอร์
CREATE TABLE orders (
    order_id     SERIAL PRIMARY KEY,
    customer_id  INTEGER REFERENCES customers(customer_id),
    employee_id  INTEGER REFERENCES employees(employee_id),
    order_date   TIMESTAMPTZ NOT NULL DEFAULT now(),
    status       VARCHAR(20) NOT NULL DEFAULT 'pending'
);

-- รายการสินค้าในแต่ละออเดอร์
CREATE TABLE order_items (
    order_item_id SERIAL PRIMARY KEY,
    order_id      INTEGER NOT NULL REFERENCES orders(order_id),
    product_id    INTEGER NOT NULL REFERENCES products(product_id),
    quantity      INTEGER NOT NULL CHECK (quantity > 0),
    unit_price    NUMERIC(10,2) NOT NULL
);

-- ข้อมูลตัวอย่างเล็กน้อย
INSERT INTO categories (category_name, description) VALUES
    ('กาแฟ', 'เครื่องดื่มกาแฟทุกชนิด'),
    ('ชา', 'เครื่องดื่มชาทุกชนิด'),
    ('เบเกอรี่', 'ขนมปังและของว่าง');

INSERT INTO products (product_name, category_id, price) VALUES
    ('เอสเพรสโซ่', 1, 45.00),
    ('ลาเต้', 1, 55.00),
    ('ชาไทย', 2, 50.00),
    ('ครัวซองต์', 3, 65.00);

INSERT INTO customers (full_name, email, phone) VALUES
    ('สมชาย ใจดี', 'somchai@example.com', '0812345678'),
    ('สมหญิง รักไทย', 'somying@example.com', '0898765432');

INSERT INTO employees (full_name, position, hourly_wage) VALUES
    ('บาริสต้าเอก', 'barista', 120.00),
    ('แคชเชียร์โบว์', 'cashier', 100.00);
```

ผลลัพธ์ที่ควรได้ (ตัวอย่าง):

```
CREATE TABLE
CREATE TABLE
CREATE TABLE
CREATE TABLE
CREATE TABLE
CREATE TABLE
INSERT 0 3
INSERT 0 4
INSERT 0 2
INSERT 0 2
```

ตรวจสอบว่าตารางถูกสร้างครบ:

```sql
\dt
```

```
              List of relations
 Schema |     Name      | Type  |  Owner
--------+---------------+-------+---------
 public | categories    | table | pguser
 public | customers     | table | pguser
 public | employees     | table | pguser
 public | order_items   | table | pguser
 public | orders        | table | pguser
 public | products      | table | pguser
(6 rows)
```

จากบทที่แล้ว (Part 018) เราเรียนรู้ constraint ต่าง ๆ ตอนสร้างตารางไปแล้ว บทนี้จะโฟกัสที่การ **แก้ไขโครงสร้างตารางที่มีอยู่แล้ว** ด้วยคำสั่ง `ALTER TABLE` ซึ่งเป็นทักษะที่ใช้บ่อยที่สุดในชีวิตจริง เพราะธุรกิจเปลี่ยนความต้องการตลอดเวลา ตารางที่ออกแบบไว้ตอนแรกแทบไม่เคยอยู่นิ่งตลอดอายุของระบบ

---

## Step 181: ALTER TABLE ADD COLUMN

### Syntax พื้นฐาน

```sql
ALTER TABLE table_name
    ADD COLUMN column_name data_type [column_constraint ...];
```

ตัวอย่าง: ร้านกาแฟอยากเริ่มระบบสะสมแต้ม จึงต้องเพิ่มคอลัมน์ `loyalty_points` ให้ตาราง `customers`

```sql
ALTER TABLE customers
    ADD COLUMN loyalty_points INTEGER NOT NULL DEFAULT 0;
```

```
ALTER TABLE
```

ตรวจสอบผล:

```sql
SELECT customer_id, full_name, loyalty_points FROM customers;
```

```
 customer_id |   full_name    | loyalty_points
-------------+----------------+----------------
           1 | สมชาย ใจดี     |              0
           2 | สมหญิง รักไทย  |              0
(2 rows)
```

### เพิ่มได้หลายคอลัมน์พร้อมกันในคำสั่งเดียว

```sql
ALTER TABLE products
    ADD COLUMN sku VARCHAR(20),
    ADD COLUMN weight_grams INTEGER;
```

```
ALTER TABLE
```

> **ทำไมต้องรวมหลาย ADD COLUMN ไว้ในคำสั่งเดียว?**
> เพราะ `ALTER TABLE` แต่ละคำสั่งต้องขอ `ACCESS EXCLUSIVE LOCK` (อธิบายละเอียดใน Step 188) การรวมหลายการเปลี่ยนแปลงไว้ใน statement เดียวหมายถึงขอ lock แค่ครั้งเดียว แทนที่จะขอหลายครั้งติดกัน ลดโอกาสเกิด lock queue บนตารางที่มี traffic สูง

### เรื่องสำคัญ: DEFAULT แบบไม่ rewrite ตารางทั้งใบ (PostgreSQL 11+)

ก่อน PostgreSQL 11 การเพิ่มคอลัมน์ที่มี `DEFAULT` (โดยเฉพาะกับตารางที่มีข้อมูลอยู่แล้ว) ต้อง **rewrite ตารางทั้งใบ** คือ PostgreSQL ต้องเขียนค่า default ลงไปทุกแถวที่มีอยู่จริง ซึ่งถ้าตารางมีข้อมูลหลายสิบล้านแถว การ `ALTER TABLE ADD COLUMN ... DEFAULT ...` แบบนี้จะ**ค้างนาน**และล็อกตารางตลอดเวลานั้น

ตั้งแต่ PostgreSQL 11 เป็นต้นมา ถ้า default value เป็น **ค่าคงที่ (constant)** ที่ไม่ผันแปร (non-volatile) เช่น ตัวเลข, string, boolean, `NULL` — PostgreSQL จะเก็บค่า default นั้นไว้ใน system catalog (`pg_attribute`) แทน โดยไม่ไปเขียนทับทุกแถวในตาราง การอ่านแถวเก่าจะได้ค่า default นี้แบบ "เสมือนมีอยู่จริง" (on-the-fly) จนกว่าจะมีการ `UPDATE` แถวนั้นจริง ๆ ทำให้คำสั่งนี้เป็น **metadata-only operation** เสร็จเกือบจะทันที ไม่ว่าตารางจะมีกี่ล้านแถวก็ตาม

```sql
-- นี่คือคำสั่งที่เร็วมาก แม้ customers จะมีล้านแถว เพราะ 0 เป็นค่าคงที่
ALTER TABLE customers ADD COLUMN vip_tier VARCHAR(20) NOT NULL DEFAULT 'standard';
```

```
ALTER TABLE
```

แต่ถ้า default เป็นฟังก์ชันแบบ **volatile** (ค่าที่เปลี่ยนไปในแต่ละแถว) เช่น `now()`, `random()`, `nextval()` (ยกเว้น `SERIAL`/`IDENTITY` ที่มีกลไกพิเศษของมันเอง) — PostgreSQL ยังคง**ต้อง rewrite ตารางทั้งใบ** เพราะแต่ละแถวต้องได้ค่าที่ไม่เหมือนกัน

```sql
-- คำสั่งนี้ "ต้อง" rewrite ตารางทั้งใบ เพราะ now() ต้อง evaluate ต่อแถว (จริง ๆ now() คงที่ตลอด transaction
-- แต่ PostgreSQL จัดหมวดฟังก์ชันนี้เป็น volatile-category ตาม pg_proc จึงเลือกเส้นทาง rewrite เพื่อความปลอดภัย)
ALTER TABLE products ADD COLUMN created_at TIMESTAMPTZ DEFAULT now();
```

```sql
-- คำสั่งนี้ "ต้อง" rewrite ตารางทั้งใบแน่นอน เพราะแต่ละแถวได้ค่าสุ่มไม่เท่ากัน
ALTER TABLE products ADD COLUMN random_code NUMERIC DEFAULT random();
```

ตารางสรุปเปรียบเทียบ:

| ลักษณะ DEFAULT | ตัวอย่าง | ต้อง rewrite ตารางหรือไม่ | ความเร็ว |
|---|---|---|---|
| ค่าคงที่ (constant) | `DEFAULT 0`, `DEFAULT 'active'`, `DEFAULT false`, `DEFAULT NULL` | ไม่ต้อง (PG 11+) | เร็วมาก (metadata-only) |
| ฟังก์ชัน volatile | `DEFAULT now()`, `DEFAULT random()`, `DEFAULT gen_random_uuid()` | ต้อง | ช้า ขึ้นกับขนาดตาราง |
| ฟังก์ชัน stable/immutable ที่ evaluate ได้ครั้งเดียว | บางกรณี PostgreSQL ยัง optimize ให้ | ขึ้นกับ planner | แล้วแต่กรณี |

> **Tips สำหรับมือโปร:** ตรวจสอบว่าคำสั่ง `ADD COLUMN` ของคุณ rewrite ตารางหรือไม่ได้ด้วยการดู `pg_stat_activity` ระหว่างรัน หรือใน production ที่ critical มาก ให้ทดสอบบน replica/staging ที่มีขนาดข้อมูลใกล้เคียงกับ production ก่อนเสมอ

---

## Step 182: ALTER TABLE DROP COLUMN

### Syntax พื้นฐาน

```sql
ALTER TABLE table_name DROP COLUMN column_name [RESTRICT | CASCADE];
```

ตัวอย่าง: สมมติเราตัดสินใจว่าจะไม่เก็บ `weight_grams` ของสินค้าแล้ว

```sql
ALTER TABLE products DROP COLUMN weight_grams;
```

```
ALTER TABLE
```

### ผลกระทบต่อสิ่งที่พึ่งพาคอลัมน์นั้น

`DROP COLUMN` มีผลกระทบเป็นลูกโซ่ต่อวัตถุอื่นที่อ้างอิงคอลัมน์นั้นอยู่ มาดูทีละกรณี

**1) Index ที่สร้างจากคอลัมน์นั้น** — ถูกลบอัตโนมัติไปพร้อมกับคอลัมน์ ไม่ต้องใช้ `CASCADE`

```sql
CREATE INDEX idx_products_sku ON products(sku);

ALTER TABLE products DROP COLUMN sku;
```

```
CREATE INDEX
ALTER TABLE
```

ตรวจสอบว่า index หายไปด้วย:

```sql
\d products
```

```
                                     Table "public.products"
   Column     |     Type      | Collation | Nullable |                Default
--------------+---------------+-----------+----------+----------------------------------------
 product_id   | integer       |           | not null | nextval('products_product_id_seq'...)
 product_name | character varying(100) |  | not null |
 category_id  | integer       |           |          |
 price        | numeric(10,2) |           | not null |
 in_stock     | boolean       |           | not null | true
-- (สังเกตว่าไม่มี sku และไม่มี index idx_products_sku แล้ว)
```

**2) Constraint ที่ใช้คอลัมน์นั้น** — เช่น `CHECK`, `UNIQUE` ที่อ้างอิงคอลัมน์เดียวจะถูกลบอัตโนมัติ; ถ้าเป็น `PRIMARY KEY` หรือ `FOREIGN KEY` ที่มีตารางอื่นอ้างอิงต่อ (referenced by) จะ**ไม่**ให้ลบตรง ๆ (ต้อง `CASCADE` และควรระวังมาก)

**3) View ที่ SELECT คอลัมน์นั้น** — จะทำให้ `DROP COLUMN` **ล้มเหลว** ถ้าไม่ใส่ `CASCADE`

```sql
CREATE VIEW customer_contact AS
    SELECT customer_id, full_name, email, phone FROM customers;

-- ลองลบ phone โดยไม่ใส่ CASCADE
ALTER TABLE customers DROP COLUMN phone;
```

```
ERROR:  cannot drop column phone of table customers because other objects depend on it
DETAIL:  view customer_contact depends on column phone of table customers
HINT:  Use DROP ... CASCADE to drop the dependent objects too.
```

ถ้ายืนยันจะลบ ต้องใช้ `CASCADE` ซึ่งจะ**ลบ view นั้นไปด้วย**

```sql
ALTER TABLE customers DROP COLUMN phone CASCADE;
```

```
NOTICE:  drop cascades to view customer_contact
ALTER TABLE
```

> **คำเตือน:** `CASCADE` ใน `DROP COLUMN` เป็นคำสั่งที่อันตรายในระบบ production เพราะมันลบ view/rule/trigger ที่พึ่งพาคอลัมน์นั้นไปเงียบ ๆ โดยไม่ถามซ้ำ ก่อนใช้ควรรัน query ตรวจสอบ dependency ก่อนเสมอ:
> ```sql
> SELECT DISTINCT dependent_ns.nspname AS dependent_schema,
>        dependent_view.relname     AS dependent_view
> FROM pg_depend
> JOIN pg_rewrite ON pg_depend.objid = pg_rewrite.oid
> JOIN pg_class AS dependent_view ON pg_rewrite.ev_class = dependent_view.oid
> JOIN pg_class AS source_table   ON pg_depend.refobjid = source_table.oid
> JOIN pg_attribute ON pg_depend.refobjid = pg_attribute.attrelid
>                   AND pg_depend.refobjsubid = pg_attribute.attnum
> JOIN pg_namespace dependent_ns  ON dependent_view.relnamespace = dependent_ns.oid
> WHERE source_table.relname = 'customers'
>   AND pg_attribute.attname = 'email';
> ```

### DROP COLUMN ไม่คืนพื้นที่ดิสก์ทันที

สิ่งที่คนมักเข้าใจผิด: `DROP COLUMN` **ไม่ใช่** การ rewrite ตารางทั้งใบ และ**ไม่ใช่**การลบข้อมูลจริงออกจาก disk ทันที สิ่งที่เกิดขึ้นจริงคือ PostgreSQL แค่ทำเครื่องหมายคอลัมน์นั้นว่า `attisdropped = true` ใน system catalog `pg_attribute` — ข้อมูลจริงในแต่ละแถวยังคงอยู่ในไฟล์ตารางจนกว่าจะมีการ `UPDATE` แถวนั้น (ซึ่งจะเขียนแถวใหม่โดยไม่มีคอลัมน์นั้น) หรือมี `VACUUM FULL` / `CLUSTER` มา rewrite ตารางทั้งใบ

```sql
SELECT attname, attisdropped
FROM pg_attribute
WHERE attrelid = 'customers'::regclass
ORDER BY attnum;
```

```
     attname      | attisdropped
-------------------+--------------
 customer_id       | f
 full_name         | f
 email             | f
 created_at        | f
 loyalty_points    | f
 vip_tier          | f
 ........pg.dropped.4........ | t
(7 rows)
```

สังเกตว่า PostgreSQL เปลี่ยนชื่อคอลัมน์ `phone` ที่ถูกลบเป็น `........pg.dropped.4........` แทนที่จะลบ metadata ทิ้งไปเลย นี่คือเหตุผลที่ `DROP COLUMN` เป็นการดำเนินการที่**เร็ว**แม้ตารางจะใหญ่แค่ไหนก็ตาม (เป็น metadata-only เหมือนกับ ADD COLUMN แบบ default คงที่)

---

## Step 183: ALTER TABLE ALTER COLUMN TYPE

### Syntax พื้นฐาน

```sql
ALTER TABLE table_name
    ALTER COLUMN column_name TYPE new_data_type [USING expression];
```

### กรณีที่ไม่ต้องใช้ USING (แปลงได้อัตโนมัติ)

การขยายความยาว `VARCHAR` หรือเปลี่ยนจาก `VARCHAR(n)` เป็น `TEXT` ไม่ต้องใช้ `USING` เพราะ PostgreSQL แปลงให้อัตโนมัติแบบ implicit cast และในหลายกรณีเป็น metadata-only ด้วย (ดู Step 188)

```sql
-- ขยายความยาว phone ให้รองรับเบอร์ต่างประเทศได้
ALTER TABLE customers ADD COLUMN phone VARCHAR(10);  -- เผื่อ demo ต่อ (เพิ่มกลับมาใหม่หลังจากลบไปใน Step 182)

ALTER TABLE customers ALTER COLUMN phone TYPE VARCHAR(20);
```

```
ALTER TABLE
ALTER TABLE
```

### กรณีที่ต้องใช้ USING (แปลงข้ามชนิดข้อมูล)

เมื่อ PostgreSQL ไม่รู้ว่าจะแปลงข้อมูลเดิมอย่างไร (เช่น จาก `TEXT` เป็น `INTEGER`) เราต้องระบุ `USING` เพื่อบอกวิธีแปลงค่าทุกแถวที่มีอยู่แล้ว

สมมติ ตอนแรกออกแบบ `hourly_wage` เป็น `VARCHAR` ผิดพลาด (ในโลกจริงเกิดขึ้นได้บ่อย):

```sql
-- จำลองสถานการณ์: สมมติมีตาราง staging ที่เก็บ wage เป็น text
CREATE TABLE wage_staging (
    employee_id INTEGER,
    wage_text   TEXT
);

INSERT INTO wage_staging VALUES (1, '120.00'), (2, '100.50');

-- ลองแปลง wage_text เป็น numeric ตรง ๆ โดยไม่ใช้ USING
ALTER TABLE wage_staging ALTER COLUMN wage_text TYPE NUMERIC(10,2);
```

```
ERROR:  column "wage_text" cannot be cast automatically to type numeric
HINT:  You might need to specify "USING wage_text::numeric".
```

PostgreSQL บอก hint มาให้เลยว่าต้องใช้ `USING` อย่างไร:

```sql
ALTER TABLE wage_staging
    ALTER COLUMN wage_text TYPE NUMERIC(10,2) USING wage_text::numeric;
```

```
ALTER TABLE
```

```sql
\d wage_staging
```

```
             Table "public.wage_staging"
   Column    |     Type      | Collation | Nullable | Default
-------------+---------------+-----------+----------+---------
 employee_id | integer       |           |          |
 wage_text   | numeric(10,2) |           |          |
```

### ตัวอย่าง USING ที่ซับซ้อนขึ้น

```sql
-- แปลง status ของ orders จาก VARCHAR เป็น ENUM ที่จำกัดค่าได้ชัดเจนขึ้น
CREATE TYPE order_status AS ENUM ('pending', 'preparing', 'ready', 'completed', 'cancelled');

ALTER TABLE orders
    ALTER COLUMN status TYPE order_status USING status::order_status;
```

```
CREATE TYPE
ALTER TABLE
```

```sql
-- อีกตัวอย่าง: แปลง TEXT ที่เก็บวันที่ในรูปแบบ 'YYYY-MM-DD' เป็น DATE จริง
-- (สมมติมีคอลัมน์ hire_date_text เก็บ '2024-01-15')
ALTER TABLE employees ADD COLUMN hire_date_text TEXT DEFAULT '2024-01-15';

ALTER TABLE employees
    ALTER COLUMN hire_date_text TYPE DATE USING hire_date_text::date;
```

```
ALTER TABLE
ALTER TABLE
```

> **ข้อควรระวัง:** ถ้าข้อมูลบางแถวแปลงไม่ได้ (เช่น text ที่ไม่ใช่ตัวเลขจริง หรือ enum ที่ไม่มีค่านั้นอยู่) คำสั่งทั้งหมดจะ **rollback ทั้ง transaction** — `ALTER COLUMN TYPE` จะทำ full table scan เพื่อ validate และแปลงทุกแถว ถ้ามีแถวเดียวแปลงไม่ผ่าน ทุกอย่างล้มเหลว ควร clean ข้อมูลให้เรียบร้อยก่อนเปลี่ยน type บนตาราง production เสมอ

---

## Step 184: SET/DROP DEFAULT และ SET/DROP NOT NULL

### SET DEFAULT / DROP DEFAULT

การเปลี่ยนค่า default ของคอลัมน์ที่มีอยู่แล้ว ไม่กระทบแถวเดิม มีผลแค่กับ `INSERT` ใหม่ในอนาคต และเป็น metadata-only เสมอ (เร็วมาก ไม่ว่าตารางจะใหญ่แค่ไหน)

```sql
-- ตั้งค่า default ใหม่ให้ hourly_wage เป็นอัตราขั้นต่ำมาตรฐาน
ALTER TABLE employees ALTER COLUMN hourly_wage SET DEFAULT 100.00;
```

```
ALTER TABLE
```

```sql
-- ทดสอบ: เพิ่มพนักงานใหม่โดยไม่ระบุ hourly_wage
INSERT INTO employees (full_name, position) VALUES ('น้องพลอย', 'barista');

SELECT employee_id, full_name, hourly_wage FROM employees;
```

```
INSERT 0 1
 employee_id |    full_name    | hourly_wage
-------------+------------------+-------------
           1 | บาริสต้าเอก     |      120.00
           2 | แคชเชียร์โบว์   |      100.00
           3 | น้องพลอย        |      100.00
(3 rows)
```

ยกเลิก default (กลับไปเป็น `NULL` เมื่อไม่ระบุค่า):

```sql
ALTER TABLE employees ALTER COLUMN hourly_wage DROP DEFAULT;
```

```
ALTER TABLE
```

### SET NOT NULL / DROP NOT NULL

การเพิ่ม `NOT NULL` ให้คอลัมน์ที่มีอยู่แล้ว **ต้องตรวจสอบว่าไม่มีแถวไหนเป็น `NULL` อยู่เลย** ถ้ามี จะ error ทันที

```sql
-- ลองบังคับ hourly_wage ห้ามเป็น NULL ทั้งที่ยังมีบางแถวเป็น NULL อยู่ (เพราะเรา DROP DEFAULT ไปแล้ว
-- และ น้องพลอย ถูก insert ตอนที่ยังมี default อยู่ จึงไม่เป็น NULL -- ลอง insert แถวใหม่ที่เป็น NULL ก่อน)
INSERT INTO employees (full_name, position, hourly_wage) VALUES ('พี่มด', 'manager', NULL);

ALTER TABLE employees ALTER COLUMN hourly_wage SET NOT NULL;
```

```
INSERT 0 1
ERROR:  column "hourly_wage" of relation "employees" contains null values
```

ต้อง backfill ข้อมูลให้ครบก่อน แล้วค่อยตั้ง `NOT NULL`:

```sql
UPDATE employees SET hourly_wage = 100.00 WHERE hourly_wage IS NULL;

ALTER TABLE employees ALTER COLUMN hourly_wage SET NOT NULL;
```

```
UPDATE 1
ALTER TABLE
```

ยกเลิก `NOT NULL` (อนุญาตให้เป็น `NULL` ได้อีกครั้ง):

```sql
ALTER TABLE employees ALTER COLUMN hourly_wage DROP NOT NULL;
```

```
ALTER TABLE
```

> **หมายเหตุเชิงลึก (PostgreSQL 12+):** ปกติ `SET NOT NULL` ต้องสแกนทั้งตารางเพื่อตรวจสอบว่าไม่มี `NULL` เลย ซึ่งบนตารางใหญ่ใช้เวลานานและถือ lock ตลอดการสแกน แต่ถ้าตารางมี **`CHECK` constraint ที่ validate แล้ว** ซึ่งเทียบเท่ากับ `NOT NULL` (เช่น `CHECK (col IS NOT NULL)`) อยู่ก่อนแล้ว PostgreSQL 12 ขึ้นไปจะ**ข้ามการสแกนซ้ำ** เพราะรู้อยู่แล้วว่าไม่มี `NULL` แน่นอน — นี่คือเทคนิคสำคัญสำหรับ zero-downtime migration ที่จะพูดถึงใน Step 189

```sql
-- เทคนิค: เพิ่ม CHECK NOT VALID ก่อน (เร็ว ไม่ scan) แล้ว VALIDATE (scan แต่ lock เบา)
-- จากนั้น SET NOT NULL จะข้าม scan ไปเลย เพราะรู้ผลจาก constraint ที่ validate แล้ว
ALTER TABLE employees ADD CONSTRAINT hourly_wage_not_null_check
    CHECK (hourly_wage IS NOT NULL) NOT VALID;

ALTER TABLE employees VALIDATE CONSTRAINT hourly_wage_not_null_check;

ALTER TABLE employees ALTER COLUMN hourly_wage SET NOT NULL;

-- หลังจากนั้นลบ CHECK constraint ที่ใช้เป็นสะพานออกได้ เพราะ NOT NULL ทำหน้าที่แทนแล้ว
ALTER TABLE employees DROP CONSTRAINT hourly_wage_not_null_check;
```

```
ALTER TABLE
ALTER TABLE
ALTER TABLE
ALTER TABLE
```

---

## Step 185: RENAME COLUMN, RENAME TO

### เปลี่ยนชื่อคอลัมน์

```sql
ALTER TABLE table_name RENAME COLUMN old_name TO new_name;
```

ตัวอย่าง: เปลี่ยนชื่อ `full_name` เป็น `customer_name` ให้สื่อความหมายชัดเจนขึ้น

```sql
ALTER TABLE customers RENAME COLUMN full_name TO customer_name;
```

```
ALTER TABLE
```

```sql
SELECT customer_id, customer_name FROM customers LIMIT 2;
```

```
 customer_id | customer_name
-------------+----------------
           1 | สมชาย ใจดี
           2 | สมหญิง รักไทย
(2 rows)
```

เปลี่ยนกลับเพื่อความต่อเนื่องกับบทอื่นในหลักสูตร:

```sql
ALTER TABLE customers RENAME COLUMN customer_name TO full_name;
```

```
ALTER TABLE
```

### เปลี่ยนชื่อตาราง

```sql
ALTER TABLE table_name RENAME TO new_table_name;
```

ตัวอย่าง: สมมติจะเปลี่ยนชื่อตาราง `employees` เป็น `staff` เพื่อให้ตรงกับศัพท์ธุรกิจใหม่

```sql
ALTER TABLE employees RENAME TO staff;
```

```
ALTER TABLE
```

```sql
\dt staff
```

```
        List of relations
 Schema | Name  | Type  |  Owner
--------+-------+-------+---------
 public | staff | table | pguser
(1 row)
```

เปลี่ยนกลับเพื่อความต่อเนื่องกับบทอื่น:

```sql
ALTER TABLE staff RENAME TO employees;
```

```
ALTER TABLE
```

### RENAME เป็น metadata-only แต่ต้องระวังเรื่อง dependency

ทั้ง `RENAME COLUMN` และ `RENAME TO` เป็นการดำเนินการที่ **เร็วมาก** เพราะเป็นแค่การแก้ชื่อใน system catalog (`pg_attribute.attname`, `pg_class.relname`) — ไม่กระทบข้อมูลจริงเลย

สิ่งที่น่าสนใจคือ **view ที่พึ่งพาคอลัมน์นั้นจะไม่พัง** เพราะภายใน view PostgreSQL เก็บ reference เป็น `attnum` (ลำดับคอลัมน์) ไม่ใช่ชื่อ — ระบบจะ resolve ชื่อใหม่ให้อัตโนมัติ

```sql
CREATE VIEW customer_email_list AS
    SELECT customer_id, email FROM customers;

ALTER TABLE customers RENAME COLUMN email TO email_address;

-- view ยังทำงานได้ปกติ แม้จะ SELECT ชื่อคอลัมน์เดิมตอนสร้าง
SELECT * FROM customer_email_list LIMIT 1;
```

```
ALTER TABLE
 customer_id |    email
-------------+---------------------
           1 | somchai@example.com
(1 row)
```

> **แต่ระวัง:** สิ่งที่ **พังแน่นอน** เมื่อ rename คือ **โค้ดฝั่งแอปพลิเคชัน** ที่ระบุชื่อคอลัมน์ตรง ๆ (`SELECT full_name FROM customers`), ORM mapping, และ query ที่เขียนด้วยมือใน stored procedure/function ที่อ้างชื่อคอลัมน์แบบ hardcode ควรตรวจสอบ dependency ทั้งหมดก่อน rename บน production เสมอ

เปลี่ยนกลับ:

```sql
ALTER TABLE customers RENAME COLUMN email_address TO email;
DROP VIEW customer_email_list;
```

```
ALTER TABLE
DROP VIEW
```

---

## Step 186: ALTER TABLE SET SCHEMA

### แนวคิด

Schema เปรียบเสมือน "โฟลเดอร์" ที่ใช้จัดกลุ่มตารางภายในฐานข้อมูลเดียวกัน (ทบทวนได้จาก Part 005) การย้ายตารางข้าม schema มีประโยชน์มากเวลาต้องการ:

- แยกตารางที่ archive แล้วออกจากตารางที่ใช้งานจริง
- จัดโครงสร้างข้อมูลใหม่ตาม domain (เช่น `sales`, `inventory`, `hr`)
- เตรียมข้อมูลสำหรับ multi-tenant architecture

### Syntax

```sql
ALTER TABLE table_name SET SCHEMA new_schema;
```

ตัวอย่าง: ร้านกาแฟต้องการเก็บออเดอร์เก่าที่จบแล้วไว้ใน schema แยกชื่อ `archive`

```sql
CREATE SCHEMA IF NOT EXISTS archive;

-- สมมติสร้างตารางสำหรับเก็บออเดอร์เก่าที่ปิดจบแล้ว (โครงสร้างเหมือน orders)
CREATE TABLE completed_orders_2024 (
    order_id     INTEGER PRIMARY KEY,
    customer_id  INTEGER,
    employee_id  INTEGER,
    order_date   TIMESTAMPTZ,
    status       VARCHAR(20)
);

ALTER TABLE completed_orders_2024 SET SCHEMA archive;
```

```
CREATE SCHEMA
CREATE TABLE
ALTER TABLE
```

ตรวจสอบว่าตารางย้ายไปอยู่ schema ใหม่แล้ว:

```sql
SELECT table_schema, table_name
FROM information_schema.tables
WHERE table_name = 'completed_orders_2024';
```

```
 table_schema |       table_name
--------------+------------------------
 archive      | completed_orders_2024
(1 row)
```

ตอนนี้ต้องอ้างอิงตารางแบบเต็ม (qualified name) หรือปรับ `search_path`:

```sql
SELECT * FROM archive.completed_orders_2024;
```

### คุณสมบัติสำคัญของ SET SCHEMA

1. **เป็น metadata-only เสมอ** — ไม่ว่าตารางจะมีข้อมูลกี่ล้านแถว การย้าย schema ใช้เวลาแทบจะ instant เพราะ PostgreSQL แค่เปลี่ยนค่า `relnamespace` ใน `pg_class` ไม่มีการย้ายไฟล์ข้อมูลจริงบน disk (ตราบใดที่ schema เดิมและใหม่อยู่ใน database เดียวกัน และไม่เกี่ยวกับ tablespace)
2. **ต้องมีสิทธิ์ที่เหมาะสม** — ผู้ใช้ต้องเป็นเจ้าของตาราง (หรือมีสิทธิ์ superuser/role ที่เหมาะสม) และต้องมีสิทธิ์ `CREATE` บน schema ปลายทาง
3. **ไม่ย้าย dependent objects โดยอัตโนมัติเสมอไป** — index, constraint, trigger ที่อยู่ใน schema เดียวกับตารางจะย้ายตามไปด้วย แต่ view/function ที่อยู่ schema อื่นและอ้างอิงตารางนี้แบบไม่ qualify ชื่อ (ใช้ `search_path`) อาจหาไม่เจอถ้า `search_path` ไม่ครอบคลุม schema ใหม่

```sql
-- ทดสอบสิทธิ์ที่ไม่พอ (ตัวอย่างข้อความ error ถ้าไม่มีสิทธิ์ CREATE บน schema ปลายทาง)
-- ALTER TABLE some_table SET SCHEMA restricted_schema;
-- ERROR:  permission denied for schema restricted_schema
```

> **ตัวอย่างการใช้งานจริง:** บริษัทหลายแห่งใช้แพทเทิร์นนี้สำหรับ "table partition ตามปี" แบบ manual — สร้างตารางใหม่ทุกปีใน schema หลัก แล้วพอผ่านไป 1-2 ปี ก็ `SET SCHEMA` ย้ายไปยัง schema `archive` เพื่อแยกออกจาก backup/query workload หลัก โดยที่ query เดิมยังทำงานได้ถ้าปรับ `search_path` หรือ view ให้รองรับ

---

## Step 187: การเพิ่ม/ลบ Constraint ทั้งหมดผ่าน ALTER TABLE

บทนี้เป็นการทบทวนรวม (comprehensive review) การเพิ่ม/ลบ constraint ทุกประเภทที่เรียนมาใน Part 018 แต่คราวนี้เน้นที่การ**เพิ่มทีหลัง** บนตารางที่มีอยู่แล้ว ซึ่งต่างจากตอนสร้างตารางตรงที่ต้องคำนึงถึงข้อมูลเดิมที่มีอยู่ด้วย

### PRIMARY KEY

```sql
-- สมมติตาราง wage_staging ที่สร้างไว้ก่อนหน้ายังไม่มี PK
ALTER TABLE wage_staging ADD CONSTRAINT wage_staging_pkey PRIMARY KEY (employee_id);
```

```
ALTER TABLE
```

ลบ Primary Key:

```sql
ALTER TABLE wage_staging DROP CONSTRAINT wage_staging_pkey;
```

```
ALTER TABLE
```

### FOREIGN KEY

```sql
-- เพิ่มความสัมพันธ์ order_items -> products ถ้ายังไม่มี (ในที่นี้มีอยู่แล้วจากตอนสร้างตาราง
-- แสดงตัวอย่างการเพิ่มบน column ใหม่แทน)
ALTER TABLE order_items ADD COLUMN prepared_by INTEGER;

ALTER TABLE order_items
    ADD CONSTRAINT fk_order_items_employee
    FOREIGN KEY (prepared_by) REFERENCES employees(employee_id);
```

```
ALTER TABLE
ALTER TABLE
```

ลบ Foreign Key:

```sql
ALTER TABLE order_items DROP CONSTRAINT fk_order_items_employee;
```

```
ALTER TABLE
```

### UNIQUE

```sql
ALTER TABLE products ADD CONSTRAINT uq_product_name UNIQUE (product_name);
```

```
ALTER TABLE
```

ลบ Unique constraint:

```sql
ALTER TABLE products DROP CONSTRAINT uq_product_name;
```

```
ALTER TABLE
```

### CHECK

```sql
ALTER TABLE order_items
    ADD CONSTRAINT chk_unit_price_positive CHECK (unit_price > 0);
```

```
ALTER TABLE
```

ลบ Check constraint:

```sql
ALTER TABLE order_items DROP CONSTRAINT chk_unit_price_positive;
```

### ตารางสรุป syntax ทั้งหมด

| Constraint | เพิ่ม | ลบ |
|---|---|---|
| PRIMARY KEY | `ADD CONSTRAINT name PRIMARY KEY (col)` | `DROP CONSTRAINT name` |
| FOREIGN KEY | `ADD CONSTRAINT name FOREIGN KEY (col) REFERENCES tbl(col)` | `DROP CONSTRAINT name` |
| UNIQUE | `ADD CONSTRAINT name UNIQUE (col)` | `DROP CONSTRAINT name` |
| CHECK | `ADD CONSTRAINT name CHECK (expr)` | `DROP CONSTRAINT name` |
| NOT NULL | `ALTER COLUMN col SET NOT NULL` | `ALTER COLUMN col DROP NOT NULL` |
| DEFAULT | `ALTER COLUMN col SET DEFAULT val` | `ALTER COLUMN col DROP DEFAULT` |

> **หมายเหตุ:** `NOT NULL` และ `DEFAULT` ทางเทคนิคไม่ได้เก็บในรูปแบบ named constraint แบบเดียวกับ PK/FK/UNIQUE/CHECK (ไม่มีชื่อใน `pg_constraint` ยกเว้น `NOT NULL` ตั้งแต่ PostgreSQL 18 ที่เริ่มเก็บเป็น constraint ที่มีชื่อได้ในบาง build) จึงใช้ syntax `ALTER COLUMN` แทน `ADD/DROP CONSTRAINT`

### ตรวจสอบ constraint ทั้งหมดของตาราง

```sql
SELECT conname, contype, pg_get_constraintdef(oid) AS definition
FROM pg_constraint
WHERE conrelid = 'order_items'::regclass;
```

```
      conname       | contype |                       definition
---------------------+---------+----------------------------------------------------------
 order_items_pkey    | p       | PRIMARY KEY (order_item_id)
 order_items_order_id_fkey   | f | FOREIGN KEY (order_id) REFERENCES orders(order_id)
 order_items_product_id_fkey | f | FOREIGN KEY (product_id) REFERENCES products(product_id)
 order_items_quantity_check  | c | CHECK (quantity > 0)
(4 rows)
```

(ตัวอักษรใน `contype`: `p` = primary key, `f` = foreign key, `u` = unique, `c` = check)

---

## Step 188: ผลกระทบด้าน Performance ของ ALTER TABLE บนตารางขนาดใหญ่

นี่คือหัวข้อที่ **สำคัญที่สุด** ในบทนี้สำหรับคนที่จะทำงานกับระบบ production จริง เพราะ `ALTER TABLE` ที่ดูเหมือนคำสั่งง่าย ๆ อาจทำให้ระบบทั้งระบบ**ค้างหลายนาทีหรือหลายชั่วโมง**ได้ถ้าใช้ผิดวิธีบนตารางที่มีข้อมูลมากและมี traffic สูง

### ACCESS EXCLUSIVE LOCK คืออะไร

คำสั่ง `ALTER TABLE` ส่วนใหญ่ต้องการ **`ACCESS EXCLUSIVE LOCK`** ซึ่งเป็น lock ระดับสูงสุดในระบบ lock ของ PostgreSQL — มันบล็อกการดำเนินการ**ทุกชนิด**บนตารางนั้น ไม่ว่าจะเป็น `SELECT`, `INSERT`, `UPDATE`, `DELETE` หรือแม้แต่ query ที่แค่จะอ่าน schema ของตาราง

```sql
BEGIN;
ALTER TABLE orders ADD COLUMN notes TEXT;
-- ระหว่างที่ transaction นี้ยังไม่ COMMIT/ROLLBACK
-- query อื่นที่พยายาม SELECT จาก orders จะถูกบล็อกรอคิวอยู่
COMMIT;
```

ตรวจสอบ lock ที่ถืออยู่ระหว่าง transaction (เปิด session ที่ 2 มาดู):

```sql
SELECT locktype, relation::regclass, mode, granted
FROM pg_locks
WHERE relation = 'orders'::regclass;
```

```
 locktype |  relation  |        mode         | granted
----------+------------+----------------------+---------
 relation | orders     | AccessExclusiveLock  | t
(1 row)
```

### จำแนก: metadata-only เร็ว vs table rewrite ช้า

| คำสั่ง | ต้อง rewrite ตาราง? | ระยะเวลาที่ถือ lock | หมายเหตุ |
|---|---|---|---|
| `ADD COLUMN` + default คงที่หรือไม่มี default | ไม่ต้อง | สั้นมาก (metadata) | PG 11+ |
| `ADD COLUMN` + default แบบ volatile | ต้อง | เท่าเวลาเขียนทุกแถว | เลี่ยงถ้าเป็นไปได้ |
| `DROP COLUMN` | ไม่ต้อง | สั้นมาก (metadata) | ข้อมูลจริงลบทีหลังตอน VACUUM/UPDATE |
| `RENAME COLUMN` / `RENAME TO` | ไม่ต้อง | สั้นมาก | แค่แก้ catalog |
| `SET SCHEMA` | ไม่ต้อง | สั้นมาก | แค่แก้ catalog |
| `ALTER COLUMN TYPE` (ขยาย varchar length, varchar→text) | ไม่ต้อง* | สั้นมาก | *ต้องเช็ค version และ compatibility |
| `ALTER COLUMN TYPE` (เปลี่ยนชนิดข้อมูลจริง เช่น text→int, numeric precision เปลี่ยน) | ต้อง | เท่าเวลาสแกน+เขียนทุกแถว | คำสั่งหนักสุดในกลุ่มนี้ |
| `SET/DROP DEFAULT` | ไม่ต้อง | สั้นมาก | ไม่กระทบแถวเดิม |
| `SET NOT NULL` | ไม่ต้อง rewrite แต่ต้อง**สแกนตรวจสอบ** | เท่าเวลาสแกนทั้งตาราง (แต่ไม่เขียน) | ข้ามได้ถ้ามี valid CHECK constraint แล้ว (PG12+) |
| `DROP NOT NULL` | ไม่ต้อง | สั้นมาก | |
| `ADD CONSTRAINT ... CHECK/FK` (แบบปกติ) | ไม่ rewrite แต่**สแกนตรวจสอบ** | เท่าเวลาสแกนทั้งตาราง | ถือ ACCESS EXCLUSIVE ตลอดการสแกน |
| `ADD CONSTRAINT ... NOT VALID` | ไม่ต้อง | สั้นมาก | ตรวจสอบทีหลังด้วย `VALIDATE CONSTRAINT` |
| `VALIDATE CONSTRAINT` | ไม่ rewrite | เท่าเวลาสแกน แต่ใช้ `SHARE UPDATE EXCLUSIVE` (เบากว่ามาก) | อนุญาตให้ read/write พร้อมกันได้ |
| `ADD CONSTRAINT ... PRIMARY KEY / UNIQUE` (ไม่มี index อยู่แล้ว) | ต้องสร้าง index (คล้าย rewrite) | เท่าเวลาสร้าง index | ใช้ `CREATE UNIQUE INDEX CONCURRENTLY` แล้วค่อย `ADD CONSTRAINT ... UNIQUE USING INDEX` แทนได้ |

### ทำไม ADD CONSTRAINT CHECK/FK ปกติถึงช้า

การเพิ่ม `CHECK` หรือ `FOREIGN KEY` แบบปกติ (ไม่ใส่ `NOT VALID`) PostgreSQL ต้อง**สแกนข้อมูลทุกแถวที่มีอยู่**เพื่อยืนยันว่าไม่มีแถวไหนละเมิด constraint นั้น และตลอดการสแกนนี้ **ยังถือ `ACCESS EXCLUSIVE LOCK` อยู่** — หมายความว่าตารางถูกบล็อกทั้งหมดตลอดเวลาที่ใช้สแกน ถ้าตารางมี 100 ล้านแถว การสแกนอาจใช้เวลาหลายนาทีถึงหลายสิบนาที ระบบทั้งหมดที่ใช้ตารางนี้จะค้างตลอดเวลานั้น

```sql
-- อันตราย: บนตารางใหญ่ คำสั่งนี้ถือ ACCESS EXCLUSIVE ตลอดการสแกนทุกแถว
ALTER TABLE order_items
    ADD CONSTRAINT fk_order_items_prepared_by
    FOREIGN KEY (prepared_by) REFERENCES employees(employee_id);
```

วิธีที่ดีกว่า (จะเจาะลึกใน Step 189 และ Part 079): ใช้ `NOT VALID` ก่อน แล้วค่อย `VALIDATE CONSTRAINT` แยกต่างหาก เพราะ `VALIDATE CONSTRAINT` ใช้ lock ระดับ `SHARE UPDATE EXCLUSIVE` ซึ่งเบากว่ามาก — อนุญาตให้ `SELECT`, `INSERT`, `UPDATE`, `DELETE` ทำงานพร้อมกันได้ตามปกติระหว่างที่กำลังสแกนตรวจสอบอยู่ (บล็อกแค่ DDL อื่น ๆ ที่จะมาแก้ตารางพร้อมกัน)

```sql
-- ปลอดภัยกว่า: แยกเป็น 2 ขั้นตอน
ALTER TABLE order_items
    ADD CONSTRAINT fk_order_items_prepared_by
    FOREIGN KEY (prepared_by) REFERENCES employees(employee_id)
    NOT VALID;              -- ขั้นตอนนี้เร็วมาก (แค่ metadata, ไม่สแกน)

ALTER TABLE order_items
    VALIDATE CONSTRAINT fk_order_items_prepared_by;  -- ขั้นตอนนี้สแกน แต่ lock เบา
```

```
ALTER TABLE
ALTER TABLE
```

> **ข้อควรรู้:** `NOT VALID` หมายความว่า constraint นี้จะถูกบังคับใช้กับ**แถวใหม่ที่เขียนตั้งแต่ตอนนี้ทันที** (แถวใหม่ต้องผ่านเงื่อนไข) แต่**ยังไม่รับประกัน**ว่าแถวเก่าทั้งหมดผ่านเงื่อนไขหรือไม่ จนกว่าจะรัน `VALIDATE CONSTRAINT` สำเร็จ

### เทคนิคเพิ่มเติมสำหรับตารางใหญ่มาก ๆ

1. **`lock_timeout`** — ตั้งค่า timeout ให้คำสั่ง `ALTER TABLE` ยอมแพ้แทนที่จะรอ lock เป็นเวลานาน (ป้องกัน lock queue สะสม)

```sql
SET lock_timeout = '2s';
ALTER TABLE orders ADD COLUMN notes TEXT;
```

```
SET
ALTER TABLE
```

ถ้ามี lock ค้างจาก transaction อื่นเกิน 2 วินาที จะได้ error แทนที่จะรอไม่จำกัดเวลา:

```
ERROR:  canceling statement due to lock timeout
```

2. **รวมหลาย ALTER TABLE ไว้ในคำสั่งเดียว** — ลด lock acquisition รอบ

3. **หลีกเลี่ยง ALTER TABLE ในช่วง peak traffic** — ทำในช่วง maintenance window ถ้าเป็นไปได้

4. **ใช้ `CREATE INDEX CONCURRENTLY`** แทน index ที่มากับ `PRIMARY KEY`/`UNIQUE` บน production เพื่อไม่ให้ตารางถูกล็อกตอนสร้าง index

```sql
CREATE UNIQUE INDEX CONCURRENTLY idx_customers_email_uniq ON customers(email);

ALTER TABLE customers
    ADD CONSTRAINT customers_email_key UNIQUE USING INDEX idx_customers_email_uniq;
```

```
CREATE INDEX
ALTER TABLE
```

---

## Step 189: แนวทาง Zero-Downtime Schema Migration เบื้องต้น

ในระบบ production ที่มี traffic ตลอด 24 ชั่วโมง (เช่น แอประบบสั่งกาแฟที่ต้องทำงานได้ตลอดเวลา) การหยุดระบบเพื่อแก้ schema ไม่ใช่ทางเลือกที่ยอมรับได้ บทนี้จะแนะนำแนวคิดพื้นฐาน — รายละเอียดเชิงลึกและเครื่องมือเฉพาะทาง (เช่น `pg_repack`, online schema change tools, การจัดการกับ FK แบบ batch) จะอยู่ใน **Part 079**

### หลักการสำคัญ: แยกการเปลี่ยนแปลงเป็นขั้นตอนเล็ก ๆ ที่แต่ละขั้นเร็วและปลอดภัย

แพทเทิร์นคลาสสิกที่สุดคือ **"เพิ่มคอลัมน์แบบ nullable ก่อน → backfill ข้อมูล → ค่อยบังคับ NOT NULL ทีหลัง"** แทนที่จะพยายามทำทุกอย่างในคำสั่งเดียว

สมมติสถานการณ์: ร้านกาแฟต้องการเพิ่มคอลัมน์ `loyalty_tier` ให้ `customers` และต้องการบังคับว่าห้ามเป็น `NULL` (ทุกคนต้องมีระดับสมาชิก)

**วิธีที่ไม่ควรทำบนตารางใหญ่ (มีความเสี่ยง lock นาน):**

```sql
-- อันตรายถ้า customers มีข้อมูลหลายล้านแถว และ 'bronze' เป็น constant ก็จริง (เร็วในการ ADD)
-- แต่การรวม NOT NULL ตั้งแต่แรกโดยไม่ backfill เป็นขั้นตอนแยก จะทำให้ debug และ rollback ยากกว่า
ALTER TABLE customers ADD COLUMN loyalty_tier VARCHAR(20) NOT NULL DEFAULT 'bronze';
```

แม้คำสั่งข้างบนจะทำงานได้เร็วจริง (เพราะ PG11+ metadata-only) แต่ปัญหาจะเกิดขึ้นเมื่อธุรกิจต้องการ **backfill ค่าที่แตกต่างกันในแต่ละแถว** (เช่น ลูกค้าเก่าที่ซื้อเกิน 100 ครั้งควรได้ `'gold'` ทันที ไม่ใช่ `'bronze'` ทุกคน) การรวม default คงที่ตัวเดียวจึงไม่ตอบโจทย์ธุรกิจจริงเสมอไป

**แนวทางที่ปลอดภัยกว่า แบ่งเป็น 4 ขั้นตอน:**

**ขั้นตอนที่ 1 — เพิ่มคอลัมน์แบบ nullable ไม่มี NOT NULL (เร็ว, metadata-only)**

```sql
ALTER TABLE customers ADD COLUMN loyalty_tier VARCHAR(20);
```

```
ALTER TABLE
```

**ขั้นตอนที่ 2 — Backfill ข้อมูลเป็น batch เล็ก ๆ (ไม่ทำทีเดียวทั้งตาราง)**

การ `UPDATE` ทีเดียวทั้งตารางบนตารางใหญ่จะสร้าง transaction ยาวที่ทำให้เกิด table bloat และอาจ lock แถวจำนวนมากพร้อมกัน วิธีที่ดีกว่าคือแบ่งเป็น batch เล็ก ๆ

```sql
-- แนวคิด batch update (รันซ้ำเป็นรอบ ๆ จนกว่าจะครบทุกแถว)
UPDATE customers
SET loyalty_tier = CASE
        WHEN loyalty_points >= 1000 THEN 'gold'
        WHEN loyalty_points >= 300  THEN 'silver'
        ELSE 'bronze'
    END
WHERE customer_id IN (
    SELECT customer_id FROM customers
    WHERE loyalty_tier IS NULL
    LIMIT 5000                      -- ทำทีละ 5,000 แถว
);
```

```
UPDATE 2
```

(ในตัวอย่างนี้มีแค่ 2 แถว จึงจบใน 1 รอบ แต่ในตารางจริงที่มีล้านแถว ต้องเขียน loop ฝั่ง application หรือ script ที่รันคำสั่งนี้ซ้ำจนกว่า `UPDATE 0`)

**ขั้นตอนที่ 3 — เพิ่ม CHECK NOT VALID แล้ว VALIDATE (lock เบา)**

```sql
ALTER TABLE customers
    ADD CONSTRAINT chk_loyalty_tier_not_null CHECK (loyalty_tier IS NOT NULL) NOT VALID;

ALTER TABLE customers VALIDATE CONSTRAINT chk_loyalty_tier_not_null;
```

```
ALTER TABLE
ALTER TABLE
```

**ขั้นตอนที่ 4 — SET NOT NULL จริง (ข้าม scan เพราะมี valid CHECK อยู่แล้ว, PG12+)**

```sql
ALTER TABLE customers ALTER COLUMN loyalty_tier SET NOT NULL;

-- ลบ CHECK constraint ที่ใช้เป็นสะพานชั่วคราวออก เพราะ NOT NULL ทำหน้าที่แทนแล้ว
ALTER TABLE customers DROP CONSTRAINT chk_loyalty_tier_not_null;
```

```
ALTER TABLE
ALTER TABLE
```

ตรวจสอบผลลัพธ์สุดท้าย:

```sql
SELECT customer_id, full_name, loyalty_points, loyalty_tier FROM customers;
```

```
 customer_id |   full_name    | loyalty_points | loyalty_tier
-------------+----------------+----------------+---------------
           1 | สมชาย ใจดี     |              0 | bronze
           2 | สมหญิง รักไทย  |              0 | bronze
(2 rows)
```

### สรุปหลักการ Zero-Downtime Migration เบื้องต้น

| ขั้นตอน | สิ่งที่ทำ | ความเสี่ยง lock |
|---|---|---|
| 1. เพิ่มคอลัมน์ nullable | `ADD COLUMN` ไม่มี `NOT NULL` | ต่ำมาก |
| 2. Backfill เป็น batch | `UPDATE ... LIMIT n` วนซ้ำ | ต่ำ (แต่ละ batch สั้น) |
| 3. เพิ่ม CHECK NOT VALID + VALIDATE | 2 คำสั่งแยกกัน | ต่ำ (VALIDATE ใช้ lock เบา) |
| 4. SET NOT NULL | ข้าม scan เพราะมี CHECK validate แล้ว | ต่ำมาก |

> **สิ่งที่ต้องจำ:** เป้าหมายของ zero-downtime migration ไม่ใช่การทำให้ทุกอย่างเร็วขึ้น แต่คือการ**แบ่งงานหนักออกเป็นชิ้นเล็ก ๆ ที่แต่ละชิ้นถือ lock สั้นและเบาที่สุดเท่าที่จะทำได้** เพื่อไม่ให้ระบบ production หยุดชะงักแม้แต่วินาทีเดียวขณะที่กำลัง migrate อยู่ เนื้อหาเชิงลึกเรื่องนี้ — รวมถึงการจัดการ FK แบบ batch, การใช้เครื่องมือ online migration, และการ rename ตารางแบบไม่มี downtime ด้วยเทคนิค view-swap — จะอยู่ใน **Part 079: Zero-Downtime Migration**

---

## Step 190: แบบฝึกหัดรวม — วิวัฒนาการ Schema ของระบบร้านกาแฟ

มาลองประกอบทุกอย่างที่เรียนมาในบทนี้เข้าด้วยกัน โดยจำลองสถานการณ์ธุรกิจจริงที่ระบบร้านกาแฟต้องผ่านการ migrate schema หลายครั้งติดต่อกันตามความต้องการทางธุรกิจที่เปลี่ยนไป

### สถานการณ์: ร้านกาแฟขยายกิจการ

ร้านกาแฟกำลังจะเปิดสาขาใหม่ และทีมพัฒนาต้องปรับ schema ให้รองรับฟีเจอร์ใหม่ 5 อย่างตามลำดับ ให้เดินตามทุก migration ด้านล่างนี้ตามลำดับ

**Migration 1 — เพิ่มการรองรับหลายสาขา**

```sql
-- 1.1 สร้างตารางสาขาใหม่
CREATE TABLE branches (
    branch_id   SERIAL PRIMARY KEY,
    branch_name VARCHAR(100) NOT NULL,
    address     TEXT
);

INSERT INTO branches (branch_name, address) VALUES
    ('สาขาสยาม', 'สยามสแควร์ กรุงเทพฯ'),
    ('สาขาเชียงใหม่', 'นิมมานเหมินทร์ เชียงใหม่');

-- 1.2 เพิ่มคอลัมน์ branch_id ให้ orders แบบ nullable ก่อน (metadata-only, เร็ว)
ALTER TABLE orders ADD COLUMN branch_id INTEGER;

-- 1.3 backfill: ออเดอร์เก่าทั้งหมดถือว่าเป็นของสาขาสยาม (สาขาแรก)
UPDATE orders SET branch_id = 1 WHERE branch_id IS NULL;

-- 1.4 เพิ่ม FK แบบ NOT VALID ก่อน แล้ว validate
ALTER TABLE orders
    ADD CONSTRAINT fk_orders_branch FOREIGN KEY (branch_id) REFERENCES branches(branch_id)
    NOT VALID;
ALTER TABLE orders VALIDATE CONSTRAINT fk_orders_branch;

-- 1.5 บังคับ NOT NULL ผ่านสะพาน CHECK NOT VALID
ALTER TABLE orders ADD CONSTRAINT chk_branch_id_not_null CHECK (branch_id IS NOT NULL) NOT VALID;
ALTER TABLE orders VALIDATE CONSTRAINT chk_branch_id_not_null;
ALTER TABLE orders ALTER COLUMN branch_id SET NOT NULL;
ALTER TABLE orders DROP CONSTRAINT chk_branch_id_not_null;
```

```
CREATE TABLE
INSERT 0 2
ALTER TABLE
UPDATE 0
ALTER TABLE
ALTER TABLE
ALTER TABLE
ALTER TABLE
ALTER TABLE
ALTER TABLE
```

**Migration 2 — เปลี่ยนวิธีเก็บราคาให้รองรับส่วนลด**

```sql
-- 2.1 เพิ่มคอลัมน์ discount_percent
ALTER TABLE products ADD COLUMN discount_percent NUMERIC(5,2) NOT NULL DEFAULT 0
    CHECK (discount_percent BETWEEN 0 AND 100);

-- 2.2 เปลี่ยนความละเอียดของ price จาก NUMERIC(10,2) เป็น NUMERIC(12,2)
--     เพื่อรองรับราคาสินค้าพิเศษ (เช่น ชุดของขวัญ) ที่อาจสูงกว่าเดิม
ALTER TABLE products ALTER COLUMN price TYPE NUMERIC(12,2);
```

```
ALTER TABLE
ALTER TABLE
```

**Migration 3 — เปลี่ยนชื่อคอลัมน์ให้สื่อความหมายชัดเจนขึ้นตาม naming convention ใหม่**

```sql
-- ทีมตัดสินใจใช้ naming convention "is_" prefix สำหรับ boolean ทุกคอลัมน์
ALTER TABLE products RENAME COLUMN in_stock TO is_in_stock;
```

```
ALTER TABLE
```

**Migration 4 — ย้ายตารางสาขาเก่าที่ปิดกิจการไปเก็บใน archive schema**

```sql
CREATE SCHEMA IF NOT EXISTS archive;

CREATE TABLE closed_branches (
    branch_id   INTEGER PRIMARY KEY,
    branch_name VARCHAR(100),
    closed_date DATE DEFAULT CURRENT_DATE
);

ALTER TABLE closed_branches SET SCHEMA archive;
```

```
CREATE SCHEMA
CREATE TABLE
ALTER TABLE
```

**Migration 5 — ยกเลิกฟีเจอร์เก่าที่ไม่ใช้แล้ว (ลบคอลัมน์ที่ deprecated)**

```sql
-- ทีมตัดสินใจว่า hire_date_text (จากตัวอย่าง Step 183) เป็นข้อมูลซ้ำซ้อนกับ hire_date แล้ว ไม่ต้องใช้อีกต่อไป
ALTER TABLE employees DROP COLUMN IF EXISTS hire_date_text;
```

```
ALTER TABLE
```

### ตรวจสอบ schema สุดท้ายหลังทุก migration

```sql
\d orders
```

```
                                      Table "public.orders"
   Column    |           Type           | Collation | Nullable |            Default
-------------+---------------------------+-----------+----------+---------------------------------
 order_id    | integer                   |           | not null | nextval('orders_order_id_seq'..)
 customer_id | integer                   |           |          |
 employee_id | integer                   |           |          |
 order_date  | timestamp with time zone |           | not null | now()
 status      | order_status              |           | not null | 'pending'::order_status
 branch_id   | integer                   |           | not null |
Indexes:
    "orders_pkey" PRIMARY KEY, btree (order_id)
Foreign-key constraints:
    "fk_orders_branch" FOREIGN KEY (branch_id) REFERENCES branches(branch_id)
    "orders_customer_id_fkey" FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
    "orders_employee_id_fkey" FOREIGN KEY (employee_id) REFERENCES employees(employee_id)
```

```sql
\d products
```

```
                                    Table "public.products"
      Column      |     Type      | Collation | Nullable |               Default
-------------------+---------------+-----------+----------+---------------------------------------
 product_id        | integer       |           | not null | nextval('products_product_id_seq'..)
 product_name      | character varying(100) | |  not null |
 category_id       | integer       |           |          |
 price             | numeric(12,2) |           | not null |
 is_in_stock       | boolean       |           | not null | true
 discount_percent  | numeric(5,2)  |           | not null | 0
Check constraints:
    "products_discount_percent_check" CHECK (discount_percent >= 0 AND discount_percent <= 100)
    "products_price_check" CHECK (price >= 0::numeric)
```

จะเห็นได้ว่าตลอด 5 migration นี้ เราไม่เคยต้อง `DROP TABLE` หรือหยุดระบบเลยสักครั้ง — ทุกการเปลี่ยนแปลงทำผ่าน `ALTER TABLE` แบบค่อยเป็นค่อยไป ซึ่งเป็นวิธีที่ระบบ production จริงใช้กันเป็นมาตรฐาน

---

## สรุปท้ายบท

ในบทนี้เราครอบคลุมคำสั่ง `ALTER TABLE` ในทุกมิติที่จำเป็นสำหรับการดูแล schema ของระบบจริง:

- **`ADD COLUMN`** — เพิ่มคอลัมน์ได้แบบ metadata-only ถ้า default เป็นค่าคงที่ (PostgreSQL 11+) แต่ default แบบ volatile (เช่น `now()`, `random()`) ยังคงต้อง rewrite ตารางทั้งใบ
- **`DROP COLUMN`** — เร็วเสมอ (metadata-only) แต่ต้องระวัง view ที่พึ่งพาคอลัมน์นั้น (ต้อง `CASCADE` และเสี่ยงลบ view ทิ้งไปด้วย) ข้อมูลจริงไม่ถูกลบทันทีจาก disk
- **`ALTER COLUMN TYPE`** — บางกรณี (ขยาย varchar) ทำได้แบบไม่ต้อง `USING` และเร็ว แต่การแปลงข้ามชนิดข้อมูลจริง ๆ ต้องใช้ `USING` และมักต้อง rewrite ตารางทั้งใบ
- **`SET/DROP DEFAULT`, `SET/DROP NOT NULL`** — DEFAULT เปลี่ยนได้แบบ metadata-only เสมอ ส่วน `SET NOT NULL` ต้องสแกนตรวจสอบทั้งตาราง ยกเว้นมี valid CHECK constraint รองรับอยู่แล้ว (PG12+)
- **`RENAME COLUMN`, `RENAME TO`** — metadata-only เร็วมาก view ไม่พังเพราะอ้างอิงด้วย attnum แต่โค้ดแอปพลิเคชันที่ hardcode ชื่อคอลัมน์จะพัง
- **`SET SCHEMA`** — ย้ายตารางข้าม schema แบบ metadata-only ไม่ย้ายข้อมูลจริงบน disk
- **การเพิ่ม/ลบ constraint** — PK, FK, UNIQUE, CHECK ทำผ่าน `ADD/DROP CONSTRAINT`; ใช้ `NOT VALID` + `VALIDATE CONSTRAINT` เพื่อลด lock time บนตารางใหญ่
- **ACCESS EXCLUSIVE LOCK** — คือ lock ที่ `ALTER TABLE` ส่วนใหญ่ต้องใช้ ซึ่งบล็อกทุกการเข้าถึงตาราง ต้องแยกแยะให้ออกว่าคำสั่งไหน metadata-only (เร็ว) กับคำสั่งไหน rewrite ตาราง (ช้าตามขนาดตาราง)
- **Zero-downtime migration** — แพทเทิร์นพื้นฐานคือ เพิ่มคอลัมน์ nullable → backfill เป็น batch → เพิ่ม constraint แบบ NOT VALID แล้ว validate → ค่อยบังคับ NOT NULL ทีหลัง (รายละเอียดเชิงลึกอยู่ใน Part 079)

ทักษะเหล่านี้คือสิ่งที่แยกระหว่าง DBA/Backend Engineer มือใหม่กับมือโปร — มือใหม่รัน `ALTER TABLE` ตรงไปตรงมาแล้วต้องมาแก้ปัญหาระบบค้างทีหลัง ในขณะที่มือโปรวางแผนทุก migration ล่วงหน้าโดยคำนึงถึง lock, ขนาดตาราง, และ traffic ของระบบเสมอ

---

## แบบฝึกหัด

<details>
<summary><strong>แบบฝึกหัดที่ 1:</strong> เขียนคำสั่งเพิ่มคอลัมน์ <code>table_number</code> ชนิด <code>INTEGER</code> ให้ตาราง <code>orders</code> โดยไม่มีค่า default</summary>

```sql
ALTER TABLE orders ADD COLUMN table_number INTEGER;
```

คำสั่งนี้เป็น metadata-only ทำงานเร็วมากแม้ตารางจะมีข้อมูลจำนวนมาก เพราะไม่มี default ให้ต้อง backfill
</details>

<details>
<summary><strong>แบบฝึกหัดที่ 2:</strong> อธิบายว่าทำไมคำสั่ง <code>ALTER TABLE customers ADD COLUMN signup_source TEXT DEFAULT 'website'</code> บนตารางที่มี 50 ล้านแถวถึงทำงานเร็ว ทั้งที่ต้องกำหนดค่า default ให้ทุกแถว</summary>

เพราะ `'website'` เป็นค่าคงที่ (constant, non-volatile) ตั้งแต่ PostgreSQL 11 เป็นต้นมา PostgreSQL ไม่ได้เขียนค่า default ลงไปในทุกแถวจริง ๆ แต่เก็บค่า default นี้ไว้ใน system catalog (`pg_attribute`) แล้ว "ส่งคืน" ค่านี้แบบ on-the-fly เมื่อมีการอ่านแถวเก่าที่ยังไม่มีค่าจริงในคอลัมน์นี้ ทำให้เป็น metadata-only operation ไม่ว่าตารางจะมีกี่แถวก็ตาม ต่างจาก default ที่เป็นฟังก์ชัน volatile เช่น `now()` หรือ `random()` ซึ่งต้อง evaluate แยกต่อแถวและต้อง rewrite ตารางทั้งใบ
</details>

<details>
<summary><strong>แบบฝึกหัดที่ 3:</strong> มี view ชื่อ <code>product_summary</code> ที่ <code>SELECT product_name, price FROM products</code> ถ้าต้องการลบคอลัมน์ <code>price</code> ออกจากตาราง <code>products</code> จะเกิดอะไรขึ้น และต้องทำอย่างไร</summary>

ถ้ารัน `ALTER TABLE products DROP COLUMN price;` ตรง ๆ จะได้ error:

```
ERROR:  cannot drop column price of table products because other objects depend on it
DETAIL:  view product_summary depends on column price of table products
HINT:  Use DROP ... CASCADE to drop the dependent objects too.
```

ถ้ายืนยันจะลบ ต้องใช้ `CASCADE`:

```sql
ALTER TABLE products DROP COLUMN price CASCADE;
```

แต่คำสั่งนี้จะ**ลบ view `product_summary` ไปด้วย** ดังนั้นก่อนลบควรตรวจสอบ dependency ทั้งหมดก่อน และถ้ายังต้องการเก็บ view ไว้ ควรแก้ view ให้ไม่ใช้คอลัมน์ `price` ก่อน แล้วค่อยลบคอลัมน์แยกต่างหาก
</details>

<details>
<summary><strong>แบบฝึกหัดที่ 4:</strong> ตาราง <code>employees</code> มีคอลัมน์ <code>position</code> ชนิด <code>VARCHAR(50)</code> เก็บค่าเป็น text เช่น <code>'barista'</code>, <code>'cashier'</code>, <code>'manager'</code> ต้องการแปลงเป็น ENUM type ชื่อ <code>employee_position</code> จงเขียนคำสั่งทั้งหมด</summary>

```sql
-- 1. สร้าง ENUM type ก่อน
CREATE TYPE employee_position AS ENUM ('barista', 'cashier', 'manager');

-- 2. แปลงคอลัมน์โดยใช้ USING เพื่อ cast ค่าเดิม
ALTER TABLE employees
    ALTER COLUMN position TYPE employee_position USING position::employee_position;
```

ต้องใช้ `USING` เพราะ PostgreSQL ไม่รู้วิธีแปลง `VARCHAR` เป็น ENUM โดยอัตโนมัติ ต้องระบุการ cast ชัดเจน และถ้ามีค่าในตารางที่ไม่ตรงกับสมาชิกใน ENUM (เช่นมี `'supervisor'` ที่ไม่ได้ประกาศไว้) คำสั่งจะ error และ rollback ทั้งหมด
</details>

<details>
<summary><strong>แบบฝึกหัดที่ 5:</strong> อธิบายความแตกต่างระหว่าง <code>ACCESS EXCLUSIVE LOCK</code> ที่ใช้ตอน <code>ADD CONSTRAINT ... CHECK (...)</code> แบบปกติ กับ lock ที่ใช้ตอน <code>VALIDATE CONSTRAINT</code></summary>

`ADD CONSTRAINT ... CHECK (...)` แบบปกติ (ไม่มี `NOT VALID`) ต้องสแกนทั้งตารางเพื่อตรวจสอบว่าทุกแถวผ่านเงื่อนไข และ**ถือ `ACCESS EXCLUSIVE LOCK` ตลอดการสแกนนั้น** ซึ่งบล็อกการอ่าน/เขียนทุกชนิดบนตาราง

ในขณะที่ `VALIDATE CONSTRAINT` (ที่ใช้คู่กับ `ADD CONSTRAINT ... NOT VALID` ที่เพิ่มไว้ก่อนหน้า) ก็สแกนทั้งตารางเหมือนกัน แต่ใช้ lock ระดับ `SHARE UPDATE EXCLUSIVE` ซึ่งเบากว่ามาก อนุญาตให้ `SELECT`, `INSERT`, `UPDATE`, `DELETE` ทำงานพร้อมกันได้ตามปกติ (บล็อกแค่ DDL อื่นที่จะมาแก้ตารางพร้อมกัน) จึงเหมาะกับตารางใหญ่ในระบบ production มากกว่า
</details>

<details>
<summary><strong>แบบฝึกหัดที่ 6:</strong> ต้องการเปลี่ยนชื่อตาราง <code>categories</code> เป็น <code>product_categories</code> และเปลี่ยนชื่อคอลัมน์ <code>category_name</code> เป็น <code>name</code> ในคำสั่งเดียวกันได้หรือไม่ จงเขียนคำสั่งที่ถูกต้อง</summary>

ไม่สามารถทำในคำสั่งเดียวได้ เพราะ `RENAME TO` (เปลี่ยนชื่อตาราง) และ `RENAME COLUMN` (เปลี่ยนชื่อคอลัมน์) เป็นคนละ sub-command กัน และ PostgreSQL ไม่อนุญาตให้รวม `RENAME` มากกว่าหนึ่งอย่างในคำสั่ง `ALTER TABLE` เดียว (ต่างจาก `ADD COLUMN`/`DROP COLUMN` ที่รวมกันได้) ต้องแยกเป็น 2 คำสั่ง:

```sql
ALTER TABLE categories RENAME TO product_categories;
ALTER TABLE product_categories RENAME COLUMN category_name TO name;
```

ทั้งสองคำสั่งเป็น metadata-only เร็วมากทั้งคู่ จึงไม่มีปัญหาเรื่อง performance แม้จะแยกเป็น 2 คำสั่ง
</details>

<details>
<summary><strong>แบบฝึกหัดที่ 7:</strong> ตาราง <code>orders</code> มีข้อมูล 20 ล้านแถว ต้องการเพิ่มคอลัมน์ <code>priority_level</code> ชนิด <code>INTEGER NOT NULL</code> โดยที่ค่าเริ่มต้นต้องคำนวณจากข้อมูลอื่น (ไม่ใช่ค่าคงที่เดียวกันทุกแถว) จงออกแบบขั้นตอนการ migrate ที่ปลอดภัย</summary>

เนื่องจากค่าเริ่มต้นต้องคำนวณต่อแถว (ไม่ใช่ constant เดียว) จึงไม่สามารถใช้ประโยชน์จาก fast-default ของ PostgreSQL 11+ ได้ ต้องทำตามแพทเทิร์น zero-downtime migration:

```sql
-- ขั้นที่ 1: เพิ่มคอลัมน์แบบ nullable ก่อน (เร็ว, metadata-only)
ALTER TABLE orders ADD COLUMN priority_level INTEGER;

-- ขั้นที่ 2: backfill เป็น batch เล็ก ๆ (ตัวอย่างเงื่อนไขสมมติ)
UPDATE orders
SET priority_level = CASE WHEN status = 'pending' THEN 1 ELSE 2 END
WHERE order_id IN (
    SELECT order_id FROM orders WHERE priority_level IS NULL LIMIT 10000
);
-- รันซ้ำ (loop) จนกว่าจะได้ UPDATE 0

-- ขั้นที่ 3: เพิ่ม CHECK NOT VALID แล้ว validate (lock เบา)
ALTER TABLE orders ADD CONSTRAINT chk_priority_not_null CHECK (priority_level IS NOT NULL) NOT VALID;
ALTER TABLE orders VALIDATE CONSTRAINT chk_priority_not_null;

-- ขั้นที่ 4: SET NOT NULL จริง (ข้าม scan เพราะมี valid CHECK แล้ว, PG12+)
ALTER TABLE orders ALTER COLUMN priority_level SET NOT NULL;
ALTER TABLE orders DROP CONSTRAINT chk_priority_not_null;
```

การแบ่งเป็น 4 ขั้นตอนนี้ทำให้แต่ละคำสั่งถือ lock สั้นและเบาที่สุด ไม่กระทบผู้ใช้งานระบบขณะที่กำลัง migrate
</details>

<details>
<summary><strong>แบบฝึกหัดที่ 8:</strong> ต้องการย้ายตาราง <code>completed_orders_2024</code> จาก schema <code>archive</code> กลับมาที่ schema <code>public</code> จงเขียนคำสั่ง และอธิบายว่าคำสั่งนี้ย้ายข้อมูลจริงบน disk หรือไม่</summary>

```sql
ALTER TABLE archive.completed_orders_2024 SET SCHEMA public;
```

คำสั่งนี้**ไม่ย้ายข้อมูลจริงบน disk** เพราะ `SET SCHEMA` เป็น metadata-only operation ที่แค่เปลี่ยนค่า `relnamespace` ใน system catalog `pg_class` ให้ตารางชี้ไปยัง schema ใหม่ ไฟล์ข้อมูลจริงของตารางยังอยู่ที่เดิมใน tablespace เดียวกัน (schema เป็นแค่ namespace เชิง logical ไม่ใช่ physical location) จึงทำงานเร็วมากไม่ว่าตารางจะมีข้อมูลกี่แถวก็ตาม
</details>

<details>
<summary><strong>แบบฝึกหัดที่ 9:</strong> คำสั่งใดต่อไปนี้ "ไม่" ต้อง rewrite ตารางทั้งใบ: (ก) <code>ALTER COLUMN price TYPE NUMERIC(14,4)</code> จาก <code>NUMERIC(10,2)</code>, (ข) <code>ALTER COLUMN name TYPE VARCHAR(200)</code> จาก <code>VARCHAR(100)</code>, (ค) <code>ADD COLUMN created_at TIMESTAMPTZ DEFAULT now()</code></summary>

คำตอบคือ **(ข)** เท่านั้นที่ไม่ต้อง rewrite ตารางทั้งใบ

- **(ก)** การเปลี่ยน precision/scale ของ `NUMERIC` (เช่นจาก `NUMERIC(10,2)` เป็น `NUMERIC(14,4)`) โดยทั่วไปต้อง rewrite ตารางทั้งใบ เพราะการจัดเก็บค่าตัวเลขภายในอาจเปลี่ยนไป
- **(ข)** การขยายความยาว `VARCHAR` (จาก 100 เป็น 200 ตัวอักษร) เป็นการเพิ่ม constraint ที่หลวมขึ้น ไม่ต้อง rewrite ข้อมูลเดิม เป็น metadata-only (ทำงานเร็ว)
- **(ค)** `now()` เป็นฟังก์ชัน volatile ต้อง evaluate แยกต่อแถว จึงต้อง rewrite ตารางทั้งใบเพื่อกำหนดค่า timestamp ให้ทุกแถวที่มีอยู่แล้ว
</details>

<details>
<summary><strong>แบบฝึกหัดที่ 10:</strong> เขียน query เพื่อตรวจสอบว่าตาราง <code>orders</code> มี constraint อะไรอยู่บ้างในปัจจุบัน โดยแสดงชื่อ constraint ประเภท และคำนิยามแบบเต็ม</summary>

```sql
SELECT conname, contype, pg_get_constraintdef(oid) AS definition
FROM pg_constraint
WHERE conrelid = 'orders'::regclass
ORDER BY contype, conname;
```

ตัวอย่างผลลัพธ์:

```
        conname          | contype |                     definition
--------------------------+---------+------------------------------------------------------
 fk_orders_branch         | f       | FOREIGN KEY (branch_id) REFERENCES branches(branch_id)
 orders_customer_id_fkey  | f       | FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
 orders_employee_id_fkey  | f       | FOREIGN KEY (employee_id) REFERENCES employees(employee_id)
 orders_pkey              | p       | PRIMARY KEY (order_id)
```

อีกวิธีที่ใช้งานง่ายกว่าสำหรับดูแบบภาพรวมคือใช้ `\d orders` ใน `psql` ซึ่งจะแสดง constraint ทั้งหมดพร้อม index และความสัมพันธ์กับตารางอื่นในรูปแบบที่อ่านง่าย
</details>

---

**บทถัดไป:** [Part 020: Capstone Project — ระบบห้องสมุด](./part-020-capstone-library.md)
