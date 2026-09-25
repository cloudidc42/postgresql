# ตอนที่ 028: GROUP BY, HAVING และการสรุปข้อมูลหลายมิติ (GROUPING SETS, ROLLUP, CUBE)

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับกลาง | Part 028

บทนี้เป็นส่วนหนึ่งของชุดบทเรียนที่ใช้ฐานข้อมูลอีคอมเมิร์ซชุดเดียวกันตั้งแต่ Part 021 ถึง Part 039 เพื่อให้ผู้เรียนเห็นภาพรวมของระบบจริงและฝึกเขียนคิวรีที่ซับซ้อนขึ้นเรื่อย ๆ บนโครงสร้างข้อมูลเดิม โดย Part นี้จะเน้นเรื่องการจัดกลุ่มข้อมูล (`GROUP BY`), การกรองข้อมูลหลังจัดกลุ่ม (`HAVING`) และเทคนิคการสรุปข้อมูลหลายมิติด้วย `GROUPING SETS`, `ROLLUP`, `CUBE` และฟังก์ชัน `GROUPING()`

## เป้าหมายการเรียนรู้

เมื่อจบบทนี้ ผู้เรียนจะสามารถ:

- ใช้ `GROUP BY` จัดกลุ่มแถวข้อมูลร่วมกับ aggregate function (`COUNT`, `SUM`, `AVG`, `MIN`, `MAX`) ได้อย่างถูกต้อง
- จัดกลุ่มข้อมูลด้วยหลายคอลัมน์พร้อมกัน เพื่อสร้างรายงานที่ละเอียดขึ้น
- แยกความแตกต่างระหว่าง `WHERE` (กรองก่อนจัดกลุ่ม) กับ `HAVING` (กรองหลังจัดกลุ่ม) และเลือกใช้ให้ถูกจุด
- เข้าใจกฎการเขียน `SELECT` ร่วมกับ `GROUP BY` และอ่าน error message ที่พบบ่อยออก
- เขียนคิวรี `GROUP BY` ร่วมกับ `JOIN` เพื่อสร้างรายงานเชิงธุรกิจ เช่น ยอดขายต่อลูกค้า ยอดขายต่อพนักงาน
- ใช้ `GROUPING SETS` เพื่อสร้างผลสรุปหลายระดับในคิวรีเดียว แทนการเขียน `UNION ALL` หลายรอบ
- ใช้ `ROLLUP` สร้างรายงานสรุปแบบลำดับชั้น (subtotal → grand total)
- ใช้ `CUBE` สร้างรายงานสรุปแบบไขว้ทุกชุดผสมของ dimension
- ใช้ฟังก์ชัน `GROUPING()` เพื่อแยกแยะแถวที่เป็นค่าจริง (`NULL` จากข้อมูล) ออกจากแถวที่เป็น subtotal/grand total (`NULL` จากการสรุป)
- สร้างรายงานยอดขายหลายมิติแบบมืออาชีพที่ใช้ในงาน analytics จริง

---

## เตรียมข้อมูล

รันสคริปต์ต่อไปนี้เพื่อสร้างตารางและข้อมูลตัวอย่างที่จะใช้ตลอด Part 021-039 หากเคยสร้างตารางชุดนี้ไว้แล้วจากบทก่อนหน้า สามารถข้ามส่วนนี้ไปได้ (โครงสร้างตารางเหมือนกันทุก Part)

```sql
-- ลบตารางเดิม (ถ้ามี) เพื่อให้สคริปต์รันซ้ำได้
DROP TABLE IF EXISTS payments CASCADE;
DROP TABLE IF EXISTS reviews CASCADE;
DROP TABLE IF EXISTS order_items CASCADE;
DROP TABLE IF EXISTS orders CASCADE;
DROP TABLE IF EXISTS employees CASCADE;
DROP TABLE IF EXISTS customers CASCADE;
DROP TABLE IF EXISTS products CASCADE;
DROP TABLE IF EXISTS suppliers CASCADE;
DROP TABLE IF EXISTS categories CASCADE;

-- โครงสร้างตาราง
CREATE TABLE categories (
    category_id SERIAL PRIMARY KEY,
    category_name VARCHAR(100) NOT NULL,
    parent_category_id INTEGER REFERENCES categories(category_id)
);

CREATE TABLE suppliers (
    supplier_id SERIAL PRIMARY KEY,
    supplier_name VARCHAR(150) NOT NULL,
    country VARCHAR(60)
);

CREATE TABLE products (
    product_id SERIAL PRIMARY KEY,
    product_name VARCHAR(150) NOT NULL,
    category_id INTEGER REFERENCES categories(category_id),
    supplier_id INTEGER REFERENCES suppliers(supplier_id),
    unit_price NUMERIC(10,2) NOT NULL CHECK (unit_price >= 0),
    stock_quantity INTEGER NOT NULL DEFAULT 0,
    is_active BOOLEAN NOT NULL DEFAULT true
);

CREATE TABLE customers (
    customer_id SERIAL PRIMARY KEY,
    first_name VARCHAR(60) NOT NULL,
    last_name VARCHAR(60) NOT NULL,
    email VARCHAR(150) UNIQUE,
    country VARCHAR(60),
    signup_date DATE NOT NULL DEFAULT CURRENT_DATE
);

CREATE TABLE employees (
    employee_id SERIAL PRIMARY KEY,
    first_name VARCHAR(60) NOT NULL,
    last_name VARCHAR(60) NOT NULL,
    hire_date DATE NOT NULL,
    manager_id INTEGER REFERENCES employees(employee_id),
    department VARCHAR(60)
);

CREATE TABLE orders (
    order_id SERIAL PRIMARY KEY,
    customer_id INTEGER REFERENCES customers(customer_id),
    employee_id INTEGER REFERENCES employees(employee_id),
    order_date TIMESTAMPTZ NOT NULL DEFAULT now(),
    status VARCHAR(20) NOT NULL DEFAULT 'pending',
    ship_country VARCHAR(60)
);

CREATE TABLE order_items (
    order_item_id SERIAL PRIMARY KEY,
    order_id INTEGER REFERENCES orders(order_id),
    product_id INTEGER REFERENCES products(product_id),
    quantity INTEGER NOT NULL CHECK (quantity > 0),
    unit_price NUMERIC(10,2) NOT NULL
);

CREATE TABLE reviews (
    review_id SERIAL PRIMARY KEY,
    product_id INTEGER REFERENCES products(product_id),
    customer_id INTEGER REFERENCES customers(customer_id),
    rating INTEGER NOT NULL CHECK (rating BETWEEN 1 AND 5),
    review_text TEXT,
    review_date DATE NOT NULL DEFAULT CURRENT_DATE
);

CREATE TABLE payments (
    payment_id SERIAL PRIMARY KEY,
    order_id INTEGER REFERENCES orders(order_id),
    payment_date TIMESTAMPTZ NOT NULL DEFAULT now(),
    amount NUMERIC(10,2) NOT NULL,
    payment_method VARCHAR(30)
);

-- ข้อมูลตัวอย่าง: categories (มีลำดับชั้น parent/child)
INSERT INTO categories (category_name, parent_category_id) VALUES
('Electronics', NULL),
('Computers & Laptops', 1),
('Mobile Phones', 1),
('Home Appliances', NULL),
('Kitchen Appliances', 4),
('Fashion', NULL),
('Men''s Clothing', 6),
('Women''s Clothing', 6),
('Books', NULL),
('Sports & Outdoors', NULL);

-- ข้อมูลตัวอย่าง: suppliers
INSERT INTO suppliers (supplier_name, country) VALUES
('TechSource Co., Ltd.', 'Thailand'),
('Global Gadgets Inc.', 'USA'),
('EuroHome Supplies', 'Germany'),
('Asia Appliance Group', 'China'),
('Fashion Forward Ltd.', 'Italy'),
('BookWorld Distribution', 'UK'),
('SportsGear International', 'USA'),
('Siam Electronics', 'Thailand');

-- ข้อมูลตัวอย่าง: products
INSERT INTO products (product_name, category_id, supplier_id, unit_price, stock_quantity, is_active) VALUES
('Laptop Pro 15"', 2, 1, 32900.00, 25, true),
('UltraBook Air 13"', 2, 2, 28500.00, 15, true),
('Gaming Laptop X', 2, 1, 45900.00, 10, true),
('Smartphone Galaxy S', 3, 2, 24900.00, 40, true),
('Smartphone Lite', 3, 8, 8900.00, 60, true),
('Wireless Earbuds Pro', 1, 2, 2990.00, 100, true),
('Smart Watch Series 5', 1, 8, 6500.00, 35, true),
('Blender Max 900W', 5, 3, 1590.00, 50, true),
('Air Fryer Deluxe', 5, 4, 2490.00, 45, true),
('Microwave Oven 25L', 5, 4, 3990.00, 20, true),
('Robot Vacuum Cleaner', 4, 3, 8990.00, 18, true),
('Men''s Denim Jacket', 7, 5, 1290.00, 70, true),
('Men''s Running Shoes', 7, 7, 2190.00, 55, true),
('Women''s Summer Dress', 8, 5, 990.00, 80, true),
('Women''s Handbag', 8, 5, 1890.00, 40, true),
('PostgreSQL Mastery Book', 9, 6, 890.00, 30, true),
('Yoga Mat Premium', 10, 7, 690.00, 60, true),
('Camping Tent 4-Person', 10, 7, 4990.00, 12, false);

-- ข้อมูลตัวอย่าง: customers
INSERT INTO customers (first_name, last_name, email, country, signup_date) VALUES
('Somchai', 'Jaidee', 'somchai.j@example.com', 'Thailand', '2022-03-15'),
('Suda', 'Meesuk', 'suda.m@example.com', 'Thailand', '2022-05-20'),
('John', 'Smith', 'john.smith@example.com', 'USA', '2021-11-02'),
('Emily', 'Johnson', 'emily.j@example.com', 'USA', '2023-01-10'),
('Hiroshi', 'Tanaka', 'hiroshi.t@example.com', 'Japan', '2022-07-08'),
('Yuki', 'Sato', 'yuki.sato@example.com', 'Japan', '2023-02-14'),
('Wei', 'Chen', 'wei.chen@example.com', 'Singapore', '2021-09-25'),
('Mei', 'Lin', 'mei.lin@example.com', 'Singapore', '2023-04-30'),
('Oliver', 'Brown', 'oliver.b@example.com', 'UK', '2022-01-18'),
('Charlotte', 'Davies', 'charlotte.d@example.com', 'UK', '2023-06-05'),
('Hans', 'Mueller', 'hans.m@example.com', 'Germany', '2021-12-12'),
('Anna', 'Schmidt', 'anna.schmidt@example.com', 'Germany', '2022-10-22'),
('James', 'Wilson', 'james.w@example.com', 'Australia', '2022-08-01'),
('Olivia', 'Taylor', 'olivia.t@example.com', 'Australia', '2023-03-17'),
('Nattapong', 'Srisuk', 'nattapong.s@example.com', 'Thailand', '2023-05-09');

-- ข้อมูลตัวอย่าง: employees (มีลำดับชั้นผู้จัดการ)
INSERT INTO employees (first_name, last_name, hire_date, manager_id, department) VALUES
('Piya', 'Wongsakul', '2019-03-01', NULL, 'Management'),
('Kanya', 'Rattana', '2020-01-15', 1, 'Sales'),
('Anurak', 'Boonmee', '2020-06-01', 1, 'Sales'),
('Siriporn', 'Chaiyo', '2021-02-10', 1, 'Support'),
('Thanawat', 'Kittisak', '2021-08-20', 4, 'Support'),
('Napassorn', 'Intra', '2022-01-05', 1, 'Marketing'),
('Korn', 'Suwannakit', '2022-05-15', 2, 'Sales'),
('Pimchanok', 'Wattana', '2023-03-01', 4, 'Support');

-- ข้อมูลตัวอย่าง: orders
INSERT INTO orders (customer_id, employee_id, order_date, status, ship_country) VALUES
(1, 2, '2024-01-05 10:15:00+07', 'completed', 'Thailand'),
(3, 3, '2024-01-12 14:30:00+07', 'completed', 'USA'),
(5, 2, '2024-01-20 09:00:00+07', 'completed', 'Japan'),
(7, 7, '2024-02-03 11:45:00+07', 'completed', 'Singapore'),
(9, 3, '2024-02-14 16:20:00+07', 'cancelled', 'UK'),
(11, 2, '2024-02-22 13:10:00+07', 'completed', 'Germany'),
(2, 7, '2024-03-01 08:50:00+07', 'completed', 'Thailand'),
(13, 3, '2024-03-10 10:05:00+07', 'completed', 'Australia'),
(4, 2, '2024-03-18 15:40:00+07', 'shipped', 'USA'),
(6, 7, '2024-03-25 12:00:00+07', 'completed', 'Japan'),
(8, 3, '2024-04-02 09:30:00+07', 'completed', 'Singapore'),
(10, 2, '2024-04-11 14:15:00+07', 'pending', 'UK'),
(12, 7, '2024-04-19 11:20:00+07', 'completed', 'Germany'),
(14, 3, '2024-04-28 16:05:00+07', 'completed', 'Australia'),
(1, 2, '2024-05-05 10:40:00+07', 'completed', 'Thailand'),
(15, 7, '2024-05-14 13:55:00+07', 'completed', 'Thailand'),
(3, 3, '2024-05-22 09:10:00+07', 'cancelled', 'USA'),
(5, 2, '2024-06-01 15:25:00+07', 'completed', 'Japan'),
(9, 7, '2024-06-12 12:30:00+07', 'completed', 'UK'),
(11, 3, '2024-06-20 10:50:00+07', 'completed', 'Germany');

-- ข้อมูลตัวอย่าง: order_items
INSERT INTO order_items (order_id, product_id, quantity, unit_price) VALUES
(1, 1, 1, 32900.00),
(1, 6, 2, 2990.00),
(2, 4, 1, 24900.00),
(3, 2, 1, 28500.00),
(3, 7, 1, 6500.00),
(4, 5, 2, 8900.00),
(5, 9, 1, 2490.00),
(6, 11, 1, 8990.00),
(6, 8, 1, 1590.00),
(7, 14, 2, 990.00),
(7, 15, 1, 1890.00),
(8, 13, 1, 2190.00),
(8, 17, 2, 690.00),
(9, 1, 1, 32900.00),
(9, 6, 1, 2990.00),
(10, 4, 1, 24900.00),
(10, 7, 1, 6500.00),
(11, 5, 3, 8900.00),
(12, 10, 1, 3990.00),
(13, 2, 1, 28500.00),
(13, 6, 2, 2990.00),
(14, 12, 2, 1290.00),
(14, 16, 3, 890.00),
(15, 3, 1, 45900.00),
(15, 7, 1, 6500.00),
(16, 9, 2, 2490.00),
(16, 18, 1, 4990.00),
(17, 1, 1, 32900.00),
(18, 4, 1, 24900.00),
(18, 5, 1, 8900.00),
(19, 11, 1, 8990.00),
(19, 8, 2, 1590.00),
(20, 2, 1, 28500.00),
(20, 6, 3, 2990.00),
(20, 17, 1, 690.00);

-- ข้อมูลตัวอย่าง: reviews
INSERT INTO reviews (product_id, customer_id, rating, review_text, review_date) VALUES
(1, 1, 5, 'แล็ปท็อปแรงมาก ทำงานลื่นไหลดีมาก', '2024-01-10'),
(4, 3, 4, 'สมาร์ทโฟนกล้องสวย ใช้งานคุ้มราคา', '2024-01-18'),
(2, 5, 5, 'บางเบา พกพาง่าย แบตอึด', '2024-01-25'),
(5, 7, 4, 'ราคาดี สเปคเหมาะกับงานทั่วไป', '2024-02-08'),
(11, 11, 5, 'หุ่นยนต์ดูดฝุ่นทำงานเงียบและมีประสิทธิภาพ', '2024-02-25'),
(14, 2, 3, 'ผ้าดีแต่ไซส์เล็กกว่าที่คาดไว้', '2024-03-05'),
(15, 2, 4, 'กระเป๋าสวยงาม วัสดุดี', '2024-03-05'),
(13, 13, 5, 'รองเท้าใส่สบาย วิ่งได้ไกลขึ้น', '2024-03-12'),
(1, 4, 4, 'ประสิทธิภาพดี แต่ราคาค่อนข้างสูง', '2024-03-20'),
(4, 6, 5, 'ชอบมาก กล้องถ่ายรูปสวย', '2024-03-28'),
(5, 8, 3, 'ใช้งานได้ปกติ ไม่มีอะไรโดดเด่น', '2024-04-05'),
(2, 12, 5, 'คุ้มค่ามากสำหรับงานเอกสาร', '2024-04-22'),
(16, 14, 5, 'หนังสือสอน PostgreSQL เข้าใจง่ายมาก', '2024-04-30'),
(3, 1, 5, 'เล่นเกมลื่นไหล การ์ดจอแรงมาก', '2024-05-08'),
(9, 15, 4, 'ทอดอาหารกรอบอร่อย ทำความสะอาดง่าย', '2024-05-16');

-- ข้อมูลตัวอย่าง: payments
INSERT INTO payments (order_id, payment_date, amount, payment_method) VALUES
(1, '2024-01-05 10:20:00+07', 38880.00, 'credit_card'),
(2, '2024-01-12 14:35:00+07', 24900.00, 'promptpay'),
(3, '2024-01-20 09:05:00+07', 35000.00, 'credit_card'),
(4, '2024-02-03 11:50:00+07', 17800.00, 'bank_transfer'),
(6, '2024-02-22 13:15:00+07', 10580.00, 'paypal'),
(7, '2024-03-01 08:55:00+07', 3870.00, 'promptpay'),
(8, '2024-03-10 10:10:00+07', 3570.00, 'credit_card'),
(9, '2024-03-18 15:45:00+07', 35890.00, 'credit_card'),
(10, '2024-03-25 12:05:00+07', 31400.00, 'bank_transfer'),
(11, '2024-04-02 09:35:00+07', 26700.00, 'paypal'),
(13, '2024-04-19 11:25:00+07', 34480.00, 'credit_card'),
(14, '2024-04-28 16:10:00+07', 5250.00, 'promptpay'),
(15, '2024-05-05 10:45:00+07', 52400.00, 'credit_card'),
(16, '2024-05-14 14:00:00+07', 9970.00, 'bank_transfer'),
(18, '2024-06-01 15:30:00+07', 33800.00, 'credit_card'),
(19, '2024-06-12 12:35:00+07', 12170.00, 'paypal'),
(20, '2024-06-20 10:55:00+07', 38160.00, 'promptpay');
```

ตรวจสอบว่าข้อมูลถูกโหลดครบด้วยคิวรีสั้น ๆ:

```sql
SELECT
    (SELECT COUNT(*) FROM categories)  AS categories,
    (SELECT COUNT(*) FROM suppliers)   AS suppliers,
    (SELECT COUNT(*) FROM products)    AS products,
    (SELECT COUNT(*) FROM customers)   AS customers,
    (SELECT COUNT(*) FROM employees)   AS employees,
    (SELECT COUNT(*) FROM orders)      AS orders,
    (SELECT COUNT(*) FROM order_items) AS order_items,
    (SELECT COUNT(*) FROM reviews)     AS reviews,
    (SELECT COUNT(*) FROM payments)    AS payments;
```

```
 categories | suppliers | products | customers | employees | orders | order_items | reviews | payments
------------+-----------+----------+-----------+-----------+--------+-------------+---------+----------
         10 |         8 |       18 |        15 |         8 |     20 |          35 |      15 |       17
```

---

## Step 271: GROUP BY พื้นฐาน

`GROUP BY` ใช้จัดกลุ่มแถวข้อมูลที่มีค่าคอลัมน์ที่ระบุเหมือนกันเข้าด้วยกัน แล้วให้ aggregate function เช่น `COUNT()`, `SUM()`, `AVG()`, `MIN()`, `MAX()` คำนวณค่าสรุปของแต่ละกลุ่ม แทนที่จะคำนวณจากทั้งตาราง

ตัวอย่าง: นับจำนวนสินค้าและราคาเฉลี่ยในแต่ละหมวดหมู่

```sql
SELECT
    c.category_name,
    COUNT(*)                    AS product_count,
    ROUND(AVG(p.unit_price), 2) AS avg_price,
    SUM(p.stock_quantity)       AS total_stock
FROM products p
JOIN categories c ON c.category_id = p.category_id
GROUP BY c.category_name
ORDER BY product_count DESC, c.category_name;
```

```
    category_name    | product_count | avg_price | total_stock
----------------------+---------------+-----------+-------------
 Computers & Laptops  |             3 |  35766.67 |          50
 Kitchen Appliances   |             3 |   2690.00 |         115
 Electronics          |             2 |   4745.00 |         135
 Mobile Phones        |             2 |  16900.00 |         100
 Men's Clothing       |             2 |   1740.00 |         125
 Women's Clothing     |             2 |   1440.00 |         120
 Sports & Outdoors    |             2 |   2840.00 |          72
 Home Appliances      |             1 |   8990.00 |          18
 Books                |             1 |    890.00 |          30
```

จุดสำคัญ:

- `GROUP BY c.category_name` บอก PostgreSQL ให้รวมแถวที่มี `category_name` เดียวกันไว้ด้วยกัน
- ทุกแถวใน 1 กลุ่ม จะถูกส่งเข้า aggregate function ทีละกลุ่ม ผลลัพธ์จึงมี 1 แถวต่อ 1 กลุ่ม
- หมวดหมู่ `Fashion` ไม่ปรากฏในผลลัพธ์ เพราะไม่มีสินค้าใดผูกกับ `category_id` ของ `Fashion` โดยตรง (มีแต่หมวดย่อย `Men's Clothing` และ `Women's Clothing`) — `GROUP BY` จะสร้างกลุ่มเฉพาะค่าที่ "มีอยู่จริง" ในข้อมูลเท่านั้น ไม่ได้สร้างกลุ่มว่างขึ้นมาเอง
- ถ้าไม่มี `GROUP BY` เลย และมี aggregate function ในคิวรี ทั้งตารางจะถูกมองเป็น 1 กลุ่มใหญ่ก้อนเดียว

ตัวอย่างแบบไม่มี `GROUP BY` (สรุปทั้งตารางเป็นก้อนเดียว):

```sql
SELECT COUNT(*) AS total_products, ROUND(AVG(unit_price), 2) AS avg_price
FROM products;
```

```
 total_products | avg_price
-----------------+-----------
              18 |  11986.67
```

---

## Step 272: GROUP BY หลายคอลัมน์

สามารถระบุหลายคอลัมน์ใน `GROUP BY` เพื่อจัดกลุ่มแบบละเอียดขึ้น — PostgreSQL จะถือว่าแถวสองแถวอยู่กลุ่มเดียวกันก็ต่อเมื่อ**ค่าทุกคอลัมน์**ที่ระบุใน `GROUP BY` เหมือนกันทั้งหมด

ตัวอย่าง: ยอดขาย (quantity × unit_price) แยกตามหมวดหมู่สินค้า และประเทศปลายทางการจัดส่ง

```sql
SELECT
    c.category_name,
    o.ship_country,
    SUM(oi.quantity * oi.unit_price) AS total_sales,
    COUNT(*)                          AS line_items
FROM order_items oi
JOIN orders o     ON o.order_id = oi.order_id
JOIN products p   ON p.product_id = oi.product_id
JOIN categories c ON c.category_id = p.category_id
GROUP BY c.category_name, o.ship_country
ORDER BY c.category_name, total_sales DESC;
```

```
   category_name    | ship_country | total_sales | line_items
----------------------+--------------+-------------+------------
 Computers & Laptops  | Thailand     |   111700.00 |          3
 Computers & Laptops  | USA          |    32900.00 |          1
 Computers & Laptops  | Germany      |    34480.00 |          1
 Computers & Laptops  | UK           |    28500.00 |          1
 Electronics          | Thailand     |    15470.00 |          3
 Electronics          | Japan        |    13000.00 |          2
 Electronics          | Germany      |     5980.00 |          1
 Electronics          | UK           |     6500.00 |          1
 ...
```

*(ตัดแสดงบางส่วนเพื่อความกระชับ — รันคิวรีจริงเพื่อดูผลลัพธ์ครบทุกแถว)*

ข้อสังเกต:

- จำนวนแถวในผลลัพธ์จะเพิ่มขึ้นตามจำนวน "ชุดผสม" ของค่าที่พบจริงในข้อมูล ยิ่งจัดกลุ่มหลายคอลัมน์ ยิ่งได้รายงานละเอียดขึ้นแต่แถวเยอะขึ้น
- ลำดับคอลัมน์ใน `GROUP BY` ไม่มีผลต่อผลลัพธ์ (ต่างจาก `ORDER BY` ที่ลำดับมีผล) — `GROUP BY a, b` ให้ผลลัพธ์ชุดข้อมูลเดียวกับ `GROUP BY b, a` เพียงแต่จัดเรียงคอลัมน์ในผลลัพธ์ต่างกันตามที่ระบุใน `SELECT`
- สามารถใช้ตำแหน่งคอลัมน์แทนชื่อได้ เช่น `GROUP BY 1, 2` (อ้างอิงคอลัมน์ที่ 1 และ 2 ใน `SELECT`) ซึ่งสะดวกเวลาคอลัมน์มี expression ซับซ้อน แต่ควรใช้ชื่อคอลัมน์เต็มในโค้ด production เพื่อความชัดเจนและป้องกันบั๊กเวลาแก้ `SELECT`

```sql
-- เทียบเท่ากับคิวรีด้านบน โดยใช้ตำแหน่งคอลัมน์
SELECT c.category_name, o.ship_country, SUM(oi.quantity * oi.unit_price) AS total_sales
FROM order_items oi
JOIN orders o     ON o.order_id = oi.order_id
JOIN products p   ON p.product_id = oi.product_id
JOIN categories c ON c.category_id = p.category_id
GROUP BY 1, 2
ORDER BY 1, 3 DESC;
```

---

## Step 273: HAVING — กรองผลลัพธ์หลังจัดกลุ่ม

`WHERE` กรอง**แถวดิบ**ก่อนที่จะถูกจัดกลุ่ม ส่วน `HAVING` กรอง**กลุ่ม**หลังจากคำนวณ aggregate function เสร็จแล้ว ทั้งสองมีหน้าที่ต่างกันโดยสิ้นเชิง และใช้แทนกันไม่ได้

ลำดับการประมวลผลของ PostgreSQL (แนวคิด ไม่ใช่ลำดับการเขียนในคิวรี):

```
FROM/JOIN  →  WHERE  →  GROUP BY  →  HAVING  →  SELECT  →  ORDER BY  →  LIMIT
```

- `WHERE` ทำงานก่อน `GROUP BY` จึงใช้ aggregate function ใน `WHERE` ไม่ได้ (เพราะตอนนั้นยังไม่มีกลุ่มให้คำนวณ)
- `HAVING` ทำงานหลัง `GROUP BY` จึงใช้ aggregate function ได้ตามปกติ และยังสามารถอ้างอิงคอลัมน์ที่ใช้ `GROUP BY` ได้ด้วย

ตัวอย่าง: หาประเทศที่มีลูกค้าสมัครสมาชิกตั้งแต่ 3 คนขึ้นไป

```sql
SELECT country, COUNT(*) AS customer_count
FROM customers
GROUP BY country
HAVING COUNT(*) >= 3
ORDER BY customer_count DESC;
```

```
 country  | customer_count
----------+-----------------
 Thailand |               3
```

เทียบกับการใช้ `WHERE` ผิดที่ (จะเกิด error ทันที เพราะยังไม่มีการจัดกลุ่มตอนที่ `WHERE` ทำงาน):

```sql
SELECT country, COUNT(*) AS customer_count
FROM customers
WHERE COUNT(*) >= 3   -- ผิด!
GROUP BY country;
```

```
ERROR:  aggregate functions are not allowed in WHERE
LINE 3: WHERE COUNT(*) >= 3
              ^
```

ตัวอย่างที่ใช้ทั้ง `WHERE` และ `HAVING` ร่วมกัน — เป็นรูปแบบที่พบบ่อยที่สุดในงานจริง: `WHERE` กรองแถวดิบก่อน (เช่น เฉพาะออเดอร์ที่สำเร็จ) แล้ว `HAVING` กรองผลสรุปอีกชั้น (เช่น เฉพาะลูกค้าที่ซื้อรวมเกินเกณฑ์)

```sql
SELECT
    cu.customer_id,
    cu.first_name || ' ' || cu.last_name AS customer_name,
    SUM(oi.quantity * oi.unit_price)     AS total_spent
FROM customers cu
JOIN orders o      ON o.customer_id = cu.customer_id
JOIN order_items oi ON oi.order_id = o.order_id
WHERE o.status = 'completed'              -- ขั้นที่ 1: กรองเฉพาะออเดอร์สำเร็จ ก่อนจัดกลุ่ม
GROUP BY cu.customer_id, customer_name
HAVING SUM(oi.quantity * oi.unit_price) > 30000  -- ขั้นที่ 2: กรองกลุ่มที่ยอดรวมเกิน 30,000
ORDER BY total_spent DESC;
```

```
 customer_id | customer_name  | total_spent
--------------+----------------+-------------
            1 | Somchai Jaidee |    91280.00
            5 | Hiroshi Tanaka |    35000.00
```

สรุปความแตกต่าง:

| ประเด็น | WHERE | HAVING |
|---|---|---|
| ทำงานตอนไหน | ก่อนจัดกลุ่ม (กรองแถวดิบ) | หลังจัดกลุ่ม (กรองกลุ่ม) |
| ใช้ aggregate function ได้ไหม | ไม่ได้ | ได้ |
| ใช้กับคอลัมน์ดิบของตารางได้ไหม | ได้ | ได้ (ถ้าอยู่ใน GROUP BY) |
| ผลต่อ performance | กรองแถวออกเร็ว ลดงานที่ต้องจัดกลุ่ม | กรองหลังคำนวณเสร็จแล้ว |

**เคล็ดลับด้านประสิทธิภาพ**: ควรใส่เงื่อนไขที่กรองแถวดิบได้ (เช่น ช่วงวันที่ สถานะ) ไว้ใน `WHERE` เสมอ แทนที่จะปล่อยให้ไปกรองใน `HAVING` ทีหลัง เพราะ `WHERE` ช่วยลดจำนวนแถวที่ต้องนำไปจัดกลุ่มและคำนวณ aggregate ตั้งแต่ต้น ทำให้คิวรีเร็วขึ้นอย่างมีนัยสำคัญเมื่อข้อมูลมีปริมาณมาก

---

## Step 274: กฎการเขียน SELECT ร่วมกับ GROUP BY

กฎเหล็กของ SQL มาตรฐาน (และ PostgreSQL): **ทุกคอลัมน์ใน `SELECT` ที่ไม่ได้อยู่ใน aggregate function จะต้องปรากฏใน `GROUP BY` ด้วย** เหตุผลคือเมื่อจัดกลุ่มแล้ว แต่ละกลุ่มอาจมีหลายแถว หาก `SELECT` คอลัมน์ที่ไม่ได้จัดกลุ่มและไม่ใช่ aggregate ออกมา PostgreSQL จะไม่รู้ว่าควรเลือกค่าจากแถวไหนในกลุ่มมาแสดง จึงถือเป็น error

ตัวอย่างที่ผิด:

```sql
SELECT product_name, category_id, SUM(unit_price) AS total_price
FROM products
GROUP BY category_id;
```

```
ERROR:  column "products.product_name" must appear in the GROUP BY clause
        or be used in an aggregate function
LINE 1: SELECT product_name, category_id, SUM(unit_price) AS total...
               ^
```

วิธีแก้มี 2 ทาง:

**ทางที่ 1** — เพิ่ม `product_name` เข้าไปใน `GROUP BY` (แต่จะทำให้จัดกลุ่มละเอียดขึ้น กลายเป็นกลุ่มต่อสินค้าแทนที่จะเป็นกลุ่มต่อหมวดหมู่ ซึ่งอาจไม่ใช่สิ่งที่ต้องการ)

```sql
SELECT product_name, category_id, SUM(unit_price) AS total_price
FROM products
GROUP BY product_name, category_id;
```

**ทางที่ 2** — ครอบ `product_name` ด้วย aggregate function เช่น `array_agg()` เพื่อรวมชื่อสินค้าทั้งหมดในกลุ่มไว้เป็น array เดียว

```sql
SELECT
    category_id,
    array_agg(product_name ORDER BY product_name) AS product_names,
    SUM(unit_price) AS total_price
FROM products
GROUP BY category_id
ORDER BY category_id;
```

```
 category_id |                    product_names                     | total_price
--------------+-------------------------------------------------------+-------------
            1 | {"Smart Watch Series 5","Wireless Earbuds Pro"}      |     9490.00
            2 | {"Gaming Laptop X","Laptop Pro 15\"","UltraBook Air 13\""} |   107300.00
            3 | {"Smartphone Galaxy S","Smartphone Lite"}            |    33800.00
```

**ข้อยกเว้นที่น่าสนใจของ PostgreSQL**: ถ้า `GROUP BY` เป็น primary key ของตาราง PostgreSQL อนุญาตให้ `SELECT` คอลัมน์อื่น ๆ ของตารางเดียวกันได้โดยไม่ต้องอยู่ใน `GROUP BY` เพราะ PostgreSQL รู้ว่า primary key กำหนดค่าคอลัมน์อื่นทั้งหมดในแถวนั้นโดยอัตโนมัติ (functional dependency) — เป็นพฤติกรรมที่เกินกว่า SQL มาตรฐานกำหนด และหลายฐานข้อมูลอื่นไม่รองรับ

```sql
-- ใช้ได้ใน PostgreSQL เพราะ product_id เป็น PRIMARY KEY
-- (PostgreSQL รู้ว่า 1 product_id มีได้แค่ 1 product_name, 1 unit_price เสมอ)
SELECT product_id, product_name, unit_price
FROM products
GROUP BY product_id;
```

ข้อผิดพลาดที่พบบ่อยอีกแบบ: ลืมว่า `WHERE` ทำงานก่อน `GROUP BY` จึงพยายามใช้ชื่อ alias ของ aggregate ใน `WHERE`

```sql
SELECT category_id, SUM(unit_price) AS total_price
FROM products
WHERE total_price > 10000   -- ผิด! total_price ยังไม่ถูกคำนวณตอนที่ WHERE ทำงาน
GROUP BY category_id;
```

```
ERROR:  column "total_price" does not exist
LINE 3: WHERE total_price > 10000
              ^
```

ต้องใช้ `HAVING SUM(unit_price) > 10000` แทน (หรือ `HAVING total_price > 10000` ก็ได้ เพราะ `HAVING` ทำงานหลัง `SELECT` คำนวณ alias เสร็จแล้วใน PostgreSQL — ต่างจาก `WHERE` ที่ยังไม่รู้จัก alias)

---

## Step 275: GROUP BY ร่วมกับ JOIN ในสถานการณ์ analytics จริง

งาน analytics ส่วนใหญ่ต้อง `JOIN` หลายตารางก่อนแล้วค่อย `GROUP BY` เพื่อสร้างรายงานสรุป ตัวอย่างที่พบบ่อยที่สุดคือ "ยอดขายต่อลูกค้า" และ "ยอดขายต่อพนักงาน"

### ยอดขายต่อลูกค้า (ไม่รวมออเดอร์ที่ถูกยกเลิก)

```sql
SELECT
    cu.customer_id,
    cu.first_name || ' ' || cu.last_name AS customer_name,
    cu.country,
    COUNT(DISTINCT o.order_id)           AS order_count,
    SUM(oi.quantity * oi.unit_price)     AS total_spent
FROM customers cu
JOIN orders o       ON o.customer_id = cu.customer_id
JOIN order_items oi ON oi.order_id = o.order_id
WHERE o.status <> 'cancelled'
GROUP BY cu.customer_id, customer_name, cu.country
ORDER BY total_spent DESC
LIMIT 5;
```

```
 customer_id | customer_name  |  country  | order_count | total_spent
--------------+----------------+-----------+--------------+-------------
            1 | Somchai Jaidee | Thailand  |            2 |    91280.00
            5 | Hiroshi Tanaka | Japan     |            2 |    68800.00
           11 | Hans Mueller   | Germany   |            2 |    48740.00
            9 | Oliver Brown   | UK        |            1 |    12170.00
            2 | Suda Meesuk    | Thailand  |            1 |     3870.00
```

`COUNT(DISTINCT o.order_id)` สำคัญมากในกรณีนี้ เพราะการ `JOIN` กับ `order_items` ทำให้แต่ละออเดอร์ปรากฏซ้ำหลายแถว (1 แถวต่อ 1 รายการสินค้า) ถ้าใช้ `COUNT(o.order_id)` เฉย ๆ จะนับจำนวน **รายการสินค้า** ไม่ใช่จำนวน **ออเดอร์**

### ยอดขายต่อพนักงาน (เฉพาะออเดอร์ที่ completed)

```sql
SELECT
    e.employee_id,
    e.first_name || ' ' || e.last_name AS employee_name,
    e.department,
    COUNT(DISTINCT o.order_id)         AS orders_handled,
    SUM(oi.quantity * oi.unit_price)   AS total_sales
FROM employees e
JOIN orders o       ON o.employee_id = e.employee_id
JOIN order_items oi ON oi.order_id = o.order_id
WHERE o.status = 'completed'
GROUP BY e.employee_id, employee_name, e.department
HAVING SUM(oi.quantity * oi.unit_price) > 30000
ORDER BY total_sales DESC;
```

```
 employee_id | employee_name  | department | orders_handled | total_sales
--------------+----------------+------------+-----------------+-------------
            2 | Kanya Rattana  | Sales      |               5 |   170660.00
            7 | Korn Suwannakit| Sales      |               6 |   109690.00
            3 | Anurak Boonmee | Sales      |               5 |    98580.00
```

จุดสำคัญของการใช้ `GROUP BY` ร่วมกับ `JOIN`:

- ควรกรองเงื่อนไขที่ทำได้ด้วย `WHERE` ก่อนเสมอ (เช่น `status = 'completed'`) เพื่อไม่ให้ข้อมูลที่ไม่เกี่ยวข้องถูกดึงเข้ามาคำนวณ
- เมื่อ `JOIN` ตารางที่มีความสัมพันธ์แบบ 1-ต่อ-หลาย (เช่น 1 order มีหลาย order_items) ต้องระวังการนับซ้ำ (double counting) เสมอ ใช้ `COUNT(DISTINCT ...)` หรือคำนวณยอดรวมจากตารางลูกให้ถูกระดับ
- ถ้าต้องการรวมลูกค้า/พนักงานที่ "ไม่มี" ออเดอร์เลยในรายงานด้วย (เช่น แสดงยอดขาย 0 แทนที่จะหายไปจากรายงาน) ต้องใช้ `LEFT JOIN` แทน `JOIN` และใช้ `COALESCE(SUM(...), 0)` เพื่อแปลง `NULL` เป็น `0`

```sql
-- ตัวอย่าง LEFT JOIN: แสดงพนักงานทุกคน แม้ไม่มียอดขายก็ยังปรากฏเป็น 0
SELECT
    e.employee_id,
    e.first_name || ' ' || e.last_name AS employee_name,
    COALESCE(SUM(oi.quantity * oi.unit_price), 0) AS total_sales
FROM employees e
LEFT JOIN orders o       ON o.employee_id = e.employee_id AND o.status = 'completed'
LEFT JOIN order_items oi ON oi.order_id = o.order_id
GROUP BY e.employee_id, employee_name
ORDER BY total_sales DESC;
```

สังเกตว่าเงื่อนไข `o.status = 'completed'` ถูกย้ายไปไว้ใน `ON` แทน `WHERE` — เพราะถ้าใส่ใน `WHERE` มันจะไปกรองแถว `NULL` ที่เกิดจาก `LEFT JOIN` ออกไปด้วย ทำให้พนักงานที่ไม่มีออเดอร์ completed หายไปจากผลลัพธ์ทั้งที่ต้องการให้แสดงเป็น 0

---

## Step 276: GROUPING SETS — สร้างหลายระดับการสรุปในคิวรีเดียว

ในงานจริง เรามักต้องการรายงานสรุปหลายมุมมองพร้อมกัน เช่น "ยอดขายรวมต่อหมวดหมู่" และ "ยอดขายรวมต่อประเทศ" และ "ยอดขายรวมทั้งหมด" วิธีเดิมคือเขียน 3 คิวรีแยกแล้ว `UNION ALL` เข้าด้วยกัน แต่ `GROUPING SETS` ทำสิ่งนี้ได้ในคิวรีเดียว โดย PostgreSQL จะสแกนข้อมูลครั้งเดียวแล้วคำนวณทุกระดับการจัดกลุ่มที่ระบุพร้อมกัน — เร็วกว่าและกระชับกว่า

รูปแบบ:

```sql
SELECT col_a, col_b, agg_function(...)
FROM ...
GROUP BY GROUPING SETS ( (col_a), (col_b), () )
```

แต่ละวงเล็บคือ "ชุด" ของคอลัมน์ที่จะใช้จัดกลุ่มในระดับนั้น ๆ และ `()` (วงเล็บว่าง) หมายถึง "ไม่จัดกลุ่มเลย" คือยอดรวมทั้งหมด (grand total)

ตัวอย่าง: สรุปยอดขายตามหมวดหมู่, ตามประเทศ, และยอดรวมทั้งหมด ในคิวรีเดียว

```sql
SELECT
    c.category_name,
    o.ship_country,
    SUM(oi.quantity * oi.unit_price) AS total_sales
FROM order_items oi
JOIN orders o     ON o.order_id = oi.order_id
JOIN products p   ON p.product_id = oi.product_id
JOIN categories c ON c.category_id = p.category_id
GROUP BY GROUPING SETS (
    (c.category_name),
    (o.ship_country),
    ()
)
ORDER BY c.category_name NULLS LAST, o.ship_country NULLS LAST;
```

```
    category_name    | ship_country | total_sales
----------------------+--------------+-------------
 Books                |              |     2670.00
 Computers & Laptops  |              |   230100.00
 Electronics          |              |    43420.00
 Home Appliances      |              |    17980.00
 Kitchen Appliances   |              |    16230.00
 Men's Clothing       |              |     4770.00
 Mobile Phones        |              |   128100.00
 Sports & Outdoors    |              |     7060.00
 Women's Clothing     |              |     3870.00
                      | Australia    |     8820.00
                      | Germany      |    83220.00
                      | Japan        |   100200.00
                      | Singapore    |    44500.00
                      | Thailand     |   105120.00
                      | UK           |    18650.00
                      | USA          |    93690.00
                      |              |   454200.00
```

สังเกต:

- แถวที่จัดกลุ่มตามหมวดหมู่ จะมีค่า `ship_country` เป็น `NULL` (เพราะระดับนั้นไม่ได้จัดกลุ่มตามประเทศ)
- แถวที่จัดกลุ่มตามประเทศ จะมีค่า `category_name` เป็น `NULL`
- แถวสุดท้าย (ทั้งสองคอลัมน์เป็น `NULL`) คือยอดรวมทั้งหมด 454,200.00 บาท จากชุด `()`
- `NULLS LAST` ใน `ORDER BY` ช่วยจัดให้แถวสรุป (ที่มี `NULL`) ไปอยู่ท้ายกลุ่มของแต่ละคอลัมน์ อ่านง่ายขึ้น

`GROUPING SETS` ยังรองรับการจัดกลุ่มแบบผสมหลายคอลัมน์ในชุดเดียวได้ เช่น

```sql
GROUP BY GROUPING SETS (
    (c.category_name, o.ship_country),  -- จัดกลุ่มตามทั้งสองคอลัมน์พร้อมกัน
    (c.category_name),                   -- สรุปตามหมวดหมู่อย่างเดียว
    ()                                    -- ยอดรวมทั้งหมด
)
```

**ข้อดีของ `GROUPING SETS` เทียบกับ `UNION ALL`**: สแกนตารางต้นทางเพียงครั้งเดียว (PostgreSQL วางแผนให้ใช้ sort หรือ hash เดียวกันคำนวณหลายระดับพร้อมกัน) ในขณะที่ `UNION ALL` ของหลายคิวรีจะสแกนตารางซ้ำหลายรอบ — ยิ่งข้อมูลใหญ่ ความต่างด้าน performance ยิ่งชัดเจน

---

## Step 277: ROLLUP — สรุปแบบลำดับชั้น

`ROLLUP` เป็นรูปแบบย่อของ `GROUPING SETS` ที่ใช้สำหรับข้อมูลที่มี**ลำดับชั้น** (hierarchy) เช่น หมวดหมู่ใหญ่ → หมวดหมู่ย่อย, หรือ ปี → เดือน → วัน โดย `ROLLUP(a, b, c)` จะสร้างชุดการจัดกลุ่มแบบ "ลดหลั่น" จากซ้ายไปขวาให้อัตโนมัติ เทียบเท่ากับ:

```sql
GROUPING SETS ( (a, b, c), (a, b), (a), () )
```

สังเกตว่า `ROLLUP` **ไม่สมมาตร** — ลำดับคอลัมน์มีความหมาย (ต่างจาก `GROUPING SETS` ที่ระบุชุดเองได้อิสระ)

ตัวอย่าง: สรุปยอดขายตามลำดับชั้นหมวดหมู่ใหญ่ → หมวดหมู่ย่อย โดยใช้ความสัมพันธ์ `parent_category_id` ในตาราง `categories`

```sql
SELECT
    COALESCE(parent.category_name, c.category_name) AS major_category,
    CASE WHEN c.parent_category_id IS NOT NULL THEN c.category_name END AS sub_category,
    SUM(oi.quantity * oi.unit_price) AS total_sales
FROM order_items oi
JOIN products p    ON p.product_id = oi.product_id
JOIN categories c  ON c.category_id = p.category_id
LEFT JOIN categories parent ON parent.category_id = c.parent_category_id
GROUP BY ROLLUP (
    COALESCE(parent.category_name, c.category_name),
    CASE WHEN c.parent_category_id IS NOT NULL THEN c.category_name END
)
ORDER BY 1, 2 NULLS FIRST;
```

```
   major_category    |      sub_category      | total_sales
----------------------+-------------------------+-------------
 Books                |                         |     2670.00
 Books                |                         |     2670.00
 Electronics          |                         |    43420.00
 Electronics          | Computers & Laptops     |   230100.00
 Electronics          | Mobile Phones           |   128100.00
 Electronics          |                         |   401620.00
 Fashion              | Men's Clothing          |     4770.00
 Fashion              | Women's Clothing        |     3870.00
 Fashion              |                         |     8640.00
 Home Appliances      |                         |    17980.00
 Home Appliances      | Kitchen Appliances      |    16230.00
 Home Appliances      |                         |    34210.00
 Sports & Outdoors    |                         |     7060.00
 Sports & Outdoors    |                         |     7060.00
                      |                         |   454200.00
```

อ่านโครงสร้างผลลัพธ์:

- `ROLLUP(major_category, sub_category)` สร้าง 3 ระดับ: `(major, sub)` (รายละเอียด) → `(major)` (ผลรวมย่อยของแต่ละหมวดใหญ่) → `()` (ยอดรวมทั้งหมด)
- สำหรับ `Electronics` ซึ่งมีทั้งสินค้าที่ผูกกับหมวดใหญ่โดยตรง (เช่น Wireless Earbuds, Smart Watch) และสินค้าที่ผูกกับหมวดย่อย (Computers & Laptops, Mobile Phones) จะเห็นว่ามี**สองแถว**ที่ `sub_category` เป็นค่าว่าง: แถวหนึ่งคือยอดขาย 43,420.00 (สินค้าที่ผูกกับ Electronics โดยตรง ไม่มีหมวดย่อย) และอีกแถวคือ 401,620.00 (ผลรวมย่อยของ Electronics ทั้งหมด รวมทุกหมวดย่อย) — **สองแถวนี้ดูเหมือนกันแต่ความหมายต่างกันโดยสิ้นเชิง** นี่คือปัญหาคลาสสิกของ `ROLLUP`/`CUBE` ที่จะแก้ด้วย `GROUPING()` ใน Step 279
- แถวสุดท้ายที่ทั้งสองคอลัมน์ว่าง คือยอดรวมทั้งหมด 454,200.00 บาท

---

## Step 278: CUBE — สรุปทุกชุดผสมที่เป็นไปได้ของ dimension

`CUBE(a, b)` ต่างจาก `ROLLUP(a, b)` ตรงที่จะสร้าง**ทุกชุดผสมที่เป็นไปได้**ของคอลัมน์ที่ระบุ ไม่ใช่แค่แบบลดหลั่นจากซ้ายไปขวา เทียบเท่ากับ:

```sql
GROUPING SETS ( (a, b), (a), (b), () )
```

จำนวนชุดผสมของ `CUBE` คือ 2 ยกกำลังจำนวนคอลัมน์ (2 คอลัมน์ = 4 ชุด, 3 คอลัมน์ = 8 ชุด) จึงเหมาะกับ dimension ที่ไม่มีลำดับชั้นตายตัว และต้องการดูมุมมองแบบไขว้ (cross-tab) ทุกทิศทาง

ตัวอย่าง: สรุปยอดขายไขว้ระหว่างสถานะออเดอร์ (`status`) กับโซนจัดส่ง (แบ่งเป็นในประเทศ/ต่างประเทศ)

```sql
SELECT
    o.status,
    CASE WHEN o.ship_country = 'Thailand' THEN 'Domestic' ELSE 'International' END AS region,
    SUM(oi.quantity * oi.unit_price) AS total_sales
FROM order_items oi
JOIN orders o ON o.order_id = oi.order_id
GROUP BY CUBE (
    o.status,
    CASE WHEN o.ship_country = 'Thailand' THEN 'Domestic' ELSE 'International' END
)
ORDER BY o.status NULLS LAST, region NULLS LAST;
```

```
  status   |     region      | total_sales
-----------+-----------------+-------------
 cancelled | International   |    35390.00
 cancelled |                 |    35390.00
 completed | Domestic        |   105120.00
 completed | International   |   273810.00
 completed |                 |   378930.00
 pending   | International   |     3990.00
 pending   |                 |     3990.00
 shipped   | International   |    35890.00
 shipped   |                 |    35890.00
           | Domestic        |   105120.00
           | International   |   349080.00
           |                 |   454200.00
```

อ่านผลลัพธ์:

- 4 แถวแรกของแต่ละ `status` ที่มี `region` ระบุชัดเจน คือรายละเอียดจริง (ชุด `(status, region)`)
- แถวที่ `status` มีค่าแต่ `region` เป็น `NULL` คือผลรวมของ `status` นั้นข้ามทุกภูมิภาค (ชุด `(status)`)
- แถวที่ `status` เป็น `NULL` แต่ `region` มีค่า คือผลรวมของ `region` นั้นข้ามทุกสถานะ (ชุด `(region)`)
- แถวสุดท้ายที่ทั้งคู่เป็น `NULL` คือยอดรวมทั้งหมด (ชุด `()`)
- สังเกตว่าบางชุดผสมหายไปจากผลลัพธ์ เช่น `(cancelled, Domestic)` — เพราะไม่มีออเดอร์ที่ถูกยกเลิกและจัดส่งในประเทศไทยเลยในข้อมูลตัวอย่างนี้ `CUBE` จะไม่สร้างแถวสำหรับชุดผสมที่ไม่มีข้อมูลรองรับ (ต่างจากการ "เติมศูนย์ทุกช่อง" แบบ pivot table บางเครื่องมือ)
- อีกครั้งที่เห็นปัญหาความกำกวม: แถว `(cancelled, International)` = 35390.00 และแถว `(cancelled, NULL)` = 35390.00 มีค่าตัวเลขเท่ากันเพราะ `cancelled` มีข้อมูลเฉพาะฝั่ง International เท่านั้น — ต้องใช้ `GROUPING()` เพื่อแยกให้ชัดว่าแถวไหนคือรายละเอียด แถวไหนคือ subtotal

**ข้อควรระวัง**: `CUBE` กับ `ROLLUP` บนคอลัมน์ที่มี cardinality (จำนวนค่าที่เป็นไปได้) สูงหลายคอลัมน์พร้อมกัน จะทำให้จำนวนแถวผลลัพธ์เพิ่มแบบทวีคูณ (2^n สำหรับ CUBE) ควรใช้กับจำนวน dimension ที่จำกัด (2-4 คอลัมน์) และ cardinality ไม่สูงเกินไป มิฉะนั้นผลลัพธ์จะใหญ่และช้าเกินความจำเป็น

---

## Step 279: GROUPING() function — แยกแยะ subtotal และ grand total

จาก Step 277 และ 278 จะเห็นปัญหาสำคัญ: เมื่อคอลัมน์ที่จัดกลุ่มมีค่า `NULL` เป็นข้อมูลจริงอยู่แล้ว (เช่น สินค้าที่ผูกกับหมวดใหญ่โดยตรง ไม่มีหมวดย่อย) เราไม่สามารถแยกได้ด้วยตาเปล่าว่า `NULL` ในผลลัพธ์แถวหนึ่ง ๆ เป็น **"ค่า NULL จริงจากข้อมูล"** หรือ **"NULL ที่ ROLLUP/CUBE สร้างขึ้นเพื่อแทน subtotal/grand total"**

ฟังก์ชัน `GROUPING(column)` แก้ปัญหานี้โดยตรง: คืนค่า `0` ถ้าคอลัมน์นั้นถูกใช้จัดกลุ่มจริงในแถวนั้น (ค่าที่เห็นเป็นค่าจริง) และคืนค่า `1` ถ้าคอลัมน์นั้นถูก "รวบ" เป็น subtotal/grand total (ค่าที่เห็นเป็น NULL ที่มาจากการสรุป)

กลับไปที่ตัวอย่าง `ROLLUP` ใน Step 277 พร้อมเพิ่ม `GROUPING()`:

```sql
SELECT
    COALESCE(parent.category_name, c.category_name) AS major_category,
    CASE WHEN c.parent_category_id IS NOT NULL THEN c.category_name END AS sub_category,
    SUM(oi.quantity * oi.unit_price) AS total_sales,
    GROUPING(COALESCE(parent.category_name, c.category_name)) AS is_major_rolled_up,
    GROUPING(CASE WHEN c.parent_category_id IS NOT NULL THEN c.category_name END) AS is_sub_rolled_up,
    CASE
        WHEN GROUPING(COALESCE(parent.category_name, c.category_name)) = 1 THEN 'ยอดรวมทั้งหมด'
        WHEN GROUPING(CASE WHEN c.parent_category_id IS NOT NULL THEN c.category_name END) = 1
             THEN 'รวมย่อยตามหมวดใหญ่: ' || COALESCE(parent.category_name, c.category_name)
        ELSE 'รายละเอียด'
    END AS row_label
FROM order_items oi
JOIN products p    ON p.product_id = oi.product_id
JOIN categories c  ON c.category_id = p.category_id
LEFT JOIN categories parent ON parent.category_id = c.parent_category_id
GROUP BY ROLLUP (
    COALESCE(parent.category_name, c.category_name),
    CASE WHEN c.parent_category_id IS NOT NULL THEN c.category_name END
)
ORDER BY 1, 2 NULLS FIRST;
```

```
 major_category | sub_category | total_sales | is_major_rolled_up | is_sub_rolled_up |              row_label
-----------------+--------------+-------------+---------------------+-------------------+---------------------------------------
 Electronics     |              |    43420.00 |                   0 |                 0 | รายละเอียด
 Electronics     | Computers ...|   230100.00 |                   0 |                 0 | รายละเอียด
 Electronics     | Mobile Ph...  |   128100.00 |                   0 |                 0 | รายละเอียด
 Electronics     |              |   401620.00 |                   0 |                 1 | รวมย่อยตามหมวดใหญ่: Electronics
 ...
                 |              |   454200.00 |                   1 |                 1 | ยอดรวมทั้งหมด
```

ตอนนี้แถวที่ `total_sales = 43420.00` (สินค้าที่ผูกกับ Electronics โดยตรง) มี `is_sub_rolled_up = 0` เพราะเป็นค่าจริง ในขณะที่แถวที่ `total_sales = 401620.00` (ผลรวมย่อยของ Electronics ทั้งหมวด) มี `is_sub_rolled_up = 1` เพราะเป็น subtotal ที่ ROLLUP สร้างขึ้น — แยกความกำกวมได้ชัดเจนแล้ว

**`GROUPING()` กับหลายคอลัมน์พร้อมกัน**: สามารถส่งหลายคอลัมน์เข้า `GROUPING()` ในครั้งเดียวได้ ซึ่งจะคืนค่าเป็นเลขฐานสองรวมกัน (bitmask) มีประโยชน์เมื่อต้องเช็คหลายคอลัมน์พร้อมกันแบบกระชับ:

```sql
SELECT
    o.status,
    CASE WHEN o.ship_country = 'Thailand' THEN 'Domestic' ELSE 'International' END AS region,
    SUM(oi.quantity * oi.unit_price) AS total_sales,
    GROUPING(o.status, CASE WHEN o.ship_country = 'Thailand' THEN 'Domestic' ELSE 'International' END) AS grp_id
FROM order_items oi
JOIN orders o ON o.order_id = oi.order_id
GROUP BY CUBE (
    o.status,
    CASE WHEN o.ship_country = 'Thailand' THEN 'Domestic' ELSE 'International' END
)
ORDER BY grp_id, o.status NULLS LAST;
```

ค่า `grp_id` ที่ได้: `0` = รายละเอียดครบทั้งสองคอลัมน์, `1` = คอลัมน์ที่สอง (region) ถูกรวบ, `2` = คอลัมน์แรก (status) ถูกรวบ, `3` = ทั้งสองคอลัมน์ถูกรวบ (grand total) — ใช้ `grp_id` เป็นตัวช่วยจัดลำดับการแสดงผลรายงานได้สะดวก (เช่น จัดให้ grand total อยู่ล่างสุดเสมอ)

---

## Step 280: แบบฝึกหัดรวม — รายงานยอดขายหลายมิติด้วย ROLLUP/CUBE

มาประกอบทุกเทคนิคที่เรียนมาเข้าด้วยกัน เพื่อสร้างรายงานยอดขายแบบมืออาชีพที่มีหลายมิติ: **หมวดหมู่สินค้า (major category) × เดือน** โดยใช้ `ROLLUP` เพื่อให้ได้ลำดับชั้น: รายละเอียดต่อเดือน → รวมต่อหมวดหมู่ (ทุกเดือน) → รวมทั้งหมด

```sql
SELECT
    COALESCE(parent.category_name, c.category_name)  AS major_category,
    to_char(o.order_date, 'YYYY-MM')                   AS sales_month,
    SUM(oi.quantity * oi.unit_price)                   AS total_sales,
    GROUPING(to_char(o.order_date, 'YYYY-MM'))          AS is_month_subtotal,
    GROUPING(COALESCE(parent.category_name, c.category_name)) AS is_grand_total
FROM order_items oi
JOIN orders o       ON o.order_id = oi.order_id
JOIN products p     ON p.product_id = oi.product_id
JOIN categories c   ON c.category_id = p.category_id
LEFT JOIN categories parent ON parent.category_id = c.parent_category_id
GROUP BY ROLLUP (
    COALESCE(parent.category_name, c.category_name),
    to_char(o.order_date, 'YYYY-MM')
)
ORDER BY
    is_grand_total,
    major_category NULLS LAST,
    is_month_subtotal,
    sales_month NULLS LAST;
```

ผลลัพธ์ตัวอย่าง (แสดงเฉพาะหมวด `Electronics` ทั้งหมด + แถวยอดรวม เพื่อความกระชับ — หมวดหมู่อื่นแสดงในลักษณะเดียวกันเมื่อรันคิวรีจริง):

```
 major_category | sales_month | total_sales | is_month_subtotal | is_grand_total
-----------------+-------------+-------------+---------------------+-----------------
 Electronics     | 2024-01     |    98780.00 |                   0 |               0
 Electronics     | 2024-02     |    17800.00 |                   0 |               0
 Electronics     | 2024-03     |    67290.00 |                   0 |               0
 Electronics     | 2024-04     |    61180.00 |                   0 |               0
 Electronics     | 2024-05     |    85300.00 |                   0 |               0
 Electronics     | 2024-06     |    71270.00 |                   0 |               0
 Electronics     |             |   401620.00 |                   1 |               0
 ...             | ...         |         ... |                 ... |             ...
                 |             |   454200.00 |                   1 |               1
```

*(ผลลัพธ์เต็มมีทั้งหมด 5 หมวดใหญ่ × สูงสุด 6 เดือน + 5 แถว subtotal ต่อหมวด + 1 แถว grand total — รันคิวรีเองเพื่อดูรายงานฉบับเต็ม)*

**ขยายเป็นมุมมองไขว้เต็มรูปแบบด้วย `CUBE`**: หากต้องการดูยอดขายไขว้ระหว่างหมวดหมู่และประเทศแบบครบทุกทิศทาง (ไม่ใช่แค่ลำดับชั้น) ให้เปลี่ยนจาก `ROLLUP` เป็น `CUBE`:

```sql
SELECT
    COALESCE(parent.category_name, c.category_name) AS major_category,
    o.ship_country,
    SUM(oi.quantity * oi.unit_price) AS total_sales,
    GROUPING(COALESCE(parent.category_name, c.category_name), o.ship_country) AS grp_id
FROM order_items oi
JOIN orders o       ON o.order_id = oi.order_id
JOIN products p     ON p.product_id = oi.product_id
JOIN categories c   ON c.category_id = p.category_id
LEFT JOIN categories parent ON parent.category_id = c.parent_category_id
GROUP BY CUBE (
    COALESCE(parent.category_name, c.category_name),
    o.ship_country
)
ORDER BY grp_id, major_category NULLS LAST, o.ship_country NULLS LAST;
```

คิวรีนี้ให้ทั้งยอดขายราย (หมวดหมู่, ประเทศ), ยอดรวมต่อหมวดหมู่ข้ามทุกประเทศ, ยอดรวมต่อประเทศข้ามทุกหมวดหมู่ และยอดรวมทั้งหมด — ครบทุกมุมมองในคิวรีเดียว เหมาะสำหรับสร้าง pivot table หรือ export ไปทำ dashboard ต่อ

**แนวทางเลือกใช้งานจริง**:

| ต้องการ | ใช้ |
|---|---|
| สรุปเพียงระดับเดียว | `GROUP BY` ปกติ |
| สรุปหลายชุดที่ไม่สัมพันธ์กันแบบลำดับชั้น | `GROUPING SETS` |
| สรุปแบบลำดับชั้น (parent → child, ปี → เดือน → วัน) | `ROLLUP` |
| สรุปแบบไขว้ทุกมุมมองของ dimension ที่เท่าเทียมกัน | `CUBE` |
| ต้องแยกแถว subtotal/grand total ออกจากข้อมูลจริง | เพิ่ม `GROUPING()` เสมอ |

---

## สรุปท้ายบท

- `GROUP BY` จัดกลุ่มแถวตามค่าคอลัมน์ที่ระบุ แล้วให้ aggregate function คำนวณค่าสรุปต่อกลุ่ม ยิ่งระบุหลายคอลัมน์ ยิ่งได้รายงานละเอียดขึ้น
- `WHERE` กรองแถวดิบก่อนจัดกลุ่ม (ใช้ aggregate function ไม่ได้) ส่วน `HAVING` กรองกลุ่มหลังคำนวณ aggregate แล้ว (ใช้ aggregate function ได้) — ควรกรองด้วย `WHERE` ให้มากที่สุดเท่าที่ทำได้เพื่อประสิทธิภาพที่ดีกว่า
- ทุกคอลัมน์ใน `SELECT` ที่ไม่ใช่ aggregate function ต้องอยู่ใน `GROUP BY` ยกเว้นกรณี PostgreSQL รู้ functional dependency จาก primary key
- การ `JOIN` ตารางแบบ 1-ต่อ-หลายก่อน `GROUP BY` ต้องระวังการนับซ้ำ ใช้ `COUNT(DISTINCT ...)` และเลือกใช้ `LEFT JOIN` เมื่อต้องการรวมกลุ่มที่ไม่มีข้อมูลจับคู่ด้วย
- `GROUPING SETS` สร้างหลายระดับการสรุปในคิวรีเดียว แทนการ `UNION ALL` หลายคิวรี ทำให้สแกนข้อมูลครั้งเดียวและเร็วกว่า
- `ROLLUP(a, b, c)` สร้างชุดสรุปแบบลดหลั่นจากซ้ายไปขวา เหมาะกับข้อมูลที่มีลำดับชั้นตามธรรมชาติ
- `CUBE(a, b, c)` สร้างทุกชุดผสมที่เป็นไปได้ (2^n ชุด) เหมาะกับ dimension ที่ต้องการมุมมองไขว้ทุกทิศทาง
- `GROUPING(column)` คืนค่า `0` หากคอลัมน์นั้นเป็นค่าจริงในแถวนั้น และ `1` หากเป็น `NULL` ที่เกิดจากการสรุป (subtotal/grand total) — จำเป็นเมื่อคอลัมน์ที่จัดกลุ่มอาจมีค่า `NULL` จริงปนอยู่ในข้อมูล เพื่อไม่ให้สับสนกับ `NULL` ที่ ROLLUP/CUBE สร้างขึ้น

บทถัดไปจะเริ่มเข้าสู่ **Window Functions** ซึ่งเป็นเครื่องมือทรงพลังสำหรับการคำนวณค่าต่อแถวโดยยังคงเห็นแถวอื่นในกลุ่มเดียวกันได้ ต่างจาก `GROUP BY` ที่ยุบหลายแถวเหลือแถวเดียว — Window Functions จะรักษาจำนวนแถวเดิมไว้ครบทุกแถว พร้อมคำนวณค่าสรุปประกอบไปด้วย

**บทถัดไป**: [Part 029 — Window Functions พื้นฐาน](./part-029-window-functions-basics.md)

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1

หายอดขายรวม (`quantity * unit_price`) แยกตามหมวดหมู่สินค้า (`category_name` จากตาราง `products` โดยตรง ไม่ต้องดูหมวดใหญ่) เรียงจากมากไปน้อย

<details>
<summary>เฉลย</summary>

```sql
SELECT
    c.category_name,
    SUM(oi.quantity * oi.unit_price) AS total_sales
FROM order_items oi
JOIN products p   ON p.product_id = oi.product_id
JOIN categories c ON c.category_id = p.category_id
GROUP BY c.category_name
ORDER BY total_sales DESC;
```

ผลลัพธ์ที่คาดหวัง (แถวบนสุด): `Computers & Laptops` = 230,100.00, `Mobile Phones` = 128,100.00

</details>

### แบบฝึกหัดที่ 2

หาจำนวนลูกค้าที่สมัครสมาชิกในแต่ละประเทศ เฉพาะประเทศที่มีลูกค้าตั้งแต่ 2 คนขึ้นไป เรียงจากมากไปน้อย

<details>
<summary>เฉลย</summary>

```sql
SELECT country, COUNT(*) AS customer_count
FROM customers
GROUP BY country
HAVING COUNT(*) >= 2
ORDER BY customer_count DESC, country;
```

ผลลัพธ์: `Thailand` = 3 คน ส่วนที่เหลือ (USA, Japan, Singapore, UK, Germany, Australia) มี 2 คนเท่ากันทุกประเทศ

</details>

### แบบฝึกหัดที่ 3

หาพนักงานที่มียอดขายรวม (จากออเดอร์ที่ `status = 'completed'` เท่านั้น) มากกว่า 100,000 บาท พร้อมแสดงจำนวนออเดอร์ที่รับผิดชอบ

<details>
<summary>เฉลย</summary>

```sql
SELECT
    e.employee_id,
    e.first_name || ' ' || e.last_name AS employee_name,
    COUNT(DISTINCT o.order_id)         AS orders_handled,
    SUM(oi.quantity * oi.unit_price)   AS total_sales
FROM employees e
JOIN orders o       ON o.employee_id = e.employee_id
JOIN order_items oi ON oi.order_id = o.order_id
WHERE o.status = 'completed'
GROUP BY e.employee_id, employee_name
HAVING SUM(oi.quantity * oi.unit_price) > 100000
ORDER BY total_sales DESC;
```

ผลลัพธ์: มีพนักงานที่เข้าเงื่อนไข 2 คน คือ Kanya Rattana (170,660.00) และ Korn Suwannakit (109,690.00)

</details>

### แบบฝึกหัดที่ 4

หาสินค้าที่มีคะแนนรีวิวเฉลี่ยตั้งแต่ 4.0 ขึ้นไป พร้อมจำนวนรีวิวทั้งหมด เรียงตามคะแนนเฉลี่ยจากมากไปน้อย

<details>
<summary>เฉลย</summary>

```sql
SELECT
    p.product_name,
    COUNT(*)                    AS review_count,
    ROUND(AVG(r.rating), 2)     AS avg_rating
FROM reviews r
JOIN products p ON p.product_id = r.product_id
GROUP BY p.product_name
HAVING AVG(r.rating) >= 4.0
ORDER BY avg_rating DESC, review_count DESC;
```

ตัวอย่างผลลัพธ์: สินค้าที่มีรีวิวเดียวและได้ 5 ดาวจะขึ้นนำ เช่น `Robot Vacuum Cleaner`, `Gaming Laptop X`, `PostgreSQL Mastery Book` เป็นต้น ตามด้วยสินค้าที่มีหลายรีวิวและค่าเฉลี่ยสูง

</details>

### แบบฝึกหัดที่ 5

หายอดขายรวมแยกตามเดือน (`YYYY-MM` จาก `order_date`) และสถานะออเดอร์ (`status`) โดยใช้ `GROUP BY` สองคอลัมน์

<details>
<summary>เฉลย</summary>

```sql
SELECT
    to_char(o.order_date, 'YYYY-MM') AS sales_month,
    o.status,
    SUM(oi.quantity * oi.unit_price) AS total_sales
FROM order_items oi
JOIN orders o ON o.order_id = oi.order_id
GROUP BY sales_month, o.status
ORDER BY sales_month, o.status;
```

ผลลัพธ์จะมีหนึ่งแถวต่อทุกชุดผสม (เดือน, สถานะ) ที่พบจริงในข้อมูล เช่น `2024-02 | cancelled | 2490.00`, `2024-02 | completed | 28380.00` เป็นต้น

</details>

### แบบฝึกหัดที่ 6

ใช้ `HAVING` หาหมวดหมู่สินค้าที่มีจำนวนสินค้าที่ `is_active = true` น้อยกว่า 3 ชิ้น (ต้องกรองด้วย `WHERE` ก่อน แล้วนับด้วย `HAVING`)

<details>
<summary>เฉลย</summary>

```sql
SELECT
    c.category_name,
    COUNT(*) AS active_product_count
FROM products p
JOIN categories c ON c.category_id = p.category_id
WHERE p.is_active = true
GROUP BY c.category_name
HAVING COUNT(*) < 3
ORDER BY active_product_count, c.category_name;
```

หมายเหตุ: `Sports & Outdoors` มีสินค้าทั้งหมด 2 ชิ้น แต่ `Camping Tent 4-Person` มี `is_active = false` จึงเหลือสินค้า active เพียง 1 ชิ้น (`Yoga Mat Premium`) ทำให้เข้าเงื่อนไข

</details>

### แบบฝึกหัดที่ 7

ใช้ `GROUPING SETS` สร้างรายงานยอดขายสรุปตาม (หมวดหมู่สินค้า, วิธีการชำระเงิน) และยอดรวมทั้งหมด ในคิวรีเดียว (ใช้ตาราง `payments` join กับ `orders`, `order_items`, `products`, `categories`)

<details>
<summary>เฉลย</summary>

```sql
SELECT
    c.category_name,
    pm.payment_method,
    SUM(oi.quantity * oi.unit_price) AS total_sales
FROM order_items oi
JOIN orders o      ON o.order_id = oi.order_id
JOIN payments pm   ON pm.order_id = o.order_id
JOIN products p    ON p.product_id = oi.product_id
JOIN categories c  ON c.category_id = p.category_id
GROUP BY GROUPING SETS (
    (c.category_name, pm.payment_method),
    ()
)
ORDER BY c.category_name NULLS LAST, pm.payment_method NULLS LAST;
```

หมายเหตุ: joined ผ่าน `payments` ทำให้เฉพาะออเดอร์ที่มีการชำระเงินแล้ว (`completed`/`shipped`) เท่านั้นที่ปรากฏในรายงาน ออเดอร์ที่ `pending`/`cancelled` จะไม่มีข้อมูลในตาราง `payments` จึงไม่ถูกนับ

</details>

### แบบฝึกหัดที่ 8

ใช้ `ROLLUP` หายอดขายของออเดอร์ที่ `status = 'completed'` ตามลำดับชั้น ปี → เดือน (ให้แสดงผลรวมย่อยของแต่ละปี และยอดรวมทั้งหมด)

<details>
<summary>เฉลย</summary>

```sql
SELECT
    to_char(o.order_date, 'YYYY') AS sales_year,
    to_char(o.order_date, 'YYYY-MM') AS sales_month,
    SUM(oi.quantity * oi.unit_price) AS total_sales
FROM order_items oi
JOIN orders o ON o.order_id = oi.order_id
WHERE o.status = 'completed'
GROUP BY ROLLUP (
    to_char(o.order_date, 'YYYY'),
    to_char(o.order_date, 'YYYY-MM')
)
ORDER BY sales_year NULLS LAST, sales_month NULLS LAST;
```

เนื่องจากข้อมูลตัวอย่างทั้งหมดอยู่ในปี 2024 ปีเดียว ผลลัพธ์จะมีแถวรายเดือนของปี 2024 ตามด้วยแถวรวมของปี 2024 (`sales_month IS NULL`) และแถวสุดท้ายเป็นยอดรวมทั้งหมด (`sales_year IS NULL`) ซึ่งในกรณีนี้ค่าจะเท่ากับแถวรวมของปี 2024 พอดี

</details>

### แบบฝึกหัดที่ 9

ใช้ `CUBE` หายอดขายไขว้ระหว่างหมวดหมู่สินค้าใหญ่ (`Electronics`, `Home Appliances`, `Fashion`, `Books`, `Sports & Outdoors`) และวิธีการชำระเงิน (`payment_method`)

<details>
<summary>เฉลย</summary>

```sql
SELECT
    COALESCE(parent.category_name, c.category_name) AS major_category,
    pm.payment_method,
    SUM(oi.quantity * oi.unit_price) AS total_sales
FROM order_items oi
JOIN orders o       ON o.order_id = oi.order_id
JOIN payments pm    ON pm.order_id = o.order_id
JOIN products p     ON p.product_id = oi.product_id
JOIN categories c   ON c.category_id = p.category_id
LEFT JOIN categories parent ON parent.category_id = c.parent_category_id
GROUP BY CUBE (
    COALESCE(parent.category_name, c.category_name),
    pm.payment_method
)
ORDER BY major_category NULLS LAST, pm.payment_method NULLS LAST;
```

ผลลัพธ์จะมีแถวรายละเอียดของแต่ละ (หมวดใหญ่, วิธีชำระเงิน) ที่มีข้อมูลจริง ตามด้วยแถวรวมต่อหมวดใหญ่ (payment_method เป็น NULL) แถวรวมต่อวิธีชำระเงิน (major_category เป็น NULL) และแถวยอดรวมทั้งหมด (ทั้งคู่เป็น NULL)

</details>

### แบบฝึกหัดที่ 10

ปรับปรุงคิวรีจากแบบฝึกหัดที่ 8 ให้เพิ่มคอลัมน์ที่บอกว่าแถวนั้นเป็น "รายละเอียดรายเดือน", "ผลรวมรายปี" หรือ "ยอดรวมทั้งหมด" โดยใช้ฟังก์ชัน `GROUPING()`

<details>
<summary>เฉลย</summary>

```sql
SELECT
    to_char(o.order_date, 'YYYY') AS sales_year,
    to_char(o.order_date, 'YYYY-MM') AS sales_month,
    SUM(oi.quantity * oi.unit_price) AS total_sales,
    CASE
        WHEN GROUPING(to_char(o.order_date, 'YYYY')) = 1 THEN 'ยอดรวมทั้งหมด'
        WHEN GROUPING(to_char(o.order_date, 'YYYY-MM')) = 1 THEN 'ผลรวมรายปี'
        ELSE 'รายละเอียดรายเดือน'
    END AS row_type
FROM order_items oi
JOIN orders o ON o.order_id = oi.order_id
WHERE o.status = 'completed'
GROUP BY ROLLUP (
    to_char(o.order_date, 'YYYY'),
    to_char(o.order_date, 'YYYY-MM')
)
ORDER BY sales_year NULLS LAST, sales_month NULLS LAST;
```

ผลลัพธ์: แถวรายเดือนทั้งหมดจะมี `row_type = 'รายละเอียดรายเดือน'`, แถวที่ `sales_month IS NULL` แต่ `sales_year` ยังมีค่า จะมี `row_type = 'ผลรวมรายปี'`, และแถวสุดท้ายที่ทั้งสองคอลัมน์เป็น `NULL` จะมี `row_type = 'ยอดรวมทั้งหมด'` — เป็นเทคนิคที่ใช้บ่อยมากในการสร้างรายงานที่ต้องแสดงผลใน BI tool หรือหน้าเว็บ เพื่อจัด style (ตัวหนา, สีพื้นหลัง) ให้แถว subtotal แตกต่างจากแถวรายละเอียด

</details>

---

**บทถัดไป**: [Part 029 — Window Functions พื้นฐาน](./part-029-window-functions-basics.md)
