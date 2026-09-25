# Database Migration Strategies (Zero-downtime Migration)

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับมืออาชีพ | Part 079

## เป้าหมายการเรียนรู้

เมื่อเรียนจบบทนี้ ผู้เรียนจะสามารถ:

- อธิบายแนวคิด **database migration** และ **schema versioning** ได้อย่างถูกต้อง เข้าใจว่าทำไมทีมพัฒนาซอฟต์แวร์ระดับมืออาชีพถึงต้อง "จัดเวอร์ชัน" การเปลี่ยนแปลงโครงสร้างฐานข้อมูลเหมือนกับที่จัดเวอร์ชัน source code
- เข้าใจภาพรวมของเครื่องมือ migration ยอดนิยม (Flyway, Liquibase, node-pg-migrate, Django migrations, Rails migrations) และแนวคิดร่วมที่เครื่องมือเหล่านี้ใช้เหมือนกัน
- เขียน migration ที่ดีตามหลัก **idempotent** และ **reversible** (มี up/down) พร้อมแบ่งเป็นขั้นตอนย่อยที่ปลอดภัย
- เข้าใจปัญหาของการรัน `ALTER TABLE` บนตารางขนาดใหญ่ใน production ว่าทำไมถึงทำให้ระบบ "ค้าง" (ACCESS EXCLUSIVE LOCK)
- ออกแบบและเขียน **zero-downtime migration pattern** สำหรับสถานการณ์ที่พบบ่อยที่สุด 5 แบบ ได้แก่ เพิ่มคอลัมน์, เปลี่ยนชื่อคอลัมน์, เปลี่ยน data type, เพิ่ม index, และเพิ่ม NOT NULL constraint
- วางแผน migration script แบบหลายขั้นตอน (multi-phase deployment) สำหรับการเปลี่ยนแปลง schema ครั้งใหญ่ โดยไม่ทำให้ระบบ e-commerce ที่กำลังให้บริการลูกค้าอยู่หยุดทำงาน

---

## โครงสร้างฐานข้อมูลตัวอย่างที่ใช้ในบทนี้

บทนี้ใช้ schema ระบบ e-commerce แบบย่อ ประกอบด้วยตาราง `customers`, `products`, `orders`, `order_items` ผู้เรียนสามารถรันโค้ดชุดนี้ในฐานข้อมูลทดสอบเพื่อไล่ตามตัวอย่างทุกขั้นตอนในบทนี้ได้จริง

```sql
-- ล้างของเก่า (กรณีรันซ้ำ)
DROP TABLE IF EXISTS order_items, orders, products, customers CASCADE;

CREATE TABLE customers (
    customer_id  BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    email        TEXT NOT NULL UNIQUE,
    full_name    TEXT NOT NULL,
    created_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE products (
    product_id   BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    sku          TEXT NOT NULL UNIQUE,
    product_name TEXT NOT NULL,
    price        NUMERIC(12,2) NOT NULL,
    stock_qty    INTEGER NOT NULL DEFAULT 0,
    created_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE orders (
    order_id      BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    customer_id   BIGINT NOT NULL REFERENCES customers(customer_id),
    order_status  TEXT NOT NULL DEFAULT 'pending',
    total_amount  NUMERIC(12,2) NOT NULL,
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE order_items (
    order_item_id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    order_id      BIGINT NOT NULL REFERENCES orders(order_id),
    product_id    BIGINT NOT NULL REFERENCES products(product_id),
    quantity      INTEGER NOT NULL,
    unit_price    NUMERIC(12,2) NOT NULL
);

-- ข้อมูลตัวอย่างพอสังเขป
INSERT INTO customers (email, full_name) VALUES
    ('somchai@example.com', 'Somchai Jaidee'),
    ('malee@example.com',   'Malee Suksan');

INSERT INTO products (sku, product_name, price, stock_qty) VALUES
    ('SKU-001', 'Mechanical Keyboard', 1990.00, 120),
    ('SKU-002', 'Wireless Mouse',       590.00, 300);

INSERT INTO orders (customer_id, order_status, total_amount) VALUES
    (1, 'pending', 1990.00),
    (2, 'paid',     590.00);

INSERT INTO order_items (order_id, product_id, quantity, unit_price) VALUES
    (1, 1, 1, 1990.00),
    (2, 2, 1,  590.00);
```

สมมติในบทนี้ว่าตาราง `orders` และ `order_items` ในระบบจริงมีขนาดหลายสิบล้านแถว และระบบให้บริการลูกค้าตลอด 24 ชั่วโมง (จองตั๋ว ขายของ ชำระเงิน) — จึง**ไม่มีเวลา maintenance window** ที่จะปิดระบบเพื่อแก้ schema ได้ ทุก migration ในบทนี้จึงต้องออกแบบให้ "zero-downtime"

---

## Step 781: Database Migration คืออะไร — การเปลี่ยนแปลง schema แบบมีเวอร์ชัน (schema versioning)

### นิยาม

**Database migration** คือกระบวนการเปลี่ยนแปลงโครงสร้างฐานข้อมูล (schema) หรือข้อมูล (data) อย่างมีระบบ มีลำดับ และสามารถ**ทำซ้ำได้ (reproducible)** บนฐานข้อมูลหลายชุด เช่น เครื่อง developer, staging, production โดยแต่ละการเปลี่ยนแปลงจะถูกเขียนเป็นไฟล์ script ที่มี**เวอร์ชัน**กำกับ และรันตามลำดับเวอร์ชันเสมอ

ลองเปรียบเทียบกับ source code: เราไม่แก้โค้ดบน production server ตรง ๆ แต่เขียนเป็น commit → ผ่าน code review → deploy ตามลำดับ commit database migration ก็ใช้แนวคิดเดียวกัน คือไม่แก้ schema บน production ด้วยมือ (manual `ALTER TABLE` ผ่าน psql) แต่เขียนเป็นไฟล์ migration ที่ไล่เรียงเวอร์ชัน แล้วให้ระบบ deploy รันตามลำดับโดยอัตโนมัติ

### ทำไมต้อง "จัดเวอร์ชัน" schema

| ปัญหาถ้าไม่มี migration | ผลลัพธ์ |
|---|---|
| แก้ schema ผ่าน GUI/psql มือเปล่า | ไม่มีบันทึกว่าใครแก้อะไร เมื่อไหร่ |
| dev แต่ละคน schema ไม่ตรงกัน | บั๊กที่เกิดเฉพาะบางเครื่อง ("works on my machine") |
| deploy production ไม่รู้ว่าต้องรัน SQL อะไรบ้าง | ลืมรัน ALTER TABLE บาง statement ระบบ error |
| ต้อง rollback แต่ไม่รู้จะย้อนยังไง | ต้อง restore จาก backup ทั้งฐาน สูญเสียข้อมูลใหม่ |
| หลายทีมแก้ schema พร้อมกัน | เกิด conflict ไม่รู้ใครแก้ทับใคร |

### กลไกพื้นฐานของ schema versioning

ทุกเครื่องมือ migration (ไม่ว่าจะ Flyway, Liquibase, node-pg-migrate, Rails, Django) ใช้กลไกร่วมกันคือ:

1. **ไฟล์ migration ที่เรียงลำดับได้** เช่น ใช้ตัวเลขหรือ timestamp นำหน้าชื่อไฟล์
2. **ตารางบันทึกประวัติ (migration history table)** ในฐานข้อมูลเอง เก็บว่า migration ไหนรันไปแล้วบ้าง เมื่อไหร่ ใครรัน checksum เป็นอะไร
3. **ตัว runner** ที่เชื่อมต่อฐานข้อมูล เทียบไฟล์ migration บน disk กับตาราง history แล้วรันเฉพาะไฟล์ที่ยังไม่เคยรัน ตามลำดับ

เราสามารถสร้างกลไกนี้แบบง่าย ๆ ด้วยมือเพื่อความเข้าใจ ก่อนจะไปดูเครื่องมือจริงใน Step 782:

```sql
-- ตารางบันทึกประวัติ migration (แนวคิดเดียวกับทุกเครื่องมือ)
CREATE TABLE IF NOT EXISTS schema_migrations (
    version      TEXT PRIMARY KEY,        -- เช่น '001', '20260115103000'
    description  TEXT NOT NULL,
    checksum     TEXT,                    -- hash ของเนื้อไฟล์ ใช้ตรวจว่ามีใครแก้ไฟล์เก่าย้อนหลังไหม
    applied_by   TEXT NOT NULL DEFAULT current_user,
    applied_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    execution_ms INTEGER
);
```

ตัวอย่างไฟล์ migration แบบง่าย (โครงสร้างไดเรกทอรีสมมติ):

```
migrations/
  001_create_customers_table.sql
  002_create_products_table.sql
  003_create_orders_and_order_items.sql
  004_add_loyalty_points_to_customers.sql
```

และเมื่อรันแต่ละไฟล์ ตัว runner จะทำสิ่งนี้เสมอ (แนวคิด pseudo-transaction):

```sql
BEGIN;

-- 1) เนื้อหาจริงของ migration เช่น
ALTER TABLE customers ADD COLUMN IF NOT EXISTS loyalty_points INTEGER NOT NULL DEFAULT 0;

-- 2) บันทึกว่ารันแล้ว ในธุรกรรมเดียวกัน (atomic)
INSERT INTO schema_migrations (version, description, checksum)
VALUES ('004', 'add_loyalty_points_to_customers', 'sha256:abcd1234...');

COMMIT;
```

จุดสำคัญ: การ `INSERT INTO schema_migrations` ต้องอยู่ใน**ธุรกรรมเดียวกัน**กับการเปลี่ยนแปลง schema เสมอ เพื่อรับประกันว่าถ้า migration ล้มเหลวกลางทาง จะไม่มีการบันทึกว่า "รันสำเร็จ" ทั้งที่จริง ๆ ยังไม่สำเร็จ (ป้องกัน state ไม่ตรงกันระหว่าง schema จริงกับ history table)

> **หมายเหตุ:** statement บางตัว เช่น `CREATE INDEX CONCURRENTLY` และ `ALTER TYPE ... ADD VALUE` (สำหรับ enum) **ไม่สามารถรันภายใน transaction block ได้** เราจะพูดถึงผลกระทบนี้ต่อการออกแบบ migration ใน Step 788

### เป้าหมายสามข้อของ migration ที่ดี

1. **Repeatable** — รันบนฐานข้อมูลไหนก็ได้ผลลัพธ์เหมือนกัน
2. **Ordered** — มีลำดับที่ชัดเจน ไม่กำกวม
3. **Auditable** — ตรวจสอบย้อนหลังได้ว่า schema ปัจจุบันมาจาก migration ชุดไหนบ้าง

---

## Step 782: Migration Tools — Flyway, Liquibase, node-pg-migrate, Django migrations, Rails migrations (ภาพรวมแนวคิดร่วม)

เครื่องมือ migration แต่ละตัวมีไวยากรณ์และภาษาต่างกัน แต่**สถาปัตยกรรมเบื้องหลังเหมือนกันเกือบทั้งหมด** ตารางด้านล่างเปรียบเทียบให้เห็นภาพ

| เครื่องมือ | ภาษาที่เขียน migration | ตาราง history | รูปแบบชื่อไฟล์ |
|---|---|---|---|
| **Flyway** | SQL ล้วน (หรือ Java callback) | `flyway_schema_history` | `V1__create_customers.sql`, `V2__add_index.sql` |
| **Liquibase** | XML / YAML / JSON / SQL (changelog) | `DATABASECHANGELOG` | `changelog-master.xml` อ้างอิงไฟล์ย่อย |
| **node-pg-migrate** | JavaScript/TypeScript (หรือ SQL) | `pgmigrations` | `1700000000000_add-loyalty-points.js` |
| **Django migrations** | Python (`migrations.Migration` class) | `django_migrations` | `0004_add_loyalty_points.py` |
| **Rails (ActiveRecord)** | Ruby (`ActiveRecord::Migration`) | `schema_migrations` + `ar_internal_metadata` | `20260115103000_add_loyalty_points_to_customers.rb` |

### ตัวอย่างการเขียน migration เดียวกันในหลายเครื่องมือ

**Flyway** (`V4__add_loyalty_points_to_customers.sql`) — SQL ล้วน รันตรง ๆ:

```sql
-- V4__add_loyalty_points_to_customers.sql
ALTER TABLE customers
    ADD COLUMN loyalty_points INTEGER NOT NULL DEFAULT 0;
```

Flyway จะตรวจ prefix `V4__` เทียบกับ `flyway_schema_history.installed_rank` และ `version` โดยอัตโนมัติ พร้อมเก็บ checksum ของไฟล์ — ถ้ามีใครแก้ไฟล์ `V4__...sql` ที่เคย apply ไปแล้ว Flyway จะ**ปฏิเสธการรันครั้งถัดไปทันที** (validation error) เพื่อป้องกัน schema drift

**node-pg-migrate** (`1700000000000_add-loyalty-points.js`):

```javascript
exports.up = (pgm) => {
  pgm.addColumn('customers', {
    loyalty_points: { type: 'integer', notNull: true, default: 0 },
  });
};

exports.down = (pgm) => {
  pgm.dropColumn('customers', 'loyalty_points');
};
```

**Django migrations** (`0004_add_loyalty_points.py`):

```python
from django.db import migrations, models

class Migration(migrations.Migration):
    dependencies = [('shop', '0003_orders_and_order_items')]

    operations = [
        migrations.AddField(
            model_name='customer',
            name='loyalty_points',
            field=models.IntegerField(default=0),
        ),
    ]
```

**Rails** (`20260115103000_add_loyalty_points_to_customers.rb`):

```ruby
class AddLoyaltyPointsToCustomers < ActiveRecord::Migration[7.1]
  def change
    add_column :customers, :loyalty_points, :integer, null: false, default: 0
  end
end
```

### สิ่งที่เหมือนกันในทุกเครื่องมือ

1. **มีตาราง/กลไกติดตามว่า migration ไหนรันไปแล้ว** — ป้องกันการรันซ้ำ
2. **รันตามลำดับที่กำหนดตายตัว** — ไม่ว่าจะเป็นเลขเวอร์ชัน (Flyway) หรือ timestamp (Rails, node-pg-migrate) หรือ dependency graph (Django)
3. **แยกไฟล์ต่อการเปลี่ยนแปลงหนึ่งเรื่อง** — ไม่รวมหลาย ๆ การเปลี่ยนแปลงไว้ในไฟล์เดียวมั่ว ๆ
4. **บางตัวรองรับ auto-rollback (down migration)** — Django และ Rails generate ทั้ง apply/reverse ให้อัตโนมัติในหลายกรณี ส่วน Flyway community edition **ไม่รองรับ automated rollback** (ต้องเขียน migration ใหม่เพื่อ "undo" — แนวคิด "roll forward")
5. **ทำงานเป็นส่วนหนึ่งของ CI/CD pipeline** — รันตอน deploy อัตโนมัติ ไม่ใช่ให้คนรันมือ

### แนวทางเลือกเครื่องมือ (โดยสังเขป)

- โปรเจกต์ **Node.js / TypeScript** ที่ไม่ใช้ ORM ตัวเต็ม → `node-pg-migrate` หรือ Knex migrations
- โปรเจกต์ **polyglot / หลายภาษา / หลาย microservice** → **Flyway** หรือ **Liquibase** (เพราะเป็น standalone CLI ไม่ผูกกับภาษา application)
- โปรเจกต์ **Django** → ใช้ Django migrations ในตัว ไม่ต้องหาเครื่องมือเสริม
- โปรเจกต์ **Rails** → ใช้ ActiveRecord migrations ในตัว
- ต้องการ **audit ละเอียด ควบคุม rollback เข้มงวด** (เช่นองค์กรการเงิน) → Liquibase ซึ่งรองรับ `<rollback>` block ในทุก changeset

ไม่ว่าจะเลือกเครื่องมือใด **หลักการเขียน migration ที่ดี** ใน Step 783 ใช้ได้กับทุกเครื่องมือเหมือนกัน

---

## Step 783: หลักการเขียน Migration ที่ดี — idempotent, reversible (up/down), แยก migration เล็กๆ ทีละขั้น

### หลักที่ 1: Idempotent — รันซ้ำได้โดยไม่พัง

Migration ที่ดีควรออกแบบให้ถ้าเผลอรันซ้ำ (เช่น deploy script ค้างแล้วรันใหม่) จะไม่เกิด error ทำลายระบบ แม้ว่าเครื่องมือ migration ส่วนใหญ่จะกันการรันซ้ำผ่าน history table อยู่แล้ว แต่การเขียนแบบ idempotent ยังช่วยเวลาต้อง**รัน migration script ตรง ๆ ด้วยมือ** ในสถานการณ์ฉุกเฉิน

**ตัวอย่างที่ไม่ idempotent (อันตราย):**

```sql
-- ถ้ารันครั้งที่สอง จะ error: column "loyalty_points" already exists
ALTER TABLE customers ADD COLUMN loyalty_points INTEGER NOT NULL DEFAULT 0;
```

**ตัวอย่างที่ idempotent:**

```sql
ALTER TABLE customers ADD COLUMN IF NOT EXISTS loyalty_points INTEGER NOT NULL DEFAULT 0;
```

ตัวอย่างการสร้าง index แบบ idempotent:

```sql
CREATE INDEX IF NOT EXISTS idx_orders_customer_id ON orders (customer_id);
```

ตัวอย่างการเพิ่ม constraint แบบ idempotent ด้วย `DO` block (เพราะ `ADD CONSTRAINT` ไม่มี `IF NOT EXISTS` ในตัว):

```sql
DO $$
BEGIN
    IF NOT EXISTS (
        SELECT 1 FROM pg_constraint WHERE conname = 'chk_orders_total_amount_positive'
    ) THEN
        ALTER TABLE orders
            ADD CONSTRAINT chk_orders_total_amount_positive CHECK (total_amount >= 0);
    END IF;
END $$;
```

### หลักที่ 2: Reversible — ทุก up ควรมี down

การมี **down migration** (หรือ "rollback script") ช่วยให้เมื่อ deploy แล้วพบปัญหา สามารถย้อนกลับได้เร็วโดยไม่ต้อง restore backup ทั้งฐาน

```sql
-- up: 005_add_loyalty_points.sql
ALTER TABLE customers ADD COLUMN IF NOT EXISTS loyalty_points INTEGER NOT NULL DEFAULT 0;

-- down: 005_add_loyalty_points_rollback.sql
ALTER TABLE customers DROP COLUMN IF EXISTS loyalty_points;
```

แต่ไม่ใช่ทุก migration จะ reversible ได้แบบไม่มีต้นทุน ตัวอย่างเช่น การ `DROP COLUMN` — ถ้า rollback กลับด้วย `ADD COLUMN` ใหม่ **ข้อมูลเดิมในคอลัมน์นั้นจะหายไปถาวร** ไม่สามารถกู้คืนได้ด้วย migration เฉย ๆ

| ประเภทการเปลี่ยนแปลง | Reversible ได้ง่าย | หมายเหตุ |
|---|---|---|
| `ADD COLUMN` | ✅ ย้อนด้วย `DROP COLUMN` | ถ้ายังไม่มีแอปเขียนข้อมูลลงคอลัมน์ใหม่ ย้อนได้ปลอดภัย |
| `CREATE INDEX` | ✅ ย้อนด้วย `DROP INDEX` | ปลอดภัยเสมอ (ไม่กระทบข้อมูล) |
| `DROP COLUMN` | ⚠️ ย้อนไม่ได้จริง | ข้อมูลหายถาวร ต้องมี backup/snapshot แยก |
| `RENAME COLUMN` | ✅ ย้อนด้วย `RENAME` กลับ | แต่กระทบแอปที่กำลังรันอยู่ทันที (ดู Step 786) |
| `ALTER COLUMN TYPE` | ⚠️ ขึ้นกับทิศทาง | แปลงกลับอาจสูญเสีย precision เช่น NUMERIC → INTEGER |

**แนวทางปฏิบัติ:** สำหรับการเปลี่ยนแปลงที่ทำลายข้อมูล (destructive) เช่น `DROP COLUMN`, `DROP TABLE` ให้แยก migration ออกเป็นสองขั้นเสมอ — ขั้นแรก "หยุดใช้งาน" (deprecate) แล้วรอผ่านไปหลาย deploy cycle ค่อยมี migration แยกต่างหากสำหรับ "ลบจริง" เมื่อมั่นใจว่าไม่มีใครใช้แล้ว

### หลักที่ 3: แยก Migration เล็ก ๆ ทีละขั้น (small, focused migrations)

migration หนึ่งไฟล์ควรทำ**เรื่องเดียว** ไม่ควรรวมหลายการเปลี่ยนแปลงที่ไม่เกี่ยวข้องกันไว้ในไฟล์เดียว

**ไม่ควรทำ (migration ใหญ่เกินไป รวมหลายเรื่อง):**

```sql
-- ❌ 006_big_changes.sql — รวมทุกอย่างไว้ในไฟล์เดียว เสี่ยงสูง
ALTER TABLE customers ADD COLUMN phone TEXT;
ALTER TABLE products ADD COLUMN category TEXT;
ALTER TABLE orders RENAME COLUMN order_status TO status;
ALTER TABLE orders ADD COLUMN shipped_at TIMESTAMPTZ;
CREATE INDEX idx_products_category ON products (category);
```

ปัญหาของการรวมไฟล์เดียว: ถ้า statement ที่ 3 ล้มเหลว (เช่นเพราะ column ถูก view อ้างอิงอยู่) จะไม่รู้ว่า statement 1-2 สำเร็จไปแล้วหรือยัง (ขึ้นกับว่าอยู่ใน transaction เดียวกันหรือไม่) และยากต่อการ review/rollback แยกเรื่อง

**ควรทำ (แยกไฟล์ตามเรื่อง):**

```
006_add_phone_to_customers.sql
007_add_category_to_products.sql
008_rename_orders_status_column.sql
009_add_shipped_at_to_orders.sql
010_add_index_products_category.sql
```

ข้อดีของการแยกเล็ก ๆ:

- **Review ง่ายขึ้น** — reviewer เห็นการเปลี่ยนแปลงทีละเรื่องชัดเจน
- **Deploy แยกได้อิสระ** — ถ้า migration หนึ่งมีปัญหา ไม่กระทบอีกไฟล์
- **Rollback แม่นยำ** — ย้อนเฉพาะไฟล์ที่มีปัญหาได้ ไม่ต้องย้อนทั้งชุด
- **สอดคล้องกับ zero-downtime pattern** — อย่างที่จะเห็นใน Step 785-789 การเปลี่ยนแปลงแบบ zero-downtime หนึ่งเรื่อง มักต้องแตกเป็นหลาย migration ที่ deploy คนละรอบอยู่แล้ว (เช่น "เพิ่มคอลัมน์" ต้องแยกเป็น เพิ่มคอลัมน์ → backfill → เพิ่ม constraint คนละไฟล์ คนละรอบ deploy)

### สรุปหลักการ 3 ข้อ

```
1. Idempotent   : IF NOT EXISTS / IF EXISTS / DO block ตรวจก่อนเสมอ
2. Reversible   : ทุก up มี down คู่กัน ระวัง destructive change
3. Small steps  : 1 migration = 1 เรื่อง ไม่รวมหลายการเปลี่ยนแปลง
```

---

## Step 784: ปัญหาของการ ALTER TABLE บนตารางใหญ่ใน production — ACCESS EXCLUSIVE LOCK

ก่อนจะไปถึง zero-downtime pattern เราต้องเข้าใจก่อนว่า**ทำไมมันถึงจำเป็น** — เพราะคำสั่ง `ALTER TABLE` หลายรูปแบบ ต้องการ **ACCESS EXCLUSIVE LOCK** ซึ่งเป็น lock ที่เข้มงวดที่สุดใน PostgreSQL (ทบทวนเชื่อมโยงกับ Part 019 เรื่อง Lock Types)

### ACCESS EXCLUSIVE LOCK คืออะไร

ระดับ lock ใน PostgreSQL มีตั้งแต่เบาไปหนัก (`ACCESS SHARE`, `ROW SHARE`, ..., `ACCESS EXCLUSIVE`) โดย `ACCESS EXCLUSIVE` คือระดับที่**บล็อกทุกอย่าง** — แม้แต่ `SELECT` ธรรมดาก็ต้องรอ เพราะ `SELECT` ต้องขอ `ACCESS SHARE` lock ซึ่ง**ชนกัน (conflict)** กับ `ACCESS EXCLUSIVE`

คำสั่งที่ต้องใช้ `ACCESS EXCLUSIVE LOCK` ที่พบบ่อย:

| คำสั่ง | ต้อง ACCESS EXCLUSIVE? | เหตุผล |
|---|---|---|
| `ALTER TABLE ... ADD COLUMN col TYPE` (ไม่มี DEFAULT หรือ DEFAULT เป็นค่าคงที่/non-volatile, PG11+) | สั้นมาก (metadata only) แต่ยังคง lock ระดับนี้ตอนแก้ catalog | เปลี่ยนแค่ system catalog ไม่ rewrite ตาราง — เร็ว |
| `ALTER TABLE ... ADD COLUMN col TYPE DEFAULT random_func()` (volatile default) | ใช่ พร้อม **table rewrite** | ต้องคำนวณค่า default ทุกแถว |
| `ALTER TABLE ... ALTER COLUMN TYPE` | ใช่ พร้อม **table rewrite** ทั้งตาราง | ต้องเขียนข้อมูลใหม่ทุกแถวเป็น type ใหม่ |
| `ALTER TABLE ... ADD COLUMN col TYPE NOT NULL` (ไม่มี DEFAULT) | ใช่ พร้อม **table scan** ตรวจทุกแถว | ต้อง validate ว่าไม่มีแถวไหนเป็น NULL |
| `ALTER TABLE ... ADD CONSTRAINT ... CHECK (...)` (ไม่ใส่ NOT VALID) | ใช่ พร้อม **table scan** | ต้อง validate ทุกแถวทันที |
| `CREATE INDEX` (ไม่ใส่ CONCURRENTLY) | ใช่ (`SHARE` lock จริง ๆ แต่บล็อก write ทั้งหมด) | ต้องสแกนทั้งตารางเพื่อสร้าง index |
| `ALTER TABLE ... RENAME COLUMN` | ใช่ แต่เร็วมาก (metadata only) | แต่ query ที่ยังอ้างชื่อเดิมจะ error ทันที |

จุดที่อันตรายที่สุดไม่ใช่แค่ "lock ระดับสูง" อย่างเดียว แต่คือ**ระยะเวลาที่ถือ lock นาน** เมื่อรวมกับตารางขนาดใหญ่ (หลายสิบล้านแถว) การ `ALTER TABLE` ที่ต้อง rewrite ตารางหรือสแกนทั้งตารางอาจใช้เวลาหลายนาทีถึงหลายชั่วโมง — และตลอดเวลานั้น **query อื่นทั้งหมดที่แตะตารางนี้จะค้างรอ**

### สาธิตปัญหา: lock queue

ลองจำลองสถานการณ์ด้วยสอง session

**Session A (รัน migration):**

```sql
BEGIN;
-- คำสั่งนี้ต้องสแกนทั้งตารางเพื่อ validate NOT NULL
-- สมมติ orders มี 50 ล้านแถว อาจใช้เวลาหลายนาที
ALTER TABLE orders ALTER COLUMN order_status SET NOT NULL;
-- ยังไม่ COMMIT ... session ค้างอยู่ตรงนี้
```

**Session B (ผู้ใช้งานทั่วไปพยายาม query):**

```sql
-- แค่ SELECT ธรรมดา ก็ยังต้องรอ เพราะโดน ACCESS EXCLUSIVE lock ของ Session A บล็อก
SELECT order_id, total_amount FROM orders WHERE customer_id = 42;
-- ค้างรอจนกว่า Session A จะ COMMIT หรือ ROLLBACK
```

**Session C (ตรวจสอบว่าใครบล็อกใคร):**

```sql
SELECT
    blocked.pid            AS blocked_pid,
    blocked.query          AS blocked_query,
    blocking.pid            AS blocking_pid,
    blocking.query          AS blocking_query,
    now() - blocked.query_start AS blocked_duration
FROM pg_stat_activity AS blocked
JOIN pg_locks AS bl ON bl.pid = blocked.pid AND NOT bl.granted
JOIN pg_locks AS kl ON kl.locktype = bl.locktype
                    AND kl.database IS NOT DISTINCT FROM bl.database
                    AND kl.relation IS NOT DISTINCT FROM bl.relation
                    AND kl.granted
JOIN pg_stat_activity AS blocking ON blocking.pid = kl.pid
WHERE blocked.pid <> blocking.pid;
```

ผลลัพธ์ในสถานการณ์นี้จะแสดง Session B ถูกบล็อกโดย Session A ชัดเจน และในระบบจริงที่มี traffic สูง สิ่งนี้จะเกิด**เป็นลูกโซ่** — connection pool เต็มไปด้วย query ที่รอ lock, health check ของ load balancer timeout, ระบบถูกมองว่า "ล่ม" ทั้งที่ฐานข้อมูลยังทำงานอยู่ เพียงแต่ค้างรอ lock เดียว

### ทางป้องกัน: lock_timeout และ statement_timeout

หลักปฏิบัติสำคัญ: **ทุก migration ที่รันบน production ต้องตั้ง `lock_timeout` เสมอ** เพื่อไม่ให้ migration ไปค้างรอ lock นานเกินไปจนกลายเป็นตัวบล็อก queue ยาว (เพราะยิ่งรอ lock นาน ยิ่งมี query อื่นมาต่อคิวรอ lock เดียวกันเพิ่มขึ้นเรื่อย ๆ)

```sql
-- ตั้งไว้ต้น session ก่อนรัน migration ที่มีความเสี่ยง
SET lock_timeout = '2s';       -- ถ้าขอ lock แล้วรอเกิน 2 วิ ให้ error ออกมาเลย ไม่ใช่ไปนอนรอ
SET statement_timeout = '30s'; -- กันไม่ให้ statement รันนานเกินคาด

ALTER TABLE orders ALTER COLUMN order_status SET NOT NULL;
```

ถ้า `lock_timeout` ทำงาน จะได้ error แบบนี้แทนที่จะค้างเงียบ ๆ:

```
ERROR:  canceling statement due to lock timeout
```

แนวทางนี้เปลี่ยนปัญหาจาก "ระบบค้างไม่รู้สาเหตุ นานไม่จำกัด" เป็น "migration ล้มเหลวชัดเจน รู้ทันที แก้ไขแล้วรันใหม่ได้" — เป็นจุดเริ่มต้นสำคัญก่อนเข้าสู่ pattern แบบ zero-downtime โดยละเอียดใน Step 785-789

---

## Step 785: Zero-downtime Migration Pattern — เพิ่มคอลัมน์ (เพิ่มแบบ nullable ก่อน → backfill → NOT NULL)

โจทย์: ต้องการเพิ่มคอลัมน์ `shipping_country` ในตาราง `orders` (50 ล้านแถว) และในที่สุดต้องการให้เป็น `NOT NULL` โดยไม่ให้ระบบหยุดทำงาน

### ทำไม "เพิ่มคอลัมน์ NOT NULL DEFAULT ค่าคงที่ทันที" ถึงอันตรายในตารางใหญ่มาก (บางกรณี)

ตั้งแต่ PostgreSQL 11 เป็นต้นมา การ `ADD COLUMN ... DEFAULT <ค่าคงที่ที่ไม่ volatile>` **ไม่ต้อง rewrite ตารางทั้งหมด** อีกต่อไป (เก็บ default ไว้ใน catalog แล้วคืนค่านั้นแบบ virtual สำหรับแถวเก่า) ทำให้เร็วมากแม้ตารางใหญ่:

```sql
-- PostgreSQL 11+: เร็วมาก ไม่ rewrite ตาราง เพราะ DEFAULT เป็นค่าคงที่
ALTER TABLE orders ADD COLUMN shipping_country TEXT DEFAULT 'TH';
```

**แต่** ถ้าเพิ่มพร้อม `NOT NULL` โดยไม่มี default (หรือมี default ที่เป็น volatile function เช่น `now()`, `random()`) PostgreSQL ยังต้อง**สแกนทั้งตาราง** เพื่อ validate หรือคำนวณค่าให้ทุกแถว ซึ่งใช้เวลานานและถือ `ACCESS EXCLUSIVE LOCK` ตลอด:

```sql
-- ❌ อันตรายกับตารางใหญ่: ต้องสแกนทุกแถวเพื่อคำนวณ default และ validate NOT NULL
ALTER TABLE orders ADD COLUMN tracking_code TEXT NOT NULL DEFAULT gen_random_uuid()::text;
```

และในหลายสถานการณ์จริง คอลัมน์ใหม่ต้อง backfill ด้วย**ค่าที่คำนวณจากข้อมูลเดิมของแต่ละแถว** (ไม่ใช่ค่าคงที่เดียวกันหมด) เช่น `shipping_country` ต้องได้จากที่อยู่ของลูกค้าแต่ละคน ซึ่งไม่มีทางทำในคำสั่งเดียวแบบ metadata-only ได้เลย จึงต้องใช้ pattern สามขั้นตอนต่อไปนี้

### Pattern: 3 ขั้นตอน (3 deploy แยกกัน)

```
ขั้นตอนที่ 1: เพิ่มคอลัมน์แบบ nullable (metadata-only เร็วมาก)
ขั้นตอนที่ 2: Backfill ข้อมูลย้อนหลังเป็น batch (ไม่ล็อกนาน)
ขั้นตอนที่ 3: เพิ่ม NOT NULL constraint แบบปลอดภัย (ดูรายละเอียดเต็มใน Step 789)
```

#### ขั้นตอนที่ 1 — เพิ่มคอลัมน์แบบ nullable

Migration ไฟล์ `020_add_shipping_country_nullable.sql` — deploy พร้อมกับ**แอปเวอร์ชันใหม่ที่เขียนค่าลงคอลัมน์นี้สำหรับ order ใหม่ทุกตัว** (dual-write เริ่มจากจุดนี้):

```sql
-- ก่อน: orders ไม่มีคอลัมน์ shipping_country
-- ระหว่าง: เพิ่มคอลัมน์ nullable ไม่มี DEFAULT บังคับ -> metadata-only เร็วมาก ไม่ scan ตาราง
ALTER TABLE orders ADD COLUMN IF NOT EXISTS shipping_country TEXT;

-- หลัง: คอลัมน์มีอยู่แล้ว ค่าเก่าทุกแถวเป็น NULL รออยู่ แถวใหม่จากนี้แอปตัวใหม่จะเขียนค่าให้
```

ตรวจสอบสถานะหลังรัน:

```sql
SELECT column_name, is_nullable, column_default
FROM information_schema.columns
WHERE table_name = 'orders' AND column_name = 'shipping_country';
--  column_name      | is_nullable | column_default
--  shipping_country  | YES         | NULL
```

**สำคัญ:** จุดนี้ต้อง deploy โค้ดแอปพลิเคชันเวอร์ชันใหม่ที่เขียน `shipping_country` ทุกครั้งที่สร้าง order ใหม่**ก่อน**จะไปขั้นตอนถัดไป เพื่อไม่ให้มีแถวใหม่เพิ่มเข้ามาเป็น NULL ระหว่างที่กำลัง backfill

#### ขั้นตอนที่ 2 — Backfill เป็น batch

ห้าม backfill ด้วย `UPDATE` ตัวเดียวครอบคลุมทั้งตาราง เพราะจะสร้าง transaction ยาวที่ล็อกแถวจำนวนมากพร้อมกัน และเสี่ยง bloat จาก dead tuples จำนวนมหาศาลรวดเดียว:

```sql
-- ❌ ห้ามทำกับตารางใหญ่: UPDATE เดียวกระทบ 50 ล้านแถว
UPDATE orders SET shipping_country = 'TH' WHERE shipping_country IS NULL;
```

ให้ backfill เป็น batch เล็ก ๆ แทน โดยใช้ primary key range หรือ `LIMIT` วนลูป (เขียนเป็น script ฝั่ง application หรือ `DO` block ที่ commit เป็นช่วง ๆ):

```sql
-- ตัวอย่าง backfill แบบ batch ด้วย DO block + LOOP
-- (ในระบบจริงมักเขียนเป็น script ภายนอกที่ควบคุม sleep/commit ระหว่าง batch ได้ดีกว่า)
DO $$
DECLARE
    batch_size   CONSTANT INTEGER := 5000;
    rows_updated INTEGER;
BEGIN
    LOOP
        UPDATE orders
        SET shipping_country = 'TH'
        WHERE order_id IN (
            SELECT order_id FROM orders
            WHERE shipping_country IS NULL
            ORDER BY order_id
            LIMIT batch_size
        );

        GET DIAGNOSTICS rows_updated = ROW_COUNT;
        EXIT WHEN rows_updated = 0;

        RAISE NOTICE 'Backfilled % rows', rows_updated;
        COMMIT;                       -- ต้องรันใน procedure/DO block ที่รองรับ COMMIT (PG11+ ใน procedure)
        PERFORM pg_sleep(0.1);        -- เว้นจังหวะเล็กน้อย ลดแรงกดดันต่อ replication/IO
    END LOOP;
END $$;
```

> **หมายเหตุ:** การ `COMMIT` ภายใน `DO` block ธรรมดาทำไม่ได้ — ต้องใช้ **stored procedure** (`CREATE PROCEDURE`) ที่เรียกผ่าน `CALL` จึงจะ `COMMIT` ระหว่างลูปได้จริง ตัวอย่างที่ถูกต้องตามหลัก PostgreSQL 16/17:

```sql
CREATE OR REPLACE PROCEDURE backfill_shipping_country(batch_size INTEGER DEFAULT 5000)
LANGUAGE plpgsql
AS $$
DECLARE
    rows_updated INTEGER;
BEGIN
    LOOP
        UPDATE orders
        SET shipping_country = 'TH'
        WHERE order_id IN (
            SELECT order_id FROM orders
            WHERE shipping_country IS NULL
            ORDER BY order_id
            LIMIT batch_size
        );

        GET DIAGNOSTICS rows_updated = ROW_COUNT;
        EXIT WHEN rows_updated = 0;

        RAISE NOTICE 'Backfilled % rows at %', rows_updated, clock_timestamp();
        COMMIT;                -- commit ทีละ batch จริง ปลดล็อกแถวอื่นให้ query ปกติแทรกได้
        PERFORM pg_sleep(0.1);
    END LOOP;
END;
$$;

-- เรียกใช้งาน (รันนอกเวลาเร่งด่วน หรือคุมความเร็วให้เหมาะกับ traffic)
CALL backfill_shipping_country(5000);
```

ข้อดีของแนวทาง batch + COMMIT ทีละช่วง:

- แต่ละ `UPDATE` ล็อกแค่ 5,000 แถว ไม่ใช่ 50 ล้านแถว
- ระหว่าง batch คำสั่งอื่นสามารถเข้าถึงตารางได้ตามปกติ
- ถ้า backfill ถูกขัดจังหวะกลางทาง (เช่น deploy ใหม่ทับ) รันซ้ำได้ทันทีเพราะ `WHERE shipping_country IS NULL` ทำให้ idempotent อยู่แล้ว — แถวที่ backfill ไปแล้วจะไม่ถูกแตะซ้ำ

ตรวจสอบความคืบหน้าระหว่างรัน:

```sql
SELECT
    count(*) FILTER (WHERE shipping_country IS NULL) AS remaining_null,
    count(*) AS total_rows
FROM orders;
```

#### ขั้นตอนที่ 3 — เพิ่ม NOT NULL อย่างปลอดภัย

เมื่อ `remaining_null = 0` แล้ว (backfill เสร็จสมบูรณ์ และแอปเขียนค่าให้ทุกแถวใหม่มาสักระยะแล้ว) จึงค่อยบังคับ `NOT NULL` — รายละเอียดเทคนิค `NOT VALID` + `VALIDATE CONSTRAINT` อธิบายเต็มใน **Step 789** แต่สรุปสั้น ๆ ที่นี่:

```sql
-- ก่อน: shipping_country เป็น nullable ค่าครบทุกแถวแล้ว (validated ด้วย query ข้างบน)
ALTER TABLE orders
    ADD CONSTRAINT chk_orders_shipping_country_not_null
    CHECK (shipping_country IS NOT NULL) NOT VALID;   -- เพิ่ม constraint ทันที ไม่ scan (เร็ว)

ALTER TABLE orders
    VALIDATE CONSTRAINT chk_orders_shipping_country_not_null; -- scan ทีหลัง แต่ใช้ lock เบากว่า

-- PostgreSQL 12+: เมื่อมี CHECK constraint ที่ validate แล้วว่าห้าม NULL
-- การ SET NOT NULL จะ "ฉลาดพอ" ที่จะข้าม full table scan ซ้ำ เพราะรู้ว่า constraint พิสูจน์ไว้แล้ว
ALTER TABLE orders ALTER COLUMN shipping_country SET NOT NULL;

-- ทำความสะอาด: ลบ CHECK constraint ที่ซ้ำซ้อนกับ NOT NULL แล้ว (ไม่บังคับ แต่ช่วยให้ schema สะอาด)
ALTER TABLE orders DROP CONSTRAINT chk_orders_shipping_country_not_null;
```

### สรุป before/during/after ของ pattern นี้

| ช่วงเวลา | สถานะคอลัมน์ | สถานะแอป | risk |
|---|---|---|---|
| **ก่อน** | ไม่มีคอลัมน์ | แอปเวอร์ชันเก่า | - |
| **หลังขั้นตอนที่ 1** | มีคอลัมน์ nullable | แอปใหม่เขียนค่าให้ order ใหม่ | ต่ำมาก (metadata-only) |
| **ระหว่างขั้นตอนที่ 2** | บาง row เป็น NULL บาง row มีค่า | แอปใหม่ dual-write | ต่ำ (lock สั้นเป็น batch) |
| **หลังขั้นตอนที่ 3** | NOT NULL บังคับแล้ว | แอปใหม่ทำงานปกติ | ต่ำ (ใช้ NOT VALID trick) |

---

## Step 786: Zero-downtime Migration Pattern — เปลี่ยนชื่อคอลัมน์ (ใช้ view หรือ dual-write แทนการ RENAME ตรงๆ)

โจทย์: ต้องการเปลี่ยนชื่อคอลัมน์ `order_status` เป็น `status` ในตาราง `orders` แต่มีแอปพลิเคชันหลายตัว (web app, mobile API, reporting job) ที่อ้างชื่อคอลัมน์เดิมอยู่ และ**ไม่สามารถ deploy ทุกตัวพร้อมกันในวินาทีเดียวกันได้**

### ทำไม RENAME COLUMN ตรง ๆ ถึงอันตราย

`ALTER TABLE ... RENAME COLUMN` เป็น metadata-only operation เร็วมาก (ไม่ scan ตาราง) แต่ปัญหาไม่ได้อยู่ที่ความเร็ว — ปัญหาคือ**ทันทีที่ COMMIT** ชื่อคอลัมน์เดิมจะหายไปทันที query ใด ๆ ที่ยังอ้าง `order_status` (จากแอปเวอร์ชันเก่าที่ยัง deploy ไม่ครบ หรือ replica lag) จะ error ทันที:

```sql
-- ❌ อันตราย: ถ้ามีแอปเวอร์ชันเก่าที่ยังอ้าง order_status รันอยู่แม้เพียง instance เดียว
ALTER TABLE orders RENAME COLUMN order_status TO status;
-- แอปเก่า: ERROR:  column "order_status" of relation "orders" does not exist
```

ในระบบที่ deploy แบบ **rolling deployment** (ทยอย deploy ทีละ instance ไม่ใช่ deploy พร้อมกันหมด) จะมีช่วงเวลาที่ instance เก่ากับใหม่รันพร้อมกัน — การ RENAME ตรง ๆ จึงทำให้ instance เก่า error ทันที นี่คือ downtime แบบหนึ่งแม้ระบบโดยรวมยัง "ทำงาน" อยู่

### แนวทางที่ 1: ใช้ view เป็นสะพานเชื่อม (compatibility view)

```sql
-- ก่อน: ตาราง orders มีคอลัมน์ order_status ตรง ๆ
-- (แอปทุกตัว query ตาราง orders ตรง ๆ)

-- ระหว่าง ขั้นที่ 1: RENAME คอลัมน์จริงในตาราง แล้วสร้าง view ชื่อเดิมทับไว้
BEGIN;

ALTER TABLE orders RENAME COLUMN order_status TO status;

-- แต่ view ชื่อ orders_compat ทำหน้าที่เป็น "หน้ากาก" ให้แอปเก่ายัง query ชื่อเดิมได้
-- หมายเหตุ: ไม่สามารถสร้าง view ชื่อ orders ทับตารางจริงชื่อ orders ได้ในชื่อเดียวกัน
-- แนวทางจริงจึงมักใช้อีกชื่อ เช่น orders_legacy_view แล้วให้แอปเก่า "ชี้" มาที่ view นี้แทน
CREATE VIEW orders_legacy_view AS
SELECT
    order_id,
    customer_id,
    status AS order_status,   -- คอลัมน์ใหม่ชื่อ status แต่ expose เป็นชื่อเก่าให้แอปเก่า
    total_amount,
    created_at
FROM orders;

COMMIT;
```

> **ข้อจำกัดสำคัญที่ต้องรู้:** PostgreSQL **ไม่อนุญาตให้ view ชื่อซ้ำกับ table** ในสคีมาเดียวกัน ดังนั้นแนวทาง "view สวมชื่อเดิม" ในทางปฏิบัติมักทำสลับด้าน คือ**เปลี่ยนชื่อตารางจริงออกไปก่อน** แล้วสร้าง view ที่ใช้**ชื่อเดิมของตาราง** มาทับ ตัวอย่างที่ถูกต้องและรันได้จริง:

```sql
BEGIN;

-- 1) เปลี่ยนชื่อตารางจริงออกจากชื่อเดิมชั่วคราว
ALTER TABLE orders RENAME TO orders_base;

-- 2) เปลี่ยนชื่อคอลัมน์ในตารางจริง (orders_base) ตามที่ต้องการ
ALTER TABLE orders_base RENAME COLUMN order_status TO status;

-- 3) สร้าง view ชื่อ "orders" (ชื่อเดิมของตาราง) ให้ expose คอลัมน์ชื่อเก่า order_status
--    เพื่อให้แอปเวอร์ชันเก่า (ที่ query "orders" และอ้าง order_status) ยังทำงานได้ไม่มี downtime
CREATE VIEW orders AS
SELECT
    order_id,
    customer_id,
    status AS order_status,
    total_amount,
    created_at
FROM orders_base;

COMMIT;
```

จุดสำคัญ: view ธรรมดาแบบ `SELECT` เดียวไม่มี join/aggregate จะเป็น **automatically updatable view** ใน PostgreSQL หมายความว่าแอปเก่าที่ยังทำ `INSERT`/`UPDATE`/`DELETE` ผ่านชื่อ `orders` (view) จะยังคงทำงานได้ปกติผ่านกลไก updatable view ของ PostgreSQL เอง โดยไม่ต้องเขียน `INSTEAD OF` trigger เพิ่ม — ตราบใดที่ view นั้น map คอลัมน์แบบ 1:1 ไม่มีการ aggregate หรือ DISTINCT

ทดสอบว่า view ยัง insert ได้:

```sql
INSERT INTO orders (customer_id, order_status, total_amount)
VALUES (1, 'pending', 1200.00);

SELECT * FROM orders_base ORDER BY order_id DESC LIMIT 1;
--  จะเห็นว่าแถวใหม่ถูกเขียนเข้า orders_base จริง ผ่าน view orders ที่ map order_status -> status
```

เมื่อทุก instance ของแอปทั้งหมด deploy เป็นเวอร์ชันใหม่ (อ้างชื่อ `status` ผ่านตาราง `orders_base` โดยตรง หรือปรับ config ให้ชี้ตารางใหม่) เรียบร้อยแล้ว ค่อยลบ view สะพานทิ้งในภายหลัง:

```sql
-- ทำความสะอาดหลังยืนยันว่าไม่มีแอปตัวไหนอ้าง view/ชื่อคอลัมน์เก่าแล้ว (เช่น เฝ้าดู log/pg_stat_activity สัก 1-2 สัปดาห์)
DROP VIEW IF EXISTS orders;
ALTER TABLE orders_base RENAME TO orders;
```

### แนวทางที่ 2: Dual-write ด้วยคอลัมน์คู่ขนาน (สำหรับกรณีซับซ้อนกว่า view)

บางกรณี view เพียงอย่างเดียวไม่พอ (เช่น ต้องการเปลี่ยนทั้งชื่อและความหมายไปพร้อมกัน หรือมี ORM ที่ generate SQL เองไม่ผ่าน view ได้ง่าย) จึงใช้แนวทาง **dual-write column** แทน:

```sql
-- ขั้นที่ 1: เพิ่มคอลัมน์ใหม่ชื่อ status คู่ขนานกับ order_status เดิม
ALTER TABLE orders ADD COLUMN IF NOT EXISTS status TEXT;

-- ขั้นที่ 2: สร้าง trigger ให้เขียนพร้อมกันทั้งสองคอลัมน์ ระหว่างที่แอปยัง deploy ไม่ครบ
CREATE OR REPLACE FUNCTION sync_order_status_columns()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        NEW.status := COALESCE(NEW.status, NEW.order_status);
        NEW.order_status := COALESCE(NEW.order_status, NEW.status);
    ELSIF TG_OP = 'UPDATE' THEN
        IF NEW.order_status IS DISTINCT FROM OLD.order_status THEN
            NEW.status := NEW.order_status;              -- แอปเก่าเขียน order_status -> sync ไป status
        ELSIF NEW.status IS DISTINCT FROM OLD.status THEN
            NEW.order_status := NEW.status;               -- แอปใหม่เขียน status -> sync ไป order_status
        END IF;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_sync_order_status
BEFORE INSERT OR UPDATE ON orders
FOR EACH ROW EXECUTE FUNCTION sync_order_status_columns();

-- ขั้นที่ 3: backfill คอลัมน์ status จากของเดิมที่มีอยู่ก่อน trigger ถูกสร้าง (เป็น batch แบบ Step 785)
UPDATE orders SET status = order_status WHERE status IS NULL;

-- ขั้นที่ 4 (deploy คนละรอบ หลังแอปทุกตัวย้ายไปใช้ status แล้ว):
DROP TRIGGER IF EXISTS trg_sync_order_status ON orders;
DROP FUNCTION IF EXISTS sync_order_status_columns();
ALTER TABLE orders DROP COLUMN order_status;
```

### เปรียบเทียบสองแนวทาง

| แนวทาง | เหมาะกับ | ข้อดี | ข้อเสีย |
|---|---|---|---|
| **Compatibility view** | ORM/แอปที่ query ผ่าน table/view name ปกติ ไม่มี logic ซับซ้อน | ตั้งค่าเร็ว ไม่ต้องแก้ schema ซ้ำ | จำกัดกับ view ที่ updatable ได้ (1:1 mapping) |
| **Dual-write + trigger** | ต้องเปลี่ยนความหมายคอลัมน์ไปด้วย หรือ ORM generate SQL ตรงเข้าตารางเท่านั้น | ยืดหยุ่นสูงกว่า | ซับซ้อนกว่า มี trigger overhead เล็กน้อยทุก write |

ไม่ว่าจะเลือกแนวทางใด หลักการร่วมคือ: **อย่าลบ/เปลี่ยนชื่อ ในจังหวะเดียวกับที่ต้อง deploy แอปใหม่พร้อมกันเป๊ะ** ให้มีช่วง "สะพานเชื่อม" ที่ทั้งชื่อเก่าและชื่อใหม่ใช้งานได้พร้อมกันเสมอ

---

## Step 787: Zero-downtime Migration Pattern — เปลี่ยน data type (สร้างคอลัมน์ใหม่ dual-write แล้วค่อย migrate/สลับ)

โจทย์: ตาราง `products` เก็บ `price` เป็น `NUMERIC(12,2)` แต่ทีมต้องการเปลี่ยนไปเป็น `NUMERIC(14,4)` เพื่อรองรับสกุลเงินที่มีทศนิยมละเอียดกว่า (หรือกรณีทั่วไปกว่าคือเปลี่ยนจาก `INTEGER` เป็น `BIGINT`, จาก `TEXT` เป็น `VARCHAR(50)` ที่มี constraint ความยาว, หรือจาก `TEXT` เป็น `ENUM`)

### ทำไม ALTER COLUMN TYPE ตรง ๆ ถึงอันตรายกับตารางใหญ่

```sql
-- ❌ อันตรายกับตารางใหญ่มาก: PostgreSQL ต้อง rewrite ตารางทั้งหมดใหม่
--    ถือ ACCESS EXCLUSIVE LOCK ตลอดเวลาที่ rewrite (อาจนานหลายนาทีถึงชั่วโมงถ้าตารางใหญ่มาก)
ALTER TABLE products ALTER COLUMN price TYPE NUMERIC(14,4);
```

แม้ในบางกรณีพิเศษ (เช่นเปลี่ยนแค่ precision/scale ของ `NUMERIC` แบบกว้างขึ้น หรือ `VARCHAR(n)` ขยายความยาว) PostgreSQL รุ่นใหม่จะฉลาดพอที่จะรู้ว่าไม่ต้อง rewrite เพราะ binary representation เข้ากันได้ (compatible cast) แต่กรณีทั่วไป เช่น เปลี่ยน type ข้ามชนิดกันจริง ๆ (`TEXT` → `INTEGER`, `INTEGER` → `BIGINT` ไม่นับเพราะจริง ๆ ก็ยัง rewrite เสมอ) ยังคง**ต้อง rewrite ทั้งตาราง**เสมอ และ**ล็อกอ่าน/เขียนทั้งหมดระหว่างนั้น**

### Pattern: New column + dual-write + backfill + switch

```
ขั้นตอนที่ 1: เพิ่มคอลัมน์ใหม่ type ใหม่ (nullable)
ขั้นตอนที่ 2: Dual-write ผ่าน trigger (หรือ application-level) ให้ทุก write ไปที่ทั้งสองคอลัมน์
ขั้นตอนที่ 3: Backfill ข้อมูลเก่าเป็น batch
ขั้นตอนที่ 4: สลับแอปให้อ่านจากคอลัมน์ใหม่ (read cut-over)
ขั้นตอนที่ 5: ลบคอลัมน์เก่าและ trigger (หลังมั่นใจว่าไม่มีใครใช้แล้ว)
```

#### ขั้นตอนที่ 1 — เพิ่มคอลัมน์ใหม่

```sql
-- ก่อน: products.price เป็น NUMERIC(12,2)
ALTER TABLE products ADD COLUMN IF NOT EXISTS price_v2 NUMERIC(14,4);
```

#### ขั้นตอนที่ 2 — Dual-write ด้วย trigger

```sql
CREATE OR REPLACE FUNCTION sync_products_price_v2()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        IF NEW.price_v2 IS NULL AND NEW.price IS NOT NULL THEN
            NEW.price_v2 := NEW.price::NUMERIC(14,4);
        END IF;
    ELSIF TG_OP = 'UPDATE' THEN
        -- ถ้าแอปเก่าเขียน price -> sync ไป price_v2
        IF NEW.price IS DISTINCT FROM OLD.price THEN
            NEW.price_v2 := NEW.price::NUMERIC(14,4);
        -- ถ้าแอปใหม่เขียน price_v2 -> sync กลับไป price (คง compat เผื่อยัง rollback)
        ELSIF NEW.price_v2 IS DISTINCT FROM OLD.price_v2 THEN
            NEW.price := NEW.price_v2::NUMERIC(12,2);
        END IF;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_sync_products_price_v2
BEFORE INSERT OR UPDATE ON products
FOR EACH ROW EXECUTE FUNCTION sync_products_price_v2();
```

ทดสอบว่า trigger sync ถูกต้อง:

```sql
UPDATE products SET price = 2100.00 WHERE sku = 'SKU-001';
SELECT sku, price, price_v2 FROM products WHERE sku = 'SKU-001';
--  sku      | price   | price_v2
--  SKU-001  | 2100.00 | 2100.0000
```

#### ขั้นตอนที่ 3 — Backfill เป็น batch (เหมือนหลักการใน Step 785)

```sql
CREATE OR REPLACE PROCEDURE backfill_products_price_v2(batch_size INTEGER DEFAULT 2000)
LANGUAGE plpgsql
AS $$
DECLARE
    rows_updated INTEGER;
BEGIN
    LOOP
        UPDATE products
        SET price_v2 = price::NUMERIC(14,4)
        WHERE product_id IN (
            SELECT product_id FROM products
            WHERE price_v2 IS NULL
            ORDER BY product_id
            LIMIT batch_size
        );

        GET DIAGNOSTICS rows_updated = ROW_COUNT;
        EXIT WHEN rows_updated = 0;

        RAISE NOTICE 'Backfilled % product rows', rows_updated;
        COMMIT;
        PERFORM pg_sleep(0.1);
    END LOOP;
END;
$$;

CALL backfill_products_price_v2(2000);

-- ตรวจสอบว่า backfill ครบ
SELECT count(*) FILTER (WHERE price_v2 IS NULL) AS remaining FROM products;
```

#### ขั้นตอนที่ 4 — สลับแอปให้อ่านคอลัมน์ใหม่ (read cut-over)

จุดนี้คือการ deploy แอปพลิเคชันเวอร์ชันใหม่ที่**อ่าน** `price_v2` แทน `price` (แต่ trigger ยัง dual-write อยู่เผื่อต้อง rollback แอปกลับไปเวอร์ชันเก่าฉุกเฉิน) ปล่อยให้รันแบบนี้สักระยะ (เช่น 1-2 สัปดาห์) เพื่อมั่นใจว่าไม่มีปัญหา แล้วค่อยเข้าขั้นตอนสุดท้าย

#### ขั้นตอนที่ 5 — สลับชื่อคอลัมน์และลบของเก่า (cleanup)

```sql
BEGIN;

-- ปลด trigger dual-write ออกก่อน (ไม่ต้อง sync แล้ว)
DROP TRIGGER IF EXISTS trg_sync_products_price_v2 ON products;
DROP FUNCTION IF EXISTS sync_products_price_v2();

-- ลบคอลัมน์เก่า แล้วเปลี่ยนชื่อคอลัมน์ใหม่ให้เป็นชื่อมาตรฐาน (metadata-only ทั้งคู่ เร็วมาก)
ALTER TABLE products DROP COLUMN price;
ALTER TABLE products RENAME COLUMN price_v2 TO price;

-- เพิ่ม NOT NULL ให้คอลัมน์ใหม่ด้วยเทคนิคปลอดภัยจาก Step 789 (ถ้าต้องการบังคับ)
ALTER TABLE products
    ADD CONSTRAINT chk_products_price_not_null CHECK (price IS NOT NULL) NOT VALID;
ALTER TABLE products VALIDATE CONSTRAINT chk_products_price_not_null;
ALTER TABLE products ALTER COLUMN price SET NOT NULL;
ALTER TABLE products DROP CONSTRAINT chk_products_price_not_null;

COMMIT;
```

### สรุป before/during/after

| ช่วง | โครงสร้างตาราง | ความเสี่ยง |
|---|---|---|
| ก่อน | `price NUMERIC(12,2)` เท่านั้น | - |
| หลังขั้น 1-2 | มีทั้ง `price` และ `price_v2` sync กันด้วย trigger | ต่ำ (write overhead เล็กน้อยจาก trigger) |
| หลังขั้น 3 | ข้อมูลครบทั้งสองคอลัมน์ | ต่ำ (backfill เป็น batch) |
| หลังขั้น 4 | แอปอ่าน `price_v2` แต่ schema ยังมีสองคอลัมน์ | ต่ำมาก — rollback แอปได้ทันทีถ้ามีปัญหา |
| หลังขั้น 5 | เหลือคอลัมน์เดียวชื่อ `price` type ใหม่ | rollback ยากขึ้น (ต้อง migration ย้อนใหม่) |

> ข้อสังเกตสำคัญ: pattern เปลี่ยน data type นี้ **เหมือนกับ pattern เปลี่ยนชื่อคอลัมน์ใน Step 786 มาก** เพราะทั้งคู่ใช้หลัก "dual-write ชั่วคราว แล้วค่อย cut-over" — นี่คือแก่นของ zero-downtime migration แทบทุกรูปแบบ: **อย่าเปลี่ยนของเดิมทันที ให้สร้างของใหม่คู่ขนานก่อน แล้วค่อยย้ายทีละขั้น**

---

## Step 788: Zero-downtime Migration Pattern — เพิ่ม index บนตารางใหญ่ (CREATE INDEX CONCURRENTLY)

โจทย์: ต้องการเพิ่ม index บนคอลัมน์ `orders.customer_id` เพื่อเร่งความเร็ว query "ดูประวัติคำสั่งซื้อของลูกค้า" แต่ตาราง `orders` มีหลายสิบล้านแถวและรับ traffic ต่อเนื่อง (ทบทวนเชื่อมโยงกับ Part 041 เรื่อง Index Strategy และ Part 054 เรื่อง Query Performance Tuning)

### ปัญหาของ CREATE INDEX แบบปกติ

```sql
-- ❌ อันตรายกับตารางใหญ่ที่รับ traffic ต่อเนื่อง
CREATE INDEX idx_orders_customer_id ON orders (customer_id);
```

คำสั่งนี้ใช้ `SHARE` lock บนตาราง ซึ่งฟังดูไม่รุนแรงเท่า `ACCESS EXCLUSIVE` แต่ในทางปฏิบัติ **`SHARE` lock บล็อกคำสั่ง `INSERT`, `UPDATE`, `DELETE` ทั้งหมด** (อนุญาตแค่ `SELECT`) และเนื่องจากต้องสแกนทั้งตารางเพื่อสร้าง index การล็อกนี้จะคงอยู่ตลอดเวลาที่สร้าง index ซึ่งอาจนานหลายนาทีถึงหลายชั่วโมงสำหรับตารางขนาดใหญ่ — ระหว่างนั้น**order ใหม่จะสร้างไม่ได้เลย**

### ทางออก: CREATE INDEX CONCURRENTLY

```sql
CREATE INDEX CONCURRENTLY idx_orders_customer_id ON orders (customer_id);
```

`CONCURRENTLY` ทำงานต่างจากปกติคือ:

- สแกนตารางเป็น**สองรอบ**แทนที่จะเป็นรอบเดียว (รอบแรกสร้าง index เบื้องต้น รอบสองตรวจสอบแถวที่เปลี่ยนระหว่างรอบแรก) ทำให้**ใช้เวลานานกว่า** CREATE INDEX ปกติ
- ระหว่างสร้าง index **ไม่ล็อก write** — `INSERT`/`UPDATE`/`DELETE` ยังทำงานได้ตามปกติตลอดกระบวนการ
- แลกมาด้วยข้อจำกัดสำคัญ: **ไม่สามารถรันภายใน transaction block ได้** (`BEGIN ... CREATE INDEX CONCURRENTLY ... COMMIT` จะ error)

```
ERROR:  CREATE INDEX CONCURRENTLY cannot run inside a transaction block
```

นี่คือเหตุผลที่เครื่องมือ migration หลายตัว (เช่น Flyway) ต้อง**ปิด wrap-in-transaction เฉพาะไฟล์นี้** เช่น Flyway ใช้ syntax พิเศษในไฟล์:

```sql
-- V21__add_index_orders_customer_id.sql
-- Flyway: ปิดการ wrap ด้วย transaction สำหรับไฟล์นี้ (ต้องตั้งค่า flyway.executeInTransaction=false
-- หรือใช้ comment พิเศษขึ้นกับเวอร์ชัน Flyway)
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_orders_customer_id ON orders (customer_id);
```

### ความเสี่ยงที่ต้องระวัง: INVALID index เมื่อ CONCURRENTLY ล้มเหลว

ถ้า `CREATE INDEX CONCURRENTLY` ล้มเหลวกลางทาง (เช่น ถูก cancel, database crash, หรือมี deadlock กับ transaction อื่น) จะเหลือ index ที่สร้างไม่สมบูรณ์ค้างอยู่ในสถานะ **`INVALID`** — index นี้จะไม่ถูกใช้งานโดย query planner แต่ยังกิน storage และยังถูก maintain (ช้าลงเปล่า ๆ)

ตรวจสอบ index ที่ invalid:

```sql
SELECT
    schemaname,
    indexrelid::regclass AS index_name,
    indrelid::regclass  AS table_name,
    indisvalid
FROM pg_index
WHERE indisvalid = false;
```

วิธีแก้: ต้องลบ index ที่ invalid ทิ้งก่อน แล้วสร้างใหม่ (ใช้ `CONCURRENTLY` เช่นกันตอนลบ เพื่อไม่ล็อกตาราง):

```sql
DROP INDEX CONCURRENTLY IF EXISTS idx_orders_customer_id;

-- สร้างใหม่อีกครั้ง
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_orders_customer_id ON orders (customer_id);
```

### Checklist การใช้ CREATE INDEX CONCURRENTLY ใน migration

```sql
-- 1) ตรวจสอบก่อนว่ามี index ชื่อนี้ค้างเป็น INVALID จากความพยายามครั้งก่อนหรือไม่
SELECT indexrelid::regclass, indisvalid
FROM pg_index
WHERE indexrelid = 'idx_orders_customer_id'::regclass;

-- 2) ถ้ามีและ invalid ให้ DROP CONCURRENTLY ก่อน
DROP INDEX CONCURRENTLY IF EXISTS idx_orders_customer_id;

-- 3) สร้างใหม่ด้วย CONCURRENTLY + IF NOT EXISTS (idempotent ตาม Step 783)
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_orders_customer_id ON orders (customer_id);

-- 4) ตรวจสอบว่าสร้างสำเร็จและ valid
SELECT indexrelid::regclass, indisvalid
FROM pg_index
WHERE indexrelid = 'idx_orders_customer_id'::regclass;
```

### ตัวอย่างเพิ่มเติม: index หลายคอลัมน์และ partial index บนตารางใหญ่

```sql
-- composite index สำหรับ query ที่กรองด้วยหลายเงื่อนไข
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_orders_customer_status
    ON orders (customer_id, order_status);

-- partial index เฉพาะออเดอร์ที่ยัง pending (เล็กกว่า full index มาก เหมาะกับ query แดชบอร์ด)
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_orders_pending_only
    ON orders (created_at)
    WHERE order_status = 'pending';
```

**สรุป:** ทุกครั้งที่ต้องเพิ่ม index บนตารางที่มีขนาดใหญ่พอที่จะทำให้ `CREATE INDEX` ปกติใช้เวลานานเกินไม่กี่วินาที ให้ใช้ `CREATE INDEX CONCURRENTLY` เป็นค่าเริ่มต้นเสมอใน production migration และเตรียม script ตรวจ/ลบ `INVALID` index ไว้เป็นส่วนหนึ่งของ migration pipeline

---

## Step 789: Zero-downtime Migration Pattern — เพิ่ม NOT NULL constraint (ADD CONSTRAINT NOT VALID → VALIDATE)

โจทย์: ต้องการบังคับให้ `orders.total_amount` ห้ามเป็น NULL (สมมติว่าปัจจุบันเป็น nullable และมีข้อมูลอยู่แล้วครบทุกแถว) โดยไม่ล็อกตารางนานเกินไป (ทบทวนเชื่อมโยงกับ Part 018 เรื่อง Constraints)

### ทำไม SET NOT NULL ตรง ๆ ถึงมีปัญหา

```sql
-- ก่อน PostgreSQL 12 หรือในกรณีทั่วไป: ต้องสแกนทั้งตารางเพื่อ validate ว่าไม่มี NULL เลย
-- ถือ ACCESS EXCLUSIVE LOCK ตลอดการสแกน
ALTER TABLE orders ALTER COLUMN total_amount SET NOT NULL;
```

ถ้าตารางมี 50 ล้านแถว การสแกนนี้อาจใช้เวลาหลายนาที และตลอดเวลานั้น**ทุก query ที่แตะตาราง `orders` จะค้าง** (ตามที่สาธิตไปแล้วใน Step 784)

### ทางออก: ใช้ CHECK constraint แบบ NOT VALID ก่อน แล้วค่อย VALIDATE

หลักการคือแยกงาน "ประกาศกฎ" กับ "ตรวจสอบข้อมูลทั้งหมดตามกฎ" ออกจากกันเป็นสองคำสั่ง:

```sql
-- ขั้นตอนที่ 1: เพิ่ม CHECK constraint พร้อม NOT VALID
-- คำสั่งนี้เร็วมาก (แค่บันทึกกฎลง catalog) ใช้ ACCESS EXCLUSIVE LOCK แต่แค่เสี้ยววินาที
-- เพราะไม่ scan ข้อมูลเดิมเลย -- กฎนี้จะบังคับใช้กับแถวใหม่ที่ insert/update ตั้งแต่วินาทีนี้เป็นต้นไปทันที
ALTER TABLE orders
    ADD CONSTRAINT chk_orders_total_amount_not_null
    CHECK (total_amount IS NOT NULL) NOT VALID;
```

ทดสอบว่ากฎมีผลกับแถวใหม่ทันที แม้ยังไม่ validate แถวเก่า:

```sql
-- แถวใหม่ที่พยายามใส่ NULL จะถูกปฏิเสธทันที แม้ constraint ยัง NOT VALID อยู่
INSERT INTO orders (customer_id, order_status, total_amount)
VALUES (1, 'pending', NULL);
-- ERROR:  new row for relation "orders" violates check constraint "chk_orders_total_amount_not_null"
```

```sql
-- ขั้นตอนที่ 2: VALIDATE CONSTRAINT — สแกนตรวจแถวเก่าทั้งหมดว่าผ่านกฎหรือไม่
-- คำสั่งนี้ใช้เวลานาน (ต้องสแกนทั้งตาราง) แต่ใช้ lock ระดับเบากว่า
-- คือ SHARE UPDATE EXCLUSIVE ซึ่งอนุญาตให้ SELECT, INSERT, UPDATE, DELETE ทำงานพร้อมกันได้ตามปกติ
-- (บล็อกแค่ DDL อื่นที่มากระทบตารางเดียวกันพร้อมกัน)
ALTER TABLE orders VALIDATE CONSTRAINT chk_orders_total_amount_not_null;
```

นี่คือหัวใจของเทคนิคนี้: `VALIDATE CONSTRAINT` ใช้ lock ระดับ **`SHARE UPDATE EXCLUSIVE`** ไม่ใช่ `ACCESS EXCLUSIVE` — แม้จะสแกนทั้งตารางเหมือนกัน แต่**ไม่บล็อก DML** (read/write ปกติ) ระหว่างการสแกน ต่างจากการทำ `SET NOT NULL` ตรง ๆ ที่บล็อกทุกอย่างตลอดการสแกน

```sql
-- ตรวจสอบว่า constraint validate สำเร็จแล้ว
SELECT conname, convalidated
FROM pg_constraint
WHERE conname = 'chk_orders_total_amount_not_null';
--  conname                             | convalidated
--  chk_orders_total_amount_not_null    | t
```

```sql
-- ขั้นตอนที่ 3 (ทางเลือก, PostgreSQL 12+): แปลงเป็น NOT NULL จริงแบบ metadata-only
-- เมื่อมี CHECK constraint ที่ validate แล้วว่าไม่มี NULL อยู่ PostgreSQL planner
-- จะรู้และ "เชื่อ" constraint นี้ จึงข้าม full table scan ตอน SET NOT NULL ได้ (เร็วมาก)
ALTER TABLE orders ALTER COLUMN total_amount SET NOT NULL;

-- ขั้นตอนที่ 4: ลบ CHECK constraint ที่ซ้ำซ้อนกับ NOT NULL แล้ว เพื่อความสะอาดของ schema
ALTER TABLE orders DROP CONSTRAINT chk_orders_total_amount_not_null;
```

> **ทำไมยังต้องมีขั้นตอนที่ 3-4 ถ้า CHECK constraint ก็บังคับ NOT NULL ได้อยู่แล้ว?** เพราะ `NOT NULL` metadata แบบเนทีฟมีประโยชน์มากกว่า CHECK constraint ทั่วไป เช่น query planner ใช้ข้อมูลนี้ optimize ได้ตรงไปตรงมากว่า, เครื่องมือ introspection/ORM จำนวนมากอ่านค่า `is_nullable` จาก `information_schema.columns` โดยตรงแทนที่จะแกะ CHECK constraint, และมันสื่อความหมาย (self-documenting) ชัดเจนกว่า

### เปรียบเทียบ lock ระหว่างสองวิธี

| ขั้นตอน | คำสั่ง | ประเภท lock | ต้อง scan ตาราง? | ระยะเวลาถือ lock หนัก |
|---|---|---|---|---|
| วิธีตรง ๆ (อันตราย) | `SET NOT NULL` | ACCESS EXCLUSIVE | ใช่ | นานเท่าที่ scan (อาจหลายนาที) |
| วิธีปลอดภัย ขั้น 1 | `ADD CONSTRAINT ... NOT VALID` | ACCESS EXCLUSIVE | ไม่ | เสี้ยววินาที |
| วิธีปลอดภัย ขั้น 2 | `VALIDATE CONSTRAINT` | SHARE UPDATE EXCLUSIVE | ใช่ | ไม่บล็อก DML แม้ scan นาน |
| วิธีปลอดภัย ขั้น 3 | `SET NOT NULL` (หลัง validate CHECK แล้ว) | ACCESS EXCLUSIVE | ไม่ (ข้ามได้เพราะเชื่อ CHECK) | เสี้ยววินาที |

### นำเทคนิคเดียวกันไปใช้กับ FOREIGN KEY และ CHECK constraint ทั่วไป

เทคนิค `NOT VALID` → `VALIDATE CONSTRAINT` ใช้ได้กับทุก constraint ที่ไม่ใช่ `UNIQUE`/`PRIMARY KEY` เช่น foreign key ด้วย:

```sql
-- ตัวอย่าง: เพิ่ม FK จาก order_items.product_id -> products.product_id แบบปลอดภัย
ALTER TABLE order_items
    ADD CONSTRAINT fk_order_items_product
    FOREIGN KEY (product_id) REFERENCES products(product_id) NOT VALID;

ALTER TABLE order_items VALIDATE CONSTRAINT fk_order_items_product;
```

```sql
-- ตัวอย่าง: เพิ่ม CHECK constraint ทั่วไปแบบปลอดภัยกับตารางใหญ่
ALTER TABLE orders
    ADD CONSTRAINT chk_orders_total_amount_nonneg
    CHECK (total_amount >= 0) NOT VALID;

ALTER TABLE orders VALIDATE CONSTRAINT chk_orders_total_amount_nonneg;
```

**ข้อยกเว้นสำคัญ:** `UNIQUE` และ `PRIMARY KEY` constraint **ไม่รองรับ `NOT VALID`** เพราะธรรมชาติของมันคือต้องอาศัย unique index ซึ่งสร้างได้ด้วยการรวมกับ `CREATE UNIQUE INDEX CONCURRENTLY` ก่อน แล้วค่อยผูกเป็น constraint ภายหลัง:

```sql
-- สร้าง unique index แบบไม่ล็อกยาว (เหมือนหลักการ Step 788)
CREATE UNIQUE INDEX CONCURRENTLY IF NOT EXISTS uq_products_sku ON products (sku);

-- ผูก index ที่สร้างไว้แล้วให้กลายเป็น PRIMARY KEY / UNIQUE constraint แบบ metadata-only (เร็วมาก)
ALTER TABLE products ADD CONSTRAINT uq_products_sku UNIQUE USING INDEX uq_products_sku;
```

---

## Step 790: แบบฝึกหัดรวม — วางแผนและเขียน migration script แบบ zero-downtime สำหรับการเปลี่ยนแปลง schema ครั้งใหญ่ของระบบ e-commerce

### โจทย์ใหญ่

ทีมสถาปัตยกรรมตัดสินใจปรับปรุงระบบ e-commerce ครั้งใหญ่ โดยต้องการ **สี่การเปลี่ยนแปลงพร้อมกัน** บนตาราง `orders` (สมมติมี 80 ล้านแถว รับ traffic ตลอด 24 ชม.):

1. เพิ่มคอลัมน์ `discount_amount NUMERIC(12,2)` ที่ต้องมีค่าเสมอ (NOT NULL) ค่าเริ่มต้นคือ backfill จาก `total_amount * 0` (คือ 0) สำหรับข้อมูลเก่า แต่ order ใหม่คำนวณจาก business logic จริง
2. เปลี่ยนชื่อคอลัมน์ `order_status` เป็น `status` (แอปหลายตัว deploy แบบ rolling ไม่พร้อมกัน)
3. เปลี่ยน type ของ `total_amount` จาก `NUMERIC(12,2)` เป็น `NUMERIC(14,4)`
4. เพิ่ม index บน `(customer_id, status)` เพื่อรองรับหน้าประวัติคำสั่งซื้อ

ให้วางแผนเป็น migration หลายไฟล์ หลายรอบ deploy โดยห้ามระบบหยุดทำงานแม้แต่วินาทีเดียว

### แผนการ deploy แบบ multi-phase

การเปลี่ยนแปลงทั้งสี่เรื่องข้างต้น**ห้ามรวมกันในไฟล์เดียวหรือรอบ deploy เดียว** ตามหลักการ Step 783 (small steps) และต้องเรียงลำดับตาม dependency ให้ถูกต้อง ตัวอย่างแผนที่ปลอดภัย:

```
Deploy Round 1 (schema change เล็ก ไม่กระทบแอป):
  031_add_discount_amount_nullable.sql        -- Step 785 ขั้น 1
  032_add_total_amount_v2_nullable.sql        -- Step 787 ขั้น 1
  033_add_status_column_dualwrite_trigger.sql -- Step 786 แนวทาง dual-write

Deploy Round 2 (deploy แอปเวอร์ชันใหม่ที่ dual-write ทุกคอลัมน์ใหม่):
  -- (ไม่มี migration SQL ในรอบนี้ เป็นการ deploy โค้ดแอปพลิเคชันอย่างเดียว)

Deploy Round 3 (backfill ข้อมูลเก่า เป็น batch, รันตอน traffic ต่ำ):
  034_backfill_discount_amount.sql
  035_backfill_total_amount_v2.sql
  -- status ถูก sync อัตโนมัติผ่าน trigger ตั้งแต่ round 1 แล้ว แต่ยังต้อง backfill
  -- ข้อมูลเก่าที่มีอยู่ก่อน trigger ถูกสร้าง (แถวเก่าที่ order_status มีค่าแต่ status ยังไม่ sync)
  036_backfill_status_column.sql

Deploy Round 4 (deploy แอปเวอร์ชันใหม่: อ่านคอลัมน์ใหม่แทนคอลัมน์เก่า):
  -- (ไม่มี migration SQL อีกเช่นกัน เป็น read cut-over ฝั่งแอป)

Deploy Round 5 (เพิ่ม constraint + index บนคอลัมน์ใหม่ที่ backfill ครบแล้ว):
  037_add_index_orders_customer_status.sql    -- Step 788 CONCURRENTLY
  038_add_notnull_discount_amount.sql         -- Step 789 NOT VALID -> VALIDATE

Deploy Round 6 (cleanup: ลบของเก่าหลังมั่นใจว่าไม่มีแอปตัวไหนอ้างแล้ว เฝ้าดูสัก 1-2 สัปดาห์):
  039_cleanup_drop_old_columns_and_triggers.sql
```

### เนื้อหาไฟล์ migration ทั้งหมด (รันได้จริง เรียงตามลำดับ)

**Round 1 — `031_add_discount_amount_nullable.sql`:**

```sql
ALTER TABLE orders ADD COLUMN IF NOT EXISTS discount_amount NUMERIC(12,2);
```

**Round 1 — `032_add_total_amount_v2_nullable.sql`:**

```sql
ALTER TABLE orders ADD COLUMN IF NOT EXISTS total_amount_v2 NUMERIC(14,4);
```

**Round 1 — `033_add_status_column_dualwrite_trigger.sql`:**

```sql
ALTER TABLE orders ADD COLUMN IF NOT EXISTS status TEXT;

CREATE OR REPLACE FUNCTION sync_orders_migration_columns()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        NEW.status := COALESCE(NEW.status, NEW.order_status);
        NEW.order_status := COALESCE(NEW.order_status, NEW.status);

        NEW.total_amount_v2 := COALESCE(NEW.total_amount_v2, NEW.total_amount::NUMERIC(14,4));
        NEW.total_amount := COALESCE(NEW.total_amount, NEW.total_amount_v2::NUMERIC(12,2));

        NEW.discount_amount := COALESCE(NEW.discount_amount, 0);
    ELSIF TG_OP = 'UPDATE' THEN
        IF NEW.order_status IS DISTINCT FROM OLD.order_status THEN
            NEW.status := NEW.order_status;
        ELSIF NEW.status IS DISTINCT FROM OLD.status THEN
            NEW.order_status := NEW.status;
        END IF;

        IF NEW.total_amount IS DISTINCT FROM OLD.total_amount THEN
            NEW.total_amount_v2 := NEW.total_amount::NUMERIC(14,4);
        ELSIF NEW.total_amount_v2 IS DISTINCT FROM OLD.total_amount_v2 THEN
            NEW.total_amount := NEW.total_amount_v2::NUMERIC(12,2);
        END IF;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_sync_orders_migration_columns
BEFORE INSERT OR UPDATE ON orders
FOR EACH ROW EXECUTE FUNCTION sync_orders_migration_columns();
```

**Round 3 — `034_backfill_discount_amount.sql`:**

```sql
CREATE OR REPLACE PROCEDURE backfill_orders_discount_amount(batch_size INTEGER DEFAULT 5000)
LANGUAGE plpgsql
AS $$
DECLARE
    rows_updated INTEGER;
BEGIN
    LOOP
        UPDATE orders
        SET discount_amount = 0
        WHERE order_id IN (
            SELECT order_id FROM orders
            WHERE discount_amount IS NULL
            ORDER BY order_id
            LIMIT batch_size
        );
        GET DIAGNOSTICS rows_updated = ROW_COUNT;
        EXIT WHEN rows_updated = 0;
        RAISE NOTICE 'discount_amount backfilled % rows', rows_updated;
        COMMIT;
        PERFORM pg_sleep(0.1);
    END LOOP;
END;
$$;

CALL backfill_orders_discount_amount(5000);
```

**Round 3 — `035_backfill_total_amount_v2.sql`:**

```sql
CREATE OR REPLACE PROCEDURE backfill_orders_total_amount_v2(batch_size INTEGER DEFAULT 5000)
LANGUAGE plpgsql
AS $$
DECLARE
    rows_updated INTEGER;
BEGIN
    LOOP
        UPDATE orders
        SET total_amount_v2 = total_amount::NUMERIC(14,4)
        WHERE order_id IN (
            SELECT order_id FROM orders
            WHERE total_amount_v2 IS NULL
            ORDER BY order_id
            LIMIT batch_size
        );
        GET DIAGNOSTICS rows_updated = ROW_COUNT;
        EXIT WHEN rows_updated = 0;
        RAISE NOTICE 'total_amount_v2 backfilled % rows', rows_updated;
        COMMIT;
        PERFORM pg_sleep(0.1);
    END LOOP;
END;
$$;

CALL backfill_orders_total_amount_v2(5000);
```

**Round 3 — `036_backfill_status_column.sql`:**

```sql
CREATE OR REPLACE PROCEDURE backfill_orders_status(batch_size INTEGER DEFAULT 5000)
LANGUAGE plpgsql
AS $$
DECLARE
    rows_updated INTEGER;
BEGIN
    LOOP
        UPDATE orders
        SET status = order_status
        WHERE order_id IN (
            SELECT order_id FROM orders
            WHERE status IS NULL
            ORDER BY order_id
            LIMIT batch_size
        );
        GET DIAGNOSTICS rows_updated = ROW_COUNT;
        EXIT WHEN rows_updated = 0;
        RAISE NOTICE 'status backfilled % rows', rows_updated;
        COMMIT;
        PERFORM pg_sleep(0.1);
    END LOOP;
END;
$$;

CALL backfill_orders_status(5000);

-- ตรวจสอบก่อนไปรอบถัดไปว่าไม่มีแถวไหนหลงเหลือ NULL ในคอลัมน์ใหม่ทั้งสาม
SELECT
    count(*) FILTER (WHERE discount_amount IS NULL)  AS null_discount,
    count(*) FILTER (WHERE total_amount_v2 IS NULL)  AS null_total_v2,
    count(*) FILTER (WHERE status IS NULL)            AS null_status
FROM orders;
-- ต้องได้ 0, 0, 0 ทั้งหมดก่อนดำเนินการต่อ
```

**Round 5 — `037_add_index_orders_customer_status.sql`:**

```sql
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_orders_customer_status
    ON orders (customer_id, status);
```

**Round 5 — `038_add_notnull_discount_amount.sql`:**

```sql
ALTER TABLE orders
    ADD CONSTRAINT chk_orders_discount_amount_not_null
    CHECK (discount_amount IS NOT NULL) NOT VALID;

ALTER TABLE orders VALIDATE CONSTRAINT chk_orders_discount_amount_not_null;

ALTER TABLE orders ALTER COLUMN discount_amount SET NOT NULL;

ALTER TABLE orders DROP CONSTRAINT chk_orders_discount_amount_not_null;
```

**Round 6 — `039_cleanup_drop_old_columns_and_triggers.sql`** (รันเฉพาะหลังจากยืนยันแล้วว่าแอปทุก instance deploy เวอร์ชันใหม่ครบ 100% และเฝ้าสังเกตอย่างน้อย 1-2 สัปดาห์โดยไม่มีปัญหา):

```sql
BEGIN;

DROP TRIGGER IF EXISTS trg_sync_orders_migration_columns ON orders;
DROP FUNCTION IF EXISTS sync_orders_migration_columns();

ALTER TABLE orders DROP COLUMN IF EXISTS order_status;
ALTER TABLE orders DROP COLUMN IF EXISTS total_amount;

ALTER TABLE orders RENAME COLUMN total_amount_v2 TO total_amount;

-- เพิ่ม NOT NULL ให้ total_amount ใหม่ด้วยเทคนิคปลอดภัยเช่นเดิม
ALTER TABLE orders
    ADD CONSTRAINT chk_orders_total_amount_not_null
    CHECK (total_amount IS NOT NULL) NOT VALID;
ALTER TABLE orders VALIDATE CONSTRAINT chk_orders_total_amount_not_null;
ALTER TABLE orders ALTER COLUMN total_amount SET NOT NULL;
ALTER TABLE orders DROP CONSTRAINT chk_orders_total_amount_not_null;

COMMIT;
```

### ทำไมต้องแบ่งเป็น 6 รอบ deploy ไม่ทำน้อยกว่านี้

| รอบ | เหตุผลที่ต้องแยก |
|---|---|
| Round 1 vs Round 2 | ต้องมีคอลัมน์และ trigger รองรับ **ก่อน** แอปเวอร์ชันใหม่เริ่ม dual-write ไม่งั้นแอปใหม่จะ error เพราะคอลัมน์ยังไม่มี |
| Round 2 vs Round 3 | ต้องรอให้แอปทุก instance เขียนคอลัมน์ใหม่แล้ว (ผ่าน trigger sync) ก่อน backfill ไม่งั้นข้อมูลที่เพิ่งเขียนเข้ามาระหว่าง backfill อาจถูกข้าม |
| Round 3 vs Round 4 | ต้อง backfill ให้ครบ 100% ก่อนให้แอปเริ่ม**อ่าน**คอลัมน์ใหม่ ไม่งั้นแอปจะอ่านค่า NULL ผิดพลาด |
| Round 4 vs Round 5 | ต้องมั่นใจว่าแอปอ่าน/เขียนคอลัมน์ใหม่เสถียรแล้ว ก่อนบังคับ constraint และสร้าง index ซึ่งเป็นการเปลี่ยนแปลงที่ rollback ยากกว่า |
| Round 5 vs Round 6 | การลบคอลัมน์เก่า (`DROP COLUMN`) เป็น**การเปลี่ยนแปลงที่ทำลายข้อมูล ย้อนกลับไม่ได้** ต้องมั่นใจสูงสุดและเฝ้าระวังนานที่สุดก่อนลงมือ |

หลักที่ใช้ตลอดทั้งแผน: **ทุกขั้นตอนต้อง backward-compatible กับแอปเวอร์ชันก่อนหน้าเสมอ** เพื่อรองรับ rolling deployment และรองรับการ rollback แอปกลับไปเวอร์ชันก่อนได้ทุกจุดโดยไม่กระทบข้อมูล (ยกเว้น Round 6 ที่เป็นจุด "ไม่มีทางถอย" ต้องทำหลังสุดและมั่นใจที่สุดเท่านั้น)

---

## สรุปท้ายบท

บทนี้ครอบคลุมกลยุทธ์การทำ database migration ตั้งแต่แนวคิดพื้นฐาน (schema versioning, เครื่องมือ migration) ไปจนถึงเทคนิคขั้นสูงสำหรับทำ **zero-downtime migration** บนตารางขนาดใหญ่ในระบบ production ที่ให้บริการตลอด 24 ชั่วโมง

### หลักการสำคัญที่ต้องจำ

1. **Migration ต้องมีเวอร์ชัน ตรวจสอบย้อนหลังได้ และรันผ่าน pipeline อัตโนมัติเสมอ** ไม่แก้ schema บน production ด้วยมือ
2. **Migration ที่ดีต้อง idempotent, reversible เท่าที่เป็นไปได้ และแบ่งเป็นขั้นตอนเล็ก ๆ**
3. **`ALTER TABLE` หลายรูปแบบต้องใช้ `ACCESS EXCLUSIVE LOCK`** ซึ่งบล็อกทุก query แม้แต่ `SELECT` — ยิ่งตารางใหญ่ ยิ่งอันตราย
4. **ตั้ง `lock_timeout` เสมอ** ก่อนรัน migration ที่มีความเสี่ยงบน production เพื่อให้ fail เร็วแทนที่จะค้างเงียบ ๆ
5. หัวใจของทุก zero-downtime pattern คือ **"อย่าเปลี่ยนของเดิมทันที ให้สร้างของใหม่คู่ขนานก่อน (dual-write) แล้วค่อย backfill และสลับทีละขั้น"**

### Checklist Zero-downtime Migration

```
[ ] แยก migration เป็นไฟล์เล็ก ๆ ทีละเรื่อง (ไม่รวมหลายการเปลี่ยนแปลงในไฟล์เดียว)
[ ] ทุก DDL เขียนแบบ idempotent (IF NOT EXISTS / IF EXISTS / DO block ตรวจก่อน)
[ ] ตั้ง lock_timeout และ statement_timeout ก่อนรัน migration บน production เสมอ
[ ] เพิ่มคอลัมน์ใหม่แบบ nullable ก่อนเสมอ ไม่ใส่ NOT NULL ทันทีถ้าต้อง backfill ข้อมูลจากแถวเดิม
[ ] Backfill ข้อมูลเป็น batch เล็ก ๆ (ผ่าน PROCEDURE ที่ COMMIT ทีละช่วง) ไม่ใช้ UPDATE เดียวทั้งตาราง
[ ] เปลี่ยนชื่อคอลัมน์/ตาราง ผ่าน compatibility view หรือ dual-write trigger ไม่ RENAME ตรง ๆ ทันที
[ ] เปลี่ยน data type ผ่านการสร้างคอลัมน์ใหม่ dual-write แล้วค่อย cut-over ไม่ ALTER COLUMN TYPE ตรง ๆ
[ ] สร้าง/ลบ index บนตารางใหญ่ด้วย CREATE/DROP INDEX CONCURRENTLY เสมอ และตรวจ INVALID index ทุกครั้ง
[ ] เพิ่ม constraint (NOT NULL, CHECK, FOREIGN KEY) ด้วย NOT VALID ก่อน แล้วค่อย VALIDATE CONSTRAINT ทีหลัง
[ ] UNIQUE/PRIMARY KEY ใหม่ สร้างผ่าน CREATE UNIQUE INDEX CONCURRENTLY แล้วค่อยผูกด้วย USING INDEX
[ ] แต่ละ deploy round ต้อง backward-compatible กับแอปเวอร์ชันก่อนหน้าเสมอ (รองรับ rolling deploy/rollback)
[ ] การเปลี่ยนแปลงที่ทำลายข้อมูล (DROP COLUMN/TABLE) ทำเป็นรอบสุดท้ายเสมอ หลังเฝ้าสังเกตระยะหนึ่งแล้ว
[ ] มี monitoring (pg_stat_activity, pg_locks) เฝ้าดูระหว่างรัน migration จริงบน production
```

---

## แบบฝึกหัด

<details>
<summary><strong>แบบฝึกหัดที่ 1:</strong> อธิบายว่าทำไมการแก้ schema ผ่าน psql มือเปล่าบน production โดยไม่ผ่านระบบ migration ถึงเป็นความเสี่ยง ยกตัวอย่างปัญหาอย่างน้อย 3 ข้อ</summary>

**เฉลย:**

1. **ไม่มีบันทึกประวัติ (no audit trail)** — ไม่รู้ว่าใครแก้อะไร เมื่อไหร่ ทำไม ยากต่อการ debug เมื่อเกิดปัญหาภายหลัง
2. **Schema drift ระหว่างสภาพแวดล้อม** — dev, staging, production มี schema ไม่ตรงกัน เพราะแก้ด้วยมือแยกกันคนละที่ ทำให้เกิดบั๊กที่เจอเฉพาะบาง environment
3. **Reproducibility เสีย** — ถ้าต้องสร้างฐานข้อมูลใหม่ (เช่น disaster recovery, environment ใหม่) ไม่มีทางรู้ชัดว่าต้องรัน SQL อะไรบ้างตามลำดับใดจึงจะได้ schema ปัจจุบัน
4. **ไม่มี rollback plan** — ถ้าแก้ผิดพลาด ไม่มี down migration ให้ย้อน ต้องพึ่ง backup ทั้งฐานซึ่งอาจทำให้สูญเสียข้อมูลใหม่ที่เกิดขึ้นระหว่างนั้น
5. **ไม่ผ่าน code review** — การแก้ผ่าน psql มือเปล่ามักข้ามกระบวนการตรวจสอบก่อน deploy ทำให้พลาดปัญหาที่ผู้อื่นอาจมองเห็นได้ เช่นลืมนึกถึง lock ที่จะเกิดบนตารางใหญ่

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 2:</strong> เขียน migration แบบ idempotent สำหรับเพิ่มคอลัมน์ <code>weight_kg NUMERIC(6,3)</code> ในตาราง <code>products</code> พร้อมเพิ่ม index บนคอลัมน์นี้แบบ idempotent เช่นกัน</summary>

**เฉลย:**

```sql
ALTER TABLE products ADD COLUMN IF NOT EXISTS weight_kg NUMERIC(6,3);

CREATE INDEX IF NOT EXISTS idx_products_weight_kg ON products (weight_kg);
```

ทั้งสองคำสั่งใช้ `IF NOT EXISTS` ทำให้รันซ้ำกี่ครั้งก็ไม่ error หากคอลัมน์หรือ index มีอยู่แล้ว

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 3:</strong> จำลองสถานการณ์: Session A เปิด transaction แล้วรัน <code>ALTER TABLE orders ADD COLUMN note TEXT NOT NULL DEFAULT ''</code> โดยยังไม่ COMMIT เขียน query ที่ใช้ตรวจสอบว่า Session ไหนบ้างกำลังถูกบล็อกอยู่ และบล็อกโดยใคร</summary>

**เฉลย:**

```sql
SELECT
    blocked.pid                    AS blocked_pid,
    blocked.usename                AS blocked_user,
    blocked.query                  AS blocked_query,
    blocking.pid                   AS blocking_pid,
    blocking.usename               AS blocking_user,
    blocking.query                 AS blocking_query,
    now() - blocked.query_start    AS waiting_duration
FROM pg_stat_activity AS blocked
JOIN pg_locks AS bl ON bl.pid = blocked.pid AND NOT bl.granted
JOIN pg_locks AS kl ON kl.locktype = bl.locktype
                    AND kl.database IS NOT DISTINCT FROM bl.database
                    AND kl.relation IS NOT DISTINCT FROM bl.relation
                    AND kl.granted
JOIN pg_stat_activity AS blocking ON blocking.pid = kl.pid
WHERE blocked.pid <> blocking.pid
ORDER BY waiting_duration DESC;
```

query นี้จะโชว์ว่า session ที่รอ lock (`blocked_pid`) ถูกบล็อกโดย session ไหน (`blocking_pid`) พร้อมระยะเวลาที่รออยู่ ในกรณีนี้ session ที่พยายาม `SELECT`/`UPDATE`/`INSERT` บนตาราง `orders` ระหว่างที่ Session A ยังไม่ COMMIT จะถูกบล็อกโดย Session A ทั้งหมด เพราะ `ADD COLUMN ... NOT NULL DEFAULT ''` (ค่าคงที่) ในเวอร์ชันใหม่จริง ๆ แล้วเร็วมาก แต่ตัวอย่างนี้จำลองว่า Session A ค้าง (ยังไม่ COMMIT) ด้วยเหตุผลอื่น เช่น รอ lock อีกตัวอยู่ก่อน

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 4:</strong> ตาราง <code>order_items</code> มี 30 ล้านแถว ต้องการเพิ่มคอลัมน์ <code>discount_code TEXT</code> ที่สุดท้ายต้องเป็น NOT NULL โดยมีค่า default เป็น <code>'NONE'</code> สำหรับแถวเก่าทั้งหมด จงวางแผน migration เป็นขั้นตอน (ระบุ SQL ในแต่ละขั้น)</summary>

**เฉลย:**

```sql
-- ขั้นที่ 1: เพิ่มคอลัมน์ nullable (metadata-only เร็วมาก)
ALTER TABLE order_items ADD COLUMN IF NOT EXISTS discount_code TEXT;

-- ขั้นที่ 2: backfill เป็น batch (แถวเก่าทั้งหมดตั้งค่าเป็น 'NONE')
CREATE OR REPLACE PROCEDURE backfill_order_items_discount_code(batch_size INTEGER DEFAULT 5000)
LANGUAGE plpgsql
AS $$
DECLARE
    rows_updated INTEGER;
BEGIN
    LOOP
        UPDATE order_items
        SET discount_code = 'NONE'
        WHERE order_item_id IN (
            SELECT order_item_id FROM order_items
            WHERE discount_code IS NULL
            ORDER BY order_item_id
            LIMIT batch_size
        );
        GET DIAGNOSTICS rows_updated = ROW_COUNT;
        EXIT WHEN rows_updated = 0;
        COMMIT;
        PERFORM pg_sleep(0.1);
    END LOOP;
END;
$$;

CALL backfill_order_items_discount_code(5000);

-- ขั้นที่ 3: เพิ่ม NOT NULL แบบปลอดภัย
ALTER TABLE order_items
    ADD CONSTRAINT chk_order_items_discount_code_not_null
    CHECK (discount_code IS NOT NULL) NOT VALID;

ALTER TABLE order_items VALIDATE CONSTRAINT chk_order_items_discount_code_not_null;

ALTER TABLE order_items ALTER COLUMN discount_code SET NOT NULL;

ALTER TABLE order_items DROP CONSTRAINT chk_order_items_discount_code_not_null;

-- (ทางเลือก) ใส่ DEFAULT ให้แถวใหม่ในอนาคตไม่ต้องพึ่งแอปส่งค่ามาเสมอ
ALTER TABLE order_items ALTER COLUMN discount_code SET DEFAULT 'NONE';
```

**ข้อควรระวัง:** ในกรณีนี้เนื่องจากค่า default เป็น `'NONE'` คงที่เหมือนกันทุกแถว จริง ๆ แล้วสามารถใช้ `ADD COLUMN discount_code TEXT NOT NULL DEFAULT 'NONE'` ในคำสั่งเดียวได้เลยตั้งแต่ PostgreSQL 11+ (เพราะ default เป็นค่าคงที่ ไม่ volatile จึงไม่ rewrite ตาราง) วิธี batch ข้างต้นจำเป็นเฉพาะเมื่อค่าที่ backfill ต้องคำนวณต่างกันในแต่ละแถว — แบบฝึกหัดนี้จงใจแสดงวิธี batch เพื่อฝึกรูปแบบที่ใช้ได้ทั่วไปในทุกกรณี

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 5:</strong> ทีมต้องการเปลี่ยนชื่อคอลัมน์ <code>customers.full_name</code> เป็น <code>customers.display_name</code> โดยมี mobile app เวอร์ชันเก่าที่ยังอ้างชื่อ <code>full_name</code> อยู่และไม่สามารถบังคับอัปเดตได้ทันที จงออกแบบ migration ด้วยแนวทาง compatibility view</summary>

**เฉลย:**

```sql
BEGIN;

-- 1) ย้ายตารางจริงออกจากชื่อเดิมชั่วคราว
ALTER TABLE customers RENAME TO customers_base;

-- 2) เปลี่ยนชื่อคอลัมน์ในตารางจริง
ALTER TABLE customers_base RENAME COLUMN full_name TO display_name;

-- 3) สร้าง view ชื่อ customers (ชื่อเดิม) expose คอลัมน์ชื่อเก่าให้ mobile app เก่าใช้ต่อได้
CREATE VIEW customers AS
SELECT
    customer_id,
    email,
    display_name AS full_name,
    created_at
FROM customers_base;

COMMIT;
```

หลังจากมั่นใจว่า mobile app ทุกเวอร์ชันอัปเดตมาใช้ `display_name` ผ่าน `customers_base` แล้ว (อาจใช้เวลาหลายเดือนเพราะควบคุม mobile app version ในมือผู้ใช้ไม่ได้ทันที) จึงค่อยลบ view และเปลี่ยนชื่อตารางกลับ:

```sql
DROP VIEW IF EXISTS customers;
ALTER TABLE customers_base RENAME TO customers;
```

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 6:</strong> เขียน trigger function สำหรับ dual-write ระหว่างคอลัมน์ <code>products.stock_qty</code> (INTEGER) กับคอลัมน์ใหม่ <code>products.stock_qty_v2</code> (BIGINT) ที่กำลังจะมาแทนที่</summary>

**เฉลย:**

```sql
ALTER TABLE products ADD COLUMN IF NOT EXISTS stock_qty_v2 BIGINT;

CREATE OR REPLACE FUNCTION sync_products_stock_qty_v2()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        IF NEW.stock_qty_v2 IS NULL AND NEW.stock_qty IS NOT NULL THEN
            NEW.stock_qty_v2 := NEW.stock_qty::BIGINT;
        END IF;
    ELSIF TG_OP = 'UPDATE' THEN
        IF NEW.stock_qty IS DISTINCT FROM OLD.stock_qty THEN
            NEW.stock_qty_v2 := NEW.stock_qty::BIGINT;
        ELSIF NEW.stock_qty_v2 IS DISTINCT FROM OLD.stock_qty_v2 THEN
            NEW.stock_qty := NEW.stock_qty_v2::INTEGER;  -- ระวัง overflow ถ้าค่าเกินขอบเขต INTEGER
        END IF;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_sync_products_stock_qty_v2
BEFORE INSERT OR UPDATE ON products
FOR EACH ROW EXECUTE FUNCTION sync_products_stock_qty_v2();
```

**ข้อสังเกต:** ทิศทาง `BIGINT -> INTEGER` เสี่ยง overflow ถ้าค่าที่เขียนผ่าน `stock_qty_v2` เกินขอบเขตของ `INTEGER` (มากกว่า 2,147,483,647) ในทางปฏิบัติควรจำกัดว่าเฉพาะ "แอปเก่า" เท่านั้นที่เขียน `stock_qty` (ทิศทางเดียว) ระหว่างช่วง dual-write เพื่อลดความเสี่ยงนี้ หรือเพิ่ม CHECK constraint ป้องกัน

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 7:</strong> อธิบายความแตกต่างระหว่าง lock ที่ใช้ใน <code>CREATE INDEX</code> ธรรมดา กับ <code>CREATE INDEX CONCURRENTLY</code> และอธิบายว่าทำไม <code>CONCURRENTLY</code> ถึงใช้เวลานานกว่า</summary>

**เฉลย:**

- `CREATE INDEX` ธรรมดา ใช้ `SHARE` lock บนตาราง ซึ่งบล็อก `INSERT`/`UPDATE`/`DELETE` ทั้งหมด (อนุญาตแค่ `SELECT`) ตลอดเวลาที่สแกนตารางสร้าง index (สแกนรอบเดียว)
- `CREATE INDEX CONCURRENTLY` ใช้ lock ที่เบากว่า (`SHARE UPDATE EXCLUSIVE`) ซึ่งไม่บล็อก DML เลย แต่ต้องสแกนตาราง**สองรอบ**: รอบแรกสร้าง index จากข้อมูล ณ ขณะนั้น รอบสองตรวจสอบและรวมแถวที่ถูกเปลี่ยนแปลง (insert/update/delete) ระหว่างที่รอบแรกกำลังทำงานอยู่ เพื่อให้ index สมบูรณ์และสอดคล้องกับข้อมูลจริง การสแกนสองรอบนี้เองที่ทำให้ `CONCURRENTLY` ใช้เวลารวมนานกว่า `CREATE INDEX` ปกติ แต่แลกมาด้วยการไม่บล็อก write ระหว่างทาง ซึ่งคุ้มค่ามากสำหรับตารางที่รับ traffic ต่อเนื่องตลอดเวลา

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 8:</strong> รัน <code>CREATE INDEX CONCURRENTLY</code> แล้วถูก cancel กลางทาง (เช่น connection หลุด) จงเขียน query ตรวจสอบว่ามี index ใดในระบบที่อยู่ในสถานะ invalid บ้าง และเขียนขั้นตอนแก้ไข</summary>

**เฉลย:**

```sql
-- ตรวจสอบ index ทั้งหมดในระบบที่อยู่ในสถานะ invalid
SELECT
    n.nspname                    AS schema_name,
    i.indexrelid::regclass       AS index_name,
    i.indrelid::regclass         AS table_name,
    i.indisvalid,
    i.indisready
FROM pg_index i
JOIN pg_class c ON c.oid = i.indexrelid
JOIN pg_namespace n ON n.oid = c.relnamespace
WHERE i.indisvalid = false;
```

ขั้นตอนแก้ไข: index ที่ invalid ใช้งานไม่ได้ (planner จะไม่เลือกใช้) แต่ยังกิน storage และถูก maintain ทุกครั้งที่มี write โดยเปล่าประโยชน์ ต้องลบทิ้งแล้วสร้างใหม่ด้วย `CONCURRENTLY` อีกครั้ง (การ `DROP` ก็ควรใช้ `CONCURRENTLY` เช่นกัน เพื่อไม่ล็อกตารางตอนลบ):

```sql
DROP INDEX CONCURRENTLY IF EXISTS idx_orders_customer_id;

CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_orders_customer_id ON orders (customer_id);

-- ตรวจสอบซ้ำว่าสร้างสำเร็จและ valid แล้ว
SELECT indexrelid::regclass, indisvalid
FROM pg_index
WHERE indexrelid = 'idx_orders_customer_id'::regclass;
```

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 9:</strong> ตาราง <code>orders</code> มีคอลัมน์ <code>customer_id</code> ที่ปัจจุบันเป็น nullable และมีข้อมูลอยู่แล้วครบทุกแถว (ไม่มี NULL เลย) จงเขียน migration เพิ่ม NOT NULL constraint แบบปลอดภัยที่สุด พร้อมอธิบายว่าทำไมขั้นตอนสุดท้ายถึงเร็ว</summary>

**เฉลย:**

```sql
-- ขั้นที่ 1: เพิ่ม CHECK constraint แบบ NOT VALID (เร็วมาก ไม่ scan)
ALTER TABLE orders
    ADD CONSTRAINT chk_orders_customer_id_not_null
    CHECK (customer_id IS NOT NULL) NOT VALID;

-- ขั้นที่ 2: validate ข้อมูลเดิมทั้งหมด (scan ตาราง แต่ไม่บล็อก DML)
ALTER TABLE orders VALIDATE CONSTRAINT chk_orders_customer_id_not_null;

-- ขั้นที่ 3: แปลงเป็น NOT NULL จริง
ALTER TABLE orders ALTER COLUMN customer_id SET NOT NULL;

-- ขั้นที่ 4: ลบ CHECK constraint ที่ซ้ำซ้อน
ALTER TABLE orders DROP CONSTRAINT chk_orders_customer_id_not_null;
```

**เหตุผลที่ขั้นตอนที่ 3 เร็ว:** ตั้งแต่ PostgreSQL 12 เป็นต้นมา เมื่อ planner เจอ `SET NOT NULL` มันจะตรวจสอบ `pg_constraint` ก่อนว่ามี CHECK constraint ที่ validate แล้ว (`convalidated = true`) ที่พิสูจน์ว่าคอลัมน์นั้นห้ามเป็น NULL อยู่แล้วหรือไม่ ถ้ามี PostgreSQL จะ**เชื่อ**ผลการ validate นั้นและข้ามการสแกนทั้งตารางซ้ำ ทำให้ `SET NOT NULL` กลายเป็น metadata-only operation ที่เร็วมาก แทนที่จะต้องสแกน 80 ล้านแถวอีกรอบ

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 10:</strong> ระบบ e-commerce ต้องการเปลี่ยน <code>orders.order_status</code> จาก <code>TEXT</code> อิสระ (ใครจะใส่ค่าอะไรก็ได้) ให้กลายเป็นค่าที่ถูกจำกัดเฉพาะ <code>'pending'</code>, <code>'paid'</code>, <code>'shipped'</code>, <code>'cancelled'</code> เท่านั้น โดยไม่ล็อกตารางขนาดใหญ่และไม่ทำให้แอปที่กำลังรันอยู่ error จงออกแบบ migration</summary>

**เฉลย:**

แนวทางที่ปลอดภัยที่สุดคือใช้ **CHECK constraint แบบ NOT VALID → VALIDATE** (ไม่ใช่การเปลี่ยนเป็น native `ENUM` type ตรง ๆ เพราะ `ALTER TYPE ... ADD VALUE` ใช้งานได้ดีสำหรับ "เพิ่ม" ค่าใหม่ แต่การเปลี่ยนคอลัมน์จาก `TEXT` ไปเป็น `ENUM` ต้อง `ALTER COLUMN TYPE` ซึ่ง rewrite ทั้งตาราง เสี่ยงเช่นเดียวกับ Step 787)

```sql
-- ขั้นที่ 1: ตรวจสอบก่อนว่ามีค่าที่ผิดเงื่อนไขอยู่แล้วหรือไม่ (ต้องทำความสะอาดข้อมูลก่อนถ้ามี)
SELECT DISTINCT order_status
FROM orders
WHERE order_status NOT IN ('pending', 'paid', 'shipped', 'cancelled');
-- ถ้ามีผลลัพธ์ ต้องจัดการข้อมูลเหล่านั้นก่อน (แก้ไขหรือ map ไปเป็นค่าใดค่าหนึ่งที่ถูกต้อง)

-- ขั้นที่ 2: เพิ่ม CHECK constraint แบบ NOT VALID (เร็ว ไม่กระทบแอปที่กำลังใช้ค่าที่ถูกต้องอยู่แล้ว)
ALTER TABLE orders
    ADD CONSTRAINT chk_orders_order_status_allowed
    CHECK (order_status IN ('pending', 'paid', 'shipped', 'cancelled')) NOT VALID;

-- ขั้นที่ 3: validate ข้อมูลเดิมทั้งหมด (ไม่บล็อก DML ระหว่างสแกน)
ALTER TABLE orders VALIDATE CONSTRAINT chk_orders_order_status_allowed;
```

**ข้อควรระวัง:** หลังจากขั้นตอนที่ 2 ทันที กฎนี้จะมีผลกับ**แถวใหม่**ทั้งหมดที่ insert/update เข้ามา ดังนั้นต้อง deploy แอปพลิเคชันเวอร์ชันที่รับประกันว่าจะไม่ส่งค่า `order_status` นอกเหนือจาก 4 ค่านี้**ก่อน**รันขั้นตอนที่ 2 เสมอ ไม่เช่นนั้นแอปเก่าที่พยายามเขียนค่าอื่น (เช่น จาก bug หรือ business logic เก่า) จะถูก reject ทันทีด้วย error แบบ:

```
ERROR:  new row for relation "orders" violates check constraint "chk_orders_order_status_allowed"
```

ซึ่งถือเป็นพฤติกรรมที่ตั้งใจ (fail fast ป้องกันข้อมูลเสีย) แต่ทีมต้อง**สื่อสารและเตรียมพร้อม**ก่อนเปิดใช้งานกฎนี้จริงบน production เสมอ

</details>

---

## บทถัดไป

บทนี้เป็นบทสุดท้ายก่อนบทปิดหลักสูตรภาคมืออาชีพ ผู้เรียนได้ครบทุกเทคนิคที่จำเป็นสำหรับดูแลฐานข้อมูล PostgreSQL ระดับ production แล้ว — ตั้งแต่ backup/recovery, replication, high availability, security, ไปจนถึง zero-downtime migration ในบทนี้ ถึงเวลานำทุกอย่างมารวมกันเป็นโปรเจกต์จริง

ไปต่อที่ **[Part 080: Capstone Project — สร้างระบบ SaaS แบบครบวงจร](./part-080-capstone-saas.md)** เพื่อประยุกต์ใช้ความรู้ทั้งหมดในหลักสูตรนี้กับโปรเจกต์ capstone ขนาดใหญ่