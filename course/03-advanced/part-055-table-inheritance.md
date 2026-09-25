# Table Inheritance และเปรียบเทียบกับ Partitioning

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับสูง | Part 055

---

## เป้าหมายการเรียนรู้

เมื่อจบบทเรียนนี้ ผู้เรียนจะสามารถ:

1. อธิบายแนวคิด Table Inheritance ใน PostgreSQL และความสัมพันธ์กับแนวคิด object-oriented
2. สร้างตารางลูก (child table) ที่สืบทอดคอลัมน์จากตารางแม่ (parent table) ด้วย `INHERITS`
3. เข้าใจพฤติกรรมของการ query ผ่านตารางแม่ ทั้งแบบที่รวมข้อมูลจากตารางลูก และแบบที่ใช้ `ONLY` เพื่อดูเฉพาะตารางแม่
4. รู้ข้อจำกัดสำคัญของ Inheritance เรื่อง constraint ที่ไม่ถูกสืบทอดโดยอัตโนมัติ (PK, FK, UNIQUE)
5. เข้าใจ Multiple Inheritance และความซับซ้อนที่ตามมา
6. เข้าใจประวัติศาสตร์ว่า Inheritance เคยถูกใช้ทำ partitioning แบบ manual ก่อนที่ PostgreSQL 10 จะมี Declarative Partitioning
7. เปรียบเทียบ Table Inheritance กับ Declarative Partitioning อย่างละเอียด ทั้งเรื่อง constraint exclusion vs partition pruning และความซับซ้อนในการดูแลรักษา
8. ระบุ use case ที่ยังเหมาะกับ Inheritance ในปัจจุบัน เช่น polymorphic data model และ schema versioning
9. ตัดสินใจเลือกใช้ Inheritance, Partitioning หรือ Separate Tables ได้อย่างเหมาะสมกับสถานการณ์จริง

---

## เตรียมข้อมูล

ในบทนี้เราจะใช้ตัวอย่างระบบสินค้า (products) ที่มีสินค้าหลายประเภท ซึ่งแต่ละประเภทมีคุณสมบัติเฉพาะตัวที่ไม่เหมือนกัน เป็นตัวอย่างคลาสสิกที่เหมาะกับการอธิบาย Table Inheritance

```sql
-- ตารางแม่ (parent table) เก็บคุณสมบัติร่วมของสินค้าทุกประเภท
CREATE TABLE products_base (
    product_id SERIAL PRIMARY KEY,
    product_name VARCHAR(150) NOT NULL,
    unit_price NUMERIC(10,2) NOT NULL,
    created_at TIMESTAMPTZ DEFAULT now()
);

-- ตารางลูก: สินค้าอิเล็กทรอนิกส์ สืบทอดคอลัมน์จาก products_base
-- แล้วเพิ่มคอลัมน์เฉพาะของตัวเอง
CREATE TABLE electronics_products (
    warranty_months INTEGER,
    voltage VARCHAR(20)
) INHERITS (products_base);

-- ตารางลูก: สินค้าเสื้อผ้า สืบทอดคอลัมน์จาก products_base
-- แล้วเพิ่มคอลัมน์เฉพาะของตัวเอง
CREATE TABLE clothing_products (
    size VARCHAR(10),
    material VARCHAR(50)
) INHERITS (products_base);
```

ตรวจสอบโครงสร้างที่ได้:

```sql
\d products_base
\d electronics_products
```

ผลลัพธ์โดยประมาณของ `\d electronics_products`:

```
                                        Table "public.electronics_products"
     Column      |           Type           | Collation | Nullable |                    Default
------------------+--------------------------+-----------+----------+-----------------------------------------------
 product_id       | integer                  |           | not null | nextval('products_base_product_id_seq'::regclass)
 product_name     | character varying(150)   |           | not null |
 unit_price       | numeric(10,2)            |           | not null |
 created_at       | timestamp with time zone |           |          | now()
 warranty_months  | integer                  |           |          |
 voltage          | character varying(20)    |           |          |
Inherits: products_base
```

สังเกตว่า `electronics_products` มีคอลัมน์ทั้งหมดของ `products_base` บวกกับ `warranty_months` และ `voltage` ที่เพิ่มเข้ามาเอง และตาราง `products_base` เองก็ยังคง sequence `product_id` ไว้ให้ตารางลูกทุกตารางใช้ร่วมกัน (เพราะ `SERIAL` สร้าง sequence ไว้ที่ตารางแม่ และตารางลูกอ้างอิง default เดียวกัน)

เพิ่มข้อมูลตัวอย่าง:

```sql
INSERT INTO electronics_products (product_name, unit_price, warranty_months, voltage)
VALUES
    ('Smart TV 55 inch', 15990.00, 24, '220V'),
    ('Wireless Router', 1290.00, 12, '220V'),
    ('Bluetooth Speaker', 890.00, 6, '5V');

INSERT INTO clothing_products (product_name, unit_price, size, material)
VALUES
    ('Cotton T-Shirt', 259.00, 'L', 'Cotton'),
    ('Denim Jacket', 1590.00, 'M', 'Denim'),
    ('Running Shorts', 490.00, 'XL', 'Polyester');

-- และอาจมีสินค้าที่ไม่ระบุประเภทเฉพาะ ใส่ตรงตารางแม่ได้เลย
INSERT INTO products_base (product_name, unit_price)
VALUES ('Gift Voucher 500 THB', 500.00);
```

พร้อมแล้ว มาเริ่มเรียนรู้ Table Inheritance กันทีละ Step

---

## Step 541: Table Inheritance คืออะไร

Table Inheritance เป็นคุณสมบัติที่ PostgreSQL มีมาตั้งแต่เวอร์ชันแรก ๆ และถือเป็นหนึ่งใน "ของแปลก" ที่ทำให้ PostgreSQL แตกต่างจากฐานข้อมูลเชิงสัมพันธ์ (relational database) ทั่วไป เพราะ PostgreSQL เริ่มต้นจากโปรเจกต์ชื่อ **POSTGRES** ที่มหาวิทยาลัย Berkeley ในยุค 1980s ซึ่งมีเป้าหมายผสมผสานแนวคิด object-oriented เข้ากับฐานข้อมูลเชิงสัมพันธ์ (จึงมีคำว่า "object-relational database" ติดตัว PostgreSQL มาตลอด)

แนวคิดหลักของ Table Inheritance คล้ายกับการสืบทอด class ในภาษาโปรแกรมเชิงวัตถุ (OOP):

- ตารางแม่ (parent table) นิยาม attribute พื้นฐานที่ทุก subtype มีร่วมกัน
- ตารางลูก (child table) "สืบทอด" (`INHERITS`) คอลัมน์ทั้งหมดจากตารางแม่ แล้วสามารถเพิ่มคอลัมน์เฉพาะของตัวเองได้
- Query ที่ทำกับตารางแม่จะ "มองเห็น" แถวของตารางลูกทั้งหมดโดยอัตโนมัติ (เหมือนกับ polymorphism ใน OOP ที่เรียก method ผ่าน base class แล้วได้ผลลัพธ์ตาม subclass จริง)

Syntax พื้นฐาน:

```sql
CREATE TABLE child_table (
    -- คอลัมน์เพิ่มเติมเฉพาะของตารางลูก
) INHERITS (parent_table);
```

ข้อควรรู้เชิงแนวคิดที่สำคัญ:

| แนวคิด OOP | สิ่งที่ตรงกันใน PostgreSQL Inheritance |
|---|---|
| Base class | Parent table |
| Subclass | Child table |
| Inherited fields | คอลัมน์ที่สืบทอดมาจาก parent |
| Polymorphic collection | Query จาก parent table เห็นแถวจากทุก child |
| Method override | **ไม่มี** — Inheritance ใน PostgreSQL เป็นเรื่องโครงสร้างข้อมูล (data) ไม่ใช่ behavior |

สิ่งสำคัญที่ต้องเข้าใจตั้งแต่ต้น: Table Inheritance ใน PostgreSQL เป็นกลไกระดับ **storage และ query planning** ไม่ใช่ full object-relational mapping ตารางลูกแต่ละตารางเป็นตารางจริงที่มี physical storage (heap) ของตัวเอง ไม่ได้ผสมข้อมูลไว้ในตารางเดียวกันแบบที่บาง ORM ทำ (single table inheritance pattern)

ตรวจสอบความสัมพันธ์ parent-child ผ่าน catalog:

```sql
SELECT
    c.relname AS child_table,
    p.relname AS parent_table
FROM pg_inherits i
JOIN pg_class c ON c.oid = i.inhrelid
JOIN pg_class p ON p.oid = i.inhparent
ORDER BY parent_table, child_table;
```

ผลลัพธ์:

```
    child_table      |   parent_table
----------------------+------------------
 electronics_products | products_base
 clothing_products    | products_base
(2 rows)
```

`pg_inherits` คือ catalog table ที่เก็บความสัมพันธ์การสืบทอดทั้งหมดในฐานข้อมูล และเป็น catalog เดียวกับที่ Declarative Partitioning ใช้เก็บความสัมพันธ์ partition ด้วย (เพราะ partitioning สร้างบนกลไก inheritance เดิมนั่นเอง เดี๋ยวจะอธิบายละเอียดใน Step 546-547)

---

## Step 542: สร้างตารางลูกที่สืบทอดคอลัมน์ พร้อมเพิ่มคอลัมน์เฉพาะตัว

จาก "เตรียมข้อมูล" เราสร้างตารางลูกไปแล้วสองตาราง มาดูรายละเอียดเพิ่มเติมของการสืบทอดคอลัมน์ และวิธีเพิ่ม constraint พิเศษให้ตารางลูกแต่ละตัว

```sql
-- ตารางลูกสามารถมี CHECK constraint เฉพาะของตัวเองได้
CREATE TABLE furniture_products (
    weight_kg NUMERIC(6,2),
    assembly_required BOOLEAN DEFAULT true,
    CHECK (weight_kg > 0)
) INHERITS (products_base);

INSERT INTO furniture_products (product_name, unit_price, weight_kg, assembly_required)
VALUES ('Office Desk', 3990.00, 25.5, true);

-- ลองใส่ค่าที่ผิด CHECK constraint
INSERT INTO furniture_products (product_name, unit_price, weight_kg)
VALUES ('Broken Chair', 500.00, -5);
```

ผลลัพธ์:

```
ERROR:  new row for relation "furniture_products" violates check constraint "furniture_products_weight_kg_check"
DETAIL:  Failing row contains (4, Broken Chair, 500.00, ..., -5, true).
```

การเพิ่มคอลัมน์ที่ตารางแม่ภายหลัง จะกระจายไปยังตารางลูกทุกตารางโดยอัตโนมัติ (ซึ่งต่างจาก MySQL ที่ไม่มีแนวคิดนี้เลย):

```sql
ALTER TABLE products_base ADD COLUMN is_active BOOLEAN DEFAULT true;

-- ตารางลูกทุกตารางจะมีคอลัมน์นี้ด้วยทันที
\d clothing_products
```

ผลลัพธ์ (บางส่วน):

```
                                    Table "public.clothing_products"
    Column     |          Type          | Collation | Nullable |      Default
----------------+------------------------+-----------+----------+--------------------
 product_id     | integer                |           | not null | nextval(...)
 product_name   | character varying(150) |           | not null |
 unit_price     | numeric(10,2)          |           | not null |
 created_at     | timestamp with time zone |         |          | now()
 is_active      | boolean                |           |          | true
 size           | character varying(10)  |           |          |
 material       | character varying(50)  |           |          |
Inherits: products_base
```

ในทางกลับกัน การ `ALTER TABLE` ที่ตารางลูกโดยตรง (เช่นเพิ่มคอลัมน์เฉพาะที่ไม่ต้องการให้กระทบตารางแม่) ก็ทำได้ตามปกติ เพราะตารางลูกคือตารางอิสระที่มี schema เป็นของตัวเอง เพียงแต่ "รวม" คอลัมน์จากตารางแม่เข้ามาด้วย

ข้อควรระวัง: การลบคอลัมน์ที่ตารางแม่ (`ALTER TABLE products_base DROP COLUMN ...`) จะลบคอลัมน์นั้นออกจากตารางลูกทุกตารางเช่นกัน และถ้าตารางลูกมีการอ้างอิงคอลัมน์นั้นในลักษณะพิเศษ (เช่นเป็นส่วนหนึ่งของ local constraint) อาจทำให้เกิด error ได้

---

## Step 543: Query จากตารางแม่เห็นข้อมูลทุกตารางลูกโดยอัตโนมัติ (และ ONLY keyword)

นี่คือพฤติกรรมที่สำคัญที่สุดของ Table Inheritance: เมื่อ query จากตารางแม่แบบปกติ PostgreSQL จะรวมข้อมูลจากตารางลูกทุกตารางเข้ามาด้วยโดยอัตโนมัติ

```sql
-- Query ตารางแม่แบบปกติ: เห็นข้อมูลจากตัวเองและตารางลูกทั้งหมด
SELECT product_id, product_name, unit_price, tableoid::regclass AS source_table
FROM products_base
ORDER BY product_id;
```

ผลลัพธ์:

```
 product_id |     product_name      | unit_price |    source_table
-------------+----------------------+------------+----------------------
           1 | Gift Voucher 500 THB |     500.00 | products_base
           2 | Smart TV 55 inch     |   15990.00 | electronics_products
           3 | Wireless Router      |    1290.00 | electronics_products
           4 | Bluetooth Speaker    |     890.00 | electronics_products
           5 | Cotton T-Shirt       |     259.00 | clothing_products
           6 | Denim Jacket         |    1590.00 | clothing_products
           7 | Running Shorts       |     490.00 | clothing_products
           8 | Office Desk          |    3990.00 | furniture_products
(8 rows)
```

คอลัมน์พิเศษ `tableoid` (มีในทุกตาราง) บอกว่าแถวนั้นมาจากตารางจริง ๆ ตารางไหน ซึ่งมีประโยชน์มากเวลา debug query ที่ใช้ inheritance เพราะไม่เช่นนั้นเราจะไม่รู้ว่าแถวที่เห็นมาจากตารางลูกตัวไหน

ถ้าต้องการดู **เฉพาะ** ข้อมูลของตารางแม่จริง ๆ โดยไม่รวมตารางลูก ให้ใช้ keyword `ONLY`:

```sql
SELECT product_id, product_name, unit_price
FROM ONLY products_base
ORDER BY product_id;
```

ผลลัพธ์:

```
 product_id |     product_name      | unit_price
-------------+----------------------+------------
           1 | Gift Voucher 500 THB |     500.00
(1 row)
```

เห็นเฉพาะแถวที่ insert ตรงเข้าไปที่ `products_base` เท่านั้น (Gift Voucher) ไม่รวมแถวจากตารางลูกทั้งสาม

`ONLY` ใช้ได้กับคำสั่งอื่นด้วย ไม่ใช่แค่ `SELECT`:

```sql
-- UPDATE เฉพาะแถวในตารางแม่ ไม่กระทบตารางลูก
UPDATE ONLY products_base SET is_active = false WHERE product_id = 1;

-- DELETE เฉพาะแถวในตารางแม่
DELETE FROM ONLY products_base WHERE product_id = 1;
```

และยังใช้ wildcard `*` ต่อท้ายชื่อตารางเพื่อบอกอย่างชัดเจนว่าต้องการรวมตารางลูกด้วย (ซึ่งเป็นพฤติกรรม default อยู่แล้ว การเขียน `*` จึงเป็นแค่การทำให้ query อ่านง่ายขึ้นเท่านั้น):

```sql
-- เทียบเท่ากับ SELECT ... FROM products_base (ไม่มี *)
SELECT count(*) FROM products_base*;
```

สรุปพฤติกรรม:

| รูปแบบ query | ผลลัพธ์ |
|---|---|
| `SELECT * FROM products_base` | รวมข้อมูลจากตารางแม่ + ตารางลูกทั้งหมด (default) |
| `SELECT * FROM products_base*` | เหมือนด้านบน เขียนชัดเจนขึ้นเฉย ๆ |
| `SELECT * FROM ONLY products_base` | เฉพาะข้อมูลในตารางแม่เท่านั้น ไม่รวมตารางลูก |

การ query ผ่าน `psql` ด้วย `\d+ products_base` จะโชว์ท้ายตารางว่ามี "Child tables" อะไรบ้าง ซึ่งช่วยให้เห็นภาพรวมได้เร็ว:

```sql
\d+ products_base
```

```
...
Child tables: clothing_products,
              electronics_products,
              furniture_products
```

---

## Step 544: ข้อจำกัดสำคัญ — PK/FK/UNIQUE ไม่ถูกสืบทอดอัตโนมัติ

นี่คือข้อจำกัดที่สร้างความประหลาดใจให้ผู้เริ่มต้นใช้ Inheritance มากที่สุด และเป็นสาเหตุสำคัญที่ทำให้ Inheritance ไม่เหมาะกับหลาย use case

ลองดู constraint ของตารางแม่และตารางลูก:

```sql
SELECT conrelid::regclass AS table_name, conname, contype, pg_get_constraintdef(oid) AS definition
FROM pg_constraint
WHERE conrelid IN ('products_base'::regclass, 'electronics_products'::regclass)
ORDER BY table_name;
```

ผลลัพธ์:

```
   table_name    |            conname             | contype |          definition
-------------------+--------------------------------+---------+--------------------------------
 products_base     | products_base_pkey             | p       | PRIMARY KEY (product_id)
 furniture_products | furniture_products_weight_kg_check | c   | CHECK (weight_kg > 0)
(2 rows)
```

สังเกตว่า `electronics_products` **ไม่มี** primary key เลย! แม้ว่ามันจะ "สืบทอด" คอลัมน์ `product_id` มาจาก `products_base` ก็ตาม นี่คือความจริงที่สำคัญ:

> **PostgreSQL Inheritance สืบทอดเฉพาะ column definitions (ชื่อคอลัมน์, ชนิดข้อมูล, NOT NULL, CHECK บางกรณี) แต่ไม่สืบทอด PRIMARY KEY, UNIQUE constraint, และ FOREIGN KEY**

ทดสอบให้เห็นจริง:

```sql
-- product_id ซ้ำกันได้ในตารางลูกคนละตาราง เพราะไม่มี PK ผูกร่วมกัน
-- (แม้ sequence จะทำให้ปกติไม่ซ้ำ แต่ไม่มีอะไรบังคับ)
INSERT INTO electronics_products (product_id, product_name, unit_price)
VALUES (5, 'Duplicate ID Gadget', 999.00);

SELECT product_id, product_name, tableoid::regclass
FROM products_base
WHERE product_id = 5;
```

ผลลัพธ์:

```
 product_id |     product_name      |     tableoid
-------------+-----------------------+----------------------
           5 | Cotton T-Shirt        | clothing_products
           5 | Duplicate ID Gadget   | electronics_products
(2 rows)
```

เกิด `product_id = 5` ซ้ำกันสองแถวคนละตาราง เพราะ **primary key ของ `products_base` ไม่ครอบคลุมถึงตารางลูก** — PK เป็น local constraint ของตารางแม่เท่านั้น

วิธีแก้: ต้องประกาศ constraint แยกในแต่ละตารางลูกเอง

```sql
-- เพิ่ม UNIQUE constraint แยกที่ตารางลูกแต่ละตัว
ALTER TABLE electronics_products ADD PRIMARY KEY (product_id);
ALTER TABLE clothing_products ADD PRIMARY KEY (product_id);
ALTER TABLE furniture_products ADD PRIMARY KEY (product_id);
```

แต่ถึงจะทำแบบนี้ ก็ยังมีปัญหาเชิงตรรกะเหลืออยู่: **PK ของแต่ละตารางลูกเป็นอิสระจากกัน** ไม่มีอะไรบังคับว่า `product_id` จะไม่ซ้ำกัน **ข้าม** ตารางลูก (คือ electronics_products กับ clothing_products ยังมี product_id ซ้ำกันได้อยู่ดี เพียงแต่ภายในตัวเองแต่ละตารางจะไม่ซ้ำ) เพราะ PostgreSQL ไม่มีกลไก "global unique constraint across inheritance tree"

Foreign Key ก็มีปัญหาเดียวกัน:

```sql
CREATE TABLE product_reviews (
    review_id SERIAL PRIMARY KEY,
    product_id INTEGER REFERENCES products_base(product_id),  -- อ้างอิงตารางแม่เท่านั้น
    rating INTEGER CHECK (rating BETWEEN 1 AND 5)
);

-- FK นี้ตรวจสอบกับ ONLY products_base จริง ๆ หรือรวมตารางลูกด้วย?
INSERT INTO product_reviews (product_id, rating) VALUES (2, 5);  -- product_id 2 อยู่ใน electronics_products
```

ผลลัพธ์:

```
INSERT 0 1
```

จริง ๆ แล้ว FK ที่อ้างอิงตารางแม่ **จะตรวจสอบรวมข้อมูลจากตารางลูกด้วย** (เพราะ FK ใช้กลไก query แบบเดียวกับ `SELECT` ปกติที่รวม child tables) แต่ในทางกลับกัน **ไม่มีทางสร้าง FK ที่ชี้ตรงไปยังตารางลูกตัวใดตัวหนึ่งแล้วครอบคลุม PK ข้ามตารางลูกได้** เพราะแต่ละตารางลูกมี PK อิสระของตัวเอง และ PostgreSQL ไม่รองรับ FK ที่อ้างอิง "รวมทุกตารางในลำดับชั้น inheritance" ได้โดยตรง

สรุปข้อจำกัดสำคัญ:

| Constraint | สืบทอดจาก parent หรือไม่ | หมายเหตุ |
|---|---|---|
| Column definition (name, type) | ใช่ สืบทอด | รวมถึง default ด้วย |
| `NOT NULL` | ใช่ สืบทอด | |
| `CHECK` | ใช่ สืบทอด (ตั้งแต่ PG ยุคหลัง) | ยกเว้นระบุ `NO INHERIT` |
| `PRIMARY KEY` | **ไม่** | ต้องประกาศแยกในแต่ละ child |
| `UNIQUE` | **ไม่** | ต้องประกาศแยกในแต่ละ child |
| `FOREIGN KEY` (เป็น referencing table) | **ไม่** | ต้องประกาศแยกในแต่ละ child |
| `FOREIGN KEY` (เป็น referenced table) | ใช้ได้ผ่าน parent แต่ตรวจสอบรวม child (ผลข้างเคียง ไม่ใช่การออกแบบเพื่อสิ่งนี้) | มีพฤติกรรมกำกวมที่ควรระวัง |

นี่คือเหตุผลหลักที่ Inheritance ไม่เหมาะกับการสร้างระบบที่ต้องพึ่งพา referential integrity อย่างเข้มงวด

---

## Step 545: Multiple Inheritance

PostgreSQL รองรับ **Multiple Inheritance** คือตารางลูกสามารถสืบทอดจากตารางแม่ได้มากกว่าหนึ่งตารางพร้อมกัน คล้ายกับ multiple inheritance ในภาษา C++ (ต่างจาก Java ที่ class สืบทอดได้ตัวเดียว)

```sql
-- ตารางแม่อีกตัวหนึ่ง: เก็บข้อมูลเกี่ยวกับการจัดส่ง
CREATE TABLE shippable_items (
    weight_kg NUMERIC(6,2),
    requires_signature BOOLEAN DEFAULT false
);

-- สินค้าอิเล็กทรอนิกส์ระดับพรีเมียม ที่ต้องมีทั้งคุณสมบัติสินค้า และคุณสมบัติการจัดส่ง
CREATE TABLE premium_electronics (
    extended_warranty_years INTEGER
) INHERITS (electronics_products, shippable_items);
```

ตรวจสอบโครงสร้าง:

```sql
\d premium_electronics
```

ผลลัพธ์:

```
                          Table "public.premium_electronics"
          Column          |          Type          | Collation | Nullable | Default
---------------------------+------------------------+-----------+----------+---------
 product_id                | integer                |           | not null |
 product_name              | character varying(150) |           | not null |
 unit_price                | numeric(10,2)          |           | not null |
 created_at                | timestamp with time zone|           |          | now()
 is_active                 | boolean                |           |          | true
 warranty_months           | integer                |           |          |
 voltage                   | character varying(20)  |           |          |
 weight_kg                 | numeric(6,2)           |           |          |
 requires_signature        | boolean                |           |          | false
 extended_warranty_years   | integer                |           |          |
Inherits: electronics_products, shippable_items
```

`premium_electronics` มีคอลัมน์รวมกันจากทั้งสองสาย: `electronics_products` (ซึ่งสืบทอดมาจาก `products_base` อีกที) และ `shippable_items` — ทำให้เกิดโครงสร้างแบบ **diamond-shaped inheritance** ได้ (คือ `premium_electronics` ก็ยังนับเป็นลูกของ `products_base` ทางอ้อมผ่าน `electronics_products`)

ตรวจสอบว่าอยู่ในสาย inheritance ของ `products_base` จริงหรือไม่:

```sql
SELECT count(*) FROM products_base;  -- ควรรวม premium_electronics ด้วย เพราะสืบทอดผ่าน electronics_products
```

ปัญหาและความซับซ้อนของ Multiple Inheritance:

1. **Column name conflict** — ถ้าตารางแม่สองตัวมีคอลัมน์ชื่อเดียวกันแต่ type ต่างกัน (เช่น ทั้งคู่มีคอลัมน์ `weight_kg` แต่ type ไม่ตรงกัน) จะเกิด error ทันทีตอนสร้างตาราง

```sql
CREATE TABLE conflict_example (
    id INTEGER
) INHERITS (electronics_products, furniture_products);
-- furniture_products ไม่มี weight_kg ชนกับใคร ตัวอย่างนี้จะผ่าน
-- แต่ถ้าสองตารางแม่มีคอลัมน์ชื่อเดียวกันคนละ type จะได้:
-- ERROR: column "xxx" has a type conflict
```

2. **Constraint ซ้ำซ้อนหรือขัดแย้ง** — ถ้าตารางแม่หลายตัวมี CHECK constraint ชื่อเดียวกันแต่เงื่อนไขต่างกัน ต้องจัดการ merge เอง
3. **การดูแลรักษายากขึ้นทวีคูณ** — ยิ่งมีสายการสืบทอดซับซ้อน ยิ่งยากต่อการเข้าใจว่าตารางหนึ่ง ๆ มีคอลัมน์อะไรบ้างมาจากไหน (ต้องไล่ดู `\d+` และ `pg_inherits` ทุกครั้ง)
4. **Order-dependent behavior บางกรณี** — ลำดับที่ระบุตารางแม่ใน `INHERITS (a, b)` มีผลต่อการ resolve default value และลำดับ column บางกรณี

โดยทั่วไป Multiple Inheritance ในทางปฏิบัติจริงถูกใช้น้อยมาก เพราะความซับซ้อนที่เพิ่มขึ้นมักไม่คุ้มกับประโยชน์ที่ได้ ส่วนใหญ่แนะนำให้ใช้ **composition** (สร้างตารางแยกแล้ว JOIN) แทนการทำ multiple inheritance

---

## Step 546: ประวัติศาสตร์ — Inheritance เคยถูกใช้ทำ Partitioning แบบ Manual

ก่อนที่ PostgreSQL 10 (ปล่อยปี 2017) จะแนะนำ **Declarative Partitioning** (`PARTITION BY RANGE/LIST/HASH` ที่เราเรียนไปใน Part 052-054) นักพัฒนา PostgreSQL ใช้ Table Inheritance เป็นกลไกหลักในการทำ **table partitioning แบบ manual** มาตั้งแต่ PostgreSQL 8.1 (ปี 2005)

แนวคิดของ "Manual Partitioning ผ่าน Inheritance" มีดังนี้:

```sql
-- ตัวอย่างแนวคิดเดิม (ก่อน PG 10) — ไม่ใช่ syntax ที่แนะนำให้ใช้ในปัจจุบัน
CREATE TABLE orders_manual (
    order_id BIGSERIAL,
    order_date DATE NOT NULL,
    customer_id INTEGER,
    total_amount NUMERIC(12,2)
);

-- สร้างตารางลูกสำหรับแต่ละช่วงเวลา พร้อม CHECK constraint กำกับขอบเขต
CREATE TABLE orders_2024 (
    CHECK (order_date >= '2024-01-01' AND order_date < '2025-01-01')
) INHERITS (orders_manual);

CREATE TABLE orders_2025 (
    CHECK (order_date >= '2025-01-01' AND order_date < '2026-01-01')
) INHERITS (orders_manual);

-- ต้องเขียน trigger เองเพื่อ route ข้อมูลไปตารางลูกที่ถูกต้อง
CREATE OR REPLACE FUNCTION orders_manual_insert_trigger()
RETURNS TRIGGER AS $$
BEGIN
    IF NEW.order_date >= '2024-01-01' AND NEW.order_date < '2025-01-01' THEN
        INSERT INTO orders_2024 VALUES (NEW.*);
    ELSIF NEW.order_date >= '2025-01-01' AND NEW.order_date < '2026-01-01' THEN
        INSERT INTO orders_2025 VALUES (NEW.*);
    ELSE
        RAISE EXCEPTION 'Date out of range, add a new partition';
    END IF;
    RETURN NULL;  -- ป้องกันไม่ให้ insert ลงตารางแม่จริง ๆ
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER insert_orders_trigger
    BEFORE INSERT ON orders_manual
    FOR EACH ROW EXECUTE FUNCTION orders_manual_insert_trigger();
```

สิ่งที่ทำให้แนวทางนี้ "พอใช้งานได้" ในฐานะ partitioning คือฟีเจอร์ที่เรียกว่า **Constraint Exclusion** — planner จะอ่าน `CHECK constraint` ของตารางลูกแต่ละตัว แล้วตัดตารางลูกที่ไม่เกี่ยวข้องออกจาก query plan โดยอัตโนมัติ ถ้า query มีเงื่อนไข `WHERE` ที่ระบุช่วงตรงกับ CHECK constraint:

```sql
SET constraint_exclusion = partition;  -- ต้องเปิดเอง (ค่า default คือ partition อยู่แล้วใน PG สมัยใหม่)

EXPLAIN SELECT * FROM orders_manual WHERE order_date >= '2025-06-01' AND order_date < '2025-07-01';
```

Planner จะข้ามการสแกน `orders_2024` เพราะรู้จาก CHECK constraint ว่าไม่มีทางมีแถวที่ตรงเงื่อนไข

ปัญหาของแนวทางนี้ (ที่เป็นแรงผลักดันให้เกิด Declarative Partitioning ใน PG10):

1. ต้องเขียน trigger function เองทุกครั้ง (INSERT routing) — เสี่ยง bug และมี overhead ของการยิง trigger ทุกแถว
2. การเพิ่ม partition ใหม่ต้องแก้ trigger function ด้วยมือ (เพิ่ม `ELSIF` branch)
3. Constraint Exclusion อาศัยการวิเคราะห์ CHECK constraint ที่ query planner ต้อง evaluate ทุกตารางลูก ซึ่งช้ากว่า partition pruning ของ Declarative Partitioning มาก (โดยเฉพาะเมื่อมี partition จำนวนมาก)
4. ไม่มี `UPDATE`/`DELETE` routing อัตโนมัติ — ถ้า `order_date` ของแถวถูก update จนข้ามช่วง partition แถวนั้นจะ "ค้าง" อยู่ผิดตารางลูก (ไม่มีกลไกย้ายอัตโนมัติ)
5. ไม่มี concept ของ "default partition" หรือการ validate ว่าช่วงของ partition ไม่ทับซ้อนกัน (ต้องระวังเอง)

Manual Inheritance-based Partitioning ยังพบได้ในระบบเก่าที่สร้างมาก่อนปี 2017 และยังไม่ได้ migrate มาใช้ Declarative Partitioning แต่สำหรับระบบใหม่ **ไม่แนะนำให้ใช้แนวทางนี้อีกต่อไป** ให้ใช้ Declarative Partitioning ตามที่เรียนใน Part 052-054 แทนเสมอ

---

## Step 547: เปรียบเทียบ Table Inheritance กับ Declarative Partitioning อย่างละเอียด

Declarative Partitioning (ที่เราเรียนไปใน Part 052-054 ด้วยตาราง `orders_partitioned`) **ถูกสร้างขึ้นบนกลไก catalog เดียวกันกับ Table Inheritance** (คือใช้ `pg_inherits` เก็บความสัมพันธ์ parent-partition เหมือนกัน) แต่ PostgreSQL เพิ่ม metadata และ optimization พิเศษเข้าไปอีกชั้น ทำให้พฤติกรรมต่างจาก manual inheritance อย่างมาก

ทบทวนตัวอย่าง `orders_partitioned` จาก Part 054 สั้น ๆ:

```sql
-- (ทบทวนจาก Part 054 — โครงสร้างเดิม ไม่ต้องสร้างใหม่ในบทนี้)
-- CREATE TABLE orders_partitioned (
--     order_id BIGSERIAL,
--     order_date DATE NOT NULL,
--     ...
-- ) PARTITION BY RANGE (order_date);
--
-- CREATE TABLE orders_partitioned_2024 PARTITION OF orders_partitioned
--     FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');
```

ตารางเปรียบเทียบอย่างละเอียด:

| ประเด็น | Table Inheritance (manual) | Declarative Partitioning |
|---|---|---|
| Syntax สร้างตารางลูก | `CREATE TABLE child (...) INHERITS (parent)` | `CREATE TABLE child PARTITION OF parent FOR VALUES ...` |
| การกำหนดขอบเขตข้อมูล | CHECK constraint ที่เขียนเอง | `FOR VALUES FROM/TO`, `IN`, `WITH (MODULUS, REMAINDER)` เป็น metadata ของระบบ |
| การ route ข้อมูลตอน INSERT | ต้องเขียน trigger เอง | อัตโนมัติ — planner รู้ว่า insert ไปตาราง partition ไหนจาก partition key |
| การตัดตารางที่ไม่เกี่ยวข้องออกจาก plan | **Constraint Exclusion** — ประเมิน CHECK constraint ทีละตารางตอน planning (runtime cost สูงขึ้นตามจำนวนตาราง) | **Partition Pruning** — ใช้ partition metadata โดยตรง มีทั้งแบบ planning-time และ **runtime pruning** (สำหรับ parameterized query/prepared statement) |
| ประสิทธิภาพเมื่อมีตารางลูกจำนวนมาก (หลักร้อย-พัน) | ช้าลงชัดเจน (evaluate CHECK ทุกตัว) | Pruning algorithm แบบ binary search ทำงานเร็วกว่ามาก |
| PRIMARY KEY / UNIQUE ครอบคลุมทุก partition | ทำไม่ได้ | รองรับ (ตั้งแต่ PG ใหม่ ๆ) ผ่าน constraint ที่รวม partition key เช่น `PRIMARY KEY (order_id, order_date)` |
| FOREIGN KEY อ้างอิง partitioned table | มีปัญหา/ไม่รองรับเต็มรูปแบบ | รองรับ (ตั้งแต่ PG 12+ อ้างอิงมาที่ partitioned table ได้ตรง ๆ) |
| UPDATE ที่ทำให้แถวย้ายข้าม partition | ไม่มีกลไกอัตโนมัติ (แถวค้างผิดตาราง) | ย้ายอัตโนมัติ (ตั้งแต่ PG 11+) |
| `ATTACH`/`DETACH` partition โดยไม่ล็อกตารางนาน | ไม่มี concept นี้ | รองรับ `ATTACH PARTITION` / `DETACH PARTITION CONCURRENTLY` |
| Default partition (รับข้อมูลที่ไม่ตรง partition ไหนเลย) | ไม่มี concept นี้ (ต้องดักด้วย trigger เอง) | รองรับ `CREATE TABLE ... PARTITION OF ... DEFAULT` |
| `EXPLAIN` แสดงผลลัพธ์ | แสดงตารางที่ scan ตามผล constraint exclusion เท่านั้น (ไม่ได้บอกชัดว่า "pruned") | แสดง "Subplans Removed" ชัดเจนเมื่อ pruning ทำงาน |
| Indexing ข้ามตารางลูก | สร้าง index แยกทีละตารางเอง ไม่มี global index concept | รองรับ index ระดับ partitioned table เอง (สร้างครั้งเดียว กระจายไปทุก partition อัตโนมัติ) |
| ความยืดหยุ่นของ schema แต่ละตารางลูก | สูงมาก — ตารางลูกแต่ละตัวมีคอลัมน์ต่างกันได้ (นี่คือจุดเด่นจริงของ inheritance) | ต่ำ — partition ทุกตัวต้องมี schema เดียวกันกับ parent (เพิ่มได้แค่ index/constraint เฉพาะบางอย่าง) |
| การใช้งานหลักในปัจจุบัน | Polymorphic data model, ไม่ใช่ partitioning | Partitioning ทุกกรณีสำหรับระบบใหม่ |

ตัวอย่างเปรียบเทียบ `EXPLAIN` ให้เห็นภาพความต่าง:

```sql
-- Manual inheritance + constraint exclusion
EXPLAIN SELECT * FROM orders_manual WHERE order_date = '2025-03-15';
```

```
Append  (cost=0.00..45.50 rows=10 width=48)
  ->  Seq Scan on orders_2025 orders_manual_1
        Filter: (order_date = '2025-03-15'::date)
  (orders_2024 ถูกตัดออกเพราะ CHECK constraint ไม่ match — ผลจาก constraint_exclusion)
```

```sql
-- Declarative partitioning
EXPLAIN SELECT * FROM orders_partitioned WHERE order_date = '2025-03-15';
```

```
Append  (cost=0.00..25.30 rows=8 width=48)
  ->  Index Scan using orders_partitioned_2025_order_date_idx on orders_partitioned_2025
        Index Cond: (order_date = '2025-03-15'::date)
(1 row)
```

สังเกตว่า Declarative Partitioning ใน `EXPLAIN` จะไม่แสดง partition ที่ถูกตัดออกเลย (pruning เกิดตั้งแต่ planning time อย่างชัดเจนและมีประสิทธิภาพ) ในขณะที่ constraint exclusion ต้อง evaluate constraint ของทุกตารางในสายก่อนตัดสินใจ ซึ่งมี cost เพิ่มขึ้นตามจำนวนตารางลูก

---

## Step 548: Use Case ที่ยังเหมาะกับ Inheritance ในปัจจุบัน (ไม่ใช่ Partitioning)

แม้ Inheritance จะไม่เหมาะกับ partitioning อีกต่อไปแล้ว แต่ยังมี use case บางแบบที่ตัวมันเองมีคุณค่าเฉพาะตัวที่ Declarative Partitioning ทำแทนไม่ได้ เพราะจุดต่างสำคัญคือ **Inheritance อนุญาตให้ตารางลูกมี schema ต่างกันได้** ในขณะที่ partition ทุกตัวต้องมี schema เหมือน parent ทุกประการ

### Use case 1: Polymorphic Data Model

เมื่อ entity มีหลาย subtype ที่มีคุณสมบัติเฉพาะตัวต่างกันชัดเจน (เหมือนตัวอย่าง `products_base` ในบทนี้) และต้องการ query แบบ "รวมทุก subtype" ได้บ่อย ๆ พร้อมกับ query แบบ "เฉพาะ subtype นี้" ได้เจาะจง

```sql
-- Query สินค้าทั้งหมดแบบ polymorphic ไม่ว่าจะเป็น subtype ไหน
SELECT product_id, product_name, unit_price FROM products_base WHERE unit_price > 1000;

-- Query เฉพาะ subtype เพื่อดูคุณสมบัติเฉพาะตัว
SELECT product_name, warranty_months, voltage FROM electronics_products WHERE warranty_months >= 12;
```

ตัวอย่างโลกจริงที่นิยมใช้แนวทางนี้: ระบบ CMS ที่มี "content items" หลายชนิด (article, video, podcast) ที่มี metadata ต่างกัน แต่ต้องการ list รวมกันในหน้า "content ล่าสุด" ได้

### Use case 2: Schema Versioning / Event Sourcing แบบมี evolving schema

ระบบ log เหตุการณ์ที่ schema เปลี่ยนแปลงไปตามเวลา (เช่น event version 1 มี field ชุดหนึ่ง, version 2 เพิ่ม field ใหม่) สามารถใช้ inheritance:

```sql
CREATE TABLE events_base (
    event_id BIGSERIAL PRIMARY KEY,
    event_type VARCHAR(50) NOT NULL,
    occurred_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE events_v1 (
    payload_v1 JSONB
) INHERITS (events_base);

CREATE TABLE events_v2 (
    payload_v2 JSONB,
    trace_id UUID
) INHERITS (events_base);
```

Query รวม event ทุก version ได้จากตารางแม่ ในขณะที่แต่ละ version เก็บ field เฉพาะของตัวเองแยกกันชัดเจน โดยไม่ต้องมี NULL column จำนวนมากปนกันในตารางเดียว (แบบที่จะเกิดถ้าใช้ single table with all-version columns)

### Use case 3: การจำลอง "abstract base table" สำหรับ tooling หรือ testing เฉพาะทาง

บาง tool หรือ extension ใช้ inheritance เพื่อสร้างโครงสร้างที่ query ร่วมกันได้ง่าย โดยไม่สนใจเรื่อง partitioning เลย เช่น การสร้างตารางแม่ที่ไม่มีข้อมูลจริง (empty parent) ทำหน้าที่เป็นแค่ "interface" ให้ query รวมได้:

```sql
-- ตารางแม่ว่างเปล่า ไม่มีการ insert ตรง ๆ เลย ทำหน้าที่เป็น "abstract" table
CREATE TABLE audit_log_base (
    log_id BIGSERIAL,
    logged_at TIMESTAMPTZ DEFAULT now()
) ;
-- ไม่ต้องมี PK เพราะไม่มีข้อมูลจริงในตารางนี้
```

หลักการเลือกใช้ที่สำคัญ: **ถ้าเกณฑ์การแบ่งข้อมูลคือ "เวลา" หรือ "ช่วงค่า" (range/list/hash) และ schema ของทุกส่วนเหมือนกันหมด → ใช้ Declarative Partitioning เสมอ** แต่ถ้าเกณฑ์การแบ่งคือ "ประเภทของข้อมูลที่มี attribute ต่างกันจริง ๆ" (แต่ยังต้องการ query รวมกันได้บางครั้ง) → Inheritance ยังเป็นตัวเลือกที่สมเหตุสมผล

---

## Step 549: ข้อจำกัดและปัญหาที่พบบ่อยของ Inheritance

สรุปรวมปัญหาที่ทำให้ Inheritance ไม่แนะนำสำหรับ use case ใหม่ส่วนใหญ่ในปัจจุบัน:

### 1. ไม่มี Global Uniqueness / Referential Integrity ที่แท้จริง

ตามที่แสดงใน Step 544 — PK/UNIQUE/FK ไม่ครอบคลุมข้าม hierarchy การรับประกันความถูกต้องของข้อมูล (data integrity) ต้องพึ่ง application logic เพิ่มเติม ซึ่งเสี่ยงต่อ bug

### 2. Query Performance ไม่ scale ดีเมื่อจำนวนตารางลูกมาก

ทุก query ที่ query ผ่านตารางแม่ (ไม่ใช้ `ONLY`) ต้องพิจารณาตารางลูกทุกตัว แม้จะมี constraint exclusion ช่วยตัด แต่ก็ยังมี planning overhead ที่สูงกว่า partition pruning มาก โดยเฉพาะเมื่อมีตารางลูกหลักร้อยตัวขึ้นไป

### 3. ไม่มี `ATTACH`/`DETACH` ที่ปลอดภัยแบบ concurrent

Declarative Partitioning มี `ALTER TABLE ... DETACH PARTITION ... CONCURRENTLY` ที่ไม่ล็อกตารางนาน แต่ Inheritance ไม่มี concept นี้ การ "ปลด" ตารางลูกออกจากสาย inheritance ต้องใช้ `ALTER TABLE child NO INHERIT parent` ซึ่งอาจต้องพิจารณา lock เพิ่มเติม และไม่มี tooling ระดับเดียวกัน

```sql
ALTER TABLE furniture_products NO INHERIT products_base;
```

### 4. `INSERT ... RETURNING`, `COPY`, และคำสั่งบางตัวมีพฤติกรรมสับสน

การ `INSERT` ลงตารางแม่โดยตรง (ไม่ผ่าน trigger routing) จะเข้าตารางแม่เท่านั้น ไม่กระจายไปตารางลูกอัตโนมัติ ต่างจาก Declarative Partitioning ที่ `INSERT` เข้า parent จะ route ไป partition ที่ถูกต้องเสมอ:

```sql
-- Insert เข้า parent โดยตรง ไม่มี trigger คอย route
INSERT INTO products_base (product_name, unit_price) VALUES ('Uncategorized Item', 99.00);
-- แถวนี้จะอยู่ที่ products_base เท่านั้น ไม่ได้ถูกจัดเข้า electronics/clothing/furniture ใด ๆ
-- (ต่างจาก partitioned table ที่ insert เข้า parent จะ error ถ้าไม่มี partition ไหนรองรับ หรือ route เข้า default partition)
```

### 5. Schema Drift ที่ตรวจสอบยาก

เพราะตารางลูกแต่ละตัวมี schema อิสระ เมื่อเวลาผ่านไปทีมพัฒนาต่างคนต่างแก้ตารางลูกของตัวเอง อาจเกิด schema drift ที่ทำให้ query ผ่าน parent มีพฤติกรรมไม่คาดคิด (เช่น บาง child มีคอลัมน์ที่ไม่มีใน child อื่น ทำให้ query แบบ `SELECT specific_column FROM products_base` error เพราะ column นั้นไม่ได้อยู่ใน parent แต่อยู่ใน child บางตัวเท่านั้น)

### 6. ขาดการสนับสนุนจาก ORM และ tooling สมัยใหม่

ORM ส่วนใหญ่ (Django, SQLAlchemy, Prisma, TypeORM ฯลฯ) มี native support สำหรับ Declarative Partitioning ในระดับหนึ่ง แต่แทบไม่มี ORM ไหนรองรับ PostgreSQL table inheritance โดยตรง ทำให้การใช้งานร่วมกับ framework สมัยใหม่ยุ่งยากกว่า

### 7. Backup/Restore และ Replication ซับซ้อนขึ้น

เครื่องมือบางตัว (logical replication บางกรณี, บาง third-party backup tool) มีการจัดการ partitioned table ที่ดีกว่า inheritance tree ทั่วไป เพราะ Declarative Partitioning เป็น "first-class citizen" ในระบบ catalog ของ PostgreSQL สมัยใหม่ ในขณะที่ inheritance เป็นกลไกทั่วไปที่ไม่มี metadata พิเศษบอกว่า "นี่คือ partitioning scheme"

**ข้อสรุป**: สำหรับ use case ใหม่เกือบทั้งหมดที่เคยคิดจะใช้ Inheritance เพื่อแบ่งข้อมูลตามช่วงเวลาหรือช่วงค่า ให้ใช้ **Declarative Partitioning** แทนเสมอ ส่วน Inheritance ให้เก็บไว้ใช้เฉพาะกรณี polymorphic data model ที่ schema ของแต่ละ subtype ต่างกันจริง ๆ เท่านั้น

---

## สรุปท้ายบท

### ตารางเปรียบเทียบ Inheritance vs Partitioning vs Separate Tables

| มิติ | Table Inheritance | Declarative Partitioning | Separate Tables (ไม่เชื่อมกันเลย) |
|---|---|---|---|
| เหมาะกับ | Polymorphic data ที่แต่ละ subtype มี attribute ต่างกัน | ข้อมูลจำนวนมากที่แบ่งตาม range/list/hash แต่ schema เหมือนกันทุก partition | Entity ที่ไม่มีความสัมพันธ์เชิงโครงสร้างร่วมกันเลย หรือความสัมพันธ์อ่อนมาก |
| Schema ของแต่ละส่วนย่อย | ต่างกันได้ (จุดเด่น) | ต้องเหมือนกับ parent ทุกประการ | อิสระอย่างสมบูรณ์ |
| Query รวมทุกส่วนย่อยได้ไหม | ได้ (ผ่าน parent, default behavior) | ได้ (partition pruning ช่วยให้เร็ว) | ไม่ได้ (ต้องเขียน UNION เอง) |
| Global PK/UNIQUE/FK | ไม่รองรับ | รองรับ (ต้องรวม partition key ใน PK) | รองรับเต็มรูปแบบต่อตาราง (แต่ไม่ครอบคลุมข้ามตาราง) |
| Performance เมื่อจำนวนส่วนย่อยมาก | ลดลงตามจำนวน (constraint exclusion) | Scale ดี (partition pruning ระดับ catalog) | Scale ดีที่สุด (query แยกอิสระ) |
| Insert routing อัตโนมัติ | ไม่มี (ต้องเขียน trigger เอง) | มี (อัตโนมัติตาม partition key) | ไม่มี concept นี้ |
| Attach/Detach concurrent | ไม่มี | มี | ไม่เกี่ยวข้อง |
| ความซับซ้อนในการดูแลระยะยาว | สูง (โดยเฉพาะเมื่อมี child จำนวนมาก) | ปานกลาง (มี tooling รองรับดี) | ต่ำ (แต่ต้องเขียน UNION/application logic เพื่อรวมข้อมูลเอง) |
| ตัวอย่างในบทเรียนนี้ | `products_base` → `electronics_products`, `clothing_products`, `furniture_products` | `orders_partitioned` (Part 052-054) | ตารางอิสระที่ไม่มี relationship เชิงโครงสร้าง |

### หลักการตัดสินใจอย่างย่อ

1. **ข้อมูลจำนวนมากที่ต้องแบ่งตามเวลา/ช่วงค่า/hash เพื่อ performance และ maintenance** (เช่น log, order, event ปริมาณมหาศาล) → **Declarative Partitioning** เสมอ
2. **Entity ที่มีหลาย subtype ที่มี attribute เฉพาะตัวต่างกันจริง ๆ และต้องการ query รวมกันบ่อย ๆ** → **Table Inheritance** เป็นตัวเลือกที่สมเหตุสมผล (แต่ต้องยอมรับข้อจำกัดเรื่อง constraint)
3. **Entity ที่ไม่มีความสัมพันธ์เชิงโครงสร้างร่วมกันชัดเจน หรือแค่ต้องการแยกเก็บเพื่อความสะดวก** → **Separate Tables** พร้อม foreign key เชื่อมกันตามปกติ มักจะปลอดภัยและดูแลง่ายที่สุดในระยะยาว
4. เมื่อไม่แน่ใจ ให้เอียงไปทาง **Separate Tables + Foreign Key** ก่อนเสมอ เพราะมันคือแนวทางที่เครื่องมือ, ORM, และ PostgreSQL เองรองรับดีที่สุด ส่วน Inheritance ควรใช้ก็ต่อเมื่อมี use case ที่ชัดเจนจริง ๆ ว่า polymorphic query เป็นความต้องการหลัก

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1

จากตาราง `products_base`, `electronics_products`, `clothing_products` ที่สร้างไว้ในบทนี้ เขียน query เพื่อนับจำนวนสินค้าทั้งหมด (รวมทุกตารางลูก) ที่มีราคาต่ำกว่า 1000 บาท

<details>
<summary>เฉลย</summary>

```sql
SELECT count(*) FROM products_base WHERE unit_price < 1000;
```

เพราะ query ผ่าน `products_base` แบบไม่ใส่ `ONLY` จะรวมข้อมูลจากตารางลูกทุกตารางโดยอัตโนมัติ (electronics_products, clothing_products, furniture_products) ตามพฤติกรรม default ของ Table Inheritance

</details>

---

### แบบฝึกหัดที่ 2

เขียน query ที่แสดง**เฉพาะ**แถวที่ insert ตรงเข้าไปที่ `products_base` เท่านั้น ไม่รวมข้อมูลจากตารางลูกใด ๆ

<details>
<summary>เฉลย</summary>

```sql
SELECT * FROM ONLY products_base;
```

keyword `ONLY` ทำให้ query จำกัดขอบเขตไว้เฉพาะตารางที่ระบุ ไม่รวมตารางลูกในสาย inheritance

</details>

---

### แบบฝึกหัดที่ 3

อธิบายว่าทำไม `product_id` ในตัวอย่างของบทนี้จึงสามารถซ้ำกันได้ระหว่าง `electronics_products` และ `clothing_products` แม้ว่า `products_base` จะมี `PRIMARY KEY (product_id)` ก็ตาม

<details>
<summary>เฉลย</summary>

เพราะ **PRIMARY KEY ไม่ถูกสืบทอด (inherit) ไปยังตารางลูกโดยอัตโนมัติ** PK ที่ประกาศไว้ที่ `products_base` เป็น local constraint ที่บังคับใช้เฉพาะแถวที่อยู่จริงในตาราง `products_base` เท่านั้น ไม่ได้ครอบคลุมแถวในตารางลูก ดังนั้นแต่ละตารางลูกจึงต้องประกาศ PK ของตัวเองแยกต่างหาก (เช่น `ALTER TABLE electronics_products ADD PRIMARY KEY (product_id)`) และถึงจะทำแบบนั้น ก็ยังไม่มีกลไกป้องกัน `product_id` ซ้ำกัน**ข้าม**ตารางลูกอยู่ดี เพราะ PostgreSQL ไม่รองรับ unique constraint ที่ครอบคลุมทั้ง hierarchy ของ inheritance

</details>

---

### แบบฝึกหัดที่ 4

สร้างตารางลูกใหม่ชื่อ `books_products` ที่สืบทอดจาก `products_base` โดยมีคอลัมน์เพิ่มเติมคือ `author VARCHAR(100)` และ `isbn VARCHAR(20)` พร้อมกับเพิ่ม UNIQUE constraint ให้ `isbn` ไม่ซ้ำกันภายในตารางนี้

<details>
<summary>เฉลย</summary>

```sql
CREATE TABLE books_products (
    author VARCHAR(100),
    isbn VARCHAR(20),
    UNIQUE (isbn)
) INHERITS (products_base);
```

เนื่องจาก UNIQUE constraint ไม่ถูกสืบทอดจาก parent จึงต้องประกาศไว้ตรงในนิยามของตารางลูกเองเสมอ

</details>

---

### แบบฝึกหัดที่ 5

อธิบายความแตกต่างระหว่าง **Constraint Exclusion** กับ **Partition Pruning** โดยยกตัวอย่างว่าแบบไหนเหมาะกับระบบที่มีตารางย่อยจำนวนหลักพันตาราง

<details>
<summary>เฉลย</summary>

- **Constraint Exclusion** เป็นกลไกที่ query planner ใช้กับ Table Inheritance (แบบ manual partitioning) โดยจะประเมิน (evaluate) CHECK constraint ของตารางลูก**ทุกตาราง**เทียบกับเงื่อนไขใน `WHERE` clause ก่อนตัดสินใจว่าตารางไหนต้อง scan บ้าง ค่าใช้จ่ายด้าน planning time จึงเพิ่มขึ้นตามจำนวนตารางลูกแบบเป็นสัดส่วน (linear หรือแย่กว่า)
- **Partition Pruning** เป็นกลไกเฉพาะของ Declarative Partitioning ที่ใช้ partition metadata ในระบบ catalog โดยตรง (ไม่ต้อง evaluate constraint แบบทั่วไป) และมี algorithm ที่มีประสิทธิภาพกว่ามาก (คล้าย binary search บน partition bounds) รองรับทั้ง planning-time pruning และ runtime pruning (สำหรับ parameterized queries)

สำหรับระบบที่มีตารางย่อยจำนวนหลักพันตาราง **Declarative Partitioning พร้อม Partition Pruning เหมาะสมกว่ามาก** เพราะ constraint exclusion จะทำให้ planning time ช้าลงอย่างเห็นได้ชัดเมื่อจำนวนตารางลูกมากขนาดนั้น

</details>

---

### แบบฝึกหัดที่ 6

ทีมพัฒนาต้องการเก็บข้อมูล "การแจ้งเตือน" (notifications) ของผู้ใช้ ซึ่งมีหลายประเภท เช่น email notification (มี field `email_address`, `subject`), push notification (มี field `device_token`, `badge_count`), sms notification (มี field `phone_number`) โดยต้องการหน้า dashboard ที่แสดงการแจ้งเตือนทั้งหมดรวมกันเรียงตามเวลา ควรเลือกใช้ Inheritance, Partitioning หรือ Separate Tables? เพราะเหตุใด

<details>
<summary>เฉลย</summary>

กรณีนี้เหมาะกับ **Table Inheritance** เพราะ:

1. แต่ละประเภท notification มี attribute เฉพาะตัวที่แตกต่างกันชัดเจน (ไม่ใช่แค่แบ่งตามช่วงเวลาหรือช่วงค่า)
2. ต้องการ query รวมทุกประเภทเพื่อแสดงบน dashboard เรียงตามเวลา ซึ่งตรงกับพฤติกรรม default ของ inheritance ที่ query ผ่าน parent จะเห็นข้อมูลจากทุก child โดยอัตโนมัติ
3. ไม่ใช่ partitioning เพราะเกณฑ์การแบ่งไม่ใช่ range/list/hash ของค่าเดียวกัน แต่เป็นความต่างเชิง schema

```sql
CREATE TABLE notifications_base (
    notification_id BIGSERIAL PRIMARY KEY,
    user_id INTEGER NOT NULL,
    created_at TIMESTAMPTZ DEFAULT now(),
    is_read BOOLEAN DEFAULT false
);

CREATE TABLE email_notifications (
    email_address VARCHAR(255),
    subject VARCHAR(255)
) INHERITS (notifications_base);

CREATE TABLE push_notifications (
    device_token VARCHAR(255),
    badge_count INTEGER
) INHERITS (notifications_base);

CREATE TABLE sms_notifications (
    phone_number VARCHAR(20)
) INHERITS (notifications_base);
```

หมายเหตุ: ต้องเพิ่ม PK/index แยกในแต่ละตารางลูกด้วยตนเอง และถ้าปริมาณข้อมูลใหญ่มากในอนาคต อาจพิจารณา partition ตัวตารางลูกแต่ละตัวอีกชั้นตามเวลา (ผสมสองแนวทางเข้าด้วยกัน)

</details>

---

### แบบฝึกหัดที่ 7

จากตาราง `product_reviews` ในบทนี้ที่มี FK ชี้ไปที่ `products_base(product_id)` ทดลองอธิบายว่าทำไม `INSERT INTO product_reviews (product_id, rating) VALUES (2, 5)` จึงสำเร็จ ทั้งที่ `product_id = 2` ไม่ได้อยู่ในตาราง `products_base` โดยตรง แต่อยู่ใน `electronics_products`

<details>
<summary>เฉลย</summary>

เพราะการตรวจสอบ FOREIGN KEY constraint ใช้กลไก query แบบเดียวกับ `SELECT` ปกติที่ query ผ่าน parent table โดยไม่ใส่ `ONLY` — ดังนั้นเมื่อ PostgreSQL ตรวจสอบว่า `product_id = 2` มีอยู่ใน `products_base` หรือไม่ มันจะรวมข้อมูลจากตารางลูกทั้งหมดด้วย (ซึ่งรวมถึง `electronics_products` ที่มี `product_id = 2` อยู่) จึงพบว่าค่าตรงตามเงื่อนไข FK และ INSERT สำเร็จ

พฤติกรรมนี้เป็น "ผลข้างเคียง" ของกลไก inheritance มากกว่าจะเป็นการออกแบบมาเพื่อรองรับ FK ข้ามลำดับชั้นโดยเฉพาะ และอาจสร้างความสับสนได้ถ้าไม่เข้าใจกลไกเบื้องหลัง

</details>

---

### แบบฝึกหัดที่ 8

อธิบายว่าทำไมการทำ `ALTER TABLE products_base DROP COLUMN created_at;` ถึงส่งผลกระทบต่อตารางลูกทุกตารางด้วย และมีอะไรที่ต้องระวังก่อนทำ

<details>
<summary>เฉลย</summary>

เพราะคอลัมน์ `created_at` ในตารางลูกทุกตัว**สืบทอดมาจาก parent** ไม่ใช่คอลัมน์อิสระ ดังนั้นเมื่อลบคอลัมน์นี้ที่ parent ระบบจะลบคอลัมน์เดียวกันออกจากตารางลูกทุกตารางโดยอัตโนมัติด้วย (เพื่อรักษาความสอดคล้องของ schema ในสาย inheritance)

สิ่งที่ต้องระวังก่อนทำ:
1. ตรวจสอบว่าตารางลูกใดมีการใช้คอลัมน์นี้ใน local constraint, index, หรือ trigger หรือไม่ (อาจเกิด error หรือพฤติกรรมไม่คาดคิดถ้ามี dependency)
2. ตรวจสอบว่า application หรือ view ใดอ้างอิงคอลัมน์นี้ผ่านตารางลูกโดยตรงหรือไม่
3. พิจารณาทำ backup หรือทดสอบใน environment ที่ไม่ใช่ production ก่อนเสมอ เพราะการ DROP COLUMN เป็นปฏิบัติการที่ไม่สามารถ rollback ข้อมูลกลับมาได้ง่าย (ข้อมูลในคอลัมน์นั้นหายถาวร)

</details>

---

### แบบฝึกหัดที่ 9

ให้เปรียบเทียบสถานการณ์ต่อไปนี้ แล้วระบุว่าควรใช้ **Inheritance**, **Partitioning** หรือ **Separate Tables** — พร้อมเหตุผลสั้น ๆ:

(ก) ตาราง sensor_readings ที่มีข้อมูลนับพันล้านแถวต่อปี เก็บค่า timestamp, sensor_id, value เหมือนกันทุกแถว ต้องการ query ตามช่วงเวลาเป็นหลัก

(ข) ตาราง `payment_methods` ที่มี credit_card (เก็บ card_number_masked, expiry), bank_transfer (เก็บ bank_name, account_number), e_wallet (เก็บ wallet_provider, wallet_id) โดยระบบ checkout ต้องแสดงประวัติการชำระเงินทั้งหมดรวมกันเรียงเวลา

(ค) ตาราง `users` กับตาราง `products` ในระบบ e-commerce

<details>
<summary>เฉลย</summary>

**(ก) Declarative Partitioning** — ข้อมูลปริมาณมหาศาล, schema เหมือนกันทุกแถว, เกณฑ์แบ่งคือช่วงเวลา (timestamp) ตรงตามจุดแข็งของ Partitioning ทุกประการ ควรใช้ `PARTITION BY RANGE (timestamp)` และตั้ง retention policy ให้ DROP partition เก่าได้ง่าย

**(ข) Table Inheritance** — แต่ละประเภทการชำระเงินมี attribute เฉพาะตัวต่างกันชัดเจน (ไม่ใช่แค่ range/list ของค่าเดียวกัน) และต้องการ query รวมกันเพื่อแสดงประวัติ ตรงกับจุดแข็งของ Inheritance พอดี (ต้องระวังเพิ่ม PK/index แยกในแต่ละตารางลูกด้วย)

**(ค) Separate Tables** — `users` และ `products` เป็น entity คนละประเภทที่ไม่มีความสัมพันธ์เชิงโครงสร้างแบบ parent-child เลย (ไม่ใช่ subtype ของกันและกัน) ควรเป็นตารางอิสระที่เชื่อมกันผ่าน foreign key ตามปกติ (เช่นผ่านตาราง `orders` ที่อ้างอิงทั้งสองฝั่ง)

</details>

---

### แบบฝึกหัดที่ 10

เขียนคำสั่ง SQL เพื่อตรวจสอบว่าตารางใดบ้างในฐานข้อมูลปัจจุบันที่เป็นส่วนหนึ่งของ inheritance hierarchy (ทั้งฝั่ง parent และ child) โดยใช้ catalog table `pg_inherits`

<details>
<summary>เฉลย</summary>

```sql
SELECT DISTINCT relname, 'parent' AS role
FROM pg_inherits i
JOIN pg_class c ON c.oid = i.inhparent
UNION
SELECT DISTINCT relname, 'child' AS role
FROM pg_inherits i
JOIN pg_class c ON c.oid = i.inhrelid
ORDER BY relname;
```

หรือดูแบบสรุปความสัมพันธ์เต็มรูปแบบ:

```sql
SELECT
    p.relname AS parent_table,
    c.relname AS child_table,
    CASE WHEN p.relkind = 'p' THEN 'Declarative Partitioning' ELSE 'Table Inheritance' END AS mechanism
FROM pg_inherits i
JOIN pg_class c ON c.oid = i.inhrelid
JOIN pg_class p ON p.oid = i.inhparent
ORDER BY parent_table, child_table;
```

คอลัมน์ `relkind` ของ `pg_class` จะเป็น `'p'` (partitioned table) สำหรับ parent ที่สร้างด้วย Declarative Partitioning และเป็น `'r'` (ordinary table) สำหรับ parent ที่ใช้ manual inheritance ทั่วไป ซึ่งเป็นวิธีแยกแยะสองกลไกนี้จาก catalog ได้อย่างชัดเจน แม้ทั้งคู่จะใช้ `pg_inherits` ร่วมกันก็ตาม

</details>

---

## บทถัดไป

เรียนรู้เกี่ยวกับการเชื่อมต่อ PostgreSQL กับแหล่งข้อมูลภายนอก (external data sources) ผ่านกลไก Foreign Data Wrappers ที่ [Part 056: Foreign Data Wrappers](./part-056-foreign-data-wrappers.md)
