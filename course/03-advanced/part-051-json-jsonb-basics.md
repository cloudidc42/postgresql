# บทที่ 51: JSON และ JSONB พื้นฐาน

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับสูง | Part 051

PostgreSQL เป็นฐานข้อมูลเชิงสัมพันธ์ (relational database) แต่ก็รองรับข้อมูลแบบกึ่งโครงสร้าง (semi-structured data) ได้อย่างสมบูรณ์แบบผ่านชนิดข้อมูล `JSON` และ `JSONB` ในบทนี้เราจะเริ่มต้นจากศูนย์ ทำความเข้าใจว่า JSON คืออะไร ทำไมต้องมีสองชนิด (`JSON` vs `JSONB`) วิธีสร้าง อ่าน แก้ไขค่า JSON ในตาราง ไปจนถึงการแปลงผลลัพธ์ query ทั้งก้อนให้กลายเป็น JSON เพื่อส่งออกไปให้ frontend หรือ API ใช้งานต่อ

บทนี้เป็นบทพื้นฐานของซีรีส์ JSON/JSONB ทั้งหมด ส่วนเทคนิคขั้นสูงกว่า เช่น GIN index, containment operators (`@>`, `<@`), `jsonpath`, และการดีไซน์สคีมาแบบผสม (relational + JSONB) จะอยู่ใน [Part 052: JSONB ขั้นสูง](./part-052-jsonb-advanced.md)

---

## เป้าหมายการเรียนรู้

เมื่อจบบทนี้ ผู้เรียนจะสามารถ:

1. อธิบายได้ว่า JSON คืออะไร และทำไม PostgreSQL ถึงเลือกรองรับ semi-structured data ในฐานข้อมูลเชิงสัมพันธ์
2. แยกความแตกต่างระหว่างชนิดข้อมูล `JSON` และ `JSONB` ทั้งในแง่การจัดเก็บ (storage), ความเร็วในการ query, และความสามารถในการทำ index
3. สร้างและ insert ค่า JSON/JSONB ลงในตาราง พร้อมเข้าใจกลไก validation อัตโนมัติของ PostgreSQL
4. ใช้ operator `->` และ `->>` เพื่อดึงค่าจาก JSON/JSONB รวมถึงการเข้าถึงค่าที่ซ้อนกันหลายชั้น (nested path)
5. ใช้ operator `#>` และ `#>>` เพื่อเข้าถึง path ที่ลึกหลายชั้นด้วย array of keys
6. ใช้ฟังก์ชัน `jsonb_set` เพื่อแก้ไขค่าบางส่วนใน JSONB โดยไม่ต้องเขียนทั้งก้อนใหม่
7. แปลงผลลัพธ์ query เป็น JSON ด้วย `row_to_json`, `to_jsonb`, `json_build_object`, และ `json_agg`
8. แตก (expand) JSONB object/array ให้กลายเป็นแถวข้อมูลด้วย `jsonb_each`, `jsonb_object_keys`, `jsonb_array_elements`
9. รวมสอง JSONB เข้าด้วยกันด้วย `||`, ลบ key ด้วย `-`, และล้างค่า `null` ด้วย `jsonb_strip_nulls`
10. ประยุกต์ใช้ทุกเทคนิคข้างต้นกับข้อมูลจริง — จัดการ `products.attributes` และ `orders.metadata` แบบยืดหยุ่นในระบบอีคอมเมิร์ซ

---

## เตรียมข้อมูล

เราจะใช้สคีมาฐานข้อมูลอีคอมเมิร์ซชุดเดิมที่ใช้ตลอดทั้งหลักสูตร โดยเพิ่มคอลัมน์ `attributes JSONB` ในตาราง `products` (สำหรับเก็บ spec สินค้าที่มีรูปแบบไม่ตายตัวในแต่ละหมวดหมู่) และคอลัมน์ `metadata JSONB` ในตาราง `orders` (สำหรับเก็บข้อมูลเสริมของคำสั่งซื้อ เช่น คูปอง, ช่องทางที่มา, การห่อของขวัญ)

### โครงสร้างตาราง (schema)

```sql
DROP TABLE IF EXISTS order_items, orders, customers, products, suppliers, categories CASCADE;

CREATE TABLE categories (
    category_id         SERIAL PRIMARY KEY,
    category_name       VARCHAR(100) NOT NULL,
    parent_category_id  INTEGER REFERENCES categories(category_id)
);

CREATE TABLE suppliers (
    supplier_id    SERIAL PRIMARY KEY,
    supplier_name  VARCHAR(150) NOT NULL,
    country        VARCHAR(60)
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
    customer_id   SERIAL PRIMARY KEY,
    first_name    VARCHAR(60) NOT NULL,
    last_name     VARCHAR(60) NOT NULL,
    email         VARCHAR(150) UNIQUE,
    country       VARCHAR(60),
    signup_date   DATE NOT NULL DEFAULT CURRENT_DATE
);

CREATE TABLE orders (
    order_id      SERIAL PRIMARY KEY,
    customer_id   INTEGER REFERENCES customers(customer_id),
    order_date    TIMESTAMPTZ NOT NULL DEFAULT now(),
    status        VARCHAR(20) NOT NULL DEFAULT 'pending',
    ship_country  VARCHAR(60),
    metadata      JSONB
);

CREATE TABLE order_items (
    order_item_id  SERIAL PRIMARY KEY,
    order_id       INTEGER REFERENCES orders(order_id),
    product_id     INTEGER REFERENCES products(product_id),
    quantity       INTEGER NOT NULL CHECK (quantity > 0),
    unit_price     NUMERIC(10,2) NOT NULL
);
```

> **หมายเหตุ**: โจทย์กำหนดให้สคีมามาพร้อมคอลัมน์ `attributes` และ `metadata` อยู่แล้ว (ในสถานการณ์จริงที่ตารางมีอยู่ก่อน เราจะเพิ่มคอลัมน์ด้วย `ALTER TABLE products ADD COLUMN attributes JSONB;` และ `ALTER TABLE orders ADD COLUMN metadata JSONB;` ซึ่งเป็น operation ที่เร็วมากเพราะ PostgreSQL ไม่ต้อง rewrite ตารางเมื่อค่า default เป็น `NULL`)

### ข้อมูลตัวอย่าง (seed data)

**categories** (10 แถว มีลำดับชั้น parent → child):

```sql
INSERT INTO categories (category_name, parent_category_id) VALUES
('Electronics', NULL),               -- 1
('Computers', 1),                    -- 2
('Mobile Phones', 1),                -- 3
('Fashion', NULL),                   -- 4
('Men''s Clothing', 4),              -- 5
('Women''s Clothing', 4),            -- 6
('Books', NULL),                     -- 7
('Home & Kitchen', NULL),            -- 8
('Sports & Outdoor', NULL),          -- 9
('Beauty & Personal Care', NULL);    -- 10
```

**suppliers** (10 แถว):

```sql
INSERT INTO suppliers (supplier_name, country) VALUES
('TechSource Co., Ltd.', 'Thailand'),        -- 1
('Global Gadgets Inc.', 'USA'),              -- 2
('Fashion Forward Ltd.', 'Thailand'),        -- 3
('BookWorm Publishing', 'UK'),               -- 4
('HomeStyle Supplies', 'Thailand'),          -- 5
('SportsPro Distribution', 'Germany'),       -- 6
('Beauty Bliss Co.', 'South Korea'),         -- 7
('Digital World Trading', 'China'),          -- 8
('Premium Textiles', 'Vietnam'),             -- 9
('Kitchen Master Co.', 'Thailand');          -- 10
```

**products** (20 แถว — จุดสำคัญคือ `attributes` มีโครงสร้างต่างกันไปตามหมวดหมู่สินค้า นี่คือเหตุผลหลักที่เราใช้ JSONB แทนที่จะสร้างคอลัมน์ตายตัวแบบ `color`, `size`, `weight_kg` ในตาราง เพราะสินค้าแต่ละประเภทมี spec ไม่เหมือนกันเลย):

```sql
INSERT INTO products (product_name, category_id, supplier_id, unit_price, stock_quantity, is_active, attributes) VALUES
('iPhone 15 Pro Max', 3, 8, 42900.00, 35, true,
 '{"color":"titanium blue","storage_gb":256,"weight_kg":0.221,"tags":["bestseller","5g","new"],
   "specs":{"battery":"4441mAh","screen_inch":6.7,"5g":true}}'),
('Samsung Galaxy S24 Ultra', 3, 8, 39900.00, 28, true,
 '{"color":"black","storage_gb":512,"weight_kg":0.232,"tags":["bestseller","5g"],
   "specs":{"battery":"5000mAh","screen_inch":6.8,"5g":true}}'),
('MacBook Pro 14" M3', 2, 1, 79900.00, 12, true,
 '{"color":"space gray","ram_gb":18,"storage_gb":512,"weight_kg":1.55,"tags":["premium"],
   "specs":{"battery":"70Wh","cpu":"Apple M3 Pro"}}'),
('Dell XPS 13', 2, 1, 45900.00, 18, true,
 '{"color":"platinum silver","ram_gb":16,"storage_gb":1024,"weight_kg":1.24,
   "specs":{"battery":"55Wh","cpu":"Intel Core i7"}}'),
('Sony WH-1000XM5 Headphones', 1, 2, 12900.00, 40, true,
 '{"color":"black","weight_kg":0.25,
   "specs":{"battery_life_hr":30,"noise_cancelling":true,"bluetooth":"5.2"}}'),
('Men''s Cotton T-Shirt', 5, 9, 350.00, 200, true,
 '{"color":"navy","size":"L","material":"100% cotton","weight_kg":0.2}'),
('Men''s Slim Fit Jeans', 5, 3, 890.00, 150, true,
 '{"color":"dark blue","size":"32","material":"denim","weight_kg":0.6}'),
('Women''s Summer Dress', 6, 3, 1290.00, 90, true,
 '{"color":"red","size":"M","material":"polyester","weight_kg":0.3}'),
('Women''s Yoga Pants', 6, 9, 690.00, 120, true,
 '{"color":"black","size":"S","material":"spandex blend","weight_kg":0.25}'),
('Clean Code', 7, 4, 890.00, 60, true,
 '{"pages":464,"language":"English","isbn":"978-0132350884","format":"paperback"}'),
('Atomic Habits', 7, 4, 450.00, 100, true,
 '{"pages":320,"language":"English","isbn":"978-0735211292","format":"paperback"}'),
('หนังสือ Python เบื้องต้น', 7, 4, 350.00, 75, true,
 '{"pages":280,"language":"Thai","isbn":"978-6162536123","format":"paperback"}'),
('Stand Mixer 5L', 8, 10, 6900.00, 20, true,
 '{"color":"red","capacity_liters":5,"weight_kg":6.5,
   "specs":{"power_watt":500,"speeds":10}}'),
('Air Fryer 4.5L', 8, 10, 2590.00, 45, true,
 '{"color":"black","capacity_liters":4.5,"weight_kg":4.2,
   "specs":{"power_watt":1400,"digital_display":true}}'),
('Yoga Mat Premium', 9, 6, 590.00, 80, true,
 '{"color":"purple","size":"183x61cm","material":"TPE","weight_kg":1.2}'),
('Running Shoes Pro', 9, 6, 3200.00, 55, true,
 '{"color":"white/blue","size":"42","weight_kg":0.65,"tags":["bestseller"],
   "specs":{"cushion_type":"air","waterproof":false}}'),
('Facial Serum Vitamin C', 10, 7, 690.00, 100, true,
 '{"volume_ml":30,"skin_type":"all",
   "specs":{"key_ingredient":"vitamin C","concentration_percent":15}}'),
('Moisturizing Cream', 10, 7, 590.00, 90, true,
 '{"volume_ml":50,"skin_type":"dry",
   "specs":{"key_ingredient":"hyaluronic acid"}}'),
('iPad Air', 1, 8, 22900.00, 30, true,
 '{"color":"blue","storage_gb":128,"weight_kg":0.462,
   "specs":{"screen_inch":10.9,"5g":false}}'),
('Wireless Mouse', 1, 2, 590.00, 150, true,
 '{"color":"black","weight_kg":0.09,
   "specs":{"dpi":1600,"wireless":true,"battery":"AA x1"}}');
```

**customers** (15 แถว):

```sql
INSERT INTO customers (first_name, last_name, email, country, signup_date) VALUES
('Somchai', 'Jaidee', 'somchai.j@example.com', 'Thailand', '2023-01-15'),
('Suda', 'Meesuk', 'suda.m@example.com', 'Thailand', '2023-02-20'),
('John', 'Smith', 'john.smith@example.com', 'USA', '2023-03-05'),
('Ananya', 'Wong', 'ananya.w@example.com', 'Thailand', '2023-04-10'),
('Wei', 'Zhang', 'wei.zhang@example.com', 'China', '2023-05-01'),
('Emma', 'Johnson', 'emma.j@example.com', 'UK', '2023-05-15'),
('Kittipong', 'Suksawat', 'kitti.s@example.com', 'Thailand', '2023-06-01'),
('Yuki', 'Tanaka', 'yuki.t@example.com', 'Japan', '2023-06-20'),
('Pranee', 'Chaiyaporn', 'pranee.c@example.com', 'Thailand', '2023-07-10'),
('Michael', 'Brown', 'michael.b@example.com', 'USA', '2023-08-01'),
('Nattaya', 'Rungrueang', 'nattaya.r@example.com', 'Thailand', '2023-08-15'),
('Lars', 'Andersen', 'lars.a@example.com', 'Germany', '2023-09-01'),
('Siriporn', 'Boonmee', 'siriporn.b@example.com', 'Thailand', '2023-09-20'),
('David', 'Lee', 'david.lee@example.com', 'South Korea', '2023-10-05'),
('Waraporn', 'Intharak', 'waraporn.i@example.com', 'Thailand', '2023-10-25');
```

**orders** (18 แถว — สังเกตว่า `metadata` แต่ละใบมี key ไม่เหมือนกัน บางใบไม่มี `coupon_code`, บางใบมี `utm` ซ้อนอยู่ข้างใน, บางใบมีค่า JSON `null`):

```sql
INSERT INTO orders (customer_id, order_date, status, ship_country, metadata) VALUES
(1,  '2024-01-05 10:00+07', 'paid',      'Thailand', '{"coupon_code":"SALE10","referrer":"facebook","gift_wrap":false}'),
(2,  '2024-01-08 11:30+07', 'completed', 'Thailand', '{"coupon_code":null,"referrer":"google","gift_wrap":true,"note":"deliver to office"}'),
(3,  '2024-01-10 09:15+07', 'completed', 'USA',      '{"referrer":"instagram","gift_wrap":false}'),
(4,  '2024-01-12 14:45+07', 'shipped',   'Thailand', '{"coupon_code":"NEWYEAR20","referrer":"friend_referral","gift_wrap":true}'),
(5,  '2024-01-15 16:20+07', 'paid',      'China',    '{"referrer":"tiktok","gift_wrap":false}'),
(6,  '2024-01-20 08:05+07', 'completed', 'Thailand', '{"coupon_code":"SALE10","referrer":"email_campaign","gift_wrap":false,"utm":{"source":"newsletter","campaign":"jan_sale"}}'),
(6,  '2024-01-22 13:10+07', 'cancelled', 'UK',       '{"referrer":"google","gift_wrap":false,"cancel_reason":"changed mind"}'),
(7,  '2024-01-25 17:40+07', 'paid',      'Thailand', '{"coupon_code":"VIP15","referrer":"line","gift_wrap":true}'),
(8,  '2024-02-01 10:55+07', 'completed', 'Japan',    '{"referrer":"organic","gift_wrap":false}'),
(9,  '2024-02-03 12:00+07', 'shipped',   'Thailand', '{"coupon_code":"SALE10","referrer":"facebook","gift_wrap":false}'),
(10, '2024-02-05 15:25+07', 'completed', 'USA',      '{"referrer":"google","gift_wrap":true,"note":"birthday gift"}'),
(2,  '2024-02-10 09:50+07', 'paid',      'Thailand', '{"referrer":"instagram","gift_wrap":false}'),
(11, '2024-02-12 11:15+07', 'completed', 'Thailand', '{"coupon_code":"SALE10","referrer":"tiktok","gift_wrap":false}'),
(12, '2024-02-15 14:05+07', 'pending',   'Germany',  '{"referrer":"google","gift_wrap":false}'),
(13, '2024-02-18 18:30+07', 'completed', 'Thailand', '{"coupon_code":"MEMBER5","referrer":"line","gift_wrap":true}'),
(14, '2024-02-20 10:10+07', 'completed', 'South Korea', '{"referrer":"facebook","gift_wrap":false}'),
(15, '2024-02-22 09:00+07', 'paid',      'Thailand', '{"coupon_code":"SALE10","referrer":"email_campaign","gift_wrap":false,"utm":{"source":"newsletter","campaign":"feb_promo"}}'),
(3,  '2024-02-25 16:45+07', 'shipped',   'USA',      '{"referrer":null,"gift_wrap":false}');
```

**order_items** (25 แถว):

```sql
INSERT INTO order_items (order_id, product_id, quantity, unit_price) VALUES
(1, 1, 1, 42900.00), (1, 20, 1, 590.00),
(2, 10, 2, 890.00), (2, 11, 1, 450.00),
(3, 5, 1, 12900.00),
(4, 6, 3, 350.00), (4, 7, 2, 890.00),
(5, 2, 1, 39900.00),
(6, 1, 1, 42900.00), (6, 19, 1, 22900.00),
(7, 16, 1, 3200.00),
(8, 13, 1, 6900.00),
(9, 17, 2, 690.00), (9, 18, 1, 590.00),
(10, 8, 2, 1290.00), (10, 9, 1, 690.00),
(11, 3, 1, 79900.00),
(12, 14, 1, 2590.00),
(13, 15, 2, 590.00),
(14, 4, 1, 45900.00),
(15, 6, 5, 350.00),
(16, 12, 3, 350.00),
(17, 20, 2, 590.00), (17, 5, 1, 12900.00),
(18, 2, 1, 39900.00);
```

ตรวจสอบว่าข้อมูลครบถ้วนก่อนเริ่มบทเรียน:

```sql
SELECT
    (SELECT COUNT(*) FROM categories)   AS categories,
    (SELECT COUNT(*) FROM suppliers)    AS suppliers,
    (SELECT COUNT(*) FROM products)     AS products,
    (SELECT COUNT(*) FROM customers)    AS customers,
    (SELECT COUNT(*) FROM orders)       AS orders,
    (SELECT COUNT(*) FROM order_items)  AS order_items;
```

```
 categories | suppliers | products | customers | orders | order_items
------------+-----------+----------+-----------+--------+-------------
         10 |        10 |       20 |        15 |     18 |          25
(1 row)
```

---

## Step 501: JSON คืออะไร

**JSON (JavaScript Object Notation)** คือรูปแบบการแทนข้อมูลแบบข้อความ (text-based data format) ที่มนุษย์อ่านง่ายและโปรแกรมแปลผลได้ง่าย ประกอบด้วยโครงสร้างพื้นฐาน 6 แบบ:

| ชนิดข้อมูล | ตัวอย่าง |
|---|---|
| object | `{"key": "value"}` |
| array | `[1, 2, 3]` |
| string | `"hello"` |
| number | `42`, `3.14` |
| boolean | `true`, `false` |
| null | `null` |

JSON กลายเป็นมาตรฐานโดยพฤตินัย (de facto standard) ในการแลกเปลี่ยนข้อมูลระหว่างระบบ โดยเฉพาะ REST API เพราะมันเบา (lightweight), อ่านง่าย, และรองรับโครงสร้างข้อมูลที่ซ้อนกันได้ (nested structure)

### ทำไม PostgreSQL ถึงรองรับ JSON ในฐานข้อมูลเชิงสัมพันธ์?

ฐานข้อมูลเชิงสัมพันธ์ (relational database) ถูกออกแบบมาให้ข้อมูลมี **schema ตายตัว** — ทุกแถวในตารางเดียวกันต้องมีคอลัมน์เหมือนกันหมด แต่ในโลกจริงมีข้อมูลหลายประเภทที่ **โครงสร้างไม่แน่นอน** หรือ **เปลี่ยนแปลงบ่อย** เช่น:

- **Product attributes**: สินค้าอิเล็กทรอนิกส์มี `screen_inch`, `cpu`, `ram_gb` แต่สินค้าเสื้อผ้ามี `size`, `color`, `material` — ถ้าจะสร้างคอลัมน์แยกสำหรับทุก attribute ที่เป็นไปได้ ตารางจะมีคอลัมน์เป็นร้อยและส่วนใหญ่จะเป็น `NULL`
- **API request/response logs**: payload จาก third-party API ที่มีโครงสร้างเปลี่ยนไปตาม version
- **Configuration/settings**: การตั้งค่าของผู้ใช้ที่มีตัวเลือกจำนวนมากและเพิ่มขึ้นเรื่อยๆ
- **Event/audit data**: metadata ของ event ที่แตกต่างกันไปในแต่ละประเภท event

การมี column แบบตายตัวสำหรับทุกกรณีข้างต้นจะทำให้:

1. ต้อง `ALTER TABLE` บ่อยมากทุกครั้งที่มี attribute ใหม่ (เสี่ยง downtime ในตารางใหญ่)
2. ตารางมีคอลัมน์จำนวนมากที่ส่วนใหญ่เป็น `NULL` (sparse table) สิ้นเปลืองพื้นที่และงงเวลา query
3. ยากต่อการรองรับ attribute ที่ nested (ซ้อนกันหลายชั้น) เช่น specs ของสินค้าที่มีทั้ง `specs.battery`, `specs.cpu.cores`

PostgreSQL แก้ปัญหานี้ด้วยการให้ **JSON/JSONB เป็นชนิดข้อมูลหนึ่งในคอลัมน์** — เราจึงได้ข้อดีของทั้งสองโลก:

- คอลัมน์ที่มีโครงสร้างแน่นอน (เช่น `product_id`, `unit_price`, `category_id`) ยังคงเป็น relational column ปกติ ใช้ foreign key, constraint, index แบบ B-tree ได้เต็มที่
- คอลัมน์ที่มีโครงสร้างไม่แน่นอน (เช่น `attributes`, `metadata`) เก็บเป็น JSONB ยืดหยุ่นเต็มที่ แต่ยังคง query ได้ด้วย SQL และยัง index ได้ด้วย (จะเรียนใน Part 052)

แนวทางนี้เรียกว่า **hybrid model** หรือ **polyglot persistence ในตัวเดียว** — เราไม่จำเป็นต้องแยกไปใช้ NoSQL database (เช่น MongoDB) เพื่อรองรับข้อมูลแบบยืดหยุ่น เพราะ PostgreSQL ทำได้ในฐานข้อมูลเดียวกัน พร้อม transaction, foreign key, และ ACID guarantee ครบถ้วน

ลองดูตัวอย่างจริงจากข้อมูลที่เพิ่งเตรียมไว้:

```sql
SELECT product_name, unit_price, attributes
FROM products
WHERE product_id IN (1, 6, 10);
```

```
     product_name     | unit_price |                                        attributes
-----------------------+------------+--------------------------------------------------------------------------------------------
 iPhone 15 Pro Max     |   42900.00 | {"color": "titanium blue", "storage_gb": 256, "weight_kg": 0.221, "tags": [...], "specs": {...}}
 Men's Cotton T-Shirt  |     350.00 | {"color": "navy", "size": "L", "material": "100% cotton", "weight_kg": 0.2}
 Clean Code            |     890.00 | {"pages": 464, "language": "English", "isbn": "978-0132350884", "format": "paperback"}
(3 rows)
```

สังเกตว่าทั้งสามแถวอยู่ในตารางเดียวกัน แต่ `attributes` มี key ไม่เหมือนกันเลย — โทรศัพท์มี `specs.battery`, เสื้อผ้ามี `size`/`material`, หนังสือมี `pages`/`isbn` นี่คือพลังของ JSONB ที่ column แบบตายตัวทำไม่ได้อย่างเป็นธรรมชาติ

---

## Step 502: JSON type vs JSONB type — ความแตกต่างสำคัญ

PostgreSQL มีชนิดข้อมูลสำหรับ JSON สองแบบคือ `json` และ `jsonb` (JSON Binary) ทั้งสองเก็บข้อมูลแบบเดียวกันตามมาตรฐาน JSON แต่มีวิธีจัดเก็บภายใน (internal storage) ที่ต่างกันโดยสิ้นเชิง

### 1. Storage: text vs binary

- **`json`** เก็บข้อมูลเป็น **exact copy ของข้อความ input** แบบ text ล้วนๆ (คล้าย `text`/`varchar`) — ไม่มีการแปลงโครงสร้างใดๆ
- **`jsonb`** เก็บข้อมูลในรูปแบบ **decomposed binary format** — คือแปลง JSON เป็นโครงสร้างไบนารีที่จัดเรียงและ index ภายในไว้ล่วงหน้า ทำให้ query เร็วกว่ามากเพราะไม่ต้อง parse ข้อความใหม่ทุกครั้ง

ผลที่ตามมา:

| ประเด็น | `json` | `jsonb` |
|---|---|---|
| ความเร็วตอน **insert** | เร็วกว่า (แค่ validate syntax แล้วเก็บ text ตรงๆ) | ช้ากว่าเล็กน้อย (ต้อง parse และแปลงเป็น binary) |
| ความเร็วตอน **query/process** | ช้ากว่า (ต้อง parse ทุกครั้งที่อ่าน) | เร็วกว่ามาก (ไม่ต้อง re-parse) |
| **ช่องว่าง (whitespace)** ในข้อความต้นฉบับ | เก็บไว้ครบ | ไม่เก็บ (normalize แล้ว) |
| **ลำดับ key** ใน object | เก็บตามลำดับที่ผู้ใช้ป้อน | ไม่รับประกันลำดับเดิม (จัดเรียงใหม่ภายใน) |
| **key ซ้ำกัน** ใน object เดียว | เก็บทุกตัว (คงค่าตัวสุดท้ายไว้ตอน process) | เก็บเฉพาะตัวสุดท้าย (ตัวซ้ำจะถูกตัดทิ้งตั้งแต่ insert) |
| **Indexing** (GIN index, containment operators) | ทำไม่ได้โดยตรง | ทำได้เต็มรูปแบบ |
| ฟังก์ชัน/operator ที่รองรับ | มีจำกัดกว่า | มีครบถ้วนกว่า (รวมถึง `-`, `#-`, `||`) |

### 2. ตัวอย่างที่แสดงความแตกต่างชัดเจน

**(ก) ลำดับ key และ whitespace**

```sql
SELECT
    '{"b": 2,   "a": 1}'::json  AS as_json,
    '{"b": 2,   "a": 1}'::jsonb AS as_jsonb;
```

```
       as_json        | as_jsonb
-----------------------+----------
 {"b": 2,   "a": 1}    | {"a": 1, "b": 2}
(1 row)
```

`json` เก็บ text ต้นฉบับไว้ทุกตัวอักษร (รวม whitespace และลำดับ key `b` มาก่อน `a`) ส่วน `jsonb` แปลงเป็น binary แล้วจึงพิมพ์กลับมาในลำดับ key ที่ PostgreSQL จัดเรียงเอง (โดยทั่วไปคือเรียงตามความยาว key แล้วตามตัวอักษร) และตัด whitespace ที่ไม่จำเป็นออก

**(ข) key ซ้ำกัน**

```sql
SELECT
    '{"x": 1, "x": 2}'::json  AS as_json,
    '{"x": 1, "x": 2}'::jsonb AS as_jsonb;
```

```
      as_json       | as_jsonb
---------------------+-----------
 {"x": 1, "x": 2}    | {"x": 2}
(1 row)
```

`jsonb` เก็บเฉพาะ key `x` ตัวสุดท้าย (`2`) เพราะตอนแปลงเป็น binary มันสร้าง object ที่แต่ละ key ไม่ซ้ำ ส่วน `json` เป็นแค่ text จึงยังเก็บ `"x": 1, "x": 2` ไว้ทั้งคู่ (แต่ถ้าใช้ฟังก์ชันดึงค่าอย่าง `->` จะได้ค่า key ตัวสุดท้ายเช่นกัน)

**(ค) ความเร็วในการ query ซ้ำๆ**

เพราะ `json` เป็น text ธรรมดา ทุกครั้งที่เราใช้ operator เช่น `->>'key'` PostgreSQL ต้อง **parse ข้อความทั้งก้อนใหม่ทุกครั้ง** ส่วน `jsonb` ถูก parse ไว้แล้วตั้งแต่ insert จึงดึงค่าได้เร็วกว่ามากเมื่อ query ซ้ำๆ บนข้อมูลจำนวนมาก — ในตารางขนาดใหญ่ที่ query field ใน JSON บ่อยๆ ความต่างนี้มีผลต่อ performance อย่างมีนัยสำคัญ

### 3. แล้วเมื่อไหร่ควรใช้ `json` แทน `jsonb`?

โดยทั่วไป **แนะนำให้ใช้ `jsonb` เป็นค่าเริ่มต้นเสมอ** เว้นแต่มีเหตุผลเฉพาะเจาะจงดังนี้:

- ต้องการเก็บ **ข้อความต้นฉบับแบบ byte ต่อ byte** (เช่น เก็บ log ของ payload ที่ได้รับมา เพื่อ audit หรือ debug และไม่เคย query field ข้างในเลย)
- ต้องการรักษาลำดับ key เดิมไว้เพื่อส่งออกกลับไปแบบเดิมเป๊ะๆ
- Insert บ่อยมากแต่แทบไม่เคย query field ข้างใน (`json` insert เร็วกว่าเล็กน้อยเพราะไม่ต้องแปลง binary)

ในบทนี้และตลอดหลักสูตร เราจะใช้ **`JSONB` เป็นหลัก** ตามที่กำหนดไว้ในสคีมา (`products.attributes`, `orders.metadata`) เพราะสอดคล้องกับ use case จริงในระบบอีคอมเมิร์ซที่ต้อง query, filter, และแก้ไข field ข้างในบ่อยมาก

### 4. ตรวจสอบขนาดพื้นที่จัดเก็บ

```sql
SELECT
    pg_column_size('{"a":1,"b":2,"c":[1,2,3]}'::json)  AS json_size_bytes,
    pg_column_size('{"a":1,"b":2,"c":[1,2,3]}'::jsonb) AS jsonb_size_bytes;
```

```
 json_size_bytes | jsonb_size_bytes
------------------+-------------------
               26 |                40
(1 row)
```

จะเห็นว่า `jsonb` มักใช้พื้นที่ **มากกว่า** `json` เล็กน้อยสำหรับข้อมูลชิ้นเล็กๆ เนื่องจากต้องเก็บโครงสร้าง (offset, type tag) เพิ่มเติมเพื่อให้ query ได้เร็ว แต่ข้อแลกเปลี่ยนนี้คุ้มค่ามากในระยะยาวเมื่อข้อมูลมีขนาดใหญ่และถูก query บ่อย

---

## Step 503: การสร้างและ INSERT ค่า JSON/JSONB

### 1. Literal syntax

ค่าคงที่ (literal) ของ JSON/JSONB เขียนเป็น string แล้ว cast ด้วย `::json` หรือ `::jsonb`:

```sql
SELECT '{"name": "iPhone", "price": 42900}'::jsonb;
```

```
                jsonb
--------------------------------------
 {"name": "iPhone", "price": 42900}
(1 row)
```

รองรับทุกชนิดของ JSON value รวมถึง scalar เดี่ยวๆ (ตั้งแต่ PostgreSQL 9.4 ขึ้นไป มาตรฐาน JSON อนุญาตให้ top-level เป็น scalar ได้ ไม่จำเป็นต้องเป็น object หรือ array เท่านั้น):

```sql
SELECT
    '"hello"'::jsonb   AS a_string,
    '42'::jsonb        AS a_number,
    'true'::jsonb      AS a_boolean,
    'null'::jsonb      AS a_null,
    '[1,2,3]'::jsonb   AS an_array;
```

```
 a_string | a_number | a_boolean | a_null | an_array
----------+----------+-----------+--------+-----------
 "hello"  | 42       | true      | null   | [1, 2, 3]
(1 row)
```

> **ข้อควรระวัง**: `'null'::jsonb` คือค่า JSON null ที่อยู่**ข้างใน**ค่า jsonb (ไม่เท่ากับ SQL `NULL`) ส่วน `NULL::jsonb` คือ SQL NULL ธรรมดาที่ไม่มีค่าอะไรอยู่เลย ทั้งสองต่างกันเวลาใช้ `IS NULL` เทียบกับ `= 'null'::jsonb`

### 2. Automatic validation

เมื่อ cast string เป็น `json` หรือ `jsonb`, PostgreSQL จะ **validate syntax ให้อัตโนมัติ** — ถ้าข้อความไม่ใช่ JSON ที่ถูกต้องตามมาตรฐาน (RFC 8259) จะเกิด error ทันที ป้องกันไม่ให้ข้อมูลเสียหลุดเข้าตารางได้เลย:

```sql
SELECT '{"name": "iPhone", price: 42900}'::jsonb;  -- key "price" ไม่มี quote ครอบ
```

```
ERROR:  invalid input syntax for type json
LINE 1: SELECT '{"name": "iPhone", price: 42900}'::jsonb;
                                    ^
DETAIL:  Token "price" is invalid.
CONTEXT:  JSON data, line 1: {"name": "iPhone", price...
```

```sql
SELECT '{"name": "iPhone",}'::jsonb;  -- มี comma เกินก่อนปิด }
```

```
ERROR:  invalid input syntax for type json
DETAIL:  Expected string or "}", but found "}".
```

นี่คือข้อดีสำคัญของการเก็บ JSON ในคอลัมน์ที่ type เป็น `json`/`jsonb` แทนที่จะเก็บเป็น `text` ธรรมดา — เรามั่นใจได้ว่าทุกแถวในคอลัมน์นี้เป็น **JSON ที่ถูกต้องตามรูปแบบเสมอ** โดยไม่ต้องเขียน validation logic เองในแอปพลิเคชัน

### 3. INSERT ค่า JSON/JSONB เข้าตาราง

เนื่องจากคอลัมน์ `attributes` และ `metadata` ถูกประกาศเป็น `JSONB` อยู่แล้ว การ INSERT string literal ธรรมดาจะถูก cast ให้อัตโนมัติ ไม่จำเป็นต้องเขียน `::jsonb` ก็ได้ (แต่การเขียนกำกับไว้ชัดเจนก็เป็น practice ที่ดี):

```sql
INSERT INTO products
    (product_name, category_id, supplier_id, unit_price, stock_quantity, attributes)
VALUES
    ('Bluetooth Speaker Mini', 1, 2, 1190.00, 60,
     '{"color":"orange","weight_kg":0.35,"specs":{"battery_life_hr":12,"waterproof":true}}');
```

```
INSERT 0 1
```

ตรวจสอบผลลัพธ์:

```sql
SELECT product_id, product_name, attributes
FROM products
WHERE product_name = 'Bluetooth Speaker Mini';
```

```
 product_id |      product_name      |                              attributes
------------+-------------------------+------------------------------------------------------------------------
         21 | Bluetooth Speaker Mini | {"color": "orange", "weight_kg": 0.35, "specs": {"battery_life_hr": 12, "waterproof": true}}
(1 row)
```

### 4. สร้าง JSONB จาก parameter ในแอปพลิเคชัน (แนวทางที่ปลอดภัยกว่า)

ในการเขียนโค้ดจริง เราไม่ควรต่อ string ด้วยมือ (เสี่ยง SQL injection และ escaping ผิดพลาด) แต่ควรใช้ parameterized query ผ่าน driver ของภาษาที่ใช้ เช่น ใน Node.js (`node-postgres`):

```js
await pool.query(
  'INSERT INTO products (product_name, unit_price, attributes) VALUES ($1, $2, $3)',
  ['Test Item', 100, JSON.stringify({ color: 'green', size: 'M' })]
);
```

หรือในฝั่ง SQL ล้วนๆ สามารถใช้ `json_build_object` แทนการเขียน literal ก็ได้ (จะสอนละเอียดใน Step 507):

```sql
INSERT INTO products (product_name, unit_price, attributes)
VALUES ('Test Item', 100, json_build_object('color', 'green', 'size', 'M'));
```

```
INSERT 0 1
```

### 5. ลบข้อมูลทดสอบออกก่อนไปต่อ

```sql
DELETE FROM products WHERE product_name IN ('Bluetooth Speaker Mini', 'Test Item');
```

```
DELETE 2
```

(เราลบทิ้งเพื่อให้จำนวนแถวใน `products` กลับไปเป็น 20 แถวตามชุดข้อมูลตั้งต้น เพื่อให้ตัวอย่างในบทถัดๆ ไปมี product_id ตรงกับที่อ้างอิงไว้)

---

## Step 504: การเข้าถึงค่าใน JSON ด้วย `->` และ `->>`

PostgreSQL มี operator สองตัวหลักสำหรับดึงค่าจาก JSON/JSONB ตาม key หรือ index:

| Operator | ความหมาย | ชนิดผลลัพธ์ |
|---|---|---|
| `->` | ดึงค่าตาม key (object) หรือ index (array) | คืนค่าเป็น `json`/`jsonb` (ยังเป็น JSON อยู่) |
| `->>` | ดึงค่าตาม key (object) หรือ index (array) | คืนค่าเป็น `text` (แปลงเป็นข้อความแล้ว) |

### 1. `->` ดึงค่าจาก object โดย key (คืน JSON)

```sql
SELECT product_name, attributes -> 'color' AS color_json
FROM products
WHERE product_id = 6;
```

```
     product_name     | color_json
-----------------------+------------
 Men's Cotton T-Shirt | "navy"
(1 row)
```

สังเกตว่าผลลัพธ์คือ `"navy"` ที่**ยังมี double quote ล้อมอยู่** เพราะ `->` คืนค่าเป็นชนิด `jsonb` เสมอ (แม้ค่าข้างในจะเป็น string ก็ตาม มันก็ยังเป็น "JSON string" ไม่ใช่ SQL text ธรรมดา)

### 2. `->>` ดึงค่าจาก object โดย key (คืน text)

```sql
SELECT product_name, attributes ->> 'color' AS color_text
FROM products
WHERE product_id = 6;
```

```
     product_name     | color_text
-----------------------+------------
 Men's Cotton T-Shirt | navy
(1 row)
```

คราวนี้ได้ `navy` แบบไม่มี quote เพราะ `->>` แปลงเป็น `text` แท้ๆ ทันที — นี่คือ operator ที่ใช้บ่อยที่สุดเวลาต้องการนำค่าไปใช้ต่อในเงื่อนไข `WHERE`, `ORDER BY`, หรือแสดงผลให้ผู้ใช้เห็น

### 3. เมื่อไหร่ใช้ `->` เมื่อไหร่ใช้ `->>`

- ใช้ **`->>`** เมื่อต้องการ**ค่าสุดท้าย**ไปเปรียบเทียบ, filter, หรือแสดงผล (ต้องการ text/number/boolean จริงๆ)
- ใช้ **`->`** เมื่อต้องการ**เจาะลึกต่อไปอีกชั้น** เพราะ `->>` คืน `text` ซึ่งไม่สามารถ chain ต่อด้วย `->` หรือ `->>` ได้อีก (ต้องใช้กับ JSON เท่านั้น)

ตัวอย่างการเจาะ nested path หลายชั้น — เข้าถึง `attributes.specs.battery`:

```sql
SELECT
    product_name,
    attributes -> 'specs' AS specs_json,          -- ชั้นที่ 1: ได้ JSON object ของ specs
    attributes -> 'specs' ->> 'battery' AS battery -- ชั้นที่ 2: เจาะเข้า specs แล้วดึง battery เป็น text
FROM products
WHERE product_id = 1;
```

```
     product_name     |                         specs_json                          | battery
-----------------------+---------------------------------------------------------------+----------
 iPhone 15 Pro Max     | {"battery": "4441mAh", "screen_inch": 6.7, "5g": true}        | 4441mAh
(1 row)
```

สังเกต pattern: **ใช้ `->` เพื่อเดินไปทีละชั้น แล้วปิดท้ายด้วย `->>` เมื่อถึงชั้นสุดท้ายที่ต้องการค่า text จริงๆ เท่านั้น**

> **ข้อควรระวัง**: ถ้าเขียน `attributes ->> 'specs' ->> 'battery'` (ใช้ `->>` ทั้งสองจุด) จะเกิด `ERROR: operator does not exist: text ->> unknown` ทันที เพราะ `attributes ->> 'specs'` คืน `text` ไปแล้ว จะเอา `->>` ไปต่อกับ `text` ไม่ได้อีก — ต้องใช้ `->` ระหว่างทางเสมอ แล้วปิดท้ายด้วย `->>` ที่ชั้นสุดท้ายเท่านั้น

### 4. เข้าถึง array element ด้วย index

`->` และ `->>` ใช้กับ array ได้เช่นกัน โดยระบุ **ตำแหน่ง index แบบ 0-based** (เริ่มนับจาก 0 ไม่ใช่ 1 แบบ SQL ทั่วไป):

```sql
SELECT
    product_name,
    attributes -> 'tags' AS all_tags,
    attributes -> 'tags' -> 0 AS first_tag_json,
    attributes -> 'tags' ->> 0 AS first_tag_text
FROM products
WHERE product_id = 1;
```

```
     product_name     |            all_tags            | first_tag_json | first_tag_text
-----------------------+----------------------------------+-----------------+-----------------
 iPhone 15 Pro Max     | ["bestseller", "5g", "new"]      | "bestseller"    | bestseller
(1 row)
```

ถ้า index เกินขอบเขต array จะได้ `NULL` (SQL NULL ไม่ใช่ JSON null) ไม่ error:

```sql
SELECT attributes -> 'tags' ->> 10 AS out_of_bounds
FROM products WHERE product_id = 1;
```

```
 out_of_bounds
----------------

(1 row)
```

### 5. ใช้ `->>` ใน `WHERE` clause เพื่อ filter สินค้า

`->>` ใช้ใน `WHERE` ได้เหมือนคอลัมน์ทั่วไป และเมื่อเปรียบเทียบกับ boolean/number จริงๆ ต้อง cast กลับให้ตรงชนิด (เพราะ `->>` คืน `text` เสมอ):

```sql
SELECT product_name, unit_price, attributes ->> 'color' AS color
FROM products
WHERE attributes ->> 'color' = 'black'
   OR (attributes -> 'specs' ->> '5g')::boolean = true;
```

```
      product_name        | unit_price | color
----------------------------+------------+--------
 iPhone 15 Pro Max         |   42900.00 |
 Samsung Galaxy S24 Ultra  |   39900.00 | black
 Sony WH-1000XM5 Headphones|   12900.00 | black
 Women's Yoga Pants        |     690.00 | black
 Wireless Mouse            |     590.00 | black
(5 rows)
```

### 6. ใช้กับ `orders.metadata` — ดึง referrer และ coupon

```sql
SELECT order_id, status, metadata ->> 'referrer' AS referrer, metadata ->> 'coupon_code' AS coupon
FROM orders
ORDER BY order_id
LIMIT 6;
```

```
 order_id |  status   |   referrer      | coupon
----------+-----------+------------------+---------
        1 | paid      | facebook         | SALE10
        2 | completed | google           |
        3 | completed | instagram        |
        4 | shipped   | friend_referral  | NEWYEAR20
        5 | paid      | tiktok           |
        6 | completed | email_campaign   | SALE10
(6 rows)
```

สังเกต order_id 2: คอลัมน์ `coupon` ว่าง เพราะค่า `metadata.coupon_code` ถูกเก็บเป็น **JSON `null`** ไม่ใช่ไม่มี key นี้เลย — `->>'coupon_code'` บนค่า JSON null จะคืน SQL `NULL` เช่นกัน (ทั้งกรณี "ไม่มี key" และ "key มีค่าเป็น null" จะให้ผลลัพธ์เหมือนกันเมื่อใช้ `->>`)

---

## Step 505: `#>` และ `#>>` สำหรับเข้าถึง path ที่ลึกหลายชั้น

เมื่อ path ที่ต้องการเจาะลึกมีหลายชั้นมาก การเขียน `->` ต่อกันยาวๆ (`a -> 'b' -> 'c' -> 'd'`) จะอ่านยากขึ้นเรื่อยๆ PostgreSQL จึงมี operator แบบ **path-based** ที่รับ path ทั้งหมดเป็น **text array ในครั้งเดียว**:

| Operator | ความหมาย | ชนิดผลลัพธ์ |
|---|---|---|
| `#>` | เข้าถึงค่าตาม path ที่ระบุ (array ของ key/index ทีละชั้น) | `json`/`jsonb` |
| `#>>` | เข้าถึงค่าตาม path ที่ระบุ | `text` |

### 1. ตัวอย่างพื้นฐาน — เปรียบเทียบกับ `->` ต่อเนื่อง

```sql
SELECT
    product_name,
    attributes -> 'specs' -> 'battery'   AS via_arrow,   -- แบบเดิม (ได้ jsonb)
    attributes #> '{specs,battery}'      AS via_hash,    -- แบบ path (ได้ jsonb)
    attributes #>> '{specs,battery}'     AS via_hash_txt -- แบบ path คืน text
FROM products
WHERE product_id = 1;
```

```
     product_name     | via_arrow    | via_hash   | via_hash_txt
-----------------------+--------------+------------+---------------
 iPhone 15 Pro Max     | "4441mAh"    | "4441mAh"  | 4441mAh
(1 row)
```

ผลลัพธ์ของ `via_arrow` และ `via_hash` เหมือนกันทุกประการ แต่ syntax ของ `#>` เขียน path เป็น `'{key1,key2,key3}'` (text array literal ของ PostgreSQL) ในครั้งเดียว อ่านง่ายกว่าเมื่อ path ลึกหลายชั้น ส่วน `#>>` (แบบเดียวกับ `->>`) คืนผลเป็น `text` ตรงๆ โดยไม่มี quote ครอบ ซึ่งเป็นรูปแบบที่ใช้บ่อยที่สุดในทางปฏิบัติ

### 2. เจาะลึกหลายชั้นพร้อม array index ผสมกัน

path array สามารถผสม key (สำหรับ object) และ index แบบตัวเลข (สำหรับ array) ไว้ด้วยกันได้:

```sql
SELECT
    product_name,
    attributes #>> '{tags,0}' AS first_tag,   -- key "tags" แล้วตามด้วย index 0
    attributes #>> '{tags,2}' AS third_tag
FROM products
WHERE product_id = 1;
```

```
     product_name     | first_tag   | third_tag
-----------------------+-------------+------------
 iPhone 15 Pro Max     | bestseller  | new
(1 row)
```

### 3. Path ที่ไม่มีอยู่จริง — คืน NULL อย่างปลอดภัย

ข้อดีสำคัญของ `#>` / `#>>` (และ `->` / `->>` ด้วย) คือ **ไม่เกิด error เมื่อ path ไม่มีอยู่จริง** แต่คืน `NULL` แทน ซึ่งปลอดภัยกว่าการเขียน error handling เอง:

```sql
SELECT
    product_name,
    attributes #>> '{specs,cpu,cores}' AS nonexistent_deep_path
FROM products
WHERE product_id = 6;  -- เสื้อยืด ไม่มี specs เลย
```

```
     product_name     | nonexistent_deep_path
-----------------------+------------------------
 Men's Cotton T-Shirt |
(1 row)
```

พฤติกรรมนี้สำคัญมากเวลาทำงานกับตารางที่แต่ละแถวมีโครงสร้าง JSONB ต่างกัน (เหมือนตาราง `products` ของเรา) เพราะเราไม่ต้องเขียน `CASE WHEN` เช็คว่า key มีอยู่จริงก่อนทุกครั้ง

### 4. ตัวอย่างใช้งานจริง — `orders.metadata` และการ cast ชนิดข้อมูล

เจาะ path ลึกของ `orders.metadata` (`utm.source`, `utm.campaign`) และใช้ `#>>` ร่วมกับ `(attributes #>> '{weight_kg}')::numeric` เพื่อเปรียบเทียบเชิงตัวเลข (เพราะผลลัพธ์ของ `#>>` เป็น `text` เสมอ จึงต้อง cast กลับให้ตรงชนิดทุกครั้งก่อนนำไปคำนวณหรือเปรียบเทียบ):

```sql
SELECT order_id, metadata #>> '{utm,source}' AS utm_source, metadata #>> '{utm,campaign}' AS utm_campaign
FROM orders
WHERE metadata ? 'utm'
ORDER BY order_id;
```

```
 order_id | utm_source |  utm_campaign
----------+------------+-----------------
        6 | newsletter | jan_sale
       17 | newsletter | feb_promo
(2 rows)
```

> **หมายเหตุ**: operator `?` ในตัวอย่างข้างต้น (เช็คว่ามี key อยู่ไหม) เป็นหนึ่งใน containment operator ของ JSONB ที่จะอธิบายละเอียดใน [Part 052](./part-052-jsonb-advanced.md)

อีกตัวอย่างหนึ่งของการ cast กลับเป็น `numeric` เพื่อกรองและเรียงลำดับเชิงตัวเลข:

```sql
SELECT
    product_name,
    (attributes #>> '{weight_kg}')::numeric AS weight_kg
FROM products
WHERE (attributes #>> '{weight_kg}')::numeric > 1.0
ORDER BY weight_kg DESC;
```

```
     product_name     | weight_kg
-----------------------+-----------
 MacBook Pro 14" M3    |      1.55
 Stand Mixer 5L        |      6.50
 Air Fryer 4.5L        |      4.20
 Dell XPS 13           |      1.24
 Yoga Mat Premium      |      1.20
(5 rows)
```

### สรุปเปรียบเทียบ `->`/`->>`  กับ `#>`/`#>>`

| สถานการณ์ | แนะนำให้ใช้ |
|---|---|
| เข้าถึงแค่ 1 ชั้น | `->` หรือ `->>` |
| เข้าถึงหลายชั้น (2 ชั้นขึ้นไป) | `#>` หรือ `#>>` อ่านง่ายกว่า |
| path ถูกสร้างแบบ dynamic (เก็บเป็นตัวแปร array) | `#>` / `#>>` เพราะรับ array ได้โดยตรง |
| ต้องการผลลัพธ์เป็น JSON เพื่อเจาะต่อ | `->` หรือ `#>` |
| ต้องการผลลัพธ์เป็น text/ค่าสุดท้าย | `->>` หรือ `#>>` |

---

## Step 506: `jsonb_set` — การแก้ไขค่าบางส่วนใน JSONB

ถ้าต้องการแก้ไข field เดียวใน JSONB โดยไม่กระทบ field อื่น เราไม่จำเป็นต้องเขียนค่าทั้งก้อนใหม่ — ใช้ฟังก์ชัน `jsonb_set()` ได้เลย

### 1. Syntax

```sql
jsonb_set(target jsonb, path text[], new_value jsonb [, create_missing boolean DEFAULT true])
```

- `target`: ค่า JSONB ต้นฉบับ
- `path`: array ของ key/index ที่จะไปถึง field ที่ต้องการแก้ (เหมือน path ของ `#>`)
- `new_value`: ค่าใหม่ที่จะแทนที่ (ต้องเป็น jsonb)
- `create_missing`: ถ้า path นั้นไม่มีอยู่จริง จะสร้างขึ้นใหม่หรือไม่ (default `true`)

### 2. ตัวอย่างพื้นฐาน — แก้ไข stock กับ 1 field ระดับบนสุด

```sql
SELECT
    product_name,
    attributes AS before,
    jsonb_set(attributes, '{color}', '"red"') AS after
FROM products
WHERE product_id = 6;
```

```
     product_name     |                          before                          |                          after
-----------------------+------------------------------------------------------------+-----------------------------------------------------------
 Men's Cotton T-Shirt | {"color": "navy", "size": "L", "material": "100% cotton", "weight_kg": 0.2} | {"color": "red", "size": "L", "material": "100% cotton", "weight_kg": 0.2}
(1 row)
```

สังเกตว่ามีแค่ `color` ที่เปลี่ยนจาก `navy` เป็น `red` ส่วน `size`, `material`, `weight_kg` ยังอยู่ครบเหมือนเดิม — นี่คือข้อดีหลักของ `jsonb_set` เทียบกับการ `UPDATE ... SET attributes = '...ทั้งก้อนใหม่...'`

> **ข้อควรระวังเรื่อง syntax**: ค่าที่เป็น string ต้องเขียนเป็น JSON string ที่มี double quote ล้อม เช่น `'"red"'` (ไม่ใช่ `'red'` เฉยๆ) เพราะ parameter ตัวที่ 3 ของ `jsonb_set` ต้องเป็นชนิด `jsonb` เสมอ

### 3. แก้ไขค่าที่ซ้อนลึก (nested path)

```sql
SELECT
    product_name,
    jsonb_set(attributes, '{specs,battery}', '"5352mAh"') AS updated_battery
FROM products
WHERE product_id = 1;
```

```
     product_name     |                                              updated_battery
-----------------------+---------------------------------------------------------------------------------------------------------
 iPhone 15 Pro Max     | {"color": "titanium blue", "storage_gb": 256, "weight_kg": 0.221, "tags": [...], "specs": {"battery": "5352mAh", "screen_inch": 6.7, "5g": true}}
(1 row)
```

field `specs.battery` เปลี่ยนไปตามที่ต้องการ ส่วน `specs.screen_inch`, `specs.5g` และ field อื่นๆ นอก `specs` ไม่กระทบเลย

### 4. UPDATE ตารางจริงด้วย `jsonb_set`

ตัวอย่าง: ลดราคาสต็อกสินค้า และอัปเดต `attributes.weight_kg` ของสินค้าหนึ่งรายการหลังจากชั่งน้ำหนักใหม่:

```sql
UPDATE products
SET attributes = jsonb_set(attributes, '{weight_kg}', '0.225')
WHERE product_id = 1;
```

```
UPDATE 1
```

ตรวจสอบผล:

```sql
SELECT product_name, attributes ->> 'weight_kg' AS weight_kg
FROM products WHERE product_id = 1;
```

```
     product_name     | weight_kg
-----------------------+-----------
 iPhone 15 Pro Max     | 0.225
(1 row)
```

### 5. `create_missing` — สร้าง key ใหม่ถ้ายังไม่มี

ถ้า `create_missing = true` (ค่า default) และ path ที่ระบุยังไม่มีอยู่ใน JSONB, PostgreSQL จะ**สร้าง key นั้นขึ้นมาใหม่**:

```sql
SELECT jsonb_set(
    '{"color":"red"}'::jsonb,
    '{size}',
    '"XL"',
    true   -- create_missing = true (default)
) AS with_new_key;
```

```
      with_new_key
-------------------------
 {"color": "red", "size": "XL"}
(1 row)
```

ถ้าตั้ง `create_missing = false` และ key ไม่มีอยู่จริง ค่าจะไม่เปลี่ยนแปลง (ไม่สร้างเพิ่ม):

```sql
SELECT jsonb_set(
    '{"color":"red"}'::jsonb,
    '{size}',
    '"XL"',
    false   -- ห้ามสร้าง key ใหม่
) AS without_new_key;
```

```
   without_new_key
-----------------------
 {"color": "red"}
(1 row)
```

> **ข้อควรระวัง**: ถ้า path ตรงกลางไม่มีอยู่จริง (เช่น อยากแก้ `{specs,battery}` แต่ `specs` เองยังไม่มีเลย) `jsonb_set` จะ **ไม่สร้างให้** แม้ `create_missing = true` เพราะมันสร้างได้แค่ key สุดท้ายของ path เท่านั้น ไม่ใช่ทั้งสาย path:

```sql
SELECT jsonb_set(
    '{"color":"navy"}'::jsonb,   -- ไม่มี "specs" อยู่เลย
    '{specs,battery}',
    '"1000mAh"'
) AS result;
```

```
        result
-----------------------
 {"color": "navy"}
(1 row)
```

ไม่มีอะไรเปลี่ยนแปลง เพราะ path กลาง (`specs`) ไม่มีอยู่จริง ถ้าต้องการเพิ่ม nested object ใหม่ทั้งชั้น ให้ใส่ค่าทั้งก้อนที่ level แรกแทน:

```sql
SELECT jsonb_set(
    '{"color":"navy"}'::jsonb,
    '{specs}',
    '{"battery":"1000mAh"}'
) AS result;
```

```
                result
---------------------------------------
 {"color": "navy", "specs": {"battery": "1000mAh"}}
(1 row)
```

### 6. ตัวอย่างใช้งานจริง — เพิ่มคูปองส่วนลดใหม่ให้คำสั่งซื้อที่ยังไม่มี

```sql
UPDATE orders
SET metadata = jsonb_set(metadata, '{coupon_code}', '"LOYALTY5"')
WHERE order_id = 3
RETURNING order_id, metadata;
```

```
 order_id |                              metadata
----------+----------------------------------------------------------------------
        3 | {"referrer": "instagram", "gift_wrap": false, "coupon_code": "LOYALTY5"}
(1 row)
```

order_id 3 เดิมไม่มี key `coupon_code` เลย แต่ `jsonb_set` สร้าง key ใหม่ให้อัตโนมัติ (เพราะ `create_missing` เป็น `true` โดย default) โดยไม่กระทบ `referrer` และ `gift_wrap` ที่มีอยู่เดิม

---

## Step 507: การสร้าง JSON จากผลลัพธ์ query

เมื่อต้อง export ข้อมูลจากตาราง SQL ปกติไปเป็น JSON (เช่นเพื่อส่งให้ REST API หรือ frontend) PostgreSQL มีฟังก์ชันสำเร็จรูปให้ใช้โดยไม่ต้องประกอบ string เอง

### 1. `row_to_json` — แปลงหนึ่งแถวเป็น JSON object

```sql
SELECT row_to_json(c) AS customer_json
FROM (
    SELECT customer_id, first_name, last_name, country
    FROM customers
    WHERE customer_id = 1
) c;
```

```
                                 customer_json
---------------------------------------------------------------------------------
 {"customer_id":1,"first_name":"Somchai","last_name":"Jaidee","country":"Thailand"}
(1 row)
```

`row_to_json` รับ **row/record type** เป็น input (ในที่นี้คือ alias `c` ของ subquery) แล้วแปลง column ทุกตัวของแถวนั้นเป็น key-value ใน JSON object — คืนผลลัพธ์เป็นชนิด `json` (ไม่ใช่ `jsonb`)

### 2. `to_jsonb` — แปลงค่าใดๆ (รวมถึงทั้งแถว) เป็น jsonb

`to_jsonb` ทำงานคล้าย `row_to_json` แต่ยืดหยุ่นกว่า เพราะรับได้ทั้ง scalar, array, และ row แล้วคืนผลเป็น `jsonb` (ซึ่งเป็นชนิดที่เราอยากได้บ่อยกว่าในการทำงานต่อ):

```sql
SELECT to_jsonb(p) AS product_json
FROM (
    SELECT product_id, product_name, unit_price
    FROM products
    WHERE product_id = 1
) p;
```

```
                                product_json
------------------------------------------------------------------------------
 {"product_id": 1, "product_name": "iPhone 15 Pro Max", "unit_price": 42900.00}
(1 row)
```

`to_jsonb` ยังใช้แปลง scalar เดี่ยวๆ ได้ด้วย เช่น:

```sql
SELECT to_jsonb(ARRAY[1,2,3]) AS arr, to_jsonb('hello'::text) AS str, to_jsonb(42) AS num;
```

```
    arr    |   str   | num
-----------+---------+------
 [1, 2, 3] | "hello" |   42
(1 row)
```

### 3. `json_build_object` / `jsonb_build_object` — สร้าง JSON object กำหนดเอง

เมื่อไม่ต้องการ column ทั้งหมดของแถว หรือต้องการตั้งชื่อ key เอง หรือสร้างโครงสร้างซ้อนกันเอง ใช้ `json_build_object` (คืน `json`) หรือ `jsonb_build_object` (คืน `jsonb`) — รับ argument เป็นคู่ `key, value, key, value, ...`:

```sql
SELECT jsonb_build_object(
    'id', product_id,
    'name', product_name,
    'price', unit_price,
    'in_stock', stock_quantity > 0
) AS product_summary
FROM products
WHERE product_id IN (1, 6);
```

```
                              product_summary
------------------------------------------------------------------------------
 {"id": 1, "name": "iPhone 15 Pro Max", "price": 42900.00, "in_stock": true}
 {"id": 6, "name": "Men's Cotton T-Shirt", "price": 350.00, "in_stock": true}
(2 rows)
```

สามารถสร้าง object ซ้อนกันได้โดย nest ฟังก์ชันเข้าไปอีกชั้น เช่นสร้าง response ที่มี `category` เป็น object ย่อย:

```sql
SELECT jsonb_build_object(
    'product_id', p.product_id,
    'name', p.product_name,
    'category', jsonb_build_object(
        'id', c.category_id,
        'name', c.category_name
    ),
    'attributes', p.attributes
) AS full_product
FROM products p
JOIN categories c ON c.category_id = p.category_id
WHERE p.product_id = 1;
```

```
                                                      full_product
----------------------------------------------------------------------------------------------------------------------
 {"product_id": 1, "name": "iPhone 15 Pro Max", "category": {"id": 3, "name": "Mobile Phones"}, "attributes": {...}}
(1 row)
```

### 4. `json_agg` / `jsonb_agg` — รวมหลายแถวเป็น JSON array

`json_agg` เป็น aggregate function ที่รวบรวมค่าจากหลายแถว (เหมือน `SUM`, `COUNT`) ให้กลายเป็น **JSON array เดียว**:

```sql
SELECT jsonb_agg(product_name) AS product_names
FROM products
WHERE category_id = 7;  -- หมวดหนังสือ
```

```
                          product_names
--------------------------------------------------------------------
 ["Clean Code", "Atomic Habits", "หนังสือ Python เบื้องต้น"]
(1 row)
```

รวมกับ `jsonb_build_object` เพื่อสร้าง array ของ object (pattern ที่ใช้บ่อยมากเวลา export ข้อมูลเป็น JSON สำหรับ API):

```sql
SELECT jsonb_agg(
    jsonb_build_object('id', product_id, 'name', product_name, 'price', unit_price)
) AS books_json
FROM products
WHERE category_id = 7;
```

```
                                                    books_json
---------------------------------------------------------------------------------------------------------------
 [{"id": 10, "name": "Clean Code", "price": 890.00}, {"id": 11, "name": "Atomic Habits", "price": 450.00}, {"id": 12, "name": "หนังสือ Python เบื้องต้น", "price": 350.00}]
(1 row)
```

### 5. ตัวอย่าง real-world — สร้าง JSON ของคำสั่งซื้อพร้อมรายการสินค้า (nested aggregation)

โจทย์ยอดฮิตเมื่อทำ REST API: ต้องการ endpoint ที่คืนคำสั่งซื้อ 1 ใบ พร้อม array ของ order_items ทั้งหมดในใบนั้น รวมอยู่ใน JSON object เดียว:

```sql
SELECT jsonb_build_object(
    'order_id', o.order_id,
    'status', o.status,
    'order_date', o.order_date,
    'metadata', o.metadata,
    'items', (
        SELECT jsonb_agg(
            jsonb_build_object(
                'product_name', p.product_name,
                'quantity', oi.quantity,
                'unit_price', oi.unit_price,
                'subtotal', oi.quantity * oi.unit_price
            )
        )
        FROM order_items oi
        JOIN products p ON p.product_id = oi.product_id
        WHERE oi.order_id = o.order_id
    )
) AS order_json
FROM orders o
WHERE o.order_id = 1;
```

```
                                                          order_json
---------------------------------------------------------------------------------------------------------------------------------
 {"order_id": 1, "status": "paid", "order_date": "2024-01-05T10:00:00+07:00",
  "metadata": {"coupon_code": "SALE10", "referrer": "facebook", "gift_wrap": false},
  "items": [
    {"product_name": "iPhone 15 Pro Max", "quantity": 1, "unit_price": 42900.00, "subtotal": 42900.00},
    {"product_name": "Wireless Mouse", "quantity": 1, "unit_price": 590.00, "subtotal": 590.00}
  ]}
(1 row)
```

นี่คือรูปแบบ query ที่ทรงพลังมาก — เราสร้าง JSON response ที่มีโครงสร้างซ้อนกัน (nested) ได้ตรงตามที่ API ต้องการ **ในคำสั่ง SQL เดียว** โดยไม่ต้องเขียนโค้ด backend มา loop join ข้อมูลเอง

### สรุปฟังก์ชันแปลง SQL → JSON

| ฟังก์ชัน | Input | Output | ใช้เมื่อ |
|---|---|---|---|
| `row_to_json(record)` | 1 แถว | `json` | ต้องการทุกคอลัมน์ของแถวนั้นเป็น JSON เร็วๆ |
| `to_jsonb(any)` | scalar / array / record | `jsonb` | ทั่วไป ยืดหยุ่นกว่า และได้ `jsonb` มาทำงานต่อ |
| `json_build_object(...)` / `jsonb_build_object(...)` | คู่ key, value | `json`/`jsonb` | กำหนด key เอง หรือสร้าง nested object |
| `json_agg(expr)` / `jsonb_agg(expr)` | หลายแถว (aggregate) | `json`/`jsonb` array | รวมหลายแถวเป็น JSON array เดียว |

---

## Step 508: `jsonb_each`, `jsonb_object_keys`, `jsonb_array_elements`

บางครั้งเราต้องการทำทิศทางตรงข้ามกับ Step 507 — คือแตก (expand) ค่า JSONB ก้อนเดียวให้กลายเป็นหลายแถวของผลลัพธ์ SQL ปกติ เพื่อนำไป `JOIN`, `GROUP BY`, หรือ filter ต่อ

### 1. `jsonb_object_keys` — ดึงรายชื่อ key ทั้งหมดออกมาเป็นแถว

```sql
SELECT jsonb_object_keys(attributes) AS attribute_key
FROM products
WHERE product_id = 1;
```

```
 attribute_key
----------------
 color
 storage_gb
 weight_kg
 tags
 specs
(5 rows)
```

ใช้บ่อยเวลาต้องการสำรวจว่า JSONB column มี key อะไรบ้างในข้อมูลจริง (เพราะแต่ละแถวอาจมี key ไม่เหมือนกัน) — เช่น หา **key ทั้งหมดที่เคยถูกใช้ใน `attributes` ของทุกสินค้า**:

```sql
SELECT DISTINCT jsonb_object_keys(attributes) AS all_keys_used
FROM products
ORDER BY all_keys_used;
```

```
 all_keys_used
----------------
 capacity_liters
 color
 format
 isbn
 language
 material
 pages
 size
 skin_type
 specs
 storage_gb
 tags
 volume_ml
 weight_kg
 ram_gb
(15 rows)
```

query นี้มีประโยชน์มากในการทำ **schema discovery** — เวลาเราสืบทอดตาราง JSONB ที่มีข้อมูลหลากหลายมาก่อน แล้วอยากรู้ว่าจริงๆ แล้วมี field อะไรบ้างที่เคยถูกใช้งาน

### 2. `jsonb_each` — แตก object เป็นคู่ key-value (คืน key เป็น text, value เป็น jsonb)

```sql
SELECT *
FROM jsonb_each(
    (SELECT attributes FROM products WHERE product_id = 6)
);
```

```
    key    |   value
-----------+------------
 color     | "navy"
 size      | "L"
 material  | "100% cotton"
 weight_kg | 0.2
(4 rows)
```

`jsonb_each` คืน table 2 คอลัมน์: `key` (ชนิด `text`) และ `value` (ชนิด `jsonb`) — ถ้าต้องการ value เป็น `text` ตรงๆ (ไม่มี quote ครอบ string) ให้ใช้ `jsonb_each_text` แทน:

```sql
SELECT *
FROM jsonb_each_text(
    (SELECT attributes FROM products WHERE product_id = 6)
);
```

```
    key    |   value
-----------+---------------
 color     | navy
 size      | L
 material  | 100% cotton
 weight_kg | 0.2
(4 rows)
```

### 3. ใช้ `jsonb_each` ร่วมกับ `LATERAL JOIN` — แตก attributes ของทุกสินค้าพร้อมกัน

การเรียก `jsonb_each` แบบ subquery ตรงๆ ใช้ได้กับแถวเดียว แต่ถ้าต้องการแตก JSONB ของ **หลายแถวพร้อมกัน** ต้องใช้ `CROSS JOIN LATERAL` (หรือ comma-join กับฟังก์ชันแบบ set-returning ก็ได้ผลเหมือนกัน):

```sql
SELECT p.product_id, p.product_name, kv.key, kv.value
FROM products p
CROSS JOIN LATERAL jsonb_each_text(p.attributes) AS kv
WHERE p.product_id IN (6, 10)
ORDER BY p.product_id, kv.key;
```

```
 product_id |      product_name      |    key    |    value
------------+--------------------------+-----------+---------------
          6 | Men's Cotton T-Shirt    | color     | navy
          6 | Men's Cotton T-Shirt    | material  | 100% cotton
          6 | Men's Cotton T-Shirt    | size      | L
          6 | Men's Cotton T-Shirt    | weight_kg | 0.2
         10 | Clean Code              | format    | paperback
         10 | Clean Code              | isbn      | 978-0132350884
         10 | Clean Code              | language  | English
         10 | Clean Code              | pages     | 464
(8 rows)
```

pattern นี้มีประโยชน์มากเวลาต้องการแปลง JSONB attributes ให้เป็นรูปแบบ **EAV (Entity-Attribute-Value)** เพื่อนำไปวิเคราะห์ในเครื่องมือที่ไม่รองรับ JSON โดยตรง (เช่น BI tool บางตัว หรือ export เป็น CSV แบบ long format)

### 4. `jsonb_array_elements` — แตก array เป็นหลายแถว

ใช้กับค่าที่เป็น **JSON array** (ไม่ใช่ object) เพื่อแตก element แต่ละตัวออกมาเป็นแถว:

```sql
SELECT jsonb_array_elements(attributes -> 'tags') AS tag
FROM products
WHERE product_id = 1;
```

```
    tag
-------------
 "bestseller"
 "5g"
 "new"
(3 rows)
```

ถ้าต้องการ text ล้วนไม่มี quote ให้ใช้ `jsonb_array_elements_text`:

```sql
SELECT jsonb_array_elements_text(attributes -> 'tags') AS tag
FROM products
WHERE product_id = 1;
```

```
    tag
-------------
 bestseller
 5g
 new
(3 rows)
```

### 5. ใช้ `jsonb_array_elements_text` กับ `LATERAL` เพื่อหาสินค้าที่มี tag ร่วมกัน

```sql
SELECT p.product_name, t.tag
FROM products p
CROSS JOIN LATERAL jsonb_array_elements_text(p.attributes -> 'tags') AS t(tag)
WHERE p.attributes ? 'tags'
ORDER BY t.tag, p.product_name;
```

```
      product_name       |    tag
---------------------------+-------------
 iPhone 15 Pro Max        | 5g
 Samsung Galaxy S24 Ultra | 5g
 iPhone 15 Pro Max        | bestseller
 Samsung Galaxy S24 Ultra | bestseller
 Running Shoes Pro        | bestseller
 iPhone 15 Pro Max        | new
 MacBook Pro 14" M3       | premium
(7 rows)
```

จาก query นี้เราสามารถต่อยอดหาว่า **tag ไหนถูกใช้บ่อยที่สุด** ได้ทันทีด้วยการเติม `GROUP BY t.tag, COUNT(*)` ต่อท้าย

### สรุปฟังก์ชันแตก JSONB เป็นแถว

| ฟังก์ชัน | ใช้กับ | คืนค่า |
|---|---|---|
| `jsonb_object_keys(jsonb)` | object | 1 คอลัมน์: key (text) |
| `jsonb_each(jsonb)` | object | 2 คอลัมน์: key (text), value (jsonb) |
| `jsonb_each_text(jsonb)` | object | 2 คอลัมน์: key (text), value (text) |
| `jsonb_array_elements(jsonb)` | array | 1 คอลัมน์: value (jsonb) |
| `jsonb_array_elements_text(jsonb)` | array | 1 คอลัมน์: value (text) |

> ทุกฟังก์ชันข้างต้นเป็น **set-returning function (SRF)** — ต้องใช้ใน `FROM` clause หรือ `SELECT` list โดยตรง (PostgreSQL รองรับ SRF ใน `SELECT` list ได้ แต่แนะนำให้ใช้ผ่าน `FROM ... LATERAL` เพื่อความชัดเจนและควบคุมพฤติกรรมเวลามีหลาย column expression)

---

## Step 509: รวม JSONB (`||`), ลบ key (`-`), และ `jsonb_strip_nulls`

### 1. `||` — รวม (merge) สอง JSONB object เข้าด้วยกัน

Operator `||` รวม key จากทั้งสองฝั่งเข้าด้วยกัน โดย **ถ้ามี key ซ้ำกัน ค่าจากฝั่งขวาจะทับฝั่งซ้าย**:

```sql
SELECT
    '{"color":"navy","size":"L"}'::jsonb
    ||
    '{"size":"XL","material":"cotton"}'::jsonb
    AS merged;
```

```
                     merged
--------------------------------------------------
 {"color": "navy", "size": "XL", "material": "cotton"}
```

`size` มีอยู่ทั้งสองฝั่ง — ผลลัพธ์ใช้ค่าจากฝั่งขวา (`XL`) ทับค่าฝั่งซ้าย (`L`) ส่วน `color` (มีแค่ฝั่งซ้าย) กับ `material` (มีแค่ฝั่งขวา) ถูกรวมเข้ามาทั้งคู่

> **ข้อควรรู้**: `||` merge เฉพาะ **ระดับบนสุด (top level)** เท่านั้น ถ้า value เป็น nested object ที่ซ้ำ key กัน มันจะ**ทับทั้งก้อน ไม่ deep-merge**:

```sql
SELECT
    '{"specs":{"battery":"5000mAh","cpu":"A17"}}'::jsonb
    ||
    '{"specs":{"battery":"6000mAh"}}'::jsonb
    AS shallow_merge;
```

```
               shallow_merge
---------------------------------------
 {"specs": {"battery": "6000mAh"}}
```

สังเกตว่า `specs.cpu` **หายไปทั้งหมด** เพราะ `||` เห็นว่า key `specs` ซ้ำกันที่ระดับบนสุด จึงเอาค่าทั้งก้อนของฝั่งขวามาแทนที่ทั้งก้อนของฝั่งซ้าย ไม่ได้ merge ลึกลงไปถึงข้างใน `specs` — นี่เป็นจุดที่มือใหม่มักเข้าใจผิดบ่อยมาก ถ้าต้องการ deep merge ต้องเขียน logic เพิ่มเอง (หรือใช้ `jsonb_set` ไล่ทีละ field)

### 2. ใช้งานจริง — เพิ่ม field ใหม่เข้าไปใน attributes โดยไม่ลบของเดิม

```sql
UPDATE products
SET attributes = attributes || '{"warranty_months": 12, "eco_friendly": true}'::jsonb
WHERE product_id = 1
RETURNING product_name, attributes;
```

```
     product_name     |                                                         attributes
-----------------------+-------------------------------------------------------------------------------------------------------------------------------
 iPhone 15 Pro Max     | {"color": "titanium blue", "storage_gb": 256, "weight_kg": 0.225, "tags": [...], "specs": {...}, "warranty_months": 12, "eco_friendly": true}
(1 row)
```

field เดิมทั้งหมดยังอยู่ครบ เพิ่มแค่ `warranty_months` และ `eco_friendly` เข้ามาใหม่ — วิธีนี้สะดวกกว่า `jsonb_set` เมื่อต้องการเพิ่มหลาย key พร้อมกันในครั้งเดียว

### 3. `-` — ลบ key ออกจาก JSONB object

Operator `-` (ชนิด `jsonb - text`) ลบ key ที่ระบุออกจาก object ระดับบนสุด:

```sql
SELECT '{"color":"navy","size":"L","material":"cotton"}'::jsonb - 'material' AS result;
```

```
              result
-----------------------------
 {"color": "navy", "size": "L"}
```

**ลบหลาย key พร้อมกัน** ด้วย `-` และ text array (`jsonb - text[]`):

```sql
SELECT '{"color":"navy","size":"L","material":"cotton","weight_kg":0.2}'::jsonb
       - ARRAY['material', 'weight_kg']
       AS result;
```

```
              result
-----------------------------
 {"color": "navy", "size": "L"}
```

**ลบ element ออกจาก array ตาม index** ด้วย `-` และ integer (`jsonb - int`):

```sql
SELECT '["a","b","c"]'::jsonb - 1 AS result;  -- ลบ index 1 คือ "b"
```

```
   result
------------
 ["a", "c"]
```

**ลบ field ที่ซ้อนลึก** ต้องใช้ operator `#-` (path-based delete) แทน เพราะ `-` ทำได้แค่ระดับบนสุด:

```sql
SELECT '{"specs":{"battery":"5000mAh","cpu":"A17"}}'::jsonb #- '{specs,cpu}' AS result;
```

```
                result
-----------------------------------
 {"specs": {"battery": "5000mAh"}}
```

### 4. ใช้งานจริง — ลบ field ที่ผิดพลาดออกจาก metadata คำสั่งซื้อ

```sql
UPDATE orders
SET metadata = metadata - 'cancel_reason'
WHERE order_id = 7
RETURNING order_id, metadata;
```

```
 order_id |                                 metadata
----------+---------------------------------------------------------------------------
        7 | {"referrer": "google", "gift_wrap": false}
(1 row)
```

### 5. `jsonb_strip_nulls` — ล้างค่า key ที่เป็น JSON null ทิ้งทั้งหมด

ฟังก์ชันนี้ไล่ลบทุก key ที่มีค่าเป็น `null` ออกจาก JSONB แบบ **recursive** (ลึกไปทุกชั้น) — มีประโยชน์มากเวลา clean ข้อมูลก่อนบันทึกจริง หรือก่อนส่งออกไปที่ระบบอื่นที่ไม่อยากเห็น key ที่ไม่มีค่า:

```sql
SELECT jsonb_strip_nulls(
    '{"coupon_code": null, "referrer": "google", "gift_wrap": true, "note": null}'::jsonb
) AS cleaned;
```

```
                     cleaned
--------------------------------------------------
 {"referrer": "google", "gift_wrap": true}
```

`coupon_code` และ `note` (ที่มีค่าเป็น `null`) ถูกลบทิ้งไปทั้งคู่ ส่วน key ที่มีค่าจริงยังอยู่ครบ

**ตัวอย่างแบบ recursive** — ลบ null แม้อยู่ในระดับที่ซ้อนลึก:

```sql
SELECT jsonb_strip_nulls(
    '{"a": 1, "b": null, "c": {"d": null, "e": 2}}'::jsonb
) AS cleaned;
```

```
             cleaned
-----------------------------
 {"a": 1, "c": {"e": 2}}
```

key `b` (ระดับบนสุด) และ `c.d` (ระดับซ้อนใน) ถูกลบทั้งคู่ เพราะ `jsonb_strip_nulls` ทำงานแบบ recursive ไล่ทุกชั้น

การใช้ `jsonb_strip_nulls(metadata)` ตรงๆ ในคอลัมน์ของทุกแถวในตาราง (แทนที่จะเทสต์กับ literal เดี่ยวๆ อย่างด้านบน) ก็ทำงานแบบเดียวกัน — เหมาะสำหรับใช้ก่อนส่งข้อมูลออก API ที่ไม่ต้องการเห็น key ว่างๆ ปะปนอยู่

### สรุป operator สำหรับแก้ไข/รวม/ลบ JSONB

| Operator/ฟังก์ชัน | ความหมาย | ตัวอย่าง |
|---|---|---|
| `a \|\| b` | รวม 2 object (shallow merge, ขวาทับซ้าย) | `'{"a":1}' \|\| '{"b":2}'` → `{"a":1,"b":2}` |
| `jsonb - 'key'` | ลบ 1 key ระดับบนสุด | `'{"a":1,"b":2}' - 'a'` → `{"b":2}` |
| `jsonb - ARRAY[...]` | ลบหลาย key ระดับบนสุด | `'{"a":1,"b":2}' - ARRAY['a']` |
| `jsonb - int` | ลบ array element ตาม index | `'[1,2,3]' - 1` → `[1,3]` |
| `jsonb #- '{path}'` | ลบ field ที่ path ลึกกว่า 1 ชั้น | `'{"a":{"b":1}}' #- '{a,b}'` → `{"a":{}}` |
| `jsonb_strip_nulls(jsonb)` | ลบทุก key ที่ค่าเป็น null (recursive) | `'{"a":null,"b":1}'` → `{"b":1}` |

---

## Step 510: แบบฝึกหัดรวม — จัดการ product attributes และ order metadata แบบยืดหยุ่น

มาลองรวมทุกเทคนิคที่เรียนมาในบทนี้ แก้โจทย์ระดับ real-world กัน

### โจทย์ 1: สร้างรายงานสรุปสินค้าอิเล็กทรอนิกส์พร้อม spec สำคัญ

ต้องการรายงานสินค้าในหมวด Electronics/Computers/Mobile Phones (category_id 1, 2, 3) ที่แสดง `battery` หรือ `cpu` ถ้ามี พร้อมจำนวน tag:

```sql
SELECT
    p.product_name,
    c.category_name,
    p.attributes ->> 'color' AS color,
    COALESCE(p.attributes #>> '{specs,battery}', p.attributes #>> '{specs,cpu}', 'n/a') AS key_spec,
    COALESCE(jsonb_array_length(p.attributes -> 'tags'), 0) AS tag_count
FROM products p
JOIN categories c ON c.category_id = p.category_id
WHERE p.category_id IN (1, 2, 3)
ORDER BY p.product_id;
```

```
      product_name       | category_name |    color     | key_spec  | tag_count
--------------------------+----------------+--------------+-----------+------------
 iPhone 15 Pro Max        | Mobile Phones  | titanium blue| 4441mAh   |          3
 Samsung Galaxy S24 Ultra | Mobile Phones  | black        | 5000mAh   |          2
 MacBook Pro 14" M3       | Computers      | space gray   | Apple M3 Pro |       1
 Dell XPS 13              | Computers      | platinum silver | Intel Core i7 |    0
 Sony WH-1000XM5 Headphones | Electronics  | black        | n/a       |          0
 iPad Air                 | Electronics    | blue         | n/a       |          0
 Wireless Mouse           | Electronics    | black        | n/a       |          0
(7 rows)
```

*หมายเหตุ*: `jsonb_array_length()` เป็นฟังก์ชันที่คืนจำนวน element ใน JSON array (จะกล่าวถึงเพิ่มเติมใน Part 052) — ใช้ที่นี่เพื่อนับ tag

### โจทย์ 2: หาคำสั่งซื้อทั้งหมดที่มาจาก social media

```sql
SELECT
    o.order_id,
    cu.first_name || ' ' || cu.last_name AS customer_name,
    o.metadata ->> 'referrer' AS referrer,
    (o.metadata ->> 'gift_wrap')::boolean AS gift_wrap
FROM orders o
JOIN customers cu ON cu.customer_id = o.customer_id
WHERE o.metadata ->> 'referrer' IN ('facebook', 'instagram', 'tiktok')
ORDER BY o.order_id;
```

```
 order_id |  customer_name    |  referrer  | gift_wrap
----------+--------------------+------------+------------
        1 | Somchai Jaidee     | facebook   | false
        3 | John Smith         | instagram  | false
        5 | Wei Zhang          | tiktok     | false
        9 | Pranee Chaiyaporn  | facebook   | false
       13 | Siriporn Boonmee   | tiktok     | false
       16 | David Lee          | facebook   | false
(6 rows)
```

### โจทย์ 3: อัปเดต attributes หลายสินค้าพร้อมกันด้วย field ใหม่ `on_sale`

เพิ่ม `"on_sale": true` ให้สินค้าทุกชิ้นที่ราคาต่ำกว่า 1000 บาท โดยใช้ `||`:

```sql
UPDATE products
SET attributes = attributes || '{"on_sale": true}'::jsonb
WHERE unit_price < 1000
RETURNING product_id, product_name, unit_price, attributes ->> 'on_sale' AS on_sale;
```

```
 product_id |      product_name      | unit_price | on_sale
------------+--------------------------+------------+----------
          6 | Men's Cotton T-Shirt    |     350.00 | true
          7 | Men's Slim Fit Jeans    |     890.00 | true
          9 | Women's Yoga Pants      |     690.00 | true
         10 | Clean Code              |     890.00 | true
         11 | Atomic Habits           |     450.00 | true
         12 | หนังสือ Python เบื้องต้น |     350.00 | true
         15 | Yoga Mat Premium        |     590.00 | true
         17 | Facial Serum Vitamin C  |     690.00 | true
         18 | Moisturizing Cream      |     590.00 | true
         20 | Wireless Mouse          |     590.00 | true
(10 rows)
```

### โจทย์ 4: สร้างสรุป order พร้อม items แบบ JSON เพื่อส่งออกเป็น API response (ทุกออเดอร์)

ผสาน `jsonb_build_object` + `jsonb_agg` + subquery correlated แบบ Step 507 แต่ทำกับทุกออเดอร์พร้อมกัน:

```sql
SELECT jsonb_agg(order_summary ORDER BY (order_summary ->> 'order_id')::int) AS all_orders
FROM (
    SELECT jsonb_build_object(
        'order_id', o.order_id,
        'customer', cu.first_name || ' ' || cu.last_name,
        'status', o.status,
        'total', (
            SELECT SUM(oi.quantity * oi.unit_price)
            FROM order_items oi WHERE oi.order_id = o.order_id
        ),
        'metadata', jsonb_strip_nulls(o.metadata)
    ) AS order_summary
    FROM orders o
    JOIN customers cu ON cu.customer_id = o.customer_id
    WHERE o.order_id <= 3
) sub;
```

```
                                                                     all_orders
------------------------------------------------------------------------------------------------------------------------------------------------------------
 [{"order_id": 1, "customer": "Somchai Jaidee", "status": "paid", "total": 43490.00, "metadata": {"coupon_code": "SALE10", "referrer": "facebook", "gift_wrap": false}},
  {"order_id": 2, "customer": "Suda Meesuk", "status": "completed", "total": 2230.00, "metadata": {"referrer": "google", "gift_wrap": true, "note": "deliver to office"}},
  {"order_id": 3, "customer": "John Smith", "status": "completed", "total": 12900.00, "metadata": {"referrer": "instagram", "gift_wrap": false}}]
(1 row)
```

### โจทย์ 5: ล้างค่าและ normalize attributes ทุกสินค้าให้อยู่รูปแบบเดียวกัน

ลบ field ที่ไม่ควรอยู่ในหน้า listing สาธารณะ (เช่น field ภายใน) แล้วเพิ่ม timestamp การอัปเดตล่าสุด:

```sql
SELECT
    product_id,
    product_name,
    (attributes - 'tags') || jsonb_build_object('last_synced', now()::date) AS public_attributes
FROM products
WHERE product_id = 1;
```

```
 product_id |     product_name     |                                                       public_attributes
------------+------------------------+-----------------------------------------------------------------------------------------------------------------------------
          1 | iPhone 15 Pro Max     | {"color": "titanium blue", "storage_gb": 256, "weight_kg": 0.225, "specs": {...}, "warranty_months": 12, "eco_friendly": true, "last_synced": "2026-09-25"}
(1 row)
```

โจทย์นี้แสดงให้เห็นว่าเราสามารถ **chain operator หลายตัวต่อกัน** (`-` แล้วตามด้วย `||`) ในคำสั่งเดียวได้อย่างอ่านง่าย — นี่คือจุดแข็งของการออกแบบ operator แบบ SQL-native ของ JSONB ใน PostgreSQL

---

## สรุปท้ายบท

### ตารางเปรียบเทียบ JSON vs JSONB

| คุณสมบัติ | `json` | `jsonb` |
|---|---|---|
| วิธีจัดเก็บ | text ต้นฉบับ (exact copy) | decomposed binary format |
| ความเร็วตอน insert | เร็วกว่า | ช้ากว่าเล็กน้อย (ต้อง parse) |
| ความเร็วตอน query/process | ช้ากว่า (parse ทุกครั้ง) | เร็วกว่ามาก (parse ไว้แล้ว) |
| เก็บ whitespace/ลำดับ key เดิม | เก็บ | ไม่เก็บ (normalize) |
| key ซ้ำ | เก็บทุกตัว | เก็บเฉพาะตัวสุดท้าย |
| รองรับ GIN index / containment (`@>`, `?`) | ไม่รองรับ | รองรับ |
| รองรับ `-`, `#-` (ลบ key) | ไม่รองรับ | รองรับ |
| แนะนำใช้เมื่อ | เก็บ log/audit แบบ read-only, ต้องการ text เป๊ะ | การใช้งานทั่วไป (ค่าเริ่มต้นที่แนะนำ) |

### ตาราง operator และฟังก์ชันทั้งหมดในบทนี้

| Operator/ฟังก์ชัน | ความหมาย | Output |
|---|---|---|
| `->` | ดึงค่าตาม key/index (1 ชั้น) | `jsonb` |
| `->>` | ดึงค่าตาม key/index (1 ชั้น) | `text` |
| `#>` | ดึงค่าตาม path (หลายชั้น) | `jsonb` |
| `#>>` | ดึงค่าตาม path (หลายชั้น) | `text` |
| `\|\|` | รวม 2 JSONB object (shallow merge) | `jsonb` |
| `-` (jsonb - text) | ลบ key ระดับบนสุด | `jsonb` |
| `-` (jsonb - text[]) | ลบหลาย key ระดับบนสุด | `jsonb` |
| `-` (jsonb - int) | ลบ array element ตาม index | `jsonb` |
| `#-` | ลบ field ที่ path ลึกกว่า 1 ชั้น | `jsonb` |
| `jsonb_set(target, path, value [, create_missing])` | แก้ไขค่าบางส่วน | `jsonb` |
| `row_to_json(record)` | แปลง 1 แถวเป็น JSON | `json` |
| `to_jsonb(any)` | แปลงค่าใดๆ เป็น jsonb | `jsonb` |
| `json_build_object(...)` / `jsonb_build_object(...)` | สร้าง object กำหนดเอง | `json`/`jsonb` |
| `json_agg(expr)` / `jsonb_agg(expr)` | รวมหลายแถวเป็น array | `json`/`jsonb` |
| `jsonb_object_keys(jsonb)` | ดึง key ทั้งหมดเป็นแถว | table (text) |
| `jsonb_each(jsonb)` | แตก object เป็น key/value | table (text, jsonb) |
| `jsonb_each_text(jsonb)` | แตก object เป็น key/value (text) | table (text, text) |
| `jsonb_array_elements(jsonb)` | แตก array เป็นแถว | table (jsonb) |
| `jsonb_array_elements_text(jsonb)` | แตก array เป็นแถว (text) | table (text) |
| `jsonb_strip_nulls(jsonb)` | ลบทุก key ที่ค่าเป็น null (recursive) | `jsonb` |
| `jsonb_array_length(jsonb)` | นับจำนวน element ใน array | `int` |

### สิ่งที่ควรจำ

1. **ใช้ `JSONB` เป็นค่าเริ่มต้นเสมอ** เว้นแต่มีเหตุผลเฉพาะที่ต้องใช้ `JSON`
2. **`->>` และ `#>>` ปลอดภัยกว่า** เวลาไม่แน่ใจว่า path มีอยู่จริงหรือไม่ เพราะคืน `NULL` แทนที่จะ error
3. **`jsonb_set` แก้เฉพาะจุด** โดยไม่กระทบ field อื่น แต่ **ไม่สร้าง intermediate path ที่ยังไม่มีอยู่ให้อัตโนมัติ**
4. **`||` merge แบบ shallow เท่านั้น** — key ซ้ำกันที่เป็น nested object จะถูกทับทั้งก้อน ไม่ deep-merge
5. อย่าลืม **cast กลับเป็นชนิดที่ถูกต้อง** (`::numeric`, `::boolean`, `::int`) ทุกครั้งที่ดึงค่าด้วย `->>` หรือ `#>>` แล้วจะนำไปคำนวณหรือเปรียบเทียบเชิงตัวเลข/บูลีน
6. บทนี้ยังไม่ได้พูดถึง **index บน JSONB** (GIN index), **containment operators** (`@>`, `<@`, `?`, `?|`, `?&`), และ **jsonpath** (`@?`, `jsonb_path_query`) ซึ่งเป็นหัวข้อสำคัญสำหรับ production ที่มีข้อมูลขนาดใหญ่ — ไปต่อกันได้ใน Part 052

---

## แบบฝึกหัดท้ายบท

### แบบฝึกหัดที่ 1
เขียน query แสดง `product_name` และค่า `material` จาก `attributes` ของสินค้าทุกชิ้นที่มี key `material` (ใช้ operator ธรรมดา ไม่ต้องใช้ containment operator)

<details>
<summary>เฉลย</summary>

```sql
SELECT product_name, attributes ->> 'material' AS material
FROM products
WHERE attributes ->> 'material' IS NOT NULL
ORDER BY product_id;
```

ผลลัพธ์ตัวอย่าง:

```
      product_name      |   material
--------------------------+---------------
 Men's Cotton T-Shirt    | 100% cotton
 Men's Slim Fit Jeans    | denim
 Women's Summer Dress    | polyester
 Women's Yoga Pants      | spandex blend
 Yoga Mat Premium        | TPE
(5 rows)
```

</details>

### แบบฝึกหัดที่ 2
เขียน query ดึงค่า `screen_inch` จาก `attributes.specs.screen_inch` ของสินค้าทุกชิ้น โดยใช้ operator `#>>` (ไม่ใช่ `->`) และแสดงเฉพาะสินค้าที่มีค่านี้

<details>
<summary>เฉลย</summary>

```sql
SELECT product_name, (attributes #>> '{specs,screen_inch}')::numeric AS screen_inch
FROM products
WHERE attributes #>> '{specs,screen_inch}' IS NOT NULL
ORDER BY screen_inch DESC;
```

```
      product_name       | screen_inch
--------------------------+--------------
 Samsung Galaxy S24 Ultra |         6.8
 iPhone 15 Pro Max        |         6.7
 iPad Air                 |        10.9
(3 rows)
```

*(หมายเหตุ: ผลลัพธ์เรียงตามตัวเลขจริง ไม่ใช่ตามลำดับที่แสดงในโจทย์ตัวอย่างข้างบน — ตรวจสอบว่า `ORDER BY screen_inch DESC` ให้ iPad Air ซึ่งมีจอใหญ่ที่สุดอยู่บนสุด)*

</details>

### แบบฝึกหัดที่ 3
ใช้ `jsonb_set` เพิ่มค่า `"currency": "THB"` ให้กับ `attributes` ของสินค้าทุกชิ้นที่ยังไม่มี key นี้ (คำใบ้: ใช้ `UPDATE ... SET attributes = jsonb_set(...)`)

<details>
<summary>เฉลย</summary>

```sql
UPDATE products
SET attributes = jsonb_set(attributes, '{currency}', '"THB"')
WHERE NOT (attributes ? 'currency');
```

ตรวจสอบผล:

```sql
SELECT product_name, attributes ->> 'currency' AS currency
FROM products
LIMIT 3;
```

```
     product_name     | currency
-----------------------+-----------
 iPhone 15 Pro Max     | THB
 Samsung Galaxy S24 Ultra | THB
 MacBook Pro 14" M3    | THB
(3 rows)
```

*(operator `?` ที่ใช้ในเฉลยนี้เป็น containment operator ซึ่งจะอธิบายละเอียดใน Part 052 — ในที่นี้หมายถึง "มี key นี้อยู่หรือไม่")*

</details>

### แบบฝึกหัดที่ 4
เขียน query แสดง `order_id`, `status`, และ `utm.source` (ถ้ามี) ของทุกคำสั่งซื้อ โดยแสดง `'direct'` แทนถ้าไม่มี `utm.source`

<details>
<summary>เฉลย</summary>

```sql
SELECT
    order_id,
    status,
    COALESCE(metadata #>> '{utm,source}', 'direct') AS utm_source
FROM orders
ORDER BY order_id;
```

```
 order_id |  status   | utm_source
----------+-----------+-------------
        1 | paid      | direct
        2 | completed | direct
        ...
        6 | completed | newsletter
        ...
       17 | paid      | newsletter
       18 | shipped   | direct
(18 rows)
```

</details>

### แบบฝึกหัดที่ 5
ใช้ `to_jsonb` และ `jsonb_agg` เขียน query สร้าง JSON array ของลูกค้าทุกคนจากประเทศไทย (`country = 'Thailand'`) โดยแต่ละ element มี `customer_id`, `full_name` (รวมชื่อ-นามสกุล), และ `signup_date`

<details>
<summary>เฉลย</summary>

```sql
SELECT jsonb_agg(
    jsonb_build_object(
        'customer_id', customer_id,
        'full_name', first_name || ' ' || last_name,
        'signup_date', signup_date
    )
) AS thai_customers
FROM customers
WHERE country = 'Thailand';
```

```
                                                         thai_customers
-------------------------------------------------------------------------------------------------------------------------------
 [{"customer_id": 1, "full_name": "Somchai Jaidee", "signup_date": "2023-01-15"}, {"customer_id": 2, "full_name": "Suda Meesuk", ...}, ...]
(1 row)
```

</details>

### แบบฝึกหัดที่ 6
เขียน query แตก `attributes.tags` ของสินค้าทุกชิ้นที่มี tags ออกมาเป็นแถว พร้อมชื่อสินค้า แล้วนับว่าสินค้ากี่ชิ้นที่มี tag `"bestseller"`

<details>
<summary>เฉลย</summary>

```sql
SELECT COUNT(DISTINCT p.product_id) AS bestseller_count
FROM products p
CROSS JOIN LATERAL jsonb_array_elements_text(p.attributes -> 'tags') AS t(tag)
WHERE t.tag = 'bestseller';
```

```
 bestseller_count
-------------------
                 3
(1 row)
```

</details>

### แบบฝึกหัดที่ 7
ใช้ operator `-` ลบ key `note` ออกจาก `metadata` ของทุกคำสั่งซื้อที่มี key นี้ (คำใบ้: ใช้ `-` แบบ single key)

<details>
<summary>เฉลย</summary>

```sql
UPDATE orders
SET metadata = metadata - 'note'
WHERE metadata ? 'note'
RETURNING order_id, metadata;
```

```
 order_id |                              metadata
----------+----------------------------------------------------------------------
        2 | {"coupon_code": null, "referrer": "google", "gift_wrap": true}
       10 | {"referrer": "google", "gift_wrap": true}
(2 rows)
```

</details>

### แบบฝึกหัดที่ 8
ใช้ `jsonb_strip_nulls` ร่วมกับ `||` เขียน query ที่ล้าง null ออกจาก `metadata` แล้วเพิ่ม field `"processed_at": <วันที่ปัจจุบัน>` เข้าไปในคราวเดียว (สำหรับ order_id = 2)

<details>
<summary>เฉลย</summary>

```sql
SELECT
    order_id,
    jsonb_strip_nulls(metadata) || jsonb_build_object('processed_at', now()::date) AS result
FROM orders
WHERE order_id = 2;
```

```
 order_id |                                          result
----------+---------------------------------------------------------------------------------------
        2 | {"referrer": "google", "gift_wrap": true, "processed_at": "2026-09-25"}
(1 row)
```

</details>

### แบบฝึกหัดที่ 9
เขียน query หาสินค้าทั้งหมดที่มี `weight_kg` มากกว่า 1 กิโลกรัม พร้อมแสดงชื่อหมวดหมู่ (ใช้ `#>>` และ cast เป็น `numeric`)

<details>
<summary>เฉลย</summary>

```sql
SELECT
    p.product_name,
    c.category_name,
    (p.attributes #>> '{weight_kg}')::numeric AS weight_kg
FROM products p
JOIN categories c ON c.category_id = p.category_id
WHERE (p.attributes #>> '{weight_kg}')::numeric > 1
ORDER BY weight_kg DESC;
```

```
     product_name      | category_name  | weight_kg
------------------------+-----------------+-----------
 Stand Mixer 5L         | Home & Kitchen  |      6.50
 Air Fryer 4.5L         | Home & Kitchen  |      4.20
 MacBook Pro 14" M3     | Computers       |      1.55
 Dell XPS 13            | Computers       |      1.24
 Yoga Mat Premium       | Sports & Outdoor|      1.20
(5 rows)
```

</details>

### แบบฝึกหัดที่ 10
สร้าง query เดียวที่คืน JSON object ของสินค้าหนึ่งชิ้น (product_id = 3) ที่มีโครงสร้างดังนี้: `{"id":..., "name":..., "price":..., "category":..., "supplier":..., "specs":...}` โดยดึง `category` และ `supplier` จากการ `JOIN` ตารางที่เกี่ยวข้อง และ `specs` มาจาก `attributes.specs`

<details>
<summary>เฉลย</summary>

```sql
SELECT jsonb_build_object(
    'id', p.product_id,
    'name', p.product_name,
    'price', p.unit_price,
    'category', c.category_name,
    'supplier', s.supplier_name,
    'specs', p.attributes -> 'specs'
) AS product_detail
FROM products p
JOIN categories c ON c.category_id = p.category_id
JOIN suppliers s ON s.supplier_id = p.supplier_id
WHERE p.product_id = 3;
```

```
                                                                  product_detail
--------------------------------------------------------------------------------------------------------------------------------------------------
 {"id": 3, "name": "MacBook Pro 14\" M3", "price": 79900.00, "category": "Computers", "supplier": "TechSource Co., Ltd.", "specs": {"battery": "70Wh", "cpu": "Apple M3 Pro"}}
(1 row)
```

</details>

---

## บทถัดไป

บทนี้ได้ปูพื้นฐานการทำงานกับ JSON/JSONB ใน PostgreSQL ตั้งแต่การสร้าง อ่าน แก้ไข ไปจนถึงการแปลงข้อมูลไปมาระหว่างรูปแบบ relational และ JSON แล้ว ในบทถัดไปเราจะเจาะลึกเทคนิคขั้นสูงที่จำเป็นสำหรับระบบ production จริง ได้แก่ **containment operators** (`@>`, `<@`, `?`, `?|`, `?&`), **GIN index สำหรับ JSONB**, **jsonpath expressions** (`jsonb_path_query`, `@?`, `@@`), และแนวทางออกแบบสคีมาที่ผสมผสานระหว่าง relational column กับ JSONB column อย่างเหมาะสม

**บทถัดไป**: [Part 052: JSONB ขั้นสูง — Containment, GIN Index และ jsonpath](./part-052-jsonb-advanced.md)
