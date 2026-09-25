# Part 080: โปรเจกต์ปิดท้ายระดับมืออาชีพ — สร้างสถาปัตยกรรม Production-Grade Multi-tenant SaaS

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับมืออาชีพ | Part 080 (โปรเจกต์ปิดท้ายระดับมืออาชีพ)

---

## เป้าหมายการเรียนรู้

เมื่อจบบทนี้ ผู้เรียนจะสามารถ:

1. **สังเคราะห์** ความรู้ทั้งหมดจากระดับมืออาชีพ (Part 061–079) ให้กลายเป็นสถาปัตยกรรมระบบเดียวที่ทำงานร่วมกันได้จริง แทนที่จะเป็นความรู้แยกส่วนทีละหัวข้อ
2. **ออกแบบ** สคีมา multi-tenant SaaS ระดับ production ที่มี Row Level Security (RLS) ครอบคลุมทุกตาราง พร้อมพิสูจน์ด้วย SQL ที่รันได้จริง
3. **ออกแบบระบบ Role/Privilege** แบบแบ่งชั้นสำหรับทีมงานหลายบทบาท (application, read-replica analyst, DBA/admin, CI/CD) โดยยึดหลัก least privilege
4. **วางแผนกลยุทธ์ Backup/PITR** ที่สอดคล้องกับเป้าหมาย RPO/RTO ระดับ SaaS สัญญา SLA กับลูกค้า
5. **ออกแบบสถาปัตยกรรม High Availability** แบบเต็มรูปแบบ (Primary + Synchronous Standby + Asynchronous Standby + Patroni + etcd) พร้อมอธิบาย failover flow
6. **ออกแบบ Connection Pooling และ Load Balancing layer** ที่รองรับ tenant จำนวนมากโดยไม่ทำให้ PostgreSQL connection ล้น
7. **วางแผน Security แบบ defense-in-depth** ครอบคลุม TLS, encryption at rest, audit logging, และการปฏิบัติตามมาตรฐาน (compliance)
8. **ออกแบบ Monitoring/Alerting stack** พร้อมกำหนด SLI/SLO และ alert rule ที่สำคัญสำหรับ SaaS หลาย tenant
9. **วางแผน Capacity Planning และ Scaling** เพื่อรองรับการเติบโตของจำนวน tenant ตั้งแต่ 10 ไปจนถึง 100,000+ tenants
10. **จัดทำเอกสารสถาปัตยกรรมรวม (architecture document)** ที่ใช้เป็น blueprint จริงในการ implement ระบบ production และประเมินความพร้อมก่อนก้าวสู่ระดับ World-Class

> **หมายเหตุสำคัญเกี่ยวกับรูปแบบเนื้อหา:** บทนี้เป็น "โปรเจกต์ปิดท้าย" (capstone) ไม่ใช่บทสอนฟีเจอร์ใหม่ ดังนั้นเนื้อหาจะแบ่งเป็น 2 ลักษณะชัดเจน:
> - **ส่วนที่รันได้จริง (runnable):** สคีมา, RLS policy, roles/privileges — เป็น SQL ที่ทดสอบในเครื่องได้จริงตาม "เตรียมข้อมูล" ด้านล่าง
> - **ส่วนที่เป็นเอกสารออกแบบ (design document):** backup/PITR, HA, pooling, security, monitoring, capacity — เป็น config snippet และสถาปัตยกรรมอ้างอิง (reference architecture) ที่ต้องนำไป deploy บน infrastructure จริง (หลายเครื่อง, หลาย availability zone) จึงไม่สามารถรันในสภาพแวดล้อมเดียวของบทเรียนได้ แต่เขียนในรูปแบบที่นำไปใช้งานจริงได้ทันที

---

## เตรียมข้อมูล

เราจะสร้างระบบหลังบ้านของ **"ShopFlow"** — แพลตฟอร์ม e-commerce แบบ SaaS ที่ให้ร้านค้า (tenant) หลายพันร้านมาเช่าใช้ระบบขายของออนไลน์ร่วมกันบนฐานข้อมูลเดียวกัน (shared database, shared schema, row-level isolation) ซึ่งเป็นสถาปัตยกรรม multi-tenant ที่ประหยัดต้นทุนที่สุดและได้รับความนิยมสูงในธุรกิจ SaaS ขนาดกลาง

โครงสร้างนี้ต่อยอดจากรูปแบบที่วางไว้ใน **Part 069 (Row Level Security)** โดยขยายให้ครอบคลุมตารางธุรกิจจริงของระบบ e-commerce ทั้งหมด

```sql
-- ==========================================================
-- ShopFlow Multi-tenant SaaS — Core Schema
-- ==========================================================

DROP TABLE IF EXISTS order_items, orders, customers, products, tenant_users, tenants CASCADE;

-- ตารางแม่: บริษัท/ร้านค้าที่เช่าระบบ (1 แถว = 1 tenant)
CREATE TABLE tenants (
    tenant_id    SERIAL PRIMARY KEY,
    tenant_name  VARCHAR(100) NOT NULL,
    plan         VARCHAR(20)  NOT NULL DEFAULT 'free'
                 CHECK (plan IN ('free', 'starter', 'pro', 'enterprise')),
    is_active    BOOLEAN      NOT NULL DEFAULT true,
    created_at   TIMESTAMPTZ  DEFAULT now()
);

-- ผู้ใช้งานระบบของแต่ละ tenant (แอดมินร้าน, พนักงาน)
CREATE TABLE tenant_users (
    user_id        SERIAL PRIMARY KEY,
    tenant_id      INTEGER NOT NULL REFERENCES tenants(tenant_id),
    email          VARCHAR(150) NOT NULL,
    password_hash  TEXT NOT NULL,
    role           VARCHAR(20) NOT NULL DEFAULT 'member'
                   CHECK (role IN ('owner', 'admin', 'member')),
    created_at     TIMESTAMPTZ DEFAULT now(),
    UNIQUE (tenant_id, email)
);

-- สินค้าของแต่ละร้าน
CREATE TABLE products (
    product_id    SERIAL PRIMARY KEY,
    tenant_id     INTEGER NOT NULL REFERENCES tenants(tenant_id),
    product_name  VARCHAR(150) NOT NULL,
    unit_price    NUMERIC(10,2) NOT NULL CHECK (unit_price >= 0),
    stock_qty     INTEGER NOT NULL DEFAULT 0,
    created_at    TIMESTAMPTZ DEFAULT now()
);

-- ลูกค้าปลายทางของแต่ละร้าน (ผู้ซื้อ ไม่ใช่ผู้ดูแลระบบ)
CREATE TABLE customers (
    customer_id  SERIAL PRIMARY KEY,
    tenant_id    INTEGER NOT NULL REFERENCES tenants(tenant_id),
    first_name   VARCHAR(60),
    email        VARCHAR(150),
    created_at   TIMESTAMPTZ DEFAULT now()
);

-- คำสั่งซื้อ
CREATE TABLE orders (
    order_id      SERIAL PRIMARY KEY,
    tenant_id     INTEGER NOT NULL REFERENCES tenants(tenant_id),
    customer_id   INTEGER REFERENCES customers(customer_id),
    order_date    TIMESTAMPTZ DEFAULT now(),
    status        VARCHAR(20) NOT NULL DEFAULT 'pending'
                  CHECK (status IN ('pending','paid','shipped','cancelled')),
    total_amount  NUMERIC(12,2)
);

-- รายการสินค้าในคำสั่งซื้อ (ตารางลูกของ orders — ขยายจากโจทย์ตั้งต้นเพื่อความสมจริง)
CREATE TABLE order_items (
    order_item_id  SERIAL PRIMARY KEY,
    tenant_id      INTEGER NOT NULL REFERENCES tenants(tenant_id),
    order_id       INTEGER NOT NULL REFERENCES orders(order_id),
    product_id     INTEGER NOT NULL REFERENCES products(product_id),
    quantity       INTEGER NOT NULL CHECK (quantity > 0),
    unit_price     NUMERIC(10,2) NOT NULL
);

-- ดัชนีที่จำเป็นสำหรับ multi-tenant workload: tenant_id ต้องนำหน้าแทบทุก index
CREATE INDEX idx_tenant_users_tenant   ON tenant_users (tenant_id);
CREATE INDEX idx_products_tenant       ON products (tenant_id);
CREATE INDEX idx_customers_tenant      ON customers (tenant_id);
CREATE INDEX idx_orders_tenant_date    ON orders (tenant_id, order_date DESC);
CREATE INDEX idx_order_items_tenant    ON order_items (tenant_id, order_id);
```

จากนั้นใส่ข้อมูลตัวอย่างให้มีหลาย tenant เพื่อพิสูจน์การแยกข้อมูล (isolation):

```sql
-- Tenant ตัวอย่าง 3 ร้าน แต่ละร้านอยู่คนละแผน (plan)
INSERT INTO tenants (tenant_name, plan) VALUES
    ('BangkokGadget', 'pro'),        -- tenant_id = 1
    ('ChiangMaiCraft', 'starter'),   -- tenant_id = 2
    ('PhuketSurfShop', 'enterprise');-- tenant_id = 3

INSERT INTO tenant_users (tenant_id, email, password_hash, role) VALUES
    (1, 'owner@bangkokgadget.com',  'hash1', 'owner'),
    (1, 'staff@bangkokgadget.com',  'hash2', 'member'),
    (2, 'owner@chiangmaicraft.com', 'hash3', 'owner'),
    (3, 'owner@phuketsurf.com',     'hash4', 'owner'),
    (3, 'admin@phuketsurf.com',     'hash5', 'admin');

INSERT INTO products (tenant_id, product_name, unit_price, stock_qty) VALUES
    (1, 'หูฟังไร้สาย รุ่น X1', 1290.00, 50),
    (1, 'พาวเวอร์แบงค์ 20000mAh', 590.00, 120),
    (2, 'กระเป๋าสานผักตบชวา', 450.00, 30),
    (2, 'ผ้าทอมือเชียงใหม่', 890.00, 15),
    (3, 'บอร์ดโต้คลื่น 6 ฟุต', 8900.00, 8),
    (3, 'ชุดว่ายน้ำกันแดด', 1200.00, 40);

INSERT INTO customers (tenant_id, first_name, email) VALUES
    (1, 'สมชาย', 'somchai@example.com'),
    (1, 'สมหญิง', 'somying@example.com'),
    (2, 'มานะ', 'mana@example.com'),
    (3, 'อารีย์', 'aree@example.com');

INSERT INTO orders (tenant_id, customer_id, status, total_amount) VALUES
    (1, 1, 'paid', 1880.00),
    (1, 2, 'pending', 590.00),
    (2, 3, 'shipped', 450.00),
    (3, 4, 'paid', 10100.00);

INSERT INTO order_items (tenant_id, order_id, product_id, quantity, unit_price) VALUES
    (1, 1, 1, 1, 1290.00),
    (1, 1, 2, 1, 590.00),
    (1, 2, 2, 1, 590.00),
    (2, 3, 3, 1, 450.00),
    (3, 4, 5, 1, 8900.00),
    (3, 4, 6, 1, 1200.00);
```

ตรวจสอบข้อมูลก่อนเริ่ม (ยังไม่เปิด RLS จึงเห็นทุก tenant):

```sql
SELECT t.tenant_name, count(o.order_id) AS order_count, sum(o.total_amount) AS revenue
FROM tenants t
LEFT JOIN orders o ON o.tenant_id = t.tenant_id
GROUP BY t.tenant_name
ORDER BY revenue DESC NULLS LAST;
```

ข้อมูลชุดนี้จะถูกใช้ตลอดทั้งบท ทั้งในการสาธิต RLS (Step 792), roles (Step 793) และใช้เป็นบริบทอ้างอิงเวลาพูดถึง backup, HA, pooling, security, monitoring และ capacity planning (Step 794–799)

---

## Step 791: ภาพรวมโปรเจกต์ — จาก 19 หัวข้อ สู่ 1 สถาปัตยกรรม

### 791.1 โจทย์ธุรกิจ: ShopFlow ต้องเป็นอย่างไร

ShopFlow คือ SaaS ที่ต้องรองรับลักษณะงานดังนี้:

| ความต้องการ | รายละเอียด |
|---|---|
| จำนวน tenant | เริ่มที่ ~500 ร้าน เป้าหมายปีถัดไป 20,000+ ร้าน |
| Data isolation | ร้านหนึ่งต้อง**ไม่มีทางเห็น**ข้อมูลอีกร้านหนึ่งได้เลย แม้เกิด bug ใน application layer |
| Availability SLA | 99.95% uptime ต่อเดือน (สัญญากับลูกค้า plan `pro`/`enterprise`) → downtime ยอมได้ไม่เกิน ~21.6 นาที/เดือน |
| RPO (Recovery Point Objective) | สูญเสียข้อมูลได้ไม่เกิน 5 นาที กรณีเกิดภัยพิบัติ |
| RTO (Recovery Time Objective) | ระบบต้องกลับมาใช้งานได้ภายใน 15 นาที กรณี primary database ล่ม |
| Compliance | ต้องเก็บ audit log การเข้าถึงข้อมูลลูกค้า (PDPA / GDPR-like) อย่างน้อย 1 ปี |
| Connection scale | รองรับ backend service instance หลักร้อยตัว โดยแต่ละตัวเปิด connection pool ของตัวเอง |
| Read scaling | รายงาน/แดชบอร์ดสำหรับทีม data analyst ต้องไม่กระทบ workload OLTP หลัก |
| Growth | ต้องวางแผนรองรับการเติบโต 40 เท่าโดยไม่ redesign schema ใหม่ทั้งหมด |

โจทย์นี้ทำให้ ShopFlow ไม่สามารถแก้ปัญหาด้วยฟีเจอร์เดียวได้ ต้องอาศัย**การผสาน 19 หัวข้อจาก Part 061–079** เข้าด้วยกันเป็นสถาปัตยกรรมเดียว ซึ่งคือหัวใจของบทนี้

### 791.2 ตารางสรุป Mapping: หัวข้อ Professional Level → บทบาทในสถาปัตยกรรม ShopFlow

| Part | หัวข้อที่เรียนมา | บทบาทใน ShopFlow | ใช้ในบทนี้ที่ Step |
|---|---|---|---|
| 061 | Backup Strategies (pg_dump, pg_basebackup) | Full backup รายวันของทุก tenant | 794 |
| 062 | Point-in-Time Recovery (WAL archiving) | กู้ข้อมูลย้อนเวลาเมื่อมี incident เช่น bug ลบข้อมูลลูกค้าผิด tenant | 794 |
| 063 | Streaming Replication | Standby แบบ real-time สำหรับ HA และ read scaling | 795, 796 |
| 064 | Logical Replication | ส่งข้อมูลเฉพาะ tenant ไป data warehouse/analytics โดยไม่ปนกับ OLTP | 799 |
| 065 | High Availability (failover) | Patroni + auto failover เพื่อรักษา SLA 99.95% | 795 |
| 066 | Connection Pooling (PgBouncer) | รองรับ backend หลักร้อย instance โดยไม่ทำให้ PostgreSQL connection ล้น | 796 |
| 067 | Load Balancing | กระจาย read query ไปยัง read replica อย่างปลอดภัย | 796 |
| 068 | Roles & Privileges | แบ่งสิทธิ์ app role / analyst role / admin role ตาม least privilege | 793 |
| 069 | Row Level Security | รากฐานการแยกข้อมูล tenant บนฐานข้อมูลเดียว | 792 |
| 070 | SSL/TLS & Encryption | เข้ารหัสการเชื่อมต่อทุกช่องทาง | 797 |
| 071 | Encryption at Rest & Audit Logging | ปกป้องข้อมูลในดิสก์ + บันทึกการเข้าถึงเพื่อ compliance | 797 |
| 072 | Monitoring & Observability | pg_stat_statements, Prometheus/Grafana, alerting | 798 |
| 073 | Capacity Planning | คาดการณ์การใช้ CPU/RAM/Storage ตามจำนวน tenant | 799 |
| 074 | Partitioning at Scale | แบ่งพาร์ทิชันตารางใหญ่ (orders, order_items) ตามเวลา | 799 |
| 075 | Scaling Strategies (Sharding) | แผนย้ายจาก single-database ไป multi-shard เมื่อ tenant เกิน capacity | 799 |
| 076 | Query Performance Tuning ขั้นสูง | Optimize query ที่ RLS ทำให้ planner เลือก plan ไม่ดี | 792, 799 |
| 077 | Vacuum & Autovacuum Tuning | ป้องกัน bloat บนตารางที่ทุก tenant เขียนพร้อมกัน (hot table) | 799 |
| 078 | Extensions & ฟีเจอร์ขั้นสูง | pg_cron (job เก็บกวาดข้อมูล), pg_partman (partition อัตโนมัติ) | 799 |
| 079 | Disaster Recovery Planning | DR drill, runbook, การทดสอบ failover จริง | 794, 795, 800 |

> ตารางนี้คือ**หัวใจของ Step 791** — มันบอกว่าแต่ละ Part ที่เราเรียนแยกกันมา 20 บท ไม่ใช่ความรู้ที่ใช้แยกกัน แต่ประกอบกันเป็น "ชั้น" (layer) ของสถาปัตยกรรมเดียวกัน เหมือนที่วิศวกรฐานข้อมูลระดับ Staff/Principal ต้องมองเห็นภาพรวมทั้งระบบ ไม่ใช่แค่ feature เดียว

### 791.3 สถาปัตยกรรมระดับสูง (High-level view) ก่อนลงรายละเอียด

```
                         ┌─────────────────────────────┐
                         │   Client (Web / Mobile App)  │
                         └───────────────┬───────────────┘
                                         │ HTTPS
                         ┌───────────────▼───────────────┐
                         │   API Gateway / App Servers    │  (stateless, N instances)
                         │   - เซ็ต app.current_tenant     │
                         │   - authenticate → get tenant_id│
                         └───────────────┬───────────────┘
                                         │ SSL/TLS
                         ┌───────────────▼───────────────┐
                         │   PgBouncer (Pooling Layer)     │  ← Step 796
                         │   transaction pooling           │
                         └──────┬─────────────────┬────────┘
                                │ write               │ read
                    ┌───────────▼─────────┐   ┌──────▼───────────┐
                    │   Primary (RW)        │   │  Read Replicas    │ ← Step 795/796
                    │   RLS enforced         │──▶│  (async, N นอด)   │
                    │   Patroni managed      │   └───────────────────┘
                    └─────┬──────────┬──────┘
                          │ sync      │ async
                ┌──────────▼───┐  ┌───▼─────────┐
                │ Sync Standby   │  │Async Standby │        ← Step 795
                │ (same AZ/DC)   │  │(DR site)      │
                └────────────────┘  └───────────────┘
```

รายละเอียดของแต่ละชั้นจะขยายความในแต่ละ Step ถัดไป

---

## Step 792: ออกแบบ Multi-tenant Schema แบบเต็มรูปแบบพร้อม Row Level Security ทุกตาราง

### 792.1 ทบทวนหลักการจาก Part 069 และขยายให้ครอบคลุมทุกตาราง

ใน Part 069 เราเรียนรู้การใช้ `CREATE POLICY` ร่วมกับ session variable (`current_setting`) เพื่อบังคับให้ query แต่ละครั้งเห็นเฉพาะข้อมูลของ tenant ตนเอง หลักการเดิมยังใช้ได้ แต่ระดับ production ต้องทำ 3 อย่างเพิ่มเติมที่ Part 069 (บทเดี่ยว) มักไม่มีพื้นที่พอจะลงลึก:

1. เปิด RLS **ทุกตาราง** ที่มี `tenant_id` ไม่ใช่แค่ตารางเดียว รวมถึงตารางลูก (`order_items`) ที่ผูกกับตารางแม่ผ่าน foreign key
2. ใช้ `FORCE ROW LEVEL SECURITY` เพื่อบังคับใช้ policy แม้กับ**เจ้าของตาราง** (table owner) — ค่า default ของ PostgreSQL คือ table owner จะ bypass RLS เสมอ ซึ่งอันตรายมากถ้า application role ดันเป็น owner ของตาราง
3. แยก policy สำหรับแต่ละคำสั่ง (`SELECT`, `INSERT`, `UPDATE`, `DELETE`) เพื่อป้องกันการ "เขียนข้าม tenant" ไม่ใช่แค่ "อ่านข้าม tenant"

### 792.2 ตั้งค่า session variable ที่เป็นรากฐานของ RLS ทั้งระบบ

```sql
-- Application จะ SET ค่านี้ทุกครั้งที่เปิด connection/transaction ใหม่
-- โดยได้ tenant_id มาจากการ authenticate ผู้ใช้ (JWT claim, session, ฯลฯ)
-- ใช้ SET LOCAL เพื่อให้ผูกกับ transaction เดียว ไม่หลุดข้าม transaction อื่นใน connection pool

-- ตัวอย่างการเซ็ตค่า (ฝั่ง application จะ execute ก่อนทุก query):
-- SET LOCAL app.current_tenant = '1';

-- ฟังก์ชัน helper สำหรับดึงค่า tenant ปัจจุบันแบบปลอดภัย (คืนค่า NULL ถ้ายังไม่ได้ตั้งค่า แทนที่จะ error)
CREATE OR REPLACE FUNCTION current_tenant_id() RETURNS INTEGER AS $$
    SELECT NULLIF(current_setting('app.current_tenant', true), '')::INTEGER;
$$ LANGUAGE sql STABLE;
```

### 792.3 เปิด RLS และสร้าง Policy ครบทุกตาราง

```sql
-- ---------------------------------------------------------
-- 1) tenants — ตารางนี้พิเศษ: เก็บ metadata ของทุก tenant
--    ผู้ใช้ทั่วไปเห็นได้เฉพาะแถวของตัวเอง (ใช้ตอน tenant settings page)
-- ---------------------------------------------------------
ALTER TABLE tenants ENABLE ROW LEVEL SECURITY;
ALTER TABLE tenants FORCE ROW LEVEL SECURITY;

CREATE POLICY tenants_isolation ON tenants
    FOR ALL
    USING (tenant_id = current_tenant_id())
    WITH CHECK (tenant_id = current_tenant_id());

-- ---------------------------------------------------------
-- 2) tenant_users
-- ---------------------------------------------------------
ALTER TABLE tenant_users ENABLE ROW LEVEL SECURITY;
ALTER TABLE tenant_users FORCE ROW LEVEL SECURITY;

CREATE POLICY tenant_users_isolation ON tenant_users
    FOR ALL
    USING (tenant_id = current_tenant_id())
    WITH CHECK (tenant_id = current_tenant_id());

-- ---------------------------------------------------------
-- 3) products
-- ---------------------------------------------------------
ALTER TABLE products ENABLE ROW LEVEL SECURITY;
ALTER TABLE products FORCE ROW LEVEL SECURITY;

CREATE POLICY products_isolation ON products
    FOR ALL
    USING (tenant_id = current_tenant_id())
    WITH CHECK (tenant_id = current_tenant_id());

-- ---------------------------------------------------------
-- 4) customers
-- ---------------------------------------------------------
ALTER TABLE customers ENABLE ROW LEVEL SECURITY;
ALTER TABLE customers FORCE ROW LEVEL SECURITY;

CREATE POLICY customers_isolation ON customers
    FOR ALL
    USING (tenant_id = current_tenant_id())
    WITH CHECK (tenant_id = current_tenant_id());

-- ---------------------------------------------------------
-- 5) orders
-- ---------------------------------------------------------
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;
ALTER TABLE orders FORCE ROW LEVEL SECURITY;

CREATE POLICY orders_isolation ON orders
    FOR ALL
    USING (tenant_id = current_tenant_id())
    WITH CHECK (tenant_id = current_tenant_id());

-- ---------------------------------------------------------
-- 6) order_items — ตารางลูก ต้องเปิด RLS ด้วยตัวเอง
--    (RLS ของ orders ไม่ "ไหล" ไปยัง order_items โดยอัตโนมัติ)
-- ---------------------------------------------------------
ALTER TABLE order_items ENABLE ROW LEVEL SECURITY;
ALTER TABLE order_items FORCE ROW LEVEL SECURITY;

CREATE POLICY order_items_isolation ON order_items
    FOR ALL
    USING (tenant_id = current_tenant_id())
    WITH CHECK (tenant_id = current_tenant_id());
```

> **ข้อควรระวังสำคัญ:** เพราะเราใส่ `tenant_id` ลงใน `order_items` โดยตรง (denormalize เล็กน้อย) แทนที่จะ join ผ่าน `orders` ทุกครั้ง — นี่คือ trade-off ที่ตั้งใจ: การ denormalize `tenant_id` ทำให้ RLS policy ของตารางลูกทำงานเร็ว (ใช้ index `idx_order_items_tenant` ตรงๆ) แทนที่จะต้อง subquery join กับ `orders` ทุกครั้งซึ่งทำให้ planner วางแผนได้แย่ลงมากเมื่อข้อมูลโต

### 792.4 ทดสอบ Isolation จริง — พิสูจน์ว่า RLS ทำงาน

```sql
-- จำลอง session ของพนักงานร้าน BangkokGadget (tenant_id = 1)
BEGIN;
SET LOCAL app.current_tenant = '1';

SELECT tenant_id, product_name, unit_price FROM products;
-- ผลลัพธ์: เห็นเฉพาะ 2 แถวของ tenant_id = 1 เท่านั้น แม้ตารางจริงมี 6 แถว

SELECT tenant_id, order_id, total_amount FROM orders;
-- เห็นเฉพาะ order ของ tenant 1

-- พยายามแอบดู order ของ tenant อื่นตรงๆ ด้วย id ที่รู้ (order_id = 4 เป็นของ tenant 3)
SELECT * FROM orders WHERE order_id = 4;
-- ผลลัพธ์: 0 แถว — ถึงจะรู้ primary key ก็มองไม่เห็น เพราะ RLS กรองที่ระดับ storage/planner

COMMIT;
```

```sql
-- ทดสอบว่า "เขียนข้าม tenant" ก็ถูกบล็อกเช่นกัน (WITH CHECK)
BEGIN;
SET LOCAL app.current_tenant = '1';

-- พยายามสร้างสินค้าแล้วสวมรอยใส่ tenant_id ของร้านอื่น
INSERT INTO products (tenant_id, product_name, unit_price)
VALUES (2, 'สินค้าปลอมแฝงเข้า tenant อื่น', 1.00);
-- ผลลัพธ์: ERROR: new row violates row-level security policy for table "products"
-- เพราะ WITH CHECK (tenant_id = current_tenant_id()) ตรวจตอน INSERT ด้วย ไม่ใช่แค่ SELECT

ROLLBACK;
```

```sql
-- ทดสอบว่าไม่ตั้งค่า tenant เลย (ลืม SET LOCAL) จะไม่เห็นอะไรเลย ไม่ใช่เห็นทุก tenant
BEGIN;
-- ไม่ SET app.current_tenant

SELECT count(*) FROM orders;
-- current_tenant_id() คืนค่า NULL → tenant_id = NULL เป็นเท็จเสมอ → เห็น 0 แถว
-- นี่คือพฤติกรรม "fail closed" ที่ถูกต้องสำหรับระบบ SaaS: ลืมตั้งค่า = ไม่เห็นอะไรเลย
-- ดีกว่า "fail open" ที่จะเห็นข้อมูลทุก tenant โดยไม่ได้ตั้งใจ

ROLLBACK;
```

### 792.5 Performance ของ RLS ในตาราง orders — ทำไมต้องดู EXPLAIN

```sql
BEGIN;
SET LOCAL app.current_tenant = '1';

EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders WHERE status = 'paid';

-- Plan ที่คาดหวัง (ตัวอย่าง):
-- Index Scan using idx_orders_tenant_date on orders
--   Index Cond: (tenant_id = ($1)::integer)     ← มาจาก RLS policy โดยอัตโนมัติ
--   Filter: (status = 'paid'::character varying)
-- ==> RLS ถูก "push down" เข้าไปรวมกับเงื่อนไขของ query จริง
--     และเลือกใช้ index ที่ tenant_id นำหน้าได้พอดี (idx_orders_tenant_date)

ROLLBACK;
```

**บทเรียนสำคัญ:** RLS จะ**ไม่**ทำให้ query ช้าลงอย่างมีนัยสำคัญ **ถ้าหากทุก index ที่เกี่ยวข้องมี `tenant_id` เป็นคอลัมน์แรก** เพราะ planner จะรวมเงื่อนไข RLS เข้ากับ `WHERE` clause ของผู้ใช้แล้วเลือก index scan ตามปกติ แต่ถ้า index ไม่มี `tenant_id` นำหน้า planner อาจเลือก sequential scan ทั้งตารางแล้วกรองทีหลัง — นี่คือจุดเชื่อมกับ **Part 076 (Query Performance Tuning)** ที่จะกล่าวถึงอีกครั้งใน Step 799

### 792.6 Default สำหรับ tenant_id — กันการลืมใส่ค่าตอน INSERT

```sql
-- ตั้งค่า default ให้ tenant_id ดึงจาก session อัตโนมัติ ลดโอกาส human error ที่ app ลืมส่งค่า
ALTER TABLE products    ALTER COLUMN tenant_id SET DEFAULT current_tenant_id();
ALTER TABLE customers   ALTER COLUMN tenant_id SET DEFAULT current_tenant_id();
ALTER TABLE orders      ALTER COLUMN tenant_id SET DEFAULT current_tenant_id();
ALTER TABLE order_items ALTER COLUMN tenant_id SET DEFAULT current_tenant_id();

-- ทดสอบ: ไม่ต้องระบุ tenant_id เอง ระบบดึงจาก session ให้อัตโนมัติ
BEGIN;
SET LOCAL app.current_tenant = '2';
INSERT INTO products (product_name, unit_price) VALUES ('สินค้าใหม่ของร้านเชียงใหม่', 300.00)
RETURNING tenant_id, product_name;
-- tenant_id ที่ได้ = 2 โดยไม่ต้องเขียนเอง
ROLLBACK;
```

### 792.7 สรุป Step 792

| หลักการ | เหตุผล |
|---|---|
| RLS ทุกตารางที่มี tenant_id (รวมตารางลูก) | RLS ไม่ไหลผ่าน JOIN โดยอัตโนมัติ ต้องเปิดทีละตาราง |
| `FORCE ROW LEVEL SECURITY` | ป้องกัน table owner bypass policy โดยไม่ตั้งใจ |
| แยก `USING` และ `WITH CHECK` | ป้องกันทั้งการอ่านและการเขียนข้าม tenant |
| Index ที่มี `tenant_id` นำหน้า | ทำให้ RLS ไม่ทำลาย performance |
| Default = `current_tenant_id()` | ลด human error ระดับ application code |
| Fail closed เมื่อไม่ตั้งค่า session | ปลอดภัยกว่าการ fail open |

---

## Step 793: ออกแบบระบบ Role และ Privilege สำหรับทีมต่างๆ

### 793.1 ทบทวนหลักการจาก Part 068 และขยายสู่บริบท SaaS

Part 068 สอนหลักการ `CREATE ROLE`, `GRANT`, `REVOKE` และ least privilege ในระดับทั่วไป สำหรับ ShopFlow เราต้องออกแบบ role ให้ตรงกับ**ทีมงานจริง**ที่เข้าถึงฐานข้อมูล ได้แก่:

| Role | ใครใช้ | สิทธิ์ที่ต้องมี |
|---|---|---|
| `shopflow_app` | Backend application (ผ่าน PgBouncer) | CRUD บนตารางธุรกิจ, ต้องอยู่ภายใต้ RLS เสมอ |
| `shopflow_migrator` | CI/CD pipeline (schema migration) | DDL (CREATE/ALTER TABLE), แยกจาก app role |
| `shopflow_analyst` | ทีม Data/BI ที่ query จาก read replica | SELECT อย่างเดียว, ยังต้องผ่าน RLS หรือดูได้ทุก tenant แบบมี audit |
| `shopflow_admin` | DBA/Platform engineer | สิทธิ์เต็มสำหรับ operation งาน (backup, vacuum, monitoring) แต่ยังไม่ใช่ superuser |
| `shopflow_readonly_support` | ทีม Customer Support (ดูข้อมูลช่วยลูกค้า) | SELECT เฉพาะ tenant ที่กำลัง troubleshoot พร้อม audit log |

### 793.2 สร้าง Role ทั้งหมดพร้อมหลักการ Least Privilege

```sql
-- ---------------------------------------------------------
-- 1) Application role — role หลักที่ backend service ใช้เชื่อมต่อ
--    - NOLOGIN โดยตรงจากมนุษย์ (ใช้ผ่าน connection string ของ service)
--    - ไม่ใช่ superuser, ไม่ใช่ owner ของตาราง (จึงถูก RLS บังคับเสมอ)
-- ---------------------------------------------------------
CREATE ROLE shopflow_app WITH LOGIN PASSWORD 'use_a_secret_manager_not_this_string';
ALTER ROLE shopflow_app CONNECTION LIMIT 200;   -- จำกัด connection ต่อ role กันชนความจุ

GRANT CONNECT ON DATABASE shopflow TO shopflow_app;
GRANT USAGE ON SCHEMA public TO shopflow_app;
GRANT SELECT, INSERT, UPDATE, DELETE
    ON tenants, tenant_users, products, customers, orders, order_items
    TO shopflow_app;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO shopflow_app;
-- ห้าม GRANT สิทธิ์ DDL (CREATE/ALTER/DROP) ให้ role นี้โดยเด็ดขาด

-- ---------------------------------------------------------
-- 2) Migrator role — ใช้เฉพาะตอน deploy schema change ผ่าน CI/CD
--    แยกจาก app role เพื่อไม่ให้ compromised application เปลี่ยนโครงสร้างตารางได้
-- ---------------------------------------------------------
CREATE ROLE shopflow_migrator WITH LOGIN PASSWORD 'use_a_secret_manager_not_this_string';
GRANT CONNECT ON DATABASE shopflow TO shopflow_migrator;
GRANT CREATE, USAGE ON SCHEMA public TO shopflow_migrator;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO shopflow_migrator;
GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA public TO shopflow_migrator;
-- ตั้ง default privilege เพื่อให้ตารางใหม่ที่ migrator สร้าง ให้สิทธิ์ shopflow_app อัตโนมัติ
ALTER DEFAULT PRIVILEGES FOR ROLE shopflow_migrator IN SCHEMA public
    GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO shopflow_app;

-- ---------------------------------------------------------
-- 3) Analyst role — ใช้บน READ REPLICA เท่านั้น (Part 067 load balancing routing)
--    SELECT-only, ยังคงอยู่ภายใต้ RLS (เห็นทีละ tenant เหมือน app)
--    เว้นแต่งาน "cross-tenant aggregate report" ที่ได้รับอนุมัติเป็นพิเศษ (ดู bypass role ด้านล่าง)
-- ---------------------------------------------------------
CREATE ROLE shopflow_analyst WITH LOGIN PASSWORD 'use_a_secret_manager_not_this_string';
ALTER ROLE shopflow_analyst CONNECTION LIMIT 20;
GRANT CONNECT ON DATABASE shopflow TO shopflow_analyst;
GRANT USAGE ON SCHEMA public TO shopflow_analyst;
GRANT SELECT ON tenants, tenant_users, products, customers, orders, order_items
    TO shopflow_analyst;
-- analyst ไม่มี INSERT/UPDATE/DELETE เลย — ตรวจสอบด้วย:
--   SELECT grantee, table_name, privilege_type
--   FROM information_schema.role_table_grants WHERE grantee = 'shopflow_analyst';

-- ---------------------------------------------------------
-- 4) Cross-tenant reporting role — สำหรับรายงานผู้บริหาร (เช่น "รายได้รวมทุก tenant")
--    role นี้ "bypass" RLS ได้โดยตั้งใจ แต่ต้องมี audit ทุกครั้งที่ใช้ (ดู Step 797)
-- ---------------------------------------------------------
CREATE ROLE shopflow_reporting WITH LOGIN PASSWORD 'use_a_secret_manager_not_this_string';
GRANT CONNECT ON DATABASE shopflow TO shopflow_reporting;
GRANT USAGE ON SCHEMA public TO shopflow_reporting;
GRANT SELECT ON tenants, orders, order_items, products TO shopflow_reporting;
ALTER TABLE tenants    FORCE ROW LEVEL SECURITY;  -- (ย้ำ) FORCE ใช้กับทุก role ยกเว้น BYPASSRLS
ALTER ROLE shopflow_reporting BYPASSRLS;          -- role นี้เท่านั้นที่เห็นข้าม tenant ได้

-- ---------------------------------------------------------
-- 5) Support (customer service) role — เห็นได้ทีละ tenant ผ่านการ SET LOCAL เหมือน app
--    ต่างจาก app ตรงที่ SELECT อย่างเดียว และห้ามแตะ password_hash
-- ---------------------------------------------------------
CREATE ROLE shopflow_support WITH LOGIN PASSWORD 'use_a_secret_manager_not_this_string';
GRANT CONNECT ON DATABASE shopflow TO shopflow_support;
GRANT USAGE ON SCHEMA public TO shopflow_support;
GRANT SELECT (user_id, tenant_id, email, role, created_at) ON tenant_users TO shopflow_support;
-- column-level privilege: เห็นได้ทุกคอลัมน์ ยกเว้น password_hash
GRANT SELECT ON tenants, products, customers, orders, order_items TO shopflow_support;

-- ---------------------------------------------------------
-- 6) Admin/DBA role — งาน operation (VACUUM, monitoring, backup) โดยไม่ใช่ superuser เต็มรูปแบบ
--    ใช้ predefined role ของ PostgreSQL (ตั้งแต่ v14+) แทนการให้ SUPERUSER ตรงๆ
-- ---------------------------------------------------------
CREATE ROLE shopflow_admin WITH LOGIN PASSWORD 'use_a_secret_manager_not_this_string';
GRANT pg_monitor TO shopflow_admin;                 -- ดู pg_stat_activity, pg_stat_statements ของทุก session
GRANT pg_read_all_stats TO shopflow_admin;
GRANT CONNECT ON DATABASE shopflow TO shopflow_admin;
ALTER ROLE shopflow_admin BYPASSRLS;                -- DBA ต้อง debug ข้าม tenant ได้ แต่ถูก audit (Step 797)
GRANT shopflow_admin TO CURRENT_USER;               -- ตัวอย่างมอบสิทธิ์ให้ผู้ดูแลระบบคนปัจจุบัน (ปรับตามจริง)
```

### 793.3 ตรวจสอบภาพรวมสิทธิ์ทั้งหมดที่ตั้งไว้

```sql
-- สรุป role ทั้งหมดในระบบ พร้อมคุณสมบัติสำคัญ
SELECT rolname, rolsuper, rolcanlogin, rolbypassrls, rolconnlimit
FROM pg_roles
WHERE rolname LIKE 'shopflow_%'
ORDER BY rolname;

-- ตรวจสอบว่า role ไหนมีสิทธิ์อะไรบนตารางไหนบ้าง (ใช้ตรวจ compliance / security review)
SELECT grantee, table_name,
       string_agg(privilege_type, ', ' ORDER BY privilege_type) AS privileges
FROM information_schema.role_table_grants
WHERE grantee LIKE 'shopflow_%'
GROUP BY grantee, table_name
ORDER BY grantee, table_name;
```

### 793.4 หลักการออกแบบที่ต้องยึดถือ (Least Privilege Checklist)

1. **แยก role ตามหน้าที่ ไม่ใช่ตามบุคคล** — บุคคลแต่ละคนใช้ personal login role แล้ว `GRANT` เข้ากลุ่ม (`shopflow_admin`, ฯลฯ) เพื่อ audit ได้ว่า "ใคร" ทำอะไร ไม่ใช่แค่ "role อะไร" ทำ
2. **BYPASSRLS ต้องมีจำนวนน้อยที่สุดเท่าที่จำเป็น** และทุกครั้งที่ใช้ role เหล่านี้ต้องถูก audit log (เชื่อมกับ Step 797)
3. **App role ห้ามเป็น table owner** — table owner ควรเป็น role แยกต่างหาก (เช่น `shopflow_owner`) ที่ไม่มีใคร login ตรงๆ เพื่อให้ `FORCE ROW LEVEL SECURITY` มีความหมายจริง
4. **Connection limit ต่อ role** ช่วยป้องกัน role เดียวใช้ connection จนหมดโควตาของทั้งระบบ (เชื่อมกับ pooling ที่ Step 796)
5. **Migration role แยกจาก app role เสมอ** — ป้องกันช่องโหว่แบบ SQL injection ที่หลุดมาจาก application ไม่ให้สามารถ `DROP TABLE` ได้

---

## Step 794: แผนกลยุทธ์ Backup และ PITR สำหรับ SaaS ที่ต้อง RPO/RTO เข้มงวด

> **หมายเหตุ:** ตั้งแต่ Step นี้เป็นต้นไป เนื้อหาจะเปลี่ยนเป็น**เอกสารออกแบบสถาปัตยกรรม (design document)** ที่มาพร้อม config ตัวอย่างจริง เนื่องจากหัวข้อเหล่านี้ (backup ระดับ production, HA แบบหลายโหนด, pooling แยกเครื่อง) ต้องใช้โครงสร้าง infrastructure หลายเครื่อง/หลาย availability zone ซึ่งไม่สามารถรันสาธิตในสภาพแวดล้อมเดียวของบทเรียนนี้ได้จริง — แต่ config ทุกชิ้นเขียนในรูปแบบที่นำไปใช้งานจริงได้ทันที

### 794.1 ทบทวนจาก Part 061–062 และแปลงเป็นนโยบายของ ShopFlow

จากโจทย์ Step 791: **RPO ≤ 5 นาที, RTO ≤ 15 นาที** สำหรับ plan `pro`/`enterprise` เราจึงออกแบบกลยุทธ์ backup เป็น 3 ชั้น:

| ชั้น | เครื่องมือ | ความถี่ | Retention | วัตถุประสงค์ |
|---|---|---|---|---|
| 1. Base backup | `pg_basebackup` (จาก standby ไม่ใช่ primary เพื่อลดโหลด) | ทุกวัน 02:00 น. | 14 วัน | จุดตั้งต้นสำหรับ PITR |
| 2. WAL archiving | `archive_command` ส่งไป object storage | ต่อเนื่อง (near real-time) | 35 วัน | ทำให้ RPO ≤ 5 นาที |
| 3. Logical backup | `pg_dump --format=custom` ต่อ schema | สัปดาห์ละครั้ง | 90 วัน | กู้แบบเลือกเฉพาะตาราง/สคีมา, ย้ายเวอร์ชัน PostgreSQL |

### 794.2 ตั้งค่า WAL Archiving (postgresql.conf)

```ini
# postgresql.conf — ตั้งค่าสำหรับ continuous archiving (ต่อยอด Part 062)
wal_level = replica
archive_mode = on
archive_command = 'wal-g wal-push %p'
    # ใช้ wal-g หรือ pgBackRest ส่ง WAL ไปยัง S3-compatible object storage
    # ห้ามใช้ cp ธรรมดาไปยัง local disk เดียวกับ data directory (ไม่กันภัยพิบัติจริง)
archive_timeout = 60
    # บังคับ switch WAL segment ทุก 60 วินาทีแม้ไม่เต็ม segment
    # เพื่อรับประกันว่า RPO จะไม่เกิน ~1 นาที ไม่ใช่รอจน segment 16MB เต็มเอง
max_wal_senders = 10
wal_keep_size = '4GB'
```

### 794.3 กำหนดตารางเวลาด้วย pgBackRest (ทางเลือกที่นิยมกว่าสำหรับ production จริง)

```ini
# /etc/pgbackrest/pgbackrest.conf
[global]
repo1-type=s3
repo1-s3-bucket=shopflow-pg-backups
repo1-s3-region=ap-southeast-1
repo1-retention-full=2
repo1-retention-diff=7
repo1-path=/pgbackrest
process-max=4
compress-type=zst
compress-level=3

[shopflow]
pg1-path=/var/lib/postgresql/16/main
pg1-port=5432
```

```bash
# Cron schedule บนเครื่อง standby (ลด I/O บน primary):
# Full backup ทุกวันอาทิตย์ 02:00
0 2 * * 0  pgbackrest --stanza=shopflow --type=full backup
# Differential backup ทุกวันอื่น 02:00
0 2 * * 1-6 pgbackrest --stanza=shopflow --type=diff backup
```

### 794.4 แผน PITR สำหรับ Incident Response — ต่อยอด Part 062

สถานการณ์จำลอง: วิศวกรรัน migration script พลาด ลบข้อมูล `orders` ของ tenant_id = 5 ทั้งหมดเมื่อเวลา 14:32:10 วันนี้ ต้องกู้กลับเฉพาะข้อมูลก่อนเวลานั้น

```bash
# 1) กู้ base backup ล่าสุดไปยังเครื่องแยก (recovery instance ไม่ใช่ production)
pgbackrest --stanza=shopflow --type=time \
    --target="2026-09-25 14:32:00+07" \
    --target-action=promote \
    restore

# 2) postgresql.auto.conf ที่ pgBackRest generate ให้อัตโนมัติจะมี:
#    restore_command = 'pgbackrest --stanza=shopflow archive-get %f %p'
#    recovery_target_time = '2026-09-25 14:32:00+07'

# 3) เมื่อ instance กู้เสร็จและ promote แล้ว → ตรวจสอบข้อมูลที่ recovery instance
#    แล้ว export เฉพาะ tenant_id = 5 กลับเข้า production ด้วย pg_dump --table + filter
pg_dump -h recovery-host -U shopflow_admin -d shopflow \
    --table=orders --table=order_items \
    -a --format=custom -f recovered_tenant5_orders.dump
# จากนั้นใช้สคริปต์ restore เฉพาะ WHERE tenant_id = 5 เข้า production (ไม่ restore ทั้งตาราง)
```

> **บทเรียนสำคัญ:** การมี RLS (Step 792) ทำให้ "การกู้ข้อมูลเฉพาะ tenant" เป็นไปได้ง่ายขึ้นมาก เพราะทุกตารางมี `tenant_id` กำกับชัดเจนอยู่แล้ว ต่างจากระบบที่ไม่มี multi-tenant design ที่ชัดเจนซึ่งอาจต้องกู้ทั้งฐานข้อมูลแล้วเสี่ยง overwrite ข้อมูล tenant อื่นที่เปลี่ยนไปแล้วหลัง incident

### 794.5 คำนวณ RPO/RTO จริงจากค่า config

| พารามิเตอร์ | ค่า | ผลต่อ RPO/RTO |
|---|---|---|
| `archive_timeout = 60s` | WAL segment ถูกส่งอย่างน้อยทุก 60 วินาที | RPO เชิงทฤษฎี ≤ 60 วินาที (ดีกว่าเป้าหมาย 5 นาที) |
| Full backup ทุกวัน + diff รายวัน | การ replay WAL ตอน PITR ไม่ต้องย้อนเกิน 24 ชม. | ลดเวลา replay WAL ตอนกู้ระบบ |
| Recovery instance เตรียม provision ล่วงหน้า (warm standby image) | ไม่ต้องรอ provision เครื่องใหม่ตอน incident | ช่วยให้ RTO ≤ 15 นาทีเป็นไปได้จริง |
| ทดสอบ restore drill ทุกเดือน (Part 079 DR Planning) | ยืนยันว่า backup ใช้ได้จริง ไม่ใช่แค่ "มีไฟล์อยู่" | ลดความเสี่ยง backup เสียหายแบบไม่รู้ตัว |

### 794.6 Backup Runbook สรุปสั้น (สำหรับทีม on-call)

```text
[ALERT] Data loss incident detected
  1. หยุด write ที่ tenant/table ที่ได้รับผลกระทบทันที (application feature flag)
  2. ระบุเวลาที่ต้องการกู้คืน (recovery_target_time) จาก log/audit trail
  3. Spin up recovery instance จาก backup ล่าสุดที่เก่ากว่าเวลานั้น
  4. Restore ด้วย pgBackRest --type=time --target=<timestamp>
  5. ตรวจสอบข้อมูลบน recovery instance ก่อน merge กลับ production เสมอ
  6. Export เฉพาะข้อมูลที่จำเป็น (filter ด้วย tenant_id) กลับเข้า production
  7. เปิด write กลับคืน + เขียน incident postmortem
```

---

## Step 795: สถาปัตยกรรม High Availability แบบเต็มรูปแบบ (Primary + Sync Standby + Async Standby + Patroni)

### 795.1 ทบทวนจาก Part 063–065 และประกอบเป็นสถาปัตยกรรมเดียว

SLA 99.95% (downtime ≤ 21.6 นาที/เดือน) ทำให้ ShopFlow เลือกใช้สถาปัตยกรรม **3-node cluster ต่อ region** บริหารจัดการด้วย **Patroni** (ต่อยอด Part 065) ผสาน **Streaming Replication** (Part 063) แบบ synchronous กับโหนดในโซนเดียวกัน และ **asynchronous** ไปยัง DR site อีกภูมิภาค

```
                           ┌────────────────────────────────────┐
                           │              etcd cluster            │  (3 nodes, consensus store)
                           │        เก็บ leader lock + config      │
                           └───────┬──────────┬──────────┬───────┘
                                   │           │          │
                     ┌─────────────▼──┐ ┌──────▼──────┐ ┌─▼─────────────┐
                     │ Patroni + PG    │ │ Patroni + PG │ │ Patroni + PG   │
                     │ node-1 (Primary)│ │ node-2 (Sync)│ │ node-3 (Async) │
                     │ AZ-a            │ │ AZ-b         │ │ AZ-c / DR-region│
                     └─────────────────┘ └──────────────┘ └────────────────┘
                              │  synchronous_commit = on   │  async streaming
                              └─────────────────────────────┘
```

### 795.2 ตั้งค่า Patroni (patroni.yml) — โหนดหลัก

```yaml
# /etc/patroni/patroni.yml (node-1)
scope: shopflow-cluster
namespace: /shopflow/
name: node-1

restapi:
  listen: 0.0.0.0:8008
  connect_address: 10.0.1.11:8008

etcd3:
  hosts: 10.0.1.21:2379,10.0.1.22:2379,10.0.1.23:2379

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576   # 1MB — ห้าม promote node ที่ lag เกินนี้
    synchronous_mode: true
    synchronous_mode_strict: false      # ถ้า sync standby ล่มหมด ยังเขียนต่อได้ (เลือก availability)
    postgresql:
      use_pg_rewind: true
      parameters:
        wal_level: replica
        hot_standby: "on"
        max_connections: 500
        synchronous_commit: "on"
        synchronous_standby_names: "ANY 1 (node-2, node-3)"
        shared_buffers: "8GB"
        effective_cache_size: "24GB"

postgresql:
  listen: 0.0.0.0:5432
  connect_address: 10.0.1.11:5432
  data_dir: /var/lib/postgresql/16/main
  authentication:
    replication:
      username: replicator
      password: "${REPLICATOR_PASSWORD}"
    superuser:
      username: postgres
      password: "${POSTGRES_SUPERUSER_PASSWORD}"

tags:
  nofailover: false
  noloadbalance: false
  clonefrom: true
```

### 795.3 กลไก Failover — ขั้นตอนที่เกิดขึ้นอัตโนมัติ

1. Patroni บน primary (`node-1`) ต่ออายุ leader lock ใน etcd ทุก `loop_wait` (10 วินาที) ผ่าน TTL 30 วินาที
2. หาก `node-1` ล่ม → ไม่ต่ออายุ lock → lock หมดอายุใน etcd ภายใน ≤ 30 วินาที
3. Patroni บนโหนดที่เหลือแข่งกัน acquire lock — เลือกโหนดที่มี WAL ทันสมัยที่สุด (ตรวจสอบผ่าน `maximum_lag_on_failover`)
4. โหนดที่ชนะ (มักเป็น `node-2` เพราะ sync standby มี WAL ทันสมัยเสมอ) รัน `pg_promote()` กลายเป็น primary ใหม่
5. โหนดอื่นที่เหลือ reconfigure ตัวเองเป็น standby ของ leader ใหม่ผ่าน `pg_rewind` (ไม่ต้อง full re-clone)
6. HAProxy/PgBouncer (Step 796) ตรวจสุขภาพผ่าน Patroni REST API แล้วสลับปลายทาง write ไปโหนดใหม่ภายในไม่กี่วินาที

**เวลารวมของ failover โดยทั่วไป: 15–30 วินาที** ซึ่งอยู่ภายใต้งบ downtime 21.6 นาที/เดือนได้สบาย แม้เกิด failover เดือนละหลายครั้ง

### 795.4 เหตุใดต้องมีทั้ง Sync และ Async Standby

| ประเภท | บทบาท | Trade-off |
|---|---|---|
| **Synchronous standby** (node-2, โซนเดียวกัน) | รับประกันว่าข้อมูลที่ commit แล้วจะไม่หายแม้ primary ล่มกะทันหัน (RPO = 0 สำหรับ transaction ที่ commit แล้ว) | เพิ่ม write latency เล็กน้อย (ต้องรอ ack จาก standby) |
| **Asynchronous standby** (node-3, ต่าง region) | ใช้เป็น Disaster Recovery site กรณีทั้ง region ล่ม (เช่น data center ไฟไหม้) | อาจสูญเสียข้อมูลไม่กี่วินาทีสุดท้าย (RPO > 0) หากเกิด regional disaster |

การตั้งค่า `synchronous_standby_names = 'ANY 1 (node-2, node-3)'` หมายถึง primary จะรอ ack จากโหนดใดก็ได้ 1 โหนดในกลุ่มนี้ก่อน commit — ทำให้ระบบยังทนต่อการที่โหนดใดโหนดหนึ่งใน 2 โหนดนี้ล่มโดยไม่หยุดรับ write

### 795.5 คำนวณ Availability กับ SLA 99.95%

```
Downtime budget ต่อเดือน = 30 วัน × 24 ชม. × 60 นาที × (1 - 0.9995) = 21.6 นาที

สมมติ:
  - Failover 1 ครั้ง ใช้เวลาเฉลี่ย 20 วินาที
  - Planned maintenance (minor version patch) ทำผ่าน rolling restart ไม่มี downtime
    (Patroni switchover แบบ manual, zero-downtime)
  - เหตุ failover ที่คาดไว้: ไม่เกิน 4 ครั้ง/เดือน (hardware issue, AZ blip)

Downtime ที่คาดไว้ = 4 × 20 วินาที = 80 วินาที ≈ 1.3 นาที
=> เหลือ budget อีก ~20 นาที สำหรับเหตุการณ์ไม่คาดฝัน (unplanned)
```

### 795.6 เชื่อมโยงกับ Part 079 (Disaster Recovery Planning)

สถาปัตยกรรมนี้ต้องผ่านการทดสอบ **DR drill รายไตรมาส**: จำลองการ kill `node-1` แบบสุ่ม (chaos engineering) แล้ววัดเวลา failover จริง เปรียบเทียบกับตัวเลขทฤษฎีด้านบน — สิ่งนี้คือสิ่งที่ Part 079 สอนไว้ และนำมาปฏิบัติจริงในระบบ ShopFlow

---

## Step 796: Connection Pooling และ Load Balancing Layer สำหรับรองรับ Tenant จำนวนมาก

### 796.1 ปัญหาที่ต้องแก้ — ทบทวนจาก Part 066–067

PostgreSQL แต่ละ connection ใช้ process แยก (ไม่ใช่ thread) กิน RAM ~5–10MB ต่อ connection และ `max_connections` ที่ตั้งไว้สูงเกินไป (เช่น 2,000) จะทำให้ context-switching overhead สูงจนระบบช้าลงทั้งระบบ

ShopFlow มี backend service instance ~150 ตัว (auto-scaling ตาม load) แต่ละตัวเปิด pool ของตัวเอง หากไม่มี pooling layer กลาง จำนวน connection ที่ PostgreSQL primary ต้องรับจะเป็น `150 instances × 20 connections/instance = 3,000 connections` ซึ่งเกินขีดความสามารถของ PostgreSQL มาก

### 796.2 สถาปัตยกรรม PgBouncer แบบ Transaction Pooling

```
150 backend instances (×20 conn each = 3,000 client conn)
                    │
                    ▼
        ┌───────────────────────┐
        │   PgBouncer cluster     │   3 nodes (active-active behind L4 LB)
        │   pool_mode = transaction│
        └───────────┬─────────────┘
                    │  รวมเหลือ ~300 conn จริงไปยัง PostgreSQL
                    ▼
        ┌───────────────────────┐
        │  PostgreSQL Primary     │   max_connections = 500
        └─────────────────────────┘
```

```ini
# /etc/pgbouncer/pgbouncer.ini
[databases]
shopflow_rw = host=patroni-vip port=5432 dbname=shopflow
shopflow_ro = host=patroni-replica-vip port=5432 dbname=shopflow

[pgbouncer]
listen_addr = 0.0.0.0
listen_port = 6432
auth_type = scram-sha-256
auth_file = /etc/pgbouncer/userlist.txt

pool_mode = transaction
    ; transaction pooling: คืน connection กลับ pool ทันทีที่ transaction จบ
    ; เหมาะกับ ShopFlow เพราะทุก request ใช้ SET LOCAL app.current_tenant ต่อ transaction
    ; (SET LOCAL จะไม่หลุดข้าม transaction เมื่อใช้ transaction pooling — เป็นค่าที่ตั้งใจเลือก)

max_client_conn = 5000
default_pool_size = 25
    ; ต่อ (user, database) pair — คำนวณจาก: 500 max_connections / จำนวน role ที่ active
reserve_pool_size = 5
reserve_pool_timeout = 3

server_idle_timeout = 600
query_wait_timeout = 30
```

> **ข้อควรระวังสำคัญเรื่อง RLS + Transaction Pooling:** เพราะเราใช้ `SET LOCAL app.current_tenant` (ไม่ใช่ `SET` เฉยๆ) ค่านี้จะถูกล้างอัตโนมัติเมื่อ transaction จบ — ปลอดภัยสำหรับ transaction pooling ที่ connection ถูกใช้ซ้ำโดย request อื่นทันที **ถ้าหากใช้ `SET` ธรรมดาแทน `SET LOCAL` จะเกิดช่องโหว่ร้ายแรง**: tenant ถัดไปที่ใช้ connection เดียวกันอาจได้รับค่า `app.current_tenant` ของ tenant ก่อนหน้าที่ค้างอยู่ ทำให้เกิด **cross-tenant data leak** — นี่คือจุดตัดสำคัญที่เชื่อม Part 066 (Pooling) เข้ากับ Part 069 (RLS) โดยตรง

### 796.3 Load Balancing สำหรับ Read Replica — ทบทวนจาก Part 067

```
                     ┌──────────────────────┐
                     │  Application (read)   │
                     └───────────┬────────────┘
                                 │
                     ┌───────────▼────────────┐
                     │  HAProxy (health check   │
                     │  via Patroni REST /replica)│
                     └──────┬────────────┬───────┘
                            │            │
                 ┌───────────▼──┐   ┌─────▼─────────┐
                 │ Read Replica 1│   │ Read Replica 2 │
                 └────────────────┘  └────────────────┘
```

```
# /etc/haproxy/haproxy.cfg (ส่วน read replica pool)
backend shopflow_read_replicas
    option httpchk GET /replica
    http-check expect status 200
    default-server inter 3s fall 3 rise 2
    server replica1 10.0.1.12:6432 check port 8008
    server replica2 10.0.1.13:6432 check port 8008
    balance leastconn
```

`GET /replica` เป็น endpoint ของ Patroni REST API ที่ตอบ HTTP 200 เฉพาะเมื่อโหนดเป็น standby ที่ healthy — ทำให้ HAProxy ไม่ส่ง traffic ไปยังโหนดที่กำลัง lag สูงหรือกำลังถูก promote

### 796.4 คำนวณขนาด Pool ที่เหมาะสม (Little's Law)

```
สูตรประมาณการ: connections ที่ต้องใช้ ≈ (throughput req/sec) × (avg query time sec)

ตัวอย่าง ShopFlow ช่วง peak:
  throughput = 3,000 req/sec
  avg query time = 8ms = 0.008 sec
  connections ที่ต้องใช้จริง ≈ 3,000 × 0.008 = 24 connections

=> ตั้ง default_pool_size ที่ ~25-30 ก็เพียงพอสำหรับ throughput ระดับนี้
   ไม่จำเป็นต้องเปิด max_connections สูงๆ เลย นี่คือพลังของ pooling
```

### 796.5 เชื่อมกับ Step 793 (Roles): แยก pool ตาม role

เพราะเรามีหลาย role (`shopflow_app`, `shopflow_analyst`, `shopflow_reporting`) แต่ละ role ควรมี pool แยกใน PgBouncer (`default_pool_size` แยกต่อ role) เพื่อไม่ให้ query หนักจากฝั่ง analyst ไปแย่ง connection budget ของ application หลัก — ป้องกัน "noisy neighbor" ระดับ connection pool

---

## Step 797: แผน Security แบบเต็มรูปแบบ — SSL/TLS, Encryption, Audit Logging

### 797.1 แนวคิด Defense-in-Depth — ทบทวนจาก Part 070–071

ระบบ SaaS ที่เก็บข้อมูลลูกค้าของหลายบริษัท ต้องมีการป้องกันหลายชั้น ไม่พึ่งพา RLS เพียงอย่างเดียว:

```
ชั้นที่ 1: Network       →  TLS ทุกการเชื่อมต่อ, firewall/security group จำกัด IP
ชั้นที่ 2: Authentication →  scram-sha-256, certificate-based auth สำหรับ admin
ชั้นที่ 3: Authorization  →  Roles (Step 793) + RLS (Step 792)
ชั้นที่ 4: Encryption at rest → disk-level encryption + column-level สำหรับข้อมูลอ่อนไหว
ชั้นที่ 5: Audit          →  pgaudit บันทึกทุกการเข้าถึงข้อมูลอ่อนไหว
```

### 797.2 บังคับ SSL/TLS ทุกการเชื่อมต่อ

```ini
# postgresql.conf
ssl = on
ssl_cert_file = '/etc/postgresql/certs/server.crt'
ssl_key_file = '/etc/postgresql/certs/server.key'
ssl_ca_file = '/etc/postgresql/certs/ca.crt'
ssl_min_protocol_version = 'TLSv1.2'
ssl_prefer_server_ciphers = on
```

```conf
# pg_hba.conf — บังคับ hostssl เท่านั้น ปฏิเสธการเชื่อมต่อแบบไม่เข้ารหัส
# TYPE      DATABASE   USER                 ADDRESS            METHOD
hostssl     shopflow   shopflow_app         10.0.2.0/24        scram-sha-256
hostssl     shopflow   shopflow_migrator    10.0.3.0/24        scram-sha-256   clientcert=verify-full
hostssl     shopflow   shopflow_analyst     10.0.4.0/24        scram-sha-256
hostssl     shopflow   shopflow_admin       10.0.5.0/24        cert            clientcert=verify-full
host        all        all                  0.0.0.0/0          reject
```

`clientcert=verify-full` สำหรับ `shopflow_migrator` และ `shopflow_admin` หมายถึงต้องมี client certificate ที่เชื่อถือได้ ไม่ใช่แค่ password — เพิ่มชั้นความปลอดภัยสำหรับ role ที่มีสิทธิ์สูง (DDL, BYPASSRLS)

### 797.3 Encryption at Rest

| ระดับ | เทคนิค | ใช้เมื่อไร |
|---|---|---|
| Disk-level (Transparent) | LUKS (Linux) หรือ cloud-managed disk encryption (EBS encryption, GCP CMEK) | ป้องกันการขโมย physical disk/snapshot |
| Column-level (pgcrypto) | `pgp_sym_encrypt()` สำหรับฟิลด์อ่อนไหวเป็นพิเศษ | ข้อมูลที่แม้ DBA ก็ไม่ควรอ่านตรงๆ ได้ เช่น เลขบัตรเครดิตที่ถูกบันทึกไว้ชั่วคราว |

```sql
-- ตัวอย่าง column-level encryption สำหรับข้อมูลที่อ่อนไหวเป็นพิเศษ (ใช้ pgcrypto)
CREATE EXTENSION IF NOT EXISTS pgcrypto;

-- สมมติเพิ่มคอลัมน์เก็บ token การชำระเงินแบบเข้ารหัส (ไม่ใช่เลขบัตรจริง ซึ่งไม่ควรเก็บเองอยู่แล้ว)
ALTER TABLE customers ADD COLUMN payment_note_encrypted BYTEA;

-- เข้ารหัสตอน insert/update
UPDATE customers
SET payment_note_encrypted = pgp_sym_encrypt('reference-token-xyz', current_setting('app.encryption_key'))
WHERE customer_id = 1;

-- ถอดรหัสตอนอ่าน (เฉพาะ role ที่มีสิทธิ์และรู้ key เท่านั้น)
SELECT pgp_sym_decrypt(payment_note_encrypted, current_setting('app.encryption_key'))
FROM customers WHERE customer_id = 1;
```

> ในระบบจริง `app.encryption_key` ไม่ควรอยู่ใน connection string หรือ config ธรรมดา แต่ควรดึงจาก secret manager (AWS KMS, HashiCorp Vault) แบบ per-request หรือใช้ envelope encryption

### 797.4 Audit Logging ด้วย pgaudit — ทบทวนจาก Part 071

```ini
# postgresql.conf
shared_preload_libraries = 'pgaudit'
pgaudit.log = 'write, ddl, role'
    # write = INSERT/UPDATE/DELETE, ddl = CREATE/ALTER/DROP, role = GRANT/REVOKE
pgaudit.log_catalog = off
pgaudit.log_parameter = on
    # บันทึกค่า parameter จริง (ต้องระวังข้อมูลอ่อนไหวรั่วใน log — มาสก์ก่อน export)
pgaudit.log_relation = on
```

```sql
-- บังคับ audit แบบเข้มข้นเป็นพิเศษสำหรับ role ที่มี BYPASSRLS
-- (ทุกครั้งที่ shopflow_reporting หรือ shopflow_admin query ข้าม tenant ต้องมี log)
ALTER ROLE shopflow_reporting SET pgaudit.log = 'read, write';
ALTER ROLE shopflow_admin     SET pgaudit.log = 'read, write, ddl, role';
```

```sql
-- ตารางเก็บ audit trail ระดับ application (เสริมจาก pgaudit) — สำหรับ trace การเข้าถึงข้าม tenant
CREATE TABLE cross_tenant_access_log (
    log_id       BIGSERIAL PRIMARY KEY,
    accessed_by  TEXT NOT NULL DEFAULT current_user,
    accessed_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    tenant_id    INTEGER,
    reason       TEXT,
    query_text   TEXT
);

-- ตัวอย่าง trigger บันทึกทุกครั้งที่ shopflow_support เปิดดู tenant ใด (สำหรับ compliance)
CREATE OR REPLACE FUNCTION log_support_access() RETURNS TRIGGER AS $$
BEGIN
    IF current_user = 'shopflow_support' THEN
        INSERT INTO cross_tenant_access_log (tenant_id, reason, query_text)
        VALUES (NEW.tenant_id, 'customer_support_lookup', current_query());
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

### 797.5 Compliance Checklist (PDPA/GDPR-like) สำหรับ SaaS

| ข้อกำหนด | วิธีที่ ShopFlow ตอบสนอง |
|---|---|
| Right to erasure (ลบข้อมูลลูกค้าตามคำขอ) | `DELETE FROM customers WHERE customer_id = ... AND tenant_id = ...` ภายใต้ RLS + เก็บ log การลบ |
| Data minimization | Column-level privilege (Step 793) ป้องกันการ SELECT คอลัมน์ที่ไม่จำเป็น เช่น `password_hash` |
| Audit trail การเข้าถึง | pgaudit + `cross_tenant_access_log` เก็บ 1 ปีตามข้อกำหนด |
| Encryption in transit/at rest | TLS ทุก connection + disk encryption + column encryption สำหรับข้อมูลอ่อนไหว |
| Data residency (ถ้าลูกค้าต้องการเก็บข้อมูลในภูมิภาคเฉพาะ) | วางแผนแบ่ง shard ตามภูมิภาคใน Step 799 (scaling) |

---

## Step 798: Monitoring และ Alerting Stack สำหรับ SaaS หลาย Tenant

### 798.1 สถาปัตยกรรม Monitoring — ทบทวนจาก Part 072

```
┌──────────────┐     ┌──────────────┐     ┌───────────────┐
│ PostgreSQL     │     │ node_exporter  │     │ pgbouncer_exp  │
│ + pg_stat_*    │     │ (OS metrics)   │     │ (pool metrics) │
│ postgres_exporter│    └──────┬────────┘     └───────┬────────┘
└───────┬─────────┘            │                       │
        └───────────┬──────────┴───────────┬───────────┘
                     ▼                      ▼
              ┌────────────────────────────────┐
              │           Prometheus              │  (scrape ทุก 15s, เก็บ 30 วัน)
              └───────────────┬────────────────┘
                              ▼
              ┌────────────────────────────────┐
              │      Alertmanager → PagerDuty/   │
              │         Slack (on-call)          │
              └────────────────────────────────┘
                              │
              ┌───────────────▼────────────────┐
              │           Grafana Dashboards      │
              └────────────────────────────────┘
```

### 798.2 กำหนด SLI/SLO สำหรับ ShopFlow

| SLI (Service Level Indicator) | SLO (เป้าหมาย) | วิธีวัด |
|---|---|---|
| Availability | 99.95% ต่อเดือน | `up{job="postgres_exporter"}` |
| Query latency (p95) | < 100ms สำหรับ OLTP query | `pg_stat_statements.mean_exec_time` |
| Replication lag | < 5 วินาที (async standby) | `pg_stat_replication.replay_lag` |
| Connection pool saturation | < 80% ของ `default_pool_size` | PgBouncer `SHOW POOLS` |
| Backup success rate | 100% ของ scheduled backup ต้องสำเร็จ | pgBackRest exit code + Prometheus pushgateway |

### 798.3 Dashboard หลักที่ต้องมีบน Grafana (ต่อยอด Part 072)

1. **Tenant Health Overview** — จำนวน active tenant, query count ต่อ tenant top 10 (หา noisy tenant)
2. **Replication Topology** — lag ของแต่ละ standby, สถานะ Patroni leader/replica แบบ real-time
3. **Connection Pool Saturation** — PgBouncer pool utilization ต่อ role/database
4. **Query Performance** — top 10 query จาก `pg_stat_statements` เรียงตาม total_exec_time
5. **Vacuum & Bloat** — ตารางที่ dead tuple สูงสุด, autovacuum ที่ค้างนาน
6. **Disk & WAL** — WAL generation rate, disk free space, archive lag

### 798.4 Alert Rules ที่สำคัญ (Prometheus Alertmanager)

```yaml
# alert_rules.yml
groups:
  - name: shopflow_postgres_critical
    rules:
      - alert: PostgreSQLPrimaryDown
        expr: pg_up{role="primary"} == 0
        for: 30s
        labels: { severity: critical }
        annotations:
          summary: "Primary PostgreSQL ไม่ตอบสนอง — ตรวจสอบ Patroni failover ทันที"

      - alert: ReplicationLagHigh
        expr: pg_replication_lag_seconds > 10
        for: 2m
        labels: { severity: warning }
        annotations:
          summary: "Replication lag เกิน 10 วินาที — เสี่ยงกระทบ RPO"

      - alert: ConnectionPoolSaturation
        expr: pgbouncer_pools_client_waiting_count > 0
        for: 1m
        labels: { severity: warning }
        annotations:
          summary: "มี client รอ connection ใน PgBouncer pool — พิจารณาขยาย pool_size"

      - alert: LongRunningTransaction
        expr: pg_stat_activity_max_tx_duration_seconds > 300
        for: 1m
        labels: { severity: warning }
        annotations:
          summary: "มี transaction ค้างนานเกิน 5 นาที — เสี่ยงบล็อก autovacuum และ hold lock"

      - alert: BackupFailed
        expr: pgbackrest_backup_last_success_seconds > 90000
          # 25 ชม. — ไม่มี backup สำเร็จในรอบวันล่าสุด
        for: 5m
        labels: { severity: critical }
        annotations:
          summary: "ไม่มี backup สำเร็จในรอบ 25 ชม.ล่าสุด — เสี่ยง RPO/RTO SLA"

      - alert: TenantAnomalousQueryVolume
        expr: rate(shopflow_query_count{tenant_id!=""}[5m]) > 500
        for: 3m
        labels: { severity: warning }
        annotations:
          summary: "Tenant {{ $labels.tenant_id }} มี query volume ผิดปกติ — ตรวจสอบ noisy neighbor หรือ abuse"
```

### 798.5 เชื่อมโยง Application-level Metrics เข้ากับ Prometheus

```sql
-- Custom view สำหรับ export metric ระดับ tenant ผ่าน postgres_exporter custom queries
CREATE OR REPLACE VIEW metrics_tenant_activity AS
SELECT t.tenant_id, t.tenant_name, t.plan,
       count(o.order_id) FILTER (WHERE o.order_date > now() - interval '1 hour') AS orders_last_hour,
       count(DISTINCT o.customer_id) FILTER (WHERE o.order_date > now() - interval '1 day') AS active_customers_today
FROM tenants t
LEFT JOIN orders o ON o.tenant_id = t.tenant_id
GROUP BY t.tenant_id, t.tenant_name, t.plan;
```

```yaml
# postgres_exporter custom-queries.yaml
shopflow_tenant_activity:
  query: "SELECT tenant_id, plan, orders_last_hour FROM metrics_tenant_activity"
  metrics:
    - tenant_id: { usage: "LABEL" }
    - plan: { usage: "LABEL" }
    - orders_last_hour: { usage: "GAUGE" }
```

---

## Step 799: แผน Capacity Planning และ Scaling สำหรับการเติบโตของ Tenant

### 799.1 คาดการณ์การเติบโต — ทบทวนจาก Part 073–075

จากโจทย์ Step 791: tenant จะโต จาก 500 → 20,000 ร้าน ภายใน 1-2 ปี เราวางแผนเป็น 3 phase:

| Phase | จำนวน Tenant | สถาปัตยกรรม | จุดเปลี่ยนสำคัญ |
|---|---|---|---|
| Phase 1 (ปัจจุบัน) | < 2,000 | Single database, RLS, 1 primary + 2 standby | ตามที่ออกแบบใน Step 792–796 |
| Phase 2 | 2,000–20,000 | เพิ่ม partitioning ตามเวลา + read replica เพิ่ม + vertical scale primary | Step 799.2–799.4 |
| Phase 3 | 20,000+ | Sharding ตาม tenant_id (Citus หรือ application-level shard) | Step 799.5 |

### 799.2 Partitioning ตารางใหญ่ — ทบทวนจาก Part 074

ตาราง `orders` และ `order_items` จะเติบโตเร็วที่สุด (ทุก tenant เขียนตลอดเวลา) จึงต้อง partition ตามเวลาแบบ range partitioning ร่วมกับ `tenant_id` เป็น index รอง:

```sql
-- ออกแบบใหม่สำหรับ Phase 2: orders แบบ partition by range (order_date)
-- (แสดงแนวทาง — การ migrate ตารางจริงต้องทำผ่านกระบวนการ migration แยกต่างหาก)

CREATE TABLE orders_partitioned (
    order_id      BIGSERIAL,
    tenant_id     INTEGER NOT NULL,
    customer_id   INTEGER,
    order_date    TIMESTAMPTZ NOT NULL DEFAULT now(),
    status        VARCHAR(20) NOT NULL DEFAULT 'pending',
    total_amount  NUMERIC(12,2),
    PRIMARY KEY (order_id, order_date)
) PARTITION BY RANGE (order_date);

-- Partition รายเดือน สร้างล่วงหน้าด้วย pg_partman (Part 078) แบบอัตโนมัติ
CREATE TABLE orders_y2026m09 PARTITION OF orders_partitioned
    FOR VALUES FROM ('2026-09-01') TO ('2026-10-01');
CREATE TABLE orders_y2026m10 PARTITION OF orders_partitioned
    FOR VALUES FROM ('2026-10-01') TO ('2026-11-01');

-- Index ต้องมี tenant_id นำหน้าเสมอในทุก partition (เชื่อมกับหลักการ RLS ของ Step 792)
CREATE INDEX idx_orders_part_tenant ON orders_partitioned (tenant_id, order_date DESC);

-- เปิด RLS บนตาราง partition แม่ — จะ apply ไปยังทุก partition ลูกโดยอัตโนมัติ
ALTER TABLE orders_partitioned ENABLE ROW LEVEL SECURITY;
ALTER TABLE orders_partitioned FORCE ROW LEVEL SECURITY;
CREATE POLICY orders_partitioned_isolation ON orders_partitioned
    FOR ALL USING (tenant_id = current_tenant_id())
    WITH CHECK (tenant_id = current_tenant_id());
```

```sql
-- pg_partman (Part 078) ตั้งค่าสร้าง partition ล่วงหน้าอัตโนมัติ + ลบ partition เก่าตาม retention
SELECT partman.create_parent(
    p_parent_table => 'public.orders_partitioned',
    p_control      => 'order_date',
    p_type         => 'range',
    p_interval     => 'monthly',
    p_premake      => 3
);

-- ตั้งค่า retention: เก็บ 25 เดือน (2 ปี + กันชน) แล้วลบ partition เก่าอัตโนมัติผ่าน pg_cron
UPDATE partman.part_config
SET retention = '25 months', retention_keep_table = false
WHERE parent_table = 'public.orders_partitioned';
```

**ประโยชน์:** query ที่กรองช่วงเวลา (เช่น รายงานยอดขายเดือนนี้) จะ scan เฉพาะ partition ที่เกี่ยวข้อง (partition pruning) ไม่ต้อง scan ตารางทั้งหมดที่มีข้อมูลย้อนหลังหลายปี และการลบข้อมูลเก่าทำได้เร็วมากด้วย `DROP TABLE` ต่อ partition แทนการ `DELETE` แถวทีละล้านแถว

### 799.3 Vacuum & Autovacuum Tuning สำหรับตารางที่เขียนหนัก — ทบทวนจาก Part 077

```sql
-- ตาราง orders ถูกเขียน/อัปเดตสถานะบ่อยมาก (pending → paid → shipped) — ต้อง tune autovacuum เฉพาะตาราง
ALTER TABLE orders SET (
    autovacuum_vacuum_scale_factor = 0.02,   -- ค่า default 0.2 หยวนเกินไปสำหรับตารางที่ churn สูง
    autovacuum_vacuum_cost_limit   = 2000,
    autovacuum_analyze_scale_factor = 0.01
);

-- ตรวจสอบ bloat และสถานะ autovacuum ต่อตาราง (query สำหรับ dashboard, Part 072/077)
SELECT relname,
       n_live_tup, n_dead_tup,
       round(n_dead_tup::numeric / NULLIF(n_live_tup + n_dead_tup, 0) * 100, 2) AS dead_pct,
       last_autovacuum, last_autoanalyze
FROM pg_stat_user_tables
WHERE schemaname = 'public'
ORDER BY n_dead_tup DESC;
```

### 799.4 Vertical Scaling บน Primary — ค่าที่ต้องปรับตามจำนวน tenant

| จำนวน Tenant | vCPU | RAM | shared_buffers | max_connections (จริงหลัง pooling) | effective_cache_size |
|---|---|---|---|---|---|
| < 2,000 | 8 | 32GB | 8GB | 500 | 24GB |
| 2,000–10,000 | 16 | 64GB | 16GB | 500 (คงที่ เพราะมี pooling) | 48GB |
| 10,000–20,000 | 32 | 128GB | 32GB | 500 | 96GB |

สังเกตว่า `max_connections` **ไม่ต้องเพิ่มตามจำนวน tenant** เพราะ PgBouncer (Step 796) ดูดซับ connection จำนวนมากไว้แล้ว — นี่คือเหตุผลที่ pooling ถูกออกแบบไว้ตั้งแต่ Phase 1 ไม่ใช่มาเพิ่มทีหลัง

### 799.5 แผน Sharding เมื่อ Tenant เกิน 20,000 ร้าน — ทบทวนจาก Part 075

เมื่อ single-primary เริ่มชนขีดจำกัดด้าน write throughput หรือ storage (เช่น เกิน 5TB) ShopFlow จะย้ายไปสถาปัตยกรรม **shard ตาม tenant_id**:

```
Shard Router (application-level หรือ Citus coordinator)
        │
        ├── tenant_id % 4 == 0  →  Shard A (Primary + standby ชุดของตัวเอง)
        ├── tenant_id % 4 == 1  →  Shard B
        ├── tenant_id % 4 == 2  →  Shard C
        └── tenant_id % 4 == 3  →  Shard D

แต่ละ shard คือ PostgreSQL cluster สมบูรณ์ (Patroni + backup + monitoring ของตัวเอง)
ตามสถาปัตยกรรมที่ออกแบบใน Step 792–798 ทั้งหมด — สังเกตว่า "1 shard = 1 ระบบย่อยที่สมบูรณ์"
```

**ทางเลือกสำหรับการ shard:**

1. **Application-level sharding** — เขียน routing logic เองในชั้น API gateway (`shard = hash(tenant_id) % N`) ควบคุมได้เต็มที่ แต่ต้องดูแล cross-shard query เอง (เช่น รายงานผู้บริหารข้าม shard)
2. **Citus extension** — เปลี่ยนตารางเป็น distributed table (`SELECT create_distributed_table('orders', 'tenant_id')`) ให้ PostgreSQL/Citus จัดการ routing และ cross-shard query ให้อัตโนมัติ เหมาะกับทีมที่ต้องการลด operational overhead

```sql
-- ตัวอย่างแนวคิด (ถ้าเลือกใช้ Citus ในอนาคต) — แสดงเพื่อความเข้าใจ ไม่ใช่ต้อง apply ทันที
-- SELECT create_distributed_table('orders', 'tenant_id');
-- SELECT create_distributed_table('order_items', 'tenant_id');
-- Citus จะ colocate ตารางที่มี distribution key เดียวกัน (tenant_id) ไว้ shard เดียวกัน
-- ทำให้ JOIN ระหว่าง orders กับ order_items ของ tenant เดียวกันไม่ต้องข้าม network เลย
```

### 799.6 Query Performance Tuning ที่เกี่ยวพันกับ RLS ในสเกลใหญ่ — เชื่อม Part 076

เมื่อข้อมูลโตมาก query ที่ RLS ครอบไว้ต้องได้รับการ monitor เป็นพิเศษ เพราะ planner อาจเปลี่ยน plan เมื่อสถิติข้อมูล (statistics) เปลี่ยนไปตามขนาดตาราง:

```sql
-- ตรวจสอบว่า planner ยังใช้ index scan ที่มี tenant_id นำหน้าอยู่หรือไม่ หลังข้อมูลโตมาก
BEGIN;
SET LOCAL app.current_tenant = '1';
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT o.order_id, o.total_amount, c.first_name
FROM orders o
JOIN customers c ON c.customer_id = o.customer_id
WHERE o.order_date > now() - interval '7 days';
ROLLBACK;

-- ถ้าพบ Seq Scan แทน Index Scan ทั้งที่มี index อยู่ ให้ตรวจสอบ:
--   1. ANALYZE ตารางล่าสุดหรือยัง (สถิติเก่าทำให้ planner ประเมินผิด)
--   2. default_statistics_target สำหรับคอลัมน์ tenant_id เพียงพอหรือไม่ (ค่า default 100 อาจไม่พอ
--      เมื่อ tenant กระจายไม่สม่ำเสมอ เช่น tenant ใหญ่มีข้อมูลเป็นล้านแถว tenant เล็กมีสิบแถว)
ALTER TABLE orders ALTER COLUMN tenant_id SET STATISTICS 500;
ANALYZE orders;
```

---

## Step 800: ทบทวนสรุปทั้งหมดที่เรียนมาในระดับมืออาชีพ (Part 061–080)

### 800.1 ตาราง Mapping ฉบับสมบูรณ์: Part → บทบาทในสถาปัตยกรรม → สถานะการนำไปใช้ใน ShopFlow

| Part | หัวข้อ | นำไปใช้ใน ShopFlow อย่างไร |
|---|---|---|
| 061 | Backup Strategies | pgBackRest full/diff backup รายวัน (Step 794) |
| 062 | Point-in-Time Recovery | WAL archiving + `archive_timeout=60s` + PITR runbook (Step 794) |
| 063 | Streaming Replication | รากฐานของ sync/async standby (Step 795) |
| 064 | Logical Replication | ส่งข้อมูลไป data warehouse แยกจาก OLTP (Step 799 แนวคิด) |
| 065 | High Availability | Patroni cluster + etcd + auto failover (Step 795) |
| 066 | Connection Pooling | PgBouncer transaction pooling รองรับ 150 backend instances (Step 796) |
| 067 | Load Balancing | HAProxy กระจาย read query ไป replica (Step 796) |
| 068 | Roles & Privileges | 6 role แยกตามหน้าที่ ยึด least privilege (Step 793) |
| 069 | Row Level Security | RLS ทุกตาราง + FORCE RLS + fail-closed design (Step 792) |
| 070 | SSL/TLS & Encryption | `hostssl` บังคับทุก connection, TLS 1.2+ (Step 797) |
| 071 | Encryption at Rest & Audit | pgcrypto column encryption + pgaudit (Step 797) |
| 072 | Monitoring & Observability | Prometheus + Grafana + SLI/SLO (Step 798) |
| 073 | Capacity Planning | ตาราง sizing ตามจำนวน tenant (Step 799) |
| 074 | Partitioning at Scale | Range partition ตามเดือนบน `orders` (Step 799) |
| 075 | Scaling Strategies | แผน shard ตาม tenant_id เมื่อเกิน 20,000 ร้าน (Step 799) |
| 076 | Query Performance Tuning | ตรวจ EXPLAIN ให้ RLS ยัง push down เป็น index scan (Step 792, 799) |
| 077 | Vacuum & Autovacuum | Tune ตาราง `orders` ที่ churn สูงเป็นพิเศษ (Step 799) |
| 078 | Extensions ขั้นสูง | pg_partman (auto-partition), pgcrypto, pgaudit, pg_cron | 794, 797, 799 |
| 079 | Disaster Recovery Planning | DR drill รายไตรมาส, chaos testing failover (Step 795, 800) |
| 080 | **โปรเจกต์ปิดท้าย (บทนี้)** | สังเคราะห์ทั้งหมดเป็นสถาปัตยกรรมเดียว | — |

### 800.2 สถาปัตยกรรมรวมฉบับเต็ม (ASCII Diagram)

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                          CLIENT LAYER (Web / Mobile App)                       │
└───────────────────────────────────┬────────────────────────────────────────────┘
                                    │ HTTPS (TLS 1.2+)
┌───────────────────────────────────▼────────────────────────────────────────────┐
│  API GATEWAY / APPLICATION SERVERS (150 instances, auto-scaling)                │
│  - Authenticate → resolve tenant_id → SET LOCAL app.current_tenant             │
│  - ใช้ role shopflow_app (least privilege, ไม่มี DDL)                            │
└───────────────────┬─────────────────────────────────┬───────────────────────────┘
                    │ write (RW)                       │ read (RO)
┌───────────────────▼───────────────┐   ┌──────────────▼───────────────────────┐
│   PgBouncer RW cluster              │   │   PgBouncer RO cluster                 │
│   pool_mode=transaction              │   │   pool_mode=transaction                │
│   (Step 796 — ลด 3,000→300 conn)     │   │                                        │
└───────────────────┬─────────────────┘   └──────────────┬───────────────────────┘
                    │                                    │
                    ▼                                    ▼
        ┌───────────────────────┐              ┌─────────────────────────┐
        │  HAProxy (write VIP)    │              │  HAProxy (read, leastconn)│
        │  → ชี้ไปยัง Patroni leader│              │  → health check /replica  │
        └───────────┬─────────────┘              └───────┬──────────┬───────┘
                    │                                    │          │
┌───────────────────▼──────────────────────────────────▼──┐    ┌───▼────────────┐
│              PATRONI + ETCD CLUSTER (Step 795)              │    │ Read Replica 2  │
│  ┌────────────────┐  sync   ┌────────────────┐             │    │ (analytics only, │
│  │  Primary (RW)    │────────▶│ Sync Standby    │             │    │  shopflow_analyst)│
│  │  AZ-a             │        │ AZ-b            │◀────────────┘    └──────────────────┘
│  │  RLS FORCE ON     │        └────────────────┘                                        
│  │  ทุกตาราง (Step 792)│              │ async
│  └─────────┬──────────┘              ▼
│            │              ┌────────────────────┐
│            │ async        │  Async Standby (DR)  │
│            └─────────────▶│  ต่าง region          │
│                            └────────────────────┘
└──────────────────────────────────────────────────────────────────────────────────┘
        │ archive_command                    │ pg_stat_*, custom views
        ▼                                    ▼
┌──────────────────────────┐    ┌──────────────────────────────────────────┐
│  Object Storage (S3)       │    │  MONITORING STACK (Step 798)                │
│  - WAL archive              │    │  postgres_exporter → Prometheus → Grafana   │
│  - pgBackRest full/diff      │    │  Alertmanager → PagerDuty/Slack             │
│  (Step 794 — RPO ≤ 60s)     │    │  SLI/SLO: availability, lag, pool sat.      │
└──────────────────────────┘    └──────────────────────────────────────────┘

  SECURITY LAYER (คลุมทุกจุดในภาพ — Step 797):
  TLS ทุกการเชื่อมต่อ | pgaudit บันทึกทุก DDL/role/BYPASSRLS query |
  pgcrypto สำหรับข้อมูลอ่อนไหว | pg_hba.conf บังคับ hostssl + client cert สำหรับ admin

  SCALING ROADMAP (Step 799):
  Phase 1 (<2K tenants): สถาปัตยกรรมข้างต้นทั้งหมด
  Phase 2 (2K-20K):      + partitioning (pg_partman) + vertical scale + tuned autovacuum
  Phase 3 (20K+):        + sharding ตาม tenant_id (Citus / application-level)
```

### 800.3 Checklist ความพร้อมก่อนขึ้นระดับ World-Class

ก่อนไปต่อยัง **05-world-class**, ให้ตรวจสอบว่าตัวเองตอบคำถามเหล่านี้ได้อย่างมั่นใจ (ทำเครื่องหมายในใจ หรือ print ออกมาเช็คจริงก็ได้):

- [ ] อธิบายได้ว่าทำไม RLS ต้องเปิดที่ตารางลูกด้วย ไม่ไหลผ่าน JOIN จากตารางแม่
- [ ] อธิบายความแตกต่างระหว่าง `USING` กับ `WITH CHECK` ใน RLS policy ได้อย่างชัดเจน
- [ ] ออกแบบ role hierarchy ตาม least privilege ให้ทีมงาน 4-5 บทบาทที่ต่างกันได้
- [ ] อธิบายทำไม `FORCE ROW LEVEL SECURITY` จำเป็นเมื่อ table owner กับ app role อาจเป็น role เดียวกัน
- [ ] คำนวณ RPO/RTO จาก config `archive_timeout`, backup schedule ได้
- [ ] อธิบายกลไก failover ของ Patroni ตั้งแต่ leader lock หมดอายุจนถึง promote สำเร็จ
- [ ] อธิบายว่าทำไม synchronous replication ต้อง trade-off กับ write latency
- [ ] อธิบายได้ว่าทำไม `SET LOCAL` (ไม่ใช่ `SET`) จำเป็นเมื่อใช้ transaction pooling ร่วมกับ RLS
- [ ] คำนวณขนาด connection pool ที่เหมาะสมด้วย Little's Law ได้
- [ ] ออกแบบ SLI/SLO และ alert rule ที่ครอบคลุม availability, latency, replication lag, backup health
- [ ] อธิบายเหตุผลที่ต้อง partition ตารางตามเวลา + เก็บ `tenant_id` นำหน้า index เสมอ
- [ ] อธิบายทางเลือกระหว่าง application-level sharding กับ Citus ได้ พร้อมข้อดี/ข้อเสีย
- [ ] อ่าน EXPLAIN ANALYZE ของ query ที่มี RLS แล้วบอกได้ว่า planner push down เงื่อนไข RLS สำเร็จหรือไม่

ถ้าตอบได้ครบทุกข้อ แปลว่าพร้อมสำหรับระดับ **World-Class** ซึ่งจะลงลึกไปถึงระดับ **internal ของ PostgreSQL เอง** (process architecture, storage internals, MVCC internals, WAL internals, query planner internals) ซึ่งเป็นความรู้ที่ทำให้เข้าใจ "ทำไม" เบื้องหลังทุกสิ่งที่เราออกแบบมาตลอดระดับมืออาชีพ

---

## สรุปหลักสูตรระดับมืออาชีพ (Part 061–079)

ตลอดระดับมืออาชีพ เราเดินทางผ่าน 4 กลุ่มความรู้ใหญ่ ซึ่งบทนี้ (Part 080) ได้สังเคราะห์รวมเป็นสถาปัตยกรรมเดียวของ ShopFlow:

**กลุ่มที่ 1 — Resilience (ความทนทาน):** Part 061 Backup Strategies, 062 PITR, 063 Streaming Replication, 064 Logical Replication, 065 High Availability, 079 Disaster Recovery Planning — ทั้งหมดนี้ตอบคำถามเดียวกันคือ **"ถ้าระบบล่ม เราจะกู้กลับมาได้เร็วแค่ไหน และเสียข้อมูลไปเท่าไร"** (RTO/RPO) ซึ่งใน ShopFlow แปลงเป็น Patroni cluster + pgBackRest + PITR runbook (Step 794–795)

**กลุ่มที่ 2 — Scale (การรองรับปริมาณ):** Part 066 Connection Pooling, 067 Load Balancing, 073 Capacity Planning, 074 Partitioning, 075 Scaling Strategies, 076 Query Performance Tuning, 077 Vacuum Tuning, 078 Extensions — ตอบคำถาม **"ระบบจะยังเร็วอยู่ไหมเมื่อข้อมูลและ traffic โตขึ้น 10 เท่า 100 เท่า"** ซึ่งใน ShopFlow แปลงเป็น PgBouncer + partitioning + sharding roadmap (Step 796, 799)

**กลุ่มที่ 3 — Isolation & Access Control (การแบ่งแยกและควบคุมสิทธิ์):** Part 068 Roles & Privileges, 069 Row Level Security — ตอบคำถาม **"ใครควรเห็นอะไรได้บ้าง"** ซึ่งเป็นหัวใจของการทำ multi-tenant SaaS บนฐานข้อมูลเดียว (Step 792–793)

**กลุ่มที่ 4 — Security & Observability (ความปลอดภัยและการมองเห็นระบบ):** Part 070 SSL/TLS, 071 Encryption/Audit, 072 Monitoring — ตอบคำถาม **"เรารู้ได้อย่างไรว่าระบบปลอดภัยและกำลังทำงานถูกต้อง"** (Step 797–798)

ทั้ง 4 กลุ่มนี้ไม่ได้ทำงานแยกกัน — ตัวอย่างที่ชัดเจนที่สุดในบทนี้คือจุดตัดระหว่าง **Pooling (กลุ่ม 2)** กับ **RLS (กลุ่ม 3)** ที่ Step 796.2: การใช้ `SET LOCAL` แทน `SET` ธรรมดา ไม่ใช่แค่รายละเอียดทางเทคนิค แต่คือจุดที่การออกแบบผิดพลาดเพียงจุดเดียวอาจทำให้เกิด **cross-tenant data leak** ระดับร้ายแรงได้ นี่คือเหตุผลที่วิศวกรฐานข้อมูลระดับมืออาชีพต้องเข้าใจภาพรวมทั้งระบบ ไม่ใช่แค่ feature เดียวแบบแยกส่วน

---

## สรุปท้ายบท

บทนี้ไม่ได้สอนฟีเจอร์ใหม่ของ PostgreSQL แต่พา**สังเคราะห์**ความรู้ 19 หัวข้อจากระดับมืออาชีพให้กลายเป็นสถาปัตยกรรม **ShopFlow** — แพลตฟอร์ม SaaS e-commerce แบบ multi-tenant ที่:

1. แยกข้อมูลแต่ละ tenant ด้วย **Row Level Security** ทุกตาราง พร้อม `FORCE ROW LEVEL SECURITY` และ fail-closed design (Step 792)
2. แบ่งสิทธิ์ทีมงานตาม **least privilege** ด้วย 6 role ที่ชัดเจน (Step 793)
3. รับประกัน **RPO ≤ 60 วินาที, RTO ≤ 15 นาที** ด้วย WAL archiving + pgBackRest (Step 794)
4. รักษา **SLA 99.95%** ด้วย Patroni cluster แบบ sync + async standby (Step 795)
5. รองรับ backend หลักร้อย instance ด้วย **PgBouncer transaction pooling** โดยไม่ทำให้ PostgreSQL connection ล้น (Step 796)
6. ป้องกันข้อมูลด้วย **TLS, encryption at rest, pgaudit** ครบทุกชั้น (Step 797)
7. มองเห็นสุขภาพระบบแบบ real-time ด้วย **Prometheus/Grafana + SLI/SLO ที่ชัดเจน** (Step 798)
8. วางแผนรองรับการเติบโตจาก 500 ไปจนถึง 20,000+ tenant ด้วย **partitioning และ sharding roadmap** (Step 799)

ความรู้ทั้งหมดนี้ไม่ใช่จุดจบ แต่คือ**รากฐาน**ที่จะทำให้การเรียนระดับ **World-Class** ในบทถัดไปมีความหมายมากขึ้น เพราะเมื่อเข้าใจ "ทำไม" ของแต่ละการตัดสินใจในสถาปัตยกรรมนี้แล้ว การเจาะลึกไปถึงกลไกภายในของ PostgreSQL เอง (process architecture, storage engine, MVCC, WAL internals) จะช่วยให้ debug และ optimize ระบบระดับนี้ได้อย่างแท้จริง ไม่ใช่แค่ทำตาม best practice โดยไม่เข้าใจเบื้องหลัง

---

## แบบฝึกหัดขยายโปรเจกต์ (10 ข้อ)

**ข้อ 1.** จากสคีมา ShopFlow ปัจจุบัน ให้เพิ่มตาราง `product_reviews` (รีวิวสินค้าจากลูกค้า) ที่มี `tenant_id`, `product_id`, `customer_id`, `rating`, `comment` แล้วเขียน RLS policy ให้ครบทั้ง `FOR ALL` พร้อม `FORCE ROW LEVEL SECURITY`

<details>
<summary>เฉลยข้อ 1</summary>

```sql
CREATE TABLE product_reviews (
    review_id    SERIAL PRIMARY KEY,
    tenant_id    INTEGER NOT NULL REFERENCES tenants(tenant_id),
    product_id   INTEGER NOT NULL REFERENCES products(product_id),
    customer_id  INTEGER NOT NULL REFERENCES customers(customer_id),
    rating       SMALLINT NOT NULL CHECK (rating BETWEEN 1 AND 5),
    comment      TEXT,
    created_at   TIMESTAMPTZ DEFAULT now()
);

CREATE INDEX idx_product_reviews_tenant ON product_reviews (tenant_id, product_id);

ALTER TABLE product_reviews ENABLE ROW LEVEL SECURITY;
ALTER TABLE product_reviews FORCE ROW LEVEL SECURITY;

CREATE POLICY product_reviews_isolation ON product_reviews
    FOR ALL
    USING (tenant_id = current_tenant_id())
    WITH CHECK (tenant_id = current_tenant_id());

ALTER TABLE product_reviews ALTER COLUMN tenant_id SET DEFAULT current_tenant_id();
```
</details>

**ข้อ 2.** เขียน RLS policy แบบละเอียดกว่าเดิมสำหรับ `tenant_users`: อนุญาตให้ `role = 'owner'` เท่านั้นที่ `DELETE` แถวใน `tenant_users` ได้ (สมาชิกทั่วไปดู/แก้ตัวเองได้ แต่ลบคนอื่นไม่ได้)

<details>
<summary>เฉลยข้อ 2</summary>

```sql
-- ลบ policy รวมเดิม แล้วแยกเป็นรายคำสั่งเพื่อควบคุมละเอียดขึ้น
DROP POLICY tenant_users_isolation ON tenant_users;

CREATE POLICY tenant_users_select ON tenant_users
    FOR SELECT USING (tenant_id = current_tenant_id());

CREATE POLICY tenant_users_insert ON tenant_users
    FOR INSERT WITH CHECK (tenant_id = current_tenant_id());

CREATE POLICY tenant_users_update ON tenant_users
    FOR UPDATE
    USING (tenant_id = current_tenant_id())
    WITH CHECK (tenant_id = current_tenant_id());

-- เฉพาะ owner เท่านั้นที่ลบได้ ต้องมีฟังก์ชันตรวจสอบ role ของผู้ใช้ปัจจุบันเพิ่ม
CREATE OR REPLACE FUNCTION current_user_is_owner() RETURNS BOOLEAN AS $$
    SELECT EXISTS (
        SELECT 1 FROM tenant_users
        WHERE tenant_id = current_tenant_id()
          AND user_id = NULLIF(current_setting('app.current_user_id', true), '')::INTEGER
          AND role = 'owner'
    );
$$ LANGUAGE sql STABLE;

CREATE POLICY tenant_users_delete ON tenant_users
    FOR DELETE
    USING (tenant_id = current_tenant_id() AND current_user_is_owner());
```
</details>

**ข้อ 3.** ออกแบบ role ใหม่ชื่อ `shopflow_webhook` สำหรับ service ที่รับ webhook จากผู้ให้บริการชำระเงินภายนอก โดยต้องเขียนได้เฉพาะ `orders.status` (อัปเดตสถานะเป็น `paid`) แต่ห้ามแก้ไขคอลัมน์อื่น และห้าม `INSERT`/`DELETE`

<details>
<summary>เฉลยข้อ 3</summary>

```sql
CREATE ROLE shopflow_webhook WITH LOGIN PASSWORD 'use_a_secret_manager_not_this_string';
GRANT CONNECT ON DATABASE shopflow TO shopflow_webhook;
GRANT USAGE ON SCHEMA public TO shopflow_webhook;

-- column-level privilege: UPDATE ได้เฉพาะคอลัมน์ status
GRANT SELECT (order_id, tenant_id, status) ON orders TO shopflow_webhook;
GRANT UPDATE (status) ON orders TO shopflow_webhook;
-- ไม่ GRANT INSERT, DELETE ให้ role นี้เลย

-- RLS เดิมยังบังคับใช้ต่อ (ต้อง SET LOCAL app.current_tenant ตามปกติ)
```
</details>

**ข้อ 4.** เขียน query สำหรับ `shopflow_reporting` role (ที่มี `BYPASSRLS`) เพื่อดูยอดขายรวมข้าม tenant ทั้งหมด แยกตาม `plan` พร้อมเขียน INSERT log เข้า `cross_tenant_access_log` ก่อน query (เพื่อ audit)

<details>
<summary>เฉลยข้อ 4</summary>

```sql
-- (รันในฐานะ shopflow_reporting)
INSERT INTO cross_tenant_access_log (accessed_by, reason, query_text)
VALUES (current_user, 'executive_revenue_report',
        'SELECT plan, SUM(total_amount) FROM orders JOIN tenants...');

SELECT t.plan,
       count(DISTINCT t.tenant_id) AS tenant_count,
       sum(o.total_amount) AS total_revenue,
       round(avg(o.total_amount), 2) AS avg_order_value
FROM tenants t
JOIN orders o ON o.tenant_id = t.tenant_id
WHERE o.status = 'paid'
GROUP BY t.plan
ORDER BY total_revenue DESC;
```
</details>

**ข้อ 5.** สมมติเกิด incident: engineer รัน `UPDATE products SET unit_price = 0` โดยลืมใส่ `WHERE tenant_id = ...` ทำให้ราคาสินค้าทุก tenant กลายเป็น 0 เมื่อเวลา 10:15:00 วันนี้ ให้เขียนขั้นตอน PITR (แบบ pseudo-command) เพื่อกู้เฉพาะคอลัมน์ `unit_price`

<details>
<summary>เฉลยข้อ 5</summary>

```bash
# 1) หยุดรับ order ใหม่ชั่วคราว (feature flag) เพื่อไม่ให้เกิด order ราคา 0 เพิ่ม
# 2) กู้ backup ไปยัง recovery instance โดย target เวลาก่อนเกิดเหตุ
pgbackrest --stanza=shopflow --type=time \
    --target="2026-09-25 10:14:55+07" \
    --target-action=promote restore

# 3) บน recovery instance: export เฉพาะ product_id, tenant_id, unit_price ที่ถูกต้อง
psql -h recovery-host -d shopflow -c \
  "COPY (SELECT product_id, tenant_id, unit_price FROM products) TO STDOUT WITH CSV" \
  > correct_prices.csv

# 4) เขียนสคริปต์ UPDATE กลับเข้า production โดย join กับไฟล์ที่กู้มา (ใช้ temp table + UPDATE ... FROM)
psql -h production-host -d shopflow <<'SQL'
CREATE TEMP TABLE correct_prices (product_id INT, tenant_id INT, unit_price NUMERIC(10,2));
\copy correct_prices FROM 'correct_prices.csv' WITH CSV
UPDATE products p
SET unit_price = c.unit_price
FROM correct_prices c
WHERE p.product_id = c.product_id AND p.tenant_id = c.tenant_id;
SQL

# 5) ตรวจสอบผลลัพธ์ + เปิดรับ order กลับคืน + เขียน incident postmortem
```
</details>

**ข้อ 6.** อธิบายว่าจะเกิดอะไรขึ้นถ้าตั้ง `synchronous_standby_names = ''` (ว่างเปล่า) ในระบบ ShopFlow และมันจะกระทบ RPO/RTO อย่างไร

<details>
<summary>เฉลยข้อ 6</summary>

หากตั้งค่าเป็นค่าว่าง ระบบจะกลายเป็น **asynchronous replication ล้วน** — primary จะ commit transaction ทันทีโดยไม่รอ ack จาก standby ใดๆ เลย

ผลกระทบ:
- **RPO เปลี่ยนจาก 0 เป็น > 0**: หาก primary ล่มกะทันหันหลัง commit แต่ก่อนที่ WAL จะถูกส่งไปถึง standby ข้อมูล transaction ล่าสุดจะสูญหายไปกับ primary ที่ล่ม (เสี่ยงเกินเป้าหมาย RPO ≤ 5 นาที ถ้าเกิดในจังหวะที่ replication lag สูง)
- **Write latency ลดลง**: primary ไม่ต้องรอ ack จาก standby จึงเร็วขึ้น (trade-off ตรงข้ามกับ RPO)
- สำหรับ ShopFlow ที่มี SLA เข้มงวดด้าน RPO จึงต้องคงการตั้งค่า `synchronous_standby_names = 'ANY 1 (node-2, node-3)'` ไว้ แม้จะแลกกับ latency ที่สูงขึ้นเล็กน้อย เพราะความเสี่ยงข้อมูลหายกระทบความน่าเชื่อถือ (trust) ของ SaaS มากกว่า latency ไม่กี่มิลลิวินาที
</details>

**ข้อ 7.** ออกแบบ PgBouncer config เพิ่มเติมสำหรับ role `shopflow_analyst` ให้แยก pool ต่างหากจาก `shopflow_app` โดยจำกัด `default_pool_size` ไม่ให้เกิน 10 connection (ป้องกัน analyst query หนักแย่ง resource จาก production traffic)

<details>
<summary>เฉลยข้อ 7</summary>

```ini
[databases]
; แยก database alias สำหรับ analyst ให้ชี้ไปที่ read replica และจำกัด pool
shopflow_analytics = host=patroni-replica-vip port=5432 dbname=shopflow \
    pool_size=10 pool_mode=transaction

[pgbouncer]
; ใน userlist.txt กำหนดให้ shopflow_analyst เชื่อมผ่าน database "shopflow_analytics" เท่านั้น
; ไม่ใช่ "shopflow_rw" หรือ "shopflow_ro" ที่ application หลักใช้
; ทำให้ pool ของ analyst แยกขาดจาก pool ของ production traffic โดยสมบูรณ์
```

เหตุผล: การแยก database alias (ไม่ใช่แค่แยก user) ทำให้ PgBouncer สร้าง pool ที่แยกกันจริงในหน่วยความจำ ไม่ปะปนกับ pool ของ `shopflow_app` แม้จะชี้ไปที่ PostgreSQL instance เดียวกัน (ในที่นี้คือ read replica)
</details>

**ข้อ 8.** เขียน Prometheus alert rule เพิ่มเติมสำหรับตรวจจับ "tenant ที่ query volume พุ่งสูงผิดปกติจนอาจเป็น noisy neighbor" โดยใช้ metric จาก `metrics_tenant_activity` view ที่สร้างไว้ใน Step 798

<details>
<summary>เฉลยข้อ 8</summary>

```yaml
groups:
  - name: shopflow_tenant_fairness
    rules:
      - alert: SingleTenantDominatingLoad
        expr: |
          (shopflow_tenant_orders_last_hour
            / on() group_left sum(shopflow_tenant_orders_last_hour)) > 0.5
        for: 10m
        labels: { severity: warning }
        annotations:
          summary: "Tenant {{ $labels.tenant_id }} คิดเป็นสัดส่วนเกิน 50% ของ order ทั้งระบบใน 1 ชม. — เสี่ยง noisy neighbor"
          runbook: "พิจารณาย้าย tenant นี้ไปยัง shard/pool แยก หรือ rate-limit ที่ API gateway"
```

แนวทางแก้เมื่อ alert นี้ทำงาน: ใช้ rate limiting ที่ API gateway ต่อ tenant, หรือถ้าเป็น pattern ที่เกิดซ้ำ พิจารณาย้าย tenant นั้นไปยัง dedicated shard (เชื่อมกับแผน sharding ใน Step 799.5)
</details>

**ข้อ 9.** อธิบายว่าทำไมตาราง `order_items` ในบทนี้จึงมีคอลัมน์ `tenant_id` ของตัวเอง (denormalize) แทนที่จะ join ผ่าน `orders` เพื่อหา tenant — พร้อมเปรียบเทียบข้อดี/ข้อเสีย

<details>
<summary>เฉลยข้อ 9</summary>

**ข้อดีของการ denormalize `tenant_id` ลงใน `order_items` โดยตรง:**
- RLS policy ของ `order_items` เช็คเงื่อนไขได้ทันทีจาก column ของตัวเอง (`tenant_id = current_tenant_id()`) โดยไม่ต้อง subquery ไป `orders` ก่อน ทำให้ planner ใช้ index `idx_order_items_tenant` ได้ตรงๆ เร็วกว่ามาก โดยเฉพาะเมื่อข้อมูลโตหลักล้านแถว
- ป้องกันกรณี `orders` ถูก partition (Step 799) แล้วการ join ข้าม partition boundary ทำให้ planner วางแผนได้แย่ลง

**ข้อเสีย:**
- เสี่ยงข้อมูลไม่ตรงกัน (data inconsistency) ถ้า `orders.tenant_id` ถูกแก้ไขแต่ `order_items.tenant_id` ไม่ได้อัปเดตตาม (ในทางปฏิบัติ `tenant_id` ไม่ควรเปลี่ยนได้เลยหลังสร้างแถว จึงมักจัดการความเสี่ยงนี้ด้วย `CHECK` constraint หรือ trigger ป้องกันการแก้ไข `tenant_id`)
- เพิ่ม storage เล็กน้อย (คอลัมน์ integer เพิ่มต่อแถว) ซึ่งเทียบไม่ได้กับ performance ที่ได้กลับมา

โดยรวมแล้ว trade-off นี้คุ้มค่าสำหรับระบบ multi-tenant ที่ RLS ต้องทำงานบนทุก query
</details>

**ข้อ 10.** ออกแบบแผน migration ขั้นสูง: เมื่อ ShopFlow ตัดสินใจ shard tenant_id ที่เป็นเลขคู่ไปอยู่ shard ใหม่ (Shard B) โดยไม่ downtime ให้ระบุขั้นตอนคร่าวๆ โดยใช้ความรู้จาก Part 064 (Logical Replication)

<details>
<summary>เฉลยข้อ 10</summary>

```text
ขั้นตอน zero-downtime tenant migration ไปยัง Shard B (ใช้ Logical Replication, Part 064):

1. เตรียม Shard B (PostgreSQL cluster ใหม่ พร้อม schema, roles, RLS เหมือน Shard A ทุกประการ)

2. สร้าง PUBLICATION บน Shard A ที่กรองเฉพาะแถวของ tenant ที่จะย้าย (row filter, PG15+):
   CREATE PUBLICATION migrate_even_tenants
     FOR TABLE orders, order_items, products, customers, tenant_users, tenants
     WHERE (tenant_id % 2 = 0);

3. สร้าง SUBSCRIPTION บน Shard B เพื่อดึงข้อมูลมาแบบ streaming (initial copy + ongoing changes):
   CREATE SUBSCRIPTION migrate_even_tenants_sub
     CONNECTION 'host=shard-a dbname=shopflow ...'
     PUBLICATION migrate_even_tenants;

4. รอจน initial sync เสร็จ แล้ว monitor replication lag ให้ใกล้ 0
   (ตรวจผ่าน pg_stat_subscription เหมือนที่เรียนใน Part 064)

5. เมื่อ lag ใกล้ศูนย์ → เข้าสู่ maintenance window สั้นๆ (วินาทีถึงไม่กี่นาที):
   a. หยุด write ของ tenant กลุ่มนี้ที่ API gateway (feature flag เฉพาะ tenant_id % 2 = 0)
   b. รอ replication lag = 0 จริง (final catch-up)
   c. เปลี่ยน routing ที่ Shard Router ให้ tenant กลุ่มนี้ชี้ไปยัง Shard B
   d. เปิด write กลับคืนที่ Shard B

6. ยกเลิก subscription/publication แล้วลบข้อมูล tenant กลุ่มนี้ออกจาก Shard A
   (ผ่าน DELETE ที่กรองด้วย tenant_id % 2 = 0 ภายใต้ RLS-aware script)

7. อัปเดต monitoring, backup schedule, HA config ให้ Shard B เป็นระบบสมบูรณ์เหมือน Shard A
```

ขั้นตอนนี้ใช้ **Logical Replication (Part 064)** เป็นแกนหลัก เพราะสามารถ copy ข้อมูลเริ่มต้น + stream การเปลี่ยนแปลงต่อเนื่องได้พร้อมกัน ทำให้ downtime ที่ต้องหยุด write เหลือเพียงไม่กี่วินาทีตอน cutover เท่านั้น แทนที่จะต้องหยุดทั้งระบบเพื่อ dump/restore แบบ full
</details>

---

## ก้าวต่อไป: ระดับ World-Class

ยินดีด้วย — คุณได้จบระดับ **มืออาชีพ (Professional)** ของหลักสูตร PostgreSQL ฉบับสมบูรณ์แล้ว ความรู้ตลอด 20 บทที่ผ่านมา (Part 061–080) ได้ประกอบกันเป็นสถาปัตยกรรม production-grade ที่สมบูรณ์แบบหนึ่งระบบ

ระดับถัดไปคือ **World-Class / ผู้เชี่ยวชาญ** ซึ่งจะพาไปเข้าใจ**เบื้องหลัง**ของทุกสิ่งที่เราออกแบบมาในบทนี้ — ทำไม MVCC ถึงทำงานแบบนี้, process architecture ของ PostgreSQL เป็นอย่างไรเวลามี connection นับพัน, WAL ถูกเขียนและ replay อย่างไรในระดับ byte — เริ่มต้นที่:

**[Part 081: Internals — Process Architecture](../05-world-class/part-081-internals-process-architecture.md)**
