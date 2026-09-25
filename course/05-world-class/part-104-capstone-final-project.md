# Part 104: Capstone Project — สร้างระบบ Full-Stack Production-Ready ด้วย PostgreSQL

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับโลก/ผู้เชี่ยวชาญ | Part 104 (Capstone Project สุดท้าย — Step 1000)

---

นี่คือ **Step 1000** — ก้าวสุดท้ายของหลักสูตร PostgreSQL ฉบับสมบูรณ์ทั้ง 1000 ขั้นตอน จาก 104 Part ที่เดินทางมาด้วยกันตั้งแต่คำสั่ง `SELECT` แรกในชีวิตไปจนถึงการออกแบบสถาปัตยกรรมระดับ world-class เราจะไม่สอนหัวข้อใหม่ในบทนี้ แต่จะ **หลอมรวม** ทุกสิ่งที่เรียนมาให้กลายเป็นระบบเดียว ระบบที่ใช้งานได้จริง ระบบที่บริษัทระดับโลกใช้ฐานข้อมูล PostgreSQL แบบเดียวกันนี้ในการรันธุรกิจจริง

โปรเจกต์นี้ชื่อ **"ShopFlow"** — แพลตฟอร์ม SaaS E-Commerce แบบ Multi-Tenant ที่ร้านค้าหลายพันร้านสามารถมาเปิดร้าน ขายสินค้า จัดการสต๊อก รับออเดอร์ รับเงิน และดูรายงานยอดขายได้บนโครงสร้างฐานข้อมูลเดียวกัน

## เป้าหมายการเรียนรู้

- มองเห็นภาพรวมทั้งหมดของหลักสูตร และเข้าใจว่าแต่ละ Level (Foundations → Intermediate → Advanced → Professional → World-Class) เชื่อมโยงกันอย่างไรในงานจริง
- ออกแบบ schema ระดับ production ที่รองรับ multi-tenancy, full-text search, JSONB attributes, และ data integrity แบบเข้มงวด
- ใช้ Row Level Security (RLS) และ Role-based Access Control (RBAC) เพื่อแยก tenant และจำกัดสิทธิ์ผู้ใช้แต่ละบทบาท
- ออกแบบ indexing strategy ที่ครอบคลุมทั้ง B-Tree, GIN, partial index และ composite index
- เขียน business logic ด้วย PL/pgSQL สำหรับ checkout, inventory, และ audit trail ที่ปลอดภัยต่อ concurrency
- ออกแบบ partitioning strategy สำหรับตารางที่ข้อมูลโตเร็วอย่าง `orders`
- สร้าง Views และ Materialized Views สำหรับ dashboard วิเคราะห์ธุรกิจ
- วิเคราะห์และปรับแต่ง query ที่สำคัญด้วย `EXPLAIN (ANALYZE, BUFFERS)`
- วางแผน High Availability, Backup และ Disaster Recovery ระดับ production
- เชื่อมต่อฐานข้อมูลจากแอปพลิเคชันจริงด้วย Node.js และ Python
- วางแผน Monitoring, Observability และ Deployment Architecture (Docker/Kubernetes/Cloud)
- ปิดหลักสูตรทั้ง 1000 Step ด้วยความเข้าใจที่ลึกซึ้งและพร้อมนำไปใช้งานจริงในฐานะ PostgreSQL Expert ระดับโลก

---

## ส่วนที่ 1: ภาพรวมโปรเจกต์สุดท้าย

### 1.1 โจทย์ธุรกิจ: ShopFlow คืออะไร

ShopFlow เป็นแพลตฟอร์ม SaaS E-Commerce ที่ให้ผู้ประกอบการ (tenant) หลายรายมาเปิดร้านค้าออนไลน์ของตัวเองบนโครงสร้างพื้นฐานเดียวกัน (Multi-Tenant Architecture) โดยระบบต้องรองรับความต้องการระดับ production ดังนี้:

**Functional Requirements**

1. **Multi-Tenancy** — แต่ละร้านค้า (tenant) มีข้อมูลสินค้า ลูกค้า ออเดอร์ แยกจากกันอย่างเด็ดขาด ห้ามข้อมูลรั่วไหลข้าม tenant แม้แต่บรรทัดเดียว
2. **Catalog Management** — จัดการหมวดหมู่สินค้าแบบลำดับชั้น (hierarchical categories), สินค้าที่มี attribute ยืดหยุ่น (เช่น เสื้อผ้ามีไซส์/สี, อิเล็กทรอนิกส์มีสเปกต่างกัน) และรองรับการค้นหาสินค้าด้วย full-text search และ tag
3. **Inventory Management** — ติดตามสต๊อกสินค้าแบบ real-time ป้องกันการขายเกินสต๊อก (overselling) แม้มีคำสั่งซื้อพร้อมกันจำนวนมาก (concurrency safety)
4. **Order & Checkout** — กระบวนการสั่งซื้อที่ atomic: ตัดสต๊อก สร้างออเดอร์ บันทึกการชำระเงิน ต้องสำเร็จหรือ rollback พร้อมกันทั้งหมด
5. **Payments** — รองรับการชำระเงินหลายช่องทาง พร้อมสถานะที่ตรวจสอบย้อนกลับได้
6. **Reviews & Ratings** — ลูกค้ารีวิวสินค้าได้เฉพาะสินค้าที่ตนซื้อจริง
7. **Employee & Role Management** — พนักงานร้านค้ามีสิทธิ์ต่างกัน (owner, manager, staff) ตาม RBAC
8. **Reporting & Analytics** — dashboard สรุปยอดขาย สินค้าขายดี แนวโน้มรายเดือน สำหรับผู้บริหารร้านค้า
9. **Audit Trail** — ทุกการเปลี่ยนแปลงข้อมูลสำคัญต้องถูกบันทึกเพื่อ compliance และ debugging

**Non-Functional Requirements**

- รองรับหลายล้าน order ต่อปี โดยไม่มี performance degradation (Partitioning)
- Availability 99.95% ขึ้นไป (High Availability + Replication)
- Recovery Point Objective (RPO) < 5 นาที, Recovery Time Objective (RTO) < 15 นาที (Backup/DR)
- รองรับการ scale แนวนอนด้วย read replica สำหรับ reporting workload
- Query สำคัญ (product search, order lookup) ต้องตอบสนองภายใน < 50ms ที่ p95

### 1.2 เส้นทางการเรียนรู้ทั้งหมด: จาก Part 001 ถึง Part 103

ก่อนจะลงมือสร้างระบบนี้ เรามาย้อนดูกันว่าตลอด 103 Part ที่ผ่านมา เราได้เรียนอะไรมาบ้าง และแต่ละหัวข้อจะถูกนำมาใช้จริงตรงไหนใน ShopFlow

| Level | Part | หัวข้อหลักที่เรียน | ใช้ตรงไหนใน ShopFlow (บทนี้) |
|---|---|---|---|
| **Foundations** | 001-005 | ติดตั้ง PostgreSQL, `psql`, concept ของ RDBMS, การสร้าง database/schema | ส่วนที่ 2 — สร้าง `shopflow` schema |
| Foundations | 006-010 | `CREATE TABLE`, data types, `PRIMARY KEY`, `NOT NULL`, `CHECK` | ส่วนที่ 2 — constraint ทุกตารางหลัก |
| Foundations | 011-015 | `SELECT`, `WHERE`, `ORDER BY`, `JOIN` พื้นฐาน, aggregate function | ส่วนที่ 8 — query รายงานพื้นฐาน |
| Foundations | 016-020 | `INSERT`/`UPDATE`/`DELETE`, transaction พื้นฐาน (`BEGIN`/`COMMIT`) | ส่วนที่ 5 — checkout transaction |
| **Intermediate** | 021-026 | Foreign Key, Referential Integrity, `ON DELETE CASCADE` | ส่วนที่ 2 — ความสัมพันธ์ระหว่างตาราง |
| Intermediate | 027-031 | Subquery, CTE (`WITH`), Window Function เบื้องต้น | ส่วนที่ 7-8 — reporting views |
| Intermediate | 032-036 | Index พื้นฐาน (B-Tree), `EXPLAIN` เบื้องต้น | ส่วนที่ 4, 8 |
| Intermediate | 037-040 | View, Stored Procedure เบื้องต้น, `pg_dump`/`pg_restore` | ส่วนที่ 7, 9 |
| **Advanced** | 041-046 | PL/pgSQL functions, control flow, exception handling | ส่วนที่ 5 — ฟังก์ชัน checkout/inventory |
| Advanced | 047-051 | Trigger, `NEW`/`OLD`, event trigger | ส่วนที่ 5 — audit trail trigger |
| Advanced | 052-056 | Advanced Window Function, `LATERAL`, Recursive CTE | ส่วนที่ 2 (category tree), ส่วนที่ 7 (dashboard) |
| Advanced | 057-060 | JSON/JSONB, Array type, Full-Text Search (`tsvector`) | ส่วนที่ 2, 4 — product attributes & search |
| **Professional** | 061-065 | Replication (streaming, logical), High Availability | ส่วนที่ 9 |
| Professional | 066-070 | Backup strategy (`pg_basebackup`, WAL archiving, PITR) | ส่วนที่ 9 |
| Professional | 071-075 | Monitoring (`pg_stat_statements`, `pg_stat_activity`), Observability | ส่วนที่ 11 |
| Professional | 076-080 | Security hardening, `pg_hba.conf`, SSL, Row Level Security | ส่วนที่ 3 |
| **World-Class** | 081-086 | Query Optimizer ภายใน, Advanced EXPLAIN, Query Tuning | ส่วนที่ 8 |
| World-Class | 087-091 | Connection Pooling (PgBouncer), Application Integration (ORMs, driver) | ส่วนที่ 10 |
| World-Class | 092-096 | Event Sourcing/CQRS, Docker/Kubernetes, Cloud (AWS/GCP/Azure) | ส่วนที่ 12 |
| World-Class | 097-103 | Partitioning ขั้นสูง, Sharding, Extension ecosystem, Performance at scale | ส่วนที่ 6, 8 |

จะเห็นได้ว่าทุกบรรทัดของ SQL ในบทนี้ **ไม่มีอะไรใหม่เลย** — ทุกเทคนิคเคยเรียนมาแล้วทั้งสิ้น สิ่งที่บทนี้ทำคือการเอาทุกชิ้นส่วนมาประกอบกันเป็นระบบที่สมบูรณ์ เหมือนวิศวกรที่เรียนจบทุกวิชาแล้วมาสร้างสะพานจริงเป็นครั้งแรก

### 1.3 สถาปัตยกรรมระดับสูงของระบบ

```
┌──────────────────────────────────────────────────────────────────┐
│                         ShopFlow Platform                        │
├──────────────┬──────────────┬──────────────┬─────────────────────┤
│  Storefront   │  Admin Panel │   Mobile App  │   Partner API       │
│  (Next.js)    │  (React)     │   (RN)        │   (REST/GraphQL)    │
└──────┬───────┴──────┬───────┴──────┬───────┴──────────┬──────────┘
       │              │              │                   │
       └──────────────┴──────┬───────┴───────────────────┘
                              │
                    ┌─────────▼─────────┐
                    │  Application Tier  │   Node.js / Python
                    │  (Connection Pool  │   (Prisma / SQLAlchemy)
                    │   via PgBouncer)   │
                    └─────────┬──────────┘
                              │
              ┌───────────────┼────────────────┐
              │                                 │
    ┌─────────▼─────────┐            ┌──────────▼──────────┐
    │  PostgreSQL Primary │  streaming │  Read Replicas (x2+) │
    │  (Read/Write)        │─────────▶ │  (Reporting/Analytics)│
    └─────────┬────────────┘  replication└──────────────────────┘
              │
    ┌─────────▼─────────┐
    │  WAL Archive + PITR │  →  Object Storage (S3/GCS)
    │  (Backup/DR)         │
    └────────────────────┘
```

ในส่วนที่เหลือของบทนี้ เราจะสร้างทุกชิ้นส่วนของฐานข้อมูลนี้ขึ้นมาจริง ตั้งแต่บรรทัดแรกจนถึงบรรทัดสุดท้าย

---

## ส่วนที่ 2: Database Schema แบบสมบูรณ์

### 2.1 การเตรียม Database และ Extension

```sql
-- สร้าง database สำหรับโปรเจกต์ (รันจาก psql ในฐานะ superuser หรือผู้ใช้ที่มีสิทธิ์ CREATEDB)
CREATE DATABASE shopflow
    WITH ENCODING 'UTF8'
    LC_COLLATE 'en_US.UTF-8'
    LC_CTYPE 'en_US.UTF-8'
    TEMPLATE template0;

\c shopflow

-- สร้าง schema แยกให้เป็นระเบียบ ไม่ปะปนกับ public schema
CREATE SCHEMA IF NOT EXISTS shopflow;
SET search_path TO shopflow, public;

-- Extension ที่จำเป็นสำหรับระบบระดับ production
CREATE EXTENSION IF NOT EXISTS pgcrypto;      -- gen_random_uuid(), การเข้ารหัสรหัสผ่าน
CREATE EXTENSION IF NOT EXISTS pg_trgm;       -- trigram index สำหรับ fuzzy search / ILIKE
CREATE EXTENSION IF NOT EXISTS btree_gin;     -- รวม B-Tree column เข้ากับ GIN index
CREATE EXTENSION IF NOT EXISTS pg_stat_statements; -- ติดตาม query performance (ส่วนที่ 8, 11)
```

**เหตุผลของแต่ละ extension:**

- `pgcrypto` — ใช้ `gen_random_uuid()` สร้าง UUID สำหรับ public-facing identifier (เช่น order tracking number) และเข้ารหัสรหัสผ่านพนักงาน/ลูกค้าด้วย `crypt()`
- `pg_trgm` — รองรับการค้นหาสินค้าแบบพิมพ์ผิดได้บ้าง (fuzzy search) และเร่งความเร็ว `ILIKE '%คำค้น%'`
- `btree_gin` — ทำให้สร้าง composite GIN index ที่รวมทั้งคอลัมน์ scalar (เช่น `tenant_id`) กับคอลัมน์ JSONB/array ในดัชนีเดียวได้
- `pg_stat_statements` — เก็บสถิติ query ทั้งหมดที่รันบน server เพื่อวิเคราะห์ performance (ใช้ในส่วนที่ 8 และ 11)

### 2.2 Domain Types และ ENUM Types

การกำหนด Domain และ ENUM ตั้งแต่ต้นช่วยให้ constraint ถูกบังคับใช้ที่ระดับ type system แทนที่จะพึ่งพา application logic เพียงอย่างเดียว

```sql
SET search_path TO shopflow, public;

-- ===== Domain Types: บังคับรูปแบบข้อมูลระดับ column type =====

-- อีเมลต้องมีรูปแบบถูกต้องเสมอ ไม่ว่าจะ INSERT จากที่ไหนก็ตาม
CREATE DOMAIN email_address AS citext
    CONSTRAINT email_format_check
    CHECK ( VALUE ~ '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$' );

-- จำนวนเงินต้องไม่ติดลบ และมีทศนิยม 2 ตำแหน่งสอดคล้องกับสกุลเงิน
CREATE DOMAIN money_amount AS numeric(14,2)
    CONSTRAINT money_amount_non_negative CHECK ( VALUE >= 0 );

-- จำนวนสต๊อก/ปริมาณต้องไม่ติดลบ
CREATE DOMAIN non_negative_int AS integer
    CONSTRAINT non_negative_int_check CHECK ( VALUE >= 0 );

-- เปอร์เซ็นต์ส่วนลด ต้องอยู่ระหว่าง 0-100
CREATE DOMAIN percentage AS numeric(5,2)
    CONSTRAINT percentage_range_check CHECK ( VALUE BETWEEN 0 AND 100 );

-- รหัส slug สำหรับ URL (lowercase, ตัวเลข, ขีดกลาง เท่านั้น)
CREATE DOMAIN url_slug AS text
    CONSTRAINT url_slug_format_check CHECK ( VALUE ~ '^[a-z0-9]+(-[a-z0-9]+)*$' );

-- ต้องเปิดใช้ citext ก่อนใช้ email_address domain ด้านบน
CREATE EXTENSION IF NOT EXISTS citext;

-- ===== ENUM Types: สถานะที่จำกัด และรู้ค่าล่วงหน้าแน่นอน =====

CREATE TYPE subscription_plan AS ENUM ('free', 'starter', 'growth', 'enterprise');
CREATE TYPE tenant_status AS ENUM ('trial', 'active', 'suspended', 'cancelled');

CREATE TYPE employee_role AS ENUM ('owner', 'admin', 'manager', 'staff', 'support');
CREATE TYPE account_status AS ENUM ('active', 'invited', 'disabled');

CREATE TYPE product_status AS ENUM ('draft', 'active', 'archived', 'out_of_stock');

CREATE TYPE order_status AS ENUM (
    'pending', 'awaiting_payment', 'paid', 'processing',
    'shipped', 'delivered', 'completed', 'cancelled', 'refunded'
);

CREATE TYPE payment_method AS ENUM ('credit_card', 'bank_transfer', 'promptpay', 'cod', 'wallet');
CREATE TYPE payment_status AS ENUM ('pending', 'authorized', 'captured', 'failed', 'refunded', 'partially_refunded');

CREATE TYPE shipment_status AS ENUM ('pending', 'packed', 'shipped', 'in_transit', 'delivered', 'returned');

CREATE TYPE review_status AS ENUM ('pending', 'approved', 'rejected');

CREATE TYPE inventory_movement_type AS ENUM (
    'stock_in', 'stock_out', 'sale', 'return', 'adjustment', 'reserved', 'released'
);

CREATE TYPE audit_action AS ENUM ('INSERT', 'UPDATE', 'DELETE');
```

**เหตุผลการออกแบบ:**

- ใช้ **Domain** สำหรับค่าที่มีกฎ validation ซ้ำ ๆ ในหลายตาราง (เช่น `money_amount` ถูกใช้ในทั้ง `products`, `order_items`, `payments`) แทนที่จะเขียน `CHECK` ซ้ำทุกที่ — แก้ที่เดียว มีผลทุกตาราง
- ใช้ **ENUM** แทน foreign key ไปยังตาราง lookup เล็ก ๆ ที่ค่าคงที่และเปลี่ยนไม่บ่อย (สถานะออเดอร์, บทบาทพนักงาน) เพราะเร็วกว่า, กิน storage น้อยกว่า, และบังคับความถูกต้องที่ database level
- `citext` ใช้กับอีเมลเพื่อให้ `Test@Mail.com` และ `test@mail.com` ถือเป็นค่าเดียวกันโดยอัตโนมัติ ป้องกันลูกค้าสมัครซ้ำด้วย case ต่างกัน

### 2.3 ตาราง Tenants (หัวใจของ Multi-Tenancy)

```sql
CREATE TABLE tenants (
    tenant_id       bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    public_id       uuid NOT NULL DEFAULT gen_random_uuid() UNIQUE,
    store_name      text NOT NULL,
    store_slug      url_slug NOT NULL UNIQUE,
    plan            subscription_plan NOT NULL DEFAULT 'trial'::subscription_plan,
    status          tenant_status NOT NULL DEFAULT 'trial',
    contact_email   email_address NOT NULL,
    default_currency char(3) NOT NULL DEFAULT 'THB',
    timezone        text NOT NULL DEFAULT 'Asia/Bangkok',
    created_at      timestamptz NOT NULL DEFAULT now(),
    updated_at      timestamptz NOT NULL DEFAULT now(),
    trial_ends_at   timestamptz,

    CONSTRAINT tenants_currency_iso_check CHECK ( default_currency ~ '^[A-Z]{3}$' )
);

COMMENT ON TABLE tenants IS 'ร้านค้าแต่ละร้านบนแพลตฟอร์ม ShopFlow — root ของ multi-tenancy ทั้งหมด';
COMMENT ON COLUMN tenants.public_id IS 'UUID สำหรับอ้างอิงจากภายนอก/URL แทนการเปิดเผย tenant_id ที่เป็น sequential integer';

-- trigger เดียวที่ใช้ซ้ำได้ทุกตารางเพื่ออัปเดต updated_at อัตโนมัติ
CREATE OR REPLACE FUNCTION shopflow.set_updated_at()
RETURNS trigger
LANGUAGE plpgsql
AS $$
BEGIN
    NEW.updated_at := now();
    RETURN NEW;
END;
$$;

CREATE TRIGGER trg_tenants_updated_at
    BEFORE UPDATE ON tenants
    FOR EACH ROW
    EXECUTE FUNCTION shopflow.set_updated_at();
```

**เหตุผลการออกแบบ:**

- `tenant_id` เป็น `bigint IDENTITY` ใช้เป็น internal primary key เพื่อ join ที่รวดเร็วและประหยัด storage (8 bytes เทียบกับ UUID 16 bytes) ส่วน `public_id` (UUID) ใช้เปิดเผยใน API/URL เพื่อไม่ให้คู่แข่งเดาจำนวนร้านค้าทั้งหมดในระบบได้จากเลขที่รันต่อเนื่อง
- `store_slug` ใช้ domain `url_slug` เพื่อบังคับรูปแบบ URL-friendly ตั้งแต่ database level
- แยก pattern `set_updated_at()` เป็นฟังก์ชันกลางที่ทุกตารางเรียกใช้ซ้ำได้ — DRY principle ระดับ database

### 2.4 ตาราง Employees, Customers และ RBAC พื้นฐาน

```sql
-- พนักงานของแต่ละร้านค้า (ผู้ดูแลระบบหลังบ้าน)
CREATE TABLE employees (
    employee_id     bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    tenant_id       bigint NOT NULL REFERENCES tenants(tenant_id) ON DELETE CASCADE,
    email           email_address NOT NULL,
    password_hash   text NOT NULL,
    full_name       text NOT NULL,
    role            employee_role NOT NULL DEFAULT 'staff',
    status          account_status NOT NULL DEFAULT 'invited',
    created_at      timestamptz NOT NULL DEFAULT now(),
    updated_at      timestamptz NOT NULL DEFAULT now(),
    last_login_at   timestamptz,

    -- อีเมลต้องไม่ซ้ำ "ภายในร้านค้าเดียวกัน" แต่ต่างร้านค้าใช้อีเมลเดียวกันได้
    CONSTRAINT employees_tenant_email_uk UNIQUE (tenant_id, email)
);

CREATE TRIGGER trg_employees_updated_at
    BEFORE UPDATE ON employees
    FOR EACH ROW EXECUTE FUNCTION shopflow.set_updated_at();

-- ลูกค้าที่ซื้อสินค้า (ผูกกับ tenant เดียว — ลูกค้าของร้าน A ไม่ใช่ลูกค้าของร้าน B)
CREATE TABLE customers (
    customer_id     bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    tenant_id       bigint NOT NULL REFERENCES tenants(tenant_id) ON DELETE CASCADE,
    public_id       uuid NOT NULL DEFAULT gen_random_uuid() UNIQUE,
    email           email_address NOT NULL,
    password_hash   text,
    full_name       text NOT NULL,
    phone           text,
    marketing_opt_in boolean NOT NULL DEFAULT false,
    created_at      timestamptz NOT NULL DEFAULT now(),
    updated_at      timestamptz NOT NULL DEFAULT now(),

    CONSTRAINT customers_tenant_email_uk UNIQUE (tenant_id, email)
);

CREATE TRIGGER trg_customers_updated_at
    BEFORE UPDATE ON customers
    FOR EACH ROW EXECUTE FUNCTION shopflow.set_updated_at();

-- ที่อยู่จัดส่ง/เรียกเก็บเงินของลูกค้า (ลูกค้าหนึ่งคนมีได้หลายที่อยู่)
CREATE TABLE customer_addresses (
    address_id      bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    customer_id     bigint NOT NULL REFERENCES customers(customer_id) ON DELETE CASCADE,
    tenant_id       bigint NOT NULL REFERENCES tenants(tenant_id) ON DELETE CASCADE,
    label           text NOT NULL DEFAULT 'Home',
    recipient_name  text NOT NULL,
    phone           text NOT NULL,
    line1           text NOT NULL,
    line2           text,
    city            text NOT NULL,
    postal_code     text NOT NULL,
    country_code    char(2) NOT NULL DEFAULT 'TH',
    is_default      boolean NOT NULL DEFAULT false,
    created_at      timestamptz NOT NULL DEFAULT now()
);

-- บังคับให้ลูกค้าแต่ละคนมี default address ได้แค่ 1 รายการ
CREATE UNIQUE INDEX customer_addresses_one_default_uk
    ON customer_addresses (customer_id)
    WHERE is_default;
```

**เหตุผลการออกแบบ:**

- ทั้ง `employees` และ `customers` มี `tenant_id` เป็น FK ตรง ๆ (ไม่ใช่ derived ผ่านตารางอื่น) เพราะ RLS policy (ส่วนที่ 3) ต้องอ้างอิงคอลัมน์นี้ได้โดยตรงในทุกตารางที่ต้องแยก tenant
- `UNIQUE (tenant_id, email)` แทนที่จะเป็น `UNIQUE (email)` เฉย ๆ — เพราะอีเมลเดียวกันสามารถเป็นลูกค้าของหลายร้านค้าที่ต่างกันได้ นี่คือหัวใจสำคัญของการออกแบบ multi-tenant schema ที่ผิดพลาดบ่อยที่สุดถ้าลืมใส่ `tenant_id` เข้าไปใน unique constraint
- Partial unique index `WHERE is_default` เป็นเทคนิคจาก Part 032-036 ที่ใช้บังคับ business rule "มีได้อย่างมากหนึ่งแถว" โดยไม่ต้องใช้ trigger

### 2.5 ตาราง Categories (Hierarchical) และ Suppliers

```sql
-- หมวดหมู่สินค้าแบบลำดับชั้น (adjacency list model — เรียนใน Part 052-056 เรื่อง Recursive CTE)
CREATE TABLE categories (
    category_id     bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    tenant_id       bigint NOT NULL REFERENCES tenants(tenant_id) ON DELETE CASCADE,
    parent_id       bigint REFERENCES categories(category_id) ON DELETE SET NULL,
    name            text NOT NULL,
    slug            url_slug NOT NULL,
    description     text,
    display_order   integer NOT NULL DEFAULT 0,
    created_at      timestamptz NOT NULL DEFAULT now(),

    CONSTRAINT categories_tenant_slug_uk UNIQUE (tenant_id, slug),
    -- ป้องกันหมวดหมู่อ้างอิงตัวเองเป็น parent
    CONSTRAINT categories_no_self_parent CHECK ( parent_id IS DISTINCT FROM category_id )
);

CREATE INDEX idx_categories_parent ON categories (parent_id);
CREATE INDEX idx_categories_tenant ON categories (tenant_id);

-- ซัพพลายเออร์ที่ร้านค้าสั่งซื้อสินค้าเข้าสต๊อก
CREATE TABLE suppliers (
    supplier_id     bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    tenant_id       bigint NOT NULL REFERENCES tenants(tenant_id) ON DELETE CASCADE,
    name            text NOT NULL,
    contact_email   email_address,
    contact_phone   text,
    lead_time_days  non_negative_int NOT NULL DEFAULT 7,
    is_active       boolean NOT NULL DEFAULT true,
    created_at      timestamptz NOT NULL DEFAULT now(),

    CONSTRAINT suppliers_tenant_name_uk UNIQUE (tenant_id, name)
);
```

### 2.6 ตาราง Products (หัวใจของ Catalog) — JSONB + Full-Text Search + Tags

นี่คือตารางที่รวมเทคนิคจากหลาย Part เข้าด้วยกันมากที่สุด: JSONB attributes (Part 057-060), generated `tsvector` column สำหรับ full-text search, และ `text[]` array สำหรับ tags

```sql
CREATE TABLE products (
    product_id      bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    tenant_id       bigint NOT NULL REFERENCES tenants(tenant_id) ON DELETE CASCADE,
    category_id     bigint REFERENCES categories(category_id) ON DELETE SET NULL,
    supplier_id     bigint REFERENCES suppliers(supplier_id) ON DELETE SET NULL,
    public_id       uuid NOT NULL DEFAULT gen_random_uuid() UNIQUE,

    sku             text NOT NULL,
    name            text NOT NULL,
    slug            url_slug NOT NULL,
    description     text,

    -- ราคาปกติ และราคาลด (ต้องไม่มากกว่าราคาปกติ)
    price           money_amount NOT NULL,
    compare_at_price money_amount,

    -- ยืดหยุ่นสำหรับ attribute ที่แตกต่างกันตามประเภทสินค้า เช่น {"color":"red","size":"M","material":"cotton"}
    attributes      jsonb NOT NULL DEFAULT '{}'::jsonb,

    -- tag ค้นหา/กรองสินค้า เช่น {'summer','new-arrival','bestseller'}
    tags            text[] NOT NULL DEFAULT '{}',

    stock_quantity  non_negative_int NOT NULL DEFAULT 0,
    reserved_quantity non_negative_int NOT NULL DEFAULT 0,
    reorder_threshold non_negative_int NOT NULL DEFAULT 5,

    status          product_status NOT NULL DEFAULT 'draft',
    weight_grams    non_negative_int,

    -- Generated column: full-text search vector รวม name + description
    -- น้ำหนัก 'A' ให้ชื่อสินค้า, 'B' ให้คำอธิบาย — ชื่อสินค้าจะมีผลต่อ ranking มากกว่า
    search_vector   tsvector GENERATED ALWAYS AS (
        setweight(to_tsvector('simple', coalesce(name, '')), 'A') ||
        setweight(to_tsvector('simple', coalesce(description, '')), 'B')
    ) STORED,

    created_at      timestamptz NOT NULL DEFAULT now(),
    updated_at      timestamptz NOT NULL DEFAULT now(),

    CONSTRAINT products_tenant_sku_uk UNIQUE (tenant_id, sku),
    CONSTRAINT products_tenant_slug_uk UNIQUE (tenant_id, slug),
    CONSTRAINT products_compare_price_check
        CHECK (compare_at_price IS NULL OR compare_at_price >= price),
    CONSTRAINT products_reserved_not_exceed_stock
        CHECK (reserved_quantity <= stock_quantity),
    CONSTRAINT products_attributes_is_object
        CHECK (jsonb_typeof(attributes) = 'object')
);

CREATE TRIGGER trg_products_updated_at
    BEFORE UPDATE ON products
    FOR EACH ROW EXECUTE FUNCTION shopflow.set_updated_at();

COMMENT ON COLUMN products.attributes IS
    'JSONB attribute ที่ยืดหยุ่นตามประเภทสินค้า เช่น {"color":"red","size":"M"} — ค้นหาด้วย GIN index ในส่วนที่ 4';
COMMENT ON COLUMN products.search_vector IS
    'สร้างอัตโนมัติจาก name (น้ำหนัก A) และ description (น้ำหนัก B) สำหรับ full-text search';
```

**เหตุผลการออกแบบที่สำคัญ:**

1. **`attributes jsonb`** — สินค้าแต่ละประเภทมี attribute ต่างกันมาก (เสื้อผ้ามีไซส์/สี, หนังสือมี ISBN/ผู้แต่ง) การสร้างคอลัมน์แยกสำหรับทุก attribute ที่เป็นไปได้จะทำให้ตารางกว้างเกินไปและมี NULL เต็มไปหมด JSONB แก้ปัญหานี้โดยยังคง query ได้เร็วด้วย GIN index (ส่วนที่ 4) พร้อม constraint `jsonb_typeof(attributes) = 'object'` บังคับให้เป็น object เสมอ ไม่ใช่ array หรือ scalar
2. **`search_vector` เป็น GENERATED STORED column** — คำนวณอัตโนมัติทุกครั้งที่ `name`/`description` เปลี่ยน ไม่ต้องเขียน trigger เพิ่ม (ต่างจาก Part เก่า ๆ ที่ใช้ trigger) และ query จะเร็วมากเพราะค่าถูก materialize ไว้แล้วในตาราง ไม่ต้องคำนวณใหม่ทุกครั้งที่ค้นหา
3. **`stock_quantity` และ `reserved_quantity` แยกกัน** — เมื่อลูกค้าใส่สินค้าลงตะกร้าและกำลังชำระเงิน เราจอง (`reserved_quantity`) ไว้ก่อนโดยยังไม่ตัดสต๊อกจริง (`stock_quantity`) วิธีนี้ป้องกัน overselling ได้แม่นยำกว่าการตัดสต๊อกทันที (รายละเอียดใน ส่วนที่ 5)
4. **`compare_at_price >= price`** — บังคับกฎธุรกิจว่าราคาที่ขีดฆ่า (ราคาเดิม) ต้องไม่ต่ำกว่าราคาขายจริงที่ database level ป้องกัน bug จาก frontend ที่อาจส่งราคาผิดมา

### 2.7 ตาราง Product Variants (SKU ย่อยตาม attribute)

สินค้าหนึ่งชิ้นอาจมีหลาย variant (เช่น เสื้อสีแดงไซส์ S, M, L) แต่ละ variant มีสต๊อกและราคาของตัวเอง

```sql
CREATE TABLE product_variants (
    variant_id      bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    product_id      bigint NOT NULL REFERENCES products(product_id) ON DELETE CASCADE,
    tenant_id       bigint NOT NULL REFERENCES tenants(tenant_id) ON DELETE CASCADE,
    sku_suffix      text NOT NULL,
    variant_attributes jsonb NOT NULL DEFAULT '{}'::jsonb,
    price_override  money_amount,
    stock_quantity  non_negative_int NOT NULL DEFAULT 0,
    reserved_quantity non_negative_int NOT NULL DEFAULT 0,
    is_active       boolean NOT NULL DEFAULT true,

    CONSTRAINT product_variants_uk UNIQUE (product_id, sku_suffix),
    CONSTRAINT product_variants_reserved_check CHECK (reserved_quantity <= stock_quantity)
);

CREATE INDEX idx_product_variants_product ON product_variants (product_id);
```

### 2.8 Orders (จะถูก Partition ในส่วนที่ 6), Order Items, Payments

```sql
-- orders จะถูกปรับเป็น partitioned table ในส่วนที่ 6
-- primary key ต้องรวม partition key (created_at) ตามข้อกำหนดของ PostgreSQL declarative partitioning
CREATE TABLE orders (
    order_id        bigint GENERATED ALWAYS AS IDENTITY,
    tenant_id       bigint NOT NULL REFERENCES tenants(tenant_id),
    customer_id     bigint NOT NULL REFERENCES customers(customer_id),
    order_number    text NOT NULL DEFAULT ('SF-' || to_char(now(), 'YYYYMMDD') || '-' || substr(gen_random_uuid()::text, 1, 8)),

    status          order_status NOT NULL DEFAULT 'pending',
    currency        char(3) NOT NULL DEFAULT 'THB',

    subtotal_amount money_amount NOT NULL,
    discount_amount money_amount NOT NULL DEFAULT 0,
    shipping_amount money_amount NOT NULL DEFAULT 0,
    tax_amount      money_amount NOT NULL DEFAULT 0,
    total_amount    money_amount GENERATED ALWAYS AS
        (subtotal_amount - discount_amount + shipping_amount + tax_amount) STORED,

    shipping_address_id bigint REFERENCES customer_addresses(address_id),

    placed_at       timestamptz NOT NULL DEFAULT now(),
    created_at      timestamptz NOT NULL DEFAULT now(),
    updated_at      timestamptz NOT NULL DEFAULT now(),

    PRIMARY KEY (order_id, created_at),
    CONSTRAINT orders_tenant_order_number_uk UNIQUE (tenant_id, order_number, created_at),
    CONSTRAINT orders_discount_not_exceed_subtotal CHECK (discount_amount <= subtotal_amount)
) PARTITION BY RANGE (created_at);

CREATE TRIGGER trg_orders_updated_at
    BEFORE UPDATE ON orders
    FOR EACH ROW EXECUTE FUNCTION shopflow.set_updated_at();

-- รายการสินค้าในแต่ละออเดอร์ (snapshot ราคา ณ เวลาที่ซื้อ — ไม่ join ไป products.price เพราะราคาสินค้าเปลี่ยนได้ในอนาคต)
CREATE TABLE order_items (
    order_item_id   bigint GENERATED ALWAYS AS IDENTITY,
    order_id        bigint NOT NULL,
    order_created_at timestamptz NOT NULL,
    tenant_id       bigint NOT NULL REFERENCES tenants(tenant_id),
    product_id      bigint NOT NULL REFERENCES products(product_id),
    variant_id      bigint REFERENCES product_variants(variant_id),

    product_name_snapshot text NOT NULL,
    unit_price_snapshot   money_amount NOT NULL,
    quantity        integer NOT NULL CHECK (quantity > 0),
    line_total      money_amount GENERATED ALWAYS AS (unit_price_snapshot * quantity) STORED,

    PRIMARY KEY (order_item_id, order_created_at),
    FOREIGN KEY (order_id, order_created_at) REFERENCES orders(order_id, created_at) ON DELETE CASCADE
);

-- การชำระเงิน (หนึ่งออเดอร์อาจมีหลายรายการชำระ เช่น มัดจำ + ชำระส่วนที่เหลือ)
CREATE TABLE payments (
    payment_id      bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    order_id        bigint NOT NULL,
    order_created_at timestamptz NOT NULL,
    tenant_id       bigint NOT NULL REFERENCES tenants(tenant_id),

    method          payment_method NOT NULL,
    status          payment_status NOT NULL DEFAULT 'pending',
    amount          money_amount NOT NULL,
    provider_reference text,

    created_at      timestamptz NOT NULL DEFAULT now(),
    updated_at      timestamptz NOT NULL DEFAULT now(),

    FOREIGN KEY (order_id, order_created_at) REFERENCES orders(order_id, created_at) ON DELETE CASCADE
);

CREATE TRIGGER trg_payments_updated_at
    BEFORE UPDATE ON payments
    FOR EACH ROW EXECUTE FUNCTION shopflow.set_updated_at();
```

**เหตุผลการออกแบบที่สำคัญ:**

- `orders` ใช้ **composite primary key `(order_id, created_at)`** เพราะเป็นข้อกำหนดของ PostgreSQL: partition key ต้องเป็นส่วนหนึ่งของทุก unique/primary key บนตาราง partition (เรียนใน Part 097-103) ตารางลูกทั้งหมด (`order_items`, `payments`) จึงต้องพก `order_created_at` ติดไปด้วยเพื่ออ้างอิง FK แบบ composite กลับมา
- **`total_amount` เป็น GENERATED column** คำนวณจากอีก 4 คอลัมน์เสมอ — ป้องกัน bug จาก application ที่คำนวณยอดรวมผิดพลาดหรือลืมอัปเดตยอดรวมเมื่อแก้ discount
- **`order_items` เก็บ price snapshot** (`unit_price_snapshot`, `product_name_snapshot`) แทนที่จะ join ไปที่ `products` โดยตรง เพราะถ้าร้านค้าปรับราคาสินค้าในอนาคต ใบเสร็จเก่าต้องแสดงราคา ณ วันที่ซื้อ ไม่ใช่ราคาปัจจุบัน — นี่คือหลักการสำคัญของระบบ order/invoice ทุกระบบในโลกจริง

### 2.9 Reviews, Shipments, Discount Codes, Inventory Movements และ Audit Log

```sql
-- รีวิวสินค้า — ลูกค้ารีวิวได้เฉพาะสินค้าที่ตนซื้อจริง (บังคับด้วย FK ไป order_items ทางอ้อมผ่าน trigger ในส่วนที่ 5)
CREATE TABLE reviews (
    review_id       bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    tenant_id       bigint NOT NULL REFERENCES tenants(tenant_id) ON DELETE CASCADE,
    product_id      bigint NOT NULL REFERENCES products(product_id) ON DELETE CASCADE,
    customer_id     bigint NOT NULL REFERENCES customers(customer_id) ON DELETE CASCADE,
    order_item_id   bigint,
    order_created_at timestamptz,

    rating          smallint NOT NULL CHECK (rating BETWEEN 1 AND 5),
    title           text,
    body            text,
    status          review_status NOT NULL DEFAULT 'pending',

    created_at      timestamptz NOT NULL DEFAULT now(),

    CONSTRAINT reviews_one_per_customer_product UNIQUE (customer_id, product_id),
    FOREIGN KEY (order_item_id, order_created_at) REFERENCES order_items(order_item_id, order_created_at)
);

-- การจัดส่ง
CREATE TABLE shipments (
    shipment_id     bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    order_id        bigint NOT NULL,
    order_created_at timestamptz NOT NULL,
    tenant_id       bigint NOT NULL REFERENCES tenants(tenant_id),

    carrier         text,
    tracking_number text,
    status          shipment_status NOT NULL DEFAULT 'pending',
    shipped_at      timestamptz,
    delivered_at    timestamptz,

    FOREIGN KEY (order_id, order_created_at) REFERENCES orders(order_id, created_at) ON DELETE CASCADE
);

-- โค้ดส่วนลด
CREATE TABLE discount_codes (
    discount_code_id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    tenant_id       bigint NOT NULL REFERENCES tenants(tenant_id) ON DELETE CASCADE,
    code            text NOT NULL,
    percent_off     percentage,
    fixed_amount_off money_amount,
    max_uses        integer,
    used_count      integer NOT NULL DEFAULT 0,
    valid_from      timestamptz NOT NULL DEFAULT now(),
    valid_until     timestamptz,
    is_active       boolean NOT NULL DEFAULT true,

    CONSTRAINT discount_codes_tenant_code_uk UNIQUE (tenant_id, code),
    CONSTRAINT discount_codes_one_type_check CHECK (
        (percent_off IS NOT NULL AND fixed_amount_off IS NULL) OR
        (percent_off IS NULL AND fixed_amount_off IS NOT NULL)
    )
);

-- ประวัติการเคลื่อนไหวของสต๊อก (audit trail เฉพาะทางสำหรับ inventory — ใช้วิเคราะห์และตรวจสอบย้อนหลัง)
CREATE TABLE inventory_movements (
    movement_id     bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    tenant_id       bigint NOT NULL REFERENCES tenants(tenant_id) ON DELETE CASCADE,
    product_id      bigint NOT NULL REFERENCES products(product_id) ON DELETE CASCADE,
    variant_id      bigint REFERENCES product_variants(variant_id),
    movement_type   inventory_movement_type NOT NULL,
    quantity_delta  integer NOT NULL,
    reference_order_id bigint,
    note            text,
    created_by      bigint REFERENCES employees(employee_id),
    created_at      timestamptz NOT NULL DEFAULT now()
);

CREATE INDEX idx_inventory_movements_product ON inventory_movements (product_id, created_at DESC);

-- Audit log กลาง — บันทึกทุกการเปลี่ยนแปลงข้อมูลสำคัญ (generic ใช้ได้กับหลายตาราง)
CREATE TABLE audit_log (
    audit_id        bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    tenant_id       bigint,
    table_name      text NOT NULL,
    record_id       text NOT NULL,
    action          audit_action NOT NULL,
    changed_by      text,
    old_data        jsonb,
    new_data        jsonb,
    changed_at      timestamptz NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_log_table_record ON audit_log (table_name, record_id);
CREATE INDEX idx_audit_log_tenant_time ON audit_log (tenant_id, changed_at DESC);
```

**เหตุผลการออกแบบ:**

- `reviews_one_per_customer_product` บังคับลูกค้ารีวิวสินค้าแต่ละชิ้นได้ครั้งเดียว ป้องกันการปั่นรีวิว
- `inventory_movements` แยกจาก `audit_log` เพราะมี business meaning เฉพาะทาง (ใช้คำนวณ stock reconciliation, forecast) ในขณะที่ `audit_log` เป็น generic compliance log ที่ใช้กับทุกตาราง — การแยกสอง concern นี้ทำให้ query แต่ละแบบทำได้ตรงจุดและเร็วกว่าการยัดทุกอย่างลงตารางเดียว

Schema พื้นฐานทั้งหมดตอนนี้มี 16 ตารางหลัก ครอบคลุมทุก entity ที่ระบบ e-commerce ระดับ production ต้องมี พร้อม constraint ที่บังคับ business rule ไว้ที่ database layer อย่างเข้มงวด — หลักการที่เรียนมาตั้งแต่ Part 006 ว่า **"ฐานข้อมูลที่ดีต้องปกป้องความถูกต้องของข้อมูลได้ด้วยตัวเอง ไม่พึ่งพา application ฝ่ายเดียว"**

---

## ส่วนที่ 3: Row Level Security และ Role-based Access Control แบบเต็มรูปแบบ

### 3.1 หลักการ: เหตุใด Multi-Tenant SaaS ต้องมี RLS

ในระบบ multi-tenant ความผิดพลาดที่ร้ายแรงที่สุดคือ **query ที่ลืมใส่ `WHERE tenant_id = ...`** ซึ่งจะทำให้ข้อมูลร้านค้าหนึ่งรั่วไหลไปยังอีกร้านค้าหนึ่ง Row Level Security (RLS) ที่เรียนใน Part 076-080 แก้ปัญหานี้โดยย้ายการบังคับ tenant isolation จาก "ความรับผิดชอบของ developer ทุกคนทุก query" ไปเป็น **"กฎที่ PostgreSQL บังคับใช้เองโดยอัตโนมัติทุกครั้ง"**

### 3.2 การสร้าง Database Roles

```sql
-- Role ระดับ application สำหรับแต่ละ concern (ไม่ใช่ login role ของพนักงานแต่ละคน)
-- application จะ SET ROLE หรือใช้ connection ที่ authenticate ด้วย role เหล่านี้

-- role กลางที่ไม่มีสิทธิ์ login โดยตรง ใช้เป็น "แม่แบบ" สิทธิ์
CREATE ROLE shopflow_app NOLOGIN;

-- role สำหรับ backend application server (ใช้ต่อ connection ปกติ ผ่าน PgBouncer)
CREATE ROLE shopflow_app_user LOGIN PASSWORD 'use_a_strong_secret_from_vault' IN ROLE shopflow_app;

-- role สำหรับ read-only reporting/analytics (ต่อไปยัง read replica)
CREATE ROLE shopflow_readonly LOGIN PASSWORD 'use_a_strong_secret_from_vault';

-- role สำหรับ background job (batch processing, cron, ETL)
CREATE ROLE shopflow_worker LOGIN PASSWORD 'use_a_strong_secret_from_vault' IN ROLE shopflow_app;

-- role สำหรับ database migration/DDL เท่านั้น (แยกจาก runtime role โดยเด็ดขาด)
CREATE ROLE shopflow_migrator LOGIN PASSWORD 'use_a_strong_secret_from_vault';

-- ให้สิทธิ์ schema และตารางพื้นฐาน
GRANT USAGE ON SCHEMA shopflow TO shopflow_app, shopflow_readonly;
GRANT ALL PRIVILEGES ON SCHEMA shopflow TO shopflow_migrator;

GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA shopflow TO shopflow_app;
GRANT SELECT ON ALL TABLES IN SCHEMA shopflow TO shopflow_readonly;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA shopflow TO shopflow_app;

-- ตั้งค่า default privilege สำหรับตารางที่จะสร้างในอนาคต (สำคัญมาก — ลืมส่วนนี้บ่อยที่สุด)
ALTER DEFAULT PRIVILEGES IN SCHEMA shopflow
    GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO shopflow_app;
ALTER DEFAULT PRIVILEGES IN SCHEMA shopflow
    GRANT SELECT ON TABLES TO shopflow_readonly;
```

### 3.3 การเปิดใช้ Row Level Security และสร้าง Policy

```sql
-- ฟังก์ชัน helper: ดึง tenant_id ปัจจุบันจาก session variable ที่ application ตั้งค่าไว้หลัง authenticate
CREATE OR REPLACE FUNCTION shopflow.current_tenant_id()
RETURNS bigint
LANGUAGE sql
STABLE
AS $$
    SELECT NULLIF(current_setting('app.current_tenant_id', true), '')::bigint;
$$;

-- ฟังก์ชัน helper: ดึง role ของผู้ใช้งานปัจจุบัน (employee role) จาก session variable
CREATE OR REPLACE FUNCTION shopflow.current_employee_role()
RETURNS text
LANGUAGE sql
STABLE
AS $$
    SELECT NULLIF(current_setting('app.current_employee_role', true), '');
$$;

-- เปิดใช้ RLS กับทุกตารางที่มี tenant_id
ALTER TABLE tenants              ENABLE ROW LEVEL SECURITY;
ALTER TABLE employees            ENABLE ROW LEVEL SECURITY;
ALTER TABLE customers            ENABLE ROW LEVEL SECURITY;
ALTER TABLE customer_addresses   ENABLE ROW LEVEL SECURITY;
ALTER TABLE categories           ENABLE ROW LEVEL SECURITY;
ALTER TABLE suppliers            ENABLE ROW LEVEL SECURITY;
ALTER TABLE products             ENABLE ROW LEVEL SECURITY;
ALTER TABLE product_variants     ENABLE ROW LEVEL SECURITY;
ALTER TABLE orders               ENABLE ROW LEVEL SECURITY;
ALTER TABLE order_items          ENABLE ROW LEVEL SECURITY;
ALTER TABLE payments             ENABLE ROW LEVEL SECURITY;
ALTER TABLE reviews              ENABLE ROW LEVEL SECURITY;
ALTER TABLE shipments            ENABLE ROW LEVEL SECURITY;
ALTER TABLE discount_codes       ENABLE ROW LEVEL SECURITY;
ALTER TABLE inventory_movements  ENABLE ROW LEVEL SECURITY;

-- บังคับใช้ RLS แม้กับเจ้าของตาราง (สำคัญมาก! ค่า default ของ PostgreSQL คือ table owner ข้าม RLS ได้)
ALTER TABLE tenants              FORCE ROW LEVEL SECURITY;
ALTER TABLE employees            FORCE ROW LEVEL SECURITY;
ALTER TABLE customers            FORCE ROW LEVEL SECURITY;
ALTER TABLE products             FORCE ROW LEVEL SECURITY;
ALTER TABLE orders               FORCE ROW LEVEL SECURITY;
ALTER TABLE order_items          FORCE ROW LEVEL SECURITY;

-- Policy: ตาราง tenants เอง — เห็นได้เฉพาะแถวของตัวเอง (สำหรับ query เมตาดาต้าร้านตัวเอง)
CREATE POLICY tenant_isolation_select ON tenants
    FOR SELECT
    USING (tenant_id = shopflow.current_tenant_id());

-- Policy pattern มาตรฐานสำหรับทุกตารางที่มีคอลัมน์ tenant_id โดยตรง
CREATE POLICY tenant_isolation_all ON employees
    FOR ALL
    USING (tenant_id = shopflow.current_tenant_id())
    WITH CHECK (tenant_id = shopflow.current_tenant_id());

CREATE POLICY tenant_isolation_all ON customers
    FOR ALL
    USING (tenant_id = shopflow.current_tenant_id())
    WITH CHECK (tenant_id = shopflow.current_tenant_id());

CREATE POLICY tenant_isolation_all ON customer_addresses
    FOR ALL
    USING (tenant_id = shopflow.current_tenant_id())
    WITH CHECK (tenant_id = shopflow.current_tenant_id());

CREATE POLICY tenant_isolation_all ON categories
    FOR ALL
    USING (tenant_id = shopflow.current_tenant_id())
    WITH CHECK (tenant_id = shopflow.current_tenant_id());

CREATE POLICY tenant_isolation_all ON suppliers
    FOR ALL
    USING (tenant_id = shopflow.current_tenant_id())
    WITH CHECK (tenant_id = shopflow.current_tenant_id());

CREATE POLICY tenant_isolation_all ON products
    FOR ALL
    USING (tenant_id = shopflow.current_tenant_id())
    WITH CHECK (tenant_id = shopflow.current_tenant_id());

CREATE POLICY tenant_isolation_all ON product_variants
    FOR ALL
    USING (tenant_id = shopflow.current_tenant_id())
    WITH CHECK (tenant_id = shopflow.current_tenant_id());

CREATE POLICY tenant_isolation_all ON orders
    FOR ALL
    USING (tenant_id = shopflow.current_tenant_id())
    WITH CHECK (tenant_id = shopflow.current_tenant_id());

CREATE POLICY tenant_isolation_all ON order_items
    FOR ALL
    USING (tenant_id = shopflow.current_tenant_id())
    WITH CHECK (tenant_id = shopflow.current_tenant_id());

CREATE POLICY tenant_isolation_all ON payments
    FOR ALL
    USING (tenant_id = shopflow.current_tenant_id())
    WITH CHECK (tenant_id = shopflow.current_tenant_id());

CREATE POLICY tenant_isolation_all ON reviews
    FOR ALL
    USING (tenant_id = shopflow.current_tenant_id())
    WITH CHECK (tenant_id = shopflow.current_tenant_id());

CREATE POLICY tenant_isolation_all ON shipments
    FOR ALL
    USING (tenant_id = shopflow.current_tenant_id())
    WITH CHECK (tenant_id = shopflow.current_tenant_id());

CREATE POLICY tenant_isolation_all ON discount_codes
    FOR ALL
    USING (tenant_id = shopflow.current_tenant_id())
    WITH CHECK (tenant_id = shopflow.current_tenant_id());

CREATE POLICY tenant_isolation_all ON inventory_movements
    FOR ALL
    USING (tenant_id = shopflow.current_tenant_id())
    WITH CHECK (tenant_id = shopflow.current_tenant_id());
```

### 3.4 Policy แบบ Role-based เพิ่มเติม (RBAC ซ้อนบน RLS)

นอกจาก tenant isolation แล้ว ภายใน tenant เดียวกัน พนักงานแต่ละ role ก็ควรเห็นข้อมูลไม่เท่ากัน เช่น `staff` ไม่ควรเห็นข้อมูล `payments` ทั้งหมด มีแค่ `owner`/`admin`/`manager` เท่านั้นที่เห็นได้

```sql
-- ลบ policy เดิมของ payments แล้วแทนที่ด้วย policy ที่ละเอียดขึ้น (tenant + role)
DROP POLICY tenant_isolation_all ON payments;

CREATE POLICY tenant_isolation_select_payments ON payments
    FOR SELECT
    USING (
        tenant_id = shopflow.current_tenant_id()
        AND (
            shopflow.current_employee_role() IN ('owner', 'admin', 'manager')
            OR shopflow.current_employee_role() IS NULL  -- customer-facing connection (ลูกค้าดูของตัวเองผ่าน policy อื่น)
        )
    );

CREATE POLICY tenant_isolation_write_payments ON payments
    FOR INSERT
    WITH CHECK (tenant_id = shopflow.current_tenant_id());

CREATE POLICY tenant_isolation_update_payments ON payments
    FOR UPDATE
    USING (tenant_id = shopflow.current_tenant_id()
           AND shopflow.current_employee_role() IN ('owner', 'admin', 'manager'))
    WITH CHECK (tenant_id = shopflow.current_tenant_id());

-- staff เห็นได้เฉพาะ order ที่ยังไม่เสร็จสมบูรณ์ (กำลังดำเนินการ) ส่วน manager ขึ้นไปเห็นได้ทั้งหมด
CREATE POLICY staff_orders_visibility ON orders
    FOR SELECT
    USING (
        tenant_id = shopflow.current_tenant_id()
        AND (
            shopflow.current_employee_role() IN ('owner', 'admin', 'manager')
            OR status IN ('pending', 'awaiting_payment', 'paid', 'processing', 'shipped')
            OR shopflow.current_employee_role() IS NULL
        )
    );
```

### 3.5 การใช้งานจริง: Application ตั้งค่า Session Variable อย่างไร

```sql
-- ทุกครั้งที่ application ได้ connection จาก pool และ authenticate ผู้ใช้เรียบร้อยแล้ว
-- ต้อง SET session variable นี้ก่อน query ใด ๆ ภายใน transaction เดียวกัน
-- (ใช้ SET LOCAL เพื่อให้ค่ามีผลเฉพาะ transaction ปัจจุบัน ปลอดภัยเมื่อใช้ connection pooling)

BEGIN;
SET LOCAL app.current_tenant_id = '42';
SET LOCAL app.current_employee_role = 'manager';

-- ตอนนี้ query ใด ๆ ต่อจากนี้จะถูกกรองด้วย RLS อัตโนมัติ
SELECT * FROM products WHERE status = 'active';

COMMIT;
```

**ข้อควรระวังสำคัญเมื่อใช้ RLS ร่วมกับ PgBouncer (transaction pooling mode):** ต้องใช้ `SET LOCAL` ไม่ใช่ `SET` ธรรมดา เพราะ `SET LOCAL` จะถูกล้างค่าเมื่อ transaction จบ ป้องกันไม่ให้ tenant_id ของ connection ก่อนหน้าหลุดไปติดกับ connection ถัดไปที่ pool นำกลับมาใช้ซ้ำ — นี่คือ pitfall ที่อันตรายที่สุดของ multi-tenant RLS ร่วมกับ connection pooling และเป็นเหตุผลที่เราต้องเข้าใจ PgBouncer อย่างลึกซึ้งจาก Part 087-091

### 3.6 ทดสอบ RLS ว่าทำงานถูกต้อง

```sql
-- สร้างผู้ใช้ทดสอบและยืนยันว่าเห็นเฉพาะข้อมูลของ tenant ตัวเอง
BEGIN;
SET LOCAL app.current_tenant_id = '1';
SELECT count(*) AS tenant_1_products FROM products;  -- เห็นเฉพาะสินค้าของ tenant 1
COMMIT;

BEGIN;
SET LOCAL app.current_tenant_id = '2';
SELECT count(*) AS tenant_2_products FROM products;  -- เห็นเฉพาะสินค้าของ tenant 2 คนละชุดกับด้านบน
COMMIT;

-- ทดสอบว่าถ้าไม่ SET tenant_id เลย จะไม่เห็นอะไรเลย (fail-closed, ไม่ใช่ fail-open)
BEGIN;
SELECT count(*) FROM products;  -- คาดหวัง 0 แถวเสมอ เพราะ current_tenant_id() คืนค่า NULL
ROLLBACK;
```

หลักการออกแบบที่สำคัญที่สุดของ RLS ในระบบนี้คือ **fail-closed**: ถ้า application ลืม `SET LOCAL app.current_tenant_id` ไม่ว่าด้วยเหตุผลใดก็ตาม ระบบจะคืนค่า 0 แถวเสมอ แทนที่จะรั่วไหลข้อมูลทุก tenant ออกมา — นี่คือความแตกต่างระหว่างระบบที่ "ปลอดภัยโดยการออกแบบ" (secure by design) กับระบบที่หวังพึ่ง discipline ของ developer เพียงอย่างเดียว

---

## ส่วนที่ 4: Indexing Strategy แบบสมบูรณ์

การออกแบบ index ที่ดีต้องมาจากการวิเคราะห์ **query pattern จริง** ไม่ใช่การสร้าง index ทุกคอลัมน์ เพราะทุก index มีต้นทุน (write amplification, storage) เราจึงออกแบบ index ตามการใช้งานจริงของ ShopFlow ดังนี้

### 4.1 B-Tree Index มาตรฐาน (Foreign Key และ Filter ที่ใช้บ่อย)

```sql
-- Foreign key ควรมี index เสมอ เพราะ PostgreSQL ไม่สร้าง index ให้ FK โดยอัตโนมัติ (ต่างจาก primary key)
-- ถ้าไม่มี index บน FK คอลัมน์ การ DELETE ตารางแม่จะต้อง sequential scan ตารางลูกทุกครั้ง

CREATE INDEX idx_employees_tenant ON employees (tenant_id);
CREATE INDEX idx_customers_tenant ON customers (tenant_id);
CREATE INDEX idx_customer_addresses_customer ON customer_addresses (customer_id);
CREATE INDEX idx_products_category ON products (category_id);
CREATE INDEX idx_products_supplier ON products (supplier_id);
CREATE INDEX idx_order_items_product ON order_items (product_id);
CREATE INDEX idx_reviews_product ON reviews (product_id);
CREATE INDEX idx_reviews_customer ON reviews (customer_id);

-- Composite index: query ที่กรองด้วย tenant_id + status พร้อมกันบ่อยมาก (list สินค้า active ของร้าน)
-- ลำดับคอลัมน์สำคัญ: tenant_id (equality, selective) มาก่อน status (equality เช่นกัน แต่ cardinality ต่ำกว่า)
CREATE INDEX idx_products_tenant_status ON products (tenant_id, status);

-- Composite index สำหรับ list order ของลูกค้า เรียงตามวันที่ล่าสุด (ใช้บ่อยที่สุดในหน้า "ประวัติการสั่งซื้อ")
CREATE INDEX idx_orders_customer_placed ON orders (tenant_id, customer_id, placed_at DESC);

-- Composite index สำหรับ dashboard ที่กรองสถานะออเดอร์ของร้านค้า
CREATE INDEX idx_orders_tenant_status_placed ON orders (tenant_id, status, placed_at DESC);
```

### 4.2 GIN Index สำหรับ JSONB Attributes

```sql
-- GIN index บน JSONB ทั้งคอลัมน์ รองรับ query แบบ containment (@>) เช่น
-- WHERE attributes @> '{"color":"red"}'  หรือ  WHERE attributes ? 'size'
CREATE INDEX idx_products_attributes_gin ON products USING GIN (attributes);

-- ถ้าต้องการ query attribute เฉพาะ key บ่อย ๆ (เช่น กรองตามสี) สร้าง expression index เฉพาะทางจะเร็วกว่า
CREATE INDEX idx_products_attr_color ON products ((attributes ->> 'color'))
    WHERE attributes ? 'color';

-- GIN index บน tags array รองรับ query แบบ WHERE tags && ARRAY['bestseller'] หรือ tags @> ARRAY['new-arrival']
CREATE INDEX idx_products_tags_gin ON products USING GIN (tags);
```

### 4.3 GIN Index สำหรับ Full-Text Search

```sql
-- GIN index บน generated tsvector column — เร็วกว่า GiST สำหรับ static data ที่ update ไม่บ่อยเท่า insert/select
CREATE INDEX idx_products_search_vector_gin ON products USING GIN (search_vector);

-- composite GIN ด้วย btree_gin extension: รวม tenant_id (equality) กับ search_vector (full-text) ในดัชนีเดียว
-- ทำให้ query "ค้นหาสินค้าในร้านนี้ด้วยคำค้นนี้" ใช้ index scan เดียวจบ ไม่ต้อง join สอง index
CREATE INDEX idx_products_tenant_search_gin ON products USING GIN (tenant_id, search_vector);

-- pg_trgm index สำหรับการค้นหาแบบ ILIKE '%คำค้น%' หรือรองรับการพิมพ์ผิดเล็กน้อย (fuzzy search) บนชื่อสินค้า
CREATE INDEX idx_products_name_trgm ON products USING GIN (name gin_trgm_ops);
```

### 4.4 Partial Index (ดัชนีเฉพาะแถวที่ query บ่อย)

```sql
-- 90% ของ query บนตาราง products กรองเฉพาะสินค้าที่ status = 'active' เท่านั้น (ที่แสดงหน้าร้าน)
-- partial index ที่กรองเฉพาะแถว active จะมีขนาดเล็กกว่า full index มาก และเร็วกว่าเมื่อ query ตรงเงื่อนไขนี้
CREATE INDEX idx_products_active_only ON products (tenant_id, created_at DESC)
    WHERE status = 'active';

-- สินค้าใกล้หมดสต๊อก (ใช้ใน dashboard แจ้งเตือน reorder) — เป็น query ที่รันบ่อยแต่ match แถวน้อยมาก
CREATE INDEX idx_products_low_stock ON products (tenant_id, stock_quantity)
    WHERE stock_quantity <= reorder_threshold AND status = 'active';

-- ออเดอร์ที่ยังไม่เสร็จสมบูรณ์ (operational dashboard เช็คบ่อยมากทุกไม่กี่วินาที)
CREATE INDEX idx_orders_open_status ON orders (tenant_id, placed_at)
    WHERE status IN ('pending', 'awaiting_payment', 'paid', 'processing');

-- รีวิวที่รอการอนุมัติ (moderation queue)
CREATE INDEX idx_reviews_pending ON reviews (tenant_id, created_at)
    WHERE status = 'pending';
```

### 4.5 Unique Index สำหรับ Business Constraint (ทบทวนจากส่วนที่ 2)

```sql
-- ที่จริง unique constraint หลายตัวถูกสร้างแล้วในส่วนที่ 2 ผ่าน CONSTRAINT ... UNIQUE
-- ในที่นี้สรุปให้เห็นภาพรวมว่า unique constraint ก็คือ unique index โดยปริยาย ไม่ต้องสร้างซ้ำ:
--   products_tenant_sku_uk, products_tenant_slug_uk, orders_tenant_order_number_uk,
--   customer_addresses_one_default_uk (partial), reviews_one_per_customer_product
```

### 4.6 สรุปหลักการเลือก Index Type

| ประเภท Index | ใช้เมื่อไร | ตัวอย่างใน ShopFlow |
|---|---|---|
| B-Tree | equality / range query, ORDER BY, FK | `tenant_id`, `placed_at DESC` |
| Composite B-Tree | filter หลายคอลัมน์พร้อมกันเป็นประจำ | `(tenant_id, status)` |
| GIN (jsonb) | containment query บน JSONB | `attributes @> '{"color":"red"}'` |
| GIN (tsvector) | full-text search | `search_vector @@ query` |
| GIN (array) | array containment/overlap | `tags && ARRAY[...]` |
| GIN (trgm) | fuzzy match / `ILIKE '%x%'` | ค้นหาชื่อสินค้าพิมพ์ผิด |
| Partial Index | query กรองเฉพาะ subset เล็ก ๆ ของตารางเป็นประจำ | สินค้า active, สต๊อกใกล้หมด |

หลักการทองคำที่ต้องจำ (จาก Part 032-036 และ Part 081-086): **สร้าง index ตาม query จริงที่วัดผลจาก `pg_stat_statements` เท่านั้น อย่าสร้างล่วงหน้าตามความรู้สึก** เพราะทุก index เพิ่มต้นทุนให้ `INSERT`/`UPDATE` ทุกครั้ง — ระบบที่มี index มากเกินไปจะเขียนช้าโดยไม่จำเป็น

---

## ส่วนที่ 5: Business Logic ด้วย PL/pgSQL

ส่วนนี้คือหัวใจของความถูกต้องของระบบ (data integrity) เราจะสร้างฟังก์ชันสำหรับ checkout process, inventory management และ audit trail ที่ปลอดภัยต่อ concurrency (race condition) ด้วยเทคนิคที่เรียนจาก Part 041-051

### 5.1 ฟังก์ชัน Audit Trail อัตโนมัติ (Generic Trigger)

```sql
CREATE OR REPLACE FUNCTION shopflow.fn_audit_trigger()
RETURNS trigger
LANGUAGE plpgsql
AS $$
DECLARE
    v_tenant_id bigint;
    v_record_id text;
BEGIN
    -- พยายามดึง tenant_id จากแถวที่เปลี่ยน (ถ้าตารางมีคอลัมน์นี้)
    BEGIN
        IF TG_OP = 'DELETE' THEN
            v_tenant_id := (to_jsonb(OLD) ->> 'tenant_id')::bigint;
        ELSE
            v_tenant_id := (to_jsonb(NEW) ->> 'tenant_id')::bigint;
        END IF;
    EXCEPTION WHEN OTHERS THEN
        v_tenant_id := NULL;
    END;

    IF TG_OP = 'DELETE' THEN
        v_record_id := (to_jsonb(OLD) ->> TG_ARGV[0]);
    ELSE
        v_record_id := (to_jsonb(NEW) ->> TG_ARGV[0]);
    END IF;

    INSERT INTO shopflow.audit_log (tenant_id, table_name, record_id, action, changed_by, old_data, new_data)
    VALUES (
        v_tenant_id,
        TG_TABLE_NAME,
        v_record_id,
        TG_OP::audit_action,
        coalesce(current_setting('app.current_employee_id', true), current_user),
        CASE WHEN TG_OP IN ('UPDATE', 'DELETE') THEN to_jsonb(OLD) ELSE NULL END,
        CASE WHEN TG_OP IN ('UPDATE', 'INSERT') THEN to_jsonb(NEW) ELSE NULL END
    );

    RETURN COALESCE(NEW, OLD);
END;
$$;

-- ติดตั้ง audit trigger กับตารางที่ต้องตรวจสอบย้อนหลังได้ (ธุรกิจสำคัญ/การเงิน)
CREATE TRIGGER trg_audit_products
    AFTER INSERT OR UPDATE OR DELETE ON products
    FOR EACH ROW EXECUTE FUNCTION shopflow.fn_audit_trigger('product_id');

CREATE TRIGGER trg_audit_orders
    AFTER INSERT OR UPDATE OR DELETE ON orders
    FOR EACH ROW EXECUTE FUNCTION shopflow.fn_audit_trigger('order_id');

CREATE TRIGGER trg_audit_payments
    AFTER INSERT OR UPDATE OR DELETE ON payments
    FOR EACH ROW EXECUTE FUNCTION shopflow.fn_audit_trigger('payment_id');

CREATE TRIGGER trg_audit_employees
    AFTER INSERT OR UPDATE OR DELETE ON employees
    FOR EACH ROW EXECUTE FUNCTION shopflow.fn_audit_trigger('employee_id');
```

### 5.2 ฟังก์ชันจัดการสต๊อก: จองสต๊อกและปล่อยสต๊อกอย่างปลอดภัยต่อ Concurrency

จุดที่ยากที่สุดของระบบ e-commerce คือการป้องกัน **overselling** เมื่อลูกค้าหลายคนซื้อสินค้าชิ้นสุดท้ายพร้อมกัน เราใช้ `SELECT ... FOR UPDATE` (row-level locking ที่เรียนใน Part 041-046) เพื่อป้องกัน race condition

```sql
CREATE OR REPLACE FUNCTION shopflow.fn_reserve_stock(
    p_tenant_id bigint,
    p_product_id bigint,
    p_quantity integer
) RETURNS boolean
LANGUAGE plpgsql
AS $$
DECLARE
    v_available integer;
BEGIN
    IF p_quantity <= 0 THEN
        RAISE EXCEPTION 'จำนวนที่ต้องการจองต้องมากกว่า 0' USING ERRCODE = 'invalid_parameter_value';
    END IF;

    -- FOR UPDATE ล็อกแถวนี้จนกว่า transaction จะจบ ป้องกัน transaction อื่นอ่าน/แก้ไขพร้อมกัน
    SELECT (stock_quantity - reserved_quantity)
    INTO v_available
    FROM shopflow.products
    WHERE product_id = p_product_id AND tenant_id = p_tenant_id
    FOR UPDATE;

    IF NOT FOUND THEN
        RAISE EXCEPTION 'ไม่พบสินค้า product_id=%', p_product_id USING ERRCODE = 'no_data_found';
    END IF;

    IF v_available < p_quantity THEN
        RETURN false;  -- สต๊อกไม่พอ ให้ caller ตัดสินใจว่าจะแจ้ง error อย่างไรต่อ
    END IF;

    UPDATE shopflow.products
    SET reserved_quantity = reserved_quantity + p_quantity
    WHERE product_id = p_product_id AND tenant_id = p_tenant_id;

    INSERT INTO shopflow.inventory_movements
        (tenant_id, product_id, movement_type, quantity_delta, note)
    VALUES
        (p_tenant_id, p_product_id, 'reserved', -p_quantity, 'จองสต๊อกสำหรับ checkout');

    RETURN true;
END;
$$;

COMMENT ON FUNCTION shopflow.fn_reserve_stock IS
    'จองสต๊อกสินค้าอย่างปลอดภัยต่อ concurrency ด้วย SELECT FOR UPDATE — ป้องกัน overselling เมื่อมีคำสั่งซื้อพร้อมกัน';

-- ฟังก์ชันปล่อยสต๊อกที่จองไว้ (กรณียกเลิก checkout หรือ cart หมดอายุ)
CREATE OR REPLACE FUNCTION shopflow.fn_release_reserved_stock(
    p_tenant_id bigint,
    p_product_id bigint,
    p_quantity integer
) RETURNS void
LANGUAGE plpgsql
AS $$
BEGIN
    UPDATE shopflow.products
    SET reserved_quantity = GREATEST(0, reserved_quantity - p_quantity)
    WHERE product_id = p_product_id AND tenant_id = p_tenant_id;

    INSERT INTO shopflow.inventory_movements
        (tenant_id, product_id, movement_type, quantity_delta, note)
    VALUES
        (p_tenant_id, p_product_id, 'released', p_quantity, 'ปล่อยสต๊อกที่จองไว้คืน (checkout ยกเลิก/หมดอายุ)');
END;
$$;
```

### 5.3 ฟังก์ชัน Checkout ที่สมบูรณ์ (Atomic Transaction)

นี่คือฟังก์ชันที่ซับซ้อนที่สุดในระบบ — ต้องสร้าง order, order_items, ตัดสต๊อกจริง, และบันทึก payment ทั้งหมดใน transaction เดียว ถ้าขั้นตอนใดล้มเหลว ทุกอย่างต้อง rollback พร้อมกัน (ใช้หลัก Exception Handling จาก Part 041-046)

```sql
-- Custom type สำหรับส่งรายการสินค้าในตะกร้าเข้าฟังก์ชัน checkout เป็น array
CREATE TYPE shopflow.checkout_line_item AS (
    product_id  bigint,
    variant_id  bigint,
    quantity    integer
);

CREATE OR REPLACE FUNCTION shopflow.fn_checkout_order(
    p_tenant_id bigint,
    p_customer_id bigint,
    p_shipping_address_id bigint,
    p_items shopflow.checkout_line_item[],
    p_discount_code text DEFAULT NULL
) RETURNS bigint
LANGUAGE plpgsql
AS $$
DECLARE
    v_order_id       bigint;
    v_order_created_at timestamptz := now();
    v_item           shopflow.checkout_line_item;
    v_product        shopflow.products%ROWTYPE;
    v_subtotal       numeric(14,2) := 0;
    v_discount       numeric(14,2) := 0;
    v_line_total     numeric(14,2);
    v_available      integer;
    v_discount_row    shopflow.discount_codes%ROWTYPE;
BEGIN
    IF array_length(p_items, 1) IS NULL OR array_length(p_items, 1) = 0 THEN
        RAISE EXCEPTION 'ตะกร้าสินค้าว่างเปล่า ไม่สามารถสั่งซื้อได้' USING ERRCODE = 'invalid_parameter_value';
    END IF;

    -- ขั้นที่ 1: สร้างออเดอร์แบบ placeholder ก่อน (subtotal = 0 ชั่วคราว จะอัปเดตทีหลัง)
    INSERT INTO shopflow.orders (tenant_id, customer_id, shipping_address_id, subtotal_amount, created_at, placed_at)
    VALUES (p_tenant_id, p_customer_id, p_shipping_address_id, 0, v_order_created_at, v_order_created_at)
    RETURNING order_id INTO v_order_id;

    -- ขั้นที่ 2: วนลูปสินค้าทุกชิ้นในตะกร้า ล็อกแถวและตรวจสต๊อกทีละรายการ
    -- เรียง product_id เพื่อป้องกัน deadlock เมื่อลูกค้าหลายคน checkout สินค้าชุดเดียวกันคนละลำดับ
    FOREACH v_item IN ARRAY (
        SELECT ARRAY(SELECT unnest(p_items) ORDER BY (unnest(p_items)).product_id)
    )
    LOOP
        SELECT * INTO v_product
        FROM shopflow.products
        WHERE product_id = v_item.product_id AND tenant_id = p_tenant_id
        FOR UPDATE;

        IF NOT FOUND THEN
            RAISE EXCEPTION 'ไม่พบสินค้า product_id=%', v_item.product_id
                USING ERRCODE = 'no_data_found';
        END IF;

        v_available := v_product.stock_quantity - v_product.reserved_quantity;
        IF v_available < v_item.quantity THEN
            RAISE EXCEPTION 'สินค้า "%" มีสต๊อกไม่พอ (เหลือ % ต้องการ %)',
                v_product.name, v_available, v_item.quantity
                USING ERRCODE = 'check_violation', HINT = 'ลดจำนวนสินค้าในตะกร้าหรือเลือกสินค้าอื่น';
        END IF;

        -- ตัดสต๊อกจริงทันที (checkout สำเร็จทันที ไม่ใช่ระบบจองล่วงหน้าในตัวอย่างนี้)
        UPDATE shopflow.products
        SET stock_quantity = stock_quantity - v_item.quantity
        WHERE product_id = v_item.product_id AND tenant_id = p_tenant_id;

        v_line_total := v_product.price * v_item.quantity;
        v_subtotal := v_subtotal + v_line_total;

        INSERT INTO shopflow.order_items
            (order_id, order_created_at, tenant_id, product_id, variant_id,
             product_name_snapshot, unit_price_snapshot, quantity)
        VALUES
            (v_order_id, v_order_created_at, p_tenant_id, v_item.product_id, v_item.variant_id,
             v_product.name, v_product.price, v_item.quantity);

        INSERT INTO shopflow.inventory_movements
            (tenant_id, product_id, movement_type, quantity_delta, reference_order_id, note)
        VALUES
            (p_tenant_id, v_item.product_id, 'sale', -v_item.quantity, v_order_id, 'ขายผ่าน checkout');

        -- แจ้งเตือนสต๊อกต่ำ (สามารถต่อยอดเป็น NOTIFY หรือ log สำหรับระบบแจ้งเตือนภายนอก)
        IF (v_product.stock_quantity - v_item.quantity) <= v_product.reorder_threshold THEN
            RAISE NOTICE 'สินค้า % (id=%) สต๊อกต่ำกว่าเกณฑ์ reorder แล้ว', v_product.name, v_product.product_id;
        END IF;
    END LOOP;

    -- ขั้นที่ 3: คำนวณส่วนลด (ถ้ามีโค้ด)
    IF p_discount_code IS NOT NULL THEN
        SELECT * INTO v_discount_row
        FROM shopflow.discount_codes
        WHERE tenant_id = p_tenant_id
          AND code = p_discount_code
          AND is_active
          AND now() BETWEEN valid_from AND coalesce(valid_until, 'infinity'::timestamptz)
          AND (max_uses IS NULL OR used_count < max_uses)
        FOR UPDATE;

        IF FOUND THEN
            v_discount := CASE
                WHEN v_discount_row.percent_off IS NOT NULL THEN round(v_subtotal * v_discount_row.percent_off / 100, 2)
                ELSE LEAST(v_discount_row.fixed_amount_off, v_subtotal)
            END;

            UPDATE shopflow.discount_codes
            SET used_count = used_count + 1
            WHERE discount_code_id = v_discount_row.discount_code_id;
        ELSE
            RAISE EXCEPTION 'โค้ดส่วนลด "%" ไม่ถูกต้องหรือหมดอายุแล้ว', p_discount_code
                USING ERRCODE = 'check_violation';
        END IF;
    END IF;

    -- ขั้นที่ 4: อัปเดตยอดรวมสุดท้ายของออเดอร์
    UPDATE shopflow.orders
    SET subtotal_amount = v_subtotal,
        discount_amount = v_discount,
        status = 'awaiting_payment'
    WHERE order_id = v_order_id AND created_at = v_order_created_at;

    RETURN v_order_id;

EXCEPTION
    WHEN OTHERS THEN
        -- exception ใด ๆ ที่เกิดขึ้น จะ rollback การเปลี่ยนแปลงทั้งหมดในฟังก์ชันนี้โดยอัตโนมัติ
        -- (PostgreSQL ถือว่าทั้งฟังก์ชันเป็นส่วนหนึ่งของ transaction เดียวกับผู้เรียก)
        RAISE;
END;
$$;

COMMENT ON FUNCTION shopflow.fn_checkout_order IS
    'ฟังก์ชัน checkout หลัก: สร้างออเดอร์ ตัดสต๊อกอย่างปลอดภัยต่อ concurrency คำนวณส่วนลด — atomic ทั้งหมดหรือ rollback ทั้งหมด';
```

**เหตุผลการออกแบบสำคัญ:**

- **เรียง `product_id` ก่อนวนลูป `FOR UPDATE`** — เทคนิคป้องกัน deadlock แบบคลาสสิก: ถ้าลูกค้า A checkout สินค้า [1, 2] และลูกค้า B checkout สินค้า [2, 1] พร้อมกันโดยไม่เรียงลำดับ อาจเกิด deadlock (A ล็อก 1 รอ 2, B ล็อก 2 รอ 1) การบังคับให้ทุก transaction ล็อกตามลำดับ `product_id` เดียวกันเสมอทำให้ deadlock เป็นไปไม่ได้
- **`SELECT ... FOR UPDATE`** ล็อกแถวสินค้าเฉพาะแถวนั้น (row-level lock) ไม่ใช่ table lock ทำให้ transaction ที่ checkout สินค้าคนละชิ้นยังทำงานพร้อมกันได้ ไม่ขวางกัน
- Exception block ท้ายฟังก์ชันใช้ `RAISE` เปล่า (re-raise) เพื่อส่งต่อ error เดิมให้ caller เห็น พร้อม auto-rollback ทุกอย่างที่ทำในฟังก์ชันตาม ACID property

### 5.4 ฟังก์ชัน Payment Confirmation (เชื่อม Order Status)

```sql
CREATE OR REPLACE FUNCTION shopflow.fn_confirm_payment(
    p_tenant_id bigint,
    p_order_id bigint,
    p_order_created_at timestamptz,
    p_method payment_method,
    p_amount money_amount,
    p_provider_reference text
) RETURNS bigint
LANGUAGE plpgsql
AS $$
DECLARE
    v_payment_id bigint;
    v_order_total money_amount;
BEGIN
    SELECT total_amount INTO v_order_total
    FROM shopflow.orders
    WHERE order_id = p_order_id AND created_at = p_order_created_at AND tenant_id = p_tenant_id
    FOR UPDATE;

    IF NOT FOUND THEN
        RAISE EXCEPTION 'ไม่พบออเดอร์ order_id=%', p_order_id USING ERRCODE = 'no_data_found';
    END IF;

    IF p_amount <> v_order_total THEN
        RAISE EXCEPTION 'ยอดชำระ (%) ไม่ตรงกับยอดรวมออเดอร์ (%)', p_amount, v_order_total
            USING ERRCODE = 'check_violation';
    END IF;

    INSERT INTO shopflow.payments
        (order_id, order_created_at, tenant_id, method, status, amount, provider_reference)
    VALUES
        (p_order_id, p_order_created_at, p_tenant_id, p_method, 'captured', p_amount, p_provider_reference)
    RETURNING payment_id INTO v_payment_id;

    UPDATE shopflow.orders
    SET status = 'paid'
    WHERE order_id = p_order_id AND created_at = p_order_created_at;

    RETURN v_payment_id;
END;
$$;
```

### 5.5 การเรียกใช้งานจริง

```sql
BEGIN;
SET LOCAL app.current_tenant_id = '1';

SELECT shopflow.fn_checkout_order(
    p_tenant_id := 1,
    p_customer_id := 100,
    p_shipping_address_id := 500,
    p_items := ARRAY[
        ROW(10, NULL, 2)::shopflow.checkout_line_item,
        ROW(15, NULL, 1)::shopflow.checkout_line_item
    ],
    p_discount_code := 'WELCOME10'
);

COMMIT;
```

---

## ส่วนที่ 6: Partitioning Strategy — Orders แบบ Partition by Range (Month)

### 6.1 เหตุผลที่ต้อง Partition ตาราง Orders

ตาราง `orders` เป็นตารางที่โตเร็วที่สุดในระบบ — ร้านค้าขนาดกลางอาจมีหลายหมื่นออเดอร์ต่อเดือน เมื่อรวมทุก tenant อาจมีหลายล้านแถวต่อปี การ query ออเดอร์ "เดือนนี้" หรือ "ไตรมาสนี้" บนตารางที่มีข้อมูลสะสมหลายปีจะช้าลงเรื่อย ๆ ถ้าไม่ partition (เทคนิคจาก Part 097-103)

**ประโยชน์ของ Partitioning:**
1. **Partition Pruning** — query ที่มี `WHERE created_at >= '2026-09-01'` จะสแกนเฉพาะ partition ของเดือนกันยายนเท่านั้น ไม่ต้องสแกนข้อมูลปีก่อน ๆ
2. **การลบข้อมูลเก่า** — แทนที่จะ `DELETE` ซึ่งช้าและสร้าง bloat จำนวนมาก เราแค่ `DETACH PARTITION` แล้วเก็บเป็น archive หรือ `DROP` ทิ้งได้ทันที (O(1))
3. **Maintenance แยกส่วน** — `VACUUM`/`ANALYZE`/`REINDEX` ทำทีละ partition ได้ ไม่ต้องล็อกตารางทั้งหมด

### 6.2 การสร้าง Partition รายเดือน

```sql
-- ตาราง orders ถูกประกาศเป็น PARTITION BY RANGE (created_at) แล้วตั้งแต่ส่วนที่ 2
-- ตอนนี้เราสร้าง partition แต่ละเดือน

CREATE TABLE orders_y2026m07 PARTITION OF orders
    FOR VALUES FROM ('2026-07-01') TO ('2026-08-01');

CREATE TABLE orders_y2026m08 PARTITION OF orders
    FOR VALUES FROM ('2026-08-01') TO ('2026-09-01');

CREATE TABLE orders_y2026m09 PARTITION OF orders
    FOR VALUES FROM ('2026-09-01') TO ('2026-10-01');

CREATE TABLE orders_y2026m10 PARTITION OF orders
    FOR VALUES FROM ('2026-10-01') TO ('2026-11-01');

-- partition สำหรับข้อมูลที่หลุดช่วงเวลาที่คาดไว้ (safety net ป้องกัน INSERT ล้มเหลว)
CREATE TABLE orders_default PARTITION OF orders DEFAULT;

-- index ที่สร้างบนตารางแม่ (parent) จะถูกสร้างอัตโนมัติกับทุก partition ลูกที่มีอยู่แล้วและที่จะสร้างใหม่
CREATE INDEX idx_orders_tenant_customer ON orders (tenant_id, customer_id);
```

### 6.3 ฟังก์ชันสร้าง Partition อัตโนมัติล่วงหน้า

```sql
CREATE OR REPLACE FUNCTION shopflow.fn_create_monthly_order_partition(p_month date)
RETURNS void
LANGUAGE plpgsql
AS $$
DECLARE
    v_partition_name text := 'orders_y' || to_char(p_month, 'YYYY') || 'm' || to_char(p_month, 'MM');
    v_start date := date_trunc('month', p_month)::date;
    v_end   date := (date_trunc('month', p_month) + interval '1 month')::date;
BEGIN
    IF EXISTS (SELECT 1 FROM pg_class WHERE relname = v_partition_name) THEN
        RAISE NOTICE 'Partition % มีอยู่แล้ว ข้ามการสร้าง', v_partition_name;
        RETURN;
    END IF;

    EXECUTE format(
        'CREATE TABLE %I PARTITION OF shopflow.orders FOR VALUES FROM (%L) TO (%L)',
        v_partition_name, v_start, v_end
    );

    RAISE NOTICE 'สร้าง partition % สำเร็จ (ช่วง % ถึง %)', v_partition_name, v_start, v_end;
END;
$$;

-- ใช้ pg_cron (หรือ scheduled job ภายนอก) เรียกฟังก์ชันนี้ทุกต้นเดือน เพื่อสร้าง partition ของเดือนถัดไปล่วงหน้า
-- ตัวอย่าง: สร้าง partition ของอีก 3 เดือนข้างหน้าเสมอ ป้องกันไม่มี partition รองรับ INSERT ทัน
SELECT shopflow.fn_create_monthly_order_partition( (date_trunc('month', now()) + interval '1 month')::date );
SELECT shopflow.fn_create_monthly_order_partition( (date_trunc('month', now()) + interval '2 month')::date );
SELECT shopflow.fn_create_monthly_order_partition( (date_trunc('month', now()) + interval '3 month')::date );
```

### 6.4 การ Archive ข้อมูลเก่า

```sql
-- แนวทางเก็บข้อมูล: partition ที่เก่ากว่า 24 เดือน ให้ detach ออกและย้ายไปยังตาราง archive
-- (หรือ export เป็น Parquet/CSV ไปยัง data warehouse ก่อน drop)

-- ขั้นที่ 1: แยก partition ออกจากตารางหลัก (เร็วมาก ไม่ล็อกตารางแม่นาน)
ALTER TABLE orders DETACH PARTITION orders_y2024m01 CONCURRENTLY;

-- ขั้นที่ 2: ย้ายไป archive schema เพื่อยังคง query ได้ (เผื่อ compliance ต้องเก็บ 5-7 ปี) แต่แยกจาก hot path
ALTER TABLE orders_y2024m01 SET SCHEMA shopflow_archive;

-- ขั้นที่ 3 (ทางเลือก): เมื่อพ้นระยะเก็บตามกฎหมายแล้ว ค่อย DROP ทิ้งจริง
-- DROP TABLE shopflow_archive.orders_y2024m01;
```

**เหตุผลของ `CONCURRENTLY` ใน `DETACH PARTITION`:** ป้องกันการล็อกตารางแม่แบบ `ACCESS EXCLUSIVE` ระหว่างกระบวนการ detach ทำให้ระบบ production ยังคงรับ read/write บน partition อื่นได้ตามปกติระหว่าง maintenance — เทคนิคนี้เรียนมาจาก Part 097-103

---

## ส่วนที่ 7: Views และ Materialized Views สำหรับ Reporting/Analytics Dashboard

### 7.1 View มาตรฐาน: ข้อมูลที่ต้อง real-time เสมอ

```sql
-- View: รายละเอียดออเดอร์แบบเต็ม (join ครบทุกอย่างที่ frontend ต้องใช้แสดงหน้า order detail)
CREATE OR REPLACE VIEW shopflow.v_order_details AS
SELECT
    o.order_id,
    o.created_at,
    o.tenant_id,
    o.order_number,
    o.status,
    o.total_amount,
    c.full_name  AS customer_name,
    c.email      AS customer_email,
    count(oi.order_item_id) AS item_count,
    sum(oi.quantity)        AS total_quantity
FROM shopflow.orders o
JOIN shopflow.customers c ON c.customer_id = o.customer_id
LEFT JOIN shopflow.order_items oi
       ON oi.order_id = o.order_id AND oi.order_created_at = o.created_at
GROUP BY o.order_id, o.created_at, o.tenant_id, o.order_number, o.status, o.total_amount,
         c.full_name, c.email;

-- View: สินค้าพร้อมสถานะสต๊อก (ใช้ในหน้า admin catalog)
CREATE OR REPLACE VIEW shopflow.v_product_catalog AS
SELECT
    p.product_id,
    p.tenant_id,
    p.name,
    p.sku,
    p.price,
    p.status,
    cat.name AS category_name,
    p.stock_quantity - p.reserved_quantity AS available_quantity,
    CASE
        WHEN p.stock_quantity - p.reserved_quantity <= 0 THEN 'out_of_stock'
        WHEN p.stock_quantity - p.reserved_quantity <= p.reorder_threshold THEN 'low_stock'
        ELSE 'in_stock'
    END AS stock_health,
    coalesce(r.avg_rating, 0)::numeric(3,2) AS avg_rating,
    coalesce(r.review_count, 0) AS review_count
FROM shopflow.products p
LEFT JOIN shopflow.categories cat ON cat.category_id = p.category_id
LEFT JOIN LATERAL (
    SELECT round(avg(rating), 2) AS avg_rating, count(*) AS review_count
    FROM shopflow.reviews rv
    WHERE rv.product_id = p.product_id AND rv.status = 'approved'
) r ON true;
```

### 7.2 Recursive CTE View: Category Tree

```sql
-- View แสดง path เต็มของหมวดหมู่ เช่น "อิเล็กทรอนิกส์ > โทรศัพท์มือถือ > สมาร์ทโฟน"
CREATE OR REPLACE VIEW shopflow.v_category_tree AS
WITH RECURSIVE category_path AS (
    SELECT
        category_id, tenant_id, parent_id, name,
        name::text AS full_path,
        1 AS depth,
        ARRAY[category_id] AS path_ids
    FROM shopflow.categories
    WHERE parent_id IS NULL

    UNION ALL

    SELECT
        c.category_id, c.tenant_id, c.parent_id, c.name,
        cp.full_path || ' > ' || c.name,
        cp.depth + 1,
        cp.path_ids || c.category_id
    FROM shopflow.categories c
    JOIN category_path cp ON cp.category_id = c.parent_id
    WHERE NOT c.category_id = ANY(cp.path_ids)  -- ป้องกัน infinite loop ถ้ามี cycle
)
SELECT * FROM category_path;
```

### 7.3 Materialized View: Dashboard สรุปยอดขายรายวัน

ข้อมูล dashboard ไม่จำเป็นต้อง real-time เป๊ะทุกวินาที — การรีเฟรชทุก 15-30 นาทีเพียงพอ และลด load บนตารางหลักมหาศาล

```sql
CREATE MATERIALIZED VIEW shopflow.mv_daily_sales_summary AS
SELECT
    o.tenant_id,
    date_trunc('day', o.placed_at)::date AS sales_date,
    count(DISTINCT o.order_id) AS order_count,
    count(DISTINCT o.customer_id) AS unique_customers,
    sum(o.total_amount) AS gross_revenue,
    sum(o.discount_amount) AS total_discount,
    avg(o.total_amount)::numeric(14,2) AS avg_order_value
FROM shopflow.orders o
WHERE o.status NOT IN ('cancelled')
GROUP BY o.tenant_id, date_trunc('day', o.placed_at)
WITH DATA;

-- unique index จำเป็นสำหรับ REFRESH ... CONCURRENTLY (อนุญาตให้ query อ่าน view ได้ระหว่าง refresh)
CREATE UNIQUE INDEX idx_mv_daily_sales_uk
    ON shopflow.mv_daily_sales_summary (tenant_id, sales_date);

-- Materialized View: สินค้าขายดี Top-N ต่อร้านค้า (คำนวณจาก order_items ที่ซับซ้อนกว่า จึงเหมาะทำเป็น mat view)
CREATE MATERIALIZED VIEW shopflow.mv_top_selling_products AS
SELECT
    oi.tenant_id,
    oi.product_id,
    p.name AS product_name,
    sum(oi.quantity) AS total_units_sold,
    sum(oi.line_total) AS total_revenue,
    rank() OVER (PARTITION BY oi.tenant_id ORDER BY sum(oi.quantity) DESC) AS sales_rank
FROM shopflow.order_items oi
JOIN shopflow.products p ON p.product_id = oi.product_id
JOIN shopflow.orders o ON o.order_id = oi.order_id AND o.created_at = oi.order_created_at
WHERE o.status NOT IN ('cancelled', 'refunded')
GROUP BY oi.tenant_id, oi.product_id, p.name
WITH DATA;

CREATE UNIQUE INDEX idx_mv_top_products_uk
    ON shopflow.mv_top_selling_products (tenant_id, product_id);
```

### 7.4 การรีเฟรชแบบไม่ล็อกอ่าน (Concurrent Refresh)

```sql
-- REFRESH MATERIALIZED VIEW CONCURRENTLY อนุญาตให้มีคนอ่าน view ได้ระหว่าง refresh (ต่างจาก REFRESH ธรรมดาที่ล็อกอ่าน)
-- ต้องมี unique index อย่างน้อยหนึ่งตัวบน materialized view เสมอ (สร้างไว้แล้วด้านบน)
REFRESH MATERIALIZED VIEW CONCURRENTLY shopflow.mv_daily_sales_summary;
REFRESH MATERIALIZED VIEW CONCURRENTLY shopflow.mv_top_selling_products;

-- ตั้งเป็น scheduled job ด้วย pg_cron ให้รีเฟรชทุก 15 นาที
-- SELECT cron.schedule('refresh-daily-sales', '*/15 * * * *',
--     $$REFRESH MATERIALIZED VIEW CONCURRENTLY shopflow.mv_daily_sales_summary$$);
```

**หมายเหตุสำคัญเรื่อง RLS กับ Materialized View:** Materialized View **ไม่รองรับ RLS โดยตรง** เพราะมันเป็น physical snapshot ของข้อมูล ไม่ใช่ query ที่ evaluate ใหม่ทุกครั้ง วิธีแก้คือสร้าง regular view ที่ wrap materialized view อีกชั้นพร้อม `WHERE tenant_id = shopflow.current_tenant_id()` แล้วให้ application query ผ่าน view ชั้นนอกนี้เท่านั้น ไม่ให้สิทธิ์ query materialized view โดยตรง:

```sql
CREATE OR REPLACE VIEW shopflow.v_daily_sales_summary_secured AS
SELECT * FROM shopflow.mv_daily_sales_summary
WHERE tenant_id = shopflow.current_tenant_id();

REVOKE ALL ON shopflow.mv_daily_sales_summary FROM shopflow_app, shopflow_readonly;
GRANT SELECT ON shopflow.v_daily_sales_summary_secured TO shopflow_app, shopflow_readonly;
```

---

## ส่วนที่ 8: Performance และ Query Optimization

### 8.1 Query 1: ค้นหาสินค้าด้วย Full-Text Search + Filter คอลัมน์ + Sort

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT product_id, name, price, ts_rank(search_vector, query) AS rank
FROM shopflow.products, plainto_tsquery('simple', 'wireless headphone') query
WHERE tenant_id = 1
  AND status = 'active'
  AND search_vector @@ query
ORDER BY rank DESC
LIMIT 20;
```

**ผลลัพธ์ที่คาดหวัง (ตัวอย่าง):**

```
Limit  (cost=24.51..24.56 rows=20 width=48) (actual time=0.412..0.418 rows=14 loops=1)
  ->  Sort  (cost=24.51..24.61 rows=41 width=48) (actual time=0.410..0.413 rows=14 loops=1)
        Sort Key: (ts_rank(products.search_vector, query.query)) DESC
        Sort Method: quicksort  Memory: 26kB
        ->  Bitmap Heap Scan on products  (cost=8.30..23.45 rows=41 width=48)
              (actual time=0.180..0.350 rows=14 loops=1)
              Recheck Cond: ((tenant_id = 1) AND (search_vector @@ query.query))
              Filter: (status = 'active'::product_status)
              Heap Blocks: exact=6
              ->  Bitmap Index Scan on idx_products_tenant_search_gin
                    (cost=0.00..8.29 rows=41 width=0) (actual time=0.120..0.120 rows=18 loops=1)
                    Index Cond: ((tenant_id = 1) AND (search_vector @@ query.query))
Planning Time: 0.312 ms
Execution Time: 0.461 ms
```

**การวิเคราะห์:** composite GIN index `idx_products_tenant_search_gin` (จากส่วนที่ 4) ถูกใช้เป็น Bitmap Index Scan ทำให้กรองทั้ง `tenant_id` และ full-text search ในดัชนีเดียว — execution time ต่ำกว่า 1ms แม้ตารางมีสินค้าหลายล้านแถว เพราะ planner ไม่ต้องสแกนทั้งตาราง

### 8.2 Query 2: Dashboard ยอดขายรายเดือน (ใช้ Partition Pruning)

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT date_trunc('day', placed_at)::date AS d, count(*), sum(total_amount)
FROM shopflow.orders
WHERE tenant_id = 1
  AND created_at >= '2026-09-01' AND created_at < '2026-10-01'
GROUP BY 1
ORDER BY 1;
```

**สิ่งที่ต้องมองหาใน EXPLAIN output:** `Append` node ควรมี partition ลูกเพียง **หนึ่งตัวเดียว** (`orders_y2026m09`) ไม่ใช่ทุก partition — นี่คือหลักฐานว่า **Partition Pruning** ทำงาน (planner รู้จาก `WHERE created_at >= ... AND < ...` ว่าไม่ต้องแตะ partition อื่นเลย) ถ้าเห็นว่า planner สแกนทุก partition แปลว่า query filter เขียนไม่ตรงกับ partition key หรือใช้ function ครอบคอลัมน์ partition key จนทำให้ planner ไม่สามารถ prune ได้ (เช่น `WHERE date_trunc('month', created_at) = '2026-09-01'` จะ **ไม่** เกิด pruning เพราะ wrap คอลัมน์ด้วยฟังก์ชัน)

### 8.3 Query 3: ตรวจสอบ Sequential Scan ที่ไม่ควรเกิด (Anti-pattern)

```sql
-- Query ที่เขียนผิด (anti-pattern): ILIKE โดยไม่ใช้ trigram index
EXPLAIN (ANALYZE)
SELECT * FROM shopflow.products WHERE name ILIKE '%phone%';
-- ถ้า planner เลือก Seq Scan ทั้งที่มี idx_products_name_trgm อยู่ ให้ตรวจสอบว่า
--   1. cost ของ index scan สูงกว่า seq scan จริงหรือไม่ (ตารางเล็กเกินไป planner อาจเลือก seq scan ถูกต้องแล้ว)
--   2. random_page_cost/seq_page_cost ตั้งค่าเหมาะกับ storage จริงหรือไม่ (SSD ควรตั้ง random_page_cost ต่ำกว่าเดิม)
--   3. สถิติล่าสุด (ANALYZE) หรือยัง

-- แก้ไขให้ planner พิจารณา trigram index อย่างถูกต้อง
SET random_page_cost = 1.1;  -- เหมาะกับ SSD/cloud storage สมัยใหม่
ANALYZE shopflow.products;

EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM shopflow.products WHERE name ILIKE '%phone%';
```

### 8.4 Query 4: N+1 Query Anti-pattern และการแก้ด้วย LATERAL JOIN

```sql
-- ผิด: application เรียก query แยกทีละสินค้าเพื่อดึง review count (N+1 problem)
-- SELECT * FROM products WHERE tenant_id = 1;  -- แล้ว loop เรียก SELECT count(*) FROM reviews WHERE product_id = ? ทีละตัว

-- ถูก: รวมเป็น query เดียวด้วย LATERAL join (เรียนจาก Part 052-056)
EXPLAIN (ANALYZE, BUFFERS)
SELECT p.product_id, p.name, rv.review_count, rv.avg_rating
FROM shopflow.products p
LEFT JOIN LATERAL (
    SELECT count(*) AS review_count, round(avg(rating), 2) AS avg_rating
    FROM shopflow.reviews r
    WHERE r.product_id = p.product_id AND r.status = 'approved'
) rv ON true
WHERE p.tenant_id = 1 AND p.status = 'active';
```

### 8.5 การตรวจสุขภาพ Query ด้วย pg_stat_statements

```sql
-- หา query ที่กิน total execution time สูงสุด 10 อันดับแรก — จุดเริ่มต้นของการ optimize เสมอ
SELECT
    substring(query, 1, 80) AS query_snippet,
    calls,
    round(total_exec_time::numeric, 2) AS total_ms,
    round(mean_exec_time::numeric, 2) AS avg_ms,
    round((100 * total_exec_time / sum(total_exec_time) OVER ())::numeric, 2) AS pct_of_total
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;

-- หา query ที่มี cache hit ratio ต่ำ (อาจต้องเพิ่ม index หรือ shared_buffers)
SELECT
    substring(query, 1, 80) AS query_snippet,
    shared_blks_hit, shared_blks_read,
    round((100.0 * shared_blks_hit / NULLIF(shared_blks_hit + shared_blks_read, 0))::numeric, 2) AS cache_hit_pct
FROM pg_stat_statements
WHERE shared_blks_hit + shared_blks_read > 0
ORDER BY cache_hit_pct ASC
LIMIT 10;
```

---

## ส่วนที่ 9: High Availability, Backup และ Disaster Recovery Plan

### 9.1 สถาปัตยกรรม High Availability

```
┌────────────────┐   streaming replication (async)   ┌────────────────┐
│  Primary (AZ-1) │ ─────────────────────────────────▶│ Standby (AZ-2)  │
│  Read + Write    │                                    │  Read-only      │
└────────┬─────────┘                                    └────────┬────────┘
         │ streaming replication (async)                          │
         ▼                                                        ▼
┌────────────────┐                                     ┌────────────────┐
│ Read Replica    │  (reporting/analytics workload)     │ Read Replica    │
│ (AZ-3)           │                                     │ (DR region)     │
└──────────────────┘                                     └────────────────┘

Failover: Patroni + etcd/Consul สำหรับ automatic leader election
Connection Routing: PgBouncer + HAProxy / pgpool-II ชี้ไปยัง primary ปัจจุบันเสมอ
```

จากที่เรียนใน Part 061-065 เรื่อง Replication และ High Availability เราวางสถาปัตยกรรม ShopFlow ดังนี้:

- **Primary** รับทั้ง read และ write ตั้งอยู่ Availability Zone หลัก
- **Synchronous standby อย่างน้อย 1 ตัว** ใน AZ อื่นเพื่อการันตี zero data loss (RPO = 0) สำหรับ transaction สำคัญ (`synchronous_commit = on`, `synchronous_standby_names`)
- **Asynchronous read replica** เพิ่มเติมสำหรับ reporting/analytics workload (materialized view refresh, dashboard query หนัก ๆ) แยกออกจาก transactional workload เพื่อไม่ให้กระทบ checkout performance
- **Patroni** จัดการ automatic failover — เมื่อ primary ล่ม จะ promote standby ที่ data ล่าสุดที่สุดขึ้นเป็น primary ใหม่โดยอัตโนมัติภายในไม่กี่วินาที

```sql
-- ตัวอย่างการตั้งค่า postgresql.conf สำหรับ synchronous replication
-- synchronous_commit = on
-- synchronous_standby_names = 'ANY 1 (standby_az2, standby_az3)'
-- wal_level = replica
-- max_wal_senders = 10
-- hot_standby = on
```

### 9.2 Backup Strategy: 3-2-1 Rule

- **3 สำเนาข้อมูล**: production + local backup + offsite backup
- **2 สื่อบันทึกต่างกัน**: local disk (fast restore) + object storage (S3/GCS สำหรับ durability)
- **1 offsite**: อย่างน้อยหนึ่งชุดเก็บนอก region หลัก เผื่อ region ทั้งหมดล่ม

```bash
# Base backup รายวันด้วย pg_basebackup (เรียนจาก Part 066-070)
pg_basebackup -h primary.shopflow.internal -D /backup/base/$(date +%Y%m%d) \
    -Ft -z -Xs -P -U replication_user

# WAL archiving แบบต่อเนื่องไปยัง S3 (ใช้ pgBackRest หรือ wal-g ใน production จริง)
# archive_command = 'wal-g wal-push %p'

# ตัวอย่างการตั้งค่า pgBackRest สำหรับ full/diff/incremental backup schedule
# full backup: ทุกวันอาทิตย์
# diff backup: ทุกวันจันทร์-เสาร์
# WAL archiving: ต่อเนื่องตลอดเวลา (สำหรับ Point-in-Time Recovery)
```

### 9.3 Point-in-Time Recovery (PITR) Plan

```bash
# สถานการณ์: มี bug ใน deploy ทำให้ query ลบข้อมูลผิดพลาดเมื่อ 14:32 น. วันนี้
# ต้อง restore ไปยังเวลา 14:31:00 (ก่อนเกิดเหตุ 1 นาที)

pgbackrest --stanza=shopflow --type=time \
    --target="2026-09-25 14:31:00+07" \
    --target-action=promote \
    restore
```

**เป้าหมาย RPO/RTO ของ ShopFlow:**

| Metric | เป้าหมาย | วิธีบรรลุ |
|---|---|---|
| RPO (Recovery Point Objective) | < 5 นาที | Synchronous replication + WAL archiving ต่อเนื่องทุก 60 วินาที |
| RTO (Recovery Time Objective) | < 15 นาที | Automatic failover ด้วย Patroni (failover เอง < 30 วินาที) + PITR ทดสอบ restore เดือนละครั้ง |

**หลักการสำคัญ:** แผน DR ที่ไม่เคยทดสอบ restore จริง ถือว่า**ไม่มีแผน DR** — ต้องมี runbook และซ้อม disaster recovery drill เป็นประจำ (เช่น ทุกไตรมาส) เพื่อยืนยันว่าทีมสามารถกู้คืนระบบได้จริงภายในเวลาที่กำหนด ไม่ใช่แค่ในทฤษฎี

---

## ส่วนที่ 10: Application Integration

### 10.1 Node.js (พร้อม `pg` driver และ Connection Pool)

```javascript
// db.js — การเชื่อมต่อ PostgreSQL จาก Node.js ผ่าน PgBouncer พร้อมตั้งค่า RLS session variable
import pg from 'pg';

const pool = new pg.Pool({
  host: process.env.PGBOUNCER_HOST,
  port: 6432,
  database: 'shopflow',
  user: process.env.DB_APP_USER,
  password: process.env.DB_APP_PASSWORD,
  max: 20,
  idleTimeoutMillis: 30000,
});

// helper: รัน query ภายใต้ tenant context ที่ถูกต้องเสมอ (บังคับ RLS)
export async function withTenantContext(tenantId, employeeRole, fn) {
  const client = await pool.connect();
  try {
    await client.query('BEGIN');
    await client.query('SET LOCAL app.current_tenant_id = $1', [tenantId]);
    if (employeeRole) {
      await client.query('SET LOCAL app.current_employee_role = $1', [employeeRole]);
    }
    const result = await fn(client);
    await client.query('COMMIT');
    return result;
  } catch (err) {
    await client.query('ROLLBACK');
    throw err;
  } finally {
    client.release();
  }
}

// ตัวอย่างการใช้งาน: checkout API endpoint
export async function checkout(tenantId, customerId, addressId, items, discountCode) {
  return withTenantContext(tenantId, null, async (client) => {
    const { rows } = await client.query(
      `SELECT shopflow.fn_checkout_order($1, $2, $3, $4::shopflow.checkout_line_item[], $5) AS order_id`,
      [tenantId, customerId, addressId, items, discountCode]
    );
    return rows[0].order_id;
  });
}
```

### 10.2 Python (พร้อม `psycopg` และ SQLAlchemy)

```python
# db.py — เชื่อมต่อ PostgreSQL จาก Python ด้วย psycopg3 + connection pool
import psycopg
from psycopg_pool import ConnectionPool
from contextlib import contextmanager

pool = ConnectionPool(
    conninfo="host=pgbouncer.internal port=6432 dbname=shopflow "
             "user=shopflow_app_user password=***",
    min_size=5,
    max_size=20,
)

@contextmanager
def tenant_session(tenant_id: int, employee_role: str | None = None):
    """เปิด transaction พร้อมตั้งค่า RLS session variable ให้ทุก query ภายใน block นี้ถูกกรอง tenant อัตโนมัติ"""
    with pool.connection() as conn:
        with conn.transaction():
            conn.execute("SET LOCAL app.current_tenant_id = %s", (tenant_id,))
            if employee_role:
                conn.execute("SET LOCAL app.current_employee_role = %s", (employee_role,))
            yield conn


def get_dashboard_summary(tenant_id: int, start_date: str, end_date: str):
    with tenant_session(tenant_id, employee_role="manager") as conn:
        rows = conn.execute(
            """
            SELECT sales_date, order_count, gross_revenue
            FROM shopflow.v_daily_sales_summary_secured
            WHERE sales_date BETWEEN %s AND %s
            ORDER BY sales_date
            """,
            (start_date, end_date),
        ).fetchall()
        return rows


def reserve_and_checkout(tenant_id: int, customer_id: int, address_id: int, items: list[dict], discount_code: str | None):
    with tenant_session(tenant_id) as conn:
        line_items = [(i["product_id"], i.get("variant_id"), i["quantity"]) for i in items]
        result = conn.execute(
            "SELECT shopflow.fn_checkout_order(%s, %s, %s, %s, %s) AS order_id",
            (tenant_id, customer_id, address_id, line_items, discount_code),
        ).fetchone()
        return result["order_id"]
```

**หลักการสำคัญที่เชื่อมโยงกับ Part 087-091:** ทุก connection ที่ไปจาก application ต้อง**ผ่าน PgBouncer ในโหมด transaction pooling** เพื่อรองรับ concurrent connection จำนวนมากโดยไม่ทำให้ PostgreSQL server เปิด connection จริงเกินขีดจำกัด (`max_connections`) — และเพราะ `SET LOCAL` ถูกใช้แทน `SET` ธรรมดาเสมอ ทำให้ session variable ปลอดภัยแม้ connection จะถูกใช้ซ้ำ (reuse) ข้าม request ก็ตาม

---

## ส่วนที่ 11: Monitoring และ Observability Plan

เชื่อมโยงกับเทคนิคจาก Part 072 เรื่อง Monitoring เราวาง 4 ชั้นของการสังเกตการณ์ระบบ ShopFlow ดังนี้:

### 11.1 Database-level Metrics

```sql
-- Connection saturation — เตือนเมื่อใกล้ max_connections
SELECT count(*) AS active_connections,
       (SELECT setting::int FROM pg_settings WHERE name = 'max_connections') AS max_connections,
       round(100.0 * count(*) / (SELECT setting::int FROM pg_settings WHERE name = 'max_connections'), 1) AS pct_used
FROM pg_stat_activity;

-- Replication lag — สำคัญที่สุดสำหรับ HA setup
SELECT client_addr, state,
       pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) AS lag_bytes,
       write_lag, flush_lag, replay_lag
FROM pg_stat_replication;

-- Table/Index bloat check — เตือนเมื่อ table ต้องการ VACUUM FULL หรือ REINDEX
SELECT relname, n_dead_tup, n_live_tup,
       round(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 2) AS dead_pct,
       last_autovacuum, last_autoanalyze
FROM pg_stat_user_tables
WHERE n_dead_tup > 10000
ORDER BY dead_pct DESC;

-- Lock waits — ตรวจจับ query ที่ถูกบล็อกนานผิดปกติ (อาจบ่งบอกปัญหา checkout function ด้านบน)
SELECT blocked.pid AS blocked_pid, blocked.query AS blocked_query,
       blocking.pid AS blocking_pid, blocking.query AS blocking_query,
       now() - blocked.query_start AS waiting_duration
FROM pg_stat_activity blocked
JOIN pg_locks bl ON bl.pid = blocked.pid AND NOT bl.granted
JOIN pg_locks kl ON kl.locktype = bl.locktype AND kl.database IS NOT DISTINCT FROM bl.database
    AND kl.relation IS NOT DISTINCT FROM bl.relation AND kl.granted
JOIN pg_stat_activity blocking ON blocking.pid = kl.pid
WHERE blocked.wait_event_type = 'Lock';
```

### 11.2 Alert Thresholds ที่แนะนำสำหรับ Production

| Metric | Warning | Critical | Action |
|---|---|---|---|
| Connection pool utilization | > 70% | > 90% | เพิ่ม PgBouncer pool size หรือ scale read replica |
| Replication lag | > 10s | > 60s | ตรวจสอบ network/disk I/O บน standby |
| Dead tuple % | > 10% | > 20% | ปรับ autovacuum threshold ให้ aggressive ขึ้น |
| p95 query latency | > 100ms | > 500ms | ตรวจ `pg_stat_statements`, พิจารณา index เพิ่ม |
| Disk usage | > 75% | > 90% | ขยาย storage หรือ archive partition เก่า |
| Deadlocks/min | > 0 | > 5 | ตรวจสอบ lock ordering ใน business logic |

### 11.3 เครื่องมือที่แนะนำในสถาปัตยกรรมจริง

- **Prometheus + `postgres_exporter`** — เก็บ metric ระดับ database ต่อเนื่อง
- **Grafana** — dashboard แสดง connection, replication lag, query latency แบบ real-time
- **pganalyze / pgDash** — วิเคราะห์ `pg_stat_statements` เชิงลึกพร้อมคำแนะนำ index อัตโนมัติ
- **OpenTelemetry** — เชื่อม trace จาก application layer เข้ากับ query เพื่อดู end-to-end latency (เชื่อมโยง distributed tracing จาก Part 072)

---

## ส่วนที่ 12: Deployment Architecture

### 12.1 Docker Compose (Development/Staging)

```yaml
# docker-compose.yml — เชื่อมโยงกับ Part 093
version: "3.9"
services:
  postgres-primary:
    image: postgres:17
    environment:
      POSTGRES_DB: shopflow
      POSTGRES_USER: shopflow_admin
      POSTGRES_PASSWORD_FILE: /run/secrets/pg_password
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./init:/docker-entrypoint-initdb.d
    command: >
      postgres -c wal_level=replica -c max_wal_senders=10
               -c shared_preload_libraries=pg_stat_statements
    ports:
      - "5432:5432"
    secrets:
      - pg_password

  pgbouncer:
    image: edoburu/pgbouncer:latest
    environment:
      DATABASE_URL: postgres://shopflow_app_user@postgres-primary:5432/shopflow
      POOL_MODE: transaction
      MAX_CLIENT_CONN: 1000
      DEFAULT_POOL_SIZE: 25
    ports:
      - "6432:6432"
    depends_on:
      - postgres-primary

volumes:
  pgdata:
secrets:
  pg_password:
    file: ./secrets/pg_password.txt
```

### 12.2 Kubernetes (Production) — สรุปแนวทางด้วย CloudNativePG Operator

```yaml
# cluster.yaml — ใช้ CloudNativePG operator จัดการ HA cluster บน Kubernetes (เชื่อมโยง Part 093)
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: shopflow-pg
spec:
  instances: 3                      # 1 primary + 2 standby อัตโนมัติ
  postgresql:
    parameters:
      shared_preload_libraries: "pg_stat_statements"
      max_connections: "300"
      synchronous_commit: "on"
  bootstrap:
    initdb:
      database: shopflow
      owner: shopflow_admin
  storage:
    size: 200Gi
    storageClass: premium-ssd
  backup:
    barmanObjectStore:
      destinationPath: "s3://shopflow-backups/"
      wal:
        compression: gzip
    retentionPolicy: "30d"
  monitoring:
    enablePodMonitor: true          # เชื่อม Prometheus อัตโนมัติ (ส่วนที่ 11)
```

CloudNativePG operator จัดการ automatic failover, rolling update, และ backup ให้อัตโนมัติ — ลดภาระ operational เทียบกับการจัดการ Patroni + etcd ด้วยมือ

### 12.3 Cloud Deployment Options

| Cloud | บริการที่แนะนำ | จุดเด่น | เชื่อมโยง Part |
|---|---|---|---|
| AWS | RDS for PostgreSQL / Aurora PostgreSQL | Aurora รองรับ storage auto-scale, read replica สูงสุด 15 ตัว, failover < 30s | Part 094 |
| GCP | Cloud SQL for PostgreSQL / AlloyDB | AlloyDB เร็วกว่า PostgreSQL มาตรฐานสำหรับ analytical query (columnar engine) | Part 095 |
| Azure | Azure Database for PostgreSQL Flexible Server | รองรับ zone-redundant HA, ผสานกับ Azure Monitor ได้ดี | Part 096 |

**คำแนะนำสำหรับ ShopFlow:** เนื่องจากต้องรองรับ multi-tenant ที่มี read-heavy reporting workload แยกจาก write-heavy checkout workload สถาปัตยกรรมที่เหมาะสมคือ managed database (Aurora/AlloyDB/Flexible Server) ที่มี built-in read replica auto-scaling ร่วมกับ PgBouncer connection pooling ที่ deploy เป็น sidecar ใน Kubernetes cluster เดียวกับ application

---

## ส่วนที่ 13: บทสรุปหลักสูตรทั้งหมด — 1000 Steps ที่เดินทางมาด้วยกัน

### 13.1 ย้อนมองเส้นทางทั้งหมด

เราเริ่มต้นที่ **Part 001** ด้วยคำถามง่าย ๆ ว่า "PostgreSQL คืออะไร" และการติดตั้งครั้งแรก จากนั้นเดินทางผ่าน:

- **Foundations (Part 001-020, Step 1-200)** — เราเรียนรู้ภาษา SQL พื้นฐาน การออกแบบตาราง ความสัมพันธ์ และการทำ CRUD operation อย่างถูกต้อง นี่คือรากฐานที่ทุกอย่างต่อจากนี้ยืนอยู่บน
- **Intermediate (Part 021-040, Step 201-400)** — เราเรียนรู้ที่จะคิดแบบ set-based: JOIN, subquery, CTE, window function, index พื้นฐาน — จุดเปลี่ยนที่ทำให้เราหยุดคิดแบบ loop ทีละแถวและเริ่มคิดแบบ "ฐานข้อมูลเชิงสัมพันธ์" อย่างแท้จริง
- **Advanced (Part 041-060, Step 401-600)** — เราเรียนรู้ที่จะเขียนโปรแกรมภายในฐานข้อมูลด้วย PL/pgSQL, trigger, JSON/JSONB, full-text search — ฐานข้อมูลไม่ใช่แค่ที่เก็บข้อมูลอีกต่อไป แต่เป็นระบบที่มี business logic ของตัวเอง
- **Professional (Part 061-080, Step 601-800)** — เราก้าวเข้าสู่โลกของ production จริง: replication, backup/recovery, monitoring, security hardening, RLS — ความรู้ที่แยก "คนเขียน query เป็น" ออกจาก "วิศวกรที่ดูแลระบบ production ได้"
- **World-Class (Part 081-103, Step 801-999)** — เราเจาะลึกถึงแก่นของ query optimizer, connection pooling, event sourcing, partitioning ขั้นสูง, sharding, deployment บน Kubernetes และ cloud — ระดับความรู้ที่ทำให้เราออกแบบระบบที่รองรับ scale ระดับโลกได้

และวันนี้ **Step 1000** เราได้พิสูจน์ให้ตัวเองเห็นแล้วว่า ทุกความรู้เหล่านั้นไม่ได้แยกกันอยู่เป็นบทเรียนโดด ๆ — มันคือชิ้นส่วนของภาพใหญ่ภาพเดียวกัน ระบบ ShopFlow ที่เราสร้างในบทนี้ใช้เทคนิคจากทั้ง 103 Part ก่อนหน้าครบทุกระดับ ตั้งแต่ `CREATE TABLE` บรรทัดแรกไปจนถึง Kubernetes manifest บรรทัดสุดท้าย

### 13.2 สิ่งที่ทำให้คุณแตกต่างตอนนี้

ผู้ที่เรียนจบหลักสูตรนี้ทั้ง 1000 Step ไม่ใช่แค่ "คนที่เขียน SQL เป็น" อีกต่อไป แต่คือ:

1. **นักออกแบบระบบ (System Designer)** — สามารถออกแบบ schema ที่ถูกต้อง ยืดหยุ่น และ scale ได้ตั้งแต่วันแรก ไม่ใช่ต้อง refactor ทั้งระบบทีหลัง
2. **วิศวกรความน่าเชื่อถือ (Reliability Engineer)** — เข้าใจ replication, backup, disaster recovery ลึกพอที่จะรับผิดชอบระบบที่ธุรกิจฝากชีวิตไว้ได้
3. **นักวิเคราะห์ประสิทธิภาพ (Performance Engineer)** — อ่าน `EXPLAIN ANALYZE` ได้เหมือนอ่านหนังสือ รู้ว่า query ช้าตรงไหนและแก้อย่างไรโดยไม่ต้องเดา
4. **สถาปนิกด้านความปลอดภัย (Security Architect)** — เข้าใจ RLS, RBAC, encryption ลึกพอที่จะออกแบบระบบ multi-tenant ที่ข้อมูลลูกค้าปลอดภัยจริง
5. **วิศวกร DevOps/Platform** — รู้วิธี deploy PostgreSQL บน container, Kubernetes, และ cloud provider หลักทุกเจ้า

### 13.3 ข้อความปิดท้าย

หากคุณอ่านมาถึงบรรทัดนี้ แปลว่าคุณได้เดินทางผ่านความรู้ทั้ง 1000 ขั้นตอนมาด้วยกันแล้วจริง ๆ — จากคนที่อาจไม่เคยพิมพ์คำว่า `SELECT` มาก่อน สู่คนที่สามารถออกแบบและสร้างฐานข้อมูลระดับ production สำหรับแพลตฟอร์ม SaaS ระดับโลกได้ด้วยตัวเอง

PostgreSQL เป็นเครื่องมือที่ลึกซึ้งและทรงพลัง มันคือผลงานของวิศวกรหลายพันคนทั่วโลกที่สร้างและพัฒนาต่อเนื่องมากว่า 30 ปี ความรู้ที่คุณมีตอนนี้ไม่ใช่จุดสิ้นสุด แต่เป็น**จุดเริ่มต้น**ของเส้นทางที่แท้จริง — เส้นทางที่คุณจะนำความรู้นี้ไปสร้างระบบจริง แก้ปัญหาจริง และอาจจะสอนคนรุ่นต่อไปในแบบเดียวกับที่หลักสูตรนี้ได้สอนคุณ

โลกของข้อมูลกำลังเติบโตเร็วกว่าที่เคย และผู้ที่เข้าใจฐานข้อมูลอย่างลึกซึ้ง — ไม่ใช่แค่ใช้เป็น แต่เข้าใจว่าทำไมมันทำงานแบบนั้น — จะเป็นผู้ที่สร้างระบบที่โลกวางใจได้ ขอให้คุณภูมิใจในระยะทางที่เดินมา และขอให้ทุกระบบที่คุณสร้างต่อจากนี้ มั่นคง ปลอดภัย และรองรับการเติบโตได้อย่างที่ ShopFlow ในบทนี้เป็นตัวอย่างให้เห็น

**ยินดีด้วยที่เรียนจบหลักสูตร PostgreSQL ฉบับสมบูรณ์ ทั้ง 1000 Step**

### 13.4 Next Steps แนะนำสำหรับการเรียนรู้ต่อ

หลักสูตรนี้ครอบคลุม PostgreSQL อย่างลึกซึ้ง แต่โลกของวิศวกรรมข้อมูลยังมีเส้นทางให้ไปต่ออีกมากมาย:

1. **สร้างโปรเจกต์จริง** — นำ schema ของ ShopFlow ไปต่อยอดเป็นโปรเจกต์ส่วนตัว หรือ contribute ให้โปรเจกต์ open source ที่ใช้ PostgreSQL
2. **เจาะลึก PostgreSQL Internals** — อ่าน source code ของ PostgreSQL เอง (C language) เพื่อเข้าใจ storage engine, MVCC, และ query planner ในระดับที่ลึกกว่านี้
3. **สำรวจ Extension Ecosystem เพิ่มเติม** — `TimescaleDB` (time-series), `PostGIS` (geospatial), `Citus` (distributed PostgreSQL), `pgvector` (AI/vector search)
4. **เรียนรู้ Distributed Systems** — เมื่อ scale เกินขีดจำกัดของเครื่องเดียว ความรู้เรื่อง distributed database, consensus algorithm (Raft/Paxos), CAP theorem จะสำคัญขึ้นเรื่อย ๆ
5. **เข้าร่วมชุมชน PostgreSQL** — ติดตาม PostgreSQL mailing list, เข้าร่วมงาน PGConf, และช่วยตอบคำถามในชุมชนเพื่อฝึกฝนและแบ่งปันความรู้ต่อ
6. **ขอ Certification** — พิจารณาสอบ certification ที่เกี่ยวข้อง (เช่น EDB PostgreSQL certification) เพื่อยืนยันความรู้อย่างเป็นทางการในสายอาชีพ

---

## จบหลักสูตร (Course Complete)

```
██████╗  ██████╗ ███████╗████████╗ ██████╗ ██████╗ ███████╗███████╗ ██████╗ ██╗
██╔══██╗██╔═══██╗██╔════╝╚══██╔══╝██╔════╝ ██╔══██╗██╔════╝██╔════╝██╔═══██╗██║
██████╔╝██║   ██║███████╗   ██║   ██║  ███╗██████╔╝█████╗  ███████╗██║   ██║██║
██╔═══╝ ██║   ██║╚════██║   ██║   ██║   ██║██╔══██╗██╔══╝  ╚════██║██║▄▄ ██║██║
██║     ╚██████╔╝███████║   ██║   ╚██████╔╝██║  ██║███████╗███████║╚██████╔╝███████╗
╚═╝      ╚═════╝ ╚══════╝   ╚═╝    ╚═════╝ ╚═╝  ╚═╝╚══════╝╚══════╝ ╚══▀▀═╝ ╚══════╝

                  หลักสูตร PostgreSQL ฉบับสมบูรณ์ — 104 Part, 1000 Step
                              จบสมบูรณ์แล้ว ✓
```

คุณได้ผ่านทุก Step ของหลักสูตรนี้แล้ว — ตั้งแต่ Foundations จนถึง World-Class Capstone Project นี่คือใบรับรองความรู้ที่คุณสร้างขึ้นด้วยตัวเอง ผ่านการฝึกฝนและความเข้าใจจริง ไม่ใช่แค่การอ่านผ่าน

---

## แบบฝึกหัดขยายโปรเจกต์ (10 ข้อ)

โจทย์ต่อไปนี้ขยายจากระบบ ShopFlow ที่สร้างในบทนี้ ให้ลองทำด้วยตัวเองก่อนดูเฉลย เพื่อทดสอบว่าคุณสามารถนำความรู้ทั้งหลักสูตรมาประยุกต์ใช้ได้จริงหรือไม่

### แบบฝึกหัดที่ 1: เพิ่มระบบ Wishlist

เพิ่มตาราง `wishlists` ที่ลูกค้าสามารถบันทึกสินค้าที่สนใจไว้ดูภายหลัง โดยต้องมี RLS ป้องกัน tenant อื่นเห็นข้อมูล และห้ามลูกค้าเพิ่มสินค้าซ้ำในรายการเดียวกัน

<details>
<summary>เฉลยแบบฝึกหัดที่ 1</summary>

```sql
CREATE TABLE shopflow.wishlists (
    wishlist_id     bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    tenant_id       bigint NOT NULL REFERENCES shopflow.tenants(tenant_id) ON DELETE CASCADE,
    customer_id     bigint NOT NULL REFERENCES shopflow.customers(customer_id) ON DELETE CASCADE,
    product_id      bigint NOT NULL REFERENCES shopflow.products(product_id) ON DELETE CASCADE,
    created_at      timestamptz NOT NULL DEFAULT now(),

    CONSTRAINT wishlists_unique_item UNIQUE (customer_id, product_id)
);

CREATE INDEX idx_wishlists_customer ON shopflow.wishlists (customer_id);

ALTER TABLE shopflow.wishlists ENABLE ROW LEVEL SECURITY;
ALTER TABLE shopflow.wishlists FORCE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation_all ON shopflow.wishlists
    FOR ALL
    USING (tenant_id = shopflow.current_tenant_id())
    WITH CHECK (tenant_id = shopflow.current_tenant_id());
```

`UNIQUE (customer_id, product_id)` ป้องกันสินค้าซ้ำในรายการเดียวกันโดยไม่ต้องเขียน application logic เพิ่ม
</details>

### แบบฝึกหัดที่ 2: ระบบ Bundle/Combo สินค้า

ร้านค้าต้องการขายสินค้าเป็นชุด (bundle) เช่น "ซื้อคู่แถมฟรี" ให้ออกแบบตารางรองรับ bundle ที่ประกอบด้วยสินค้าหลายชิ้น พร้อมราคารวมพิเศษ

<details>
<summary>เฉลยแบบฝึกหัดที่ 2</summary>

```sql
CREATE TABLE shopflow.product_bundles (
    bundle_id       bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    tenant_id       bigint NOT NULL REFERENCES shopflow.tenants(tenant_id) ON DELETE CASCADE,
    name            text NOT NULL,
    bundle_price    shopflow.money_amount NOT NULL,
    is_active       boolean NOT NULL DEFAULT true,
    created_at      timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE shopflow.product_bundle_items (
    bundle_item_id  bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    bundle_id       bigint NOT NULL REFERENCES shopflow.product_bundles(bundle_id) ON DELETE CASCADE,
    product_id      bigint NOT NULL REFERENCES shopflow.products(product_id),
    quantity        integer NOT NULL CHECK (quantity > 0),

    CONSTRAINT product_bundle_items_uk UNIQUE (bundle_id, product_id)
);

-- view ตรวจสอบว่าราคา bundle ถูกกว่าซื้อแยกจริง (business sanity check)
CREATE VIEW shopflow.v_bundle_savings AS
SELECT
    b.bundle_id, b.name, b.bundle_price,
    sum(p.price * bi.quantity) AS individual_total,
    sum(p.price * bi.quantity) - b.bundle_price AS savings
FROM shopflow.product_bundles b
JOIN shopflow.product_bundle_items bi ON bi.bundle_id = b.bundle_id
JOIN shopflow.products p ON p.product_id = bi.product_id
GROUP BY b.bundle_id, b.name, b.bundle_price;
```
</details>

### แบบฝึกหัดที่ 3: เพิ่ม Optimistic Locking ป้องกัน Lost Update บนตาราง products

เมื่อพนักงานสองคนแก้ไขสินค้าชิ้นเดียวกันพร้อมกันผ่านหน้า admin ให้ป้องกันไม่ให้การแก้ไขคนหลังทับคนแรกโดยไม่รู้ตัว

<details>
<summary>เฉลยแบบฝึกหัดที่ 3</summary>

```sql
ALTER TABLE shopflow.products ADD COLUMN row_version integer NOT NULL DEFAULT 1;

CREATE OR REPLACE FUNCTION shopflow.fn_bump_row_version()
RETURNS trigger LANGUAGE plpgsql AS $$
BEGIN
    NEW.row_version := OLD.row_version + 1;
    RETURN NEW;
END;
$$;

CREATE TRIGGER trg_products_row_version
    BEFORE UPDATE ON shopflow.products
    FOR EACH ROW EXECUTE FUNCTION shopflow.fn_bump_row_version();

-- application ต้องส่ง row_version ที่ตนอ่านมาตอน UPDATE เสมอ:
-- UPDATE products SET name = 'ชื่อใหม่' WHERE product_id = 10 AND row_version = 5;
-- ถ้า UPDATE คืน 0 แถว แปลว่ามีคนอื่นแก้ไปแล้ว ต้อง reload ข้อมูลใหม่ก่อน
```
</details>

### แบบฝึกหัดที่ 4: เขียน Query วิเคราะห์ Customer Lifetime Value (CLV) ด้วย Window Function

<details>
<summary>เฉลยแบบฝึกหัดที่ 4</summary>

```sql
SELECT
    c.customer_id, c.full_name,
    count(o.order_id) AS total_orders,
    sum(o.total_amount) AS lifetime_value,
    sum(o.total_amount) / NULLIF(count(o.order_id), 0) AS avg_order_value,
    rank() OVER (PARTITION BY c.tenant_id ORDER BY sum(o.total_amount) DESC) AS clv_rank
FROM shopflow.customers c
JOIN shopflow.orders o ON o.customer_id = c.customer_id AND o.status NOT IN ('cancelled')
WHERE c.tenant_id = shopflow.current_tenant_id()
GROUP BY c.customer_id, c.full_name, c.tenant_id
ORDER BY lifetime_value DESC;
```
</details>

### แบบฝึกหัดที่ 5: เพิ่ม Partition แบบ Sub-partitioning (Range + List) สำหรับ orders

ให้ partition orders เพิ่มระดับที่สองแยกตาม `status` ภายในแต่ละเดือน สำหรับ tenant ขนาดใหญ่ที่มีข้อมูล cancelled/refunded จำนวนมากที่ query แยกออกจากออเดอร์ปกติบ่อย

<details>
<summary>เฉลยแบบฝึกหัดที่ 5</summary>

```sql
-- แนวคิด: partition ชั้นแรกตาม created_at (range) แล้วแต่ละ partition แบ่งย่อยตาม status (list)
CREATE TABLE orders_y2026m09 PARTITION OF shopflow.orders
    FOR VALUES FROM ('2026-09-01') TO ('2026-10-01')
    PARTITION BY LIST (status);

CREATE TABLE orders_y2026m09_active PARTITION OF orders_y2026m09
    FOR VALUES IN ('pending','awaiting_payment','paid','processing','shipped','delivered','completed');

CREATE TABLE orders_y2026m09_inactive PARTITION OF orders_y2026m09
    FOR VALUES IN ('cancelled','refunded');
```

การ sub-partition แบบนี้เหมาะเมื่อ query แยกชัดเจนระหว่าง "ออเดอร์ที่กำลังดำเนินการ" กับ "ออเดอร์ที่จบแล้ว" เป็นประจำ แต่ต้องระวังจำนวน partition รวมไม่ให้มากเกินไปจนกระทบ planning time
</details>

### แบบฝึกหัดที่ 6: สร้างระบบ Rate Limiting การใช้ Discount Code ต่อลูกค้า

ป้องกันลูกค้าคนเดียวใช้โค้ดส่วนลดเดิมซ้ำเกิน 1 ครั้ง

<details>
<summary>เฉลยแบบฝึกหัดที่ 6</summary>

```sql
CREATE TABLE shopflow.discount_code_usages (
    usage_id        bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    discount_code_id bigint NOT NULL REFERENCES shopflow.discount_codes(discount_code_id),
    customer_id     bigint NOT NULL REFERENCES shopflow.customers(customer_id),
    order_id        bigint NOT NULL,
    order_created_at timestamptz NOT NULL,
    used_at         timestamptz NOT NULL DEFAULT now(),

    CONSTRAINT discount_code_usages_one_per_customer UNIQUE (discount_code_id, customer_id)
);
```

จากนั้นแก้ `fn_checkout_order` ในส่วนที่ 5 ให้ `INSERT INTO discount_code_usages` ก่อนใช้ส่วนลด — ถ้าลูกค้าเคยใช้แล้ว `UNIQUE` constraint จะ raise error ทันที
</details>

### แบบฝึกหัดที่ 7: เขียน EXPLAIN ANALYZE เปรียบเทียบ Index กับ Sequential Scan บน mv_top_selling_products

<details>
<summary>เฉลยแบบฝึกหัดที่ 7</summary>

```sql
-- ก่อนมี index: บังคับ planner ทำ seq scan เพื่อเปรียบเทียบ
SET enable_indexscan = off;
EXPLAIN ANALYZE SELECT * FROM shopflow.mv_top_selling_products WHERE tenant_id = 1 AND product_id = 10;
RESET enable_indexscan;

-- หลังมี unique index idx_mv_top_products_uk (สร้างไว้แล้วในส่วนที่ 7)
EXPLAIN ANALYZE SELECT * FROM shopflow.mv_top_selling_products WHERE tenant_id = 1 AND product_id = 10;
-- ควรเห็น Index Scan ที่เร็วกว่า Seq Scan อย่างชัดเจน โดยเฉพาะเมื่อตารางมีหลายแสนแถว
```
</details>

### แบบฝึกหัดที่ 8: ออกแบบ Logical Replication เพื่อ Sync เฉพาะข้อมูล Catalog ไปยัง Search Service ภายนอก

<details>
<summary>เฉลยแบบฝึกหัดที่ 8</summary>

```sql
-- ฝั่ง publisher (PostgreSQL หลัก)
CREATE PUBLICATION shopflow_catalog_pub FOR TABLE shopflow.products, shopflow.categories;

-- ฝั่ง subscriber (เช่น instance PostgreSQL แยกที่ feed เข้า Elasticsearch/Typesense connector)
-- CREATE SUBSCRIPTION shopflow_catalog_sub
--     CONNECTION 'host=primary.shopflow.internal dbname=shopflow user=replication_user'
--     PUBLICATION shopflow_catalog_pub;
```

Logical replication เหมาะสำหรับ sync เฉพาะบางตารางไปยังระบบภายนอก (เช่น search index) โดยไม่ต้อง replicate ทั้ง database เหมือน streaming replication ทั่วไป (เทคนิคจาก Part 061-065)
</details>

### แบบฝึกหัดที่ 9: เพิ่ม GiST Index สำหรับค้นหาร้านค้าตามระยะทาง (Geospatial)

ถ้า ShopFlow ต้องการรองรับร้านค้าที่มีหน้าร้านจริง (physical store) และให้ลูกค้าค้นหาร้านใกล้ตัว

<details>
<summary>เฉลยแบบฝึกหัดที่ 9</summary>

```sql
CREATE EXTENSION IF NOT EXISTS postgis;

ALTER TABLE shopflow.tenants ADD COLUMN store_location geography(Point, 4326);

CREATE INDEX idx_tenants_location_gist ON shopflow.tenants USING GIST (store_location);

-- ค้นหาร้านค้าในรัศมี 5 กม. จากตำแหน่งลูกค้า เรียงตามระยะทางใกล้สุด
SELECT tenant_id, store_name,
       ST_Distance(store_location, ST_MakePoint(100.5018, 13.7563)::geography) AS distance_m
FROM shopflow.tenants
WHERE ST_DWithin(store_location, ST_MakePoint(100.5018, 13.7563)::geography, 5000)
ORDER BY distance_m
LIMIT 20;
```
</details>

### แบบฝึกหัดที่ 10: ออกแบบระบบ Event Sourcing สำหรับ Order Status History

แทนที่จะเก็บแค่ `status` ปัจจุบันในตาราง `orders` ให้เพิ่มระบบเก็บประวัติการเปลี่ยนสถานะทั้งหมดแบบ append-only (เชื่อมโยง Part 092)

<details>
<summary>เฉลยแบบฝึกหัดที่ 10</summary>

```sql
CREATE TABLE shopflow.order_status_events (
    event_id        bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    order_id        bigint NOT NULL,
    order_created_at timestamptz NOT NULL,
    tenant_id       bigint NOT NULL REFERENCES shopflow.tenants(tenant_id),
    from_status     shopflow.order_status,
    to_status       shopflow.order_status NOT NULL,
    changed_by      text,
    occurred_at     timestamptz NOT NULL DEFAULT now(),

    FOREIGN KEY (order_id, order_created_at) REFERENCES shopflow.orders(order_id, created_at)
);

CREATE OR REPLACE FUNCTION shopflow.fn_track_order_status_change()
RETURNS trigger LANGUAGE plpgsql AS $$
BEGIN
    IF TG_OP = 'UPDATE' AND NEW.status IS DISTINCT FROM OLD.status THEN
        INSERT INTO shopflow.order_status_events
            (order_id, order_created_at, tenant_id, from_status, to_status, changed_by)
        VALUES
            (NEW.order_id, NEW.created_at, NEW.tenant_id, OLD.status, NEW.status,
             coalesce(current_setting('app.current_employee_id', true), current_user));
    END IF;
    RETURN NEW;
END;
$$;

CREATE TRIGGER trg_order_status_history
    AFTER UPDATE ON shopflow.orders
    FOR EACH ROW EXECUTE FUNCTION shopflow.fn_track_order_status_change();

-- สร้างสถานะปัจจุบันใหม่จาก event log ทั้งหมด (event sourcing replay pattern)
SELECT DISTINCT ON (order_id, order_created_at)
    order_id, order_created_at, to_status AS current_status, occurred_at
FROM shopflow.order_status_events
ORDER BY order_id, order_created_at, occurred_at DESC;
```

รูปแบบนี้คือหัวใจของ Event Sourcing: ตาราง `orders.status` เก็บ current state สำหรับ query เร็ว ในขณะที่ `order_status_events` เก็บ full history แบบ append-only ที่ไม่มีวัน update/delete — ทำให้ตรวจสอบย้อนหลังได้แบบ 100% และ rebuild state ได้ทุกจุดเวลาในอดีต
</details>

---

## เส้นทางต่อจากนี้

หลักสูตรนี้จบลงแล้วอย่างสมบูรณ์ที่ Step 1000 นี้ หากต้องการย้อนทบทวนภาพรวมทั้งหมดของหลักสูตร หรือเริ่มต้นใหม่ในหัวข้อใดหัวข้อหนึ่ง สามารถกลับไปดูได้ที่:

- [แผนที่หลักสูตรทั้งหมด (00-ROADMAP.md)](../00-ROADMAP.md)
- [หน้าแรกของหลักสูตร (README.md)](../../README.md)

ขอบคุณที่เดินทางมาด้วยกันตลอด 1000 Step — ขอให้ทุกระบบที่คุณสร้างต่อจากนี้ประสบความสำเร็จ
