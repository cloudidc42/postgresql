# Part 091: Database Design Patterns สำหรับสถาปัตยกรรม Microservices

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับโลก/ผู้เชี่ยวชาญ | Part 091

---

## เป้าหมายการเรียนรู้

เมื่อเรียนจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายความแตกต่างระหว่างการออกแบบฐานข้อมูลสำหรับ monolith และสำหรับ microservices ได้อย่างชัดเจน
2. เข้าใจและนำ **Database per Service Pattern** ไปใช้ พร้อมอธิบายเหตุผลเชิง loose coupling
3. วิเคราะห์และรับมือกับปัญหาที่เกิดขึ้นเมื่อไม่มี JOIN ข้าม service และไม่มี distributed transaction แบบง่ายๆ อีกต่อไป
4. ระบุและหลีกเลี่ยง **Shared Database Anti-pattern**
5. ออกแบบและ implement **API Composition Pattern** สำหรับการรวมข้อมูลข้าม service
6. ออกแบบ **Saga Pattern** ทั้งแบบ Choreography และ Orchestration พร้อม compensating transaction
7. Implement **Outbox Pattern** ด้วย PostgreSQL เพื่อรับประกัน reliable event publishing
8. เชื่อมโยง **Change Data Capture (CDC)** เข้ากับ logical replication (ทบทวน Part 064) และ Debezium
9. ออกแบบสถาปัตยกรรมฐานข้อมูล microservices แบบเต็มรูปแบบสำหรับระบบ e-commerce จริง

**Domain ตัวอย่างที่ใช้ตลอดบทนี้:** ระบบ e-commerce ที่แยกเป็น 4 microservices คือ `order-service`, `inventory-service`, `customer-service`, และ `payment-service` — แต่ละตัวมีฐานข้อมูล PostgreSQL ของตัวเอง

---

## Step 901: Monolith เทียบกับ Microservices — ผลกระทบต่อการออกแบบฐานข้อมูล

### 901.1 สถาปัตยกรรม Monolith แบบดั้งเดิม

ในสถาปัตยกรรม monolith ทั้งระบบ (order, inventory, customer, payment) มักใช้ฐานข้อมูล PostgreSQL **ตัวเดียว** ที่มีทุกตารางอยู่รวมกัน:

```
┌─────────────────────────────────────────────────────┐
│                  Monolith Application                │
│  ┌───────────┐ ┌───────────┐ ┌───────────┐          │
│  │  Order     │ │ Inventory │ │ Customer  │  ...     │
│  │  Module    │ │  Module   │ │  Module   │          │
│  └─────┬─────┘ └─────┬─────┘ └─────┬─────┘          │
└────────┼─────────────┼─────────────┼─────────────────┘
         │             │             │
         └─────────────┼─────────────┘
                        ▼
              ┌───────────────────┐
              │  PostgreSQL (1 DB) │
              │  ┌───────────┐    │
              │  │  orders    │    │
              │  │  order_items│   │
              │  │  products   │   │
              │  │  inventory  │   │
              │  │  customers  │   │
              │  │  payments   │   │
              │  └───────────┘    │
              └───────────────────┘
```

**ข้อดีของ monolith database:**

- JOIN ข้ามตารางได้อย่างอิสระ เช่น รายงานยอดขายที่ join `orders`, `customers`, `products` ในคำสั่งเดียว
- Transaction เดียวครอบคลุมหลายตาราง (ACID เต็มรูปแบบ) — เช่น การสั่งซื้อที่ต้องตัด stock พร้อมกับสร้าง order ใน transaction เดียวกัน
- Schema เดียว ง่ายต่อการดูแล constraint, foreign key ข้าม domain
- Consistency แบบ strong consistency ทันที (read-your-writes เสมอ)

```sql
-- ตัวอย่าง transaction เดียวใน monolith ที่ทำได้ง่ายมาก
BEGIN;

INSERT INTO orders (customer_id, status, total_amount)
VALUES (1001, 'PENDING', 1500.00)
RETURNING id;

UPDATE inventory
SET quantity = quantity - 2
WHERE product_id = 55 AND quantity >= 2;

UPDATE customers
SET loyalty_points = loyalty_points + 15
WHERE id = 1001;

COMMIT;
```

โค้ดข้างบนนี้ทำงานได้อย่างสวยงามเพราะทุกตารางอยู่ในฐานข้อมูลเดียวกัน PostgreSQL รับประกัน atomicity ให้ทั้งหมดโดยอัตโนมัติ

### 901.2 เมื่อ Monolith เติบโตจนกลายเป็นปัญหา

เมื่อทีมพัฒนาขยายใหญ่ขึ้น (หลายสิบถึงหลายร้อยคน) monolith เริ่มมีปัญหา:

- **Deploy coupling**: ทุกทีมต้อง deploy พร้อมกัน แก้ inventory module นิดเดียวก็ต้อง deploy ทั้งระบบ
- **Scaling ไม่ยืดหยุ่น**: inventory-service อาจต้องการ scale มากกว่า customer-service มาก แต่ scale แยกกันไม่ได้เพราะรวมกันอยู่
- **Schema contention**: หลายทีมแก้ schema เดียวกัน migration ชนกันบ่อย
- **Blast radius ใหญ่**: bug ใน module หนึ่งอาจทำให้ connection pool หมดและกระทบทุก module
- **Technology lock-in**: ทุกทีมต้องใช้ PostgreSQL เดียวกัน เวอร์ชันเดียวกัน แม้ workload ต่างกันมาก

### 901.3 สถาปัตยกรรม Microservices

แนวทาง microservices แยกแต่ละ business capability ออกเป็น service อิสระ พร้อมฐานข้อมูลของตัวเอง:

```
┌───────────────┐   ┌───────────────┐   ┌───────────────┐   ┌───────────────┐
│ order-service │   │inventory-svc  │   │customer-svc   │   │ payment-svc   │
│  (REST/gRPC)  │   │  (REST/gRPC)  │   │  (REST/gRPC)  │   │  (REST/gRPC)  │
└───────┬───────┘   └───────┬───────┘   └───────┬───────┘   └───────┬───────┘
        │                   │                   │                   │
        ▼                   ▼                   ▼                   ▼
┌───────────────┐   ┌───────────────┐   ┌───────────────┐   ┌───────────────┐
│  order_db     │   │  inventory_db │   │  customer_db  │   │  payment_db   │
│  (PostgreSQL) │   │  (PostgreSQL) │   │  (PostgreSQL) │   │  (PostgreSQL) │
│  - orders     │   │  - products   │   │  - customers  │   │  - payments   │
│  - order_items│   │  - stock      │   │  - addresses  │   │  - refunds    │
└───────────────┘   └───────────────┘   └───────────────┘   └───────────────┘
```

**ผลกระทบต่อการออกแบบฐานข้อมูลที่สำคัญที่สุด 4 ข้อ:**

| ประเด็น | Monolith | Microservices |
|---|---|---|
| Transaction ข้าม domain | ACID เต็มรูปแบบใน 1 transaction | ต้องใช้ Saga / eventual consistency |
| JOIN ข้าม domain | `JOIN` SQL ปกติ | ต้องใช้ API Composition หรือ data replication |
| Consistency | Strong consistency | ส่วนใหญ่เป็น eventual consistency |
| Schema ownership | ทีมกลาง / DBA กลาง | แต่ละทีมเป็นเจ้าของ schema ของตัวเอง |
| Scaling | Scale ทั้งฐานข้อมูลพร้อมกัน | Scale แยกอิสระตาม service |
| Failure isolation | ล้มพร้อมกันทั้งระบบได้ง่าย | แยก failure domain ได้ |

> **ข้อคิดสำคัญ:** การย้ายจาก monolith ไป microservices ไม่ใช่แค่การแยกโค้ด แต่คือการเปลี่ยนโมเดลความคิดเรื่อง **consistency** จาก "ทุกอย่างต้อง consistent ทันที (strong consistency)" ไปเป็น "ระบบ consistent ในที่สุด (eventual consistency)" — นี่คือหัวใจของทั้งบทนี้

### 901.4 เมื่อไหร่ควรใช้ Microservices Database Pattern

Microservices ไม่ใช่ silver bullet เสมอไป ควรพิจารณาใช้เมื่อ:

- ทีมพัฒนาใหญ่พอที่จะแบ่งความรับผิดชอบตาม domain ได้ชัดเจน (Conway's Law)
- แต่ละ domain มี scaling profile ต่างกันมาก (เช่น inventory read-heavy, payment write-heavy + strict consistency)
- ต้องการ deploy อิสระบ่อยครั้งโดยไม่กระทบ domain อื่น
- มี domain boundary ที่ชัดเจนอยู่แล้ว (DDD Bounded Context)

หากทีมยังเล็ก หรือ domain ยังไม่ชัดเจน **การเริ่มจาก monolith ที่ออกแบบ schema แยกเป็น logical schema ตาม domain** (เช่น `order.orders`, `inventory.products`) แล้วค่อย extract ออกเป็น microservice ทีหลัง มักเป็นทางเลือกที่ปลอดภัยกว่า

---

## Step 902: Database per Service Pattern

### 902.1 นิยามของ Pattern

**Database per Service** คือหลักการที่ว่า **แต่ละ microservice ต้องเป็นเจ้าของฐานข้อมูลของตัวเองแต่เพียงผู้เดียว** และห้าม service อื่นเข้าถึงฐานข้อมูลนั้นโดยตรง (ไม่ว่าจะเป็นการ query, join, หรือแม้แต่ read-only) การเข้าถึงข้อมูลข้าม service ต้องผ่าน API (REST/gRPC/event) เท่านั้น

```
                    ❌ ห้ามทำ (cross-service DB access)
     order-service ──────────X──────────▶ customer_db

                    ✅ ต้องทำแบบนี้
     order-service ──── HTTP/gRPC call ────▶ customer-service ──▶ customer_db
```

### 902.2 ตัวอย่างการ Provision ฐานข้อมูลแยกต่อ Service

ในทางปฏิบัติ แต่ละ service อาจ:
- ใช้ PostgreSQL instance แยกกันคนละเครื่อง/คนละ cluster (isolation สูงสุด)
- ใช้ PostgreSQL instance เดียวกันแต่คนละ **database** (`CREATE DATABASE`)
- ใช้ database เดียวกันแต่คนละ **schema** พร้อม role/permission แยกกันอย่างเข้มงวด (ทางเลือกประหยัดสำหรับทีมเล็ก แต่ต้องมีวินัยสูง)

```sql
-- ตัวอย่าง: provisioning ฐานข้อมูลแยกสำหรับแต่ละ service บน PostgreSQL cluster เดียวกัน
-- (รันโดย DBA/admin role เท่านั้น)

-- 1) order-service
CREATE DATABASE order_db;
CREATE ROLE order_service_app WITH LOGIN PASSWORD 'change_me_in_vault';
GRANT ALL PRIVILEGES ON DATABASE order_db TO order_service_app;

-- 2) inventory-service
CREATE DATABASE inventory_db;
CREATE ROLE inventory_service_app WITH LOGIN PASSWORD 'change_me_in_vault';
GRANT ALL PRIVILEGES ON DATABASE inventory_db TO inventory_service_app;

-- 3) customer-service
CREATE DATABASE customer_db;
CREATE ROLE customer_service_app WITH LOGIN PASSWORD 'change_me_in_vault';
GRANT ALL PRIVILEGES ON DATABASE customer_db TO customer_service_app;

-- 4) payment-service
CREATE DATABASE payment_db;
CREATE ROLE payment_service_app WITH LOGIN PASSWORD 'change_me_in_vault';
GRANT ALL PRIVILEGES ON DATABASE payment_db TO payment_service_app;
```

**สำคัญ:** ห้ามให้ `order_service_app` มีสิทธิ์เข้าถึง `inventory_db` เลยแม้แต่ read-only ในระดับ network ก็ควรบล็อกด้วย `pg_hba.conf` หรือ security group เพื่อบังคับ boundary ในเชิง infrastructure ไม่ใช่แค่ระดับ permission

```
# ตัวอย่างแนวคิด pg_hba.conf ต่อ instance (ถ้าแยก instance กันจริง)
# host    database      user                    address          method
  host    order_db      order_service_app       10.0.1.0/24      scram-sha-256
  host    order_db      all                     0.0.0.0/0        reject
```

### 902.3 แต่ละ Service ออกแบบ Schema ตาม Domain ของตัวเอง

แต่ละ service **เป็นเจ้าของ data model ของตัวเอง** และสามารถออกแบบ schema ให้เหมาะกับ use case ของตัวเองได้อย่างอิสระ โดยไม่ต้องประนีประนอมกับ service อื่น

```sql
-- ========== order_db (order-service) ==========
CREATE TABLE orders (
    id              BIGSERIAL PRIMARY KEY,
    customer_id     BIGINT NOT NULL,        -- เป็นแค่ "reference" ไม่ใช่ FK จริง เพราะ
                                             -- customer อยู่คนละฐานข้อมูล!
    status          TEXT NOT NULL DEFAULT 'PENDING',
    total_amount    NUMERIC(12,2) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE order_items (
    id              BIGSERIAL PRIMARY KEY,
    order_id        BIGINT NOT NULL REFERENCES orders(id),
    product_id      BIGINT NOT NULL,        -- reference ไป inventory-service
    product_name    TEXT NOT NULL,          -- denormalized snapshot ณ เวลาสั่งซื้อ
    unit_price      NUMERIC(12,2) NOT NULL,
    quantity        INT NOT NULL CHECK (quantity > 0)
);
```

```sql
-- ========== inventory_db (inventory-service) ==========
CREATE TABLE products (
    id              BIGSERIAL PRIMARY KEY,
    sku             TEXT NOT NULL UNIQUE,
    name            TEXT NOT NULL,
    price           NUMERIC(12,2) NOT NULL
);

CREATE TABLE stock_levels (
    product_id      BIGINT PRIMARY KEY REFERENCES products(id),
    quantity_available INT NOT NULL DEFAULT 0,
    quantity_reserved   INT NOT NULL DEFAULT 0,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

```sql
-- ========== customer_db (customer-service) ==========
CREATE TABLE customers (
    id              BIGSERIAL PRIMARY KEY,
    email           TEXT NOT NULL UNIQUE,
    full_name       TEXT NOT NULL,
    loyalty_points  INT NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

```sql
-- ========== payment_db (payment-service) ==========
CREATE TABLE payments (
    id              BIGSERIAL PRIMARY KEY,
    order_id        BIGINT NOT NULL,        -- reference ไป order-service
    customer_id     BIGINT NOT NULL,        -- reference ไป customer-service
    amount          NUMERIC(12,2) NOT NULL,
    status          TEXT NOT NULL DEFAULT 'PENDING',
    provider_ref    TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

สังเกตว่า `orders.customer_id` และ `order_items.product_id` **ไม่มี** `FOREIGN KEY` แท้จริงไปยังตารางข้ามฐานข้อมูล — PostgreSQL ไม่รองรับ FK ข้ามฐานข้อมูลอยู่แล้ว และแม้จะทำได้ (เช่นผ่าน trigger หรือ FDW) ก็ **ไม่ควรทำ** เพราะจะทำลายหลักการ loose coupling ของ pattern นี้ทันที

### 902.4 ทำไม Loose Coupling ถึงสำคัญ

Loose coupling ในบริบทฐานข้อมูลหมายถึง service หนึ่งสามารถ:

1. **เปลี่ยน schema ได้อย่างอิสระ** — inventory-service เพิ่มคอลัมน์ `warehouse_id` ได้โดยไม่ต้องแจ้ง order-service ล่วงหน้า (ตราบใดที่ API contract ไม่เปลี่ยน)
2. **เปลี่ยน database engine ได้** — สมมติ inventory-service ต้องการ high write throughput มากจนอยากย้ายไป Cassandra หรือ scale ผ่าน Citus ก็ทำได้โดยไม่กระทบ service อื่นเลย ตราบใดที่ API ยังคงเดิม
3. **Scale อิสระ** — payment-service ที่ต้องการ strict consistency สูง อาจใช้ PostgreSQL แบบ single primary + synchronous replica ในขณะที่ inventory-service ใช้ read replica จำนวนมากเพื่อรองรับ read-heavy traffic
4. **Deploy อิสระ** — migration ของ order-service ไม่ block การ deploy ของ customer-service

```
        Coupling ระดับสูง (ไม่ดี)              Coupling ระดับต่ำ (ดี)
   ┌──────────────────────────┐          ┌──────────────────────────┐
   │ Service A ──┐             │          │ Service A ── API ──▶ Service B │
   │             ▼             │          │                            │
   │         Shared Table      │          │ Service A [DB A]           │
   │             ▲             │          │ Service B [DB B]           │
   │ Service B ──┘             │          │ (แก้ schema แยกกันได้)      │
   └──────────────────────────┘          └──────────────────────────┘
   เปลี่ยน schema กระทบทั้งคู่ทันที         เปลี่ยน schema ภายในกระทบแค่ตัวเอง
```

**บทสรุป Step 902:** Database per Service คือรากฐานของ microservices architecture ทั้งหมด — ถ้าไม่ทำตาม pattern นี้อย่างเคร่งครัด ระบบจะกลายเป็น "distributed monolith" ที่ได้ข้อเสียของทั้งสองโลกแต่ไม่ได้ข้อดีของโลกไหนเลย

---

## Step 903: ปัญหาที่เกิดขึ้นเมื่อแยกฐานข้อมูล

การแยกฐานข้อมูลแก้ปัญหา coupling แต่สร้างปัญหาใหม่ 4 กลุ่มหลักที่ทุกทีมต้องเผชิญ

### 903.1 ปัญหาที่ 1: ไม่มี JOIN ข้าม Service ได้อีกต่อไป

ใน monolith เราสามารถเขียน query แบบนี้ได้ง่ายๆ:

```sql
-- ทำได้ใน monolith (ทุกตารางอยู่ DB เดียวกัน)
SELECT
    o.id AS order_id,
    o.total_amount,
    c.full_name,
    c.email,
    array_agg(oi.product_name) AS items
FROM orders o
JOIN customers c ON c.id = o.customer_id
JOIN order_items oi ON oi.order_id = o.id
WHERE o.created_at >= now() - interval '7 days'
GROUP BY o.id, o.total_amount, c.full_name, c.email;
```

แต่ใน microservices `orders` อยู่ใน `order_db` และ `customers` อยู่ใน `customer_db` คนละ PostgreSQL instance กัน **query นี้เป็นไปไม่ได้เลยในระดับ SQL** ไม่มีทางที่ PostgreSQL instance หนึ่งจะ JOIN ตารางกับอีก instance หนึ่งได้โดยตรง (ยกเว้นใช้ `postgres_fdw` ซึ่งเป็น anti-pattern ตามที่จะกล่าวใน Step 904)

**ทางออก:** ต้องใช้ **API Composition** (Step 905) หรือสร้าง **read model / materialized view** ที่ replicate ข้อมูลที่จำเป็นมาเก็บไว้ใน service ที่ต้องใช้ (ผ่าน event-driven replication)

### 903.2 ปัญหาที่ 2: ไม่มี Distributed Transaction แบบง่ายๆ

ลองพิจารณา flow "checkout" ที่ต้องทำ 3 อย่างให้สำเร็จพร้อมกัน (all-or-nothing):

1. `order-service`: สร้าง order สถานะ `PENDING`
2. `inventory-service`: ตัด stock (reserve)
3. `payment-service`: เรียกเก็บเงิน (charge)

ใน monolith นี่คือ 1 transaction เดียว แต่ใน microservices นี่คือ 3 transaction แยกกันใน 3 ฐานข้อมูล ถ้า step 3 ล้มเหลว เราจะ rollback step 1 และ 2 อย่างไร?

```
   order-service DB       inventory-service DB      payment-service DB
   ┌──────────────┐       ┌──────────────────┐      ┌──────────────────┐
   │ BEGIN;        │       │ BEGIN;             │      │ BEGIN;             │
   │ INSERT order  │       │ UPDATE stock       │      │ INSERT payment     │
   │ COMMIT;  ✅   │       │ COMMIT;  ✅        │      │ (payment gateway   │
   │                │       │                    │      │  timeout!) ❌      │
   └──────────────┘       └──────────────────┘      └──────────────────┘
        │                        │                          │
        └──── ไม่มีกลไกใดผูกทั้ง 3 transaction เข้าด้วยกันโดยอัตโนมัติ ────┘
```

**ทำไมไม่ใช้ Two-Phase Commit (2PC)?** PostgreSQL รองรับ `PREPARE TRANSACTION` (2PC) ผ่าน `max_prepared_transactions` แต่ในทางปฏิบัติ 2PC ข้าม microservices **ไม่แนะนำ** เพราะ:

- ต้องการ **transaction coordinator** กลาง ที่กลายเป็น single point of failure และ bottleneck
- ทำให้ service **coupled กันแบบ synchronous** — ถ้า service ใดตอบช้าหรือดับ transaction ทั้งหมดจะค้าง (blocking) จนกว่า timeout
- ขัดกับหลักการ availability ของ microservices (CAP theorem: 2PC เอนเอียงไปทาง Consistency แลกกับ Availability อย่างรุนแรง)
- Payment gateway ภายนอก (เช่น Stripe, Omise) ไม่รองรับ 2PC protocol อยู่แล้ว

**ทางออกมาตรฐานของอุตสาหกรรม:** ใช้ **Saga Pattern** (Step 906-907) ซึ่งใช้ลำดับของ local transaction ที่แต่ละ service ทำ commit ของตัวเอง ร่วมกับ **compensating transaction** เพื่อ "ยกเลิก" ผลของ step ก่อนหน้าเมื่อเกิดความล้มเหลว

### 903.3 ปัญหาที่ 3: Data Duplication และ Eventual Consistency

เพื่อลดความจำเป็นในการเรียก API ข้าม service บ่อยๆ ทีมมักจะ denormalize ข้อมูลบางส่วน เช่น `order_items.product_name` ที่ copy มาจาก inventory-service ไว้ตอนสั่งซื้อ ซึ่งหมายความว่า:

- ข้อมูลชุดเดียวกันอยู่หลายที่ (duplication) — ต้องยอมรับว่าอาจไม่ sync กันทันที
- ถ้า inventory-service เปลี่ยนชื่อสินค้า ชื่อเก่าที่ denormalize ไว้ใน order-service จะไม่เปลี่ยนตาม (ซึ่งจริงๆ อาจเป็นพฤติกรรมที่ "ถูกต้อง" เพราะ order เป็น snapshot ของราคา/ชื่อ ณ เวลาสั่งซื้อ)
- ทีมต้องออกแบบระบบให้ "ทนต่อความไม่ sync ชั่วคราว" (tolerate temporary inconsistency) ได้

### 903.4 ปัญหาที่ 4: Debugging และ Observability ยากขึ้น

เมื่อ request หนึ่งวิ่งผ่านหลาย service หลายฐานข้อมูล การ debug "ทำไม order #1234 ค้างอยู่ที่สถานะ PENDING" ต้องตรวจสอบ log/state ใน 3-4 ฐานข้อมูลพร้อมกัน ต้องมี:

- **Correlation ID** ที่ติดไปกับทุก request/event เพื่อ trace ข้าม service
- **Distributed tracing** (เช่น OpenTelemetry) เพื่อดู timeline ของแต่ละ step
- **Saga state table** ที่บันทึกว่า saga แต่ละ instance อยู่ที่ step ไหน (จะกล่าวใน Step 907)

### 903.5 สรุปตารางเปรียบเทียบ

| ปัญหา | Monolith | Microservices | ทางออก |
|---|---|---|---|
| JOIN ข้าม domain | ทำได้ตรงๆ | ทำไม่ได้ | API Composition / CQRS read model |
| Transaction ข้าม domain | ACID เต็มรูปแบบ | ไม่มีในตัว | Saga Pattern |
| Consistency | Strong ทันที | Eventual | ออกแบบ UX ให้รองรับ |
| Debug/trace | อ่าน log เดียว | ต้อง trace ข้าม service | Correlation ID + distributed tracing |

---

## Step 904: Shared Database Anti-pattern

### 904.1 นิยามของ Anti-pattern

**Shared Database** คือสถานการณ์ที่หลาย microservice **เข้าถึงฐานข้อมูลเดียวกัน (หรือตารางเดียวกัน) โดยตรง** แม้จะถูก "แบ่ง service" ในระดับโค้ดแล้วก็ตาม เป็น anti-pattern ที่พบบ่อยที่สุดเมื่อทีมพยายาม migrate จาก monolith แบบเร่งรีบ

```
              ❌ Shared Database Anti-pattern
   ┌───────────────┐  ┌───────────────┐  ┌───────────────┐
   │ order-service │  │inventory-svc  │  │customer-svc   │
   └───────┬───────┘  └───────┬───────┘  └───────┬───────┘
           │                  │                  │
           └──────────────────┼──────────────────┘
                               ▼
                    ┌─────────────────────┐
                    │   shared_db          │
                    │  - orders             │
                    │  - products           │
                    │  - customers          │
                    │  (ทุก service query    │
                    │   ตารางเดียวกันหมด)   │
                    └─────────────────────┘
```

หลายทีมเริ่มต้นด้วยแนวคิดนี้เพราะ "ง่าย" — แยกโค้ดเป็น service ได้เร็ว ไม่ต้องคิดเรื่อง API Composition หรือ Saga ให้ปวดหัว แต่ในระยะยาวจะสร้างปัญหาที่ร้ายแรงกว่าเดิม

### 904.2 ทำไม Shared Database ถึงเป็นปัญหาในระยะยาว

**1. Schema เปลี่ยนแล้วพัง service อื่นโดยไม่รู้ตัว**

```sql
-- ทีม inventory-service ต้องการเปลี่ยนชื่อคอลัมน์เพื่อความชัดเจน
ALTER TABLE products RENAME COLUMN qty TO quantity_available;

-- ผลลัพธ์: order-service ที่ query products.qty ตรงๆ พังทันที
-- โดยที่ทีม inventory-service ไม่รู้ด้วยซ้ำว่ามีใครใช้คอลัมน์นี้บ้าง
```

เพราะไม่มี API contract เป็นตัวกลาง การเปลี่ยน schema กลายเป็น **breaking change ที่มองไม่เห็น** จนกว่า production จะพัง

**2. Deploy Coupling กลับมาอีกครั้ง**

แม้จะแยก service เป็นโค้ดคนละ repo คนละ deploy pipeline แล้ว แต่ถ้ายังใช้ DB เดียวกัน การ migration ของตารางหนึ่งอาจต้อง lock ตารางที่ service อื่นกำลังใช้งานอยู่ (เช่น `ALTER TABLE ... ADD COLUMN` แบบ `NOT NULL DEFAULT` ใน PostgreSQL เก่าที่ rewrite ทั้งตาราง) ทำให้ service อื่น downtime ไปด้วย — นี่คือข้อเสียของ monolith ที่ "ทำเหมือน microservices" แต่ยังไม่ได้ประโยชน์อะไรเลย

**3. Connection Pool และ Resource Contention**

ทุก service แย่งกันใช้ `max_connections` เดียวกัน, buffer cache เดียวกัน, WAL เดียวกัน ถ้า inventory-service มี query หนักๆ (batch job ตอนเที่ยงคืน) มันจะกระทบ latency ของ order-service ทันที แม้จะเป็นคนละ business domain

**4. ไม่มี Ownership ที่ชัดเจน**

เมื่อทุก service query ตารางเดียวกันได้ ไม่มีใครเป็น "เจ้าของ" schema อย่างแท้จริง การเพิ่ม constraint, index, trigger กลายเป็นการตัดสินใจที่ต้องประชุมข้ามทีมทุกครั้ง ทำให้ velocity ของทีมช้าลงเหมือนสมัย monolith

**5. Transaction Isolation ผิดที่ผิดทาง**

บางทีมพยายามแก้ปัญหา distributed transaction (Step 903) โดยใช้ shared database เพื่อ "เอา ACID กลับมา" — แต่นี่คือการแลก loose coupling (ที่เป็นเหตุผลหลักที่ทำ microservices) ทิ้งไปทั้งหมด เพื่อแก้ปัญหาที่ควรแก้ด้วย Saga Pattern แทน

### 904.3 กรณีศึกษา: postgres_fdw ก็เป็น Anti-pattern เช่นกันถ้าใช้ผิดทาง

บางทีมพยายามใช้ `postgres_fdw` (Foreign Data Wrapper) เพื่อให้ order-service "query" ตาราง customers ใน customer_db ได้โดยตรงผ่าน foreign table:

```sql
-- ⚠️ ตัวอย่างสิ่งที่ไม่ควรทำใน production microservices
CREATE EXTENSION IF NOT EXISTS postgres_fdw;

CREATE SERVER customer_db_server
    FOREIGN DATA WRAPPER postgres_fdw
    OPTIONS (host 'customer-db-host', dbname 'customer_db', port '5432');

CREATE FOREIGN TABLE customers_fdw (
    id BIGINT,
    full_name TEXT,
    email TEXT
) SERVER customer_db_server
  OPTIONS (schema_name 'public', table_name 'customers');

-- ตอนนี้ order-service เขียน JOIN ข้าม service ได้เหมือนเดิม
SELECT o.id, c.full_name
FROM orders o
JOIN customers_fdw c ON c.id = o.customer_id;
```

แม้จะทำได้ทางเทคนิค แต่นี่คือ **shared database anti-pattern ที่แต่งตัวใหม่** — สร้าง tight coupling ระดับ schema เหมือนเดิม (ถ้า customer_db เปลี่ยน schema, foreign table พังทันที) แถมยังเพิ่ม network dependency แบบ synchronous ที่ query planner มองไม่เห็น cost ที่แท้จริง (คาดเดา cardinality ยาก, join ข้าม network ช้า) `postgres_fdw` เหมาะกับงาน migration ชั่วคราวหรือ data warehousing มากกว่าการใช้เป็นสถาปัตยกรรมถาวรของ microservices

### 904.4 ทางออกที่ถูกต้อง

| ต้องการทำอะไร | อย่าทำ (anti-pattern) | ควรทำ |
|---|---|---|
| อ่านข้อมูลลูกค้าจาก order-service | Query `customer_db` ตรงๆ หรือใช้ FDW | เรียก customer-service API หรือเก็บ read model ผ่าน event |
| ต้องการ transaction ข้าม domain | ใช้ shared DB เพื่อ ACID | ใช้ Saga Pattern |
| ต้องการ report รวมข้อมูลหลาย domain | JOIN ข้าม shared DB | ใช้ data warehouse/analytics DB แยกต่างหาก ที่ sync ผ่าน CDC |
| ต้องการลด latency การเรียก API ซ้ำๆ | เข้าถึง DB อีก service ตรงๆ | Denormalize/cache ข้อมูลที่จำเป็นไว้ใน service ตัวเอง ผ่าน event subscription |

> **หลักการทอง:** ถ้าจะข้าม service boundary ให้ข้ามผ่าน **API หรือ event เท่านั้น** ไม่ว่าจะสะดวกแค่ไหนที่จะ query database ตรงๆ ก็ตาม วินัยข้อนี้คือสิ่งที่แยกระหว่าง "microservices ที่แท้จริง" กับ "distributed monolith"

---

## Step 905: API Composition Pattern

### 905.1 แนวคิด

**API Composition** คือการย้ายการ "รวมข้อมูล" (join) จากระดับ database ไปเป็นระดับ **application/API layer** แทน โดยมี "composer" (มักเป็น API Gateway, BFF — Backend for Frontend, หรือ service ที่ทำหน้าที่ query aggregator) เรียก API ของหลาย service พร้อมกัน แล้วนำผลลัพธ์มารวมกันในหน่วยความจำ

```
                         Client Request:
                "ขอดูรายละเอียด order #1234 พร้อมชื่อลูกค้าและสถานะการชำระเงิน"

                              ┌─────────────────┐
                              │   API Composer    │
                              │ (order-detail-api) │
                              └─────────┬─────────┘
                    ┌────────────────────┼────────────────────┐
                    ▼                    ▼                    ▼
          ┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐
          │  order-service    │ │ customer-service  │ │ payment-service   │
          │ GET /orders/1234  │ │ GET /customers/55  │ │ GET /payments?    │
          │                    │ │                    │ │   order_id=1234   │
          └──────────────────┘ └──────────────────┘ └──────────────────┘
                    │                    │                    │
                    └────────────────────┼────────────────────┘
                                          ▼
                              ┌─────────────────────┐
                              │  รวมผลลัพธ์เป็น JSON   │
                              │  แล้วส่งกลับ client    │
                              └─────────────────────┘
```

### 905.2 ตัวอย่าง Pseudocode ของ API Composer

```javascript
// order-detail-api (composer service) — Node.js pseudocode
async function getOrderDetail(orderId) {
  // เรียก order-service ก่อนเพื่อรู้ customer_id และ payment reference
  const order = await orderServiceClient.getOrder(orderId);

  // เรียก customer-service และ payment-service แบบขนานกัน (parallel)
  const [customer, payment] = await Promise.all([
    customerServiceClient.getCustomer(order.customerId),
    paymentServiceClient.getPaymentByOrderId(orderId),
  ]);

  // รวมผลลัพธ์ที่ระดับ application
  return {
    orderId: order.id,
    status: order.status,
    totalAmount: order.totalAmount,
    items: order.items,
    customer: {
      name: customer.fullName,
      email: customer.email,
    },
    payment: {
      status: payment ? payment.status : 'NOT_FOUND',
      paidAt: payment ? payment.createdAt : null,
    },
  };
}
```

### 905.3 ข้อดีและข้อเสียของ API Composition

**ข้อดี:**

- รักษา loose coupling ของ database ไว้ได้เต็มที่ — ไม่ต้อง query ข้าม service DB
- ง่ายต่อการทำความเข้าใจ (สำหรับ query ที่ไม่ซับซ้อนมาก)
- ไม่ต้องมี infrastructure เพิ่มเติม (ไม่ต้องมี message broker, CDC pipeline)

**ข้อเสีย:**

1. **N+1 query problem ในระดับ service**: ถ้าต้องดึงรายการ order 100 รายการ พร้อมชื่อลูกค้าแต่ละคน composer อาจต้องยิง API ไป customer-service 100 ครั้ง (ต้องแก้ด้วย batch endpoint เช่น `GET /customers?ids=1,2,3,...`)
2. **Latency สูงขึ้น**: การรวม 3 API call (แม้จะ parallel) ก็ยังช้ากว่า SQL JOIN เดียวในฐานข้อมูลเดียวกันมาก
3. **การ filter/sort ข้าม service ทำได้ยาก**: เช่น "หาออเดอร์ทั้งหมดของลูกค้าที่อยู่จังหวัดเชียงใหม่ที่มียอดรวมมากกว่า 1000 บาท" — ต้อง filter จังหวัดที่ customer-service ก่อน ได้ list of customer_id แล้วเอาไป filter order-service อีกที ซึ่งไม่มีประสิทธิภาพเท่า SQL query เดียว
4. **Partial failure handling**: ถ้า payment-service ล่มระหว่าง compose ต้องตัดสินใจว่าจะ fail ทั้งหมด หรือส่งข้อมูลบางส่วนกลับไปพร้อม fallback value

### 905.4 ตัวอย่าง SQL-level: การจำลอง Composition ด้วย Batch Query

เพื่อแก้ปัญหา N+1 เวลาต้อง compose ข้อมูลจำนวนมาก แต่ละ service ควรมี endpoint ที่รับ **หลาย ID พร้อมกัน** และใช้ `= ANY(...)` แทนการ loop query ทีละตัว:

```sql
-- ภายใน customer-service เมื่อรับ request GET /customers?ids=1,2,3,4,5
SELECT id, full_name, email
FROM customers
WHERE id = ANY($1::bigint[]);   -- $1 = ARRAY[1,2,3,4,5]
```

```javascript
// composer เรียก batch endpoint แทนการ loop
const orders = await orderServiceClient.listRecentOrders();
const customerIds = [...new Set(orders.map(o => o.customerId))];
const customers = await customerServiceClient.getCustomersByIds(customerIds); // 1 call เดียว
const customerMap = new Map(customers.map(c => [c.id, c]));

const result = orders.map(o => ({
  ...o,
  customerName: customerMap.get(o.customerId)?.fullName ?? 'Unknown',
}));
```

### 905.5 เมื่อไหร่ควรใช้ API Composition เทียบกับ CQRS Read Model

| สถานการณ์ | แนะนำ |
|---|---|
| Query แบบ ad-hoc ไม่บ่อย, latency ไม่ critical | API Composition |
| ต้องการ real-time consistency สูงระหว่าง 2 service | API Composition (synchronous) |
| ต้องการ query ที่ซับซ้อน filter/sort/aggregate ข้าม domain บ่อยๆ | สร้าง read-optimized view ผ่าน CQRS + event replication (จะกล่าวลึกใน Part 092) |
| ต้องการ full-text search ข้ามหลาย domain | สร้าง search index (เช่น Elasticsearch) ที่ sync ผ่าน event/CDC |

API Composition เหมาะกับกรณีที่ไม่ซับซ้อนมาก ส่วนกรณีที่ต้อง query ซับซ้อนบ่อยๆ ควรพิจารณา **CQRS (Command Query Responsibility Segregation)** ซึ่งจะกล่าวถึงอย่างละเอียดใน Part 092 — บทนี้จะโฟกัสที่การส่งข้อมูลผ่าน event (Outbox + CDC) ซึ่งเป็นรากฐานของ CQRS read model เช่นกัน

---

## Step 906: Saga Pattern

### 906.1 ปัญหาที่ Saga แก้

จาก Step 903 เราเห็นแล้วว่าไม่มี distributed transaction แบบง่ายๆ ข้าม microservices **Saga Pattern** คือคำตอบมาตรฐานของอุตสาหกรรมสำหรับปัญหานี้

**นิยาม:** Saga คือลำดับของ **local transaction** ที่แต่ละ service ทำการ commit ของตัวเองตามลำดับ โดยแต่ละ step จะ trigger step ถัดไป หากมี step ใดล้มเหลว ระบบจะรัน **compensating transaction** ย้อนกลับ (จาก step ที่ล้มเหลวไล่ย้อนไปจนถึง step แรก) เพื่อ "ยกเลิก" ผลของ step ที่ทำสำเร็จไปแล้ว

```
   Happy path (ทุก step สำเร็จ):

   [T1: Create Order] ──▶ [T2: Reserve Stock] ──▶ [T3: Charge Payment] ──▶ [T4: Confirm Order]
        ✅                      ✅                      ✅                      ✅


   Failure path (step T3 ล้มเหลว ต้อง compensate ย้อนกลับ):

   [T1: Create Order] ──▶ [T2: Reserve Stock] ──▶ [T3: Charge Payment]
        ✅                      ✅                      ❌ (บัตรถูกปฏิเสธ)
        │                      │
        │                      ▼
        │            [C2: Release Stock]  (compensating transaction)
        │                      ✅
        ▼
   [C1: Cancel Order]  (compensating transaction)
        ✅
```

### 906.2 คุณสมบัติสำคัญของ Saga

1. **แต่ละ local transaction เป็น ACID ภายใน service ของตัวเอง** — ไม่มี distributed lock ข้าม service
2. **Compensating transaction ต้องเป็น semantic rollback ไม่ใช่ database rollback** — เพราะ transaction ก่อนหน้า commit ไปแล้วจริงๆ (ข้อมูลถูกเขียนลง disk แล้ว) เราจึงต้องสร้าง transaction ใหม่ที่ "ทำผลตรงข้าม" เช่น ถ้า T2 คือ `reserve stock` ตัว compensating C2 คือ `release stock` (ไม่ใช่ physical rollback)
3. **Saga ให้ eventual consistency ไม่ใช่ strong consistency** — ระหว่างที่ saga ยังไม่จบ ระบบอาจอยู่ในสถานะกลางๆ ที่มองเห็นได้จาก service อื่น (เช่น order สถานะ PENDING ค้างอยู่ 2 วินาที)
4. **Compensating transaction ต้องเป็น idempotent** — เพราะอาจถูกเรียกซ้ำเนื่องจาก retry (network failure, message redelivery)

### 906.3 หลักการออกแบบ Compensating Transaction

| Forward transaction | Compensating transaction | หมายเหตุ |
|---|---|---|
| Reserve stock (ลด `quantity_available`, เพิ่ม `quantity_reserved`) | Release stock (คืนค่ากลับ) | ต้อง idempotent — release ซ้ำต้องไม่ทำให้ stock เกินจริง |
| Charge payment | Refund payment | บาง payment gateway refund ไม่ได้ทันที ต้องมี state "REFUND_PENDING" |
| Create order (status=PENDING) | Cancel order (status=CANCELLED) | ไม่ใช่การลบ record — เก็บ audit trail ไว้เสมอ |
| Send confirmation email | (ไม่มี compensating — เป็น action ที่ "unlockable") | บาง action ทำ compensate ไม่ได้จริง ต้องออกแบบลำดับให้ action แบบนี้อยู่ท้ายสุดของ saga เสมอ |

> **ข้อควรระวังสำคัญ:** ไม่ใช่ทุก action จะ compensate ได้ (เช่น "ส่งอีเมลยืนยันไปแล้ว" ย้อนกลับไม่ได้จริง) ดังนั้นหลักการออกแบบ saga ที่ดีคือ **จัดลำดับให้ action ที่ compensate ไม่ได้ (non-compensatable) อยู่ท้ายสุดเสมอ** หลังจาก step ที่มีความเสี่ยงล้มเหลวสูงผ่านไปหมดแล้ว**

### 906.4 Semantic Lock และปัญหา Isolation ของ Saga

เนื่องจาก Saga ไม่มี distributed lock ข้าม service ระหว่างที่ saga กำลังทำงาน อาจมี process อื่นมาเห็นข้อมูล "กลางๆ" ได้ (เช่น stock ถูก reserve ไปแล้วแต่ order ยังไม่ confirm) วิธีแก้ปัญหานี้ที่นิยมใช้ ได้แก่:

- **Semantic lock**: ใช้ field สถานะ (เช่น `status = 'RESERVED'` แทน `'AVAILABLE'`) เพื่อบอกว่า record นี้ "ถูกจองไว้ชั่วคราว" — request อื่นที่มาถึงระหว่างนี้จะเห็นว่าไม่ available และต้องรอหรือ reject
- **Commutative update**: ออกแบบ operation ให้ "สลับลำดับได้" เช่น การบวก/ลบยอด แทนการ set ค่าตรงๆ เพื่อลดโอกาสเกิด lost update
- **Timeout + expiry**: reservation ที่ค้างนานเกินไป (เช่น payment ไม่ตอบกลับภายใน 5 นาที) ต้องมี background job มา expire และ release คืนอัตโนมัติ

```sql
-- ตัวอย่างการ reserve stock ด้วย semantic lock ผ่านสถานะ
-- ภายใน inventory-service เมื่อรับ event "OrderCreated"

BEGIN;

UPDATE stock_levels
SET quantity_available = quantity_available - $2,
    quantity_reserved   = quantity_reserved + $2,
    updated_at = now()
WHERE product_id = $1
  AND quantity_available >= $2   -- ป้องกัน overselling
RETURNING product_id;

-- ถ้า UPDATE ไม่มี row ที่ตรงเงื่อนไข (stock ไม่พอ) แปลว่า reserve ล้มเหลว
-- ต้องส่ง event "StockReservationFailed" กลับไปให้ saga orchestrator

COMMIT;
```

```sql
-- Compensating transaction: release stock ที่เคย reserve ไว้ (ต้อง idempotent)
BEGIN;

UPDATE stock_levels
SET quantity_available = quantity_available + $2,
    quantity_reserved   = quantity_reserved - $2,
    updated_at = now()
WHERE product_id = $1
  AND quantity_reserved >= $2;   -- ป้องกันการ release เกินกว่าที่เคย reserve จริง

COMMIT;
```

การใช้ `WHERE quantity_available >= $2` และ `WHERE quantity_reserved >= $2` เป็นเทคนิคสำคัญ: มันทำให้ operation เป็น **atomic conditional update** ในตัวเอง (ไม่ต้อง `SELECT ... FOR UPDATE` แยกต่างหาก) และช่วยให้ retry ซ้ำ (idempotent) ไม่ทำให้ตัวเลขผิดเพี้ยนหากมีการเรียกซ้ำโดยไม่ตั้งใจ (แม้จะยังไม่ perfect idempotency 100% — จะกล่าวเพิ่มเติมเรื่อง idempotency key ใน Step 908)

---

## Step 907: Saga แบบ Choreography เทียบกับ Orchestration

Saga มีสอง implementation style หลักคือ **Choreography** (แบบกระจาย ไม่มีศูนย์กลาง) และ **Orchestration** (แบบมีศูนย์กลางคอยสั่งการ) เราจะใช้ตัวอย่างเดียวกันตลอด: **Order Saga** — `reserve inventory → charge payment → confirm order`

### 907.1 Choreography-based Saga

ใน choreography แต่ละ service **subscribe event ของกันและกัน** และตัดสินใจทำ action ต่อไปด้วยตัวเอง ไม่มี "ผู้ควบคุมกลาง"

```
   order-service          inventory-service         payment-service
        │                        │                         │
        │  1. OrderCreated       │                         │
        │───────────event───────▶│                         │
        │                        │                         │
        │                        │ 2. StockReserved        │
        │◀────event──────────────│─────────event──────────▶│
        │  (order-service        │                         │
        │   ฟัง event นี้ด้วย     │                         │
        │   เพื่ออัปเดตสถานะ)    │                         │
        │                        │                         │
        │                        │                         │ 3. PaymentCharged
        │◀───────────────────event──────────────────────────│
        │  (order-service ฟัง event นี้                     │
        │   แล้วอัปเดต order.status = CONFIRMED)            │
        ▼                        ▼                         ▼
```

**รายละเอียด flow:**

1. `order-service` สร้าง order (status=`PENDING`) แล้ว publish event `OrderCreated`
2. `inventory-service` subscribe `OrderCreated` → พยายาม reserve stock
   - สำเร็จ → publish `StockReserved`
   - ล้มเหลว → publish `StockReservationFailed`
3. `payment-service` subscribe `StockReserved` → พยายาม charge เงิน
   - สำเร็จ → publish `PaymentCharged`
   - ล้มเหลว → publish `PaymentFailed`
4. `order-service` subscribe `PaymentCharged` → อัปเดต order.status = `CONFIRMED`
5. หากมี event `StockReservationFailed` หรือ `PaymentFailed` เกิดขึ้นระหว่างทาง `order-service` จะ subscribe event เหล่านั้นเพื่อ cancel order, และ `inventory-service` จะ subscribe `PaymentFailed` เพื่อ release stock กลับคืน (compensating)

```
   Failure case: PaymentFailed

   payment-service publish "PaymentFailed"
            │
            ├──▶ order-service:      subscribe → UPDATE orders SET status='CANCELLED'
            │
            └──▶ inventory-service:  subscribe → release stock (compensating)
```

**ข้อดีของ Choreography:**
- ไม่มี single point of failure ที่ระดับ coordinator
- แต่ละ service loosely coupled กันมาก — เพิ่ม service ใหม่เข้า flow ได้โดยแค่ subscribe event ที่มีอยู่แล้ว ไม่ต้องแก้โค้ด orchestrator
- เหมาะกับ flow ที่ไม่ซับซ้อนมาก จำนวน step น้อย

**ข้อเสียของ Choreography:**
- เมื่อจำนวน step เพิ่มขึ้น การเข้าใจ "ภาพรวมของ flow ทั้งหมด" ยากขึ้นมาก เพราะ logic กระจายอยู่ในหลาย service (ไม่มีที่เดียวที่อ่านแล้วเห็น flow ทั้งหมด)
- Debug ยาก — ต้องไล่ event log ข้ามหลาย service เพื่อ reconstruct ว่าเกิดอะไรขึ้น
- เสี่ยงเกิด **circular dependency ของ event** ถ้าออกแบบไม่ดี (service A ฟัง event ของ B และ B ก็ฟัง event ของ A กลับ)
- การเพิ่ม stepใหม่ตรงกลาง flow (เช่น เพิ่ม fraud-check-service ก่อน charge payment) อาจต้องแก้หลาย service พร้อมกัน

### 907.2 Orchestration-based Saga

ใน orchestration มี **saga orchestrator** (เป็น component หรือ service กลาง) ที่รู้ flow ทั้งหมด และเป็นผู้สั่งการแต่ละ service ตามลำดับอย่างชัดเจน (command-based แทน event-based)

```
                        ┌─────────────────────────┐
                        │   Order Saga Orchestrator │
                        │   (มี saga_state table)    │
                        └─────────────┬─────────────┘
              1. CreateOrder          │          2. ReserveStock
        ┌──────────────────────────────┼──────────────────────────────┐
        ▼                              │                              ▼
┌───────────────┐                     │                     ┌───────────────────┐
│ order-service  │                     │                     │ inventory-service  │
└───────────────┘                     │                     └───────────────────┘
        ▲                              │                              │
        │        4. ConfirmOrder       │        3. ChargePayment      │
        │      ┌────────────────────────┘        │                    │
        │      │                                  ▼                    │
        │      │                     ┌───────────────────┐            │
        │      │                     │  payment-service    │            │
        │      │                     └───────────────────┘            │
        └──────┴──────────────────────────────────────────────────────┘
                        orchestrator รอ response ของแต่ละ step
                        แล้วตัดสินใจว่าจะไป step ถัดไป หรือ compensate
```

**รายละเอียด flow:**

1. Orchestrator สั่ง `CreateOrder` ไปที่ order-service → รอผลลัพธ์
2. ถ้าสำเร็จ สั่ง `ReserveStock` ไปที่ inventory-service → รอผลลัพธ์
3. ถ้าสำเร็จ สั่ง `ChargePayment` ไปที่ payment-service → รอผลลัพธ์
4. ถ้าสำเร็จ สั่ง `ConfirmOrder` ไปที่ order-service → จบ saga (สถานะ `COMPLETED`)
5. ถ้า step ใด step หนึ่งล้มเหลว orchestrator จะสั่ง compensating command ย้อนกลับตามลำดับ (reverse order)

### 907.3 ตัวอย่าง Pseudocode ของ Saga Orchestrator

```python
# order_saga_orchestrator.py — pseudocode

class OrderSagaOrchestrator:

    STEPS = [
        ("CreateOrder",     "order-service",     "CancelOrder"),
        ("ReserveStock",    "inventory-service", "ReleaseStock"),
        ("ChargePayment",   "payment-service",   "RefundPayment"),
        ("ConfirmOrder",    "order-service",      None),  # step สุดท้ายไม่มี compensating
    ]

    def run(self, saga_id, order_request):
        # บันทึกสถานะเริ่มต้นของ saga ลงตาราง saga_state (ดู 907.4)
        self.save_saga_state(saga_id, step_index=0, status="STARTED")

        completed_steps = []

        for index, (command, service, compensating_command) in enumerate(self.STEPS):
            try:
                result = self.send_command(service, command, order_request, saga_id)

                if not result.success:
                    raise SagaStepFailed(f"{command} failed: {result.error}")

                completed_steps.append((service, compensating_command))
                self.save_saga_state(saga_id, step_index=index + 1, status="IN_PROGRESS")

            except SagaStepFailed as e:
                self.log_failure(saga_id, command, e)
                self.compensate(saga_id, completed_steps)
                self.save_saga_state(saga_id, step_index=index, status="COMPENSATED")
                return SagaResult(success=False, reason=str(e))

        self.save_saga_state(saga_id, step_index=len(self.STEPS), status="COMPLETED")
        return SagaResult(success=True)

    def compensate(self, saga_id, completed_steps):
        # ย้อนกลับตามลำดับย้อนกลับ (reverse order) — สำคัญมาก!
        for service, compensating_command in reversed(completed_steps):
            if compensating_command is None:
                continue
            self.send_command(service, compensating_command, saga_id=saga_id)
```

### 907.4 Saga State Table — บันทึกสถานะของ Orchestrator ใน PostgreSQL

Orchestrator ต้อง **persist สถานะของแต่ละ saga instance** ลงฐานข้อมูล เพื่อให้สามารถ recover ได้เมื่อ orchestrator ล่มกลางทาง (crash recovery) — นี่คือฐานข้อมูลของ orchestrator เอง (อาจเป็น service แยกชื่อ `saga-orchestrator-service` พร้อม `saga_db` ของตัวเอง)

```sql
-- ========== saga_db (saga-orchestrator-service) ==========

CREATE TABLE order_sagas (
    saga_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id        BIGINT,
    current_step    TEXT NOT NULL,           -- เช่น 'RESERVE_STOCK', 'CHARGE_PAYMENT'
    status          TEXT NOT NULL DEFAULT 'STARTED'
                        CHECK (status IN ('STARTED','IN_PROGRESS','COMPLETED',
                                          'COMPENSATING','COMPENSATED','FAILED')),
    payload         JSONB NOT NULL,          -- ข้อมูลตั้งต้นของ saga (order request)
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE order_saga_steps (
    id              BIGSERIAL PRIMARY KEY,
    saga_id         UUID NOT NULL REFERENCES order_sagas(saga_id),
    step_name       TEXT NOT NULL,           -- 'CreateOrder','ReserveStock', ...
    status          TEXT NOT NULL
                        CHECK (status IN ('PENDING','SUCCESS','FAILED','COMPENSATED')),
    request_payload  JSONB,
    response_payload JSONB,
    started_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    finished_at     TIMESTAMPTZ
);

-- Index สำหรับ query saga ที่ค้างนาน (stuck saga) เพื่อ alert/retry
CREATE INDEX idx_order_sagas_in_progress
    ON order_sagas (updated_at)
    WHERE status IN ('STARTED', 'IN_PROGRESS', 'COMPENSATING');
```

```sql
-- ตัวอย่าง query หา saga ที่ค้างนานเกิน 5 นาที (อาจเป็นสัญญาณของ service ที่ไม่ตอบสนอง)
SELECT saga_id, current_step, status, updated_at
FROM order_sagas
WHERE status IN ('STARTED', 'IN_PROGRESS', 'COMPENSATING')
  AND updated_at < now() - interval '5 minutes'
ORDER BY updated_at;
```

Partial index ข้างต้น (`WHERE status IN (...)`) ช่วยให้ query ตรวจจับ stuck saga ทำงานเร็วมาก เพราะสแกนเฉพาะ saga ที่ยัง active อยู่ ไม่ต้องสแกน saga ที่ `COMPLETED` ไปแล้วนับล้านแถว — นี่คือการนำความรู้เรื่อง partial index จาก Part ก่อนๆ มาประยุกต์ใช้ในบริบท microservices

### 907.5 เปรียบเทียบ Choreography vs Orchestration

| ประเด็น | Choreography | Orchestration |
|---|---|---|
| Central control | ไม่มี | มี (saga orchestrator) |
| Coupling | หลวมมาก (แต่ละ service รู้แค่ event ที่ subscribe) | มี coupling กับ orchestrator (แต่ service ไม่ต้องรู้จักกัน) |
| ความซับซ้อนของ flow ที่รองรับได้ดี | Flow สั้น ไม่กี่ step | Flow ยาว ซับซ้อน หลาย step หลาย branch |
| Debuggability | ยาก (ต้องไล่ event ข้ามหลาย service) | ง่ายกว่า (ดูที่ saga_state table ที่เดียว) |
| Single point of failure | ไม่มี | orchestrator ต้องออกแบบให้ HA (high availability) |
| เพิ่ม step ใหม่ | ต้องแก้หลาย service ที่เกี่ยวข้อง | แก้ที่ orchestrator ที่เดียว |
| เหมาะกับทีม | ทีมเล็ก, flow เรียบง่าย | ทีมใหญ่, flow ซับซ้อน, ต้องการ visibility |

**คำแนะนำในทางปฏิบัติ:** สำหรับ flow ที่มีมากกว่า 3-4 step ขึ้นไป หรือ flow ที่มี business logic ซับซ้อน (branching, retry policy ต่างกันในแต่ละ step) **Orchestration มักจะดูแลรักษาง่ายกว่าในระยะยาว** แม้จะมี component เพิ่มขึ้นมา (orchestrator) เพราะรวม business logic ของ flow ไว้ที่เดียว ทำให้เห็นภาพรวมและแก้ไขได้ง่าย ส่วน Choreography เหมาะกับ flow ที่สั้น ตรงไปตรงมา และต้องการความเป็นอิสระของ service สูงสุด

---

## Step 908: Outbox Pattern

### 908.1 ปัญหาที่ Outbox แก้: Dual Write Problem

เมื่อ service หนึ่งต้อง (1) เขียนข้อมูลลงฐานข้อมูลของตัวเอง **และ** (2) publish event ไปยัง message broker (เช่น Kafka, RabbitMQ) เพื่อแจ้ง service อื่น — นี่คือ **dual write problem**: สอง operation นี้เขียนไปยังระบบสองระบบที่แยกกัน (database และ message broker) ซึ่งไม่มี transaction ร่วมกัน

```
   ❌ ปัญหา Dual Write

   BEGIN;
   INSERT INTO orders (...) VALUES (...);
   COMMIT;                                    ← เขียนสำเร็จลง DB

   kafkaProducer.send("OrderCreated", {...}); ← ถ้า service crash ตรงนี้พอดี
                                                  (หลัง commit DB แต่ก่อนส่ง event สำเร็จ)
                                                  event นี้จะหายไปตลอดกาล!
                                                  inventory-service จะไม่มีวันรู้ว่ามี order ใหม่
```

หรือในทางกลับกัน ถ้า publish event ก่อนแล้ว DB commit ล้มเหลว จะกลายเป็น event ที่ "โกหก" — บอกว่ามี order ใหม่ทั้งที่จริงไม่มี ทั้งสองกรณีทำให้ระบบ **inconsistent อย่างถาวร**

### 908.2 หลักการของ Outbox Pattern

**Outbox Pattern** แก้ปัญหานี้โดยการ **เขียน event ลงตาราง `outbox_events` ในฐานข้อมูลเดียวกัน ในทรานแซคชันเดียวกันกับข้อมูลจริง** — เพราะทั้งสองอยู่ใน PostgreSQL instance เดียวกัน จึงได้ atomicity ของ ACID transaction ตามปกติ (ไม่มี dual write อีกต่อไป เพราะเป็น "single write" ไปยัง database เดียว) จากนั้นมี process แยกต่างหาก (message relay หรือ CDC) คอยอ่านตาราง outbox แล้วส่งไปยัง message broker จริงในภายหลัง

```
   ✅ Outbox Pattern

   BEGIN;
   INSERT INTO orders (...) VALUES (...);
   INSERT INTO outbox_events (...) VALUES (...);   ← event ถูกเขียนใน transaction เดียวกัน
   COMMIT;                                          ← atomic: ทั้งคู่สำเร็จ หรือทั้งคู่ไม่สำเร็จ

                            │
                            ▼
              (async) Message Relay / CDC process
                     อ่าน outbox_events แล้วส่งไป Kafka/RabbitMQ
                            │
                            ▼
                     inventory-service subscribe ได้แน่นอน
                     (at-least-once delivery guarantee)
```

### 908.3 การออกแบบตาราง Outbox

```sql
-- ========== order_db (order-service) ==========

CREATE TABLE outbox_events (
    id              BIGSERIAL PRIMARY KEY,
    aggregate_type  TEXT NOT NULL,        -- เช่น 'Order'
    aggregate_id    TEXT NOT NULL,        -- เช่น order.id (เก็บเป็น text เพื่อความยืดหยุ่น)
    event_type      TEXT NOT NULL,        -- เช่น 'OrderCreated', 'OrderCancelled'
    payload         JSONB NOT NULL,       -- ข้อมูล event ทั้งหมดที่ consumer ต้องใช้
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    processed_at    TIMESTAMPTZ,          -- NULL = ยังไม่ถูกส่งออกไป
    -- idempotency key ป้องกัน consumer ประมวลผลซ้ำซ้อน (ดู 908.6)
    event_id        UUID NOT NULL DEFAULT gen_random_uuid()
);

-- Index สำคัญที่สุด: หา event ที่ยังไม่ได้ประมวลผล เรียงตามลำดับเวลา
CREATE INDEX idx_outbox_unprocessed
    ON outbox_events (created_at)
    WHERE processed_at IS NULL;
```

### 908.4 การเขียน Event ลง Outbox ในทรานแซคชันเดียวกัน (Application-level)

วิธีที่ตรงไปตรงมาที่สุดคือให้ application code เขียนทั้งสองตารางในทรานแซคชันเดียว:

```sql
-- Application (order-service) รันคำสั่งนี้ทั้งหมดใน transaction เดียว
-- เมื่อลูกค้ากด "checkout"

BEGIN;

INSERT INTO orders (id, customer_id, status, total_amount)
VALUES (1234, 1001, 'PENDING', 1500.00);

INSERT INTO order_items (order_id, product_id, product_name, unit_price, quantity)
VALUES
    (1234, 55, 'เสื้อยืดสีขาว Size L', 350.00, 2),
    (1234, 78, 'กางเกงยีนส์', 800.00, 1);

-- เขียน event ลง outbox ใน transaction เดียวกัน — นี่คือหัวใจของ pattern นี้
INSERT INTO outbox_events (aggregate_type, aggregate_id, event_type, payload)
VALUES (
    'Order',
    '1234',
    'OrderCreated',
    jsonb_build_object(
        'orderId', 1234,
        'customerId', 1001,
        'totalAmount', 1500.00,
        'items', jsonb_build_array(
            jsonb_build_object('productId', 55, 'quantity', 2),
            jsonb_build_object('productId', 78, 'quantity', 1)
        ),
        'occurredAt', now()
    )
);

COMMIT;
-- ทั้งหมดนี้ atomic: ถ้า crash ก่อน COMMIT ไม่มีอะไรถูกเขียนเลย
-- ถ้า COMMIT สำเร็จ ทั้ง order และ event ถูกบันทึกแน่นอนทั้งคู่
```

### 908.5 Trigger-based Insertion — ให้ PostgreSQL Trigger เขียน Outbox ให้อัตโนมัติ

อีกแนวทางหนึ่งคือใช้ **trigger** เพื่อสร้าง outbox event โดยอัตโนมัติทุกครั้งที่ตาราง `orders` เปลี่ยนแปลง แทนที่จะพึ่ง application code ต้องจำ insert เอง (ลดความเสี่ยงที่ developer จะลืม insert outbox event ในบาง code path)

```sql
-- Trigger function: สร้าง outbox event อัตโนมัติเมื่อ orders ถูก INSERT/UPDATE
CREATE OR REPLACE FUNCTION fn_orders_to_outbox()
RETURNS TRIGGER AS $$
DECLARE
    v_event_type TEXT;
    v_payload    JSONB;
BEGIN
    IF TG_OP = 'INSERT' THEN
        v_event_type := 'OrderCreated';
        v_payload := to_jsonb(NEW);
    ELSIF TG_OP = 'UPDATE' AND OLD.status IS DISTINCT FROM NEW.status THEN
        v_event_type := 'OrderStatusChanged';
        v_payload := jsonb_build_object(
            'orderId', NEW.id,
            'oldStatus', OLD.status,
            'newStatus', NEW.status
        );
    ELSE
        -- UPDATE ที่ไม่เปลี่ยน status ไม่ต้องสร้าง event (ปรับตาม business rule ได้)
        RETURN NEW;
    END IF;

    INSERT INTO outbox_events (aggregate_type, aggregate_id, event_type, payload)
    VALUES ('Order', NEW.id::text, v_event_type, v_payload);

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_orders_outbox
    AFTER INSERT OR UPDATE ON orders
    FOR EACH ROW
    EXECUTE FUNCTION fn_orders_to_outbox();
```

**เปรียบเทียบสองแนวทาง:**

| แนวทาง | ข้อดี | ข้อเสีย |
|---|---|---|
| **Application-level insert** | ควบคุม payload ของ event ได้เต็มที่ (เลือกได้ว่าจะใส่ field อะไรบ้าง, รวมข้อมูลจากหลายตาราง) | Developer ต้องจำ insert ทุก code path ที่เกี่ยวข้อง เสี่ยงลืม |
| **Trigger-based** | รับประกันว่า event ถูกสร้างเสมอไม่ว่าจะแก้ข้อมูลผ่านทางไหน (แม้แต่ manual SQL หรือ batch job) | Payload ถูกจำกัดด้วยข้อมูลในตารางนั้น (`to_jsonb(NEW)`) การปรับแต่ง logic ซับซ้อนทำได้ยากกว่า |

ในทางปฏิบัติ ทีมส่วนใหญ่เลือกใช้ **application-level insert** เพราะควบคุม payload structure (event schema) ได้ดีกว่า และ payload ของ domain event มักจะซับซ้อนกว่าแค่ "แปลงแถวเป็น JSON" ตรงๆ — trigger-based เหมาะกับกรณีที่ต้องการ safety net เพิ่มเติม หรือใช้กับตารางที่มีการแก้ไขผ่านหลายช่องทาง

### 908.6 Idempotency: การรับมือกับ Event ที่ถูกส่งซ้ำ

Outbox pattern รับประกัน **at-least-once delivery** (event จะถูกส่งอย่างน้อยหนึ่งครั้งแน่นอน) แต่ไม่รับประกัน **exactly-once** — เป็นไปได้ที่ event เดียวกันจะถูกส่งซ้ำ (เช่น relay process ส่งสำเร็จแต่ crash ก่อนจะ mark `processed_at` ทำให้ retry ส่งซ้ำ) ดังนั้น **consumer (inventory-service) ต้องออกแบบให้ idempotent**:

```sql
-- ========== inventory_db (inventory-service) ==========
-- ตารางเก็บ event_id ที่เคยประมวลผลแล้ว เพื่อป้องกันการประมวลผลซ้ำ

CREATE TABLE processed_events (
    event_id        UUID PRIMARY KEY,
    processed_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

```sql
-- เมื่อ inventory-service รับ event "OrderCreated" (event_id = $1)
BEGIN;

-- ลองบันทึก event_id นี้ก่อน ถ้ามีอยู่แล้ว (ON CONFLICT DO NOTHING ไม่ insert)
-- แล้วตรวจว่า insert สำเร็จจริงหรือไม่ ถ้าไม่สำเร็จ = event นี้เคยประมวลผลแล้ว ให้ skip
INSERT INTO processed_events (event_id) VALUES ($1)
ON CONFLICT (event_id) DO NOTHING
RETURNING event_id;

-- ถ้า RETURNING ไม่มีแถว (เพราะ conflict) แปลว่าเคยประมวลผล event นี้แล้ว
-- application code จะ ROLLBACK และไม่ทำ business logic ซ้ำ

-- ถ้าสำเร็จ (มีแถว returning) จึงทำ business logic จริง
UPDATE stock_levels
SET quantity_available = quantity_available - 2,
    quantity_reserved   = quantity_reserved + 2
WHERE product_id = 55;

COMMIT;
```

รูปแบบนี้เรียกว่า **idempotency key pattern** — การรวม "การบันทึกว่าเคยประมวลผลแล้ว" กับ "การทำ business logic" ไว้ใน transaction เดียวกัน ทำให้ทั้งคู่ atomic ด้วยกัน ป้องกัน race condition ที่ process ล่มระหว่างสอง step

### 908.7 Message Relay Process — การส่ง Event จาก Outbox ไปยัง Message Broker

Process ที่อ่านตาราง `outbox_events` แล้วส่งไปยัง message broker จริง มี 2 แนวทางหลัก:

**แนวทาง A: Polling Publisher** — process แยกต่างหาก poll ตาราง outbox เป็นระยะ

```sql
-- Polling query: ดึง event ที่ยังไม่ประมวลผล เรียงตามเวลา ล็อคแถวกันชนกันถ้ามีหลาย worker
SELECT id, aggregate_type, aggregate_id, event_type, payload, event_id
FROM outbox_events
WHERE processed_at IS NULL
ORDER BY created_at
LIMIT 100
FOR UPDATE SKIP LOCKED;   -- ป้องกัน worker หลายตัวหยิบ event เดียวกันซ้ำ
```

```python
# message_relay_worker.py — pseudocode
def poll_and_publish():
    while True:
        with db.transaction():
            rows = db.execute("""
                SELECT id, event_type, payload, event_id
                FROM outbox_events
                WHERE processed_at IS NULL
                ORDER BY created_at
                LIMIT 100
                FOR UPDATE SKIP LOCKED
            """)

            for row in rows:
                kafka_producer.send(
                    topic="order-events",
                    key=row.event_id,
                    value=row.payload,
                )
                db.execute(
                    "UPDATE outbox_events SET processed_at = now() WHERE id = %s",
                    [row.id],
                )
            # commit transaction: mark processed เกิดพร้อมกับการส่งสำเร็จ
        sleep(0.5)  # poll ทุกครึ่งวินาที
```

ข้อดี: implement ง่าย ไม่ต้องพึ่ง infrastructure พิเศษ
ข้อเสีย: มี latency (polling interval), เพิ่ม load บนฐานข้อมูลจาก polling ที่ถี่, ต้องจัดการ `FOR UPDATE SKIP LOCKED` ให้ดีถ้ามีหลาย worker

**แนวทาง B: Change Data Capture (CDC)** — ใช้ PostgreSQL logical replication อ่านการเปลี่ยนแปลงจาก WAL โดยตรง ไม่ต้อง polling เลย ซึ่งจะกล่าวถึงอย่างละเอียดใน Step 909

### 908.8 การทำความสะอาด Outbox Table

Outbox table ควรมี retention policy เพื่อไม่ให้โตไม่จำกัด — event ที่ `processed_at IS NOT NULL` และเก่ากว่าระยะเวลาหนึ่ง (เช่น 7 วัน สำหรับ audit purpose) ควรถูกลบด้วย batch job:

```sql
-- Cleanup job รันเป็น cron/scheduled job แยกต่างหาก
DELETE FROM outbox_events
WHERE processed_at IS NOT NULL
  AND processed_at < now() - interval '7 days';
```

ควรรันคำสั่งนี้เป็น batch เล็กๆ (เช่น `LIMIT 1000` วนหลายรอบ) แทนการลบทีเดียวจำนวนมาก เพื่อลด lock contention และ WAL bloat ในช่วงเวลาสั้นๆ — เทคนิคนี้เหมือนกับที่กล่าวถึงใน Part ที่ว่าด้วยการทำ batch delete ขนาดใหญ่

---

## Step 909: Change Data Capture (CDC)

### 909.1 ทำไมต้องมี CDC เมื่อมี Outbox Pattern แล้ว

Polling Publisher (Step 908.7 แนวทาง A) ใช้งานได้จริง แต่มีข้อจำกัดคือ:

- เพิ่ม **load บนฐานข้อมูล** จากการ poll ถี่ๆ (แม้จะ index ดีแล้วก็ตาม)
- มี **latency** เท่ากับ polling interval (ถ้า poll ทุก 500ms ก็มี delay เฉลี่ยนั้น)
- Scale ยากขึ้นเมื่อ throughput สูงมาก (หลายพัน event/วินาที)

**Change Data Capture (CDC)** แก้ปัญหานี้ด้วยการ**อ่านการเปลี่ยนแปลงข้อมูลโดยตรงจาก Write-Ahead Log (WAL)** ของ PostgreSQL แทนการ poll ตาราง — นี่คือกลไกเดียวกับที่ใช้ทำ **logical replication** ซึ่งเราได้เรียนละเอียดไปแล้วใน **Part 064** (Logical Replication) บทนี้จะทบทวนแนวคิดหลักและเชื่อมโยงเข้ากับ Outbox Pattern โดยเฉพาะ

```
   ┌──────────────────┐
   │   order_db         │
   │  ┌──────────────┐  │
   │  │ outbox_events │  │──── INSERT เข้าตาราง (ระหว่าง transaction ปกติ) ────┐
   │  └──────────────┘  │                                                       │
   │        │            │                                                       ▼
   │        ▼            │                                              ┌──────────────┐
   │   Write-Ahead Log    │───── logical decoding (pgoutput/wal2json) ──▶│  WAL stream    │
   │   (WAL)               │                                              └──────┬───────┘
   └──────────────────┘                                                       │
                                                                                ▼
                                                          ┌──────────────────────────────┐
                                                          │  CDC connector (Debezium /     │
                                                          │  custom logical replication    │
                                                          │  subscriber)                    │
                                                          └───────────────┬──────────────┘
                                                                          ▼
                                                                ┌───────────────────┐
                                                                │  Kafka topic        │
                                                                │  "order-events"      │
                                                                └───────────────────┘
                                                                          │
                                                     ┌────────────────────┼────────────────────┐
                                                     ▼                    ▼                    ▼
                                          inventory-service     payment-service        analytics-service
                                          (subscribe topic)     (subscribe topic)      (subscribe topic)
```

### 909.2 ทบทวน Logical Replication (เชื่อมโยง Part 064)

จาก Part 064 เราเรียนรู้ว่า PostgreSQL logical replication ทำงานผ่านกลไก:

1. **Replication slot** — จองตำแหน่งใน WAL ไว้เพื่อไม่ให้ PostgreSQL ลบ WAL segment ที่ยังไม่ถูกอ่าน
2. **Publication** — กำหนดว่าตาราง/การเปลี่ยนแปลงชนิดไหนบ้างที่จะถูก "publish" ออกไป
3. **Logical decoding plugin** — แปลง WAL record ให้เป็นรูปแบบที่ consumer อ่านได้ (เช่น `pgoutput` ที่มากับ PostgreSQL, หรือ `wal2json`, `decoderbufs`)

การนำมาใช้กับ Outbox Pattern คือการสร้าง publication **เฉพาะตาราง `outbox_events`** เท่านั้น (ไม่ publish ตารางอื่น) เพื่อให้ CDC connector อ่านเฉพาะ event ที่ตั้งใจจะ export ออกไปข้างนอก:

```sql
-- ใน order_db (order-service)
-- 1) ต้องตั้งค่า wal_level = logical ใน postgresql.conf ก่อน (ทบทวน Part 064)
--    แล้ว restart PostgreSQL

-- 2) สร้าง publication เฉพาะตาราง outbox_events
CREATE PUBLICATION order_outbox_publication
    FOR TABLE outbox_events;

-- 3) สร้าง replication slot สำหรับ CDC connector (เช่น Debezium)
--    (โดยทั่วไป Debezium จะสร้าง slot ให้อัตโนมัติเมื่อ deploy connector
--     แต่แสดงคำสั่งไว้เพื่อความเข้าใจกลไกภายใน)
SELECT pg_create_logical_replication_slot('debezium_order_slot', 'pgoutput');
```

```sql
-- ตรวจสอบสถานะ replication slot (ทบทวนจาก Part 064)
SELECT slot_name, plugin, slot_type, active, confirmed_flush_lsn
FROM pg_replication_slots
WHERE slot_name = 'debezium_order_slot';

-- สำคัญ: ต้องมอนิเตอร์ว่า slot ไม่ค้าง (lag ไม่สูงเกินไป) เพราะถ้า CDC connector
-- หยุดทำงานนาน WAL จะสะสมค้างอยู่ในดิสก์จนกินพื้นที่จนเต็ม (ปัญหาเดียวกับที่กล่าวใน Part 064)
SELECT slot_name,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), confirmed_flush_lsn)) AS replication_lag
FROM pg_replication_slots;
```

### 909.3 Debezium — CDC Connector มาตรฐานอุตสาหกรรม

**Debezium** คือ open-source CDC platform (ต่อยอดจาก Kafka Connect) ที่ใช้กันแพร่หลายที่สุดสำหรับทำ CDC จาก PostgreSQL ไปยัง Kafka หลักการทำงาน:

```
┌────────────┐   logical replication   ┌───────────────────┐   produce   ┌───────────────┐
│ PostgreSQL  │ ───────────────────────▶│ Debezium Connector  │────────────▶│ Kafka topic     │
│ (order_db)  │   (pgoutput plugin)      │ (Kafka Connect task) │             │ order_db.public.│
└────────────┘                          └───────────────────┘             │ outbox_events   │
                                                                            └───────────────┘
```

Debezium มี **Outbox Event Router SMT (Single Message Transform)** ที่ออกแบบมาสำหรับ Outbox Pattern โดยเฉพาะ — มันจะอ่านแถวจากตาราง `outbox_events` แล้วแปลง (route) เป็น Kafka message ที่มี:
- **Topic name** ที่กำหนดจาก `aggregate_type` (เช่น topic `order-events` มาจาก `aggregate_type = 'Order'`)
- **Message key** จาก `aggregate_id` (เพื่อรับประกัน ordering ภายใน aggregate เดียวกัน เพราะ Kafka รับประกัน ordering เฉพาะภายใน partition เดียวกัน และ Kafka partition ข้อมูลตาม key)
- **Message value** จาก `payload`

ตัวอย่าง configuration ของ Debezium connector (JSON config ส่งให้ Kafka Connect REST API):

```json
{
  "name": "order-outbox-connector",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "database.hostname": "order-db-host",
    "database.port": "5432",
    "database.user": "debezium_replicator",
    "database.password": "${vault:secret/debezium#password}",
    "database.dbname": "order_db",
    "topic.prefix": "order_db",
    "plugin.name": "pgoutput",
    "publication.name": "order_outbox_publication",
    "slot.name": "debezium_order_slot",
    "table.include.list": "public.outbox_events",

    "transforms": "outbox",
    "transforms.outbox.type": "io.debezium.transforms.outbox.EventRouter",
    "transforms.outbox.table.field.event.id": "event_id",
    "transforms.outbox.table.field.event.key": "aggregate_id",
    "transforms.outbox.table.field.event.type": "event_type",
    "transforms.outbox.table.field.event.payload": "payload",
    "transforms.outbox.route.by.field": "aggregate_type",
    "transforms.outbox.route.topic.replacement": "${routedByValue}.events"
  }
}
```

Config นี้จะสร้าง topic ชื่อ `Order.events` โดยอัตโนมัติ (แทนค่า `${routedByValue}` ด้วยค่าจาก `aggregate_type`) และทุกครั้งที่มีแถวใหม่ insert ใน `outbox_events` Debezium จะจับได้จาก WAL แทบจะทันที (sub-second latency) แล้ว publish ไป Kafka โดยไม่ต้อง polling ฐานข้อมูลเลย

### 909.4 ข้อดีของ CDC เทียบกับ Polling Publisher

| ประเด็น | Polling Publisher | CDC (Debezium/logical replication) |
|---|---|---|
| Latency | ขึ้นกับ polling interval (เช่น 500ms) | ใกล้เคียง real-time (WAL stream) |
| Load บนฐานข้อมูล | เพิ่ม query load ต่อเนื่อง | อ่านจาก WAL ไม่กระทบ query workload โดยตรง |
| Scalability | จำกัดด้วย polling frequency | รองรับ throughput สูงมาก (WAL-based) |
| Infrastructure | ง่าย ไม่ต้องมี component เพิ่ม | ต้องมี Kafka Connect cluster, ZooKeeper/KRaft, เพิ่ม operational complexity |
| Ordering guarantee | ควบคุมได้ตรงไปตรงมา (`ORDER BY created_at`) | ต้องพึ่ง Kafka partitioning ด้วย key ที่เหมาะสม |
| เหมาะกับทีม | ทีมเล็ก, throughput ไม่สูงมาก | ทีมใหญ่, throughput สูง, ต้องการ near real-time |

### 909.5 ข้อควรระวังในการใช้ CDC กับ Microservices

1. **Replication slot ที่ไม่ active จะทำให้ WAL โตไม่จำกัด** — ถ้า Debezium connector ดับไปนาน (เช่น Kafka Connect cluster ล่ม) ต้องมี monitoring และ alert เมื่อ `replication lag` สูงเกิน threshold (ทบทวน Part 064) มิเช่นนั้น disk ของ order_db อาจเต็มจน PostgreSQL หยุดทำงาน
2. **Schema evolution ต้องระวัง** — ถ้าเปลี่ยน schema ของ `outbox_events` (เช่นเพิ่มคอลัมน์) ต้องดูแลให้ Debezium/consumer รองรับ backward compatibility ของ payload JSON (ใช้ schema registry เช่น Confluent Schema Registry ร่วมกับ Avro/Protobuf ช่วยจัดการเรื่องนี้ได้ดีในระบบใหญ่)
3. **อย่า CDC ตารางธุรกิจโดยตรง (เช่น `orders`) แทนที่จะ CDC ผ่าน outbox** — แม้ CDC จากตาราง `orders` ตรงๆ จะทำได้ทางเทคนิค แต่จะทำให้ payload ของ event ผูกติดกับ physical schema ของตาราง (schema ของ internal table รั่วไหลออกไปเป็น public contract ให้ service อื่น) ซึ่งขัดกับหลักการ **information hiding** — Outbox table ทำหน้าที่เป็น "explicit public contract" ที่ควบคุมได้ว่าจะ expose field อะไรบ้าง แยกจาก internal schema
4. **Monitoring consumer lag ที่ปลายทาง (Kafka consumer group lag)** เพิ่มเติมจาก replication lag ของ PostgreSQL — เพราะ CDC มี 2 จุดที่อาจล่าช้า: PostgreSQL → Kafka (replication lag) และ Kafka → consumer service (consumer lag)

### 909.6 การเลือกใช้ Polling Publisher หรือ CDC

| สถานการณ์ | แนะนำ |
|---|---|
| Startup/MVP, throughput ต่ำ (< 100 event/วินาที), ทีมเล็ก | Polling Publisher — ง่าย ไม่ต้องเพิ่ม infrastructure |
| Production ที่ throughput สูง, ต้องการ near real-time, มีทีม platform/infra ดูแล Kafka อยู่แล้ว | CDC ผ่าน Debezium |
| องค์กรที่ใช้ Kafka เป็น central event backbone อยู่แล้ว | CDC (integrate เข้ากับ ecosystem ที่มีอยู่ได้ทันที) |
| ต้องการความง่ายในการ debug และควบคุม ordering แบบตรงไปตรงมา | Polling Publisher |

หลายทีมเริ่มจาก Polling Publisher ในช่วงแรก แล้วค่อย migrate ไป CDC เมื่อ throughput หรือ latency requirement เพิ่มขึ้น — ข้อดีของการออกแบบตาราง `outbox_events` ให้ดีตั้งแต่ต้น (Step 908) คือการ migrate นี้ **ไม่กระทบ consumer เลย** เพราะ event contract (topic, payload schema) ยังคงเดิม เปลี่ยนแค่กลไกการส่งเท่านั้น

---

## สรุปท้ายบท

ใน Part นี้เราได้เรียนรู้การออกแบบฐานข้อมูลสำหรับสถาปัตยกรรม microservices อย่างครบวงจร โดยใช้ระบบ e-commerce (`order-service`, `inventory-service`, `customer-service`, `payment-service`) เป็นตัวอย่างตลอดทั้งบท:

1. **Monolith vs Microservices** — การย้ายไป microservices คือการเปลี่ยนโมเดลความคิดจาก strong consistency ไปเป็น eventual consistency ไม่ใช่แค่การแยกโค้ด
2. **Database per Service** คือรากฐานสำคัญที่สุด — แต่ละ service ต้องเป็นเจ้าของฐานข้อมูลของตัวเอง ห้าม service อื่นเข้าถึงโดยตรง
3. เมื่อแยกฐานข้อมูลแล้ว จะเจอปัญหาหลัก 2 อย่างคือ **ไม่มี JOIN ข้าม service** และ **ไม่มี distributed transaction ง่ายๆ** — ทั้งสองต้องแก้ด้วยแนวคิดและ pattern ใหม่
4. **Shared Database คือ anti-pattern** ที่ต้องหลีกเลี่ยงอย่างเด็ดขาด แม้จะดูสะดวกในระยะสั้น แต่จะทำลาย loose coupling และสร้างปัญหาระยะยาวหนักกว่าเดิม
5. **API Composition** ใช้แก้ปัญหา JOIN ข้าม service สำหรับ query ที่ไม่ซับซ้อนมาก โดยย้ายการรวมข้อมูลไปที่ application layer
6. **Saga Pattern** ใช้แก้ปัญหา distributed transaction ด้วยลำดับ local transaction + compensating transaction โดยมีสอง implementation style คือ **Choreography** (กระจาย ไม่มีศูนย์กลาง เหมาะกับ flow สั้น) และ **Orchestration** (มีศูนย์กลาง เหมาะกับ flow ซับซ้อน)
7. **Outbox Pattern** แก้ปัญหา dual write โดยเขียน event ลงตาราง `outbox_events` ในทรานแซคชันเดียวกับข้อมูลจริง รับประกัน atomicity ผ่าน ACID ปกติของ PostgreSQL
8. **Change Data Capture (CDC)** ใช้ logical replication (ทบทวน Part 064) หรือ Debezium อ่านตาราง outbox จาก WAL โดยตรง ให้ latency ต่ำและ scale ได้ดีกว่า polling

Pattern เหล่านี้ทั้งหมดล้วนหมุนรอบหลักการเดียวกัน: **ยอมรับ eventual consistency ในระดับ data ที่ข้าม service boundary แลกกับ loose coupling ที่ทำให้แต่ละทีมพัฒนา, deploy, และ scale ได้อย่างอิสระ** ซึ่งเป็นเป้าหมายที่แท้จริงของสถาปัตยกรรม microservices

ใน **Part 092** เราจะต่อยอดจาก Outbox และ CDC ที่เรียนในบทนี้ ไปสู่แนวคิด **Event Sourcing และ CQRS** อย่างละเอียด ซึ่งเป็นวิวัฒนาการขั้นถัดไปของการออกแบบระบบแบบ event-driven

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1

อธิบายว่าทำไม `postgres_fdw` ที่ใช้เพื่อ JOIN ตารางข้าม microservice ถึงยังถือเป็น anti-pattern แม้จะทำงานได้ทางเทคนิคก็ตาม

<details>
<summary>เฉลย</summary>

แม้ `postgres_fdw` จะทำให้เขียน SQL JOIN ข้าม database ได้จริงทางเทคนิค แต่มันยังคงเป็น anti-pattern เพราะ:

1. **Tight coupling ที่ schema level** — ถ้าฝั่ง customer_db เปลี่ยนชื่อคอลัมน์หรือลบคอลัมน์ foreign table ใน order_db จะพังทันที เหมือนกับ shared database anti-pattern
2. **ข้าม service boundary โดยไม่ผ่าน API contract** — ทำให้ไม่มี versioning, validation, หรือ business logic ของ customer-service เข้ามาเกี่ยวข้อง (เช่น authorization, data masking)
3. **Query planner คาดเดา cost ยาก** — cardinality estimation ข้าม network ไม่แม่นยำ ทำให้ query plan อาจแย่โดยไม่รู้ตัว
4. **สร้าง synchronous network dependency ที่มองไม่เห็น** — ถ้า customer_db ช้าหรือดับ query ของ order-service ก็จะช้า/ดับตามไปด้วย ทำลายหลักการ failure isolation ของ microservices
5. **ทำลายความเป็นอิสระในการ scale/เปลี่ยน technology** ของแต่ละ service ซึ่งเป็นเหตุผลหลักที่ทำ microservices ตั้งแต่แรก

`postgres_fdw` เหมาะกับงาน migration ชั่วคราว หรือ data warehousing/analytics ที่ยอมรับ coupling ระดับหนึ่งได้ แต่ไม่เหมาะเป็นสถาปัตยกรรมถาวรสำหรับ production microservices
</details>

---

### แบบฝึกหัดที่ 2

ออกแบบตาราง `orders` และ `order_items` ใน `order_db` ให้เหมาะกับหลักการ database per service (ไม่มี FK ข้าม database) พร้อมอธิบายว่าทำไม `order_items.product_name` และ `unit_price` ถึงต้อง denormalize เก็บไว้แทนที่จะอ้างอิงจาก inventory-service ทุกครั้ง

<details>
<summary>เฉลย</summary>

```sql
CREATE TABLE orders (
    id              BIGSERIAL PRIMARY KEY,
    customer_id     BIGINT NOT NULL,   -- reference เท่านั้น ไม่มี FK จริง
    status          TEXT NOT NULL DEFAULT 'PENDING',
    total_amount    NUMERIC(12,2) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE order_items (
    id              BIGSERIAL PRIMARY KEY,
    order_id        BIGINT NOT NULL REFERENCES orders(id),  -- FK ภายใน DB เดียวกัน ทำได้ปกติ
    product_id      BIGINT NOT NULL,   -- reference ไป inventory-service เท่านั้น
    product_name    TEXT NOT NULL,     -- snapshot ณ เวลาสั่งซื้อ
    unit_price      NUMERIC(12,2) NOT NULL,  -- snapshot ณ เวลาสั่งซื้อ
    quantity        INT NOT NULL CHECK (quantity > 0)
);
```

เหตุผลที่ต้อง denormalize `product_name` และ `unit_price`:

1. **ความถูกต้องเชิงธุรกิจ (business correctness)** — ใบสั่งซื้อต้องสะท้อนราคาและชื่อสินค้า "ณ เวลาที่สั่งซื้อ" ไม่ใช่ราคาปัจจุบัน ถ้า inventory-service เปลี่ยนราคาสินค้าภายหลัง order เก่าต้องไม่เปลี่ยนตาม (นี่คือ requirement ทางธุรกิจ ไม่ใช่แค่ทางเทคนิค)
2. **Performance** — ไม่ต้องเรียก API ไป inventory-service ทุกครั้งที่ต้องแสดงรายละเอียด order เก่า ลด latency และลด coupling แบบ synchronous
3. **Availability** — แม้ inventory-service จะล่ม order-service ก็ยังแสดงประวัติ order เก่าได้ครบถ้วน เพราะข้อมูลที่จำเป็นอยู่ใน order_db เอง
4. **Loose coupling** — order-service ไม่ต้อง "พึ่งพา" inventory-service แบบ real-time เพื่อแสดงข้อมูลพื้นฐานของ order ที่สร้างไปแล้ว
</details>

---

### แบบฝึกหัดที่ 3

ในระบบ e-commerce ที่แยก service แล้ว มีความต้องการใหม่คือหน้า "Order History" ต้องแสดงรายการ order 50 รายการล่าสุดพร้อมชื่อลูกค้าของแต่ละ order ให้ออกแบบ solution ด้วย API Composition Pattern พร้อมแก้ปัญหา N+1 query

<details>
<summary>เฉลย</summary>

```javascript
// composer (เช่น order-history-api หรือ BFF)
async function getOrderHistory(page = 1, pageSize = 50) {
  // 1) ดึงรายการ order 50 รายการล่าสุดจาก order-service (1 call)
  const orders = await orderServiceClient.listRecentOrders({ page, pageSize });

  // 2) รวบรวม customer_id ที่ไม่ซ้ำกัน
  const customerIds = [...new Set(orders.map(o => o.customerId))];

  // 3) เรียก customer-service แบบ batch (1 call เดียว แทนที่จะ loop 50 ครั้ง)
  //    ต้องมี endpoint ฝั่ง customer-service ที่รองรับ GET /customers?ids=1,2,3,...
  const customers = await customerServiceClient.getCustomersByIds(customerIds);
  const customerMap = new Map(customers.map(c => [c.id, c]));

  // 4) รวมผลลัพธ์ในหน่วยความจำ
  return orders.map(o => ({
    orderId: o.id,
    status: o.status,
    totalAmount: o.totalAmount,
    customerName: customerMap.get(o.customerId)?.fullName ?? 'Unknown',
  }));
}
```

จุดสำคัญของการแก้ N+1 คือ customer-service ต้องมี **batch endpoint** (`GET /customers?ids=1,2,3,...` ที่ภายในใช้ `WHERE id = ANY($1::bigint[])`) เพื่อให้ composer เรียก API แค่ 2 ครั้งทั้งหมด (1 ครั้งไป order-service, 1 ครั้งไป customer-service) แทนที่จะเรียก customer-service ทีละ order (51 ครั้ง)

หากต้องการ query ลักษณะนี้บ่อยมากและมี filter/sort ซับซ้อนขึ้นเรื่อยๆ ควรพิจารณาสร้าง read model แยกต่างหาก (CQRS) ที่ sync ข้อมูลผ่าน event แทน ซึ่งจะกล่าวถึงใน Part 092
</details>

---

### แบบฝึกหัดที่ 4

อธิบายว่าทำไม Two-Phase Commit (2PC) ถึงไม่เหมาะกับการใช้ทำ distributed transaction ข้าม microservices ในทางปฏิบัติ ทั้งที่ PostgreSQL รองรับ `PREPARE TRANSACTION` อยู่แล้ว

<details>
<summary>เฉลย</summary>

แม้ PostgreSQL จะรองรับ 2PC ผ่าน `PREPARE TRANSACTION` / `COMMIT PREPARED` / `ROLLBACK PREPARED` แต่ไม่เหมาะกับ microservices เพราะ:

1. **ต้องการ transaction coordinator กลาง** ที่กลายเป็น single point of failure — ถ้า coordinator ล่มระหว่าง phase 2 (commit) resource ทุกตัวที่ prepare ไว้แล้วจะค้างอยู่ในสถานะ "in-doubt" locked ไว้จนกว่า coordinator จะกลับมา
2. **Blocking protocol** — ระหว่างที่รอ vote จากทุก participant, resource (row, table) จะถูก lock ค้างไว้ ทำให้ throughput ต่ำลงมาก โดยเฉพาะเมื่อ participant ตัวใดตัวหนึ่งตอบช้า
3. **ขัดกับ availability ที่ microservices ต้องการ** — 2PC เอนเอียงไปทาง strong consistency แลกกับ availability อย่างหนัก (ตาม CAP theorem) ซึ่งขัดกับเป้าหมายหลักของ microservices ที่ต้องการให้แต่ละ service ทำงานอิสระได้แม้ service อื่นจะช้า/ดับ
4. **Service ภายนอกไม่รองรับ** — payment gateway ภายนอก (Stripe, Omise ฯลฯ) ไม่มีทาง participate ใน 2PC protocol ของเราได้อยู่แล้ว ทำให้ 2PC ใช้ไม่ได้จริงกับ flow ที่มี external dependency
5. **Coupling ระดับ synchronous สูงมาก** — ทุก service ต้อง "รอ" กันแบบ synchronous ตลอด transaction ทำลาย loose coupling ที่เป็นจุดประสงค์หลักของการแยก microservices

ด้วยเหตุนี้อุตสาหกรรมจึงหันมาใช้ **Saga Pattern** ที่ยอมรับ eventual consistency แลกกับ availability และ loose coupling ที่ดีกว่ามาก
</details>

---

### แบบฝึกหัดที่ 5

ออกแบบ compensating transaction สำหรับแต่ละ forward transaction ต่อไปนี้ในบริบท order saga: (a) `CreateOrder`, (b) `ReserveStock`, (c) `ChargePayment` พร้อมระบุว่า transaction ใดใน 3 ตัวนี้ที่ไม่ควรวางไว้เป็น step สุดท้ายของ saga เพราะ compensate ยาก

<details>
<summary>เฉลย</summary>

| Forward | Compensating |
|---|---|
| `CreateOrder` (INSERT order status=PENDING) | `CancelOrder` (UPDATE status=CANCELLED — ไม่ลบแถว เพื่อรักษา audit trail) |
| `ReserveStock` (ลด quantity_available, เพิ่ม quantity_reserved) | `ReleaseStock` (คืนค่ากลับ — ต้อง idempotent ด้วย `WHERE quantity_reserved >= n`) |
| `ChargePayment` (เรียกเก็บเงินจริงผ่าน payment gateway) | `RefundPayment` (คืนเงิน — มักไม่ instant, ต้องมี state กลาง เช่น `REFUND_PENDING`) |

ในสามตัวนี้ **`ChargePayment` เป็นตัวที่ compensate ยากที่สุด** เพราะ:
- การ refund มักไม่ instant (บาง payment gateway ใช้เวลาหลายวัน) จึงไม่ควรวาง `ChargePayment` ไว้เป็น step แรกๆ ของ saga เพราะถ้า step หลังจากนั้นล้มเหลว จะต้องเข้า flow refund ที่ซับซ้อนกว่าการยกเลิก order/release stock
- หลักการออกแบบที่ดีคือ **จัดลำดับ saga ให้ step ที่ compensate ยาก/แพง อยู่ท้ายสุด** (หลังจาก step ที่มีโอกาสล้มเหลวสูงกว่าผ่านไปหมดแล้ว) — ในตัวอย่างของบทนี้จึงจัดลำดับเป็น `ReserveStock` ก่อน `ChargePayment` เพราะ "stock ไม่พอ" เป็นเหตุผลที่พบบ่อยกว่าและ compensate ง่ายกว่า (release stock ทำได้ทันที ไม่ต้องรอ)
</details>

---

### แบบฝึกหัดที่ 6

เปรียบเทียบข้อดีข้อเสียของ Saga แบบ Choreography และ Orchestration แล้วเลือกว่า flow ต่อไปนี้เหมาะกับแบบไหนมากกว่า พร้อมเหตุผล: "checkout flow ที่มี 6 steps รวม fraud detection, loyalty point calculation, multi-currency conversion, และ manual review สำหรับ order มูลค่าสูง"

<details>
<summary>เฉลย</summary>

Flow นี้ควรใช้ **Orchestration** เพราะ:

1. **จำนวน step มาก (6 steps) และมี branching logic ซับซ้อน** (เช่น order มูลค่าสูงต้องมี manual review step เพิ่ม) — Choreography จะทำให้ logic การตัดสินใจว่า "จะไป step ไหนต่อ" กระจายอยู่ในหลาย service ทำให้เข้าใจภาพรวมยากมาก
2. **ต้องการ visibility และ debuggability สูง** — เมื่อมีปัญหา (เช่น order ค้างที่ manual review) ทีม support ต้องเห็นสถานะปัจจุบันของ saga ได้จากที่เดียว (saga_state table) แทนที่จะไล่ event log ข้าม 6 service
3. **มี conditional step** (manual review เฉพาะ order มูลค่าสูง) — orchestrator จัดการ conditional logic นี้ได้ตรงไปตรงมาด้วย if/else ใน orchestrator code ในขณะที่ choreography ต้องอาศัย event ที่ซับซ้อนขึ้น (เช่น ต้องมี event แยกสำหรับ high-value order)
4. **การเพิ่ม/แก้ step ในอนาคต** (เช่นเพิ่ม step ใหม่ระหว่าง fraud detection กับ payment) ทำได้ง่ายกว่ามากถ้าแก้ที่ orchestrator จุดเดียว เทียบกับการต้องประสานหลายทีมแก้ event subscription พร้อมกันใน choreography

Choreography จะเหมาะกว่าถ้า flow นี้มีแค่ 2-3 step ตรงไปตรงมา ไม่มี branching ซับซ้อน
</details>

---

### แบบฝึกหัดที่ 7

เขียน SQL สร้างตาราง `outbox_events` พร้อม index ที่เหมาะสม และอธิบายว่าทำไมต้อง insert ลงตารางนี้ "ในทรานแซคชันเดียวกัน" กับการเขียนข้อมูลธุรกิจ ไม่ใช่เรียก `INSERT` แยกกันคนละ transaction

<details>
<summary>เฉลย</summary>

```sql
CREATE TABLE outbox_events (
    id              BIGSERIAL PRIMARY KEY,
    aggregate_type  TEXT NOT NULL,
    aggregate_id    TEXT NOT NULL,
    event_type      TEXT NOT NULL,
    payload         JSONB NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    processed_at    TIMESTAMPTZ,
    event_id        UUID NOT NULL DEFAULT gen_random_uuid()
);

CREATE INDEX idx_outbox_unprocessed
    ON outbox_events (created_at)
    WHERE processed_at IS NULL;
```

เหตุผลที่ต้อง insert ในทรานแซคชันเดียวกัน:

ถ้าแยก transaction กัน (เช่น commit ข้อมูลธุรกิจก่อน แล้วค่อย insert outbox event ใน transaction ที่สอง) จะเกิด **dual write problem** — มีความเป็นไปได้ที่ transaction แรกสำเร็จ (ข้อมูลธุรกิจถูกบันทึก) แต่ transaction ที่สองล้มเหลว (เช่น application crash พอดีตรงกลาง, connection หลุด) ทำให้ event หายไปทั้งที่ข้อมูลจริงถูกบันทึกแล้ว — service อื่นๆ ที่ subscribe event นี้จะไม่มีวันรู้ว่ามีการเปลี่ยนแปลงเกิดขึ้น ทำให้ระบบ inconsistent อย่างถาวรโดยไม่มีทาง detect ได้เลยถ้าไม่มี reconciliation job แยกต่างหาก

เมื่อ insert ทั้งสองอยู่ใน transaction เดียวกัน PostgreSQL รับประกัน atomicity ตามปกติของ ACID: ถ้า COMMIT สำเร็จ ทั้งข้อมูลธุรกิจและ event ถูกบันทึกพร้อมกันแน่นอน ถ้า transaction ถูก rollback (ไม่ว่าด้วยเหตุผลอะไร) ทั้งคู่จะไม่ถูกบันทึกเลยทั้งคู่เช่นกัน — ไม่มีทางเกิดสถานะ "กลางๆ" ที่ inconsistent
</details>

---

### แบบฝึกหัดที่ 8

Consumer service (inventory-service) รับ event ซ้ำสองครั้งจาก message broker เนื่องจาก network retry ให้ออกแบบกลไก idempotency ด้วย SQL เพื่อป้องกันไม่ให้ business logic (การตัด stock) ถูกรันซ้ำสองครั้ง

<details>
<summary>เฉลย</summary>

```sql
-- ตารางเก็บ event_id ที่ประมวลผลแล้ว
CREATE TABLE processed_events (
    event_id     UUID PRIMARY KEY,
    processed_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

```sql
-- Transaction เดียวที่รวมทั้งการเช็ค idempotency และ business logic
BEGIN;

WITH inserted AS (
    INSERT INTO processed_events (event_id)
    VALUES ($1)
    ON CONFLICT (event_id) DO NOTHING
    RETURNING event_id
)
UPDATE stock_levels
SET quantity_available = quantity_available - $3,
    quantity_reserved   = quantity_reserved + $3
WHERE product_id = $2
  AND EXISTS (SELECT 1 FROM inserted);   -- update เฉพาะถ้า insert สำเร็จจริง (ไม่ใช่ event ซ้ำ)

COMMIT;
```

หลักการทำงาน: ใช้ CTE (`WITH inserted AS (...)`) เพื่อพยายาม insert `event_id` ก่อน ถ้า event_id นี้เคยถูกประมวลผลมาแล้ว `ON CONFLICT DO NOTHING` จะทำให้ CTE ไม่มีแถวคืนกลับมา ส่งผลให้ `EXISTS (SELECT 1 FROM inserted)` เป็น false และ `UPDATE stock_levels` จะไม่ update อะไรเลย (WHERE clause ไม่ match) — ทำให้ business logic รันแค่ครั้งเดียวแน่นอน แม้ event จะถูกส่งมาซ้ำกี่ครั้งก็ตาม และทั้งหมดนี้อยู่ใน transaction เดียวกัน จึงเป็น atomic operation ที่ปลอดภัยจาก race condition ด้วย
</details>

---

### แบบฝึกหัดที่ 9

อธิบายความแตกต่างระหว่าง Polling Publisher และ CDC (ผ่าน logical replication/Debezium) ในการอ่านข้อมูลจากตาราง outbox แล้วบอกว่าทำไมการออกแบบตาราง `outbox_events` ให้ดีตั้งแต่แรกทำให้การ migrate จาก Polling Publisher ไป CDC ในอนาคต "ไม่กระทบ consumer เลย"

<details>
<summary>เฉลย</summary>

**Polling Publisher**: process แยกต่างหาก query ตาราง `outbox_events` เป็นระยะ (เช่นทุก 500ms) ด้วย `SELECT ... WHERE processed_at IS NULL ... FOR UPDATE SKIP LOCKED` แล้วส่งไป message broker จากนั้น mark `processed_at`

**CDC**: อ่านการเปลี่ยนแปลงโดยตรงจาก WAL ผ่าน logical replication (publication + replication slot) หรือใช้ Debezium ซึ่งเป็น Kafka Connect connector ที่ทำ logical decoding ให้อัตโนมัติ ไม่ต้อง query ฐานข้อมูลเลย latency ต่ำกว่ามาก และไม่เพิ่ม load จากการ polling

**ทำไม migrate ไม่กระทบ consumer**: เพราะไม่ว่าจะใช้กลไกไหนในการ "ส่ง" event (polling หรือ CDC) สิ่งที่ consumer (inventory-service) เห็นคือ **event message บน topic เดียวกัน ด้วย payload schema เดียวกัน** (มาจากคอลัมน์ `event_type` และ `payload` ของตาราง outbox) — consumer ไม่รู้และไม่สนใจว่า producer ฝั่ง order-service ใช้กลไกอะไรส่ง event มา ตราบใดที่ **contract ของ event (topic name, key, payload structure) ไม่เปลี่ยน** การเปลี่ยนกลไกภายในจาก Polling Publisher ไปเป็น Debezium จึงเป็นการเปลี่ยนแค่ "internal implementation detail" ของ order-service เท่านั้น ซึ่งเป็นผลพลอยได้จากการยึดหลัก **Database per Service + explicit contract ผ่าน outbox table** ตั้งแต่ Step 902 และ 908
</details>

---

### แบบฝึกหัดที่ 10 (แบบฝึกหัดรวม)

ออกแบบสถาปัตยกรรม database แบบ microservices เต็มรูปแบบสำหรับระบบ e-commerce ที่มี 4 services: `order-service`, `inventory-service`, `customer-service`, `payment-service` โดยต้องมี:
1. Diagram แสดงว่าแต่ละ service มีฐานข้อมูลอะไร ตารางหลักอะไรบ้าง
2. Saga flow แบบ orchestration สำหรับ checkout (สร้าง order → reserve stock → charge payment → confirm order) พร้อม compensating transaction ของแต่ละ step
3. การใช้ Outbox Pattern ใน order-service เพื่อ publish event `OrderCreated`

<details>
<summary>เฉลย</summary>

**1) สถาปัตยกรรมโดยรวม**

```
┌────────────────┐  ┌────────────────────┐  ┌────────────────┐  ┌────────────────┐
│ order-service    │  │ inventory-service     │  │ customer-service │  │ payment-service  │
└────────┬────────┘  └──────────┬──────────┘  └────────┬────────┘  └────────┬────────┘
         ▼                      ▼                       ▼                    ▼
┌────────────────┐  ┌────────────────────┐  ┌────────────────┐  ┌────────────────┐
│ order_db         │  │ inventory_db          │  │ customer_db      │  │ payment_db       │
│ - orders          │  │ - products             │  │ - customers       │  │ - payments        │
│ - order_items      │  │ - stock_levels          │  │ - addresses        │  │ - refunds          │
│ - outbox_events    │  │ - outbox_events         │  │                     │  │ - outbox_events    │
│ - processed_events  │  │ - processed_events        │  │                     │  │ - processed_events   │
└────────────────┘  └────────────────────┘  └────────────────┘  └────────────────┘
         ▲                      ▲                       ▲                    ▲
         └──────────────────────┴───────────┬───────────┴────────────────────┘
                                              │
                                  ┌───────────────────────┐
                                  │  Saga Orchestrator       │
                                  │  (saga_db: order_sagas,   │
                                  │   order_saga_steps)        │
                                  └───────────────────────┘
```

**2) Saga flow (orchestration) สำหรับ checkout**

| Step | Forward command | Service | Compensating command |
|---|---|---|---|
| 1 | `CreateOrder` (status=PENDING) | order-service | `CancelOrder` (status=CANCELLED) |
| 2 | `ReserveStock` | inventory-service | `ReleaseStock` |
| 3 | `ChargePayment` | payment-service | `RefundPayment` |
| 4 | `ConfirmOrder` (status=CONFIRMED) | order-service | — (ไม่มี, เป็น step สุดท้าย) |

```python
STEPS = [
    ("CreateOrder",   "order-service",     "CancelOrder"),
    ("ReserveStock",  "inventory-service", "ReleaseStock"),
    ("ChargePayment", "payment-service",   "RefundPayment"),
    ("ConfirmOrder",  "order-service",      None),
]

def run_checkout_saga(saga_id, request):
    completed = []
    for command, service, compensate in STEPS:
        result = send_command(service, command, request, saga_id)
        if not result.success:
            for svc, comp in reversed(completed):
                if comp:
                    send_command(svc, comp, saga_id=saga_id)
            mark_saga_failed(saga_id)
            return False
        completed.append((service, compensate))
    mark_saga_completed(saga_id)
    return True
```

Orchestrator บันทึกสถานะทุก step ลง `order_sagas` / `order_saga_steps` (ตามที่ออกแบบใน Step 907.4) เพื่อรองรับ crash recovery และ monitoring stuck saga

**3) Outbox Pattern ใน order-service**

```sql
BEGIN;

INSERT INTO orders (id, customer_id, status, total_amount)
VALUES (2001, 1001, 'PENDING', 2400.00);

INSERT INTO order_items (order_id, product_id, product_name, unit_price, quantity)
VALUES (2001, 88, 'รองเท้าผ้าใบ', 2400.00, 1);

INSERT INTO outbox_events (aggregate_type, aggregate_id, event_type, payload)
VALUES (
    'Order', '2001', 'OrderCreated',
    jsonb_build_object(
        'orderId', 2001,
        'customerId', 1001,
        'items', jsonb_build_array(
            jsonb_build_object('productId', 88, 'quantity', 1)
        ),
        'totalAmount', 2400.00
    )
);

COMMIT;
```

จากนั้น message relay (polling หรือ CDC ผ่าน Debezium ตาม Step 908-909) จะอ่านตาราง `outbox_events` แล้ว publish event `OrderCreated` ไปยัง Kafka topic `Order.events` ซึ่ง `saga-orchestrator-service` หรือ `inventory-service` (ในกรณี choreography) จะ subscribe เพื่อดำเนิน step ถัดไปของ saga

**สรุปการออกแบบทั้งหมด**: ทุก service มีฐานข้อมูลแยกกันเด็ดขาด (Step 902), ไม่มี query ข้าม service โดยตรง (หลีกเลี่ยง Step 904 anti-pattern), การรวมข้อมูลข้าม service ทำผ่าน API Composition สำหรับ read (Step 905) และผ่าน Saga สำหรับ write ที่ต้อง atomic ข้าม service (Step 906-907) โดยการสื่อสารแบบ asynchronous ทั้งหมดรับประกันความน่าเชื่อถือด้วย Outbox Pattern (Step 908) และ scale ได้ด้วย CDC (Step 909)
</details>

---

**บทถัดไป:** [Part 092: Event Sourcing และ CQRS](./part-092-event-sourcing-cqrs.md)
