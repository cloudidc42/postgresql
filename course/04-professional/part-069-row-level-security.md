# Part 069: Row Level Security (RLS) และ Multi-tenant Security

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับมืออาชีพ | Part 069

---

## เป้าหมายการเรียนรู้

เมื่อจบบทนี้ ผู้เรียนจะสามารถ:

1. อธิบายได้ว่า Row Level Security (RLS) คืออะไร และแตกต่างจากการควบคุมสิทธิ์แบบ `GRANT`/`REVOKE` ระดับตารางอย่างไร
2. อธิบายได้ว่าทำไม RLS จึงเป็นกลไกสำคัญสำหรับสถาปัตยกรรม Multi-tenant SaaS และเชื่อมโยงกับแนวคิด security barrier view ที่เคยเรียนใน Part 035
3. เปิดใช้งาน RLS บนตารางด้วย `ALTER TABLE ... ENABLE ROW LEVEL SECURITY` และเข้าใจผลกระทบทันทีที่เกิดขึ้น
4. เขียน `CREATE POLICY` พื้นฐานโดยใช้ `USING` และ `WITH CHECK` ได้อย่างถูกต้อง
5. ใช้ `current_setting()` ร่วมกับ `SET LOCAL` เพื่อส่งค่า context (เช่น tenant_id ปัจจุบัน) เข้าไปให้ policy ตรวจสอบ
6. เขียน policy แยกตามคำสั่ง (`FOR SELECT`, `FOR INSERT`, `FOR UPDATE`, `FOR DELETE`, `FOR ALL`) และกำหนด role เป้าหมายด้วย `TO`
7. เข้าใจความแตกต่างระหว่าง PERMISSIVE policy (รวมกันด้วย OR) กับ RESTRICTIVE policy (เพิ่มเงื่อนไขด้วย AND) และเลือกใช้ให้เหมาะสม
8. เข้าใจ `BYPASSRLS` attribute และบทบาทของ superuser/role พิเศษในการข้าม RLS สำหรับงาน admin หรือ migration
9. วิเคราะห์ผลกระทบด้านประสิทธิภาพของ RLS ด้วย `EXPLAIN (ANALYZE, BUFFERS)` และรู้วิธี optimize policy ที่ซับซ้อน
10. ออกแบบและ implement ระบบ multi-tenant SaaS เต็มรูปแบบด้วย RLS พร้อมพิสูจน์ด้วยการทดสอบจริงว่า tenant หนึ่งไม่สามารถเห็นข้อมูลของอีก tenant ได้

---

## เตรียมข้อมูล

บทนี้จะปรับ schema e-commerce เดิมให้เป็นแบบ **multi-tenant** โดยเพิ่มคอลัมน์ `tenant_id` เข้าไปในทุกตารางหลัก เพื่อจำลองสถานการณ์ SaaS platform ที่ร้านค้าหลายร้าน (tenant) ใช้ฐานข้อมูลชุดเดียวกันร่วมกัน (shared database, shared schema) ซึ่งเป็น use case คลาสสิกที่สุดของ RLS

รันสคริปต์ต่อไปนี้ในฐานข้อมูลทดสอบใหม่ (แนะนำให้สร้างฐานข้อมูลแยก เช่น `rls_course`) เพื่อไม่ให้ชนกับ schema ในบทอื่น:

```sql
-- แนะนำให้สร้างฐานข้อมูลใหม่สำหรับบทนี้โดยเฉพาะ
-- CREATE DATABASE rls_course;
-- \c rls_course

DROP TABLE IF EXISTS orders, customers, products, tenants CASCADE;

CREATE TABLE tenants (
    tenant_id   SERIAL PRIMARY KEY,
    tenant_name VARCHAR(100) NOT NULL
);

CREATE TABLE products (
    product_id   SERIAL PRIMARY KEY,
    tenant_id    INTEGER NOT NULL REFERENCES tenants(tenant_id),
    product_name VARCHAR(150) NOT NULL,
    unit_price   NUMERIC(10,2) NOT NULL
);

CREATE TABLE customers (
    customer_id SERIAL PRIMARY KEY,
    tenant_id   INTEGER NOT NULL REFERENCES tenants(tenant_id),
    first_name  VARCHAR(60),
    last_name   VARCHAR(60),
    email       VARCHAR(150)
);

CREATE TABLE orders (
    order_id     SERIAL PRIMARY KEY,
    tenant_id    INTEGER NOT NULL REFERENCES tenants(tenant_id),
    customer_id  INTEGER REFERENCES customers(customer_id),
    order_date   TIMESTAMPTZ DEFAULT now(),
    total_amount NUMERIC(12,2)
);

-- สร้าง index บน tenant_id ไว้ล่วงหน้า (สำคัญมากสำหรับ Step 689)
CREATE INDEX idx_products_tenant  ON products(tenant_id);
CREATE INDEX idx_customers_tenant ON customers(tenant_id);
CREATE INDEX idx_orders_tenant    ON orders(tenant_id);
```

ใส่ข้อมูลจำลอง 3 ร้านค้า (3 tenants) ที่ใช้แพลตฟอร์มเดียวกัน:

```sql
INSERT INTO tenants (tenant_name) VALUES
    ('ร้านกาแฟดารา'),        -- tenant_id = 1
    ('ร้านหนังสือปัญญา'),     -- tenant_id = 2
    ('ร้านเสื้อผ้าแฟชั่นดี');  -- tenant_id = 3

-- สินค้า
INSERT INTO products (tenant_id, product_name, unit_price) VALUES
    (1, 'กาแฟลาเต้ร้อน',        65.00),
    (1, 'กาแฟอเมริกาโน่เย็น',   55.00),
    (1, 'ครัวซองต์เนยสด',       45.00),
    (2, 'หนังสือนิยายไทยยอดนิยม', 250.00),
    (2, 'หนังสือพัฒนาตนเอง',     190.00),
    (2, 'สมุดโน้ตปกหนัง',        80.00),
    (3, 'เสื้อยืดคอกลม',        290.00),
    (3, 'กางเกงยีนส์ทรงตรง',     890.00),
    (3, 'หมวกแก๊ปลายปัก',        250.00);

-- ลูกค้า
INSERT INTO customers (tenant_id, first_name, last_name, email) VALUES
    (1, 'สมชาย', 'ใจดี',      'somchai@example.com'),
    (1, 'สมหญิง', 'รักเรียน',  'somying@example.com'),
    (2, 'วิชัย',  'ปัญญาดี',   'wichai@example.com'),
    (2, 'มานี',   'มีสุข',     'manee@example.com'),
    (3, 'อรทัย',  'แฟชั่นนิสต้า', 'orathai@example.com');

-- คำสั่งซื้อ
INSERT INTO orders (tenant_id, customer_id, total_amount) VALUES
    (1, 1, 165.00),
    (1, 2, 65.00),
    (1, 1, 100.00),
    (2, 3, 440.00),
    (2, 4, 80.00),
    (3, 5, 1180.00),
    (3, 5, 250.00);
```

ตรวจสอบข้อมูลก่อนเริ่ม:

```sql
SELECT t.tenant_name, COUNT(DISTINCT p.product_id) AS n_products,
       COUNT(DISTINCT c.customer_id) AS n_customers,
       COUNT(DISTINCT o.order_id) AS n_orders
FROM tenants t
LEFT JOIN products p  ON p.tenant_id = t.tenant_id
LEFT JOIN customers c ON c.tenant_id = t.tenant_id
LEFT JOIN orders o    ON o.tenant_id = t.tenant_id
GROUP BY t.tenant_id, t.tenant_name
ORDER BY t.tenant_id;
```

```
    tenant_name       | n_products | n_customers | n_orders
-----------------------+------------+-------------+----------
 ร้านกาแฟดารา          |          3 |           2 |        3
 ร้านหนังสือปัญญา       |          3 |           2 |        2
 ร้านเสื้อผ้าแฟชั่นดี    |          3 |           1 |        2
```

ข้อมูลนี้จำลองสถานการณ์ SaaS platform ที่ทั้ง 3 ร้านค้าใช้ตาราง `products`, `customers`, `orders` ชุดเดียวกัน (ไม่มีการแยก schema หรือแยกฐานข้อมูล) — นี่คือจุดที่ RLS จะเข้ามาทำหน้าที่เป็น "กำแพงความปลอดภัย" กั้นระหว่างร้านค้าแต่ละร้าน

---

## Step 681: Row Level Security คืออะไร

**Row Level Security (RLS)** คือกลไกของ PostgreSQL ที่ให้เรากำหนด "กฎ" (policy) ว่า **แถวไหน (row)** ในตารางที่ role หนึ่ง ๆ สามารถ **มองเห็น** หรือ **แก้ไข** ได้บ้าง — โดยกฎเหล่านี้ถูกบังคับใช้ที่ **ระดับฐานข้อมูลเอง** ไม่ใช่ที่ระดับ application

ก่อนหน้านี้เราคุ้นเคยกับการควบคุมสิทธิ์ด้วย `GRANT`/`REVOKE` ซึ่งเป็นการควบคุมที่ **ระดับตาราง** ทั้งตาราง เช่น "role A อ่านตาราง orders ได้" หรือ "role B เขียนตาราง products ไม่ได้" แต่ `GRANT`/`REVOKE` ไม่สามารถตอบคำถามที่ละเอียดกว่านั้นได้ เช่น:

- "role A อ่านตาราง orders ได้ **เฉพาะแถวของ tenant ตัวเอง**"
- "พนักงานขายแก้ไขคำสั่งซื้อได้ **เฉพาะที่ตัวเองเป็นคนสร้าง**"
- "ลูกค้าดูข้อมูลตัวเองได้ **เฉพาะ record ที่ผูกกับ customer_id ของตัวเอง**"

RLS เติมเต็มช่องว่างตรงนี้ โดยทำงานคล้ายกับการเพิ่ม `WHERE` clause แบบ "บังคับ" (invisible, mandatory) เข้าไปในทุกคำสั่ง SQL ที่ query ตารางนั้นโดยอัตโนมัติ ไม่ว่า application จะเขียนคำสั่งอย่างไรก็ตาม

ลองดูตัวอย่างแนวคิดแบบง่ายที่สุดก่อน — สมมติว่าถ้าไม่มี RLS แอปพลิเคชันต้องเขียนแบบนี้ในทุก query:

```sql
-- แบบเดิม (ไม่มี RLS) — ต้องพึ่งวินัยของโปรแกรมเมอร์ทุกคนทุกจุดในโค้ด
SELECT * FROM orders WHERE tenant_id = 1;   -- ร้านกาแฟดารา query แบบนี้
SELECT * FROM orders WHERE tenant_id = 2;   -- ร้านหนังสือปัญญา query แบบนี้
```

ปัญหาคือ ถ้ามีจุดใดจุดหนึ่งในโค้ด (endpoint, report, batch job, raw SQL ที่เขียนรีบ ๆ) **ลืม** ใส่ `WHERE tenant_id = ...` แม้เพียงครั้งเดียว ข้อมูลของ tenant อื่นจะรั่วไหลทันที นี่คือความเสี่ยงร้ายแรงของสถาปัตยกรรม multi-tenant ที่ใช้ shared database + shared schema

RLS แก้ปัญหานี้โดยย้าย "กฎการกรอง" ไปไว้ที่ **ฐานข้อมูล** แทนที่จะฝากความหวังไว้กับวินัยของโปรแกรมเมอร์:

```sql
-- แนวคิด: เมื่อเปิด RLS และสร้าง policy แล้ว
SELECT * FROM orders;   -- คำสั่งนี้ "ดูเหมือน" จะดึงทุกแถว
-- แต่ PostgreSQL จะแปลงเป็นภายในเป็นประมาณ:
-- SELECT * FROM orders WHERE tenant_id = current_setting('app.current_tenant_id')::int;
-- โดยอัตโนมัติ ไม่ว่า SQL ที่เขียนมาจะหน้าตาอย่างไร
```

จุดสำคัญที่ต้องเข้าใจ:

1. **RLS ทำงานโปร่งใส (transparent)** — แอปพลิเคชันไม่ต้องรู้ว่ามี RLS อยู่เลย เขียน `SELECT * FROM orders` ธรรมดา แต่ผลลัพธ์ที่ได้จะถูกกรองแล้ว
2. **RLS บังคับใช้กับทุกช่องทาง** — ไม่ว่าจะ query ผ่าน ORM, raw SQL, `psql`, BI tool ใด ๆ ก็ตาม ถ้า connection ใช้ role ที่ไม่มี `BYPASSRLS` policy จะถูกบังคับใช้เสมอ
3. **RLS ไม่ใช่ column masking** — RLS ซ่อน/กรอง **ทั้งแถว** ไม่ใช่การซ่อนบางคอลัมน์ (ถ้าต้องการซ่อนบางคอลัมน์ ให้ใช้ view หรือ column privileges ที่เคยเรียนมาก่อนหน้านี้)
4. **RLS ทำงานที่ระดับ table owner เป็นข้อยกเว้น** — โดย default เจ้าของตาราง (table owner) และ superuser จะ **ไม่ถูก** RLS บังคับ (จะอธิบายรายละเอียดใน Step 683 และ 688)

> **เปรียบเทียบง่าย ๆ**: `GRANT`/`REVOKE` เหมือนการ์ดผ่านประตูที่บอกว่า "คุณเข้าห้องนี้ได้หรือไม่ได้" ส่วน RLS เหมือนแว่นตาพิเศษที่ใส่แล้วทำให้มองเห็นเฉพาะของบางชิ้นในห้องนั้น แม้จะเข้าห้องได้แล้วก็ตาม

---

## Step 682: ทำไม RLS สำคัญสำหรับ Multi-tenant SaaS

ใน Part 035 เราเคยเรียนเรื่อง **security barrier view** ซึ่งเป็นเทคนิคสร้าง view ที่มี `WHERE` clause กรองข้อมูลไว้ และใช้ option `security_barrier` เพื่อป้องกันไม่ให้ query planner "ดัน" (push down) เงื่อนไขจากภายนอกเข้าไปก่อนที่ view จะกรองข้อมูลเสร็จ (ซึ่งอาจทำให้ฟังก์ชันที่มี side-effect หรือ error-based information leak รั่วไหลข้อมูลได้)

ทบทวนแนวคิดเดิมแบบสั้น ๆ:

```sql
-- แนวทางแบบ view (ที่เคยเรียนใน Part 035)
CREATE VIEW orders_tenant_1 WITH (security_barrier = true) AS
SELECT * FROM orders WHERE tenant_id = 1;

GRANT SELECT ON orders_tenant_1 TO tenant1_app_role;
```

วิธีนี้ **ใช้งานได้** แต่มีข้อจำกัดร้ายแรงเมื่อนำไปใช้กับ SaaS ที่มี tenant นับสิบ นับร้อย หรือนับพันราย:

| ปัญหาของแนวทาง View ต่อ tenant | ผลกระทบ |
|---|---|
| ต้องสร้าง view ใหม่ทุกครั้งที่มี tenant ใหม่ | ไม่ scale — 1,000 tenants = 1,000 views (คูณด้วยจำนวนตาราง) |
| view ผูกกับ tenant_id แบบ hardcode | เปลี่ยน tenant ไม่ได้แบบ dynamic ต้องสร้าง view ใหม่ตลอด |
| INSERT/UPDATE ผ่าน view ซับซ้อน | ต้องจัดการ `WITH CHECK OPTION` เองในทุก view |
| Schema เดียวมี view เพิ่มขึ้นเรื่อย ๆ | metadata bloat, บำรุงรักษายาก |

**RLS แก้ปัญหานี้ทั้งหมด** ด้วยแนวคิดที่ต่างออกไป: แทนที่จะสร้าง view แยกตาม tenant เราสร้าง **policy เดียว** ที่ตรวจสอบค่า `tenant_id` ของแถวเทียบกับค่า "tenant ปัจจุบันของ session" (ซึ่งถูกตั้งค่าแบบ dynamic ตอน connection/transaction เริ่มต้น) — ไม่ว่าจะมี tenant กี่รายก็ใช้ policy เดียวกันได้ทั้งหมด:

```sql
-- แนวคิด RLS (รายละเอียดเต็มจะอยู่ใน Step 683-684)
CREATE POLICY tenant_isolation ON orders
    USING (tenant_id = current_setting('app.current_tenant_id')::int);
```

policy เดียวนี้ครอบคลุมทุก tenant เพราะค่า `current_setting('app.current_tenant_id')` จะถูกกำหนดแบบ dynamic โดย application ทุกครั้งที่เปิด connection หรือเริ่ม transaction

**เหตุผลที่ RLS เหมาะกับสถาปัตยกรรม SaaS โดยเฉพาะ:**

1. **Defense in depth (การป้องกันหลายชั้น)** — แม้ application layer จะมีบั๊ก ลืมกรอง `tenant_id`, หรือถูกโจมตีด้วย SQL injection ที่หลุดผ่าน parameterized query มาได้ ฐานข้อมูลก็ยังคงกรองข้อมูลให้ถูกต้องอยู่ดี เพราะกฎอยู่ที่ database ไม่ใช่ที่โค้ด
2. **ลดความซับซ้อนของโค้ด application** — ไม่ต้องเขียน `WHERE tenant_id = ?` ซ้ำ ๆ ในทุก query ทุก endpoint ทุก report
3. **รองรับ audit และ compliance** — กฎการเข้าถึงข้อมูลถูกบันทึกเป็นส่วนหนึ่งของ database schema (`\d+`, `pg_policies`) ตรวจสอบและ review ได้ง่ายกว่าไล่หาทั่วโค้ด application
4. **ทำงานร่วมกับทุกช่องทางการเข้าถึง** — BI tools, ad-hoc query จาก DBA, batch jobs, replication consumers ล้วนถูกกรองเหมือนกันหมด (ยกเว้น role ที่มี `BYPASSRLS`)
5. **Scale ได้กับจำนวน tenant ที่เพิ่มขึ้น** — ไม่ต้องสร้าง objectใหม่เมื่อมี tenant ใหม่ เพียงแค่ insert แถวใหม่ที่มี `tenant_id` ถูกต้อง

> **ข้อควรระวัง**: RLS ไม่ใช่ "กระสุนวิเศษ" ที่แทนที่การออกแบบระบบความปลอดภัยที่ดีทั้งหมด มันเป็น **เลเยอร์เสริม** ที่ควรใช้ร่วมกับการตรวจสอบสิทธิ์ที่ application layer, การเข้ารหัสข้อมูล, และหลักการ least privilege ไม่ใช่ใช้แทนกันได้ทั้งหมด

---

## Step 683: ALTER TABLE ... ENABLE ROW LEVEL SECURITY

ก่อนที่ policy ใด ๆ จะมีผล เราต้อง **เปิดใช้งาน RLS** บนตารางก่อนด้วยคำสั่ง `ALTER TABLE ... ENABLE ROW LEVEL SECURITY`

```sql
ALTER TABLE orders    ENABLE ROW LEVEL SECURITY;
ALTER TABLE products  ENABLE ROW LEVEL SECURITY;
ALTER TABLE customers ENABLE ROW LEVEL SECURITY;
```

**สิ่งที่เกิดขึ้นทันทีหลังรันคำสั่งนี้** เป็นเรื่องที่ผู้เรียนต้องเข้าใจให้แม่นยำมาก เพราะเป็นจุดที่หลายคนพลาดตอนใช้งานจริง:

> เมื่อเปิด RLS บนตารางแล้ว **แต่ยังไม่มี policy ใด ๆ** ถูกสร้างขึ้น ตารางนั้นจะกลายเป็น **"ปิดสนิท" สำหรับทุก role ที่ไม่ใช่เจ้าของตาราง** — คือ SELECT/INSERT/UPDATE/DELETE จะคืนค่าศูนย์แถวเสมอ (ไม่ error แต่ได้ 0 rows) ซึ่งเป็นพฤติกรรมแบบ **default deny** (ปลอดภัยไว้ก่อน)

มาพิสูจน์ด้วยตัวอย่างจริง เราจะสร้าง role สำหรับ application ก่อน:

```sql
-- สร้าง role สำหรับ application (ยังไม่มี policy ใด ๆ)
CREATE ROLE app_user LOGIN PASSWORD 'app_user_pass_2024';
GRANT USAGE ON SCHEMA public TO app_user;
GRANT SELECT, INSERT, UPDATE, DELETE ON tenants, products, customers, orders TO app_user;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO app_user;
```

ทดสอบด้วย role `app_user` (เปิด session ใหม่ `psql -U app_user -d rls_course`):

```sql
-- เชื่อมต่อด้วย role app_user
SET SESSION AUTHORIZATION app_user;

SELECT * FROM orders;
```

```
 order_id | tenant_id | customer_id | order_date | total_amount
----------+-----------+-------------+------------+--------------
(0 rows)
```

จะเห็นว่าได้ **0 แถว** ทั้งที่ตาราง `orders` มีข้อมูลอยู่จริง 7 แถว — นี่คือผลของการเปิด RLS โดยไม่มี policy รองรับ ตารางถูก "ล็อก" ไว้ทั้งหมดสำหรับ role ที่ไม่ใช่เจ้าของ

```sql
-- กลับมาเป็น superuser/owner
RESET SESSION AUTHORIZATION;

SELECT * FROM orders;   -- owner ยังเห็นข้อมูลปกติ เพราะ owner ไม่ถูก RLS บังคับโดย default
```

```
 order_id | tenant_id | customer_id |          order_date          | total_amount
----------+-----------+-------------+-------------------------------+--------------
        1 |         1 |           1 | 2026-09-25 10:00:00+00        |       165.00
        2 |         1 |           2 | 2026-09-25 10:00:00+00        |        65.00
        ...
(7 rows)
```

**คำสั่งที่เกี่ยวข้องอีก 3 คำสั่งที่ควรรู้จักคู่กัน:**

```sql
-- ปิดการใช้งาน RLS (policy ยังอยู่ แต่ไม่ถูกบังคับใช้)
ALTER TABLE orders DISABLE ROW LEVEL SECURITY;

-- บังคับให้ RLS มีผลแม้กระทั่งกับ table owner (สำคัญมากในทางปฏิบัติ)
ALTER TABLE orders FORCE ROW LEVEL SECURITY;

-- ยกเลิกการบังคับกับ owner (กลับสู่ default)
ALTER TABLE orders NO FORCE ROW LEVEL SECURITY;
```

เรื่อง `FORCE ROW LEVEL SECURITY` เป็นจุดที่มักถูกมองข้าม: โดย default แม้ตารางจะเปิด RLS ไว้ **เจ้าของตาราง (table owner)** จะยังคงมองเห็นข้อมูลได้ทั้งหมดเสมอ (เหมือนที่เราเห็นในตัวอย่างข้างบน) เพราะ PostgreSQL ถือว่า owner คือผู้ที่ "เชื่อถือได้เต็มที่" อยู่แล้ว หากต้องการให้ policy มีผลแม้กระทั่งกับ owner เอง (เช่น เพื่อทดสอบ policy โดยไม่ต้องสลับ role) ต้องเพิ่ม `FORCE ROW LEVEL SECURITY` เข้าไปด้วย

```sql
-- ทดลอง: บังคับ RLS กับ owner ด้วย
ALTER TABLE orders FORCE ROW LEVEL SECURITY;

SELECT * FROM orders;   -- รันในฐานะ owner
```

```
 order_id | tenant_id | customer_id | order_date | total_amount
----------+-----------+-------------+------------+--------------
(0 rows)
```

ตอนนี้แม้แต่ owner ก็เห็น 0 แถวเช่นกัน (เพราะยังไม่มี policy) เราจะยกเลิกไว้ก่อนเพื่อความสะดวกในการทดสอบ policy ต่อไปในบทนี้ (จะนำกลับมาใช้ตอน Step 690):

```sql
ALTER TABLE orders NO FORCE ROW LEVEL SECURITY;
```

**ตรวจสอบสถานะ RLS ของตาราง** ผ่าน catalog `pg_tables` หรือคำสั่ง `\d`:

```sql
SELECT tablename, rowsecurity, forcerowsecurity
FROM pg_tables
WHERE schemaname = 'public'
ORDER BY tablename;
```

```
 tablename | rowsecurity | forcerowsecurity
-----------+-------------+-------------------
 customers | t           | f
 orders    | t           | f
 products  | t           | f
 tenants   | f           | f
```

สังเกตว่า `tenants` ยังไม่ได้เปิด RLS (เพราะตาราง tenant master นี้มักจะให้ทุก role อ่านได้ หรือใช้ policy คนละแบบ — จะพูดถึงใน Step 690)

> **ข้อควรระวังสำคัญที่สุดของ Step นี้**: การเปิด RLS โดยไม่สร้าง policy ทันที = ตารางถูกล็อกสำหรับทุก role ที่ไม่ใช่ owner หากทำในระบบ production โดยไม่วางแผนดี แอปพลิเคชันจะพังทันที (query คืนค่าว่างเปล่าทุกจุด) ควรเปิด RLS พร้อมกับสร้าง policy ใน transaction เดียวกันเสมอ

---

## Step 684: CREATE POLICY พื้นฐาน — USING และ WITH CHECK

`CREATE POLICY` คือคำสั่งที่กำหนดกฎการกรองแถว รูปแบบพื้นฐานคือ:

```sql
CREATE POLICY policy_name ON table_name
    [ AS { PERMISSIVE | RESTRICTIVE } ]
    [ FOR { ALL | SELECT | INSERT | UPDATE | DELETE } ]
    [ TO role_name [, ...] ]
    [ USING (using_expression) ]
    [ WITH CHECK (check_expression) ];
```

สองส่วนที่สำคัญที่สุดคือ `USING` และ `WITH CHECK` ซึ่งมีความหมายต่างกันโดยพื้นฐาน:

| Clause | ใช้กับคำสั่ง | ความหมาย |
|---|---|---|
| `USING` | SELECT, UPDATE, DELETE | ตรวจสอบว่า **แถวที่มีอยู่แล้ว** (existing row) ผ่านเงื่อนไขหรือไม่ — ถ้าไม่ผ่าน แถวนั้นจะถูก "ซ่อน" จากผลลัพธ์ หรือไม่สามารถ UPDATE/DELETE ได้ |
| `WITH CHECK` | INSERT, UPDATE | ตรวจสอบว่า **แถวใหม่ที่กำลังจะถูกเขียน** (new row ที่จะ insert หรือ ผลลัพธ์หลัง update) ผ่านเงื่อนไขหรือไม่ — ถ้าไม่ผ่าน คำสั่งจะ **error** ทันที |

ให้จำง่าย ๆ ว่า: **`USING` = กรองสิ่งที่จะอ่าน/ลบ**, **`WITH CHECK` = ตรวจสอบสิ่งที่จะเขียนเข้าไปใหม่**

มาสร้าง policy แรกสำหรับตาราง `orders`:

```sql
CREATE POLICY tenant_isolation_orders ON orders
    USING (tenant_id = current_setting('app.current_tenant_id', true)::int)
    WITH CHECK (tenant_id = current_setting('app.current_tenant_id', true)::int);
```

อธิบายทีละส่วน:

- `USING (tenant_id = current_setting('app.current_tenant_id', true)::int)` — เมื่อมีการ SELECT, UPDATE, หรือ DELETE แถวใน `orders` จะแสดงผล/แก้ไข/ลบได้ **เฉพาะแถวที่ `tenant_id` ตรงกับค่าที่ตั้งไว้ใน session variable `app.current_tenant_id`**
- `WITH CHECK (...)` — เมื่อมีการ INSERT หรือ UPDATE แถวใหม่ที่จะเขียนต้องมี `tenant_id` ตรงกับ session variable นี้เช่นกัน มิฉะนั้นจะ error (ป้องกันไม่ให้ tenant A แอบ insert ข้อมูลปลอมเข้าไปเป็นของ tenant B)
- `current_setting('app.current_tenant_id', true)` — พารามิเตอร์ตัวที่สอง `true` หมายถึง "missing_ok" คือถ้ายังไม่มีการตั้งค่านี้เลยให้คืนค่า `NULL` แทนที่จะ error (จะอธิบายรายละเอียดใน Step 685)

สร้าง policy สำหรับตารางอื่น ๆ ด้วยรูปแบบเดียวกัน:

```sql
CREATE POLICY tenant_isolation_products ON products
    USING (tenant_id = current_setting('app.current_tenant_id', true)::int)
    WITH CHECK (tenant_id = current_setting('app.current_tenant_id', true)::int);

CREATE POLICY tenant_isolation_customers ON customers
    USING (tenant_id = current_setting('app.current_tenant_id', true)::int)
    WITH CHECK (tenant_id = current_setting('app.current_tenant_id', true)::int);
```

ทดสอบผลลัพธ์ทันที ด้วย role `app_user`:

```sql
SET SESSION AUTHORIZATION app_user;

-- ยังไม่ตั้งค่า app.current_tenant_id เลย
SELECT * FROM orders;
```

```
 order_id | tenant_id | customer_id | order_date | total_amount
----------+-----------+-------------+------------+--------------
(0 rows)
```

เพราะ `current_setting('app.current_tenant_id', true)` คืนค่า `NULL` และ `tenant_id = NULL` จะประเมินผลเป็น `NULL` (ไม่ใช่ `true`) เสมอ — ตาม 3-valued logic ของ SQL แถวใด ๆ ก็ตามจะไม่ผ่านเงื่อนไขนี้ จึงยังคงเห็น 0 แถว ทีนี้ลองตั้งค่า session variable แล้วทดสอบใหม่:

```sql
SET app.current_tenant_id = '1';

SELECT * FROM orders;
```

```
 order_id | tenant_id | customer_id |          order_date          | total_amount
----------+-----------+-------------+-------------------------------+--------------
        1 |         1 |           1 | 2026-09-25 10:00:00+00        |       165.00
        2 |         1 |           2 | 2026-09-25 10:00:00+00        |        65.00
        3 |         1 |           1 | 2026-09-25 10:00:00+00        |       100.00
(3 rows)
```

ตอนนี้เราเห็นเฉพาะ 3 แถวของ **ร้านกาแฟดารา (tenant_id = 1)** เท่านั้น! ลองเปลี่ยนเป็น tenant อื่น:

```sql
SET app.current_tenant_id = '2';

SELECT * FROM orders;
```

```
 order_id | tenant_id | customer_id |          order_date          | total_amount
----------+-----------+-------------+-------------------------------+--------------
        4 |         2 |           3 | 2026-09-25 10:00:00+00        |       440.00
        5 |         2 |           4 | 2026-09-25 10:00:00+00        |        80.00
(2 rows)
```

ทดสอบ `WITH CHECK` โดยพยายามแอบ insert ข้อมูลที่ `tenant_id` ไม่ตรงกับ session ปัจจุบัน:

```sql
SET app.current_tenant_id = '1';   -- session นี้เป็นตัวแทนของ tenant 1

-- พยายามแอบสร้างคำสั่งซื้อให้ tenant 2 ทั้งที่ session เป็นของ tenant 1
INSERT INTO orders (tenant_id, customer_id, total_amount)
VALUES (2, 3, 999.00);
```

```
ERROR:  new row violates row-level security policy for table "orders"
```

`WITH CHECK` ปฏิเสธคำสั่งนี้ทันที เพราะแถวใหม่ที่พยายาม insert มี `tenant_id = 2` ซึ่งไม่ตรงกับ `app.current_tenant_id = '1'` — นี่คือการป้องกัน **cross-tenant data injection** ที่สำคัญมากในระบบ SaaS

ทดสอบ insert ที่ถูกต้อง:

```sql
INSERT INTO orders (tenant_id, customer_id, total_amount)
VALUES (1, 1, 199.00)
RETURNING *;
```

```
 order_id | tenant_id | customer_id |          order_date          | total_amount
----------+-----------+-------------+-------------------------------+--------------
        8 |         1 |           1 | 2026-09-25 10:05:00+00        |       199.00
(1 row)
```

สำเร็จ เพราะ `tenant_id = 1` ตรงกับ session variable

> **ข้อสังเกตสำคัญ**: ถ้า `CREATE POLICY` ไม่ระบุ `FOR` clause จะถือว่าเป็น `FOR ALL` โดย default (ครอบคลุมทั้ง SELECT, INSERT, UPDATE, DELETE) และถ้าไม่ระบุ `WITH CHECK` แต่มี `USING` สำหรับ policy ที่ใช้กับ INSERT/UPDATE ด้วย PostgreSQL จะใช้ `USING` expression เดียวกันเป็น `WITH CHECK` โดยอัตโนมัติ

---

## Step 685: current_setting() และ SET LOCAL

หัวใจของการ implement RLS แบบ multi-tenant คือการส่งค่า "ตัวตนปัจจุบัน" (เช่น tenant_id) เข้าไปใน session ให้ policy นำไปใช้ตรวจสอบ กลไกที่ใช้กันมากที่สุดคือ **custom configuration parameter** ผ่าน `SET`/`SET LOCAL` คู่กับฟังก์ชัน `current_setting()`

### รูปแบบของ current_setting()

```sql
current_setting(setting_name)
current_setting(setting_name, missing_ok)
```

- `current_setting('app.current_tenant_id')` — ถ้ายังไม่เคยตั้งค่าตัวแปรนี้เลย จะ **error**: `unrecognized configuration parameter`
- `current_setting('app.current_tenant_id', true)` — ถ้ายังไม่เคยตั้งค่า จะคืนค่า `NULL` แทน (ไม่ error) — พารามิเตอร์ `missing_ok = true` นี้สำคัญมากสำหรับ policy เพราะเราไม่ต้องการให้ query ทั้งหมด error เพียงเพราะ session ยังไม่ได้ set ค่า

ทดสอบความแตกต่าง:

```sql
-- ยังไม่เคยตั้งค่า app.some_random_key มาก่อน
SELECT current_setting('app.some_random_key');
```

```
ERROR:  unrecognized configuration parameter "app.some_random_key"
```

```sql
SELECT current_setting('app.some_random_key', true);
```

```
 current_setting
------------------
 (null)
(1 row)
```

> **ข้อกำหนดสำคัญ**: custom parameter ที่ไม่ใช่ของ PostgreSQL core (เช่น `app.current_tenant_id`) **ต้องมีจุด (`.`) คั่นเสมอ** — รูปแบบ `<prefix>.<name>` เป็นข้อบังคับของ PostgreSQL สำหรับ custom GUC (Grand Unified Configuration) parameters การตั้งชื่อโดยไม่มีจุดจะ error

### SET vs SET LOCAL

ความแตกต่างระหว่าง `SET` และ `SET LOCAL` เป็นเรื่องสำคัญมากในการ implement RLS อย่างปลอดภัย:

| คำสั่ง | ขอบเขตของค่าที่ตั้ง | ใช้เมื่อไร |
|---|---|---|
| `SET app.current_tenant_id = '1'` | มีผลตลอด **session** จนกว่าจะเปลี่ยนหรือปิด connection | เหมาะกับการทดสอบใน `psql` แบบ manual |
| `SET LOCAL app.current_tenant_id = '1'` | มีผลเฉพาะภายใน **transaction ปัจจุบัน** เท่านั้น (rollback ไปเมื่อ COMMIT/ROLLBACK) | **แนะนำสำหรับ production** เพราะป้องกัน connection pool leak ค่าข้ามระหว่าง request |

**เหตุผลที่ `SET LOCAL` ปลอดภัยกว่าสำหรับ production**: ระบบ SaaS ส่วนใหญ่ใช้ **connection pooling** (เช่น PgBouncer, หรือ pool ของ ORM) ซึ่ง connection เดียวกันอาจถูกใช้ซ้ำโดย request ของ tenant ต่างกันในเวลาไล่เลี่ยกัน ถ้าใช้ `SET` ธรรมดาแล้วลืม reset ค่า tenant_id ของ request ก่อนหน้าอาจ "รั่วไหล" ไปยัง request ถัดไปที่ใช้ connection เดียวกัน (แม้จะเป็นคนละ transaction) แต่ `SET LOCAL` จะถูกล้างค่าอัตโนมัติทันทีที่ transaction จบ (COMMIT หรือ ROLLBACK) ทำให้ปลอดภัยกว่ามาก

ตัวอย่างรูปแบบที่แนะนำในการเขียนโค้ด application (pseudo-code แสดงแนวคิด):

```sql
BEGIN;
SET LOCAL app.current_tenant_id = '2';

SELECT * FROM orders;   -- เห็นเฉพาะของ tenant 2

INSERT INTO orders (tenant_id, customer_id, total_amount)
VALUES (2, 4, 120.00);

COMMIT;   -- transaction จบ -> app.current_tenant_id ถูกล้างอัตโนมัติ

-- นอก transaction ใหม่ ถ้าไม่ SET LOCAL ใหม่ ค่าจะกลับเป็นค่า default (ไม่มีค่า)
SELECT current_setting('app.current_tenant_id', true);
```

```
 current_setting
------------------
 (null)
(1 row)
```

เทียบกับการใช้ `SET` ธรรมดาที่ค่ายัง "ค้าง" อยู่ข้าม transaction:

```sql
BEGIN;
SET app.current_tenant_id = '3';   -- ไม่ใช่ SET LOCAL
COMMIT;

SELECT current_setting('app.current_tenant_id', true);   -- ยังคงเห็นค่า '3' อยู่!
```

```
 current_setting
------------------
 3
(1 row)
```

นี่แสดงให้เห็นว่าค่าที่ตั้งด้วย `SET` ธรรมดาจะ "ค้าง" อยู่ในระดับ session ต่อไปแม้ transaction จะจบแล้ว ซึ่งเป็นความเสี่ยงถ้า connection ถูกนำกลับไปใช้ใน connection pool โดยไม่ได้ reset — ด้วยเหตุนี้แนวทางที่ถูกต้องสำหรับ production คือ:

```sql
-- แนวทางที่แนะนำ: ทุก request/transaction เริ่มต้นด้วย SET LOCAL เสมอ
BEGIN;
SET LOCAL app.current_tenant_id = '1';
-- ... ทำงานต่าง ๆ ...
COMMIT;
```

### สร้างฟังก์ชัน helper เพื่อความสะดวก

ในทางปฏิบัติมักห่อ `current_setting()` ไว้ในฟังก์ชันเพื่อให้ policy อ่านง่ายขึ้นและมี default value ที่ชัดเจน:

```sql
CREATE OR REPLACE FUNCTION current_tenant_id() RETURNS INTEGER AS $$
    SELECT NULLIF(current_setting('app.current_tenant_id', true), '')::int;
$$ LANGUAGE sql STABLE;
```

ฟังก์ชันนี้ใช้ `NULLIF(..., '')` เพื่อป้องกันกรณีที่ค่าถูกตั้งเป็น empty string (`''`) ซึ่งจะ cast เป็น `int` ไม่ได้ — ถ้าเป็น empty string จะแปลงเป็น `NULL` แทน ทำให้ policy ปลอดภัยขึ้น

ทดสอบ:

```sql
SET app.current_tenant_id = '';
SELECT current_tenant_id();
```

```
 current_tenant_id
--------------------
 (null)
(1 row)
```

```sql
SET app.current_tenant_id = '2';
SELECT current_tenant_id();
```

```
 current_tenant_id
--------------------
                  2
(1 row)
```

เราจะปรับ policy ให้ใช้ฟังก์ชันนี้แทนใน Step ถัดไปเพื่อความสะอาดของโค้ด

> **หมายเหตุเรื่องความปลอดภัย**: ฟังก์ชัน `current_tenant_id()` ในตัวอย่างนี้เป็นเพียงจุดเริ่มต้น ในระบบจริงมักจะออกแบบให้ค่านี้มาจากการตรวจสอบ JWT/session token ที่ application layer เป็นผู้ set ค่าให้ **หลังจาก** ยืนยันตัวตนผู้ใช้แล้วเท่านั้น ไม่ควรให้ผู้ใช้ปลายทาง (end user) ส่งค่า tenant_id มาเองโดยตรงผ่าน API เพราะจะเปิดช่องให้ปลอมแปลง (spoof) ค่าได้

---

## Step 686: Policy แยกตามคำสั่ง (FOR) และ role (TO)

`CREATE POLICY` สามารถระบุให้มีผลกับ **คำสั่งเฉพาะ** (`FOR SELECT`, `FOR INSERT`, `FOR UPDATE`, `FOR DELETE`, หรือ `FOR ALL`) และ **role เฉพาะ** (`TO role_name`) ได้ ทำให้ออกแบบกฎที่ละเอียดขึ้นได้มาก

ลองออกแบบสถานการณ์ที่ซับซ้อนขึ้น: สมมติว่าแพลตฟอร์มมี 2 บทบาท (role) หลักสำหรับผู้ใช้ในแต่ละร้านค้า:

- **`shop_staff`** — พนักงานร้านค้า อ่าน/เขียนข้อมูลได้ตามปกติ แต่ห้ามลบคำสั่งซื้อ (order) เพื่อความปลอดภัยของประวัติการขาย
- **`shop_owner`** — เจ้าของร้าน ทำได้ทุกอย่างรวมถึงลบคำสั่งซื้อ

```sql
-- ลบ policy เดิมที่สร้างแบบ FOR ALL ออกก่อน เพื่อสร้างใหม่แบบละเอียดขึ้น
DROP POLICY tenant_isolation_orders ON orders;

-- สร้าง role สำหรับพนักงานและเจ้าของร้าน
CREATE ROLE shop_staff LOGIN PASSWORD 'staff_pass_2024';
CREATE ROLE shop_owner LOGIN PASSWORD 'owner_pass_2024';

GRANT USAGE ON SCHEMA public TO shop_staff, shop_owner;
GRANT SELECT, INSERT, UPDATE, DELETE ON orders, products, customers TO shop_staff, shop_owner;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO shop_staff, shop_owner;
```

สร้าง policy แยกตามคำสั่งและ role:

```sql
-- SELECT: ทั้งสอง role เห็นเฉพาะ tenant ตัวเอง
CREATE POLICY orders_select_policy ON orders
    FOR SELECT
    TO shop_staff, shop_owner
    USING (tenant_id = current_tenant_id());

-- INSERT: ทั้งสอง role สร้างคำสั่งซื้อได้ ต้องเป็น tenant ตัวเองเท่านั้น
CREATE POLICY orders_insert_policy ON orders
    FOR INSERT
    TO shop_staff, shop_owner
    WITH CHECK (tenant_id = current_tenant_id());

-- UPDATE: ทั้งสอง role แก้ไขได้ เฉพาะแถวของ tenant ตัวเอง และห้ามเปลี่ยน tenant_id ออกนอก tenant ตัวเอง
CREATE POLICY orders_update_policy ON orders
    FOR UPDATE
    TO shop_staff, shop_owner
    USING (tenant_id = current_tenant_id())
    WITH CHECK (tenant_id = current_tenant_id());

-- DELETE: เฉพาะ shop_owner เท่านั้นที่ลบได้ (shop_staff จะไม่มี policy DELETE เลย = ลบไม่ได้)
CREATE POLICY orders_delete_policy ON orders
    FOR DELETE
    TO shop_owner
    USING (tenant_id = current_tenant_id());
```

ทดสอบด้วย `shop_staff`:

```sql
SET SESSION AUTHORIZATION shop_staff;
SET app.current_tenant_id = '1';

SELECT COUNT(*) FROM orders;   -- SELECT ได้ตามปกติ
```

```
 count
-------
     4
(1 row)
```

```sql
DELETE FROM orders WHERE order_id = 1;
```

```
DELETE 0
```

สังเกตว่า `DELETE 0` — ไม่ error แต่ **ไม่มีแถวไหนถูกลบเลย** เพราะ `shop_staff` ไม่มี policy สำหรับ `FOR DELETE` เลย (ตาราง orders ถูกเปิด RLS แต่ไม่มี policy DELETE ที่ครอบคลุม role นี้ = default deny สำหรับคำสั่งนั้น) พฤติกรรมนี้ต่างจาก error เพราะ RLS มองว่า "ไม่มีแถวไหนผ่านเงื่อนไข" ไม่ใช่ "คำสั่งนี้ผิดกฎ"

ทดสอบด้วย `shop_owner`:

```sql
RESET SESSION AUTHORIZATION;
SET SESSION AUTHORIZATION shop_owner;
SET app.current_tenant_id = '1';

DELETE FROM orders WHERE order_id = 1
RETURNING *;
```

```
 order_id | tenant_id | customer_id |          order_date          | total_amount
----------+-----------+-------------+-------------------------------+--------------
        1 |         1 |           1 | 2026-09-25 10:00:00+00        |       165.00
(1 row)

DELETE 1
```

สำเร็จ เพราะ `shop_owner` มี policy `orders_delete_policy` รองรับ

**ตรวจสอบ policy ทั้งหมดที่มีในระบบ** ผ่าน catalog view `pg_policies`:

```sql
RESET SESSION AUTHORIZATION;

SELECT policyname, tablename, cmd, roles, qual, with_check
FROM pg_policies
WHERE tablename = 'orders'
ORDER BY policyname;
```

```
      policyname       | tablename |  cmd   |         roles          |              qual               |            with_check
------------------------+-----------+--------+-------------------------+----------------------------------+------------------------------------
 orders_delete_policy   | orders    | DELETE | {shop_owner}            | (tenant_id = current_tenant_id())| (null)
 orders_insert_policy   | orders    | INSERT | {shop_staff,shop_owner} | (null)                           | (tenant_id = current_tenant_id())
 orders_select_policy   | orders    | SELECT | {shop_staff,shop_owner} | (tenant_id = current_tenant_id())| (null)
 orders_update_policy   | orders    | UPDATE | {shop_staff,shop_owner} | (tenant_id = current_tenant_id())| (tenant_id = current_tenant_id())
(4 rows)
```

คอลัมน์ `cmd` แสดงคำสั่งที่ policy นั้นครอบคลุม, `roles` แสดง role ที่ policy มีผลด้วย, `qual` คือ `USING` expression, `with_check` คือ `WITH CHECK` expression

> **หมายเหตุเรื่อง TO ที่ไม่ระบุ**: ถ้า `CREATE POLICY` ไม่มี `TO` clause เลย จะถือว่า policy นั้นมีผลกับ **`PUBLIC`** (ทุก role) โดย default ซึ่งบางครั้งอาจไม่ใช่สิ่งที่ต้องการ — ควรระบุ `TO` ให้ชัดเจนเสมอในระบบที่มีหลาย role เพื่อป้องกันความสับสน

---

## Step 687: Multiple Policies — PERMISSIVE เทียบกับ RESTRICTIVE

เมื่อตารางเดียวมี **หลาย policy** ที่ครอบคลุมคำสั่งและ role เดียวกัน PostgreSQL จะรวมผลลัพธ์ของ policy เหล่านั้นตาม attribute `PERMISSIVE` (ค่า default) หรือ `RESTRICTIVE`

### PERMISSIVE (ค่า default) — รวมกันด้วย OR

Policy แบบ PERMISSIVE (ไม่ระบุ = default เป็นแบบนี้) จะถูกนำมา **OR** รวมกัน หมายความว่า **แถวใดแถวหนึ่งผ่านแค่ policy เดียวก็เพียงพอ** ที่จะทำให้แถวนั้นมองเห็นได้/แก้ไขได้

### RESTRICTIVE — เพิ่มเงื่อนไขด้วย AND

Policy แบบ RESTRICTIVE จะถูกนำมา **AND** กับผลลัพธ์ของ PERMISSIVE policies ทั้งหมด หมายความว่า **ทุก RESTRICTIVE policy ต้องผ่านทั้งหมด** ถึงจะเห็น/แก้ไขแถวนั้นได้ แม้ PERMISSIVE policy จะอนุญาตแล้วก็ตาม

สูตรคร่าว ๆ คือ:

```
final_result = (permissive_1 OR permissive_2 OR ...) AND (restrictive_1 AND restrictive_2 AND ...)
```

มาจำลองสถานการณ์ที่ใช้ RESTRICTIVE policy จริง: สมมติว่าแพลตฟอร์มต้องการเพิ่มฟีเจอร์ **"ระงับบัญชีร้านค้า" (tenant suspension)** — เมื่อร้านค้าถูกระงับ (เช่น ค้างชำระค่าบริการ) จะต้อง **ไม่สามารถเข้าถึงข้อมูลใด ๆ ได้เลย แม้จะเป็น tenant ของตัวเองก็ตาม** โดยไม่ต้องแก้ policy เดิมที่มีอยู่แล้ว

```sql
-- เพิ่มคอลัมน์สถานะการระงับใน tenants
ALTER TABLE tenants ADD COLUMN is_suspended BOOLEAN NOT NULL DEFAULT false;

-- เปิด RLS บนตาราง tenants ด้วย (ให้ทุก role อ่านได้ แต่แก้ไขไม่ได้ ยกเว้น admin)
ALTER TABLE tenants ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenants_select_all ON tenants
    FOR SELECT
    TO shop_staff, shop_owner
    USING (true);   -- อ่านตาราง tenant master ได้ทั้งหมด (ไม่มีข้อมูลอ่อนไหว)
```

สร้าง **RESTRICTIVE policy** ที่ตรวจสอบว่า tenant ปัจจุบันถูกระงับหรือไม่:

```sql
CREATE POLICY orders_not_suspended ON orders
    AS RESTRICTIVE
    FOR ALL
    TO shop_staff, shop_owner
    USING (
        NOT EXISTS (
            SELECT 1 FROM tenants t
            WHERE t.tenant_id = orders.tenant_id
              AND t.is_suspended = true
        )
    );
```

ทดสอบสถานการณ์ปกติก่อน (ยังไม่มีใครถูกระงับ):

```sql
SET SESSION AUTHORIZATION shop_staff;
SET app.current_tenant_id = '1';

SELECT COUNT(*) FROM orders;
```

```
 count
-------
     3
(1 row)
```

เห็นข้อมูลปกติ เพราะ RESTRICTIVE policy `orders_not_suspended` ผ่าน (tenant 1 ยังไม่ถูกระงับ) และ PERMISSIVE policy `orders_select_policy` ก็ผ่านเช่นกัน (tenant_id ตรงกับ session)

ตอนนี้ลองระงับ tenant 1:

```sql
RESET SESSION AUTHORIZATION;

UPDATE tenants SET is_suspended = true WHERE tenant_id = 1;
```

ทดสอบอีกครั้ง:

```sql
SET SESSION AUTHORIZATION shop_staff;
SET app.current_tenant_id = '1';

SELECT COUNT(*) FROM orders;
```

```
 count
-------
     0
(1 row)
```

แม้ว่า PERMISSIVE policy `orders_select_policy` จะยังคงอนุญาตให้เห็น orders ของ tenant 1 (เพราะ `tenant_id = current_tenant_id()` ยังเป็นจริงอยู่) แต่ RESTRICTIVE policy `orders_not_suspended` **ปิดกั้น** ทุกอย่างเพราะ tenant 1 ถูกระงับแล้ว — ผลลัพธ์สุดท้ายคือ 0 แถวเสมอ

ทดสอบว่า tenant อื่นที่ไม่ถูกระงับยังทำงานปกติ:

```sql
RESET SESSION AUTHORIZATION;

SET SESSION AUTHORIZATION shop_staff;
SET app.current_tenant_id = '2';

SELECT COUNT(*) FROM orders;
```

```
 count
-------
     2
(1 row)
```

tenant 2 ยังทำงานได้ปกติ เพราะไม่ถูกระงับ

คืนค่า tenant 1 กลับสู่สถานะปกติเพื่อใช้ในบทถัดไป:

```sql
RESET SESSION AUTHORIZATION;
UPDATE tenants SET is_suspended = false WHERE tenant_id = 1;
```

**ทำไมต้องใช้ RESTRICTIVE แยกต่างหาก แทนที่จะแก้ policy เดิมให้ซับซ้อนขึ้น?**

ข้อดีของการแยก RESTRICTIVE policy ออกมาต่างหากคือ **การแยกความรับผิดชอบ (separation of concerns)** — policy การ "ระงับบัญชี" เป็นกฎทางธุรกิจคนละเรื่องกับ "แบ่งข้อมูลตาม tenant" การเก็บแยกกันทำให้:

1. ทีมพัฒนาสามารถเปิด/ปิด/แก้ไขฟีเจอร์ suspension ได้โดยไม่แตะ policy หลักที่ทำงานสำคัญกว่า (tenant isolation)
2. อ่าน `pg_policies` แล้วเข้าใจได้ทันทีว่ามีกฎกี่ชั้น แต่ละชั้นทำหน้าที่อะไร
3. ลดความเสี่ยงจากการเขียน policy ตัวเดียวที่ยาวและซับซ้อนเกินไปจนแก้ไขผิดพลาดง่าย

> **กฎจำง่าย**: ใช้ PERMISSIVE (default) สำหรับ "เงื่อนไขทางเลือก" ที่แค่ผ่านอันใดอันหนึ่งก็พอ (เช่น "เห็นได้ถ้าเป็นเจ้าของ **หรือ** ถ้าเป็น admin ของทีม") และใช้ RESTRICTIVE สำหรับ "เงื่อนไขบังคับที่ต้องผ่านเสมอไม่ว่าอะไรจะเกิดขึ้น" (เช่น "ต้องไม่ถูกระงับบัญชี" หรือ "ต้องอยู่ในช่วงเวลาทำการ")

---

## Step 688: BYPASSRLS attribute

บางครั้งเราต้องการให้ role บางตัว **ข้าม RLS ไปเลย** เพื่อทำงานที่ต้องเข้าถึงข้อมูลข้าม tenant ทั้งหมด เช่น:

- Superuser ที่ดูแลระบบฐานข้อมูลโดยรวม
- Role สำหรับรัน migration script หรือ batch job ที่ต้องประมวลผลข้อมูลทุก tenant
- Role สำหรับทีม platform admin ที่ต้องดูภาพรวมข้ามร้านค้าทั้งหมด (เช่น รายงานสรุปยอดขายรวมของทั้งแพลตฟอร์ม)

PostgreSQL มี role attribute ชื่อ `BYPASSRLS` สำหรับกรณีนี้โดยเฉพาะ:

```sql
CREATE ROLE platform_admin LOGIN PASSWORD 'platform_admin_pass_2024' BYPASSRLS;

GRANT USAGE ON SCHEMA public TO platform_admin;
GRANT SELECT, INSERT, UPDATE, DELETE ON tenants, products, customers, orders TO platform_admin;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO platform_admin;
```

ทดสอบว่า `platform_admin` เห็นข้อมูลของทุก tenant พร้อมกัน โดยไม่ต้องตั้งค่า `app.current_tenant_id` เลย:

```sql
SET SESSION AUTHORIZATION platform_admin;

SELECT o.order_id, t.tenant_name, o.total_amount
FROM orders o
JOIN tenants t ON t.tenant_id = o.tenant_id
ORDER BY o.order_id;
```

```
 order_id |    tenant_name        | total_amount
----------+------------------------+--------------
        2 | ร้านกาแฟดารา           |        65.00
        3 | ร้านกาแฟดารา           |       100.00
        4 | ร้านหนังสือปัญญา        |       440.00
        5 | ร้านหนังสือปัญญา        |        80.00
        6 | ร้านเสื้อผ้าแฟชั่นดี     |      1180.00
        7 | ร้านเสื้อผ้าแฟชั่นดี     |       250.00
        8 | ร้านกาแฟดารา           |       199.00
(7 rows)
```

`platform_admin` เห็นข้อมูลของ **ทั้ง 3 tenant พร้อมกัน** เพราะ `BYPASSRLS` ทำให้ RLS ทุก policy บนทุกตารางไม่มีผลกับ role นี้เลย เหมาะสำหรับงาน admin/reporting ข้าม tenant

**ตรวจสอบว่า role ใดมี BYPASSRLS บ้าง**:

```sql
RESET SESSION AUTHORIZATION;

SELECT rolname, rolbypassrls, rolsuper
FROM pg_roles
WHERE rolname IN ('app_user', 'shop_staff', 'shop_owner', 'platform_admin')
ORDER BY rolname;
```

```
    rolname     | rolbypassrls | rolsuper
-----------------+--------------+----------
 app_user        | f            | f
 platform_admin  | t            | f
 shop_owner      | f            | f
 shop_staff      | f            | f
(4 rows)
```

**การเปิด/ปิด BYPASSRLS ในภายหลัง:**

```sql
-- ให้สิทธิ์ BYPASSRLS
ALTER ROLE some_role BYPASSRLS;

-- เพิกถอนสิทธิ์ BYPASSRLS
ALTER ROLE some_role NOBYPASSRLS;
```

> **superuser ก็ bypass RLS โดยอัตโนมัติเสมอ** — ไม่ว่าจะมี `BYPASSRLS` attribute ชัดเจนหรือไม่ก็ตาม เพราะ superuser มีสิทธิ์สูงสุดในระบบอยู่แล้ว ผู้เรียนที่ทดสอบผ่าน `psql` ด้วย user เริ่มต้น (เช่น `postgres`) ที่เป็น superuser จะไม่เห็นผลของ RLS เลยจนกว่าจะสลับ role ด้วย `SET SESSION AUTHORIZATION` หรือ `SET ROLE` มาเป็น role ธรรมดาก่อน — เป็นสาเหตุที่พบบ่อยที่สุดที่ทำให้ผู้เรียนสับสนว่า "ทำไม policy ไม่ทำงาน" ทั้งที่จริง ๆ แล้ว policy ทำงานถูกต้อง แต่ทดสอบด้วย role ที่ bypass RLS อยู่

**หลักปฏิบัติด้านความปลอดภัยเกี่ยวกับ BYPASSRLS:**

1. ให้สิทธิ์ `BYPASSRLS` กับ role ที่จำเป็นจริง ๆ เท่านั้น ตามหลัก **least privilege**
2. Role ที่ application เชื่อมต่อเพื่อให้บริการผู้ใช้ปลายทาง (end user) **ไม่ควรมี** `BYPASSRLS` เด็ดขาด — ควรสงวนไว้สำหรับ role ของ admin/migration/ops เท่านั้น
3. ควร audit ว่ามี role ใดบ้างที่มี `BYPASSRLS` อยู่เป็นระยะ ผ่าน query `pg_roles` ข้างต้น เพื่อป้องกัน privilege creep

```sql
RESET SESSION AUTHORIZATION;
```

---

## Step 689: Performance ของ RLS

RLS ทำงานโดยการ "แทรก" เงื่อนไขจาก policy เข้าไปใน query plan โดยอัตโนมัติ ซึ่งหมายความว่า **ประสิทธิภาพของ policy expression มีผลโดยตรงต่อประสิทธิภาพของทุก query** บนตารางนั้น ยิ่ง policy ซับซ้อนเท่าไร (เช่น มี subquery, join ข้ามตาราง, เรียกฟังก์ชันที่ไม่ใช่ `STABLE`/`IMMUTABLE`) ยิ่งกระทบ query plan มากขึ้นเท่านั้น

### ตรวจสอบ query plan ของ query ที่มี RLS

```sql
SET SESSION AUTHORIZATION shop_staff;
SET app.current_tenant_id = '1';

EXPLAIN (ANALYZE, BUFFERS, COSTS true)
SELECT * FROM orders WHERE customer_id = 1;
```

```
                                                    QUERY PLAN
--------------------------------------------------------------------------------------------------------------
 Seq Scan on orders  (cost=0.00..1.11 rows=1 width=28) (actual time=0.015..0.018 rows=1 loops=1)
   Filter: ((tenant_id = current_tenant_id()) AND (customer_id = 1) AND (NOT (SubPlan 1)))
   Rows Removed by Filter: 6
   SubPlan 1
     ->  Seq Scan on tenants t  (cost=0.00..1.04 rows=1 width=0) (actual time=0.002..0.002 rows=0 loops=1)
           Filter: ((tenant_id = orders.tenant_id) AND is_suspended)
           Rows Removed by Filter: 3
 Planning Time: 0.312 ms
 Execution Time: 0.045 ms
```

สังเกตส่วนสำคัญใน plan นี้:

1. **`Filter:`** แสดงให้เห็นว่า PostgreSQL รวมเงื่อนไข RLS (`tenant_id = current_tenant_id()`) เข้ากับเงื่อนไขจาก `WHERE` ที่เราเขียนเอง (`customer_id = 1`) และ RESTRICTIVE policy (`NOT (SubPlan 1)`) เข้าด้วยกันทั้งหมดในขั้นตอนเดียว
2. **`SubPlan 1`** คือผลจาก RESTRICTIVE policy ที่มี subquery ไปเช็คตาราง `tenants` — เห็นได้ชัดว่า policy ที่ซับซ้อนกว่า (มี subquery) จะทำให้ plan ซับซ้อนตามไปด้วย และต้องสแกนตาราง `tenants` เพิ่มเติมสำหรับทุกแถวที่ประเมิน

เมื่อตารางมีข้อมูลน้อย (หลักสิบแถวแบบในบทนี้) ผลกระทบด้านความเร็วแทบไม่รู้สึก แต่เมื่อตาราง `orders` มีข้อมูลนับล้านแถว การมี subquery ซ้อนอยู่ใน RESTRICTIVE policy ทุกครั้งที่ query จะกลายเป็นคอขวดสำคัญได้

### เทคนิคที่ 1: ใช้ index บนคอลัมน์ที่ policy ใช้กรอง

เพราะ policy จะถูกแปลงเป็น `WHERE` เพิ่มเข้าไปเสมอ index บนคอลัมน์ `tenant_id` (ที่เราสร้างไว้ตั้งแต่ต้นบท) จึงสำคัญมาก — ลอง drop index ชั่วคราวเพื่อดูผลต่าง:

```sql
RESET SESSION AUTHORIZATION;

DROP INDEX idx_orders_tenant;

SET SESSION AUTHORIZATION shop_staff;
SET app.current_tenant_id = '1';

EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders WHERE tenant_id = current_tenant_id();
```

```
 Seq Scan on orders  (cost=0.00..1.09 rows=3 width=28) (actual time=0.010..0.016 rows=3 loops=1)
   Filter: (tenant_id = current_tenant_id())
   Rows Removed by Filter: 4
 Planning Time: 0.098 ms
 Execution Time: 0.028 ms
```

ด้วยข้อมูลจำนวนน้อยนี้ planner เลือก Seq Scan อยู่ดี (เร็วกว่า index scan สำหรับตารางเล็ก) แต่เมื่อข้อมูลมีจำนวนมาก (สมมติหลักล้านแถว, หลายพัน tenant) การไม่มี index บน `tenant_id` จะบังคับให้ทุก query ต้อง scan ทั้งตารางเสมอ ซึ่งช้ามาก สร้าง index กลับคืน:

```sql
RESET SESSION AUTHORIZATION;
CREATE INDEX idx_orders_tenant ON orders(tenant_id);

-- สำหรับ query ที่มักกรองด้วย tenant_id ร่วมกับคอลัมน์อื่นบ่อย ๆ
-- ควรพิจารณาสร้าง composite index ด้วย เช่น:
CREATE INDEX idx_orders_tenant_customer ON orders(tenant_id, customer_id);
```

### เทคนิคที่ 2: ใช้ฟังก์ชันแบบ STABLE เสมอ

ฟังก์ชันที่ใช้ใน policy (เช่น `current_tenant_id()`) ควรถูกประกาศเป็น `STABLE` (ไม่ใช่ `VOLATILE` ซึ่งเป็นค่า default) เพราะ:

- `STABLE` บอก planner ว่าฟังก์ชันนี้คืนค่าเดิมเสมอภายใน statement เดียวกัน (ไม่เปลี่ยนแปลงระหว่างแถว) ทำให้ PostgreSQL สามารถ **cache ผลลัพธ์ไว้ครั้งเดียว** แทนที่จะเรียกฟังก์ชันซ้ำทุกแถว
- ถ้าใช้ฟังก์ชันแบบ `VOLATILE` (default) planner จะไม่กล้า optimize เพราะไม่รู้ว่าค่าจะเปลี่ยนระหว่างแถวหรือไม่ ทำให้เรียกฟังก์ชันซ้ำทุกแถว ซึ่งช้ากว่ามาก

ตรวจสอบว่าฟังก์ชันของเรา (จาก Step 685) เป็น `STABLE` แล้ว:

```sql
SELECT proname, provolatile
FROM pg_proc
WHERE proname = 'current_tenant_id';
```

```
      proname       | provolatile
---------------------+-------------
 current_tenant_id   | s
```

`s` หมายถึง STABLE — ถูกต้องแล้วตามที่ประกาศไว้ (`s` = stable, `i` = immutable, `v` = volatile)

### เทคนิคที่ 3: ห่อ current_setting() ด้วย subquery `(select ...)` เพื่อบังคับ initPlan

เทคนิคขั้นสูงที่ใช้กันในระบบจริงคือการห่อฟังก์ชันด้วย `(select ...)` เพื่อบอก planner ให้ประเมินค่าเพียง **ครั้งเดียว** ต่อ statement (กลายเป็น InitPlan) แทนที่จะประเมินซ้ำทุกแถว แม้ฟังก์ชันจะเป็น `STABLE` อยู่แล้วก็ตาม บาง PostgreSQL version วางแผนได้ดีกว่าเมื่อเขียนแบบนี้ชัดเจน:

```sql
-- แทนที่จะเขียน:
--   USING (tenant_id = current_tenant_id())
-- เขียนแบบนี้เพื่อบังคับ initPlan:
--   USING (tenant_id = (SELECT current_tenant_id()))
```

ทดสอบเปรียบเทียบ plan:

```sql
RESET SESSION AUTHORIZATION;

DROP POLICY orders_select_policy ON orders;

CREATE POLICY orders_select_policy ON orders
    FOR SELECT
    TO shop_staff, shop_owner
    USING (tenant_id = (SELECT current_tenant_id()));

SET SESSION AUTHORIZATION shop_staff;
SET app.current_tenant_id = '1';

EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders;
```

```
 Seq Scan on orders  (cost=0.01..1.10 rows=3 width=28) (actual time=0.014..0.020 rows=3 loops=1)
   Filter: ((tenant_id = $0) AND (NOT (SubPlan 2)))
   InitPlan 1 (returns $0)
     ->  Result  (cost=0.00..0.01 rows=1 width=4) (actual time=0.001..0.001 rows=1 loops=1)
   SubPlan 2
     ->  Seq Scan on tenants t  (cost=0.00..1.04 rows=1 width=0) (actual time=0.001..0.001 rows=1 loops=1)
           Filter: ((tenant_id = orders.tenant_id) AND is_suspended)
           Rows Removed by Filter: 2
 Planning Time: 0.201 ms
 Execution Time: 0.033 ms
```

สังเกตว่าตอนนี้มี **`InitPlan 1 (returns $0)`** แยกออกมาต่างหาก และ `Filter` ใช้ `$0` (ค่าคงที่ที่ถูกคำนวณครั้งเดียว) แทนที่จะเรียก `current_tenant_id()` ซ้ำทุกแถว — สำหรับตารางขนาดใหญ่ (นับล้านแถว) ความแตกต่างนี้มีนัยสำคัญมาก เพราะลดจำนวนครั้งที่ต้องเรียก `current_setting()` จาก "ทุกแถว" เหลือ "ครั้งเดียวต่อ query"

### เทคนิคที่ 4: หลีกเลี่ยง policy ที่มี JOIN/subquery ซับซ้อนเกินจำเป็น

จากตัวอย่าง `orders_not_suspended` ใน Step 687 ที่มี `EXISTS (SELECT ... FROM tenants ...)` — ถ้าตาราง `tenants` มี index บน `tenant_id` (เป็น primary key อยู่แล้ว) subquery นี้จะเร็วมากเพราะเป็นการ lookup ด้วย index เสมอ แต่ถ้า policy ซับซ้อนกว่านี้ เช่น join กับหลายตาราง หรือมี aggregate function ควรพิจารณา:

- แคชผลลัพธ์ไว้ใน session variable ตั้งแต่ต้น (เช่น ตรวจสอบสถานะ suspended ตอน login แล้ว set ค่าผ่าน `app.tenant_suspended` แทนที่จะ query ทุกครั้ง)
- ใช้ **materialized view** สำหรับข้อมูลที่ policy ต้องอ้างอิงบ่อย ๆ แต่เปลี่ยนแปลงไม่บ่อย
- Benchmark ด้วย `EXPLAIN (ANALYZE, BUFFERS)` เปรียบเทียบก่อน/หลังเพิ่ม policy เสมอ ก่อนนำขึ้น production

```sql
RESET SESSION AUTHORIZATION;
```

> **สรุปหลักปฏิบัติด้านประสิทธิภาพ RLS**: (1) สร้าง index บนคอลัมน์ที่ policy ใช้กรองเสมอ (2) ประกาศฟังก์ชันที่ policy เรียกใช้เป็น `STABLE` หรือ `IMMUTABLE` เท่าที่เป็นไปได้ (3) พิจารณาห่อฟังก์ชันด้วย `(SELECT ...)` เพื่อบังคับ initPlan ในกรณีที่ query ซับซ้อน (4) รัน `EXPLAIN (ANALYZE, BUFFERS)` ทดสอบเปรียบเทียบเสมอเมื่อเพิ่ม policy ใหม่ โดยเฉพาะกับตารางที่มีข้อมูลจำนวนมาก

---

## Step 690: แบบฝึกหัดรวม — Multi-tenant SaaS เต็มรูปแบบด้วย RLS

ในขั้นตอนสุดท้ายนี้ เราจะรวมทุกสิ่งที่เรียนมาในบทนี้ ทำความสะอาด policy ทั้งหมด และสร้างระบบ RLS ที่สมบูรณ์แบบ production-ready พร้อมพิสูจน์ด้วยการทดสอบจริงแบบละเอียดว่า tenant หนึ่งไม่สามารถเห็นหรือแก้ไขข้อมูลของอีก tenant ได้จริง ไม่ว่าจะพยายามด้วยวิธีใดก็ตาม

### ขั้นที่ 1: ทำความสะอาด policy เดิมทั้งหมด

```sql
DROP POLICY IF EXISTS tenant_isolation_products  ON products;
DROP POLICY IF EXISTS tenant_isolation_customers ON customers;
DROP POLICY IF EXISTS orders_select_policy        ON orders;
DROP POLICY IF EXISTS orders_insert_policy        ON orders;
DROP POLICY IF EXISTS orders_update_policy        ON orders;
DROP POLICY IF EXISTS orders_delete_policy        ON orders;
DROP POLICY IF EXISTS orders_not_suspended        ON orders;
DROP POLICY IF EXISTS tenants_select_all          ON tenants;
```

### ขั้นที่ 2: สร้างชุด policy ที่สมบูรณ์แบบ production-ready

```sql
-- ===== ตาราง tenants =====
-- ทุก role ที่ login ได้อ่านรายชื่อ tenant ได้ (ไม่มีข้อมูลอ่อนไหว) แต่แก้ไขได้เฉพาะ platform_admin
CREATE POLICY tenants_read_all ON tenants
    FOR SELECT
    TO shop_staff, shop_owner, app_user
    USING (true);

-- ===== ตาราง products =====
CREATE POLICY products_tenant_isolation ON products
    FOR ALL
    TO shop_staff, shop_owner, app_user
    USING (tenant_id = (SELECT current_tenant_id()))
    WITH CHECK (tenant_id = (SELECT current_tenant_id()));

-- ===== ตาราง customers =====
CREATE POLICY customers_tenant_isolation ON customers
    FOR ALL
    TO shop_staff, shop_owner, app_user
    USING (tenant_id = (SELECT current_tenant_id()))
    WITH CHECK (tenant_id = (SELECT current_tenant_id()));

-- ===== ตาราง orders =====
CREATE POLICY orders_select_policy ON orders
    FOR SELECT
    TO shop_staff, shop_owner, app_user
    USING (tenant_id = (SELECT current_tenant_id()));

CREATE POLICY orders_insert_policy ON orders
    FOR INSERT
    TO shop_staff, shop_owner, app_user
    WITH CHECK (tenant_id = (SELECT current_tenant_id()));

CREATE POLICY orders_update_policy ON orders
    FOR UPDATE
    TO shop_staff, shop_owner, app_user
    USING (tenant_id = (SELECT current_tenant_id()))
    WITH CHECK (tenant_id = (SELECT current_tenant_id()));

CREATE POLICY orders_delete_policy ON orders
    FOR DELETE
    TO shop_owner
    USING (tenant_id = (SELECT current_tenant_id()));

-- RESTRICTIVE policy: บล็อกทุกการเข้าถึงถ้า tenant ถูกระงับ
CREATE POLICY orders_not_suspended ON orders
    AS RESTRICTIVE
    FOR ALL
    TO shop_staff, shop_owner, app_user
    USING (
        NOT EXISTS (
            SELECT 1 FROM tenants t
            WHERE t.tenant_id = orders.tenant_id AND t.is_suspended = true
        )
    );
```

### ขั้นที่ 3: บังคับ RLS แม้กับ table owner เพื่อความปลอดภัยสูงสุด

```sql
ALTER TABLE tenants   FORCE ROW LEVEL SECURITY;
ALTER TABLE products  FORCE ROW LEVEL SECURITY;
ALTER TABLE customers FORCE ROW LEVEL SECURITY;
ALTER TABLE orders    FORCE ROW LEVEL SECURITY;
```

### ขั้นที่ 4: ตรวจสอบภาพรวม policy ทั้งหมดในระบบ

```sql
SELECT tablename, policyname, permissive, cmd, roles
FROM pg_policies
ORDER BY tablename, policyname;
```

```
 tablename |         policyname          | permissive |  cmd   |            roles
-----------+------------------------------+------------+--------+-------------------------------
 customers | customers_tenant_isolation   | PERMISSIVE | ALL    | {shop_staff,shop_owner,app_user}
 orders    | orders_delete_policy         | PERMISSIVE | DELETE | {shop_owner}
 orders    | orders_insert_policy         | PERMISSIVE | INSERT | {shop_staff,shop_owner,app_user}
 orders    | orders_not_suspended         | RESTRICTIVE| ALL    | {shop_staff,shop_owner,app_user}
 orders    | orders_select_policy         | PERMISSIVE | SELECT | {shop_staff,shop_owner,app_user}
 orders    | orders_update_policy         | PERMISSIVE | UPDATE | {shop_staff,shop_owner,app_user}
 products  | products_tenant_isolation    | PERMISSIVE | ALL    | {shop_staff,shop_owner,app_user}
 tenants   | tenants_read_all             | PERMISSIVE | SELECT | {shop_staff,shop_owner,app_user}
(8 rows)
```

### ขั้นที่ 5: ชุดทดสอบเต็มรูปแบบ — พิสูจน์ tenant isolation

**การทดสอบที่ 1: tenant 1 (shop_staff) เห็นเฉพาะข้อมูลของตัวเอง**

```sql
SET SESSION AUTHORIZATION shop_staff;
SET app.current_tenant_id = '1';

SELECT product_name, unit_price FROM products ORDER BY product_id;
```

```
     product_name       | unit_price
--------------------------+------------
 กาแฟลาเต้ร้อน            |      65.00
 กาแฟอเมริกาโน่เย็น        |      55.00
 ครัวซองต์เนยสด            |      45.00
(3 rows)
```

**การทดสอบที่ 2: สลับเป็น tenant 2 ในเชื่อมต่อเดียวกัน — เห็นข้อมูลคนละชุดทันที**

```sql
SET app.current_tenant_id = '2';

SELECT product_name, unit_price FROM products ORDER BY product_id;
```

```
      product_name        | unit_price
----------------------------+------------
 หนังสือนิยายไทยยอดนิยม      |     250.00
 หนังสือพัฒนาตนเอง           |     190.00
 สมุดโน้ตปกหนัง               |      80.00
(3 rows)
```

**การทดสอบที่ 3: พยายาม query ข้าม tenant ด้วยการระบุ tenant_id ตรง ๆ ใน WHERE (โจมตีแบบ manual)**

```sql
-- session นี้ยังคงเป็น tenant 2 แต่พยายามสอดแนม tenant 1 ด้วยการระบุ WHERE ตรง ๆ
SELECT * FROM products WHERE tenant_id = 1;
```

```
 product_id | tenant_id | product_name | unit_price
------------+-----------+--------------+------------
(0 rows)
```

แม้จะระบุ `WHERE tenant_id = 1` ตรง ๆ ก็ไม่สามารถเห็นข้อมูลของ tenant 1 ได้ เพราะ RLS บังคับ `AND tenant_id = current_tenant_id()` เข้าไปทับซ้อนกับ `WHERE` ที่ผู้ใช้เขียนเองเสมอ — นี่คือหัวใจของความปลอดภัยที่ RLS มอบให้ ซึ่ง `GRANT`/`REVOKE` ธรรมดาไม่สามารถทำได้

**การทดสอบที่ 4: พยายาม UPDATE ข้ามไปยัง tenant อื่น**

```sql
-- session ยังคงเป็น tenant 2, พยายามแก้ไขราคาสินค้าของ tenant 1 (product_id 1)
UPDATE products SET unit_price = 0.01 WHERE product_id = 1
RETURNING *;
```

```
UPDATE 0
```

`UPDATE 0` — ไม่มีแถวใดถูกแก้ไข เพราะ `USING` clause ของ policy กรอง product_id นั้นออกไปตั้งแต่ต้น (มองไม่เห็นแถวนั้นด้วยซ้ำ)

**การทดสอบที่ 5: พยายาม INSERT ข้อมูลปลอมแฝงเป็น tenant อื่น**

```sql
INSERT INTO customers (tenant_id, first_name, last_name, email)
VALUES (1, 'แฮกเกอร์', 'จอมปลอม', 'hacker@evil.com');
```

```
ERROR:  new row violates row-level security policy for table "customers"
```

`WITH CHECK` บล็อกการพยายามสร้างข้อมูลปลอมที่แอบอ้างเป็นของ tenant อื่นทันที

**การทดสอบที่ 6: พยายาม JOIN ข้าม tenant เพื่อดึงข้อมูลลูกค้าของร้านอื่น**

```sql
-- session ยังเป็น tenant 2, พยายาม join กับ orders ของ tenant 1 โดยหวังว่า join จะข้าม RLS ได้
SELECT c.first_name, o.order_id, o.total_amount
FROM customers c
JOIN orders o ON o.customer_id = c.customer_id
WHERE o.tenant_id = 1;
```

```
 first_name | order_id | total_amount
------------+----------+--------------
(0 rows)
```

RLS ทำงานกับทุกตารางที่เกี่ยวข้องใน JOIN โดยอิสระต่อกัน — ทั้ง `customers` และ `orders` ถูกกรองด้วย policy ของตัวเอง ทำให้ JOIN ไม่สามารถหลุดรอดออกจาก tenant boundary ได้เลยไม่ว่าจะซับซ้อนแค่ไหน

**การทดสอบที่ 7: ทดสอบ RESTRICTIVE policy (ระงับบัญชี) ทำงานถูกต้อง**

```sql
RESET SESSION AUTHORIZATION;
UPDATE tenants SET is_suspended = true WHERE tenant_id = 2;

SET SESSION AUTHORIZATION shop_staff;
SET app.current_tenant_id = '2';

SELECT COUNT(*) FROM orders;
SELECT COUNT(*) FROM products;
```

```
 count
-------
     0
(1 row)

 count
-------
     3
(1 row)
```

สังเกตว่า `orders` คืน 0 แถว (เพราะมี RESTRICTIVE policy `orders_not_suspended` ครอบอยู่) แต่ `products` ยัง**เห็นข้อมูลตามปกติ (3 แถว)** — นี่เป็นพฤติกรรมที่ถูกต้องตามที่ออกแบบไว้ เพราะเราสร้าง RESTRICTIVE policy `orders_not_suspended` ไว้เฉพาะตาราง `orders` เท่านั้น ไม่ได้ครอบคลุม `products`/`customers` ด้วย — นี่เป็นตัวอย่างที่ดีว่าทำไมต้องออกแบบ policy ให้ครอบคลุมทุกตารางที่เกี่ยวข้องกับ business rule เดียวกันอย่างสม่ำเสมอ (ในระบบจริงควรเพิ่ม RESTRICTIVE policy แบบเดียวกันให้ `products` และ `customers` ด้วยถ้าต้องการให้ suspension มีผลครอบคลุมทั้งหมด)

คืนค่ากลับสู่สถานะปกติ:

```sql
RESET SESSION AUTHORIZATION;
UPDATE tenants SET is_suspended = false WHERE tenant_id = 2;
```

**การทดสอบที่ 8: platform_admin (BYPASSRLS) เห็นข้อมูลทุก tenant พร้อมกันสำหรับรายงานสรุป**

```sql
SET SESSION AUTHORIZATION platform_admin;

SELECT t.tenant_name,
       COUNT(o.order_id) AS total_orders,
       COALESCE(SUM(o.total_amount), 0) AS total_revenue
FROM tenants t
LEFT JOIN orders o ON o.tenant_id = t.tenant_id
GROUP BY t.tenant_id, t.tenant_name
ORDER BY t.tenant_id;
```

```
      tenant_name        | total_orders | total_revenue
---------------------------+--------------+----------------
 ร้านกาแฟดารา              |            3 |         364.00
 ร้านหนังสือปัญญา           |            2 |         520.00
 ร้านเสื้อผ้าแฟชั่นดี        |            2 |        1430.00
(3 rows)
```

`platform_admin` สามารถสร้างรายงานสรุปข้ามทุก tenant ได้อย่างสมบูรณ์ เพราะมี `BYPASSRLS` — แสดงให้เห็นว่าระบบสามารถรองรับทั้งการแยก tenant อย่างเข้มงวดสำหรับ end user และการดูภาพรวมสำหรับทีม platform ได้พร้อมกัน โดยใช้ role ที่ต่างกันเท่านั้น

```sql
RESET SESSION AUTHORIZATION;
```

**การทดสอบที่ 9: ยืนยันว่า customer ของ tenant หนึ่งไม่ปรากฏใน query ของอีก tenant แม้ email ซ้ำ**

```sql
-- เพิ่มลูกค้า email ซ้ำกันในคนละ tenant เพื่อทดสอบว่าไม่ปนกัน
INSERT INTO customers (tenant_id, first_name, last_name, email) VALUES
    (1, 'กิตติ', 'ทดสอบ', 'shared@example.com'),
    (2, 'กิตติ', 'ทดสอบ', 'shared@example.com');

SET SESSION AUTHORIZATION shop_staff;
SET app.current_tenant_id = '1';

SELECT customer_id, tenant_id, email FROM customers WHERE email = 'shared@example.com';
```

```
 customer_id | tenant_id |        email
-------------+-----------+---------------------
           6 |         1 | shared@example.com
(1 row)
```

แม้จะมี email ซ้ำกันในระบบ (ข้ามคนละ tenant) แต่ `shop_staff` ที่อยู่ใน context ของ tenant 1 เห็นเพียงแถวเดียวที่เป็นของ tenant ตัวเองเท่านั้น — พิสูจน์ว่าการแยกข้อมูลทำงานถูกต้องแม้ข้อมูลจะดู "เหมือนกัน" ในเชิง business ก็ตาม

```sql
RESET SESSION AUTHORIZATION;
```

**สรุปผลการทดสอบทั้งหมด**: ระบบ multi-tenant SaaS ที่สร้างขึ้นด้วย RLS ในบทนี้ผ่านการทดสอบครบทุกกรณี ทั้งการ SELECT, INSERT, UPDATE, DELETE, JOIN ข้ามตาราง, และแม้แต่ความพยายาม "โจมตี" ด้วยการระบุ `WHERE tenant_id = ...` ตรง ๆ ก็ไม่สามารถข้ามพ้น tenant boundary ได้เลย ในขณะที่ role พิเศษอย่าง `platform_admin` (BYPASSRLS) ยังคงทำงานข้าม tenant ได้ตามที่ต้องการสำหรับงาน admin

---

## สรุปท้ายบท

ในบทนี้เราได้เรียนรู้ Row Level Security (RLS) ซึ่งเป็นกลไกความปลอดภัยระดับแถวที่ทำงานอยู่ที่ตัวฐานข้อมูลเอง ไม่ใช่ที่ application layer สาระสำคัญที่ควรจำ:

1. **RLS คือการกรองแถวที่ระดับฐานข้อมูล** — เปิดใช้งานด้วย `ALTER TABLE ... ENABLE ROW LEVEL SECURITY` และกำหนดกฎด้วย `CREATE POLICY` เมื่อเปิด RLS แต่ยังไม่มี policy = **default deny** (ไม่มีใครเห็นข้อมูลเลยยกเว้น owner)
2. **RLS เป็น defense in depth ที่สำคัญที่สุดสำหรับสถาปัตยกรรม multi-tenant SaaS** — ทำงานทุกช่องทางการเข้าถึงข้อมูล และไม่ต้องพึ่งวินัยของโปรแกรมเมอร์ในการใส่ `WHERE tenant_id = ...` ทุกจุด
3. **`USING`** ควบคุมว่าแถวที่มีอยู่แล้วมองเห็น/แก้ไข/ลบได้หรือไม่ ส่วน **`WITH CHECK`** ควบคุมว่าแถวใหม่ที่กำลังจะเขียนเข้าไปถูกต้องหรือไม่
4. **`current_setting()` คู่กับ `SET LOCAL`** เป็นกลไกมาตรฐานในการส่งค่า context (เช่น tenant_id ปัจจุบัน) เข้าไปให้ policy ใช้ — ควรใช้ `SET LOCAL` ในระบบจริงเพื่อป้องกันค่ารั่วไหลข้าม transaction เมื่อใช้ connection pooling
5. **Policy แยกตามคำสั่ง (`FOR`) และ role (`TO`)** ทำให้ออกแบบสิทธิ์ได้ละเอียด เช่น พนักงานแก้ไขได้แต่ลบไม่ได้ ในขณะที่เจ้าของร้านทำได้ทุกอย่าง
6. **PERMISSIVE policy รวมกันด้วย OR, RESTRICTIVE policy บังคับด้วย AND** — ใช้ RESTRICTIVE สำหรับกฎที่ต้องผ่านเสมอไม่มีข้อยกเว้น เช่น การระงับบัญชี
7. **`BYPASSRLS`** เหมาะสำหรับ role พิเศษที่ต้องข้าม RLS เช่น platform admin หรือ migration script ควรให้สิทธิ์นี้อย่างระมัดระวังตามหลัก least privilege
8. **ประสิทธิภาพของ RLS ขึ้นอยู่กับ index, ความ STABLE ของฟังก์ชัน, และความซับซ้อนของ policy expression** — ตรวจสอบด้วย `EXPLAIN (ANALYZE, BUFFERS)` เสมอก่อนนำขึ้น production
9. **การทดสอบจริงเป็นสิ่งจำเป็น** — ไม่ควรเชื่อว่า policy ทำงานถูกต้องจนกว่าจะทดสอบด้วยการสลับ role/session variable จริง และพยายาม "โจมตี" ระบบด้วยวิธีต่าง ๆ ด้วยตัวเอง

RLS เป็นหนึ่งในเครื่องมือทรงพลังที่สุดของ PostgreSQL สำหรับการสร้างระบบ multi-tenant ที่ปลอดภัยในระดับฐานข้อมูล และเป็นพื้นฐานสำคัญก่อนที่จะก้าวไปสู่หัวข้อความปลอดภัยขั้นสูงถัดไปอย่างการเข้ารหัสการเชื่อมต่อด้วย SSL/TLS

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1
อธิบายความแตกต่างระหว่างการควบคุมสิทธิ์ด้วย `GRANT`/`REVOKE` กับ Row Level Security (RLS) ว่าแต่ละแบบเหมาะกับสถานการณ์ใด

<details>
<summary>เฉลย</summary>

`GRANT`/`REVOKE` ควบคุมสิทธิ์ที่ **ระดับ object ทั้งชิ้น** เช่น ตารางทั้งตาราง หรือคอลัมน์ทั้งคอลัมน์ — ตอบคำถามว่า "role นี้ทำอะไรกับตารางนี้ได้บ้าง (SELECT/INSERT/UPDATE/DELETE)" แต่ไม่สามารถจำกัดได้ว่า **แถวไหน** ที่ role นั้นเข้าถึงได้

RLS ควบคุมสิทธิ์ที่ **ระดับแถว (row)** ภายในตารางเดียวกัน — ตอบคำถามว่า "จากแถวทั้งหมดในตารางนี้ที่ role มีสิทธิ์เข้าถึงอยู่แล้ว (ผ่าน GRANT) แถวไหนที่มองเห็น/แก้ไขได้จริง"

ทั้งสองแบบทำงานร่วมกัน ไม่ใช่แทนที่กัน: `GRANT` ให้สิทธิ์ระดับตารางก่อน แล้ว RLS จึงกรองแถวภายในตารางนั้นอีกชั้นหนึ่ง — เหมาะกับ multi-tenant SaaS, ระบบที่ผู้ใช้แต่ละคนเห็นเฉพาะข้อมูลตัวเอง, หรือระบบที่มีข้อมูลอ่อนไหวที่ต้องจำกัดการเข้าถึงแบบละเอียด
</details>

### แบบฝึกหัดที่ 2
จากตาราง `products` ในบทนี้ ให้เขียนคำสั่งเปิดใช้งาน RLS แล้วทดสอบว่าถ้ายังไม่มี policy ใด ๆ role `app_user` จะเห็นข้อมูลกี่แถว พร้อมอธิบายเหตุผล

<details>
<summary>เฉลย</summary>

```sql
ALTER TABLE products ENABLE ROW LEVEL SECURITY;

SET SESSION AUTHORIZATION app_user;
SELECT COUNT(*) FROM products;
-- ผลลัพธ์: 0

RESET SESSION AUTHORIZATION;
```

เห็น **0 แถว** เพราะเมื่อเปิด RLS บนตารางแล้วแต่ยังไม่มี `CREATE POLICY` ใด ๆ รองรับ role นั้น พฤติกรรม default ของ RLS คือ **default deny** — ไม่มีแถวใดผ่านเงื่อนไข (เพราะไม่มีเงื่อนไขให้ผ่านเลย) role ที่ไม่ใช่ table owner จึงมองไม่เห็นข้อมูลแม้แต่แถวเดียว
</details>

### แบบฝึกหัดที่ 3
อธิบายความแตกต่างระหว่าง `USING` และ `WITH CHECK` ใน `CREATE POLICY` พร้อมยกตัวอย่างสถานการณ์ที่ต้องใช้ทั้งสองอย่างร่วมกัน

<details>
<summary>เฉลย</summary>

- `USING` ตรวจสอบ **แถวที่มีอยู่แล้ว** ใช้กับ SELECT (กรองว่าแสดงแถวไหน), UPDATE และ DELETE (กรองว่าแก้ไข/ลบแถวไหนได้) — ถ้าแถวไม่ผ่าน `USING` จะถูกซ่อนไปเฉย ๆ ไม่ error
- `WITH CHECK` ตรวจสอบ **แถวใหม่ที่กำลังจะเขียน** ใช้กับ INSERT (ค่าที่กำลังจะ insert) และ UPDATE (ค่าผลลัพธ์หลัง update) — ถ้าไม่ผ่าน จะ **error** ทันที (`new row violates row-level security policy`)

ตัวอย่างที่ต้องใช้ร่วมกัน: policy สำหรับ `FOR UPDATE` ในระบบ multi-tenant ต้องมีทั้งคู่ — `USING (tenant_id = current_tenant_id())` เพื่อจำกัดว่าแก้ไขได้เฉพาะแถวของ tenant ตัวเอง และ `WITH CHECK (tenant_id = current_tenant_id())` เพื่อป้องกันไม่ให้ระหว่างการ UPDATE มีการเปลี่ยน `tenant_id` ของแถวนั้นให้กลายเป็นของ tenant อื่น (data exfiltration ผ่านการ "ย้ายเจ้าของ" แถว)
</details>

### แบบฝึกหัดที่ 4
เขียน policy สำหรับตาราง `customers` ที่อนุญาตให้ role `shop_staff` ทำได้เฉพาะ SELECT และ UPDATE เท่านั้น (ห้าม INSERT และ DELETE) โดยจำกัดตาม tenant_id ปัจจุบัน

<details>
<summary>เฉลย</summary>

```sql
CREATE POLICY customers_staff_select ON customers
    FOR SELECT
    TO shop_staff
    USING (tenant_id = current_tenant_id());

CREATE POLICY customers_staff_update ON customers
    FOR UPDATE
    TO shop_staff
    USING (tenant_id = current_tenant_id())
    WITH CHECK (tenant_id = current_tenant_id());
```

เนื่องจากไม่สร้าง policy สำหรับ `FOR INSERT` และ `FOR DELETE` ให้ role `shop_staff` เลย คำสั่ง INSERT/DELETE จากบทบาทนี้จะถูกปฏิเสธโดยอัตโนมัติ (INSERT จะ error เพราะไม่มี `WITH CHECK` ที่อนุญาต, DELETE จะคืนค่า 0 แถวเสมอเพราะไม่มี `USING` ที่อนุญาตให้เห็นแถวใดเลยสำหรับคำสั่งนี้)
</details>

### แบบฝึกหัดที่ 5
อธิบายว่าทำไมควรใช้ `SET LOCAL` แทน `SET` ธรรมดาในการตั้งค่า `app.current_tenant_id` สำหรับระบบ production ที่ใช้ connection pooling

<details>
<summary>เฉลย</summary>

`SET` ธรรมดาจะตั้งค่าที่ระดับ **session** ซึ่งจะคงอยู่จนกว่าจะเปลี่ยนหรือปิด connection ในขณะที่ระบบ production มักใช้ connection pooling (เช่น PgBouncer หรือ pool ของ ORM) ที่ connection เดียวกันจะถูกใช้ซ้ำโดย request ของผู้ใช้/tenant ต่างกันในเวลาไล่เลี่ยกัน ถ้าใช้ `SET` ธรรมดาแล้วลืม reset ค่า มีความเสี่ยงที่ request ของ tenant B จะ "สืบทอด" ค่า `app.current_tenant_id` ของ tenant A ที่ค้างอยู่จาก request ก่อนหน้าบน connection เดียวกัน ทำให้เกิด cross-tenant data leak ร้ายแรง

`SET LOCAL` ตั้งค่าที่ระดับ **transaction** เท่านั้น และจะถูกล้างค่าอัตโนมัติทันทีที่ transaction จบ (ทั้ง COMMIT และ ROLLBACK) ทำให้ปลอดภัยกว่ามากในสภาพแวดล้อมที่มีการใช้ connection ซ้ำ — แนวทางที่แนะนำคือเริ่มทุก request ด้วย `BEGIN; SET LOCAL app.current_tenant_id = '...'; ... COMMIT;` เสมอ
</details>

### แบบฝึกหัดที่ 6
กำหนดสถานการณ์: มี policy สองตัวบนตาราง `orders` — Policy A เป็น PERMISSIVE ที่อนุญาตให้เห็นแถวที่ `tenant_id = current_tenant_id()`, Policy B เป็น PERMISSIVE ที่อนุญาตให้เห็นแถวที่ `total_amount > 1000` (สำหรับ role ผู้ตรวจสอบพิเศษ) ถามว่าถ้าทั้งสอง policy มีผลกับ role เดียวกัน แถวที่มี `total_amount > 1000` แต่เป็นของ tenant อื่น จะมองเห็นได้หรือไม่ เพราะเหตุใด

<details>
<summary>เฉลย</summary>

**เห็นได้** เพราะทั้ง Policy A และ Policy B เป็น PERMISSIVE ซึ่งถูกรวมกันด้วย **OR** — แถวใดแถวหนึ่งผ่านแค่ policy เดียวก็เพียงพอที่จะมองเห็นได้ ดังนั้นแถวที่ `total_amount > 1000` (ผ่าน Policy B) แม้จะเป็นของ tenant อื่น (ไม่ผ่าน Policy A) ก็ยังคงมองเห็นได้อยู่ดี เพราะ `(Policy A) OR (Policy B)` เป็นจริงเมื่อฝั่งใดฝั่งหนึ่งเป็นจริง

หากต้องการให้ tenant isolation เป็นเงื่อนไข **บังคับที่ต้องผ่านเสมอ** ไม่ว่า policy อื่นจะอนุญาตอะไรก็ตาม ต้องเปลี่ยน Policy A (การกรอง tenant) ให้เป็น `AS RESTRICTIVE` แทน เพื่อให้ผลลัพธ์กลายเป็น `(Policy B) AND (Policy A)` ซึ่งจะบังคับให้ต้องอยู่ใน tenant ตัวเองเสมอ ไม่ว่า Policy B จะอนุญาตอะไรก็ตาม
</details>

### แบบฝึกหัดที่ 7
สร้าง role ชื่อ `audit_reader` ที่มีสิทธิ์ `BYPASSRLS` และสามารถ SELECT ได้ทุกตารางในบทนี้ (แต่ไม่ให้ INSERT/UPDATE/DELETE) แล้วทดสอบว่า role นี้เห็นจำนวนแถวทั้งหมดในตาราง `orders` เท่ากับกี่แถว (ไม่ต้องตั้งค่า `app.current_tenant_id`)

<details>
<summary>เฉลย</summary>

```sql
CREATE ROLE audit_reader LOGIN PASSWORD 'audit_pass_2024' BYPASSRLS;

GRANT USAGE ON SCHEMA public TO audit_reader;
GRANT SELECT ON tenants, products, customers, orders TO audit_reader;

SET SESSION AUTHORIZATION audit_reader;
SELECT COUNT(*) FROM orders;   -- เห็นทุกแถวของทุก tenant เพราะมี BYPASSRLS
RESET SESSION AUTHORIZATION;
```

เนื่องจากมี `BYPASSRLS` role นี้จะเห็นข้อมูลทั้งหมดในตาราง `orders` ของทุก tenant รวมกัน (ตามจำนวนแถวจริงในตาราง ไม่ถูกกรองโดย policy ใด ๆ เลย) แม้จะไม่ได้ตั้งค่า `app.current_tenant_id` เลยก็ตาม เพราะ `BYPASSRLS` ทำให้ policy ทุกตัวบนทุกตารางไม่มีผลกับ role นี้ตั้งแต่แรก
</details>

### แบบฝึกหัดที่ 8
เขียน query `EXPLAIN (ANALYZE, BUFFERS)` เพื่อตรวจสอบว่า policy ที่ใช้ในตาราง `orders` (จาก Step 690) ถูกแปลงเป็นเงื่อนไขอะไรใน query plan เมื่อรันคำสั่ง `SELECT * FROM orders WHERE total_amount > 100;` ในฐานะ `shop_staff` ที่ตั้งค่า `app.current_tenant_id = '3'`

<details>
<summary>เฉลย</summary>

```sql
SET SESSION AUTHORIZATION shop_staff;
SET app.current_tenant_id = '3';

EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders WHERE total_amount > 100;

RESET SESSION AUTHORIZATION;
```

ใน `Filter:` ของ query plan จะเห็นเงื่อนไขที่ PostgreSQL รวมเข้าด้วยกันโดยอัตโนมัติ ประกอบด้วย 3 ส่วน:
1. เงื่อนไขจาก `WHERE` ที่เขียนเอง: `total_amount > 100`
2. เงื่อนไขจาก PERMISSIVE policy `orders_select_policy`: `tenant_id = current_tenant_id()` (หรือ `$0` ถ้าห่อด้วย subquery)
3. เงื่อนไขจาก RESTRICTIVE policy `orders_not_suspended`: `NOT (SubPlan ...)` ที่ตรวจสอบสถานะ suspended ของ tenant ผ่านการ query ตาราง `tenants`

ทั้งสามเงื่อนไขจะถูก AND รวมกันในขั้นตอนเดียว แสดงให้เห็นว่า RLS ไม่ได้สร้าง query แยกต่างหาก แต่ "ผสาน" เข้ากับ query plan เดียวกับที่ผู้ใช้เขียนโดยตรง
</details>

### แบบฝึกหัดที่ 9
สมมติว่าทีมพัฒนาต้องการเพิ่มฟีเจอร์ "โหมดบำรุงรักษาฉุกเฉิน" (emergency read-only mode) ที่เมื่อเปิดใช้งาน จะทำให้ **ทุก role ยกเว้น platform_admin** ไม่สามารถ INSERT/UPDATE/DELETE ข้อมูลในตาราง `orders` ได้เลย (อ่านได้อย่างเดียว) ให้ออกแบบวิธี implement ด้วย RLS

<details>
<summary>เฉลย</summary>

วิธีที่เหมาะสมคือสร้าง **RESTRICTIVE policy** ที่ครอบคลุมเฉพาะคำสั่งเขียนข้อมูล (INSERT, UPDATE, DELETE) โดยตรวจสอบค่า config แบบ global (ไม่ผูกกับ tenant) ว่าอยู่ในโหมดบำรุงรักษาหรือไม่:

```sql
-- ตั้งค่า global config (เช่นเก็บไว้ใน postgresql.conf, ALTER SYSTEM, หรือตารางตั้งค่า)
-- สมมติใช้ session variable global สำหรับตัวอย่างนี้:
-- app.maintenance_mode = 'true' หมายถึงเปิดโหมดบำรุงรักษา

CREATE POLICY orders_maintenance_lock ON orders
    AS RESTRICTIVE
    FOR INSERT
    TO shop_staff, shop_owner, app_user
    WITH CHECK (
        COALESCE(current_setting('app.maintenance_mode', true), 'false') <> 'true'
    );

CREATE POLICY orders_maintenance_lock_update ON orders
    AS RESTRICTIVE
    FOR UPDATE
    TO shop_staff, shop_owner, app_user
    USING (true)
    WITH CHECK (
        COALESCE(current_setting('app.maintenance_mode', true), 'false') <> 'true'
    );

CREATE POLICY orders_maintenance_lock_delete ON orders
    AS RESTRICTIVE
    FOR DELETE
    TO shop_staff, shop_owner, app_user
    USING (
        COALESCE(current_setting('app.maintenance_mode', true), 'false') <> 'true'
    );
```

เนื่องจากไม่ได้กำหนด policy เหล่านี้ให้กับ `platform_admin` (ซึ่งมี `BYPASSRLS` อยู่แล้ว) role นี้จะไม่ถูกจำกัดโดย RESTRICTIVE policy เหล่านี้เลยไม่ว่ากรณีใด เพราะ `BYPASSRLS` ทำให้ RLS ทั้งหมดไม่มีผลตั้งแต่ต้น ส่วน role อื่นเมื่อตั้งค่า `app.maintenance_mode = 'true'` จะถูกบล็อกการเขียนข้อมูลทันทีในทุก tenant พร้อมกัน โดยยังคง SELECT (อ่าน) ได้ตามปกติ เพราะ policy ที่สร้างครอบคลุมเฉพาะ INSERT/UPDATE/DELETE เท่านั้น
</details>

### แบบฝึกหัดที่ 10
เขียนชุดคำสั่งทดสอบครบวงจรเพื่อพิสูจน์ว่า role `shop_staff` ที่อยู่ใน context ของ tenant 3 (ร้านเสื้อผ้าแฟชั่นดี) **ไม่สามารถ** มองเห็น แก้ไข หรือ ลบ ข้อมูลคำสั่งซื้อ (orders) ของ tenant 1 (ร้านกาแฟดารา) ได้เลย ไม่ว่าจะพยายามด้วยวิธีใด (SELECT ตรง ๆ, UPDATE, DELETE, และ JOIN)

<details>
<summary>เฉลย</summary>

```sql
SET SESSION AUTHORIZATION shop_staff;
SET app.current_tenant_id = '3';

-- (1) พยายาม SELECT ตรง ๆ ด้วยการระบุ order_id ของ tenant 1
SELECT * FROM orders WHERE order_id = 2;   -- order_id 2 เป็นของ tenant 1
-- คาดหวัง: 0 rows

-- (2) พยายามระบุ tenant_id ตรง ๆ ใน WHERE
SELECT * FROM orders WHERE tenant_id = 1;
-- คาดหวัง: 0 rows

-- (3) พยายาม UPDATE
UPDATE orders SET total_amount = 0 WHERE order_id = 2;
-- คาดหวัง: UPDATE 0

-- (4) พยายาม DELETE (แม้ shop_staff จะไม่มี policy DELETE เลยอยู่แล้ว)
DELETE FROM orders WHERE order_id = 2;
-- คาดหวัง: DELETE 0

-- (5) พยายาม JOIN เพื่อดึงข้อมูลผ่านทางอ้อม
SELECT c.first_name, o.total_amount
FROM customers c
JOIN orders o ON o.customer_id = c.customer_id
WHERE c.tenant_id = 1;
-- คาดหวัง: 0 rows (customers ของ tenant 1 ก็มองไม่เห็นเช่นกัน)

-- (6) ยืนยันว่า tenant 3 เองยังทำงานได้ปกติ (เป็นการพิสูจน์ว่า RLS ไม่ได้ปิดกั้นทุกอย่าง)
SELECT COUNT(*) FROM orders;
-- คาดหวัง: 2 (จำนวน orders ของ tenant 3 เท่านั้น)

RESET SESSION AUTHORIZATION;
```

ผลลัพธ์ทั้งหมดที่คาดหวังคือ **0 แถวหรือ 0 การเปลี่ยนแปลง** สำหรับทุกความพยายามเข้าถึงข้อมูลของ tenant 1 ในขณะที่ query ของ tenant ตัวเอง (ข้อ 6) ยังคงทำงานได้ตามปกติ — พิสูจน์ว่า tenant isolation ผ่าน RLS ทำงานถูกต้องสมบูรณ์ ไม่ว่าจะพยายามเข้าถึงด้วยวิธีใดก็ตาม (SELECT ตรง, WHERE เจาะจง, UPDATE, DELETE, หรือ JOIN ทางอ้อม)
</details>

---

บทถัดไป: [Part 070 — SSL/TLS Encryption และการเข้ารหัสการเชื่อมต่อฐานข้อมูล](./part-070-ssl-encryption.md)
