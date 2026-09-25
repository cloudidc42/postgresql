# Window Functions พื้นฐาน: OVER, PARTITION BY

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับกลาง | Part 029

## เป้าหมายการเรียนรู้

เมื่อเรียนจบบทนี้ ผู้เรียนจะสามารถ:

- อธิบายความแตกต่างระหว่าง **aggregate function** ธรรมดา (ที่ใช้กับ `GROUP BY`) กับ **window function** ได้อย่างชัดเจน โดยเฉพาะเรื่องการ "ไม่ยุบแถว"
- ใช้ `OVER()` เพื่อคำนวณค่าทางสถิติ (SUM, AVG, COUNT) โดยมองข้อมูลทั้งหมดเป็น window เดียว
- ใช้ `PARTITION BY` เพื่อแบ่งข้อมูลเป็นกลุ่มย่อยก่อนคำนวณ window function ในแต่ละกลุ่มแยกกัน
- นำ aggregate function เช่น `SUM()`, `AVG()`, `COUNT()` มาใช้ในรูปแบบ window function ได้อย่างถูกต้อง
- เปรียบเทียบค่าของแต่ละแถวกับค่าเฉลี่ยของกลุ่มที่แถวนั้นสังกัดอยู่ (เช่น เปรียบเทียบยอดขายต่อ order กับค่าเฉลี่ยของลูกค้าคนนั้น)
- ใช้ `ORDER BY` ภายใน `OVER()` เพื่อสร้าง running total และ running average
- เข้าใจ **frame clause** เบื้องต้น: `ROWS BETWEEN`, `RANGE BETWEEN`, `UNBOUNDED PRECEDING/FOLLOWING`
- สร้าง moving average และ running total ด้วย frame ที่กำหนดเอง
- รวม window function หลายตัวในคิวรีเดียว และใช้ `WINDOW` clause เพื่อลดการเขียนซ้ำ
- ประยุกต์ใช้ความรู้ทั้งหมดสร้างรายงานวิเคราะห์ยอดขายแบบ running total และเปรียบเทียบราคาสินค้ากับค่าเฉลี่ยของหมวดหมู่

---

## เตรียมข้อมูล

บทนี้ใช้ชุดข้อมูลอีคอมเมิร์ซเดียวกันกับ Part 021-039 ทั้งหมด หากคุณสร้างฐานข้อมูลนี้ไว้แล้วจากบทก่อนหน้า สามารถข้ามส่วนนี้ไปได้ แต่ถ้าต้องการรันคิวรีในบทนี้แบบสด ๆ แนะนำให้สร้างฐานข้อมูลใหม่แล้วรันสคริปต์ด้านล่างทั้งหมด เพื่อให้ผลลัพธ์ตรงกับตัวอย่างในเอกสารทุกประการ

### โครงสร้างตาราง (Schema)

```sql
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
```

### ข้อมูลตัวอย่าง (Seed Data)

```sql
-- categories
INSERT INTO categories (category_id, category_name, parent_category_id) VALUES
(1, 'Electronics', NULL),
(2, 'Computers', 1),
(3, 'Smartphones', 1),
(4, 'Home & Kitchen', NULL),
(5, 'Furniture', 4),
(6, 'Books', NULL),
(7, 'Fiction', 6),
(8, 'Non-Fiction', 6);
SELECT setval('categories_category_id_seq', 8);

-- suppliers
INSERT INTO suppliers (supplier_id, supplier_name, country) VALUES
(1, 'Global Tech Supply', 'USA'),
(2, 'Bangkok Electronics Co.', 'Thailand'),
(3, 'EuroHome Distribution', 'Germany'),
(4, 'Pacific Traders', 'Singapore'),
(5, 'NordicDesign Furniture', 'Sweden'),
(6, 'Sunrise Books Publishing', 'Thailand'),
(7, 'Shenzhen Digital Ltd.', 'China'),
(8, 'American Appliance Corp.', 'USA');
SELECT setval('suppliers_supplier_id_seq', 8);

-- products
INSERT INTO products (product_id, product_name, category_id, supplier_id, unit_price, stock_quantity, is_active) VALUES
(1, 'Laptop Pro 14', 2, 1, 42900.00, 25, true),
(2, 'Laptop Air 13', 2, 7, 32900.00, 40, true),
(3, 'Wireless Mouse', 2, 2, 590.00, 200, true),
(4, 'Mechanical Keyboard', 2, 2, 2490.00, 80, true),
(5, 'Smartphone X12', 3, 7, 24900.00, 60, true),
(6, 'Smartphone Lite', 3, 7, 9900.00, 90, true),
(7, 'Bluetooth Earbuds', 3, 2, 1990.00, 150, true),
(8, 'Smartwatch Series 5', 3, 1, 8900.00, 45, true),
(9, 'Dining Table Oak', 5, 5, 15900.00, 10, true),
(10, 'Office Chair Ergo', 5, 5, 5900.00, 35, true),
(11, 'Bookshelf 5-Tier', 5, 3, 3200.00, 20, true),
(12, 'Sofa 3-Seater', 5, 5, 22900.00, 8, true),
(13, 'Rice Cooker Deluxe', 4, 8, 1590.00, 60, true),
(14, 'Air Fryer XL', 4, 8, 2990.00, 55, true),
(15, 'The Art of Programming', 8, 6, 590.00, 100, true),
(16, 'Thai Cuisine Cookbook', 8, 6, 450.00, 70, true),
(17, 'Mystery of the Lost City', 7, 6, 350.00, 85, true),
(18, 'Whispers in the Dark', 7, 6, 380.00, 65, false);
SELECT setval('products_product_id_seq', 18);

-- customers
INSERT INTO customers (customer_id, first_name, last_name, email, country, signup_date) VALUES
(1, 'Somchai', 'Jaidee', 'somchai.j@example.com', 'Thailand', '2023-01-15'),
(2, 'Nattaya', 'Suksawat', 'nattaya.s@example.com', 'Thailand', '2023-02-20'),
(3, 'John', 'Smith', 'john.smith@example.com', 'USA', '2023-03-05'),
(4, 'Emma', 'Johnson', 'emma.j@example.com', 'USA', '2023-03-18'),
(5, 'Kenji', 'Tanaka', 'kenji.t@example.com', 'Japan', '2023-04-02'),
(6, 'Li', 'Wei', 'li.wei@example.com', 'China', '2023-04-25'),
(7, 'Piyanuch', 'Rattana', 'piyanuch.r@example.com', 'Thailand', '2023-05-10'),
(8, 'Michael', 'Brown', 'michael.b@example.com', 'UK', '2023-05-22'),
(9, 'Sirinya', 'Boonmee', 'sirinya.b@example.com', 'Thailand', '2023-06-14'),
(10, 'David', 'Wilson', 'david.w@example.com', 'USA', '2023-07-01'),
(11, 'Anong', 'Phetchara', 'anong.p@example.com', 'Thailand', '2023-07-19'),
(12, 'Sophie', 'Martin', 'sophie.m@example.com', 'France', '2023-08-08'),
(13, 'Thawatchai', 'Meesuk', 'thawatchai.m@example.com', 'Thailand', '2023-09-01'),
(14, 'Yuki', 'Yamamoto', 'yuki.y@example.com', 'Japan', '2023-09-20');
SELECT setval('customers_customer_id_seq', 14);

-- employees
INSERT INTO employees (employee_id, first_name, last_name, hire_date, manager_id, department) VALUES
(1, 'Suda', 'Charoen', '2020-01-10', NULL, 'Sales'),
(2, 'Anan', 'Wongsa', '2020-03-15', 1, 'Sales'),
(3, 'Kittipong', 'Srisuk', '2021-02-01', 1, 'Sales'),
(4, 'Malee', 'Thongdee', '2021-06-20', 1, 'Sales'),
(5, 'Preecha', 'Boonrod', '2022-01-05', 1, 'Customer Service'),
(6, 'Ratana', 'Wilaiwan', '2022-08-11', 1, 'Customer Service'),
(7, 'Chalermchai', 'Intharak', '2023-01-20', 1, 'Sales');
SELECT setval('employees_employee_id_seq', 7);

-- orders
INSERT INTO orders (order_id, customer_id, employee_id, order_date, status, ship_country) VALUES
(1, 1, 2, '2024-08-01 09:15:00+07', 'completed', 'Thailand'),
(2, 3, 3, '2024-08-01 11:30:00+07', 'completed', 'USA'),
(3, 5, 2, '2024-08-01 14:00:00+07', 'completed', 'Japan'),
(4, 2, 4, '2024-08-02 10:00:00+07', 'completed', 'Thailand'),
(5, 7, 2, '2024-08-02 13:20:00+07', 'completed', 'Thailand'),
(6, 4, 3, '2024-08-03 09:45:00+07', 'completed', 'USA'),
(7, 9, 4, '2024-08-03 15:10:00+07', 'completed', 'Thailand'),
(8, 6, 3, '2024-08-04 10:30:00+07', 'completed', 'China'),
(9, 1, 2, '2024-08-04 16:00:00+07', 'completed', 'Thailand'),
(10, 8, 4, '2024-08-05 11:00:00+07', 'cancelled', 'UK'),
(11, 11, 2, '2024-08-05 14:45:00+07', 'completed', 'Thailand'),
(12, 10, 3, '2024-08-06 09:00:00+07', 'completed', 'USA'),
(13, 3, 3, '2024-08-06 13:30:00+07', 'completed', 'USA'),
(14, 13, 4, '2024-08-07 10:15:00+07', 'completed', 'Thailand'),
(15, 5, 2, '2024-08-07 15:50:00+07', 'completed', 'Japan'),
(16, 12, 3, '2024-08-08 11:20:00+07', 'completed', 'France'),
(17, 2, 4, '2024-08-08 16:40:00+07', 'completed', 'Thailand'),
(18, 14, 2, '2024-08-09 09:30:00+07', 'completed', 'Japan'),
(19, 9, 4, '2024-08-09 14:10:00+07', 'completed', 'Thailand'),
(20, 7, 2, '2024-08-10 10:50:00+07', 'pending', 'Thailand');
SELECT setval('orders_order_id_seq', 20);

-- order_items
INSERT INTO order_items (order_item_id, order_id, product_id, quantity, unit_price) VALUES
(1, 1, 1, 1, 42900.00),
(2, 1, 3, 2, 590.00),
(3, 2, 5, 1, 24900.00),
(4, 2, 7, 1, 1990.00),
(5, 3, 6, 1, 9900.00),
(6, 4, 2, 1, 32900.00),
(7, 4, 4, 1, 2490.00),
(8, 5, 15, 3, 590.00),
(9, 5, 17, 2, 350.00),
(10, 6, 9, 1, 15900.00),
(11, 7, 13, 2, 1590.00),
(12, 7, 14, 1, 2990.00),
(13, 8, 8, 1, 8900.00),
(14, 9, 3, 1, 590.00),
(15, 9, 4, 1, 2490.00),
(16, 9, 7, 2, 1990.00),
(17, 11, 10, 1, 5900.00),
(18, 11, 11, 1, 3200.00),
(19, 12, 1, 1, 42900.00),
(20, 12, 3, 3, 590.00),
(21, 13, 5, 1, 24900.00),
(22, 14, 16, 4, 450.00),
(23, 14, 15, 2, 590.00),
(24, 15, 6, 1, 9900.00),
(25, 15, 7, 1, 1990.00),
(26, 16, 12, 1, 22900.00),
(27, 17, 2, 1, 32900.00),
(28, 18, 8, 1, 8900.00),
(29, 18, 7, 2, 1990.00),
(30, 19, 13, 1, 1590.00),
(31, 19, 14, 1, 2990.00),
(32, 20, 17, 1, 350.00),
(33, 20, 18, 1, 380.00);
SELECT setval('order_items_order_item_id_seq', 33);

-- reviews
INSERT INTO reviews (review_id, product_id, customer_id, rating, review_text, review_date) VALUES
(1, 1, 3, 5, 'Great laptop, fast and light.', '2024-08-05'),
(2, 5, 5, 4, 'Good phone but battery could be better.', '2024-08-06'),
(3, 7, 2, 5, 'Amazing sound quality!', '2024-08-07'),
(4, 9, 4, 4, 'Sturdy table, easy to assemble.', '2024-08-08'),
(5, 13, 9, 3, 'Works fine, a bit noisy.', '2024-08-09'),
(6, 15, 1, 5, 'Best programming book I have read.', '2024-08-10'),
(7, 17, 7, 4, 'Gripping mystery novel.', '2024-08-11'),
(8, 2, 12, 5, 'Super lightweight, perfect for travel.', '2024-08-12'),
(9, 6, 14, 3, 'Average phone for the price.', '2024-08-13'),
(10, 10, 11, 5, 'Very comfortable chair.', '2024-08-14'),
(11, 8, 8, 4, 'Nice smartwatch, good battery.', '2024-08-15'),
(12, 14, 13, 4, 'Air fryer works great.', '2024-08-16');
SELECT setval('reviews_review_id_seq', 12);

-- payments
INSERT INTO payments (payment_id, order_id, payment_date, amount, payment_method) VALUES
(1, 1, '2024-08-01 09:20:00+07', 44080.00, 'credit_card'),
(2, 2, '2024-08-01 11:35:00+07', 26890.00, 'credit_card'),
(3, 3, '2024-08-01 14:05:00+07', 9900.00, 'paypal'),
(4, 4, '2024-08-02 10:05:00+07', 35390.00, 'credit_card'),
(5, 5, '2024-08-02 13:25:00+07', 2470.00, 'paypal'),
(6, 6, '2024-08-03 09:50:00+07', 15900.00, 'bank_transfer'),
(7, 7, '2024-08-03 15:15:00+07', 6170.00, 'credit_card'),
(8, 8, '2024-08-04 10:35:00+07', 8900.00, 'credit_card'),
(9, 9, '2024-08-04 16:05:00+07', 7060.00, 'paypal'),
(10, 11, '2024-08-05 14:50:00+07', 9100.00, 'credit_card'),
(11, 12, '2024-08-06 09:05:00+07', 44670.00, 'credit_card'),
(12, 13, '2024-08-06 13:35:00+07', 24900.00, 'paypal'),
(13, 14, '2024-08-07 10:20:00+07', 2980.00, 'credit_card'),
(14, 15, '2024-08-07 15:55:00+07', 11890.00, 'bank_transfer'),
(15, 16, '2024-08-08 11:25:00+07', 22900.00, 'credit_card'),
(16, 17, '2024-08-08 16:45:00+07', 32900.00, 'credit_card'),
(17, 18, '2024-08-09 09:35:00+07', 12880.00, 'paypal'),
(18, 19, '2024-08-09 14:15:00+07', 4580.00, 'credit_card');
SELECT setval('payments_payment_id_seq', 18);
```

> **หมายเหตุ:** order `10` ถูกยกเลิก (`cancelled`) จึงไม่มี `order_items` และ `payments` ส่วน order `20` ยังอยู่ในสถานะ `pending` มี `order_items` แล้วแต่ยังไม่มี `payments` — ข้อมูลสองรายการนี้จงใจใส่ไว้เพื่อฝึกกรองข้อมูลด้วย `WHERE status = 'completed'` ก่อนคำนวณ window function ในหลายตัวอย่างของบทนี้

---

## Step 281: Window function คืออะไร — ความแตกต่างจาก aggregate function

ก่อนจะไปถึง syntax เรามาทำความเข้าใจแนวคิดหลักก่อน เพราะเป็นสิ่งที่ผู้เรียนใหม่สับสนมากที่สุด

**Aggregate function** ที่ใช้คู่กับ `GROUP BY` (เช่น `SUM()`, `AVG()`, `COUNT()` ที่เราเรียนใน Part ก่อนหน้า) จะ **ยุบหลายแถวให้เหลือแถวเดียวต่อกลุ่ม** ตัวอย่างเช่น ถ้าเรา group สินค้าตาม `category_id` แล้วหา `AVG(unit_price)` ผลลัพธ์จะมีแค่ 1 แถวต่อ 1 หมวดหมู่ ข้อมูลระดับสินค้าแต่ละชิ้นจะหายไป

```sql
-- Aggregate function ธรรมดา: ยุบแถวเหลือ 1 แถวต่อ category
SELECT category_id, COUNT(*) AS product_count, AVG(unit_price) AS avg_price
FROM products
WHERE is_active = true
GROUP BY category_id
ORDER BY category_id;
```

ผลลัพธ์:

| category_id | product_count | avg_price |
|---|---|---|
| 2 | 4 | 19720.00 |
| 3 | 4 | 11422.50 |
| 4 | 2 | 2290.00 |
| 5 | 4 | 11975.00 |
| 7 | 1 | 350.00 |
| 8 | 2 | 520.00 |

สังเกตว่าเราไม่เห็นชื่อสินค้าแต่ละชิ้นเลย — ข้อมูลถูกยุบหายไปหมด

**Window function** ทำงานต่างออกไปโดยสิ้นเชิง: มันคำนวณค่าทางสถิติ (เช่น ผลรวม ค่าเฉลี่ย อันดับ) โดย **"มอง" ไปยังกลุ่มแถวที่เกี่ยวข้อง (window) แต่ไม่ยุบแถวเดิมทิ้ง** ทุกแถวต้นฉบับยังคงอยู่ครบ เพียงแต่มีคอลัมน์เพิ่มเข้ามาที่บอกผลการคำนวณจาก window นั้น ๆ

```sql
-- Window function: แถวสินค้าแต่ละชิ้นยังอยู่ครบ พร้อมค่าเฉลี่ยของหมวดหมู่แนบมาด้วย
SELECT product_id, product_name, category_id, unit_price,
       COUNT(*) OVER (PARTITION BY category_id) AS product_count,
       ROUND(AVG(unit_price) OVER (PARTITION BY category_id), 2) AS avg_price_in_category
FROM products
WHERE is_active = true
ORDER BY category_id, product_id;
```

ผลลัพธ์ (แสดงบางส่วน):

| product_id | product_name | category_id | unit_price | product_count | avg_price_in_category |
|---|---|---|---|---|---|
| 1 | Laptop Pro 14 | 2 | 42900.00 | 4 | 19720.00 |
| 2 | Laptop Air 13 | 2 | 32900.00 | 4 | 19720.00 |
| 3 | Wireless Mouse | 2 | 590.00 | 4 | 19720.00 |
| 4 | Mechanical Keyboard | 2 | 2490.00 | 4 | 19720.00 |
| 5 | Smartphone X12 | 3 | 24900.00 | 4 | 11422.50 |
| 6 | Smartphone Lite | 3 | 9900.00 | 4 | 11422.50 |
| 7 | Bluetooth Earbuds | 3 | 1990.00 | 4 | 11422.50 |
| 8 | Smartwatch Series 5 | 3 | 8900.00 | 4 | 11422.50 |

ตัวเลข `avg_price_in_category = 19720.00` สำหรับหมวด Computers ตรงกับค่าที่ได้จาก `GROUP BY` ในตัวอย่างแรกทุกประการ — แต่ครั้งนี้เรายังเห็น `product_name` และ `unit_price` ของสินค้าแต่ละชิ้นได้ครบถ้วน นี่คือหัวใจของ window function: **คำนวณค่าสรุประดับกลุ่ม แต่คงรายละเอียดระดับแถวไว้** ทำให้สามารถนำค่าสรุปนั้นไปเปรียบเทียบกับแถวต้นฉบับได้ในบรรทัดเดียวกัน (เช่น "สินค้าชิ้นนี้แพงกว่าค่าเฉลี่ยของหมวดหมู่หรือไม่")

ข้อแตกต่างสำคัญอีกข้อคือ **ตำแหน่งที่เขียนได้**: aggregate function ธรรมดาต้องอยู่คู่กับ `GROUP BY` (หรืออย่างน้อยไม่มีคอลัมน์อื่นที่ไม่ได้ aggregate) แต่ window function เขียนต่อท้ายด้วยคีย์เวิร์ด `OVER (...)` และสามารถปรากฏใน `SELECT` list ได้โดยไม่ต้องมี `GROUP BY` เลย ทำให้เขียนคู่กับคอลัมน์อื่นที่ไม่ได้ aggregate ได้อย่างอิสระ

> **กฎสำคัญ:** window function จะถูกประมวลผล **หลังจาก** `WHERE`, `GROUP BY`, `HAVING` แต่ **ก่อน** `ORDER BY` และ `LIMIT` เสมอ (ตามลำดับการประมวลผลของ SQL) ดังนั้นถ้าต้องการกรองแถวก่อนคำนวณ window function ให้ใช้ `WHERE` ตามปกติ แต่ถ้าต้องการกรองผลลัพธ์ **หลังจาก** คำนวณ window function แล้ว (เช่น เอาเฉพาะแถวที่สูงกว่าค่าเฉลี่ย) จะต้องครอบด้วย subquery หรือ CTE ก่อน เพราะ `WHERE` ใช้ window function โดยตรงไม่ได้

---

## Step 282: OVER() พื้นฐาน — มองทั้งผลลัพธ์เป็น window เดียว

รูปแบบที่ง่ายที่สุดของ window function คือการเขียน `OVER()` แบบไม่มีอะไรอยู่ข้างในวงเล็บเลย ซึ่งหมายความว่า **ทุกแถวในผลลัพธ์ถือเป็น window เดียวกันทั้งหมด** ไม่มีการแบ่งกลุ่มใด ๆ

ลองมาดูตัวอย่างการเปรียบเทียบยอดขายของแต่ละ order กับค่าเฉลี่ยของยอดขายทุก order ที่ `completed`:

```sql
SELECT o.order_id,
       o.customer_id,
       o.order_date::date AS order_date,
       SUM(oi.quantity * oi.unit_price) AS order_total,
       ROUND(AVG(SUM(oi.quantity * oi.unit_price)) OVER (), 2) AS avg_all_orders
FROM orders o
JOIN order_items oi ON oi.order_id = o.order_id
WHERE o.status = 'completed'
GROUP BY o.order_id, o.customer_id, o.order_date
ORDER BY o.order_id;
```

ผลลัพธ์:

| order_id | customer_id | order_date | order_total | avg_all_orders |
|---|---|---|---|---|
| 1 | 1 | 2024-08-01 | 44080.00 | 17975.56 |
| 2 | 3 | 2024-08-01 | 26890.00 | 17975.56 |
| 3 | 5 | 2024-08-01 | 9900.00 | 17975.56 |
| 4 | 2 | 2024-08-02 | 35390.00 | 17975.56 |
| 5 | 7 | 2024-08-02 | 2470.00 | 17975.56 |
| 6 | 4 | 2024-08-03 | 15900.00 | 17975.56 |
| 7 | 9 | 2024-08-03 | 6170.00 | 17975.56 |
| 8 | 6 | 2024-08-04 | 8900.00 | 17975.56 |
| 9 | 1 | 2024-08-04 | 7060.00 | 17975.56 |
| 11 | 11 | 2024-08-05 | 9100.00 | 17975.56 |
| 12 | 10 | 2024-08-06 | 44670.00 | 17975.56 |
| 13 | 3 | 2024-08-06 | 24900.00 | 17975.56 |
| 14 | 13 | 2024-08-07 | 2980.00 | 17975.56 |
| 15 | 5 | 2024-08-07 | 11890.00 | 17975.56 |
| 16 | 12 | 2024-08-08 | 22900.00 | 17975.56 |
| 17 | 2 | 2024-08-08 | 32900.00 | 17975.56 |
| 18 | 14 | 2024-08-09 | 12880.00 | 17975.56 |
| 19 | 9 | 2024-08-09 | 4580.00 | 17975.56 |

สังเกตว่าคอลัมน์ `avg_all_orders` มีค่า **17975.56 เหมือนกันทุกแถว** เพราะ `OVER()` แบบไม่ใส่อะไรเลยจะมองทั้งผลลัพธ์ (หลัง `GROUP BY` แล้ว) เป็น window เดียว ค่าเฉลี่ยที่ได้จึงเป็นค่าเฉลี่ยของทั้งชุดข้อมูล 18 orders (44080+26890+...+4580)/18 = 17975.56

จุดที่ต้องสังเกตให้ดีคือการซ้อน `SUM()` สองชั้น: `AVG(SUM(oi.quantity * oi.unit_price)) OVER ()` — ชั้นในสุด `SUM(oi.quantity * oi.unit_price)` เป็น aggregate function ธรรมดาที่ทำงานร่วมกับ `GROUP BY o.order_id` เพื่อรวมยอดขายของแต่ละ order ก่อน จากนั้น `AVG(...) OVER ()` ที่ครอบอยู่ด้านนอกจึงเป็น window function ที่คำนวณค่าเฉลี่ยของผลลัพธ์ที่ถูก group แล้วอีกที นี่คือรูปแบบที่พบบ่อยมากเวลาต้องการเปรียบเทียบ "ยอดรวมต่อ order" กับ "ค่าเฉลี่ยของยอดรวมทั้งหมด" ในคิวรีเดียว

> **เทคนิค:** เมื่อ query มี `GROUP BY` อยู่แล้ว window function จะถูกคำนวณจากผลลัพธ์ **หลังจาก** group เสร็จแล้ว ไม่ใช่จากแถวดิบก่อน group ดังนั้น window function ที่ครอบ aggregate function อีกชั้นแบบนี้จึงเป็นวิธีมาตรฐานในการเปรียบเทียบค่าที่ถูกสรุปแล้วกับค่าสรุปภาพรวม

---

## Step 283: PARTITION BY — แบ่งข้อมูลเป็นกลุ่มย่อยก่อนคำนวณ

หาก `OVER()` มองทั้งผลลัพธ์เป็น window เดียว `PARTITION BY` ก็คือการบอกให้ PostgreSQL **แบ่งแถวออกเป็นกลุ่มย่อยตามค่าของคอลัมน์ที่ระบุก่อน แล้วค่อยคำนวณ window function แยกกันในแต่ละกลุ่ม** เหมือนกับ `GROUP BY` แต่ไม่ยุบแถว

ลองปรับตัวอย่างก่อนหน้าให้แบ่งกลุ่มตามลูกค้า เพื่อดูว่าลูกค้าแต่ละคนมียอดสั่งซื้อเฉลี่ยเท่าไร:

```sql
SELECT o.order_id,
       o.customer_id,
       o.order_date::date AS order_date,
       SUM(oi.quantity * oi.unit_price) AS order_total,
       ROUND(AVG(SUM(oi.quantity * oi.unit_price)) OVER (PARTITION BY o.customer_id), 2) AS avg_per_customer
FROM orders o
JOIN order_items oi ON oi.order_id = o.order_id
WHERE o.status = 'completed'
GROUP BY o.order_id, o.customer_id, o.order_date
ORDER BY o.customer_id, o.order_id;
```

ผลลัพธ์:

| order_id | customer_id | order_date | order_total | avg_per_customer |
|---|---|---|---|---|
| 1 | 1 | 2024-08-01 | 44080.00 | **25570.00** |
| 9 | 1 | 2024-08-04 | 7060.00 | **25570.00** |
| 4 | 2 | 2024-08-02 | 35390.00 | **34145.00** |
| 17 | 2 | 2024-08-08 | 32900.00 | **34145.00** |
| 2 | 3 | 2024-08-01 | 26890.00 | **25895.00** |
| 13 | 3 | 2024-08-06 | 24900.00 | **25895.00** |
| 6 | 4 | 2024-08-03 | 15900.00 | 15900.00 |
| 3 | 5 | 2024-08-01 | 9900.00 | **10895.00** |
| 15 | 5 | 2024-08-07 | 11890.00 | **10895.00** |
| 8 | 6 | 2024-08-04 | 8900.00 | 8900.00 |
| 5 | 7 | 2024-08-02 | 2470.00 | 2470.00 |
| 7 | 9 | 2024-08-03 | 6170.00 | **5375.00** |
| 19 | 9 | 2024-08-09 | 4580.00 | **5375.00** |
| 12 | 10 | 2024-08-06 | 44670.00 | 44670.00 |
| 11 | 11 | 2024-08-05 | 9100.00 | 9100.00 |
| 16 | 12 | 2024-08-08 | 22900.00 | 22900.00 |
| 14 | 13 | 2024-08-07 | 2980.00 | 2980.00 |
| 18 | 14 | 2024-08-09 | 12880.00 | 12880.00 |

สังเกตขอบเขตของแต่ละ partition ให้ชัดเจน: ลูกค้า `customer_id = 1` มี 2 orders (order 1 และ 9) และทั้งสองแถวมี `avg_per_customer = 25570.00` เท่ากัน (เฉลี่ยของ 44080 และ 7060) ส่วนลูกค้า `customer_id = 4, 6, 7, 10, 11, 12, 13, 14` ที่มีเพียง order เดียว ค่าเฉลี่ยจึงเท่ากับ `order_total` ของตัวเองพอดี เพราะ partition ของเขามีสมาชิกแค่แถวเดียว

**เปรียบเทียบกับ `GROUP BY customer_id`:** ถ้าใช้ `GROUP BY` แบบเดิมเราจะได้ผลลัพธ์แค่ 14 แถว (1 แถวต่อลูกค้า) และไม่เห็นว่า order แต่ละใบมียอดเท่าไร แต่ด้วย `PARTITION BY` เราได้ผลลัพธ์ครบ 18 แถวเหมือนเดิม พร้อมค่าเฉลี่ยของกลุ่มแนบมาในทุกแถว ทำให้สามารถนำ `order_total` ไปเทียบกับ `avg_per_customer` ในบรรทัดเดียวกันได้ทันที (จะทำใน Step 285)

Syntax พื้นฐานของ window function คือ:

```sql
function_name(argument) OVER (
    [PARTITION BY column1, column2, ...]
    [ORDER BY column3, column4, ...]
    [frame_clause]
)
```

ทุกส่วนใน `OVER(...)` เป็นทางเลือก (optional) นอกจาก `OVER()` เอง ซึ่งขาดไม่ได้ — และสามารถ `PARTITION BY` ได้มากกว่า 1 คอลัมน์พร้อมกัน เช่น `PARTITION BY category_id, supplier_id` เพื่อแบ่งกลุ่มละเอียดขึ้น

---

## Step 284: Aggregate function ในรูปแบบ window function — SUM(), AVG(), COUNT()

Aggregate function ที่เราคุ้นเคย เช่น `SUM()`, `AVG()`, `COUNT()`, `MIN()`, `MAX()` ทุกตัวสามารถนำมาใช้เป็น window function ได้ทันทีเพียงแค่เติม `OVER (...)` ต่อท้าย โดยไม่จำเป็นต้องมี `GROUP BY` เลยด้วยซ้ำ

ลองดูตัวอย่างที่ใช้ทั้งสามตัวพร้อมกันบนตาราง `products` โดย partition ตาม `category_id`:

```sql
SELECT product_id,
       product_name,
       category_id,
       unit_price,
       COUNT(*) OVER (PARTITION BY category_id) AS products_in_category,
       ROUND(AVG(unit_price) OVER (PARTITION BY category_id), 2) AS avg_price_in_category,
       SUM(unit_price) OVER (PARTITION BY category_id) AS sum_price_in_category
FROM products
WHERE is_active = true
ORDER BY category_id, product_id;
```

ผลลัพธ์:

| product_id | product_name | category_id | unit_price | products_in_category | avg_price_in_category | sum_price_in_category |
|---|---|---|---|---|---|---|
| 1 | Laptop Pro 14 | 2 | 42900.00 | 4 | 19720.00 | 78880.00 |
| 2 | Laptop Air 13 | 2 | 32900.00 | 4 | 19720.00 | 78880.00 |
| 3 | Wireless Mouse | 2 | 590.00 | 4 | 19720.00 | 78880.00 |
| 4 | Mechanical Keyboard | 2 | 2490.00 | 4 | 19720.00 | 78880.00 |
| 5 | Smartphone X12 | 3 | 24900.00 | 4 | 11422.50 | 45690.00 |
| 6 | Smartphone Lite | 3 | 9900.00 | 4 | 11422.50 | 45690.00 |
| 7 | Bluetooth Earbuds | 3 | 1990.00 | 4 | 11422.50 | 45690.00 |
| 8 | Smartwatch Series 5 | 3 | 8900.00 | 4 | 11422.50 | 45690.00 |
| 13 | Rice Cooker Deluxe | 4 | 1590.00 | 2 | 2290.00 | 4580.00 |
| 14 | Air Fryer XL | 4 | 2990.00 | 2 | 2290.00 | 4580.00 |
| 9 | Dining Table Oak | 5 | 15900.00 | 4 | 11975.00 | 47900.00 |
| 10 | Office Chair Ergo | 5 | 5900.00 | 4 | 11975.00 | 47900.00 |
| 11 | Bookshelf 5-Tier | 5 | 3200.00 | 4 | 11975.00 | 47900.00 |
| 12 | Sofa 3-Seater | 5 | 22900.00 | 4 | 11975.00 | 47900.00 |
| 17 | Mystery of the Lost City | 7 | 350.00 | 1 | 350.00 | 350.00 |
| 15 | The Art of Programming | 8 | 590.00 | 2 | 520.00 | 1040.00 |
| 16 | Thai Cuisine Cookbook | 8 | 450.00 | 2 | 520.00 | 1040.00 |

ข้อสังเกตสำคัญ:

- `COUNT(*) OVER (PARTITION BY category_id)` นับจำนวนสินค้าในหมวดหมู่เดียวกันทั้งหมด แม้ว่าสินค้าแต่ละชิ้นจะเป็นคนละแถวก็ตาม — หมวด `category_id = 2` (Computers) มี 4 ชิ้น ทุกแถวในหมวดนี้จึงได้ `products_in_category = 4` เหมือนกัน
- `category_id = 7` (Fiction) มีสินค้า active เพียงชิ้นเดียวคือ `Mystery of the Lost City` เพราะ `Whispers in the Dark` (product_id 18) ถูกตั้งเป็น `is_active = false` และถูกกรองออกด้วย `WHERE` ไปแล้วตั้งแต่ก่อนคำนวณ window function — ดังนั้น `products_in_category = 1` และ `avg_price_in_category = sum_price_in_category` เพราะมีสมาชิกแค่ตัวเดียว
- `category_id = 6` (Books, หมวดแม่) ไม่ปรากฏในผลลัพธ์เลย เพราะไม่มีสินค้าใดถูก assign `category_id = 6` โดยตรง (สินค้าทั้งหมดถูก assign ไปที่หมวดย่อย `7` และ `8` แทน)

Window function เหล่านี้ยังสามารถใช้กับ `MIN()` และ `MAX()` ได้ในลักษณะเดียวกัน เช่น การหาสินค้าที่แพงที่สุดในแต่ละหมวดหมู่ (`MAX(unit_price) OVER (PARTITION BY category_id)`) ซึ่งจะเป็นพื้นฐานสำคัญสำหรับการทำ ranking ใน Part ถัดไป

---

## Step 285: ตัวอย่างจริง — เปรียบเทียบยอดขายแต่ละ order กับค่าเฉลี่ยของลูกค้าคนนั้น

ตอนนี้เรามาผสาน Step 282-284 เข้าด้วยกันเพื่อแก้ปัญหาทางธุรกิจจริง: **"order นี้มียอดสูงหรือต่ำกว่าค่าเฉลี่ยของลูกค้าคนนี้เท่าไร"** ซึ่งเป็นคำถามที่ทีมการตลาดมักถามเพื่อระบุ "ลูกค้าที่กำลังซื้อน้อยกว่าปกติ" (churn risk) หรือ "ลูกค้าที่กำลังซื้อมากกว่าปกติ" (upsell opportunity)

```sql
SELECT o.order_id,
       o.customer_id,
       o.order_date::date AS order_date,
       SUM(oi.quantity * oi.unit_price) AS order_total,
       ROUND(AVG(SUM(oi.quantity * oi.unit_price)) OVER (PARTITION BY o.customer_id), 2) AS customer_avg,
       ROUND(
           SUM(oi.quantity * oi.unit_price)
           - AVG(SUM(oi.quantity * oi.unit_price)) OVER (PARTITION BY o.customer_id),
       2) AS diff_from_avg
FROM orders o
JOIN order_items oi ON oi.order_id = o.order_id
WHERE o.status = 'completed'
GROUP BY o.order_id, o.customer_id, o.order_date
ORDER BY o.customer_id, o.order_id;
```

ผลลัพธ์:

| order_id | customer_id | order_date | order_total | customer_avg | diff_from_avg |
|---|---|---|---|---|---|
| 1 | 1 | 2024-08-01 | 44080.00 | 25570.00 | **+18510.00** |
| 9 | 1 | 2024-08-04 | 7060.00 | 25570.00 | **-18510.00** |
| 4 | 2 | 2024-08-02 | 35390.00 | 34145.00 | +1245.00 |
| 17 | 2 | 2024-08-08 | 32900.00 | 34145.00 | -1245.00 |
| 2 | 3 | 2024-08-01 | 26890.00 | 25895.00 | +995.00 |
| 13 | 3 | 2024-08-06 | 24900.00 | 25895.00 | -995.00 |
| 6 | 4 | 2024-08-03 | 15900.00 | 15900.00 | 0.00 |
| 3 | 5 | 2024-08-01 | 9900.00 | 10895.00 | -995.00 |
| 15 | 5 | 2024-08-07 | 11890.00 | 10895.00 | +995.00 |
| 8 | 6 | 2024-08-04 | 8900.00 | 8900.00 | 0.00 |
| 5 | 7 | 2024-08-02 | 2470.00 | 2470.00 | 0.00 |
| 7 | 9 | 2024-08-03 | 6170.00 | 5375.00 | +795.00 |
| 19 | 9 | 2024-08-09 | 4580.00 | 5375.00 | -795.00 |
| 12 | 10 | 2024-08-06 | 44670.00 | 44670.00 | 0.00 |
| 11 | 11 | 2024-08-05 | 9100.00 | 9100.00 | 0.00 |
| 16 | 12 | 2024-08-08 | 22900.00 | 22900.00 | 0.00 |
| 14 | 13 | 2024-08-07 | 2980.00 | 2980.00 | 0.00 |
| 18 | 14 | 2024-08-09 | 12880.00 | 12880.00 | 0.00 |

ตัวอย่างที่เห็นชัดเจนที่สุดคือลูกค้า `customer_id = 1` (Somchai Jaidee): order แรก (`order_id = 1`) มียอด 44,080 บาท ซึ่งสูงกว่าค่าเฉลี่ยของตัวเองถึง **+18,510 บาท** ในขณะที่ order ที่สอง (`order_id = 9`) มียอดเพียง 7,060 บาท ต่ำกว่าค่าเฉลี่ยถึง **-18,510 บาท** (ตัวเลขบวกและลบรวมกันเท่ากับ 0 เสมอ เพราะเป็นการเทียบกับค่าเฉลี่ยของกลุ่มตัวเอง) ข้อมูลแบบนี้ช่วยให้ทีมขายเห็นภาพได้ทันทีว่าพฤติกรรมการซื้อของลูกค้าแต่ละคนผันผวนแค่ไหน โดยไม่ต้องเขียนคิวรีแยกสองรอบแล้วเอามา join กันเอง

> **เทียบกับวิธีเขียนแบบเก่า:** ถ้าไม่มี window function เราจะต้องเขียน subquery แยกต่างหากเพื่อหาค่าเฉลี่ยต่อลูกค้าก่อน แล้วค่อย `JOIN` กลับเข้ากับตาราง orders อีกที ซึ่งอ่านยากกว่าและมักจะ join ผิดคอลัมน์ได้ง่าย ในขณะที่ window function ทำทุกอย่างในคิวรีเดียว อ่านง่ายกว่า และ PostgreSQL optimizer มักจะทำงานได้เร็วกว่าด้วย

---

## Step 286: ORDER BY ภายใน OVER() — Running Total และ Running Average

เมื่อเราเพิ่ม `ORDER BY` เข้าไปภายใน `OVER(...)` พฤติกรรมของ window function จะเปลี่ยนไปอย่างมีนัยสำคัญ: แทนที่จะคำนวณจาก **สมาชิกทุกตัวในกลุ่ม** มันจะคำนวณจาก **สมาชิกตั้งแต่แถวแรกของกลุ่มจนถึงแถวปัจจุบัน** (ตามลำดับที่ `ORDER BY` กำหนด) เท่านั้น พฤติกรรมนี้เรียกว่า **default frame** ซึ่งเราจะอธิบายรายละเอียดใน Step 287 แต่ตอนนี้ให้เข้าใจผลลัพธ์ก่อน: นี่คือกลไกเบื้องหลังของ **running total** (ยอดสะสม) และ **running average** (ค่าเฉลี่ยสะสม)

มาสร้างรายงานยอดขายสะสมรายวันกัน โดยเริ่มจากการรวมยอดขายต่อวันด้วย CTE ก่อน แล้วค่อยคำนวณยอดสะสม:

```sql
WITH daily AS (
    SELECT o.order_date::date AS order_day,
           SUM(oi.quantity * oi.unit_price) AS daily_total
    FROM orders o
    JOIN order_items oi ON oi.order_id = o.order_id
    WHERE o.status = 'completed'
    GROUP BY o.order_date::date
)
SELECT order_day,
       daily_total,
       SUM(daily_total) OVER (ORDER BY order_day) AS running_total,
       ROUND(AVG(daily_total) OVER (ORDER BY order_day), 2) AS running_avg
FROM daily
ORDER BY order_day;
```

ผลลัพธ์:

| order_day | daily_total | running_total | running_avg |
|---|---|---|---|
| 2024-08-01 | 80870.00 | 80870.00 | 80870.00 |
| 2024-08-02 | 37860.00 | 118730.00 | 59365.00 |
| 2024-08-03 | 22070.00 | 140800.00 | 46933.33 |
| 2024-08-04 | 15960.00 | 156760.00 | 39190.00 |
| 2024-08-05 | 9100.00 | 165860.00 | 33172.00 |
| 2024-08-06 | 69570.00 | 235430.00 | 39238.33 |
| 2024-08-07 | 14870.00 | 250300.00 | 35757.14 |
| 2024-08-08 | 55800.00 | 306100.00 | 38262.50 |
| 2024-08-09 | 17460.00 | 323560.00 | 35951.11 |

ตรวจสอบตรรกะทีละแถว:

- แถวที่ 1 (`2024-08-01`): ยังไม่มีวันก่อนหน้า `running_total = daily_total = 80870.00`
- แถวที่ 2 (`2024-08-02`): `running_total = 80870.00 + 37860.00 = 118730.00` — เป็นผลรวมของวันที่ 1 และ 2
- แถวที่ 3 (`2024-08-03`): `running_total = 118730.00 + 22070.00 = 140800.00` — เป็นผลรวมของวันที่ 1, 2, 3
- แถวสุดท้าย (`2024-08-09`): `running_total = 323560.00` ซึ่งเท่ากับผลรวมของยอดขายทุกวันรวมกัน

ในทำนองเดียวกัน `running_avg` ก็คือค่าเฉลี่ยของ `daily_total` ตั้งแต่วันแรกจนถึงวันปัจจุบัน ไม่ใช่ค่าเฉลี่ยของทั้งหมด — จะสังเกตว่า `running_avg` ของแถวสุดท้าย (35951.11) แตกต่างจากค่าเฉลี่ยของ `OVER()` แบบไม่มี `ORDER BY` (ซึ่งจะเท่ากับ 323560.00/9 = 35951.11 พอดี เพราะเป็นแถวสุดท้ายที่ครอบคลุมสมาชิกครบทุกตัวแล้ว)

> **กฎสำคัญที่ต้องจำ:** เมื่อใส่ `ORDER BY` ภายใน `OVER(...)` โดยไม่ระบุ frame clause เพิ่มเติม PostgreSQL จะใช้ default frame เป็น `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` โดยอัตโนมัติ ซึ่งหมายถึง "ตั้งแต่แถวแรกสุดของ partition จนถึงแถวปัจจุบัน" นี่คือเหตุผลที่ `SUM() OVER (ORDER BY ...)` กลายเป็น running total โดยอัตโนมัติทันทีที่มี `ORDER BY`

การใช้ `PARTITION BY` ร่วมกับ `ORDER BY` ก็ทำได้เช่นกัน เช่น `SUM(order_total) OVER (PARTITION BY customer_id ORDER BY order_date)` จะให้ยอดสะสมของแต่ละลูกค้าแยกกัน (running total ต่อคน) ซึ่งเป็นรูปแบบที่ใช้บ่อยมากในรายงานการเงิน

---

## Step 287: Frame Clause เบื้องต้น — ROWS BETWEEN, RANGE BETWEEN, UNBOUNDED

ใน Step 286 เราเห็นว่าการมี `ORDER BY` ทำให้เกิด default frame แบบอัตโนมัติ แต่ในความเป็นจริง เราสามารถ **กำหนด frame เองอย่างชัดเจน** ได้ ซึ่งจะให้ความยืดหยุ่นมากกว่ามาก เพื่อสร้างรายงานแบบ moving average, running total แบบย้อนกลับ หรือ "ผลรวมของแถวข้างเคียง"

**Syntax ของ frame clause:**

```sql
{ROWS | RANGE} BETWEEN frame_start AND frame_end
```

โดย `frame_start` และ `frame_end` เลือกได้จาก:

- `UNBOUNDED PRECEDING` — ตั้งแต่แถวแรกสุดของ partition
- `N PRECEDING` — ย้อนหลังไป N แถว (หรือ N หน่วยตาม `ORDER BY` ถ้าใช้ `RANGE`)
- `CURRENT ROW` — แถวปัจจุบัน
- `N FOLLOWING` — ไปข้างหน้า N แถว
- `UNBOUNDED FOLLOWING` — ไปจนถึงแถวสุดท้ายของ partition

**ความแตกต่างระหว่าง `ROWS` กับ `RANGE`:**

- `ROWS BETWEEN ...` นับ **จำนวนแถวจริง** ไม่สนใจว่าค่าของคอลัมน์ `ORDER BY` จะซ้ำกันหรือไม่
- `RANGE BETWEEN ...` นับ **ตามค่าของคอลัมน์ `ORDER BY`** — ถ้าหลายแถวมีค่าเท่ากัน (ties) จะถูกรวมเข้ามาในกรอบเดียวกันทั้งหมด แม้จะทำให้กรอบมีจำนวนแถวมากกว่าที่คาดไว้

มาดูตัวอย่างเปรียบเทียบทั้งสามแบบบนข้อมูลยอดขายรายวันเดิม:

```sql
WITH daily AS (
    SELECT o.order_date::date AS order_day,
           SUM(oi.quantity * oi.unit_price) AS daily_total
    FROM orders o
    JOIN order_items oi ON oi.order_id = o.order_id
    WHERE o.status = 'completed'
    GROUP BY o.order_date::date
)
SELECT order_day,
       daily_total,
       SUM(daily_total) OVER (
           ORDER BY order_day
           ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
       ) AS running_total_rows,
       SUM(daily_total) OVER (
           ORDER BY order_day
           RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
       ) AS running_total_range,
       SUM(daily_total) OVER (
           ORDER BY order_day
           ROWS BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING
       ) AS remaining_total
FROM daily
ORDER BY order_day;
```

ผลลัพธ์:

| order_day | daily_total | running_total_rows | running_total_range | remaining_total |
|---|---|---|---|---|
| 2024-08-01 | 80870.00 | 80870.00 | 80870.00 | 323560.00 |
| 2024-08-02 | 37860.00 | 118730.00 | 118730.00 | 242690.00 |
| 2024-08-03 | 22070.00 | 140800.00 | 140800.00 | 204830.00 |
| 2024-08-04 | 15960.00 | 156760.00 | 156760.00 | 182760.00 |
| 2024-08-05 | 9100.00 | 165860.00 | 165860.00 | 166800.00 |
| 2024-08-06 | 69570.00 | 235430.00 | 235430.00 | 157700.00 |
| 2024-08-07 | 14870.00 | 250300.00 | 250300.00 | 88130.00 |
| 2024-08-08 | 55800.00 | 306100.00 | 306100.00 | 73260.00 |
| 2024-08-09 | 17460.00 | 323560.00 | 323560.00 | 17460.00 |

สังเกตว่า `running_total_rows` และ `running_total_range` **ให้ผลลัพธ์เหมือนกันทุกแถว** ในตัวอย่างนี้ เพราะ `order_day` (จากมุมมองของ CTE `daily` ที่ group ไว้แล้ว) ไม่มีค่าซ้ำกันเลย แต่ละวันปรากฏเพียงครั้งเดียว — ถ้าข้อมูลมี `ORDER BY` ที่มีค่าซ้ำกัน (เช่นเรียงตาม order_date แบบไม่ group ก่อน ซึ่งหลายรายการอาจเกิดวันเดียวกัน) ผลลัพธ์ของ `ROWS` และ `RANGE` จะเริ่มต่างกัน เพราะ `RANGE` จะรวมทุกแถวที่มีค่าเท่ากันเข้าไว้ในกรอบเดียวกันเสมอ

คอลัมน์ `remaining_total` แสดงอีกแนวคิดหนึ่ง: frame `ROWS BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING` หมายถึง "ผลรวมตั้งแต่แถวปัจจุบันไปจนถึงแถวสุดท้าย" ซึ่งเป็น **reverse running total** — แถวแรก (`2024-08-01`) ได้ผลรวมของทั้งหมด (323560.00) เพราะนับจากตัวเองไปจนจบ ส่วนแถวสุดท้าย (`2024-08-09`) ได้แค่ค่าของตัวเอง (17460.00) เพราะไม่มีแถวถัดไปให้นับรวมแล้ว มีประโยชน์มากเวลาต้องการดูว่า "เหลือยอดขายอีกเท่าไรกว่าจะครบทั้งเดือน"

> **ข้อควรระวัง:** `RANGE` แบบพื้นฐาน (ไม่ระบุ offset ตัวเลข เช่น `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`) ใช้ได้กับข้อมูลทุกประเภท แต่ `RANGE BETWEEN N PRECEDING AND ...` ที่มี offset เป็นตัวเลขนั้น ต้องการให้คอลัมน์ที่ `ORDER BY` เป็นชนิดตัวเลขหรือวันที่/เวลาเท่านั้น (เพราะต้อง "บวกลบ" ค่าได้) ในขณะที่ `ROWS BETWEEN N PRECEDING AND ...` ใช้ได้กับข้อมูลทุกประเภทเสมอ เพราะนับจากจำนวนแถวไม่ใช่ค่าตัวเลข ด้วยเหตุนี้ในทางปฏิบัติจึงนิยมใช้ `ROWS` มากกว่า `RANGE` เวลาต้องการทำ moving average ตามจำนวนแถวที่แน่นอน

---

## Step 288: Moving Average และ Running Total ด้วย Frame ที่กำหนดเอง

เมื่อเข้าใจ frame clause แล้ว เราสามารถสร้าง **moving average** (ค่าเฉลี่ยเคลื่อนที่) ได้อย่างแม่นยำ ซึ่งมีประโยชน์มากในการดูแนวโน้มยอดขายโดยลด "สัญญาณรบกวน" จากความผันผวนรายวัน

ลองสร้าง moving average แบบ 3 วัน (รวมวันปัจจุบันและ 2 วันก่อนหน้า) และ centered sum แบบ 3 วัน (รวมวันก่อนหน้า 1 วัน วันปัจจุบัน และวันถัดไป 1 วัน):

```sql
WITH daily AS (
    SELECT o.order_date::date AS order_day,
           SUM(oi.quantity * oi.unit_price) AS daily_total
    FROM orders o
    JOIN order_items oi ON oi.order_id = o.order_id
    WHERE o.status = 'completed'
    GROUP BY o.order_date::date
)
SELECT order_day,
       daily_total,
       ROUND(AVG(daily_total) OVER (
           ORDER BY order_day
           ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
       ), 2) AS moving_avg_3day,
       SUM(daily_total) OVER (
           ORDER BY order_day
           ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING
       ) AS centered_sum_3day
FROM daily
ORDER BY order_day;
```

ผลลัพธ์:

| order_day | daily_total | moving_avg_3day | centered_sum_3day |
|---|---|---|---|
| 2024-08-01 | 80870.00 | 80870.00 | 118730.00 |
| 2024-08-02 | 37860.00 | 59365.00 | 140800.00 |
| 2024-08-03 | 22070.00 | 46933.33 | 75890.00 |
| 2024-08-04 | 15960.00 | 25296.67 | 47130.00 |
| 2024-08-05 | 9100.00 | 15710.00 | 94630.00 |
| 2024-08-06 | 69570.00 | 31543.33 | 93540.00 |
| 2024-08-07 | 14870.00 | 31180.00 | 140240.00 |
| 2024-08-08 | 55800.00 | 46746.67 | 88130.00 |
| 2024-08-09 | 17460.00 | 29376.67 | 73260.00 |

**อธิบาย `moving_avg_3day` (`ROWS BETWEEN 2 PRECEDING AND CURRENT ROW`):**

- แถวที่ 1 (`08-01`): ไม่มีแถวก่อนหน้าให้ย้อนไป 2 แถว PostgreSQL จะใช้เท่าที่มี (แค่ตัวเอง) → `moving_avg_3day = 80870.00`
- แถวที่ 2 (`08-02`): ย้อนได้แค่ 1 แถว (มีแค่ 08-01 กับ 08-02) → เฉลี่ยของ 80870 และ 37860 = `59365.00`
- แถวที่ 3 (`08-03`): ตอนนี้มีครบ 3 แถวแล้ว (08-01, 08-02, 08-03) → เฉลี่ยของ 80870+37860+22070 = 140800/3 = `46933.33`
- แถวที่ 4 (`08-04`): กรอบเลื่อนไปเป็น (08-02, 08-03, 08-04) → เฉลี่ยของ 37860+22070+15960 = 75890/3 = `25296.67`

จะเห็นว่ากรอบ "เลื่อน" ไปทีละแถวตามลำดับ `ORDER BY` นี่คือที่มาของชื่อ "moving average" — ค่าเฉลี่ยจะถูกคำนวณใหม่จากสมาชิก 3 ตัวล่าสุดเสมอ (หรือน้อยกว่านั้นถ้ายังไม่มีข้อมูลครบ 3 ตัวในช่วงเริ่มต้น)

**อธิบาย `centered_sum_3day` (`ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING`):**

frame นี้ดึงข้อมูล "1 แถวก่อนหน้า + แถวปัจจุบัน + 1 แถวถัดไป" ทำให้แถวปัจจุบันอยู่ตรงกลางของกรอบพอดี (centered window) เช่น แถว `08-03` ได้ผลรวมของ `08-02 + 08-03 + 08-04` = 37860+22070+15960 = `75890.00` ส่วนแถวแรก (`08-01`) ไม่มีแถวก่อนหน้า จึงได้แค่ `08-01 + 08-02` = 80870+37860 = `118730.00`

> **การเลือกใช้งานจริง:** moving average แบบ `N PRECEDING AND CURRENT ROW` เหมาะกับการดูแนวโน้มแบบ "ข้อมูลล่าสุด" (เช่น dashboard เรียลไทม์) เพราะไม่ต้องรอข้อมูลอนาคต ในขณะที่ centered window (`N PRECEDING AND N FOLLOWING`) เหมาะกับการวิเคราะห์ข้อมูลย้อนหลังทั้งหมดแล้ว (เช่น รายงานสรุปประจำเดือน) เพราะให้ภาพที่ smooth และสมมาตรกว่า แต่ใช้กับข้อมูลเรียลไทม์ไม่ได้เพราะยังไม่รู้ค่าของอนาคต

---

## Step 289: รวม Window Function หลายตัวในคิวรีเดียว และ WINDOW Clause

ในการทำรายงานจริง เรามักต้องการ window function มากกว่าหนึ่งตัวในคิวรีเดียวกัน และหลายครั้งก็ใช้ `PARTITION BY`/`ORDER BY` แบบเดียวกันซ้ำ ๆ กัน ซึ่งถ้าเขียน `OVER (PARTITION BY ... ORDER BY ...)` ซ้ำทุกครั้งจะทำให้คิวรียาวและแก้ไขยาก — PostgreSQL จึงมี **`WINDOW` clause** ให้ตั้งชื่อ window definition ไว้ล่วงหน้า แล้วเรียกใช้ซ้ำได้

ตัวอย่างที่ไม่ใช้ `WINDOW` clause (เขียน `PARTITION BY o.customer_id` ซ้ำ 3 รอบ):

```sql
SELECT o.order_id,
       o.customer_id,
       SUM(oi.quantity * oi.unit_price) AS order_total,
       ROUND(AVG(SUM(oi.quantity * oi.unit_price)) OVER (PARTITION BY o.customer_id), 2) AS customer_avg,
       COUNT(*) OVER (PARTITION BY o.customer_id) AS customer_order_count,
       SUM(SUM(oi.quantity * oi.unit_price)) OVER (PARTITION BY o.customer_id) AS customer_total
FROM orders o
JOIN order_items oi ON oi.order_id = o.order_id
WHERE o.status = 'completed'
GROUP BY o.order_id, o.customer_id
ORDER BY o.customer_id, o.order_id;
```

ตัวอย่างเดียวกันแต่เขียนให้กระชับขึ้นด้วย `WINDOW` clause:

```sql
SELECT o.order_id,
       o.customer_id,
       o.order_date::date AS order_date,
       SUM(oi.quantity * oi.unit_price) AS order_total,
       ROUND(AVG(SUM(oi.quantity * oi.unit_price)) OVER w, 2) AS customer_avg,
       COUNT(*) OVER w AS customer_order_count,
       SUM(SUM(oi.quantity * oi.unit_price)) OVER w AS customer_total
FROM orders o
JOIN order_items oi ON oi.order_id = o.order_id
WHERE o.status = 'completed'
GROUP BY o.order_id, o.customer_id, o.order_date
WINDOW w AS (PARTITION BY o.customer_id)
ORDER BY o.customer_id, o.order_id;
```

ทั้งสองคิวรีให้ผลลัพธ์เดียวกัน:

| order_id | customer_id | order_date | order_total | customer_avg | customer_order_count | customer_total |
|---|---|---|---|---|---|---|
| 1 | 1 | 2024-08-01 | 44080.00 | 25570.00 | 2 | 51140.00 |
| 9 | 1 | 2024-08-04 | 7060.00 | 25570.00 | 2 | 51140.00 |
| 4 | 2 | 2024-08-02 | 35390.00 | 34145.00 | 2 | 68290.00 |
| 17 | 2 | 2024-08-08 | 32900.00 | 34145.00 | 2 | 68290.00 |
| 2 | 3 | 2024-08-01 | 26890.00 | 25895.00 | 2 | 51790.00 |
| 13 | 3 | 2024-08-06 | 24900.00 | 25895.00 | 2 | 51790.00 |
| 6 | 4 | 2024-08-03 | 15900.00 | 15900.00 | 1 | 15900.00 |
| 3 | 5 | 2024-08-01 | 9900.00 | 10895.00 | 2 | 21790.00 |
| 15 | 5 | 2024-08-07 | 11890.00 | 10895.00 | 2 | 21790.00 |
| 8 | 6 | 2024-08-04 | 8900.00 | 8900.00 | 1 | 8900.00 |
| 5 | 7 | 2024-08-02 | 2470.00 | 2470.00 | 1 | 2470.00 |
| 7 | 9 | 2024-08-03 | 6170.00 | 5375.00 | 2 | 10750.00 |
| 19 | 9 | 2024-08-09 | 4580.00 | 5375.00 | 2 | 10750.00 |
| 12 | 10 | 2024-08-06 | 44670.00 | 44670.00 | 1 | 44670.00 |
| 11 | 11 | 2024-08-05 | 9100.00 | 9100.00 | 1 | 9100.00 |
| 16 | 12 | 2024-08-08 | 22900.00 | 22900.00 | 1 | 22900.00 |
| 14 | 13 | 2024-08-07 | 2980.00 | 2980.00 | 1 | 2980.00 |
| 18 | 14 | 2024-08-09 | 12880.00 | 12880.00 | 1 | 12880.00 |

`WINDOW w AS (PARTITION BY o.customer_id)` ประกาศชื่อ window ว่า `w` ไว้ครั้งเดียว จากนั้นเรียกใช้ในทุกฟังก์ชันด้วย `OVER w` แทนที่จะต้องเขียน `OVER (PARTITION BY o.customer_id)` ซ้ำ 3 รอบ — นอกจากจะทำให้โค้ดอ่านง่ายขึ้นแล้ว ยังลดโอกาสพิมพ์ผิดเวลาต้องแก้ partition หรือ order ทีหลัง เพราะแก้ที่เดียวจบ

`WINDOW` clause ยังรองรับการ "ต่อยอด" window ที่ประกาศไว้ได้ด้วย เช่น:

```sql
WINDOW
    w AS (PARTITION BY o.customer_id),
    w_ordered AS (w ORDER BY o.order_date)
```

ในกรณีนี้ `w_ordered` จะสืบทอด `PARTITION BY o.customer_id` มาจาก `w` แล้วเพิ่ม `ORDER BY o.order_date` เข้าไปอีกชั้นหนึ่ง เป็นประโยชน์มากเมื่อต้องใช้ทั้ง window แบบไม่มี `ORDER BY` (สำหรับค่าสรุปรวม) และแบบมี `ORDER BY` (สำหรับ running total) ในคิวรีเดียวกัน โดยไม่ต้องเขียน `PARTITION BY` ซ้ำสองที่

---

## Step 290: แบบฝึกหัดรวม — Running Total รายวัน และเปรียบเทียบสินค้ากับค่าเฉลี่ยหมวดหมู่

มาถึงจุดสรุปของบทนี้ เราจะประกอบทุกเทคนิคที่เรียนมาเข้าด้วยกันเป็นสองรายงานที่ใช้งานได้จริงในธุรกิจอีคอมเมิร์ซ

### รายงานที่ 1: Running Total ยอดขายรายวัน

```sql
WITH daily AS (
    SELECT o.order_date::date AS order_day,
           SUM(oi.quantity * oi.unit_price) AS daily_total
    FROM orders o
    JOIN order_items oi ON oi.order_id = o.order_id
    WHERE o.status = 'completed'
    GROUP BY o.order_date::date
)
SELECT order_day,
       daily_total,
       SUM(daily_total) OVER (ORDER BY order_day) AS running_total
FROM daily
ORDER BY order_day;
```

ผลลัพธ์:

| order_day | daily_total | running_total |
|---|---|---|
| 2024-08-01 | 80870.00 | 80870.00 |
| 2024-08-02 | 37860.00 | 118730.00 |
| 2024-08-03 | 22070.00 | 140800.00 |
| 2024-08-04 | 15960.00 | 156760.00 |
| 2024-08-05 | 9100.00 | 165860.00 |
| 2024-08-06 | 69570.00 | 235430.00 |
| 2024-08-07 | 14870.00 | 250300.00 |
| 2024-08-08 | 55800.00 | 306100.00 |
| 2024-08-09 | 17460.00 | 323560.00 |

รายงานนี้ตอบคำถามว่า "ถึงวันนี้ ร้านทำยอดขายสะสมไปเท่าไรแล้ว" ซึ่งเป็นรายงานพื้นฐานที่ dashboard ธุรกิจแทบทุกที่ต้องมี

### รายงานที่ 2: เปรียบเทียบราคาสินค้าแต่ละชิ้นกับค่าเฉลี่ยของหมวดหมู่

```sql
SELECT product_id,
       product_name,
       category_id,
       unit_price,
       ROUND(AVG(unit_price) OVER (PARTITION BY category_id), 2) AS category_avg_price,
       ROUND(unit_price - AVG(unit_price) OVER (PARTITION BY category_id), 2) AS diff_from_category_avg
FROM products
WHERE is_active = true
ORDER BY category_id, unit_price DESC;
```

ผลลัพธ์:

| product_id | product_name | category_id | unit_price | category_avg_price | diff_from_category_avg |
|---|---|---|---|---|---|
| 1 | Laptop Pro 14 | 2 | 42900.00 | 19720.00 | **+23180.00** |
| 2 | Laptop Air 13 | 2 | 32900.00 | 19720.00 | +13180.00 |
| 4 | Mechanical Keyboard | 2 | 2490.00 | 19720.00 | -17230.00 |
| 3 | Wireless Mouse | 2 | 590.00 | 19720.00 | **-19130.00** |
| 5 | Smartphone X12 | 3 | 24900.00 | 11422.50 | +13477.50 |
| 6 | Smartphone Lite | 3 | 9900.00 | 11422.50 | -1522.50 |
| 8 | Smartwatch Series 5 | 3 | 8900.00 | 11422.50 | -2522.50 |
| 7 | Bluetooth Earbuds | 3 | 1990.00 | 11422.50 | -9432.50 |
| 14 | Air Fryer XL | 4 | 2990.00 | 2290.00 | +700.00 |
| 13 | Rice Cooker Deluxe | 4 | 1590.00 | 2290.00 | -700.00 |
| 12 | Sofa 3-Seater | 5 | 22900.00 | 11975.00 | +10925.00 |
| 9 | Dining Table Oak | 5 | 15900.00 | 11975.00 | +3925.00 |
| 10 | Office Chair Ergo | 5 | 5900.00 | 11975.00 | -6075.00 |
| 11 | Bookshelf 5-Tier | 5 | 3200.00 | 11975.00 | -8775.00 |
| 17 | Mystery of the Lost City | 7 | 350.00 | 350.00 | 0.00 |
| 15 | The Art of Programming | 8 | 590.00 | 520.00 | +70.00 |
| 16 | Thai Cuisine Cookbook | 8 | 450.00 | 520.00 | -70.00 |

รายงานนี้ทำให้ทีมจัดซื้อเห็นได้ทันทีว่าสินค้าตัวไหน "แพงผิดปกติ" หรือ "ถูกผิดปกติ" เมื่อเทียบกับสินค้าอื่นในหมวดเดียวกัน เช่น `Laptop Pro 14` แพงกว่าค่าเฉลี่ยของหมวด Computers ถึง 23,180 บาท ในขณะที่ `Wireless Mouse` ถูกกว่าค่าเฉลี่ยถึง 19,130 บาท — ข้อมูลเชิงลึกแบบนี้ทำได้ในคิวรีเดียว ไม่ต้องคำนวณ 2 รอบแล้วมา join เอง ซึ่งคือประโยชน์หลักของ window functions

---

## สรุปท้ายบท

- **Window function** ต่างจาก aggregate function ธรรมดาตรงที่ **ไม่ยุบแถว** — มันคำนวณค่าสรุประดับกลุ่มแล้วแนบกลับเข้าไปในทุกแถวต้นฉบับ ทำให้เปรียบเทียบค่าระดับแถวกับค่าสรุประดับกลุ่มได้ในคิวรีเดียว
- ทุก window function ต้องมี `OVER (...)` ต่อท้ายเสมอ — `OVER()` แบบว่างเปล่าหมายถึงมองทั้งผลลัพธ์เป็น window เดียว
- `PARTITION BY column` แบ่งข้อมูลเป็นกลุ่มย่อยก่อนคำนวณ เทียบเท่ากับ `GROUP BY` แต่ไม่ยุบแถว และรองรับหลายคอลัมน์พร้อมกัน
- Aggregate function ที่คุ้นเคย (`SUM`, `AVG`, `COUNT`, `MIN`, `MAX`) ใช้เป็น window function ได้ทันทีเพียงเติม `OVER (...)`
- การเพิ่ม `ORDER BY` ภายใน `OVER(...)` เปลี่ยนพฤติกรรมจาก "คำนวณจากทั้งกลุ่ม" เป็น "คำนวณจากแถวแรกถึงแถวปัจจุบัน" โดยอัตโนมัติ ทำให้เกิด running total และ running average
- **Frame clause** (`ROWS BETWEEN` / `RANGE BETWEEN`) ควบคุมขอบเขตของ window ได้ละเอียดขึ้น เช่น `UNBOUNDED PRECEDING`, `N PRECEDING`, `CURRENT ROW`, `N FOLLOWING`, `UNBOUNDED FOLLOWING` — โดยทั่วไปนิยมใช้ `ROWS` มากกว่า `RANGE` เพราะทำงานได้แน่นอนกว่าเมื่อมีค่าซ้ำใน `ORDER BY`
- Frame ที่กำหนดเองทำให้สร้าง **moving average** (`N PRECEDING AND CURRENT ROW`) และ **centered window** (`N PRECEDING AND N FOLLOWING`) ได้อย่างแม่นยำ
- **`WINDOW` clause** ช่วยตั้งชื่อ window definition ไว้ล่วงหน้าและเรียกใช้ซ้ำด้วย `OVER w` ลดการเขียน `PARTITION BY`/`ORDER BY` ซ้ำซ้อนในคิวรีที่มี window function หลายตัว
- Window function ถูกประมวลผลหลัง `WHERE`/`GROUP BY`/`HAVING` แต่ก่อน `ORDER BY`/`LIMIT` และไม่สามารถใช้ผลลัพธ์ของมันใน `WHERE` ได้โดยตรง (ต้องครอบด้วย subquery หรือ CTE)

บทถัดไปจะพาไปลึกขึ้นสู่กลุ่ม window function ที่ทรงพลังกว่านี้ ได้แก่ `ROW_NUMBER()`, `RANK()`, `DENSE_RANK()`, `NTILE()`, `LAG()`, `LEAD()`, `FIRST_VALUE()`, `LAST_VALUE()` ซึ่งใช้สำหรับการจัดอันดับ การเปรียบเทียบกับแถวก่อนหน้า/ถัดไป และการวิเคราะห์แนวโน้มขั้นสูง

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1

เขียนคิวรีแสดง `product_id`, `product_name`, `unit_price` ของสินค้าที่ `is_active = true` ทั้งหมด พร้อมคอลัมน์ `avg_all_active` ที่เป็นค่าเฉลี่ยราคาของสินค้า active ทั้งหมด (ปัดเศษ 2 ตำแหน่ง) โดยใช้ `OVER()` แบบไม่ partition

<details>
<summary>เฉลย</summary>

```sql
SELECT product_id, product_name, unit_price,
       ROUND(AVG(unit_price) OVER (), 2) AS avg_all_active
FROM products
WHERE is_active = true
ORDER BY product_id;
```

ผลลัพธ์:

| product_id | product_name | unit_price | avg_all_active |
|---|---|---|---|
| 1 | Laptop Pro 14 | 42900.00 | 10496.47 |
| 2 | Laptop Air 13 | 32900.00 | 10496.47 |
| 3 | Wireless Mouse | 590.00 | 10496.47 |
| 4 | Mechanical Keyboard | 2490.00 | 10496.47 |
| 5 | Smartphone X12 | 24900.00 | 10496.47 |
| 6 | Smartphone Lite | 9900.00 | 10496.47 |
| 7 | Bluetooth Earbuds | 1990.00 | 10496.47 |
| 8 | Smartwatch Series 5 | 8900.00 | 10496.47 |
| 9 | Dining Table Oak | 15900.00 | 10496.47 |
| 10 | Office Chair Ergo | 5900.00 | 10496.47 |
| 11 | Bookshelf 5-Tier | 3200.00 | 10496.47 |
| 12 | Sofa 3-Seater | 22900.00 | 10496.47 |
| 13 | Rice Cooker Deluxe | 1590.00 | 10496.47 |
| 14 | Air Fryer XL | 2990.00 | 10496.47 |
| 15 | The Art of Programming | 590.00 | 10496.47 |
| 16 | Thai Cuisine Cookbook | 450.00 | 10496.47 |
| 17 | Mystery of the Lost City | 350.00 | 10496.47 |

`avg_all_active` เท่ากับ 10496.47 ทุกแถว เพราะ `OVER()` มองทั้งผลลัพธ์ (สินค้า active ทั้ง 17 ชิ้น) เป็น window เดียว สินค้า `Whispers in the Dark` (product_id 18) ไม่ปรากฏเพราะถูกกรองออกด้วย `WHERE is_active = true` ตั้งแต่ก่อนคำนวณ window function

</details>

### แบบฝึกหัดที่ 2

เขียนคิวรีแสดง `product_id`, `product_name`, `supplier_id` ของสินค้าทุกชิ้น (รวม inactive) พร้อมคอลัมน์ `products_per_supplier` ที่นับจำนวนสินค้าที่แต่ละ supplier จัดส่งให้ทั้งหมด โดยใช้ `PARTITION BY supplier_id`

<details>
<summary>เฉลย</summary>

```sql
SELECT product_id, product_name, supplier_id,
       COUNT(*) OVER (PARTITION BY supplier_id) AS products_per_supplier
FROM products
ORDER BY supplier_id, product_id;
```

ผลลัพธ์:

| product_id | product_name | supplier_id | products_per_supplier |
|---|---|---|---|
| 1 | Laptop Pro 14 | 1 | 2 |
| 8 | Smartwatch Series 5 | 1 | 2 |
| 3 | Wireless Mouse | 2 | 3 |
| 4 | Mechanical Keyboard | 2 | 3 |
| 7 | Bluetooth Earbuds | 2 | 3 |
| 11 | Bookshelf 5-Tier | 3 | 1 |
| 9 | Dining Table Oak | 5 | 3 |
| 10 | Office Chair Ergo | 5 | 3 |
| 12 | Sofa 3-Seater | 5 | 3 |
| 15 | The Art of Programming | 6 | 4 |
| 16 | Thai Cuisine Cookbook | 6 | 4 |
| 17 | Mystery of the Lost City | 6 | 4 |
| 18 | Whispers in the Dark | 6 | 4 |
| 2 | Laptop Air 13 | 7 | 3 |
| 5 | Smartphone X12 | 7 | 3 |
| 6 | Smartphone Lite | 7 | 3 |
| 13 | Rice Cooker Deluxe | 8 | 2 |
| 14 | Air Fryer XL | 8 | 2 |

`supplier_id = 4` (Pacific Traders) ไม่มีสินค้าเลยจึงไม่ปรากฏในผลลัพธ์ — สังเกตว่า `Sunrise Books Publishing` (supplier_id 6) จัดส่งสินค้าให้มากที่สุดคือ 4 ชิ้น รวมถึง `Whispers in the Dark` ที่เป็น inactive ด้วย เพราะคิวรีนี้ไม่ได้กรอง `is_active`

</details>

### แบบฝึกหัดที่ 3

เขียนคิวรีแสดง `payment_id`, `order_id`, `amount` ของการชำระเงินทุกรายการ พร้อมคอลัมน์ `avg_all_payments` ที่เป็นค่าเฉลี่ยของยอดชำระเงินทั้งหมด (ปัดเศษ 2 ตำแหน่ง) โดยใช้ `AVG() OVER()`

<details>
<summary>เฉลย</summary>

```sql
SELECT payment_id, order_id, amount,
       ROUND(AVG(amount) OVER (), 2) AS avg_all_payments
FROM payments
ORDER BY payment_id;
```

ผลลัพธ์:

| payment_id | order_id | amount | avg_all_payments |
|---|---|---|---|
| 1 | 1 | 44080.00 | 17975.56 |
| 2 | 2 | 26890.00 | 17975.56 |
| 3 | 3 | 9900.00 | 17975.56 |
| 4 | 4 | 35390.00 | 17975.56 |
| 5 | 5 | 2470.00 | 17975.56 |
| 6 | 6 | 15900.00 | 17975.56 |
| 7 | 7 | 6170.00 | 17975.56 |
| 8 | 8 | 8900.00 | 17975.56 |
| 9 | 9 | 7060.00 | 17975.56 |
| 10 | 11 | 9100.00 | 17975.56 |
| 11 | 12 | 44670.00 | 17975.56 |
| 12 | 13 | 24900.00 | 17975.56 |
| 13 | 14 | 2980.00 | 17975.56 |
| 14 | 15 | 11890.00 | 17975.56 |
| 15 | 16 | 22900.00 | 17975.56 |
| 16 | 17 | 32900.00 | 17975.56 |
| 17 | 18 | 12880.00 | 17975.56 |
| 18 | 19 | 4580.00 | 17975.56 |

ค่า `avg_all_payments = 17975.56` ตรงกับ `avg_all_orders` ใน Step 282 พอดี เพราะแต่ละ payment สอดคล้องกับ order เพียงใบเดียวและยอดชำระตรงกับยอดสั่งซื้อทั้งหมด

</details>

### แบบฝึกหัดที่ 4

เขียนคิวรีแสดง `order_id`, `customer_id` ของ order ที่ `status = 'completed'` ทั้งหมด พร้อมคอลัมน์ `orders_by_customer` ที่นับจำนวน order ของลูกค้าคนนั้น โดยใช้ `COUNT(*) OVER (PARTITION BY customer_id)`

<details>
<summary>เฉลย</summary>

```sql
SELECT o.order_id, o.customer_id,
       COUNT(*) OVER (PARTITION BY o.customer_id) AS orders_by_customer
FROM orders o
WHERE o.status = 'completed'
ORDER BY o.customer_id, o.order_id;
```

ผลลัพธ์:

| order_id | customer_id | orders_by_customer |
|---|---|---|
| 1 | 1 | 2 |
| 9 | 1 | 2 |
| 4 | 2 | 2 |
| 17 | 2 | 2 |
| 2 | 3 | 2 |
| 13 | 3 | 2 |
| 6 | 4 | 1 |
| 3 | 5 | 2 |
| 15 | 5 | 2 |
| 8 | 6 | 1 |
| 5 | 7 | 1 |
| 7 | 9 | 2 |
| 19 | 9 | 2 |
| 12 | 10 | 1 |
| 11 | 11 | 1 |
| 16 | 12 | 1 |
| 14 | 13 | 1 |
| 18 | 14 | 1 |

ลูกค้า 1, 2, 3, 5, 9 มี order ที่ completed คนละ 2 ใบ ส่วนที่เหลือมีคนละ 1 ใบ (customer_id = 8 ไม่ปรากฏเลยเพราะ order เดียวของเขาคือ order_id 10 ถูกยกเลิก)

</details>

### แบบฝึกหัดที่ 5

เขียนคิวรีแสดง `payment_id`, `order_id`, `payment_date` (แสดงเฉพาะวันที่), `amount` ของทุกรายการชำระเงิน พร้อมคอลัมน์ `running_total_payments` ที่เป็นยอดชำระเงินสะสม เรียงตาม `payment_date`

<details>
<summary>เฉลย</summary>

```sql
SELECT payment_id, order_id, payment_date::date AS payment_day, amount,
       SUM(amount) OVER (ORDER BY payment_date) AS running_total_payments
FROM payments
ORDER BY payment_date;
```

ผลลัพธ์:

| payment_id | order_id | payment_day | amount | running_total_payments |
|---|---|---|---|---|
| 1 | 1 | 2024-08-01 | 44080.00 | 44080.00 |
| 2 | 2 | 2024-08-01 | 26890.00 | 70970.00 |
| 3 | 3 | 2024-08-01 | 9900.00 | 80870.00 |
| 4 | 4 | 2024-08-02 | 35390.00 | 116260.00 |
| 5 | 5 | 2024-08-02 | 2470.00 | 118730.00 |
| 6 | 6 | 2024-08-03 | 15900.00 | 134630.00 |
| 7 | 7 | 2024-08-03 | 6170.00 | 140800.00 |
| 8 | 8 | 2024-08-04 | 8900.00 | 149700.00 |
| 9 | 9 | 2024-08-04 | 7060.00 | 156760.00 |
| 10 | 11 | 2024-08-05 | 9100.00 | 165860.00 |
| 11 | 12 | 2024-08-06 | 44670.00 | 210530.00 |
| 12 | 13 | 2024-08-06 | 24900.00 | 235430.00 |
| 13 | 14 | 2024-08-07 | 2980.00 | 238410.00 |
| 14 | 15 | 2024-08-07 | 11890.00 | 250300.00 |
| 15 | 16 | 2024-08-08 | 22900.00 | 273200.00 |
| 16 | 17 | 2024-08-08 | 32900.00 | 306100.00 |
| 17 | 18 | 2024-08-09 | 12880.00 | 318980.00 |
| 18 | 19 | 2024-08-09 | 4580.00 | 323560.00 |

ยอดสะสมสุดท้าย 323,560.00 ตรงกับผลรวมของ `daily_total` ทั้งหมดใน Step 286 พอดี เพราะเป็นการเรียงตาม `payment_date` ที่ละเอียดถึงระดับนาที (ต่างจาก Step 286 ที่ group ตามวันก่อน) — สังเกตว่าวันที่ 08-01 มี 3 รายการชำระเงินและยอดสะสมค่อย ๆ เพิ่มทีละแถว ไม่ใช่กระโดดทีเดียวเหมือนรายงานรายวัน

</details>

### แบบฝึกหัดที่ 6

เขียนคิวรีหาค่าเฉลี่ยเคลื่อนที่ (moving average) แบบ 3 วัน ของ **จำนวน order ต่อวัน** (ไม่ใช่ยอดขาย) โดยใช้ `ROWS BETWEEN 2 PRECEDING AND CURRENT ROW`

<details>
<summary>เฉลย</summary>

```sql
WITH daily_orders AS (
    SELECT order_date::date AS order_day, COUNT(*) AS order_count
    FROM orders
    WHERE status = 'completed'
    GROUP BY order_date::date
)
SELECT order_day, order_count,
       ROUND(AVG(order_count) OVER (
           ORDER BY order_day
           ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
       ), 2) AS moving_avg_orders_3d
FROM daily_orders
ORDER BY order_day;
```

ผลลัพธ์:

| order_day | order_count | moving_avg_orders_3d |
|---|---|---|
| 2024-08-01 | 3 | 3.00 |
| 2024-08-02 | 2 | 2.50 |
| 2024-08-03 | 2 | 2.33 |
| 2024-08-04 | 2 | 2.00 |
| 2024-08-05 | 1 | 1.67 |
| 2024-08-06 | 2 | 1.67 |
| 2024-08-07 | 2 | 1.67 |
| 2024-08-08 | 2 | 2.00 |
| 2024-08-09 | 2 | 2.00 |

วันที่ `08-01` มี order มากที่สุด (3 orders) ค่าเฉลี่ยเคลื่อนที่ในช่วงกลางเดือนจึงลดลงมาอยู่ที่ประมาณ 1.67-2.00 orders ต่อวัน แสดงให้เห็นว่าจำนวน order ต่อวันค่อนข้างคงที่ตลอดช่วงเวลาที่มีข้อมูล

</details>

### แบบฝึกหัดที่ 7

เขียนคิวรีแสดงสินค้าที่ active ทั้งหมด พร้อมคอลัมน์ `sum_price_cat`, `avg_price_cat`, `count_cat` (ผลรวม ค่าเฉลี่ย และจำนวนสินค้าของหมวดหมู่เดียวกัน) โดยใช้ `WINDOW` clause เพื่อไม่ต้องเขียน `PARTITION BY category_id` ซ้ำ 3 รอบ

<details>
<summary>เฉลย</summary>

```sql
SELECT product_id, product_name, category_id, unit_price,
       SUM(unit_price) OVER w AS sum_price_cat,
       ROUND(AVG(unit_price) OVER w, 2) AS avg_price_cat,
       COUNT(*) OVER w AS count_cat
FROM products
WHERE is_active = true
WINDOW w AS (PARTITION BY category_id)
ORDER BY category_id, product_id;
```

ผลลัพธ์:

| product_id | product_name | category_id | unit_price | sum_price_cat | avg_price_cat | count_cat |
|---|---|---|---|---|---|---|
| 1 | Laptop Pro 14 | 2 | 42900.00 | 78880.00 | 19720.00 | 4 |
| 2 | Laptop Air 13 | 2 | 32900.00 | 78880.00 | 19720.00 | 4 |
| 3 | Wireless Mouse | 2 | 590.00 | 78880.00 | 19720.00 | 4 |
| 4 | Mechanical Keyboard | 2 | 2490.00 | 78880.00 | 19720.00 | 4 |
| 5 | Smartphone X12 | 3 | 24900.00 | 45690.00 | 11422.50 | 4 |
| 6 | Smartphone Lite | 3 | 9900.00 | 45690.00 | 11422.50 | 4 |
| 7 | Bluetooth Earbuds | 3 | 1990.00 | 45690.00 | 11422.50 | 4 |
| 8 | Smartwatch Series 5 | 3 | 8900.00 | 45690.00 | 11422.50 | 4 |
| 13 | Rice Cooker Deluxe | 4 | 1590.00 | 4580.00 | 2290.00 | 2 |
| 14 | Air Fryer XL | 4 | 2990.00 | 4580.00 | 2290.00 | 2 |
| 9 | Dining Table Oak | 5 | 15900.00 | 47900.00 | 11975.00 | 4 |
| 10 | Office Chair Ergo | 5 | 5900.00 | 47900.00 | 11975.00 | 4 |
| 11 | Bookshelf 5-Tier | 5 | 3200.00 | 47900.00 | 11975.00 | 4 |
| 12 | Sofa 3-Seater | 5 | 22900.00 | 47900.00 | 11975.00 | 4 |
| 17 | Mystery of the Lost City | 7 | 350.00 | 350.00 | 350.00 | 1 |
| 15 | The Art of Programming | 8 | 590.00 | 1040.00 | 520.00 | 2 |
| 16 | Thai Cuisine Cookbook | 8 | 450.00 | 1040.00 | 520.00 | 2 |

ทั้งสามคอลัมน์อ้างอิง `w` ตัวเดียวกันซึ่งประกาศไว้แค่ครั้งเดียวใน `WINDOW` clause ทำให้โค้ดกระชับกว่าการเขียน `OVER (PARTITION BY category_id)` ซ้ำ 3 รอบ และตัวเลขทั้งหมดตรงกับ Step 284 ทุกประการ

</details>

### แบบฝึกหัดที่ 8

จากรายงานยอดขายรายวัน (Step 286) ให้เขียนคิวรีเปรียบเทียบผลลัพธ์ของ `SUM() OVER (ORDER BY order_day ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)` กับ `SUM() OVER (ORDER BY order_day RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)` แล้วอธิบายว่าทำไมผลลัพธ์ทั้งสองคอลัมน์จึงเหมือนกันทุกแถวในกรณีนี้

<details>
<summary>เฉลย</summary>

```sql
WITH daily AS (
    SELECT o.order_date::date AS order_day,
           SUM(oi.quantity * oi.unit_price) AS daily_total
    FROM orders o
    JOIN order_items oi ON oi.order_id = o.order_id
    WHERE o.status = 'completed'
    GROUP BY o.order_date::date
)
SELECT order_day, daily_total,
       SUM(daily_total) OVER (
           ORDER BY order_day
           ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
       ) AS total_rows,
       SUM(daily_total) OVER (
           ORDER BY order_day
           RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
       ) AS total_range
FROM daily
ORDER BY order_day;
```

ผลลัพธ์:

| order_day | daily_total | total_rows | total_range |
|---|---|---|---|
| 2024-08-01 | 80870.00 | 80870.00 | 80870.00 |
| 2024-08-02 | 37860.00 | 118730.00 | 118730.00 |
| 2024-08-03 | 22070.00 | 140800.00 | 140800.00 |
| 2024-08-04 | 15960.00 | 156760.00 | 156760.00 |
| 2024-08-05 | 9100.00 | 165860.00 | 165860.00 |
| 2024-08-06 | 69570.00 | 235430.00 | 235430.00 |
| 2024-08-07 | 14870.00 | 250300.00 | 250300.00 |
| 2024-08-08 | 55800.00 | 306100.00 | 306100.00 |
| 2024-08-09 | 17460.00 | 323560.00 | 323560.00 |

**คำอธิบาย:** ผลลัพธ์เหมือนกันทุกแถวเพราะคอลัมน์ `order_day` ในมุมมองของ CTE `daily` (ที่ผ่านการ `GROUP BY order_date::date` มาแล้ว) **ไม่มีค่าซ้ำกันเลย** — แต่ละวันปรากฏเพียงหนึ่งแถวเท่านั้น เมื่อไม่มี ties, `RANGE` กับ `ROWS` จะให้ผลลัพธ์เหมือนกันเสมอ เพราะ `RANGE` จะรวมแถวที่มีค่า `ORDER BY` เท่ากันเข้ากรอบเดียวกัน แต่ในเมื่อไม่มีแถวไหนมีค่าเท่ากันเลย กรอบของ `RANGE` และ `ROWS` จึงครอบคลุมแถวชุดเดียวกันพอดีในทุกตำแหน่ง ถ้าใช้ตาราง `payments` ที่มี `payment_date` ระดับนาที (ซึ่งไม่ซ้ำกันเช่นกัน) ก็จะได้ผลลัพธ์เหมือนกันด้วยเหตุผลเดียวกัน แต่ถ้า `ORDER BY` เป็นคอลัมน์ที่มีค่าซ้ำกันได้ เช่น `ORDER BY category_id` ผลลัพธ์ของสองแบบจะเริ่มต่างกันทันที

</details>

### แบบฝึกหัดที่ 9

เขียนคิวรีแสดง `employee_id`, `first_name`, `department`, `hire_date` ของพนักงานทุกคน พร้อมคอลัมน์ `hire_seq_in_dept` ที่นับลำดับการเข้าร่วมงานของแต่ละแผนก (คนแรกของแผนก = 1, คนที่สอง = 2, ...) โดยใช้ `PARTITION BY department ORDER BY hire_date`

<details>
<summary>เฉลย</summary>

```sql
SELECT employee_id, first_name, department, hire_date,
       COUNT(*) OVER (PARTITION BY department ORDER BY hire_date) AS hire_seq_in_dept
FROM employees
ORDER BY department, hire_date;
```

ผลลัพธ์:

| employee_id | first_name | department | hire_date | hire_seq_in_dept |
|---|---|---|---|---|
| 5 | Preecha | Customer Service | 2022-01-05 | 1 |
| 6 | Ratana | Customer Service | 2022-08-11 | 2 |
| 1 | Suda | Sales | 2020-01-10 | 1 |
| 2 | Anan | Sales | 2020-03-15 | 2 |
| 3 | Kittipong | Sales | 2021-02-01 | 3 |
| 4 | Malee | Sales | 2021-06-20 | 4 |
| 7 | Chalermchai | Sales | 2023-01-20 | 5 |

ตัวอย่างนี้ใช้ `COUNT(*)` ร่วมกับ `ORDER BY` ภายใน `OVER(...)` ซึ่งอาศัย default frame (`RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`) ทำให้ `COUNT(*)` กลายเป็น "นับสะสม" แทนที่จะนับสมาชิกทั้งกลุ่ม — เป็นอีกวิธีหนึ่งในการสร้างลำดับ (นอกจากฟังก์ชัน `ROW_NUMBER()` ที่จะเรียนใน Part ถัดไป) เพราะเมื่อ `hire_date` ไม่ซ้ำกัน `COUNT(*) OVER (... ORDER BY hire_date)` ก็ทำหน้าที่เหมือน running count ได้พอดี

</details>

### แบบฝึกหัดที่ 10

เขียนคิวรีหา order ทั้งหมดที่มียอดขาย (`order_total`) **สูงกว่า** ค่าเฉลี่ยของลูกค้าคนนั้น โดยต้องใช้ CTE ครอบ window function ก่อน แล้วค่อยกรองด้วย `WHERE` (เพราะ `WHERE` เรียกใช้ window function โดยตรงไม่ได้)

<details>
<summary>เฉลย</summary>

```sql
WITH order_totals AS (
    SELECT o.order_id,
           o.customer_id,
           SUM(oi.quantity * oi.unit_price) AS order_total,
           AVG(SUM(oi.quantity * oi.unit_price)) OVER (PARTITION BY o.customer_id) AS customer_avg
    FROM orders o
    JOIN order_items oi ON oi.order_id = o.order_id
    WHERE o.status = 'completed'
    GROUP BY o.order_id, o.customer_id
)
SELECT order_id, customer_id, order_total, ROUND(customer_avg, 2) AS customer_avg
FROM order_totals
WHERE order_total > customer_avg
ORDER BY customer_id, order_id;
```

ผลลัพธ์:

| order_id | customer_id | order_total | customer_avg |
|---|---|---|---|
| 1 | 1 | 44080.00 | 25570.00 |
| 4 | 2 | 35390.00 | 34145.00 |
| 2 | 3 | 26890.00 | 25895.00 |
| 15 | 5 | 11890.00 | 10895.00 |
| 7 | 9 | 6170.00 | 5375.00 |

ได้ผลลัพธ์ 5 orders ที่ "ซื้อเยอะกว่าปกติ" เมื่อเทียบกับลูกค้าคนนั้นเอง — สังเกตว่าลูกค้าที่มี order เดียว (เช่น customer_id 4, 6, 7, 10, 11, 12, 13, 14) จะไม่ปรากฏในผลลัพธ์นี้เลย เพราะ `order_total` ของพวกเขาเท่ากับ `customer_avg` พอดี (ไม่มีทางสูงกว่าค่าเฉลี่ยของตัวเองได้) จุดสำคัญของแบบฝึกหัดนี้คือการเข้าใจว่าทำไมต้องครอบด้วย CTE ก่อน — ถ้าลองเขียน `WHERE order_total > AVG(...) OVER (...)` ตรง ๆ ในคิวรีเดียวกับที่มี `GROUP BY` จะได้ error `window functions are not allowed in WHERE` ทันที เพราะลำดับการประมวลผลของ SQL คำนวณ `WHERE` ก่อนคำนวณ window function เสมอ

</details>

---

**บทถัดไป:** [Part 030 — Window Functions ขั้นสูง: Ranking และ Navigation Functions](./part-030-window-functions-advanced.md)
