# PL/pgSQL ขั้นสูง: Loop, Exception Handling, Dynamic SQL

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับสูง | Part 047

---

## เป้าหมายการเรียนรู้

หลังจากเรียนจบบทนี้ ผู้เรียนจะสามารถ:

- ใช้โครงสร้างการวนซ้ำ (loop) ทั้ง 4 แบบใน PL/pgSQL ได้แก่ `LOOP`, `WHILE`, `FOR`, และ `FOREACH` ได้อย่างถูกต้องและเหมาะสมกับสถานการณ์
- ควบคุมการไหลของ loop ด้วย `CONTINUE` และ `EXIT` รวมถึงการใช้ label กับ nested loop ที่ซับซ้อน
- ออกแบบระบบดักจับข้อผิดพลาด (Exception Handling) ด้วย `BEGIN...EXCEPTION...END` เพื่อทำให้ฟังก์ชันทนทานต่อความผิดพลาด (robust) ไม่ล้มทั้งระบบเมื่อเจอ error
- รู้จัก exception ที่พบบ่อยที่สุดในงานจริง เช่น `unique_violation`, `foreign_key_violation`, `division_by_zero` และเขียนโค้ดรับมือแต่ละกรณีอย่างเหมาะสม
- ใช้ `RAISE` เพื่อสร้าง notice, warning, exception ของตัวเอง พร้อมกำหนดข้อความและ error code
- เขียน Dynamic SQL ด้วย `EXECUTE` เพื่อประกอบคำสั่ง SQL ขึ้นระหว่าง runtime พร้อมเข้าใจความเสี่ยงของ SQL Injection และวิธีป้องกันด้วย `format()`, `quote_ident()`, `quote_literal()`
- ใช้ Cursor เพื่ออ่านข้อมูลจำนวนมากทีละแถวโดยไม่ทำให้หน่วยความจำล้น
- ประยุกต์ทุกเทคนิคเข้าด้วยกันเพื่อเขียนฟังก์ชันประมวลผลข้อมูลแบบ batch ที่ทนทาน มี log error และปลอดภัยจาก SQL Injection

---

## เตรียมข้อมูล

บทนี้ใช้ฐานข้อมูลอีคอมเมิร์ซชุดเดิมที่ใช้ต่อเนื่องมาตลอดหลักสูตร ประกอบด้วย 6 ตาราง: `categories`, `suppliers`, `products`, `customers`, `orders`, `order_items`

```sql
-- ล้างตารางเดิม (ถ้ามี) เพื่อเริ่มต้นใหม่ให้สะอาด
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
    is_active       BOOLEAN NOT NULL DEFAULT true
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
    ship_country  VARCHAR(60)
);

CREATE TABLE order_items (
    order_item_id  SERIAL PRIMARY KEY,
    order_id       INTEGER REFERENCES orders(order_id),
    product_id     INTEGER REFERENCES products(product_id),
    quantity       INTEGER NOT NULL CHECK (quantity > 0),
    unit_price     NUMERIC(10,2) NOT NULL
);
```

### ข้อมูลตัวอย่าง (Seed Data)

```sql
-- categories: หมวดหมู่สินค้า (มีหมวดย่อยที่อ้างอิงหมวดแม่)
INSERT INTO categories (category_name, parent_category_id) VALUES
('Electronics', NULL),
('Computers', 1),
('Mobile Phones', 1),
('Home Appliances', NULL),
('Kitchen', 4),
('Fashion', NULL),
('Men Clothing', 6),
('Women Clothing', 6),
('Books', NULL),
('Sports & Outdoor', NULL),
('Toys', NULL),
('Beauty & Personal Care', NULL);

-- suppliers: ซัพพลายเออร์
INSERT INTO suppliers (supplier_name, country) VALUES
('Bangkok Tech Distribution', 'Thailand'),
('Shenzhen Global Trading', 'China'),
('Osaka Electronics Co.', 'Japan'),
('Seoul Digital Supply', 'South Korea'),
('Hanoi Textile House', 'Vietnam'),
('Singapore Logistics Hub', 'Singapore'),
('Jakarta Home Goods', 'Indonesia'),
('Chiang Mai Craft Supply', 'Thailand'),
('Kuala Lumpur Gadget Co.', 'Malaysia'),
('Manila Fashion Group', 'Philippines');

-- products: สินค้า
INSERT INTO products (product_name, category_id, supplier_id, unit_price, stock_quantity, is_active) VALUES
('Wireless Mouse M1', 2, 1, 290.00, 150, true),
('Mechanical Keyboard K87', 2, 2, 1890.00, 60, true),
('Laptop Stand Aluminum', 2, 1, 590.00, 80, true),
('Smartphone Zeta 12', 3, 4, 15900.00, 40, true),
('Smartphone Zeta 12 Lite', 3, 4, 9900.00, 55, true),
('Bluetooth Earbuds Pro', 1, 3, 1290.00, 200, true),
('4K Monitor 27-inch', 2, 3, 7900.00, 25, true),
('Rice Cooker Deluxe', 5, 7, 1450.00, 70, true),
('Air Fryer XL', 5, 7, 2390.00, 45, true),
('Blender Pro 900W', 5, 7, 1190.00, 90, true),
('Men Cotton T-Shirt', 7, 5, 250.00, 300, true),
('Men Slim Jeans', 7, 5, 690.00, 120, true),
('Women Summer Dress', 8, 10, 590.00, 100, true),
('Women Handbag Classic', 8, 10, 1290.00, 65, true),
('PostgreSQL Mastery Book', 9, 6, 890.00, 40, true),
('Running Shoes AirFlex', 10, 9, 2190.00, 75, false),
('Yoga Mat Premium', 10, 8, 490.00, 130, true),
('Building Blocks Set', 11, 2, 690.00, 85, true),
('Remote Control Car', 11, 2, 990.00, 50, true),
('Facial Serum Vitamin C', 12, 9, 590.00, 160, true);

-- customers: ลูกค้า
INSERT INTO customers (first_name, last_name, email, country, signup_date) VALUES
('Somchai', 'Jaidee', 'somchai.j@example.com', 'Thailand', '2023-01-15'),
('Suda', 'Rungrueang', 'suda.r@example.com', 'Thailand', '2023-02-20'),
('Anan', 'Wongsawat', 'anan.w@example.com', 'Thailand', '2023-03-05'),
('Malee', 'Sukjai', 'malee.s@example.com', 'Thailand', '2023-04-10'),
('Wirat', 'Chaiyaporn', 'wirat.c@example.com', 'Thailand', '2023-05-18'),
('Nattaya', 'Boonmee', 'nattaya.b@example.com', 'Thailand', '2023-06-22'),
('John', 'Smith', 'john.smith@example.com', 'USA', '2023-02-01'),
('Emily', 'Johnson', 'emily.j@example.com', 'USA', '2023-03-14'),
('Akira', 'Tanaka', 'akira.t@example.com', 'Japan', '2023-04-01'),
('Yuki', 'Sato', 'yuki.s@example.com', 'Japan', '2023-05-09'),
('Minjun', 'Kim', 'minjun.k@example.com', 'South Korea', '2023-06-11'),
('Lin', 'Wang', 'lin.wang@example.com', 'China', '2023-07-02'),
('Siti', 'Aminah', 'siti.a@example.com', 'Indonesia', '2023-07-19'),
('Nguyen', 'Van A', 'nguyen.va@example.com', 'Vietnam', '2023-08-03'),
('Pichit', 'Meesuk', 'pichit.m@example.com', 'Thailand', '2023-08-20');

-- orders: คำสั่งซื้อ
INSERT INTO orders (customer_id, order_date, status, ship_country) VALUES
(1, '2024-01-05 10:15:00+07', 'completed', 'Thailand'),
(2, '2024-01-08 14:30:00+07', 'completed', 'Thailand'),
(3, '2024-01-10 09:00:00+07', 'completed', 'Thailand'),
(1, '2024-01-20 16:45:00+07', 'completed', 'Thailand'),
(4, '2024-02-02 11:20:00+07', 'cancelled', 'Thailand'),
(5, '2024-02-10 13:00:00+07', 'completed', 'Thailand'),
(7, '2024-02-14 08:30:00+07', 'completed', 'USA'),
(8, '2024-02-15 19:10:00+07', 'shipped', 'USA'),
(9, '2024-02-18 07:45:00+07', 'completed', 'Japan'),
(6, '2024-03-01 12:00:00+07', 'pending', 'Thailand'),
(10, '2024-03-03 15:25:00+07', 'completed', 'Japan'),
(11, '2024-03-05 10:10:00+07', 'completed', 'South Korea'),
(2, '2024-03-12 17:40:00+07', 'shipped', 'Thailand'),
(12, '2024-03-15 09:55:00+07', 'completed', 'China'),
(3, '2024-03-20 14:05:00+07', 'pending', 'Thailand'),
(13, '2024-03-22 11:30:00+07', 'completed', 'Indonesia'),
(14, '2024-03-25 16:00:00+07', 'cancelled', 'Vietnam'),
(15, '2024-04-01 08:20:00+07', 'completed', 'Thailand');

-- order_items: รายการสินค้าในคำสั่งซื้อ
INSERT INTO order_items (order_id, product_id, quantity, unit_price) VALUES
(1, 1, 2, 290.00),
(1, 6, 1, 1290.00),
(2, 4, 1, 15900.00),
(3, 11, 3, 250.00),
(3, 12, 1, 690.00),
(4, 2, 1, 1890.00),
(4, 3, 1, 590.00),
(5, 8, 1, 1450.00),
(6, 9, 1, 2390.00),
(6, 10, 1, 1190.00),
(7, 15, 2, 890.00),
(8, 5, 1, 9900.00),
(9, 7, 1, 7900.00),
(10, 13, 2, 590.00),
(11, 14, 1, 1290.00),
(12, 6, 3, 1290.00),
(13, 17, 2, 490.00),
(14, 18, 1, 690.00),
(14, 19, 1, 990.00),
(15, 20, 2, 590.00),
(16, 1, 5, 290.00),
(17, 2, 1, 1890.00),
(18, 11, 4, 250.00);
```

ตรวจสอบว่าข้อมูลถูกต้อง:

```sql
SELECT 'categories' AS tbl, count(*) FROM categories
UNION ALL SELECT 'suppliers', count(*) FROM suppliers
UNION ALL SELECT 'products', count(*) FROM products
UNION ALL SELECT 'customers', count(*) FROM customers
UNION ALL SELECT 'orders', count(*) FROM orders
UNION ALL SELECT 'order_items', count(*) FROM order_items;
```

```
    tbl      | count
-------------+-------
 categories  |    12
 suppliers   |    10
 products    |    20
 customers   |    15
 orders      |    18
 order_items |    23
```

---

## Step 461: LOOP พื้นฐาน — LOOP...EXIT WHEN

`LOOP` เป็นโครงสร้างการวนซ้ำที่ "ดิบ" ที่สุดใน PL/pgSQL คือวนไปเรื่อย ๆ แบบไม่มีเงื่อนไขในตัวเอง ผู้เขียนโค้ดต้องกำหนดเงื่อนไขหยุดเอง ผ่านคำสั่ง `EXIT` หรือ `EXIT WHEN <condition>` มิฉะนั้นจะกลายเป็น infinite loop ที่รันไม่มีวันจบ

รูปแบบพื้นฐาน:

```sql
<<label>>  -- ตั้ง label ได้ (optional)
LOOP
    -- คำสั่งที่จะทำซ้ำ
    EXIT WHEN <เงื่อนไข>;   -- ออกจาก loop เมื่อเงื่อนไขเป็นจริง
END LOOP;
```

### ตัวอย่าง 1: นับถอยหลังสต็อกสินค้าแบบง่าย

สร้างฟังก์ชันที่จำลองการตัดสต็อกทีละ 10 ชิ้น จนกว่าสต็อกจะเหลือน้อยกว่า 10

```sql
CREATE OR REPLACE FUNCTION demo_loop_basic(p_product_id INTEGER)
RETURNS VOID AS $$
DECLARE
    v_stock INTEGER;
    v_round INTEGER := 0;
BEGIN
    SELECT stock_quantity INTO v_stock
    FROM products
    WHERE product_id = p_product_id;

    IF v_stock IS NULL THEN
        RAISE NOTICE 'ไม่พบสินค้า product_id = %', p_product_id;
        RETURN;
    END IF;

    LOOP
        v_round := v_round + 1;
        EXIT WHEN v_stock < 10;

        v_stock := v_stock - 10;
        RAISE NOTICE 'รอบที่ %: ตัดสต็อก 10 ชิ้น เหลือ % ชิ้น', v_round, v_stock;
    END LOOP;

    RAISE NOTICE 'สิ้นสุดการวน: สต็อกคงเหลือ % ชิ้น (เหลือน้อยกว่า 10 แล้วหยุด)', v_stock;
END;
$$ LANGUAGE plpgsql;

SELECT demo_loop_basic(1);  -- Wireless Mouse M1 มีสต็อก 150
```

ผลลัพธ์ (`RAISE NOTICE`):

```
NOTICE:  รอบที่ 1: ตัดสต็อก 10 ชิ้น เหลือ 140 ชิ้น
NOTICE:  รอบที่ 2: ตัดสต็อก 10 ชิ้น เหลือ 130 ชิ้น
...
NOTICE:  รอบที่ 14: ตัดสต็อก 10 ชิ้น เหลือ 0 ชิ้น
NOTICE:  สิ้นสุดการวน: สต็อกคงเหลือ 0 ชิ้น (เหลือน้อยกว่า 10 แล้วหยุด)
```

### ตัวอย่าง 2: การใช้ EXIT แบบมีเงื่อนไขซ้อน IF

บางครั้งเราต้องการเงื่อนไขที่ซับซ้อนกว่า `EXIT WHEN` เพียงบรรทัดเดียว สามารถใช้ `IF ... THEN EXIT; END IF;` แทนได้

```sql
CREATE OR REPLACE FUNCTION demo_loop_with_if(p_start_qty INTEGER)
RETURNS INTEGER AS $$
DECLARE
    v_qty INTEGER := p_start_qty;
    v_total_discounted NUMERIC := 0;
BEGIN
    LOOP
        -- เงื่อนไขหยุดที่ซับซ้อน: หยุดเมื่อจำนวนน้อยกว่าหรือเท่ากับ 0 หรือ discount รวมเกิน 1000
        IF v_qty <= 0 OR v_total_discounted > 1000 THEN
            EXIT;
        END IF;

        v_total_discounted := v_total_discounted + (v_qty * 0.5);
        v_qty := v_qty - 20;
    END LOOP;

    RAISE NOTICE 'ยอด discount สะสม = %, จำนวนคงเหลือ = %', v_total_discounted, v_qty;
    RETURN v_qty;
END;
$$ LANGUAGE plpgsql;

SELECT demo_loop_with_if(300);
```

```
NOTICE:  ยอด discount สะสม = 1055.0, จำนวนคงเหลือ = 40
 demo_loop_with_if
--------------------
                 40
```

> **ข้อควรระวัง:** `LOOP` ธรรมดาไม่มีเงื่อนไขหยุดในตัว หากลืมเขียน `EXIT` หรือ `EXIT WHEN` จะทำให้ฟังก์ชันรันค้างไม่รู้จบ (infinite loop) ซึ่งจะทำให้ connection ค้างและกิน CPU เต็ม 100% ควรทดสอบด้วยจำนวนรอบจำกัดก่อนนำไปใช้งานจริงเสมอ

---

## Step 462: WHILE loop และ FOR loop

### 462.1 WHILE loop

`WHILE` จะตรวจสอบเงื่อนไข **ก่อน** ทำงานในแต่ละรอบ ถ้าเงื่อนไขเป็นเท็จตั้งแต่ต้นจะไม่เข้า loop เลยแม้แต่ครั้งเดียว ต่างจาก `LOOP` ที่ต้องเข้าไปทำงานอย่างน้อย 1 รอบก่อนจะเช็คเงื่อนไข (ผ่าน `EXIT WHEN`)

```sql
WHILE <เงื่อนไข> LOOP
    -- คำสั่งที่จะทำซ้ำ
END LOOP;
```

ตัวอย่าง: คำนวณส่วนลดแบบขั้นบันได (tiered discount) จนกว่ายอดคงเหลือจะหมด

```sql
CREATE OR REPLACE FUNCTION demo_while_tiered_discount(p_amount NUMERIC)
RETURNS NUMERIC AS $$
DECLARE
    v_remaining NUMERIC := p_amount;
    v_tier      INTEGER := 1;
    v_rate      NUMERIC;
    v_total_discount NUMERIC := 0;
    v_chunk     NUMERIC;
BEGIN
    WHILE v_remaining > 0 LOOP
        -- อัตราส่วนลดลดหลั่นตามช่วง (tier ยิ่งสูง ส่วนลดยิ่งน้อยลง)
        v_rate := GREATEST(0.10 - (v_tier - 1) * 0.02, 0.02);
        v_chunk := LEAST(v_remaining, 5000);

        v_total_discount := v_total_discount + (v_chunk * v_rate);
        RAISE NOTICE 'Tier %: ช่วงเงิน % บาท อัตรา %%%, ส่วนลด % บาท',
            v_tier, v_chunk, v_rate * 100, round(v_chunk * v_rate, 2);

        v_remaining := v_remaining - v_chunk;
        v_tier := v_tier + 1;
    END LOOP;

    RETURN round(v_total_discount, 2);
END;
$$ LANGUAGE plpgsql;

SELECT demo_while_tiered_discount(18000) AS total_discount;
```

```
NOTICE:  Tier 1: ช่วงเงิน 5000 บาท อัตรา 10.00%, ส่วนลด 500.00 บาท
NOTICE:  Tier 2: ช่วงเงิน 5000 บาท อัตรา 8.00%, ส่วนลด 400.00 บาท
NOTICE:  Tier 3: ช่วงเงิน 5000 บาท อัตรา 6.00%, ส่วนลด 300.00 บาท
NOTICE:  Tier 4: ช่วงเงิน 3000 บาท อัตรา 4.00%, ส่วนลด 120.00 บาท
 total_discount
-----------------
         1320.00
```

### 462.2 FOR loop แบบตัวเลข (numeric FOR)

ใช้เมื่อรู้จำนวนรอบที่แน่นอนล่วงหน้า มีรูปแบบ `FOR i IN low..high [BY step] LOOP`

```sql
CREATE OR REPLACE FUNCTION demo_for_numeric()
RETURNS VOID AS $$
DECLARE
    v_month TEXT;
BEGIN
    FOR i IN 1..12 LOOP
        v_month := to_char(make_date(2024, i, 1), 'Month');
        RAISE NOTICE 'เดือนที่ %: %', i, trim(v_month);
    END LOOP;

    RAISE NOTICE '--- วนถอยหลังทีละ 2 ---';
    FOR i IN REVERSE 10..0 BY 2 LOOP
        RAISE NOTICE 'ตัวนับ: %', i;
    END LOOP;
END;
$$ LANGUAGE plpgsql;

SELECT demo_for_numeric();
```

```
NOTICE:  เดือนที่ 1: January
NOTICE:  เดือนที่ 2: February
...
NOTICE:  เดือนที่ 12: December
NOTICE:  --- วนถอยหลังทีละ 2 ---
NOTICE:  ตัวนับ: 10
NOTICE:  ตัวนับ: 8
NOTICE:  ตัวนับ: 6
NOTICE:  ตัวนับ: 4
NOTICE:  ตัวนับ: 2
NOTICE:  ตัวนับ: 0
```

> ตัวแปรควบคุม loop (เช่น `i`) ถูกประกาศโดยอัตโนมัติ ไม่ต้องประกาศใน `DECLARE` และมี scope อยู่เฉพาะภายใน loop เท่านั้น

### 462.3 FOR loop แบบวนตามผลลัพธ์ query (FOR...IN SELECT)

รูปแบบที่ใช้บ่อยที่สุดใน PL/pgSQL คือการวนอ่านผลลัพธ์จาก query ทีละแถว โดยแต่ละแถวจะถูกเก็บไว้ใน record variable

```sql
CREATE OR REPLACE FUNCTION demo_for_query_loop()
RETURNS VOID AS $$
DECLARE
    v_rec RECORD;
    v_grand_total NUMERIC := 0;
BEGIN
    FOR v_rec IN
        SELECT c.customer_id, c.first_name, c.last_name,
               count(o.order_id) AS order_count,
               coalesce(sum(oi.quantity * oi.unit_price), 0) AS total_spent
        FROM customers c
        JOIN orders o ON o.customer_id = c.customer_id AND o.status = 'completed'
        JOIN order_items oi ON oi.order_id = o.order_id
        GROUP BY c.customer_id, c.first_name, c.last_name
        ORDER BY total_spent DESC
    LOOP
        RAISE NOTICE 'ลูกค้า % % : % คำสั่งซื้อ ยอดรวม % บาท',
            v_rec.first_name, v_rec.last_name, v_rec.order_count, v_rec.total_spent;
        v_grand_total := v_grand_total + v_rec.total_spent;
    END LOOP;

    RAISE NOTICE '=== ยอดรวมทั้งหมด: % บาท ===', v_grand_total;
END;
$$ LANGUAGE plpgsql;

SELECT demo_for_query_loop();
```

```
NOTICE:  ลูกค้า Somchai Jaidee : 2 คำสั่งซื้อ ยอดรวม 2870.00 บาท
NOTICE:  ลูกค้า John Smith : 1 คำสั่งซื้อ ยอดรวม 1780.00 บาท
NOTICE:  ลูกค้า Emily Johnson : 1 คำสั่งซื้อ ยอดรวม 9900.00 บาท
...
NOTICE:  === ยอดรวมทั้งหมด: XXXXX.XX บาท ===
```

> **เคล็ดลับประสิทธิภาพ:** `FOR...IN SELECT` ใช้ cursor ภายในโดยอัตโนมัติ (implicit cursor) ทำให้ PostgreSQL ไม่โหลดผลลัพธ์ทั้งหมดเข้าหน่วยความจำพร้อมกัน เหมาะกับการวนอ่านข้อมูลจำนวนมาก และดีกว่าการ `SELECT ... INTO` array แล้ววนด้วย `FOREACH`

---

## Step 463: FOREACH loop สำหรับวนข้อมูลใน array

`FOREACH` ใช้วนซ้ำสมาชิกใน array โดยเฉพาะ มีรูปแบบ:

```sql
FOREACH <element_var> [SLICE n] IN ARRAY <array_expression> LOOP
    -- คำสั่งที่จะทำซ้ำ
END LOOP;
```

### ตัวอย่าง 1: วนตรวจสอบรายการ product_id จาก array

```sql
CREATE OR REPLACE FUNCTION demo_foreach_products(p_ids INTEGER[])
RETURNS VOID AS $$
DECLARE
    v_id INTEGER;
    v_name TEXT;
    v_price NUMERIC;
BEGIN
    FOREACH v_id IN ARRAY p_ids LOOP
        SELECT product_name, unit_price INTO v_name, v_price
        FROM products
        WHERE product_id = v_id;

        IF NOT FOUND THEN
            RAISE NOTICE 'product_id % : ไม่พบสินค้า', v_id;
        ELSE
            RAISE NOTICE 'product_id % : % (ราคา % บาท)', v_id, v_name, v_price;
        END IF;
    END LOOP;
END;
$$ LANGUAGE plpgsql;

SELECT demo_foreach_products(ARRAY[1, 4, 999, 15]);
```

```
NOTICE:  product_id 1 : Wireless Mouse M1 (ราคา 290.00 บาท)
NOTICE:  product_id 4 : Smartphone Zeta 12 (ราคา 15900.00 บาท)
NOTICE:  product_id 999 : ไม่พบสินค้า
NOTICE:  product_id 15 : PostgreSQL Mastery Book (ราคา 890.00 บาท)
```

### ตัวอย่าง 2: FOREACH กับ array 2 มิติและ SLICE

`SLICE` ใช้เมื่อ array เป็นแบบหลายมิติ และต้องการวนดึงทีละ "แถวย่อย" (sub-array) แทนที่จะดึงทีละสมาชิกเดี่ยว

```sql
CREATE OR REPLACE FUNCTION demo_foreach_2d()
RETURNS VOID AS $$
DECLARE
    v_matrix INTEGER[][] := ARRAY[[1,2,3],[4,5,6],[7,8,9]];
    v_row    INTEGER[];
    v_val    INTEGER;
BEGIN
    -- SLICE 1: วนทีละแถว (sub-array)
    FOREACH v_row SLICE 1 IN ARRAY v_matrix LOOP
        RAISE NOTICE 'แถว: %', v_row;
    END LOOP;

    RAISE NOTICE '--- วนทีละสมาชิกเดี่ยว (ไม่ใช้ SLICE) ---';
    FOREACH v_val IN ARRAY v_matrix LOOP
        RAISE NOTICE 'ค่า: %', v_val;
    END LOOP;
END;
$$ LANGUAGE plpgsql;

SELECT demo_foreach_2d();
```

```
NOTICE:  แถว: {1,2,3}
NOTICE:  แถว: {4,5,6}
NOTICE:  แถว: {7,8,9}
NOTICE:  --- วนทีละสมาชิกเดี่ยว (ไม่ใช้ SLICE) ---
NOTICE:  ค่า: 1
NOTICE:  ค่า: 2
NOTICE:  ค่า: 3
...
NOTICE:  ค่า: 9
```

### ตัวอย่าง 3: ใช้ FOREACH ร่วมกับ array_agg เพื่อ batch process รายชื่อประเทศ

```sql
CREATE OR REPLACE FUNCTION demo_foreach_countries()
RETURNS VOID AS $$
DECLARE
    v_countries TEXT[];
    v_country   TEXT;
    v_customer_count INTEGER;
BEGIN
    SELECT array_agg(DISTINCT country ORDER BY country) INTO v_countries
    FROM customers;

    RAISE NOTICE 'พบทั้งหมด % ประเทศ: %', array_length(v_countries, 1), v_countries;

    FOREACH v_country IN ARRAY v_countries LOOP
        SELECT count(*) INTO v_customer_count
        FROM customers
        WHERE country = v_country;

        RAISE NOTICE '  - %: % คน', v_country, v_customer_count;
    END LOOP;
END;
$$ LANGUAGE plpgsql;

SELECT demo_foreach_countries();
```

```
NOTICE:  พบทั้งหมด 6 ประเทศ: {China,Indonesia,Japan,"South Korea",Thailand,USA,Vietnam}
NOTICE:    - China: 1 คน
NOTICE:    - Indonesia: 1 คน
NOTICE:    - Japan: 2 คน
NOTICE:    - South Korea: 1 คน
NOTICE:    - Thailand: 8 คน
NOTICE:    - USA: 2 คน
NOTICE:    - Vietnam: 1 คน
```

---

## Step 464: CONTINUE และ EXIT พร้อม label สำหรับ nested loop

- `CONTINUE` ใช้ข้ามไปทำรอบถัดไปทันที โดยไม่ทำคำสั่งที่เหลือในรอบปัจจุบัน
- `EXIT` ใช้ออกจาก loop ทั้งหมด
- ทั้งคู่รองรับเงื่อนไข `WHEN` ต่อท้ายได้ เช่น `CONTINUE WHEN ...`, `EXIT WHEN ...`
- เมื่อมี **nested loop** (loop ซ้อน loop) และต้องการควบคุม loop ชั้นนอกจากภายใน loop ชั้นใน จำเป็นต้องใช้ **label** กำกับ loop นั้น ๆ

รูปแบบ label:

```sql
<<outer_loop>>
LOOP
    <<inner_loop>>
    LOOP
        EXIT outer_loop WHEN <เงื่อนไข>;   -- ออกจาก loop ชั้นนอกโดยตรง
        CONTINUE outer_loop WHEN <เงื่อนไข>; -- ข้ามไปรอบถัดไปของ loop ชั้นนอก
    END LOOP inner_loop;
END LOOP outer_loop;
```

### ตัวอย่าง 1: CONTINUE เพื่อข้ามสินค้าที่ไม่ active

```sql
CREATE OR REPLACE FUNCTION demo_continue_skip_inactive()
RETURNS VOID AS $$
DECLARE
    v_rec RECORD;
    v_processed INTEGER := 0;
    v_skipped   INTEGER := 0;
BEGIN
    FOR v_rec IN SELECT product_id, product_name, is_active, stock_quantity FROM products ORDER BY product_id
    LOOP
        -- ข้ามสินค้าที่ปิดการขาย (is_active = false)
        CONTINUE WHEN NOT v_rec.is_active;

        -- ข้ามสินค้าที่สต็อกเป็น 0
        IF v_rec.stock_quantity = 0 THEN
            v_skipped := v_skipped + 1;
            CONTINUE;
        END IF;

        v_processed := v_processed + 1;
        RAISE NOTICE 'ประมวลผล: % (สต็อก %)', v_rec.product_name, v_rec.stock_quantity;
    END LOOP;

    RAISE NOTICE 'สรุป: ประมวลผล % รายการ, ข้าม % รายการ (สต็อกหมด)', v_processed, v_skipped;
END;
$$ LANGUAGE plpgsql;

SELECT demo_continue_skip_inactive();
```

```
NOTICE:  ประมวลผล: Wireless Mouse M1 (สต็อก 150)
NOTICE:  ประมวลผล: Mechanical Keyboard K87 (สต็อก 60)
...
NOTICE:  สรุป: ประมวลผล 19 รายการ, ข้าม 0 รายการ (สต็อกหมด)
```

> สินค้า `Running Shoes AirFlex` มี `is_active = false` จึงถูกข้ามด้วย `CONTINUE WHEN NOT v_rec.is_active` ไม่นับรวมใน `v_processed`

### ตัวอย่าง 2: Nested loop พร้อม label — หาคู่สินค้าที่ราคารวมใกล้เคียงงบประมาณ

สมมติต้องการหาคู่สินค้า 2 ชิ้นที่ราคารวมไม่เกินงบ 2,000 บาท แต่หยุดค้นหาทันทีเมื่อเจอคู่แรกที่ผลรวมมากกว่า 1,900 บาท (ใกล้เคียงงบมากที่สุด) โดยใช้ label ควบคุม loop ทั้งสองชั้น

```sql
CREATE OR REPLACE FUNCTION demo_nested_label_search(p_budget NUMERIC)
RETURNS TABLE(product_a TEXT, product_b TEXT, total_price NUMERIC) AS $$
DECLARE
    v_a RECORD;
    v_b RECORD;
    v_best_a TEXT;
    v_best_b TEXT;
    v_best_total NUMERIC := 0;
BEGIN
    <<outer_scan>>
    FOR v_a IN SELECT product_id, product_name, unit_price FROM products WHERE is_active LOOP
        <<inner_scan>>
        FOR v_b IN SELECT product_id, product_name, unit_price FROM products WHERE is_active LOOP
            -- ข้ามคู่ที่เป็นสินค้าเดียวกัน หรือ id ซ้ำ (ป้องกันนับคู่ซ้ำ)
            CONTINUE inner_scan WHEN v_b.product_id <= v_a.product_id;

            DECLARE
                v_sum NUMERIC := v_a.unit_price + v_b.unit_price;
            BEGIN
                CONTINUE inner_scan WHEN v_sum > p_budget;

                IF v_sum > v_best_total THEN
                    v_best_total := v_sum;
                    v_best_a := v_a.product_name;
                    v_best_b := v_b.product_name;
                END IF;

                -- ถ้าใกล้งบมากพอแล้ว (เกิน 95% ของงบ) ให้หยุดค้นหาทั้งหมดทันที
                EXIT outer_scan WHEN v_sum >= p_budget * 0.95;
            END;
        END LOOP inner_scan;
    END LOOP outer_scan;

    RETURN QUERY SELECT v_best_a, v_best_b, v_best_total;
END;
$$ LANGUAGE plpgsql;

SELECT * FROM demo_nested_label_search(2000);
```

```
 product_a          | product_b               | total_price
---------------------+-------------------------+-------------
 Wireless Mouse M1    | Air Fryer XL            |     2390.00 (ตัวอย่าง - ผลจริงขึ้นกับลำดับข้อมูล)
```

> **ข้อสังเกตสำคัญ:** คำสั่ง `EXIT outer_scan` และ `CONTINUE inner_scan` ทำให้เรา "กระโดด" ควบคุม loop ชั้นนอกจากภายใน loop ชั้นในได้โดยตรง หากไม่ใช้ label การ `EXIT`/`CONTINUE` เฉย ๆ จะมีผลกับ loop ที่อยู่ใกล้ที่สุด (innermost) เท่านั้น

---

## Step 465: Exception Handling — BEGIN...EXCEPTION...END

PL/pgSQL รองรับการดักจับข้อผิดพลาดด้วยบล็อก `EXCEPTION` ภายใน `BEGIN...END` โดยเมื่อเกิด error ในบล็อกใด PostgreSQL จะ **rollback การเปลี่ยนแปลงทั้งหมดภายในบล็อกนั้น** (ไม่ใช่ทั้ง transaction) แล้วเข้าสู่ส่วน `EXCEPTION` ที่ตรงกับประเภท error

รูปแบบ:

```sql
BEGIN
    -- คำสั่งที่อาจเกิด error
EXCEPTION
    WHEN <condition_name_1> THEN
        -- โค้ดรับมือ error ประเภทที่ 1
    WHEN <condition_name_2> OR <condition_name_3> THEN
        -- โค้ดรับมือ error หลายประเภทพร้อมกัน
    WHEN OTHERS THEN
        -- โค้ดรับมือ error ประเภทอื่น ๆ ที่ไม่ได้ระบุไว้ข้างต้น
END;
```

### ตัวอย่าง 1: จับ error หารด้วยศูนย์

```sql
CREATE OR REPLACE FUNCTION demo_safe_divide(p_numerator NUMERIC, p_denominator NUMERIC)
RETURNS NUMERIC AS $$
DECLARE
    v_result NUMERIC;
BEGIN
    BEGIN
        v_result := p_numerator / p_denominator;
    EXCEPTION
        WHEN division_by_zero THEN
            RAISE NOTICE 'ตรวจพบการหารด้วยศูนย์ (% / %) — คืนค่า NULL แทน', p_numerator, p_denominator;
            v_result := NULL;
    END;

    RETURN v_result;
END;
$$ LANGUAGE plpgsql;

SELECT demo_safe_divide(100, 5)  AS ok_case;
SELECT demo_safe_divide(100, 0)  AS zero_case;
```

```
 ok_case
---------
      20

NOTICE:  ตรวจพบการหารด้วยศูนย์ (100 / 0) — คืนค่า NULL แทน
 zero_case
-----------

```

### ตัวอย่าง 2: จับ error หลายประเภทในฟังก์ชันเดียว — เพิ่มลูกค้าใหม่อย่างปลอดภัย

```sql
CREATE OR REPLACE FUNCTION demo_add_customer_safe(
    p_first_name TEXT,
    p_last_name  TEXT,
    p_email      TEXT,
    p_country    TEXT
)
RETURNS TEXT AS $$
DECLARE
    v_new_id INTEGER;
BEGIN
    INSERT INTO customers (first_name, last_name, email, country)
    VALUES (p_first_name, p_last_name, p_email, p_country)
    RETURNING customer_id INTO v_new_id;

    RETURN format('เพิ่มลูกค้าสำเร็จ: customer_id = %s', v_new_id);

EXCEPTION
    WHEN unique_violation THEN
        RETURN format('ล้มเหลว: อีเมล "%s" มีอยู่ในระบบแล้ว', p_email);
    WHEN not_null_violation THEN
        RETURN 'ล้มเหลว: มีข้อมูลจำเป็นที่ยังไม่ได้กรอก (NOT NULL)';
    WHEN OTHERS THEN
        RETURN format('ล้มเหลวด้วยสาเหตุที่ไม่คาดคิด: %s (SQLSTATE: %s)', SQLERRM, SQLSTATE);
END;
$$ LANGUAGE plpgsql;

SELECT demo_add_customer_safe('Kanya', 'Srisuk', 'kanya.s@example.com', 'Thailand');
SELECT demo_add_customer_safe('Duplicate', 'Test', 'somchai.j@example.com', 'Thailand'); -- email ซ้ำ
```

```
                demo_add_customer_safe
--------------------------------------------------------
 เพิ่มลูกค้าสำเร็จ: customer_id = 16

                demo_add_customer_safe
--------------------------------------------------------
 ล้มเหลว: อีเมล "somchai.j@example.com" มีอยู่ในระบบแล้ว
```

### 465.1 หลักการสำคัญของ Exception Handling

1. **บล็อกที่มี `EXCEPTION` จะสร้าง implicit savepoint** — PostgreSQL ต้องเก็บ state ก่อนเข้าบล็อกไว้เพื่อ rollback กลับได้ ทำให้มี overhead เล็กน้อย จึงไม่ควรครอบ `EXCEPTION` รอบโค้ดจำนวนมากโดยไม่จำเป็น (เช่น ในลูปที่วนหลายพันรอบ ควรครอบเฉพาะคำสั่งที่เสี่ยง error)
2. **`WHEN OTHERS` ควรอยู่ลำดับสุดท้ายเสมอ** เพราะเป็นตัวดักจับทุกอย่างที่เหลือ
3. การ error ที่ไม่ได้ถูกจับ จะทำให้ **ทั้ง transaction ถูก rollback** ไม่ใช่แค่บล็อกนั้น
4. ใช้ `GET STACKED DIAGNOSTICS` เพื่อดึงรายละเอียด error เพิ่มเติมได้ เช่น

```sql
CREATE OR REPLACE FUNCTION demo_diagnostics_example()
RETURNS VOID AS $$
DECLARE
    v_msg   TEXT;
    v_detail TEXT;
    v_hint  TEXT;
    v_state TEXT;
BEGIN
    BEGIN
        INSERT INTO order_items (order_id, product_id, quantity, unit_price)
        VALUES (1, 1, -5, 100);  -- quantity ติดลบ ผิด CHECK constraint
    EXCEPTION
        WHEN OTHERS THEN
            GET STACKED DIAGNOSTICS
                v_msg    = MESSAGE_TEXT,
                v_detail = PG_EXCEPTION_DETAIL,
                v_hint   = PG_EXCEPTION_HINT,
                v_state  = RETURNED_SQLSTATE;

            RAISE NOTICE 'Error message: %', v_msg;
            RAISE NOTICE 'Error detail : %', coalesce(v_detail, '(ไม่มี)');
            RAISE NOTICE 'SQLSTATE     : %', v_state;
    END;
END;
$$ LANGUAGE plpgsql;

SELECT demo_diagnostics_example();
```

```
NOTICE:  Error message: new row for relation "order_items" violates check constraint "order_items_quantity_check"
NOTICE:  Error detail : Failing row contains (24, 1, 1, -5, 100.00).
NOTICE:  SQLSTATE     : 23514
```

---

## Step 466: รายชื่อ exception ที่พบบ่อยและการเขียนโค้ดรับมือ

PostgreSQL กำหนดชื่อ condition (error) ไว้เป็นมาตรฐานจำนวนมาก (อ้างอิงจาก Appendix A: PostgreSQL Error Codes) ในงานจริงมีอยู่ไม่กี่ตัวที่เจอบ่อยที่สุด

### 466.1 unique_violation (SQLSTATE 23505)

เกิดเมื่อพยายาม insert/update ข้อมูลที่ซ้ำกับ unique constraint หรือ primary key

```sql
CREATE OR REPLACE FUNCTION demo_handle_unique_violation(p_email TEXT)
RETURNS TEXT AS $$
BEGIN
    INSERT INTO customers (first_name, last_name, email, country)
    VALUES ('Test', 'User', p_email, 'Thailand');

    RETURN 'เพิ่มข้อมูลสำเร็จ';
EXCEPTION
    WHEN unique_violation THEN
        RETURN format('อีเมล %s ถูกใช้แล้วในระบบ กรุณาใช้อีเมลอื่น', p_email);
END;
$$ LANGUAGE plpgsql;

SELECT demo_handle_unique_violation('john.smith@example.com');
```

```
                    demo_handle_unique_violation
----------------------------------------------------------------
 อีเมล john.smith@example.com ถูกใช้แล้วในระบบ กรุณาใช้อีเมลอื่น
```

### 466.2 foreign_key_violation (SQLSTATE 23503)

เกิดเมื่อ insert/update ข้อมูลที่อ้างอิง foreign key ซึ่งไม่มีอยู่จริง หรือพยายามลบแถวที่ยังถูกอ้างอิงอยู่

```sql
CREATE OR REPLACE FUNCTION demo_handle_fk_violation(p_customer_id INTEGER)
RETURNS TEXT AS $$
BEGIN
    INSERT INTO orders (customer_id, status, ship_country)
    VALUES (p_customer_id, 'pending', 'Thailand');

    RETURN 'สร้างคำสั่งซื้อสำเร็จ';
EXCEPTION
    WHEN foreign_key_violation THEN
        RETURN format('ไม่พบลูกค้า customer_id = %s ในระบบ ไม่สามารถสร้างคำสั่งซื้อได้', p_customer_id);
END;
$$ LANGUAGE plpgsql;

SELECT demo_handle_fk_violation(9999);  -- ไม่มีลูกค้ารหัสนี้
```

```
                        demo_handle_fk_violation
----------------------------------------------------------------------
 ไม่พบลูกค้า customer_id = 9999 ในระบบ ไม่สามารถสร้างคำสั่งซื้อได้
```

### 466.3 division_by_zero (SQLSTATE 22012)

เกิดเมื่อคำนวณหารด้วยศูนย์ (ตัวอย่างแสดงไปแล้วใน Step 465) มักเจอในการคำนวณอัตราส่วน เช่น conversion rate, average

### 466.4 ตัวอย่างรวม: ฟังก์ชันเดียวจับหลาย exception พร้อมกัน

```sql
CREATE OR REPLACE FUNCTION demo_add_order_item_safe(
    p_order_id   INTEGER,
    p_product_id INTEGER,
    p_quantity   INTEGER
)
RETURNS TEXT AS $$
DECLARE
    v_price NUMERIC;
BEGIN
    SELECT unit_price INTO v_price FROM products WHERE product_id = p_product_id;

    IF NOT FOUND THEN
        RAISE EXCEPTION 'ไม่พบสินค้า product_id = %', p_product_id
            USING ERRCODE = 'no_data_found';
    END IF;

    INSERT INTO order_items (order_id, product_id, quantity, unit_price)
    VALUES (p_order_id, p_product_id, p_quantity, v_price);

    RETURN 'เพิ่มรายการสินค้าในคำสั่งซื้อสำเร็จ';

EXCEPTION
    WHEN foreign_key_violation THEN
        RETURN format('order_id %s ไม่มีอยู่จริง', p_order_id);
    WHEN check_violation THEN
        RETURN format('จำนวน (%s) ต้องมากกว่า 0', p_quantity);
    WHEN no_data_found THEN
        RETURN SQLERRM;
    WHEN OTHERS THEN
        RETURN format('เกิดข้อผิดพลาดไม่คาดคิด: % (SQLSTATE %)', SQLERRM, SQLSTATE);
END;
$$ LANGUAGE plpgsql;

SELECT demo_add_order_item_safe(1, 2, 1);       -- สำเร็จ
SELECT demo_add_order_item_safe(9999, 2, 1);    -- order_id ไม่มีจริง -> foreign_key_violation
SELECT demo_add_order_item_safe(1, 2, -3);      -- quantity ติดลบ -> check_violation
SELECT demo_add_order_item_safe(1, 9999, 1);    -- product_id ไม่มีจริง -> no_data_found
```

```
 demo_add_order_item_safe
----------------------------------------
 เพิ่มรายการสินค้าในคำสั่งซื้อสำเร็จ

 demo_add_order_item_safe
----------------------------------------
 order_id 9999 ไม่มีอยู่จริง

 demo_add_order_item_safe
----------------------------------------
 จำนวน (-3) ต้องมากกว่า 0

 demo_add_order_item_safe
----------------------------------------
 ไม่พบสินค้า product_id = 9999
```

---

## Step 467: RAISE — สร้าง notice/warning/exception เอง

คำสั่ง `RAISE` มีระดับความรุนแรง (level) หลายแบบ:

| Level | ผลลัพธ์ | ใช้เมื่อ |
|---|---|---|
| `DEBUG` | log เฉพาะเมื่อตั้งค่า client_min_messages/log_min_messages | ข้อมูล debug ละเอียด |
| `LOG` | บันทึกลง server log | เหตุการณ์ระดับ server |
| `NOTICE` | ส่งข้อความให้ client เห็น (ค่า default ของ RAISE เปล่า ๆ) | แจ้งเตือนทั่วไป |
| `INFO` | ส่งข้อความให้ client เห็นเสมอ | ข้อมูลที่ต้องการให้ผู้ใช้เห็น |
| `WARNING` | แจ้งเตือนระดับสูงกว่า NOTICE | เหตุการณ์ผิดปกติแต่ยังทำงานต่อได้ |
| `EXCEPTION` | หยุดการทำงานทันทีและ throw error | ข้อผิดพลาดร้ายแรงที่ต้อง rollback |

### 467.1 รูปแบบการใช้งาน RAISE

```sql
RAISE [level] 'format_message' [, argument, ...] [USING option = expression, ...];
```

ตัวอย่างครบทุก level:

```sql
CREATE OR REPLACE FUNCTION demo_raise_levels()
RETURNS VOID AS $$
BEGIN
    RAISE DEBUG 'ข้อความระดับ DEBUG: เริ่มการทำงาน';
    RAISE LOG 'ข้อความระดับ LOG: บันทึกลง server log';
    RAISE NOTICE 'ข้อความระดับ NOTICE: ทั่วไป';
    RAISE INFO 'ข้อความระดับ INFO: ข้อมูลสำคัญ';
    RAISE WARNING 'ข้อความระดับ WARNING: ระวัง มีบางอย่างผิดปกติ';
    RAISE NOTICE 'จบการทำงานปกติ (ยังไม่ raise exception)';
END;
$$ LANGUAGE plpgsql;

SELECT demo_raise_levels();
```

```
NOTICE:  ข้อความระดับ NOTICE: ทั่วไป
INFO:  ข้อความระดับ INFO: ข้อมูลสำคัญ
WARNING:  ข้อความระดับ WARNING: ระวัง มีบางอย่างผิดปกติ
NOTICE:  จบการทำงานปกติ (ยังไม่ raise exception)
```

> `DEBUG` และ `LOG` จะไม่แสดงใน client ด้วยค่าตั้งต้น (`client_min_messages = notice`) แต่จะถูกบันทึกลง server log ตาม `log_min_messages`

### 467.2 RAISE EXCEPTION พร้อมข้อความและ error code กำหนดเอง

```sql
CREATE OR REPLACE FUNCTION demo_raise_custom_exception(p_stock INTEGER, p_requested INTEGER)
RETURNS VOID AS $$
BEGIN
    IF p_requested > p_stock THEN
        RAISE EXCEPTION 'สต็อกไม่เพียงพอ: มีอยู่ % ชิ้น แต่ต้องการ % ชิ้น', p_stock, p_requested
            USING
                ERRCODE = 'P0001',
                HINT    = 'กรุณาลดจำนวนที่สั่งซื้อ หรือรอสินค้าเข้าสต็อกใหม่',
                DETAIL  = 'ตรวจสอบจาก products.stock_quantity ก่อนสั่งซื้อเสมอ';
    END IF;

    RAISE NOTICE 'สั่งซื้อสำเร็จ: % ชิ้น จากสต็อก % ชิ้น', p_requested, p_stock;
END;
$$ LANGUAGE plpgsql;

SELECT demo_raise_custom_exception(10, 5);   -- ผ่าน
SELECT demo_raise_custom_exception(10, 50);  -- error
```

```
NOTICE:  สั่งซื้อสำเร็จ: 5 ชิ้น จากสต็อก 10 ชิ้น

ERROR:  สต็อกไม่เพียงพอ: มีอยู่ 10 ชิ้น แต่ต้องการ 50 ชิ้น
DETAIL:  ตรวจสอบจาก products.stock_quantity ก่อนสั่งซื้อเสมอ
HINT:  กรุณาลดจำนวนที่สั่งซื้อ หรือรอสินค้าเข้าสต็อกใหม่
```

### 467.3 กำหนด error code เอง (custom SQLSTATE) เพื่อให้แอปพลิเคชันดักจับเฉพาะเจาะจง

PostgreSQL แนะนำให้ error code ที่กำหนดเองอยู่ในช่วง `P0001`–`P0009` หรือขึ้นต้นด้วยตัวอักษรที่ไม่ชนกับมาตรฐาน (เช่น class `70000`-`99999` ตาม convention) เพื่อไม่ให้ชนกับ error code มาตรฐานของ PostgreSQL

```sql
CREATE OR REPLACE FUNCTION demo_custom_errcode(p_qty INTEGER)
RETURNS VOID AS $$
BEGIN
    IF p_qty <= 0 THEN
        RAISE EXCEPTION 'จำนวนต้องมากกว่า 0 (ได้รับ %)', p_qty
            USING ERRCODE = 'INVQT';  -- custom 5-character SQLSTATE
    END IF;
END;
$$ LANGUAGE plpgsql;

-- ฝั่งแอปพลิเคชัน (เช่น Python/psycopg2, Node/pg) สามารถดักจับด้วย err.code === 'INVQT' ได้
DO $$
BEGIN
    BEGIN
        PERFORM demo_custom_errcode(-1);
    EXCEPTION
        WHEN SQLSTATE 'INVQT' THEN
            RAISE NOTICE 'ดักจับ custom error code INVQT ได้สำเร็จ: %', SQLERRM;
    END;
END;
$$;
```

```
NOTICE:  ดักจับ custom error code INVQT ได้สำเร็จ: จำนวนต้องมากกว่า 0 (ได้รับ -1)
```

### 467.4 RAISE แบบสั้น — re-raise exception เดิม

ภายในบล็อก `EXCEPTION` สามารถเขียน `RAISE;` เฉย ๆ (ไม่มีข้อความ) เพื่อโยน exception เดิมต่อไปยังชั้นที่สูงกว่า มีประโยชน์เมื่อต้องการ log ก่อนแล้วค่อยปล่อยให้ error กระจายต่อ

```sql
CREATE OR REPLACE FUNCTION demo_reraise_example(p_val INTEGER)
RETURNS INTEGER AS $$
BEGIN
    BEGIN
        RETURN 100 / p_val;
    EXCEPTION
        WHEN division_by_zero THEN
            RAISE WARNING 'บันทึก log: ตรวจพบการหารด้วยศูนย์ที่ demo_reraise_example';
            RAISE;  -- โยน error เดิมต่อ ไม่ได้จับไว้เฉย ๆ
    END;
END;
$$ LANGUAGE plpgsql;

SELECT demo_reraise_example(0);
```

```
WARNING:  บันทึก log: ตรวจพบการหารด้วยศูนย์ที่ demo_reraise_example
ERROR:  division by zero
```

---

## Step 468: Dynamic SQL ด้วย EXECUTE

บางสถานการณ์เราไม่ทราบชื่อตาราง คอลัมน์ หรือโครงสร้างคำสั่ง SQL ล่วงหน้าตอนเขียนโค้ด (compile time) แต่ต้องประกอบขึ้นระหว่างการทำงานจริง (runtime) เช่น รายงานที่เลือก sort column ได้เอง, ฟังก์ชัน generic ที่ทำงานกับหลายตาราง — ใช้ `EXECUTE` เพื่อรัน SQL ที่เป็น string

### 468.1 รูปแบบพื้นฐาน

```sql
EXECUTE dynamic_sql_string [INTO variable] [USING param1, param2, ...];
```

```sql
CREATE OR REPLACE FUNCTION demo_execute_basic(p_table_name TEXT)
RETURNS BIGINT AS $$
DECLARE
    v_count BIGINT;
    v_sql   TEXT;
BEGIN
    v_sql := 'SELECT count(*) FROM ' || p_table_name;
    EXECUTE v_sql INTO v_count;
    RETURN v_count;
END;
$$ LANGUAGE plpgsql;

SELECT demo_execute_basic('products');
```

```
 demo_execute_basic
---------------------
                  20
```

### 468.2 ช่องโหว่ SQL Injection — ตัวอย่างที่ **อันตราย**

โค้ดข้างบนดู "ทำงานได้" แต่มีช่องโหว่ร้ายแรง เพราะต่อ string ตรง ๆ โดยไม่กรองข้อมูล ลองดูตัวอย่างการโจมตี:

```sql
-- ตัวอย่างการโจมตีสมมติ (ห้ามใช้ pattern นี้ในโค้ดจริง!)
SELECT demo_execute_basic('products; DROP TABLE customers; --');
```

หากฟังก์ชันเขียนแบบอันตรายกว่านี้ เช่น รับ `WHERE` clause จาก input ผู้ใช้โดยตรง จะยิ่งเสี่ยงมาก:

```sql
-- ❌ ตัวอย่างโค้ดที่ "ห้ามเขียน" แบบนี้เด็ดขาด
CREATE OR REPLACE FUNCTION demo_UNSAFE_search(p_name_filter TEXT)
RETURNS SETOF products AS $$
DECLARE
    v_sql TEXT;
BEGIN
    -- อันตราย: ต่อ string ของผู้ใช้เข้ากับ SQL โดยตรง
    v_sql := 'SELECT * FROM products WHERE product_name = ''' || p_name_filter || '''';
    RETURN QUERY EXECUTE v_sql;
END;
$$ LANGUAGE plpgsql;

-- หากผู้ใช้ส่งค่านี้เข้ามา:
-- p_name_filter = ''' OR ''1''=''1'
-- v_sql จะกลายเป็น: SELECT * FROM products WHERE product_name = '' OR '1'='1'
-- ผลลัพธ์คือดึงข้อมูล "ทุกแถว" ในตาราง ทั้งที่ตั้งใจจะกรองเฉพาะชื่อที่ตรงกัน
SELECT * FROM demo_UNSAFE_search(''' OR ''1''=''1');
```

```
-- ผลลัพธ์: คืนสินค้าทั้ง 20 รายการ ทั้งที่ผู้ใช้ไม่ได้ระบุชื่อสินค้าจริง
-- นี่คือช่องโหว่ SQL Injection แบบคลาสสิก
```

### 468.3 วิธีป้องกันที่ 1 — ใช้ `USING` เพื่อ bind parameter (แนะนำที่สุดสำหรับค่าข้อมูล)

```sql
CREATE OR REPLACE FUNCTION demo_safe_search_using(p_name_filter TEXT)
RETURNS SETOF products AS $$
BEGIN
    RETURN QUERY EXECUTE 'SELECT * FROM products WHERE product_name = $1'
        USING p_name_filter;
END;
$$ LANGUAGE plpgsql;

SELECT * FROM demo_safe_search_using('Wireless Mouse M1');
-- แม้ผู้ใช้ส่ง ''' OR ''1''=''1' เข้ามา ก็จะถูกมองเป็น "ค่า string" ธรรมดา ไม่ใช่โค้ด SQL
SELECT * FROM demo_safe_search_using(''' OR ''1''=''1');
```

```
 product_id |    product_name    | category_id | supplier_id | unit_price | stock_quantity | is_active
------------+---------------------+-------------+-------------+------------+-----------------+-----------
          1 | Wireless Mouse M1   |           2 |           1 |     290.00 |             150 | t

(0 rows)   -- ค่า injection string ไม่ match ชื่อสินค้าใด ๆ จึงไม่มีผลลัพธ์ ปลอดภัย
```

### 468.4 วิธีป้องกันที่ 2 — `quote_literal()` เมื่อต้องต่อ string ค่าข้อมูลเข้ากับ SQL

ในบางกรณี (เช่น ต้องประกอบ SQL ที่ซับซ้อนซึ่ง `USING` ไม่รองรับโดยตรง) จำเป็นต้องต่อ string เอง — ต้องใช้ `quote_literal()` เพื่อ escape ค่าให้ปลอดภัยเสมอ

```sql
CREATE OR REPLACE FUNCTION demo_quote_literal_example(p_name_filter TEXT)
RETURNS TEXT AS $$
DECLARE
    v_sql TEXT;
BEGIN
    v_sql := 'SELECT product_name FROM products WHERE product_name = ' || quote_literal(p_name_filter);
    RAISE NOTICE 'SQL ที่จะรัน: %', v_sql;
    RETURN v_sql;
END;
$$ LANGUAGE plpgsql;

SELECT demo_quote_literal_example(''' OR ''1''=''1');
```

```
NOTICE:  SQL ที่จะรัน: SELECT product_name FROM products WHERE product_name = ''' OR ''1''=''1'
```

`quote_literal()` จะ escape single quote ทั้งหมดให้อัตโนมัติ ทำให้ string ที่ต่อกันไม่สามารถ "หลุด" ออกจากขอบเขตของ literal เดิมได้ — ผลลัพธ์จึงถูกมองเป็นค่าข้อมูลล้วน ๆ ไม่ใช่โค้ด SQL

### 468.5 วิธีป้องกันที่ 3 — `quote_ident()` เมื่อชื่อตาราง/คอลัมน์มาจาก input

`quote_literal()` ใช้กับ **ค่าข้อมูล (value)** แต่เมื่อต้องรับ **ชื่อ object** เช่น ชื่อตาราง ชื่อคอลัมน์ จาก input ต้องใช้ `quote_ident()` แทน เพราะไม่สามารถใช้ `USING` bind parameter แทนชื่อ object ได้ (parameter bind ใช้แทนค่าได้เท่านั้น ไม่สามารถแทนชื่อตาราง/คอลัมน์)

```sql
CREATE OR REPLACE FUNCTION demo_dynamic_sort(p_table_name TEXT, p_sort_column TEXT, p_limit INTEGER DEFAULT 5)
RETURNS SETOF RECORD AS $$
DECLARE
    v_sql TEXT;
BEGIN
    -- ตรวจสอบว่าตาราง/คอลัมน์มีอยู่จริงในระบบก่อนเสมอ (whitelist validation)
    IF NOT EXISTS (
        SELECT 1 FROM information_schema.columns
        WHERE table_name = p_table_name AND column_name = p_sort_column
    ) THEN
        RAISE EXCEPTION 'ตาราง % หรือคอลัมน์ % ไม่มีอยู่จริง', p_table_name, p_sort_column;
    END IF;

    v_sql := format(
        'SELECT * FROM %I ORDER BY %I DESC LIMIT %L',
        p_table_name, p_sort_column, p_limit
    );
    RAISE NOTICE 'SQL ที่จะรัน: %', v_sql;

    RETURN QUERY EXECUTE v_sql;
END;
$$ LANGUAGE plpgsql;

SELECT * FROM demo_dynamic_sort('products', 'unit_price', 3)
    AS t(product_id INTEGER, product_name TEXT, category_id INTEGER, supplier_id INTEGER,
         unit_price NUMERIC, stock_quantity INTEGER, is_active BOOLEAN);
```

```
NOTICE:  SQL ที่จะรัน: SELECT * FROM "products" ORDER BY "unit_price" DESC LIMIT '3'

 product_id |     product_name     | ... | unit_price
------------+-----------------------+-----+------------
          4 | Smartphone Zeta 12    | ... |   15900.00
          7 | 4K Monitor 27-inch    | ... |    7900.00
          5 | Smartphone Zeta 12 Lite | ... |   9900.00
```

### 468.6 `format()` — วิธีที่แนะนำที่สุดในการประกอบ Dynamic SQL

`format()` เป็นฟังก์ชันที่รวมพลังของ `quote_ident()` (`%I`) และ `quote_literal()` (`%L`) ไว้ในที่เดียว อ่านง่ายกว่าการต่อ string ด้วย `||` มาก และลดโอกาสลืม escape

| Format specifier | ความหมาย |
|---|---|
| `%s` | แทรกค่าแบบ string ธรรมดา (ไม่ escape — ใช้กับค่าที่เชื่อถือได้เท่านั้น เช่น ตัวเลขคงที่) |
| `%I` | แทรกเป็น **identifier** (ชื่อตาราง/คอลัมน์) พร้อม escape/quote ให้อัตโนมัติ |
| `%L` | แทรกเป็น **literal** (ค่าข้อมูล) พร้อม escape/quote ให้อัตโนมัติ เหมือน `quote_literal()` |

```sql
CREATE OR REPLACE FUNCTION demo_format_generic_update(
    p_table_name  TEXT,
    p_id_column   TEXT,
    p_id_value    INTEGER,
    p_set_column  TEXT,
    p_set_value   TEXT
)
RETURNS TEXT AS $$
DECLARE
    v_sql TEXT;
    v_row_count INTEGER;
BEGIN
    -- whitelist ตรวจสอบตาราง/คอลัมน์ก่อนเสมอ (ป้องกันชื่อ object ปลอม)
    IF NOT EXISTS (
        SELECT 1 FROM information_schema.columns
        WHERE table_name = p_table_name AND column_name IN (p_id_column, p_set_column)
        HAVING count(*) = 2
    ) THEN
        RAISE EXCEPTION 'ตาราง/คอลัมน์ที่ระบุไม่ถูกต้อง: table=%, id_col=%, set_col=%',
            p_table_name, p_id_column, p_set_column;
    END IF;

    v_sql := format(
        'UPDATE %I SET %I = %L WHERE %I = %L',
        p_table_name, p_set_column, p_set_value, p_id_column, p_id_value
    );
    RAISE NOTICE 'SQL ที่จะรัน: %', v_sql;

    EXECUTE v_sql;
    GET DIAGNOSTICS v_row_count = ROW_COUNT;

    RETURN format('อัปเดตสำเร็จ %s แถว', v_row_count);
END;
$$ LANGUAGE plpgsql;

SELECT demo_format_generic_update('products', 'product_id', 16, 'is_active', 'true');
```

```
NOTICE:  SQL ที่จะรัน: UPDATE "products" SET "is_active" = 'true' WHERE "product_id" = '16'
 demo_format_generic_update
-----------------------------
 อัปเดตสำเร็จ 1 แถว
```

> **สรุปหลักการป้องกัน SQL Injection:**
> 1. ค่าข้อมูล (value) → ใช้ `USING` bind parameter เป็นอันดับแรก หรือ `%L`/`quote_literal()` ถ้าต้องต่อ string
> 2. ชื่อ object (ตาราง/คอลัมน์) → ใช้ `%I`/`quote_ident()` เสมอ พร้อม whitelist validate กับ `information_schema` หรือรายการที่อนุญาตไว้ล่วงหน้า
> 3. หลีกเลี่ยงการต่อ string ด้วย `||` โดยตรงถ้าใช้ `format()` แทนได้
> 4. ห้ามเชื่อถือ input จากผู้ใช้โดยเด็ดขาด แม้จะมาจากระบบภายในก็ตาม

---

## Step 469: Cursor เบื้องต้น — DECLARE CURSOR, FETCH

Cursor คือกลไกที่ให้เราอ่านผลลัพธ์ของ query **ทีละแถว** โดยไม่ต้องโหลดข้อมูลทั้งหมดเข้าหน่วยความจำในครั้งเดียว เหมาะกับการประมวลผลข้อมูลขนาดใหญ่มาก (หลักล้านแถว) ที่การใช้ `FOR...IN SELECT` (ซึ่งใช้ cursor แบบ implicit อยู่แล้ว) อาจไม่สะดวกพอ เช่น ต้องการควบคุมการ fetch ทีละหลายแถว หรือต้องเปิด-ปิด cursor ข้าม transaction

### 469.1 รูปแบบพื้นฐานของ Explicit Cursor

```sql
DECLARE cursor_name CURSOR FOR SELECT ...;
OPEN cursor_name;
FETCH cursor_name INTO variable_list;
CLOSE cursor_name;
```

### ตัวอย่าง 1: วนอ่านคำสั่งซื้อทีละแถวด้วย cursor

```sql
CREATE OR REPLACE FUNCTION demo_cursor_basic()
RETURNS VOID AS $$
DECLARE
    v_cur CURSOR FOR
        SELECT order_id, customer_id, status
        FROM orders
        ORDER BY order_id;
    v_order_id    INTEGER;
    v_customer_id INTEGER;
    v_status      VARCHAR(20);
    v_count       INTEGER := 0;
BEGIN
    OPEN v_cur;

    LOOP
        FETCH v_cur INTO v_order_id, v_customer_id, v_status;
        EXIT WHEN NOT FOUND;   -- ไม่มีแถวเหลือแล้ว -> ออกจาก loop

        v_count := v_count + 1;
        RAISE NOTICE 'แถวที่ %: order_id=%, customer_id=%, status=%',
            v_count, v_order_id, v_customer_id, v_status;
    END LOOP;

    CLOSE v_cur;
    RAISE NOTICE 'อ่านทั้งหมด % แถว', v_count;
END;
$$ LANGUAGE plpgsql;

SELECT demo_cursor_basic();
```

```
NOTICE:  แถวที่ 1: order_id=1, customer_id=1, status=completed
NOTICE:  แถวที่ 2: order_id=2, customer_id=2, status=completed
...
NOTICE:  แถวที่ 18: order_id=18, customer_id=15, status=completed
NOTICE:  อ่านทั้งหมด 18 แถว
```

### ตัวอย่าง 2: Cursor แบบมี parameter (bound cursor)

```sql
CREATE OR REPLACE FUNCTION demo_cursor_with_param(p_country TEXT)
RETURNS VOID AS $$
DECLARE
    v_cur CURSOR (p_ctry TEXT) FOR
        SELECT customer_id, first_name, last_name
        FROM customers
        WHERE country = p_ctry
        ORDER BY customer_id;
    v_rec RECORD;
BEGIN
    OPEN v_cur(p_country);

    LOOP
        FETCH v_cur INTO v_rec;
        EXIT WHEN NOT FOUND;
        RAISE NOTICE 'ลูกค้าจาก %: % %', p_country, v_rec.first_name, v_rec.last_name;
    END LOOP;

    CLOSE v_cur;
END;
$$ LANGUAGE plpgsql;

SELECT demo_cursor_with_param('Japan');
```

```
NOTICE:  ลูกค้าจาก Japan: Akira Tanaka
NOTICE:  ลูกค้าจาก Japan: Yuki Sato
```

### ตัวอย่าง 3: FETCH หลายรูปแบบ (FORWARD, RELATIVE, ABSOLUTE) กับ scrollable cursor

ปกติ cursor จะอ่านทีละแถวไปข้างหน้าเท่านั้น แต่หากประกาศเป็น `SCROLL` จะสามารถเลื่อนไปมาได้

```sql
CREATE OR REPLACE FUNCTION demo_scrollable_cursor()
RETURNS VOID AS $$
DECLARE
    v_cur SCROLL CURSOR FOR
        SELECT product_id, product_name, unit_price
        FROM products
        ORDER BY unit_price DESC;
    v_rec RECORD;
BEGIN
    OPEN v_cur;

    -- ดึงแถวแรก
    FETCH FIRST FROM v_cur INTO v_rec;
    RAISE NOTICE 'แถวแรก (แพงสุด): % - %', v_rec.product_name, v_rec.unit_price;

    -- เลื่อนไปข้างหน้า 2 แถว
    FETCH RELATIVE 2 FROM v_cur INTO v_rec;
    RAISE NOTICE 'เลื่อนไปอีก 2 แถว: % - %', v_rec.product_name, v_rec.unit_price;

    -- ดึงแถวสุดท้าย
    FETCH LAST FROM v_cur INTO v_rec;
    RAISE NOTICE 'แถวสุดท้าย (ถูกสุด): % - %', v_rec.product_name, v_rec.unit_price;

    -- ถอยหลังกลับไป 1 แถว
    FETCH PRIOR FROM v_cur INTO v_rec;
    RAISE NOTICE 'ถอยกลับ 1 แถว: % - %', v_rec.product_name, v_rec.unit_price;

    CLOSE v_cur;
END;
$$ LANGUAGE plpgsql;

SELECT demo_scrollable_cursor();
```

```
NOTICE:  แถวแรก (แพงสุด): Smartphone Zeta 12 - 15900.00
NOTICE:  เลื่อนไปอีก 2 แถว: 4K Monitor 27-inch - 7900.00
NOTICE:  แถวสุดท้าย (ถูกสุด): Men Cotton T-Shirt - 250.00
NOTICE:  ถอยกลับ 1 แถว: Yoga Mat Premium - 490.00
```

### 469.2 ทำไมต้องใช้ Cursor เมื่อข้อมูลใหญ่มาก

- `FOR rec IN SELECT ...` ใช้ cursor แบบ implicit อยู่แล้ว **เพียงพอสำหรับงานส่วนใหญ่** และเขียนง่ายกว่า ควรเลือกใช้ก่อนเสมอ
- Explicit cursor เหมาะกับกรณีที่ต้องการ:
  - ควบคุมการ fetch แบบละเอียด (fetch N แถวต่อครั้ง, scroll ไปมา)
  - แบ่งการประมวลผลเป็น batch เพื่อ commit เป็นช่วง ๆ ลดขนาด transaction log และหลีกเลี่ยง lock ค้างนาน
  - ส่ง cursor เป็น `REFCURSOR` กลับไปให้ฝั่ง client จัดการต่อ (ผ่าน `RETURN` เป็น `refcursor`)

### ตัวอย่าง 4: ประมวลผลแบบ batch ด้วย cursor เพื่อลดภาระต่อ transaction เดียว

```sql
CREATE OR REPLACE FUNCTION demo_cursor_batch_update(p_batch_size INTEGER DEFAULT 5)
RETURNS TEXT AS $$
DECLARE
    v_cur CURSOR FOR
        SELECT product_id, stock_quantity
        FROM products
        WHERE is_active = true
        ORDER BY product_id
        FOR UPDATE;
    v_rec RECORD;
    v_batch_count INTEGER := 0;
    v_total_count INTEGER := 0;
BEGIN
    OPEN v_cur;

    LOOP
        FETCH v_cur INTO v_rec;
        EXIT WHEN NOT FOUND;

        UPDATE products
        SET stock_quantity = stock_quantity + 1  -- ตัวอย่าง: เติมสต็อก +1 ทุกตัว
        WHERE CURRENT OF v_cur;

        v_batch_count := v_batch_count + 1;
        v_total_count := v_total_count + 1;

        IF v_batch_count >= p_batch_size THEN
            RAISE NOTICE 'ประมวลผลครบ % แถวในรอบนี้ (สะสม % แถว)', v_batch_count, v_total_count;
            v_batch_count := 0;
        END IF;
    END LOOP;

    CLOSE v_cur;
    RETURN format('เสร็จสิ้น: ประมวลผลทั้งหมด %s แถว', v_total_count);
END;
$$ LANGUAGE plpgsql;

SELECT demo_cursor_batch_update(5);
```

```
NOTICE:  ประมวลผลครบ 5 แถวในรอบนี้ (สะสม 5 แถว)
NOTICE:  ประมวลผลครบ 5 แถวในรอบนี้ (สะสม 10 แถว)
NOTICE:  ประมวลผลครบ 5 แถวในรอบนี้ (สะสม 15 แถว)
 demo_cursor_batch_update
----------------------------
 เสร็จสิ้น: ประมวลผลทั้งหมด 19 แถว
```

> `WHERE CURRENT OF v_cur` เป็นเทคนิคพิเศษที่ให้ `UPDATE`/`DELETE` กระทำกับ **แถวปัจจุบัน** ที่ cursor ชี้อยู่พอดี โดยไม่ต้องระบุเงื่อนไขซ้ำ ต้องใช้คู่กับ cursor ที่ประกาศด้วย `FOR UPDATE` เท่านั้น

---

## Step 470: แบบฝึกหัดรวม — ฟังก์ชันประมวลผล Batch แบบครบวงจร

ในหัวข้อนี้เราจะรวมทุกเทคนิคที่เรียนมาในบทนี้ — **loop, exception handling, dynamic SQL, cursor, RAISE** — เข้าด้วยกันในฟังก์ชันเดียว เพื่อสร้างระบบ **batch update ราคาสินค้าตามเงื่อนไขซับซ้อน** พร้อม log error ที่เกิดขึ้นระหว่างทาง โดยไม่ทำให้ทั้ง batch ล้มเหลวเมื่อมีบางแถวมีปัญหา

### 470.1 สร้างตาราง log สำหรับเก็บผลลัพธ์การประมวลผล

```sql
CREATE TABLE IF NOT EXISTS batch_price_update_log (
    log_id       SERIAL PRIMARY KEY,
    batch_run_id UUID NOT NULL,
    product_id   INTEGER,
    old_price    NUMERIC(10,2),
    new_price    NUMERIC(10,2),
    status       VARCHAR(20) NOT NULL,  -- 'SUCCESS' หรือ 'FAILED'
    error_message TEXT,
    processed_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### 470.2 ฟังก์ชันหลัก: batch update ราคาสินค้าตามหมวดหมู่ พร้อม dynamic SQL และ error handling

โจทย์: ปรับราคาสินค้าทุกตัวในหมวดหมู่ที่กำหนด โดย

- เพิ่มราคา (`p_adjust_percent`) เป็นเปอร์เซ็นต์ (เช่น 10 หมายถึงเพิ่ม 10%)
- ชื่อคอลัมน์ที่จะใช้กรอง (`p_filter_column`) รับมาแบบ dynamic เพื่อให้ยืดหยุ่น (เช่น จะ filter ด้วย `category_id` หรือ `supplier_id` ก็ได้) — ใช้ `format()` + `quote_ident()` ป้องกัน SQL Injection
- ราคาที่คำนวณได้ต้องไม่ต่ำกว่า 1.00 บาท มิฉะนั้นถือเป็น error และบันทึก log แล้วข้ามไปสินค้าตัวถัดไป (ไม่ให้ทั้ง batch ล้ม)
- ใช้ cursor วนอ่านทีละแถวเพื่อรองรับข้อมูลจำนวนมาก
- จับ exception ทุกประเภทที่อาจเกิดขึ้นระหว่าง update แต่ละแถว แล้วบันทึกลง log พร้อม `RAISE WARNING`

```sql
CREATE OR REPLACE FUNCTION batch_update_prices(
    p_filter_column   TEXT,      -- เช่น 'category_id' หรือ 'supplier_id'
    p_filter_value    INTEGER,   -- ค่าที่ใช้กรอง
    p_adjust_percent  NUMERIC    -- เปอร์เซ็นต์ที่จะปรับ (บวกคือขึ้นราคา, ลบคือลดราคา)
)
RETURNS TABLE(
    total_processed INTEGER,
    total_success   INTEGER,
    total_failed    INTEGER,
    run_id          UUID
) AS $$
DECLARE
    v_run_id       UUID := gen_random_uuid();
    v_sql          TEXT;
    v_cur          REFCURSOR;
    v_rec          RECORD;
    v_new_price    NUMERIC(10,2);
    v_count_total  INTEGER := 0;
    v_count_ok     INTEGER := 0;
    v_count_fail   INTEGER := 0;
    v_allowed_columns TEXT[] := ARRAY['category_id', 'supplier_id'];
BEGIN
    -- 1) Validate: ตรวจสอบว่าคอลัมน์ filter อยู่ใน whitelist เท่านั้น (ป้องกัน SQL Injection ที่ชื่อ column)
    IF NOT (p_filter_column = ANY(v_allowed_columns)) THEN
        RAISE EXCEPTION 'คอลัมน์ % ไม่อยู่ในรายการที่อนุญาต (%)', p_filter_column, v_allowed_columns
            USING ERRCODE = 'invalid_column_reference';
    END IF;

    -- 2) Validate: เปอร์เซ็นต์ปรับราคาต้องอยู่ในช่วงที่สมเหตุสมผล
    IF p_adjust_percent < -90 OR p_adjust_percent > 500 THEN
        RAISE EXCEPTION 'เปอร์เซ็นต์การปรับราคา (%) ผิดปกติ ต้องอยู่ระหว่าง -90 ถึง 500', p_adjust_percent;
    END IF;

    RAISE NOTICE '=== เริ่ม batch run: % (filter %.% = %, adjust %%%) ===',
        v_run_id, 'products', p_filter_column, p_filter_value, p_adjust_percent;

    -- 3) ประกอบ dynamic SQL อย่างปลอดภัยด้วย format() + %I (identifier) + %L (literal)
    v_sql := format(
        'SELECT product_id, product_name, unit_price FROM products WHERE %I = %L ORDER BY product_id',
        p_filter_column, p_filter_value
    );

    OPEN v_cur FOR EXECUTE v_sql;

    LOOP
        FETCH v_cur INTO v_rec;
        EXIT WHEN NOT FOUND;

        v_count_total := v_count_total + 1;

        -- ครอบแต่ละแถวด้วย sub-block ของตัวเองเพื่อไม่ให้ error หนึ่งแถวทำให้ทั้ง batch ล้ม
        BEGIN
            v_new_price := round(v_rec.unit_price * (1 + p_adjust_percent / 100.0), 2);

            IF v_new_price < 1.00 THEN
                RAISE EXCEPTION 'ราคาที่คำนวณได้ (%) ต่ำกว่าขั้นต่ำที่อนุญาต (1.00 บาท)', v_new_price
                    USING ERRCODE = 'P0002';
            END IF;

            UPDATE products
            SET unit_price = v_new_price
            WHERE product_id = v_rec.product_id;

            INSERT INTO batch_price_update_log
                (batch_run_id, product_id, old_price, new_price, status)
            VALUES
                (v_run_id, v_rec.product_id, v_rec.unit_price, v_new_price, 'SUCCESS');

            v_count_ok := v_count_ok + 1;
            RAISE NOTICE '  [OK] product_id=% (%): % -> %',
                v_rec.product_id, v_rec.product_name, v_rec.unit_price, v_new_price;

        EXCEPTION
            WHEN SQLSTATE 'P0002' THEN
                v_count_fail := v_count_fail + 1;
                INSERT INTO batch_price_update_log
                    (batch_run_id, product_id, old_price, new_price, status, error_message)
                VALUES
                    (v_run_id, v_rec.product_id, v_rec.unit_price, v_new_price, 'FAILED', SQLERRM);
                RAISE WARNING '  [FAILED] product_id=%: %', v_rec.product_id, SQLERRM;

            WHEN OTHERS THEN
                v_count_fail := v_count_fail + 1;
                INSERT INTO batch_price_update_log
                    (batch_run_id, product_id, old_price, new_price, status, error_message)
                VALUES
                    (v_run_id, v_rec.product_id, v_rec.unit_price, NULL, 'FAILED', SQLERRM);
                RAISE WARNING '  [FAILED] product_id=% เกิดข้อผิดพลาดไม่คาดคิด: %', v_rec.product_id, SQLERRM;
        END;
    END LOOP;

    CLOSE v_cur;

    RAISE NOTICE '=== จบ batch run: % (ทั้งหมด % / สำเร็จ % / ล้มเหลว %) ===',
        v_run_id, v_count_total, v_count_ok, v_count_fail;

    RETURN QUERY SELECT v_count_total, v_count_ok, v_count_fail, v_run_id;
END;
$$ LANGUAGE plpgsql;
```

### 470.3 ทดสอบใช้งาน

**กรณีที่ 1: ปรับราคาสินค้าหมวด Computers (category_id = 2) ขึ้น 8%**

```sql
SELECT * FROM batch_update_prices('category_id', 2, 8);
```

```
NOTICE:  === เริ่ม batch run: a1b2c3d4-... (filter products.category_id = 2, adjust 8%) ===
NOTICE:    [OK] product_id=1 (Wireless Mouse M1): 290.00 -> 313.20
NOTICE:    [OK] product_id=2 (Mechanical Keyboard K87): 1890.00 -> 2041.20
NOTICE:    [OK] product_id=3 (Laptop Stand Aluminum): 590.00 -> 637.20
NOTICE:    [OK] product_id=7 (4K Monitor 27-inch): 7900.00 -> 8532.00
NOTICE:  === จบ batch run: a1b2c3d4-... (ทั้งหมด 4 / สำเร็จ 4 / ล้มเหลว 0) ===

 total_processed | total_success | total_failed |               run_id
------------------+----------------+---------------+--------------------------------------
                4 |              4 |             0 | a1b2c3d4-e5f6-7890-abcd-ef1234567890
```

**กรณีที่ 2: ปรับราคาลดลง 95% เพื่อจำลองราคาต่ำกว่าขั้นต่ำ (ทดสอบ error handling)**

```sql
SELECT * FROM batch_update_prices('category_id', 7, -95);
```

```
NOTICE:  === เริ่ม batch run: b2c3d4e5-... (filter products.category_id = 7, adjust -95%) ===
WARNING:    [FAILED] product_id=11: ราคาที่คำนวณได้ (12.50) ต่ำกว่าขั้นต่ำที่อนุญาต (1.00 บาท)
WARNING:    [FAILED] product_id=12: ราคาที่คำนวณได้ (34.50) ต่ำกว่าขั้นต่ำที่อนุญาต (1.00 บาท)
NOTICE:  === จบ batch run: b2c3d4e5-... (ทั้งหมด 2 / สำเร็จ 0 / ล้มเหลว 2) ===

 total_processed | total_success | total_failed |               run_id
------------------+----------------+---------------+--------------------------------------
                2 |              0 |             2 | b2c3d4e5-f6a7-8901-bcde-f12345678901
```

**กรณีที่ 3: ทดสอบ SQL Injection ป้องกันการโจมตี**

```sql
-- พยายามส่งชื่อคอลัมน์ที่ไม่อยู่ใน whitelist
SELECT * FROM batch_update_prices('unit_price; DROP TABLE products; --', 1, 10);
```

```
ERROR:  คอลัมน์ unit_price; DROP TABLE products; -- ไม่อยู่ในรายการที่อนุญาต ({category_id,supplier_id})
```

> ฟังก์ชันปฏิเสธคำสั่งตั้งแต่ขั้นตอน validation ก่อนที่จะแตะ `EXECUTE` เลยด้วยซ้ำ — นี่คือหลักการ **"validate ก่อนประกอบ SQL เสมอ"**

**ตรวจสอบ log ที่บันทึกไว้:**

```sql
SELECT batch_run_id, product_id, old_price, new_price, status, error_message
FROM batch_price_update_log
ORDER BY log_id DESC
LIMIT 10;
```

```
              batch_run_id            | product_id | old_price | new_price | status  |                    error_message
---------------------------------------+------------+-----------+-----------+---------+-------------------------------------------------------
 b2c3d4e5-f6a7-8901-bcde-f12345678901  |         12 |    690.00 |     34.50 | FAILED  | ราคาที่คำนวณได้ (34.50) ต่ำกว่าขั้นต่ำที่อนุญาต (1.00 บาท)
 b2c3d4e5-f6a7-8901-bcde-f12345678901  |         11 |    250.00 |     12.50 | FAILED  | ราคาที่คำนวณได้ (12.50) ต่ำกว่าขั้นต่ำที่อนุญาต (1.00 บาท)
 a1b2c3d4-e5f6-7890-abcd-ef1234567890  |          7 |   7900.00 |   8532.00 | SUCCESS |
 a1b2c3d4-e5f6-7890-abcd-ef1234567890  |          3 |    590.00 |    637.20 | SUCCESS |
 a1b2c3d4-e5f6-7890-abcd-ef1234567890  |          2 |   1890.00 |   2041.20 | SUCCESS |
 a1b2c3d4-e5f6-7890-abcd-ef1234567890  |          1 |    290.00 |    313.20 | SUCCESS |
```

ฟังก์ชันนี้แสดงให้เห็นภาพรวมของการเขียน PL/pgSQL ขั้นสูงในงานจริง:

1. **Cursor (`REFCURSOR` + dynamic `OPEN ... FOR EXECUTE`)** — อ่านข้อมูลทีละแถวโดยไม่โหลดทั้งหมดเข้าหน่วยความจำ
2. **Dynamic SQL ที่ปลอดภัย** — ใช้ `format()` กับ `%I`/`%L` ร่วมกับ whitelist validation ป้องกัน SQL Injection ทั้งที่ชื่อคอลัมน์และค่าข้อมูล
3. **Exception handling แบบ per-row** — ครอบ `BEGIN...EXCEPTION...END` รอบแต่ละแถว ทำให้ error หนึ่งแถวไม่ทำให้ batch ทั้งหมดล้มเหลว
4. **RAISE หลายระดับ** — `NOTICE` สำหรับ progress, `WARNING` สำหรับแถวที่ล้มเหลว, `EXCEPTION` สำหรับ input ที่ผิดตั้งแต่ต้น
5. **Loop with cursor** — `LOOP...FETCH...EXIT WHEN NOT FOUND` เป็น pattern มาตรฐานสำหรับ batch processing
6. **Logging** — บันทึกทุก transaction ลงตาราง log เพื่อ audit และ troubleshoot ย้อนหลังได้

---

## สรุปท้ายบท

บทนี้ครอบคลุมเครื่องมือสำคัญที่ทำให้ PL/pgSQL กลายเป็นภาษาโปรแกรมที่ทรงพลังสำหรับเขียน business logic ฝั่งฐานข้อมูล:

- **Loop 4 รูปแบบ**: `LOOP...EXIT WHEN` (ไม่มีเงื่อนไขในตัว), `WHILE` (เช็คก่อนเข้า), `FOR` (ตัวเลขหรือผลลัพธ์ query), `FOREACH` (วน array)
- **CONTINUE/EXIT พร้อม label** — จำเป็นเมื่อต้องควบคุม nested loop จากชั้นใน
- **Exception handling** — `BEGIN...EXCEPTION...END` ทำให้ฟังก์ชันทนทานต่อ error โดยไม่ทำให้ทั้งระบบล่ม
- **RAISE** — สร้างข้อความแจ้งเตือนและ error เองได้ครบทุกระดับความรุนแรง พร้อมกำหนด error code เอง
- **Dynamic SQL** — `EXECUTE` ทำให้เขียนฟังก์ชัน generic ได้ แต่ต้องระวัง SQL Injection เสมอด้วย `format()`, `%I`, `%L`
- **Cursor** — เครื่องมือสำหรับอ่านข้อมูลขนาดใหญ่ทีละแถวอย่างมีประสิทธิภาพ

### ตารางสรุป Exception ที่พบบ่อย

| Condition Name | SQLSTATE | เกิดเมื่อ | ตัวอย่างสถานการณ์ในระบบอีคอมเมิร์ซ |
|---|---|---|---|
| `unique_violation` | 23505 | ข้อมูลซ้ำกับ UNIQUE/PRIMARY KEY constraint | อีเมลลูกค้าซ้ำตอนสมัครสมาชิก |
| `foreign_key_violation` | 23503 | อ้างอิง key ที่ไม่มีอยู่จริง หรือลบแถวที่ยังถูกอ้างอิง | สร้าง order ด้วย customer_id ที่ไม่มีในระบบ |
| `not_null_violation` | 23502 | ใส่ค่า NULL ให้คอลัมน์ที่เป็น NOT NULL | ลืมกรอก `product_name` ตอน insert สินค้า |
| `check_violation` | 23514 | ข้อมูลผิดเงื่อนไข CHECK constraint | ใส่ `quantity` ติดลบใน order_items |
| `division_by_zero` | 22012 | หารด้วยศูนย์ | คำนวณ conversion rate เมื่อยอดขายเป็น 0 |
| `numeric_value_out_of_range` | 22003 | ตัวเลขเกินขอบเขตของชนิดข้อมูล | คำนวณราคารวมเกิน `NUMERIC(10,2)` |
| `no_data_found` | P0002 | คาดหวังว่าจะมีข้อมูลแต่ไม่พบ (เช่นใน `SELECT INTO STRICT`) | ค้นหาสินค้าด้วย product_id ที่ไม่มีอยู่ |
| `too_many_rows` | P0003 | `SELECT INTO STRICT` ได้ผลลัพธ์มากกว่า 1 แถว | ใช้เงื่อนไขที่ไม่ unique พอในการดึงค่าเดี่ยว |
| `invalid_text_representation` | 22P02 | แปลงชนิดข้อมูลไม่สำเร็จ (เช่น cast string เป็น INTEGER) | รับ input จาก dynamic SQL ที่ format ผิด |
| `insufficient_privilege` | 42501 | สิทธิ์ไม่เพียงพอในการดำเนินการ | role ที่ไม่มีสิทธิ์ UPDATE ตาราง |

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1
เขียนฟังก์ชัน `count_up_to(p_max INTEGER)` ที่ใช้ `LOOP...EXIT WHEN` นับจาก 1 ถึง `p_max` และ `RAISE NOTICE` แสดงตัวเลขแต่ละตัว จากนั้นคืนค่าผลรวมทั้งหมด

<details>
<summary>เฉลย</summary>

```sql
CREATE OR REPLACE FUNCTION count_up_to(p_max INTEGER)
RETURNS INTEGER AS $$
DECLARE
    v_i   INTEGER := 1;
    v_sum INTEGER := 0;
BEGIN
    LOOP
        EXIT WHEN v_i > p_max;
        RAISE NOTICE 'ตัวเลข: %', v_i;
        v_sum := v_sum + v_i;
        v_i := v_i + 1;
    END LOOP;
    RETURN v_sum;
END;
$$ LANGUAGE plpgsql;

SELECT count_up_to(5);
-- ผลรวม = 15 (1+2+3+4+5)
```
</details>

### แบบฝึกหัดที่ 2
เขียนฟังก์ชัน `find_low_stock_products(p_threshold INTEGER)` ที่ใช้ `FOR...IN SELECT` วนดูสินค้าทั้งหมด และ `RAISE WARNING` เมื่อพบสินค้าที่ `stock_quantity` ต่ำกว่า `p_threshold`

<details>
<summary>เฉลย</summary>

```sql
CREATE OR REPLACE FUNCTION find_low_stock_products(p_threshold INTEGER)
RETURNS INTEGER AS $$
DECLARE
    v_rec RECORD;
    v_count INTEGER := 0;
BEGIN
    FOR v_rec IN SELECT product_id, product_name, stock_quantity FROM products ORDER BY stock_quantity
    LOOP
        IF v_rec.stock_quantity < p_threshold THEN
            RAISE WARNING 'สต็อกต่ำ: % (product_id=%) เหลือ % ชิ้น',
                v_rec.product_name, v_rec.product_id, v_rec.stock_quantity;
            v_count := v_count + 1;
        END IF;
    END LOOP;
    RETURN v_count;
END;
$$ LANGUAGE plpgsql;

SELECT find_low_stock_products(50);
```
</details>

### แบบฝึกหัดที่ 3
เขียนฟังก์ชัน `sum_array(p_values NUMERIC[])` ที่ใช้ `FOREACH` วนรวมค่าทั้งหมดใน array แล้วคืนค่าผลรวมและค่าเฉลี่ยเป็น RECORD (สร้าง OUT parameters หรือ return type ตามความเหมาะสม)

<details>
<summary>เฉลย</summary>

```sql
CREATE OR REPLACE FUNCTION sum_array(p_values NUMERIC[])
RETURNS TABLE(total NUMERIC, average NUMERIC, item_count INTEGER) AS $$
DECLARE
    v_val NUMERIC;
    v_sum NUMERIC := 0;
    v_count INTEGER := 0;
BEGIN
    FOREACH v_val IN ARRAY p_values LOOP
        v_sum := v_sum + v_val;
        v_count := v_count + 1;
    END LOOP;

    RETURN QUERY SELECT v_sum, round(v_sum / NULLIF(v_count, 0), 2), v_count;
END;
$$ LANGUAGE plpgsql;

SELECT * FROM sum_array(ARRAY[100, 250, 75.5, 400]);
```
</details>

### แบบฝึกหัดที่ 4
เขียนฟังก์ชัน `find_matching_pair_amount(p_target NUMERIC)` ที่ใช้ nested loop พร้อม label เพื่อหาคำสั่งซื้อ 2 รายการ (จากตาราง `orders`) ที่ผลรวมยอดขาย (จาก `order_items`) เท่ากับ `p_target` พอดี และ `EXIT` ออกจาก loop ทั้งสองชั้นทันทีเมื่อเจอคำตอบแรก

<details>
<summary>เฉลย</summary>

```sql
CREATE OR REPLACE FUNCTION find_matching_pair_amount(p_target NUMERIC)
RETURNS TABLE(order_id_a INTEGER, order_id_b INTEGER, total_a NUMERIC, total_b NUMERIC) AS $$
DECLARE
    v_a RECORD;
    v_b RECORD;
    v_found BOOLEAN := false;
BEGIN
    <<outer_loop>>
    FOR v_a IN
        SELECT o.order_id, sum(oi.quantity * oi.unit_price) AS total
        FROM orders o JOIN order_items oi ON oi.order_id = o.order_id
        GROUP BY o.order_id
    LOOP
        <<inner_loop>>
        FOR v_b IN
            SELECT o.order_id, sum(oi.quantity * oi.unit_price) AS total
            FROM orders o JOIN order_items oi ON oi.order_id = o.order_id
            GROUP BY o.order_id
        LOOP
            CONTINUE inner_loop WHEN v_b.order_id <= v_a.order_id;

            IF v_a.total + v_b.total = p_target THEN
                order_id_a := v_a.order_id;
                order_id_b := v_b.order_id;
                total_a := v_a.total;
                total_b := v_b.total;
                v_found := true;
                RETURN NEXT;
                EXIT outer_loop;
            END IF;
        END LOOP inner_loop;
    END LOOP outer_loop;

    IF NOT v_found THEN
        RAISE NOTICE 'ไม่พบคู่คำสั่งซื้อที่ยอดรวมเท่ากับ %', p_target;
    END IF;
END;
$$ LANGUAGE plpgsql;

SELECT * FROM find_matching_pair_amount(3170);
```
</details>

### แบบฝึกหัดที่ 5
เขียนฟังก์ชัน `safe_get_customer_email(p_customer_id INTEGER)` ที่ใช้ `SELECT ... INTO STRICT` และดักจับทั้ง `no_data_found` และ `too_many_rows` แยกกัน คืนข้อความที่เหมาะสมในแต่ละกรณี

<details>
<summary>เฉลย</summary>

```sql
CREATE OR REPLACE FUNCTION safe_get_customer_email(p_customer_id INTEGER)
RETURNS TEXT AS $$
DECLARE
    v_email TEXT;
BEGIN
    SELECT email INTO STRICT v_email
    FROM customers
    WHERE customer_id = p_customer_id;

    RETURN v_email;

EXCEPTION
    WHEN no_data_found THEN
        RETURN format('ไม่พบลูกค้า customer_id = %s', p_customer_id);
    WHEN too_many_rows THEN
        RETURN format('พบข้อมูลลูกค้ามากกว่า 1 แถวสำหรับ customer_id = %s (ข้อมูลผิดปกติ)', p_customer_id);
END;
$$ LANGUAGE plpgsql;

SELECT safe_get_customer_email(1);
SELECT safe_get_customer_email(9999);
```
</details>

### แบบฝึกหัดที่ 6
เขียนฟังก์ชัน `raise_stock_alert(p_product_id INTEGER, p_min_stock INTEGER)` ที่ raise exception แบบกำหนด error code เองเป็น `'LOWST'` เมื่อสต็อกต่ำกว่าค่าขั้นต่ำ แล้วเขียน anonymous block (`DO $$ ... $$`) ทดสอบดักจับด้วย `WHEN SQLSTATE 'LOWST'`

<details>
<summary>เฉลย</summary>

```sql
CREATE OR REPLACE FUNCTION raise_stock_alert(p_product_id INTEGER, p_min_stock INTEGER)
RETURNS VOID AS $$
DECLARE
    v_stock INTEGER;
BEGIN
    SELECT stock_quantity INTO v_stock FROM products WHERE product_id = p_product_id;

    IF v_stock < p_min_stock THEN
        RAISE EXCEPTION 'สต็อกสินค้า % ต่ำกว่าขั้นต่ำ (มี % ต้องการอย่างน้อย %)',
            p_product_id, v_stock, p_min_stock
            USING ERRCODE = 'LOWST';
    END IF;

    RAISE NOTICE 'สต็อกเพียงพอ: % ชิ้น', v_stock;
END;
$$ LANGUAGE plpgsql;

DO $$
BEGIN
    BEGIN
        PERFORM raise_stock_alert(7, 100);  -- 4K Monitor มีสต็อก 25 < 100
    EXCEPTION
        WHEN SQLSTATE 'LOWST' THEN
            RAISE NOTICE 'ดักจับ LOWST สำเร็จ: %', SQLERRM;
    END;
END;
$$;
```
</details>

### แบบฝึกหัดที่ 7
เขียนฟังก์ชัน `count_rows_dynamic(p_table_name TEXT, p_where_column TEXT, p_where_value TEXT)` ที่ใช้ `format()` และ `USING` เพื่อนับจำนวนแถวในตารางใด ๆ ตามเงื่อนไข column = value โดยปลอดภัยจาก SQL Injection ทั้งในส่วนชื่อตาราง/คอลัมน์ และค่าเงื่อนไข

<details>
<summary>เฉลย</summary>

```sql
CREATE OR REPLACE FUNCTION count_rows_dynamic(
    p_table_name   TEXT,
    p_where_column TEXT,
    p_where_value  TEXT
)
RETURNS BIGINT AS $$
DECLARE
    v_sql   TEXT;
    v_count BIGINT;
BEGIN
    IF NOT EXISTS (
        SELECT 1 FROM information_schema.columns
        WHERE table_name = p_table_name AND column_name = p_where_column
    ) THEN
        RAISE EXCEPTION 'ตาราง % หรือคอลัมน์ % ไม่มีอยู่จริง', p_table_name, p_where_column;
    END IF;

    v_sql := format('SELECT count(*) FROM %I WHERE %I = $1', p_table_name, p_where_column);
    EXECUTE v_sql INTO v_count USING p_where_value;

    RETURN v_count;
END;
$$ LANGUAGE plpgsql;

SELECT count_rows_dynamic('products', 'category_id', '2');
SELECT count_rows_dynamic('customers', 'country', 'Thailand');
```
</details>

### แบบฝึกหัดที่ 8
เขียนฟังก์ชัน `export_products_summary()` ที่ใช้ explicit cursor (`DECLARE ... CURSOR FOR`) วนอ่านสินค้าทุกตัวที่ `is_active = true` เรียงตามราคาจากมากไปน้อย และสร้าง string สรุปแบบ `"ชื่อสินค้า: ราคา บาท"` คั่นด้วยขึ้นบรรทัดใหม่ คืนค่าเป็น TEXT ก้อนเดียว

<details>
<summary>เฉลย</summary>

```sql
CREATE OR REPLACE FUNCTION export_products_summary()
RETURNS TEXT AS $$
DECLARE
    v_cur CURSOR FOR
        SELECT product_name, unit_price
        FROM products
        WHERE is_active = true
        ORDER BY unit_price DESC;
    v_rec RECORD;
    v_result TEXT := '';
BEGIN
    OPEN v_cur;
    LOOP
        FETCH v_cur INTO v_rec;
        EXIT WHEN NOT FOUND;
        v_result := v_result || format('%s: %s บาท', v_rec.product_name, v_rec.unit_price) || E'\n';
    END LOOP;
    CLOSE v_cur;

    RETURN v_result;
END;
$$ LANGUAGE plpgsql;

SELECT export_products_summary();
```
</details>

### แบบฝึกหัดที่ 9
เขียนฟังก์ชัน `bulk_cancel_pending_orders(p_older_than_days INTEGER)` ที่วนอ่าน `orders` ที่ `status = 'pending'` และเก่ากว่าจำนวนวันที่กำหนด แล้วพยายาม `UPDATE` เป็น `'cancelled'` ทีละแถวภายใน sub-block ที่มี exception handling ของตัวเอง หากแถวใด update ไม่สำเร็จ (จำลองด้วยการ raise exception เอง) ให้บันทึก `RAISE WARNING` และไปแถวถัดไปโดยไม่หยุดทั้งฟังก์ชัน สุดท้ายคืนจำนวนแถวที่ยกเลิกสำเร็จ

<details>
<summary>เฉลย</summary>

```sql
CREATE OR REPLACE FUNCTION bulk_cancel_pending_orders(p_older_than_days INTEGER)
RETURNS INTEGER AS $$
DECLARE
    v_rec RECORD;
    v_success_count INTEGER := 0;
BEGIN
    FOR v_rec IN
        SELECT order_id, order_date
        FROM orders
        WHERE status = 'pending'
          AND order_date < now() - (p_older_than_days || ' days')::INTERVAL
        ORDER BY order_id
    LOOP
        BEGIN
            -- จำลองเงื่อนไขที่อาจทำให้ update ล้มเหลว: ห้ามยกเลิก order ที่มี order_id หาร 7 ลงตัว (ตัวอย่างสมมติ)
            IF v_rec.order_id % 7 = 0 THEN
                RAISE EXCEPTION 'order_id % ถูกล็อคไว้ไม่ให้ยกเลิกอัตโนมัติ', v_rec.order_id;
            END IF;

            UPDATE orders SET status = 'cancelled' WHERE order_id = v_rec.order_id;
            v_success_count := v_success_count + 1;
            RAISE NOTICE 'ยกเลิกสำเร็จ: order_id = %', v_rec.order_id;

        EXCEPTION
            WHEN OTHERS THEN
                RAISE WARNING 'ยกเลิกไม่สำเร็จ order_id = %: %', v_rec.order_id, SQLERRM;
        END;
    END LOOP;

    RETURN v_success_count;
END;
$$ LANGUAGE plpgsql;

SELECT bulk_cancel_pending_orders(0);
```
</details>

### แบบฝึกหัดที่ 10
รวมทุกเทคนิค: เขียนฟังก์ชัน `generic_batch_normalize(p_table_name TEXT, p_text_column TEXT)` ที่ใช้ cursor แบบ dynamic (`OPEN cursor FOR EXECUTE`) วนอ่านทุกแถวของตารางที่ระบุ แล้ว `UPDATE` ให้ค่าคอลัมน์ข้อความ (`p_text_column`) กลายเป็นตัวพิมพ์ใหญ่ทั้งหมด (`UPPER()`) โดยต้อง validate ชื่อตาราง/คอลัมน์ผ่าน whitelist จาก `information_schema` ก่อนเสมอ และครอบแต่ละแถวด้วย exception handling พร้อมบันทึกจำนวนที่สำเร็จ/ล้มเหลว

<details>
<summary>เฉลย</summary>

```sql
CREATE OR REPLACE FUNCTION generic_batch_normalize(p_table_name TEXT, p_text_column TEXT)
RETURNS TABLE(success_count INTEGER, fail_count INTEGER) AS $$
DECLARE
    v_pk_column TEXT;
    v_sql_select TEXT;
    v_sql_update TEXT;
    v_cur REFCURSOR;
    v_rec RECORD;
    v_ok INTEGER := 0;
    v_fail INTEGER := 0;
BEGIN
    -- Validate ว่าตาราง/คอลัมน์มีอยู่จริง (ป้องกัน SQL Injection ที่ identifier)
    IF NOT EXISTS (
        SELECT 1 FROM information_schema.columns
        WHERE table_name = p_table_name AND column_name = p_text_column
          AND data_type IN ('character varying', 'text', 'character')
    ) THEN
        RAISE EXCEPTION 'ตาราง % หรือคอลัมน์ข้อความ % ไม่ถูกต้อง', p_table_name, p_text_column;
    END IF;

    -- หา primary key column แบบ dynamic
    SELECT kcu.column_name INTO v_pk_column
    FROM information_schema.table_constraints tc
    JOIN information_schema.key_column_usage kcu
        ON tc.constraint_name = kcu.constraint_name
    WHERE tc.table_name = p_table_name AND tc.constraint_type = 'PRIMARY KEY'
    LIMIT 1;

    IF v_pk_column IS NULL THEN
        RAISE EXCEPTION 'ไม่พบ primary key ของตาราง %', p_table_name;
    END IF;

    v_sql_select := format('SELECT %I AS pk_val, %I AS text_val FROM %I ORDER BY %I',
        v_pk_column, p_text_column, p_table_name, v_pk_column);

    OPEN v_cur FOR EXECUTE v_sql_select;

    LOOP
        FETCH v_cur INTO v_rec;
        EXIT WHEN NOT FOUND;

        BEGIN
            v_sql_update := format('UPDATE %I SET %I = UPPER(%I) WHERE %I = %L',
                p_table_name, p_text_column, p_text_column, v_pk_column, v_rec.pk_val);

            EXECUTE v_sql_update;
            v_ok := v_ok + 1;

        EXCEPTION
            WHEN OTHERS THEN
                v_fail := v_fail + 1;
                RAISE WARNING 'ล้มเหลวที่ % = %: %', v_pk_column, v_rec.pk_val, SQLERRM;
        END;
    END LOOP;

    CLOSE v_cur;

    RAISE NOTICE 'สรุป: สำเร็จ % แถว, ล้มเหลว % แถว', v_ok, v_fail;
    RETURN QUERY SELECT v_ok, v_fail;
END;
$$ LANGUAGE plpgsql;

-- ตัวอย่างการใช้งาน (ทดสอบกับ suppliers.country)
SELECT * FROM generic_batch_normalize('suppliers', 'country');
SELECT supplier_name, country FROM suppliers LIMIT 5;
```
</details>

---

## บทถัดไป

เรียนรู้เรื่อง Trigger ขั้นสูงต่อได้ที่ **[Part 048: Triggers](./part-048-triggers.md)**
