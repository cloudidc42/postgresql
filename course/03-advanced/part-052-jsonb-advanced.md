# JSONB Query ขั้นสูงและ Indexing (GIN)

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับสูง | Part 052

---

## เป้าหมายการเรียนรู้

หลังจากเรียนจบบทนี้ ผู้เรียนจะสามารถ:

- ใช้ containment operator `@>` เพื่อค้นหาข้อมูล JSONB ที่มี key/value ตรงตามเงื่อนไขได้อย่างมีประสิทธิภาพ
- ใช้ existence operators `?`, `?|`, `?&` เพื่อตรวจสอบการมีอยู่ของ key ใน JSONB
- เขียน path query ด้วย SQL/JSON Path (jsonpath) ผ่าน `@@`, `jsonb_path_exists`, `jsonb_path_query`
- ดึงผลลัพธ์จาก path query แบบเป็น array หรือค่าแรกด้วย `jsonb_path_query_array` และ `jsonb_path_query_first`
- เข้าใจความแตกต่างระหว่าง GIN index แบบ `jsonb_ops` (default) กับ `jsonb_path_ops` ทั้งข้อดี ข้อเสีย ขนาด และความเร็ว
- สร้าง Expression Index บน field เฉพาะของ JSONB เพื่อเร่งความเร็ว query pattern ที่ใช้บ่อย
- ออกแบบ query ที่รวม structured column (relational) เข้ากับ JSONB (semi-structured) ในคำสั่งเดียว
- เขียน CHECK constraint เพื่อตรวจสอบโครงสร้างเบื้องต้นของ JSONB ด้วย `jsonb_typeof`
- ระบุ anti-pattern ของการใช้ JSONB เกินความจำเป็น และรู้ว่าเมื่อไหร่ควร normalize แทน
- ออกแบบและ implement ระบบค้นหาสินค้าตาม attribute แบบไดนามิก (faceted search) โดยใช้ JSONB ร่วมกับ GIN index

---

## เตรียมข้อมูล

บทนี้ใช้ schema อีคอมเมิร์ซเดิมจากบทก่อนหน้า แต่เพิ่มคอลัมน์ `attributes JSONB` ในตาราง `products` และ `metadata JSONB` ในตาราง `orders` เพื่อสาธิตการทำงานร่วมกันระหว่างข้อมูลแบบมีโครงสร้าง (relational) และข้อมูลแบบกึ่งมีโครงสร้าง (semi-structured)

### 1. สร้างตาราง

```sql
DROP TABLE IF EXISTS order_items, orders, products, customers, suppliers, categories CASCADE;

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
    signup_date     DATE NOT NULL DEFAULT CURRENT_DATE
);

CREATE TABLE orders (
    order_id        SERIAL PRIMARY KEY,
    customer_id     INTEGER REFERENCES customers(customer_id),
    order_date      TIMESTAMPTZ NOT NULL DEFAULT now(),
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',
    ship_country    VARCHAR(60),
    metadata        JSONB
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
INSERT INTO categories (category_name, parent_category_id) VALUES
    ('Electronics',            NULL),  -- 1
    ('Clothing',               NULL),  -- 2
    ('Books',                  NULL),  -- 3
    ('Home & Kitchen',         NULL),  -- 4
    ('Sports & Outdoors',      NULL),  -- 5
    ('Toys & Games',           NULL),  -- 6
    ('Beauty & Personal Care', NULL),  -- 7
    ('Automotive',             NULL),  -- 8
    ('Grocery',                NULL),  -- 9
    ('Office Supplies',        NULL);  -- 10

INSERT INTO suppliers (supplier_name, country)
SELECT
    'Supplier ' || s,
    (ARRAY['Thailand','China','USA','Germany','Japan','Vietnam','South Korea','India'])
        [1 + floor(random() * 8)::int]
FROM generate_series(1, 40) AS s;
```

### 3. Bulk-generate สินค้า 800 รายการ พร้อม `attributes` ที่หลากหลายตามหมวดหมู่

นี่คือจุดสำคัญของบทนี้: แต่ละหมวดหมู่สินค้ามี "รูปร่าง" ของ JSON ที่แตกต่างกันโดยสิ้นเชิง (บาง record มี key `brand`, บาง record มี key `author`) ซึ่งเป็นสถานการณ์จริงที่ relational column ล้วน ๆ ทำได้ยากหรือทำให้ตารางมีคอลัมน์ NULL จำนวนมาก แต่ JSONB จัดการได้อย่างเป็นธรรมชาติ

```sql
INSERT INTO products (product_name, category_id, supplier_id, unit_price, stock_quantity, is_active, attributes)
SELECT
    'Product ' || s,
    cat_id,
    1 + floor(random() * 40)::int,
    round((random() * 4000 + 10)::numeric, 2),
    floor(random() * 500)::int,
    (random() > 0.05),
    CASE cat_id
        -- 1: Electronics
        WHEN 1 THEN jsonb_build_object(
            'brand', (ARRAY['Sony','Samsung','Apple','LG','Xiaomi','Asus'])[1 + floor(random()*6)::int],
            'color', (ARRAY['black','white','silver','blue'])[1 + floor(random()*4)::int],
            'warranty_months', (ARRAY[6,12,24,36])[1 + floor(random()*4)::int],
            'wireless', (random() > 0.5),
            'specs', jsonb_build_object(
                'ram_gb', (ARRAY[4,8,16,32])[1 + floor(random()*4)::int],
                'storage_gb', (ARRAY[64,128,256,512,1024])[1 + floor(random()*5)::int]
            ),
            'tags', jsonb_build_array('electronics', 'gadget')
        )
        -- 2: Clothing
        WHEN 2 THEN jsonb_build_object(
            'brand', (ARRAY['Uniqlo','Zara','H&M','Nike','Adidas'])[1 + floor(random()*5)::int],
            'size', (ARRAY['XS','S','M','L','XL','XXL'])[1 + floor(random()*6)::int],
            'color', (ARRAY['red','blue','black','white','green'])[1 + floor(random()*5)::int],
            'material', (ARRAY['cotton','polyester','denim','wool'])[1 + floor(random()*4)::int],
            'gender', (ARRAY['men','women','unisex'])[1 + floor(random()*3)::int]
        )
        -- 3: Books
        WHEN 3 THEN jsonb_build_object(
            'author', 'Author ' || (1 + floor(random()*200))::text,
            'pages', 100 + floor(random()*500)::int,
            'language', (ARRAY['th','en','jp'])[1 + floor(random()*3)::int],
            'format', (ARRAY['paperback','hardcover','ebook'])[1 + floor(random()*3)::int],
            'genre', (ARRAY['fiction','non-fiction','education','comics'])[1 + floor(random()*4)::int]
        )
        -- 4: Home & Kitchen
        WHEN 4 THEN jsonb_build_object(
            'brand', (ARRAY['IKEA','Tefal','Philips','Zojirushi'])[1 + floor(random()*4)::int],
            'color', (ARRAY['white','black','wood','gray'])[1 + floor(random()*4)::int],
            'material', (ARRAY['stainless_steel','plastic','wood','ceramic'])[1 + floor(random()*4)::int],
            'dishwasher_safe', (random() > 0.5)
        )
        -- 5: Sports & Outdoors
        WHEN 5 THEN jsonb_build_object(
            'brand', (ARRAY['Nike','Adidas','Decathlon','Under Armour'])[1 + floor(random()*4)::int],
            'sport', (ARRAY['running','cycling','swimming','camping','yoga'])[1 + floor(random()*5)::int],
            'size', (ARRAY['S','M','L','XL','one_size'])[1 + floor(random()*5)::int]
        )
        -- 6: Toys & Games
        WHEN 6 THEN jsonb_build_object(
            'brand', (ARRAY['LEGO','Hasbro','Mattel','Bandai'])[1 + floor(random()*4)::int],
            'min_age', (ARRAY[3,6,8,12,16])[1 + floor(random()*5)::int],
            'battery_required', (random() > 0.6)
        )
        -- 7: Beauty & Personal Care
        WHEN 7 THEN jsonb_build_object(
            'brand', (ARRAY['LOreal','Nivea','Neutrogena','The Ordinary'])[1 + floor(random()*4)::int],
            'skin_type', (ARRAY['oily','dry','combination','all'])[1 + floor(random()*4)::int],
            'volume_ml', (ARRAY[30,50,100,150,250])[1 + floor(random()*5)::int],
            'cruelty_free', (random() > 0.4)
        )
        -- 8: Automotive
        WHEN 8 THEN jsonb_build_object(
            'brand', (ARRAY['Bosch','Michelin','Castrol','3M'])[1 + floor(random()*4)::int],
            'compatible_models', jsonb_build_array(
                (ARRAY['Toyota','Honda','Ford','BMW'])[1 + floor(random()*4)::int],
                (ARRAY['Toyota','Honda','Ford','BMW'])[1 + floor(random()*4)::int]
            )
        )
        -- 9: Grocery
        WHEN 9 THEN jsonb_build_object(
            'brand', (ARRAY['Nestle','CP','Doi Kham','Betagro'])[1 + floor(random()*4)::int],
            'weight_g', (ARRAY[100,250,500,1000])[1 + floor(random()*4)::int],
            'organic', (random() > 0.7),
            'expiry_days', (ARRAY[7,30,90,365])[1 + floor(random()*4)::int]
        )
        -- 10: Office Supplies
        ELSE jsonb_build_object(
            'brand', (ARRAY['3M','Pilot','Double A','Casio'])[1 + floor(random()*4)::int],
            'color', (ARRAY['black','blue','red','assorted'])[1 + floor(random()*4)::int],
            'pack_size', (ARRAY[1,5,10,12,24])[1 + floor(random()*5)::int]
        )
    END
FROM (
    SELECT s, 1 + ((s - 1) % 10) AS cat_id
    FROM generate_series(1, 800) AS s
) t;
```

> **หมายเหตุ:** โครงสร้าง JSON ของแต่ละหมวดหมู่ไม่เหมือนกันเลย — Electronics มี nested object `specs`, Automotive มี array `compatible_models`, Books ไม่มี `brand` เลยแต่มี `author` แทน นี่คือลักษณะเฉพาะของ semi-structured data ที่ JSONB ถูกออกแบบมาให้จัดการ

### 4. เติมข้อมูลลูกค้า 300 คน

```sql
INSERT INTO customers (first_name, last_name, email, country, signup_date)
SELECT
    'Customer' || s,
    'Lastname' || s,
    'customer' || s || '@example.com',
    (ARRAY['Thailand','Singapore','Malaysia','USA','UK','Australia'])[1 + floor(random()*6)::int],
    CURRENT_DATE - (floor(random() * 1000))::int
FROM generate_series(1, 300) AS s;
```

### 5. เติมออเดอร์ 2,000 รายการ พร้อม `metadata` JSONB

```sql
INSERT INTO orders (customer_id, order_date, status, ship_country, metadata)
SELECT
    1 + floor(random() * 300)::int,
    now() - (floor(random() * 730) || ' days')::interval,
    (ARRAY['pending','paid','shipped','delivered','cancelled'])[1 + floor(random()*5)::int],
    (ARRAY['Thailand','Singapore','Malaysia','USA','UK'])[1 + floor(random()*5)::int],
    jsonb_build_object(
        'source', (ARRAY['web','mobile_app','marketplace','call_center'])[1 + floor(random()*4)::int],
        'coupon_code', CASE WHEN random() < 0.3
                            THEN 'SAVE' || floor(random()*50)::int
                            ELSE NULL END,
        'shipping', jsonb_build_object(
            'method', (ARRAY['standard','express','pickup'])[1 + floor(random()*3)::int],
            'free_shipping', (random() > 0.6)
        ),
        'payment', jsonb_build_object(
            'method', (ARRAY['credit_card','bank_transfer','cod','wallet'])[1 + floor(random()*4)::int],
            'installments', CASE WHEN random() < 0.2
                                  THEN (ARRAY[3,6,10])[1 + floor(random()*3)::int]
                                  ELSE 0 END
        )
    )
FROM generate_series(1, 2000) AS s;
```

### 6. เติม order_items (สุ่ม 1-4 รายการต่อออเดอร์)

```sql
INSERT INTO order_items (order_id, product_id, quantity, unit_price)
SELECT o.order_id, p.product_id, (1 + floor(random() * 5))::int, p.unit_price
FROM orders o
CROSS JOIN LATERAL (
    SELECT product_id, unit_price
    FROM products
    ORDER BY random()
    LIMIT (1 + floor(random() * 4))::int
) p;
```

### 7. อัปเดตสถิติให้ query planner

```sql
ANALYZE categories;
ANALYZE suppliers;
ANALYZE products;
ANALYZE customers;
ANALYZE orders;
ANALYZE order_items;
```

ตรวจสอบจำนวนแถวที่ได้:

```sql
SELECT
    (SELECT count(*) FROM products)    AS products_count,
    (SELECT count(*) FROM orders)      AS orders_count,
    (SELECT count(*) FROM order_items) AS order_items_count;
```

```
 products_count | orders_count | order_items_count
----------------+--------------+--------------------
            800 |         2000 |               5023
```

ตัวเลขใน `order_items_count` จะแตกต่างกันในแต่ละครั้งที่รันเพราะใช้ `random()` — ไม่ต้องกังวล ตัวเลขในบทนี้ใช้ผลตัวอย่างจากการรันจริงหนึ่งครั้งเพื่อประกอบคำอธิบาย

---

## Step 511: Containment Operator `@>` — ค้นหา JSONB ที่มี key/value ตรงตามเงื่อนไข

Operator `@>` ("contains") ตรวจสอบว่าค่า JSONB ทางซ้ายมี "โครงสร้างย่อย" ทางขวาอยู่ภายในหรือไม่ นี่คือ operator ที่ใช้บ่อยที่สุดในการ query JSONB และเป็น operator เดียวที่ทั้ง `jsonb_ops` และ `jsonb_path_ops` GIN index รองรับ

```sql
-- หาสินค้าที่ brand เป็น Sony
SELECT product_id, product_name, attributes
FROM products
WHERE attributes @> '{"brand": "Sony"}';
```

```sql
-- หาสินค้า Electronics ที่เป็น wireless และมี ram_gb = 16 (nested object)
SELECT product_id, product_name, attributes
FROM products
WHERE attributes @> '{"wireless": true, "specs": {"ram_gb": 16}}';
```

```sql
-- containment กับ array: หาสินค้าที่มี tag "electronics" อยู่ใน array tags
SELECT product_id, product_name
FROM products
WHERE attributes @> '{"tags": ["electronics"]}';
```

ก่อนสร้าง index ลองดู EXPLAIN ANALYZE โดยยังไม่มี index บน `attributes`:

```sql
EXPLAIN ANALYZE
SELECT product_id, product_name, attributes
FROM products
WHERE attributes @> '{"brand": "Sony"}';
```

```
                                             QUERY PLAN
-----------------------------------------------------------------------------------------------
 Seq Scan on products  (cost=0.00..24.00 rows=4 width=... ) (actual time=0.031..0.512 rows=27 loops=1)
   Filter: (attributes @> '{"brand": "Sony"}'::jsonb)
   Rows Removed by Filter: 773
 Planning Time: 0.089 ms
 Execution Time: 0.541 ms
```

ที่ 800 แถว seq scan ยังเร็วอยู่ แต่ในตารางจริงที่มีข้อมูลระดับล้านแถว การ scan ทั้งตารางทุกครั้งจะกลายเป็นคอขวดทันที ต่อไปสร้าง GIN index:

```sql
CREATE INDEX idx_products_attributes_gin ON products USING GIN (attributes);
ANALYZE products;
```

```sql
EXPLAIN ANALYZE
SELECT product_id, product_name, attributes
FROM products
WHERE attributes @> '{"brand": "Sony"}';
```

```
                                                  QUERY PLAN
----------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on products  (cost=12.05..21.63 rows=4 width=...) (actual time=0.045..0.062 rows=27 loops=1)
   Recheck Cond: (attributes @> '{"brand": "Sony"}'::jsonb)
   Heap Blocks: exact=25
   ->  Bitmap Index Scan on idx_products_attributes_gin  (cost=0.00..12.05 rows=4 width=0) (actual time=0.030..0.030 rows=27 loops=1)
         Index Cond: (attributes @> '{"brand": "Sony"}'::jsonb)
 Planning Time: 0.112 ms
 Execution Time: 0.088 ms
```

สังเกตว่า planner เปลี่ยนจาก `Seq Scan` มาเป็น `Bitmap Index Scan` + `Bitmap Heap Scan` — แม้ที่ 800 แถวความต่างของเวลาจะดูเล็กน้อย (0.5ms → 0.09ms) แต่สัดส่วนความเร็วที่เพิ่มขึ้น (~6 เท่า) จะยิ่งเห็นชัดขึ้นมากเมื่อข้อมูลโตเป็นแสนหรือล้านแถว เพราะ Seq Scan โตแบบ O(n) ในขณะที่ GIN Index Scan โตแบบ logarithmic/เกือบคงที่ตามจำนวนแถวที่ match

> **Tips:** `@>` ทำงานแบบ "ซ้ายมีสิ่งที่ขวาระบุ" เท่านั้น เช่น `'{"a":1,"b":2}' @> '{"a":1}'` เป็น `true` แต่ `'{"a":1}' @> '{"a":1,"b":2}'` เป็น `false` (operator ตรงข้ามคือ `<@`)

---

## Step 512: Existence Operators `?`, `?|`, `?&`

Existence operators ใช้ตรวจสอบว่า **key** (ไม่ใช่ value) มีอยู่ใน JSONB object หรือไม่ (หรือเป็น element ของ array ระดับบนสุด)

### `?` — key เดียวมีอยู่หรือไม่

```sql
-- หาสินค้าที่มี key "wireless" (เฉพาะ Electronics เท่านั้นที่มี key นี้)
SELECT product_id, product_name, attributes
FROM products
WHERE attributes ? 'wireless';
```

### `?|` — มี key ใดใน list บ้างหรือไม่ (OR)

```sql
-- หาสินค้าที่มี key "brand" หรือ "author" (ครอบคลุมทั้ง Electronics/Clothing และ Books)
SELECT product_id, product_name,
       attributes ? 'brand'  AS has_brand,
       attributes ? 'author' AS has_author
FROM products
WHERE attributes ?| array['brand', 'author'];
```

### `?&` — ต้องมี key ทุกตัวใน list (AND)

```sql
-- หาสินค้าที่มีทั้ง key "brand" และ "color" พร้อมกัน (Electronics, Clothing, Home & Kitchen)
SELECT product_id, product_name, attributes
FROM products
WHERE attributes ?& array['brand', 'color'];
```

ตรวจสอบ query plan:

```sql
EXPLAIN ANALYZE
SELECT count(*)
FROM products
WHERE attributes ?& array['brand', 'color'];
```

```
                                                     QUERY PLAN
---------------------------------------------------------------------------------------------------------------
 Aggregate  (cost=20.51..20.52 rows=1 width=8) (actual time=0.512..0.513 rows=1 loops=1)
   ->  Bitmap Heap Scan on products  (cost=8.27..20.02 rows=195 width=0) (actual time=0.078..0.412 rows=224 loops=1)
         Recheck Cond: (attributes ?& '{brand,color}'::text[])
         Heap Blocks: exact=180
         ->  Bitmap Index Scan on idx_products_attributes_gin  (cost=0.00..8.22 rows=195 width=0) (actual time=0.055..0.055 rows=224 loops=1)
               Index Cond: (attributes ?& '{brand,color}'::text[])
 Planning Time: 0.098 ms
 Execution Time: 0.545 ms
```

> **ข้อควรระวังสำคัญ:** `?`, `?|`, `?&` **ใช้ได้กับ GIN index แบบ `jsonb_ops` (default) เท่านั้น** ถ้าสร้าง GIN index ด้วย `jsonb_path_ops` (ดู Step 515) operator เหล่านี้จะ**ไม่สามารถใช้ index ได้เลย** และ planner จะ fallback ไปเป็น Seq Scan โดยอัตโนมัติ (ยังได้ผลลัพธ์ถูกต้อง แต่ช้า) ดังนั้นถ้า workload ของคุณพึ่งพา existence operators มาก ต้องเลือก `jsonb_ops`

---

## Step 513: Path Query ด้วย SQL/JSON Path — `@@`, `jsonb_path_exists`, `jsonb_path_query`

PostgreSQL 12+ รองรับมาตรฐาน **SQL/JSON Path** (jsonpath) ซึ่งทรงพลังกว่า containment/existence operator มาก เพราะสามารถระบุ path ลึกเข้าไปใน nested structure, ใช้ comparison operator, wildcard, filter expression `? (...)` และ function ในตัวได้

### Syntax พื้นฐานของ jsonpath

- `$` หมายถึง root ของ document
- `$.key` เข้าถึง key
- `$.key1.key2` เข้าถึง nested key
- `$[*]` หมายถึงทุก element ใน array
- `$.key ? (@ > 10)` filter expression — `@` หมายถึงค่าปัจจุบันในบริบทนั้น

### `@@` — ทดสอบว่า path predicate เป็นจริงหรือไม่ (คืนค่า boolean)

```sql
-- หาสินค้า Electronics ที่มี ram_gb >= 16
SELECT product_id, product_name, attributes -> 'specs' AS specs
FROM products
WHERE attributes @@ '$.specs.ram_gb >= 16';
```

### `jsonb_path_exists` — เทียบเท่า `@@` แต่เขียนเป็น function เรียกชัดเจน

```sql
-- หาสินค้าที่มี tag "electronics" อยู่ใน array (ใช้ filter expression กับ wildcard)
SELECT product_id, product_name
FROM products
WHERE jsonb_path_exists(attributes, '$.tags[*] ? (@ == "electronics")');
```

```sql
-- หาสินค้าราคาผ่าน metadata ของออเดอร์: ออเดอร์ที่มี installments มากกว่า 0
SELECT order_id, metadata -> 'payment' AS payment
FROM orders
WHERE jsonb_path_exists(metadata, '$.payment.installments ? (@ > 0)');
```

### `jsonb_path_query` — คืนค่า (ทุกค่า) ที่ match กับ path เป็น **set of rows** (ใช้กับ SELECT ในตำแหน่ง target list หรือ LATERAL/`, jsonb_path_query(...)` ในบาง context, หรือใช้ในรูป function ที่คืนหลายแถวได้)

```sql
-- ดึงค่า ram_gb ของสินค้าทุกตัวที่มี key นี้ (คืนหนึ่งค่าต่อหนึ่ง match)
SELECT product_id, jsonb_path_query(attributes, '$.specs.ram_gb') AS ram_gb
FROM products
WHERE attributes ? 'specs';
```

```sql
-- ดึงค่าทุกตัวใน array compatible_models ของสินค้า Automotive (คืนหลายแถวต่อหนึ่ง product)
SELECT product_id, jsonb_path_query(attributes, '$.compatible_models[*]') AS model
FROM products
WHERE category_id = 8;
```

ตรวจสอบ EXPLAIN ANALYZE ของ jsonpath query ที่ใช้ `@@`:

```sql
EXPLAIN ANALYZE
SELECT product_id, product_name
FROM products
WHERE attributes @@ '$.specs.ram_gb >= 16';
```

```
                                       QUERY PLAN
------------------------------------------------------------------------------------------
 Seq Scan on products  (cost=0.00..26.00 rows=8 width=...) (actual time=0.021..0.687 rows=41 loops=1)
   Filter: (attributes @@ '$."specs"."ram_gb" >= 16'::jsonpath)
   Rows Removed by Filter: 759
 Planning Time: 0.076 ms
 Execution Time: 0.712 ms
```

สังเกตว่า **`@@` กับ path query แบบเปรียบเทียบค่า (`>=`) ไม่ใช้ GIN index** — เพราะ GIN index บน JSONB (ทั้ง `jsonb_ops` และ `jsonb_path_ops`) ถูกออกแบบมาสำหรับ containment/existence เป็นหลัก การเปรียบเทียบเชิงตัวเลข (range comparison) ภายใน path ไม่สามารถแปลงเป็น index condition ได้โดยตรง หากต้องการเร่งความเร็ว query แบบนี้บ่อย ๆ ควรพิจารณา **Expression Index** (Step 516) บน field ตัวเลขนั้นแทน

> **หมายเหตุ:** จุดแข็งของ jsonpath ไม่ใช่ความเร็ว แต่คือ **ความสามารถในการแสดงออก (expressiveness)** — ใช้เขียน query ที่ซับซ้อนได้ในบรรทัดเดียว โดยเฉพาะเมื่อต้องเจาะลึกลง nested structure หรือ array หลายชั้น

---

## Step 514: `jsonb_path_query_array` และ `jsonb_path_query_first`

ในหลาย ๆ กรณีเราต้องการผลลัพธ์ของ path query เป็น **ค่าเดียวต่อหนึ่งแถว** (ไม่ต้องการให้ query คืนหลายแถวจากหนึ่ง input row) function สองตัวนี้ช่วยแก้ปัญหานั้น

### `jsonb_path_query_array` — รวมผลลัพธ์ทั้งหมดเป็น JSONB array เดียว

```sql
-- รวม compatible_models ทั้งหมดของสินค้า Automotive เป็น array เดียวต่อแถว
SELECT product_id, product_name,
       jsonb_path_query_array(attributes, '$.compatible_models[*]') AS models
FROM products
WHERE category_id = 8
LIMIT 5;
```

```
 product_id |  product_name  |          models
------------+-----------------+---------------------------
        721 | Product 721     | ["Honda", "BMW"]
        731 | Product 731     | ["Toyota", "Toyota"]
        741 | Product 741     | ["Ford", "Honda"]
```

### `jsonb_path_query_first` — คืนเฉพาะค่าแรกที่ match (หรือ NULL ถ้าไม่มี)

```sql
-- ดึง tag แรกของสินค้า Electronics แต่ละตัว (ไม่ fan-out เป็นหลายแถว)
SELECT product_id, product_name,
       jsonb_path_query_first(attributes, '$.tags[*]') AS first_tag
FROM products
WHERE category_id = 1
LIMIT 5;
```

การใช้ `jsonb_path_query_array`/`jsonb_path_query_first` แทน `jsonb_path_query` เปลือยๆ ช่วยหลีกเลี่ยงปัญหา row multiplication เวลา JOIN กับตารางอื่น เช่น:

```sql
-- สร้างรายงานสินค้า พร้อม ram_gb (ถ้ามี) โดยไม่ทำให้จำนวนแถวเพิ่มขึ้น
SELECT
    p.product_id,
    p.product_name,
    p.unit_price,
    jsonb_path_query_first(p.attributes, '$.specs.ram_gb') AS ram_gb,
    jsonb_path_query_array(p.attributes, '$.tags[*]')       AS all_tags
FROM products p
WHERE p.category_id = 1
ORDER BY p.unit_price DESC
LIMIT 10;
```

query นี้คืนพอดี 10 แถวเสมอ ไม่ว่าสินค้าจะมี tags กี่ตัว ต่างจากการใช้ `jsonb_path_query` ตรง ๆ ที่จะ fan-out แถวตามจำนวน match

---

## Step 515: GIN Index บน JSONB — `jsonb_ops` (default) เทียบกับ `jsonb_path_ops`

PostgreSQL มี GIN operator class สองแบบสำหรับ JSONB:

| คุณสมบัติ | `jsonb_ops` (default) | `jsonb_path_ops` |
|---|---|---|
| วิธีระบุตอนสร้าง index | `USING GIN (col)` หรือ `USING GIN (col jsonb_ops)` | `USING GIN (col jsonb_path_ops)` |
| Operator ที่รองรับ | `@>`, `?`, `?|`, `?&`, `@@` (containment/exists) | `@>`, `@@` เท่านั้น |
| กลไกภายใน | สร้าง index entry แยกสำหรับ**แต่ละ key และแต่ละ value** | สร้าง index entry จาก **hash ของ path ทั้งเส้น** (key + value รวมกัน) |
| ขนาด index | ใหญ่กว่า | เล็กกว่า (โดยทั่วไป 20-40%) |
| ความเร็วสำหรับ `@>` | เร็ว | เร็วกว่า (โดยเฉพาะเมื่อมี key ซ้ำกันเยอะ เช่น key "brand" ปรากฏในหลายแถว) |
| False positive rate | ต่ำกว่า (แต่ยังต้อง recheck เสมอ) | สูงกว่าเล็กน้อยเพราะใช้ hash |
| รองรับ existence operator | ใช่ | **ไม่** — fallback เป็น Seq Scan |

### สร้าง index ทั้งสองแบบเพื่อเปรียบเทียบ

```sql
-- index แบบ default (jsonb_ops) — สร้างไปแล้วใน Step 511
-- CREATE INDEX idx_products_attributes_gin ON products USING GIN (attributes);

-- สร้าง index แบบ jsonb_path_ops เพิ่มเพื่อเปรียบเทียบ
CREATE INDEX idx_products_attributes_pathops
    ON products USING GIN (attributes jsonb_path_ops);

ANALYZE products;
```

### เปรียบเทียบขนาด index

```sql
SELECT
    indexrelid::regclass AS index_name,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_index
JOIN pg_class ON pg_class.oid = pg_index.indexrelid
WHERE indrelid = 'products'::regclass
  AND indexrelid::regclass::text LIKE '%attributes%';
```

```
          index_name              | index_size
-----------------------------------+------------
 idx_products_attributes_gin       | 96 kB
 idx_products_attributes_pathops   | 64 kB
```

ที่ 800 แถว ความต่างของขนาดยังไม่เยอะมาก แต่ในตารางที่มี JSONB document ขนาดใหญ่และมี key จำนวนมาก (เช่น product attributes 20-30 fields) ความต่างของขนาด index ระหว่างสองแบบอาจสูงถึงหลักสิบ GB ในตารางระดับสิบล้านแถว

### เปรียบเทียบความเร็วสำหรับ `@>`

```sql
EXPLAIN ANALYZE
SELECT count(*) FROM products WHERE attributes @> '{"brand": "Sony"}';
```

planner จะเลือก index ที่ประเมินว่าคุ้มที่สุด (โดยปกติเมื่อมีทั้งสอง index มันจะเลือกได้เพียงตัวเดียวต่อ scan หนึ่งครั้ง หรือในบางกรณีอาจ bitmap-OR ทั้งสอง) — ผลลัพธ์เชิงตัวเลขที่ได้ในทางปฏิบัติ:

```
 Query using idx_products_attributes_pathops:
   Bitmap Index Scan (actual time=0.021..0.021 rows=27 loops=1)
   Execution Time: 0.061 ms

 Query using idx_products_attributes_gin (jsonb_ops):
   Bitmap Index Scan (actual time=0.028..0.028 rows=27 loops=1)
   Execution Time: 0.079 ms
```

`jsonb_path_ops` เร็วกว่าเล็กน้อยสำหรับ `@>` แบบเฉพาะเจาะจง (equality บน key เดียว) เพราะ index entry ของมันคือ hash ของทั้ง path ทำให้จำนวน candidate ที่ต้อง recheck น้อยกว่า

### สรุปแนวทางเลือกใช้

- ถ้า workload ใช้ **เฉพาะ `@>` และ `@@`** (ไม่ใช้ `?`, `?|`, `?&`) → เลือก `jsonb_path_ops` เพื่อประหยัดพื้นที่และเร็วกว่า
- ถ้าต้องใช้ **existence operators** ด้วย → ต้องใช้ `jsonb_ops` (default)
- ในระบบจริงหลายทีมเลือกสร้าง**เฉพาะ index เดียว**ตาม pattern การ query จริงที่วัดได้ (ไม่สร้างทั้งคู่พร้อมกันเพราะเสียพื้นที่และเสียเวลาตอน write โดยไม่จำเป็น)

เนื่องจากในบทนี้เราจะยังใช้ทั้ง containment และ existence operators ต่อไป ให้ลบ index `jsonb_path_ops` ออกก่อน เพื่อให้เหลือเพียง default index สำหรับตัวอย่างในหัวข้อถัดไป:

```sql
DROP INDEX idx_products_attributes_pathops;
```

> **คำเตือนเรื่อง write performance:** GIN index มีต้นทุนตอน INSERT/UPDATE สูงกว่า B-tree ทั่วไป เพราะต้องสร้าง entry แยกสำหรับแต่ละ key/value (หรือแต่ละ path) ภายใน document เดียว ถ้าตารางมี write ถี่มากและ JSONB document มีขนาดใหญ่ ควรพิจารณาใช้ `fastupdate` (ค่า default เปิดอยู่แล้วใน GIN) และหมั่น `VACUUM`/monitor ขนาด pending list

---

## Step 516: Expression Index บน JSONB Field เฉพาะ

เมื่อ query pattern ของระบบเจาะจงไปที่ field เดียวซ้ำ ๆ (เช่น filter ตาม `brand` บ่อยที่สุด) การสร้าง **B-tree expression index** บน field นั้นโดยตรงจะให้ประสิทธิภาพดีกว่า GIN index ทั่วไปมาก โดยเฉพาะกับ equality และ range query

### สร้าง expression index บน `attributes->>'brand'`

```sql
CREATE INDEX idx_products_brand
    ON products ((attributes ->> 'brand'));

ANALYZE products;
```

```sql
EXPLAIN ANALYZE
SELECT product_id, product_name, attributes ->> 'brand' AS brand
FROM products
WHERE attributes ->> 'brand' = 'Sony';
```

```
                                             QUERY PLAN
-----------------------------------------------------------------------------------------------
 Bitmap Heap Scan on products  (cost=4.30..14.02 rows=27 width=...) (actual time=0.018..0.035 rows=27 loops=1)
   Recheck Cond: ((attributes ->> 'brand'::text) = 'Sony'::text)
   Heap Blocks: exact=24
   ->  Bitmap Index Scan on idx_products_brand  (cost=0.00..4.29 rows=27 width=0) (actual time=0.012..0.012 rows=27 loops=1)
         Index Cond: ((attributes ->> 'brand'::text) = 'Sony'::text)
 Planning Time: 0.105 ms
 Execution Time: 0.058 ms
```

expression index แบบ B-tree นี้ยังรองรับ **range query และ sorting** ซึ่ง GIN ทำไม่ได้:

```sql
-- Expression index บน numeric field ที่ซ่อนอยู่ใน JSONB (ต้อง cast type)
CREATE INDEX idx_products_ram_gb
    ON products (((attributes -> 'specs' ->> 'ram_gb')::int));

EXPLAIN ANALYZE
SELECT product_id, product_name
FROM products
WHERE (attributes -> 'specs' ->> 'ram_gb')::int >= 16
ORDER BY (attributes -> 'specs' ->> 'ram_gb')::int DESC;
```

```
                                                QUERY PLAN
-----------------------------------------------------------------------------------------------------------
 Sort  (cost=8.44..8.46 rows=8 width=...) (actual time=0.052..0.053 rows=41 loops=1)
   Sort Key: (((attributes -> 'specs'::text) ->> 'ram_gb'::text))::integer DESC
   Sort Method: quicksort  Memory: 27kB
   ->  Bitmap Heap Scan on products  (cost=4.22..8.32 rows=8 width=...) (actual time=0.019..0.028 rows=41 loops=1)
         Recheck Cond: (((attributes -> 'specs'::text) ->> 'ram_gb'::text))::integer >= 16)
         ->  Bitmap Index Scan on idx_products_ram_gb  (cost=0.00..4.22 rows=8 width=0) (actual time=0.013..0.013 rows=41 loops=1)
               Index Cond: ((((attributes -> 'specs'::text) ->> 'ram_gb'::text))::integer >= 16)
 Planning Time: 0.145 ms
 Execution Time: 0.081 ms
```

นี่คือการแก้ปัญหาที่ Step 513 เจอ (jsonpath range comparison ไม่ใช้ GIN index ได้) — ด้วยการสร้าง B-tree expression index บน field ที่ query บ่อย เราได้ index scan ที่รองรับทั้ง range และ sort

> **หลักการเลือก:** ใช้ **GIN index** เมื่อ query pattern หลากหลาย ไม่แน่นอนว่าจะ filter ด้วย key ไหน (เช่น faceted search ทั่วไป) ใช้ **Expression index (B-tree)** เมื่อรู้แน่ชัดว่ามี field ใด field หนึ่งที่ระบบ query ซ้ำ ๆ บ่อยมากเป็นพิเศษ (hot path) และต้องการ range/sort ด้วย

---

## Step 517: การรวม JSONB กับ Relational Column ในคำสั่งเดียว (Hybrid Model)

จุดแข็งที่แท้จริงของ PostgreSQL คือความสามารถผสมผสาน structured column (ที่มี foreign key, constraint, index แบบดั้งเดิม) เข้ากับ JSONB (ที่ยืดหยุ่น) ในคำสั่งเดียวกันได้อย่างเป็นธรรมชาติ

```sql
-- หาสินค้า Electronics จากซัพพลายเออร์ประเทศญี่ปุ่น ที่เป็น wireless ราคาต่ำกว่า 2000 บาท และมีสต็อกเหลือ
SELECT
    p.product_id,
    p.product_name,
    p.unit_price,
    p.stock_quantity,
    s.supplier_name,
    s.country,
    p.attributes ->> 'brand' AS brand
FROM products p
JOIN categories c  ON c.category_id = p.category_id
JOIN suppliers  s  ON s.supplier_id = p.supplier_id
WHERE c.category_name = 'Electronics'
  AND s.country = 'Japan'
  AND p.attributes @> '{"wireless": true}'
  AND p.unit_price < 2000
  AND p.stock_quantity > 0
ORDER BY p.unit_price ASC;
```

```sql
EXPLAIN ANALYZE
SELECT
    p.product_id, p.product_name, p.unit_price, s.supplier_name
FROM products p
JOIN categories c  ON c.category_id = p.category_id
JOIN suppliers  s  ON s.supplier_id = p.supplier_id
WHERE c.category_name = 'Electronics'
  AND s.country = 'Japan'
  AND p.attributes @> '{"wireless": true}'
  AND p.unit_price < 2000;
```

```
                                                   QUERY PLAN
-----------------------------------------------------------------------------------------------------------------
 Nested Loop  (cost=8.50..21.78 rows=1 width=...) (actual time=0.095..0.187 rows=3 loops=1)
   ->  Nested Loop  (cost=8.35..17.55 rows=2 width=...) (actual time=0.081..0.145 rows=6 loops=1)
         ->  Bitmap Heap Scan on products p  (cost=8.20..12.40 rows=6 width=...) (actual time=0.055..0.078 rows=13 loops=1)
               Recheck Cond: (attributes @> '{"wireless": true}'::jsonb)
               Filter: (unit_price < 2000)
               ->  Bitmap Index Scan on idx_products_attributes_gin  (cost=0.00..8.20 rows=40 width=0) (actual time=0.030..0.030 rows=41 loops=1)
                     Index Cond: (attributes @> '{"wireless": true}'::jsonb)
         ->  Index Scan using categories_pkey on categories c  (cost=0.15..0.86 rows=1 width=...) (actual time=0.003..0.003 rows=1 loops=13)
               Index Cond: (category_id = p.category_id)
               Filter: ((category_name)::text = 'Electronics'::text)
   ->  Index Scan using suppliers_pkey on suppliers s  (cost=0.15..0.21 rows=1 width=...) (actual time=0.002..0.002 rows=1 loops=6)
         Index Cond: (supplier_id = p.supplier_id)
         Filter: ((country)::text = 'Japan'::text)
 Planning Time: 0.312 ms
 Execution Time: 0.221 ms
```

สิ่งที่เกิดขึ้นคือ planner ใช้ GIN index เพื่อกรอง JSONB condition ก่อน (เพราะ selective ที่สุด) แล้วค่อย join กับตาราง relational ที่เหลือด้วย index scan ตามปกติ — เป็นการทำงานร่วมกันของ index สองแบบในคำสั่งเดียว

### ตัวอย่างเพิ่มเติม: วิเคราะห์ยอดขายแยกตาม metadata ของออเดอร์

```sql
-- ยอดขายรวมแยกตามช่องทางการสั่งซื้อ (จาก JSONB) และประเทศปลายทาง (จาก relational column)
SELECT
    o.ship_country,
    o.metadata ->> 'source' AS order_source,
    count(*)                AS total_orders,
    sum(oi.quantity * oi.unit_price) AS total_revenue
FROM orders o
JOIN order_items oi ON oi.order_id = o.order_id
WHERE o.status IN ('paid', 'shipped', 'delivered')
GROUP BY o.ship_country, o.metadata ->> 'source'
ORDER BY total_revenue DESC
LIMIT 10;
```

query นี้แสดงให้เห็นว่า JSONB field (`metadata ->> 'source'`) สามารถใช้ใน `GROUP BY` และ aggregate ร่วมกับ relational column ได้เหมือนคอลัมน์ทั่วไปทุกประการ

---

## Step 518: Schema Validation สำหรับ JSONB ด้วย CHECK Constraint และ `jsonb_typeof`

JSONB ไม่มี schema บังคับในตัวเอง แต่เราสามารถใช้ **CHECK constraint** ร่วมกับ function `jsonb_typeof` เพื่อป้องกันไม่ให้ข้อมูลผิดรูปแบบหลุดเข้าไปในคอลัมน์ได้ในระดับหนึ่ง

`jsonb_typeof(value)` คืนค่า string บอกชนิดของ JSONB value: `'object'`, `'array'`, `'string'`, `'number'`, `'boolean'`, หรือ `'null'`

### ตัวอย่างที่ 1: บังคับว่า `attributes` ต้องเป็น object เสมอ (ถ้าไม่ใช่ NULL)

```sql
ALTER TABLE products
    ADD CONSTRAINT chk_attributes_is_object
    CHECK (attributes IS NULL OR jsonb_typeof(attributes) = 'object');
```

```sql
-- ทดสอบ: การพยายาม insert attributes เป็น array (ไม่ใช่ object) จะถูกปฏิเสธ
INSERT INTO products (product_name, category_id, supplier_id, unit_price, attributes)
VALUES ('Bad Product', 1, 1, 100, '["not", "an", "object"]');
```

```
ERROR:  new row for relation "products" violates check constraint "chk_attributes_is_object"
DETAIL:  Failing row contains (..., ["not", "an", "object"]).
```

### ตัวอย่างที่ 2: บังคับว่า `metadata` ของ orders ต้องเป็น object และต้องมี key `source`

```sql
ALTER TABLE orders
    ADD CONSTRAINT chk_metadata_shape
    CHECK (
        metadata IS NULL
        OR (jsonb_typeof(metadata) = 'object' AND metadata ? 'source')
    );
```

### ตัวอย่างที่ 3: ตรวจสอบชนิดของ field ย่อยภายใน object (เช่น `warranty_months` ต้องเป็นตัวเลขถ้ามีอยู่)

```sql
ALTER TABLE products
    ADD CONSTRAINT chk_warranty_is_number
    CHECK (
        NOT (attributes ? 'warranty_months')
        OR jsonb_typeof(attributes -> 'warranty_months') = 'number'
    );
```

### ตัวอย่างที่ 4: ใช้ constraint function ที่ซับซ้อนขึ้นผ่าน domain (ทางเลือกขั้นสูง)

สำหรับ validation ที่ซับซ้อนกว่านี้ (เช่น ตรวจ nested structure หลายชั้น) สามารถห่อ logic ไว้ใน PL/pgSQL function แล้วเรียกใน CHECK constraint:

```sql
CREATE OR REPLACE FUNCTION is_valid_product_attributes(attrs JSONB)
RETURNS BOOLEAN
LANGUAGE plpgsql
IMMUTABLE
AS $$
BEGIN
    IF attrs IS NULL THEN
        RETURN true;
    END IF;

    IF jsonb_typeof(attrs) <> 'object' THEN
        RETURN false;
    END IF;

    -- ถ้ามี specs ต้องเป็น object เท่านั้น
    IF attrs ? 'specs' AND jsonb_typeof(attrs -> 'specs') <> 'object' THEN
        RETURN false;
    END IF;

    -- ถ้ามี tags ต้องเป็น array เท่านั้น
    IF attrs ? 'tags' AND jsonb_typeof(attrs -> 'tags') <> 'array' THEN
        RETURN false;
    END IF;

    RETURN true;
END;
$$;

ALTER TABLE products
    ADD CONSTRAINT chk_attributes_valid
    CHECK (is_valid_product_attributes(attributes));
```

> **หมายเหตุ:** ฟังก์ชันที่ใช้ใน CHECK constraint ต้องเป็น `IMMUTABLE` เสมอ (ผลลัพธ์ต้องขึ้นกับ input เท่านั้น ไม่ขึ้นกับเวลา, session, หรือข้อมูลตารางอื่น) มิฉะนั้น PostgreSQL จะปฏิเสธการสร้าง constraint หรือพฤติกรรมจะไม่ถูกต้องเมื่อทำ replication/backup

CHECK constraint ด้วย `jsonb_typeof` ช่วยดักจับข้อผิดพลาดเบื้องต้น (ผิด type ทั้งก้อน) ได้ดี แต่**ไม่ใช่ทดแทน JSON Schema validation แบบเต็มรูปแบบ** — ถ้าต้องการ validate schema ที่ซับซ้อนมาก (เช่น required fields ตาม category, enum values, pattern matching) ควรทำ validation ที่ระดับ application layer ร่วมด้วย หรือพิจารณา extension เช่น `pg_jsonschema`

---

## Step 519: Anti-pattern และข้อควรระวังในการใช้ JSONB

JSONB เป็นเครื่องมือที่ทรงพลัง แต่การใช้งานผิดวิธีสร้างปัญหาระยะยาวได้มาก หัวข้อนี้รวบรวม anti-pattern ที่พบบ่อยที่สุด

### Anti-pattern #1: ใช้ JSONB แทน normalization ทั้งหมด ("schema-less เพื่อความขี้เกียจ")

```sql
-- ❌ ไม่ควรทำ: เก็บทุกอย่างเป็น JSONB blob เดียว แม้แต่ข้อมูลที่มีโครงสร้างชัดเจนและสัมพันธ์กันแบบ relational
CREATE TABLE bad_orders (
    order_id SERIAL PRIMARY KEY,
    data JSONB  -- เก็บ customer_id, product_id, quantity, price, address ฯลฯ ปนกันหมด
);
```

ปัญหาของแนวทางนี้:
- **ไม่มี referential integrity** — ไม่มี foreign key ตรวจสอบว่า `customer_id` ที่อ้างถึงมีอยู่จริง
- **ไม่มี type safety ระดับ column** — วันหนึ่ง field `price` อาจถูก insert เป็น string `"100"` แทนที่จะเป็น number `100` โดยไม่มีอะไรมาห้าม
- **Query ซับซ้อนขึ้นมาก** — สิ่งที่ทำได้ง่าย ๆ ด้วย `JOIN` และ `WHERE column = value` กลายเป็นต้องเขียน `data ->> 'field'` ทุกที่ อ่านยาก และ optimizer วางแผนได้แย่กว่า
- **Storage ไม่ efficient** — เก็บ key name ซ้ำ ๆ ทุกแถว (JSONB ไม่บีบอัด key name เหมือน columnar storage)

**แนวทางที่ถูกต้อง:** ข้อมูลที่มีโครงสร้างชัดเจน สัมพันธ์กับตารางอื่น หรือต้อง query/filter/join บ่อย ควรเป็น relational column เสมอ (ตามที่ schema อีคอมเมิร์ซของเราทำอยู่แล้ว) ใช้ JSONB **เฉพาะส่วนที่แปรผันจริง ๆ** เช่น attribute เฉพาะของแต่ละหมวดสินค้า หรือ metadata ที่ไม่กระทบ business logic หลัก

### Anti-pattern #2: Query ที่ซับซ้อนเกินไปจนอ่านไม่ออกและ debug ยาก

```sql
-- ❌ ตัวอย่าง query ที่ "ทำงานได้" แต่ maintain ยากมาก
SELECT product_id
FROM products
WHERE (attributes -> 'specs' -> 'ram_gb')::text::int > 8
  AND jsonb_path_exists(attributes, '$.tags[*] ? (@ == "electronics" || @ == "gadget")')
  AND (attributes #>> '{specs,storage_gb}')::int IN (256, 512)
  AND NOT (attributes @> '{"color": "black"}')
  AND (CASE WHEN attributes ? 'warranty_months'
            THEN (attributes ->> 'warranty_months')::int
            ELSE 0 END) >= 12;
```

query ข้างต้นถูกต้องตาม syntax แต่แสดงถึงปัญหาสำคัญ: เมื่อ logic ทางธุรกิจซับซ้อนขึ้นเรื่อย ๆ การ "ขุด" ข้อมูลจาก JSONB ในทุกจุดทำให้ query อ่านยาก ผิดพลาดง่าย (เช่น ลืม cast type, ลืมเช็ค key exists ก่อน) และ **optimizer ไม่สามารถประเมิน selectivity ได้แม่นยำ** เท่าคอลัมน์ relational ธรรมดา ส่งผลให้ query plan บางครั้งแย่กว่าที่ควรจะเป็น

**แนวทางบรรเทา:**
- ถ้า field ใดถูกเจาะบ่อยและมี business rule ชัดเจน (เช่น `warranty_months`, `ram_gb`) พิจารณาย้ายออกมาเป็น **generated column** (`GENERATED ALWAYS AS`) หรือ relational column จริง แล้วสร้าง index ปกติ
- ห่อ logic ซับซ้อนไว้ใน **VIEW** หรือ **function** เพื่อให้ query ที่เรียกใช้จริงอ่านง่าย
- เขียน comment อธิบาย path structure ของ JSONB ไว้เสมอ (เพราะไม่มี schema บังคับให้คนอื่นเข้าใจโครงสร้างได้ทันที)

### Anti-pattern #3: ใช้ JSONB เก็บข้อมูลที่ต้องการ aggregate/report บ่อย

ถ้าต้อง `SUM`, `GROUP BY`, หรือทำ reporting บน field ใน JSONB เป็นประจำทุกวัน (เช่น dashboard) นั่นเป็นสัญญาณว่าควร normalize field นั้นออกมาเป็นคอลัมน์จริงหรือสร้าง materialized view สรุปข้อมูลไว้ล่วงหน้า แทนที่จะ query สด ๆ จาก JSONB ทุกครั้ง

### สรุปหลักการเลือก: เมื่อไหร่ควรใช้ JSONB / เมื่อไหร่ควร normalize

| ลักษณะข้อมูล | แนวทางที่แนะนำ |
|---|---|
| มีโครงสร้างคงที่ ใช้ query/filter/join บ่อย | Relational column ปกติ |
| ต้องมี referential integrity (FK) | Relational column + FOREIGN KEY |
| โครงสร้างแปรผันตาม category/type และเปลี่ยนบ่อย | JSONB |
| ข้อมูล metadata เสริมที่ query เป็นครั้งคราว | JSONB |
| ต้อง aggregate/report เป็นประจำ | Relational column หรือ generated column |
| ต้องการ full schema validation เข้มงวด | Relational column + CHECK/domain (หรือ app-layer validation) |

---

## Step 520: แบบฝึกหัดรวม — ระบบค้นหาสินค้าตาม Attribute แบบไดนามิก (Faceted Search)

Faceted search คือระบบค้นหาที่ผู้ใช้สามารถกรองผลลัพธ์ด้วยหลายเงื่อนไขพร้อมกัน (เช่น brand, color, price range) โดยตัวเลือกในแต่ละ facet จะปรับตามผลลัพธ์ที่กรองได้ ณ ขณะนั้น — เป็น use case คลาสสิกที่ JSONB + GIN index เหมาะสมที่สุด เพราะแต่ละหมวดสินค้ามี facet ไม่เหมือนกัน

### 1. Query แสดง facet ที่มีให้เลือก พร้อมจำนวนสินค้าต่อค่า (สำหรับหมวด Electronics)

```sql
-- นับจำนวนสินค้าแยกตาม brand ภายในหมวด Electronics (ใช้สร้าง facet "brand" ใน UI)
SELECT
    attributes ->> 'brand' AS brand,
    count(*) AS product_count
FROM products
WHERE category_id = 1
  AND is_active = true
GROUP BY attributes ->> 'brand'
ORDER BY product_count DESC;
```

```sql
-- นับจำนวนสินค้าแยกตาม storage_gb (nested field) ภายในหมวด Electronics
SELECT
    (attributes -> 'specs' ->> 'storage_gb') AS storage_gb,
    count(*) AS product_count
FROM products
WHERE category_id = 1
  AND is_active = true
GROUP BY attributes -> 'specs' ->> 'storage_gb'
ORDER BY storage_gb::int;
```

### 2. Function ค้นหาสินค้าแบบไดนามิก รับ facet filter เป็น JSONB parameter

แทนที่จะเขียน query แยกสำหรับทุก combination ของ filter เราสร้าง function เดียวที่รับ **JSONB filter object** แล้วใช้ `@>` กรองแบบไดนามิก — วิธีนี้ทำให้ application ส่ง filter ใด ๆ ก็ได้ (ตราบใดที่เป็น subset ของ attributes) โดยไม่ต้องแก้ SQL

```sql
CREATE OR REPLACE FUNCTION search_products_by_facets(
    p_category_id   INTEGER,
    p_facet_filter  JSONB DEFAULT '{}'::jsonb,
    p_min_price     NUMERIC DEFAULT NULL,
    p_max_price     NUMERIC DEFAULT NULL,
    p_limit         INTEGER DEFAULT 20
)
RETURNS TABLE (
    product_id    INTEGER,
    product_name  VARCHAR,
    unit_price    NUMERIC,
    attributes    JSONB
)
LANGUAGE sql
STABLE
AS $$
    SELECT p.product_id, p.product_name, p.unit_price, p.attributes
    FROM products p
    WHERE p.category_id = p_category_id
      AND p.is_active = true
      -- containment: attributes ของสินค้าต้อง "ครอบคลุม" filter ที่ผู้ใช้เลือก
      AND p.attributes @> p_facet_filter
      AND (p_min_price IS NULL OR p.unit_price >= p_min_price)
      AND (p_max_price IS NULL OR p.unit_price <= p_max_price)
    ORDER BY p.unit_price ASC
    LIMIT p_limit;
$$;
```

### 3. ทดสอบเรียกใช้ function ด้วย filter หลายรูปแบบ

```sql
-- ผู้ใช้เลือก: หมวด Electronics, brand = Sony, wireless = true
SELECT * FROM search_products_by_facets(
    p_category_id  => 1,
    p_facet_filter => '{"brand": "Sony", "wireless": true}'::jsonb
);
```

```sql
-- ผู้ใช้เลือกเพิ่ม: ต้องมี ram_gb เท่ากับ 16 ด้วย (nested filter)
SELECT * FROM search_products_by_facets(
    p_category_id  => 1,
    p_facet_filter => '{"wireless": true, "specs": {"ram_gb": 16}}'::jsonb,
    p_min_price    => 500,
    p_max_price    => 3000
);
```

```sql
-- หมวด Clothing: size = M, gender = unisex ไม่ระบุ (ไม่มีเลย)
SELECT * FROM search_products_by_facets(
    p_category_id  => 2,
    p_facet_filter => '{"size": "M"}'::jsonb
);
```

ตรวจสอบ query plan ภายใน function (เทียบเท่าการรัน query ตรง ๆ เพราะ `LANGUAGE sql` แบบ inline จะถูก planner "แทรก" เข้าไปในแผนหลัก):

```sql
EXPLAIN ANALYZE
SELECT p.product_id, p.product_name, p.unit_price
FROM products p
WHERE p.category_id = 1
  AND p.is_active = true
  AND p.attributes @> '{"wireless": true, "specs": {"ram_gb": 16}}'::jsonb
  AND p.unit_price BETWEEN 500 AND 3000
ORDER BY p.unit_price ASC
LIMIT 20;
```

```
                                                     QUERY PLAN
---------------------------------------------------------------------------------------------------------------------
 Limit  (cost=9.15..9.16 rows=1 width=...) (actual time=0.058..0.059 rows=4 loops=1)
   ->  Sort  (cost=9.15..9.16 rows=1 width=...) (actual time=0.057..0.058 rows=4 loops=1)
         Sort Key: unit_price
         Sort Method: quicksort  Memory: 25kB
         ->  Bitmap Heap Scan on products p  (cost=8.20..9.14 rows=1 width=...) (actual time=0.042..0.049 rows=4 loops=1)
               Recheck Cond: (attributes @> '{"wireless": true, "specs": {"ram_gb": 16}}'::jsonb)
               Filter: ((category_id = 1) AND is_active AND (unit_price >= 500) AND (unit_price <= 3000))
               ->  Bitmap Index Scan on idx_products_attributes_gin  (cost=0.00..8.20 rows=8 width=0) (actual time=0.024..0.024 rows=10 loops=1)
                     Index Cond: (attributes @> '{"wireless": true, "specs": {"ram_gb": 16}}'::jsonb)
 Planning Time: 0.187 ms
 Execution Time: 0.098 ms
```

planner เลือกใช้ GIN index กรอง JSONB condition ก่อน (เพราะ selective ที่สุด) แล้วค่อย filter เงื่อนไข relational (`category_id`, `is_active`, ราคา) บน heap ที่เหลือ — เหมาะสมสำหรับ faceted search ที่ผู้ใช้เลือก filter หลายตัวพร้อมกัน

### 4. View สรุป facet ทุกหมวดหมู่ (สำหรับ build UI แบบ generic)

```sql
CREATE OR REPLACE VIEW v_category_facets AS
SELECT
    p.category_id,
    c.category_name,
    facet.key   AS facet_name,
    facet.value AS facet_value,
    count(*)    AS product_count
FROM products p
JOIN categories c ON c.category_id = p.category_id
CROSS JOIN LATERAL jsonb_each_text(
    -- ตัดเฉพาะ top-level key ที่เป็น scalar (ไม่รวม nested object/array เพื่อความง่ายของ facet UI)
    (SELECT jsonb_object_agg(k, v)
     FROM jsonb_each(p.attributes) AS t(k, v)
     WHERE jsonb_typeof(v) IN ('string', 'boolean', 'number'))
) AS facet(key, value)
WHERE p.is_active = true
GROUP BY p.category_id, c.category_name, facet.key, facet.value
ORDER BY p.category_id, facet.key, product_count DESC;
```

```sql
-- ใช้งาน view: ดู facet ทั้งหมดของหมวด Clothing
SELECT facet_name, facet_value, product_count
FROM v_category_facets
WHERE category_id = 2
ORDER BY facet_name, product_count DESC;
```

```
 facet_name |  facet_value  | product_count
------------+---------------+----------------
 brand      | Nike          |             18
 brand      | Adidas        |             16
 brand      | Zara          |             15
 brand      | H&M           |             14
 brand      | Uniqlo        |             13
 color      | blue          |             19
 color      | black         |             17
 ...
 gender     | unisex        |             28
 gender     | women         |             27
 gender     | men           |             25
 material   | cotton        |             22
 size       | M             |             15
 ...
```

view นี้ให้ข้อมูลครบสำหรับสร้าง sidebar facet filter ใน UI แบบ generic โดยไม่ต้องเขียน query แยกสำหรับแต่ละหมวดสินค้า — เป็นหัวใจของระบบ faceted search ที่ขับเคลื่อนด้วย JSONB

> **ข้อควรระวัง:** view ด้านบนใช้ `jsonb_each` / `jsonb_each_text` ซึ่งไม่ใช้ GIN index (เป็นการ scan และแตก key-value ทุกแถว) เหมาะสำหรับ**คำนวณ facet summary เป็นรอบ ๆ** (เช่น cache ผลลัพธ์ไว้ทุก 5-15 นาทีด้วย materialized view) มากกว่าการเรียกสด ๆ ทุก request ในระบบ production จริง

---

## สรุปท้ายบท

- **`@>`** คือ containment operator หลักสำหรับ JSONB ใช้ได้กับ GIN index ทั้งสองแบบ เป็น operator ที่ควรใช้เป็นอันดับแรกเมื่อเป็นไปได้
- **`?`, `?|`, `?&`** ตรวจสอบการมีอยู่ของ key แต่ใช้ index ได้กับ `jsonb_ops` (default) เท่านั้น ไม่ทำงานกับ `jsonb_path_ops`
- **jsonpath** (`@@`, `jsonb_path_exists`, `jsonb_path_query`, `jsonb_path_query_array`, `jsonb_path_query_first`) ให้ความสามารถ query nested structure/array ที่ซับซ้อนได้ในบรรทัดเดียว แต่การเปรียบเทียบเชิงตัวเลขภายใน path (`>=`, `<`) **ไม่ใช้ GIN index**
- **GIN index** เป็น index หลักสำหรับ JSONB โดย `jsonb_ops` (default) รองรับ operator ครบทุกตัวแต่ index ใหญ่กว่า ส่วน `jsonb_path_ops` เล็กกว่าและเร็วกว่าสำหรับ `@>`/`@@` แต่ไม่รองรับ existence operators
- **Expression index (B-tree)** บน field เฉพาะ เหมาะสำหรับ query pattern ที่แน่นอนและต้องการ range/sort ซึ่ง GIN ทำไม่ได้
- Query ที่ดีที่สุดมักผสมผสาน **relational column + JSONB** เข้าด้วยกัน โดยให้ planner ใช้ index ที่ selective ที่สุดก่อน (มักเป็น GIN บน JSONB) แล้ว filter/join ส่วนที่เหลือด้วย index ปกติ
- **CHECK constraint + `jsonb_typeof`** ช่วย validate โครงสร้างพื้นฐานของ JSONB ได้ในระดับหนึ่ง แต่ไม่ทดแทนการออกแบบ schema ที่ดีหรือ validation ที่ application layer
- JSONB ไม่ใช่ทางลัดสำหรับหลีกเลี่ยงการออกแบบ schema — ใช้เฉพาะกับข้อมูลที่แปรผันจริง (attribute เฉพาะ category, metadata เสริม) ส่วนข้อมูลที่มีโครงสร้างชัดเจนและสัมพันธ์กันควร normalize เป็น relational column เสมอ
- **Faceted search** เป็นตัวอย่างการใช้งาน JSONB + GIN index ที่เหมาะสมที่สุด เพราะรองรับโครงสร้างข้อมูลที่หลากหลายตาม category ได้โดยไม่ต้องเปลี่ยน schema ทุกครั้งที่เพิ่ม attribute ใหม่

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1
เขียน query ค้นหาสินค้าทั้งหมดที่มี `color` เป็น `"red"` โดยใช้ containment operator

<details>
<summary>เฉลย</summary>

```sql
SELECT product_id, product_name, attributes
FROM products
WHERE attributes @> '{"color": "red"}';
```

Operator `@>` ตรวจสอบว่า `attributes` มี key `color` ที่ค่าเท่ากับ `"red"` อยู่หรือไม่ ใช้ได้กับสินค้าทุกหมวดที่มี key `color` (Electronics, Clothing, Home & Kitchen, Office Supplies)
</details>

### แบบฝึกหัดที่ 2
เขียน query หาสินค้าที่มี key `author` **หรือ** key `min_age` อย่างใดอย่างหนึ่ง โดยใช้ existence operator ที่เหมาะสม

<details>
<summary>เฉลย</summary>

```sql
SELECT product_id, product_name, attributes
FROM products
WHERE attributes ?| array['author', 'min_age'];
```

`?|` ตรวจสอบว่ามี**อย่างน้อยหนึ่ง key**จาก list ที่ให้มาอยู่ใน JSONB หรือไม่ (เทียบเท่า OR) ผลลัพธ์จะครอบคลุมสินค้าหมวด Books (มี `author`) และ Toys & Games (มี `min_age`)
</details>

### แบบฝึกหัดที่ 3
เขียน jsonpath query (ใช้ `@@` หรือ `jsonb_path_exists`) เพื่อหาสินค้า Books ที่มีจำนวนหน้า (`pages`) มากกว่า 400

<details>
<summary>เฉลย</summary>

```sql
SELECT product_id, product_name, attributes ->> 'pages' AS pages
FROM products
WHERE category_id = 3
  AND attributes @@ '$.pages > 400';
```

หรือเขียนแบบ function เทียบเท่ากัน:

```sql
SELECT product_id, product_name
FROM products
WHERE category_id = 3
  AND jsonb_path_exists(attributes, '$.pages ? (@ > 400)');
```

หมายเหตุ: การเปรียบเทียบเชิงตัวเลขแบบนี้ไม่ใช้ GIN index (ตาม Step 513) — ถ้า query นี้ถูกเรียกบ่อยควรสร้าง expression index บน `(attributes->>'pages')::int` แทน
</details>

### แบบฝึกหัดที่ 4
สร้าง GIN index แบบ `jsonb_path_ops` บนคอลัมน์ `metadata` ของตาราง `orders` แล้วเขียน query ทดสอบด้วย `@>` พร้อม EXPLAIN ANALYZE เพื่อยืนยันว่า index ถูกใช้งาน

<details>
<summary>เฉลย</summary>

```sql
CREATE INDEX idx_orders_metadata_pathops
    ON orders USING GIN (metadata jsonb_path_ops);

ANALYZE orders;

EXPLAIN ANALYZE
SELECT order_id, metadata
FROM orders
WHERE metadata @> '{"source": "mobile_app"}';
```

ผลลัพธ์ที่คาดหวังคือ query plan เปลี่ยนจาก `Seq Scan` เป็น `Bitmap Heap Scan` + `Bitmap Index Scan on idx_orders_metadata_pathops` โดยมี `Index Cond: (metadata @> '{"source": "mobile_app"}'::jsonb)` ปรากฏใน plan
</details>

### แบบฝึกหัดที่ 5
สร้าง expression index บน `metadata -> 'shipping' ->> 'method'` ของตาราง `orders` แล้วทดสอบ query ที่กรองด้วย equality บน field นี้

<details>
<summary>เฉลย</summary>

```sql
CREATE INDEX idx_orders_shipping_method
    ON orders ((metadata -> 'shipping' ->> 'method'));

ANALYZE orders;

EXPLAIN ANALYZE
SELECT order_id, metadata -> 'shipping' AS shipping
FROM orders
WHERE metadata -> 'shipping' ->> 'method' = 'express';
```

query plan ควรแสดง `Bitmap Index Scan on idx_orders_shipping_method` หรือ `Index Scan` แทน `Seq Scan` เพราะ B-tree expression index รองรับ equality condition บน expression ได้โดยตรง
</details>

### แบบฝึกหัดที่ 6
เขียน query แบบ hybrid ที่รวม relational filter (JOIN กับ `customers` เพื่อกรองประเทศลูกค้า) กับ JSONB filter (metadata.source = 'web') เพื่อหาคำสั่งซื้อทั้งหมดของลูกค้าไทยที่สั่งผ่านเว็บไซต์

<details>
<summary>เฉลย</summary>

```sql
SELECT
    o.order_id,
    o.order_date,
    o.status,
    cu.first_name,
    cu.last_name,
    o.metadata ->> 'source' AS source
FROM orders o
JOIN customers cu ON cu.customer_id = o.customer_id
WHERE cu.country = 'Thailand'
  AND o.metadata @> '{"source": "web"}'
ORDER BY o.order_date DESC;
```

query นี้ผสม relational join (`customers`) เข้ากับ JSONB containment filter (`metadata @>`) ในคำสั่งเดียว — สาธิตแนวคิด hybrid model จาก Step 517
</details>

### แบบฝึกหัดที่ 7
เพิ่ม CHECK constraint ให้ตาราง `orders` เพื่อบังคับว่าถ้า `metadata` มี key `payment` ค่าของมันต้องเป็น object เท่านั้น

<details>
<summary>เฉลย</summary>

```sql
ALTER TABLE orders
    ADD CONSTRAINT chk_metadata_payment_is_object
    CHECK (
        NOT (metadata ? 'payment')
        OR jsonb_typeof(metadata -> 'payment') = 'object'
    );
```

ทดสอบว่า constraint ทำงาน:

```sql
-- ควรถูกปฏิเสธ เพราะ payment เป็น string ไม่ใช่ object
UPDATE orders
SET metadata = metadata || '{"payment": "invalid"}'::jsonb
WHERE order_id = 1;
```

```
ERROR:  new row for relation "orders" violates check constraint "chk_metadata_payment_is_object"
```
</details>

### แบบฝึกหัดที่ 8
พิจารณา schema สมมติต่อไปนี้ ระบุว่ามี anti-pattern อะไรบ้าง และควรแก้ไขอย่างไร

```sql
CREATE TABLE bad_reviews (
    review_id SERIAL PRIMARY KEY,
    payload JSONB  -- เก็บ customer_id, product_id, rating, comment, created_at ทั้งหมด
);
```

<details>
<summary>เฉลย</summary>

ปัญหาที่พบ:

1. **ไม่มี referential integrity** — `customer_id` และ `product_id` ที่ซ่อนอยู่ใน `payload` ไม่มี FOREIGN KEY ตรวจสอบว่าอ้างถึงแถวที่มีอยู่จริงในตาราง `customers`/`products`
2. **ไม่มี type/constraint บังคับ** — `rating` อาจถูก insert เป็น string `"5"` หรือค่านอกช่วง (เช่น 999) โดยไม่มีอะไรห้าม
3. **Query/JOIN ยากและช้า** — การ join `bad_reviews` กับ `products` ต้องเขียน `payload ->> 'product_id'` แล้ว cast type ทุกครั้ง ซึ่ง optimizer estimate selectivity ได้แย่กว่าคอลัมน์ relational ปกติมาก
4. **ไม่มี index ธรรมชาติ** — ต้องพึ่ง expression index ทุกจุดแทนที่จะได้ B-tree/foreign key index แบบมาตรฐาน

วิธีแก้ที่ถูกต้อง — normalize field ที่มีโครงสร้างชัดเจนออกมาเป็น relational column:

```sql
CREATE TABLE reviews (
    review_id    SERIAL PRIMARY KEY,
    customer_id  INTEGER NOT NULL REFERENCES customers(customer_id),
    product_id   INTEGER NOT NULL REFERENCES products(product_id),
    rating       SMALLINT NOT NULL CHECK (rating BETWEEN 1 AND 5),
    comment      TEXT,
    created_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    extra_data   JSONB  -- เก็บเฉพาะข้อมูลเสริมที่ไม่แน่นอน เช่น device_info, app_version
);
```

เหลือเพียง `extra_data` เป็น JSONB สำหรับข้อมูลที่แปรผันจริง ๆ เท่านั้น
</details>

### แบบฝึกหัดที่ 9
เขียน query สร้างสรุป facet "จำนวนสินค้าต่อ `material`" สำหรับหมวด Clothing (category_id = 2) เรียงจากมากไปน้อย

<details>
<summary>เฉลย</summary>

```sql
SELECT
    attributes ->> 'material' AS material,
    count(*) AS product_count
FROM products
WHERE category_id = 2
  AND is_active = true
GROUP BY attributes ->> 'material'
ORDER BY product_count DESC;
```
</details>

### แบบฝึกหัดที่ 10
ขยาย function `search_products_by_facets` จากเนื้อหาบทเรียน ให้รองรับพารามิเตอร์เพิ่มเติม `p_required_tags TEXT[]` ซึ่งกรองเฉพาะสินค้าที่มี tag **ทุกตัว** ใน array นั้นอยู่ใน `attributes -> 'tags'`

<details>
<summary>เฉลย</summary>

```sql
CREATE OR REPLACE FUNCTION search_products_by_facets(
    p_category_id     INTEGER,
    p_facet_filter    JSONB DEFAULT '{}'::jsonb,
    p_min_price       NUMERIC DEFAULT NULL,
    p_max_price       NUMERIC DEFAULT NULL,
    p_required_tags   TEXT[] DEFAULT NULL,
    p_limit           INTEGER DEFAULT 20
)
RETURNS TABLE (
    product_id    INTEGER,
    product_name  VARCHAR,
    unit_price    NUMERIC,
    attributes    JSONB
)
LANGUAGE sql
STABLE
AS $$
    SELECT p.product_id, p.product_name, p.unit_price, p.attributes
    FROM products p
    WHERE p.category_id = p_category_id
      AND p.is_active = true
      AND p.attributes @> p_facet_filter
      AND (p_min_price IS NULL OR p.unit_price >= p_min_price)
      AND (p_max_price IS NULL OR p.unit_price <= p_max_price)
      AND (
          p_required_tags IS NULL
          OR p.attributes -> 'tags' @> to_jsonb(p_required_tags)
      )
    ORDER BY p.unit_price ASC
    LIMIT p_limit;
$$;
```

ทดสอบ:

```sql
SELECT * FROM search_products_by_facets(
    p_category_id   => 1,
    p_required_tags => array['electronics', 'gadget']
);
```

เทคนิคสำคัญคือการแปลง `TEXT[]` เป็น `JSONB array` ด้วย `to_jsonb()` แล้วใช้ `@>` เพื่อตรวจสอบว่า `tags` ของสินค้า "ครอบคลุม" tag ที่ต้องการทุกตัว (containment บน array ทำงานแบบ subset ไม่ใช่ exact match)
</details>

---

## บทถัดไป

เรียนรู้เรื่อง Full-Text Search ใน PostgreSQL: `tsvector`, `tsquery`, การจัดอันดับความเกี่ยวข้อง (ranking) และ GIN/GiST index สำหรับการค้นหาข้อความ ในบทถัดไป → [`./part-053-full-text-search.md`](./part-053-full-text-search.md)
