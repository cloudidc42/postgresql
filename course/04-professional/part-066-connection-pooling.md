# Connection Pooling: PgBouncer, Pgpool-II

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับมืออาชีพ | Part 066

---

## เป้าหมายการเรียนรู้

เมื่อเรียนจบบทนี้ ผู้เรียนจะสามารถ:

1. อธิบายได้ว่าทำไม PostgreSQL ถึง "แพง" ในการเปิด connection ใหม่ทุกครั้ง เพราะสถาปัตยกรรมแบบ process-per-connection และจะเกิดผลอย่างไรกับระบบที่มี traffic สูงอย่างเว็บอีคอมเมิร์ซ
2. เข้าใจข้อจำกัดของการแก้ปัญหาด้วยการเพิ่ม `max_connections` แบบไม่จำกัด และผลกระทบต่อ memory และ context switching
3. ติดตั้งและตั้งค่า **PgBouncer** ตั้งแต่ศูนย์ ทั้งไฟล์ `pgbouncer.ini` และ `userlist.txt`
4. เลือก pool mode ที่เหมาะสม (session / transaction / statement) ตามลักษณะการใช้งานของแอปพลิเคชัน พร้อมเข้าใจข้อจำกัดของแต่ละโหมด
5. เข้าใจว่า **Pgpool-II** ทำอะไรได้มากกว่า PgBouncer (query routing, load balancing, replication, watchdog) และเมื่อไหร่ควรเลือกใช้
6. ใช้ admin console ของ PgBouncer (`SHOW POOLS`, `SHOW STATS`, `SHOW CLIENTS`, `SHOW SERVERS`) เพื่อ monitor สุขภาพของ pool
7. ออกแบบ connection pooling layer ที่เหมาะสมสำหรับระบบอีคอมเมิร์ซที่มี traffic สูง โดยพิจารณาทั้งเรื่อง availability, scalability และ operational complexity

---

## บริบท: ฐานข้อมูลอีคอมเมิร์ซของเรา

ตลอดบทนี้เราจะใช้ฐานข้อมูล `ecommerce_db` ที่มีตารางหลักคือ `customers`, `products`, `orders`, `order_items`, `payments`, `inventory` เป็นตัวอย่าง สมมติสถาปัตยกรรมของระบบเป็นดังนี้:

- **Web/API servers**: มี 8 instances รันบน container แต่ละตัวสร้าง connection pool ฝั่งแอป (เช่น HikariCP, node-postgres pool, SQLAlchemy pool) ขนาด 20-50 connections ต่อ instance
- **Background workers**: มี 4 instances สำหรับประมวลผล order queue, email notification, inventory sync
- **Reporting/BI**: มี dashboard ที่ query ฐานข้อมูลแบบ ad-hoc เป็นระยะ
- **PostgreSQL primary**: 1 เครื่องหลัก พร้อม read replica 2 เครื่อง

ถ้าคำนวณคร่าวๆ: 8 web servers × 50 connections + 4 workers × 20 + reporting อีกหลักสิบ = อาจแตะ **500-600 connections พร้อมกัน** เข้าสู่ PostgreSQL โดยตรง นี่คือจุดเริ่มต้นของปัญหาที่บทนี้จะแก้ไข

---

## Step 651: ทำไมต้อง Connection Pooling

### สถาปัตยกรรมแบบ process-per-connection ของ PostgreSQL

PostgreSQL ใช้สถาปัตยกรรมแบบ **multi-process** ไม่ใช่ multi-thread เหมือนฐานข้อมูลบางตัว (เช่น MySQL ที่ใช้ thread-per-connection ได้) ทุกครั้งที่มี client เชื่อมต่อเข้ามาใหม่ กระบวนการทำงานคือ:

1. Client ส่ง TCP connection request มาที่ `postmaster` (main process)
2. `postmaster` ทำการ `fork()` process ลูกใหม่ทั้งหมดขึ้นมา 1 ตัว เรียกว่า **backend process**
3. Backend process นี้จะรับผิดชอบ connection นี้ไปตลอดอายุของมัน — ทำ authentication, จัดสรร memory ส่วนตัว (เช่น `work_mem`), เปิด/ปิด transaction, รัน query ทั้งหมด

> หมายเหตุ: รายละเอียดเชิงลึกเกี่ยวกับสถาปัตยกรรม process และ memory ของ PostgreSQL (shared buffers, background processes, WAL writer ฯลฯ) จะกล่าวถึงอย่างละเอียดใน **Part 081**

### ต้นทุนของการเปิด connection ใหม่ทุกครั้ง

การ `fork()` process ใหม่ทุกครั้งไม่ใช่ของฟรี มีต้นทุนหลายด้าน:

| ต้นทุน | รายละเอียด |
|---|---|
| **CPU สำหรับ fork()** | การสร้าง process ใหม่ต้อง copy page table, file descriptor table ฯลฯ ใช้เวลาระดับ millisecond แต่ถ้าเกิดถี่มากๆ (หลายร้อย/วินาที) จะกินซีพียูจริงจัง |
| **Memory ต่อ connection** | แต่ละ backend process จองหน่วยความจำส่วนตัวขั้นต่ำหลาย MB (stack, local buffers, catalog cache) ยังไม่รวม `work_mem` ที่อาจถูกใช้หลายครั้งต่อ query ซับซ้อน |
| **Authentication overhead** | ทุก connection ใหม่ต้องผ่านกระบวนการ auth (เช่น SCRAM-SHA-256 handshake) ซึ่งมี computational cost ไม่น้อย |
| **Context switching** | เมื่อจำนวน process ที่ active พร้อมกันมากเกินจำนวน CPU core มากๆ OS ต้องสลับ context บ่อยขึ้น ทำให้ throughput รวมลดลง |
| **Catalog cache แยกต่อ process** | connection ใหม่ทุกตัวต้อง populate system catalog cache ของตัวเอง (ไม่ share กับ process อื่น) ทำให้ query แรกๆ ของ connection ช้ากว่าที่ควร |

### ทำไมมันสำคัญกับ e-commerce

ลองนึกภาพช่วง **flash sale** ของร้านค้าออนไลน์ที่เราใช้เป็นตัวอย่าง — มี traffic พุ่งขึ้นหลายเท่าใน 1-2 นาที ถ้าแอปพลิเคชันแต่ละ request สร้าง connection ใหม่ไปที่ PostgreSQL โดยตรง (short-lived connection pattern ที่พบบ่อยในบาง framework หรือ serverless function) จะเกิดปัญหา:

- Connection storm: หลายร้อย/พันการเชื่อมต่อใหม่ในเวลาไล่เลี่ยกัน ทำให้ `postmaster` ยุ่งกับการ fork process จนตอบสนอง query จริงช้าลง
- Backend process ที่ idle รออยู่ (เช่น connection ที่แอปเปิดค้างแต่ไม่ได้ใช้งาน) ยังคงกิน memory และนับรวมใน `max_connections` limit
- Latency ของ query แรกในแต่ละ connection สูงกว่าปกติเพราะต้อง warm-up cache

Connection pooling แก้ปัญหานี้โดยการ **นำ connection ที่มีอยู่แล้วกลับมาใช้ซ้ำ (reuse)** แทนที่จะสร้างและทำลายใหม่ทุกครั้ง ทำให้:

- ลดจำนวน physical connection ที่ PostgreSQL ต้อง fork process จริง
- ลด latency ของแต่ละ request เพราะไม่ต้องรอ authentication handshake ใหม่
- ควบคุมจำนวน concurrent connection ไปยัง PostgreSQL ให้อยู่ในระดับที่ database รับไหว โดยไม่จำกัดจำนวน client ที่แอปพลิเคชันรองรับได้

```
[ไม่มี pooling]
Web Server 1 ──┐
Web Server 2 ──┼── (500+ connections พร้อมกัน) ──> PostgreSQL (fork 500+ backend process)
Web Server 8 ──┘

[มี pooling]
Web Server 1 ──┐
Web Server 2 ──┼──> PgBouncer (pool 500+ client conn) ──> PostgreSQL (แค่ 20-50 backend process)
Web Server 8 ──┘
```

---

## Step 652: max_connections และข้อจำกัด

### ทำไมไม่เพิ่ม max_connections ไปเลยเยอะๆ

พารามิเตอร์ `max_connections` ใน `postgresql.conf` กำหนดจำนวน connection สูงสุดที่ PostgreSQL รับได้พร้อมกัน ค่า default คือ 100 หลายคนเมื่อเจอ error `FATAL: too many connections for role` หรือ `FATAL: sorry, too many clients already` มักจะแก้ปัญหาด้วยการเพิ่มค่านี้ให้สูงขึ้นเรื่อยๆ เช่น 500, 1000, 2000

แต่นี่ไม่ใช่คำตอบที่ดีเสมอไป ด้วยเหตุผลดังนี้:

#### 1. Memory overhead แบบ linear

แต่ละ connection ที่เปิดไว้ (ไม่ว่าจะ active หรือ idle) กินหน่วยความจำอย่างน้อย:

```
memory ต่อ connection ≈ (stack + local buffers + catalog cache warm)
                       ≈ 5-10 MB (ขั้นต่ำ แบบไม่มี query หนัก)
```

ถ้าตั้ง `max_connections = 2000` และทุก connection ถูกใช้งานจริง อาจต้องการ RAM สำรองไว้สูงถึงหลาย GB เฉพาะสำหรับ connection overhead เพียงอย่างเดียว ยังไม่นับ `work_mem` ที่แต่ละ query อาจใช้ (และ query ที่ซับซ้อนอาจใช้ `work_mem` หลายก้อนพร้อมกันสำหรับแต่ละ sort/hash node)

```
worst-case work_mem usage = max_connections × work_mem × (จำนวน sort/hash operations ต่อ query)
```

ถ้า `work_mem = 4MB` และมี query ที่ใช้ 3 operations พร้อมกัน:

```
2000 connections × 4MB × 3 = 24,000 MB = 24 GB (แค่สำหรับ work_mem อย่างเดียว!)
```

#### 2. Context switching และ CPU contention

เมื่อจำนวน **active** connection (ที่กำลังรัน query จริง ไม่ใช่ idle) เกินจำนวน CPU core มากๆ ประสิทธิภาพโดยรวมจะ**ลดลง** ไม่ใช่เพิ่มขึ้น เพราะ OS scheduler ต้องสลับ context ระหว่าง process บ่อยขึ้น มีงานวิจัยและ benchmark จาก community (เช่นจากทีม PgBouncer และบทความของ Bruce Momjian) ที่แสดงให้เห็นว่า throughput (TPS) มักจะ**พีคที่จำนวน active connection ใกล้เคียงกับ 2-4 เท่าของจำนวน CPU core** แล้วหลังจากนั้นจะเริ่ม**ลดลง**เมื่อเพิ่ม connection ต่อไป

```
Throughput
    ▲
    │        ___
    │      /     \___
    │    /            \___
    │  /                   \____
    │/                           \___
    └──────────────────────────────────> จำนวน active connections
       (sweet spot ~2-4x CPU core)
```

#### 3. Lock contention และ snapshot overhead

PostgreSQL ใช้ MVCC (Multi-Version Concurrency Control) ทุก transaction ต้องมองเห็น "snapshot" ของข้อมูล ยิ่งมี active connection/transaction เยอะพร้อมกัน ยิ่งมีโอกาสเกิด:

- Lock contention บน row ที่ถูกแก้ไขพร้อมกันบ่อย (เช่นตาราง `inventory` ที่ถูก update stock ตลอดเวลาช่วง flash sale)
- `ProcArrayLock` contention เมื่อระบบต้อง maintain snapshot ของ transaction จำนวนมาก
- Bloat สะสมเร็วขึ้นถ้ามี long-running idle-in-transaction connection ค้างอยู่ (vacuum ทำงานไม่ได้เต็มที่)

#### 4. สรุปแนวทางที่ถูกต้อง

แทนที่จะเพิ่ม `max_connections` ไปเรื่อยๆ แนวทางที่แนะนำคือ:

- ตั้ง `max_connections` ให้อยู่ในระดับที่ database server รับไหวจริง (มักจะอยู่ที่หลักร้อยเท่านั้น เช่น 100-300 ขึ้นกับ CPU/RAM)
- ใช้ **connection pooler** (PgBouncer หรือ Pgpool-II) เป็นตัวกลางที่รับ connection จำนวนมากจากฝั่ง client แต่ส่งต่อไปยัง PostgreSQL ด้วยจำนวน connection ที่จำกัดและ reuse ได้

สำหรับฐานข้อมูล `ecommerce_db` ของเรา แทนที่จะตั้ง `max_connections = 2000` เพื่อรองรับ 8 web servers × 50 connections เราจะตั้ง `max_connections` ไว้ที่ระดับที่เหมาะสม (เช่น 200) แล้วให้ PgBouncer เป็นผู้รับ connection จำนวนมากจากฝั่งแอปแทน

```ini
# postgresql.conf (ฝั่ง PostgreSQL server)
max_connections = 200          # เผื่อไว้สำหรับ pooler connections + superuser + replication
superuser_reserved_connections = 5
```

---

## Step 653: PgBouncer คืออะไร

### ภาพรวม

**PgBouncer** คือ lightweight connection pooler สำหรับ PostgreSQL เขียนด้วยภาษา C ออกแบบมาให้เบาที่สุดเท่าที่จะทำได้ (single-threaded event loop คล้าย nginx) จุดประสงค์หลักคือทำหน้าที่เป็น **ตัวกลาง (proxy)** ระหว่าง client กับ PostgreSQL server:

```
Application ──> PgBouncer (listen port 6432) ──> PostgreSQL (port 5432)
```

PgBouncer จะ maintain "pool" ของ connection จริงไปยัง PostgreSQL ไว้จำนวนหนึ่ง แล้วให้ client connection จำนวนมากมา "แชร์" ใช้ connection เหล่านั้น (ตาม pool mode ที่เลือก จะกล่าวถึงใน Step 654)

### จุดเด่นของ PgBouncer

- **เบามาก**: ใช้ RAM ต่อ connection เพียง ~2KB (เทียบกับ PostgreSQL backend process ที่ใช้หลาย MB)
- **รองรับ connection จำนวนมาก**: รองรับได้หลักหมื่น client connections พร้อมกันด้วย resource ที่น้อย
- **ติดตั้งง่าย** ไฟล์ config เดียว (`pgbouncer.ini`) และไฟล์ authentication เดียว (`userlist.txt`)
- **TLS support**: รองรับการเข้ารหัสทั้งฝั่ง client-to-pgbouncer และ pgbouncer-to-server
- **Admin console**: สามารถ connect เข้าไปที่ virtual database ชื่อ `pgbouncer` เพื่อรัน SQL-like command ดู stats ได้แบบ real-time

### การติดตั้งเบื้องต้น

#### บน Ubuntu/Debian

```bash
sudo apt-get update
sudo apt-get install -y pgbouncer

# ตรวจสอบเวอร์ชัน
pgbouncer --version
# pgbouncer version 1.22.1
```

#### บน RHEL/Rocky Linux/AlmaLinux

```bash
sudo dnf install -y pgbouncer
```

#### จาก source (สำหรับ control เวอร์ชันแบบละเอียด)

```bash
wget https://www.pgbouncer.org/downloads/files/1.22.1/pgbouncer-1.22.1.tar.gz
tar xzf pgbouncer-1.22.1.tar.gz
cd pgbouncer-1.22.1
./configure --prefix=/usr/local/pgbouncer
make
sudo make install
```

#### โครงสร้างไฟล์หลังติดตั้ง (Ubuntu/Debian ตัวอย่าง)

```
/etc/pgbouncer/pgbouncer.ini     # ไฟล์ config หลัก
/etc/pgbouncer/userlist.txt      # รายชื่อ user + password hash
/var/log/postgresql/pgbouncer.log
/var/run/postgresql/pgbouncer.pid
```

### ทดสอบการเชื่อมต่อเบื้องต้น

เมื่อตั้งค่าเสร็จแล้ว (รายละเอียดใน Step 655-656) เราจะเชื่อมต่อผ่าน PgBouncer แทนที่จะต่อ PostgreSQL โดยตรง โดยเปลี่ยนแค่ **พอร์ต** ในฝั่งแอปพลิเคชัน:

```bash
# เดิม: เชื่อมต่อ PostgreSQL โดยตรง (พอร์ต 5432)
psql -h db-primary.internal -p 5432 -U app_user -d ecommerce_db

# ใหม่: เชื่อมต่อผ่าน PgBouncer (พอร์ต 6432)
psql -h pgbouncer.internal -p 6432 -U app_user -d ecommerce_db
```

จากมุมมองของแอปพลิเคชัน แทบไม่มีความแตกต่างในการเขียนโค้ด — แค่เปลี่ยน connection string ชี้ไปที่ PgBouncer แทน โดยส่วนใหญ่แค่เปลี่ยน host/port ใน environment variable

---

## Step 654: Pool Mode ของ PgBouncer

นี่คือแนวคิดที่**สำคัญที่สุด**ของ PgBouncer — pool mode กำหนดว่า "connection จริง" ไปยัง PostgreSQL จะถูก**คืนกลับเข้า pool เมื่อไหร่** เพื่อให้ client connection อื่นเอาไปใช้ต่อได้ PgBouncer มี 3 โหมด กำหนดผ่านพารามิเตอร์ `pool_mode`

### 1. Session pooling (`pool_mode = session`)

```
Client connect ──> ได้ server connection มา 1 ตัว ──> ใช้ตลอดอายุของ session
                                                         (จนกว่า client จะ disconnect)
```

- Server connection จะถูก**จอง**ไว้ให้ client ตัวนั้นตลอดเวลาที่ client ยัง connect อยู่ (ไม่ว่าจะรัน query หรือ idle)
- คืนกลับ pool ก็ต่อเมื่อ client ตัดการเชื่อมต่อ (`disconnect`)
- **ปลอดภัยที่สุด** — รองรับฟีเจอร์ของ PostgreSQL ได้ครบทุกอย่าง เพราะ session behavior เหมือนต่อตรงทุกประการ
- แต่ **ประหยัด connection น้อยที่สุด** เพราะแทบไม่ต่างจากไม่มี pooling เลยในแง่จำนวน backend process ที่ต้องใช้ (ได้ประโยชน์แค่เรื่องลด overhead ตอนสร้าง/authentication connection ใหม่)

ใช้เหมาะกับ: แอปพลิเคชันที่เปิด connection แบบ long-lived อยู่แล้ว (เช่น batch job, admin tool) หรือระบบที่ใช้ session-level feature เยอะ

### 2. Transaction pooling (`pool_mode = transaction`)

```
Client connect ──> ได้ server connection มา ──> ใช้จนจบ 1 transaction ──> คืนกลับ pool ทันที
                                                                          (client connection ยังอยู่ แต่รอ server ใหม่ในรอบถัดไป)
```

- Server connection จะถูกจองให้เฉพาะช่วง**หนึ่ง transaction** เท่านั้น (ตั้งแต่ `BEGIN` จนถึง `COMMIT`/`ROLLBACK` หรือถ้าไม่มี explicit transaction ก็คือ 1 statement)
- เมื่อ transaction จบ server connection จะถูกคืนกลับ pool ทันที แม้ client connection จะยังไม่ disconnect
- **นี่คือโหมดที่ได้รับความนิยมมากที่สุด** เพราะให้ประสิทธิภาพการ reuse connection สูงสุดโดยที่ยังใช้งาน transaction ได้ตามปกติ
- **ข้อจำกัดสำคัญ**: ฟีเจอร์ที่ผูกกับ session (ไม่ใช่ transaction) จะ**ใช้ไม่ได้** หรือทำงานไม่ถูกต้อง เพราะ query แต่ละครั้งอาจไปตกลงที่ server connection คนละตัวกัน:

| ฟีเจอร์ | ใช้งานได้ใน transaction mode หรือไม่ |
|---|---|
| `SET` session-level variable (เช่น `SET search_path`) | ❌ ไม่ได้ — จะหายไปเมื่อ transaction จบ (ต้องใช้ `SET LOCAL` แทน) |
| `PREPARE` statement (server-side prepared statement) | ❌ ไม่ปลอดภัย — statement อาจไม่มีอยู่ใน server connection ตัวถัดไป (ต้องเปิด `max_prepared_statements` หรือใช้ client-side prepare) |
| `LISTEN`/`NOTIFY` | ❌ ไม่ทำงาน — ต้องมี session ที่คงอยู่ต่อเนื่อง |
| `WITH HOLD` cursor | ❌ ไม่ได้ |
| Advisory lock (`pg_advisory_lock`) | ⚠️ อันตราย — lock อาจค้างอยู่ที่ server connection ที่ client ไม่ได้ถืออยู่แล้ว |
| Temporary table | ⚠️ มีปัญหา — temp table ผูกกับ session ไม่ใช่ transaction (ยกเว้นสร้างและใช้ในทรานแซคชันเดียวแล้ว drop) |
| `SET LOCAL` | ✅ ได้ — เพราะจำกัดขอบเขตแค่ใน transaction เดียวกัน |
| Autocommit query แบบ single statement | ✅ ได้ตามปกติ |

ใช้เหมาะกับ: เว็บแอปพลิเคชันทั่วไปที่แต่ละ request ทำงานสั้นๆ แบบ transaction เดี่ยว เช่น API ของ `ecommerce_db` ที่รับ order, query product, update cart — **นี่คือโหมดที่แนะนำสำหรับ web/API tier ของระบบอีคอมเมิร์ซ**

### 3. Statement pooling (`pool_mode = statement`)

```
Client connect ──> ได้ server connection มา ──> ใช้แค่ 1 statement ──> คืนกลับ pool ทันที
```

- จองแค่ช่วง**หนึ่ง SQL statement**เท่านั้น
- Aggressive ที่สุดในการ reuse connection
- **ข้อจำกัดหนักที่สุด**: **ไม่รองรับ multi-statement transaction เลย** (ถ้า client พยายามส่ง `BEGIN` จะ error) เหมาะกับ workload ที่เป็น autocommit query ล้วนๆ เท่านั้น
- แทบไม่ค่อยถูกใช้ในทางปฏิบัติ ยกเว้นกรณีพิเศษเช่น PgBouncer เป็น front-end สำหรับระบบที่ทำ read-only query แบบสั้นๆ จำนวนมาก และไม่มี transaction logic เลย

### ตารางเปรียบเทียบสรุป

| คุณสมบัติ | session | transaction | statement |
|---|---|---|---|
| Connection reuse efficiency | ต่ำ | สูง | สูงสุด |
| รองรับ multi-statement transaction | ✅ | ✅ | ❌ |
| รองรับ session variable (`SET`) | ✅ | ❌ (ต้องใช้ `SET LOCAL`) | ❌ |
| รองรับ `PREPARE` แบบ server-side | ✅ | ⚠️ ต้องระวัง | ❌ |
| รองรับ `LISTEN/NOTIFY` | ✅ | ❌ | ❌ |
| รองรับ advisory lock ข้าม statement | ✅ | ⚠️ อันตราย | ❌ |
| เหมาะกับ | batch job, admin tool | web/API tier ทั่วไป | read-only stateless query |

### การตั้งค่าใน pgbouncer.ini

```ini
[databases]
ecommerce_db = host=db-primary.internal port=5432 dbname=ecommerce_db pool_mode=transaction

[pgbouncer]
;; ตั้ง default pool mode สำหรับทุก database ที่ไม่ได้ระบุ pool_mode เจาะจง
pool_mode = transaction
```

สามารถกำหนด `pool_mode` แยกเป็นราย database ได้ใน section `[databases]` เช่น กรณีของเราอาจตั้งให้ `ecommerce_db` (ใช้จาก web/API) เป็น `transaction` แต่ database สำหรับ admin/reporting เป็น `session`:

```ini
[databases]
ecommerce_db          = host=db-primary.internal dbname=ecommerce_db pool_mode=transaction
ecommerce_db_reports   = host=db-replica1.internal dbname=ecommerce_db pool_mode=session
```

---

## Step 655: การตั้งค่า pgbouncer.ini

ไฟล์ `pgbouncer.ini` คือหัวใจของการตั้งค่า PgBouncer แบ่งเป็นหลาย section โดยหลักๆ คือ `[databases]`, `[users]` (ไม่บังคับ), และ `[pgbouncer]`

### ตัวอย่างไฟล์ pgbouncer.ini แบบเต็มสำหรับระบบอีคอมเมิร์ซของเรา

```ini
;; /etc/pgbouncer/pgbouncer.ini
;; ================================================
;; PgBouncer configuration สำหรับ ecommerce_db
;; ================================================

[databases]
;; database หลักที่ web/API server ใช้งาน (traffic สูงสุด)
ecommerce_db = host=db-primary.internal port=5432 dbname=ecommerce_db pool_mode=transaction pool_size=25

;; database สำหรับ background worker (batch job, ต้องการ session feature บางส่วน)
ecommerce_worker = host=db-primary.internal port=5432 dbname=ecommerce_db pool_mode=session pool_size=15

;; ชี้ไปที่ read replica สำหรับ reporting/BI (read-only)
ecommerce_reports = host=db-replica1.internal port=5432 dbname=ecommerce_db pool_mode=transaction pool_size=10

;; wildcard: database อื่นที่ไม่ได้ระบุไว้ชัดเจน ใช้ค่า default
* = host=db-primary.internal port=5432

[pgbouncer]
;; -------------------- Network --------------------
listen_addr = 0.0.0.0
listen_port = 6432
unix_socket_dir = /var/run/postgresql

;; -------------------- Authentication --------------------
auth_type = scram-sha-256
auth_file = /etc/pgbouncer/userlist.txt
;; ต้องมี user ที่ query pg_shadow ได้ ถ้าใช้ auth_query แทน userlist.txt
;; auth_query = SELECT usename, passwd FROM pg_shadow WHERE usename=$1

;; -------------------- Admin --------------------
admin_users = pgbouncer_admin
stats_users = pgbouncer_stats, monitoring_user

;; -------------------- Pool sizing --------------------
;; ค่า default หากไม่ระบุ pool_size ต่อ database
default_pool_size = 20
;; จำนวน connection สำรองที่เผื่อไว้ตอน pool เต็ม (burst capacity)
reserve_pool_size = 5
reserve_pool_timeout = 3

;; จำนวน client connection สูงสุดที่ PgBouncer รับได้ (รวมทุก database)
max_client_conn = 2000

;; จำนวน server connection สูงสุดต่อ database (นับรวมทุก pool ของ database เดียวกัน)
max_db_connections = 50
;; จำนวน server connection สูงสุดต่อ user (กันไม่ให้ user เดียวยึด connection ทั้งหมด)
max_user_connections = 40

;; -------------------- Pool mode default --------------------
pool_mode = transaction

;; -------------------- Timeouts --------------------
server_idle_timeout = 600        ;; ปิด server connection ที่ idle เกิน 10 นาที
server_lifetime = 3600           ;; รีไซเคิล server connection ทุก 1 ชั่วโมง (กัน memory leak สะสม)
server_connect_timeout = 15
query_timeout = 0                ;; 0 = ไม่จำกัด (ควรตั้งที่ฝั่งแอปแทนถ้าจำเป็น)
query_wait_timeout = 120         ;; ถ้า client รอ server connection ว่างเกิน 120s ให้ error
client_idle_timeout = 0          ;; 0 = ไม่ตัด idle client (ระวังถ้าแอปเปิด conn ค้าง)
idle_transaction_timeout = 60    ;; ตัด client ที่ idle-in-transaction เกิน 60s (ป้องกัน lock ค้าง)

;; -------------------- Logging --------------------
logfile = /var/log/postgresql/pgbouncer.log
pidfile = /var/run/postgresql/pgbouncer.pid
log_connections = 1
log_disconnections = 1
log_pooler_errors = 1
log_stats = 1
stats_period = 60

;; -------------------- TLS (แนะนำสำหรับ production) --------------------
client_tls_sslmode = prefer
client_tls_cert_file = /etc/pgbouncer/server.crt
client_tls_key_file = /etc/pgbouncer/server.key
server_tls_sslmode = prefer

;; -------------------- Misc --------------------
ignore_startup_parameters = extra_float_digits
application_name_add_host = 1
```

### อธิบายพารามิเตอร์สำคัญ

#### `pool_size` / `default_pool_size`

จำนวน server connection สูงสุด**ต่อ 1 pool** (pool = การรวมกันของ database + user) ที่ PgBouncer จะเปิดไปยัง PostgreSQL จริง นี่คือค่าที่กำหนดว่า PostgreSQL จะเห็น connection กี่ตัวจริงๆ

ตัวอย่างการคำนวณสำหรับ `ecommerce_db`:

```
web/API servers 8 instances × client connection สูงสุด 50 ตัว/instance = 400 client connections
                                                    │
                                                    ▼
                                    PgBouncer: pool_size = 25
                                                    │
                                                    ▼
                              PostgreSQL เห็นแค่ ~25 backend process จริง
```

การตั้ง `pool_size` ที่เหมาะสมควรอ้างอิงจาก:

```
pool_size ที่แนะนำ ≈ (CPU core ของ PostgreSQL server × 2) ถึง (CPU core × 4)
```

เช่นถ้า PostgreSQL server มี 8 core → `pool_size` ประมาณ 16-32 ต่อ database ก็มักจะเพียงพอสำหรับ workload ทั่วไป (ต้อง benchmark จริงเพื่อ fine-tune)

#### `max_client_conn`

จำนวน **client connection** สูงสุดที่ PgBouncer เองรับได้ (ฝั่งที่คุยกับแอปพลิเคชัน) ค่านี้ควรตั้งสูงกว่า `pool_size` มาก เพราะจุดประสงค์คือให้ client เยอะๆ มาแชร์ pool connection จำนวนน้อยกว่า

```
max_client_conn = 2000   ;; รองรับ client ได้สูงสุด 2000 ราย
default_pool_size = 20   ;; แต่ใช้ server connection จริงแค่ 20 ต่อ pool
```

หากจำนวน client ที่ active พร้อมกันเกิน pool_size ที่มี PgBouncer จะให้ client เหล่านั้น**รอคิว** (queue) จนกว่าจะมี server connection ว่าง (ตาม `query_wait_timeout`)

#### `reserve_pool_size` และ `reserve_pool_timeout`

เผื่อ "connection สำรอง" ไว้สำหรับกรณี burst traffic ที่ pool หลักเต็มพอดี ถ้า client รอเกิน `reserve_pool_timeout` วินาที PgBouncer จะเปิด connection เพิ่มจาก reserve pool ชั่วคราว (มีประโยชน์มากช่วง flash sale ที่ traffic พุ่งกะทันหัน)

#### `server_lifetime` และ `server_idle_timeout`

- `server_lifetime`: อายุสูงสุดของ server connection ก่อนถูก recycle (แม้กำลังถูกใช้งานอยู่ก็จะถูกปิดหลัง transaction ปัจจุบันจบ) ช่วยป้องกันปัญหา memory bloat สะสมในระยะยาวจาก long-lived connection
- `server_idle_timeout`: ปิด server connection ที่ไม่ได้ใช้งานนานเกินกำหนด เพื่อคืนทรัพยากรกลับ

#### `idle_transaction_timeout`

สำคัญมากสำหรับป้องกันปัญหา "idle in transaction" ที่ค้าง lock ไว้นาน เช่น ถ้า worker ของเราเปิด transaction แล้วลืม commit (บั๊กในโค้ด) PgBouncer จะตัดการเชื่อมต่อนั้นทิ้งหลัง 60 วินาที ป้องกันไม่ให้ lock บนตาราง `inventory` ค้างจนกระทบ order อื่น

### ตรวจสอบและรีโหลด config

```bash
# ตรวจสอบ syntax ก่อน apply (ไม่มี built-in dry-run แต่ตรวจผ่าน pgbouncer -v)
pgbouncer -v /etc/pgbouncer/pgbouncer.ini

# start service
sudo systemctl start pgbouncer
sudo systemctl enable pgbouncer

# reload config โดยไม่ตัดการเชื่อมต่อที่มีอยู่ (RELOAD ผ่าน admin console)
psql -h 127.0.0.1 -p 6432 -U pgbouncer_admin pgbouncer -c "RELOAD"

# หรือผ่าน systemd
sudo systemctl reload pgbouncer
```

---

## Step 656: userlist.txt และ authentication ของ PgBouncer

### รูปแบบไฟล์ userlist.txt

PgBouncer ต้องการรายชื่อ user และ password hash ของตัวเองเพื่อ authenticate client (ไม่ได้ไปถาม PostgreSQL โดยตรงทุกครั้ง เว้นแต่ใช้ `auth_query`) รูปแบบไฟล์เป็น plain text แบบง่าย:

```
"username" "password_hash"
```

### สร้าง userlist.txt ด้วย SCRAM-SHA-256 (แนะนำสำหรับ PostgreSQL 10+)

วิธีที่ปลอดภัยที่สุดคือ**คัดลอก password hash ที่มีอยู่แล้วจาก PostgreSQL** (ที่เก็บใน `pg_authid`) แทนที่จะสร้าง plaintext password เอง เพราะ PostgreSQL เก็บ password แบบ hashed อยู่แล้ว:

```sql
-- รันบน PostgreSQL เพื่อดู password hash ของ user ที่ต้องการ
SELECT usename, passwd FROM pg_shadow WHERE usename IN ('app_user', 'worker_user', 'report_user');
```

ผลลัพธ์ตัวอย่าง (ค่า `passwd` จะเป็น SCRAM verifier รูปแบบ `SCRAM-SHA-256$...`):

```
   usename    |                              passwd
---------------+--------------------------------------------------------------------
 app_user      | SCRAM-SHA-256$4096:8Kx3z...==$hR9j...==:vN2q...==
 worker_user   | SCRAM-SHA-256$4096:Lm1p...==$xQ8t...==:kJ3w...==
 report_user   | SCRAM-SHA-256$4096:Zc7v...==$dF5m...==:tY9n...==
```

นำค่าเหล่านี้ไปใส่ใน `userlist.txt` โดยตรง (ไม่ต้องรู้ plaintext password เลย):

```
# /etc/pgbouncer/userlist.txt
"app_user"      "SCRAM-SHA-256$4096:8Kx3z...==$hR9j...==:vN2q...=="
"worker_user"   "SCRAM-SHA-256$4096:Lm1p...==$xQ8t...==:kJ3w...=="
"report_user"   "SCRAM-SHA-256$4096:Zc7v...==$dF5m...==:tY9n...=="
"pgbouncer_admin" "SCRAM-SHA-256$4096:Qw2e...==$Rt6y...==:Ui8o...=="
"pgbouncer_stats" "SCRAM-SHA-256$4096:As3d...==$Fg7h...==:Jk9l...=="
```

### สคริปต์ช่วยสร้าง userlist.txt อัตโนมัติ

```bash
#!/bin/bash
# generate_userlist.sh
# ดึง password hash จาก PostgreSQL มาสร้าง userlist.txt โดยอัตโนมัติ

PGHOST="db-primary.internal"
PGUSER="postgres"
OUTPUT="/etc/pgbouncer/userlist.txt"

psql -h "$PGHOST" -U "$PGUSER" -d ecommerce_db -t -A -F',' \
  -c "SELECT usename, passwd FROM pg_shadow WHERE usename IN ('app_user','worker_user','report_user','pgbouncer_admin','pgbouncer_stats')" \
  | awk -F',' '{ print "\""$1"\" \""$2"\"" }' > "$OUTPUT"

chmod 640 "$OUTPUT"
chown postgres:postgres "$OUTPUT"
echo "userlist.txt updated: $(wc -l < $OUTPUT) users"
```

### auth_type ที่รองรับ

| auth_type | คำอธิบาย |
|---|---|
| `scram-sha-256` | มาตรฐานที่แนะนำสำหรับ PostgreSQL 10+ ปลอดภัยที่สุด |
| `md5` | รองรับ backward compatibility กับระบบเก่า (ควร migrate ออกถ้าทำได้) |
| `trust` | ไม่ตรวจสอบ password เลย **ห้ามใช้ใน production** ใช้ได้แค่ทดสอบ local |
| `plain` | ส่ง password แบบ plaintext **ไม่ปลอดภัย** ไม่ควรใช้ |
| `cert` | ใช้ TLS client certificate แทน password |

### auth_query แบบ dynamic (ทางเลือกแทน userlist.txt แบบ static)

หากไม่อยากดูแล sync `userlist.txt` ด้วยมือทุกครั้งที่มี user ใหม่ สามารถใช้ `auth_query` ให้ PgBouncer ไป query PostgreSQL โดยตรงแบบ real-time แทน:

```ini
[pgbouncer]
auth_type = scram-sha-256
auth_query = SELECT usename, passwd FROM pg_shadow WHERE usename=$1
;; ต้องระบุ user ที่ใช้รัน auth_query ด้วย (ควรเป็น user สิทธิ์จำกัด อ่านได้แค่ pg_shadow)
auth_user = pgbouncer_auth
```

ต้องสร้าง function หรือ view ที่มีสิทธิ์เพียงพอสำหรับ user `pgbouncer_auth` ให้ query ได้ (บางกรณีต้องสร้าง `SECURITY DEFINER` function แทนการให้สิทธิ์ตรงกับ `pg_shadow`):

```sql
-- ตัวอย่าง: สร้าง function สำหรับ auth_query แบบปลอดภัยกว่า
CREATE OR REPLACE FUNCTION pgbouncer_auth_lookup(p_username text)
RETURNS TABLE(usename name, passwd text)
LANGUAGE sql SECURITY DEFINER
AS $$
    SELECT usename, passwd FROM pg_shadow WHERE usename = p_username;
$$;

REVOKE ALL ON FUNCTION pgbouncer_auth_lookup(text) FROM PUBLIC;
GRANT EXECUTE ON FUNCTION pgbouncer_auth_lookup(text) TO pgbouncer_auth;
```

```ini
auth_query = SELECT usename, passwd FROM pgbouncer_auth_lookup($1)
```

**ข้อดีของ auth_query**: ไม่ต้อง sync ไฟล์ทุกครั้งที่เพิ่ม/แก้ user, password เปลี่ยนแล้วมีผลทันที
**ข้อเสีย**: เพิ่ม query ไปยัง PostgreSQL ทุกครั้งที่มี client connection ใหม่ (แต่ PgBouncer มี cache ผลลัพธ์อยู่ระยะหนึ่งเพื่อลด overhead)

### เชื่อมโยง userlist.txt กับ [databases] section

เมื่อ client ส่ง username/password มาที่ PgBouncer, PgBouncer จะ:

1. ตรวจสอบ username นั้นมีใน `userlist.txt` หรือไม่ (หรือผ่าน `auth_query`)
2. ตรวจสอบ password hash ตรงกันหรือไม่ (ใช้ SCRAM challenge-response ไม่ส่ง plaintext ผ่าน network)
3. ถ้าผ่าน ค้นหา database ที่ระบุใน connection string เทียบกับ `[databases]` section
4. จับคู่กับ pool ที่มีอยู่ (หรือสร้างใหม่) ตาม database + username combination

```
Client: psql -h pgbouncer -p 6432 -U app_user ecommerce_db
              │
              ▼
   1. ตรวจสอบ "app_user" ใน userlist.txt ✓
   2. ตรวจสอบ password (SCRAM) ✓
   3. หา "ecommerce_db" ใน [databases] section ✓
   4. จับคู่กับ pool (ecommerce_db, app_user) → ใช้ server connection ที่มีอยู่ หรือเปิดใหม่ถ้า pool ยังไม่เต็ม
```

---

## Step 657: Pgpool-II คืออะไร

### ภาพรวม

**Pgpool-II** คือ middleware สำหรับ PostgreSQL เช่นกัน แต่มีขอบเขตความสามารถ**กว้างกว่า** PgBouncer มาก ไม่ได้ทำหน้าที่แค่ connection pooling แต่ยังมี:

1. **Connection pooling** — คล้าย PgBouncer แต่ implementation ต่างกัน
2. **Load balancing** — กระจาย read query ไปยัง replica หลายตัวโดยอัตโนมัติ (แยก read/write query ให้เอง)
3. **Query routing / Query caching** — วิเคราะห์ query ที่เข้ามาว่าเป็น read หรือ write แล้วส่งไปยัง node ที่เหมาะสม (in-memory query cache สำหรับ SELECT ที่ซ้ำ)
4. **Replication mode** — (แบบเก่า สำหรับ PostgreSQL ที่ไม่มี native streaming replication) Pgpool-II เองทำหน้าที่ replicate ข้อมูลไปยังหลาย server — ปัจจุบันไม่ค่อยใช้แล้วเพราะ PostgreSQL มี built-in streaming replication ที่ดีกว่า
5. **Automatic failover** — ตรวจจับเมื่อ primary node ล่ม แล้วสลับไปยัง standby โดยอัตโนมัติ พร้อม `watchdog` สำหรับทำ high-availability ของตัว Pgpool-II เอง (กันไม่ให้ Pgpool-II เป็น single point of failure)
6. **Parallel query** — (ฟีเจอร์ขั้นสูง) กระจาย query ไปประมวลผลบนหลาย node พร้อมกันแล้วรวมผลลัพธ์ (ใช้ยากและมีข้อจำกัดเยอะในทางปฏิบัติ)

### สถาปัตยกรรมของ Pgpool-II ในระบบอีคอมเมิร์ซ

```
                         ┌─────────────────────┐
   Web/API servers ────> │      Pgpool-II       │
   Reporting/BI    ────> │  (query routing +    │
                         │   load balancing +    │
                         │   connection pooling) │
                         └──────────┬────────────┘
                                    │
                    ┌───────────────┼───────────────┐
                    ▼               ▼               ▼
              PostgreSQL       PostgreSQL       PostgreSQL
              (Primary)        (Replica 1)      (Replica 2)
              รับ INSERT/       รับ SELECT       รับ SELECT
              UPDATE/DELETE     (load balance)   (load balance)
```

ตัวอย่างเช่น query `SELECT * FROM products WHERE category_id = 5` (read-only) Pgpool-II จะวิเคราะห์และส่งไปที่ replica ตัวใดตัวหนึ่งโดยอัตโนมัติ ในขณะที่ `INSERT INTO orders (...) VALUES (...)` จะถูกส่งไปที่ primary เท่านั้น — ทั้งหมดนี้**โปร่งใสต่อแอปพลิเคชัน** ไม่ต้องเขียนโค้ดแยก read/write connection เอง

### ตัวอย่างการตั้งค่าเบื้องต้นใน pgpool.conf

```ini
# /etc/pgpool-II/pgpool.conf (ตัดมาเฉพาะส่วนสำคัญ)

# -------------------- Listen --------------------
listen_addresses = '*'
port = 9999

# -------------------- Backend connection settings --------------------
backend_hostname0 = 'db-primary.internal'
backend_port0 = 5432
backend_weight0 = 0              # weight = 0 หมายถึงไม่ส่ง SELECT มาที่ node นี้ (กัน primary ไว้รับ write อย่างเดียว หากต้องการ)
backend_flag0 = 'ALWAYS_PRIMARY'
backend_application_name0 = 'primary'

backend_hostname1 = 'db-replica1.internal'
backend_port1 = 5432
backend_weight1 = 1
backend_flag1 = 'DISALLOW_TO_FAILOVER'
backend_application_name1 = 'replica1'

backend_hostname2 = 'db-replica2.internal'
backend_port2 = 5432
backend_weight2 = 1
backend_flag2 = 'DISALLOW_TO_FAILOVER'
backend_application_name2 = 'replica2'

# -------------------- Pooling --------------------
num_init_children = 32          # จำนวน pgpool child process (≈ จำนวน concurrent connection สูงสุด)
max_pool = 4                    # จำนวน connection cache ต่อ child process ต่อ database/user
child_life_time = 300
connection_life_time = 0

# -------------------- Load balancing --------------------
load_balance_mode = on
statement_level_load_balance = off   # false = ตัดสินใจ load balance ต่อ session, true = ต่อ statement

# -------------------- Streaming replication check --------------------
sr_check_period = 10
sr_check_user = 'repl_check_user'
sr_check_database = 'ecommerce_db'

# -------------------- Failover --------------------
failover_command = '/etc/pgpool-II/failover.sh %d %h %p %D %m %M %H %P %r %R'
```

### ความสามารถของ Pgpool-II ที่ PgBouncer ไม่มี

| ความสามารถ | PgBouncer | Pgpool-II |
|---|---|---|
| Connection pooling | ✅ | ✅ |
| Query routing (read/write split) | ❌ | ✅ |
| Load balancing ระหว่าง replica | ❌ | ✅ |
| Automatic failover | ❌ | ✅ (ต้อง config เพิ่ม) |
| Watchdog (HA ของตัว pooler เอง) | ❌ (ต้องพึ่ง HAProxy/Keepalived ภายนอก) | ✅ built-in |
| In-memory query cache | ❌ | ✅ |
| Parallel query | ❌ | ✅ (จำกัดการใช้งาน) |

---

## Step 658: เมื่อไหร่ควรใช้ PgBouncer เทียบกับ Pgpool-II

### หลักการเลือก

การเลือกระหว่าง PgBouncer กับ Pgpool-II ไม่ใช่แค่เรื่อง "ตัวไหนดีกว่า" แต่เป็นเรื่อง **trade-off ระหว่างความซับซ้อนกับความสามารถ (complexity vs. capability)**

#### เลือก PgBouncer เมื่อ:

- ต้องการแค่ **connection pooling อย่างเดียว** ไม่ต้องการ query routing หรือ load balancing (เพราะแอปพลิเคชันจัดการ read/write splitting เองอยู่แล้ว หรือระบบมี PostgreSQL เดียว ไม่มี replica)
- ต้องการ**ประสิทธิภาพสูงสุด**และ overhead ต่ำที่สุด — PgBouncer เบากว่ามาก ใช้ทรัพยากรน้อยกว่า Pgpool-II หลายเท่า
- ทีมต้องการ**ความเรียบง่ายในการดูแลรักษา** — config file เดียว ไม่ซับซ้อน debug ง่าย
- ระบบมีการทำ HA/failover อยู่แล้วผ่านเครื่องมืออื่น (เช่น Patroni + HAProxy, หรือ pgcat) และต้องการแค่ layer pooling เสริมเข้าไป

#### เลือก Pgpool-II เมื่อ:

- ต้องการ **read/write splitting อัตโนมัติ** โดยไม่อยากให้แอปพลิเคชันต้องรู้เรื่อง replica เลย (transparent load balancing)
- ต้องการ built-in failover mechanism โดยไม่อยากพึ่งเครื่องมือภายนอกเพิ่ม
- มี read replica หลายตัวและต้องการกระจายโหลด query แบบอัตโนมัติ ไม่อยากเขียน logic แยก connection string เอง
- ยอมรับ**ความซับซ้อนในการดูแลรักษาที่เพิ่มขึ้น** และ overhead ที่สูงกว่า เพื่อแลกกับฟีเจอร์ที่ครบกว่า

### แนวทางที่ใช้จริงในหลายองค์กร (hybrid approach)

ในทางปฏิบัติ หลายทีมเลือกใช้**ทั้งสองตัวร่วมกัน** โดยวาง PgBouncer ไว้เป็น pooling layer ที่ใกล้กับ PostgreSQL แต่ละ node (ทั้ง primary และ replica) และให้ Pgpool-II หรือ logic ฝั่งแอปพลิเคชันทำหน้าที่ routing:

```
                    ┌──────────────┐
  App servers ────> │  Pgpool-II   │  (routing: write → primary, read → replica)
                    └──────┬───────┘
                           │
          ┌────────────────┼────────────────┐
          ▼                ▼                ▼
   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
   │  PgBouncer   │  │  PgBouncer   │  │  PgBouncer   │
   └──────┬───────┘  └──────┬───────┘  └──────┬───────┘
          ▼                 ▼                 ▼
      Primary            Replica1          Replica2
```

หรืออีกแนวทางที่**ง่ายกว่าและเป็นที่นิยมมากขึ้นเรื่อยๆ ในระบบสมัยใหม่** คือใช้แค่ **PgBouncer** ร่วมกับการทำ **read/write splitting ที่ฝั่งแอปพลิเคชันเอง** (เช่น ใช้ library ที่รองรับ multi-host connection string อย่าง libpq `target_session_attrs`, หรือ ORM-level routing) แล้วใช้เครื่องมือ HA แยกต่างหาก เช่น **Patroni** (สำหรับจัดการ failover ของ PostgreSQL cluster) ร่วมกับ **HAProxy** (สำหรับ routing แบบ TCP-level ตาม health check)

### ตารางสรุปการตัดสินใจ

| สถานการณ์ | คำแนะนำ |
|---|---|
| PostgreSQL เดี่ยว ไม่มี replica ต้องการแค่ลด connection overhead | PgBouncer เท่านั้น |
| มี replica แต่ต้องการควบคุม routing เอง (การเลือก replica ขึ้นกับ business logic) | PgBouncer ที่แต่ละ node + routing ที่แอป |
| มี replica และต้องการ automatic transparent load balancing | Pgpool-II (หรือ Pgpool-II + PgBouncer แบบ hybrid) |
| ต้องการ throughput สูงสุด latency ต่ำสุด และทีมมีความเชี่ยวชาญเรื่อง PostgreSQL HA อยู่แล้ว | PgBouncer + Patroni + HAProxy |
| ทีมเล็ก ต้องการ all-in-one solution ที่จัดการหลายอย่างในเครื่องมือเดียว | Pgpool-II |

สำหรับระบบอีคอมเมิร์ซของเราที่มี traffic สูงและมี read replica 2 เครื่อง แนวทางที่เราจะแนะนำใน Step 660 คือ**เริ่มจาก PgBouncer เป็น pooling layer หลัก** ที่ primary (เพราะ write traffic คือคอขวดที่สำคัญที่สุดของอีคอมเมิร์ซ เช่นตอน checkout, update inventory) และประเมิน Pgpool-II หรือ application-level routing สำหรับ read replica ในภายหลังตามความจำเป็น

---

## Step 659: Monitoring connection pool

PgBouncer มี **admin console** ที่เข้าถึงได้ผ่าน `psql` ปกติ โดย connect ไปยัง virtual database ชื่อ `pgbouncer` (ไม่ใช่ database จริง) ด้วย user ที่อยู่ใน `admin_users` หรือ `stats_users`

### เชื่อมต่อ admin console

```bash
psql -h 127.0.0.1 -p 6432 -U pgbouncer_admin pgbouncer
```

### SHOW POOLS — ดูสถานะของแต่ละ pool

```sql
pgbouncer=# SHOW POOLS;
```

```
     database      |    user     | cl_active | cl_waiting | cl_active_cancel_req | cl_waiting_cancel_req | sv_active | sv_idle | sv_used | sv_tested | sv_login | maxwait | maxwait_us | pool_mode
--------------------+-------------+-----------+------------+-----------------------+------------------------+-----------+---------+---------+-----------+----------+---------+------------+-------------
 ecommerce_db       | app_user    |       142 |          8 |                     0 |                      0 |        22 |       3 |       0 |         0 |        0 |       0 |        842 | transaction
 ecommerce_worker   | worker_user |        12 |          0 |                     0 |                      0 |         9 |       6 |       0 |         0 |        0 |       0 |          0 | session
 ecommerce_reports  | report_user |         5 |          0 |                     0 |                      0 |         3 |       7 |       0 |         0 |        0 |       0 |          0 | transaction
 pgbouncer          | pgbouncer_admin |    1 |          0 |                     0 |                      0 |         0 |       0 |       0 |         0 |        0 |       0 |          0 | statement
```

คอลัมน์สำคัญที่ต้องจับตา:

| คอลัมน์ | ความหมาย | สิ่งที่ต้องระวัง |
|---|---|---|
| `cl_active` | จำนวน client connection ที่กำลังถูกจับคู่กับ server connection อยู่ | ปกติ |
| `cl_waiting` | จำนวน client connection ที่**รอคิว**อยู่เพราะ pool เต็ม | ถ้าค่านี้สูงต่อเนื่อง → pool_size เล็กเกินไป ต้องปรับเพิ่ม หรือ query ทำงานช้าเกินไปจนไม่คืน connection ทันเวลา |
| `sv_active` | จำนวน server connection ที่กำลังทำงานจริง | ปกติ |
| `sv_idle` | จำนวน server connection ที่ว่างอยู่ในpool พร้อมใช้ | ถ้าน้อยมากตลอดเวลาขณะที่ cl_waiting สูง แปลว่า pool ตึงเกินไป |
| `maxwait` / `maxwait_us` | เวลารอนานที่สุดของ client ที่กำลังรอคิว (วินาที/ไมโครวินาที) | ถ้าค่าสูง → latency ที่ผู้ใช้ปลายทางรู้สึกได้จริง ต้องรีบแก้ |

**ตัวอย่างการวิเคราะห์**: จากผลลัพธ์ข้างบน `ecommerce_db` มี `cl_waiting = 8` และ `maxwait_us = 842` (0.842 ms) — ยังถือว่าอยู่ในเกณฑ์ที่รับได้ แต่ถ้าเห็น `cl_waiting` เพิ่มขึ้นเรื่อยๆ ระหว่างช่วง flash sale ควรพิจารณาเพิ่ม `pool_size` หรือตรวจสอบว่ามี query ที่ทำงานนานผิดปกติ (holding transaction) หรือไม่

### SHOW STATS — ดูสถิติการใช้งานสะสม

```sql
pgbouncer=# SHOW STATS;
```

```
     database      | total_xact_count | total_query_count | total_received | total_sent | total_xact_time | total_query_time | total_wait_time | avg_xact_count | avg_query_count | avg_xact_time | avg_query_time | avg_wait_time
--------------------+-------------------+---------------------+------------------+--------------+--------------------+---------------------+--------------------+------------------+--------------------+------------------+-------------------+-----------------
 ecommerce_db       |          8342190  |           15203841  |       4213556123 |   9812345671 |          52341203  |            31204551 |             820341 |             1842 |               3521 |             1204 |               612 |              18
 ecommerce_worker   |           124532  |             341203  |         84123551 |    192341203 |            8123401 |             4203112 |               1203 |               45  |               102  |             2103 |              940 |               3
```

คอลัมน์สำคัญ:

- `avg_xact_time` / `avg_query_time`: เวลาเฉลี่ยของ transaction/query (microsecond) — ใช้ตรวจจับว่า query เริ่มช้าลงเมื่อเทียบกับ baseline หรือไม่
- `avg_wait_time`: เวลาเฉลี่ยที่ client ต้องรอก่อนได้ server connection — ตัวชี้วัดสำคัญของ **pool saturation** ถ้าค่านี้เพิ่มขึ้นต่อเนื่อง แปลว่า pool เริ่มไม่พอ
- `total_xact_count` / `total_query_count`: จำนวน transaction/query สะสมทั้งหมด — ใช้คำนวณ throughput (TPS/QPS) เมื่อเทียบกับช่วงเวลา

### คำสั่ง admin console อื่นๆ ที่มีประโยชน์

```sql
-- ดูรายละเอียด client connection แต่ละตัว
SHOW CLIENTS;

-- ดูรายละเอียด server connection แต่ละตัว (เชื่อมไปยัง PostgreSQL จริง)
SHOW SERVERS;

-- ดู database ที่ config ไว้ทั้งหมด
SHOW DATABASES;

-- ดู config parameter ปัจจุบันทั้งหมด
SHOW CONFIG;

-- ดู version
SHOW VERSION;

-- ดู memory usage โดยประมาณของ PgBouncer เอง
SHOW MEM;

-- Reload config โดยไม่ตัด connection ที่มีอยู่
RELOAD;

-- Pause pool ชั่วคราว (รอ transaction ปัจจุบันจบแล้วหยุดรับ query ใหม่ ใช้ตอน maintenance)
PAUSE ecommerce_db;

-- Resume หลัง pause
RESUME ecommerce_db;

-- ปิด server connection ทั้งหมดใน pool แบบนุ่มนวล (รอ transaction จบก่อน)
RECONNECT ecommerce_db;

-- Kill client connection ที่ค้างทันที (ใช้ระมัดระวัง)
KILL ecommerce_db;
```

### ตัวอย่าง SHOW CLIENTS

```sql
pgbouncer=# SHOW CLIENTS;
```

```
 type |   user    |     database      |    state    |   addr        | port  | local_addr | local_port |       connect_time       |       request_time       | wait  | wait_us | close_needed |  ptr   |  link  | remote_pid | tls
------+-----------+--------------------+--------------+----------------+-------+-------------+-------------+-----------------------------+-----------------------------+-------+---------+---------------+--------+--------+------------+-----
 C    | app_user  | ecommerce_db       | active       | 10.0.1.42      | 51234 | 10.0.1.10   |        6432 | 2026-09-25 08:12:03.120441  | 2026-09-25 08:14:22.881203  |     0 |       0 |             0 | 0x5621 | 0x5f3a |          0 |
 C    | app_user  | ecommerce_db       | waiting      | 10.0.1.43      | 51890 | 10.0.1.10   |        6432 | 2026-09-25 08:14:20.001102  | 2026-09-25 08:14:22.881301  |     0 |    2801 |             0 | 0x5722 |        |          0 |
```

การเห็น `state = waiting` จำนวนมากพร้อม `wait_us` สูง เป็นสัญญาณเตือนว่า pool ตึงและควรพิจารณาปรับ `pool_size` หรือหาสาเหตุ query ที่ทำงานนานผิดปกติ

### แนวทาง monitoring ในระยะยาว

สำหรับ production ควรดึงค่าจาก `SHOW POOLS`/`SHOW STATS` เข้า monitoring stack อย่างสม่ำเสมอ (เช่นทุก 15-30 วินาที) แทนการรันคำสั่งด้วยมือ ตัวอย่างเครื่องมือที่นิยมใช้ร่วมกับ PgBouncer:

- **pgbouncer_exporter** (Prometheus exporter) — ดึงค่าจาก admin console แปลงเป็น metrics แล้วให้ Prometheus scrape ไปแสดงบน Grafana dashboard
- Alert rule ตัวอย่างที่ควรตั้งไว้:
  - `cl_waiting > 0` ต่อเนื่องเกิน 1 นาที → แจ้งเตือนทีม
  - `avg_wait_time` เพิ่มขึ้นเกิน threshold ที่กำหนด (เช่น > 50ms) → สัญญาณของ pool saturation
  - `sv_idle = 0` ต่อเนื่อง → pool ใช้งานเต็มตลอดเวลา ควรพิจารณาเพิ่ม pool_size หรือ scale PostgreSQL

```yaml
# ตัวอย่าง docker-compose สำหรับ pgbouncer_exporter (แนวคิดคร่าวๆ)
services:
  pgbouncer_exporter:
    image: prometheuscommunity/pgbouncer-exporter
    environment:
      PGBOUNCER_EXPORTER_CONNECTION_STRING: "postgres://pgbouncer_stats:password@pgbouncer:6432/pgbouncer?sslmode=disable"
    ports:
      - "9127:9127"
```

---

## สรุปท้ายบท

### ตารางเปรียบเทียบ PgBouncer vs Pgpool-II

| หัวข้อ | PgBouncer | Pgpool-II |
|---|---|---|
| **จุดประสงค์หลัก** | Connection pooling อย่างเดียว | Connection pooling + query routing + HA |
| **ภาษาที่พัฒนา** | C (event loop เดี่ยว คล้าย nginx) | C |
| **ความเบา (footprint)** | เบามาก (~2KB ต่อ client connection) | หนักกว่า (แต่ละ child process ใช้ resource มากกว่า) |
| **Pool mode** | session / transaction / statement | connection pooling แบบพื้นฐาน (ไม่มีแนวคิด pool mode ละเอียดเท่า) |
| **Load balancing ระหว่าง replica** | ไม่มี (ต้องทำเองที่แอป หรือใช้เครื่องมืออื่นร่วม) | มีในตัว (transparent read/write splitting) |
| **Automatic failover** | ไม่มี (ต้องพึ่ง Patroni/HAProxy/Keepalived) | มีในตัว พร้อม watchdog สำหรับ HA ของตัวเอง |
| **Query caching** | ไม่มี | มี (in-memory query cache) |
| **ความซับซ้อนในการตั้งค่า** | ต่ำ (config file เดียว ง่ายต่อการ debug) | สูงกว่า (ต้องตั้งค่า backend node, watchdog, failover script) |
| **Overhead ต่อ query** | ต่ำมาก | สูงกว่า (เพราะต้อง parse query เพื่อ routing decision) |
| **Admin/monitoring interface** | SQL-like console (`SHOW POOLS`, `SHOW STATS` ฯลฯ) | `pcp` command-line tools + SQL-like console |
| **เหมาะกับ** | ระบบที่ต้องการแค่ลด connection overhead ประสิทธิภาพสูงสุด | ระบบที่มี read replica หลายตัวและต้องการ transparent routing/HA ในเครื่องมือเดียว |
| **การใช้งานร่วมกัน** | ใช้เป็น pooling layer หน้า PostgreSQL แต่ละ node ได้ | ใช้เป็น front-end layer ที่ route ไปยัง PgBouncer อีกที (hybrid) |

### สิ่งที่ต้องจำ

1. Connection pooling ไม่ใช่ทางเลือก แต่เป็น**สิ่งจำเป็น**สำหรับระบบที่มี traffic สูงอย่างอีคอมเมิร์ซ เพราะ PostgreSQL ใช้สถาปัตยกรรม process-per-connection ที่มีต้นทุนสูงถ้าเปิด/ปิด connection บ่อย
2. การเพิ่ม `max_connections` ไม่ใช่คำตอบระยะยาว เพราะมีข้อจำกัดด้าน memory, CPU context switching, และ lock contention
3. PgBouncer เหมาะกับกรณีที่ต้องการ pooling แบบเบาและเรียบง่าย ส่วน `pool_mode = transaction` คือค่าที่เหมาะสมที่สุดสำหรับ web/API tier ส่วนใหญ่ — แต่ต้องเข้าใจข้อจำกัดเรื่อง session-level feature ที่ใช้ไม่ได้
4. Pgpool-II เหมาะกับกรณีที่ต้องการ query routing/load balancing/failover ในเครื่องมือเดียว แลกกับความซับซ้อนที่เพิ่มขึ้น
5. Monitoring ผ่าน `SHOW POOLS` และ `SHOW STATS` เป็นสิ่งที่ต้องทำอย่างสม่ำเสมอ โดยเฉพาะการจับตา `cl_waiting` และ `avg_wait_time` เพื่อตรวจจับสัญญาณของ pool saturation ก่อนที่จะกระทบผู้ใช้จริง

ในบทถัดไป (**Part 067**) เราจะพูดถึง **Load Balancing** ในระดับที่กว้างขึ้น ทั้งการกระจายโหลดระหว่าง read replica, HAProxy, และการออกแบบ high-availability topology แบบเต็มรูปแบบสำหรับระบบอีคอมเมิร์ซของเรา

**บทถัดไป**: [Part 067 — Load Balancing](./part-067-load-balancing.md)

---

## แบบฝึกหัด

<details>
<summary><b>แบบฝึกหัดที่ 1</b>: อธิบายว่าทำไมการเพิ่ม max_connections ใน PostgreSQL จาก 100 เป็น 2000 โดยตรง ไม่ใช่วิธีแก้ปัญหา connection ไม่พอที่ดีเสมอไป (ระบุอย่างน้อย 3 เหตุผล)</summary>

**เฉลย**:

1. **Memory overhead**: แต่ละ connection ใช้ memory อย่างน้อย 5-10 MB (stack, catalog cache, local buffers) ยังไม่รวม `work_mem` ที่แต่ละ query อาจใช้หลายก้อนพร้อมกัน ถ้าทุก connection ถูกใช้งานจริงพร้อมกัน อาจต้องการ RAM หลาย GB ถึงหลักสิบ GB เฉพาะสำหรับ overhead นี้
2. **CPU context switching**: เมื่อจำนวน active connection (ที่กำลังรัน query จริง) เกินจำนวน CPU core มากๆ throughput โดยรวมจะเริ่มลดลงแทนที่จะเพิ่มขึ้น เพราะ OS ต้องสลับ context ระหว่าง process บ่อยเกินไป
3. **Lock contention**: ยิ่งมี active transaction พร้อมกันมาก ยิ่งมีโอกาสเกิด lock contention บน row ที่ถูกแก้ไขบ่อย และเพิ่ม overhead ในการ maintain snapshot (MVCC) ของระบบ
4. ทางออกที่ถูกต้องคือใช้ connection pooler (PgBouncer/Pgpool-II) เพื่อจำกัดจำนวน physical connection ที่ PostgreSQL ต้องรับจริง ในขณะที่ยังรองรับ client จำนวนมากได้

</details>

<details>
<summary><b>แบบฝึกหัดที่ 2</b>: ตาราง `orders` ในระบบอีคอมเมิร์ซของเราถูก query ผ่าน PgBouncer ที่ตั้ง `pool_mode = transaction` แอปพลิเคชันพยายามใช้โค้ดแบบนี้:

```sql
SET search_path TO ecommerce_schema;
SELECT * FROM orders WHERE customer_id = 123;
```

จะเกิดอะไรขึ้น และควรแก้ไขอย่างไร?</summary>

**เฉลย**:

ใน `pool_mode = transaction` คำสั่ง `SET` (session-level) จะมีผลแค่ในช่วงที่ server connection นั้นถูกจองให้ client อยู่ (คือช่วง 1 transaction เท่านั้น) เมื่อ transaction นั้นจบ (เช่นถ้าไม่มี `BEGIN`/`COMMIT` ครอบ แต่ละ statement ถือเป็น transaction แยกกันในโหมด autocommit) server connection จะถูกคืนกลับ pool และในการ query ครั้งถัดไป (`SELECT * FROM orders...`) อาจได้ server connection คนละตัวที่ไม่มีการตั้งค่า `search_path` ไว้ ทำให้ query หา table ไม่เจอ หรือ query ผิด schema

วิธีแก้ไข:
- ใช้ `SET LOCAL` แทน `SET` ภายใน transaction เดียวกัน แล้วครอบทุกอย่างด้วย `BEGIN...COMMIT`:
```sql
BEGIN;
SET LOCAL search_path TO ecommerce_schema;
SELECT * FROM orders WHERE customer_id = 123;
COMMIT;
```
- หรือกำหนด `search_path` ที่ระดับ role/database แทนการ SET ทุกครั้ง (เช่น `ALTER ROLE app_user SET search_path TO ecommerce_schema`)
- หรือถ้าจำเป็นต้องใช้ session-level SET จริงๆ ให้เปลี่ยนไปใช้ `pool_mode = session` สำหรับ use case นั้นโดยเฉพาะ

</details>

<details>
<summary><b>แบบฝึกหัดที่ 3</b>: กำหนดค่า pgbouncer.ini section [databases] สำหรับสถานการณ์นี้: ต้องการให้ web application เชื่อมต่อ database ชื่อ `ecommerce_db` ผ่าน PgBouncer โดยข้างหลังจริงชี้ไปที่ PostgreSQL server ที่ `10.0.2.5` พอร์ต `5432` และต้องการ pool_size 30 ด้วย pool_mode แบบ transaction</summary>

**เฉลย**:

```ini
[databases]
ecommerce_db = host=10.0.2.5 port=5432 dbname=ecommerce_db pool_mode=transaction pool_size=30
```

หมายเหตุ: ชื่อ database ทางซ้ายของเครื่องหมาย `=` (`ecommerce_db`) คือชื่อที่ client จะใช้เชื่อมต่อผ่าน PgBouncer ส่วน `dbname=` ทางขวาคือชื่อ database จริงบน PostgreSQL server (สามารถตั้งชื่อต่างกันได้ถ้าต้องการ เช่นให้ client เชื่อมด้วยชื่อ `ecommerce_web` แต่ไปยัง database จริงชื่อ `ecommerce_db`)

</details>

<details>
<summary><b>แบบฝึกหัดที่ 4</b>: อธิบายความแตกต่างระหว่าง `max_client_conn` และ `default_pool_size` ใน pgbouncer.ini พร้อมยกตัวอย่างตัวเลขที่เหมาะสมสำหรับระบบที่มี web server 8 ตัว แต่ละตัวเปิด connection pool ฝั่งแอปสูงสุด 50 connections</summary>

**เฉลย**:

- `max_client_conn` คือจำนวน**client connection** สูงสุดที่ PgBouncer เองรับได้ทั้งหมด (ฝั่งที่คุยกับแอปพลิเคชัน) — ควรตั้งให้ครอบคลุมผลรวมของ connection ที่ทุก web server อาจเปิดมาพร้อมกัน
- `default_pool_size` คือจำนวน**server connection** สูงสุดต่อ 1 pool ที่ PgBouncer จะเปิดไปยัง PostgreSQL จริง (ค่านี้ที่ PostgreSQL มองเห็นจริงๆ)

สำหรับระบบที่มี web server 8 ตัว × 50 connections/ตัว = สูงสุด 400 client connections ที่อาจเกิดขึ้นพร้อมกัน:

```ini
[pgbouncer]
max_client_conn = 500           ;; เผื่อ headroom เล็กน้อยเกิน 400 (รองรับ worker/monitoring เพิ่มด้วย)
default_pool_size = 25          ;; server connection จริงไปยัง PostgreSQL แค่ ~25 ตัว (อิงจาก CPU core ของ DB server)
```

หลักการคือ `max_client_conn` ควรสูงกว่า `default_pool_size` มาก เพราะจุดประสงค์ของ pooling คือให้ client จำนวนมาก "แชร์" การใช้ server connection จำนวนน้อยกว่าอย่างมีประสิทธิภาพ

</details>