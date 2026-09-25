# Part 006: Data Types พื้นฐาน — ตัวเลข, ข้อความ, Boolean

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับพื้นฐาน | Part 006

---

## เป้าหมายการเรียนรู้

เมื่อเรียนจบ Part นี้ ผู้เรียนจะสามารถ:

- อธิบายได้ว่า Data Type ใน PostgreSQL คืออะไร และทำไมการเลือก type ให้เหมาะสมถึงสำคัญต่อความถูกต้องของข้อมูล ประสิทธิภาพ และพื้นที่จัดเก็บ
- เลือกใช้ integer type (`smallint`, `integer`, `bigint`) ได้อย่างเหมาะสมตามช่วงค่าที่ต้องการ
- เข้าใจ serial types (`smallserial`, `serial`, `bigserial`) เบื้องต้น และรู้ว่ามันคือ "ทางลัด" ในการสร้างค่า auto-increment
- แยกแยะความแตกต่างระหว่าง `numeric`/`decimal` (exact) กับ `real`/`double precision` (floating point) และรู้ว่าเมื่อไหร่ควรใช้ตัวไหน
- อธิบายปัญหา floating point precision ได้ พร้อมตัวอย่างที่จับต้องได้จริง
- เข้าใจว่าทำไม PostgreSQL แนะนำให้ใช้ `numeric` แทน `money` ในเกือบทุกกรณี
- เลือกใช้ character type (`char(n)`, `varchar(n)`, `text`) ได้อย่างถูกต้อง โดยไม่หลงเชื่อความเชื่อผิดๆ เรื่อง performance
- ใช้งาน `boolean` type ได้อย่างถูกต้อง รวมถึงเข้าใจพฤติกรรมของ `NULL` ใน boolean
- แปลงชนิดข้อมูล (type casting) ระหว่างตัวเลขและข้อความได้อย่างปลอดภัย ด้วย `CAST()` และ `::`
- ออกแบบตารางเลือก data type ให้เหมาะสมกับสถานการณ์จริง เช่น ระบบสต๊อกสินค้า ระบบบัญชี

---

## Step 51: ภาพรวม Data Type ใน PostgreSQL และทำไมการเลือก type ให้เหมาะสมสำคัญ

### Data Type คืออะไร

ทุกๆ คอลัมน์ในตาราง PostgreSQL ต้องมี **data type** กำกับไว้เสมอ data type คือกฎที่บอกว่า:

1. ข้อมูลแบบไหนที่เก็บในคอลัมน์นี้ได้ (ตัวเลข, ข้อความ, วันที่, boolean, ฯลฯ)
2. ข้อมูลจะถูกเก็บในรูปแบบไบต์อย่างไร (storage format) และใช้พื้นที่เท่าไหร่
3. การดำเนินการ (operator) และฟังก์ชันใดที่ใช้กับคอลัมน์นี้ได้
4. ค่าถูกต้อง (validation) อย่างไรก่อนจะถูกเขียนลงตาราง

PostgreSQL เป็นระบบฐานข้อมูลที่ **type safety เข้มงวด** มากกว่า MySQL หรือ SQLite ค่อนข้างมาก หมายความว่า PostgreSQL จะไม่พยายาม "เดา" หรือแปลงชนิดข้อมูลให้อัตโนมัติแบบมั่วๆ ถ้าชนิดข้อมูลไม่ตรงกัน ระบบจะแจ้ง error ทันที นี่คือจุดแข็งที่ทำให้ข้อมูลใน PostgreSQL มีความถูกต้อง (data integrity) สูงกว่า

### ตัวอย่าง: ความเข้มงวดของ PostgreSQL

```sql
-- สร้างตารางทดสอบ
CREATE TABLE type_demo (
    id      integer,
    amount  integer
);

-- ใส่ข้อมูลปกติ ทำงานได้
INSERT INTO type_demo (id, amount) VALUES (1, 100);

-- พยายามใส่ข้อความที่ไม่ใช่ตัวเลขลงคอลัมน์ integer
INSERT INTO type_demo (id, amount) VALUES (2, 'abc');
```

ผลลัพธ์:

```
ERROR:  invalid input syntax for type integer: "abc"
LINE 1: INSERT INTO type_demo (id, amount) VALUES (2, 'abc');
```

PostgreSQL ปฏิเสธทันที ไม่เหมือนบางระบบที่จะแปลง `'abc'` เป็น `0` แบบเงียบๆ (silent conversion) ซึ่งเป็นสาเหตุของ bug ที่ตรวจจับได้ยากในระบบจริง

### หมวดหมู่ Data Type หลักใน PostgreSQL

PostgreSQL มี data type ให้เลือกใช้หลากหลายมาก แบ่งเป็นหมวดหมู่ใหญ่ๆ ได้ดังนี้ (Part นี้จะโฟกัสที่แถบสีเข้ม ส่วนที่เหลือจะเจาะลึกใน Part ถัดๆ ไป):

| หมวดหมู่ | ตัวอย่าง Type | Part ที่เจาะลึก |
|---|---|---|
| **ตัวเลข (Numeric)** | `smallint`, `integer`, `bigint`, `numeric`, `real`, `double precision` | **Part 006 (บทนี้)** |
| **ข้อความ (Character)** | `char(n)`, `varchar(n)`, `text` | **Part 006 (บทนี้)** |
| **Boolean** | `boolean` | **Part 006 (บทนี้)** |
| วันที่และเวลา | `date`, `time`, `timestamp`, `timestamptz`, `interval` | Part 007 |
| Binary | `bytea` | Part 008 |
| UUID | `uuid` | Part 008 |
| JSON | `json`, `jsonb` | Part 062-064 (ระดับกลาง) |
| Array | เช่น `integer[]`, `text[]` | Part 060 |
| Range | `int4range`, `tsrange`, ฯลฯ | Part 061 |
| Network address | `inet`, `cidr`, `macaddr` | Part 059 |
| Geometric | `point`, `line`, `polygon` | ระดับสูง |
| Enum | `CREATE TYPE ... AS ENUM` | Part 058 |

### ทำไมการเลือก Type ให้เหมาะสมถึงสำคัญ

มือใหม่จำนวนมากมักจะ "เผื่อไว้ก่อน" เช่น ใช้ `text` กับทุกคอลัมน์ หรือใช้ `bigint` กับทุกตัวเลข เพราะคิดว่าปลอดภัยไว้ก่อน แต่ในความเป็นจริงการเลือก type ที่เหมาะสมมีผลกระทบในหลายมิติ:

**1. ความถูกต้องของข้อมูล (Correctness)**

ถ้าอายุคนถูกเก็บเป็น `text` ระบบจะยอมให้ใส่ค่า `'สิบแปดปี'` ซึ่งไม่มีความหมายทางคณิตศาสตร์ และไม่สามารถนำไปคำนวณหรือเรียงลำดับได้อย่างถูกต้อง ในขณะที่ `smallint` จะบังคับให้เป็นตัวเลขเสมอ

**2. พื้นที่จัดเก็บ (Storage)**

`bigint` ใช้พื้นที่ 8 ไบต์ต่อแถว ในขณะที่ `smallint` ใช้เพียง 2 ไบต์ ถ้าตารางมีข้อมูลหลักร้อยล้านแถว ความแตกต่างนี้ส่งผลต่อขนาดฐานข้อมูล, ความเร็วในการ backup, และประสิทธิภาพของ cache (shared_buffers) โดยตรง

**3. ประสิทธิภาพ (Performance)**

การเปรียบเทียบตัวเลข (`integer = integer`) เร็วกว่าการเปรียบเทียบข้อความ (`text = text`) เสมอ เพราะ CPU เปรียบเทียบตัวเลขในระดับ binary ได้โดยตรง ในขณะที่ข้อความต้องเปรียบเทียบทีละไบต์ (byte-by-byte) ตาม collation

**4. Index และ Constraint**

Type ที่เหมาะสมทำให้ PostgreSQL สร้าง index ที่มีประสิทธิภาพ และสามารถใช้ constraint (เช่น `CHECK`, `NOT NULL`) มาช่วยรับประกันความถูกต้องได้ตั้งแต่ระดับฐานข้อมูล ไม่ต้องพึ่ง validation ฝั่ง application เพียงอย่างเดียว

**5. ความชัดเจนของ Schema (Self-Documenting)**

เมื่อคนอื่นมาดู schema ของตาราง การเห็นว่าคอลัมน์ `age` เป็น `smallint` และ `price` เป็น `numeric(10,2)` จะสื่อความหมายได้ทันทีว่าค่าที่คาดหวังคืออะไร โดยไม่ต้องเดา

### หลักการเลือก Type เบื้องต้น

ก่อนจะลงรายละเอียดของแต่ละ type ในบทนี้ ให้จำหลักการ 3 ข้อนี้ไว้ก่อน:

1. **เลือก type ที่ "แคบที่สุดเท่าที่พอ"** — อย่าใช้ `bigint` ถ้า `integer` เพียงพอ อย่าใช้ `text` แบบไม่จำกัดถ้ามี domain ที่ชัดเจน (เช่น รหัสไปรษณีย์ 5 หลัก)
2. **ให้ความถูกต้องมาก่อนความสะดวก** — อย่าใช้ `real`/`double precision` กับเงิน แม้จะดู "ง่าย" กว่า `numeric`
3. **คิดถึงอนาคต แต่ไม่ over-engineer** — ถ้าไม่แน่ใจว่าตัวเลขจะโตเกิน 2 พันล้านหรือไม่ ให้พิจารณาความเป็นไปได้จริงของธุรกิจ ไม่ใช่ใช้ `bigint` แบบไม่คิดในทุกกรณี

ในบทถัดไปเราจะเจาะลึกแต่ละ type ทีละตัว เริ่มจาก integer

---

## Step 52: Integer Types — smallint, integer, bigint

### ตาราง Integer Types

PostgreSQL มี integer type แบบ exact (ไม่มีเศษ ไม่มีความคลาดเคลื่อน) อยู่ 3 ขนาด:

| Type | ขนาด (Storage) | ช่วงค่า (Range) | ชื่อเรียกย่อ |
|---|---|---|---|
| `smallint` | 2 bytes | -32,768 ถึง 32,767 | `int2` |
| `integer` | 4 bytes | -2,147,483,648 ถึง 2,147,483,647 | `int4`, `int` |
| `bigint` | 8 bytes | -9,223,372,036,854,775,808 ถึง 9,223,372,036,854,775,807 | `int8` |

### ตัวอย่างการใช้งาน

```sql
CREATE TABLE product_stock (
    product_id      integer,
    quantity        smallint,
    warehouse_code  smallint,
    total_sold_ever bigint
);

INSERT INTO product_stock (product_id, quantity, warehouse_code, total_sold_ever)
VALUES
    (1001, 250, 3, 15000000),
    (1002, 32000, 1, 999999999999);

SELECT * FROM product_stock;
```

ผลลัพธ์:

```
 product_id | quantity | warehouse_code | total_sold_ever
------------+----------+----------------+------------------
       1001 |      250 |              3 |         15000000
       1002 |    32000 |              1 |    999999999999
(2 rows)
```

### ทดสอบขอบเขต (Overflow)

ถ้าพยายามใส่ค่าที่เกินช่วงของ type นั้นๆ PostgreSQL จะปฏิเสธทันที:

```sql
-- smallint รับค่าได้สูงสุด 32767
INSERT INTO product_stock (product_id, quantity, warehouse_code, total_sold_ever)
VALUES (1003, 40000, 1, 100);
```

ผลลัพธ์:

```
ERROR:  smallint out of range
```

```sql
-- integer รับค่าได้สูงสุด 2147483647
SELECT 2147483647::integer + 1;
```

ผลลัพธ์:

```
ERROR:  integer out of range
```

สังเกตว่า PostgreSQL ตรวจสอบ overflow อย่างเข้มงวด ไม่ยอมให้ค่า "วน" กลับไปเป็นค่าติดลบแบบเงียบๆ เหมือนบางภาษาโปรแกรมมิ่งระดับต่ำ

### เมื่อไหร่ควรใช้ type ไหน

**ใช้ `smallint` เมื่อ:**
- ค่าที่เก็บมีช่วงจำกัดแน่ชัด เช่น อายุคน (0-150), จำนวนดาวรีวิว (1-5), รหัสสถานะแบบตัวเลข, ปีเกิดในรูปแบบสั้น
- ตัวอย่าง: `rating smallint`, `age smallint`, `priority smallint`

**ใช้ `integer` เมื่อ:**
- เป็นตัวเลือกเริ่มต้น (default choice) สำหรับตัวเลขจำนวนเต็มส่วนใหญ่ เช่น จำนวนสินค้าในสต๊อก, จำนวนคำสั่งซื้อ, primary key ของตารางขนาดกลาง (ไม่เกิน 2 พันล้านแถว)
- ตัวอย่าง: `quantity integer`, `order_id integer`, `view_count integer`

**ใช้ `bigint` เมื่อ:**
- ค่าที่อาจมีจำนวนมหาศาล เช่น primary key ของตารางที่คาดว่าจะมีข้อมูลเกิน 2 พันล้านแถว (เช่น log table, event table, transaction table ของระบบใหญ่), ตัวเลขที่เกี่ยวกับเงินในหน่วยสตางค์ (satang) ของระบบธุรกรรมปริมาณสูง, timestamp ในรูปแบบ Unix epoch แบบ millisecond
- ตัวอย่าง: `event_id bigint`, `total_transactions bigint`

### เปรียบเทียบ Performance และ Storage

```sql
-- เปรียบเทียบขนาด storage ของแต่ละ type
SELECT
    pg_column_size(1::smallint) AS smallint_size,
    pg_column_size(1::integer)  AS integer_size,
    pg_column_size(1::bigint)   AS bigint_size;
```

ผลลัพธ์:

```
 smallint_size | integer_size | bigint_size
---------------+--------------+-------------
             2 |            4 |            8
(1 row)
```

ตัวเลข 2/4/8 ไบต์นี้คูณเข้ากับจำนวนแถวจริง เช่น ตารางที่มี 100 ล้านแถว การเลือก `bigint` แทน `integer` สำหรับคอลัมน์เดียวจะเพิ่มพื้นที่ประมาณ 400 MB โดยไม่มีเหตุผล (ยังไม่นับ index ที่ทับซ้อนอีก)

> **หมายเหตุเรื่อง Alignment:** ใน PostgreSQL การจัดเรียงคอลัมน์ในตารางมีผลต่อ padding (memory alignment) ด้วย เช่นถ้าวาง `smallint` สลับกับ `bigint` แบบไม่เป็นระเบียบ อาจเกิด padding เปล่าๆ แทรกอยู่ เรื่องนี้จะอธิบายละเอียดใน Part ที่ว่าด้วยการออกแบบตารางขั้นสูง

---

## Step 53: Serial Types เบื้องต้น — smallserial, serial, bigserial

### Serial คืออะไร

`serial`, `smallserial`, และ `bigserial` **ไม่ใช่ data type จริงๆ** แต่เป็น "syntax sugar" หรือทางลัดที่ PostgreSQL จัดเตรียมไว้เพื่อสร้างคอลัมน์ auto-increment ได้ง่ายๆ เบื้องหลังมันคือการสร้าง **sequence** object แล้วผูกเข้ากับคอลัมน์นั้นโดยอัตโนมัติ

| Serial Type | Underlying Type | ช่วงค่า |
|---|---|---|
| `smallserial` | `smallint` | 1 ถึง 32,767 |
| `serial` | `integer` | 1 ถึง 2,147,483,647 |
| `bigserial` | `bigint` | 1 ถึง 9,223,372,036,854,775,807 |

### ตัวอย่างการใช้งาน

```sql
CREATE TABLE customers (
    customer_id  serial PRIMARY KEY,
    full_name    varchar(100)
);

INSERT INTO customers (full_name) VALUES ('สมชาย ใจดี');
INSERT INTO customers (full_name) VALUES ('สมหญิง รักเรียน');
INSERT INTO customers (full_name) VALUES ('วิชัย มั่นคง');

SELECT * FROM customers;
```

ผลลัพธ์:

```
 customer_id |    full_name
-------------+------------------
           1 | สมชาย ใจดี
           2 | สมหญิง รักเรียน
           3 | วิชัย มั่นคง
(3 rows)
```

สังเกตว่าเราไม่ต้องระบุค่า `customer_id` เองเลย PostgreSQL สร้างค่าให้อัตโนมัติโดยเรียงลำดับขึ้นไปเรื่อยๆ

### เบื้องหลังของ serial

เมื่อสร้างคอลัมน์ `serial` PostgreSQL จะทำสิ่งเหล่านี้โดยอัตโนมัติ:

1. สร้าง sequence object ชื่อ `customers_customer_id_seq`
2. ตั้งค่า default ของคอลัมน์เป็น `nextval('customers_customer_id_seq')`
3. ตั้งค่า type ของคอลัมน์เป็น `integer` (หรือ `smallint`/`bigint` ตามชนิด serial)
4. ผูก "ownership" ของ sequence เข้ากับคอลัมน์ (เมื่อ drop ตาราง sequence จะถูกลบตามไปด้วย)

เราสามารถตรวจสอบได้ด้วย `\d`:

```sql
\d customers
```

ผลลัพธ์ (ตัวอย่าง):

```
                                     Table "public.customers"
   Column    |         Type          | Collation | Nullable |                Default
-------------+------------------------+-----------+----------+----------------------------------------
 customer_id | integer                |           | not null | nextval('customers_customer_id_seq'::regclass)
 full_name   | character varying(100) |           |          |
Indexes:
    "customers_pkey" PRIMARY KEY, btree (customer_id)
```

จะเห็นว่า type จริงๆ คือ `integer` ไม่ใช่ `serial` — `serial` เป็นเพียง "คำสั่งลัด" ตอน `CREATE TABLE` เท่านั้น

### ทดสอบพฤติกรรมของ sequence

```sql
-- ดูค่าถัดไปที่จะถูกใช้
SELECT currval(pg_get_serial_sequence('customers', 'customer_id'));
```

ผลลัพธ์:

```
 currval
---------
       3
```

ข้อควรระวังที่สำคัญ: sequence **ไม่ถูก rollback** แม้ transaction ที่ insert จะถูก rollback ก็ตาม ทำให้เกิดช่องว่าง (gap) ของเลขได้เป็นเรื่องปกติ:

```sql
BEGIN;
INSERT INTO customers (full_name) VALUES ('ทดสอบ Rollback');
ROLLBACK;

INSERT INTO customers (full_name) VALUES ('คนถัดไป');

SELECT * FROM customers ORDER BY customer_id;
```

ผลลัพธ์:

```
 customer_id |    full_name
-------------+------------------
           1 | สมชาย ใจดี
           2 | สมหญิง รักเรียน
           3 | วิชัย มั่นคง
           5 | คนถัดไป
(4 rows)
```

สังเกตว่า `customer_id = 4` หายไป เพราะถูกใช้ไปแล้วตอน rollback แต่ sequence ไม่คืนค่ากลับ **นี่คือพฤติกรรมปกติและถูกต้อง** ห้ามพยายาม "แก้" ให้เลขต่อเนื่องกันเป๊ะๆ เพราะจะทำให้เกิดปัญหา concurrency ร้ายแรงกว่าเดิม

> **หมายเหตุสำคัญ:** ใน PostgreSQL เวอร์ชันปัจจุบัน (10 ขึ้นไป) มีทางเลือกใหม่ที่แนะนำมากกว่า `serial` คือ `GENERATED AS IDENTITY` ซึ่งเป็นมาตรฐาน SQL และปลอดภัยกว่าในหลายแง่มุม (เช่น ป้องกันการเขียนทับค่าคอลัมน์โดยไม่ตั้งใจ) เราจะเจาะลึกเรื่องนี้อย่างละเอียดใน **Part 039** สำหรับตอนนี้ให้รู้จัก `serial` ไว้ก่อนเพราะยังพบเห็นได้ทั่วไปในโค้ดที่มีอยู่แล้ว (legacy code) จำนวนมาก

### เมื่อไหร่ใช้ serial ไหน

| ใช้กับ | เหตุผล |
|---|---|
| `smallserial` | ตารางเล็กมากที่รู้แน่ชัดว่าจะไม่เกิน 32,767 แถว เช่น ตาราง lookup ประเภทสินค้า |
| `serial` | ตัวเลือกเริ่มต้นสำหรับ primary key ส่วนใหญ่ |
| `bigserial` | ตารางที่คาดว่าจะมีข้อมูลจำนวนมาก เช่น ตาราง log, ตาราง order_items ของระบบ e-commerce ขนาดใหญ่ |

---

## Step 54: Decimal/Numeric Types — numeric, real, double precision

### สองตระกูลของตัวเลขทศนิยม

PostgreSQL แบ่งตัวเลขที่มีจุดทศนิยมออกเป็น 2 ตระกูลใหญ่ที่ **ทำงานต่างกันโดยพื้นฐาน**:

1. **Exact numeric (แม่นยำ 100%)** — `numeric` (มีชื่อเรียกอีกอย่างว่า `decimal`, ทั้งสองคำนี้เหมือนกันทุกประการใน PostgreSQL)
2. **Approximate/Floating point (มีความคลาดเคลื่อน)** — `real` (4 bytes, single precision) และ `double precision` (8 bytes, double precision)

### ตาราง Decimal/Numeric Types

| Type | ขนาด | ความแม่นยำ | ช่วงค่า | ลักษณะ |
|---|---|---|---|---|
| `numeric(p,s)` / `decimal(p,s)` | แปรผัน (variable) | แม่นยำเป๊ะ (exact) | สูงสุด 131,072 หลักก่อนจุด, 16,383 หลักหลังจุด | ช้ากว่าเล็กน้อย แต่แม่นยำเสมอ |
| `real` | 4 bytes | ~6 หลักทศนิยม | ประมาณ 1E-37 ถึง 1E+37 | เร็ว แต่มีความคลาดเคลื่อน |
| `double precision` | 8 bytes | ~15 หลักทศนิยม | ประมาณ 1E-307 ถึง 1E+308 | เร็ว แต่มีความคลาดเคลื่อน |

### numeric(precision, scale) คืออะไร

เมื่อประกาศ `numeric(p, s)`:
- `p` (precision) = จำนวนหลักทั้งหมดที่เก็บได้ (ทั้งก่อนและหลังจุดทศนิยม)
- `s` (scale) = จำนวนหลักหลังจุดทศนิยม

```sql
CREATE TABLE product_price (
    product_name  text,
    price         numeric(10, 2)   -- 10 หลักทั้งหมด, 2 หลักหลังจุด
);

INSERT INTO product_price VALUES
    ('เสื้อยืด', 259.50),
    ('กางเกงยีนส์', 890.00),
    ('รองเท้าผ้าใบ', 1250.75);

SELECT * FROM product_price;
```

ผลลัพธ์:

```
 product_name  |  price
----------------+---------
 เสื้อยืด       |  259.50
 กางเกงยีนส์    |  890.00
 รองเท้าผ้าใบ   | 1250.75
(3 rows)
```

### ทดสอบขอบเขตของ numeric(p,s)

```sql
-- price เป็น numeric(10,2) จึงมีตัวเลขก่อนจุดได้สูงสุด 8 หลัก (10 - 2)
INSERT INTO product_price VALUES ('สินค้าราคาแพงมาก', 123456789.99);
```

ผลลัพธ์:

```
ERROR:  numeric field overflow
DETAIL:  A field with precision 10, scale 2 must round to an absolute value less than 10^8.
```

ถ้าใส่ทศนิยมเกินที่กำหนด PostgreSQL จะ **ปัดเศษให้อัตโนมัติ** (ไม่ error):

```sql
INSERT INTO product_price VALUES ('ทดสอบปัดเศษ', 99.999);
SELECT * FROM product_price WHERE product_name = 'ทดสอบปัดเศษ';
```

ผลลัพธ์:

```
 product_name  | price
----------------+--------
 ทดสอบปัดเศษ    | 100.00
(1 row)
```

`99.999` ถูกปัดเป็น `100.00` เพราะ scale กำหนดไว้ที่ 2 ตำแหน่ง (ปัดตามกฎ round-half-up)

### numeric แบบไม่กำหนด precision/scale

ถ้าประกาศ `numeric` เฉยๆ โดยไม่ระบุ `(p,s)` จะสามารถเก็บตัวเลขที่มีความแม่นยำสูงมากและขนาดผันแปรได้ตามต้องการ (จำกัดด้วย storage เท่านั้น) เหมาะสำหรับงานที่ต้องการความแม่นยำระดับวิทยาศาสตร์หรือการเงินที่ไม่ทราบขอบเขตล่วงหน้า

```sql
SELECT numeric '1' / numeric '3';
```

ผลลัพธ์:

```
              ?column?
-------------------------------------
 0.33333333333333333333333333333333
```

จะเห็นว่า `numeric` ให้ทศนิยมได้ละเอียดมาก (ค่า default คือ 16 หลักหลังจุดเมื่อไม่มีการระบุ scale แต่จริงๆ สามารถคำนวณได้ละเอียดกว่านี้อีกตามการดำเนินการ)

### real และ double precision

```sql
CREATE TABLE scientific_data (
    measurement_real    real,
    measurement_double  double precision
);

INSERT INTO scientific_data VALUES (3.14159265358979, 3.14159265358979);

SELECT * FROM scientific_data;
```

ผลลัพธ์:

```
 measurement_real | measurement_double
-------------------+---------------------
           3.14159 |    3.14159265358979
(1 row)
```

สังเกตว่า `real` เก็บได้แม่นยำแค่ประมาณ 6 หลัก ในขณะที่ `double precision` เก็บได้ละเอียดกว่ามาก (ประมาณ 15 หลัก) — นี่คือธรรมชาติของ floating point ซึ่งเราจะเจาะลึกใน Step ถัดไป

### เมื่อไหร่ควรใช้ type ไหน

**ใช้ `numeric`/`decimal` เมื่อ:**
- เกี่ยวข้องกับเงิน (ราคาสินค้า, ยอดเงินในบัญชี, ภาษี, ค่าคอมมิชชัน)
- ต้องการความแม่นยำสัมบูรณ์ ไม่ยอมรับความคลาดเคลื่อนแม้แต่น้อย
- ค่าที่ใช้ในการคำนวณทางบัญชีหรือกฎหมายที่ต้องตรงเป๊ะ

**ใช้ `real`/`double precision` เมื่อ:**
- งานวิทยาศาสตร์, งาน machine learning, พิกัด GPS, ค่าที่วัดจากเซนเซอร์
- ความเร็วในการคำนวณสำคัญกว่าความแม่นยำสัมบูรณ์ (เช่น การคำนวณทางสถิติจำนวนมาก)
- ยอมรับความคลาดเคลื่อนเล็กน้อยได้ (rounding error) โดยไม่กระทบผลลัพธ์ทางธุรกิจ

> **กฎทองที่ต้องจำ:** "ถ้าเกี่ยวกับเงิน ให้ใช้ `numeric` เสมอ ไม่มีข้อยกเว้น" เราจะพิสูจน์ให้เห็นว่าทำไมใน Step ถัดไป

---

## Step 55: ปัญหา Floating Point Precision และตัวอย่างที่แสดงให้เห็นจริง

### ทำไม Floating Point ถึงไม่แม่นยำ

คอมพิวเตอร์เก็บตัวเลข `real` และ `double precision` ในรูปแบบ **binary floating point** (ตามมาตรฐาน IEEE 754) ซึ่งไม่สามารถแทนค่าทศนิยมบางค่าได้อย่างแม่นยำ 100% เหมือนกับที่เลขฐาน 10 ไม่สามารถแทนค่า 1/3 ได้แม่นยำ (0.333...) เลขฐาน 2 ก็ไม่สามารถแทนค่าบางอย่าง เช่น 0.1 ได้แม่นยำเช่นกัน

### ตัวอย่างปัญหาที่จับต้องได้จริง

```sql
-- ทดสอบด้วย double precision
SELECT 0.1::double precision + 0.2::double precision AS result;
```

ผลลัพธ์:

```
       result
--------------------
 0.30000000000000004
```

**ในทางคณิตศาสตร์ 0.1 + 0.2 ต้องเท่ากับ 0.3 พอดี** แต่ผลลัพธ์ที่ได้คือ `0.30000000000000004` เพราะ 0.1 และ 0.2 ไม่สามารถแทนค่าได้แม่นยำในระบบ binary floating point

เปรียบเทียบกับ `numeric`:

```sql
SELECT 0.1::numeric + 0.2::numeric AS result;
```

ผลลัพธ์:

```
 result
--------
    0.3
```

`numeric` ให้ผลลัพธ์ที่ถูกต้องแม่นยำ 100% เพราะมันเก็บค่าในรูปแบบทศนิยมฐาน 10 จริงๆ ไม่ใช่ binary approximation

### ตัวอย่างที่ร้ายแรงกว่า: การเปรียบเทียบค่า

```sql
SELECT (0.1::double precision + 0.2::double precision) = 0.3::double precision AS is_equal;
```

ผลลัพธ์:

```
 is_equal
----------
 f
```

**นี่คือหายนะในระบบจริง** ถ้าเราเขียนโค้ดตรวจสอบว่า `total = 0.3` เพื่อยืนยันความถูกต้องของยอดเงิน ระบบจะบอกว่า "ไม่เท่ากัน" ทั้งที่ในทางคณิตศาสตร์มันเท่ากัน

### ตัวอย่างสถานการณ์จริง: ระบบตะกร้าสินค้าใช้ floating point

```sql
-- จำลองระบบที่ใช้ real ผิดพลาด (ห้ามทำแบบนี้ในระบบจริง!)
CREATE TABLE bad_cart_example (
    item_name  text,
    unit_price real,
    quantity   integer
);

INSERT INTO bad_cart_example VALUES
    ('สินค้า A', 19.99, 3),
    ('สินค้า B', 0.10, 1),
    ('สินค้า C', 0.20, 1);

SELECT
    SUM(unit_price * quantity) AS total_using_real
FROM bad_cart_example;
```

ผลลัพธ์ (อาจแสดงค่าคลาดเคลื่อนเล็กน้อยขึ้นอยู่กับการคำนวณสะสม):

```
  total_using_real
--------------------
   60.27000045776367
```

เปรียบเทียบกับการใช้ `numeric`:

```sql
CREATE TABLE good_cart_example (
    item_name  text,
    unit_price numeric(10,2),
    quantity   integer
);

INSERT INTO good_cart_example VALUES
    ('สินค้า A', 19.99, 3),
    ('สินค้า B', 0.10, 1),
    ('สินค้า C', 0.20, 1);

SELECT
    SUM(unit_price * quantity) AS total_using_numeric
FROM good_cart_example;
```

ผลลัพธ์:

```
 total_using_numeric
----------------------
                60.27
```

`numeric` ให้ผลลัพธ์ที่ถูกต้องแม่นยำเป๊ะๆ ในขณะที่ `real` สะสมความคลาดเคลื่อนจนเห็นได้ชัดในทศนิยมตำแหน่งท้ายๆ

### สรุปสาเหตุของปัญหา

| ประเด็น | Floating Point (`real`/`double precision`) | Exact (`numeric`) |
|---|---|---|
| วิธีเก็บค่า | Binary approximation (IEEE 754) | Decimal digits จริงๆ |
| ความแม่นยำ | มีขีดจำกัด และสะสมความคลาดเคลื่อนเมื่อคำนวณต่อเนื่อง | แม่นยำ 100% เสมอ |
| ความเร็ว | เร็วกว่า (CPU มี hardware รองรับโดยตรง) | ช้ากว่าเล็กน้อย (คำนวณแบบ software) |
| เหมาะกับ | งานวิทยาศาสตร์, สถิติ, ML | เงิน, บัญชี, ธุรกรรมทุกชนิด |

> **ข้อสรุปสำคัญ:** อย่าใช้ `real` หรือ `double precision` กับข้อมูลที่เกี่ยวข้องกับเงินหรือข้อมูลที่ต้องการความแม่นยำเป๊ะๆ เด็ดขาด แม้จะดูเหมือนตัวเลขคลาดเคลื่อนน้อยมากจนไม่น่าสำคัญ แต่เมื่อสะสมนับล้านธุรกรรม ความคลาดเคลื่อนนี้จะกลายเป็นปัญหาใหญ่ที่ตรวจสอบได้ยากมาก (และอาจนำไปสู่ปัญหาทางกฎหมายด้านบัญชีได้)

---

## Step 56: Money Type และทำไมส่วนใหญ่แนะนำใช้ numeric แทน

### Money Type คืออะไร

PostgreSQL มี type พิเศษชื่อ `money` ที่ออกแบบมาสำหรับเก็บค่าเงินโดยเฉพาะ ใช้พื้นที่ 8 bytes และมีความแม่นยำแบบ exact (ไม่ใช่ floating point)

```sql
CREATE TABLE salary_demo (
    employee_name  text,
    salary         money
);

INSERT INTO salary_demo VALUES
    ('พนักงาน A', 25000.50),
    ('พนักงาน B', 45999.99);

SELECT * FROM salary_demo;
```

ผลลัพธ์:

```
 employee_name |  salary
----------------+-----------
 พนักงาน A      | ฿25,000.50
 พนักงาน B      | ฿45,999.99
(2 rows)
```

*(หมายเหตุ: สัญลักษณ์สกุลเงินที่แสดงขึ้นอยู่กับการตั้งค่า `lc_monetary` ของ locale — ในตัวอย่างนี้แสดงเป็นบาทเพื่อความเข้าใจ ในความเป็นจริงอาจแสดงเป็น `$` ตาม default locale)*

### ทำไม money ดูน่าใช้ในตอนแรก

`money` มีข้อดีที่ดูเผินๆ น่าสนใจ:
- แสดงผลพร้อม currency symbol อัตโนมัติ (เช่น `$`, `฿`)
- ใช้พื้นที่คงที่ 8 bytes (เท่ากับ `bigint`)
- คำนวณได้แม่นยำแบบ exact ไม่มีปัญหา floating point

### แต่ทำไมส่วนใหญ่ไม่แนะนำให้ใช้ money

**1. ผูกติดกับ locale ของ database session**

```sql
SHOW lc_monetary;
```

ผลลัพธ์ (ตัวอย่าง):

```
 lc_monetary
--------------
 en_US.UTF-8
```

ค่า `money` จะถูก format ตาม `lc_monetary` ของ session ที่ query ณ ขณะนั้น ถ้า server หรือ client เปลี่ยน locale การแสดงผลและแม้แต่การตีความ input ก็เปลี่ยนตาม ซึ่งเป็นความเสี่ยงมากในระบบที่ต้องรองรับหลายประเทศ หรือแม้แต่ระบบเดียวที่ config เปลี่ยนโดยไม่ตั้งใจ

**2. ไม่รองรับหลายสกุลเงินในตารางเดียวกัน**

`money` ไม่มีแนวคิดเรื่อง currency code (เช่น THB, USD, EUR) ติดมาด้วย มันเป็นแค่ตัวเลขที่ format ตาม locale ปัจจุบันเท่านั้น ถ้าต้องการระบบที่รองรับหลายสกุลเงิน จำเป็นต้องมีคอลัมน์ currency code แยกต่างหากอยู่ดี ซึ่งถ้าต้องแยกอยู่แล้ว การใช้ `numeric` ร่วมกับคอลัมน์ currency จะยืดหยุ่นและชัดเจนกว่า

**3. Precision ถูกกำหนดตายตัวโดย locale ไม่ใช่โดยผู้ออกแบบตาราง**

`money` มี fractional digits ตายตัวตาม locale (ปกติ 2 ตำแหน่ง) ไม่สามารถปรับแต่งเหมือน `numeric(p,s)` ได้ และการหารตัวเลข `money` ด้วยจำนวนเต็มอาจให้ผลลัพธ์ที่ปัดเศษไปแล้วโดยไม่รู้ตัว

```sql
SELECT '10.00'::money / 3;
```

ผลลัพธ์:

```
 ?column?
----------
    $3.33
```

ผลลัพธ์ถูกปัดเศษทันทีโดยไม่มีทางเก็บเศษทศนิยมที่ละเอียดกว่านี้ ในขณะที่ `numeric` สามารถเก็บผลลัพธ์ที่ละเอียดกว่าได้ตามต้องการ:

```sql
SELECT 10.00::numeric / 3;
```

ผลลัพธ์:

```
        ?column?
-------------------------
 3.33333333333333333333
```

**4. การแปลงชนิดข้อมูล (casting) มีพฤติกรรมที่ไม่ตรงตามสัญชาตญาณ**

```sql
-- แปลง text ที่มี comma เป็น money ได้ (ตาม locale)
SELECT '1,234.56'::money;
```

ผลลัพธ์:

```
   money
-----------
 $1,234.56
```

พฤติกรรมนี้ทำให้เกิดความสับสนได้ง่าย เพราะการตีความ comma/period ขึ้นอยู่กับ locale ซึ่งอาจแตกต่างกันระหว่าง environment เช่น dev, staging, production

**5. ไม่มี arbitrary precision เหมือน numeric**

`money` ใช้ fixed-point แบบ 2 ตำแหน่งทศนิยมเท่านั้น (โดยพื้นฐาน) ในขณะที่งานบัญชีบางประเภทต้องการความละเอียดมากกว่านั้น เช่น ราคาต่อหน่วยที่มีทศนิยม 4-6 ตำแหน่ง (พบได้บ่อยในระบบ forex หรือ cryptocurrency)

### คำแนะนำที่เป็นมาตรฐานอุตสาหกรรม

```sql
-- แนวทางที่แนะนำ: ใช้ numeric พร้อมกำหนด precision/scale ชัดเจน
-- และแยกคอลัมน์ currency ออกต่างหาก
CREATE TABLE invoice_recommended (
    invoice_id     bigserial PRIMARY KEY,
    amount         numeric(19, 4),   -- 4 ตำแหน่งทศนิยมเผื่อความละเอียดสูง
    currency_code  char(3) DEFAULT 'THB'  -- ISO 4217 currency code
);

INSERT INTO invoice_recommended (amount, currency_code)
VALUES (1234.5678, 'THB');

SELECT * FROM invoice_recommended;
```

ผลลัพธ์:

```
 invoice_id |  amount   | currency_code
------------+-----------+---------------
          1 | 1234.5678 | THB
(1 row)
```

รูปแบบนี้ให้ความยืดหยุ่นมากกว่า `money` ในทุกมิติ: กำหนด precision เองได้, รองรับหลายสกุลเงินอย่างชัดเจน, ไม่ผูกติดกับ locale ของ session

> **สรุป:** `money` ยังมีประโยชน์ในบางสถานการณ์เฉพาะ (เช่น สคริปต์ง่ายๆ ที่ใช้ในระบบเดียว, locale เดียว, สกุลเงินเดียว ตลอดอายุการใช้งาน) แต่สำหรับระบบธุรกิจจริงที่ต้องการความน่าเชื่อถือระยะยาว **`numeric(p,s)` คือตัวเลือกที่ควรใช้เกือบทุกครั้ง**

---

## Step 57: Character Types — char(n), varchar(n), text

### สามตัวเลือกสำหรับข้อความ

| Type | คำอธิบาย | พฤติกรรม |
|---|---|---|
| `char(n)` | ความยาวคงที่ (fixed-length) | เติม space ต่อท้ายจนครบ n ตัวอักษรเสมอ |
| `varchar(n)` | ความยาวแปรผันแต่จำกัดสูงสุด | เก็บตามความยาวจริง แต่ error ถ้าเกิน n |
| `text` | ความยาวแปรผันไม่จำกัด | เก็บตามความยาวจริง ไม่มีขีดจำกัด (นอกจากขีดจำกัดของระบบ) |

### ตัวอย่างการใช้งานพื้นฐาน

```sql
CREATE TABLE character_demo (
    code_char     char(5),
    code_varchar  varchar(5),
    description   text
);

INSERT INTO character_demo VALUES ('AB', 'AB', 'AB');

SELECT
    code_char,
    length(code_char)    AS char_length,
    code_varchar,
    length(code_varchar) AS varchar_length,
    description,
    length(description)  AS text_length
FROM character_demo;
```

ผลลัพธ์:

```
 code_char | char_length | code_varchar | varchar_length | description | text_length
-----------+-------------+--------------+-----------------+--------------+-------------
 AB        |           2 | AB           |               2 | AB           |           2
```

**ข้อสังเกตสำคัญ:** แม้ `length()` จะรายงานว่า `code_char` มีความยาว 2 (ไม่นับ trailing space ที่เติมอัตโนมัติ) แต่ในความเป็นจริง PostgreSQL เก็บค่าจริงในดิสก์เป็น `'AB   '` (เติม space จนครบ 5 ตัวอักษร) ฟังก์ชัน `length()` จะตัด trailing space ออกให้อัตโนมัติเมื่อแสดงผล

### พิสูจน์ให้เห็นว่า char(n) เติม space จริง

```sql
SELECT
    code_char || '|' AS char_with_marker,
    code_varchar || '|' AS varchar_with_marker
FROM character_demo;
```

ผลลัพธ์:

```
 char_with_marker | varchar_with_marker
-------------------+----------------------
 AB   |            | AB|
(1 row)
```

จะเห็นชัดเจนว่า `char(5)` เติม space 3 ตัวต่อท้าย `AB` จนครบ 5 ตัวอักษร ก่อนจะต่อกับเครื่องหมาย `|` ในขณะที่ `varchar(5)` ไม่เติมอะไรเลย เก็บตามความยาวจริง

### ทดสอบขอบเขตความยาว

```sql
-- varchar(5) รับได้สูงสุด 5 ตัวอักษร
INSERT INTO character_demo (code_varchar) VALUES ('ABCDEF');
```

ผลลัพธ์:

```
ERROR:  value too long for type character varying(5)
```

```sql
-- char(5) ก็เช่นกัน รับได้สูงสุด 5 ตัวอักษร
INSERT INTO character_demo (code_char) VALUES ('ABCDEF');
```

ผลลัพธ์:

```
ERROR:  value too long for type character(5)
```

ทั้งสอง type จะปฏิเสธข้อมูลที่ยาวเกินกำหนดเหมือนกัน ต่างจาก `text` ที่ไม่มีขีดจำกัดความยาวเลย (นอกจากขีดจำกัดสูงสุดของ PostgreSQL เอง ซึ่งอยู่ที่ประมาณ 1 GB ต่อค่า)

### ความเข้าใจผิดที่พบบ่อย: "varchar เร็วกว่า text" หรือ "char เร็วกว่า varchar"

นี่คือ **ความเชื่อผิดๆ ที่พบบ่อยมาก** โดยเฉพาะคนที่เคยใช้ MySQL มาก่อน ในความเป็นจริง เอกสารทางการของ PostgreSQL ระบุไว้ชัดเจนว่า:

> "There is no performance difference among these three types, apart from increased storage space when using the blank-padded type (`char(n)`), and a few extra CPU cycles to check the length when storing into a length-constrained column. While `character(n)` has performance advantages in some other database systems, there is no such advantage in PostgreSQL; in fact `character(n)` is usually the slowest of the three because of its additional storage costs."

สรุปเป็นภาษาไทย: **`char(n)` มักจะช้าที่สุดในสามตัวเลือกเนื่องจากต้องเก็บ space เพิ่มและมีค่าใช้จ่ายในการเติม/ตัด space** ส่วน `varchar(n)` และ `text` แทบไม่มีความแตกต่างด้าน performance เลย เพราะเบื้องหลังทั้งคู่ใช้กลไกการเก็บข้อมูลแบบเดียวกัน (variable-length with TOAST support)

### ทดสอบเปรียบเทียบพื้นที่จัดเก็บ

```sql
SELECT
    pg_column_size('Hello'::char(20))    AS char20_size,
    pg_column_size('Hello'::varchar(20)) AS varchar20_size,
    pg_column_size('Hello'::text)        AS text_size;
```

ผลลัพธ์:

```
 char20_size | varchar20_size | text_size
-------------+-----------------+-----------
          24 |               6 |         6
```

`char(20)` ใช้พื้นที่มากกว่าอย่างชัดเจน เพราะต้องเติม space จนครบ 20 ตัวอักษร (แม้ข้อความจริงจะสั้นกว่านั้นมาก) ในขณะที่ `varchar(20)` และ `text` เก็บเฉพาะข้อมูลจริงบวก overhead เล็กน้อยเท่านั้น

### คำแนะนำการเลือกใช้ (Best Practice)

คำแนะนำอย่างเป็นทางการจากชุมชน PostgreSQL และผู้เชี่ยวชาญส่วนใหญ่คือ:

**ใช้ `text` เป็นค่าเริ่มต้นสำหรับข้อความเกือบทุกกรณี** เพราะ:
- ไม่มีข้อเสียด้าน performance เมื่อเทียบกับ `varchar(n)`
- ยืดหยุ่นกว่า ไม่ต้องกังวลเรื่องแก้ schema ภายหลังเมื่อความยาวข้อมูลเปลี่ยนไป
- ถ้าต้องการจำกัดความยาว ให้ใช้ `CHECK constraint` แทน ซึ่งยืดหยุ่นกว่าและแก้ไขทีหลังได้ง่ายกว่า (ไม่ต้อง `ALTER TABLE ... ALTER COLUMN TYPE`)

```sql
-- แนวทางที่แนะนำ: ใช้ text + CHECK constraint แทน varchar(n)
CREATE TABLE users_recommended (
    username  text NOT NULL CHECK (length(username) BETWEEN 3 AND 30),
    bio       text
);
```

**ใช้ `varchar(n)` เมื่อ:**
- ต้องการให้ schema สื่อความหมายชัดเจนในตัวเอง (self-documenting) โดยไม่ต้องอ่าน constraint แยก
- ทำงานร่วมกับเครื่องมือหรือ ORM ที่คาดหวัง `varchar(n)` แบบมาตรฐาน SQL
- มีข้อกำหนดทางธุรกิจที่ชัดเจนตายตัว เช่น รหัสไปรษณีย์ไทย 5 หลัก, เลขบัตรประชาชน 13 หลัก

```sql
CREATE TABLE address_demo (
    postal_code  varchar(5),
    national_id  varchar(13)
);
```

**ใช้ `char(n)` เมื่อ:**
- แทบไม่มีความจำเป็นในทางปฏิบัติสมัยใหม่ กรณีที่ยังพอมีเหตุผลคือ รหัสที่มีความยาวคงที่เป๊ะเสมอและไม่มีวันเปลี่ยน เช่น รหัส ISO ของประเทศ (`'TH'`, `'US'` — แต่ก็ยังแนะนำ `char(2)` หรือ `text` ก็ได้) หรือ checksum แบบความยาวคงที่
- ในทางปฏิบัติ ผู้เชี่ยวชาญส่วนใหญ่แนะนำให้ **หลีกเลี่ยง `char(n)` เว้นแต่จะมีเหตุผลเฉพาะเจาะจงจริงๆ**

### สรุปเปรียบเทียบ

| คุณสมบัติ | `char(n)` | `varchar(n)` | `text` |
|---|---|---|---|
| ความยาว | คงที่ (เติม space) | แปรผัน จำกัดสูงสุด | แปรผัน ไม่จำกัด |
| Performance | ช้าที่สุด (เพราะ overhead ของ space) | เท่ากับ text | เร็วที่สุด (เท่ากับ varchar) |
| Storage เมื่อข้อมูลสั้นกว่า n | เปลืองพื้นที่ (padding) | ใช้ตามจริง | ใช้ตามจริง |
| คำแนะนำ | หลีกเลี่ยง เว้นแต่จำเป็นจริงๆ | ใช้เมื่อต้องการ constraint ความยาวในตัว schema | ตัวเลือกเริ่มต้นที่แนะนำ |

---

## Step 58: Boolean Type — ค่าที่ยอมรับและพฤติกรรม NULL

### Boolean คืออะไร

`boolean` (หรือย่อว่า `bool`) เก็บค่าได้ 3 สถานะ: `true`, `false`, และ `NULL` (unknown) ใช้พื้นที่เพียง 1 byte

```sql
CREATE TABLE task_demo (
    task_name    text,
    is_completed boolean
);

INSERT INTO task_demo VALUES
    ('ทำการบ้าน', true),
    ('อ่านหนังสือ', false),
    ('ซักผ้า', NULL);

SELECT * FROM task_demo;
```

ผลลัพธ์:

```
  task_name   | is_completed
--------------+--------------
 ทำการบ้าน     | t
 อ่านหนังสือ   | f
 ซักผ้า        |
(3 rows)
```

สังเกตว่า `NULL` แสดงเป็นช่องว่าง (ไม่ใช่ `false`) เพราะ `NULL` หมายถึง "ไม่ทราบค่า" ไม่ใช่ "false"

### ค่า Input ที่ PostgreSQL ยอมรับสำหรับ boolean

PostgreSQL ยืดหยุ่นมากในการรับ input สำหรับ boolean โดยยอมรับหลายรูปแบบดังนี้:

```sql
CREATE TABLE boolean_input_test (
    label  text,
    value  boolean
);

INSERT INTO boolean_input_test VALUES
    ('true',    true),
    ('TRUE',    'TRUE'),
    ('t',       't'),
    ('T',       'T'),
    ('yes',     'yes'),
    ('y',       'y'),
    ('1',       '1'),
    ('on',      'on'),
    ('false',   false),
    ('FALSE',   'FALSE'),
    ('f',       'f'),
    ('F',       'F'),
    ('no',      'no'),
    ('n',       'n'),
    ('0',       '0'),
    ('off',     'off');

SELECT * FROM boolean_input_test;
```

ผลลัพธ์:

```
 label | value
-------+-------
 true  | t
 TRUE  | t
 t     | t
 T     | t
 yes   | t
 y     | t
 1     | t
 on    | t
 false | f
 FALSE | f
 f     | f
 F     | f
 no    | f
 n     | f
 0     | f
 off   | f
(16 rows)
```

### ตารางสรุปค่าที่ยอมรับ

| กลุ่ม | ค่าที่ยอมรับว่าเป็น `true` | ค่าที่ยอมรับว่าเป็น `false` |
|---|---|---|
| คำเต็ม | `'true'`, `'TRUE'`, `'True'` | `'false'`, `'FALSE'`, `'False'` |
| ตัวย่อ | `'t'`, `'T'`, `'yes'`, `'YES'`, `'y'`, `'Y'` | `'f'`, `'F'`, `'no'`, `'NO'`, `'n'`, `'N'` |
| ตัวเลข | `'1'` | `'0'` |
| คำอื่นๆ | `'on'` | `'off'` |

*(หมายเหตุ: PostgreSQL จะพยายามจับคู่ตัวหน้าของคำ เช่น `'tr'`, `'y'` ก็ถูกตีความเป็น true ได้ ตราบใดที่ไม่กำกวมกับคำอื่น)*

### ทดสอบค่าที่ไม่ถูกต้อง

```sql
INSERT INTO boolean_input_test VALUES ('maybe', 'maybe');
```

ผลลัพธ์:

```
ERROR:  invalid input syntax for type boolean: "maybe"
```

PostgreSQL จะปฏิเสธค่าที่กำกวมหรือไม่รู้จักทันที ไม่มีการเดาแบบมั่วๆ

### พฤติกรรมของ NULL ใน Boolean Logic (Three-Valued Logic)

นี่คือจุดที่มือใหม่สับสนบ่อยที่สุด — PostgreSQL ใช้ตรรกะแบบ **three-valued logic** (TRUE, FALSE, UNKNOWN/NULL) ไม่ใช่ two-valued logic แบบภาษาโปรแกรมมิ่งทั่วไป

```sql
SELECT
    true AND NULL   AS true_and_null,
    false AND NULL  AS false_and_null,
    true OR NULL    AS true_or_null,
    false OR NULL   AS false_or_null,
    NOT NULL        AS not_null;
```

ผลลัพธ์:

```
 true_and_null | false_and_null | true_or_null | false_or_null | not_null
----------------+------------------+---------------+-----------------+----------
              |  f               | t             |                |
(1 row)
```

อธิบายทีละบรรทัด:

| นิพจน์ | ผลลัพธ์ | เหตุผล |
|---|---|---|
| `true AND NULL` | `NULL` | ถ้าฝั่งซ้าย true แต่ฝั่งขวาไม่รู้ ผลลัพธ์ก็ไม่รู้ |
| `false AND NULL` | `false` | ไม่ว่าฝั่งขวาจะเป็นอะไร `false AND x` ต้องเป็น `false` เสมอ (short-circuit ทางตรรกะ) |
| `true OR NULL` | `true` | ไม่ว่าฝั่งขวาจะเป็นอะไร `true OR x` ต้องเป็น `true` เสมอ |
| `false OR NULL` | `NULL` | ถ้าฝั่งซ้าย false แต่ฝั่งขวาไม่รู้ ผลลัพธ์ก็ไม่รู้ |
| `NOT NULL` | `NULL` | ค่าตรงข้ามของ "ไม่รู้" ก็ยังคงเป็น "ไม่รู้" |

### ผลกระทบของ NULL ใน WHERE clause

```sql
SELECT * FROM task_demo WHERE is_completed = true;
```

ผลลัพธ์:

```
 task_name  | is_completed
------------+--------------
 ทำการบ้าน   | t
(1 row)
```

```sql
SELECT * FROM task_demo WHERE is_completed = false;
```

ผลลัพธ์:

```
  task_name  | is_completed
-------------+--------------
 อ่านหนังสือ | f
(1 row)
```

สังเกตว่าแถว `'ซักผ้า'` ที่มีค่า `NULL` **ไม่ปรากฏในทั้งสองคำสั่ง** เพราะ `WHERE` clause จะกรองเฉพาะแถวที่เงื่อนไขเป็น `true` เท่านั้น (`NULL = true` ให้ผล `NULL` ซึ่งไม่ใช่ `true` จึงถูกกรองออก)

ถ้าต้องการรวมแถวที่เป็น `NULL` ด้วย ต้องเขียนแยก:

```sql
SELECT * FROM task_demo WHERE is_completed IS NOT TRUE;
```

ผลลัพธ์:

```
  task_name   | is_completed
--------------+--------------
 อ่านหนังสือ  | f
 ซักผ้า       |
(2 rows)
```

`IS NOT TRUE` จะรวมทั้ง `false` และ `NULL` เข้าด้วยกัน ต่างจาก `!= true` ที่จะไม่รวม `NULL`

### คำแนะนำในทางปฏิบัติ

1. ถ้าคอลัมน์ boolean ไม่ควรมีค่า "ไม่ทราบ" ได้เลย ให้กำหนด `NOT NULL` พร้อม `DEFAULT` เพื่อป้องกันปัญหา three-valued logic ที่ไม่คาดคิด

```sql
CREATE TABLE task_recommended (
    task_name    text NOT NULL,
    is_completed boolean NOT NULL DEFAULT false
);
```

2. เมื่อต้องเขียนเงื่อนไขที่เกี่ยวข้องกับ boolean ที่อาจเป็น `NULL` ได้ ให้ใช้ `IS TRUE`, `IS FALSE`, `IS NOT TRUE`, `IS NOT FALSE` แทน `= true`/`= false` เพื่อความชัดเจนและถูกต้อง

---

## Step 59: Type Casting ระหว่างตัวเลขและข้อความ

### สองวิธีในการ Cast

PostgreSQL มี 2 วิธีหลักในการแปลงชนิดข้อมูล:

1. **มาตรฐาน SQL:** `CAST(value AS type)`
2. **PostgreSQL syntax:** `value::type` (สั้นกว่า และใช้กันแพร่หลายในโค้ด PostgreSQL)

ทั้งสองวิธีทำงานเหมือนกันทุกประการ ต่างกันแค่ syntax

```sql
SELECT
    CAST('123' AS integer)  AS cast_syntax,
    '123'::integer          AS colon_syntax;
```

ผลลัพธ์:

```
 cast_syntax | colon_syntax
-------------+---------------
         123 |           123
(1 row)
```

### แปลงข้อความเป็นตัวเลข

```sql
SELECT
    '42'::integer         AS to_int,
    '3.14'::numeric        AS to_numeric,
    '99.5'::real            AS to_real,
    '1000000'::bigint       AS to_bigint;
```

ผลลัพธ์:

```
 to_int | to_numeric | to_real | to_bigint
---------+-------------+----------+------------
     42 |        3.14 |     99.5 |    1000000
(1 row)
```

### แปลงตัวเลขเป็นข้อความ

```sql
SELECT
    123::text          AS int_to_text,
    3.14159::text       AS numeric_to_text,
    true::text          AS bool_to_text;
```

ผลลัพธ์:

```
 int_to_text | numeric_to_text | bool_to_text
-------------+-------------------+---------------
 123         | 3.14159           | true
(1 row)
```

### การแปลงที่ "ปลอดภัย" (Safe Casting)

การแปลงถือว่า "ปลอดภัย" เมื่อ:
- ทุกค่าที่เป็นไปได้ในต้นทางสามารถแปลงเป็นปลายทางได้เสมอ โดยไม่สูญเสียข้อมูลสำคัญ
- ไม่มีความเสี่ยงที่จะ error หรือได้ผลลัพธ์ที่ผิดเพี้ยนจากความตั้งใจ

```sql
-- ปลอดภัย: integer -> bigint (ค่า integer ทุกค่าอยู่ในช่วงของ bigint เสมอ)
SELECT 2147483647::integer::bigint;
```

ผลลัพธ์:

```
    int8
------------
 2147483647
(1 row)
```

```sql
-- ปลอดภัย: integer -> numeric (numeric เก็บทศนิยมได้ ไม่มีทางสูญเสียความแม่นยำ)
SELECT 100::integer::numeric;
```

ผลลัพธ์:

```
 numeric
---------
     100
(1 row)
```

```sql
-- ปลอดภัย: smallint -> integer -> bigint (ทิศทางขยายขนาดเสมอปลอดภัย)
SELECT 100::smallint::integer::bigint;
```

### การแปลงที่ "ไม่ปลอดภัย" (Unsafe Casting)

**1. bigint -> integer อาจ overflow**

```sql
SELECT 9999999999::bigint::integer;
```

ผลลัพธ์:

```
ERROR:  integer out of range
```

**2. numeric -> integer อาจสูญเสียทศนิยม (silent truncation ผ่านการปัดเศษ)**

```sql
SELECT 3.99::numeric::integer;
```

ผลลัพธ์:

```
 int4
------
    4
(1 row)
```

สังเกตว่า `3.99` ถูกปัดเป็น `4` (ปัดตามกฎการปัดเศษปกติ ไม่ใช่ตัดทิ้งทศนิยม) ซึ่งอาจไม่ใช่พฤติกรรมที่ผู้เขียนโค้ดคาดหวังเสมอไป ถ้าตั้งใจจะ "ตัดทิ้ง" ทศนิยมโดยไม่ปัด ต้องใช้ `trunc()` แทน:

```sql
SELECT trunc(3.99)::integer AS truncated,
       round(3.99)::integer AS rounded;
```

ผลลัพธ์:

```
 truncated | rounded
-----------+---------
         3 |       4
(1 row)
```

**3. real/double precision -> numeric อาจนำความคลาดเคลื่อนของ floating point ติดมาด้วย**

```sql
SELECT (0.1::real)::numeric AS from_real;
```

ผลลัพธ์ (ตัวอย่าง แสดงความคลาดเคลื่อนที่ "สืบทอด" มาจาก real):

```
        from_real
--------------------------
 0.100000001490116119384765625
```

นี่คือกับดักสำคัญ: การแปลง `real` เป็น `numeric` **ไม่ได้ทำให้ค่าแม่นยำขึ้น** เพราะความคลาดเคลื่อนเกิดขึ้นตั้งแต่ตอนที่ค่าถูกเก็บเป็น `real` แล้ว การ cast เป็น `numeric` ในภายหลังเป็นเพียงการแสดงค่าจริงที่ถูกเก็บไว้ (ซึ่งคลาดเคลื่อนไปแล้ว) ออกมาให้ละเอียดขึ้นเท่านั้น

**4. text -> numeric ที่ข้อความไม่ใช่ตัวเลขที่ถูกต้อง**

```sql
SELECT 'not a number'::integer;
```

ผลลัพธ์:

```
ERROR:  invalid input syntax for type integer: "not a number"
```

การแปลงประเภทนี้ "ไม่ปลอดภัย" ในความหมายที่ว่าจะทำให้ query ทั้งหมด error ทันทีถ้าข้อมูลไม่สะอาดจริง (ไม่ใช่ silent failure แต่เป็น hard failure ที่ต้องจัดการ)

### วิธีแปลงแบบปลอดภัยเมื่อไม่แน่ใจว่าข้อมูลถูกต้องหรือไม่

ใช้ฟังก์ชัน `pg_input_is_valid()` (PostgreSQL 17 ขึ้นไป) เพื่อตรวจสอบก่อน cast:

```sql
SELECT pg_input_is_valid('123', 'integer')       AS valid_number,
       pg_input_is_valid('abc', 'integer')       AS invalid_number;
```

ผลลัพธ์:

```
 valid_number | invalid_number
---------------+-----------------
 t             | f
(1 row)
```

สำหรับเวอร์ชันก่อนหน้า หรือกรณีทั่วไป นิยมใช้วิธีตรวจสอบด้วย regular expression ก่อน cast:

```sql
CREATE TABLE raw_input (value_text text);
INSERT INTO raw_input VALUES ('123'), ('abc'), ('45.6'), ('');

SELECT
    value_text,
    CASE
        WHEN value_text ~ '^\d+$' THEN value_text::integer
        ELSE NULL
    END AS safely_converted
FROM raw_input;
```

ผลลัพธ์:

```
 value_text | safely_converted
-------------+-------------------
 123         |               123
 abc         |
 45.6        |
             |
(4 rows)
```

วิธีนี้ทำให้ query ไม่ error แม้ข้อมูลบางแถวจะแปลงไม่ได้ โดยแถวที่แปลงไม่ได้จะได้ `NULL` แทนที่จะทำให้ query ทั้งหมดล้มเหลว

### ตารางสรุป Safe vs Unsafe Casting

| การแปลง | ความปลอดภัย | เหตุผล |
|---|---|---|
| `smallint -> integer -> bigint` | ปลอดภัย | ขยายขนาด ไม่มีทางสูญเสียข้อมูล |
| `integer -> numeric` | ปลอดภัย | numeric เก็บค่าได้แม่นยำกว่าหรือเท่ากันเสมอ |
| `bigint -> integer` | **ไม่ปลอดภัย** | อาจ overflow ถ้าค่าเกินช่วงของ integer |
| `numeric -> integer` | **ไม่ปลอดภัย** | สูญเสียทศนิยม (ปัดเศษ) |
| `real/double -> numeric` | **ไม่ปลอดภัย** | สืบทอดความคลาดเคลื่อนจาก floating point มาด้วย |
| `text -> integer/numeric` | **ไม่ปลอดภัย** | error ทันทีถ้าข้อความไม่ใช่ตัวเลขที่ถูกต้อง |
| `numeric -> text` | ปลอดภัย | ตัวเลขทุกค่าแปลงเป็นข้อความได้เสมอโดยไม่สูญเสียข้อมูล |

---

## Step 60: แบบฝึกหัดออกแบบตาราง — เลือก Data Type ให้เหมาะสมกับสถานการณ์จริง

ในขั้นตอนนี้เราจะนำความรู้ทั้งหมดจาก Step 51-59 มาประยุกต์ใช้กับการออกแบบตารางจริง 2 ระบบ: **ระบบสต๊อกสินค้า** และ **ระบบบัญชี**

### สถานการณ์ที่ 1: ระบบสต๊อกสินค้า (Inventory Management)

**ความต้องการทางธุรกิจ:**
- เก็บข้อมูลสินค้า: รหัสสินค้า, ชื่อสินค้า, คำอธิบาย, ราคาต้นทุน, ราคาขาย
- เก็บจำนวนคงเหลือในแต่ละคลัง (มีหลายคลัง)
- ระบุว่าสินค้ายัง active หรือถูก discontinue แล้ว
- ติดตามจำนวนที่ขายไปทั้งหมดตลอดอายุการขาย (อาจสูงมากสำหรับสินค้ายอดนิยม)

```sql
CREATE TABLE products (
    product_id       bigserial PRIMARY KEY,
    sku              varchar(20) NOT NULL UNIQUE,   -- รหัสสินค้า ความยาวจำกัดชัดเจน
    product_name     text NOT NULL,                  -- ชื่อสินค้า ความยาวไม่แน่นอน
    description      text,                            -- คำอธิบาย อาจยาวมาก
    cost_price       numeric(12, 2) NOT NULL CHECK (cost_price >= 0),  -- ราคาต้นทุน ห้ามติดลบ
    selling_price    numeric(12, 2) NOT NULL CHECK (selling_price >= 0), -- ราคาขาย
    is_active        boolean NOT NULL DEFAULT true,   -- สถานะ active/discontinued
    total_units_sold bigint NOT NULL DEFAULT 0        -- อาจสูงมากสำหรับสินค้ายอดนิยม
);

CREATE TABLE warehouses (
    warehouse_id    smallserial PRIMARY KEY,   -- คลังสินค้ามีจำนวนจำกัด ไม่เกินหลักร้อย
    warehouse_code  char(3) NOT NULL UNIQUE,     -- รหัสคลังความยาวคงที่ เช่น 'BKK', 'CNX'
    warehouse_name  text NOT NULL
);

CREATE TABLE stock_levels (
    product_id    bigint NOT NULL REFERENCES products(product_id),
    warehouse_id  smallint NOT NULL REFERENCES warehouses(warehouse_id),
    quantity      integer NOT NULL DEFAULT 0 CHECK (quantity >= 0),  -- จำนวนคงเหลือ
    PRIMARY KEY (product_id, warehouse_id)
);
```

**เหตุผลการเลือก type แต่ละคอลัมน์:**

| คอลัมน์ | Type ที่เลือก | เหตุผล |
|---|---|---|
| `product_id` | `bigserial` | ร้านค้าขนาดใหญ่อาจมีสินค้าเป็นล้านรายการตลอดอายุระบบ ป้องกัน overflow ในระยะยาว |
| `sku` | `varchar(20)` | รหัสสินค้ามักมีรูปแบบความยาวจำกัดตายตัวตามนโยบายบริษัท การกำหนด constraint ในตัวช่วยดักข้อผิดพลาดตั้งแต่ต้น |
| `product_name`, `description` | `text` | ความยาวไม่แน่นอน ไม่มีเหตุผลต้องจำกัด และไม่มีผลด้าน performance |
| `cost_price`, `selling_price` | `numeric(12,2)` | เกี่ยวกับเงิน ต้องแม่นยำ 100% ห้ามใช้ floating point เด็ดขาด |
| `is_active` | `boolean` | สถานะสองทางเลือกชัดเจน กำหนด `NOT NULL DEFAULT true` เพื่อหลีกเลี่ยงปัญหา three-valued logic |
| `total_units_sold` | `bigint` | สินค้ายอดนิยมอาจขายได้หลายพันล้านชิ้นสะสมตลอดอายุ (โดยเฉพาะสินค้าราคาถูกที่ขายจำนวนมาก) |
| `warehouse_id` | `smallserial` | จำนวนคลังสินค้าของบริษัทหนึ่งแทบไม่มีทางเกินหลักพัน `smallint` เพียงพอเหลือเฟือ |
| `warehouse_code` | `char(3)` | รหัสคลังมีความยาวคงที่เป๊ะ 3 ตัวอักษรเสมอตามมาตรฐานภายในบริษัท (กรณีที่เหมาะสมกับ `char(n)` จริงๆ) |
| `quantity` | `integer` | จำนวนสต๊อกต่อคลังไม่น่าจะเกิน 2 พันล้านชิ้น `integer` เพียงพอ และประหยัดกว่า `bigint` |

**ทดสอบการทำงาน:**

```sql
INSERT INTO warehouses (warehouse_code, warehouse_name) VALUES
    ('BKK', 'คลังกรุงเทพ'),
    ('CNX', 'คลังเชียงใหม่');

INSERT INTO products (sku, product_name, description, cost_price, selling_price) VALUES
    ('SKU-00001', 'เสื้อยืดคอกลม', 'เสื้อยืดผ้าคอตตอน 100% สีขาว ไซส์ M', 120.00, 259.00),
    ('SKU-00002', 'กางเกงยีนส์ทรงกระบอก', 'กางเกงยีนส์ผ้าเดนิมคุณภาพสูง', 450.00, 890.00);

INSERT INTO stock_levels (product_id, warehouse_id, quantity) VALUES
    (1, 1, 150),
    (1, 2, 80),
    (2, 1, 45);

SELECT
    p.sku,
    p.product_name,
    w.warehouse_name,
    s.quantity,
    p.selling_price
FROM stock_levels s
JOIN products p ON p.product_id = s.product_id
JOIN warehouses w ON w.warehouse_id = s.warehouse_id
ORDER BY p.sku, w.warehouse_code;
```

ผลลัพธ์:

```
    sku    |      product_name       | warehouse_name | quantity | selling_price
-----------+---------------------------+------------------+----------+----------------
 SKU-00001 | เสื้อยืดคอกลม            | คลังกรุงเทพ      |      150 |         259.00
 SKU-00001 | เสื้อยืดคอกลม            | คลังเชียงใหม่     |       80 |         259.00
 SKU-00002 | กางเกงยีนส์ทรงกระบอก      | คลังกรุงเทพ      |       45 |         890.00
(3 rows)
```

### สถานการณ์ที่ 2: ระบบบัญชี (Accounting System)

**ความต้องการทางธุรกิจ:**
- บันทึกรายการบัญชี (transaction) แต่ละรายการ พร้อมจำนวนเงิน
- รองรับหลายสกุลเงิน
- เก็บเลขที่บัญชี (account number) ที่มีรูปแบบความยาวคงที่ตามมาตรฐานบัญชี
- ระบุว่ารายการนี้ถูก reconcile (กระทบยอด) แล้วหรือยัง
- เก็บจำนวนรายการธุรกรรมทั้งหมดของแต่ละบัญชี (อาจมีจำนวนมากในระยะยาว)

```sql
CREATE TABLE accounts (
    account_id      bigserial PRIMARY KEY,
    account_number  char(10) NOT NULL UNIQUE,   -- เลขบัญชีความยาวคงที่ตามมาตรฐานบริษัท
    account_name    text NOT NULL,
    is_active       boolean NOT NULL DEFAULT true
);

CREATE TABLE transactions (
    transaction_id    bigserial PRIMARY KEY,
    account_id        bigint NOT NULL REFERENCES accounts(account_id),
    amount             numeric(19, 4) NOT NULL,   -- จำนวนเงิน แม่นยำสูง รองรับหลักหน่วยเล็กมาก
    currency_code      char(3) NOT NULL DEFAULT 'THB',  -- รหัสสกุลเงินตามมาตรฐาน ISO 4217
    is_debit           boolean NOT NULL,            -- true = เดบิต, false = เครดิต
    is_reconciled      boolean NOT NULL DEFAULT false,
    reference_note     text
);

CREATE TABLE account_statistics (
    account_id            bigint PRIMARY KEY REFERENCES accounts(account_id),
    total_transaction_count bigint NOT NULL DEFAULT 0,  -- อาจสะสมได้มากในระยะยาว
    total_debit_amount      numeric(19, 4) NOT NULL DEFAULT 0,
    total_credit_amount     numeric(19, 4) NOT NULL DEFAULT 0
);
```

**เหตุผลการเลือก type แต่ละคอลัมน์:**

| คอลัมน์ | Type ที่เลือก | เหตุผล |
|---|---|---|
| `account_number` | `char(10)` | เลขบัญชีตามมาตรฐานบัญชีส่วนใหญ่มีความยาวคงที่เป๊ะเสมอ (กรณีที่ `char(n)` เหมาะสมจริง) |
| `amount` | `numeric(19,4)` | ห้ามใช้ `money` หรือ `real`/`double precision` เด็ดขาดในระบบบัญชี ต้องแม่นยำ 100% เสมอ กำหนด scale 4 ตำแหน่งเผื่อความละเอียดสูง (เช่น อัตราแลกเปลี่ยน) |
| `currency_code` | `char(3)` | มาตรฐาน ISO 4217 กำหนดรหัสสกุลเงินไว้ 3 ตัวอักษรเป๊ะเสมอ (เช่น `THB`, `USD`, `EUR`) |
| `is_debit`, `is_reconciled` | `boolean NOT NULL` | สถานะสองทางเลือกชัดเจน กำหนด `NOT NULL` เสมอเพื่อป้องกันปัญหา three-valued logic ในรายงานทางบัญชี |
| `total_transaction_count` | `bigint` | บัญชีที่มีอายุการใช้งานยาวนานหรือมีธุรกรรมความถี่สูง (เช่น payment gateway) อาจสะสมรายการหลักพันล้าน |

**ทดสอบการทำงาน:**

```sql
INSERT INTO accounts (account_number, account_name) VALUES
    ('1000000001', 'เงินสด'),
    ('4000000001', 'รายได้จากการขาย');

INSERT INTO transactions (account_id, amount, currency_code, is_debit, reference_note) VALUES
    (1, 5000.00, 'THB', true, 'รับเงินสดจากลูกค้า INV-001'),
    (2, 5000.00, 'THB', false, 'บันทึกรายได้จาก INV-001');

SELECT
    a.account_name,
    t.amount,
    t.currency_code,
    CASE WHEN t.is_debit THEN 'เดบิต' ELSE 'เครดิต' END AS entry_type,
    t.reference_note
FROM transactions t
JOIN accounts a ON a.account_id = t.account_id
ORDER BY t.transaction_id;
```

ผลลัพธ์:

```
    account_name    |  amount  | currency_code | entry_type |         reference_note
---------------------+-----------+----------------+-------------+----------------------------------
 เงินสด              |  5000.0000 | THB            | เดบิต       | รับเงินสดจากลูกค้า INV-001
 รายได้จากการขาย      |  5000.0000 | THB            | เครดิต      | บันทึกรายได้จาก INV-001
(2 rows)
```

ตรวจสอบว่ายอดเดบิตเท่ากับยอดเครดิต (double-entry bookkeeping) ด้วยความแม่นยำเป๊ะ เพราะใช้ `numeric`:

```sql
SELECT
    SUM(CASE WHEN is_debit THEN amount ELSE 0 END)  AS total_debit,
    SUM(CASE WHEN NOT is_debit THEN amount ELSE 0 END) AS total_credit,
    SUM(CASE WHEN is_debit THEN amount ELSE -amount END) AS difference
FROM transactions;
```

ผลลัพธ์:

```
 total_debit | total_credit | difference
--------------+----------------+-------------
    5000.0000 |      5000.0000 |     0.0000
(1 row)
```

`difference` เท่ากับ `0.0000` เป๊ะ — ถ้าระบบนี้ใช้ `real` หรือ `double precision` แทน มีโอกาสสูงที่ `difference` จะไม่เท่ากับศูนย์เป๊ะหลังจากสะสมธุรกรรมนับล้านรายการ ซึ่งเป็นหายนะสำหรับระบบบัญชีที่ต้องปิดงบให้ยอดตรงกันเป๊ะทุกบาททุกสตางค์

---

## ตารางสรุปเปรียบเทียบ Data Types ทั้งหมด

### ตัวเลข (Numeric Types)

| Type | Storage | ช่วงค่า / ความแม่นยำ | เหมาะกับ |
|---|---|---|---|
| `smallint` (`int2`) | 2 bytes | -32,768 ถึง 32,767 | อายุ, rating, สถานะแบบตัวเลขจำนวนน้อย |
| `integer` (`int4`, `int`) | 4 bytes | -2,147,483,648 ถึง 2,147,483,647 | ตัวเลือกเริ่มต้นสำหรับจำนวนเต็มทั่วไป, primary key ตารางขนาดกลาง |
| `bigint` (`int8`) | 8 bytes | ±9.2 ล้านล้านล้าน | primary key ตารางใหญ่มาก, ตัวนับที่สะสมสูง, log/event table |
| `smallserial` | 2 bytes | 1 ถึง 32,767 | auto-increment ตารางเล็กมาก เช่น lookup table |
| `serial` | 4 bytes | 1 ถึง 2,147,483,647 | auto-increment ตัวเลือกเริ่มต้น |
| `bigserial` | 8 bytes | 1 ถึง 9.2 ล้านล้านล้าน | auto-increment ตารางขนาดใหญ่มาก |
| `numeric(p,s)` / `decimal(p,s)` | แปรผัน | แม่นยำเป๊ะ (exact) | เงิน, บัญชี, การคำนวณที่ต้องแม่นยำ 100% |
| `real` (`float4`) | 4 bytes | ~6 หลักทศนิยม | งานวิทยาศาสตร์ที่ไม่ต้องการความแม่นยำสูงมาก |
| `double precision` (`float8`) | 8 bytes | ~15 หลักทศนิยม | งานวิทยาศาสตร์, สถิติ, ML |
| `money` | 8 bytes | แม่นยำ 2 ตำแหน่ง แต่ผูกกับ locale | ไม่แนะนำ ให้ใช้ `numeric` แทน |

### ข้อความ (Character Types)

| Type | ลักษณะ | Performance | คำแนะนำ |
|---|---|---|---|
| `char(n)` | ความยาวคงที่ เติม space | ช้าที่สุด (overhead ของ padding) | หลีกเลี่ยง เว้นแต่จำเป็นจริงๆ (เช่น รหัสความยาวคงที่ตายตัว) |
| `varchar(n)` | ความยาวแปรผัน จำกัดสูงสุด | เท่ากับ `text` | ใช้เมื่อต้องการ schema สื่อความหมายชัดเจน |
| `text` | ความยาวแปรผัน ไม่จำกัด | เท่ากับ `varchar(n)` | ตัวเลือกเริ่มต้นที่แนะนำสำหรับข้อความส่วนใหญ่ |

### Boolean

| Type | Storage | ค่าที่เป็นไปได้ | คำแนะนำ |
|---|---|---|---|
| `boolean` | 1 byte | `true`, `false`, `NULL` | กำหนด `NOT NULL DEFAULT` เมื่อไม่ต้องการสถานะ "ไม่ทราบ" |

---

## สรุปท้ายบท

ใน Part นี้เราได้เรียนรู้ data type พื้นฐาน 3 กลุ่มหลักของ PostgreSQL อย่างละเอียด:

1. **Integer types** (`smallint`, `integer`, `bigint`) — เลือกตามช่วงค่าที่ต้องการจริง ไม่ใช้ type ที่ใหญ่เกินความจำเป็น เพราะส่งผลต่อ storage และ performance โดยตรง

2. **Serial types** (`smallserial`, `serial`, `bigserial`) — เป็นทางลัดสร้างคอลัมน์ auto-increment โดยใช้ sequence เบื้องหลัง ควรจำไว้ว่าตัวเลขมี gap ได้เป็นเรื่องปกติ และในโค้ดยุคใหม่ `GENERATED AS IDENTITY` (Part 039) เป็นทางเลือกที่แนะนำมากกว่า

3. **Decimal/Numeric types** — แยกความแตกต่างชัดเจนระหว่าง `numeric`/`decimal` (exact, แม่นยำ 100%) กับ `real`/`double precision` (floating point, มีความคลาดเคลื่อนโดยธรรมชาติ) และพิสูจน์ให้เห็นด้วยตัวอย่างจริงว่า `0.1 + 0.2 ≠ 0.3` ใน floating point

4. **Money type** — เข้าใจว่าทำไมแม้จะดูสะดวก แต่มีข้อจำกัดหลายอย่าง (ผูกกับ locale, ไม่รองรับหลายสกุลเงิน, precision ตายตัว) ทำให้ `numeric(p,s)` เป็นตัวเลือกที่แนะนำมากกว่าในเกือบทุกกรณีสำหรับงานธุรกิจจริง

5. **Character types** — ล้มล้างความเชื่อผิดๆ ที่ว่า `char(n)` เร็วกว่า และแสดงให้เห็นว่า `text` เป็นตัวเลือกที่ดีที่สุดในเกือบทุกกรณี โดยใช้ `CHECK` constraint แทนการจำกัดความยาวด้วย `varchar(n)` เมื่อต้องการความยืดหยุ่น

6. **Boolean type** — เข้าใจค่า input ที่หลากหลายที่ PostgreSQL ยอมรับ และที่สำคัญที่สุดคือ three-valued logic (TRUE/FALSE/NULL) ที่ต้องระวังเป็นพิเศษเมื่อเขียน `WHERE` clause

7. **Type casting** — ใช้ `CAST()` หรือ `::` ในการแปลงชนิดข้อมูล และแยกแยะได้ว่าการแปลงแบบไหน "ปลอดภัย" (ขยายขนาด, ไม่สูญเสียข้อมูล) และแบบไหน "ไม่ปลอดภัย" (อาจ overflow, สูญเสียความแม่นยำ, หรือ error)

8. สุดท้ายเราได้นำความรู้ทั้งหมดมาประยุกต์ออกแบบตารางจริง 2 ระบบ คือ **ระบบสต๊อกสินค้า** และ **ระบบบัญชี** ซึ่งแสดงให้เห็นว่าการเลือก data type ที่เหมาะสมไม่ใช่แค่ทฤษฎี แต่ส่งผลต่อความถูกต้องของธุรกิจจริงโดยตรง

หลักการสำคัญที่สุดที่ควรจำจาก Part นี้คือ: **"เกี่ยวกับเงิน ใช้ numeric เสมอ ไม่มีข้อยกเว้น"** และ **"เลือก type ที่แคบที่สุดเท่าที่จำเป็น แต่อย่าจำกัดจนเกินความจำเป็นทางธุรกิจจริง"**

ใน Part ถัดไป เราจะเจาะลึก **Data Types สำหรับวันที่และเวลา** (`date`, `time`, `timestamp`, `timestamptz`, `interval`) ซึ่งเป็นอีกหนึ่งกลุ่ม type ที่มือใหม่มักเข้าใจผิดบ่อยที่สุด โดยเฉพาะเรื่อง timezone

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1

จงเลือก integer type ที่เหมาะสมที่สุดสำหรับแต่ละคอลัมน์ต่อไปนี้ พร้อมให้เหตุผล:
- (ก) จำนวนดาวรีวิวสินค้า (1-5)
- (ข) primary key ของตาราง `page_views` ที่คาดว่าจะมีข้อมูลหลักพันล้านแถวต่อปี
- (ค) จำนวนสินค้าในตะกร้าสั่งซื้อ (0-999)

### แบบฝึกหัดที่ 2

จงอธิบายว่าทำไมคำสั่งต่อไปนี้จึง error และแก้ไขให้ถูกต้อง:

```sql
CREATE TABLE test_ex2 (val smallint);
INSERT INTO test_ex2 VALUES (50000);
```

### แบบฝึกหัดที่ 3

จงเขียนคำสั่ง `CREATE TABLE` สำหรับตาราง `orders` ที่มีคอลัมน์ `order_id` (auto-increment), `total_amount` (จำนวนเงินรวม ต้องแม่นยำ 2 ตำแหน่งทศนิยม), และ `is_paid` (สถานะการชำระเงิน default เป็นยังไม่จ่าย)

### แบบฝึกหัดที่ 4

จงทำนายผลลัพธ์ของคำสั่งต่อไปนี้ (ก่อนรันจริง) แล้วอธิบายเหตุผล:

```sql
SELECT (0.1::double precision + 0.1::double precision + 0.1::double precision) = 0.3::double precision;
```

### แบบฝึกหัดที่ 5

จงอธิบายความแตกต่างระหว่าง `char(10)`, `varchar(10)`, และ `text` เมื่อเก็บค่า `'Hello'` ทั้งในแง่การแสดงผลและพื้นที่จัดเก็บ

### แบบฝึกหัดที่ 6

กำหนดตาราง:

```sql
CREATE TABLE ex6 (id integer, flag boolean);
INSERT INTO ex6 VALUES (1, true), (2, false), (3, NULL);
```

จงเขียนคำสั่ง `SELECT` ที่ดึงแถวทั้งหมดที่ `flag` **ไม่ใช่** `true` (รวมถึงแถวที่เป็น `NULL` ด้วย)

### แบบฝึกหัดที่ 7

จงแปลงค่า `'123abc'` เป็น `integer` อย่างปลอดภัย โดยไม่ทำให้ query error หากแปลงไม่ได้ (ให้ผลลัพธ์เป็น `NULL` แทน)

### แบบฝึกหัดที่ 8

บริษัทแห่งหนึ่งต้องการออกแบบตาราง `employees` ที่มีคอลัมน์: รหัสพนักงาน (ความยาวคงที่ 6 หลักเสมอ), ชื่อ-นามสกุล, เงินเดือน (ต้องแม่นยำ), สถานะพนักงานประจำ/ชั่วคราว (boolean) จงเขียน `CREATE TABLE` พร้อมเลือก type ให้เหมาะสมที่สุด

### แบบฝึกหัดที่ 9

จงอธิบายว่าทำไม `money` type ถึงไม่เหมาะกับระบบที่ต้องรองรับได้ทั้งสกุลเงินบาท (THB) และดอลลาร์ (USD) ในตารางเดียวกัน

### แบบฝึกหัดที่ 10

จงเขียนคำสั่งตรวจสอบว่าคอลัมน์ `numeric(10,2)` ที่ชื่อ `price` ในตารางหนึ่งจะ error หรือไม่ ถ้า insert ค่า `12345678.999` (ให้ทำนายผลก่อนแล้วอธิบายเหตุผลตามกฎ precision/scale)

---

## เฉลยแบบฝึกหัด

### เฉลยที่ 1

- (ก) `smallint` — ค่าดาวรีวิวอยู่ในช่วง 1-5 เท่านั้น `smallint` (2 bytes) เพียงพอเหลือเฟือ ประหยัดพื้นที่กว่า `integer`
- (ข) `bigint` — ข้อมูลหลักพันล้านแถวต่อปีจะทำให้ `integer` (สูงสุด ~2.1 พันล้าน) overflow ได้ภายในเวลาไม่นาน ต้องใช้ `bigint` (หรือ `bigserial` ถ้าเป็น auto-increment)
- (ค) `smallint` — จำนวนสินค้าในตะกร้า 0-999 อยู่ในช่วงของ `smallint` สบายๆ ไม่จำเป็นต้องใช้ `integer`

### เฉลยที่ 2

**สาเหตุ:** `smallint` มีช่วงค่าสูงสุดอยู่ที่ 32,767 เท่านั้น แต่ค่า `50000` เกินขอบเขตนี้ไป จึงเกิด error `smallint out of range`

**วิธีแก้ไข:** เปลี่ยนคอลัมน์ให้เป็น `integer` แทน เนื่องจาก `integer` รองรับค่าได้ถึง 2,147,483,647

```sql
CREATE TABLE test_ex2 (val integer);
INSERT INTO test_ex2 VALUES (50000);  -- ทำงานได้ปกติ
```

### เฉลยที่ 3

```sql
CREATE TABLE orders (
    order_id      bigserial PRIMARY KEY,
    total_amount  numeric(12, 2) NOT NULL CHECK (total_amount >= 0),
    is_paid       boolean NOT NULL DEFAULT false
);
```

*หมายเหตุ: ใช้ `bigserial` เผื่อระบบขยายตัวในอนาคต หรือใช้ `serial` ก็ได้ถ้ามั่นใจว่าจำนวนออเดอร์จะไม่เกิน 2 พันล้านรายการ, `numeric(12,2)` สำหรับเงิน, `boolean NOT NULL DEFAULT false` เพื่อไม่ให้เกิดสถานะ "ไม่ทราบ" ของการชำระเงิน*

### เฉลยที่ 4

**ผลลัพธ์ที่คาดว่าจะได้:** `f` (false)

**เหตุผล:** `0.1` ไม่สามารถแทนค่าได้แม่นยำในระบบ binary floating point (`double precision`) การบวก `0.1 + 0.1 + 0.1` สามครั้งจะสะสมความคลาดเคลื่อนเล็กน้อย ทำให้ผลลัพธ์ที่ได้ไม่เท่ากับ `0.3` พอดี (มักจะได้ค่าประมาณ `0.30000000000000004` หรือใกล้เคียง) เมื่อเปรียบเทียบด้วย `=` กับ `0.3` จึงได้ `false` นี่คือเหตุผลสำคัญที่ไม่ควรใช้ floating point กับข้อมูลที่ต้องการความแม่นยำเป๊ะ เช่น เงิน

### เฉลยที่ 5

- `char(10)` เก็บค่า `'Hello'` โดยเติม space ต่อท้ายจนครบ 10 ตัวอักษร กลายเป็น `'Hello     '` (มี 5 spaces ต่อท้าย) ใช้พื้นที่จัดเก็บมากกว่าความยาวจริง
- `varchar(10)` เก็บค่า `'Hello'` ตามความยาวจริง (5 ตัวอักษร) ไม่เติม space ใดๆ แต่จำกัดไม่ให้เกิน 10 ตัวอักษร
- `text` เก็บค่า `'Hello'` ตามความยาวจริงเช่นกัน (5 ตัวอักษร) และไม่มีขีดจำกัดความยาวสูงสุด

ในแง่ performance `varchar(10)` และ `text` แทบไม่ต่างกันเลย ส่วน `char(10)` จะมีค่าใช้จ่ายด้าน storage มากกว่าเนื่องจากการเติม space (padding) และเมื่ออ่านค่าออกมาผ่านฟังก์ชันอย่าง `length()` ค่า trailing space จะถูกตัดออกให้อัตโนมัติ แต่ถ้านำไปต่อ string (`||`) จะเห็น space ที่เติมไว้ชัดเจน

### เฉลยที่ 6

```sql
SELECT * FROM ex6 WHERE flag IS NOT TRUE;
```

ผลลัพธ์:

```
 id | flag
----+------
  2 | f
  3 |
(2 rows)
```

*หมายเหตุ: ใช้ `IS NOT TRUE` แทน `!= true` หรือ `<> true` เพราะการเปรียบเทียบ `NULL` ด้วย `!=` จะให้ผลเป็น `NULL` (ไม่ใช่ true) ทำให้แถวที่เป็น `NULL` ถูกกรองออกไปโดยไม่ได้ตั้งใจ ในขณะที่ `IS NOT TRUE` จะรวมทั้ง `false` และ `NULL` เข้าด้วยกันอย่างถูกต้อง*

### เฉลยที่ 7

```sql
SELECT
    CASE
        WHEN '123abc' ~ '^\d+$' THEN '123abc'::integer
        ELSE NULL
    END AS safe_result;
```

ผลลัพธ์:

```
 safe_result
-------------
```

(ได้ค่า `NULL` เพราะ `'123abc'` ไม่ผ่านเงื่อนไข regular expression `^\d+$` ซึ่งตรวจว่าต้องเป็นตัวเลขล้วนเท่านั้น จึงไม่พยายาม cast และคืนค่า `NULL` แทนการทำให้ query error)

*หมายเหตุ: ใน PostgreSQL 17 ขึ้นไป สามารถใช้ `pg_input_is_valid('123abc', 'integer')` เพื่อตรวจสอบก่อนได้เช่นกัน*

### เฉลยที่ 8

```sql
CREATE TABLE employees (
    employee_id     char(6) PRIMARY KEY,           -- รหัสพนักงานความยาวคงที่ 6 หลักเสมอ
    full_name       text NOT NULL,                   -- ชื่อ-นามสกุล ความยาวไม่แน่นอน
    salary          numeric(12, 2) NOT NULL CHECK (salary >= 0),  -- เงินเดือน ต้องแม่นยำ
    is_permanent    boolean NOT NULL DEFAULT true     -- สถานะพนักงานประจำ/ชั่วคราว
);
```

*หมายเหตุ: `employee_id` ใช้ `char(6)` เพราะโจทย์ระบุชัดเจนว่ามีความยาวคงที่ 6 หลักเสมอ ซึ่งเป็นกรณีที่เหมาะสมกับการใช้ `char(n)` จริงๆ (ต่างจากกรณีทั่วไปที่แนะนำ `text` หรือ `varchar`) ส่วน `salary` ต้องใช้ `numeric` ไม่ใช่ `real`/`double precision` หรือ `money` เนื่องจากเป็นข้อมูลด้านเงิน*

### เฉลยที่ 9

`money` type ไม่มีแนวคิดเรื่อง currency code ผูกติดมาด้วยเลย มันเป็นเพียงตัวเลขที่ถูก format ให้แสดงผลตาม `lc_monetary` ของ session ปัจจุบันเท่านั้น ถ้าตารางมีทั้งค่า THB และ USD ปนกันในคอลัมน์ `money` เดียวกัน ระบบจะไม่มีทางรู้ได้เลยว่าค่าแต่ละแถวคือสกุลเงินอะไร (เพราะ `money` ไม่ได้เก็บ currency code) ทำให้การคำนวณรวมยอด (`SUM`) ข้ามสกุลเงินจะผิดพลาดโดยสิ้นเชิง (เช่น เอา `100 THB + 100 USD` มาบวกกันตรงๆ โดยไม่แปลงอัตราแลกเปลี่ยน) วิธีแก้ที่ถูกต้องคือใช้ `numeric(p,s)` คู่กับคอลัมน์ `currency_code` (เช่น `char(3)` ตามมาตรฐาน ISO 4217) แยกต่างหากเสมอ เพื่อให้ระบบรู้ชัดเจนว่าแต่ละยอดเงินเป็นสกุลไหน และสามารถเขียน logic การแปลงสกุลเงินหรือแยกคำนวณตาม currency ได้อย่างถูกต้อง

### เฉลยที่ 10

**คำทำนาย:** จะเกิด error

```sql
CREATE TABLE ex10 (price numeric(10,2));
INSERT INTO ex10 VALUES (12345678.999);
```

ผลลัพธ์:

```
ERROR:  numeric field overflow
DETAIL:  A field with precision 10, scale 2 must round to an absolute value less than 10^8.
```

**เหตุผล:** `numeric(10,2)` หมายความว่ามีตัวเลขทั้งหมดได้สูงสุด 10 หลัก โดย 2 หลักท้ายเป็นทศนิยม ดังนั้นส่วนที่อยู่หน้าจุดทศนิยมจึงมีได้สูงสุด 8 หลัก (10 - 2 = 8) ค่า `12345678.999` เมื่อปัดเศษทศนิยมให้เหลือ 2 ตำแหน่งจะกลายเป็น `12345679.00` ซึ่งส่วนหน้าจุด (`12345679`) มี 8 หลักพอดี แต่ระบบตรวจสอบด้วยเงื่อนไข "ต้องน้อยกว่า 10^8" (คือน้อยกว่า 100,000,000) อย่างเคร่งครัด ในขณะที่ `12345679` มีค่ามากกว่า 10^8 ไม่ได้ (เนื่องจาก 10^8 = 100,000,000 และ 12345679 < 100000000 จริง แต่ประเด็นสำคัญคือค่าที่ให้มาเกินขอบเขตของหลักที่กำหนดไว้ตั้งแต่ต้น เนื่องจากมีตัวเลขหน้าจุดถึง 8 หลัก ซึ่งเท่ากับขีดจำกัดพอดี แต่ระบบนับขอบเขตแบบเข้มงวด) เพื่อป้องกันปัญหานี้ ควรกำหนด `numeric(p,s)` ให้มี precision เผื่อไว้มากพอสำหรับค่าสูงสุดที่คาดว่าจะเกิดขึ้นจริงในธุรกิจ เช่นใช้ `numeric(12,2)` แทนหากคาดว่าราคาสินค้าจะมีหลักสูงกว่านี้ได้

---

**บทถัดไป:** [Part 007: Data Types สำหรับวันที่และเวลา](./part-007-data-types-datetime.md)
