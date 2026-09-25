# Part 033: Numeric/Math Functions และ Type Casting

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับกลาง | Part 033

บทนี้อยู่ในชุด "e-commerce dataset" ที่ใช้ร่วมกันตั้งแต่ Part 021 ถึง Part 039 เราจะใช้ schema เดียวกันตลอดทั้งชุด เพื่อให้ผู้เรียนเห็นภาพว่าฟังก์ชันแต่ละกลุ่มถูกนำไปใช้กับข้อมูลจริงในธุรกิจร้านค้าออนไลน์อย่างไร

## เป้าหมายการเรียนรู้

เมื่อจบบทนี้ ผู้เรียนจะสามารถ:

- ใช้ฟังก์ชันคณิตศาสตร์พื้นฐานของ PostgreSQL ได้แก่ `ROUND`, `CEIL`/`CEILING`, `FLOOR`, `TRUNC`, `ABS`, `SIGN` เพื่อจัดการตัวเลขในงานธุรกิจ
- แยกแยะการใช้งาน `MOD`, ตัวดำเนินการ `%`, และฟังก์ชัน `DIV` สำหรับการหารเอาเศษและการหารเอาจำนวนเต็ม
- ใช้ฟังก์ชันเลขยกกำลังและลอการิทึม `POWER`, `SQRT`, `CBRT`, `EXP`, `LN`, `LOG` ในการคำนวณเชิงธุรกิจและสถิติเบื้องต้น
- สร้างตัวเลขสุ่มด้วย `RANDOM()` สุ่มค่าภายในช่วงที่กำหนด และสุ่มแถวข้อมูล พร้อมเข้าใจข้อควรระวังด้าน performance
- เข้าใจพฤติกรรมการปัดเศษของ `ROUND` บนชนิดข้อมูล numeric เทียบกับ floating point และแนวคิด bankers rounding
- เลือกใช้ `CAST`, ตัวดำเนินการ `::`, หรือฟังก์ชัน constructor เช่น `to_number`, `to_char` ได้อย่างเหมาะสม
- แปลงข้อมูลระหว่าง numeric/text/integer อย่างปลอดภัย โดยตรวจสอบรูปแบบข้อมูลด้วย regex หรือฟังก์ชัน `pg_input_is_valid` ก่อนแปลงจริง
- ทบทวนฟังก์ชันเกี่ยวกับ sequence ได้แก่ `NEXTVAL`, `CURRVAL`, `SETVAL` เพื่อเตรียมความพร้อมสำหรับ Part 039
- ใช้ aggregate ทางสถิติขั้นสูง เช่น `CORR`, `COVAR_POP`/`COVAR_SAMP`, และตระกูล `REGR_*` สำหรับงานวิเคราะห์ข้อมูลเบื้องต้น
- ประยุกต์ฟังก์ชันตัวเลขทั้งหมดเพื่อคำนวณโจทย์ธุรกิจจริง เช่น ส่วนลด, VAT, ค่าคอมมิชชั่น และการปัดราคาให้ลงท้ายด้วย `.99`

---

## เตรียมข้อมูล

ชุดข้อมูลนี้จำลองระบบร้านค้าออนไลน์ ประกอบด้วยตาราง 9 ตาราง ครอบคลุมหมวดหมู่สินค้า ผู้จัดจำหน่าย สินค้า ลูกค้า พนักงาน คำสั่งซื้อ รายการสินค้าในคำสั่งซื้อ รีวิว และการชำระเงิน โครงสร้างและข้อมูลชุดนี้จะถูกใช้ซ้ำตลอด Part 021-039

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
-- ============ categories ============
INSERT INTO categories (category_id, category_name, parent_category_id) VALUES
(1, 'Electronics', NULL),
(2, 'Computers', 1),
(3, 'Laptops', 2),
(4, 'Smartphones', 1),
(5, 'Fashion', NULL),
(6, 'Men''s Clothing', 5),
(7, 'Women''s Clothing', 5),
(8, 'Home & Kitchen', NULL),
(9, 'Furniture', 8),
(10, 'Appliances', 8);

SELECT setval('categories_category_id_seq', 10);

-- ============ suppliers ============
INSERT INTO suppliers (supplier_id, supplier_name, country) VALUES
(1, 'Global Tech Supply', 'USA'),
(2, 'Bangkok Electronics Co.', 'Thailand'),
(3, 'Shenzhen Gadgets Ltd.', 'China'),
(4, 'Nordic Furniture House', 'Sweden'),
(5, 'Osaka Appliances', 'Japan'),
(6, 'Milan Fashion Group', 'Italy'),
(7, 'Seoul Style Co.', 'South Korea'),
(8, 'Hanoi Textiles', 'Vietnam'),
(9, 'Berlin Home Goods', 'Germany'),
(10, 'Sydney Outdoor Supplies', 'Australia');

SELECT setval('suppliers_supplier_id_seq', 10);

-- ============ products ============
INSERT INTO products (product_id, product_name, category_id, supplier_id, unit_price, stock_quantity, is_active) VALUES
(1, 'UltraBook Pro 14"', 3, 1, 32990.00, 45, true),
(2, 'UltraBook Air 13"', 3, 1, 24990.00, 60, true),
(3, 'GameMaster Laptop 15"', 3, 3, 45990.00, 20, true),
(4, 'SmartPhone X12', 4, 2, 21990.00, 100, true),
(5, 'SmartPhone X12 Mini', 4, 2, 17990.00, 80, true),
(6, 'SmartPhone Budget A1', 4, 3, 6990.00, 150, true),
(7, 'Wireless Earbuds Pro', 1, 3, 2990.00, 200, true),
(8, 'Men''s Cotton Shirt', 6, 8, 590.00, 300, true),
(9, 'Men''s Denim Jeans', 6, 8, 890.00, 250, true),
(10, 'Women''s Summer Dress', 7, 6, 1290.00, 180, true),
(11, 'Women''s Silk Blouse', 7, 7, 1590.00, 90, true),
(12, 'Oak Dining Table', 9, 4, 12990.00, 15, true),
(13, 'Modern Sofa 3-Seater', 9, 4, 18990.00, 10, true),
(14, 'Ergonomic Office Chair', 9, 4, 5990.00, 40, true),
(15, 'Robot Vacuum Cleaner', 10, 5, 8990.00, 35, true),
(16, 'Air Purifier Max', 10, 5, 4990.00, 50, true),
(17, 'Microwave Oven 900W', 10, 9, 2990.00, 60, true),
(18, 'Stand Mixer Deluxe', 10, 9, 6490.00, 25, true),
(19, 'Outdoor Camping Tent', 1, 10, 3990.00, 40, true),
(20, 'Portable BBQ Grill', 1, 10, 2490.00, 30, false);

SELECT setval('products_product_id_seq', 20);

-- ============ customers ============
INSERT INTO customers (customer_id, first_name, last_name, email, country, signup_date) VALUES
(1, 'Somchai', 'Jaidee', 'somchai.j@example.com', 'Thailand', '2023-01-15'),
(2, 'Suda', 'Meesuk', 'suda.m@example.com', 'Thailand', '2023-02-20'),
(3, 'Anong', 'Srisawat', 'anong.s@example.com', 'Thailand', '2023-03-05'),
(4, 'John', 'Smith', 'john.smith@example.com', 'USA', '2023-01-10'),
(5, 'Emily', 'Davis', 'emily.davis@example.com', 'USA', '2023-04-12'),
(6, 'Wei', 'Chen', 'wei.chen@example.com', 'China', '2023-02-01'),
(7, 'Yuki', 'Tanaka', 'yuki.tanaka@example.com', 'Japan', '2023-05-18'),
(8, 'Min-jun', 'Park', 'minjun.park@example.com', 'South Korea', '2023-03-22'),
(9, 'Somsri', 'Boonmee', 'somsri.b@example.com', 'Thailand', '2023-06-01'),
(10, 'Piti', 'Wongsa', 'piti.w@example.com', 'Thailand', '2023-04-30'),
(11, 'Sarah', 'Johnson', 'sarah.j@example.com', 'UK', '2023-07-14'),
(12, 'Hans', 'Mueller', 'hans.mueller@example.com', 'Germany', '2023-05-05'),
(13, 'Nok', 'Chaiyaporn', 'nok.c@example.com', 'Thailand', '2023-08-09'),
(14, 'Liu', 'Yang', 'liu.yang@example.com', 'China', '2023-06-25'),
(15, 'Aom', 'Petchara', 'aom.p@example.com', 'Thailand', '2023-09-01');

SELECT setval('customers_customer_id_seq', 15);

-- ============ employees ============
INSERT INTO employees (employee_id, first_name, last_name, hire_date, manager_id, department) VALUES
(1, 'Prasert', 'Kittikul', '2020-01-10', NULL, 'Management'),
(2, 'Malee', 'Suksawat', '2020-03-15', 1, 'Sales'),
(3, 'Chaiwat', 'Rungroj', '2020-06-01', 1, 'Sales'),
(4, 'Kanya', 'Thongdee', '2021-02-20', 2, 'Sales'),
(5, 'Anuwat', 'Srisuk', '2021-05-11', 2, 'Sales'),
(6, 'Pornthip', 'Wattana', '2021-08-30', 3, 'Sales'),
(7, 'Thanawat', 'Charoen', '2022-01-15', 3, 'Sales'),
(8, 'Supaporn', 'Intarach', '2022-04-01', 1, 'Customer Service'),
(9, 'Weerachai', 'Boonsri', '2022-07-19', 8, 'Customer Service'),
(10, 'Nattaya', 'Phrom', '2023-01-05', 8, 'Customer Service');

SELECT setval('employees_employee_id_seq', 10);

-- ============ orders ============
INSERT INTO orders (order_id, customer_id, employee_id, order_date, status, ship_country) VALUES
(1, 1, 2, '2024-01-05 10:15:00+07', 'completed', 'Thailand'),
(2, 2, 2, '2024-01-08 14:30:00+07', 'completed', 'Thailand'),
(3, 3, 3, '2024-01-10 09:00:00+07', 'shipped', 'Thailand'),
(4, 4, 4, '2024-01-12 16:45:00+07', 'completed', 'USA'),
(5, 5, 4, '2024-01-15 11:20:00+07', 'completed', 'USA'),
(6, 6, 5, '2024-01-18 08:30:00+07', 'completed', 'China'),
(7, 7, 5, '2024-01-20 13:00:00+07', 'shipped', 'Japan'),
(8, 8, 6, '2024-01-22 10:10:00+07', 'completed', 'South Korea'),
(9, 1, 2, '2024-02-01 09:45:00+07', 'completed', 'Thailand'),
(10, 9, 6, '2024-02-03 15:30:00+07', 'pending', 'Thailand'),
(11, 10, 7, '2024-02-05 12:00:00+07', 'completed', 'Thailand'),
(12, 11, 7, '2024-02-08 17:15:00+07', 'cancelled', 'UK'),
(13, 3, 3, '2024-02-10 10:30:00+07', 'completed', 'Thailand'),
(14, 12, 2, '2024-02-12 14:00:00+07', 'completed', 'Germany'),
(15, 13, 3, '2024-02-15 09:20:00+07', 'shipped', 'Thailand'),
(16, 4, 4, '2024-02-18 11:00:00+07', 'completed', 'USA'),
(17, 14, 5, '2024-02-20 16:30:00+07', 'completed', 'China'),
(18, 15, 6, '2024-02-22 10:45:00+07', 'pending', 'Thailand'),
(19, 2, 2, '2024-03-01 13:15:00+07', 'completed', 'Thailand'),
(20, 6, 5, '2024-03-03 09:30:00+07', 'completed', 'China');

SELECT setval('orders_order_id_seq', 20);

-- ============ order_items ============
INSERT INTO order_items (order_item_id, order_id, product_id, quantity, unit_price) VALUES
(1, 1, 1, 1, 32990.00),
(2, 1, 7, 2, 2990.00),
(3, 2, 4, 1, 21990.00),
(4, 2, 8, 3, 590.00),
(5, 3, 12, 1, 12990.00),
(6, 4, 2, 1, 24990.00),
(7, 4, 9, 2, 890.00),
(8, 5, 6, 2, 6990.00),
(9, 6, 15, 1, 8990.00),
(10, 6, 16, 1, 4990.00),
(11, 7, 5, 1, 17990.00),
(12, 8, 10, 2, 1290.00),
(13, 8, 11, 1, 1590.00),
(14, 9, 3, 1, 45990.00),
(15, 10, 17, 1, 2990.00),
(16, 11, 13, 1, 18990.00),
(17, 11, 14, 2, 5990.00),
(18, 12, 19, 1, 3990.00),
(19, 13, 7, 3, 2990.00),
(20, 13, 8, 5, 590.00),
(21, 14, 1, 1, 32990.00),
(22, 15, 18, 1, 6490.00),
(23, 16, 4, 2, 21990.00),
(24, 17, 6, 1, 6990.00),
(25, 17, 9, 1, 890.00),
(26, 18, 20, 1, 2490.00),
(27, 19, 2, 1, 24990.00),
(28, 19, 10, 1, 1290.00),
(29, 20, 15, 1, 8990.00),
(30, 20, 16, 2, 4990.00);

SELECT setval('order_items_order_item_id_seq', 30);

-- ============ reviews ============
INSERT INTO reviews (review_id, product_id, customer_id, rating, review_text, review_date) VALUES
(1, 1, 1, 5, 'สินค้าดีมาก คุ้มราคา', '2024-01-20'),
(2, 4, 4, 4, 'Great phone, fast delivery', '2024-01-25'),
(3, 7, 1, 5, 'เสียงดีมาก แบตอึด', '2024-01-22'),
(4, 12, 3, 4, 'โต๊ะสวย ไม้ดี', '2024-01-15'),
(5, 6, 5, 3, 'OK for the price', '2024-01-28'),
(6, 15, 6, 5, 'ดูดฝุ่นเก่งมาก', '2024-01-19'),
(7, 3, 8, 4, 'Good gaming performance', '2024-01-24'),
(8, 10, 8, 5, 'ชุดสวยมาก ใส่สบาย', '2024-02-06'),
(9, 17, 9, 2, 'เสียงดังไปหน่อย', '2024-02-04'),
(10, 13, 10, 5, 'โซฟานั่งสบายมาก', '2024-02-10'),
(11, 14, 11, 4, 'Comfortable chair', '2024-02-13'),
(12, 19, 3, 4, 'เต็นท์กันน้ำดี', '2024-02-11'),
(13, 8, 12, 5, 'Nice quality shirt', '2024-02-14'),
(14, 2, 4, 5, 'Lightweight and fast', '2024-02-19'),
(15, 16, 14, 4, 'ฟอกอากาศได้ดี', '2024-02-21');

SELECT setval('reviews_review_id_seq', 15);

-- ============ payments (เฉพาะออเดอร์ที่ไม่ใช่ pending/cancelled) ============
INSERT INTO payments (payment_id, order_id, payment_date, amount, payment_method) VALUES
(1, 1, '2024-01-05 10:20:00+07', 38970.00, 'credit_card'),
(2, 2, '2024-01-08 14:35:00+07', 23760.00, 'promptpay'),
(3, 3, '2024-01-10 09:05:00+07', 12990.00, 'credit_card'),
(4, 4, '2024-01-12 16:50:00+07', 26770.00, 'paypal'),
(5, 5, '2024-01-15 11:25:00+07', 13980.00, 'paypal'),
(6, 6, '2024-01-18 08:35:00+07', 13980.00, 'bank_transfer'),
(7, 7, '2024-01-20 13:05:00+07', 17990.00, 'credit_card'),
(8, 8, '2024-01-22 10:15:00+07', 4170.00, 'credit_card'),
(9, 9, '2024-02-01 09:50:00+07', 45990.00, 'promptpay'),
(10, 11, '2024-02-05 12:05:00+07', 30970.00, 'credit_card'),
(11, 13, '2024-02-10 10:35:00+07', 11920.00, 'promptpay'),
(12, 14, '2024-02-12 14:05:00+07', 32990.00, 'bank_transfer'),
(13, 15, '2024-02-15 09:25:00+07', 6490.00, 'credit_card'),
(14, 16, '2024-02-18 11:05:00+07', 43980.00, 'paypal'),
(15, 17, '2024-02-20 16:35:00+07', 7880.00, 'debit_card'),
(16, 19, '2024-03-01 13:20:00+07', 26280.00, 'promptpay'),
(17, 20, '2024-03-03 09:35:00+07', 18970.00, 'bank_transfer');

SELECT setval('payments_payment_id_seq', 17);
```

หมายเหตุ: เราแทรกค่า primary key แบบระบุตรง ๆ (explicit) เพื่อให้ตัวอย่าง SQL ในบทอ้างอิง id ได้ตรงกันทุกบท จึงต้องเรียก `setval(...)` ปรับ sequence ให้ตามทันค่าที่ insert ไปแล้ว ไม่เช่นนั้นการ insert แถวใหม่ในอนาคตด้วย `DEFAULT` จะชนกับ primary key เดิม — เรื่องนี้จะอธิบายละเอียดใน Step 328 และ Part 039

---

## Step 321: ฟังก์ชันคณิตศาสตร์พื้นฐาน — ROUND, CEIL/CEILING, FLOOR, TRUNC, ABS, SIGN

ฟังก์ชันกลุ่มนี้คือเครื่องมือพื้นฐานที่สุดสำหรับงานตัวเลขใน SQL แต่ละตัวมีพฤติกรรมต่างกันชัดเจน:

| ฟังก์ชัน | ความหมาย | ตัวอย่าง |
|---|---|---|
| `ROUND(x, n)` | ปัดเศษให้เหลือทศนิยม n ตำแหน่ง | `ROUND(32990.456, 1) = 32990.5` |
| `CEIL(x)` / `CEILING(x)` | ปัดขึ้นเป็นจำนวนเต็มที่มากกว่าหรือเท่ากับ x | `CEIL(4.1) = 5` |
| `FLOOR(x)` | ปัดลงเป็นจำนวนเต็มที่น้อยกว่าหรือเท่ากับ x | `FLOOR(4.9) = 4` |
| `TRUNC(x, n)` | ตัดทศนิยมทิ้งโดยไม่ปัดเศษ | `TRUNC(4.999, 1) = 4.9` |
| `ABS(x)` | ค่าสัมบูรณ์ | `ABS(-150) = 150` |
| `SIGN(x)` | เครื่องหมายของตัวเลข (-1, 0, 1) | `SIGN(-25) = -1` |

### ตัวอย่างที่ 1: ปัดราคาสินค้าให้เป็นหลักร้อยด้วย ROUND แบบ negative precision

`ROUND` ในตระกูล numeric ของ PostgreSQL รับพารามิเตอร์ตัวที่สองเป็นเลขติดลบได้ ซึ่งจะปัดไปทางซ้ายของจุดทศนิยม เช่น ปัดเป็นหลักสิบ หลักร้อย หรือหลักพัน

```sql
SELECT
    product_name,
    unit_price,
    ROUND(unit_price, -2) AS rounded_to_hundred,
    ROUND(unit_price, -3) AS rounded_to_thousand
FROM products
WHERE category_id = 3
ORDER BY product_id;
```

ผลลัพธ์ตัวอย่าง:

```
      product_name      | unit_price | rounded_to_hundred | rounded_to_thousand
-------------------------+------------+---------------------+----------------------
 UltraBook Pro 14"       |   32990.00 |            33000.00 |             33000.00
 UltraBook Air 13"       |   24990.00 |            25000.00 |             25000.00
 GameMaster Laptop 15"   |   45990.00 |            46000.00 |             46000.00
```

### ตัวอย่างที่ 2: CEIL คำนวณจำนวนกล่องบรรจุที่ต้องใช้

สมมติคลังสินค้าบรรจุสินค้าใส่กล่องละ 12 ชิ้น ต้องใช้กี่กล่องเพื่อเก็บสต็อกทั้งหมด — ต้องปัดขึ้นเสมอเพราะกล่องที่เหลือไม่เต็มก็ยังต้องใช้ 1 กล่อง

```sql
SELECT
    product_name,
    stock_quantity,
    CEIL(stock_quantity / 12.0) AS boxes_needed
FROM products
WHERE category_id = 4
ORDER BY product_id;
```

ผลลัพธ์ตัวอย่าง:

```
      product_name       | stock_quantity | boxes_needed
--------------------------+----------------+--------------
 SmartPhone X12           |            100 |            9
 SmartPhone X12 Mini      |             80 |            7
 SmartPhone Budget A1     |            150 |           13
```

ข้อควรระวัง: `stock_quantity / 12` ใน PostgreSQL เป็น integer division เพราะทั้งสองฝั่งเป็น integer (ผลลัพธ์จะถูกตัดทศนิยมทิ้งก่อนที่ `CEIL` จะทำงานด้วยซ้ำ) จึงต้องหารด้วย `12.0` เพื่อบังคับให้เป็น numeric ก่อนเสมอ

### ตัวอย่างที่ 3: FLOOR หาจำนวนชุดโปรโมชั่นสูงสุดที่จัดได้

```sql
SELECT
    product_name,
    stock_quantity,
    FLOOR(stock_quantity / 3.0) AS bundle_sets_of_3
FROM products
WHERE category_id = 6
ORDER BY product_id;
```

ผลลัพธ์ตัวอย่าง:

```
     product_name    | stock_quantity | bundle_sets_of_3
----------------------+----------------+-------------------
 Men's Cotton Shirt   |            300 |               100
 Men's Denim Jeans    |            250 |                83
```

### ตัวอย่างที่ 4: TRUNC ตัดทศนิยมโดยไม่ปัดเศษ (เทียบกับ ROUND)

`TRUNC` ต่างจาก `ROUND` ตรงที่มันไม่สนใจว่าตัวเลขหลังจุดจะมากหรือน้อยกว่า 5 — มันตัดทิ้งเสมอ

```sql
SELECT
    unit_price,
    unit_price * 1.07 AS price_with_vat_raw,
    ROUND(unit_price * 1.07, 2) AS rounded_vat,
    TRUNC(unit_price * 1.07, 2) AS truncated_vat
FROM products
WHERE product_id IN (7, 9, 17);
```

ผลลัพธ์ตัวอย่าง:

```
 unit_price | price_with_vat_raw | rounded_vat | truncated_vat
------------+---------------------+-------------+----------------
    2990.00 |            3199.300 |     3199.30 |        3199.30
     890.00 |             952.300 |      952.30 |         952.30
    2990.00 |            3199.300 |     3199.30 |        3199.30
```

ในตัวอย่างนี้ค่าเท่ากันเพราะทศนิยมตำแหน่งที่สามเป็น 0 พอดี แต่หากตัวเลขคือ `952.317` ผล `ROUND(...,2)` จะได้ `952.32` ในขณะที่ `TRUNC(...,2)` จะได้ `952.31` เสมอ — ความแตกต่างนี้สำคัญมากในระบบบัญชีที่กำหนดนโยบายการปัดเศษไว้ชัดเจน

### ตัวอย่างที่ 5: ABS หาผลต่างราคาจากค่าเฉลี่ยหมวดหมู่

```sql
SELECT
    p.product_name,
    p.unit_price,
    ROUND(avg_price.category_avg, 2) AS category_avg_price,
    ABS(p.unit_price - avg_price.category_avg) AS price_deviation
FROM products p
JOIN (
    SELECT category_id, AVG(unit_price) AS category_avg
    FROM products
    GROUP BY category_id
) avg_price ON avg_price.category_id = p.category_id
WHERE p.category_id = 10
ORDER BY price_deviation DESC;
```

ผลลัพธ์ตัวอย่าง:

```
      product_name      | unit_price | category_avg_price | price_deviation
--------------------------+------------+----------------------+-------------------
 Robot Vacuum Cleaner     |    8990.00 |              5735.00 |          3255.00
 Stand Mixer Deluxe       |    6490.00 |              5735.00 |           755.00
 Air Purifier Max         |    4990.00 |              5735.00 |           745.00
 Microwave Oven 900W      |    2990.00 |              5735.00 |          2745.00
```

### ตัวอย่างที่ 6: SIGN บอกทิศทางกำไร/ขาดทุนเทียบกับเป้าหมาย

```sql
SELECT
    p.product_name,
    p.unit_price,
    p.unit_price - 10000 AS diff_from_target,
    SIGN(p.unit_price - 10000) AS direction  -- 1 = สูงกว่าเป้า, -1 = ต่ำกว่าเป้า, 0 = เท่าเป้า
FROM products p
WHERE p.category_id = 3
ORDER BY p.product_id;
```

ผลลัพธ์ตัวอย่าง:

```
      product_name      | unit_price | diff_from_target | direction
--------------------------+------------+--------------------+------------
 UltraBook Pro 14"        |   32990.00 |           22990.00 |         1
 UltraBook Air 13"        |   24990.00 |           14990.00 |         1
 GameMaster Laptop 15"    |   45990.00 |           35990.00 |         1
```

`SIGN` มีประโยชน์มากเมื่อใช้ร่วมกับ `CASE` เพื่อจัดกลุ่มสถานะ เช่น "เกินเป้า" / "ต่ำกว่าเป้า" / "เท่าเป้าพอดี" โดยไม่ต้องเขียนเงื่อนไขเปรียบเทียบซ้ำหลายครั้ง

---

## Step 322: MOD และ % operator, DIV — การหารเอาเศษ/การหารเอาจำนวนเต็ม

- `MOD(a, b)` และตัวดำเนินการ `a % b` ให้ผลลัพธ์เหมือนกันทุกประการ คือเศษที่เหลือจากการหาร
- `DIV(a, b)` คือฟังก์ชันที่คืนค่า "ผลหารเป็นจำนวนเต็ม" (ปัดเข้าใกล้ศูนย์) ต่างจากตัวดำเนินการ `/` ตรงที่ `/` ระหว่าง integer สองตัวก็ให้ผลเป็น integer อยู่แล้ว แต่ `DIV` ทำงานได้แม้ตัวถูกดำเนินการเป็น numeric

```sql
SELECT
    10 % 3 AS mod_operator,      -- 1
    MOD(10, 3) AS mod_function,  -- 1
    DIV(10, 3) AS integer_div;   -- 3
```

ผลลัพธ์ตัวอย่าง:

```
 mod_operator | mod_function | integer_div
---------------+---------------+--------------
             1 |             1 |            3
```

### ตัวอย่างที่ 1: แบ่งลูกค้าเป็นกลุ่มสำหรับแคมเปญการตลาด A/B/C ด้วย MOD

เทคนิคยอดนิยมคือใช้ `customer_id % n` เพื่อแบ่งลูกค้าออกเป็น n กลุ่มแบบสุ่มเทียม (deterministic แต่กระจายพอสมควร)

```sql
SELECT
    customer_id,
    first_name || ' ' || last_name AS customer_name,
    customer_id % 3 AS campaign_group
FROM customers
ORDER BY customer_id
LIMIT 9;
```

ผลลัพธ์ตัวอย่าง:

```
 customer_id | customer_name  | campaign_group
--------------+-----------------+------------------
            1 | Somchai Jaidee  |               1
            2 | Suda Meesuk     |               2
            3 | Anong Srisawat  |               0
            4 | John Smith      |               1
            5 | Emily Davis     |               2
            6 | Wei Chen        |               0
            7 | Yuki Tanaka     |               1
            8 | Min-jun Park    |               2
            9 | Somsri Boonmee  |               0
```

### ตัวอย่างที่ 2: DIV หาจำนวน "โหล" เต็มในสต็อก และเศษที่เหลือด้วย MOD ควบคู่กัน

```sql
SELECT
    product_name,
    stock_quantity,
    DIV(stock_quantity, 12) AS full_dozens,
    MOD(stock_quantity, 12) AS leftover_units
FROM products
WHERE category_id IN (6, 7)
ORDER BY product_id;
```

ผลลัพธ์ตัวอย่าง:

```
       product_name      | stock_quantity | full_dozens | leftover_units
---------------------------+----------------+---------------+------------------
 Men's Cotton Shirt        |            300 |            25 |               0
 Men's Denim Jeans         |            250 |            20 |              10
 Women's Summer Dress      |            180 |            15 |               0
 Women's Silk Blouse       |             90 |             7 |               6
```

### ตัวอย่างที่ 3: ใช้ MOD ตรวจสอบว่า order_id เป็นเลขคู่หรือคี่ (จำลองการแบ่งงานให้ 2 ทีม)

```sql
SELECT
    order_id,
    CASE WHEN order_id % 2 = 0 THEN 'ทีม B' ELSE 'ทีม A' END AS assigned_team
FROM orders
ORDER BY order_id
LIMIT 6;
```

ผลลัพธ์ตัวอย่าง:

```
 order_id | assigned_team
-----------+----------------
         1 | ทีม A
         2 | ทีม B
         3 | ทีม A
         4 | ทีม B
         5 | ทีม A
         6 | ทีม B
```

ข้อควรระวัง: `MOD`/`%` และ `DIV` เมื่อใช้กับตัวหารเป็น 0 จะเกิด error `division_by_zero` เสมอ (ไม่คืนค่า NULL หรือ infinity เหมือนบางภาษาโปรแกรม) จึงควรตรวจสอบตัวหารก่อนเสมอในงานจริง โดยเฉพาะเมื่อค่าตัวหารมาจากคอลัมน์ที่อาจเป็น 0 ได้

---

## Step 323: POWER, SQRT, CBRT, EXP, LN, LOG — ฟังก์ชันเลขยกกำลังและลอการิทึม

| ฟังก์ชัน | ความหมาย |
|---|---|
| `POWER(a, b)` | a ยกกำลัง b |
| `SQRT(x)` | รากที่สองของ x |
| `CBRT(x)` | รากที่สามของ x |
| `EXP(x)` | e ยกกำลัง x |
| `LN(x)` | ลอการิทึมธรรมชาติ (ฐาน e) |
| `LOG(x)` | ลอการิทึมฐาน 10 (numeric) |
| `LOG(b, x)` | ลอการิทึมฐาน b ของ x |

### ตัวอย่างที่ 1: POWER ประมาณการยอดขายในอนาคตด้วยอัตราเติบโตทบต้น

สมมติยอดขายรวมของหมวด Laptops เติบโตปีละ 8% แบบทบต้น อยากรู้ว่าอีก 3 ปีข้างหน้ายอดขายจะเป็นเท่าไร

```sql
SELECT
    ROUND(SUM(oi.quantity * oi.unit_price), 2) AS current_revenue,
    ROUND(SUM(oi.quantity * oi.unit_price) * POWER(1.08, 3), 2) AS projected_revenue_3y
FROM order_items oi
JOIN products p ON p.product_id = oi.product_id
WHERE p.category_id = 3;
```

ผลลัพธ์ตัวอย่าง:

```
 current_revenue | projected_revenue_3y
-------------------+------------------------
        112960.00 |             142291.52
```

### ตัวอย่างที่ 2: SQRT ใช้ประกอบการหาส่วนเบี่ยงเบนมาตรฐาน (standard deviation) ด้วยมือ

PostgreSQL มีฟังก์ชัน `STDDEV` สำเร็จรูปอยู่แล้ว (ดู Part 027) แต่การเข้าใจว่า `STDDEV = SQRT(VARIANCE)` ช่วยให้เห็นภาพว่าฟังก์ชันสถิติเหล่านี้คำนวณมาจากอะไร

```sql
SELECT
    category_id,
    ROUND(VARIANCE(unit_price), 2) AS price_variance,
    ROUND(SQRT(VARIANCE(unit_price)), 2) AS price_stddev_manual,
    ROUND(STDDEV(unit_price), 2) AS price_stddev_builtin
FROM products
WHERE category_id = 10
GROUP BY category_id;
```

ผลลัพธ์ตัวอย่าง:

```
 category_id | price_variance | price_stddev_manual | price_stddev_builtin
--------------+------------------+-----------------------+-------------------------
           10 |        5364075.00 |              2316.05 |               2316.05
```

ค่าทั้งสองคอลัมน์ท้ายเท่ากันเสมอ เพราะ `STDDEV` ภายในก็คือ `SQRT(VARIANCE(...))` นั่นเอง

### ตัวอย่างที่ 3: CBRT คำนวณความยาวด้านของลังไม้ (cube) จากปริมาตรที่ต้องใช้

```sql
SELECT
    product_name,
    stock_quantity,
    -- สมมติสินค้าต้องบรรจุในลังทรงลูกบาศก์ ปริมาตรรวม = stock_quantity หน่วยลูกบาศก์
    ROUND(CBRT(stock_quantity)::numeric, 2) AS approx_cube_side
FROM products
WHERE product_id IN (13, 12, 3);
```

ผลลัพธ์ตัวอย่าง:

```
      product_name      | stock_quantity | approx_cube_side
--------------------------+----------------+-------------------
 Modern Sofa 3-Seater     |             10 |             2.15
 Oak Dining Table         |             15 |             2.47
 GameMaster Laptop 15"    |             20 |             2.71
```

### ตัวอย่างที่ 4: EXP และ LN จำลองการเติบโตแบบต่อเนื่อง (continuous growth)

```sql
-- ถ้าฐานลูกค้าตอนนี้คือ 15 คน เติบโตแบบต่อเนื่องด้วยอัตรา 5% ต่อเดือน
-- จำนวนลูกค้าโดยประมาณหลังผ่านไป t เดือน = 15 * EXP(0.05 * t)
SELECT
    t AS months,
    ROUND(15 * EXP(0.05 * t), 1) AS projected_customers
FROM generate_series(0, 12, 3) AS t;
```

ผลลัพธ์ตัวอย่าง:

```
 months | projected_customers
---------+------------------------
       0 |                 15.0
       3 |                 17.4
       6 |                 20.2
       9 |                 23.6
      12 |                 27.3
```

`LN` คือฟังก์ชันผกผันของ `EXP` เช่น หาว่าต้องใช้เวลากี่เดือนกว่าฐานลูกค้าจะถึง 30 คน:

```sql
SELECT LN(30.0 / 15) / 0.05 AS months_to_reach_30;
```

ผลลัพธ์ตัวอย่าง:

```
 months_to_reach_30
----------------------
    13.86294361119891
```

### ตัวอย่างที่ 5: LOG หาลำดับขนาด (order of magnitude) ของยอดขาย

```sql
SELECT
    o.order_id,
    p.amount AS order_total,
    ROUND(LOG(p.amount), 2) AS log10_magnitude
FROM payments p
JOIN orders o ON o.order_id = p.order_id
ORDER BY p.amount DESC
LIMIT 5;
```

ผลลัพธ์ตัวอย่าง:

```
 order_id | order_total | log10_magnitude
-----------+---------------+-------------------
         9 |     45990.00 |            4.66
        16 |     43980.00 |            4.64
         1 |     38970.00 |            4.59
        14 |     32990.00 |            4.52
        11 |     30970.00 |            4.49
```

รูปแบบ `LOG(base, x)` สองพารามิเตอร์ก็ใช้ได้เช่นกัน เช่น `LOG(2, 8) = 3` สำหรับงานที่ต้องแปลงหน่วยเป็นฐาน 2 (ไบต์คอมพิวเตอร์) หรือฐานอื่น ๆ

---

## Step 324: Random Number Generation — RANDOM(), การสุ่มในช่วงที่กำหนด, การสุ่มแถว และ Performance

`RANDOM()` คืนค่า `double precision` แบบสุ่มในช่วง `[0, 1)` ทุกครั้งที่เรียกจะได้ค่าใหม่ (เป็น volatile function)

### ตัวอย่างที่ 1: การสุ่มตัวเลขในช่วงที่กำหนด

สูตรมาตรฐานคือ `FLOOR(RANDOM() * (max - min + 1)) + min` สำหรับจำนวนเต็ม

```sql
SELECT
    generate_series AS run_no,
    FLOOR(RANDOM() * (100 - 1 + 1) + 1)::int AS random_int_1_to_100
FROM generate_series(1, 5);
```

ผลลัพธ์ตัวอย่าง (ค่าจะเปลี่ยนทุกครั้งที่รัน):

```
 run_no | random_int_1_to_100
---------+------------------------
      1 |                    47
      2 |                    12
      3 |                    88
      4 |                     3
      5 |                    61
```

สำหรับตัวเลขทศนิยมในช่วงที่กำหนด เช่น ราคาสุ่มระหว่าง 100.00 - 500.00 บาท:

```sql
SELECT ROUND((RANDOM() * (500 - 100) + 100)::numeric, 2) AS random_price
FROM generate_series(1, 3);
```

### ตัวอย่างที่ 2: การสุ่มแถวข้อมูลด้วย ORDER BY RANDOM() LIMIT n

```sql
SELECT product_name, unit_price
FROM products
WHERE is_active = true
ORDER BY RANDOM()
LIMIT 3;
```

ผลลัพธ์ตัวอย่าง (สุ่มได้ต่างกันทุกครั้ง):

```
      product_name       | unit_price
---------------------------+-------------
 Air Purifier Max          |    4990.00
 Women's Silk Blouse       |    1590.00
 UltraBook Pro 14"         |   32990.00
```

### ข้อควรระวังด้าน Performance ของ ORDER BY RANDOM()

`ORDER BY RANDOM() LIMIT n` มีปัญหาสำคัญ: PostgreSQL ต้องคำนวณค่าสุ่มให้ **ทุกแถว** ในตาราง แล้ว sort ทั้งหมดก่อนจะเลือกแค่ n แถวแรก บนตารางเล็ก ๆ อย่าง `products` ในบทเรียนนี้ (20 แถว) ไม่มีปัญหา แต่บนตารางระดับล้านแถวขึ้นไป วิธีนี้จะช้ามากเพราะเป็น full table scan + full sort ทุกครั้ง

แนวทางที่ดีกว่าสำหรับตารางใหญ่:

**วิธีที่ 1 — `TABLESAMPLE` (เร็วมาก แต่ไม่สุ่มแบบสม่ำเสมอเป๊ะ เหมาะกับการสุ่มตัวอย่างโดยประมาณ)**

```sql
SELECT product_name, unit_price
FROM products TABLESAMPLE BERNOULLI (20)  -- สุ่มประมาณ 20% ของแถว
LIMIT 3;
```

`TABLESAMPLE SYSTEM (n)` จะเร็วกว่า `BERNOULLI` เพราะสุ่มเลือกทั้ง page แทนที่จะสุ่มทีละแถว แต่กระจายตัวไม่สม่ำเสมอเท่า

**วิธีที่ 2 — สุ่มจาก primary key range (เหมาะกับตารางที่ id ต่อเนื่องและไม่มีช่องว่างมาก)**

```sql
SELECT p.product_name, p.unit_price
FROM products p
WHERE p.product_id >= (
    SELECT FLOOR(RANDOM() * (SELECT MAX(product_id) FROM products))::int
)
ORDER BY p.product_id
LIMIT 3;
```

**วิธีที่ 3 — `setseed()` เพื่อให้ผลลัพธ์สุ่มซ้ำได้ (reproducible)** มีประโยชน์มากตอนเขียน automated test หรือ demo ที่ต้องการผลลัพธ์เดิมทุกครั้ง

```sql
SELECT setseed(0.42);
SELECT product_name FROM products ORDER BY RANDOM() LIMIT 3;
```

เมื่อ `setseed()` ถูกตั้งค่าเดิมก่อนรันคำสั่งเดิม ผลลัพธ์จาก `RANDOM()` ในเซสชันนั้นจะเหมือนเดิมทุกครั้ง (ค่าที่ตั้งมีผลเฉพาะ session ปัจจุบัน)

---

## Step 325: ROUND กับทศนิยมและการปัดเศษเงิน — Bankers Rounding เทียบกับ Standard Rounding

### พฤติกรรมการปัดเศษของ ROUND บน numeric

PostgreSQL ปัด `ROUND` บนชนิด `numeric` ด้วยกฎ **"round half away from zero"** (ปัดครึ่งออกจากศูนย์ หรือที่เรียกกันว่า "arithmetic rounding" / "commercial rounding") ซึ่งต่างจาก **"round half to even"** หรือ **bankers rounding** ที่บางภาษาโปรแกรม (เช่น Python 3 `round()`, IEEE 754 default) ใช้

```sql
SELECT
    ROUND(2.5::numeric)  AS round_2_5,   -- 3  (ปัดออกจากศูนย์)
    ROUND(3.5::numeric)  AS round_3_5,   -- 4  (ปัดออกจากศูนย์)
    ROUND(-2.5::numeric) AS round_neg_2_5; -- -3 (ปัดออกจากศูนย์ ไปทางลบ)
```

ผลลัพธ์ตัวอย่าง:

```
 round_2_5 | round_3_5 | round_neg_2_5
------------+------------+-----------------
         3 |          4 |             -3
```

ถ้าเป็น bankers rounding (round half to even) แบบที่ Python ใช้ `ROUND(2.5)` จะได้ `2` (ปัดเข้าเลขคู่ที่ใกล้ที่สุด) และ `ROUND(3.5)` จะได้ `4` เช่นกัน (เข้าเลขคู่) — ผลต่างนี้สำคัญมากเวลาย้ายระบบคำนวณจากภาษาอื่นมา PostgreSQL หรือเทียบผลลัพธ์ระหว่างระบบ

### ROUND บน numeric เทียบกับ double precision

ข้อควรระวังอีกจุดคือ `ROUND(double precision)` ในบางเวอร์ชันของ PostgreSQL **ไม่รองรับพารามิเตอร์ตัวที่สอง** (จำนวนตำแหน่งทศนิยม) เพราะ `ROUND(dp, int)` ไม่มีอยู่ในระบบ — ต้อง cast เป็น `numeric` ก่อนเสมอถ้าต้องการปัดทศนิยมหลายตำแหน่ง

```sql
-- ตัวอย่างที่ทำให้เกิด error เพราะ 3.14159::float8 ไม่มี ROUND(dp, int)
-- SELECT ROUND(3.14159::float8, 2);  -- ERROR: function round(double precision, integer) does not exist

-- วิธีที่ถูกต้อง: cast เป็น numeric ก่อน
SELECT ROUND(3.14159::numeric, 2) AS correct_round;
```

ผลลัพธ์ตัวอย่าง:

```
 correct_round
----------------
           3.14
```

### floating point กับปัญหาความแม่นยำ (สำคัญมากสำหรับเงิน)

การเก็บราคาสินค้าด้วย `NUMERIC(10,2)` ในตาราง `products` ของเราคือแนวปฏิบัติที่ถูกต้อง เพราะ `numeric` เก็บค่าแบบ exact decimal ในขณะที่ `float`/`double precision` เก็บแบบ binary floating point ซึ่งอาจมีความคลาดเคลื่อนเล็กน้อย

```sql
SELECT
    (0.1::float8 + 0.2::float8) AS float_sum,          -- อาจได้ 0.30000000000000004
    (0.1::numeric + 0.2::numeric) AS numeric_sum;        -- ได้ 0.3 พอดี
```

ผลลัพธ์ตัวอย่าง:

```
      float_sum       | numeric_sum
------------------------+---------------
    0.30000000000000004 |         0.3
```

นี่คือเหตุผลว่าทำไมคอลัมน์เงินในระบบจริงทุกระบบ (รวมถึง `unit_price`, `amount` ในบทนี้) ควรเป็น `NUMERIC(p, s)` เสมอ ไม่ใช่ `REAL` หรือ `DOUBLE PRECISION`

### ตัวอย่างประยุกต์: ปัดยอดชำระเงินให้เป็น 2 ตำแหน่งทศนิยมตามมาตรฐานบัญชี

```sql
SELECT
    o.order_id,
    ROUND(SUM(oi.quantity * oi.unit_price), 2) AS subtotal,
    ROUND(SUM(oi.quantity * oi.unit_price) * 1.07, 2) AS total_with_vat
FROM order_items oi
JOIN orders o ON o.order_id = oi.order_id
WHERE o.order_id IN (8, 13)
GROUP BY o.order_id
ORDER BY o.order_id;
```

ผลลัพธ์ตัวอย่าง:

```
 order_id | subtotal | total_with_vat
-----------+------------+------------------
         8 |    4170.00 |        4461.90
        13 |   11920.00 |       12754.40
```

---

## Step 326: Type Casting เชิงลึก — CAST vs `::` vs Constructor Function

PostgreSQL มีวิธีแปลงชนิดข้อมูลอยู่หลายแบบ ซึ่งแต่ละแบบมีจุดแข็งต่างกัน:

| วิธี | ตัวอย่าง | มาตรฐาน | หมายเหตุ |
|---|---|---|---|
| `CAST(x AS type)` | `CAST('123' AS integer)` | SQL standard | พกพาข้ามฐานข้อมูลได้ อ่านง่าย |
| `x::type` | `'123'::integer` | PostgreSQL เฉพาะ | สั้น กระชับ นิยมใช้ในงานจริง |
| Constructor function | `to_number('฿1,234.50', '฿999,999.99')` | PostgreSQL function | ควบคุม format ได้ละเอียด เหมาะกับ text ที่มี format พิเศษ |

### ตัวอย่างที่ 1: CAST และ :: ให้ผลลัพธ์เหมือนกันทุกประการ

```sql
SELECT
    CAST('32990' AS numeric) AS cast_syntax,
    '32990'::numeric AS double_colon_syntax,
    CAST(unit_price AS integer) AS cast_price_int,
    unit_price::integer AS colon_price_int
FROM products
WHERE product_id = 1;
```

ผลลัพธ์ตัวอย่าง:

```
 cast_syntax | double_colon_syntax | cast_price_int | colon_price_int
--------------+------------------------+-------------------+-------------------
        32990 |                  32990 |             32990 |            32990
```

ทั้งสองรูปแบบคอมไพล์ไปเป็น operation เดียวกันในระดับ internal ไม่มีผลต่าง performance เลย ต่างกันแค่ syntax — `CAST` เหมาะกับโค้ดที่ต้องรันบนหลายฐานข้อมูล (portability) ส่วน `::` เป็นที่นิยมในหมู่นักพัฒนา PostgreSQL เพราะสั้นกว่า

### ตัวอย่างที่ 2: to_char แปลงตัวเลขเป็นข้อความจัดรูปแบบสำหรับแสดงผล

```sql
SELECT
    product_name,
    unit_price,
    to_char(unit_price, 'FM999,999,999.00') AS formatted_price,
    to_char(unit_price, 'FM฿999,999,999.00') AS formatted_price_thb
FROM products
WHERE product_id IN (1, 8, 12);
```

ผลลัพธ์ตัวอย่าง:

```
      product_name      | unit_price | formatted_price | formatted_price_thb
--------------------------+------------+--------------------+------------------------
 UltraBook Pro 14"        |   32990.00 |        32,990.00 |            ฿32,990.00
 Men's Cotton Shirt       |     590.00 |           590.00 |               ฿590.00
 Oak Dining Table         |   12990.00 |        12,990.00 |            ฿12,990.00
```

รูปแบบ `FM` ที่นำหน้า template หมายถึง "fill mode" ซึ่งตัดช่องว่างส่วนเกินที่ `to_char` ปกติจะเติมไว้ (padding) ออกไป

### ตัวอย่างที่ 3: to_number แปลงข้อความที่มี format พิเศษกลับเป็นตัวเลข

```sql
SELECT
    to_number('32,990.00', '999,999.99') AS parsed_price,
    to_number('฿4,990.00', 'FM฿999,999.99') AS parsed_price_thb,
    to_number('  1234  ', '9999') AS parsed_with_spaces;
```

ผลลัพธ์ตัวอย่าง:

```
 parsed_price | parsed_price_thb | parsed_with_spaces
---------------+---------------------+-----------------------
      32990.00 |             4990.00 |                 1234
```

`to_number` มีประโยชน์มากเมื่อต้อง import ข้อมูลจากไฟล์ CSV หรือระบบภายนอกที่ส่งตัวเลขมาเป็น text พร้อมสัญลักษณ์สกุลเงินหรือ comma คั่นหลักพัน ซึ่ง `CAST`/`::` ธรรมดาจะแปลงไม่ได้เลย (จะเกิด error ทันที)

```sql
-- ตัวอย่างที่ error เพราะมี comma และสัญลักษณ์เงินปน
-- SELECT '32,990.00'::numeric;  -- ERROR: invalid input syntax for type numeric
```

### เมื่อไรควรใช้อะไร

- ใช้ `::` สำหรับงานทั่วไปในโค้ด PostgreSQL เพราะอ่านง่ายและกระชับ
- ใช้ `CAST(...)` เมื่อเขียน SQL ที่ต้องรันข้ามระบบฐานข้อมูล (portability) หรือทำตามมาตรฐานขององค์กร
- ใช้ `to_number`/`to_char` เมื่อข้อมูลมี format พิเศษ (comma, สัญลักษณ์สกุลเงิน, เว้นวรรค) ที่ `CAST`/`::` จัดการไม่ได้โดยตรง

---

## Step 327: การแปลงระหว่าง numeric/text/integer อย่างปลอดภัย พร้อมดักจับ Error ด้วย Regex

ปัญหาที่พบบ่อยในงานจริงคือข้อมูลจากภายนอก (เช่น import จาก Excel, รับค่าจากฟอร์มเว็บ) มักเป็น text ที่ไม่รับประกันว่าจะแปลงเป็นตัวเลขได้เสมอ การ cast ตรง ๆ โดยไม่ตรวจสอบก่อนจะทำให้ query ทั้งก้อนล้มเหลวทันทีเมื่อเจอข้อมูลเสียแม้เพียงแถวเดียว

### ตัวอย่างปัญหา: cast ข้อมูลเสียทำให้ query ล้มเหลวทั้งชุด

```sql
-- จำลองข้อมูล text ที่มีทั้งตัวเลขปกติและข้อมูลเสีย
CREATE TEMP TABLE raw_import (id INT, raw_value TEXT);
INSERT INTO raw_import (id, raw_value) VALUES
    (1, '32990.00'),
    (2, '24990.50'),
    (3, 'N/A'),
    (4, '45990'),
    (5, ''),
    (6, '6,990.00');

-- คำสั่งนี้จะ ERROR ทันทีที่เจอแถว id=3 ('N/A' ไม่ใช่ตัวเลข)
-- SELECT id, raw_value::numeric FROM raw_import;
```

### วิธีที่ 1: ตรวจสอบด้วย Regular Expression ก่อนแปลง

```sql
SELECT
    id,
    raw_value,
    CASE
        WHEN raw_value ~ '^\s*[0-9]+(\.[0-9]+)?\s*$' THEN raw_value::numeric
        ELSE NULL
    END AS safe_numeric_value
FROM raw_import
ORDER BY id;
```

ผลลัพธ์ตัวอย่าง:

```
 id | raw_value  | safe_numeric_value
-----+-------------+-----------------------
  1 | 32990.00    |            32990.00
  2 | 24990.50    |            24990.50
  3 | N/A         |                 <NULL>
  4 | 45990       |            45990.00
  5 |             |                 <NULL>
  6 | 6,990.00    |                 <NULL>
```

สังเกตว่าแถว id=6 (`6,990.00`) ก็ถูกกรองเป็น `NULL` เช่นกัน เพราะมี comma ปนซึ่งไม่ตรงกับ pattern ที่กำหนด — ถ้าต้องการรองรับ comma คั่นหลักพันด้วย ต้องปรับ regex หรือ `REPLACE(raw_value, ',', '')` ก่อนตรวจสอบ:

```sql
SELECT
    id,
    raw_value,
    CASE
        WHEN REPLACE(raw_value, ',', '') ~ '^\s*[0-9]+(\.[0-9]+)?\s*$'
            THEN REPLACE(raw_value, ',', '')::numeric
        ELSE NULL
    END AS safe_numeric_value
FROM raw_import
ORDER BY id;
```

ผลลัพธ์ตัวอย่าง:

```
 id | raw_value  | safe_numeric_value
-----+-------------+-----------------------
  1 | 32990.00    |            32990.00
  2 | 24990.50    |            24990.50
  3 | N/A         |                 <NULL>
  4 | 45990       |            45990.00
  5 |             |                 <NULL>
  6 | 6,990.00    |             6990.00
```

### วิธีที่ 2: pg_input_is_valid() (PostgreSQL 16 ขึ้นไป)

ตั้งแต่ PostgreSQL 16 มีฟังก์ชัน `pg_input_is_valid(text, type_name)` ที่ตรวจสอบได้ตรงกับ parser ภายในจริง ๆ ของฐานข้อมูล แม่นยำกว่าการเขียน regex เอง (เพราะ regex อาจพลาด edge case เช่น `1e10`, `Infinity`, `NaN` ซึ่งเป็นค่าที่ `numeric`/`float` ยอมรับได้จริง)

```sql
SELECT
    id,
    raw_value,
    pg_input_is_valid(raw_value, 'numeric') AS is_valid_numeric,
    CASE
        WHEN pg_input_is_valid(raw_value, 'numeric') THEN raw_value::numeric
        ELSE NULL
    END AS safe_value
FROM raw_import
ORDER BY id;
```

ผลลัพธ์ตัวอย่าง:

```
 id | raw_value  | is_valid_numeric | safe_value
-----+-------------+---------------------+---------------
  1 | 32990.00    | t                   |     32990.00
  2 | 24990.50    | t                   |     24990.50
  3 | N/A         | f                   |         <NULL>
  4 | 45990       | t                   |     45990.00
  5 |             | f                   |         <NULL>
  6 | 6,990.00    | f                   |         <NULL>
```

`pg_input_is_valid` ยังมีฟังก์ชันคู่หูชื่อ `pg_input_error_message(text, type_name)` ที่คืนข้อความ error จริงหากอยากรู้สาเหตุที่แปลงไม่ได้ ซึ่งมีประโยชน์มากสำหรับ log การนำเข้าข้อมูล

### วิธีที่ 3: ห่อด้วยฟังก์ชัน PL/pgSQL และ EXCEPTION (รองรับ PostgreSQL ทุกเวอร์ชัน)

```sql
CREATE OR REPLACE FUNCTION safe_to_numeric(input_text TEXT)
RETURNS NUMERIC AS $$
BEGIN
    RETURN input_text::numeric;
EXCEPTION
    WHEN invalid_text_representation THEN
        RETURN NULL;
END;
$$ LANGUAGE plpgsql IMMUTABLE;

SELECT id, raw_value, safe_to_numeric(raw_value) AS safe_value
FROM raw_import
ORDER BY id;
```

ผลลัพธ์ตัวอย่าง:

```
 id | raw_value  | safe_value
-----+-------------+---------------
  1 | 32990.00    |     32990.00
  2 | 24990.50    |     24990.50
  3 | N/A         |         <NULL>
  4 | 45990       |     45990.00
  5 |             |         <NULL>
  6 | 6,990.00    |         <NULL>
```

วิธีนี้ครอบคลุมทุกกรณีเพราะจับ error จริงจาก parser แต่มีค่าใช้จ่าย (overhead) ของการเรียกฟังก์ชันต่อแถวสูงกว่าการตรวจ regex หรือ `pg_input_is_valid` ตรง ๆ ในกรณีที่ต้องประมวลผลข้อมูลจำนวนมาก แนะนำให้ใช้ `pg_input_is_valid` (PG16+) เป็นตัวเลือกแรกเสมอ

---

## Step 328: Sequence-related Functions — NEXTVAL, CURRVAL, SETVAL (ทบทวนเชื่อมโยงกับ Part 039)

ทุกตารางในชุดข้อมูลนี้ใช้ `SERIAL` ซึ่งภายในคือ integer ที่ผูกกับ sequence object โดยอัตโนมัติ (เช่น `products_product_id_seq` สำหรับตาราง `products`) บทนี้ทบทวนฟังก์ชันสามตัวหลักที่ใช้ควบคุม sequence โดยตรง ส่วนรายละเอียดเชิงลึกเรื่อง identity columns, generated columns และกลยุทธ์การออกแบบ primary key จะอยู่ใน **Part 039**

### NEXTVAL — ดึงค่าถัดไปจาก sequence (และเพิ่มค่าไปด้วย)

```sql
SELECT nextval('products_product_id_seq');
```

ผลลัพธ์ตัวอย่าง (ต่อจากที่เรา `setval` ไว้ที่ 20 ในขั้นเตรียมข้อมูล):

```
 nextval
----------
      21
```

ทุกครั้งที่เรียก `nextval` ค่าจะเพิ่มขึ้นถาวร (ไม่สามารถ rollback ได้แม้ transaction ที่เรียกจะถูก ROLLBACK ก็ตาม เพราะ sequence เป็น non-transactional object) นี่คือเหตุผลที่เลข id จาก `SERIAL` อาจมีช่องว่าง (gap) ได้ในสถานการณ์จริง เช่น เมื่อ insert ล้มเหลวกลางคัน

### CURRVAL — ดูค่าล่าสุดที่ session ปัจจุบัน "เรียก" ไปแล้วจาก sequence นั้น

```sql
SELECT currval('products_product_id_seq');
```

ผลลัพธ์ตัวอย่าง:

```
 currval
----------
      21
```

ข้อควรระวังสำคัญ: `currval()` จะ error ทันทีหาก session ปัจจุบันยังไม่เคยเรียก `nextval()` จาก sequence นั้นมาก่อนเลย (error: `currval of sequence "..." is not yet defined in this session`) มันไม่ใช่ฟังก์ชันที่ดู "ค่าปัจจุบันของ sequence โดยรวม" แต่เป็น "ค่าล่าสุดที่ session นี้ดึงไป" เท่านั้น

### SETVAL — ตั้งค่า sequence ใหม่โดยตรง

รูปแบบมีสองแบบ:

```sql
-- แบบที่ 1: is_called = true (ค่าเริ่มต้น) หมายความว่า nextval ครั้งถัดไปจะได้ n+1
SELECT setval('products_product_id_seq', 30);
SELECT nextval('products_product_id_seq'); -- จะได้ 31

-- แบบที่ 2: is_called = false หมายความว่า nextval ครั้งถัดไปจะได้ n พอดี
SELECT setval('products_product_id_seq', 30, false);
SELECT nextval('products_product_id_seq'); -- จะได้ 30 พอดี
```

### กรณีใช้งานจริงที่พบบ่อย: ซิงก์ sequence หลัง import ข้อมูลแบบระบุ id ตรง ๆ

นี่คือสิ่งที่เราทำไปแล้วในขั้นเตรียมข้อมูลของบทนี้ — เมื่อ insert แถวโดยระบุ primary key เอง (ไม่ปล่อยให้ `DEFAULT` จาก sequence ทำงาน) sequence จะไม่รู้ว่ามีการใช้ id ไปแล้ว จึงต้องซิงก์ด้วยมือ ไม่เช่นนั้นการ insert ครั้งถัดไปแบบไม่ระบุ id จะพยายามใช้ id ที่ชนกับของเดิม

```sql
-- รูปแบบมาตรฐานสำหรับซิงก์ sequence ให้ตรงกับค่าสูงสุดในตารางจริง
SELECT setval(
    pg_get_serial_sequence('orders', 'order_id'),
    (SELECT COALESCE(MAX(order_id), 1) FROM orders)
);
```

`pg_get_serial_sequence(table, column)` คืนชื่อ sequence เต็มที่ผูกกับคอลัมน์นั้น ทำให้ไม่ต้องจำชื่อ sequence เอง (ซึ่งบางครั้งถูกตั้งชื่อไม่ตรง pattern มาตรฐานถ้ามีการ rename ตารางหรือคอลัมน์มาก่อน) — เทคนิคนี้ปลอดภัยกว่าการพิมพ์ชื่อ sequence ตรง ๆ และเป็นแนวทางที่แนะนำเสมอในงานจริง

หัวข้อเรื่อง `GENERATED ALWAYS AS IDENTITY` (ทางเลือกใหม่กว่า `SERIAL` ตั้งแต่ PostgreSQL 10) การกำหนด `START WITH`/`INCREMENT BY`/`CACHE`/`CYCLE` และกลยุทธ์เลือกใช้ระหว่าง sequence-based key กับ UUID จะอธิบายละเอียดใน **Part 039: Sequences, Identity Columns และ Primary Key Strategies**

---

## Step 329: Aggregate ทางสถิติเพิ่มเติม — CORR, COVAR_POP/SAMP, REGR_* Functions

PostgreSQL มี aggregate function สำหรับสถิติเชิงสองตัวแปร (bivariate statistics) ในตัว ซึ่งมีประโยชน์มากสำหรับงาน data analysis เบื้องต้นโดยไม่ต้องดึงข้อมูลออกไปคำนวณด้วยเครื่องมืออื่น ฟังก์ชันกลุ่มนี้ทั้งหมดรับพารามิเตอร์เป็น `(Y, X)` ตามลำดับ (ตัวแปรตาม, ตัวแปรอิสระ) ซึ่งเป็นข้อตกลงตามมาตรฐาน SQL สำหรับการวิเคราะห์ regression

| ฟังก์ชัน | ความหมาย |
|---|---|
| `CORR(Y, X)` | สัมประสิทธิ์สหสัมพันธ์ (correlation coefficient) ค่าอยู่ระหว่าง -1 ถึง 1 |
| `COVAR_POP(Y, X)` | ความแปรปรวนร่วมของประชากร (population covariance) |
| `COVAR_SAMP(Y, X)` | ความแปรปรวนร่วมของกลุ่มตัวอย่าง (sample covariance) |
| `REGR_SLOPE(Y, X)` | ความชันของเส้น linear regression |
| `REGR_INTERCEPT(Y, X)` | จุดตัดแกน Y ของเส้น regression |
| `REGR_R2(Y, X)` | ค่า R-squared (ความสามารถในการอธิบายความแปรผัน) |
| `REGR_COUNT(Y, X)` | จำนวนคู่ข้อมูล (Y, X) ที่ไม่เป็น NULL ทั้งคู่ |

### ตัวอย่างที่ 1: สหสัมพันธ์ระหว่างราคาสินค้ากับปริมาณที่ขายได้ต่อออเดอร์

คำถามธุรกิจ: สินค้าที่ราคาแพงขึ้น มักถูกซื้อในปริมาณต่อครั้งที่น้อยลงหรือไม่?

```sql
SELECT
    ROUND(CORR(oi.quantity, oi.unit_price)::numeric, 4) AS corr_qty_price,
    ROUND(COVAR_POP(oi.quantity, oi.unit_price)::numeric, 2) AS covar_pop,
    ROUND(COVAR_SAMP(oi.quantity, oi.unit_price)::numeric, 2) AS covar_samp,
    REGR_COUNT(oi.quantity, oi.unit_price) AS data_points
FROM order_items oi;
```

ผลลัพธ์ตัวอย่าง (ค่าที่แสดงเป็นตัวอย่างประกอบแนวคิด อาจต่างจากการรันจริงเล็กน้อยขึ้นกับชุดข้อมูล):

```
 corr_qty_price | covar_pop | covar_samp | data_points
------------------+-------------+--------------+---------------
         -0.3126 |  -4521.35   |  -4677.47    |          30
```

ค่า `corr_qty_price` ที่เป็นลบและมีขนาดปานกลาง (ประมาณ -0.31) บ่งชี้ว่ามีแนวโน้มอ่อน ๆ ที่สินค้าราคาสูงจะถูกซื้อในปริมาณต่อรายการที่น้อยกว่า ซึ่งสอดคล้องกับสามัญสำนึกทางธุรกิจ — คนไม่ค่อยซื้อโน้ตบุ๊กราคาสามหมื่นทีละหลายเครื่อง แต่จะซื้อเสื้อยืดราคาหลักร้อยทีละหลายตัวได้

### ตัวอย่างที่ 2: Linear Regression หาสมการทำนายจำนวนที่ขายจากราคา

```sql
SELECT
    ROUND(REGR_SLOPE(oi.quantity, oi.unit_price)::numeric, 6) AS slope,
    ROUND(REGR_INTERCEPT(oi.quantity, oi.unit_price)::numeric, 4) AS intercept,
    ROUND(REGR_R2(oi.quantity, oi.unit_price)::numeric, 4) AS r_squared
FROM order_items oi;
```

ผลลัพธ์ตัวอย่าง:

```
 slope    | intercept | r_squared
-----------+-------------+-------------
 -0.000018 |      1.7842 |    0.0977
```

จากสมการ `quantity ≈ intercept + slope * unit_price` เราสามารถประมาณคร่าว ๆ ได้ว่าสินค้าราคา 10,000 บาท มักถูกซื้อเฉลี่ยประมาณ `1.7842 + (-0.000018 * 10000) ≈ 1.6` ชิ้นต่อรายการ ส่วนค่า `r_squared` ที่ต่ำ (ประมาณ 0.10) บอกว่าราคาสินค้าเพียงอย่างเดียวอธิบายความแปรผันของปริมาณการซื้อได้แค่ส่วนน้อย ยังมีปัจจัยอื่นที่มีอิทธิพลมากกว่า เช่น ประเภทสินค้าหรือพฤติกรรมลูกค้า

### ตัวอย่างที่ 3: สหสัมพันธ์ระหว่างราคาสินค้ากับคะแนนรีวิวเฉลี่ย

```sql
SELECT
    ROUND(CORR(r.rating, p.unit_price)::numeric, 4) AS corr_rating_price
FROM reviews r
JOIN products p ON p.product_id = r.product_id;
```

ผลลัพธ์ตัวอย่าง:

```
 corr_rating_price
----------------------
             0.1842
```

ค่าที่ใกล้ 0 บอกว่าราคาสินค้ากับความพึงพอใจของลูกค้า (คะแนนรีวิว) ในชุดข้อมูลตัวอย่างนี้แทบไม่มีความสัมพันธ์เชิงเส้นกัน — ข้อสรุปแบบนี้เป็นจุดตั้งต้นที่ดีสำหรับทีมธุรกิจในการตั้งสมมติฐานเพิ่มเติม เช่น อาจต้องแยกวิเคราะห์เป็นรายหมวดหมู่สินค้าแทนที่จะดูภาพรวมทั้งร้าน

ข้อควรระวัง: ฟังก์ชันกลุ่มนี้ทั้งหมดรับ/คืนค่าเป็น `double precision` ไม่ใช่ `numeric` จึงมักต้อง cast ผลลัพธ์เป็น `numeric` ก่อน `ROUND` เสมอ (ตามที่ทำในตัวอย่างข้างต้นด้วย `::numeric`) และควรระวังกรณีข้อมูลมีจำนวนคู่ (Y, X) น้อยกว่า 2 คู่ เพราะฟังก์ชันเหล่านี้จะคืนค่า `NULL` แทนที่จะ error

---

## Step 330: แบบฝึกหัดรวม — คำนวณตัวเลขธุรกิจจริง

บทนี้จะรวมฟังก์ชันทั้งหมดที่เรียนมาเข้าด้วยกัน เพื่อแก้โจทย์ธุรกิจสี่แบบที่พบบ่อยที่สุดในระบบร้านค้า: **ส่วนลด**, **VAT**, **ค่าคอมมิชชั่นพนักงานขาย** และ **การปัดราคาให้ลงท้ายด้วย .99**

### 1. คำนวณส่วนลดแบบขั้นบันได (tiered discount)

กฎธุรกิจ: ยอดซื้อต่อออเดอร์เกิน 30,000 บาท ลด 10%, เกิน 15,000 บาท ลด 5%, นอกนั้นไม่ลด

```sql
SELECT
    o.order_id,
    subtotal.amount AS subtotal,
    CASE
        WHEN subtotal.amount > 30000 THEN 0.10
        WHEN subtotal.amount > 15000 THEN 0.05
        ELSE 0
    END AS discount_rate,
    ROUND(
        subtotal.amount * (1 - CASE
            WHEN subtotal.amount > 30000 THEN 0.10
            WHEN subtotal.amount > 15000 THEN 0.05
            ELSE 0
        END), 2
    ) AS after_discount
FROM orders o
JOIN (
    SELECT order_id, SUM(quantity * unit_price) AS amount
    FROM order_items
    GROUP BY order_id
) subtotal ON subtotal.order_id = o.order_id
WHERE o.order_id IN (1, 9, 11, 13)
ORDER BY o.order_id;
```

ผลลัพธ์ตัวอย่าง:

```
 order_id | subtotal | discount_rate | after_discount
-----------+------------+------------------+-------------------
         1 |  38970.00 |             0.10 |        35073.00
         9 |  45990.00 |             0.10 |        41391.00
        11 |  30970.00 |             0.10 |        27873.00
        13 |  11920.00 |             0.00 |        11920.00
```

### 2. เพิ่ม VAT 7% หลังหักส่วนลด

```sql
SELECT
    o.order_id,
    subtotal.amount AS subtotal,
    ROUND(subtotal.amount * 0.95, 2) AS after_discount_5pct,
    ROUND(subtotal.amount * 0.95 * 1.07, 2) AS total_with_vat
FROM orders o
JOIN (
    SELECT order_id, SUM(quantity * unit_price) AS amount
    FROM order_items
    GROUP BY order_id
) subtotal ON subtotal.order_id = o.order_id
WHERE o.order_id = 13;
```

ผลลัพธ์ตัวอย่าง:

```
 order_id | subtotal | after_discount_5pct | total_with_vat
-----------+------------+------------------------+------------------
        13 |  11920.00 |              11324.00 |        12116.68
```

### 3. คำนวณค่าคอมมิชชั่นพนักงานขายแบบขั้นบันไดตามยอดขายรวม

กฎธุรกิจ: พนักงานที่ยอดขายรวม (จากออเดอร์ที่ `status = 'completed'`) เกิน 50,000 บาท ได้คอมมิชชั่น 5% ของยอดขายทั้งหมด ส่วนที่ต่ำกว่านั้นได้ 3%

```sql
SELECT
    e.employee_id,
    e.first_name || ' ' || e.last_name AS employee_name,
    ROUND(sales.total_sales, 2) AS total_sales,
    CASE WHEN sales.total_sales > 50000 THEN 0.05 ELSE 0.03 END AS commission_rate,
    ROUND(
        sales.total_sales * CASE WHEN sales.total_sales > 50000 THEN 0.05 ELSE 0.03 END,
        2
    ) AS commission_amount
FROM employees e
JOIN (
    SELECT o.employee_id, SUM(oi.quantity * oi.unit_price) AS total_sales
    FROM orders o
    JOIN order_items oi ON oi.order_id = o.order_id
    WHERE o.status = 'completed'
    GROUP BY o.employee_id
) sales ON sales.employee_id = e.employee_id
ORDER BY commission_amount DESC;
```

ผลลัพธ์ตัวอย่าง:

```
 employee_id | employee_name    | total_sales | commission_rate | commission_amount
--------------+--------------------+---------------+--------------------+----------------------
            2 | Malee Suksawat     |   96500.00 |               0.05 |            4825.00
            4 | Kanya Thongdee     |   48760.00 |               0.03 |            1462.80
            3 | Chaiwat Rungroj    |   57910.00 |               0.05 |            2895.50
            5 | Anuwat Srisuk      |   13980.00 |               0.03 |             419.40
            6 | Pornthip Wattana   |   13980.00 |               0.03 |             419.40
```

### 4. ปัดราคาให้ลงท้ายด้วย .99 (psychological pricing)

เทคนิคนี้เป็นที่นิยมในธุรกิจค้าปลีก: ปัดราคาขึ้นไปยังจำนวนเต็มถัดไปก่อน แล้วลบ 0.01 เพื่อให้ลงท้ายด้วย `.99` เสมอ

```sql
SELECT
    product_name,
    unit_price,
    CEIL(unit_price) - 0.01 AS price_ending_99
FROM products
WHERE product_id IN (7, 9, 17, 20)
ORDER BY product_id;
```

ผลลัพธ์ตัวอย่าง:

```
      product_name       | unit_price | price_ending_99
---------------------------+------------+-------------------
 Wireless Earbuds Pro      |    2990.00 |         2999.99
 Men's Denim Jeans         |     890.00 |          899.99
 Microwave Oven 900W       |    2990.00 |         2999.99
 Portable BBQ Grill        |    2490.00 |         2499.99
```

สังเกตว่า `CEIL(2990.00)` ให้ผลลัพธ์เป็น `2990` พอดี (เพราะเป็นจำนวนเต็มอยู่แล้ว) ลบ 0.01 จึงได้ `2989.99` ซึ่งอาจไม่ใช่สิ่งที่ธุรกิจต้องการ (อยากได้ `2999.99` เพื่อ "ปัดขึ้น" ไปอีกระดับหนึ่ง) ในทางปฏิบัติจึงมักใช้สูตรที่ปัดขึ้นไปอีกหลักสิบหรือหลักร้อยก่อนเสมอ:

```sql
SELECT
    product_name,
    unit_price,
    (CEIL(unit_price / 100) * 100) - 0.01 AS price_ending_99_rounded_up_hundred
FROM products
WHERE product_id IN (7, 9, 17, 20)
ORDER BY product_id;
```

ผลลัพธ์ตัวอย่าง:

```
      product_name       | unit_price | price_ending_99_rounded_up_hundred
---------------------------+------------+---------------------------------------
 Wireless Earbuds Pro      |    2990.00 |                            2999.99
 Men's Denim Jeans         |     890.00 |                             899.99
 Microwave Oven 900W       |    2990.00 |                            2999.99
 Portable BBQ Grill        |    2490.00 |                            2499.99
```

สูตรนี้ปัด `unit_price` ขึ้นไปยังหลักร้อยถัดไปก่อน (เช่น 2990 → 3000) แล้วลบ 0.01 ออก ทำให้ได้ `2999.99` ซึ่งเป็นราคาที่สูงกว่าราคาต้นทางเสมอ (ไม่มีทางได้ราคาต่ำกว่าราคาตั้งต้น) เหมาะสำหรับใช้เป็นราคาตั้งโปรโมชั่นแบบ "จิตวิทยาราคา"

---

## สรุปท้ายบท

ในบทนี้เราครอบคลุมฟังก์ชันตัวเลขและการแปลงชนิดข้อมูลของ PostgreSQL อย่างครบถ้วน ตั้งแต่ระดับพื้นฐานไปจนถึงระดับที่ใช้งานได้จริงในธุรกิจ:

- **ฟังก์ชันคณิตศาสตร์พื้นฐาน** (`ROUND`, `CEIL`, `FLOOR`, `TRUNC`, `ABS`, `SIGN`) คือเครื่องมือที่ใช้บ่อยที่สุดในงานรายงานและการคำนวณราคา
- **MOD/`%`/DIV** ช่วยจัดการการหารเอาเศษและการแบ่งกลุ่มข้อมูลแบบ deterministic
- **POWER, SQRT, CBRT, EXP, LN, LOG** เป็นรากฐานของการคำนวณเชิงการเงินและสถิติ เช่น การเติบโตทบต้นและการวัดการกระจายตัว
- **RANDOM()** มีประโยชน์มากแต่ต้องระวังเรื่อง performance บนตารางขนาดใหญ่ — `TABLESAMPLE` คือทางเลือกที่ปลอดภัยกว่า `ORDER BY RANDOM()`
- **การปัดเศษ** ใน PostgreSQL ใช้กฎ round half away from zero ไม่ใช่ bankers rounding และควรใช้ `numeric` เสมอสำหรับข้อมูลเงิน ไม่ใช่ `float`/`double precision`
- **CAST/`::`/to_number/to_char** แต่ละแบบมีจุดแข็งต่างกัน — `::` สำหรับงานทั่วไป, `CAST` สำหรับ portability, `to_number`/`to_char` สำหรับข้อมูลที่มี format พิเศษ
- **การแปลงข้อมูลอย่างปลอดภัย** ควรตรวจสอบก่อนแปลงเสมอ ด้วย regex, `pg_input_is_valid` (PG16+), หรือฟังก์ชัน wrapper ที่ดักจับ exception
- **NEXTVAL/CURRVAL/SETVAL** คือเครื่องมือควบคุม sequence โดยตรง สำคัญมากเมื่อต้อง import ข้อมูลแบบระบุ id เอง
- **CORR, COVAR_POP/SAMP, REGR_\*** เปิดทางให้ทำ data analysis เบื้องต้นได้ตรงในฐานข้อมูล โดยไม่ต้องส่งข้อมูลออกไปเครื่องมืออื่น
- การผสมผสานฟังก์ชันเหล่านี้เข้าด้วยกันคือหัวใจของการคำนวณโจทย์ธุรกิจจริง เช่น ส่วนลด, VAT, ค่าคอมมิชชั่น และกลยุทธ์การตั้งราคา

ในบทถัดไป เราจะเปลี่ยนโฟกัสไปที่ **Conditional Logic** — การใช้ `CASE`, `COALESCE`, `NULLIF`, และฟังก์ชันเงื่อนไขอื่น ๆ เพื่อสร้างตรรกะทางธุรกิจที่ซับซ้อนขึ้นภายใน SQL โดยตรง

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1

หาสินค้าทั้งหมดที่ `unit_price` เมื่อปัดเศษเป็นหลักพัน (`ROUND(unit_price, -3)`) แล้วมีค่ามากกว่าหรือเท่ากับ 20,000 บาท เรียงจากมากไปน้อย

<details>
<summary>เฉลย</summary>

```sql
SELECT
    product_name,
    unit_price,
    ROUND(unit_price, -3) AS rounded_thousand
FROM products
WHERE ROUND(unit_price, -3) >= 20000
ORDER BY unit_price DESC;
```

ผลลัพธ์ตัวอย่าง:

```
      product_name      | unit_price | rounded_thousand
--------------------------+------------+---------------------
 GameMaster Laptop 15"    |   45990.00 |            46000.00
 UltraBook Pro 14"        |   32990.00 |            33000.00
 UltraBook Air 13"        |   24990.00 |            25000.00
 SmartPhone X12           |   21990.00 |            22000.00
```

</details>

### แบบฝึกหัดที่ 2

คำนวณจำนวนกล่อง (แต่ละกล่องบรรจุ 25 ชิ้น) ที่ต้องใช้สำหรับสต็อกของสินค้าในหมวด `category_id = 1` (Electronics ระดับบนสุด ไม่รวมหมวดย่อย) โดยใช้ `CEIL`

<details>
<summary>เฉลย</summary>

```sql
SELECT
    product_name,
    stock_quantity,
    CEIL(stock_quantity / 25.0) AS boxes_needed
FROM products
WHERE category_id = 1
ORDER BY product_id;
```

ผลลัพธ์ตัวอย่าง:

```
      product_name       | stock_quantity | boxes_needed
---------------------------+----------------+---------------
 Wireless Earbuds Pro      |            200 |             8
 Outdoor Camping Tent      |             40 |             2
 Portable BBQ Grill        |             30 |             2
```

</details>

### แบบฝึกหัดที่ 3

ใช้ `MOD` แบ่งพนักงานทั้งหมดออกเป็น 2 กะทำงาน (กะเช้า/กะบ่าย) โดยใช้ `employee_id % 2` (0 = กะเช้า, 1 = กะบ่าย)

<details>
<summary>เฉลย</summary>

```sql
SELECT
    employee_id,
    first_name || ' ' || last_name AS employee_name,
    CASE WHEN MOD(employee_id, 2) = 0 THEN 'กะเช้า' ELSE 'กะบ่าย' END AS shift
FROM employees
ORDER BY employee_id;
```

ผลลัพธ์ตัวอย่าง:

```
 employee_id | employee_name       | shift
--------------+-----------------------+----------
            1 | Prasert Kittikul      | กะบ่าย
            2 | Malee Suksawat        | กะเช้า
            3 | Chaiwat Rungroj       | กะบ่าย
            4 | Kanya Thongdee        | กะเช้า
            5 | Anuwat Srisuk         | กะบ่าย
            ...
```

</details>

### แบบฝึกหัดที่ 4

ใช้ `POWER` คำนวณว่าถ้ายอดขายรวมทั้งร้าน (จาก `payments.amount`) เติบโตปีละ 12% แบบทบต้น อีก 5 ปีข้างหน้ายอดขายจะเป็นเท่าไร (ปัดเศษ 2 ตำแหน่ง)

<details>
<summary>เฉลย</summary>

```sql
SELECT
    ROUND(SUM(amount), 2) AS current_total,
    ROUND(SUM(amount) * POWER(1.12, 5), 2) AS projected_5y
FROM payments;
```

ผลลัพธ์ตัวอย่าง:

```
 current_total | projected_5y
------------------+----------------
       381030.00 |    671542.58
```

</details>

### แบบฝึกหัดที่ 5

เขียนคำสั่งสุ่มเลือกลูกค้า 3 คนจากตาราง `customers` ด้วย `ORDER BY RANDOM() LIMIT 3` จากนั้นอธิบายว่าทำไมวิธีนี้ถึงไม่เหมาะกับตารางลูกค้าที่มีหลายล้านแถว และเสนอทางเลือกอื่น

<details>
<summary>เฉลย</summary>

```sql
SELECT customer_id, first_name, last_name
FROM customers
ORDER BY RANDOM()
LIMIT 3;
```

เหตุผลที่ไม่เหมาะกับตารางขนาดใหญ่: `ORDER BY RANDOM()` ต้องคำนวณค่าสุ่มให้ **ทุกแถว** ในตาราง แล้วนำมาเรียงลำดับ (sort) ทั้งหมดก่อนจะตัดเอาแค่ 3 แถวแรก ซึ่งเป็นการทำ full table scan บวก full sort แม้จะต้องการผลลัพธ์แค่ไม่กี่แถว บนตารางระดับล้านแถวจะช้ามากและกิน CPU/memory สูง

ทางเลือกที่ดีกว่า:

```sql
-- ใช้ TABLESAMPLE เพื่อสุ่มแบบประมาณโดยไม่ต้องสแกนทั้งตาราง
SELECT customer_id, first_name, last_name
FROM customers TABLESAMPLE SYSTEM (5)
LIMIT 3;
```

</details>

### แบบฝึกหัดที่ 6

จงพิสูจน์ด้วย query ว่า `ROUND` บน `numeric` ใน PostgreSQL ใช้กฎ "round half away from zero" ไม่ใช่ bankers rounding โดยทดสอบกับค่า 0.5, 1.5, 2.5, 3.5

<details>
<summary>เฉลย</summary>

```sql
SELECT
    ROUND(0.5::numeric) AS r_0_5,
    ROUND(1.5::numeric) AS r_1_5,
    ROUND(2.5::numeric) AS r_2_5,
    ROUND(3.5::numeric) AS r_3_5;
```

ผลลัพธ์ตัวอย่าง:

```
 r_0_5 | r_1_5 | r_2_5 | r_3_5
--------+--------+--------+--------
      1 |      2 |      3 |      4
```

ถ้าเป็น bankers rounding (round half to even) ผลลัพธ์ควรเป็น `0, 2, 2, 4` (ปัดเข้าเลขคู่ที่ใกล้ที่สุดเสมอ) แต่ผลจริงคือ `1, 2, 3, 4` ซึ่งทุกค่า `.5` ถูกปัดขึ้น (ออกจากศูนย์) เสมอ ยืนยันว่า PostgreSQL ใช้กฎ round half away from zero

</details>

### แบบฝึกหัดที่ 7

ตาราง `raw_import` ด้านล่างมีข้อมูลราคาที่อาจไม่ถูกต้อง จงเขียน query ที่แปลงเฉพาะแถวที่ถูกต้องเป็น `numeric` และส่งคืน `NULL` สำหรับแถวที่แปลงไม่ได้ โดยใช้ `pg_input_is_valid`

```sql
CREATE TEMP TABLE raw_import2 (id INT, raw_value TEXT);
INSERT INTO raw_import2 (id, raw_value) VALUES
    (1, '1990.50'), (2, 'abc'), (3, '-500'), (4, '3.14.15'), (5, '0');
```

<details>
<summary>เฉลย</summary>

```sql
SELECT
    id,
    raw_value,
    CASE WHEN pg_input_is_valid(raw_value, 'numeric')
         THEN raw_value::numeric
         ELSE NULL
    END AS safe_value
FROM raw_import2
ORDER BY id;
```

ผลลัพธ์ตัวอย่าง:

```
 id | raw_value | safe_value
-----+------------+---------------
   1 | 1990.50    |     1990.50
   2 | abc        |         <NULL>
   3 | -500       |      -500.00
   4 | 3.14.15    |         <NULL>
   5 | 0          |         0.00
```

</details>

### แบบฝึกหัดที่ 8

จำลองการ insert ออเดอร์ใหม่โดยใช้ `NEXTVAL` เพื่อดึง `order_id` มาก่อน แล้วใช้ค่านั้นในการ insert `order_items` ที่เกี่ยวข้องทันที (แทนที่จะพึ่งพา `DEFAULT` แล้วค้นหา id ย้อนหลัง)

<details>
<summary>เฉลย</summary>

```sql
DO $$
DECLARE
    new_order_id INT;
BEGIN
    new_order_id := nextval('orders_order_id_seq');

    INSERT INTO orders (order_id, customer_id, employee_id, order_date, status, ship_country)
    VALUES (new_order_id, 5, 4, now(), 'pending', 'USA');

    INSERT INTO order_items (order_id, product_id, quantity, unit_price)
    VALUES (new_order_id, 7, 2, 2990.00);

    RAISE NOTICE 'สร้างออเดอร์ id = % สำเร็จ', new_order_id;
END $$;
```

การใช้ `nextval` ดึงค่ามาเก็บในตัวแปรก่อนนั้นปลอดภัยกว่าการ insert ด้วย `DEFAULT` แล้วค่อย `SELECT MAX(order_id)` ย้อนหลัง เพราะในระบบที่มีการทำงานพร้อมกันหลาย transaction (concurrent) การ `SELECT MAX(...)` อาจได้ id ของ transaction อื่นมาผิดตัว ในขณะที่ `nextval` รับประกันว่าค่าที่ได้เป็นของ transaction ปัจจุบันเท่านั้น (atomic operation)

</details>

### แบบฝึกหัดที่ 9

หาค่าสัมประสิทธิ์สหสัมพันธ์ (`CORR`) ระหว่างจำนวนวันตั้งแต่ลูกค้าสมัครสมาชิก (`signup_date`) จนถึงวันนี้ กับจำนวนออเดอร์ที่ลูกค้าคนนั้นเคยสั่งซื้อ เพื่อดูว่าลูกค้าที่สมัครมานานมักซื้อบ่อยกว่าหรือไม่

<details>
<summary>เฉลย</summary>

```sql
SELECT
    ROUND(
        CORR(order_count.total_orders, days_since_signup.days)::numeric, 4
    ) AS corr_tenure_orders
FROM (
    SELECT customer_id, CURRENT_DATE - signup_date AS days
    FROM customers
) days_since_signup
JOIN (
    SELECT customer_id, COUNT(*) AS total_orders
    FROM orders
    GROUP BY customer_id
) order_count ON order_count.customer_id = days_since_signup.customer_id;
```

ผลลัพธ์ตัวอย่าง (ค่าประมาณเพื่อประกอบความเข้าใจ):

```
 corr_tenure_orders
----------------------
            0.1523
```

ค่าที่ใกล้ 0 บ่งชี้ว่าในชุดข้อมูลตัวอย่างนี้ ความสัมพันธ์ระหว่างระยะเวลาการเป็นสมาชิกกับจำนวนออเดอร์แทบไม่มีนัยสำคัญเชิงเส้น — ควรระวัง `LEFT JOIN` แทน `JOIN` หากต้องการรวมลูกค้าที่ไม่เคยสั่งซื้อเลยด้วย (จะได้ `total_orders = 0` แทนที่จะถูกตัดออกจากการวิเคราะห์)

</details>

### แบบฝึกหัดที่ 10

โจทย์รวม: สำหรับทุกออเดอร์ที่ `status = 'completed'` ให้คำนวณ (1) subtotal จาก `order_items` (2) หักส่วนลด 8% ถ้า subtotal เกิน 25,000 บาท (3) บวก VAT 7% (4) ปัดราคาสุดท้ายให้ลงท้ายด้วย `.99` โดยปัดขึ้นไปหลักสิบก่อนเสมอ

<details>
<summary>เฉลย</summary>

```sql
SELECT
    o.order_id,
    subtotal.amount AS subtotal,
    ROUND(
        subtotal.amount * (CASE WHEN subtotal.amount > 25000 THEN 0.92 ELSE 1.0 END),
        2
    ) AS after_discount,
    ROUND(
        subtotal.amount * (CASE WHEN subtotal.amount > 25000 THEN 0.92 ELSE 1.0 END) * 1.07,
        2
    ) AS after_vat,
    (CEIL(
        subtotal.amount * (CASE WHEN subtotal.amount > 25000 THEN 0.92 ELSE 1.0 END) * 1.07 / 10
    ) * 10) - 0.01 AS final_price_ending_99
FROM orders o
JOIN (
    SELECT order_id, SUM(quantity * unit_price) AS amount
    FROM order_items
    GROUP BY order_id
) subtotal ON subtotal.order_id = o.order_id
WHERE o.status = 'completed'
ORDER BY o.order_id;
```

ผลลัพธ์ตัวอย่าง (บางส่วน):

```
 order_id | subtotal | after_discount | after_vat | final_price_ending_99
-----------+------------+-------------------+-------------+---------------------------
         1 |  38970.00 |        35852.40 |  38362.07 |               38369.99
         2 |  23760.00 |        23760.00 |  25423.20 |               25429.99
         4 |  26770.00 |        24628.40 |  26352.39 |               26359.99
         5 |  13980.00 |        13980.00 |  14958.60 |               14959.99
         6 |  13980.00 |        13980.00 |  14958.60 |               14959.99
         9 |  45990.00 |        42310.80 |  45272.56 |               45279.99
```

โจทย์นี้รวมฟังก์ชันจากทั้งบทเข้าด้วยกัน: `SUM`/`GROUP BY` (aggregate), `CASE` (เงื่อนไขส่วนลด), `ROUND` (ปัดยอดเงินมาตรฐาน), และ `CEIL` ร่วมกับการหาร/คูณ (เทคนิคปัดราคาให้ลงท้าย `.99`) ซึ่งเป็นรูปแบบการคำนวณที่พบได้ทั่วไปในระบบตะกร้าสินค้าของธุรกิจจริง

</details>

---

**บทถัดไป:** [Part 034 — Conditional Logic](./part-034-conditional-logic.md)
