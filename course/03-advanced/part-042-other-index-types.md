# Part 042: Index ขั้นสูง — Hash, GiST, GIN, BRIN, SP-GiST

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับสูง | Part 042

---

## เป้าหมายการเรียนรู้

เมื่อเรียนจบบทนี้ ผู้เรียนจะสามารถ:

1. อธิบายภาพรวมของ index access method ทั้งหมดที่ PostgreSQL รองรับ (B-Tree, Hash, GIN, GiST, SP-GiST, BRIN) และเลือกใช้ให้เหมาะกับชนิดข้อมูลและรูปแบบ query
2. เข้าใจโครงสร้างและข้อจำกัดของ **Hash Index** และรู้ว่าเมื่อไรควรใช้แทน B-Tree
3. เข้าใจแนวคิด **Inverted Index (GIN)** และสร้าง index บนคอลัมน์ JSONB เพื่อเร่งความเร็ว containment query (`@>`)
4. เข้าใจแนวคิด **GiST** และนำไปใช้กับ range type รวมถึงสร้าง exclusion constraint เพื่อป้องกันข้อมูลทับซ้อนกัน
5. ใช้ **BRIN** กับข้อมูลขนาดใหญ่ที่เรียงตามลำดับทางกายภาพ เช่น timestamp การสั่งซื้อ พร้อมเข้าใจข้อจำกัดเรื่อง correlation
6. เข้าใจแนวคิด **SP-GiST** และรู้จัก use case เช่น IP address หรือข้อมูลที่กระจายตัวไม่สมดุล
7. ติดตั้งและใช้งาน extension **pg_trgm** เพื่อเร่งความเร็ว fuzzy text search และ `ILIKE '%...%'`
8. วิเคราะห์และเลือก index type ที่เหมาะสมที่สุดสำหรับ query pattern จริงในระบบ e-commerce

---

## เตรียมข้อมูล

บทนี้ใช้ schema ฐานข้อมูล e-commerce ชุดเดียวกับ Part 041 แต่เพิ่มคอลัมน์ใหม่ 2 คอลัมน์เพื่อรองรับตัวอย่าง index ขั้นสูง:

- `products.attributes JSONB` — เก็บคุณสมบัติสินค้าที่หลากหลาย (สี, ไซส์, วัสดุ, แบรนด์) สำหรับตัวอย่าง GIN บน JSONB
- `products.product_name` — ข้อความที่หลากหลายมากขึ้น สำหรับตัวอย่าง GIN/GiST trigram (pg_trgm)
- `customers.last_login TIMESTAMPTZ` — เวลาที่ลูกค้า login ล่าสุด ซึ่งในตัวอย่างนี้เราจะจำลองให้มีลักษณะ "เรียงตามลำดับการ insert" (append-only) เพื่อสาธิต BRIN

> หมายเหตุ: บทนี้ยังไม่ลงลึกเรื่อง JSONB ทั้งหมด (เช่น `jsonb_path_query`, GIN operator class ครบชุด) เพราะเป็นเนื้อหาของ Part 051-052 ในบทนี้เราแค่ "ชิมรส" การใช้ GIN กับ JSONB เพื่อให้เข้าใจภาพรวมของ index type เท่านั้น

### 1. สร้างตาราง

```sql
-- ลบตารางเดิมถ้ามี (สำหรับรันซ้ำในสภาพแวดล้อมฝึกฝน)
DROP TABLE IF EXISTS order_items CASCADE;
DROP TABLE IF EXISTS orders CASCADE;
DROP TABLE IF EXISTS products CASCADE;
DROP TABLE IF EXISTS customers CASCADE;
DROP TABLE IF EXISTS suppliers CASCADE;
DROP TABLE IF EXISTS categories CASCADE;

CREATE TABLE categories (
    category_id         SERIAL PRIMARY KEY,
    category_name       VARCHAR(100) NOT NULL,
    parent_category_id  INTEGER REFERENCES categories(category_id)
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
    stock_quantity  INTEGER NOT NULL DEFAULT 0,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    attributes      JSONB
);

CREATE TABLE customers (
    customer_id     SERIAL PRIMARY KEY,
    first_name      VARCHAR(60) NOT NULL,
    last_name       VARCHAR(60) NOT NULL,
    email           VARCHAR(150) UNIQUE,
    country         VARCHAR(60),
    signup_date     DATE NOT NULL DEFAULT CURRENT_DATE,
    last_login      TIMESTAMPTZ
);

CREATE TABLE orders (
    order_id        SERIAL PRIMARY KEY,
    customer_id     INTEGER REFERENCES customers(customer_id),
    order_date      TIMESTAMPTZ NOT NULL DEFAULT now(),
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',
    ship_country    VARCHAR(60)
);

CREATE TABLE order_items (
    order_item_id   SERIAL PRIMARY KEY,
    order_id        INTEGER REFERENCES orders(order_id),
    product_id      INTEGER REFERENCES products(product_id),
    quantity        INTEGER NOT NULL CHECK (quantity > 0),
    unit_price      NUMERIC(10,2) NOT NULL
);
```

### 2. เติมข้อมูล categories และ suppliers

```sql
-- หมวดหมู่หลัก 10 หมวด
INSERT INTO categories (category_name, parent_category_id)
SELECT 'หมวดหมู่หลัก ' || gs, NULL
FROM generate_series(1, 10) AS gs;

-- หมวดหมู่ย่อยอีก 15 หมวด อ้างอิงหมวดหมู่หลัก
INSERT INTO categories (category_name, parent_category_id)
SELECT 'หมวดหมู่ย่อย ' || gs, 1 + floor(random() * 10)::int
FROM generate_series(11, 25) AS gs;

-- ซัพพลายเออร์ 50 ราย
INSERT INTO suppliers (supplier_name, country)
SELECT
    'ซัพพลายเออร์ ' || gs,
    (ARRAY['Thailand','China','Japan','USA','Germany','Vietnam','South Korea','India'])
        [1 + floor(random() * 8)::int]
FROM generate_series(1, 50) AS gs;
```

### 3. เติมข้อมูลสินค้า พร้อม `attributes JSONB` ที่หลากหลาย

```sql
INSERT INTO products (product_name, category_id, supplier_id, unit_price, stock_quantity, is_active, attributes)
SELECT
    (ARRAY[
        'เสื้อยืดคอกลม', 'กางเกงยีนส์', 'รองเท้าผ้าใบ', 'กระเป๋าเป้สะพายหลัง',
        'หูฟังไร้สาย', 'สมาร์ทวอทช์', 'แว่นกันแดด', 'เข็มขัดหนัง',
        'หมวกแก๊ป', 'ผ้าพันคอ', 'เสื้อแจ็คเก็ต', 'กระเป๋าสตางค์',
        'ลำโพงบลูทูธ', 'พาวเวอร์แบงก์', 'เสื้อฮู้ด', 'รองเท้าแตะ'
    ])[1 + floor(random() * 16)::int]
        || ' รุ่น ' || gs
        || ' ' || (ARRAY['Standard','Premium','Limited Edition','Classic','Pro','Lite'])
                    [1 + floor(random() * 6)::int]
    AS product_name,
    1 + floor(random() * 25)::int AS category_id,
    1 + floor(random() * 50)::int AS supplier_id,
    round((random() * 4900 + 100)::numeric, 2) AS unit_price,
    floor(random() * 500)::int AS stock_quantity,
    (random() > 0.05) AS is_active,
    jsonb_build_object(
        'color', (ARRAY['red','blue','black','white','green','yellow','gray','pink'])
                    [1 + floor(random() * 8)::int],
        'size', (ARRAY['S','M','L','XL','XXL','Free Size'])[1 + floor(random() * 6)::int],
        'material', (ARRAY['cotton','polyester','leather','denim','nylon','canvas'])
                    [1 + floor(random() * 6)::int],
        'brand', 'Brand-' || (1 + floor(random() * 20)::int),
        'is_limited', (random() > 0.85),
        'warranty_months', (ARRAY[0, 3, 6, 12, 24])[1 + floor(random() * 5)::int]
    ) AS attributes
FROM generate_series(1, 900) AS gs;
```

### 4. เติมข้อมูลลูกค้า พร้อม `last_login` ที่เรียงตามลำดับการ insert

เพื่อสาธิต BRIN ให้เห็นผลชัดเจน เราจะจำลองว่าลูกค้าทยอยสมัครและ login เข้าระบบเรียงตามเวลา (ค่า `customer_id` ที่มากขึ้น ↔ `last_login` ที่ช้าลง) ซึ่งเป็นลักษณะทั่วไปของข้อมูล append-only:

```sql
INSERT INTO customers (first_name, last_name, email, country, signup_date, last_login)
SELECT
    'ลูกค้า',
    'หมายเลข' || gs,
    'customer' || gs || '@example.com',
    (ARRAY['Thailand','China','Japan','USA','Germany','Vietnam','South Korea','India'])
        [1 + floor(random() * 8)::int],
    (DATE '2023-01-01' + (gs / 2))::date,
    TIMESTAMPTZ '2023-01-01 00:00:00+07'
        + (gs || ' minutes')::interval
        + (floor(random() * 30) || ' minutes')::interval  -- jitter เล็กน้อยแต่ยังคงลำดับใกล้เคียง
FROM generate_series(1, 500) AS gs;
```

### 5. เติมข้อมูลคำสั่งซื้อ (orders) — เรียงตามเวลาธรรมชาติ

```sql
INSERT INTO orders (customer_id, order_date, status, ship_country)
SELECT
    1 + floor(random() * 500)::int,
    TIMESTAMPTZ '2024-01-01 00:00:00+07' + (gs || ' minutes')::interval,
    (ARRAY['pending','paid','shipped','completed','cancelled'])[1 + floor(random() * 5)::int],
    (ARRAY['Thailand','China','Japan','USA','Germany'])[1 + floor(random() * 5)::int]
FROM generate_series(1, 1200) AS gs;
```

### 6. เติมรายการสินค้าในคำสั่งซื้อ (order_items)

```sql
INSERT INTO order_items (order_id, product_id, quantity, unit_price)
SELECT
    o.order_id,
    1 + floor(random() * 900)::int,
    1 + floor(random() * 5)::int,
    round((random() * 4900 + 100)::numeric, 2)
FROM orders o
CROSS JOIN LATERAL generate_series(1, 1 + floor(random() * 4)::int) AS item_no;
```

### 7. เก็บสถิติให้ query planner

```sql
ANALYZE categories;
ANALYZE suppliers;
ANALYZE products;
ANALYZE customers;
ANALYZE orders;
ANALYZE order_items;
```

ตรวจสอบจำนวนแถว:

```sql
SELECT 'products' AS table_name, count(*) FROM products
UNION ALL SELECT 'customers', count(*) FROM customers
UNION ALL SELECT 'orders', count(*) FROM orders
UNION ALL SELECT 'order_items', count(*) FROM order_items;
```

```
 table_name  | count
-------------+-------
 products    |   900
 customers   |   500
 orders      |  1200
 order_items |  3006
```

ทุกอย่างพร้อมแล้ว มาเริ่มสำรวจ index type ขั้นสูงกันเลย

---

## Step 411: ภาพรวม Index Type ทั้งหมดใน PostgreSQL

PostgreSQL ไม่ได้มี index แบบเดียว แต่มีระบบที่เรียกว่า **Pluggable Index Access Method** ซึ่งหมายความว่าเราเลือก "วิธีสร้างโครงสร้างข้อมูลของ index" ได้หลายแบบ ขึ้นกับชนิดข้อมูลและรูปแบบ query ที่ต้องใช้

ตรวจสอบ access method ที่ระบบรองรับได้จาก catalog `pg_am`:

```sql
SELECT amname, amtype
FROM pg_am
ORDER BY amname;
```

```
 amname | amtype
--------+--------
 brin   | i
 btree  | i
 gin    | i
 gist   | i
 hash   | i
 spgist | i
```

(`amtype = 'i'` หมายถึง index access method — ยังมี `t` สำหรับ table access method เช่น heap ซึ่งเป็นคนละเรื่องกัน)

### ตารางเปรียบเทียบภาพรวม

| Index Type | โครงสร้างข้อมูล | Operator ที่รองรับ | เหมาะกับข้อมูล/สถานการณ์ | ขนาด index | อัปเดตข้อมูล |
|---|---|---|---|---|---|
| **B-Tree** | Balanced tree | `=`, `<`, `<=`, `>`, `>=`, `BETWEEN`, `ORDER BY`, `IS NULL` | ข้อมูลทั่วไป, primary/foreign key, range query, sort | กลาง | เร็ว |
| **Hash** | Hash table | `=` เท่านั้น | equality lookup ล้วนๆ บนคีย์ขนาดใหญ่ | เล็ก-กลาง | เร็ว |
| **GIN** | Inverted index | `@>`, `<@`, `?`, `?|`, `?&`, `&&`, full text `@@` | array, jsonb, full text search, ค่าที่มีหลายรายการต่อแถว | ใหญ่ | ช้า (batch ได้ดีกว่า) |
| **GiST** | Balanced tree (generalized) | `&&`, `<<`, `<->` (KNN), range overlap, geometric | range type, geometric, nearest-neighbor, exclusion constraint | กลาง-ใหญ่ | ปานกลาง |
| **SP-GiST** | Space-partitioned tree (non-balanced) | prefix match, `<<=`, `>>=`, geometric | ข้อมูลกระจายตัวไม่สมดุล เช่น IP address, quad-tree, trie | เล็ก-กลาง | เร็ว |
| **BRIN** | Block Range Index (summary) | `=`, `<`, `<=`, `>`, `>=`, `BETWEEN` | ข้อมูลขนาดใหญ่มากที่เรียงตามลำดับทางกายภาพ เช่น timestamp | เล็กมาก | เร็วมาก (แต่พึ่ง correlation) |

### หลักการเลือกแบบสรุปเร็ว

- ไม่รู้จะใช้อะไร → เริ่มจาก **B-Tree** (default เมื่อไม่ระบุ `USING`)
- คอลัมน์ query ด้วย `=` อย่างเดียว คีย์ยาวมาก และไม่สนใจ sort → พิจารณา **Hash**
- คอลัมน์เก็บ array / JSONB / full text ที่มีหลายค่าในแถวเดียว → **GIN**
- คอลัมน์เป็น range, geometric, หรือ query แบบ "หาใกล้สุด" (nearest-neighbor) → **GiST**
- ข้อมูลกระจายตัวไม่สม่ำเสมอ เช่น IP address, ต้องการ prefix search ที่มีประสิทธิภาพ → **SP-GiST**
- ตารางใหญ่มาก (สิบล้าน–พันล้านแถว) และคอลัมน์มีความสัมพันธ์กับลำดับการ insert เช่น timestamp, id ที่เพิ่มขึ้นเรื่อยๆ → **BRIN**

บทนี้จะพาไปดูทีละตัวพร้อมตัวอย่างจริงบนฐานข้อมูล e-commerce

---

## Step 412: Hash Index

### โครงสร้าง

Hash Index เก็บข้อมูลด้วย **hash table** — เมื่อ insert ค่า PostgreSQL จะคำนวณ hash code ของค่านั้น แล้วเก็บ pointer ไปยังแถวข้อมูลไว้ใน "bucket" ที่สอดคล้องกับ hash code นั้น เมื่อค้นหา ก็แค่คำนวณ hash code ของค่าที่ต้องการแล้วกระโดดไปที่ bucket ตรงๆ (O(1) โดยประมาณ) ต่างจาก B-Tree ที่ต้องไล่ระดับ (O(log n))

ตั้งแต่ PostgreSQL 10 เป็นต้นไป Hash Index เป็น **WAL-logged** แล้ว (ก่อนหน้านั้นไม่ crash-safe และไม่รองรับ replication) จึงใช้งานได้ปลอดภัยใน production ตั้งแต่เวอร์ชันนั้นเป็นต้นมา

### สร้าง Hash Index

```sql
CREATE INDEX idx_orders_status_hash
    ON orders USING hash (status);
```

ทดสอบ:

```sql
EXPLAIN ANALYZE
SELECT order_id, customer_id, order_date, status
FROM orders
WHERE status = 'shipped';
```

```
                                                     QUERY PLAN
---------------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on orders  (cost=4.53..38.21 rows=240 width=24) (actual time=0.045..0.198 rows=238 loops=1)
   Recheck Cond: (status = 'shipped'::text)
   Heap Blocks: exact=97
   ->  Bitmap Index Scan on idx_orders_status_hash  (cost=0.00..4.47 rows=240 width=0)
         (actual time=0.028..0.028 rows=238 loops=1)
         Index Cond: (status = 'shipped'::text)
 Planning Time: 0.112 ms
 Execution Time: 0.231 ms
```

Planner เลือกใช้ hash index ผ่าน Bitmap Index Scan สำหรับ equality condition ได้อย่างมีประสิทธิภาพ

### ข้อจำกัดเทียบกับ B-Tree

| หัวข้อ | Hash | B-Tree |
|---|---|---|
| Operator ที่รองรับ | `=` เท่านั้น | `=`, `<`, `>`, `<=`, `>=`, `BETWEEN`, `LIKE 'prefix%'` |
| รองรับ `ORDER BY` | ไม่ได้ | ได้ |
| Multi-column index | ไม่ได้ (single column เท่านั้น) | ได้ |
| ใช้เป็น UNIQUE/PRIMARY KEY constraint | ไม่ได้ | ได้ |
| Index-only scan | ไม่รองรับ | รองรับ |
| ขนาด index เทียบกับคีย์ยาวๆ | เล็กกว่า (เก็บแค่ hash code ไม่ใช่ค่าเต็ม) | ใหญ่กว่าถ้าคีย์ยาวมาก |

ลองเทียบ query ที่เป็น range เพื่อดูว่า hash index ใช้ไม่ได้:

```sql
EXPLAIN SELECT * FROM orders WHERE status > 'p';
```

```
                              QUERY PLAN
----------------------------------------------------------------
 Seq Scan on orders  (cost=0.00..31.00 rows=680 width=44)
   Filter: (status > 'p'::text)
```

Planner **ไม่สามารถใช้ `idx_orders_status_hash`** กับ operator `>` ได้เลย ต้องหันไป Seq Scan แทน — นี่คือข้อจำกัดสำคัญที่สุดของ Hash Index

### สรุปเมื่อไรควรใช้ Hash Index

ในทางปฏิบัติ Hash Index มีประโยชน์จำกัดมากเมื่อเทียบกับ B-Tree เพราะ B-Tree ก็รองรับ `=` ได้ดีอยู่แล้ว (แค่ไม่ O(1) เป๊ะ) และยังรองรับ operator อื่นด้วย สถานการณ์ที่ Hash Index อาจคุ้มค่า:

- คอลัมน์เป็น **string ยาวมาก** (เช่น hash, UUID แบบ text, URL) และ query ใช้ `=` **เท่านั้น** ไม่เคย range/sort — hash index จะมีขนาดเล็กกว่าและ lookup อาจเร็วกว่าเล็กน้อย
- ต้องการลดขนาด index บนตารางใหญ่มากที่ query แบบ equality อย่างเดียว

ในกรณีทั่วไป **แนะนำให้ใช้ B-Tree เป็นค่าเริ่มต้น** เว้นแต่จะพิสูจน์แล้วด้วย benchmark ว่า Hash Index ให้ประโยชน์จริง

---

## Step 413: GIN (Generalized Inverted Index) — แนวคิด

### ปัญหาที่ B-Tree แก้ไม่ได้ดี

B-Tree และ Hash ถูกออกแบบมาสำหรับ **หนึ่งค่าต่อหนึ่งแถว** (scalar value) แต่ในโลกจริงเรามักเจอคอลัมน์ที่ **หนึ่งแถวมีได้หลายค่า** เช่น:

- Array: `tags TEXT[]` — หนึ่งสินค้ามีหลาย tag
- JSONB: `attributes JSONB` — หนึ่งแถวมีหลาย key-value
- Full text search: `tsvector` — หนึ่งเอกสารมีหลาย lexeme (คำ)

ถ้าใช้ B-Tree กับข้อมูลแบบนี้ เราจะสร้าง index ได้แค่ "ทั้งค่า" (เช่น array ทั้งก้อน) ซึ่งใช้กับ query แบบ "มีค่านี้อยู่ในนั้นหรือไม่" ไม่ได้เลย

### แนวคิด Inverted Index

GIN แก้ปัญหานี้ด้วยแนวคิด **Inverted Index** ซึ่งเป็นแนวคิดเดียวกับที่ search engine ใช้:

- **B-Tree**: 1 entry ต่อ 1 แถว → entry ชี้กลับไปที่แถวเดียว
- **GIN**: 1 entry ต่อ 1 "องค์ประกอบย่อย" (key ของ jsonb, element ของ array, lexeme ของ tsvector) → entry ชี้ไปที่ **รายการของแถวทั้งหมด** ที่มีองค์ประกอบนั้น (เรียกว่า posting list)

ตัวอย่างเชิงแนวคิด ถ้ามีสินค้า 3 ชิ้นที่มี `tags`:

```
product 1: tags = {red, cotton}
product 2: tags = {blue, cotton}
product 3: tags = {red, denim}
```

GIN จะสร้างโครงสร้างประมาณนี้:

```
"red"    -> [product 1, product 3]
"blue"   -> [product 2]
"cotton" -> [product 1, product 2]
"denim"  -> [product 3]
```

เมื่อ query `WHERE tags @> ARRAY['red']` ระบบแค่ไปดู posting list ของ `"red"` ได้ทันที ไม่ต้องสแกนทุกแถว

### คุณสมบัติสำคัญของ GIN

- เหมาะกับข้อมูลที่มีความ "หลายต่อหนึ่ง" (multi-valued per row)
- ขนาด index มักใหญ่กว่า B-Tree เพราะเก็บ entry ต่อองค์ประกอบ ไม่ใช่ต่อแถว
- การ **insert/update ช้ากว่า** B-Tree มาก เพราะต้อง maintain posting list หลายรายการต่อการเปลี่ยนแปลง 1 แถว — PostgreSQL แก้ปัญหานี้บางส่วนด้วย `gin_pending_list_limit` (buffer การเขียนแบบ batch แล้วค่อย merge)
- **การอ่าน (SELECT) เร็วมาก** สำหรับ containment query

ดู operator class ที่รองรับ GIN:

```sql
SELECT opcname, opcintype::regtype
FROM pg_opclass
WHERE opcmethod = (SELECT oid FROM pg_am WHERE amname = 'gin')
ORDER BY opcname;
```

```
      opcname       | opcintype
---------------------+-----------
 array_ops           | anyarray
 jsonb_ops           | jsonb
 jsonb_path_ops      | jsonb
 tsvector_ops        | tsvector
 gin_trgm_ops        | text        <- มาจาก extension pg_trgm (Step 419)
 ...
```

Step ถัดไปเราจะลงมือสร้าง GIN index บนคอลัมน์ JSONB จริง

---

## Step 414: ตัวอย่าง GIN บน JSONB Column (attributes)

### สร้าง GIN Index แบบ default (jsonb_ops)

```sql
CREATE INDEX idx_products_attributes_gin
    ON products USING gin (attributes);
```

Operator class เริ่มต้นของ `gin` บน jsonb คือ `jsonb_ops` ซึ่ง index ทั้ง **key และ value** ของทุกระดับใน JSON object ทำให้รองรับ operator ได้หลากหลาย:

- `@>` — containment (ค่าซ้ายมีค่าขวาอยู่ข้างในหรือไม่)
- `?` — มี key นี้อยู่ระดับบนสุดหรือไม่
- `?|` — มี key ใดๆ ในลิสต์นี้อยู่หรือไม่
- `?&` — มีทุก key ในลิสต์นี้อยู่หรือไม่

### ทดสอบ containment query `@>`

```sql
EXPLAIN ANALYZE
SELECT product_id, product_name, attributes
FROM products
WHERE attributes @> '{"color": "red"}';
```

**ก่อนมี index** (สมมติถ้ายังไม่ได้สร้าง):

```
                                     QUERY PLAN
------------------------------------------------------------------------------------
 Seq Scan on products  (cost=0.00..47.75 rows=5 width=98)
                        (actual time=0.021..1.842 rows=113 loops=1)
   Filter: (attributes @> '{"color": "red"}'::jsonb)
   Rows Removed by Filter: 787
 Planning Time: 0.089 ms
 Execution Time: 1.879 ms
```

**หลังมี GIN index**:

```
                                                    QUERY PLAN
--------------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on products  (cost=8.02..29.15 rows=5 width=98) (actual time=0.052..0.121 rows=113 loops=1)
   Recheck Cond: (attributes @> '{"color": "red"}'::jsonb)
   Heap Blocks: exact=58
   ->  Bitmap Index Scan on idx_products_attributes_gin  (cost=0.00..8.02 rows=5 width=0)
         (actual time=0.031..0.031 rows=113 loops=1)
         Index Cond: (attributes @> '{"color": "red"}'::jsonb)
 Planning Time: 0.098 ms
 Execution Time: 0.156 ms
```

จาก ~1.9 ms เหลือ ~0.16 ms (ในตารางเล็กความต่างยังไม่มาก แต่บนตารางหลักล้านแถวผลต่างจะชัดเจนมาก เพราะ Seq Scan โต O(n) ในขณะที่ Bitmap Index Scan โตช้ากว่ามาก)

ลองทดสอบ operator `?` (มี key นี้หรือไม่):

```sql
EXPLAIN ANALYZE
SELECT product_id, attributes
FROM products
WHERE attributes ? 'is_limited';
```

```
                                                QUERY PLAN
------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on products  (cost=4.51..30.02 rows=450 width=98) (actual time=0.041..0.198 rows=900 loops=1)
   Recheck Cond: (attributes ? 'is_limited'::text)
   Heap Blocks: exact=112
   ->  Bitmap Index Scan on idx_products_attributes_gin  (cost=0.00..4.40 rows=450 width=0)
 Planning Time: 0.076 ms
 Execution Time: 0.241 ms
```

(ทุกแถวมี key `is_limited` เพราะเราสร้างด้วย `jsonb_build_object` เหมือนกันหมด — ในข้อมูลจริงที่ schema ไม่ตายตัว query นี้จะมีประโยชน์มากกว่า)

### `jsonb_path_ops` — GIN opclass อีกแบบที่เล็กและเร็วกว่า

```sql
CREATE INDEX idx_products_attributes_pathops
    ON products USING gin (attributes jsonb_path_ops);
```

`jsonb_path_ops` **รองรับเฉพาะ `@>`** (ไม่รองรับ `?`, `?|`, `?&`) แต่แลกมาด้วยข้อดี:

- index มีขนาดเล็กกว่า `jsonb_ops` มาก (เพราะเก็บ hash ของทั้ง path แทนที่จะเก็บ key/value แยกกัน)
- performance ของ `@>` เร็วกว่า `jsonb_ops` โดยเฉพาะเมื่อ query มีเงื่อนไขซับซ้อน (หลาย key พร้อมกัน)

เปรียบเทียบขนาด index:

```sql
SELECT
    indexname,
    pg_size_pretty(pg_relation_size(indexname::regclass)) AS index_size
FROM pg_indexes
WHERE tablename = 'products'
  AND indexname LIKE 'idx_products_attributes%';
```

```
             indexname              | index_size
-------------------------------------+------------
 idx_products_attributes_gin        | 128 kB
 idx_products_attributes_pathops    | 64 kB
```

`jsonb_path_ops` เล็กกว่าประมาณครึ่งหนึ่งในตัวอย่างนี้ และยิ่งข้อมูลใหญ่ขึ้นความต่างยิ่งชัดเจน

### สรุปการเลือก JSONB GIN opclass

| ต้องการ | ใช้ opclass |
|---|---|
| ต้องการ `@>` เท่านั้น เน้นขนาดเล็ก/เร็ว | `jsonb_path_ops` |
| ต้องการ `?`, `?|`, `?&` ด้วย | `jsonb_ops` (default) |
| ต้องการทั้งสองแบบพร้อมกัน | สร้าง 2 index แยกกัน (ตามที่สาธิตข้างต้น) |

> เนื้อหาการดัชนี JSONB แบบลึก เช่น indexing เฉพาะบาง path ด้วย expression index, `jsonb_path_query`, GIN บน array ของ jsonb ฯลฯ จะกล่าวถึงอย่างละเอียดใน **Part 051-052**

---

## Step 415: GiST (Generalized Search Tree) — แนวคิด

### GiST คืออะไร

GiST เป็น **balanced tree แบบทั่วไป (generalized)** ที่คล้าย B-Tree ในเชิงโครงสร้าง (มี root, internal node, leaf node) แต่ต่างตรงที่ B-Tree ผูกติดกับ "การเรียงลำดับเชิงเส้น" (`<`, `=`, `>`) เท่านั้น ในขณะที่ GiST เปิดให้ผู้พัฒนา operator class กำหนด logic ของตัวเองผ่าน 7 support function หลัก เช่น `consistent`, `union`, `penalty`, `picksplit`, `same` ทำให้ GiST รองรับชนิดข้อมูลที่ "ไม่มีการเรียงลำดับเชิงเส้นตายตัว" ได้ เช่น:

- **ข้อมูลเชิงพื้นที่ (geometric/spatial)**: จุด, เส้น, รูปหลายเหลี่ยม, PostGIS geometry
- **Range types**: `int4range`, `tsrange`, `tstzrange`, `numrange`, `daterange`
- **Full text search** (`tsvector`) — เป็นทางเลือกแทน GIN (เล็กกว่าแต่ lossy/ช้ากว่าตอนค้นหา)
- **Nearest-neighbor search (KNN)** ผ่าน operator `<->` เช่น "หาจุด 5 จุดที่ใกล้กับจุดนี้ที่สุด"

### ความแตกต่างสำคัญ: Lossy Index

GiST index เป็น **lossy** ได้ (ต่างจาก B-Tree ที่ไม่ lossy) หมายความว่า index อาจเก็บข้อมูลแบบ "โดยประมาณ" (เช่น bounding box ของรูปทรงแทนที่จะเก็บรูปทรงทั้งหมด) แล้วให้ PostgreSQL **recheck** เงื่อนไขจริงจากข้อมูลในตาราง (heap) อีกครั้งหลัง scan index — นี่คือเหตุผลที่เรามักเห็น `Recheck Cond:` ใน EXPLAIN ของ Bitmap scan ที่ใช้ GiST/GIN

### ดู operator class ของ GiST

```sql
SELECT opcname, opcintype::regtype
FROM pg_opclass
WHERE opcmethod = (SELECT oid FROM pg_am WHERE amname = 'gist')
ORDER BY opcname
LIMIT 15;
```

```
      opcname       |  opcintype
---------------------+--------------
 box_ops             | box
 circle_ops          | circle
 point_ops           | point
 poly_ops            | polygon
 range_ops           | anyrange
 tsvector_ops        | tsvector
 gist_trgm_ops       | text          <- จาก pg_trgm
 inet_ops            | inet
 ...
```

สังเกตว่า `range_ops` รองรับทุก range type (`anyrange`) — นี่คือหัวข้อที่เราจะลงตัวอย่างจริงใน Step ถัดไป

---

## Step 416: GiST กับ Range Type และ Exclusion Constraint

### ทบทวนเชื่อมโยง Part 018

ใน Part 018 เราได้เรียนเรื่อง **Range Types** (`int4range`, `tstzrange`, ฯลฯ) และ **Exclusion Constraints** ไปแล้วในเชิงแนวคิด บทนี้เราจะมาดูว่า **เบื้องหลัง exclusion constraint ที่ใช้กับ range นั้น PostgreSQL ใช้ GiST index ทำงานอยู่**

### สถานการณ์: โปรโมชันสินค้าห้ามช่วงเวลาทับซ้อนกัน

สมมติร้านค้าต้องการสร้างตารางโปรโมชันสินค้า โดยมีกฎว่า **สินค้าชิ้นเดียวกันห้ามมีโปรโมชัน 2 รายการที่ช่วงเวลาทับซ้อนกัน**

```sql
CREATE TABLE product_promotions (
    promotion_id      SERIAL PRIMARY KEY,
    product_id        INTEGER NOT NULL REFERENCES products(product_id),
    promo_period       TSTZRANGE NOT NULL,
    discount_percent   NUMERIC(5,2) NOT NULL CHECK (discount_percent BETWEEN 0 AND 90),
    EXCLUDE USING gist (product_id WITH =, promo_period WITH &&)
);
```

เมื่อสร้างตารางนี้ PostgreSQL จะสร้าง **GiST index** ให้อัตโนมัติเบื้องหลัง `EXCLUDE` constraint เพื่อตรวจสอบว่าไม่มีคู่แถวใดที่ `product_id` เท่ากัน **และ** `promo_period` ทับซ้อนกัน (`&&`) พร้อมกัน

ตรวจสอบ index ที่ถูกสร้างอัตโนมัติ:

```sql
SELECT indexname, indexdef
FROM pg_indexes
WHERE tablename = 'product_promotions';
```

```
              indexname               |                                    indexdef
---------------------------------------+---------------------------------------------------------------------------------
 product_promotions_pkey               | CREATE UNIQUE INDEX product_promotions_pkey ON product_promotions USING btree (promotion_id)
 product_promotions_product_id_promo_period_excl | CREATE INDEX ... USING gist (product_id, promo_period gist_range_ops_2)
```

### ทดสอบ insert ข้อมูล

```sql
INSERT INTO product_promotions (product_id, promo_period, discount_percent) VALUES
    (1, TSTZRANGE('2026-01-01', '2026-01-15'), 10.00),
    (1, TSTZRANGE('2026-02-01', '2026-02-10'), 15.00),
    (2, TSTZRANGE('2026-01-01', '2026-01-31'), 20.00);
```

ลอง insert โปรโมชันของ product_id = 1 ที่ช่วงเวลาทับกับรายการแรก:

```sql
INSERT INTO product_promotions (product_id, promo_period, discount_percent)
VALUES (1, TSTZRANGE('2026-01-10', '2026-01-20'), 25.00);
```

```
ERROR:  conflicting key value violates exclusion constraint
        "product_promotions_product_id_promo_period_excl"
DETAIL:  Key (product_id, promo_period)=(1, ["2026-01-10 00:00:00+07","2026-01-20 00:00:00+07"))
         conflicts with existing key
         (product_id, promo_period)=(1, ["2026-01-01 00:00:00+07","2026-01-15 00:00:00+07")).
```

Database ปฏิเสธการ insert ให้อัตโนมัติ — ไม่ต้องเขียน application logic มาเช็คเอง!

### GiST ช่วยเร่ง query แบบ overlap ด้วย

Index เดียวกันนี้ยังใช้เร่ง query ที่ค้นหา "โปรโมชันที่ทับซ้อนกับช่วงเวลาที่กำหนด" ได้ด้วย:

```sql
EXPLAIN ANALYZE
SELECT promotion_id, product_id, promo_period, discount_percent
FROM product_promotions
WHERE promo_period && TSTZRANGE('2026-01-05', '2026-01-12');
```

```
                                                       QUERY PLAN
--------------------------------------------------------------------------------------------------------------------
 Index Scan using product_promotions_product_id_promo_period_excl on product_promotions
     (cost=0.14..8.16 rows=1 width=44) (actual time=0.024..0.026 rows=1 loops=1)
   Index Cond: (promo_period && '["2026-01-05 00:00:00+07","2026-01-12 00:00:00+07")'::tstzrange)
 Planning Time: 0.095 ms
 Execution Time: 0.048 ms
```

Planner ใช้ GiST index สแกนหาช่วงที่ overlap ได้โดยตรง ไม่ต้อง Seq Scan ทุกแถวแล้วมาเช็คทีละคู่

### Nearest-neighbor (KNN) ด้วย GiST — ตัวอย่างแนวคิด

GiST ยังรองรับ operator `<->` สำหรับ query แบบ "หาที่ใกล้ที่สุด" เช่น ถ้ามีคอลัมน์ point (พิกัด):

```sql
-- ตัวอย่างแนวคิด (ไม่ใช้กับ schema ของบทนี้)
-- SELECT store_id, location <-> point(100.53, 13.75) AS distance
-- FROM stores
-- ORDER BY location <-> point(100.53, 13.75)
-- LIMIT 5;
```

รูปแบบนี้เรียกว่า **KNN-GiST** (K-Nearest Neighbor) ซึ่งเป็นความสามารถเฉพาะของ GiST ที่ B-Tree ทำไม่ได้เลย เรื่องนี้จะกล่าวถึงอีกครั้งอย่างละเอียดเมื่อพูดถึง PostGIS และ trigram similarity search (Step 419)

---

## Step 417: BRIN (Block Range Index)

### แนวคิด: Summary Index ไม่ใช่ Detailed Index

BRIN แตกต่างจาก index ทุกแบบที่ผ่านมาแบบสุดขั้ว — แทนที่จะเก็บ entry ต่อแถว (หรือต่อองค์ประกอบ) BRIN จะแบ่งตารางออกเป็น **"ช่วงของ block"** (block range เช่น ทุก 128 page/block) แล้วเก็บแค่ **สรุปสถิติ** ของแต่ละช่วง เช่น ค่า **min/max** ของคอลัมน์ในช่วงนั้น

ตัวอย่างเชิงแนวคิด สมมติ `order_date` เรียงตามลำดับการ insert (เพราะเป็นตาราง append-only):

```
Block range 1 (page 0-127):    order_date อยู่ระหว่าง 2024-01-01 ถึง 2024-01-02
Block range 2 (page 128-255):  order_date อยู่ระหว่าง 2024-01-02 ถึง 2024-01-03
Block range 3 (page 256-383):  order_date อยู่ระหว่าง 2024-01-03 ถึง 2024-01-04
...
```

เมื่อ query `WHERE order_date BETWEEN '2024-01-02' AND '2024-01-03'` PostgreSQL แค่ดู summary แล้ว **ข้าม (skip)** block range ที่ไม่เกี่ยวข้องไปเลย โดยไม่ต้องอ่านข้อมูลจริงในช่วงนั้นแม้แต่ page เดียว

### ข้อกำหนดสำคัญ: Correlation

BRIN จะได้ผลดี **ก็ต่อเมื่อ** ค่าของคอลัมน์มีความสัมพันธ์ (correlation) กับ**ตำแหน่งทางกายภาพ**ของแถวในตาราง — พูดง่ายๆ คือค่าต้อง "เรียงตามลำดับการจัดเก็บ" ไม่มากก็น้อย สถานการณ์ที่เหมาะสมที่สุดคือ **append-only log**: `order_date`, `created_at`, `log_timestamp` ที่ insert เรียงเวลาไปเรื่อยๆ และไม่ค่อยมีการ UPDATE ย้อนหลัง

ตรวจสอบ correlation ได้จาก `pg_stats`:

```sql
SELECT attname, correlation
FROM pg_stats
WHERE tablename = 'orders' AND attname = 'order_date';
```

```
  attname   | correlation
-------------+-------------
 order_date  |         1.0
```

ค่า correlation ใกล้ `1.0` (หรือ `-1.0`) หมายถึงเรียงตามลำดับทางกายภาพเกือบสมบูรณ์แบบ — เหมาะกับ BRIN มาก ถ้า correlation ใกล้ `0` แปลว่าข้อมูลกระจัดกระจาย BRIN จะไม่ได้ประโยชน์อะไรเลย (ต้องอ่านเกือบทุก block range อยู่ดี)

### สร้าง BRIN Index บน orders.order_date

```sql
CREATE INDEX idx_orders_order_date_brin
    ON orders USING brin (order_date);
```

### เปรียบเทียบขนาด index: BRIN vs B-Tree

```sql
CREATE INDEX idx_orders_order_date_btree
    ON orders USING btree (order_date);

SELECT indexname, pg_size_pretty(pg_relation_size(indexname::regclass)) AS index_size
FROM pg_indexes
WHERE tablename = 'orders'
  AND indexname LIKE 'idx_orders_order_date%';
```

```
           indexname            | index_size
----------------------------------+------------
 idx_orders_order_date_brin       | 24 kB
 idx_orders_order_date_btree      | 48 kB
```

แม้ในตารางเล็ก (1,200 แถว) ความต่างก็เริ่มเห็น — แต่ **ในตารางจริงระดับสิบ/ร้อยล้านแถว** BRIN index มักมีขนาดแค่ **ไม่กี่สิบ KB ถึงไม่กี่ MB** ในขณะที่ B-Tree index บนคอลัมน์เดียวกันอาจมีขนาดหลาย **GB** เพราะ BRIN ไม่ได้เก็บ entry ต่อแถวเลย

### ทดสอบ EXPLAIN ANALYZE

```sql
EXPLAIN ANALYZE
SELECT order_id, customer_id, order_date, status
FROM orders
WHERE order_date BETWEEN '2024-01-05' AND '2024-01-06';
```

```
                                                      QUERY PLAN
------------------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on orders  (cost=8.53..25.10 rows=127 width=44) (actual time=0.061..0.089 rows=126 loops=1)
   Recheck Cond: ((order_date >= '2024-01-05 00:00:00+07') AND (order_date <= '2024-01-06 00:00:00+07'))
   Rows Removed by Index Recheck: 3
   Heap Blocks: lossy=6
   ->  Bitmap Index Scan on idx_orders_order_date_brin  (cost=0.00..8.51 rows=142 width=0)
         (actual time=0.032..0.032 rows=60 loops=1)
         Index Cond: ((order_date >= '2024-01-05 00:00:00+07') AND (order_date <= '2024-01-06 00:00:00+07'))
 Planning Time: 0.088 ms
 Execution Time: 0.121 ms
```

สังเกต `Heap Blocks: lossy=6` — นี่เป็นลักษณะเฉพาะของ BRIN: มันบอกแค่ว่า "block range นี้ **อาจ** มีค่าที่ตรงเงื่อนไข" (lossy) ระบบเลยต้องอ่านทั้ง block range นั้นมา **recheck** เงื่อนไขจริงอีกที (`Rows Removed by Index Recheck: 3`) ต่างจาก B-Tree ที่ชี้ตรงไปยังแถวที่ตรงเงื่อนไขแบบแม่นยำ

เทียบกับ B-Tree บนคอลัมน์เดียวกัน:

```sql
DROP INDEX idx_orders_order_date_brin;  -- ปิดใช้ BRIN ชั่วคราวเพื่อบังคับให้ใช้ B-Tree

EXPLAIN ANALYZE
SELECT order_id, customer_id, order_date, status
FROM orders
WHERE order_date BETWEEN '2024-01-05' AND '2024-01-06';
```

```
                                                    QUERY PLAN
------------------------------------------------------------------------------------------------------------------
 Index Scan using idx_orders_order_date_btree on orders
     (cost=0.29..12.84 rows=127 width=44) (actual time=0.019..0.058 rows=126 loops=1)
   Index Cond: ((order_date >= '2024-01-05 00:00:00+07') AND (order_date <= '2024-01-06 00:00:00+07'))
 Planning Time: 0.081 ms
 Execution Time: 0.083 ms
```

ในตารางเล็ก B-Tree เร็วกว่าเล็กน้อยเพราะไม่ต้อง recheck — **แต่ BRIN ชนะขาดในเรื่องขนาด index และความเร็วในการ insert/maintain** บนตารางที่ใหญ่มากจนไม่สามารถเก็บ B-Tree index ทั้งก้อนไว้ใน memory (shared_buffers) ได้อีกต่อไป

> ในทางปฏิบัติ BRIN เหมาะกับตาราง log/audit/transaction ที่มีขนาดหลักสิบ-ร้อยล้านแถวขึ้นไป ถ้าตารางเล็ก B-Tree มักจะดีกว่าเสมอ

### ทดสอบกับ customers.last_login

```sql
CREATE INDEX idx_customers_last_login_brin
    ON customers USING brin (last_login);

EXPLAIN ANALYZE
SELECT customer_id, first_name, last_login
FROM customers
WHERE last_login >= now() - interval '1000 minutes';
```

```
                                                  QUERY PLAN
----------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on customers  (cost=2.15..14.30 rows=42 width=32) (actual time=0.028..0.045 rows=41 loops=1)
   Recheck Cond: (last_login >= (now() - '01:00:00'::interval))
   Heap Blocks: lossy=2
   ->  Bitmap Index Scan on idx_customers_last_login_brin  (cost=0.00..2.14 rows=48 width=0)
 Planning Time: 0.072 ms
 Execution Time: 0.068 ms
```

### การปรับ `pages_per_range`

ค่า default ของ block range คือ 128 page ต่อ 1 summary entry เราปรับได้ด้วย storage parameter:

```sql
CREATE INDEX idx_orders_order_date_brin_fine
    ON orders USING brin (order_date)
    WITH (pages_per_range = 32);
```

- `pages_per_range` **น้อยลง** → summary ละเอียดขึ้น (แม่นยำขึ้น, false positive น้อยลง) แต่ index **ใหญ่ขึ้น**
- `pages_per_range` **มากขึ้น** → index เล็กลงมาก แต่ต้อง recheck ข้อมูลเยอะขึ้นต่อ 1 range (lossy มากขึ้น)

### ข้อควรระวัง: correlation เสื่อมลงเมื่อมี UPDATE แทรก

ถ้าตารางมีการ `UPDATE` ค่าของคอลัมน์ที่ทำ BRIN บ่อยๆ (เช่น `last_login` ที่อัปเดตทุกครั้งที่ลูกค้า login จริง ไม่ใช่แค่ insert ครั้งเดียวแบบในตัวอย่างนี้) แถวจะถูกย้ายไปเก็บที่ block ใหม่ (เพราะ MVCC สร้างแถวใหม่ทุกครั้งที่ UPDATE) ทำให้ correlation ลดลงเรื่อยๆ ตามเวลา วิธีรับมือ:

- รัน `VACUUM` สม่ำเสมอ (ไม่ได้แก้ correlation แต่ช่วยเรื่องพื้นที่)
- ใช้ `CLUSTER` หรือ `pg_repack` เพื่อจัดเรียงตารางใหม่ตามคอลัมน์นั้นเป็นระยะ (มี downtime หรือ lock พิจารณาดีๆ)
- สำหรับคอลัมน์ที่ update บ่อยและต้องการ index ที่แม่นยำ ให้ใช้ B-Tree แทน — **BRIN เหมาะกับคอลัมน์ที่ insert ครั้งเดียวแล้วไม่ (หรือแทบไม่) แก้ไข** เช่น `created_at`, `order_date` มากกว่า `last_login` ที่อัปเดตบ่อย (ในตัวอย่างของบทนี้เราจำลอง `last_login` แบบ insert-only เพื่อการสาธิตเท่านั้น)

---

## Step 418: SP-GiST (Space-Partitioned GiST)

### แนวคิด: Non-Balanced Tree

SP-GiST ย่อมาจาก **Space-Partitioned Generalized Search Tree** ต่างจาก B-Tree และ GiST ตรงที่ SP-GiST เป็น tree แบบ **ไม่ balance (non-balanced)** — โครงสร้างแบ่งพื้นที่ข้อมูล (partition the space) แบบ recursive คล้ายกับ:

- **Quad-tree**: แบ่งพื้นที่ 2 มิติออกเป็น 4 ส่วนซ้ำๆ (ใช้กับข้อมูลเชิงพื้นที่)
- **k-d tree**: แบ่งพื้นที่หลายมิติ
- **Radix tree / Trie**: แบ่งตาม prefix ของ string (ใช้กับ text, IP address)

SP-GiST เหมาะกับข้อมูลที่มี **โครงสร้างตามธรรมชาติแบบไม่สม่ำเสมอ (non-balanced/skewed)** เช่น ข้อมูลที่กระจุกตัวในบางพื้นที่หนาแน่นกว่าพื้นที่อื่นมาก ซึ่ง balanced tree อย่าง B-Tree/GiST จะไม่ efficient เท่า

### Use case ที่พบบ่อย: IP Address

`inet`/`cidr` เป็นตัวอย่างคลาสสิกของ SP-GiST เพราะ IP address มีโครงสร้างแบบ hierarchical (prefix-based) ตามธรรมชาติ — เหมาะกับ radix tree partitioning

สร้างตาราง log การ login เพื่อสาธิต:

```sql
CREATE TABLE login_audit (
    audit_id      BIGSERIAL PRIMARY KEY,
    customer_id   INTEGER REFERENCES customers(customer_id),
    login_ip      INET NOT NULL,
    login_time    TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- เติมข้อมูลจำลอง 3,000 รายการ (IP กระจายในหลาย subnet)
INSERT INTO login_audit (customer_id, login_ip, login_time)
SELECT
    1 + floor(random() * 500)::int,
    (
        (ARRAY['203.150', '110.164', '49.230', '1.10', '124.122'])[1 + floor(random() * 5)::int]
        || '.' || floor(random() * 256)::int
        || '.' || floor(random() * 256)::int
    )::inet,
    now() - (floor(random() * 90) || ' days')::interval
FROM generate_series(1, 3000) AS gs;

ANALYZE login_audit;
```

สร้าง SP-GiST index:

```sql
CREATE INDEX idx_login_audit_ip_spgist
    ON login_audit USING spgist (login_ip inet_ops);
```

ทดสอบ query แบบ "IP นี้อยู่ใน subnet ไหนหรือไม่" (`<<=` คือ "is contained by or equal to"):

```sql
EXPLAIN ANALYZE
SELECT audit_id, customer_id, login_ip, login_time
FROM login_audit
WHERE login_ip <<= inet '203.150.0.0/16';
```

```
                                                      QUERY PLAN
--------------------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on login_audit  (cost=6.31..45.20 rows=580 width=32) (actual time=0.058..0.201 rows=612 loops=1)
   Recheck Cond: (login_ip <<= '203.150.0.0/16'::inet)
   Heap Blocks: exact=142
   ->  Bitmap Index Scan on idx_login_audit_ip_spgist  (cost=0.00..6.17 rows=580 width=0)
         (actual time=0.038..0.038 rows=612 loops=1)
 Planning Time: 0.101 ms
 Execution Time: 0.234 ms
```

เทียบกับตอนไม่มี index (Seq Scan ต้องคำนวณ subnet containment ทุกแถว) SP-GiST ช่วยให้ query แบบ subnet เร็วขึ้นมาก เพราะ radix tree partition ตาม prefix ของ IP ได้อย่างเป็นธรรมชาติ — B-Tree ทำได้ไม่ดีเท่าเพราะการเปรียบเทียบ subnet containment ไม่ใช่การเรียงลำดับเชิงเส้นแบบง่ายๆ

### Use case อื่นๆ ของ SP-GiST

| Data Type | Opclass | ใช้ทำอะไร |
|---|---|---|
| `inet`/`cidr` | `inet_ops` | ค้นหา IP/subnet containment |
| `text` | `text_ops` | Prefix search แบบ radix tree (คล้าย trie) เร็วกว่า B-Tree เมื่อ string มี prefix ซ้ำกันเยอะ |
| `point` | `kd_point_ops`, `quad_point_ops` | ข้อมูลพิกัด 2 มิติที่กระจายตัวไม่สม่ำเสมอ |
| `range` type | `range_ops` | ทางเลือกแทน GiST สำหรับบาง use case |

ตัวอย่าง prefix search ด้วย SP-GiST บน text (แนวคิด):

```sql
-- ตัวอย่างแนวคิด: ถ้ามีคอลัมน์ SKU ที่ prefix ซ้ำกันเยอะ เช่น "TH-ELEC-", "TH-CLOTH-"
-- CREATE INDEX idx_products_sku_spgist ON products USING spgist (sku text_ops);
-- SELECT * FROM products WHERE sku LIKE 'TH-ELEC-%';
```

### SP-GiST vs GiST — ต่างกันอย่างไร

| หัวข้อ | GiST | SP-GiST |
|---|---|---|
| โครงสร้าง | Balanced tree | Non-balanced (space-partitioned) tree |
| เหมาะกับข้อมูลกระจายสม่ำเสมอ | ดี | อาจไม่ efficient เท่า (partition ไม่สมดุล) |
| เหมาะกับข้อมูลกระจายไม่สม่ำเสมอ/มี hierarchy | พอใช้ได้ | ดีกว่ามาก (ใช้ประโยชน์จากโครงสร้าง hierarchy) |
| ตัวอย่าง data type | range, geometric, tsvector | inet, text prefix, point (quad-tree) |
| รองรับ exclusion constraint | ได้ | ได้ (ตั้งแต่ PG13+) |

---

## Step 419: pg_trgm — Fuzzy Text Search และเร่งความเร็ว ILIKE

### ปัญหา: `ILIKE '%คำค้น%'` ช้าเสมอด้วย B-Tree

Query แบบค้นหาคำที่ปรากฏ **ตรงไหนก็ได้** ในข้อความ (`ILIKE '%...%'`) เป็นปัญหาคลาสสิกที่ B-Tree index **ช่วยไม่ได้เลย** เพราะ B-Tree เรียงข้อมูลตามลำดับตัวอักษรจาก**ต้นสตริง** เท่านั้น การค้นหาแบบ "มีคำนี้อยู่ตรงกลางหรือท้ายสตริง" จึงบังคับให้เกิด Seq Scan เสมอ

```sql
EXPLAIN ANALYZE
SELECT product_id, product_name
FROM products
WHERE product_name ILIKE '%เสื้อ%';
```

```
                                    QUERY PLAN
--------------------------------------------------------------------------------
 Seq Scan on products  (cost=0.00..51.25 rows=90 width=27)
                        (actual time=0.028..1.412 rows=182 loops=1)
   Filter: (product_name ~~* '%เสื้อ%'::text)
   Rows Removed by Filter: 718
 Planning Time: 0.052 ms
 Execution Time: 1.451 ms
```

### ติดตั้ง Extension pg_trgm

`pg_trgm` (trigram) เป็น extension ที่แตกข้อความออกเป็นชุดตัวอักษร 3 ตัวติดกัน (trigram) แล้วสร้าง index (GIN หรือ GiST) บนชุด trigram เหล่านั้น ทำให้ query แบบ substring match และ similarity search เร็วขึ้นมาก

```sql
CREATE EXTENSION IF NOT EXISTS pg_trgm;
```

ทดลองดูว่าข้อความถูกแตกเป็น trigram อย่างไร:

```sql
SELECT show_trgm('เสื้อยืด');
```

```
                        show_trgm
-----------------------------------------------------------
 {"  เ"," เส",ยืด,"อย","สื",เสื,"ือ","ด ",ยื}
```

(trigram แต่ละชุดคือ 3 ตัวอักษรที่ต่อเนื่องกัน รวม padding ช่องว่างที่ต้น/ท้ายสตริงด้วย)

### สร้าง GIN Trigram Index

```sql
CREATE INDEX idx_products_name_trgm_gin
    ON products USING gin (product_name gin_trgm_ops);
```

ทดสอบผลลัพธ์:

```sql
EXPLAIN ANALYZE
SELECT product_id, product_name
FROM products
WHERE product_name ILIKE '%เสื้อ%';
```

```
                                                     QUERY PLAN
----------------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on products  (cost=12.05..38.44 rows=90 width=27) (actual time=0.089..0.187 rows=182 loops=1)
   Recheck Cond: (product_name ~~* '%เสื้อ%'::text)
   Heap Blocks: exact=95
   ->  Bitmap Index Scan on idx_products_name_trgm_gin  (cost=0.00..12.03 rows=90 width=0)
         (actual time=0.061..0.061 rows=182 loops=1)
         Index Cond: (product_name ~~* '%เสื้อ%'::text)
 Planning Time: 0.078 ms
 Execution Time: 0.221 ms
```

จาก ~1.45 ms เหลือ ~0.22 ms — และในตารางที่มีข้อมูลหลักล้านแถว ความต่างจาก Seq Scan (โต linear) กับ GIN trigram scan จะยิ่งมหาศาล

### GIN vs GiST Trigram — เลือกอย่างไร

```sql
CREATE INDEX idx_products_name_trgm_gist
    ON products USING gist (product_name gist_trgm_ops);
```

| หัวข้อ | GIN trigram | GiST trigram |
|---|---|---|
| ความเร็วค้นหา (`ILIKE`) | เร็วกว่า | ช้ากว่าเล็กน้อย (lossy) |
| ความเร็ว insert/update | ช้ากว่า | เร็วกว่า |
| ขนาด index | ใหญ่กว่า | เล็กกว่า |
| รองรับ KNN (`<->` หา string คล้ายที่สุด) | ไม่รองรับ | **รองรับ** |
| เหมาะกับ | read-heavy, ต้องการเร็วสุด | write-heavy หรือ ต้องการ similarity ranking |

### Similarity Search ด้วย `similarity()` และ `<->`

pg_trgm ยังให้ฟังก์ชัน `similarity(text, text)` คืนค่า 0-1 (ยิ่งใกล้ 1 ยิ่งคล้ายกันมาก) และ operator `<->` (distance = `1 - similarity`) ที่ใช้กับ GiST trigram index เพื่อทำ **KNN-style fuzzy search**:

```sql
-- หาสินค้าที่ชื่อ "คล้าย" กับคำค้น (พิมพ์ผิดเล็กน้อยก็ยังเจอ) เรียงจากคล้ายที่สุด
SELECT product_name, similarity(product_name, 'เสื้อยิด') AS sim_score
FROM products
ORDER BY product_name <-> 'เสื้อยิด'
LIMIT 5;
```

```
              product_name               | sim_score
-------------------------------------------+-----------
 เสื้อยืดคอกลม รุ่น 42 Standard             |      0.35
 เสื้อยืดคอกลม รุ่น 108 Pro                 |      0.35
 เสื้อยืดคอกลม รุ่น 7 Classic                |      0.35
 เสื้อฮู้ด รุ่น 210 Lite                     |      0.18
 เสื้อแจ็คเก็ต รุ่น 55 Premium               |      0.15
```

```sql
EXPLAIN ANALYZE
SELECT product_name
FROM products
ORDER BY product_name <-> 'เสื้อยิด'
LIMIT 5;
```

```
                                                       QUERY PLAN
------------------------------------------------------------------------------------------------------------------------
 Limit  (cost=0.28..2.15 rows=5 width=27) (actual time=0.312..0.398 rows=5 loops=1)
   ->  Index Scan using idx_products_name_trgm_gist on products
         (cost=0.28..335.78 rows=900 width=27) (actual time=0.310..0.395 rows=5 loops=1)
         Order By: (product_name <-> 'เสื้อยิด'::text)
 Planning Time: 0.089 ms
 Execution Time: 0.421 ms
```

Query แบบนี้ **ใช้ได้กับ GiST index เท่านั้น** (ไม่ใช่ GIN) เพราะเป็น KNN-style ordering ซึ่ง GIN ไม่รองรับ

### ตั้งค่า threshold ของ similarity

```sql
SET pg_trgm.similarity_threshold = 0.3;

SELECT product_name
FROM products
WHERE product_name % 'เสื้อยิด'   -- operator % คือ "similar enough" ตาม threshold
LIMIT 10;
```

Operator `%` ใช้ threshold ที่ตั้งไว้ใน `pg_trgm.similarity_threshold` (default 0.3) เพื่อกรองเฉพาะผลลัพธ์ที่คล้ายกันมากพอ และสามารถใช้ index (ทั้ง GIN และ GiST trigram) ได้เช่นกัน

---

## Step 420: แบบฝึกหัดรวม — เลือก Index Type ที่เหมาะสมสำหรับ Query Pattern

ตารางด้านล่างสรุป query pattern จริงในระบบ e-commerce ของเรา พร้อมเหตุผลว่าทำไมควรเลือก index type นั้น:

| # | Query Pattern | Index Type ที่เหมาะสม | เหตุผล |
|---|---|---|---|
| 1 | `WHERE customer_id = 123` (foreign key lookup) | **B-Tree** | equality + join, มาตรฐานทั่วไป, รองรับ ORDER BY ด้วย |
| 2 | `WHERE order_date BETWEEN '...' AND '...'` บนตาราง orders ขนาด 500 ล้านแถว append-only | **BRIN** | ข้อมูลเรียงตามลำดับ insert, ขนาด index เล็กมาก ประหยัด storage/maintenance |
| 3 | `WHERE attributes @> '{"color":"red","size":"L"}'` | **GIN** (jsonb_path_ops) | containment query บน JSONB หลาย key พร้อมกัน |
| 4 | `WHERE product_name ILIKE '%กระเป๋า%'` | **GIN trigram** (pg_trgm) | substring match ที่ B-Tree ทำไม่ได้ |
| 5 | ป้องกันโปรโมชันสินค้าเดียวกันมีช่วงเวลาทับซ้อน | **GiST** (ผ่าน EXCLUDE constraint) | range overlap + exclusion constraint ต้องพึ่ง GiST |
| 6 | `WHERE login_ip <<= '203.0.113.0/24'` | **SP-GiST** | IP/subnet containment เหมาะกับ radix-tree partitioning |
| 7 | `WHERE status = 'shipped'` (คอลัมน์สั้น cardinality ต่ำ, equality อย่างเดียว) | **B-Tree** (หรือ Hash ถ้าคีย์ยาวมากและมั่นใจว่าใช้ `=` เท่านั้น) | B-Tree ยืดหยุ่นกว่าและ rarely คุ้มที่จะสลับไป Hash |
| 8 | `ORDER BY product_name <-> 'คำค้น' LIMIT 10` (fuzzy ranking) | **GiST trigram** | ต้องการ KNN ordering ซึ่งมีแค่ GiST รองรับ |
| 9 | `WHERE tags && ARRAY['sale','new']` (array overlap) | **GIN** (array_ops) | array เป็นข้อมูลหลายค่าต่อแถว เหมาะกับ inverted index |
| 10 | `WHERE last_login >= now() - interval '7 days'` บนตาราง customers ที่มีการ UPDATE last_login บ่อยมาก | **B-Tree** | correlation จะเสื่อมเร็วเพราะ UPDATE บ่อย ทำให้ BRIN ไม่ได้ผล ต้องใช้ B-Tree แทน |

### ข้อคิดสำคัญ

สังเกตว่า **ข้อ 10** เป็นตัวอย่างที่ดีของ "คอลัมน์เดียวกัน (timestamp) แต่ pattern การใช้งานต่างกัน → เลือก index ต่างกัน" ถ้า `last_login` เป็นแบบ append-only (insert ครั้งเดียวไม่แก้ไข เหมือนที่เราจำลองไว้ในบทนี้) BRIN จะเหมาะมาก แต่ถ้าระบบจริง update `last_login` ทุกครั้งที่ผู้ใช้ login (ซึ่งเป็นพฤติกรรมทั่วไปของคอลัมน์นี้ในระบบจริง) correlation จะเสื่อมเร็วมาก ทำให้ B-Tree เป็นตัวเลือกที่ดีกว่า — **นี่คือเหตุผลที่การเลือก index type ต้องเข้าใจทั้ง "ชนิดข้อมูล" และ "พฤติกรรมการเขียนข้อมูล (write pattern)" ไม่ใช่แค่ชนิดของ query อย่างเดียว**

---

## สรุปท้ายบท

### ตารางสรุป Index Type ทั้งหมด

| Index Type | โครงสร้าง | Best Use Case ในระบบ e-commerce | คำสั่งสร้างตัวอย่าง |
|---|---|---|---|
| **B-Tree** | Balanced tree | Primary key, foreign key, equality, range, sort — ค่า default ที่ใช้ได้ 80% ของกรณี | `CREATE INDEX ON t (col);` |
| **Hash** | Hash table | Equality query ล้วนๆ บนคีย์ยาวมาก ไม่ sort/range | `CREATE INDEX ON t USING hash (col);` |
| **GIN** | Inverted index | JSONB containment, array overlap, full text search, trigram substring search | `CREATE INDEX ON t USING gin (col);` |
| **GiST** | Generalized balanced tree (lossy ได้) | Range overlap, exclusion constraint, geometric/spatial, KNN nearest-neighbor, trigram similarity ranking | `CREATE INDEX ON t USING gist (col);` |
| **SP-GiST** | Space-partitioned tree (non-balanced) | IP/CIDR containment, ข้อมูลกระจายตัวไม่สม่ำเสมอ, text prefix, quad-tree geometric | `CREATE INDEX ON t USING spgist (col);` |
| **BRIN** | Block range summary (min/max ต่อช่วง block) | ตารางขนาดใหญ่มากที่ข้อมูลเรียงตามลำดับ insert เช่น timestamp บน log/transaction table | `CREATE INDEX ON t USING brin (col);` |

### Checklist การเลือก Index Type

1. ถ้าไม่แน่ใจ เริ่มจาก **B-Tree** เสมอ แล้ววัดผลจริงด้วย `EXPLAIN ANALYZE`
2. คอลัมน์เก็บ **หลายค่าในแถวเดียว** (array, jsonb, tsvector) → พิจารณา **GIN**
3. ต้องการ **ป้องกันข้อมูลทับซ้อนเชิงช่วง** หรือ query แบบ **geometric/nearest-neighbor** → **GiST**
4. ตารางใหญ่มาก ข้อมูล **เรียงตามลำดับ insert** และไม่ค่อยแก้ไข → **BRIN**
5. ข้อมูลมีลักษณะ **hierarchy/กระจายไม่สม่ำเสมอ** เช่น IP address → **SP-GiST**
6. ต้องการ **fuzzy text search / ILIKE '%...%'** → **pg_trgm + GIN (หรือ GiST ถ้าต้องการ KNN)**
7. เสมอ **วัดขนาด index** (`pg_relation_size`) และ **เวลา query** (`EXPLAIN ANALYZE`) ก่อน-หลังสร้าง เพื่อยืนยันว่า index ที่เลือกช่วยได้จริง ไม่ใช่แค่ทฤษฎี
8. อย่าลืมว่า **index ทุกตัวมีต้นทุน** — ทั้งพื้นที่ storage และความเร็วในการ insert/update/delete ที่ช้าลง ยิ่ง GIN/GiST/SP-GiST มักมีต้นทุนการเขียนสูงกว่า B-Tree พอสมควร

### สิ่งที่ควรรู้เพิ่มเติม (จะกล่าวถึงในบทถัดๆ ไป)

- **Partial Index** และ **Expression Index** — สร้าง index เฉพาะบางแถว หรือ index บนผลลัพธ์ของ expression (Part 043)
- **Covering Index (INCLUDE)** และ **Index-Only Scan** — ลดการเข้าถึง heap เพิ่มเติม
- **Multi-column GIN/GiST**, **JSONB indexing แบบลึก** — Part 051-052
- **PostGIS spatial indexing** ด้วย GiST — สำหรับข้อมูลภูมิศาสตร์จริง
- **Index maintenance**: `REINDEX`, `pg_repack`, การติดตาม bloat ของแต่ละ index type

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1

อธิบายว่าทำไม Hash Index ถึง**ไม่สามารถ**ใช้กับ query `WHERE unit_price > 500` ได้ ทั้งที่สร้าง hash index บนคอลัมน์ `unit_price` ไว้แล้ว

<details>
<summary>เฉลย</summary>

Hash Index ทำงานโดยคำนวณ **hash code** ของค่าที่ query แล้วไปหา bucket ที่ตรงกันโดยตรง มันไม่มีแนวคิดเรื่อง "ลำดับ" (ordering) ของค่าเลย — hash code ของ 500 กับ 501 อาจอยู่คนละ bucket ที่ไม่มีความสัมพันธ์กันเลยในเชิงตำแหน่ง ดังนั้น hash index รองรับได้แค่ operator `=` เท่านั้น query ที่เป็น range (`>`, `<`, `BETWEEN`) ต้องใช้ B-Tree ซึ่งเก็บข้อมูลแบบเรียงลำดับ (sorted) จริงๆ

</details>

---

### แบบฝึกหัดที่ 2

เขียนคำสั่ง SQL สร้าง GIN index สองแบบ (`jsonb_ops` และ `jsonb_path_ops`) บนคอลัมน์ `attributes` ของตาราง `products` (สมมติยังไม่มี index ใดๆ) แล้วอธิบายว่า query แบบไหนที่ใช้ได้กับ `jsonb_path_ops` แต่ใช้กับ `jsonb_ops` ไม่ได้ (หรือกลับกัน)

<details>
<summary>เฉลย</summary>

```sql
CREATE INDEX idx_products_attr_ops ON products USING gin (attributes);
CREATE INDEX idx_products_attr_pathops ON products USING gin (attributes jsonb_path_ops);
```

ทั้งสอง opclass รองรับ `@>` เหมือนกัน แต่ **`jsonb_ops`** (default) รองรับ operator เพิ่มเติมคือ `?`, `?|`, `?&` (ตรวจสอบการมีอยู่ของ key) ในขณะที่ **`jsonb_path_ops` ไม่รองรับ operator เหล่านี้เลย** — ใช้ได้แค่ `@>` เท่านั้น ดังนั้น query เช่น `WHERE attributes ? 'is_limited'` จะใช้ได้กับ index `idx_products_attr_ops` เท่านั้น ส่วน query `WHERE attributes @> '{"color":"red"}'` ใช้ได้กับทั้งสอง index (แต่ `jsonb_path_ops` มักเร็ว/เล็กกว่า)

</details>

---

### แบบฝึกหัดที่ 3

ตารางชื่อ `flash_sale_slots` ต้องการเก็บ "ช่วงเวลาขายแฟลชเซลของสินค้าแต่ละชิ้น" โดยห้ามสินค้าชิ้นเดียวกันมีช่วงเวลาแฟลชเซลทับซ้อนกัน จงเขียน `CREATE TABLE` ที่มี constraint ป้องกันเรื่องนี้ พร้อมระบุว่า PostgreSQL ใช้ index type อะไรทำงานเบื้องหลัง

<details>
<summary>เฉลย</summary>

```sql
CREATE TABLE flash_sale_slots (
    slot_id      SERIAL PRIMARY KEY,
    product_id   INTEGER NOT NULL REFERENCES products(product_id),
    sale_period  TSTZRANGE NOT NULL,
    EXCLUDE USING gist (product_id WITH =, sale_period WITH &&)
);
```

PostgreSQL จะสร้าง **GiST index** อัตโนมัติเบื้องหลัง `EXCLUDE` constraint เพื่อตรวจสอบว่าไม่มี 2 แถวที่ `product_id` เท่ากัน **และ** `sale_period` ทับซ้อนกัน (`&&`) อยู่พร้อมกัน — ถ้าพยายาม insert ข้อมูลที่ทับซ้อน จะได้ error `conflicting key value violates exclusion constraint`

</details>

---

### แบบฝึกหัดที่ 4

ตาราง `orders` มี 300 ล้านแถว และ `order_date` มี correlation = 0.98 (เกือบเรียงตามลำดับ insert สมบูรณ์) ทีมงานสร้าง B-Tree index บน `order_date` และพบว่า index มีขนาดถึง 6 GB ทำให้กิน RAM ใน shared_buffers เยอะเกินไป จงแนะนำวิธีแก้และอธิบายเหตุผล

<details>
<summary>เฉลย</summary>

ควรเปลี่ยนไปใช้ **BRIN index** แทน B-Tree เพราะ:

```sql
DROP INDEX idx_orders_order_date_btree;
CREATE INDEX idx_orders_order_date_brin ON orders USING brin (order_date);
```

เนื่องจาก correlation สูงมาก (0.98) ข้อมูล `order_date` เรียงตามลำดับทางกายภาพเกือบสมบูรณ์ ทำให้ BRIN ทำงานได้อย่างมีประสิทธิภาพ — BRIN เก็บแค่ min/max ต่อช่วง block (128 page ต่อ entry โดย default) แทนที่จะเก็บ entry ต่อแถวเหมือน B-Tree ขนาด index จึงเล็กลงมหาศาล (มักจะเหลือแค่ไม่กี่สิบ MB จาก 6 GB) ทำให้ประหยัด RAM และ WAL เวลา insert ด้วย แลกมาด้วยการ query ที่อาจต้อง recheck ข้อมูลเพิ่มเล็กน้อย (lossy) แต่ในภาพรวมยังคุ้มค่ามากสำหรับตารางขนาดนี้

</details>

---

### แบบฝึกหัดที่ 5

จงอธิบายความแตกต่างระหว่าง GIN trigram index กับ GiST trigram index ทั้งในแง่ประสิทธิภาพและความสามารถที่รองรับ พร้อมยกตัวอย่างว่าแต่ละแบบเหมาะกับสถานการณ์ไหน

<details>
<summary>เฉลย</summary>

- **GIN trigram** (`gin_trgm_ops`): ค้นหาเร็วกว่า (เหมาะกับ read-heavy workload) แต่ insert/update ช้ากว่าและ index มีขนาดใหญ่กว่า ไม่รองรับ KNN ordering (`<->`) เหมาะกับตารางที่มีการอ่าน `ILIKE '%...%'` บ่อยมากแต่แก้ไขข้อมูลไม่บ่อย เช่น catalog สินค้าที่ query ค้นหาบ่อยแต่ไม่ค่อยเปลี่ยนชื่อสินค้า

- **GiST trigram** (`gist_trgm_ops`): insert/update เร็วกว่า index เล็กกว่า แต่ค้นหาช้ากว่าเล็กน้อยเพราะเป็น lossy index ข้อดีสำคัญคือ**รองรับ KNN-style ordering ด้วย `<->`** เช่น `ORDER BY col <-> 'search term' LIMIT 10` ซึ่ง GIN ทำไม่ได้เลย เหมาะกับ use case "fuzzy ranking / similarity ranking" หรือตารางที่มีการเขียนบ่อย

</details>

---

### แบบฝึกหัดที่ 6

เขียน query ที่ใช้ SP-GiST index `idx_login_audit_ip_spgist` เพื่อหา login ทั้งหมดที่มาจาก subnet `110.164.0.0/16` และอธิบายว่าทำไม B-Tree จึงไม่เหมาะกับ query ลักษณะนี้

<details>
<summary>เฉลย</summary>

```sql
SELECT audit_id, customer_id, login_ip, login_time
FROM login_audit
WHERE login_ip <<= inet '110.164.0.0/16';
```

B-Tree ไม่เหมาะเพราะการเปรียบเทียบ "IP นี้อยู่ใน subnet นั้นหรือไม่" (`<<=`) ไม่ใช่การเปรียบเทียบเชิงเส้นแบบธรรมดา (ไม่ใช่แค่ "น้อยกว่า/มากกว่า") แต่เป็นการเปรียบเทียบ**เชิง containment ของช่วง bit** ซึ่งต้องอาศัยโครงสร้างที่เข้าใจ hierarchy ของ IP address (เช่น radix tree/trie) — SP-GiST ถูกออกแบบมาสำหรับโครงสร้างแบบนี้โดยเฉพาะ ทำให้ partition พื้นที่ IP ได้อย่างมีประสิทธิภาพ ในขณะที่ B-Tree จะต้อง Seq Scan หรือทำงานได้ไม่ดีเพราะไม่มีแนวคิดเรื่อง prefix hierarchy

</details>

---

### แบบฝึกหัดที่ 7

ทีมงานสร้าง Hash Index บนคอลัมน์ `email` ของตาราง `customers` เพื่อหวังว่าจะช่วย enforce ว่า email ต้องไม่ซ้ำกัน (แทนที่จะใช้ `UNIQUE` constraint) จงอธิบายว่าทำไมวิธีนี้ **ใช้ไม่ได้**

<details>
<summary>เฉลย</summary>

Hash Index **ไม่สามารถใช้เป็นฐานของ UNIQUE constraint หรือ PRIMARY KEY ได้** ใน PostgreSQL — ระบบ constraint แบบ UNIQUE/PRIMARY KEY ผูกกับ B-Tree index เท่านั้น (เพราะต้องอาศัยการเรียงลำดับเพื่อตรวจสอบความซ้ำอย่างมีประสิทธิภาพและสอดคล้องกับ MVCC) หากต้องการบังคับว่า email ห้ามซ้ำ ต้องใช้:

```sql
ALTER TABLE customers ADD CONSTRAINT customers_email_unique UNIQUE (email);
```

ซึ่งจะสร้าง unique B-Tree index ให้อัตโนมัติ ไม่ใช่ hash index

</details>

---

### แบบฝึกหัดที่ 8

จงเขียนคำสั่ง SQL เพื่อตรวจสอบ **correlation** ของคอลัมน์ `signup_date` ในตาราง `customers` แล้วอธิบายว่าถ้าค่า correlation ที่ได้เท่ากับ 0.05 ควรเลือกใช้ index type ใดสำหรับ query แบบ range บนคอลัมน์นี้ในตารางขนาดใหญ่

<details>
<summary>เฉลย</summary>

```sql
SELECT attname, correlation
FROM pg_stats
WHERE tablename = 'customers' AND attname = 'signup_date';
```

ถ้า correlation ใกล้ 0 (เช่น 0.05) แปลว่าค่าของ `signup_date` **กระจัดกระจายไม่สัมพันธ์กับตำแหน่งทางกายภาพของแถวในตารางเลย** (เช่น ข้อมูลถูก migrate มาแบบสุ่มลำดับ หรือมีการ UPDATE คอลัมน์นี้บ่อยจนแถวถูกย้ายที่เก็บ) ในกรณีนี้ **BRIN จะไม่ได้ประโยชน์อะไรเลย** เพราะแต่ละ block range จะมีค่าครอบคลุมช่วงกว้างมาก (min/max แทบจะเท่ากับ min/max ของทั้งตาราง) ทำให้ query แทบไม่สามารถ skip block range ใดได้ ควรใช้ **B-Tree** แทนสำหรับ range query บนคอลัมน์ที่มี correlation ต่ำ

</details>

---

### แบบฝึกหัดที่ 9

อธิบายว่าทำไม GIN index ถึง**เขียนข้อมูล (insert/update) ช้ากว่า** B-Tree โดยทั่วไป และ PostgreSQL มีกลไกอะไรช่วยบรรเทาปัญหานี้บ้าง

<details>
<summary>เฉลย</summary>

GIN เป็น inverted index ที่เก็บ entry ต่อ "องค์ประกอบย่อย" ไม่ใช่ต่อแถว ดังนั้นเมื่อ insert/update 1 แถวที่มีค่าในคอลัมน์ index หลายค่า (เช่น jsonb ที่มีหลาย key, array ที่มีหลาย element) ระบบต้องอัปเดต posting list ของ**ทุกองค์ประกอบ**ในแถวนั้น ซึ่งอาจหมายถึงการแก้ไขหลายตำแหน่งใน index พร้อมกัน ต่างจาก B-Tree ที่แก้แค่ 1 entry ต่อ 1 แถว

PostgreSQL บรรเทาปัญหานี้ด้วย **GIN pending list** — เมื่อ insert ใหม่ ระบบจะเขียนลง "pending list" (คล้าย buffer) ก่อนแบบเร็วๆ แล้วค่อย merge เข้า main GIN structure เป็น batch ในภายหลัง (ตอน `VACUUM`, autovacuum, หรือเมื่อ pending list เต็มตาม `gin_pending_list_limit`) ทำให้ insert ทีละแถวเร็วขึ้นมาก แลกกับ query ที่ต้องเช็คทั้ง pending list และ main structure จนกว่าจะ merge เสร็จ

</details>

---

### แบบฝึกหัดที่ 10

ระบบ e-commerce ต้องการ feature ใหม่ "ค้นหาสินค้าด้วยคำที่พิมพ์ผิดได้บ้าง" (fuzzy search) โดยต้องการ:
1. ค้นหาแบบ `ILIKE '%...%'` ที่เร็ว
2. แสดงผลลัพธ์เรียงจาก "คล้ายที่สุด" ไปหา "คล้ายน้อยที่สุด"

จงออกแบบว่าจะสร้าง index อะไรบ้าง (อาจมากกว่า 1 index) พร้อมเขียนคำสั่ง SQL ประกอบ

<details>
<summary>เฉลย</summary>

ต้องใช้ pg_trgm extension และพิจารณาสร้าง index สองแบบตามความต้องการที่ต่างกัน:

```sql
CREATE EXTENSION IF NOT EXISTS pg_trgm;

-- สำหรับ ILIKE '%...%' ที่ต้องการความเร็วสูงสุด (read-heavy)
CREATE INDEX idx_products_name_trgm_gin
    ON products USING gin (product_name gin_trgm_ops);

-- สำหรับ ORDER BY ... <-> ... LIMIT n (fuzzy ranking, KNN)
CREATE INDEX idx_products_name_trgm_gist
    ON products USING gist (product_name gist_trgm_ops);
```

การใช้งาน:

```sql
-- Requirement 1: ILIKE ที่เร็ว (ใช้ GIN index)
SELECT product_id, product_name
FROM products
WHERE product_name ILIKE '%เสื้อยิด%';

-- Requirement 2: เรียงจากคล้ายที่สุด (ใช้ GiST index ผ่าน KNN operator)
SELECT product_id, product_name, similarity(product_name, 'เสื้อยิด') AS score
FROM products
ORDER BY product_name <-> 'เสื้อยิด'
LIMIT 10;
```

เหตุผลที่ต้องสร้างทั้งสอง index: GIN เร็วกว่าสำหรับ substring filter ธรรมดา แต่ **ไม่รองรับ** KNN ordering (`<->`) ในขณะที่ GiST รองรับ KNN แต่ค้นหาแบบ filter ธรรมดาช้ากว่า GIN เล็กน้อย — ถ้าระบบต้องการทั้งสอง pattern พร้อมกันบ่อยๆ การสร้างทั้งสอง index (แลกกับพื้นที่ storage และ write overhead ที่เพิ่มขึ้น) เป็นทางเลือกที่สมเหตุสมผล

</details>

---

## บทถัดไป

เมื่อเข้าใจ index type ต่างๆ ครบถ้วนแล้ว บทถัดไปจะพาไปลึกขึ้นอีกระดับ — การสร้าง index ที่ **ฉลาดกว่าเดิม** โดยไม่ต้อง index ทั้งคอลัมน์ แต่ index เฉพาะส่วนที่จำเป็นจริงๆ ผ่าน **Partial Index** และ **Expression Index** ซึ่งช่วยลดขนาด index และเพิ่มความเร็วได้อีกขั้น

**บทถัดไป:** [Part 043 — Partial Index และ Expression Index](./part-043-partial-expression-index.md)
