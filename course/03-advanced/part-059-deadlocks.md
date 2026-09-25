# Deadlock: สาเหตุ การตรวจจับ และการป้องกัน

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับสูง | Part 059

---

## เป้าหมายการเรียนรู้

เมื่อจบบทนี้ ผู้เรียนจะสามารถ:

1. อธิบายได้ว่า deadlock คืออะไร เกิดขึ้นได้อย่างไร และทำไมระบบฐานข้อมูลถึงต้องมีกลไกจัดการมัน
2. จำลอง deadlock จริงด้วยสอง psql session และอ่านข้อความ error `deadlock detected` ได้อย่างเข้าใจทุกส่วน
3. เข้าใจการทำงานของ Deadlock Detector ใน PostgreSQL รวมถึง parameter `deadlock_timeout` และหลักการเลือก "เหยื่อ" (victim transaction)
4. เปิดและอ่าน log ที่เกี่ยวกับ lock wait ด้วย `log_lock_waits` เพื่อสืบสวนปัญหาที่เกิดใน production
5. ออกแบบโค้ดแอปพลิเคชันด้วยหลัก **consistent lock ordering** เพื่อป้องกัน deadlock ตั้งแต่ต้นทาง
6. ลดขนาดและระยะเวลาของ transaction เพื่อลดโอกาสเกิด deadlock และปัญหา lock contention โดยรวม
7. ใช้ `SELECT ... FOR UPDATE` เพื่อ "จอง" แถวล่วงหน้าตามลำดับที่แน่นอน
8. ตั้งค่า `lock_timeout` และ `statement_timeout` เพื่อป้องกันไม่ให้ transaction ค้างรอ lock นานเกินไป
9. วิเคราะห์และแก้ไขโค้ดจริงที่มีความเสี่ยง deadlock ในสถานการณ์โอนเงิน/โอนสต๊อกระหว่างบัญชี

---

## เตรียมข้อมูล

บทนี้ใช้ schema อีคอมเมิร์ซชุดเดิมที่ใช้ต่อเนื่องมาตลอดหลักสูตร คือ `customers`, `products`, `orders` โดยเน้นที่ตาราง `customers` และ `products` เพราะ deadlock มักเกิดจากการ UPDATE แถวหลายแถวในลำดับที่ต่างกันระหว่างสอง transaction

```sql
-- ล้างของเก่า (ถ้ามี) เพื่อให้รันซ้ำได้
DROP TABLE IF EXISTS orders;
DROP TABLE IF EXISTS products;
DROP TABLE IF EXISTS customers;

CREATE TABLE customers (
    customer_id   SERIAL PRIMARY KEY,
    first_name    VARCHAR(60) NOT NULL,
    balance       NUMERIC(12,2) NOT NULL DEFAULT 0
);

CREATE TABLE products (
    product_id      SERIAL PRIMARY KEY,
    product_name    VARCHAR(150) NOT NULL,
    unit_price      NUMERIC(10,2) NOT NULL,
    stock_quantity  INTEGER NOT NULL DEFAULT 0
);

CREATE TABLE orders (
    order_id     SERIAL PRIMARY KEY,
    customer_id  INTEGER REFERENCES customers(customer_id),
    status       VARCHAR(20) NOT NULL DEFAULT 'pending'
);

-- ข้อมูลลูกค้า (ใช้จำลองการโอนเงินระหว่างบัญชี)
INSERT INTO customers (first_name, balance) VALUES
    ('สมชาย',   50000.00),
    ('สมหญิง',  32000.00),
    ('วิภา',    15000.00),
    ('อนุชา',   8000.00),
    ('ปิยะดา',  120000.00);

-- ข้อมูลสินค้า (ใช้จำลองการโอนสต๊อกระหว่างคลัง/ระหว่าง SKU)
INSERT INTO products (product_name, unit_price, stock_quantity) VALUES
    ('เมาส์ไร้สาย Logitech M185',      290.00,  120),
    ('คีย์บอร์ดกลไก Keychron K2',      2490.00, 45),
    ('หูฟังบลูทูธ Sony WH-CH520',      1590.00, 60),
    ('จอมอนิเตอร์ 24 นิ้ว Dell',       4990.00, 20),
    ('เว็บแคม Logitech C920',          1990.00, 35);

INSERT INTO orders (customer_id, status) VALUES
    (1, 'pending'),
    (2, 'paid'),
    (3, 'pending');
```

ตรวจสอบข้อมูลก่อนเริ่ม:

```sql
SELECT * FROM customers ORDER BY customer_id;
SELECT * FROM products  ORDER BY product_id;
SELECT * FROM orders    ORDER BY order_id;
```

> **หมายเหตุ:** ตัวอย่างในบทนี้ต้องใช้ **สอง session พร้อมกัน** เพื่อจำลอง deadlock จริง เราจะเรียกมันว่า **Session A** และ **Session B** ตลอดทั้งบท ให้เปิด terminal สองหน้าต่าง แล้ว `psql` เข้าฐานข้อมูลเดียวกันทั้งคู่

---

## Step 581: Deadlock คืออะไร — circular wait ที่ไม่มีทางไปต่อ

**Deadlock** คือสถานการณ์ที่ transaction สองตัว (หรือมากกว่า) แต่ละตัวถือ lock อยู่บนทรัพยากรหนึ่ง แล้วรอ lock อีกทรัพยากรหนึ่งที่อีก transaction ถืออยู่ — เกิดเป็น **วงจรการรอ (circular wait)** ที่ไม่มีทางคลี่คลายได้เองเลย ไม่ว่าจะรอไปนานแค่ไหน

ลองนึกภาพง่าย ๆ:

```
Transaction A: ถือ Lock บนแถว X, ต้องการ Lock บนแถว Y
Transaction B: ถือ Lock บนแถว Y, ต้องการ Lock บนแถว X

A รอ B ปล่อย Y  →  แต่ B กำลังรอ A ปล่อย X
B รอ A ปล่อย X  →  แต่ A กำลังรอ B ปล่อย Y

ผลลัพธ์: ทั้งสองฝ่ายรอกันไปตลอดกาล (ถ้าไม่มีใครมาแทรกแซง)
```

เงื่อนไข 4 ข้อที่ทำให้เกิด deadlock ได้ (Coffman conditions) คือ:

1. **Mutual exclusion** — ทรัพยากรถูกถือครองแบบ exclusive (เช่น row lock สำหรับ UPDATE)
2. **Hold and wait** — transaction ถือ lock หนึ่งอยู่ พร้อมกับรอ lock อีกตัวหนึ่ง
3. **No preemption** — ไม่มีใครมาแย่ง lock คืนจาก transaction ที่ถือครองอยู่ได้ (ต้องรอให้ปล่อยเอง)
4. **Circular wait** — เกิดวงจรการรอแบบวนกลับมาที่ตัวเอง

ใน PostgreSQL เงื่อนไข 3 ข้อแรกเป็นธรรมชาติของระบบ transaction/locking อยู่แล้ว สิ่งที่แอปพลิเคชันควบคุมได้คือ **ข้อ 4 — circular wait** ซึ่งเกิดจาก "ลำดับการเข้าถึงทรัพยากรที่ไม่สอดคล้องกัน" ระหว่างสอง transaction

จุดสำคัญที่ต้องเข้าใจ:

- Deadlock **ไม่ใช่** lock contention ธรรมดา (ที่ transaction หนึ่งรอ transaction อื่นแล้วในที่สุดก็ได้ lock) — deadlock คือสถานการณ์ที่ **ไม่มีทางได้ lock เลย** ถ้าไม่มีใครถูกยกเลิก
- PostgreSQL มี **deadlock detector** ที่ทำงานอัตโนมัติ คอยตรวจจับวงจรการรอ แล้วเลือก transaction หนึ่งมา ROLLBACK เพื่อทำลายวงจร (จะอธิบายละเอียดใน Step 583)
- Deadlock เป็นเรื่องปกติที่เกิดขึ้นได้ในระบบที่มี concurrency สูง — ไม่ใช่ "บั๊ก" ของ PostgreSQL แต่เป็นผลจาก**การออกแบบโค้ดแอปพลิเคชัน**ที่ไม่ได้ควบคุมลำดับการล็อก

ตารางเปรียบเทียบ:

| ลักษณะ | Lock Wait ธรรมดา | Deadlock |
|---|---|---|
| จำนวน transaction ที่เกี่ยวข้อง | 2 หรือมากกว่า | 2 หรือมากกว่า (เป็นวงจร) |
| ผลลัพธ์สุดท้าย | รอจนกว่าเจ้าของ lock จะ COMMIT/ROLLBACK แล้วได้ lock | ไม่มีทางได้ lock เอง ต้องมีการ ROLLBACK แทรกแซง |
| PostgreSQL จัดการอย่างไร | รอเฉย ๆ (จนกว่าจะปล่อย หรือ timeout ถ้าตั้งไว้) | ตรวจจับวงจร แล้ว ROLLBACK transaction หนึ่งโดยอัตโนมัติ |
| ความถี่ | เกิดบ่อยเป็นปกติ | เกิดเมื่อโค้ดเข้าถึงทรัพยากรคนละลำดับ |

ใน Step ถัดไป เราจะจำลอง deadlock แบบคลาสสิกให้เห็นภาพจริงด้วยสอง session

---

## Step 582: ตัวอย่าง Deadlock คลาสสิก — UPDATE สองแถวสลับลำดับกัน

สถานการณ์: ระบบมีสอง transaction ที่ต้องอัปเดต **ยอดเงิน (balance)** ของลูกค้าสองคน คือ `customer_id = 1` (สมชาย) และ `customer_id = 2` (สมหญิง) แต่ transaction แรกอัปเดต 1 ก่อนแล้วค่อย 2 ในขณะที่ transaction ที่สองอัปเดต 2 ก่อนแล้วค่อย 1 — นี่คือสูตรคลาสสิกของ deadlock

### เตรียมสถานการณ์

สมมติว่าเรามีสอง business process:

- **Session A** จำลอง "โอนเงินจากสมชาย (id=1) ไปสมหญิง (id=2)"
- **Session B** จำลอง "โอนเงินจากสมหญิง (id=2) ไปสมชาย (id=1)" ที่เกิดขึ้น**พร้อมกัน**โดยบังเอิญ (เช่น ลูกค้าสองคนโอนเงินคืนกันในเวลาไล่เลี่ยกัน)

โค้ดของทั้งสอง transaction (แบบที่ **ยังไม่ได้ป้องกัน deadlock**):

```sql
-- ============ Session A: โอนจาก customer_id=1 ไป customer_id=2 ============
BEGIN;
UPDATE customers SET balance = balance - 1000 WHERE customer_id = 1;
-- (หน่วงเวลาตรงนี้เพื่อจำลอง business logic ระหว่างสอง UPDATE)
UPDATE customers SET balance = balance + 1000 WHERE customer_id = 2;
COMMIT;
```

```sql
-- ============ Session B: โอนจาก customer_id=2 ไป customer_id=1 ============
BEGIN;
UPDATE customers SET balance = balance - 500 WHERE customer_id = 2;
-- (หน่วงเวลาตรงนี้เพื่อจำลอง business logic ระหว่างสอง UPDATE)
UPDATE customers SET balance = balance + 500 WHERE customer_id = 1;
COMMIT;
```

### ขั้นตอนจำลองจริงด้วยสอง psql session

เปิด terminal สองหน้าต่าง เชื่อมต่อฐานข้อมูลเดียวกันทั้งคู่ (`psql -d your_db`) แล้วทำตามลำดับนี้ **ทีละบรรทัด สลับกันไปมาตามลำดับเวลา**:

```
เวลา   Session A                                  Session B
-----  ------------------------------------------ ------------------------------------------
T1     BEGIN;
T2     UPDATE customers SET balance = balance
       - 1000 WHERE customer_id = 1;
       -- สำเร็จ: A ถือ lock บนแถว id=1
T3                                                 BEGIN;
T4                                                 UPDATE customers SET balance = balance
                                                    - 500 WHERE customer_id = 2;
                                                    -- สำเร็จ: B ถือ lock บนแถว id=2
T5     UPDATE customers SET balance = balance
       + 1000 WHERE customer_id = 2;
       -- **ค้าง** เพราะแถว id=2 ถูก B ล็อกอยู่ (A รอ B)
T6                                                 UPDATE customers SET balance = balance
                                                    + 500 WHERE customer_id = 1;
                                                    -- **ค้าง** เพราะแถว id=1 ถูก A ล็อกอยู่ (B รอ A)
                                                    -- ตอนนี้เกิด circular wait: A รอ B, B รอ A
T7     <<< หลังจากผ่านไปประมาณ deadlock_timeout (ค่า default 1 วินาที)
       PostgreSQL ตรวจพบวงจร แล้วเลือกหนึ่งใน A/B เป็น "เหยื่อ" (victim)
       Session ที่เป็นเหยื่อจะได้ error ทันที ส่วนอีก session จะได้ lock ต่อและทำงานสำเร็จ
```

ในเชิงปฏิบัติ ให้พิมพ์คำสั่งตามลำดับ T1–T6 ในแต่ละ session (คั่นเวลาสัก 1-2 วินาทีต่อบรรทัดที่ T5→T6 เพื่อให้แน่ใจว่า circular wait เกิดขึ้นจริง) แล้วสังเกตว่า session ใด session หนึ่งจะได้ข้อความ error ทันทีที่ผ่านไปประมาณ 1 วินาทีหลัง T6

### ข้อความ error ที่จะได้ (ตัวอย่างจริง)

Session หนึ่ง (สมมติว่าเป็น Session A ในตัวอย่างนี้ ผลจริงอาจสลับกันได้เพราะ detector เลือกเหยื่อตามกฎของมันเอง) จะได้:

```
ERROR:  deadlock detected
DETAIL:  Process 48213 waits for ShareLock on transaction 892; blocked by process 48227.
Process 48227 waits for ShareLock on transaction 891; blocked by process 48213.
HINT:  See server log for query details.
CONTEXT:  while updating tuple (0,2) in relation "customers"
```

ส่วน Session B จะได้รับ lock และทำงานต่อจนจบตามปกติ:

```
UPDATE 1
COMMIT
```

> **สังเกต:** Session ที่ถูกเลือกเป็นเหยื่อจะต้อง `ROLLBACK;` (หรือ PostgreSQL จะปฏิเสธคำสั่งถัดไปจนกว่าจะ ROLLBACK เพราะ transaction อยู่ในสถานะ aborted) แล้วแอปพลิเคชันควรมี**ตรรกะ retry** เพื่อลองทำ transaction นั้นใหม่อีกครั้ง

ตรวจสอบผลลัพธ์หลังเหตุการณ์:

```sql
SELECT customer_id, first_name, balance FROM customers ORDER BY customer_id;
```

จะเห็นว่ามีเพียง transaction เดียว (ผู้ชนะ) ที่ผลลัพธ์ถูกบันทึกจริง ส่วน transaction ที่เป็นเหยื่อจะถูก rollback ทั้งหมด ยอดเงินของมันจะไม่ถูกเปลี่ยนแปลงเลย (all-or-nothing ตามหลัก atomicity)

---

## Step 583: PostgreSQL Deadlock Detector — ทำงานอย่างไร

### `deadlock_timeout` คืออะไร

PostgreSQL **ไม่ได้ตรวจจับ deadlock ทันที** ที่เกิด circular wait เพราะการตรวจสอบวงจรการรอ (wait-for graph) มี**ค่าใช้จ่าย (overhead)** — ถ้าตรวจทุกครั้งที่มีการรอ lock จะทำให้ระบบช้าลงโดยไม่จำเป็น เพราะ lock wait ส่วนใหญ่เป็นเรื่องปกติ ไม่ใช่ deadlock

แทนที่จะเป็นแบบนั้น PostgreSQL ใช้กลยุทธ์นี้:

1. เมื่อ process ต้องรอ lock มันจะรอแบบปกติก่อน (เหมือน lock wait ทั่วไป)
2. ถ้ารอนานเกิน `deadlock_timeout` (ค่า default = `1s`) PostgreSQL จะเริ่ม**สร้าง wait-for graph** เพื่อตรวจสอบว่ามีวงจรการรอหรือไม่
3. ถ้าพบวงจร → เลือกเหยื่อ แล้ว ROLLBACK transaction นั้นด้วย error `deadlock detected`
4. ถ้าไม่พบวงจร (เป็นแค่ lock wait ปกติ) → process ยังคงรอต่อไปตามปกติ และจะ**ตรวจซ้ำทุก ๆ `deadlock_timeout`** จนกว่าจะได้ lock หรือพบวงจรจริง

ดูค่าปัจจุบัน:

```sql
SHOW deadlock_timeout;
--  deadlock_timeout
-- ------------------
--  1s
```

ปรับค่าได้ (ทั้ง session level และ system-wide ผ่าน `postgresql.conf`):

```sql
-- ปรับเฉพาะ session นี้ (ใช้ทดสอบ/debug)
SET deadlock_timeout = '500ms';

-- ปรับถาวรใน postgresql.conf แล้ว reload
-- deadlock_timeout = 1s
```

**ข้อควรพิจารณาในการปรับค่านี้:**

| ค่า `deadlock_timeout` | ผลกระทบ |
|---|---|
| ตั้งต่ำเกินไป (เช่น 100ms) | ตรวจจับ deadlock เร็วขึ้น แต่เพิ่มภาระ CPU จากการสร้าง wait-for graph บ่อยเกินจำเป็น โดยเฉพาะระบบที่มี lock wait ปกติเยอะอยู่แล้ว |
| ค่า default (1s) | สมดุลดีสำหรับงานส่วนใหญ่ |
| ตั้งสูงเกินไป (เช่น 10s) | แอปพลิเคชันที่ติด deadlock จริงจะต้องรอนานกว่าจะรู้ตัว ส่งผลต่อ user experience |

### การเลือก "เหยื่อ" (victim) ทำงานอย่างไร

เมื่อ PostgreSQL พบวงจรการรอจริง มันจะเลือก transaction หนึ่งในวงจรมาเป็นเหยื่อ ROLLBACK เกณฑ์ที่ PostgreSQL ใช้โดยประมาณ (อ้างอิงจาก `deadlock.c` ใน source code):

1. PostgreSQL สร้าง **wait-for graph** จาก process ทั้งหมดที่กำลังรอ lock
2. หาวงจร (cycle) ในกราฟนั้น
3. เมื่อพบวงจร จะพยายามเลือก process ที่ทำให้การ "คลาย" วงจรใช้ transaction น้อยที่สุดเท่าที่จะทำได้ (ไม่จำเป็นต้องเป็น transaction ที่ "อายุน้อยที่สุด" หรือ "ใหม่ที่สุด" เสมอไป — มันขึ้นกับโครงสร้างของกราฟการรอ ณ ขณะนั้น)
4. Process ที่ถูกเลือกจะได้รับ error `deadlock detected` และ transaction ของมันจะถูก ROLLBACK ทันที
5. Process ที่เหลือในวงจรจะได้ lock ต่อและทำงานต่อไปตามปกติ

**สิ่งสำคัญที่ต้องรู้:** แอปพลิเคชัน**ไม่สามารถกำหนดล่วงหน้าได้อย่างแม่นยำ 100%** ว่า session ไหนจะเป็นเหยื่อ ดังนั้นโค้ดฝั่งแอปพลิเคชันทุกส่วนที่อาจเกี่ยวข้องกับ deadlock **ต้องมี logic ดักจับ error และ retry เสมอ** ไม่ควรสมมติว่า transaction ของตัวเองจะ "รอด" ทุกครั้ง

ตัวอย่างการดูกระบวนการที่กำลังรอ lock กันแบบ real-time (รันใน session ที่สาม ขณะที่ A/B กำลังค้างกันอยู่ที่ T5-T6 จาก Step 582):

```sql
SELECT
    blocked.pid              AS blocked_pid,
    blocked.query            AS blocked_query,
    blocking.pid              AS blocking_pid,
    blocking.query           AS blocking_query
FROM pg_locks bl
JOIN pg_stat_activity blocked  ON bl.pid = blocked.pid
JOIN pg_locks kl ON kl.locktype = bl.locktype
    AND kl.database IS NOT DISTINCT FROM bl.database
    AND kl.relation IS NOT DISTINCT FROM bl.relation
    AND kl.page IS NOT DISTINCT FROM bl.page
    AND kl.tuple IS NOT DISTINCT FROM bl.tuple
    AND kl.transactionid IS NOT DISTINCT FROM bl.transactionid
    AND kl.pid != bl.pid
    AND kl.granted
JOIN pg_stat_activity blocking ON kl.pid = blocking.pid
WHERE NOT bl.granted;
```

หรือใช้ฟังก์ชันสำเร็จรูปที่อ่านง่ายกว่า:

```sql
SELECT pid, wait_event_type, wait_event, state, query
FROM pg_stat_activity
WHERE wait_event_type = 'Lock';
```

---

## Step 584: อ่านและตีความข้อความ `deadlock detected`

ข้อความ error deadlock มีโครงสร้างที่ให้ข้อมูลครบถ้วนพอจะสืบสวนได้ทันที มาดูตัวอย่างเต็มอีกครั้งพร้อมคำอธิบายทีละส่วน:

```
ERROR:  deadlock detected
DETAIL:  Process 48213 waits for ShareLock on transaction 892; blocked by process 48227.
Process 48227 waits for ShareLock on transaction 891; blocked by process 48213.
HINT:  See server log for query details.
CONTEXT:  while updating tuple (0,2) in relation "customers"
```

แยกส่วนอ่าน:

| ส่วนของข้อความ | ความหมาย |
|---|---|
| `ERROR: deadlock detected` | สรุปสั้น ๆ ว่าเกิด deadlock — transaction นี้ถูก ROLLBACK แล้ว |
| `Process 48213 waits for ShareLock on transaction 892` | process ที่ได้รับ error นี้ (48213) กำลังรอ lock ประเภท `ShareLock` บน **transaction ID 892** (ไม่ใช่รอ table หรือ row โดยตรง แต่รอให้ transaction 892 จบก่อน เพราะ PostgreSQL ใช้ transaction lock เพื่อบอกว่า "รอให้แถวที่ถูกแก้ไขนี้ commit/rollback ก่อน") |
| `blocked by process 48227` | transaction 892 นั้นคือของ process 48227 — พูดง่าย ๆ คือ 48213 กำลังรอ 48227 |
| `Process 48227 waits for ShareLock on transaction 891; blocked by process 48213.` | สลับกัน: 48227 กำลังรอ transaction 891 ซึ่งเป็นของ process 48213 — นี่คือจุดที่ทำให้เกิด**วงจร**: 48213 → รอ 48227 → รอ 48213 |
| `HINT: See server log for query details.` | บอกว่า error message นี้ไม่ได้แสดง SQL statement เต็ม ๆ ต้องไปดูใน server log (ต้องเปิด `log_lock_waits` หรือ log level ที่เหมาะสม — ดู Step 585) |
| `CONTEXT: while updating tuple (0,2) in relation "customers"` | บอกว่า error เกิดขึ้นขณะพยายาม UPDATE tuple ที่ตำแหน่ง physical `(0,2)` (page 0, tuple index 2) ในตาราง `customers` — นี่คือ "แถวที่กำลังจะถูกล็อก" ตอนที่ deadlock เกิดขึ้น |

### คำศัพท์สำคัญที่ต้องเข้าใจ

- **"holds lock"** — transaction ที่ถือ lock อยู่แล้ว (ได้ lock สำเร็จ)
- **"wants lock" / "waits for"** — transaction ที่กำลังพยายามขอ lock แต่ยังไม่ได้ (ถูกบล็อกอยู่)
- **"blocked by"** — ระบุตัวการที่ทำให้เกิดการรอ (คือ process/transaction ที่ถือ lock ตัวที่เรากำลังรออยู่)
- **ShareLock บน transaction ID** — นี่คือกลไกภายในของ PostgreSQL: เมื่อ transaction A แก้ไขแถวหนึ่ง แถวนั้นจะถูก "แท็ก" ด้วย transaction ID ของ A ถ้า transaction B ต้องการแก้ไขแถวเดียวกัน B จะต้อง "รอ transaction ID ของ A" ให้จบ (commit หรือ rollback) ก่อน ซึ่งจริง ๆ แล้วก็คือการรอ row lock นั่นเอง เพียงแต่แสดงผลผ่าน transaction ID

### ฝึกอ่านอีกตัวอย่างหนึ่ง (กรณี 3 transaction วนกัน)

```
ERROR:  deadlock detected
DETAIL:  Process 1001 waits for ShareLock on transaction 500; blocked by process 1002.
Process 1002 waits for ShareLock on transaction 501; blocked by process 1003.
Process 1003 waits for ShareLock on transaction 499; blocked by process 1001.
HINT:  See server log for query details.
```

วิเคราะห์: 1001 → รอ 1002 → รอ 1003 → รอ 1001 (วนกลับมาที่ตัวเอง) — เป็นวงจร 3 เส้นทาง ซึ่งแสดงให้เห็นว่า deadlock ไม่จำเป็นต้องเกิดแค่ 2 transaction เท่านั้น สามารถเกิดจากหลาย transaction ที่รอกันเป็นห่วงโซ่ได้เช่นกัน

---

## Step 585: Deadlock Log — `log_lock_waits` สืบสวน deadlock ใน production

ข้อความ error ที่ผู้ใช้เห็น (`ERROR: deadlock detected`) มักไม่มีรายละเอียด **SQL statement** ที่แต่ละ process กำลังรันอยู่ตอนเกิด deadlock — ซึ่งจำเป็นมากสำหรับการสืบสวนย้อนหลัง โชคดีที่ PostgreSQL มี parameter `log_lock_waits` ที่ช่วยได้

### เปิดใช้งาน `log_lock_waits`

```sql
SHOW log_lock_waits;
--  log_lock_waits
-- ----------------
--  off        -- ค่า default มักปิดอยู่

-- เปิดใช้งานระดับ session (ทดสอบ)
SET log_lock_waits = on;

-- เปิดใช้งานถาวรใน postgresql.conf แล้ว reload (แนะนำสำหรับ production)
-- log_lock_waits = on
```

เมื่อเปิด `log_lock_waits = on` PostgreSQL จะบันทึก log **ทุกครั้งที่ process ต้องรอ lock นานเกิน `deadlock_timeout`** (ไม่ใช่แค่ตอนเกิด deadlock จริง แต่รวมถึง lock wait ธรรมดาที่ยาวนานด้วย) ทำให้เห็นภาพรวมของปัญหา lock contention ในระบบ

### ตัวอย่าง log ที่จะเห็นเมื่อเกิด deadlock (จำลองจาก Step 582)

ในไฟล์ log ของ PostgreSQL (`postgresql.log` หรือปลายทางที่ตั้งค่าไว้) จะปรากฏประมาณนี้:

```
2026-09-25 10:15:42.104 +07 [48213] LOG:  process 48213 still waiting for ShareLock on transaction 892 after 1000.123 ms
2026-09-25 10:15:42.104 +07 [48213] DETAIL:  Process holding the lock: 48227. Wait queue: 48213.
2026-09-25 10:15:42.104 +07 [48213] CONTEXT:  while updating tuple (0,2) in relation "customers"
2026-09-25 10:15:42.104 +07 [48213] STATEMENT:  UPDATE customers SET balance = balance + 1000 WHERE customer_id = 2;

2026-09-25 10:15:42.106 +07 [48213] ERROR:  deadlock detected
2026-09-25 10:15:42.106 +07 [48213] DETAIL:  Process 48213 waits for ShareLock on transaction 892; blocked by process 48227.
	Process 48227 waits for ShareLock on transaction 891; blocked by process 48213.
	Process 48213: UPDATE customers SET balance = balance + 1000 WHERE customer_id = 2;
	Process 48227: UPDATE customers SET balance = balance + 500 WHERE customer_id = 1;
2026-09-25 10:15:42.106 +07 [48213] HINT:  See server log for query details.
2026-09-25 10:15:42.106 +07 [48213] CONTEXT:  while updating tuple (0,2) in relation "customers"
2026-09-25 10:15:42.106 +07 [48213] STATEMENT:  UPDATE customers SET balance = balance + 1000 WHERE customer_id = 2;
```

สังเกตว่า log ใน production (ที่ log level เหมาะสม เช่น มี `STATEMENT:` แสดงด้วย) จะ**บอก SQL statement เต็ม ๆ ของทั้งสอง process** ที่เกี่ยวข้อง — ข้อมูลนี้คือสิ่งที่ error message ธรรมดาที่ client เห็นไม่มี (client เห็นแค่ transaction ID) ทำให้เรารู้ทันทีว่าโค้ดส่วนไหนของแอปพลิเคชันที่ทำให้เกิดปัญหา

### Parameter อื่น ๆ ที่ควรพิจารณาคู่กัน

```sql
-- แสดง SQL statement ที่กำลังรันเมื่อ error เกิดขึ้น (ควรเปิดคู่กับ log_lock_waits)
SHOW log_min_error_statement;   -- ควรตั้งเป็น 'error' หรือต่ำกว่า เพื่อให้เห็น STATEMENT: ตอน error

-- ปรับ log_line_prefix ให้มีข้อมูล pid, timestamp, application_name (ช่วยไล่ log ง่ายขึ้น)
SHOW log_line_prefix;
-- ตัวอย่างค่าแนะนำ: '%m [%p] %q%u@%d '
```

### แนวทางตั้งค่าสำหรับ production

```ini
# postgresql.conf
log_lock_waits = on
deadlock_timeout = 1s
log_min_error_statement = error
log_line_prefix = '%m [%p] %q%u@%d '
```

จากนั้น `SELECT pg_reload_conf();` เพื่อให้ค่าที่แก้ไม่ต้อง restart server:

```sql
SELECT pg_reload_conf();
```

**เคล็ดลับสืบสวน production:** เมื่อพบ deadlock บ่อยครั้ง ให้กรอง log ด้วยคำว่า `deadlock detected` และดู `DETAIL:` ที่ตามมาเพื่อเทียบ pattern ของ SQL statement สองฝั่ง — ถ้าเห็นว่าเป็น UPDATE ตารางเดียวกันแต่เรียงลำดับ WHERE ต่างกัน (เช่น `customer_id = 1` ก่อน กับ `customer_id = 2` ก่อน) นั่นคือสัญญาณชัดเจนของปัญหา **inconsistent lock ordering** ซึ่งจะแก้ใน Step ถัดไป

---

## Step 586: การป้องกัน Deadlock ด้วย Consistent Lock Ordering

นี่คือ**หลักการป้องกัน deadlock ที่สำคัญที่สุด**และใช้ได้ผลเกือบทุกกรณี: **บังคับให้ทุก transaction เข้าถึง (ล็อก) ทรัพยากรตามลำดับคงที่เดียวกันเสมอ** ไม่ว่าจะเป็น transaction ประเภทไหนหรือถูกเรียกจากทิศทางไหน

### ทำไมการเรียงลำดับถึงป้องกัน deadlock ได้

ย้อนกลับไปดู Step 582: ปัญหาคือ Session A ล็อก `id=1` ก่อน `id=2` ในขณะที่ Session B ล็อก `id=2` ก่อน `id=1` — ทำให้เกิดวงจรได้ ถ้าเรา**บังคับทั้งสอง session ให้ล็อกจาก id น้อยไปมากเสมอ** วงจรจะเกิดขึ้นไม่ได้เลยในทางคณิตศาสตร์ เพราะไม่มีทางที่ transaction หนึ่งจะ "ย้อนกลับ" ไปรอ id ที่เล็กกว่าที่ตัวเองยังไม่ได้ล็อก

### แก้ตัวอย่าง Step 582 ด้วย Consistent Ordering

โค้ดเดิม (มีความเสี่ยง):

```sql
-- ผิด: ลำดับ WHERE ขึ้นกับว่าใครโอนให้ใคร
-- Session A: โอนจาก 1 ไป 2
UPDATE customers SET balance = balance - 1000 WHERE customer_id = 1;
UPDATE customers SET balance = balance + 1000 WHERE customer_id = 2;

-- Session B: โอนจาก 2 ไป 1
UPDATE customers SET balance = balance - 500 WHERE customer_id = 2;
UPDATE customers SET balance = balance + 500 WHERE customer_id = 1;
```

โค้ดที่แก้ไขแล้ว — **เรียง customer_id จากน้อยไปมากก่อนเสมอ ไม่ว่าเงินจะไหลทิศทางไหน**:

```sql
-- ============ Session A: โอนจาก customer_id=1 ไป customer_id=2 ============
-- เรียงแล้ว: 1 < 2 จึงล็อก 1 ก่อน (บังเอิญตรงกับลำดับ business logic เดิม)
BEGIN;
UPDATE customers SET balance = balance - 1000 WHERE customer_id = 1;  -- id น้อยกว่า ล็อกก่อน
UPDATE customers SET balance = balance + 1000 WHERE customer_id = 2;  -- id มากกว่า ล็อกทีหลัง
COMMIT;
```

```sql
-- ============ Session B: โอนจาก customer_id=2 ไป customer_id=1 ============
-- แม้ business logic จะ "โอนจาก 2 ไป 1" แต่การล็อกต้องเรียง id น้อยไปมากเสมอ
-- จึงต้องล็อก id=1 ก่อน (แม้จะเป็นฝั่งผู้รับเงินก็ตาม) แล้วค่อยล็อก id=2
BEGIN;
UPDATE customers SET balance = balance + 500 WHERE customer_id = 1;   -- id น้อยกว่า ล็อกก่อน (แม้จะเป็นการบวก)
UPDATE customers SET balance = balance - 500 WHERE customer_id = 2;   -- id มากกว่า ล็อกทีหลัง
COMMIT;
```

ตอนนี้ทั้ง Session A และ Session B ต่างก็ล็อก `customer_id = 1` ก่อนเสมอ ไม่ว่าเงินจะไหลทิศทางไหน — วงจรการรอจึงเป็นไปไม่ได้ในทางทฤษฎี ถ้าสอง session พยายามล็อก id=1 พร้อมกัน จะมีแค่ session เดียวที่ได้ก่อน อีก session รอเฉย ๆ (ไม่ใช่ deadlock แค่ lock wait ปกติ) แล้วเมื่อ session แรก COMMIT ก็จะปล่อยให้ session ที่สองไปต่อได้ทันที

### เขียนเป็นฟังก์ชันที่ใช้ได้ทั่วไป (Generalized Pattern)

ในทางปฏิบัติ เราไม่อยากให้ผู้พัฒนาต้อง "จำ" ที่จะเรียง id เองทุกครั้ง ควรห่อ logic นี้ไว้ในฟังก์ชันหรือ stored procedure กลาง:

```sql
CREATE OR REPLACE FUNCTION transfer_balance(
    p_from_id INTEGER,
    p_to_id   INTEGER,
    p_amount  NUMERIC
) RETURNS VOID AS $$
DECLARE
    v_first_id  INTEGER;
    v_second_id INTEGER;
BEGIN
    -- บังคับลำดับการล็อกเสมอ: id น้อยกว่าถูกแตะก่อน ไม่ว่าเงินจะไหลทิศทางไหน
    IF p_from_id < p_to_id THEN
        v_first_id  := p_from_id;
        v_second_id := p_to_id;
    ELSE
        v_first_id  := p_to_id;
        v_second_id := p_from_id;
    END IF;

    -- ล็อกแถวแรก (id น้อยกว่า) ก่อนเสมอ ด้วย SELECT ... FOR UPDATE
    PERFORM 1 FROM customers WHERE customer_id = v_first_id  FOR UPDATE;
    PERFORM 1 FROM customers WHERE customer_id = v_second_id FOR UPDATE;

    -- เมื่อจองแถวทั้งคู่แล้ว จึงค่อยทำ business logic จริง
    UPDATE customers SET balance = balance - p_amount WHERE customer_id = p_from_id;
    UPDATE customers SET balance = balance + p_amount WHERE customer_id = p_to_id;

    IF (SELECT balance FROM customers WHERE customer_id = p_from_id) < 0 THEN
        RAISE EXCEPTION 'ยอดเงินของลูกค้า % ไม่เพียงพอ', p_from_id;
    END IF;
END;
$$ LANGUAGE plpgsql;
```

ทดสอบ:

```sql
BEGIN;
SELECT transfer_balance(1, 2, 1000);  -- โอนจาก 1 ไป 2
COMMIT;

BEGIN;
SELECT transfer_balance(2, 1, 500);   -- โอนจาก 2 ไป 1
COMMIT;

SELECT customer_id, first_name, balance FROM customers ORDER BY customer_id;
```

ไม่ว่าจะเรียก `transfer_balance(1, 2, ...)` หรือ `transfer_balance(2, 1, ...)` พร้อมกันกี่ครั้งก็ตาม ฟังก์ชันนี้จะล็อก id น้อยกว่าก่อนเสมอ **การันตีว่าจะไม่เกิด deadlock จากคู่นี้อีกเลย**

### หลักการทั่วไป (ใช้ได้กับทุกกรณี ไม่จำกัดแค่ 2 แถว)

เมื่อ transaction ต้องล็อกหลายแถวหรือหลายตาราง ให้ยึดกฎ:

1. กำหนด **key เดียวที่ใช้เปรียบเทียบลำดับได้เสมอ** (เช่น primary key)
2. ก่อนเริ่ม UPDATE ให้ **เรียง (sort) รายการทรัพยากรที่จะล็อกตาม key นั้นก่อนเสมอ**
3. ล็อก/UPDATE ตามลำดับที่เรียงแล้ว ไม่ว่า business logic จะระบุลำดับมาแบบไหน

ตัวอย่างสำหรับกรณีล็อกหลายแถวพร้อมกัน (เช่น ตัดสต๊อกสินค้าหลาย SKU ในออเดอร์เดียว):

```sql
-- สมมติออเดอร์หนึ่งต้องตัดสต๊อกสินค้า product_id = 4, 2, 5 (ลำดับตามที่ผู้ใช้กดเลือกในตะกร้า)
-- ห้ามล็อกตามลำดับที่ผู้ใช้กด ต้องเรียงก่อนเสมอ
BEGIN;

SELECT product_id, stock_quantity
FROM products
WHERE product_id IN (4, 2, 5)
ORDER BY product_id          -- <-- กุญแจสำคัญ: เรียง id ก่อนล็อกเสมอ
FOR UPDATE;

UPDATE products SET stock_quantity = stock_quantity - 1 WHERE product_id = 2;
UPDATE products SET stock_quantity = stock_quantity - 3 WHERE product_id = 4;
UPDATE products SET stock_quantity = stock_quantity - 2 WHERE product_id = 5;

COMMIT;
```

การ `SELECT ... FOR UPDATE ... ORDER BY product_id` ล่วงหน้าแบบนี้ทำให้ไม่ว่าคำสั่ง UPDATE ด้านล่างจะเขียนเรียงลำดับอย่างไร แถวทั้งหมดก็ถูกจองไว้แล้วตามลำดับ id จากน้อยไปมากตั้งแต่ต้น (รายละเอียดเพิ่มเติมเรื่องนี้ใน Step 588)

---

## Step 587: การป้องกัน Deadlock ด้วยการลดขนาด Transaction ให้สั้นที่สุด

Consistent lock ordering (Step 586) แก้ปัญหาที่ **ต้นเหตุของ circular wait** แต่ยังมีอีกปัจจัยหนึ่งที่ทำให้ deadlock **เกิดบ่อยขึ้นหรือน้อยลง** นั่นคือ **ระยะเวลาที่ transaction ถือ lock ไว้** — ยิ่ง transaction เปิดค้างนาน โอกาสที่ transaction อื่นจะมาแย่งล็อกแถวเดียวกันในเวลาที่ทับซ้อนกันก็ยิ่งสูงขึ้น

### หลักการ: Transaction ควรสั้น กระชับ และไม่รอ I/O ภายนอก

ตัวอย่างโค้ดที่ **อันตรายมาก** (พบได้บ่อยในระบบจริง) คือการเปิด transaction แล้วรอ **user interaction** หรือเรียก API ภายนอกระหว่างทาง:

```sql
-- ตัวอย่างแย่มาก: อย่าทำแบบนี้
BEGIN;
UPDATE products SET stock_quantity = stock_quantity - 1 WHERE product_id = 2;
-- แอปพลิเคชันหยุดรอ: แสดงหน้ายืนยันให้ผู้ใช้กด "ยืนยันการชำระเงิน"
-- ผู้ใช้อาจใช้เวลา 10 วินาที, 30 วินาที, หรือปิดแท็บทิ้งเลยก็ได้!
-- ระหว่างนี้ row lock บน product_id=2 ยังคงถูกถือครองอยู่ตลอดเวลา
UPDATE orders SET status = 'confirmed' WHERE order_id = 1;
COMMIT;
```

ปัญหาของโค้ดนี้:

1. Row lock บน `product_id = 2` ถูกถือครองตลอดเวลาที่รอผู้ใช้ ทำให้ transaction อื่นที่ต้องการแก้ไขแถวเดียวกัน (เช่น ลูกค้าคนอื่นซื้อสินค้าเดียวกัน) ต้องรอนานผิดปกติ
2. ยิ่ง lock ถูกถือนาน ยิ่งมีโอกาสสูงที่จะมี transaction ที่สองเข้ามาถือ lock บนทรัพยากรอื่นแล้วย้อนมาต้องการทรัพยากรที่ transaction แรกถืออยู่ — เพิ่มโอกาสเกิด circular wait
3. ถ้าผู้ใช้ปิดแท็บ/เน็ตหลุด connection อาจค้างอยู่ (idle in transaction) จนกว่า timeout จะตัดการเชื่อมต่อ

### วิธีแก้: แยก "การรอผู้ใช้" ออกจาก transaction ฐานข้อมูล

```sql
-- ขั้นที่ 1: ตรวจสอบสต๊อกแบบ read-only (ไม่ล็อก ไม่เปิด transaction ยาว)
SELECT stock_quantity FROM products WHERE product_id = 2;
-- แอปพลิเคชันแสดงหน้ายืนยันให้ผู้ใช้ดู "มีสินค้า X ชิ้น พร้อมส่ง"
-- ผู้ใช้กดยืนยัน (ใช้เวลาเท่าไหร่ก็ได้ ไม่กระทบฐานข้อมูล เพราะยังไม่มี transaction เปิดค้าง)

-- ขั้นที่ 2: เมื่อผู้ใช้กดยืนยันแล้ว ค่อยเปิด transaction แบบสั้น กระชับ ไม่มีการรอ I/O ภายนอก
BEGIN;
UPDATE products SET stock_quantity = stock_quantity - 1
WHERE product_id = 2 AND stock_quantity > 0;   -- ตรวจซ้ำแบบ atomic ป้องกัน race condition

-- ตรวจว่า UPDATE สำเร็จจริงหรือไม่ (กรณีสต๊อกหมดพอดีระหว่างที่ผู้ใช้กำลังกดยืนยัน)
-- (แอปพลิเคชันเช็ค ROW_COUNT แล้วตัดสินใจ)

UPDATE orders SET status = 'confirmed' WHERE order_id = 1;
COMMIT;   -- ปิด transaction ทันทีหลังทำงานเสร็จ ไม่รอผู้ใช้
```

### หลักปฏิบัติ (Best Practices) สำหรับ transaction สั้น

| หลักการ | คำอธิบาย |
|---|---|
| **ห้ามรอ user interaction ระหว่าง transaction เปิดอยู่** | อย่าเปิด `BEGIN` แล้วรอให้ผู้ใช้กดปุ่ม พิมพ์ข้อความ หรือโหลดหน้าถัดไป |
| **ห้ามเรียก external API ระหว่าง transaction เปิดอยู่** | การเรียก payment gateway, ส่งอีเมล, เรียก third-party service ที่มี latency ไม่แน่นอน ควรทำ**ก่อน**หรือ**หลัง**transaction ไม่ใช่ระหว่างนั้น |
| **เตรียมข้อมูลทั้งหมดก่อนเปิด transaction** | คำนวณ, validate, query ข้อมูลอ้างอิงที่ไม่ต้องล็อกให้เสร็จก่อน แล้วค่อย BEGIN เมื่อพร้อมจะเขียนจริง ๆ |
| **ปิด transaction ทันทีที่ทำงานเสร็จ** | อย่าปล่อยให้ connection อยู่ในสถานะ `idle in transaction` — ใช้ `idle_in_transaction_session_timeout` เป็นตาข่ายนิรภัย |
| **แยก batch งานใหญ่เป็นชิ้นเล็ก** | แทนที่จะ UPDATE ทีเดียวหนึ่งล้านแถวใน transaction เดียว ให้แบ่งเป็น batch ละพันแถว COMMIT ทีละ batch |

### ตั้ง safety net ด้วย `idle_in_transaction_session_timeout`

```sql
SHOW idle_in_transaction_session_timeout;

-- ตั้งค่าระดับ session/database เพื่อป้องกัน connection ค้างในสถานะเปิด transaction เฉย ๆ
SET idle_in_transaction_session_timeout = '30s';

-- หรือปรับ config ระดับ database
ALTER DATABASE your_db SET idle_in_transaction_session_timeout = '30s';
```

เมื่อ connection ค้างในสถานะ `idle in transaction` เกินเวลาที่กำหนด PostgreSQL จะยกเลิก transaction นั้นให้อัตโนมัติ พร้อม log:

```
FATAL:  terminating connection due to idle-in-transaction timeout
```

พารามิเตอร์นี้เป็นเกราะป้องกันชั้นสุดท้ายเมื่อโค้ดแอปพลิเคชันมีบั๊กที่ลืมปิด transaction — มันไม่ได้ป้องกัน deadlock โดยตรง แต่ช่วยลด**เวลาที่ lock ถูกถือครองโดยไม่จำเป็น** ซึ่งลดโอกาสเกิด lock contention และ deadlock โดยรวม

---

## Step 588: ใช้ `SELECT ... FOR UPDATE` จองแถวล่วงหน้าตามลำดับที่แน่นอน

Step 586 แนะนำหลักการ consistent ordering แล้ว แต่ในทางปฏิบัติ เมื่อ transaction ต้องแตะหลายแถวพร้อมกัน (โดยเฉพาะเมื่อจำนวนแถวหรือรายการ id มาจาก input ของผู้ใช้ที่ไม่แน่นอน) วิธีที่ปลอดภัยและเป็นระบบที่สุดคือ **จองแถวทั้งหมดล่วงหน้าด้วย `SELECT ... FOR UPDATE ... ORDER BY`** ก่อนแล้วค่อยทำ UPDATE จริง

### ทำไมต้อง `FOR UPDATE` ล่วงหน้า

`SELECT ... FOR UPDATE` จะขอ **row-level exclusive lock** บนแถวที่ query คืนมา โดยไม่ต้องรอจนกว่าจะถึงคำสั่ง UPDATE จริง วิธีนี้ทำให้เรา**ควบคุมลำดับการล็อกได้อย่างชัดเจนในที่เดียว** (ที่ query `SELECT ... FOR UPDATE`) แทนที่จะกระจายอยู่ในหลายคำสั่ง UPDATE ที่อาจเขียนลำดับสลับกันโดยไม่ตั้งใจ

### ตัวอย่าง: ระบบโอนสต๊อกสินค้าระหว่าง SKU (คลังสินค้าภายใน)

สถานการณ์: มีฟีเจอร์ "รวม stock จาก SKU ที่ใกล้เลิกผลิตไปยัง SKU หลัก" ซึ่งต้องล็อกสินค้าหลายตัวพร้อมกัน และอาจถูกเรียกพร้อมกันจากหลาย process ด้วยชุด product_id ที่ทับซ้อนกันบางส่วน

```sql
CREATE OR REPLACE FUNCTION consolidate_stock(
    p_product_ids INTEGER[],   -- รายการ product_id ที่ต้องล็อก (ลำดับมาจาก input ผู้ใช้ ไม่แน่นอน)
    p_target_id   INTEGER,     -- SKU หลักที่จะรวม stock เข้าไป
    p_amounts     INTEGER[]    -- จำนวนที่จะย้ายจากแต่ละ SKU (ลำดับตรงกับ p_product_ids)
) RETURNS VOID AS $$
DECLARE
    v_locked_ids INTEGER[];
    v_id INTEGER;
    v_idx INTEGER;
BEGIN
    -- ขั้นที่ 1: รวม product_id ทั้งหมดที่เกี่ยวข้อง (รวมปลายทางด้วย) แล้วจองล็อกตามลำดับ id
    SELECT array_agg(product_id ORDER BY product_id)
    INTO v_locked_ids
    FROM products
    WHERE product_id = ANY (p_product_ids || p_target_id)
    FOR UPDATE;                          -- <-- จองทุกแถวที่เกี่ยวข้องล่วงหน้า เรียงตาม product_id เสมอ

    -- ขั้นที่ 2: เมื่อทุกแถวถูกจองแล้ว (การันตีลำดับคงที่) จึงค่อยทำ business logic จริง
    FOR v_idx IN 1 .. array_length(p_product_ids, 1) LOOP
        v_id := p_product_ids[v_idx];

        UPDATE products
        SET stock_quantity = stock_quantity - p_amounts[v_idx]
        WHERE product_id = v_id AND stock_quantity >= p_amounts[v_idx];

        IF NOT FOUND THEN
            RAISE EXCEPTION 'สินค้า product_id=% มีสต๊อกไม่พอสำหรับย้าย % ชิ้น', v_id, p_amounts[v_idx];
        END IF;

        UPDATE products
        SET stock_quantity = stock_quantity + p_amounts[v_idx]
        WHERE product_id = p_target_id;
    END LOOP;
END;
$$ LANGUAGE plpgsql;
```

ทดสอบ (จำลองสอง process เรียกพร้อมกันด้วยชุด id ที่ทับซ้อนกันบางส่วน):

```sql
-- Session A: รวม product_id 3, 5 เข้า SKU หลัก 1
BEGIN;
SELECT consolidate_stock(ARRAY[3,5], 1, ARRAY[10,5]);
COMMIT;
```

```sql
-- Session B: รวม product_id 5, 3 เข้า SKU หลัก 2 (ลำดับ input สลับกับ A แต่ผลลัพธ์การล็อกจะเหมือนกันเพราะ ORDER BY)
BEGIN;
SELECT consolidate_stock(ARRAY[5,3], 2, ARRAY[2,4]);
COMMIT;
```

แม้ Session A ส่ง `ARRAY[3,5]` และ Session B ส่ง `ARRAY[5,3]` (คนละลำดับ) แต่เพราะฟังก์ชันมี `SELECT ... FOR UPDATE ... ORDER BY product_id` ทั้งคู่จะพยายามล็อกแถว id=1 (target ปนกันไปด้วย), 3, 5 **ตามลำดับ id จากน้อยไปมากเหมือนกันเสมอ** — ไม่มีทางเกิด circular wait

### เปรียบเทียบ: จองล่วงหน้า (ปลอดภัย) กับ ไม่จอง (เสี่ยง)

```sql
-- ❌ เสี่ยง: UPDATE ตรง ๆ ตามลำดับ input โดยไม่จองล่วงหน้า
-- ถ้า input มาจากผู้ใช้/ตะกร้าสินค้า ลำดับไม่แน่นอน มีโอกาสสูงที่จะชนกันเป็น deadlock
FOREACH v_id IN ARRAY p_product_ids LOOP
    UPDATE products SET stock_quantity = stock_quantity - 1 WHERE product_id = v_id;
END LOOP;
```

```sql
-- ✅ ปลอดภัย: จองแถวทั้งหมดล่วงหน้าตามลำดับคงที่ก่อน แล้วค่อยวนลูป UPDATE
SELECT 1 FROM products
WHERE product_id = ANY(p_product_ids)
ORDER BY product_id
FOR UPDATE;

FOREACH v_id IN ARRAY p_product_ids LOOP
    UPDATE products SET stock_quantity = stock_quantity - 1 WHERE product_id = v_id;
END LOOP;
```

### ข้อควรระวังเรื่อง `FOR UPDATE` กับ `ORDER BY`

`SELECT ... FOR UPDATE` จะล็อกแถวตามลำดับที่ **executor ประมวลผลจริง** ไม่ใช่ลำดับที่ query วางแผนไว้เสมอไป การใส่ `ORDER BY` ใน query ที่มี `FOR UPDATE` **ช่วยให้ optimizer เลือก sort ก่อนแล้วค่อยล็อกตามลำดับนั้น** ในทางปฏิบัติ PostgreSQL จะ sort ผลลัพธ์ก่อนแล้วค่อยขอ lock ทีละแถวตามลำดับที่ sort แล้ว ทำให้พฤติกรรมสอดคล้องกับที่เราต้องการ — แต่ควรทดสอบด้วย `EXPLAIN (ANALYZE)` เพื่อยืนยัน plan จริงในกรณีที่ query ซับซ้อนขึ้น:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT product_id FROM products
WHERE product_id = ANY(ARRAY[5,3,1])
ORDER BY product_id
FOR UPDATE;
```

---

## Step 589: Lock Timeout — `statement_timeout` และ `lock_timeout`

Consistent ordering (Step 586) และ short transaction (Step 587) ป้องกัน deadlock ได้ดีมาก แต่ในระบบจริงที่ซับซ้อน (โดยเฉพาะเมื่อมีโค้ด legacy หรือหลายทีมเขียนโค้ดที่แตะตารางเดียวกัน) เราอาจ**ไม่สามารถการันตีได้ 100%** ว่าจะไม่มีทาง deadlock เกิดขึ้นเลย ดังนั้นควรมี **มาตรการสำรอง (safety net)** เพื่อไม่ให้ transaction ค้างรอ lock นานเกินไปโดยไม่จำเป็น — นั่นคือ `lock_timeout` และ `statement_timeout`

### `lock_timeout` คืออะไร

`lock_timeout` กำหนดระยะเวลาสูงสุดที่ statement หนึ่งจะรอ lock ได้ ถ้าเกินเวลานี้ statement จะถูกยกเลิกด้วย error ทันที **โดยไม่ต้องรอ deadlock detector** เลย (เพราะบางครั้งปัญหาไม่ใช่ deadlock จริง แค่ lock contention ปกติที่รอนานเกินไป)

```sql
SHOW lock_timeout;
--  lock_timeout
-- --------------
--  0        -- ค่า default = 0 หมายถึง "ไม่มี timeout รอได้ไม่จำกัดเวลา"

-- ตั้งค่าระดับ session
SET lock_timeout = '3s';
```

ทดสอบ:

```sql
-- Session A
BEGIN;
UPDATE customers SET balance = balance - 100 WHERE customer_id = 1;
-- ยังไม่ COMMIT ค้างไว้แบบนี้
```

```sql
-- Session B
SET lock_timeout = '3s';
BEGIN;
UPDATE customers SET balance = balance + 100 WHERE customer_id = 1;
-- จะรอ 3 วินาที แล้วได้ error:
```

```
ERROR:  canceling statement due to lock timeout
```

Session B จะได้ error นี้ทันทีหลังผ่านไป 3 วินาที **โดยไม่ต้องรอให้เกิด deadlock จริง** ต่างจาก deadlock detector ที่ทำงานเฉพาะเมื่อมีวงจรการรอเท่านั้น — `lock_timeout` ทำงานแม้ในกรณี lock wait ปกติที่ไม่ใช่ deadlock เลย (เช่น session A แค่ค้าง transaction ไว้นาน ๆ โดยไม่มี circular wait)

### `statement_timeout` คืออะไร

`statement_timeout` กำหนดเวลาสูงสุดที่ **statement หนึ่งคำสั่งใดๆ** (ไม่ใช่แค่การรอ lock) จะได้รับอนุญาตให้รันได้ ครอบคลุมทั้งเวลาประมวลผล query และเวลารอ lock ด้วย

```sql
SHOW statement_timeout;
--  statement_timeout
-- -------------------
--  0

SET statement_timeout = '10s';
```

ถ้า statement รันเกิน 10 วินาที (ไม่ว่าจะติดขัดที่ lock หรือ query ช้าเอง) จะได้:

```
ERROR:  canceling statement due to statement timeout
```

### เปรียบเทียบ `lock_timeout` vs `statement_timeout`

| | `lock_timeout` | `statement_timeout` |
|---|---|---|
| ครอบคลุมอะไร | เฉพาะเวลาที่**รอ lock** เท่านั้น | เวลารันทั้งหมดของ statement (รวมทั้งประมวลผลและรอ lock) |
| Error message | `canceling statement due to lock timeout` | `canceling statement due to statement timeout` |
| ใช้เมื่อไหร่ | ต้องการจำกัดเฉพาะการรอ lock โดยไม่กระทบ query ที่ประมวลผลนาน (เช่น report query) | ต้องการเพดานเวลารวมของทุก statement แบบเข้มงวด |
| ค่า default | `0` (ไม่จำกัด) | `0` (ไม่จำกัด) |

### ตั้งค่าแนะนำสำหรับระบบ OLTP (เช่น อีคอมเมิร์ซ)

```sql
-- ตั้งระดับ role สำหรับ application user (แนะนำมากกว่าตั้ง global เพราะงาน batch/report อาจต้องการเวลานานกว่า)
ALTER ROLE app_user SET lock_timeout = '3s';
ALTER ROLE app_user SET statement_timeout = '15s';

-- หรือตั้งระดับ database
ALTER DATABASE ecommerce_db SET lock_timeout = '3s';
```

### จุดสำคัญ: `lock_timeout` ไม่ใช่การป้องกัน deadlock แต่เป็นการจำกัดความเสียหาย

ต้องเข้าใจให้ชัดว่า:

- **Deadlock detector** (Step 583) ทำงานเฉพาะเมื่อเกิด **circular wait จริง** และจะ ROLLBACK เหยื่อโดยอัตโนมัติเสมอ (ไม่ว่าจะตั้ง `lock_timeout` หรือไม่)
- **`lock_timeout`** ทำงานกับ **ทุกกรณีที่รอ lock นานเกินกำหนด** ไม่ว่าจะเป็น deadlock จริงหรือแค่ lock contention ปกติ — มันเป็นกลไกอิสระที่ **ทำงานเร็วกว่า** deadlock detector ได้ (ถ้าตั้งค่าต่ำกว่า `deadlock_timeout`) และช่วยจำกัดผลกระทบของ lock contention ที่ยืดเยื้อ (ป้องกันไม่ให้ผู้ใช้ต้องรอนานเกินไปโดยไม่จำเป็น แม้จะไม่ใช่ deadlock ก็ตาม)
- ทั้งสองกลไกควรใช้**ร่วมกัน**: consistent ordering ป้องกัน deadlock จากรากเหง้า, deadlock detector เป็นตาข่ายสุดท้ายสำหรับกรณีที่หลุดรอด, และ `lock_timeout`/`statement_timeout` ป้องกันไม่ให้ผู้ใช้รอนานเกินไปในทุกสถานการณ์ที่เกี่ยวกับ lock

---

## Step 590: แบบฝึกหัดรวม — วิเคราะห์และแก้ไขโค้ดเสี่ยง Deadlock

สถานการณ์: ระบบมีฟีเจอร์ "โอนสต๊อกสินค้าระหว่างคำสั่งซื้อ" (ย้ายสินค้าจากออเดอร์ที่ยกเลิกไปเติมให้ออเดอร์ที่รอสินค้า) เขียนเป็นฟังก์ชันดังนี้:

```sql
-- ⚠️ โค้ดนี้มีความเสี่ยง deadlock — วิเคราะห์และแก้ไขก่อนนำไปใช้จริง
CREATE OR REPLACE FUNCTION move_stock_between_products(
    p_source_id INTEGER,
    p_dest_id   INTEGER,
    p_qty       INTEGER
) RETURNS VOID AS $$
BEGIN
    UPDATE products SET stock_quantity = stock_quantity - p_qty WHERE product_id = p_source_id;
    -- แอปพลิเคชันเรียก external logging API ตรงนี้เพื่อบันทึกการโอน (ใช้เวลาไม่แน่นอน)
    -- PERFORM external_log_transfer(p_source_id, p_dest_id, p_qty);
    UPDATE products SET stock_quantity = stock_quantity + p_qty WHERE product_id = p_dest_id;
END;
$$ LANGUAGE plpgsql;
```

เรียกใช้งานจากสองจุดในระบบพร้อมกัน:

```sql
-- Client A (โอนจากสินค้า 4 ไป 2)
BEGIN;
SELECT move_stock_between_products(4, 2, 5);
COMMIT;

-- Client B (โอนจากสินค้า 2 ไป 4) — เกิดขึ้นพร้อมกันโดยบังเอิญ
BEGIN;
SELECT move_stock_between_products(2, 4, 3);
COMMIT;
```

### โจทย์

1. อธิบายว่าทำไมโค้ดนี้เสี่ยงเกิด deadlock (ระบุจุดที่ล็อกและลำดับการล็อกของ Client A กับ Client B)
2. โค้ดนี้มีปัญหาอีกอย่างหนึ่งที่ไม่เกี่ยวกับ deadlock โดยตรง แต่ทำให้ transaction เปิดค้างนานเกินจำเป็น — คืออะไร?
3. แก้ไขฟังก์ชันให้ปลอดภัยจาก deadlock ด้วยหลัก consistent lock ordering (Step 586 + 588) และย้าย logging ออกจาก transaction (Step 587)
4. เพิ่ม `lock_timeout` เป็นเกราะป้องกันชั้นสุดท้าย (Step 589)

<details>
<summary><strong>เฉลยแบบฝึกหัดรวม (คลิกเพื่อดู)</strong></summary>

**ข้อ 1 — วิเคราะห์จุดเสี่ยง:**

Client A เรียก `move_stock_between_products(4, 2, 5)`:
- ล็อก `product_id = 4` ก่อน (UPDATE แรก)
- ล็อก `product_id = 2` ทีหลัง (UPDATE ที่สอง)

Client B เรียก `move_stock_between_products(2, 4, 3)`:
- ล็อก `product_id = 2` ก่อน (UPDATE แรก)
- ล็อก `product_id = 4` ทีหลัง (UPDATE ที่สอง)

ถ้าทั้งสอง Client เริ่ม UPDATE แรกพร้อมกัน (Client A ล็อก id=4 สำเร็จ, Client B ล็อก id=2 สำเร็จ) แล้วต่างฝ่ายต่างพยายามทำ UPDATE ที่สอง (Client A ต้องการ id=2 ซึ่ง B ถืออยู่, Client B ต้องการ id=4 ซึ่ง A ถืออยู่) — เกิด **circular wait ทันที** เป็น deadlock แบบคลาสสิกเหมือน Step 582 ทุกประการ เพียงแค่เปลี่ยนจากตาราง `customers` มาเป็น `products`

**ข้อ 2 — ปัญหาอื่นที่ไม่เกี่ยวกับ deadlock โดยตรง:**

คอมเมนต์ `PERFORM external_log_transfer(...)` แสดงว่ามีการเรียก **external API ระหว่าง transaction เปิดอยู่** (ระหว่าง UPDATE แรกกับ UPDATE ที่สอง) — ตามหลักการ Step 587 นี่คือความเสี่ยงร้ายแรง เพราะ:
- ถ้า external API ช้า (network latency, timeout) transaction จะถือ lock บน `product_id = p_source_id` ค้างไว้นานเกินจำเป็น
- ยิ่งถือ lock นาน ยิ่งเพิ่ม**หน้าต่างเวลา (window)** ที่อาจเกิด circular wait กับ transaction อื่น เพิ่มโอกาส deadlock โดยรวมของทั้งระบบ (แม้จะแก้ปัญหาข้อ 1 ด้วย consistent ordering แล้ว การถือ lock นานก็ยังเพิ่ม lock contention โดยทั่วไป)

**ข้อ 3 — แก้ไขด้วย consistent lock ordering + ย้าย logging ออกจาก transaction:**

```sql
CREATE OR REPLACE FUNCTION move_stock_between_products(
    p_source_id INTEGER,
    p_dest_id   INTEGER,
    p_qty       INTEGER
) RETURNS VOID AS $$
DECLARE
    v_first_id  INTEGER;
    v_second_id INTEGER;
BEGIN
    -- (1) กำหนดเวลารอ lock สูงสุด เป็นเกราะป้องกันชั้นสุดท้าย
    SET LOCAL lock_timeout = '3s';

    -- (2) บังคับลำดับการล็อก: id น้อยกว่าเสมอถูกจองก่อน ไม่ว่าเงิน/สต๊อกจะไหลทิศทางไหน
    IF p_source_id < p_dest_id THEN
        v_first_id  := p_source_id;
        v_second_id := p_dest_id;
    ELSE
        v_first_id  := p_dest_id;
        v_second_id := p_source_id;
    END IF;

    -- (3) จองแถวทั้งสองล่วงหน้าตามลำดับคงที่ ด้วย SELECT ... FOR UPDATE
    PERFORM 1 FROM products WHERE product_id = v_first_id  FOR UPDATE;
    PERFORM 1 FROM products WHERE product_id = v_second_id FOR UPDATE;

    -- (4) ตรวจสต๊อกให้เพียงพอก่อนตัดจริง (ป้องกันค่าติดลบ)
    IF (SELECT stock_quantity FROM products WHERE product_id = p_source_id) < p_qty THEN
        RAISE EXCEPTION 'สินค้า product_id=% มีสต๊อกไม่พอ (ต้องการ % ชิ้น)', p_source_id, p_qty;
    END IF;

    -- (5) ทำ business logic จริง (สั้น กระชับ ไม่มีการรอ I/O ภายนอกใด ๆ)
    UPDATE products SET stock_quantity = stock_quantity - p_qty WHERE product_id = p_source_id;
    UPDATE products SET stock_quantity = stock_quantity + p_qty WHERE product_id = p_dest_id;
END;
$$ LANGUAGE plpgsql;

-- การเรียก external_log_transfer() ต้องย้ายออกไปทำ "หลัง COMMIT" ในโค้ดแอปพลิเคชัน
-- ไม่ใช่อยู่ในฟังก์ชัน/transaction นี้อีกต่อไป:
--
--   BEGIN;
--   SELECT move_stock_between_products(4, 2, 5);
--   COMMIT;
--   -- แอปพลิเคชันเรียก external_log_transfer(4, 2, 5) ที่นี่ (หลัง COMMIT สำเร็จแล้ว)
```

**ข้อ 4 — ทดสอบว่าแก้ปัญหาได้จริง:**

```sql
-- เรียกพร้อมกันจากสอง session ด้วยทิศทางสลับกัน เหมือนสถานการณ์ตั้งต้น
-- Session A
BEGIN;
SELECT move_stock_between_products(4, 2, 5);
COMMIT;

-- Session B
BEGIN;
SELECT move_stock_between_products(2, 4, 3);
COMMIT;
```

ตอนนี้ทั้งสอง session จะพยายามล็อก `product_id = 2` ก่อนเสมอ (เพราะ 2 < 4) แล้วค่อยล็อก `product_id = 4` — ถ้าเรียกพร้อมกัน จะมีแค่ session เดียวที่ได้ล็อก id=2 ก่อน อีก session **รอเฉย ๆ แบบ lock wait ปกติ** (ไม่ใช่ circular wait) และถ้าเผื่อว่ามีเหตุผิดปกติอื่นทำให้รอนานเกิน 3 วินาที `lock_timeout` ที่ตั้งด้วย `SET LOCAL` จะตัดจบด้วย error `canceling statement due to lock timeout` แทนที่จะปล่อยให้ค้างไม่จำกัดเวลา

ตรวจสอบผลลัพธ์:

```sql
SELECT product_id, product_name, stock_quantity FROM products ORDER BY product_id;
```

</details>

---

## สรุปท้ายบท

- **Deadlock** คือวงจรการรอ (circular wait) ระหว่างสอง transaction ขึ้นไป ที่แต่ละฝ่ายถือ lock อยู่และรอ lock ที่อีกฝ่ายถือ — ไม่มีทางคลี่คลายได้เองถ้าไม่มีการแทรกแซง
- PostgreSQL มี **deadlock detector** ในตัว ทำงานหลังจากรอ lock นานเกิน `deadlock_timeout` (ค่า default 1 วินาที) โดยจะสร้าง wait-for graph ตรวจหาวงจร แล้วเลือก transaction หนึ่งเป็น "เหยื่อ" มา ROLLBACK ด้วย error `deadlock detected`
- ข้อความ error deadlock บอกได้ครบว่า process ไหนรออะไร ถูกบล็อกโดยใคร — ถ้าต้องการเห็น SQL statement เต็ม ๆ ต้องเปิด `log_lock_waits = on` และดูใน server log
- วิธีป้องกัน deadlock ที่ได้ผลที่สุดคือ **consistent lock ordering** — บังคับให้ทุก transaction เข้าถึงทรัพยากรตามลำดับคงที่เดียวกันเสมอ (เช่น เรียงตาม primary key จากน้อยไปมาก) ไม่ว่า business logic จะระบุทิศทางแบบไหน
- ควรลด**ขนาดและระยะเวลา**ของ transaction ให้สั้นที่สุด หลีกเลี่ยงการรอ user interaction หรือเรียก external API ระหว่าง transaction เปิดอยู่ และใช้ `idle_in_transaction_session_timeout` เป็นตาข่ายนิรภัย
- `SELECT ... FOR UPDATE ... ORDER BY` เป็นเครื่องมือสำคัญในการจองแถวหลายแถวตามลำดับที่แน่นอนล่วงหน้า ก่อนเริ่ม UPDATE จริง
- `lock_timeout` และ `statement_timeout` เป็นมาตรการสำรอง (safety net) ที่ตัดจบการรอ lock ที่นานเกินไป **แม้ในกรณีที่ไม่ใช่ deadlock จริง** ควรใช้ร่วมกับ consistent ordering เสมอ ไม่ใช่ใช้แทนกัน
- deadlock เป็นเรื่องปกติที่จัดการได้เมื่อออกแบบโค้ดถูกต้อง แต่แอปพลิเคชันทุกส่วนที่แก้ไขข้อมูลพร้อมกันได้ **ควรมี logic ดักจับ error `deadlock detected` และ retry เสมอ** เป็นแนวปฏิบัติมาตรฐาน

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1

จากตาราง `customers` และ `products` ที่เตรียมไว้ตอนต้นบท จงอธิบายด้วยคำพูดของตัวเองว่าทำไม deadlock ถึง **ไม่ใช่** บั๊กของ PostgreSQL แต่เป็นผลจากการออกแบบโค้ดแอปพลิเคชัน

<details>
<summary>เฉลย</summary>

PostgreSQL ทำหน้าที่ตามหลัก ACID คือรับประกันว่าแต่ละ transaction จะเห็นและแก้ไขข้อมูลอย่างถูกต้องแบบ isolated จากกัน การล็อกแถวเมื่อมีการ UPDATE เป็นกลไกที่จำเป็นเพื่อป้องกัน race condition ซึ่งเป็นพฤติกรรมที่ถูกต้องและจำเป็น deadlock เกิดขึ้นเมื่อ**โค้ดแอปพลิเคชัน**เขียนลำดับการเข้าถึง (ล็อก) ทรัพยากรที่**ไม่สอดคล้องกัน**ระหว่างสอง transaction — ถ้าโค้ดถูกออกแบบให้ล็อกตามลำดับคงที่เสมอ (consistent ordering) deadlock จากคู่ทรัพยากรนั้นจะไม่มีทางเกิดขึ้นได้เลยในทางคณิตศาสตร์ PostgreSQL เพียงแค่มีกลไก deadlock detector ไว้เป็น "ตาข่ายนิรภัย" เผื่อกรณีที่โค้ดควบคุมลำดับไม่ได้ครบถ้วน ไม่ใช่สาเหตุของปัญหา
</details>

---

### แบบฝึกหัดที่ 2

จำลอง deadlock ด้วยตาราง `products` (แทนที่จะเป็น `customers`) โดยใช้ product_id = 1 และ 3 เขียนโค้ด SQL สำหรับ Session A และ Session B ที่ทำให้เกิด deadlock จากการ UPDATE `stock_quantity` สลับลำดับกัน

<details>
<summary>เฉลย</summary>

```sql
-- Session A: ลดสต๊อกสินค้า 1 ก่อน แล้วค่อยสินค้า 3
BEGIN;
UPDATE products SET stock_quantity = stock_quantity - 5 WHERE product_id = 1;
-- รอสักครู่
UPDATE products SET stock_quantity = stock_quantity - 3 WHERE product_id = 3;
COMMIT;
```

```sql
-- Session B: ลดสต๊อกสินค้า 3 ก่อน แล้วค่อยสินค้า 1
BEGIN;
UPDATE products SET stock_quantity = stock_quantity - 2 WHERE product_id = 3;
-- รอสักครู่
UPDATE products SET stock_quantity = stock_quantity - 1 WHERE product_id = 1;
COMMIT;
```

เมื่อรันสลับกันตามจังหวะที่ทั้งคู่ทำ UPDATE แรกสำเร็จแล้วพยายามทำ UPDATE ที่สองพร้อมกัน จะเกิด circular wait และได้ error `deadlock detected` ในหนึ่งใน session ทั้งสองหลังผ่านไปประมาณ `deadlock_timeout` (ค่า default 1 วินาที)
</details>

---

### แบบฝึกหัดที่ 3

`DETAIL` ในข้อความ error ต่อไปนี้บอกอะไรบ้าง? จงระบุว่า process ใดถือ lock อะไร และ process ใดกำลังรออะไร

```
DETAIL:  Process 5001 waits for ShareLock on transaction 700; blocked by process 5002.
Process 5002 waits for ShareLock on transaction 699; blocked by process 5001.
```

<details>
<summary>เฉลย</summary>

- Process 5001 กำลังรอ ShareLock บน transaction 700 ซึ่งเป็นของ process 5002 — หมายความว่า process 5002 ถือ lock บนแถวที่ 5001 ต้องการอยู่
- Process 5002 กำลังรอ ShareLock บน transaction 699 ซึ่งเป็นของ process 5001 — หมายความว่า process 5001 ถือ lock บนแถวที่ 5002 ต้องการอยู่

เกิดวงจร: 5001 รอ 5002 → 5002 รอ 5001 → วนกลับมาที่ 5001 นี่คือ circular wait แบบคลาสสิกระหว่างสอง process ทำให้ deadlock detector ต้องเลือกหนึ่งในสองมาเป็นเหยื่อ
</details>

---

### แบบฝึกหัดที่ 4

เปิด `log_lock_waits = on` แล้วอธิบายว่า parameter นี้ช่วยอะไรที่ error message ปกติ (ที่ client เห็น) ไม่สามารถให้ได้

<details>
<summary>เฉลย</summary>

Error message ปกติที่ client เห็น (เช่น `ERROR: deadlock detected` พร้อม `DETAIL`) จะบอกแค่ **process ID และ transaction ID** ที่เกี่ยวข้อง แต่ไม่ได้แสดง SQL statement จริงที่แต่ละ process กำลังรันอยู่ตอนเกิดปัญหา เมื่อเปิด `log_lock_waits = on` PostgreSQL จะบันทึกลง server log ทุกครั้งที่มีการรอ lock นานเกิน `deadlock_timeout` พร้อมกับ `STATEMENT:` ที่แสดง SQL เต็ม ๆ ของ process ที่เกี่ยวข้องทั้งหมด ทำให้ทีม DBA/นักพัฒนาสามารถสืบสวนย้อนหลังได้ว่าโค้ดส่วนไหนของแอปพลิเคชันที่ทำให้เกิด deadlock หรือ lock contention ที่ยืดเยื้อ โดยไม่ต้องอาศัยการเดา
</details>

---

### แบบฝึกหัดที่ 5

จงเขียนฟังก์ชัน `place_order_with_stock_check(p_customer_id INTEGER, p_product_ids INTEGER[], p_quantities INTEGER[])` ที่รับรายการสินค้าหลายชิ้นพร้อมจำนวน แล้วตัดสต๊อกทีละชิ้นอย่างปลอดภัยจาก deadlock โดยใช้หลัก consistent lock ordering

<details>
<summary>เฉลย</summary>

```sql
CREATE OR REPLACE FUNCTION place_order_with_stock_check(
    p_customer_id  INTEGER,
    p_product_ids  INTEGER[],
    p_quantities   INTEGER[]
) RETURNS INTEGER AS $$
DECLARE
    v_new_order_id INTEGER;
    v_idx          INTEGER;
    v_product_id   INTEGER;
BEGIN
    SET LOCAL lock_timeout = '3s';

    -- จองแถวสินค้าทั้งหมดล่วงหน้า เรียงตาม product_id เสมอ ไม่ว่า input จะมาลำดับไหน
    PERFORM 1
    FROM products
    WHERE product_id = ANY(p_product_ids)
    ORDER BY product_id
    FOR UPDATE;

    -- ตรวจสต๊อกให้พอทุกชิ้นก่อนตัดจริง
    FOR v_idx IN 1 .. array_length(p_product_ids, 1) LOOP
        v_product_id := p_product_ids[v_idx];
        IF (SELECT stock_quantity FROM products WHERE product_id = v_product_id) < p_quantities[v_idx] THEN
            RAISE EXCEPTION 'สินค้า product_id=% สต๊อกไม่พอ', v_product_id;
        END IF;
    END LOOP;

    -- ตัดสต๊อกจริง
    FOR v_idx IN 1 .. array_length(p_product_ids, 1) LOOP
        UPDATE products
        SET stock_quantity = stock_quantity - p_quantities[v_idx]
        WHERE product_id = p_product_ids[v_idx];
    END LOOP;

    INSERT INTO orders (customer_id, status) VALUES (p_customer_id, 'paid')
    RETURNING order_id INTO v_new_order_id;

    RETURN v_new_order_id;
END;
$$ LANGUAGE plpgsql;
```

จุดสำคัญ: การใช้ `ORDER BY product_id` ใน query `FOR UPDATE` ทำให้ไม่ว่า `p_product_ids` จะถูกส่งมาด้วยลำดับใด แถวจะถูกจองตามลำดับ id จากน้อยไปมากเสมอ — ป้องกัน deadlock ระหว่างการเรียกฟังก์ชันนี้พร้อมกันหลายครั้งด้วยชุดสินค้าที่ทับซ้อนกัน
</details>

---

### แบบฝึกหัดที่ 6

`deadlock_timeout` กับ `lock_timeout` ต่างกันอย่างไร? ถ้าตั้ง `lock_timeout = 500ms` และ `deadlock_timeout = 1s` (ค่า default) จะเกิดอะไรขึ้นเมื่อ transaction ติด circular wait จริง?

<details>
<summary>เฉลย</summary>

- `deadlock_timeout` คือเวลาที่ PostgreSQL รอก่อนจะเริ่ม**ตรวจสอบว่ามีวงจรการรอ (circular wait) หรือไม่** ถ้าตรวจแล้วพบวงจรจริง จะ ROLLBACK เหยื่อด้วย error `deadlock detected`
- `lock_timeout` คือเวลาสูงสุดที่ statement หนึ่งจะรอ lock ได้ ไม่ว่าจะเป็น deadlock จริงหรือแค่ lock wait ปกติ ถ้าเกินเวลานี้จะถูกยกเลิกด้วย error `canceling statement due to lock timeout` ทันที โดยไม่ต้องรอตรวจสอบวงจรเลย

ถ้าตั้ง `lock_timeout = 500ms` ซึ่ง**น้อยกว่า** `deadlock_timeout = 1s` แล้วเกิด circular wait จริง — statement ที่รออยู่จะถูกยกเลิกด้วย `canceling statement due to lock timeout` ที่ 500ms **ก่อน**ที่ deadlock detector จะได้มีโอกาสทำงานที่ 1 วินาทีเสียอีก ผลลัพธ์สุดท้ายคล้ายกัน (transaction หนึ่งถูกยกเลิก อีกฝ่ายได้ไปต่อ) แต่ error message ที่ได้จะต่างกัน และการยกเลิกจะเร็วกว่า
</details>

---

### แบบฝึกหัดที่ 7

โค้ดต่อไปนี้มีความเสี่ยงอะไรบ้างเกี่ยวกับ deadlock และ lock contention (ไม่ใช่แค่ประเด็นเดียว)?

```sql
BEGIN;
UPDATE customers SET balance = balance - 200 WHERE customer_id = 3;
SELECT pg_sleep(15);  -- จำลองการรอ external payment gateway callback
UPDATE orders SET status = 'paid' WHERE order_id = 3;
COMMIT;
```

<details>
<summary>เฉลย</summary>

1. **Transaction เปิดค้างนานเกินจำเป็น (Step 587):** `pg_sleep(15)` จำลองการรอ external service ระหว่าง transaction เปิดอยู่ ทำให้ row lock บน `customer_id = 3` ถูกถือครองนานถึง 15 วินาทีโดยไม่จำเป็น เพิ่มโอกาส lock contention กับ transaction อื่นที่ต้องการแก้ไขลูกค้าคนเดียวกัน
2. **เพิ่มโอกาส deadlock โดยรวม:** ยิ่งถือ lock นาน ยิ่งมีโอกาสที่ transaction อื่นจะเข้ามาถือ lock บนทรัพยากรอื่นแล้วย้อนมาต้องการ `customer_id = 3` พร้อมกับที่ transaction นี้ต้องการทรัพยากรที่อีกฝ่ายถืออยู่ — เพิ่มหน้าต่างเวลาที่อาจเกิด circular wait
3. **ควรย้าย external interaction ออกจาก transaction:** การเรียก/รอ payment gateway ควรทำก่อนเปิด transaction (ตรวจสอบผลลัพธ์จาก callback ก่อน) แล้วค่อยเปิด transaction แบบสั้นเพื่อบันทึกผลลัพธ์เท่านั้น
4. หากไม่มี `idle_in_transaction_session_timeout` ตั้งไว้ และ connection หลุดระหว่างรอ 15 วินาที transaction อาจค้างอยู่ในสถานะ `idle in transaction` นานเกินควร

วิธีแก้: เรียก/รอผลจาก payment gateway **ก่อน** เปิด transaction แล้วเมื่อได้ผลลัพธ์ (สำเร็จ/ไม่สำเร็จ) แล้วค่อย `BEGIN` เพื่อบันทึกผลลัพธ์แบบสั้นและเร็วที่สุด
</details>

---

### แบบฝึกหัดที่ 8

จงอธิบายว่าทำไมการเพิ่ม `ORDER BY product_id` ใน `SELECT ... FOR UPDATE` ถึงช่วยป้องกัน deadlock ได้ ทั้งที่ query ไม่มี `ORDER BY` ก็ยังคืนแถวเดียวกันอยู่ดี

<details>
<summary>เฉลย</summary>

ถ้าไม่มี `ORDER BY` PostgreSQL executor อาจ**ประมวลผลและขอ lock แถวตามลำดับที่ query planner เลือก** ซึ่งไม่รับประกันว่าจะเป็นลำดับเดียวกันทุกครั้ง (อาจขึ้นกับ index scan order, sequential scan order, หรือ plan ที่เปลี่ยนไปตามสถิติของตาราง) ทำให้สอง transaction ที่เรียก query เดียวกันด้วยชุด id เดียวกันอาจล็อกแถวคนละลำดับกันได้ในบางกรณี

การใส่ `ORDER BY product_id` บังคับให้ PostgreSQL **sort ผลลัพธ์ตาม product_id ก่อน แล้วค่อยขอ lock ทีละแถวตามลำดับที่ sort แล้ว** ทำให้ทุก transaction ที่รัน query แบบเดียวกัน (แม้จะส่ง input เป็นชุด id คนละลำดับ) จะล็อกแถวจากน้อยไปมากเหมือนกันเสมอ — นี่คือสิ่งที่การันตี consistent lock ordering ได้อย่างแท้จริง ไม่ใช่แค่ "หวังว่า" executor จะเลือกลำดับที่เราต้องการเอง
</details>

---

### แบบฝึกหัดที่ 9

ถ้าแอปพลิเคชันของคุณเจอ error `deadlock detected` เป็นครั้งคราว (ไม่บ่อยมาก) ในสภาพแวดล้อม production ควรจัดการอย่างไรในระดับโค้ดแอปพลิเคชัน โดยไม่ต้องรื้อ schema หรือ business logic ใหม่ทั้งหมด?

<details>
<summary>เฉลย</summary>

1. **เพิ่ม retry logic** ในชั้นแอปพลิเคชัน: เมื่อได้รับ error ที่มี SQLSTATE `40P01` (deadlock_detected) ให้ ROLLBACK แล้วลองรัน transaction เดิมซ้ำใหม่ (มักใช้ exponential backoff เล็กน้อยเพื่อลดโอกาสชนซ้ำ) เช่น ลองใหม่สูงสุด 3 ครั้งก่อนจะแจ้ง error ให้ผู้ใช้จริง ๆ
2. **เปิด `log_lock_waits = on`** ใน production เพื่อเก็บ log รายละเอียดไว้สืบสวนว่า deadlock เกิดจากคู่ query ไหนบ้าง
3. **วิเคราะห์ pattern จาก log** ว่า deadlock เกิดซ้ำ ๆ ระหว่างคู่ query ใด แล้วค่อยไปแก้ไขเฉพาะจุดนั้นด้วย consistent lock ordering (ไม่จำเป็นต้องรื้อทั้งระบบ แก้เฉพาะฟังก์ชัน/endpoint ที่มีปัญหาจริง)
4. **ตั้ง `lock_timeout`** เป็นค่าที่เหมาะสม เพื่อไม่ให้ผู้ใช้ต้องรอนานเกินไปแม้ในกรณีที่ไม่ใช่ deadlock

ตัวอย่าง SQLSTATE code สำหรับดักจับใน PL/pgSQL:

```sql
BEGIN
    -- business logic ที่เสี่ยง deadlock
EXCEPTION
    WHEN deadlock_detected THEN
        RAISE NOTICE 'พบ deadlock กำลังลองใหม่...';
        -- แอปพลิเคชันชั้นบน (เช่น Python/Java) ควรเป็นผู้ retry transaction ทั้งหมดใหม่
        -- ไม่ใช่ retry แค่ภายในฟังก์ชันเดียว เพราะ transaction context เปลี่ยนไปแล้ว
END;
```
</details>

---

### แบบฝึกหัดที่ 10

จงออกแบบ (เขียน SQL) การทดสอบอัตโนมัติแบบง่าย ๆ ที่ยืนยันว่าฟังก์ชัน `transfer_balance` จาก Step 586 **ไม่เกิด deadlock** อีกต่อไป แม้จะเรียกพร้อมกันด้วยทิศทางสลับกันซ้ำหลายรอบ (อธิบายแนวคิดการทดสอบ ไม่ต้องรันจริงก็ได้)

<details>
<summary>เฉลย</summary>

แนวคิดการทดสอบ:

1. เปิดสอง connection (เช่นผ่านสอง psql session หรือสองเธรดในโปรแกรมทดสอบ)
2. ให้แต่ละ connection เรียก `transfer_balance` สลับทิศทางกันซ้ำ ๆ หลายรอบ (เช่น 100 รอบ) พร้อมกันอย่างต่อเนื่อง เพื่อเพิ่มโอกาสที่จะชนกันถ้ายังมีบั๊ก:

```sql
-- Connection 1 (ในลูปโปรแกรมทดสอบ วนซ้ำ 100 ครั้ง)
BEGIN;
SELECT transfer_balance(1, 2, 10);
COMMIT;

-- Connection 2 (พร้อมกัน วนซ้ำ 100 ครั้ง)
BEGIN;
SELECT transfer_balance(2, 1, 10);
COMMIT;
```

3. ตรวจสอบว่าไม่มี connection ใดได้รับ error `deadlock detected` เลยตลอด 100 รอบ (ถ้ายังเจอ แสดงว่า consistent ordering ยังไม่ถูกนำไปใช้ครบทุกจุดในฟังก์ชัน)
4. ตรวจสอบความถูกต้องของข้อมูลหลังทดสอบ: ผลรวม (`SUM(balance)`) ของลูกค้าทั้งสองคนต้อง**เท่าเดิม**ก่อนและหลังการทดสอบ (เพราะเป็นการโอนเงินไปมา ไม่มีเงินหายหรืองอกขึ้นมา):

```sql
SELECT SUM(balance) FROM customers WHERE customer_id IN (1, 2);
-- ค่าต้องเท่ากับก่อนเริ่มทดสอบเป๊ะ ไม่ว่าจะรันไปกี่รอบก็ตาม
```

การทดสอบแบบนี้เรียกว่า **stress test สำหรับ concurrency correctness** ซึ่งเป็นวิธีมาตรฐานในการยืนยันว่าโค้ดที่แก้ปัญหา deadlock แล้วนั้นทำงานถูกต้องจริงภายใต้ภาระการใช้งานพร้อมกันสูง ไม่ใช่แค่ "ดูโค้ดแล้วคิดว่าน่าจะโอเค"
</details>

---

**บทถัดไป:** [Part 060 — VACUUM และ Autovacuum](./part-060-vacuum-autovacuum.md)
