# Window Functions ขั้นสูง: RANK, DENSE_RANK, ROW_NUMBER, LAG/LEAD, NTILE

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับกลาง | Part 030

---

## เป้าหมายการเรียนรู้

หลังจากเรียนจบบทนี้ ผู้เรียนจะสามารถ:

1. ใช้ `ROW_NUMBER()` สร้างเลขลำดับแถวที่ไม่ซ้ำกัน และนำไปใช้เลือก "แถวล่าสุดต่อกลุ่ม" แทน `DISTINCT ON`
2. เข้าใจความแตกต่างระหว่าง `RANK()`, `DENSE_RANK()` และ `ROW_NUMBER()` เมื่อข้อมูลมีค่าเท่ากัน (ties)
3. ใช้ `PERCENT_RANK()` และ `CUME_DIST()` คำนวณอันดับเชิงสัดส่วน (percentile-style ranking)
4. ใช้ `NTILE(n)` แบ่งข้อมูลออกเป็นกลุ่มเท่า ๆ กัน เช่น แบ่งลูกค้าเป็นควอร์ไทล์ตามยอดซื้อ
5. ใช้ `LAG()` เปรียบเทียบค่าปัจจุบันกับค่าก่อนหน้าในกลุ่มเดียวกัน เช่น ยอดขายเดือนนี้เทียบเดือนก่อน
6. ใช้ `LEAD()` ดึงค่าจากแถวถัดไป เช่น คำนวณระยะเวลาห่างระหว่างคำสั่งซื้อของลูกค้าคนเดียวกัน
7. ใช้ `FIRST_VALUE()`, `LAST_VALUE()`, `NTH_VALUE()` พร้อมระบุ frame clause ให้ถูกต้อง และรู้ทันข้อผิดพลาดยอดฮิตของ `LAST_VALUE()`
8. ผสมผสาน window function หลายตัวเข้าด้วยกันเพื่อสร้างรายงานวิเคราะห์ธุรกิจ (customer segmentation, month-over-month growth)
9. อ่านและเขียน query เชิงวิเคราะห์ระดับ Business Intelligence ด้วย window function ได้อย่างมั่นใจ
10. เลือกใช้ window function ที่ "ใช่" กับโจทย์แต่ละแบบ แทนการจำสูตรแบบท่องจำ

> **ข้อกำหนดเบื้องต้น**: บทนี้ต่อยอดจาก window function พื้นฐาน (`OVER()`, `PARTITION BY`, aggregate window functions) หากยังไม่คุ้นเคย แนะนำให้ทบทวนบทก่อนหน้าในหมวด window functions เบื้องต้นก่อน

---

## เตรียมข้อมูล

บทนี้และบทถัดไปในชุด Part 021-039 ใช้ชุดข้อมูลร้านค้าออนไลน์ (e-commerce) เดียวกัน เพื่อให้ผู้เรียนเห็นภาพรวมทางธุรกิจที่ต่อเนื่องกัน โครงสร้างตารางมีดังนี้

```sql
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
```

### ข้อมูลตัวอย่าง (seed data)

```sql
INSERT INTO categories (category_id, category_name, parent_category_id) VALUES
(1,'Electronics', NULL),
(2,'Computers', 1),
(3,'Smartphones', 1),
(4,'Accessories', 1),
(5,'Home & Kitchen', NULL),
(6,'Furniture', 5),
(7,'Books', NULL),
(8,'Toys & Games', NULL),
(9,'Sports & Outdoor', NULL),
(10,'Fashion', NULL),
(11,'Beauty & Personal Care', NULL),
(12,'Groceries', NULL);
SELECT setval('categories_category_id_seq', 12);

INSERT INTO suppliers (supplier_id, supplier_name, country) VALUES
(1,'TechSource Co., Ltd.','Thailand'),
(2,'Global Gadgets Inc.','USA'),
(3,'Shenzhen Electronics Trading','China'),
(4,'Nordic Home Design','Sweden'),
(5,'BookWorld Publishing','UK'),
(6,'PlayTime Toys','Thailand'),
(7,'FitLife Sports','Germany'),
(8,'Bangkok Fashion House','Thailand');
SELECT setval('suppliers_supplier_id_seq', 8);

INSERT INTO products (product_id, product_name, category_id, supplier_id, unit_price, stock_quantity, is_active) VALUES
(1,'Laptop Pro 14"',2,2,42900.00,25,true),
(2,'Wireless Mouse M1',4,3,590.00,150,true),
(3,'Mechanical Keyboard K2',4,3,2490.00,80,true),
(4,'Smartphone X12',3,2,24900.00,40,true),
(5,'Smartphone Lite',3,3,8900.00,60,true),
(6,'USB-C Hub 7-in-1',4,1,890.00,200,true),
(7,'Bluetooth Earbuds Pro',4,2,3290.00,120,true),
(8,'27" 4K Monitor',2,2,12900.00,30,true),
(9,'Gaming Chair Deluxe',6,4,6990.00,15,true),
(10,'Study Desk Oak',6,4,4590.00,20,true),
(11,'Non-stick Pan Set',5,4,1290.00,50,true),
(12,'Air Fryer 5L',5,4,2990.00,35,true),
(13,'PostgreSQL for Developers',7,5,890.00,100,true),
(14,'The Data Engineer''s Handbook',7,5,750.00,90,true),
(15,'Building Blocks Set 500pcs',8,6,1190.00,45,true),
(16,'RC Racing Car',8,6,1590.00,30,false),
(17,'Yoga Mat Premium',9,7,690.00,100,true),
(18,'Running Shoes Elite',9,7,3490.00,55,true),
(19,'Cotton T-Shirt (5-pack)',10,8,590.00,200,true),
(20,'Denim Jacket',10,8,1990.00,40,true);
SELECT setval('products_product_id_seq', 20);

INSERT INTO customers (customer_id, first_name, last_name, email, country, signup_date) VALUES
(1,'สมชาย','ใจดี','somchai.j@email.com','Thailand','2023-01-15'),
(2,'สมหญิง','รักเรียน','somying.r@email.com','Thailand','2023-02-20'),
(3,'วิชัย','พัฒนา','wichai.p@email.com','Thailand','2023-03-10'),
(4,'กมลวรรณ','สุขใจ','kamolwan.s@email.com','Thailand','2023-04-05'),
(5,'ธนากร','มั่งมี','thanakorn.m@email.com','Thailand','2023-05-12'),
(6,'ปิยะดา','แสงทอง','piyada.s@email.com','Thailand','2023-06-18'),
(7,'อนุชา','ทองดี','anucha.t@email.com','Thailand','2023-07-22'),
(8,'นภัสสร','วงศ์สุวรรณ','napassorn.w@email.com','Thailand','2023-08-30'),
(9,'John','Smith','john.smith@email.com','USA','2023-09-05'),
(10,'Emily','Johnson','emily.j@email.com','USA','2023-10-10'),
(11,'Li','Wei','li.wei@email.com','China','2023-11-15'),
(12,'Yuki','Tanaka','yuki.t@email.com','Japan','2023-12-01'),
(13,'ประภาส','ศรีสุข','prapas.s@email.com','Thailand','2024-01-08'),
(14,'สุนิสา','จันทร์เพ็ญ','sunisa.j@email.com','Thailand','2024-02-14'),
(15,'David','Lee','david.lee@email.com','Singapore','2024-03-20');
SELECT setval('customers_customer_id_seq', 15);

INSERT INTO employees (employee_id, first_name, last_name, hire_date, manager_id, department) VALUES
(1,'มานพ','ชาญชัย','2020-01-10',NULL,'Management'),
(2,'สุภาพร','เก่งงาน','2020-03-15',1,'Sales'),
(3,'ธีรพงษ์','ยอดขยัน','2021-02-01',2,'Sales'),
(4,'อรทัย','มีสุข','2021-06-10',2,'Sales'),
(5,'กิตติศักดิ์','แข็งแรง','2022-01-20',1,'Support'),
(6,'วราภรณ์','ใจเย็น','2022-05-05',5,'Support'),
(7,'ณัฐพล','รุ่งเรือง','2022-09-12',2,'Sales'),
(8,'พิมพ์ใจ','สดใส','2023-03-01',5,'Support');
SELECT setval('employees_employee_id_seq', 8);

INSERT INTO orders (order_id, customer_id, employee_id, order_date, status, ship_country) VALUES
(1,1,2,'2025-01-05 10:15:00+07','completed','Thailand'),
(2,2,3,'2025-01-08 14:30:00+07','completed','Thailand'),
(3,9,4,'2025-01-12 09:00:00+07','completed','USA'),
(4,3,2,'2025-01-15 16:45:00+07','completed','Thailand'),
(5,11,7,'2025-01-18 11:20:00+07','completed','China'),
(6,4,3,'2025-01-22 13:10:00+07','shipped','Thailand'),
(7,1,2,'2025-01-27 08:50:00+07','completed','Thailand'),
(8,5,4,'2025-02-02 10:00:00+07','completed','Thailand'),
(9,2,3,'2025-02-06 15:15:00+07','completed','Thailand'),
(10,10,4,'2025-02-10 09:30:00+07','completed','USA'),
(11,6,2,'2025-02-14 12:00:00+07','completed','Thailand'),
(12,1,2,'2025-02-18 17:20:00+07','completed','Thailand'),
(13,12,7,'2025-02-21 10:40:00+07','shipped','Japan'),
(14,7,3,'2025-02-25 14:00:00+07','pending','Thailand'),
(15,3,2,'2025-03-03 09:10:00+07','completed','Thailand'),
(16,8,4,'2025-03-07 11:30:00+07','completed','Thailand'),
(17,2,3,'2025-03-11 16:00:00+07','completed','Thailand'),
(18,13,2,'2025-03-15 10:20:00+07','completed','Thailand'),
(19,9,4,'2025-03-19 13:45:00+07','cancelled','USA'),
(20,1,2,'2025-03-23 15:30:00+07','completed','Thailand'),
(21,14,3,'2025-03-28 09:00:00+07','completed','Thailand'),
(22,4,3,'2025-04-02 10:10:00+07','completed','Thailand'),
(23,2,3,'2025-04-07 14:20:00+07','completed','Thailand'),
(24,15,7,'2025-04-12 11:00:00+07','completed','Singapore'),
(25,5,4,'2025-04-17 16:30:00+07','shipped','Thailand'),
(26,1,2,'2025-04-22 09:45:00+07','completed','Thailand'),
(27,11,7,'2025-04-27 13:00:00+07','completed','China'),
(28,6,2,'2025-05-03 10:15:00+07','completed','Thailand'),
(29,2,3,'2025-05-08 15:40:00+07','completed','Thailand'),
(30,3,2,'2025-05-12 09:20:00+07','completed','Thailand'),
(31,10,4,'2025-05-16 11:50:00+07','completed','USA'),
(32,7,3,'2025-05-21 14:10:00+07','completed','Thailand'),
(33,1,2,'2025-05-25 16:00:00+07','pending','Thailand'),
(34,12,7,'2025-05-29 10:30:00+07','completed','Japan'),
(35,8,4,'2025-06-02 09:00:00+07','completed','Thailand'),
(36,2,3,'2025-06-06 13:20:00+07','completed','Thailand'),
(37,9,4,'2025-06-10 15:45:00+07','completed','USA'),
(38,1,2,'2025-06-15 10:00:00+07','completed','Thailand'),
(39,13,2,'2025-06-19 12:30:00+07','completed','Thailand'),
(40,3,2,'2025-06-24 14:50:00+07','completed','Thailand');
SELECT setval('orders_order_id_seq', 40);

INSERT INTO order_items (order_item_id, order_id, product_id, quantity, unit_price) VALUES
(1,1,4,1,24900.00),(2,1,2,2,590.00),(3,2,13,2,890.00),(4,3,1,1,42900.00),
(5,4,7,1,3290.00),(6,4,6,1,890.00),(7,5,5,1,8900.00),(8,6,3,1,2490.00),
(9,7,19,3,590.00),(10,8,8,1,12900.00),(11,9,2,1,590.00),(12,9,3,1,2490.00),
(13,10,4,1,24900.00),(14,11,11,1,1290.00),(15,12,14,2,750.00),(16,13,9,1,6990.00),
(17,14,17,2,690.00),(18,15,1,1,42900.00),(19,16,12,1,2990.00),(20,17,7,1,3290.00),
(21,18,18,1,3490.00),(22,19,5,1,8900.00),(23,20,13,1,890.00),(24,20,14,1,750.00),
(25,21,20,1,1990.00),(26,22,10,1,4590.00),(27,23,2,2,590.00),(28,24,4,1,24900.00),
(29,25,6,2,890.00),(30,26,19,2,590.00),(31,27,5,1,8900.00),(32,28,11,1,1290.00),
(33,29,3,1,2490.00),(34,30,1,1,42900.00),(35,31,8,1,12900.00),(36,32,17,1,690.00),
(37,33,15,1,1190.00),(38,34,9,1,6990.00),(39,35,12,1,2990.00),(40,36,7,1,3290.00),
(41,37,4,1,24900.00),(42,38,2,1,590.00),(43,38,6,1,890.00),(44,39,18,1,3490.00),
(45,40,20,1,1990.00);
SELECT setval('order_items_order_item_id_seq', 45);

INSERT INTO reviews (review_id, product_id, customer_id, rating, review_text, review_date) VALUES
(1,1,3,5,'โน้ตบุ๊กแรงมาก คุ้มราคา','2025-01-20'),
(2,1,15,4,'Great laptop, fast shipping','2025-04-15'),
(3,4,1,5,'สมาร์ทโฟนกล้องสวยมาก','2025-01-10'),
(4,4,9,3,'Decent phone but battery drains fast','2025-01-20'),
(5,5,11,4,'ราคาคุ้มค่า ใช้งานลื่น','2025-01-25'),
(6,7,2,5,'เสียงดีมาก คู่ควรกับราคา','2025-02-10'),
(7,8,5,4,'จอสวยคมชัด','2025-02-08'),
(8,9,12,5,'Very comfortable chair','2025-02-25'),
(9,13,2,5,'หนังสือดีมาก เข้าใจง่าย','2025-02-12'),
(10,13,7,4,'เนื้อหาแน่น เหมาะกับมือใหม่','2025-02-28'),
(11,3,4,2,'ปุ่มกดแข็งไปหน่อย','2025-01-25'),
(12,18,8,5,'รองเท้าใส่สบายมาก วิ่งลื่น','2025-03-10'),
(13,19,1,4,'เนื้อผ้านุ่ม ใส่สบาย','2025-01-30'),
(14,6,5,3,'ใช้งานได้ตามปกติ','2025-04-20'),
(15,11,6,5,'กระทะทำอาหารดีมาก ไม่ติดกระทะ','2025-05-05'),
(16,17,7,4,'เสื่อโยคะหนาดี ไม่ลื่น','2025-05-23'),
(17,2,1,5,'เมาส์ลื่น ราคาถูก','2025-06-16'),
(18,20,3,4,'แจ็คเก็ตทรงสวย ใส่ได้ทุกโอกาส','2025-06-26');
SELECT setval('reviews_review_id_seq', 18);

INSERT INTO payments (payment_id, order_id, payment_date, amount, payment_method) VALUES
(1,1,'2025-01-05 10:20:00+07',26080.00,'credit_card'),
(2,2,'2025-01-08 14:35:00+07',1780.00,'promptpay'),
(3,3,'2025-01-12 09:05:00+07',42900.00,'credit_card'),
(4,4,'2025-01-15 16:50:00+07',4180.00,'bank_transfer'),
(5,5,'2025-01-18 11:25:00+07',8900.00,'credit_card'),
(6,6,'2025-01-22 13:15:00+07',2490.00,'cod'),
(7,7,'2025-01-27 08:55:00+07',1770.00,'promptpay'),
(8,8,'2025-02-02 10:05:00+07',12900.00,'credit_card'),
(9,9,'2025-02-06 15:20:00+07',3080.00,'promptpay'),
(10,10,'2025-02-10 09:35:00+07',24900.00,'credit_card'),
(11,11,'2025-02-14 12:05:00+07',1290.00,'bank_transfer'),
(12,12,'2025-02-18 17:25:00+07',1500.00,'promptpay'),
(13,13,'2025-02-21 10:45:00+07',6990.00,'credit_card'),
(14,15,'2025-03-03 09:15:00+07',42900.00,'credit_card'),
(15,16,'2025-03-07 11:35:00+07',2990.00,'bank_transfer'),
(16,17,'2025-03-11 16:05:00+07',3290.00,'promptpay'),
(17,18,'2025-03-15 10:25:00+07',3490.00,'credit_card'),
(18,20,'2025-03-23 15:35:00+07',1640.00,'promptpay'),
(19,21,'2025-03-28 09:05:00+07',1990.00,'cod'),
(20,22,'2025-04-02 10:15:00+07',4590.00,'bank_transfer'),
(21,23,'2025-04-07 14:25:00+07',1180.00,'promptpay'),
(22,24,'2025-04-12 11:05:00+07',24900.00,'credit_card'),
(23,25,'2025-04-17 16:35:00+07',1780.00,'cod'),
(24,26,'2025-04-22 09:50:00+07',1180.00,'promptpay'),
(25,27,'2025-04-27 13:05:00+07',8900.00,'credit_card'),
(26,28,'2025-05-03 10:20:00+07',1290.00,'bank_transfer'),
(27,29,'2025-05-08 15:45:00+07',2490.00,'promptpay'),
(28,30,'2025-05-12 09:25:00+07',42900.00,'credit_card'),
(29,31,'2025-05-16 11:55:00+07',12900.00,'credit_card'),
(30,32,'2025-05-21 14:15:00+07',690.00,'promptpay'),
(31,34,'2025-05-29 10:35:00+07',6990.00,'credit_card'),
(32,35,'2025-06-02 09:05:00+07',2990.00,'bank_transfer'),
(33,36,'2025-06-06 13:25:00+07',3290.00,'promptpay'),
(34,37,'2025-06-10 15:50:00+07',24900.00,'credit_card'),
(35,38,'2025-06-15 10:05:00+07',1480.00,'promptpay'),
(36,39,'2025-06-19 12:35:00+07',3490.00,'credit_card'),
(37,40,'2025-06-24 14:55:00+07',1990.00,'cod');
SELECT setval('payments_payment_id_seq', 37);
```

> **หมายเหตุเรื่อง timezone**: ตัวอย่างผลลัพธ์ทั้งหมดในบทนี้รันด้วย `SET TIME ZONE 'Asia/Bangkok';` ก่อน query เพื่อให้เวลาที่แสดงตรงกับเวลาไทย (`order_date` ถูกบันทึกแบบ `+07` อยู่แล้ว แต่ session timezone ของ PostgreSQL อาจแสดงเป็น UTC โดยดีฟอลต์)

โครงสร้างธุรกิจของข้อมูลชุดนี้:
- ลูกค้า `สมชาย ใจดี` (customer_id = 1) เป็นลูกค้าประจำที่สั่งซื้อทุกเดือนตั้งแต่มกราคมถึงมิถุนายน 2025 (7 ออเดอร์) — เหมาะสำหรับสาธิต `LAG`/`LEAD`
- มียอดขายรายเดือนตั้งแต่มกราคม–มิถุนายน 2025 ที่ขึ้น ๆ ลง ๆ — เหมาะสำหรับสาธิต month-over-month growth
- ลูกค้าหลายคนมีจำนวนออเดอร์เท่ากัน (ค่า tie) — เหมาะสำหรับเปรียบเทียบ `RANK()` กับ `DENSE_RANK()`

---

## Step 291: ROW_NUMBER() — เลขลำดับแถวไม่ซ้ำ

`ROW_NUMBER()` เป็น window function ที่ให้เลขลำดับ **ไม่ซ้ำกันเสมอ** แก่ทุกแถวภายในแต่ละ partition แม้ค่าที่ใช้ `ORDER BY` จะเท่ากันก็ตาม (ตัดสินด้วยลำดับที่ physical/ที่ระบุ tie-breaker เพิ่มเติม)

```sql
SELECT
    order_id,
    customer_id,
    order_date,
    ROW_NUMBER() OVER (ORDER BY order_date) AS rn
FROM orders
WHERE customer_id = 1
ORDER BY order_date;
```

รูปแบบพื้นฐาน:

```
ROW_NUMBER() OVER (
    [PARTITION BY partition_expression, ...]
    [ORDER BY sort_expression [ASC|DESC], ...]
)
```

### กรณีใช้งานที่พบบ่อยที่สุด: หา "แถวล่าสุดต่อกลุ่ม"

โจทย์ธุรกิจที่พบบ่อยมากคือ "หาออเดอร์ล่าสุดของลูกค้าแต่ละคน" วิธีคลาสสิกคือใช้ `DISTINCT ON` (เฉพาะ PostgreSQL) แต่ `ROW_NUMBER()` เป็นวิธีที่ portable ไปยังฐานข้อมูลอื่นได้ และยืดหยุ่นกว่าเมื่อโจทย์ซับซ้อนขึ้น (เช่น ต้องการ top-N ต่อกลุ่ม ไม่ใช่แค่ top-1):

```sql
SELECT order_id, customer_id, order_date, status
FROM (
    SELECT
        order_id,
        customer_id,
        order_date,
        status,
        ROW_NUMBER() OVER (
            PARTITION BY customer_id
            ORDER BY order_date DESC
        ) AS rn
    FROM orders
) ranked
WHERE rn = 1
ORDER BY customer_id;
```

**ผลลัพธ์ (15 แถวแรก แสดงบางส่วน):**

| order_id | customer_id | order_date | status | rn |
|---|---|---|---|---|
| 38 | 1 | 2025-06-15 10:00 | completed | 1 |
| 36 | 2 | 2025-06-06 13:20 | completed | 1 |
| 40 | 3 | 2025-06-24 14:50 | completed | 1 |
| 22 | 4 | 2025-04-02 10:10 | completed | 1 |
| 25 | 5 | 2025-04-17 16:30 | shipped | 1 |
| 28 | 6 | 2025-05-03 10:15 | completed | 1 |
| 32 | 7 | 2025-05-21 14:10 | completed | 1 |
| 35 | 8 | 2025-06-02 09:00 | completed | 1 |
| 37 | 9 | 2025-06-10 15:45 | completed | 1 |
| 31 | 10 | 2025-05-16 11:50 | completed | 1 |

> เทียบกับ `DISTINCT ON (customer_id) ... ORDER BY customer_id, order_date DESC` ที่ให้ผลลัพธ์เดียวกันแต่สั้นกว่า — `DISTINCT ON` เหมาะเมื่ออยากได้แค่ top-1 ต่อกลุ่มและรันบน PostgreSQL เท่านั้น ส่วน `ROW_NUMBER()` ควรใช้เมื่อต้องการความยืดหยุ่นมากกว่า เช่น top-3 ต่อกลุ่ม หรือมี logic การกรองอื่นร่วมด้วย

### เพจจิเนชันด้วย ROW_NUMBER()

`ROW_NUMBER()` ยังนิยมใช้ทำ pagination แบบ deterministic (แถวไม่เลื่อนสลับกันเวลาเปลี่ยนหน้า เพราะมีเลขลำดับตายตัว):

```sql
SELECT product_id, product_name, unit_price, rn
FROM (
    SELECT
        product_id,
        product_name,
        unit_price,
        ROW_NUMBER() OVER (ORDER BY unit_price DESC, product_id) AS rn
    FROM products
    WHERE is_active = true
) numbered
WHERE rn BETWEEN 6 AND 10;
```

**ผลลัพธ์:**

| product_id | product_name | unit_price | rn |
|---|---|---|---|
| 3 | Mechanical Keyboard K2 | 2490.00 | 6 |
| 12 | Air Fryer 5L | 2990.00 | 7 |
| 7 | Bluetooth Earbuds Pro | 3290.00 | 8 |
| 18 | Running Shoes Elite | 3490.00 | 9 |
| 10 | Study Desk Oak | 4590.00 | 10 |

> **ข้อควรระวัง**: ถ้า `ORDER BY` ไม่มีคอลัมน์ที่การันตีความไม่ซ้ำ (เช่น `unit_price` เพียงอย่างเดียวและมีสินค้าราคาเท่ากัน) เลขลำดับของแถวที่ราคาเท่ากันอาจสลับที่กันได้ในการรันครั้งถัดไป ควรเติม tie-breaker เช่น `product_id` เสมอเพื่อผลลัพธ์ deterministic

---

## Step 292: RANK() — อันดับที่ข้ามเลขเมื่อมีค่าเท่ากัน

`RANK()` ให้อันดับตามค่าที่ `ORDER BY` ระบุ โดย **แถวที่มีค่าเท่ากันจะได้อันดับเดียวกัน** และอันดับถัดไปจะ **ข้ามเลข** ไปเท่ากับจำนวนแถวที่เสมอกัน (เช่น ถ้ามี 3 แถวเสมอกันที่อันดับ 1 แถวถัดไปจะเป็นอันดับ 4 ไม่ใช่ 2)

```sql
SELECT
    c.customer_id,
    c.first_name || ' ' || c.last_name AS customer_name,
    COUNT(o.order_id) AS order_count,
    RANK() OVER (ORDER BY COUNT(o.order_id) DESC) AS rnk
FROM customers c
JOIN orders o ON o.customer_id = c.customer_id
GROUP BY c.customer_id, customer_name
ORDER BY order_count DESC, c.customer_id;
```

**ผลลัพธ์:**

| customer_id | customer_name | order_count | rnk |
|---|---|---|---|
| 1 | สมชาย ใจดี | 7 | 1 |
| 2 | สมหญิง รักเรียน | 6 | 2 |
| 3 | วิชัย พัฒนา | 4 | 3 |
| 9 | John Smith | 3 | 4 |
| 4 | กมลวรรณ สุขใจ | 2 | **5** |
| 5 | ธนากร มั่งมี | 2 | **5** |
| 6 | ปิยะดา แสงทอง | 2 | **5** |
| 7 | อนุชา ทองดี | 2 | **5** |
| 8 | นภัสสร วงศ์สุวรรณ | 2 | **5** |
| 10 | Emily Johnson | 2 | **5** |
| 11 | Li Wei | 2 | **5** |
| 12 | Yuki Tanaka | 2 | **5** |
| 13 | ประภาส ศรีสุข | 2 | **5** |
| 14 | สุนิสา จันทร์เพ็ญ | 1 | **14** |
| 15 | David Lee | 1 | **14** |

สังเกตพฤติกรรมของ `RANK()`:
- ลูกค้า 10 คนที่มี `order_count = 2` **ทุกคนได้อันดับ 5 เท่ากัน**
- แถวถัดไป (ลูกค้าที่มี `order_count = 1`) ได้อันดับ **14** (ไม่ใช่ 6) เพราะข้ามเลข 6-13 ไปตามจำนวนแถวที่เสมอกันที่อันดับ 5 (มี 10 แถว → 5+10-1=14)

นี่คือพฤติกรรมแบบ "การแข่งขันกีฬา" (Olympic ranking / competition ranking) — ถ้ามีนักกีฬา 3 คนเข้าเส้นชัยพร้อมกันที่อันดับ 1 คนถัดไปจะได้อันดับ 4 เสมอ ไม่ใช่อันดับ 2

---

## Step 293: DENSE_RANK() — อันดับที่ไม่ข้ามเลข

`DENSE_RANK()` ทำงานคล้าย `RANK()` ทุกประการ ยกเว้นว่า **อันดับถัดไปจะไม่ข้ามเลข** แม้จะมีแถวเสมอกันหลายแถวก็ตาม — อันดับจะเรียงต่อเนื่องกันเสมอ (1, 2, 3, ... โดยไม่มีช่องว่าง)

มาดูข้อมูลชุดเดียวกันเทียบทั้งสามฟังก์ชันพร้อมกัน เพื่อเห็นความแตกต่างชัดเจน:

```sql
SELECT
    c.customer_id,
    c.first_name || ' ' || c.last_name AS customer_name,
    COUNT(o.order_id) AS order_count,
    RANK()       OVER (ORDER BY COUNT(o.order_id) DESC) AS rnk,
    DENSE_RANK() OVER (ORDER BY COUNT(o.order_id) DESC) AS dense_rnk,
    ROW_NUMBER() OVER (ORDER BY COUNT(o.order_id) DESC, c.customer_id) AS row_num
FROM customers c
JOIN orders o ON o.customer_id = c.customer_id
GROUP BY c.customer_id, customer_name
ORDER BY order_count DESC, c.customer_id;
```

**ผลลัพธ์ (เปรียบเทียบทั้งสามฟังก์ชัน):**

| customer_name | order_count | RANK | DENSE_RANK | ROW_NUMBER |
|---|---|---|---|---|
| สมชาย ใจดี | 7 | 1 | 1 | 1 |
| สมหญิง รักเรียน | 6 | 2 | 2 | 2 |
| วิชัย พัฒนา | 4 | 3 | 3 | 3 |
| John Smith | 3 | 4 | 4 | 4 |
| กมลวรรณ สุขใจ | 2 | 5 | **5** | 5 |
| ธนากร มั่งมี | 2 | 5 | **5** | 6 |
| ปิยะดา แสงทอง | 2 | 5 | **5** | 7 |
| อนุชา ทองดี | 2 | 5 | **5** | 8 |
| นภัสสร วงศ์สุวรรณ | 2 | 5 | **5** | 9 |
| Emily Johnson | 2 | 5 | **5** | 10 |
| Li Wei | 2 | 5 | **5** | 11 |
| Yuki Tanaka | 2 | 5 | **5** | 12 |
| ประภาส ศรีสุข | 2 | 5 | **5** | 13 |
| สุนิสา จันทร์เพ็ญ | 1 | **14** | **6** | 14 |
| David Lee | 1 | **14** | **6** | 15 |

จุดที่ต้องสังเกตให้ชัด:

| ฟังก์ชัน | ลูกค้าที่ order_count=2 (10 คน) | ลูกค้าที่ order_count=1 (2 คน) | เลขซ้ำได้ไหม | เลขข้ามช่องว่างไหม |
|---|---|---|---|---|
| `ROW_NUMBER()` | ได้เลข 5–14 (ไม่ซ้ำกันเลย) | ได้เลข 14, 15 | ไม่ได้ | — (เพราะไม่มีเลขซ้ำอยู่แล้ว) |
| `RANK()` | ได้อันดับ **5 ทุกคน** | ได้อันดับ **14** | ได้ | **ได้** (ข้ามอันดับ 6–13) |
| `DENSE_RANK()` | ได้อันดับ **5 ทุกคน** | ได้อันดับ **6** | ได้ | **ไม่ได้** (อันดับต่อเนื่อง) |

**เลือกใช้แบบไหนดี?**
- ใช้ `ROW_NUMBER()` เมื่อต้องการเลขลำดับที่ไม่ซ้ำแน่นอน (pagination, top-N ต่อกลุ่ม)
- ใช้ `RANK()` เมื่อโจทย์ทางธุรกิจสนใจ "อันดับจริง" แบบที่นับรวมคู่แข่งที่เสมอกันด้วย เช่น "ลูกค้าคนนี้อยู่อันดับที่เท่าไรในบรรดาลูกค้าทั้งหมด" (ถ้ามี 10 คนเสมอกันที่อันดับ 5 แปลว่ามี "ผู้ที่ดีกว่า" 4 คนจริง ๆ)
- ใช้ `DENSE_RANK()` เมื่อสนใจ "จำนวนระดับ/ชั้นที่แตกต่างกัน" มากกว่าจำนวนคู่แข่ง เช่น "มีสินค้ากี่ระดับราคาที่แตกต่างกัน" หรือทำ tier/level ของข้อมูล

---

## Step 294: PERCENT_RANK() และ CUME_DIST() — อันดับเชิงสัดส่วน

ฟังก์ชันทั้งสองตัวนี้ให้ผลลัพธ์เป็นสัดส่วน (0 ถึง 1) แทนที่จะเป็นเลขจำนวนเต็ม เหมาะกับการหา percentile ของข้อมูล

### PERCENT_RANK()

คำนวณจากสูตร:

```
PERCENT_RANK() = (rank - 1) / (จำนวนแถวทั้งหมดใน partition - 1)
```

แถวแรกสุด (อันดับ 1) จะได้ค่า `0` เสมอ และแถวสุดท้ายจะได้ค่า `1` เสมอ

### CUME_DIST() (Cumulative Distribution)

คำนวณจากสูตร:

```
CUME_DIST() = (จำนวนแถวที่มีค่า <= แถวปัจจุบัน) / (จำนวนแถวทั้งหมดใน partition)
```

ค่าที่ได้จะไม่มีวันเป็น 0 (แถวแรกก็ยังนับตัวเองรวมอยู่ในตัวตั้ง)

```sql
SELECT
    c.customer_id,
    c.first_name || ' ' || c.last_name AS customer_name,
    COALESCE(SUM(oi.quantity * oi.unit_price), 0) AS total_spent,
    ROUND(PERCENT_RANK() OVER (ORDER BY COALESCE(SUM(oi.quantity * oi.unit_price), 0))::numeric, 3) AS percent_rank,
    ROUND(CUME_DIST()    OVER (ORDER BY COALESCE(SUM(oi.quantity * oi.unit_price), 0))::numeric, 3) AS cume_dist
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.customer_id AND o.status <> 'cancelled'
LEFT JOIN order_items oi ON oi.order_id = o.order_id
GROUP BY c.customer_id, customer_name
ORDER BY total_spent;
```

**ผลลัพธ์:**

| customer_name | total_spent | percent_rank | cume_dist |
|---|---|---|---|
| สุนิสา จันทร์เพ็ญ | 1,990.00 | 0.000 | 0.067 |
| อนุชา ทองดี | 2,070.00 | 0.071 | 0.133 |
| ปิยะดา แสงทอง | 2,580.00 | 0.143 | 0.200 |
| นภัสสร วงศ์สุวรรณ | 5,980.00 | 0.214 | 0.267 |
| ประภาส ศรีสุข | 6,980.00 | 0.286 | 0.333 |
| กมลวรรณ สุขใจ | 7,080.00 | 0.357 | 0.400 |
| Yuki Tanaka | 13,980.00 | 0.429 | 0.467 |
| ธนากร มั่งมี | 14,680.00 | 0.500 | 0.533 |
| สมหญิง รักเรียน | 15,110.00 | 0.571 | 0.600 |
| Li Wei | 17,800.00 | 0.643 | 0.667 |
| David Lee | 24,900.00 | 0.714 | 0.733 |
| สมชาย ใจดี | 34,840.00 | 0.786 | 0.800 |
| Emily Johnson | 37,800.00 | 0.857 | 0.867 |
| John Smith | 67,800.00 | 0.929 | 0.933 |
| วิชัย พัฒนา | 91,970.00 | 1.000 | 1.000 |

จากตาราง: ลูกค้า `สุนิสา จันทร์เพ็ญ` มี `percent_rank = 0` (เป็นค่าต่ำสุด) และมี `cume_dist ≈ 0.067` หมายความว่ามีลูกค้าประมาณ 6.7% (คือตัวเองคนเดียวจาก 15 คน) ที่มียอดซื้อ "น้อยกว่างหรือเท่ากับ" เขา ส่วน `วิชัย พัฒนา` ที่ยอดซื้อสูงสุดจะได้ `percent_rank = 1` และ `cume_dist = 1` เสมอ (แถวสุดท้ายของทุก partition)

> **ข้อแตกต่างสำคัญ**: `PERCENT_RANK` ของแถวแรกจะเป็น `0` เสมอ ในขณะที่ `CUME_DIST` ของแถวแรกจะไม่เป็น `0` (เพราะนับรวมตัวมันเองด้วย) ทั้งสองฟังก์ชันนี้ **ไม่รับ argument** และต้องมี `ORDER BY` ใน `OVER()` เสมอ มิฉะนั้นทุกแถวจะถือว่าเสมอกันหมดและได้ค่า 1 ทั้งหมด (`CUME_DIST` แบบไม่มี order)

---

## Step 295: NTILE(n) — แบ่งข้อมูลเป็น n กลุ่มเท่า ๆ กัน

`NTILE(n)` แบ่งแถวทั้งหมดใน partition ออกเป็น `n` กลุ่ม (bucket) ที่มีขนาดใกล้เคียงกันที่สุด โดยกลุ่มแรก ๆ จะได้แถวมากกว่าถ้าหารไม่ลงตัว (เช่น 15 แถวแบ่งเป็น 4 กลุ่ม จะได้ 4,4,4,3 ไม่ใช่ 3,3,3,3,3)

ตัวอย่างคลาสสิก: แบ่งลูกค้าออกเป็น 4 กลุ่มตามยอดซื้อ (quartile) เพื่อทำ customer segmentation:

```sql
SELECT
    c.customer_id,
    c.first_name || ' ' || c.last_name AS customer_name,
    COALESCE(SUM(oi.quantity * oi.unit_price), 0) AS total_spent,
    NTILE(4) OVER (ORDER BY COALESCE(SUM(oi.quantity * oi.unit_price), 0) DESC) AS spend_quartile
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.customer_id AND o.status <> 'cancelled'
LEFT JOIN order_items oi ON oi.order_id = o.order_id
GROUP BY c.customer_id, customer_name
ORDER BY total_spent DESC;
```

**ผลลัพธ์:**

| customer_name | total_spent | spend_quartile |
|---|---|---|
| วิชัย พัฒนา | 91,970.00 | **1** |
| John Smith | 67,800.00 | **1** |
| Emily Johnson | 37,800.00 | **1** |
| สมชาย ใจดี | 34,840.00 | **1** |
| David Lee | 24,900.00 | 2 |
| Li Wei | 17,800.00 | 2 |
| สมหญิง รักเรียน | 15,110.00 | 2 |
| ธนากร มั่งมี | 14,680.00 | 2 |
| Yuki Tanaka | 13,980.00 | 3 |
| กมลวรรณ สุขใจ | 7,080.00 | 3 |
| ประภาส ศรีสุข | 6,980.00 | 3 |
| นภัสสร วงศ์สุวรรณ | 5,980.00 | 3 |
| ปิยะดา แสงทอง | 2,580.00 | **4** |
| อนุชา ทองดี | 2,070.00 | **4** |
| สุนิสา จันทร์เพ็ญ | 1,990.00 | **4** |

ลูกค้ามี 15 คน แบ่งเป็น 4 กลุ่ม → 15 ÷ 4 = 3 เศษ 3 ดังนั้น 3 กลุ่มแรกจะได้ 4 คน และกลุ่มสุดท้ายจะได้ 3 คน (4+4+4+3 = 15) ตรงกับผลลัพธ์ที่เห็น

จากนี้สามารถนำ `spend_quartile` ไปแปลงเป็นชื่อ segment ทางธุรกิจได้ทันทีด้วย `CASE`:

```sql
SELECT
    customer_name,
    total_spent,
    spend_quartile,
    CASE spend_quartile
        WHEN 1 THEN 'VIP'
        WHEN 2 THEN 'Gold'
        WHEN 3 THEN 'Silver'
        ELSE 'Bronze'
    END AS segment
FROM (
    SELECT
        c.first_name || ' ' || c.last_name AS customer_name,
        COALESCE(SUM(oi.quantity * oi.unit_price), 0) AS total_spent,
        NTILE(4) OVER (ORDER BY COALESCE(SUM(oi.quantity * oi.unit_price), 0) DESC) AS spend_quartile
    FROM customers c
    LEFT JOIN orders o ON o.customer_id = c.customer_id AND o.status <> 'cancelled'
    LEFT JOIN order_items oi ON oi.order_id = o.order_id
    GROUP BY c.customer_id, customer_name
) t
ORDER BY total_spent DESC;
```

> **ข้อควรระวัง**: `NTILE(n)` ไม่ได้แบ่งตามค่าจริง (value-based) แบบ `PERCENT_RANK` แต่แบ่งตาม **จำนวนแถว** (row count-based) ดังนั้นถ้าข้อมูลกระจุกตัวมาก (เช่น มีลูกค้า 10 คนยอดซื้อ 100 บาทเท่ากัน และอีก 5 คนยอดซื้อ 1 ล้านบาท) `NTILE(4)` ก็ยังจะพยายามแบ่งเป็น 4 กลุ่มเท่า ๆ กันตามจำนวนคน ไม่ใช่ตามช่วงของยอดซื้อ ทำให้บางกลุ่มอาจมีทั้งคนที่ยอดซื้อต่างกันมากปนอยู่ด้วยกัน — หากต้องการแบ่งตามช่วงค่าจริง ควรใช้ `CASE WHEN` กำหนดขอบเขตเอง หรือ `width_bucket()`

---

## Step 296: LAG() — ดึงค่าจากแถวก่อนหน้า

`LAG()` ดึงค่าจากแถวที่อยู่ **ก่อนหน้า** แถวปัจจุบัน (นับตาม `ORDER BY` ภายใน partition) มาแสดงในแถวปัจจุบัน เหมาะมากสำหรับการเปรียบเทียบค่าระหว่างช่วงเวลา เช่น เดือนนี้เทียบเดือนก่อน

รูปแบบ:

```
LAG(expression [, offset [, default]]) OVER (...)
```

- `offset` (ค่าเริ่มต้น = 1): ย้อนกลับไปกี่แถว
- `default` (ค่าเริ่มต้น = `NULL`): ค่าที่ใช้แทนเมื่อไม่มีแถวก่อนหน้า (เช่นแถวแรกสุดของ partition)

### ตัวอย่าง: เปรียบเทียบยอดขายเดือนนี้กับเดือนก่อน

```sql
WITH monthly_sales AS (
    SELECT
        date_trunc('month', o.order_date)::date AS sales_month,
        SUM(oi.quantity * oi.unit_price) AS revenue
    FROM orders o
    JOIN order_items oi ON oi.order_id = o.order_id
    WHERE o.status <> 'cancelled'
    GROUP BY sales_month
)
SELECT
    to_char(sales_month, 'YYYY-MM') AS month,
    revenue,
    LAG(revenue) OVER (ORDER BY sales_month) AS prev_month_revenue,
    revenue - LAG(revenue) OVER (ORDER BY sales_month) AS diff,
    ROUND(
        (revenue - LAG(revenue) OVER (ORDER BY sales_month))
        / LAG(revenue) OVER (ORDER BY sales_month) * 100,
        2
    ) AS pct_change
FROM monthly_sales
ORDER BY sales_month;
```

**ผลลัพธ์:**

| month | revenue | prev_month_revenue | diff | pct_change |
|---|---|---|---|---|
| 2025-01 | 88,100.00 | NULL | NULL | NULL |
| 2025-02 | 52,040.00 | 88,100.00 | -36,060.00 | -40.93 |
| 2025-03 | 56,300.00 | 52,040.00 | 4,260.00 | 8.19 |
| 2025-04 | 42,530.00 | 56,300.00 | -13,770.00 | -24.46 |
| 2025-05 | 68,450.00 | 42,530.00 | 25,920.00 | 60.95 |
| 2025-06 | 38,140.00 | 68,450.00 | -30,310.00 | -44.28 |

เดือนมกราคมไม่มีเดือนก่อนหน้าให้เทียบ จึงได้ `NULL` ทั้งสามคอลัมน์ — ตรงนี้เป็นพฤติกรรมปกติของ `LAG()` เมื่อไม่มีแถวก่อนหน้าใน partition

หากต้องการกำหนดค่าดีฟอลต์แทน `NULL` (เช่น ใส่ 0 แทน) สามารถระบุ argument ที่สามได้:

```sql
SELECT
    to_char(sales_month, 'YYYY-MM') AS month,
    revenue,
    LAG(revenue, 1, 0) OVER (ORDER BY sales_month) AS prev_month_revenue
FROM monthly_sales
ORDER BY sales_month;
```

### LAG() แบบมี PARTITION BY: เปรียบเทียบยอดซื้อของลูกค้าแต่ละคนกับครั้งก่อน

```sql
SELECT
    customer_id,
    order_id,
    order_date::date AS order_date,
    LAG(order_date::date) OVER (PARTITION BY customer_id ORDER BY order_date) AS prev_order_date
FROM orders
WHERE customer_id IN (1, 2)
ORDER BY customer_id, order_date;
```

ในกรณีนี้ `PARTITION BY customer_id` ทำให้ `LAG()` มองย้อนกลับเฉพาะภายในออเดอร์ของลูกค้าคนเดียวกันเท่านั้น ไม่ข้ามไปดึงข้อมูลจากลูกค้าคนอื่น

---

## Step 297: LEAD() — ดึงค่าจากแถวถัดไป

`LEAD()` ทำงานตรงข้ามกับ `LAG()` คือดึงค่าจากแถว **ถัดไป** มาแสดงในแถวปัจจุบัน syntax เหมือนกันทุกประการ:

```
LEAD(expression [, offset [, default]]) OVER (...)
```

### ตัวอย่าง: คำนวณระยะเวลาระหว่างคำสั่งซื้อของลูกค้าคนเดียวกัน (customer_id = 1)

```sql
SELECT
    customer_id,
    order_id,
    order_date::date AS order_date,
    LEAD(order_date::date) OVER (
        PARTITION BY customer_id ORDER BY order_date
    ) AS next_order_date,
    LEAD(order_date::date) OVER (
        PARTITION BY customer_id ORDER BY order_date
    ) - order_date::date AS days_to_next_order
FROM orders
WHERE customer_id = 1
ORDER BY order_date;
```

**ผลลัพธ์:**

| customer_id | order_id | order_date | next_order_date | days_to_next_order |
|---|---|---|---|---|
| 1 | 1 | 2025-01-05 | 2025-01-27 | 22 |
| 1 | 7 | 2025-01-27 | 2025-02-18 | 22 |
| 1 | 12 | 2025-02-18 | 2025-03-23 | 33 |
| 1 | 20 | 2025-03-23 | 2025-04-22 | 30 |
| 1 | 26 | 2025-04-22 | 2025-05-25 | 33 |
| 1 | 33 | 2025-05-25 | 2025-06-15 | 21 |
| 1 | 38 | 2025-06-15 | NULL | NULL |

ลูกค้า `สมชาย ใจดี` สั่งซื้อทุก ๆ 21-33 วันโดยเฉลี่ย ออเดอร์สุดท้าย (order_id = 38) ไม่มี "ออเดอร์ถัดไป" จึงได้ `NULL` — ข้อมูลแบบนี้มีประโยชน์มากในการวิเคราะห์ **customer purchase cycle** เช่น การส่งแคมเปญกระตุ้นให้ซื้อซ้ำก่อนที่รอบการซื้อปกติของลูกค้าจะมาถึง

### นำ LAG และ LEAD มาใช้ร่วมกัน

บางครั้งอยากรู้ทั้ง "ก่อนหน้า" และ "ถัดไป" ในคิวรีเดียว เพื่อดูว่าออเดอร์ปัจจุบันอยู่ตรงกลางของช่วงเวลาห่างแบบไหน:

```sql
SELECT
    order_id,
    order_date::date AS order_date,
    LAG(order_date::date)  OVER (PARTITION BY customer_id ORDER BY order_date) AS prev_date,
    LEAD(order_date::date) OVER (PARTITION BY customer_id ORDER BY order_date) AS next_date
FROM orders
WHERE customer_id = 1
ORDER BY order_date;
```

---

## Step 298: FIRST_VALUE(), LAST_VALUE(), NTH_VALUE() และ frame clause

ฟังก์ชันกลุ่มนี้ดึงค่าจากตำแหน่งเฉพาะภายใน **window frame** (ไม่ใช่ทั้ง partition เสมอไป — นี่คือจุดที่มือใหม่พลาดบ่อยที่สุด)

- `FIRST_VALUE(expr)` — ค่าจากแถวแรกสุดของ frame
- `LAST_VALUE(expr)` — ค่าจากแถวสุดท้ายของ frame
- `NTH_VALUE(expr, n)` — ค่าจากแถวที่ n ของ frame (นับจากแถวแรก)

### ทำความเข้าใจ default frame ก่อน

เมื่อระบุ `ORDER BY` ใน `OVER()` โดยไม่ระบุ frame clause อย่างชัดเจน PostgreSQL จะใช้ default frame เป็น:

```
RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
```

นั่นหมายความว่า frame ของแต่ละแถวจะขยายจาก "แถวแรกสุด" ไปถึง **"แถวปัจจุบัน" เท่านั้น** ไม่ใช่ถึงแถวสุดท้ายของ partition! นี่คือสาเหตุที่ `LAST_VALUE()` แบบไม่ระบุ frame มักให้ผลลัพธ์ที่ "ดูผิด" — มันไม่ได้ผิด แต่หมายถึง "ค่าสุดท้ายเท่าที่เห็น ณ แถวนี้" ซึ่งก็คือค่าของแถวปัจจุบันเองเสมอ

### ตัวอย่างข้อผิดพลาดยอดฮิต

```sql
-- ผิดพลาด: ลืมระบุ frame ให้ LAST_VALUE
SELECT
    category_id,
    product_name,
    unit_price,
    LAST_VALUE(product_name) OVER (
        PARTITION BY category_id ORDER BY unit_price
    ) AS wrong_last_value
FROM products
WHERE category_id = 4
ORDER BY category_id, unit_price;
```

**ผลลัพธ์ (ผิดที่คาดหวังไว้):**

| category_id | product_name | unit_price | wrong_last_value |
|---|---|---|---|
| 4 | Wireless Mouse M1 | 590.00 | Wireless Mouse M1 |
| 4 | USB-C Hub 7-in-1 | 890.00 | USB-C Hub 7-in-1 |
| 4 | Mechanical Keyboard K2 | 2490.00 | Mechanical Keyboard K2 |
| 4 | Bluetooth Earbuds Pro | 3290.00 | Bluetooth Earbuds Pro |

จะเห็นว่า `wrong_last_value` เท่ากับ `product_name` ของแถวนั้น ๆ เองทุกแถว! เพราะ default frame ขยายถึงแค่ "แถวปัจจุบัน" เท่านั้น ทำให้ "ค่าสุดท้ายของ frame" ก็คือค่าของแถวปัจจุบันนั่นเอง

### วิธีแก้: ระบุ frame ให้ครอบคลุมทั้ง partition

```sql
SELECT
    category_id,
    product_name,
    unit_price,
    FIRST_VALUE(product_name) OVER (
        PARTITION BY category_id ORDER BY unit_price
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS cheapest_in_category,
    LAST_VALUE(product_name) OVER (
        PARTITION BY category_id ORDER BY unit_price
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS priciest_in_category,
    NTH_VALUE(product_name, 2) OVER (
        PARTITION BY category_id ORDER BY unit_price
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS second_cheapest
FROM products
WHERE category_id IN (2, 4, 9)
ORDER BY category_id, unit_price;
```

**ผลลัพธ์ (ถูกต้อง):**

| category_id | product_name | unit_price | cheapest_in_category | priciest_in_category | second_cheapest |
|---|---|---|---|---|---|
| 2 | 27" 4K Monitor | 12,900.00 | 27" 4K Monitor | Laptop Pro 14" | Laptop Pro 14" |
| 2 | Laptop Pro 14" | 42,900.00 | 27" 4K Monitor | Laptop Pro 14" | Laptop Pro 14" |
| 4 | Wireless Mouse M1 | 590.00 | Wireless Mouse M1 | Bluetooth Earbuds Pro | USB-C Hub 7-in-1 |
| 4 | USB-C Hub 7-in-1 | 890.00 | Wireless Mouse M1 | Bluetooth Earbuds Pro | USB-C Hub 7-in-1 |
| 4 | Mechanical Keyboard K2 | 2,490.00 | Wireless Mouse M1 | Bluetooth Earbuds Pro | USB-C Hub 7-in-1 |
| 4 | Bluetooth Earbuds Pro | 3,290.00 | Wireless Mouse M1 | Bluetooth Earbuds Pro | USB-C Hub 7-in-1 |
| 9 | Yoga Mat Premium | 690.00 | Yoga Mat Premium | Running Shoes Elite | Running Shoes Elite |
| 9 | Running Shoes Elite | 3,490.00 | Yoga Mat Premium | Running Shoes Elite | Running Shoes Elite |

คราวนี้ `priciest_in_category` แสดงสินค้าที่แพงที่สุดในทุกแถวของ partition ได้ถูกต้องแล้ว เพราะ frame ถูกขยายให้ครอบคลุม `UNBOUNDED FOLLOWING` ด้วย

> **สรุปกฎเหล็ก**: ทุกครั้งที่ใช้ `LAST_VALUE()` (และควรใช้กับ `FIRST_VALUE()`/`NTH_VALUE()` เพื่อความชัดเจนด้วย) ให้ใส่ frame clause `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING` เสมอ ถ้าต้องการให้ผลลัพธ์อ้างอิงทั้ง partition ไม่ใช่แค่ "จนถึงแถวปัจจุบัน"
>
> **ข้อควรระวังเพิ่มเติมเรื่อง RANGE vs ROWS**: เมื่อ `ORDER BY` มีค่าที่ซ้ำกัน (peer rows) `RANGE` จะถือว่าทุกแถวที่มีค่า order-by เท่ากันอยู่ใน frame เดียวกันเสมอ (ไม่ตัดกลางกลุ่ม peer) ในขณะที่ `ROWS` จะนับทีละแถวตามตำแหน่งจริงโดยไม่สนใจว่าค่าจะเท่ากันหรือไม่ — โดยทั่วไปแนะนำให้ใช้ `ROWS` เมื่อทำงานกับ `FIRST_VALUE`/`LAST_VALUE`/`NTH_VALUE` เพราะพฤติกรรมเข้าใจง่ายและคาดเดาได้กว่า

---

## Step 299: รวม Window Function หลายตัวสร้าง Analytics ที่ซับซ้อน

พลังที่แท้จริงของ window function อยู่ที่การนำหลาย ๆ ฟังก์ชันมาผสมกันใน query เดียว เพื่อสร้างรายงานวิเคราะห์แบบ Business Intelligence ตัวอย่างต่อไปนี้รวม `NTILE`, `RANK`, `SUM() OVER()` เข้าด้วยกันเพื่อทำ **customer segmentation report** ที่สมบูรณ์:

```sql
WITH customer_spend AS (
    SELECT
        c.customer_id,
        c.first_name || ' ' || c.last_name AS customer_name,
        c.country,
        COALESCE(SUM(oi.quantity * oi.unit_price), 0) AS total_spent,
        COUNT(DISTINCT o.order_id) AS order_count
    FROM customers c
    LEFT JOIN orders o ON o.customer_id = c.customer_id AND o.status <> 'cancelled'
    LEFT JOIN order_items oi ON oi.order_id = o.order_id
    GROUP BY c.customer_id, customer_name, c.country
)
SELECT
    customer_name,
    country,
    total_spent,
    order_count,
    NTILE(4) OVER (ORDER BY total_spent DESC) AS spend_quartile,
    CASE NTILE(4) OVER (ORDER BY total_spent DESC)
        WHEN 1 THEN 'VIP'
        WHEN 2 THEN 'Gold'
        WHEN 3 THEN 'Silver'
        ELSE 'Bronze'
    END AS segment,
    RANK() OVER (ORDER BY total_spent DESC) AS spend_rank,
    ROUND(100 * total_spent / SUM(total_spent) OVER (), 2) AS pct_of_total_revenue
FROM customer_spend
ORDER BY total_spent DESC;
```

**ผลลัพธ์:**

| customer_name | country | total_spent | order_count | spend_quartile | segment | spend_rank | pct_of_total_revenue |
|---|---|---|---|---|---|---|---|
| วิชัย พัฒนา | Thailand | 91,970.00 | 4 | 1 | VIP | 1 | 26.61 |
| John Smith | USA | 67,800.00 | 2 | 1 | VIP | 2 | 19.62 |
| Emily Johnson | USA | 37,800.00 | 2 | 1 | VIP | 3 | 10.94 |
| สมชาย ใจดี | Thailand | 34,840.00 | 7 | 1 | VIP | 4 | 10.08 |
| David Lee | Singapore | 24,900.00 | 1 | 2 | Gold | 5 | 7.21 |
| Li Wei | China | 17,800.00 | 2 | 2 | Gold | 6 | 5.15 |
| สมหญิง รักเรียน | Thailand | 15,110.00 | 6 | 2 | Gold | 7 | 4.37 |
| ธนากร มั่งมี | Thailand | 14,680.00 | 2 | 2 | Gold | 8 | 4.25 |
| Yuki Tanaka | Japan | 13,980.00 | 2 | 3 | Silver | 9 | 4.05 |
| กมลวรรณ สุขใจ | Thailand | 7,080.00 | 2 | 3 | Silver | 10 | 2.05 |
| ประภาส ศรีสุข | Thailand | 6,980.00 | 2 | 3 | Silver | 11 | 2.02 |
| นภัสสร วงศ์สุวรรณ | Thailand | 5,980.00 | 2 | 3 | Silver | 12 | 1.73 |
| ปิยะดา แสงทอง | Thailand | 2,580.00 | 2 | 4 | Bronze | 13 | 0.75 |
| อนุชา ทองดี | Thailand | 2,070.00 | 2 | 4 | Bronze | 14 | 0.60 |
| สุนิสา จันทร์เพ็ญ | Thailand | 1,990.00 | 1 | 4 | Bronze | 15 | 0.58 |

รายงานนี้บอกอะไรได้หลายอย่างในคิวรีเดียว: ลูกค้ากลุ่ม VIP (quartile 1) สร้างรายได้รวมกันถึง 67.25% ของรายได้ทั้งหมด (26.61+19.62+10.94+10.08) — เป็นข้อมูลเชิงลึกแบบ Pareto (80/20) ที่ผู้บริหารสนใจมาก

### ตัวอย่างที่สอง: Month-over-Month Growth แยกตามหมวดหมู่สินค้า

```sql
WITH category_monthly AS (
    SELECT
        cat.category_name,
        date_trunc('month', o.order_date)::date AS sales_month,
        SUM(oi.quantity * oi.unit_price) AS revenue
    FROM order_items oi
    JOIN orders o ON o.order_id = oi.order_id
    JOIN products p ON p.product_id = oi.product_id
    JOIN categories cat ON cat.category_id = p.category_id
    WHERE o.status <> 'cancelled'
    GROUP BY cat.category_name, sales_month
)
SELECT
    category_name,
    to_char(sales_month, 'YYYY-MM') AS month,
    revenue,
    LAG(revenue) OVER (PARTITION BY category_name ORDER BY sales_month) AS prev_revenue,
    ROUND(
        100.0 * (revenue - LAG(revenue) OVER (PARTITION BY category_name ORDER BY sales_month))
        / NULLIF(LAG(revenue) OVER (PARTITION BY category_name ORDER BY sales_month), 0),
        2
    ) AS mom_growth_pct
FROM category_monthly
ORDER BY category_name, sales_month;
```

คิวรีนี้ใช้ `PARTITION BY category_name` เพื่อให้ `LAG()` เปรียบเทียบเฉพาะภายในหมวดหมู่เดียวกัน ไม่ปะปนข้ามหมวด และใช้ `NULLIF(..., 0)` เพื่อป้องกัน division by zero เมื่อเดือนก่อนหน้าไม่มียอดขายเลย (revenue = 0)

> **เทคนิคสำคัญ**: เมื่อผสม window function หลายตัวใน `SELECT` เดียวกัน ควรจัดกลุ่มด้วย CTE (`WITH`) ก่อนเพื่อให้ query อ่านง่าย และหลีกเลี่ยงการเขียน expression ที่ซับซ้อนซ้ำ ๆ หลายครั้ง (เช่นในตัวอย่างข้างบนที่ `LAG(revenue) OVER (...)` ถูกเขียนซ้ำ) — หากต้องการความกระชับยิ่งขึ้น สามารถใช้ named window ผ่าน `WINDOW` clause ได้ เช่น:

```sql
SELECT
    category_name,
    to_char(sales_month, 'YYYY-MM') AS month,
    revenue,
    LAG(revenue) OVER w AS prev_revenue,
    revenue - LAG(revenue) OVER w AS diff
FROM category_monthly
WINDOW w AS (PARTITION BY category_name ORDER BY sales_month)
ORDER BY category_name, sales_month;
```

`WINDOW w AS (...)` ช่วยลดความซ้ำซ้อนเมื่อต้องใช้ window definition เดียวกันกับหลายฟังก์ชันในคิวรีเดียว

---

## Step 300: แบบฝึกหัดรวม — โจทย์วิเคราะห์ข้อมูลระดับ Business Intelligence

มาดูโจทย์ระดับ BI ที่รวมแทบทุก window function ที่เรียนมาในบทนี้เข้าด้วยกัน — รายงาน "สรุปผลประกอบการรายเดือนของบริษัท" ที่ทีมผู้บริหารต้องการเห็นในที่เดียว:

```sql
WITH monthly AS (
    SELECT
        date_trunc('month', o.order_date)::date AS sales_month,
        SUM(oi.quantity * oi.unit_price) AS revenue
    FROM orders o
    JOIN order_items oi ON oi.order_id = o.order_id
    WHERE o.status <> 'cancelled'
    GROUP BY sales_month
)
SELECT
    to_char(sales_month, 'YYYY-MM') AS month,
    revenue,
    SUM(revenue) OVER (ORDER BY sales_month) AS running_total,
    ROUND(
        100.0 * (revenue - LAG(revenue) OVER (ORDER BY sales_month))
        / LAG(revenue) OVER (ORDER BY sales_month),
        2
    ) AS mom_growth_pct,
    RANK() OVER (ORDER BY revenue DESC) AS revenue_rank,
    ROUND(PERCENT_RANK() OVER (ORDER BY revenue)::numeric, 3) AS pct_rank
FROM monthly
ORDER BY sales_month;
```

**ผลลัพธ์:**

| month | revenue | running_total | mom_growth_pct | revenue_rank | pct_rank |
|---|---|---|---|---|---|
| 2025-01 | 88,100.00 | 88,100.00 | NULL | 1 | 1.000 |
| 2025-02 | 52,040.00 | 140,140.00 | -40.93 | 4 | 0.400 |
| 2025-03 | 56,300.00 | 196,440.00 | 8.19 | 3 | 0.600 |
| 2025-04 | 42,530.00 | 238,970.00 | -24.46 | 5 | 0.200 |
| 2025-05 | 68,450.00 | 307,420.00 | 60.95 | 2 | 0.800 |
| 2025-06 | 38,140.00 | 345,560.00 | -44.28 | 6 | 0.000 |

รายงานนี้ตอบคำถามทางธุรกิจได้พร้อมกันหลายข้อในคิวรีเดียว:
- **รายได้สะสม** (`running_total`) ใช้ `SUM() OVER (ORDER BY ...)` แบบไม่มี frame ชัดเจน (ใช้ default frame `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` ซึ่งเหมาะกับ running total พอดี)
- **การเติบโตเทียบเดือนก่อน** (`mom_growth_pct`) ใช้ `LAG()`
- **อันดับเดือนที่ขายดีที่สุด** (`revenue_rank`) ใช้ `RANK()` — เดือนพฤษภาคมเป็นเดือนที่ขายดีเป็นอันดับ 2 แม้จะมาหลังเดือนที่ตกต่ำ (เมษายน)
- **เปอร์เซ็นไทล์ของแต่ละเดือน** (`pct_rank`) ใช้ `PERCENT_RANK()` — เดือนมิถุนายนแย่ที่สุด (0.000) ส่วนมกราคมดีที่สุด (1.000)

จากตัวเลขจะเห็น pattern ที่น่าสนใจ: ธุรกิจนี้มียอดขายแบบ "ฟันปลา" (sawtooth) คือขึ้น-ลงสลับกันเกือบทุกเดือน ไม่มีแนวโน้มเติบโตต่อเนื่องชัดเจน ซึ่งเป็น insight ที่ได้จาก window function ล้วน ๆ โดยไม่ต้องดึงข้อมูลออกไปวิเคราะห์ในเครื่องมืออื่น

---

## สรุปท้ายบท

ตารางเปรียบเทียบ window function จัดอันดับทั้งสามตัวที่ใช้บ่อยที่สุด:

| คุณสมบัติ | `ROW_NUMBER()` | `RANK()` | `DENSE_RANK()` |
|---|---|---|---|
| ค่าที่เท่ากันได้เลขเดียวกันหรือไม่ | ไม่ได้ (ไม่ซ้ำเสมอ) | ได้ | ได้ |
| อันดับถัดไปข้ามเลขหรือไม่เมื่อมี tie | ไม่เกี่ยวข้อง (ไม่มี tie) | **ข้าม** เท่าจำนวนแถวที่เสมอกัน | **ไม่ข้าม** เรียงต่อเนื่อง |
| เหมาะกับ | pagination, top-N ต่อกลุ่ม, DISTINCT ON ทางเลือก | อันดับจริงที่นับคู่แข่งที่เสมอกันด้วย (competition ranking) | นับจำนวน "ระดับ/ชั้น" ที่แตกต่างกัน (tiering) |
| ต้องมี `ORDER BY` ใน `OVER()` ไหม | ควรมี (มิฉะนั้นลำดับไม่แน่นอน) | ต้องมี | ต้องมี |

สรุปฟังก์ชันอื่น ๆ ที่เรียนในบทนี้:

| ฟังก์ชัน | หน้าที่ | ค่าที่ได้ |
|---|---|---|
| `PERCENT_RANK()` | อันดับเชิงสัดส่วน (แถวแรก = 0, แถวสุดท้าย = 1) | 0 ถึง 1 |
| `CUME_DIST()` | สัดส่วนแถวที่มีค่า ≤ แถวปัจจุบัน | มากกว่า 0 ถึง 1 |
| `NTILE(n)` | แบ่งแถวเป็น n กลุ่มขนาดใกล้เคียงกัน | เลขจำนวนเต็ม 1 ถึง n |
| `LAG(expr, offset, default)` | ดึงค่าจากแถวก่อนหน้า | ชนิดเดียวกับ expr หรือ default |
| `LEAD(expr, offset, default)` | ดึงค่าจากแถวถัดไป | ชนิดเดียวกับ expr หรือ default |
| `FIRST_VALUE(expr)` | ค่าจากแถวแรกของ frame | ชนิดเดียวกับ expr |
| `LAST_VALUE(expr)` | ค่าจากแถวสุดท้ายของ frame (**ต้องระบุ frame ให้ครอบคลุมทั้ง partition ถ้าต้องการค่าจริงของ partition**) | ชนิดเดียวกับ expr |
| `NTH_VALUE(expr, n)` | ค่าจากแถวที่ n ของ frame | ชนิดเดียวกับ expr |

**หลักการจำง่าย ๆ**:
1. ถ้าอยากได้ "แถวล่าสุด/แถวแรกสุดต่อกลุ่ม" → `ROW_NUMBER()` (หรือ `DISTINCT ON` ถ้าใช้ PostgreSQL เท่านั้น)
2. ถ้าอยากได้ "อันดับ" ที่มีความหมายทางสถิติ → `RANK()` / `DENSE_RANK()` แล้วแต่ว่าสนใจคู่แข่งหรือสนใจระดับชั้น
3. ถ้าอยากได้ "เปอร์เซ็นไทล์" → `PERCENT_RANK()` / `CUME_DIST()`
4. ถ้าอยากแบ่งกลุ่มเท่า ๆ กัน → `NTILE(n)`
5. ถ้าอยากเทียบกับแถวก่อน/หลัง → `LAG()` / `LEAD()`
6. ถ้าอยากได้ค่าตำแหน่งเฉพาะของกลุ่ม (แรก/สุดท้าย/ที่ n) → `FIRST_VALUE()` / `LAST_VALUE()` / `NTH_VALUE()` **พร้อมระบุ frame เสมอ**

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1

เขียน query หาสินค้าขายดีที่สุด 3 อันดับแรก (วัดจากจำนวนที่ขายได้รวม `quantity`) โดยใช้ `RANK()` และแสดงเฉพาะแถวที่ `rnk <= 3`

<details>
<summary>เฉลย</summary>

```sql
SELECT product_name, total_qty, rnk
FROM (
    SELECT
        p.product_name,
        SUM(oi.quantity) AS total_qty,
        RANK() OVER (ORDER BY SUM(oi.quantity) DESC) AS rnk
    FROM products p
    JOIN order_items oi ON oi.product_id = p.product_id
    GROUP BY p.product_name
) t
WHERE rnk <= 3
ORDER BY rnk, product_name;
```

**ผลลัพธ์:**

| product_name | total_qty | rnk |
|---|---|---|
| Wireless Mouse M1 | 6 | 1 |
| Cotton T-Shirt (5-pack) | 5 | 2 |
| Smartphone X12 | 4 | 3 |
| USB-C Hub 7-in-1 | 4 | 3 |

สังเกตว่ามี 2 สินค้าที่เสมอกันในอันดับ 3 (`Smartphone X12` และ `USB-C Hub 7-in-1` ขายได้ 4 ชิ้นเท่ากัน) — ถ้าใช้ `DENSE_RANK()` แทน จะได้อันดับ 1,2,3,3 เหมือนกัน แต่ถ้ามีสินค้าอันดับถัดไปจะได้อันดับ 4 (ไม่ใช่ 5 แบบ `RANK()`)

</details>

### แบบฝึกหัดที่ 2

เขียน query แบ่งหน้า (pagination) แสดงสินค้าที่ active ทั้งหมด เรียงตามราคาจากถูกไปแพง หน้าละ 5 รายการ โดยขอดู "หน้าที่ 2" (แถวที่ 6-10) ด้วย `ROW_NUMBER()`

<details>
<summary>เฉลย</summary>

```sql
SELECT product_id, product_name, unit_price, rn
FROM (
    SELECT
        product_id,
        product_name,
        unit_price,
        ROW_NUMBER() OVER (ORDER BY unit_price ASC, product_id) AS rn
    FROM products
    WHERE is_active = true
) numbered
WHERE rn BETWEEN 6 AND 10
ORDER BY rn;
```

การกำหนด `rn BETWEEN (page-1)*page_size + 1 AND page*page_size` คือสูตรทั่วไปในการทำ pagination ด้วย `ROW_NUMBER()` — สำหรับหน้า 2 ขนาดหน้าละ 5: `BETWEEN 6 AND 10`

</details>

### แบบฝึกหัดที่ 3

เขียน query จัดอันดับลูกค้าตามยอดซื้อ **แยกตามประเทศ** (ใช้ `DENSE_RANK()` แบบมี `PARTITION BY country`) แสดงเฉพาะลูกค้าจากประเทศไทย

<details>
<summary>เฉลย</summary>

```sql
WITH customer_spend AS (
    SELECT
        c.customer_id,
        c.first_name || ' ' || c.last_name AS name,
        c.country,
        COALESCE(SUM(oi.quantity * oi.unit_price), 0) AS total_spent
    FROM customers c
    LEFT JOIN orders o ON o.customer_id = c.customer_id AND o.status <> 'cancelled'
    LEFT JOIN order_items oi ON oi.order_id = o.order_id
    GROUP BY c.customer_id, name, c.country
)
SELECT
    name,
    country,
    total_spent,
    DENSE_RANK() OVER (PARTITION BY country ORDER BY total_spent DESC) AS rank_in_country
FROM customer_spend
WHERE country = 'Thailand'
ORDER BY total_spent DESC;
```

**ผลลัพธ์ (10 แถวแรก):**

| name | country | total_spent | rank_in_country |
|---|---|---|---|
| วิชัย พัฒนา | Thailand | 91,970.00 | 1 |
| สมชาย ใจดี | Thailand | 34,840.00 | 2 |
| สมหญิง รักเรียน | Thailand | 15,110.00 | 3 |
| ธนากร มั่งมี | Thailand | 14,680.00 | 4 |
| กมลวรรณ สุขใจ | Thailand | 7,080.00 | 5 |
| ประภาส ศรีสุข | Thailand | 6,980.00 | 6 |
| นภัสสร วงศ์สุวรรณ | Thailand | 5,980.00 | 7 |
| ปิยะดา แสงทอง | Thailand | 2,580.00 | 8 |
| อนุชา ทองดี | Thailand | 2,070.00 | 9 |
| สุนิสา จันทร์เพ็ญ | Thailand | 1,990.00 | 10 |

`PARTITION BY country` ทำให้อันดับเริ่มนับใหม่ทุกประเทศ — ในที่นี้ไม่มีค่าที่ยอดซื้อเท่ากันพอดีในกลุ่มไทย จึงไม่เห็นความต่างระหว่าง `RANK` กับ `DENSE_RANK` ในผลลัพธ์นี้ (ลองเปลี่ยนไปใช้ `RANK()` แล้วเทียบดูเป็นแบบฝึกหัดเพิ่มเติม)

</details>

### แบบฝึกหัดที่ 4

เขียน query แบ่งสินค้า active ทั้งหมดออกเป็น 3 กลุ่มราคา (cheap/mid/premium) ด้วย `NTILE(3)`

<details>
<summary>เฉลย</summary>

```sql
SELECT
    product_name,
    unit_price,
    NTILE(3) OVER (ORDER BY unit_price) AS price_tier
FROM products
WHERE is_active = true
ORDER BY unit_price;
```

**ผลลัพธ์ (19 สินค้า active, แบ่งเป็น 3 กลุ่ม: 7/6/6):**

| product_name | unit_price | price_tier |
|---|---|---|
| Cotton T-Shirt (5-pack) | 590.00 | 1 |
| Wireless Mouse M1 | 590.00 | 1 |
| Yoga Mat Premium | 690.00 | 1 |
| The Data Engineer's Handbook | 750.00 | 1 |
| PostgreSQL for Developers | 890.00 | 1 |
| USB-C Hub 7-in-1 | 890.00 | 1 |
| Building Blocks Set 500pcs | 1,190.00 | 1 |
| Non-stick Pan Set | 1,290.00 | 2 |
| Denim Jacket | 1,990.00 | 2 |
| Mechanical Keyboard K2 | 2,490.00 | 2 |
| Air Fryer 5L | 2,990.00 | 2 |
| Bluetooth Earbuds Pro | 3,290.00 | 2 |
| Running Shoes Elite | 3,490.00 | 2 |
| Study Desk Oak | 4,590.00 | 3 |
| Gaming Chair Deluxe | 6,990.00 | 3 |
| Smartphone Lite | 8,900.00 | 3 |
| 27" 4K Monitor | 12,900.00 | 3 |
| Smartphone X12 | 24,900.00 | 3 |
| Laptop Pro 14" | 42,900.00 | 3 |

19 สินค้า ÷ 3 กลุ่ม = 6 เศษ 1 → กลุ่มแรกได้ 7 ชิ้น กลุ่มที่เหลือได้กลุ่มละ 6 ชิ้น (7+6+6=19)

</details>

### แบบฝึกหัดที่ 5

เขียน query แสดง % การเติบโตของยอดขายรายเดือน **เฉพาะหมวดหมู่ Electronics** (category_id 1, 2, 3, 4 รวมกัน) โดยใช้ `LAG()`

<details>
<summary>เฉลย</summary>

```sql
WITH electronics_monthly AS (
    SELECT
        date_trunc('month', o.order_date)::date AS sales_month,
        SUM(oi.quantity * oi.unit_price) AS revenue
    FROM order_items oi
    JOIN orders o ON o.order_id = oi.order_id
    JOIN products p ON p.product_id = oi.product_id
    WHERE p.category_id IN (1, 2, 3, 4)
      AND o.status <> 'cancelled'
    GROUP BY sales_month
)
SELECT
    to_char(sales_month, 'YYYY-MM') AS month,
    revenue,
    LAG(revenue) OVER (ORDER BY sales_month) AS prev_revenue,
    ROUND(
        100.0 * (revenue - LAG(revenue) OVER (ORDER BY sales_month))
        / NULLIF(LAG(revenue) OVER (ORDER BY sales_month), 0),
        2
    ) AS growth_pct
FROM electronics_monthly
ORDER BY sales_month;
```

หมายเหตุ: ใช้ `NULLIF(..., 0)` เพื่อป้องกันข้อผิดพลาด division by zero หากมีเดือนที่ไม่มียอดขายในหมวด Electronics เลย

</details>

### แบบฝึกหัดที่ 6

เขียน query หาว่าลูกค้าแต่ละคนเขียนรีวิวห่างกันกี่วัน (ใช้ `LEAD()`) สำหรับลูกค้าที่เขียนรีวิวมากกว่า 1 ครั้ง

<details>
<summary>เฉลย</summary>

```sql
SELECT
    customer_id,
    review_id,
    review_date,
    product_id,
    LEAD(review_date) OVER (PARTITION BY customer_id ORDER BY review_date) AS next_review_date,
    LEAD(review_date) OVER (PARTITION BY customer_id ORDER BY review_date) - review_date
        AS days_until_next_review
FROM reviews
ORDER BY customer_id, review_date;
```

**ผลลัพธ์ (เฉพาะลูกค้าที่มีมากกว่า 1 รีวิว):**

| customer_id | review_id | review_date | next_review_date | days_until_next_review |
|---|---|---|---|---|
| 1 | 3 | 2025-01-10 | 2025-01-30 | 20 |
| 1 | 13 | 2025-01-30 | 2025-06-16 | 137 |
| 1 | 17 | 2025-06-16 | NULL | NULL |
| 2 | 6 | 2025-02-10 | 2025-02-12 | 2 |
| 2 | 9 | 2025-02-12 | NULL | NULL |
| 3 | 1 | 2025-01-20 | 2025-06-26 | 157 |
| 3 | 18 | 2025-06-26 | NULL | NULL |
| 5 | 7 | 2025-02-08 | 2025-04-20 | 71 |
| 5 | 14 | 2025-04-20 | NULL | NULL |
| 7 | 10 | 2025-02-28 | 2025-05-23 | 84 |
| 7 | 16 | 2025-05-23 | NULL | NULL |

ลูกค้า `id=2` เขียนรีวิว 2 ครั้งห่างกันแค่ 2 วัน ในขณะที่ลูกค้าคนอื่นเขียนรีวิวห่างกันเป็นเดือน — ข้อมูลแบบนี้ช่วยทีม Product วิเคราะห์ "engagement pattern" ของลูกค้าที่ชอบรีวิวได้

</details>

### แบบฝึกหัดที่ 7

เขียน query เปรียบเทียบราคาสินค้าที่ลูกค้าซื้อในแต่ละ `order_item` กับราคาสินค้าที่ **ถูกที่สุดในหมวดหมู่เดียวกัน** โดยใช้ `FIRST_VALUE()` พร้อม frame ที่ถูกต้อง

<details>
<summary>เฉลย</summary>

```sql
SELECT
    p.product_name,
    cat.category_name,
    p.unit_price,
    FIRST_VALUE(p.unit_price) OVER (
        PARTITION BY cat.category_id ORDER BY p.unit_price
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS cheapest_price_in_category,
    p.unit_price - FIRST_VALUE(p.unit_price) OVER (
        PARTITION BY cat.category_id ORDER BY p.unit_price
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS price_gap
FROM products p
JOIN categories cat ON cat.category_id = p.category_id
WHERE p.is_active = true
ORDER BY cat.category_id, p.unit_price;
```

คิวรีนี้แสดงให้เห็นว่าสินค้าแต่ละชิ้นแพงกว่าสินค้าที่ถูกที่สุดในหมวดเดียวกันเท่าไร — มีประโยชน์สำหรับทีม pricing ที่อยากรู้ว่ามี "ช่องว่างราคา" มากแค่ไหนภายในแต่ละหมวดหมู่สินค้า

</details>

### แบบฝึกหัดที่ 8

เขียน query หาสินค้า active ที่มี `stock_quantity` น้อยที่สุด 8 อันดับแรก พร้อมคำนวณ `PERCENT_RANK()` ของ stock เพื่อดูว่าสินค้านั้นอยู่ในเปอร์เซ็นไทล์ที่เท่าไรของสต็อกทั้งหมด

<details>
<summary>เฉลย</summary>

```sql
SELECT
    product_name,
    stock_quantity,
    ROUND(PERCENT_RANK() OVER (ORDER BY stock_quantity)::numeric, 3) AS pct_rank_stock
FROM products
WHERE is_active = true
ORDER BY stock_quantity
LIMIT 8;
```

**ผลลัพธ์:**

| product_name | stock_quantity | pct_rank_stock |
|---|---|---|
| Gaming Chair Deluxe | 15 | 0.000 |
| Study Desk Oak | 20 | 0.056 |
| Laptop Pro 14" | 25 | 0.111 |
| 27" 4K Monitor | 30 | 0.167 |
| Air Fryer 5L | 35 | 0.222 |
| Denim Jacket | 40 | 0.278 |
| Smartphone X12 | 40 | 0.278 |
| Building Blocks Set 500pcs | 45 | 0.389 |

สังเกตว่า `Denim Jacket` และ `Smartphone X12` มี `stock_quantity = 40` เท่ากัน จึงได้ `pct_rank_stock` เท่ากัน (0.278) — พฤติกรรมนี้สอดคล้องกับหลักการของ `PERCENT_RANK()` ที่คำนวณจาก rank ซึ่งมีการจัดการค่าเสมอกันแบบเดียวกับ `RANK()`

</details>

### แบบฝึกหัดที่ 9

เขียน query หาสัดส่วน (`CUME_DIST()`) ของพนักงานแต่ละคนตามจำนวนออเดอร์ที่รับผิดชอบ

<details>
<summary>เฉลย</summary>

```sql
SELECT
    e.first_name || ' ' || e.last_name AS employee_name,
    COUNT(o.order_id) AS orders_handled,
    ROUND(CUME_DIST() OVER (ORDER BY COUNT(o.order_id))::numeric, 3) AS cume_dist
FROM employees e
JOIN orders o ON o.employee_id = e.employee_id
GROUP BY e.employee_id, employee_name
ORDER BY orders_handled;
```

**ผลลัพธ์:**

| employee_name | orders_handled | cume_dist |
|---|---|---|
| ณัฐพล รุ่งเรือง | 5 | 0.250 |
| อรทัย มีสุข | 9 | 0.500 |
| ธีรพงษ์ ยอดขยัน | 11 | 0.750 |
| สุภาพร เก่งงาน | 15 | 1.000 |

มีพนักงานฝ่ายขาย 4 คนที่รับออเดอร์ (พนักงานฝ่าย Support ไม่ปรากฏเพราะไม่มี `employee_id` ผูกกับตาราง `orders`) — `สุภาพร เก่งงาน` รับออเดอร์มากที่สุดจึงมี `cume_dist = 1.000` เสมอ (ตำแหน่งสุดท้ายของ partition)

</details>

### แบบฝึกหัดที่ 10 (โจทย์รวม)

สร้างรายงาน "Executive Dashboard" หนึ่งคิวรีที่แสดง: รายได้รายเดือน, รายได้สะสม (running total), % การเติบโตจากเดือนก่อน, อันดับเดือนที่ขายดีที่สุด (แบบข้ามเลขเมื่อเสมอ), และเปอร์เซ็นไทล์ของแต่ละเดือนเทียบกับเดือนอื่นทั้งหมด — ใช้ window function อย่างน้อย 4 ตัวจากที่เรียนมาในบทนี้

<details>
<summary>เฉลย</summary>

```sql
WITH monthly AS (
    SELECT
        date_trunc('month', o.order_date)::date AS sales_month,
        SUM(oi.quantity * oi.unit_price) AS revenue
    FROM orders o
    JOIN order_items oi ON oi.order_id = o.order_id
    WHERE o.status <> 'cancelled'
    GROUP BY sales_month
)
SELECT
    to_char(sales_month, 'YYYY-MM') AS month,
    revenue,
    SUM(revenue) OVER (ORDER BY sales_month) AS running_total,
    ROUND(
        100.0 * (revenue - LAG(revenue) OVER (ORDER BY sales_month))
        / LAG(revenue) OVER (ORDER BY sales_month),
        2
    ) AS mom_growth_pct,
    RANK() OVER (ORDER BY revenue DESC) AS revenue_rank,
    ROUND(PERCENT_RANK() OVER (ORDER BY revenue)::numeric, 3) AS pct_rank
FROM monthly
ORDER BY sales_month;
```

**ผลลัพธ์:**

| month | revenue | running_total | mom_growth_pct | revenue_rank | pct_rank |
|---|---|---|---|---|---|
| 2025-01 | 88,100.00 | 88,100.00 | NULL | 1 | 1.000 |
| 2025-02 | 52,040.00 | 140,140.00 | -40.93 | 4 | 0.400 |
| 2025-03 | 56,300.00 | 196,440.00 | 8.19 | 3 | 0.600 |
| 2025-04 | 42,530.00 | 238,970.00 | -24.46 | 5 | 0.200 |
| 2025-05 | 68,450.00 | 307,420.00 | 60.95 | 2 | 0.800 |
| 2025-06 | 38,140.00 | 345,560.00 | -44.28 | 6 | 0.000 |

คิวรีนี้ใช้ `SUM() OVER()` (running total), `LAG()` (month-over-month), `RANK()` (อันดับ) และ `PERCENT_RANK()` (เปอร์เซ็นไทล์) รวม 4 ฟังก์ชันในคิวรีเดียว — จาก insight ที่ได้ ทีมผู้บริหารจะเห็นทันทีว่าเดือนมกราคมทำรายได้ดีที่สุด (`revenue_rank = 1`, `pct_rank = 1.000`) ในขณะที่เดือนมิถุนายนแย่ที่สุด และรายได้รวมสะสมทั้ง 6 เดือนอยู่ที่ 345,560 บาท

**ข้อสังเกตเพิ่มเติมสำหรับผู้เรียนระดับสูง**: หากต้องการให้ `mom_growth_pct` ของเดือนแรก (NULL) แสดงเป็น "N/A" แทนที่จะเป็น NULL ในรายงาน สามารถห่อด้วย `COALESCE(..., 'N/A')` ได้ แต่ต้อง cast ผลลัพธ์เป็น `TEXT` ก่อน เนื่องจากคอลัมน์เดิมเป็นชนิดตัวเลข

</details>

---

## บทถัดไป

เรียนรู้การจัดการและประมวลผลข้อความใน PostgreSQL อย่างมืออาชีพต่อได้ที่ [Part 031: String Functions](./part-031-string-functions.md)
