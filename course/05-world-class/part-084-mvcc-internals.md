# MVCC Internals เชิงลึก: Tuple Visibility, xmin/xmax

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับโลก/ผู้เชี่ยวชาญ | Part 084

---

## เป้าหมายการเรียนรู้

เมื่อจบบทนี้ ผู้เรียนจะสามารถ:

1. อธิบาย MVCC (Multi-Version Concurrency Control) ในระดับ **แถวข้อมูลจริงบนดิสก์** ไม่ใช่แค่ระดับแนวคิด
2. อ่านและตีความ system columns `xmin`, `xmax`, `ctid` ที่ PostgreSQL แนบมากับทุกแถวในทุกตาราง
3. เข้าใจว่า Transaction ID (XID) คืออะไร ทำไมเป็นเลข 32-bit และทำไมต้องกังวลเรื่อง wraparound
4. เข้าใจกฎ **Tuple Visibility** ที่ PostgreSQL ใช้ตัดสินว่าแถวไหน "มองเห็นได้" สำหรับ transaction หนึ่งๆ
5. เข้าใจว่า Snapshot คืออะไร มี `xmin`, `xmax`, `xip_list` ของตัวเองอย่างไร และเชื่อมโยงกับ Isolation Level
6. เห็นด้วยตาตัวเองว่า `INSERT`, `UPDATE`, `DELETE` แต่ละคำสั่ง **ทำอะไรกับ tuple จริงๆ** ผ่านการสังเกตค่า `xmin`/`xmax`/`ctid` ที่เปลี่ยนไป
7. เข้าใจว่าทำไม `UPDATE` ไม่ได้ "แก้ไขแถวเดิม" แต่สร้างแถวใหม่ และทำไม `DELETE` ไม่ได้ "ลบข้อมูลทันที"
8. เข้าใจกลไก Heap-Only Tuple (HOT) update และประโยชน์ในการลดภาระ index maintenance
9. เชื่อมโยงความรู้นี้เข้ากับ VACUUM (Part 060) และ Transaction ID Wraparound (Part 060) ที่เคยเรียนมาก่อน

บทนี้เป็นบทที่ **ลึกที่สุด** เกี่ยวกับ MVCC ในหลักสูตรทั้งหมด ต่อยอดจาก Part 058 ที่แนะนำ MVCC ในระดับภาพรวม (แถวหลายเวอร์ชัน, Isolation Level, การไม่ล็อกอ่าน) มาสู่ระดับ **internals จริง** ที่วิศวกรฐานข้อมูลระดับโลกต้องเข้าใจ — สิ่งที่เกิดขึ้นจริงบนดิสก์เมื่อคุณรัน `INSERT`/`UPDATE`/`DELETE` หนึ่งคำสั่ง

---

## เตรียมข้อมูล

ตลอดบทนี้เราจะใช้ตารางเดียวกันทั้งหมด เพื่อให้ผลลัพธ์ทุกตัวอย่างเปรียบเทียบกันได้ง่าย:

```sql
DROP TABLE IF EXISTS accounts;

CREATE TABLE accounts (
    account_id  SERIAL PRIMARY KEY,
    owner_name  VARCHAR(100),
    balance     NUMERIC(12,2)
);

INSERT INTO accounts (owner_name, balance) VALUES
    ('Somchai',  10000.00),
    ('Malee',     5000.00),
    ('Anan',     15000.00);
```

> **หมายเหตุสำคัญ**: ค่า `xmin`, `xmax`, `ctid` ที่ปรากฏในตัวอย่างของบทนี้เป็น **ค่าตัวอย่างที่สังเกตได้จริงในสภาพแวดล้อมทดสอบ** แต่ตัวเลข XID จริงในเครื่องของผู้เรียนจะไม่ตรงกันเป๊ะ (ขึ้นกับว่ามี transaction อะไรวิ่งมาก่อนหน้าบ้าง) — **รูปแบบและพฤติกรรมของการเปลี่ยนแปลง** ต่างหากที่เป็นสาระสำคัญที่ต้องสังเกต ให้รันตามจริงในเครื่องของตัวเองควบคู่ไปด้วย

ตรวจสอบว่าตารางสร้างสำเร็จและมี system columns ให้ดูจริง:

```sql
SELECT xmin, xmax, ctid, * FROM accounts ORDER BY account_id;
```

ผลลัพธ์ตัวอย่าง (ตัวเลข XID ขึ้นกับสถานะเครื่องของคุณ):

```
 xmin | xmax | ctid  | account_id | owner_name | balance
------+------+-------+------------+------------+----------
  742 |    0 | (0,1) |          1 | Somchai    | 10000.00
  742 |    0 | (0,2) |          2 | Malee      |  5000.00
  742 |    0 | (0,3) |          3 | Anan       | 15000.00
(3 rows)
```

สังเกตว่าทั้งสามแถวมี `xmin` เท่ากัน (`742`) เพราะถูก INSERT ด้วย transaction เดียวกัน (statement เดียวที่ insert 3 แถวพร้อมกัน) และ `xmax = 0` หมายถึง "ยังไม่ถูกลบ/แก้ไข" ส่วน `ctid` คือตำแหน่งจริงบนดิสก์ในรูปแบบ `(block_number, tuple_index)`

---

## Step 831: ทบทวน MVCC โดยสรุปเชิงลึก — แต่ละแถวคือหนึ่งใน "หลายเวอร์ชัน" ที่มีอยู่จริง

ใน Part 058 เราเรียนรู้แนวคิด MVCC ในระดับภาพรวมว่า PostgreSQL ไม่ล็อกการอ่านด้วยการเขียน (readers don't block writers, writers don't block readers) เพราะแต่ละ transaction เห็น "snapshot" ของข้อมูล ณ จุดเวลาหนึ่ง

สิ่งที่บทนี้จะพาไปดูลึกกว่านั้นคือ **PostgreSQL ทำสิ่งนี้ได้อย่างไรในระดับไฟล์ดิสก์จริง**

หัวใจของ MVCC ใน PostgreSQL คือ:

> **PostgreSQL ไม่เคยแก้ไขข้อมูลแถวเดิม (in-place update) เมื่อคุณ `UPDATE` หรือ `DELETE`** แต่จะสร้าง **tuple เวอร์ชันใหม่** ขึ้นมา (สำหรับ UPDATE) หรือทำเครื่องหมายว่า tuple เดิม "ตายแล้ว" (สำหรับ DELETE) โดยที่ **tuple เวอร์ชันเก่ายังคงอยู่บนดิสก์จริงๆ** จนกว่า VACUUM จะมาเก็บกวาดทิ้ง

พูดง่ายๆ คือ ตาราง `accounts` ที่คุณเห็นว่ามี 3 แถวในผลลัพธ์ `SELECT * FROM accounts` นั้น **ไฟล์บนดิสก์อาจมี tuple มากกว่า 3 ชุด** ซ้อนกันอยู่ — บางชุดเป็นเวอร์ชันเก่าที่ "ตายแล้ว" (dead tuple) แต่ยังไม่ถูกลบจริง

นี่คือเหตุผลที่มาของคำว่า **Multi-Version** ใน MVCC: ไม่ใช่แค่แนวคิดนามธรรม แต่หมายถึง **หลายแถวจริงๆ ที่กินพื้นที่จริงบนดิสก์จริง** อยู่ในไฟล์ heap เดียวกัน ณ เวลาเดียวกัน

```sql
-- ทดลองดูจำนวน tuple จริงที่มีอยู่ในตาราง (ทั้ง live และ dead)
-- ก่อนทำอะไรเพิ่มเติม ลองดูสถิติปัจจุบัน
SELECT relname, n_live_tup, n_dead_tup
FROM pg_stat_user_tables
WHERE relname = 'accounts';
```

```
 relname  | n_live_tup | n_dead_tup
----------+------------+------------
 accounts |          3 |          0
(1 row)
```

ตอนนี้ `n_dead_tup = 0` เพราะยังไม่มีการ UPDATE/DELETE ใดๆ เกิดขึ้น เดี๋ยวใน Step ถัดๆ ไป เราจะเห็นตัวเลขนี้เปลี่ยนแปลงจริงเมื่อเราทำ UPDATE/DELETE

**สิ่งที่ต้องจำ**: ทุกแถวที่คุณเห็นในผลลัพธ์ query คือ "หนึ่งในหลายเวอร์ชัน" ที่ถูกกรองมาแล้วโดยกฎ Visibility (Step 834) — มันไม่ใช่ "ข้อมูลปัจจุบันเพียงชุดเดียว" ในความหมายที่ว่าไฟล์มีข้อมูลแค่นั้น แต่มันคือ **เวอร์ชันที่ transaction ของคุณ ณ ตอนนี้ ควรจะเห็น**

---

## Step 832: System Columns — xmin, xmax, ctid

PostgreSQL แนบ **system columns** (คอลัมน์ระบบ) ให้กับทุกตารางโดยอัตโนมัติ โดยไม่ต้องประกาศเอง คอลัมน์เหล่านี้ไม่ปรากฏใน `SELECT *` ตามปกติ แต่สามารถเรียกดูได้ชัดเจนด้วยการระบุชื่อ

คอลัมน์ที่สำคัญที่สุดสำหรับ MVCC มีสามตัว:

| คอลัมน์ | ความหมาย |
|---|---|
| `xmin` | Transaction ID ของ transaction ที่ **สร้าง** tuple นี้ (INSERT หรือ UPDATE ที่สร้างเวอร์ชันใหม่) |
| `xmax` | Transaction ID ของ transaction ที่ **ลบหรือแทนที่** tuple นี้ (DELETE หรือ UPDATE ที่ทำให้ tuple นี้กลายเป็นเวอร์ชันเก่า) — ค่า `0` หมายถึงยังไม่ถูกลบ/แทนที่ |
| `ctid` | ตำแหน่งจริงของ tuple บนดิสก์ ในรูปแบบ `(block_number, item_offset)` — เปลี่ยนทุกครั้งที่มีการสร้าง tuple ใหม่ทางกายภาพ |

นอกจากนี้ยังมีคอลัมน์ระบบอื่นๆ ที่เกี่ยวข้อง (แต่ใช้บ่อยน้อยกว่าในงานประจำวัน):

- `cmin` / `cmax` — command ID ภายใน transaction เดียวกัน (แยกความแตกต่างระหว่างหลาย statement ใน transaction เดียว)
- `tableoid` — OID ของตารางที่แถวนี้อยู่ (มีประโยชน์เมื่อ query จาก inherited table หรือ partition)

ลองดูค่าจริงทั้งหมด:

```sql
SELECT xmin, xmax, cmin, cmax, ctid, tableoid::regclass, *
FROM accounts
ORDER BY account_id;
```

```
 xmin | xmax | cmin | cmax | ctid  | tableoid | account_id | owner_name | balance
------+------+------+------+-------+----------+------------+------------+----------
  742 |    0 |    0 |    0 | (0,1) | accounts |          1 | Somchai    | 10000.00
  742 |    0 |    0 |    0 | (0,2) | accounts |          2 | Malee      |  5000.00
  742 |    0 |    0 |    0 | (0,3) | accounts |          3 | Anan       | 15000.00
(3 rows)
```

**ประเด็นสำคัญ**: `xmin`, `xmax`, `ctid` **ไม่ใช่คอลัมน์ที่ประกาศไว้ใน `CREATE TABLE`** — มันเป็นสิ่งที่ storage engine ของ PostgreSQL (heap access method) แนบมาให้กับทุก tuple โดยอัตโนมัติ เป็นส่วนหนึ่งของ **tuple header** (โครงสร้าง `HeapTupleHeaderData`) ที่อยู่ก่อนข้อมูลจริงของแถวในทุก tuple

คุณสามารถตรวจสอบด้วย `\d accounts` ใน psql และจะไม่เห็นคอลัมน์เหล่านี้ในรายการเลย เพราะมันไม่ใช่ "user column" แต่เป็นเมทาดาทาระดับ storage:

```sql
\d accounts
```

```
                                     Table "public.accounts"
   Column   |          Type          | Collation | Nullable |              Default
------------+-------------------------+-----------+----------+-------------------------------------
 account_id | integer                 |           | not null | nextval('accounts_account_id_seq...
 owner_name | character varying(100)  |           |          |
 balance    | numeric(12,2)           |           |          |
Indexes:
    "accounts_pkey" PRIMARY KEY, btree (account_id)
```

จะไม่เห็น `xmin`/`xmax`/`ctid` ในนี้เลย — เพราะมันคือ *system columns* ที่มีอยู่ใน **ทุกตารางเสมอ** โดยไม่ต้องประกาศ

**ทดสอบเชิงประจักษ์**: ลองใช้กับตารางอื่นใดก็ได้ในฐานข้อมูล จะพบว่า `xmin`/`xmax`/`ctid` มีอยู่เสมอ — นี่คือกลไกพื้นฐานที่สุดของ MVCC ใน PostgreSQL

---

## Step 833: Transaction ID (XID) — เลขที่เพิ่มขึ้นเรื่อยๆ และปัญหา wraparound

ค่า `xmin`/`xmax` ที่เราเห็นคือ **Transaction ID (XID)** — ตัวเลขที่ PostgreSQL แจกให้กับทุก transaction ที่ "เขียน" ข้อมูล (transaction ที่อ่านอย่างเดียวโดยทั่วไปจะไม่ถูกแจก XID จริงจนกว่าจะต้องเขียนบางอย่าง)

ดู XID ของ transaction ปัจจุบันได้ด้วยฟังก์ชัน `txid_current()`:

```sql
BEGIN;
SELECT txid_current();
```

```
 txid_current
--------------
          748
```

```sql
COMMIT;
```

ลองรันใหม่อีกครั้งในอีก transaction หนึ่ง:

```sql
BEGIN;
SELECT txid_current();
COMMIT;
```

```
 txid_current
--------------
          749
```

จะเห็นว่า **XID เพิ่มขึ้นทีละ 1 ทุกครั้งที่มี transaction ใหม่ที่ต้องเขียนข้อมูล** — เป็นตัวนับ (counter) แบบ global ที่ใช้ร่วมกันทั้งฐานข้อมูล ไม่ใช่เฉพาะตาราง

### ทำไมต้องเป็น 32-bit?

XID ภายในระบบ (ค่าที่เก็บใน `xmin`/`xmax` จริงๆ บนดิสก์) เป็นจำนวนเต็มแบบ **32-bit unsigned integer** ซึ่งหมายความว่ามีค่าได้ตั้งแต่ 0 ถึงประมาณ 4.2 พันล้าน (2^32)

> หมายเหตุ: `txid_current()` คืนค่าแบบ 64-bit (`bigint`) เพื่อความสะดวกในการเปรียบเทียบข้ามช่วง wraparound แต่ภายใน tuple header จริงบนดิสก์ ค่า `xmin`/`xmax` ยังคงเป็น 32-bit เสมอ นี่คือเหตุผลที่ระบบมีกลไก "epoch" ซ่อนอยู่เบื้องหลัง `txid_current()`

ตรวจสอบขนาดพื้นที่ที่ XID เก็บจริงในตาราง:

```sql
SELECT xmin, xmax
FROM accounts
LIMIT 1;
```

ค่า `xmin` ที่เห็น เช่น `742` คือ 32-bit unsigned integer ล้วนๆ — **ไม่มี epoch/timestamp ติดมาด้วยในคอลัมน์นี้**

### ปัญหา Wraparound (เชื่อมโยง Part 060)

เนื่องจาก XID เป็น 32-bit ทำให้มีค่าจำกัดอยู่ที่ประมาณ 4.29 พันล้านค่า เมื่อระบบสร้าง transaction ไปเรื่อยๆ จนตัวเลขใกล้ถึงขีดจำกัด **ตัวเลขจะวนกลับมาเริ่มใหม่ (wraparound)** ซึ่งจะทำให้เกิดปัญหาร้ายแรง: transaction เก่าที่เคย "อยู่ในอดีต" (XID น้อยกว่า) อาจถูกตีความผิดว่า "อยู่ในอนาคต" (เพราะตัวเลขวนกลับมาน้อยกว่าปัจจุบัน) ทำให้ข้อมูลที่ควร visible กลับกลายเป็น invisible โดยไม่คาดคิด

นี่คือเหตุผลที่ PostgreSQL มีกลไก **Freezing** — เมื่อ tuple แก่พอ (เก่ากว่า `vacuum_freeze_min_age`) VACUUM จะเขียน tuple header ใหม่โดยเปลี่ยนค่า `xmin` ให้เป็นค่าพิเศษ `FrozenTransactionId` (ซึ่งถือว่า "เก่าที่สุดเสมอ ไม่ต้องเทียบกับตัวนับ XID อีกต่อไป")

ตรวจสอบอายุของ transaction ปัจจุบันเทียบกับ wraparound limit ของตาราง:

```sql
SELECT relname,
       age(relfrozenxid) AS xid_age,
       relfrozenxid
FROM pg_class
WHERE relname = 'accounts';
```

```
 relname  | xid_age | relfrozenxid
----------+---------+--------------
 accounts |      12 |          736
```

`age(relfrozenxid)` บอกว่า tuple ที่เก่าที่สุดในตาราง (ที่ยังไม่ถูก freeze) มีอายุห่างจาก transaction ปัจจุบันกี่ transaction — ถ้าตัวเลขนี้เข้าใกล้ `autovacuum_freeze_max_age` (ค่า default 200 ล้าน) ระบบจะบังคับให้ autovacuum ทำงานแบบ aggressive เพื่อ freeze tuple เก่าๆ ก่อนที่จะเกิด wraparound

> **เชื่อมโยง Part 060**: บทนั้นได้อธิบายรายละเอียดของ VACUUM และ wraparound protection ไว้แล้วในระดับปฏิบัติการ (monitoring, tuning `autovacuum_freeze_max_age` ฯลฯ) บทนี้เพียงแสดงให้เห็นว่า **ตัวเลข XID ที่มองไม่เห็นในบทนั้น จริงๆ แล้วคือค่าเดียวกับที่อยู่ใน `xmin`/`xmax` ที่เราเห็นอยู่ตรงหน้านี้เอง**

---

## Step 834: Tuple Visibility Rules — กฎการตัดสินว่าแถวไหน "มองเห็นได้"

นี่คือหัวใจที่แท้จริงของ MVCC: PostgreSQL ตัดสินว่า tuple หนึ่งๆ "มองเห็นได้" (visible) สำหรับ transaction ปัจจุบันหรือไม่ โดยเปรียบเทียบ `xmin`/`xmax` ของ tuple กับ **snapshot** ของ transaction นั้น

กฎแบบง่าย (simplified visibility rule) มีดังนี้:

```
tuple นี้ "มองเห็นได้" ก็ต่อเมื่อ:

1. xmin ของ tuple ต้อง "committed" แล้ว  AND
2. xmin ของ tuple ต้องเกิดขึ้น "ก่อน" snapshot ของ transaction ปัจจุบัน (ไม่ใช่ transaction ที่ยังไม่ commit ณ ตอนสร้าง snapshot)  AND
3. (xmax ของ tuple = 0)  OR  (xmax ยังไม่ commit)  OR  (xmax เกิดขึ้น "หลัง" snapshot ของ transaction ปัจจุบัน)
```

พูดเป็นภาษาคน: tuple จะ visible ถ้า **"คนที่สร้างมัน" commit ไปแล้วก่อนที่เราจะเริ่มดู** และ **"คนที่ลบ/แทนที่มัน" ยังไม่ commit หรือยังไม่ได้ลบเลย**

ลองพิสูจน์ด้วยการทดลองสองหน้าต่าง (Session A และ Session B) เพื่อดูกฎนี้ทำงานจริง:

**Session A:**
```sql
BEGIN;
SELECT txid_current();  -- สมมติได้ 750
```

```
 txid_current
--------------
          750
```

**Session B (แยกหน้าต่างใหม่ อยู่นอก transaction ของ A):**
```sql
UPDATE accounts SET balance = balance + 500 WHERE account_id = 1;
COMMIT;  -- (autocommit อยู่แล้วถ้าไม่ได้เปิด BEGIN)

SELECT xmin, xmax, ctid, * FROM accounts WHERE account_id = 1;
```

```
 xmin | xmax | ctid  | account_id | owner_name | balance
------+------+-------+------------+------------+----------
  751 |    0 | (0,4) |          1 | Somchai    | 10500.00
```

**กลับมาที่ Session A (ยังอยู่ใน transaction เดิม ยังไม่ commit):**
```sql
SELECT xmin, xmax, ctid, * FROM accounts WHERE account_id = 1;
COMMIT;
```

ถ้า Session A ใช้ **Repeatable Read** หรือ **Serializable** isolation level ผลลัพธ์จะยังเป็นค่าเก่า:

```
 xmin | xmax | ctid  | account_id | owner_name | balance
------+------+-------+------------+------------+----------
  742 |  751 | (0,1) |          1 | Somchai    | 10000.00
```

สังเกตให้ดี: **tuple เก่า (`xmin=742`, `ctid=(0,1)`) ยังคงอยู่บนดิสก์และยังคง visible สำหรับ Session A** เพราะ snapshot ของ Session A ถูกสร้างตอน XID = 750 ซึ่งเกิดขึ้น **ก่อน** transaction 751 ของ Session B ที่ทำ UPDATE — กฎ visibility ข้อ 3 บอกว่า "xmax ยังไม่ commit ณ ตอน snapshot ถูกสร้าง" หรือพูดให้แม่นคือ "751 ยังไม่ได้เกิดในมุมมองของ snapshot ที่ XID=750" ดังนั้น tuple ที่ถูก `xmax=751` มาร์กไว้ ยังคง **visible** สำหรับ transaction 750

แต่ถ้า Session A ใช้ **Read Committed** (ค่า default ของ PostgreSQL) แต่ละ statement จะสร้าง snapshot ใหม่ ดังนั้นการ `SELECT` ครั้งที่สองใน Session A (หลัง Session B commit ไปแล้ว) จะเห็นค่าใหม่ทันที:

```
 xmin | xmax | ctid  | account_id | owner_name | balance
------+------+-------+------------+------------+----------
  751 |    0 | (0,4) |          1 | Somchai    | 10500.00
```

นี่คือหลักฐานที่จับต้องได้ว่า **Isolation Level ไม่ใช่แค่คำอธิบายเชิงทฤษฎี แต่คือกฎที่กำหนดว่า snapshot ไหนถูกใช้เปรียบเทียบกับ `xmin`/`xmax` ในการตัดสิน visibility จริงๆ**

---

## Step 835: Snapshot คืออะไร — ภาพรวมของ transaction ที่ commit แล้ว ณ จุดเวลาหนึ่ง

**Snapshot** คือโครงสร้างข้อมูลที่ PostgreSQL สร้างขึ้นเมื่อเริ่ม transaction (หรือเริ่ม statement ใน Read Committed) เพื่อบันทึกว่า **ณ ขณะนั้น transaction ไหนบ้างที่ถือว่า "committed แล้ว" และไหนบ้างที่ "ยังไม่ commit"**

Snapshot ประกอบด้วยองค์ประกอบหลักสามส่วน:

| องค์ประกอบ | ความหมาย |
|---|---|
| `xmin` (ของ snapshot) | XID ที่เล็กที่สุดที่ยังทำงานอยู่ (running) ขณะสร้าง snapshot — transaction ใดที่มี XID น้อยกว่านี้ถือว่า committed แน่นอนแล้ว |
| `xmax` (ของ snapshot) | XID ถัดไปที่ยังไม่ถูกแจกใช้ ณ ตอนสร้าง snapshot — transaction ใดที่มี XID มากกว่าหรือเท่านี้ถือว่ายังไม่เริ่ม (อนาคต) |
| `xip_list` | รายการ XID ที่ **กำลังทำงานอยู่ (in-progress)** ณ ตอนสร้าง snapshot — แม้ XID จะอยู่ระหว่าง `xmin` กับ `xmax` แต่ถ้าอยู่ใน `xip_list` แสดงว่ายัง**ไม่ commit** |

ฟังก์ชัน `pg_current_snapshot()` (PostgreSQL 13 ขึ้นไป) ให้เราดู snapshot ปัจจุบันได้โดยตรง:

```sql
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT pg_current_snapshot();
```

```
   pg_current_snapshot
---------------------------
 752:755:752,753
```

รูปแบบผลลัพธ์คือ `xmin:xmax:xip_list` อ่านได้ว่า:

- `xmin = 752` → transaction ใดที่ XID < 752 ถือว่า committed แน่นอน (visible ถ้าไม่ถูกลบ)
- `xmax = 755` → transaction ใดที่ XID >= 755 ถือว่ายังไม่เริ่ม (ในอนาคต, ไม่ visible)
- `xip_list = 752,753` → transaction 752 และ 753 กำลังทำงานอยู่ (in-progress) ณ ตอนสร้าง snapshot แม้จะอยู่ในช่วง `[xmin, xmax)` ก็ถือว่า **ไม่ commit** ดังนั้น**ไม่ visible**

ส่วน transaction 754 (ซึ่งอยู่ในช่วง `[752, 755)` แต่ไม่อยู่ใน `xip_list`) ถือว่า **committed แล้ว** และ visible

```sql
COMMIT;
```

สามารถดูรายละเอียด XID ที่ committed หรือไม่ ด้วยฟังก์ชันช่วยตรวจสอบ:

```sql
SELECT
    pg_visible_in_snapshot('751'::xid8, pg_current_snapshot()) AS is_751_visible;
```

> หมายเหตุ: ในเวอร์ชันเก่ากว่า PostgreSQL 13 จะใช้ `txid_current_snapshot()` แทน ซึ่งให้รูปแบบผลลัพธ์เดียวกัน

### เชื่อมโยง Snapshot กับ Isolation Level

- **Read Committed** (default): สร้าง snapshot ใหม่ **ทุก statement** — เห็นข้อมูลล่าสุดที่ commit แล้ว ณ ตอนเริ่ม statement นั้น
- **Repeatable Read**: สร้าง snapshot **ครั้งเดียวตอนเริ่ม transaction** และใช้ snapshot เดิมตลอด transaction — เห็นข้อมูลเหมือนเดิมทุก statement แม้ transaction อื่นจะ commit ระหว่างทาง
- **Serializable**: ใช้กลไก snapshot แบบเดียวกับ Repeatable Read บวกกับการตรวจจับ serialization anomaly เพิ่มเติม

นี่คือเหตุผลเชิง internals ที่แท้จริงว่าทำไม Repeatable Read ถึง "เห็นข้อมูลคงที่ตลอด transaction" — เพราะ **snapshot ตัวเดียวถูกใช้เปรียบเทียบกับ `xmin`/`xmax` ของทุก tuple ตลอดทั้ง transaction** ไม่ใช่เพราะ PostgreSQL "ล็อกข้อมูล" หรือ "copy ข้อมูล" ไว้ที่ไหน

---

## Step 836: INSERT ทำอะไรจริงๆ — สร้าง tuple ใหม่พร้อม xmin = current XID

มาดูกันทีละขั้นว่า `INSERT` ทำอะไรกับ heap file จริงๆ

```sql
SELECT txid_current();
```

```
 txid_current
--------------
          756
```

```sql
INSERT INTO accounts (owner_name, balance) VALUES ('Preecha', 20000.00);

SELECT xmin, xmax, ctid, * FROM accounts WHERE owner_name = 'Preecha';
```

```
 xmin | xmax | ctid  | account_id | owner_name | balance
------+------+-------+------------+------------+----------
  756 |    0 | (0,5) |          4 | Preecha    | 20000.00
```

สิ่งที่เกิดขึ้นจริง:

1. PostgreSQL แจก XID ใหม่ให้ transaction นี้ (`756`)
2. สร้าง tuple ใหม่ทางกายภาพในไฟล์ heap ที่ตำแหน่ง `ctid = (0,5)` (block 0, offset 5)
3. เขียนค่า `xmin = 756` ลงใน tuple header — บอกว่า "transaction 756 คือผู้สร้าง tuple นี้"
4. เขียนค่า `xmax = 0` — บอกว่า "ยังไม่มีใครลบ/แทนที่ tuple นี้"
5. คัดลอกข้อมูลคอลัมน์จริง (`account_id`, `owner_name`, `balance`) ลงในส่วน data ของ tuple

ทันทีที่ transaction commit เรียบร้อย `xmin=756` จะถูกถือว่า "committed" และ tuple นี้จะ visible สำหรับ transaction อื่นที่มี snapshot ซึ่ง `xmin` ของ snapshot มากกว่า 756 (หรือ 756 committed แล้วตอนสร้าง snapshot)

**ทดลองเปรียบเทียบ INSERT หลายแถวในหลาย statement:**

```sql
INSERT INTO accounts (owner_name, balance) VALUES ('Wichai', 8000.00);
INSERT INTO accounts (owner_name, balance) VALUES ('Sunee', 12000.00);

SELECT xmin, xmax, ctid, * FROM accounts
WHERE owner_name IN ('Wichai', 'Sunee');
```

```
 xmin | xmax | ctid  | account_id | owner_name | balance
------+------+-------+------------+------------+----------
  757 |    0 | (0,6) |          5 | Wichai     |  8000.00
  758 |    0 | (0,7) |          6 | Sunee      | 12000.00
```

สังเกตว่า `xmin` **ต่างกัน** ระหว่างสองแถวนี้ (757 กับ 758) เพราะแต่ละ `INSERT` เป็นคนละ transaction (เนื่องจากอยู่ใน autocommit mode คนละ statement) แต่ถ้าเรารวมอยู่ใน `BEGIN...COMMIT` เดียวกัน ทั้งสองแถวจะได้ `xmin` เดียวกัน เพราะเป็น transaction เดียวกัน แม้จะเป็นคนละ statement ก็ตาม (แต่ `cmin` จะต่างกัน):

```sql
BEGIN;
INSERT INTO accounts (owner_name, balance) VALUES ('Kamon', 3000.00);
INSERT INTO accounts (owner_name, balance) VALUES ('Nid', 7000.00);

SELECT xmin, cmin, xmax, ctid, * FROM accounts
WHERE owner_name IN ('Kamon', 'Nid');
COMMIT;
```

```
 xmin | cmin | xmax | ctid  | account_id | owner_name | balance
------+------+------+-------+------------+------------+---------
  759 |    0 |    0 | (0,8) |          7 | Kamon      | 3000.00
  759 |    1 |    0 | (0,9) |          8 | Nid        | 7000.00
```

นี่คือหลักฐานที่แสดงชัดเจนว่า `xmin` ผูกกับ **transaction** ไม่ใช่ **statement** ในขณะที่ `cmin` ผูกกับ **command ลำดับที่เท่าไรภายใน transaction นั้น** (statement แรกในทรานแซคชันคือ `cmin=0`, statement ที่สองคือ `cmin=1`)

---

## Step 837: UPDATE ทำอะไรจริงๆ — mark tuple เดิมด้วย xmax แล้วสร้าง tuple ใหม่

นี่คือส่วนที่ผู้เรียนจำนวนมากเข้าใจผิดมากที่สุด: `UPDATE` **ไม่ได้แก้ไขข้อมูลในตำแหน่งเดิมบนดิสก์ (in-place)** แต่ทำสองสิ่งนี้:

1. **Mark tuple เดิม**: เขียนค่า `xmax = current_XID` ลงใน tuple เดิม (บอกว่า "transaction นี้ทำให้ tuple นี้กลายเป็นเวอร์ชันเก่าแล้ว")
2. **สร้าง tuple ใหม่**: สร้าง tuple ใหม่ทั้งชุดที่ตำแหน่ง `ctid` ใหม่ พร้อม `xmin = current_XID` และ `xmax = 0` โดยมีข้อมูลคอลัมน์ที่อัปเดตแล้ว

มาดูให้เห็นจริงทีละขั้น:

```sql
-- ดูค่าปัจจุบันของแถว Somchai ก่อน UPDATE
SELECT xmin, xmax, ctid, * FROM accounts WHERE account_id = 1;
```

```
 xmin | xmax | ctid  | account_id | owner_name | balance
------+------+-------+------------+------------+----------
  751 |    0 | (0,4) |          1 | Somchai    | 10500.00
```

```sql
SELECT txid_current();
```

```
 txid_current
--------------
          760
```

```sql
UPDATE accounts SET balance = balance + 1000 WHERE account_id = 1;

SELECT xmin, xmax, ctid, * FROM accounts WHERE account_id = 1;
```

```
 xmin | xmax | ctid   | account_id | owner_name | balance
------+------+--------+------------+------------+----------
  760 |    0 | (0,10) |          1 | Somchai    | 11500.00
```

สังเกตว่า `SELECT` หลัง `UPDATE` แสดงแถว**ใหม่**ที่มี `xmin=760` (ตรงกับ transaction ที่ทำ UPDATE) และ `ctid` เปลี่ยนจาก `(0,4)` เป็น `(0,10)` — คือ **ตำแหน่งจริงบนดิสก์เปลี่ยนไป** เพราะเป็น tuple คนละชุดกันเลย

คำถามคือ: **แล้ว tuple เดิมที่ `ctid=(0,4)` หายไปไหน?**

คำตอบคือ **มันยังอยู่บนดิสก์** เพียงแต่ถูก mark ด้วย `xmax = 760` แล้ว ทำให้ query ปกติไม่แสดงมันอีกต่อไป (เพราะ visibility rule บอกว่า tuple ที่มี `xmax` committed แล้วและเกิดก่อน snapshot ปัจจุบัน ถือว่า "ตายแล้ว" ไม่ visible)

เราสามารถพิสูจน์การมีอยู่ของ tuple เก่าได้ด้วยส่วนขยาย `pageinspect`:

```sql
CREATE EXTENSION IF NOT EXISTS pageinspect;

SELECT lp, lp_off, lp_flags, t_xmin, t_xmax, t_ctid
FROM heap_page_items(get_raw_page('accounts', 0))
ORDER BY lp;
```

```
 lp | lp_off | lp_flags | t_xmin | t_xmax | t_ctid
----+--------+----------+--------+--------+--------
  1 |   8152 |        1 |    742 |    751 | (0,4)
  2 |   8096 |        1 |    742 |      0 | (0,2)
  3 |   8040 |        1 |    742 |      0 | (0,3)
  4 |   7984 |        1 |    751 |    760 | (0,10)
  5 |   7928 |        1 |    756 |      0 | (0,5)
  ...
 10 |   7560 |        1 |    760 |      0 | (0,10)
```

จะเห็น **tuple แถวที่ 1 (`t_ctid` เดิมของ account_id=1 ชุดแรกสุด)** ยังคงอยู่จริง มี `t_xmax = 751` (ถูก UPDATE ครั้งแรกมาร์กไว้) และแถวที่ 4 (ชุดที่สอง `ctid=(0,4)`) ก็ยังอยู่ มี `t_xmax = 760` (ถูก UPDATE ครั้งที่สองมาร์กไว้) — ส่วนแถวสุดท้าย (`ctid=(0,10)`) คือเวอร์ชันล่าสุดที่ `t_xmax = 0` (ยังไม่ตาย)

นี่คือหลักฐานที่จับต้องได้ที่สุดว่า **UPDATE ในความเป็นจริงคือ DELETE เชิงตรรกะ (mark xmax) + INSERT ของแถวใหม่** และก่อนที่ VACUUM จะมาทำงาน **tuple ทั้งสามเวอร์ชันของ account_id=1 (742→751, 751→760, 760→alive) ยังคงกินพื้นที่จริงบนดิสก์พร้อมกัน**

### ผลต่อ n_dead_tup

```sql
SELECT relname, n_live_tup, n_dead_tup
FROM pg_stat_user_tables
WHERE relname = 'accounts';
```

```
 relname  | n_live_tup | n_dead_tup
----------+------------+------------
 accounts |          7 |          2
```

`n_dead_tup = 2` สอดคล้องกับ tuple เก่าสองเวอร์ชันของ account_id=1 ที่ถูก mark `xmax` ไปแล้ว (จากการ UPDATE ถึงสองครั้งในตัวอย่างของเรา) นี่คือความเชื่อมโยงตรงไปยัง VACUUM (Part 060) — เพราะ dead tuple เหล่านี้คือสิ่งที่ VACUUM ต้องมาเก็บกวาดทิ้ง เพื่อคืนพื้นที่และป้องกัน table bloat

### HOT Update — เกริ่นสั้นๆ (รายละเอียดเต็มใน Step 839)

ในตัวอย่างข้างต้น `UPDATE` ทำให้ `ctid` เปลี่ยนไปเป็นบล็อกใหม่ในบางกรณี แต่ถ้าตารางมี **free space เพียงพอในบล็อกเดิม** และ **คอลัมน์ที่ update ไม่ใช่คอลัมน์ที่มี index** PostgreSQL จะพยายามวาง tuple ใหม่ **ในบล็อกเดียวกัน** และใช้กลไกพิเศษที่เรียกว่า **HOT (Heap-Only Tuple) update** เพื่อ**ไม่ต้อง**แก้ไข index เลย เราจะพิสูจน์เรื่องนี้อย่างละเอียดใน Step 839

---

## Step 838: DELETE ทำอะไรจริงๆ — แค่ mark xmax ไม่ได้ลบข้อมูลจริงทันที

หลักการเดียวกับ UPDATE แต่ไม่มีการสร้าง tuple ใหม่: `DELETE` เพียงแค่ **เขียนค่า `xmax = current_XID` ลงใน tuple ที่มีอยู่แล้ว** ข้อมูลจริงยังคงอยู่บนดิสก์ทั้งหมด ไม่มีการลบ bytes ใดๆ ออกจากไฟล์ในตอนนั้น

```sql
SELECT xmin, xmax, ctid, * FROM accounts WHERE owner_name = 'Kamon';
```

```
 xmin | xmax | ctid  | account_id | owner_name | balance
------+------+-------+------------+------------+---------
  759 |    0 | (0,8) |          7 | Kamon      | 3000.00
```

```sql
SELECT txid_current();
```

```
 txid_current
--------------
          761
```

```sql
DELETE FROM accounts WHERE owner_name = 'Kamon';

-- ตอนนี้ SELECT ปกติจะไม่เจอแถวนี้แล้ว เพราะ visibility rule กรองออก
SELECT xmin, xmax, ctid, * FROM accounts WHERE owner_name = 'Kamon';
```

```
(0 rows)
```

แต่ถ้าเราส่องด้วย `pageinspect` (ที่เห็นค่าดิบไม่ผ่านการกรอง visibility) จะพบว่า tuple **ยังคงอยู่จริง**:

```sql
SELECT lp, t_xmin, t_xmax, t_ctid,
       (t_infomask & 1024) != 0 AS xmax_committed,
       (t_infomask2 & 16384) != 0 AS heap_only_tuple
FROM heap_page_items(get_raw_page('accounts', 0))
WHERE t_xmin = 759;
```

```
 lp | t_xmin | t_xmax | t_ctid | xmax_committed | heap_only_tuple
----+--------+--------+--------+----------------+-----------------
  8 |    759 |    761 | (0,8)  | t              | f
```

จะเห็นว่า tuple ของ `Kamon` (ที่ `xmin=759`) ยังคงอยู่จริงบนดิสก์ ข้อมูลคอลัมน์ทั้งหมด (`owner_name='Kamon'`, `balance=3000.00`) ยังอยู่ครบ เพียงแต่ `t_xmax = 761` ทำให้ **query ปกติมองไม่เห็นมันอีก** เพราะ transaction 761 committed ไปแล้ว และ visibility rule ตัดสินว่า tuple นี้ "ตายแล้วสำหรับผู้สังเกตในปัจจุบัน"

ตรวจสอบ `n_dead_tup` อีกครั้ง:

```sql
SELECT relname, n_live_tup, n_dead_tup
FROM pg_stat_user_tables
WHERE relname = 'accounts';
```

```
 relname  | n_live_tup | n_dead_tup
----------+------------+------------
 accounts |          6 |          3
```

`n_dead_tup` เพิ่มขึ้นเป็น 3 (จาก 2 หลัง UPDATE บวก 1 จาก DELETE นี้)

### พื้นที่จะถูกคืนเมื่อไร?

พื้นที่ของ dead tuple เหล่านี้จะไม่ถูกคืนจนกว่า **VACUUM** จะทำงาน (autovacuum ทำงานอัตโนมัติตามเงื่อนไข หรือรัน `VACUUM` ด้วยตนเอง):

```sql
VACUUM accounts;

SELECT relname, n_live_tup, n_dead_tup
FROM pg_stat_user_tables
WHERE relname = 'accounts';
```

```
 relname  | n_live_tup | n_dead_tup
----------+------------+------------
 accounts |          6 |          0
```

หลัง VACUUM, `n_dead_tup` กลับเป็น 0 แต่**พื้นที่ (space) ที่ VACUUM ธรรมดาคืนนั้นยังคงอยู่ภายในไฟล์ heap** — ไฟล์ไม่ได้เล็กลงทันที (พื้นที่ถูก mark ว่า "reusable" สำหรับ tuple ใหม่ในอนาคต) ถ้าต้องการคืนพื้นที่กลับสู่ระบบปฏิบัติการจริงๆ ต้องใช้ `VACUUM FULL` (ซึ่งจะ lock ตารางแบบ exclusive — รายละเอียดเต็มอยู่ใน Part 060)

> **เชื่อมโยง Part 060**: บทนั้นสอน "ทำไมต้อง VACUUM" และ "จูน autovacuum อย่างไร" ในเชิงปฏิบัติการ บทนี้แสดงให้เห็น **สาเหตุระดับ tuple** ว่าทำไม VACUUM ถึงจำเป็น — เพราะทุก UPDATE/DELETE ทิ้ง dead tuple ไว้จริงบนดิสก์ ไม่ใช่แค่แนวคิดนามธรรม

---

## Step 839: Heap-Only Tuple (HOT) Update — optimization พิเศษ

ปัญหาหนึ่งของ MVCC คือ: ทุกครั้งที่ `UPDATE` สร้าง tuple ใหม่ (`ctid` ใหม่) โดยปกติแล้ว **ทุก index บนตารางนั้นต้องถูกอัปเดตด้วย** เพื่อให้ index entry ชี้ไปที่ `ctid` ใหม่ — ถ้าตารางมีหลาย index การ UPDATE หนึ่งครั้งอาจทำให้ต้องแก้ไข index หลายตัวพร้อมกัน ซึ่งมีต้นทุนสูง

PostgreSQL มีกลไกที่เรียกว่า **HOT (Heap-Only Tuple) Update** เพื่อลดภาระนี้ในกรณีที่เข้าเงื่อนไขสองข้อ:

1. **คอลัมน์ที่ถูก UPDATE ไม่ใช่คอลัมน์ที่มี index ใดๆ อ้างอิงอยู่** (ไม่กระทบ indexed column เลย)
2. **มีพื้นที่ว่างเพียงพอในบล็อก (page) เดียวกัน** เพื่อวาง tuple ใหม่ในบล็อกเดิม

เมื่อเข้าเงื่อนไขทั้งสองข้อ PostgreSQL จะ:

- สร้าง tuple ใหม่ **ในบล็อกเดียวกัน** กับ tuple เดิม
- **ไม่แก้ไข index เลยแม้แต่ตัวเดียว**
- ใช้กลไก "redirect" ภายในบล็อกเดิม (line pointer chain) เพื่อให้ index เดิมที่ชี้ไปยัง tuple เก่า สามารถตามไปยัง tuple ใหม่ได้ผ่านการไล่ chain ภายในบล็อก

มาพิสูจน์กัน โดยสร้างตารางใหม่ที่มี index เฉพาะบางคอลัมน์:

```sql
CREATE INDEX idx_accounts_owner ON accounts (owner_name);
```

ตอนนี้ `owner_name` มี index แต่ `balance` **ไม่มี** index

```sql
SELECT xmin, xmax, ctid, * FROM accounts WHERE account_id = 2;
```

```
 xmin | xmax | ctid  | account_id | owner_name | balance
------+------+-------+------------+------------+---------
  742 |    0 | (0,2) |          2 | Malee      |  5000.00
```

```sql
-- UPDATE คอลัมน์ balance เท่านั้น (ไม่กระทบ owner_name ที่มี index)
UPDATE accounts SET balance = balance + 100 WHERE account_id = 2;

SELECT xmin, xmax, ctid, * FROM accounts WHERE account_id = 2;
```

```
 xmin | xmax | ctid   | account_id | owner_name | balance
------+------+--------+------------+------------+---------
  763 |    0 | (0,11) |          2 | Malee      |  5100.00
```

ค่า `ctid` เปลี่ยนเป็น `(0,11)` — ยังคงอยู่ใน **block 0 เดิม** (แค่ offset เปลี่ยน) แสดงว่า tuple ใหม่ถูกวางในบล็อกเดียวกัน ซึ่งเป็นสัญญาณว่ากลไก HOT อาจถูกใช้งาน

ตรวจสอบด้วย `pageinspect` ว่าเป็น Heap-Only Tuple จริงหรือไม่:

```sql
SELECT lp, t_xmin, t_xmax, t_ctid,
       (t_infomask2 & 16384) != 0 AS is_heap_only,
       (t_infomask2 & 8192)  != 0 AS is_hot_updated
FROM heap_page_items(get_raw_page('accounts', 0))
WHERE t_xmin IN (742, 763) OR t_xmax = 763;
```

```
 lp | t_xmin | t_xmax | t_ctid | is_heap_only | is_hot_updated
----+--------+--------+--------+--------------+-----------------
  2 |    742 |    763 | (0,11) | f            | t
 11 |    763 |      0 | (0,11) | t            | f
```

ผลลัพธ์ยืนยันชัดเจน:

- Tuple เดิม (`lp=2`, `t_xmin=742`) ถูก mark `t_xmax=763` และมี flag `is_hot_updated = true` — หมายความว่า "การ update ที่ทำให้ tuple นี้ตาย เป็น HOT update"
- Tuple ใหม่ (`lp=11`, `t_xmin=763`) มี flag `is_heap_only = true` — หมายความว่า "tuple นี้ถูกสร้างจาก HOT update ไม่มี index entry ชี้มาที่มันโดยตรง (index ยังคงชี้ไปที่ tuple เดิมที่ `lp=2` แล้ว 'ไล่ chain' ภายในบล็อกไปหา tuple นี้)"

เปรียบเทียบกับกรณีที่ UPDATE คอลัมน์ที่ **มี index** (`owner_name`):

```sql
UPDATE accounts SET owner_name = 'Malee Somsri' WHERE account_id = 2;

SELECT lp, t_xmin, t_xmax, t_ctid,
       (t_infomask2 & 16384) != 0 AS is_heap_only,
       (t_infomask2 & 8192)  != 0 AS is_hot_updated
FROM heap_page_items(get_raw_page('accounts', 0))
WHERE t_xmax = txid_current()::text::xid OR t_xmin::text::bigint > 763
ORDER BY lp;
```

ในกรณีนี้ tuple ใหม่ที่ถูกสร้างจะมี `is_heap_only = false` เพราะ index บน `owner_name` **ต้อง** ได้รับ index entry ใหม่ชี้ไปยัง tuple ใหม่โดยตรง (ไม่สามารถใช้กลไก redirect chain ได้ เนื่องจากค่าคอลัมน์ที่ index อ้างอิงเปลี่ยนไป) — จึงไม่ใช่ HOT update

### ทำไม HOT ถึงสำคัญ

ประโยชน์ของ HOT update:

1. **ลดจำนวนครั้งที่ต้องแก้ไข index** — สำหรับตารางที่มีหลาย index การ UPDATE บ่อยๆ บนคอลัมน์ที่ไม่มี index จะเร็วขึ้นมาก เพราะไม่ต้องเขียน index ใหม่ทุกครั้ง
2. **ลด index bloat** — index ไม่โตขึ้นจาก dead entry ที่เกิดจาก UPDATE ซ้ำๆ
3. **HOT pruning** — PostgreSQL สามารถทำความสะอาด (prune) dead heap-only tuple ภายในบล็อกเดียวกันได้ทันทีระหว่างการอ่านปกติ (ไม่ต้องรอ VACUUM เต็มรูปแบบ) เพราะไม่มี index ใดอ้างอิง tuple เหล่านี้โดยตรง ทำให้จัดการได้ในระดับ page เพียงอย่างเดียว

**เงื่อนไขที่ทำให้ HOT ไม่เกิดขึ้น** (ต้องจำให้แม่น):

- UPDATE คอลัมน์ที่มี index อ้างอิงอยู่ (ไม่ว่า index เดียวหรือหลาย index)
- บล็อก (page) เดิมไม่มีพื้นที่ว่างเพียงพอ (`fillfactor` ต่ำเกินไปหรือ page เต็ม) — tuple ใหม่จึงต้องย้ายไปบล็อกอื่น
- ตารางมี `fillfactor = 100` (ค่า default) ซึ่งไม่เผื่อพื้นที่ว่างไว้เลย ทำให้ HOT เกิดขึ้นได้ยากขึ้นเมื่อข้อมูลแน่น — การตั้ง `fillfactor` ต่ำกว่า 100 (เช่น 90) จะเผื่อพื้นที่ว่างในแต่ละ page ไว้สำหรับ HOT update โดยเฉพาะ

ทดลองตั้งค่า `fillfactor` เพื่อเพิ่มโอกาส HOT update:

```sql
ALTER TABLE accounts SET (fillfactor = 90);
VACUUM FULL accounts;  -- จัดระเบียบใหม่พร้อมเผื่อพื้นที่ว่างตาม fillfactor
```

> ข้อควรระวัง: `VACUUM FULL` ล็อกตารางแบบ exclusive ระหว่างทำงาน ห้ามใช้กับตารางขนาดใหญ่ใน production โดยไม่วางแผน (รายละเอียดเต็มอยู่ใน Part 060)

---

## Step 840: แบบฝึกหัดรวม — ทดลองดู xmin/xmax/ctid เปลี่ยนแปลงจริง

มาสรุปทุกอย่างด้วยการทดลองแบบครบวงจร ตั้งแต่ INSERT → UPDATE → DELETE → VACUUM แล้วสังเกตค่าทุกขั้นตอน

```sql
-- ล้างตารางใหม่เพื่อทดลองแบบสะอาด
DROP TABLE IF EXISTS lab_accounts;

CREATE TABLE lab_accounts (
    account_id  SERIAL PRIMARY KEY,
    owner_name  VARCHAR(100),
    balance     NUMERIC(12,2)
);
```

### ขั้นที่ 1: INSERT — สังเกต xmin ใหม่

```sql
SELECT txid_current();  -- จด XID ไว้ก่อน
INSERT INTO lab_accounts (owner_name, balance) VALUES ('Test User', 1000.00);
SELECT xmin, xmax, ctid, * FROM lab_accounts;
```

```
 xmin | xmax | ctid  | account_id | owner_name | balance
------+------+-------+------------+------------+---------
  770 |    0 | (0,1) |          1 | Test User  | 1000.00
```

**สังเกต**: `xmin = 770` ตรงกับ XID ของ transaction ที่ INSERT พอดี, `xmax = 0` เพราะยังไม่มีใครลบ/แทนที่

### ขั้นที่ 2: UPDATE ครั้งที่ 1 — สังเกต xmax ของแถวเก่า + xmin ของแถวใหม่

```sql
UPDATE lab_accounts SET balance = balance + 500 WHERE account_id = 1;
SELECT xmin, xmax, ctid, * FROM lab_accounts;
```

```
 xmin | xmax | ctid  | account_id | owner_name | balance
------+------+-------+------------+------------+---------
  771 |    0 | (0,2) |          1 | Test User  | 1500.00
```

**สังเกต**: `SELECT` แสดงแถวใหม่เท่านั้น (`ctid=(0,2)`, `xmin=771`) แถวเก่า (`ctid=(0,1)`) ยังอยู่บนดิสก์แต่ถูกกรองออกเพราะ `xmax` ของมันถูก set เป็น 771 แล้ว

### ขั้นที่ 3: UPDATE ครั้งที่ 2 — สังเกตแนวโน้มต่อเนื่อง

```sql
UPDATE lab_accounts SET balance = balance - 200 WHERE account_id = 1;
SELECT xmin, xmax, ctid, * FROM lab_accounts;
```

```
 xmin | xmax | ctid  | account_id | owner_name | balance
------+------+-------+------------+------------+---------
  772 |    0 | (0,3) |          1 | Test User  | 1300.00
```

### ขั้นที่ 4: ตรวจสอบ dead tuple ทั้งหมดที่สะสมอยู่

```sql
SELECT relname, n_live_tup, n_dead_tup
FROM pg_stat_user_tables
WHERE relname = 'lab_accounts';
```

```
 relname      | n_live_tup | n_dead_tup
--------------+------------+------------
 lab_accounts |          1 |          2
```

**สังเกต**: มีแถว "live" เพียง 1 แถวในมุมมองของผู้ใช้ แต่จริงๆ มี dead tuple สะสมอยู่ 2 แถว (เวอร์ชันแรกที่ INSERT และเวอร์ชันที่สองจาก UPDATE ครั้งแรก) — รวมเป็น 3 เวอร์ชันจริงบนดิสก์ ณ ขณะนี้

### ขั้นที่ 5: DELETE — สังเกตว่าแถวยังอยู่จริงแต่มองไม่เห็น

```sql
SELECT txid_current();
DELETE FROM lab_accounts WHERE account_id = 1;
SELECT * FROM lab_accounts;  -- ว่างเปล่าในมุมมองปกติ
```

```
(0 rows)
```

```sql
-- แต่ pageinspect เห็นทุกเวอร์ชันจริง
SELECT lp, t_xmin, t_xmax, t_ctid
FROM heap_page_items(get_raw_page('lab_accounts', 0))
ORDER BY lp;
```

```
 lp | t_xmin | t_xmax | t_ctid
----+--------+--------+--------
  1 |    770 |    771 | (0,2)
  2 |    771 |    772 | (0,3)
  3 |    772 |    773 | (0,3)
```

**สังเกต**: มี tuple สามชุดในบล็อกเดียวกัน โดยชุดสุดท้าย (`lp=3`) มี `t_xmax=773` (จาก DELETE) แสดงว่า "ตายแล้ว" ทั้งหมด — ไม่มีชุดไหนเลยที่ `t_xmax=0` อีกต่อไป เพราะเวอร์ชันล่าสุดถูก DELETE ไปแล้ว

### ขั้นที่ 6: VACUUM — สังเกตการเก็บกวาด

```sql
VACUUM lab_accounts;
SELECT relname, n_live_tup, n_dead_tup
FROM pg_stat_user_tables
WHERE relname = 'lab_accounts';
```

```
 relname      | n_live_tup | n_dead_tup
--------------+------------+------------
 lab_accounts |          0 |          0
```

หลัง VACUUM ทุกอย่างถูกนับใหม่และ dead tuple ถูกทำเครื่องหมายว่า "reusable" (พร้อมให้ tuple ใหม่ในอนาคตมาใช้พื้นที่นี้แทน)

### สรุปสิ่งที่สังเกตได้จากแบบฝึกหัดนี้

| การกระทำ | xmin เปลี่ยนไหม | xmax เปลี่ยนไหม | ctid เปลี่ยนไหม | tuple เดิมหายจริงไหม |
|---|---|---|---|---|
| INSERT | ตั้งค่าใหม่ = current XID | คงที่ = 0 | ตำแหน่งใหม่ | (ไม่มี tuple เดิม) |
| UPDATE | tuple ใหม่ได้ xmin ใหม่ | tuple เดิมได้ xmax ใหม่ | เปลี่ยน (เว้นแต่ HOT ในบล็อกเดิม) | ไม่ ยังอยู่จนกว่า VACUUM |
| DELETE | ไม่เปลี่ยน | ตั้งค่า = current XID | ไม่เปลี่ยน | ไม่ ยังอยู่จนกว่า VACUUM |
| VACUUM | ไม่เปลี่ยน (ยกเว้น freeze) | ไม่เปลี่ยน | ไม่เปลี่ยน space ถูก reclaim ให้ reuse ได้ |

---

## สรุปท้ายบท

บทนี้พาเจาะลึก MVCC ไปถึงระดับที่ลึกที่สุดของหลักสูตร — ระดับ **tuple จริงบนดิสก์** ประเด็นสำคัญที่ต้องจำ:

1. **MVCC ไม่ใช่แค่แนวคิด** — มันคือกลไกจริงที่ทำงานผ่าน system columns `xmin`, `xmax`, `ctid` ที่แนบมากับทุก tuple โดยอัตโนมัติ
2. **`xmin`/`xmax` คือ Transaction ID (XID)** ซึ่งเป็นตัวเลข 32-bit ที่เพิ่มขึ้นเรื่อยๆ และมีปัญหา wraparound ที่ต้องจัดการด้วย VACUUM Freezing (เชื่อมโยง Part 060)
3. **Tuple Visibility** ถูกตัดสินโดยเปรียบเทียบ `xmin`/`xmax` ของ tuple กับ **snapshot** ของ transaction ปัจจุบัน — snapshot ประกอบด้วย `xmin`, `xmax`, `xip_list` ของตัวมันเอง
4. **Isolation Level กำหนดว่า snapshot ถูกสร้างเมื่อไร**: Read Committed สร้างทุก statement, Repeatable Read/Serializable สร้างครั้งเดียวตอนเริ่ม transaction
5. **INSERT** สร้าง tuple ใหม่พร้อม `xmin = current XID`
6. **UPDATE ไม่ได้แก้ไข tuple เดิม** — มัน mark `xmax` บน tuple เดิม แล้วสร้าง tuple ใหม่ทั้งชุดพร้อม `xmin` ใหม่
7. **DELETE ไม่ได้ลบข้อมูลทันที** — มัน mark `xmax` บน tuple ที่มีอยู่ ข้อมูลจริงยังอยู่บนดิสก์จนกว่า VACUUM จะมาเก็บกวาด
8. **HOT (Heap-Only Tuple) Update** เป็น optimization ที่ช่วยให้ UPDATE ที่ไม่กระทบ indexed column และมีพื้นที่ว่างในบล็อกเดิมเพียงพอ **ไม่ต้องแก้ไข index เลย** ลดภาระอย่างมากสำหรับตารางที่มีหลาย index
9. เครื่องมือสำคัญที่ใช้สำรวจ internals เหล่านี้ได้แก่ system column queries (`SELECT xmin, xmax, ctid, ...`), `pg_stat_user_tables` (`n_live_tup`, `n_dead_tup`), และ extension `pageinspect` (`heap_page_items`, `get_raw_page`)

ความเข้าใจระดับนี้คือสิ่งที่แยกวิศวกรฐานข้อมูลระดับ "ใช้งานเป็น" ออกจาก "เข้าใจว่าทำไมมันทำงานแบบนั้น" — เมื่อคุณเห็น table bloat, index bloat, หรือปัญหา wraparound ในอนาคต คุณจะเข้าใจ **สาเหตุที่แท้จริงระดับ tuple** ไม่ใช่แค่จำสูตรการแก้ปัญหา

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1

จากตาราง `accounts` ที่เตรียมไว้ตอนต้นบท จงเขียนคำสั่ง SQL เพื่อดูค่า `xmin`, `xmax`, `ctid` ของทุกแถว เรียงตาม `account_id`

<details>
<summary>เฉลย</summary>

```sql
SELECT xmin, xmax, ctid, account_id, owner_name, balance
FROM accounts
ORDER BY account_id;
```

`xmin` และ `xmax` เป็น system column ชนิด `xid` และ `ctid` เป็นชนิด `tid` — ทั้งสามคอลัมน์นี้มีอยู่ในทุกตารางแม้ไม่ได้ประกาศไว้ใน `CREATE TABLE`

</details>

---

### แบบฝึกหัดที่ 2

หลังจากรัน `INSERT INTO accounts (owner_name, balance) VALUES ('Test', 100)` ค่า `xmax` ของแถวใหม่นี้ควรเป็นเท่าไร และเพราะเหตุใด

<details>
<summary>เฉลย</summary>

`xmax = 0` เสมอสำหรับแถวที่เพิ่ง INSERT ใหม่ เพราะยังไม่มี transaction ใดมาลบหรือแทนที่ tuple นี้ — ค่า `0` ในบริบทของ `xmax` หมายถึง "ไม่มี transaction ที่ทำให้ tuple นี้กลายเป็นเวอร์ชันเก่า" (invalid transaction id) ซึ่งต่างจาก `xmin` ที่จะเป็น 0 ไม่ได้เพราะทุก tuple ต้องมีผู้สร้าง

```sql
INSERT INTO accounts (owner_name, balance) VALUES ('Test', 100);
SELECT xmin, xmax FROM accounts WHERE owner_name = 'Test';
-- xmax จะเป็น 0
```

</details>

---

### แบบฝึกหัดที่ 3

Transaction A เปิดด้วย `BEGIN ISOLATION LEVEL REPEATABLE READ;` แล้ว `SELECT` ตาราง `accounts` หนึ่งครั้ง จากนั้น Transaction B (คนละ session) `UPDATE` แถวหนึ่งใน `accounts` แล้ว commit สำเร็จ ถ้า Transaction A `SELECT` ตารางเดิมอีกครั้ง (ยังไม่ commit) ผลลัพธ์จะเป็นอย่างไร และเพราะกฎ visibility ข้อใด

<details>
<summary>เฉลย</summary>

Transaction A จะยังเห็น**ค่าเก่า** (ก่อน UPDATE ของ B) เพราะ Repeatable Read สร้าง snapshot เพียงครั้งเดียวตอนเริ่ม transaction และใช้ snapshot นั้นตลอดทั้ง transaction

ตาม visibility rule: transaction ของ B ที่ทำ UPDATE (สมมติ XID = 900) ไม่อยู่ในช่วงที่ snapshot ของ A มองว่า "committed แล้ว" เพราะ snapshot ของ A ถูกสร้างก่อนที่ B จะเริ่มทำงานด้วยซ้ำ — ดังนั้น `xmax=900` ที่ถูก mark บน tuple เดิมจึงถือว่า "ยังไม่ commit" ในมุมมองของ A ทำให้ tuple เดิมยังคง **visible** สำหรับ A

ถ้า A ใช้ Read Committed แทน ผลลัพธ์ที่สองจะเห็นค่าใหม่ทันที เพราะ Read Committed สร้าง snapshot ใหม่ทุก statement

</details>

---

### แบบฝึกหัดที่ 4

เพราะเหตุใด PostgreSQL จึงต้องมีกลไก "Freezing" ผ่าน VACUUM และมันแก้ปัญหาอะไร

<details>
<summary>เฉลย</summary>

XID เป็นเลข 32-bit จึงมีขอบเขตจำกัด (ประมาณ 4.2 พันล้านค่า) เมื่อระบบสร้าง transaction ไปเรื่อยๆ จนใกล้ถึงขีดจำกัด ตัวเลขจะ **วนกลับมาเริ่มใหม่ (wraparound)** ซึ่งจะทำให้ transaction เก่าที่เคยมี XID น้อยกว่าปัจจุบัน (และควร committed แล้ว) กลับถูกตีความผิดว่าเป็น "อนาคต" (XID ที่ยังไม่เกิด) ทำให้ข้อมูลที่ควร visible กลายเป็น invisible ทันที — เป็นหายนะข้อมูลระดับร้ายแรง

Freezing แก้ปัญหานี้โดยการเขียน `xmin` ของ tuple เก่าให้เป็นค่าพิเศษ `FrozenTransactionId` ซึ่งถือว่า "เก่าที่สุดเสมอ" ไม่ต้องเทียบกับตัวนับ XID ปกติอีกต่อไป ทำให้ tuple เหล่านั้นปลอดภัยจาก wraparound ตลอดไป

</details>

---

### แบบฝึกหัดที่ 5

จงอธิบายว่าทำไมคำกล่าวที่ว่า "`UPDATE` แก้ไขข้อมูลในตำแหน่งเดิม (in-place)" เป็นความเข้าใจที่ผิดสำหรับ PostgreSQL พร้อมยกตัวอย่างค่า `ctid` ที่เปลี่ยนแปลงประกอบ

<details>
<summary>เฉลย</summary>

ผิด เพราะ PostgreSQL ใช้ MVCC ที่ไม่แก้ไข tuple เดิม แต่สร้าง tuple ใหม่ทั้งชุด แล้ว mark tuple เดิมด้วย `xmax` ตัวอย่าง:

```sql
SELECT ctid FROM accounts WHERE account_id = 1;  -- (0,1)
UPDATE accounts SET balance = balance + 1 WHERE account_id = 1;
SELECT ctid FROM accounts WHERE account_id = 1;  -- (0,10) ตำแหน่งเปลี่ยนไปแล้ว
```

`ctid` คือตำแหน่งจริงบนดิสก์ เมื่อมันเปลี่ยนแปลงหลัง UPDATE แสดงว่า tuple ถูกสร้างขึ้นใหม่ทางกายภาพ ไม่ใช่การแก้ไข bytes เดิม ยกเว้นในกรณี HOT update ที่ tuple ใหม่อาจอยู่ใน block เดิมแต่ offset ต่างกัน — แต่ก็ยังคือ tuple คนละชุดกันเสมอ ไม่ใช่การแก้ไข in-place

</details>

---

### แบบฝึกหัดที่ 6

หลังจาก `DELETE FROM accounts WHERE account_id = 5;` แล้ว รัน `SELECT * FROM accounts WHERE account_id = 5;` จะได้ผลลัพธ์เป็น 0 แถว แต่ข้อมูลของแถวนั้นยังอยู่บนดิสก์จริงหรือไม่ จงอธิบายพร้อมวิธีตรวจสอบ

<details>
<summary>เฉลย</summary>

ยังอยู่จริง เพราะ `DELETE` เพียงแค่ mark `xmax = current_XID` บน tuple โดยไม่ได้ลบ bytes ใดๆ ออกจากไฟล์ heap ทันที ข้อมูลจะถูกเก็บกวาดจริงเมื่อ VACUUM ทำงานเท่านั้น

ตรวจสอบด้วย `pageinspect`:

```sql
CREATE EXTENSION IF NOT EXISTS pageinspect;
SELECT t_xmin, t_xmax, t_ctid
FROM heap_page_items(get_raw_page('accounts', 0));
```

จะพบ tuple ที่มี `t_xmax` เป็นค่า XID ของ transaction ที่ DELETE แทนที่จะเป็น 0 — แสดงว่าข้อมูลยังอยู่จริงแต่ถูกกรองออกจาก query ปกติ

</details>

---

### แบบฝึกหัดที่ 7

เงื่อนไขสองข้อที่ทำให้ `UPDATE` เข้าเงื่อนไข HOT (Heap-Only Tuple) update คืออะไรบ้าง และถ้าไม่เข้าเงื่อนไข จะเกิดอะไรขึ้นกับ index

<details>
<summary>เฉลย</summary>

เงื่อนไข HOT update:

1. คอลัมน์ที่ถูก UPDATE **ไม่ใช่** คอลัมน์ที่มี index ใดๆ อ้างอิงอยู่
2. บล็อก (page) เดิมมีพื้นที่ว่างเพียงพอสำหรับวาง tuple ใหม่ในบล็อกเดียวกัน

ถ้าไม่เข้าเงื่อนไข (เช่น UPDATE คอลัมน์ที่มี index หรือ page เต็ม) PostgreSQL ต้อง:
- สร้าง index entry ใหม่ชี้ไปยัง tuple ใหม่โดยตรงในทุก index ที่เกี่ยวข้อง
- ทิ้ง index entry เก่าไว้เป็น dead entry ที่ต้องรอ VACUUM มาเก็บกวาด (ทำให้ index bloat มากขึ้น)
- มีต้นทุน I/O และ CPU สูงกว่า HOT update อย่างมาก โดยเฉพาะเมื่อตารางมีหลาย index

</details>

---

### แบบฝึกหัดที่ 8

จงเขียนคำสั่งเพื่อตรวจสอบว่าตาราง `accounts` มี dead tuple สะสมอยู่กี่แถว และอธิบายว่าตัวเลขนี้มาจากการกระทำใดบ้าง

<details>
<summary>เฉลย</summary>

```sql
SELECT relname, n_live_tup, n_dead_tup
FROM pg_stat_user_tables
WHERE relname = 'accounts';
```

`n_dead_tup` คือจำนวน tuple ที่ถูก mark `xmax` แล้ว (จาก DELETE หรือจาก UPDATE ที่ทำให้ tuple เดิมกลายเป็นเวอร์ชันเก่า) แต่ยังไม่ถูก VACUUM เก็บกวาด — ตัวเลขนี้สะสมจากทุก UPDATE (นับ 1 dead tuple ต่อการ update 1 ครั้งที่สร้างเวอร์ชันเก่า) และทุก DELETE (นับ 1 dead tuple ต่อแถวที่ลบ)

</details>

---

### แบบฝึกหัดที่ 9

`txid_current()` และค่า `xmin` ที่เห็นใน `SELECT xmin FROM accounts` เป็นชนิดข้อมูลเดียวกันหรือไม่ ต่างกันอย่างไร

<details>
<summary>เฉลย</summary>

ไม่เหมือนกันเสียทีเดียว: `xmin` ของ tuple (system column) เป็นชนิด `xid` ซึ่งเป็น 32-bit unsigned integer ล้วนๆ ตามที่เก็บจริงในดิสก์ ในขณะที่ `txid_current()` คืนค่าชนิด `bigint` (64-bit) ที่รวม "epoch" เข้าไปด้วย เพื่อให้สามารถเปรียบเทียบ XID ข้ามรอบ wraparound ได้อย่างถูกต้องโดยไม่ทำให้ตัวเลขวนกลับมาซ้ำกัน

ในทางปฏิบัติ เมื่อ transaction XID ยังไม่ข้าม wraparound ค่าตัวเลขส่วนล่าง 32 บิตของ `txid_current()` จะตรงกับค่า `xmin`/`xmax` ที่ tuple header เก็บไว้จริง

</details>

---

### แบบฝึกหัดที่ 10

ทดลองเขียนสคริปต์ SQL แบบครบวงจร (INSERT → UPDATE 2 ครั้ง → DELETE → VACUUM) บนตารางใหม่ชื่อ `lab_test` แล้วอธิบายว่าค่า `n_live_tup`/`n_dead_tup` ควรเปลี่ยนแปลงอย่างไรในแต่ละขั้นตอน

<details>
<summary>เฉลย</summary>

```sql
CREATE TABLE lab_test (id SERIAL PRIMARY KEY, val TEXT);

-- ขั้นที่ 1: INSERT
INSERT INTO lab_test (val) VALUES ('a');
-- คาดว่า n_live_tup=1, n_dead_tup=0 (หลัง ANALYZE/autovacuum อัปเดตสถิติ)

-- ขั้นที่ 2: UPDATE ครั้งที่ 1
UPDATE lab_test SET val = 'b' WHERE id = 1;
-- tuple เก่า (val='a') กลายเป็น dead tuple 1 แถว, tuple ใหม่เป็น live

-- ขั้นที่ 3: UPDATE ครั้งที่ 2
UPDATE lab_test SET val = 'c' WHERE id = 1;
-- dead tuple สะสมเป็น 2 แถว, tuple ใหม่ (val='c') ยังเป็น live

-- ขั้นที่ 4: DELETE
DELETE FROM lab_test WHERE id = 1;
-- แถว live กลายเป็น 0, dead tuple สะสมเป็น 3 แถว

-- ขั้นที่ 5: VACUUM
VACUUM lab_test;
-- n_live_tup=0, n_dead_tup=0 (พื้นที่ถูกทำเครื่องหมายว่า reusable)

SELECT relname, n_live_tup, n_dead_tup
FROM pg_stat_user_tables WHERE relname = 'lab_test';
```

หมายเหตุ: `n_live_tup`/`n_dead_tup` เป็นค่าประมาณที่อัปเดตโดย statistics collector ไม่ใช่ค่านับแบบ real-time เป๊ะ 100% อาจต้องรอ autovacuum/ANALYZE เพื่อให้ตัวเลขอัปเดตแน่นอน หรือใช้ `VACUUM (ANALYZE)` เพื่อบังคับอัปเดตทันที

</details>

---

## บทถัดไป

เมื่อเข้าใจ MVCC internals ระดับ tuple แล้ว บทถัดไปจะพาไปสู่การขยายความสามารถของ PostgreSQL ในอีกมิติหนึ่ง — การเขียน extension ด้วยภาษา C เพื่อสร้างฟังก์ชันและชนิดข้อมูลที่ทำงานเร็วในระดับ native code

**บทถัดไป**: [`./part-085-c-extension.md`](./part-085-c-extension.md) — การพัฒนา PostgreSQL Extension ด้วยภาษา C
