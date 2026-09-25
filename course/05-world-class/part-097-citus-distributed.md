# Part 097: Distributed PostgreSQL — Citus และ Sharding Strategy

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับโลก/ผู้เชี่ยวชาญ | Part 097

## เป้าหมายการเรียนรู้

เมื่อจบบทนี้ ผู้เรียนจะสามารถ:

- อธิบายได้ว่าทำไม single-node PostgreSQL ถึงมีข้อจำกัด และเมื่อไรที่องค์กรควรพิจารณาไปสู่ distributed database
- เข้าใจแนวคิด Sharding และแยกความแตกต่างจาก Partitioning (Part 054) ได้อย่างชัดเจน
- อธิบายสถาปัตยกรรมของ Citus — Coordinator Node, Worker Node, Shard, Metadata
- สร้างและจัดการ Distributed Table ด้วย `create_distributed_table()` พร้อมเลือก distribution column (shard key) ได้อย่างเหมาะสม
- สร้าง Reference Table ด้วย `create_reference_table()` สำหรับตารางขนาดเล็กที่ต้องอยู่ครบทุก worker
- ออกแบบ Co-located Tables เพื่อให้ JOIN ทำงานได้เร็วโดยไม่ต้องส่งข้อมูลข้าม network
- เข้าใจกลไก Query Routing ของ Citus — Router Query เทียบกับ Scatter-Gather Query
- เข้าใจแนวคิด Rebalancing และคำสั่ง `citus_rebalance_start()`
- ออกแบบกลยุทธ์ sharding สำหรับระบบ e-commerce แบบ multi-tenant SaaS ด้วย Citus ได้จริง

> **หมายเหตุสำคัญเกี่ยวกับสภาพแวดล้อม**: บทนี้เป็นเนื้อหาเชิงสถาปัตยกรรมและแนวคิด (architectural/conceptual) ที่มาพร้อมไวยากรณ์ SQL ที่ถูกต้องแม่นยำสำหรับคำสั่งของ Citus ระบบ Citus cluster แบบหลาย node (multi-node) ไม่สามารถติดตั้งและรันได้จริงในสภาพแวดล้อมฝึกฝนของหลักสูตรนี้ เนื่องจากต้องใช้เครื่องหลายเครื่อง (coordinator + workers) พร้อมระบบเครือข่ายเชื่อมต่อกัน อย่างไรก็ตาม โค้ด SQL ทุกชิ้นในบทนี้เขียนขึ้นให้ตรงกับไวยากรณ์จริงของ Citus (อ้างอิงจาก Citus เวอร์ชันล่าสุดที่เป็น open source ภายใต้ Microsoft/Citus Data) และสามารถนำไปรันได้จริงบน Citus cluster จริง ไม่ว่าจะเป็นแบบ self-hosted หรือบริการ managed เช่น Azure Cosmos DB for PostgreSQL

---

## Step 956: ทำไมต้อง Distributed Database

### ข้อจำกัดของ Single-Node PostgreSQL

ใน Part 075 เราได้พูดถึงเรื่อง Scaling Strategies ไปแล้วว่า PostgreSQL แบบ single-node สามารถ scale ได้สองทาง คือ **Vertical Scaling** (เพิ่ม CPU, RAM, disk ให้เครื่องเดียวแรงขึ้น) และ **Read Replica** (กระจาย read load ออกไปหลายเครื่อง แต่ write ยังคงกระจุกอยู่ที่ primary เดียว) ทั้งสองแนวทางนี้มีเพดานที่ชนได้เสมอ:

```
┌─────────────────────────────────────────────────────────┐
│              ข้อจำกัดของ Single-Node PostgreSQL           │
├─────────────────────────────────────────────────────────┤
│                                                           │
│  1. Storage Ceiling                                      │
│     - Disk เดียวมีขนาดจำกัด (แม้จะเป็น network storage   │
│       อย่าง EBS/PD ก็มี IOPS และ throughput จำกัด)        │
│     - ตารางที่มีหลายพันล้านแถวทำให้ index bloat,          │
│       vacuum ใช้เวลานานขึ้นเรื่อย ๆ                       │
│                                                           │
│  2. Write Throughput Ceiling                              │
│     - Write ทั้งหมดต้องผ่าน WAL ของ primary node เดียว    │
│     - Read replica ช่วย read ได้ แต่ไม่ช่วย write เลย      │
│     - เมื่อ write TPS แตะเพดานของ disk I/O หรือ CPU        │
│       single core (WAL เขียนแบบ sequential) ก็ scale      │
│       ต่อไม่ได้อีก                                        │
│                                                           │
│  3. Memory Ceiling                                        │
│     - shared_buffers, work_mem ถูกจำกัดด้วย RAM ของ       │
│       เครื่องเดียว เมื่อ working set ใหญ่กว่า RAM         │
│       เครื่องเดียวที่หาซื้อได้ (เช่น เกิน 1-2 TB) ก็ต้อง   │
│       เผชิญ disk I/O มากขึ้น                              │
│                                                           │
│  4. Single Point of Bottleneck                             │
│     - แม้จะมี HA (Part 072-074) แต่ ณ เวลาใดเวลาหนึ่ง      │
│       มี "เครื่องเดียว" ที่รับ write ทั้งหมด                │
│     - Geographic latency: ผู้ใช้งานทั่วโลกต้องยิง write   │
│       ไปยัง region เดียวเสมอ                              │
│                                                           │
└─────────────────────────────────────────────────────────┘
```

### เมื่อไรที่ Vertical Scaling และ Read Replica ไม่พอ

ในทางปฏิบัติ องค์กรส่วนใหญ่ไม่จำเป็นต้องไปถึงจุดนี้ — PostgreSQL แบบ single-node ที่ tune อย่างดี (ตามที่เราเรียนมาตลอดทั้งหลักสูตร) สามารถรองรับ workload ได้มากกว่าที่คนทั่วไปคิดมาก เครื่องระดับ 128 core, RAM 1TB, NVMe SSD สามารถรองรับได้หลายหมื่น TPS และข้อมูลระดับหลายสิบ TB ได้สบาย

แต่มีสถานการณ์ที่ single-node "ไม่พอ" จริง ๆ:

| สถานการณ์ | ทำไม Vertical Scaling ไม่พอ |
|---|---|
| ข้อมูลรวมเกินความจุ disk ของเครื่องที่ใหญ่ที่สุดที่หาซื้อได้ (หลาย PB) | ไม่มี disk เดียวที่ใหญ่พอ หรือแพงเกินจะคุ้มค่า |
| Write throughput เกินเพดานของ CPU/disk เดี่ยว (แสน TPS ขึ้นไป) | WAL write เป็น sequential bottleneck ที่ไม่กระจายได้ในเครื่องเดียว |
| ต้องการ Geo-distribution (write ใกล้ผู้ใช้ทั่วโลก) | Single primary อยู่ region เดียวเสมอ |
| Multi-tenant SaaS ที่ tenant เพิ่มขึ้นเรื่อย ๆ จนข้อมูลรวมใหญ่เกินเครื่องเดียวรองรับ | จำนวน tenant และข้อมูลต่อ tenant โตแบบไม่มีเพดาน |
| ต้องการ Compute แบบขนาน (parallel) ข้าม node สำหรับ analytical query บนข้อมูลขนาดใหญ่มาก | Parallel query (Part 060) ใน PostgreSQL เดี่ยวจำกัดที่ core ของเครื่องเดียว |

### ทางเลือกเมื่อ Vertical Scaling หมดทาง

```
┌───────────────────────────────────────────────────────────┐
│         ทางเลือกเมื่อ Single-Node ไปต่อไม่ไหว              │
├───────────────────────────────────────────────────────────┤
│                                                               │
│  A) Application-level Sharding (Manual)                     │
│     - แยก database ตาม tenant/region เอง ในระดับ           │
│       application code (connection routing เอง)             │
│     - ควบคุมได้เต็มที่ แต่ต้องเขียน logic เองทั้งหมด         │
│       (routing, cross-shard join, rebalancing)               │
│                                                               │
│  B) Distributed SQL Database จากศูนย์ (เช่น CockroachDB,     │
│     YugabyteDB, Google Spanner)                              │
│     - ออกแบบมาเพื่อ distributed ตั้งแต่ต้น                  │
│     - ไม่ใช้ PostgreSQL engine จริง (แม้จะพูด wire protocol  │
│       เข้ากันได้) — ecosystem, extension, behavior ต่างกัน   │
│                                                               │
│  C) Citus — PostgreSQL Extension สำหรับ Distributed          │
│     - ยังคงเป็น PostgreSQL แท้ ๆ (fork ของ Postgres core     │
│       ไม่มี, ใช้ extension mechanism)                        │
│     - ใช้ SQL, extension, driver, tooling ของ PostgreSQL     │
│       เดิมได้ทั้งหมด                                         │
│     - นี่คือหัวข้อหลักของบทนี้                                │
│                                                               │
└───────────────────────────────────────────────────────────┘
```

ข้อดีของ Citus คือมันไม่ใช่ database ใหม่ที่ต้องเรียนรู้ syntax ใหม่ทั้งหมด — มันคือ **extension** ที่ครอบอยู่บน PostgreSQL ปกติ ความรู้ทั้งหมดที่เราเรียนมาตลอด 96 บทก่อนหน้านี้ (indexing, MVCC, WAL, query planner, replication, partitioning) ยังคงใช้ได้ เพียงแต่ Citus จะเพิ่มมิติของการ "กระจาย" ข้อมูลออกไปหลายเครื่องเข้ามา

---

## Step 957: Sharding คืออะไร

### นิยาม

**Sharding** คือการแบ่งข้อมูลของตารางหนึ่งออกเป็นส่วนย่อย ๆ ที่เรียกว่า **shard** โดยแต่ละ shard จะถูกกระจายไปเก็บไว้ยังเครื่อง (node) ที่แตกต่างกัน การแบ่งจะอิงตามค่าของคอลัมน์ใดคอลัมน์หนึ่งที่เรียกว่า **shard key** (หรือ distribution column)

ตัวอย่างเช่น ตาราง `orders` ที่มี 1,000 ล้านแถว หาก shard ด้วย `customer_id` เป็น 32 shard ข้อมูลของลูกค้าแต่ละคนจะถูก hash และกระจายไปอยู่ใน shard ใด shard หนึ่งจาก 32 shard — และแต่ละ shard อาจอยู่คนละเครื่องกันได้

```
┌─────────────────────────────────────────────────────────────┐
│                    แนวคิดของ Sharding                        │
├─────────────────────────────────────────────────────────────┤
│                                                                 │
│   ตาราง orders (logical, มองจาก application เห็นเป็น 1 ตาราง) │
│                                                                 │
│         hash(customer_id) กำหนดว่าแถวไปอยู่ shard ไหน           │
│                                                                 │
│   ┌───────────┐   ┌───────────┐   ┌───────────┐               │
│   │  Shard 1  │   │  Shard 2  │   │  Shard 3  │   ...          │
│   │ (Node A)  │   │ (Node B)  │   │ (Node C)  │               │
│   │ cust 1-3  │   │ cust 4-7  │   │ cust 8-9  │               │
│   └───────────┘   └───────────┘   └───────────┘               │
│                                                                 │
│   แต่ละ shard คือ "ตารางจริง" ที่มีข้อมูลเพียงบางส่วน          │
│   และอาศัยอยู่บนเครื่องคนละเครื่องกัน (คนละ CPU, RAM, disk)     │
│                                                                 │
└─────────────────────────────────────────────────────────────┘
```

### Sharding เทียบกับ Partitioning (ทบทวน Part 054)

นี่คือจุดที่ผู้เรียนมักสับสนมากที่สุด เพราะทั้งสองแนวคิด "ฟังดูคล้ายกัน" — ทั้งคู่คือการแบ่งตารางใหญ่ออกเป็นส่วนย่อย แต่ **ความแตกต่างที่สำคัญที่สุดคือตำแหน่งทางกายภาพของข้อมูล**

| มิติ | Partitioning (Part 054) | Sharding (Citus) |
|---|---|---|
| ข้อมูลอยู่ที่ไหน | อยู่ในเครื่อง (instance) เดียวกันทั้งหมด | กระจายอยู่คนละเครื่อง (node) กัน |
| แก้ปัญหาอะไร | จัดการตารางใหญ่ให้ query/maintenance เร็วขึ้น (partition pruning, vacuum แยกส่วน) | แก้ปัญหาที่เครื่องเดียวไม่พอ (CPU, RAM, disk, write throughput) |
| CPU/RAM ที่ใช้ | ใช้ CPU/RAM ของเครื่องเดียว ร่วมกันทุก partition | แต่ละ shard ใช้ CPU/RAM ของ node ตัวเอง แยกจากกัน |
| Scale ต่อได้ไหม | ไม่ scale เรื่อง compute — ยังจำกัดด้วยเครื่องเดียว | Scale ได้โดยเพิ่ม worker node เข้าไปเรื่อย ๆ (horizontal) |
| กลไกเบื้องหลัง | PostgreSQL native partitioning (`PARTITION BY`) | Citus distributed table (`create_distributed_table`) ซึ่งใช้ partitioning + foreign/local tables ผสมกันภายใน |
| ความสัมพันธ์กัน | เป็นเทคนิคระดับ "ตาราง" | มักใช้ "ร่วมกับ" partitioning ได้ — แต่ละ shard ในแต่ละ node ก็สามารถถูก partition ต่อภายในได้อีกชั้น |

**ข้อสังเกตสำคัญ**: Sharding และ Partitioning ไม่ใช่สิ่งที่ต้องเลือกอย่างใดอย่างหนึ่ง ในระบบขนาดใหญ่จริงมักใช้ **ทั้งคู่ร่วมกัน** — ใช้ Citus sharding เพื่อกระจายข้อมูลข้าม node ตาม `tenant_id` แล้วภายในแต่ละ shard (ซึ่งเป็นตารางจริงบน worker node) ก็ยังสามารถทำ time-based partitioning ตาม `created_at` ได้อีกชั้นหนึ่งตามที่เรียนใน Part 054

```
┌───────────────────────────────────────────────────────────┐
│     Sharding (ข้าม node) + Partitioning (ภายใน node)       │
├───────────────────────────────────────────────────────────┤
│                                                               │
│   Node A (Worker 1)                Node B (Worker 2)         │
│   ┌─────────────────────┐         ┌─────────────────────┐   │
│   │ orders_shard_1       │         │ orders_shard_2       │   │
│   │  ├─ orders_2024_q1   │         │  ├─ orders_2024_q1   │   │
│   │  ├─ orders_2024_q2   │         │  ├─ orders_2024_q2   │   │
│   │  └─ orders_2024_q3   │         │  └─ orders_2024_q3   │   │
│   │  (native partition)  │         │  (native partition)  │   │
│   └─────────────────────┘         └─────────────────────┘   │
│                                                               │
└───────────────────────────────────────────────────────────┘
```

### Sharding ในโลกกว้าง: ไม่ใช่แนวคิดใหม่

Sharding เป็นแนวคิดที่ใช้กันมานานในระบบขนาดใหญ่ — MongoDB, Cassandra, Vitess (MySQL sharding), DynamoDB ล้วนใช้แนวคิดนี้ สิ่งที่ Citus ทำให้พิเศษคือการนำแนวคิดนี้มาใช้กับ **PostgreSQL ของแท้** โดยไม่ต้องทิ้ง SQL, ACID transaction (ภายใน shard เดียว), extension ecosystem, และเครื่องมือที่คุ้นเคยไปเลย

---

## Step 958: Citus คืออะไร

### นิยามและปรัชญา

**Citus** คือ PostgreSQL extension (ไม่ใช่ fork, ไม่ใช่ database แยกต่างหาก) ที่เปลี่ยน PostgreSQL instance ให้กลายเป็นส่วนหนึ่งของ distributed database cluster ได้ โดยที่:

- Citus ถูกพัฒนาโดยบริษัท Citus Data ซึ่งถูก Microsoft ซื้อกิจการไปในปี 2019
- เป็น **open source** ภายใต้ AGPLv3 license
- ติดตั้งผ่าน `CREATE EXTENSION citus;` เหมือน extension ทั่วไป (`pg_stat_statements`, `postgis` ฯลฯ)
- ยังคงใช้ PostgreSQL query planner, executor, storage engine, WAL, MVCC เดิมทุกประการในระดับ node เดียว — Citus เพิ่ม "ชั้นของการกระจาย" (distributed layer) ครอบไว้ด้านบน

```sql
-- การติดตั้ง Citus extension (รันบนทุก node — coordinator และ worker)
CREATE EXTENSION citus;

-- ตรวจสอบเวอร์ชัน
SELECT citus_version();

-- ดู extension ที่ติดตั้งอยู่
SELECT * FROM pg_extension WHERE extname = 'citus';
```

### ทำไมยังใช้ SQL/PostgreSQL Ecosystem เดิมได้

นี่คือจุดขายหลักของ Citus เมื่อเทียบกับ distributed database อื่น ๆ:

```
┌─────────────────────────────────────────────────────────────┐
│              สิ่งที่ยังคงใช้ได้เหมือนเดิมกับ Citus            │
├─────────────────────────────────────────────────────────────┤
│                                                                 │
│  ✓ Standard SQL (SELECT, INSERT, UPDATE, DELETE, JOIN)        │
│  ✓ Transactions (BEGIN/COMMIT) — ภายใน shard เดียวได้ ACID    │
│    เต็มรูปแบบ, ข้าม shard ได้ 2PC (two-phase commit)           │
│  ✓ Indexes (B-tree, GIN, GiST, BRIN) — Part 015-017            │
│  ✓ psql, pgAdmin, DBeaver และ client tools ทั้งหมด             │
│  ✓ Driver ภาษาต่าง ๆ (psycopg2, node-postgres, pgx, JDBC)     │
│    ที่เราเรียนใน Part 087-090 — เชื่อมต่อ Citus ผ่าน            │
│    connection string ปกติ ไม่ต้องเปลี่ยน driver               │
│  ✓ PostgreSQL extensions อื่น ๆ ที่ทำงานร่วมกันได้            │
│    (PostGIS, pg_trgm, pgvector, TimescaleDB บางส่วน)          │
│  ✓ Window functions, CTE, JSON/JSONB functions                │
│  ✓ Replication, backup tools (pg_dump ใช้ได้ในระดับหนึ่ง,      │
│    แนะนำใช้เครื่องมือเฉพาะของ Citus สำหรับ production)         │
│                                                                 │
│  ✗ Foreign key ข้าม shard ไม่รองรับแบบ native ทั้งหมด          │
│  ✗ ไม่ใช่ทุก query pattern ที่ optimize ได้เท่า single-node    │
│    (query ที่ join ข้าม distribution column ต้องส่งข้อมูล      │
│    ข้าม network)                                               │
│  ✗ ตารางบาง feature เช่น SERIAL อาจต้องปรับ (แนะนำใช้           │
│    globally unique id generation เช่น UUID หรือ                │
│    sequence แบบ per-node)                                      │
│                                                                 │
└─────────────────────────────────────────────────────────────┘
```

### Citus ในฐานะ "Deployment Option" ของ PostgreSQL

สิ่งสำคัญที่ต้องเข้าใจคือ Citus **ไม่ได้บังคับ** ให้ทุกตารางต้องกระจาย เราสามารถมี:

1. **Distributed table** — ตารางที่กระจายเป็น shard ข้าม worker (เนื้อหาหลักของบทนี้)
2. **Reference table** — ตารางเล็กที่ copy เต็มไปทุก worker (Step 961)
3. **Local table (Citus local table)** — ตารางที่อยู่บน coordinator เท่านั้น เหมือน PostgreSQL ปกติ ไม่ถูกกระจาย

การออกแบบระบบจริงจึงเป็นการ "ผสม" ทั้งสามแบบนี้เข้าด้วยกันตามความเหมาะสมของแต่ละตาราง ไม่ใช่ทุกตารางต้องถูก shard

### Deployment Models

```
┌───────────────────────────────────────────────────────────┐
│                  รูปแบบการ deploy Citus                     │
├───────────────────────────────────────────────────────────┤
│                                                               │
│  1) Self-hosted Citus (Open Source)                          │
│     - ติดตั้ง PostgreSQL + citus extension เอง               │
│     - ควบคุม coordinator/worker เอง                          │
│     - เหมาะกับ on-premise, ต้องการควบคุมเต็มที่               │
│                                                               │
│  2) Azure Cosmos DB for PostgreSQL (Managed Citus)            │
│     - บริการ managed ของ Microsoft                            │
│     - จัดการ node provisioning, backup, monitoring ให้        │
│                                                               │
│  3) Citus บน Kubernetes (ผ่าน Citus Operator หรือ Helm)       │
│     - เชื่อมโยงกับ Part 093 (Docker/Kubernetes)               │
│     - เหมาะกับ cloud-native infrastructure                    │
│                                                               │
└───────────────────────────────────────────────────────────┘
```

---

## Step 959: สถาปัตยกรรม Citus — Coordinator และ Worker Node

### ภาพรวมสถาปัตยกรรม

Citus cluster ประกอบด้วย node สองประเภทที่ทำหน้าที่ต่างกันชัดเจน:

```
┌─────────────────────────────────────────────────────────────────┐
│                     Citus Cluster Architecture                    │
│                                                                     │
│                         Application / Client                       │
│                                 │                                   │
│                                 │  (psql, driver ปกติ)              │
│                                 ▼                                   │
│                  ┌───────────────────────────┐                    │
│                  │      COORDINATOR NODE       │                    │
│                  │  (Coordinator / Master)     │                    │
│                  │                             │                    │
│                  │  - รับ query จาก client     │                    │
│                  │  - เก็บ metadata:           │                    │
│                  │    pg_dist_partition,       │                    │
│                  │    pg_dist_shard,           │                    │
│                  │    pg_dist_placement,       │                    │
│                  │    pg_dist_node             │                    │
│                  │  - Query planner ตัดสินใจ   │                    │
│                  │    ว่า query ต้องไปที่       │                    │
│                  │    shard ไหนบ้าง            │                    │
│                  │  - รวมผล (aggregate) จาก     │                    │
│                  │    หลาย worker กลับมา       │                    │
│                  └──────────┬──────────────────┘                  │
│                              │                                      │
│              ┌───────────────┼───────────────┐                    │
│              │               │               │                     │
│              ▼               ▼               ▼                     │
│    ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              │
│    │  WORKER 1   │  │  WORKER 2   │  │  WORKER 3   │   ...        │
│    │             │  │             │  │             │              │
│    │ shard_101   │  │ shard_103   │  │ shard_105   │              │
│    │ shard_102   │  │ shard_104   │  │ shard_106   │              │
│    │             │  │             │  │             │              │
│    │ (ตารางจริง  │  │ (ตารางจริง  │  │ (ตารางจริง  │              │
│    │  PostgreSQL │  │  PostgreSQL │  │  PostgreSQL │              │
│    │  ปกติ)      │  │  ปกติ)      │  │  ปกติ)      │              │
│    └─────────────┘  └─────────────┘  └─────────────┘              │
│                                                                     │
└─────────────────────────────────────────────────────────────────┘
```

### บทบาทของ Coordinator Node

Coordinator (บางเอกสารรุ่นเก่าเรียกว่า Master node) ทำหน้าที่เป็น "สมองส่วนกลาง" ของ cluster:

1. **เก็บ metadata ของการกระจายข้อมูล** — Coordinator รู้ว่าตาราง `orders` ถูกแบ่งเป็นกี่ shard, แต่ละ shard มี hash range เท่าไหร่ และอยู่ worker node ไหน ข้อมูลนี้เก็บใน catalog table พิเศษของ Citus:

```sql
-- Metadata table หลักที่ coordinator ใช้เก็บข้อมูลการกระจาย

-- ตารางไหนถูก distribute แล้วบ้าง, distribution column คืออะไร
SELECT * FROM pg_dist_partition;

-- แต่ละ shard มี id อะไร, min/max hash range เท่าไหร่
SELECT * FROM pg_dist_shard;

-- shard แต่ละอันอยู่ node ไหน (placement)
SELECT * FROM pg_dist_placement;

-- รายชื่อ node ทั้งหมดใน cluster (coordinator รู้จัก worker ไหนบ้าง)
SELECT * FROM pg_dist_node;

-- View ที่อ่านง่ายกว่า รวมข้อมูล shard + node + ขนาด
SELECT * FROM citus_shards;
```

2. **Query Planning & Routing** — เมื่อ query เข้ามา coordinator จะวิเคราะห์ว่า query นี้ต้องการข้อมูลจาก shard ไหนบ้าง (รายละเอียดใน Step 963)

3. **Transaction Coordination** — สำหรับ transaction ที่ต้องเขียนข้าม node coordinator ทำหน้าที่เป็น 2PC (two-phase commit) coordinator

4. **Result Aggregation** — เมื่อ worker หลายตัวส่งผลกลับมา (เช่น query ที่มี `COUNT(*)`, `SUM()`) coordinator จะรวมผลจากทุก worker เข้าด้วยกันก่อนส่งกลับ client

### บทบาทของ Worker Node

Worker node คือ PostgreSQL instance ปกติ (มี extension citus ติดตั้งเหมือนกัน) ที่เก็บข้อมูลจริง:

- แต่ละ worker เก็บ **shard** ซึ่งก็คือตารางจริง ๆ ในเครื่องนั้น (physical table ที่มีชื่อรูปแบบ `tablename_shardid` เช่น `orders_102008`)
- Worker รัน query แบบ local เหมือน PostgreSQL ปกติทุกประการ — ใช้ index, query planner, MVCC ของตัวเองในการประมวลผล shard ที่ตัวเองรับผิดชอบ
- Worker ไม่รู้จัก "ภาพรวม" ของ cluster ทั้งหมด — มันรู้แค่ว่าตัวเองมี shard อะไรบ้าง

```sql
-- คำสั่งเหล่านี้รันบน coordinator เพื่อเพิ่ม worker node เข้า cluster
-- (สมมติว่า worker ติดตั้ง PostgreSQL + citus extension ไว้แล้ว
--  และเปิด network access ระหว่าง coordinator-worker ไว้แล้ว)

SELECT citus_add_node('worker-node-1.internal', 5432);
SELECT citus_add_node('worker-node-2.internal', 5432);
SELECT citus_add_node('worker-node-3.internal', 5432);

-- ตรวจสอบว่า worker เชื่อมต่อสำเร็จ และ active หรือไม่
SELECT * FROM citus_get_active_worker_nodes();

-- ตรวจสุขภาพการเชื่อมต่อไปยังทุก node
SELECT * FROM citus_check_cluster_node_health();
```

### Coordinator เป็น Single Point of Failure หรือไม่

คำถามที่พบบ่อยคือ "ถ้า coordinator ล่ม ทั้ง cluster จะล่มไหม" — คำตอบคือใช่ในทางทฤษฎี coordinator ยังเป็นจุดสำคัญที่ต้องรับ query ทั้งหมด แต่ในทางปฏิบัติ:

- Coordinator เองก็เป็น PostgreSQL ธรรมดา จึงสามารถตั้ง **Streaming Replication + Standby** (ตามที่เรียนใน Part 071-074) ให้กับ coordinator ได้เหมือน PostgreSQL ทั่วไป
- Metadata (pg_dist_*) จะถูก replicate ไปยัง standby ของ coordinator ด้วย เพราะมันคือข้อมูลใน catalog table ปกติที่อยู่ใน WAL
- เมื่อ coordinator ล่ม สามารถ failover ไปยัง standby ได้เหมือนที่เรียนมาใน Part 072-074 (Patroni, repmgr ฯลฯ)
- Citus เวอร์ชันใหม่ยังรองรับแนวคิด "coordinator metadata sync ไปยัง worker" ผ่าน `citus.enable_metadata_sync` ทำให้ worker บางส่วนสามารถทำหน้าที่ query routing ได้ด้วยในบางกรณี ลดการพึ่งพา coordinator เดียว

---

## Step 960: Distributed Table — create_distributed_table()

### แนวคิดของ Distributed Table

Distributed table คือตารางที่ Citus จะแบ่งข้อมูลออกเป็นหลาย shard โดยอัตโนมัติ ตาม **distribution column** (หรือเรียกอีกชื่อว่า shard key) ที่เราเลือก ทุกครั้งที่ insert แถวใหม่ Citus จะคำนวณ `hash(distribution_column)` แล้วตัดสินใจว่าแถวนั้นควรไปอยู่ shard ไหน

### ขั้นตอนพื้นฐานในการสร้าง Distributed Table

```sql
-- Step 1: สร้างตารางปกติแบบที่เราคุ้นเคยจากทั้งหลักสูตร (ยังไม่ distributed)
CREATE TABLE companies (
    id          bigserial,
    name        text NOT NULL,
    domain      text,
    created_at  timestamptz NOT NULL DEFAULT now(),
    PRIMARY KEY (id)
);

CREATE TABLE campaigns (
    id            bigserial,
    company_id    bigint NOT NULL,
    name          text NOT NULL,
    budget        numeric(12,2),
    created_at    timestamptz NOT NULL DEFAULT now(),
    PRIMARY KEY (id, company_id)
);

-- Step 2: แปลงให้เป็น distributed table
-- Argument ที่สองคือ distribution column (shard key)
SELECT create_distributed_table('companies', 'id');
SELECT create_distributed_table('campaigns', 'company_id');

-- ตรวจสอบว่าตารางถูก distribute แล้ว
SELECT logicalrelid, partmethod, distribution_column_name
FROM pg_dist_partition
JOIN LATERAL (
    SELECT column_to_column_name(logicalrelid, partkey) AS distribution_column_name
) t ON true;
```

> **หมายเหตุ**: ตัวอย่างข้างต้นแสดง pattern ทั่วไปของการดู distribution column — ในทางปฏิบัติมักใช้ view สำเร็จรูปอย่าง `citus_tables` ที่จะแนะนำถัดไป ซึ่งอ่านง่ายกว่า

```sql
-- วิธีที่แนะนำและใช้กันจริง: view citus_tables อ่านง่ายกว่ามาก
SELECT table_name, citus_table_type, distribution_column, shard_count
FROM citus_tables;
```

ผลลัพธ์ตัวอย่าง:

```
     table_name      | citus_table_type | distribution_column | shard_count
----------------------+------------------+----------------------+-------------
 companies            | distributed      | id                   | 32
 campaigns            | distributed      | company_id           | 32
```

### การกำหนดจำนวน Shard

```sql
-- กำหนดจำนวน shard ล่วงหน้าก่อนสร้างตารางแรก (ค่า default คือ 32)
SET citus.shard_count = 64;

CREATE TABLE events (
    id          bigserial,
    tenant_id   bigint NOT NULL,
    event_type  text NOT NULL,
    payload     jsonb,
    created_at  timestamptz NOT NULL DEFAULT now(),
    PRIMARY KEY (id, tenant_id)
);

SELECT create_distributed_table('events', 'tenant_id');

-- ปรับจำนวน shard ของตารางที่ distribute ไปแล้ว (rebalance ตามมาด้วย)
SELECT alter_distributed_table('events', shard_count => 128, cascade_to_colocated => true);
```

**หลักการเลือกจำนวน shard**: จำนวน shard ควรมากกว่าจำนวน worker node ที่คาดว่าจะมีในอนาคต (ไม่ใช่แค่ปัจจุบัน) เพราะการเพิ่ม shard ใหม่ทีหลังทำได้ยากกว่าการมี shard เผื่อไว้แล้วค่อย rebalance ไปยัง worker ใหม่ แนวทางทั่วไปคือตั้งจำนวน shard ให้เป็น 2-4 เท่าของจำนวน worker สูงสุดที่คาดว่าจะขยายไปถึง เช่น ถ้าคาดว่าจะมี worker สูงสุด 32 ตัว อาจตั้ง shard count ไว้ที่ 128-256

### การเลือก Distribution Column ที่เหมาะสม

นี่คือการตัดสินใจที่ **สำคัญที่สุด** ในการออกแบบระบบด้วย Citus เพราะเปลี่ยนทีหลังทำได้ยากและมีค่าใช้จ่ายสูง (ต้อง `undistribute_table()` แล้วสร้างใหม่ หรือใช้ `alter_distributed_table` ที่มีข้อจำกัด)

```
┌─────────────────────────────────────────────────────────────┐
│              หลักการเลือก Distribution Column (Shard Key)     │
├─────────────────────────────────────────────────────────────┤
│                                                                 │
│  ✓ High Cardinality — มีค่าที่แตกต่างกันเยอะพอที่จะกระจาย      │
│    โหลดได้สม่ำเสมอ (เช่น tenant_id ที่มีเป็นหมื่น tenant       │
│    ดีกว่า status ที่มีแค่ 3-4 ค่า)                              │
│                                                                 │
│  ✓ Even Distribution — ค่าต่าง ๆ ควรมีปริมาณข้อมูลใกล้เคียงกัน  │
│    หลีกเลี่ยง "hot shard" ที่ tenant รายใหญ่รายเดียวมีข้อมูล    │
│    มากกว่า tenant อื่นเป็นร้อยเท่า (data skew)                  │
│                                                                 │
│  ✓ ปรากฏใน WHERE clause ของ query ส่วนใหญ่ — เพื่อให้เกิด       │
│    Router Query (Step 963) แทนที่จะเป็น scatter-gather         │
│    ทุกครั้ง                                                    │
│                                                                 │
│  ✓ ใช้เป็น Join Key ระหว่างตารางที่เกี่ยวข้องกัน — เพื่อทำ       │
│    Co-location ได้ (Step 962)                                  │
│                                                                 │
│  ✗ หลีกเลี่ยงคอลัมน์ที่เปลี่ยนค่าบ่อย (UPDATE บน distribution   │
│    column ทำได้แต่มีค่าใช้จ่ายสูง เพราะอาจต้องย้ายแถวข้าม       │
│    shard)                                                       │
│                                                                 │
│  ✗ หลีกเลี่ยงคอลัมน์ที่มี NULL เยอะ (NULL ทั้งหมดจะกระจุก        │
│    อยู่ hash เดียวกัน)                                          │
│                                                                 │
└─────────────────────────────────────────────────────────────┘
```

### ตัวอย่างการเลือก Shard Key ที่ดีและไม่ดี

| Use Case | Shard Key ที่ดี | เหตุผล | Shard Key ที่ไม่ดี | เหตุผล |
|---|---|---|---|---|
| Multi-tenant SaaS | `tenant_id` / `company_id` | Query ส่วนใหญ่ filter ด้วย tenant, cardinality สูง, isolation ชัดเจน | `created_at` | Query ต้อง scan ทุก shard เสมอ, ข้อมูลใหม่กระจุกที่ shard ล่าสุด (hot shard) |
| E-commerce (จะกล่าวถึงใน Step 965) | `customer_id` หรือ `store_id` (ขึ้นกับ query pattern) | สอดคล้องกับ access pattern หลัก | `product_category` | Cardinality ต่ำ (มีไม่กี่ category), กระจายไม่สม่ำเสมอ |
| IoT Sensor Data | `device_id` | สอดคล้องกับ query pattern "ดูข้อมูลของ device นี้" | `sensor_reading_id` (auto-increment) | ไม่มีความหมายทาง business, ทำ co-location กับตารางอื่นไม่ได้ |
| Social Media Posts | `user_id` | Query ส่วนใหญ่คือ "ดูโพสต์ของ user นี้" | `post_id` (random UUID) | ไม่สัมพันธ์กับ access pattern, join กับ users ยาก |

### Data Skew — ปัญหาที่ต้องระวัง

```sql
-- ตรวจสอบขนาดของแต่ละ shard เพื่อดู data skew
SELECT * FROM citus_shards
WHERE table_name = 'campaigns'::regclass
ORDER BY shard_size DESC
LIMIT 10;

-- ดูขนาดรวมของแต่ละ shard พร้อม node ที่มันอยู่
SELECT shardid, nodename, nodeport, shard_size
FROM citus_shards
ORDER BY shard_size DESC;
```

หาก shard บางอันมีขนาดใหญ่กว่า shard อื่นมาก (เช่น tenant รายใหญ่รายเดียวมีข้อมูล 40% ของทั้งหมด) นี่คือสัญญาณของ **data skew** ซึ่งทำให้ worker node ที่เก็บ shard นั้นรับโหลดไม่สมส่วนกับ worker อื่น วิธีแก้ไขที่ใช้กันในระบบจริงคือ:

1. เลือก shard key ที่มี cardinality สูงกว่า (เช่น ใช้ `(tenant_id, user_id)` แทน `tenant_id` เดี่ยว ๆ)
2. สำหรับ tenant ขนาดใหญ่ผิดปกติ (whale tenant) อาจแยกไปอยู่ dedicated node ต่างหาก
3. ใช้ **Isolate Tenant** feature ของ Citus (`isolate_tenant_to_new_shard()`) เพื่อแยก tenant รายใหญ่ออกจาก shard ที่ใช้ร่วมกับ tenant อื่น

```sql
-- แยก tenant ที่มีข้อมูลมากผิดปกติออกไปเป็น shard เฉพาะของตัวเอง
SELECT isolate_tenant_to_new_shard('campaigns', 424242, 'CASCADE');
```

---

## Step 961: Reference Table — create_reference_table()

### ทำไมต้องมี Reference Table

ลองนึกภาพตาราง `countries` ที่มีข้อมูลแค่ ~250 แถว (รายชื่อประเทศ) หรือ `categories` ที่มีสัก 100 แถว ตารางเหล่านี้:

- มีขนาดเล็กมาก
- แทบไม่เปลี่ยนแปลง (read-heavy, write แทบไม่มี)
- ถูก JOIN บ่อยมากกับตารางอื่นที่ distribute อยู่ (เช่น `orders JOIN countries ON orders.country_id = countries.id`)

ถ้าเรา distribute ตาราง `countries` ด้วย shard key ปกติ ทุกครั้งที่มีการ JOIN กับ `orders` (ซึ่ง shard ด้วย `customer_id`) Citus จะต้องส่งข้อมูล `countries` ข้าม network ไปมาระหว่าง worker ตลอดเวลา — นี่คือสิ่งที่เราต้องการหลีกเลี่ยง

**Reference Table** คือคำตอบ: ตารางนี้จะถูก **copy เต็มรูปแบบไปยังทุก worker node** (ไม่ใช่แบ่ง shard) ทำให้ทุก worker มีข้อมูลตารางนี้ครบสมบูรณ์อยู่ในเครื่องตัวเอง การ JOIN จึงทำได้แบบ local เสมอ ไม่ต้องส่งข้อมูลข้าม network เลย

```
┌───────────────────────────────────────────────────────────┐
│                  Reference Table Replication                │
├───────────────────────────────────────────────────────────┤
│                                                                 │
│         ตาราง countries (250 แถว)                             │
│                                                                 │
│    ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│    │  WORKER 1   │  │  WORKER 2   │  │  WORKER 3   │        │
│    │             │  │             │  │             │        │
│    │ countries   │  │ countries   │  │ countries   │        │
│    │ (250 แถว    │  │ (250 แถว    │  │ (250 แถว    │        │
│    │  ครบทุก     │  │  ครบทุก     │  │  ครบทุก     │        │
│    │  แถว)       │  │  แถว)       │  │  แถว)       │        │
│    │             │  │             │  │             │        │
│    │ orders      │  │ orders      │  │ orders      │        │
│    │ shard_1     │  │ shard_2     │  │ shard_3     │        │
│    │ (บาง        │  │ (บาง        │  │ (บาง        │        │
│    │  ส่วน)      │  │  ส่วน)      │  │  ส่วน)      │        │
│    └─────────────┘  └─────────────┘  └─────────────┘        │
│                                                                 │
│    JOIN orders + countries ทำได้แบบ local ในแต่ละ worker      │
│    ไม่ต้องส่งข้อมูลข้าม network เลย                            │
│                                                                 │
└───────────────────────────────────────────────────────────┘
```

### วิธีสร้าง Reference Table

```sql
CREATE TABLE countries (
    id          smallint PRIMARY KEY,
    iso_code    char(2) NOT NULL UNIQUE,
    name        text NOT NULL
);

INSERT INTO countries (id, iso_code, name) VALUES
    (1, 'TH', 'Thailand'),
    (2, 'US', 'United States'),
    (3, 'JP', 'Japan');
    -- ... ประเทศอื่น ๆ

-- ประกาศให้เป็น reference table — Citus จะ copy ไปทุก worker ให้อัตโนมัติ
SELECT create_reference_table('countries');

CREATE TABLE product_categories (
    id          smallint PRIMARY KEY,
    name        text NOT NULL,
    parent_id   smallint REFERENCES product_categories(id)
);

SELECT create_reference_table('product_categories');

-- ตรวจสอบว่าตารางไหนเป็น reference table
SELECT table_name, citus_table_type, shard_count
FROM citus_tables
WHERE citus_table_type = 'reference';
```

### ลักษณะเฉพาะของ Reference Table

```sql
-- Reference table มี "1 shard" เสมอ แต่ shard นั้นถูก replicate
-- ไปทุก worker (ไม่ใช่แบ่งข้อมูล แต่ copy ข้อมูลทั้งหมด)
SELECT logicalrelid, shardid, nodename, nodeport
FROM pg_dist_shard
JOIN pg_dist_shard_placement USING (shardid)
WHERE logicalrelid = 'countries'::regclass;
```

ผลลัพธ์จะแสดงให้เห็นว่า shard เดียวของ `countries` มี placement อยู่บน **ทุก worker node** (ต่างจาก distributed table ที่แต่ละ shard จะอยู่ node เดียว)

**ข้อควรระวัง**: การ INSERT/UPDATE/DELETE บน reference table จะต้องเขียนไปยังทุก worker พร้อมกัน (ผ่าน 2PC) ดังนั้น reference table ควรเป็นตารางที่ write **น้อยมาก** ถ้าตารางมี write บ่อย ควรพิจารณาเป็น distributed table แทน หรือถ้าข้อมูลเล็กมากและไม่ค่อยเปลี่ยน อาจพิจารณาเก็บไว้ที่ application-level cache แทน

### Foreign Key ระหว่าง Distributed Table และ Reference Table

จุดแข็งอีกอย่างของ Reference Table คือมันรองรับ foreign key จาก distributed table ได้:

```sql
CREATE TABLE orders (
    id           bigserial,
    customer_id  bigint NOT NULL,
    country_id   smallint NOT NULL REFERENCES countries(id),
    total_amount numeric(12,2),
    created_at   timestamptz NOT NULL DEFAULT now(),
    PRIMARY KEY (id, customer_id)
);

SELECT create_distributed_table('orders', 'customer_id');

-- Foreign key นี้ทำงานได้เพราะ countries เป็น reference table
-- ที่มีอยู่ครบทุก worker — ทุก shard ของ orders จึงมี countries
-- ให้ตรวจสอบ constraint ได้ในเครื่องตัวเอง
```

---

## Step 962: Co-located Tables — การออกแบบให้ JOIN เร็วโดยไม่ข้าม Network

### แนวคิดของ Co-location

**Co-location** คือการออกแบบให้ตารางหลายตารางที่มักถูก JOIN กันบ่อย ๆ ใช้ **distribution column เดียวกัน** และมี **จำนวน shard เท่ากัน** ทำให้ Citus สามารถวางแถวที่มีค่า distribution column เดียวกันจากทุกตารางไว้บน **worker node เดียวกัน** เสมอ

```
┌───────────────────────────────────────────────────────────────┐
│                   Co-located Tables Example                      │
│                                                                     │
│   companies (shard key: id)          campaigns (shard key: company_id) │
│                                                                     │
│   ┌─────────────────────────────────────────────────────┐        │
│   │                    WORKER 2                            │        │
│   │                                                          │        │
│   │   companies (shard 2)         campaigns (shard 2)       │        │
│   │   ┌──────────────────┐       ┌──────────────────┐      │        │
│   │   │ id=42, "Acme Co"  │       │ company_id=42,    │      │        │
│   │   │                    │       │ "Q4 Campaign"     │      │        │
│   │   │                    │       │ company_id=42,    │      │        │
│   │   │                    │       │ "Holiday Promo"   │      │        │
│   │   └──────────────────┘       └──────────────────┘      │        │
│   │                                                          │        │
│   │   ทั้ง companies id=42 และ campaigns ที่ company_id=42    │        │
│   │   อยู่บน worker เดียวกัน! JOIN ทำได้แบบ local             │        │
│   └─────────────────────────────────────────────────────┘        │
│                                                                     │
└───────────────────────────────────────────────────────────────┘
```

### วิธีทำ Co-location

Citus จะ co-locate ตารางที่มี **distribution column type เดียวกัน** และ **shard count เท่ากัน** โดยอัตโนมัติ (จัดกลุ่มเป็น "colocation group" เดียวกัน) แต่วิธีที่ชัดเจนและแนะนำคือการระบุ `colocate_with` อย่างชัดแจ้ง:

```sql
-- ตารางหลัก: companies เป็น "root" ของ co-location group
SELECT create_distributed_table('companies', 'id');

-- ตาราง campaigns ต้องการ JOIN กับ companies บ่อยผ่าน company_id
-- ระบุ colocate_with ชัดเจนเพื่อให้แน่ใจว่ามันอยู่ colocation group เดียวกัน
SELECT create_distributed_table(
    'campaigns',
    'company_id',
    colocate_with => 'companies'
);

-- ตาราง ad_creatives ก็ JOIN กับ campaigns ผ่าน company_id เช่นกัน
CREATE TABLE ad_creatives (
    id            bigserial,
    company_id    bigint NOT NULL,
    campaign_id   bigint NOT NULL,
    creative_url  text,
    PRIMARY KEY (id, company_id)
);

SELECT create_distributed_table(
    'ad_creatives',
    'company_id',
    colocate_with => 'companies'
);
```

### ตรวจสอบ Co-location Group

```sql
-- ดู colocation group ของแต่ละตาราง — ตารางที่มี colocationid เดียวกัน
-- คือ co-located กัน
SELECT logicalrelid, colocationid
FROM pg_dist_partition
ORDER BY colocationid, logicalrelid;

-- View ที่อ่านง่ายกว่า
SELECT table_name, colocation_id, distribution_column
FROM citus_tables
ORDER BY colocation_id;
```

ผลลัพธ์ตัวอย่าง:

```
   table_name    | colocation_id | distribution_column
------------------+----------------+----------------------
 companies        |              5 | id
 campaigns        |              5 | company_id
 ad_creatives     |              5 | company_id
 countries        |           NULL | (reference table)
```

ทั้งสามตาราง (`companies`, `campaigns`, `ad_creatives`) มี `colocation_id = 5` เหมือนกัน — นี่คือสิ่งที่ทำให้ query ต่อไปนี้ทำงานได้อย่างมีประสิทธิภาพ:

```sql
-- Query นี้ join 3 ตารางที่ co-located กัน โดยกรองด้วย company_id เดียว
-- Citus รู้ว่าข้อมูลทั้งหมดของ company_id=42 อยู่ worker เดียวกัน
-- จึงส่ง query ทั้งชุดไปประมวลผลที่ worker นั้น worker เดียว (router query)
SELECT c.name, cam.name AS campaign_name, ac.creative_url
FROM companies c
JOIN campaigns cam ON cam.company_id = c.id
JOIN ad_creatives ac ON ac.campaign_id = cam.id AND ac.company_id = c.id
WHERE c.id = 42;
```

### ทำไม Co-location ถึงสำคัญมาก

ถ้าไม่ทำ co-location (เช่น `campaigns` ใช้ shard key เป็น `id` ของตัวเองแทนที่จะเป็น `company_id`) การ JOIN ระหว่าง `companies` และ `campaigns` จะกลายเป็น **repartition join** — Citus ต้องดึงข้อมูลจากทุก shard ของทั้งสองตารางออกมา จัดเรียงใหม่ตาม join key แล้วส่งข้าม network ไปมาระหว่าง worker เพื่อจับคู่ให้ตรงกัน ซึ่งช้ากว่า co-located join มาก (เป็นสิบถึงร้อยเท่าในกรณีข้อมูลใหญ่)

```
┌─────────────────────────────────────────────────────────────┐
│         Co-located Join เทียบกับ Repartition Join            │
├─────────────────────────────────────────────────────────────┤
│                                                                 │
│  Co-located Join (companies + campaigns ใช้ company_id ร่วมกัน)│
│  ───────────────────────────────────────────────────────────  │
│  Worker 1: JOIN local (ไม่มี network I/O)                     │
│  Worker 2: JOIN local (ไม่มี network I/O)                     │
│  Worker 3: JOIN local (ไม่มี network I/O)                     │
│  Coordinator: รวมผลจากทุก worker                              │
│  → เร็ว, scale ได้ดีตามจำนวน worker                           │
│                                                                 │
│  Repartition Join (shard key ไม่ตรงกัน)                       │
│  ───────────────────────────────────────────────────────────  │
│  Worker 1-3: อ่านข้อมูลของตัวเอง แล้ว "repartition"            │
│              (จัดกลุ่มใหม่ตาม join key) ส่งไปยัง worker         │
│              อื่นที่ควรรับผิดชอบ join key นั้น                  │
│  → เกิด network shuffle จำนวนมาก, ช้ากว่ามาก                   │
│  → ยังทำงานได้ (Citus รองรับ) แต่ performance ต่ำกว่ามาก        │
│                                                                 │
└─────────────────────────────────────────────────────────────┘
```

```sql
-- เปิดใช้งาน repartition join สำหรับกรณีที่จำเป็นจริง ๆ
-- (ค่า default คือปิดไว้เพื่อป้องกัน query ที่ไม่ได้ตั้งใจให้ช้า)
SET citus.enable_repartition_joins = true;
```

### หลักการออกแบบ Co-location ในระบบจริง

1. เลือกตารางหลัก (root entity) ที่เป็นศูนย์กลางของความสัมพันธ์ เช่น `tenant`, `company`, `customer`
2. ตารางลูกทุกตารางที่มักถูก JOIN กับตารางหลักควรใช้ foreign key column เดียวกันเป็น distribution column
3. ถ้าตารางลูกมี primary key ของตัวเอง (เช่น `campaigns.id`) primary key ควรเป็น composite key ที่รวม distribution column ด้วย เช่น `PRIMARY KEY (id, company_id)` เพื่อให้ constraint ตรวจสอบได้ภายใน shard เดียว
4. ตารางที่ "ไม่เกี่ยวข้อง" กับ root entity เลย (เช่น audit log ของระบบ, configuration กลาง) อาจไม่ต้อง co-locate — พิจารณาเป็น reference table หรือ local table แทน

---

## Step 963: Query Routing — Router Query กับ Scatter-Gather Query

### สองรูปแบบหลักของการประมวลผล Query ใน Citus

เมื่อ query เข้ามาที่ coordinator, Citus query planner จะวิเคราะห์และจัดประเภท query เป็นสองแบบหลัก ๆ:

```
┌─────────────────────────────────────────────────────────────┐
│                Router Query vs Scatter-Gather Query           │
├─────────────────────────────────────────────────────────────┤
│                                                                 │
│  ROUTER QUERY (เร็วที่สุด)                                     │
│  ─────────────────────────                                     │
│  Query มี WHERE clause ที่ระบุค่าของ distribution column        │
│  แบบ exact match (เช่น WHERE company_id = 42)                  │
│                                                                 │
│  Coordinator                                                   │
│      │  hash(42) → รู้ทันทีว่าอยู่ shard ไหน, worker ไหน        │
│      ▼                                                          │
│  ┌─────────────┐                                                │
│  │  Worker 2   │  ← ส่ง query ทั้งหมดไปที่ worker นี้ worker    │
│  │  (เท่านั้น) │    เดียว ประมวลผล local ทั้งหมด (รวม JOIN      │
│  └─────────────┘    ถ้า co-located) แล้วส่งผลกลับ               │
│                                                                 │
│                                                                 │
│  SCATTER-GATHER QUERY (ช้ากว่า แต่จำเป็นบางกรณี)                │
│  ─────────────────────────────────────────────                 │
│  Query ไม่ได้ filter ด้วย distribution column แบบ exact         │
│  (เช่น aggregate ทั้งตาราง, range query, ไม่มี WHERE เลย)        │
│                                                                 │
│  Coordinator                                                   │
│      │  ส่ง query (หรือ query ที่ปรับแล้ว) ไปทุก worker         │
│      ├──────────┬──────────┬──────────┐                       │
│      ▼          ▼          ▼          ▼                       │
│  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐                  │
│  │Worker 1│ │Worker 2│ │Worker 3│ │Worker 4│                  │
│  └────────┘ └────────┘ └────────┘ └────────┘                  │
│      │          │          │          │                       │
│      └──────────┴──────────┴──────────┘                       │
│                    │                                            │
│                    ▼                                            │
│            Coordinator รวมผล                                    │
│            (SUM, COUNT, sort, merge ฯลฯ)                        │
│                                                                 │
└─────────────────────────────────────────────────────────────┘
```

### ตัวอย่าง Router Query

```sql
-- Router Query: filter ด้วย company_id ชัดเจน → ไปที่ worker เดียว
SELECT * FROM campaigns WHERE company_id = 42;

-- Router Query ที่ join ตาราง co-located กัน ก็ยังเป็น router query
-- ตราบใดที่ทุกตารางถูก filter ด้วย distribution column เดียวกัน
SELECT c.name, cam.name
FROM companies c
JOIN campaigns cam ON cam.company_id = c.id
WHERE c.id = 42;

-- INSERT/UPDATE/DELETE ที่ระบุ distribution column ชัดเจน
-- ก็เป็น router query เช่นกัน (เขียนไป shard เดียว)
INSERT INTO campaigns (company_id, name, budget)
VALUES (42, 'New Year Sale', 50000.00);

UPDATE campaigns SET budget = 60000.00
WHERE company_id = 42 AND id = 501;
```

### ตัวอย่าง Scatter-Gather Query

```sql
-- ไม่มี filter บน distribution column → ต้องกระจายไปทุก shard
SELECT COUNT(*) FROM campaigns WHERE budget > 10000;

-- Aggregate ข้ามทั้งตาราง → scatter-gather เสมอ
SELECT company_id, SUM(budget) AS total_budget
FROM campaigns
GROUP BY company_id
ORDER BY total_budget DESC
LIMIT 10;

-- Full table scan โดยไม่ filter อะไรเลย
SELECT * FROM campaigns ORDER BY created_at DESC LIMIT 100;
```

query เหล่านี้ยังคงทำงานได้ถูกต้อง (Citus จะ scatter ไปทุก worker แล้ว gather ผลกลับมา รวมถึงทำ `ORDER BY ... LIMIT` แบบ distributed merge sort ที่ coordinator) แต่ค่าใช้จ่ายด้าน latency และ network I/O สูงกว่า router query มาก เพราะต้อง fan-out ไปทุก worker

### การตรวจสอบว่า Query เป็นแบบไหน

```sql
-- ใช้ EXPLAIN เพื่อดูว่า Citus วางแผน query อย่างไร
EXPLAIN (VERBOSE, COSTS OFF)
SELECT * FROM campaigns WHERE company_id = 42;
```

ผลลัพธ์ของ router query จะมี node ชื่อ `Custom Scan (Citus Adaptive)` พร้อม `Task Count: 1` ระบุชัดเจนว่าถูกส่งไปแค่ 1 shard/worker เท่านั้น ส่วน scatter-gather query จะแสดง `Task Count` เท่ากับจำนวน shard ทั้งหมด (เช่น 32) พร้อมขั้นตอนการรวมผล (`MergeAggregate`, `Sort` เพิ่มเติม)

```sql
-- ตัวอย่างผลลัพธ์ EXPLAIN ของ router query (สรุปแนวคิด)
--  Custom Scan (Citus Adaptive)
--    Task Count: 1
--    Tasks Shown: All
--    ->  Task
--          Node: host=worker2.internal port=5432
--          ->  Index Scan using campaigns_pkey on campaigns_102045
--                Index Cond: (company_id = 42)

-- ตัวอย่างผลลัพธ์ EXPLAIN ของ scatter-gather query (สรุปแนวคิด)
--  Aggregate
--    ->  Custom Scan (Citus Adaptive)
--          Task Count: 32
--          Tasks Shown: One of 32
--          ->  Task
--                Node: host=worker1.internal port=5432
--                ->  Aggregate
--                      ->  Seq Scan on campaigns_102001
```

### ผลกระทบต่อการออกแบบ Application

```
┌─────────────────────────────────────────────────────────────┐
│        แนวปฏิบัติที่ดีเมื่อออกแบบ Application บน Citus         │
├─────────────────────────────────────────────────────────────┤
│                                                                 │
│  ✓ ทุก query ที่ใช้บ่อย (hot path) ควรรวม distribution         │
│    column ไว้ใน WHERE เสมอ (เช่น ทุก API request ที่มี          │
│    tenant context ต้องแนบ tenant_id ใน query)                  │
│                                                                 │
│  ✓ เก็บ query แบบ scatter-gather (analytics, reporting,        │
│    admin dashboard) ไว้เป็นกลุ่มที่ยอมรับ latency สูงกว่าได้    │
│    แยกจาก transactional hot path                               │
│                                                                 │
│  ✓ พิจารณาทำ materialized view หรือ pre-aggregation             │
│    สำหรับ query ที่ scatter-gather บ่อยและต้องการความเร็ว        │
│                                                                 │
│  ✓ Application layer ควรรู้จัก "tenant context" เสมอ           │
│    (middleware ที่ดึง tenant_id จาก session/JWT แล้วแนบ         │
│    เข้าไปในทุก query โดยอัตโนมัติ)                              │
│                                                                 │
└─────────────────────────────────────────────────────────────┘
```

---

## Step 964: Rebalancing — citus_rebalance_start()

### ทำไมต้อง Rebalance

เมื่อระบบเติบโตขึ้น สถานการณ์ต่อไปนี้อาจเกิดขึ้นได้:

1. **เพิ่ม worker node ใหม่** — worker ใหม่ที่เพิ่งเข้า cluster ยังไม่มี shard เลย ในขณะที่ worker เก่ามี shard เต็มไปหมด โหลดจึงไม่สมดุล
2. **Data skew เกิดขึ้นตามเวลา** — บาง shard โตเร็วกว่า shard อื่น (เช่น tenant บางรายเติบโตเร็วผิดปกติ)
3. **Worker node บางตัวมีปัญหา hardware** — ต้องการย้าย shard ออกจาก worker นั้นก่อน decommission

**Rebalancing** คือกระบวนการย้าย shard ระหว่าง worker node เพื่อให้โหลด (จำนวน shard, ขนาดข้อมูล) กระจายอย่างสมดุลมากขึ้น โดยไม่กระทบต่อ availability ของระบบ (shard ยังคง online ระหว่างการย้าย)

```
┌───────────────────────────────────────────────────────────────┐
│                      Rebalancing Concept                         │
│                                                                     │
│   ก่อน Rebalance:                                                  │
│   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              │
│   │  Worker 1   │  │  Worker 2   │  │  Worker 3   │  ← เพิ่งเพิ่ม│
│   │  16 shards  │  │  16 shards  │  │   0 shards  │              │
│   │  (เต็ม)     │  │  (เต็ม)     │  │  (ว่างเปล่า) │              │
│   └─────────────┘  └─────────────┘  └─────────────┘              │
│                                                                     │
│                          citus_rebalance_start()                  │
│                                    │                                │
│                                    ▼                                │
│   หลัง Rebalance:                                                   │
│   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              │
│   │  Worker 1   │  │  Worker 2   │  │  Worker 3   │              │
│   │  ~11 shards │  │  ~11 shards │  │  ~10 shards │              │
│   │  (สมดุล)    │  │  (สมดุล)    │  │  (สมดุล)    │              │
│   └─────────────┘  └─────────────┘  └─────────────┘              │
│                                                                     │
│   ระหว่างการย้าย shard ยังคง online (ใช้ logical replication      │
│   เพื่อ sync ข้อมูลไปยัง worker ปลายทางก่อน แล้วค่อย switch         │
│   traffic — คล้ายหลักการ near-zero-downtime migration)             │
│                                                                     │
└───────────────────────────────────────────────────────────────┘
```

### ขั้นตอนการ Rebalance

```sql
-- Step 1: เพิ่ม worker node ใหม่เข้า cluster ก่อน
SELECT citus_add_node('worker-node-4.internal', 5432);

-- Step 2: ดูแผนการ rebalance ล่วงหน้าก่อนรันจริง (dry-run)
SELECT * FROM get_rebalance_table_shards_plan('campaigns');

-- Step 3: เริ่มกระบวนการ rebalance จริง
-- shard_transfer_mode:
--   'auto'         - ให้ Citus เลือกวิธีที่เหมาะสมอัตโนมัติ
--   'force_logical'- ใช้ logical replication เสมอ (แนะนำสำหรับ
--                     production เพราะ downtime ต่ำที่สุด)
--   'block_writes' - บล็อก write ระหว่างย้าย (เร็วกว่าแต่มี downtime
--                     สั้น ๆ เหมาะกับ maintenance window)
SELECT citus_rebalance_start(
    rebalance_strategy := 'by_shard_count',
    shard_transfer_mode := 'force_logical'
);

-- Step 4: ติดตามความคืบหน้าของการ rebalance
SELECT * FROM citus_rebalance_status();

-- Step 5: หากต้องการหยุด rebalance ที่กำลังทำงานอยู่
SELECT citus_rebalance_stop();
```

### Rebalance Strategy

Citus รองรับกลยุทธ์การ rebalance หลายแบบ:

```sql
-- ดู strategy ที่มีให้เลือก
SELECT * FROM pg_dist_rebalance_strategy;
```

| Strategy | คำอธิบาย |
|---|---|
| `by_shard_count` | สมดุลตาม "จำนวน shard" ต่อ worker (ค่า default) เหมาะเมื่อ shard มีขนาดใกล้เคียงกัน |
| `by_disk_size` | สมดุลตาม "ขนาดข้อมูลจริง" (bytes) ต่อ worker เหมาะเมื่อมี data skew ระหว่าง shard |

```sql
-- ใช้ strategy ตามขนาด disk แทนจำนวน shard (เหมาะกับกรณีมี data skew)
SELECT citus_rebalance_start(
    rebalance_strategy := 'by_disk_size',
    shard_transfer_mode := 'force_logical'
);
```

### การย้าย Shard แบบเจาะจง (Manual Shard Move)

บางครั้งเราต้องการควบคุมการย้ายเอง แทนที่จะให้ algorithm ตัดสินใจอัตโนมัติทั้งหมด:

```sql
-- ย้าย shard ที่ระบุ id จาก worker ต้นทางไปยังปลายทางโดยตรง
SELECT citus_move_shard_placement(
    shard_id := 102045,
    source_node_name := 'worker-node-1.internal',
    source_node_port := 5432,
    target_node_name := 'worker-node-4.internal',
    target_node_port := 5432,
    shard_transfer_mode := 'force_logical'
);
```

### การถอด Worker Node ออกจาก Cluster อย่างปลอดภัย

```sql
-- Step 1: สั่งให้ Citus ย้าย shard ทั้งหมดออกจาก node นี้ก่อน
-- (drain = ระบายข้อมูลออก คล้ายการ cordon+drain node ใน Kubernetes
--  ที่เรียนใน Part 093)
SELECT citus_drain_node(
    'worker-node-1.internal',
    5432,
    shard_transfer_mode := 'force_logical'
);

-- Step 2: ตรวจสอบว่า node ไม่มี shard เหลืออยู่แล้ว
SELECT * FROM citus_shards WHERE nodename = 'worker-node-1.internal';
-- (ควรไม่มีแถวคืนกลับมา)

-- Step 3: ถอด node ออกจาก cluster metadata
SELECT citus_remove_node('worker-node-1.internal', 5432);
```

### ข้อควรระวังในการ Rebalance บน Production

```
┌─────────────────────────────────────────────────────────────┐
│              แนวปฏิบัติที่ดีเมื่อ Rebalance Production          │
├─────────────────────────────────────────────────────────────┤
│                                                                 │
│  ✓ ใช้ shard_transfer_mode = 'force_logical' เสมอใน            │
│    production เพื่อลด downtime ให้ใกล้ศูนย์ที่สุด               │
│                                                                 │
│  ✓ Rebalance ในช่วง traffic ต่ำ แม้จะ near-zero-downtime        │
│    เพราะการย้ายข้อมูลใช้ CPU/network bandwidth เพิ่มเติม        │
│    ที่อาจกระทบ latency ของ query อื่น ๆ                         │
│                                                                 │
│  ✓ Monitor citus_rebalance_status() อย่างต่อเนื่องระหว่าง       │
│    กระบวนการ (อาจใช้เวลาหลายชั่วโมงถ้าข้อมูลใหญ่มาก)             │
│                                                                 │
│  ✓ ทดสอบกระบวนการ rebalance บน staging environment ก่อน         │
│    เสมอ โดยเฉพาะเมื่อเพิ่ม worker node ครั้งแรก                  │
│                                                                 │
│  ✓ มี monitoring ติดตาม disk usage, replication lag ระหว่าง     │
│    rebalance (เชื่อมโยงกับ Part 070 เรื่อง Monitoring)          │
│                                                                 │
└─────────────────────────────────────────────────────────────┘
```

---

## Step 965: แบบฝึกหัดรวม — ออกแบบ Sharding Strategy สำหรับ E-commerce Multi-Tenant SaaS

### โจทย์

ทบทวนจาก Part 080 (Multi-tenancy Patterns) — เราจะออกแบบระบบ e-commerce แบบ **multi-tenant SaaS** ที่ให้บริการร้านค้าออนไลน์หลายร้าน (แต่ละร้านคือ 1 tenant) บนแพลตฟอร์มเดียวกัน ระบบมีลักษณะดังนี้:

- มีร้านค้า (tenant) รวมกันหลายหมื่นร้าน ขนาดแตกต่างกันมาก ตั้งแต่ร้านเล็ก ๆ ไปจนถึงร้านใหญ่ที่มีคำสั่งซื้อวันละหลายหมื่นรายการ
- แต่ละร้านมีตาราง: `stores` (ข้อมูลร้าน), `products` (สินค้า), `orders` (คำสั่งซื้อ), `order_items` (รายการสินค้าในคำสั่งซื้อ), `customers` (ลูกค้าของร้าน)
- มีตารางกลางที่ใช้ร่วมกันทุก tenant: `countries`, `currencies`, `shipping_carriers`
- Query ที่พบบ่อยที่สุด: "ดูคำสั่งซื้อทั้งหมดของร้าน X", "ดูสินค้าของร้าน X", "สร้างคำสั่งซื้อใหม่ให้ร้าน X"
- Query ที่พบน้อยกว่าแต่สำคัญ: "รายงานสรุปยอดขายรวมทุกร้าน" (สำหรับทีม platform เอง)

### การวิเคราะห์และออกแบบ

#### 1) เลือก Shard Key

```
┌─────────────────────────────────────────────────────────────┐
│                   วิเคราะห์ตัวเลือก Shard Key                │
├─────────────────────────────────────────────────────────────┤
│                                                                 │
│  ตัวเลือก A: store_id (tenant_id)                              │
│  ✓ Cardinality สูง (หลายหมื่นร้าน)                             │
│  ✓ ตรงกับ query pattern หลัก ("ดูข้อมูลของร้าน X")             │
│  ✓ Isolation ชัดเจนระหว่าง tenant (ความปลอดภัยของข้อมูล        │
│    เชื่อมโยงกับ Row-Level Security ใน Part 080 ได้ด้วย)         │
│  ✗ ร้านใหญ่มากอาจทำให้เกิด data skew (ต้องมีแผนรองรับ           │
│    ด้วย isolate_tenant_to_new_shard)                           │
│                                                                 │
│  ตัวเลือก B: customer_id                                       │
│  ✗ ลูกค้าคนหนึ่งอาจซื้อจากหลายร้าน → ข้อมูล order ของร้าน       │
│    เดียวกันกระจายไปคนละ shard กัน ทำให้ query "ดูคำสั่งซื้อ      │
│    ทั้งหมดของร้าน X" กลายเป็น scatter-gather เสมอ                │
│  ✗ ไม่สอดคล้องกับโมเดล multi-tenant                            │
│                                                                 │
│  สรุป: เลือก store_id เป็น shard key หลัก                       │
│                                                                 │
└─────────────────────────────────────────────────────────────┘
```

#### 2) ออกแบบ Co-location Group

```sql
-- 1. Root table ของ co-location group: stores
CREATE TABLE stores (
    id           bigserial,
    name         text NOT NULL,
    plan_tier    text NOT NULL DEFAULT 'starter',
    country_id   smallint,
    created_at   timestamptz NOT NULL DEFAULT now(),
    PRIMARY KEY (id)
);

SELECT create_distributed_table('stores', 'id');

-- 2. products: shard ด้วย store_id, co-located กับ stores
CREATE TABLE products (
    id           bigserial,
    store_id     bigint NOT NULL,
    sku          text NOT NULL,
    name         text NOT NULL,
    price_cents  integer NOT NULL,
    created_at   timestamptz NOT NULL DEFAULT now(),
    PRIMARY KEY (id, store_id)
);

SELECT create_distributed_table('products', 'store_id', colocate_with => 'stores');

-- 3. customers: shard ด้วย store_id เพราะลูกค้าถือว่าสังกัดร้าน
--    (ในโมเดล SaaS แบบ "ร้านค้าเป็นเจ้าของฐานลูกค้า" — ถ้าธุรกิจจริง
--     เป็นแบบ marketplace ที่ลูกค้าใช้ร่วมกันหลายร้าน ต้องออกแบบต่างออกไป)
CREATE TABLE customers (
    id           bigserial,
    store_id     bigint NOT NULL,
    email        text NOT NULL,
    full_name    text,
    created_at   timestamptz NOT NULL DEFAULT now(),
    PRIMARY KEY (id, store_id)
);

SELECT create_distributed_table('customers', 'store_id', colocate_with => 'stores');

-- 4. orders: shard ด้วย store_id, co-located
CREATE TABLE orders (
    id            bigserial,
    store_id      bigint NOT NULL,
    customer_id   bigint NOT NULL,
    status        text NOT NULL DEFAULT 'pending',
    total_cents   integer NOT NULL,
    created_at    timestamptz NOT NULL DEFAULT now(),
    PRIMARY KEY (id, store_id)
);

SELECT create_distributed_table('orders', 'store_id', colocate_with => 'stores');

-- 5. order_items: shard ด้วย store_id, co-located
--    (แม้ order_items จะ "อ้างอิง" order_id เป็นหลัก แต่การใส่
--     store_id ลงไปด้วยและใช้เป็น shard key ทำให้มันอยู่ worker
--     เดียวกับ orders ของร้านเดียวกันเสมอ)
CREATE TABLE order_items (
    id            bigserial,
    store_id      bigint NOT NULL,
    order_id      bigint NOT NULL,
    product_id    bigint NOT NULL,
    quantity      integer NOT NULL,
    unit_price_cents integer NOT NULL,
    PRIMARY KEY (id, store_id)
);

SELECT create_distributed_table('order_items', 'store_id', colocate_with => 'stores');
```

#### 3) ออกแบบ Reference Table

```sql
-- ตารางกลางที่ใช้ร่วมกันทุก tenant, ขนาดเล็ก, write น้อยมาก
CREATE TABLE countries (
    id        smallint PRIMARY KEY,
    iso_code  char(2) NOT NULL,
    name      text NOT NULL
);
SELECT create_reference_table('countries');

CREATE TABLE currencies (
    id      smallint PRIMARY KEY,
    code    char(3) NOT NULL,
    symbol  text
);
SELECT create_reference_table('currencies');

CREATE TABLE shipping_carriers (
    id      smallint PRIMARY KEY,
    name    text NOT NULL,
    api_url text
);
SELECT create_reference_table('shipping_carriers');
```

#### 4) ตรวจสอบภาพรวมของการออกแบบ

```sql
SELECT table_name, citus_table_type, distribution_column, colocation_id, shard_count
FROM citus_tables
ORDER BY colocation_id NULLS LAST, table_name;
```

ผลลัพธ์ที่คาดหวัง:

```
     table_name      | citus_table_type | distribution_column | colocation_id | shard_count
----------------------+------------------+----------------------+----------------+-------------
 customers            | distributed      | store_id             |              7 | 32
 order_items          | distributed      | store_id             |              7 | 32
 orders               | distributed      | store_id             |              7 | 32
 products             | distributed      | store_id             |              7 | 32
 stores               | distributed      | id                   |              7 | 32
 countries            | reference        | (none)               |           NULL |    1
 currencies           | reference        | (none)               |           NULL |    1
 shipping_carriers    | reference        | (none)               |           NULL |    1
```

#### 5) ตัวอย่าง Query ที่ได้ประโยชน์เต็มที่ (Router Query)

```sql
-- Query สร้างคำสั่งซื้อใหม่ — router query เขียนไป shard เดียว
BEGIN;
INSERT INTO orders (store_id, customer_id, status, total_cents)
VALUES (777, 5001, 'pending', 129900)
RETURNING id;
-- สมมติได้ order id = 88801

INSERT INTO order_items (store_id, order_id, product_id, quantity, unit_price_cents)
VALUES
    (777, 88801, 4021, 2, 49900),
    (777, 88801, 4033, 1, 30100);
COMMIT;
-- Transaction ทั้งหมดนี้เกิดขึ้นบน worker เดียว (worker ที่ store_id=777
-- ถูก hash ไปอยู่) เพราะทุกตารางใช้ store_id เป็น shard key เดียวกัน
-- และ co-located กันหมด — ได้ทั้งความเร็วและ ACID guarantee เต็มรูปแบบ

-- Query "ดูคำสั่งซื้อของร้าน X พร้อมรายการสินค้า" — router query
-- ที่ join 3 ตาราง (orders, order_items, products) แบบ local
SELECT o.id, o.status, o.total_cents,
       oi.product_id, p.name, oi.quantity, oi.unit_price_cents
FROM orders o
JOIN order_items oi ON oi.order_id = o.id AND oi.store_id = o.store_id
JOIN products p ON p.id = oi.product_id AND p.store_id = o.store_id
WHERE o.store_id = 777
ORDER BY o.created_at DESC
LIMIT 50;
```

#### 6) ตัวอย่าง Query ที่หลีกเลี่ยงไม่ได้ (Scatter-Gather) และวิธีรับมือ

```sql
-- Query สำหรับทีม platform: รายงานยอดขายรวมทุกร้าน
-- (ไม่มี filter ด้วย store_id → scatter-gather เสมอ)
SELECT DATE_TRUNC('day', created_at) AS day,
       SUM(total_cents) AS revenue_cents,
       COUNT(*) AS order_count
FROM orders
WHERE created_at >= now() - interval '30 days'
GROUP BY 1
ORDER BY 1;
```

สำหรับ query ประเภทนี้ที่ไม่หลีกเลี่ยง scatter-gather ได้ (เพราะ business requirement ต้องการภาพรวมข้าม tenant) แนวทางที่ใช้กันจริงคือ:

1. รันเป็น **scheduled batch job** (เช่น ทุกคืน) แทนการ query real-time ทุกครั้งที่มีคนเปิด dashboard
2. เก็บผลลัพธ์ไว้ใน **summary table** (materialized) ที่ update เป็นระยะ — คล้ายแนวคิด materialized view ที่เรียนใน Part 051
3. ถ้าต้องการ real-time analytics จริง ๆ ให้พิจารณาส่งข้อมูลไปยัง data warehouse แยกต่างหาก (เช่นผ่าน CDC/logical replication ที่เรียนใน Part 076-078) แทนที่จะ query จาก transactional cluster โดยตรง

#### 7) แผนรองรับ Data Skew (Whale Tenant)

```sql
-- ตรวจสอบ tenant ที่มีข้อมูลมากผิดปกติเป็นระยะ
SELECT cs.shardid, cs.nodename, cs.shard_size
FROM citus_shards cs
WHERE cs.table_name = 'orders'::regclass
ORDER BY cs.shard_size DESC
LIMIT 5;

-- หากพบว่าร้านค้ารายใหญ่ (store_id = 999) แชร์ shard กับร้านอื่น
-- และทำให้ shard นั้นใหญ่ผิดปกติ ให้แยกออกเป็น shard เฉพาะของตัวเอง
SELECT isolate_tenant_to_new_shard('orders', 999, 'CASCADE');
```

### สรุปกลยุทธ์การออกแบบทั้งหมด

```
┌───────────────────────────────────────────────────────────────┐
│      สรุป Sharding Strategy สำหรับ E-commerce Multi-Tenant       │
├───────────────────────────────────────────────────────────────┤
│                                                                     │
│  Shard Key:        store_id (tenant_id)                           │
│  Co-location Group: stores, products, customers, orders,          │
│                      order_items (ทั้งหมดใช้ store_id)             │
│  Reference Tables:  countries, currencies, shipping_carriers      │
│  Hot Path Queries:  router query เสมอ (มี store_id ใน WHERE)      │
│  Cold Path Queries: scatter-gather (platform-level reporting)     │
│                      → ย้ายไป batch job / summary table /          │
│                        data warehouse แยกต่างหาก                   │
│  Data Skew Plan:    monitor shard size เป็นระยะ, ใช้              │
│                      isolate_tenant_to_new_shard สำหรับ            │
│                      whale tenant                                  │
│  Growth Plan:       ตั้ง shard_count สูงกว่าจำนวน worker           │
│                      ปัจจุบันหลายเท่า, ใช้ citus_rebalance_start   │
│                      เมื่อเพิ่ม worker node ใหม่                    │
│                                                                     │
└───────────────────────────────────────────────────────────────┘
```

โจทย์นี้แสดงให้เห็นว่าการออกแบบ Citus sharding ไม่ใช่แค่การเรียกคำสั่ง `create_distributed_table()` แต่ต้องคิดตลอดทั้ง pipeline ตั้งแต่การเลือก shard key, การออกแบบ schema ให้ co-locate กันได้, การแยกประเภท query ที่จะเป็น router กับ scatter-gather, ไปจนถึงแผนรองรับการเติบโตและ data skew ในระยะยาว

---

## สรุปท้ายบท

ในบทนี้เราได้เรียนรู้การขยาย PostgreSQL ให้เป็น distributed database ด้วย Citus:

- **ข้อจำกัดของ single-node PostgreSQL** เกิดขึ้นเมื่อข้อมูลหรือ write throughput เกินความสามารถของเครื่องเดียว ไม่ว่าจะ vertical scale หรือเพิ่ม read replica แค่ไหนก็ไม่พอ
- **Sharding** คือการกระจายข้อมูลข้าม node ตาม shard key ซึ่งต่างจาก **Partitioning** (Part 054) ที่ยังอยู่ในเครื่องเดียว — ทั้งสองแนวคิดสามารถใช้ร่วมกันได้ในระบบจริง
- **Citus** คือ PostgreSQL extension ที่แปลง instance ให้กลายเป็นส่วนหนึ่งของ distributed cluster โดยยังคงใช้ SQL, driver, extension ecosystem เดิมได้
- สถาปัตยกรรมประกอบด้วย **Coordinator Node** (เก็บ metadata, วางแผน query, รวมผล) และ **Worker Node** (เก็บ shard จริง, ประมวลผล local)
- **Distributed Table** สร้างด้วย `create_distributed_table()` โดยการเลือก distribution column ที่มี cardinality สูง กระจายสม่ำเสมอ และตรงกับ query pattern คือหัวใจของการออกแบบทั้งระบบ
- **Reference Table** สร้างด้วย `create_reference_table()` สำหรับตารางเล็กที่ต้อง copy ครบทุก worker เพื่อให้ JOIN ทำงานแบบ local ได้
- **Co-location** คือการให้ตารางที่เกี่ยวข้องกันใช้ shard key เดียวกัน ทำให้ JOIN เป็น router query แทนที่จะเป็น repartition join ที่ช้ากว่ามาก
- **Query Routing** แบ่งเป็น router query (ไปที่ worker เดียว, เร็ว) กับ scatter-gather query (กระจายไปทุก worker แล้วรวมผล, ช้ากว่าแต่บางครั้งจำเป็น)
- **Rebalancing** ผ่าน `citus_rebalance_start()` ใช้เมื่อเพิ่ม worker ใหม่หรือโหลดไม่สมดุล โดยรองรับการย้ายแบบ near-zero-downtime ด้วย logical replication

การออกแบบระบบ multi-tenant SaaS ด้วย Citus แสดงให้เห็นภาพรวมทั้งหมด: เลือก tenant_id เป็น shard key, ทำให้ทุกตารางที่เกี่ยวข้อง co-locate กัน, แยก query แบบ router ออกจาก scatter-gather อย่างชัดเจน และมีแผนรองรับการเติบโตและ data skew ล่วงหน้า

Citus เหมาะสำหรับองค์กรที่ต้องการอยู่ใน PostgreSQL ecosystem แต่ต้องการ horizontal scaling ที่แท้จริง — เป็นทางเลือกที่ยังคงความคุ้นเคยของ SQL และเครื่องมือ PostgreSQL ที่เราเรียนมาตลอดทั้งหลักสูตรไว้ได้อย่างเต็มที่

ในบทถัดไป (Part 098) เราจะเรียนรู้การใช้ `pgbench` เพื่อทำ **Benchmarking** วัดประสิทธิภาพของ PostgreSQL อย่างเป็นระบบ ซึ่งเป็นทักษะสำคัญที่ใช้ตัดสินใจได้ว่าระบบของเรา "ถึงเวลาต้อง scale" ตามที่พูดถึงใน Step 956 หรือยัง

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1

จงอธิบายความแตกต่างระหว่าง Sharding และ Partitioning โดยเน้นที่ "ตำแหน่งทางกายภาพของข้อมูล" เป็นหลัก

<details>
<summary>เฉลย</summary>

**Partitioning** (Part 054) คือการแบ่งตารางใหญ่ออกเป็นตารางย่อย (partition) หลายตาราง แต่ทุก partition ยังคงอยู่ใน **PostgreSQL instance เดียวกัน** ใช้ CPU, RAM, disk ของเครื่องเดียวร่วมกันทั้งหมด จุดประสงค์หลักคือทำให้ query เร็วขึ้นด้วย partition pruning และทำให้งาน maintenance (VACUUM, index rebuild) แบ่งทำเป็นส่วนย่อยได้

**Sharding** (Citus) คือการแบ่งตารางออกเป็น shard ที่กระจายไปอยู่ **คนละเครื่อง (node) กัน** แต่ละ shard ใช้ CPU, RAM, disk ของ node ตัวเอง แยกขาดจาก node อื่น จุดประสงค์หลักคือแก้ปัญหาที่เครื่องเดียวไม่พอรองรับข้อมูลหรือ throughput ได้อีกต่อไป (แก้ปัญหาด้าน compute/storage capacity ไม่ใช่แค่ query performance)

ในระบบจริงมักใช้ทั้งคู่ร่วมกัน: Citus sharding กระจายข้อมูลข้าม worker node ตาม tenant_id แล้วภายในแต่ละ worker ก็ยังทำ time-based partitioning ได้อีกชั้นหนึ่ง

</details>

### แบบฝึกหัดที่ 2

ทำไม Citus ถึงถูกเรียกว่าเป็น "extension" ไม่ใช่ database แยกต่างหาก และข้อดีของแนวทางนี้คืออะไร

<details>
<summary>เฉลย</summary>

Citus ติดตั้งผ่าน `CREATE EXTENSION citus;` เหมือน extension ทั่วไปของ PostgreSQL (เช่น `pg_stat_statements`, `postgis`) ไม่ใช่การ fork PostgreSQL core หรือเขียน database engine ใหม่ทั้งหมด มันครอบชั้นของ "การกระจายข้อมูล" (distributed layer) ไว้บน PostgreSQL engine เดิม โดยที่ query planner, executor, storage engine, WAL, MVCC ในระดับ node เดียวยังเป็นของ PostgreSQL แท้ ๆ ทั้งหมด

ข้อดีของแนวทางนี้คือทุกอย่างที่เราคุ้นเคยจาก PostgreSQL ยังใช้ได้เหมือนเดิม: SQL มาตรฐาน, driver ภาษาต่าง ๆ (psycopg2, pgx, JDBC), client tools (psql, pgAdmin), extension อื่น ๆ ที่ทำงานร่วมกันได้ (PostGIS, pgvector) และความรู้เรื่อง indexing, tuning, replication ที่เรียนมาตลอดหลักสูตรยังคงนำมาใช้ได้โดยตรง แทนที่จะต้องเรียนรู้ระบบใหม่ทั้งหมดเหมือนเปลี่ยนไปใช้ distributed database ตัวอื่น

</details>

### แบบฝึกหัดที่ 3

Coordinator Node และ Worker Node ใน Citus มีหน้าที่ต่างกันอย่างไร จงยกตัวอย่างข้อมูลที่ coordinator เก็บไว้ (metadata) อย่างน้อย 2 อย่าง

<details>
<summary>เฉลย</summary>

**Coordinator Node** ทำหน้าที่เป็นสมองส่วนกลาง: รับ query จาก client, เก็บ metadata ของการกระจายข้อมูลทั้งหมด, วางแผนว่า query ต้องไปที่ shard ไหนบ้าง (query routing), ทำหน้าที่เป็น 2PC coordinator สำหรับ transaction ข้าม node, และรวมผล (aggregate) จากหลาย worker กลับมาเป็นผลลัพธ์เดียว

**Worker Node** คือ PostgreSQL instance ปกติที่เก็บข้อมูลจริง (shard) และประมวลผล query แบบ local บน shard ที่ตัวเองรับผิดชอบ ไม่รู้จักภาพรวมทั้งหมดของ cluster

ตัวอย่าง metadata ที่ coordinator เก็บ:
- `pg_dist_partition` — บอกว่าตารางไหนถูก distribute แล้ว และ distribution column คืออะไร
- `pg_dist_shard` — บอก shard แต่ละอันมี id อะไร และ hash range เท่าไหร่
- `pg_dist_placement` — บอกว่า shard แต่ละอันอยู่ worker node ไหน
- `pg_dist_node` — รายชื่อ worker node ทั้งหมดใน cluster

</details>

### แบบฝึกหัดที่ 4

จงเขียนคำสั่ง SQL เพื่อสร้างตาราง `subscriptions` ที่มีคอลัมน์ `id`, `tenant_id`, `plan_name`, `started_at` แล้วแปลงให้เป็น distributed table โดยใช้ `tenant_id` เป็น distribution column

<details>
<summary>เฉลย</summary>

```sql
CREATE TABLE subscriptions (
    id          bigserial,
    tenant_id   bigint NOT NULL,
    plan_name   text NOT NULL,
    started_at  timestamptz NOT NULL DEFAULT now(),
    PRIMARY KEY (id, tenant_id)
);

SELECT create_distributed_table('subscriptions', 'tenant_id');
```

หมายเหตุ: primary key ต้องเป็น composite key ที่รวม distribution column (`tenant_id`) ไว้ด้วย เพราะ Citus ต้องการให้ unique constraint ตรวจสอบได้ภายใน shard เดียว (ไม่ต้องตรวจสอบข้าม node)

</details>

### แบบฝึกหัดที่ 5

Reference Table แตกต่างจาก Distributed Table อย่างไร และควรใช้ Reference Table กับตารางแบบไหน

<details>
<summary>เฉลย</summary>

Distributed Table ถูก **แบ่ง** ข้อมูลออกเป็นหลาย shard กระจายไปคนละ worker (แต่ละแถวอยู่แค่ 1 shard) ส่วน Reference Table ถูก **copy เต็มรูปแบบ (replicate)** ไปยังทุก worker node — ทุก worker มีข้อมูลครบทุกแถวเหมือนกันหมด

ควรใช้ Reference Table กับตารางที่: (1) มีขนาดเล็ก (2) แทบไม่เปลี่ยนแปลง (read-heavy, write น้อยมาก) (3) ถูก JOIN บ่อยกับตารางที่ distribute อยู่ เช่น `countries`, `currencies`, `product_categories`, `shipping_carriers` การทำแบบนี้ทำให้ JOIN ทำได้แบบ local บนทุก worker โดยไม่ต้องส่งข้อมูลข้าม network

ข้อควรระวัง: การ write บน reference table ต้องเขียนไปทุก worker พร้อมกันผ่าน 2PC จึงมีค่าใช้จ่ายสูงกว่าถ้า write บ่อย ไม่ควรใช้กับตารางที่มี write ถี่

</details>

### แบบฝึกหัดที่ 6

จงอธิบายว่า Co-location คืออะไร และเขียนคำสั่งสร้างตาราง `invoices` ที่ shard ด้วย `company_id` และ co-locate กับตาราง `companies` ที่มีอยู่แล้ว

<details>
<summary>เฉลย</summary>

Co-location คือการออกแบบให้ตารางหลายตารางที่มักถูก JOIN กันบ่อย ๆ ใช้ distribution column เดียวกันและมีจำนวน shard เท่ากัน ทำให้ Citus วางแถวที่มีค่า distribution column เดียวกันจากทุกตารางไว้บน worker node เดียวกันเสมอ ส่งผลให้การ JOIN ระหว่างตารางเหล่านี้ทำได้แบบ local (router query) โดยไม่ต้องส่งข้อมูลข้าม network

```sql
CREATE TABLE invoices (
    id            bigserial,
    company_id    bigint NOT NULL,
    amount_cents  integer NOT NULL,
    issued_at     timestamptz NOT NULL DEFAULT now(),
    PRIMARY KEY (id, company_id)
);

SELECT create_distributed_table(
    'invoices',
    'company_id',
    colocate_with => 'companies'
);
```

</details>

### แบบฝึกหัดที่ 7

Router Query กับ Scatter-Gather Query ต่างกันอย่างไร จงยกตัวอย่าง query แบบละ 1 ตัวอย่างจากตาราง `orders` ที่ shard ด้วย `store_id`

<details>
<summary>เฉลย</summary>

**Router Query** คือ query ที่มี WHERE clause ระบุค่าของ distribution column แบบ exact match ทำให้ Citus รู้ทันทีว่าต้องส่งไป worker ไหน worker เดียว ประมวลผล local ทั้งหมดแล้วส่งผลกลับ — เร็วที่สุด

```sql
SELECT * FROM orders WHERE store_id = 777;
```

**Scatter-Gather Query** คือ query ที่ไม่ได้ filter ด้วย distribution column แบบ exact match ทำให้ Citus ต้องส่ง query ไปทุก worker แล้วรวมผลกลับมาที่ coordinator — ช้ากว่าเพราะต้อง fan-out ไปทุก shard

```sql
SELECT store_id, SUM(total_cents) AS revenue
FROM orders
GROUP BY store_id
ORDER BY revenue DESC
LIMIT 10;
```

</details>

### แบบฝึกหัดที่ 8

เมื่อไรที่ควรทำ Rebalancing และคำสั่งหลักที่ใช้เริ่มกระบวนการนี้คืออะไร พร้อมอธิบายความหมายของ parameter `shard_transfer_mode`

<details>
<summary>เฉลย</summary>

ควรทำ Rebalancing เมื่อ: (1) เพิ่ม worker node ใหม่เข้า cluster และต้องการย้าย shard บางส่วนไปให้ worker ใหม่รับโหลดด้วย (2) เกิด data skew ที่ทำให้บาง worker มีข้อมูล/โหลดมากกว่า worker อื่นอย่างเห็นได้ชัด (3) ต้องการถอด worker node บางตัวออกจาก cluster

คำสั่งหลักคือ:

```sql
SELECT citus_rebalance_start(
    rebalance_strategy := 'by_shard_count',
    shard_transfer_mode := 'force_logical'
);
```

`shard_transfer_mode` กำหนดวิธีการย้ายข้อมูล:
- `'auto'` — ให้ Citus เลือกวิธีที่เหมาะสมเองอัตโนมัติ
- `'force_logical'` — ใช้ logical replication เสมอ sync ข้อมูลไปยังปลายทางก่อนแล้วค่อย switch traffic ทำให้ downtime ต่ำที่สุด เหมาะกับ production
- `'block_writes'` — บล็อก write ระหว่างย้าย เร็วกว่าแต่มี downtime สั้น ๆ เหมาะกับช่วง maintenance window

ติดตามความคืบหน้าด้วย `SELECT * FROM citus_rebalance_status();`

</details>

### แบบฝึกหัดที่ 9

บริษัทแห่งหนึ่งออกแบบ Citus cluster โดยเลือก `created_at` (วันเวลาที่สร้างแถว) เป็น distribution column ของตาราง `orders` จงวิเคราะห์ว่าการเลือกนี้มีปัญหาอะไรบ้าง และควรเลือกคอลัมน์ไหนแทน

<details>
<summary>เฉลย</summary>

ปัญหาของการเลือก `created_at` เป็น distribution column:

1. **Query ส่วนใหญ่ไม่ filter ด้วยค่าที่แน่นอนของ created_at** — โดยทั่วไป query จะใช้ range (`created_at BETWEEN ...`) หรือไม่ filter ด้วย created_at เลย ทำให้เกือบทุก query กลายเป็น scatter-gather ไม่ใช่ router query
2. **Hot shard ที่ข้อมูลใหม่กระจุกตัว** — ถ้า distribution เป็น range-based ตามเวลา ข้อมูลที่เพิ่งสร้างใหม่ (ซึ่งถูก query/write บ่อยที่สุด) จะกระจุกอยู่ shard เดียวหรือไม่กี่ shard ทำให้ worker ที่ถือ shard ล่าสุดรับโหลดหนักผิดปกติ ในขณะที่ worker ที่ถือ shard เก่าแทบไม่มีโหลด
3. **ไม่สอดคล้องกับ business entity** — created_at ไม่มีความหมายเชิงธุรกิจที่ใช้ทำ co-location กับตารางอื่นได้ (ต่างจาก store_id หรือ tenant_id)

ควรเลือกคอลัมน์ที่สะท้อน business entity หลักที่ query pattern ใช้บ่อยที่สุด เช่น `store_id` (ถ้าเป็นระบบ multi-tenant) หรือ `customer_id` และควรใช้ `created_at` เป็นคอลัมน์สำหรับทำ **native partitioning** (Part 054) ภายในแต่ละ shard แทน ซึ่งเป็นการผสมผสาน sharding (ข้าม node) กับ partitioning (ภายใน node) เข้าด้วยกันตามที่กล่าวถึงใน Step 957

</details>

### แบบฝึกหัดที่ 10

จากโจทย์ e-commerce multi-tenant SaaS ใน Step 965 จงอธิบายว่าทำไมตาราง `order_items` ถึงต้องมีคอลัมน์ `store_id` แม้ว่าในเชิง business logic มันจะเชื่อมโยงกับ `orders` ผ่าน `order_id` อยู่แล้วก็ตาม

<details>
<summary>เฉลย</summary>

แม้ว่าในเชิง business logic `order_items` จะเชื่อมโยงกับ `orders` ผ่าน `order_id` เพียงพอแล้ว (ในฐานข้อมูลปกติแบบ single-node ไม่จำเป็นต้องมี store_id ซ้ำ) แต่ในบริบทของ Citus การมี `store_id` ในตาราง `order_items` และใช้เป็น distribution column มีความจำเป็นเพื่อทำ **co-location** กับตาราง `orders`

ถ้า `order_items` ไม่มี `store_id` และต้อง shard ด้วยคอลัมน์อื่น (เช่น `id` ของตัวเอง หรือ `order_id`) Citus จะไม่สามารถรับประกันได้ว่าแถวใน `order_items` ที่เกี่ยวข้องกับ order หนึ่ง ๆ จะอยู่ worker เดียวกับแถว `orders` ที่เกี่ยวข้อง การ JOIN ระหว่างสองตารางนี้จะกลายเป็น repartition join ที่ต้องส่งข้อมูลข้าม network ระหว่าง worker ซึ่งช้ากว่า co-located join มาก

การเติม `store_id` ลงใน `order_items` และใช้ `colocate_with => 'stores'` (ผ่าน distribution column เดียวกันคือ store_id) ทำให้ Citus การันตีว่าแถวของ order_items ที่มี store_id เดียวกับ orders จะถูกวางไว้บน worker เดียวกันเสมอ ทำให้ query ที่ join `orders` กับ `order_items` (โดย filter ด้วย store_id) กลายเป็น router query ที่ทำงานแบบ local ทั้งหมด — นี่คือเหตุผลว่าทำไมในการออกแบบ schema สำหรับ Citus บางครั้งจึงต้อง "denormalize" เพิ่มคอลัมน์ shard key ลงในตารางลูกที่ปกติแล้วไม่จำเป็นต้องมี

</details>

---

**บทถัดไป**: [Part 098 — pgbench และ Benchmarking](./part-098-pgbench-benchmarking.md)
