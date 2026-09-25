# หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับมืออาชีพ | Part 067

> **Part 067 — Load Balancing และ Read Replica Strategy (Steps 661–670)**

---

## เป้าหมายการเรียนรู้

เมื่อจบบทนี้ ผู้เรียนจะสามารถ:

1. อธิบายแนวคิด Read Replica และบทบาทของ Standby server (ต่อยอดจาก Part 063 — Streaming Replication) ในการรับ traffic การอ่านเพื่อลดภาระของ Primary
2. ออกแบบสถาปัตยกรรม Read/Write Splitting ที่แยก write ไปยัง Primary และ read ไปยัง Replica ได้อย่างถูกต้อง
3. เปรียบเทียบแนวทาง Application-level routing กับ Proxy-level routing พร้อมข้อดี-ข้อเสียของแต่ละแบบ
4. ตั้งค่า HAProxy สำหรับทำ routing ตาม health check รวมถึงเขียนไฟล์ `haproxy.cfg` ที่ใช้งานได้จริง
5. เข้าใจปัญหา Replication Lag และผลกระทบต่อ Read-after-write consistency พร้อมแนวทางแก้ไขแบบ read-your-writes
6. เลือกใช้ Load Balancing Algorithm ที่เหมาะสม (round-robin, least-connections) เมื่อมี replica หลายตัว
7. ออกแบบ Health Check ที่ตรวจสอบทั้งสถานะ healthy และ replication lag ก่อนตัดสินใจ route traffic
8. เข้าใจผลกระทบของ Failover ต่อ Load Balancer และการเชื่อมโยงกับ Patroni/HAProxy (ต่อยอดจาก Part 065)
9. เข้าใจแนวคิดเบื้องต้นของ Geographic/Multi-region considerations เช่น latency และการเลือก replica ที่ใกล้ผู้ใช้ที่สุด
10. ออกแบบสถาปัตยกรรม read replica + load balancing แบบเต็มรูปแบบสำหรับระบบ e-commerce ที่มีผู้ใช้จำนวนมาก

---

## บริบทก่อนเริ่มบทเรียน

ก่อนเข้าสู่เนื้อหา ขอทบทวนสิ่งที่เราเรียนมาแล้วซึ่งเป็นพื้นฐานของบทนี้:

- **Part 063 (Streaming Replication)**: เราได้เรียนรู้วิธีตั้งค่า `primary_conninfo`, `pg_basebackup`, replication slot, และการสร้าง Standby server ที่ sync ข้อมูลจาก Primary แบบ real-time (หรือใกล้ real-time)
- **Part 065 (High Availability)**: เราได้เรียนรู้ Patroni สำหรับทำ automatic failover และการใช้ HAProxy เป็น proxy หน้า cluster
- **Part 066 (Connection Pooling)**: เราได้เรียนรู้ PgBouncer และการจัดการ connection pool เพื่อลดภาระการเปิด-ปิด connection

บทนี้ (Part 067) จะเชื่อมทุกอย่างเข้าด้วยกัน โดยเน้นที่ **กลยุทธ์การกระจาย traffic การอ่าน (read traffic)** ไปยัง replica หลายตัว เพื่อให้ระบบรองรับโหลดสูงได้ — นี่คือหัวใจของการ "scale การอ่าน" (read scaling) ซึ่งแตกต่างจากการ scale การเขียนที่ทำได้ยากกว่ามากใน RDBMS แบบดั้งเดิม

```
┌─────────────────────────────────────────────────────────────────┐
│                     ภาพรวมสถาปัตยกรรมที่จะเรียนในบทนี้              │
│                                                                     │
│   Application                                                     │
│       │                                                            │
│       ▼                                                            │
│   ┌─────────────┐        Health Check (Step 667)                  │
│   │ Load Balancer│◄──────────────────────────┐                    │
│   │  (HAProxy)   │                            │                    │
│   └──────┬───────┘                            │                    │
│      ┌───┴────┬─────────┬─────────┐           │                    │
│      ▼        ▼         ▼         ▼           │                    │
│  ┌────────┐ ┌──────┐ ┌──────┐ ┌──────┐         │                    │
│  │Primary │ │Replica│ │Replica│ │Replica│◄──────┘                    │
│  │ (RW)   │ │  1(RO)│ │  2(RO)│ │  3(RO)│                            │
│  └────────┘ └──────┘ └──────┘ └──────┘                            │
│      ▲          ▲        ▲        ▲                                │
│      └──── streaming replication (Part 063) ────┘                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Step 661: Read Replica คืออะไร

### 661.1 แนวคิดพื้นฐาน

**Read Replica** คือ Standby server ที่เราสร้างขึ้นด้วยกลไก Streaming Replication (ตามที่เรียนใน Part 063) แต่แทนที่จะปล่อยให้มันรอ "เผื่อไว้ใช้ตอน failover" อย่างเดียว เราเปิดให้มันรับ connection สำหรับ **query อ่านข้อมูล (SELECT)** ได้ด้วย ผ่านโหมด **Hot Standby** ของ PostgreSQL

จุดสำคัญคือ PostgreSQL standby server ที่ตั้งค่า `hot_standby = on` (ค่า default ตั้งแต่ PostgreSQL 10 เป็นต้นมาเมื่อใช้ replication) จะสามารถรับ read-only query ได้พร้อม ๆ กับที่มันกำลัง apply WAL จาก primary อยู่เบื้องหลัง

```sql
-- ตรวจสอบว่า standby เปิด hot_standby หรือไม่
SHOW hot_standby;
--  hot_standby
-- -------------
--  on

-- ตรวจสอบว่า instance นี้เป็น standby (recovery mode) หรือไม่
SELECT pg_is_in_recovery();
--  pg_is_in_recovery
-- --------------------
--  t
```

### 661.2 ทำไมต้องใช้ Read Replica

ระบบฐานข้อมูลส่วนใหญ่ในโลกจริง โดยเฉพาะระบบ e-commerce, social media, content platform มีอัตราส่วนการอ่าน (read) ต่อการเขียน (write) สูงมาก โดยทั่วไปอยู่ที่ประมาณ **80/20 ถึง 95/5** (อ่าน 80-95%, เขียน 5-20%)

```
ตัวอย่างสัดส่วน query ของระบบ e-commerce ทั่วไป:

  READ  ████████████████████████████████████████░░░░  85%
        - แสดงสินค้า (product listing)
        - ค้นหาสินค้า (search)
        - ดูตะกร้าสินค้า
        - ดูประวัติคำสั่งซื้อ
        - แสดง dashboard/report

  WRITE ██████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  15%
        - สั่งซื้อสินค้า (checkout)
        - อัปเดตสต็อก
        - เพิ่ม/แก้ไขสินค้าในตะกร้า
        - เขียนรีวิว
```

เมื่อสัดส่วนเป็นแบบนี้ การให้ Primary รับภาระทั้งหมดคนเดียวจะทำให้:

1. **CPU/Memory/Disk I/O ของ Primary ตึงเกินไป** — เพราะต้องรับทั้ง write transaction ที่ต้องการความเร็วต่ำ (low latency) และ read query ที่อาจกิน resource สูง (เช่น report query, full-text search)
2. **Lock contention เพิ่มขึ้น** — query อ่านหนัก ๆ อาจไปแย่ง buffer cache หรือ I/O bandwidth กับ transaction เขียนที่ critical กว่า
3. **Single Point of Bottleneck** — ถ้า Primary ล่มหรือช้า ทั้งระบบ (ทั้งอ่านและเขียน) จะกระทบหมด

การกระจาย read query ไปยัง Replica จึงช่วย:

- ลดภาระ Primary ให้เหลือแต่ write และ read ที่ต้อง strong consistency จริง ๆ
- เพิ่ม throughput รวมของระบบ เพราะยิ่งมี replica มาก ยิ่งรองรับ concurrent read ได้มาก (scale out แนวนอน)
- เพิ่มความทนทาน (resilience) — ถ้า replica ตัวหนึ่งมีปัญหา ยังมีตัวอื่นรับต่อได้
- ใช้ replica เพื่อ isolate งานหนัก เช่น analytics/report queries ไม่ให้กระทบ transactional workload บน Primary

### 661.3 ข้อจำกัดที่ต้องเข้าใจก่อนออกแบบ

Read Replica ไม่ใช่ยาวิเศษ มีข้อจำกัดสำคัญที่ต้องรู้:

| ข้อจำกัด | รายละเอียด |
|---|---|
| **Replication Lag** | ข้อมูลใน replica อาจ "ตามหลัง" primary อยู่เสมอ (มิลลิวินาทีถึงวินาที หรือมากกว่านั้นถ้าโหลดสูง) — จะอธิบายละเอียดใน Step 665 |
| **Read-only เท่านั้น** | Replica รับได้แค่ SELECT ไม่สามารถรับ INSERT/UPDATE/DELETE ได้ (ถ้าพยายามเขียนจะได้ error `cannot execute INSERT in a read-only transaction`) |
| **ไม่ scale การเขียน** | Replica ช่วย scale read เท่านั้น การเขียนยังคงกระจุกอยู่ที่ Primary ตัวเดียว (การ scale write ต้องใช้เทคนิคอื่น เช่น sharding ซึ่งอยู่นอกขอบเขตบทนี้) |
| **Query บางประเภทอาจถูก cancel** | ถ้า Primary ทำ VACUUM หรือมี DDL ที่ชน WAL replay กับ query ยาว ๆ บน standby อาจโดน cancel (เกี่ยวข้องกับ `max_standby_streaming_delay`) |

```sql
-- ทดสอบว่า replica ปฏิเสธการเขียนจริง
INSERT INTO products (name, price) VALUES ('Test', 100);
-- ERROR:  cannot execute INSERT in a read-only transaction
```

### 661.4 สถาปัตยกรรมพื้นฐาน

```
                    ┌───────────────────────┐
                    │      Application       │
                    └───────────┬────────────┘
                                │
                  ตัดสินใจ: write หรือ read?
                                │
              ┌─────────────────┴─────────────────┐
              │ WRITE                              │ READ
              ▼                                     ▼
    ┌──────────────────┐               ┌─────────────────────────┐
    │     Primary       │               │   Replica Pool (RO)     │
    │  (Read + Write)   │──WAL stream──▶│  Replica-1 Replica-2 ... │
    └──────────────────┘               └─────────────────────────┘
```

> **หมายเหตุสำคัญ**: Primary เองก็ยังสามารถรับ read query ได้เสมอ (มันไม่ใช่ write-only) เพียงแต่เราพยายาม "ระบาย" read query ส่วนใหญ่ออกไปที่ replica เพื่อลดภาระ ส่วน write query ทั้งหมด **ต้อง** ไปที่ Primary เท่านั้น เพราะ PostgreSQL (ในโหมด replication มาตรฐาน ไม่รวม multi-master extension เช่น BDR) ไม่รองรับการเขียนที่ standby

---

## Step 662: Read/Write Splitting — สถาปัตยกรรมแยก Write ไป Primary, Read ไป Replica

### 662.1 หลักการออกแบบ

**Read/Write Splitting** คือรูปแบบสถาปัตยกรรมที่แยกเส้นทาง (routing path) ของ query ตามประเภทของมัน:

- **Write path**: query ที่มีการเปลี่ยนแปลงข้อมูล (INSERT, UPDATE, DELETE, รวมถึง DDL) → ส่งไปที่ **Primary** เท่านั้น
- **Read path**: query ที่อ่านข้อมูลอย่างเดียว (SELECT ที่ไม่มี `FOR UPDATE`/`FOR SHARE`) → ส่งไปที่ **Replica** (หรือ Primary ก็ได้ถ้าจำเป็นต้องอ่านข้อมูลล่าสุด)

```
┌────────────────────────────────────────────────────────────────┐
│                   Read/Write Splitting Flow                     │
│                                                                    │
│   SQL Query เข้ามา                                                │
│        │                                                          │
│        ▼                                                          │
│   ┌─────────────────────┐                                         │
│   │  Query Classifier    │  ← วิเคราะห์ว่าเป็น Read หรือ Write        │
│   └──────────┬───────────┘                                        │
│         ┌─────┴─────┐                                             │
│         │           │                                             │
│    Write Query   Read Query                                       │
│         │           │                                             │
│         ▼           ▼                                             │
│    ┌─────────┐  ┌─────────────┐                                   │
│    │ Primary │  │ Replica Pool │                                  │
│    └─────────┘  └─────────────┘                                   │
└────────────────────────────────────────────────────────────────┘
```

### 662.2 กรณีที่ query ต้อง "บังคับ" ไป Primary แม้จะเป็น SELECT

มีหลายกรณีที่แม้ query จะเป็น SELECT แต่ต้องส่งไป Primary เพื่อความถูกต้องของข้อมูล:

1. **SELECT ... FOR UPDATE / FOR SHARE** — เพราะเป็นการ lock row เพื่อเตรียมเขียนต่อ ต้องทำที่ Primary
2. **SELECT ภายใน transaction เดียวกับที่มีการเขียน** — เช่น อ่านค่า balance แล้วจะ UPDATE ต่อในทันที (read-modify-write pattern) ต้องอยู่ transaction เดียวกันที่ Primary เพื่อความ atomic
3. **SELECT ที่ต้องการข้อมูล real-time ทันทีหลังเขียน** — เกี่ยวข้องกับ read-after-write consistency (Step 665)
4. **Query ที่ใช้ session-level temporary object** เช่น `CREATE TEMP TABLE` ตามด้วย SELECT — temp table มีอยู่แค่ session เดียวที่ primary เท่านั้น
5. **Query ที่เรียก sequence เช่น `nextval()`** — sequence เปลี่ยนสถานะ ต้องทำที่ Primary

```sql
-- ตัวอย่างที่ต้องบังคับไป Primary แม้เป็น SELECT
BEGIN;
SELECT balance FROM accounts WHERE id = 1 FOR UPDATE;  -- ต้องที่ Primary
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
COMMIT;
```

### 662.3 สถาปัตยกรรมแบบ Layer โดยรวม

```
┌──────────────────────────────────────────────────────────────────────┐
│                         Full Architecture Layers                       │
│                                                                          │
│  Layer 1: Application (Business Logic)                                 │
│      ↓ เรียกผ่าน Data Access Layer / ORM                                │
│  Layer 2: Connection Routing (Application-level หรือ Proxy-level)       │
│      ↓                                                                  │
│  Layer 3: Connection Pooling (PgBouncer - Part 066)                    │
│      ↓                                                                  │
│  Layer 4: Load Balancing (HAProxy - Step 664)                          │
│      ↓                                                                  │
│  Layer 5: PostgreSQL Cluster (Primary + Replicas - Part 063)           │
└──────────────────────────────────────────────────────────────────────┘
```

> ในระบบจริงหลายแห่งจะรวม Layer 2-4 เข้าด้วยกัน เช่น PgBouncer + HAProxy ทำงานร่วมกัน หรือใช้ proxy ที่ฉลาดพอจะทำทั้ง connection pooling และ query routing ในตัวเดียว เช่น **pgpool-II** หรือ **PgCat** — แต่ในบทนี้เราจะเน้นการประกอบ component มาตรฐาน (HAProxy) เพื่อให้เข้าใจกลไกจริง

### 662.4 ตารางเปรียบเทียบ Where to Route

| ประเภท Query | ตัวอย่าง | ปลายทาง |
|---|---|---|
| Simple SELECT | `SELECT * FROM products WHERE id = 1` | Replica |
| SELECT with JOIN/aggregate | `SELECT COUNT(*) FROM orders GROUP BY status` | Replica |
| SELECT FOR UPDATE | `SELECT * FROM inventory WHERE sku='X' FOR UPDATE` | **Primary** |
| INSERT/UPDATE/DELETE | `INSERT INTO orders (...) VALUES (...)` | **Primary** |
| DDL | `ALTER TABLE products ADD COLUMN ...` | **Primary** |
| Sequence | `SELECT nextval('order_id_seq')` | **Primary** |
| Read-after-write critical | อ่านค่าที่เพิ่งเขียนเอง (เช่น หน้า "ยืนยันคำสั่งซื้อ") | **Primary** (หรือใช้เทคนิค Step 665) |
| Report/Analytics | `SELECT ... FROM sales_summary WHERE date > ...` | Replica (เฉพาะที่ยอมรับ lag ได้) |

---

## Step 663: Application-level Routing

### 663.1 แนวคิด

**Application-level routing** คือการที่โค้ดของแอปพลิเคชันเองเป็นผู้ตัดสินใจว่าจะเลือกใช้ connection ไปยัง Primary หรือ Replica โดยไม่ผ่าน proxy ตัวกลางที่ทำหน้าที่ตัดสินใจแทน

```
┌────────────────────────────────────────────────────────────┐
│                Application-level Routing                     │
│                                                                │
│   Application Code                                            │
│   ┌──────────────────────────────────────┐                    │
│   │  if (isWriteQuery(sql)) {              │                    │
│   │      conn = primaryPool.getConnection()│──────┐             │
│   │  } else {                               │      │             │
│   │      conn = replicaPool.getConnection()│──┐   │             │
│   │  }                                      │  │   │             │
│   └──────────────────────────────────────┘  │   │             │
│                                                │   │             │
│                                                ▼   ▼             │
│                                         ┌─────────────┐         │
│                                         │  Replica    │         │
│                                         │  Pool       │         │
│                                         └─────────────┘         │
│                                                    ┌─────────┐   │
│                                                    │ Primary │   │
│                                                    └─────────┘   │
└────────────────────────────────────────────────────────────┘
```

### 663.2 ตัวอย่าง Pseudocode

```python
# --- config.py ---
PRIMARY_DSN = "host=pg-primary.internal port=5432 dbname=shop"
REPLICA_DSNS = [
    "host=pg-replica-1.internal port=5432 dbname=shop",
    "host=pg-replica-2.internal port=5432 dbname=shop",
    "host=pg-replica-3.internal port=5432 dbname=shop",
]

# --- connection_router.py ---
import random
import psycopg2
from contextlib import contextmanager

class ConnectionRouter:
    def __init__(self, primary_dsn, replica_dsns):
        self.primary_pool = create_pool(primary_dsn, max_size=20)
        self.replica_pools = [
            create_pool(dsn, max_size=20) for dsn in replica_dsns
        ]

    def get_write_connection(self):
        """ทุก write ต้องผ่านฟังก์ชันนี้เสมอ"""
        return self.primary_pool.acquire()

    def get_read_connection(self):
        """เลือก replica แบบ round-robin หรือ random
        (รายละเอียด algorithm ดู Step 666)"""
        pool = random.choice(self.replica_pools)
        return pool.acquire()

    @contextmanager
    def route(self, readonly: bool = True):
        conn = self.get_read_connection() if readonly else self.get_write_connection()
        try:
            yield conn
        finally:
            conn.release()


router = ConnectionRouter(PRIMARY_DSN, REPLICA_DSNS)


# --- repository.py: ตัวอย่างการใช้งานจริงใน Data Access Layer ---
def get_product_by_id(product_id: int):
    # อ่านข้อมูล -> ไป replica
    with router.route(readonly=True) as conn:
        cur = conn.cursor()
        cur.execute("SELECT * FROM products WHERE id = %s", (product_id,))
        return cur.fetchone()

def create_order(user_id: int, items: list):
    # เขียนข้อมูล -> ไป primary เสมอ
    with router.route(readonly=False) as conn:
        cur = conn.cursor()
        cur.execute(
            "INSERT INTO orders (user_id, status) VALUES (%s, 'pending') RETURNING id",
            (user_id,)
        )
        order_id = cur.fetchone()[0]
        for item in items:
            cur.execute(
                "INSERT INTO order_items (order_id, sku, qty) VALUES (%s, %s, %s)",
                (order_id, item["sku"], item["qty"])
            )
        conn.commit()
        return order_id
```

### 663.3 แบบ Decorator/Annotation (แนวทางที่ ORM หลายตัวใช้)

หลาย framework/ORM สมัยใหม่ (เช่น Django with `using()`, Rails with multiple databases, Spring `@Transactional(readOnly=true)`) มีกลไกให้ mark ว่า method หรือ query block ไหนเป็น read-only เพื่อให้ router เลือกปลายทางอัตโนมัติ

```python
# ตัวอย่างแนวทาง decorator-based routing (แนวคิด ไม่ใช่ library จริง)
def read_only(func):
    def wrapper(*args, **kwargs):
        with router.route(readonly=True) as conn:
            return func(conn, *args, **kwargs)
    return wrapper

def read_write(func):
    def wrapper(*args, **kwargs):
        with router.route(readonly=False) as conn:
            return func(conn, *args, **kwargs)
    return wrapper


@read_only
def list_products(conn, category: str):
    cur = conn.cursor()
    cur.execute("SELECT * FROM products WHERE category = %s", (category,))
    return cur.fetchall()

@read_write
def update_stock(conn, sku: str, qty: int):
    cur = conn.cursor()
    cur.execute("UPDATE inventory SET qty = qty - %s WHERE sku = %s", (qty, sku))
    conn.commit()
```

### 663.4 ข้อดี-ข้อเสียของ Application-level Routing

| ด้าน | ข้อดี | ข้อเสีย |
|---|---|---|
| **Performance** | ไม่มี network hop เพิ่ม (ไม่ผ่าน proxy กลาง) → latency ต่ำสุด | — |
| **Flexibility** | ควบคุมได้ละเอียดระดับ query/business-logic เช่น "query นี้ยอมรับ lag ได้ 5 วินาที" | ต้องเขียนโค้ด logic เองทุกจุดที่เรียก DB |
| **ความซับซ้อนของโค้ด** | — | นักพัฒนาต้อง "จำ" ว่าต้องเลือก pool ถูกทุกครั้ง เสี่ยง human error (ลืม mark write query ว่าต้องไป primary) |
| **Failover Awareness** | — | แอปต้องรู้เองว่า replica ตัวไหน down/lag เกิน ต้อง implement health check เอง (ซ้ำซ้อนกับ Step 667) |
| **Multi-language/Multi-service** | — | ถ้าระบบมีหลายภาษา/หลาย service ต้อง implement routing logic ซ้ำในทุกที่ (ไม่ centralize) |
| **Testing** | ทดสอบ logic การ route ได้ผ่าน unit test โดยตรง | ต้องทดสอบทุก client แยกกัน |
| **Deployment** | — | เปลี่ยน topology (เช่น เพิ่ม replica ใหม่) ต้อง deploy โค้ดแอปใหม่ หรือใช้ config ภายนอก (service discovery) |

> **บทสรุป**: Application-level routing เหมาะกับทีมที่มีความสามารถควบคุมโค้ดได้เต็มที่ ต้องการ performance สูงสุด และมี service ไม่มากนัก (monolith หรือ few microservices) แต่จะเริ่มมีปัญหาเรื่อง maintainability เมื่อระบบโตขึ้นมี microservices จำนวนมาก — นี่คือเหตุผลที่หลายองค์กรหันไปใช้ Proxy-level routing แทน (Step 664)

---

## Step 664: Proxy-level Routing — HAProxy

### 664.1 แนวคิด

**Proxy-level routing** คือการวาง proxy server (ตัวกลาง) ไว้ระหว่าง Application กับ PostgreSQL cluster เพื่อทำหน้าที่ตัดสินใจ routing แทนแอป แอปเพียงแค่เชื่อมต่อไปที่ proxy จุดเดียว (หรือสองจุด แยก write endpoint / read endpoint) โดยไม่ต้องรู้ topology ของ cluster เบื้องหลังเลย

```
┌───────────────────────────────────────────────────────────────┐
│                    Proxy-level Routing (HAProxy)                 │
│                                                                     │
│   Application                                                     │
│       │                                                            │
│       │  เชื่อมต่อไป 2 endpoint:                                    │
│       │  - write.db.internal:5432  (Primary only)                 │
│       │  - read.db.internal:5432   (Replica pool)                 │
│       ▼                                                            │
│   ┌─────────────────────────────────────────┐                     │
│   │              HAProxy                      │                     │
│   │  ┌─────────────┐      ┌────────────────┐  │                     │
│   │  │ frontend     │      │ frontend        │  │                     │
│   │  │ write_front  │      │ read_front      │  │                     │
│   │  │ :5432        │      │ :5433           │  │                     │
│   │  └──────┬───────┘      └────────┬────────┘  │                     │
│   │         ▼                        ▼           │                     │
│   │  ┌─────────────┐      ┌────────────────┐  │                     │
│   │  │ backend      │      │ backend         │  │                     │
│   │  │ write_back   │      │ read_back       │  │                     │
│   │  │ (health check│      │ (health check +│  │                     │
│   │  │  is_primary) │      │  load balance) │  │                     │
│   │  └──────┬───────┘      └────────┬────────┘  │                     │
│   └─────────┼────────────────────────┼──────────┘                     │
│              ▼                        ▼                                │
│         ┌─────────┐         ┌──────┬──────┬──────┐                    │
│         │ Primary │         │Rep-1 │Rep-2 │Rep-3 │                    │
│         └─────────┘         └──────┴──────┴──────┘                    │
└───────────────────────────────────────────────────────────────┘
```

### 664.2 กลไก Health Check ที่แยกแยะ Primary vs Replica

HAProxy เองไม่เข้าใจ "PostgreSQL replication topology" โดยตรง แต่สามารถใช้ **HTTP health check endpoint** ที่รายงานสถานะ role (primary/replica) ผ่านเครื่องมือช่วย เช่น:

- **`pg_isready`** — เช็คแค่ว่า instance ตอบสนองไหม (ไม่บอก role)
- **`xinetd` + shell script ที่เรียก `pg_is_in_recovery()`** — วิธีคลาสสิกที่ HAProxy คู่กับ PostgreSQL ใช้กันมานาน
- **Patroni REST API** (`/primary`, `/replica`, `/health`) — ถ้าใช้ Patroni จัดการ cluster (ตาม Part 065) จะสะดวกที่สุด เพราะ Patroni มี HTTP endpoint บอก role ให้อยู่แล้ว

```
GET http://patroni-node:8008/primary   → HTTP 200 ถ้าเป็น primary, HTTP 503 ถ้าไม่ใช่
GET http://patroni-node:8008/replica   → HTTP 200 ถ้าเป็น replica ที่ healthy, HTTP 503 ถ้าไม่ใช่
```

### 664.3 ตัวอย่าง haproxy.cfg แบบเต็ม (ใช้ร่วมกับ Patroni REST API)

```cfg
# /etc/haproxy/haproxy.cfg
# ==============================================================
# HAProxy configuration for PostgreSQL Read/Write Splitting
# ทำงานร่วมกับ Patroni cluster (3 nodes: pg-node1, pg-node2, pg-node3)
# ==============================================================

global
    log         127.0.0.1 local2
    maxconn     4000
    daemon
    stats socket /var/run/haproxy/admin.sock mode 660 level admin

defaults
    log         global
    mode        tcp
    retries     2
    timeout client   30s
    timeout connect  4s
    timeout server   30s
    timeout check    5s

# --------------------------------------------------------------
# Stats page สำหรับ monitor สถานะ backend ทั้งหมด
# --------------------------------------------------------------
listen stats
    mode http
    bind *:7000
    stats enable
    stats uri /
    stats refresh 5s
    stats auth admin:changeme

# --------------------------------------------------------------
# WRITE endpoint: ชี้ไปที่ Primary เท่านั้น (ตัวเดียวที่ active)
# ใช้ Patroni REST API /primary เป็น health check
# --------------------------------------------------------------
listen write_pool
    bind *:5000
    option httpchk GET /primary
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions

    server pg-node1 pg-node1.internal:5432 maxconn 100 check port 8008
    server pg-node2 pg-node2.internal:5432 maxconn 100 check port 8008
    server pg-node3 pg-node3.internal:5432 maxconn 100 check port 8008

# --------------------------------------------------------------
# READ endpoint: กระจายไปยัง node ที่เป็น replica ที่ healthy
# ใช้ Patroni REST API /replica เป็น health check
# balance leastconn = เลือก node ที่มี connection น้อยที่สุด (Step 666)
# --------------------------------------------------------------
listen read_pool
    bind *:5001
    balance leastconn
    option httpchk GET /replica
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions

    server pg-node1 pg-node1.internal:5432 maxconn 200 check port 8008
    server pg-node2 pg-node2.internal:5432 maxconn 200 check port 8008
    server pg-node3 pg-node3.internal:5432 maxconn 200 check port 8008
```

**อธิบายส่วนสำคัญของไฟล์:**

| Directive | ความหมาย |
|---|---|
| `mode tcp` | HAProxy ทำงานระดับ TCP ไม่แกะ PostgreSQL protocol (เร็วและง่าย) |
| `option httpchk GET /primary` | ใช้ HTTP health check ยิงไปที่ Patroni REST API เพื่อถาม role |
| `http-check expect status 200` | ถ้า Patroni ตอบ 200 = ตรง role ที่ต้องการ, 503 = ไม่ตรง → เอาออกจาก pool |
| `check port 8008` | health check ยิงไป port 8008 (Patroni REST API) แต่ traffic จริงส่งไป port 5432 (PostgreSQL) — นี่คือเทคนิคสำคัญ "check on different port" |
| `balance leastconn` | algorithm สำหรับกระจาย load (รายละเอียด Step 666) |
| `inter 3s fall 3 rise 2` | เช็คทุก 3 วินาที, ถือว่า down หลัง fail ติดกัน 3 ครั้ง, ถือว่า up อีกครั้งหลัง pass ติดกัน 2 ครั้ง |
| `on-marked-down shutdown-sessions` | เมื่อ server ถูก mark down ให้ตัด session ที่ค้างอยู่ทันที (สำคัญมากตอน failover) |

### 664.4 ข้อสังเกตสำคัญ: ทำไมทั้ง write_pool และ read_pool ใส่ node ครบทั้ง 3 ตัว

เพราะ **role ของแต่ละ node เปลี่ยนแปลงได้ตลอดเวลา** (เช่น node2 อาจเป็น primary วันนี้ แต่พรุ่งนี้ failover ไป node3) การใส่ node ทั้งหมดในทั้งสอง pool แล้วให้ health check เป็นตัวกรองว่า "ใครคือ primary ตอนนี้" และ "ใครคือ replica ที่ healthy ตอนนี้" คือแนวทางที่ถูกต้องและทนต่อ failover ได้ดีที่สุด — HAProxy จะไม่ hardcode ว่า node ไหนคือ primary แบบตายตัว

### 664.5 ผลลัพธ์ที่แอปพลิเคชันเห็น

```python
# แอปไม่ต้องรู้เรื่อง topology เลย แค่ชี้ไป 2 endpoint
PRIMARY_DSN = "host=haproxy.internal port=5000 dbname=shop"  # write
REPLICA_DSN = "host=haproxy.internal port=5001 dbname=shop"  # read
```

### 664.6 ข้อดี-ข้อเสียของ Proxy-level Routing

| ด้าน | ข้อดี | ข้อเสีย |
|---|---|---|
| Centralization | routing logic รวมศูนย์ จุดเดียว ทุก service/ภาษาใช้ endpoint เดียวกัน | HAProxy กลายเป็น component สำคัญที่ต้อง HA เองด้วย (มักคู่กับ keepalived/VIP) |
| Latency | — | มี network hop เพิ่ม 1 ชั้น (แม้จะเล็กน้อยมาก ระดับ sub-millisecond) |
| Deployment | เพิ่ม/ลด replica โดยไม่ต้อง deploy โค้ดแอปใหม่ แค่แก้ config HAProxy | ต้องดูแล config เพิ่ม และต้องมีความรู้เฉพาะทาง |
| Query-level awareness | — | HAProxy ทำงานที่ TCP layer ไม่รู้ว่า query ไหนเป็น SELECT FOR UPDATE (ต้องแยก endpoint เองในแอป) |
| Health Check | ทำ health check ได้ centralize และ consistent | ต้องพึ่งพา external health check mechanism (เช่น Patroni) |

---

## Step 665: Replication Lag กับ Read-after-write Consistency

### 665.1 Replication Lag คืออะไร

**Replication Lag** คือ "ช่องว่างเวลา" (หรือ "ช่องว่างของข้อมูล") ระหว่างจุดที่ transaction commit สำเร็จที่ Primary กับจุดที่ WAL record นั้นถูก apply เสร็จที่ Replica

```
เวลา ──────────────────────────────────────────────────▶

Primary:    [COMMIT tx#100]
                  │
                  │  WAL ถูกส่งผ่าน network
                  │  (network latency)
                  ▼
                        [WAL received]
                              │
                              │  Replica apply WAL
                              │  (replay latency, ขึ้นกับ I/O,
                              │   CPU, query ที่รันอยู่)
                              ▼
                                    [tx#100 visible on replica]

              ◄────────── Replication Lag ──────────►
              (ช่วงเวลาที่ replica "ยังไม่เห็น" tx#100)
```

สาเหตุของ lag มีหลายอย่าง:

1. **Network latency** — โดยเฉพาะถ้า replica อยู่คนละ region/data center
2. **I/O bottleneck บน replica** — disk เขียน WAL ช้า
3. **Long-running query บน replica** — ถ้ามี query ที่ทำงานนาน และตั้งค่า `max_standby_streaming_delay` สูง WAL replay อาจถูก delay รอ query จบก่อน
4. **CPU contention** — replica ทำงานหนักจาก read traffic จนไม่มีแรงเหลือ apply WAL ทัน
5. **Heavy write burst บน Primary** — ถ้า Primary มี bulk load หรือ batch job ขนาดใหญ่ WAL จะถูกสร้างเร็วกว่าที่ replica apply ทัน

### 665.2 วิธีวัด Replication Lag

```sql
-- รันที่ Primary: ดูว่า replica แต่ละตัว lag เท่าไหร่ (เป็น byte)
SELECT
    client_addr,
    application_name,
    state,
    pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) AS lag_bytes,
    replay_lag
FROM pg_stat_replication;

--  client_addr  | application_name | state     | lag_bytes | replay_lag
-- --------------+-------------------+-----------+-----------+-------------
--  10.0.1.11    | replica-1         | streaming |    16384  | 00:00:00.12
--  10.0.1.12    | replica-2         | streaming |   131072  | 00:00:01.8
--  10.0.1.13    | replica-3         | streaming |         0 | 00:00:00

-- รันที่ Replica: ดู lag แบบเป็นเวลา (วินาที) จากมุมมองตัวเอง
SELECT
    now() - pg_last_xact_replay_timestamp() AS lag_interval,
    pg_is_in_recovery() AS is_replica,
    pg_last_wal_receive_lsn() AS received_lsn,
    pg_last_wal_replay_lsn() AS replayed_lsn;
```

### 665.3 Read-after-write Consistency Problem

นี่คือปัญหาคลาสสิกที่เกิดจาก replication lag: **ผู้ใช้เขียนข้อมูลสำเร็จ (ไปที่ Primary) แล้วรีเฟรชหน้าเว็บทันที (query ไปที่ Replica) แต่กลับไม่เห็นข้อมูลที่เพิ่งเขียนไป** เพราะ replica ยัง apply WAL ไม่ทัน

```
Timeline ตัวอย่าง: ผู้ใช้สั่งซื้อสินค้า

T=0ms    ผู้ใช้กด "ยืนยันคำสั่งซื้อ"
         → Application: INSERT INTO orders (...) VALUES (...)  [ไป Primary]
T=5ms    Primary COMMIT สำเร็จ, ตอบกลับแอปว่า "order created, id=12345"
T=6ms    แอป redirect ผู้ใช้ไปหน้า "รายละเอียดคำสั่งซื้อ"
T=8ms    หน้าใหม่ query: SELECT * FROM orders WHERE id = 12345  [ไป Replica]
T=8ms    Replica ยัง apply WAL ของ tx นี้ไม่เสร็จ (lag ~50ms)
         → ผลลัพธ์: query ไม่พบ order 12345 !!  (ผู้ใช้เห็นหน้า error/404)
T=58ms   Replica apply WAL เสร็จ ข้อมูลพร้อมแล้ว (สายไปแล้ว)
```

ปัญหานี้สร้างความสับสนและลดความน่าเชื่อถือของระบบอย่างมาก โดยเฉพาะ flow ที่ critical เช่น checkout, การโอนเงิน, การสมัครสมาชิก

### 665.4 แนวทางแก้ไข

#### แนวทางที่ 1: Read-your-writes ผ่าน "Sticky Session ชั่วคราว" (บังคับอ่านจาก Primary หลังเขียน)

วิธีที่ง่ายที่สุดคือ หลังจากเขียนข้อมูลสำเร็จ ให้บังคับอ่านข้อมูลที่เกี่ยวข้องกับ transaction นั้นจาก **Primary** แทนที่จะไป Replica ในช่วงเวลาสั้น ๆ

```python
# Pseudocode: read-your-writes ด้วยการ track "last write timestamp" ต่อ user session
class SessionRouter:
    def __init__(self):
        self.last_write_at = {}  # session_id -> timestamp
        self.STICKY_WINDOW_MS = 2000  # 2 วินาทีหลังเขียน บังคับอ่าน primary

    def mark_write(self, session_id):
        self.last_write_at[session_id] = now_ms()

    def choose_connection(self, session_id, is_read_query):
        if not is_read_query:
            self.mark_write(session_id)
            return primary_pool

        last_write = self.last_write_at.get(session_id)
        if last_write and (now_ms() - last_write) < self.STICKY_WINDOW_MS:
            # ยังอยู่ในช่วง sticky window -> อ่านจาก Primary เพื่อความชัวร์
            return primary_pool

        return replica_pool


router = SessionRouter()

def create_order(session_id, user_id, items):
    with router.choose_connection(session_id, is_read_query=False) as conn:
        # ... INSERT order ...
        pass

def get_order_confirmation(session_id, order_id):
    # จะถูก route ไป Primary อัตโนมัติถ้าเพิ่งมีการเขียนใน session นี้ไม่เกิน 2 วินาที
    with router.choose_connection(session_id, is_read_query=True) as conn:
        cur = conn.cursor()
        cur.execute("SELECT * FROM orders WHERE id = %s", (order_id,))
        return cur.fetchone()
```

#### แนวทางที่ 2: LSN-based Consistency Check (แม่นยำกว่า)

แทนที่จะใช้ "เวลา" เป็นตัวตัดสิน (ซึ่งเป็นการเดาสุ่ม) เราสามารถใช้ **LSN (Log Sequence Number)** ซึ่งเป็นตำแหน่งที่แน่นอนใน WAL เพื่อตรวจสอบว่า replica "ตามทัน" transaction ที่เพิ่งเขียนไปแล้วจริงหรือยัง

```python
# หลังเขียนที่ Primary ให้ดึง LSN ปัจจุบันกลับมา
def create_order(user_id, items):
    with primary_pool.acquire() as conn:
        cur = conn.cursor()
        cur.execute("INSERT INTO orders (...) VALUES (...) RETURNING id")
        order_id = cur.fetchone()[0]
        conn.commit()

        # ดึง LSN ของ transaction นี้
        cur.execute("SELECT pg_current_wal_lsn()")
        write_lsn = cur.fetchone()[0]

        return order_id, write_lsn


def get_order_confirmation(order_id, min_lsn=None):
    if min_lsn:
        # เลือก replica ที่ replay_lsn >= min_lsn เท่านั้น (Step 667 จะขยายเรื่องนี้)
        replica = pick_replica_with_lsn_at_least(min_lsn)
    else:
        replica = pick_any_replica()

    with replica.acquire() as conn:
        cur = conn.cursor()
        cur.execute("SELECT * FROM orders WHERE id = %s", (order_id,))
        return cur.fetchone()
```

```sql
-- ฟังก์ชันช่วยตรวจสอบที่ replica ว่า replay ผ่าน LSN เป้าหมายหรือยัง
SELECT pg_last_wal_replay_lsn() >= '0/3000060'::pg_lsn AS caught_up;
```

#### แนวทางที่ 3: Synchronous Replication สำหรับ Critical Path (ใช้ร่วมกับ Part 063)

ถ้าระบบมี flow ที่ critical มาก ๆ (เช่น การเงิน) สามารถตั้งค่า **synchronous replication** สำหรับ replica บางตัวเฉพาะ transaction สำคัญ โดยใช้ `synchronous_commit` และ `synchronous_standby_names` (รายละเอียดเต็มอยู่ใน Part 063) เพื่อการันตีว่า write จะไม่ถือว่า commit จนกว่า replica ที่กำหนดจะ apply เสร็จ — แต่วิธีนี้แลกมาด้วย write latency ที่สูงขึ้น จึงควรใช้เฉพาะจุดที่จำเป็นจริง ๆ

#### แนวทางที่ 4: Cache-aside pattern เพื่อลดการพึ่งพา Replica ในช่วงวิกฤต

สำหรับหน้าที่ critical เช่น หน้ายืนยันคำสั่งซื้อ อีกวิธีคือ ไม่ query ฐานข้อมูลซ้ำเลย แต่ใช้ข้อมูลที่ได้จากการ INSERT ตอนแรก (RETURNING clause) ส่งตรงไปแสดงผลได้เลย โดยไม่ต้อง query ซ้ำจาก replica

```python
def create_order(user_id, items):
    with primary_pool.acquire() as conn:
        cur = conn.cursor()
        cur.execute(
            """INSERT INTO orders (user_id, status, created_at)
               VALUES (%s, 'pending', now())
               RETURNING id, user_id, status, created_at""",
            (user_id,)
        )
        row = cur.fetchone()
        conn.commit()

        # ส่งข้อมูลที่ได้กลับไปแสดงผลทันที ไม่ต้อง query ซ้ำ (ไม่มีปัญหา lag เลย)
        return {
            "id": row[0],
            "user_id": row[1],
            "status": row[2],
            "created_at": row[3],
        }
```

### 665.5 ตารางสรุปแนวทางแก้ไข

| แนวทาง | ความแม่นยำ | Overhead | ความซับซ้อน | เหมาะกับ |
|---|---|---|---|---|
| Sticky window (time-based) | ปานกลาง (เป็นการเดา) | ต่ำ | ต่ำ | ระบบทั่วไป ที่ไม่ critical มาก |
| LSN-based check | สูงมาก (แม่นยำ 100%) | ปานกลาง (ต้อง query LSN เพิ่ม) | ปานกลาง-สูง | ระบบที่ต้องการความถูกต้องสูง |
| Synchronous replication | สูงสุด (การันตีระดับ transaction) | สูง (write latency เพิ่ม) | สูง | Flow การเงิน/critical เฉพาะจุด |
| Cache-aside (ใช้ผลจาก RETURNING) | สูงสุด (ไม่ query ซ้ำเลย) | ต่ำที่สุด | ต่ำ | หน้า confirmation ทันทีหลังเขียน |

---

## Step 666: Load Balancing Algorithm — Round-robin, Least-connections

เมื่อมี replica หลายตัว เราต้องมีวิธีเลือกว่าจะส่ง query ไปที่ตัวไหน วิธีเลือกนี้เรียกว่า **Load Balancing Algorithm**

### 666.1 Round-robin

**หลักการ**: กระจาย request ไปยัง server แต่ละตัวเรียงตามลำดับวนไปเรื่อย ๆ (server 1 → server 2 → server 3 → server 1 → ...)

```
Request:   1    2    3    4    5    6    7    8    9
              │    │    │    │    │    │    │    │    │
              ▼    ▼    ▼    ▼    ▼    ▼    ▼    ▼    ▼
Replica-1:  [R1]           [R4]           [R7]
Replica-2:       [R2]           [R5]           [R8]
Replica-3:            [R3]           [R6]           [R9]
```

```cfg
# haproxy.cfg: round-robin
listen read_pool
    bind *:5001
    balance roundrobin
    server pg-node1 pg-node1.internal:5432 check port 8008
    server pg-node2 pg-node2.internal:5432 check port 8008
    server pg-node3 pg-node3.internal:5432 check port 8008
```

**ข้อดี**: ง่าย เข้าใจง่าย กระจายเท่า ๆ กันตามจำนวน request
**ข้อเสีย**: ไม่คำนึงถึง "ภาระงานจริง" ของแต่ละ server — ถ้า query บาง request ใช้เวลานานกว่า (เช่น report query หนัก ๆ) round-robin จะยังส่ง request ใหม่ไปที่ server เดิมโดยไม่สนใจว่ามันกำลังยุ่งอยู่

### 666.2 Least-connections

**หลักการ**: ส่ง request ไปยัง server ที่มีจำนวน connection ที่ active อยู่ตอนนี้ **น้อยที่สุด**

```
สถานะปัจจุบัน:
  Replica-1: 15 active connections  (กำลังรัน report query หนัก)
  Replica-2: 3  active connections
  Replica-3: 8  active connections

Request ใหม่เข้ามา → ไปที่ Replica-2 (น้อยที่สุด)
```

```cfg
# haproxy.cfg: least connections
listen read_pool
    bind *:5001
    balance leastconn
    server pg-node1 pg-node1.internal:5432 maxconn 200 check port 8008
    server pg-node2 pg-node2.internal:5432 maxconn 200 check port 8008
    server pg-node3 pg-node3.internal:5432 maxconn 200 check port 8008
```

**ข้อดี**: ปรับตัวตามภาระงานจริง เหมาะกับ workload ที่ query แต่ละตัวใช้เวลาต่างกันมาก (เช่น mix ของ simple lookup กับ heavy analytical query)
**ข้อเสีย**: ซับซ้อนกว่าเล็กน้อย ต้อง track จำนวน connection ตลอดเวลา

### 666.3 Algorithm อื่น ๆ ที่ควรรู้จัก (เสริม)

| Algorithm | หลักการ | เหมาะกับ |
|---|---|---|
| **Weighted Round-robin** | เหมือน round-robin แต่กำหนดน้ำหนักให้ server แรงกว่ารับ request มากกว่า | replica ที่มี hardware spec ต่างกัน |
| **Random** | สุ่มเลือก server | ระบบขนาดใหญ่มาก ที่ random ให้ผลลัพธ์กระจายตัวใกล้เคียง round-robin แต่คำนวณเร็วกว่า |
| **Source/IP Hash** | hash จาก client IP เพื่อให้ client เดิมไปที่ server เดิมเสมอ (sticky) | เมื่อต้องการ session affinity |

```cfg
# ตัวอย่าง weighted round-robin: node1 แรงกว่า รับ traffic 2 เท่าของ node2/node3
listen read_pool
    bind *:5001
    balance roundrobin
    server pg-node1 pg-node1.internal:5432 weight 20 check port 8008
    server pg-node2 pg-node2.internal:5432 weight 10 check port 8008
    server pg-node3 pg-node3.internal:5432 weight 10 check port 8008
```

### 666.4 คำแนะนำในการเลือก Algorithm

```
┌─────────────────────────────────────────────────────────────┐
│  เลือก Algorithm อย่างไร?                                     │
│                                                                  │
│  Query workload สม่ำเสมอ (เวลาใกล้เคียงกันทุก query)             │
│      → Round-robin ก็เพียงพอ (เรียบง่าย overhead ต่ำ)            │
│                                                                  │
│  Query workload หลากหลาย (mix simple lookup + heavy report)     │
│      → Least-connections (ปรับตัวตามภาระจริง)                   │
│                                                                  │
│  Replica hardware ไม่เท่ากัน (บางตัวแรงกว่า)                     │
│      → Weighted Round-robin หรือ Weighted Least-connections     │
│                                                                  │
│  ต้องการ session affinity (เช่น caching ที่ session)             │
│      → Source/IP Hash                                           │
└─────────────────────────────────────────────────────────────┘
```

ในทางปฏิบัติ **`leastconn` เป็นตัวเลือกที่ปลอดภัยและใช้กันแพร่หลายที่สุดสำหรับ PostgreSQL read pool** เพราะ workload ของฐานข้อมูลมักไม่สม่ำเสมอ (query บางตัวเร็ว บางตัวช้า) ทำให้ least-connections กระจายภาระได้ดีกว่า round-robin ในสถานการณ์จริงส่วนใหญ่

---

## Step 667: Health Check — ตรวจสอบ Healthy และ Lag ก่อน Route Traffic

### 667.1 ทำไม Health Check ต้องมากกว่าแค่ "ตอบสนองไหม"

Health check แบบพื้นฐานที่สุดคือการเช็คว่า server "ตอบสนอง" (alive) หรือไม่ ผ่าน TCP connect หรือ `pg_isready` แต่สำหรับ Read Replica เราต้องเช็คมากกว่านั้น 2 มิติ:

1. **Availability** — instance ทำงานอยู่ไหม, accept connection ได้ไหม
2. **Role correctness** — เป็น replica จริง (ไม่ใช่ดันไปกลายเป็น primary หลัง failover)
3. **Replication lag ยอมรับได้** — lag ไม่เกิน threshold ที่ตั้งไว้ (เช่น ไม่เกิน 10 วินาที หรือไม่เกิน 100MB ของ WAL ที่ยังไม่ apply)

```
┌───────────────────────────────────────────────────────────────┐
│              Health Check Decision Flow                          │
│                                                                     │
│    Replica Node                                                   │
│         │                                                          │
│         ▼                                                          │
│  ┌──────────────────┐                                             │
│  │ 1. TCP/pg_isready │  FAIL → เอาออกจาก pool ทันที                │
│  │    Alive?          │                                             │
│  └─────────┬──────────┘                                            │
│           PASS                                                     │
│             ▼                                                      │
│  ┌──────────────────┐                                             │
│  │ 2. pg_is_in_recovery│ FALSE → กลายเป็น Primary แล้ว! เอาออกจาก   │
│  │    = true?         │          read pool ทันที                   │
│  └─────────┬──────────┘                                            │
│           TRUE                                                     │
│             ▼                                                      │
│  ┌──────────────────┐                                             │
│  │ 3. Replication lag │  เกิน threshold → เอาออกจาก pool ชั่วคราว   │
│  │    < threshold?    │  (จนกว่า lag จะกลับมาปกติ)                  │
│  └─────────┬──────────┘                                            │
│           PASS                                                     │
│             ▼                                                      │
│       ✅ Healthy → รับ traffic ได้                                  │
└───────────────────────────────────────────────────────────────┘
```

### 667.2 การ implement Health Check Script

วิธีที่นิยมใช้กับ HAProxy คือเขียน HTTP health check endpoint ด้วยเครื่องมือเล็ก ๆ (เช่น `xinetd`, หรือ script ที่รันผ่าน web server เบา ๆ) ที่คอย query สถานะ PostgreSQL แล้วตอบ HTTP status code กลับ

```bash
#!/bin/bash
# /usr/local/bin/pg_replica_healthcheck.sh
# สคริปต์นี้จะถูกเรียกผ่าน xinetd ที่ port 8009 (ตัวอย่าง)

LAG_THRESHOLD_SECONDS=10
PGHOST="localhost"
PGPORT=5432
PGUSER="healthcheck"
PGDATABASE="shop"

# 1. เช็คว่า instance ตอบสนองไหม
IS_RECOVERY=$(psql -h $PGHOST -p $PGPORT -U $PGUSER -d $PGDATABASE -t -A \
    -c "SELECT pg_is_in_recovery();" 2>/dev/null)

if [ -z "$IS_RECOVERY" ]; then
    # query ไม่สำเร็จ = instance ไม่ตอบสนอง
    echo -e "HTTP/1.1 503 Service Unavailable\r\n\r\nDOWN"
    exit 0
fi

if [ "$IS_RECOVERY" != "t" ]; then
    # ไม่ใช่ replica (อาจกลายเป็น primary แล้วหลัง failover)
    echo -e "HTTP/1.1 503 Service Unavailable\r\n\r\nNOT_A_REPLICA"
    exit 0
fi

# 2. เช็ค replication lag เป็นวินาที
LAG_SECONDS=$(psql -h $PGHOST -p $PGPORT -U $PGUSER -d $PGDATABASE -t -A \
    -c "SELECT COALESCE(EXTRACT(EPOCH FROM (now() - pg_last_xact_replay_timestamp())), 0);" \
    2>/dev/null)

LAG_INT=${LAG_SECONDS%.*}

if [ -z "$LAG_INT" ]; then
    echo -e "HTTP/1.1 503 Service Unavailable\r\n\r\nLAG_CHECK_FAILED"
    exit 0
fi

if [ "$LAG_INT" -gt "$LAG_THRESHOLD_SECONDS" ]; then
    # lag เกิน threshold -> ไม่ควร route traffic มา
    echo -e "HTTP/1.1 503 Service Unavailable\r\n\r\nLAG_TOO_HIGH:${LAG_INT}s"
    exit 0
fi

# ผ่านทุกเงื่อนไข -> healthy
echo -e "HTTP/1.1 200 OK\r\n\r\nOK:lag=${LAG_INT}s"
exit 0
```

```ini
# /etc/xinetd.d/pg_replica_healthcheck
service pg_replica_healthcheck
{
    flags           = REUSE
    socket_type     = stream
    port            = 8009
    wait            = no
    user            = postgres
    server          = /usr/local/bin/pg_replica_healthcheck.sh
    log_on_failure  += USERID
    disable         = no
}
```

```cfg
# haproxy.cfg: เช็ค health ผ่าน custom health check script (port 8009)
listen read_pool
    bind *:5001
    balance leastconn
    option httpchk GET /
    http-check expect string OK
    default-server inter 2s fall 2 rise 3

    server pg-node1 pg-node1.internal:5432 maxconn 200 check port 8009
    server pg-node2 pg-node2.internal:5432 maxconn 200 check port 8009
    server pg-node3 pg-node3.internal:5432 maxconn 200 check port 8009
```

### 667.3 การใช้ Patroni REST API แทน (แนวทางที่แนะนำเมื่อมี Patroni อยู่แล้ว)

ถ้าใช้ Patroni จัดการ cluster (Part 065) Patroni มี endpoint สำเร็จรูปที่รองรับการเช็ค lag ในตัวอยู่แล้ว ทำให้ไม่ต้องเขียน script เอง:

```
GET /replica?lag=10000000   → HTTP 200 ถ้าเป็น replica ที่ lag ไม่เกิน 10MB (10,000,000 bytes)
                               HTTP 503 ถ้าไม่เข้าเงื่อนไข (lag เกิน หรือไม่ใช่ replica)
```

```cfg
listen read_pool
    bind *:5001
    balance leastconn
    option httpchk GET /replica?lag=10000000
    http-check expect status 200
    default-server inter 3s fall 3 rise 2

    server pg-node1 pg-node1.internal:5432 maxconn 200 check port 8008
    server pg-node2 pg-node2.internal:5432 maxconn 200 check port 8008
    server pg-node3 pg-node3.internal:5432 maxconn 200 check port 8008
```

> การใช้ Patroni REST API เป็นแนวทางที่ **แนะนำที่สุด** ในระบบจริง เพราะลดความซับซ้อนของการดูแล custom script เอง และ Patroni ก็เป็นแหล่งข้อมูลที่ถูกต้องที่สุด (source of truth) เกี่ยวกับสถานะ cluster อยู่แล้ว

### 667.4 การตั้งค่า Threshold ที่เหมาะสม

การตั้งค่า lag threshold ต้องสมดุลระหว่าง 2 ด้าน:

```
Threshold ต่ำเกินไป (เช่น 1 วินาที)
  → replica หลุดจาก pool บ่อย แม้จะเป็นแค่ lag ชั่วครู่จาก network jitter
  → เหลือ replica น้อยตัวรับ traffic → เสี่ยง overload ตัวที่เหลือ

Threshold สูงเกินไป (เช่น 60 วินาที)
  → ผู้ใช้เห็นข้อมูลเก่ามาก ๆ ได้ (เสี่ยงเรื่อง consistency)
  → ประสบการณ์ผู้ใช้แย่ลง

แนวทางที่นิยม: 5-15 วินาที สำหรับ general read traffic
              (ปรับตาม SLA ของแต่ละระบบ)
```

### 667.5 Circuit Breaker Pattern เสริม

นอกจาก Health Check พื้นฐาน ระบบระดับ production มักเพิ่ม **Circuit Breaker** ที่ระดับแอปพลิเคชันด้วย เพื่อป้องกันไม่ให้ retry ซ้ำ ๆ ไปยัง replica ที่กำลังมีปัญหาจนกระทบ performance โดยรวม

```python
# Pseudocode: circuit breaker แบบง่าย
class ReplicaCircuitBreaker:
    def __init__(self, failure_threshold=5, reset_timeout_sec=30):
        self.failure_count = {}
        self.opened_at = {}
        self.failure_threshold = failure_threshold
        self.reset_timeout_sec = reset_timeout_sec

    def is_available(self, replica_id):
        if replica_id in self.opened_at:
            if now_sec() - self.opened_at[replica_id] > self.reset_timeout_sec:
                # ลองเปิดใหม่ (half-open state)
                del self.opened_at[replica_id]
                self.failure_count[replica_id] = 0
                return True
            return False
        return True

    def record_failure(self, replica_id):
        self.failure_count[replica_id] = self.failure_count.get(replica_id, 0) + 1
        if self.failure_count[replica_id] >= self.failure_threshold:
            self.opened_at[replica_id] = now_sec()

    def record_success(self, replica_id):
        self.failure_count[replica_id] = 0
```

---

## Step 668: Failover ผลกระทบต่อ Load Balancer

### 668.1 สิ่งที่เกิดขึ้นเมื่อ Failover

เมื่อ Primary ล้มเหลว (ไม่ว่าจะจาก hardware failure, network partition, หรือ planned maintenance) กลไก Patroni ที่เราเรียนใน Part 065 จะทำการ **promote** replica ตัวหนึ่งขึ้นเป็น Primary ใหม่โดยอัตโนมัติ ซึ่งเปลี่ยน topology ของ cluster ทันที — Load Balancer จำเป็นต้อง "รู้" การเปลี่ยนแปลงนี้และปรับ routing table ให้ทัน ไม่เช่นนั้นจะเกิดปัญหาร้ายแรง เช่น ส่ง write query ไปยัง node ที่ (เดิมเคยเป็น primary แต่ตอนนี้กลายเป็น) replica แล้ว ซึ่งจะ error ทันที

```
┌─────────────────────────────────────────────────────────────────┐
│                    Failover Timeline                               │
│                                                                       │
│  T=0    Primary (node1) ล่ม                                         │
│         │                                                            │
│         ▼                                                            │
│  T=0+ε  Patroni ตรวจพบ (ผ่าน DCS: etcd/Consul heartbeat หมดอายุ)       │
│         │                                                            │
│         ▼                                                            │
│  T=few sec  Patroni เลือก replica ที่ WAL ใหม่สุด (node2) แล้ว        │
│              promote เป็น Primary ใหม่                                │
│         │                                                            │
│         ▼                                                            │
│  T=+     node2 REST API /primary เริ่มตอบ 200                        │
│              node2 REST API /replica เริ่มตอบ 503                    │
│              node1 (ถ้าฟื้นมา) จะถูกตั้งเป็น replica ของ node2         │
│         │                                                            │
│         ▼                                                            │
│  HAProxy health check รอบถัดไป (ภายใน inter interval)                │
│         │                                                            │
│         ▼                                                            │
│  HAProxy อัปเดต routing table:                                       │
│    - write_pool: node1 ออก (down), node2 เข้า (primary ใหม่)          │
│    - read_pool: node2 ออก (ไม่ใช่ replica แล้ว), node1 เข้าเมื่อฟื้น    │
└─────────────────────────────────────────────────────────────────┘
```

### 668.2 ช่วงเวลาวิกฤต (Detection + Convergence Window)

ระยะเวลารวมตั้งแต่ Primary ล่มจนถึง Load Balancer ปรับ routing ให้ถูกต้องสมบูรณ์ ประกอบด้วยหลายช่วง:

```
Total Failover Impact Time =
    Patroni Detection Time         (ขึ้นกับ ttl ของ DCS lock, ปกติ 10-30s)
  + Patroni Promotion Time         (ปกติไม่กี่วินาที)
  + HAProxy Health Check Interval  (ตาม inter + fall/rise ที่ตั้งไว้)
  + Client Reconnect/Retry Time    (ขึ้นกับ connection pool/driver)
```

ตัวอย่างการคำนวณจากค่า config ที่เราตั้งไว้ใน Step 664 (`inter 3s fall 3 rise 2`):

```
เวลาที่ HAProxy จะ mark server ว่า down (worst case):
   inter (3s) × fall (3) = 9 วินาที

เวลาที่ HAProxy จะ mark server ใหม่ว่า up (worst case):
   inter (3s) × rise (2) = 6 วินาที

รวมกับ Patroni detection+promotion (~10-15s โดยทั่วไป)
   → total downtime ที่แอปพลิเคชันอาจเห็น: ~20-30 วินาที
```

> การลด `inter` ให้เร็วขึ้น (เช่น 1s) จะช่วยลด detection time แต่ต้องระวัง false positive จาก network blip ชั่วคราว — ต้อง balance ระหว่างความเร็วในการตรวจจับกับความเสถียร (ไม่ flap บ่อยเกินไป)

### 668.3 สิ่งที่แอปพลิเคชันต้องรองรับระหว่าง Failover

```python
# Pseudocode: retry logic ที่แอปควรมีเพื่อรองรับ failover
import time
import psycopg2

def execute_with_retry(pool, query, params=None, max_retries=3, backoff_base=0.5):
    last_error = None
    for attempt in range(max_retries):
        try:
            with pool.acquire() as conn:
                cur = conn.cursor()
                cur.execute(query, params)
                if not query.strip().upper().startswith("SELECT"):
                    conn.commit()
                return cur.fetchall() if cur.description else None
        except (psycopg2.OperationalError, psycopg2.InterfaceError) as e:
            # เกิดขึ้นได้ระหว่าง failover: connection ถูกตัด, server ปฏิเสธ
            last_error = e
            wait_time = backoff_base * (2 ** attempt)  # exponential backoff
            time.sleep(wait_time)
            continue
        except psycopg2.errors.ReadOnlySqlTransaction:
            # เขียนไปโดนโหนดที่เพิ่งกลายเป็น replica (edge case ระหว่าง failover)
            last_error = "Attempted write on read-only replica during failover"
            wait_time = backoff_base * (2 ** attempt)
            time.sleep(wait_time)
            continue

    raise Exception(f"Query failed after {max_retries} retries: {last_error}")
```

**หลักการสำคัญที่ควร implement ฝั่งแอป:**

1. **Retry with exponential backoff** — เมื่อเจอ connection error ให้ retry แบบเว้นระยะเวลาเพิ่มขึ้นเรื่อย ๆ ไม่ retry ถี่จนซ้ำเติมปัญหา
2. **Idempotent write design** — ออกแบบ write operation ให้ retry ซ้ำได้อย่างปลอดภัย (เช่น ใช้ unique constraint หรือ idempotency key) เพราะอาจเกิดกรณี network timeout แต่จริง ๆ Primary เขียนสำเร็จแล้ว
3. **Circuit breaker** — ถ้า retry ล้มเหลวติดต่อกันหลายครั้ง ให้ "พัก" ไม่ยิง request รัว ๆ ระยะหนึ่ง (ตามที่แนะนำใน Step 667.5)
4. **Connection pool ต้องรองรับ invalidate** — เมื่อ HAProxy เปลี่ยน routing แล้ว connection เก่าที่ pool ถืออยู่อาจชี้ไป node ผิด ต้อง validate/refresh connection ก่อนใช้ (PgBouncer ช่วยเรื่องนี้ได้ส่วนหนึ่ง ตาม Part 066)

### 668.4 การเชื่อมโยง Patroni + HAProxy อย่างสมบูรณ์ (ทบทวนจาก Part 065)

```
┌───────────────────────────────────────────────────────────────────┐
│              Full HA + Load Balancing Stack (Part 065 + 067)         │
│                                                                         │
│   ┌──────────────┐                                                    │
│   │ Application   │                                                    │
│   └──────┬───────┘                                                     │
│          ▼                                                             │
│   ┌──────────────┐    health check via REST API (port 8008)           │
│   │  HAProxy      │◄──────────────────────────────────┐                │
│   └──────┬───────┘                                     │                │
│          │                                              │                │
│   ┌──────┴──────┬─────────────┐                        │                │
│   ▼              ▼             ▼                        │                │
│ ┌────────┐  ┌────────┐   ┌────────┐                     │                │
│ │Patroni  │  │Patroni  │   │Patroni  │  ← REST API :8008 (health/role)   │
│ │+PG node1│  │+PG node2│   │+PG node3│                                   │
│ └────┬───┘  └────┬───┘   └────┬───┘                     │                │
│      │           │             │                         │                │
│      └───────────┴─────────────┴──────── DCS (etcd/Consul/ZK) ──────────┘│
│                  (เก็บ leader lock, cluster state, config)               │
└───────────────────────────────────────────────────────────────────┘
```

จุดสำคัญคือ **DCS (Distributed Configuration Store เช่น etcd)** เป็น source of truth ที่แท้จริงของ topology — Patroni บน node ต่าง ๆ จะแย่งชิง leader lock ผ่าน DCS, และ REST API ของแต่ละ Patroni node จะสะท้อนสถานะปัจจุบันตาม DCS นั้น HAProxy เพียงแค่ "อ่าน" สถานะผ่าน REST API เป็นระยะ ๆ (polling) จึงไม่มี single point of failure ที่ HAProxy เอง (ตราบใดที่ deploy HAProxy แบบ HA คู่กับ keepalived/VIP ตามที่แนะนำใน Part 065)

---

## Step 669: Geographic/Multi-region Considerations เบื้องต้น

### 669.1 ปัญหา Latency ข้าม Region

เมื่อระบบขยายตัวไปให้บริการผู้ใช้ในหลายภูมิภาค (เช่น เอเชีย, ยุโรป, อเมริกา) การมี Primary อยู่ที่เดียว (เช่น สิงคโปร์) แต่ replica กระจายไปแต่ละภูมิภาค จะช่วยลด **read latency** ได้มาก เพราะผู้ใช้ในแต่ละภูมิภาคสามารถอ่านข้อมูลจาก replica ที่อยู่ใกล้ตัวเองที่สุด แทนที่จะต้องยิง query ข้ามทวีปไปหา Primary ทุกครั้ง

```
┌────────────────────────────────────────────────────────────────────┐
│              Multi-region Read Replica Architecture                   │
│                                                                          │
│   Region: Asia (Singapore)          Region: Europe (Frankfurt)          │
│   ┌──────────────┐                  ┌──────────────┐                   │
│   │ App (Asia)    │                  │ App (Europe)  │                   │
│   └──────┬───────┘                  └──────┬───────┘                   │
│          │ read: ~1-3ms                     │ read: ~1-3ms               │
│          ▼                                   ▼                           │
│   ┌──────────────┐                  ┌──────────────┐                   │
│   │ Replica (SG)  │                  │ Replica (FRA) │                   │
│   └──────┬───────┘                  └──────┬───────┘                   │
│          │                                   │                           │
│          │        WAL streaming (~150-250ms cross-region)                │
│          └───────────────┬───────────────────┘                          │
│                           ▼                                              │
│                  ┌──────────────────┐                                    │
│                  │  Primary (SG)     │  ← write ทุกภูมิภาคยังต้องมาที่นี่    │
│                  │  (Region: Asia)   │                                    │
│                  └──────────────────┘                                    │
│                                                                          │
│   App (Europe) เขียนข้อมูล → ต้องข้าม region ไปที่ Primary (SG)          │
│   → write latency สูง (~150-250ms) แต่ read latency ต่ำ (local replica)  │
└────────────────────────────────────────────────────────────────────┘
```

### 669.2 Trade-off สำคัญ

| ด้าน | ผลกระทบ |
|---|---|
| **Read latency** | ลดลงมาก ถ้าผู้ใช้อ่านจาก replica ในภูมิภาคตัวเอง (จาก 150-250ms เหลือ 1-5ms) |
| **Write latency** | ยังคงสูง เพราะทุก write ต้องวิ่งไปหา Primary ที่อยู่ภูมิภาคเดียว (ไม่เปลี่ยนแปลงจากเดิม) |
| **Replication lag ข้าม region** | สูงกว่า replica ใน region เดียวกันมาก เพราะ network latency ระหว่างทวีป (มักหลักร้อย ms) |
| **Cost** | ค่า data transfer ข้าม region ของ cloud provider (egress cost) เพิ่มขึ้นตามปริมาณ WAL ที่ stream ข้ามไป |
| **Consistency** | ปัญหา read-after-write consistency (Step 665) จะรุนแรงขึ้น เพราะ lag ข้าม region สูงกว่าเดิมมาก |

### 669.3 กลยุทธ์การเลือก Replica ที่ใกล้ผู้ใช้ที่สุด (Geo-routing)

การเลือก replica ที่เหมาะสมตามตำแหน่งผู้ใช้ทำได้หลายระดับ:

#### ระดับ DNS (GeoDNS)

```
ผู้ใช้จาก Asia  → resolve read.db.example.com → IP ของ Replica (Singapore)
ผู้ใช้จาก Europe → resolve read.db.example.com → IP ของ Replica (Frankfurt)
```

ใช้บริการ Geo-DNS (เช่น AWS Route53 Geolocation Routing, Cloudflare Load Balancing) เพื่อให้ DNS resolution แตกต่างกันตามตำแหน่งผู้ขอ (source IP) โดยอัตโนมัติ — แอปเชื่อมต่อไปที่ domain name เดียวกันเสมอ แต่ได้ IP ปลายทางต่างกันตามภูมิภาค

#### ระดับ Application (Region-aware Config)

```python
# Pseudocode: เลือก replica pool ตาม region ที่ service deploy อยู่
import os

CURRENT_REGION = os.environ.get("DEPLOY_REGION", "ap-southeast-1")

REGION_REPLICA_MAP = {
    "ap-southeast-1": ["replica-sg-1.internal", "replica-sg-2.internal"],
    "eu-central-1":   ["replica-fra-1.internal", "replica-fra-2.internal"],
    "us-east-1":      ["replica-use1-1.internal", "replica-use1-2.internal"],
}

# fallback: ถ้า region ปัจจุบันไม่มี local replica ให้ใช้ replica ที่ใกล้ที่สุดถัดไป
REGION_FALLBACK = {
    "ap-southeast-1": "eu-central-1",  # เผื่อกรณี local replica ล่มทั้งหมด
}

def get_local_replica_pool():
    replicas = REGION_REPLICA_MAP.get(CURRENT_REGION)
    if not replicas:
        fallback_region = REGION_FALLBACK.get(CURRENT_REGION)
        replicas = REGION_REPLICA_MAP.get(fallback_region, [])
    return create_pool_from_hosts(replicas)
```

#### ระดับ Proxy (Regional HAProxy Deployment)

ในสถาปัตยกรรมระดับ enterprise มักจะ deploy HAProxy instance แยกในแต่ละภูมิภาค โดยแต่ละตัวชี้ไปที่ replica ใน region ตัวเองเป็นหลัก (primary choice) และมี replica ข้าม region เป็น fallback (ถ้า local replica ทั้งหมด down)

```cfg
# haproxy.cfg บน region Europe: ให้ priority replica ใน region เดียวกันก่อน
listen read_pool
    bind *:5001
    balance leastconn

    # local replicas (priority สูง - ใช้ก่อนเสมอถ้า healthy)
    server replica-fra-1 replica-fra-1.internal:5432 maxconn 200 check port 8008
    server replica-fra-2 replica-fra-2.internal:5432 maxconn 200 check port 8008

    # cross-region replica (backup - ใช้เฉพาะเมื่อ local ทั้งหมด down)
    server replica-sg-1 replica-sg-1.internal:5432 maxconn 100 check port 8008 backup
```

> keyword `backup` ใน HAProxy หมายความว่า server ตัวนี้จะไม่รับ traffic เลยตราบใดที่ยังมี server หลัก (non-backup) อย่างน้อย 1 ตัวที่ healthy — เป็นกลไกที่เหมาะมากสำหรับ cross-region fallback

### 669.4 ข้อควรพิจารณาเพิ่มเติม (เบื้องต้น)

- **Compliance/Data Residency**: บางประเทศมีกฎหมายกำหนดว่าข้อมูลบางประเภทต้องเก็บอยู่ในประเทศ (data residency) — ต้องตรวจสอบก่อนว่าการทำ multi-region replica ขัดกับข้อกำหนดทางกฎหมายหรือไม่
- **Clock skew**: เมื่อระบบกระจายข้าม region การพึ่งพา timestamp เพื่อเทียบ consistency (เช่นในแนวทาง sticky window ของ Step 665) ต้องระวังเรื่อง clock synchronization ระหว่าง server (แนะนำใช้ NTP ที่แม่นยำ)
- **Multi-region นี้ยังคงเป็น Single-Primary**: สถาปัตยกรรมที่อธิบายในบทนี้ยังคงมี Primary เดียว (single point สำหรับ write) การทำ **multi-region multi-primary** (เขียนได้จากหลาย region พร้อมกัน) เป็นเรื่องซับซ้อนกว่ามาก ต้องใช้เทคโนโลยีเฉพาะทาง เช่น logical replication แบบ multi-master, conflict resolution, หรือ distributed SQL database (เช่น CockroachDB, YugabyteDB) ซึ่งอยู่นอกขอบเขตของหลักสูตรนี้ — ในระดับ PostgreSQL มาตรฐาน เราเน้นเพียง "multi-region read scaling" เท่านั้น

```
┌──────────────────────────────────────────────────────────┐
│  สรุปแนวคิด Multi-region ในบทนี้ (ขอบเขตที่ครอบคลุม)           │
│                                                               │
│   ✅ Single Primary + Multi-region Read Replicas             │
│      (สิ่งที่บทนี้ครอบคลุม)                                    │
│                                                               │
│   ❌ Multi-region Multi-primary / Active-Active               │
│      (นอกขอบเขต ต้องใช้เทคโนโลยีเฉพาะทาง)                     │
└──────────────────────────────────────────────────────────┘
```

---

## Step 670: แบบฝึกหัดรวม — ออกแบบสถาปัตยกรรม Read Replica + Load Balancing สำหรับ E-commerce

### 670.1 โจทย์

บริษัท "ShopTown" กำลังจะขยายระบบ e-commerce ที่มีลักษณะดังนี้:

- ผู้ใช้งาน ~500,000 คนต่อวัน, peak traffic ช่วงโปรโมชั่นสูงถึง 10 เท่าของค่าเฉลี่ย
- สัดส่วน read/write query ประมาณ 90/10
- มีผู้ใช้งานหลักใน 2 ภูมิภาค: เอเชียตะวันออกเฉียงใต้ (70% ของผู้ใช้) และยุโรป (30% ของผู้ใช้)
- มี flow สำคัญที่ต้องการความถูกต้องสูง (strong consistency): checkout/ชำระเงิน, ตรวจสอบสต็อกสินค้าก่อนขาย
- มี flow ที่ไม่ critical: แสดงสินค้า, ค้นหาสินค้า, ดูรีวิว, แสดง recommendation
- ต้องการระบบที่ทนต่อ failover ได้ โดย downtime ไม่เกิน 30 วินาที
- มี dashboard สำหรับทีม analytics ที่รัน query หนัก ๆ (aggregate ยอดขายรายวัน) ซึ่งไม่ควรกระทบ transactional workload

**ให้ออกแบบสถาปัตยกรรมที่ครอบคลุม:**
1. จำนวนและตำแหน่งของ Primary/Replica
2. กลยุทธ์ Read/Write Splitting และการจัดการ flow ที่ critical
3. Load Balancer และ health check strategy
4. การจัดการ replication lag/consistency
5. แนวทางรองรับ failover

### 670.2 แนวทางคำตอบ (Reference Solution)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                  ShopTown: Full Architecture Design                        │
│                                                                              │
│  Region: ap-southeast-1 (Singapore) - PRIMARY REGION                       │
│  ┌────────────────────────────────────────────────────────────────┐       │
│  │                                                                     │       │
│  │   App Servers (SG)          Analytics Dashboard                    │       │
│  │        │                          │                                 │       │
│  │        ▼                          ▼                                 │       │
│  │   ┌─────────────┐         ┌─────────────┐                          │       │
│  │   │  PgBouncer   │         │  PgBouncer   │  (Part 066)             │       │
│  │   │ (per app tier)│        │ (analytics)  │                         │       │
│  │   └──────┬───────┘         └──────┬──────┘                          │       │
│  │          ▼                         ▼                                 │       │
│  │   ┌─────────────────────────────────────┐                          │       │
│  │   │         HAProxy (SG) - HA pair        │                          │       │
│  │   │   write:5000     read:5001            │                          │       │
│  │   │                  analytics-read:5002  │  ← แยก pool วิเคราะห์      │       │
│  │   └───┬───────────┬────────────┬─────────┘                          │       │
│  │       ▼            ▼            ▼                                    │       │
│  │  ┌─────────┐ ┌──────────┐ ┌──────────┐                              │       │
│  │  │ Primary  │ │ Replica-1 │ │ Replica-2 │  ← general read pool        │       │
│  │  │ (SG)     │ │ (SG)      │ │ (SG)      │                              │       │
│  │  └────┬────┘ └──────────┘ └──────────┘                              │       │
│  │       │                                                               │       │
│  │       │      ┌──────────────┐                                        │       │
│  │       └─────▶│ Replica-3     │  ← dedicated analytics replica          │       │
│  │              │ (SG, isolated)│    (แยกออกจาก general pool             │       │
│  │              └──────────────┘     กันไม่ให้ heavy query กระทบ read ปกติ)│       │
│  │                                                                     │       │
│  └────────────────────────────────────────────────────────────────┘       │
│                              │                                              │
│                              │ Cross-region streaming replication            │
│                              │ (async, ~150-200ms lag typical)               │
│                              ▼                                              │
│  Region: eu-central-1 (Frankfurt) - READ REGION                            │
│  ┌────────────────────────────────────────────────────────────────┐       │
│  │                                                                     │       │
│  │   App Servers (EU)                                                  │       │
│  │        │                                                            │       │
│  │        ▼                                                            │       │
│  │   ┌─────────────┐                                                   │       │
│  │   │  PgBouncer   │                                                   │       │
│  │   └──────┬───────┘                                                  │       │
│  │          ▼                                                          │       │
│  │   ┌─────────────────────────────────────┐                          │       │
│  │   │   HAProxy (EU) - HA pair              │                          │       │
│  │   │   write:5000 (→ forward to SG Primary)│                          │       │
│  │   │   read:5001  (→ local EU replicas)    │                          │       │
│  │   └───┬───────────────────┬───────────────┘                          │       │
│  │       │                    ▼                                         │       │
│  │       │             ┌──────────────┐  ┌──────────────┐               │       │
│  │       │             │ Replica-EU-1  │  │ Replica-EU-2  │               │       │
│  │       │             └──────────────┘  └──────────────┘               │       │
│  │       │                                                               │       │
│  │       └─── write query: ส่งข้าม region ไปที่ Primary (SG) โดยตรง       │       │
│  │            (ยอมรับ write latency ~150-200ms สำหรับ EU users)          │       │
│  └────────────────────────────────────────────────────────────────┘       │
└─────────────────────────────────────────────────────────────────────────┘
```

#### 1) จำนวนและตำแหน่งของ Primary/Replica

- **Primary 1 ตัว** ที่ Singapore (ภูมิภาคที่มีผู้ใช้ 70%) — ยึดหลัก "Primary อยู่ใกล้ผู้ใช้ส่วนใหญ่ที่สุด" เพื่อลด write latency ของ traffic ส่วนใหญ่
- **Replica ที่ SG จำนวน 3 ตัว**: 2 ตัวสำหรับ general read (transactional read เช่น ดูสินค้า, ตะกร้า) และ **1 ตัวแยกสำหรับ analytics dashboard โดยเฉพาะ** (isolate heavy query ไม่ให้กระทบ general read pool — ตอบโจทย์ "dashboard ไม่ควรกระทบ transactional workload")
- **Replica ที่ EU จำนวน 2 ตัว**: สำหรับ local read ของผู้ใช้ยุโรป (ลด read latency จาก ~150ms เหลือหลัก ms) ใช้ Patroni ในโหมด cascading หรือ standby cluster ต่อยอดจาก Primary ที่ SG ตามเทคนิคใน Part 063/065
- ใช้ **synchronous_commit = remote_apply** (หรือ quorum-based) เฉพาะกับ Replica-1/Replica-2 ที่ SG (ไม่รวม EU เพราะ latency สูงเกินจะทำ sync ข้าม region ได้จริง) เพื่อการันตีว่า critical write (checkout) จะไม่ตกหล่นถ้า Primary ล่มกะทันหัน — นี่คือการเชื่อมกับความรู้ synchronous replication จาก Part 063

#### 2) กลยุทธ์ Read/Write Splitting และ Flow ที่ Critical

| Flow | ปลายทาง | เหตุผล |
|---|---|---|
| ดูสินค้า, ค้นหา, ดูรีวิว | Replica (local region) | ไม่ critical ยอมรับ lag ได้ |
| แสดง recommendation | Replica (local region) | ไม่ critical, ยอมรับ lag ได้สูง |
| Checkout/ชำระเงิน | **Primary** เท่านั้น (write + read ที่เกี่ยวข้องในช่วง sticky window) | ต้องการ strong consistency สูงสุด |
| ตรวจสอบสต็อกก่อนขาย | **Primary** เท่านั้น พร้อม `SELECT ... FOR UPDATE` | ป้องกัน overselling — ต้อง lock row ที่ primary |
| Analytics dashboard | Replica-3 (dedicated analytics replica) | ยอมรับ lag ได้สูงมาก (รายงานรายวัน), isolate load |

ใช้แนวทาง **Cache-aside (RETURNING clause)** จาก Step 665.4 สำหรับหน้ายืนยันคำสั่งซื้อทันทีหลัง checkout เพื่อไม่ต้อง query ซ้ำเลย และใช้ **LSN-based consistency check** สำหรับหน้าที่ต้องอ่านข้อมูลจาก replica ในเวลาไม่นานหลังเขียน (เช่น หน้าประวัติคำสั่งซื้อที่เพิ่งสั่ง)

#### 3) Load Balancer และ Health Check Strategy

- HAProxy deploy แบบ HA pair (active-passive กับ keepalived + VIP) ในแต่ละ region ตาม Part 065
- Health check ผ่าน **Patroni REST API** (`/primary`, `/replica?lag=5000000`) — ตั้ง lag threshold 5MB (~ไม่กี่วินาทีในสภาวะปกติ) สำหรับ general read pool ที่ SG, และ threshold ที่ผ่อนปรนกว่า (เช่น 50MB) สำหรับ analytics replica เพราะยอมรับ lag ได้มากกว่า
- ใช้ **`balance leastconn`** สำหรับ general read pool (workload หลากหลาย) เพราะ query ต่าง ๆ มีเวลาไม่เท่ากัน
- EU replica pool ตั้งค่า local replica เป็นหลัก และใส่ SG replica เป็น `backup` (fallback) เผื่อ EU replicas ทั้งหมด down พร้อมกัน

#### 4) การจัดการ Replication Lag/Consistency

- flow critical (checkout, stock check) → บังคับไป Primary เสมอ ไม่ผ่าน replica เลย
- flow ทั่วไปที่ผู้ใช้อาจสังเกตเห็น inconsistency (เช่น หน้าประวัติคำสั่งซื้อหลัง checkout ทันที) → ใช้ sticky window 2-3 วินาที บังคับอ่าน Primary ชั่วคราว
- EU replica ยอมรับ lag สูงกว่า SG replica (เพราะเป็น cross-region) — สื่อสารกับทีม frontend ว่าหน้าที่ non-critical ใน EU อาจเห็นข้อมูลช้ากว่าปกติเล็กน้อย ซึ่งยอมรับได้ตาม business requirement

#### 5) แนวทางรองรับ Failover (SLA: downtime ไม่เกิน 30 วินาที)

- ตั้งค่า Patroni TTL/loop_wait ให้ detection เร็ว (เช่น ttl=15s, loop_wait=5s) ตาม Part 065
- HAProxy health check interval `inter 2s fall 2 rise 3` → detection ภายใน ~4 วินาที
- รวม Patroni detection (~10-15s) + HAProxy convergence (~4-6s) + client retry (~2-5s) = รวมประมาณ 16-26 วินาที ซึ่งอยู่ใน SLA 30 วินาที
- Application implement retry with exponential backoff + idempotency key สำหรับ write operation (เช่น checkout ใช้ idempotency key ป้องกันการสั่งซื้อซ้ำหาก retry หลัง network error)
- ทดสอบ failover scenario เป็นประจำ (chaos engineering / game day) เพื่อยืนยันว่า SLA ยังคงอยู่จริงเมื่อระบบเติบโต

### 670.3 สรุปการออกแบบเป็นตาราง Checklist

| Component | การตัดสินใจ | เหตุผลอ้างอิง Step |
|---|---|---|
| Primary location | Singapore (ตามสัดส่วนผู้ใช้ส่วนใหญ่) | 669 |
| Replica count (SG) | 3 (2 general + 1 analytics) | 661, 662 |
| Replica count (EU) | 2 | 669 |
| Routing | Proxy-level (HAProxy) + query-type aware ที่ app layer สำหรับ FOR UPDATE | 663, 664 |
| Load balance algorithm | leastconn (general), round-robin ok สำหรับ analytics (query เดี่ยว) | 666 |
| Health check | Patroni REST API + lag threshold ต่างกันตาม pool | 667 |
| Consistency | Sticky window + LSN check + cache-aside RETURNING | 665 |
| Failover SLA | Patroni tuning + HAProxy tuning + app retry | 668 |
| Multi-region | Single Primary + regional read replicas + backup fallback | 669 |

---

## สรุปท้ายบท

ในบทนี้เราได้เรียนรู้การสร้างกลยุทธ์ **Read Replica และ Load Balancing** อย่างครบวงจร โดยต่อยอดจากความรู้ Streaming Replication (Part 063) และ Connection Pooling (Part 066) ประเด็นสำคัญที่ควรจำ:

1. **Read Replica** ใช้ Standby server ที่เปิด hot_standby รับ read traffic เพื่อลดภาระ Primary — เหมาะกับระบบที่มีสัดส่วน read สูง (เช่น e-commerce 80-95%)
2. **Read/Write Splitting** ต้องแยกให้ชัดว่า write ไป Primary เสมอ ส่วน read ไป Replica ได้ ยกเว้นกรณีพิเศษ (FOR UPDATE, sequence, read-after-write critical)
3. **Application-level routing** ให้ latency ต่ำสุดและควบคุมละเอียด แต่ maintain ยากเมื่อระบบใหญ่ขึ้น ส่วน **Proxy-level routing (HAProxy)** centralize logic ได้ดีกว่าและไม่ต้อง deploy โค้ดใหม่เมื่อ topology เปลี่ยน
4. **Replication Lag** เป็นข้อจำกัดโดยธรรมชาติของ async replication — ปัญหา **read-after-write consistency** แก้ได้ด้วย sticky window, LSN-based check, synchronous replication เฉพาะจุด, หรือ cache-aside pattern
5. **Load Balancing Algorithm**: `leastconn` เหมาะกับ workload หลากหลาย (แนะนำเป็น default สำหรับ PostgreSQL read pool), `round-robin` เหมาะกับ workload สม่ำเสมอ
6. **Health Check** ต้องตรวจ 3 มิติ: availability, role correctness (ยังเป็น replica จริงไหม), และ replication lag ไม่เกิน threshold — แนะนำใช้ Patroni REST API แทนการเขียน script เอง
7. **Failover** ส่งผลกระทบต่อ Load Balancer โดยตรง ต้องคำนวณ detection + convergence time รวมทั้ง stack (Patroni + HAProxy + application retry) ให้อยู่ใน SLA ที่ยอมรับได้
8. **Multi-region** ช่วยลด read latency ได้มากด้วย local replica แต่ write latency ยังคงผูกกับตำแหน่ง Primary — ต้องพิจารณา trade-off เรื่อง cost, consistency, compliance ด้วย

บทถัดไป **Part 068 — Roles และ Privileges** จะพาเราออกจากเรื่อง architecture/scaling ไปสู่มิติด้าน **Security** ของ PostgreSQL อย่างละเอียด ทั้งการจัดการ roles, privileges, row-level security และแนวทาง access control ที่เหมาะสมสำหรับระบบระดับองค์กร

**บทถัดไป**: [Part 068 — Roles และ Privileges](./part-068-roles-privileges.md)

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1

Read Replica คืออะไร และเพราะเหตุใดจึงช่วยลดภาระของ Primary server ได้? อธิบายพร้อมยกตัวอย่างสัดส่วน read/write ของระบบจริงที่ทำให้กลยุทธ์นี้มีประโยชน์

<details>
<summary>เฉลย</summary>

Read Replica คือ Standby server (สร้างด้วยกลไก Streaming Replication ตาม Part 063) ที่เปิดโหมด `hot_standby = on` เพื่อรับ query อ่าน (SELECT) ได้ ขณะเดียวกันก็ยัง apply WAL จาก Primary อยู่เบื้องหลัง

มันช่วยลดภาระ Primary เพราะระบบส่วนใหญ่ (เช่น e-commerce) มีสัดส่วน read มากกว่า write มาก โดยทั่วไปอยู่ที่ 80/20 ถึง 95/5 (read 80-95%, write 5-20%) การส่ง read query ส่วนใหญ่ไปที่ replica แทนที่จะให้ Primary รับภาระทั้งหมด ช่วยลด CPU/Memory/Disk I/O contention บน Primary, ลด lock contention ระหว่าง transaction เขียนกับ query อ่านหนัก ๆ และเพิ่ม throughput รวมของระบบเพราะสามารถ scale read ออกในแนวนอนได้ (เพิ่ม replica เพิ่ม throughput)

ข้อจำกัดที่ต้องระวังคือ replica รับได้แค่ query อ่านเท่านั้น (read-only) ไม่สามารถ scale การเขียนได้ และมี replication lag ที่ข้อมูลอาจไม่ real-time เท่า Primary
</details>

---

### แบบฝึกหัดที่ 2

ยกตัวอย่าง query 3 ประเภทที่เป็น SELECT แต่ต้อง "บังคับ" ส่งไปที่ Primary แทนที่จะไป Replica พร้อมอธิบายเหตุผล

<details>
<summary>เฉลย</summary>

ตัวอย่างที่เป็นไปได้ (เลือกได้จาก Step 662.2):

1. **`SELECT ... FOR UPDATE` / `FOR SHARE`** — เพราะเป็นการ lock row เพื่อเตรียมเขียนต่อในภายหลัง ต้องทำที่ Primary เท่านั้นเพราะ replica ไม่สามารถทำ row-level lock แบบที่จะเขียนต่อได้ (read-only transaction)

2. **SELECT ที่อยู่ใน transaction เดียวกับ write (read-modify-write pattern)** — เช่น อ่านค่า balance ก่อนจะ UPDATE ทันที ต้องอยู่ transaction เดียวกันที่ Primary เพื่อรักษาความเป็น atomic และป้องกัน race condition

3. **`SELECT nextval('some_sequence')`** — เพราะการเรียก sequence เป็นการเปลี่ยนสถานะ (side effect) ซึ่งต้องทำที่ Primary เท่านั้น

4. (เพิ่มเติม) **Query ที่ต้องการอ่านข้อมูลทันทีหลังเขียนเอง (read-after-write critical)** เช่น หน้ายืนยันคำสั่งซื้อ เพื่อหลีกเลี่ยงปัญหา replication lag ตาม Step 665

5. (เพิ่มเติม) **Query ที่ใช้ temp table** ที่สร้างไว้ใน session เดียวกัน เพราะ temp table มีอยู่แค่ session ที่สร้างที่ Primary เท่านั้น
</details>

---

### แบบฝึกหัดที่ 3

เปรียบเทียบ Application-level routing กับ Proxy-level routing ในแง่ของ latency, maintainability และการรองรับ failover พร้อมให้คำแนะนำว่าควรเลือกใช้แบบไหนในสถานการณ์ที่มี microservices จำนวนมาก

<details>
<summary>เฉลย</summary>

**Application-level routing**: โค้ดแอปตัดสินใจ routing เอง
- Latency: ต่ำกว่า เพราะไม่มี network hop ผ่าน proxy กลาง
- Maintainability: แย่กว่าเมื่อระบบใหญ่ขึ้น เพราะต้อง implement logic ซ้ำในทุก service/ภาษา และเสี่ยง human error (ลืม mark write query)
- Failover: แอปต้อง implement health check และ retry เอง ซ้ำซ้อนในทุก service

**Proxy-level routing (เช่น HAProxy)**: Proxy กลางตัดสินใจ routing แทน
- Latency: สูงกว่าเล็กน้อยจาก network hop เพิ่ม (แต่มักเล็กน้อยระดับ sub-millisecond)
- Maintainability: ดีกว่ามาก เพราะ logic รวมศูนย์จุดเดียว เปลี่ยน topology (เพิ่ม/ลด replica) ไม่ต้อง deploy โค้ดแอปใหม่
- Failover: centralize health check ผ่าน proxy เดียว ทุก service ได้ประโยชน์จากการปรับปรุงจุดเดียว

**คำแนะนำ**: ในสถานการณ์ที่มี microservices จำนวนมาก **ควรเลือก Proxy-level routing** เพราะการรวมศูนย์ logic ที่จุดเดียวช่วยลดความซับซ้อนของการดูแลรักษาอย่างมาก ทุก service เพียงแค่เชื่อมต่อไปที่ endpoint เดียว (write/read) โดยไม่ต้องรู้รายละเอียด topology หรือ implement routing logic ซ้ำในแต่ละภาษา/framework ที่ใช้
</details>

---

### แบบฝึกหัดที่ 4

จงเขียนตัวอย่าง haproxy.cfg ที่มี 2 pool คือ write_pool (port 5000) และ read_pool (port 5001) โดยใช้ Patroni REST API เป็น health check และใช้ leastconn algorithm สำหรับ read_pool

<details>
<summary>เฉลย</summary>

```cfg
global
    log 127.0.0.1 local2
    maxconn 4000
    daemon

defaults
    log global
    mode tcp
    retries 2
    timeout client 30s
    timeout connect 4s
    timeout server 30s
    timeout check 5s

listen write_pool
    bind *:5000
    option httpchk GET /primary
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions

    server pg-node1 pg-node1.internal:5432 maxconn 100 check port 8008
    server pg-node2 pg-node2.internal:5432 maxconn 100 check port 8008
    server pg-node3 pg-node3.internal:5432 maxconn 100 check port 8008

listen read_pool
    bind *:5001
    balance leastconn
    option httpchk GET /replica
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions

    server pg-node1 pg-node1.internal:5432 maxconn 200 check port 8008
    server pg-node2 pg-node2.internal:5432 maxconn 200 check port 8008
    server pg-node3 pg-node3.internal:5432 maxconn 200 check port 8008
```

จุดสำคัญ: ต้องใส่ node ทั้ง 3 ตัวในทั้งสอง pool เพราะ role ของแต่ละ node เปลี่ยนแปลงได้ตลอดเวลาจาก failover — health check (`/primary` และ `/replica`) จะเป็นตัวกรองว่า node ไหนควรรับ traffic ประเภทไหนในขณะนั้น ไม่ใช่การ hardcode
</details>

---

### แบบฝึกหัดที่ 5

อธิบายปัญหา "Read-after-write consistency" พร้อมยกตัวอย่าง timeline ที่แสดงว่าปัญหานี้เกิดขึ้นได้อย่างไร และเสนอแนวทางแก้ไขอย่างน้อย 2 วิธี

<details>
<summary>เฉลย</summary>

**ปัญหา**: เกิดขึ้นเมื่อผู้ใช้เขียนข้อมูลสำเร็จที่ Primary แล้วทันทีหลังจากนั้นพยายามอ่านข้อมูลนั้นจาก Replica ซึ่งยัง apply WAL ของ transaction นั้นไม่ทัน (เพราะ replication เป็นแบบ asynchronous) ทำให้ query ไม่พบข้อมูลที่เพิ่งเขียนไป

**ตัวอย่าง timeline**:
```
T=0ms  ผู้ใช้กด "ยืนยันคำสั่งซื้อ" → INSERT INTO orders ... [ไป Primary]
T=5ms  Primary COMMIT สำเร็จ ตอบกลับว่า order id=12345
T=8ms  หน้าใหม่ query: SELECT * FROM orders WHERE id=12345 [ไป Replica]
T=8ms  Replica ยัง apply WAL ไม่เสร็จ → ไม่พบ order 12345 (ผู้ใช้เห็น error)
T=58ms Replica apply เสร็จ (สายไปแล้ว)
```

**แนวทางแก้ไข** (เลือกอย่างน้อย 2 ข้อ):
1. **Sticky window (time-based)**: หลังเขียนข้อมูล บังคับอ่านจาก Primary ในช่วงเวลาสั้น ๆ (เช่น 2 วินาที) ก่อนกลับไปใช้ replica ตามปกติ
2. **LSN-based consistency check**: ดึง LSN หลังเขียนที่ Primary แล้วตรวจสอบว่า replica ที่จะอ่าน replay ผ่าน LSN นั้นแล้วหรือยัง ถ้ายังให้เลือก replica อื่นหรือรอ/fallback ไป Primary
3. **Synchronous replication เฉพาะจุด**: ตั้งค่า synchronous_commit สำหรับ transaction สำคัญ เพื่อการันตีว่า replica ที่กำหนดต้อง apply เสร็จก่อนถือว่า commit สำเร็จ
4. **Cache-aside ผ่าน RETURNING clause**: ใช้ผลลัพธ์จาก INSERT...RETURNING ที่ Primary ส่งกลับมาแสดงผลตรง ๆ โดยไม่ query ซ้ำจาก replica เลย
</details>

---

### แบบฝึกหัดที่ 6

Load Balancing Algorithm แบบ `round-robin` และ `leastconn` ต่างกันอย่างไร และควรเลือกใช้แบบไหนสำหรับ read pool ของ PostgreSQL ที่มี query หลากหลายประเภท (ทั้ง simple lookup และ heavy report query)?

<details>
<summary>เฉลย</summary>

**Round-robin**: กระจาย request ไปยัง server แต่ละตัวเรียงตามลำดับวนไปเรื่อย ๆ โดยไม่คำนึงถึงภาระงานปัจจุบันของแต่ละ server

**Leastconn**: ส่ง request ไปยัง server ที่มีจำนวน connection ที่ active อยู่ตอนนี้น้อยที่สุด ซึ่งสะท้อนภาระงานจริงของแต่ละ server ได้ดีกว่า

**คำแนะนำ**: สำหรับ read pool ที่มี query หลากหลายประเภท (mix ของ simple lookup ที่เร็วกับ heavy report query ที่ใช้เวลานาน) ควรเลือกใช้ **`leastconn`** เพราะ round-robin จะยังส่ง request ใหม่ไปยัง server ที่กำลังรัน report query หนักอยู่โดยไม่สนใจว่ามันยุ่งอยู่ ทำให้เกิดการกระจายภาระที่ไม่สมดุล ในขณะที่ leastconn จะหลีกเลี่ยง server ที่มี connection ค้างอยู่เยอะ (เช่น กำลังรัน query หนัก) และส่ง request ใหม่ไปยัง server ที่ว่างกว่าแทน
</details>

---

### แบบฝึกหัดที่ 7

Health Check ที่ดีสำหรับ Read Replica ควรตรวจสอบกี่มิติ อะไรบ้าง? และเพราะเหตุใดการเช็คแค่ "server ตอบสนองไหม" จึงไม่เพียงพอ?

<details>
<summary>เฉลย</summary>

Health Check ที่ดีควรตรวจสอบ 3 มิติ:

1. **Availability** — instance ทำงานอยู่ไหม accept connection ได้ไหม (เช่นผ่าน TCP connect หรือ pg_isready)
2. **Role correctness** — เป็น replica จริงหรือไม่ (ตรวจผ่าน `pg_is_in_recovery()` หรือ Patroni REST API) เพราะหลัง failover node ที่เคยเป็น replica อาจถูก promote เป็น primary แล้ว ถ้า Load Balancer ยังส่ง read traffic ไปที่ node นั้นในฐานะ replica เดิมจะไม่มีปัญหา (เพราะ primary ก็อ่านได้) แต่ถ้าส่ง write ไปที่ node ที่กลายเป็น replica แล้วจะ error ทันที ดังนั้นการตรวจ role ให้ตรงกับ pool (write_pool ต้องเป็น primary, read_pool ต้องเป็น replica) จึงจำเป็น
3. **Replication lag ยอมรับได้** — lag ต้องไม่เกิน threshold ที่ตั้งไว้ เพื่อป้องกันไม่ให้ผู้ใช้เห็นข้อมูลเก่าเกินไป

การเช็คแค่ "ตอบสนองไหม" ไม่เพียงพอ เพราะ server อาจ "เป็น" (alive) แต่มี role ผิด (กลายเป็น primary หลัง failover ทั้งที่ยังอยู่ใน read_pool) หรือมี replication lag สูงมากจนข้อมูลเก่าเกินกว่าจะยอมรับได้ ซึ่งทั้งสองกรณีนี้ server จะตอบสนองปกติแต่ไม่ควรรับ traffic ในบทบาทนั้น ๆ
</details>

---

### แบบฝึกหัดที่ 8

จากค่า config `inter 3s fall 3 rise 2` ใน HAProxy จงคำนวณเวลา worst-case ที่ HAProxy จะใช้ในการ mark server ว่า "down" หลังจาก server นั้นเริ่มมีปัญหา

<details>
<summary>เฉลย</summary>

จากค่า `inter 3s fall 3`:
- `inter 3s` หมายถึง HAProxy เช็ค health ทุก 3 วินาที
- `fall 3` หมายถึงต้อง fail ติดต่อกัน 3 ครั้งถึงจะถือว่า down

เวลา worst-case = inter × fall = 3s × 3 = **9 วินาที**

(หมายเหตุ: นี่คือเวลาที่ HAProxy ใช้เพื่อ mark down เท่านั้น ยังไม่รวมเวลาที่ Patroni ใช้ตรวจจับและ promote replica ใหม่ ซึ่งต้องรวมเข้าไปด้วยเพื่อคำนวณ total failover impact time ตามที่อธิบายใน Step 668.2)
</details>

---

### แบบฝึกหัดที่ 9

ในสถาปัตยกรรม Multi-region (เช่น Primary ที่ Singapore, Replica ที่ Frankfurt) เพราะเหตุใด write latency ของผู้ใช้ในยุโรปจึงยังคงสูงอยู่ แม้จะมี replica ท้องถิ่นอยู่ที่ Frankfurt แล้ว?

<details>
<summary>เฉลย</summary>

เพราะ Replica เป็น **read-only** เท่านั้น ไม่สามารถรับ write query ได้เลย (PostgreSQL standard replication ไม่รองรับการเขียนที่ standby) ดังนั้นแม้จะมี Replica อยู่ใกล้ผู้ใช้ในยุโรปที่ Frankfurt แล้ว แต่ write query ทุกตัว (INSERT/UPDATE/DELETE) ยังคงต้องถูกส่งข้าม region ไปยัง **Primary ที่ Singapore** เสมอ ทำให้ write latency ของผู้ใช้ในยุโรปยังคงสูง (ประมาณ 150-250ms ตามระยะทางเครือข่ายข้ามทวีป) ในขณะที่ read latency ลดลงได้มากเพราะอ่านจาก local replica

การจะลด write latency ให้ผู้ใช้ทุกภูมิภาคพร้อมกันต้องใช้สถาปัตยกรรม multi-primary/active-active ซึ่งซับซ้อนกว่ามากและอยู่นอกขอบเขตของบทนี้ (ตามที่กล่าวใน Step 669.4)
</details>

---

### แบบฝึกหัดที่ 10

สำหรับระบบ e-commerce ที่มี flow "ตรวจสอบสต็อกสินค้าก่อนขาย" ซึ่งต้องการความถูกต้องสูงมาก (ป้องกัน overselling) จงอธิบายว่าทำไม query ประเภทนี้ไม่ควรส่งไปที่ Replica และควรออกแบบอย่างไร

<details>
<summary>เฉลย</summary>

Query ตรวจสอบสต็อกก่อนขาย ต้องมีลักษณะ "read-modify-write" คือ อ่านจำนวนสต็อกปัจจุบัน แล้วต้องแน่ใจว่าไม่มี transaction อื่นมาแย่งซื้อสินค้าตัวสุดท้ายพร้อมกัน (race condition) ซึ่งต้องใช้ `SELECT ... FOR UPDATE` เพื่อ lock row ของสินค้านั้นไว้ก่อนจะ UPDATE ลดจำนวนสต็อก

เหตุผลที่ไม่ควรส่งไป Replica:
1. Replica เป็น read-only ไม่สามารถทำ row-level lock ที่จะใช้เขียนต่อได้ (`FOR UPDATE` ต้องทำที่ Primary)
2. แม้จะอ่านได้จาก Replica แต่ข้อมูลอาจมี replication lag ทำให้เห็นจำนวนสต็อกที่ "เก่ากว่าความเป็นจริง" เสี่ยงต่อการ oversell (เช่น สต็อกเหลือ 0 จริงที่ Primary แล้ว แต่ replica ยัง apply WAL ไม่ทันจึงยังเห็นว่าเหลือ 1 ชิ้น)

การออกแบบที่ถูกต้อง:
```sql
BEGIN;
SELECT qty FROM inventory WHERE sku = 'ABC123' FOR UPDATE;  -- ต้องที่ Primary
-- ตรวจสอบว่า qty > 0
UPDATE inventory SET qty = qty - 1 WHERE sku = 'ABC123';
INSERT INTO orders (...) VALUES (...);
COMMIT;
```

ทั้งหมดนี้ต้องอยู่ใน transaction เดียวกันที่ Primary เท่านั้น เพื่อรักษาความเป็น atomic และป้องกัน overselling อย่างสมบูรณ์ — สอดคล้องกับหลักการ Read/Write Splitting ใน Step 662 ที่ระบุว่า SELECT FOR UPDATE ต้องบังคับไป Primary เสมอ
</details>

---

**บทถัดไป**: [Part 068 — Roles และ Privileges](./part-068-roles-privileges.md)
