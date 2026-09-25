# Event Sourcing และ CQRS กับ PostgreSQL

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับโลก/ผู้เชี่ยวชาญ | Part 092

---

## เป้าหมายการเรียนรู้

เมื่อเรียนจบบทนี้ ผู้เรียนจะสามารถ:

1. อธิบายแนวคิด **Event Sourcing** และเข้าใจความแตกต่างจากการเก็บข้อมูลแบบดั้งเดิม (state-based persistence)
2. ออกแบบและสร้าง **Event Store** บน PostgreSQL ด้วยตาราง `order_events` ที่รองรับการเก็บ event แบบ append-only
3. เข้าใจและ implement **Optimistic Concurrency Control** ด้วย UNIQUE constraint เพื่อป้องกัน race condition เวลาเขียน event พร้อมกัน
4. เขียน query เพื่อ **Replay Event** และประมวลผล event ทีละตัวเพื่อสร้าง current state ของ aggregate
5. ออกแบบและใช้งาน **Snapshot Pattern** เพื่อลดต้นทุนการ replay event จำนวนมาก
6. อธิบายแนวคิด **CQRS (Command Query Responsibility Segregation)** และเหตุผลที่ต้องแยก write model กับ read model
7. ผสาน CQRS เข้ากับ Event Sourcing โดยใช้ event store เป็น write model และ materialized view / projection table เป็น read model
8. สร้าง **Projection** จาก event stream ด้วย trigger หรือ application-level event handler
9. Implement ระบบ Event Sourcing + CQRS แบบสมบูรณ์สำหรับ order aggregate ตั้งแต่ต้นจนจบ

---

## เตรียมข้อมูล

ตลอดบทนี้เราจะใช้โดเมน **e-commerce order** เป็นตัวอย่างหลัก โดยมี event store กลางคือตาราง `order_events` ดังนี้

```sql
-- เปิดใช้ extension ที่จำเป็น
CREATE EXTENSION IF NOT EXISTS "pgcrypto";  -- สำหรับ gen_random_uuid()

-- Event Store หลัก: เก็บทุก event ของทุก order aggregate
CREATE TABLE order_events (
    event_id       BIGSERIAL PRIMARY KEY,
    aggregate_id   UUID NOT NULL,
    event_type     VARCHAR(50) NOT NULL,
    event_data     JSONB NOT NULL,
    event_version  INTEGER NOT NULL,
    created_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (aggregate_id, event_version)
);

-- ดัชนีช่วยให้ query event ตาม aggregate ได้เร็ว (เรียงตาม version)
CREATE INDEX idx_order_events_aggregate
    ON order_events (aggregate_id, event_version);

-- ดัชนีช่วย query ตามประเภท event (เช่น หา event ทั้งหมดที่เป็น OrderShipped)
CREATE INDEX idx_order_events_type
    ON order_events (event_type);

-- ดัชนี GIN สำหรับ query เข้าไปใน event_data (JSONB)
CREATE INDEX idx_order_events_data_gin
    ON order_events USING GIN (event_data);
```

โครงสร้างนี้คือหัวใจของทั้งบท: `aggregate_id` คือ UUID ของ order แต่ละใบ, `event_type` บอกว่าเกิดอะไรขึ้น (เช่น `OrderCreated`, `ItemAdded`), `event_data` เก็บ payload เป็น JSON, และ `event_version` คือเลขลำดับ event ภายใน aggregate นั้น (เริ่มที่ 1 แล้วไล่ขึ้นทีละ 1) — คู่ `(aggregate_id, event_version)` ต้องไม่ซ้ำกัน ซึ่งจะเป็นกลไกป้องกัน concurrency conflict ที่เราจะพูดถึงใน Step 914

---

## Step 911: Event Sourcing คืออะไร

### แนวคิดหลัก

ระบบฐานข้อมูลแบบดั้งเดิมที่เราคุ้นเคยกันมาตลอดหลักสูตรนี้ ใช้แนวคิดที่เรียกว่า **state-based persistence** คือเราเก็บ "สถานะปัจจุบัน" ของข้อมูลไว้ในตาราง แล้วเวลามีการเปลี่ยนแปลง เราก็ `UPDATE` แถวนั้นทับของเดิมไปเลย ข้อมูลในอดีตจะหายไปทันทีที่ `UPDATE` สำเร็จ (เว้นแต่จะมี audit table แยกต่างหาก)

**Event Sourcing** กลับกันโดยสิ้นเชิง: แทนที่จะเก็บ "สถานะปัจจุบัน" เราเก็บ **ลำดับเหตุการณ์ทั้งหมด (sequence of events)** ที่เคยเกิดขึ้นกับ entity นั้น ๆ ตั้งแต่ต้นจนถึงปัจจุบัน สถานะปัจจุบันไม่ได้ถูกเก็บโดยตรง แต่คำนวณได้จากการ "replay" (เล่นซ้ำ) event ทั้งหมดตามลำดับ

ลองเปรียบเทียบกับบัญชีธนาคาร: ธนาคารไม่ได้เก็บแค่ "ยอดเงินคงเหลือ" อย่างเดียว แต่เก็บ **รายการเดินบัญชี (transaction log)** ทุกรายการ ฝากเท่าไหร่ ถอนเท่าไหร่ เมื่อไหร่ ยอดคงเหลือคือผลรวมของรายการทั้งหมด — นี่คือแก่นของ Event Sourcing

### ตัวอย่างแนวคิด: Order Lifecycle

ลองนึกภาพ order หนึ่งใบที่มี lifecycle ดังนี้:

```
1. OrderCreated      -> ลูกค้าสร้าง order ใหม่
2. ItemAdded         -> เพิ่มสินค้า A จำนวน 2 ชิ้น
3. ItemAdded         -> เพิ่มสินค้า B จำนวน 1 ชิ้น
4. ItemRemoved       -> ลบสินค้า A ออก 1 ชิ้น
5. OrderShipped      -> จัดส่งแล้ว
```

ในระบบ state-based เราจะเห็นแค่ order ที่มีสถานะ `shipped` พร้อมรายการสินค้าสุดท้าย (สินค้า A จำนวน 1 ชิ้น, สินค้า B จำนวน 1 ชิ้น) — แต่ **ไม่มีทางรู้เลยว่าเคยมีการเพิ่ม A 2 ชิ้นแล้วลบออก 1 ชิ้น**

ในระบบ Event Sourcing เรามี event ทั้ง 5 รายการเก็บไว้ครบถ้วน เราสามารถตอบคำถามได้ทั้งหมด: order นี้เคยเปลี่ยนแปลงอะไรบ้าง, ใคร/เมื่อไหร่ทำอะไร, และแม้แต่ "state ของ order นี้ ณ เวลา 14:00 น. เป็นอย่างไร" (time travel) ก็ตอบได้โดยการ replay event เฉพาะที่เกิดก่อนเวลานั้น

### ลองสร้าง event ชุดแรกใน event store

```sql
-- สร้าง order ใหม่ (aggregate ใหม่ จึงเริ่มที่ event_version = 1)
INSERT INTO order_events (aggregate_id, event_type, event_data, event_version)
VALUES (
    'a1b2c3d4-0001-4000-8000-000000000001',
    'OrderCreated',
    jsonb_build_object(
        'customer_id', 'cust-1001',
        'currency', 'THB',
        'created_at', now()
    ),
    1
);

-- เพิ่มสินค้า A
INSERT INTO order_events (aggregate_id, event_type, event_data, event_version)
VALUES (
    'a1b2c3d4-0001-4000-8000-000000000001',
    'ItemAdded',
    jsonb_build_object(
        'sku', 'SKU-A100',
        'name', 'เมาส์ไร้สาย',
        'quantity', 2,
        'unit_price', 350.00
    ),
    2
);

-- เพิ่มสินค้า B
INSERT INTO order_events (aggregate_id, event_type, event_data, event_version)
VALUES (
    'a1b2c3d4-0001-4000-8000-000000000001',
    'ItemAdded',
    jsonb_build_object(
        'sku', 'SKU-B200',
        'name', 'คีย์บอร์ดกลไก',
        'quantity', 1,
        'unit_price', 1290.00
    ),
    3
);
```

ตรวจสอบ event ที่เก็บไว้:

```sql
SELECT event_id, event_type, event_version, event_data, created_at
FROM order_events
WHERE aggregate_id = 'a1b2c3d4-0001-4000-8000-000000000001'
ORDER BY event_version;
```

```
 event_id |  event_type   | event_version |                          event_data                          |          created_at
----------+---------------+---------------+---------------------------------------------------------------+-------------------------------
        1 | OrderCreated  |             1 | {"currency": "THB", "customer_id": "cust-1001", ...}          | 2026-09-25 09:00:01.123+07
        2 | ItemAdded     |             2 | {"sku": "SKU-A100", "name": "เมาส์ไร้สาย", "quantity": 2, ...} | 2026-09-25 09:00:05.456+07
        3 | ItemAdded     |             3 | {"sku": "SKU-B200", "name": "คีย์บอร์ดกลไก", "quantity": 1, ...}| 2026-09-25 09:00:12.789+07
(3 rows)
```

### หลักการสำคัญของ Event Sourcing

1. **Immutable (แก้ไขไม่ได้)** — event ที่เขียนแล้วห้ามแก้หรือลบ ตาราง `order_events` เป็น append-only เท่านั้น ถ้าทำผิดต้องเขียน event ใหม่เพื่อ "แก้ไข" ไม่ใช่ UPDATE ของเดิม
2. **Ordered (มีลำดับชัดเจน)** — event แต่ละตัวมี `event_version` ที่เรียงต่อเนื่องกัน ทำให้ replay ได้ถูกต้องตามลำดับเวลาที่เกิดขึ้นจริง
3. **Source of Truth คือ event ไม่ใช่ state** — state ปัจจุบันเป็นเพียง "ผลลัพธ์ที่คำนวณได้" (derived data) ไม่ใช่ข้อมูลต้นทาง

> **ควรระวัง:** เราสามารถบังคับความ immutable ในระดับ database ได้ด้วย `REVOKE UPDATE, DELETE ON order_events FROM application_role;` เพื่อให้ application ทำได้แค่ `INSERT` และ `SELECT` เท่านั้น — จะพูดถึงรายละเอียดเรื่อง permission และ security เพิ่มเติมใน Step 913

---

## Step 912: เปรียบเทียบ Event Sourcing กับการเก็บข้อมูลแบบดั้งเดิม

### ตารางเปรียบเทียบ

| ประเด็น | State-based (ดั้งเดิม) | Event Sourcing |
|---|---|---|
| สิ่งที่เก็บ | สถานะปัจจุบันเท่านั้น | ลำดับ event ทั้งหมด |
| การเขียน | `UPDATE`/`DELETE` ทับข้อมูลเดิม | `INSERT` เพิ่ม event ใหม่เท่านั้น (append-only) |
| Audit trail | ต้องสร้างระบบแยก (trigger, audit table) | มีในตัวโดยธรรมชาติ (event log คือ audit log) |
| Time travel | ทำไม่ได้ เว้นแต่มี history table | replay event ถึงจุดเวลาใดก็ได้ |
| Debug ปัญหาที่เกิดในอดีต | ยาก เพราะข้อมูลถูกทับไปแล้ว | ง่าย เพราะเห็นลำดับเหตุการณ์ทั้งหมด |
| ความซับซ้อนของระบบ | ต่ำ เข้าใจง่าย | สูงกว่า ต้องมี replay logic, projection, versioning |
| Query สถานะปัจจุบัน | เร็ว (`SELECT * FROM orders WHERE id = ...`) | ช้าถ้าไม่มี snapshot/projection ต้อง replay ทุกครั้ง |
| Storage | เล็กกว่า (เก็บแค่ state) | ใหญ่กว่า (เก็บทุก event ตลอดอายุ) |
| Concurrency conflict | จัดการด้วย row lock / optimistic lock บน version column เดียว | จัดการด้วย unique constraint บน (aggregate_id, version) |
| เหมาะกับ | CRUD ทั่วไป, ระบบที่ไม่ต้องการ history ละเอียด | domain ที่ audit/compliance สำคัญ, business logic ซับซ้อน, ต้องการ replay/analytics ย้อนหลัง |

### ข้อดีของ Event Sourcing

1. **Complete Audit Trail** — ทุกการเปลี่ยนแปลงถูกบันทึกโดยอัตโนมัติ เหมาะกับระบบการเงิน, e-commerce, healthcare ที่ต้องตรวจสอบย้อนหลังได้ 100%
2. **Time Travel / Temporal Query** — สามารถถามได้ว่า "order นี้มีสถานะอย่างไร ณ เวลา X" โดย replay เฉพาะ event ที่ `created_at <= X`
3. **Debugging และ Root Cause Analysis** — เมื่อเกิด bug ที่ทำให้ state ผิดพลาด สามารถไล่ดู event ทีละตัวเพื่อหาสาเหตุได้ตรงจุด
4. **Business Insight / Analytics** — event stream เป็นข้อมูลดิบสำหรับวิเคราะห์พฤติกรรม เช่น "ลูกค้ากดยกเลิก item กี่ครั้งก่อนจะ checkout จริง"
5. **Decoupling ระหว่าง Write และ Read** — เปิดทางให้ใช้ CQRS ได้อย่างเป็นธรรมชาติ (จะกล่าวถึงใน Step 917-919) สร้าง read model ใหม่ได้ตลอดเวลาโดย replay event เดิม โดยไม่กระทบ write model
6. **Extensibility** — เพิ่ม projection ใหม่ในอนาคตได้โดยไม่ต้อง migrate ข้อมูลเดิม เพียง replay event stream ที่มีอยู่แล้วสร้าง read model ใหม่

### ข้อเสียและความท้าทาย

1. **ความซับซ้อนที่เพิ่มขึ้นมาก** — ทีมต้องเข้าใจแนวคิด event, aggregate, projection, eventual consistency ซึ่งสูงกว่า CRUD ทั่วไปมาก
2. **Query สถานะปัจจุบันไม่ตรงไปตรงมา** — ต้อง replay หรือพึ่ง projection/read model เสมอ ไม่สามารถ `SELECT` ตรง ๆ จาก event store ได้อย่างมีประสิทธิภาพ
3. **Event Schema Evolution** — เมื่อ business logic เปลี่ยน โครงสร้าง event เดิมอาจไม่พอ ต้องมีกลยุทธ์ versioning ของ event schema เอง (เช่น `ItemAddedV2`) และเขียนโค้ดรองรับ event เก่าตลอดไป
4. **Eventual Consistency** — ถ้า read model อัปเดตแบบ async (เช่นผ่าน message queue) จะมีช่วงเวลาที่ read model ไม่ตรงกับ write model (จะกล่าวถึงใน Step 918-919)
5. **Snapshot Management** — aggregate ที่มี event นับพันนับหมื่นตัวจะ replay ช้า ต้องมีกลไก snapshot เพิ่มเข้ามา (Step 916) ซึ่งเพิ่ม operational overhead
6. **GDPR / Right to be Forgotten** — เมื่อข้อมูล immutable แล้ว การ "ลบข้อมูลส่วนบุคคล" ตามกฎหมายทำได้ยาก ต้องออกแบบ crypto-shredding หรือแยกข้อมูล PII ออกจาก event payload

### เมื่อไหร่ควรใช้ Event Sourcing

Event Sourcing **ไม่ใช่** pattern ที่ควรใช้กับทุกตาราง มันเหมาะกับ **aggregate ที่มี business logic ซับซ้อน มีการเปลี่ยนแปลงสถานะหลายขั้นตอน และต้องการ audit trail ที่แม่นยำ** เช่น order, การเงิน/บัญชี, inventory, workflow approval ส่วนตารางง่าย ๆ เช่น `product_categories` หรือ lookup table ทั่วไป ใช้ CRUD แบบดั้งเดิมดีกว่า เพราะ Event Sourcing มีต้นทุนความซับซ้อนที่ไม่คุ้มค่า

> **กฎทองคำ:** ใช้ Event Sourcing เฉพาะ bounded context ที่จำเป็นจริง ๆ ไม่ต้องใช้ทั้งระบบ

---

## Step 913: Event Store Design — ออกแบบตารางด้วย PostgreSQL

### ทบทวนโครงสร้างตาราง

```sql
CREATE TABLE order_events (
    event_id       BIGSERIAL PRIMARY KEY,
    aggregate_id   UUID NOT NULL,
    event_type     VARCHAR(50) NOT NULL,
    event_data     JSONB NOT NULL,
    event_version  INTEGER NOT NULL,
    created_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (aggregate_id, event_version)
);
```

มาวิเคราะห์แต่ละคอลัมน์ทีละตัวว่าทำไมต้องออกแบบแบบนี้:

| คอลัมน์ | เหตุผลในการออกแบบ |
|---|---|
| `event_id BIGSERIAL PRIMARY KEY` | เป็น **global ordering** ของ event ทั้งระบบ (ข้าม aggregate) ใช้สำหรับ event bus / projection ที่ต้องอ่าน event ทุก aggregate เรียงตามลำดับเวลาที่เกิดขึ้นจริงในระบบ (ไม่ใช่แค่ภายใน aggregate เดียว) |
| `aggregate_id UUID` | ระบุว่า event นี้เป็นของ order ใบไหน ใช้ UUID เพื่อให้ generate ID ได้แบบ distributed โดยไม่ชนกัน (เหมาะกับ microservice) |
| `event_type VARCHAR(50)` | บอกประเภทเหตุการณ์ เช่น `OrderCreated`, `ItemAdded` — ใช้เป็น "ชื่อฟังก์ชัน" ที่ replay logic จะ dispatch ไปประมวลผล |
| `event_data JSONB` | payload ของ event เป็น schema-less จึงยืดหยุ่นรองรับ event type ต่าง ๆ ที่มีโครงสร้างต่างกันได้ในตารางเดียว |
| `event_version INTEGER` | ลำดับ event **ภายใน aggregate เดียวกัน** เริ่มที่ 1 ไล่ขึ้นทีละ 1 ไม่มีการข้าม ใช้ทั้งสำหรับ replay ตามลำดับ และสำหรับ optimistic concurrency control |
| `created_at TIMESTAMPTZ` | เวลาที่ event เกิดขึ้นจริง (ใช้ `TIMESTAMPTZ` เสมอ ไม่ใช้ `TIMESTAMP` เพื่อหลีกเลี่ยงปัญหา timezone ตามที่เคยเรียนในบทต้น ๆ ของหลักสูตร) |
| `UNIQUE (aggregate_id, event_version)` | หัวใจของ optimistic concurrency control — จะอธิบายละเอียดใน Step 914 |

### เพิ่มความแข็งแรงให้ event store

ในระบบจริง เราควรเพิ่ม constraint และ metadata อีกเล็กน้อยเพื่อความสมบูรณ์:

```sql
ALTER TABLE order_events
    ADD COLUMN event_metadata JSONB NOT NULL DEFAULT '{}'::jsonb;

-- metadata เก็บข้อมูล "รอบข้าง" event เช่น ใครเป็นคนสั่ง, correlation id, causation id
-- ตัวอย่าง: {"user_id": "u-42", "correlation_id": "req-991", "source": "web-checkout"}

-- จำกัดชนิด event_type ให้อยู่ในรายการที่ยอมรับ (ป้องกัน typo)
ALTER TABLE order_events
    ADD CONSTRAINT chk_event_type CHECK (
        event_type IN (
            'OrderCreated', 'ItemAdded', 'ItemRemoved',
            'OrderShipped', 'OrderCancelled', 'OrderPaid'
        )
    );

-- ป้องกันการแก้ไข/ลบ event หลังบันทึกแล้ว (บังคับความ immutable ระดับ database)
CREATE OR REPLACE FUNCTION reject_event_mutation()
RETURNS TRIGGER AS $$
BEGIN
    RAISE EXCEPTION 'order_events เป็น append-only เท่านั้น ห้าม UPDATE/DELETE (event_id=%)',
        OLD.event_id;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_reject_update
    BEFORE UPDATE ON order_events
    FOR EACH ROW EXECUTE FUNCTION reject_event_mutation();

CREATE TRIGGER trg_reject_delete
    BEFORE DELETE ON order_events
    FOR EACH ROW EXECUTE FUNCTION reject_event_mutation();
```

ทดสอบว่า trigger ทำงานถูกต้อง:

```sql
-- พยายาม UPDATE event ที่มีอยู่แล้ว -> ต้อง error
UPDATE order_events SET event_data = '{}'::jsonb WHERE event_id = 1;
```

```
ERROR:  order_events เป็น append-only เท่านั้น ห้าม UPDATE/DELETE (event_id=1)
CONTEXT:  PL/pgSQL function reject_event_mutation() line 3 at RAISE
```

### เหตุผลที่เลือก JSONB แทน columns เฉพาะ

เราอาจสงสัยว่าทำไมไม่สร้างตารางแยกสำหรับแต่ละ event type เช่น `order_created_events`, `item_added_events` ฯลฯ เหตุผลคือ:

1. **Polymorphism ตามธรรมชาติ** — event แต่ละประเภทมี payload ต่างกันโดยสิ้นเชิง การเก็บใน column ที่ fix schema จะทำให้ตารางมี column ว่างเปล่าจำนวนมาก (sparse table)
2. **Ordering ข้าม event type** — ถ้าแยกตารางตาม type การเรียงลำดับ event ทั้งหมดของ aggregate หนึ่งตัวข้ามหลายตารางจะซับซ้อนขึ้นมาก (ต้อง `UNION ALL` แล้ว sort)
3. **Schema evolution ง่ายกว่า** — เพิ่ม field ใหม่ใน payload ไม่ต้อง `ALTER TABLE`
4. **PostgreSQL รองรับ JSONB indexing ได้ดี** — ด้วย GIN index และ operator เช่น `->`, `->>`, `@>` ทำให้ query เข้าไปใน payload ได้อย่างมีประสิทธิภาพ (ตามที่เคยเรียนเรื่อง JSONB ในบทก่อนหน้า)

> **ข้อควรระวัง:** JSONB ไม่มี foreign key หรือ type-safety ระดับ column ดังนั้น application layer (หรือ domain model) ต้องรับผิดชอบ validate โครงสร้างของ `event_data` เองก่อน insert เสมอ

---

## Step 914: Optimistic Concurrency Control ใน Event Store

### ปัญหา: Concurrent Write บน Aggregate เดียวกัน

สมมติมี 2 request เข้ามาพร้อมกันเพื่อแก้ไข order เดียวกัน (`aggregate_id` เดียวกัน) — เช่น ลูกค้ากดปุ่ม "เพิ่มสินค้า" สองครั้งเกือบพร้อมกันจากสอง tab ของ browser ทั้งสอง request จะอ่าน "version ปัจจุบัน" ของ order มาพร้อมกัน (เช่น version ล่าสุด = 3) แล้วพยายามเขียน event ใหม่เป็น version 4 ทั้งคู่ — ถ้าไม่มีการป้องกัน จะเกิด event ที่ event_version ซ้ำกัน (หรือแย่กว่านั้นคือ event หนึ่งถูก "เขียนทับ" อีกอันโดยไม่รู้ตัว)

นี่คือปัญหา classic ของ concurrent write ที่ event sourcing ต้องแก้ให้ได้

### วิธีแก้: UNIQUE constraint บน (aggregate_id, event_version)

จำ constraint ที่เราสร้างไว้ตั้งแต่ต้น:

```sql
UNIQUE (aggregate_id, event_version)
```

Constraint นี้คือกลไก **Optimistic Concurrency Control (OCC)** ที่สมบูรณ์แบบสำหรับ event store โดยไม่ต้องใช้ lock ใด ๆ เพิ่มเติม หลักการทำงาน:

1. Application อ่าน `event_version` ล่าสุดของ aggregate (สมมติ = 3)
2. Application คำนวณ business logic แล้วพยายาม `INSERT` event ใหม่ด้วย `event_version = 4`
3. ถ้ามี process อื่นแทรกเข้ามาก่อนแล้ว insert `event_version = 4` ไปแล้ว — การ `INSERT` ของเราจะ **ชน UNIQUE constraint และ error ทันที**
4. Application จับ error นี้ แล้วทำ retry: อ่าน state ล่าสุดใหม่, คำนวณใหม่, insert ด้วย version ถัดไปที่ถูกต้อง

นี่คือ "optimistic" เพราะเรา "สันนิษฐาน" ว่าจะไม่มี conflict (ไม่ lock ล่วงหน้าเหมือน pessimistic locking) แต่ถ้าชนจริง database จะบอกเราทันทีผ่าน constraint violation

### ตัวอย่างการ implement ด้วยฟังก์ชัน PL/pgSQL

```sql
CREATE OR REPLACE FUNCTION append_order_event(
    p_aggregate_id UUID,
    p_event_type   VARCHAR(50),
    p_event_data   JSONB,
    p_expected_version INTEGER  -- version ที่ client คิดว่าเป็นปัจจุบัน (ก่อนเพิ่ม event นี้)
) RETURNS BIGINT AS $$
DECLARE
    v_new_event_id BIGINT;
BEGIN
    -- พยายาม insert ด้วย version ถัดไปจาก expected_version
    INSERT INTO order_events (aggregate_id, event_type, event_data, event_version)
    VALUES (p_aggregate_id, p_event_type, p_event_data, p_expected_version + 1)
    RETURNING event_id INTO v_new_event_id;

    RETURN v_new_event_id;

EXCEPTION
    WHEN unique_violation THEN
        RAISE EXCEPTION
            'Concurrency conflict: aggregate % ถูกแก้ไขโดย process อื่นไปแล้ว (expected_version=%)',
            p_aggregate_id, p_expected_version
            USING ERRCODE = '40001';  -- ใช้ SQLSTATE ของ serialization_failure เพื่อให้ retry logic จับได้ง่าย
END;
$$ LANGUAGE plpgsql;
```

ทดสอบการทำงานปกติ:

```sql
-- อ่าน version ล่าสุดก่อน
SELECT COALESCE(MAX(event_version), 0) AS current_version
FROM order_events
WHERE aggregate_id = 'a1b2c3d4-0001-4000-8000-000000000001';
```

```
 current_version
------------------
                3
(1 row)
```

```sql
-- เพิ่ม event ใหม่โดยระบุ expected_version = 3 (จะกลายเป็น version 4)
SELECT append_order_event(
    'a1b2c3d4-0001-4000-8000-000000000001',
    'OrderShipped',
    jsonb_build_object('carrier', 'Kerry Express', 'tracking_no', 'KEX1234567890'),
    3
);
```

```
 append_order_event
---------------------
                   4
(1 row)
```

ทดสอบกรณี conflict (จำลองว่ามี process อื่น insert version 4 ไปแล้วก่อนหน้านี้):

```sql
-- process A: เข้าใจว่า current version = 3 (แต่จริง ๆ มีคนอื่น insert version 4 ไปแล้ว)
SELECT append_order_event(
    'a1b2c3d4-0001-4000-8000-000000000001',
    'OrderCancelled',
    jsonb_build_object('reason', 'ลูกค้าเปลี่ยนใจ'),
    3   -- expected_version ผิดพลาด เพราะจริง ๆ ปัจจุบันคือ 4 แล้ว
);
```

```
ERROR:  Concurrency conflict: aggregate a1b2c3d4-0001-4000-8000-000000000001 ถูกแก้ไขโดย process อื่นไปแล้ว (expected_version=3)
CONTEXT:  PL/pgSQL function append_order_event(uuid,character varying,jsonb,integer) line 13 at RAISE
```

Application เมื่อเจอ error นี้จะ:
1. Rollback transaction ปัจจุบัน
2. อ่าน state/version ล่าสุดใหม่
3. Re-apply business logic (เช่น ตรวจสอบเงื่อนไขใหม่อีกครั้งว่ายัง cancel ได้อยู่ไหม เพราะตอนนี้ order ถูก ship ไปแล้ว!)
4. ลอง insert ใหม่อีกครั้งด้วย version ที่ถูกต้อง

### ทำไมวิธีนี้ดีกว่า pessimistic locking

ถ้าใช้ `SELECT ... FOR UPDATE` แบบ pessimistic locking เราจะต้อง lock แถวไว้ตลอดช่วงเวลาที่ทำ business logic (ซึ่งอาจเรียก external service, validate กติกาธุรกิจซับซ้อน) ทำให้ throughput ต่ำลงมากในระบบที่มี concurrent load สูง ในขณะที่ optimistic concurrency ด้วย UNIQUE constraint:

- ไม่ lock อะไรล่วงหน้า อนุญาตให้อ่าน/คำนวณคู่ขนานได้เต็มที่
- Conflict เกิดได้เฉพาะตอน `INSERT` จริง ๆ ซึ่งเป็น operation ที่เร็วมาก
- เหมาะกับ event store ที่ธรรมชาติของการเขียนเป็น "append" อยู่แล้ว ไม่ใช่ "update in place"

> **เชื่อมโยงความรู้เดิม:** concept นี้คล้ายกับ optimistic locking ด้วย `version` column ที่เคยเรียนใน pattern ทั่วไป (`UPDATE ... WHERE version = ?`) แต่ในโลก event sourcing เราไม่ update เลย เราใช้ unique constraint บน insert แทน ซึ่งสอดคล้องกับหลักการ append-only

---

## Step 915: การ Replay Event เพื่อสร้าง Current State

### แนวคิด Replay

**Replay** คือกระบวนการอ่าน event ทั้งหมดของ aggregate หนึ่งตัว **เรียงตาม `event_version` จากน้อยไปมาก** แล้ว "เล่นซ้ำ" ทีละตัวเพื่อคำนวณ state ปัจจุบัน คล้ายกับการดู video ที่ตัดต่อจาก clip เหตุการณ์ทั้งหมดเรียงตามเวลา

### Query พื้นฐานสำหรับ Replay

```sql
SELECT event_type, event_data, event_version, created_at
FROM order_events
WHERE aggregate_id = 'a1b2c3d4-0001-4000-8000-000000000001'
ORDER BY event_version ASC;
```

```
  event_type   | event_version |                       event_data
----------------+---------------+---------------------------------------------------------
 OrderCreated   |             1 | {"currency": "THB", "customer_id": "cust-1001"}
 ItemAdded      |             2 | {"sku": "SKU-A100", "quantity": 2, "unit_price": 350.00}
 ItemAdded      |             3 | {"sku": "SKU-B200", "quantity": 1, "unit_price": 1290.00}
 OrderShipped   |             4 | {"carrier": "Kerry Express", "tracking_no": "KEX123..."}
(4 rows)
```

### Replay Logic ด้วย PL/pgSQL Function

เราสามารถเขียนฟังก์ชันที่ "fold" (พับรวม) event ทั้งหมดให้เป็น state เดียว โดยใช้หลักการเดียวกับ `reduce`/`fold` ในภาษาโปรแกรมทั่วไป:

```sql
CREATE OR REPLACE FUNCTION replay_order_state(p_aggregate_id UUID)
RETURNS JSONB AS $$
DECLARE
    v_event RECORD;
    v_state JSONB := jsonb_build_object(
        'status', 'draft',
        'items', '[]'::jsonb,
        'version', 0
    );
    v_items JSONB;
    v_new_item JSONB;
    v_idx INTEGER;
BEGIN
    FOR v_event IN
        SELECT event_type, event_data, event_version
        FROM order_events
        WHERE aggregate_id = p_aggregate_id
        ORDER BY event_version ASC
    LOOP
        CASE v_event.event_type
            WHEN 'OrderCreated' THEN
                v_state := v_state
                    || jsonb_build_object('status', 'created')
                    || jsonb_build_object('customer_id', v_event.event_data->>'customer_id')
                    || jsonb_build_object('currency', v_event.event_data->>'currency');

            WHEN 'ItemAdded' THEN
                v_items := v_state->'items';
                v_new_item := jsonb_build_object(
                    'sku', v_event.event_data->>'sku',
                    'quantity', (v_event.event_data->>'quantity')::int,
                    'unit_price', (v_event.event_data->>'unit_price')::numeric
                );
                v_state := jsonb_set(v_state, '{items}', v_items || jsonb_build_array(v_new_item));

            WHEN 'ItemRemoved' THEN
                -- ลบ item ที่ sku ตรงกันออกจาก array (แบบง่าย: filter ออก)
                SELECT jsonb_agg(elem) INTO v_items
                FROM jsonb_array_elements(v_state->'items') AS elem
                WHERE elem->>'sku' != (v_event.event_data->>'sku');
                v_state := jsonb_set(v_state, '{items}', COALESCE(v_items, '[]'::jsonb));

            WHEN 'OrderShipped' THEN
                v_state := v_state
                    || jsonb_build_object('status', 'shipped')
                    || jsonb_build_object('tracking_no', v_event.event_data->>'tracking_no');

            WHEN 'OrderCancelled' THEN
                v_state := v_state
                    || jsonb_build_object('status', 'cancelled')
                    || jsonb_build_object('cancel_reason', v_event.event_data->>'reason');

            ELSE
                RAISE WARNING 'ไม่รู้จัก event_type: %', v_event.event_type;
        END CASE;

        v_state := jsonb_set(v_state, '{version}', to_jsonb(v_event.event_version));
    END LOOP;

    RETURN v_state;
END;
$$ LANGUAGE plpgsql;
```

เรียกใช้งาน:

```sql
SELECT replay_order_state('a1b2c3d4-0001-4000-8000-000000000001');
```

```
                                                          replay_order_state
---------------------------------------------------------------------------------------------------------------------------------------
 {"items": [{"sku": "SKU-A100", "quantity": 2, "unit_price": 350.00}, {"sku": "SKU-B200", "quantity": 1, "unit_price": 1290.00}],
  "status": "shipped", "version": 4, "currency": "THB", "customer_id": "cust-1001", "tracking_no": "KEX1234567890"}
(1 row)
```

state ที่ได้ตรงกับสิ่งที่เกิดขึ้นจริงตามลำดับ event ทุกประการ

### Replay เพื่อดู State ณ เวลาใดเวลาหนึ่ง (Time Travel)

นี่คือประโยชน์ที่เด่นชัดที่สุดของ Event Sourcing — สามารถ replay เฉพาะ event ที่เกิดขึ้น **ก่อน** เวลาที่สนใจได้:

```sql
CREATE OR REPLACE FUNCTION replay_order_state_at(
    p_aggregate_id UUID,
    p_as_of TIMESTAMPTZ
) RETURNS JSONB AS $$
DECLARE
    v_max_version INTEGER;
BEGIN
    -- หา version สูงสุดที่เกิดขึ้นก่อนหรือเท่ากับเวลาที่สนใจ
    SELECT MAX(event_version) INTO v_max_version
    FROM order_events
    WHERE aggregate_id = p_aggregate_id
      AND created_at <= p_as_of;

    IF v_max_version IS NULL THEN
        RETURN NULL;  -- ยังไม่มี event ใด ๆ ณ เวลานั้น
    END IF;

    -- ใช้ logic เดียวกับ replay_order_state แต่กรอง event_version <= v_max_version
    -- (ในระบบจริงมักแยกฟังก์ชัน replay ให้รับ p_up_to_version เป็นพารามิเตอร์)
    RETURN replay_order_state_up_to(p_aggregate_id, v_max_version);
END;
$$ LANGUAGE plpgsql;
```

> ในตัวอย่างข้างต้นสมมติว่ามีฟังก์ชัน `replay_order_state_up_to(aggregate_id, max_version)` ที่ทำงานเหมือน `replay_order_state` แต่เพิ่มเงื่อนไข `AND event_version <= p_up_to_version` ในการ query — ผู้เรียนสามารถปรับ `replay_order_state` เดิมให้รับพารามิเตอร์เพิ่มได้ไม่ยาก (ดูแบบฝึกหัดท้ายบท)

### ปัญหาของการ Replay ทุกครั้ง

ลอง query จำนวน event ต่อ aggregate ในระบบที่ใช้งานมานาน:

```sql
SELECT aggregate_id, COUNT(*) AS event_count
FROM order_events
GROUP BY aggregate_id
ORDER BY event_count DESC
LIMIT 5;
```

```
              aggregate_id              | event_count
------------------------------------------+-------------
 f7e6d5c4-9999-4000-8000-000000009999   |       15420
 a1b2c3d4-0001-4000-8000-000000000001   |           4
 ...
```

ถ้า aggregate หนึ่งมี event นับหมื่นตัว (เช่น order ที่มีการแก้ไขบ่อยมาก หรือ aggregate ประเภทอื่นที่ event เกิดถี่ เช่น shopping cart ที่ update ตลอดเวลา) การ replay ทุกครั้งที่ต้องการอ่าน state จะ**ช้ามาก** และสิ้นเปลือง CPU/I/O โดยไม่จำเป็น — นี่คือเหตุผลที่เราต้องมี **Snapshot Pattern** ซึ่งจะเรียนใน Step ถัดไป

---

## Step 916: Snapshot Pattern

### แนวคิด

**Snapshot** คือการ "ถ่ายภาพ" state ของ aggregate ณ event_version หนึ่ง ๆ แล้วเก็บไว้ในตารางแยก เมื่อจะอ่าน state ปัจจุบัน แทนที่จะ replay ตั้งแต่ event_version = 1 เราจะ:

1. อ่าน snapshot ล่าสุดที่มี (เช่น snapshot ที่ version 1000)
2. Replay เฉพาะ event ที่มี `event_version > 1000` (event ใหม่หลัง snapshot เท่านั้น)
3. รวม snapshot + event ที่เหลือ = state ปัจจุบัน

วิธีนี้ลดจำนวน event ที่ต้อง replay จากหลักหมื่นเหลือแค่หลักสิบ/หลักร้อยเท่านั้น

### ออกแบบตาราง Snapshot

```sql
CREATE TABLE order_snapshots (
    aggregate_id     UUID NOT NULL,
    snapshot_version INTEGER NOT NULL,   -- event_version ล่าสุดที่รวมอยู่ใน snapshot นี้
    state_data       JSONB NOT NULL,
    created_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (aggregate_id, snapshot_version)
);

-- มักจะ query แค่ snapshot ล่าสุด จึงทำ index ช่วยหา MAX(snapshot_version) ต่อ aggregate ได้เร็ว
CREATE INDEX idx_order_snapshots_latest
    ON order_snapshots (aggregate_id, snapshot_version DESC);
```

### ฟังก์ชันสร้าง Snapshot

```sql
CREATE OR REPLACE FUNCTION create_order_snapshot(p_aggregate_id UUID)
RETURNS VOID AS $$
DECLARE
    v_state JSONB;
    v_version INTEGER;
BEGIN
    v_state := replay_order_state(p_aggregate_id);
    v_version := (v_state->>'version')::INTEGER;

    INSERT INTO order_snapshots (aggregate_id, snapshot_version, state_data)
    VALUES (p_aggregate_id, v_version, v_state)
    ON CONFLICT (aggregate_id, snapshot_version) DO NOTHING;
END;
$$ LANGUAGE plpgsql;
```

### ฟังก์ชัน Replay แบบใช้ Snapshot ช่วยเร่งความเร็ว

```sql
CREATE OR REPLACE FUNCTION replay_order_state_fast(p_aggregate_id UUID)
RETURNS JSONB AS $$
DECLARE
    v_snapshot RECORD;
    v_state JSONB;
    v_event RECORD;
BEGIN
    -- 1) หา snapshot ล่าสุด (ถ้ามี)
    SELECT snapshot_version, state_data
    INTO v_snapshot
    FROM order_snapshots
    WHERE aggregate_id = p_aggregate_id
    ORDER BY snapshot_version DESC
    LIMIT 1;

    IF FOUND THEN
        v_state := v_snapshot.state_data;
    ELSE
        v_state := jsonb_build_object('status', 'draft', 'items', '[]'::jsonb, 'version', 0);
        v_snapshot.snapshot_version := 0;
    END IF;

    -- 2) replay เฉพาะ event ที่เกิดหลัง snapshot
    FOR v_event IN
        SELECT event_type, event_data, event_version
        FROM order_events
        WHERE aggregate_id = p_aggregate_id
          AND event_version > v_snapshot.snapshot_version
        ORDER BY event_version ASC
    LOOP
        -- (ใช้ logic การ apply event เดียวกับ replay_order_state)
        CASE v_event.event_type
            WHEN 'OrderShipped' THEN
                v_state := v_state || jsonb_build_object('status', 'shipped');
            WHEN 'OrderCancelled' THEN
                v_state := v_state || jsonb_build_object('status', 'cancelled');
            -- ... (เคสอื่น ๆ เหมือนเดิม ตัดให้สั้นเพื่อความกระชับในตัวอย่าง)
            ELSE
                NULL;
        END CASE;
        v_state := jsonb_set(v_state, '{version}', to_jsonb(v_event.event_version));
    END LOOP;

    RETURN v_state;
END;
$$ LANGUAGE plpgsql;
```

### กลยุทธ์การสร้าง Snapshot

คำถามสำคัญ: ควรสร้าง snapshot บ่อยแค่ไหน? มีหลายแนวทาง:

1. **ทุก N event** — เช่น สร้าง snapshot ใหม่ทุก ๆ 100 event ที่เพิ่มเข้ามา
2. **ตาม schedule** — เช่น cron job รันทุกคืนเพื่อสร้าง snapshot ให้ aggregate ที่มี event เยอะ
3. **Lazy (on-demand)** — สร้าง snapshot เฉพาะตอนที่ตรวจพบว่า replay ใช้เวลานานเกิน threshold

ตัวอย่าง trigger ที่สร้าง snapshot อัตโนมัติทุก 100 event:

```sql
CREATE OR REPLACE FUNCTION maybe_create_snapshot()
RETURNS TRIGGER AS $$
BEGIN
    IF NEW.event_version % 100 = 0 THEN
        PERFORM create_order_snapshot(NEW.aggregate_id);
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_auto_snapshot
    AFTER INSERT ON order_events
    FOR EACH ROW EXECUTE FUNCTION maybe_create_snapshot();
```

> **ข้อควรระวัง:** การสร้าง snapshot ภายใน trigger แบบ synchronous (ในทุก transaction ที่ event_version หาร 100 ลงตัว) จะเพิ่ม latency ให้กับ transaction นั้น ๆ ในระบบที่ throughput สูงมาก ควรพิจารณาทำแบบ asynchronous ผ่าน background job หรือ message queue แทน เพื่อไม่ให้กระทบ write path หลัก

### Snapshot ไม่ใช่ Source of Truth

ข้อควรจำสำคัญ: **snapshot เป็นเพียง cache/optimization** ไม่ใช่ source of truth — event store (`order_events`) ยังคงเป็นข้อมูลต้นทางเสมอ ถ้า snapshot เสียหายหรือ logic การสร้าง snapshot มี bug เราสามารถลบ snapshot ทิ้งทั้งหมดแล้ว replay จาก event ใหม่ได้เสมอโดยไม่สูญเสียข้อมูลใด ๆ

```sql
-- ลบ snapshot ทั้งหมดได้อย่างปลอดภัย เพราะ event store ยังอยู่ครบ
TRUNCATE TABLE order_snapshots;
-- ระบบยังทำงานถูกต้อง เพียงแต่ replay ช้าลงจนกว่า snapshot ใหม่จะถูกสร้าง
```

---

## Step 917: CQRS คืออะไร

### แนวคิด Command Query Responsibility Segregation

**CQRS (Command Query Responsibility Segregation)** เป็น pattern ทางสถาปัตยกรรมที่เสนอโดย Greg Young โดยแนวคิดหลักคือ: **แยก model สำหรับ "เขียน" (Command) ออกจาก model สำหรับ "อ่าน" (Query) อย่างเด็ดขาด**

ในระบบ CRUD แบบดั้งเดิม เราใช้ตารางเดียวกันทั้งเขียนและอ่าน:

```
Application --> [เดียวกัน] --> Table (INSERT/UPDATE/DELETE + SELECT)
```

ใน CQRS เราแยกเป็นสองเส้นทาง:

```
Command (เขียน) --> Write Model (ออกแบบเพื่อความถูกต้องของ business logic)
Query   (อ่าน)  --> Read Model  (ออกแบบเพื่อความเร็วในการแสดงผล)
```

### ทำไมต้องแยก?

Write model และ Read model มี **requirement ที่ขัดแย้งกัน** โดยธรรมชาติ:

| ด้าน | Write Model ต้องการ | Read Model ต้องการ |
|---|---|---|
| Normalization | Normalized สูง เพื่อป้องกัน anomaly | Denormalized เพื่อ query เร็ว ไม่ต้อง JOIN เยอะ |
| Structure | สะท้อน business rule/invariant | สะท้อนสิ่งที่ UI ต้องแสดงผลพอดี |
| Consistency | Strong consistency (ACID) | Eventual consistency ยอมรับได้ |
| Optimize for | ความถูกต้อง, การตรวจสอบกติกาธุรกิจ | ความเร็วในการอ่าน, จำนวน query ที่รองรับได้ |
| จำนวน model | มักมี 1 model ต่อ aggregate | อาจมีหลาย read model สำหรับ use case ต่างกัน (list view, dashboard, report) |

ตัวอย่างที่ชัดเจน: หน้า "order summary" ของลูกค้าต้องการแสดง ชื่อลูกค้า, จำนวนสินค้ารวม, ยอดรวม, สถานะ — ทั้งหมดใน query เดียว ถ้าใช้ write model (event store) ต้อง replay event ทุกครั้งซึ่งช้ามาก แต่ถ้ามี **read model** ที่เป็นตาราง denormalized พร้อมข้อมูลเหล่านี้อยู่แล้ว การ query จะเร็วมาก (`SELECT * FROM order_summary_view WHERE order_id = ?`)

### CQRS ไม่จำเป็นต้องมาคู่กับ Event Sourcing

ข้อควรเข้าใจสำคัญ: **CQRS และ Event Sourcing เป็นคนละ pattern กัน** สามารถใช้ CQRS ได้โดยไม่ต้องมี Event Sourcing เช่น:

```
Write Model: ตาราง orders แบบ normalized ปกติ (state-based)
Read Model:  materialized view ที่ denormalize จาก orders + order_items + customers
```

แบบนี้ก็เป็น CQRS แล้ว (แยก write/read model) แต่ยังไม่ใช่ Event Sourcing (เพราะ write model ยังเก็บแค่ state ปัจจุบัน ไม่เก็บ event) — เราเคยเรียนแนวคิดคล้ายกันนี้ไปแล้วใน **Part 036 เรื่อง Materialized View** ซึ่งเป็นการสร้าง read model แบบ denormalized จาก normalized table ปกติ

แต่ **Event Sourcing กับ CQRS เข้ากันได้ดีมากเป็นพิเศษ** เพราะ:
- Write model (event store) เขียนแบบ append-only ล้วน ๆ ไม่ต้องกังวลเรื่อง UPDATE ซับซ้อน
- Read model สร้างจาก event stream ได้โดยตรง ผ่านกระบวนการที่เรียกว่า **projection**
- ถ้า read model เสียหายหรือต้องการ schema ใหม่ สามารถ "rebuild" ได้โดย replay event stream ทั้งหมดใหม่ โดยไม่กระทบ write model เลย

นี่คือเหตุผลที่บทนี้สอนทั้งสอง pattern ควบคู่กัน

### ตัวอย่างสถาปัตยกรรม CQRS + Event Sourcing แบบง่าย

```
                    ┌─────────────────┐
   Command  ───────▶│   order_events   │  (Write Model / Event Store)
                    │  (append-only)   │
                    └────────┬─────────┘
                             │ event stream
                             ▼
                    ┌──────────────────┐
                    │   Projection      │ (trigger หรือ event handler)
                    │   (subscriber)    │
                    └────────┬──────────┘
                             │ อัปเดต
                             ▼
                    ┌──────────────────┐
   Query    ◀───────│ order_summary    │  (Read Model)
                    │ (denormalized)   │
                    └──────────────────┘
```

---

## Step 918: CQRS ร่วมกับ Event Sourcing — Write Model และ Read Model

### Write Model: Event Store (ที่เราสร้างมาตั้งแต่ต้นบท)

ตาราง `order_events` **คือ** write model ของเรา ทุก command (เช่น "สร้าง order ใหม่", "เพิ่มสินค้า", "จัดส่ง") จะถูกแปลงเป็น event แล้ว append เข้าตารางนี้ พร้อมตรวจสอบ business invariant ก่อนเขียนเสมอ (เช่น ห้าม ship order ที่ถูก cancel ไปแล้ว)

```sql
-- ตัวอย่าง command handler ระดับแนวคิด (pseudo-code อธิบายด้วย SQL)
-- Command: ShipOrder
-- 1. Replay state ปัจจุบันเพื่อตรวจสอบ invariant
DO $$
DECLARE
    v_state JSONB;
BEGIN
    v_state := replay_order_state('a1b2c3d4-0001-4000-8000-000000000001');

    IF v_state->>'status' = 'cancelled' THEN
        RAISE EXCEPTION 'ไม่สามารถจัดส่ง order ที่ถูกยกเลิกไปแล้วได้';
    END IF;

    IF v_state->>'status' != 'created' THEN
        RAISE EXCEPTION 'ไม่สามารถจัดส่ง order ที่ยังไม่อยู่ในสถานะ created ได้ (สถานะปัจจุบัน: %)',
            v_state->>'status';
    END IF;

    -- 2. ผ่าน invariant check แล้ว -> append event
    PERFORM append_order_event(
        'a1b2c3d4-0001-4000-8000-000000000001',
        'OrderShipped',
        jsonb_build_object('carrier', 'Flash Express', 'tracking_no', 'FE99988877'),
        (v_state->>'version')::int
    );
END $$;
```

### Read Model: ตาราง Denormalized สำหรับแสดงผล

Read model ออกแบบตามสิ่งที่ **UI/API ต้องการแสดงผล** ไม่ใช่ตามกติกาธุรกิจ ตัวอย่างสำหรับหน้า "รายการ order ของลูกค้า":

```sql
CREATE TABLE order_summary_read_model (
    aggregate_id    UUID PRIMARY KEY,
    customer_id     VARCHAR(50) NOT NULL,
    status          VARCHAR(20) NOT NULL,
    item_count      INTEGER NOT NULL DEFAULT 0,
    total_amount    NUMERIC(12,2) NOT NULL DEFAULT 0,
    currency        VARCHAR(3) NOT NULL DEFAULT 'THB',
    tracking_no     VARCHAR(50),
    last_event_version INTEGER NOT NULL DEFAULT 0,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_summary_customer ON order_summary_read_model (customer_id);
CREATE INDEX idx_summary_status ON order_summary_read_model (status);
```

Query จากตารางนี้เร็วมาก เพราะไม่ต้อง JOIN หรือ replay อะไรเลย:

```sql
-- แสดงรายการ order ทั้งหมดของลูกค้า cust-1001 ที่ยังไม่ถูกยกเลิก
SELECT aggregate_id, status, item_count, total_amount, currency, updated_at
FROM order_summary_read_model
WHERE customer_id = 'cust-1001'
  AND status != 'cancelled'
ORDER BY updated_at DESC;
```

```
              aggregate_id              | status  | item_count | total_amount | currency |          updated_at
-----------------------------------------+---------+------------+---------------+----------+-------------------------------
 a1b2c3d4-0001-4000-8000-000000000001   | shipped |          2 |       1990.00 | THB      | 2026-09-25 09:15:00+07
(1 row)
```

### Eventual Consistency: สิ่งที่ต้องยอมรับ

เมื่อแยก write model กับ read model ออกจากกัน จะมีช่วงเวลาสั้น ๆ ที่ **read model ยังไม่ทันอัปเดตตาม write model ล่าสุด** เรียกว่า **eventual consistency** — read model จะ "ตามทัน" ในที่สุด (eventually) แต่ไม่ใช่ทันทีเสมอไป

ผลกระทบต่อการออกแบบ UX: หลังจากลูกค้ากด "สั่งซื้อ" แล้ว refresh หน้าทันที มีโอกาสน้อยมากที่ order ใหม่จะยังไม่ปรากฏใน read model (ถ้า projection ทำงานแบบ async ผ่าน queue) ระบบต้องออกแบบรับมือ เช่น แสดง optimistic UI, หรือใช้ projection แบบ synchronous (trigger ภายใน transaction เดียวกัน) สำหรับ use case ที่ consistency สำคัญมาก (จะอธิบายทั้งสองวิธีใน Step ถัดไป)

---

## Step 919: การสร้าง Read Model (Projection) จาก Event

Projection คือกระบวนการ "แปลง" event stream ให้กลายเป็น read model มีสองวิธีหลักในการ implement: **Trigger-based (synchronous)** และ **Application-level (asynchronous)**

### วิธีที่ 1: Trigger-based Projection (Synchronous)

วิธีนี้ใช้ PostgreSQL trigger ที่ทำงานทันทีเมื่อมี event ใหม่ถูก insert เข้า `order_events` — ข้อดีคือ **strong consistency** (read model อัปเดตพร้อมกับ write model ใน transaction เดียวกันเป๊ะ) ข้อเสียคือเพิ่ม latency ให้ write path และ coupling ระหว่าง write/read model แน่นขึ้น

```sql
CREATE OR REPLACE FUNCTION project_order_summary()
RETURNS TRIGGER AS $$
DECLARE
    v_item JSONB;
BEGIN
    -- ใช้ UPSERT (INSERT ... ON CONFLICT) เพื่อสร้างหรืออัปเดต read model
    CASE NEW.event_type
        WHEN 'OrderCreated' THEN
            INSERT INTO order_summary_read_model (
                aggregate_id, customer_id, status, currency, last_event_version
            ) VALUES (
                NEW.aggregate_id,
                NEW.event_data->>'customer_id',
                'created',
                COALESCE(NEW.event_data->>'currency', 'THB'),
                NEW.event_version
            )
            ON CONFLICT (aggregate_id) DO UPDATE
                SET status = 'created',
                    last_event_version = NEW.event_version,
                    updated_at = now();

        WHEN 'ItemAdded' THEN
            UPDATE order_summary_read_model
            SET item_count = item_count + (NEW.event_data->>'quantity')::int,
                total_amount = total_amount +
                    (NEW.event_data->>'quantity')::numeric * (NEW.event_data->>'unit_price')::numeric,
                last_event_version = NEW.event_version,
                updated_at = now()
            WHERE aggregate_id = NEW.aggregate_id;

        WHEN 'ItemRemoved' THEN
            UPDATE order_summary_read_model
            SET item_count = item_count - (NEW.event_data->>'quantity')::int,
                total_amount = total_amount -
                    (NEW.event_data->>'quantity')::numeric * (NEW.event_data->>'unit_price')::numeric,
                last_event_version = NEW.event_version,
                updated_at = now()
            WHERE aggregate_id = NEW.aggregate_id;

        WHEN 'OrderShipped' THEN
            UPDATE order_summary_read_model
            SET status = 'shipped',
                tracking_no = NEW.event_data->>'tracking_no',
                last_event_version = NEW.event_version,
                updated_at = now()
            WHERE aggregate_id = NEW.aggregate_id;

        WHEN 'OrderCancelled' THEN
            UPDATE order_summary_read_model
            SET status = 'cancelled',
                last_event_version = NEW.event_version,
                updated_at = now()
            WHERE aggregate_id = NEW.aggregate_id;

        ELSE
            -- event type อื่น ๆ ที่ไม่กระทบ read model นี้ ไม่ต้องทำอะไร
            NULL;
    END CASE;

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- ต้องเพิ่ม UNIQUE constraint ก่อน เพื่อให้ ON CONFLICT ทำงานได้
ALTER TABLE order_summary_read_model
    ADD CONSTRAINT uq_summary_aggregate UNIQUE (aggregate_id);

CREATE TRIGGER trg_project_order_summary
    AFTER INSERT ON order_events
    FOR EACH ROW EXECUTE FUNCTION project_order_summary();
```

ทดสอบ: สร้าง order ใหม่แล้วดู read model อัปเดตทันที

```sql
SELECT append_order_event(
    gen_random_uuid(),  -- สมมติ UUID ใหม่ เก็บไว้ตรวจสอบ
    'OrderCreated',
    jsonb_build_object('customer_id', 'cust-2002', 'currency', 'THB'),
    0
);
```

จากนั้น query read model ทันทีในธุรกรรมเดียวกัน (เพราะ trigger เป็น synchronous):

```sql
SELECT aggregate_id, customer_id, status, item_count, total_amount
FROM order_summary_read_model
WHERE customer_id = 'cust-2002';
```

```
              aggregate_id              | customer_id | status  | item_count | total_amount
-----------------------------------------+-------------+---------+------------+---------------
 c3f8a921-....                          | cust-2002   | created |          0 |          0.00
(1 row)
```

### วิธีที่ 2: Application-level Projection (Asynchronous)

ในระบบขนาดใหญ่ที่ต้องการ decouple write path ออกจาก read model โดยสิ้นเชิง (เช่น มี read model หลายตัวสำหรับ use case ต่างกัน, หรือ read model อยู่คนละฐานข้อมูล/ระบบ) มักใช้แนวทางนี้แทน:

```
1. Application เขียน event เข้า order_events (write model)
2. Event handler (background worker / message queue consumer) subscribe การเปลี่ยนแปลง
   - อาจ poll ตาราง order_events ด้วย event_id ที่ยังไม่ประมวลผล
   - หรือรับ event ผ่าน message queue (Kafka, RabbitMQ, NOTIFY/LISTEN ของ PostgreSQL เอง)
3. Event handler ประมวลผล event แล้วอัปเดต read model แบบ async
```

ตัวอย่างการ implement ด้วย PostgreSQL `LISTEN`/`NOTIFY` (ฟีเจอร์ native ของ PostgreSQL ที่เหมาะกับ use case นี้):

```sql
-- Trigger ที่ "ประกาศ" ว่ามี event ใหม่ แทนที่จะอัปเดต read model ตรง ๆ
CREATE OR REPLACE FUNCTION notify_new_order_event()
RETURNS TRIGGER AS $$
BEGIN
    PERFORM pg_notify(
        'order_events_channel',
        jsonb_build_object(
            'event_id', NEW.event_id,
            'aggregate_id', NEW.aggregate_id,
            'event_type', NEW.event_type
        )::text
    );
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_notify_order_event
    AFTER INSERT ON order_events
    FOR EACH ROW EXECUTE FUNCTION notify_new_order_event();
```

ฝั่ง application (pseudo-code เชิงแนวคิด แสดงด้วยคำอธิบาย เพราะเป็นโค้ดฝั่ง client):

```
-- worker process เชื่อมต่อ PostgreSQL แล้วรอฟัง channel นี้
LISTEN order_events_channel;

-- เมื่อได้รับ notification:
--   1. อ่าน event เต็ม ๆ จาก order_events ด้วย event_id ที่ได้รับ
--   2. เรียก logic เดียวกับ project_order_summary() แต่รันฝั่ง application
--   3. เขียนผลลัพธ์ลง read model (อาจเป็นคนละฐานข้อมูลก็ได้ เช่น Elasticsearch, Redis)
```

ข้อดีของวิธี async:
- Write path เร็วที่สุดเท่าที่จะทำได้ ไม่ต้องรอ read model อัปเดต
- Read model สามารถอยู่คนละ technology stack ได้เลย (เช่น Elasticsearch สำหรับ full-text search, Redis สำหรับ cache)
- ขยาย (scale) read model และ write model แยกจากกันได้อิสระ

ข้อเสีย:
- ต้องยอมรับ eventual consistency
- ต้องมีกลไกจัดการ event ที่ประมวลผลไม่สำเร็จ (retry, dead-letter queue)
- ซับซ้อนกว่าในการ operate (ต้องมี worker process แยก, monitoring เพิ่ม)

### Rebuild Read Model จาก Event Stream

ประโยชน์สำคัญที่สุดอย่างหนึ่งของสถาปัตยกรรมนี้คือ **read model สร้างใหม่ได้เสมอ** โดยไม่สูญเสียข้อมูล เพราะ source of truth คือ event store:

```sql
-- ลบ read model เดิมทิ้งทั้งหมด
TRUNCATE TABLE order_summary_read_model;

-- Rebuild โดย replay event ทั้งหมดในระบบเรียงตาม global order (event_id)
-- ผ่าน logic เดียวกับ trigger แต่รันแบบ batch
DO $$
DECLARE
    v_event RECORD;
BEGIN
    FOR v_event IN
        SELECT * FROM order_events ORDER BY event_id ASC
    LOOP
        -- เรียก logic เดียวกับใน project_order_summary()
        -- (ในระบบจริงมักแยก logic ออกเป็นฟังก์ชันกลางที่ trigger และ rebuild script เรียกร่วมกัน)
        PERFORM apply_event_to_summary(v_event);
    END LOOP;
END $$;
```

ความสามารถนี้มีประโยชน์มากเมื่อ:
- Read model schema เปลี่ยน (เพิ่ม column ใหม่ที่ต้องคำนวณจาก event เก่า)
- Read model เสียหายจาก bug ใน projection logic
- ต้องการสร้าง read model ใหม่สำหรับ use case ใหม่ที่ยังไม่เคยมี

> **เชื่อมโยงความรู้เดิม:** แนวคิด "rebuild read model" นี้คล้ายกับการ `REFRESH MATERIALIZED VIEW` ที่เคยเรียนใน Part 036 แต่ในที่นี้เราควบคุม logic การ rebuild เองแบบเต็มรูปแบบ ไม่ใช่แค่ re-run query เดิม

---

## Step 920: แบบฝึกหัดรวม — Implement Event Sourcing + CQRS สำหรับ Order Aggregate

มาสร้างระบบ Event Sourcing + CQRS แบบสมบูรณ์ตั้งแต่ต้นจนจบ โดยรองรับ event 4 ประเภท: `OrderCreated`, `ItemAdded`, `OrderShipped`, `OrderCancelled` พร้อม read model สำหรับแสดงผล order summary

### ขั้นที่ 1: สร้าง Schema ทั้งหมด

```sql
-- (สมมติว่า order_events ถูกสร้างไว้แล้วตามต้นบท)

-- Read model
CREATE TABLE IF NOT EXISTS order_summary_read_model (
    aggregate_id       UUID PRIMARY KEY,
    customer_id        VARCHAR(50) NOT NULL,
    status              VARCHAR(20) NOT NULL,
    item_count          INTEGER NOT NULL DEFAULT 0,
    total_amount        NUMERIC(12,2) NOT NULL DEFAULT 0,
    currency            VARCHAR(3) NOT NULL DEFAULT 'THB',
    tracking_no          VARCHAR(50),
    cancel_reason        TEXT,
    last_event_version   INTEGER NOT NULL DEFAULT 0,
    updated_at           TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### ขั้นที่ 2: ฟังก์ชัน Command สำหรับสร้าง Order ใหม่

```sql
CREATE OR REPLACE FUNCTION cmd_create_order(
    p_customer_id VARCHAR(50),
    p_currency VARCHAR(3) DEFAULT 'THB'
) RETURNS UUID AS $$
DECLARE
    v_aggregate_id UUID := gen_random_uuid();
BEGIN
    INSERT INTO order_events (aggregate_id, event_type, event_data, event_version)
    VALUES (
        v_aggregate_id,
        'OrderCreated',
        jsonb_build_object('customer_id', p_customer_id, 'currency', p_currency),
        1
    );
    RETURN v_aggregate_id;
END;
$$ LANGUAGE plpgsql;
```

### ขั้นที่ 3: ฟังก์ชัน Command สำหรับเพิ่มสินค้า (พร้อมตรวจสอบ invariant)

```sql
CREATE OR REPLACE FUNCTION cmd_add_item(
    p_aggregate_id UUID,
    p_sku VARCHAR(50),
    p_name VARCHAR(200),
    p_quantity INTEGER,
    p_unit_price NUMERIC(10,2)
) RETURNS BIGINT AS $$
DECLARE
    v_state JSONB;
    v_current_version INTEGER;
    v_new_event_id BIGINT;
BEGIN
    v_state := replay_order_state(p_aggregate_id);

    IF v_state IS NULL THEN
        RAISE EXCEPTION 'ไม่พบ order aggregate_id = %', p_aggregate_id;
    END IF;

    IF v_state->>'status' NOT IN ('created') THEN
        RAISE EXCEPTION 'ไม่สามารถเพิ่มสินค้าได้ เพราะ order อยู่ในสถานะ % แล้ว', v_state->>'status';
    END IF;

    IF p_quantity <= 0 THEN
        RAISE EXCEPTION 'จำนวนสินค้าต้องมากกว่า 0';
    END IF;

    v_current_version := (v_state->>'version')::INTEGER;

    INSERT INTO order_events (aggregate_id, event_type, event_data, event_version)
    VALUES (
        p_aggregate_id,
        'ItemAdded',
        jsonb_build_object(
            'sku', p_sku, 'name', p_name,
            'quantity', p_quantity, 'unit_price', p_unit_price
        ),
        v_current_version + 1
    )
    RETURNING event_id INTO v_new_event_id;

    RETURN v_new_event_id;
END;
$$ LANGUAGE plpgsql;
```

### ขั้นที่ 4: ฟังก์ชัน Command สำหรับจัดส่งและยกเลิก

```sql
CREATE OR REPLACE FUNCTION cmd_ship_order(
    p_aggregate_id UUID,
    p_carrier VARCHAR(100),
    p_tracking_no VARCHAR(50)
) RETURNS BIGINT AS $$
DECLARE
    v_state JSONB;
    v_new_event_id BIGINT;
BEGIN
    v_state := replay_order_state(p_aggregate_id);

    IF v_state->>'status' != 'created' THEN
        RAISE EXCEPTION 'จัดส่งได้เฉพาะ order ที่อยู่ในสถานะ created (ปัจจุบัน: %)', v_state->>'status';
    END IF;

    IF jsonb_array_length(v_state->'items') = 0 THEN
        RAISE EXCEPTION 'ไม่สามารถจัดส่ง order ที่ไม่มีสินค้าได้';
    END IF;

    INSERT INTO order_events (aggregate_id, event_type, event_data, event_version)
    VALUES (
        p_aggregate_id, 'OrderShipped',
        jsonb_build_object('carrier', p_carrier, 'tracking_no', p_tracking_no),
        (v_state->>'version')::INTEGER + 1
    )
    RETURNING event_id INTO v_new_event_id;

    RETURN v_new_event_id;
END;
$$ LANGUAGE plpgsql;

CREATE OR REPLACE FUNCTION cmd_cancel_order(
    p_aggregate_id UUID,
    p_reason TEXT
) RETURNS BIGINT AS $$
DECLARE
    v_state JSONB;
    v_new_event_id BIGINT;
BEGIN
    v_state := replay_order_state(p_aggregate_id);

    IF v_state->>'status' = 'shipped' THEN
        RAISE EXCEPTION 'ไม่สามารถยกเลิก order ที่จัดส่งไปแล้วได้';
    END IF;

    IF v_state->>'status' = 'cancelled' THEN
        RAISE EXCEPTION 'order นี้ถูกยกเลิกไปแล้ว';
    END IF;

    INSERT INTO order_events (aggregate_id, event_type, event_data, event_version)
    VALUES (
        p_aggregate_id, 'OrderCancelled',
        jsonb_build_object('reason', p_reason),
        (v_state->>'version')::INTEGER + 1
    )
    RETURNING event_id INTO v_new_event_id;

    RETURN v_new_event_id;
END;
$$ LANGUAGE plpgsql;
```

### ขั้นที่ 5: Trigger สร้าง Read Model แบบอัตโนมัติ (ครบทั้ง 4 event type)

```sql
ALTER TABLE order_summary_read_model
    ADD CONSTRAINT uq_summary_aggregate2 UNIQUE (aggregate_id);

CREATE OR REPLACE FUNCTION project_order_summary_full()
RETURNS TRIGGER AS $$
BEGIN
    CASE NEW.event_type
        WHEN 'OrderCreated' THEN
            INSERT INTO order_summary_read_model (
                aggregate_id, customer_id, status, currency, last_event_version
            ) VALUES (
                NEW.aggregate_id, NEW.event_data->>'customer_id', 'created',
                COALESCE(NEW.event_data->>'currency', 'THB'), NEW.event_version
            );

        WHEN 'ItemAdded' THEN
            UPDATE order_summary_read_model
            SET item_count = item_count + (NEW.event_data->>'quantity')::int,
                total_amount = total_amount +
                    (NEW.event_data->>'quantity')::numeric * (NEW.event_data->>'unit_price')::numeric,
                last_event_version = NEW.event_version,
                updated_at = now()
            WHERE aggregate_id = NEW.aggregate_id;

        WHEN 'OrderShipped' THEN
            UPDATE order_summary_read_model
            SET status = 'shipped', tracking_no = NEW.event_data->>'tracking_no',
                last_event_version = NEW.event_version, updated_at = now()
            WHERE aggregate_id = NEW.aggregate_id;

        WHEN 'OrderCancelled' THEN
            UPDATE order_summary_read_model
            SET status = 'cancelled', cancel_reason = NEW.event_data->>'reason',
                last_event_version = NEW.event_version, updated_at = now()
            WHERE aggregate_id = NEW.aggregate_id;

        ELSE
            NULL;
    END CASE;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_project_order_summary_full
    AFTER INSERT ON order_events
    FOR EACH ROW EXECUTE FUNCTION project_order_summary_full();
```

### ขั้นที่ 6: ทดสอบทั้งระบบแบบ End-to-End

```sql
-- 1) สร้าง order ใหม่
SELECT cmd_create_order('cust-3003', 'THB') AS new_order_id \gset
```

```sql
-- 2) เพิ่มสินค้า 2 รายการ
SELECT cmd_add_item(:'new_order_id'::uuid, 'SKU-C300', 'หูฟังบลูทูธ', 1, 990.00);
SELECT cmd_add_item(:'new_order_id'::uuid, 'SKU-D400', 'สายชาร์จ USB-C', 3, 150.00);

-- 3) จัดส่ง order
SELECT cmd_ship_order(:'new_order_id'::uuid, 'Thailand Post', 'TP778899001');
```

```sql
-- 4) ดู read model ทันที (ควรอัปเดตครบถ้วนแล้ว เพราะ trigger เป็น synchronous)
SELECT customer_id, status, item_count, total_amount, currency, tracking_no
FROM order_summary_read_model
WHERE aggregate_id = :'new_order_id'::uuid;
```

```
 customer_id | status  | item_count | total_amount | currency | tracking_no
-------------+---------+------------+---------------+----------+--------------
 cust-3003   | shipped |          4 |       1440.00 | THB      | TP778899001
(1 row)
```

```sql
-- 5) ตรวจสอบว่า replay_order_state ให้ผลลัพธ์ตรงกับ read model (ความสอดคล้องระหว่าง write/read model)
SELECT replay_order_state(:'new_order_id'::uuid);
```

```
{"items": [{"sku": "SKU-C300", "quantity": 1, "unit_price": 990.00},
           {"sku": "SKU-D400", "quantity": 3, "unit_price": 150.00}],
 "status": "shipped", "version": 3, "currency": "THB",
 "customer_id": "cust-3003", "tracking_no": "TP778899001"}
```

```sql
-- 6) ทดสอบ business invariant: ห้ามยกเลิก order ที่จัดส่งแล้ว
SELECT cmd_cancel_order(:'new_order_id'::uuid, 'ทดสอบยกเลิก');
```

```
ERROR:  ไม่สามารถยกเลิก order ที่จัดส่งไปแล้วได้
```

การทดสอบนี้แสดงให้เห็นว่าระบบ Event Sourcing + CQRS ทำงานถูกต้องครบวงจร: command ตรวจสอบ business rule ก่อนเขียน event, event ถูกเก็บ append-only, และ read model อัปเดตอัตโนมัติให้ตรงกับ write model เสมอ

---

## สรุปท้ายบท

ในบทนี้เราได้เรียนรู้การนำ PostgreSQL มาใช้ implement สถาปัตยกรรม **Event Sourcing** และ **CQRS** ซึ่งเป็น pattern ระดับสูงที่ใช้แก้ปัญหาระบบที่ต้องการ audit trail สมบูรณ์และ business logic ที่ซับซ้อน:

- **Event Sourcing** เก็บลำดับ event ทั้งหมดแทนที่จะเก็บแค่ state ปัจจุบัน ทำให้ได้ audit trail, time travel, และ debugging ที่ทรงพลัง แต่แลกมาด้วยความซับซ้อนที่เพิ่มขึ้น
- **Event Store** ออกแบบด้วยตาราง append-only ที่มี `aggregate_id`, `event_type`, `event_data` (JSONB), และ `event_version` เป็นแกนหลัก
- **UNIQUE (aggregate_id, event_version)** คือกลไก Optimistic Concurrency Control ที่เรียบง่ายแต่ทรงพลัง ป้องกัน conflict โดยไม่ต้อง lock ล่วงหน้า
- **Replay** คือการอ่าน event เรียงตามลำดับแล้วประมวลผลทีละตัวเพื่อคำนวณ state ปัจจุบัน หรือ state ณ เวลาใดเวลาหนึ่ง (time travel)
- **Snapshot Pattern** ช่วยลดต้นทุนการ replay สำหรับ aggregate ที่มี event จำนวนมาก โดยยังคงให้ event store เป็น source of truth เสมอ
- **CQRS** แยก write model (เพื่อความถูกต้องของ business logic) ออกจาก read model (เพื่อความเร็วในการ query) ซึ่งเข้ากันได้ดีเป็นพิเศษกับ Event Sourcing
- **Projection** คือกระบวนการแปลง event stream เป็น read model ทำได้ทั้งแบบ synchronous (trigger) และ asynchronous (message queue / LISTEN-NOTIFY) แต่ละแบบมี trade-off ระหว่าง consistency กับ throughput
- Read model **rebuild ได้เสมอ** จาก event stream โดยไม่สูญเสียข้อมูล เพราะ source of truth คือ event store ไม่ใช่ read model

Pattern เหล่านี้ไม่ควรใช้กับทุกระบบ แต่เหมาะอย่างยิ่งกับ domain ที่ audit trail, compliance, และ business logic ที่ซับซ้อนเป็นเรื่องสำคัญ เช่น ระบบการเงิน, e-commerce order management, และ workflow approval

---

## แบบฝึกหัด

<details>
<summary><b>แบบฝึกหัดที่ 1:</b> จงอธิบายความแตกต่างระหว่าง state-based persistence กับ Event Sourcing โดยยกตัวอย่างสถานการณ์ที่ Event Sourcing ให้ประโยชน์ที่ state-based ทำไม่ได้</summary>

**เฉลย:**

State-based persistence เก็บเฉพาะสถานะปัจจุบันของข้อมูล เมื่อมีการเปลี่ยนแปลงจะ `UPDATE` ทับข้อมูลเดิม ทำให้ข้อมูลในอดีตหายไป ส่วน Event Sourcing เก็บลำดับเหตุการณ์ (event) ทั้งหมดที่เคยเกิดขึ้น โดยสถานะปัจจุบันคำนวณได้จากการ replay event เหล่านั้น

ตัวอย่างสถานการณ์ที่ Event Sourcing ให้ประโยชน์ที่ state-based ทำไม่ได้: ลูกค้าโทรมาร้องเรียนว่า "ทำไมยอดในตะกร้าสินค้าของฉันถึงเปลี่ยนจาก 5 ชิ้นเป็น 3 ชิ้น โดยที่ฉันไม่ได้ลบอะไรเลย" ในระบบ state-based เราเห็นแค่ตัวเลข 3 ชิ้นปัจจุบัน ไม่มีทางรู้ได้เลยว่าเกิดอะไรขึ้นระหว่างทาง แต่ในระบบ Event Sourcing เราสามารถ query event ทั้งหมดของ aggregate นั้น เรียงตามเวลา แล้วพบว่ามี event `ItemRemoved` ที่เกิดจาก batch job อัตโนมัติเมื่อ 10 นาทีก่อน — ทำให้ debug root cause ได้ทันที ซึ่งเป็นสิ่งที่ state-based persistence ทำไม่ได้เลยเพราะข้อมูลถูกทับไปแล้ว
</details>

<details>
<summary><b>แบบฝึกหัดที่ 2:</b> จงเขียน SQL สร้างตาราง `product_events` สำหรับ event store ของ product aggregate (แทน order) โดยมีโครงสร้างเทียบเท่ากับ `order_events` พร้อม index ที่จำเป็น</summary>

**เฉลย:**

```sql
CREATE TABLE product_events (
    event_id       BIGSERIAL PRIMARY KEY,
    aggregate_id   UUID NOT NULL,
    event_type     VARCHAR(50) NOT NULL,
    event_data     JSONB NOT NULL,
    event_version  INTEGER NOT NULL,
    created_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (aggregate_id, event_version)
);

CREATE INDEX idx_product_events_aggregate
    ON product_events (aggregate_id, event_version);

CREATE INDEX idx_product_events_type
    ON product_events (event_type);

CREATE INDEX idx_product_events_data_gin
    ON product_events USING GIN (event_data);
```

โครงสร้างนี้เหมือนกับ `order_events` ทุกประการ เพราะเป็น pattern มาตรฐานของ event store ที่ใช้ได้กับทุก aggregate type เพียงเปลี่ยนชื่อตารางและ event_type ที่รองรับให้เหมาะกับ domain ของ product (เช่น `ProductCreated`, `PriceChanged`, `StockAdjusted`)
</details>

<details>
<summary><b>แบบฝึกหัดที่ 3:</b> เขียน query เพื่อหาว่า order ใดบ้างที่มี event มากกว่า 50 event (เป็น candidate ที่ควรสร้าง snapshot) เรียงจากมากไปน้อย</summary>

**เฉลย:**

```sql
SELECT aggregate_id, COUNT(*) AS event_count, MAX(event_version) AS latest_version
FROM order_events
GROUP BY aggregate_id
HAVING COUNT(*) > 50
ORDER BY event_count DESC;
```

Query นี้ใช้ `GROUP BY aggregate_id` เพื่อนับ event ต่อ aggregate แล้วกรองด้วย `HAVING COUNT(*) > 50` เพื่อหาเฉพาะ aggregate ที่มี event เยอะจนอาจกระทบ performance การ replay — ผลลัพธ์นี้ใช้เป็น input ให้ background job ที่รัน `create_order_snapshot()` ให้กับแต่ละ aggregate_id ที่ query ออกมาได้
</details>

<details>
<summary><b>แบบฝึกหัดที่ 4:</b> จงอธิบายว่าทำไม UNIQUE constraint บน (aggregate_id, event_version) จึงเพียงพอสำหรับ Optimistic Concurrency Control โดยไม่ต้องใช้ SELECT ... FOR UPDATE</summary>

**เฉลย:**

UNIQUE constraint ทำงานที่ระดับ database engine เมื่อมี transaction สอง process พยายาม `INSERT` event ที่มี `(aggregate_id, event_version)` คู่เดียวกันพร้อมกัน PostgreSQL จะรับประกันว่ามีเพียง transaction เดียวเท่านั้นที่ insert สำเร็จ ส่วนอีก transaction จะได้รับ `unique_violation` error ทันที

กลไกนี้เพียงพอเพราะธรรมชาติของ event store คือ **append-only** — เราไม่เคย `UPDATE` แถวเดิม มีแต่การ `INSERT` แถวใหม่เท่านั้น ดังนั้นจุดที่อาจเกิด conflict คือตอน insert เท่านั้น ซึ่ง UNIQUE constraint ดักจับได้ตรงจุดพอดี

การใช้ `SELECT ... FOR UPDATE` (pessimistic locking) จะต้อง lock แถวไว้ตลอดช่วงเวลาที่ทำ business logic ก่อน insert ทำให้ transaction อื่นที่ต้องการอ่าน/เขียน aggregate เดียวกันต้องรอ ลด throughput ของระบบ ในขณะที่ optimistic concurrency ด้วย UNIQUE constraint อนุญาตให้ทุก process อ่านและคำนวณคู่ขนานได้เต็มที่ แล้วปล่อยให้ database ตัดสินตอน insert จริงเท่านั้น ซึ่งเหมาะสมกับ workload ที่ conflict เกิดขึ้นไม่บ่อยนัก (สอดคล้องกับหลักการของ optimistic concurrency control โดยทั่วไป)
</details>

<details>
<summary><b>แบบฝึกหัดที่ 5:</b> เขียนฟังก์ชัน `replay_order_state_up_to(p_aggregate_id UUID, p_up_to_version INTEGER)` ที่ replay event เฉพาะที่ event_version ไม่เกินค่าที่กำหนด (ใช้สำหรับ time travel)</summary>

**เฉลย:**

```sql
CREATE OR REPLACE FUNCTION replay_order_state_up_to(
    p_aggregate_id UUID,
    p_up_to_version INTEGER
) RETURNS JSONB AS $$
DECLARE
    v_event RECORD;
    v_state JSONB := jsonb_build_object('status', 'draft', 'items', '[]'::jsonb, 'version', 0);
    v_items JSONB;
    v_new_item JSONB;
BEGIN
    FOR v_event IN
        SELECT event_type, event_data, event_version
        FROM order_events
        WHERE aggregate_id = p_aggregate_id
          AND event_version <= p_up_to_version
        ORDER BY event_version ASC
    LOOP
        CASE v_event.event_type
            WHEN 'OrderCreated' THEN
                v_state := v_state
                    || jsonb_build_object('status', 'created')
                    || jsonb_build_object('customer_id', v_event.event_data->>'customer_id');
            WHEN 'ItemAdded' THEN
                v_items := v_state->'items';
                v_new_item := jsonb_build_object(
                    'sku', v_event.event_data->>'sku',
                    'quantity', (v_event.event_data->>'quantity')::int
                );
                v_state := jsonb_set(v_state, '{items}', v_items || jsonb_build_array(v_new_item));
            WHEN 'OrderShipped' THEN
                v_state := v_state || jsonb_build_object('status', 'shipped');
            WHEN 'OrderCancelled' THEN
                v_state := v_state || jsonb_build_object('status', 'cancelled');
            ELSE
                NULL;
        END CASE;
        v_state := jsonb_set(v_state, '{version}', to_jsonb(v_event.event_version));
    END LOOP;

    RETURN v_state;
END;
$$ LANGUAGE plpgsql;
```

จุดสำคัญคือเงื่อนไข `AND event_version <= p_up_to_version` ใน `WHERE` clause ซึ่งจำกัดให้ replay เฉพาะ event ที่เกิดขึ้น "จนถึง" version ที่ระบุเท่านั้น ทำให้สามารถดู state ของ order ณ จุดใดจุดหนึ่งในอดีตได้ (เช่น "state ตอนที่มี 3 event แรกเท่านั้น" แม้ว่าปัจจุบันจะมี event ทั้งหมด 10 ตัวแล้ว)
</details>

<details>
<summary><b>แบบฝึกหัดที่ 6:</b> จงอธิบายว่าทำไม snapshot ไม่ควรถูกมองว่าเป็น source of truth และถ้า snapshot data เสียหาย ระบบควรทำอย่างไร</summary>

**เฉลย:**

Snapshot เป็นเพียง **cache ของผลลัพธ์การ replay ณ event_version หนึ่ง ๆ** ที่สร้างขึ้นเพื่อเหตุผลด้าน performance เท่านั้น ไม่ใช่ข้อมูลต้นทาง (source of truth) ที่แท้จริง — source of truth ที่แท้จริงคือ event ทั้งหมดใน event store (`order_events`)

เหตุผลที่ไม่ควรมองว่า snapshot เป็น source of truth:
1. Snapshot คำนวณ (derive) มาจาก event เสมอ ถ้าลบ snapshot ทิ้งทั้งหมด ระบบยังสามารถคำนวณ state เดิมกลับมาได้ครบถ้วนจาก event store
2. ถ้า logic การสร้าง snapshot มี bug (เช่น คำนวณ total_amount ผิด) snapshot ที่ error จะไม่ทำให้ข้อมูลจริงเสียหาย เพราะ event ต้นทางยังถูกต้องอยู่เสมอ

หาก snapshot data เสียหายหรือพบว่าคำนวณผิด สิ่งที่ควรทำคือ:
```sql
-- ลบ snapshot ที่เสียหายทั้งหมด (ปลอดภัย เพราะไม่กระทบ event store)
TRUNCATE TABLE order_snapshots;

-- ระบบจะกลับไป replay จาก event ทั้งหมดโดยอัตโนมัติ (ช้าลงชั่วคราว)
-- จากนั้นรัน job สร้าง snapshot ใหม่ให้กับ aggregate ที่มี event เยอะ
SELECT create_order_snapshot(aggregate_id)
FROM (
    SELECT aggregate_id, COUNT(*) AS event_count
    FROM order_events
    GROUP BY aggregate_id
    HAVING COUNT(*) > 50
) AS heavy_aggregates;
```
</details>

<details>
<summary><b>แบบฝึกหัดที่ 7:</b> จงยกตัวอย่างระบบที่ใช้ CQRS โดยไม่มี Event Sourcing และอธิบายว่า write model กับ read model ในตัวอย่างนั้นมีหน้าตาอย่างไร</summary>

**เฉลย:**

ตัวอย่าง: ระบบ blog platform ที่มี write model เป็นตาราง normalized ปกติ:

```sql
-- Write model: normalized, เน้นความถูกต้อง
CREATE TABLE posts (
    post_id BIGINT PRIMARY KEY,
    author_id BIGINT NOT NULL,
    title TEXT NOT NULL,
    body TEXT NOT NULL,
    status VARCHAR(20) NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE post_tags (
    post_id BIGINT REFERENCES posts(post_id),
    tag_id BIGINT REFERENCES tags(tag_id),
    PRIMARY KEY (post_id, tag_id)
);
```

```sql
-- Read model: materialized view แบบ denormalized เน้นความเร็วในการแสดงผลหน้า listing
CREATE MATERIALIZED VIEW post_listing_read_model AS
SELECT
    p.post_id, p.title, p.status, u.display_name AS author_name,
    string_agg(t.tag_name, ', ') AS tags,
    p.updated_at
FROM posts p
JOIN users u ON u.user_id = p.author_id
LEFT JOIN post_tags pt ON pt.post_id = p.post_id
LEFT JOIN tags t ON t.tag_id = pt.tag_id
GROUP BY p.post_id, p.title, p.status, u.display_name, p.updated_at;

REFRESH MATERIALIZED VIEW post_listing_read_model;
```

ในตัวอย่างนี้ write model (`posts`, `post_tags`) ยัง**เก็บแค่ state ปัจจุบัน** ไม่มี event ใด ๆ ถูกเก็บไว้ (จึงไม่ใช่ Event Sourcing) แต่ยังคงแยก read model ออกมาเป็น materialized view ต่างหากเพื่อความเร็วในการ query หน้า listing โดยไม่ต้อง JOIN สามตารางทุกครั้ง — นี่คือ CQRS ที่ไม่มี Event Sourcing มาเกี่ยวข้อง (concept นี้เคยกล่าวถึงใน Part 036 เรื่อง Materialized View)
</details>

<details>
<summary><b>แบบฝึกหัดที่ 8:</b> Trigger-based projection (synchronous) กับ Application-level projection (asynchronous ผ่าน LISTEN/NOTIFY) มีข้อดีข้อเสียต่างกันอย่างไร ควรเลือกใช้แบบไหนในสถานการณ์ใด</summary>

**เฉลย:**

**Trigger-based (synchronous):**
- ข้อดี: read model สอดคล้องกับ write model ทันที (strong consistency) ภายใน transaction เดียวกัน implement ง่าย ไม่ต้องมี infrastructure เพิ่ม (message queue, worker process)
- ข้อเสีย: เพิ่ม latency ให้ write transaction เพราะต้องรอ trigger ทำงานเสร็จก่อน commit; ถ้า read model มีหลายตัวและ logic ซับซ้อน จะทำให้ transaction เขียน event ช้าลงมาก; coupling ระหว่าง write และ read model แน่นเกินไป (เปลี่ยน schema read model กระทบ trigger โดยตรง)
- เหมาะกับ: ระบบขนาดเล็กถึงกลาง, use case ที่ consistency สำคัญมาก (เช่น ยอดคงเหลือบัญชีที่ต้องแม่นยำทันที), read model จำนวนน้อยและ logic ไม่ซับซ้อน

**Application-level (asynchronous):**
- ข้อดี: write path เร็วที่สุด ไม่ต้องรอ read model อัปเดต; scale write/read แยกจากกันได้อิสระ; read model อยู่คนละ technology ได้ (Elasticsearch, Redis, data warehouse); เพิ่ม read model ใหม่ได้โดยไม่กระทบ write path เลย
- ข้อเสีย: ต้องยอมรับ eventual consistency (มีช่วง delay สั้น ๆ ที่ read model ไม่ตรงกับ write model); ต้องมี infrastructure และ monitoring เพิ่ม (worker, queue, retry, dead-letter); debug ยากขึ้นเพราะ flow กระจายหลาย process
- เหมาะกับ: ระบบขนาดใหญ่ที่ throughput ของ write path สำคัญมาก, มี read model หลายตัวสำหรับหลาย use case, ระบบที่ยอมรับ eventual consistency ได้ (เช่น หน้า dashboard, search index, analytics)

โดยทั่วไประบบจริงมักใช้**ทั้งสองแบบผสมกัน**: read model ที่ consistency สำคัญมาก (เช่น current order status ที่ต้องแสดงถูกต้องทันที) ใช้ trigger-based ส่วน read model ที่ใช้เพื่อ analytics/reporting/search ใช้ asynchronous ได้
</details>

<details>
<summary><b>แบบฝึกหัดที่ 9:</b> จงเขียนฟังก์ชัน command <code>cmd_remove_item</code> ที่รองรับการลบสินค้าออกจาก order (event type <code>ItemRemoved</code>) พร้อมตรวจสอบว่าสินค้านั้นมีอยู่ในตะกร้าจริงและ order ยังไม่ถูก ship/cancel</summary>

**เฉลย:**

```sql
CREATE OR REPLACE FUNCTION cmd_remove_item(
    p_aggregate_id UUID,
    p_sku VARCHAR(50),
    p_quantity INTEGER
) RETURNS BIGINT AS $$
DECLARE
    v_state JSONB;
    v_matching_item JSONB;
    v_new_event_id BIGINT;
BEGIN
    v_state := replay_order_state(p_aggregate_id);

    IF v_state IS NULL THEN
        RAISE EXCEPTION 'ไม่พบ order aggregate_id = %', p_aggregate_id;
    END IF;

    IF v_state->>'status' != 'created' THEN
        RAISE EXCEPTION 'ไม่สามารถลบสินค้าได้ เพราะ order อยู่ในสถานะ % แล้ว', v_state->>'status';
    END IF;

    -- ตรวจสอบว่าสินค้านี้มีอยู่ในตะกร้าจริง และจำนวนที่จะลบไม่เกินจำนวนที่มี
    SELECT elem INTO v_matching_item
    FROM jsonb_array_elements(v_state->'items') AS elem
    WHERE elem->>'sku' = p_sku;

    IF v_matching_item IS NULL THEN
        RAISE EXCEPTION 'ไม่พบสินค้า sku=% ในตะกร้าของ order นี้', p_sku;
    END IF;

    IF (v_matching_item->>'quantity')::int < p_quantity THEN
        RAISE EXCEPTION 'จำนวนที่ต้องการลบ (%) มากกว่าจำนวนที่มีอยู่ในตะกร้า (%)',
            p_quantity, v_matching_item->>'quantity';
    END IF;

    INSERT INTO order_events (aggregate_id, event_type, event_data, event_version)
    VALUES (
        p_aggregate_id,
        'ItemRemoved',
        jsonb_build_object(
            'sku', p_sku,
            'quantity', p_quantity,
            'unit_price', (v_matching_item->>'unit_price')::numeric
        ),
        (v_state->>'version')::INTEGER + 1
    )
    RETURNING event_id INTO v_new_event_id;

    RETURN v_new_event_id;
END;
$$ LANGUAGE plpgsql;
```

จุดสำคัญของ command handler นี้คือการตรวจสอบ invariant สองชั้นก่อนเขียน event เสมอ: (1) order ต้องยังอยู่ในสถานะ `created` เท่านั้น ห้ามแก้ไขตะกร้าของ order ที่ shipped/cancelled ไปแล้ว และ (2) สินค้าที่จะลบต้องมีอยู่จริงและจำนวนต้องเพียงพอ — ทั้งหมดนี้ตรวจสอบโดยการ `replay_order_state()` ก่อนเสมอ ซึ่งสะท้อนหลักการสำคัญของ Event Sourcing ว่า **ทุก command ต้องอ่าน state ล่าสุดก่อนตัดสินใจเขียน event ใหม่**
</details>

<details>
<summary><b>แบบฝึกหัดที่ 10:</b> จงออกแบบและอธิบายกลยุทธ์สำหรับปัญหาต่อไปนี้: ทีมพัฒนาต้องการเพิ่ม read model ใหม่ชื่อ <code>order_analytics_read_model</code> เพื่อรายงานยอดขายรายวันแยกตาม SKU โดยไม่กระทบระบบที่ใช้งานอยู่ในปัจจุบัน (ทั้ง write model และ read model เดิม)</summary>

**เฉลย:**

ขั้นตอนกลยุทธ์ที่แนะนำ:

1. **ไม่ต้องแตะ write model เลย** — เพราะ event store (`order_events`) เป็น append-only และ schema-agnostic (payload เป็น JSONB) อยู่แล้ว read model ใหม่ใด ๆ อ่าน event เดิมได้โดยไม่ต้อง migrate หรือแก้ event structure

2. **สร้างตาราง read model ใหม่แยกต่างหาก:**

```sql
CREATE TABLE order_analytics_read_model (
    sale_date   DATE NOT NULL,
    sku         VARCHAR(50) NOT NULL,
    total_quantity INTEGER NOT NULL DEFAULT 0,
    total_revenue  NUMERIC(14,2) NOT NULL DEFAULT 0,
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (sale_date, sku)
);
```

3. **เขียน trigger หรือ event handler ใหม่แยกต่างหาก** ที่ subscribe เฉพาะ event type `ItemAdded` (event type เดียวที่เกี่ยวข้องกับรายงานนี้) โดยไม่แตะ trigger เดิมของ `order_summary_read_model` เลย:

```sql
CREATE OR REPLACE FUNCTION project_order_analytics()
RETURNS TRIGGER AS $$
BEGIN
    IF NEW.event_type = 'ItemAdded' THEN
        INSERT INTO order_analytics_read_model (sale_date, sku, total_quantity, total_revenue)
        VALUES (
            NEW.created_at::date,
            NEW.event_data->>'sku',
            (NEW.event_data->>'quantity')::int,
            (NEW.event_data->>'quantity')::numeric * (NEW.event_data->>'unit_price')::numeric
        )
        ON CONFLICT (sale_date, sku) DO UPDATE
            SET total_quantity = order_analytics_read_model.total_quantity + EXCLUDED.total_quantity,
                total_revenue = order_analytics_read_model.total_revenue + EXCLUDED.total_revenue,
                updated_at = now();
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_project_order_analytics
    AFTER INSERT ON order_events
    FOR EACH ROW EXECUTE FUNCTION project_order_analytics();
```

4. **Backfill ข้อมูลย้อนหลังด้วยการ replay event stream ทั้งหมดที่มีอยู่แล้ว** (นี่คือจุดที่ Event Sourcing ให้ประโยชน์ชัดเจนที่สุด เพราะข้อมูลย้อนหลังยังอยู่ครบใน event store):

```sql
-- Backfill ครั้งเดียวเพื่อดึงข้อมูล ItemAdded event ในอดีตทั้งหมดมาคำนวณ
INSERT INTO order_analytics_read_model (sale_date, sku, total_quantity, total_revenue)
SELECT
    created_at::date AS sale_date,
    event_data->>'sku' AS sku,
    SUM((event_data->>'quantity')::int) AS total_quantity,
    SUM((event_data->>'quantity')::numeric * (event_data->>'unit_price')::numeric) AS total_revenue
FROM order_events
WHERE event_type = 'ItemAdded'
GROUP BY created_at::date, event_data->>'sku'
ON CONFLICT (sale_date, sku) DO UPDATE
    SET total_quantity = EXCLUDED.total_quantity,
        total_revenue = EXCLUDED.total_revenue,
        updated_at = now();
```

5. **Deploy โดยไม่ downtime** — เนื่องจาก trigger ใหม่นี้เป็น `AFTER INSERT` ที่แยกจาก trigger เดิมโดยสิ้นเชิง (คนละ function, คนละ trigger object) การเพิ่มเข้ามาไม่กระทบ trigger เดิมที่อัปเดต `order_summary_read_model` เลย และไม่ต้อง lock ตาราง `order_events` เป็นเวลานาน (แค่ `CREATE TRIGGER` ซึ่งเร็วมาก)

จุดสำคัญที่ทำให้กลยุทธ์นี้เป็นไปได้อย่างปลอดภัยคือหลักการพื้นฐานของ Event Sourcing + CQRS: **write model ไม่เคยรับรู้ถึงการมีอยู่ของ read model ใด ๆ เลย** (loose coupling) การเพิ่ม, แก้ไข, หรือลบ read model จึงทำได้อย่างอิสระ ตราบใดที่ event store ยังคงมีข้อมูลครบถ้วน — นี่คือคุณค่าหลักของสถาปัตยกรรมนี้ในระยะยาว
</details>

---

## บทถัดไป

บทถัดไปจะพาไปสำรวจการนำ PostgreSQL ไปใช้งานจริงในสภาพแวดล้อม container และ orchestration ระดับ production ด้วย Docker และ Kubernetes

**[Part 093: PostgreSQL กับ Docker และ Kubernetes →](./part-093-docker-kubernetes.md)**