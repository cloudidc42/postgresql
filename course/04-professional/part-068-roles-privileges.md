# Part 068: Security — Roles, Privileges, GRANT/REVOKE

> **หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับมืออาชีพ | Part 068**

---

## เป้าหมายการเรียนรู้

เมื่อเรียนจบบทนี้ ผู้เรียนจะสามารถ:

1. อธิบายได้ว่า **Role** ใน PostgreSQL คืออะไร และทำไมจึงไม่มีความแตกต่างที่แท้จริงระหว่าง "user" กับ "role"
2. สร้างและจัดการ Role ด้วย `CREATE ROLE` พร้อม attribute สำคัญ เช่น `LOGIN`, `SUPERUSER`, `CREATEDB`, `CREATEROLE`, `PASSWORD`, `VALID UNTIL`
3. ใช้ **Role Membership** เพื่อสร้างโครงสร้าง group role และมอบสิทธิ์ผ่านการเป็นสมาชิกกลุ่ม
4. ให้และเพิกถอนสิทธิ์ระดับตาราง (`GRANT`/`REVOKE` สำหรับ `SELECT`, `INSERT`, `UPDATE`, `DELETE`)
5. จำกัดสิทธิ์ให้ละเอียดถึงระดับ **column** ด้วย `GRANT SELECT (columns) ON table`
6. จัดการสิทธิ์ระดับ **Schema** และ **Database** (`USAGE`, `CREATE`, `CONNECT`)
7. ใช้ `ALTER DEFAULT PRIVILEGES` เพื่อกำหนดสิทธิ์ล่วงหน้าสำหรับออบเจ็กต์ที่จะถูกสร้างขึ้นในอนาคต
8. เข้าใจ **PUBLIC role** สิทธิ์เริ่มต้นที่ทุก role มีร่วมกัน และความเสี่ยงด้านความปลอดภัยที่มากับมัน รวมถึงการเปลี่ยนแปลงใน PostgreSQL 15+
9. ออกแบบระบบ role ตามหลัก **Least Privilege** สำหรับแอปพลิเคชันจริง (read-only, read-write, admin)
10. สร้างระบบ role แบบครบวงจรสำหรับทีมพัฒนา ระบบ backend และทีม analyst ของระบบ e-commerce

---

## เตรียมข้อมูล

เราจะใช้สคีมาอีคอมเมิร์ซพื้นฐานเดิมที่ใช้ตลอดหลักสูตรระดับมืออาชีพ ให้รันคำสั่งต่อไปนี้ในฐานข้อมูลทดสอบ (แนะนำให้สร้างฐานข้อมูลใหม่ชื่อ `ecommerce_security` เพื่อไม่ให้ปนกับบทอื่น)

```sql
-- สร้างฐานข้อมูลสำหรับบทนี้โดยเฉพาะ (รันในฐานะ superuser เช่น postgres)
CREATE DATABASE ecommerce_security;
\c ecommerce_security

-- ตารางหมวดหมู่สินค้า
CREATE TABLE categories (
    category_id   SERIAL PRIMARY KEY,
    category_name VARCHAR(100) NOT NULL
);

-- ตารางผู้จัดจำหน่าย
CREATE TABLE suppliers (
    supplier_id   SERIAL PRIMARY KEY,
    supplier_name VARCHAR(150) NOT NULL
);

-- ตารางสินค้า
CREATE TABLE products (
    product_id   SERIAL PRIMARY KEY,
    product_name VARCHAR(150) NOT NULL,
    category_id  INTEGER REFERENCES categories(category_id),
    unit_price   NUMERIC(10,2) NOT NULL
);

-- ตารางลูกค้า
CREATE TABLE customers (
    customer_id SERIAL PRIMARY KEY,
    first_name  VARCHAR(60),
    last_name   VARCHAR(60),
    email       VARCHAR(150)
);

-- ตารางคำสั่งซื้อ
CREATE TABLE orders (
    order_id    SERIAL PRIMARY KEY,
    customer_id INTEGER REFERENCES customers(customer_id),
    order_date  TIMESTAMPTZ DEFAULT now(),
    status      VARCHAR(20)
);

-- ตารางรายการสินค้าในคำสั่งซื้อ
CREATE TABLE order_items (
    order_item_id SERIAL PRIMARY KEY,
    order_id      INTEGER REFERENCES orders(order_id),
    product_id    INTEGER REFERENCES products(product_id),
    quantity      INTEGER
);

-- ข้อมูลตัวอย่าง: หมวดหมู่
INSERT INTO categories (category_name) VALUES
('อิเล็กทรอนิกส์'), ('เครื่องใช้ในบ้าน'), ('เสื้อผ้า'), ('หนังสือ'), ('ของเล่น');

-- ข้อมูลตัวอย่าง: ผู้จัดจำหน่าย
INSERT INTO suppliers (supplier_name) VALUES
('บริษัท ไทยเทค จำกัด'), ('สยามโฮมแวร์'), ('แฟชั่นดีไซน์ จำกัด'), ('สำนักพิมพ์ปัญญาชน');

-- ข้อมูลตัวอย่าง: สินค้า
INSERT INTO products (product_name, category_id, unit_price) VALUES
('หูฟังไร้สาย รุ่น X1', 1, 1290.00),
('เตารีดไอน้ำ', 2, 890.00),
('เสื้อยืดผ้าฝ้าย', 3, 259.00),
('หนังสือ PostgreSQL ฉบับสมบูรณ์', 4, 450.00),
('ตุ๊กตาหมีเท็ดดี้', 5, 350.00),
('โน้ตบุ๊ก 14 นิ้ว', 1, 18990.00),
('พัดลมตั้งพื้น', 2, 690.00);

-- ข้อมูลตัวอย่าง: ลูกค้า
INSERT INTO customers (first_name, last_name, email) VALUES
('สมชาย', 'ใจดี', 'somchai.j@example.com'),
('สมหญิง', 'รักเรียน', 'somying.r@example.com'),
('วิชัย', 'มั่งมี', 'wichai.m@example.com'),
('ปรานี', 'ศรีสุข', 'pranee.s@example.com');

-- ข้อมูลตัวอย่าง: คำสั่งซื้อ
INSERT INTO orders (customer_id, order_date, status) VALUES
(1, now() - interval '10 days', 'completed'),
(2, now() - interval '7 days', 'completed'),
(3, now() - interval '3 days', 'pending'),
(1, now() - interval '1 day', 'pending');

-- ข้อมูลตัวอย่าง: รายการสินค้าในคำสั่งซื้อ
INSERT INTO order_items (order_id, product_id, quantity) VALUES
(1, 1, 2), (1, 4, 1),
(2, 2, 1), (2, 6, 1),
(3, 3, 3),
(4, 5, 2), (4, 7, 1);
```

> **หมายเหตุด้านความปลอดภัย:** บทนี้ต้องรันคำสั่งบางส่วนในฐานะ **superuser** (เช่น `CREATE ROLE`, `CREATE DATABASE`) เพราะ role ทั่วไปไม่มีสิทธิ์สร้าง role อื่นโดยปริยาย แนะนำให้เปิด `psql` ด้วยผู้ใช้ `postgres` ตลอดบทนี้ แล้วสลับ role ด้วยคำสั่ง `SET ROLE` หรือเปิด session ใหม่ด้วย `\c ecommerce_security role_name` เพื่อทดสอบสิทธิ์จริง

---

## Step 671: Role คืออะไรใน PostgreSQL

### แนวคิดหลัก

ใน PostgreSQL **ไม่มีสิ่งที่เรียกว่า "user" แยกออกมาจริง ๆ** — ทุกสิ่งคือ **role** ทั้งหมด คำว่า "user" เป็นเพียงคำพ้องความหมาย (synonym) ในระดับภาษาที่ใช้เรียก role ที่มี attribute `LOGIN` เท่านั้น

พูดให้ชัดเจน:

- **Role** = วัตถุ (object) ในระบบฐานข้อมูลที่เป็นตัวแทนของ "ผู้ได้รับสิทธิ์" ไม่ว่าจะเป็นคนจริง แอปพลิเคชัน หรือกลุ่มของสิทธิ์
- **User** = role ที่มี attribute `LOGIN` (สามารถเชื่อมต่อเข้าฐานข้อมูลได้)
- **Group** = role ที่ไม่มี `LOGIN` (ใช้เป็น "ถัง" รวมสิทธิ์ แล้วให้ role อื่นเป็นสมาชิก)

ในเวอร์ชันเก่าของ PostgreSQL (ก่อน 8.1) มีคำสั่ง `CREATE USER` และ `CREATE GROUP` แยกกันจริง ๆ แต่ตั้งแต่ PostgreSQL 8.1 เป็นต้นมา ทั้งสองถูกรวมเป็นแนวคิดเดียวคือ **role** โดย:

```sql
CREATE USER foo;   -- เทียบเท่ากับ CREATE ROLE foo WITH LOGIN;
CREATE GROUP bar;  -- เทียบเท่ากับ CREATE ROLE bar WITH NOLOGIN;
```

คำสั่ง `CREATE USER` ยังคงอยู่เพื่อความเข้ากันได้ย้อนหลัง (backward compatibility) แต่ในทางปฏิบัติของมืออาชีพ ควรใช้ `CREATE ROLE` เสมอ แล้วระบุ attribute ที่ต้องการอย่างชัดเจน เพราะทำให้โค้ดสื่อความหมายตรงไปตรงมาว่า role นี้ตั้งใจให้ทำหน้าที่อะไร

### ตรวจสอบ role ที่มีอยู่ในระบบ

```sql
-- ดูรายชื่อ role ทั้งหมดในคลัสเตอร์ (role เป็นระดับ cluster ไม่ใช่ระดับ database)
\du
```

ผลลัพธ์ตัวอย่างบนเครื่องที่เพิ่งติดตั้งใหม่:

```
                                   List of roles
 Role name |                         Attributes                         | Member of
-----------+------------------------------------------------------------+-----------
 postgres  | Superuser, Create role, Create DB, Replication, Bypass RLS | {}
```

หรือใช้ SQL ล้วน ๆ ผ่าน system catalog `pg_roles`:

```sql
SELECT rolname, rolsuper, rolcreaterole, rolcreatedb, rolcanlogin, rolreplication
FROM pg_roles
ORDER BY rolname;
```

```
 rolname  | rolsuper | rolcreaterole | rolcreatedb | rolcanlogin | rolreplication
----------+----------+---------------+-------------+-------------+----------------
 postgres | t        | t             | t           | t           | t
(1 row)
```

### จุดสำคัญที่ต้องจำ

| ประเด็น | รายละเอียด |
|---|---|
| ขอบเขตของ role | Role เป็นออบเจ็กต์ระดับ **cluster** (ทั้งเซิร์ฟเวอร์) ไม่ใช่ระดับ database เดียว role เดียวกันสามารถเข้าถึงหลายฐานข้อมูลในคลัสเตอร์เดียวกันได้ |
| การเก็บข้อมูล | ข้อมูล role เก็บอยู่ใน catalog `pg_authid` (มี password hash) และ view `pg_roles` (ไม่แสดง password) |
| user = role + LOGIN | ทุกครั้งที่พูดถึง "user" ในเอกสาร PostgreSQL ให้เข้าใจว่าหมายถึง role ที่ล็อกอินได้ |
| group = role ไม่มี LOGIN | ใช้เป็นกลไกจัดกลุ่มสิทธิ์ (จะอธิบายละเอียดใน Step 673) |

---

## Step 672: CREATE ROLE — attribute สำคัญ

### รูปแบบคำสั่ง

```sql
CREATE ROLE role_name [ WITH ] [ option [ ... ] ]
```

โดย `option` ที่สำคัญที่สุดมีดังนี้:

| Attribute | ความหมาย | ค่า default |
|---|---|---|
| `LOGIN` / `NOLOGIN` | อนุญาต/ไม่อนุญาตให้เชื่อมต่อฐานข้อมูล (เป็น "user" หรือไม่) | `NOLOGIN` |
| `SUPERUSER` / `NOSUPERUSER` | ข้ามการตรวจสอบสิทธิ์ทั้งหมด (อันตรายมาก) | `NOSUPERUSER` |
| `CREATEDB` / `NOCREATEDB` | สร้างฐานข้อมูลใหม่ได้ | `NOCREATEDB` |
| `CREATEROLE` / `NOCREATEROLE` | สร้าง/แก้ไข/ลบ role อื่นได้ | `NOCREATEROLE` |
| `PASSWORD 'string'` | กำหนดรหัสผ่าน (ใช้ `NULL` เพื่อลบรหัสผ่าน) | ไม่มี |
| `VALID UNTIL 'timestamp'` | วันหมดอายุของรหัสผ่าน | ไม่มีวันหมดอายุ |
| `CONNECTION LIMIT n` | จำกัดจำนวน connection พร้อมกัน (`-1` = ไม่จำกัด) | `-1` |
| `INHERIT` / `NOINHERIT` | รับสิทธิ์จาก role ที่เป็นสมาชิกโดยอัตโนมัติหรือไม่ | `INHERIT` |
| `REPLICATION` / `NOREPLICATION` | ใช้สำหรับ streaming replication | `NOREPLICATION` |
| `BYPASSRLS` / `NOBYPASSRLS` | ข้าม Row-Level Security ทั้งหมด | `NOBYPASSRLS` |

### ตัวอย่างการสร้าง role แบบต่าง ๆ

```sql
-- 1) สร้าง role สำหรับผู้ใช้ทั่วไปที่ล็อกอินได้ พร้อมรหัสผ่าน
CREATE ROLE app_developer WITH LOGIN PASSWORD 'Dev#2026Secure!' CONNECTION LIMIT 5;

-- 2) สร้าง role สำหรับแอปพลิเคชัน backend (มักตั้งชื่อเป็น service account)
CREATE ROLE ecommerce_api WITH LOGIN PASSWORD 'Api#2026Secure!';

-- 3) สร้าง role ที่มีวันหมดอายุของรหัสผ่าน (เหมาะกับพนักงานชั่วคราว/นักศึกษาฝึกงาน)
CREATE ROLE intern_analyst WITH LOGIN PASSWORD 'Intern#Temp01'
    VALID UNTIL '2026-12-31 23:59:59+07';

-- 4) สร้าง role ที่สร้าง database ได้เอง (เหมาะกับ DBA รุ่นเยาว์)
CREATE ROLE db_admin_junior WITH LOGIN PASSWORD 'DbaJr#2026' CREATEDB;

-- 5) สร้าง role ระดับ superuser (ใช้เท่าที่จำเป็นจริง ๆ เท่านั้น!)
CREATE ROLE emergency_superuser WITH LOGIN PASSWORD 'Emrg#UseRarely' SUPERUSER;
```

ตรวจสอบผลลัพธ์:

```sql
\du
```

```
                                        List of roles
      Role name       |                         Attributes                         | Member of
-----------------------+------------------------------------------------------------+-----------
 app_developer         | 5 connections                                               | {}
 db_admin_junior       | Create DB                                                   | {}
 ecommerce_api         |                                                              | {}
 emergency_superuser   | Superuser                                                   | {}
 intern_analyst        | Password valid until 2026-12-31 23:59:59+07                | {}
 postgres              | Superuser, Create role, Create DB, Replication, Bypass RLS  | {}
```

### CREATE ROLE เทียบกับ ALTER ROLE

หากต้องการแก้ไข attribute ของ role ที่มีอยู่แล้ว ใช้ `ALTER ROLE`:

```sql
-- เปลี่ยนรหัสผ่าน
ALTER ROLE app_developer WITH PASSWORD 'NewDev#2026Pass!';

-- ยกเลิกสิทธิ์ CREATEDB
ALTER ROLE db_admin_junior WITH NOCREATEDB;

-- เพิ่มวันหมดอายุ
ALTER ROLE app_developer VALID UNTIL '2027-06-30';

-- ลบ password ออก (role จะล็อกอินด้วยรหัสผ่านไม่ได้อีก แต่ยัง LOGIN ผ่านวิธีอื่น เช่น peer/cert ได้)
ALTER ROLE app_developer WITH PASSWORD NULL;
```

### ลบ role

```sql
DROP ROLE IF EXISTS intern_analyst;
```

> **ข้อควรระวัง:** จะลบ role ไม่ได้ถ้า role นั้นยังเป็นเจ้าของออบเจ็กต์ (table, database, schema) หรือมีสิทธิ์ค้างอยู่ในระบบ ต้อง `REASSIGN OWNED BY` และ `DROP OWNED BY` ก่อนเสมอ (เดี๋ยวจะกล่าวถึงในหัวข้อ Least Privilege)

### Best practice ด้านความปลอดภัยของรหัสผ่าน

- ใช้ `password_encryption = scram-sha-256` (ค่า default ตั้งแต่ PostgreSQL 14) แทน `md5` ที่ล้าสมัย
- ตรวจสอบได้ด้วย: `SHOW password_encryption;`
- อย่าฝังรหัสผ่านลงในสคริปต์ deploy โดยตรง — ใช้ secret manager หรือ environment variable แล้วค่อย interpolate ตอนรัน
- ตั้ง `VALID UNTIL` สำหรับบัญชีชั่วคราวเสมอ

---

## Step 673: Role Membership — GRANT role_a TO role_b

### แนวคิด Group Role

จุดแข็งที่สุดของระบบ role ใน PostgreSQL คือ role หนึ่งสามารถเป็น **สมาชิก** ของ role อื่นได้ เมื่อ role A เป็นสมาชิกของ role B แล้ว role A จะ **สืบทอด (inherit)** สิทธิ์ทั้งหมดที่ role B มี (ถ้า role A มี attribute `INHERIT` ซึ่งเป็นค่า default)

รูปแบบคำสั่ง:

```sql
GRANT group_role TO member_role [WITH ADMIN OPTION];
REVOKE group_role FROM member_role;
```

แนวทางที่มืออาชีพนิยมใช้คือสร้าง "group role" ที่ **ไม่มี LOGIN** ขึ้นมาเป็น "ถังสิทธิ์" แล้วค่อยให้ role ที่ login ได้จริง (คน/แอป) เข้ามาเป็นสมาชิก วิธีนี้ทำให้จัดการสิทธิ์ได้ง่ายกว่าการ GRANT สิทธิ์ตรงให้กับ role รายบุคคลทีละคน

### ตัวอย่างจริง: สร้าง group role และให้สมาชิก

```sql
-- 1) สร้าง group role (ไม่มี LOGIN) สำหรับทีมที่อ่านข้อมูลได้อย่างเดียว
CREATE ROLE readonly_group NOLOGIN;

-- 2) สร้าง group role สำหรับทีมที่อ่าน-เขียนข้อมูลได้
CREATE ROLE readwrite_group NOLOGIN;

-- 3) สร้าง group role สำหรับผู้ดูแลระบบ
CREATE ROLE admin_group NOLOGIN;

-- 4) สร้างผู้ใช้จริงที่ login ได้
CREATE ROLE analyst_narin WITH LOGIN PASSWORD 'Narin#2026Pass';
CREATE ROLE dev_supaporn WITH LOGIN PASSWORD 'Supaporn#2026Pass';
CREATE ROLE dba_kittipong WITH LOGIN PASSWORD 'Kittipong#2026Pass';

-- 5) เพิ่มสมาชิกให้แต่ละกลุ่ม
GRANT readonly_group TO analyst_narin;
GRANT readwrite_group TO dev_supaporn;
GRANT admin_group TO dba_kittipong;
```

ตรวจสอบด้วย `\du`:

```
\du
```

```
                                        List of roles
      Role name       |               Attributes                | Member of
-----------------------+------------------------------------------+---------------------
 admin_group           | Cannot login                              | {}
 analyst_narin         |                                            | {readonly_group}
 dba_kittipong         |                                            | {admin_group}
 dev_supaporn          |                                            | {readwrite_group}
 readonly_group        | Cannot login                              | {}
 readwrite_group       | Cannot login                              | {}
```

### ตรวจสอบความเป็นสมาชิกด้วย SQL

```sql
SELECT
    m.rolname   AS member_role,
    g.rolname   AS group_role,
    am.admin_option
FROM pg_auth_members am
JOIN pg_roles m ON am.member = m.oid
JOIN pg_roles g ON am.roleid = g.oid
ORDER BY g.rolname, m.rolname;
```

```
  member_role  |   group_role    | admin_option
----------------+-----------------+--------------
 analyst_narin  | readonly_group  | f
 dba_kittipong  | admin_group     | f
 dev_supaporn   | readwrite_group | f
```

### WITH ADMIN OPTION

หาก role B ได้รับ `WITH ADMIN OPTION` บน role A แปลว่า B สามารถ `GRANT`/`REVOKE` role A ให้กับ role อื่นต่อได้ (เหมือนมีสิทธิ์บริหารจัดการสมาชิกของกลุ่มนั้น)

```sql
-- ให้ dba_kittipong สามารถเพิ่ม/ลบสมาชิกของ admin_group ได้เอง
GRANT admin_group TO dba_kittipong WITH ADMIN OPTION;
```

### role หลายชั้น (nested membership)

Role membership สามารถซ้อนกันหลายชั้นได้ เช่น:

```sql
CREATE ROLE senior_dev NOLOGIN;
GRANT readwrite_group TO senior_dev;   -- senior_dev สืบทอดสิทธิ์จาก readwrite_group
GRANT senior_dev TO dev_supaporn;      -- dev_supaporn สืบทอดสิทธิ์จาก senior_dev อีกทอดหนึ่ง
```

ผลคือ `dev_supaporn` จะได้สิทธิ์ของ `readwrite_group` ทั้งทางตรง (จาก Step ก่อนหน้า) และทางอ้อมผ่าน `senior_dev`

### INHERIT vs NOINHERIT

ค่า default ของ role คือ `INHERIT` ซึ่งหมายความว่าสิทธิ์ของกลุ่มที่เป็นสมาชิกจะถูกนำมาใช้ได้ทันทีโดยไม่ต้องทำอะไรเพิ่ม แต่ถ้า role ถูกสร้างด้วย `NOINHERIT` จะต้องเรียก `SET ROLE` เพื่อ "สวมสิทธิ์" ของกลุ่มนั้นก่อนจึงจะใช้สิทธิ์ได้:

```sql
CREATE ROLE cautious_dev WITH LOGIN PASSWORD 'Cautious#2026' NOINHERIT;
GRANT readwrite_group TO cautious_dev;

-- ตอนนี้ cautious_dev ยังใช้สิทธิ์ของ readwrite_group ไม่ได้โดยอัตโนมัติ
-- ต้องสั่ง SET ROLE ก่อน:
SET ROLE readwrite_group;
-- ทำงานที่ต้องใช้สิทธิ์ readwrite_group ...
RESET ROLE;  -- กลับสู่สิทธิ์เดิมของ cautious_dev
```

รูปแบบ `NOINHERIT` มีประโยชน์เมื่อคุณต้องการบังคับให้ผู้ใช้ "ยืนยันความตั้งใจ" ก่อนใช้สิทธิ์ระดับสูง คล้ายกับแนวคิด `sudo` ใน Linux

---

## Step 674: Privilege บนตาราง — GRANT/REVOKE SELECT, INSERT, UPDATE, DELETE

### รูปแบบคำสั่ง

```sql
GRANT { SELECT | INSERT | UPDATE | DELETE | TRUNCATE | REFERENCES | TRIGGER | ALL [PRIVILEGES] }
    [, ...] ON { table_name [, ...] | ALL TABLES IN SCHEMA schema_name }
    TO role_name [, ...];

REVOKE { SELECT | INSERT | UPDATE | DELETE | ... } [, ...]
    ON { table_name [, ...] | ALL TABLES IN SCHEMA schema_name }
    FROM role_name [, ...];
```

### ให้สิทธิ์อ่านอย่างเดียวแก่กลุ่ม analyst

```sql
-- ให้สิทธิ์อ่าน (SELECT) บนทุกตารางในสคีมาอีคอมเมิร์ซแก่กลุ่ม readonly_group
GRANT SELECT ON categories, suppliers, products, customers, orders, order_items
    TO readonly_group;
```

### ให้สิทธิ์อ่าน-เขียนแก่กลุ่มพัฒนา

```sql
-- readwrite_group ต้องอ่าน/เพิ่ม/แก้ไข/ลบข้อมูลได้ (แต่ยังไม่ให้ TRUNCATE เพราะอันตรายเกินไป)
GRANT SELECT, INSERT, UPDATE, DELETE
    ON categories, suppliers, products, customers, orders, order_items
    TO readwrite_group;
```

### ให้สิทธิ์เต็มแก่ admin

```sql
-- admin_group ได้สิทธิ์ทั้งหมดรวมถึง TRUNCATE และ REFERENCES
GRANT ALL PRIVILEGES
    ON categories, suppliers, products, customers, orders, order_items
    TO admin_group;
```

### ทดสอบสิทธิ์จริงด้วยการสลับ role

เปิด session ใหม่แล้วล็อกอินเป็น `analyst_narin` (สมาชิกของ `readonly_group`) หรือใช้ `SET ROLE` ภายใน session ของ superuser เพื่อทดสอบ:

```sql
-- (จำลองการทดสอบในฐานะ superuser ด้วย SET ROLE)
SET ROLE analyst_narin;

SELECT * FROM products LIMIT 3;
```

```
 product_id |      product_name       | category_id | unit_price
------------+--------------------------+-------------+------------
          1 | หูฟังไร้สาย รุ่น X1      |           1 |    1290.00
          2 | เตารีดไอน้ำ              |           2 |     890.00
          3 | เสื้อยืดผ้าฝ้าย          |           3 |     259.00
```

ลองเพิ่มข้อมูล (ต้องถูกปฏิเสธเพราะ `analyst_narin` ไม่มีสิทธิ์ `INSERT`):

```sql
INSERT INTO products (product_name, category_id, unit_price)
VALUES ('สินค้าทดสอบ', 1, 100.00);
```

```
ERROR:  permission denied for table products
```

กลับมาเป็น superuser และสลับไปเป็น `dev_supaporn`:

```sql
RESET ROLE;
SET ROLE dev_supaporn;

INSERT INTO products (product_name, category_id, unit_price)
VALUES ('สินค้าทดสอบจาก dev', 1, 199.00);
```

```
INSERT 0 1
```

คำสั่งนี้สำเร็จเพราะ `dev_supaporn` เป็นสมาชิกของ `readwrite_group` ที่มีสิทธิ์ `INSERT`

```sql
RESET ROLE;
```

### REVOKE — เพิกถอนสิทธิ์

```sql
-- ตัวอย่าง: เพิกถอนสิทธิ์ DELETE จาก readwrite_group เพราะทีมพัฒนาไม่ควรลบข้อมูลจริงโดยตรง
REVOKE DELETE ON orders, order_items FROM readwrite_group;
```

ทดสอบอีกครั้ง:

```sql
SET ROLE dev_supaporn;
DELETE FROM orders WHERE order_id = 999;  -- id ที่ไม่มีอยู่จริง แต่ทดสอบสิทธิ์
```

```
ERROR:  permission denied for table orders
```

```sql
RESET ROLE;
```

### ตรวจสอบสิทธิ์ปัจจุบันด้วย information_schema

```sql
SELECT grantee, table_name, privilege_type
FROM information_schema.role_table_grants
WHERE table_schema = 'public'
  AND grantee IN ('readonly_group', 'readwrite_group', 'admin_group')
ORDER BY grantee, table_name, privilege_type;
```

ผลลัพธ์ตัวอย่าง (บางส่วน):

```
     grantee      | table_name  | privilege_type
-------------------+-------------+-----------------
 admin_group       | categories  | DELETE
 admin_group       | categories  | INSERT
 admin_group       | categories  | REFERENCES
 admin_group       | categories  | SELECT
 admin_group       | categories  | TRIGGER
 admin_group       | categories  | TRUNCATE
 admin_group       | categories  | UPDATE
 readonly_group    | categories  | SELECT
 readonly_group    | customers   | SELECT
 readwrite_group   | categories  | INSERT
 readwrite_group   | categories  | SELECT
 readwrite_group   | categories  | UPDATE
 readwrite_group   | order_items | INSERT
 readwrite_group   | order_items | SELECT
 readwrite_group   | order_items | UPDATE
 readwrite_group   | orders      | INSERT
 readwrite_group   | orders      | SELECT
 readwrite_group   | orders      | UPDATE
 ...
```

### ใช้ psql meta-command `\dp` เพื่อดูสิทธิ์แบบย่อ

```sql
\dp products
```

```
                                          Access privileges
 Schema |    Name    | Type  |             Access privileges              | Column privileges | Policies
--------+------------+-------+---------------------------------------------+--------------------+----------
 public | products   | table | postgres=arwdDxt/postgres                  |                    |
        |            |       | readonly_group=r/postgres                  |                    |
        |            |       | readwrite_group=arw/postgres               |                    |
        |            |       | admin_group=arwdDxt/postgres                |                    |
```

**อธิบายรหัสตัวย่อ**: `r`=SELECT, `a`=INSERT, `w`=UPDATE, `d`=DELETE, `D`=TRUNCATE, `x`=REFERENCES, `t`=TRIGGER — รูปแบบ `role=privileges/grantor` หมายถึง role นั้นได้รับสิทธิ์ดังกล่าวจาก grantor คนใด

---

## Step 675: Privilege บน Column ระดับเฉพาะเจาะจง

### ทำไมต้อง column-level privilege

บางครั้งตารางมีข้อมูลอ่อนไหว (เช่น email ลูกค้า) ที่ไม่ควรให้ทุกคนที่มีสิทธิ์ `SELECT` บนตารางเห็นได้ทั้งหมด PostgreSQL รองรับการให้สิทธิ์ **เจาะจงเป็นรายคอลัมน์** ได้

### รูปแบบคำสั่ง

```sql
GRANT SELECT (column1, column2, ...) ON table_name TO role_name;
GRANT UPDATE (column1, ...) ON table_name TO role_name;
```

### ตัวอย่างจริง: จำกัดสิทธิ์การเห็นข้อมูลลูกค้า

สมมติว่าทีม marketing (`marketing_group`) ควรเห็นเฉพาะชื่อลูกค้าเพื่อทำแคมเปญ แต่ **ไม่ควรเห็น email** เพราะเป็นข้อมูลส่วนบุคคล:

```sql
CREATE ROLE marketing_group NOLOGIN;
CREATE ROLE staff_areeya WITH LOGIN PASSWORD 'Areeya#2026Pass';
GRANT marketing_group TO staff_areeya;

-- ให้สิทธิ์อ่านเฉพาะคอลัมน์ first_name, last_name (ไม่รวม email)
GRANT SELECT (customer_id, first_name, last_name) ON customers TO marketing_group;
```

ทดสอบ:

```sql
SET ROLE staff_areeya;

-- เลือกเฉพาะคอลัมน์ที่ได้รับอนุญาต -> สำเร็จ
SELECT customer_id, first_name, last_name FROM customers;
```

```
 customer_id | first_name | last_name
-------------+------------+-----------
           1 | สมชาย      | ใจดี
           2 | สมหญิง     | รักเรียน
           3 | วิชัย       | มั่งมี
           4 | ปรานี      | ศรีสุข
```

```sql
-- พยายามเลือกคอลัมน์ email -> ถูกปฏิเสธ
SELECT customer_id, first_name, email FROM customers;
```

```
ERROR:  permission denied for table customers
```

```sql
-- แม้แต่ SELECT * ก็ถูกปฏิเสธ เพราะ * รวมคอลัมน์ email ที่ไม่ได้รับอนุญาตด้วย
SELECT * FROM customers;
```

```
ERROR:  permission denied for table customers
```

```sql
RESET ROLE;
```

> **ข้อสังเกตสำคัญ:** เมื่อให้สิทธิ์ `SELECT` แบบจำกัดคอลัมน์ การเรียก `SELECT *` หรือแม้แต่การใช้ตารางนั้นในเงื่อนไข `WHERE` ที่อ้างถึงคอลัมน์ที่ไม่ได้รับอนุญาต จะทำให้เกิด error ทันที ต้องระบุคอลัมน์ที่อนุญาตเท่านั้นในทุกส่วนของ query

### ให้สิทธิ์ UPDATE เฉพาะบางคอลัมน์

เคสที่พบบ่อย: ทีม customer support ควรแก้ไขสถานะคำสั่งซื้อได้ แต่ไม่ควรแก้ไข customer_id หรือ order_date:

```sql
CREATE ROLE support_group NOLOGIN;
GRANT SELECT ON orders TO support_group;
GRANT UPDATE (status) ON orders TO support_group;

CREATE ROLE staff_boonmee WITH LOGIN PASSWORD 'Boonmee#2026Pass';
GRANT support_group TO staff_boonmee;
```

ทดสอบ:

```sql
SET ROLE staff_boonmee;

-- แก้ไขเฉพาะ status -> สำเร็จ
UPDATE orders SET status = 'shipped' WHERE order_id = 3;
```

```
UPDATE 1
```

```sql
-- พยายามแก้ไข customer_id -> ถูกปฏิเสธ
UPDATE orders SET customer_id = 2 WHERE order_id = 3;
```

```
ERROR:  permission denied for table orders
```

```sql
RESET ROLE;
```

### ตรวจสอบ column-level privilege ด้วย information_schema

```sql
SELECT grantee, table_name, column_name, privilege_type
FROM information_schema.column_privileges
WHERE table_schema = 'public'
  AND grantee IN ('marketing_group', 'support_group')
ORDER BY grantee, table_name, column_name;
```

```
     grantee      | table_name |  column_name  | privilege_type
-------------------+------------+----------------+-----------------
 marketing_group   | customers  | customer_id    | SELECT
 marketing_group   | customers  | first_name     | SELECT
 marketing_group   | customers  | last_name      | SELECT
 support_group     | orders     | status         | UPDATE
```

หรือดูด้วย `\dp` (คอลัมน์ "Column privileges" จะแสดงรายละเอียด):

```sql
\dp customers
```

```
                                           Access privileges
 Schema |   Name    | Type  |          Access privileges          |     Column privileges      | Policies
--------+-----------+-------+--------------------------------------+------------------------------+----------
 public | customers | table | postgres=arwdDxt/postgres            | email:                       |
        |           |       | readonly_group=r/postgres            |                              |
        |           |       |                                      | first_name:                 |
        |           |       |                                      |   marketing_group=r/postgres |
        |           |       |                                      | last_name:                  |
        |           |       |                                      |   marketing_group=r/postgres |
```

---

## Step 676: Privilege บน Schema/Database — USAGE, CREATE, CONNECT

### ลำดับชั้นของสิทธิ์: Database → Schema → Table

ก่อนที่ role ใดจะเข้าถึงตารางได้ ต้องผ่านการตรวจสอบสิทธิ์ **สองชั้นก่อน**:

1. สิทธิ์ **CONNECT** บนฐานข้อมูล (database)
2. สิทธิ์ **USAGE** บนสคีมา (schema) ที่ตารางนั้นอยู่

แม้จะมีสิทธิ์ `SELECT` บนตารางแล้ว แต่ถ้าไม่มีสิทธิ์ `USAGE` บนสคีมาที่ครอบตารางนั้นอยู่ ก็จะเข้าถึงตารางไม่ได้เลย

### สิทธิ์ระดับ Database

| Privilege | ความหมาย |
|---|---|
| `CONNECT` | อนุญาตให้เชื่อมต่อ (connect) เข้าฐานข้อมูลนี้ |
| `CREATE` | อนุญาตให้สร้าง schema ใหม่ในฐานข้อมูลนี้ |
| `TEMP` / `TEMPORARY` | อนุญาตให้สร้างตารางชั่วคราว (temp table) |

```sql
-- ตรวจสอบสิทธิ์ปัจจุบันบนฐานข้อมูล
\l ecommerce_security
```

```
                                            List of databases
        Name        |  Owner   | Encoding | ... |   Access privileges
---------------------+----------+----------+-----+------------------------
 ecommerce_security  | postgres | UTF8     | ... |
```

```sql
-- ให้สิทธิ์เชื่อมต่อฐานข้อมูลแก่กลุ่มต่าง ๆ (มักจำเป็นเมื่อ log_connections เข้มงวด หรือ database ไม่ใช่ default)
GRANT CONNECT ON DATABASE ecommerce_security TO readonly_group, readwrite_group, admin_group, marketing_group, support_group;
```

### สิทธิ์ระดับ Schema

| Privilege | ความหมาย |
|---|---|
| `USAGE` | อนุญาตให้ "มองเห็น" ออบเจ็กต์ภายในสคีมา (จำเป็นก่อนเข้าถึงตารางใด ๆ ในสคีมานั้น) |
| `CREATE` | อนุญาตให้สร้างออบเจ็กต์ใหม่ (table, view, function ฯลฯ) ภายในสคีมานั้น |

```sql
-- ให้สิทธิ์ USAGE บนสคีมา public แก่ทุกกลุ่มที่ต้องอ่าน/เขียนข้อมูล
GRANT USAGE ON SCHEMA public TO readonly_group, readwrite_group, admin_group, marketing_group, support_group;

-- ให้สิทธิ์ CREATE บนสคีมา public เฉพาะทีม admin (สร้างตาราง/วิวใหม่ได้)
GRANT CREATE ON SCHEMA public TO admin_group;
```

### ตัวอย่าง: สร้างสคีมาแยกสำหรับแต่ละทีม

ในระบบจริง มืออาชีพมักแยกสคีมาตามหน้าที่ เพื่อจัดการสิทธิ์ได้ง่ายกว่าใช้สคีมา `public` เพียงอย่างเดียว:

```sql
-- สร้างสคีมาสำหรับข้อมูลวิเคราะห์ (analyst ใช้งาน)
CREATE SCHEMA analytics AUTHORIZATION admin_group;

-- สร้างสคีมาสำหรับ log การตรวจสอบ (audit)
CREATE SCHEMA audit AUTHORIZATION admin_group;

-- ให้สิทธิ์ analyst เข้าถึงเฉพาะสคีมา analytics เท่านั้น
GRANT USAGE ON SCHEMA analytics TO readonly_group;
GRANT CREATE ON SCHEMA analytics TO readwrite_group;
```

ตรวจสอบสิทธิ์บนสคีมาด้วย `\dn+`:

```sql
\dn+
```

```
                                        List of schemas
    Name    |  Owner   |          Access privileges           |          Description
-------------+----------+---------------------------------------+---------------------------------
 analytics   | postgres | admin_group=UC/postgres              +|
             |          | readonly_group=U/postgres            +|
             |          | readwrite_group=UC/postgres           |
 audit       | postgres | admin_group=UC/postgres               |
 public      | postgres | postgres=UC/postgres                 +|
             |          | readonly_group=U/postgres             +| standard public schema
             |          | readwrite_group=U/postgres            +|
             |          | admin_group=UC/postgres               +|
             |          | marketing_group=U/postgres            +|
             |          | support_group=U/postgres               |
```

(`U` = USAGE, `C` = CREATE)

### ตรวจสอบด้วย information_schema

```sql
SELECT grantee, table_schema AS schema_name, privilege_type
FROM information_schema.usage_privileges
WHERE object_type = 'SCHEMA'
ORDER BY grantee;
```

> หมายเหตุ: `information_schema` มาตรฐานไม่มี view สำหรับ schema privilege โดยตรงในบาง PostgreSQL version วิธีที่แม่นยำกว่าคือ query จาก `pg_namespace.nspacl` โดยตรง:

```sql
SELECT nspname AS schema_name, nspacl
FROM pg_namespace
WHERE nspname IN ('public', 'analytics', 'audit');
```

```
 schema_name |                                    nspacl
--------------+-----------------------------------------------------------------------------
 public       | {postgres=UC/postgres,readonly_group=U/postgres,readwrite_group=U/postgres,
               admin_group=UC/postgres,marketing_group=U/postgres,support_group=U/postgres}
 analytics    | {admin_group=UC/postgres,readonly_group=U/postgres,readwrite_group=UC/postgres}
 audit        | {admin_group=UC/postgres}
```

---

## Step 677: Default Privileges — ALTER DEFAULT PRIVILEGES

### ปัญหาที่ ALTER DEFAULT PRIVILEGES แก้

`GRANT` ที่เราทำมาทั้งหมดจนถึงตอนนี้ใช้ได้กับ **ออบเจ็กต์ที่มีอยู่แล้ว** เท่านั้น หากมีคนสร้างตารางใหม่ในสคีมาเดียวกันภายหลัง (เช่น `CREATE TABLE promotions (...)`) ตารางใหม่นั้นจะ **ไม่มี** สิทธิ์ที่เรา GRANT ไว้ก่อนหน้าโดยอัตโนมัติ ต้องมา `GRANT` ใหม่ทุกครั้ง ซึ่งไม่สะดวกและเสี่ยงต่อการลืม

`ALTER DEFAULT PRIVILEGES` แก้ปัญหานี้โดยกำหนด **กฎล่วงหน้า** ว่า "เมื่อ role X สร้างออบเจ็กต์ประเภทนี้ในสคีมานี้ในอนาคต ให้ GRANT สิทธิ์ Y แก่ role Z โดยอัตโนมัติ"

### รูปแบบคำสั่ง

```sql
ALTER DEFAULT PRIVILEGES [ FOR ROLE target_role ] [ IN SCHEMA schema_name ]
    GRANT { SELECT | INSERT | ... | ALL } ON TABLES TO role_name;
```

จุดสำคัญ: `ALTER DEFAULT PRIVILEGES` ใช้ได้กับสี่ประเภทออบเจ็กต์คือ `TABLES`, `SEQUENCES`, `FUNCTIONS`, `TYPES`, `SCHEMAS` (PostgreSQL 15+ เพิ่มเติม) และมีผลเฉพาะกับออบเจ็กต์ที่ role เป้าหมาย (`FOR ROLE`) เป็นผู้สร้าง **ในอนาคต** เท่านั้น — ไม่มีผลย้อนหลังกับออบเจ็กต์ที่มีอยู่แล้ว

### ตัวอย่างจริง: ตั้ง default privilege ให้ทีม admin เมื่อสร้างตารางใหม่

```sql
-- เมื่อ admin_group (หรือ role ที่รันคำสั่งนี้ เช่น postgres ผู้ดูแล) สร้างตารางใหม่ในสคีมา public
-- ให้ readonly_group ได้สิทธิ์ SELECT อัตโนมัติ
ALTER DEFAULT PRIVILEGES FOR ROLE postgres IN SCHEMA public
    GRANT SELECT ON TABLES TO readonly_group;

-- และให้ readwrite_group ได้สิทธิ์ SELECT, INSERT, UPDATE อัตโนมัติ
ALTER DEFAULT PRIVILEGES FOR ROLE postgres IN SCHEMA public
    GRANT SELECT, INSERT, UPDATE ON TABLES TO readwrite_group;

-- ตั้งค่าเดียวกันสำหรับ sequence (จำเป็นเมื่อใช้ SERIAL/IDENTITY เพราะต้องมีสิทธิ์ USAGE บน sequence ด้วย)
ALTER DEFAULT PRIVILEGES FOR ROLE postgres IN SCHEMA public
    GRANT USAGE, SELECT ON SEQUENCES TO readwrite_group;
```

### ทดสอบ: สร้างตารางใหม่แล้วดูว่าสิทธิ์ถูกตั้งอัตโนมัติหรือไม่

```sql
-- postgres (role ที่เราตั้ง default privilege ไว้ให้) สร้างตารางใหม่
CREATE TABLE promotions (
    promotion_id SERIAL PRIMARY KEY,
    promotion_name VARCHAR(150),
    discount_percent NUMERIC(5,2)
);
```

```sql
\dp promotions
```

```
                                          Access privileges
 Schema |     Name    | Type  |             Access privileges              | Column privileges | Policies
--------+-------------+-------+----------------------------------------------+--------------------+----------
 public | promotions  | table | postgres=arwdDxt/postgres                   |                    |
        |             |       | readonly_group=r/postgres                   |                    |
        |             |       | readwrite_group=arw/postgres                |                    |
```

สังเกตว่า `readonly_group` และ `readwrite_group` ได้สิทธิ์ทันทีโดยที่เราไม่ต้อง `GRANT` เพิ่มเลย เพราะกฎ default privilege ที่ตั้งไว้ล่วงหน้ามีผลบังคับใช้อัตโนมัติกับตารางใหม่ทุกตารางที่ `postgres` สร้างในสคีมา `public`

### ตรวจสอบกฎ default privilege ที่ตั้งไว้ทั้งหมด

```sql
\ddp
```

```
                                Default access privileges
  Owner   | Schema |   Type   |               Access privileges
----------+--------+----------+--------------------------------------------------
 postgres | public | table    | readonly_group=r/postgres                      +
          |        |          | readwrite_group=arw/postgres
 postgres | public | sequence | readwrite_group=rU/postgres
```

หรือด้วย SQL:

```sql
SELECT
    pg_get_userbyid(defaclrole) AS grantor_role,
    n.nspname AS schema_name,
    CASE defaclobjtype
        WHEN 'r' THEN 'table'
        WHEN 'S' THEN 'sequence'
        WHEN 'f' THEN 'function'
        WHEN 'T' THEN 'type'
        WHEN 'n' THEN 'schema'
    END AS object_type,
    defaclacl AS default_acl
FROM pg_default_acl d
LEFT JOIN pg_namespace n ON d.defaclnamespace = n.oid;
```

```
 grantor_role | schema_name | object_type |                default_acl
---------------+-------------+-------------+---------------------------------------------
 postgres      | public      | table       | {readonly_group=r/postgres,readwrite_group=arw/postgres}
 postgres      | public      | sequence    | {readwrite_group=rU/postgres}
```

### ข้อควรระวังสำคัญ

1. **`FOR ROLE` สำคัญมาก** — กฎ default privilege ผูกกับ "ผู้สร้างออบเจ็กต์" ไม่ใช่สคีมาเฉย ๆ ถ้าตารางใหม่ถูกสร้างโดย role อื่นที่ไม่ใช่ role ที่ระบุใน `FOR ROLE` กฎนี้จะไม่มีผล ดังนั้นในทีมงานจริง ควรตั้งกฎ `FOR ROLE` ให้ครบทุก role ที่มีสิทธิ์สร้างตาราง (เช่น `admin_group` และสมาชิกของมัน) หรือให้แอปพลิเคชัน deploy schema ด้วย role เดียวกันเสมอ (migration role)
2. **ไม่มีผลย้อนหลัง** — ต้อง `GRANT` ให้ตารางเก่าด้วยตนเองแยกต่างหาก
3. การยกเลิกกฎใช้ `ALTER DEFAULT PRIVILEGES ... REVOKE ...`:

```sql
ALTER DEFAULT PRIVILEGES FOR ROLE postgres IN SCHEMA public
    REVOKE SELECT ON TABLES FROM readonly_group;
```

---

## Step 678: PUBLIC role — สิทธิ์ default ที่ทุกคนมี และความเสี่ยง

### PUBLIC คืออะไร

`PUBLIC` ไม่ใช่ role จริงที่สร้างขึ้นมา แต่เป็น **pseudo-role พิเศษ** ที่หมายถึง "ทุก role ในระบบ" เมื่อ `GRANT` สิทธิ์ให้ `PUBLIC` เท่ากับให้สิทธิ์นั้นแก่ role ทุกตัวที่มีอยู่ในคลัสเตอร์ (รวมถึง role ที่จะถูกสร้างขึ้นในอนาคตด้วย!)

### สิทธิ์ที่ PUBLIC มีโดย default (ก่อนเราตั้งค่าอะไรเลย)

ในเวอร์ชันเก่า (PostgreSQL ≤ 14) เมื่อสร้างฐานข้อมูลใหม่ ทุก role (รวมถึง role ใหม่ที่สร้างขึ้นภายหลัง) จะมีสิทธิ์ต่อไปนี้โดยอัตโนมัติผ่าน `PUBLIC`:

- `CONNECT` และ `TEMP` บนทุกฐานข้อมูล
- `EXECUTE` บนทุกฟังก์ชันที่สร้างขึ้นใหม่ (โดย default privilege ของฟังก์ชัน)
- `USAGE` บนทุก data type และ language
- **`CREATE` บนสคีมา `public`** ← นี่คือจุดเสี่ยงที่สุด

### ความเสี่ยงของ CREATE บน public schema (ก่อน PostgreSQL 15)

ก่อน PostgreSQL 15 ทุก role ที่ล็อกอินได้ (แม้จะไม่มีสิทธิ์อะไรเลยนอกจาก `CONNECT`) สามารถสร้างตารางในสคีมา `public` ได้ทันที ซึ่งนำไปสู่ความเสี่ยงหลายอย่าง:

1. **CVE-2018-1058**: ผู้โจมตีที่มีบัญชีธรรมดาสามารถสร้างฟังก์ชัน/ตารางชื่อเดียวกับของระบบในสคีมา `public` แล้วอาศัยช่องโหว่ของ `search_path` หลอกให้ query ของ superuser เรียกใช้โค้ดอันตรายแทน (SQL injection ผ่าน schema)
2. ผู้ใช้ทั่วไปสามารถถมพื้นที่ดิสก์ด้วยการสร้างตารางจำนวนมากในสคีมาที่ใช้ร่วมกัน
3. ยากต่อการตรวจสอบว่าใครเป็นเจ้าของออบเจ็กต์ใดในสคีมา `public` เพราะทุกคนสร้างได้

### การเปลี่ยนแปลงใน PostgreSQL 15+

ตั้งแต่ **PostgreSQL 15** เป็นต้นไป พฤติกรรม default เปลี่ยนไปอย่างมีนัยสำคัญ:

> **สคีมา `public` ในฐานข้อมูลที่สร้างใหม่จะไม่มีสิทธิ์ `CREATE` ให้กับ `PUBLIC` อีกต่อไป** เจ้าของฐานข้อมูล (โดยปกติคือ role ที่รัน `CREATE DATABASE`) เท่านั้นที่มีสิทธิ์สร้างออบเจ็กต์ในสคีมา `public` ได้จนกว่าจะ `GRANT` เพิ่มเติมอย่างชัดเจน

ตรวจสอบสิทธิ์ default ของสคีมา `public` ในฐานข้อมูลใหม่บน PostgreSQL 15+:

```sql
\dn+ public
```

```
                                  List of schemas
  Name  |  Owner   |         Access privileges         | Description
--------+----------+-------------------------------------+--------------------------
 public | postgres | postgres=UC/postgres              +| standard public schema
        |          | =U/postgres                         |
```

สังเกตบรรทัด `=U/postgres` — นี่คือสิทธิ์ของ `PUBLIC` (ไม่มีชื่อ role ด้านหน้าเครื่องหมาย `=` หมายถึง `PUBLIC`) ซึ่งมีเพียง `U` (USAGE) เท่านั้น **ไม่มี `C` (CREATE)** แล้ว — ต่างจากเวอร์ชันเก่าที่จะมี `UC` ให้ `PUBLIC` ด้วย

### ตรวจสอบและจัดการสิทธิ์ของ PUBLIC ด้วยตนเอง

```sql
-- ตรวจสอบว่า PUBLIC มีสิทธิ์อะไรบนสคีมา public บ้าง
SELECT nspacl FROM pg_namespace WHERE nspname = 'public';
```

```sql
-- แนวทางที่ปลอดภัยที่สุด (best practice ทุกเวอร์ชัน แม้ใน 15+ ที่ default ปลอดภัยขึ้นแล้ว):
-- เพิกถอนสิทธิ์ CREATE จาก PUBLIC อย่างชัดเจน เพื่อไม่พึ่งพา default ของแต่ละเวอร์ชัน
REVOKE CREATE ON SCHEMA public FROM PUBLIC;

-- ตรวจสอบว่า PUBLIC ยังมีสิทธิ์ CONNECT บนฐานข้อมูลอยู่หรือไม่ (มักจำเป็นต้องมี เว้นแต่ต้องการปิดสนิท)
REVOKE ALL ON DATABASE ecommerce_security FROM PUBLIC;
GRANT CONNECT ON DATABASE ecommerce_security TO readonly_group, readwrite_group, admin_group, marketing_group, support_group;
```

### ตัวอย่างอันตราย: EXECUTE บนฟังก์ชันใหม่

จุดที่มักถูกมองข้าม: ฟังก์ชันที่สร้างใหม่จะได้ `EXECUTE` privilege ให้ `PUBLIC` โดย default (ต่างจากตารางที่ไม่มี default privilege ให้ PUBLIC) ทำให้ทุก role เรียกใช้ฟังก์ชันนั้นได้ทันที แม้เป็นฟังก์ชันที่ทำงานผ่านสิทธิ์ของเจ้าของ (`SECURITY DEFINER`) ก็ตาม:

```sql
CREATE FUNCTION calc_order_total(p_order_id INT)
RETURNS NUMERIC
LANGUAGE sql
SECURITY DEFINER
AS $$
    SELECT COALESCE(SUM(oi.quantity * p.unit_price), 0)
    FROM order_items oi
    JOIN products p ON p.product_id = oi.product_id
    WHERE oi.order_id = p_order_id;
$$;

-- ตรวจสอบสิทธิ์เริ่มต้น
\df+ calc_order_total
```

```
Access privileges: =X/postgres
                    postgres=X/postgres
```

`=X/postgres` หมายถึง `PUBLIC` มีสิทธิ์ `EXECUTE` (`X`) โดยอัตโนมัติ — หากฟังก์ชันนี้ทำงานด้วยสิทธิ์สูง (`SECURITY DEFINER`) แล้วมีช่องโหว่ ผู้ใช้ใด ๆ ก็เรียกใช้ได้ทันที จึงควรเพิกถอนเสมอสำหรับฟังก์ชันที่มีความอ่อนไหว:

```sql
REVOKE EXECUTE ON FUNCTION calc_order_total(INT) FROM PUBLIC;
GRANT EXECUTE ON FUNCTION calc_order_total(INT) TO readwrite_group, admin_group;
```

### สรุปแนวทางปฏิบัติเกี่ยวกับ PUBLIC

| แนวทาง | เหตุผล |
|---|---|
| `REVOKE CREATE ON SCHEMA public FROM PUBLIC;` ทุกครั้งหลังสร้างฐานข้อมูล | ป้องกันไม่ว่า PostgreSQL version ใด |
| `REVOKE EXECUTE ON FUNCTION ... FROM PUBLIC;` สำหรับฟังก์ชัน `SECURITY DEFINER` | ฟังก์ชันความปลอดภัยสูงไม่ควรเปิดให้ทุกคนเรียก |
| ตรวจสอบ `nspacl`, `\dp`, `\df+` เป็นระยะ | ตรวจจับสิทธิ์ที่หลุดไปถึง PUBLIC โดยไม่ตั้งใจ |
| หลีกเลี่ยงการใช้ `GRANT ... TO PUBLIC` เว้นแต่ตั้งใจให้ "ทุกคน" เข้าถึงจริง ๆ | ลดพื้นผิวการโจมตี (attack surface) |

---

## Step 679: หลักการ Least Privilege — ออกแบบ role สำหรับ application

### หลักการ Least Privilege คืออะไร

**Least Privilege (สิทธิ์น้อยที่สุดเท่าที่จำเป็น)** คือหลักการด้านความปลอดภัยที่ระบุว่า role หรือระบบใด ๆ ควรได้รับสิทธิ์ **เพียงเท่าที่จำเป็นต่อการทำงานของมันเท่านั้น** ไม่มากไปกว่านั้น เพื่อลดความเสียหายหากบัญชีนั้นถูกโจมตีหรือถูกใช้ผิดวัตถุประสงค์

ในบริบทของแอปพลิเคชัน e-commerce ทั่วไป เรามักแบ่ง role ตามบทบาทการทำงานอย่างน้อย 3 ระดับ:

1. **Read-only role** — สำหรับรายงาน, dashboard, BI tools, analyst
2. **Read-write role** — สำหรับ backend API ที่ต้องอ่านและเขียนข้อมูลผ่าน business logic
3. **Admin role** — สำหรับ DBA / migration script เท่านั้น

### ออกแบบ Read-only role สำหรับ Analyst

```sql
-- 1. สร้าง group role
CREATE ROLE app_readonly NOLOGIN;

-- 2. ให้สิทธิ์ CONNECT + USAGE ก่อน (จำเป็นเสมอ)
GRANT CONNECT ON DATABASE ecommerce_security TO app_readonly;
GRANT USAGE ON SCHEMA public TO app_readonly;

-- 3. ให้สิทธิ์ SELECT เฉพาะที่จำเป็น (ไม่ใช่ทุกตารางเสมอไป — ตัดคอลัมน์อ่อนไหวออกถ้าจำเป็น)
GRANT SELECT ON ALL TABLES IN SCHEMA public TO app_readonly;

-- 4. ตั้ง default privilege ให้ตารางใหม่ในอนาคตด้วย
ALTER DEFAULT PRIVILEGES FOR ROLE postgres IN SCHEMA public
    GRANT SELECT ON TABLES TO app_readonly;

-- 5. ห้ามให้สิทธิ์เขียนใด ๆ ทั้งสิ้น (ไม่ต้องทำอะไร เพราะ default คือไม่มีสิทธิ์)

-- 6. จำกัด connection limit เพื่อป้องกัน resource exhaustion
-- (ตั้งที่ตัว login role แต่ละตัว ไม่ใช่ group role)
```

### ออกแบบ Read-write role สำหรับ Backend API

```sql
-- 1. สร้าง group role
CREATE ROLE app_readwrite NOLOGIN;

GRANT CONNECT ON DATABASE ecommerce_security TO app_readwrite;
GRANT USAGE ON SCHEMA public TO app_readwrite;

-- 2. ให้สิทธิ์อ่าน-เขียนข้อมูล แต่ไม่ให้ TRUNCATE หรือ DDL ใด ๆ
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app_readwrite;

-- 3. ต้องมีสิทธิ์ใช้งาน sequence ด้วย เพราะตารางใช้ SERIAL (nextval ต้องมีสิทธิ์ USAGE)
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO app_readwrite;

-- 4. ตั้ง default privilege สำหรับอนาคต
ALTER DEFAULT PRIVILEGES FOR ROLE postgres IN SCHEMA public
    GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO app_readwrite;
ALTER DEFAULT PRIVILEGES FOR ROLE postgres IN SCHEMA public
    GRANT USAGE, SELECT ON SEQUENCES TO app_readwrite;

-- 5. ปฏิเสธการลบข้อมูลลูกค้าและคำสั่งซื้อโดยตรง (ธุรกิจกำหนดว่าต้องผ่าน soft-delete/business logic เท่านั้น)
REVOKE DELETE ON customers, orders FROM app_readwrite;
```

### ออกแบบ Admin role สำหรับ DBA / Migration

```sql
-- 1. สร้าง group role ที่มีสิทธิ์เต็มเฉพาะระดับ schema (ไม่ใช่ superuser)
CREATE ROLE app_admin NOLOGIN;

GRANT CONNECT ON DATABASE ecommerce_security TO app_admin;
GRANT ALL PRIVILEGES ON SCHEMA public TO app_admin;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO app_admin;
GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA public TO app_admin;

-- 2. ให้สิทธิ์เป็นเจ้าของ default privilege ด้วยตนเอง เพื่อให้ role นี้สร้างตารางแล้วตั้งสิทธิ์ตามได้
ALTER DEFAULT PRIVILEGES FOR ROLE app_admin IN SCHEMA public
    GRANT SELECT ON TABLES TO app_readonly;
ALTER DEFAULT PRIVILEGES FOR ROLE app_admin IN SCHEMA public
    GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO app_readwrite;
```

### สร้าง login role จริงและผูกกับ group role ที่เหมาะสม

```sql
-- Service account สำหรับ backend API (ใช้ connection pool)
CREATE ROLE svc_ecommerce_api WITH LOGIN PASSWORD 'Api#Svc2026Secure!' CONNECTION LIMIT 20;
GRANT app_readwrite TO svc_ecommerce_api;

-- Service account สำหรับ BI/Dashboard (Metabase, Superset ฯลฯ)
CREATE ROLE svc_bi_dashboard WITH LOGIN PASSWORD 'Bi#Svc2026Secure!' CONNECTION LIMIT 10;
GRANT app_readonly TO svc_bi_dashboard;

-- Migration role สำหรับ CI/CD pipeline (ใช้ตอน deploy schema เท่านั้น ไม่ใช้ตลอดเวลา)
CREATE ROLE svc_migration WITH LOGIN PASSWORD 'Migrate#Svc2026!' VALID UNTIL '2027-01-01';
GRANT app_admin TO svc_migration;
```

### ทดสอบ end-to-end

```sql
SET ROLE svc_bi_dashboard;
SELECT category_name, COUNT(*) AS product_count
FROM categories c
JOIN products p USING (category_id)
GROUP BY category_name
ORDER BY product_count DESC;
```

```
  category_name   | product_count
-------------------+---------------
 อิเล็กทรอนิกส์    |             2
 เครื่องใช้ในบ้าน  |             2
 เสื้อผ้า          |             1
 หนังสือ           |             1
 ของเล่น           |             1
```

```sql
-- ลองสร้างตาราง -> ต้องถูกปฏิเสธ เพราะ app_readonly ไม่มีสิทธิ์ CREATE
CREATE TABLE hack_test (id INT);
```

```
ERROR:  permission denied for schema public
```

```sql
RESET ROLE;
```

### ตารางสรุปการออกแบบ role 3 ระดับ

| Role | LOGIN | SELECT | INSERT/UPDATE | DELETE | DDL (CREATE TABLE) | ใช้งานโดย |
|---|---|---|---|---|---|---|
| `app_readonly` | ไม่ (group) | ทุกตาราง | ไม่มี | ไม่มี | ไม่มี | BI, analyst, reporting |
| `app_readwrite` | ไม่ (group) | ทุกตาราง | มี (ยกเว้น customers/orders DELETE) | จำกัด | ไม่มี | Backend API |
| `app_admin` | ไม่ (group) | ทุกตาราง | มีทั้งหมด | มี | มี | DBA, CI/CD migration |
| `svc_bi_dashboard` | มี | สืบทอดจาก `app_readonly` | - | - | - | เครื่องมือ BI |
| `svc_ecommerce_api` | มี | สืบทอดจาก `app_readwrite` | - | - | - | Backend service |
| `svc_migration` | มี | สืบทอดจาก `app_admin` | - | - | - | CI/CD pipeline |

---

## Step 680: แบบฝึกหัดรวม — ออกแบบระบบ role สำหรับทีม e-commerce

### โจทย์สถานการณ์จริง

บริษัท e-commerce แห่งหนึ่งมีทีมงาน 3 กลุ่มที่ต้องเข้าถึงฐานข้อมูล `ecommerce_security`:

1. **ทีมพัฒนา (Development Team)** — เขียนและทดสอบฟีเจอร์ใหม่ ต้องอ่าน-เขียนข้อมูลได้เกือบทุกตาราง ยกเว้นข้อมูลลูกค้าที่อ่อนไหว (เช่น email) ซึ่งต้องเห็นแบบ masked หรือจำกัด
2. **ทีม Backend/Application** — service account ของระบบที่รันจริง (production) ต้องอ่าน-เขียนข้อมูลตามธุรกิจ แต่ห้ามลบข้อมูลคำสั่งซื้อโดยตรง (ต้องผ่าน stored procedure เท่านั้น)
3. **ทีม Analyst** — วิเคราะห์ข้อมูลเชิงธุรกิจ ต้องการเข้าถึงเฉพาะข้อมูลสรุป/รายงาน (อ่านอย่างเดียว) และไม่ควรเห็นข้อมูลติดต่อลูกค้าโดยตรง

### แนวทางแก้ปัญหาแบบสมบูรณ์

```sql
-- =========================================================
-- ขั้นตอนที่ 1: ล้างของเก่า (สำหรับรันซ้ำในสภาพแวดล้อมทดสอบ)
-- =========================================================
-- (ข้ามได้หากรันครั้งแรก)

-- =========================================================
-- ขั้นตอนที่ 2: สร้าง group role หลัก 3 กลุ่ม
-- =========================================================
CREATE ROLE team_dev NOLOGIN;
CREATE ROLE team_backend NOLOGIN;
CREATE ROLE team_analyst NOLOGIN;

-- =========================================================
-- ขั้นตอนที่ 3: สิทธิ์ระดับ Database และ Schema สำหรับทุกกลุ่ม
-- =========================================================
GRANT CONNECT ON DATABASE ecommerce_security TO team_dev, team_backend, team_analyst;
GRANT USAGE ON SCHEMA public TO team_dev, team_backend, team_analyst;

-- =========================================================
-- ขั้นตอนที่ 4: สิทธิ์สำหรับทีมพัฒนา (team_dev)
--   - อ่าน-เขียนได้เกือบทุกตาราง
--   - เห็นข้อมูลลูกค้าได้ แต่ "ไม่เห็น email" (column-level privilege)
-- =========================================================
GRANT SELECT, INSERT, UPDATE, DELETE
    ON categories, suppliers, products, orders, order_items
    TO team_dev;

GRANT SELECT (customer_id, first_name, last_name), INSERT (first_name, last_name), UPDATE (first_name, last_name)
    ON customers
    TO team_dev;

GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO team_dev;

-- =========================================================
-- ขั้นตอนที่ 5: สิทธิ์สำหรับทีม Backend/Application (team_backend)
--   - อ่าน-เขียนได้ทุกตารางรวม email ลูกค้า (ต้องใช้จริงตอนสมัครสมาชิก/ส่งอีเมล)
--   - ห้ามลบ orders/order_items โดยตรง
-- =========================================================
GRANT SELECT, INSERT, UPDATE, DELETE
    ON categories, suppliers, products, customers
    TO team_backend;

GRANT SELECT, INSERT, UPDATE
    ON orders, order_items
    TO team_backend;   -- ไม่ GRANT DELETE บน orders/order_items โดยตั้งใจ

GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO team_backend;

-- สร้างฟังก์ชัน "ยกเลิกคำสั่งซื้อ" แบบมีเงื่อนไข (soft-delete ผ่าน status แทนการ DELETE จริง)
CREATE OR REPLACE FUNCTION cancel_order(p_order_id INT)
RETURNS VOID
LANGUAGE sql
SECURITY DEFINER
AS $$
    UPDATE orders SET status = 'cancelled' WHERE order_id = p_order_id;
$$;

REVOKE EXECUTE ON FUNCTION cancel_order(INT) FROM PUBLIC;
GRANT EXECUTE ON FUNCTION cancel_order(INT) TO team_backend;

-- =========================================================
-- ขั้นตอนที่ 6: สิทธิ์สำหรับทีม Analyst (team_analyst)
--   - อ่านอย่างเดียวทุกตาราง ยกเว้นคอลัมน์อ่อนไหวของลูกค้า
-- =========================================================
GRANT SELECT ON categories, suppliers, products, orders, order_items TO team_analyst;
GRANT SELECT (customer_id, first_name, last_name) ON customers TO team_analyst;

-- =========================================================
-- ขั้นตอนที่ 7: ตั้ง Default Privileges ล่วงหน้า
--   สมมติว่าทีม backend (ผ่าน migration role) เป็นผู้สร้างตารางใหม่เสมอในอนาคต
-- =========================================================
ALTER DEFAULT PRIVILEGES FOR ROLE postgres IN SCHEMA public
    GRANT SELECT ON TABLES TO team_analyst;
ALTER DEFAULT PRIVILEGES FOR ROLE postgres IN SCHEMA public
    GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO team_dev;
ALTER DEFAULT PRIVILEGES FOR ROLE postgres IN SCHEMA public
    GRANT SELECT, INSERT, UPDATE ON TABLES TO team_backend;
ALTER DEFAULT PRIVILEGES FOR ROLE postgres IN SCHEMA public
    GRANT USAGE, SELECT ON SEQUENCES TO team_dev, team_backend;

-- =========================================================
-- ขั้นตอนที่ 8: ปิดช่องโหว่ PUBLIC
-- =========================================================
REVOKE CREATE ON SCHEMA public FROM PUBLIC;
REVOKE ALL ON DATABASE ecommerce_security FROM PUBLIC;
GRANT CONNECT ON DATABASE ecommerce_security TO team_dev, team_backend, team_analyst;

-- =========================================================
-- ขั้นตอนที่ 9: สร้างบัญชีผู้ใช้จริง (login role) และผูกกับกลุ่ม
-- =========================================================
CREATE ROLE dev_thanawat  WITH LOGIN PASSWORD 'Thanawat#2026' CONNECTION LIMIT 5;
CREATE ROLE dev_kanya     WITH LOGIN PASSWORD 'Kanya#2026Dev' CONNECTION LIMIT 5;
CREATE ROLE svc_backend_prod WITH LOGIN PASSWORD 'Backend#Prod2026!' CONNECTION LIMIT 50;
CREATE ROLE analyst_chai  WITH LOGIN PASSWORD 'Chai#Analyst2026' CONNECTION LIMIT 3;
CREATE ROLE analyst_orn   WITH LOGIN PASSWORD 'Orn#Analyst2026' CONNECTION LIMIT 3;

GRANT team_dev TO dev_thanawat, dev_kanya;
GRANT team_backend TO svc_backend_prod;
GRANT team_analyst TO analyst_chai, analyst_orn;
```

### ตรวจสอบผลลัพธ์ทั้งหมดด้วย `\du`

```sql
\du
```

```
                                            List of roles
       Role name       |                        Attributes                         |     Member of
------------------------+-------------------------------------------------------------+---------------------
 analyst_chai           | 3 connections                                                | {team_analyst}
 analyst_orn            | 3 connections                                                | {team_analyst}
 dev_kanya              | 5 connections                                                | {team_dev}
 dev_thanawat           | 5 connections                                                | {team_dev}
 svc_backend_prod       | 50 connections                                               | {team_backend}
 team_analyst           | Cannot login                                                 | {}
 team_backend           | Cannot login                                                 | {}
 team_dev               | Cannot login                                                 | {}
 postgres               | Superuser, Create role, Create DB, Replication, Bypass RLS   | {}
```

### ตรวจสอบสิทธิ์ทั้งระบบด้วย query สรุป

```sql
SELECT
    grantee,
    table_name,
    string_agg(privilege_type, ', ' ORDER BY privilege_type) AS privileges
FROM information_schema.role_table_grants
WHERE table_schema = 'public'
  AND grantee IN ('team_dev', 'team_backend', 'team_analyst')
GROUP BY grantee, table_name
ORDER BY grantee, table_name;
```

```
   grantee    | table_name  |          privileges
---------------+-------------+--------------------------------
 team_analyst  | categories  | SELECT
 team_analyst  | customers   | SELECT
 team_analyst  | order_items | SELECT
 team_analyst  | orders      | SELECT
 team_analyst  | products    | SELECT
 team_analyst  | suppliers   | SELECT
 team_backend  | categories  | DELETE, INSERT, SELECT, UPDATE
 team_backend  | customers   | DELETE, INSERT, SELECT, UPDATE
 team_backend  | order_items | INSERT, SELECT, UPDATE
 team_backend  | orders      | INSERT, SELECT, UPDATE
 team_backend  | products    | DELETE, INSERT, SELECT, UPDATE
 team_backend  | suppliers   | DELETE, INSERT, SELECT, UPDATE
 team_dev      | categories  | DELETE, INSERT, SELECT, UPDATE
 team_dev      | customers   | SELECT
 team_dev      | order_items | DELETE, INSERT, SELECT, UPDATE
 team_dev      | orders      | DELETE, INSERT, SELECT, UPDATE
 team_dev      | products    | DELETE, INSERT, SELECT, UPDATE
 team_dev      | suppliers   | DELETE, INSERT, SELECT, UPDATE
```

### ทดสอบสถานการณ์จริงแบบครบวงจร

```sql
-- ทดสอบในฐานะ analyst: อ่านรายงานยอดขายได้ แต่ห้ามเห็น email
SET ROLE analyst_chai;

SELECT c.first_name, c.last_name, COUNT(o.order_id) AS total_orders
FROM customers c
JOIN orders o ON o.customer_id = c.customer_id
GROUP BY c.first_name, c.last_name
ORDER BY total_orders DESC;
```

```
 first_name | last_name | total_orders
------------+-----------+---------------
 สมชาย      | ใจดี      |             2
 สมหญิง     | รักเรียน  |             1
 วิชัย       | มั่งมี     |             1
```

```sql
-- ลองเข้าถึง email -> ถูกปฏิเสธ ตามที่ออกแบบไว้
SELECT email FROM customers;
```

```
ERROR:  permission denied for table customers
```

```sql
RESET ROLE;

-- ทดสอบในฐานะ backend service: ยกเลิกคำสั่งซื้อผ่านฟังก์ชันที่กำหนดได้
SET ROLE svc_backend_prod;
SELECT cancel_order(4);
SELECT order_id, status FROM orders WHERE order_id = 4;
```

```
 order_id |  status
----------+-----------
        4 | cancelled
```

```sql
-- แต่ backend ลบ order โดยตรงไม่ได้ (ตามที่ออกแบบไว้ว่าห้าม DELETE)
DELETE FROM orders WHERE order_id = 4;
```

```
ERROR:  permission denied for table orders
```

```sql
RESET ROLE;
```

ระบบ role นี้แสดงให้เห็นหลักการ Least Privilege อย่างสมบูรณ์: แต่ละทีมมีสิทธิ์เพียงพอต่อการทำงานจริง ไม่มากไปกว่านั้น และมีการป้องกันหลายชั้น (database → schema → table → column) ตามหลักการ defense in depth

---

## สรุปท้ายบท

ในบทนี้เราได้เรียนรู้ระบบ **Role และ Privilege** ของ PostgreSQL อย่างครบถ้วน ตั้งแต่แนวคิดพื้นฐานไปจนถึงการออกแบบระบบสิทธิ์ระดับองค์กร:

1. **Role คือหน่วยเดียวที่ใช้แทนทั้ง user และ group** — ไม่มีความแตกต่างเชิงโครงสร้างระหว่างทั้งสอง ต่างกันเพียง attribute `LOGIN`
2. **`CREATE ROLE`** พร้อม attribute เช่น `LOGIN`, `SUPERUSER`, `CREATEDB`, `CREATEROLE`, `PASSWORD`, `VALID UNTIL` ควบคุมความสามารถพื้นฐานของ role
3. **Role Membership** (`GRANT role_a TO role_b`) คือกลไกหลักที่ทำให้เราสร้างโครงสร้าง group role และมอบสิทธิ์แบบรวมศูนย์ได้ง่ายกว่าการ grant รายบุคคล
4. **Table-level privilege** (`SELECT`, `INSERT`, `UPDATE`, `DELETE`, `TRUNCATE`, `REFERENCES`, `TRIGGER`) ควบคุมว่าใครทำอะไรกับตารางได้บ้าง
5. **Column-level privilege** ให้ความละเอียดสูงขึ้นไปอีกขั้น เหมาะกับการปกป้องข้อมูลอ่อนไหว เช่น อีเมลหรือข้อมูลส่วนบุคคล
6. **Schema/Database privilege** (`CONNECT`, `USAGE`, `CREATE`) เป็นชั้นการตรวจสอบสิทธิ์ที่ต้องผ่านก่อนถึงชั้นตาราง
7. **`ALTER DEFAULT PRIVILEGES`** ทำให้สิทธิ์ถูกกำหนดล่วงหน้าสำหรับออบเจ็กต์ที่จะสร้างในอนาคต ลดความเสี่ยงจากการลืม grant
8. **PUBLIC role** เป็นดาบสองคม — สะดวกแต่เสี่ยง ต้องตรวจสอบและปิดช่องโหว่อย่างตั้งใจเสมอ โดยเฉพาะ `CREATE` บนสคีมา `public` (แม้ PostgreSQL 15+ จะปลอดภัยขึ้นโดย default แล้วก็ตาม)
9. **หลักการ Least Privilege** คือหัวใจของการออกแบบระบบความปลอดภัยที่ดี — แบ่ง role ตามบทบาทจริง (read-only / read-write / admin) และให้สิทธิ์เท่าที่จำเป็นเท่านั้น
10. การผสมผสานทุกเทคนิคเข้าด้วยกัน (group role + table privilege + column privilege + default privilege + PUBLIC hardening) คือแนวทางที่ใช้ในระบบ production จริงระดับองค์กร

ในบทถัดไป เราจะยกระดับความปลอดภัยไปอีกขั้นด้วย **Row-Level Security (RLS)** ซึ่งช่วยให้เรากำหนดได้ว่า role หนึ่ง ๆ เห็น **แถว (row)** ใดได้บ้างในตารางเดียวกัน — เช่น ให้ลูกค้าแต่ละคนเห็นเฉพาะคำสั่งซื้อของตัวเอง โดยไม่ต้องเขียน `WHERE customer_id = ...` ในทุก query ของแอปพลิเคชัน

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1

จงอธิบายว่าเพราะเหตุใด PostgreSQL จึงไม่มี "user" ที่แยกออกจาก "role" อย่างแท้จริง และ `CREATE USER foo;` เทียบเท่ากับคำสั่งใด

<details>
<summary>เฉลย</summary>

PostgreSQL ออกแบบให้ role เป็นหน่วยเดียวที่แทนทั้งผู้ใช้และกลุ่มสิทธิ์ ตั้งแต่เวอร์ชัน 8.1 เป็นต้นมา `CREATE USER foo;` เป็นเพียง syntactic sugar ที่เทียบเท่ากับ:

```sql
CREATE ROLE foo WITH LOGIN;
```

ความแตกต่างเดียวคือ attribute `LOGIN` ที่ทำให้ role นั้นสามารถเชื่อมต่อฐานข้อมูลได้ (จึงถูกเรียกว่า "user" ในทางปฏิบัติ) ขณะที่ role ที่ไม่มี `LOGIN` มักถูกใช้เป็น "group" สำหรับรวมสิทธิ์แทน
</details>

---

### แบบฝึกหัดที่ 2

จงสร้าง role ชื่อ `temp_contractor` ที่ล็อกอินได้ มีรหัสผ่านว่า `Contract#2026`, จำกัด connection ไม่เกิน 2 ครั้งพร้อมกัน และรหัสผ่านหมดอายุวันที่ 31 มีนาคม 2027

<details>
<summary>เฉลย</summary>

```sql
CREATE ROLE temp_contractor WITH
    LOGIN
    PASSWORD 'Contract#2026'
    CONNECTION LIMIT 2
    VALID UNTIL '2027-03-31 23:59:59+07';
```

ตรวจสอบผลลัพธ์:

```sql
\du temp_contractor
```

```
                                     List of roles
     Role name     |                    Attributes                     | Member of
--------------------+-----------------------------------------------------+-----------
 temp_contractor    | 2 connections, Password valid until 2027-03-31...  | {}
```
</details>

---

### แบบฝึกหัดที่ 3

กำหนดให้มี group role ชื่อ `finance_group` (ไม่มี LOGIN) และมี login role ชื่อ `staff_pattama` จงเขียนคำสั่งเพื่อให้ `staff_pattama` เป็นสมาชิกของ `finance_group` โดยที่ `staff_pattama` สามารถเพิ่มสมาชิกใหม่ให้ `finance_group` ได้ด้วยตัวเอง

<details>
<summary>เฉลย</summary>

```sql
CREATE ROLE finance_group NOLOGIN;
CREATE ROLE staff_pattama WITH LOGIN PASSWORD 'Pattama#2026';

GRANT finance_group TO staff_pattama WITH ADMIN OPTION;
```

การใช้ `WITH ADMIN OPTION` ทำให้ `staff_pattama` มีสิทธิ์บริหารจัดการสมาชิกของ `finance_group` ต่อได้เอง (เช่น `GRANT finance_group TO another_user;`) โดยไม่ต้องพึ่ง superuser
</details>

---

### แบบฝึกหัดที่ 4

จงเขียนคำสั่งเพื่อให้สิทธิ์ `SELECT` และ `INSERT` (แต่ไม่ให้ `UPDATE`/`DELETE`) บนตาราง `order_items` แก่ role ชื่อ `warehouse_group` จากนั้นเขียน query เพื่อตรวจสอบผลลัพธ์ผ่าน `information_schema.role_table_grants`

<details>
<summary>เฉลย</summary>

```sql
CREATE ROLE warehouse_group NOLOGIN;

GRANT SELECT, INSERT ON order_items TO warehouse_group;
```

ตรวจสอบ:

```sql
SELECT grantee, table_name, privilege_type
FROM information_schema.role_table_grants
WHERE grantee = 'warehouse_group'
ORDER BY privilege_type;
```

ผลลัพธ์ที่คาดหวัง:

```
     grantee      | table_name  | privilege_type
-------------------+-------------+-----------------
 warehouse_group   | order_items | INSERT
 warehouse_group   | order_items | SELECT
```
</details>

---

### แบบฝึกหัดที่ 5

ทีม HR ต้องการดูเฉพาะชื่อ-นามสกุลลูกค้า (ไม่ใช่ email) จากตาราง `customers` จงเขียนคำสั่งให้สิทธิ์แบบ column-level แก่ role ชื่อ `hr_group` แล้วทดสอบว่า `SELECT *` จะทำงานหรือไม่ พร้อมอธิบายเหตุผล

<details>
<summary>เฉลย</summary>

```sql
CREATE ROLE hr_group NOLOGIN;
GRANT SELECT (customer_id, first_name, last_name) ON customers TO hr_group;
```

ทดสอบ:

```sql
CREATE ROLE staff_hr WITH LOGIN PASSWORD 'Hr#2026Pass';
GRANT hr_group TO staff_hr;

SET ROLE staff_hr;
SELECT * FROM customers;   -- จะเกิด error
```

ผลลัพธ์:

```
ERROR:  permission denied for table customers
```

**เหตุผล**: `SELECT *` พยายามอ่านทุกคอลัมน์รวมถึง `email` ซึ่ง `hr_group` ไม่ได้รับอนุญาต เมื่อมีคอลัมน์แม้เพียงคอลัมน์เดียวที่ role ไม่มีสิทธิ์ ทั้ง query จะถูกปฏิเสธทันที ต้องระบุคอลัมน์ที่ได้รับอนุญาตอย่างชัดเจนแทน เช่น `SELECT customer_id, first_name, last_name FROM customers;`

```sql
RESET ROLE;
```
</details>

---

### แบบฝึกหัดที่ 6

อธิบายความแตกต่างระหว่างสิทธิ์ `USAGE` และ `CREATE` บนสคีมา (schema) พร้อมยกตัวอย่างสถานการณ์ที่ role หนึ่งมี `SELECT` บนตารางแล้ว แต่ยังเข้าถึงตารางนั้นไม่ได้

<details>
<summary>เฉลย</summary>

- **`USAGE`** บนสคีมา คือสิทธิ์ที่อนุญาตให้ role "มองเห็น" หรือ "อ้างอิงถึง" ออบเจ็กต์ภายในสคีมานั้นได้ (จำเป็นก่อนเข้าถึงตารางใด ๆ ในสคีมา)
- **`CREATE`** บนสคีมา คือสิทธิ์ที่อนุญาตให้ role สร้างออบเจ็กต์ใหม่ (table, view, function) ภายในสคีมานั้น

ตัวอย่างสถานการณ์: หาก role `X` ได้รับ `GRANT SELECT ON products TO X;` แต่ไม่เคยได้รับ `GRANT USAGE ON SCHEMA public TO X;` เมื่อ `X` พยายาม `SELECT * FROM products;` จะเกิด error:

```
ERROR:  permission denied for schema public
```

เพราะ PostgreSQL ตรวจสอบสิทธิ์ตามลำดับชั้น database → schema → table เสมอ แม้จะมีสิทธิ์ระดับตารางแล้ว แต่ถ้าไม่ผ่านชั้น schema ก็เข้าถึงไม่ได้
</details>

---

### แบบฝึกหัดที่ 7

จงเขียนคำสั่ง `ALTER DEFAULT PRIVILEGES` เพื่อให้ทุกตารางที่ role `app_admin` สร้างขึ้นใหม่ในสคีมา `public` ในอนาคต ให้สิทธิ์ `SELECT` แก่ `PUBLIC` โดยอัตโนมัติ จากนั้นอธิบายว่าทำไมแนวทางนี้อาจไม่ปลอดภัย

<details>
<summary>เฉลย</summary>

```sql
ALTER DEFAULT PRIVILEGES FOR ROLE app_admin IN SCHEMA public
    GRANT SELECT ON TABLES TO PUBLIC;
```

**ความเสี่ยง**: การให้สิทธิ์แก่ `PUBLIC` หมายความว่า role **ทุกตัว** ในคลัสเตอร์ รวมถึง role ที่จะถูกสร้างขึ้นในอนาคต จะสามารถ `SELECT` ข้อมูลจากทุกตารางใหม่ได้ทันทีโดยอัตโนมัติ แม้จะเป็น role ที่ไม่ได้ตั้งใจให้เข้าถึงข้อมูลธุรกิจเลยก็ตาม (เช่น service account ของระบบ monitoring ทั่วไป) ซึ่งขัดกับหลักการ Least Privilege อย่างชัดเจน ควรหลีกเลี่ยงการ grant สิทธิ์ให้ `PUBLIC` เว้นแต่ต้องการให้ "ทุกคนที่เชื่อมต่อได้" เข้าถึงข้อมูลนั้นจริง ๆ (เช่นตาราง lookup ที่ไม่มีข้อมูลอ่อนไหว)
</details>

---

### แบบฝึกหัดที่ 8

ในฐานข้อมูลที่สร้างด้วย PostgreSQL 17 role ทั่วไปที่ไม่ใช่เจ้าของฐานข้อมูล (ไม่ใช่ `postgres`) จะสามารถสร้างตารางในสคีมา `public` ได้หรือไม่โดย default เพราะเหตุใด และควรเสริมความปลอดภัยด้วยคำสั่งใด

<details>
<summary>เฉลย</summary>

**ไม่ได้** ตั้งแต่ PostgreSQL 15 เป็นต้นไป สิทธิ์ `CREATE` บนสคีมา `public` ในฐานข้อมูลที่สร้างใหม่จะ **ไม่ถูกให้แก่ `PUBLIC`** อีกต่อไป มีเพียงเจ้าของฐานข้อมูล (โดยปกติคือ role ที่รัน `CREATE DATABASE`) เท่านั้นที่สร้างออบเจ็กต์ในสคีมา `public` ได้ จนกว่าจะมีการ `GRANT CREATE` ให้ role อื่นอย่างชัดเจน

แม้จะปลอดภัยขึ้นโดย default แล้ว แนวทางที่มืออาชีพยังคงแนะนำให้ยืนยันด้วยตนเองเสมอ (ไม่พึ่งพา default ของเวอร์ชัน) ด้วยคำสั่ง:

```sql
REVOKE CREATE ON SCHEMA public FROM PUBLIC;
```

เพื่อรับประกันพฤติกรรมที่สอดคล้องกันไม่ว่าจะรันบน PostgreSQL เวอร์ชันใด
</details>

---

### แบบฝึกหัดที่ 9

จงออกแบบ role สำหรับ "read-only dashboard user" ที่ต้องการเชื่อมต่อฐานข้อมูล `ecommerce_security`, อ่านข้อมูลได้ทุกตารางในสคีมา `public`, ห้ามสร้าง/แก้ไข/ลบข้อมูลใด ๆ ทั้งสิ้น และต้องมีผลกับตารางใหม่ในอนาคตด้วย (ที่สร้างโดย `postgres`)

<details>
<summary>เฉลย</summary>

```sql
-- 1) สร้าง group role
CREATE ROLE dashboard_readonly NOLOGIN;

-- 2) สิทธิ์ database และ schema
GRANT CONNECT ON DATABASE ecommerce_security TO dashboard_readonly;
GRANT USAGE ON SCHEMA public TO dashboard_readonly;

-- 3) สิทธิ์อ่านทุกตารางที่มีอยู่แล้ว
GRANT SELECT ON ALL TABLES IN SCHEMA public TO dashboard_readonly;

-- 4) ตั้ง default privilege สำหรับตารางในอนาคตที่สร้างโดย postgres
ALTER DEFAULT PRIVILEGES FOR ROLE postgres IN SCHEMA public
    GRANT SELECT ON TABLES TO dashboard_readonly;

-- 5) สร้าง login role จริงแล้วผูกกับกลุ่ม
CREATE ROLE svc_dashboard WITH LOGIN PASSWORD 'Dashboard#2026Secure!' CONNECTION LIMIT 10;
GRANT dashboard_readonly TO svc_dashboard;
```

ไม่จำเป็นต้อง `GRANT INSERT/UPDATE/DELETE` ใด ๆ เพราะ default ของ PostgreSQL คือ "ไม่มีสิทธิ์" อยู่แล้ว การไม่ grant คือการปฏิเสธโดยปริยาย (implicit deny) ซึ่งสอดคล้องกับหลักการ Least Privilege
</details>

---

### แบบฝึกหัดที่ 10

จงออกแบบระบบ role ที่รองรับสถานการณ์นี้: บริษัทมีทีม "Customer Support" ที่ต้องดูข้อมูลคำสั่งซื้อของลูกค้าได้ทั้งหมด และแก้ไขได้เฉพาะคอลัมน์ `status` ในตาราง `orders` เท่านั้น แต่ห้ามเห็นราคาสินค้า (`unit_price` ในตาราง `products`) เพราะเป็นข้อมูลลับทางธุรกิจ (ราคาต้นทุนภายใน) จงเขียนคำสั่ง SQL ทั้งหมดตั้งแต่สร้าง role จนถึงการทดสอบ

<details>
<summary>เฉลย</summary>

```sql
-- 1) สร้าง group role
CREATE ROLE support_team NOLOGIN;

-- 2) สิทธิ์ database และ schema
GRANT CONNECT ON DATABASE ecommerce_security TO support_team;
GRANT USAGE ON SCHEMA public TO support_team;

-- 3) อ่าน orders/order_items/customers ได้เต็มที่ (จำเป็นต่อการช่วยเหลือลูกค้า)
GRANT SELECT ON orders, order_items, customers TO support_team;

-- 4) แก้ไขได้เฉพาะคอลัมน์ status ของ orders
GRANT UPDATE (status) ON orders TO support_team;

-- 5) เข้าถึง products ได้เฉพาะคอลัมน์ที่ไม่ใช่ราคา (ซ่อน unit_price)
GRANT SELECT (product_id, product_name, category_id) ON products TO support_team;

-- 6) สร้าง login role จริง
CREATE ROLE staff_sunisa WITH LOGIN PASSWORD 'Sunisa#2026Support';
GRANT support_team TO staff_sunisa;

-- ทดสอบ
SET ROLE staff_sunisa;

-- อ่านชื่อสินค้าได้ (ไม่มีราคา)
SELECT product_id, product_name FROM products LIMIT 3;

-- พยายามอ่านราคา -> ถูกปฏิเสธ
SELECT unit_price FROM products;
```

```
ERROR:  permission denied for table products
```

```sql
-- แก้ไขสถานะคำสั่งซื้อได้
UPDATE orders SET status = 'refunded' WHERE order_id = 2;
```

```
UPDATE 1
```

```sql
-- พยายามแก้ไข customer_id -> ถูกปฏิเสธ เพราะไม่ได้ grant UPDATE คอลัมน์นี้
UPDATE orders SET customer_id = 3 WHERE order_id = 2;
```

```
ERROR:  permission denied for table orders
```

```sql
RESET ROLE;
```

การออกแบบนี้แสดงให้เห็นการผสมผสาน table-level privilege และ column-level privilege เข้าด้วยกันเพื่อปกป้องทั้งข้อมูลลับทางธุรกิจ (ราคาต้นทุน) และจำกัดขอบเขตการแก้ไขข้อมูลของพนักงานให้อยู่ในหน้าที่ที่รับผิดชอบเท่านั้น ตามหลักการ Least Privilege อย่างสมบูรณ์
</details>

---

## บทถัดไป

เมื่อเข้าใจระบบ Role และ Privilege ระดับตาราง/คอลัมน์/สคีมาแล้ว ขั้นตอนถัดไปคือการควบคุมสิทธิ์ให้ละเอียดถึงระดับ **แถวข้อมูล (row)** ซึ่งจำเป็นอย่างยิ่งสำหรับระบบ multi-tenant หรือระบบที่ลูกค้าแต่ละรายต้องเห็นข้อมูลของตัวเองเท่านั้น

**บทถัดไป**: [Part 069 — Row-Level Security](./part-069-row-level-security.md)
