# Foreign Data Wrappers (postgres_fdw, file_fdw)

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับสูง | Part 056

---

## เป้าหมายการเรียนรู้

เมื่อเรียนจบบทนี้ ผู้เรียนจะสามารถ:

1. อธิบายแนวคิดของ **Foreign Data Wrapper (FDW)** และมาตรฐาน **SQL/MED (Management of External Data)** ที่ PostgreSQL ใช้เป็นพื้นฐาน
2. ติดตั้งและใช้งาน extension `postgres_fdw` เพื่อเชื่อมต่อกับ PostgreSQL server อื่น (หรือจำลองด้วย schema อื่นในฐานข้อมูลเดียวกัน)
3. สร้าง `CREATE SERVER`, `CREATE USER MAPPING` และเข้าใจการทำงานของ authentication ระหว่างเชื่อมต่อ
4. ใช้ `IMPORT FOREIGN SCHEMA` เพื่อนำเข้าตารางจำนวนมากจาก remote schema แบบอัตโนมัติ
5. สร้าง `CREATE FOREIGN TABLE` แบบ manual เพื่อควบคุมคอลัมน์และชนิดข้อมูลอย่างละเอียด
6. Query foreign table ได้เหมือนตารางปกติ พร้อมเข้าใจข้อจำกัดด้าน performance ที่มาจาก network round-trip
7. เข้าใจกลไก **Query Pushdown** ที่ PostgreSQL ส่ง `WHERE`/`JOIN`/`ORDER BY`/aggregate บางส่วนไปประมวลผลที่ remote server เพื่อลดปริมาณข้อมูลที่ต้องโอนผ่านเครือข่าย
8. ใช้ `file_fdw` เพื่ออ่านไฟล์ CSV/text เป็นตาราง (ในเชิงแนวคิดและ syntax)
9. เข้าใจ use case จริงของ FDW เช่น data federation, ETL แบบเบา, การ migrate ข้อมูลระหว่างระบบ และ multi-tenant sharding เบื้องต้น (เชื่อมโยงกับ Citus ใน Part 097)
10. ออกแบบสถาปัตยกรรมที่ใช้ FDW เชื่อมข้อมูลจากหลายฐานข้อมูลในระบบ e-commerce ได้จริง

---

## เตรียมข้อมูล

บทนี้ใช้ schema ฐานข้อมูล e-commerce ที่ใช้ต่อเนื่องมาตลอดหลักสูตร แต่เพื่อสาธิต FDW ให้ทำงานได้จริงในสภาพแวดล้อมเดียว (ไม่ต้องมี PostgreSQL server ตัวที่สองจริง ๆ) เราจะสร้าง **schema ที่สอง** ชื่อ `remote_warehouse` ขึ้นมาในฐานข้อมูลเดียวกัน เพื่อจำลองว่าเป็น "ระบบคลังสินค้า (Warehouse Management System)" ที่แยกฐานข้อมูลออกจากระบบขาย (Sales/e-commerce) — ซึ่งเป็นสถานการณ์ที่พบบ่อยมากในองค์กรจริงที่ทีมต่างกันดูแลระบบต่างกัน

### 1. Schema หลัก (ระบบขาย/e-commerce) — เก็บใน schema `public`

```sql
-- ล้างของเก่า (ถ้ามี) เพื่อให้รันซ้ำได้
DROP SCHEMA IF EXISTS public CASCADE;
CREATE SCHEMA public;

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
    unit_price      NUMERIC(10,2) NOT NULL,
    stock_quantity  INTEGER NOT NULL DEFAULT 0
);

CREATE TABLE customers (
    customer_id     SERIAL PRIMARY KEY,
    first_name      VARCHAR(60) NOT NULL,
    last_name       VARCHAR(60) NOT NULL,
    email           VARCHAR(150) UNIQUE,
    country         VARCHAR(60)
);

CREATE TABLE orders (
    order_id        SERIAL PRIMARY KEY,
    customer_id     INTEGER REFERENCES customers(customer_id),
    order_date      TIMESTAMPTZ NOT NULL DEFAULT now(),
    status          VARCHAR(20) NOT NULL DEFAULT 'pending'
);
```

### 2. ข้อมูลตัวอย่าง (seed data)

```sql
INSERT INTO categories (category_name) VALUES
    ('อุปกรณ์อิเล็กทรอนิกส์'), ('เสื้อผ้าแฟชั่น'), ('ของใช้ในบ้าน'),
    ('หนังสือ'), ('ของเล่นและเกม'), ('อุปกรณ์กีฬา'),
    ('เครื่องสำอาง'), ('อาหารและเครื่องดื่ม'), ('เฟอร์นิเจอร์'),
    ('อุปกรณ์สำนักงาน'), ('เครื่องใช้ไฟฟ้าในครัว'), ('อุปกรณ์กลางแจ้ง');

INSERT INTO suppliers (supplier_name, country) VALUES
    ('Bangkok Electronics Co.', 'Thailand'),
    ('Global Textile Partners', 'Vietnam'),
    ('HomeStyle Manufacturing', 'Thailand'),
    ('Pacific Book Distributors', 'Singapore'),
    ('FunTime Toys Ltd.', 'China'),
    ('ProSport Equipment', 'Thailand'),
    ('Beauty Origin Co.', 'South Korea'),
    ('FreshFood Trading', 'Thailand'),
    ('Nordic Furniture House', 'Sweden'),
    ('OfficePro Supplies', 'Thailand'),
    ('KitchenMaster Appliances', 'Japan'),
    ('OutdoorLife Gear', 'Thailand'),
    ('Siam Digital Imports', 'Thailand'),
    ('EverGreen Living', 'Malaysia');

INSERT INTO products (product_name, category_id, supplier_id, unit_price, stock_quantity) VALUES
    ('หูฟังไร้สาย Bluetooth 5.3', 1, 1, 890.00, 150),
    ('เพาเวอร์แบงค์ 20000mAh', 1, 13, 650.00, 200),
    ('เสื้อยืดคอตตอน 100%', 2, 2, 259.00, 500),
    ('กางเกงยีนส์ทรงสลิม', 2, 2, 890.00, 300),
    ('หมอนอิงโซฟา', 3, 3, 350.00, 120),
    ('ผ้าปูที่นอนไมโครไฟเบอร์', 3, 3, 690.00, 180),
    ('นิยายวิทยาศาสตร์ปกแข็ง', 4, 4, 420.00, 90),
    ('หนังสือคู่มือ SQL เบื้องต้น', 4, 4, 350.00, 140),
    ('หุ่นยนต์ของเล่นบังคับวิทยุ', 5, 5, 1290.00, 60),
    ('เซตบล็อกตัวต่อเสริมทักษะ', 5, 5, 780.00, 110),
    ('ลูกฟุตบอลมาตรฐานแข่งขัน', 6, 6, 590.00, 95),
    ('เสื่อโยคะกันลื่น', 6, 6, 450.00, 220),
    ('ลิปสติกแมทต์เนื้อบาง', 7, 7, 320.00, 400),
    ('ครีมกันแดด SPF50', 7, 7, 280.00, 350),
    ('กาแฟคั่วบดพรีเมียม 250g', 8, 8, 210.00, 300),
    ('ชาเขียวมัทฉะออร์แกนิก', 8, 8, 260.00, 180),
    ('โต๊ะทำงานไม้โอ๊ค', 9, 9, 4590.00, 25),
    ('เก้าอี้สำนักงานเพื่อสุขภาพ', 9, 9, 3990.00, 40),
    ('ปากกาหมึกซึมพรีเมียม', 10, 10, 150.00, 500),
    ('เครื่องปิ้งขนมปัง 2 ช่อง', 11, 11, 990.00, 70);

INSERT INTO customers (first_name, last_name, email, country) VALUES
    ('สมชาย', 'ใจดี', 'somchai.j@example.com', 'Thailand'),
    ('สมหญิง', 'รักเรียน', 'somying.r@example.com', 'Thailand'),
    ('วิชัย', 'แสงทอง', 'wichai.s@example.com', 'Thailand'),
    ('มาลี', 'ศรีสุข', 'malee.s@example.com', 'Thailand'),
    ('ประยุทธ', 'มั่นคง', 'prayuth.m@example.com', 'Thailand'),
    ('นิภา', 'พงษ์ไพร', 'nipa.p@example.com', 'Thailand'),
    ('อนุชา', 'ทองแท้', 'anucha.t@example.com', 'Thailand'),
    ('กมลวรรณ', 'สายชล', 'kamolwan.s@example.com', 'Thailand'),
    ('ธีรพงษ์', 'บุญมี', 'teerapong.b@example.com', 'Thailand'),
    ('John', 'Smith', 'john.smith@example.com', 'United States'),
    ('Emily', 'Johnson', 'emily.j@example.com', 'United Kingdom'),
    ('Nguyen', 'Van An', 'van.an.n@example.com', 'Vietnam'),
    ('Lisa', 'Müller', 'lisa.m@example.com', 'Germany'),
    ('Haruto', 'Sato', 'haruto.s@example.com', 'Japan'),
    ('Siti', 'Rahman', 'siti.r@example.com', 'Malaysia'),
    ('ปิยะดา', 'วงศ์ษา', 'piyada.w@example.com', 'Thailand');

INSERT INTO orders (customer_id, order_date, status) VALUES
    (1, now() - interval '30 days', 'delivered'),
    (2, now() - interval '28 days', 'delivered'),
    (3, now() - interval '25 days', 'delivered'),
    (1, now() - interval '20 days', 'delivered'),
    (4, now() - interval '18 days', 'shipped'),
    (5, now() - interval '15 days', 'shipped'),
    (6, now() - interval '12 days', 'processing'),
    (7, now() - interval '10 days', 'processing'),
    (2, now() - interval '9 days', 'processing'),
    (8, now() - interval '7 days', 'pending'),
    (10, now() - interval '6 days', 'pending'),
    (11, now() - interval '5 days', 'pending'),
    (9, now() - interval '4 days', 'cancelled'),
    (12, now() - interval '3 days', 'pending'),
    (3, now() - interval '2 days', 'pending'),
    (13, now() - interval '1 days', 'pending');
```

### 3. Schema ที่สอง — จำลองระบบคลังสินค้าแยกฐานข้อมูล (`remote_warehouse`)

ในองค์กรจริง ระบบคลังสินค้า (Warehouse Management System - WMS) มักถูกดูแลโดยทีมอื่น และอยู่คนละฐานข้อมูล (หรือแม้แต่คนละ PostgreSQL server) จากระบบขาย เราจะสร้าง schema `remote_warehouse` ขึ้นมาในฐานข้อมูลเดียวกันนี้ เพื่อ **จำลองสถานการณ์นั้นให้ทดสอบได้จริง** — ตารางในนี้จะถือว่าเป็น "remote database" ที่เราจะเชื่อมเข้าไปด้วย `postgres_fdw`

```sql
CREATE SCHEMA IF NOT EXISTS remote_warehouse;

-- ตารางระดับสต็อกจริงในคลัง (อาจไม่ตรงกับ stock_quantity ฝั่งขายเป๊ะ ๆ เพราะคนละระบบ)
CREATE TABLE remote_warehouse.stock_levels (
    warehouse_sku     VARCHAR(30) PRIMARY KEY,
    product_ref_id    INTEGER NOT NULL,     -- อ้างอิง product_id ฝั่งระบบขาย (ไม่มี FK ข้ามฐานข้อมูลจริง)
    warehouse_code    VARCHAR(10) NOT NULL,
    quantity_on_hand  INTEGER NOT NULL,
    quantity_reserved INTEGER NOT NULL DEFAULT 0,
    last_counted_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ตารางการจัดส่งจากคลังไปยังลูกค้า
CREATE TABLE remote_warehouse.warehouse_shipments (
    shipment_id       SERIAL PRIMARY KEY,
    order_ref_id      INTEGER NOT NULL,     -- อ้างอิง order_id ฝั่งระบบขาย
    warehouse_code    VARCHAR(10) NOT NULL,
    shipped_at        TIMESTAMPTZ,
    carrier           VARCHAR(50),
    tracking_number   VARCHAR(50)
);

INSERT INTO remote_warehouse.stock_levels
    (warehouse_sku, product_ref_id, warehouse_code, quantity_on_hand, quantity_reserved, last_counted_at) VALUES
    ('WH-SKU-0001', 1, 'BKK-01', 142, 12, now() - interval '2 days'),
    ('WH-SKU-0002', 2, 'BKK-01', 205, 30, now() - interval '2 days'),
    ('WH-SKU-0003', 3, 'BKK-02', 480, 20, now() - interval '1 days'),
    ('WH-SKU-0004', 4, 'BKK-02', 290, 15, now() - interval '1 days'),
    ('WH-SKU-0005', 5, 'CNX-01', 115, 5,  now() - interval '3 days'),
    ('WH-SKU-0006', 6, 'CNX-01', 170, 8,  now() - interval '3 days'),
    ('WH-SKU-0007', 7, 'BKK-01', 88,  2,  now() - interval '5 days'),
    ('WH-SKU-0008', 8, 'BKK-01', 135, 10, now() - interval '5 days'),
    ('WH-SKU-0009', 9, 'CNX-02', 55,  4,  now() - interval '4 days'),
    ('WH-SKU-0010', 10,'CNX-02', 105, 6,  now() - interval '4 days'),
    ('WH-SKU-0011', 11,'BKK-02', 90,  3,  now() - interval '2 days'),
    ('WH-SKU-0012', 12,'BKK-02', 210, 18, now() - interval '2 days');

INSERT INTO remote_warehouse.warehouse_shipments
    (order_ref_id, warehouse_code, shipped_at, carrier, tracking_number) VALUES
    (1, 'BKK-01', now() - interval '28 days', 'Kerry Express', 'KE1000000001'),
    (2, 'BKK-01', now() - interval '26 days', 'Flash Express', 'FE2000000002'),
    (3, 'BKK-02', now() - interval '23 days', 'Kerry Express', 'KE1000000003'),
    (4, 'BKK-02', now() - interval '18 days', 'Thailand Post', 'TP3000000004'),
    (5, 'CNX-01', now() - interval '15 days', 'Flash Express', 'FE2000000005'),
    (6, 'CNX-01', now() - interval '12 days', 'Kerry Express', 'KE1000000006');
```

> **หมายเหตุสำคัญ:** ตัวอย่าง `postgres_fdw` ทั้งหมดในบทนี้เชื่อมต่อกลับเข้ามาที่ **ฐานข้อมูลเดียวกัน** ผ่าน `localhost` เพียงแต่ข้ามไปยัง schema `remote_warehouse` เพื่อจำลองว่าเป็น "remote server" — วิธีนี้ทำให้ทุกคำสั่งสามารถรันทดสอบได้จริงโดยไม่ต้องมี PostgreSQL instance ตัวที่สอง แนวคิดและ syntax เหมือนกันทุกประการกับการเชื่อมต่อไปยังเซิร์ฟเวอร์จริงที่อยู่คนละเครื่อง คนละ data center หรือแม้แต่คนละ cloud provider

---

## Step 551: Foreign Data Wrapper (FDW) คืออะไร

**Foreign Data Wrapper (FDW)** คือกลไกใน PostgreSQL ที่ทำให้เราสามารถ **query ข้อมูลจากแหล่งข้อมูลภายนอก** (external data source) ได้ราวกับว่ามันเป็นตารางธรรมดาใน PostgreSQL เอง แหล่งข้อมูลภายนอกนั้นอาจเป็น:

- PostgreSQL server อีกเครื่องหนึ่ง (ผ่าน `postgres_fdw`)
- ไฟล์ CSV/text บนดิสก์ (ผ่าน `file_fdw`)
- ฐานข้อมูลอื่น เช่น MySQL, Oracle, SQL Server, MongoDB (ผ่าน FDW ของ community เช่น `mysql_fdw`, `oracle_fdw`, `mongo_fdw`)
- REST API หรือบริการ cloud เช่น Redis, S3 (ผ่าน FDW ของ third-party)

### มาตรฐาน SQL/MED

FDW ใน PostgreSQL implement ตามมาตรฐาน **SQL/MED (SQL Management of External Data)** ซึ่งเป็นส่วนขยายของมาตรฐาน SQL:2003 ที่นิยามวิธีมาตรฐานในการเข้าถึงข้อมูลนอกฐานข้อมูลผ่านภาษา SQL เดียวกัน โดยมีองค์ประกอบหลัก 4 ส่วนที่ต้องรู้จัก:

| องค์ประกอบ | หน้าที่ |
|---|---|
| **Foreign Data Wrapper** | driver/extension ที่รู้วิธีคุยกับแหล่งข้อมูลภายนอกชนิดหนึ่ง ๆ |
| **Foreign Server** | การตั้งค่าการเชื่อมต่อไปยังแหล่งข้อมูลหนึ่งจุด (host, port, database) |
| **User Mapping** | การแม็ป local role กับ credential ที่ใช้ล็อกอินฝั่ง remote |
| **Foreign Table** | ตารางเสมือนที่ผูกกับข้อมูลจริงในฝั่ง remote |

### ทำไมต้องใช้ FDW

1. **Data federation** — รวมข้อมูลจากหลายระบบเข้าด้วยกันโดยไม่ต้อง copy ข้อมูลจริง
2. **ลดความซับซ้อนของ ETL** — query ข้าม system ได้ตรง ๆ ผ่าน SQL แทนที่จะเขียน pipeline แยก
3. **Migration แบบค่อยเป็นค่อยไป** — ย้ายระบบทีละส่วนโดยยังอ้างอิงข้อมูลเก่าผ่าน FDW ได้
4. **Sharding เบื้องต้น** — กระจายข้อมูลไปหลาย node แล้ว query รวมผ่าน foreign table (รากฐานของ Citus ที่จะเรียนใน Part 097)

```sql
-- ตรวจสอบว่า FDW ที่ติดตั้งมากับ PostgreSQL core มีอะไรบ้าง
SELECT fdwname, fdwhandler::regproc, fdwvalidator::regproc
FROM pg_foreign_data_wrapper;
```

ก่อนติดตั้ง extension ใด ๆ คำสั่งนี้อาจไม่คืนแถวเลย เพราะ FDW ต้องถูกลงทะเบียนผ่าน `CREATE EXTENSION` หรือ `CREATE FOREIGN DATA WRAPPER` ก่อนเสมอ

### ตรวจสอบ extension ที่มีให้ใช้ในระบบ

```sql
SELECT name, default_version, installed_version, comment
FROM pg_available_extensions
WHERE name IN ('postgres_fdw', 'file_fdw')
ORDER BY name;
```

ผลลัพธ์ควรแสดงทั้ง `postgres_fdw` และ `file_fdw` เป็น extension ที่มากับ PostgreSQL core ทุก distribution มาตรฐาน (ไม่ต้องติดตั้งเพิ่มจากภายนอก)

---

## Step 552: CREATE EXTENSION postgres_fdw

`postgres_fdw` เป็น FDW ที่ทำให้ PostgreSQL server หนึ่งสามารถ query ข้อมูลจาก PostgreSQL server อื่น (หรือฐานข้อมูล/schema อื่น) ได้ มันเป็น FDW ที่ **สมบูรณ์แบบที่สุด** ในบรรดา FDW ทั้งหมด เพราะรองรับทั้ง read และ write, รองรับ query pushdown ระดับสูง (WHERE, JOIN, ORDER BY, LIMIT, aggregate) และรองรับ transaction/savepoint ข้าม server

### การติดตั้ง

```sql
-- โดยทั่วไปต้องเป็น superuser หรือ role ที่มีสิทธิ์ CREATEDB/CREATE บน database
CREATE EXTENSION IF NOT EXISTS postgres_fdw;
```

ตรวจสอบว่าติดตั้งสำเร็จ:

```sql
SELECT extname, extversion
FROM pg_extension
WHERE extname = 'postgres_fdw';
```

> **ข้อควรระวังเรื่องสิทธิ์:** `postgres_fdw` ไม่ใช่ "trusted extension" (ต่างจาก `pg_stat_statements` บางเวอร์ชัน) ดังนั้นโดยดีฟอลต์การ `CREATE EXTENSION postgres_fdw` ต้องทำโดย superuser หรือ role ที่ได้รับสิทธิ์ `pg_database_owner`/`CREATE` บน database นั้นอย่างชัดเจน ผู้ดูแลระบบสามารถ `GRANT` การใช้งาน FDW handler ให้ role อื่นภายหลังได้ผ่าน `GRANT USAGE ON FOREIGN DATA WRAPPER postgres_fdw TO some_role;`

### แนวคิดการเชื่อมต่อ (จำลองด้วย schema เดียวกัน)

ปกติเมื่อจะเชื่อม PostgreSQL server A ไปยัง PostgreSQL server B เราต้องมี:

- Server B ต้องเปิดรับ connection จาก network ที่ A อยู่ (`listen_addresses`, firewall)
- ไฟล์ `pg_hba.conf` ของ B ต้องอนุญาต host/user ที่จะเชื่อมเข้ามา
- ต้องมี credential (username/password หรือ certificate) สำหรับ authentication

ในบทนี้เราจำลองโดยเชื่อมกลับเข้า **`localhost`** ของฐานข้อมูลเดียวกัน — ซึ่งมักจะผ่าน authentication ได้ทันทีเพราะ `pg_hba.conf` ดีฟอลต์อนุญาต local connection อยู่แล้ว วิธีนี้ทำให้เราทดสอบทุกคำสั่งได้จริง โดย syntax และพฤติกรรมเหมือนกับการเชื่อมข้าม server จริงทุกประการ

```sql
-- ตรวจสอบชื่อฐานข้อมูลปัจจุบัน (จะใช้อ้างอิงตอนสร้าง SERVER)
SELECT current_database();
```

สมมติว่าผลลัพธ์คือ `ecommerce_db` เราจะใช้ชื่อนี้ในขั้นตอนถัดไป

---

## Step 553: CREATE SERVER และ CREATE USER MAPPING

### CREATE SERVER

`CREATE SERVER` คือการนิยาม "จุดเชื่อมต่อ" หนึ่งจุดไปยังแหล่งข้อมูลภายนอก โดยระบุว่าใช้ FDW ตัวไหน และมี option การเชื่อมต่ออะไรบ้าง (host, port, dbname เป็นต้น)

```sql
CREATE SERVER warehouse_srv
    FOREIGN DATA WRAPPER postgres_fdw
    OPTIONS (
        host 'localhost',
        port '5432',
        dbname 'ecommerce_db'   -- แก้เป็นชื่อฐานข้อมูลจริงของคุณจาก current_database()
    );
```

ตรวจสอบ server ที่สร้างไว้:

```sql
SELECT srvname, srvoptions
FROM pg_foreign_server
WHERE srvname = 'warehouse_srv';
```

> ในสถานการณ์จริง หาก remote server อยู่คนละเครื่อง ให้เปลี่ยน `host` เป็น IP หรือ hostname จริง และ `port` ตามที่ PostgreSQL ฝั่งนั้นเปิดรับ (ดีฟอลต์ 5432) นอกจากนี้ยังตั้งค่า option เสริมได้ เช่น `sslmode 'require'`, `connect_timeout '10'`, `fetch_size '10000'` (ควบคุมจำนวนแถวต่อรอบที่ดึงจาก remote server มา cache ฝั่ง local)

### CREATE USER MAPPING

`USER MAPPING` คือการบอกว่า **local role** คนไหน จะใช้ credential อะไรตอนล็อกอินเข้า remote server ผ่าน server ที่สร้างไว้

```sql
-- แม็ป role ปัจจุบัน (current_user) เข้ากับ user/password ฝั่ง remote
CREATE USER MAPPING FOR CURRENT_USER
    SERVER warehouse_srv
    OPTIONS (
        user 'postgres',        -- username ที่จะใช้ล็อกอินฝั่ง remote (ในที่นี้คือฐานข้อมูลเดียวกัน)
        password 'your_password_here'
    );
```

> **หมายเหตุด้าน authentication:** ถ้า `pg_hba.conf` ตั้งค่า method เป็น `trust` สำหรับ local connection (พบได้บ่อยในสภาพแวดล้อมพัฒนา/ทดสอบ) การใส่ `password` อาจไม่จำเป็นเลยก็ได้ แต่การระบุไว้ไม่ทำให้เกิดปัญหา เพราะ PostgreSQL จะยึดตาม `pg_hba.conf` ของฝั่ง remote เป็นหลักเสมอ ในสภาพแวดล้อม production ควรใช้ method ที่ปลอดภัยกว่า เช่น `scram-sha-256` พร้อม password จริง หรือ SSL client certificate

ตรวจสอบ user mapping ที่สร้างไว้ (ไม่แสดง password จริงเพื่อความปลอดภัย):

```sql
SELECT srvname, usename, umoptions
FROM pg_user_mappings
WHERE srvname = 'warehouse_srv';
```

### GRANT สิทธิ์การใช้งาน server ให้ role อื่น

ถ้าต้องการให้ role อื่น (ที่ไม่ใช่คนสร้าง server) ใช้งาน server นี้ได้ ต้อง `GRANT USAGE`:

```sql
GRANT USAGE ON FOREIGN SERVER warehouse_srv TO sales_app_role;
```

โดย `sales_app_role` เองก็ยังต้องมี `USER MAPPING` ของตัวเองด้วย มิเช่นนั้นจะเชื่อมต่อไม่ได้แม้จะมีสิทธิ์ `USAGE` บน server แล้วก็ตาม

---

## Step 554: IMPORT FOREIGN SCHEMA

เมื่อ remote schema มีตารางจำนวนมาก การเขียน `CREATE FOREIGN TABLE` ทีละตารางเป็นเรื่องน่าเบื่อและเสี่ยงพิมพ์คอลัมน์ผิด PostgreSQL จึงมีคำสั่ง **`IMPORT FOREIGN SCHEMA`** ที่จะไปสำรวจโครงสร้างตารางทั้งหมดใน remote schema แล้วสร้าง foreign table ให้อัตโนมัติ พร้อมคอลัมน์และชนิดข้อมูลที่ตรงกัน

### สร้าง local schema ไว้รับ foreign table

```sql
CREATE SCHEMA IF NOT EXISTS fdw_warehouse;
```

### Import ทั้ง schema

```sql
IMPORT FOREIGN SCHEMA remote_warehouse
    FROM SERVER warehouse_srv
    INTO fdw_warehouse;
```

คำสั่งนี้จะสร้าง foreign table `fdw_warehouse.stock_levels` และ `fdw_warehouse.warehouse_shipments` ให้อัตโนมัติ โดยมีคอลัมน์ตรงกับต้นฉบับทุกประการ

ตรวจสอบผลลัพธ์:

```sql
SELECT foreign_table_schema, foreign_table_name
FROM information_schema.foreign_tables
WHERE foreign_table_schema = 'fdw_warehouse'
ORDER BY foreign_table_name;
```

```sql
-- ดูโครงสร้างคอลัมน์ของ foreign table ที่ import มา
\d fdw_warehouse.stock_levels
```

### Import แบบเลือกเฉพาะบางตาราง (LIMIT TO)

```sql
IMPORT FOREIGN SCHEMA remote_warehouse
    LIMIT TO (stock_levels)
    FROM SERVER warehouse_srv
    INTO fdw_warehouse;
```

### Import แบบยกเว้นบางตาราง (EXCEPT)

```sql
IMPORT FOREIGN SCHEMA remote_warehouse
    EXCEPT (warehouse_shipments)
    FROM SERVER warehouse_srv
    INTO fdw_warehouse;
```

> **ข้อดี** ของ `IMPORT FOREIGN SCHEMA` คือรวดเร็วและแม่นยำเมื่อ remote schema มีตารางเป็นสิบเป็นร้อยตาราง เหมาะกับ initial setup หรือเมื่อ schema ฝั่ง remote มีการเปลี่ยนแปลงบ่อย
>
> **ข้อจำกัด** คือมันไม่ยืดหยุ่นเท่าการ `CREATE FOREIGN TABLE` แบบ manual — ไม่สามารถเลือก subset ของคอลัมน์ หรือแปลงชนิดข้อมูลระหว่างทางได้ (ต้องใช้ `IMPORT FOREIGN SCHEMA` ตามด้วย `ALTER FOREIGN TABLE ... DROP COLUMN` ถ้าต้องการตัดบางคอลัมน์ทิ้งภายหลัง)

---

## Step 555: CREATE FOREIGN TABLE แบบ manual

บางครั้งเราต้องการควบคุมโครงสร้าง foreign table เอง เช่น ต้องการเฉพาะบางคอลัมน์, ต้องการแปลงชนิดข้อมูล, หรือต้องการตั้งชื่อคอลัมน์ให้ต่างจากต้นฉบับ — กรณีเหล่านี้ใช้ `CREATE FOREIGN TABLE` โดยตรง

### Syntax พื้นฐาน

```sql
CREATE FOREIGN TABLE fdw_warehouse.stock_levels_manual (
    warehouse_sku     VARCHAR(30),
    product_ref_id    INTEGER,
    warehouse_code    VARCHAR(10),
    quantity_on_hand  INTEGER,
    quantity_reserved INTEGER,
    last_counted_at   TIMESTAMPTZ
)
SERVER warehouse_srv
OPTIONS (
    schema_name 'remote_warehouse',
    table_name  'stock_levels'
);
```

ทดสอบ query:

```sql
SELECT * FROM fdw_warehouse.stock_levels_manual ORDER BY warehouse_sku LIMIT 5;
```

### เลือกเฉพาะบางคอลัมน์และเปลี่ยนชื่อคอลัมน์

สมมติทีมขายต้องการเห็นเฉพาะ "จำนวนพร้อมขาย" (คำนวณจาก on_hand - reserved) โดยไม่สนใจ field อื่น เราสามารถสร้าง foreign table ที่ map เฉพาะคอลัมน์ที่ต้องการ พร้อมตั้งชื่อ local column ต่างจาก remote column ได้ด้วย option `column_name`:

```sql
CREATE FOREIGN TABLE fdw_warehouse.stock_summary (
    sku              VARCHAR(30)   OPTIONS (column_name 'warehouse_sku'),
    product_id       INTEGER       OPTIONS (column_name 'product_ref_id'),
    on_hand          INTEGER       OPTIONS (column_name 'quantity_on_hand'),
    reserved         INTEGER       OPTIONS (column_name 'quantity_reserved')
)
SERVER warehouse_srv
OPTIONS (
    schema_name 'remote_warehouse',
    table_name  'stock_levels'
);

SELECT sku, product_id, on_hand - reserved AS available_qty
FROM fdw_warehouse.stock_summary
ORDER BY available_qty DESC
LIMIT 5;
```

### Option ระดับ foreign table ที่ควรรู้จัก

| Option | ความหมาย |
|---|---|
| `schema_name` | ชื่อ schema ของตารางจริงฝั่ง remote |
| `table_name` | ชื่อตารางจริงฝั่ง remote (ถ้าไม่ระบุจะใช้ชื่อเดียวกับ foreign table) |
| `column_name` | (ระดับคอลัมน์) แม็ปชื่อคอลัมน์ local ไปยังชื่อคอลัมน์จริงฝั่ง remote |
| `updatable` | ถ้าตั้งเป็น `'false'` foreign table นั้นจะเป็น read-only แม้ server จะรองรับ write ก็ตาม |

```sql
-- ตัวอย่างการทำให้ foreign table เป็น read-only อย่างชัดเจน
CREATE FOREIGN TABLE fdw_warehouse.shipments_readonly (
    shipment_id     INTEGER,
    order_ref_id    INTEGER,
    warehouse_code  VARCHAR(10),
    shipped_at      TIMESTAMPTZ,
    carrier         VARCHAR(50),
    tracking_number VARCHAR(50)
)
SERVER warehouse_srv
OPTIONS (
    schema_name 'remote_warehouse',
    table_name  'warehouse_shipments',
    updatable   'false'
);
```

---

## Step 556: การ Query Foreign Table

จุดเด่นที่สุดของ FDW คือเมื่อสร้างเสร็จแล้ว **เรา query foreign table ได้เหมือนตารางปกติทุกประการ** ไม่ต้องเขียน syntax พิเศษใด ๆ เพิ่มเติม

### ตัวอย่างการ JOIN foreign table กับ local table

ลองหาว่าสินค้าตัวไหนที่ "ขายดีในระบบ order" แต่ "สต็อกในคลังเหลือน้อย" — คำถามนี้ต้องรวมข้อมูลจาก 2 ระบบเข้าด้วยกัน ซึ่งปกติต้องทำ ETL แต่ด้วย FDW เรา join ตรง ๆ ได้เลย:

```sql
SELECT
    p.product_id,
    p.product_name,
    p.stock_quantity        AS sales_system_stock,
    w.quantity_on_hand      AS warehouse_on_hand,
    w.quantity_reserved     AS warehouse_reserved,
    (w.quantity_on_hand - w.quantity_reserved) AS available_in_warehouse
FROM products p
JOIN fdw_warehouse.stock_levels w
    ON w.product_ref_id = p.product_id
WHERE (w.quantity_on_hand - w.quantity_reserved) < 100
ORDER BY available_in_warehouse ASC;
```

### ตัวอย่างการดึงประวัติการจัดส่งของลูกค้าคนหนึ่ง

```sql
SELECT
    o.order_id,
    c.first_name || ' ' || c.last_name AS customer_name,
    o.status,
    s.carrier,
    s.tracking_number,
    s.shipped_at
FROM orders o
JOIN customers c ON c.customer_id = o.customer_id
LEFT JOIN fdw_warehouse.warehouse_shipments s
    ON s.order_ref_id = o.order_id
WHERE c.customer_id = 1
ORDER BY o.order_date;
```

### ข้อจำกัดด้าน Performance — Network Round-Trip

แม้ query จะเขียนได้เหมือนตารางปกติ แต่เบื้องหลังนั้น **ต่างกันมาก**:

1. **Latency ต่อ round-trip** — ทุกครั้งที่ local server ต้องขอข้อมูลจาก remote server จะมี network latency เกิดขึ้น แม้จะเป็น localhost latency ก็ยังมากกว่าการอ่าน buffer ในเครื่องเดียวกันตรง ๆ ถ้าเป็นการเชื่อมข้าม data center latency นี้อาจสูงถึงหลักสิบหรือหลักร้อยมิลลิวินาทีต่อ round-trip
2. **ไม่มี index บน foreign table โดยตรง** — planner ฝั่ง local ไม่เห็น index ของตารางจริงฝั่ง remote นอกจาก `postgres_fdw` จะ "ส่งเงื่อนไข" (pushdown) ไปให้ remote planner ใช้ index ของมันเอง (จะอธิบายละเอียดใน Step 557)
3. **Cost estimation ไม่แม่นยำ 100%** — planner ฝั่ง local ต้องประมาณ cost ของการ query remote table จาก statistics ที่ import เข้ามา (ผ่าน `ANALYZE`) ซึ่งอาจไม่ตรงกับความเป็นจริง ทำให้ query plan บางครั้งไม่เหมาะสม
4. **Transaction overhead** — ถ้า foreign table participate ใน transaction เดียวกับ local table, PostgreSQL ต้องเปิด remote transaction คู่ขนานและซิงค์ commit/rollback ให้ตรงกัน เพิ่ม overhead

```sql
-- รัน ANALYZE บน foreign table เพื่อให้ planner มี statistics ที่แม่นยำขึ้น
ANALYZE fdw_warehouse.stock_levels;
```

```sql
-- ดู execution plan เพื่อสังเกตว่ามี remote round-trip เกิดขึ้นตรงไหน
EXPLAIN (VERBOSE, ANALYZE)
SELECT * FROM fdw_warehouse.stock_levels WHERE warehouse_code = 'BKK-01';
```

สังเกตว่า plan จะมี node ชื่อ **`Foreign Scan`** ซึ่งจะแสดง `Remote SQL:` บอกว่า PostgreSQL ส่ง SQL อะไรไปให้ remote server รันจริง ๆ — นี่คือหัวใจของ Step ถัดไป

### แนวทางลด overhead

- ใช้ `fetch_size` option ที่เหมาะสม (ค่าดีฟอลต์คือ 100 แถวต่อรอบ) เพื่อลดจำนวน round-trip เมื่อดึงข้อมูลจำนวนมาก
- หลีกเลี่ยงการ `SELECT *` ถ้าไม่จำเป็น เพราะ column ที่ไม่ใช้ก็ยังถูกโอนผ่านเครือข่ายเว้นแต่ planner ตัดออกได้
- พิจารณาใช้ **materialized view** ครอบ foreign table ถ้าข้อมูลไม่จำเป็นต้อง real-time (จะเรียนละเอียดใน Part เกี่ยวกับ Materialized View)

```sql
-- ตัวอย่าง fetch_size ที่ปรับต่อ foreign table ได้เช่นกัน
ALTER FOREIGN TABLE fdw_warehouse.stock_levels
    OPTIONS (SET fetch_size '5000');
```

---

## Step 557: Query Pushdown

**Query Pushdown** คือความสามารถของ `postgres_fdw` ในการ "ส่งบางส่วนของ query" ไปให้ remote server ประมวลผลแทนที่จะดึงข้อมูลทั้งหมดมากรองที่ local — นี่คือสิ่งที่ทำให้ `postgres_fdw` แตกต่างจาก FDW แบบพื้นฐานอื่น ๆ อย่างมาก

### WHERE Pushdown

```sql
EXPLAIN (VERBOSE)
SELECT * FROM fdw_warehouse.stock_levels
WHERE warehouse_code = 'BKK-01' AND quantity_on_hand > 100;
```

ใน `Remote SQL:` ของ plan จะเห็นว่า PostgreSQL แปลง query เป็นประมาณนี้ก่อนส่งไปรันที่ remote:

```sql
SELECT warehouse_sku, product_ref_id, warehouse_code,
       quantity_on_hand, quantity_reserved, last_counted_at
FROM remote_warehouse.stock_levels
WHERE ((warehouse_code = 'BKK-01')) AND ((quantity_on_hand > 100))
```

นั่นแปลว่า remote server เป็นผู้กรองข้อมูลให้เสร็จก่อน แล้วส่งเฉพาะแถวที่ตรงเงื่อนไขกลับมาเท่านั้น — ลดปริมาณข้อมูลที่ต้องโอนผ่านเครือข่ายได้มาก โดยเฉพาะเมื่อตารางต้นทางมีข้อมูลนับล้านแถวแต่ผลลัพธ์ที่ต้องการมีเพียงหลักสิบหรือหลักร้อยแถว

### JOIN Pushdown (สำคัญมาก)

ถ้า **ทั้งสองตาราง** ที่ join กันเป็น foreign table บน **server เดียวกัน** และ **user mapping เดียวกัน** PostgreSQL จะสามารถส่งทั้ง JOIN ไปทำที่ remote server ได้เลย โดยไม่ต้องดึงทั้งสองตารางมา join ที่ local

```sql
EXPLAIN (VERBOSE)
SELECT s.warehouse_sku, s.quantity_on_hand, sh.carrier, sh.tracking_number
FROM fdw_warehouse.stock_levels s
JOIN fdw_warehouse.warehouse_shipments sh
    ON sh.warehouse_code = s.warehouse_code
WHERE s.quantity_on_hand > 50;
```

ถ้าเงื่อนไขครบถ้วน (server เดียวกัน, user mapping เดียวกัน, ไม่มี function ที่ evaluate ได้เฉพาะฝั่ง local ปนอยู่) plan จะแสดง node เดียวเป็น **`Foreign Scan`** ที่มี `Relations:` บอกว่าเป็นการ join ของทั้งสองตาราง และ `Remote SQL:` จะเป็นคำสั่ง `SELECT ... FROM remote_warehouse.stock_levels ... INNER JOIN remote_warehouse.warehouse_shipments ...` ที่สมบูรณ์ — นี่คือกรณีที่ดีที่สุด เพราะ **JOIN ทั้งหมดถูกทำที่ remote server จุดเดียว มีเพียงผลลัพธ์สุดท้ายเท่านั้นที่ถูกส่งกลับมา**

> เทียบกับกรณีที่ตารางฝั่งหนึ่งเป็น local table (เช่น `products` join กับ `fdw_warehouse.stock_levels`) — กรณีนี้ join **ไม่สามารถ pushdown ได้ทั้งหมด** เพราะ local table ไม่ได้อยู่บน remote server เดียวกัน สิ่งที่เกิดขึ้นคือ WHERE condition ที่เกี่ยวกับ foreign table เพียงอย่างเดียวจะถูก pushdown ไปกรองที่ remote ก่อน แล้วผลลัพธ์ที่ถูกกรองแล้วนั้นถึงจะถูกดึงมา join กับ local table ที่ local server

### Aggregate Pushdown

ตั้งแต่ PostgreSQL 10 เป็นต้นมา `postgres_fdw` ยังรองรับการส่ง aggregate function บางส่วนไปประมวลผลที่ remote server ได้ด้วย (เช่น `COUNT`, `SUM`, `AVG`, `MIN`, `MAX` แบบง่าย ๆ ที่ไม่ผสมกับ expression ซับซ้อนเกินไป)

```sql
EXPLAIN (VERBOSE)
SELECT warehouse_code, SUM(quantity_on_hand) AS total_stock, COUNT(*) AS sku_count
FROM fdw_warehouse.stock_levels
GROUP BY warehouse_code;
```

ถ้า aggregate pushdown ทำงาน `Remote SQL:` จะแสดง `SELECT warehouse_code, sum(quantity_on_hand), count(*) FROM remote_warehouse.stock_levels GROUP BY warehouse_code` — หมายความว่าการรวมยอด (aggregation) เกิดขึ้นที่ remote server ทั้งหมด local server เพียงรับผลสรุปมาแสดงเท่านั้น

### ORDER BY / LIMIT Pushdown

```sql
EXPLAIN (VERBOSE)
SELECT * FROM fdw_warehouse.stock_levels
ORDER BY quantity_on_hand DESC
LIMIT 3;
```

`ORDER BY` และ `LIMIT` มักถูก pushdown ไปด้วยเช่นกัน หมายความว่า remote server จะเรียงลำดับและตัดจำนวนแถวให้เสร็จก่อนส่งกลับมา แทนที่จะดึงข้อมูลทั้งหมดมาเรียงที่ local

### ปัจจัยที่ทำให้ pushdown "ไม่เกิดขึ้น"

1. ใช้ function ฝั่ง local ที่ remote server ไม่รู้จัก หรือมี **collation** ต่างกัน
2. ใช้ operator หรือ data type ที่ extension ฝั่ง remote ไม่มี (เช่น custom type ที่ยังไม่ได้ติดตั้งฝั่ง remote)
3. มี `volatile` function ปะปนใน WHERE clause (เช่น `random()`, `now()` ในบาง context) เพราะ semantics ของ evaluation อาจเปลี่ยนไปถ้า evaluate ที่ remote
4. Query มี CTE หรือ subquery ที่ผสม local table เข้ามาก่อนถึงชั้น filter

```sql
-- ตรวจสอบว่า extension ที่จำเป็นถูกติดตั้งครบทั้งสองฝั่งหรือไม่ (มีผลต่อ pushdown ของ custom operator)
SELECT extname FROM pg_extension ORDER BY extname;
```

**บทเรียนสำคัญ:** เวลาทำงานกับ FDW ควรใช้ `EXPLAIN (VERBOSE)` เช็ค `Remote SQL:` เสมอ เพื่อยืนยันว่า pushdown เกิดขึ้นจริงตามที่คาดหวัง ไม่ใช่แค่เชื่อว่ามันจะฉลาดพอเอง

---

## Step 558: file_fdw — การอ่านไฟล์ CSV/text เป็นตาราง

> **หมายเหตุสำคัญก่อนเริ่ม:** ตัวอย่างใน Step นี้เป็น **ตัวอย่างเชิงแนวคิด (conceptual example)** เพื่อสาธิต syntax และวิธีคิดของ `file_fdw` เท่านั้น เนื่องจากสภาพแวดล้อมสำหรับบทเรียนนี้ไม่มีไฟล์จริงบนดิสก์ให้ทดสอบ ผู้เรียนสามารถนำ syntax นี้ไปทดลองจริงในเครื่องของตัวเองได้ โดยสร้างไฟล์ CSV ตามที่อธิบายไว้ก่อน

### สถานการณ์สมมติ

ทีมจัดซื้อได้รับไฟล์ `supplier_price_list.csv` จาก supplier ภายนอกทุกสัปดาห์ (ส่งมาทาง email หรือ SFTP) มีเนื้อหาประมาณนี้:

```
supplier_sku,product_name,unit_cost,currency,valid_from
SUP-1001,หูฟังไร้สาย Bluetooth 5.3,650.00,THB,2026-09-01
SUP-1002,เพาเวอร์แบงค์ 20000mAh,480.00,THB,2026-09-01
SUP-1003,เสื้อยืดคอตตอน 100%,180.00,THB,2026-09-01
SUP-1004,กางเกงยีนส์ทรงสลิม,610.00,THB,2026-09-01
SUP-1005,หมอนอิงโซฟา,220.00,THB,2026-09-01
```

แทนที่จะเขียนสคริปต์แยกเพื่อ import ไฟล์นี้เข้าฐานข้อมูลทุกครั้งที่ได้รับ เราสามารถใช้ `file_fdw` เพื่อ "มองไฟล์นี้เป็นตาราง" ได้โดยตรง — ทุกครั้งที่ query ตาราง PostgreSQL จะอ่านไฟล์สด ๆ จากดิสก์ (ไม่มีการ cache ข้อมูลไว้ล่วงหน้า)

### ติดตั้ง extension (แนวคิด)

```sql
CREATE EXTENSION IF NOT EXISTS file_fdw;
```

### สร้าง server สำหรับ file_fdw (แนวคิด)

`file_fdw` ไม่ได้เชื่อมต่อผ่านเครือข่าย จึงไม่ต้องมี `host`/`port` เหมือน `postgres_fdw` — สร้าง server แบบไม่มี option เพิ่มเติมได้เลย:

```sql
CREATE SERVER file_srv
    FOREIGN DATA WRAPPER file_fdw;
```

`file_fdw` ไม่ต้องใช้ `USER MAPPING` เพราะไม่มีการ authenticate ไปยังระบบอื่น (การเข้าถึงไฟล์ถูกควบคุมด้วยสิทธิ์ระดับ OS/PostgreSQL role แทน)

### สร้าง foreign table ชี้ไปยังไฟล์ CSV (แนวคิด — how it would look)

```sql
-- ตัวอย่างเชิงแนวคิด: สมมติว่าไฟล์อยู่ที่ /var/lib/postgresql/imports/supplier_price_list.csv
-- บนเครื่องจริง path นี้ต้องเป็น path ที่ PostgreSQL server process อ่านได้ (ไม่ใช่ path ฝั่ง client)
CREATE FOREIGN TABLE fdw_warehouse.supplier_price_list (
    supplier_sku    VARCHAR(30),
    product_name    VARCHAR(150),
    unit_cost       NUMERIC(10,2),
    currency        VARCHAR(5),
    valid_from      DATE
)
SERVER file_srv
OPTIONS (
    filename        '/var/lib/postgresql/imports/supplier_price_list.csv',
    format           'csv',
    header           'true',
    delimiter        ',',
    null              ''
);
```

เมื่อไฟล์นี้มีอยู่จริงและ path ถูกต้อง เราจะ query ได้เหมือนตารางปกติ:

```sql
-- ตัวอย่างเชิงแนวคิด — จะรันได้จริงเมื่อมีไฟล์ตามที่ระบุใน OPTIONS เท่านั้น
SELECT * FROM fdw_warehouse.supplier_price_list
WHERE unit_cost > 500
ORDER BY unit_cost DESC;
```

### Option สำคัญของ file_fdw

| Option | ความหมาย |
|---|---|
| `filename` | path เต็มของไฟล์บนเครื่อง server (ต้องเป็น absolute path) |
| `format` | รูปแบบไฟล์ เช่น `csv`, `text`, `binary` (เหมือน option ของคำสั่ง `COPY`) |
| `header` | ถ้าเป็น `csv` และไฟล์มีแถวหัวตาราง ให้ตั้งเป็น `'true'` เพื่อข้ามแถวแรก |
| `delimiter` | ตัวคั่นแต่ละ field (ดีฟอลต์ของ CSV คือ comma) |
| `null` | string ที่แทนค่า NULL ในไฟล์ (เช่น string ว่าง หรือ `\N`) |
| `quote` | อักขระที่ใช้ครอบ field ที่มีตัวคั่นปนอยู่ (ดีฟอลต์ `"`) |
| `encoding` | encoding ของไฟล์ เช่น `UTF8` |

### ข้อจำกัดสำคัญของ file_fdw

1. **Read-only เสมอ** — ไม่สามารถ `INSERT`/`UPDATE`/`DELETE` ผ่าน foreign table ชนิดนี้ได้ (ต่างจาก `postgres_fdw` ที่รองรับ write)
2. **สิทธิ์การเข้าถึงไฟล์** — โดยดีฟอลต์เฉพาะ superuser หรือ role ที่มี privilege พิเศษ (เช่น `pg_read_server_files`) เท่านั้นที่ `CREATE FOREIGN TABLE` แบบระบุ `filename` ได้ เพราะเป็นการเปิดช่องให้อ่านไฟล์ใด ๆ บนเครื่อง server ซึ่งมีความเสี่ยงด้านความปลอดภัยหากเปิดให้ทุกคนทำได้
3. **ไม่มี index** — ทุกครั้งที่ query ต้องอ่านทั้งไฟล์ (full scan) ไม่มีทางทำ index scan บนไฟล์ text ได้
4. **ไฟล์ต้องอยู่บนเครื่อง PostgreSQL server เอง** ไม่ใช่เครื่อง client — ถ้า client อยู่คนละเครื่องกับ server ต้อง copy ไฟล์ไปไว้ที่เครื่อง server ก่อน
5. **ไม่ตรวจจับการเปลี่ยนแปลงไฟล์อัตโนมัติ** ระหว่าง query เดียว (แต่ query ครั้งถัดไปจะอ่านเนื้อหาล่าสุดเสมอ เพราะไม่มี caching)

### เมื่อไหร่ควรใช้ file_fdw จริง ๆ

- Log analysis เบื้องต้นจากไฟล์ text/CSV ที่มีอยู่แล้วบนเครื่อง server โดยไม่ต้อง import
- Staging area ชั่วคราวสำหรับตรวจสอบข้อมูลก่อนตัดสินใจ `INSERT ... SELECT` เข้าตารางจริง
- สถานการณ์ ad-hoc ที่ได้รับไฟล์ dump มาแล้วต้องการ query อย่างรวดเร็วโดยไม่อยาก `COPY` เข้าตารางถาวร

---

## Step 559: Use Case จริงของ FDW

FDW ไม่ใช่แค่ของเล่นทางเทคนิค แต่ถูกใช้งานจริงในระบบ production หลายรูปแบบ:

### 1. Data Federation — รวมข้อมูลจากหลายระบบโดยไม่ต้อง copy

องค์กรขนาดใหญ่มักมีหลายฐานข้อมูลที่แยกกันตามทีม/domain เช่น ระบบขาย, ระบบคลังสินค้า, ระบบบัญชี, ระบบ CRM แต่ละระบบมี database ของตัวเอง เมื่อต้องการรายงานที่รวมข้อมูลข้ามระบบ (cross-domain reporting) FDW ช่วยให้ query ข้ามระบบได้โดยตรงโดยไม่ต้องสร้าง data warehouse หรือ pipeline ซับซ้อนสำหรับ use case ที่ไม่ต้องการ real-time มากนัก

```sql
-- ตัวอย่าง: รายงานยอดขายรวมกับสถานะสต็อกจริง สำหรับทีมบริหาร
SELECT
    c.category_name,
    COUNT(DISTINCT p.product_id)      AS product_count,
    SUM(p.stock_quantity)             AS sales_system_stock,
    SUM(w.quantity_on_hand)           AS warehouse_actual_stock
FROM categories c
JOIN products p ON p.category_id = c.category_id
LEFT JOIN fdw_warehouse.stock_levels w ON w.product_ref_id = p.product_id
GROUP BY c.category_name
ORDER BY warehouse_actual_stock DESC NULLS LAST;
```

### 2. ETL แบบเบา (Lightweight ETL)

แทนที่จะเขียนสคริปต์ Python/Airflow แยกต่างหากเพื่อดึงข้อมูลจากระบบ A มาลง B ทุกคืน สำหรับ dataset ขนาดไม่ใหญ่มาก เราสามารถใช้ FDW ร่วมกับ `INSERT INTO ... SELECT FROM foreign_table` ได้ตรง ๆ ผ่าน SQL เดียว:

```sql
-- ตัวอย่าง: ซิงค์ข้อมูลการจัดส่งจากระบบคลังเข้ามาเก็บ snapshot รายวันในระบบขาย
CREATE TABLE IF NOT EXISTS shipment_snapshot (
    shipment_id     INTEGER PRIMARY KEY,
    order_ref_id    INTEGER,
    warehouse_code  VARCHAR(10),
    shipped_at      TIMESTAMPTZ,
    carrier         VARCHAR(50),
    tracking_number VARCHAR(50),
    synced_at       TIMESTAMPTZ NOT NULL DEFAULT now()
);

INSERT INTO shipment_snapshot (shipment_id, order_ref_id, warehouse_code, shipped_at, carrier, tracking_number)
SELECT shipment_id, order_ref_id, warehouse_code, shipped_at, carrier, tracking_number
FROM fdw_warehouse.warehouse_shipments
ON CONFLICT (shipment_id) DO UPDATE
    SET shipped_at = EXCLUDED.shipped_at,
        carrier = EXCLUDED.carrier,
        tracking_number = EXCLUDED.tracking_number,
        synced_at = now();
```

วิธีนี้เหมาะกับข้อมูลขนาดกลาง (พัน–แสนแถวต่อรอบ) ถ้าข้อมูลใหญ่ระดับล้านแถวขึ้นไปทุกวัน ควรพิจารณาเครื่องมือ ETL เฉพาะทาง (เช่น Airflow, dbt, Debezium) แทน เพราะ FDW ไม่ได้ optimize สำหรับ bulk transfer ขนาดใหญ่เท่า `COPY` โดยตรง

### 3. Migration ข้อมูลระหว่างระบบแบบค่อยเป็นค่อยไป

เวลาย้ายระบบเก่าไปสู่ระบบใหม่ (เช่น เปลี่ยน schema, เปลี่ยน database version, หรือแม้แต่เปลี่ยน database engine) มักไม่สามารถ cutover ได้ในคราวเดียว FDW ช่วยให้:

- ระบบใหม่ query ข้อมูลบางส่วนที่ยังไม่ได้ migrate จากระบบเก่าผ่าน foreign table ไปพลาง ๆ
- ทำ **dual-write / dual-read** ระหว่างช่วง transition โดยไม่ต้องหยุดระบบ
- ตรวจสอบความถูกต้องของข้อมูลที่ migrate แล้ว โดยเทียบกับต้นฉบับผ่าน query เดียวที่ join ทั้งสองฝั่ง

```sql
-- ตัวอย่าง: เทียบจำนวนแถวระหว่างตารางเก่า (ผ่าน FDW) กับตารางใหม่ที่ migrate แล้ว เพื่อยืนยันความถูกต้อง
SELECT
    (SELECT COUNT(*) FROM fdw_warehouse.stock_levels) AS old_system_count,
    (SELECT COUNT(*) FROM remote_warehouse.stock_levels) AS new_system_count;
```

### 4. Multi-tenant Sharding เบื้องต้น

เมื่อระบบ multi-tenant เติบโตจนฐานข้อมูลเดียวรับโหลดไม่ไหว แนวทางหนึ่งคือแบ่ง tenant ออกเป็นหลายฐานข้อมูล (sharding) แล้วใช้ foreign table เป็น "หน้าต่าง" ให้ query รวมข้อมูลข้าม shard ได้ในกรณีที่จำเป็นต้องดูภาพรวม (เช่น รายงานระดับ admin):

```sql
-- แนวคิด: แต่ละ tenant/region อาจอยู่คนละ database/server
-- shard 1: ลูกค้าในประเทศไทย, shard 2: ลูกค้าต่างประเทศ เป็นต้น
-- CREATE SERVER shard_th ... / CREATE SERVER shard_intl ...
-- แล้ว UNION ALL ผ่าน foreign table ของแต่ละ shard เพื่อดู aggregate รวม
```

แนวทางนี้เป็นจุดเริ่มต้นแนวคิดของการทำ **distributed PostgreSQL** ด้วยมือ (manual sharding) ซึ่งมีข้อจำกัดมากเมื่อจำนวน shard เพิ่มขึ้น (ต้องจัดการ routing, rebalancing, cross-shard transaction เอง) — ใน **Part 097 เราจะเรียนรู้ Citus** ซึ่งเป็น extension ที่สร้างขึ้นบนแนวคิดเดียวกับ FDW (distributed foreign table + intelligent query planner) แต่ทำให้การ sharding อัตโนมัติ โปร่งใส และ scale ได้ดีกว่าการทำ manual sharding ด้วย FDW เปล่า ๆ มาก

### สรุปเปรียบเทียบเมื่อไหร่ควรใช้ FDW เทียบกับทางเลือกอื่น

| สถานการณ์ | FDW เหมาะสม? | ทางเลือกอื่นที่ควรพิจารณา |
|---|---|---|
| Query ข้ามระบบเป็นครั้งคราว, ข้อมูลไม่ใหญ่มาก | เหมาะสม | - |
| Real-time dashboard ที่ query ถี่มาก | ต้องระวัง latency | Materialized view + refresh schedule |
| Bulk transfer ข้อมูลล้านแถวทุกวัน | ไม่เหมาะ | `COPY`, logical replication, dedicated ETL tool |
| ต้องการ scale เขียน/อ่านข้ามหลาย node จริงจัง | จุดเริ่มต้นเท่านั้น | Citus (Part 097), sharding ระดับ application |
| อ่านไฟล์ CSV ที่ได้รับเป็นครั้งคราว | เหมาะสม | `COPY` ถ้าต้องการ import ถาวรเข้าตาราง |

---

## Step 560: แบบฝึกหัดรวม — ออกแบบสถาปัตยกรรมที่ใช้ FDW

**โจทย์:** บริษัท e-commerce แห่งหนึ่งมี 3 ระบบแยกฐานข้อมูลกัน:

1. **Sales DB** — เก็บ `categories`, `suppliers`, `products`, `customers`, `orders` (ตารางที่เราใช้ตลอดบทนี้)
2. **Warehouse DB** — เก็บ `stock_levels`, `warehouse_shipments` (จำลองด้วย `remote_warehouse` schema)
3. **Analytics DB** (สมมติเพิ่มเติม) — เก็บสรุปยอดขายรายวันสำหรับทีมผู้บริหาร ต้องการดึงข้อมูลจากทั้ง Sales DB และ Warehouse DB มาสรุปทุกคืน

ให้ออกแบบว่า Analytics DB ควรตั้งค่า FDW อย่างไร และเขียน SQL ตัวอย่างสำหรับสร้างตารางสรุปที่ดึงข้อมูลข้าม 2 ระบบ

### แนวทางการออกแบบ

```sql
-- (รันบน Analytics DB ในสถานการณ์จริง — ในบทนี้จำลองด้วย schema เดียวกัน)
CREATE EXTENSION IF NOT EXISTS postgres_fdw;

CREATE SERVER sales_srv
    FOREIGN DATA WRAPPER postgres_fdw
    OPTIONS (host 'sales-db.internal', port '5432', dbname 'sales_db');

CREATE SERVER warehouse_srv_prod
    FOREIGN DATA WRAPPER postgres_fdw
    OPTIONS (host 'warehouse-db.internal', port '5432', dbname 'warehouse_db');

CREATE USER MAPPING FOR analytics_etl_role
    SERVER sales_srv
    OPTIONS (user 'readonly_reporter', password 'xxxxx');

CREATE USER MAPPING FOR analytics_etl_role
    SERVER warehouse_srv_prod
    OPTIONS (user 'readonly_reporter', password 'xxxxx');

CREATE SCHEMA fdw_sales;
CREATE SCHEMA fdw_warehouse_prod;

IMPORT FOREIGN SCHEMA public
    LIMIT TO (orders, customers, products)
    FROM SERVER sales_srv
    INTO fdw_sales;

IMPORT FOREIGN SCHEMA public
    FROM SERVER warehouse_srv_prod
    INTO fdw_warehouse_prod;

-- ตารางสรุปที่รันทุกคืนผ่าน cron job / pg_cron
CREATE TABLE daily_sales_warehouse_summary (
    summary_date        DATE PRIMARY KEY,
    total_orders         INTEGER,
    total_products_sold  INTEGER,
    total_warehouse_stock INTEGER,
    generated_at         TIMESTAMPTZ NOT NULL DEFAULT now()
);

INSERT INTO daily_sales_warehouse_summary (summary_date, total_orders, total_products_sold, total_warehouse_stock)
SELECT
    CURRENT_DATE,
    (SELECT COUNT(*) FROM fdw_sales.orders WHERE order_date::date = CURRENT_DATE),
    (SELECT COUNT(*) FROM fdw_sales.products),
    (SELECT SUM(quantity_on_hand) FROM fdw_warehouse_prod.stock_levels)
ON CONFLICT (summary_date) DO UPDATE
    SET total_orders = EXCLUDED.total_orders,
        total_products_sold = EXCLUDED.total_products_sold,
        total_warehouse_stock = EXCLUDED.total_warehouse_stock,
        generated_at = now();
```

### ประเด็นสถาปัตยกรรมที่ต้องพิจารณา

1. **Read-only role ฝั่ง remote** — สร้าง role เฉพาะสำหรับ reporting ที่มีสิทธิ์ `SELECT` เท่านั้น ไม่ใช้ role ที่มีสิทธิ์เขียนข้อมูล production
2. **Network security** — เชื่อมต่อผ่าน private network/VPC เท่านั้น ไม่เปิด FDW connection ผ่าน public internet โดยไม่มี SSL/VPN
3. **เวลาที่รัน sync** — เลือกช่วงที่โหลดของระบบต้นทางต่ำ (เช่น กลางดึก) เพื่อลดผลกระทบต่อระบบ production
4. **Timeout และ retry** — ตั้ง `connect_timeout` และเขียน error handling ในกรณี remote server ไม่ตอบสนอง (เช่นใช้ `pg_cron` + retry logic)
5. **Monitoring** — เฝ้าดู `pg_stat_activity` ทั้งสองฝั่งเพื่อดูว่า cross-database query ใช้เวลานานผิดปกติหรือไม่
6. **แนวโน้มการเติบโต** — ถ้า Analytics DB ต้อง query ข้าม FDW บ่อยขึ้นเรื่อย ๆ และ dataset โตขึ้นมาก ควรพิจารณาเปลี่ยนไปใช้ logical replication หรือ dedicated data warehouse (เช่น ส่งข้อมูลเข้า columnar store) แทนการพึ่ง FDW ตลอดไป

---

## สรุปท้ายบท

- **Foreign Data Wrapper (FDW)** คือกลไกตามมาตรฐาน **SQL/MED** ที่ทำให้ PostgreSQL เข้าถึงข้อมูลภายนอกได้เหมือนตารางปกติ ผ่านองค์ประกอบหลัก 4 อย่าง: **Foreign Data Wrapper, Foreign Server, User Mapping, Foreign Table**
- **`postgres_fdw`** เป็น FDW ที่สมบูรณ์แบบที่สุด ใช้เชื่อม PostgreSQL server หนึ่งไปยังอีก server หนึ่ง รองรับทั้ง read/write และ query pushdown ระดับสูง
- การตั้งค่าต้องผ่าน `CREATE EXTENSION` → `CREATE SERVER` → `CREATE USER MAPPING` → `IMPORT FOREIGN SCHEMA` (อัตโนมัติ) หรือ `CREATE FOREIGN TABLE` (manual, ควบคุมได้ละเอียดกว่า)
- **Query pushdown** (WHERE, JOIN, aggregate, ORDER BY/LIMIT) คือหัวใจของ performance ที่ดีเมื่อใช้ FDW — ต้องตรวจสอบด้วย `EXPLAIN (VERBOSE)` และดู `Remote SQL:` เสมอเพื่อยืนยันว่า pushdown เกิดขึ้นจริง
- ข้อจำกัดหลักของ FDW คือ **network round-trip latency**, การไม่เห็น index ฝั่ง remote โดยตรง, และ cost estimation ที่อาจไม่แม่นยำ
- **`file_fdw`** ใช้อ่านไฟล์ CSV/text เป็นตารางแบบ read-only เหมาะกับข้อมูลที่ไม่ต้อง import ถาวร แต่ต้องระวังเรื่องสิทธิ์การเข้าถึงไฟล์และไม่มี index
- Use case จริงของ FDW ครอบคลุม **data federation, lightweight ETL, migration แบบค่อยเป็นค่อยไป และจุดเริ่มต้นของ multi-tenant sharding** — ซึ่งจะนำไปสู่การเรียนรู้ **Citus** ใน Part 097 สำหรับการ scale แบบ distributed PostgreSQL อย่างจริงจัง

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1
ให้อธิบายความแตกต่างระหว่าง `CREATE SERVER`, `CREATE USER MAPPING`, และ `CREATE FOREIGN TABLE` — แต่ละคำสั่งรับผิดชอบส่วนไหนของการเชื่อมต่อ?

<details>
<summary>เฉลย</summary>

- `CREATE SERVER` — นิยามจุดเชื่อมต่อไปยัง instance ภายนอกหนึ่งจุด (host, port, database) ผูกกับ FDW ชนิดหนึ่ง
- `CREATE USER MAPPING` — กำหนดว่า local role คนไหนใช้ credential (username/password) อะไรตอนล็อกอินเข้า server นั้น
- `CREATE FOREIGN TABLE` — นิยามตารางเสมือนฝั่ง local ที่ผูกกับตารางจริงฝั่ง remote (ผ่าน server ที่สร้างไว้) พร้อมระบุคอลัมน์และชนิดข้อมูล

ลำดับการสร้างต้องเป็น: extension → server → user mapping → foreign table (หรือ import foreign schema แทน foreign table)
</details>

### แบบฝึกหัดที่ 2
เขียน SQL เพื่อสร้าง foreign table ชื่อ `fdw_warehouse.stock_levels_v2` ที่ดึงเฉพาะคอลัมน์ `warehouse_sku`, `quantity_on_hand` จากตาราง `remote_warehouse.stock_levels` โดยตั้งชื่อ `quantity_on_hand` เป็น `on_hand_qty` ในฝั่ง local

<details>
<summary>เฉลย</summary>

```sql
CREATE FOREIGN TABLE fdw_warehouse.stock_levels_v2 (
    warehouse_sku  VARCHAR(30),
    on_hand_qty    INTEGER OPTIONS (column_name 'quantity_on_hand')
)
SERVER warehouse_srv
OPTIONS (
    schema_name 'remote_warehouse',
    table_name  'stock_levels'
);
```
</details>

### แบบฝึกหัดที่ 3
`IMPORT FOREIGN SCHEMA` กับ `CREATE FOREIGN TABLE` แบบ manual ต่างกันอย่างไร ควรเลือกใช้แบบไหนเมื่อไหร่?

<details>
<summary>เฉลย</summary>

- `IMPORT FOREIGN SCHEMA` เหมาะกับกรณีที่ต้องการนำเข้าตารางจำนวนมากอย่างรวดเร็ว โดยให้คอลัมน์ตรงกับต้นฉบับทุกประการ เหมาะกับ initial setup หรือ schema ที่มีตารางเยอะและเปลี่ยนแปลงบ่อย
- `CREATE FOREIGN TABLE` แบบ manual เหมาะเมื่อต้องการควบคุมโครงสร้างเอง เช่น เลือกเฉพาะบางคอลัมน์, เปลี่ยนชื่อคอลัมน์, จำกัดสิทธิ์เป็น read-only (`updatable 'false'`), หรือต้องการความชัดเจนของ contract ระหว่างระบบโดยไม่ผูกกับการเปลี่ยนแปลง schema ฝั่ง remote โดยอัตโนมัติ
</details>

### แบบฝึกหัดที่ 4
จากตัวอย่างในบทเรียน เขียน query ที่ join `products` (local) กับ `fdw_warehouse.stock_levels` (foreign) เพื่อหาสินค้าที่ "สต็อกในระบบขายมากกว่าสต็อกจริงในคลัง" (อาจบ่งบอกว่าข้อมูลไม่ sync กัน)

<details>
<summary>เฉลย</summary>

```sql
SELECT
    p.product_id,
    p.product_name,
    p.stock_quantity   AS sales_system_stock,
    w.quantity_on_hand AS warehouse_actual_stock,
    p.stock_quantity - w.quantity_on_hand AS discrepancy
FROM products p
JOIN fdw_warehouse.stock_levels w
    ON w.product_ref_id = p.product_id
WHERE p.stock_quantity > w.quantity_on_hand
ORDER BY discrepancy DESC;
```
</details>

### แบบฝึกหัดที่ 5
อธิบายว่าทำไม JOIN ระหว่าง foreign table สองตารางบน server เดียวกันถึง pushdown ได้สมบูรณ์กว่า JOIN ระหว่าง local table กับ foreign table

<details>
<summary>เฉลย</summary>

เมื่อทั้งสองตารางเป็น foreign table ที่ผูกกับ **server เดียวกัน** และ **user mapping เดียวกัน** PostgreSQL รู้ว่าข้อมูลทั้งสองฝั่งอยู่ที่ remote instance เดียวกันจริง ๆ จึงสามารถแปลงทั้ง JOIN เป็น SQL เดียวแล้วส่งไปให้ remote server รันได้ทั้งหมด (ผลลัพธ์เดียวถูกส่งกลับ)

แต่ถ้าฝั่งหนึ่งเป็น local table ข้อมูลของมันไม่ได้อยู่ที่ remote instance นั้น จึงไม่มีทางส่งทั้ง JOIN ไปให้ remote ทำได้ — สิ่งที่ทำได้คือ pushdown เฉพาะ WHERE condition ที่เกี่ยวกับ foreign table เพียงอย่างเดียว แล้วดึงผลลัพธ์ (ที่กรองแล้ว) มา join กับ local table ที่ local server อีกที
</details>

### แบบฝึกหัดที่ 6
เขียน query ที่ใช้ `EXPLAIN (VERBOSE)` เพื่อตรวจสอบว่า aggregate `SUM(quantity_reserved)` แยกตาม `warehouse_code` ถูก pushdown ไปที่ remote server หรือไม่ และบอกว่าต้องดูอะไรใน output

<details>
<summary>เฉลย</summary>

```sql
EXPLAIN (VERBOSE)
SELECT warehouse_code, SUM(quantity_reserved) AS total_reserved
FROM fdw_warehouse.stock_levels
GROUP BY warehouse_code;
```

ต้องดูบรรทัด `Remote SQL:` ใน output — ถ้า aggregate pushdown เกิดขึ้นจริง จะเห็นว่า remote SQL มีคำว่า `sum(quantity_reserved)` และ `GROUP BY warehouse_code` อยู่ในนั้นด้วย (หมายความว่า remote server เป็นผู้คำนวณผลรวมให้เสร็จก่อนส่งกลับ) ถ้า pushdown ไม่เกิดขึ้น local server จะต้องดึงทุกแถวมาคำนวณเอง ซึ่งจะเห็น node แยกเป็น `Aggregate` ที่อยู่เหนือ `Foreign Scan` แทนที่จะรวมอยู่ใน remote SQL เดียวกัน
</details>

### แบบฝึกหัดที่ 7
`file_fdw` มีข้อจำกัดอะไรบ้างเมื่อเทียบกับ `postgres_fdw`? ยกมาอย่างน้อย 3 ข้อ

<details>
<summary>เฉลย</summary>

1. **Read-only เสมอ** — ไม่รองรับ `INSERT`/`UPDATE`/`DELETE` ในขณะที่ `postgres_fdw` รองรับ write ได้
2. **ไม่มี index** — ทุก query ต้องอ่านทั้งไฟล์ (full scan) เพราะไฟล์ text ไม่มีโครงสร้าง index
3. **ไฟล์ต้องอยู่บนเครื่อง PostgreSQL server เอง** ไม่ใช่เครื่อง client ทำให้ใช้งานยากถ้า client กับ server อยู่คนละเครื่อง
4. **ต้องการสิทธิ์พิเศษ** (superuser หรือ `pg_read_server_files`) ในการระบุ `filename` เพราะมีความเสี่ยงด้านความปลอดภัย
5. **ไม่รองรับ pushdown ที่ซับซ้อน** เท่า `postgres_fdw` เพราะไม่มี "remote planner" ที่จะรับ SQL ไปแปลความหมายต่อ — `file_fdw` แค่ scan ไฟล์ทั้งหมดแล้วกรองที่ local เสมอ
</details>

### แบบฝึกหัดที่ 8
บริษัทหนึ่งต้องการ sync ข้อมูลจากระบบ warehouse เข้าระบบขายทุกคืน แต่ข้อมูลมีขนาดหลายล้านแถวต่อวัน ทีมงานเสนอให้ใช้ `postgres_fdw` กับ `INSERT INTO ... SELECT` ตรง ๆ ทุกคืน — คุณจะให้คำแนะนำอย่างไร?

<details>
<summary>เฉลย</summary>

ไม่แนะนำให้ใช้ FDW เป็นกลไกหลักสำหรับ bulk transfer ขนาดใหญ่ระดับล้านแถวต่อวัน เพราะ:

- FDW ทำงานผ่าน network round-trip และ query executor ปกติ ไม่ได้ optimize สำหรับการโอนข้อมูลจำนวนมหาศาลเท่ากับ `COPY` หรือ logical replication
- การ query แบบนี้อาจใช้เวลานานและกิน resource ของทั้งสองฝั่งระหว่างการ sync จนกระทบ query อื่นที่รันพร้อมกัน
- ควรพิจารณาทางเลือกที่เหมาะกับ scale นี้มากกว่า เช่น **logical replication** (ถ้าต้องการ sync ต่อเนื่องแบบ near real-time), **`pg_dump`/`COPY`** แบบ batch (ถ้า sync เป็นรอบ ๆ ไม่ต้อง real-time), หรือเครื่องมือ ETL เฉพาะทาง (Airflow, dbt) ที่ควบคุม batch size, retry, และ monitoring ได้ดีกว่า
- FDW ยังเหมาะกับ use case ที่เป็น query แบบ ad-hoc หรือข้อมูลขนาดกลาง ไม่ใช่ bulk pipeline หลักของระบบ
</details>

### แบบฝึกหัดที่ 9
เขียนคำสั่ง SQL เพื่อ `GRANT USAGE` บน foreign server `warehouse_srv` ให้ role ชื่อ `reporting_role` และอธิบายว่าทำไม role นั้นยังต้องมี `USER MAPPING` ของตัวเองด้วย

<details>
<summary>เฉลย</summary>

```sql
GRANT USAGE ON FOREIGN SERVER warehouse_srv TO reporting_role;

CREATE USER MAPPING FOR reporting_role
    SERVER warehouse_srv
    OPTIONS (user 'readonly_user', password 'xxxxx');
```

`GRANT USAGE ON FOREIGN SERVER` เป็นเพียงการอนุญาตให้ role นั้น "รู้จักและใช้ definition ของ server" ได้ (เช่น สร้าง foreign table อ้างอิง server นี้ได้) แต่ **ไม่ได้บอกว่าจะล็อกอินฝั่ง remote ด้วย credential อะไร** — ถ้าไม่มี `USER MAPPING` ของตัวเอง เมื่อ role นั้นพยายาม query foreign table จริง ๆ ระบบจะหา credential ไม่เจอและ error ทันที ดังนั้นทั้งสองสิทธิ์ (`GRANT USAGE` + `USER MAPPING`) ต้องมีคู่กันเสมอ
</details>

### แบบฝึกหัดที่ 10
ออกแบบสถาปัตยกรรมคร่าว ๆ (เขียนเป็นข้อความ หรือ pseudo-SQL) สำหรับระบบที่ต้องรวมข้อมูลจาก 3 ฐานข้อมูลย่อย: Sales DB, Warehouse DB, และ Payment DB (ระบบชำระเงินแยกต่างหาก) เข้าเป็น Analytics DB เดียว พร้อมระบุว่าจุดไหนควรใช้ FDW และจุดไหนไม่ควร

<details>
<summary>เฉลย</summary>

**แนวทางออกแบบ:**

```
Analytics DB
 ├── CREATE SERVER sales_srv     → Sales DB   (postgres_fdw)
 ├── CREATE SERVER warehouse_srv → Warehouse DB (postgres_fdw)
 └── (ไม่สร้าง FDW ไปยัง Payment DB โดยตรง)
```

- **Sales DB และ Warehouse DB** — ข้อมูลไม่อ่อนไหวด้านความปลอดภัยสูงมาก ปริมาณข้อมูลอยู่ในระดับที่ query ผ่าน FDW ไหว และทีม Analytics ต้องการความยืดหยุ่นในการ query ad-hoc ข้ามระบบบ่อย ๆ → **เหมาะกับ FDW** ผ่าน `postgres_fdw` พร้อม read-only role เฉพาะสำหรับ reporting

- **Payment DB** — ข้อมูลการชำระเงินมักมีความอ่อนไหวสูง (PCI-DSS compliance, ข้อมูลบัตรเครดิต/ธุรกรรมการเงิน) การเปิด direct query ผ่าน FDW เข้าไปยังฐานข้อมูลนี้เพิ่มความเสี่ยงด้าน security surface และ compliance scope โดยไม่จำเป็น → **ไม่ควรใช้ FDW ตรง ๆ** ควรให้ทีม Payment ส่งเฉพาะข้อมูลสรุปที่ถูก mask/anonymize แล้ว (เช่น ยอดรวมต่อวัน, สถานะธุรกรรมแบบไม่มี PII) ผ่านช่องทางที่ควบคุมได้ดีกว่า เช่น scheduled export, message queue (Kafka), หรือ API เฉพาะที่ผ่านการ audit

**สรุปหลักการ:** ใช้ FDW กับข้อมูลที่ (1) ไม่อ่อนไหวด้าน compliance, (2) ขนาดพอเหมาะสำหรับ query แบบ interactive, และ (3) ทีมเจ้าของยินดีเปิด read-only access ให้ ส่วนข้อมูลที่อ่อนไหวหรือมี regulation ควบคุม ควรผ่านชั้น anonymization/aggregation ก่อนเสมอ ไม่ query ตรงข้ามระบบผ่าน FDW
</details>

---

**บทถัดไป:** [Part 057 — Extensions](./part-057-extensions.md)
