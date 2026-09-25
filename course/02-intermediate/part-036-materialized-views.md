# Part 036: Materialized Views และการ REFRESH

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับกลาง | Part 036

---

## เป้าหมายการเรียนรู้

เมื่อเรียนจบบทนี้ ผู้เรียนจะสามารถ:

1. อธิบายความแตกต่างระหว่าง **View**, **Materialized View** และ **Table** ทั้งในแง่การเก็บข้อมูลและ performance
2. สร้างและ query Materialized View ด้วย `CREATE MATERIALIZED VIEW` ได้อย่างถูกต้อง
3. วิเคราะห์ trade-off ระหว่างความสดของข้อมูล (data freshness) กับความเร็วในการ query เพื่อตัดสินใจว่าเมื่อไหร่ควรใช้ Materialized View
4. ใช้ `REFRESH MATERIALIZED VIEW` และ `REFRESH MATERIALIZED VIEW CONCURRENTLY` พร้อมเข้าใจผลกระทบเรื่อง lock
5. สร้าง index บน Materialized View เพื่อเร่งความเร็วการ query ต่อ
6. วางแผน schedule การ refresh อัตโนมัติด้วย `pg_cron` หรือ external scheduler
7. เข้าใจแนวคิดเบื้องต้นของ incremental refresh และข้อจำกัดของ PostgreSQL ในเรื่องนี้
8. ตรวจสอบขนาดและสถิติของ Materialized View และลบมันอย่างปลอดภัยด้วย `DROP MATERIALIZED VIEW`
9. ออกแบบชุด Materialized View สำหรับ dashboard ของระบบ e-commerce พร้อมกลยุทธ์ refresh ที่เหมาะสมกับสถานการณ์จริง

---

## เตรียมข้อมูล

บทนี้ใช้ชุดข้อมูล e-commerce เดียวกันกับ Part 021–039 ทั้งหมด หากเคยสร้างตารางเหล่านี้ไว้แล้วจาก Part ก่อนหน้า สามารถข้ามไปยัง Step 351 ได้เลย แต่ถ้าต้องการรันบทนี้แบบแยกเดี่ยว ให้รันสคริปต์ด้านล่างเพื่อสร้างฐานข้อมูลตั้งต้น

```sql
-- ลบตารางเดิม (ถ้ามี) เพื่อให้ทดลองซ้ำได้สะอาด
DROP TABLE IF EXISTS payments, reviews, order_items, orders, employees,
    customers, products, suppliers, categories CASCADE;

-- ===================== ตารางหลัก =====================

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
    unit_price      NUMERIC(10,2) NOT NULL CHECK (unit_price >= 0),
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

CREATE TABLE employees (
    employee_id   SERIAL PRIMARY KEY,
    first_name    VARCHAR(60) NOT NULL,
    last_name     VARCHAR(60) NOT NULL,
    hire_date     DATE NOT NULL,
    manager_id    INTEGER REFERENCES employees(employee_id),
    department    VARCHAR(60)
);

CREATE TABLE orders (
    order_id      SERIAL PRIMARY KEY,
    customer_id   INTEGER REFERENCES customers(customer_id),
    employee_id   INTEGER REFERENCES employees(employee_id),
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

CREATE TABLE reviews (
    review_id     SERIAL PRIMARY KEY,
    product_id    INTEGER REFERENCES products(product_id),
    customer_id   INTEGER REFERENCES customers(customer_id),
    rating        INTEGER NOT NULL CHECK (rating BETWEEN 1 AND 5),
    review_text   TEXT,
    review_date   DATE NOT NULL DEFAULT CURRENT_DATE
);

CREATE TABLE payments (
    payment_id      SERIAL PRIMARY KEY,
    order_id        INTEGER REFERENCES orders(order_id),
    payment_date    TIMESTAMPTZ NOT NULL DEFAULT now(),
    amount          NUMERIC(10,2) NOT NULL,
    payment_method  VARCHAR(30)
);

-- ===================== ข้อมูลตัวอย่าง =====================

INSERT INTO categories (category_name, parent_category_id) VALUES
    ('อิเล็กทรอนิกส์', NULL),          -- 1
    ('คอมพิวเตอร์และแล็ปท็อป', 1),      -- 2
    ('โทรศัพท์มือถือ', 1),             -- 3
    ('เครื่องใช้ไฟฟ้าในบ้าน', NULL),     -- 4
    ('แฟชั่น', NULL),                 -- 5
    ('เสื้อผ้าผู้ชาย', 5),             -- 6
    ('เสื้อผ้าผู้หญิง', 5),            -- 7
    ('หนังสือ', NULL),                -- 8
    ('ของเล่นและงานอดิเรก', NULL),     -- 9
    ('กีฬาและกลางแจ้ง', NULL);        -- 10

INSERT INTO suppliers (supplier_name, country) VALUES
    ('Bangkok Tech Distribution', 'Thailand'),      -- 1
    ('Shenzhen Digital Co.', 'China'),              -- 2
    ('Osaka Electronics Ltd.', 'Japan'),            -- 3
    ('Seoul Gadget Trading', 'South Korea'),        -- 4
    ('Hanoi Textile Group', 'Vietnam'),             -- 5
    ('Chiang Mai Craft House', 'Thailand'),         -- 6
    ('Jakarta Home Goods', 'Indonesia'),             -- 7
    ('Berlin Sports Import', 'Germany'),             -- 8
    ('New York Book Traders', 'USA'),               -- 9
    ('Singapore Toy Hub', 'Singapore');             -- 10

INSERT INTO products (product_name, category_id, supplier_id, unit_price, stock_quantity, is_active) VALUES
    ('โน้ตบุ๊ก UltraSlim 14"', 2, 1, 24900.00, 45, true),
    ('เมาส์ไร้สาย ErgoClick', 2, 2, 590.00, 320, true),
    ('คีย์บอร์ดกลไก TypeMaster', 2, 2, 1890.00, 150, true),
    ('สมาร์ทโฟน NovaPhone X12', 3, 4, 18900.00, 80, true),
    ('เคสกันกระแทก NovaPhone X12', 3, 4, 350.00, 500, true),
    ('หูฟังไร้สาย SoundWave Pro', 1, 3, 3290.00, 210, true),
    ('พัดลมไอเย็น CoolBreeze', 4, 7, 2590.00, 60, true),
    ('หม้อทอดไร้น้ำมัน AirCook 5L', 4, 7, 2190.00, 95, true),
    ('เสื้อยืดผ้าคอตตอน Basic', 6, 5, 259.00, 800, true),
    ('กางเกงยีนส์ Slim Fit', 6, 5, 890.00, 400, true),
    ('เดรสลายดอกไม้ Summer', 7, 5, 690.00, 260, true),
    ('รองเท้าผ้าใบ RunFast', 10, 8, 1990.00, 180, true),
    ('นิยายวิทยาศาสตร์ Starbound', 8, 9, 350.00, 120, true),
    ('หนังสือสอนเขียนโปรแกรม SQL Mastery', 8, 9, 590.00, 75, true),
    ('ตุ๊กตาหมี TeddyLove', 9, 10, 450.00, 300, true),
    ('ชุดตัวต่อ BrickWorld 500 ชิ้น', 9, 10, 1290.00, 140, true),
    ('เต็นท์แคมป์ปิ้ง 4 คน', 10, 8, 3990.00, 40, true),
    ('จักรยานเสือภูเขา TrailBlazer', 10, 8, 12900.00, 25, true),
    ('เครื่องปั่นน้ำผลไม้ BlendMax', 4, 7, 1590.00, 110, true),
    ('แท็บเล็ต TabPro 11"', 2, 1, 15900.00, 55, false);

INSERT INTO customers (first_name, last_name, email, country, signup_date) VALUES
    ('สมชาย', 'ใจดี', 'somchai.j@example.com', 'Thailand', '2023-01-15'),
    ('สมหญิง', 'รักเรียน', 'somying.r@example.com', 'Thailand', '2023-02-20'),
    ('วิชัย', 'มั่งมี', 'wichai.m@example.com', 'Thailand', '2023-03-05'),
    ('Nguyen', 'Van An', 'nguyen.an@example.com', 'Vietnam', '2023-03-18'),
    ('Suzuki', 'Haruto', 'suzuki.h@example.com', 'Japan', '2023-04-02'),
    ('Kim', 'Minjun', 'kim.minjun@example.com', 'South Korea', '2023-04-22'),
    ('ปรียา', 'สุขใจ', 'preeya.s@example.com', 'Thailand', '2023-05-10'),
    ('John', 'Smith', 'john.smith@example.com', 'USA', '2023-05-28'),
    ('Maria', 'Garcia', 'maria.g@example.com', 'Spain', '2023-06-14'),
    ('อนุชา', 'พงษ์ไพร', 'anucha.p@example.com', 'Thailand', '2023-07-01'),
    ('Li', 'Wei', 'li.wei@example.com', 'China', '2023-07-19'),
    ('นภัสสร', 'แสงจันทร์', 'napassorn.s@example.com', 'Thailand', '2023-08-08'),
    ('David', 'Miller', 'david.m@example.com', 'UK', '2023-08-30'),
    ('ศิริพร', 'บุญมาก', 'siriporn.b@example.com', 'Thailand', '2023-09-12'),
    ('Tanaka', 'Yui', 'tanaka.yui@example.com', 'Japan', '2023-10-05');

INSERT INTO employees (first_name, last_name, hire_date, manager_id, department) VALUES
    ('ประสิทธิ์', 'ผู้บริหาร', '2020-01-10', NULL, 'Executive'),
    ('กัญญา', 'ฝ่ายขาย', '2020-06-01', 1, 'Sales'),
    ('ธนกร', 'ฝ่ายขาย', '2021-02-15', 2, 'Sales'),
    ('อัจฉรา', 'ฝ่ายขาย', '2021-08-20', 2, 'Sales'),
    ('วีระ', 'ฝ่ายคลังสินค้า', '2021-03-10', 1, 'Warehouse'),
    ('ชลธิชา', 'ฝ่ายบริการลูกค้า', '2022-01-05', 1, 'Support'),
    ('ปิยะ', 'ฝ่ายขาย', '2022-05-18', 2, 'Sales'),
    ('รัตนา', 'ฝ่ายบริการลูกค้า', '2022-09-01', 6, 'Support');

INSERT INTO orders (customer_id, employee_id, order_date, status, ship_country) VALUES
    (1, 2, '2024-01-05 10:15:00+07', 'completed', 'Thailand'),
    (2, 3, '2024-01-08 14:30:00+07', 'completed', 'Thailand'),
    (3, 2, '2024-01-12 09:00:00+07', 'completed', 'Thailand'),
    (1, 4, '2024-01-20 16:45:00+07', 'completed', 'Thailand'),
    (4, 3, '2024-02-02 11:20:00+07', 'completed', 'Vietnam'),
    (5, 2, '2024-02-10 13:00:00+07', 'completed', 'Japan'),
    (6, 7, '2024-02-14 15:10:00+07', 'cancelled', 'South Korea'),
    (7, 4, '2024-02-25 10:30:00+07', 'completed', 'Thailand'),
    (2, 3, '2024-03-01 09:45:00+07', 'completed', 'Thailand'),
    (8, 2, '2024-03-06 12:00:00+07', 'completed', 'USA'),
    (9, 7, '2024-03-15 17:25:00+07', 'completed', 'Spain'),
    (3, 4, '2024-03-22 08:50:00+07', 'shipped', 'Thailand'),
    (10, 2, '2024-04-02 14:10:00+07', 'completed', 'Thailand'),
    (11, 3, '2024-04-09 10:00:00+07', 'completed', 'China'),
    (1, 7, '2024-04-18 16:20:00+07', 'completed', 'Thailand'),
    (12, 4, '2024-04-25 11:40:00+07', 'completed', 'Thailand'),
    (13, 2, '2024-05-03 09:15:00+07', 'completed', 'UK'),
    (5, 3, '2024-05-11 15:55:00+07', 'pending', 'Japan'),
    (14, 7, '2024-05-19 13:30:00+07', 'completed', 'Thailand'),
    (15, 4, '2024-05-27 10:05:00+07', 'completed', 'Japan'),
    (2, 2, '2024-06-04 14:50:00+07', 'completed', 'Thailand'),
    (6, 3, '2024-06-12 09:30:00+07', 'completed', 'South Korea'),
    (9, 7, '2024-06-20 16:00:00+07', 'shipped', 'Spain'),
    (3, 4, '2024-06-28 11:15:00+07', 'completed', 'Thailand');

INSERT INTO order_items (order_id, product_id, quantity, unit_price) VALUES
    (1, 1, 1, 24900.00), (1, 2, 2, 590.00),
    (2, 4, 1, 18900.00), (2, 5, 1, 350.00),
    (3, 9, 3, 259.00), (3, 10, 1, 890.00),
    (4, 6, 1, 3290.00), (4, 3, 1, 1890.00),
    (5, 12, 2, 1990.00),
    (6, 13, 4, 350.00), (6, 14, 1, 590.00),
    (7, 17, 1, 3990.00),
    (8, 1, 1, 24900.00), (8, 6, 1, 3290.00),
    (9, 15, 2, 450.00), (9, 16, 1, 1290.00),
    (10, 4, 1, 18900.00), (10, 5, 2, 350.00),
    (11, 11, 1, 690.00), (11, 9, 2, 259.00),
    (12, 18, 1, 12900.00),
    (13, 7, 1, 2590.00), (13, 19, 1, 1590.00),
    (14, 2, 3, 590.00), (14, 3, 1, 1890.00),
    (15, 8, 1, 2190.00),
    (16, 1, 1, 24900.00),
    (17, 10, 2, 890.00), (17, 11, 1, 690.00),
    (18, 6, 1, 3290.00),
    (19, 4, 1, 18900.00), (19, 5, 3, 350.00),
    (20, 12, 1, 1990.00), (20, 17, 1, 3990.00),
    (21, 14, 2, 590.00),
    (22, 9, 5, 259.00), (22, 10, 1, 890.00),
    (23, 16, 2, 1290.00),
    (24, 1, 1, 24900.00), (24, 2, 1, 590.00);

INSERT INTO reviews (product_id, customer_id, rating, review_text, review_date) VALUES
    (1, 1, 5, 'โน้ตบุ๊กแรงมาก ทำงานลื่นไหลดี', '2024-01-10'),
    (1, 8, 4, 'ดีแต่แบตอยู่ได้ไม่นานเท่าที่คิด', '2024-03-12'),
    (4, 2, 5, 'สมาร์ทโฟนกล้องสวย ถ่ายรูปคมชัด', '2024-01-15'),
    (4, 10, 3, 'ใช้งานโอเค แต่ร้อนเร็วตอนเล่นเกม', '2024-03-10'),
    (6, 1, 4, 'เสียงดีคุ้มราคา', '2024-01-25'),
    (6, 13, 5, 'หูฟังตัดเสียงรบกวนได้ดีมาก', '2024-05-08'),
    (9, 3, 5, 'เนื้อผ้านุ่มใส่สบาย', '2024-01-18'),
    (9, 11, 4, 'ไซซ์ตรงตามที่สั่ง', '2024-03-20'),
    (12, 4, 4, 'รองเท้าใส่วิ่งสบาย น้ำหนักเบา', '2024-02-05'),
    (17, 7, 3, 'เต็นท์กางง่ายแต่กันน้ำได้ไม่ดีเท่าที่หวัง', '2024-02-28'),
    (18, 12, 5, 'จักรยานปั่นลื่น เกียร์นุ่มนวล', '2024-04-28'),
    (14, 6, 5, 'หนังสือสอน SQL อธิบายเข้าใจง่ายมาก', '2024-02-16'),
    (2, 5, 4, 'เมาส์คลิกลื่น ตอบสนองไว', '2024-02-12'),
    (16, 9, 5, 'ตัวต่อสนุกมาก ลูกชอบมาก', '2024-03-17');

INSERT INTO payments (order_id, payment_date, amount, payment_method) VALUES
    (1, '2024-01-05 10:20:00+07', 26080.00, 'credit_card'),
    (2, '2024-01-08 14:35:00+07', 19250.00, 'credit_card'),
    (3, '2024-01-12 09:05:00+07', 1667.00, 'promptpay'),
    (4, '2024-01-20 16:50:00+07', 5180.00, 'credit_card'),
    (5, '2024-02-02 11:25:00+07', 3980.00, 'bank_transfer'),
    (6, '2024-02-10 13:05:00+07', 1990.00, 'credit_card'),
    (8, '2024-02-25 10:35:00+07', 28190.00, 'credit_card'),
    (9, '2024-03-01 09:50:00+07', 2190.00, 'promptpay'),
    (10, '2024-03-06 12:05:00+07', 19600.00, 'credit_card'),
    (11, '2024-03-15 17:30:00+07', 1208.00, 'paypal'),
    (12, '2024-03-22 08:55:00+07', 12900.00, 'credit_card'),
    (13, '2024-04-02 14:15:00+07', 4180.00, 'bank_transfer'),
    (14, '2024-04-09 10:05:00+07', 3660.00, 'credit_card'),
    (15, '2024-04-18 16:25:00+07', 2190.00, 'promptpay'),
    (16, '2024-04-25 11:45:00+07', 24900.00, 'credit_card'),
    (17, '2024-05-03 09:20:00+07', 2470.00, 'credit_card'),
    (19, '2024-05-19 13:35:00+07', 19950.00, 'bank_transfer'),
    (20, '2024-05-27 10:10:00+07', 5980.00, 'credit_card'),
    (21, '2024-06-04 14:55:00+07', 1180.00, 'promptpay'),
    (22, '2024-06-12 09:35:00+07', 2185.00, 'credit_card'),
    (24, '2024-06-28 11:20:00+07', 25490.00, 'credit_card');
```

> **หมายเหตุ:** คำสั่ง `INSERT` ด้านบนถูกออกแบบให้สอดคล้องกับ order_items เพื่อให้ยอดรวมใน `payments` ใกล้เคียงกับผลรวมของ `order_items` (มีปัดเศษ/ค่าขนส่งเล็กน้อยในบางออเดอร์ตามความสมจริง) ผู้เรียนสามารถเพิ่มข้อมูลเองได้เพื่อฝึกฝนเพิ่มเติม

---

## Step 351: Materialized View คืออะไร — เก็บข้อมูลจริง ต่างจาก View ธรรมดาอย่างไร

ใน Part ก่อนหน้า (Part 024 เรื่อง subqueries และแนวคิดเกี่ยวกับ view ที่อาจเคยพบผ่านมา) เราทราบว่า **View ธรรมดา (regular view)** คือ query ที่ถูก "บันทึกชื่อ" ไว้ในฐานข้อมูล เวลาเรา `SELECT * FROM my_view` ระบบจะไปรัน query เบื้องหลังใหม่ทุกครั้ง — พูดง่าย ๆ คือ view ไม่ได้เก็บข้อมูลจริง เป็นเพียง "คำสั่งที่บันทึกไว้" (a stored query)

**Materialized View** แตกต่างออกไปโดยสิ้นเชิง: มันคือ view ที่ **เก็บผลลัพธ์จริงลงดิสก์** เหมือนกับตารางปกติ เมื่อสร้างแล้ว ข้อมูลจะถูกคำนวณครั้งเดียวและเก็บไว้ (materialize แปลว่า "ทำให้เป็นรูปธรรม/จับต้องได้") การ query ครั้งต่อ ๆ ไปจึงเป็นการอ่านข้อมูลที่เก็บไว้แล้ว ไม่ต้องคำนวณ join หรือ aggregate ใหม่ทุกครั้ง

### เปรียบเทียบแนวคิดด้วยตัวอย่าง

ลองสร้าง view ธรรมดาสำหรับสรุปยอดขายต่อสินค้าก่อน:

```sql
CREATE VIEW product_sales_summary AS
SELECT
    p.product_id,
    p.product_name,
    COUNT(oi.order_item_id)      AS times_ordered,
    SUM(oi.quantity)             AS total_quantity_sold,
    SUM(oi.quantity * oi.unit_price) AS total_revenue
FROM products p
JOIN order_items oi ON oi.product_id = p.product_id
GROUP BY p.product_id, p.product_name;
```

ทุกครั้งที่เรารัน `SELECT * FROM product_sales_summary`, PostgreSQL จะไปสแกนตาราง `order_items` และ `products`, ทำ `JOIN`, `GROUP BY`, `SUM` ใหม่ทั้งหมด — ถ้าตาราง `order_items` มีข้อมูลหลักล้านแถว การ query แบบนี้จะช้าลงเรื่อย ๆ ตามขนาดข้อมูล

ตอนนี้ลองสร้างเป็น Materialized View แทน:

```sql
CREATE MATERIALIZED VIEW product_sales_summary_mv AS
SELECT
    p.product_id,
    p.product_name,
    COUNT(oi.order_item_id)      AS times_ordered,
    SUM(oi.quantity)             AS total_quantity_sold,
    SUM(oi.quantity * oi.unit_price) AS total_revenue
FROM products p
JOIN order_items oi ON oi.product_id = p.product_id
GROUP BY p.product_id, p.product_name;
```

เมื่อสร้างเสร็จ PostgreSQL จะรัน query นี้ **ครั้งเดียว** แล้วเก็บผลลัพธ์ไว้ในโครงสร้างคล้ายตาราง การ `SELECT` จาก `product_sales_summary_mv` หลังจากนี้จะเร็วมาก เพราะเป็นการอ่านข้อมูลที่ materialize ไว้แล้ว ไม่ต้องคำนวณใหม่

```sql
SELECT * FROM product_sales_summary_mv ORDER BY total_revenue DESC LIMIT 5;
```

ผลลัพธ์ตัวอย่าง:

```
 product_id |        product_name         | times_ordered | total_quantity_sold | total_revenue
------------+------------------------------+----------------+----------------------+---------------
          1 | โน้ตบุ๊ก UltraSlim 14"        |              4 |                    4 |     99600.00
          4 | สมาร์ทโฟน NovaPhone X12      |              3 |                    3 |     56700.00
         18 | จักรยานเสือภูเขา TrailBlazer  |              1 |                    1 |     12900.00
         17 | เต็นท์แคมป์ปิ้ง 4 คน          |              2 |                    2 |      7980.00
          6 | หูฟังไร้สาย SoundWave Pro    |              3 |                    3 |      9870.00
(5 rows)
```

### จุดสำคัญที่ต้องเข้าใจ

| ประเด็น | View ธรรมดา | Materialized View |
|---|---|---|
| การเก็บข้อมูล | ไม่เก็บ (เก็บแค่คำสั่ง query) | เก็บข้อมูลจริงบนดิสก์ |
| ความเร็วในการอ่าน | ช้า (query ใหม่ทุกครั้ง) | เร็วมาก (อ่านข้อมูลที่เก็บไว้) |
| ความสดของข้อมูล | สดเสมอ (real-time) | อาจไม่สด ต้อง `REFRESH` เพื่ออัปเดต |
| พื้นที่จัดเก็บ | แทบไม่ใช้พื้นที่เพิ่ม | ใช้พื้นที่เท่าผลลัพธ์ของ query |
| การสร้าง index | สร้างไม่ได้โดยตรง | สร้าง index ได้เหมือนตาราง |

ลองตรวจสอบด้วย `\d` ใน `psql` จะเห็นว่า Materialized View มีสถานะคล้ายตารางมากกว่า view:

```sql
\d product_sales_summary_mv
```

```
                     Materialized view "public.product_sales_summary_mv"
       Column        |         Type          | Collation | Nullable | Default
----------------------+------------------------+-----------+----------+---------
 product_id           | integer                |           |          |
 product_name         | character varying(150) |           |          |
 times_ordered        | bigint                 |           |          |
 total_quantity_sold  | bigint                 |           |          |
 total_revenue        | numeric                |           |          |
```

สังเกตว่าหัวตารางระบุว่าเป็น **"Materialized view"** ไม่ใช่ "Table" หรือ "View" — เป็นวัตถุ (object) ประเภทที่สามในฐานข้อมูล ที่ผสมผสานคุณสมบัติของทั้งสองอย่างเข้าด้วยกัน

---

## Step 352: CREATE MATERIALIZED VIEW syntax และการ query เหมือน table ปกติ

Syntax พื้นฐานของ `CREATE MATERIALIZED VIEW` มีรูปแบบดังนี้:

```sql
CREATE MATERIALIZED VIEW [IF NOT EXISTS] view_name
[(column_name [, ...])]
[WITH (storage_parameter [= value] [, ...])]
[TABLESPACE tablespace_name]
AS query
[WITH [NO] DATA];
```

### ตัวอย่างการสร้างที่ใช้บ่อย

**1. สร้างแบบพื้นฐาน** (จะรัน query และเติมข้อมูลทันที เพราะ default คือ `WITH DATA`):

```sql
CREATE MATERIALIZED VIEW monthly_revenue_mv AS
SELECT
    date_trunc('month', o.order_date)::date AS revenue_month,
    COUNT(DISTINCT o.order_id)              AS order_count,
    SUM(oi.quantity * oi.unit_price)        AS total_revenue
FROM orders o
JOIN order_items oi ON oi.order_id = o.order_id
WHERE o.status = 'completed'
GROUP BY date_trunc('month', o.order_date)
ORDER BY revenue_month;
```

```sql
SELECT * FROM monthly_revenue_mv;
```

```
 revenue_month | order_count | total_revenue
---------------+-------------+---------------
 2024-01-01    |           4 |     51837.00
 2024-02-01    |           3 |     37360.00
 2024-03-01    |           4 |     35948.00
 2024-04-01    |           4 |     45830.00
 2024-05-01    |           2 |      6470.00
 2024-06-01    |           3 |     29735.00
(6 rows)
```

**2. สร้างแบบไม่เติมข้อมูลทันที** ด้วย `WITH NO DATA` — มีประโยชน์เมื่อต้องการสร้างโครงสร้างก่อน แล้วค่อย refresh ทีหลัง (เช่น ตอน deploy ระบบใหม่ที่ยังไม่อยากให้รัน query หนักทันที):

```sql
CREATE MATERIALIZED VIEW customer_lifetime_value_mv AS
SELECT
    c.customer_id,
    c.first_name || ' ' || c.last_name AS customer_name,
    COUNT(DISTINCT o.order_id)         AS total_orders,
    COALESCE(SUM(oi.quantity * oi.unit_price), 0) AS lifetime_value
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.customer_id AND o.status = 'completed'
LEFT JOIN order_items oi ON oi.order_id = o.order_id
GROUP BY c.customer_id, c.first_name, c.last_name
WITH NO DATA;
```

หากลองสอบถามทันทีหลังสร้างแบบ `WITH NO DATA` จะเกิด error:

```sql
SELECT * FROM customer_lifetime_value_mv;
```

```
ERROR:  materialized view "customer_lifetime_value_mv" has not been populated
HINT:  Use the REFRESH MATERIALIZED VIEW command.
```

ต้อง refresh ก่อนจึงจะ query ได้ (รายละเอียดใน Step 354):

```sql
REFRESH MATERIALIZED VIEW customer_lifetime_value_mv;

SELECT * FROM customer_lifetime_value_mv ORDER BY lifetime_value DESC LIMIT 5;
```

```
 customer_id | customer_name  | total_orders | lifetime_value
-------------+-----------------+--------------+-----------------
           1 | สมชาย ใจดี      |            3 |     56060.00
           2 | สมหญิง รักเรียน |            3 |      6519.00
           8 | John Smith      |            1 |     19600.00
          15 | Tanaka Yui      |            1 |      5980.00
          12 | นภัสสร แสงจันทร์|            1 |     12900.00
(5 rows)
```

### Query เหมือน table ปกติ

สิ่งสำคัญคือ Materialized View สามารถถูกใช้ใน query ได้เหมือนกับตารางทุกประการ — `JOIN` กับตารางอื่น, ใส่ใน `WHERE`, ใช้ใน subquery หรือแม้แต่ `JOIN` กับ Materialized View อีกตัว:

```sql
SELECT
    clv.customer_name,
    clv.lifetime_value,
    c.country
FROM customer_lifetime_value_mv clv
JOIN customers c ON c.customer_id = clv.customer_id
WHERE clv.lifetime_value > 10000
ORDER BY clv.lifetime_value DESC;
```

```
 customer_name  | lifetime_value | country
-----------------+-----------------+----------
 สมชาย ใจดี      |     56060.00    | Thailand
 John Smith      |     19600.00    | USA
 นภัสสร แสงจันทร์|     12900.00    | Thailand
(3 rows)
```

**ข้อจำกัดที่ต้องจำ:** Materialized View เป็น **read-only** — ไม่สามารถ `INSERT`, `UPDATE`, `DELETE` ลงไปตรง ๆ ได้ (ต่างจาก view ธรรมดาบางแบบที่ update ได้ในเงื่อนไขจำกัด) วิธีเดียวที่จะเปลี่ยนข้อมูลใน Materialized View คือการ `REFRESH` เท่านั้น

```sql
INSERT INTO customer_lifetime_value_mv (customer_id, customer_name, total_orders, lifetime_value)
VALUES (999, 'Test User', 0, 0);
```

```
ERROR:  cannot change materialized view "customer_lifetime_value_mv"
```

---

## Step 353: เมื่อไหร่ควรใช้ Materialized View — trade-off ระหว่างความสดของข้อมูลกับ performance

การตัดสินใจใช้ Materialized View คือการแลกเปลี่ยน (trade-off) ระหว่างสองสิ่ง:

- **Performance**: query เร็วขึ้นมาก เพราะไม่ต้องคำนวณ join/aggregate ซ้ำทุกครั้ง
- **Freshness**: ข้อมูลอาจ "เก่า" (stale) ไม่ตรงกับข้อมูลจริงในตารางต้นทาง ณ ขณะนั้น จนกว่าจะมีการ `REFRESH`

### เมื่อไหร่ควรใช้

Materialized View เหมาะกับสถานการณ์ที่มีลักษณะดังนี้:

1. **Query ซับซ้อนและหนัก** — มี `JOIN` หลายตาราง, `GROUP BY`, aggregate function จำนวนมาก ที่ใช้เวลารันนาน (หลักวินาทีถึงหลักนาที)
2. **ถูกเรียกใช้บ่อย** — เช่น dashboard ที่ผู้ใช้หลายคนเปิดดูพร้อมกันตลอดเวลา ถ้าให้แต่ละคน trigger query หนักทุกครั้งจะทำให้ฐานข้อมูลรับภาระสูงเกินจำเป็น
3. **ข้อมูลไม่จำเป็นต้อง real-time เป๊ะ** — เช่น รายงานยอดขายรายเดือน, สถิติ dashboard ผู้บริหาร ที่ข้อมูลล่าช้าไม่กี่นาทีถึงหลายชั่วโมงยอมรับได้
4. **ต้นทางข้อมูลเปลี่ยนแปลงไม่บ่อย** เมื่อเทียบกับความถี่ในการอ่าน (read-heavy, write-light)

### เมื่อไหร่ไม่ควรใช้

1. **ต้องการข้อมูล real-time เป๊ะ** เช่น ยอดคงเหลือสต็อกสินค้าที่ต้องแม่นยำตลอดเวลาเพื่อป้องกันการขายเกินสต็อก (oversell)
2. **ข้อมูลต้นทางเปลี่ยนแปลงบ่อยมาก** (เช่น ทุกวินาที) จนการ refresh ตามทันแทบเป็นไปไม่ได้ หรือ refresh บ่อยจนกลายเป็นภาระเท่า ๆ กับ query ตรงเอง
3. **Query เดิมเร็วอยู่แล้ว** (มี index ที่เหมาะสม, ข้อมูลน้อย) — การเพิ่ม Materialized View จะเป็นความซับซ้อนที่ไม่จำเป็น

### ตัวอย่างการเปรียบเทียบเวลา

ลองเปรียบเทียบเวลาในการ query ระหว่าง view ธรรมดา กับ Materialized View (ด้วยข้อมูลจริงจะเห็นความต่างชัดกว่านี้มาก แต่แนวคิดเดียวกัน):

```sql
EXPLAIN ANALYZE
SELECT * FROM product_sales_summary ORDER BY total_revenue DESC;
```

```
 Sort  (cost=45.23..45.48 rows=100 width=80) (actual time=0.412..0.415 rows=15 loops=1)
   Sort Key: (sum((oi.quantity * oi.unit_price))) DESC
   ->  HashAggregate  (cost=38.50..41.00 rows=100 width=80) (actual time=0.320..0.360 rows=15 loops=1)
         Group Key: p.product_id
         ->  Hash Join  (cost=12.00..35.00 rows=500 width=40) (actual time=0.080..0.220 rows=38 loops=1)
               ...
 Planning Time: 0.150 ms
 Execution Time: 0.480 ms
```

```sql
EXPLAIN ANALYZE
SELECT * FROM product_sales_summary_mv ORDER BY total_revenue DESC;
```

```
 Sort  (cost=1.20..1.24 rows=15 width=80) (actual time=0.025..0.027 rows=15 loops=1)
   Sort Key: total_revenue DESC
   ->  Seq Scan on product_sales_summary_mv  (cost=0.00..1.15 rows=15 width=80) (actual time=0.008..0.012 rows=15 loops=1)
 Planning Time: 0.060 ms
 Execution Time: 0.045 ms
```

ในข้อมูลตัวอย่างขนาดเล็กนี้ความต่างอาจดูไม่มาก แต่เมื่อข้อมูลจริงมีหลายล้านแถวและ query ต้อง join 4-5 ตารางพร้อม aggregate ความต่างจะเห็นชัดเจนมาก — Materialized View ตัดขั้นตอน `Hash Join` และ `HashAggregate` ออกไปเหลือแค่ `Seq Scan` (หรือ `Index Scan` ถ้ามี index) บนข้อมูลที่คำนวณไว้แล้ว

### หลักคิดสรุป

> **ถาม 3 คำถามก่อนตัดสินใจ:**
> 1. Query นี้หนักและช้าจริงหรือไม่ (วัดด้วย `EXPLAIN ANALYZE`)?
> 2. ผู้ใช้ยอมรับข้อมูลที่ล่าช้าได้กี่นาที/ชั่วโมง?
> 3. มีวิธี refresh ที่ไม่กระทบผู้ใช้งาน (เช่น `CONCURRENTLY`) หรือไม่?
>
> ถ้าตอบได้ครบทั้ง 3 ข้อและคำตอบสนับสนุนการแลกเปลี่ยน (trade-off) นี้ — Materialized View คือคำตอบที่เหมาะสม

---

## Step 354: REFRESH MATERIALIZED VIEW — การอัปเดตข้อมูลด้วยตนเอง, ผลกระทบ (lock ตารางระหว่าง refresh)

เมื่อข้อมูลในตารางต้นทางเปลี่ยนแปลง (มี order ใหม่, payment ใหม่) Materialized View จะ **ไม่อัปเดตอัตโนมัติ** ต้องสั่ง `REFRESH MATERIALIZED VIEW` ด้วยตนเอง (หรือผ่าน scheduler)

### Syntax

```sql
REFRESH MATERIALIZED VIEW [CONCURRENTLY] view_name
[WITH [NO] DATA];
```

### ตัวอย่างการทดลอง

ลองเพิ่ม order ใหม่ แล้วดูว่า Materialized View ไม่เปลี่ยนตาม:

```sql
-- เพิ่ม order ใหม่
INSERT INTO orders (customer_id, employee_id, order_date, status, ship_country)
VALUES (7, 3, '2024-07-01 10:00:00+07', 'completed', 'Thailand')
RETURNING order_id;
-- สมมติได้ order_id = 25

INSERT INTO order_items (order_id, product_id, quantity, unit_price)
VALUES (25, 1, 2, 24900.00);

-- ตรวจสอบข้อมูลใน view ธรรมดา (สดเสมอ)
SELECT * FROM product_sales_summary WHERE product_id = 1;
```

```
 product_id |     product_name      | times_ordered | total_quantity_sold | total_revenue
------------+------------------------+----------------+----------------------+---------------
          1 | โน้ตบุ๊ก UltraSlim 14" |              5 |                    6 |    149400.00
(1 row)
```

```sql
-- ตรวจสอบข้อมูลใน materialized view (ยังเป็นข้อมูลเก่า!)
SELECT * FROM product_sales_summary_mv WHERE product_id = 1;
```

```
 product_id |     product_name      | times_ordered | total_quantity_sold | total_revenue
------------+------------------------+----------------+----------------------+---------------
          1 | โน้ตบุ๊ก UltraSlim 14" |              4 |                    4 |     99600.00
(1 row)
```

จะเห็นว่า Materialized View ยังแสดง `times_ordered = 4` และ `total_revenue = 99600.00` ซึ่งเป็นข้อมูลเก่า ทั้งที่ตารางจริงมีข้อมูลใหม่แล้ว — นี่คือธรรมชาติของ Materialized View ที่ต้อง refresh เพื่ออัปเดต

```sql
REFRESH MATERIALIZED VIEW product_sales_summary_mv;

SELECT * FROM product_sales_summary_mv WHERE product_id = 1;
```

```
 product_id |     product_name      | times_ordered | total_quantity_sold | total_revenue
------------+------------------------+----------------+----------------------+---------------
          1 | โน้ตบุ๊ก UltraSlim 14" |              5 |                    6 |    149400.00
(1 row)
```

ตอนนี้ข้อมูลตรงกันแล้ว

### ผลกระทบสำคัญ: LOCK ระหว่าง Refresh

การ `REFRESH MATERIALIZED VIEW` แบบธรรมดา (ไม่ใส่ `CONCURRENTLY`) จะทำงานโดย:

1. สร้างชุดข้อมูลใหม่ทั้งหมดจาก query
2. ขอ **`ACCESS EXCLUSIVE LOCK`** บน Materialized View ระหว่างกระบวนการ
3. แทนที่ข้อมูลเก่าด้วยข้อมูลใหม่

`ACCESS EXCLUSIVE LOCK` เป็น lock ระดับสูงสุดใน PostgreSQL หมายความว่า **ระหว่าง refresh จะไม่มีใคร query Materialized View นี้ได้เลย** แม้แต่ `SELECT` ธรรมดาก็ต้องรอจนกว่า refresh จะเสร็จ

ลองจำลองสถานการณ์นี้ (แนวคิด ไม่จำเป็นต้องรันจริงเพื่อดูผล เพราะข้อมูลตัวอย่างมีขนาดเล็กมาก refresh จะเสร็จในเสี้ยววินาที):

```sql
-- session A: กำลัง refresh (สมมติ query ต้นทางหนักและใช้เวลานาน)
BEGIN;
REFRESH MATERIALIZED VIEW product_sales_summary_mv;
-- ระหว่างนี้ session B พยายาม SELECT จะต้อง "รอ" จนกว่า session A จะ COMMIT
COMMIT;
```

```sql
-- session B: รันพร้อมกันขณะ session A ยัง refresh ไม่เสร็จ
SELECT * FROM product_sales_summary_mv;
-- คำสั่งนี้จะ "ค้าง" (block) จนกว่า session A จะ refresh เสร็จและ commit
```

ตรวจสอบ lock ที่เกิดขึ้นได้ผ่าน `pg_locks`:

```sql
SELECT
    l.locktype,
    l.mode,
    l.granted,
    a.query,
    a.state
FROM pg_locks l
JOIN pg_stat_activity a ON a.pid = l.pid
WHERE l.relation = 'product_sales_summary_mv'::regclass;
```

```
 locktype |        mode          | granted |                          query                          | state
----------+-----------------------+---------+-----------------------------------------------------------+--------
 relation | AccessExclusiveLock  | t       | REFRESH MATERIALIZED VIEW product_sales_summary_mv;      | active
 relation | AccessShareLock      | f       | SELECT * FROM product_sales_summary_mv;                  | active
(2 rows)
```

จากผลลัพธ์: session ที่ต้องการแค่ `SELECT` (ขอ `AccessShareLock` ซึ่งเป็น lock ระดับต่ำสุด) ก็ยังต้อง "รอ" (`granted = f`) เพราะ `ACCESS EXCLUSIVE LOCK` block การเข้าถึงทุกรูปแบบ

**ผลกระทบในระบบจริง:** ถ้า Materialized View นี้ถูกใช้เป็น backend ของ dashboard ที่มีผู้ใช้เปิดดูตลอดเวลา การ refresh แบบธรรมดาจะทำให้ dashboard "ค้าง" ในช่วงเวลาที่ refresh — ถ้า query ต้นทางใช้เวลานาน (เช่น 30 วินาที) ผู้ใช้ทุกคนที่พยายามเข้าถึงในช่วงนั้นจะต้องรอ ซึ่งอาจไม่เป็นที่ยอมรับในระบบ production

นี่คือเหตุผลที่ PostgreSQL มี `REFRESH MATERIALIZED VIEW CONCURRENTLY` ซึ่งจะอธิบายใน Step ถัดไป

---

## Step 355: REFRESH MATERIALIZED VIEW CONCURRENTLY — refresh แบบไม่ lock การอ่าน, ข้อกำหนด (ต้องมี unique index)

`REFRESH MATERIALIZED VIEW CONCURRENTLY` ช่วยแก้ปัญหาเรื่อง lock โดยอนุญาตให้มีการ `SELECT` อ่านข้อมูลได้ตามปกติระหว่างกระบวนการ refresh — ไม่ block การอ่านเหมือนวิธีธรรมดา

### หลักการทำงานเบื้องหลัง

แทนที่จะล็อกและแทนที่ข้อมูลทั้งหมดทันที `CONCURRENTLY` จะ:

1. สร้าง**สำเนาชั่วคราว** ของผลลัพธ์ query ใหม่ (temporary table)
2. เปรียบเทียบข้อมูลเก่ากับข้อมูลใหม่แบบแถวต่อแถว (row-by-row diff) โดยอาศัย unique index
3. รัน `UPDATE`/`INSERT`/`DELETE` เฉพาะแถวที่เปลี่ยนแปลงจริงเท่านั้น (ไม่ใช่แทนที่ทั้งหมด)
4. ใช้ lock ระดับต่ำกว่า (`EXCLUSIVE LOCK` แทน `ACCESS EXCLUSIVE LOCK`) ซึ่งยังอนุญาตให้ `SELECT` อ่านได้ตามปกติ

### ข้อกำหนดสำคัญ: ต้องมี UNIQUE INDEX

เนื่องจากกระบวนการเปรียบเทียบแถวต้องอาศัยตัวระบุที่ไม่ซ้ำกัน (เพื่อรู้ว่าแถวไหนคือแถวเดียวกันระหว่างข้อมูลเก่ากับใหม่) จึง**บังคับ**ว่า Materialized View ต้องมี **UNIQUE INDEX อย่างน้อยหนึ่งตัว** ก่อนจึงจะใช้ `CONCURRENTLY` ได้

ลองทดสอบโดยไม่มี unique index ก่อน:

```sql
REFRESH MATERIALIZED VIEW CONCURRENTLY product_sales_summary_mv;
```

```
ERROR:  cannot refresh materialized view "public.product_sales_summary_mv" concurrently
HINT:  Create a unique index with no WHERE clause on one or more columns of the materialized view.
```

ต้องสร้าง unique index ก่อน:

```sql
CREATE UNIQUE INDEX idx_product_sales_summary_mv_pk
    ON product_sales_summary_mv (product_id);
```

ตอนนี้ลอง `CONCURRENTLY` อีกครั้ง:

```sql
REFRESH MATERIALIZED VIEW CONCURRENTLY product_sales_summary_mv;
```

```
REFRESH MATERIALIZED VIEW
```

สำเร็จ! ระหว่างกระบวนการนี้ session อื่นที่ทำ `SELECT` จาก `product_sales_summary_mv` จะไม่ถูก block

### เปรียบเทียบ lock mode

| วิธี | Lock Mode | บล็อกการอ่าน (SELECT)? | ต้องมี Unique Index? | ความเร็ว |
|---|---|---|---|---|
| `REFRESH MATERIALIZED VIEW` | ACCESS EXCLUSIVE | บล็อก | ไม่ต้อง | เร็วกว่า (rewrite ทั้งหมด) |
| `REFRESH MATERIALIZED VIEW CONCURRENTLY` | EXCLUSIVE | ไม่บล็อก | **ต้องมี** | ช้ากว่า (ต้อง diff ทีละแถว) |

### ข้อควรระวังเพิ่มเติม

1. **ใช้ทรัพยากรมากกว่า** เพราะต้องสร้างสำเนาชั่วคราวและเปรียบเทียบข้อมูล ใช้ CPU และดิสก์มากกว่าวิธีธรรมดา
2. **ใช้เวลานานกว่า** ในหลายกรณี เพราะกระบวนการ diff มีค่าใช้จ่ายเพิ่มขึ้น
3. **ไม่สามารถใช้ครั้งแรกที่สร้างด้วย `WITH NO DATA`** — ต้องมีข้อมูลอยู่แล้วอย่างน้อยหนึ่งครั้งก่อน (refresh ธรรมดาครั้งแรก แล้วค่อยใช้ `CONCURRENTLY` ในครั้งถัดไป):

```sql
CREATE MATERIALIZED VIEW test_concurrent_mv AS
SELECT product_id, product_name, unit_price FROM products
WITH NO DATA;

CREATE UNIQUE INDEX idx_test_concurrent_mv ON test_concurrent_mv (product_id);

-- ครั้งแรกต้อง refresh แบบไม่ CONCURRENTLY (เพราะยังไม่มีข้อมูล)
REFRESH MATERIALIZED VIEW CONCURRENTLY test_concurrent_mv;
```

```
ERROR:  CONCURRENTLY cannot be used when the materialized view is not populated
```

ต้องแก้เป็น:

```sql
REFRESH MATERIALIZED VIEW test_concurrent_mv;              -- ครั้งแรก ไม่ใส่ CONCURRENTLY
REFRESH MATERIALIZED VIEW CONCURRENTLY test_concurrent_mv; -- ครั้งต่อไปใช้ CONCURRENTLY ได้
```

```
REFRESH MATERIALIZED VIEW
REFRESH MATERIALIZED VIEW
```

### หลักปฏิบัติแนะนำ

> **แนวทางที่แนะนำสำหรับระบบ production:** สร้าง Materialized View ทุกตัวพร้อม unique index ตั้งแต่แรก และใช้ `CONCURRENTLY` เสมอสำหรับการ refresh หลังจากนั้น ยกเว้นในกรณีที่ต้องการความเร็วสูงสุดและยอมรับการ block ชั่วคราวได้ (เช่น รันตอนดึกที่ไม่มีผู้ใช้งาน)

---

## Step 356: Index บน Materialized View — เพิ่มความเร็วในการ query ต่อ

เนื่องจาก Materialized View เก็บข้อมูลจริงบนดิสก์เหมือนตาราง จึงสามารถสร้าง index ได้เหมือนตารางทุกประการ — ทั้ง B-tree, Hash, GIN, GiST ตามความเหมาะสมของ query ที่จะใช้งาน

### ทำไมต้องมี index บน Materialized View

แม้ Materialized View จะเร็วกว่า view ธรรมดาอยู่แล้ว (เพราะไม่ต้องคำนวณ join/aggregate ใหม่) แต่ถ้า Materialized View มีข้อมูลจำนวนมากและมีการ query แบบกรอง (`WHERE`) หรือ `JOIN` บ่อย ๆ การมี index ที่เหมาะสมจะช่วยให้เร็วขึ้นไปอีกขั้น

### ตัวอย่างการสร้าง index หลายแบบ

ลองสร้าง Materialized View ที่ใหญ่และซับซ้อนขึ้นสำหรับสรุปข้อมูลลูกค้ารายเดือน:

```sql
CREATE MATERIALIZED VIEW customer_monthly_orders_mv AS
SELECT
    c.customer_id,
    c.first_name || ' ' || c.last_name AS customer_name,
    c.country,
    date_trunc('month', o.order_date)::date AS order_month,
    COUNT(o.order_id)                       AS order_count,
    SUM(oi.quantity * oi.unit_price)        AS month_total
FROM customers c
JOIN orders o ON o.customer_id = c.customer_id
JOIN order_items oi ON oi.order_id = o.order_id
WHERE o.status = 'completed'
GROUP BY c.customer_id, c.first_name, c.last_name, c.country,
         date_trunc('month', o.order_date);
```

```sql
SELECT * FROM customer_monthly_orders_mv ORDER BY order_month, customer_id LIMIT 5;
```

```
 customer_id | customer_name  | country  | order_month | order_count | month_total
-------------+-----------------+----------+-------------+--------------+-------------
           1 | สมชาย ใจดี      | Thailand | 2024-01-01  |            2 |    28137.00
           2 | สมหญิง รักเรียน | Thailand | 2024-01-01  |            1 |      777.00
           3 | วิชัย มั่งมี     | Thailand | 2024-01-01  |            1 |     1667.00
           4 | Nguyen Van An   | Vietnam  | 2024-02-01  |            1 |     3980.00
           5 | Suzuki Haruto   | Japan    | 2024-02-01  |            1 |     1990.00
(5 rows)
```

**1. Unique index** (จำเป็นสำหรับ `CONCURRENTLY` และช่วยป้องกันข้อมูลซ้ำ):

```sql
CREATE UNIQUE INDEX idx_cmo_mv_customer_month
    ON customer_monthly_orders_mv (customer_id, order_month);
```

**2. Index สำหรับการกรองตามช่วงเวลา** (ใช้บ่อยใน dashboard ที่กรองตามเดือน):

```sql
CREATE INDEX idx_cmo_mv_order_month
    ON customer_monthly_orders_mv (order_month);
```

**3. Index สำหรับการกรองตามประเทศ** (ใช้ query แบบแบ่งตามภูมิภาค):

```sql
CREATE INDEX idx_cmo_mv_country
    ON customer_monthly_orders_mv (country);
```

**4. Partial index** สำหรับกรณีเฉพาะ เช่น เดือนที่ยอดสูงกว่าค่าเฉลี่ย:

```sql
CREATE INDEX idx_cmo_mv_high_value
    ON customer_monthly_orders_mv (month_total)
    WHERE month_total > 5000;
```

### ทดสอบผลของ index ด้วย EXPLAIN

```sql
EXPLAIN ANALYZE
SELECT * FROM customer_monthly_orders_mv
WHERE order_month = '2024-01-01' AND country = 'Thailand';
```

ก่อนมี index ที่เหมาะสม (สมมติมีแค่ตารางเปล่า ๆ):

```
 Seq Scan on customer_monthly_orders_mv  (cost=0.00..2.20 rows=1 width=64)
                                          (actual time=0.015..0.020 rows=3 loops=1)
   Filter: (order_month = '2024-01-01' AND country = 'Thailand')
 Planning Time: 0.080 ms
 Execution Time: 0.035 ms
```

เมื่อข้อมูลมีขนาดใหญ่ขึ้นมาก (หลักแสน-ล้านแถว) planner จะเลือกใช้ index แทน `Seq Scan` โดยอัตโนมัติ:

```
 Bitmap Heap Scan on customer_monthly_orders_mv  (cost=4.30..12.55 rows=3 width=64)
   Recheck Cond: (order_month = '2024-01-01' AND country = 'Thailand')
   ->  Bitmap Index Scan on idx_cmo_mv_order_month  (cost=0.00..4.30 rows=3 width=0)
         Index Cond: (order_month = '2024-01-01')
 Planning Time: 0.120 ms
 Execution Time: 0.045 ms
```

### ข้อควรทราบเรื่อง Statistics

หลังจาก `REFRESH` ควรพิจารณารัน `ANALYZE` เพื่ออัปเดตสถิติที่ query planner ใช้ตัดสินใจเลือกแผนการ query (แม้ว่า `REFRESH` มักจะอัปเดตสถิติให้อัตโนมัติในเวอร์ชันใหม่ ๆ แต่การรัน `ANALYZE` ซ้ำเพื่อความมั่นใจก็เป็นแนวปฏิบัติที่ดี):

```sql
REFRESH MATERIALIZED VIEW CONCURRENTLY customer_monthly_orders_mv;
ANALYZE customer_monthly_orders_mv;
```

```
REFRESH MATERIALIZED VIEW
ANALYZE
```

---

## Step 357: การ schedule refresh อัตโนมัติด้วย pg_cron หรือ external scheduler

Materialized View ที่ต้อง refresh ด้วยมือทุกครั้งไม่เหมาะกับระบบ production — เราต้องการให้ข้อมูล refresh อัตโนมัติตามรอบเวลาที่กำหนด มีสองแนวทางหลัก

### แนวทางที่ 1: pg_cron (extension ภายในฐานข้อมูล)

[`pg_cron`](https://github.com/citusdata/pg_cron) เป็น extension ยอดนิยมที่ทำให้ PostgreSQL รันคำสั่งตามตารางเวลาแบบ cron ได้จากภายในฐานข้อมูลเอง โดยไม่ต้องพึ่งเครื่องมือภายนอก

**การติดตั้ง** (ต้องมีสิทธิ์ superuser และตั้งค่าใน `postgresql.conf`):

```sql
-- ใน postgresql.conf ต้องเพิ่ม:
-- shared_preload_libraries = 'pg_cron'
-- แล้ว restart PostgreSQL ก่อน จากนั้นจึงสร้าง extension

CREATE EXTENSION IF NOT EXISTS pg_cron;
```

**การตั้ง schedule สำหรับ refresh Materialized View:**

```sql
-- refresh ทุกชั่วโมง ที่นาทีที่ 0
SELECT cron.schedule(
    'refresh-product-sales-summary',   -- ชื่อ job
    '0 * * * *',                       -- cron expression: ทุกชั่วโมง
    'REFRESH MATERIALIZED VIEW CONCURRENTLY product_sales_summary_mv'
);
```

```
 schedule
----------
        1
(1 row)
```

**ตัวอย่าง cron expression ที่ใช้บ่อย:**

| Cron Expression | ความหมาย |
|---|---|
| `*/15 * * * *` | ทุก 15 นาที |
| `0 * * * *` | ทุกชั่วโมง (นาทีที่ 0) |
| `0 */6 * * *` | ทุก 6 ชั่วโมง |
| `0 2 * * *` | ทุกวัน เวลา 02:00 น. (ช่วงที่มีคนใช้งานน้อย) |
| `0 0 * * 0` | ทุกสัปดาห์ วันอาทิตย์เที่ยงคืน |
| `0 1 1 * *` | ทุกวันที่ 1 ของเดือน เวลา 01:00 น. |

**ตั้งหลาย job พร้อมกัน** สำหรับ Materialized View หลายตัวที่มีความต้องการความสดของข้อมูลต่างกัน:

```sql
-- dashboard ยอดขายรายวัน ต้องการความสดสูง -> refresh ทุก 15 นาที
SELECT cron.schedule(
    'refresh-daily-dashboard',
    '*/15 * * * *',
    'REFRESH MATERIALIZED VIEW CONCURRENTLY customer_monthly_orders_mv'
);

-- รายงานยอดขายรายเดือนสำหรับผู้บริหาร -> refresh วันละครั้งตอนดึกพอ
SELECT cron.schedule(
    'refresh-monthly-revenue',
    '0 2 * * *',
    'REFRESH MATERIALIZED VIEW monthly_revenue_mv'
);
```

**ตรวจสอบ job ที่ตั้งไว้:**

```sql
SELECT jobid, jobname, schedule, command, active
FROM cron.job;
```

```
 jobid |           jobname            |   schedule   |                            command                                | active
-------+--------------------------------+--------------+---------------------------------------------------------------------+--------
     1 | refresh-product-sales-summary | 0 * * * *    | REFRESH MATERIALIZED VIEW CONCURRENTLY product_sales_summary_mv     | t
     2 | refresh-daily-dashboard        | */15 * * * * | REFRESH MATERIALIZED VIEW CONCURRENTLY customer_monthly_orders_mv   | t
     3 | refresh-monthly-revenue        | 0 2 * * *    | REFRESH MATERIALIZED VIEW monthly_revenue_mv                        | t
(3 rows)
```

**ดูประวัติการรัน (สำเร็จ/ล้มเหลว):**

```sql
SELECT jobid, status, return_message, start_time, end_time
FROM cron.job_run_details
ORDER BY start_time DESC
LIMIT 5;
```

```
 jobid | status  |      return_message      |         start_time         |          end_time
-------+---------+---------------------------+-----------------------------+-----------------------------
     1 | succeeded | REFRESH MATERIALIZED VIEW | 2024-07-01 10:00:00.123+07 | 2024-07-01 10:00:00.456+07
(1 row)
```

**ยกเลิก job** เมื่อไม่ต้องการแล้ว:

```sql
SELECT cron.unschedule('refresh-product-sales-summary');
```

```
 unschedule
------------
 t
(1 row)
```

### แนวทางที่ 2: External Scheduler

หากไม่สามารถติดตั้ง extension ได้ (เช่น managed database บางผู้ให้บริการที่ไม่รองรับ `pg_cron`) สามารถใช้เครื่องมือภายนอกแทน เช่น:

- **cron ของ OS (Linux)** — เรียก `psql` ผ่าน shell script
- **Airflow / Prefect / Dagster** — สำหรับ pipeline ที่ซับซ้อนมีหลายขั้นตอน
- **Kubernetes CronJob** — ในสภาพแวดล้อม containerized

**ตัวอย่าง shell script ที่เรียกผ่าน crontab ของ Linux:**

```bash
#!/bin/bash
# refresh_mv.sh
psql -h db.example.com -U app_user -d ecommerce_db \
    -c "REFRESH MATERIALIZED VIEW CONCURRENTLY product_sales_summary_mv;"
```

**ตั้งใน crontab** (`crontab -e`):

```
# refresh ทุก 30 นาที
*/30 * * * * /path/to/refresh_mv.sh >> /var/log/refresh_mv.log 2>&1
```

### ข้อดี-ข้อเสียของแต่ละแนวทาง

| แนวทาง | ข้อดี | ข้อเสีย |
|---|---|---|
| `pg_cron` | ตั้งค่าง่าย, ไม่ต้องพึ่งระบบภายนอก, ดู log ผ่าน SQL ได้ | ต้องมีสิทธิ์ superuser ติดตั้ง, ไม่ทุก managed service รองรับ |
| External scheduler | ยืดหยุ่นสูง, รวมกับ pipeline อื่นได้, ทำงานร่วมกับ multi-step logic ได้ | ซับซ้อนกว่า, ต้องดูแลระบบแยกต่างหาก, ต้องจัดการ credential เอง |

---

## Step 358: Materialized View แบบ incremental refresh — แนวคิดเบื้องต้น

### ข้อจำกัดของ PostgreSQL

PostgreSQL **ไม่รองรับ incremental refresh แบบ native** สำหรับ Materialized View กล่าวคือ ทุกครั้งที่ `REFRESH` (ไม่ว่าจะใช้ `CONCURRENTLY` หรือไม่) ระบบจะต้อง **รัน query ต้นทางใหม่ทั้งหมด** เพื่อคำนวณผลลัพธ์ใหม่ทั้งชุด — ไม่มีกลไกที่จะ "อัปเดตเฉพาะส่วนที่เปลี่ยน" โดยอัตโนมัติ

ซึ่งแตกต่างจากระบบฐานข้อมูลบางตัว (เช่น Oracle ที่มี Materialized View Log หรือ SQL Server ที่มี Indexed View พร้อม incremental maintenance) ที่รองรับการอัปเดตแบบ incremental โดยตรง

นี่หมายความว่า แม้ `CONCURRENTLY` จะช่วยเรื่อง lock ไม่ block การอ่าน แต่ **ค่าใช้จ่ายด้าน CPU/I/O ในการคำนวณ query ต้นทางใหม่ทั้งหมดยังคงเท่าเดิม** — ถ้าตารางต้นทางมีข้อมูลหลักสิบล้านแถว การ refresh แต่ละครั้งจะยังคงใช้เวลาและทรัพยากรมาก แม้จะมีแค่ไม่กี่แถวที่เปลี่ยนแปลงจริง

### แนวทาง Trick ที่ใช้กันในทางปฏิบัติ

เนื่องจากไม่มี native support จึงมีเทคนิค (trick) หลายแบบที่ทีมงานใช้แก้ปัญหานี้:

**1. แบ่ง Materialized View ตามช่วงเวลา (partitioned by time window)**

แทนที่จะมี Materialized View เดียวที่ครอบคลุมข้อมูลทั้งหมด ให้แยกเป็นส่วนที่ "นิ่งแล้ว" (historical, ไม่เปลี่ยนแปลง) กับส่วนที่ "ยังเปลี่ยนอยู่" (recent) แล้วรวมกันด้วย `UNION ALL`:

```sql
-- ส่วนข้อมูลเก่าที่ไม่เปลี่ยนแล้ว (ก่อนเดือนปัจจุบัน) -- refresh นาน ๆ ครั้งพอ
CREATE MATERIALIZED VIEW monthly_revenue_historical_mv AS
SELECT
    date_trunc('month', o.order_date)::date AS revenue_month,
    SUM(oi.quantity * oi.unit_price)        AS total_revenue
FROM orders o
JOIN order_items oi ON oi.order_id = o.order_id
WHERE o.status = 'completed'
  AND o.order_date < date_trunc('month', now())
GROUP BY date_trunc('month', o.order_date);

-- ส่วนเดือนปัจจุบันที่ยังเปลี่ยนแปลงอยู่ -- ใช้ view ธรรมดา (สดเสมอ) เพราะข้อมูลน้อยกว่ามาก
CREATE VIEW monthly_revenue_current_v AS
SELECT
    date_trunc('month', o.order_date)::date AS revenue_month,
    SUM(oi.quantity * oi.unit_price)        AS total_revenue
FROM orders o
JOIN order_items oi ON oi.order_id = o.order_id
WHERE o.status = 'completed'
  AND o.order_date >= date_trunc('month', now())
GROUP BY date_trunc('month', o.order_date);

-- รวมทั้งสองส่วนเข้าด้วยกัน
CREATE VIEW monthly_revenue_combined_v AS
SELECT * FROM monthly_revenue_historical_mv
UNION ALL
SELECT * FROM monthly_revenue_current_v;
```

ด้วยวิธีนี้ ส่วนข้อมูลเก่า (ปริมาณมากแต่นิ่งแล้ว) ไม่จำเป็นต้อง refresh บ่อย ในขณะที่ส่วนข้อมูลปัจจุบัน (ปริมาณน้อยเพราะเป็นแค่เดือนล่าสุด) ใช้ view ธรรมดาที่สดเสมอโดยไม่มีค่าใช้จ่ายสูงมาก

**2. ใช้ Trigger + Summary Table แทน Materialized View**

สำหรับกรณีที่ต้องการ incremental update จริง ๆ สามารถเขียน trigger บนตารางต้นทางเพื่ออัปเดตตาราง "summary" ด้วยตนเองแทนการใช้ Materialized View:

```sql
CREATE TABLE product_sales_running_total (
    product_id      INTEGER PRIMARY KEY REFERENCES products(product_id),
    total_quantity  INTEGER NOT NULL DEFAULT 0,
    total_revenue   NUMERIC(12,2) NOT NULL DEFAULT 0
);

CREATE OR REPLACE FUNCTION update_product_sales_running_total()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO product_sales_running_total (product_id, total_quantity, total_revenue)
    VALUES (NEW.product_id, NEW.quantity, NEW.quantity * NEW.unit_price)
    ON CONFLICT (product_id) DO UPDATE
        SET total_quantity = product_sales_running_total.total_quantity + NEW.quantity,
            total_revenue  = product_sales_running_total.total_revenue + (NEW.quantity * NEW.unit_price);
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_update_sales_running_total
    AFTER INSERT ON order_items
    FOR EACH ROW
    EXECUTE FUNCTION update_product_sales_running_total();
```

วิธีนี้ให้ผลแบบ "incremental" จริง — ทุกครั้งที่มี order item ใหม่ ตัวเลขจะอัปเดตทันทีโดยไม่ต้องคำนวณใหม่ทั้งหมด แต่แลกมาด้วยความซับซ้อนที่เพิ่มขึ้นมาก (ต้องจัดการ `UPDATE`/`DELETE` บน `order_items` ด้วย ไม่ใช่แค่ `INSERT`) และเพิ่ม overhead ให้กับทุก transaction ที่เขียนข้อมูล

**3. ใช้ Extension ของบุคคลที่สาม**

มี extension อย่าง [`pg_ivm`](https://github.com/sraoss/pg_ivm) (Incremental View Maintenance) ที่พัฒนาโดยชุมชนเพื่อเติมเต็มช่องว่างนี้โดยเฉพาะ — รองรับการสร้าง "Incrementally Maintainable Materialized View (IMMV)" ที่อัปเดตอัตโนมัติแบบ incremental เมื่อข้อมูลต้นทางเปลี่ยน (ผ่าน trigger ภายใน) แต่ ณ ปัจจุบัน extension นี้ยังไม่ได้เป็นส่วนหนึ่งของ PostgreSQL core และมีข้อจำกัดเรื่อง query ที่รองรับ (เช่น ไม่รองรับทุกรูปแบบของ `JOIN` หรือ aggregate function บางตัว)

### สรุปแนวทาง

| แนวทาง | Incremental จริงหรือไม่ | ความซับซ้อน | เหมาะกับ |
|---|---|---|---|
| `REFRESH` แบบธรรมดา/CONCURRENTLY | ไม่ (รัน query ใหม่ทั้งหมด) | ต่ำ | ส่วนใหญ่ของ use case ทั่วไป |
| แบ่งตามช่วงเวลา + UNION ALL | บางส่วน | กลาง | ข้อมูลที่มีส่วน "นิ่ง" ชัดเจน (historical) |
| Trigger + Summary Table | ใช่ | สูง | ต้องการ real-time aggregate ที่แม่นยำสุด |
| `pg_ivm` (extension) | ใช่ (ในขอบเขตที่รองรับ) | กลาง-สูง | ทีมที่ยอมรับความเสี่ยงของ extension ภายนอก |

> **คำแนะนำ:** เริ่มต้นด้วย `REFRESH MATERIALIZED VIEW CONCURRENTLY` แบบธรรมดาก่อนเสมอ เพราะง่ายและเพียงพอสำหรับ use case ส่วนใหญ่ จะพิจารณาแนวทางที่ซับซ้อนขึ้นก็ต่อเมื่อพิสูจน์แล้วว่า query ต้นทางหนักเกินไปจนกระทบ SLA ของระบบจริง ๆ

---

## Step 359: DROP MATERIALIZED VIEW, การดูขนาดและสถิติของ materialized view

### DROP MATERIALIZED VIEW

การลบ Materialized View ใช้ syntax คล้ายกับการลบ view หรือ table:

```sql
DROP MATERIALIZED VIEW [IF EXISTS] view_name [CASCADE | RESTRICT];
```

```sql
DROP MATERIALIZED VIEW IF EXISTS test_concurrent_mv;
```

```
DROP MATERIALIZED VIEW
```

หากมีวัตถุอื่นที่พึ่งพา Materialized View นี้อยู่ (เช่น view อื่นที่ query จากมัน) การลบแบบธรรมดาจะ error:

```sql
CREATE VIEW top_products_v AS
SELECT * FROM product_sales_summary_mv WHERE total_revenue > 10000;

DROP MATERIALIZED VIEW product_sales_summary_mv;
```

```
ERROR:  cannot drop materialized view product_sales_summary_mv because other objects depend on it
DETAIL:  view top_products_v depends on materialized view product_sales_summary_mv
HINT:  Use DROP ... CASCADE to drop the dependent objects too.
```

ใช้ `CASCADE` เพื่อลบทั้ง Materialized View และวัตถุที่พึ่งพามันไปพร้อมกัน (ต้องระมัดระวังเพราะจะลบ view ที่พึ่งพาด้วย):

```sql
DROP MATERIALIZED VIEW product_sales_summary_mv CASCADE;
```

```
NOTICE:  drop cascades to view top_products_v
DROP MATERIALIZED VIEW
```

### การดูรายชื่อ Materialized View ทั้งหมด

```sql
SELECT schemaname, matviewname, matviewowner, ispopulated
FROM pg_matviews
ORDER BY matviewname;
```

```
 schemaname |         matviewname          | matviewowner | ispopulated
------------+--------------------------------+---------------+-------------
 public     | customer_lifetime_value_mv     | app_user      | t
 public     | customer_monthly_orders_mv     | app_user      | t
 public     | monthly_revenue_mv              | app_user      | t
(3 rows)
```

คอลัมน์ `ispopulated` บอกว่า Materialized View นั้นมีข้อมูลแล้วหรือยัง (สำคัญกรณีสร้างด้วย `WITH NO DATA` แล้วยังไม่เคย `REFRESH`)

### การดูขนาด (Storage Size)

ใช้ฟังก์ชัน `pg_size_pretty` ร่วมกับ `pg_total_relation_size` เหมือนกับตารางทั่วไป:

```sql
SELECT
    matviewname,
    pg_size_pretty(pg_total_relation_size(schemaname || '.' || matviewname)) AS total_size,
    pg_size_pretty(pg_relation_size(schemaname || '.' || matviewname))       AS table_size,
    pg_size_pretty(
        pg_total_relation_size(schemaname || '.' || matviewname)
        - pg_relation_size(schemaname || '.' || matviewname)
    ) AS index_size
FROM pg_matviews
ORDER BY pg_total_relation_size(schemaname || '.' || matviewname) DESC;
```

```
          matviewname          | total_size | table_size | index_size
--------------------------------+------------+------------+------------
 customer_monthly_orders_mv     | 48 kB      | 16 kB      | 32 kB
 customer_lifetime_value_mv     | 24 kB      | 16 kB      | 8 kB
 monthly_revenue_mv              | 16 kB      | 16 kB      | 0 bytes
(3 rows)
```

ในข้อมูลตัวอย่างขนาดเล็กนี้ตัวเลขจะดูเล็กมาก แต่ในระบบจริงที่มีข้อมูลหลักล้านแถว ตัวเลขนี้จะช่วยให้ทีมงานประเมินได้ว่า Materialized View กินพื้นที่ดิสก์เท่าไหร่ และ index กินพื้นที่เพิ่มเท่าไหร่

### การดูสถิติการใช้งาน (การอ่าน / index usage)

Materialized View ถูกนับรวมอยู่ใน `pg_stat_user_tables` และ `pg_stat_user_indexes` เหมือนตารางทั่วไป ทำให้ตรวจสอบสถิติการใช้งานได้:

```sql
SELECT
    relname,
    seq_scan,
    seq_tup_read,
    idx_scan,
    idx_tup_fetch,
    n_live_tup,
    last_vacuum,
    last_analyze
FROM pg_stat_user_tables
WHERE relname = 'customer_monthly_orders_mv';
```

```
          relname            | seq_scan | seq_tup_read | idx_scan | idx_tup_fetch | n_live_tup | last_vacuum | last_analyze
-------------------------------+----------+---------------+----------+----------------+-------------+--------------+---------------
 customer_monthly_orders_mv    |        3 |            45 |        7 |             21 |          15 |              | 2024-07-01 ...
(1 row)
```

ข้อมูลนี้ช่วยตอบคำถามสำคัญ เช่น:
- `seq_scan` สูงเทียบกับ `idx_scan` ต่ำ → อาจบ่งบอกว่า query ที่ใช้ยังไม่ได้ใช้ index ที่สร้างไว้ ควรตรวจสอบ `EXPLAIN`
- `last_analyze` เก่ามาก → สถิติอาจไม่ตรงกับข้อมูลจริง ควรรัน `ANALYZE` เพิ่มเติม

### ตรวจสอบ index ที่มีอยู่บน Materialized View

```sql
SELECT
    indexname,
    indexdef
FROM pg_indexes
WHERE tablename = 'customer_monthly_orders_mv';
```

```
        indexname              |                                        indexdef
---------------------------------+---------------------------------------------------------------------------------------
 idx_cmo_mv_customer_month       | CREATE UNIQUE INDEX idx_cmo_mv_customer_month ON public.customer_monthly_orders_mv ...
 idx_cmo_mv_order_month          | CREATE INDEX idx_cmo_mv_order_month ON public.customer_monthly_orders_mv ...
 idx_cmo_mv_country              | CREATE INDEX idx_cmo_mv_country ON public.customer_monthly_orders_mv ...
 idx_cmo_mv_high_value           | CREATE INDEX idx_cmo_mv_high_value ON public.customer_monthly_orders_mv ...
(4 rows)
```

---

## Step 360: แบบฝึกหัดรวม — สร้าง Materialized View สำหรับ Dashboard ของระบบ E-commerce

ในสถานการณ์จริง ทีม data/backend มักถูกขอให้สร้าง dashboard สำหรับผู้บริหารที่ต้องแสดง **top products** และ **monthly revenue** ซึ่งเป็น query ที่หนักเพราะต้อง join หลายตารางและทำ aggregate ข้ามข้อมูลทั้งหมด ในสถานการณ์ที่มีผู้ใช้จำนวนมากเปิด dashboard พร้อมกัน Materialized View คือคำตอบที่เหมาะสม

### ออกแบบ Materialized View ที่ 1: Top Products Dashboard

```sql
CREATE MATERIALIZED VIEW dashboard_top_products_mv AS
SELECT
    p.product_id,
    p.product_name,
    cat.category_name,
    s.supplier_name,
    COUNT(DISTINCT oi.order_id)         AS order_count,
    SUM(oi.quantity)                    AS units_sold,
    SUM(oi.quantity * oi.unit_price)    AS total_revenue,
    ROUND(AVG(r.rating)::numeric, 2)    AS avg_rating,
    COUNT(DISTINCT r.review_id)         AS review_count
FROM products p
JOIN categories cat ON cat.category_id = p.category_id
JOIN suppliers s ON s.supplier_id = p.supplier_id
LEFT JOIN order_items oi ON oi.product_id = p.product_id
LEFT JOIN orders o ON o.order_id = oi.order_id AND o.status = 'completed'
LEFT JOIN reviews r ON r.product_id = p.product_id
GROUP BY p.product_id, p.product_name, cat.category_name, s.supplier_name
WITH NO DATA;

CREATE UNIQUE INDEX idx_dashboard_top_products_mv_pk
    ON dashboard_top_products_mv (product_id);

CREATE INDEX idx_dashboard_top_products_mv_revenue
    ON dashboard_top_products_mv (total_revenue DESC);

REFRESH MATERIALIZED VIEW dashboard_top_products_mv;
```

```sql
SELECT product_name, category_name, order_count, units_sold, total_revenue, avg_rating
FROM dashboard_top_products_mv
ORDER BY total_revenue DESC NULLS LAST
LIMIT 5;
```

```
        product_name         |     category_name      | order_count | units_sold | total_revenue | avg_rating
------------------------------+-------------------------+--------------+-------------+-----------------+------------
 โน้ตบุ๊ก UltraSlim 14"       | คอมพิวเตอร์และแล็ปท็อป  |            4 |           4 |      99600.00  |      4.50
 สมาร์ทโฟน NovaPhone X12      | โทรศัพท์มือถือ          |            3 |           3 |      56700.00  |      4.00
 จักรยานเสือภูเขา TrailBlazer | กีฬาและกลางแจ้ง         |            1 |           1 |      12900.00  |      5.00
 หูฟังไร้สาย SoundWave Pro    | อิเล็กทรอนิกส์          |            3 |           3 |       9870.00  |      4.50
 เต็นท์แคมป์ปิ้ง 4 คน          | กีฬาและกลางแจ้ง         |            2 |           2 |       7980.00  |      3.00
(5 rows)
```

### ออกแบบ Materialized View ที่ 2: Monthly Revenue Dashboard

```sql
CREATE MATERIALIZED VIEW dashboard_monthly_revenue_mv AS
SELECT
    date_trunc('month', o.order_date)::date AS revenue_month,
    o.ship_country,
    COUNT(DISTINCT o.order_id)              AS order_count,
    COUNT(DISTINCT o.customer_id)           AS unique_customers,
    SUM(oi.quantity * oi.unit_price)        AS total_revenue,
    ROUND(AVG(oi.quantity * oi.unit_price), 2) AS avg_line_item_value
FROM orders o
JOIN order_items oi ON oi.order_id = o.order_id
WHERE o.status = 'completed'
GROUP BY date_trunc('month', o.order_date), o.ship_country
WITH NO DATA;

CREATE UNIQUE INDEX idx_dashboard_monthly_revenue_mv_pk
    ON dashboard_monthly_revenue_mv (revenue_month, ship_country);

CREATE INDEX idx_dashboard_monthly_revenue_mv_month
    ON dashboard_monthly_revenue_mv (revenue_month);

REFRESH MATERIALIZED VIEW dashboard_monthly_revenue_mv;
```

```sql
SELECT revenue_month, ship_country, order_count, total_revenue
FROM dashboard_monthly_revenue_mv
WHERE revenue_month = '2024-01-01'
ORDER BY total_revenue DESC;
```

```
 revenue_month | ship_country | order_count | total_revenue
---------------+--------------+--------------+----------------
 2024-01-01    | Thailand     |            4 |     51837.00
(1 row)
```

### สร้าง View รวมสำหรับสรุปภาพรวม (ต่อยอดจาก Materialized View ทั้งสอง)

```sql
CREATE VIEW dashboard_summary_v AS
SELECT
    (SELECT SUM(total_revenue) FROM dashboard_monthly_revenue_mv) AS grand_total_revenue,
    (SELECT COUNT(*) FROM dashboard_top_products_mv WHERE total_revenue > 0) AS active_selling_products,
    (SELECT product_name FROM dashboard_top_products_mv
        ORDER BY total_revenue DESC NULLS LAST LIMIT 1) AS best_seller;

SELECT * FROM dashboard_summary_v;
```

```
 grand_total_revenue | active_selling_products |       best_seller
----------------------+---------------------------+---------------------------
             207180.00 |                         15 | โน้ตบุ๊ก UltraSlim 14"
(1 row)
```

### กลยุทธ์ Refresh สำหรับระบบจริง

ในระบบ production เราจะไม่ refresh ทุก Materialized View ด้วยความถี่เดียวกัน เพราะแต่ละตัวมีความต้องการความสดของข้อมูลต่างกัน และมีต้นทุนการคำนวณต่างกัน กลยุทธ์ที่แนะนำ:

```sql
-- ติดตั้ง pg_cron
CREATE EXTENSION IF NOT EXISTS pg_cron;

-- 1) Top Products: ข้อมูลเปลี่ยนไม่บ่อยนัก (ยอดขายสะสม) -> refresh ทุก 1 ชั่วโมงพอ
SELECT cron.schedule(
    'refresh-dashboard-top-products',
    '0 * * * *',
    $$REFRESH MATERIALIZED VIEW CONCURRENTLY dashboard_top_products_mv$$
);

-- 2) Monthly Revenue: ผู้บริหารดูภาพรวม ไม่ต้องสดมาก -> refresh ทุก 6 ชั่วโมง
SELECT cron.schedule(
    'refresh-dashboard-monthly-revenue',
    '0 */6 * * *',
    $$REFRESH MATERIALIZED VIEW CONCURRENTLY dashboard_monthly_revenue_mv$$
);

-- 3) รัน ANALYZE ตามหลัง refresh หลักทุกวันตอนดึก เพื่อให้ query planner มีสถิติล่าสุด
SELECT cron.schedule(
    'analyze-dashboard-mvs',
    '30 2 * * *',
    $$ANALYZE dashboard_top_products_mv; ANALYZE dashboard_monthly_revenue_mv;$$
);
```

**เหตุผลเบื้องหลังการออกแบบ:**

1. **ใช้ `CONCURRENTLY` เสมอ** เพราะ dashboard มีผู้ใช้เปิดดูตลอดเวลาในเวลาทำการ ไม่ต้องการให้ query ค้างระหว่าง refresh
2. **สร้าง unique index ทุกตัวตั้งแต่แรก** เพื่อให้ `CONCURRENTLY` ใช้งานได้
3. **ความถี่ refresh ต่างกันตามลักษณะข้อมูล** — ข้อมูลที่ผู้บริหารใช้ตัดสินใจเชิงกลยุทธ์ (monthly) ไม่จำเป็นต้องสดเท่าข้อมูลปฏิบัติการ (operational)
4. **แยก `ANALYZE` ออกมาเป็น job ต่างหาก** ในช่วงเวลาที่มีการใช้งานต่ำ เพื่อไม่ให้กระทบ performance ระหว่างวัน

---

## สรุปท้ายบท

### ตารางเปรียบเทียบ View vs Materialized View vs Table

| คุณสมบัติ | Regular View | Materialized View | Table |
|---|---|---|---|
| เก็บข้อมูลจริงหรือไม่ | ไม่เก็บ (แค่ query ที่บันทึกไว้) | เก็บ (ผลลัพธ์ของ query ณ เวลา refresh ล่าสุด) | เก็บ (ข้อมูลที่ insert/update จริง) |
| ความสดของข้อมูล | สดเสมอ (real-time ตามต้นทาง) | อาจไม่สด — สดเท่าที่ refresh ล่าสุด | สดเสมอ (คือแหล่งข้อมูลเอง) |
| ความเร็วในการอ่าน | ช้ากว่า (query ต้นทางใหม่ทุกครั้ง) | เร็ว (อ่านข้อมูลที่เก็บไว้แล้ว) | เร็ว (ข้อมูลตรงตัว ไม่ต้องคำนวณ) |
| สร้าง Index ได้หรือไม่ | ไม่ได้โดยตรง | ได้ (เหมือนตาราง) | ได้ |
| แก้ไขข้อมูล (INSERT/UPDATE/DELETE) | ได้ในบางเงื่อนไข (updatable view) | ไม่ได้ (read-only, ต้อง REFRESH เท่านั้น) | ได้ |
| ใช้พื้นที่จัดเก็บ | แทบไม่ใช้ | ใช้เท่าขนาดผลลัพธ์ query | ใช้ตามข้อมูลจริง |
| ต้องดูแล/บำรุงรักษาเพิ่ม | ไม่ต้อง | ต้อง (วาง schedule refresh, ตรวจสอบ freshness) | ต้อง (vacuum, index maintenance) |
| เหมาะกับ | query ที่เบา หรือข้อมูลต้อง real-time | query หนัก ที่เรียกบ่อย และยอมรับข้อมูลล่าช้าได้ | ข้อมูลต้นทางของระบบ (source of truth) |

### สิ่งที่ควรจำจากบทนี้

1. Materialized View เก็บข้อมูลจริงบนดิสก์ ต่างจาก view ธรรมดาที่เก็บแค่คำสั่ง query
2. การ query Materialized View เร็วกว่ามากเพราะไม่ต้องคำนวณ join/aggregate ใหม่ทุกครั้ง
3. ข้อมูลใน Materialized View **ไม่อัปเดตอัตโนมัติ** ต้อง `REFRESH` ด้วยตนเองหรือผ่าน scheduler
4. `REFRESH` แบบธรรมดาใช้ `ACCESS EXCLUSIVE LOCK` ซึ่ง block การอ่านทั้งหมดระหว่างกระบวนการ
5. `REFRESH ... CONCURRENTLY` ไม่ block การอ่าน แต่ **ต้องมี unique index** และใช้ทรัพยากรมากกว่า
6. สามารถสร้าง index บน Materialized View ได้เหมือนตารางทั่วไป เพื่อเร่งความเร็ว query ต่อ
7. `pg_cron` เป็นเครื่องมือยอดนิยมสำหรับ schedule การ refresh อัตโนมัติภายในฐานข้อมูล
8. PostgreSQL ไม่รองรับ incremental refresh แบบ native — ทุกครั้งที่ refresh คือการคำนวณ query ใหม่ทั้งหมด ต้องใช้ trick หรือ extension อย่าง `pg_ivm` หากต้องการ incremental จริง ๆ
9. ตรวจสอบขนาดและสถิติผ่าน `pg_matviews`, `pg_stat_user_tables`, `pg_size_pretty` ได้เหมือนตารางทั่วไป
10. การออกแบบ dashboard ที่ดีควรแยก Materialized View ตามลักษณะข้อมูล และกำหนดความถี่ refresh ให้เหมาะกับความต้องการความสดของแต่ละส่วน

---

## แบบฝึกหัด

<details>
<summary><strong>แบบฝึกหัดที่ 1:</strong> จงอธิบายความแตกต่างหลัก 3 ข้อระหว่าง Regular View กับ Materialized View</summary>

**เฉลย:**

1. **การเก็บข้อมูล** — Regular View ไม่เก็บข้อมูลจริง เป็นแค่ query ที่บันทึกชื่อไว้ ส่วน Materialized View เก็บผลลัพธ์จริงบนดิสก์
2. **ความสดของข้อมูล** — Regular View สดเสมอเพราะ query ใหม่ทุกครั้งที่เรียกใช้ ส่วน Materialized View อาจข้อมูลเก่า (stale) จนกว่าจะมีการ `REFRESH`
3. **ความสามารถในการสร้าง Index** — Regular View สร้าง index ตรง ๆ ไม่ได้ (เพราะไม่มีข้อมูลเก็บจริง) แต่ Materialized View สร้าง index ได้เหมือนตารางปกติ

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 2:</strong> เขียนคำสั่งสร้าง Materialized View ชื่อ <code>customer_review_summary_mv</code> ที่สรุปจำนวนรีวิวและคะแนนเฉลี่ยของแต่ละลูกค้า (customer) ที่เคยเขียนรีวิว</summary>

**เฉลย:**

```sql
CREATE MATERIALIZED VIEW customer_review_summary_mv AS
SELECT
    c.customer_id,
    c.first_name || ' ' || c.last_name AS customer_name,
    COUNT(r.review_id)              AS total_reviews,
    ROUND(AVG(r.rating)::numeric, 2) AS avg_rating_given
FROM customers c
JOIN reviews r ON r.customer_id = c.customer_id
GROUP BY c.customer_id, c.first_name, c.last_name;
```

```sql
SELECT * FROM customer_review_summary_mv ORDER BY total_reviews DESC;
```

```
 customer_id | customer_name  | total_reviews | avg_rating_given
-------------+-----------------+----------------+-------------------
           1 | สมชาย ใจดี      |              2 |              4.50
           ...
```

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 3:</strong> จากแบบฝึกหัดที่ 2 ให้เพิ่ม unique index ที่เหมาะสม แล้วทดลอง <code>REFRESH ... CONCURRENTLY</code></summary>

**เฉลย:**

```sql
CREATE UNIQUE INDEX idx_customer_review_summary_mv_pk
    ON customer_review_summary_mv (customer_id);

REFRESH MATERIALIZED VIEW CONCURRENTLY customer_review_summary_mv;
```

```
CREATE INDEX
REFRESH MATERIALIZED VIEW
```

ต้องมี unique index ก่อน มิฉะนั้นจะได้ error:
```
ERROR:  cannot refresh materialized view "customer_review_summary_mv" concurrently
HINT:  Create a unique index with no WHERE clause on one or more columns of the materialized view.
```

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 4:</strong> อธิบายว่าทำไม <code>REFRESH MATERIALIZED VIEW</code> (ไม่ใส่ CONCURRENTLY) จึงเป็นอันตรายต่อระบบที่มีผู้ใช้งาน dashboard พร้อมกันจำนวนมาก</summary>

**เฉลย:**

เพราะ `REFRESH MATERIALIZED VIEW` แบบธรรมดาจะขอ `ACCESS EXCLUSIVE LOCK` บน Materialized View ตลอดระยะเวลาที่กำลัง refresh ซึ่งเป็น lock ระดับสูงสุด — จะ block การเข้าถึงทุกรูปแบบ รวมถึง `SELECT` ธรรมดาด้วย ถ้ามีผู้ใช้จำนวนมากพยายาม query dashboard พร้อมกันในช่วงที่กำลัง refresh (โดยเฉพาะถ้า query ต้นทางหนักและใช้เวลานาน) ผู้ใช้ทุกคนจะต้อง "รอ" จนกว่า refresh จะเสร็จ ทำให้ dashboard ดูเหมือนค้างหรือช้าอย่างผิดปกติ วิธีแก้คือใช้ `REFRESH MATERIALIZED VIEW CONCURRENTLY` แทน ซึ่งไม่ block การอ่าน (แต่ต้องมี unique index)

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 5:</strong> เขียนคำสั่งตรวจสอบขนาด (size) ของ Materialized View ทั้งหมดในฐานข้อมูล เรียงจากใหญ่ไปเล็ก</summary>

**เฉลย:**

```sql
SELECT
    matviewname,
    pg_size_pretty(pg_total_relation_size(schemaname || '.' || matviewname)) AS total_size
FROM pg_matviews
ORDER BY pg_total_relation_size(schemaname || '.' || matviewname) DESC;
```

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 6:</strong> จงตั้ง <code>pg_cron</code> job เพื่อ refresh Materialized View ชื่อ <code>dashboard_top_products_mv</code> ทุก 30 นาที แบบไม่ block การอ่าน</summary>

**เฉลย:**

```sql
SELECT cron.schedule(
    'refresh-top-products-every-30min',
    '*/30 * * * *',
    $$REFRESH MATERIALIZED VIEW CONCURRENTLY dashboard_top_products_mv$$
);
```

ต้องแน่ใจว่า `dashboard_top_products_mv` มี unique index อยู่แล้ว ก่อนตั้ง job นี้ มิฉะนั้น job จะรันไม่สำเร็จ (ตรวจสอบผ่าน `cron.job_run_details`)

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 7:</strong> ทำไม PostgreSQL จึงไม่รองรับ incremental refresh แบบ native และมีแนวทางแก้ปัญหาอะไรบ้าง</summary>

**เฉลย:**

PostgreSQL core ออกแบบ Materialized View ให้เป็น snapshot ของผลลัพธ์ query ณ เวลาที่ refresh — ทุกครั้งที่ refresh คือการรัน query ต้นทางใหม่ทั้งหมดและแทนที่ข้อมูลเดิม (หรือ diff ทีละแถวในกรณี `CONCURRENTLY` แต่ก็ยังต้องคำนวณผลลัพธ์ทั้งชุดใหม่อยู่ดี) ไม่มีกลไก "จดจำ" ว่าแถวไหนเปลี่ยนเพื่ออัปเดตเฉพาะจุด

แนวทางแก้ปัญหาที่ใช้กันได้แก่:
1. แบ่ง Materialized View ตามช่วงเวลา (historical vs current) แล้วรวมด้วย `UNION ALL`
2. ใช้ trigger เขียนตาราง summary เอง (แลกกับความซับซ้อนที่เพิ่มขึ้น)
3. ใช้ extension ของบุคคลที่สามอย่าง `pg_ivm` ที่รองรับ Incremental View Maintenance ในขอบเขตจำกัด

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 8:</strong> เขียน query ที่ตรวจสอบว่า Materialized View ชื่อ <code>monthly_revenue_mv</code> มีข้อมูลอยู่แล้วหรือยัง (populated) โดยไม่ต้อง SELECT ข้อมูลจริง</summary>

**เฉลย:**

```sql
SELECT matviewname, ispopulated
FROM pg_matviews
WHERE matviewname = 'monthly_revenue_mv';
```

```
    matviewname     | ispopulated
---------------------+-------------
 monthly_revenue_mv  | t
(1 row)
```

ถ้า `ispopulated = f` แปลว่ายังไม่มีข้อมูล (สร้างด้วย `WITH NO DATA` แล้วยังไม่เคย `REFRESH`) การ `SELECT` จาก view นั้นตรง ๆ จะเกิด error

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 9:</strong> จงลบ Materialized View ชื่อ <code>customer_review_summary_mv</code> ที่สร้างในแบบฝึกหัดที่ 2 อย่างปลอดภัย โดยตรวจสอบก่อนว่ามี object อื่นพึ่งพาอยู่หรือไม่</summary>

**เฉลย:**

```sql
-- ตรวจสอบ dependency ก่อนลบ
SELECT
    dependent_ns.nspname AS dependent_schema,
    dependent_view.relname AS dependent_view
FROM pg_depend
JOIN pg_rewrite ON pg_depend.objid = pg_rewrite.oid
JOIN pg_class AS dependent_view ON pg_rewrite.ev_class = dependent_view.oid
JOIN pg_class AS source_table ON pg_depend.refobjid = source_table.oid
JOIN pg_namespace dependent_ns ON dependent_ns.oid = dependent_view.relnamespace
WHERE source_table.relname = 'customer_review_summary_mv'
  AND dependent_view.relname != 'customer_review_summary_mv';

-- ถ้าไม่มีผลลัพธ์ (ไม่มี object อื่นพึ่งพา) ลบได้อย่างปลอดภัย
DROP MATERIALIZED VIEW IF EXISTS customer_review_summary_mv;

-- ถ้ามี object พึ่งพา และต้องการลบทั้งหมด ใช้ CASCADE
-- DROP MATERIALIZED VIEW customer_review_summary_mv CASCADE;
```

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 10 (โจทย์รวม):</strong> ออกแบบและสร้าง Materialized View สำหรับ dashboard ที่แสดง "ยอดขายแยกตามหมวดหมู่สินค้ารายเดือน" (monthly revenue by category) พร้อม unique index และ schedule การ refresh ทุก 2 ชั่วโมงด้วย pg_cron</summary>

**เฉลย:**

```sql
-- 1. สร้าง Materialized View
CREATE MATERIALIZED VIEW category_monthly_revenue_mv AS
SELECT
    cat.category_id,
    cat.category_name,
    date_trunc('month', o.order_date)::date AS revenue_month,
    COUNT(DISTINCT o.order_id)              AS order_count,
    SUM(oi.quantity)                        AS units_sold,
    SUM(oi.quantity * oi.unit_price)        AS total_revenue
FROM categories cat
JOIN products p ON p.category_id = cat.category_id
JOIN order_items oi ON oi.product_id = p.product_id
JOIN orders o ON o.order_id = oi.order_id
WHERE o.status = 'completed'
GROUP BY cat.category_id, cat.category_name, date_trunc('month', o.order_date)
WITH NO DATA;

-- 2. สร้าง unique index (จำเป็นสำหรับ CONCURRENTLY)
CREATE UNIQUE INDEX idx_category_monthly_revenue_mv_pk
    ON category_monthly_revenue_mv (category_id, revenue_month);

-- 3. สร้าง index เสริมสำหรับกรองตามเดือน
CREATE INDEX idx_category_monthly_revenue_mv_month
    ON category_monthly_revenue_mv (revenue_month);

-- 4. refresh ครั้งแรก (ต้องเป็นแบบไม่ CONCURRENTLY เพราะยังไม่มีข้อมูล)
REFRESH MATERIALIZED VIEW category_monthly_revenue_mv;

-- 5. ตั้ง schedule ด้วย pg_cron ทุก 2 ชั่วโมง
CREATE EXTENSION IF NOT EXISTS pg_cron;

SELECT cron.schedule(
    'refresh-category-monthly-revenue',
    '0 */2 * * *',
    $$REFRESH MATERIALIZED VIEW CONCURRENTLY category_monthly_revenue_mv$$
);
```

ทดสอบ query:

```sql
SELECT category_name, revenue_month, order_count, total_revenue
FROM category_monthly_revenue_mv
ORDER BY revenue_month, total_revenue DESC
LIMIT 5;
```

```
     category_name          | revenue_month | order_count | total_revenue
------------------------------+---------------+--------------+----------------
 คอมพิวเตอร์และแล็ปท็อป       | 2024-01-01    |            2 |     27380.00
 แฟชั่น (เสื้อผ้าผู้ชาย)        | 2024-01-01    |            1 |      1667.00
 อิเล็กทรอนิกส์               | 2024-01-01    |            1 |      3290.00
 กีฬาและกลางแจ้ง              | 2024-02-01    |            1 |      3980.00
 หนังสือ                     | 2024-02-01    |            1 |      1990.00
(5 rows)
```

**เหตุผลของการออกแบบ:**
- ใช้ `WITH NO DATA` แล้ว refresh แยก เพื่อให้สามารถ deploy โครงสร้างก่อนแล้วค่อยเติมข้อมูล (ดีต่อการ migration ในระบบ production)
- Unique index บน `(category_id, revenue_month)` สะท้อน grain ที่แท้จริงของข้อมูล (หนึ่งแถวต่อหนึ่งหมวดหมู่ต่อหนึ่งเดือน) ทำให้ `CONCURRENTLY` ใช้งานได้และป้องกันข้อมูลซ้ำ
- Schedule ทุก 2 ชั่วโมงเหมาะสมเพราะข้อมูลระดับหมวดหมู่ไม่จำเป็นต้อง real-time เป๊ะ แต่ก็ไม่ควรเก่าเกินไปสำหรับ dashboard ที่ผู้จัดการหมวดหมู่สินค้าใช้ติดตาม

</details>

---

## บทถัดไป

จบบทนี้แล้ว ผู้เรียนควรเข้าใจ Materialized View ตั้งแต่แนวคิดพื้นฐานไปจนถึงการวางกลยุทธ์ refresh สำหรับระบบ dashboard จริง บทถัดไปจะพาไปสู่หัวใจสำคัญของการรับประกันความถูกต้องของข้อมูลในระบบฐานข้อมูล — **Transactions และ ACID Properties**

**อ่านต่อ: [Part 037 — Transactions และ ACID](./part-037-transactions-acid.md)**
