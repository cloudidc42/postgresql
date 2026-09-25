# Write-Ahead Logging (WAL) เชิงลึก

**หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับโลก/ผู้เชี่ยวชาญ | Part 083**

---

## เป้าหมายการเรียนรู้

หลังจบบทนี้ ผู้เรียนจะสามารถ:

- อธิบายหลักการ write-ahead logging ในเชิงทฤษฎีและเชิงวิศวกรรมได้อย่างละเอียด และเชื่อมโยงกับคุณสมบัติ Durability ใน ACID
- อธิบายโครงสร้างของ WAL record, ความหมายของ LSN (Log Sequence Number) และวิธีที่ PostgreSQL ใช้ LSN เป็น "ที่อยู่" ของทุกการเปลี่ยนแปลงในระบบ
- อธิบายการจัดเก็บ WAL เป็นไฟล์ segment ขนาด 16MB ใน `pg_wal/` รวมถึงการตั้งชื่อไฟล์ที่อิง timeline และ segment number
- แยกแยะความแตกต่างระหว่าง `wal_level = minimal / replica / logical` และผลกระทบต่อปริมาณข้อมูลที่บันทึก
- อธิบาย checkpoint เชิงลึก ความสัมพันธ์กับ crash recovery และ WAL replay
- อธิบายขั้นตอนที่ PostgreSQL ใช้ในการกู้คืนข้อมูลหลัง crash (crash recovery) แบบ step-by-step
- ใช้ data type `pg_lsn` และฟังก์ชันที่เกี่ยวข้อง เช่น `pg_current_wal_lsn()`, `pg_wal_lsn_diff()` เพื่อวัดปริมาณ WAL และ replication lag
- ใช้เครื่องมือ `pg_waldump` เพื่อตรวจสอบเนื้อหาภายใน WAL file ระดับ record จริง
- อธิบายว่าทำไมต้องมี `full_page_writes` และกลไกป้องกัน torn page
- ลงมือปฏิบัติวิเคราะห์กิจกรรม WAL จริงในเครื่อง คำนวณ WAL generation rate ด้วยตนเอง

> **การเชื่อมโยงกับบทก่อนหน้า**: บทนี้ต่อยอดจาก Part 062 (WAL เบื้องต้น สำหรับผู้เริ่มต้น) และ Part 083 (การกล่าวถึง WAL ในบริบทของ replication และ backup) โดยเจาะลึกลงไปถึงระดับ record, LSN arithmetic และเครื่องมือ debugging ระดับ expert ที่ DBA มืออาชีพและ PostgreSQL contributor ใช้งานจริง

---

## Step 821: WAL คืออะไรเชิงลึก — Write-Ahead Principle และ Durability

### หลักการพื้นฐาน: เขียน log ก่อนเขียนข้อมูลจริงเสมอ

**Write-Ahead Logging (WAL)** คือเทคนิคพื้นฐานที่สุดอย่างหนึ่งในวิศวกรรมฐานข้อมูล หลักการคือ:

> **ก่อนที่จะแก้ไขข้อมูลจริงบนดิสก์ (data page) ระบบต้องเขียนบันทึกการเปลี่ยนแปลงนั้นลงใน log ก่อน และบันทึกนั้นต้อง flush ลงดิสก์ (fsync) เรียบร้อยแล้ว จึงจะถือว่า transaction นั้น "commit" ได้**

ฟังดูสวนทางกับสามัญสำนึก เพราะเราต้องเขียนข้อมูลถึงสองครั้ง (ครั้งแรกลง log ครั้งที่สองลง data page จริง) แต่นี่คือกุญแจสำคัญที่ทำให้ระบบฐานข้อมูลทนทานต่อการล่มได้โดยไม่สูญเสียข้อมูล

```
ลำดับเหตุการณ์แบบ Write-Ahead:

  Transaction แก้ไขข้อมูล
         │
         ▼
  ┌─────────────────────┐
  │ 1. สร้าง WAL record  │  ← อธิบาย "การเปลี่ยนแปลง" ไม่ใช่ข้อมูลทั้งหมด
  │    ใน WAL buffer     │
  └──────────┬───────────┘
             │
             ▼
  ┌─────────────────────┐
  │ 2. แก้ไข data page   │  ← แก้ไขใน shared_buffers (memory)
  │    ใน shared_buffers │     ยังไม่ถูกเขียนลง disk จริง!
  └──────────┬───────────┘
             │
             ▼
  ┌─────────────────────┐
  │ 3. COMMIT: flush WAL │  ← WAL record ต้องอยู่บน disk แล้ว
  │    ลง disk (fsync)   │     (ผ่าน pg_wal/)
  └──────────┬───────────┘
             │
             ▼
     Client ได้รับ "COMMIT" 
     (transaction ถือว่าสำเร็จแล้ว)
             │
             ▼
  ┌─────────────────────┐
  │ 4. Checkpoint (ภาย   │  ← data page จริงถูกเขียนลง disk
  │    หลัง) เขียน dirty │     "ทีหลัง" ก็ได้ เพราะ WAL
  │    pages ลง disk     │     รับประกันไว้แล้ว
  └─────────────────────┘
```

สังเกตว่า **data page จริงไม่จำเป็นต้องถูกเขียนลงดิสก์ทันทีที่ COMMIT** — สิ่งที่ต้องอยู่บนดิสก์แน่นอนคือ WAL record เท่านั้น นี่คือสาระสำคัญของชื่อ "Write-Ahead": log ต้อง "เดินหน้า" (ahead) ไปก่อนข้อมูลจริงเสมอ

### ทำไมต้องทำแบบนี้? เชื่อมโยงกับ Durability (ACID)

ทบทวนจาก ACID:

- **A**tomicity — transaction ทำสำเร็จทั้งหมดหรือไม่ทำเลย
- **C**onsistency — ฐานข้อมูลอยู่ในสถานะที่ถูกต้องตามกฎเสมอ
- **I**solation — transaction ที่ทำงานพร้อมกันไม่รบกวนกัน
- **D**urability — เมื่อ commit สำเร็จแล้ว ข้อมูลต้องไม่สูญหาย แม้ระบบจะล่มทันทีหลังจากนั้น

WAL คือกลไกหลักที่ทำให้เกิด **Durability** ลองพิจารณาสถานการณ์นี้:

```
เวลา T1: Transaction UPDATE accounts SET balance = balance - 1000 WHERE id = 1;
เวลา T2: COMMIT;  ← client ได้รับ "COMMIT" สำเร็จ
เวลา T3: ไฟดับ! เซิร์ฟเวอร์ดับกะทันหัน (ก่อนที่ data page จะถูกเขียนลง disk จริง)
เวลา T4: เซิร์ฟเวอร์กลับมาทำงาน (restart)
```

คำถามคือ: ข้อมูลที่ balance ลดลง 1000 จะยังอยู่หรือไม่?

**คำตอบ: อยู่แน่นอน** เพราะที่เวลา T2 (ตอน COMMIT) PostgreSQL ได้ fsync WAL record ของการเปลี่ยนแปลงนี้ลงดิสก์เรียบร้อยแล้ว แม้ shared_buffers (memory) จะหายไปพร้อมไฟดับ แต่เมื่อ PostgreSQL restart ที่ T4 มันจะอ่าน WAL ที่บันทึกไว้ และ **replay** (เล่นซ้ำ) การเปลี่ยนแปลงนั้นเข้าไปใน data page อีกครั้ง กระบวนการนี้เรียกว่า **crash recovery** (จะลงรายละเอียดใน Step 826)

ถ้าไม่มี WAL เลย และ PostgreSQL เขียนข้อมูลลง data page โดยตรงตอน COMMIT — ปัญหาที่จะเกิดคือ:

1. **Random I/O ช้ามาก**: การ UPDATE หนึ่งแถวอาจกระทบหลาย page (table page, index pages หลายตัว) การ fsync แต่ละ page แยกกันทุกครั้งที่ commit จะช้าอย่างมหาศาล เพราะ page เหล่านี้กระจัดกระจายอยู่คนละตำแหน่งบนดิสก์ (random I/O) ในขณะที่ WAL เป็นการเขียนแบบ **sequential** (ต่อเนื่อง) ที่เร็วกว่ามาก แม้บน spinning disk แบบเก่า
2. **Torn page**: ถ้าไฟดับขณะเขียน data page (ซึ่งมีขนาด 8KB) กลางคัน อาจได้ page ที่เขียนไปครึ่งเดียว (torn page) ทำให้ข้อมูลเสียหายโดยไม่มีทางกู้คืน (เรื่องนี้จะกล่าวถึงในรายละเอียดที่ Step 829 เรื่อง full_page_writes)
3. **ไม่มีทางกู้คืนได้เลยหากล่มกลางการเขียน**: ถ้าไม่มี log แยกต่างหาก ก็ไม่มีทางรู้ว่าการเปลี่ยนแปลงไหนสำเร็จไปแล้วบ้าง

### WAL แก้ปัญหาอย่างไร

```
┌──────────────────────────────────────────────────────────┐
│  ไม่มี WAL: เขียน data page โดยตรง                         │
│                                                            │
│  COMMIT ──▶ เขียน table page (random I/O, ช้า)             │
│         ──▶ เขียน index page 1 (random I/O, ช้า)           │
│         ──▶ เขียน index page 2 (random I/O, ช้า)           │
│         ──▶ ถ้าล่มกลางคัน = ข้อมูลเสียหาย ไม่รู้จะกู้จากไหน  │
└──────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│  มี WAL: เขียน log ก่อน                                    │
│                                                            │
│  COMMIT ──▶ เขียน WAL record เดียว (sequential, เร็ว)       │
│         ──▶ fsync WAL (รับประกัน durability)                │
│         ──▶ ตอบ client ว่า COMMIT สำเร็จ                    │
│                                                            │
│  (ภายหลัง, แยกจาก commit path)                              │
│  Background ──▶ เขียน data pages ลง disk ทีละหลาย page      │
│                  รวมกันเป็น batch (checkpoint)               │
│                  ถ้าล่มกลางคัน = replay WAL แล้วได้ผลเดิม     │
└──────────────────────────────────────────────────────────┘
```

จุดสำคัญคือ WAL แปลง "การเขียนแบบสุ่มหลายจุด" (random write) ให้กลายเป็น "การเขียนต่อเนื่องจุดเดียว" (sequential append-only write) ซึ่งเร็วกว่ามาก และยังทำให้ระบบสามารถ **เลื่อนการเขียน data page จริงออกไปได้** (deferred write) โดยไม่เสี่ยงต่อการสูญเสียข้อมูล เพราะ WAL คือ "หลักฐาน" ที่สามารถ replay เพื่อสร้างสถานะล่าสุดขึ้นมาใหม่ได้เสมอ

### กฎเหล็กของ WAL

PostgreSQL ปฏิบัติตามกฎนี้อย่างเคร่งครัดในทุกกรณี:

> **WAL record ที่อธิบายการเปลี่ยนแปลงของ page ใด ๆ ต้องถูก flush ลงดิสก์ก่อนที่ page นั้น (in its modified form) จะถูกเขียนลงดิสก์**

นี่คือกฎที่เรียกว่า **"WAL-before-data" rule** สามารถเขียนเป็นสมการง่าย ๆ ได้ว่า:

```
LSN(WAL flush to disk) >= LSN(data page's pd_lsn)
                          ก่อนที่ data page จะถูกเขียนลง disk
```

ทุก data page ใน PostgreSQL มี field พิเศษชื่อ `pd_lsn` อยู่ใน page header ซึ่งบันทึก LSN ของ WAL record ล่าสุดที่แก้ไข page นั้น ก่อน PostgreSQL จะเขียน page ใด ๆ ลงดิสก์ (เช่นตอน checkpoint หรือ background writer ทำงาน) มันจะเช็คว่า WAL ที่ LSN นั้นถูก flush ลงดิสก์แล้วหรือยัง ถ้ายังไม่ถูก flush ก็ต้อง flush WAL ก่อน แล้วจึงเขียน data page ทีหลัง — นี่คือกลไกที่บังคับให้ "write-ahead" เป็นจริงเสมอในทุกกรณี ไม่ใช่แค่ตอน COMMIT เท่านั้น

---

## Step 822: WAL Record — โครงสร้างและ LSN (Log Sequence Number)

### WAL Record คืออะไร

ทุกการเปลี่ยนแปลงในฐานข้อมูล (INSERT, UPDATE, DELETE, CREATE TABLE, VACUUM, checkpoint เอง ฯลฯ) จะถูกแปลงเป็น **WAL record** อย่างน้อยหนึ่งรายการ WAL record ไม่ได้บันทึก SQL statement ดิบ ๆ (ไม่ใช่ statement-based logging เหมือน MySQL binlog แบบ STATEMENT) แต่บันทึกเป็น **physical/logical change** ในระดับ page — พูดง่าย ๆ คือ "byte ไหนของ page ไหนเปลี่ยนจากอะไรเป็นอะไร"

โครงสร้างคร่าว ๆ ของ WAL record:

```
┌─────────────────────────────────────────────────────────────┐
│                     XLogRecord Header                        │
├─────────────────────────────────────────────────────────────┤
│ xl_tot_len   : ความยาวรวมของ record (bytes)                   │
│ xl_xid       : Transaction ID ที่สร้าง record นี้              │
│ xl_prev      : LSN ของ record ก่อนหน้า (backward link)         │
│ xl_info      : flags (เช่น เป็น commit record หรือไม่)          │
│ xl_rmid      : Resource Manager ID (เช่น Heap, Btree, XLOG)    │
│ xl_crc       : CRC32C checksum ของ record ทั้งหมด              │
├─────────────────────────────────────────────────────────────┤
│                     Record Data (payload)                    │
├─────────────────────────────────────────────────────────────┤
│ block references : รายการ page ที่ถูกกระทบ (rel, fork, blkno) │
│ main data         : ข้อมูล delta เช่น ค่า column ที่เปลี่ยน     │
│ backup block image: (ถ้ามี) full page image ทั้ง page          │
│                     (เกี่ยวกับ full_page_writes, ดู Step 829)  │
└─────────────────────────────────────────────────────────────┘
```

**Resource Manager (rmgr)** คือกลไกที่ทำให้ WAL รองรับการเปลี่ยนแปลงได้ทุกประเภท — แต่ละ subsystem ของ PostgreSQL (Heap, Btree, GIN, GiST, Hash, Sequence, SPGist, BRIN, Transaction, XLOG เอง ฯลฯ) มี resource manager ของตัวเองที่รู้วิธี "เขียน" record และ "replay" record กลับ resource manager หลัก ๆ ที่พบบ่อย:

| rmgr ID | ชื่อ | ใช้สำหรับ |
|---|---|---|
| 0 | XLOG | checkpoint, switch, backup records |
| 1 | Transaction | COMMIT, ABORT records |
| 2 | Storage | สร้าง/ลบไฟล์ relation |
| 4 | Heap2 | HOT prune, vacuum, freeze |
| 10 | Heap | INSERT, UPDATE, DELETE บน heap table |
| 11 | Btree | การเปลี่ยนแปลงใน B-tree index |
| 12 | Hash | Hash index |
| 13 | Gin | GIN index |
| 14 | Gist | GiST index |
| 20 | Generic | generic WAL (ใช้โดย extension) |

### LSN (Log Sequence Number) คืออะไร

**LSN** คือหมายเลขที่ระบุตำแหน่ง byte offset ที่แน่นอนภายในสตรีม WAL ทั้งหมดตั้งแต่ต้นฐานข้อมูล (ตั้งแต่ initdb) พูดง่าย ๆ LSN คือ **"เลขที่บ้าน" ของทุกไบต์ใน WAL** — มันเป็นตัวเลข 64-bit ที่เพิ่มขึ้นเรื่อย ๆ ไม่มีวันย้อนกลับ (monotonically increasing) ตลอดอายุของ timeline หนึ่ง ๆ

LSN แสดงผลในรูปแบบ hexadecimal สองส่วนคั่นด้วย `/`:

```
   16/B374D848
   │  └───────┴── offset ภายใน segment ที่ 0xB374D848 (32-bit ต่ำ)
   └── high 32 bits (segment file ลำดับสูง)
```

พูดให้ชัดกว่านั้น: LSN คือตัวเลข 64-bit เดียว แต่ถูก "หั่น" แสดงผลเป็นสองส่วน ส่วนบน 32-bit กับส่วนล่าง 32-bit เพื่อความอ่านง่าย เวลาคำนวณจริง PostgreSQL จะมองมันเป็นตัวเลข 64-bit ตัวเดียว:

```
LSN (64-bit) = (upper_32_bits << 32) | lower_32_bits
```

```
ตัวอย่าง: LSN = 16/B374D848

  upper = 0x16         = 22  (decimal)
  lower = 0xB374D848   = 3007984712 (decimal)

  LSN แบบ 64-bit เต็ม:
  = (22 << 32) | 3007984712
  = 94489280512 + 3007984712
  = 97497265224
```

### ความสัมพันธ์ระหว่าง LSN กับทุกสิ่งใน PostgreSQL

LSN ไม่ใช่แค่ "ตำแหน่งใน WAL file" แต่คือ **แกนเวลา (timeline reference)** ที่ระบบทั้งหมดใช้อ้างอิงร่วมกัน:

```
                     LSN แกนเวลาเดียวที่ทุกอย่างอ้างอิง
   ──────────────────────────────────────────────────────▶
   0/0          0/16B4A00       0/2A3F890        0/3C9D100
    │               │                │                │
    │               │                │                │
 initdb        Transaction A     Checkpoint       Transaction B
              เขียน WAL record    เขียนใน            COMMIT
              (pd_lsn ของ page    pg_control
               ที่แก้ถูกอัปเดต)    (redo LSN)

  ใช้ที่ไหนบ้าง:
  • pd_lsn ใน page header    — page นี้ถูกแก้ล่าสุดที่ LSN ไหน
  • pg_control               — checkpoint ล่าสุดอยู่ที่ LSN ไหน (redo point)
  • replication (streaming)  — replica sync ถึง LSN ไหนแล้ว (ดู Part 063/064)
  • pg_current_wal_lsn()     — primary เขียน WAL ไปถึงไหนแล้ว ณ ขณะนี้
  • restore_point            — จุดที่ผู้ใช้ตั้งไว้สำหรับ PITR (Point-in-Time Recovery)
```

### ทดลองดู LSN จริงในระบบ

```sql
-- LSN ปัจจุบันที่ primary เขียนไปถึง
SELECT pg_current_wal_lsn();
--  pg_current_wal_lsn
-- ---------------------
--  16/B374D848
-- (1 row)

-- LSN ที่ถูก flush ลงดิสก์แล้วจริง ๆ (อาจตามหลัง current เล็กน้อย)
SELECT pg_current_wal_flush_lsn();
--  pg_current_wal_flush_lsn
-- ---------------------------
--  16/B374D820

-- ดู LSN ของ insert record ที่เพิ่งเกิดขึ้น
CREATE TABLE demo_lsn (id serial primary key, note text);
INSERT INTO demo_lsn (note) VALUES ('hello wal');

SELECT pg_current_wal_insert_lsn() AS insert_lsn,
       pg_current_wal_lsn()        AS write_lsn,
       pg_current_wal_flush_lsn()  AS flush_lsn;
--   insert_lsn  |  write_lsn  |  flush_lsn
-- --------------+-------------+-------------
--  16/B374DA10  | 16/B374DA10 | 16/B374D9F0
```

สาม LSN นี้มีความหมายต่างกันเล็กน้อยและสำคัญมากสำหรับความเข้าใจเชิงลึก:

- **insert LSN**: ตำแหน่งที่ backend เขียนข้อมูลลง WAL buffer (ในหน่วยความจำ) ล่าสุด
- **write LSN**: ตำแหน่งที่ระบบ `write()` (syscall) ข้อมูลจาก WAL buffer ออกไปยัง OS page cache แล้ว (แต่ยังไม่รับประกันว่าอยู่บน disk จริง เพราะ OS อาจยัง cache ไว้)
- **flush LSN**: ตำแหน่งที่ข้อมูลถูก `fsync()` ลง disk จริง ๆ แล้ว — **นี่คือค่าที่รับประกัน durability**

เมื่อ transaction COMMIT, PostgreSQL จะรอจนกว่า flush LSN >= LSN ของ commit record ก่อนจึงตอบ client ว่าสำเร็จ (ยกเว้นกรณีตั้งค่า `synchronous_commit = off` ซึ่งเป็นการยอมเสีย durability บางส่วนแลกกับความเร็ว)

---

## Step 823: WAL Segment File — ไฟล์ขนาด 16MB ใน pg_wal/

### โครงสร้างไดเรกทอรี pg_wal/

WAL ไม่ได้ถูกเขียนลงไฟล์เดียวยาวไม่รู้จบ แต่ถูกแบ่งเป็นไฟล์ย่อย ๆ ที่เรียกว่า **WAL segment** ขนาดปกติ (default) คือ **16MB** ต่อไฟล์ ตั้งค่าได้ตอน `initdb` ผ่าน `--wal-segsize` (ตั้งแต่ PostgreSQL 11 เป็นต้นมา ก่อนหน้านั้นต้อง compile ใหม่)

```
$ ls -la /var/lib/postgresql/16/main/pg_wal/
total 278536
drwx------ 3 postgres postgres      4096 Sep 25 10:00 .
drwx------ 20 postgres postgres     4096 Sep 20 08:15 ..
-rw------- 1 postgres postgres  16777216 Sep 25 09:40 000000010000001600000032
-rw------- 1 postgres postgres  16777216 Sep 25 09:44 000000010000001600000033
-rw------- 1 postgres postgres  16777216 Sep 25 09:48 000000010000001600000034
-rw------- 1 postgres postgres  16777216 Sep 25 09:52 000000010000001600000035  ← current
drwx------ 2 postgres postgres      4096 Sep 25 08:00 archive_status
```

ทุกไฟล์มีขนาด **16777216 bytes = 16MB** พอดี เสมอ (ไม่ว่าจะมีข้อมูลเต็มหรือไม่ก็ตาม — PostgreSQL จะ pre-allocate พื้นที่เต็มไฟล์ไว้ล่วงหน้า ไม่ใช่ค่อย ๆ โต)

### การตั้งชื่อไฟล์ WAL segment

ชื่อไฟล์ WAL ประกอบด้วย **24 ตัวอักษร hexadecimal** แบ่งเป็น 3 ส่วน ส่วนละ 8 ตัวอักษร:

```
        000000010000001600000035
        └──┬───┘└──┬───┘└──┬───┘
           │        │        │
      Timeline   Log ID   Segment ID
       ID (TLI)  (high 32  (low 32 bits
       (8 hex)    bits of  ของ LSN หาร
                  LSN)     ด้วยขนาด
                            segment)
```

อธิบายแต่ละส่วน:

1. **Timeline ID (TLI)** — 8 หลักแรก เช่น `00000001` คือ timeline หมายเลข 1 (เพิ่มขึ้นทุกครั้งที่มี failover หรือ point-in-time recovery ไปยังจุดในอดีต — จะกล่าวถึงเชิงลึกเรื่อง timeline ใน chapter เรื่อง replication ขั้นสูง)
2. **Log ID** — 8 หลักถัดมา คือ 32-bit บนของ LSN (เท่ากับส่วนหน้าของ LSN ที่แสดงผล เช่น `16` ใน `16/B374D848` — แต่ในชื่อไฟล์จะเป็น `00000016`)
3. **Segment ID** — 8 หลักสุดท้าย คือหมายเลข segment ภายใน log ID นั้น คำนวณจาก LSN offset หารด้วยขนาด segment (16MB = 0x1000000 bytes) เมื่อ segment ID ครบ 0xFF (255) จะวนไปเพิ่ม Log ID แทน (เพราะ 0xFF+1 segment ที่ขนาด 16MB = 4GB พอดี = ขนาดของ 32-bit บนของ LSN หนึ่งหน่วย)

### สูตรคำนวณชื่อไฟล์จาก LSN

```
LSN = 16/B374D848

Step 1: แยก LSN เป็น high 32-bit และ low 32-bit
        high = 0x16
        low  = 0xB374D848

Step 2: Segment ID = low 32-bit ÷ ขนาด segment (16MB = 0x1000000)
        Segment ID = 0xB374D848 ÷ 0x1000000
                   = 0xB3  (ตัดเศษทิ้ง, floor division)

Step 3: ประกอบชื่อไฟล์
        Timeline (8 หลัก) + High (8 หลัก) + Segment ID (8 หลัก)
        = 00000001 + 00000016 + 000000B3
        = 00000001000000160000000B3   ← ผิด! ต้องเป็น 8 หลักพอดี segment ID

ชื่อไฟล์ที่ถูกต้อง: 0000000100000016000000B3
```

ใช้ฟังก์ชันในตัวของ PostgreSQL เพื่อแปลง LSN เป็นชื่อไฟล์ได้โดยตรง (ไม่ต้องคำนวณมือ):

```sql
-- แปลง LSN เป็นชื่อไฟล์ WAL segment (PostgreSQL 15+)
SELECT pg_walfile_name('16/B374D848');
--      pg_walfile_name
-- --------------------------
--  0000000100000016000000B3
-- (1 row)

-- แปลง LSN เป็นชื่อไฟล์ พร้อม offset ภายในไฟล์
SELECT * FROM pg_walfile_name_offset('16/B374D848');
--       file_name         |  file_offset
-- --------------------------+--------------
--  0000000100000016000000B3 |      3671624

-- แปลงกลับ: จากชื่อไฟล์ WAL หา LSN เริ่มต้นของไฟล์นั้น
SELECT pg_walfile_name('0000000100000016000000B3'::pg_lsn);
-- (หมายเหตุ: ในทางปฏิบัติต้อง cast อย่างระมัดระวัง 
--  ปกติใช้ pg_walfile_name(lsn) ทิศทางเดียว)
```

### ทำไมต้องแบ่งเป็น segment ขนาด 16MB (ไม่ใช่ไฟล์เดียวยาว ๆ)

1. **จัดการง่าย**: การลบ/archive/copy ไฟล์ขนาดคงที่ทำได้ง่ายกว่าไฟล์เดียวที่โตไม่จำกัด
2. **Archiving แบบ incremental**: `archive_command` ทำงานทีละไฟล์ segment พอไฟล์เต็มก็ archive ได้ทันที ไม่ต้องรอทั้งฐานข้อมูล
3. **Recycling**: PostgreSQL สามารถนำไฟล์ segment เก่าที่ไม่จำเป็นแล้ว (เช่นผ่าน checkpoint และไม่มี replica ไหนต้องการแล้ว) มา **rename และใช้ซ้ำ** แทนการสร้างไฟล์ใหม่ ประหยัดเวลาในการ allocate พื้นที่ดิสก์ใหม่ (แม้ปัจจุบันหลาย filesystem จะเร็วพอจนความแตกต่างไม่มากนัก แต่ยังเป็น default behavior)
4. **จำกัดความเสียหาย**: หากไฟล์หนึ่งเสียหาย (corrupt) ผลกระทบจำกัดอยู่แค่ 16MB นั้น ไม่ใช่ทั้งประวัติศาสตร์ WAL

### พารามิเตอร์ที่เกี่ยวข้อง

```sql
-- ดูขนาด segment ปัจจุบัน (ตั้งค่าตอน initdb เท่านั้น แก้ทีหลังไม่ได้)
SHOW wal_segment_size;
--  wal_segment_size
-- -------------------
--  16MB

-- คำนวณจำนวนไฟล์สูงสุดที่จะเก็บไว้ (ป้องกัน disk เต็ม)
SHOW max_wal_size;      -- default 1GB  (soft limit ที่ checkpoint พยายามไม่เกิน)
SHOW min_wal_size;      -- default 80MB (จำนวนขั้นต่ำที่เก็บไว้ recycle)
```

---

## Step 824: WAL Levels — minimal, replica, logical

### ทบทวนแนวคิดจาก Part 063/064

พารามิเตอร์ `wal_level` กำหนดว่า WAL จะบันทึกข้อมูล "มากแค่ไหน" — ยิ่งบันทึกมาก ยิ่งใช้พื้นที่ดิสก์และ I/O มากขึ้น แต่ก็เปิดใช้ฟีเจอร์ระดับสูงได้มากขึ้นตามไปด้วย ค่าที่ใช้ได้ (ตั้งแต่ PostgreSQL 10 เป็นต้นมา) มี 3 ระดับ:

```
        minimal          replica          logical
    ┌──────────┐     ┌──────────────┐  ┌──────────────────┐
    │  น้อยสุด  │ ──▶ │  ปานกลาง      │─▶│  มากสุด            │
    │  เร็วสุด  │     │  (default)    │  │  ใช้พื้นที่มากสุด    │
    └──────────┘     └──────────────┘  └──────────────────┘
    
    รองรับ:             รองรับเพิ่ม:         รองรับเพิ่ม:
    - crash recovery    - WAL archiving      - logical decoding
                         - streaming          - logical replication
                           replication        - CDC (Change Data
                         - PITR                 Capture)
                         - hot standby
```

### minimal

```sql
-- wal_level = minimal
```

บันทึกเฉพาะข้อมูลที่จำเป็น **สำหรับ crash recovery เท่านั้น** — คือให้ instance เดียวกันสามารถกู้คืนตัวเองได้หากล่ม แต่ **ไม่รองรับการส่ง WAL ไปที่อื่น** (ไม่มี replication, ไม่มี archiving, ไม่มี PITR)

จุดพิเศษของ `minimal`: PostgreSQL สามารถข้าม (skip) การเขียน WAL สำหรับบาง operation ได้เลย เช่น `CREATE TABLE` ตามด้วย `COPY` จำนวนมากในทรานแซกชันเดียวกัน (ถ้าตารางถูกสร้างใหม่ในทรานแซกชันเดียวกับที่ใส่ข้อมูล PostgreSQL รู้ว่าถ้าล่มกลางคัน table ทั้งตารางจะหายไปพร้อมกันอยู่แล้ว เพราะไม่เคย commit จึงไม่ต้องบันทึก WAL ของการ insert แต่ละแถวเลยก็ได้ — เขียนแค่ metadata การสร้างไฟล์)

```
ข้อจำกัดสำคัญ: ในโหมด minimal ห้ามมี standby หรือ backup ใด ๆ 
ทำงานคู่กับ instance นี้ได้เลย เพราะ WAL ไม่มีข้อมูลพอที่จะ
สร้าง replica หรือ point-in-time recovery ได้
```

ปัจจุบันแทบไม่มีใครใช้ `minimal` ในระบบ production จริงจัง เพราะเสียความสามารถในการทำ backup/replication ไปหมด แลกกับความเร็วที่ได้เพิ่มมาไม่มากนักเมื่อเทียบกับความเสี่ยง

### replica (default)

```sql
SHOW wal_level;
--  wal_level
-- -----------
--  replica
```

นี่คือค่า **default** ของ PostgreSQL สมัยใหม่ (ตั้งแต่ PostgreSQL 9.6 เป็นต้นไป ระดับ `hot_standby` และ `archive` แบบเก่าถูกรวมเป็น `replica`) บันทึกข้อมูลเพียงพอสำหรับ:

- **WAL archiving** (`archive_mode = on`) — เก็บ WAL segment ไว้ทำ backup/PITR
- **Streaming replication** — ส่ง WAL ไปยัง standby server
- **Hot standby** — standby สามารถรับ read-only query ได้ระหว่าง apply WAL

`replica` **ไม่รองรับ logical decoding** ดังนั้นถ้าต้องการใช้ logical replication หรือเครื่องมือ CDC เช่น Debezium ต้องยกระดับเป็น `logical`

### logical

```sql
-- wal_level = logical
```

ระดับสูงสุด บันทึกข้อมูลเพิ่มเติมเหนือจาก `replica` คือ **ข้อมูลที่จำเป็นสำหรับ logical decoding**:

- catalog metadata ที่เพียงพอสำหรับแปลง WAL record กลับเป็น "การเปลี่ยนแปลงเชิงตรรกะ" (logical change) เช่น "แถวนี้ถูก insert เข้า table ชื่อ orders คอลัมน์ id=5, amount=100"
- ข้อมูลที่ระบุ replica identity ของแต่ละแถวที่ถูก UPDATE/DELETE (เพื่อให้ logical replication รู้ว่าแถวไหนที่ปลายทางต้องอัปเดต)

เปิดใช้เพื่อรองรับ:

- `CREATE PUBLICATION` / `CREATE SUBSCRIPTION` (logical replication ในตัว PostgreSQL)
- Logical replication slots สำหรับเครื่องมือภายนอก เช่น Debezium, pglogical, AWS DMS

```sql
-- ตัวอย่างการเปลี่ยน wal_level (ต้อง restart เพราะเป็น postmaster-context parameter)
ALTER SYSTEM SET wal_level = logical;
-- แล้วสั่ง restart service
-- $ sudo systemctl restart postgresql
```

### เปรียบเทียบปริมาณ WAL ที่เกิดขึ้นจริง

```sql
-- ทดลองวัดปริมาณ WAL ที่เกิดจาก workload เดียวกัน ที่ wal_level ต่างกัน
-- (ต้อง restart เพื่อเปลี่ยน wal_level แต่ละครั้ง)

-- ก่อนเริ่ม
SELECT pg_current_wal_lsn() AS start_lsn \gset

-- Workload เดียวกัน: insert 100,000 แถว
INSERT INTO big_table (data)
SELECT md5(g::text) FROM generate_series(1, 100000) g;

-- วัดปริมาณ WAL ที่เกิดขึ้น
SELECT pg_size_pretty(
    pg_wal_lsn_diff(pg_current_wal_lsn(), :'start_lsn')
) AS wal_generated;
```

ผลลัพธ์โดยประมาณ (ตัวเลขจริงขึ้นกับ schema, index, ฯลฯ):

| wal_level | WAL generated (workload เดียวกัน) | หมายเหตุ |
|---|---|---|
| minimal | ~8.2 MB | น้อยสุด แต่ใช้จริงไม่ได้ในระบบที่ต้อง replicate |
| replica | ~8.9 MB | เพิ่มขึ้นเล็กน้อย (~5-10%) จาก info เพิ่มเพื่อ hot standby |
| logical | ~9.6 MB | เพิ่มขึ้นอีกจาก replica (เก็บ old row values สำหรับ REPLICA IDENTITY) |

> **ข้อสังเกตเชิงลึก**: ความแตกต่างระหว่าง `replica` กับ `logical` มักไม่มากอย่างที่คาดหวัง ยกเว้นตารางที่มี `REPLICA IDENTITY FULL` ซึ่งจะทำให้ UPDATE/DELETE ต้องบันทึกค่าเดิมของ**ทุกคอลัมน์** (ไม่ใช่แค่ primary key) ทำให้ WAL โตขึ้นอย่างมีนัยสำคัญสำหรับตารางที่มีคอลัมน์จำนวนมากหรือมีขนาดใหญ่ (เช่น text/jsonb column)

---

## Step 825: Checkpoint คืออะไรเชิงลึก

### นิยาม

**Checkpoint** คือจุดเวลาที่ PostgreSQL รับประกันว่า **dirty page ทั้งหมดใน shared_buffers ที่มี pd_lsn น้อยกว่าหรือเท่ากับ checkpoint's "redo point" ได้ถูกเขียน (flush) ลงดิสก์เรียบร้อยแล้ว**

พูดง่าย ๆ checkpoint คือการประกาศว่า "ณ จุดนี้ ข้อมูลบนดิสก์ (data files) สอดคล้องกับสถานะของฐานข้อมูลจนถึง LSN ค่าหนึ่งแล้วอย่างแน่นอน — ไม่ต้องพึ่ง WAL ก่อนหน้าจุดนี้อีกต่อไปในการ recovery"

```
Timeline ของ WAL และ checkpoint:

  LSN:    0     100    200    300    400    500    600    700
          │      │      │      │      │      │      │      │
          ▼      ▼      ▼      ▼      ▼      ▼      ▼      ▼
  ────────●──────●──────●──────●──────●──────●──────●──────●──▶
          │      │             │             │
      Insert A  Update B   CHECKPOINT     Insert C   ← ปัจจุบัน
                            (redo=250)
                            
  ที่ redo point = 250:
  - Insert A (LSN 100) และ Update B (LSN 200) รับประกันว่า
    data page ของทั้งสองถูกเขียนลง disk แล้วแน่นอน
  - หากเกิด crash ที่ LSN 700, การ recovery จะเริ่ม replay
    WAL จาก LSN 250 เท่านั้น ไม่ต้องย้อนไปถึง LSN 0
```

### กระบวนการภายในของ checkpoint

Checkpoint ไม่ใช่แค่ "เขียน dirty pages ทั้งหมดทันที" (ซึ่งจะทำให้ I/O พุ่งกระทันหันและกระทบ performance) แต่มีขั้นตอนที่ออกแบบมาอย่างระมัดระวัง:

```
┌────────────────────────────────────────────────────────────┐
│                    CHECKPOINT PROCESS                        │
├────────────────────────────────────────────────────────────┤
│                                                               │
│ 1. บันทึก "REDO point" ชั่วคราว                                │
│    = LSN ปัจจุบัน ณ จุดเริ่ม checkpoint                        │
│                                                               │
│ 2. เขียน checkpoint WAL record (ชนิด XLOG_CHECKPOINT_ONLINE)  │
│                                                               │
│ 3. ค่อย ๆ เขียน (flush) dirty buffers ทั้งหมดใน shared_buffers │
│    ที่มีอยู่ ณ เวลาเริ่ม checkpoint ลงดิสก์                     │
│    → กระจายการเขียนออกตามเวลา (checkpoint spreading)          │
│      ควบคุมด้วย checkpoint_completion_target                  │
│      (ไม่ dump ทีเดียวหมด ป้องกัน I/O spike)                   │
│                                                               │
│ 4. fsync data files ทั้งหมดที่ถูกแก้ไข                         │
│                                                               │
│ 5. อัปเดต pg_control ให้ระบุ:                                  │
│    - checkPoint LSN (ตำแหน่ง checkpoint record)               │
│    - REDO location (จุดที่ recovery ต้องเริ่ม replay จากตรงนี้) │
│                                                               │
│ 6. Checkpoint เสร็จสมบูรณ์ → WAL ก่อนหน้า REDO point           │
│    ไม่จำเป็นสำหรับ crash recovery อีกต่อไป (recycle ได้)        │
│                                                               │
└────────────────────────────────────────────────────────────┘
```

**จุดสำคัญ**: REDO point ที่บันทึกใน pg_control **ไม่ใช่ LSN ตอนที่ checkpoint เขียนเสร็จ** แต่คือ LSN ตอนที่ checkpoint **เริ่มต้น** (ก่อนขั้นตอนที่ 3) เพราะระหว่างที่กำลัง flush dirty buffers อยู่นั้น อาจมี transaction ใหม่เข้ามาแก้ไข page เดิมซ้ำอีกได้ (เกิด dirty page ใหม่) checkpoint ต้อง "ระแวดระวัง" LSN ทุกตัวตั้งแต่จุดเริ่มต้น ไม่ใช่แค่ ณ ตอนจบ

### สาเหตุที่ทำให้เกิด checkpoint

```sql
-- พารามิเตอร์ที่ควบคุมความถี่ของ checkpoint
SHOW checkpoint_timeout;              -- default 5min (เวลาสูงสุดระหว่าง checkpoint)
SHOW max_wal_size;                    -- default 1GB (ปริมาณ WAL สูงสุดก่อน checkpoint)
SHOW checkpoint_completion_target;    -- default 0.9 (กระจายงานเขียนตลอด 90% ของช่วงเวลา)
```

Checkpoint เกิดขึ้นได้จาก 3 สาเหตุหลัก:

1. **Time-based**: ครบเวลา `checkpoint_timeout` (default 5 นาที) นับจาก checkpoint ก่อนหน้า
2. **WAL-size-based**: ปริมาณ WAL ที่สร้างขึ้นตั้งแต่ checkpoint ล่าสุดใกล้ถึง `max_wal_size` (เพื่อจำกัดปริมาณ WAL ที่ต้อง replay หาก crash — และจำกัดพื้นที่ดิสก์ที่ WAL ใช้)
3. **Manual**: ผู้ใช้สั่ง `CHECKPOINT;` โดยตรง (มักใช้ก่อนทำ maintenance งานสำคัญ เช่น physical backup)

```sql
-- ตรวจสอบสถิติ checkpoint ล่าสุด (PostgreSQL 17+ ใช้ pg_stat_checkpointer)
SELECT num_timed, num_requested, 
       write_time, sync_time,
       buffers_written
FROM pg_stat_checkpointer;
--  num_timed | num_requested | write_time | sync_time | buffers_written
-- -----------+---------------+------------+-----------+-----------------
--        842 |            37 |   12093822 |    340211 |         9184023

-- (PostgreSQL 16 และก่อนหน้า ใช้ pg_stat_bgwriter สำหรับข้อมูลเดียวกัน)
SELECT checkpoints_timed, checkpoints_req,
       checkpoint_write_time, checkpoint_sync_time
FROM pg_stat_bgwriter;
```

ตีความ: `checkpoints_timed` สูงกว่า `checkpoints_req` มาก ๆ ถือว่าดี (แปลว่า checkpoint ทำงานตามตารางเวลาปกติ ไม่ได้ถูก "บังคับ" ให้เกิดถี่เกินไปเพราะ WAL โตเร็วกว่าที่คาดไว้ — ถ้า `checkpoints_req` สูงเกินไปมักหมายถึง `max_wal_size` ตั้งค่าไว้เล็กเกินไปเทียบกับ write workload ทำให้ checkpoint ถี่ผิดปกติ ซึ่งกินทรัพยากร I/O มาก)

### ความสัมพันธ์กับ crash recovery (เกริ่นนำ Step 826)

```
              REDO point                 crash เกิดขึ้นที่นี่
                  │                              │
                  ▼                              ▼
  ────────────────●──────────────────────────────✕──────▶  LSN
                   │◄──────── WAL ที่ต้อง replay ─────────►│
                   
  เวลาที่ใช้ recovery ∝ ปริมาณ WAL ระหว่าง REDO point ล่าสุด
                        กับจุดที่ crash เกิดขึ้น
```

Checkpoint ที่เกิดถี่ขึ้น → WAL ระหว่าง checkpoint สั้นลง → **crash recovery เร็วขึ้น** (เพราะ replay WAL น้อยกว่า) แต่ก็แลกมาด้วย I/O overhead ที่สูงขึ้นระหว่างการทำงานปกติ นี่คือ trade-off คลาสสิกที่ DBA ต้องปรับจูนตามลักษณะงาน — production ที่ทนต่อ downtime สั้น ๆ ได้ยากมักตั้ง `checkpoint_timeout` สูงขึ้น (เช่น 15-30 นาที) พร้อม `max_wal_size` ที่ใหญ่ขึ้นตามไปด้วย เพื่อลด I/O overhead แลกกับเวลา recovery ที่นานขึ้นเล็กน้อย

---

## Step 826: Crash Recovery — เมื่อ PostgreSQL Restart หลัง Crash

### ภาพรวมกระบวนการ

เมื่อ PostgreSQL เริ่มทำงาน (startup) มันจะตรวจสอบสถานะของ `pg_control` file ก่อนเสมอ ถ้าพบว่าฐานข้อมูลไม่ได้ shutdown อย่างสะอาด (`clean shutdown`) แต่ค้างอยู่ในสถานะ `in production` (แปลว่าล่มกลางคัน) จะเข้าสู่กระบวนการ **crash recovery** โดยอัตโนมัติทันที

```
┌─────────────────────────────────────────────────────────────┐
│                   PostgreSQL Startup Process                 │
├─────────────────────────────────────────────────────────────┤
│                                                                │
│  1. อ่าน pg_control                                            │
│     ┌─────────────────────────────────┐                       │
│     │ Database cluster state:          │                      │
│     │   "in production" ← ไม่ได้ shutdown │                      │
│     │                     สะอาด! เกิด    │                      │
│     │                     crash ก่อนหน้า │                      │
│     │ Latest checkpoint's REDO location:│                      │
│     │   16/A0000028                     │                      │
│     └─────────────────────────────────┘                       │
│                       │                                       │
│                       ▼                                       │
│  2. เข้าสู่ RECOVERY MODE                                       │
│     log: "database system was not properly shut down;         │
│           automatic recovery in progress"                     │
│                       │                                       │
│                       ▼                                       │
│  3. เริ่มอ่าน WAL จาก REDO location (16/A0000028)               │
│     ไล่อ่านทีละ WAL record ตามลำดับ LSN                        │
│                       │                                       │
│                       ▼                                       │
│  4. REPLAY แต่ละ record:                                       │
│     - เปิด page ที่ record นี้อ้างถึง                          │
│     - เช็ค: page's pd_lsn >= record's LSN หรือไม่?              │
│       ├─ ใช่ (page ใหม่กว่า record นี้แล้ว) → ข้าม (idempotent) │
│       └─ ไม่ → apply การเปลี่ยนแปลงลง page นั้น                  │
│                       │                                       │
│                       ▼                                       │
│  5. ไล่ replay จนถึง WAL record สุดท้ายที่อ่านได้ (หรือจนกว่า    │
│     จะเจอ record ที่ corrupt/incomplete ซึ่งถือเป็นจุดสิ้นสุด    │
│     ของ valid WAL — เพราะ record สุดท้ายอาจเขียนไม่จบตอน crash) │
│                       │                                       │
│                       ▼                                       │
│  6. ทำ CHECKPOINT ใหม่ทันที (end-of-recovery checkpoint)        │
│     บันทึกสถานะล่าสุดลง pg_control                              │
│                       │                                       │
│                       ▼                                       │
│  7. เปลี่ยนสถานะ pg_control เป็น "in production" (ปกติ)         │
│     เปิดรับ connection จาก client ได้                          │
│                                                                │
└─────────────────────────────────────────────────────────────┘
```

### ทำไมการ replay จึงปลอดภัย (Idempotency)

จุดสำคัญที่สุดของ crash recovery คือ WAL replay ต้องเป็น **idempotent** — หมายความว่าไม่ว่าจะ replay record เดิมซ้ำกี่ครั้งก็ตาม (เช่น ถ้า crash เกิดขึ้นซ้ำระหว่างกำลัง recovery อยู่) ผลลัพธ์สุดท้ายต้องเหมือนเดิมเสมอ ไม่ทำให้ข้อมูลผิดเพี้ยนซ้ำสอง (เช่น การบวกเงินซ้ำสองครั้ง)

กลไกที่ทำให้เกิด idempotency คือการเช็ค `pd_lsn` ที่กล่าวถึงในขั้นตอนที่ 4 ข้างต้น:

```
ตัวอย่าง: WAL record ที่ LSN 16/A0000100 บอกว่า
          "เพิ่มค่า balance คอลัมน์ที่ offset 40 ใน block 5 อีก 100"

ระหว่าง replay:
  - เปิด block 5 ขึ้นมา ดู pd_lsn ของ page นั้น
  
  กรณี A: pd_lsn ของ page = 16/A0000050 (< 16/A0000100)
          → page นี้ยังไม่เคยได้รับการเปลี่ยนแปลงจาก record นี้
          → APPLY การเปลี่ยนแปลง แล้วอัปเดต pd_lsn = 16/A0000100
          
  กรณี B: pd_lsn ของ page = 16/A0000100 หรือมากกว่า
          → page นี้ได้รับการเปลี่ยนแปลงนี้ไปแล้ว (อาจจาก
            การ replay รอบก่อนที่ล่มไปกลางคัน)
          → ข้าม ไม่ apply ซ้ำ (ป้องกัน double-apply)
```

นี่คือเหตุผลที่ **ทุก data page ต้องมี pd_lsn กำกับอยู่เสมอ** — มันทำหน้าที่เป็น "ตราประทับเวลา" ที่บอกว่า page เวอร์ชันนี้ทันสมัยแค่ไหน เทียบกับ WAL record

### ตัวอย่างข้อความ log จริงระหว่าง crash recovery

```
2026-09-25 09:15:02.104 UTC [1042] LOG:  database system was not properly shut down; automatic recovery in progress
2026-09-25 09:15:02.108 UTC [1042] LOG:  redo starts at 16/A0000028
2026-09-25 09:15:02.341 UTC [1042] LOG:  invalid record length at 16/A3F91A08: expected at least 24, got 0
2026-09-25 09:15:02.341 UTC [1042] LOG:  redo done at 16/A3F919E0 system usage: CPU: user: 0.18 s, system: 0.05 s, elapsed: 0.23 s
2026-09-25 09:15:02.341 UTC [1042] LOG:  last completed transaction was at log time 2026-09-25 09:14:58.902112+00
2026-09-25 09:15:02.412 UTC [1042] LOG:  checkpoint starting: end-of-recovery immediate
2026-09-25 09:15:02.498 UTC [1042] LOG:  checkpoint complete: wrote 128 buffers (0.8%); 0 WAL file(s) added, 0 removed, 2 recycled
2026-09-25 09:15:02.502 UTC [1042] LOG:  database system is ready to accept connections
```

สังเกตบรรทัด `invalid record length at 16/A3F91A08: expected at least 24, got 0` — นี่คือ **เรื่องปกติ** ไม่ใช่ error ที่น่ากังวล มันหมายความว่า recovery เจอตำแหน่งที่ WAL record เขียนไม่สมบูรณ์ (เพราะ crash เกิดขึ้นระหว่างกำลังเขียน record นั้นพอดี) ซึ่งคือ "จุดสิ้นสุดของ valid WAL" — PostgreSQL หยุด replay ที่จุดนี้พอดี เพราะรู้ว่า record ที่เขียนไม่จบไม่มีทางถูกนำไปใช้ยืนยัน COMMIT กับ client ได้อยู่แล้ว (ถ้า COMMIT record เขียนไม่จบ = client ไม่เคยได้รับคำตอบ COMMIT สำเร็จ = ไม่ผิดที่จะไม่ apply)

### แนวคิดสำคัญ: Redo point ไม่ใช่จุดเริ่มต้นของ "ข้อมูลที่ยังไม่ถูกเขียน" ทั้งหมด

หลายคนเข้าใจผิดว่า WAL ระหว่าง checkpoint ก่อนหน้ากับปัจจุบันคือ "ข้อมูลที่ยังไม่ถูกเขียนลงดิสก์เลย" — ในความเป็นจริง background writer (bgwriter) และ checkpointer process อาจเขียน dirty page บางส่วนลงดิสก์ไปแล้วก่อนถึง checkpoint ครั้งถัดไปด้วยซ้ำ (เพื่อกระจายภาระ I/O) กลไก `pd_lsn` idempotency check ที่กล่าวไปข้างต้นนี่เองที่ทำให้ recovery ไม่สนใจว่า page ไหนถูกเขียนไปแล้วหรือยัง — มัน replay WAL ทุก record ตั้งแต่ REDO point เสมอ แล้วปล่อยให้กลไก pd_lsn เป็นตัวตัดสินว่า record ไหนต้อง apply จริง ไม่ต้อง track สถานะที่ซับซ้อนกว่านั้น — นี่คือความสง่างามของการออกแบบ (design elegance) ที่ทำให้ recovery ง่ายและถูกต้องเสมอ

---

## Step 827: pg_lsn Data Type และฟังก์ชันที่เกี่ยวข้อง

### pg_lsn คือ data type

PostgreSQL มี data type พิเศษชื่อ `pg_lsn` สำหรับเก็บค่า LSN โดยเฉพาะ (ภายในคือ 64-bit unsigned integer) รองรับ operator เปรียบเทียบและคำนวณระยะห่างได้

```sql
-- สร้าง column ชนิด pg_lsn ได้โดยตรง
CREATE TABLE wal_snapshots (
    id serial PRIMARY KEY,
    captured_at timestamptz DEFAULT now(),
    lsn pg_lsn
);

INSERT INTO wal_snapshots (lsn) VALUES (pg_current_wal_lsn());

-- เปรียบเทียบ LSN ได้โดยตรงด้วย operator < > = 
SELECT lsn > '16/A0000000'::pg_lsn FROM wal_snapshots;
```

### ฟังก์ชันหลักที่เกี่ยวกับ WAL/LSN

```sql
-- 1. LSN ปัจจุบันที่ backend เขียนเข้า WAL buffer (เฉพาะ primary)
SELECT pg_current_wal_insert_lsn();

-- 2. LSN ที่ write() ออกไปยัง OS แล้ว (เฉพาะ primary)
SELECT pg_current_wal_lsn();

-- 3. LSN ที่ fsync ลงดิสก์แล้วแน่นอน (เฉพาะ primary) -- สำคัญที่สุดสำหรับ durability
SELECT pg_current_wal_flush_lsn();

-- 4. บน replica: LSN ล่าสุดที่ replay (apply) ไปแล้ว
SELECT pg_last_wal_replay_lsn();

-- 5. บน replica: LSN ล่าสุดที่ receive จาก primary แล้ว (แต่อาจยังไม่ apply)
SELECT pg_last_wal_receive_lsn();

-- 6. คำนวณระยะห่างระหว่าง LSN สองค่า (คืนค่าเป็น bytes, ชนิด numeric)
SELECT pg_wal_lsn_diff('16/B374D848', '16/A0000000');
--  pg_wal_lsn_diff
-- ------------------
--     306184776

-- 7. แปลง LSN เป็นชื่อไฟล์ WAL segment
SELECT pg_walfile_name('16/B374D848');

-- 8. คำนวณ % การใช้พื้นที่ WAL เทียบกับ max_wal_size
SELECT pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), '0/0')) AS total_wal_written;
```

### การใช้งานจริงที่สำคัญที่สุด: วัด Replication Lag แบบละเอียด

ในระบบ streaming replication คำถามที่พบบ่อยที่สุดคือ "standby ตามหลัง primary อยู่กี่ byte / กี่วินาที?" การวัดด้วย LSN arithmetic ให้ความละเอียดสูงกว่าการวัดด้วยเวลาเพียงอย่างเดียว เพราะบอกปริมาณ **ข้อมูล** ที่ค้างอยู่ ไม่ใช่แค่เวลาที่ผ่านไป

```
                     PRIMARY                        STANDBY
        
   pg_current_wal_lsn()                    pg_last_wal_receive_lsn()
        │                                          │
        │◄──────────── streaming ────────────────►│
        │              replication                 │
        ▼                                          ▼
    16/B374D848                              16/B374A020
        │                                          │
        └──────────────────┬───────────────────────┘
                            │
                pg_wal_lsn_diff() = ปริมาณ WAL ที่ยังไม่ถึง standby (bytes)
                            
                                                     │
                                             pg_last_wal_replay_lsn()
                                                     │
                                                     ▼
                                               16/B3749800
                            
                pg_wal_lsn_diff(receive_lsn, replay_lsn)
                = WAL ที่รับมาแล้วแต่ยัง apply ไม่ทัน (replay lag)
```

**บน primary** สามารถดูสถานะของทุก standby ผ่าน `pg_stat_replication`:

```sql
SELECT
    application_name,
    client_addr,
    state,
    pg_current_wal_lsn()                              AS primary_lsn,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn,
    pg_wal_lsn_diff(pg_current_wal_lsn(), sent_lsn)    AS send_lag_bytes,
    pg_wal_lsn_diff(sent_lsn, write_lsn)               AS write_lag_bytes,
    pg_wal_lsn_diff(write_lsn, flush_lsn)              AS flush_lag_bytes,
    pg_wal_lsn_diff(flush_lsn, replay_lsn)             AS replay_lag_bytes,
    pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn)  AS total_lag_bytes,
    write_lag,    -- interval, PostgreSQL 10+
    flush_lag,    -- interval
    replay_lag    -- interval
FROM pg_stat_replication;
```

ตัวอย่างผลลัพธ์:

```
 application_name | client_addr  |   state   | primary_lsn |  sent_lsn   | replay_lsn  | total_lag_bytes | replay_lag
-------------------+-------------+-----------+--------------+-------------+-------------+------------------+-------------
 standby-fra-1     | 10.0.4.22   | streaming | 16/B374D848  | 16/B374D848 | 16/B374A020 |          307272  | 00:00:00.891
 standby-fra-2     | 10.0.4.23   | streaming | 16/B374D848  | 16/B374D848 | 16/B3749800 |          311624  | 00:00:01.203
```

การมี lag ทั้งในหน่วย **bytes** (จาก LSN diff) และ **เวลา** (จาก `*_lag` interval) พร้อมกัน ช่วยให้วินิจฉัยปัญหาได้แม่นยำกว่า:

- ถ้า lag bytes น้อย แต่ lag เวลานาน → เครือข่ายหรือ replay ทำงานช้าผิดปกติเมื่อเทียบกับปริมาณข้อมูล (เช่น standby CPU ไม่พอ, I/O ช้า)
- ถ้า lag bytes มาก และ lag เวลาก็นานตามสัดส่วน → primary กำลังสร้าง WAL เร็วมาก (write-heavy workload) ตามธรรมชาติ ไม่ใช่ความผิดปกติของ standby

### บน replica: เช็คสถานะตัวเอง

```sql
-- บน standby: ดูว่าตัวเองอยู่ใน recovery mode หรือไม่
SELECT pg_is_in_recovery();
--  pg_is_in_recovery
-- --------------------
--  t

-- บน standby: คำนวณ lag ของตัวเองเทียบกับเวลาปัจจุบัน (last replayed transaction timestamp)
SELECT now() - pg_last_xact_replay_timestamp() AS replication_delay;
--  replication_delay
-- --------------------
--  00:00:01.884213

-- บน standby: LSN ที่ receive มาแล้ว vs LSN ที่ replay (apply) แล้ว
SELECT
    pg_last_wal_receive_lsn() AS received,
    pg_last_wal_replay_lsn()  AS replayed,
    pg_wal_lsn_diff(pg_last_wal_receive_lsn(), pg_last_wal_replay_lsn()) AS pending_bytes;
--   received    |   replayed    | pending_bytes
-- ---------------+---------------+----------------
--  16/B374D848   | 16/B374A020   |         307272
```

### ทำไมวัดเป็น byte-precision ถึงสำคัญกว่าดูแค่เวลา

ในระบบที่ต้องตัดสินใจแบบอัตโนมัติ เช่น load balancer ที่ต้อง route read query ไป standby ตัวไหน หรือระบบ monitoring ที่ต้อง alert ก่อนที่ lag จะเป็นปัญหาจริง การวัดเป็น **bytes ผ่าน pg_lsn arithmetic** ให้ค่าที่ deterministic และเปรียบเทียบข้าม metric อื่นได้ตรงไปตรงมากว่า (เช่น เทียบกับ bandwidth เครือข่ายที่มีอยู่ เพื่อประมาณเวลาที่ standby จะตามทัน) ในขณะที่ interval-based lag (`replay_lag`) อาจผันผวนได้จาก clock skew หรือช่วงที่ primary ไม่มี write เลย (ซึ่งจะทำให้ interval แสดงค่าเก่าค้างไว้ ไม่อัปเดต)

---

## Step 828: pg_waldump — เครื่องมือ Inspect เนื้อหา WAL ระดับ Record

### pg_waldump คืออะไร

`pg_waldump` (เดิมชื่อ `pg_xlogdump` ก่อน PostgreSQL 10) เป็นเครื่องมือ command-line ที่มาพร้อมกับการติดตั้ง PostgreSQL standard ใช้สำหรับอ่านและแสดงเนื้อหา **ดิบ** ของไฟล์ WAL segment ออกมาเป็นรูปแบบที่มนุษย์อ่านได้ ทีละ record มีประโยชน์มากสำหรับ:

- Debugging ปัญหา replication หรือ corruption
- ทำความเข้าใจว่า operation หนึ่ง ๆ สร้าง WAL record กี่รายการ ชนิดใดบ้าง
- ตรวจสอบว่า full page write เกิดขึ้นเมื่อไหร่ (ดู Step 829)
- วิเคราะห์ WAL generation pattern เพื่อ capacity planning

### วิธีใช้งานพื้นฐาน

```bash
# ตรวจสอบว่ามี pg_waldump ติดตั้งอยู่หรือไม่ (มาพร้อม PostgreSQL server package)
$ pg_waldump --version
pg_waldump (PostgreSQL) 16.4

# ดู help
$ pg_waldump --help
```

**ต้องรันด้วยสิทธิ์ที่อ่านไฟล์ใน pg_wal/ ได้** (โดยทั่วไปคือ user `postgres` หรือ root):

```bash
# วิธีที่ 1: ระบุไฟล์ WAL segment ตรง ๆ
$ pg_waldump /var/lib/postgresql/16/main/pg_wal/000000010000001600000035

# วิธีที่ 2: ระบุ path ของ pg_wal directory (ต้อง cd เข้าไปหรือใช้ full path)
$ cd /var/lib/postgresql/16/main
$ pg_waldump pg_wal/000000010000001600000035

# วิธีที่ 3: ระบุช่วง LSN ที่ต้องการ (ข้ามการ dump ทั้งไฟล์)
$ pg_waldump -p pg_wal -s 16/B3740000 -e 16/B3750000
```

### ตัวอย่างผลลัพธ์จริง

```bash
$ pg_waldump -p /var/lib/postgresql/16/main/pg_wal \
    -s 16/B374D800 -e 16/B374DA00

rmgr: Heap        len (rec/tot):     54/    54, tx:       48291, lsn: 16/B374D800, prev 16/B374D7C8, desc: INSERT off: 12, flags: 0x00, blkref #0: rel 1663/16412/16420 blk 3847
rmgr: Btree       len (rec/tot):     64/    64, tx:       48291, lsn: 16/B374D838, prev 16/B374D800, desc: INSERT_LEAF off: 145, blkref #0: rel 1663/16412/16423 blk 512
rmgr: Transaction len (rec/tot):     34/    34, tx:       48291, lsn: 16/B374D878, prev 16/B374D838, desc: COMMIT 2026-09-25 09:52:11.204112 UTC
rmgr: Heap        len (rec/tot):     58/  8218, tx:       48292, lsn: 16/B374D89C, prev 16/B374D878, desc: INSERT off: 13, flags: 0x00, blkref #0: rel 1663/16412/16420 blk 3847 FPW
rmgr: Btree       len (rec/tot):     64/    64, tx:       48292, lsn: 16/B374DA10, prev 16/B374D89C, desc: INSERT_LEAF off: 146, blkref #0: rel 1663/16412/16423 blk 512
rmgr: Transaction len (rec/tot):     34/    34, tx:       48292, lsn: 16/B374DA34, prev 16/B374DA10, desc: COMMIT 2026-09-25 09:52:11.207893 UTC
```

### อ่านผลลัพธ์แต่ละ field

```
rmgr: Heap        len (rec/tot):     54/    54, tx:       48291, lsn: 16/B374D800, prev 16/B374D7C8, desc: INSERT off: 12, flags: 0x00, blkref #0: rel 1663/16412/16420 blk 3847
 │                    │       │        │             │                    │              │
 │                    │       │        │             │                    │              └─ block ที่ถูกกระทบ: 
 │                    │       │        │             │                    │                 tablespace/db/relfilenode blk เลขที่
 │                    │       │        │             │                    └─ LSN ของ record ก่อนหน้า (backward link)
 │                    │       │        │             └─ LSN ของ record นี้เอง
 │                    │       │        └─ Transaction ID ที่สร้าง record นี้
 │                    │       └─ ความยาว: rec (record data ล้วน) / tot (รวม backup block image ถ้ามี)
 │                    └─ resource manager ที่เขียน record นี้
 └─ resource manager name
```

field `desc` คือคำอธิบายเฉพาะของแต่ละชนิด record ซึ่งต่างกันไปตาม resource manager:

- `INSERT off: 12` = insert แถวใหม่ที่ offset (line pointer) 12 ใน page
- `INSERT_LEAF` = เพิ่ม entry ใหม่ใน B-tree leaf page (ของ index)
- `COMMIT ...timestamp...` = commit record พร้อม timestamp ที่ commit
- `FPW` (ในบรรทัดที่ 4) = record นี้มี **Full Page Write** แนบมาด้วย (สังเกตว่า `len (rec/tot)` คือ `58/8218` — ตัวเลข 8218 คือขนาดใหญ่ผิดปกติเพราะรวม full page image ทั้ง page 8KB เข้าไปด้วย ดู Step 829)

### ใช้ filter เพื่อโฟกัสเฉพาะสิ่งที่สนใจ

```bash
# ดูเฉพาะ record ของ resource manager ชนิดหนึ่ง เช่น เฉพาะ Btree
$ pg_waldump -p pg_wal --rmgr=Btree -s 16/B3740000 -e 16/B3750000

# ดูเฉพาะ record ที่เกี่ยวกับ relation (table/index) หนึ่ง ๆ
$ pg_waldump -p pg_wal --relation=1663/16412/16420 -s 16/B3740000

# ดูเฉพาะ record ของ transaction ID หนึ่ง ๆ (ตามด้วย commit/abort ของมัน)
$ pg_waldump -p pg_wal --xid=48291

# แสดงสถิติสรุป (aggregate) แทนการแสดงทีละ record — มีประโยชน์มากสำหรับวิเคราะห์
$ pg_waldump -p pg_wal -s 16/B3700000 -e 16/B3800000 --stats

Type                             N      (%)          Record size      (%)             FPI size      (%)        Combined size      (%)
----                             -      ---          -----------      ---             --------      ---        -------------      ---
Heap/INSERT                  84213 ( 41.02)              4547502 ( 27.85)             41254912 ( 62.14)             45802414 ( 55.31)
Btree/INSERT_LEAF            91820 ( 44.72)              5876480 ( 35.99)             23887872 ( 35.97)             29764352 ( 35.92)
Transaction/COMMIT            5127 (  2.50)               174318 (  1.07)                    0 (  0.00)               174318 (  0.21)
Heap/HOT_UPDATE               19004 (  9.25)              1102232 (  6.75)              1015808 (  1.53)              2118040 (  2.56)
Heap2/PRUNE                    3821 (  1.86)               167324 (  1.02)                    0 (  0.00)               167324 (  0.20)
XLOG/FPI_FOR_HINT                918 (  0.45)                49572 (  0.30)               372736 (  0.56)               422308 (  0.51)
----                             -      ---          -----------      ---             --------      ---        -------------      ---
Total                        205281                    16337221 [19.72%]         66407475 [80.28%]         82858... [100%]
```

โหมด `--stats` มีประโยชน์อย่างมากในเชิง capacity planning — ทำให้เห็นทันทีว่า resource manager ชนิดไหนหรือ operation ชนิดไหนที่สร้าง WAL มากที่สุด (ในตัวอย่างข้างต้น: Heap/INSERT และ Btree/INSERT_LEAF ครองสัดส่วนใหญ่ที่สุด และ FPI — full page image — กิน 80% ของขนาดรวม ซึ่งเป็นสัญญาณว่าอาจต้องพิจารณาเรื่อง checkpoint frequency หรือ compression)

### ข้อจำกัดของ pg_waldump

- ต้องรันในเครื่องเดียวกับที่มีไฟล์ WAL (หรือ copy ไฟล์มา — ไม่สามารถ query ผ่าน network ได้)
- ไม่สามารถ "แปลกลับ" WAL record เป็น SQL statement ต้นฉบับได้ (เพราะ WAL เก็บเป็น physical change ไม่ใช่ statement — ถ้าต้องการดู logical change ต้องใช้ logical decoding กับ `wal_level = logical` แทน ซึ่งเป็นคนละเครื่องมือ)
- อ่านได้เฉพาะไฟล์ WAL ที่ยัง valid (ไม่ corrupt เกินกว่าจะ parse header ได้)

---

## Step 829: full_page_writes — ป้องกัน Torn Page

### ปัญหา Torn Page คืออะไร

PostgreSQL เขียนและอ่านข้อมูลเป็นหน่วย **page ขนาด 8KB** (ค่า default, กำหนดด้วย `BLCKSZ` ตอน compile) แต่ระบบปฏิบัติการและฮาร์ดแวร์ดิสก์ไม่ได้รับประกันว่าการเขียน 8KB จะเป็น **atomic operation** เสมอไป โดยเฉพาะอย่างยิ่ง:

- Disk sector ทั่วไปมีขนาด 512 bytes หรือ 4KB (ไม่ใช่ 8KB)
- Filesystem อาจเขียนเป็น block เล็กกว่า page ของ PostgreSQL
- ถ้าไฟดับหรือ OS crash ขณะกำลังเขียน 8KB page อยู่ อาจเขียนสำเร็จแค่บางส่วน (เช่น 4KB แรกเขียนสำเร็จ แต่ 4KB หลังยังเป็นข้อมูลเก่าหรือข้อมูลขยะ)

```
Page ขนาด 8KB ก่อนเขียน (บน disk, เวอร์ชันเก่า):
┌────────────────────────────────────────────┐
│              OLD DATA (8KB)                 │
└────────────────────────────────────────────┘

ระหว่างกำลังเขียน page ใหม่ลงไป แล้วเกิดไฟดับกลางคัน:
┌──────────────────┬───────────────────────────┐
│   NEW DATA (4KB)  │      OLD DATA (4KB)        │  ← TORN PAGE!
│   (เขียนสำเร็จ)    │  (ยังไม่ถูกเขียนทับ)        │     ข้อมูลครึ่งเก่า
└──────────────────┴───────────────────────────┘     ครึ่งใหม่ปนกัน!
```

Torn page คือ page ที่มีเนื้อหา**ครึ่งเก่าครึ่งใหม่**ปะปนกัน — สถานะนี้ไม่ตรงกับสถานะใด ๆ ที่เคยมีอยู่จริง (ไม่ใช่ทั้งเวอร์ชันเก่าที่สมบูรณ์ ไม่ใช่เวอร์ชันใหม่ที่สมบูรณ์) และอันตรายมากเพราะ:

1. Checksum (ถ้าเปิด `data_checksums`) จะ fail ทันที เพราะเนื้อหาไม่ตรงกับ checksum ที่บันทึกไว้
2. โครงสร้างภายใน page (เช่น line pointer array, tuple header) อาจเสียหายจนอ่านไม่ได้เลย ทำให้ PostgreSQL crash ซ้ำเมื่อพยายามอ่าน page นี้

### เหตุใด WAL replay ปกติเอาไม่อยู่กับปัญหานี้

ปกติ WAL record จะบันทึกแค่ **delta** (ส่วนต่าง) เช่น "เปลี่ยน byte ที่ offset 40 จาก X เป็น Y" — วิธีนี้ประหยัดพื้นที่มาก แต่มันตั้งอยู่บนสมมติฐานว่า **page เดิมที่จะ apply delta ลงไปนั้นต้องอยู่ในสภาพสมบูรณ์ก่อน** ถ้า page เดิมเป็น torn page (เสียหายบางส่วน) การ apply delta ("เปลี่ยน byte offset 40") ลงบน page ที่เสียหายไปแล้วก็ยังได้ page ที่เสียหายอยู่ดี — delta ไม่สามารถ "ซ่อม" ส่วนที่เสียหายอื่น ๆ ของ page ได้เลย

### ทางแก้: Full Page Write (FPW)

PostgreSQL แก้ปัญหานี้ด้วยกลไก **full_page_writes** (เปิดเป็นค่า default เสมอ `full_page_writes = on`):

> **กฎ**: สำหรับ page ใด ๆ ก็ตาม การแก้ไขครั้งแรกหลังจาก checkpoint ล่าสุด จะไม่บันทึกแค่ delta อย่างเดียว แต่จะแนบสำเนาทั้ง page (เรียกว่า **Full Page Image — FPI**) เข้าไปใน WAL record ด้วย

```
Timeline ของ page X:

  CHECKPOINT              1st write             2nd write         3rd write
  (redo point)             หลัง checkpoint        หลัง 1st         หลัง 2nd
      │                        │                     │                 │
      ▼                        ▼                     ▼                 ▼
──────●────────────────────────●─────────────────────●─────────────────●──▶ LSN
                                │                     │                 │
                          WAL record ของ         WAL record          WAL record
                          1st write บันทึก        ของ 2nd write       ของ 3rd write
                          FULL PAGE IMAGE          บันทึกแค่ DELTA     บันทึกแค่ DELTA
                          (8KB เต็ม)               (เล็กมาก)           (เล็กมาก)
                          
                          ← เพราะเป็นครั้งแรกที่page ← page นี้ปลอดภัยแล้ว
                            นี้ถูกแก้หลัง checkpoint   เพราะมี FPI ที่ก่อน
                            จึงต้องมี "จุดเริ่มต้น       หน้านี้เป็นฐานที่แน่นอน
                            ที่สมบูรณ์" ไว้กันเหนียว     บน disk ที่สมบูรณ์แล้ว
```

**เหตุผลที่แนบเฉพาะ "ครั้งแรกหลัง checkpoint"**: หลังจาก checkpoint เกิดขึ้น เรารู้ว่า data page บนดิสก์ ณ ขณะนั้นถูกเขียนโดย OS/filesystem เสร็จสมบูรณ์แล้ว (checkpoint fsync ทุกอย่างเรียบร้อย) แต่การเขียนครั้ง**ถัดไป**ของ page นั้น (การ UPDATE ครั้งใหม่หลัง checkpoint) มีความเสี่ยงที่จะเกิด torn page ได้ ถ้า crash เกิดขึ้นระหว่างกำลังเขียนครั้งนั้นพอดี ดังนั้น PostgreSQL จึงต้องมี "หลักฐานฉบับเต็ม" (FPI) เก็บไว้ใน WAL เพื่อใช้เป็นจุดตั้งต้นในการ **เขียนทับทั้ง page ใหม่หมด** (ไม่ใช่ apply delta) ในกรณีที่ต้อง recovery — เพราะการเขียนทับทั้ง page ด้วยข้อมูลที่รู้แน่ชัดว่าถูกต้อง 100% นั้นปลอดภัยเสมอ ไม่ว่า page บนดิสก์ ณ ตอนนั้นจะ torn หรือไม่ก็ตาม

หลังจากมี FPI ของ page นั้นบันทึกไว้แล้ว การแก้ไขครั้งต่อ ๆ ไปในช่วง checkpoint เดียวกันสามารถบันทึกแค่ delta ได้ตามปกติ (ประหยัดพื้นที่) เพราะถ้าต้อง recovery ระบบจะ apply FPI ก่อน (ได้ page ที่สมบูรณ์แน่นอน) แล้วค่อย apply delta ที่ตามมาทีละอันบน page ที่สมบูรณ์นั้น

### ผลกระทบต่อขนาด WAL

Full page write คือสาเหตุหลักที่ทำให้ **ปริมาณ WAL พุ่งสูงขึ้นทันทีหลัง checkpoint** (จาก log `--stats` ใน Step 828 จะเห็นว่า FPI กินสัดส่วนขนาด WAL ได้มากถึง 60-80% ในบาง workload ที่มีการเขียนกระจายทั่วตารางขนาดใหญ่)

```sql
-- ทดสอบ: ปิด full_page_writes ชั่วคราว (ระวัง! ไม่แนะนำใน production 
-- เพราะเสี่ยงต่อ torn page — ใช้ทดสอบเพื่อการศึกษาเท่านั้น)
SHOW full_page_writes;
--  full_page_writes
-- -------------------
--  on

-- เปรียบเทียบขนาด WAL ก่อน/หลัง checkpoint ด้วย workload เดียวกัน
CHECKPOINT;
SELECT pg_current_wal_lsn() AS before_update \gset

UPDATE demo_lsn SET note = note || '!' WHERE id <= 1000;

SELECT pg_size_pretty(
    pg_wal_lsn_diff(pg_current_wal_lsn(), :'before_update')
) AS wal_after_checkpoint;
-- มักจะเห็นขนาดใหญ่กว่าปกติ เพราะแต่ละ page ที่ถูกแก้ครั้งแรก
-- หลัง checkpoint ต้องแนบ FPI

-- ทำ UPDATE ซ้ำ (ไม่ผ่าน checkpoint ใหม่) เทียบขนาด
SELECT pg_current_wal_lsn() AS before_update2 \gset
UPDATE demo_lsn SET note = note || '!' WHERE id <= 1000;
SELECT pg_size_pretty(
    pg_wal_lsn_diff(pg_current_wal_lsn(), :'before_update2')
) AS wal_second_round;
-- ควรเล็กกว่ารอบแรกมาก เพราะ page เดิมมี FPI แนบไว้แล้วในช่วง checkpoint นี้
```

ผลลัพธ์ตัวอย่าง:

```
 wal_after_checkpoint | wal_second_round
-----------------------+-------------------
 8760 kB               | 512 kB
```

จะเห็นว่ารอบสอง (ที่ page เดิมมี FPI แนบไว้แล้วในช่วง checkpoint เดียวกัน) ใช้พื้นที่ WAL น้อยกว่ารอบแรกอย่างชัดเจน (~17 เท่า) เพราะไม่ต้องแนบ full page image ซ้ำ

### เมื่อไหร่ที่ปิด full_page_writes ได้บ้าง

โดยทั่วไป **ไม่แนะนำให้ปิด** ยกเว้นในกรณีที่มั่นใจว่า filesystem/storage รับประกัน atomic page write อยู่แล้ว เช่น:

- ใช้ filesystem แบบ copy-on-write ที่รับประกัน atomic write เช่น ZFS (เมื่อ configure ให้ตรงกับ page size) — แต่ต้องพิจารณาอย่างรอบคอบเป็นราย ๆ ไป
- ใช้ storage ที่มี battery-backed write cache และรับประกัน atomic sector write ในระดับที่ครอบคลุม 8KB เต็ม

ในทางปฏิบัติ ผู้เชี่ยวชาญส่วนใหญ่แนะนำให้ **เปิดไว้เสมอ** (ค่า default) เพราะความเสี่ยงของ data corruption สูงกว่าประโยชน์ด้าน performance/พื้นที่ดิสก์ที่ประหยัดได้มาก ทางเลือกที่ปลอดภัยกว่าในการลดผลกระทบของ FPW คือ:

- เพิ่ม `checkpoint_timeout` และ `max_wal_size` ให้ checkpoint เกิดห่างขึ้น (ลดความถี่ที่ page ต้องแนบ FPI ใหม่)
- ใช้ WAL compression: `wal_compression = on` (PostgreSQL 15+ รองรับ `lz4` และ `zstd` นอกเหนือจาก `pglz` แบบเดิม) ซึ่งบีบอัดเฉพาะส่วน FPI ทำให้ขนาด WAL ลดลงอย่างมีนัยสำคัญโดยไม่กระทบความปลอดภัย

```sql
-- ตรวจสอบและตั้งค่า wal_compression
SHOW wal_compression;
ALTER SYSTEM SET wal_compression = 'zstd';
-- ต้อง reload (ไม่ต้อง restart)
SELECT pg_reload_conf();
```

---

## Step 830: แบบฝึกหัดรวม — วิเคราะห์กิจกรรม WAL จริงในเครื่อง

### เป้าหมายของแบบฝึกหัดนี้

เราจะรวบรวมความรู้จาก Step 821-829 มาใช้งานจริง โดยจะ:

1. วัดปริมาณ WAL ที่เกิดขึ้นจาก workload หนึ่ง ๆ ด้วย `pg_lsn` functions
2. ใช้ `pg_waldump` ตรวจสอบเนื้อหา WAL ที่เกิดขึ้นจริง
3. คำนวณ **WAL generation rate** (อัตราการสร้าง WAL ต่อวินาที) ซึ่งเป็นตัวเลขสำคัญมากสำหรับ capacity planning (เช่น การกำหนดขนาด network bandwidth ที่ standby ต้องมี, การประมาณพื้นที่ดิสก์ที่ archive ต้องรองรับ)

### ขั้นตอนที่ 1: เตรียมตารางทดสอบ

```sql
CREATE TABLE wal_workload_test (
    id bigserial PRIMARY KEY,
    payload text,
    created_at timestamptz DEFAULT now()
);

CREATE INDEX idx_wal_workload_created ON wal_workload_test (created_at);
```

### ขั้นตอนที่ 2: จับเวลาและ LSN ก่อนเริ่ม workload

```sql
SELECT clock_timestamp() AS t_start, pg_current_wal_lsn() AS lsn_start
\gset

-- ตรวจสอบค่าที่บันทึกไว้
SELECT :'t_start' AS started_at, :'lsn_start' AS start_lsn;
--          started_at          | start_lsn
-- -------------------------------+-------------
--  2026-09-25 10:05:00.128331+00 | 16/C0001000
```

### ขั้นตอนที่ 3: รัน workload จำลอง (insert จำนวนมาก)

```sql
INSERT INTO wal_workload_test (payload)
SELECT repeat(md5(g::text), 4)  -- payload ~128 bytes ต่อแถว
FROM generate_series(1, 500000) g;
```

### ขั้นตอนที่ 4: จับเวลาและ LSN หลังจบ workload

```sql
SELECT clock_timestamp() AS t_end, pg_current_wal_lsn() AS lsn_end
\gset

SELECT :'t_end' AS finished_at, :'lsn_end' AS end_lsn;
--          finished_at         |  end_lsn
-- -------------------------------+-------------
--  2026-09-25 10:05:14.902558+00 | 16/C512A448
```

### ขั้นตอนที่ 5: คำนวณ WAL Generation Rate

```sql
SELECT
    :'t_start'::timestamptz                                       AS started_at,
    :'t_end'::timestamptz                                         AS finished_at,
    EXTRACT(EPOCH FROM (:'t_end'::timestamptz - :'t_start'::timestamptz)) AS duration_seconds,
    pg_wal_lsn_diff(:'lsn_end'::pg_lsn, :'lsn_start'::pg_lsn)      AS wal_bytes,
    pg_size_pretty(pg_wal_lsn_diff(:'lsn_end'::pg_lsn, :'lsn_start'::pg_lsn)) AS wal_pretty,
    pg_size_pretty(
        (pg_wal_lsn_diff(:'lsn_end'::pg_lsn, :'lsn_start'::pg_lsn)
         / EXTRACT(EPOCH FROM (:'t_end'::timestamptz - :'t_start'::timestamptz)))::numeric
    ) || '/s'                                                       AS wal_generation_rate;
```

ผลลัพธ์ตัวอย่าง:

```
          started_at           |          finished_at          | duration_seconds | wal_bytes  | wal_pretty | wal_generation_rate
--------------------------------+--------------------------------+-------------------+------------+------------+-----------------------
 2026-09-25 10:05:00.128331+00 | 2026-09-25 10:05:14.902558+00 |         14.774227 |   82285128 | 78 MB      | 5581 kB/s
```

**การตีความ**: workload นี้สร้าง WAL ในอัตราประมาณ **5.5 MB/s** — นี่คือตัวเลขที่นำไปใช้คำนวณต่อได้ทันที เช่น:

- ถ้ามี standby เชื่อมผ่านเครือข่ายที่มี bandwidth 10 Mbps (~1.25 MB/s) → standby **ตามไม่ทัน** primary แน่นอน จะเกิด lag สะสมเรื่อย ๆ
- ถ้า `max_wal_size = 1GB` และอัตรานี้คงที่ → checkpoint จะถูกกระตุ้นทุก ๆ ประมาณ `1024 MB / 5.5 MB/s ≈ 186 วินาที` (ประมาณ 3 นาที) ซึ่งถี่กว่า `checkpoint_timeout` default (5 นาที) แปลว่าตัวกระตุ้น checkpoint หลักคือขนาด WAL ไม่ใช่เวลา

### ขั้นตอนที่ 6: ใช้ pg_waldump ตรวจสอบ WAL segment ที่เกี่ยวข้อง

```bash
# หาว่า LSN ช่วงนี้ ครอบคลุม WAL segment ไฟล์ไหนบ้าง
$ psql -c "SELECT pg_walfile_name('16/C0001000')"
      pg_walfile_name
-----------------------------
 0000000100000016000000C0

$ psql -c "SELECT pg_walfile_name('16/C512A448')"
      pg_walfile_name
-----------------------------
 0000000100000016000000C5

# แปลว่า workload นี้ครอบคลุมไฟล์ตั้งแต่ ...C0 ถึง ...C5 (5 ไฟล์ x 16MB = 80MB
# สอดคล้องกับตัวเลข 78MB ที่คำนวณได้จาก pg_wal_lsn_diff)

# วิเคราะห์เนื้อหาด้วย --stats
$ pg_waldump -p /var/lib/postgresql/16/main/pg_wal \
    -s 16/C0001000 -e 16/C512A448 --stats

Type                       N       (%)          Record size      (%)             FPI size      (%)        Combined size      (%)
----                       -       ---          -----------      ---             --------      ---        -------------      ---
Heap/INSERT            500000  ( 89.24)             27500000  ( 47.84)              6832128  ( 33.12)             34332128  ( 44.14)
Btree/INSERT_LEAF       60214  ( 10.76)              3853696  (  6.70)             13459456  ( 65.24)             17313152  ( 22.26)
XLOG/FPI_FOR_HINT           0  (  0.00)                    0  (  0.00)                    0  (  0.00)                    0  (  0.00)
Transaction/COMMIT          1  (  0.00)                   34  (  0.00)                    0  (  0.00)                   34  (  0.00)
----                       -       ---          -----------      ---             --------      ---        -------------      ---
Total                  560215                      57506306 [73.92%]         20291584 [26.08%]           77797890 [100%]
```

**การตีความผลลัพธ์**:

- 89% ของ record ทั้งหมดคือ `Heap/INSERT` (สอดคล้องกับ workload ที่ insert 500,000 แถว) แต่กิน byte แค่ 44% ของ WAL ทั้งหมด
- `Btree/INSERT_LEAF` มีจำนวน record แค่ 10.76% แต่กิน byte ถึง 22.26% เพราะ FPI ของ B-tree page (ที่ถูกแก้ไขบ่อยเนื่องจาก insert เรียงตาม timestamp ทำให้กระจุกตัวอยู่ที่ rightmost leaf page ไม่กี่หน้า) มีสัดส่วนสูงถึง 65.24% ของขนาด FPI ทั้งหมด — นี่คือสัญญาณคลาสสิกของ **index bloat จาก sequential insert pattern** ที่ DBA ควรจับตา
- transaction เดียว (COMMIT 1 ครั้ง) เพราะทำ insert ทั้งหมดในทรานแซกชันเดียว (ถ้าแยกเป็นหลาย transaction จะมี COMMIT record หลายตัว ซึ่งกิน WAL เพิ่มขึ้นตามจำนวน transaction)

### ขั้นตอนที่ 7: เปรียบเทียบกับ WAL ที่เกิดจาก autovacuum (background activity)

```bash
# ดูว่าในช่วงเวลาเดียวกัน มี WAL จาก vacuum/checkpoint แทรกอยู่ด้วยหรือไม่
$ pg_waldump -p /var/lib/postgresql/16/main/pg_wal \
    -s 16/C0001000 -e 16/C512A448 --rmgr=Heap2

rmgr: Heap2       len (rec/tot):     64/    64, tx:          0, lsn: 16/C3204A18, prev 16/C3204908, desc: PRUNE snapshotConflictHorizon: 0, isCatalogRel: F, nredirected: 3, ndead: 12, nunused: 0, redirected: [...], dead: [...], unused: []
```

`Heap2/PRUNE` ที่มี `tx: 0` (ไม่มี transaction ID ผูกอยู่) บ่งบอกว่านี่คือ WAL record ที่เกิดจาก **HOT pruning** ที่ทำงานอัตโนมัติระหว่าง query ปกติ (opportunistic pruning) ไม่ใช่จาก autovacuum โดยตรง เป็นตัวอย่างที่ดีว่า WAL ไม่ได้เกิดจาก DML ของผู้ใช้เท่านั้น แต่เกิดจากกิจกรรมภายในของ PostgreSQL เองด้วย

---

## สรุปท้ายบท

ในบทนี้เราได้เจาะลึกกลไก **Write-Ahead Logging (WAL)** ซึ่งเป็นหัวใจสำคัญที่สุดอย่างหนึ่งของสถาปัตยกรรม PostgreSQL:

- **Write-ahead principle** (Step 821): ทุกการเปลี่ยนแปลงต้องถูกเขียนลง WAL และ fsync ก่อนที่จะรับประกัน commit เสมอ นี่คือรากฐานของ **Durability** ใน ACID และช่วยแปลง random I/O ให้เป็น sequential I/O ที่เร็วกว่ามาก
- **WAL Record และ LSN** (Step 822): ทุกการเปลี่ยนแปลงถูกห่อเป็น record ที่มี resource manager กำกับ และมี LSN เป็น "เลขที่บ้าน" อ้างอิงตำแหน่งที่แน่นอนในสตรีม WAL ทั้งหมด — LSN คือแกนเวลาที่ทุก subsystem (page, checkpoint, replication) ใช้อ้างอิงร่วมกัน
- **WAL Segment File** (Step 823): WAL ถูกเก็บเป็นไฟล์ 16MB ใน `pg_wal/` ชื่อไฟล์ประกอบด้วย timeline ID, log ID และ segment ID ซึ่งคำนวณได้จาก LSN โดยตรง
- **WAL Levels** (Step 824): `minimal` → `replica` → `logical` แต่ละระดับบันทึกข้อมูลเพิ่มขึ้นเพื่อรองรับความสามารถที่มากขึ้นตามลำดับ (crash recovery → replication/archiving → logical decoding)
- **Checkpoint** (Step 825): จุดที่รับประกันว่า dirty pages ทั้งหมดจนถึง redo point ถูกเขียนลงดิสก์แล้ว เป็นตัวกำหนดว่า crash recovery จะต้อง replay WAL ย้อนไปไกลแค่ไหน
- **Crash Recovery** (Step 826): เมื่อ restart หลัง crash PostgreSQL จะ replay WAL จาก redo point ล่าสุด โดยอาศัยกลไก `pd_lsn` ทำให้การ replay เป็น idempotent เสมอ ปลอดภัยแม้จะล่มซ้ำระหว่าง recovery
- **pg_lsn และฟังก์ชัน** (Step 827): data type และฟังก์ชันชุดนี้เปิดให้วัดปริมาณ WAL และ replication lag ได้ในระดับ byte-precision ซึ่งแม่นยำกว่าการวัดด้วยเวลาเพียงอย่างเดียว
- **pg_waldump** (Step 828): เครื่องมือสำหรับ inspect เนื้อหา WAL ระดับ record จริง มีโหมด `--stats` ที่มีประโยชน์มากสำหรับวิเคราะห์ว่า operation ชนิดใดสร้าง WAL มากที่สุด
- **full_page_writes** (Step 829): กลไกป้องกัน torn page โดยแนบ full page image ในการแก้ไข page ครั้งแรกหลัง checkpoint แลกกับขนาด WAL ที่เพิ่มขึ้น — เป็น trade-off ที่แทบไม่ควรปิดใน production
- **แบบฝึกหัดรวม** (Step 830): การผสาน `pg_lsn` functions กับ `pg_waldump` เพื่อคำนวณ WAL generation rate จริง ซึ่งเป็นตัวเลขสำคัญสำหรับวางแผน capacity ของเครือข่าย replication และพื้นที่จัดเก็บ archive

ความเข้าใจ WAL ในระดับนี้คือพื้นฐานสำคัญสำหรับหัวข้อที่จะตามมา โดยเฉพาะ **MVCC internals** ซึ่งพึ่งพา WAL ในการบันทึกการเปลี่ยนแปลงของ tuple version, และหัวข้อ replication ขั้นสูงที่ทั้งหมดตั้งอยู่บนกลไก LSN และ WAL streaming ที่เราเพิ่งศึกษาไป

---

## แบบฝึกหัด

<details>
<summary><b>แบบฝึกหัดที่ 1:</b> ทำไม PostgreSQL ต้องเขียน WAL ก่อนเขียน data page จริง ทั้งที่ดูเหมือนจะทำให้ต้องเขียนข้อมูลซ้ำสองรอบ (เขียนทั้ง WAL และเขียนทั้ง data page)? อธิบายอย่างน้อย 2 เหตุผล</summary>

**เฉลย:**

1. **Durability**: WAL คือหลักฐานที่ fsync ลงดิสก์ก่อนตอบ client ว่า commit สำเร็จ ทำให้แม้ data page จริงจะยังไม่ถูกเขียนลงดิสก์ (อยู่แค่ใน shared_buffers) ระบบก็ยังสามารถ replay WAL เพื่อสร้างสถานะล่าสุดขึ้นมาใหม่ได้หากเกิด crash
2. **Performance (I/O pattern)**: การเขียน WAL เป็นการเขียนแบบ sequential (ต่อท้ายไฟล์เรื่อย ๆ) ซึ่งเร็วกว่าการเขียน data page ที่กระจัดกระจายอยู่คนละตำแหน่งบนดิสก์ (random I/O) มาก — การเขียนซ้ำสองรอบแต่ครั้งแรกเป็น sequential เร็ว ยังคุ้มกว่าการต้อง fsync data page แบบ random ทุกครั้งที่ commit
3. **Deferred write / batching**: เพราะ WAL รับประกัน durability ไว้แล้ว data page จริงจึงไม่ต้องเขียนทันที สามารถรอรวมกันเป็น batch ตอน checkpoint ได้ ลดจำนวนครั้งของการเขียน page เดิมซ้ำ ๆ (ถ้า page ถูกแก้หลายครั้งก่อนถึง checkpoint ก็เขียนลงดิสก์แค่ครั้งเดียว)

</details>

<details>
<summary><b>แบบฝึกหัดที่ 2:</b> LSN คือ `3A/8F2C1000` จงอธิบายว่าตัวเลขนี้ประกอบด้วยส่วนใดบ้าง และแปลงเป็นค่า 64-bit เดียวได้อย่างไร</summary>

**เฉลย:**

LSN แสดงผลเป็น `<high 32-bit hex>/<low 32-bit hex>`:
- high = `0x3A`
- low = `0x8F2C1000`

แปลงเป็นตัวเลข 64-bit เดียว:
```
LSN = (0x3A << 32) | 0x8F2C1000
    = (58 × 4294967296) + 2401906688
    = 249108183168 + 2401906688
    = 251510089856
```

LSN นี้ไม่ใช่แค่ "หมายเลขอ้างอิง" ลอย ๆ แต่คือ byte offset ที่แน่นอนภายในสตรีม WAL ทั้งหมดตั้งแต่ initdb (นับรวมทุก timeline switch ตามกฎของแต่ละ timeline)

</details>

<details>
<summary><b>แบบฝึกหัดที่ 3:</b> ทำไมชื่อไฟล์ WAL segment ถึงยาว 24 ตัวอักษร hexadecimal และแบ่งเป็น 3 ส่วน ส่วนละ 8 ตัวอักษร? แต่ละส่วนคืออะไร</summary>

**เฉลย:**

ชื่อไฟล์ประกอบด้วย 3 ส่วน ส่วนละ 8 hex digit (32 bit):

1. **Timeline ID (TLI)**: ระบุว่าไฟล์นี้อยู่ใน timeline ไหน (เพิ่มขึ้นทุกครั้งที่มี failover/PITR ไปยังจุดในอดีต)
2. **Log ID (32-bit บนของ LSN)**: ส่วนหน้าของ LSN
3. **Segment ID**: หมายเลข segment ภายใน log ID นั้น คำนวณจาก 32-bit ล่างของ LSN หารด้วยขนาด segment (default 16MB)

การแบ่งเป็น 3 ส่วนแยกกันทำให้สามารถคำนวณชื่อไฟล์จาก LSN ได้ตรงไปตรงมา (ผ่าน `pg_walfile_name()`) และทำให้แต่ละไฟล์มีขนาดคงที่ 16MB จัดการง่าย รวมถึงรองรับการ recycle ไฟล์เก่ากลับมาใช้ใหม่ได้

</details>

<details>
<summary><b>แบบฝึกหัดที่ 4:</b> ระบบฐานข้อมูลตั้งค่า `wal_level = replica` อยู่ ต้องการเปิดใช้ logical replication (`CREATE PUBLICATION`) จะทำอย่างไร และมีผลกระทบอะไรบ้าง?</summary>

**เฉลย:**

ต้องเปลี่ยน `wal_level` เป็น `logical`:

```sql
ALTER SYSTEM SET wal_level = logical;
```

แล้ว **restart** PostgreSQL service (เพราะ `wal_level` เป็น parameter context `postmaster` — เปลี่ยนได้เฉพาะตอน restart ไม่ใช่แค่ reload)

ผลกระทบ:
- ปริมาณ WAL ที่เกิดขึ้นจะเพิ่มขึ้นเล็กน้อยจากเดิม (บันทึก catalog metadata และข้อมูล replica identity เพิ่มเติม)
- ตารางที่ต้องการ logical replicate แต่ไม่มี primary key ต้องตั้ง `REPLICA IDENTITY FULL` หรือ `REPLICA IDENTITY USING INDEX` ซึ่งจะทำให้ WAL ของ UPDATE/DELETE โตขึ้นอย่างมีนัยสำคัญ (ต้องบันทึกค่าเดิมทุกคอลัมน์)
- เปิดความสามารถให้ใช้ `CREATE PUBLICATION`/`CREATE SUBSCRIPTION` หรือเครื่องมือ CDC ภายนอก เช่น Debezium ได้

</details>

<details>
<summary><b>แบบฝึกหัดที่ 5:</b> คำสั่ง `pg_stat_bgwriter` (หรือ `pg_stat_checkpointer` ใน PostgreSQL 17+) แสดงค่า `checkpoints_timed = 850` และ `checkpoints_req = 620` จงตีความและแนะนำการปรับจูน</summary>

**เฉลย:**

`checkpoints_req` (requested เพราะ WAL เต็ม `max_wal_size`) สูงถึง 620 เทียบกับ `checkpoints_timed` (ตามเวลาปกติ) ที่ 850 — สัดส่วนนี้สูงผิดปกติ (ปกติอยากให้ `checkpoints_req` เป็นสัดส่วนเล็กน้อยเทียบกับ `checkpoints_timed`) แปลว่า checkpoint ถูก "บังคับ" ให้เกิดถี่เพราะ WAL โตเร็วกว่าที่ `max_wal_size` รองรับ ไม่ใช่เกิดตามตารางเวลาปกติ

การปรับจูนที่แนะนำ:
- เพิ่มค่า `max_wal_size` ให้ใหญ่ขึ้น (เช่น จาก 1GB เป็น 4GB หรือมากกว่า ขึ้นกับ workload) เพื่อให้ checkpoint เกิดตามเวลา (`checkpoint_timeout`) เป็นหลัก ไม่ใช่เกิดเพราะ WAL เต็ม
- พิจารณาเพิ่ม `checkpoint_timeout` ควบคู่กันถ้าต้องการลด I/O overhead จาก checkpoint ที่ถี่เกินไป
- ข้อแลกเปลี่ยน: `max_wal_size` ที่ใหญ่ขึ้นหมายถึงพื้นที่ดิสก์ที่ WAL อาจใช้มากขึ้น และเวลา crash recovery ที่นานขึ้น (เพราะ WAL ระหว่าง checkpoint มีมากขึ้น)

</details>

<details>
<summary><b>แบบฝึกหัดที่ 6:</b> อธิบายว่าทำไม WAL replay ระหว่าง crash recovery จึงต้องเป็น idempotent operation และกลไกใดใน PostgreSQL ที่ทำให้เกิด idempotency นี้</summary>

**เฉลย:**

WAL replay ต้อง idempotent เพราะสถานการณ์ที่เป็นไปได้คือ crash เกิดขึ้นซ้ำระหว่างกำลัง recovery อยู่ (เช่น ไฟดับซ้ำ) ถ้า replay ไม่ idempotent การ apply record เดิมซ้ำสองครั้งอาจทำให้ข้อมูลผิดเพี้ยน (เช่น เพิ่มค่า balance ซ้ำสองครั้งโดยไม่ตั้งใจ)

กลไกที่ทำให้เกิด idempotency คือ field **`pd_lsn`** ใน page header ของทุก data page ซึ่งบันทึก LSN ของ WAL record ล่าสุดที่เคยแก้ไข page นั้น ระหว่าง replay แต่ละ record ระบบจะเปิด page ที่ record นั้นอ้างถึง แล้วเช็คว่า `pd_lsn` ของ page มากกว่าหรือเท่ากับ LSN ของ record หรือไม่ — ถ้าใช่ แปลว่า page นี้ทันสมัยกว่าหรือเท่ากับ record นี้แล้ว (เคย apply ไปแล้ว) จึงข้ามไม่ apply ซ้ำ ถ้าไม่ใช่จึง apply แล้วอัปเดต `pd_lsn` ตามค่า LSN ของ record นั้น

</details>

<details>
<summary><b>แบบฝึกหัดที่ 7:</b> บน primary มี `pg_current_wal_lsn()` คืนค่า `2A/50000000` และ standby ตัวหนึ่งมี `sent_lsn = 2A/48000000`, `replay_lsn = 2A/40000000` ใน `pg_stat_replication` จงคำนวณ (ก) send lag เป็น byte (ข) replay lag รวมทั้งหมดเป็น byte