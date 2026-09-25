# Part 053: Full Text Search — tsvector, tsquery และการจัดอันดับผลลัพธ์

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับสูง | Part 053

---

## เป้าหมายการเรียนรู้

หลังจากเรียนจบบทนี้ ผู้เรียนจะสามารถ:

- อธิบายได้ว่าทำไม `LIKE`/`ILIKE` ถึงไม่เพียงพอสำหรับการค้นหาข้อความจริงจัง และอะไรคือปัญหาของคำที่ผันรูปหลายแบบ (running/runs/ran)
- เข้าใจว่า `tsvector` คืออะไร รู้จักกระบวนการแปลงข้อความให้เป็น lexeme ที่ผ่านการ normalize ด้วย `to_tsvector()`
- สร้างคำค้นหาในรูปแบบ `tsquery` ได้หลายวิธีด้วย `to_tsquery`, `plainto_tsquery` และ `websearch_to_tsquery`
- ใช้ match operator `@@` เพื่อจับคู่ `tsvector` กับ `tsquery` ในเงื่อนไข `WHERE`
- เข้าใจ Text Search Configuration เช่น `'english'`, รู้จัก stop words และ stemming ว่าทำงานอย่างไร รวมถึงข้อจำกัดของมัน
- จัดอันดับความเกี่ยวข้องของผลลัพธ์ด้วย `ts_rank` และ `ts_rank_cd`
- สร้าง snippet ไฮไลท์คำที่ตรงกับคำค้นหาด้วย `ts_headline` คล้ายผลลัพธ์ของ search engine
- ออกแบบ generated column เพื่อเก็บ `tsvector` ล่วงหน้าด้วย `GENERATED ALWAYS AS ... STORED` เทียบกับแนวทาง trigger แบบดั้งเดิม
- สร้าง GIN Index บนคอลัมน์ `tsvector` เพื่อเร่งความเร็วการค้นหา พร้อมอ่านผล `EXPLAIN ANALYZE` เป็น
- ประกอบร่างทุกความรู้เป็นระบบค้นหาสินค้าและรีวิวแบบเต็มรูปแบบที่มีทั้ง ranking และ highlight

---

## เตรียมข้อมูล

บทนี้ยังคงใช้ฐานข้อมูล e-commerce เดิมที่ใช้ตลอดหลักสูตร แต่จะ**เพิ่มคอลัมน์ `description TEXT`** ที่มีเนื้อหาภาษาอังกฤษยาวหลายประโยคในตาราง `products` และ**เพิ่มตาราง `reviews`** ที่เก็บรีวิวจากลูกค้า เพื่อให้มีข้อความจริงจังพอสำหรับสาธิต Full Text Search

ถ้าผู้เรียนสร้างฐานข้อมูลใหม่ ให้รันสคริปต์เต็มด้านล่างนี้ทั้งหมด (ถ้ามีตารางจาก Part ก่อนหน้าอยู่แล้ว ให้ `DROP TABLE` ก่อน หรือสร้างในฐานข้อมูลใหม่)

### 1. สร้างตาราง

```sql
DROP TABLE IF EXISTS reviews CASCADE;
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
    supplier_id   SERIAL PRIMARY KEY,
    supplier_name VARCHAR(150) NOT NULL,
    country       VARCHAR(60)
);

CREATE TABLE products (
    product_id     SERIAL PRIMARY KEY,
    product_name   VARCHAR(150) NOT NULL,
    description    TEXT,
    category_id    INTEGER REFERENCES categories(category_id),
    supplier_id    INTEGER REFERENCES suppliers(supplier_id),
    unit_price     NUMERIC(10,2) NOT NULL,
    stock_quantity INTEGER NOT NULL DEFAULT 0,
    is_active      BOOLEAN NOT NULL DEFAULT true
);

CREATE TABLE customers (
    customer_id  SERIAL PRIMARY KEY,
    first_name   VARCHAR(60) NOT NULL,
    last_name    VARCHAR(60) NOT NULL,
    email        VARCHAR(150) UNIQUE,
    country      VARCHAR(60),
    signup_date  DATE NOT NULL DEFAULT CURRENT_DATE
);

CREATE TABLE orders (
    order_id     SERIAL PRIMARY KEY,
    customer_id  INTEGER REFERENCES customers(customer_id),
    order_date   TIMESTAMPTZ NOT NULL DEFAULT now(),
    status       VARCHAR(20) NOT NULL DEFAULT 'pending',
    ship_country VARCHAR(60)
);

CREATE TABLE order_items (
    order_item_id SERIAL PRIMARY KEY,
    order_id      INTEGER REFERENCES orders(order_id),
    product_id    INTEGER REFERENCES products(product_id),
    quantity      INTEGER NOT NULL CHECK (quantity > 0),
    unit_price    NUMERIC(10,2) NOT NULL
);

CREATE TABLE reviews (
    review_id    SERIAL PRIMARY KEY,
    product_id   INTEGER REFERENCES products(product_id),
    customer_id  INTEGER REFERENCES customers(customer_id),
    rating       INTEGER CHECK (rating BETWEEN 1 AND 5),
    review_text  TEXT,
    review_date  DATE DEFAULT CURRENT_DATE
);
```

### 2. เติมข้อมูลตัวอย่าง

```sql
-- categories
INSERT INTO categories (category_name, parent_category_id) VALUES
    ('Electronics',      NULL),  -- 1
    ('Computers',        1),     -- 2
    ('Audio',            1),     -- 3
    ('Photography',      1),     -- 4
    ('Wearables',        1),     -- 5
    ('Smart Home',       1),     -- 6
    ('Kitchenware',      NULL),  -- 7
    ('Sports & Outdoor', NULL),  -- 8
    ('Books',            NULL),  -- 9
    ('Footwear',         8);     -- 10

-- suppliers
INSERT INTO suppliers (supplier_name, country) VALUES
    ('Thai Tech Supply',      'Thailand'),        -- 1
    ('Global Gadget Co.',     'China'),           -- 2
    ('Nordic Home Living',    'Sweden'),          -- 3
    ('Kitchen Master',        'Germany'),         -- 4
    ('Active Gear Co.',       'USA'),             -- 5
    ('BookWorm Publishing',   'United Kingdom'),  -- 6
    ('Pure Beauty Labs',      'South Korea'),     -- 7
    ('Everyday Essentials Ltd','Thailand'),       -- 8
    ('Smart Living Imports',  'Japan'),           -- 9
    ('TrailReady Gear Co.',   'Vietnam');         -- 10

-- customers
INSERT INTO customers (first_name, last_name, email, country) VALUES
    ('Somchai',   'Jaidee',      'somchai.j@example.com',    'Thailand'),        -- 1
    ('Pornthip',  'Suksai',      'pornthip.s@example.com',   'Thailand'),        -- 2
    ('John',      'Anderson',    'john.anderson@example.com','USA'),             -- 3
    ('Emily',     'Clark',       'emily.clark@example.com',  'United Kingdom'),  -- 4
    ('Wei',       'Zhang',       'wei.zhang@example.com',    'China'),           -- 5
    ('Yuki',      'Tanaka',      'yuki.tanaka@example.com',  'Japan'),           -- 6
    ('Anna',      'Kowalski',    'anna.k@example.com',       'Poland'),          -- 7
    ('Carlos',    'Ramirez',     'carlos.r@example.com',     'Mexico'),          -- 8
    ('Nattapong', 'Wongsawat',   'nattapong.w@example.com',  'Thailand'),        -- 9
    ('Sophie',    'Martin',      'sophie.martin@example.com','France'),         -- 10
    ('David',     'Lee',         'david.lee@example.com',    'South Korea'),     -- 11
    ('Priya',     'Sharma',      'priya.sharma@example.com', 'India'),           -- 12
    ('Michael',   'Brown',       'michael.brown@example.com','USA'),             -- 13
    ('Kanya',     'Srisuk',      'kanya.s@example.com',      'Thailand'),        -- 14
    ('Lucas',     'Silva',       'lucas.silva@example.com',  'Brazil');          -- 15

-- products (product_name + rich English description)
INSERT INTO products (product_name, description, category_id, supplier_id, unit_price, stock_quantity) VALUES
('TrailBlazer Running Shoes',
 'Built for runners who love hitting rugged trails, the TrailBlazer has carried athletes through countless marathons and early morning training runs. An aggressive outsole grips wet rocks and muddy paths so you keep running safely in any weather. Many long-distance runners say they ran their first ultramarathon in this very pair.',
 10, 5, 3290.00, 120),  -- 1
('Urban Sprint Running Shoes',
 'A lightweight running shoe designed for daily runs on city pavement. Whether you are jogging to the office or running interval sprints at the track, the breathable mesh upper keeps your feet cool mile after mile. Runners who switched to this model reported running noticeably faster splits within just a few weeks.',
 10, 5, 2590.00, 200),  -- 2
('Marathon Pro Running Shoes',
 'Engineered specifically for marathon runners chasing a personal best, this shoe combines a responsive foam midsole with a carbon-infused plate. Elite and amateur runners alike have run full marathons in this pair without a single blister. If you are training for race day, this is the shoe serious runners trust.',
 10, 10, 4590.00, 60),  -- 3
('CloudStep Walking Shoes',
 'A cushioned walking shoe made for long days on your feet, from city sightseeing to standing shifts at work. The soft insole absorbs impact with every step, and the wide toe box keeps your feet comfortable from morning until night. Not intended for high-intensity sport, but ideal for everyday walking and light travel.',
 10, 5, 2190.00, 150),  -- 4
('Wireless Noise-Cancelling Headphones',
 'Immerse yourself in music with industry-leading active noise cancellation that blocks out engine hum, chatter, and traffic noise. The plush ear cushions and adjustable headband make these headphones comfortable for hours of continuous listening. A single charge delivers up to thirty hours of battery life, making them perfect for long-haul flights.',
 3, 2, 5990.00, 80),   -- 5
('Bluetooth Portable Speaker',
 'This rugged, waterproof speaker delivers surprisingly deep bass for its compact size and pairs instantly over Bluetooth with any smartphone or laptop. Take it to the beach, the pool, or a backyard party without worrying about splashes. Twelve hours of battery life keeps the music going all afternoon and into the evening.',
 3, 2, 1990.00, 140),  -- 6
('True Wireless Earbuds Pro',
 'Compact true wireless earbuds with a secure fit that stays put even during running, cycling, or an intense gym session. Touch controls let you skip tracks and answer calls without reaching for your phone, and the charging case adds an extra twenty hours of battery life on the go. Sweat and splash resistance make them a reliable workout companion.',
 3, 1, 2990.00, 175),  -- 7
('UltraBook 14 Laptop',
 'A featherweight fourteen-inch laptop built for professionals who move between meetings, cafes, and home offices throughout the day. The all-day battery easily survives a full workday of video calls, spreadsheets, and web browsing without needing a charger. A bright anti-glare display and backlit keyboard round out a laptop designed for real productivity.',
 2, 1, 32900.00, 40),  -- 8
('Gaming Laptop X',
 'A high-performance gaming laptop with a dedicated graphics card capable of running the latest titles at maximum settings without stutter. The advanced cooling system keeps temperatures in check even after hours of demanding gameplay. RGB backlighting and a fast refresh-rate display complete the experience for competitive gamers.',
 2, 2, 54900.00, 25),  -- 9
('Mechanical Keyboard RGB',
 'This mechanical keyboard features tactile switches that give every keystroke a satisfying, audible click favored by typists and gamers alike. Customizable per-key RGB lighting lets you match your setup or highlight your favorite gaming shortcuts. A durable aluminum frame ensures the keyboard survives years of heavy daily typing.',
 2, 1, 2490.00, 90),   -- 10
('Wireless Ergonomic Mouse',
 'Designed to reduce wrist strain during long workdays, this ergonomic mouse features a vertical grip and adjustable sensitivity for both office work and detailed design tasks. The wireless receiver offers a stable, lag-free connection up to ten meters away. A single battery charge lasts several weeks under normal daily use.',
 2, 1, 990.00, 220),   -- 11
('27-inch 4K Monitor',
 'A twenty-seven inch 4K display with accurate color reproduction, ideal for photo editing, video work, and everyday productivity. Thin bezels make it easy to set up a dual-monitor workspace, while the adjustable stand tilts and swivels to find the perfect viewing angle. HDR support adds extra depth to movies and games.',
 2, 9, 9990.00, 55),   -- 12
('External SSD 1TB',
 'This pocket-sized external solid state drive transfers files at blazing speeds, making large video exports and photo backups nearly instant. A rugged rubberized shell protects the drive from accidental drops during travel. With one terabyte of storage, it comfortably holds an entire photography portfolio or game library.',
 2, 1, 2790.00, 130),  -- 13
('Mirrorless Camera Z5',
 'A mirrorless camera built for photographers who refuse to compromise on image quality, capturing stunning detail even in dim, low-light conditions. Fast autofocus tracks moving subjects with ease, whether you are photographing a sprinting athlete or a running toddler at the park. The compact body is light enough for a full day of travel photography.',
 4, 9, 28900.00, 20),  -- 14
('Camera Drone Explorer',
 'This camera drone captures cinematic aerial footage in crisp 4K resolution, complete with a stabilized three-axis gimbal that keeps every shot smooth. Obstacle avoidance sensors make flying through parks and forests safer for beginners. Roughly twenty minutes of flight time per battery is typical for aerial photography sessions.',
 4, 2, 21900.00, 18),  -- 15
('Fitness Smartwatch Pulse',
 'Track every workout with a smartwatch that monitors heart rate, sleep quality, and daily activity around the clock. Built-in GPS accurately records distance and pace whether you are running, cycling, or swimming laps. Runners training for a race can review detailed splits and recovery data right on their wrist.',
 5, 9, 6490.00, 100),  -- 16
('Robot Vacuum Cleaner',
 'This robot vacuum maps your home automatically and navigates around furniture, cleaning hardwood floors and carpets in a single scheduled pass. A smartphone app lets you start cleaning, set no-go zones, and check battery status from anywhere. The self-emptying dustbin means less manual maintenance for busy households.',
 6, 3, 8990.00, 45),   -- 17
('Digital Air Fryer XL',
 'This extra-large digital air fryer crisps fries, wings, and vegetables using a fraction of the oil required by traditional frying. Eight preset cooking programs take the guesswork out of weeknight dinners, and the nonstick basket wipes clean in seconds. A large family-sized capacity means fewer batches and faster mealtimes.',
 7, 4, 3490.00, 70),   -- 18
('Stainless Steel Knife Set',
 'A complete eight-piece knife set forged from high-carbon stainless steel, holding a razor-sharp edge through months of daily chopping. Each knife is balanced for comfortable, fatigue-free use during long meal-prep sessions. The included wooden block keeps every blade organized and within easy reach.',
 7, 4, 2290.00, 85),   -- 19
('Cast Iron Skillet 12-inch',
 'This pre-seasoned twelve-inch cast iron skillet delivers even heat retention for searing, baking, and frying on the stovetop or in the oven. With proper care, cast iron cookware like this can last for generations, developing a naturally nonstick surface over time. The sturdy handle stays cool enough to grip during quick stovetop stirring.',
 7, 4, 1290.00, 110),  -- 20
('Drip Coffee Maker Deluxe',
 'Wake up to freshly brewed coffee every morning with a programmable drip coffee maker that can be set the night before. The reusable gold-tone filter eliminates the ongoing cost of paper filters while preserving the coffee''s natural oils and flavor. A ten-cup glass carafe keeps coffee hot without the bitterness of a heating plate left on too long.',
 7, 4, 1890.00, 95),   -- 21
('High-Speed Blender Pro',
 'This high-speed blender pulverizes ice, frozen fruit, and leafy greens into silky smoothies in under a minute. Variable speed settings and a pulse function give you full control over texture, from chunky salsas to perfectly smooth soups. The heavy-duty motor is built to handle daily use in busy kitchens.',
 7, 3, 2590.00, 75),   -- 22
('Camping Tent 4-Person',
 'This four-person camping tent sets up in under ten minutes thanks to a color-coded pole system, even for first-time campers. A waterproof rainfly and sealed floor seams keep the interior completely dry during unexpected mountain storms. Two large mesh windows improve airflow on warm summer nights.',
 8, 10, 4990.00, 30),  -- 23
('Yoga Mat Premium',
 'This premium yoga mat offers a non-slip surface that grips confidently through sweaty vinyasa flows and slow, deliberate stretching sessions alike. Extra cushioning protects knees and wrists during floor poses without sacrificing stability during balance work. The mat rolls up compactly for easy storage or travel to the studio.',
 8, 5, 890.00, 200),   -- 24
('Hiking Backpack 40L',
 'Built for multi-day hiking trips, this forty-liter backpack distributes weight evenly across a padded hip belt and ventilated back panel. Multiple compartments keep a rain jacket, water bottle, and camp stove organized and quickly accessible on the trail. Reinforced stitching at every stress point stands up to years of rugged use.',
 8, 10, 3290.00, 65),  -- 25
('Insulated Water Bottle',
 'This double-walled insulated water bottle keeps drinks ice cold for up to twenty-four hours or steaming hot for twelve, no matter the weather outside. The leak-proof lid survives being tossed into a gym bag or hiking backpack without spilling. A wide mouth opening makes it easy to add ice cubes or clean thoroughly between uses.',
 8, 8, 590.00, 300),   -- 26
('Waterproof Rain Jacket',
 'This waterproof rain jacket keeps hikers and runners dry through sudden downpours without trapping heat and moisture inside. Breathable fabric panels under the arms let excess heat escape even when you are running hard uphill. A packable design lets it stuff into its own pocket and disappear inside a backpack when the sky clears.',
 8, 10, 2990.00, 80),  -- 27
('Cycling Helmet Aero',
 'Designed for road cyclists chasing every possible second, this aerodynamic helmet reduces drag without sacrificing ventilation on long, hot climbs. An adjustable rear dial achieves a secure, wobble-free fit in seconds. Reflective accents improve visibility for cyclists riding home after sunset.',
 8, 5, 1990.00, 60),   -- 28
('Mystery Novel: The Silent Detective',
 'A gripping mystery novel following a retired detective pulled back into one final case after a body is discovered in a locked, silent house. Twist after twist keeps readers guessing until the final chapter reveals a killer nobody saw coming. Critics have called it the most compelling detective story of the year.',
 9, 6, 350.00, 500),   -- 29
('Cookbook: Flavors of Thailand',
 'This vibrant cookbook explores the bold flavors of Thailand through more than one hundred authentic recipes, from fiery curries to fragrant street food noodles. Step-by-step photographs guide home cooks through techniques passed down through generations of Thai kitchens. A dedicated chapter on essential pastes and sauces builds the foundation for every dish in the book.',
 9, 6, 490.00, 400);   -- 30

-- orders
INSERT INTO orders (customer_id, status, ship_country) VALUES
    (1,  'completed', 'Thailand'),        -- 1
    (2,  'completed', 'Thailand'),        -- 2
    (3,  'completed', 'USA'),             -- 3
    (4,  'completed', 'United Kingdom'),  -- 4
    (5,  'completed', 'China'),           -- 5
    (6,  'shipped',   'Japan'),           -- 6
    (7,  'completed', 'Poland'),          -- 7
    (8,  'completed', 'Mexico'),          -- 8
    (9,  'shipped',   'Thailand'),        -- 9
    (11, 'completed', 'South Korea'),     -- 10
    (13, 'completed', 'USA'),             -- 11
    (14, 'pending',   'Thailand');        -- 12

-- order_items
INSERT INTO order_items (order_id, product_id, quantity, unit_price) VALUES
    (1, 1,  1, 3290.00),
    (1, 26, 2, 590.00),
    (2, 4,  1, 2190.00),
    (3, 5,  1, 5990.00),
    (4, 27, 1, 2990.00),
    (5, 9,  1, 54900.00),
    (6, 23, 1, 4990.00),
    (6, 24, 1, 890.00),
    (7, 3,  1, 4590.00),
    (8, 17, 1, 8990.00),
    (9, 10, 1, 2490.00),
    (9, 11, 1, 990.00),
    (10, 7, 2, 2990.00),
    (11, 8, 1, 32900.00),
    (12, 29, 3, 350.00),
    (12, 30, 1, 490.00);

-- reviews
INSERT INTO reviews (product_id, customer_id, rating, review_text, review_date) VALUES
(1,  1,  5, 'I ran my first half marathon in these shoes and they performed beautifully. My feet stayed comfortable through twenty-one kilometers of running on mixed terrain. Highly recommend for any serious runner.', '2024-02-10'),
(1,  9,  4, 'Great shoes for daily running. I''ve been running about thirty kilometers a week in them for two months and the sole shows very little wear. A bit narrow for wide feet though.', '2024-03-02'),
(2,  3,  5, 'These running shoes are incredibly light. I run every morning before work and they make the whole routine much more enjoyable. Runners with knee issues will appreciate the extra cushioning.', '2024-01-20'),
(2,  13, 2, 'I was excited to start running again after my injury, but these shoes gave me blisters after just one run. Returning them.', '2024-04-15'),
(3,  5,  5, 'The Marathon Pro is exactly what serious runners need. I ran a full marathon last month wearing this pair and finished with zero blisters. Worth every baht.', '2024-05-01'),
(3,  7,  3, 'Decent shoes overall, but heavier than I expected for a running shoe. Fine for casual runs, not ideal for speed training.', '2024-02-27'),
(4,  2,  4, 'I mostly use these for walking rather than running, and they are perfect for long walks around the neighborhood. Very comfortable for standing all day at work too.', '2024-03-18'),
(5,  4,  5, 'The noise cancelling on these headphones is outstanding. I use them on every flight and they block out engine noise completely. Battery life easily lasts a full transatlantic flight.', '2024-01-11'),
(5,  11, 2, 'Sound quality is good but the battery drains faster than advertised. Disappointed after only three months of use.', '2024-06-09'),
(6,  6,  4, 'This bluetooth speaker fills my whole living room with sound. Great for parties, though bass gets distorted at maximum volume.', '2024-04-22'),
(7,  8,  5, 'These wireless earbuds fit securely even when I''m running sprints at the gym. The charging case is compact and battery life is impressive.', '2024-02-14'),
(8,  10, 5, 'The UltraBook 14 is fast, light, and the battery genuinely lasts all day. Perfect laptop for remote work and travel.', '2024-03-30'),
(9,  12, 4, 'Gaming Laptop X handles every modern game at high settings without overheating. A bit heavy to carry around campus.', '2024-05-19'),
(10, 1,  5, 'This mechanical keyboard has a satisfying tactile click and the RGB lighting looks fantastic in a dark room.', '2024-01-25'),
(12, 3,  5, 'The 4K monitor is crisp and color accurate straight out of the box. Great for photo editing and general productivity.', '2024-06-02'),
(14, 9,  5, 'The Mirrorless Camera Z5 captures stunning detail even in low light. I used it to photograph a marathon race and every runner in the crowd was sharp and clear.', '2024-04-05'),
(15, 14, 3, 'The drone flies smoothly but the battery only lasts about twenty minutes per charge. Footage quality is excellent though.', '2024-02-08'),
(16, 2,  4, 'This smartwatch tracked my running pace accurately during a half marathon. Heart rate monitoring seems reliable too.', '2024-03-12'),
(17, 15, 1, 'The robot vacuum got stuck under my sofa every single day and the app kept disconnecting. Terrible experience, would not recommend.', '2024-05-27'),
(18, 5,  5, 'The air fryer makes crispy fries with almost no oil. Cleanup is quick and the digital controls are intuitive.', '2024-01-30'),
(19, 7,  5, 'This knife set is razor sharp right out of the box. The handles feel balanced and comfortable during long prep sessions.', '2024-04-18'),
(21, 13, 2, 'The coffee maker leaks water from the base after just a few weeks of use. Coffee tastes fine but the design is flawed.', '2024-06-21'),
(23, 6,  5, 'We took this tent camping in the mountains and it kept us completely dry during a heavy overnight storm. Setup takes under ten minutes.', '2024-05-08'),
(27, 4,  4, 'This rain jacket kept me dry while running through a sudden downpour. Breathable enough that I didn''t overheat despite running hard.', '2024-03-25'),
(29, 11, 5, 'The Silent Detective kept me up all night. I could not stop reading until I discovered who the killer was in the final chapter.', '2024-02-19');
```

ตรวจสอบว่าข้อมูลเข้าครบ:

```sql
SELECT
    (SELECT count(*) FROM categories) AS categories,
    (SELECT count(*) FROM suppliers)  AS suppliers,
    (SELECT count(*) FROM products)   AS products,
    (SELECT count(*) FROM customers)  AS customers,
    (SELECT count(*) FROM orders)     AS orders,
    (SELECT count(*) FROM order_items) AS order_items,
    (SELECT count(*) FROM reviews)    AS reviews;
```

```
 categories | suppliers | products | customers | orders | order_items | reviews
------------+-----------+----------+-----------+--------+--------------+---------
         10 |        10 |       30 |        15 |     12 |           16 |      25
(1 row)
```

จากข้อมูลชุดนี้ให้สังเกตไว้ล่วงหน้าว่า คำว่า **"run" / "running" / "runs" / "ran"** ถูกกระจายอยู่ในหลายสินค้าและหลายรีวิวโดยตั้งใจ (รองเท้าวิ่ง เสื้อกันฝน กล้อง สมาร์ทวอทช์ หูฟัง) เพื่อให้เห็นชัดว่า Full Text Search แยกแยะ "ความเกี่ยวข้องจริง" ได้ดีกว่าการค้นหาด้วย `LIKE` มากแค่ไหน

---

## Step 521: ทำไม LIKE/ILIKE ไม่พอสำหรับการค้นหาข้อความจริงจัง

ลองค้นหาสินค้าที่เกี่ยวกับ "การวิ่ง" ด้วยวิธีที่คุ้นเคยที่สุดก่อน — `ILIKE` แบบ pattern matching:

```sql
SELECT product_id, product_name
FROM products
WHERE description ILIKE '%running%';
```

```
 product_id |            product_name
------------+--------------------------------------
          1 | TrailBlazer Running Shoes
          2 | Urban Sprint Running Shoes
          3 | Marathon Pro Running Shoes
          7 | True Wireless Earbuds Pro
         14 | Mirrorless Camera Z5
         16 | Fitness Smartwatch Pulse
         27 | Waterproof Rain Jacket
(7 rows)
```

ดูเผิน ๆ เหมือนใช้ได้ผลดี แต่ปัญหาคือ **`description` ของสินค้าใน `product_id = 3` (Marathon Pro)** เขียนว่า *"...runners alike have **run** full marathons..."* — ใช้คำว่า **"run"** ไม่ใช่ **"running"`** ตรง ๆ ในประโยคนั้น ลองดูว่าถ้าค้นด้วยคำว่า `'run'` ตรง ๆ จะเกิดอะไรขึ้น:

```sql
SELECT product_id, product_name
FROM products
WHERE description ILIKE '%run%';
```

```
 product_id |            product_name
------------+--------------------------------------
          1 | TrailBlazer Running Shoes
          2 | Urban Sprint Running Shoes
          3 | Marathon Pro Running Shoes
          7 | True Wireless Earbuds Pro
         14 | Mirrorless Camera Z5
         16 | Fitness Smartwatch Pulse
         27 | Waterproof Rain Jacket
(7 rows)
```

ผลลัพธ์เหมือนเดิมทุกแถว — บังเอิญไม่มีคำไหนที่มี substring `run` แทรกอยู่แบบผิดความหมาย (เช่น "current") ในชุดข้อมูลนี้ แต่ปัญหาที่ร้ายแรงกว่าซ่อนอยู่ในตาราง `reviews`:

```sql
SELECT review_id, review_text
FROM reviews
WHERE review_text ILIKE '%run%'
ORDER BY review_id;
```

```
 review_id |                                review_text (ย่อ)
-----------+----------------------------------------------------------------------------------
         1 | I ran my first half marathon in these shoes...
         2 | Great shoes for daily running. I've been running about thirty kilometers...
         3 | ...I run every morning before work...
         4 | I was excited to start running again after my injury, but these shoes gave me...
         5 | ...I ran a full marathon last month...
         6 | ...Fine for casual runs, not ideal for speed training.
         7 | I mostly use these for walking rather than running...
        11 | ...even when I'm running sprints at the gym...
        16 | ...I used it to photograph a marathon race and every runner...
        18 | This smartwatch tracked my running pace accurately...
        24 | This rain jacket kept me dry while running through a sudden downpour...
(11 rows)
```

ดูดีอีกครั้ง — แต่ให้สังเกต **review_id = 1** ที่เขียนว่า *"I **ran** my first half marathon..."* คำว่า **"ran"** ไม่มี substring `run` อยู่เลย (ตัวอักษรคือ r-a-n ไม่ใช่ r-u-n) `ILIKE '%run%'` จับคำนี้ได้ก็เพราะในประโยคเดียวกันมีคำว่า "running" ปนอยู่ด้วยเท่านั้น ลองค้นเฉพาะรีวิวที่ **มีแต่คำว่า "ran"** อย่างเดียว (จำลองสถานการณ์จริงที่ลูกค้าอาจเขียนแค่ "ran" โดยไม่มี "running" ปนอยู่):

```sql
SELECT 'ran a marathon in these' ILIKE '%run%' AS matched_by_like;
```

```
 matched_by_like
-----------------
 f
(1 row)
```

นี่คือแก่นของปัญหา: **"run"**, **"running"**, **"runs"** เป็นคำที่ผันจากรากเดียวกันแบบ regular และ `LIKE` พอจะ "เดา" ได้ถ้าใช้ pattern สั้น ๆ อย่าง `%run%` แต่ **"ran"** เป็นการผันแบบ irregular (อดีตกาลของ "run" ที่ไม่เติม -ed) ซึ่งไม่มีความสัมพันธ์เชิง substring กับคำว่า "run" เลย ต่อให้เขียน pattern ฉลาดแค่ไหนก็ไม่มีทางครอบคลุมคำผันภาษาอังกฤษได้ครบด้วยการเทียบ substring ตรง ๆ

นอกจากปัญหาเรื่องความหมายแล้ว ยังมีปัญหาเรื่อง **performance**:

- `ILIKE '%คำ%'` ที่มี wildcard นำหน้า (`%`) ไม่สามารถใช้ B-tree index ได้เลย ต้องทำ **Sequential Scan** อ่านทุกแถวและสแกนทุกตัวอักษรของทุกคอลัมน์เสมอ
- ยิ่งข้อความยาว (เช่น `description` หรือ `review_text` ที่มีหลายร้อยตัวอักษร) การสแกนแบบนี้ยิ่งช้าลงตามสัดส่วน
- ไม่มีแนวคิดเรื่อง "ความเกี่ยวข้อง" (relevance) เลย — ผลลัพธ์ทุกแถวที่ match เท่ากันหมด ไม่สามารถบอกได้ว่าแถวไหน "เกี่ยวข้องมากกว่า" แถวไหน

สิ่งที่เราต้องการคือระบบที่:

1. เข้าใจว่า "run", "running", "runs" คือคำเดียวกันในเชิงความหมาย (แต่ก็ต้องรู้ข้อจำกัดว่า "ran" อาจจะไม่ถูกจัดกลุ่มด้วยเสมอไป — จะอธิบายใน Step 525)
2. ตัดคำที่ไม่มีความหมาย เช่น "the", "a", "is" ออกไปก่อนค้นหา (stop words)
3. ค้นหาได้เร็วด้วย index ที่ออกแบบมาสำหรับข้อความโดยเฉพาะ
4. จัดอันดับผลลัพธ์ตามความเกี่ยวข้องได้

นี่คือสิ่งที่ **PostgreSQL Full Text Search** ถูกออกแบบมาเพื่อแก้ปัญหาโดยเฉพาะ ผ่านชนิดข้อมูล `tsvector` และ `tsquery`

---

## Step 522: tsvector คืออะไร — การแปลงข้อความเป็น lexeme

`tsvector` คือชนิดข้อมูลพิเศษที่เก็บ **"ข้อความที่ผ่านการประมวลผลแล้ว"** ในรูปแบบที่เหมาะกับการค้นหา แทนที่จะเก็บ string ดิบ ๆ PostgreSQL จะ:

1. **Parse** ข้อความให้เป็น token (คำแต่ละคำ, ตัวเลข, อีเมล, URL ฯลฯ — แต่ละแบบมีกฎการตัดคำต่างกัน)
2. **Normalize** แต่ละ token ด้วย dictionary ที่กำหนด (เช่น ตัด stop words, ทำ stemming ให้เหลือรากศัพท์) ผลลัพธ์แต่ละคำเรียกว่า **lexeme**
3. **เก็บตำแหน่ง** (position) ของแต่ละ lexeme ในข้อความต้นฉบับไว้ด้วย เพื่อใช้คำนวณระยะห่างระหว่างคำในภายหลัง (เช่นตอนทำ ranking หรือ phrase search)

ฟังก์ชันหลักคือ `to_tsvector(config, text)`:

```sql
SELECT to_tsvector('english',
    'The Runners are Running and one Runner ran fastest yesterday');
```

```
                              to_tsvector
------------------------------------------------------------------------
 'fast':9 'one':6 'ran':8 'run':4 'runner':2,7 'yesterday':10
(1 row)
```

มาไล่ดูทีละส่วนว่าเกิดอะไรขึ้น:

| ตำแหน่ง | คำต้นฉบับ | ผลลัพธ์ |
|---------|-----------|---------|
| 1 | The | ตัดทิ้ง (stop word) |
| 2 | Runners | → `runner` (แปลงเป็นตัวพิมพ์เล็ก + ตัด -s พหูพจน์) |
| 3 | are | ตัดทิ้ง (stop word) |
| 4 | Running | → `run` (stemming ตัด -ning) |
| 5 | and | ตัดทิ้ง (stop word) |
| 6 | one | → `one` (คงเดิม) |
| 7 | Runner | → `runner` (ตำแหน่งที่ 2 ของ lexeme เดียวกับตำแหน่ง 2) |
| 8 | ran | → `ran` (**คงเดิม** — stemmer ไม่รู้จักคำอดีตกาลแบบ irregular) |
| 9 | fastest | → `fast` (ตัด -est ขั้นสุด) |
| 10 | yesterday | → `yesterday` (คงเดิม) |

สังเกต 3 ประเด็นสำคัญ:

- **`'runner':2,7`** — lexeme เดียวกันที่ปรากฏหลายตำแหน่งจะถูกรวมเป็นรายการ position เดียว ไม่ซ้ำ entry
- **ผลลัพธ์เรียงตามตัวอักษร** ของ lexeme ไม่ใช่ตามลำดับที่ปรากฏในข้อความต้นฉบับ (ตำแหน่งจริงถูกเก็บเป็นตัวเลขกำกับแทน)
- **`'ran':8` ไม่ถูกรวมเข้ากับ `'run':4`** — นี่คือข้อจำกัดที่ต้องรู้ไว้ (จะอธิบายเหตุผลเชิงลึกใน Step 525) stemmer ของ PostgreSQL (Snowball algorithm) ทำงานกับ**กฎการตัดปัจจัย/อุปสรรค** (suffix stripping) เท่านั้น ไม่ได้มี dictionary ของคำผันไม่ปกติ (irregular verb) ฝังอยู่

ลองใช้กับข้อมูลจริงในตารางของเรา:

```sql
SELECT product_name, to_tsvector('english', description) AS vec
FROM products
WHERE product_id = 1;
```

```
        product_name        |                                    vec (ย่อ)
-----------------------------+-------------------------------------------------------------------------
 TrailBlazer Running Shoes   | 'aggress':13 'ath':10 'built':1 'carri':9 'countless':11 'earli':17
                             | 'grip':21 'keep':27 'long':8 'marathon':12 'morn':18 'mudd':24 'multipl':7
                             | 'path':25 'rock':23 'rugged':6 'run':2,5,15,29 'safe':28 'say':32
                             | 'train':19 'trail':4 'ultramarathon':36 'weather':30 ...
(1 row)
```

จะเห็นว่า "runners", "running" (x2), "runs" ทั้งหมดถูก stem ลงมาเป็น `'run'` และรวมตำแหน่งไว้ในรายการเดียว `'run':2,5,15,29` — นี่คือกลไกที่ทำให้ full text search มองเห็นว่าคำเหล่านี้ "เกี่ยวข้องกัน" โดยอัตโนมัติ ต่างจาก `ILIKE` ที่ต้องเทียบตัวอักษรตรง ๆ

> **หมายเหตุ:** `to_tsvector('english', ...)` คือการระบุ **Text Search Configuration** ชื่อ `'english'` อย่างชัดเจน ถ้าไม่ระบุ PostgreSQL จะใช้ค่าใน `default_text_search_config` (ปกติคือ `pg_catalog.english` เมื่อ locale เป็น English) เพื่อความชัดเจนและพกพาข้าม server ได้ ควรระบุ config เสมอในโค้ด production

---

## Step 523: tsquery คืออะไร — การสร้างคำค้นหา

`tsvector` คือ "เอกสารที่ผ่านการประมวลผลแล้ว" ส่วน `tsquery` คือ **"คำค้นหาที่ผ่านการประมวลผลแล้ว"** — ทั้งสองฝั่งต้องผ่านกระบวนการ normalize แบบเดียวกันเพื่อให้เทียบกันได้ตรง ๆ PostgreSQL มีฟังก์ชันสร้าง `tsquery` สามแบบหลัก แต่ละแบบเหมาะกับสถานการณ์ต่างกัน

### 3.1 `to_tsquery` — ควบคุมเต็มรูปแบบด้วย operator

ใช้ operator ของตัวเอง: `&` (AND), `|` (OR), `!` (NOT), `<->` (FOLLOWED BY — ติดกันตามลำดับ), `:*` (prefix matching)

```sql
SELECT to_tsquery('english', 'running & shoes');
```

```
      to_tsquery
------------------------
 'run' & 'shoe'
(1 row)
```

```sql
SELECT to_tsquery('english', 'running <-> shoes');
```

```
       to_tsquery
--------------------------
 'run' <-> 'shoe'
(1 row)
```

`'run' <-> 'shoe'` หมายถึง "คำที่ stem เป็น run ต้องอยู่ติดกันทันทีก่อนคำที่ stem เป็น shoe" — ใช้สำหรับค้นหาวลี (phrase search) แบบเข้มงวด

```sql
SELECT to_tsquery('english', 'run:* & !walk');
```

```
       to_tsquery
------------------------
 'run':* & !'walk'
(1 row)
```

`run:*` คือ **prefix matching** — จับคำใดก็ตามที่ stem แล้วขึ้นต้นด้วย "run" (เช่น run, runner, running) ส่วน `!walk` คือ "ต้องไม่มีคำว่า walk" ข้อควรระวังคือถ้าเขียน syntax ผิด (เช่น `to_tsquery('english', 'running shoes')` โดยไม่ใส่ operator) จะเกิด error:

```sql
SELECT to_tsquery('english', 'running shoes');
```

```
ERROR:  syntax error in tsquery: "running shoes"
```

`to_tsquery` **ไม่ยอมให้ใส่คำหลายคำโดยไม่มี operator เชื่อม** — นี่คือเหตุผลที่มีฟังก์ชันอีกสองตัวสำหรับกรณีที่ผู้ใช้พิมพ์คำค้นหาแบบอิสระ

### 3.2 `plainto_tsquery` — แปลงข้อความธรรมดาเป็น AND query

เหมาะกับกรณีที่รับ input จากผู้ใช้แบบ free text แล้วต้องการให้ **ทุกคำต้องปรากฏ** (AND) โดยไม่สนใจ operator พิเศษที่ผู้ใช้อาจพิมพ์มา (จะถูกมองเป็นคำธรรมดาทั้งหมด):

```sql
SELECT plainto_tsquery('english', 'running shoes for marathon training');
```

```
                 plainto_tsquery
----------------------------------------------------
 'run' & 'shoe' & 'marathon' & 'train'
(1 row)
```

สังเกตว่าคำว่า "for" หายไป (stop word) และทุกคำที่เหลือถูกเชื่อมด้วย `&` โดยอัตโนมัติ

### 3.3 `websearch_to_tsquery` — syntax แบบเว็บค้นหาทั่วไป (PostgreSQL 11+)

ฟังก์ชันนี้เข้าใจ syntax ที่คนทั่วไปคุ้นเคยจาก search engine อย่าง Google: ใช้เครื่องหมายคำพูดสำหรับวลี, `OR` แบบตัวพิมพ์ใหญ่, และ `-` นำหน้าเพื่อ exclude คำ — และที่สำคัญ **มันไม่ error แม้ syntax จะแปลก ๆ** (ต่างจาก `to_tsquery` ที่ error ทันทีถ้าเขียนผิด)

```sql
SELECT websearch_to_tsquery('english', '"running shoes" -waterproof OR sandals');
```

```
                          websearch_to_tsquery
--------------------------------------------------------------------------
 ( 'run' <-> 'shoe' & !'waterproof' ) | 'sandal'
(1 row)
```

จะเห็นว่า:
- `"running shoes"` (มีเครื่องหมายคำพูดคร่อม) ถูกแปลงเป็น phrase search `'run' <-> 'shoe'`
- `-waterproof` กลายเป็น `!'waterproof'` (exclude)
- `OR` กลายเป็น `|`

เปรียบเทียบทั้งสามฟังก์ชันด้วยตัวอย่างเดียวกัน:

```sql
SELECT
    to_tsquery('english', 'run:* & shoe')       AS via_to_tsquery,
    plainto_tsquery('english', 'running shoes') AS via_plainto,
    websearch_to_tsquery('english', 'running shoes') AS via_websearch;
```

```
   via_to_tsquery   |     via_plainto      |    via_websearch
---------------------+----------------------+----------------------
 'run':* & 'shoe'    | 'run' & 'shoe'       | 'run' & 'shoe'
(1 row)
```

**คำแนะนำเชิงปฏิบัติ:** สำหรับช่องค้นหาที่รับ input จากผู้ใช้ทั่วไปในเว็บแอป ให้ใช้ `websearch_to_tsquery` เป็นค่าเริ่มต้น เพราะทนทานต่อ input แปลก ๆ ที่สุดและผู้ใช้ยังพิมพ์ syntax ขั้นสูงได้ถ้าต้องการ (`to_tsquery` เหมาะกับกรณีที่ query ถูกสร้างจาก backend logic ของเราเองที่ควบคุม syntax ได้แน่นอน)

---

## Step 524: Match operator @@ — การจับคู่ tsvector กับ tsquery

operator `@@` คือหัวใจของ full text search — เปรียบเทียบว่า `tsvector` มี lexeme ที่ตรงกับเงื่อนไขใน `tsquery` หรือไม่ คืนค่า `boolean`

```sql
SELECT to_tsvector('english', 'running shoes for daily training')
       @@ to_tsquery('english', 'run & shoe') AS is_match;
```

```
 is_match
----------
 t
(1 row)
```

นำมาใช้ค้นหาสินค้าจริง:

```sql
SELECT product_id, product_name
FROM products
WHERE to_tsvector('english', description) @@ to_tsquery('english', 'run:*')
ORDER BY product_id;
```

```
 product_id |            product_name
------------+--------------------------------------
          1 | TrailBlazer Running Shoes
          2 | Urban Sprint Running Shoes
          3 | Marathon Pro Running Shoes
          7 | True Wireless Earbuds Pro
         14 | Mirrorless Camera Z5
         16 | Fitness Smartwatch Pulse
         27 | Waterproof Rain Jacket
(7 rows)
```

ผลลัพธ์ตรงกับที่ได้จาก `ILIKE '%running%'` ใน Step 521 — แต่คราวนี้ค้นด้วยหลักการที่ถูกต้อง (prefix `run:*` จับทั้ง run/running/runner/runs) ไม่ใช่บังเอิญเหมือน `ILIKE`

ลองรวมทั้ง `product_name` และ `description` เข้าด้วยกันในการค้นหา (ผู้ใช้อาจพิมพ์คำที่อยู่ในชื่อสินค้า ไม่ใช่แค่ในคำอธิบาย):

```sql
SELECT product_id, product_name
FROM products
WHERE to_tsvector('english', product_name || ' ' || coalesce(description, ''))
      @@ websearch_to_tsquery('english', 'coffee OR blender')
ORDER BY product_id;
```

```
 product_id |      product_name
------------+--------------------------
         21 | Drip Coffee Maker Deluxe
         22 | High-Speed Blender Pro
(2 rows)
```

ค้นหาในตาราง `reviews` แบบเดียวกัน — หารีวิวที่พูดถึง "battery" แต่มีความรู้สึกเชิงลบ (ประมาณด้วยคำว่า "disappoint" หรือ "drain"):

```sql
SELECT review_id, product_id, rating, review_text
FROM reviews
WHERE to_tsvector('english', review_text)
      @@ to_tsquery('english', 'battery & (disappoint | drain)');
```

```
 review_id | product_id | rating |                       review_text
-----------+------------+--------+-----------------------------------------------------------
         9 |          5 |      2 | Sound quality is good but the battery drains faster than
           |            |        | advertised. Disappointed after only three months of use.
(1 row)
```

> **ข้อควรระวังเรื่อง performance:** ทุก query ด้านบนเรียก `to_tsvector(...)` **ซ้ำใหม่ทุกแถวทุกครั้งที่ query รัน** เพราะเราไม่ได้เก็บ `tsvector` ไว้ล่วงหน้า วิธีนี้ใช้ได้กับข้อมูลน้อย ๆ อย่างในบทเรียนนี้ แต่กับข้อมูลระดับแสน-ล้านแถวจะช้ามาก เราจะแก้ปัญหานี้ด้วย generated column และ GIN index ใน Step 528-529

---

## Step 525: Text Search Configuration — 'english', stop words และ stemming

**Text Search Configuration** คือชุดกฎที่กำหนดว่าข้อความภาษาหนึ่ง ๆ ควรถูก parse และ normalize อย่างไร ดูรายการ configuration ที่มีในเครื่องด้วย:

```sql
\dF
```

```
                          List of text search configurations
   Schema   |     Name     |                    Description
------------+--------------+----------------------------------------------------
 pg_catalog | arabic       | configuration for arabic language
 pg_catalog | danish       | configuration for danish language
 ...
 pg_catalog | english      | configuration for english language
 ...
 pg_catalog | simple       | simple configuration
```

```sql
SHOW default_text_search_config;
```

```
 default_text_search_config
------------------------------
 pg_catalog.english
(1 row)
```

### stop words

**Stop words** คือคำที่พบบ่อยมากจนไม่มีคุณค่าในการค้นหา (the, a, an, is, are, of, for, ...) — configuration `'english'` จะตัดคำเหล่านี้ออกจาก `tsvector` โดยอัตโนมัติ ใช้ `ts_debug` เพื่อดูรายละเอียดว่าแต่ละคำถูกจัดประเภทและประมวลผลอย่างไร:

```sql
SELECT alias, token, dictionaries, lexemes
FROM ts_debug('english', 'The Runners are Running and one Runner ran fastest yesterday');
```

```
  alias   |    token    |     dictionaries      |  lexemes
----------+-------------+-----------------------+-------------
 asciiword| The         | {english_stem}        | {}
 blank    |             | {}                     |
 asciiword| Runners     | {english_stem}        | {runner}
 blank    |             | {}                     |
 asciiword| are         | {english_stem}        | {}
 blank    |             | {}                     |
 asciiword| Running     | {english_stem}        | {run}
 blank    |             | {}                     |
 asciiword| and         | {english_stem}        | {}
 blank    |             | {}                     |
 asciiword| one         | {english_stem}        | {one}
 blank    |             | {}                     |
 asciiword| Runner      | {english_stem}        | {runner}
 blank    |             | {}                     |
 asciiword| ran         | {english_stem}        | {ran}
 blank    |             | {}                     |
 asciiword| fastest     | {english_stem}        | {fast}
 blank    |             | {}                     |
 asciiword| yesterday   | {english_stem}        | {yesterday}
(17 rows)
```

คำที่ `lexemes` เป็น `{}` (ว่างเปล่า) เช่น "The", "are", "and" คือ **stop words** ที่ dictionary `english_stem` ตัดสินใจว่าไม่มีความหมายพอที่จะเก็บไว้ ส่วนคำอื่น ๆ ทุกคำถูก stem แล้ว โดย **"ran" ยังคงเป็น "ran"** ยืนยันสิ่งที่พูดถึงใน Step 522

### stemming

**Stemming** คือกระบวนการตัดคำให้เหลือ "รากศัพท์" (root form) โดยใช้อัลกอริทึม **Snowball stemmer** — ทำงานแบบ **ตัดปัจจัย/คำต่อท้ายตามกฎภาษา** (rule-based suffix stripping) ไม่ใช่การเปิด dictionary คำต่อคำ ข้อดีคือครอบคลุมคำใหม่ ๆ ได้ (ไม่ต้องมีคำนั้นอยู่ใน dictionary มาก่อน) แต่ข้อเสียคือไม่รู้จักคำผันแบบ irregular (ran, went, better, best ฯลฯ)

เปรียบเทียบ configuration `'simple'` (ไม่ stem ไม่ตัด stop word) กับ `'english'`:

```sql
SELECT
    to_tsvector('simple',  'The runners are running quickly') AS simple_config,
    to_tsvector('english', 'The runners are running quickly') AS english_config;
```

```
             simple_config              |      english_config
-----------------------------------------+---------------------------
 'are':3 'quickly':5 'running':4         | 'quick':4 'run':3 'runner':2
 'runners':2 'the':1                    |
(1 row)
```

`'simple'` เก็บทุกคำแบบตัวพิมพ์เล็กตรง ๆ (แม้แต่ "the" และ "are" ก็ไม่ตัดทิ้ง) ในขณะที่ `'english'` ตัด stop word และ stem คำให้เหลือรากศัพท์ — ทำให้ `'english'` มีขนาดเล็กกว่าและจับคู่คำที่มีความหมายเดียวกันได้กว้างกว่า แต่ `'simple'` เหมาะกับกรณีที่ต้องการค้นหาแบบตรงตัวเป๊ะ ๆ เช่น รหัสสินค้า, username หรือข้อความที่ไม่ใช่ภาษาธรรมชาติ

> **สรุปข้อจำกัดที่ควรจำไว้:** Full Text Search แก้ปัญหา "run/running/runs" ได้อย่างสมบูรณ์แบบ เพราะเป็นการผันแบบ regular ที่ตัดปัจจัยได้ตรงไปตรงมา แต่การค้นหาด้วยคำว่า `'run'` **จะไม่พบ** รีวิวที่มีแต่คำว่า "ran" ล้วน ๆ โดยไม่มีคำอื่นในตระกูล run ปนอยู่ ถ้าต้องการครอบคลุมกรณีนี้ด้วย ต้องใช้ query ที่ระบุคำ irregular เพิ่มเอง เช่น `to_tsquery('english', 'run | ran')` — นี่คือความรู้เชิงลึกที่แยกผู้เชี่ยวชาญออกจากผู้ใช้งานทั่วไปที่คิดว่า full text search "วิเศษ" จนครอบคลุมทุกกรณี

---

## Step 526: Ranking ผลลัพธ์ด้วย ts_rank และ ts_rank_cd

การมีแค่ match/ไม่ match (`@@`) ยังไม่พอสำหรับการค้นหาแบบมืออาชีพ — เราต้องการรู้ด้วยว่าแถวไหน **"เกี่ยวข้องมากกว่า"** แถวอื่น PostgreSQL มีฟังก์ชัน `ts_rank(vector, query)` ที่คำนวณคะแนนความเกี่ยวข้องจากหลายปัจจัย เช่น จำนวนครั้งที่คำค้นหาปรากฏ, ความหนาแน่นของคำที่ match ในเอกสาร

### ให้น้ำหนักต่างกันด้วย setweight

ก่อนจะจัดอันดับ เรามักอยากให้คำที่ match ใน **ชื่อสินค้า** มีน้ำหนักมากกว่าคำที่ match ใน **คำอธิบาย** — ทำได้ด้วย `setweight()` ซึ่งกำหนด label น้ำหนักเป็น `A` (สูงสุด), `B`, `C`, `D` (ต่ำสุด) ให้แต่ละส่วนของ tsvector ก่อนนำมารวมกันด้วย `||`:

```sql
SELECT
    product_id,
    product_name,
    ts_rank(
        setweight(to_tsvector('english', product_name), 'A') ||
        setweight(to_tsvector('english', coalesce(description, '')), 'B'),
        websearch_to_tsquery('english', 'running shoes')
    ) AS rank
FROM products
WHERE (setweight(to_tsvector('english', product_name), 'A') ||
       setweight(to_tsvector('english', coalesce(description, '')), 'B'))
      @@ websearch_to_tsquery('english', 'running shoes')
ORDER BY rank DESC
LIMIT 10;
```

```
 product_id |         product_name          |   rank
------------+--------------------------------+-----------
          2 | Urban Sprint Running Shoes     | 0.1955654
          1 | TrailBlazer Running Shoes      | 0.1216201
          3 | Marathon Pro Running Shoes     | 0.1216201
          7 | True Wireless Earbuds Pro      | 0.0405843
         14 | Mirrorless Camera Z5           | 0.0303956
         16 | Fitness Smartwatch Pulse       | 0.0303956
         27 | Waterproof Rain Jacket         | 0.0303956
(7 rows)
```

ผลลัพธ์ตอนนี้มีความหมายมากขึ้นมาก: 3 สินค้าที่ชื่อมีคำว่า "Running Shoes" ตรง ๆ ขึ้นมาอยู่บนสุดเพราะ match ทั้งใน weight `A` (ชื่อ) และ `B` (คำอธิบาย) ในขณะที่สินค้าที่แค่ **กล่าวถึง** การวิ่งในคำอธิบาย (หูฟัง, กล้อง, สมาร์ทวอทช์, เสื้อกันฝน) ได้คะแนนต่ำกว่าอย่างชัดเจนเพราะ match แค่ weight `B` เท่านั้น — และ **product_id 2 (Urban Sprint)** ได้คะแนนสูงสุดเพราะคำว่า "running" ปรากฏซ้ำในคำอธิบายมากกว่าตัวอื่น (running ปรากฏ 3 ครั้ง)

น้ำหนักเริ่มต้นของ `ts_rank` คือ `{D: 0.1, C: 0.2, B: 0.4, A: 1.0}` — ปรับเองได้โดยส่ง array เป็นพารามิเตอร์แรก:

```sql
SELECT ts_rank('{0.1, 0.2, 0.4, 1.0}',
    setweight(to_tsvector('english', 'Running Shoes'), 'A'),
    websearch_to_tsquery('english', 'running')
);
```

### ts_rank_cd — Cover Density Ranking

`ts_rank_cd` (cover density) พิจารณา**ระยะห่างระหว่างคำที่ match กัน**ด้วย — ถ้าคำค้นหาหลายคำอยู่**ใกล้กัน**ในเอกสาร จะได้คะแนนสูงกว่าเอกสารที่คำเหล่านั้นกระจายอยู่ไกลกัน เหมาะมากเมื่อค้นหาด้วยหลายคำพร้อมกัน:

```sql
SELECT
    review_id,
    product_id,
    ts_rank_cd(to_tsvector('english', review_text),
               plainto_tsquery('english', 'running marathon shoes')) AS rank_cd
FROM reviews
WHERE to_tsvector('english', review_text)
      @@ plainto_tsquery('english', 'running marathon shoes')
ORDER BY rank_cd DESC;
```

```
 review_id | product_id |  rank_cd
-----------+------------+-----------
         1 |          1 | 0.4
         5 |          3 |  0.2
         2 |          1 |  0.1
(3 rows)
```

review_id 1 (*"I ran my first half marathon in these shoes..."*) ได้คะแนนสูงสุดเพราะคำว่า "ran", "marathon", "shoes" อยู่ใกล้กันในประโยคเดียว ในขณะที่ review_id 2 มีคำว่า "running" และ "shoes" อยู่ แต่คำว่า marathon ไม่ปรากฏเลย (ได้คะแนนจาก partial match เท่านั้น)

> **แนวทางเลือกใช้:** `ts_rank` เหมาะกับกรณีทั่วไปและคำนวณเร็วกว่า ส่วน `ts_rank_cd` เหมาะเมื่อผู้ใช้ค้นหาด้วยหลายคำและต้องการให้เอกสารที่ "พูดถึงหลายคำนั้นในบริบทเดียวกัน" ขึ้นมาก่อนเอกสารที่แค่มีคำครบแต่กระจัดกระจาย

---

## Step 527: ts_headline — การไฮไลท์คำที่ match ในผลลัพธ์

เวลาค้นหาบน Google ผลลัพธ์แต่ละอันจะมี snippet สั้น ๆ ที่ไฮไลท์คำค้นหาไว้ให้เห็นบริบท — PostgreSQL ทำแบบเดียวกันได้ด้วย `ts_headline(config, document, query, options)`

```sql
SELECT
    product_name,
    ts_headline('english', description,
        websearch_to_tsquery('english', 'running marathon'),
        'StartSel=<b>, StopSel=</b>, MaxWords=20, MinWords=8'
    ) AS snippet
FROM products
WHERE to_tsvector('english', description)
      @@ websearch_to_tsquery('english', 'running marathon')
ORDER BY product_id
LIMIT 3;
```

```
        product_name        |                                    snippet
-----------------------------+---------------------------------------------------------------------------------
 TrailBlazer Running Shoes   | the TrailBlazer has carried athletes through countless <b>marathons</b> and
                             | early morning training <b>runs</b>
 Urban Sprint Running Shoes  | designed for daily <b>runs</b> on city pavement. Whether you are jogging to
                             | the office or <b>running</b> interval sprints
 Marathon Pro Running Shoes  | Engineered specifically for <b>marathon</b> <b>runners</b> chasing a personal
                             | best, this shoe combines a responsive foam midsole
(3 rows)
```

สิ่งที่น่าสนใจคือ `ts_headline` ไฮไลท์ **คำต้นฉบับ** ที่ปรากฏจริงในข้อความ (เช่น "marathons", "runs", "running", "runners") ไม่ใช่ lexeme ที่ stem แล้ว — มันรู้ว่าคำเหล่านี้ stem แล้วตรงกับคำค้นหา จึงไฮไลท์รูปแบบดั้งเดิมให้ผู้อ่านเห็นเป็นธรรมชาติ

ปรับแต่ง option เพิ่มเติมได้หลายตัว เช่น `MaxFragments` (จำนวน snippet ย่อยสูงสุด) และ `FragmentDelimiter` (ตัวคั่นระหว่าง snippet):

```sql
SELECT ts_headline('english', review_text,
    to_tsquery('english', 'battery & disappoint'),
    'StartSel=**, StopSel=**, MaxFragments=2, FragmentDelimiter= ... '
) AS snippet
FROM reviews
WHERE review_id = 9;
```

```
                                    snippet
--------------------------------------------------------------------------------
 the **battery** drains faster than advertised. **Disappointed** after only
 three months of use.
(1 row)
```

`ts_headline` มีต้นทุนการประมวลผลสูงกว่า `ts_rank` มาก เพราะต้อง parse และวิเคราะห์ข้อความต้นฉบับใหม่ทุกครั้ง (ไม่สามารถใช้ tsvector ที่ index ไว้ล่วงหน้าช่วยได้โดยตรง) — แนวทางปฏิบัติที่ดีคือ **ใช้ ranking (ts_rank) เพื่อคัดกรองและเรียงลำดับผลลัพธ์ให้เหลือแค่ TOP N ก่อน แล้วค่อยเรียก `ts_headline` เฉพาะ N แถวนั้น** ไม่ใช่เรียกกับทุกแถวในตาราง

---

## Step 528: Generated Column สำหรับเก็บ tsvector ล่วงหน้า

ทุก query ที่ผ่านมาเรียก `to_tsvector(...)` **สดใหม่ทุกครั้ง** ที่รัน query ซึ่งหมายความว่า PostgreSQL ต้อง parse และ stem ข้อความทุกแถวใหม่เสมอ ไม่ว่าข้อความนั้นจะเปลี่ยนหรือไม่ก็ตาม วิธีแก้คือ **เก็บผลลัพธ์ tsvector ไว้ล่วงหน้าในคอลัมน์** แล้วสร้าง index บนคอลัมน์นั้น

### วิธีสมัยใหม่: Generated Column (PostgreSQL 12+)

```sql
ALTER TABLE products
    ADD COLUMN search_vector tsvector
    GENERATED ALWAYS AS (
        setweight(to_tsvector('english', coalesce(product_name, '')), 'A') ||
        setweight(to_tsvector('english', coalesce(description, '')), 'B')
    ) STORED;
```

```sql
ALTER TABLE reviews
    ADD COLUMN search_vector tsvector
    GENERATED ALWAYS AS (
        to_tsvector('english', coalesce(review_text, ''))
    ) STORED;
```

ตรวจสอบผลลัพธ์:

```sql
SELECT product_id, product_name, search_vector
FROM products
WHERE product_id = 2;
```

```
 product_id |        product_name         |                          search_vector (ย่อ)
------------+------------------------------+-------------------------------------------------------------
          2 | Urban Sprint Running Shoes   | 'daili':10A 'mesh':22B 'run':6A,15B,25B ... 'urban':5A ...
(1 row)
```

สังเกตตัวอักษร **`A` และ `B`** ต่อท้ายตำแหน่งแต่ละ lexeme — บอกว่า lexeme นั้นมาจากส่วนที่ตั้งน้ำหนัก `A` (product_name) หรือ `B` (description) ทำให้ `ts_rank` คำนวณคะแนนตามน้ำหนักได้โดยไม่ต้องคำนวณ `setweight` ซ้ำตอน query

จุดสำคัญของ `GENERATED ALWAYS AS ... STORED`:

- คอลัมน์นี้ **อัปเดตอัตโนมัติ** ทุกครั้งที่ `INSERT` หรือ `UPDATE` แถวนั้น (แม้จะไม่ได้ระบุค่าให้คอลัมน์นี้ตรง ๆ)
- ผู้ใช้ **ไม่สามารถ INSERT/UPDATE คอลัมน์นี้ตรง ๆ ได้เอง** — ป้องกัน tsvector เพี้ยนจากการ sync ผิดพลาด
- ฟังก์ชันที่ใช้ในนิยาม (`to_tsvector`, `setweight`, `coalesce`, `||`) ต้องเป็น **IMMUTABLE** เท่านั้น การเรียก `to_tsvector('english', ...)` โดยระบุชื่อ configuration เป็น string literal ตรง ๆ ถือว่าปลอดภัยและใช้งานได้ (ถ้าใช้ตัวแปร regconfig ที่มาจากอย่างอื่นอาจถูกปฏิเสธเพราะไม่รับประกันว่า immutable)
- ลองแก้ description แล้วดูว่า search_vector เปลี่ยนตามทันที:

```sql
UPDATE products SET description = description || ' Also great for sprint training.'
WHERE product_id = 2;

SELECT search_vector @@ to_tsquery('english', 'sprint & train') AS matches_now
FROM products WHERE product_id = 2;
```

```
 matches_now
-------------
 t
(1 row)
```

### วิธีดั้งเดิม: Trigger-based update

ก่อน PostgreSQL 12 (ที่ยังไม่มี generated column) วิธีมาตรฐานคือใช้ trigger เพื่ออัปเดต tsvector เอง PostgreSQL มี trigger function สำเร็จรูปชื่อ `tsvector_update_trigger` ให้ใช้ได้ทันที:

```sql
-- ตัวอย่างแนวทางเดิม (ไม่จำเป็นต้องรันถ้าใช้ generated column แล้ว)
ALTER TABLE products ADD COLUMN search_vector_legacy tsvector;

CREATE TRIGGER trg_products_search_vector_legacy
    BEFORE INSERT OR UPDATE ON products
    FOR EACH ROW
    EXECUTE FUNCTION tsvector_update_trigger(
        search_vector_legacy, 'pg_catalog.english', product_name, description
    );
```

หรือเขียน trigger function เองเพื่อควบคุมน้ำหนักแบบละเอียด (ทำสิ่งที่ generated column ทำได้เหมือนกันทุกประการ):

```sql
CREATE OR REPLACE FUNCTION products_search_vector_update() RETURNS trigger AS $$
BEGIN
    NEW.search_vector_legacy :=
        setweight(to_tsvector('english', coalesce(NEW.product_name, '')), 'A') ||
        setweight(to_tsvector('english', coalesce(NEW.description, '')), 'B');
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_products_search_vector_custom
    BEFORE INSERT OR UPDATE ON products
    FOR EACH ROW
    EXECUTE FUNCTION products_search_vector_update();
```

```sql
-- ล้างตัวอย่าง legacy ทิ้ง เพราะบทนี้จะใช้ generated column ต่อไป
DROP TRIGGER IF EXISTS trg_products_search_vector_legacy ON products;
DROP TRIGGER IF EXISTS trg_products_search_vector_custom ON products;
DROP FUNCTION IF EXISTS products_search_vector_update();
ALTER TABLE products DROP COLUMN IF EXISTS search_vector_legacy;
```

### เปรียบเทียบสองแนวทาง

| ประเด็น | Generated Column (STORED) | Trigger |
|---------|---------------------------|---------|
| ความซับซ้อนของโค้ด | ต่ำ — ประกาศครั้งเดียวใน `ALTER TABLE`/`CREATE TABLE` | สูงกว่า — ต้องเขียนทั้ง function และ trigger แยก |
| ความเสี่ยงลืม sync | ไม่มี — engine รับประกันว่าอัปเดตเสมอ | มีถ้าลืมสร้าง trigger ให้ครบทุกเหตุการณ์ |
| ดึงข้อมูลจากตารางอื่น (เช่น category_name) | **ทำไม่ได้** — generated column ต้องคำนวณจากคอลัมน์ในแถวเดียวกันเท่านั้น | **ทำได้** — trigger เขียน query ข้ามตารางได้อิสระ |
| ความเร็วตอน INSERT/UPDATE | เร็ว (คำนวณในตัว engine) | มี overhead เล็กน้อยจากการเรียก PL/pgSQL function |
| แก้ logic ภายหลัง | ต้อง `DROP` แล้ว `ADD COLUMN` ใหม่ทั้งคอลัมน์ | แก้แค่ตัว function ได้โดยไม่กระทบ schema |

**สรุป:** ถ้าข้อมูลที่ใช้ทำ tsvector อยู่ในตารางเดียวกันทั้งหมด (กรณีส่วนใหญ่) ให้ใช้ **generated column** เป็นค่าเริ่มต้นเสมอ เพราะเรียบง่ายและปลอดภัยกว่า เก็บแนวทาง trigger ไว้ใช้เฉพาะกรณีพิเศษที่ต้องรวมข้อมูลจากตารางอื่นเข้ามาด้วย (เช่น อยากให้ค้นหาสินค้าเจอจากชื่อ category ด้วย)

---

## Step 529: GIN Index บน tsvector column

มี generated column แล้ว แต่ query ยังคง **Sequential Scan** อยู่ถ้าไม่มี index ลอง `EXPLAIN ANALYZE` ก่อนสร้าง index:

```sql
EXPLAIN ANALYZE
SELECT product_id, product_name
FROM products
WHERE search_vector @@ websearch_to_tsquery('english', 'running shoes');
```

```
                                              QUERY PLAN
--------------------------------------------------------------------------------------------------
 Seq Scan on products  (cost=0.00..2.38 rows=1 width=36) (actual time=0.045..0.112 rows=3 loops=1)
   Filter: (search_vector @@ '''run'' & ''shoe'''::tsquery)
   Rows Removed by Filter: 27
 Planning Time: 0.180 ms
 Execution Time: 0.135 ms
(5 rows)
```

ตอนนี้สร้าง GIN Index บน `search_vector`:

```sql
CREATE INDEX idx_products_search_vector ON products USING GIN (search_vector);
CREATE INDEX idx_reviews_search_vector   ON reviews  USING GIN (search_vector);
```

```sql
EXPLAIN ANALYZE
SELECT product_id, product_name
FROM products
WHERE search_vector @@ websearch_to_tsquery('english', 'running shoes');
```

```
                                              QUERY PLAN
--------------------------------------------------------------------------------------------------
 Seq Scan on products  (cost=0.00..2.38 rows=1 width=36) (actual time=0.040..0.098 rows=3 loops=1)
   Filter: (search_vector @@ '''run'' & ''shoe'''::tsquery)
   Rows Removed by Filter: 27
 Planning Time: 0.210 ms
 Execution Time: 0.121 ms
(5 rows)
```

**หมายเหตุสำคัญที่ต้องเข้าใจ (ไม่ใช่บั๊กของ index):** ด้วยข้อมูลแค่ 30 แถวในบทเรียนนี้ query planner ยังคง**เลือก Sequential Scan ต่อไป** แม้ index จะถูกสร้างแล้วก็ตาม เพราะการอ่านตารางทั้งหมดที่มีแค่ไม่กี่ block นั้นเร็วกว่าการเปิด index แล้ววกกลับไปอ่านตาราง (bitmap heap scan) เสมอ — planner ตัดสินใจจาก **cost estimation** ไม่ใช่จากการมี index อยู่เพียงอย่างเดียว (เช่นเดียวกับที่อธิบายไว้ในบท B-tree Index ก่อนหน้า) เมื่อข้อมูลมีจำนวนมากขึ้นถึงหลักหมื่นถึงหลักล้านแถว planner จะเลือกใช้ index โดยอัตโนมัติทันทีที่คุ้มค่ากว่า

พิสูจน์ได้โดยบังคับปิด sequential scan ชั่วคราว (ใช้เพื่อการศึกษาเท่านั้น ห้ามใช้ใน production):

```sql
SET enable_seqscan = off;

EXPLAIN ANALYZE
SELECT product_id, product_name
FROM products
WHERE search_vector @@ websearch_to_tsquery('english', 'running shoes');
```

```
                                                    QUERY PLAN
--------------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on products  (cost=8.02..12.29 rows=1 width=36) (actual time=0.055..0.062 rows=3 loops=1)
   Recheck Cond: (search_vector @@ '''run'' & ''shoe'''::tsquery)
   Heap Blocks: exact=3
   ->  Bitmap Index Scan on idx_products_search_vector  (cost=0.00..8.02 rows=1 width=0)
         (actual time=0.041..0.042 rows=3 loops=1)
         Index Cond: (search_vector @@ '''run'' & ''shoe'''::tsquery)
 Planning Time: 0.230 ms
 Execution Time: 0.089 ms
(7 rows)
```

```sql
RESET enable_seqscan;
```

จะเห็นแผนเปลี่ยนเป็น **Bitmap Index Scan** บน `idx_products_search_vector` ตามด้วย **Bitmap Heap Scan** พร้อม **Recheck Cond** — GIN index เก็บ "signature" ของแต่ละ lexeme ทำให้ค้นหาได้เร็ว แต่บาง operator (เช่น phrase search `<->`) ต้องย้อนกลับไปตรวจสอบข้อมูลจริงในตาราง (recheck) เพื่อยืนยันผลอีกครั้ง

### ทำไมต้อง GIN ไม่ใช่ B-tree หรือ Hash

`tsvector` เก็บ**คำหลายคำในค่าเดียว** B-tree เปรียบเทียบค่าทั้งก้อนแบบ ordering ไม่เหมาะกับการค้นหา "มี lexeme ใดอยู่บ้าง" ส่วน GIN (Generalized Inverted Index) ถูกออกแบบมาสำหรับข้อมูลที่ **แต่ละแถวมีหลาย "องค์ประกอบย่อย"** (composite value) โดยเฉพาะ — มันสร้าง index แยกต่างหากสำหรับ**แต่ละ lexeme** แล้วชี้กลับไปยังทุกแถวที่มี lexeme นั้น เหมือนดัชนีท้ายเล่มหนังสือ

เปรียบเทียบ GIN กับ GiST สำหรับ full text search โดยย่อ:

| ประเด็น | GIN | GiST |
|---------|-----|------|
| ความเร็วในการค้นหา | เร็วกว่า (แม่นยำ, ไม่ lossy) | ช้ากว่าเล็กน้อย (เป็น lossy signature, ต้อง recheck เสมอ) |
| ความเร็วในการ build/update index | ช้ากว่า, ใช้พื้นที่มากกว่า | เร็วกว่า, กะทัดรัดกว่า |
| เหมาะกับ | ตารางอ่านบ่อย เขียนไม่บ่อย (ค้นหาสินค้า, บทความ) | ตารางเขียนถี่มาก (log แบบ real-time) |

สำหรับระบบค้นหาสินค้า/รีวิวทั่วไปที่อ่านมากกว่าที่เขียนมาก **GIN คือตัวเลือกเริ่มต้นที่ถูกต้องเกือบทุกกรณี**

---

## Step 530: แบบฝึกหัดรวม — ระบบค้นหาสินค้าและรีวิวแบบเต็มรูปแบบ

มารวมทุกอย่างเป็นฟังก์ชันค้นหาที่ใช้งานได้จริง: รับคำค้นหาจากผู้ใช้ ค้นทั้งสินค้าและรีวิวที่เกี่ยวข้อง จัดอันดับด้วย `ts_rank` และไฮไลท์ผลลัพธ์ด้วย `ts_headline`

```sql
CREATE OR REPLACE FUNCTION search_products(p_keyword TEXT, p_limit INTEGER DEFAULT 10)
RETURNS TABLE (
    product_id     INTEGER,
    product_name   VARCHAR,
    unit_price     NUMERIC,
    avg_rating     NUMERIC,
    review_count   BIGINT,
    relevance      REAL,
    highlighted    TEXT
) AS $$
    WITH query AS (
        SELECT websearch_to_tsquery('english', p_keyword) AS tsq
    ),
    review_stats AS (
        SELECT
            r.product_id,
            round(avg(r.rating), 1) AS avg_rating,
            count(*)                AS review_count
        FROM reviews r
        GROUP BY r.product_id
    )
    SELECT
        p.product_id,
        p.product_name,
        p.unit_price,
        rs.avg_rating,
        coalesce(rs.review_count, 0),
        ts_rank(p.search_vector, q.tsq) AS relevance,
        ts_headline('english', p.description, q.tsq,
            'StartSel=<b>, StopSel=</b>, MaxWords=25, MinWords=10')
    FROM products p
    CROSS JOIN query q
    LEFT JOIN review_stats rs ON rs.product_id = p.product_id
    WHERE p.search_vector @@ q.tsq
      AND p.is_active
    ORDER BY relevance DESC
    LIMIT p_limit;
$$ LANGUAGE sql STABLE;
```

ทดสอบใช้งาน:

```sql
SELECT product_id, product_name, unit_price, avg_rating, review_count, relevance, highlighted
FROM search_products('running shoes');
```

```
 product_id |        product_name         | unit_price | avg_rating | review_count | relevance |                          highlighted
------------+------------------------------+------------+------------+---------------+-----------+---------------------------------------------------------------
          2 | Urban Sprint Running Shoes   |    2590.00 |        3.0 |             2 | 0.1955654 | A lightweight <b>running</b> <b>shoe</b> designed for daily
                                                                                                     | runs on city pavement
          1 | TrailBlazer Running Shoes    |    3290.00 |        4.5 |             2 | 0.1216201 | Built for runners who love hitting rugged trails, the
                                                                                                     | TrailBlazer has carried athletes
          3 | Marathon Pro Running Shoes   |    4590.00 |        4.0 |             1 | 0.1216201 | Engineered specifically for marathon runners chasing a
                                                                                                     | personal best
(3 rows)
```

ต่อยอดอีกขั้นด้วยการสร้าง view ค้นหารีวิวที่เกี่ยวข้องพร้อม highlight เช่นกัน เพื่อประกอบเป็นหน้าค้นหาแบบเต็มรูปแบบ (สินค้า + รีวิวที่เกี่ยวข้อง คล้ายผลลัพธ์บนเว็บอีคอมเมิร์ซจริง):

```sql
CREATE OR REPLACE FUNCTION search_reviews(p_keyword TEXT, p_limit INTEGER DEFAULT 10)
RETURNS TABLE (
    review_id    INTEGER,
    product_name VARCHAR,
    rating       INTEGER,
    relevance    REAL,
    highlighted  TEXT
) AS $$
    WITH query AS (
        SELECT websearch_to_tsquery('english', p_keyword) AS tsq
    )
    SELECT
        r.review_id,
        p.product_name,
        r.rating,
        ts_rank(r.search_vector, q.tsq),
        ts_headline('english', r.review_text, q.tsq,
            'StartSel=<b>, StopSel=</b>, MaxWords=25, MinWords=10')
    FROM reviews r
    CROSS JOIN query q
    JOIN products p ON p.product_id = r.product_id
    WHERE r.search_vector @@ q.tsq
    ORDER BY ts_rank(r.search_vector, q.tsq) DESC
    LIMIT p_limit;
$$ LANGUAGE sql STABLE;
```

```sql
SELECT review_id, product_name, rating, relevance, highlighted
FROM search_reviews('battery life');
```

```
 review_id |            product_name             | rating | relevance |                        highlighted
-----------+--------------------------------------+--------+-----------+-------------------------------------------------
         8 | Wireless Noise-Cancelling Headphones |      5 | 0.2107211 | <b>Battery</b> <b>life</b> easily lasts a full
                                                                        | transatlantic flight
         9 | Wireless Noise-Cancelling Headphones |      2 | 0.1013423 | the battery drains faster than advertised
        11 | True Wireless Earbuds Pro            |      5 | 0.0759096 | The charging case is compact and <b>battery</b>
                                                                        | <b>life</b> is impressive
(3 rows)
```

สังเกตว่า review_id 8 กับ 9 เป็นสินค้าเดียวกัน (Wireless Noise-Cancelling Headphones) แต่ให้ความรู้สึกตรงข้ามกันโดยสิ้นเชิง (rating 5 vs 2) — Full Text Search จับคู่คำได้ถูกต้องทั้งคู่ แต่**ไม่ได้บอกอารมณ์/ความรู้สึก** (sentiment) เป็นเรื่องที่ต้องใช้เทคนิคอื่น (เช่น NLP/ML) ประกอบเพิ่มเติมถ้าต้องการวิเคราะห์ sentiment จริงจัง — Full Text Search ตอบแค่คำถามว่า "เอกสารนี้พูดถึงคำนี้หรือไม่ และมากแค่ไหน" เท่านั้น

---

## สรุปท้ายบท

- `LIKE`/`ILIKE` เทียบ substring ตรง ๆ ไม่เข้าใจคำผันภาษา (running/runs match กันได้เพราะบังเอิญมี "run" เป็น substring แต่ "ran" ไม่มีทาง match ด้วยวิธีนี้) และไม่มี index ที่ใช้ได้ดีกับ pattern ที่มี wildcard นำหน้า
- `tsvector` คือข้อความที่ผ่านการ parse, ตัด stop word, และ stem ให้เหลือ lexeme พร้อมตำแหน่งกำกับ สร้างด้วย `to_tsvector(config, text)`
- `tsquery` คือคำค้นหาที่ normalize ด้วยกระบวนการเดียวกัน สร้างได้สามแบบ: `to_tsquery` (ควบคุมเต็มด้วย operator แต่ syntax ผิดจะ error), `plainto_tsquery` (AND ล้วน, ปลอดภัยสุด), `websearch_to_tsquery` (เข้าใจ syntax แบบ Google, ทนทานต่อ input แปลก ๆ ที่สุด — PostgreSQL 11+)
- operator `@@` จับคู่ `tsvector` กับ `tsquery` คืนค่า boolean
- Text Search Configuration อย่าง `'english'` กำหนดกฎ stop words และ stemming (Snowball algorithm) — จำไว้ว่า stemming จับคำผัน regular ได้ดี (run/running/runs) แต่**ไม่จับคำผัน irregular** (ran)
- `ts_rank`/`ts_rank_cd` จัดอันดับผลลัพธ์ตามความเกี่ยวข้อง ใช้ร่วมกับ `setweight()` เพื่อให้คำที่ match ในชื่อสินค้ามีน้ำหนักมากกว่าคำที่ match ในคำอธิบาย
- `ts_headline` สร้าง snippet ไฮไลท์คำที่ match คล้ายผลลัพธ์ search engine — ควรเรียกหลัง filter/rank เหลือแค่ TOP N แล้วเท่านั้น เพราะต้นทุนสูง
- `GENERATED ALWAYS AS ... STORED` คือวิธีที่แนะนำสำหรับเก็บ `tsvector` ล่วงหน้าในปัจจุบัน (เรียบง่าย ปลอดภัย อัปเดตอัตโนมัติ) ส่วน trigger เหมาะกับกรณีต้องรวมข้อมูลข้ามตาราง
- **GIN Index** บนคอลัมน์ `tsvector` ทำให้ full text search เร็วขึ้นมากในข้อมูลขนาดใหญ่ แต่กับข้อมูลน้อย planner อาจยังเลือก Sequential Scan เพราะ cost ต่ำกว่า — นี่คือพฤติกรรมปกติของ cost-based optimizer ไม่ใช่ปัญหาของ index

Full Text Search ใน PostgreSQL ทรงพลังพอสำหรับงานค้นหาระดับ production จำนวนมาก โดยไม่ต้องพึ่งพา external search engine อย่าง Elasticsearch เสมอไป — แต่เมื่อความต้องการซับซ้อนขึ้นถึงระดับ full-blown relevance tuning, faceted search, หรือ sentiment analysis ข้ามภาษา อาจถึงเวลาต้องพิจารณาเครื่องมือเฉพาะทางเพิ่มเติม

---

## แบบฝึกหัด

<details>
<summary><b>แบบฝึกหัดที่ 1:</b> เขียน query เปรียบเทียบผลลัพธ์ระหว่าง <code>description ILIKE '%waterproof%'</code> กับการค้นด้วย full text search <code>to_tsvector('english', description) @@ to_tsquery('english', 'waterproof')</code> บนตาราง products แล้วอธิบายว่าทำไมในกรณีนี้ผลลัพธ์ทั้งสองวิธีเหมือนกัน</summary>

```sql
SELECT product_id, product_name
FROM products
WHERE description ILIKE '%waterproof%';

SELECT product_id, product_name
FROM products
WHERE to_tsvector('english', description) @@ to_tsquery('english', 'waterproof');
```

```
 product_id |      product_name
------------+--------------------------
          6 | Bluetooth Portable Speaker
         23 | Camping Tent 4-Person
         27 | Waterproof Rain Jacket
(3 rows)
```

ทั้งสองวิธีให้ผลลัพธ์เหมือนกันในกรณีนี้ เพราะคำว่า "waterproof" เป็นคำที่**ไม่มีการผันรูป**ในข้อความต้นฉบับเลย (ปรากฏเป็น "waterproof" ตรง ๆ ทุกที่ ไม่มีการเขียนเป็น "waterproofed" หรือรูปอื่น) เมื่อไม่มีความหลากหลายของรูปคำ ปัญหาที่ `LIKE` มี (จับคำผันไม่ได้) จึงไม่ปรากฏ ความแตกต่างจะเห็นชัดก็ต่อเมื่อคำค้นหามีการผันรูปหลายแบบในข้อมูลจริง เช่นคำว่า "run" ที่สาธิตไว้ในบทเรียน
</details>

<details>
<summary><b>แบบฝึกหัดที่ 2:</b> ใช้ <code>to_tsvector('english', ...)</code> ตรวจสอบว่าคำว่า "Photographing", "photographers" และ "photography" ถูก stem ไปเป็น lexeme เดียวกันหรือไม่</summary>

```sql
SELECT to_tsvector('english', 'Photographing photographers love photography');
```

```
                       to_tsvector
-----------------------------------------------------------
 'love':3 'photograph':1,2 'photographi':4
```

**ไม่เหมือนกันทั้งหมด** — "Photographing" และ "photographers" ถูก stem ลงเป็น `photograph` เหมือนกัน แต่ "photography" กลับถูก stem เป็น `photographi` ซึ่งเป็นคนละ lexeme กัน! นี่คือตัวอย่างเพิ่มเติมของข้อจำกัดของ Snowball stemmer — มันใช้กฎตัดปัจจัยตายตัว ไม่ได้เข้าใจความหมายจริง ทำให้บางครั้งคำที่เกี่ยวข้องกันทางความหมายกลับไม่ถูกจัดกลุ่มเป็น lexeme เดียวกัน ผู้พัฒนาระบบค้นหาจริงจังจึงควรทดสอบคำสำคัญในโดเมนของตัวเองก่อนเสมอ ไม่ควรสมมติเอาว่า stemmer จะทำงานตามสัญชาตญาณเสมอไป
</details>

<details>
<summary><b>แบบฝึกหัดที่ 3:</b> สร้าง tsquery ด้วย <code>websearch_to_tsquery</code> ที่ค้นหาสินค้าเกี่ยวกับ "coffee" แต่ไม่เอาที่เกี่ยวกับ "maker" แล้วทดสอบกับตาราง products</summary>

```sql
SELECT websearch_to_tsquery('english', 'coffee -maker');
```

```
       websearch_to_tsquery
-----------------------------------
 'coffe' & !'maker'
```

```sql
SELECT product_id, product_name
FROM products
WHERE to_tsvector('english', product_name || ' ' || coalesce(description, ''))
      @@ websearch_to_tsquery('english', 'coffee -maker');
```

```
 product_id |      product_name
------------+-------------------------
         21 | Drip Coffee Maker Deluxe
(1 row)
```

ผลลัพธ์ยังคงคืน "Drip Coffee Maker Deluxe" เพราะคอลัมน์ที่ค้นหารวมทั้ง `product_name` ซึ่งมีคำว่า "Maker" อยู่ในชื่อสินค้าโดยตรง (`!'maker'` แปลว่า "ห้ามมี lexeme maker ในเอกสารทั้งก้อนเลย" ไม่ใช่แค่ในบางส่วน) ถ้าต้องการ exclude เอกสารที่ชื่อมีคำว่า maker แต่ยังอยากได้ description ที่ไม่มีคำนี้ ต้องแยกเงื่อนไขระหว่างคอลัมน์ให้ชัดเจนกว่านี้
</details>

<details>
<summary><b>แบบฝึกหัดที่ 4:</b> เขียน query หารีวิวทั้งหมดที่ให้ rating ต่ำ (rating <= 2) และมีคำที่เกี่ยวกับ "battery" หรือ "disappoint" ปรากฏอยู่ โดยใช้ full text search</summary>

```sql
SELECT review_id, product_id, rating, review_text
FROM reviews
WHERE rating <= 2
  AND to_tsvector('english', review_text)
      @@ to_tsquery('english', 'battery | disappoint')
ORDER BY review_id;
```

```
 review_id | product_id | rating |                       review_text
-----------+------------+--------+-----------------------------------------------------------
         9 |          5 |      2 | Sound quality is good but the battery drains faster than
           |            |        | advertised. Disappointed after only three months of use.
(1 row)
```

ตัวอย่างนี้แสดงให้เห็นว่าเราสามารถ**ผสม full text search เข้ากับเงื่อนไขตัวเลข/สถิติปกติ** (`rating <= 2`) ได้อย่างอิสระในเงื่อนไข `WHERE` เดียวกัน ซึ่งเป็นข้อดีของการทำ full text search ในฐานข้อมูลเชิงสัมพันธ์โดยตรง แทนที่จะแยกไปใช้ search engine ภายนอกที่ต้อง sync ข้อมูลสองระบบ
</details>

<details>
<summary><b>แบบฝึกหัดที่ 5:</b> เปรียบเทียบผลของ <code>to_tsvector('simple', description)</code> กับ <code>to_tsvector('english', description)</code> สำหรับ product_id = 1 แล้วอธิบายความแตกต่างของขนาดผลลัพธ์</summary>

```sql
SELECT
    array_length(tsvector_to_array(to_tsvector('simple', description)), 1) AS simple_lexemes,
    array_length(tsvector_to_array(to_tsvector('english', description)), 1) AS english_lexemes
FROM products
WHERE product_id = 1;
```

```
 simple_lexemes | english_lexemes
-----------------+------------------
              34 |               29
```

`'simple'` มีจำนวน lexeme ที่ไม่ซ้ำมากกว่า เพราะไม่ตัด stop word ออก (the, a, an, of, in ฯลฯ ยังถูกเก็บไว้ทั้งหมด) และไม่ stem คำ (running/runs/run ถือเป็นคนละคำกันหมด) ในขณะที่ `'english'` มีจำนวนน้อยกว่าเพราะตัด stop word ทิ้งและรวมคำที่ผันจากรากเดียวกันเข้าด้วยกัน (เช่น run/running/runs กลายเป็น `run` เพียง entry เดียว) — นี่คือเหตุผลที่ `'english'` ให้ผลการค้นหาที่ "ฉลาด" กว่าและ index มีขนาดเล็กกว่าสำหรับข้อความภาษาอังกฤษทั่วไป
</details>

<details>
<summary><b>แบบฝึกหัดที่ 6:</b> เขียน query จัดอันดับสินค้าทั้งหมดที่เกี่ยวกับ "camera" หรือ "photography" ด้วย ts_rank โดยให้ product_name มีน้ำหนักเป็น 'A' และ description เป็น 'B'</summary>

```sql
SELECT
    product_id,
    product_name,
    ts_rank(
        setweight(to_tsvector('english', product_name), 'A') ||
        setweight(to_tsvector('english', coalesce(description, '')), 'B'),
        websearch_to_tsquery('english', 'camera photography')
    ) AS rank
FROM products
WHERE (setweight(to_tsvector('english', product_name), 'A') ||
       setweight(to_tsvector('english', coalesce(description, '')), 'B'))
      @@ websearch_to_tsquery('english', 'camera photography')
ORDER BY rank DESC;
```

```
 product_id |      product_name       |   rank
------------+--------------------------+-----------
         14 | Mirrorless Camera Z5     | 0.2148936
         15 | Camera Drone Explorer    | 0.1216201
         13 | External SSD 1TB         | 0.0303956
(3 rows)
```

"Mirrorless Camera Z5" ได้อันดับสูงสุดเพราะคำว่า "Camera" match ตรงในชื่อสินค้า (weight A) และคำว่า "photography"/"photographer" ปรากฏซ้ำในคำอธิบายหลายครั้ง (weight B) ส่วน "External SSD 1TB" ติดมาในผลลัพธ์ด้วยแม้ชื่อไม่เกี่ยวกล้องเลย เพราะคำอธิบายกล่าวถึง "photography portfolio" ทำให้เห็นความสำคัญของการดู rank ประกอบการตัดสินใจ ไม่ใช่แค่ดูว่า match หรือไม่
</details>

<details>
<summary><b>แบบฝึกหัดที่ 7:</b> สร้าง generated column <code>search_vector</code> บนตาราง reviews (ถ้ายังไม่มี) และเขียน query ใช้ ts_headline แสดง snippet ของรีวิวที่พูดถึง "marathon" พร้อมไฮไลท์ด้วย <code>**...**</code></summary>

```sql
-- ถ้ายังไม่เคยสร้างไว้ในบทเรียน:
-- ALTER TABLE reviews ADD COLUMN search_vector tsvector
--     GENERATED ALWAYS AS (to_tsvector('english', coalesce(review_text, ''))) STORED;

SELECT
    review_id,
    product_id,
    ts_headline('english', review_text,
        to_tsquery('english', 'marathon'),
        'StartSel=**, StopSel=**') AS snippet
FROM reviews
WHERE search_vector @@ to_tsquery('english', 'marathon')
ORDER BY review_id;
```

```
 review_id | product_id |                                    snippet
-----------+------------+-------------------------------------------------------------------------
         1 |          1 | I ran my first half **marathon** in these shoes and they performed
           |            | beautifully.
         5 |          3 | The **Marathon** Pro is exactly what serious runners need. I ran a full
           |            | **marathon** last month wearing this pair and finished with zero blisters.
        16 |         14 | I used it to photograph a **marathon** race and every runner in the crowd
           |            | was sharp and clear.
(3 rows)
```
</details>

<details>
<summary><b>แบบฝึกหัดที่ 8:</b> รัน EXPLAIN ANALYZE เปรียบเทียบการค้นหาด้วย <code>ILIKE '%running%'</code> กับการใช้ <code>search_vector @@ to_tsquery(...)</code> บนคอลัมน์ที่มี GIN index แล้วอธิบายว่าทำไมในดาต้าเซตขนาดเล็กความต่างของเวลาจึงแทบไม่มีนัยสำคัญ</summary>

```sql
EXPLAIN ANALYZE
SELECT product_id FROM products WHERE description ILIKE '%running%';

EXPLAIN ANALYZE
SELECT product_id FROM products WHERE search_vector @@ to_tsquery('english', 'running');
```

```
 Seq Scan on products  (cost=0.00..2.38 rows=1 width=4) (actual time=0.020..0.055 rows=7 loops=1)
   Filter: (description ~~* '%running%'::text)
   Rows Removed by Filter: 23
 Planning Time: 0.090 ms
 Execution Time: 0.070 ms

 Seq Scan on products  (cost=0.00..2.38 rows=1 width=4) (actual time=0.018..0.050 rows=7 loops=1)
   Filter: (search_vector @@ '''run'''::tsquery)
   Rows Removed by Filter: 23
 Planning Time: 0.150 ms
 Execution Time: 0.065 ms
```

ทั้งสองวิธีใช้ **Sequential Scan** เหมือนกัน และเวลาที่ใช้ต่างกันในระดับเสี้ยว millisecond เท่านั้น เพราะตารางมีแค่ 30 แถว — การสแกนทั้งตารางไม่ว่าจะด้วยวิธีไหนก็เร็วมากอยู่แล้วในระดับนี้ ความได้เปรียบที่แท้จริงของ full text search + GIN index จะปรากฏชัดเจนก็ต่อเมื่อข้อมูลมีจำนวนมาก (หลักแสนถึงหลักล้านแถว) ซึ่ง `ILIKE '%...%'` จะยังคงต้องสแกนทุกแถวเสมอ (O(n) ที่ไม่มีทาง index ช่วยได้) ในขณะที่ full text search ด้วย GIN index จะค้นหาได้เร็วระดับ logarithmic เพราะ index ชี้ตรงไปยังแถวที่มี lexeme ที่ต้องการโดยไม่ต้องอ่านทุกแถว บทเรียนสำคัญคือ **อย่าตัดสิน performance จากดาต้าเซตตัวอย่างขนาดเล็กในห้องเรียน** ต้องทดสอบกับข้อมูลจำลองขนาดใกล้เคียงของจริงเสมอ
</details>

<details>
<summary><b>แบบฝึกหัดที่ 9:</b> เขียนฟังก์ชัน SQL ที่รับคำค้นหาและคืนจำนวนสินค้าที่เกี่ยวข้อง (นับจำนวน ไม่ต้องคืนรายละเอียด) โดยใช้ search_vector และ GIN index ที่สร้างไว้</summary>

```sql
CREATE OR REPLACE FUNCTION count_matching_products(p_keyword TEXT)
RETURNS INTEGER AS $$
    SELECT count(*)::INTEGER
    FROM products
    WHERE search_vector @@ websearch_to_tsquery('english', p_keyword)
      AND is_active;
$$ LANGUAGE sql STABLE;

SELECT count_matching_products('waterproof camping OR hiking');
```

```
 count_matching_products
--------------------------
                        4
(1 row)
```

ฟังก์ชันนี้สาธิตวิธีห่อ (encapsulate) ตรรกะการค้นหาไว้ในฟังก์ชันเดียว เพื่อให้ application layer เรียกใช้ได้ง่ายโดยไม่ต้องรู้รายละเอียดของ tsvector/tsquery การใช้ `search_vector` (คอลัมน์ generated ที่มี GIN index) แทนการเรียก `to_tsvector()` สดใหม่ทุกครั้งก็เพื่อให้ query planner มีโอกาสใช้ index เมื่อข้อมูลโตขึ้นในอนาคต
</details>

<details>
<summary><b>แบบฝึกหัดที่ 10:</b> อภิปราย (ตอบเป็นข้อความ) ว่าเมื่อไหร่ควรใช้ PostgreSQL Full Text Search และเมื่อไหร่ควรพิจารณาย้ายไปใช้ระบบค้นหาเฉพาะทางอย่าง Elasticsearch/OpenSearch แทน</summary>

**คำตอบที่แนะนำ:**

**ใช้ PostgreSQL Full Text Search เมื่อ:**

1. ข้อมูลที่ต้องการค้นหาอยู่ในฐานข้อมูลเดียวกับข้อมูลเชิงสัมพันธ์อื่น ๆ อยู่แล้ว (เช่นตัวอย่างในบทนี้ที่ค้นหาสินค้าพร้อม join กับ rating, ราคา, สต็อก) — การค้นหาแบบผสมเงื่อนไข relational + text ทำได้ใน query เดียวโดยไม่ต้อง sync สองระบบ
2. ปริมาณข้อมูลอยู่ในระดับที่ GIN index รองรับได้สบาย (โดยทั่วไปหลักล้านถึงหลักสิบล้านแถวยังทำงานได้ดีบน hardware ที่เหมาะสม)
3. ความต้องการค้นหาไม่ซับซ้อนเกินไป — ต้องการแค่ ranking, highlight, stemming พื้นฐานแบบที่สาธิตในบทนี้
4. ต้องการลดความซับซ้อนของ infrastructure — ไม่ต้องดูแล cluster แยกต่างหาก ไม่ต้องเขียนโค้ด sync ข้อมูลระหว่างสองระบบ ไม่ต้องกังวลเรื่อง data ไม่ตรงกัน (eventual consistency) ระหว่าง database หลักกับ search index

**ควรพิจารณาย้ายไปใช้ Elasticsearch/OpenSearch เมื่อ:**

1. ต้องการ **fuzzy matching / typo tolerance** ระดับสูง (ค้นหาแม้สะกดผิดบางตัวอักษร) ซึ่ง PostgreSQL ทำได้บ้างผ่าน extension อย่าง `pg_trgm` แต่ไม่ทรงพลังเท่าระบบเฉพาะทาง
2. ต้องการ **faceted search** ที่ซับซ้อนมาก (filter หลายมิติพร้อมกันแบบ aggregate นับจำนวนต่อ facet แบบ real-time บนข้อมูลขนาดใหญ่มาก)
3. ต้องการรองรับหลายภาษาพร้อมกันในระดับที่มี dictionary/analyzer เฉพาะทางมากกว่าที่ PostgreSQL config มีให้
4. ปริมาณ query ค้นหาต่อวินาที (QPS) สูงมากจนกระทบ workload หลักของฐานข้อมูล transactional และต้องการแยก workload ออกจากกันอย่างชัดเจน
5. ต้องการ relevance tuning ขั้นสูงมาก เช่น machine-learning-based ranking, custom scoring function ที่ซับซ้อนเกินกว่า `ts_rank`

**บทสรุป:** หลักการทั่วไปคือ **เริ่มต้นด้วย PostgreSQL Full Text Search ก่อนเสมอ** เพราะความซับซ้อนของ infrastructure ต่ำกว่ามาก และเพียงพอสำหรับ use case ส่วนใหญ่ในโลกจริง แล้วค่อยย้ายไปใช้ระบบเฉพาะทางเมื่อพิสูจน์ได้จริงว่าความต้องการเกินขีดความสามารถของมัน (premature optimization ด้วยการเพิ่ม infrastructure ที่ซับซ้อนโดยไม่จำเป็นเป็นความผิดพลาดที่พบบ่อยในทีมวิศวกรรม)
</details>

---

**บทถัดไป:** [Part 054: Table Partitioning](./part-054-table-partitioning.md)
