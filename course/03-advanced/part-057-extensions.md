# Part 057: Extensions ที่สำคัญ — pg_stat_statements, pgcrypto, uuid-ossp, pg_trgm

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับสูง | Part 057

## เป้าหมายการเรียนรู้

เมื่อจบบทนี้ ผู้เรียนจะสามารถ:

1. อธิบายได้ว่า extension ใน PostgreSQL คืออะไร ทำงานอย่างไร และติดตั้ง/ถอดถอนอย่างไรด้วย `CREATE EXTENSION` / `DROP EXTENSION`
2. ตรวจสอบ extension ที่มีอยู่ในระบบ (`pg_available_extensions`) และที่ติดตั้งแล้วใน database ปัจจุบัน (`pg_extension`, `\dx`)
3. ใช้ `pgcrypto` เพื่อ hash รหัสผ่านอย่างปลอดภัยด้วย `crypt()` และ `gen_salt()` แบบ bcrypt
4. ใช้ `pgcrypto` เพื่อเข้ารหัสข้อมูลอ่อนไหว (PII) แบบ symmetric encryption ด้วย `pgp_sym_encrypt()` / `pgp_sym_decrypt()`
5. เปรียบเทียบ `uuid-ossp` กับฟังก์ชัน built-in `gen_random_uuid()` (ตั้งแต่ PostgreSQL 13) และตัดสินใจได้ว่าเมื่อไหร่ควรใช้ UUID เป็น primary key แทน serial/identity
6. ใช้ `pg_trgm` สำหรับ trigram similarity search ต่อยอดจากความรู้เรื่อง index ใน Part 042
7. เปิดใช้งาน `pg_stat_statements` ผ่านการแก้ไข `postgresql.conf` และ `shared_preload_libraries`
8. วิเคราะห์ query ที่ช้าที่สุดและถูกเรียกบ่อยที่สุดในระบบจริงด้วย `pg_stat_statements`
9. เข้าใจ `hstore` ในฐานะ key-value store รุ่นก่อน JSONB และเหตุผลที่ปัจจุบันแนะนำ JSONB มากกว่า
10. รู้จัก extension เสริมอื่น ๆ ที่สำคัญ เช่น `btree_gin`, `btree_gist`, `pg_repack` และรู้ว่าควรไปศึกษาเพิ่มที่บทไหน
11. ประยุกต์ extension ทั้งหมดที่เรียนมาเข้ากับระบบ e-commerce จริงได้ในแบบฝึกหัดรวม

---

## เตรียมข้อมูล

บทนี้ใช้ schema ฐานข้อมูลร้านค้าออนไลน์ (e-commerce) ชุดเดิมที่ใช้ต่อเนื่องมาตลอดหลักสูตร โดยเพิ่มคอลัมน์ `password_hash` (สำหรับสาธิต pgcrypto) และ `customer_uuid` (สำหรับสาธิต uuid-ossp/gen_random_uuid) เข้าไปในตาราง `customers`

```sql
-- ล้างของเดิมก่อน (ถ้ามี) เพื่อให้รันซ้ำได้
DROP TABLE IF EXISTS orders, customers, products, suppliers, categories CASCADE;

CREATE TABLE categories (
    category_id     SERIAL PRIMARY KEY,
    category_name   VARCHAR(100) NOT NULL
);

CREATE TABLE suppliers (
    supplier_id     SERIAL PRIMARY KEY,
    supplier_name   VARCHAR(150) NOT NULL,
    country         VARCHAR(60)
);

CREATE TABLE products (
    product_id      SERIAL PRIMARY KEY,
    product_name    VARCHAR(150) NOT NULL,
    category_id     INTEGER REFERENCES categories(category_id),
    supplier_id     INTEGER REFERENCES suppliers(supplier_id),
    unit_price      NUMERIC(10,2) NOT NULL
);

CREATE TABLE customers (
    customer_id     SERIAL PRIMARY KEY,
    customer_uuid   UUID DEFAULT gen_random_uuid(),
    first_name      VARCHAR(60) NOT NULL,
    last_name       VARCHAR(60) NOT NULL,
    email           VARCHAR(150) UNIQUE,
    password_hash   TEXT,
    country         VARCHAR(60)
);

CREATE TABLE orders (
    order_id        SERIAL PRIMARY KEY,
    customer_id     INTEGER REFERENCES customers(customer_id),
    order_date      TIMESTAMPTZ NOT NULL DEFAULT now(),
    status          VARCHAR(20) NOT NULL DEFAULT 'pending'
);
```

> หมายเหตุ: คอลัมน์ `customer_uuid UUID DEFAULT gen_random_uuid()` จะยังใช้งานไม่ได้จนกว่าเราจะเปิดใช้งานฟังก์ชันนี้ก่อน (ใน PostgreSQL 13+ ฟังก์ชันนี้เป็น built-in อยู่แล้วใน schema `pg_catalog` จึงเรียกใช้ได้ทันทีโดยไม่ต้อง `CREATE EXTENSION` ใด ๆ — รายละเอียดอยู่ใน Step 564)

### seed data — categories (12 แถว)

```sql
INSERT INTO categories (category_name) VALUES
('Electronics'),
('Computers & Laptops'),
('Mobile Phones'),
('Home Appliances'),
('Furniture'),
('Books'),
('Toys & Games'),
('Sports & Outdoors'),
('Beauty & Personal Care'),
('Groceries'),
('Fashion & Apparel'),
('Automotive Accessories');
```

### seed data — suppliers (12 แถว)

```sql
INSERT INTO suppliers (supplier_name, country) VALUES
('Bangkok Tech Distribution', 'Thailand'),
('Shenzhen Electronics Co.', 'China'),
('Global Gadgets Inc.', 'USA'),
('Osaka Home Appliances', 'Japan'),
('Berlin Furniture Werks', 'Germany'),
('Seoul Mobile Corp.', 'South Korea'),
('Hanoi Textile Group', 'Vietnam'),
('Taipei Components Ltd.', 'Taiwan'),
('Singapore Trading House', 'Singapore'),
('Kuala Lumpur Supplies', 'Malaysia'),
('Mumbai Import Export', 'India'),
('London Retail Partners', 'UK');
```

### seed data — products (20 แถว)

```sql
INSERT INTO products (product_name, category_id, supplier_id, unit_price) VALUES
('Wireless Mouse M1', 1, 2, 259.00),
('Mechanical Keyboard K2', 1, 2, 1290.00),
('27-inch 4K Monitor', 2, 3, 8990.00),
('Business Laptop Pro 14', 2, 3, 32900.00),
('Gaming Laptop X15', 2, 8, 45900.00),
('Smartphone Alpha 12', 3, 6, 18990.00),
('Smartphone Alpha 12 Lite', 3, 6, 12990.00),
('Bluetooth Earbuds Air', 1, 9, 1490.00),
('Robot Vacuum Cleaner R5', 4, 4, 9990.00),
('Air Purifier Home 3', 4, 4, 5990.00),
('Ergonomic Office Chair', 5, 5, 4990.00),
('Standing Desk 120cm', 5, 5, 7990.00),
('Bestseller Novel: The Silent River', 6, 12, 259.00),
('Children Puzzle Set 100pc', 7, 10, 199.00),
('Remote Control Drone Mini', 7, 2, 1990.00),
('Yoga Mat Premium', 8, 7, 590.00),
('Running Shoes UltraFit', 8, 9, 2490.00),
('Vitamin C Serum 30ml', 9, 11, 490.00),
('Organic Green Tea 100g', 10, 11, 150.00),
('Car Phone Holder Magnetic', 12, 8, 299.00);
```

### seed data — customers (15 แถว, ยังไม่ใส่ password_hash)

```sql
INSERT INTO customers (first_name, last_name, email, country) VALUES
('Somchai', 'Jaidee', 'somchai.j@example.com', 'Thailand'),
('Suda', 'Boonmee', 'suda.b@example.com', 'Thailand'),
('Anan', 'Wongsa', 'anan.w@example.com', 'Thailand'),
('Malee', 'Srisuk', 'malee.s@example.com', 'Thailand'),
('Piti', 'Chaiyaporn', 'piti.c@example.com', 'Thailand'),
('Wanida', 'Thongdee', 'wanida.t@example.com', 'Thailand'),
('John', 'Smith', 'john.smith@example.com', 'USA'),
('Emily', 'Johnson', 'emily.j@example.com', 'USA'),
('Hiroshi', 'Tanaka', 'hiroshi.t@example.com', 'Japan'),
('Yuki', 'Sato', 'yuki.sato@example.com', 'Japan'),
('Li', 'Wei', 'li.wei@example.com', 'China'),
('Min-jun', 'Kim', 'minjun.kim@example.com', 'South Korea'),
('Nguyen', 'Van An', 'van.an@example.com', 'Vietnam'),
('Siti', 'Rahman', 'siti.rahman@example.com', 'Malaysia'),
('Oliver', 'Brown', 'oliver.brown@example.com', 'UK');
```

เราจะเติมค่า `password_hash` ให้ลูกค้ากลุ่มนี้ด้วย `pgcrypto` ใน Step 562 (แทนที่จะ hard-code ค่า hash ไว้ล่วงหน้า เพราะค่า hash ที่ถูกต้องต้องสร้างจากฟังก์ชันของ extension โดยตรง)

### seed data — orders (20 แถว)

```sql
INSERT INTO orders (customer_id, order_date, status) VALUES
(1,  '2026-01-05 09:12:00+07', 'completed'),
(2,  '2026-01-06 14:30:00+07', 'completed'),
(1,  '2026-01-10 11:05:00+07', 'completed'),
(3,  '2026-02-01 08:45:00+07', 'shipped'),
(4,  '2026-02-03 19:20:00+07', 'completed'),
(5,  '2026-02-14 10:10:00+07', 'cancelled'),
(6,  '2026-02-20 16:40:00+07', 'completed'),
(7,  '2026-03-01 22:05:00+07', 'completed'),
(8,  '2026-03-02 07:55:00+07', 'pending'),
(9,  '2026-03-05 13:15:00+07', 'completed'),
(10, '2026-03-08 09:30:00+07', 'shipped'),
(2,  '2026-03-11 12:00:00+07', 'completed'),
(11, '2026-03-15 18:45:00+07', 'completed'),
(12, '2026-03-18 21:30:00+07', 'pending'),
(13, '2026-03-20 06:20:00+07', 'completed'),
(14, '2026-03-22 15:10:00+07', 'shipped'),
(15, '2026-03-25 10:00:00+07', 'completed'),
(3,  '2026-04-01 09:45:00+07', 'completed'),
(6,  '2026-04-03 17:25:00+07', 'cancelled'),
(9,  '2026-04-05 11:11:00+07', 'completed');
```

ทดสอบว่าโครงสร้างพร้อมแล้ว:

```sql
SELECT c.category_name, COUNT(p.product_id) AS product_count
FROM categories c
LEFT JOIN products p USING (category_id)
GROUP BY c.category_name
ORDER BY product_count DESC;
```

```
      category_name       | product_count
---------------------------+---------------
 Electronics               |             3
 Computers & Laptops       |             3
 Home Appliances           |             2
 Mobile Phones             |             2
 Furniture                 |             2
 Sports & Outdoors         |             2
 Books                     |             1
 Toys & Games              |             2
 Beauty & Personal Care    |             1
 Groceries                 |             1
 Automotive Accessories    |             1
 Fashion & Apparel         |             0
(12 rows)
```

ทุกอย่างพร้อมแล้ว ต่อไปเรามาทำความรู้จักกับ "extension" ระบบปฏิบัติการเสริมของ PostgreSQL กัน

---

## Step 561: Extension ใน PostgreSQL คืออะไร

### แนวคิดพื้นฐาน

PostgreSQL ถูกออกแบบมาให้ **ขยายความสามารถได้ (extensible)** ตั้งแต่แกนกลาง นักพัฒนาสามารถเขียนชุดฟังก์ชัน, data type, operator, index access method ใหม่ ๆ แล้วห่อรวมเป็น "extension" หนึ่งก้อน เพื่อให้ผู้ใช้ติดตั้งและถอดถอนได้ง่ายด้วยคำสั่งเดียว แทนที่จะต้องไล่รัน SQL script ทีละไฟล์เหมือนสมัยก่อน PostgreSQL 9.1

extension แต่ละตัวประกอบด้วย:

- **control file** (`.control`) — metadata บอกชื่อ, เวอร์ชัน, คำอธิบาย
- **SQL script** — คำสั่งสร้าง function, type, operator ต่าง ๆ
- **shared library** (`.so` บน Linux) — โค้ดภาษา C ที่ compile ไว้แล้ว (ถ้า extension นั้นต้องมีโค้ดระดับ native)

extension บางตัว เช่น `pgcrypto`, `pg_trgm`, `uuid-ossp` ถูก "แพ็คมาด้วย" กับ PostgreSQL อยู่แล้ว (เรียกว่า **contrib module**) เพียงแค่เรารันคำสั่ง `CREATE EXTENSION` ก็ใช้งานได้ทันที ไม่ต้องดาวน์โหลดอะไรเพิ่ม เพราะไฟล์ทั้งหมดถูกติดตั้งไว้ในเครื่อง server ตั้งแต่ตอนติดตั้ง PostgreSQL แล้ว (หรือผ่าน `postgresql-contrib` package บน Linux)

### ตรวจสอบ extension ที่ "มีให้ใช้" ในเครื่อง server

```sql
SELECT name, default_version, comment
FROM pg_available_extensions
ORDER BY name
LIMIT 15;
```

```
       name        | default_version |                        comment
--------------------+------------------+---------------------------------------------------------
 adminpack          | 2.1              | administrative functions for PostgreSQL
 amcheck            | 1.4              | functions for verifying relation integrity
 autoinc            | 1.0              | functions for autoincrementing fields
 bloom              | 1.0              | bloom access method - signature file based index
 btree_gin          | 1.3              | btree_gin support for common types
 btree_gist         | 1.7              | btree_gist support for common types
 citext             | 1.6              | data type for citext
 cube               | 1.5              | data type for multidimensional cubes
 dict_int           | 1.4              | dict_int - text search dictionary template for integers
 earthdistance      | 1.1              | contrib module for earth distance calculations
 fuzzystrmatch      | 1.2              | determine string similarity based on Levenshtein distance
 hstore             | 1.8              | key-value store for PostgreSQL
 pg_stat_statements | 1.11             | track planning and execution statistics of all SQL...
 pg_trgm            | 1.6              | text similarity measurement and index searching...
 pgcrypto           | 1.3              | cryptographic functions
(15 rows)
```

ค่านี้ขึ้นกับว่า server ติดตั้ง contrib package ครบหรือไม่ ถ้าค้นหาแล้วไม่เจอ extension ที่ต้องการ แปลว่าต้องติดตั้ง package เพิ่มระดับ OS (เช่น `apt install postgresql-contrib` หรือใน managed service ต้องเปิดผ่านหน้า console)

### ตรวจสอบ extension ที่ "ติดตั้งแล้ว" ใน database ปัจจุบัน

```sql
\dx
```

```
                                    List of installed extensions
   Name   | Version |   Schema   |                         Description
----------+---------+------------+--------------------------------------------------------------
 plpgsql  | 1.0     | pg_catalog | PL/pgSQL procedural language
(1 row)
```

ในฐานข้อมูลใหม่ ตามปกติจะมีแค่ `plpgsql` ติดตั้งมาให้เป็นค่าเริ่มต้นเท่านั้น หรือใช้ query แบบ SQL ล้วนก็ได้ (สำหรับใช้ในสคริปต์อัตโนมัติที่ไม่ผ่าน psql):

```sql
SELECT extname, extversion, extnamespace::regnamespace AS schema
FROM pg_extension
ORDER BY extname;
```

```
 extname | extversion |   schema
---------+------------+------------
 plpgsql | 1.0        | pg_catalog
(1 row)
```

### ติดตั้งและถอดถอน extension

```sql
CREATE EXTENSION IF NOT EXISTS pgcrypto;
```

```
CREATE EXTENSION
```

`IF NOT EXISTS` ช่วยให้ script รันซ้ำได้โดยไม่ error ถ้า extension ถูกติดตั้งไว้แล้ว การติดตั้งจะสร้างฟังก์ชัน/type ทั้งหมดของ extension นั้นไว้ใน schema ที่กำหนด (ปกติคือ `public` ถ้าไม่ระบุ) ตรวจสอบซ้ำ:

```sql
\dx pgcrypto
```

```
                             List of installed extensions
   Name   | Version |   Schema   |                Description
----------+---------+------------+--------------------------------------------
 pgcrypto | 1.3     | public     | cryptographic functions
(1 row)
```

ถอดถอนด้วย:

```sql
DROP EXTENSION IF EXISTS pgcrypto;
```

```
DROP EXTENSION
```

> **ข้อควรระวัง**: การ `DROP EXTENSION` จะลบฟังก์ชันทั้งหมดของ extension นั้นออกไปด้วย ถ้ามี object อื่น (เช่น column ที่ตั้ง default เป็นฟังก์ชันของ extension, หรือ index ที่ใช้ operator class จาก extension) ยังอ้างอิงอยู่ PostgreSQL จะปฏิเสธการ DROP (เว้นแต่ใช้ `CASCADE`) เพื่อป้องกันความเสียหาย

### ระบุ schema และ version

```sql
CREATE EXTENSION IF NOT EXISTS pg_trgm SCHEMA public VERSION '1.6';
```

```
CREATE EXTENSION
```

อัปเกรดเวอร์ชันของ extension (เมื่อ PostgreSQL เวอร์ชันใหม่มาพร้อม extension เวอร์ชันใหม่กว่า):

```sql
ALTER EXTENSION pg_trgm UPDATE TO '1.6';
```

```
ALTER EXTENSION
```

### เตรียม extension ทั้งหมดที่ใช้ในบทนี้

เพื่อความสะดวก เราติดตั้ง extension หลักที่จะใช้ตลอดบทนี้ไว้ล่วงหน้า:

```sql
CREATE EXTENSION IF NOT EXISTS pgcrypto;
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE EXTENSION IF NOT EXISTS hstore;
```

```
CREATE EXTENSION
CREATE EXTENSION
CREATE EXTENSION
CREATE EXTENSION
```

> สังเกตว่า `uuid-ossp` ต้องใส่ในเครื่องหมายคำพูด (`"uuid-ossp"`) เพราะชื่อมีเครื่องหมายขีด (`-`) ซึ่งไม่ใช่อักขระที่ใช้ได้ใน identifier ปกติของ SQL

`pg_stat_statements` จะพิเศษกว่าตัวอื่น เพราะต้องแก้ `postgresql.conf` ก่อนถึงจะ `CREATE EXTENSION` ได้สำเร็จ — รายละเอียดอยู่ใน Step 566

ตรวจสอบรายการที่ติดตั้งแล้วทั้งหมด:

```sql
\dx
```

```
                                       List of installed extensions
   Name    | Version |   Schema   |                            Description
------------+---------+------------+--------------------------------------------------------------
 hstore     | 1.8     | public     | key-value store for PostgreSQL
 pg_trgm    | 1.6     | public     | text similarity measurement and index searching...
 pgcrypto   | 1.3     | public     | cryptographic functions
 plpgsql    | 1.0     | pg_catalog | PL/pgSQL procedural language
 uuid-ossp  | 1.1     | public     | generate universally unique identifiers (UUIDs)
(5 rows)
```

---

## Step 562: pgcrypto — เข้ารหัสและ hash รหัสผ่านด้วย crypt() และ gen_salt()

### ทำไมห้าม hash รหัสผ่านด้วย MD5 หรือ SHA ตรง ๆ

ข้อผิดพลาดคลาสสิกของมือใหม่คือเก็บรหัสผ่านด้วย `md5('password')` หรือ `sha256('password')` ตรง ๆ ปัญหาคือ:

1. **ไม่มี salt** — รหัสผ่านเดียวกันจะได้ hash เดียวกันเสมอ ทำให้แฮกเกอร์ใช้ rainbow table โจมตีได้ง่าย
2. **เร็วเกินไป** — MD5/SHA ถูกออกแบบมาให้คำนวณเร็ว (เหมาะกับ checksum) ซึ่งกลับกลายเป็นจุดอ่อนสำหรับรหัสผ่าน เพราะแฮกเกอร์ brute-force ได้เร็วตามไปด้วย

`pgcrypto` มีฟังก์ชัน `crypt()` ที่รองรับอัลกอริทึม **bcrypt** (`bf`) ซึ่งออกแบบมาเพื่อการ hash รหัสผ่านโดยเฉพาะ — มี salt แบบสุ่มในตัว และปรับ "cost factor" ให้คำนวณช้าลงได้ตามต้องการ (ยิ่งช้า ยิ่งทนต่อการ brute-force)

### gen_salt() และ crypt()

```sql
SELECT gen_salt('bf', 10);
```

```
              gen_salt
--------------------------------------
 $2a$10$N9qo8uLOickgx2ZMRZoMye
(1 row)
```

`gen_salt('bf', 10)` สร้าง salt แบบ bcrypt โดยมี cost factor เท่ากับ 10 (ยิ่งตัวเลขสูง ยิ่งใช้เวลาคำนวณนานขึ้นแบบ exponential — ค่ามาตรฐานที่แนะนำอยู่ระหว่าง 10-12 สำหรับปี 2026)

```sql
SELECT crypt('mySecretPass123', gen_salt('bf', 10));
```

```
                            crypt
--------------------------------------------------------------
 $2a$10$eW5UsL0m8T2vQ7YbXjKz0eqfN1c8s0rP4hVn1cRZ0Iu3q7YtL2wSa
(1 row)
```

ค่าที่ได้คือ hash string ที่มีทั้ง algorithm identifier (`$2a$`), cost factor (`10$`), salt และ hash รวมอยู่ในสตริงเดียว — เก็บสตริงนี้ทั้งหมดลงในคอลัมน์ `password_hash` ได้เลยโดยไม่ต้องแยกเก็บ salt ต่างหาก

### เติมรหัสผ่านให้ลูกค้าในตาราง customers

```sql
UPDATE customers
SET password_hash = crypt('Passw0rd_' || customer_id, gen_salt('bf', 10));
```

```
UPDATE 15
```

ตรวจสอบผลลัพธ์:

```sql
SELECT customer_id, email, password_hash
FROM customers
ORDER BY customer_id
LIMIT 3;
```

```
 customer_id |          email           |                         password_hash
-------------+--------------------------+------------------------------------------------------------
           1 | somchai.j@example.com    | $2a$10$3fH9k1QpXG5aVn0e2Jc8ZuYtR6mLb2QwOo9sD8vNx1Ck7Pj3Hs2q
           2 | suda.b@example.com       | $2a$10$Vb2nM8c1Gk0RfT3xYqL9WeUp6Zj2Ao5Ds7Nh1Xc4Vw9Kt0Br6Lm
           3 | anan.w@example.com       | $2a$10$Qc8Rn2Kv5Ht3Mp1Jd0Wz6XuLb4Yf9Gs2Ao7Tk1Nc3Vh5Qw8Rj2S
(3 rows)
```

> **หมายเหตุ**: ค่า hash ในตัวอย่างเป็นตัวอย่างสมมติเพื่อการสาธิตรูปแบบ (format) เท่านั้น เมื่อรันจริงบนเครื่องของท่าน bcrypt จะสุ่ม salt ใหม่ทุกครั้ง จึงได้ค่าจริงที่แตกต่างออกไปเสมอ แม้ input จะเหมือนกันทุกประการ

### ตรวจสอบรหัสผ่านตอน login

หลักการคือ นำรหัสผ่านที่ผู้ใช้กรอกมา ไป `crypt()` ด้วย **salt เดิมที่สกัดมาจาก hash ที่เก็บไว้** (ฟังก์ชัน `crypt()` ฉลาดพอที่จะอ่าน salt จากพารามิเตอร์ตัวที่สองได้เอง) แล้วเทียบสตริงผลลัพธ์กับ hash ที่เก็บไว้:

```sql
SELECT customer_id, email
FROM customers
WHERE email = 'somchai.j@example.com'
  AND password_hash = crypt('Passw0rd_1', password_hash);
```

```
 customer_id |         email
-------------+------------------------
           1 | somchai.j@example.com
(1 row)
```

ถ้ารหัสผ่านผิด จะไม่มีแถวใดตรงเงื่อนไข:

```sql
SELECT customer_id, email
FROM customers
WHERE email = 'somchai.j@example.com'
  AND password_hash = crypt('WrongPassword', password_hash);
```

```
 customer_id | email
-------------+-------
(0 rows)
```

### ห่อเป็นฟังก์ชันใช้งานจริง

เพื่อให้ใช้งานสะดวกและลดโอกาสเขียนผิด ควรห่อ logic นี้เป็นฟังก์ชัน:

```sql
CREATE OR REPLACE FUNCTION verify_customer_login(p_email TEXT, p_password TEXT)
RETURNS BOOLEAN
LANGUAGE sql
STABLE
AS $$
    SELECT EXISTS (
        SELECT 1
        FROM customers
        WHERE email = p_email
          AND password_hash = crypt(p_password, password_hash)
    );
$$;
```

```
CREATE FUNCTION
```

```sql
SELECT verify_customer_login('suda.b@example.com', 'Passw0rd_2');
SELECT verify_customer_login('suda.b@example.com', 'incorrect');
```

```
 verify_customer_login
------------------------
 t
(1 row)

 verify_customer_login
------------------------
 f
(1 row)
```

### cost factor สูงแค่ไหนถึงจะพอ

```sql
-- ทดสอบเวลาที่ใช้ในการ hash ที่ cost factor ต่าง ๆ
\timing on
SELECT crypt('test', gen_salt('bf', 12));
```

```
                            crypt
--------------------------------------------------------------
 $2a$12$xxxxxxxxxxxxxxxxxxxxxOeYoBz1kIz3s5rQ9uVw2xJc4Nn7Tp8Ka
(1 row)

Time: 312.845 ms
```

cost factor 12 ใช้เวลาประมาณ 300 มิลลิวินาทีต่อการ hash หนึ่งครั้ง ซึ่งเหมาะสำหรับ authentication endpoint ที่ไม่ต้องรับ traffic สูงมาก แต่ถ้าระบบมีผู้ใช้ login พร้อมกันจำนวนมาก อาจพิจารณาลดเหลือ cost 10-11 หรือย้าย logic การ hash ไปทำที่ชั้น application (เช่น bcrypt library ใน backend) แทนที่จะให้ database รับภาระ CPU ทั้งหมด — เป็นการตัดสินใจ trade-off ระหว่างความปลอดภัยกับ throughput ที่ทีม backend ต้องช่วยกันพิจารณา

---

## Step 563: pgcrypto ขั้นสูง — pgp_sym_encrypt/decrypt สำหรับข้อมูลอ่อนไหว

### ความแตกต่างระหว่าง hash กับ encryption

`crypt()` ใน Step 562 เป็น **hash แบบทางเดียว (one-way)** — เราตรวจสอบได้ว่ารหัสผ่านตรงกันหรือไม่ แต่ **ไม่สามารถถอดกลับเป็นข้อความต้นฉบับได้** เหมาะกับรหัสผ่านที่ระบบไม่จำเป็นต้องรู้ค่าจริง

แต่สำหรับข้อมูลอ่อนไหวบางประเภท เช่น เลขบัตรประชาชน, เลขบัญชีธนาคาร, หรือที่อยู่ลูกค้า ระบบ **จำเป็นต้องอ่านค่ากลับมาได้** (เช่น แสดงผลในหน้าโปรไฟล์ หรือส่งไปประมวลผลการเงิน) กรณีนี้ต้องใช้ **symmetric encryption แบบสองทาง (two-way)** ด้วย `pgp_sym_encrypt()` / `pgp_sym_decrypt()`

### เข้ารหัสข้อมูลด้วย pgp_sym_encrypt

```sql
SELECT pgp_sym_encrypt('1234567890123', 'my-encryption-passphrase');
```

```
                                          pgp_sym_encrypt
----------------------------------------------------------------------------------------------------
 \xc30d04070302a1b2c3d4e5f6...  (binary bytea, ตัดให้สั้นเพื่อการแสดงผล)
(1 row)
```

ผลลัพธ์เป็นชนิด `bytea` (binary) ที่เข้ารหัสด้วยมาตรฐาน OpenPGP โดยใช้ passphrase เป็นกุญแจ ดังนั้นคอลัมน์ที่จะเก็บค่านี้ต้องเป็นชนิด `bytea` ไม่ใช่ `TEXT`

### ตัวอย่าง: เพิ่มคอลัมน์เก็บเลขบัตรประชาชนแบบเข้ารหัส

```sql
ALTER TABLE customers ADD COLUMN national_id_encrypted BYTEA;
```

```
ALTER TABLE
```

```sql
UPDATE customers
SET national_id_encrypted = pgp_sym_encrypt(
    '1' || LPAD(customer_id::TEXT, 12, '0'),  -- เลขบัตรสมมติ
    'app-secret-passphrase-2026'
)
WHERE customer_id <= 5;
```

```
UPDATE 5
```

### ถอดรหัสกลับ

```sql
SELECT
    customer_id,
    email,
    pgp_sym_decrypt(national_id_encrypted, 'app-secret-passphrase-2026') AS national_id
FROM customers
WHERE customer_id <= 5;
```

```
 customer_id |          email           | national_id
-------------+--------------------------+---------------
           1 | somchai.j@example.com    | 1000000000001
           2 | suda.b@example.com       | 1000000000002
           3 | anan.w@example.com       | 1000000000003
           4 | malee.s@example.com      | 1000000000004
           5 | piti.c@example.com       | 1000000000005
(5 rows)
```

ถ้าใส่ passphrase ผิด PostgreSQL จะโยน error ทันที (ป้องกันการเดาสุ่ม):

```sql
SELECT pgp_sym_decrypt(national_id_encrypted, 'wrong-passphrase')
FROM customers WHERE customer_id = 1;
```

```
ERROR:  Wrong key or corrupt data
```

### ทางเลือกอัลกอริทึมและ compression

`pgp_sym_encrypt` รับพารามิเตอร์เสริมเพื่อกำหนด cipher algorithm ได้:

```sql
SELECT pgp_sym_encrypt(
    'sensitive data',
    'passphrase',
    'cipher-algo=aes256, compress-algo=1, compress-level=6'
);
```

```
                                      pgp_sym_encrypt
--------------------------------------------------------------------------------------------
 \x8c0d04...  (bytea, ใช้ AES-256 แทน default 3DES/CAST5)
(1 row)
```

ค่า default ของ `pgp_sym_encrypt` คือ cipher `AES128` ซึ่งปลอดภัยเพียงพอสำหรับงานส่วนใหญ่ แต่หากนโยบายความปลอดภัยขององค์กรกำหนดให้ต้องใช้ AES-256 ก็ระบุ `cipher-algo=aes256` เพิ่มได้ตามตัวอย่าง

### ข้อควรระวังสำคัญเรื่องการจัดการ key

จุดอ่อนที่สุดของวิธีนี้ไม่ใช่ตัวอัลกอริทึม แต่คือ **การจัดเก็บ passphrase**:

- **ห้าม hard-code passphrase ไว้ใน SQL script หรือ commit ลง version control**
- ควรอ่าน passphrase จาก secret manager (เช่น AWS Secrets Manager, HashiCorp Vault) ที่ชั้น application แล้วส่งเข้ามาเป็น parameter ของ query เท่านั้น
- ถ้า passphrase หลุด เท่ากับข้อมูลที่เข้ารหัสทั้งหมดหลุดตามไปด้วย เพราะเป็น symmetric key (กุญแจเดียวใช้ทั้งเข้าและถอดรหัส)
- ควรพิจารณาเรื่อง **key rotation** — เปลี่ยน passphrase เป็นระยะ โดยถอดรหัสด้วย key เก่าแล้วเข้ารหัสใหม่ด้วย key ใหม่ทั้งฐานข้อมูล (batch job)
- pgcrypto เข้ารหัส/ถอดรหัสที่ฝั่ง database server ซึ่งหมายความว่า plaintext และ passphrase ต้องเดินทางผ่าน network connection ไปยัง server (ควรบังคับใช้ `sslmode=require` หรือสูงกว่าเสมอ)

สำหรับข้อมูลที่ sensitive ระดับสูงมาก (เช่น ข้อมูลบัตรเครดิตแบบเต็ม) หลายองค์กรเลือกใช้ **application-level encryption** (เข้ารหัสที่ backend ก่อนส่งเข้า database) หรือใช้บริการ tokenization จากภายนอกแทน เพื่อไม่ให้ database เห็น plaintext เลยแม้แต่ชั่วขณะ — `pgcrypto` เหมาะกับกรณีที่ยอมรับความเสี่ยงว่า database และ connection ระหว่างทางเชื่อถือได้ในระดับหนึ่ง

---

## Step 564: uuid-ossp vs gen_random_uuid() — เมื่อไหร่ควรใช้ UUID เป็น primary key

### uuid-ossp คืออะไร

`uuid-ossp` เป็น extension ดั้งเดิมที่ให้ฟังก์ชันสร้าง UUID ตามมาตรฐาน RFC 4122 หลายเวอร์ชัน:

```sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

SELECT uuid_generate_v1() AS v1_time_based,
       uuid_generate_v4() AS v4_random;
```

```
             v1_time_based             |              v4_random
----------------------------------------+---------------------------------------
 a1b2c3d4-1234-11ef-9abc-0242ac120002   | 6f8e9d2c-5a3b-4f1e-8c7d-2b1a9e0f3c4d
(1 row)
```

- **`uuid_generate_v1()`** — สร้างจาก timestamp + MAC address ของเครื่อง ทำให้ **เดาลำดับเวลาการสร้างได้** (เรียงตามเวลาโดยประมาณ) แต่ก็มีข้อเสียคือรั่วไหลข้อมูล MAC address และเดา pattern ได้ในระดับหนึ่ง
- **`uuid_generate_v4()`** — สุ่มล้วน 122 บิต (random) ไม่มี pattern ให้เดา เป็นตัวที่นิยมใช้เป็น primary key มากที่สุด

### gen_random_uuid() — built-in ตั้งแต่ PostgreSQL 13

ตั้งแต่ PostgreSQL 13 เป็นต้นมา มีฟังก์ชัน `gen_random_uuid()` อยู่ใน `pg_catalog` (built-in) โดยไม่ต้องติดตั้ง extension ใด ๆ เลย:

```sql
SELECT gen_random_uuid();
```

```
             gen_random_uuid
---------------------------------------
 3d9f4a1e-7b2c-4e6d-9a1f-8c3b5d7e9f01
(1 row)
```

ค่าที่ได้เทียบเท่ากับ `uuid_generate_v4()` ทุกประการ (สุ่มแบบ version 4) เพียงแต่**ไม่ต้อง `CREATE EXTENSION`** เนื่องจากพึ่งพา cryptographically strong random function ที่ built เข้ามาใน PostgreSQL core โดยตรง

### ตารางเปรียบเทียบ

| หัวข้อ | `uuid-ossp` (`uuid_generate_v4()`) | built-in `gen_random_uuid()` (PG13+) |
|---|---|---|
| ต้องติดตั้ง extension | ต้องมี (`CREATE EXTENSION "uuid-ossp"`) | ไม่ต้อง (built-in) |
| ใช้ได้ตั้งแต่ | ทุกเวอร์ชันของ PostgreSQL | PostgreSQL 13 ขึ้นไป |
| รองรับ UUID v1 (time-based) | รองรับ | ไม่รองรับ (สร้างได้แค่ v4) |
| ความเร็ว | ใกล้เคียงกัน | ใกล้เคียงกัน |
| Portability ข้าม managed service | บาง managed service (เช่น บาง cloud provider รุ่นเก่า) จำกัดสิทธิ์ superuser ในการติดตั้ง extension | ใช้ได้เสมอโดยไม่ต้องขอสิทธิ์เพิ่ม |
| คำแนะนำปี 2026 | ใช้เมื่อจำเป็นต้องมี UUID v1/v3/v5 เท่านั้น | **แนะนำเป็นค่าเริ่มต้นสำหรับ PK แบบ random UUID** |

> สรุปสั้น ๆ: ถ้าใช้ PostgreSQL 13 ขึ้นไป (ซึ่งปัจจุบันแทบทุกระบบควรเป็นเวอร์ชัน 16/17 อยู่แล้ว) และต้องการแค่ UUID สุ่ม ให้ใช้ `gen_random_uuid()` เป็นค่าเริ่มต้น ไม่จำเป็นต้องติดตั้ง `uuid-ossp` เพิ่มเลย — นี่คือเหตุผลที่ schema ของเราออกแบบ `customer_uuid UUID DEFAULT gen_random_uuid()` มาตั้งแต่ต้น

### ตรวจสอบว่า default ทำงานถูกต้อง

```sql
SELECT customer_id, customer_uuid, email
FROM customers
ORDER BY customer_id
LIMIT 5;
```

```
 customer_id |             customer_uuid             |          email
-------------+----------------------------------------+--------------------------
           1 | 3d9f4a1e-7b2c-4e6d-9a1f-8c3b5d7e9f01     | somchai.j@example.com
           2 | 9a2c1f0e-4b8d-4a3c-b7e1-5d6f8a0c2e4b     | suda.b@example.com
           3 | 5e7f2a9c-1d3b-4c6e-8a0f-2b4d6e8f0a1c     | anan.w@example.com
           4 | 8b1d3f5a-6c0e-4d2b-9a4c-7e1f3a5b7c9d     | malee.s@example.com
           5 | 2c4e6a8b-0d1f-4b3c-5a7e-9c1d3f5a7b9c     | piti.c@example.com
(5 rows)
```

ทุกแถวมีค่า UUID แล้วโดยอัตโนมัติเพราะเราตั้ง `DEFAULT gen_random_uuid()` ไว้ตอนสร้างตาราง โดยที่ไม่ต้องระบุค่าตอน `INSERT`

### เมื่อไหร่ควรใช้ UUID เป็น primary key แทน SERIAL/IDENTITY

**ข้อดีของ UUID เป็น PK:**

1. **สร้างได้ที่ฝั่ง client ก่อน insert ลง database** — เหมาะกับระบบ distributed ที่ต้องการรู้ ID ล่วงหน้าก่อนเขียนลงฐานข้อมูล (เช่น สร้าง object ในหลาย microservice พร้อมกันโดยไม่ต้อง round-trip ไปขอ ID จาก database ก่อน)
2. **รวมข้อมูลจากหลายฐานข้อมูล/หลาย shard ได้โดยไม่ชนกัน** — ต่างจาก `SERIAL` ที่ ID อาจซ้ำกันข้าม database instance
3. **ไม่เปิดเผยจำนวนแถวหรืออัตราการเติบโตของธุรกิจ** — URL แบบ `/orders/1042` บอกคู่แข่งได้ทันทีว่าร้านมีออเดอร์ประมาณเท่าไหร่ ขณะที่ `/orders/9a2c1f0e-...` ไม่รั่วไหลข้อมูลนี้
4. **ป้องกันการเดา ID เพื่อ enumerate ข้อมูลคนอื่น** (IDOR-style attack) ได้ในระดับหนึ่ง

**ข้อเสียของ UUID เป็น PK:**

1. **ขนาดใหญ่กว่า** — UUID กิน 16 bytes ต่อแถว เทียบกับ `INTEGER` 4 bytes หรือ `BIGINT` 8 bytes ส่งผลต่อขนาด index ทุกตัวที่อ้างอิงคอลัมน์นี้ (ทั้ง PK index และ FK ในตารางลูกทั้งหมด)
2. **random UUID (v4) ทำให้ B-tree index กระจายแบบสุ่ม** — insert ใหม่แต่ละครั้งอาจตกไปคนละ page ของ index ทำให้เกิด **random I/O** และ **page split** บ่อยกว่า sequential ID มาก ส่งผลเสียต่อ cache locality และ write amplification โดยเฉพาะบนตารางขนาดใหญ่มาก
3. **อ่านด้วยตาคนยาก** เวลา debug หรือคุยกันในทีม เทียบ `order_id = 1042` กับ `order_id = '9a2c1f0e-4b8d-...'`

**แนวทางที่นิยมในระบบระดับ production ปี 2026** คือใช้ **ทั้งสองอย่างร่วมกัน**: เก็บ `SERIAL`/`BIGINT GENERATED ALWAYS AS IDENTITY` เป็น internal primary key สำหรับ join และ index ภายใน แต่เปิดเผย `UUID` (เช่นคอลัมน์ `customer_uuid` ในตัวอย่างของเรา) ให้กับภายนอกผ่าน API เป็น "public identifier" แทน — ได้ประสิทธิภาพของ integer PK และความปลอดภัย/ความยืดหยุ่นของ UUID ไปพร้อมกัน ซึ่งเป็นเหตุผลที่ schema ในบทนี้ออกแบบให้ `customers` มีทั้ง `customer_id SERIAL` (ใช้ join ภายใน) และ `customer_uuid UUID` (เปิดเผยผ่าน API)

```sql
-- ตัวอย่าง: API endpoint ควรค้นหาด้วย uuid ไม่ใช่ id ตรง ๆ
SELECT customer_id, first_name, last_name, email
FROM customers
WHERE customer_uuid = '3d9f4a1e-7b2c-4e6d-9a1f-8c3b5d7e9f01';
```

```
 customer_id | first_name | last_name |          email
-------------+------------+-----------+--------------------------
           1 | Somchai     | Jaidee    | somchai.j@example.com
(1 row)
```

อย่าลืมสร้าง index บนคอลัมน์ `customer_uuid` ถ้าจะใช้ query แบบนี้บ่อย:

```sql
CREATE UNIQUE INDEX idx_customers_uuid ON customers (customer_uuid);
```

```
CREATE INDEX
```

หากต้องการ UUID ที่ **เรียงตามเวลาได้ (time-ordered)** เพื่อลดปัญหา random I/O ของ v4 ขณะยังคงข้อดีของ UUID ไว้ PostgreSQL 18 ได้เพิ่มฟังก์ชัน `uuidv7()` (UUID version 7 ตามมาตรฐาน RFC 9562) เข้ามาเป็น built-in ซึ่งฝัง timestamp ไว้ในบิตต้น ๆ ทำให้ insert ใหม่มักตกท้าย index เหมือน sequential ID แต่ยังคงเดายากเหมือน UUID ทั่วไป — เป็นหัวข้อที่ควรติดตามเมื่อทีมอัปเกรดไปยัง PostgreSQL เวอร์ชันนั้นในอนาคต

---

## Step 565: pg_trgm — Trigram Similarity Search (ทบทวนเชื่อมโยง Part 042)

### ทบทวนสั้น ๆ จาก Part 042

ใน Part 042 เราเคยแนะนำ `pg_trgm` ในฐานะหนึ่งใน index type พิเศษ (GIN/GiST ที่ใช้ operator class `gin_trgm_ops`) สำหรับเร่งความเร็ว `LIKE '%keyword%'` และ fuzzy text search บทนี้เราจะเจาะลึกฟังก์ชันเบื้องหลังของ extension นี้อย่างครบถ้วน

### แนวคิด trigram

trigram คือการตัดข้อความออกเป็นชุดตัวอักษรต่อเนื่อง 3 ตัว (3-gram) เช่นคำว่า `"cat"` เมื่อเติม padding พิเศษที่ขอบจะถูกตัดเป็น: `"  c"`, `" ca"`, `"cat"`, `"at "` ยิ่งสองข้อความมี trigram ร่วมกันมาก ยิ่งถือว่า "คล้ายกัน" มาก

```sql
SELECT show_trgm('PostgreSQL');
```

```
                                    show_trgm
------------------------------------------------------------------------------
 {"  p"," po",gre,ost,pos,"ql ",res,sql,stg,tgr}
(1 row)
```

### similarity()

ฟังก์ชัน `similarity()` คืนค่าคะแนนความคล้าย 0.0 (ไม่คล้ายเลย) ถึง 1.0 (เหมือนกันทุกตัวอักษร):

```sql
SELECT similarity('Wireless Mouse M1', 'wireless mouse');
```

```
 similarity
------------
       0.56
(1 row)
```

```sql
SELECT product_name, similarity(product_name, 'wireless mouse') AS score
FROM products
ORDER BY score DESC
LIMIT 5;
```

```
       product_name        | score
----------------------------+-------
 Wireless Mouse M1          |  0.56
 Bluetooth Earbuds Air      |  0.10
 Remote Control Drone Mini  |  0.08
 Smartphone Alpha 12        |  0.07
 Robot Vacuum Cleaner R5    |  0.07
(5 rows)
```

ใช้กรณี "ค้นหาสินค้าที่สะกดผิดเล็กน้อย" ได้ดีมาก เช่นลูกค้าพิมพ์ `"wireles mouse"` (สะกดผิด) ก็ยังค้นเจอสินค้าที่ถูกต้อง

### word_similarity() — เทียบคำกับส่วนหนึ่งของประโยคยาว

`similarity()` เทียบทั้งสองข้อความแบบเต็ม แต่ถ้าต้องการเทียบว่า "คำสั้น ๆ นี้ ปรากฏอยู่ในประโยคยาวหรือไม่" (ไม่สนใจว่าส่วนที่เหลือของประโยคจะต่างกันแค่ไหน) ให้ใช้ `word_similarity()`:

```sql
SELECT word_similarity('drone', 'Remote Control Drone Mini');
```

```
 word_similarity
------------------
             0.45
(1 row)
```

```sql
SELECT word_similarity('novel', 'Bestseller Novel: The Silent River');
```

```
 word_similarity
------------------
             0.62
(1 row)
```

### % operator — ตัวดำเนินการเปรียบเทียบความคล้าย

`pg_trgm` เพิ่ม operator `%` ที่คืนค่า `true`/`false` โดยเทียบกับ threshold (ค่าเริ่มต้น 0.3 ปรับได้ด้วย `pg_trgm.similarity_threshold`):

```sql
SELECT product_name
FROM products
WHERE product_name % 'wireles mouse';
```

```
     product_name
----------------------
 Wireless Mouse M1
(1 row)
```

ปรับ threshold ให้เข้มงวดขึ้นหรือหลวมขึ้นได้ในระดับ session:

```sql
SET pg_trgm.similarity_threshold = 0.15;

SELECT product_name, similarity(product_name, 'earbud')
FROM products
WHERE product_name % 'earbud';
```

```
     product_name      | similarity
------------------------+------------
 Bluetooth Earbuds Air  |       0.20
(1 row)
```

มี operator `<%` และ `%>` สำหรับ word similarity แบบทิศทางเดียวเช่นกัน:

```sql
SELECT product_name
FROM products
WHERE 'drone' <% product_name;
```

```
        product_name
-----------------------------
 Remote Control Drone Mini
(1 row)
```

### สร้าง GIN index เร่งความเร็ว (ทบทวนจาก Part 042)

```sql
CREATE INDEX idx_products_name_trgm
ON products USING GIN (product_name gin_trgm_ops);
```

```
CREATE INDEX
```

ทดสอบว่า planner เลือกใช้ index จริงหรือไม่:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT product_name
FROM products
WHERE product_name % 'wireles mouse';
```

```
 Bitmap Heap Scan on products  (cost=12.02..16.04 rows=1 width=19)
                               (actual time=0.045..0.048 rows=1 loops=1)
   Recheck Cond: (product_name % 'wireles mouse'::text)
   Heap Blocks: exact=1
   ->  Bitmap Index Scan on idx_products_name_trgm
       (cost=0.00..12.02 rows=1 width=0) (actual time=0.030..0.031 rows=1 loops=1)
         Index Cond: (product_name % 'wireles mouse'::text)
 Planning Time: 0.312 ms
 Execution Time: 0.081 ms
(7 rows)
```

Index เดียวกันนี้ยังช่วยเร่ง `LIKE '%...%'` และ `ILIKE` แบบไม่มี anchor ที่หน้าข้อความได้ด้วย ซึ่งปกติ B-tree index ทำไม่ได้เลย (ต้อง full table scan) — รายละเอียดเชิงลึกเรื่องการเลือก GIN vs GiST สำหรับ trigram สามารถย้อนกลับไปอ่านได้ที่ Part 042

### จัดอันดับผลการค้นหาแบบ fuzzy search จริง

```sql
SELECT product_name, unit_price,
       similarity(product_name, 'gaming laptop') AS score
FROM products
WHERE product_name % 'gaming laptop'
ORDER BY score DESC, unit_price ASC;
```

```
     product_name     | unit_price | score
-----------------------+------------+-------
 Gaming Laptop X15     |   45900.00 |  0.71
(1 row)
```

pattern นี้ (filter ด้วย `%` แล้ว sort ด้วย `similarity()`) เป็นสูตรมาตรฐานสำหรับสร้างช่อง "search" ในเว็บ e-commerce ที่ทนต่อการพิมพ์ผิดของผู้ใช้

---

## Step 566: pg_stat_statements — เปิดใช้งาน Extension สำคัญที่สุดสำหรับ Performance Monitoring

### ทำไม pg_stat_statements ถึงสำคัญที่สุด

ถ้าต้องเลือก extension เดียวที่ DBA/Backend engineer ทุกคนควรเปิดใช้งานในทุกระบบ production คำตอบคือ `pg_stat_statements` โดยไม่ต้องคิดเยอะ เพราะมันตอบคำถามที่สำคัญที่สุดของการดูแลฐานข้อมูล: **"query ตัวไหนกินทรัพยากรมากที่สุดในระบบจริง"**

ต่างจาก `EXPLAIN ANALYZE` (Part 044) ที่วิเคราะห์ query ทีละตัวที่เรารันเอง `pg_stat_statements` จะ **เก็บสถิติสะสมของทุก query ที่เคยรันผ่าน server จริง** โดยอัตโนมัติ ไม่ว่าจะมาจาก application, cron job, หรือ connection ไหนก็ตาม — ทำให้เห็นภาพรวมการใช้งานจริงที่ไม่มีใครจำลองขึ้นมาเองได้แม่นยำเท่า

### ทำไมถึง CREATE EXTENSION ตรง ๆ ไม่ได้ทันที

`pg_stat_statements` ต่างจาก `pgcrypto`/`pg_trgm`/`uuid-ossp` ตรงที่มันต้อง **hook เข้าไปในกระบวนการ query execution ของ PostgreSQL ตั้งแต่ตอน server เริ่มทำงาน (postmaster startup)** จึงต้องโหลดเป็น **shared library ที่ preload ไว้ล่วงหน้า** ผ่านพารามิเตอร์ `shared_preload_libraries` ใน `postgresql.conf` — พารามิเตอร์นี้เป็นระดับ server (ไม่ใช่ระดับ session) และต้อง **restart PostgreSQL server** ถึงจะมีผล

หากลองรันตรง ๆ โดยไม่ตั้งค่าก่อน:

```sql
CREATE EXTENSION pg_stat_statements;
```

```
CREATE EXTENSION
```

คำสั่งนี้อาจจะสำเร็จ (เพราะมันแค่สร้าง view/function wrapper) แต่ตัว view จะไม่มีข้อมูลอะไรเลย หรือ error เมื่อพยายาม query:

```sql
SELECT * FROM pg_stat_statements LIMIT 1;
```

```
ERROR:  pg_stat_statements must be loaded via "shared_preload_libraries"
```

### ขั้นตอนเปิดใช้งานให้ถูกต้อง

**ขั้นตอนที่ 1**: แก้ไขไฟล์ `postgresql.conf` (หาตำแหน่งไฟล์ด้วย `SHOW config_file;`)

```sql
SHOW config_file;
```

```
                  config_file
-------------------------------------------------
 /etc/postgresql/17/main/postgresql.conf
(1 row)
```

เปิดไฟล์นี้ด้วย text editor แล้วเพิ่ม/แก้บรรทัด:

```ini
# postgresql.conf
shared_preload_libraries = 'pg_stat_statements'

# ถ้ามี extension อื่นที่ต้อง preload อยู่แล้ว ให้คั่นด้วย comma
# shared_preload_libraries = 'pg_stat_statements, pg_cron, auto_explain'

pg_stat_statements.max = 10000
pg_stat_statements.track = all
pg_stat_statements.track_utility = on
pg_stat_statements.save = on
```

พารามิเตอร์เสริมที่ควรรู้จัก:

| พารามิเตอร์ | ความหมาย | ค่าแนะนำ |
|---|---|---|
| `pg_stat_statements.max` | จำนวน query pattern สูงสุดที่เก็บสถิติพร้อมกัน (เกินแล้วตัวที่ใช้น้อยสุดจะถูกลบทิ้ง) | 5000-10000 |
| `pg_stat_statements.track` | ระดับการติดตาม: `none`, `top` (เฉพาะ query ระดับบนสุด), `all` (รวม query ใน function ด้วย) | `all` |
| `pg_stat_statements.track_utility` | ติดตามคำสั่งที่ไม่ใช่ DML เช่น `VACUUM`, `CREATE INDEX` ด้วยหรือไม่ | `on` |
| `pg_stat_statements.save` | เก็บสถิติไว้ข้าม server restart หรือไม่ (เขียนลงไฟล์) | `on` |

**ขั้นตอนที่ 2**: Restart PostgreSQL server (คำสั่งขึ้นกับ OS)

```bash
sudo systemctl restart postgresql
# หรือบนบาง distro:
sudo pg_ctlcluster 17 main restart
```

> **คำเตือนสำคัญ**: การ restart จะตัดการเชื่อมต่อทุก connection ที่เปิดอยู่ชั่วขณะ ต้องวางแผน maintenance window ให้เหมาะสมในระบบ production ห้าม restart กลางช่วง peak traffic โดยไม่แจ้งทีมที่เกี่ยวข้องล่วงหน้า

**ขั้นตอนที่ 3**: ตรวจสอบว่า preload สำเร็จแล้ว

```sql
SHOW shared_preload_libraries;
```

```
   shared_preload_libraries
--------------------------------
 pg_stat_statements
(1 row)
```

**ขั้นตอนที่ 4**: สร้าง extension ในแต่ละ database ที่ต้องการเก็บสถิติ

```sql
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
```

```
CREATE EXTENSION
```

```sql
\dx pg_stat_statements
```

```
                                    List of installed extensions
        Name        | Version |   Schema   |                     Description
---------------------+---------+------------+-------------------------------------------------------
 pg_stat_statements  | 1.11    | public     | track planning and execution statistics of all SQL...
(1 row)
```

### สำรวจโครงสร้างของ view

```sql
SELECT column_name, data_type
FROM information_schema.columns
WHERE table_name = 'pg_stat_statements'
ORDER BY ordinal_position
LIMIT 15;
```

```
       column_name       |        data_type
--------------------------+---------------------------
 userid                   | oid
 dbid                     | oid
 toplevel                 | boolean
 queryid                  | bigint
 query                    | text
 plans                    | bigint
 total_plan_time          | double precision
 calls                    | bigint
 total_exec_time          | double precision
 min_exec_time            | double precision
 max_exec_time            | double precision
 mean_exec_time           | double precision
 stddev_exec_time         | double precision
 rows                     | bigint
 shared_blks_hit          | bigint
 shared_blks_read         | bigint
(15 rows)
```

คอลัมน์สำคัญที่จะใช้บ่อยที่สุด:

- **`query`** — ข้อความ query (มี placeholder `$1`, `$2` แทนค่าคงที่ที่ถูก normalize ออก เพื่อรวม query ที่ต่างกันแค่ค่าพารามิเตอร์เข้าเป็น pattern เดียวกัน)
- **`calls`** — จำนวนครั้งที่ query pattern นี้ถูกเรียกทั้งหมด
- **`total_exec_time`** — เวลารวมทั้งหมดที่ query นี้ใช้ (มิลลิวินาที) — ตัวชี้วัดที่สำคัญที่สุด เพราะสะท้อน "ภาระรวม" ที่ query นี้สร้างให้ระบบ
- **`mean_exec_time`** — เวลาเฉลี่ยต่อครั้ง
- **`rows`** — จำนวนแถวรวมที่ query คืนค่าทั้งหมด
- **`shared_blks_hit`** / **`shared_blks_read`** — จำนวน buffer page ที่ hit จาก cache กับที่ต้องอ่านจาก disk จริง (ใช้ประเมิน cache efficiency)

### ทดลองสร้าง workload แล้วดูผล

```sql
-- รัน query จำลองหลายแบบ
SELECT * FROM customers WHERE country = 'Thailand';
SELECT * FROM customers WHERE country = 'Japan';
SELECT * FROM customers WHERE country = 'USA';
SELECT o.order_id, c.first_name, c.last_name, o.status
FROM orders o JOIN customers c USING (customer_id)
WHERE o.status = 'completed';
```

```sql
SELECT query, calls, round(total_exec_time::numeric, 2) AS total_ms
FROM pg_stat_statements
WHERE query ILIKE '%customers%'
ORDER BY calls DESC
LIMIT 5;
```

```
                             query                              | calls | total_ms
------------------------------------------------------------------+-------+----------
 SELECT * FROM customers WHERE country = $1                       |     3 |     1.24
 SELECT o.order_id, c.first_name, c.last_name, o.status +         |     1 |     0.89
 FROM orders o JOIN customers c USING (customer_id)               |       |
 WHERE o.status = $1                                              |       |
(2 rows)
```

สังเกตว่าคำสั่ง `SELECT * FROM customers WHERE country = 'Thailand'` และ `= 'Japan'` และ `= 'USA'` ถูก **normalize รวมเป็น pattern เดียวกัน** (`country = $1`) พร้อม `calls = 3` — นี่คือจุดแข็งสำคัญของ `pg_stat_statements` ที่ทำให้เห็นภาพรวมของ "รูปแบบ query" แทนที่จะเห็นทุก query เป็นรายการแยกกันจนวิเคราะห์ไม่ไหว

### รีเซ็ตสถิติ

เมื่อต้องการเริ่มเก็บสถิติใหม่ (เช่นหลัง deploy โค้ดเวอร์ชันใหม่ อยากดูผลกระทบแยกจากของเก่า):

```sql
SELECT pg_stat_statements_reset();
```

```
 pg_stat_statements_reset
---------------------------
 2026-04-10 09:00:00+07
(1 row)
```

---

## Step 567: ใช้ pg_stat_statements หา Query ที่ช้าที่สุดและถูกเรียกบ่อยที่สุด

เมื่อเปิดใช้งานและปล่อยให้ระบบรันจริงสักระยะ (เช่น 1 วันหรือ 1 สัปดาห์) เราจะมีข้อมูลมากพอสำหรับวิเคราะห์เชิงลึก ต่อไปนี้คือชุด query วิเคราะห์ที่ DBA ระดับมืออาชีพใช้เป็นประจำ

### 1. Query ที่กิน "เวลารวม" มากที่สุด (top offender ตัวจริง)

นี่คือคำถามที่สำคัญที่สุด เพราะ query ที่กินเวลารวมมากที่สุด คือตัวที่ ถ้า optimize สำเร็จ จะเห็นผลกระทบเชิงบวกต่อระบบโดยรวมชัดเจนที่สุด (ไม่ใช่แค่ query ที่ "ช้าต่อครั้ง" แต่ถูกเรียกน้อยจนไม่กระทบภาพรวม)

```sql
SELECT
    round(total_exec_time::numeric, 2) AS total_ms,
    calls,
    round(mean_exec_time::numeric, 2) AS avg_ms,
    round((100 * total_exec_time / sum(total_exec_time) OVER ())::numeric, 2) AS pct_of_total,
    query
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;
```

```
 total_ms  | calls | avg_ms | pct_of_total |                          query
-----------+-------+--------+--------------+---------------------------------------------------------
  48210.55 | 92104 |   0.52 |        38.20 | SELECT * FROM orders WHERE customer_id = $1
  22105.90 |   540 |  40.94 |        17.51 | SELECT p.*, c.category_name FROM products p +
           |       |        |              | JOIN categories c USING (category_id) ORDER BY p.unit_price
  15877.30 |  8102 |   1.96 |        12.58 | UPDATE orders SET status = $1 WHERE order_id = $2
   9012.44 |    12 | 751.04 |         7.14 | SELECT * FROM products WHERE product_name ILIKE $1
   6544.20 |  3021 |   2.17 |         5.18 | INSERT INTO orders (customer_id, status) VALUES ($1, $2)
(5 rows)
```

จากตัวอย่างนี้ query แรก (`SELECT * FROM orders WHERE customer_id = $1`) กินเวลารวมมากที่สุดถึง 38% ของทั้งระบบ แม้จะเร็วต่อครั้ง (เฉลี่ย 0.52ms) แต่เพราะถูกเรียกถึง 92,104 ครั้ง ผลรวมจึงมหาศาล — นี่คือตัวอย่างคลาสสิกของ **"death by a thousand cuts"** ที่การมองแค่ query เดี่ยว ๆ ด้วย `EXPLAIN` มองไม่เห็น แต่ `pg_stat_statements` เผยให้เห็นทันที

### 2. Query ที่ "ช้าต่อครั้ง" มากที่สุดโดยเฉลี่ย

เหมาะสำหรับหา query ที่ผู้ใช้น่าจะรู้สึกได้ว่า "หน้าเว็บโหลดช้า" แม้จะไม่ได้ถูกเรียกบ่อย:

```sql
SELECT
    round(mean_exec_time::numeric, 2) AS avg_ms,
    round(max_exec_time::numeric, 2) AS max_ms,
    calls,
    query
FROM pg_stat_statements
WHERE calls > 5   -- กรอง query ที่รันแค่ 1-2 ครั้งออก (อาจเป็น ad-hoc query ของ DBA เอง)
ORDER BY mean_exec_time DESC
LIMIT 10;
```

```
 avg_ms  | max_ms  | calls |                       query
---------+---------+-------+----------------------------------------------------
  751.04 | 2204.11 |    12 | SELECT * FROM products WHERE product_name ILIKE $1
   40.94 |   88.30 |   540 | SELECT p.*, c.category_name FROM products p +
         |         |       | JOIN categories c USING (category_id) ORDER BY ...
    2.17 |   15.02 |  3021 | INSERT INTO orders (customer_id, status) VALUES ...
(3 rows)
```

query แรกน่าสงสัยมาก — `ILIKE` แบบไม่มี anchor ที่หน้าข้อความมักหมายถึง sequential scan ทั้งตาราง แนวทางแก้ควรพิจารณาใช้ `pg_trgm` GIN index (ตามที่เรียนใน Step 565) แทน

### 3. Query ที่ถูกเรียกบ่อยที่สุด (เพื่อพิจารณา caching หรือ connection pooling)

```sql
SELECT calls, round(mean_exec_time::numeric, 3) AS avg_ms, query
FROM pg_stat_statements
ORDER BY calls DESC
LIMIT 10;
```

```
 calls | avg_ms |                    query
-------+--------+---------------------------------------------
 92104 |  0.524 | SELECT * FROM orders WHERE customer_id = $1
  8102 |  1.960 | UPDATE orders SET status = $1 WHERE order_id = $2
  3021 |  2.170 | INSERT INTO orders (customer_id, status) VALUES ($1, $2)
(3 rows)
```

query ที่ถูกเรียกบ่อยมาก ๆ (หลักหมื่น-แสนครั้ง) เป็นตัวเลือกดีสำหรับ:
- พิจารณาทำ **application-level caching** (เช่น Redis) เพื่อลดจำนวนครั้งที่ต้องยิงมาถึง database เลย
- ตรวจสอบว่ามี index รองรับครบถ้วนหรือไม่ (เพราะแม้แต่ query ที่ "ต่อครั้งเร็ว" ก็ยังคุ้มค่าที่จะบีบให้เร็วขึ้นอีกเมื่อคูณด้วยจำนวนครั้งมหาศาล)

### 4. หา query ที่มี cache hit ratio ต่ำ (ต้องอ่านจาก disk บ่อย)

```sql
SELECT
    query,
    calls,
    shared_blks_hit,
    shared_blks_read,
    round(
        100.0 * shared_blks_hit / NULLIF(shared_blks_hit + shared_blks_read, 0),
        2
    ) AS cache_hit_pct
FROM pg_stat_statements
WHERE shared_blks_hit + shared_blks_read > 0
ORDER BY cache_hit_pct ASC NULLS LAST
LIMIT 10;
```

```
                      query                          | calls | shared_blks_hit | shared_blks_read | cache_hit_pct
-------------------------------------------------------+-------+-----------------+-------------------+---------------
 SELECT * FROM products WHERE product_name ILIKE $1     |    12 |             450 |              8802 |          4.86
 SELECT p.*, c.category_name FROM products p JOIN ...    |   540 |           98200 |              1200 |         98.79
(2 rows)
```

`cache_hit_pct` ต่ำมาก (เช่นต่ำกว่า 90%) บ่งชี้ว่า query นั้นต้องเข้า disk I/O บ่อยผิดปกติ — อาจเพราะไม่มี index เหมาะสม, ตารางใหญ่เกิน `shared_buffers` ที่มี, หรือข้อมูลที่ query เข้าถึงกระจัดกระจายมากในดิสก์ (เช่นปัญหา random UUID PK ที่กล่าวถึงใน Step 564)

### 5. เชื่อมกับ EXPLAIN เพื่อวิเคราะห์ลึกขึ้น

เมื่อเจอ query ต้องสงสัยจาก `pg_stat_statements` แล้ว ขั้นตอนถัดไปคือนำ query text นั้น (แทนค่า placeholder ด้วยค่าจริง) ไปรัน `EXPLAIN (ANALYZE, BUFFERS)` ตามที่เรียนใน Part 044 เพื่อดู query plan โดยละเอียด:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM products WHERE product_name ILIKE '%mouse%';
```

```
 Seq Scan on products  (cost=0.00..2.25 rows=1 width=48)
                       (actual time=0.021..0.045 rows=1 loops=1)
   Filter: (product_name ~~* '%mouse%'::text)
   Rows Removed by Filter: 19
   Buffers: shared hit=1
 Planning Time: 0.089 ms
 Execution Time: 0.062 ms
(6 rows)
```

ในตารางเล็ก ๆ แบบตัวอย่างนี้ sequential scan ยังเร็วอยู่ แต่เมื่อตาราง `products` เติบโตถึงหลักแสน/ล้านแถวในระบบจริง query แบบนี้จะเป็นสาเหตุอันดับต้น ๆ ที่ปรากฏใน top ของ `pg_stat_statements` — นี่คือวงจรการทำงานจริงของ performance tuning: **`pg_stat_statements` บอกว่า "ควรดู query ไหน" ส่วน `EXPLAIN ANALYZE` บอกว่า "ทำไมมันถึงช้า"**

### 6. Dashboard สรุปภาพรวมแบบเดียวจบ

```sql
SELECT
    'Total unique queries' AS metric, count(*)::text AS value
FROM pg_stat_statements
UNION ALL
SELECT 'Total calls across all queries', sum(calls)::text
FROM pg_stat_statements
UNION ALL
SELECT 'Total execution time (sec)', round(sum(total_exec_time)/1000, 2)::text
FROM pg_stat_statements;
```

```
              metric               |   value
-------------------------------------+------------
 Total unique queries               | 47
 Total calls across all queries     | 105318
 Total execution time (sec)         | 126.45
(3 rows)
```

ควรรัน query เชิงสรุปเหล่านี้เป็นประจำ (เช่นทุกเช้าผ่าน dashboard หรือ cron job แจ้งเตือน) เพื่อจับความผิดปกติแต่เนิ่น ๆ ก่อนที่ query ตัวใดตัวหนึ่งจะกลายเป็นปัญหาใหญ่ในช่วง peak traffic

---

## Step 568: hstore — Key-Value Store ก่อนยุค JSONB

### hstore คืออะไร

`hstore` เป็น extension ที่เก่าแก่มาก (มีมาตั้งแต่ PostgreSQL 8.2) ให้ data type สำหรับเก็บคู่ key-value แบบ flat (ไม่ซ้อนกันเป็นชั้น ๆ) โดยทั้ง key และ value เป็น text ล้วน:

```sql
CREATE EXTENSION IF NOT EXISTS hstore;

SELECT 'brand => "Logitech", color => "black", wireless => "true"'::hstore;
```

```
                              hstore
--------------------------------------------------------------------
 "brand"=>"Logitech", "color"=>"black", "wireless"=>"true"
(1 row)
```

### ตัวอย่างการใช้งาน

```sql
ALTER TABLE products ADD COLUMN attributes hstore;

UPDATE products
SET attributes = 'brand=>"Logitech-clone", color=>"black", wireless=>"true"'
WHERE product_id = 1;

SELECT product_name, attributes -> 'brand' AS brand, attributes -> 'color' AS color
FROM products
WHERE product_id = 1;
```

```
    product_name    |     brand      | color
----------------------+-----------------+-------
 Wireless Mouse M1    | Logitech-clone  | black
(1 row)
```

ค้นหาสินค้าที่มี key `wireless` เท่ากับ `"true"`:

```sql
SELECT product_name FROM products WHERE attributes @> 'wireless=>"true"';
```

```
     product_name
----------------------
 Wireless Mouse M1
(1 row)
```

### ทำไมปัจจุบัน (2026) แนะนำ JSONB มากกว่า hstore เกือบทุกกรณี

ตั้งแต่ PostgreSQL 9.4 เปิดตัว **JSONB** (binary JSON) เป็นต้นมา `hstore` แทบไม่มีเหตุผลให้เลือกใช้ในโปรเจกต์ใหม่อีกต่อไป เพราะ:

| ประเด็น | hstore | JSONB |
|---|---|---|
| โครงสร้างข้อมูล | flat key-value เท่านั้น (value เป็น text/NULL อย่างเดียว) | ซ้อนกันได้หลายชั้น (nested object, array), รองรับ type ย่อย (number, boolean, null, string) |
| มาตรฐาน | เฉพาะของ PostgreSQL | มาตรฐาน JSON ที่ใช้กันทั่วโลก แปลงไป-มากับ application layer (JavaScript, Python, ฯลฯ) ได้ตรงไปตรงมา |
| Indexing | GIN index รองรับ | GIN index รองรับเช่นกัน (และดีกว่าในหลายกรณีด้วย `jsonb_path_ops`) |
| operator/ฟังก์ชันที่มีให้ | จำกัด | มีฟังก์ชันครบครัน (`jsonb_set`, `jsonb_path_query`, `->`, `->>`, `#>`, `@@` กับ jsonpath ฯลฯ) และมีการพัฒนาต่อเนื่องในทุกเวอร์ชันใหม่ |
| การใช้งานร่วมกับ API สมัยใหม่ | ต้องแปลงไป-มา | ใช้ได้ตรง ๆ กับ REST/GraphQL API ที่ทำงานบน JSON อยู่แล้ว |
| ทิศทางการพัฒนาของ PostgreSQL core | หยุดพัฒนาฟีเจอร์ใหม่มานาน | ยังคงได้รับการพัฒนาต่อเนื่อง (เช่น SQL/JSON standard functions ใน PostgreSQL 15-17) |

จะเห็นได้ว่า JSONB ทำทุกอย่างที่ `hstore` ทำได้ และทำได้ดีกว่าในเกือบทุกมิติ ยกเว้นกรณีเดียวที่ `hstore` ยังพอมีที่ยืนอยู่บ้างคือ **ระบบเก่าที่มีข้อมูลเก็บเป็น `hstore` อยู่แล้วเป็นจำนวนมาก** และการ migrate มาเป็น JSONB ยังไม่คุ้มค่าใช้จ่ายในระยะสั้น หรือกรณีที่ทีมต้องการ data model แบบ flat key-value ล้วน ๆ อย่างเคร่งครัด (ป้องกันนักพัฒนาใส่ nested object เข้ามาโดยไม่ตั้งใจ) แต่กรณีเช่นนี้พบได้น้อยลงเรื่อย ๆ ในระบบใหม่

**คำแนะนำสำหรับโปรเจกต์ใหม่**: ใช้ `JSONB` เป็นค่าเริ่มต้นเสมอสำหรับข้อมูลแบบ semi-structured (เช่นคอลัมน์ `attributes` ในตัวอย่างข้างบน ควรเป็น `JSONB` ไม่ใช่ `hstore`) — รายละเอียดเชิงลึกเรื่อง JSONB มีอยู่แล้วในบทก่อนหน้าของหลักสูตรนี้ (ดู Part ที่ว่าด้วย JSON/JSONB ในหมวด intermediate)

```sql
-- แนวทางที่แนะนำในปี 2026: ใช้ JSONB แทน hstore
ALTER TABLE products ADD COLUMN attributes_jsonb JSONB;

UPDATE products
SET attributes_jsonb = '{"brand": "Logitech-clone", "color": "black", "wireless": true, "specs": {"dpi": 1600, "buttons": 3}}'::jsonb
WHERE product_id = 1;

SELECT product_name, attributes_jsonb -> 'specs' -> 'dpi' AS dpi
FROM products WHERE product_id = 1;
```

```
    product_name    | dpi
----------------------+------
 Wireless Mouse M1    | 1600
(1 row)
```

สังเกตว่า JSONB รองรับ **nested object** (`specs.dpi`) และ **boolean จริง** (`true` ไม่ใช่ string `"true"`) ซึ่ง `hstore` ทำไม่ได้เลย — ลบคอลัมน์ทดลองทิ้งเพื่อความสะอาดของ schema:

```sql
ALTER TABLE products DROP COLUMN attributes;
ALTER TABLE products DROP COLUMN attributes_jsonb;
```

```
ALTER TABLE
ALTER TABLE
```

---

## Step 569: Extension อื่นที่ควรรู้จัก — btree_gin, btree_gist, pg_repack

### btree_gin

โดยปกติ GIN index (ที่ใช้กับ array, JSONB, full-text search, pg_trgm) ไม่รองรับ data type พื้นฐานอย่าง `INTEGER`, `TEXT`, `TIMESTAMPTZ` โดยตรง — `btree_gin` เพิ่ม operator class ที่ทำให้ GIN index รองรับ type พื้นฐานเหล่านี้ได้ด้วย ประโยชน์หลักคือการสร้าง **composite GIN index ที่รวมทั้งคอลัมน์ทั่วไปกับคอลัมน์ array/JSONB ไว้ใน index เดียวกัน**

```sql
CREATE EXTENSION IF NOT EXISTS btree_gin;

-- ตัวอย่างแนวคิด: ผสม category_id (integer ปกติ) กับ full-text search ในดัชนีเดียว
-- CREATE INDEX idx_products_mixed ON products USING GIN (category_id, to_tsvector('english', product_name));
```

หัวข้อนี้เกี่ยวโยงกับความรู้เรื่อง full-text search และ GIN index composite ที่จะกล่าวถึงลึกขึ้นในบทที่ว่าด้วย **advanced indexing patterns** ในหมวด world-class ของหลักสูตร

### btree_gist

คล้ายกับ `btree_gin` แต่สำหรับ **GiST index** — เพิ่ม operator class ให้ type พื้นฐานใช้กับ GiST ได้ ประโยชน์สำคัญที่สุดคือการทำ **exclusion constraint** ที่ผสมความเท่ากัน (equality) กับ range overlap เข้าด้วยกัน เช่น "ห้ามลูกค้าคนเดียวกันมีการจองห้องประชุมในช่วงเวลาที่ทับซ้อนกัน":

```sql
CREATE EXTENSION IF NOT EXISTS btree_gist;

-- ตัวอย่างแนวคิด (ไม่ได้สร้างตารางจริงในบทนี้):
-- CREATE TABLE booking (
--     room_id INTEGER,
--     during  TSRANGE,
--     EXCLUDE USING GIST (room_id WITH =, during WITH &&)
-- );
```

รูปแบบ `EXCLUDE USING GIST` นี้เป็นเทคนิคระดับสูงที่ใช้แก้ปัญหา "การจองที่ทับซ้อนกัน" ได้อย่างสวยงามในระดับ database constraint โดยไม่ต้องพึ่ง application logic เลย — จะกล่าวถึงโดยละเอียดในบทที่ว่าด้วย **range types และ exclusion constraints** ของหมวด advanced

### pg_repack

`pg_repack` เป็น extension (และมาพร้อม command-line tool) ที่ใช้ **บีบอัดพื้นที่ที่สูญเสียไปจาก table/index bloat** โดยไม่ต้อง lock ตารางนานเหมือนคำสั่ง `VACUUM FULL` แบบดั้งเดิม (ซึ่งจะ lock ตารางทั้งหมดตลอดกระบวนการ ทำให้ระบบ production หยุดชะงัก)

หลักการทำงานคร่าว ๆ ของ `pg_repack` คือสร้างตารางใหม่คู่ขนาน คัดลอกข้อมูล พร้อมติดตาม change ที่เกิดขึ้นระหว่างทางผ่าน trigger แล้วสลับตารางแบบรวดเร็วในตอนท้าย (คล้ายหลักการ zero-downtime migration)

```bash
# ติดตั้ง package ระดับ OS ก่อน (ไม่ใช่แค่ CREATE EXTENSION)
# apt install postgresql-17-repack

pg_repack --table=orders --host=localhost --dbname=ecommerce
```

การใช้งานจริง การตั้งค่า และข้อควรระวังของ `pg_repack` (เช่นเรื่อง disk space สำรองที่ต้องมีระหว่าง repack, ผลกระทบต่อ replication) จะกล่าวถึงโดยละเอียดในบทที่ว่าด้วย **table bloat และ maintenance operations ระดับ production** ในหมวด world-class ของหลักสูตรนี้ — บทนี้เพียงแค่แนะนำให้รู้จักชื่อและแนวคิดเบื้องต้นไว้ก่อน

---

## สรุปท้ายบท

ตารางสรุป extension ทั้งหมดที่กล่าวถึงในหลักสูตรนี้จนถึง Part 057:

| Extension | มาจากบท | ใช้ทำอะไร | ต้องแก้ postgresql.conf หรือไม่ |
|---|---|---|---|
| `plpgsql` | ติดตั้งมาให้เป็นค่าเริ่มต้น | ภาษาสำหรับเขียน function/trigger/procedure (Part 046-048) | ไม่ต้อง |
| `pg_trgm` | Part 042, 057 | trigram similarity search, fuzzy text matching, เร่งความเร็ว `LIKE`/`ILIKE` | ไม่ต้อง |
| `pgcrypto` | Part 057 | hash รหัสผ่านแบบ bcrypt (`crypt`, `gen_salt`), เข้ารหัสข้อมูล PII แบบสองทาง (`pgp_sym_encrypt/decrypt`) | ไม่ต้อง |
| `uuid-ossp` | Part 057 | สร้าง UUID (v1, v3, v4, v5) — ปัจจุบันส่วนใหญ่ใช้ built-in `gen_random_uuid()` แทนสำหรับ v4 | ไม่ต้อง |
| `pg_stat_statements` | Part 057 | เก็บสถิติการรัน query ทุกตัวในระบบ เพื่อวิเคราะห์ performance | **ต้อง** (`shared_preload_libraries` + restart) |
| `hstore` | Part 057 | key-value store แบบ flat รุ่นก่อน JSONB (ปัจจุบันแนะนำ JSONB แทนเกือบทุกกรณี) | ไม่ต้อง |
| `btree_gin` | Part 057 (เกริ่นนำ) | เพิ่ม operator class ให้ type พื้นฐานใช้กับ GIN index ได้ (composite index) | ไม่ต้อง |
| `btree_gist` | Part 057 (เกริ่นนำ) | เพิ่ม operator class ให้ type พื้นฐานใช้กับ GiST index ได้ (ใช้คู่กับ exclusion constraint) | ไม่ต้อง |
| `pg_repack` | Part 057 (เกริ่นนำ) | ลด table/index bloat โดยไม่ lock ตารางนานเหมือน `VACUUM FULL` | ต้องติดตั้ง package ระดับ OS เพิ่มเติม |

**หลักการเลือกใช้ extension โดยสรุป:**

1. **`pg_stat_statements` ควรเปิดในทุกระบบ production โดยไม่มีข้อยกเว้น** — เป็นข้อมูลพื้นฐานที่สุดสำหรับการดูแลระบบเชิงรุก
2. **`pgcrypto`** จำเป็นเสมอเมื่อระบบต้องเก็บรหัสผ่านหรือข้อมูลอ่อนไหว — อย่าใช้ MD5/SHA เก็บรหัสผ่านตรง ๆ เด็ดขาด
3. **UUID เป็น PK**: ใช้ `gen_random_uuid()` built-in เป็นค่าเริ่มต้น (ไม่ต้องพึ่ง `uuid-ossp`) และพิจารณาแพทเทิร์น "internal integer PK + public UUID" สำหรับระบบที่ต้องเปิดเผย ID ผ่าน API
4. **`pg_trgm`** เหมาะกับทุกระบบที่มีช่องค้นหาแบบ fuzzy/autocomplete
5. **`hstore`** แทบไม่มีเหตุผลให้เลือกใช้ในระบบใหม่แล้ว — ใช้ JSONB แทน
6. extension ขั้นสูงอื่น ๆ (`btree_gin`, `btree_gist`, `pg_repack`) ค่อยศึกษาเชิงลึกเมื่อถึงบทที่เกี่ยวข้องโดยตรง

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1

ตรวจสอบว่า extension `pgcrypto`, `pg_trgm`, `uuid-ossp`, `hstore` ติดตั้งอยู่ใน database ปัจจุบันหรือไม่ ด้วยคำสั่ง SQL (ไม่ใช้ `\dx`)

<details>
<summary>เฉลย</summary>

```sql
SELECT extname, extversion
FROM pg_extension
WHERE extname IN ('pgcrypto', 'pg_trgm', 'uuid-ossp', 'hstore')
ORDER BY extname;
```

```
 extname  | extversion
----------+------------
 hstore   | 1.8
 pg_trgm  | 1.6
 pgcrypto | 1.3
 uuid-ossp| 1.1
(4 rows)
```

</details>

### แบบฝึกหัดที่ 2

เขียนฟังก์ชัน `hash_password(p_plain TEXT)` ที่รับรหัสผ่านแบบข้อความธรรมดา แล้วคืนค่า bcrypt hash โดยใช้ cost factor 11

<details>
<summary>เฉลย</summary>

```sql
CREATE OR REPLACE FUNCTION hash_password(p_plain TEXT)
RETURNS TEXT
LANGUAGE sql
IMMUTABLE
AS $$
    SELECT crypt(p_plain, gen_salt('bf', 11));
$$;
```

```sql
SELECT hash_password('TestPass123');
```

```
                          hash_password
------------------------------------------------------------------
 $2a$11$8kR2nP0vXcQ5oWs9Tg1FeuZb3Nh6Vm2Do7Jc4Rk1Xw9Sp0Yt5Lb2Qa
(1 row)
```

> หมายเหตุ: ในทางเทคนิค `crypt()`/`gen_salt()` ไม่ได้เป็น `IMMUTABLE` จริง ๆ (เพราะสุ่ม salt ใหม่ทุกครั้ง ผลลัพธ์จึงต่างกันแม้ input เดิม) แต่ PostgreSQL อนุญาตให้ประกาศเช่นนี้ได้เพื่อความสะดวก ในทางปฏิบัติควรระวังไม่นำฟังก์ชันนี้ไปใช้ในบริบทที่พึ่งพาคุณสมบัติ IMMUTABLE จริงจัง เช่น expression index

</details>

### แบบฝึกหัดที่ 3

ใช้ `pgp_sym_encrypt` เข้ารหัสคอลัมน์ email ของลูกค้าคนที่ `customer_id = 7` ด้วย passphrase `'demo-key-2026'` แล้วถอดรหัสกลับมาดูผล

<details>
<summary>เฉลย</summary>

```sql
SELECT
    pgp_sym_encrypt(email, 'demo-key-2026') AS encrypted
FROM customers
WHERE customer_id = 7;
```

```sql
SELECT
    pgp_sym_decrypt(
        pgp_sym_encrypt(email, 'demo-key-2026'),
        'demo-key-2026'
    ) AS decrypted_email
FROM customers
WHERE customer_id = 7;
```

```
   decrypted_email
------------------------
 john.smith@example.com
(1 row)
```

</details>

### แบบฝึกหัดที่ 4

เพิ่ม unique index บนคอลัมน์ `customer_uuid` ของตาราง `customers` (ถ้ายังไม่มี) แล้วเขียน query ค้นหาลูกค้าด้วยค่า UUID แทนการใช้ `customer_id`

<details>
<summary>เฉลย</summary>

```sql
CREATE UNIQUE INDEX IF NOT EXISTS idx_customers_uuid
ON customers (customer_uuid);
```

```sql
SELECT customer_id, first_name, last_name
FROM customers
WHERE customer_uuid = (
    SELECT customer_uuid FROM customers WHERE customer_id = 3
);
```

```
 customer_id | first_name | last_name
-------------+------------+-----------
           3 | Anan       | Wongsa
(1 row)
```

</details>

### แบบฝึกหัดที่ 5

ใช้ `similarity()` จาก `pg_trgm` ค้นหาสินค้าที่ชื่อคล้ายกับคำว่า `"laptp"` (สะกดผิดโดยตั้งใจ) แล้วเรียงตามคะแนนความคล้ายจากมากไปน้อย

<details>
<summary>เฉลย</summary>

```sql
SELECT product_name, similarity(product_name, 'laptp') AS score
FROM products
WHERE product_name % 'laptp'
   OR similarity(product_name, 'laptp') > 0.1
ORDER BY score DESC;
```

```
       product_name        | score
----------------------------+-------
 Business Laptop Pro 14     |  0.20
 Gaming Laptop X15          |  0.19
(2 rows)
```

</details>

### แบบฝึกหัดที่ 6

สมมติว่าลืมเปิด `pg_stat_statements` ไว้ล่วงหน้า ให้เขียนขั้นตอนทั้งหมด (เป็นข้อความอธิบาย ไม่ใช่แค่ SQL) ที่ต้องทำเพื่อเปิดใช้งานให้สมบูรณ์

<details>
<summary>เฉลย</summary>

1. หาตำแหน่งไฟล์ config: `SHOW config_file;`
2. แก้ไขไฟล์ `postgresql.conf` เพิ่มบรรทัด `shared_preload_libraries = 'pg_stat_statements'` (ถ้ามีค่าอื่นอยู่แล้ว ให้เติมต่อด้วย comma ไม่ใช่เขียนทับ)
3. บันทึกไฟล์แล้ว **restart PostgreSQL server** ทั้งตัว (ไม่ใช่แค่ reload) เพราะ `shared_preload_libraries` เป็นพารามิเตอร์ระดับ `postmaster` ที่ reload อย่างเดียวไม่พอ
4. ตรวจสอบว่า preload สำเร็จด้วย `SHOW shared_preload_libraries;`
5. รัน `CREATE EXTENSION IF NOT EXISTS pg_stat_statements;` ในแต่ละ database ที่ต้องการเก็บสถิติ (extension ผูกกับ database ไม่ใช่ server ทั้งหมด)
6. ทดสอบด้วย `SELECT * FROM pg_stat_statements LIMIT 1;` ควรได้ผลลัพธ์โดยไม่มี error

</details>

### แบบฝึกหัดที่ 7

เขียน query หา 3 อันดับ query ที่กิน `total_exec_time` มากที่สุดจาก `pg_stat_statements` พร้อมแสดงเปอร์เซ็นต์ที่แต่ละ query คิดเป็นสัดส่วนของเวลารวมทั้งหมด

<details>
<summary>เฉลย</summary>

```sql
SELECT
    query,
    calls,
    round(total_exec_time::numeric, 2) AS total_ms,
    round((100 * total_exec_time / sum(total_exec_time) OVER ())::numeric, 2) AS pct_of_total
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 3;
```

ผลลัพธ์ตัวอย่าง (ตัวเลขจะต่างกันตาม workload จริงของแต่ละระบบ):

```
                          query                          | calls | total_ms | pct_of_total
-----------------------------------------------------------+-------+----------+---------------
 SELECT * FROM orders WHERE customer_id = $1                | 92104 | 48210.55 |         38.20
 SELECT p.*, c.category_name FROM products p JOIN ...        |   540 | 22105.90 |         17.51
 UPDATE orders SET status = $1 WHERE order_id = $2           |  8102 | 15877.30 |         12.58
(3 rows)
```

</details>

### แบบฝึกหัดที่ 8

อธิบายว่าทำไมคอลัมน์ที่เก็บผลลัพธ์จาก `pgp_sym_encrypt()` ต้องเป็นชนิด `BYTEA` ไม่ใช่ `TEXT` และจะเกิดอะไรขึ้นถ้าพยายามเก็บลงคอลัมน์ `TEXT` โดยไม่แปลงชนิด

<details>
<summary>เฉลย</summary>

`pgp_sym_encrypt()` คืนค่าเป็นข้อมูล binary ดิบ (encrypted bytes ตามมาตรฐาน OpenPGP) ซึ่งอาจมี byte sequence ที่ไม่ใช่อักขระข้อความที่ valid ในการเข้ารหัสอักขระ (encoding) ที่ database ใช้อยู่ (เช่น UTF-8) การพยายามเก็บ binary data แบบนี้ลงในคอลัมน์ `TEXT` โดยตรงจะทำให้ PostgreSQL แจ้ง error ทันที เนื่องจากคอลัมน์ `TEXT` คาดหวังว่าข้อมูลทั้งหมดต้องเป็น valid character encoding เสมอ (เช่น byte บางตัวไม่ใช่ valid UTF-8 sequence)

ถ้าจำเป็นต้องเก็บในรูปแบบข้อความจริง ๆ (เช่นต้องใส่ใน JSON field) ต้องแปลงเป็น base64 ก่อนด้วย `encode(pgp_sym_encrypt(...), 'base64')` แล้วตอนถอดรหัสต้อง `decode(...)` กลับเป็น `bytea` ก่อนส่งเข้า `pgp_sym_decrypt()`

```sql
SELECT encode(pgp_sym_encrypt('secret', 'key'), 'base64');
```

</details>

### แบบฝึกหัดที่ 9

เปรียบเทียบข้อดี-ข้อเสียของการใช้ `hstore` กับ `JSONB` สำหรับเก็บ attribute ของสินค้าที่มีโครงสร้างซับซ้อน เช่น `{"dimensions": {"width": 10, "height": 20}, "colors": ["red", "blue"]}`

<details>
<summary>เฉลย</summary>

`hstore` **ทำไม่ได้เลย** สำหรับโครงสร้างข้อมูลนี้ เพราะ `hstore` รองรับแค่คู่ key-value แบบ flat เท่านั้น (value เป็น string/NULL อย่างเดียว) ไม่รองรับ:
- nested object (`dimensions.width`, `dimensions.height`)
- array (`colors: ["red", "blue"]`)
- type ที่ไม่ใช่ string (number, boolean)

หากพยายามยัดข้อมูลนี้ลง `hstore` จะต้อง flatten โครงสร้างเอง (เช่นเก็บเป็น string คีย์ `"dimensions_width" => "10"` และแปลง array เป็น string คั่นด้วย comma) ซึ่งเสียคุณสมบัติของ structured data ไปเกือบหมด และ query กลับมาใช้งานยากมาก

`JSONB` รองรับโครงสร้างนี้ได้ตรงไปตรงมา 100% พร้อมมีฟังก์ชัน/operator ครบสำหรับ query เข้าไปในระดับลึก (`attributes -> 'dimensions' ->> 'width'`) และมี GIN index รองรับการค้นหาแบบ containment (`@>`) ได้ดีเช่นกัน จึงเป็นตัวเลือกที่เหมาะสมกว่าอย่างชัดเจนสำหรับกรณีนี้

</details>

### แบบฝึกหัดที่ 10 (แบบฝึกหัดรวม)

ระบบ e-commerce ของเราต้องการยกระดับความปลอดภัยและความสามารถโดยใช้ extension ทั้งหมดที่เรียนมาในบทนี้ ให้ทำภารกิจต่อไปนี้ครบทุกข้อในคำสั่งเดียวหรือชุดคำสั่งเดียวกัน:

1. เข้ารหัสรหัสผ่านของลูกค้าใหม่ที่ยังไม่มี `password_hash` (ถ้ามี) ด้วย bcrypt cost 10
2. สร้าง view ชื่อ `customer_public_profile` ที่ expose เฉพาะ `customer_uuid` (ไม่ใช่ `customer_id`), `first_name`, `last_name`, `country` — เพื่อไม่ให้ internal ID รั่วไหลออกไปสู่ภายนอก
3. เขียนฟังก์ชันค้นหาสินค้าแบบ fuzzy search ที่รับคำค้นและคืนสินค้าที่คล้ายกันเกิน threshold 0.2 เรียงตามความคล้าย
4. เขียน query จาก `pg_stat_statements` เพื่อหา query ที่เกี่ยวกับตาราง `products` ที่ถูกเรียกบ่อยที่สุด 3 อันดับแรก

<details>
<summary>เฉลย</summary>

**1. เข้ารหัสรหัสผ่านที่ยังไม่มี:**

```sql
UPDATE customers
SET password_hash = crypt('TempPass_' || customer_id || '!', gen_salt('bf', 10))
WHERE password_hash IS NULL;
```

```
UPDATE 0
```

(ในกรณีนี้ได้ 0 แถวเพราะเราเติม password_hash ให้ลูกค้าทุกคนไปแล้วใน Step 562 — แต่ pattern การเช็ค `WHERE password_hash IS NULL` นี้คือแนวทางที่ถูกต้องสำหรับ migration script ที่ปลอดภัย รันซ้ำได้โดยไม่กระทบข้อมูลที่ตั้งไว้แล้ว)

**2. สร้าง view เปิดเผยเฉพาะ UUID:**

```sql
CREATE OR REPLACE VIEW customer_public_profile AS
SELECT
    customer_uuid,
    first_name,
    last_name,
    country
FROM customers;
```

```sql
SELECT * FROM customer_public_profile LIMIT 3;
```

```
             customer_uuid             | first_name | last_name | country
----------------------------------------+------------+-----------+----------
 3d9f4a1e-7b2c-4e6d-9a1f-8c3b5d7e9f01     | Somchai     | Jaidee    | Thailand
 9a2c1f0e-4b8d-4a3c-b7e1-5d6f8a0c2e4b     | Suda        | Boonmee   | Thailand
 5e7f2a9c-1d3b-4c6e-8a0f-2b4d6e8f0a1c     | Anan        | Wongsa    | Thailand
(3 rows)
```

**3. ฟังก์ชัน fuzzy search สินค้า:**

```sql
CREATE OR REPLACE FUNCTION search_products_fuzzy(p_keyword TEXT)
RETURNS TABLE (product_id INTEGER, product_name VARCHAR, unit_price NUMERIC, score REAL)
LANGUAGE sql
STABLE
AS $$
    SELECT
        p.product_id,
        p.product_name,
        p.unit_price,
        similarity(p.product_name, p_keyword) AS score
    FROM products p
    WHERE similarity(p.product_name, p_keyword) > 0.2
    ORDER BY score DESC;
$$;
```

```sql
SELECT * FROM search_products_fuzzy('gaming laptp');
```

```
 product_id |    product_name    | unit_price | score
------------+---------------------+------------+-------
          5 | Gaming Laptop X15   |   45900.00 |  0.55
(1 row)
```

**4. Query ที่เกี่ยวกับ products ถูกเรียกบ่อยที่สุด:**

```sql
SELECT calls, round(mean_exec_time::numeric, 3) AS avg_ms, query
FROM pg_stat_statements
WHERE query ILIKE '%products%'
  AND query NOT ILIKE '%pg_stat_statements%'
ORDER BY calls DESC
LIMIT 3;
```

```
 calls | avg_ms |                          query
-------+--------+------------------------------------------------------
   540 | 40.940 | SELECT p.*, c.category_name FROM products p JOIN ...
    12 |751.040 | SELECT * FROM products WHERE product_name ILIKE $1
     3 |  0.180 | SELECT * FROM search_products_fuzzy($1)
(3 rows)
```

ภารกิจนี้แสดงให้เห็นว่า extension ทั้งสี่ตัว (`pgcrypto`, UUID built-in, `pg_trgm`, `pg_stat_statements`) ทำงานประสานกันได้จริงในระบบเดียว: ปกป้องรหัสผ่าน, ปกปิด internal ID, เพิ่มความสามารถค้นหา, และเฝ้าระวังประสิทธิภาพอย่างต่อเนื่อง — ครบทุกมิติของความปลอดภัยและคุณภาพระบบที่ระบบ production ระดับ world-class ควรมี

</details>

---

## บทถัดไป

extension เป็นเพียงเครื่องมือเสริมที่ทำงานอยู่ **เหนือ** กลไกพื้นฐานของ PostgreSQL แต่กลไกที่แท้จริงซึ่งกำหนดว่าฐานข้อมูลจะทำงานถูกต้องภายใต้ transaction พร้อมกันหลายตัวได้อย่างไรนั้น คือระบบที่เรียกว่า **MVCC (Multi-Version Concurrency Control)** ซึ่งเป็นหัวใจของ PostgreSQL มาตั้งแต่ต้น และเป็นสิ่งที่ทุก DBA/engineer ระดับ world-class ต้องเข้าใจอย่างลึกซึ้ง

ไปต่อกันที่ [Part 058: MVCC Deep Dive](./part-058-mvcc-deep-dive.md)
