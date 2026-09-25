# Part 008: การสร้างตาราง CREATE TABLE และการออกแบบคอลัมน์

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับพื้นฐาน | Part 008

## เป้าหมายการเรียนรู้

หลังจากจบบทนี้ ผู้เรียนจะสามารถ:

- เขียน `CREATE TABLE` ได้ครบทุกรูปแบบ ทั้ง column-level constraints และ table-level constraints
- ตั้งชื่อตารางและคอลัมน์ตามหลัก naming convention ที่เป็นมาตรฐานอุตสาหกรรม และรู้ว่า reserved word คำไหนที่ต้องระวัง
- ตัดสินใจเลือก Primary Key ระหว่าง natural key กับ surrogate key ได้อย่างมีเหตุผล
- ใช้ `IF NOT EXISTS`, `CREATE TABLE ... LIKE`, และ `CREATE TABLE AS SELECT` (CTAS) เพื่อลดงานซ้ำซ้อน
- เข้าใจและเลือกใช้ Temporary Table และ Unlogged Table ได้ถูกสถานการณ์
- สร้างคอลัมน์คำนวณอัตโนมัติด้วย `GENERATED ALWAYS AS (...) STORED`
- ทำความเข้าใจหลักการ Normalization ระดับ 1NF, 2NF, 3NF พร้อมยกตัวอย่างก่อน-หลัง
- เข้าใจ trade-off ระหว่าง Normalization กับ Denormalization และตัดสินใจเลือกได้ตามบริบทงานจริง
- เขียนเอกสารประกอบ schema ด้วย `COMMENT ON`
- ออกแบบและสร้างฐานข้อมูลระบบร้านกาแฟแบบครบวงจรตั้งแต่การวิเคราะห์ requirement จนถึงการเขียน SQL จริง

ก่อนเริ่มบทนี้ ผู้เรียนควรเข้าใจ data type พื้นฐานของ PostgreSQL (จาก Part 007) และคำสั่งพื้นฐานอย่าง `CREATE DATABASE`, `\d` มาแล้ว เพราะบทนี้จะเป็นบทที่พาไปสู่การ "ออกแบบตารางจริง" ซึ่งเป็นทักษะหัวใจของการเป็น Database Developer/Administrator ที่ดี

---

## Step 71: Syntax เต็มของ CREATE TABLE

คำสั่ง `CREATE TABLE` คือคำสั่งที่ใช้บ่อยที่สุดคำสั่งหนึ่งใน PostgreSQL เพราะทุกฐานข้อมูลเริ่มต้นจากการสร้างตาราง มาดู syntax แบบเต็มกันก่อน

```sql
CREATE [ TEMPORARY | TEMP | UNLOGGED ] TABLE [ IF NOT EXISTS ] table_name (
    { column_name data_type [ COLLATE collation ] [ column_constraint [ ... ] ]
      | table_constraint
      | LIKE source_table [ like_option ... ] }
    [, ... ]
)
[ INHERITS ( parent_table [, ... ] ) ]
[ PARTITION BY { RANGE | LIST | HASH } ( column_name [, ...] ) ]
[ WITH ( storage_parameter [= value] [, ... ] ) | WITHOUT OIDS ]
[ TABLESPACE tablespace_name ];
```

อย่ากังวลถ้ายังไม่เข้าใจทุกส่วน เราจะค่อย ๆ แจกแจงทีละส่วนในบทนี้และบทถัด ๆ ไป (เช่น `INHERITS` และ `PARTITION BY` จะพูดถึงลึกในบทที่ว่าด้วย Table Partitioning) วันนี้เราจะโฟกัสที่ core syntax ซึ่งใช้งานจริง 90% ของเวลา

### ตัวอย่างพื้นฐานที่สุด

```sql
CREATE TABLE employees (
    employee_id    INTEGER,
    first_name     VARCHAR(50),
    last_name      VARCHAR(50),
    email          VARCHAR(255),
    hire_date      DATE,
    salary         NUMERIC(10, 2)
);
```

นี่คือตัวอย่างที่ยังไม่มี constraint ใด ๆ เลย ใช้งานได้ก็จริงแต่ในทางปฏิบัติแทบไม่มีใครสร้างตารางแบบนี้ เพราะขาดการควบคุมความถูกต้องของข้อมูล (data integrity)

### Column-level Constraints

Column-level constraint คือ constraint ที่เขียนติดกับคอลัมน์นั้นโดยตรง

```sql
CREATE TABLE employees (
    employee_id    INTEGER PRIMARY KEY,
    first_name     VARCHAR(50) NOT NULL,
    last_name      VARCHAR(50) NOT NULL,
    email          VARCHAR(255) UNIQUE NOT NULL,
    hire_date      DATE NOT NULL DEFAULT CURRENT_DATE,
    salary         NUMERIC(10, 2) CHECK (salary > 0),
    department_id  INTEGER REFERENCES departments(department_id)
);
```

จุดสำคัญของ column-level constraint:

| Constraint | ความหมาย |
|---|---|
| `PRIMARY KEY` | คอลัมน์นี้เป็นกุญแจหลัก ต้องไม่ซ้ำ ต้องไม่เป็น NULL |
| `NOT NULL` | ห้ามเป็นค่าว่าง |
| `UNIQUE` | ค่าต้องไม่ซ้ำกันในตาราง (แต่เป็น NULL ได้หลายแถว) |
| `DEFAULT` | ค่าเริ่มต้นถ้าไม่ระบุตอน INSERT |
| `CHECK` | เงื่อนไขที่ค่าต้องผ่าน |
| `REFERENCES` | Foreign Key อ้างอิงตารางอื่น |

### Table-level Constraints

Table-level constraint คือ constraint ที่แยกออกมาเขียนต่างหาก มักใช้เมื่อ constraint นั้นเกี่ยวข้องกับหลายคอลัมน์พร้อมกัน (composite constraint) หรือเมื่อต้องการตั้งชื่อ constraint เอง

```sql
CREATE TABLE order_items (
    order_id       INTEGER NOT NULL,
    product_id     INTEGER NOT NULL,
    quantity       INTEGER NOT NULL,
    unit_price     NUMERIC(10, 2) NOT NULL,

    -- table-level constraints
    CONSTRAINT pk_order_items PRIMARY KEY (order_id, product_id),
    CONSTRAINT fk_order FOREIGN KEY (order_id) REFERENCES orders(order_id),
    CONSTRAINT fk_product FOREIGN KEY (product_id) REFERENCES products(product_id),
    CONSTRAINT chk_quantity_positive CHECK (quantity > 0)
);
```

ข้อสังเกตสำคัญ:

- `PRIMARY KEY (order_id, product_id)` คือ **composite primary key** ซึ่งเขียนแบบ column-level ไม่ได้ ต้องเขียนแบบ table-level เท่านั้น
- การตั้งชื่อ constraint ด้วย `CONSTRAINT constraint_name` เป็นแนวปฏิบัติที่ดี เพราะถ้าไม่ตั้งชื่อ PostgreSQL จะตั้งชื่ออัตโนมัติให้ (เช่น `order_items_pkey`) ซึ่งอ่านยากเมื่อเกิด error หรือเมื่อต้องแก้ไขภายหลัง

### เปรียบเทียบ column-level กับ table-level

```sql
-- แบบ column-level: กระชับ เหมาะกับ constraint ที่เกี่ยวกับคอลัมน์เดียว
CREATE TABLE products_v1 (
    product_id  INTEGER PRIMARY KEY,
    price       NUMERIC(10, 2) CHECK (price >= 0)
);

-- แบบ table-level: เทียบเท่ากัน แต่เขียนแยก และตั้งชื่อได้
CREATE TABLE products_v2 (
    product_id  INTEGER,
    price       NUMERIC(10, 2),
    CONSTRAINT pk_products_v2 PRIMARY KEY (product_id),
    CONSTRAINT chk_price_non_negative CHECK (price >= 0)
);
```

ทั้งสองแบบให้ผลลัพธ์เหมือนกันทุกประการในกรณีที่ constraint เกี่ยวข้องกับคอลัมน์เดียว ความต่างอยู่ที่ความสามารถในการตั้งชื่อและอ่านง่ายเมื่อ schema ใหญ่ขึ้น

> **แนวปฏิบัติที่แนะนำ**: ในโปรเจกต์จริงระดับ production ควรตั้งชื่อ constraint เองเสมอ (โดยเฉพาะ CHECK และ FOREIGN KEY) เพราะเมื่อเกิด error เช่น `duplicate key value violates unique constraint "employees_pkey"` การมีชื่อที่สื่อความหมาย เช่น `uq_employees_email` จะช่วยให้ debug ได้เร็วกว่ามาก

### ตรวจสอบโครงสร้างตารางที่สร้าง

```sql
\d employees
\d+ employees   -- แสดงรายละเอียดเพิ่มเติม เช่น storage, description
```

---

## Step 72: การตั้งชื่อตารางและคอลัมน์

การตั้งชื่อที่ดีคือรากฐานของ schema ที่ดูแลรักษาง่าย (maintainable) การตั้งชื่อไม่สอดคล้องกันคือปัญหาที่พบบ่อยที่สุดในฐานข้อมูลที่เติบโตแบบไม่มีมาตรฐาน

### หลัก naming convention ที่แนะนำ: snake_case

PostgreSQL แปลง identifier ที่ไม่ได้ใส่ quote เป็นตัวพิมพ์เล็กเสมอ ดังนั้นการตั้งชื่อแบบ `snake_case` (ตัวพิมพ์เล็กทั้งหมด คั่นด้วย underscore) จึงเป็นธรรมเนียมที่เข้ากับพฤติกรรมของ PostgreSQL ได้ดีที่สุด

```sql
-- แนะนำ: snake_case
CREATE TABLE customer_orders (
    order_id       INTEGER PRIMARY KEY,
    customer_name  VARCHAR(100),
    order_date     DATE,
    total_amount   NUMERIC(12, 2)
);

-- ไม่แนะนำ: camelCase (ต้อง quote ตลอดเวลา ไม่งั้นจะกลายเป็นตัวเล็กหมด)
CREATE TABLE "customerOrders" (
    "orderId"      INTEGER PRIMARY KEY,
    "customerName" VARCHAR(100)
);
```

ทำไมถึงไม่แนะนำ camelCase? เพราะถ้าสร้างตารางด้วย `"customerOrders"` แล้ว query โดยไม่ใส่ quote:

```sql
SELECT * FROM customerOrders;  -- Error! PostgreSQL หา "customerorders" (ตัวเล็กหมด) ไม่เจอ
```

จะเกิด error ทันที เพราะ PostgreSQL แปลง `customerOrders` (ไม่มี quote) เป็น `customerorders` โดยอัตโนมัติ ซึ่งไม่ตรงกับชื่อจริงที่มี quote ครอบไว้ นี่คือกับดักที่มือใหม่เจอบ่อยมาก

### กฎการตั้งชื่อที่แนะนำ

1. **ใช้ snake_case ตัวพิมพ์เล็กทั้งหมด** — `order_items` ไม่ใช่ `OrderItems` หรือ `orderItems`
2. **ตารางใช้คำนามพหูพจน์หรือเอกพจน์ให้สอดคล้องกันทั้งระบบ** — เลือกอย่างใดอย่างหนึ่ง เช่น `customers`, `orders`, `products` (พหูพจน์) แล้วใช้ให้เหมือนกันทุกตาราง อย่าผสมกัน
3. **คอลัมน์ Primary Key ควรตั้งชื่อให้สื่อความหมายและสอดคล้องกัน** — เช่น `id` หรือ `<table_name_singular>_id` เช่น `customer_id`, `order_id`
4. **Foreign Key ควรตั้งชื่อให้สื่อว่าอ้างอิงตารางไหน** — เช่น `customer_id` ใน `orders` ที่อ้างอิงไปยัง `customers.id`
5. **หลีกเลี่ยงชื่อที่กำกวมหรือย่อจนอ่านไม่รู้เรื่อง** — `qty` พอเข้าใจได้ แต่ `q` ไม่ควร
6. **Boolean column ควรขึ้นต้นด้วย `is_`, `has_`, หรือ `can_`** — เช่น `is_active`, `has_discount`, `can_cancel`
7. **Timestamp column ควรมี suffix ที่บอกความหมาย** — `created_at`, `updated_at`, `deleted_at` (ไม่ใช่ `create_time` ปนกับ `updatedDate`)

```sql
-- ตัวอย่างการตั้งชื่อที่ดี
CREATE TABLE products (
    id              SERIAL PRIMARY KEY,
    sku             VARCHAR(50) NOT NULL UNIQUE,
    name            VARCHAR(200) NOT NULL,
    description     TEXT,
    price           NUMERIC(10, 2) NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    is_discontinued BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### Reserved Words ที่ต้องระวัง

PostgreSQL มี reserved word (คำสงวน) จำนวนมากที่ห้ามใช้เป็นชื่อ identifier โดยตรง (หรือถ้าจะใช้ต้อง quote เสมอ) คำที่พบบ่อยและมือใหม่มักไปชนคือ:

| คำสงวนที่ควรเลี่ยง | ควรใช้แทนว่า |
|---|---|
| `user` | `app_user`, `users` (พหูพจน์มักใช้ได้เพราะไม่ตรงกับ keyword `USER`) |
| `order` | `orders`, `customer_order` |
| `group` | `groups`, `user_group` |
| `table` | `data_table`, `reference_table` |
| `check` | `validation_check`, `review` |
| `column` | `field`, `attribute` |
| `references` | `refs`, `reference_data` |
| `primary` | `is_primary`, `main` |
| `select`, `from`, `where` | ห้ามใช้เด็ดขาดถ้าไม่ quote |

```sql
-- ปัญหา: user เป็น reserved word ใน SQL standard
CREATE TABLE user (         -- ควรเลี่ยง แม้ PostgreSQL จะยอมให้สร้างได้ถ้า quote
    id INTEGER PRIMARY KEY
);

-- วิธีตรวจสอบ error ที่จะเกิดถ้าไม่ quote
CREATE TABLE user (
    id INTEGER PRIMARY KEY
);
-- ERROR:  syntax error at or near "user"

-- แก้ด้วยการ quote (ทำได้ แต่ไม่แนะนำ เพราะต้อง quote ทุกครั้งที่ใช้งาน)
CREATE TABLE "user" (
    id INTEGER PRIMARY KEY
);

-- แนวทางที่แนะนำจริง ๆ: เปลี่ยนชื่อเลย ไม่ต้องพึ่ง quote
CREATE TABLE users (
    id INTEGER PRIMARY KEY
);
```

สามารถดูรายการ reserved word ทั้งหมดของ PostgreSQL ได้จาก:

```sql
SELECT word, catcode, catdesc
FROM pg_get_keywords()
WHERE catcode = 'R'   -- R = reserved
ORDER BY word;
```

`catcode` มีความหมายดังนี้:

- `R` = Reserved (สงวนเต็มรูปแบบ ห้ามใช้เป็น identifier โดยไม่ quote)
- `T` = Reserved (แต่สามารถใช้เป็นชื่อฟังก์ชันหรือ type ได้)
- `U` = Unreserved (ใช้เป็น identifier ได้ แต่มีความหมายพิเศษในบาง context)
- `C` = Unreserved (Column name) — ใช้เป็นชื่อคอลัมน์ได้อย่างอิสระ

### Quoted Identifiers

ถ้าจำเป็นต้องใช้ตัวพิมพ์ใหญ่ เว้นวรรค หรือคำสงวนจริง ๆ สามารถใช้ double quote ครอบชื่อได้ (ต่างจาก single quote ที่ใช้ครอบ string literal)

```sql
CREATE TABLE "Customer Orders" (
    "Order ID"     INTEGER PRIMARY KEY,
    "Customer Name" VARCHAR(100)
);

-- ต้อง quote ทุกครั้งที่อ้างถึง มิเช่นนั้นจะ error
SELECT "Order ID", "Customer Name" FROM "Customer Orders";
```

แต่ในทางปฏิบัติ **ไม่แนะนำให้ทำแบบนี้** เพราะจะทำให้ทุกคำสั่ง SQL ในอนาคตต้อง quote ตลอด เพิ่มความเสี่ยงในการเขียนผิดพลาดโดยไม่จำเป็น กฎทองคือ: **ตั้งชื่อด้วยตัวพิมพ์เล็ก ไม่มีเว้นวรรค ไม่ใช้คำสงวน แล้วจะไม่ต้อง quote อะไรเลยตลอดชีวิตการใช้งานตารางนั้น**

---

## Step 73: การออกแบบ Primary Key

Primary Key (PK) คือคอลัมน์ (หรือกลุ่มคอลัมน์) ที่ใช้ระบุตัวตนของแต่ละแถวอย่างไม่ซ้ำกัน เป็นหัวใจของการออกแบบตารางเชิงสัมพันธ์ (relational design)

### Natural Key คืออะไร

**Natural Key** คือคีย์ที่มาจากข้อมูลจริงที่มีความหมายทางธุรกิจอยู่แล้ว เช่น เลขบัตรประชาชน, อีเมล, รหัสสินค้า (SKU), เลขทะเบียนรถ

```sql
-- ใช้ natural key: email เป็น primary key
CREATE TABLE customers_natural (
    email       VARCHAR(255) PRIMARY KEY,
    full_name   VARCHAR(200) NOT NULL,
    phone       VARCHAR(20)
);
```

**ข้อดีของ natural key:**
- ไม่ต้องมีคอลัมน์เพิ่มขึ้นมาเปล่า ๆ
- มีความหมายในตัวเอง อ่านแล้วเข้าใจทันที

**ข้อเสียของ natural key:**
- ข้อมูลจริงมักเปลี่ยนแปลงได้ (เช่น ลูกค้าเปลี่ยนอีเมล) ทำให้การเปลี่ยน PK ส่งผลกระทบไปยังทุกตารางที่ foreign key อ้างอิงถึง (cascading update ที่แพงมาก)
- บางครั้งข้อมูลที่คิดว่าไม่ซ้ำ กลับซ้ำได้ในความเป็นจริง (เช่น เลขบัตรประชาชนของคนละประเทศอาจซ้ำกันได้ในระบบที่ไม่ตรวจสอบ)
- ขนาดคีย์อาจใหญ่ (เช่น VARCHAR ยาว ๆ) ทำให้ index ใหญ่และช้ากว่าตัวเลข integer

### Surrogate Key คืออะไร

**Surrogate Key** คือคีย์ที่ระบบสร้างขึ้นเอง ไม่มีความหมายทางธุรกิจใด ๆ มักเป็นตัวเลขที่รันขึ้นเรื่อย ๆ (auto-increment) หรือ UUID

```sql
-- ใช้ surrogate key: id เป็น primary key, email เป็นแค่ unique constraint
CREATE TABLE customers_surrogate (
    id          BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    email       VARCHAR(255) NOT NULL UNIQUE,
    full_name   VARCHAR(200) NOT NULL,
    phone       VARCHAR(20)
);
```

**ข้อดีของ surrogate key:**
- ไม่เปลี่ยนแปลงตลอดอายุของแถวข้อมูล แม้ข้อมูลทางธุรกิจ (เช่น email) จะเปลี่ยนไป
- เป็นตัวเลข ทำให้ index มีขนาดเล็ก เปรียบเทียบเร็ว JOIN เร็ว
- ใช้งานง่ายและสม่ำเสมอในทุกตาราง (ทุกตารางมี `id` เหมือนกันหมด ลด cognitive load ตอนออกแบบ)

**ข้อเสียของ surrogate key:**
- ไม่มีความหมายในตัวเอง ต้อง JOIN เพื่อดูข้อมูลจริง
- อาจทำให้เผลอลืมใส่ `UNIQUE` บนคอลัมน์ natural key ที่ควรจะไม่ซ้ำ (ต้องระวังเพิ่ม constraint เอง)

### ทำไมส่วนใหญ่แนะนำ surrogate key

ในทางปฏิบัติของอุตสาหกรรม (industry best practice) ส่วนใหญ่แนะนำให้ใช้ **surrogate key เป็น Primary Key เสมอ** และใส่ `UNIQUE` constraint แยกต่างหากให้กับ natural key (ถ้ามี) ด้วยเหตุผลดังนี้:

1. **ความเสถียร (stability)**: ข้อมูลธุรกิจเปลี่ยนได้เสมอ แต่ surrogate key ไม่เปลี่ยน การ JOIN และ Foreign Key ที่อ้างอิง PK ที่ไม่เปลี่ยนแปลงจะปลอดภัยกว่ามาก
2. **ประสิทธิภาพ**: Integer/Bigint เปรียบเทียบและ index เร็วกว่า string เสมอ
3. **ความสม่ำเสมอของ schema**: เมื่อทุกตารางมีรูปแบบ PK เหมือนกัน (เช่น `id BIGINT GENERATED ALWAYS AS IDENTITY`) โค้ดที่เขียนเพื่อจัดการ ORM หรือ migration tools จะทำงานได้ง่ายและคาดเดาได้
4. **รองรับกรณีที่ natural key ไม่แน่นอนในอนาคต**: สมมติวันนี้คิดว่า email ไม่ซ้ำแน่นอน แต่พรุ่งนี้ business requirement เปลี่ยนเป็นอนุญาตให้ 1 คนมีได้หลาย email หากใช้ email เป็น PK มาตั้งแต่แรกจะต้อง migrate ทั้งระบบ

```sql
-- แนวทางที่แนะนำ (best practice): surrogate key เป็น PK + natural key เป็น UNIQUE
CREATE TABLE customers (
    id            BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    email         VARCHAR(255) NOT NULL,
    national_id   VARCHAR(13),
    full_name     VARCHAR(200) NOT NULL,
    created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    CONSTRAINT uq_customers_email UNIQUE (email),
    CONSTRAINT uq_customers_national_id UNIQUE (national_id)
);
```

### ข้อยกเว้น: เมื่อไหร่ที่ natural key ยังเหมาะสม

ไม่ใช่ทุกกรณีที่ surrogate key จะดีที่สุดเสมอไป มีบางสถานการณ์ที่ natural key ยังเป็นตัวเลือกที่ดีกว่า:

- **Lookup table / Reference table ที่ค่าคงที่แน่นอนและไม่เปลี่ยนแปลง** เช่น รหัส ISO ของประเทศ (`ISO 3166-1 alpha-2` เช่น `TH`, `US`) เพราะมาตรฐานสากลไม่เปลี่ยนแปลง

```sql
CREATE TABLE countries (
    country_code  CHAR(2) PRIMARY KEY,   -- natural key: 'TH', 'US', 'JP'
    country_name  VARCHAR(100) NOT NULL
);
```

- **Junction table (ตารางเชื่อมความสัมพันธ์แบบ many-to-many)** มักใช้ composite key จาก foreign key สองตัวเป็น PK โดยตรง แทนที่จะสร้าง surrogate key เพิ่ม (แต่ทั้งสองแนวทางก็ยอมรับได้ ขึ้นกับทีม)

```sql
CREATE TABLE order_items (
    order_id    BIGINT NOT NULL REFERENCES orders(id),
    product_id  BIGINT NOT NULL REFERENCES products(id),
    quantity    INTEGER NOT NULL CHECK (quantity > 0),
    PRIMARY KEY (order_id, product_id)   -- composite natural-ish key
);
```

โดยสรุป: **กฎเริ่มต้น (default rule) ให้ใช้ surrogate key เสมอ** แล้วพิจารณายกเว้นเฉพาะกรณีที่มีเหตุผลชัดเจน เช่น reference table ที่ค่าคงที่ตายตัว

---

## Step 74: IF NOT EXISTS, CREATE TABLE ... LIKE, CREATE TABLE AS SELECT (CTAS)

### IF NOT EXISTS

การรันสคริปต์สร้างตารางซ้ำสองครั้งจะทำให้เกิด error `relation "xxx" already exists` การใช้ `IF NOT EXISTS` ช่วยให้สคริปต์รันซ้ำได้โดยไม่ error (idempotent script) ซึ่งสำคัญมากสำหรับ migration script และ deployment automation

```sql
CREATE TABLE IF NOT EXISTS products (
    id     BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name   VARCHAR(200) NOT NULL,
    price  NUMERIC(10, 2) NOT NULL
);

-- รันซ้ำได้โดยไม่ error
CREATE TABLE IF NOT EXISTS products (
    id     BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name   VARCHAR(200) NOT NULL,
    price  NUMERIC(10, 2) NOT NULL
);
-- NOTICE:  relation "products" already exists, skipping
```

**ข้อควรระวัง**: `IF NOT EXISTS` เช็คแค่ว่าตารางมีอยู่หรือไม่ ไม่ได้เช็คว่า schema ตรงกันหรือไม่ ถ้าตารางมีอยู่แล้วแต่โครงสร้างคอลัมน์ต่างจากที่กำหนดใหม่ PostgreSQL จะไม่แจ้งเตือนหรือแก้ไขให้ ระบบจะข้ามการสร้างไปเฉย ๆ (skip) ดังนั้นจึงไม่ใช่เครื่องมือสำหรับ "sync schema" แต่เป็นเครื่องมือป้องกัน error ตอนรันซ้ำเท่านั้น

### CREATE TABLE ... LIKE

`LIKE` ใช้คัดลอกโครงสร้างคอลัมน์จากตารางที่มีอยู่แล้ว มีประโยชน์มากเมื่อต้องการสร้างตารางสำรอง (staging table) หรือตารางที่มีโครงสร้างคล้ายกัน

```sql
CREATE TABLE products_staging (LIKE products);

-- ตรวจสอบ: จะได้คอลัมน์เหมือน products ทุกประการ แต่ไม่มี constraint/index ใด ๆ
\d products_staging
```

ค่าเริ่มต้นของ `LIKE` จะคัดลอกเฉพาะชื่อคอลัมน์และ data type เท่านั้น ไม่รวม constraint, default, index หรือ comment สามารถระบุ `like_option` เพิ่มเติมเพื่อคัดลอกส่วนอื่นได้:

```sql
CREATE TABLE products_staging_full (
    LIKE products
    INCLUDING DEFAULTS
    INCLUDING CONSTRAINTS
    INCLUDING INDEXES
    INCLUDING COMMENTS
);

-- หรือใช้ INCLUDING ALL เพื่อคัดลอกทุกอย่าง
CREATE TABLE products_staging_all (LIKE products INCLUDING ALL);
```

ตาราง `like_option` ที่ใช้ได้:

| Option | ความหมาย |
|---|---|
| `INCLUDING DEFAULTS` | คัดลอกค่า DEFAULT ของคอลัมน์ |
| `INCLUDING CONSTRAINTS` | คัดลอก CHECK constraint (ไม่รวม PK/FK/UNIQUE) |
| `INCLUDING INDEXES` | คัดลอก index รวมถึง PRIMARY KEY และ UNIQUE constraint |
| `INCLUDING COMMENTS` | คัดลอก comment ของคอลัมน์ |
| `INCLUDING IDENTITY` | คัดลอกคุณสมบัติ GENERATED AS IDENTITY |
| `INCLUDING GENERATED` | คัดลอก generated column expression |
| `INCLUDING STORAGE` | คัดลอกการตั้งค่า storage (PLAIN/EXTENDED/...) |
| `INCLUDING ALL` | รวมทุกอย่างข้างต้น |

ข้อสังเกตสำคัญ: `LIKE` **ไม่คัดลอก Foreign Key constraint** แม้จะใช้ `INCLUDING ALL` ก็ตาม เพราะ FK ผูกกับตารางอื่นที่อาจยังไม่มีอยู่ในบริบทใหม่ ต้องเพิ่ม FK เองภายหลังถ้าต้องการ

### CREATE TABLE AS SELECT (CTAS)

CTAS ใช้สร้างตารางใหม่จากผลลัพธ์ของ query ทันที ทั้งโครงสร้างและข้อมูลจะถูกคัดลอกมาด้วย เหมาะสำหรับการทำ snapshot, สร้างตารางสรุปผล (summary table), หรือสร้างข้อมูลทดสอบ

```sql
-- สมมติมีตาราง orders อยู่แล้ว พร้อมข้อมูล
CREATE TABLE high_value_orders AS
SELECT order_id, customer_id, total_amount, order_date
FROM orders
WHERE total_amount > 5000;

-- ตรวจสอบผลลัพธ์
SELECT COUNT(*) FROM high_value_orders;
\d high_value_orders
```

จุดสำคัญของ CTAS ที่ต้องระวัง:

- **CTAS ไม่คัดลอก constraint, index, หรือ default value ใด ๆ** จากตารางต้นทาง — ได้แค่ชื่อคอลัมน์และ data type ที่อนุมานจาก query result เท่านั้น
- ตารางที่ได้ **ไม่มี Primary Key** ต้องเพิ่มเองภายหลังถ้าต้องการ

```sql
-- เพิ่ม PK และ index ภายหลังจาก CTAS
ALTER TABLE high_value_orders ADD COLUMN id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY;
CREATE INDEX idx_high_value_orders_customer ON high_value_orders (customer_id);
```

สามารถใช้ร่วมกับ `WITH NO DATA` เพื่อคัดลอกแค่โครงสร้างโดยไม่เอาข้อมูล (มีประโยชน์เมื่อจะใช้เป็น template):

```sql
CREATE TABLE orders_template AS
SELECT * FROM orders
WITH NO DATA;

SELECT COUNT(*) FROM orders_template;  -- ได้ 0 เสมอ เพราะไม่มีข้อมูล แต่มีโครงสร้างครบ
```

**เปรียบเทียบ LIKE vs CTAS:**

| | `LIKE` | CTAS (`CREATE TABLE AS SELECT`) |
|---|---|---|
| คัดลอกข้อมูล | ไม่ (โครงสร้างอย่างเดียว) | คัดลอกด้วย (ยกเว้นใช้ `WITH NO DATA`) |
| คัดลอก constraint | ได้ถ้าระบุ `INCLUDING` | ไม่คัดลอกเลย |
| รองรับ query ซับซ้อน (JOIN, aggregate) | ไม่ | ได้ |
| เหมาะกับ | สร้างตารางโครงสร้างเหมือนเดิม | Snapshot, summary table, ทดสอบข้อมูล |

---

## Step 75: Temporary Tables และ Unlogged Tables

### Temporary Tables

Temporary Table คือตารางที่มีอายุอยู่แค่ใน session (การเชื่อมต่อ) เดียว เมื่อ session จบ ตารางจะถูกลบทิ้งโดยอัตโนมัติ และตารางชั่วคราวนี้จะไม่ถูกมองเห็นโดย session อื่นเลย แม้จะตั้งชื่อซ้ำกับตารางจริงก็ตาม

```sql
CREATE TEMPORARY TABLE temp_calculation (
    id       INTEGER,
    result   NUMERIC
);

-- หรือใช้คำย่อ TEMP
CREATE TEMP TABLE temp_calculation2 (
    id       INTEGER,
    result   NUMERIC
);

-- ใช้งานเหมือนตารางปกติทุกประการ
INSERT INTO temp_calculation (id, result) VALUES (1, 100.50);
SELECT * FROM temp_calculation;
```

**คุณสมบัติสำคัญของ Temporary Table:**

- ตารางจะหายไปโดยอัตโนมัติเมื่อ session จบ (disconnect) — เว้นแต่จะระบุ `ON COMMIT` เพื่อคุมพฤติกรรมเพิ่มเติม
- แต่ละ session เห็น temp table ของตัวเองเท่านั้น แม้จะตั้งชื่อซ้ำกัน ก็ไม่ชนกัน เพราะ PostgreSQL เก็บ temp table ไว้ใน schema พิเศษ (`pg_temp_N`) ที่แยกต่อ session
- Temp table ไม่ถูกบันทึกลง WAL (Write-Ahead Log) ในหลายกรณี ทำให้เขียนได้เร็วกว่าตารางปกติ

**ควบคุมอายุด้วย ON COMMIT:**

```sql
-- ค่าเริ่มต้น: ตารางอยู่ตลอด session (จนกว่าจะ disconnect หรือ DROP เอง)
CREATE TEMP TABLE t1 (x INT) ON COMMIT PRESERVE ROWS;

-- ลบข้อมูลทั้งหมดเมื่อ transaction commit แต่ตารางยังอยู่
CREATE TEMP TABLE t2 (x INT) ON COMMIT DELETE ROWS;

-- ลบตารางทั้งหมดทันทีเมื่อ transaction commit (ใช้เฉพาะใน transaction เดียว)
CREATE TEMP TABLE t3 (x INT) ON COMMIT DROP;
```

**เมื่อไหร่ควรใช้ Temporary Table:**

1. **การคำนวณขั้นกลางที่ซับซ้อน** ที่ต้องใช้หลาย query ต่อเนื่องกัน และไม่ต้องการเก็บผลลัพธ์ถาวร
2. **ETL/Data migration script** ที่ต้องการพื้นที่พักข้อมูลระหว่างการแปลงข้อมูล
3. **Batch processing** ที่ประมวลผลข้อมูลจำนวนมากเป็นขั้นตอน (staging → transform → load)
4. **การทดสอบ query ซับซ้อนแบบไม่กระทบข้อมูลจริง**

```sql
-- ตัวอย่าง: ใช้ temp table ช่วยคำนวณยอดขายสะสมก่อนสรุปผล
CREATE TEMP TABLE temp_monthly_sales AS
SELECT
    DATE_TRUNC('month', order_date) AS month,
    customer_id,
    SUM(total_amount) AS monthly_total
FROM orders
GROUP BY DATE_TRUNC('month', order_date), customer_id;

-- ใช้ query ต่อยอดจาก temp table
SELECT customer_id, AVG(monthly_total) AS avg_monthly_spend
FROM temp_monthly_sales
GROUP BY customer_id
ORDER BY avg_monthly_spend DESC
LIMIT 10;
```

### Unlogged Tables

Unlogged Table คือตารางที่ **ไม่บันทึกข้อมูลลง WAL (Write-Ahead Log)** ทำให้เขียนข้อมูล (INSERT/UPDATE/DELETE) ได้เร็วกว่าตารางปกติมาก แต่แลกมาด้วยการไม่รับประกันความปลอดภัยของข้อมูล

```sql
CREATE UNLOGGED TABLE session_cache (
    session_id   VARCHAR(64) PRIMARY KEY,
    payload      JSONB,
    expires_at   TIMESTAMPTZ NOT NULL
);
```

**ข้อควรทราบสำคัญเกี่ยวกับ Unlogged Table:**

- **ข้อมูลจะหายทั้งหมดถ้าเซิร์ฟเวอร์ crash หรือ restart แบบไม่ปกติ (unclean shutdown)** เพราะไม่มี WAL ให้ recover
- **ไม่ถูกคัดลอกไปยัง replica (streaming replication)** เพราะ replication ทำงานผ่าน WAL — ดังนั้นข้อมูลใน unlogged table จะไม่มีบน standby server เลย
- เขียนได้เร็วกว่าตารางปกติอย่างมีนัยสำคัญ เพราะข้าม overhead ของการเขียน WAL
- รองรับ index, constraint และ feature อื่น ๆ เหมือนตารางปกติทุกประการ

**เมื่อไหร่ควรใช้ Unlogged Table:**

1. **Cache / Session data** ที่ยอมรับได้ถ้าหายไปเมื่อ restart (เพราะ regenerate ใหม่ได้)
2. **ตารางพักข้อมูลชั่วคราวระหว่างขั้นตอน ETL** ที่ต้องการความเร็วสูงและข้อมูลไม่จำเป็นต้องคงทน
3. **Log table ที่ไม่ critical** เช่น access log ที่ยอมรับการสูญหายได้บางส่วน

```sql
-- แปลงตารางปกติเป็น unlogged (และกลับกัน) ได้ภายหลังด้วย ALTER TABLE
ALTER TABLE session_cache SET LOGGED;    -- แปลงเป็น logged (ปลอดภัยขึ้น แต่ช้าลง)
ALTER TABLE session_cache SET UNLOGGED;  -- แปลงกลับเป็น unlogged (เร็วขึ้น แต่เสี่ยงข้อมูลหาย)
```

**เปรียบเทียบ Temporary vs Unlogged vs Normal Table:**

| คุณสมบัติ | Normal Table | Unlogged Table | Temporary Table |
|---|---|---|---|
| เขียนลง WAL | ใช่ | ไม่ | ไม่ (ส่วนใหญ่) |
| อยู่ข้าม session | ใช่ | ใช่ | ไม่ (หายเมื่อ session จบ) |
| Replicate ไป standby | ใช่ | ไม่ | ไม่เกี่ยวข้อง (มองไม่เห็นข้ามเครื่องอยู่แล้ว) |
| รอด crash/restart | ใช่ | ไม่ (ข้อมูล truncate) | ไม่เกี่ยวข้อง |
| ความเร็วในการเขียน | ปกติ | เร็วกว่า | เร็วที่สุด (ในหลายกรณี) |
| เหมาะกับ | ข้อมูลจริงที่ต้องคงทน | cache, staging | คำนวณขั้นกลางใน session เดียว |

---

## Step 76: คอลัมน์ที่คำนวณอัตโนมัติ — Generated Columns

PostgreSQL (ตั้งแต่เวอร์ชัน 12 เป็นต้นมา) รองรับ **Generated Column** ซึ่งเป็นคอลัมน์ที่ค่าคำนวณมาจากคอลัมน์อื่นในแถวเดียวกันโดยอัตโนมัติ ไม่ต้องเขียนค่าตอน INSERT/UPDATE เอง

### Syntax พื้นฐาน

```sql
CREATE TABLE order_items (
    id          BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    quantity    INTEGER NOT NULL,
    unit_price  NUMERIC(10, 2) NOT NULL,
    subtotal    NUMERIC(12, 2) GENERATED ALWAYS AS (quantity * unit_price) STORED
);
```

ตอนนี้ทุกครั้งที่ INSERT หรือ UPDATE ค่า `quantity` หรือ `unit_price` PostgreSQL จะคำนวณ `subtotal` ให้อัตโนมัติ

```sql
INSERT INTO order_items (quantity, unit_price) VALUES (3, 45.00);

SELECT * FROM order_items;
--  id | quantity | unit_price | subtotal
-- ----+----------+------------+----------
--   1 |        3 |      45.00 |   135.00
```

ถ้าพยายามระบุค่า `subtotal` เองตอน INSERT จะเกิด error ทันที:

```sql
INSERT INTO order_items (quantity, unit_price, subtotal) VALUES (2, 50.00, 999.00);
-- ERROR:  cannot insert a non-DEFAULT value into column "subtotal"
-- DETAIL:  Column "subtotal" is a generated column.
```

### ข้อจำกัดสำคัญของ Generated Column

1. **PostgreSQL รองรับเฉพาะ `STORED` เท่านั้น** (ไม่รองรับ `VIRTUAL` เหมือนฐานข้อมูลอื่น เช่น MySQL) หมายความว่าค่าที่คำนวณได้จะถูก**เขียนลงดิสก์จริง** ไม่ใช่คำนวณสด ๆ ตอน query
2. **Generated column อ้างอิงได้เฉพาะคอลัมน์อื่นในแถวเดียวกันเท่านั้น** ห้ามอ้างอิงแถวอื่น ตารางอื่น หรือใช้ subquery
3. **ห้ามใช้ฟังก์ชันที่ไม่ deterministic** เช่น `NOW()`, `RANDOM()` เพราะค่าต้องคำนวณซ้ำได้เสมอ
4. **ไม่สามารถมี DEFAULT ของตัวเองได้** เพราะค่าคำนวณจาก expression เสมอ
5. **แก้ไขค่าโดยตรงผ่าน UPDATE ไม่ได้** ต้องแก้ที่คอลัมน์ต้นทางแทน

```sql
-- ตัวอย่างที่ผิด: ใช้ NOW() ซึ่งไม่ deterministic
CREATE TABLE bad_example (
    created_date DATE GENERATED ALWAYS AS (NOW()::DATE) STORED  -- ERROR
);
-- ERROR: generation expression is not immutable
```

### ตัวอย่างการใช้งานจริง

**ตัวอย่างที่ 1: คำนวณชื่อเต็มจากชื่อ-นามสกุล**

```sql
CREATE TABLE employees (
    id          BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    first_name  VARCHAR(50) NOT NULL,
    last_name   VARCHAR(50) NOT NULL,
    full_name   VARCHAR(101) GENERATED ALWAYS AS (first_name || ' ' || last_name) STORED
);

INSERT INTO employees (first_name, last_name) VALUES ('สมชาย', 'ใจดี');
SELECT full_name FROM employees;  -- 'สมชาย ใจดี'
```

**ตัวอย่างที่ 2: คำนวณราคาสุทธิหลังหักส่วนลด**

```sql
CREATE TABLE products (
    id             BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name           VARCHAR(200) NOT NULL,
    list_price     NUMERIC(10, 2) NOT NULL,
    discount_pct   NUMERIC(5, 2) NOT NULL DEFAULT 0 CHECK (discount_pct BETWEEN 0 AND 100),
    net_price      NUMERIC(10, 2) GENERATED ALWAYS AS
                        (list_price * (1 - discount_pct / 100.0)) STORED
);

INSERT INTO products (name, list_price, discount_pct) VALUES ('เมล็ดกาแฟอราบิก้า 250g', 250.00, 10);
SELECT name, list_price, discount_pct, net_price FROM products;
--         name           | list_price | discount_pct | net_price
-- -----------------------+------------+--------------+-----------
--  เมล็ดกาแฟอราบิก้า 250g |     250.00 |        10.00 |    225.00
```

**ตัวอย่างที่ 3: สร้าง search-friendly column จาก JSONB**

```sql
CREATE TABLE product_catalog (
    id        BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    data      JSONB NOT NULL,
    category  TEXT GENERATED ALWAYS AS (data->>'category') STORED
);

CREATE INDEX idx_product_catalog_category ON product_catalog (category);
```

### ข้อดีของ Generated Column เทียบกับ Trigger

ก่อนมี Generated Column นักพัฒนาต้องใช้ `TRIGGER` เพื่อคำนวณค่าให้อัตโนมัติ ซึ่งซับซ้อนกว่ามาก Generated Column ให้ข้อดีดังนี้:

- เขียนโค้ดน้อยกว่ามาก (บรรทัดเดียวใน `CREATE TABLE`)
- PostgreSQL รับประกันความสอดคล้อง (consistency) ให้เอง ไม่มีโอกาสลืมอัปเดต trigger
- Query planner รู้จัก generated column และสามารถใช้ index บนคอลัมน์นี้ได้อย่างมีประสิทธิภาพ (ต่างจาก virtual column ในบางระบบที่ index ไม่ได้)

---

## Step 77: การออกแบบตารางแบบ Normalization เบื้องต้น (1NF, 2NF, 3NF)

**Normalization** คือกระบวนการจัดโครงสร้างตารางเพื่อลดความซ้ำซ้อนของข้อมูล (data redundancy) และป้องกันความผิดปกติที่อาจเกิดขึ้นเมื่อ insert, update หรือ delete ข้อมูล (เรียกว่า anomaly)

### ตัวอย่างตารางที่ยังไม่ Normalize

เริ่มจากตารางที่ออกแบบไม่ดี เพื่อดูปัญหาที่เกิดขึ้น:

```sql
-- ตารางแบบ "flat" ที่ยัดทุกอย่างไว้ด้วยกัน (ยังไม่ normalize)
CREATE TABLE orders_bad (
    order_id          INTEGER,
    order_date         DATE,
    customer_name      VARCHAR(200),
    customer_phone     VARCHAR(20),
    customer_address   VARCHAR(500),
    product_names       VARCHAR(1000),   -- เก็บ "กาแฟลาเต้, ครัวซองต์, เอสเพรสโซ่" คั่นด้วย comma
    product_prices      VARCHAR(500),    -- เก็บ "60,45,50" คั่นด้วย comma
    total_amount       NUMERIC(12, 2)
);
```

ปัญหาของตารางนี้:

1. **ละเมิด 1NF** — คอลัมน์ `product_names` และ `product_prices` เก็บค่าหลายค่าไว้ในเซลล์เดียว (multi-valued attribute) ทำให้ query หา "จำนวนสินค้าทั้งหมดที่ขาย" หรือ "ยอดขายของสินค้าแต่ละชิ้น" ทำได้ยากมาก ต้อง parse string เอง
2. **ข้อมูลลูกค้าซ้ำซ้อน** — ถ้าลูกค้าคนเดิมสั่งซื้อหลายครั้ง ชื่อ เบอร์โทร ที่อยู่ จะถูกเก็บซ้ำในทุกแถว ทำให้ถ้าลูกค้าเปลี่ยนเบอร์โทร ต้อง UPDATE หลายแถว (update anomaly)
3. **Insert Anomaly** — ถ้าต้องการเพิ่มลูกค้าใหม่ที่ยังไม่มีคำสั่งซื้อ จะทำไม่ได้เลยเพราะตารางนี้ผูกกับ order เท่านั้น
4. **Delete Anomaly** — ถ้าลบ order เดียวที่เป็น order สุดท้ายของลูกค้าคนหนึ่ง ข้อมูลลูกค้าทั้งหมดจะหายไปด้วย ทั้งที่ควรเก็บไว้

### First Normal Form (1NF)

**กฎของ 1NF**: ทุกคอลัมน์ต้องเก็บค่าเดี่ยว (atomic value) ห้ามเก็บหลายค่าในเซลล์เดียว และห้ามมีกลุ่มคอลัมน์ซ้ำ (repeating groups)

```sql
-- แก้ปัญหา 1NF: แยก product ออกมาเป็นแถวต่างหาก (แต่ยังไม่แก้ปัญหาลูกค้าซ้ำซ้อน)
CREATE TABLE order_items_1nf (
    order_id       INTEGER,
    order_date     DATE,
    customer_name  VARCHAR(200),
    customer_phone VARCHAR(20),
    product_name   VARCHAR(200),
    product_price  NUMERIC(10, 2),
    quantity       INTEGER
);
```

ตอนนี้แต่ละแถวเก็บสินค้าหนึ่งชิ้นต่อหนึ่งคำสั่งซื้อ ทำให้ query หาสินค้าแต่ละชิ้นได้ตรงไปตรงมา แต่ข้อมูลลูกค้ายังซ้ำซ้อนอยู่ (ถ้า order เดียวมี 3 สินค้า ชื่อลูกค้าจะซ้ำ 3 แถว)

### Second Normal Form (2NF)

**กฎของ 2NF**: ต้องผ่าน 1NF ก่อน และทุกคอลัมน์ที่ไม่ใช่ key ต้องขึ้นอยู่กับ **primary key ทั้งชุด** (full functional dependency) ไม่ใช่แค่บางส่วนของ composite key — ปัญหานี้เกิดเฉพาะเมื่อ PK เป็น composite key เท่านั้น

จากตัวอย่างข้างต้น ถ้า PK คือ `(order_id, product_name)` จะพบว่า `customer_name`, `customer_phone`, `order_date` ขึ้นอยู่กับ `order_id` เพียงอย่างเดียว (ไม่ขึ้นกับ `product_name`) และ `product_price` ขึ้นอยู่กับ `product_name` เพียงอย่างเดียว — นี่คือ **partial dependency** ที่ละเมิด 2NF

```sql
-- แยกเป็น 3 ตารางเพื่อให้ผ่าน 2NF
CREATE TABLE customers_2nf (
    customer_id    SERIAL PRIMARY KEY,
    customer_name  VARCHAR(200) NOT NULL,
    customer_phone VARCHAR(20)
);

CREATE TABLE orders_2nf (
    order_id     SERIAL PRIMARY KEY,
    order_date   DATE NOT NULL,
    customer_id  INTEGER NOT NULL REFERENCES customers_2nf(customer_id)
);

CREATE TABLE products_2nf (
    product_id     SERIAL PRIMARY KEY,
    product_name   VARCHAR(200) NOT NULL,
    product_price  NUMERIC(10, 2) NOT NULL
);

CREATE TABLE order_items_2nf (
    order_id    INTEGER NOT NULL REFERENCES orders_2nf(order_id),
    product_id  INTEGER NOT NULL REFERENCES products_2nf(product_id),
    quantity    INTEGER NOT NULL CHECK (quantity > 0),
    PRIMARY KEY (order_id, product_id)
);
```

ตอนนี้ข้อมูลลูกค้าเก็บครั้งเดียว ข้อมูลสินค้าเก็บครั้งเดียว และ `order_items_2nf` เก็บแค่ความสัมพันธ์ (relationship) ระหว่าง order กับ product เท่านั้น

### Third Normal Form (3NF)

**กฎของ 3NF**: ต้องผ่าน 2NF ก่อน และคอลัมน์ที่ไม่ใช่ key ต้องไม่ขึ้นอยู่กับคอลัมน์ที่ไม่ใช่ key อื่น (ต้องไม่มี **transitive dependency**) พูดง่าย ๆ คือ "ทุกคอลัมน์ต้องขึ้นอยู่กับ key, ทั้ง key, และไม่มีอะไรนอกจาก key" (The key, the whole key, and nothing but the key)

สมมติมีตารางที่ยังละเมิด 3NF:

```sql
-- ละเมิด 3NF: department_name และ department_location ขึ้นอยู่กับ department_id
-- ซึ่ง department_id ก็เป็นแค่คอลัมน์ที่ไม่ใช่ key (ไม่ใช่ employee_id)
-- นี่คือ transitive dependency: employee_id -> department_id -> department_name
CREATE TABLE employees_bad_3nf (
    employee_id          INTEGER PRIMARY KEY,
    employee_name        VARCHAR(200) NOT NULL,
    department_id        INTEGER,
    department_name      VARCHAR(100),   -- ขึ้นอยู่กับ department_id ไม่ใช่ employee_id โดยตรง
    department_location  VARCHAR(200)    -- เช่นเดียวกัน
);
```

ปัญหา: ถ้าแผนก "Engineering" ย้ายที่ตั้ง ต้อง UPDATE ทุกแถวของพนักงานในแผนกนั้น และถ้าแผนกยังไม่มีพนักงานเลย จะไม่สามารถเก็บข้อมูลแผนกได้เลย (insert anomaly อีกแล้ว)

```sql
-- แก้ให้ผ่าน 3NF: แยก department ออกมาเป็นตารางของตัวเอง
CREATE TABLE departments_3nf (
    department_id        SERIAL PRIMARY KEY,
    department_name      VARCHAR(100) NOT NULL,
    department_location  VARCHAR(200)
);

CREATE TABLE employees_3nf (
    employee_id     SERIAL PRIMARY KEY,
    employee_name   VARCHAR(200) NOT NULL,
    department_id   INTEGER REFERENCES departments_3nf(department_id)
);
```

ตอนนี้ `employees_3nf` มีแค่คอลัมน์ที่ขึ้นอยู่กับ `employee_id` โดยตรงเท่านั้น ส่วนข้อมูลแผนกแยกไปอยู่คนละตาราง เชื่อมกันด้วย foreign key

### สรุปภาพรวมสามระดับ Normal Form

| Normal Form | กฎหลัก | แก้ปัญหาอะไร |
|---|---|---|
| **1NF** | ทุกเซลล์เก็บค่าเดียว (atomic), ไม่มี repeating group | ข้อมูลหลายค่าในเซลล์เดียว, array ที่ไม่ควรเป็น array |
| **2NF** | ผ่าน 1NF + ไม่มี partial dependency (สำคัญเฉพาะ composite PK) | คอลัมน์ที่ขึ้นกับแค่บางส่วนของ composite key |
| **3NF** | ผ่าน 2NF + ไม่มี transitive dependency | คอลัมน์ที่ขึ้นกับคอลัมน์อื่นที่ไม่ใช่ key |

ตัวอย่างสุดท้าย เมื่อ normalize ครบทั้งระบบร้านกาแฟแบบง่าย ๆ (ก่อนเข้าโปรเจกต์เต็มใน Step 80):

```sql
-- โครงสร้างที่ผ่าน 3NF แล้ว: แยกความรับผิดชอบของแต่ละ entity ชัดเจน
CREATE TABLE customers_demo (
    id     SERIAL PRIMARY KEY,
    name   VARCHAR(200) NOT NULL,
    phone  VARCHAR(20)
);

CREATE TABLE products_demo (
    id     SERIAL PRIMARY KEY,
    name   VARCHAR(200) NOT NULL,
    price  NUMERIC(10, 2) NOT NULL
);

CREATE TABLE orders_demo (
    id           SERIAL PRIMARY KEY,
    customer_id  INTEGER NOT NULL REFERENCES customers_demo(id),
    order_date   DATE NOT NULL DEFAULT CURRENT_DATE
);

CREATE TABLE order_items_demo (
    order_id    INTEGER NOT NULL REFERENCES orders_demo(id),
    product_id  INTEGER NOT NULL REFERENCES products_demo(id),
    quantity    INTEGER NOT NULL CHECK (quantity > 0),
    unit_price  NUMERIC(10, 2) NOT NULL,  -- เก็บราคา ณ เวลาสั่งซื้อ (snapshot) ไม่ดึงจาก products_demo.price
    PRIMARY KEY (order_id, product_id)
);
```

สังเกตว่า `order_items_demo.unit_price` **จงใจเก็บซ้ำ** กับ `products_demo.price` — นี่ไม่ใช่ความผิดพลาด แต่เป็นการตัดสินใจออกแบบที่ถูกต้อง เพราะราคาสินค้าตอนสั่งซื้อต้องเป็น "ภาพนิ่ง" (snapshot) ณ เวลานั้น ไม่ควรเปลี่ยนตามราคาปัจจุบันของสินค้า ถ้าร้านปรับราคากาแฟพรุ่งนี้ ใบเสร็จของเมื่อวานต้องยังแสดงราคาเดิม เรื่องนี้จะอธิบายลึกขึ้นใน Step 78

---

## Step 78: เมื่อไหร่ควร Denormalize

**Denormalization** คือการจงใจเพิ่มความซ้ำซ้อนของข้อมูลกลับเข้าไปในตารางที่ normalize แล้ว เพื่อแลกกับประสิทธิภาพในการอ่านข้อมูล (read performance) ที่ดีขึ้น

### ทำไมต้อง Denormalize

Normalization ที่สมบูรณ์แบบ (fully normalized) ให้ความถูกต้องของข้อมูลสูงสุดและลดความซ้ำซ้อน แต่แลกมาด้วยการต้อง `JOIN` หลายตารางทุกครั้งที่ query ซึ่งในระบบที่มีข้อมูลจำนวนมากและ traffic การอ่านสูง การ JOIN จำนวนมากอาจกลายเป็นคอขวด (bottleneck) ด้านประสิทธิภาพ

### ตัวอย่างสถานการณ์ที่ควร Denormalize

**ตัวอย่างที่ 1: เก็บราคา ณ เวลาขาย (price snapshot)**

ตามที่กล่าวไปใน Step 77 การเก็บ `unit_price` ซ้ำใน `order_items` แม้จะมี `products.price` อยู่แล้ว คือการ denormalize ที่ถูกต้อง เพราะ:

```sql
-- ถ้าไม่ denormalize (ดึงราคาจาก products ตรง ๆ ทุกครั้ง)
SELECT oi.quantity, p.price, oi.quantity * p.price AS subtotal
FROM order_items oi
JOIN products p ON p.id = oi.product_id
WHERE oi.order_id = 1001;
-- ปัญหา: ถ้าร้านปรับราคากาแฟพรุ่งนี้ ใบเสร็จเก่าจะแสดงราคาผิด! เพราะดึงราคาปัจจุบันเสมอ

-- แนวทางที่ถูกต้อง: เก็บ unit_price ไว้ตอน insert order_items
INSERT INTO order_items_demo (order_id, product_id, quantity, unit_price)
SELECT 1001, id, 2, price FROM products_demo WHERE id = 5;
-- ราคาที่เก็บไว้จะ "แช่แข็ง" ตามราคา ณ เวลาสั่งซื้อ ไม่เปลี่ยนตามราคาปัจจุบัน
```

**ตัวอย่างที่ 2: เก็บยอดรวมที่คำนวณไว้ล่วงหน้า (aggregate caching)**

```sql
-- แบบ normalize เต็มรูปแบบ: ต้องคำนวณ total ทุกครั้งที่ query
SELECT o.id, SUM(oi.quantity * oi.unit_price) AS total
FROM orders_demo o
JOIN order_items_demo oi ON oi.order_id = o.id
GROUP BY o.id;

-- แบบ denormalize: เพิ่มคอลัมน์ total_amount ใน orders เก็บยอดรวมไว้เลย
ALTER TABLE orders_demo ADD COLUMN total_amount NUMERIC(12, 2) NOT NULL DEFAULT 0;

-- อ่านค่าได้ทันทีโดยไม่ต้อง JOIN หรือ GROUP BY
SELECT id, total_amount FROM orders_demo WHERE id = 1001;
```

การ denormalize แบบนี้เหมาะมากกับหน้าจอ "ประวัติคำสั่งซื้อ" ที่ผู้ใช้เปิดดูบ่อย ๆ เพราะไม่ต้องคำนวณซ้ำทุกครั้ง แต่ **ต้องมีกลไกดูแลความสอดคล้อง** (เช่น trigger หรือ application logic) เพื่อให้ `total_amount` อัปเดตตรงกับผลรวมจริงของ `order_items` เสมอ ไม่เช่นนั้นข้อมูลจะไม่ตรงกัน (data inconsistency)

**ตัวอย่างที่ 3: เก็บชื่อที่ใช้แสดงผลบ่อย ๆ ซ้ำไว้ (avoid frequent JOIN)**

```sql
-- ระบบรายงานที่ query บ่อยมาก อาจเก็บ customer_name ซ้ำไว้ใน orders
-- เพื่อไม่ต้อง JOIN กับ customers ทุกครั้งที่แสดงรายการ order
ALTER TABLE orders_demo ADD COLUMN customer_name_snapshot VARCHAR(200);
```

### Trade-off ระหว่าง Normalization กับ Performance

| ด้าน | Normalized (normalize เต็มที่) | Denormalized (denormalize บางส่วน) |
|---|---|---|
| ความถูกต้องของข้อมูล | สูง (single source of truth) | เสี่ยงข้อมูลไม่ตรงกันถ้าดูแลไม่ดี |
| พื้นที่จัดเก็บ | ใช้น้อยกว่า | ใช้มากกว่า (มีข้อมูลซ้ำ) |
| ความเร็วในการอ่าน (SELECT) | ช้ากว่า (ต้อง JOIN หลายตาราง) | เร็วกว่า (อ่านตารางเดียวหรือ JOIN น้อยกว่า) |
| ความเร็วในการเขียน (INSERT/UPDATE) | เร็วกว่า (แก้ที่เดียว) | ช้ากว่า/ซับซ้อนกว่า (ต้องอัปเดตหลายที่ให้ตรงกัน) |
| ความซับซ้อนของ application logic | น้อยกว่า | มากกว่า (ต้องดูแล consistency เอง) |
| เหมาะกับ | ระบบที่เน้นความถูกต้อง (OLTP ทั่วไป, ระบบการเงิน) | ระบบที่เน้นอ่านเร็ว (dashboard, reporting, read-heavy API) |

### หลักการตัดสินใจ

1. **เริ่มต้นด้วยการ normalize ให้ถึง 3NF เสมอ** — นี่คือ default ที่ปลอดภัยที่สุด เพราะป้องกัน anomaly ต่าง ๆ ได้ดี
2. **Denormalize เฉพาะเมื่อมีเหตุผลที่วัดผลได้จริง** เช่น query ช้าเกินไปจนวัดผลกระทบต่อผู้ใช้จริง (ไม่ใช่ denormalize ล่วงหน้าเพราะ "คิดว่า" จะช้า — นี่คือ premature optimization)
3. **ข้อมูลที่เป็น "เหตุการณ์ในอดีต" (historical fact) ควรเก็บเป็น snapshot เสมอ** เช่น ราคาสินค้า ณ เวลาขาย, ที่อยู่จัดส่ง ณ เวลาสั่งซื้อ — เพราะข้อมูลปัจจุบันของ entity อาจเปลี่ยนไปแล้ว แต่ประวัติต้องไม่เปลี่ยน
4. **ถ้า denormalize ต้องมีกลไกรักษาความสอดคล้อง** เช่น database trigger, application-level transaction, หรือ scheduled job ที่ sync ข้อมูลให้ตรงกัน
5. **พิจารณาทางเลือกอื่นก่อน denormalize เต็มรูปแบบ** เช่น Materialized View (จะเรียนในบทขั้นสูง), การเพิ่ม index ที่เหมาะสม, หรือ caching layer ภายนอก (เช่น Redis) ซึ่งบางครั้งแก้ปัญหาประสิทธิภาพได้โดยไม่ต้องแตะโครงสร้างตาราง

---

## Step 79: Comment บนตาราง/คอลัมน์ — การทำเอกสารประกอบ Schema

เมื่อ schema เติบโตขึ้น การจดจำความหมายของแต่ละตารางและคอลัมน์ด้วยความจำอย่างเดียวเป็นไปไม่ได้ PostgreSQL มีกลไก `COMMENT ON` ที่ฝังเอกสารประกอบไว้ใน database เองโดยตรง ทำให้ทุกคนที่เชื่อมต่อฐานข้อมูลเห็นคำอธิบายได้ทันทีโดยไม่ต้องเปิดเอกสารแยก

### Syntax ของ COMMENT ON

```sql
COMMENT ON TABLE table_name IS 'คำอธิบาย';
COMMENT ON COLUMN table_name.column_name IS 'คำอธิบาย';
```

รองรับ object type อื่น ๆ อีกมาก เช่น `COMMENT ON DATABASE`, `COMMENT ON SCHEMA`, `COMMENT ON INDEX`, `COMMENT ON CONSTRAINT`, `COMMENT ON FUNCTION` เป็นต้น

### ตัวอย่างการใช้งาน

```sql
CREATE TABLE customers (
    id          BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    email       VARCHAR(255) NOT NULL UNIQUE,
    full_name   VARCHAR(200) NOT NULL,
    tier        VARCHAR(20) NOT NULL DEFAULT 'standard',
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

COMMENT ON TABLE customers IS
    'เก็บข้อมูลลูกค้าทั้งหมดของระบบ รวมถึงลูกค้าที่ยังไม่เคยสั่งซื้อ';

COMMENT ON COLUMN customers.email IS
    'อีเมลของลูกค้า ใช้สำหรับ login และรับการแจ้งเตือน ต้องไม่ซ้ำกันในระบบ';

COMMENT ON COLUMN customers.tier IS
    'ระดับสมาชิก: standard, silver, gold, platinum กำหนดจากยอดใช้จ่ายสะสมรายปี '
    '(ดู business rule เต็มที่เอกสาร loyalty-program.md)';

COMMENT ON CONSTRAINT customers_email_key ON customers IS
    'ป้องกันไม่ให้สมัครสมาชิกซ้ำด้วยอีเมลเดียวกัน';
```

### วิธีดู comment ที่บันทึกไว้

```sql
-- ผ่าน psql meta-command (แสดง comment ของตารางในผลลัพธ์ \d+)
\d+ customers

-- หรือ query จาก system catalog โดยตรง
SELECT
    c.relname AS table_name,
    pg_catalog.obj_description(c.oid, 'pg_class') AS table_comment
FROM pg_catalog.pg_class c
WHERE c.relname = 'customers';

-- ดู comment ของแต่ละคอลัมน์
SELECT
    a.attname AS column_name,
    pg_catalog.col_description(a.attrelid, a.attnum) AS column_comment
FROM pg_catalog.pg_attribute a
WHERE a.attrelid = 'customers'::regclass
  AND a.attnum > 0
  AND NOT a.attisdropped;
```

### การลบ comment

ตั้งค่า comment เป็น `NULL` เพื่อลบออก:

```sql
COMMENT ON COLUMN customers.tier IS NULL;
```

### ทำไม COMMENT ON ถึงสำคัญ

1. **เอกสารไม่มีวันตกยุค (never goes stale)** — ต่างจากเอกสารภายนอก (Word, Wiki, Confluence) ที่มักไม่มีใครอัปเดตตามโค้ดจริง comment ที่ฝังใน database จะอยู่คู่กับ schema เสมอ ถ้ามีการ migrate/backup/restore ก็ติดไปด้วย
2. **เครื่องมือหลายตัวอ่าน comment ได้อัตโนมัติ** เช่น `pgAdmin`, `DBeaver`, เครื่องมือสร้าง ER diagram, หรือ documentation generator ต่าง ๆ สามารถดึง comment มาแสดงเป็นเอกสารได้ทันที
3. **ช่วยลดคำถามซ้ำซ้อนในทีม** — เมื่อสมาชิกใหม่เข้าทีม การมี comment อธิบาย business rule ที่ซับซ้อน (เช่น field `tier` มีเกณฑ์อย่างไร) ช่วยลดเวลาที่ต้องถามคนอื่นหรือขุดหาเอกสารเก่า
4. **สำคัญมากสำหรับคอลัมน์ที่ชื่อไม่สื่อความหมายชัดเจน** เช่น legacy column ที่ชื่อกำกวมแต่แก้ชื่อไม่ได้เพราะกระทบระบบเดิม comment ช่วยอธิบายเจตนาได้โดยไม่ต้องแก้โครงสร้าง

### แนวปฏิบัติที่แนะนำ

- เขียน comment ให้กับ **ทุกตาราง** อย่างน้อยหนึ่งประโยคสรุปว่าตารางนี้เก็บอะไร
- เขียน comment ให้กับ **คอลัมน์ที่มี business logic ซับซ้อน** หรือ **หน่วยวัด** ที่ไม่ชัดเจนจากชื่อ เช่น ราคาเป็นสกุลเงินอะไร, เวลาเก็บเป็น timezone ไหน, ค่าที่เป็นไปได้ของ enum-like column มีอะไรบ้าง
- ไม่จำเป็นต้อง comment ทุกคอลัมน์ที่ชื่อสื่อความหมายชัดเจนอยู่แล้ว (เช่น `created_at TIMESTAMPTZ` ไม่จำเป็นต้อง comment ว่า "วันที่สร้าง")
- ใส่ comment เป็นส่วนหนึ่งของ migration script เสมอ ไม่ใช่มาเพิ่มทีหลังแบบแยกขั้นตอน

---

## Step 80: โปรเจกต์ Mini — ออกแบบและสร้างตารางระบบ "ร้านกาแฟ"

ถึงเวลานำทุกสิ่งที่เรียนมาในบทนี้มาประยุกต์ใช้จริง เราจะออกแบบฐานข้อมูลสำหรับระบบร้านกาแฟ (Coffee Shop) ตั้งแต่การวิเคราะห์ requirement ไปจนถึงการเขียน SQL สร้างตารางที่สมบูรณ์ โครงสร้างนี้จะถูกนำไปใช้ต่อเนื่องในบทถัด ๆ ไปของหลักสูตร (INSERT, SELECT ขั้นสูง, JOIN, Index, ฯลฯ) จึงควรทำความเข้าใจให้ถ่องแท้

### ขั้นตอนที่ 1: วิเคราะห์ Requirement

ร้านกาแฟของเราต้องการระบบที่รองรับความสามารถดังนี้:

1. เก็บรายการสินค้า (เมนูกาแฟ, ขนม) พร้อมราคาและหมวดหมู่
2. เก็บข้อมูลลูกค้าที่เป็นสมาชิก (membership)
3. บันทึกคำสั่งซื้อแต่ละครั้ง พร้อมรายการสินค้าที่สั่งในแต่ละคำสั่งซื้อ
4. รองรับได้ทั้งลูกค้าที่เป็นสมาชิกและลูกค้าทั่วไป (walk-in ที่ไม่ต้องสมัครสมาชิก)
5. ราคาสินค้าที่แสดงในใบเสร็จเก่าต้องไม่เปลี่ยนแปลงแม้ราคาปัจจุบันของสินค้าจะเปลี่ยนไปแล้ว
6. รองรับหมวดหมู่สินค้า เช่น เครื่องดื่มร้อน, เครื่องดื่มเย็น, เบเกอรี่

### ขั้นตอนที่ 2: ระบุ Entity หลักและความสัมพันธ์

จาก requirement ข้างต้น เราสามารถระบุ entity หลักได้ดังนี้:

- **categories** (หมวดหมู่สินค้า) — 1 หมวดหมู่มีได้หลายสินค้า (1-to-many กับ products)
- **products** (สินค้า/เมนู) — สินค้าหนึ่งชิ้นอยู่ในหมวดหมู่เดียว แต่ถูกสั่งซื้อได้หลายครั้ง
- **customers** (ลูกค้าสมาชิก) — ลูกค้าหนึ่งคนสั่งซื้อได้หลายครั้ง (1-to-many กับ orders)
- **orders** (คำสั่งซื้อ/บิล) — คำสั่งซื้อหนึ่งใบมีสินค้าได้หลายรายการ (many-to-many กับ products ผ่าน order_items)
- **order_items** (รายการสินค้าในแต่ละคำสั่งซื้อ) — junction table เชื่อม orders กับ products

แผนภาพความสัมพันธ์แบบข้อความ:

```
categories (1) ──< (many) products
customers  (1) ──< (many) orders
orders     (1) ──< (many) order_items >── (many) products
```

### ขั้นตอนที่ 3: สร้างฐานข้อมูลและ Schema

```sql
-- สร้างฐานข้อมูลสำหรับโปรเจกต์ (รันครั้งเดียวนอก transaction)
CREATE DATABASE coffee_shop;
```

จากนี้ไปให้เชื่อมต่อเข้าฐานข้อมูล `coffee_shop` ก่อนรันคำสั่งถัดไป:

```sql
\c coffee_shop
```

### ขั้นตอนที่ 4: สร้างตาราง categories

```sql
CREATE TABLE categories (
    id          SMALLINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name        VARCHAR(100) NOT NULL,
    description TEXT,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    CONSTRAINT uq_categories_name UNIQUE (name)
);

COMMENT ON TABLE categories IS 'หมวดหมู่ของสินค้าในร้าน เช่น เครื่องดื่มร้อน, เครื่องดื่มเย็น, เบเกอรี่';
COMMENT ON COLUMN categories.name IS 'ชื่อหมวดหมู่ ต้องไม่ซ้ำกัน';
```

**เหตุผลการออกแบบ:**
- ใช้ `SMALLINT GENERATED ALWAYS AS IDENTITY` แทน `SERIAL`/`BIGINT` เพราะหมวดหมู่สินค้าร้านกาแฟมีจำนวนน้อยมาก (ไม่เกินหลักสิบ) การใช้ `SMALLINT` ประหยัดพื้นที่โดยไม่มีความเสี่ยงเรื่อง overflow เลย (`SMALLINT` รองรับถึง 32,767 ซึ่งเกินพอ)
- `name` มี `UNIQUE` constraint เพราะไม่ควรมีหมวดหมู่ซ้ำชื่อกัน (เช่น "เครื่องดื่มร้อน" ซ้ำสองแถว)
- `description` เป็น `TEXT` และไม่มี `NOT NULL` เพราะไม่ใช่ข้อมูลบังคับ อาจมีบางหมวดหมู่ที่ไม่จำเป็นต้องอธิบายเพิ่ม

### ขั้นตอนที่ 5: สร้างตาราง products

```sql
CREATE TABLE products (
    id              BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    category_id     SMALLINT NOT NULL REFERENCES categories(id),
    sku             VARCHAR(30) NOT NULL,
    name            VARCHAR(200) NOT NULL,
    description     TEXT,
    price           NUMERIC(10, 2) NOT NULL CHECK (price >= 0),
    is_available    BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    CONSTRAINT uq_products_sku UNIQUE (sku)
);

COMMENT ON TABLE products IS 'สินค้า/เมนูทั้งหมดที่ร้านมีให้บริการ ทั้งที่ยังขายอยู่และเลิกขายแล้ว';
COMMENT ON COLUMN products.sku IS 'รหัสสินค้าภายในร้าน (Stock Keeping Unit) ใช้สำหรับอ้างอิงในระบบสต๊อกและ POS';
COMMENT ON COLUMN products.price IS 'ราคาปัจจุบันของสินค้า หน่วยเป็นบาท (THB) ราคาในใบเสร็จเก่าจะเก็บแยกไว้ที่ order_items.unit_price';
COMMENT ON COLUMN products.is_available IS 'สถานะว่าสินค้านี้ยังเปิดขายอยู่หรือไม่ ใช้ soft-hide แทนการลบแถวออกจากตาราง';
```

**เหตุผลการออกแบบ:**
- `category_id` เป็น `NOT NULL REFERENCES categories(id)` — บังคับให้ทุกสินค้าต้องมีหมวดหมู่ เพราะเป็นข้อมูลที่จำเป็นสำหรับการจัดกลุ่มเมนู
- `sku` เป็น `UNIQUE` เพื่อป้องกันรหัสสินค้าซ้ำ ซึ่งจะกระทบระบบสต๊อกถ้าซ้ำกัน
- `price` มี `CHECK (price >= 0)` ป้องกันการใส่ราคาติดลบโดยผิดพลาด
- `is_available BOOLEAN` ใช้แทนการลบสินค้าจริง (soft delete pattern) เพราะสินค้าที่เลิกขายแล้วยังต้องถูกอ้างอิงจาก `order_items` ของคำสั่งซื้อในอดีตอยู่ ถ้าลบตารางจริงจะทำให้ประวัติการขายเสียหาย
- `updated_at` มีไว้เพื่อติดตามว่าข้อมูลสินค้าถูกแก้ไขล่าสุดเมื่อไหร่ (จะมาคู่กับ trigger auto-update ในบทขั้นสูงถัดไป)

### ขั้นตอนที่ 6: สร้างตาราง customers

```sql
CREATE TABLE customers (
    id              BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    email           VARCHAR(255) NOT NULL,
    phone           VARCHAR(20),
    full_name       VARCHAR(200) NOT NULL,
    membership_tier VARCHAR(20) NOT NULL DEFAULT 'standard',
    joined_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    CONSTRAINT uq_customers_email UNIQUE (email),
    CONSTRAINT chk_customers_membership_tier
        CHECK (membership_tier IN ('standard', 'silver', 'gold', 'platinum'))
);

COMMENT ON TABLE customers IS 'ลูกค้าที่สมัครสมาชิกร้าน ลูกค้า walk-in ที่ไม่สมัครสมาชิกจะไม่มีแถวในตารางนี้ (ดู orders.customer_id ที่อนุญาตให้เป็น NULL ได้)';
COMMENT ON COLUMN customers.membership_tier IS 'ระดับสมาชิก จำกัดค่าที่เป็นไปได้ด้วย CHECK constraint: standard, silver, gold, platinum';
```

**เหตุผลการออกแบบ:**
- `email` เป็น `UNIQUE` เพราะใช้เป็นตัวระบุตัวตนหลักในการ login/ติดต่อ แต่ **ไม่ใช้เป็น Primary Key** — ยึดหลัก surrogate key ตาม Step 73 เพราะอีเมลอาจเปลี่ยนแปลงได้ในอนาคต
- `phone` ไม่มี `NOT NULL` เพราะบางระบบสมัครสมาชิกผ่านอีเมลอย่างเดียวได้ ไม่บังคับเบอร์โทร
- `membership_tier` ใช้ `CHECK` constraint จำกัดค่าที่เป็นไปได้ แทนที่จะปล่อยให้เป็น string อิสระ ป้องกันการใส่ค่าผิดเพี้ยน เช่น `"Gold"` ตัวใหญ่ปนตัวเล็ก หรือ `"golde"` พิมพ์ผิด (ในบทขั้นสูงเรื่อง ENUM type เราจะเห็นอีกวิธีที่ดีกว่านี้)

### ขั้นตอนที่ 7: สร้างตาราง orders

```sql
CREATE TABLE orders (
    id             BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    customer_id    BIGINT REFERENCES customers(id),
    order_number   VARCHAR(20) NOT NULL,
    status         VARCHAR(20) NOT NULL DEFAULT 'pending',
    ordered_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    CONSTRAINT uq_orders_order_number UNIQUE (order_number),
    CONSTRAINT chk_orders_status
        CHECK (status IN ('pending', 'paid', 'preparing', 'completed', 'cancelled'))
);

COMMENT ON TABLE orders IS 'คำสั่งซื้อ/บิลแต่ละใบของร้าน หนึ่งแถวคือหนึ่งบิล (ไม่ใช่หนึ่งสินค้า)';
COMMENT ON COLUMN orders.customer_id IS 'ผู้สั่งซื้อ อนุญาตให้เป็น NULL ได้สำหรับลูกค้า walk-in ที่ไม่สมัครสมาชิก';
COMMENT ON COLUMN orders.order_number IS 'เลขที่บิลที่แสดงให้ลูกค้าเห็น (human-readable) แยกจาก id ที่เป็น internal surrogate key';
COMMENT ON COLUMN orders.status IS 'สถานะของคำสั่งซื้อ: pending (รอชำระ), paid (ชำระแล้ว), preparing (กำลังทำ), completed (เสร็จสิ้น), cancelled (ยกเลิก)';
```

**เหตุผลการออกแบบ:**
- `customer_id` **ไม่ใส่ `NOT NULL`** โดยตั้งใจ เพื่อรองรับลูกค้า walk-in ที่ไม่ได้สมัครสมาชิก — นี่คือการตัดสินใจออกแบบที่สำคัญตาม requirement ข้อ 4
- `order_number` แยกจาก `id` เพราะ `id` เป็น surrogate key ภายใน (internal) ที่ไม่ควรโชว์ให้ลูกค้าเห็นตรง ๆ ส่วน `order_number` เป็นรูปแบบที่ธุรกิจกำหนดเอง เช่น `"ORD-20260925-0001"` ที่มีความหมายและอ่านง่ายกว่า
- `status` ใช้ `CHECK` จำกัดค่าที่เป็นไปได้ ป้องกันสถานะที่ไม่ถูกต้องหลุดเข้าระบบ
- **หมายเหตุสำคัญ**: ตารางนี้ **ไม่มีคอลัมน์ `total_amount`** ในเวอร์ชันนี้ เพราะยังไม่ได้เรียนเรื่อง Trigger หรือ Aggregate query ที่จะมาช่วยรักษาความสอดคล้องของยอดรวม (ตามหลักการใน Step 78) ยอดรวมจะคำนวณจาก `order_items` ด้วย query ในบทที่ว่าด้วย JOIN และ Aggregate Function — นี่เป็นตัวอย่างของการเริ่มต้นด้วย normalized design ก่อนเสมอ แล้วค่อยพิจารณา denormalize ทีหลังเมื่อมีเหตุผลชัดเจนและมีเครื่องมือ (trigger) พร้อมใช้งาน

### ขั้นตอนที่ 8: สร้างตาราง order_items

```sql
CREATE TABLE order_items (
    order_id    BIGINT NOT NULL REFERENCES orders(id),
    product_id  BIGINT NOT NULL REFERENCES products(id),
    quantity    INTEGER NOT NULL CHECK (quantity > 0),
    unit_price  NUMERIC(10, 2) NOT NULL CHECK (unit_price >= 0),
    subtotal    NUMERIC(12, 2) GENERATED ALWAYS AS (quantity * unit_price) STORED,

    PRIMARY KEY (order_id, product_id)
);

COMMENT ON TABLE order_items IS 'รายการสินค้าแต่ละชิ้นในแต่ละคำสั่งซื้อ (junction table ระหว่าง orders กับ products)';
COMMENT ON COLUMN order_items.unit_price IS
    'ราคาต่อหน่วย ณ เวลาที่สั่งซื้อ (snapshot) จงใจเก็บแยกจาก products.price '
    'เพื่อไม่ให้ใบเสร็จเก่าเปลี่ยนแปลงตามราคาปัจจุบันของสินค้า';
COMMENT ON COLUMN order_items.subtotal IS 'ยอดรวมของรายการนี้ คำนวณอัตโนมัติจาก quantity คูณ unit_price';
```

**เหตุผลการออกแบบ:**
- `PRIMARY KEY (order_id, product_id)` เป็น composite key — สมเหตุสมผลเพราะสินค้าชิ้นเดียวกันไม่ควรปรากฏซ้ำสองแถวในบิลเดียวกัน (ถ้าลูกค้าสั่งกาแฟลาเต้ 2 แก้ว ควรเก็บเป็นแถวเดียวที่ `quantity = 2` ไม่ใช่ 2 แถวแยกกัน)
- `unit_price` เก็บแยกจาก `products.price` โดยจงใจ — นี่คือตัวอย่างการ denormalize ที่ถูกต้องตามหลักการใน Step 78 เพราะเป็นข้อมูลเชิงประวัติศาสตร์ (historical fact) ที่ต้องไม่เปลี่ยนตามราคาปัจจุบัน
- `subtotal` ใช้ Generated Column ตามที่เรียนใน Step 76 แทนที่จะคำนวณเองทุกครั้งตอน query หรือใช้ trigger ที่ซับซ้อนกว่า
- ทั้ง `quantity` และ `unit_price` มี `CHECK` constraint ป้องกันค่าที่ผิดปกติ (จำนวนติดลบหรือศูนย์, ราคาติดลบ)

### ขั้นตอนที่ 9: สร้าง Index เสริมสำหรับ Foreign Key (เตรียมพร้อมสำหรับ performance)

Foreign Key ใน PostgreSQL **ไม่ได้สร้าง index ให้อัตโนมัติ** (ต่างจาก Primary Key ที่มี index ให้เสมอ) ในทางปฏิบัติควรสร้าง index บนคอลัมน์ foreign key เพื่อให้ JOIN และการค้นหาทำงานได้เร็ว (รายละเอียดเชิงลึกเรื่อง Index จะอยู่ในบทถัดไปของหลักสูตร แต่ในที่นี้ขอแนะนำเบื้องต้นเพื่อความสมบูรณ์ของโปรเจกต์)

```sql
CREATE INDEX idx_products_category_id ON products (category_id);
CREATE INDEX idx_orders_customer_id ON orders (customer_id);
CREATE INDEX idx_order_items_product_id ON order_items (product_id);
```

> หมายเหตุ: `order_items.order_id` ไม่จำเป็นต้องสร้าง index แยก เพราะเป็นคอลัมน์แรกของ composite primary key `(order_id, product_id)` ซึ่ง PostgreSQL สร้าง index ให้อัตโนมัติอยู่แล้วและ index แบบ B-tree บนคอลัมน์แรกของ composite key ใช้ค้นหาด้วย `order_id` เพียงตัวเดียวได้อย่างมีประสิทธิภาพ

### ขั้นตอนที่ 10: ตรวจสอบโครงสร้างทั้งหมด

```sql
-- ดูรายการตารางทั้งหมดในฐานข้อมูล
\dt

-- ดูโครงสร้างและ constraint ของแต่ละตารางโดยละเอียด
\d+ categories
\d+ products
\d+ customers
\d+ orders
\d+ order_items
```

ผลลัพธ์ที่คาดหวังจาก `\dt`:

```
              List of relations
 Schema |     Name     | Type  |  Owner
--------+--------------+-------+----------
 public | categories   | table | postgres
 public | customers    | table | postgres
 public | order_items  | table | postgres
 public | orders       | table | postgres
 public | products     | table | postgres
```

### สรุปแผนภาพความสัมพันธ์ฉบับสมบูรณ์

```
┌──────────────┐       ┌──────────────┐
│  categories  │       │  customers   │
│──────────────│       │──────────────│
│ id (PK)      │       │ id (PK)      │
│ name         │       │ email        │
│ description  │       │ phone        │
└──────┬───────┘       │ full_name    │
       │ 1              │ membership_ │
       │                │   tier      │
       │ many           └──────┬───────┘
┌──────▼───────┐               │ 1 (nullable)
│   products   │               │
│──────────────│               │ many
│ id (PK)      │       ┌──────▼───────┐
│ category_id  │(FK)   │    orders    │
│ sku          │       │──────────────│
│ name         │       │ id (PK)      │
│ price        │       │ customer_id  │(FK, nullable)
│ is_available │       │ order_number │
└──────┬───────┘       │ status       │
       │ many          └──────┬───────┘
       │                      │ 1
       │              ┌───────▼──────┐
       └─────many─────►  order_items  │
                      │──────────────│
                      │ order_id(PK,FK)
                      │ product_id(PK,FK)
                      │ quantity     │
                      │ unit_price   │
                      │ subtotal (generated)
                      └──────────────┘
```

โครงสร้างนี้ผ่าน Normalization ถึงระดับ 3NF ทุกตาราง (ยกเว้นจุดที่จงใจ denormalize อย่างมีเหตุผลคือ `order_items.unit_price`) และพร้อมใช้เป็นรากฐานสำหรับบทถัดไปที่จะสอนการ `INSERT` ข้อมูลจริงเข้าไปในตารางเหล่านี้

---

## สรุปท้ายบท

ในบทนี้เราได้เรียนรู้การสร้างตารางใน PostgreSQL อย่างครบวงจร ตั้งแต่ syntax พื้นฐานไปจนถึงการออกแบบ schema ระดับมืออาชีพ:

- **Syntax ของ `CREATE TABLE`** ทั้งแบบ column-level และ table-level constraints และเมื่อไหร่ควรใช้แบบไหน
- **การตั้งชื่อ** ด้วย snake_case อย่างสม่ำเสมอ และการหลีกเลี่ยง reserved word เพื่อไม่ต้องพึ่ง quoted identifier
- **การเลือก Primary Key** ระหว่าง natural key กับ surrogate key โดยยึดหลัก "surrogate key เป็นค่าเริ่มต้น" เว้นแต่มีเหตุผลชัดเจนที่จะใช้ natural key
- **เครื่องมือช่วยสร้างตารางลดงานซ้ำ** ได้แก่ `IF NOT EXISTS` สำหรับ idempotent script, `LIKE` สำหรับคัดลอกโครงสร้าง, และ `CTAS` สำหรับสร้างตารางจากผลลัพธ์ query
- **Temporary Table และ Unlogged Table** สำหรับกรณีพิเศษที่ต้องการความเร็วสูงหรือข้อมูลชั่วคราว โดยแลกกับความทนทานของข้อมูล (durability)
- **Generated Column** สำหรับคำนวณค่าอัตโนมัติจากคอลัมน์อื่นในแถวเดียวกัน ลดความซับซ้อนเทียบกับการใช้ trigger
- **Normalization** ตั้งแต่ 1NF, 2NF, ถึง 3NF พร้อมเหตุผลของแต่ละระดับ และปัญหา anomaly ที่ normalization แก้ไขให้
- **Denormalization** และ trade-off ที่ต้องพิจารณา รวมถึงหลักการว่าเมื่อไหร่ควรเริ่ม denormalize
- **`COMMENT ON`** สำหรับทำเอกสารประกอบ schema ที่อยู่คู่กับฐานข้อมูลตลอดไป
- **โปรเจกต์ระบบร้านกาแฟ** ที่นำทุกหลักการมาประยุกต์ใช้จริง ตั้งแต่การวิเคราะห์ requirement จนถึงการเขียน SQL ที่สมบูรณ์ พร้อมคำอธิบายเหตุผลของทุกการตัดสินใจในการออกแบบ

Schema ของร้านกาแฟที่สร้างในบทนี้ (`categories`, `products`, `customers`, `orders`, `order_items`) จะถูกใช้ต่อเนื่องในบทถัดไปเพื่อฝึก `INSERT` ข้อมูลจริง ดังนั้นควรรันคำสั่งทั้งหมดในเครื่องของตัวเองให้เรียบร้อยก่อนไปต่อ

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1

จงเขียน `CREATE TABLE` สำหรับตาราง `books` ที่มีคอลัมน์ `id` (surrogate key), `isbn` (ต้องไม่ซ้ำ), `title` (ห้ามว่าง), `price` (ต้องไม่ติดลบ) โดยใช้ table-level constraint ทั้งหมด และตั้งชื่อ constraint ให้สื่อความหมาย

<details>
<summary>เฉลย</summary>

```sql
CREATE TABLE books (
    id      BIGINT GENERATED ALWAYS AS IDENTITY,
    isbn    VARCHAR(20),
    title   VARCHAR(300),
    price   NUMERIC(10, 2),

    CONSTRAINT pk_books PRIMARY KEY (id),
    CONSTRAINT uq_books_isbn UNIQUE (isbn),
    CONSTRAINT nn_books_title CHECK (title IS NOT NULL),
    CONSTRAINT chk_books_price_non_negative CHECK (price >= 0)
);
```

หมายเหตุ: `NOT NULL` โดยทั่วไปนิยมเขียนแบบ column-level (`title VARCHAR(300) NOT NULL`) มากกว่าใช้ `CHECK` แต่โจทย์นี้ต้องการฝึกเขียนแบบ table-level ล้วน ๆ ซึ่งก็ทำได้ผ่าน `CHECK (title IS NOT NULL)` เช่นกัน (แม้ไม่ใช่วิธีที่นิยมที่สุดในทางปฏิบัติ)

</details>

### แบบฝึกหัดที่ 2

ตารางต่อไปนี้มีปัญหาเรื่องการตั้งชื่อและ reserved word จงระบุปัญหาและแก้ไข

```sql
CREATE TABLE Order (
    orderID INTEGER,
    "Customer Email" VARCHAR(255),
    Group VARCHAR(50)
);
```

<details>
<summary>เฉลย</summary>

ปัญหาที่พบ:
1. `Order` เป็น reserved word และใช้ตัวพิมพ์ใหญ่ (ต้อง quote ทุกครั้งที่ใช้)
2. `orderID` ใช้ camelCase แทนที่จะเป็น snake_case
3. `"Customer Email"` มีเว้นวรรค ต้อง quote ตลอดเวลาที่ใช้งาน
4. `Group` เป็น reserved word เช่นกัน

แก้ไข:

```sql
CREATE TABLE orders (
    order_id       INTEGER,
    customer_email VARCHAR(255),
    group_name     VARCHAR(50)
);
```

</details>

### แบบฝึกหัดที่ 3

จงอธิบายว่าทำไมการใช้ `national_id` (เลขบัตรประชาชน) เป็น Primary Key โดยตรงจึงไม่ใช่แนวทางที่แนะนำ และเสนอทางเลือกที่ดีกว่า

<details>
<summary>เฉลย</summary>

เหตุผลที่ไม่แนะนำให้ใช้ `national_id` เป็น PK โดยตรง:

1. ข้อมูลอาจผิดพลาดหรือไม่สมบูรณ์ได้ (เช่น ชาวต่างชาติที่ไม่มีเลขบัตรประชาชนไทย)
2. ถ้าพบว่าข้อมูลกรอกผิดในภายหลัง การแก้ไข PK จะกระทบ Foreign Key ที่อ้างอิงในทุกตาราง (cascading effect)
3. เป็นข้อมูลอ่อนไหว (sensitive data) การใช้เป็น PK ที่ปรากฏใน URL หรือ log ต่าง ๆ เพิ่มความเสี่ยงด้านความเป็นส่วนตัว

ทางเลือกที่ดีกว่า: ใช้ surrogate key เช่น `id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY` แล้วเก็บ `national_id` เป็นคอลัมน์ธรรมดาที่มี `UNIQUE` constraint แยกต่างหาก

```sql
CREATE TABLE citizens (
    id            BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    national_id   VARCHAR(13) UNIQUE,
    full_name     VARCHAR(200) NOT NULL
);
```

</details>

### แบบฝึกหัดที่ 4

จงเขียนคำสั่งสร้างตาราง `products_backup` ที่มีโครงสร้างคอลัมน์เหมือนตาราง `products` (จากโปรเจกต์ร้านกาแฟ) ทุกประการ รวมถึง constraint และ index ทั้งหมด แต่ไม่ต้องคัดลอกข้อมูล

<details>
<summary>เฉลย</summary>

```sql
CREATE TABLE products_backup (LIKE products INCLUDING ALL);
```

`INCLUDING ALL` จะคัดลอก DEFAULT, CONSTRAINTS (รวม CHECK), INDEXES (รวม PK/UNIQUE), COMMENTS, IDENTITY และ STORAGE ทั้งหมด แต่จะไม่คัดลอก Foreign Key constraint (ต้องเพิ่มเองถ้าต้องการ เพราะ `LIKE` ไม่คัดลอก FK ไม่ว่าจะระบุ option ใดก็ตาม) และไม่คัดลอกข้อมูลจริง (คัดลอกแค่โครงสร้าง)

</details>

### แบบฝึกหัดที่ 5

จงอธิบายความแตกต่างระหว่าง Temporary Table และ Unlogged Table โดยยกตัวอย่างสถานการณ์ที่เหมาะกับแต่ละแบบ

<details>
<summary>เฉลย</summary>

**Temporary Table**: มีอายุอยู่แค่ใน session เดียว หายไปอัตโนมัติเมื่อ disconnect แต่ละ session เห็นเฉพาะ temp table ของตัวเอง เหมาะกับการคำนวณขั้นกลางที่ซับซ้อนภายใน session เดียว เช่น การเตรียมข้อมูลสำหรับรายงานที่ต้องผ่านหลายขั้นตอนก่อนสรุปผล

**Unlogged Table**: อยู่ถาวรข้าม session ได้เหมือนตารางปกติ แต่ไม่บันทึกลง WAL ทำให้เขียนเร็วกว่า แลกกับการที่ข้อมูลจะหายไปถ้าเซิร์ฟเวอร์ crash และไม่ replicate ไป standby เหมาะกับข้อมูล cache หรือ session data ที่ regenerate ใหม่ได้ถ้าหาย เช่น ตารางเก็บ session token ชั่วคราวของเว็บแอปพลิเคชัน

</details>

### แบบฝึกหัดที่ 6

จากตาราง `order_items` ในโปรเจกต์ร้านกาแฟ จงอธิบายว่าทำไม `subtotal` จึงใช้ Generated Column แทนการคำนวณผ่าน Trigger หรือการคำนวณใน Application layer

<details>
<summary>เฉลย</summary>

Generated Column ให้ข้อดีดังนี้เทียบกับทางเลือกอื่น:

- เทียบกับ Trigger: Generated Column เขียนโค้ดสั้นกว่ามาก (บรรทัดเดียวในนิยามคอลัมน์) และ PostgreSQL รับประกันว่าค่าจะถูกคำนวณใหม่ทุกครั้งที่ `quantity` หรือ `unit_price` เปลี่ยน โดยไม่มีโอกาสที่นักพัฒนาจะลืมเขียน trigger ให้ครบทุก event (INSERT/UPDATE)
- เทียบกับการคำนวณใน Application layer: ถ้าคำนวณใน application อาจมีบางจุดของระบบ (เช่น สคริปต์ migrate ข้อมูล, การแก้ไขผ่าน SQL โดยตรง) ที่ไม่ผ่าน application logic ทำให้ `subtotal` ไม่ตรงกับความเป็นจริง Generated Column รับประกันความถูกต้องที่ระดับฐานข้อมูลเสมอ ไม่ว่าจะเขียนข้อมูลผ่านช่องทางไหน

</details>

### แบบฝึกหัดที่ 7

ตารางต่อไปนี้ละเมิด Normal Form ข้อใด จงระบุและแก้ไขให้ถูกต้อง

```sql
CREATE TABLE students_courses (
    student_id    INTEGER,
    student_name  VARCHAR(200),
    course_ids    VARCHAR(500)   -- เก็บ '101,102,205' คั่นด้วย comma
);
```

<details>
<summary>เฉลย</summary>

ละเมิด **1NF** เพราะคอลัมน์ `course_ids` เก็บหลายค่าไว้ในเซลล์เดียว (ไม่ atomic)

แก้ไขโดยแยกเป็นตารางความสัมพันธ์ (junction table):

```sql
CREATE TABLE students (
    id    SERIAL PRIMARY KEY,
    name  VARCHAR(200) NOT NULL
);

CREATE TABLE courses (
    id    SERIAL PRIMARY KEY,
    name  VARCHAR(200) NOT NULL
);

CREATE TABLE student_courses (
    student_id  INTEGER NOT NULL REFERENCES students(id),
    course_id   INTEGER NOT NULL REFERENCES courses(id),
    PRIMARY KEY (student_id, course_id)
);
```

</details>

### แบบฝึกหัดที่ 8

จงยกตัวอย่างสถานการณ์ 2 แบบที่การ denormalize เหมาะสม พร้อมอธิบายเหตุผลและความเสี่ยงที่ต้องจัดการ

<details>
<summary>เฉลย</summary>

**ตัวอย่างที่ 1**: เก็บราคาสินค้า ณ เวลาขาย (`order_items.unit_price`) แยกจากราคาปัจจุบันของสินค้า (`products.price`) — เหตุผลคือข้อมูลในใบเสร็จต้องเป็น historical fact ที่ไม่เปลี่ยนตามราคาปัจจุบัน ความเสี่ยงคือถ้านักพัฒนาลืมเก็บ `unit_price` ตอน insert แล้วดึงจาก `products.price` แทน จะทำให้ประวัติผิดเพี้ยนเมื่อราคาสินค้าถูกเปลี่ยนในอนาคต จึงต้องมี business logic หรือ default ที่ถูกต้องตอน insert เสมอ

**ตัวอย่างที่ 2**: เก็บยอดรวม `total_amount` ไว้ในตาราง `orders` แทนที่จะคำนวณจาก `SUM` ของ `order_items` ทุกครั้ง — เหตุผลคือช่วยให้หน้าจอแสดงประวัติคำสั่งซื้อโหลดเร็วขึ้นมากเมื่อมีข้อมูลจำนวนมาก ความเสี่ยงคือต้องมีกลไก (เช่น trigger) คอยอัปเดต `total_amount` ทุกครั้งที่ `order_items` มีการเปลี่ยนแปลง ไม่เช่นนั้นยอดรวมที่แสดงจะไม่ตรงกับความจริง

</details>

### แบบฝึกหัดที่ 9

จงเขียนคำสั่ง `COMMENT ON` สำหรับตาราง `orders` และคอลัมน์ `status` ในโปรเจกต์ร้านกาแฟ โดยอธิบายว่าค่าที่เป็นไปได้ของ `status` มีอะไรบ้างและแต่ละค่าหมายถึงอะไร

<details>
<summary>เฉลย</summary>

```sql
COMMENT ON TABLE orders IS
    'คำสั่งซื้อ/บิลแต่ละใบของร้าน หนึ่งแถวแทนหนึ่งบิล ไม่ใช่หนึ่งสินค้า';

COMMENT ON COLUMN orders.status IS
    'สถานะคำสั่งซื้อ: '
    'pending = รอชำระเงิน, '
    'paid = ชำระเงินแล้ว, '
    'preparing = กำลังเตรียมเครื่องดื่ม/อาหาร, '
    'completed = ส่งมอบสินค้าเรียบร้อยแล้ว, '
    'cancelled = ยกเลิกคำสั่งซื้อ';
```

</details>

### แบบฝึกหัดที่ 10

จงออกแบบตารางเพิ่มเติมชื่อ `payments` สำหรับระบบร้านกาแฟ ที่เก็บข้อมูลการชำระเงินของแต่ละคำสั่งซื้อ โดยคำนึงถึง: หนึ่งคำสั่งซื้ออาจชำระเงินได้มากกว่าหนึ่งครั้ง (เช่น จ่ายบางส่วนด้วยเงินสด บางส่วนด้วยบัตร), ต้องเก็บวิธีการชำระเงิน (เงินสด/บัตร/QR code), และต้องเก็บจำนวนเงินและเวลาที่ชำระ

<details>
<summary>เฉลย</summary>

```sql
CREATE TABLE payments (
    id              BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    order_id        BIGINT NOT NULL REFERENCES orders(id),
    payment_method  VARCHAR(20) NOT NULL,
    amount          NUMERIC(10, 2) NOT NULL CHECK (amount > 0),
    paid_at         TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    CONSTRAINT chk_payments_method
        CHECK (payment_method IN ('cash', 'credit_card', 'qr_code', 'e_wallet'))
);

CREATE INDEX idx_payments_order_id ON payments (order_id);

COMMENT ON TABLE payments IS
    'การชำระเงินของแต่ละคำสั่งซื้อ หนึ่งคำสั่งซื้ออาจมีได้หลายรายการชำระเงิน '
    '(เช่น แบ่งจ่ายเงินสดบางส่วนและบัตรบางส่วน)';
COMMENT ON COLUMN payments.payment_method IS
    'วิธีการชำระเงิน: cash (เงินสด), credit_card (บัตรเครดิต/เดบิต), '
    'qr_code (พร้อมเพย์/QR), e_wallet (กระเป๋าเงินอิเล็กทรอนิกส์)';
```

ออกแบบเป็นตารางแยกต่างหาก (ไม่รวมเข้ากับ `orders`) เพราะความสัมพันธ์เป็น 1-to-many (หนึ่งคำสั่งซื้อมีได้หลายการชำระเงิน) ถ้ารวมไว้ใน `orders` โดยตรงจะไม่สามารถรองรับการแบ่งจ่ายหลายวิธีได้ และจะกลับไปละเมิด 1NF เหมือนปัญหาที่อธิบายไว้ใน Step 77

</details>

---

## บทถัดไป

เมื่อออกแบบและสร้างตารางครบถ้วนแล้ว ขั้นตอนถัดไปคือการเติมข้อมูลเข้าไปในตารางเหล่านี้ ไปต่อกันที่ **[Part 009: การเพิ่มข้อมูลด้วย INSERT](./part-009-insert.md)** ซึ่งจะสอนการ insert ข้อมูลหลากหลายรูปแบบ ทั้งแบบทีละแถว หลายแถวพร้อมกัน, `INSERT ... RETURNING`, `INSERT ... ON CONFLICT` (upsert), และการเติมข้อมูลตัวอย่างเข้าสู่ schema ร้านกาแฟที่สร้างไว้ในบทนี้
