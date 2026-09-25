# Part 098: Benchmarking ด้วย pgbench และ Load Testing

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับโลก/ผู้เชี่ยวชาญ | Part 098

---

## เป้าหมายการเรียนรู้

เมื่อจบบทนี้ ผู้เรียนจะสามารถ:

1. อธิบายได้ว่าทำไมการ Benchmark คือขั้นตอนที่ขาดไม่ได้ก่อนและหลังการเปลี่ยนแปลงระบบฐานข้อมูล แทนที่จะ "เดา" ว่าเร็วขึ้นหรือช้าลง
2. ติดตั้งและใช้งาน `pgbench` ซึ่งเป็นเครื่องมือ benchmark มาตรฐานที่มาพร้อม PostgreSQL ได้อย่างคล่องแคล่ว
3. รัน benchmark เบื้องต้นด้วย `-c`, `-j`, `-T`, `-t` และอ่านผลลัพธ์ TPS (Transactions Per Second) และ latency ได้อย่างถูกต้อง
4. เขียน Custom Script ด้วย flag `-f` เพื่อจำลอง workload ของระบบจริง แทนที่จะใช้ default TPC-B benchmark
5. ใช้ pgbench variables (`\set`, `random()`, `random_zipfian()` ฯลฯ) เพื่อสร้างข้อมูลทดสอบที่ใกล้เคียงพฤติกรรมผู้ใช้จริง
6. ออกแบบการทดสอบ A/B เพื่อเปรียบเทียบผลลัพธ์ก่อน-หลังการเพิ่ม index หรือเปลี่ยน configuration อย่างมีหลักการทางสถิติ
7. ทำ Connection Scaling Test เพื่อหา sweet spot ของจำนวน connection หรือขนาด pool ที่เหมาะสมกับ hardware ที่มี
8. ออกแบบ Custom Script ที่จำลอง Read-heavy และ Write-heavy workload เพื่อเปรียบเทียบพฤติกรรมของระบบภายใต้ภาระงานที่ต่างกัน
9. รู้จักเครื่องมือ Load Testing ระดับ Application เช่น k6, JMeter, Locust และเข้าใจว่าเมื่อไรควรใช้เครื่องมือเหล่านี้แทนหรือควบคู่กับ pgbench
10. ออกแบบและรัน Benchmark Suite แบบเต็มรูปแบบสำหรับระบบ e-commerce ได้ พร้อมวิเคราะห์และสรุปผลลัพธ์อย่างเป็นระบบ

---

## เตรียมข้อมูล

บทนี้เราจะใช้ pgbench ทดสอบกับฐานข้อมูล PostgreSQL จริงบนเครื่องของผู้เรียนเอง ทุกคำสั่งใน bash code block รันได้จริงถ้ามี PostgreSQL ติดตั้งอยู่ (PostgreSQL 14 ขึ้นไปแนะนำ เพราะ syntax ของ pgbench variables บางตัว เช่น `random_zipfian()` มีใน PostgreSQL 12+)

### 1) ตรวจสอบว่า pgbench มีอยู่แล้วหรือไม่

`pgbench` เป็นส่วนหนึ่งของ PostgreSQL client tools (หรืออยู่ใน package `postgresql-contrib` ในบาง distro) ไม่ต้องติดตั้งเพิ่มถ้าติดตั้ง PostgreSQL แบบเต็มแล้ว

```bash
pgbench --version
```

ผลลัพธ์ตัวอย่าง:

```
pgbench (PostgreSQL) 16.4
```

ถ้าไม่มี ให้ติดตั้งเพิ่ม:

```bash
# Debian / Ubuntu
sudo apt-get install postgresql-contrib

# RHEL / Fedora
sudo dnf install postgresql-contrib

# macOS (Homebrew)
brew install postgresql@16
```

### 2) สร้างฐานข้อมูลสำหรับบทนี้

```bash
createdb ecommerce_bench
```

### 3) สร้าง Schema แบบ e-commerce

เราจะใช้ schema แบบง่ายที่ครอบคลุมตารางหลักของระบบ e-commerce ได้แก่ `categories`, `products`, `customers`, `orders`, `order_items` — schema นี้จะเป็นเป้าหมายของ Custom Script ที่เราจะเขียนใน Step 969 เป็นต้นไป

```sql
-- save เป็นไฟล์ schema.sql แล้วรันด้วย: psql -d ecommerce_bench -f schema.sql

CREATE TABLE categories (
    category_id   SERIAL PRIMARY KEY,
    category_name VARCHAR(100) NOT NULL,
    parent_id     INT REFERENCES categories(category_id)
);

CREATE TABLE products (
    product_id     SERIAL PRIMARY KEY,
    category_id    INT NOT NULL REFERENCES categories(category_id),
    product_name   VARCHAR(200) NOT NULL,
    price          NUMERIC(10,2) NOT NULL,
    stock_quantity INT NOT NULL DEFAULT 0,
    created_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE customers (
    customer_id SERIAL PRIMARY KEY,
    full_name   VARCHAR(150) NOT NULL,
    email       VARCHAR(150) UNIQUE NOT NULL,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE orders (
    order_id     BIGSERIAL PRIMARY KEY,
    customer_id  INT NOT NULL REFERENCES customers(customer_id),
    order_status VARCHAR(20) NOT NULL DEFAULT 'pending',
    total_amount NUMERIC(12,2) NOT NULL DEFAULT 0,
    created_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE order_items (
    order_item_id BIGSERIAL PRIMARY KEY,
    order_id      BIGINT NOT NULL REFERENCES orders(order_id),
    product_id    INT NOT NULL REFERENCES products(product_id),
    quantity      INT NOT NULL,
    unit_price    NUMERIC(10,2) NOT NULL
);
```

### 4) สร้างข้อมูลทดสอบด้วย `generate_series`

เราจะสร้างข้อมูลขนาดพอสมควรเพื่อให้ผล benchmark สมจริง (ไม่เล็กเกินจนทุกอย่าง fit ใน cache ทั้งหมด และไม่ใหญ่เกินจนรันช้าเกินไปสำหรับการฝึก)

```sql
INSERT INTO categories (category_name)
SELECT 'Category ' || g
FROM generate_series(1, 20) g;

INSERT INTO products (category_id, product_name, price, stock_quantity)
SELECT (random() * 19 + 1)::int,
       'Product ' || g,
       (random() * 2000 + 50)::numeric(10,2),
       (random() * 500)::int
FROM generate_series(1, 5000) g;

INSERT INTO customers (full_name, email)
SELECT 'Customer ' || g,
       'customer' || g || '@example.com'
FROM generate_series(1, 20000) g;

INSERT INTO orders (customer_id, order_status, total_amount, created_at)
SELECT (random() * 19999 + 1)::int,
       (ARRAY['pending','paid','shipped','completed','cancelled'])[ceil(random()*5)],
       (random() * 5000 + 100)::numeric(12,2),
       now() - (random() * 365 || ' days')::interval
FROM generate_series(1, 200000) g;

INSERT INTO order_items (order_id, product_id, quantity, unit_price)
SELECT (random() * 199999 + 1)::bigint,
       (random() * 4999 + 1)::int,
       (random() * 5 + 1)::int,
       (random() * 2000 + 50)::numeric(10,2)
FROM generate_series(1, 600000) g;

ANALYZE;
```

หลังรันเสร็จ เราจะมี customers 20,000 คน, products 5,000 รายการ, orders 200,000 รายการ และ order_items 600,000 รายการ — ขนาดนี้เพียงพอที่จะทำให้เห็นผลกระทบของ index หรือ configuration ได้ชัดเจนเมื่อทดสอบกับ pgbench

> **หมายเหตุ:** schema และข้อมูลชุดนี้ตั้งใจให้ "self-contained" อยู่ในบทนี้ทั้งหมด ไม่ต้องพึ่งพาไฟล์จากบทอื่น ผู้เรียนสามารถรันตามได้ทันทีตั้งแต่ต้นจนจบบท

---

## Step 966: ทำไมต้อง Benchmark

### วัดผลก่อน-หลัง แทนการเดา

ในบทที่ 074 (Step 739) เราเรียนรู้การใช้ `EXPLAIN ANALYZE` เพื่อดูว่า query planner เลือก execution plan แบบไหน และ query หนึ่งตัวใช้เวลาเท่าไร นั่นคือการวัดผลใน **ระดับ query เดียว** แต่ในการทำงานจริง วิศวกรฐานข้อมูลต้องตอบคำถามที่ใหญ่กว่านั้นบ่อยมาก เช่น:

- "ถ้าเพิ่ม index ตัวนี้ ระบบทั้งระบบจะเร็วขึ้นจริงไหม หรือจะช้าลงเพราะ write overhead?"
- "เซิร์ฟเวอร์นี้รองรับ concurrent user ได้กี่คนก่อนที่ latency จะเริ่มพุ่ง?"
- "เปลี่ยน `shared_buffers` จาก 2GB เป็น 8GB แล้วคุ้มค่าจริงหรือเปล่า?"
- "PostgreSQL 16 เร็วกว่า PostgreSQL 14 สำหรับ workload ของเราแค่ไหน?"

คำถามเหล่านี้ตอบด้วย `EXPLAIN ANALYZE` เพียงอย่างเดียวไม่ได้ เพราะมันวัดแค่ query เดียวแบบ isolated ไม่ได้จำลองสภาพที่มี concurrent client หลายสิบหลายร้อยตัวยิง query พร้อมกัน แย่ง lock กัน แย่ง I/O กัน แย่ง CPU กัน — นี่คือหน้าที่ของ **benchmark**

### หลักการ: วัด Baseline ก่อนเสมอ

กฎเหล็กของการทำ performance engineering คือ:

> **ห้ามเปลี่ยนอะไรโดยไม่มี baseline ให้เทียบ**

ถ้าไม่มี baseline การเปลี่ยนแปลงใด ๆ ก็ตาม (เพิ่ม index, เปลี่ยน config, upgrade version, เปลี่ยน hardware) จะเป็นการ "เดา" ว่าดีขึ้นหรือแย่ลง ทั้งที่บางครั้งการเปลี่ยนแปลงที่ดูสมเหตุสมผลกลับทำให้ระบบช้าลงก็มี เช่น:

- เพิ่ม index มากเกินไป → INSERT/UPDATE ช้าลงเพราะต้องอัปเดตหลาย index
- เพิ่ม `work_mem` มากเกินไปในระบบที่มี concurrent connection สูง → memory pressure จนเกิด swap
- เปลี่ยน `max_connections` สูงขึ้นโดยไม่ใช้ connection pooling → context switching overhead จนช้าลง

### กระบวนการ Benchmark ที่ถูกต้อง

```
1. กำหนดคำถามที่ต้องการตอบ (Hypothesis)
   เช่น "การเพิ่ม index บน orders.customer_id จะทำให้ TPS ของ order history query สูงขึ้น"

2. วัด Baseline (สถานะปัจจุบัน) หลายรอบ
   รันซ้ำอย่างน้อย 3 รอบ เพื่อดูความแปรปรวน (variance)

3. ทำการเปลี่ยนแปลงเพียง "หนึ่งอย่าง" ต่อรอบทดสอบ
   (เปลี่ยนหลายอย่างพร้อมกัน จะไม่รู้ว่าอะไรคือสาเหตุที่แท้จริง)

4. วัดผลหลังเปลี่ยนแปลง ด้วยเงื่อนไขเดียวกันทุกประการ
   (จำนวน client เท่าเดิม, duration เท่าเดิม, ข้อมูลขนาดเท่าเดิม)

5. เปรียบเทียบผลลัพธ์ด้วยตัวเลข ไม่ใช่ความรู้สึก
   TPS สูงขึ้นกี่ %, latency p99 ลดลงกี่ ms

6. สรุปผลและตัดสินใจ
```

### ข้อควรระวังที่พบบ่อย (Benchmark Pitfalls)

| ปัญหา | ผลกระทบ | วิธีแก้ |
|---|---|---|
| **Cache ยังไม่ warm** | รอบแรกช้ากว่ารอบถัดไปมาก เพราะข้อมูลยังไม่อยู่ใน shared_buffers / OS page cache | รันรอบ warm-up ทิ้งก่อน แล้วค่อยเก็บผลจากรอบถัดไป |
| **มี process อื่นแย่ง resource** | ผลลัพธ์แกว่งเพราะ CPU/disk ถูกใช้งานจากงานอื่น | รัน benchmark บนเครื่องที่ไม่มี workload อื่นรบกวน |
| **รันแค่ 1 ครั้ง** | ไม่รู้ว่าความแตกต่างที่เห็นเป็นของจริงหรือ noise | รันอย่างน้อย 3-5 รอบ ดู median และ standard deviation |
| **Benchmark client กับ database อยู่เครื่องเดียวกัน** | client แย่ง CPU/network กับ database เอง ผลลัพธ์ไม่สะท้อน production | แยกเครื่อง client และ server ถ้าเป็นไปได้ หรืออย่างน้อยต้องรู้ข้อจำกัดนี้ |
| **ข้อมูลทดสอบเล็กเกินไป** | ทุกอย่าง fit ใน cache หมด ไม่เจอปัญหา I/O ที่จะเจอจริงใน production | ใช้ scale factor หรือขนาดข้อมูลที่ใกล้เคียง production |
| **เปลี่ยนหลายตัวแปรพร้อมกัน** | ไม่รู้ว่าอะไรคือสาเหตุของความเปลี่ยนแปลง | เปลี่ยนทีละตัวแปรเท่านั้น |

บทนี้ทั้งบทจะพาไปรู้จักเครื่องมือที่ใช้ทำตามกระบวนการข้างต้นอย่างเป็นระบบ นั่นคือ **pgbench**

---

## Step 967: pgbench พื้นฐาน

### pgbench คืออะไร

`pgbench` เป็นเครื่องมือ benchmark แบบง่ายที่ **มาพร้อมกับ PostgreSQL เอง** (เป็นส่วนหนึ่งของ client applications) ออกแบบมาเพื่อจำลอง workload แบบ TPC-B (Transaction Processing Performance Council Benchmark B) ซึ่งเป็นมาตรฐานอุตสาหกรรมสำหรับวัดประสิทธิภาพระบบ OLTP (Online Transaction Processing)

จุดเด่นของ pgbench:

- ติดตั้งมาพร้อม PostgreSQL แล้ว ไม่ต้องหาเครื่องมือเสริม
- รองรับทั้ง benchmark สำเร็จรูป (built-in TPC-B-like) และ Custom Script ของตัวเอง
- วัดได้ทั้ง TPS, latency, และรายงานแบบละเอียดต่อ statement
- รองรับการจำลอง concurrent client จำนวนมาก
- เข้ากันได้กับทุกแพลตฟอร์มที่ PostgreSQL รองรับ

### pgbench -i : การ Initialize

ก่อนรัน benchmark แบบ built-in เราต้องสร้างตารางมาตรฐานของ pgbench ก่อนด้วย flag `-i` (initialize):

```bash
pgbench -i -s 10 ecommerce_bench
```

ผลลัพธ์ตัวอย่าง:

```
dropping old tables...
NOTICE:  table "pgbench_accounts" does not exist, skipping
NOTICE:  table "pgbench_branches" does not exist, skipping
NOTICE:  table "pgbench_history" does not exist, skipping
NOTICE:  table "pgbench_tellers" does not exist, skipping
creating tables...
generating data (client-side)...
1000000 of 1000000 tuples (100%) done (elapsed 1.42 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 2.81 s (drop tables 0.01 s, create tables 0.02 s, client-side generate 1.53 s, vacuum 0.38 s, primary keys 0.87 s).
```

คำสั่งนี้จะสร้างตารางมาตรฐาน 4 ตารางในฐานข้อมูล `ecommerce_bench`:

| ตาราง | ความหมาย | จำนวนแถวเมื่อ scale=N |
|---|---|---|
| `pgbench_branches` | สาขาธนาคาร (จำลอง) | 1 × N |
| `pgbench_tellers` | พนักงานเทลเลอร์ | 10 × N |
| `pgbench_accounts` | บัญชีลูกค้า | 100,000 × N |
| `pgbench_history` | ประวัติธุรกรรม (เริ่มต้นว่างเปล่า) | 0 (เติมระหว่างรัน benchmark) |

จาก `-s 10` ข้างต้น หมายความว่า scale factor = 10 ดังนั้น `pgbench_accounts` จะมี 1,000,000 แถว (100,000 × 10)

> **สำคัญ:** ตารางเหล่านี้เป็นตารางแยกต่างหากจาก schema e-commerce ที่เราสร้างไว้ก่อนหน้า มันจะถูกสร้างในฐานข้อมูลเดียวกัน (`ecommerce_bench`) แต่ไม่ปะปนกับตาราง `orders`, `products` ฯลฯ ของเรา — เราจะใช้ตาราง pgbench มาตรฐานนี้สำหรับ Step 967-968 เพื่อทำความเข้าใจพื้นฐาน จากนั้นตั้งแต่ Step 969 เป็นต้นไปจะย้ายไปใช้ Custom Script บน schema e-commerce ของเราเอง

### Options สำคัญของ `pgbench -i`

```bash
pgbench -i -s SCALE_FACTOR [OPTIONS] DBNAME
```

| Option | ความหมาย |
|---|---|
| `-s, --scale=NUM` | กำหนด scale factor (ค่าเริ่มต้น = 1) |
| `-F, --fillfactor=NUM` | กำหนด fillfactor ของตาราง (default 100) ลดค่านี้ถ้าต้องการทดสอบ HOT update |
| `-n, --no-vacuum` | ข้ามการ VACUUM หลังสร้างข้อมูล (เร็วขึ้นแต่สถิติอาจไม่แม่น) |
| `-q, --quiet` | ลดข้อความ output ระหว่าง initialize |
| `--unlogged-tables` | สร้างตารางแบบ UNLOGGED (เร็วกว่าแต่ไม่ crash-safe เหมาะกับการทดสอบเท่านั้น) |
| `--partitions=NUM` | แบ่ง `pgbench_accounts` เป็น partition กี่ส่วน (ใช้ทดสอบ partitioning) |
| `--tablespace=NAME` | ระบุ tablespace ที่จะสร้างตาราง |

ตัวอย่างการเลือก scale factor ตามขนาดข้อมูลที่ต้องการ:

```bash
# scale เล็ก สำหรับทดสอบเร็ว ๆ บนโน้ตบุ๊ก
pgbench -i -s 1 ecommerce_bench     # accounts = 100,000 แถว (~15 MB)

# scale กลาง ใกล้เคียงระบบขนาดเล็ก-กลาง
pgbench -i -s 50 ecommerce_bench    # accounts = 5,000,000 แถว (~750 MB)

# scale ใหญ่ จำลองระบบ production
pgbench -i -s 500 ecommerce_bench   # accounts = 50,000,000 แถว (~7.5 GB)
```

กฎง่าย ๆ ในการเลือก scale factor: **ควรเลือกให้ขนาดข้อมูลรวมใหญ่กว่า `shared_buffers` ของเซิร์ฟเวอร์** เพื่อบังคับให้เกิด I/O จริง ไม่ใช่ทุกอย่างอยู่ใน memory หมด ซึ่งจะทำให้ผลลัพธ์ benchmark ดูดีเกินจริงเมื่อเทียบกับ production

---

## Step 968: การรัน pgbench เบื้องต้น

### รัน Benchmark แบบ Default (TPC-B-like)

หลัง initialize ตารางแล้ว รัน benchmark ได้ทันทีด้วยคำสั่งพื้นฐาน:

```bash
pgbench -c 10 -j 2 -T 60 ecommerce_bench
```

ความหมายของแต่ละ flag:

| Flag | ความหมาย |
|---|---|
| `-c, --client=NUM` | จำนวน concurrent client (connection) ที่จะจำลอง — ค่าเริ่มต้นคือ 1 |
| `-j, --jobs=NUM` | จำนวน worker thread ของตัว pgbench เอง (ไม่ใช่ของ PostgreSQL) — ควรตั้งใกล้เคียงจำนวน CPU core ของเครื่องที่รัน pgbench |
| `-T, --time=NUM` | ระยะเวลาที่จะรัน (วินาที) — ใช้แทน `-t` เมื่อต้องการควบคุมด้วยเวลา |
| `-t, --transactions=NUM` | จำนวน transaction ที่แต่ละ client ต้องรันให้ครบ (mutually exclusive กับ `-T`) |

ผลลัพธ์ตัวอย่าง:

```
pgbench (16.4)
starting vacuum...end.
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 10
query mode: simple
number of clients: 10
number of threads: 2
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 458213
number of failed transactions: 0 (0.000%)
latency average = 1.310 ms
latency stddev = 0.842 ms
initial connection time = 15.234 ms
tps = 7635.221577 (without initial connection time)
```

### วิธีอ่านผลลัพธ์

- **`number of transactions actually processed`** — จำนวน transaction ทั้งหมดที่รันสำเร็จตลอด 60 วินาที
- **`tps` (Transactions Per Second)** — ตัวเลขที่สำคัญที่สุด คือปริมาณงานที่ระบบรับได้ต่อวินาที ยิ่งสูงยิ่งดี คำนวณจาก `จำนวน transaction / duration`
- **`latency average`** — เวลาเฉลี่ยที่แต่ละ transaction ใช้ตั้งแต่เริ่มจนจบ (ms) ยิ่งต่ำยิ่งดี
- **`latency stddev`** — ส่วนเบี่ยงเบนมาตรฐานของ latency สะท้อนความสม่ำเสมอ ถ้าค่าสูงมากเทียบกับ average แปลว่า latency แกว่งเยอะ (บาง transaction เร็ว บาง transaction ช้ามาก)
- **`initial connection time`** — เวลาที่ใช้สร้าง connection ทั้งหมดตอนเริ่มต้น (แยกออกจาก tps เพื่อไม่ให้ปนกับเวลาทำงานจริง)

### ใช้ `-t` แทน `-T` เมื่อต้องการนับ Transaction ที่แน่นอน

```bash
pgbench -c 10 -j 2 -t 1000 ecommerce_bench
```

คำสั่งนี้แต่ละ client จะรัน 1,000 transaction แล้วหยุด (รวม 10,000 transaction ทั้งหมด) เหมาะเมื่อต้องการเปรียบเทียบ "งานปริมาณเท่ากัน" ระหว่างสองรอบทดสอบ ในขณะที่ `-T` เหมาะเมื่อต้องการเปรียบเทียบ "ในเวลาที่เท่ากัน"

### ดูรายงานความคืบหน้าระหว่างรันด้วย `-P`

```bash
pgbench -c 20 -j 4 -T 120 -P 10 ecommerce_bench
```

ผลลัพธ์ตัวอย่าง (แสดงทุก 10 วินาทีตามที่กำหนด):

```
progress: 10.0 s, 7820.3 tps, lat 2.548 ms stddev 1.203, 0 failed
progress: 20.0 s, 7791.6 tps, lat 2.562 ms stddev 1.245, 0 failed
progress: 30.0 s, 7688.4 tps, lat 2.598 ms stddev 1.312, 0 failed
progress: 40.0 s, 7702.1 tps, lat 2.590 ms stddev 1.289, 0 failed
...
```

มีประโยชน์มากเมื่อรัน benchmark นาน ๆ เพราะเห็นได้ทันทีว่า performance เริ่มตกลงระหว่างทางหรือไม่ (เช่น เกิดจาก checkpoint, autovacuum, หรือ bloat)

### รายงานละเอียดต่อ Statement ด้วย `-r`

```bash
pgbench -c 10 -j 2 -T 30 -r ecommerce_bench
```

ผลลัพธ์ตัวอย่าง (ส่วนท้ายจะเพิ่มรายละเอียด latency ของแต่ละคำสั่งใน script):

```
...
tps = 7532.884213 (without initial connection time)
statement latencies in milliseconds and failures:
         0.006           0  \set aid random(1, 100000 * :scale)
         0.003           0  \set bid random(1, 1 * :scale)
         0.003           0  \set tid random(1, 10 * :scale)
         0.003           0  \set delta random(-5000, 5000)
         0.021           0  BEGIN;
         0.412           0  UPDATE pgbench_accounts SET abalance = abalance + :delta WHERE aid = :aid;
         0.198           0  SELECT abalance FROM pgbench_accounts WHERE aid = :aid;
         0.089           0  UPDATE pgbench_tellers SET tbalance = tbalance + :delta WHERE tid = :tid;
         0.056           0  UPDATE pgbench_branches SET bbalance = bbalance + :delta WHERE bid = :bid;
         0.145           0  INSERT INTO pgbench_history (tid, bid, aid, delta, mtime) VALUES (:tid, :bid, :aid, :delta, CURRENT_TIMESTAMP);
         0.393           0  END;
```

จากรายงานนี้เห็นได้ทันทีว่า `UPDATE pgbench_accounts` เป็นคำสั่งที่ใช้เวลานานที่สุด (0.412 ms) เพราะตารางนี้ใหญ่ที่สุดและมี contention สูงสุด (มี client หลายตัวแย่ง row เดียวกันได้ เพราะ `aid` สุ่มในช่วงกว้าง)

### โหมด Read-only ด้วย `-S`

ถ้าต้องการทดสอบเฉพาะ SELECT อย่างเดียว (ไม่มี UPDATE/INSERT) ใช้:

```bash
pgbench -c 20 -j 4 -T 60 -S ecommerce_bench
```

ผลลัพธ์ตัวอย่าง:

```
transaction type: <builtin: select only>
scaling factor: 10
query mode: simple
number of clients: 20
number of threads: 4
duration: 60 s
number of transactions actually processed: 1842560
number of failed transactions: 0 (0.000%)
latency average = 0.651 ms
latency stddev = 0.298 ms
initial connection time = 18.442 ms
tps = 30707.812455 (without initial connection time)
```

สังเกตว่า TPS ของ read-only (30,707) สูงกว่า TPS ของ default TPC-B (7,635) มาก เพราะไม่มี lock contention จาก UPDATE และไม่มี WAL write overhead — นี่คือตัวอย่างที่ชัดเจนว่า **workload แบบไหนก็ให้ผลลัพธ์ที่ต่างกันมาก** ซึ่งนำไปสู่ความสำคัญของ Custom Script ใน Step ถัดไป

### สรุป Flag พื้นฐานที่ใช้บ่อย

```bash
pgbench \
  -c 10 \      # 10 concurrent clients
  -j 4 \       # 4 pgbench worker threads
  -T 60 \      # รัน 60 วินาที
  -P 10 \      # progress report ทุก 10 วินาที
  -r \         # รายงาน latency แยกตาม statement
  ecommerce_bench
```

---

## Step 969: Custom Script ด้วย `-f`

### ทำไมต้อง Custom Script

Default TPC-B-like benchmark ของ pgbench จำลองงานของ "ธนาคาร" (โอนเงินระหว่างบัญชี) ซึ่งไม่ตรงกับ workload จริงของระบบ e-commerce ที่มีการค้นหาสินค้า ดูประวัติคำสั่งซื้อ เพิ่มสินค้าลงตะกร้า และ checkout ดังนั้นในการทำงานจริง เราแทบจะต้อง **เขียน Custom Script** เสมอ เพื่อให้ benchmark สะท้อนพฤติกรรมจริงของระบบ

### Syntax พื้นฐานของ pgbench Script

ไฟล์ script ของ pgbench คือไฟล์ SQL ธรรมดา ที่อาจมีคำสั่งพิเศษของ pgbench (ขึ้นต้นด้วย `\`) แทรกอยู่ด้วย เช่น `\set` สำหรับกำหนดตัวแปร

### ตัวอย่างที่ 1: Custom Script สำหรับ "ดูรายละเอียดสินค้า"

```sql
-- product_lookup.sql
-- จำลองพฤติกรรมลูกค้าเปิดดูหน้ารายละเอียดสินค้าหนึ่งชิ้น

\set product_id random(1, 5000)

SELECT p.product_id, p.product_name, p.price, p.stock_quantity, c.category_name
FROM products p
JOIN categories c ON c.category_id = p.category_id
WHERE p.product_id = :product_id;
```

รันด้วยคำสั่ง:

```bash
pgbench -f product_lookup.sql -c 20 -j 4 -T 30 ecommerce_bench
```

ผลลัพธ์ตัวอย่าง:

```
pgbench (16.4)
transaction type: product_lookup.sql
scaling factor: 1
query mode: simple
number of clients: 20
number of threads: 4
duration: 30 s
number of transactions actually processed: 612480
number of failed transactions: 0 (0.000%)
latency average = 0.979 ms
latency stddev = 0.421 ms
initial connection time = 16.108 ms
tps = 20415.834122 (without initial connection time)
```

### ตัวอย่างที่ 2: Custom Script สำหรับ "บันทึกคำสั่งซื้อใหม่" (Checkout Flow)

การ checkout จริงเป็น transaction ที่มีหลาย statement ต่อเนื่องกัน (สร้าง order → เพิ่ม order_items → อัปเดต stock) ห่อด้วย `BEGIN`/`COMMIT` เพื่อให้เป็น atomic transaction:

```sql
-- checkout.sql
-- จำลอง transaction การสั่งซื้อสินค้า 1 รายการ

\set customer_id random(1, 20000)
\set product_id random(1, 5000)
\set quantity random(1, 3)

BEGIN;

INSERT INTO orders (customer_id, order_status, total_amount)
VALUES (:customer_id, 'pending', 0)
RETURNING order_id \gset

INSERT INTO order_items (order_id, product_id, quantity, unit_price)
SELECT :order_id, :product_id, :quantity, price
FROM products
WHERE product_id = :product_id;

UPDATE orders
SET total_amount = (
    SELECT SUM(quantity * unit_price)
    FROM order_items
    WHERE order_id = :order_id
)
WHERE order_id = :order_id;

UPDATE products
SET stock_quantity = stock_quantity - :quantity
WHERE product_id = :product_id;

COMMIT;
```

> **หมายเหตุ:** `\gset` เป็นคำสั่งพิเศษของ pgbench (และ psql) ที่ดึงค่าจากผลลัพธ์ query ล่าสุดมาเก็บไว้ในตัวแปร (ในที่นี้คือ `order_id`) เพื่อใช้ต่อใน statement ถัดไปภายใน transaction เดียวกัน — เป็นเทคนิคสำคัญสำหรับจำลอง flow ที่ต้องใช้ ID ที่เพิ่ง insert ไป

รันด้วยคำสั่ง:

```bash
pgbench -f checkout.sql -c 15 -j 4 -T 60 ecommerce_bench
```

ผลลัพธ์ตัวอย่าง:

```
pgbench (16.4)
transaction type: checkout.sql
scaling factor: 1
query mode: simple
number of clients: 15
number of threads: 4
duration: 60 s
number of transactions actually processed: 48360
number of failed transactions: 0 (0.000%)
latency average = 18.601 ms
latency stddev = 6.734 ms
initial connection time = 14.877 ms
tps = 806.078445 (without initial connection time)
```

สังเกตว่า TPS ของ checkout flow (806) ต่ำกว่า product lookup (20,415) มาก เพราะ checkout มีหลาย statement ต่อ transaction, มี write lock, และมี foreign key check — นี่คือความแตกต่างที่ benchmark เผยให้เห็นได้ ซึ่งการดู query เดียว ๆ ด้วย `EXPLAIN ANALYZE` จะไม่เห็นภาพรวมนี้

### การรันหลาย Script พร้อมกันด้วย Weight

pgbench รองรับการรันหลาย script พร้อมกันในสัดส่วนที่กำหนดได้ด้วยการใส่ `-f` หลายครั้ง และกำหนด weight ด้วย `@`:

```bash
pgbench -f product_lookup.sql@7 -f checkout.sql@1 -c 20 -j 4 -T 60 ecommerce_bench
```

ความหมายคือ ทุก ๆ 8 transaction จะเป็น `product_lookup.sql` ประมาณ 7 ครั้ง และ `checkout.sql` ประมาณ 1 ครั้ง (สัดส่วน 7:1) จำลองสถานการณ์จริงที่ลูกค้าดูสินค้าหลายครั้งกว่าจะตัดสินใจซื้อจริง — แนวคิดนี้เราจะขยายผลต่อใน Step 973

---

## Step 970: pgbench Variables

### `\set` และการกำหนดค่าตัวแปร

คำสั่ง `\set` ใช้กำหนดตัวแปรที่จะใช้ใน SQL statement ถัดไปด้วย syntax `:variable_name` ตัวแปรรองรับ expression ทางคณิตศาสตร์และฟังก์ชันสุ่มหลายแบบ

```sql
\set x 10
\set y random(1, 100)
\set z :x + :y
```

### ฟังก์ชันสุ่มที่ pgbench มีให้

| ฟังก์ชัน | ความหมาย |
|---|---|
| `random(min, max)` | สุ่มค่า uniform (กระจายเท่ากันทุกค่า) ระหว่าง min และ max |
| `random_exponential(min, max, parameter)` | สุ่มแบบ exponential decay — ค่าใกล้ min มีโอกาสออกมากกว่า |
| `random_gaussian(min, max, parameter)` | สุ่มแบบ normal distribution (โค้งระฆังคว่ำ) |
| `random_zipfian(min, max, parameter)` | สุ่มแบบ Zipfian — เหมาะจำลอง "สินค้ายอดฮิต" ที่บางรายการถูกเลือกบ่อยกว่ามาก |

### ทำไม Uniform Random ไม่สมจริง

ถ้าใช้ `random(1, 5000)` สุ่มเลือก `product_id` แบบ uniform หมายความว่าสินค้าทุกชิ้นมีโอกาสถูกค้นหาเท่ากันหมด แต่ในโลกจริง สินค้ายอดนิยม (เช่น "10 อันดับสินค้าขายดี") จะถูกค้นหาบ่อยกว่าสินค้าทั่วไปหลายสิบเท่า — พฤติกรรมแบบนี้เรียกว่า **Zipfian distribution** หรือ "80/20 rule"

### ตัวอย่าง: จำลองสินค้ายอดฮิตด้วย `random_zipfian`

```sql
-- hot_product_lookup.sql
-- จำลองการค้นหาสินค้าที่มีสินค้ายอดนิยมถูกเลือกบ่อยกว่าสินค้าทั่วไป

\set product_id random_zipfian(1, 5000, 1.2)

SELECT product_id, product_name, price, stock_quantity
FROM products
WHERE product_id = :product_id;
```

ค่า parameter `1.2` ในที่นี้คือความ "เอียง" ของการกระจาย (skew) ยิ่งค่าสูง ยิ่งกระจุกตัวที่ค่าน้อย ๆ (คือ `product_id` เลขต่ำ ๆ) มากขึ้น ค่ามาตรฐานที่นิยมใช้อยู่ระหว่าง 1.0-2.0

รันเปรียบเทียบกับแบบ uniform:

```bash
# Uniform random - ทุกสินค้าถูกเลือกเท่ากัน
pgbench -f product_lookup.sql -c 20 -j 4 -T 30 ecommerce_bench

# Zipfian - สินค้ายอดฮิตถูกเลือกบ่อยกว่า (สมจริงกว่า)
pgbench -f hot_product_lookup.sql -c 20 -j 4 -T 30 ecommerce_bench
```

ผลลัพธ์ตัวอย่างของ Zipfian:

```
transaction type: hot_product_lookup.sql
scaling factor: 1
query mode: simple
number of clients: 20
number of threads: 4
duration: 30 s
number of transactions actually processed: 698420
number of failed transactions: 0 (0.000%)
latency average = 0.858 ms
latency stddev = 0.312 ms
initial connection time = 15.902 ms
tps = 23280.923410 (without initial connection time)
```

สังเกตว่า TPS ของ Zipfian (23,280) สูงกว่า uniform เล็กน้อย (20,415 จาก Step 969) เพราะ row ที่ถูกเลือกซ้ำ ๆ บ่อย ๆ จะอยู่ใน buffer cache เสมอ (cache hit ratio สูงขึ้น) — นี่คือตัวอย่างที่ชัดเจนว่า **การเลือกรูปแบบการสุ่มข้อมูลมีผลต่อผลลัพธ์ benchmark อย่างมาก** และถ้าอยากให้ผลลัพธ์ใกล้เคียง production จริง ต้องเลือกฟังก์ชันสุ่มที่ตรงกับพฤติกรรมจริง

### ตัวแปรมาตรฐานที่ pgbench เตรียมไว้ให้

pgbench มีตัวแปรที่กำหนดค่าอัตโนมัติให้ใช้ได้เลยโดยไม่ต้อง `\set` เอง:

| ตัวแปร | ความหมาย |
|---|---|
| `:scale` | scale factor ที่ใช้ตอน `-i` (หรือกำหนดผ่าน `-s` ตอนรัน) |
| `:client_id` | หมายเลข client ปัจจุบัน (0 ถึง จำนวน client-1) |
| `:random_seed` | ค่า seed ที่ใช้สำหรับการสุ่ม (ถ้ากำหนดผ่าน `--random-seed`) |

### กำหนดตัวแปรจาก command line ด้วย `-D`

นอกจาก `\set` ในไฟล์ script แล้ว ยังกำหนดตัวแปรจากภายนอกได้ด้วย `-D`:

```bash
pgbench -f product_lookup.sql -D max_product=5000 -c 20 -T 30 ecommerce_bench
```

แล้วในไฟล์ script อ้างอิงด้วย `:max_product` ได้ทันที ประโยชน์คือทำให้ script เดียวใช้ซ้ำได้กับหลายขนาดข้อมูล โดยไม่ต้องแก้ไฟล์

### ตรวจสอบผลลัพธ์ Distribution ด้วยตัวเอง (เพื่อความเข้าใจ)

ก่อนใช้ `random_zipfian` ใน benchmark จริง แนะนำให้ทดลองดูการกระจายตัวก่อนด้วยการรันแบบสั้น ๆ พร้อม log:

```bash
pgbench -f hot_product_lookup.sql -c 1 -t 10000 -l ecommerce_bench
```

flag `-l` จะเขียน log ของแต่ละ transaction ลงไฟล์ `pgbench_log.<pid>` ซึ่งสามารถนำไปวิเคราะห์การกระจายตัวของค่าที่สุ่มได้ด้วยเครื่องมืออื่น เช่น `awk` หรือ Python

---

## Step 971: การเปรียบเทียบผลลัพธ์ A/B

### หลักการของ A/B Testing บนฐานข้อมูล

การทดสอบ A/B ในบริบทฐานข้อมูล หมายถึงการรัน benchmark **แบบเดียวกันทุกประการ** สองครั้ง — ครั้งแรกคือ "ก่อนเปลี่ยน" (Baseline / Control) และครั้งที่สองคือ "หลังเปลี่ยน" (Treatment) โดยเปลี่ยนตัวแปรเพียงอย่างเดียว แล้วเทียบผลลัพธ์

เราจะเชื่อมโยงกับสิ่งที่เรียนไปแล้วใน Part 073 (การปรับแต่ง Configuration) และ Part 074 (Query Performance Tuning, Step 739) โดยใช้ pgbench เป็นเครื่องมือ "พิสูจน์" ว่าสิ่งที่ปรับแต่งไปนั้นได้ผลจริงในระดับ workload ไม่ใช่แค่ query เดียว

### ตัวอย่างที่ 1: A/B Test การเพิ่ม Index

**สถานการณ์:** เราสงสัยว่า query ค้นหาประวัติคำสั่งซื้อของลูกค้า (`orders WHERE customer_id = ...`) ช้าเพราะไม่มี index บน `customer_id`

**Custom Script สำหรับทดสอบ:**

```sql
-- order_history.sql
\set customer_id random(1, 20000)

SELECT order_id, order_status, total_amount, created_at
FROM orders
WHERE customer_id = :customer_id
ORDER BY created_at DESC
LIMIT 10;
```

**ขั้นตอนที่ 1 — วัด Baseline (ยังไม่มี index):**

```bash
# ตรวจสอบก่อนว่ายังไม่มี index บน customer_id
psql -d ecommerce_bench -c "\d orders" | grep -i index
```

```bash
pgbench -f order_history.sql -c 20 -j 4 -T 60 -P 15 ecommerce_bench
```

ผลลัพธ์ Baseline:

```
progress: 15.0 s, 891.2 tps, lat 22.398 ms stddev 8.112, 0 failed
progress: 30.0 s, 878.4 tps, lat 22.712 ms stddev 8.501, 0 failed
progress: 45.0 s, 883.7 tps, lat 22.564 ms stddev 8.244, 0 failed
progress: 60.0 s, 876.9 tps, lat 22.850 ms stddev 8.390, 0 failed
transaction type: order_history.sql
scaling factor: 1
number of clients: 20
duration: 60 s
number of transactions actually processed: 52968
latency average = 22.632 ms
latency stddev = 8.312 ms
tps = 882.598234 (without initial connection time)
```

**ขั้นตอนที่ 2 — เพิ่ม Index:**

```sql
CREATE INDEX CONCURRENTLY idx_orders_customer_id ON orders (customer_id, created_at DESC);
ANALYZE orders;
```

**ขั้นตอนที่ 3 — วัดผลหลังเพิ่ม Index (เงื่อนไขเดียวกันทุกประการ):**

```bash
pgbench -f order_history.sql -c 20 -j 4 -T 60 -P 15 ecommerce_bench
```

ผลลัพธ์หลังเพิ่ม Index:

```
progress: 15.0 s, 18420.6 tps, lat 1.085 ms stddev 0.421, 0 failed
progress: 30.0 s, 18512.3 tps, lat 1.079 ms stddev 0.398, 0 failed
progress: 45.0 s, 18398.7 tps, lat 1.088 ms stddev 0.415, 0 failed
progress: 60.0 s, 18466.1 tps, lat 1.083 ms stddev 0.406, 0 failed
transaction type: order_history.sql
scaling factor: 1
number of clients: 20
duration: 60 s
number of transactions actually processed: 1108080
latency average = 1.084 ms
latency stddev = 0.410 ms
tps = 18449.412330 (without initial connection time)
```

**ตารางสรุปผล A/B:**

| Metric | Before (no index) | After (with index) | ผลต่าง |
|---|---|---|---|
| TPS | 882.6 | 18,449.4 | **เร็วขึ้น 20.9 เท่า** |
| Latency avg | 22.632 ms | 1.084 ms | **ลดลง 95.2%** |
| Latency stddev | 8.312 ms | 0.410 ms | เสถียรขึ้นมาก |

ผลลัพธ์นี้ชัดเจนกว่าการดู `EXPLAIN ANALYZE` เพียงครั้งเดียว เพราะเราเห็น **ผลกระทบภายใต้ concurrent load จริง** ซึ่งบางครั้งอาจต่างจากที่เห็นตอนรัน query เดี่ยว ๆ (เช่น ถ้า index ทำให้เกิด lock contention เพิ่มตอน write ก็จะเห็นผลลบใน benchmark แต่ไม่เห็นใน `EXPLAIN ANALYZE` ของ SELECT อย่างเดียว)

### ตัวอย่างที่ 2: A/B Test การเปลี่ยน Configuration

**สถานการณ์:** ทดสอบผลของการเพิ่ม `work_mem` จาก 4MB (default) เป็น 64MB สำหรับ query ที่มีการ sort/hash จำนวนมาก

```sql
-- top_selling_products.sql
-- query ที่ใช้ hash aggregate หนัก ๆ

SELECT p.product_id, p.product_name, SUM(oi.quantity) AS total_sold
FROM order_items oi
JOIN products p ON p.product_id = oi.product_id
GROUP BY p.product_id, p.product_name
ORDER BY total_sold DESC
LIMIT 20;
```

```bash
# Baseline: work_mem = 4MB (default)
psql -d ecommerce_bench -c "SHOW work_mem;"
pgbench -f top_selling_products.sql -c 10 -j 4 -T 60 ecommerce_bench
```

ผลลัพธ์ Baseline (work_mem = 4MB):

```
tps = 42.318560 (without initial connection time)
latency average = 236.312 ms
```

```bash
# เปลี่ยน work_mem แล้วทดสอบใหม่ (ปรับใน postgresql.conf แล้ว reload หรือ SET ระดับ session)
psql -d ecommerce_bench -c "ALTER SYSTEM SET work_mem = '64MB';"
psql -d ecommerce_bench -c "SELECT pg_reload_conf();"
pgbench -f top_selling_products.sql -c 10 -j 4 -T 60 ecommerce_bench
```

ผลลัพธ์หลังเปลี่ยน (work_mem = 64MB):

```
tps = 187.542890 (without initial connection time)
latency average = 53.322 ms
```

| Metric | work_mem=4MB | work_mem=64MB | ผลต่าง |
|---|---|---|---|
| TPS | 42.3 | 187.5 | เร็วขึ้น 4.4 เท่า |
| Latency avg | 236.3 ms | 53.3 ms | ลดลง 77.4% |

**ข้อควรระวัง:** ผลลัพธ์นี้มาจากการรันด้วย `-c 10` concurrent client เท่านั้น ถ้าเพิ่มจำนวน client ให้สูงขึ้นมาก ๆ การเพิ่ม `work_mem` แบบ per-connection แบบนี้อาจทำให้ memory รวมทั้งระบบพุ่งสูงจน swap ได้ — ซึ่งนำไปสู่ความจำเป็นของ Step 972 คือการทดสอบ scaling ที่หลายระดับ connection ไม่ใช่แค่ระดับเดียว

### หลักสถิติที่ควรทำ: รันซ้ำหลายรอบ

เพื่อความน่าเชื่อถือ ควรรัน benchmark เดียวกันอย่างน้อย 3-5 รอบ แล้วดู median (ไม่ใช่ average เพราะ average ไวต่อค่าผิดปกติ) ตัวอย่างสคริปต์ bash สำหรับรันซ้ำอัตโนมัติ:

```bash
#!/bin/bash
# run_multiple.sh — รัน pgbench ซ้ำ 5 รอบ แล้วเก็บ tps ของแต่ละรอบ

for i in 1 2 3 4 5; do
    echo "=== Run $i ==="
    pgbench -f order_history.sql -c 20 -j 4 -T 30 ecommerce_bench \
        | grep "tps ="
done
```

ผลลัพธ์ตัวอย่าง:

```
=== Run 1 ===
tps = 18449.412330 (without initial connection time)
=== Run 2 ===
tps = 18502.887210 (without initial connection time)
=== Run 3 ===
tps = 18021.334550 (without initial connection time)
=== Run 4 ===
tps = 18466.129980 (without initial connection time)
=== Run 5 ===
tps = 18398.765400 (without initial connection time)
```

ค่าทั้ง 5 รอบอยู่ในช่วง 18,021-18,502 (แกว่งประมาณ 2.6%) ถือว่าเสถียรและน่าเชื่อถือพอที่จะสรุปผลได้ ถ้าค่าแกว่งมากกว่า 15-20% ควรตรวจสอบว่ามี noise จากภายนอกรบกวนหรือไม่ (เช่น autovacuum, checkpoint, process อื่นในเครื่อง)

---

## Step 972: Connection Scaling Test

### หา Sweet Spot ของจำนวน Connection

คำถามที่พบบ่อยที่สุดในการ tuning ระบบ production คือ "ควรตั้ง `max_connections` หรือ pool size เท่าไร?" คำตอบไม่ใช่ "ยิ่งเยอะยิ่งดี" เพราะ PostgreSQL แต่ละ connection ใช้ process แยกต่างหาก (ไม่ใช่ thread) การมี connection มากเกินจำนวน CPU core หรือ I/O capacity ที่มีจริง จะทำให้เกิด **context switching overhead** และ **lock contention** จนประสิทธิภาพตกลง แทนที่จะดีขึ้น

### สคริปต์ทดสอบ Scaling อัตโนมัติ

```bash
#!/bin/bash
# connection_scaling_test.sh
# ทดสอบ TPS ที่ระดับ concurrent client ต่าง ๆ แล้วบันทึกผลเป็น CSV

DB=ecommerce_bench
SCRIPT=order_history.sql
DURATION=30
OUTPUT=scaling_results.csv

echo "clients,tps,latency_avg_ms,latency_stddev_ms" > $OUTPUT

for c in 1 5 10 20 40 80 120 160 200; do
    echo "Testing with $c clients..."
    result=$(pgbench -f $SCRIPT -c $c -j 4 -T $DURATION $DB 2>&1)

    tps=$(echo "$result" | grep "tps =" | awk '{print $3}')
    lat_avg=$(echo "$result" | grep "latency average" | awk '{print $4}')
    lat_stddev=$(echo "$result" | grep "latency stddev" | awk '{print $4}')

    echo "$c,$tps,$lat_avg,$lat_stddev" >> $OUTPUT

    # พักระหว่างรอบเพื่อให้ระบบกลับสู่สภาพปกติ (cool-down)
    sleep 5
done

echo "Done. Results saved to $OUTPUT"
cat $OUTPUT
```

รันด้วย:

```bash
chmod +x connection_scaling_test.sh
./connection_scaling_test.sh
```

ผลลัพธ์ตัวอย่าง (`scaling_results.csv`) บนเครื่องที่มี 8 CPU core:

```
clients,tps,latency_avg_ms,latency_stddev_ms
1,2140.203,0.467,0.089
5,9812.556,0.509,0.201
10,17024.887,0.587,0.298
20,18449.412,1.084,0.410
40,18902.556,2.113,0.932
80,18780.223,4.259,2.104
120,17650.998,6.798,4.556
160,15230.442,10.502,8.901
200,12890.117,15.512,13.204
```

### วิเคราะห์ผลลัพธ์

| จำนวน Client | สังเกต |
|---|---|
| 1-10 | TPS เพิ่มขึ้นเกือบเป็นเส้นตรงตามจำนวน client — ระบบยังมี capacity เหลือ |
| 20-40 | TPS เริ่มขึ้นช้าลง (diminishing returns) — เข้าใกล้ **sweet spot** |
| 40 | **จุดสูงสุด** ของ TPS (18,902) — เกิน 8 core ไปเล็กน้อยเพราะมี I/O wait ช่วยให้ scheduler สลับงานได้ |
| 80+ | TPS เริ่มลดลง ในขณะที่ latency เพิ่มขึ้นแบบก้าวกระโดด — เกิด **contention** ชัดเจน |
| 200 | TPS ตกลงต่ำกว่าตอน 10 client ทั้งที่มี "แรงงาน" มากกว่า 20 เท่า — เป็นสัญญาณของ **thrashing** |

**สรุป:** สำหรับเครื่องนี้ (8 core) sweet spot อยู่ที่ประมาณ 30-40 concurrent connection ไม่ใช่การเปิด connection ให้มากที่สุดเท่าที่จะทำได้ กฎง่าย ๆ ที่ใช้เป็นจุดเริ่มต้นได้คือ:

```
max_connections ที่มีประสิทธิภาพ ≈ (CPU core count × 2) ถึง (CPU core count × 4)
```

แต่ตัวเลขจริงต้องวัดด้วย benchmark เสมอ เพราะขึ้นกับลักษณะ workload (read-heavy vs write-heavy), ความเร็วของ storage (SSD vs NVMe), และประเภทของ query ด้วย

### เมื่อ Sweet Spot ต่ำกว่าจำนวน User จริงที่ต้องรองรับ — ใช้ Connection Pooling

ถ้าระบบ production ต้องรองรับผู้ใช้ 1,000 คนพร้อมกัน แต่ sweet spot ของฐานข้อมูลอยู่ที่ 40 connection คำตอบไม่ใช่เปิด connection ตรงถึงฐานข้อมูล 1,000 เส้น แต่คือการใช้ **connection pooler** เช่น PgBouncer เพื่อรวม request จำนวนมากให้ใช้ connection pool ขนาดเล็กที่มีประสิทธิภาพสูงสุดแทน (แนวคิดเรื่อง connection pooling ควรทบทวนจากบทที่เกี่ยวกับ PgBouncer ในหลักสูตรนี้)

การทดสอบด้วย pgbench ยังใช้ยืนยันประสิทธิภาพของ pooler ได้ด้วย โดยเปลี่ยนพอร์ตปลายทางจากพอร์ต PostgreSQL ตรง ไปเป็นพอร์ตของ PgBouncer:

```bash
# ทดสอบตรงไปที่ PostgreSQL (พอร์ต 5432)
pgbench -f order_history.sql -c 200 -j 8 -T 30 -h localhost -p 5432 ecommerce_bench

# ทดสอบผ่าน PgBouncer (พอร์ต 6432) ที่ตั้ง pool_size=40
pgbench -f order_history.sql -c 200 -j 8 -T 30 -h localhost -p 6432 ecommerce_bench
```

ผลลัพธ์ทั่วไปคือ การผ่าน pooler ที่ตั้งค่าเหมาะสม จะให้ TPS สูงกว่าและ latency ต่ำกว่าการยิงตรงเข้า PostgreSQL ด้วย connection จำนวนมากเกิน sweet spot — เพราะ pooler จะจัดคิวคำขอให้ connection จริงที่มีอยู่อย่างมีประสิทธิภาพแทนที่จะปล่อยให้ PostgreSQL ต้องจัดการ process จำนวนมากเกินไปเอง

### ทดสอบด้วยการเปิด Connection ใหม่ทุกครั้งด้วย `-C`

ปกติ pgbench จะเปิด connection ครั้งเดียวแล้วใช้ซ้ำตลอดการทดสอบ (คล้ายกับแอปพลิเคชันที่ใช้ connection pool) แต่ถ้าต้องการทดสอบ **cost ของการเปิด connection ใหม่ทุกครั้ง** (คล้ายแอปพลิเคชันที่ไม่มี pooling) ใช้ `-C`:

```bash
pgbench -f product_lookup.sql -c 20 -j 4 -T 30 -C ecommerce_bench
```

ผลลัพธ์ตัวอย่าง:

```
number of transactions actually processed: 8420
latency average = 71.234 ms
tps = 280.667123 (including reconnections times)
```

เทียบกับไม่มี `-C` ที่ TPS อยู่ที่ 20,415 — จะเห็นว่าการเปิด connection ใหม่ทุกครั้งทำให้ TPS ตกลงมากกว่า 70 เท่า! นี่คือหลักฐานเชิงตัวเลขว่าทำไม connection pooling ถึงสำคัญมากสำหรับแอปพลิเคชันที่มี transaction สั้น ๆ จำนวนมาก

---

## Step 973: Read-heavy เทียบกับ Write-heavy Workload

### ทำไมต้องแยกทดสอบ Read และ Write

ระบบ e-commerce จริงมีสัดส่วนการอ่าน (ดูสินค้า, ค้นหา, ดูประวัติคำสั่งซื้อ) มากกว่าการเขียน (สั่งซื้อ, อัปเดตสถานะ) อย่างชัดเจน โดยทั่วไปอัตราส่วนอยู่ที่ประมาณ **80% read / 20% write** หรือมากกว่านั้นในบางระบบ (90/10 หรือ 95/5) การ benchmark ด้วยสัดส่วนที่ใกล้เคียงจริงจะให้ผลลัพธ์ที่นำไปใช้ตัดสินใจได้แม่นยำกว่าการทดสอบ read อย่างเดียวหรือ write อย่างเดียว

### ออกแบบ Custom Script ชุด Read

```sql
-- read_product_search.sql
-- จำลองการค้นหาสินค้าตามหมวดหมู่และช่วงราคา

\set category_id random(1, 20)
\set min_price random(50, 1000)
\set max_price :min_price + 500

SELECT product_id, product_name, price, stock_quantity
FROM products
WHERE category_id = :category_id
  AND price BETWEEN :min_price AND :max_price
ORDER BY price
LIMIT 20;
```

```sql
-- read_order_history.sql
-- จำลองการดูประวัติคำสั่งซื้อ (ต้องมี index จาก Step 971 แล้ว)

\set customer_id random(1, 20000)

SELECT order_id, order_status, total_amount, created_at
FROM orders
WHERE customer_id = :customer_id
ORDER BY created_at DESC
LIMIT 10;
```

```sql
-- read_product_detail.sql
-- จำลองการเปิดดูสินค้าแบบสุ่มด้วย Zipfian (สินค้ายอดฮิต)

\set product_id random_zipfian(1, 5000, 1.2)

SELECT p.product_id, p.product_name, p.price, p.stock_quantity, c.category_name
FROM products p
JOIN categories c ON c.category_id = p.category_id
WHERE p.product_id = :product_id;
```

### ออกแบบ Custom Script ชุด Write

```sql
-- write_checkout.sql
-- จำลอง transaction สั่งซื้อ (เหมือน checkout.sql จาก Step 969)

\set customer_id random(1, 20000)
\set product_id random(1, 5000)
\set quantity random(1, 3)

BEGIN;

INSERT INTO orders (customer_id, order_status, total_amount)
VALUES (:customer_id, 'pending', 0)
RETURNING order_id \gset

INSERT INTO order_items (order_id, product_id, quantity, unit_price)
SELECT :order_id, :product_id, :quantity, price
FROM products
WHERE product_id = :product_id;

UPDATE orders
SET total_amount = (
    SELECT SUM(quantity * unit_price)
    FROM order_items
    WHERE order_id = :order_id
)
WHERE order_id = :order_id;

UPDATE products
SET stock_quantity = stock_quantity - :quantity
WHERE product_id = :product_id;

COMMIT;
```

```sql
-- write_update_status.sql
-- จำลองการอัปเดตสถานะคำสั่งซื้อ (เช่น จาก pending เป็น paid)

\set order_id random(1, 200000)

UPDATE orders
SET order_status = 'paid'
WHERE order_id = :order_id
  AND order_status = 'pending';
```

### รัน Workload แบบผสม 80/20 (Read-heavy)

ใช้ weight กับ `-f` หลายไฟล์เพื่อสร้างสัดส่วนตามต้องการ ในที่นี้เราต้องการ 80% read (แบ่งเป็น 3 ชนิด) และ 20% write (แบ่งเป็น 2 ชนิด):

```bash
pgbench \
  -f read_product_search.sql@30 \
  -f read_order_history.sql@25 \
  -f read_product_detail.sql@25 \
  -f write_checkout.sql@12 \
  -f write_update_status.sql@8 \
  -c 40 -j 8 -T 120 -P 20 \
  ecommerce_bench
```

สัดส่วน weight ข้างต้นรวมกันเป็น 100 (30+25+25+12+8) ตรงกับสัดส่วน read 80% (30+25+25) / write 20% (12+8) พอดี

ผลลัพธ์ตัวอย่าง:

```
progress: 20.0 s, 6820.4 tps, lat 5.867 ms stddev 4.201, 0 failed
progress: 40.0 s, 6902.1 tps, lat 5.792 ms stddev 4.089, 0 failed
progress: 60.0 s, 6845.7 tps, lat 5.842 ms stddev 4.155, 0 failed
progress: 80.0 s, 6798.3 tps, lat 5.891 ms stddev 4.223, 0 failed
progress: 100.0 s, 6811.9 tps, lat 5.869 ms stddev 4.190, 0 failed
progress: 120.0 s, 6788.2 tps, lat 5.902 ms stddev 4.244, 0 failed
transaction type: multiple scripts
scaling factor: 1
number of clients: 40
number of threads: 8
duration: 120 s
number of transactions actually processed: 818760
number of failed transactions: 0 (0.000%)
latency average = 5.857 ms
latency stddev = 4.184 ms
initial connection time = 42.556 ms
tps = 6827.883991 (without initial connection time)

SQL script 1: read_product_search.sql
 - weight: 30 (approx 30.0% of total)
 - 245628 transactions (30.0% of total, tps = 2048.365)
 - latency average = 3.402 ms
 - latency stddev = 1.203 ms

SQL script 2: read_order_history.sql
 - weight: 25 (approx 25.0% of total)
 - 204690 transactions (25.0% of total, tps = 1706.971)
 - latency average = 1.098 ms
 - latency stddev = 0.412 ms

SQL script 3: read_product_detail.sql
 - weight: 25 (approx 25.0% of total)
 - 204690 transactions (25.0% of total, tps = 1706.971)
 - latency average = 0.879 ms
 - latency stddev = 0.298 ms

SQL script 4: write_checkout.sql
 - weight: 12 (approx 12.0% of total)
 - 98251 transactions (12.0% of total, tps = 819.144)
 - latency average = 18.902 ms
 - latency stddev = 6.845 ms

SQL script 5: write_update_status.sql
 - weight: 8 (approx 8.0% of total)
 - 65501 transactions (8.0% of total, tps = 546.088)
 - latency average = 1.456 ms
 - latency stddev = 0.876 ms
```

pgbench แสดงผลแยกตาม script ให้เองอัตโนมัติเมื่อใช้หลาย `-f` — มีประโยชน์มากเพราะเห็นได้ทันทีว่า statement ไหนคือคอขวดของระบบ (ในที่นี้ `write_checkout.sql` มี latency สูงสุดที่ 18.902 ms เพราะมีหลาย statement และ write lock)

### เปรียบเทียบกับ Workload แบบ Write-heavy (50/50)

```bash
pgbench \
  -f read_product_search.sql@25 \
  -f read_order_history.sql@25 \
  -f write_checkout.sql@30 \
  -f write_update_status.sql@20 \
  -c 40 -j 8 -T 120 -P 20 \
  ecommerce_bench
```

ผลลัพธ์ตัวอย่างโดยรวม:

```
number of transactions actually processed: 512340
latency average = 9.362 ms
tps = 4269.478213 (without initial connection time)
```

### ตารางสรุปเปรียบเทียบ

| Workload | TPS รวม | Latency avg | สังเกต |
|---|---|---|---|
| 80% Read / 20% Write | 6,827.9 | 5.857 ms | ใกล้เคียงระบบ e-commerce ทั่วไปช่วงเวลาปกติ |
| 50% Read / 50% Write | 4,269.5 | 9.362 ms | จำลองช่วง Flash Sale ที่มีคำสั่งซื้อเข้ามาถี่ผิดปกติ |

การทดสอบสองแบบนี้ช่วยให้ทีมวางแผนได้ว่า ถ้าระบบต้องรองรับช่วง Flash Sale ที่สัดส่วน write เพิ่มขึ้น ระบบจะรับได้ที่ TPS เท่าไร และต้อง scale (แนวนอนหรือแนวตั้ง) เพิ่มเท่าไรเพื่อรองรับ

---

## Step 974: เครื่องมือ Load Testing อื่น ๆ

### ทำไม pgbench ไม่พอในบางสถานการณ์

pgbench ยอดเยี่ยมสำหรับทดสอบ **ระดับฐานข้อมูลโดยตรง** (raw SQL) แต่ในระบบจริง ผู้ใช้ไม่ได้ยิง SQL ตรงเข้า PostgreSQL — พวกเขาเรียก API ผ่าน HTTP ที่ผ่าน load balancer, application server, cache layer, business logic ก่อนจะถึงฐานข้อมูล ถ้าต้องการทดสอบ **ทั้งระบบ (end-to-end)** รวมถึงชั้น application ด้วย จำเป็นต้องใช้เครื่องมือ Load Testing ระดับ HTTP/Application แทนหรือควบคู่กับ pgbench

### k6

[k6](https://k6.io) เป็นเครื่องมือ load testing แบบ modern เขียน test script ด้วย JavaScript รันแบบ command line เหมาะสำหรับทีม DevOps/SRE ที่ต้องการรวม load test เข้ากับ CI/CD pipeline

```javascript
// checkout_flow_test.js
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  vus: 50,          // virtual users
  duration: '60s',
};

export default function () {
  // จำลองการดูสินค้า
  const productRes = http.get('https://api.example.com/products/1234');
  check(productRes, { 'status is 200': (r) => r.status === 200 });

  sleep(1);

  // จำลองการ checkout
  const checkoutRes = http.post(
    'https://api.example.com/orders',
    JSON.stringify({ product_id: 1234, quantity: 1 }),
    { headers: { 'Content-Type': 'application/json' } }
  );
  check(checkoutRes, { 'checkout succeeded': (r) => r.status === 201 });

  sleep(1);
}
```

```bash
k6 run checkout_flow_test.js
```

k6 วัด HTTP-level metrics เช่น request duration, error rate, throughput ของทั้ง API ไม่ใช่แค่ database query

### JMeter

[Apache JMeter](https://jmeter.apache.org) เป็นเครื่องมือ load testing ที่เก่าแก่และครบเครื่องที่สุดตัวหนึ่ง รองรับโปรโตคอลหลากหลาย (HTTP, JDBC, FTP, JMS, ฯลฯ) มี GUI สำหรับออกแบบ test plan แบบ visual เหมาะกับทีม QA ในองค์กรขนาดใหญ่ที่ต้องการ test plan ที่ซับซ้อนและรายงานผลแบบละเอียด

จุดเด่นของ JMeter คือรองรับการทดสอบ **JDBC โดยตรง** ได้ด้วย (คล้าย pgbench) ผ่าน "JDBC Request" sampler แต่ส่วนใหญ่มักใช้ทดสอบระดับ HTTP/API มากกว่า เพราะมี ecosystem ของ plugin กว้างขวาง

### Locust

[Locust](https://locust.io) เป็นเครื่องมือ load testing แบบ Python-based เขียน test scenario เป็นโค้ด Python ธรรมดา เหมาะกับทีมที่คุ้นเคยกับ Python และต้องการความยืดหยุ่นสูงในการเขียน logic ของ test scenario ที่ซับซ้อน

```python
# locustfile.py
from locust import HttpUser, task, between

class EcommerceUser(HttpUser):
    wait_time = between(1, 3)

    @task(7)
    def browse_product(self):
        self.client.get("/products/1234")

    @task(2)
    def search_products(self):
        self.client.get("/products/search?category=electronics")

    @task(1)
    def checkout(self):
        self.client.post("/orders", json={"product_id": 1234, "quantity": 1})
```

```bash
locust -f locustfile.py --host=https://api.example.com
```

Locust มี Web UI แบบ real-time ให้ดู request rate และ failure rate ขณะทดสอบ และรองรับการ scale แบบ distributed (รัน worker หลายเครื่องพร้อมกัน) ได้ง่าย

### ตารางเปรียบเทียบเครื่องมือ

| เครื่องมือ | ระดับที่ทดสอบ | ภาษาที่ใช้เขียน script | เหมาะกับ |
|---|---|---|---|
| **pgbench** | Database (SQL โดยตรง) | SQL + pgbench syntax | ทดสอบประสิทธิภาพ PostgreSQL, index, config, hardware โดยตัดชั้น application ออกไปทั้งหมด |
| **k6** | HTTP / API | JavaScript | ทดสอบ API/microservice, รวมเข้ากับ CI/CD ได้ง่าย |
| **JMeter** | HTTP, JDBC, และอื่น ๆ | GUI + XML (หรือ Groovy) | Enterprise QA, test plan ซับซ้อน, หลายโปรโตคอล |
| **Locust** | HTTP / API | Python | ทีมที่ถนัด Python, ต้องการ scenario แบบยืดหยุ่นสูง, distributed load test |

### เมื่อไรควรใช้อะไร

- **ใช้ pgbench** เมื่อต้องการแยกปัจจัยของฐานข้อมูลออกจากชั้นอื่น ๆ ทั้งหมด เพื่อตอบคำถามเฉพาะเจาะจงเกี่ยวกับ PostgreSQL เช่น "index นี้ช่วยไหม", "hardware นี้พอไหม", "config นี้ดีขึ้นไหม" — นี่คือสิ่งที่บทนี้เน้น
- **ใช้ k6/JMeter/Locust** เมื่อต้องการทดสอบว่าทั้งระบบ (frontend, API, cache, database) รับโหลดของผู้ใช้จริงได้ไหม ก่อนเปิดตัว feature ใหม่ หรือก่อนงาน Flash Sale ที่คาดว่าจะมี traffic พุ่งสูง

ในทางปฏิบัติ ทีมวิศวกรรมที่ดีมักใช้ **ทั้งสองระดับร่วมกัน**: ใช้ pgbench เพื่อ tune ฐานข้อมูลให้ดีที่สุดก่อน แล้วใช้ k6/Locust เพื่อยืนยันว่าเมื่อรวมทั้งระบบเข้าด้วยกันแล้ว ยังรองรับโหลดเป้าหมายได้จริง

---

## Step 975: แบบฝึกหัดรวม — ออกแบบและรัน Benchmark Suite แบบเต็มรูปแบบ

### โจทย์

สมมติเราเป็นวิศวกรฐานข้อมูลของบริษัท e-commerce ที่กำลังจะจัดแคมเปญ "Mega Sale" ในอีก 2 สัปดาห์ ทีมต้องการรู้ว่าระบบปัจจุบันรองรับโหลดสูงสุดได้เท่าไร ก่อนตัดสินใจ scale โครงสร้างพื้นฐาน เราจะออกแบบ **Benchmark Suite** ที่ครอบคลุม 3 flow หลักของระบบ: **Product Search**, **Checkout Flow**, และ **Order History**

### ขั้นตอนที่ 1: เตรียม Custom Scripts ทั้ง 3 Flow

```sql
-- suite_product_search.sql
\set category_id random(1, 20)
\set keyword_id random_zipfian(1, 5000, 1.1)

SELECT product_id, product_name, price, stock_quantity
FROM products
WHERE category_id = :category_id
  AND product_id = :keyword_id
ORDER BY price
LIMIT 20;
```

```sql
-- suite_checkout_flow.sql
\set customer_id random(1, 20000)
\set product_id random_zipfian(1, 5000, 1.3)
\set quantity random(1, 3)

BEGIN;

INSERT INTO orders (customer_id, order_status, total_amount)
VALUES (:customer_id, 'pending', 0)
RETURNING order_id \gset

INSERT INTO order_items (order_id, product_id, quantity, unit_price)
SELECT :order_id, :product_id, :quantity, price
FROM products
WHERE product_id = :product_id;

UPDATE orders
SET total_amount = (
    SELECT SUM(quantity * unit_price)
    FROM order_items
    WHERE order_id = :order_id
)
WHERE order_id = :order_id;

UPDATE products
SET stock_quantity = GREATEST(stock_quantity - :quantity, 0)
WHERE product_id = :product_id;

COMMIT;
```

```sql
-- suite_order_history.sql
\set customer_id random(1, 20000)

SELECT order_id, order_status, total_amount, created_at
FROM orders
WHERE customer_id = :customer_id
ORDER BY created_at DESC
LIMIT 10;
```

### ขั้นตอนที่ 2: สคริปต์ควบคุมการรัน Benchmark Suite ทั้งหมด

```bash
#!/bin/bash
# benchmark_suite.sh
# รัน benchmark suite เต็มรูปแบบ: baseline, scaling test, และ mixed workload

set -e

DB=ecommerce_bench
RESULT_DIR=./benchmark_results
mkdir -p $RESULT_DIR

echo "=========================================="
echo "  Mega Sale Readiness Benchmark Suite"
echo "=========================================="

# --- Phase 1: Baseline แต่ละ flow แยกกัน ---
echo ""
echo "[Phase 1] Baseline per-flow test (c=20, T=30s)"

for flow in product_search checkout_flow order_history; do
    echo "  -> Testing suite_${flow}.sql"
    pgbench -f suite_${flow}.sql -c 20 -j 4 -T 30 $DB \
        > "$RESULT_DIR/baseline_${flow}.log" 2>&1
    tps=$(grep "tps =" "$RESULT_DIR/baseline_${flow}.log" | awk '{print $3}')
    lat=$(grep "latency average" "$RESULT_DIR/baseline_${flow}.log" | awk '{print $4}')
    echo "     tps=$tps, latency_avg=${lat}ms"
done

# --- Phase 2: Connection scaling test บน mixed workload ---
echo ""
echo "[Phase 2] Connection scaling test (mixed 70% read / 30% write)"
echo "clients,tps,latency_avg_ms" > "$RESULT_DIR/scaling.csv"

for c in 10 25 50 100 150 200; do
    result=$(pgbench \
        -f suite_product_search.sql@40 \
        -f suite_order_history.sql@30 \
        -f suite_checkout_flow.sql@30 \
        -c $c -j 8 -T 30 $DB 2>&1)
    tps=$(echo "$result" | grep "tps =" | awk '{print $3}')
    lat=$(echo "$result" | grep "latency average" | awk '{print $4}')
    echo "  clients=$c -> tps=$tps, latency=${lat}ms"
    echo "$c,$tps,$lat" >> "$RESULT_DIR/scaling.csv"
    sleep 3
done

# --- Phase 3: Peak load simulation (Mega Sale scenario) ---
echo ""
echo "[Phase 3] Mega Sale simulation (heavier checkout ratio, longer duration)"
pgbench \
    -f suite_product_search.sql@30 \
    -f suite_order_history.sql@20 \
    -f suite_checkout_flow.sql@50 \
    -c 100 -j 8 -T 180 -P 30 -r $DB \
    > "$RESULT_DIR/mega_sale_simulation.log" 2>&1

echo "Done. Results in $RESULT_DIR/"
echo "=========================================="
```

### ขั้นตอนที่ 3: ผลลัพธ์ตัวอย่างและการวิเคราะห์

**Phase 1 — Baseline per-flow:**

| Flow | TPS | Latency avg |
|---|---|---|
| Product Search | 15,802.4 | 1.265 ms |
| Order History | 18,120.7 | 1.102 ms |
| Checkout Flow | 795.3 | 25.144 ms |

**สังเกต:** Checkout Flow ช้ากว่า flow อื่นถึง ~20 เท่า เพราะเป็น multi-statement write transaction — นี่คือ **คอขวดที่แท้จริง** ของระบบ ทีมควรโฟกัสการ optimize ที่ flow นี้เป็นอันดับแรก

**Phase 2 — Connection Scaling (mixed workload):**

```
clients,tps,latency_avg_ms
10,4820.665,2.074
25,9210.335,2.712
50,11540.887,4.331
100,11982.445,8.345
150,10230.112,14.667
200,8455.998,23.652
```

**สังเกต:** Sweet spot อยู่ที่ประมาณ 100 concurrent connection (TPS สูงสุด 11,982) เกินจากนี้ latency เพิ่มขึ้นเร็วกว่าที่ TPS จะเพิ่ม — ทีมควรตั้ง connection pool ไว้ที่ประมาณ 80-100 ไม่ใช่เปิดตรงเป็นพัน connection ตอน traffic พุ่ง

**Phase 3 — Mega Sale Simulation (checkout-heavy, 180 วินาที):**

```
progress: 30.0 s, 3204.7 tps, lat 31.203 ms stddev 18.442, 0 failed
progress: 60.0 s, 3156.2 tps, lat 31.678 ms stddev 19.001, 0 failed
progress: 90.0 s, 2988.4 tps, lat 33.452 ms stddev 21.220, 3 failed
progress: 120.0 s, 2801.6 tps, lat 35.789 ms stddev 24.556, 12 failed
progress: 150.0 s, 2650.3 tps, lat 37.902 ms stddev 27.881, 28 failed
progress: 180.0 s, 2512.8 tps, lat 40.221 ms stddev 31.005, 41 failed

number of transactions actually processed: 528940
number of failed transactions: 84 (0.016%)
latency average = 34.271 ms
tps = 2938.556781 (without initial connection time)

statement latencies in milliseconds and failures:
         0.892           0  BEGIN;
        12.045          31  INSERT INTO orders ...
         8.221           9  INSERT INTO order_items ...
         6.789          15  UPDATE orders SET total_amount ...
         5.212          29  UPDATE products SET stock_quantity ...
         1.112           0  COMMIT;
```

**สังเกตสำคัญ:** เมื่อจำลองสัดส่วน checkout สูงขึ้น (50%) ภายใต้ concurrent client 100 ตัวต่อเนื่อง 180 วินาที เริ่มมี **failed transactions เกิดขึ้น** (84 รายการจาก 528,940 หรือ 0.016%) และ TPS ลดลงเรื่อย ๆ ตามเวลา (จาก 3,204 → 2,512) พร้อม latency ที่เพิ่มขึ้นต่อเนื่อง — นี่คือสัญญาณของ **lock contention สะสม** บนแถว `products` ที่ถูก UPDATE บ่อยจาก stock_quantity โดยเฉพาะสินค้ายอดฮิตที่สุ่มด้วย `random_zipfian`

### ขั้นตอนที่ 4: สรุปผลและข้อเสนอแนะ (ตัวอย่างรายงาน)

```
รายงานผล Benchmark Suite: ความพร้อมของระบบสำหรับ Mega Sale
================================================================

1. คอขวดหลัก: Checkout Flow
   - Latency สูงกว่า flow อื่น 20 เท่า (25ms เทียบ 1-1.3ms)
   - ภายใต้โหลดสูงต่อเนื่อง เกิด failed transaction และ latency
     เพิ่มขึ้นเรื่อย ๆ ตามเวลา บ่งชี้ lock contention บนสินค้ายอดฮิต

2. Connection Pool Sweet Spot: ~100 connections
   - เกินจากนี้ TPS ลดลงขณะ latency พุ่งสูง
   - แนะนำตั้ง PgBouncer pool_size = 80-100 แทนการเปิด connection
     ตรงจากทุก request

3. ข้อเสนอแนะก่อน Mega Sale:
   a) พิจารณา optimistic locking หรือ batching การอัปเดต stock_quantity
      เพื่อลด contention บนสินค้ายอดฮิต
   b) ยืนยัน PgBouncer ตั้งค่า pool_size ตาม sweet spot ที่วัดได้
   c) รัน benchmark suite นี้ซ้ำหลัง apply การแก้ไขแต่ละจุด เพื่อวัด
      ผลจริงแบบ A/B (อ้างอิงกระบวนการจาก Step 971)
   d) ใช้ k6 หรือ Locust ทดสอบทั้งระบบ (รวม API + frontend) เพิ่มเติม
      ก่อนวันงานจริง เพื่อยืนยันว่าชั้น application ไม่เป็นคอขวดเพิ่ม
================================================================
```

แบบฝึกหัดนี้แสดงให้เห็นวงจรการทำงานแบบมืออาชีพครบวงจร: **ออกแบบ script ที่สะท้อน workload จริง → รัน benchmark หลายมุมมอง (per-flow, scaling, peak simulation) → วิเคราะห์ผลลัพธ์เชิงตัวเลข → สรุปเป็นข้อเสนอแนะที่นำไปปฏิบัติได้จริง** ซึ่งเป็นทักษะหลักของวิศวกรฐานข้อมูลระดับ world-class

---

## สรุปท้ายบท

- **Benchmark ไม่ใช่ทางเลือก แต่เป็นข้อบังคับ** ก่อนและหลังการเปลี่ยนแปลงระบบที่สำคัญ เพื่อพิสูจน์ผลด้วยตัวเลข ไม่ใช่การเดา — ต่อยอดจากการวัดระดับ query เดียวด้วย `EXPLAIN ANALYZE` ใน Part 074 ไปสู่การวัดระดับ workload ทั้งระบบ
- **`pgbench`** คือเครื่องมือ benchmark มาตรฐานที่มาพร้อม PostgreSQL ใช้ `-i` เพื่อสร้างข้อมูลทดสอบ และใช้ `-c`, `-j`, `-T`, `-t` ควบคุมรูปแบบการทดสอบ
- ผลลัพธ์หลักที่ต้องอ่านให้เป็นคือ **TPS** (ปริมาณงาน) และ **latency average/stddev** (ความเร็วและความสม่ำเสมอ)
- **Custom Script ด้วย `-f`** คือหัวใจของการ benchmark ที่มีความหมาย เพราะ workload จริงของแต่ละระบบไม่เหมือนกัน (e-commerce ต่างจาก banking ที่ default TPC-B จำลอง)
- **pgbench variables** (`\set`, `random()`, `random_zipfian()`) ช่วยสร้างข้อมูลทดสอบที่ใกล้เคียงพฤติกรรมจริง โดยเฉพาะการจำลอง "สินค้ายอดฮิต" ด้วย Zipfian distribution
- **A/B Testing** ด้วยการวัด baseline ก่อนเปลี่ยน แล้ววัดซ้ำหลังเปลี่ยนภายใต้เงื่อนไขเดียวกัน คือวิธีพิสูจน์ผลของ index หรือ configuration ที่น่าเชื่อถือที่สุด
- **Connection Scaling Test** ช่วยหา sweet spot ของจำนวน connection ที่เหมาะสม ซึ่งมักจะต่ำกว่าที่คาดคิดไว้มาก และนำไปสู่ความจำเป็นของ connection pooling
- การออกแบบ workload แบบ **Read-heavy/Write-heavy ผสม** ด้วย weighted `-f` ให้ผลลัพธ์ที่สะท้อนสภาพจริงของระบบมากกว่าการทดสอบ read หรือ write แยกกัน
- **k6, JMeter, Locust** คือเครื่องมือ load testing ระดับ application ที่ทดสอบทั้งระบบ (HTTP/API) ไม่ใช่แค่ database — ใช้ร่วมกับ pgbench เพื่อความมั่นใจแบบครบวงจร
- Benchmark Suite ที่ดีต้องครอบคลุมหลายมุมมอง: per-flow baseline, scaling test, และ peak load simulation พร้อมสรุปผลเป็นข้อเสนอแนะที่นำไปปฏิบัติได้จริง

---

## แบบฝึกหัด

<details>
<summary><strong>แบบฝึกหัดที่ 1:</strong> อธิบายความแตกต่างระหว่างการวัดผลด้วย <code>EXPLAIN ANALYZE</code> กับการวัดผลด้วย <code>pgbench</code></summary>

**เฉลย:**

`EXPLAIN ANALYZE` วัดประสิทธิภาพของ **query เดียว** แบบ isolated รันครั้งเดียวไม่มี concurrent load บอกได้ว่า planner เลือก execution plan แบบไหน และแต่ละ node ใช้เวลาเท่าไร

`pgbench` วัดประสิทธิภาพของระบบภายใต้ **concurrent workload จริง** ที่มีหลาย client ยิง transaction พร้อมกัน วัดผลรวมเป็น TPS และ latency ซึ่งจะเผยให้เห็นปัญหาที่ query เดียวไม่เจอ เช่น lock contention, connection overhead, และผลของ concurrent I/O — ทั้งสองเครื่องมือเสริมกัน: ใช้ `EXPLAIN ANALYZE` เพื่อ debug query เฉพาะจุด และใช้ `pgbench` เพื่อยืนยันผลในระดับระบบจริง

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 2:</strong> รัน <code>pgbench -i -s 20</code> บนฐานข้อมูลใหม่ แล้วบอกว่าตาราง <code>pgbench_accounts</code> จะมีกี่แถว</summary>

**เฉลย:**

```bash
createdb pgbench_test
pgbench -i -s 20 pgbench_test
```

`pgbench_accounts` จะมี **100,000 × 20 = 2,000,000 แถว** เพราะสูตรคำนวณคือ `100,000 × scale_factor` ส่วน `pgbench_branches` จะมี 20 แถว (1×20) และ `pgbench_tellers` จะมี 200 แถว (10×20)

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 3:</strong> อธิบายความแตกต่างระหว่าง <code>-T 60</code> กับ <code>-t 1000</code> และบอกว่าควรใช้แบบไหนเมื่อต้องการเปรียบเทียบ TPS ก่อน-หลังเพิ่ม index</summary>

**เฉลย:**

`-T 60` กำหนด**เวลา**ที่จะรัน (60 วินาที) โดยจำนวน transaction ที่รันได้จะแตกต่างกันไปตามความเร็วของระบบ

`-t 1000` กำหนด**จำนวน transaction** ที่แต่ละ client ต้องรันให้ครบ (1000 transaction ต่อ client) โดยเวลาที่ใช้จะต่างกันไปตามความเร็วของระบบ

เมื่อต้องการเปรียบเทียบ TPS ก่อน-หลังเพิ่ม index ทั้งสองแบบใช้ได้ แต่ **`-T` เป็นที่นิยมมากกว่า** เพราะควบคุมเวลาการทดสอบให้เท่ากันทุกรอบ ทำให้เปรียบเทียบ TPS ได้ตรงไปตรงมา (TPS คำนวณจาก transaction/เวลา อยู่แล้ว) ในขณะที่ `-t` จะทำให้แต่ละรอบใช้เวลาไม่เท่ากัน ซึ่งอาจกระทบต่อผลลัพธ์เพราะ cache warm-up หรือ background process (เช่น checkpoint) ที่เกิดขึ้นในช่วงเวลาต่างกัน

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 4:</strong> เขียน Custom Script pgbench สำหรับจำลองการ "เพิ่มสินค้าลงตะกร้า" โดยสมมติมีตาราง <code>cart_items (cart_id, product_id, quantity)</code></summary>

**เฉลย:**

```sql
-- add_to_cart.sql
\set cart_id random(1, 20000)
\set product_id random_zipfian(1, 5000, 1.2)
\set quantity random(1, 5)

INSERT INTO cart_items (cart_id, product_id, quantity)
VALUES (:cart_id, :product_id, :quantity)
ON CONFLICT (cart_id, product_id)
DO UPDATE SET quantity = cart_items.quantity + EXCLUDED.quantity;
```

รันด้วย:

```bash
pgbench -f add_to_cart.sql -c 30 -j 4 -T 60 ecommerce_bench
```

ใช้ `random_zipfian` สำหรับ `product_id` เพราะสินค้ายอดฮิตมักถูกเพิ่มลงตะกร้าบ่อยกว่าสินค้าทั่วไป และใช้ `ON CONFLICT ... DO UPDATE` เพื่อจำลองพฤติกรรมจริงที่ลูกค้าอาจเพิ่มสินค้าซ้ำเดิมในตะกร้า

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 5:</strong> อธิบายว่าทำไมการใช้ <code>random(1, 5000)</code> (uniform) สำหรับ <code>product_id</code> อาจให้ผล benchmark ที่คลาดเคลื่อนจากความจริง เทียบกับการใช้ <code>random_zipfian()</code></summary>

**เฉลย:**

`random(1, 5000)` แบบ uniform ทำให้สินค้าทุกชิ้นมีโอกาสถูกเลือกเท่ากันหมด ซึ่งไม่ตรงกับพฤติกรรมจริงที่สินค้ายอดนิยมถูกค้นหา/ซื้อบ่อยกว่าสินค้าทั่วไปมาก (Zipfian/80-20 rule) ผลกระทบต่อ benchมีสองทาง:

1. **Cache hit ratio ผิดจากจริง** — uniform random จะกระจาย row ที่ถูกเข้าถึงทั่วทั้งตาราง ทำให้ buffer cache ไม่สามารถเก็บ "hot set" ไว้ได้อย่างมีประสิทธิภาพ ในขณะที่ของจริงจะมี hot set เล็ก ๆ ที่อยู่ใน cache ตลอดเวลา (cache hit สูงกว่า)
2. **Lock contention ผิดจากจริง** — ถ้าเป็น workload write (เช่น update stock) การใช้ Zipfian จะทำให้เกิด contention สูงบน row ยอดฮิตเหมือนสถานการณ์จริง (เช่น ตอน flash sale สินค้าตัวเดียวถูกซื้อพร้อมกันหลายคน) ในขณะที่ uniform random จะกระจาย contention ออกไปจนแทบไม่เห็นปัญหานี้เลย

ดังนั้นถ้าต้องการผล benchmark ที่นำไปตัดสินใจ production ได้จริง ควรใช้ `random_zipfian()` แทน `random()` เมื่อจำลองการเข้าถึงข้อมูลที่มีความนิยมไม่เท่ากัน

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 6:</strong> ออกแบบขั้นตอน A/B Testing เพื่อพิสูจน์ว่าการเปลี่ยน <code>shared_buffers</code> จาก 128MB (default) เป็น 4GB ช่วยเพิ่ม TPS จริงหรือไม่</summary>

**เฉลย:**

```
1. กำหนด Custom Script ที่จะใช้ทดสอบ (เช่น mix ของ read query ที่กระทบข้อมูลใหญ่กว่า cache)

2. วัด Baseline:
   psql -c "SHOW shared_buffers;"   -- ยืนยันว่ายังเป็น 128MB
   pgbench -f test_script.sql -c 20 -j 4 -T 60 -P 15 mydb
   (บันทึกผล tps, latency average/stddev — รันซ้ำ 3-5 รอบเก็บ median)

3. เปลี่ยนค่า:
   ALTER SYSTEM SET shared_buffers = '4GB';
   -- ต้อง restart PostgreSQL เพราะ shared_buffers เป็น parameter
   -- ที่เปลี่ยนแบบ reload ไม่ได้ ต้อง restart service เท่านั้น
   sudo systemctl restart postgresql

4. วัดผลหลังเปลี่ยน (เงื่อนไขเดียวกันทุกประการ: script, -c, -j, -T):
   pgbench -f test_script.sql -c 20 -j 4 -T 60 -P 15 mydb
   (รันซ้ำ 3-5 รอบเช่นกัน)

5. เปรียบเทียบผลลัพธ์:
   - ดู % การเปลี่ยนแปลงของ TPS และ latency average
   - ตรวจสอบว่าความแตกต่างมากกว่าความแปรปรวนปกติ (จาก 3-5 รอบ) หรือไม่

6. สรุปผล: ถ้า TPS เพิ่มขึ้นอย่างมีนัยสำคัญและ latency ลดลง แสดงว่าการเพิ่ม
   shared_buffers ช่วยจริง (โดยเฉพาะถ้าขนาดข้อมูล > 128MB เดิม แต่ < 4GB ใหม่
   ซึ่งหมายถึงข้อมูลที่เคยต้องอ่านจาก disk จะ fit ใน buffer cache ได้มากขึ้น)
```

ข้อควรระวัง: `shared_buffers` เป็น parameter ที่ต้อง **restart PostgreSQL** ถึงจะมีผล (ไม่ใช่แค่ `pg_reload_conf()`) ต้องวางแผนช่วงเวลาทดสอบให้เหมาะสม

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 7:</strong> จากผลการทำ Connection Scaling Test พบว่า TPS สูงสุดอยู่ที่ 40 connections บนเครื่อง 8 core แต่ระบบ production ต้องรองรับผู้ใช้พร้อมกัน 2,000 คน ควรทำอย่างไร</summary>

**เฉลย:**

ไม่ควรเปิด connection ตรงจากทุก user เข้า PostgreSQL (2,000 connections) เพราะเกิน sweet spot มาก จะทำให้ TPS ตกและ latency พุ่งสูงตามที่เห็นในผลทดสอบ (เช่นในตัวอย่าง Step 972 ที่ 200 connections ให้ TPS ต่ำกว่า 10 connections)

วิธีที่ถูกต้องคือใช้ **Connection Pooler** เช่น PgBouncer วางไว้ระหว่าง application กับ PostgreSQL โดยตั้ง `pool_size` ให้ใกล้เคียง sweet spot ที่วัดได้ (เช่น 40-50 connections จริงไปยัง PostgreSQL) ในขณะที่ application เชื่อมต่อ PgBouncer ได้มากถึง 2,000 connection พร้อมกัน โดย PgBouncer จะจัดคิวคำขอให้ connection จริงที่มีจำกัดอย่างมีประสิทธิภาพ

ควรทดสอบยืนยันด้วย pgbench ผ่าน PgBouncer โดยตรง (เปลี่ยน `-p` ไปที่พอร์ตของ PgBouncer) เพื่อยืนยันว่าการตั้งค่า pool_size ให้ TPS และ latency ที่ยอมรับได้ภายใต้โหลด 2,000 concurrent client จริง

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 8:</strong> เขียนคำสั่ง pgbench ที่รัน workload แบบผสม 3 scripts ในสัดส่วน 50% search, 35% order history, 15% checkout ด้วย 50 concurrent clients เป็นเวลา 90 วินาที</summary>

**เฉลย:**

```bash
pgbench \
  -f search.sql@50 \
  -f order_history.sql@35 \
  -f checkout.sql@15 \
  -c 50 -j 8 -T 90 -P 15 -r \
  ecommerce_bench
```

น้ำหนัก (weight) 50, 35, 15 รวมกันเป็น 100 ตรงกับสัดส่วนเปอร์เซ็นต์ที่ต้องการพอดี (50%, 35%, 15%) การเพิ่ม `-r` ช่วยให้เห็น latency แยกตาม statement ภายในแต่ละ script ซึ่งมีประโยชน์ในการหาคอขวด และ `-P 15` ให้เห็น progress ทุก 15 วินาทีเพื่อสังเกตว่า performance คงที่ตลอด 90 วินาทีหรือไม่

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 9:</strong> อธิบายว่าทำไมทีม QA ที่ต้องการทดสอบว่า Frontend + API + Database ทั้งระบบรองรับ 5,000 concurrent user ได้หรือไม่ ไม่ควรใช้ pgbench เพียงอย่างเดียว</summary>

**เฉลย:**

`pgbench` ทดสอบเฉพาะระดับ **SQL/database โดยตรง** เท่านั้น ไม่ผ่าน load balancer, application server, business logic, caching layer, หรือ network latency ระหว่าง client กับ API เหมือนที่ผู้ใช้จริงเจอ ดังนั้นผลลัพธ์จาก pgbench บอกได้แค่ว่า "ฐานข้อมูลเพียงอย่างเดียว" รองรับโหลดได้เท่าไร แต่บอกไม่ได้ว่า **ทั้งระบบ** (รวม frontend rendering, API response time, caching, rate limiting ฯลฯ) รองรับ 5,000 concurrent user ได้จริงหรือไม่ เพราะคอขวดอาจอยู่ที่ชั้นอื่นที่ไม่ใช่ database เลยก็ได้ (เช่น API server เดี่ยวรับ connection ไม่พอ, cache layer ทำงานผิดพลาด, network bandwidth ไม่พอ)

ทีม QA ควรใช้เครื่องมือระดับ HTTP/application เช่น k6, JMeter, หรือ Locust ที่จำลอง user เรียก API จริงผ่าน HTTP เพื่อทดสอบทั้ง stack ควบคู่ไปกับการใช้ pgbench เพื่อยืนยันว่าฐานข้อมูล (ซึ่งมักเป็นจุดที่ scale ยากที่สุด) พร้อมรองรับ load ที่ต้นทางส่งมาจริง

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 10:</strong> ออกแบบ Benchmark Suite แบบย่อ (mini) สำหรับฟีเจอร์ "รีวิวสินค้า" ที่มีตาราง <code>reviews (review_id, product_id, customer_id, rating, comment, created_at)</code> โดยต้องมีทั้ง read query (ดูรีวิวของสินค้า) และ write query (เขียนรีวิวใหม่)</summary>

**เฉลย:**

```sql
-- review_read.sql
\set product_id random_zipfian(1, 5000, 1.2)

SELECT review_id, customer_id, rating, comment, created_at
FROM reviews
WHERE product_id = :product_id
ORDER BY created_at DESC
LIMIT 10;
```

```sql
-- review_write.sql
\set product_id random_zipfian(1, 5000, 1.2)
\set customer_id random(1, 20000)
\set rating random(1, 5)

INSERT INTO reviews (product_id, customer_id, rating, comment, created_at)
VALUES (:product_id, :customer_id, :rating, 'Sample review text', now());
```

รัน Benchmark Suite แบบผสม (สมมติ read:write = 90:10 เพราะคนอ่านรีวิวมากกว่าคนเขียนมาก):

```bash
pgbench \
  -f review_read.sql@90 \
  -f review_write.sql@10 \
  -c 30 -j 4 -T 60 -P 15 -r \
  ecommerce_bench
```

แนวคิดสำคัญที่ใช้:
- ใช้ `random_zipfian` สำหรับ `product_id` เพราะสินค้ายอดฮิตมักถูกอ่าน/เขียนรีวิวมากกว่า
- สัดส่วน weight 90:10 สะท้อนพฤติกรรมจริงที่คนอ่านรีวิวมากกว่าคนเขียนมาก
- ควรทดสอบต่อว่า ถ้ามี index บน `reviews(product_id, created_at DESC)` จะช่วย query read ได้แค่ไหน โดยทำ A/B Test ตามกระบวนการใน Step 971

</details>

---

**บทถัดไป:** [Part 099: Disaster Recovery](./part-099-disaster-recovery.md)
