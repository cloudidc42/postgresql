# MVCC เชิงลึก: Locking และ Concurrency Control

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับสูง | Part 058

---

## เป้าหมายการเรียนรู้

เมื่อจบบทนี้ ผู้เรียนจะสามารถ:

1. อธิบายได้ว่า MVCC (Multi-Version Concurrency Control) คืออะไร และทำไม PostgreSQL จึงเลือกใช้แนวทางนี้แทนการ lock ทุกแถวแบบ traditional locking systems
2. เข้าใจความหมายของ system column `xmin` และ `xmax` ที่ผูกอยู่กับทุกแถวในตาราง และใช้มันตรวจสอบ "เวอร์ชัน" ของแถวได้
3. อธิบายได้ว่าทำไม `UPDATE` ใน PostgreSQL จึงไม่ได้แก้ไขแถวเดิม แต่เป็นการสร้างแถวใหม่ (dead tuple) และเข้าใจผลกระทบต่อ table bloat
4. แยกแยะ row-level lock ทั้ง 4 แบบ (`FOR UPDATE`, `FOR NO KEY UPDATE`, `FOR SHARE`, `FOR KEY SHARE`) และเลือกใช้ให้เหมาะกับสถานการณ์
5. เขียน `SELECT ... FOR UPDATE` เพื่อป้องกัน race condition ในสถานการณ์จริง เช่น การจองสินค้า/ที่นั่ง
6. ใช้ `NOWAIT` และ `SKIP LOCKED` เพื่อออกแบบระบบที่ไม่ต้องรอ lock แบบไม่มีที่สิ้นสุด รวมถึงสร้าง job queue pattern
7. เข้าใจ table-level lock modes ทั้ง 8 ระดับ และตาราง lock compatibility matrix
8. รู้ว่าคำสั่ง SQL แต่ละแบบ (SELECT, UPDATE, ALTER TABLE, DROP TABLE, VACUUM ฯลฯ) ขอ lock ระดับใดโดยปริยาย และผลกระทบต่อระบบ production
9. ใช้ Advisory Lock (`pg_advisory_lock`) สำหรับสร้าง application-level lock ที่ไม่ผูกกับแถวหรือตารางใด ๆ
10. ออกแบบระบบจองคิว/จองสินค้าที่ปลอดภัยจาก race condition โดยผสมผสานเทคนิค locking ที่เหมาะสม

**ความสัมพันธ์กับบทอื่น:** บทนี้เป็นการเจาะลึกต่อจาก Part 037 (Transactions และ ACID) และ Part 038 (Isolation Levels) ที่ได้แนะนำ MVCC และ Isolation Level แบบภาพรวมไปแล้ว ในบทนี้เราจะลงลึกถึงกลไกภายในของการ lock ระดับแถวและระดับตาราง ส่วนรายละเอียดเชิงลึกของ `xmin`/`xmax`, transaction ID wraparound และ VACUUM internals จะถูกอธิบายอย่างละเอียดใน Part 084 และบทที่เกี่ยวกับ Vacuum โดยเฉพาะ ส่วน deadlock detection และการแก้ปัญหา deadlock จะอยู่ใน [Part 059](./part-059-deadlocks.md)

---

## เตรียมข้อมูล

เราจะใช้ schema อีคอมเมิร์ซพื้นฐานตลอดทั้งบท เพื่อจำลองสถานการณ์การจองสินค้า (`stock_quantity`) และการอัปเดตสถานะคำสั่งซื้อ (`status`) ซึ่งเป็นจุดที่เกิด concurrency conflict บ่อยที่สุดในระบบจริง

```sql
-- ล้างของเดิมถ้ามี (สำหรับรันซ้ำในบทเรียน)
DROP TABLE IF EXISTS orders;
DROP TABLE IF EXISTS products;

CREATE TABLE products (
    product_id      SERIAL PRIMARY KEY,
    product_name    VARCHAR(150) NOT NULL,
    unit_price      NUMERIC(10,2) NOT NULL,
    stock_quantity  INTEGER NOT NULL DEFAULT 0
);

CREATE TABLE orders (
    order_id     SERIAL PRIMARY KEY,
    customer_id  INTEGER,
    order_date   TIMESTAMPTZ NOT NULL DEFAULT now(),
    status       VARCHAR(20) NOT NULL DEFAULT 'pending'
);
```

ข้อมูลตัวอย่าง (seed data) ที่สมจริงพอสำหรับสาธิต concurrency:

```sql
INSERT INTO products (product_name, unit_price, stock_quantity) VALUES
    ('เสื้อยืดคอกลม สีขาว ไซส์ M',        199.00,  50),
    ('เสื้อยืดคอกลม สีดำ ไซส์ L',         199.00,  35),
    ('กางเกงยีนส์ขาตรง',                 890.00,  20),
    ('รองเท้าผ้าใบ รุ่น Runner Pro',     2490.00,   8),
    ('กระเป๋าเป้ Laptop 15 นิ้ว',        1290.00,  15),
    ('หูฟังบลูทูธ ANC รุ่น SoundMax',    3590.00,   5),
    ('นาฬิกาข้อมือสมาร์ทวอทช์ Gen 3',    4990.00,   3),
    ('พาวเวอร์แบงค์ 20000mAh',            690.00,  60),
    ('เคสโทรศัพท์กันกระแทก',              250.00, 120),
    ('ตั๋วคอนเสิร์ต โซน A (จำนวนจำกัด)', 3500.00,   1);

INSERT INTO orders (customer_id, order_date, status) VALUES
    (101, now() - interval '10 days', 'completed'),
    (102, now() - interval '9 days',  'completed'),
    (103, now() - interval '7 days',  'cancelled'),
    (104, now() - interval '5 days',  'completed'),
    (105, now() - interval '3 days',  'shipped'),
    (101, now() - interval '2 days',  'pending'),
    (106, now() - interval '1 days',  'pending'),
    (107, now(),                      'pending');
```

> **หมายเหตุการทดลอง:** ตลอดบทนี้เราจะเปิด **สองเซสชัน (Session A และ Session B)** พร้อมกัน เช่น เปิด `psql` สองหน้าต่าง หรือใช้ pgAdmin/DBeaver สองแท็บ query แล้ว copy คำสั่งไปรันตามลำดับที่กำกับไว้ในตัวอย่าง (A1 → B1 → A2 → B2 ...) เพื่อดูพฤติกรรมการ block/lock จริง

---

## Step 571: ทบทวน MVCC คืออะไร และทำไม PostgreSQL เลือกใช้แนวทางนี้

### แนวคิดของระบบ locking แบบดั้งเดิม (2PL)

ฐานข้อมูลรุ่นเก่าจำนวนมากใช้แนวทาง **Two-Phase Locking (2PL)** เพียงอย่างเดียว: ทุกครั้งที่มีการอ่านแถว (`SELECT`) ระบบจะขอ **shared lock**, และทุกครั้งที่มีการเขียน (`UPDATE`/`DELETE`) จะขอ **exclusive lock** ปัญหาคือ:

- Reader บล็อก Writer และ Writer บล็อก Reader — ถ้ามี transaction หนึ่งกำลังอ่านตารางขนาดใหญ่ transaction ที่ต้องการ `UPDATE` แถวเดียวกันจะต้องรอ
- ระบบที่มี read-heavy workload (เช่น รายงาน, dashboard) จะทำให้ transaction เขียนข้อมูล (เช่น การสั่งซื้อ) ช้าลงอย่างมาก

### แนวทางของ PostgreSQL: MVCC

PostgreSQL ใช้ **Multi-Version Concurrency Control (MVCC)** ซึ่งมีหลักการสำคัญคือ:

> **"ผู้อ่านไม่บล็อกผู้เขียน และผู้เขียนไม่บล็อกผู้อ่าน"** (Readers never block writers, writers never block readers)

หลักการทำงานคร่าว ๆ:

1. ทุกแถวในตารางไม่ได้มีอยู่แค่ "เวอร์ชันเดียว" — เมื่อมีการ `UPDATE` หรือ `DELETE`, PostgreSQL จะไม่ทำลายข้อมูลเดิมทันที แต่จะสร้าง **snapshot ของข้อมูลตามช่วงเวลา** ขึ้นมาแทน
2. แต่ละ transaction จะเห็นข้อมูล ณ "จุดเวลา" ของ snapshot ที่ตัวเองเริ่มต้น (ขึ้นกับ isolation level ที่ใช้ — ทบทวนได้จาก Part 038)
3. เมื่อ transaction A กำลังแก้ไขแถวหนึ่งอยู่ (ยังไม่ commit) transaction B ที่มาอ่านแถวเดียวกันจะยังคง**เห็นข้อมูลเวอร์ชันเก่า** ได้โดยไม่ต้องรอ A commit ก่อน — นี่คือเหตุผลที่ `SELECT` ธรรมดาไม่เคยถูก block โดย `UPDATE`

ลองดูตัวอย่างง่าย ๆ ที่แสดงให้เห็นว่า `SELECT` ไม่ถูกบล็อกแม้จะมี transaction อื่นกำลัง `UPDATE` แถวเดียวกันอยู่:

```sql
-- ===== Session A =====
BEGIN;
UPDATE products SET stock_quantity = stock_quantity - 1
WHERE product_id = 4;
-- ยังไม่ COMMIT — จงใจค้าง transaction ไว้
```

```sql
-- ===== Session B (รันขณะที่ Session A ยังไม่ COMMIT) =====
SELECT product_id, product_name, stock_quantity
FROM products
WHERE product_id = 4;
-- ผลลัพธ์: ได้ค่าทันที ไม่ต้องรอ! เห็นค่าเดิมก่อน UPDATE (stock_quantity = 8)
```

```sql
-- ===== Session A =====
COMMIT;
```

```sql
-- ===== Session B (รันหลัง Session A COMMIT แล้ว) =====
SELECT product_id, product_name, stock_quantity
FROM products
WHERE product_id = 4;
-- ผลลัพธ์: stock_quantity = 7 (เห็นเวอร์ชันใหม่)
```

จะเห็นว่า Session B **ไม่เคยถูก block เลย** ไม่ว่า Session A จะ `UPDATE` ค้างไว้นานแค่ไหน — นี่คือหัวใจของ MVCC

### ข้อดีและข้อเสียของ MVCC

| ข้อดี | ข้อเสีย/สิ่งที่ต้องระวัง |
|---|---|
| Reader ไม่ถูก writer บล็อก → throughput สูงขึ้นมากในระบบที่มีการอ่าน-เขียนพร้อมกัน | ต้องเก็บหลายเวอร์ชันของแถวไว้ในดิสก์ ทำให้เกิด "dead tuple" และ table bloat (ดู Step 573) |
| Writer ไม่ถูก reader บล็อก | ต้องมีกระบวนการ `VACUUM` มาเก็บกวาดแถวเก่าที่ไม่มีใครใช้แล้ว (รายละเอียดในบทเรื่อง VACUUM) |
| รองรับ isolation level ต่าง ๆ (Read Committed, Repeatable Read, Serializable) ได้อย่างมีประสิทธิภาพ | Transaction ID (XID) เป็นทรัพยากรจำกัด ต้องระวังปัญหา "transaction ID wraparound" (Part 084) |
| ไม่ต้องพึ่ง lock manager หนัก ๆ สำหรับ read-only queries | Writer ยังคงต้องแย่ง lock กันเองเมื่อแก้ไขแถวเดียวกัน (เนื้อหาหลักของบทนี้) |

**สรุป:** MVCC แก้ปัญหา read-write conflict ได้ดีมาก แต่ **write-write conflict** (สอง transaction พยายามแก้ไขแถวเดียวกันพร้อมกัน) ยังคงต้องใช้กลไก **row-level locking** อยู่ดี — ซึ่งเป็นหัวข้อหลักที่เราจะเจาะลึกในบทนี้

---

## Step 572: xmin/xmax เบื้องต้น — ทุกแถวมี "เวอร์ชัน" ที่ผูกกับ Transaction ID

PostgreSQL เก็บข้อมูล metadata ไว้ในทุกแถวโดยไม่ต้องประกาศ column เพิ่ม เรียกว่า **system columns** ที่สำคัญที่สุดสำหรับ MVCC คือ:

| System Column | ความหมาย |
|---|---|
| `xmin` | Transaction ID ของ transaction ที่ **สร้าง** แถวนี้ (INSERT หรือ UPDATE ที่สร้างเวอร์ชันใหม่) |
| `xmax` | Transaction ID ของ transaction ที่ **ลบ/แทนที่** แถวนี้ (DELETE หรือ UPDATE ที่ทำให้เวอร์ชันนี้กลายเป็นเวอร์ชันเก่า) — ถ้าเป็น `0` แปลว่าแถวนี้ยังไม่ถูกลบ/แทนที่ (ยัง "มีชีวิต" อยู่) |
| `ctid` | ตำแหน่งทางกายภาพของแถวใน heap (page number, offset) — เปลี่ยนทุกครั้งที่มีการสร้างเวอร์ชันใหม่ |

ลองดูค่าจริง:

```sql
SELECT xmin, xmax, ctid, product_id, product_name, stock_quantity
FROM products
WHERE product_id = 1;
```

ผลลัพธ์ตัวอย่าง (ค่า `xmin` จะแตกต่างกันไปตามจำนวน transaction ที่เคยรันบนเครื่องของแต่ละคน):

```
 xmin | xmax | ctid  | product_id |       product_name        | stock_quantity
------+------+-------+------------+----------------------------+----------------
  742 |    0 | (0,1) |          1 | เสื้อยืดคอกลม สีขาว ไซส์ M |             50
```

- `xmin = 742` หมายถึงแถวนี้ถูกสร้างโดย transaction หมายเลข 742 (คำสั่ง `INSERT` ตอนเตรียมข้อมูล)
- `xmax = 0` หมายถึงยังไม่มีใครลบหรือแทนที่แถวนี้ — นี่คือเวอร์ชัน**ล่าสุด**ที่ยัง active อยู่

ทีนี้ลอง `UPDATE` แล้วดูว่าเกิดอะไรขึ้น:

```sql
UPDATE products SET stock_quantity = 49 WHERE product_id = 1;

SELECT xmin, xmax, ctid, product_id, stock_quantity
FROM products
WHERE product_id = 1;
```

```
 xmin | xmax | ctid  | product_id | stock_quantity
------+------+-------+------------+----------------
  918 |    0 | (0,11)|          1 |             49
```

สังเกตว่า:
- `xmin` เปลี่ยนเป็น `918` (transaction ID ของ `UPDATE` ที่เพิ่งรัน) — เพราะแถวนี้เป็น**แถวใหม่**ที่ถูกสร้างขึ้น ไม่ใช่การแก้ไขแถวเดิม
- `ctid` เปลี่ยนจาก `(0,1)` เป็น `(0,11)` — ตำแหน่งทางกายภาพเปลี่ยนไป เพราะเป็นแถวคนละแถวกันจริง ๆ ในระดับ storage

คำถามคือ แล้วแถวเก่า (`xmin=742, stock_quantity=50`) หายไปไหน? คำตอบคือมันยังอยู่ในดิสก์ แต่ **`xmax` ของมันถูกตั้งเป็น 918** (transaction ที่ทำให้มันกลายเป็นแถวเก่า) — นี่คือสิ่งที่เรียกว่า **dead tuple** ซึ่งจะอธิบายละเอียดใน Step 573

ฟังก์ชันช่วยเสริมที่มีประโยชน์: ใช้ `txid_current()` เพื่อดู transaction ID ของ transaction ปัจจุบัน (หมายเหตุ: PostgreSQL 13+ แนะนำให้ใช้ `pg_current_xact_id()` แทน):

```sql
SELECT pg_current_xact_id();
```

> **หมายเหตุสำคัญ:** `xmin`/`xmax` ที่เราเห็นในบทนี้เป็นเพียงการแนะนำเบื้องต้นเพื่อให้เข้าใจธรรมชาติของ MVCC เท่านั้น รายละเอียดเชิงลึก เช่น การเปรียบเทียบ transaction ID แบบ modulo-2^32, hint bits, freeze map, และปัญหา transaction ID wraparound จะอธิบายอย่างละเอียดใน **Part 084**

---

## Step 573: แถวเก่าไม่ถูกลบทันที (Dead Tuple) — ทำไม UPDATE คือ INSERT ใหม่ + Mark เก่าเป็น Dead

จาก Step 572 เราเห็นแล้วว่า `UPDATE` ไม่ได้แก้ไข "ที่เดิม" (in-place) แต่ทำสองอย่าง:

1. **สร้างแถวใหม่** (new tuple) ที่มีข้อมูลอัปเดตแล้ว พร้อม `xmin` ใหม่
2. **ทำเครื่องหมายแถวเก่า** ว่า "ตายแล้ว" โดยตั้งค่า `xmax` ให้เท่ากับ transaction ที่ทำ UPDATE — แต่**ไม่ลบข้อมูลจริงออกจากดิสก์ทันที**

เหตุผลที่ทำเช่นนี้คือ MVCC ต้องการเก็บเวอร์ชันเก่าไว้ชั่วคราว เผื่อมี transaction อื่นที่ snapshot ยังต้องเห็นข้อมูลเวอร์ชันเก่าอยู่ (เช่น transaction ที่เริ่มก่อนและยังไม่จบ ภายใต้ Repeatable Read/Serializable)

### พิสูจน์ด้วย pg_stat_user_tables

PostgreSQL เก็บสถิติจำนวน "dead tuple" ต่อตารางไว้ที่ `pg_stat_user_tables`:

```sql
SELECT relname, n_live_tup, n_dead_tup
FROM pg_stat_user_tables
WHERE relname = 'products';
```

```
 relname  | n_live_tup | n_dead_tup
----------+------------+------------
 products |         10 |          0
```

ลอง `UPDATE` ทุกแถวในตาราง `products` แล้วดูอีกครั้ง:

```sql
UPDATE products SET stock_quantity = stock_quantity;  -- UPDATE ทุกแถว (ไม่เปลี่ยนค่าจริง แต่ยังนับเป็น UPDATE)

SELECT relname, n_live_tup, n_dead_tup
FROM pg_stat_user_tables
WHERE relname = 'products';
```

```
 relname  | n_live_tup | n_dead_tup
----------+------------+------------
 products |         10 |         10
```

ตอนนี้มี **10 dead tuples** ที่เกิดจากการ `UPDATE` ทั้งที่ตารางมีแค่ 10 แถว — นี่คือปรากฏการณ์ **table bloat**: ตารางใช้พื้นที่ดิสก์มากกว่าที่ข้อมูลจริงต้องการ เพราะมีเวอร์ชันเก่าค้างอยู่

### ใครมาเก็บกวาด dead tuple?

กระบวนการ **`VACUUM`** (ทั้งแบบ manual และ **autovacuum** ที่รันอัตโนมัติ) มีหน้าที่:

- ตรวจสอบว่า dead tuple แถวไหน **ไม่มี transaction ใดในระบบต้องใช้แล้ว** (ไม่มี snapshot ไหนอ้างอิงถึง)
- ทำเครื่องหมายพื้นที่นั้นว่า "ใช้ซ้ำได้" (ไม่ได้คืนพื้นที่ให้ OS ทันทีเว้นแต่ใช้ `VACUUM FULL`)

```sql
VACUUM (VERBOSE, ANALYZE) products;
```

```
INFO:  vacuuming "public.products"
INFO:  "products": found 10 removable, 10 nonremovable row versions in 1 out of 1 pages
...
```

หลัง `VACUUM` แล้ว `n_dead_tup` จะกลับมาเป็น 0 (แต่พื้นที่ดิสก์ของตารางจะไม่หดตัวลง — จะถูกทำเครื่องหมายให้ transaction ในอนาคตนำมาใช้ซ้ำได้แทน)

```sql
SELECT relname, n_live_tup, n_dead_tup
FROM pg_stat_user_tables
WHERE relname = 'products';
```

```
 relname  | n_live_tup | n_dead_tup
----------+------------+------------
 products |         10 |          0
```

> **ทำไมเรื่องนี้สำคัญต่อ Concurrency Control?** เพราะยิ่งมี transaction ที่เปิดค้างไว้นาน (long-running transaction) ยิ่งทำให้ `VACUUM` ไม่สามารถเก็บกวาด dead tuple ได้ (เพราะ transaction เก่ายังอาจต้องใช้ snapshot นั้นอยู่) ส่งผลให้ตาราง bloat มากขึ้นเรื่อย ๆ และ index scan ช้าลง — นี่คือเหตุผลสำคัญข้อหนึ่งที่ต้องจำกัดเวลาที่ transaction เปิดค้าง โดยเฉพาะ transaction ที่ถือ lock ไว้ด้วย (หัวข้อถัดไป)

---

## Step 574: Row-level Lock ประเภทต่าง ๆ

เมื่อสอง transaction ต้องการแก้ไขแถวเดียวกันพร้อมกัน MVCC เพียงอย่างเดียวไม่พอ — PostgreSQL ต้องใช้ **row-level lock** เพื่อป้องกัน write-write conflict การ lock ประเภทนี้มี 4 ระดับ (เรียงจากอ่อนสุดไปแรงสุด):

| Lock Mode | ใช้เมื่อ | บล็อกอะไร |
|---|---|---|
| `FOR KEY SHARE` | ต้องการอ้างอิงแถว (เช่น กำลังจะสร้าง FK ที่ชี้มาที่แถวนี้) แต่ไม่แก้ไขค่าใด ๆ | บล็อกเฉพาะ `FOR UPDATE` และการแก้ไขที่เปลี่ยน key column เท่านั้น |
| `FOR SHARE` | ต้องการอ่านแถวและมั่นใจว่าจะไม่ถูกแก้ไข/ลบระหว่างที่ transaction ยังไม่จบ (เช่น อ่านค่าไปคำนวณต่อ) | บล็อก `FOR UPDATE`, `FOR NO KEY UPDATE` และการ `UPDATE`/`DELETE` |
| `FOR NO KEY UPDATE` | จะ `UPDATE` แถวแต่ **ไม่แตะ column ที่เป็น primary key/unique key ที่มี FK อ้างอิงถึง** | บล็อก `FOR UPDATE`, `FOR SHARE` (แต่ไม่บล็อก `FOR KEY SHARE`) |
| `FOR UPDATE` | จะ `UPDATE` หรือ `DELETE` แถวนี้แน่นอน ต้องการ lock แบบเข้มที่สุด | บล็อกทุก lock mode ข้างต้น รวมถึง `FOR UPDATE` ด้วยกันเอง |

**หมายเหตุ:** ในทางปฏิบัติ เมื่อคุณเขียน `UPDATE table SET col = val WHERE ...` ตามปกติ (ไม่ใช่ผ่าน `SELECT ... FOR ...`) PostgreSQL จะเลือก lock mode ให้อัตโนมัติ: ถ้า UPDATE แตะ column ที่เป็นส่วนหนึ่งของ unique/primary key จะใช้ `FOR UPDATE` โดยปริยาย ถ้าไม่แตะจะใช้ `FOR NO KEY UPDATE`

### ตาราง Compatibility ของ Row-level Lock

เครื่องหมาย ✅ = อนุญาตให้ถือ lock พร้อมกันได้ (ไม่บล็อก), ❌ = บล็อกกัน (ต้องรอ)

| ถือ lock นี้อยู่ \ transaction ใหม่ขอ | `FOR KEY SHARE` | `FOR SHARE` | `FOR NO KEY UPDATE` | `FOR UPDATE` |
|---|:---:|:---:|:---:|:---:|
| `FOR KEY SHARE`      | ✅ | ✅ | ✅ | ❌ |
| `FOR SHARE`          | ✅ | ✅ | ❌ | ❌ |
| `FOR NO KEY UPDATE`  | ✅ | ❌ | ❌ | ❌ |
| `FOR UPDATE`         | ❌ | ❌ | ❌ | ❌ |

### ตัวอย่าง: FOR SHARE ไม่บล็อกกันเอง แต่บล็อก UPDATE

```sql
-- ===== Session A =====
BEGIN;
SELECT * FROM orders WHERE order_id = 6 FOR SHARE;
-- ได้ผลลัพธ์ทันที (ถือ lock FOR SHARE ไว้)
```

```sql
-- ===== Session B =====
BEGIN;
SELECT * FROM orders WHERE order_id = 6 FOR SHARE;
-- ได้ผลลัพธ์ทันทีเช่นกัน! FOR SHARE ไม่บล็อก FOR SHARE ด้วยกันเอง
```

```sql
-- ===== Session A (ยังไม่ COMMIT) — ลองถามอีกเซสชันมา UPDATE =====
-- ===== Session C =====
UPDATE orders SET status = 'cancelled' WHERE order_id = 6;
-- Session C จะ "ค้าง" (waiting) จนกว่า Session A และ B จะ COMMIT/ROLLBACK
```

```sql
-- กลับไปที่ Session A และ B
COMMIT;   -- ที่ Session A
COMMIT;   -- ที่ Session B
```

```sql
-- ===== Session C =====
-- ทันทีที่ A และ B ปล่อย lock, Session C จะ UPDATE สำเร็จและแสดงผล
```

### ตรวจสอบ lock จริงด้วย pg_locks

ระหว่างที่ Session A และ B ยังถือ `FOR SHARE` อยู่ (ก่อน COMMIT) ลองรันคำสั่งนี้ในเซสชันที่สาม (Session D สำหรับ monitor):

```sql
-- ===== Session D (สำหรับสังเกตการณ์) =====
SELECT
    l.pid,
    a.usename,
    l.mode,
    l.granted,
    a.query
FROM pg_locks l
JOIN pg_stat_activity a ON l.pid = a.pid
WHERE l.relation = 'orders'::regclass
ORDER BY l.pid;
```

ผลลัพธ์ตัวอย่าง (ระหว่างที่ A, B ถือ lock และ C กำลังรอ):

```
  pid  | usename  |        mode         | granted |                    query
-------+----------+---------------------+---------+----------------------------------------------
 10231 | postgres | RowShareLock        | t       | SELECT * FROM orders WHERE order_id = 6 FOR SHARE;
 10245 | postgres | RowShareLock        | t       | SELECT * FROM orders WHERE order_id = 6 FOR SHARE;
 10260 | postgres | RowExclusiveLock    | t       | UPDATE orders SET status = 'cancelled' WHERE order_id = 6;
```

**ข้อสังเกตสำคัญ:** ค่าที่แสดงใน `pg_locks.mode` เป็น **table-level lock mode** (`RowShareLock`, `RowExclusiveLock`) ที่ใช้ประกอบการทำ row-level lock — PostgreSQL ใช้กลไก 2 ชั้น: table-level lock แบบเบา (เพื่อป้องกันไม่ให้ `DROP TABLE` มาแย่งระหว่างทาง) ผสมกับ row-level lock ที่แท้จริงซึ่งเก็บอยู่ใน tuple header ของแต่ละแถวเอง (ไม่ปรากฏในมุมมองของ `pg_locks` โดยตรงเว้นแต่เกิดการรอคอย) เมื่อเกิดการรอคอยจริง (Session C ที่ค้างอยู่) จะเห็น row เพิ่มเติมที่มี `locktype = 'tuple'` และ `granted = f`:

```sql
SELECT pid, locktype, mode, granted, relation::regclass
FROM pg_locks
WHERE NOT granted;
```

```
  pid  | locktype |     mode      | granted | relation
-------+----------+---------------+---------+----------
 10260 | tuple    | ExclusiveLock | f       | orders
```

---

## Step 575: SELECT ... FOR UPDATE ตัวอย่างจริง — ป้องกัน Race Condition ตอนจองสินค้า

### ปัญหา: Race Condition แบบคลาสสิก

สมมติมีสินค้าเหลือ 1 ชิ้น ("ตั๋วคอนเสิร์ต โซน A" `product_id = 10`, `stock_quantity = 1`) และมีลูกค้าสองคนกดสั่งซื้อพร้อมกันในเวลาไล่เลี่ยกัน ถ้าแอปพลิเคชันเขียนโค้ดแบบไร้เดียงสา (naive) ดังนี้:

```sql
-- โค้ดที่ "ผิด" — มี race condition
-- Step 1: อ่าน stock ปัจจุบัน
SELECT stock_quantity FROM products WHERE product_id = 10;
-- แอปเห็นว่า stock_quantity = 1 จึงคิดว่า "พอขาย"

-- Step 2: แอปสั่ง UPDATE ลด stock
UPDATE products SET stock_quantity = stock_quantity - 1 WHERE product_id = 10;
```

ถ้าสอง transaction (ลูกค้า X และ Y) รัน `SELECT` พร้อมกันก่อนที่ใครจะ `UPDATE` ทั้งคู่จะเห็น `stock_quantity = 1` เหมือนกัน แล้วต่างฝ่ายต่างคิดว่าตัวเองซื้อได้ — ผลคือ **ขายเกินสต็อก (oversell)**

### วิธีแก้ด้วย SELECT ... FOR UPDATE

`FOR UPDATE` จะทำให้ transaction ที่อ่านแถวนั้น **ล็อกแถวไว้ทันที** และ transaction อื่นที่ต้องการ `FOR UPDATE`/`UPDATE`/`DELETE` แถวเดียวกันจะต้อง **รอ** จนกว่า transaction แรกจะ `COMMIT` หรือ `ROLLBACK`

```sql
-- ===== Session A (ลูกค้า X พยายามจองตั๋ว) =====
BEGIN;

SELECT product_id, product_name, stock_quantity
FROM products
WHERE product_id = 10
FOR UPDATE;
-- ได้ผลลัพธ์: stock_quantity = 1, และแถวนี้ถูก "ล็อก" ไว้แล้ว
```

```sql
-- ===== Session B (ลูกค้า Y พยายามจองตั๋วพร้อมกัน) =====
BEGIN;

SELECT product_id, product_name, stock_quantity
FROM products
WHERE product_id = 10
FOR UPDATE;
-- *** ค้าง (blocking) *** เพราะ Session A ถือ lock อยู่และยังไม่ COMMIT
```

```sql
-- ===== Session A (ทำงานต่อ หลังตรวจสอบว่า stock > 0) =====
UPDATE products SET stock_quantity = stock_quantity - 1 WHERE product_id = 10;

INSERT INTO orders (customer_id, status) VALUES (201, 'pending');

COMMIT;
-- ทันทีที่ COMMIT, Session B จะถูกปลดล็อกและได้รับผลลัพธ์
```

```sql
-- ===== Session B (ทำงานต่อทันทีหลัง A COMMIT) =====
-- ผลลัพธ์ที่ Session B ได้รับ: stock_quantity = 0
-- (Session B เห็นค่าล่าสุดหลัง A commit เพราะ FOR UPDATE จะไป "รีเฟรช"
--  แถวเป็นเวอร์ชันล่าสุดเสมอ ไม่ใช่ snapshot เดิมตอนเริ่ม SELECT)

-- แอปพลิเคชันของ Session B ต้อง "เช็คซ้ำ" ค่าที่ได้รับ:
-- ถ้า stock_quantity = 0 ต้องปฏิเสธการจอง ไม่ COMMIT การขาย
ROLLBACK;  -- ยกเลิก เพราะ stock หมดแล้ว
```

**บทเรียนสำคัญ:** `FOR UPDATE` ทำให้ Session B ต้อง **รอ** ไม่ใช่แค่ป้องกัน dirty read — และเมื่อ Session B ได้ทำงานต่อ มันจะเห็น**ข้อมูลเวอร์ชันล่าสุด**ของแถวนั้นเสมอ (ไม่ใช่ snapshot เก่าตอนเริ่ม transaction) ซึ่งเป็นพฤติกรรมพิเศษของ locking clause ที่ต่างจาก plain `SELECT` — ทำให้แอปพลิเคชันสามารถเช็คเงื่อนไข (`stock_quantity > 0`) ได้อย่างถูกต้องแม่นยำ ปลอดภัยจาก race condition 100%

### รูปแบบการเขียนโค้ดแอปพลิเคชันที่ถูกต้อง (pattern)

```sql
BEGIN;

-- 1. ล็อกแถวสินค้าที่จะขาย พร้อมอ่านค่าปัจจุบัน
SELECT stock_quantity INTO STRICT /* ในบริบท PL/pgSQL */
FROM products WHERE product_id = 10 FOR UPDATE;

-- 2. ตรวจสอบเงื่อนไขทางธุรกิจในแอปพลิเคชัน/PL/pgSQL
--    ถ้า stock_quantity < 1 → ROLLBACK ทันที พร้อมแจ้ง error "สินค้าหมด"

-- 3. ถ้าผ่านเงื่อนไข ค่อยลด stock และสร้าง order
UPDATE products SET stock_quantity = stock_quantity - 1 WHERE product_id = 10;
INSERT INTO orders (customer_id, status) VALUES (202, 'pending');

COMMIT;
```

ตัวอย่างฟังก์ชัน PL/pgSQL ที่ใช้ pattern นี้จริงในระบบ production (ทบทวนจาก Part 046-047):

```sql
CREATE OR REPLACE FUNCTION reserve_product(
    p_product_id INTEGER,
    p_customer_id INTEGER
) RETURNS INTEGER
LANGUAGE plpgsql
AS $$
DECLARE
    v_stock INTEGER;
    v_order_id INTEGER;
BEGIN
    -- ล็อกแถวสินค้าไว้ก่อนตรวจสอบ
    SELECT stock_quantity INTO v_stock
    FROM products
    WHERE product_id = p_product_id
    FOR UPDATE;

    IF NOT FOUND THEN
        RAISE EXCEPTION 'ไม่พบสินค้ารหัส %', p_product_id;
    END IF;

    IF v_stock < 1 THEN
        RAISE EXCEPTION 'สินค้า % หมดสต็อกแล้ว', p_product_id;
    END IF;

    UPDATE products
    SET stock_quantity = stock_quantity - 1
    WHERE product_id = p_product_id;

    INSERT INTO orders (customer_id, status)
    VALUES (p_customer_id, 'pending')
    RETURNING order_id INTO v_order_id;

    RETURN v_order_id;
END;
$$;

-- เรียกใช้งาน (แต่ละ session ที่เรียกพร้อมกันจะเข้าคิวรอ lock โดยอัตโนมัติ)
SELECT reserve_product(10, 301);
```

---

## Step 576: NOWAIT และ SKIP LOCKED — ทางเลือกเมื่อแถวถูกล็อกอยู่แล้ว

การรอ lock แบบไม่มีกำหนด (block จนกว่าอีกฝ่ายจะ commit) ไม่ใช่พฤติกรรมที่เหมาะกับทุกสถานการณ์ PostgreSQL จึงมี 2 ทางเลือกให้ควบคุมพฤติกรรมนี้:

| Clause | พฤติกรรม | ใช้เมื่อ |
|---|---|---|
| `FOR UPDATE NOWAIT` | ถ้าแถวถูกล็อกอยู่แล้ว **ให้ error ทันที** แทนที่จะรอ | ต้องการตอบสนองผู้ใช้ทันทีว่า "ตอนนี้มีคนกำลังจองอยู่ ลองใหม่อีกครั้ง" |
| `FOR UPDATE SKIP LOCKED` | ถ้าแถวถูกล็อกอยู่แล้ว **ข้ามแถวนั้นไปเลย** (ไม่ error ไม่รอ) แล้วดึงแถวถัดไปที่ยังไม่ถูกล็อก | ระบบ job queue / worker pool ที่มีงานหลายชิ้นให้เลือกทำ ไม่สนว่าจะได้ "แถวไหน" ขอแค่ได้งานที่ยังว่างอยู่ |

### ตัวอย่าง NOWAIT

```sql
-- ===== Session A =====
BEGIN;
SELECT * FROM products WHERE product_id = 10 FOR UPDATE;
-- ถือ lock ไว้ ยังไม่ COMMIT
```

```sql
-- ===== Session B =====
BEGIN;
SELECT * FROM products WHERE product_id = 10 FOR UPDATE NOWAIT;
-- ERROR:  could not obtain lock on row in relation "products"
```

Session B ได้รับ error ทันที ไม่ต้องรอเลย — แอปพลิเคชันสามารถ `catch` error นี้แล้วแสดงข้อความให้ผู้ใช้ทราบว่า "สินค้านี้กำลังถูกจองโดยผู้อื่น กรุณาลองใหม่" แทนที่จะให้ผู้ใช้เห็นหน้าจอค้าง

```sql
-- ===== Session A =====
ROLLBACK;
```

### ตัวอย่าง SKIP LOCKED — Job Queue Pattern

รูปแบบที่ใช้บ่อยที่สุดของ `SKIP LOCKED` คือการสร้าง **job queue** โดยใช้ตาราง `orders` เป็นคิวงานที่ต้อง "ประมวลผล" (เช่น เตรียมพัสดุ, เรียก payment gateway) หลาย worker process สามารถแย่งกันดึงงานได้โดยไม่ชนกัน:

```sql
-- สมมติมี worker หลายตัวที่ทำงานพร้อมกัน แต่ละตัวต้องการ "หยิบ" order
-- ที่ status = 'pending' มาประมวลผลทีละ 1 รายการ โดยไม่แย่งงานกันเอง

-- ===== Worker 1 (Session A) =====
BEGIN;

SELECT order_id, customer_id, status
FROM orders
WHERE status = 'pending'
ORDER BY order_id
FOR UPDATE SKIP LOCKED
LIMIT 1;
-- ได้ order_id = 6 (แถวแรกที่ยังไม่ถูกล็อก) และล็อกแถวนี้ไว้ทันที
```

```sql
-- ===== Worker 2 (Session B) — รันพร้อมกันกับ Worker 1 =====
BEGIN;

SELECT order_id, customer_id, status
FROM orders
WHERE status = 'pending'
ORDER BY order_id
FOR UPDATE SKIP LOCKED
LIMIT 1;
-- Worker 1 ล็อก order_id=6 ไปแล้ว → Worker 2 "ข้าม" แถวนั้นทันที (ไม่รอ ไม่ error)
-- ได้ order_id = 7 (แถวถัดไปที่ยังว่าง) และล็อกแถวนี้ไว้แทน
```

```sql
-- ===== Worker 3 (Session C) — รันพร้อมกันอีกตัว =====
BEGIN;

SELECT order_id, customer_id, status
FROM orders
WHERE status = 'pending'
ORDER BY order_id
FOR UPDATE SKIP LOCKED
LIMIT 1;
-- ได้ order_id = 8 (แถวสุดท้ายที่ยังว่าง)
```

```sql
-- แต่ละ Worker ประมวลผลงานของตัวเอง แล้วอัปเดตสถานะ + commit อิสระจากกัน
-- ===== Worker 1 =====
UPDATE orders SET status = 'shipped' WHERE order_id = 6;
COMMIT;

-- ===== Worker 2 =====
UPDATE orders SET status = 'shipped' WHERE order_id = 7;
COMMIT;

-- ===== Worker 3 =====
UPDATE orders SET status = 'shipped' WHERE order_id = 8;
COMMIT;
```

**ผลลัพธ์:** Worker ทั้งสามตัวทำงาน**พร้อมกันแบบขนานได้จริง** (true parallelism) โดยไม่มีใครต้องรอใคร และไม่มีใครหยิบงานซ้ำกัน — นี่คือรูปแบบที่ระบบคิวงานจริง (job queue) ระดับ production นิยมใช้ เช่น การประมวลผลออเดอร์, ส่งอีเมล, ประมวลผลไฟล์แบบ batch

ฟังก์ชันตัวอย่างที่ครอบ pattern นี้ให้ใช้งานสะดวก:

```sql
CREATE OR REPLACE FUNCTION dequeue_pending_order()
RETURNS TABLE(order_id INTEGER, customer_id INTEGER)
LANGUAGE plpgsql
AS $$
DECLARE
    v_order_id INTEGER;
BEGIN
    SELECT o.order_id INTO v_order_id
    FROM orders o
    WHERE o.status = 'pending'
    ORDER BY o.order_id
    FOR UPDATE SKIP LOCKED
    LIMIT 1;

    IF v_order_id IS NULL THEN
        RETURN;  -- ไม่มีงานเหลือแล้ว
    END IF;

    UPDATE orders SET status = 'processing' WHERE orders.order_id = v_order_id;

    RETURN QUERY
    SELECT v_order_id, o.customer_id FROM orders o WHERE o.order_id = v_order_id;
END;
$$;
```

> **ข้อควรระวัง:** `SKIP LOCKED` ทำให้ผลลัพธ์ **ไม่ deterministic** ในแง่ที่ว่าแถวไหนจะถูกข้าม/ถูกเลือกขึ้นกับจังหวะเวลาจริง (timing) — เหมาะกับงานที่ "หยิบอันไหนก็ได้ที่ว่าง" เท่านั้น **ห้ามใช้** กับงานที่ต้องการลำดับที่แน่นอนแบบ FIFO เข้มงวด (strict ordering) เพราะ worker ที่มาทีหลังอาจข้ามงานที่ควรจะได้คิวก่อนไปเลย

---

## Step 577: Table-level Lock Modes และตาราง Compatibility

นอกจาก row-level lock แล้ว ทุกคำสั่ง SQL ยังต้องขอ **table-level lock** ควบคู่ไปด้วยเสมอ (แม้จะเป็นแค่ lock แบบเบาที่สุด) PostgreSQL มี table-level lock ทั้งหมด **8 ระดับ** เรียงจากอ่อนสุดไปแรงสุด:

| # | Lock Mode | ใช้โดยคำสั่ง | คำอธิบาย |
|---|---|---|---|
| 1 | `ACCESS SHARE` | `SELECT` | อ่อนที่สุด — บล็อกเฉพาะ `ACCESS EXCLUSIVE` เท่านั้น |
| 2 | `ROW SHARE` | `SELECT ... FOR UPDATE/FOR SHARE` | ใช้ร่วมกับ row-level locking clause |
| 3 | `ROW EXCLUSIVE` | `UPDATE`, `DELETE`, `INSERT` | คำสั่งเขียนข้อมูลทั่วไป |
| 4 | `SHARE UPDATE EXCLUSIVE` | `VACUUM` (ไม่ full), `CREATE INDEX CONCURRENTLY`, `ANALYZE` | ป้องกันการแก้ไข schema พร้อมกัน แต่ไม่บล็อกการอ่าน-เขียนข้อมูลปกติ |
| 5 | `SHARE` | `CREATE INDEX` (ไม่ concurrently) | อนุญาตให้อ่านได้ แต่บล็อกการเขียน |
| 6 | `SHARE ROW EXCLUSIVE` | `CREATE TRIGGER`, บาง `ALTER TABLE` | เข้มกว่า `SHARE` เล็กน้อย |
| 7 | `EXCLUSIVE` | ไม่ค่อยใช้โดยตรงจากคำสั่งทั่วไป | อนุญาตแค่ `ACCESS SHARE` เท่านั้นที่ผ่านได้ |
| 8 | `ACCESS EXCLUSIVE` | `DROP TABLE`, `TRUNCATE`, `ALTER TABLE` (ส่วนใหญ่), `VACUUM FULL`, `REINDEX` (ไม่ concurrently) | แรงที่สุด — บล็อกทุกอย่าง แม้แต่ `SELECT` |

### Lock Compatibility Matrix (ตารางความเข้ากันได้แบบเต็ม)

✅ = ขอพร้อมกันได้ (ไม่บล็อก) | ❌ = ต้องรอ (บล็อกกัน)

| ถือ lock อยู่ ↓ \ ขอใหม่ → | AS | RS | RE | SUE | S | SRE | E | AE |
|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| **ACCESS SHARE (AS)**            | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ |
| **ROW SHARE (RS)**               | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ |
| **ROW EXCLUSIVE (RE)**           | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| **SHARE UPDATE EXCLUSIVE (SUE)** | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| **SHARE (S)**                    | ✅ | ✅ | ❌ | ❌ | ✅ | ❌ | ❌ | ❌ |
| **SHARE ROW EXCLUSIVE (SRE)**    | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| **EXCLUSIVE (E)**                | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| **ACCESS EXCLUSIVE (AE)**        | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |

**วิธีอ่านตาราง:** แถวคือ lock ที่ transaction A ถือไว้อยู่แล้ว คอลัมน์คือ lock ที่ transaction B กำลังขอ ถ้าตัดกันเป็น ❌ แปลว่า B ต้องรอ A ปล่อย lock ก่อน

จุดที่ควรจำขึ้นใจ:

- `ACCESS SHARE` (จาก `SELECT` ธรรมดา) เข้ากันได้กับเกือบทุกอย่าง ยกเว้น `ACCESS EXCLUSIVE`
- `ROW EXCLUSIVE` (จาก `UPDATE`/`INSERT`/`DELETE` ปกติ) เข้ากันได้กับ `SELECT` และกันเอง แต่บล็อก `CREATE INDEX` (ไม่ concurrently) และ `ALTER TABLE`
- `ACCESS EXCLUSIVE` บล็อกทุกอย่างแม้แต่ `SELECT` — เป็นสาเหตุอันดับ 1 ของ downtime ที่ไม่คาดคิดใน production เมื่อรัน `ALTER TABLE`/`DROP TABLE`/`TRUNCATE` บนตารางที่มี traffic สูง

### ทดลองดู lock mode จริงด้วย pg_locks

```sql
-- ===== Session A =====
BEGIN;
UPDATE products SET stock_quantity = stock_quantity WHERE product_id = 1;
-- ยังไม่ COMMIT
```

```sql
-- ===== Session D (monitor) =====
SELECT
    l.pid,
    l.mode,
    l.granted,
    l.relation::regclass AS table_name
FROM pg_locks l
WHERE l.relation = 'products'::regclass;
```

```
  pid  |      mode      | granted | table_name
-------+----------------+---------+------------
 10231 | RowExclusiveLock | t     | products
```

จะเห็นว่าคำสั่ง `UPDATE` ปกติขอแค่ `RowExclusiveLock` ระดับตาราง (lock อ่อน) เท่านั้น — ตัว lock จริงที่ป้องกัน write-write conflict อยู่ที่ระดับ tuple (row) ต่างหาก ระดับตารางแค่ป้องกันไม่ให้มีคนมา `DROP`/`ALTER` ตารางกลางคันเท่านั้น

---

## Step 578: คำสั่งที่ล็อกระดับต่าง ๆ โดยปริยาย และผลกระทบต่อ Production

การเข้าใจว่าคำสั่งแต่ละแบบขอ lock ระดับไหน เป็นสิ่งจำเป็นอย่างยิ่งในการวางแผน deployment/migration บนระบบ production ที่มี traffic ตลอดเวลา

### ตารางสรุปคำสั่งกับ lock mode ที่ขอ

| คำสั่ง | Lock Mode | ผลกระทบ |
|---|---|---|
| `SELECT` | `ACCESS SHARE` | ปลอดภัยที่สุด ไม่บล็อกใคร (นอกจาก `ACCESS EXCLUSIVE`) |
| `SELECT ... FOR UPDATE/SHARE` | `ROW SHARE` | ยังคง permissive ระดับตาราง |
| `INSERT`, `UPDATE`, `DELETE` | `ROW EXCLUSIVE` | ปลอดภัยสำหรับ traffic ปกติ แต่บล็อก `CREATE INDEX`/`ALTER TABLE` ที่ไม่ concurrent |
| `CREATE INDEX CONCURRENTLY` | `SHARE UPDATE EXCLUSIVE` | ปลอดภัยสำหรับ production — อนุญาตให้อ่าน-เขียนต่อไปได้ระหว่างสร้าง index (ใช้เวลานานกว่าปกติ แต่ไม่ downtime) |
| `CREATE INDEX` (ธรรมดา) | `SHARE` | **บล็อกการเขียนทั้งหมด** จนกว่าจะสร้างเสร็จ — อันตรายมากบนตารางใหญ่ที่มี traffic |
| `VACUUM` (ไม่ full) | `SHARE UPDATE EXCLUSIVE` | ทำงานคู่ขนานกับ traffic ปกติได้ ไม่บล็อกอ่าน-เขียน |
| `VACUUM FULL` | `ACCESS EXCLUSIVE` | **บล็อกทุกอย่าง** ระหว่างจัดเรียงตารางใหม่ทั้งหมด — ห้ามรันช่วง peak hours |
| `ALTER TABLE ADD COLUMN` (ไม่มี volatile default) | `ACCESS EXCLUSIVE` (แต่ใช้เวลาสั้นมาก ตั้งแต่ PG11+ ไม่ rewrite ตาราง) | บล็อกช่วงสั้น ๆ ระดับ metadata เท่านั้น |
| `ALTER TABLE ... ADD COLUMN ... DEFAULT <volatile func>` | `ACCESS EXCLUSIVE` (ใช้เวลานาน เพราะต้อง rewrite ทั้งตาราง) | อันตราย — ตารางใหญ่อาจ block นานหลายนาทีถึงชั่วโมง |
| `ALTER TABLE ... ALTER COLUMN TYPE` | `ACCESS EXCLUSIVE` (มักต้อง rewrite ตาราง + index) | อันตรายมากบนตารางใหญ่ |
| `DROP TABLE` | `ACCESS EXCLUSIVE` | บล็อกทุกอย่างจนกว่าจะลบเสร็จ (ปกติเร็ว แต่ถ้ามี transaction อื่น**ถือ lock ค้างอยู่ก่อน** คำสั่งนี้จะต้อง**รอ**ก่อน แล้ว query ใหม่ ๆ ทั้งหมดที่มาทีหลังจะต้องต่อคิวรอ `DROP TABLE` ด้วย — เกิด "lock queue pile-up") |
| `TRUNCATE` | `ACCESS EXCLUSIVE` | เช่นเดียวกับ `DROP TABLE` |
| `REINDEX` (ไม่ concurrently) | `ACCESS EXCLUSIVE` | บล็อกทุกอย่างระหว่างสร้าง index ใหม่ — ใช้ `REINDEX CONCURRENTLY` แทนใน production |

### กรณีศึกษา: Lock Queue Pile-up ใน Production

สถานการณ์ที่พบบ่อยและอันตรายมากคือ: DBA รันคำสั่งที่ต้องการ `ACCESS EXCLUSIVE` (เช่น `ALTER TABLE`) บนตารางที่มี transaction อื่นถือ lock (แม้จะเป็นแค่ `ACCESS SHARE` จาก `SELECT` ธรรมดา) ค้างอยู่นานผิดปกติ:

```sql
-- ===== Session A: transaction ที่เปิดค้างไว้นาน (อาจลืม COMMIT) =====
BEGIN;
SELECT * FROM orders WHERE order_id = 1;
-- (ลืม COMMIT ไว้เฉย ๆ นาน 10 นาที)
```

```sql
-- ===== Session B: DBA ต้องการเพิ่มคอลัมน์ (ใช้เวลาไม่นานในตัวเอง) =====
ALTER TABLE orders ADD COLUMN shipping_method VARCHAR(30);
-- คำสั่งนี้ "ค้าง" รอ ACCESS EXCLUSIVE lock เพราะ Session A ยังถือ ACCESS SHARE อยู่
```

```sql
-- ===== Session C, D, E, ... : query ปกติทุกตัวที่มาทีหลัง Session B =====
SELECT * FROM orders WHERE order_id = 2;
-- แม้จะเป็นแค่ SELECT ธรรมดา (ต้องการแค่ ACCESS SHARE) ก็ยังต้อง "ต่อคิว" รอด้วย!
-- เพราะ Session B (ซึ่งขอ ACCESS EXCLUSIVE) "จองคิว" ไว้ก่อนแล้ว
-- PostgreSQL จัดคิว lock แบบ FIFO — query ที่มาทีหลังคำสั่งที่รอ AE lock
-- จะต้องรอ AE lock ปล่อยก่อน แม้ตัวเองจะขอแค่ AS ก็ตาม
```

นี่คือปรากฏการณ์ **"lock queue pile-up"**: query ธรรมดาจำนวนมากที่ควรจะเร็ว กลับต้องรอเป็นแถวยาว ทำให้ระบบทั้งหมดดูเหมือน "ค้าง" (connection pool เต็ม, timeout ทั้งระบบ) ทั้งที่จริง ๆ แล้วต้นเหตุคือ transaction เดียวที่ลืม `COMMIT`

**วิธีตรวจจับปัญหานี้:**

```sql
-- หา transaction ที่เปิดค้างนานผิดปกติ
SELECT pid, usename, state, query,
       now() - xact_start AS transaction_duration
FROM pg_stat_activity
WHERE state != 'idle'
  AND xact_start IS NOT NULL
ORDER BY xact_start
LIMIT 10;

-- หาความสัมพันธ์ว่า pid ไหนกำลังบล็อก pid ไหน (ใช้ pg_blocking_pids)
SELECT
    blocked.pid       AS blocked_pid,
    blocked.query     AS blocked_query,
    blocking.pid      AS blocking_pid,
    blocking.query    AS blocking_query
FROM pg_stat_activity AS blocked
JOIN pg_stat_activity AS blocking
    ON blocking.pid = ANY(pg_blocking_pids(blocked.pid))
WHERE cardinality(pg_blocking_pids(blocked.pid)) > 0;
```

```
 blocked_pid |              blocked_query               | blocking_pid |            blocking_query
-------------+-------------------------------------------+--------------+---------------------------------------
       10260 | SELECT * FROM orders WHERE order_id = 2;   |        10245 | ALTER TABLE orders ADD COLUMN ...
       10270 | SELECT * FROM orders WHERE order_id = 3;   |        10245 | ALTER TABLE orders ADD COLUMN ...
```

จาก query นี้จะเห็นชัดว่า `pid 10245` (คำสั่ง `ALTER TABLE`) เป็นตัวการที่บล็อกคนอื่น — และถ้าสืบต่อไปอีกชั้น จะพบว่า `10245` เองก็กำลังรอ `pid` ของ Session A (ที่ลืม commit) อยู่

**แนวทางแก้ไข/ป้องกันในระบบ production:**

1. ตั้งค่า `statement_timeout` และ `idle_in_transaction_session_timeout` เพื่อบังคับปิด transaction ที่ค้างนานเกินไปโดยอัตโนมัติ
2. ใช้ `CREATE INDEX CONCURRENTLY` และ `REINDEX CONCURRENTLY` แทนแบบธรรมดาเสมอในตารางที่มี traffic
3. สำหรับ `ALTER TABLE` ที่ต้องใช้ `ACCESS EXCLUSIVE` ให้ตั้ง `lock_timeout` สั้น ๆ ก่อนรัน เพื่อไม่ให้คำสั่งไป "จองคิว" ค้างนาน:

```sql
BEGIN;
SET lock_timeout = '2s';  -- ถ้าขอ lock ไม่ได้ภายใน 2 วินาที ให้ยกเลิกและลองใหม่ทีหลัง
ALTER TABLE orders ADD COLUMN shipping_method VARCHAR(30);
COMMIT;
```

---

## Step 579: Advisory Lock — pg_advisory_lock สำหรับ Application-level Locking

บางครั้งเราต้องการ "ล็อก" บางสิ่งที่**ไม่ใช่แถวหรือตาราง** เช่น ล็อกเพื่อป้องกันไม่ให้ batch job สองตัวรันซ้อนกัน, ล็อกเพื่อทำ leader election, หรือล็อกเพื่อ synchronize การประมวลผลที่ผูกกับ business key (เช่น "customer_id เดียวกันห้ามประมวลผลพร้อมกัน") ซึ่งไม่ได้ผูกกับแถวจริงในตารางใดตารางหนึ่งโดยเฉพาะ

PostgreSQL มี **Advisory Lock** ที่เป็น lock ระดับ "ตกลงกันเอง" (คำว่า advisory แปลว่า "ให้คำแนะนำ" — คือ PostgreSQL จะไม่บังคับใช้กับ query อื่นใดโดยอัตโนมัติ มันทำหน้าที่แค่เป็นกลไก lock ให้แอปพลิเคชันเรียกใช้เอง)

### ฟังก์ชันหลักที่ใช้งาน

| ฟังก์ชัน | ขอบเขต | พฤติกรรม |
|---|---|---|
| `pg_advisory_lock(key)` | Session-level | ล็อกและ**รอ**จนกว่าจะได้ — ต้องปลดล็อกเองด้วย `pg_advisory_unlock` |
| `pg_try_advisory_lock(key)` | Session-level | ลองล็อก **ไม่รอ** — คืนค่า `true`/`false` ทันที |
| `pg_advisory_unlock(key)` | Session-level | ปลดล็อกที่ถืออยู่ |
| `pg_advisory_xact_lock(key)` | Transaction-level | ล็อกและรอจนกว่าจะได้ — **ปลดล็อกอัตโนมัติเมื่อ COMMIT/ROLLBACK** (ไม่ต้องปลดเอง) |
| `pg_try_advisory_xact_lock(key)` | Transaction-level | ลองล็อกแบบ transaction-level ไม่รอ |

`key` เป็นเลข `bigint` (หรือคู่ `int, int` ก็ได้ เพื่อแยก namespace เช่น `pg_advisory_lock(module_id, entity_id)`)

### ตัวอย่าง: ป้องกัน Batch Job รันซ้อนกัน

สมมติมี cron job ที่รันทุก 5 นาทีเพื่อสรุปยอดขายรายวัน แต่ถ้า job รอบก่อนยังทำไม่เสร็จ (เช่น ข้อมูลเยอะผิดปกติ) ไม่อยากให้ job รอบใหม่เริ่มซ้อนกัน:

```sql
-- ===== Session A (job รอบที่ 1 เริ่มทำงาน) =====
SELECT pg_try_advisory_lock(123456789);
-- ผลลัพธ์: true (ได้ lock)

-- ... กำลังประมวลผลสรุปยอดขาย (สมมติใช้เวลานาน) ...
```

```sql
-- ===== Session B (job รอบที่ 2 ถูกเรียกซ้ำโดย scheduler ก่อนรอบแรกจะเสร็จ) =====
SELECT pg_try_advisory_lock(123456789);
-- ผลลัพธ์: false (ไม่ได้ lock เพราะ Session A ถืออยู่)
-- แอปพลิเคชันเช็คค่านี้แล้ว "ออกจากโปรแกรมทันที" โดยไม่ทำงานซ้ำ
```

```sql
-- ===== Session A (ทำงานเสร็จแล้ว) =====
SELECT pg_advisory_unlock(123456789);
-- ผลลัพธ์: true (ปลดล็อกสำเร็จ)
```

### ตัวอย่าง: Transaction-scoped Advisory Lock สำหรับป้องกัน "double-processing" ต่อ customer

```sql
-- ป้องกันไม่ให้ระบบประมวลผล order ของ customer เดียวกันพร้อมกันสองทาง
-- (เช่น มี worker สองตัวถูก trigger พร้อมกันเพราะ event ซ้ำ)

BEGIN;

-- ใช้ customer_id เป็น key ของ advisory lock (แปลงเป็น bigint)
SELECT pg_advisory_xact_lock(101::bigint);

-- ทำงานที่เกี่ยวกับ customer_id = 101 อย่างปลอดภัย รู้แน่ว่าไม่มีใครแตะพร้อมกัน
UPDATE orders SET status = 'processing'
WHERE customer_id = 101 AND status = 'pending';

COMMIT;
-- lock ถูกปลดอัตโนมัติทันทีที่ COMMIT (หรือ ROLLBACK) — ไม่ต้องเรียก unlock เอง
```

### ตรวจสอบ Advisory Lock ที่กำลังถืออยู่

```sql
SELECT pid, mode, granted, objid AS advisory_key
FROM pg_locks
WHERE locktype = 'advisory';
```

```
  pid  |        mode         | granted | advisory_key
-------+----------------------+---------+---------------
 10231 | ExclusiveLock        | t       |     123456789
```

### ข้อควรระวังเกี่ยวกับ Advisory Lock

- **Session-level lock ต้องปลดเองเสมอ** ถ้าลืม `pg_advisory_unlock` และ connection ยังไม่ปิด lock จะค้างตลอดไป (ทางแก้คือใช้ transaction-level แทนถ้าเป็นไปได้ เพราะจะถูกปลดอัตโนมัติ)
- Advisory lock **ไม่ผูกกับข้อมูลจริง** — เป็นความรับผิดชอบของแอปพลิเคชันในการเลือก `key` ให้สื่อความหมายตรงกันในทุกจุดที่เรียกใช้ (เช่น ตกลงกันว่า `customer_id` แปลงเป็น advisory key แบบไหน)
- ถ้าใช้ connection pooler แบบ transaction pooling (เช่น PgBouncer โหมด transaction) **ห้ามใช้ session-level advisory lock** เพราะ connection อาจถูกสลับไปใช้กับ client อื่นก่อนที่จะปลดล็อก — ควรใช้ `pg_advisory_xact_lock` แทนเสมอในสภาพแวดล้อมแบบนี้

---

## Step 580: แบบฝึกหัดรวม — ออกแบบระบบจองคิว/จองสินค้าที่ปลอดภัยจาก Race Condition

มาผสมผสานทุกเทคนิคที่เรียนมาในบทนี้ เพื่อออกแบบ **ระบบจองที่นั่งคอนเสิร์ต** ที่ต้องรองรับผู้ใช้จำนวนมากกดจองพร้อมกัน โดยไม่ให้เกิดการจองซ้ำหรือขายเกินจำนวน

### ความต้องการของระบบ

1. สินค้า/ที่นั่งมีจำนวนจำกัด (`stock_quantity`) — ห้ามขายเกิน
2. ต้องการให้ระบบตอบสนองเร็ว ถ้าคนอื่นกำลังจองที่นั่งเดียวกันอยู่ ให้แจ้งผู้ใช้ทันทีว่า "กำลังมีคนจองอยู่ กรุณาลองอีกครั้ง" แทนที่จะให้หน้าจอค้างรอนาน
3. มี background worker แยกต่างหากที่คอย "ยกเลิกอัตโนมัติ" สำหรับ order ที่จองไว้แต่ไม่ชำระเงินภายใน 15 นาที (ทำงานแบบ queue โดย worker หลายตัวช่วยกันได้)
4. ป้องกันไม่ให้ batch job ที่ทำสรุปยอดขายประจำวันรันซ้อนกันสองรอบ

### แนวทางออกแบบ (Solution Design)

**ส่วนที่ 1: ฟังก์ชันจองที่นั่ง (ใช้ `FOR UPDATE NOWAIT` เพื่อตอบสนองเร็ว)**

```sql
CREATE OR REPLACE FUNCTION book_seat(
    p_product_id INTEGER,
    p_customer_id INTEGER
) RETURNS TABLE(success BOOLEAN, message TEXT, order_id INTEGER)
LANGUAGE plpgsql
AS $$
DECLARE
    v_stock INTEGER;
    v_new_order_id INTEGER;
BEGIN
    BEGIN
        SELECT stock_quantity INTO v_stock
        FROM products
        WHERE product_id = p_product_id
        FOR UPDATE NOWAIT;
    EXCEPTION
        WHEN lock_not_available THEN
            RETURN QUERY SELECT false, 'มีผู้ใช้อื่นกำลังจองที่นั่งนี้อยู่ กรุณาลองอีกครั้ง'::TEXT, NULL::INTEGER;
            RETURN;
    END;

    IF NOT FOUND THEN
        RETURN QUERY SELECT false, 'ไม่พบสินค้า/ที่นั่งนี้'::TEXT, NULL::INTEGER;
        RETURN;
    END IF;

    IF v_stock < 1 THEN
        RETURN QUERY SELECT false, 'ที่นั่งเต็มแล้ว'::TEXT, NULL::INTEGER;
        RETURN;
    END IF;

    UPDATE products SET stock_quantity = stock_quantity - 1
    WHERE product_id = p_product_id;

    INSERT INTO orders (customer_id, status)
    VALUES (p_customer_id, 'pending')
    RETURNING orders.order_id INTO v_new_order_id;

    RETURN QUERY SELECT true, 'จองสำเร็จ กรุณาชำระเงินภายใน 15 นาที'::TEXT, v_new_order_id;
END;
$$;

-- ทดสอบเรียกใช้
SELECT * FROM book_seat(10, 401);
```

**ส่วนที่ 2: Worker ยกเลิก order ที่ค้างชำระเงิน (ใช้ `SKIP LOCKED` สำหรับ worker หลายตัว)**

```sql
CREATE OR REPLACE FUNCTION cancel_expired_reservation()
RETURNS INTEGER
LANGUAGE plpgsql
AS $$
DECLARE
    v_order_id INTEGER;
    v_product_id INTEGER := 10;  -- ในระบบจริงควรมีตาราง order_items เชื่อมโยง product กับ order
BEGIN
    SELECT o.order_id INTO v_order_id
    FROM orders o
    WHERE o.status = 'pending'
      AND o.order_date < now() - interval '15 minutes'
    ORDER BY o.order_id
    FOR UPDATE SKIP LOCKED
    LIMIT 1;

    IF v_order_id IS NULL THEN
        RETURN NULL;  -- ไม่มีงานให้ทำ
    END IF;

    UPDATE orders SET status = 'cancelled' WHERE order_id = v_order_id;

    -- คืนสต็อกกลับเข้าไป (ต้องล็อกแถวสินค้าด้วย FOR UPDATE เช่นกัน)
    UPDATE products SET stock_quantity = stock_quantity + 1
    WHERE product_id = v_product_id;

    RETURN v_order_id;
END;
$$;

-- worker หลายตัวเรียกฟังก์ชันนี้พร้อมกันได้อย่างปลอดภัย เพราะ SKIP LOCKED
-- ทำให้แต่ละตัวได้ order คนละใบ ไม่ชนกัน
SELECT cancel_expired_reservation();
```

**ส่วนที่ 3: ป้องกัน batch job สรุปยอดขายรันซ้อน (ใช้ Advisory Lock)**

```sql
CREATE OR REPLACE FUNCTION run_daily_sales_summary()
RETURNS TEXT
LANGUAGE plpgsql
AS $$
DECLARE
    v_got_lock BOOLEAN;
BEGIN
    v_got_lock := pg_try_advisory_lock(hashtext('daily_sales_summary_job'));

    IF NOT v_got_lock THEN
        RETURN 'มี job รอบก่อนหน้ายังทำงานไม่เสร็จ ข้ามรอบนี้ไปก่อน';
    END IF;

    -- ทำงานสรุปยอดขาย (ตัวอย่างง่าย ๆ)
    PERFORM pg_sleep(2);  -- จำลองงานที่ใช้เวลานาน

    PERFORM pg_advisory_unlock(hashtext('daily_sales_summary_job'));

    RETURN 'สรุปยอดขายเสร็จสมบูรณ์';
END;
$$;

SELECT run_daily_sales_summary();
```

> **หมายเหตุ:** ใช้ `hashtext('daily_sales_summary_job')` เพื่อแปลง string ที่สื่อความหมายให้เป็นเลข `bigint` (จริง ๆ แล้ว `hashtext` คืนค่า `int` แต่ PostgreSQL จะแปลงให้อัตโนมัติ) วิธีนี้ทำให้โค้ดอ่านง่ายกว่าการจำเลข magic number

### สรุปการเลือกใช้เทคนิคในระบบนี้

| ปัญหา | เทคนิคที่ใช้ | เหตุผล |
|---|---|---|
| ป้องกัน oversell ตอนจอง | `SELECT ... FOR UPDATE NOWAIT` | ต้องล็อกแถวจริง และต้องการตอบกลับผู้ใช้เร็ว ไม่ให้ค้างรอ |
| Worker หลายตัวช่วยกันยกเลิก order ที่หมดเวลา | `SELECT ... FOR UPDATE SKIP LOCKED` | ต้องการให้ worker ทำงานขนานกันได้ ไม่แคร์ว่าจะได้ order ใบไหน |
| ป้องกัน batch job รันซ้อน | `pg_try_advisory_lock` | ไม่มีแถว/ตารางที่เกี่ยวข้องโดยตรง เป็น application-level coordination |

---

## สรุปท้ายบท

### ตาราง Lock Mode ทั้งหมดในระบบ (Row-level + Table-level)

**Row-level Lock (4 ระดับ):**

| Lock Mode | คำสั่งที่ใช้ | ระดับความเข้ม |
|---|---|:---:|
| `FOR KEY SHARE` | ป้องกันการเปลี่ยน key column ขณะที่ FK อ้างอิงอยู่ | อ่อนสุด |
| `FOR SHARE` | อ่านแถวและมั่นใจว่าจะไม่ถูกแก้ไข/ลบ | ⬆ |
| `FOR NO KEY UPDATE` | จะแก้ไขแถวโดยไม่แตะ key column | ⬆ |
| `FOR UPDATE` | จะแก้ไข/ลบแถวแน่นอน | แรงสุด |

**Table-level Lock (8 ระดับ เรียงจากอ่อนไปแรง):**

| # | Lock Mode | ตัวอย่างคำสั่ง |
|---|---|---|
| 1 | `ACCESS SHARE` | `SELECT` |
| 2 | `ROW SHARE` | `SELECT ... FOR UPDATE/FOR SHARE` |
| 3 | `ROW EXCLUSIVE` | `INSERT`, `UPDATE`, `DELETE` |
| 4 | `SHARE UPDATE EXCLUSIVE` | `VACUUM`, `CREATE INDEX CONCURRENTLY`, `ANALYZE` |
| 5 | `SHARE` | `CREATE INDEX` |
| 6 | `SHARE ROW EXCLUSIVE` | `CREATE TRIGGER` |
| 7 | `EXCLUSIVE` | (ไม่ค่อยพบจากคำสั่งตรง ๆ) |
| 8 | `ACCESS EXCLUSIVE` | `DROP TABLE`, `TRUNCATE`, `ALTER TABLE` ส่วนใหญ่, `VACUUM FULL` |

### คำสั่ง/ฟังก์ชันสำคัญที่ต้องจำ

```sql
-- Row-level locking
SELECT ... FOR UPDATE;                 -- ล็อกแรงสุด จะแก้ไข/ลบแถวแน่นอน
SELECT ... FOR NO KEY UPDATE;          -- จะแก้ไขแต่ไม่แตะ key column
SELECT ... FOR SHARE;                  -- อ่านแบบมั่นใจว่าไม่ถูกแก้ไข
SELECT ... FOR KEY SHARE;              -- อ่อนสุด สำหรับอ้างอิง FK
SELECT ... FOR UPDATE NOWAIT;          -- ไม่รอ error ทันทีถ้าถูกล็อก
SELECT ... FOR UPDATE SKIP LOCKED;     -- ข้ามแถวที่ถูกล็อก (job queue pattern)

-- Advisory lock
pg_advisory_lock(key);                 -- session-level, ต้องปลดเอง
pg_try_advisory_lock(key);             -- ไม่รอ คืนค่า true/false
pg_advisory_unlock(key);
pg_advisory_xact_lock(key);            -- transaction-level, ปลดอัตโนมัติเมื่อ commit/rollback

-- ตรวจสอบ lock ในระบบ
SELECT * FROM pg_locks;
SELECT * FROM pg_stat_activity;
SELECT pg_blocking_pids(pid);          -- หา pid ที่กำลังบล็อกอยู่

-- ควบคุมเวลารอ lock
SET lock_timeout = '2s';
SET statement_timeout = '30s';
SET idle_in_transaction_session_timeout = '5min';
```

### หลักการสำคัญที่ต้องจำ

1. **MVCC ทำให้ reader ไม่บล็อก writer และ writer ไม่บล็อก reader** — แต่ writer ยังต้องแย่ง lock กับ writer ด้วยกันเอง
2. **`UPDATE` = สร้างแถวใหม่ + mark แถวเก่าเป็น dead** — ต้องมี `VACUUM`/autovacuum มาเก็บกวาด ไม่งั้นจะเกิด table bloat
3. เลือก row-level lock ให้พอดี ไม่แรงเกินจำเป็น — `FOR UPDATE` ป้องกันได้ทุกกรณีแต่บล็อกมากที่สุด ควรเลือก `FOR NO KEY UPDATE`/`FOR SHARE`/`FOR KEY SHARE` เมื่อเหมาะสม
4. **`NOWAIT`** เมื่อต้องการตอบสนองเร็ว, **`SKIP LOCKED`** เมื่อทำ job queue ที่ยอมรับความไม่ deterministic ได้
5. คำสั่งที่ขอ `ACCESS EXCLUSIVE` (DDL ส่วนใหญ่) อันตรายบน production เพราะบล็อกทุกอย่าง รวมถึงทำให้เกิด **lock queue pile-up** ถ้ามี transaction ค้างอยู่ก่อน
6. **Advisory lock** เหมาะกับการ coordinate งานระดับแอปพลิเคชันที่ไม่ผูกกับแถว/ตารางใดโดยเฉพาะ — ใช้ transaction-scoped เมื่อเป็นไปได้เพื่อลดความเสี่ยงเรื่องลืมปลดล็อก

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1

จากคำอธิบายในบทนี้ อธิบายเป็นคำพูดของตัวเองว่าทำไม `SELECT` ธรรมดา (ไม่มี `FOR UPDATE`) จึงไม่เคยถูก `UPDATE` ของ transaction อื่นบล็อก แม้ตารางนั้นจะมี transaction ที่ `UPDATE` ค้างอยู่นานแค่ไหนก็ตาม

<details>
<summary>เฉลย</summary>

เพราะ PostgreSQL ใช้ MVCC ซึ่งเก็บหลายเวอร์ชันของแถวไว้พร้อมกัน เมื่อ transaction ที่ทำ `UPDATE` ยังไม่ `COMMIT` แถวเวอร์ชันเก่า (ที่มี `xmax` ยังไม่ถูก "ยืนยัน" ว่า committed) ยังคงถือว่า "มีชีวิต" อยู่สำหรับ transaction อื่นที่มาอ่าน ดังนั้น `SELECT` จะอ่านแถวเวอร์ชันเก่าได้ทันทีโดยไม่ต้องรอ ไม่จำเป็นต้องขอ lock ชนิดเดียวกับที่ `UPDATE` ถืออยู่ (`SELECT` ธรรมดาขอแค่ `ACCESS SHARE` ระดับตาราง และไม่ขอ row-level lock เลย)

</details>

### แบบฝึกหัดที่ 2

จากตาราง `products` ที่เตรียมไว้ ให้เขียนคำสั่งเพื่อดูค่า `xmin`, `xmax`, `ctid` ของแถว `product_id = 5` จากนั้น `UPDATE` แถวนั้น (เปลี่ยน `unit_price`) แล้วดูค่าเหล่านี้อีกครั้งพร้อมอธิบายว่าค่าใดเปลี่ยนและทำไม

<details>
<summary>เฉลย</summary>

```sql
-- ก่อน UPDATE
SELECT xmin, xmax, ctid, product_id, unit_price
FROM products WHERE product_id = 5;

UPDATE products SET unit_price = 1350.00 WHERE product_id = 5;

-- หลัง UPDATE
SELECT xmin, xmax, ctid, product_id, unit_price
FROM products WHERE product_id = 5;
```

`xmin` จะเปลี่ยนเป็น transaction ID ของคำสั่ง `UPDATE` (เพราะแถวใหม่ถูกสร้างขึ้นโดย transaction นี้) `ctid` จะเปลี่ยนตำแหน่งทางกายภาพ (เพราะเป็นแถวใหม่จริง ๆ ในระดับ storage ไม่ใช่การแก้ไขที่เดิม) ส่วน `xmax` ของแถวใหม่จะยังคงเป็น `0` เพราะยังไม่มีใครลบ/แทนที่แถวนี้ ในขณะที่แถวเก่า (ที่ไม่แสดงในผลลัพธ์อีกต่อไปเพราะมันกลายเป็น dead tuple) จะมี `xmax` ถูกตั้งเป็น transaction ID ของ `UPDATE` นี้แทน

</details>

### แบบฝึกหัดที่ 3

อธิบายความแตกต่างระหว่าง `FOR NO KEY UPDATE` กับ `FOR UPDATE` และยกตัวอย่างสถานการณ์ในระบบอีคอมเมิร์ซที่ควรใช้ `FOR NO KEY UPDATE` แทน `FOR UPDATE`

<details>
<summary>เฉลย</summary>

`FOR UPDATE` เป็น lock แรงสุด ใช้เมื่อจะแก้ไข/ลบแถวและอาจแตะ primary/unique key column ที่มี FK อ้างอิงถึง ส่วน `FOR NO KEY UPDATE` เข้มน้อยกว่าเล็กน้อย ใช้เมื่อจะ `UPDATE` แถวแต่**ไม่แตะ column ที่เป็น key** — ทำให้ transaction อื่นที่ทำ `FOR KEY SHARE` (เช่น ตรวจสอบ FK) ยังคงทำงานพร้อมกันได้โดยไม่บล็อกกัน

ตัวอย่าง: การอัปเดต `stock_quantity` ในตาราง `products` (ซึ่งไม่ใช่ key column) ควรใช้ `FOR NO KEY UPDATE` (ซึ่งจริง ๆ แล้ว PostgreSQL เลือกให้อัตโนมัติเมื่อใช้ `UPDATE products SET stock_quantity = ...` ตรง ๆ อยู่แล้ว) เพื่อไม่ไปบล็อก transaction อื่นที่กำลังตรวจสอบ referential integrity ผ่าน `product_id` ของแถวเดียวกันโดยไม่จำเป็น

</details>

### แบบฝึกหัดที่ 4

เปิดสองเซสชัน แล้วทดลองให้ Session A ทำ `SELECT ... FOR UPDATE` บนแถว `order_id = 7` โดยไม่ commit จากนั้นให้ Session B ลอง `UPDATE` แถวเดียวกันด้วย `NOWAIT` คาดเดาผลลัพธ์ที่จะเกิดขึ้น และเขียนคำสั่งทั้งหมด

<details>
<summary>เฉลย</summary>

```sql
-- Session A
BEGIN;
SELECT * FROM orders WHERE order_id = 7 FOR UPDATE;
-- ไม่ COMMIT

-- Session B
UPDATE orders SET status = 'cancelled' WHERE order_id = 7;
-- NOWAIT ใช้กับ SELECT เท่านั้น ไม่สามารถแนบกับ UPDATE ตรง ๆ ได้
-- วิธีทำ NOWAIT กับ UPDATE คือต้อง SELECT FOR UPDATE NOWAIT ก่อน แล้วค่อย UPDATE:
BEGIN;
SELECT * FROM orders WHERE order_id = 7 FOR UPDATE NOWAIT;
-- ERROR:  could not obtain lock on row in relation "orders"
-- เพราะ Session A ถือ lock (FOR UPDATE) ไว้อยู่ และยังไม่ COMMIT
ROLLBACK;
```

ผลลัพธ์คือ Session B จะได้รับ **error ทันที** (ไม่ต้องรอ) เพราะแถวนั้นถูกล็อกโดย Session A อยู่แล้ว และเราระบุ `NOWAIT` ไว้ชัดเจนว่าไม่ต้องการรอ

</details>

### แบบฝึกหัดที่ 5

ระบบ job queue ที่มี 3 worker กำลังแย่งกันดึงงานจากตาราง `orders` ที่ `status = 'pending'` โดยใช้ `SKIP LOCKED` เขียนคำสั่ง SQL ที่แต่ละ worker ควรใช้ และอธิบายว่าทำไมการใช้ `SKIP LOCKED` ในกรณีนี้ดีกว่าการใช้ `FOR UPDATE` เฉย ๆ (ไม่มี `SKIP LOCKED`)

<details>
<summary>เฉลย</summary>

```sql
BEGIN;
SELECT order_id FROM orders
WHERE status = 'pending'
ORDER BY order_id
FOR UPDATE SKIP LOCKED
LIMIT 1;
-- ประมวลผล แล้ว UPDATE status และ COMMIT
```

ถ้าใช้ `FOR UPDATE` เฉย ๆ โดยไม่มี `SKIP LOCKED`: worker ตัวที่ 2 และ 3 ที่พยายามดึงแถวเดียวกัน (แถวที่ query จะ match เป็นแถวแรกตามลำดับ `ORDER BY`) จะต้อง**รอ**จนกว่า worker ตัวแรกจะ commit ทำให้ทำงานแบบ**อนุกรม (serial)** ไม่ใช่ขนาน แม้ว่าจริง ๆ แล้วมีงานอื่นที่ว่างให้ทำอยู่ก็ตาม การใช้ `SKIP LOCKED` ทำให้ worker ที่มาทีหลังข้ามแถวที่ถูกล็อกไปเลือกแถวถัดไปที่ว่างได้ทันที ทำให้ทั้ง 3 worker ทำงานขนานกันได้อย่างแท้จริง เพิ่ม throughput ของระบบ

</details>

### แบบฝึกหัดที่ 6

อธิบายว่าทำไมการรัน `ALTER TABLE products ADD COLUMN category_id INTEGER;` บนตารางที่มี traffic สูงมาก อาจทำให้ระบบทั้งหมด "ค้าง" ชั่วขณะ แม้คำสั่งนี้เองจะทำงานเสร็จเร็วมากก็ตาม

<details>
<summary>เฉลย</summary>

`ALTER TABLE ADD COLUMN` ต้องขอ `ACCESS EXCLUSIVE` lock ซึ่งบล็อกทุกอย่างรวมถึง `SELECT` ธรรมดา แม้ตัวคำสั่งจะทำงานเสร็จเร็ว (เพราะตั้งแต่ PostgreSQL 11 การเพิ่มคอลัมน์ที่ไม่มี volatile default ไม่ต้อง rewrite ตารางทั้งหมด) แต่ปัญหาคือ **การขอ lock ต้องรอให้ transaction อื่นที่ถือ lock บนตารางนั้นอยู่ก่อน (แม้จะเป็นแค่ `ACCESS SHARE` จาก `SELECT`) ปล่อยก่อน** ถ้ามี transaction ใดก็ตามที่เปิดค้างไว้นาน (เช่น ลืม commit หรือ query ที่รันนาน) คำสั่ง `ALTER TABLE` จะต้อง "จองคิว" รออยู่ และในระหว่างที่รออยู่นั้น **query ใหม่ ๆ ทุกตัวที่ตามมาทีหลัง (แม้จะเป็นแค่ SELECT ที่ต้องการ ACCESS SHARE) ก็ต้องต่อคิวรอด้วย** เพราะ PostgreSQL จัดคิว lock แบบ FIFO นี่คือปรากฏการณ์ "lock queue pile-up" ที่ทำให้ระบบทั้งระบบดูเหมือนค้าง ทั้งที่คำสั่ง `ALTER TABLE` เองใช้เวลาสั้นมาก

</details>

### แบบฝึกหัดที่ 7

เขียนคำสั่ง SQL เพื่อหาว่า process ใด (pid) กำลังบล็อก process อื่นอยู่ในระบบปัจจุบัน พร้อม query ที่แต่ละฝ่ายกำลังรัน

<details>
<summary>เฉลย</summary>

```sql
SELECT
    blocked.pid       AS blocked_pid,
    blocked.query     AS blocked_query,
    blocking.pid      AS blocking_pid,
    blocking.query    AS blocking_query
FROM pg_stat_activity AS blocked
JOIN pg_stat_activity AS blocking
    ON blocking.pid = ANY(pg_blocking_pids(blocked.pid))
WHERE cardinality(pg_blocking_pids(blocked.pid)) > 0;
```

ฟังก์ชัน `pg_blocking_pids(pid)` คืนค่าเป็น array ของ pid ทั้งหมดที่กำลังบล็อก pid ที่ระบุอยู่ — join กับ `pg_stat_activity` สองครั้งเพื่อดึง query text ของทั้งฝ่ายที่ถูกบล็อกและฝ่ายที่บล็อก

</details>

### แบบฝึกหัดที่ 8

ให้เขียนฟังก์ชัน (หรือ pattern SQL) ที่ป้องกันไม่ให้ scheduled job สองตัวที่ชื่อ `'send_daily_report'` ทำงานซ้อนกัน โดยใช้ advisory lock แบบ transaction-level

<details>
<summary>เฉลย</summary>

```sql
CREATE OR REPLACE FUNCTION send_daily_report_safe()
RETURNS TEXT
LANGUAGE plpgsql
AS $$
BEGIN
    IF NOT pg_try_advisory_xact_lock(hashtext('send_daily_report')) THEN
        RETURN 'job รอบก่อนหน้ายังทำงานอยู่ ข้ามรอบนี้';
    END IF;

    -- ทำงานส่งรายงาน (อยู่ภายใน transaction เดียวกับที่ถือ lock)
    PERFORM pg_sleep(1);  -- จำลองงาน

    RETURN 'ส่งรายงานสำเร็จ';
    -- lock จะถูกปลดอัตโนมัติเมื่อ transaction จบ (COMMIT เมื่อฟังก์ชันจบการทำงานสำเร็จ)
END;
$$;

SELECT send_daily_report_safe();
```

การใช้ `pg_try_advisory_xact_lock` (แทน `pg_advisory_lock` แบบ session-level) มีข้อดีคือไม่ต้องกังวลเรื่องลืมปลดล็อก เพราะ PostgreSQL จะปลดให้อัตโนมัติทันทีที่ transaction จบไม่ว่าจะ commit หรือ rollback หรือแม้แต่ connection หลุดกลางคัน

</details>

### แบบฝึกหัดที่ 9

ตารางใน compatibility matrix ของ table-level lock (Step 577) แสดงว่า `SHARE UPDATE EXCLUSIVE` เข้ากันได้กับ `ROW EXCLUSIVE` หรือไม่ ให้อธิบายว่าเหตุใดคุณสมบัตินี้จึงทำให้ `CREATE INDEX CONCURRENTLY` เหมาะกับการใช้งานบนตาราง production ที่มี traffic สูง

<details>
<summary>เฉลย</summary>

จากตาราง compatibility matrix, `SHARE UPDATE EXCLUSIVE` (แถวที่ 4) กับ `ROW EXCLUSIVE` (คอลัมน์ที่ 3) ตัดกันเป็น ✅ (เข้ากันได้) หมายความว่า `CREATE INDEX CONCURRENTLY` (ที่ขอ `SHARE UPDATE EXCLUSIVE`) สามารถทำงานพร้อมกันกับคำสั่ง `INSERT`/`UPDATE`/`DELETE` ปกติ (ที่ขอ `ROW EXCLUSIVE`) ได้โดยไม่บล็อกกัน นี่คือเหตุผลที่ `CREATE INDEX CONCURRENTLY` ถูกออกแบบมาให้ "ไม่บล็อกการเขียนข้อมูลปกติ" ระหว่างสร้าง index แม้จะใช้เวลานานกว่า `CREATE INDEX` ธรรมดา (ที่ขอ `SHARE` ซึ่งบล็อก `ROW EXCLUSIVE` ทันที) แต่แลกมาด้วยการไม่มี downtime ซึ่งเหมาะกับตารางที่มี traffic การเขียนสูงในระบบ production ตลอดเวลา

</details>

### แบบฝึกหัดที่ 10 (ประยุกต์)

จากระบบจองที่นั่งใน Step 580 ให้ปรับปรุงฟังก์ชัน `book_seat` ให้รองรับกรณีที่สินค้าหนึ่งตัวมีได้หลายไซส์/หลายตัวเลือก (variant) โดยยังคงป้องกัน race condition ได้ พร้อมอธิบายแนวคิดโดยย่อ (ไม่จำเป็นต้องเขียน schema เพิ่มเต็มรูปแบบ)

<details>
<summary>เฉลย</summary>

แนวคิดคือยังคงหลักการเดิม: **ล็อกแถวที่เป็นหน่วยของสต็อกจริง (stock unit) ด้วย `FOR UPDATE`/`FOR UPDATE NOWAIT` เสมอ ก่อนตรวจสอบและลดจำนวน** ไม่ว่าจะมีกี่ variant ก็ตาม เพียงแต่ต้องออกแบบ schema ให้แต่ละ variant (เช่น สี, ไซส์) เป็น**แถวแยกกัน** ในตาราง (เช่น ตาราง `product_variants` ที่มี `variant_id`, `product_id`, `size`, `stock_quantity` ของตัวเอง) เพื่อให้การล็อกเกิดขึ้นเฉพาะที่ระดับ variant นั้น ๆ ไม่ใช่ล็อกทั้งแถว `products` หลัก ซึ่งจะทำให้ variant อื่นของสินค้าเดียวกัน (เช่น ไซส์ M กับไซส์ L ของเสื้อตัวเดียวกัน) ยังคงถูกจองพร้อมกันได้โดยไม่ต้องรอกัน ตัวอย่าง pattern:

```sql
-- สมมติมีตาราง product_variants(variant_id, product_id, size, stock_quantity)
SELECT stock_quantity FROM product_variants
WHERE variant_id = 42
FOR UPDATE NOWAIT;
-- ตรวจสอบและลดสต็อกเฉพาะ variant_id = 42 เท่านั้น
-- variant_id อื่น ๆ ของ product_id เดียวกันไม่ถูกกระทบ
```

หลักการสำคัญคือ **การล็อกควรเกิดขึ้นที่ระดับ "หน่วยที่เล็กที่สุดที่มีการแย่งชิงทรัพยากรจริง"** (finest-grained lock ที่ยังถูกต้องทางธุรกิจ) เพื่อลด lock contention ให้น้อยที่สุดเท่าที่จะทำได้ โดยไม่กระทบความถูกต้องของข้อมูล

</details>

---

## บทถัดไป

เมื่อสอง transaction ล็อกทรัพยากรของกันและกันแบบ**วนกลับ** (A รอ B, ในขณะที่ B ก็รอ A) ระบบจะเข้าสู่ภาวะ **deadlock** ซึ่ง PostgreSQL มีกลไกตรวจจับและจัดการโดยอัตโนมัติ — เนื้อหาเรื่องการวิเคราะห์ log ของ deadlock, การออกแบบลำดับการล็อกเพื่อป้องกัน deadlock ตั้งแต่ต้น และเทคนิค retry ที่ปลอดภัย จะอยู่ใน **[Part 059: Deadlocks](./part-059-deadlocks.md)**
