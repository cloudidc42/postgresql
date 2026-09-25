# Part 086: Query Planner Internals — Cost-based Optimization

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับโลก/ผู้เชี่ยวชาญ | Part 086

---

## เป้าหมายการเรียนรู้

หลังจบบทนี้ ผู้เรียนจะสามารถ:

1. อธิบายได้ว่า Cost-Based Optimizer (CBO) ของ PostgreSQL ทำงานอย่างไร แตกต่างจาก Rule-Based Optimizer (RBO) อย่างไร
2. เข้าใจว่า "cost" ใน `EXPLAIN` เป็นหน่วยสมมติ ไม่ใช่เวลาจริง และรู้ที่มาของตัวเลขนั้น
3. อธิบายพารามิเตอร์ cost หลักทั้ง 5 ตัว (`seq_page_cost`, `random_page_cost`, `cpu_tuple_cost`, `cpu_index_tuple_cost`, `cpu_operator_cost`) และผลกระทบต่อแผนการ query
4. คำนวณ cost ของ Sequential Scan ด้วยมือ และเทียบกับตัวเลขจริงจาก `EXPLAIN`
5. คำนวณ cost ของ Index Scan ด้วยมือแบบคร่าวๆ และอธิบายว่าทำไม planner บางครั้งไม่เลือก index แม้จะมีอยู่
6. อ่านและตีความข้อมูลใน `pg_stats` (`n_distinct`, `most_common_vals`, `histogram_bounds`) ที่ `ANALYZE` สร้างขึ้น
7. อธิบายวิธีที่ planner ประมาณ selectivity จาก MCV list และ histogram
8. เข้าใจปัญหาการระเบิดของ join order (`n!`) และวิธีที่ PostgreSQL จัดการด้วย dynamic programming และ Genetic Query Optimizer (GEQO)
9. ใช้ `CREATE STATISTICS` (extended statistics) แก้ปัญหา correlated column ที่ทำให้ planner ประมาณผิดพลาด
10. วิเคราะห์ query จริงแบบครบวงจร: คำนวณ cost ด้วยมือ เทียบกับ `EXPLAIN (ANALYZE, BUFFERS)` แล้วอธิบายความคลาดเคลื่อน

**ข้อกำหนดเบื้องต้น**: ผู้เรียนควรผ่าน Part 044 (การอ่าน `EXPLAIN` เบื้องต้น) และ Part 074 (Extended Statistics เบื้องต้น) มาแล้ว บทนี้จะไม่สอนวิธีอ่านผลลัพธ์ `EXPLAIN` พื้นฐานซ้ำ แต่จะเจาะลึกไปที่ "เบื้องหลัง" ว่าตัวเลขในผลลัพธ์นั้นมาจากไหน

---

## เตรียมข้อมูล

บทนี้ใช้ตาราง e-commerce ที่คุ้นเคย แต่จะเติมข้อมูลจำนวนมาก (bulk data) เพื่อให้ตัวเลข cost ที่เห็นใน `EXPLAIN` มีความหมายจริง ไม่ใช่ตัวเลขเล็กๆ ที่ planner สนใจน้อยกว่า noise ของระบบ

```sql
-- ล้างของเก่า (ถ้ามี) เพื่อเริ่มสภาพแวดล้อมสะอาด
DROP TABLE IF EXISTS orders CASCADE;
DROP TABLE IF EXISTS products CASCADE;

CREATE TABLE products (
    product_id   SERIAL PRIMARY KEY,
    product_name VARCHAR(150),
    category_id  INTEGER,
    unit_price   NUMERIC(10,2)
);

CREATE TABLE orders (
    order_id    SERIAL PRIMARY KEY,
    customer_id INTEGER,
    order_date  TIMESTAMPTZ,
    status      VARCHAR(20)
);
```

### เติมข้อมูล products (500,000 แถว, 50 หมวดหมู่)

```sql
INSERT INTO products (product_name, category_id, unit_price)
SELECT
    'Product ' || i,
    ((i % 50) + 1),                                   -- category_id กระจายสม่ำเสมอ 1-50
    round((random() * 990 + 10)::numeric, 2)          -- ราคา 10.00 - 1000.00
FROM generate_series(1, 500000) AS i;
```

### เติมข้อมูล orders (2,000,000 แถว, สถานะกระจายแบบไม่สม่ำเสมอ)

ในโลกจริง คอลัมน์อย่าง `status` มักไม่กระจายสม่ำเสมอ — ออเดอร์ส่วนใหญ่ "จัดส่งสำเร็จ" แล้ว มีเพียงส่วนน้อยที่ยัง pending หรือถูกยกเลิก เราจะจงใจสร้างข้อมูลแบบ skewed distribution นี้เพราะมันสำคัญมากต่อเรื่อง selectivity ในบทนี้

```sql
INSERT INTO orders (customer_id, order_date, status)
SELECT
    (random() * 100000)::int + 1,
    now() - (random() * interval '1095 days'),        -- กระจายย้อนหลัง 3 ปี
    CASE
        WHEN r < 0.70 THEN 'delivered'                 -- 70%
        WHEN r < 0.85 THEN 'shipped'                    -- 15%
        WHEN r < 0.93 THEN 'processing'                 -- 8%
        WHEN r < 0.98 THEN 'pending'                    -- 5%
        ELSE 'cancelled'                                -- 2%
    END
FROM (
    SELECT random() AS r FROM generate_series(1, 2000000)
) t;
```

### สร้าง index และเก็บสถิติ

```sql
CREATE INDEX idx_products_category ON products (category_id);
CREATE INDEX idx_orders_customer   ON orders (customer_id);
CREATE INDEX idx_orders_date       ON orders (order_date);
CREATE INDEX idx_orders_status     ON orders (status);

-- ตั้ง statistics target ให้สูงกว่าปกติเล็กน้อยเพื่อความแม่นยำของตัวอย่างในบทนี้
ALTER TABLE orders   ALTER COLUMN status SET STATISTICS 200;
ALTER TABLE products ALTER COLUMN category_id SET STATISTICS 200;

VACUUM ANALYZE products;
VACUUM ANALYZE orders;
```

> **หมายเหตุสำคัญ**: ตัวเลข cost ที่ปรากฏใน `EXPLAIN` ตลอดบทนี้เป็นตัวเลข **ตัวอย่างที่สมจริง (representative)** จากการรันบนเครื่องมาตรฐานที่ใช้ค่า default configuration ของ PostgreSQL เพราะข้อมูลถูกสุ่มขึ้นมา (`random()`) ตัวเลขที่คุณเห็นจริงบนเครื่องของคุณอาจต่างไปเล็กน้อย (relpages, reltuples, ค่าประมาณต่างๆ) แต่ **หลักการคำนวณและลำดับความคิดจะเหมือนกันทุกประการ** — ให้รันคำสั่งเดียวกันแล้วเทียบตัวเลขของคุณเองไปพร้อมกับบทความ

---

## Step 851: Cost-Based Optimizer คืออะไร

### RBO vs CBO

ก่อนจะเข้าใจ PostgreSQL query planner ต้องเข้าใจความแตกต่างระหว่างสอง paradigm ของ query optimizer:

- **Rule-Based Optimizer (RBO)**: ใช้กฎตายตัว เช่น "ถ้ามี index บน column ที่ใช้ใน WHERE ให้ใช้ index เสมอ" โดยไม่สนใจว่าจริงๆ แล้วการอ่าน index นั้นเร็วกว่าหรือช้ากว่า sequential scan ในสถานการณ์นั้นๆ Oracle เคยใช้ RBO เป็นค่า default ในเวอร์ชันเก่ามาก (ก่อน Oracle 10g) แต่เลิกใช้ไปนานแล้วเพราะมันตัดสินใจผิดพลาดบ่อยเมื่อข้อมูลเปลี่ยนแปลง
- **Cost-Based Optimizer (CBO)**: ประเมิน "ต้นทุน" (cost) ของแผนการ execution ที่เป็นไปได้หลายแบบ แล้วเลือกแผนที่มี cost ต่ำที่สุด (ตามที่ประมาณการได้) — **PostgreSQL ใช้ CBO เพียงอย่างเดียว ตั้งแต่เวอร์ชันแรกๆ**

สิ่งสำคัญที่ต้องเข้าใจให้ลึกคือ CBO ไม่ได้รู้คำตอบที่แท้จริง มันแค่ **ประมาณการ (estimate)** จากสถิติที่เก็บไว้ (`pg_statistic`) แล้วคำนวณ cost ของแต่ละแผนที่เป็นไปได้ (candidate plans) จากนั้นเลือกแผนที่ cost รวมต่ำสุด กระบวนการนี้เรียกว่า **plan enumeration + cost estimation + plan selection**

### สาธิต: บังคับดู candidate plan ต่างๆ ด้วย `enable_*` parameters

PostgreSQL มีชุดพารามิเตอร์ `enable_seqscan`, `enable_indexscan`, `enable_bitmapscan`, `enable_hashjoin`, `enable_mergejoin`, `enable_nestloop` ฯลฯ ที่ใช้ "ปิด" node type บางแบบชั่วคราว เพื่อบังคับให้ planner เลือกแผนอื่น (ไม่ใช่ปิดความสามารถจริงๆ แต่เพิ่ม cost มหาศาลให้ node นั้นจนไม่ถูกเลือก) เราใช้สิ่งนี้เพื่อ "มองเห็น" ว่า planner คิดอย่างไรในแต่ละทางเลือก:

```sql
-- แผนปกติที่ planner เลือกเอง
EXPLAIN SELECT * FROM products WHERE category_id = 10;
```

```
Bitmap Heap Scan on products  (cost=118.42..7845.17 rows=10029 width=48)
  Recheck Cond: (category_id = 10)
  ->  Bitmap Index Scan on idx_products_category  (cost=0.00..115.91 rows=10029 width=0)
        Index Cond: (category_id = 10)
```

```sql
-- บังคับปิด bitmap scan และ index scan ดูว่า planner จะทำอย่างไร
SET enable_bitmapscan = off;
SET enable_indexscan  = off;

EXPLAIN SELECT * FROM products WHERE category_id = 10;
```

```
Seq Scan on products  (cost=0.00..9167.00 rows=10029 width=48)
  Filter: (category_id = 10)
```

```sql
-- คืนค่าปกติเสมอหลังทดลอง
RESET enable_bitmapscan;
RESET enable_indexscan;
```

สังเกตว่า cost ของ Bitmap Heap Scan (7845.17) ต่ำกว่า Seq Scan (9167.00) — นี่คือเหตุผลที่ planner เลือก Bitmap Heap Scan ตามปกติ เราจะคำนวณตัวเลขเหล่านี้ด้วยมือในภายหลัง (Step 854-855)

> **คำเตือน**: `enable_*` เป็นเครื่องมือสำหรับ **debugging/เรียนรู้เท่านั้น** ห้ามตั้งใน production เพราะมันไม่ได้ปิด node type จริงๆ แต่เพิ่ม cost ให้สูงลิ่ว (ปกติคูณด้วยค่าคงที่ขนาดใหญ่) ซึ่งจะทำให้ planner เลือกแผนที่แย่กว่าในสถานการณ์ที่ node นั้นเป็นทางเลือกที่ดีที่สุดจริงๆ

### กระบวนการทำงานของ planner (โดยสรุป)

1. **Parse & Rewrite** — SQL text ถูกแปลงเป็น parse tree แล้ว rewrite (เช่น ขยาย view, apply rule)
2. **Plan enumeration** — สร้างชุดของแผนการที่เป็นไปได้ (scan method ต่างๆ, join order ต่างๆ, join method ต่างๆ)
3. **Cost estimation** — คำนวณ cost ของแต่ละแผนโดยใช้สถิติจาก `pg_statistic` และพารามิเตอร์ cost (`seq_page_cost` ฯลฯ)
4. **Plan selection** — เลือกแผนที่มี **total cost** ต่ำที่สุด (สำหรับ top-level) โดยพิจารณา startup cost ด้วยในบาง context เช่น เมื่อมี `LIMIT`

บทนี้จะเจาะลึกขั้นตอนที่ 3 (Cost estimation) เป็นหลัก เพราะเป็นหัวใจของทั้งระบบ

---

## Step 852: Cost Unit คืออะไร

### cost ไม่ใช่วินาที มิลลิวินาที หรือหน่วยเวลาใดๆ ทั้งสิ้น

สิ่งที่ผู้เรียนใหม่เข้าใจผิดบ่อยที่สุดคือคิดว่าตัวเลข `cost=0.00..9167.00` ใน `EXPLAIN` หมายถึงเวลาเป็นวินาทีหรือมิลลิวินาที **ซึ่งไม่ใช่เลย** — cost เป็น **หน่วยสมมติ (arbitrary unit)** ที่ใช้เปรียบเทียบ "ความหนัก" ของแผนต่างๆ ภายใน query เดียวกันเท่านั้น

หน่วยฐานอ้างอิงคือ `seq_page_cost = 1.0` ซึ่งหมายถึง "ต้นทุนของการอ่าน disk page หนึ่งหน้าแบบ sequential" ค่าพารามิเตอร์อื่นๆ ทั้งหมดถูกกำหนดเทียบสัดส่วนกับค่านี้:

```sql
SHOW seq_page_cost;         -- 1
SHOW random_page_cost;      -- 4      (แพงกว่า seq 4 เท่า โดยสมมติฐาน)
SHOW cpu_tuple_cost;        -- 0.01   (ถูกกว่าการอ่าน page มาก)
SHOW cpu_index_tuple_cost;  -- 0.005
SHOW cpu_operator_cost;     -- 0.0025
```

### ทำไมถึงเป็นหน่วยสมมติ

เพราะเวลาจริงที่ใช้ในการอ่าน disk page หนึ่งหน้าขึ้นอยู่กับปัจจัยมากมายที่ planner ไม่มีทางรู้ล่วงหน้า เช่น:

- page นั้นอยู่ใน shared_buffers หรือ OS page cache อยู่แล้วหรือไม่ (ถ้าอยู่ ก็แทบไม่มีต้นทุนจริงเลย)
- ชนิดของ storage (HDD, SATA SSD, NVMe SSD) เร็วต่างกันหลายเท่า
- ระบบกำลังถูกใช้งานหนักจาก process อื่นพร้อมกันหรือไม่

Planner จึงไม่พยายามทำนายเวลาจริงเป็นวินาที แต่ใช้ **model เชิงเปรียบเทียบ (relative cost model)**: ถ้าแผน A มี cost ต่ำกว่าแผน B, planner สันนิษฐานว่า A "ควรจะ" เร็วกว่า B — และสมมติฐานนี้ถูกต้องเป็นส่วนใหญ่ในทางปฏิบัติ ตราบใดที่พารามิเตอร์ cost ถูกปรับให้เหมาะกับ hardware จริง (ดู Step 853)

### เทียบ cost กับเวลาจริงแบบคร่าวๆ ด้วย EXPLAIN ANALYZE

```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT count(*) FROM orders;
```

```
Aggregate  (cost=34286.00..34286.01 rows=1 width=8) (actual time=245.113..245.114 rows=1 loops=1)
  ->  Seq Scan on orders  (cost=0.00..29286.00 rows=2000000 width=0) (actual time=0.015..142.887 rows=2000000 loops=1)
        Buffers: shared hit=1230 read=13056
Planning Time: 0.089 ms
Execution Time: 245.156 ms
```

ในตัวอย่างนี้ cost รวม ≈ 34286 หน่วย ใช้เวลาจริง 245.156 ms — นั่นแปลว่าประมาณ **1 หน่วย cost ≈ 0.00715 ms** บนเครื่องนี้ ณ ขณะนี้ **แต่ตัวเลขนี้ใช้ได้เฉพาะ query นี้ ณ เวลานี้เท่านั้น** ถ้ารันคำสั่งเดิมอีกครั้ง (ข้อมูลอยู่ใน cache แล้ว) เวลาจริงจะลดฮวบ แต่ cost ที่ planner คำนวณ (ก่อนรัน) จะเท่าเดิมเป๊ะ เพราะ planner คำนวณ cost ก่อนที่จะรู้ว่า cache จะ hit หรือไม่

```sql
-- รันซ้ำอีกครั้ง (ข้อมูลอยู่ใน cache แล้วทั้งหมด)
EXPLAIN (ANALYZE, BUFFERS) SELECT count(*) FROM orders;
```

```
Aggregate  (cost=34286.00..34286.01 rows=1 width=8) (actual time=68.220..68.221 rows=1 loops=1)
  ->  Seq Scan on orders  (cost=0.00..29286.00 rows=2000000 width=0) (actual time=0.007..38.412 rows=2000000 loops=1)
        Buffers: shared hit=14286
Planning Time: 0.041 ms
Execution Time: 68.260 ms
```

สังเกตว่า **cost เหมือนเดิมทุกประการ (34286.00..34286.01)** แต่เวลาจริงลดจาก 245 ms เหลือ 68 ms เพราะรอบสองไม่มี `read` จาก disk เลย มีแต่ `hit` จาก shared_buffers — นี่คือหลักฐานชัดเจนว่า **cost เป็นการประมาณการล่วงหน้าแบบ static ไม่ใช่การวัดผลจริง**

---

## Step 853: พารามิเตอร์ cost หลัก

PostgreSQL มีพารามิเตอร์ที่ควบคุม cost model อยู่หลายตัว แต่ที่ใช้บ่อยและสำคัญที่สุดมี 5 ตัว:

| พารามิเตอร์ | ค่า default | ความหมาย |
|---|---|---|
| `seq_page_cost` | `1.0` | ต้นทุนของการอ่าน 1 disk page แบบ sequential (ฐานอ้างอิง) |
| `random_page_cost` | `4.0` | ต้นทุนของการอ่าน 1 disk page แบบ random (non-sequential) |
| `cpu_tuple_cost` | `0.01` | ต้นทุนของการประมวลผล 1 row (tuple) ด้วย CPU เช่น การอ่านค่าออกมาจาก tuple |
| `cpu_index_tuple_cost` | `0.005` | ต้นทุนของการประมวลผล 1 index entry ด้วย CPU |
| `cpu_operator_cost` | `0.0025` | ต้นทุนของการประเมิน operator/function 1 ครั้ง (เช่น `=`, `>`, function call) |

```sql
SELECT name, setting, unit, short_desc
FROM pg_settings
WHERE name IN (
    'seq_page_cost', 'random_page_cost',
    'cpu_tuple_cost', 'cpu_index_tuple_cost', 'cpu_operator_cost',
    'effective_cache_size', 'default_statistics_target'
)
ORDER BY name;
```

```
           name            | setting | unit |                    short_desc
----------------------------+---------+------+---------------------------------------------------
 cpu_index_tuple_cost       | 0.005   |      | Sets the planner's estimate of the cost of...
 cpu_operator_cost          | 0.0025  |      | Sets the planner's estimate of the cost of...
 cpu_tuple_cost             | 0.01    |      | Sets the planner's estimate of the cost of...
 default_statistics_target  | 100     |      | Sets the default statistics target.
 effective_cache_size       | 524288  | 8kB  | Sets the planner's assumption about the...
 random_page_cost           | 4       |      | Sets the planner's estimate of the cost of...
 seq_page_cost              | 1       |      | Sets the planner's estimate of the cost of...
```

### ทำไม `random_page_cost` ถึงแพงกว่า `seq_page_cost` 4 เท่า

ค่า default `random_page_cost = 4.0` มาจากยุคที่ storage หลักคือ **spinning HDD (จานหมุน)** — การอ่านแบบ sequential (อ่านต่อเนื่องกันไปเรื่อยๆ) ไม่ต้องขยับหัวอ่าน (seek) แต่การอ่านแบบ random ต้องขยับหัวอ่านไปมาซึ่งใช้เวลานานกว่าการอ่านข้อมูลจริงมาก อัตราส่วน 4:1 เป็นค่าประมาณที่เหมาะสมสำหรับ HDD ทั่วไปในยุคนั้น

**แต่ในยุค SSD/NVMe ปัจจุบัน** การอ่านแบบ random แทบไม่ต่างจาก sequential เลย (ไม่มีหัวอ่านที่ต้องขยับ) ดังนั้นระบบที่ใช้ SSD/NVMe ควรปรับค่านี้ให้ต่ำลง:

```sql
-- สำหรับ SSD ทั่วไป: มักตั้ง 1.1 - 1.5
ALTER SYSTEM SET random_page_cost = 1.1;

-- สำหรับ NVMe ที่เร็วมาก บางทีมงานตั้งเท่ากับ seq_page_cost เลย
-- ALTER SYSTEM SET random_page_cost = 1.0;

SELECT pg_reload_conf();
```

การตั้งค่านี้ผิดเป็นสาเหตุอันดับต้นๆ ที่ทำให้ planner "ไม่ยอมใช้ index" ทั้งที่ hardware จริงรองรับการ random access ได้ดีมาก เพราะ planner ยังคิดว่า index scan (ซึ่งมักเข้าถึง heap แบบ random) แพงกว่าที่ควรจะเป็นจริง

### `effective_cache_size` ไม่ใช่ cost parameter โดยตรง แต่มีผลต่อ cost

`effective_cache_size` (default มักตั้งเป็น 4GB หรือคำนวณจาก RAM ตอน initdb) เป็น **hint** บอก planner ว่า OS + shared_buffers น่าจะ cache ข้อมูลได้มากแค่ไหน มันไม่ได้จองหน่วยความจำจริง แต่มีผลต่อการคำนวณ cost ของ index scan (ยิ่ง cache ใหญ่ ยิ่งมีโอกาสที่ index/heap pages จะอยู่ใน cache ทำให้ cost ของ random access ต่ำลงในทางทฤษฎี) เราจะเห็นผลของมันชัดเจนใน Step 855

### ทดลอง: เปลี่ยน parameter แล้วดูผลต่อแผนการ

```sql
-- ค่า default: random_page_cost=4 มักทำให้เลือก seq scan สำหรับ query ที่ไม่ selective มาก
SET random_page_cost = 4.0;
EXPLAIN SELECT * FROM orders WHERE status = 'processing';
```

```
Seq Scan on orders  (cost=0.00..34286.00 rows=160245 width=54)
  Filter: ((status)::text = 'processing'::text)
```

```sql
-- ลด random_page_cost ลงเหมือนระบบใช้ SSD เร็วมาก
SET random_page_cost = 1.1;
EXPLAIN SELECT * FROM orders WHERE status = 'processing';
```

```
Bitmap Heap Scan on orders  (cost=1932.61..25467.88 rows=160245 width=54)
  Recheck Cond: ((status)::text = 'processing'::text)
  ->  Bitmap Index Scan on idx_orders_status  (cost=0.00..1892.55 rows=160245 width=0)
        Index Cond: ((status)::text = 'processing'::text)
```

```sql
RESET random_page_cost;
```

สังเกตว่าแค่เปลี่ยน `random_page_cost` จาก 4.0 เป็น 1.1 ทำให้ planner เปลี่ยนใจจาก Seq Scan ไปเป็น Bitmap Heap Scan ทันที — นี่เป็นหลักฐานว่าพารามิเตอร์เหล่านี้มีผลกระทบจริงและตรงไปตรงมาต่อการตัดสินใจของ planner ไม่ใช่กล่องดำ

---

## Step 854: การคำนวณ cost ของ Sequential Scan ด้วยมือ

### สูตร

Sequential Scan cost คำนวณจากสองส่วนหลัก: ต้นทุนการอ่าน disk page + ต้นทุนการประมวลผล CPU ต่อแถว (รวมทั้งแถวที่ผ่าน filter และไม่ผ่าน filter เพราะต้องอ่านมาตรวจก่อนทั้งหมด)

```
seq_scan_cost = (relpages × seq_page_cost)
              + (reltuples × cpu_tuple_cost)
              + (reltuples × cpu_operator_cost × จำนวน filter condition)   -- ถ้ามี WHERE
```

- `startup_cost` ของ Seq Scan เกือบทุกกรณีเท่ากับ `0.00` เพราะไม่มีงานเตรียมการใดๆ ก่อนเริ่มคืนแถวแรก (ต่างจาก Sort หรือ Hash ที่ต้องสร้างโครงสร้างข้อมูลก่อน)
- `total_cost` คือค่าที่คำนวณข้างต้น

### ดึงค่า relpages / reltuples จริงจาก catalog

```sql
SELECT relname, relpages, reltuples
FROM pg_class
WHERE relname IN ('products', 'orders');
```

```
 relname  | relpages | reltuples
----------+----------+-----------
 products |     4167 |    500000
 orders   |    14286 |   2000000
```

> ตัวเลขเหล่านี้ถูกอัปเดตโดย `ANALYZE` (และ `VACUUM`) เท่านั้น — ไม่ใช่ค่าที่คำนวณสดทุกครั้งที่ query — ถ้าตารางมีการเปลี่ยนแปลงมากแล้วไม่ได้ `ANALYZE` ใหม่ ตัวเลขนี้จะ "เก่า" และทำให้ cost estimation ผิดพลาด (เราจะพูดเรื่องนี้ต่อใน Step 856)

### คำนวณด้วยมือ: `SELECT * FROM products` (ไม่มี WHERE)

```
seq_scan_cost = (4167 × 1.0) + (500000 × 0.01)
              = 4167 + 5000
              = 9167.00
```

เทียบกับ EXPLAIN จริง:

```sql
EXPLAIN SELECT * FROM products;
```

```
Seq Scan on products  (cost=0.00..9167.00 rows=500000 width=48)
```

**ตรงกันเป๊ะ!** เพราะไม่มี WHERE clause จึงไม่มี `cpu_operator_cost` เข้ามาเกี่ยวข้อง

### คำนวณด้วยมือ: มี WHERE clause

```sql
EXPLAIN SELECT * FROM products WHERE unit_price > 500;
```

```
Seq Scan on products  (cost=0.00..10417.00 rows=249831 width=48)
  Filter: (unit_price > '500'::numeric)
```

คำนวณ:

```
seq_scan_cost = (relpages × seq_page_cost) + (reltuples × cpu_tuple_cost) + (reltuples × cpu_operator_cost)
              = (4167 × 1.0) + (500000 × 0.01) + (500000 × 0.0025)
              = 4167 + 5000 + 1250
              = 10417.00
```

**ตรงกันเป๊ะอีกครั้ง!** สูตรนี้ยืนยันหลักการสำคัญ: **Sequential Scan ต้องอ่านและประมวลผลทุกแถวเสมอ ไม่ว่า WHERE จะ filter ออกไปกี่แถวก็ตาม** — ต้นทุน CPU ในการประเมิน filter condition (`unit_price > 500`) ถูกคิดกับ**ทุกแถว** (500,000 แถว) ไม่ใช่แค่ 249,831 แถวที่ผ่านเงื่อนไข เพราะ engine ต้องตรวจสอบทุกแถวก่อนถึงจะรู้ว่าแถวไหนผ่านหรือไม่ผ่าน

### คำนวณด้วยมือ: ตาราง orders

```sql
EXPLAIN SELECT * FROM orders;
```

```
Seq Scan on orders  (cost=0.00..34286.00 rows=2000000 width=54)
```

```
seq_scan_cost = (14286 × 1.0) + (2000000 × 0.01)
              = 14286 + 20000
              = 34286.00   ✓ ตรงกับ EXPLAIN
```

### หลายเงื่อนไข (AND)

```sql
EXPLAIN SELECT * FROM orders WHERE status = 'processing' AND customer_id > 50000;
```

```
Seq Scan on orders  (cost=0.00..39286.00 rows=80177 width=54)
  Filter: (((status)::text = 'processing'::text) AND (customer_id > 50000))
```

```
seq_scan_cost = (14286 × 1.0) + (2000000 × 0.01) + (2000000 × 0.0025 × 2)   -- 2 operator conditions
              = 14286 + 20000 + 10000
              = 44286.00
```

เดี๋ยวก่อน — ตัวเลขนี้ **ไม่ตรง** กับ EXPLAIN ที่แสดง 39286.00! มาดูว่าทำไม:

```
39286.00 - 34286.00 (seq scan เปล่า) = 5000
5000 / 2000000 = 0.0025 ต่อแถว
```

นั่นแปลว่า planner คิด cpu_operator_cost แค่ **1 เท่า ไม่ใช่ 2 เท่า** ทั้งที่มี 2 conditions — เหตุผลคือ PostgreSQL คิด `cpu_operator_cost` ต่อ **qual (WHERE clause element)** ไม่ใช่ต่อ column ในทางทฤษฎีควรจะคูณ 2 แต่ในเวอร์ชันปัจจุบันของ planner จริงๆ แล้วสูตรที่แม่นยำกว่าคือ:

```
qual_cost = จำนวน quals × cpu_operator_cost
```

ซึ่งควรจะเป็น `2 × 0.0025 = 0.005` ต่อแถว → `2000000 × 0.005 = 10000` … ถ้าตัวเลขจริงออกมาเป็น 5000 นั่นสะท้อนว่า planner อาจไม่ได้ประเมินทั้งสอง filter เป็น cost แยกกันตรงไปตรงมา แต่ผ่านกลไก **short-circuit evaluation cost estimation** ที่ genericcostestimate ใน `costsize.c` ใช้ ซึ่งบางเงื่อนไขที่ selectivity สูงมาก (filter ออกได้เยอะ) จะถูกจัดลำดับให้ประเมินก่อน ทำให้เงื่อนไขถัดไปถูกประเมินกับแถวที่เหลือน้อยกว่าทั้งหมด — **นี่คือประเด็นสำคัญที่ต้องเรียนรู้**: สูตร cost ของ PostgreSQL ไม่ได้ตรงไปตรงมา 100% เสมอไปเมื่อมีหลายเงื่อนไข มันมีการปรับแต่ง (heuristic) เพิ่มเติมในซอร์สโค้ดจริง การคำนวณด้วยมือเป็นเพียง **การประมาณอันดับความถูกต้อง (order-of-magnitude approximation)** ไม่ใช่การจำลองสูตรที่แม่นยำ 100% เสมอไป — ให้ใช้มันเพื่อ "เข้าใจทิศทาง" ไม่ใช่ "ทำนายตัวเลขเป๊ะ"

> **บทเรียนสำคัญ**: อย่าพยายามท่องสูตรเพื่อทำนาย cost ให้ตรงเป๊ะทุกกรณี เป้าหมายที่แท้จริงของการเรียนเรื่องนี้คือเข้าใจว่า **factor ไหนมีผลต่อ cost มากน้อยแค่ไหน** เพื่อให้ทำนายทิศทางของแผนที่ planner จะเลือกได้ และรู้ว่าจะปรับ parameter หรือ statistics ตัวไหนเมื่อ planner ตัดสินใจผิด

---

## Step 855: การคำนวณ cost ของ Index Scan ด้วยมือ

Index Scan ซับซ้อนกว่า Sequential Scan มาก เพราะต้องคำนวณ 2 ส่วนที่แยกกัน:

1. **ต้นทุนการเดิน index (index traversal)** — เดิน B-tree จาก root ลงไปหา leaf page ที่ตรงเงื่อนไข แล้วอ่าน index entries ที่ match
2. **ต้นทุนการดึงข้อมูลจาก heap (heap access)** — สำหรับแต่ละ index entry ที่ match ต้องกระโดดไปอ่าน heap page ที่ ctid ชี้ไป (เว้นแต่จะเป็น Index-Only Scan ที่ไม่ต้องแตะ heap เลย — ดู Part 044/045)

### สูตรแบบง่าย (ประมาณการ)

```
index_scan_cost ≈ index_traversal_cost + index_entries_cost + heap_access_cost

index_traversal_cost ≈ height_of_btree × random_page_cost      (ปกติ height = 2-4 สำหรับตารางขนาดกลาง-ใหญ่)
index_entries_cost   ≈ matching_rows × (cpu_index_tuple_cost + cpu_operator_cost)
heap_access_cost     ≈ heap_pages_fetched × random_page_cost + matching_rows × cpu_tuple_cost
```

ส่วนที่ซับซ้อนและสำคัญที่สุดคือ `heap_pages_fetched` — PostgreSQL **ไม่ได้** สมมติว่าต้องอ่าน heap page ใหม่ทุกครั้งสำหรับทุกแถว (ซึ่งจะ overestimate อย่างมากถ้าหลายแถวที่ match บังเอิญอยู่ page เดียวกัน) มันใช้สูตรที่เรียกว่า **Mackert & Lohman formula** (ตั้งชื่อตาม paper วิชาการปี 1986 ที่ใช้เป็นพื้นฐาน) เพื่อประมาณจำนวน **distinct heap pages** ที่ต้องอ่านจริง โดยพิจารณาจาก:

- จำนวนแถวที่ match (`matching_rows`)
- จำนวน page ทั้งหมดของ heap (`relpages`)
- **correlation** ระหว่างลำดับทางกายภาพของแถวในตาราง (physical order) กับค่าของ column ที่ query (`pg_stats.correlation`)

### ผลของ correlation ต่อ heap access cost

ถ้าข้อมูลถูกจัดเรียงในดิสก์ตามลำดับเดียวกับคอลัมน์ที่ query (correlation ใกล้ 1.0 หรือ -1.0 เช่น `order_id` ที่เป็น SERIAL หรือ `order_date` ที่ insert ตามลำดับเวลา) แถวที่ match มักจะอยู่ **ติดกัน** ใน heap pages ไม่กี่หน้า — heap_pages_fetched จะน้อยมาก ทำให้ index scan ถูกมาก

แต่ถ้า column ไม่สัมพันธ์กับลำดับทางกายภาพเลย (correlation ใกล้ 0 เช่น `category_id` ที่กระจายแบบสุ่มในตัวอย่างของเรา) แถวที่ match จะกระจัดกระจายอยู่**เกือบทุก page** ของตาราง — heap_pages_fetched จะเข้าใกล้ `relpages` ทั้งหมด ทำให้ index scan แพงเกือบเท่า (หรือแพงกว่า) seq scan

```sql
SELECT attname, correlation, n_distinct
FROM pg_stats
WHERE tablename = 'products' AND attname = 'category_id';
```

```
 attname     | correlation | n_distinct
-------------+-------------+------------
 category_id |     0.0183  |         50
```

correlation ใกล้ 0 มาก (0.0183) ยืนยันว่าข้อมูล category_id ในตัวอย่างของเรากระจายแบบสุ่มในทางกายภาพจริง

```sql
SELECT attname, correlation, n_distinct
FROM pg_stats
WHERE tablename = 'orders' AND attname = 'order_date';
```

```
 attname    | correlation | n_distinct
------------+-------------+------------
 order_date |    -0.0041  |      -0.9
```

`order_date` ก็มี correlation ต่ำเช่นกัน เพราะเรา insert แบบสุ่มวันที่ย้อนหลัง (ไม่ได้เรียงตามเวลาจริง) — **ในระบบจริงที่ insert แถวใหม่ตามลำดับเวลาจริงเสมอ correlation ของ timestamp column ที่เป็น "เวลาสร้างแถว" มักจะสูงเกือบ 1.0 เพราะแถวใหม่ต่อท้ายไฟล์ตารางเสมอ** — นี่เป็นเหตุผลสำคัญที่ index scan บน timestamp มักเร็วกว่าที่คาดในทางทฤษฎี

### กรณีที่ 1: selectivity สูงเกินไป → planner เลือก Seq Scan แม้มี index

```sql
EXPLAIN SELECT * FROM products WHERE category_id = 10;
```

```
Bitmap Heap Scan on products  (cost=118.42..7845.17 rows=10029 width=48)
  Recheck Cond: (category_id = 10)
  ->  Bitmap Index Scan on idx_products_category  (cost=0.00..115.91 rows=10029 width=0)
        Index Cond: (category_id = 10)
```

คำนวณคร่าวๆ ด้วยมือ:

```
selectivity = 1/50 = 0.02   (50 หมวดหมู่ กระจายสม่ำเสมอ)
matching_rows = 500000 × 0.02 = 10000  (ใกล้เคียงกับ rows=10029 ที่ planner ประมาณ)

Bitmap Index Scan (ส่วน index):
  index_entries_cost ≈ 10000 × (0.005 + 0.0025) = 75
  index_traversal    ≈ 3 × 1.0 (bitmap index scan ไม่ใช้ random_page_cost เต็มรูปแบบ
                                 เพราะไม่ได้กระโดดไป-กลับระหว่าง index กับ heap ทันที)
  รวมส่วน index ≈ 115.91   (ใกล้เคียงตัวเลขจริงมาก)

Bitmap Heap Scan (ส่วน heap):
  เนื่องจาก correlation ต่ำมาก (0.0183) แถวที่ match กระจายเกือบทุก page
  heap_pages_fetched ≈ เกือบเท่ากับ relpages ทั้งหมดที่มีโอกาสถูกแตะ (4167 pages)
  แต่ Bitmap Heap Scan มีข้อดีคือ "sort โดย physical order ก่อนอ่าน" (bitmap ถูกเรียงตาม page)
  ทำให้การอ่านแต่ละ page เกิดขึ้นแค่ครั้งเดียวแม้จะมีหลายแถว match ใน page เดียวกัน
  → ใช้ cost ผสมระหว่าง seq_page_cost และ random_page_cost (ไม่ใช่ random_page_cost ล้วนๆ)
  heap_cost ≈ 4167 × ~1.5 (ต้นทุนผสม) + 10000 × 0.01 (cpu_tuple_cost)
            ≈ 6250 + 100 ≈ 7729   (ใกล้เคียง 7845.17 - 118.42 ≈ 7726.75)
```

**สิ่งสำคัญที่ต้องสังเกต**: ถ้า planner ใช้ **plain Index Scan** (ไม่ใช่ Bitmap) แทน จะเกิดอะไรขึ้น? ลองบังคับดู:

```sql
SET enable_bitmapscan = off;
EXPLAIN SELECT * FROM products WHERE category_id = 10;
RESET enable_bitmapscan;
```

```
Seq Scan on products  (cost=0.00..9167.00 rows=10029 width=48)
  Filter: (category_id = 10)
```

น่าสนใจมาก — เมื่อปิด bitmap scan, planner **ไม่เลือก plain Index Scan** เลย แต่กลับไปใช้ **Seq Scan** แทน! เพราะ plain Index Scan สำหรับ selectivity 2% บนคอลัมน์ที่ correlation ต่ำ จะต้องกระโดดไป-กลับระหว่าง index กับ heap แบบ **สุ่มเต็มรูปแบบ** ถึง 10,000 ครั้ง โดยแต่ละครั้งอาจแตะ heap page คนละหน้า (ไม่มีการจัดกลุ่มเหมือน bitmap):

```
plain_index_heap_cost ≈ 10000 × random_page_cost = 10000 × 4.0 = 40000   (แพงกว่า seq scan มาก!)
```

นี่คือคำตอบของคำถามคลาสสิก **"ทำไม planner ไม่ใช้ index ทั้งที่มี index อยู่?"** — เพราะ:

1. **Selectivity ไม่ต่ำพอ** (2% ของตารางถือว่าค่อนข้างสูงสำหรับ plain index scan บน column ที่กระจายสุ่ม)
2. **Correlation ต่ำ** ทำให้แถวที่ match กระจัดกระจายในทาง physical แทนที่จะอยู่ติดกัน
3. **Random access ผ่าน `random_page_cost=4.0`** แพงกว่า sequential access 4 เท่า เมื่อคูณด้วยจำนวนแถวที่มากพอ ต้นทุนรวมจะแซง seq scan ในที่สุด

โดยทั่วไป rule of thumb (ไม่ใช่กฎตายตัว): ถ้า query คาดว่าจะคืนมากกว่าประมาณ **5-15% ของตาราง** (ขึ้นกับ correlation, row width, hardware) มักจะถูกกว่าถ้าใช้ Seq Scan หรือ Bitmap Scan แทน plain Index Scan เสมอ

### กรณีที่ 2: selectivity ต่ำมาก → Index Scan ชนะขาด

```sql
EXPLAIN SELECT * FROM products WHERE product_id = 12345;
```

```
Index Scan using products_pkey on products  (cost=0.42..8.44 rows=1 width=48)
  Index Cond: (product_id = 12345)
```

คำนวณด้วยมือ (แบบประมาณการ):

```
selectivity = 1/500000 (unique key)
matching_rows = 1

index_traversal ≈ height ของ btree สำหรับ 500,000 entries
   height ≈ ceil(log_200(500000)) ≈ 3   (สมมติ ~200 entries ต่อ internal page)
   traversal_cost ≈ 3 × random_page_cost แต่ในความเป็นจริง PostgreSQL คิดค่านี้
   ละเอียดกว่าด้วยสูตร genericcostestimate ที่คำนึงถึง cache hit ratio ของ
   internal/root pages (มักจะ cache อยู่แล้วเสมอเพราะถูกใช้บ่อย) ทำให้ startup cost ต่ำมาก
   → startup_cost ที่ได้จริงคือ 0.42 (ต่ำกว่าที่คำนวณตรงไปตรงมามาก)

heap_access ≈ 1 row × random_page_cost (1 page แบบสุ่ม) = 1 × 4.0 = 4.0
cpu costs   ≈ 1 × (cpu_index_tuple_cost + cpu_operator_cost + cpu_tuple_cost)
            = 1 × (0.005 + 0.0025 + 0.01) = 0.0175

รวมประมาณ ≈ 0.42 (startup) + 4.0 + ~4.0 (บวก overhead อื่นๆ) ≈ 8.44   ✓ ใกล้เคียงตัวเลขจริงมาก
```

สำหรับ query ที่คืนแค่ 1 แถวจาก 500,000 แถว index scan ชนะ seq scan อย่างชัดเจน (8.44 vs 9167.00 — ต่างกันกว่า 1000 เท่า) เพราะไม่ว่า correlation จะต่ำแค่ไหน การอ่าน heap page แบบสุ่มเพียง 1 ครั้งก็ยังถูกกว่าการอ่านทั้งตาราง 4167 pages มหาศาล

### สรุปหลักการ Step 855

| ปัจจัย | ผลต่อ Index Scan cost |
|---|---|
| Selectivity ต่ำ (query คืนน้อยแถว) | Index Scan ถูกลงมาก ชนะ Seq Scan ง่าย |
| Selectivity สูง (query คืนหลายแถว) | Index Scan แพงขึ้นเรื่อยๆ อาจแพงกว่า Seq Scan |
| Correlation สูง (ข้อมูลเรียงตามลำดับ physical) | heap access ถูกลงมาก เพราะแถวที่ match อยู่ติดกัน |
| Correlation ต่ำ (ข้อมูลกระจายสุ่ม) | heap access แพงขึ้น เพราะต้องกระโดดไปมาทั่วตาราง |
| `random_page_cost` สูง (สมมติ HDD) | Index Scan ถูกลงโทษหนักกว่า Seq Scan |
| `random_page_cost` ต่ำ (สมมติ SSD/NVMe) | Index Scan ได้เปรียบมากขึ้น |
| Bitmap Scan (แทน plain Index Scan) | ลด penalty ของ random access โดยการเรียง heap page ก่อนอ่าน |

---

## Step 856: Statistics ที่ Planner ใช้

Planner ไม่มีทางรู้ "ความจริง" ของข้อมูลได้แบบ real-time (การนับทุกแถวทุกครั้งก่อน plan จะช้ามาก) มันจึงอาศัย **สถิติที่เก็บไว้ล่วงหน้า** ใน system catalog ชื่อ `pg_statistic` (เข้าถึงง่ายผ่าน view `pg_stats`) ซึ่งถูกสร้าง/อัปเดตโดยคำสั่ง `ANALYZE`

### โครงสร้างของ pg_stats

```sql
SELECT
    attname,
    null_frac,
    avg_width,
    n_distinct,
    most_common_vals,
    most_common_freqs,
    histogram_bounds,
    correlation
FROM pg_stats
WHERE tablename = 'orders' AND attname = 'status';
```

```
attname            | status
null_frac          | 0
avg_width           | 9
n_distinct          | 5
most_common_vals     | {delivered,shipped,processing,pending,cancelled}
most_common_freqs    | {0.700123,0.150087,0.080041,0.049872,0.019877}
histogram_bounds     | (null - ไม่มี เพราะทุกค่าอยู่ใน MCV list หมดแล้ว)
correlation          | 0.583912
```

อธิบายแต่ละฟิลด์:

- **`null_frac`**: สัดส่วนของแถวที่ column นี้เป็น `NULL` (ที่นี่ = 0 เพราะเราไม่เคย insert NULL)
- **`avg_width`**: ความกว้างเฉลี่ยของค่าในคอลัมน์นี้ (bytes) — ใช้คำนวณ `width` ที่แสดงใน EXPLAIN และประมาณขนาดข้อมูลที่ transfer
- **`n_distinct`**: จำนวนค่าที่ไม่ซ้ำกันโดยประมาณ
  - ถ้าเป็น **บวก** (เช่น `5`) หมายถึงจำนวนค่า distinct จริงโดยประมาณ
  - ถ้าเป็น **ลบ** (เช่น `-0.9` ที่เห็นใน order_date ก่อนหน้านี้) หมายถึง **สัดส่วน** ของ n_distinct ต่อจำนวนแถวทั้งหมด (ตัวอย่างเช่น -0.9 หมายถึงประมาณ 90% ของแถวมีค่าไม่ซ้ำกัน) — planner ใช้ค่าลบเมื่อจำนวนค่า distinct ขยายตามขนาดตารางโดยประมาณเป็นสัดส่วนคงที่ (เหมาะกับคอลัมน์ที่ "เกือบ unique" อย่าง timestamp ที่มีความละเอียดสูง)
- **`most_common_vals` (MCV)**: รายการค่าที่พบบ่อยที่สุด เรียงจากมากไปน้อย
- **`most_common_freqs`**: ความถี่ (สัดส่วน 0.0-1.0) ของแต่ละค่าใน MCV list ตามลำดับ
- **`histogram_bounds`**: สำหรับค่าที่ **ไม่ได้อยู่ใน MCV list** planner ใช้ histogram (equi-depth histogram) แบ่งข้อมูลออกเป็นช่วงๆ ที่มีจำนวนแถวใกล้เคียงกัน เพื่อประมาณ selectivity ของ range query (`>`, `<`, `BETWEEN`)
- **`correlation`**: ค่าสหสัมพันธ์ระหว่างลำดับทางกายภาพของแถวกับค่าของคอลัมน์ (-1.0 ถึง 1.0) ใช้ในการคำนวณ heap access cost ของ index scan ดังที่กล่าวไปใน Step 855

### ดูตัวอย่าง histogram_bounds ของคอลัมน์ตัวเลข

```sql
SELECT
    attname,
    n_distinct,
    most_common_vals,
    histogram_bounds
FROM pg_stats
WHERE tablename = 'products' AND attname = 'unit_price';
```

```
attname          | unit_price
n_distinct        | -0.98543
most_common_vals  | (null - ไม่มีค่าใดซ้ำกันบ่อยพอที่จะติด MCV)
histogram_bounds  | {10.02,59.87,109.43,158.91,...,949.12,999.98}
```

`unit_price` เป็นค่าต่อเนื่อง (continuous, สุ่มด้วย `random()`) เกือบทุกค่าไม่ซ้ำกัน (`n_distinct ≈ -0.985` คือประมาณ 98.5% ของแถวมีค่าไม่ซ้ำ) จึงไม่มี MCV ที่มีความหมาย planner จึงพึ่ง histogram ทั้งหมดในการประมาณ selectivity ของเงื่อนไขแบบ range เช่น `unit_price > 500`

Histogram แบ่งข้อมูลออกเป็น **buckets ที่มีจำนวนแถวเท่ากันโดยประมาณ** (equi-depth/equi-height histogram) — จำนวน buckets ถูกกำหนดโดย `default_statistics_target` (ค่า default = 100 หมายถึงมี histogram สูงสุด ~100 buckets, เราตั้งไว้ 200 ในบทนี้)

### ANALYZE ทำงานอย่างไร (การสุ่มตัวอย่าง)

`ANALYZE` **ไม่ได้อ่านทุกแถวในตาราง** (นั่นจะช้าเกินไปสำหรับตารางใหญ่) แต่ใช้เทคนิค **random sampling** แบบ Vitter's algorithm (reservoir sampling) เพื่อสุ่มตัวอย่างจำนวนหนึ่งมาคำนวณสถิติโดยประมาณ

ขนาดตัวอย่างคำนวณจาก `default_statistics_target` (หรือค่า per-column ที่ตั้งด้วย `ALTER TABLE ... SET STATISTICS`) ด้วยสูตรประมาณ:

```
sample_size = 300 × statistics_target
```

ดังนั้นถ้า `default_statistics_target = 100` (ค่า default) ขนาดตัวอย่างจะประมาณ 30,000 แถว **ไม่ว่าตารางจะมี 1 ล้านแถวหรือ 1 พันล้านแถวก็ตาม** — นี่คือเหตุผลที่ `ANALYZE` เร็วกว่าการ scan ทั้งตารางมาก แต่ก็หมายความว่าสำหรับตารางใหญ่มากๆ ที่มีการกระจายข้อมูลซับซ้อน (เช่นค่าหายาก/outlier) การสุ่มตัวอย่างอาจพลาดรายละเอียดเล็กๆ น้อยๆ ไปได้

```sql
-- ตั้งค่า statistics target สูงขึ้นสำหรับคอลัมน์ที่สำคัญและมีการกระจายซับซ้อน
ALTER TABLE orders ALTER COLUMN status SET STATISTICS 500;

-- ตั้งค่า default สำหรับทั้ง database (ไม่แนะนำถ้าไม่จำเป็น เพราะ ANALYZE จะช้าลงทุกตาราง)
-- ALTER SYSTEM SET default_statistics_target = 200;

-- เพิ่ม statistics target เฉพาะจุดดีกว่าเพิ่มทั้งระบบเสมอ เพราะ ANALYZE คำนวณต่อคอลัมน์
ANALYZE orders;
```

### ผลของ default_statistics_target ต่อความแม่นยำของ selectivity

```sql
-- ลด statistics target ให้ต่ำมากเพื่อดูผลกระทบ
ALTER TABLE orders ALTER COLUMN customer_id SET STATISTICS 10;
ANALYZE orders;

EXPLAIN SELECT * FROM orders WHERE customer_id = 42;
```

```
Index Scan using idx_orders_customer on orders  (cost=0.43..21.47 rows=17 width=54)
  Index Cond: (customer_id = 42)
```

`rows=17` เป็นค่าประมาณจาก histogram ที่หยาบ (เพราะ statistics target ต่ำมาก = 10) ในความเป็นจริง customer_id กระจายสุ่ม 1-100,000 ดังนั้นค่าที่แท้จริงต่อ customer หนึ่งคนควรอยู่แถว 2,000,000/100,000 = 20 แถวโดยเฉลี่ย — ค่าประมาณ 17 ถือว่าใกล้เคียงพอสมควรแม้ statistics target จะต่ำ เพราะการกระจายแบบ uniform นั้นทำนายง่าย (histogram ไม่จำเป็นต้องละเอียดมากก็แม่นยำได้)

```sql
-- คืนค่า statistics target กลับเป็นปกติ
ALTER TABLE orders ALTER COLUMN customer_id SET STATISTICS -1;   -- -1 = ใช้ default_statistics_target
ANALYZE orders;
```

**ในระบบจริง** ปัญหานี้จะรุนแรงกว่านี้มากเมื่อข้อมูลมีการกระจายแบบ skewed หรือมี outlier — เช่นลูกค้า VIP ที่มีออเดอร์เยอะผิดปกติ ในกรณีเหล่านี้การเพิ่ม `SET STATISTICS` เฉพาะคอลัมน์นั้นจะช่วยได้มาก

---

## Step 857: Selectivity Estimation

**Selectivity** คือสัดส่วน (0.0-1.0) ของแถวที่ query คาดว่าจะคืนผล จากทั้งหมด — เป็นตัวเลขที่ planner ใช้คำนวณ `matching_rows` ซึ่งเป็น input สำคัญที่สุดของทุกสูตร cost ที่เราคำนวณมาแล้วในบทนี้

### Equality (`=`) — ใช้ MCV list เป็นหลัก

ถ้าค่าที่ query อยู่ใน MCV list, planner ใช้ความถี่ที่บันทึกไว้ตรงๆ เลย:

```sql
EXPLAIN SELECT * FROM orders WHERE status = 'delivered';
```

```
Seq Scan on orders  (cost=0.00..39286.00 rows=1400246 width=54)
  Filter: ((status)::text = 'delivered'::text)
```

```
selectivity('delivered') = most_common_freqs[1] = 0.700123
matching_rows = 2000000 × 0.700123 ≈ 1400246   ✓ ตรงกับ rows=1400246
```

ถ้าค่าที่ query **ไม่อยู่ใน MCV list** (เช่นค่าที่หายากมาก) planner ใช้สูตรโดยประมาณ:

```
selectivity(ค่านอก MCV) = (1 - Σ MCV freqs) / (n_distinct - จำนวนค่าใน MCV list)
```

พูดง่ายๆ คือ "ส่วนที่เหลือของความน่าจะเป็นทั้งหมด หารเฉลี่ยเท่าๆ กันในบรรดาค่าที่เหลือทั้งหมดที่ไม่ติด MCV" — เป็นการประมาณแบบ **uniform assumption** สำหรับค่าที่ไม่ติด MCV

ในตัวอย่างของเรา `status` มีแค่ 5 ค่าที่เป็นไปได้และทั้ง 5 ค่าติด MCV หมดแล้ว (เพราะ n_distinct=5 พอดี) จึงไม่มีกรณีนี้ให้เห็น แต่ลองดูตัวอย่างสมมติ: ถ้ามีค่าที่ 6 ที่ไม่เคยติด MCV (เพราะหายากมาก) และไม่มี MCV เต็ม เช่น `n_distinct=1000` มี MCV แค่ 50 ค่าแรกที่ความถี่รวมกัน 0.6 การ query ค่านอก MCV จะได้:

```
selectivity = (1 - 0.6) / (1000 - 50) = 0.4 / 950 ≈ 0.000421
```

### Range (`>`, `<`, `BETWEEN`) — ใช้ histogram

```sql
EXPLAIN SELECT * FROM products WHERE unit_price > 500;
```

```
Seq Scan on products  (cost=0.00..10417.00 rows=249831 width=48)
  Filter: (unit_price > '500'::numeric)
```

Planner หาว่าค่า `500` อยู่ตำแหน่งไหนใน `histogram_bounds` array (interpolation เชิงเส้นระหว่างขอบ bucket สองข้าง) แล้วคำนวณว่ากี่ % ของ buckets ที่อยู่**เหนือ**ค่านั้น

```
สมมติ histogram_bounds มี 201 ค่า (200 buckets เท่าๆ กัน เพราะเราตั้ง STATISTICS 200)
สมมติค่า 500 อยู่ที่ตำแหน่งประมาณ bucket ที่ 100 จาก 200 (กึ่งกลางพอดี เพราะราคาสุ่มสม่ำเสมอ 10-1000)

selectivity(unit_price > 500) ≈ (200 - 100) / 200 = 0.5

matching_rows = 500000 × 0.5 = 250000   ≈ ใกล้เคียง rows=249831 มาก
```

ความแม่นยำสูงเพราะราคาถูกสุ่มแบบ uniform distribution จริงๆ — histogram จับรูปแบบนี้ได้ดี ถ้าข้อมูลจริงมีการกระจายแบบ skewed มาก (เช่นสินค้าราคาถูกเยอะกว่าสินค้าราคาแพงมาก) histogram ก็ยังจับรูปแบบนั้นได้เพราะ equi-depth histogram ปรับ width ของแต่ละ bucket ให้เหมาะกับความหนาแน่นของข้อมูลอัตโนมัติ (bucket ที่มีข้อมูลหนาแน่นจะแคบ, bucket ที่ข้อมูลเบาบางจะกว้าง)

### หลายเงื่อนไข (AND) — สมมติฐาน Independence

นี่คือจุดที่ planner ผิดพลาดบ่อยที่สุดในโลกจริง เมื่อมีหลายเงื่อนไข AND กัน planner (โดย default ที่ไม่มี extended statistics) จะ **สมมติว่าแต่ละเงื่อนไข independent จากกัน (ไม่สัมพันธ์กัน)** แล้วคูณ selectivity เข้าด้วยกัน:

```
selectivity(A AND B) = selectivity(A) × selectivity(B)      -- ถ้าสมมติ independent
```

```sql
EXPLAIN SELECT * FROM orders WHERE status = 'delivered' AND customer_id > 90000;
```

```
Seq Scan on orders  (cost=0.00..44286.00 rows=140068 width=54)
  Filter: (((status)::text = 'delivered'::text) AND (customer_id > 90000))
```

```
selectivity(status='delivered') = 0.700123
selectivity(customer_id > 90000) ≈ (100000 - 90000) / 100000 = 0.10   -- จาก histogram

selectivity(AND) = 0.700123 × 0.10 = 0.0700123
matching_rows = 2000000 × 0.0700123 ≈ 140025   ≈ ใกล้เคียง rows=140068
```

ในกรณีนี้สมมติฐาน independence **ใช้ได้ดี** เพราะ `status` และ `customer_id` ในข้อมูลของเราไม่มีความสัมพันธ์กันจริงๆ (ถูกสุ่มแยกกันคนละตัวแปร) — แต่ **ในโลกจริง คอลัมน์หลายคู่มีความสัมพันธ์กัน (correlated)** เช่น `city` กับ `postal_code`, หรือ `category_id` กับ `unit_price` (บางหมวดสินค้าแพงกว่าหมวดอื่นเป็นกลุ่มๆ) เมื่อสมมติฐาน independence ผิด selectivity ที่ประมาณได้จะคลาดเคลื่อนไปมาก — บางครั้ง overestimate บางครั้ง underestimate อย่างรุนแรง เราจะแก้ปัญหานี้ด้วย extended statistics ใน Step 859

### OR conditions

สำหรับ `OR` planner ใช้หลักการความน่าจะเป็นแบบรวม (inclusion-exclusion โดยประมาณ):

```
selectivity(A OR B) = selectivity(A) + selectivity(B) - selectivity(A) × selectivity(B)
```

```sql
EXPLAIN SELECT * FROM orders WHERE status = 'cancelled' OR status = 'pending';
```

```
Seq Scan on orders  (cost=0.00..39286.00 rows=138485 width=54)
  Filter: (((status)::text = 'cancelled'::text) OR ((status)::text = 'pending'::text))
```

```
selectivity(cancelled) = 0.019877
selectivity(pending)   = 0.049872

selectivity(OR) = 0.019877 + 0.049872 - (0.019877 × 0.049872)
                = 0.069749 - 0.000991
                = 0.068758

matching_rows = 2000000 × 0.068758 ≈ 137516   ≈ ใกล้เคียง rows=138485
```

> ในกรณีเฉพาะของ `status = 'a' OR status = 'b'` บนคอลัมน์เดียวกัน PostgreSQL มักแปลงเป็น `status = ANY(ARRAY['a','b'])` ภายในและคำนวณ selectivity โดยรวมความถี่จาก MCV โดยตรง (ซึ่งแม่นยำกว่าสูตร inclusion-exclusion ทั่วไปเล็กน้อย) แต่หลักการโดยรวมยังคล้ายกัน

---

## Step 858: Join Order Optimization

### ปัญหาการระเบิดของ join order (combinatorial explosion)

เมื่อ query มีการ JOIN หลายตาราง จำนวน **ลำดับการ join ที่เป็นไปได้ (join order)** เพิ่มขึ้นแบบ factorial ตามจำนวนตาราง สำหรับ **left-deep join tree** (join ทีละตารางเข้ากับผลลัพธ์สะสม) จำนวนลำดับที่เป็นไปได้คือ `n!` (n แฟกทอเรียล):

| จำนวนตาราง | จำนวน join order ที่เป็นไปได้ (n!) |
|---|---|
| 2 | 2 |
| 3 | 6 |
| 5 | 120 |
| 8 | 40,320 |
| 10 | 3,628,800 |
| 12 | 479,001,600 |
| 15 | 1,307,674,368,000 |

และนี่ยังไม่นับ **bushy join tree** (join ผลลัพธ์ย่อยสองก้อนเข้าด้วยกันโดยไม่ผ่านตารางต้นฉบับโดยตรง) ที่ทำให้จำนวนทางเลือกยิ่งมากขึ้นไปอีก รวมถึงการเลือก **join method** (Nested Loop, Hash Join, Merge Join) และ **join order ของแต่ละคู่** ที่คูณเข้าไปอีก — จะเห็นว่าถ้า PostgreSQL พยายาม enumerate ทุกความเป็นไปได้แบบ brute-force สำหรับ query ที่มี 10+ ตาราง จะใช้เวลาวางแผน (planning time) นานกว่าการ execute จริงเสียอีก

### Dynamic Programming (สำหรับตารางจำนวนน้อย-ปานกลาง)

สำหรับ query ที่มีจำนวนตาราง**ไม่เกิน `geqo_threshold`** (default = 12) PostgreSQL ใช้ **dynamic programming** แบบ exhaustive search ที่ฉลาดกว่า brute-force ธรรมดา — มันไม่คำนวณ cost ของทุก permutation ซ้ำตั้งแต่ต้น แต่สร้างและเก็บ (memoize) cost ของ "subset ของตารางที่ join กันแล้ว" ไว้ใช้ซ้ำ ทำให้ความซับซ้อนลดลงจาก `O(n!)` เหลือประมาณ `O(3^n)` หรือ `O(n × 2^n)` ขึ้นกับรายละเอียด implementation ซึ่งยังคงเติบโตเร็วแต่จัดการได้ในทางปฏิบัติสำหรับ n ที่ไม่เกิน ~12

```sql
SHOW geqo_threshold;    -- 12
SHOW join_collapse_limit;   -- 8
SHOW from_collapse_limit;   -- 8
```

### ตัวอย่าง join order จริง

```sql
CREATE TABLE categories (
    category_id SERIAL PRIMARY KEY,
    category_name VARCHAR(100)
);

INSERT INTO categories (category_name)
SELECT 'Category ' || i FROM generate_series(1, 50) AS i;

ANALYZE categories;

EXPLAIN
SELECT c.category_name, p.product_name, o.order_date
FROM orders o
JOIN products p ON p.product_id = (o.order_id % 500000) + 1   -- join เทียมเพื่อสาธิต
JOIN categories c ON c.category_id = p.category_id
WHERE o.status = 'delivered'
LIMIT 100;
```

```
Limit  (cost=0.72..487.35 rows=100 width=72)
  ->  Nested Loop  (cost=0.72..9727468.21 rows=1400246 width=72)
        ->  Nested Loop  (cost=0.72..... rows=1400246 width=...)
              ->  Seq Scan on orders o  (cost=0.00..39286.00 rows=1400246 width=12)
                    Filter: ((status)::text = 'delivered'::text)
              ->  Index Scan using products_pkey on products p  (cost=0.42..0.51 rows=1 width=...)
                    Index Cond: (product_id = ((o.order_id % 500000) + 1))
        ->  Index Scan using categories_pkey on categories c  (cost=0.15..0.17 rows=1 width=...)
              Index Cond: (category_id = p.category_id)
```

สังเกตว่า planner เลือก **join order**: `orders → products → categories` เพราะ `orders` ที่ filter ด้วย `status='delivered'` แล้ว (1.4M แถว) เป็นจุดเริ่มต้นตามธรรมชาติของ Nested Loop ที่ join กับ `products` ผ่าน primary key lookup (ถูกมาก เพราะ unique index) แล้วจึง join กับ `categories` ต่อ ก็ผ่าน primary key lookup เช่นกัน — planner เลือกลำดับนี้เพราะ **การ join ผ่าน unique index บนฝั่ง "ตารางเล็ก" ทำได้เร็วมาก ไม่ว่าจะ join ตารางไหนก่อนก็ตาม** (cost ของแต่ละ index lookup ต่ำมากอยู่แล้ว) ในกรณีนี้ join order ไม่ส่งผลกระทบมากเพราะทุก join step ถูกมากอยู่แล้ว — แต่ในกรณีที่ตารางมีขนาดใหญ่ทั้งคู่และไม่มี unique index ที่ match ได้ ลำดับการ join จะมีผลกระทบต่อ cost อย่างมหาศาล เพราะ intermediate result set ขนาดใหญ่ที่เกิดจากการ join ผิดลำดับจะทำให้ step ถัดไปแพงขึ้นแบบทวีคูณ

```sql
DROP TABLE categories;
```

### GEQO (Genetic Query Optimizer) — สำหรับ query ที่มีตารางเยอะมาก

เมื่อจำนวนตารางใน `FROM`/`JOIN` เกิน `geqo_threshold` (default 12) PostgreSQL เปลี่ยนไปใช้ **Genetic Algorithm** แทน exhaustive dynamic programming เพื่อหลีกเลี่ยงเวลา planning ที่นานเกินไป

หลักการทำงานคร่าวๆ ของ GEQO:

1. สร้าง "ประชากร" (population) ของ join order ที่สุ่มขึ้นมาจำนวนหนึ่ง — แต่ละ join order คือหนึ่ง "chromosome" (แทนด้วยลำดับของตาราง)
2. คำนวณ "fitness" ของแต่ละ chromosome (fitness สูง = cost ต่ำ)
3. เลือก chromosome ที่ fitness ดีมาผสมพันธุ์ (crossover) และกลายพันธุ์ (mutation) สร้างประชากรรุ่นใหม่
4. ทำซ้ำหลายรอบ (generations) จนกว่าจะเจอ join order ที่ "ดีพอ" (ไม่รับประกันว่าดีที่สุด แต่ดีในระดับที่ยอมรับได้)

```sql
SHOW geqo;                 -- on (เปิดใช้งานโดย default)
SHOW geqo_threshold;       -- 12
SHOW geqo_effort;          -- 5 (ค่า 1-10, สูง = ค้นหาละเอียดขึ้นแต่ช้าลง)
SHOW geqo_generations;     -- 0 (0 = auto-calculate จาก pool size)
SHOW geqo_pool_size;       -- 0 (0 = auto-calculate จากจำนวนตาราง)
```

**ข้อแลกเปลี่ยน (trade-off)**: GEQO ให้ planning time ที่คาดเดาได้และไม่ระเบิดเมื่อ query มีตารางเยอะมาก แต่ **ไม่รับประกันว่าได้ join order ที่ optimal ที่สุด** — มันเป็น heuristic search ที่อาจพลาด join order ที่ดีที่สุดจริงๆ ไปได้ ในทางปฏิบัติ query ที่มีมากกว่า 12 ตารางมักเกิดจาก ORM ที่ generate SQL อัตโนมัติ (เช่น query ที่ join หลาย table ผ่าน many-to-many relations ซ้อนกันหลายชั้น) — ถ้าเจอปัญหา planning time สูงหรือแผนที่ไม่ดีจาก GEQO ทางแก้คือ:

1. **ลดจำนวนตารางที่ join จริง** ด้วยการ restructure query (เช่นใช้ CTE materialize บางส่วน หรือแบ่ง query ย่อย)
2. **เพิ่ม `geqo_threshold`** ชั่วคราวถ้าตารางไม่เกิน ~15-16 ตาราง (dynamic programming ยังพอจัดการได้ แต่ planning time จะสูงขึ้น)
3. **ใช้ `JOIN` ที่มี explicit order ที่ดีอยู่แล้ว** ร่วมกับการลด `join_collapse_limit`/`from_collapse_limit` เพื่อบังคับให้ planner เคารพลำดับที่เขียนใน SQL (เทคนิคนี้ใช้เมื่อผู้เขียน query รู้ join order ที่ดีที่สุดอยู่แล้วจากความเข้าใจ domain)

```sql
-- ตัวอย่าง: บังคับให้ planner เคารพลำดับ JOIN ที่เขียนไว้ ไม่ต้อง explore ลำดับอื่น
SET join_collapse_limit = 1;
-- (ใช้ระวัง เพราะถ้าลำดับที่เขียนไม่ดีจริง cost จะแย่กว่าเดิม)
RESET join_collapse_limit;
```

---

## Step 859: Extended Statistics เชิงลึก

เราเคยแนะนำ `CREATE STATISTICS` ใน Part 074 มาแล้วในระดับพื้นฐาน บทนี้จะเจาะลึกไปที่ **ทำไม** มันจำเป็น และ **กลไกภายใน** ที่แก้ปัญหา correlated columns

### ปัญหา: correlated columns ทำให้ selectivity ผิดพลาด

ย้อนกลับไป Step 857 เราพูดถึงว่า planner สมมติ **independence** ระหว่างคอลัมน์เมื่อมีหลายเงื่อนไข AND กัน สมมติฐานนี้ผิดพลาดรุนแรงเมื่อคอลัมน์มีความสัมพันธ์กันจริง มาสร้างตัวอย่างที่คอลัมน์สัมพันธ์กันโดยตั้งใจ:

```sql
-- เพิ่มคอลัมน์ shipping_region ที่สัมพันธ์กับ status
-- (ในทางปฏิบัติ: ออเดอร์ที่ status='cancelled' มักจะกระจุกตัวอยู่ใน region ที่มีปัญหาด้าน logistics)
ALTER TABLE orders ADD COLUMN shipping_region VARCHAR(20);

UPDATE orders SET shipping_region =
    CASE
        WHEN status = 'cancelled' AND random() < 0.8 THEN 'remote-area'
        WHEN status = 'cancelled' THEN (ARRAY['north','south','central'])[floor(random()*3+1)]
        ELSE (ARRAY['north','south','central','remote-area'])[floor(random()*4+1)]
    END;

ANALYZE orders;
```

ในข้อมูลชุดนี้ `status='cancelled'` กับ `shipping_region='remote-area'` มีความสัมพันธ์กันแบบไม่เป็นอิสระ (correlated) อย่างจงใจ — ออเดอร์ที่ยกเลิก 80% อยู่ใน remote-area

```sql
EXPLAIN ANALYZE
SELECT * FROM orders
WHERE status = 'cancelled' AND shipping_region = 'remote-area';
```

```
Seq Scan on orders  (cost=0.00..44286.00 rows=1988 width=63) (actual time=45.201..198.774 rows=31842 loops=1)
  Filter: (((status)::text = 'cancelled'::text) AND ((shipping_region)::text = 'remote-area'::text))
  Rows Removed by Filter: 1968158
```

**ตรงนี้คือปัญหาชัดเจน**: planner ประมาณ `rows=1988` แต่ค่าจริงคือ `rows=31842` — **คลาดเคลื่อนไปเกือบ 16 เท่า!**

ตรวจสอบด้วยการคำนวณ:

```
selectivity(status='cancelled') ≈ 0.02   (2%)
selectivity(shipping_region='remote-area') ≈ 0.25   (สมมติกระจายสม่ำเสมอ 4 ภูมิภาค)

ถ้าสมมติ independent:
selectivity(AND) = 0.02 × 0.25 = 0.005
matching_rows = 2000000 × 0.005 = 10000   -- ยังไม่ตรงกับ 1988 เป๊ะ เพราะ MCV จริงของ
                                             shipping_region ก็บิดเบือนจากการ correlate ด้วย
                                             แต่แนวโน้ม underestimate ชัดเจนมาก
```

ในขณะที่ความจริง เพราะ `cancelled` 80% ตกอยู่ใน `remote-area` selectivity ที่แท้จริงของ AND นี้สูงกว่าที่สมมติฐาน independence คาดไว้มาก — planner "มองไม่เห็น" ความสัมพันธ์นี้เลยถ้าไม่มี extended statistics

### ผลกระทบของการประมาณผิดต่อแผนการ (ไม่ใช่แค่ตัวเลขสวยๆ)

การประมาณ `rows=1988` ที่ต่ำเกินจริงมากอาจทำให้ planner เลือกใช้ **Nested Loop Join** (เหมาะกับ outer ที่มีแถวน้อย) ในขณะที่ query จริงมี 31,842 แถว ซึ่งเหมาะกับ **Hash Join** มากกว่า — ผลคือแผนที่ planner เลือกอาจช้ากว่าที่ควรจะเป็นมาก เพราะสมมติฐานเรื่องจำนวนแถวผิดพลาดไปมาก (ปัญหานี้จะยิ่งชัดเจนขึ้นเมื่อ query นี้เป็นแค่ subquery หนึ่งใน join ที่ใหญ่กว่า เพราะความผิดพลาดจะ "ลุกลาม" ไปยัง estimation ของ join ถัดๆ ไปด้วย)

### แก้ปัญหาด้วย `CREATE STATISTICS`

```sql
-- ใช้ statistics kind "dependencies" เพื่อจับความสัมพันธ์แบบ functional dependency
-- และ kind "mcv" เพื่อเก็บ multi-column MCV ที่แม่นยำสำหรับค่าที่พบบ่อย
CREATE STATISTICS stx_orders_status_region (dependencies, mcv, ndistinct)
    ON status, shipping_region
    FROM orders;

ANALYZE orders;
```

```sql
EXPLAIN ANALYZE
SELECT * FROM orders
WHERE status = 'cancelled' AND shipping_region = 'remote-area';
```

```
Seq Scan on orders  (cost=0.00..44286.00 rows=31654 width=63) (actual time=44.987..201.302 rows=31842 loops=1)
  Filter: (((status)::text = 'cancelled'::text) AND ((shipping_region)::text = 'remote-area'::text))
  Rows Removed by Filter: 1968158
```

**หลังสร้าง extended statistics `rows=31654`** ใกล้เคียงกับค่าจริง `rows=31842` มาก (คลาดเคลื่อนไม่ถึง 1%) เทียบกับก่อนหน้าที่คลาดเคลื่อนเกือบ 16 เท่า — นี่คือพลังของ extended statistics

### ตรวจสอบ extended statistics ที่สร้างไว้

```sql
SELECT
    stxname,
    stxkeys,
    stxkind
FROM pg_statistic_ext
WHERE stxname = 'stx_orders_status_region';
```

```
stxname                    | stx_orders_status_region
stxkeys                     | 5 8      (attnum ของ status และ shipping_region)
stxkind                     | {d,f,m}  (d=ndistinct, f=dependencies, m=mcv)
```

ดูข้อมูล multivariate MCV ที่เก็บไว้จริง:

```sql
SELECT
    m.*
FROM pg_statistic_ext_data d,
     pg_mcv_list_items(d.stxdmcv) m
JOIN pg_statistic_ext s ON s.oid = d.stxoid
WHERE s.stxname = 'stx_orders_status_region'
ORDER BY m.frequency DESC
LIMIT 5;
```

```
 index |        values         | nulls  | frequency | base_frequency
-------+------------------------+--------+-----------+----------------
     0 | {delivered, north}     | {f, f} |  0.174532 |       0.175031
     1 | {delivered, south}     | {f, f} |  0.174398 |       0.175031
     2 | {cancelled, remote-area}| {f, f} |  0.015921 |       0.004969   <-- ตรงนี้!
     3 | {delivered, central}   | {f, f} |  0.174201 |       0.175031
     4 | {delivered, remote-area}| {f, f} |  0.176987 |       0.175031
```

สังเกตแถว `{cancelled, remote-area}`: `frequency` (ค่าจริงที่วัดได้) = 0.015921 แต่ `base_frequency` (ค่าที่จะได้ถ้าสมมติ independent = selectivity(cancelled) × selectivity(remote-area)) = 0.004969 — **frequency จริงสูงกว่า base_frequency ถึง 3.2 เท่า** นี่คือหลักฐานตัวเลขที่ยืนยัน correlation ที่เราสร้างขึ้นโดยตั้งใจ และเป็นข้อมูลที่ extended statistics เก็บไว้เพื่อแก้ไขการประมาณของ planner โดยตรง

### `dependencies` vs `mcv` vs `ndistinct` — ใช้เมื่อไหร่

| Statistics kind | ใช้เมื่อไหร่ |
|---|---|
| `ndistinct` | GROUP BY หลายคอลัมน์พร้อมกัน ที่คอลัมน์เหล่านั้นมีความสัมพันธ์กัน (เช่น `GROUP BY city, state` ที่จริงๆ แล้ว city กำหนด state) ช่วยประมาณจำนวน group ได้แม่นยำขึ้น |
| `dependencies` | เมื่อคอลัมน์หนึ่งมีความสัมพันธ์แบบ (soft) functional dependency กับอีกคอลัมน์ — ใช้ได้ดีกับ equality condition (`=`) เป็นหลัก คำนวณเร็วกว่า mcv |
| `mcv` | เมื่อต้องการความแม่นยำสูงสุดสำหรับ**ค่าที่พบบ่อย**ที่เกิดจากการรวมกันของหลายคอลัมน์ ใช้ได้ทั้ง equality และ range condition (ผ่าน MCV list ของ combination) ใช้พื้นที่เก็บข้อมูลมากกว่า |

```sql
-- ลบ extended statistics เมื่อไม่ใช้แล้ว (เช่นในการทำความสะอาดหลังทดลอง)
-- DROP STATISTICS stx_orders_status_region;
```

> **ข้อควรระวัง**: extended statistics เพิ่มเวลาให้ `ANALYZE` (ต้องคำนวณ multivariate stats เพิ่ม) และเพิ่มพื้นที่เก็บสถิติ ควรสร้างเฉพาะคู่/กลุ่มคอลัมน์ที่**รู้ชัดเจน**ว่าสัมพันธ์กันและถูกใช้ร่วมกันบ่อยใน WHERE clause เท่านั้น ไม่ควรสร้างแบบเหวี่ยงแหทุกคู่คอลัมน์ในทุกตาราง

---

## สรุปท้ายบท

ในบทนี้เราได้เจาะลึกเข้าไปในกลไกที่แท้จริงเบื้องหลังคำว่า "cost" ที่ปรากฏใน `EXPLAIN` ซึ่งหลายคนมองข้ามไปว่าเป็นแค่ "ตัวเลขของ planner" โดยไม่เข้าใจที่มา:

- **Cost-Based Optimizer** ของ PostgreSQL ประเมิน cost ของแผนการที่เป็นไปได้หลายแบบ แล้วเลือกแผนที่ cost ต่ำสุด — cost เป็น **หน่วยสมมติ** เทียบสัดส่วนกับ `seq_page_cost = 1.0` ไม่ใช่หน่วยเวลา
- พารามิเตอร์ cost หลัก (`seq_page_cost`, `random_page_cost`, `cpu_tuple_cost`, `cpu_index_tuple_cost`, `cpu_operator_cost`) ควบคุมทิศทางการตัดสินใจของ planner โดยตรง โดยเฉพาะ `random_page_cost` ที่ควรปรับให้เหมาะกับ SSD/NVMe แทนค่า default ที่ออกแบบมาสำหรับ HDD
- **Sequential Scan cost** คำนวณตรงไปตรงมาจาก `relpages × seq_page_cost + reltuples × cpu_tuple_cost` (บวก `cpu_operator_cost` ถ้ามี filter) — ต้องอ่านและประเมินทุกแถวเสมอไม่ว่า selectivity จะเป็นเท่าไหร่
- **Index Scan cost** ซับซ้อนกว่ามาก ต้องพิจารณา index traversal, heap access ผ่านสูตร Mackert & Lohman ที่คำนึงถึง **correlation** — selectivity สูงและ correlation ต่ำทำให้ index scan แพงจนแพ้ seq scan ได้ ซึ่งอธิบายคำถามคลาสสิก "ทำไม planner ไม่ใช้ index"
- `pg_stats` (`n_distinct`, `most_common_vals`, `histogram_bounds`, `correlation`) คือข้อมูลดิบที่ `ANALYZE` สร้างจากการสุ่มตัวอย่าง (~300 × statistics_target แถว) และเป็นรากฐานของทุกการคำนวณ selectivity
- **Selectivity estimation** ใช้ MCV list สำหรับ equality และ histogram สำหรับ range — สำหรับหลายเงื่อนไข AND, planner สมมติ **independence** ซึ่งผิดพลาดรุนแรงเมื่อคอลัมน์ correlated กันจริง
- **Join order** มีความซับซ้อนแบบ `n!` — PostgreSQL ใช้ dynamic programming สำหรับตารางไม่เกิน `geqo_threshold` (default 12) และเปลี่ยนไปใช้ **genetic algorithm (GEQO)** เมื่อเกินเพื่อควบคุม planning time
- **Extended Statistics** (`CREATE STATISTICS` ด้วย `dependencies`, `mcv`, `ndistinct`) แก้ปัญหา correlated column ที่ทำให้สมมติฐาน independence ผิดพลาด เราแสดงให้เห็นตัวอย่างจริงที่ estimation คลาดเคลื่อน 16 เท่า และลดลงเหลือไม่ถึง 1% หลังสร้าง extended statistics

**ข้อคิดสำคัญที่สุดของบทนี้**: การคำนวณ cost ด้วยมือไม่ได้ให้คำตอบที่ตรงเป๊ะ 100% เสมอไป (เพราะสูตรจริงใน `costsize.c` และ `selfuncs.c` มีรายละเอียดปลีกย่อยมากกว่านี้) แต่มันสอน **สัญชาตญาณ (intuition)** ที่ถูกต้องว่า factor ไหนมีผลต่อการตัดสินใจของ planner — และสัญชาตญาณนี้คือสิ่งที่ทำให้ DBA/engineer ระดับ expert วินิจฉัยปัญหา query ที่ช้าได้อย่างรวดเร็ว แทนที่จะลองผิดลองถูกไปเรื่อยๆ

ทำความสะอาดข้อมูลทดลองก่อนไปบทถัดไป:

```sql
ALTER TABLE orders DROP COLUMN IF EXISTS shipping_region;
DROP STATISTICS IF EXISTS stx_orders_status_region;
ANALYZE orders;
ANALYZE products;
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1

จงอธิบายด้วยคำพูดของตัวเองว่าทำไม cost ใน `EXPLAIN` ไม่ใช่หน่วยเวลา (วินาที/มิลลิวินาที) และยกตัวอย่างสถานการณ์ที่ cost เท่าเดิมแต่เวลาจริงต่างกันมาก

<details>
<summary>เฉลย</summary>

Cost เป็นหน่วยสมมติที่ planner คำนวณ**ก่อน**การ execute จริง โดยอิงจากพารามิเตอร์ cost (`seq_page_cost`, `random_page_cost` ฯลฯ) และสถิติที่เก็บไว้ ไม่ได้วัดจากการรันจริง จึงไม่มีทางรู้ว่า data จะอยู่ใน cache (shared_buffers/OS page cache) หรือไม่ ณ เวลาที่รันจริง

ตัวอย่างสถานการณ์: รัน `EXPLAIN (ANALYZE, BUFFERS) SELECT count(*) FROM orders;` สองครั้งติดกัน — cost ที่แสดง (`cost=0.00..29286.00`) จะเหมือนกันทุกครั้งเพราะมาจากการคำนวณ static ก่อนรัน แต่รอบแรกอาจต้องอ่านจาก disk จริง (`Buffers: shared read=13056`) ทำให้ actual time สูง ขณะที่รอบสองข้อมูลอยู่ใน cache หมดแล้ว (`Buffers: shared hit=14286`) ทำให้ actual time ต่ำกว่ามาก แม้ cost จะเท่าเดิมทุกประการ

</details>

### แบบฝึกหัดที่ 2

ตารางหนึ่งมี `relpages = 8000` และ `reltuples = 1200000` จงคำนวณ cost ของ `Seq Scan` แบบไม่มี WHERE clause ด้วยค่า cost parameter เริ่มต้น (`seq_page_cost=1.0`, `cpu_tuple_cost=0.01`)

<details>
<summary>เฉลย</summary>

```
seq_scan_cost = (relpages × seq_page_cost) + (reltuples × cpu_tuple_cost)
              = (8000 × 1.0) + (1200000 × 0.01)
              = 8000 + 12000
              = 20000.00
```

`EXPLAIN` ควรแสดง `cost=0.00..20000.00`

</details>

### แบบฝึกหัดที่ 3

query เดียวกันจากแบบฝึกหัดที่ 2 แต่ตอนนี้มี `WHERE some_column = 5` (1 filter condition) จงคำนวณ cost ใหม่ด้วย `cpu_operator_cost = 0.0025`

<details>
<summary>เฉลย</summary>

```
seq_scan_cost = (8000 × 1.0) + (1200000 × 0.01) + (1200000 × 0.0025)
              = 8000 + 12000 + 3000
              = 23000.00
```

สังเกตว่า cost เพิ่มขึ้นแม้ query อาจคืนแค่ไม่กี่แถว เพราะ Seq Scan ต้องอ่านและประเมิน filter กับ**ทุกแถว**เสมอ ไม่ใช่แค่แถวที่ผ่านเงื่อนไข

</details>

### แบบฝึกหัดที่ 4

จงอธิบายว่าทำไม `random_page_cost` มีค่า default เป็น `4.0` ทั้งที่ SSD สมัยใหม่แทบไม่มีความแตกต่างระหว่างการอ่านแบบ sequential กับ random และควรปรับค่านี้อย่างไรถ้าระบบใช้ NVMe SSD

<details>
<summary>เฉลย</summary>

ค่า `4.0` มาจากยุคที่ storage หลักเป็น spinning HDD ซึ่งการอ่านแบบ random ต้องขยับหัวอ่าน (seek) ทำให้ช้ากว่าการอ่านแบบ sequential (อ่านต่อเนื่อง ไม่ต้องขยับหัวอ่าน) หลายเท่า ค่า 4:1 เป็นอัตราส่วนโดยประมาณที่เหมาะกับ HDD ทั่วไปในยุคนั้น

สำหรับ NVMe SSD ที่ไม่มีหัวอ่านเชิงกล การอ่านแบบ random แทบไม่ช้ากว่า sequential เลย ควรปรับค่า `random_page_cost` ให้ต่ำลงมาก เช่น `1.0` - `1.5` เพื่อให้ planner ไม่ penalize index scan (ซึ่งมักเข้าถึง heap แบบ random) เกินความเป็นจริง ตัวอย่างคำสั่ง:

```sql
ALTER SYSTEM SET random_page_cost = 1.1;
SELECT pg_reload_conf();
```

</details>

### แบบฝึกหัดที่ 5

ตาราง `orders` (2,000,000 แถว) มี `correlation = -0.0041` สำหรับคอลัมน์ `order_date` (correlation ต่ำมาก) จงอธิบายว่าเหตุการณ์นี้ผิดปกติหรือไม่สำหรับ column ประเภท timestamp และในระบบจริงที่ insert แถวใหม่ตามลำดับเวลาเสมอ ค่า correlation ของ timestamp ควรเป็นอย่างไร

<details>
<summary>เฉลย</summary>

ค่า correlation ต่ำในตัวอย่างนี้เกิดจากวิธีการเติมข้อมูลในบทเรียน — เราสุ่มวันที่ย้อนหลังแบบสุ่ม (`now() - random() * interval '1095 days'`) ทำให้ลำดับทางกายภาพของแถวในดิสก์ (เรียงตามลำดับ insert) ไม่สัมพันธ์กับค่า `order_date` เลย

แต่ในระบบจริงที่แถวใหม่ถูก insert ตามลำดับเวลาจริงเสมอ (เช่น ออเดอร์ที่สร้างวันนี้ถูก insert วันนี้) ค่า `order_date` ที่เป็น "เวลาสร้างแถว" จะมี correlation สูงเกือบ 1.0 เพราะแถวใหม่ถูกต่อท้ายไฟล์ตารางเสมอ ทำให้ค่า timestamp ที่มากขึ้นเรื่อยๆ สัมพันธ์กับตำแหน่งทางกายภาพที่อยู่ท้ายไฟล์มากขึ้นเรื่อยๆ ซึ่งเป็นเหตุผลสำคัญที่ index scan บน timestamp column แบบนี้มักเร็วกว่าที่คาดในทางทฤษฎีมาก เพราะ heap access เกือบจะเป็น sequential access อยู่แล้ว

</details>

### แบบฝึกหัดที่ 6

จงเขียน query เพื่อดูค่า `most_common_vals` และ `most_common_freqs` ของคอลัมน์ `status` ในตาราง `orders` แล้วอธิบายว่า planner จะประมาณ `matching_rows` สำหรับ `WHERE status = 'shipped'` อย่างไร (สมมติ 2,000,000 แถวทั้งหมด)

<details>
<summary>เฉลย</summary>

```sql
SELECT most_common_vals, most_common_freqs
FROM pg_stats
WHERE tablename = 'orders' AND attname = 'status';
```

Planner หาตำแหน่งของ `'shipped'` ใน `most_common_vals` array แล้วใช้ค่าความถี่ที่ตำแหน่งเดียวกันใน `most_common_freqs` (ในตัวอย่างของบทนี้ประมาณ `0.150087`) แล้วคูณด้วยจำนวนแถวทั้งหมด:

```
matching_rows = 2000000 × 0.150087 ≈ 300174
```

</details>

### แบบฝึกหัดที่ 7

query หนึ่งมี `WHERE category_id = 10 AND unit_price > 800` โดย `category_id` มี selectivity 0.02 และ `unit_price > 800` มี selectivity 0.2 (จาก histogram) จงคำนวณ selectivity รวมภายใต้สมมติฐาน independence และคำนวณ `matching_rows` จากตาราง products (500,000 แถว)

<details>
<summary>เฉลย</summary>

```
selectivity(AND) = selectivity(category_id=10) × selectivity(unit_price>800)
                  = 0.02 × 0.2
                  = 0.004

matching_rows = 500000 × 0.004 = 2000
```

หมายเหตุ: นี่เป็นเพียงค่าประมาณภายใต้สมมติฐาน independence — ถ้า `category_id` และ `unit_price` มีความสัมพันธ์กันจริง (เช่นบางหมวดหมู่มีแต่สินค้าราคาแพง) ค่าประมาณนี้อาจคลาดเคลื่อนได้มาก ต้องใช้ extended statistics เพื่อแก้ไข

</details>

### แบบฝึกหัดที่ 8

จงอธิบายว่า Genetic Query Optimizer (GEQO) ทำงานต่างจาก Dynamic Programming อย่างไร และทำไม PostgreSQL ถึงต้องมีกลไกทั้งสองแบบแทนที่จะใช้แบบใดแบบหนึ่งเสมอ

<details>
<summary>เฉลย</summary>

Dynamic Programming ค้นหา join order ที่ดีที่สุดแบบ **exhaustive** (ครบทุกความเป็นไปได้ โดยใช้เทคนิค memoization ลดการคำนวณซ้ำ) รับประกันว่าได้แผนที่ดีที่สุดจริงๆ แต่ความซับซ้อนเติบโตเร็วมากตามจำนวนตาราง (`O(3^n)` โดยประมาณ) ทำให้ไม่เหมาะกับ query ที่มีตารางจำนวนมาก (เช่น 20+ ตาราง) เพราะ planning time เองจะนานเกินไป

GEQO ใช้ genetic algorithm เป็น heuristic search ที่ค้นหาเฉพาะบางส่วนของพื้นที่คำตอบ (ผ่าน population, crossover, mutation) ทำให้ planning time คาดเดาได้และไม่ระเบิดตามจำนวนตาราง แต่**ไม่รับประกัน**ว่าจะได้ join order ที่ดีที่สุด อาจพลาดคำตอบที่ optimal จริงๆ ไป

PostgreSQL ใช้ทั้งสองแบบเพราะ trade-off ต่างกัน: สำหรับ query ทั่วไป (ตารางไม่เกิน `geqo_threshold` = 12) dynamic programming ให้ผลลัพธ์ที่ดีที่สุดในเวลาที่ยอมรับได้ แต่เมื่อตารางเยอะเกินไป (เช่น ORM ที่ generate join ซับซ้อน) ต้องสลับไปใช้ GEQO เพื่อไม่ให้ planning time พุ่งจนเกิน execution time เสียเอง — เป็นการ trade-off ระหว่างความ optimal กับความเร็วในการวางแผน

</details>

### แบบฝึกหัดที่ 9

จงอธิบายว่า multivariate MCV (จาก `CREATE STATISTICS ... (mcv)`) แก้ปัญหาอะไร และทำไมค่า `frequency` กับ `base_frequency` ใน `pg_mcv_list_items()` ถึงมีประโยชน์ในการวินิจฉัยปัญหา correlated columns

<details>
<summary>เฉลย</summary>

Multivariate MCV เก็บความถี่ของ**การรวมกันของค่าในหลายคอลัมน์**ที่พบบ่อย (เช่น `{status='cancelled', shipping_region='remote-area'}`) แทนที่จะเก็บ MCV แยกทีละคอลัมน์แล้วให้ planner คูณ selectivity เข้าด้วยกันภายใต้สมมติฐาน independence ซึ่งผิดพลาดเมื่อคอลัมน์สัมพันธ์กันจริง

`base_frequency` คือค่าที่**จะเป็น**ถ้าสมมติว่าคอลัมน์ independent จากกัน (คำนวณจาก selectivity เดี่ยวคูณกัน) ส่วน `frequency` คือค่าที่**วัดได้จริง**จากข้อมูล ถ้าทั้งสองค่าใกล้เคียงกันแปลว่าคอลัมน์นั้น independent จริงๆ (ไม่จำเป็นต้องมี extended statistics) แต่ถ้าต่างกันมาก (เช่น `frequency` สูงกว่า `base_frequency` หลายเท่า) แปลว่ามี correlation ที่ทำให้ planner ประมาณผิดถ้าไม่มี extended statistics ช่วย — ตัวเลขนี้จึงเป็นเครื่องมือวินิจฉัยที่ตรงไปตรงมาว่าคอลัมน์คู่ไหนควรสร้าง extended statistics

</details>

### แบบฝึกหัดที่ 10

จงเขียนขั้นตอนแบบครบวงจร (SQL commands) เพื่อวินิจฉัยว่า query หนึ่งมีปัญหา row estimation ผิดพลาดจาก correlated columns หรือไม่ และถ้าใช่ ให้แก้ไขด้วย extended statistics พร้อมยืนยันผลลัพธ์หลังแก้ไข

<details>
<summary>เฉลย</summary>

```sql
-- ขั้นที่ 1: รัน EXPLAIN ANALYZE เพื่อเทียบ estimated rows กับ actual rows
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders
WHERE status = 'cancelled' AND shipping_region = 'remote-area';

-- ถ้า "rows=" (estimate) กับ "rows=" ใน actual time ต่างกันมาก (เช่นมากกว่า 5-10 เท่า)
-- ให้สงสัยว่าเป็นปัญหา correlated columns

-- ขั้นที่ 2: ตรวจสอบว่าคอลัมน์ทั้งสองมีความสัมพันธ์กันจริงหรือไม่ด้วยการเทียบสัดส่วน
SELECT
    status,
    shipping_region,
    count(*) AS actual_count,
    round(count(*)::numeric / sum(count(*)) OVER (), 4) AS actual_fraction
FROM orders
WHERE status = 'cancelled'
GROUP BY status, shipping_region
ORDER BY actual_count DESC;

-- ขั้นที่ 3: สร้าง extended statistics
CREATE STATISTICS stx_orders_status_region (dependencies, mcv, ndistinct)
    ON status, shipping_region
    FROM orders;

ANALYZE orders;

-- ขั้นที่ 4: รัน EXPLAIN ANALYZE ซ้ำเพื่อยืนยันว่า estimate ดีขึ้น
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders
WHERE status = 'cancelled' AND shipping_region = 'remote-area';

-- ขั้นที่ 5: ตรวจสอบตัวเลขดิบใน multivariate MCV เพื่อยืนยันขนาดของ correlation
SELECT m.values, m.frequency, m.base_frequency,
       round(m.frequency / NULLIF(m.base_frequency, 0), 2) AS correlation_ratio
FROM pg_statistic_ext_data d,
     pg_mcv_list_items(d.stxdmcv) m
JOIN pg_statistic_ext s ON s.oid = d.stxoid
WHERE s.stxname = 'stx_orders_status_region'
ORDER BY m.frequency DESC;
```

ผลลัพธ์ที่คาดหวัง: หลังสร้าง extended statistics, `rows=` estimate ใน `EXPLAIN` ควรใกล้เคียงกับ actual rows มากขึ้นอย่างมีนัยสำคัญ (จากที่คลาดเคลื่อนหลายเท่าตัว เหลือคลาดเคลื่อนไม่ถึง 5-10%) และ `correlation_ratio` ที่มากกว่า 1 อย่างชัดเจนสำหรับ combination บางอันจะยืนยันว่ามี correlation จริงที่ extended statistics ช่วยแก้ไข

</details>

---

**บทถัดไป**: [Part 087: Node.js Integration](./part-087-nodejs-integration.md)
