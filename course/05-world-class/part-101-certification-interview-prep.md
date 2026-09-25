# Part 101: เตรียมสอบ PostgreSQL Certification และ System Design Interview

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับโลก/ผู้เชี่ยวชาญ | Part 101

---

## เป้าหมายการเรียนรู้

เมื่อจบบทนี้ ผู้เรียนจะสามารถ:

- รู้จัก certification ของ PostgreSQL ที่มีอยู่จริงในตลาด (EDB, JPUG, cloud vendor certs) และเลือกสอบให้เหมาะกับสายอาชีพของตนเอง
- ทบทวนหัวข้อสำคัญทั้งหมดของหลักสูตรนี้ในรูปแบบที่ map กลับไปยังแต่ละ Part เพื่อวางแผนอ่านทวนก่อนสอบ
- ฝึกทำข้อสอบสไตล์ certification exam จำนวน 15-20 ข้อ พร้อมเฉลยและคำอธิบาย
- ตอบคำถาม system design interview ที่เกี่ยวกับฐานข้อมูลได้อย่างเป็นระบบ ทั้งการออกแบบ schema, indexing strategy และ scaling strategy
- ตอบคำถาม behavioral และ technical deep-dive ที่มักถูกถามในการสัมภาษณ์ตำแหน่ง DBA / Backend Engineer / Database Engineer ได้อย่างมั่นใจ
- ใช้ checklist ทบทวนความรู้ทั้งหมดของหลักสูตร (Part 001-100) เป็นแผนที่เตรียมตัวก่อนสอบสัมภาษณ์หรือสอบ certification จริง

บทนี้เป็นบทสั้นแบบ "study guide" เน้นการทบทวนและฝึกฝนภาคปฏิบัติ มากกว่าการสอนเนื้อหาใหม่ เพราะเนื้อหาทางเทคนิคทั้งหมดได้ถูกสอนไปแล้วใน Part 001-100 บทนี้จะช่วยคุณ "แปลงความรู้" ที่มีให้กลายเป็นคะแนนสอบและ offer งาน

---

## Step 991: PostgreSQL Certification ที่มีอยู่

### 991.1 ทำไมต้องสอบ Certification

Certification ไม่ใช่ตัวชี้วัดความสามารถที่สมบูรณ์แบบ แต่มีประโยชน์จริงหลายด้าน:

1. **ผ่าน HR screening** — บริษัทขนาดใหญ่หลายแห่ง (โดยเฉพาะสาย enterprise, government, finance) ใช้ certification เป็นเกณฑ์กรองใบสมัครเบื้องต้น
2. **บังคับให้ทบทวนความรู้อย่างเป็นระบบ** — การเตรียมสอบทำให้คุณกลับไปอ่านหัวข้อที่อาจใช้ไม่บ่อยในงานประจำ เช่น isolation level, WAL internals
3. **เพิ่มความน่าเชื่อถือใน resume** โดยเฉพาะสำหรับ freelance/consultant ที่ต้องพิสูจน์ตัวเองกับลูกค้าใหม่
4. **บาง cert (โดยเฉพาะ cloud vendor)** ผูกกับ career path ที่ชัดเจนในองค์กรที่ใช้ cloud นั้นเป็นหลัก

### 991.2 EDB PostgreSQL Certifications

**EnterpriseDB (EDB)** เป็นบริษัทที่ก่อตั้งโดยผู้ร่วมพัฒนา PostgreSQL core จำนวนมาก และเป็นผู้ออก certification สาย PostgreSQL ที่ได้รับการยอมรับกว้างขวางที่สุดในอุตสาหกรรม โดยทั่วไปแบ่งเป็นสองระดับหลัก (ชื่อเรียกและเวอร์ชันอาจเปลี่ยนแปลงตามรอบการอัปเดตของ EDB จึงควรตรวจสอบหน้า certification ล่าสุดของ EDB ก่อนสมัครสอบทุกครั้ง):

| ระดับ | ชื่อโดยทั่วไป | กลุ่มเป้าหมาย | รูปแบบข้อสอบ |
|---|---|---|---|
| Associate | EDB Certified Associate — PostgreSQL Administration | DBA มือใหม่-กลาง, ผู้ดูแลระบบที่เพิ่งเริ่มทำงานกับ PostgreSQL | ปรนัย (multiple choice), ผ่าน proctored online exam |
| Professional | EDB Certified Professional — PostgreSQL Administration | Senior DBA, ผู้มีประสบการณ์ production จริง | Performance-based (ลงมือทำจริงบนเครื่อง Linux/PostgreSQL) |
| Developer track | EDB Certified Associate — PostgreSQL Application Developer | Backend Developer ที่เขียน SQL/PL-pgSQL เป็นหลัก | ปรนัย + practical query writing |

**ลักษณะเด่นของสอบ EDB:**

- เน้น hands-on จริงในระดับ Professional เช่น ให้ต่อ SSH เข้าเครื่อง แล้วแก้ปัญหา เช่น ตั้งค่า replication, กู้คืนข้อมูลด้วย PITR, วิเคราะห์ query ที่ช้า
- ระดับ Associate เน้นความเข้าใจพื้นฐาน: installation, backup/restore, roles/privileges, basic tuning
- คำถามอ้างอิงจาก PostgreSQL เวอร์ชันที่ระบุไว้ชัดเจน (เช่น PostgreSQL 15/16) ควรอ่าน release notes ของเวอร์ชันนั้นประกอบ

### 991.3 Certification อื่น ๆ ที่เกี่ยวข้อง

| ชื่อ | ผู้ออก | จุดเด่น | เหมาะกับ |
|---|---|---|---|
| PostgreSQL CE (Certified Engineer) | JPUG (Japan PostgreSQL Users Group) | เป็น community-driven cert ที่เก่าแก่ที่สุดสายหนึ่ง เนื้อหาลึกด้าน internals | ผู้ต้องการทำงานกับทีมญี่ปุ่น/เอเชีย |
| AWS Certified Database – Specialty | AWS | ครอบคลุม RDS/Aurora PostgreSQL, migration, DMS, backup strategy บน AWS | Backend/DevOps ที่ใช้ AWS เป็นหลัก (เชื่อมกับ Part 094) |
| Google Cloud Professional Data Engineer | Google Cloud | ครอบคลุม Cloud SQL/AlloyDB บางส่วน ร่วมกับ data pipeline | ผู้ทำงานสาย data บน GCP (เชื่อมกับ Part 095) |
| Microsoft Certified: Azure Database Administrator Associate | Microsoft | ครอบคลุม Azure Database for PostgreSQL | ผู้ทำงานบน Azure (เชื่อมกับ Part 096) |
| Vendor-neutral: Linux Foundation / CNCF certs | Linux Foundation | ไม่เฉพาะ PostgreSQL แต่จำเป็นสำหรับ DBA ที่ดูแล PostgreSQL บน Kubernetes | ผู้ดูแล PostgreSQL บน K8s (เชื่อมกับ Part 093) |

**คำแนะนำในการเลือกสอบ:**

- ถ้าเป้าหมายคือสาย **on-prem/traditional DBA** → เริ่มที่ EDB Associate แล้วไต่ไปที่ Professional
- ถ้าทำงานในองค์กรที่ใช้ **cloud provider ใดเป็นหลักอยู่แล้ว** → สอบ cert ของ cloud นั้นควบคู่ไปกับความรู้ PostgreSQL core เพราะข้อสอบ cloud cert จะไม่ลงลึก internals เท่า EDB
- ถ้าเป็น **Backend Developer** ที่ไม่ได้ต้องดูแล infra → เน้น EDB Developer track และให้ความสำคัญกับ portfolio/system design มากกว่า cert ก็เพียงพอ

### 991.4 แนวข้อสอบทั่วไป (Exam Blueprint)

จากประสบการณ์ของผู้สอบจริงและเอกสาร exam guide ของ EDB โดยทั่วไปข้อสอบระดับ Associate จะแบ่งสัดส่วนหัวข้อประมาณนี้ (สัดส่วนเป็นค่าประมาณ ใช้สำหรับวางแผนอ่านทวน ไม่ใช่ตัวเลขทางการ):

| หมวดหัวข้อ | สัดส่วนโดยประมาณ | อ้างอิง Part ในหลักสูตรนี้ |
|---|---|---|
| Installation, configuration, cluster management | 15% | Part 002, 004, 005 |
| Data types, DDL, table design | 10% | Part 006-008, 016-019 |
| DML, query language, joins, subqueries | 15% | Part 009-014, 021-026 |
| Indexing และ query performance | 15% | Part 041-045, 073-074 |
| Transactions, concurrency, MVCC | 10% | Part 037-038, 058-060, 084 |
| Backup, restore, PITR | 10% | Part 061-062 |
| Replication และ High Availability | 10% | Part 063-065 |
| Security: roles, privileges, RLS, SSL | 10% | Part 068-070 |
| Monitoring และ maintenance (VACUUM, logging) | 5% | Part 060, 071-072 |

ระดับ Professional จะเพิ่มน้ำหนักให้กับ **troubleshooting จริง** (deadlock, replication lag, disk full, corrupt WAL) และ **performance tuning ขั้นสูง** (Part 044-045, 073-075, 081-086)

### 991.5 หัวข้อที่ควรทบทวนก่อนสอบ — Quick Review Map

ก่อนเข้าห้องสอบ ควรอ่านทวนหัวข้อเหล่านี้เป็นพิเศษ เพราะเป็นจุดที่ผู้สอบมักตอบผิด:

1. **ความแตกต่างของ `DELETE` vs `TRUNCATE` vs `DROP TABLE`** ต่อ transaction log, trigger, sequence (Part 014, 019)
2. **NULL semantics** — `NULL = NULL` คืนค่าอะไร, `IS NULL` vs `= NULL`, พฤติกรรมใน `COUNT()`, `UNIQUE constraint` (Part 015, 016)
3. **Isolation levels** — Read Committed (default), Repeatable Read, Serializable ต่างกันอย่างไรในแง่ phenomena ที่ป้องกันได้ (dirty read, non-repeatable read, phantom read, serialization anomaly) (Part 038)
4. **Index types** — B-tree, Hash, GIN, GiST, SP-GiST, BRIN ใช้กับข้อมูลแบบไหน (Part 041-042)
5. **`EXPLAIN` vs `EXPLAIN ANALYZE`** — อ่านค่า cost, actual time, rows, loops, Buffers อย่างไร (Part 044)
6. **MVCC และ VACUUM** — tuple version, xmin/xmax, transaction ID wraparound, ทำไม autovacuum สำคัญ (Part 058, 060, 084)
7. **Streaming replication vs Logical replication** — ความแตกต่างของกลไก, use case, ข้อจำกัด (Part 063-064)
8. **PITR (Point-in-Time Recovery)** — ลำดับขั้นตอน base backup + WAL archiving + recovery target (Part 061-062)
9. **Roles และ privilege inheritance** — `GRANT`, `REVOKE`, `ROLE` vs `USER`, `NOLOGIN`, membership (Part 068)
10. **Connection pooling** — PgBouncer session/transaction/statement mode ต่างกันอย่างไร ใช้กับ prepared statement ได้ไหม (Part 066)

### 991.6 ข้อสอบฝึกฝนสไตล์ Certification (15 ข้อ)

ลองทำโดยไม่เปิดเฉลยก่อน แล้วค่อยตรวจคำตอบ

**ข้อ 1.** ในระดับ isolation `READ COMMITTED` (ค่า default ของ PostgreSQL) ปรากฏการณ์ใดต่อไปนี้ **สามารถ** เกิดขึ้นได้?

A. Dirty Read
B. Non-repeatable Read
C. Serialization Anomaly เท่านั้น
D. ไม่มีปรากฏการณ์ใดเกิดขึ้นเลย

<details>
<summary>เฉลยข้อ 1</summary>

**คำตอบ: B**

`READ COMMITTED` ป้องกัน Dirty Read ได้ (transaction จะไม่เห็นข้อมูลที่ยัง uncommitted) แต่ **ไม่ป้องกัน** Non-repeatable Read และ Phantom Read เพราะแต่ละ statement จะเห็น snapshot ใหม่ทุกครั้งที่ query เริ่มต้น ถ้าต้องการป้องกัน non-repeatable read ต้องใช้ `REPEATABLE READ` ขึ้นไป

</details>

---

**ข้อ 2.** คำสั่งใดต่อไปนี้จะ **ไม่** trigger `ON DELETE` trigger ของตาราง?

A. `DELETE FROM orders WHERE id = 1;`
B. `TRUNCATE TABLE orders;`
C. `DELETE FROM orders;`
D. ถูกทั้ง A และ C

<details>
<summary>เฉลยข้อ 2</summary>

**คำตอบ: B**

`TRUNCATE` ทำงานโดยการ deallocate หน้า (page) ของตารางทันที ไม่ได้ลบทีละแถว จึงไม่ fire row-level trigger (แต่ statement-level trigger สำหรับ `TRUNCATE` โดยเฉพาะสามารถสร้างได้ตั้งแต่ PostgreSQL 11 เป็นต้นไป)

</details>

---

**ข้อ 3.** Index type ใดเหมาะที่สุดสำหรับการค้นหาด้วย operator `@>` บนคอลัมน์ชนิด `jsonb`?

A. B-tree
B. Hash
C. GIN
D. BRIN

<details>
<summary>เฉลยข้อ 3</summary>

**คำตอบ: C**

GIN (Generalized Inverted Index) ถูกออกแบบมาสำหรับข้อมูลชนิด composite เช่น array, jsonb, tsvector ที่ต้องการค้นหาด้วย containment operator (`@>`, `?`, `?&`) B-tree ใช้กับ equality/range บนค่าสเกลาร์เท่านั้น

</details>

---

**ข้อ 4.** ใน `EXPLAIN ANALYZE` ค่า `rows` ใน node หนึ่ง ๆ หมายถึงอะไร?

A. จำนวนแถวจริงที่ query ส่งคืนทั้งหมด
B. จำนวนแถวเฉลี่ยที่ node นั้นส่งออกต่อการ loop หนึ่งครั้ง
C. จำนวนหน้า (page) ที่ถูกอ่านจาก disk
D. cost โดยประมาณของ planner

<details>
<summary>เฉลยข้อ 4</summary>

**คำตอบ: B**

ค่า `rows` ใน `EXPLAIN ANALYZE` คือจำนวนแถวเฉลี่ยต่อการ loop หนึ่งรอบ ถ้าต้องการจำนวนแถวรวมทั้งหมดต้องคูณด้วยค่า `loops` ซึ่งเป็นจุดที่ผู้เริ่มต้นตีความผิดบ่อยมาก

</details>

---

**ข้อ 5.** เหตุใด transaction ID wraparound จึงเป็นปัญหาร้ายแรงกับ PostgreSQL?

A. ทำให้ index เสียหายถาวร
B. ทำให้ tuple ที่ควรมองเห็นได้กลายเป็นมองไม่เห็น (แถวเก่ากลับดูเหมือนอยู่ใน "อนาคต")
C. ทำให้ WAL หยุดเขียน
D. ทำให้ replication หยุดทำงานทันที

<details>
<summary>เฉลยข้อ 5</summary>

**คำตอบ: B**

Transaction ID (XID) เป็นเลข 32-bit แบบ circular เมื่อ XID วนครบรอบโดยไม่มีการ freeze tuple เก่า ระบบจะตีความ tuple เก่าว่าอยู่ใน "อนาคต" เทียบกับ transaction ปัจจุบัน ทำให้มองไม่เห็นข้อมูล (data loss เชิงตรรกะ) PostgreSQL จึงบังคับให้ autovacuum ทำ freeze และจะปฏิเสธการรับ transaction ใหม่เมื่อใกล้ wraparound เพื่อป้องกันปัญหานี้

</details>

---

**ข้อ 6.** คำสั่งใดใช้สำหรับสร้าง Point-in-Time Recovery base backup?

A. `pg_dump`
B. `pg_basebackup`
C. `pg_restore`
D. `COPY`

<details>
<summary>เฉลยข้อ 6</summary>

**คำตอบ: B**

`pg_basebackup` สร้าง physical copy ของ data directory ทั้งหมด ใช้ร่วมกับ WAL archiving เพื่อทำ PITR ส่วน `pg_dump`/`pg_restore` เป็น logical backup ที่ไม่รองรับ point-in-time recovery แบบ WAL replay

</details>

---

**ข้อ 7.** ในการตั้งค่า streaming replication แบบ synchronous หาก standby ที่ระบุใน `synchronous_standby_names` หยุดทำงาน จะเกิดอะไรขึ้นกับ transaction บน primary?

A. Transaction จะ commit ได้ตามปกติทันที
B. Transaction จะ block รอจนกว่า standby จะกลับมา (หรือจนกว่าจะเปลี่ยนการตั้งค่า)
C. Primary จะ crash ทันที
D. Primary จะ auto-failover ไปยัง standby อื่นโดยอัตโนมัติ

<details>
<summary>เฉลยข้อ 7</summary>

**คำตอบ: B**

Synchronous replication รับประกันว่า transaction จะ commit ก็ต่อเมื่อ standby ที่กำหนดยืนยันว่าได้รับ WAL แล้ว หาก standby ล่ม transaction บน primary จะค้าง (hang) จนกว่า standby จะกลับมา หรือ DBA ต้อง manual intervene (เช่นเปลี่ยนเป็น async ชั่วคราว) นี่คือ trade-off สำคัญระหว่าง durability กับ availability

</details>

---

**ข้อ 8.** `GRANT SELECT ON ALL TABLES IN SCHEMA public TO readonly_role;` มีข้อจำกัดอย่างไร?

A. ใช้ได้กับตารางที่มีอยู่แล้วเท่านั้น ตารางที่สร้างใหม่ในอนาคตจะไม่ได้สิทธิ์อัตโนมัติ
B. ให้สิทธิ์ INSERT ด้วยโดยอัตโนมัติ
C. ใช้ได้กับทุก schema ในฐานข้อมูล
D. ต้องเป็น superuser เท่านั้นจึงจะรันได้

<details>
<summary>เฉลยข้อ 8</summary>

**คำตอบ: A**

คำสั่งนี้ให้สิทธิ์เฉพาะตารางที่มีอยู่ ณ ขณะรันคำสั่งเท่านั้น หากต้องการให้ role ได้สิทธิ์กับตารางที่จะถูกสร้างในอนาคตโดยอัตโนมัติ ต้องใช้ `ALTER DEFAULT PRIVILEGES` ควบคู่กันไป

</details>

---

**ข้อ 9.** เหตุใด BRIN index จึงมีขนาดเล็กกว่า B-tree index มากเมื่อใช้กับตารางขนาดใหญ่?

A. BRIN เก็บเฉพาะค่า min/max ของแต่ละช่วงบล็อกหน้า (page range) แทนที่จะเก็บทุกค่า
B. BRIN ไม่เก็บดัชนีจริง เป็นเพียง hint
C. BRIN บีบอัดข้อมูลด้วย gzip
D. BRIN ใช้ hashing แทน sorting

<details>
<summary>เฉลยข้อ 9</summary>

**คำตอบ: A**

BRIN (Block Range Index) เก็บ summary (เช่น min/max) ต่อช่วงของ physical page แทนที่จะสร้าง entry ต่อแถวเหมือน B-tree ทำให้มีขนาดเล็กมาก เหมาะกับข้อมูลที่มี correlation สูงกับลำดับการจัดเก็บ เช่น timestamp ที่ insert ตามลำดับเวลา แต่ประสิทธิภาพจะแย่ลงมากถ้าข้อมูลไม่ correlate กับลำดับ physical

</details>

---

**ข้อ 10.** คำสั่ง `VACUUM` (ไม่ใช่ `VACUUM FULL`) ทำสิ่งใดต่อไปนี้?

A. คืนพื้นที่ disk กลับให้ OS ทันที
B. ทำเครื่องหมาย (mark) พื้นที่ของ dead tuple ให้นำกลับมาใช้ใหม่ได้ภายในตารางเดิม โดยไม่ lock ตารางแบบ exclusive
C. Lock ตารางแบบ `ACCESS EXCLUSIVE` ตลอดกระบวนการ
D. Reindex ตารางโดยอัตโนมัติ

<details>
<summary>เฉลยข้อ 10</summary>

**คำตอบ: B**

`VACUUM` ธรรมดาทำงานแบบ online (ใช้ lock ระดับต่ำ) และคืนพื้นที่ว่างให้ใช้ซ้ำภายในไฟล์ตารางเดิม ไม่คืนกลับให้ OS ส่วน `VACUUM FULL` จะ rewrite ตารางทั้งหมดและคืนพื้นที่ให้ OS จริง แต่ต้องใช้ `ACCESS EXCLUSIVE LOCK` ตลอดกระบวนการ

</details>

---

**ข้อ 11.** `pgbouncer` ในโหมด `transaction pooling` มีข้อจำกัดสำคัญกับฟีเจอร์ใดของ PostgreSQL?

A. `SELECT` ธรรมดา
B. Session-level features เช่น `PREPARE` statement, advisory lock ที่ต้องคงอยู่ข้าม transaction, `SET` session variable
C. `INSERT`/`UPDATE`
D. Index scan

<details>
<summary>เฉลยข้อ 11</summary>

**คำตอบ: B**

Transaction pooling คืนค่า connection กลับสู่ pool ทันทีที่ transaction จบ (ไม่ใช่เมื่อ client disconnect) จึงไม่รับประกันว่า session state (prepared statement, session-level advisory lock, `SET` ที่ไม่ใช่ `SET LOCAL`) จะยังอยู่ในการเชื่อมต่อครั้งถัดไป ต้องออกแบบ application ให้ไม่พึ่งพา session state ข้าม transaction

</details>

---

**ข้อ 12.** เมื่อใช้ `SERIALIZABLE` isolation level PostgreSQL ใช้เทคนิคใดในการตรวจจับความขัดแย้ง?

A. Two-phase locking (2PL) แบบดั้งเดิม
B. Serializable Snapshot Isolation (SSI)
C. Optimistic locking ด้วย version column เท่านั้น
D. ไม่มีการตรวจจับ อาศัย application จัดการเอง

<details>
<summary>เฉลยข้อ 12</summary>

**คำตอบ: B**

PostgreSQL ใช้ Serializable Snapshot Isolation (SSI) ซึ่งอนุญาตให้ transaction ทำงานแบบ snapshot ตามปกติ (ไม่ block กันเหมือน 2PL) แต่จะตรวจจับ dependency cycle ที่อาจทำให้ผลลัพธ์ไม่ serializable แล้ว abort transaction หนึ่งด้วย serialization failure (SQLSTATE 40001) ซึ่ง application ต้องเตรียม retry logic ไว้รองรับ

</details>

---

**ข้อ 13.** ตาราง Partitioning แบบ `RANGE` เหมาะกับ use case ใดมากที่สุด?

A. การแบ่งข้อมูลตาม category ที่ไม่ต่อเนื่อง เช่น ประเทศ
B. การแบ่งข้อมูลตามช่วงเวลา เช่น log ที่แบ่งเป็นรายเดือน
C. การกระจายข้อมูลแบบสุ่มเพื่อ load balancing
D. การแบ่งข้อมูลตาม hash ของ primary key

<details>
<summary>เฉลยข้อ 13</summary>

**คำตอบ: B**

`RANGE` partitioning เหมาะกับข้อมูลที่มีลำดับต่อเนื่อง เช่น วันที่/เวลา ทำให้สามารถทำ partition pruning ได้อย่างมีประสิทธิภาพเมื่อ query กรองด้วยช่วงเวลา และง่ายต่อการ drop partition เก่าทิ้งเมื่อหมดอายุการเก็บ (data retention) ส่วน category ที่ไม่ต่อเนื่องเหมาะกับ `LIST` และการกระจายสุ่มเหมาะกับ `HASH`

</details>

---

**ข้อ 14.** Logical replication ใน PostgreSQL มีข้อจำกัดใดต่อไปนี้ (เลือกข้อที่ถูกต้องที่สุด)?

A. ไม่สามารถ replicate ระหว่างต่าง major version ได้
B. ไม่ replicate DDL changes โดยอัตโนมัติ (ต้องจัดการ schema change เอง)
C. ต้องใช้ shared_buffers เท่ากันทั้งสองฝั่ง
D. ใช้ได้กับ superuser เท่านั้นในการอ่านข้อมูลที่ replicate มา

<details>
<summary>เฉลยข้อ 14</summary>

**คำตอบ: B**

Logical replication ทำงานผ่าน publication/subscription และ replicate เฉพาะ DML (INSERT/UPDATE/DELETE) ไม่ replicate DDL (เช่น `ALTER TABLE`) โดยอัตโนมัติ ผู้ดูแลต้อง apply schema change ที่ฝั่ง subscriber เอง ข้อดีคือสามารถ replicate ข้าม major version ได้ ต่างจาก streaming replication ที่ต้องเป็น binary-compatible เวอร์ชันเดียวกัน

</details>

---

**ข้อ 15.** คำสั่ง `ALTER TABLE big_table ADD COLUMN status text DEFAULT 'active' NOT NULL;` บน PostgreSQL เวอร์ชัน 11 ขึ้นไป มีพฤติกรรมอย่างไร?

A. ต้อง rewrite ตารางทั้งหมดเสมอ เพราะมี `NOT NULL`
B. ไม่ต้อง rewrite ตารางทั้งหมด เพราะ PostgreSQL เก็บค่า default แบบ metadata-only สำหรับค่าคงที่ (constant default)
C. จะ error ทันทีเพราะ column ใหม่มี `NOT NULL` แต่ไม่มี default
D. ต้อง lock ตารางแบบ `ACCESS EXCLUSIVE` ตลอดการ rewrite เสมอในทุกกรณี

<details>
<summary>เฉลยข้อ 15</summary>

**คำตอบ: B**

ตั้งแต่ PostgreSQL 11 เป็นต้นไป การเพิ่มคอลัมน์ที่มี default เป็นค่าคงที่ (constant) จะถูกเก็บเป็น metadata ในระบบ catalog แทนที่จะเขียนค่าลงทุกแถวทันที (lazy/virtual default) ทำให้คำสั่งนี้ทำงานเร็วมากแม้ตารางจะมีข้อมูลนับล้านแถว (แต่ถ้า default เป็น volatile expression เช่น `now()` หรือ `random()` ยังคง rewrite ตารางเหมือนเดิม)

</details>

---

หากทำข้อสอบชุดนี้ได้ถูกต้อง 12/15 ขึ้นไป ถือว่าพร้อมสำหรับสอบ EDB Associate แล้ว หากต้องการเตรียมสอบ Professional ควรฝึก hands-on lab จริงเพิ่มเติม (ตั้งค่า replication, ทำ PITR, วิเคราะห์ slow query จริงด้วยมือ) เพราะข้อสอบระดับนั้นเน้น performance-based assessment ไม่ใช่ปรนัยเพียงอย่างเดียว

---

## Step 992: System Design Interview ที่เกี่ยวกับ Database

### 992.1 กรอบการตอบคำถาม System Design ที่เกี่ยวกับฐานข้อมูล

คำถาม system design ที่มี database เป็นศูนย์กลาง ผู้สัมภาษณ์ต้องการเห็น **กระบวนการคิด** ไม่ใช่คำตอบท่องจำ ควรตอบตามลำดับนี้เสมอ:

1. **Clarify requirements** — ถามคำถามกลับก่อนเริ่มออกแบบ: ปริมาณ traffic (read/write ratio), ขนาดข้อมูล, consistency requirement (strong vs eventual), latency requirement, budget/team size
2. **Estimate scale** — คำนวณคร่าว ๆ (back-of-envelope): QPS, storage growth ต่อวัน/ปี, จำนวน concurrent connection
3. **Design schema** — เริ่มจาก entity หลัก, ความสัมพันธ์, normalize ก่อนแล้วค่อยพิจารณา denormalize ตาม access pattern
4. **Indexing strategy** — ระบุ query pattern หลัก แล้วออกแบบ index ให้ตรงกับ query นั้น ไม่ index ทุกคอลัมน์
5. **Scaling strategy** — เริ่มจาก vertical scaling → read replica → caching → partitioning → sharding ตามลำดับความซับซ้อนที่เพิ่มขึ้น (อย่ากระโดดไป sharding ทันทีถ้ายังไม่จำเป็น)
6. **Trade-offs** — พูดถึงข้อดี/ข้อเสียของทางเลือกแต่ละแบบอย่างตรงไปตรงมา ผู้สัมภาษณ์ระดับ senior จะให้คะแนนสูงกับคนที่พูดถึง trade-off ได้ ไม่ใช่คนที่บอกว่ามีทางออกสมบูรณ์แบบ

### 992.2 Scenario 1: ออกแบบระบบ URL Shortener (เช่น bit.ly)

**Requirements ที่ clarify:** รองรับ 100 ล้าน URL ใหม่ต่อเดือน, read:write ratio ประมาณ 100:1 (คนคลิกลิงก์มากกว่าคนสร้างลิงก์มาก), ต้องการ custom alias ได้, ต้องการ analytics (click count) แบบ near-real-time ไม่ต้อง strong consistency

**Schema design:**

```sql
CREATE TABLE urls (
    id          BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    short_code  VARCHAR(10) NOT NULL,
    long_url    TEXT NOT NULL,
    user_id     BIGINT REFERENCES users(id),
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    expires_at  TIMESTAMPTZ,
    is_active   BOOLEAN NOT NULL DEFAULT true
);

CREATE UNIQUE INDEX idx_urls_short_code ON urls (short_code);

-- เก็บ click event แยกออกมาเป็นตารางต่างหาก เพื่อไม่ให้เขียนทับตาราง urls บ่อย
CREATE TABLE click_events (
    id          BIGINT GENERATED ALWAYS AS IDENTITY,
    url_id      BIGINT NOT NULL REFERENCES urls(id),
    clicked_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    referrer    TEXT,
    user_agent  TEXT,
    country     CHAR(2)
) PARTITION BY RANGE (clicked_at);
```

**การออกแบบ short_code:** ประเด็นสำคัญที่ผู้สมัครมักตอบผิดคือการใช้ auto-increment แล้วแปลงเป็น base62 ตรง ๆ ซึ่งทำให้เดา URL ถัดไปได้ง่าย (security concern) แนวทางที่ดีกว่า:

- ใช้ `BIGINT` sequence แล้ว **สลับบิต (bit shuffling)** หรือ XOR ด้วย constant ก่อนแปลงเป็น base62 เพื่อไม่ให้เรียงลำดับเดาได้
- หรือใช้ **pre-generated pool ของ short code** ที่สุ่มไว้ล่วงหน้าในตารางแยก แล้ว claim ออกมาใช้ (ลด collision check ตอน insert)
- หลีกเลี่ยงการ generate random string แล้วเช็ค collision ใน loop เพราะเมื่อ table โตขึ้น collision rate จะสูงขึ้นเรื่อย ๆ

**Indexing strategy:** unique index บน `short_code` (สำหรับ redirect lookup ซึ่งเป็น query ที่ถี่ที่สุด), index บน `(user_id, created_at)` สำหรับหน้า dashboard ของผู้ใช้ที่ต้องการดูลิงก์ของตัวเอง

**Scaling strategy:**

- **Read path (redirect):** cache ผลลัพธ์ `short_code → long_url` ไว้ที่ Redis/CDN edge เพราะ read:write = 100:1 การ hit database ทุกครั้งไม่จำเป็น — ตั้ง TTL หรือ invalidate cache เมื่อ URL ถูกปิดใช้งาน
- **Write path:** ปริมาณการสร้าง URL ใหม่ต่ำกว่ามาก ใช้ primary เดียวรองรับได้สบาย
- **Click analytics:** อย่าทำ `UPDATE urls SET click_count = click_count + 1` ทุกครั้งที่มีคนคลิก เพราะจะเกิด row contention บนแถวยอดนิยม (hot row) ให้ insert เป็น event log แล้ว aggregate แบบ async ด้วย batch job หรือ message queue (Kafka) แล้วค่อย update ยอดรวมเป็นระยะ (Part 091-092)
- **Partitioning:** `click_events` แบ่งเป็น partition รายเดือนหรือรายสัปดาห์ (Part 054) เพื่อให้ query analytics เร็วและลบข้อมูลเก่าออกได้ง่ายด้วย `DROP PARTITION`
- เมื่อ scale ใหญ่มาก (พันล้าน URL) จึงค่อยพิจารณา sharding ตาม hash ของ `short_code` แต่สำหรับ 100 ล้าน URL/เดือน database เดียวที่ tune ดีพอ + read replica + cache ก็เพียงพอแล้ว

### 992.3 Scenario 2: ออกแบบระบบ E-commerce

**Requirements ที่ clarify:** สินค้าหลักแสนถึงล้านรายการ, ต้องรองรับ flash sale (write spike สูงมากในช่วงสั้น ๆ), ต้อง strong consistency สำหรับ inventory (ห้าม oversell), ต้องการ order history เก็บระยะยาว

**Schema design (core tables):**

```sql
CREATE TABLE products (
    id            BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    sku           VARCHAR(64) UNIQUE NOT NULL,
    name          TEXT NOT NULL,
    category_id   BIGINT REFERENCES categories(id),
    price_cents   BIGINT NOT NULL CHECK (price_cents >= 0),
    is_active     BOOLEAN NOT NULL DEFAULT true
);

CREATE TABLE inventory (
    product_id    BIGINT PRIMARY KEY REFERENCES products(id),
    quantity      INT NOT NULL CHECK (quantity >= 0),
    reserved_qty  INT NOT NULL DEFAULT 0 CHECK (reserved_qty >= 0),
    updated_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE orders (
    id            BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    user_id       BIGINT NOT NULL REFERENCES users(id),
    status        TEXT NOT NULL DEFAULT 'pending',
    total_cents   BIGINT NOT NULL,
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE TABLE order_items (
    order_id      BIGINT NOT NULL REFERENCES orders(id),
    product_id    BIGINT NOT NULL REFERENCES products(id),
    quantity      INT NOT NULL CHECK (quantity > 0),
    price_cents   BIGINT NOT NULL,
    PRIMARY KEY (order_id, product_id)
);
```

**การป้องกัน oversell (จุดที่ผู้สัมภาษณ์มักถามลึก):**

```sql
BEGIN;
-- ล็อกแถว inventory ของสินค้านั้นก่อนตรวจสอบ เพื่อป้องกัน race condition
SELECT quantity - reserved_qty AS available
FROM inventory
WHERE product_id = 123
FOR UPDATE;

-- ถ้า available >= จำนวนที่ต้องการสั่งซื้อ
UPDATE inventory
SET reserved_qty = reserved_qty + 2
WHERE product_id = 123;

INSERT INTO orders (...) VALUES (...);
INSERT INTO order_items (...) VALUES (...);
COMMIT;
```

ควรอธิบายเพิ่มเติมว่า `SELECT ... FOR UPDATE` ป้องกัน race condition ระหว่าง concurrent transaction สองตัวที่พยายามจองสินค้าชิ้นสุดท้ายพร้อมกัน (คนที่มาทีหลังจะรอจน transaction แรก commit/rollback ก่อน) ทางเลือกอื่นคือ **optimistic locking** ด้วย version column (`WHERE quantity = expected_quantity`) ซึ่งเหมาะกับ contention ต่ำกว่า แต่สำหรับ flash sale ที่มี hot row ชัดเจน `FOR UPDATE` มักคาดเดาพฤติกรรมได้ง่ายกว่า

**Indexing strategy:**

- Composite index `(category_id, is_active, price_cents)` สำหรับหน้า browse/filter สินค้าตาม category และเรียงตามราคา
- Partial index `WHERE is_active = true` สำหรับ query ที่กรองเฉพาะสินค้าที่ขายอยู่ (Part 043) ลดขนาด index และเพิ่มความเร็ว
- Full-text search (Part 053) หรือพิจารณาใช้ external search engine (Elasticsearch/OpenSearch) เมื่อต้องการ relevance ranking ซับซ้อน — ควรพูดถึง trade-off ว่า PostgreSQL FTS เพียงพอสำหรับ catalog ขนาดกลาง แต่ระบบ search ที่ต้องการ typo-tolerance, faceted search ขั้นสูงอาจต้องแยก search engine ออกไป (polyglot persistence)

**Scaling strategy:**

- **Read replica** สำหรับหน้า browse/product listing ที่ traffic สูงและ tolerate eventual consistency ได้ (ราคา/สต๊อกดีเลย์เสี้ยววินาทีไม่กระทบมาก)
- **Cache layer (Redis)** สำหรับข้อมูลสินค้าที่เปลี่ยนไม่บ่อย เช่น product detail page
- **Partition ตาราง orders รายเดือน/รายไตรมาส** (Part 054) เพื่อให้ query order history เร็วและ archive ข้อมูลเก่าได้
- **แยก inventory ออกจาก catalog database ได้ในระยะยาว** ถ้า write contention สูงมาก (microservice แยก inventory service ต่างหาก ใช้ event-driven ผ่าน message queue — Part 091)
- สำหรับ flash sale ที่ write spike รุนแรงมาก อาจพิจารณา **queue-based order intake** (รับ order เข้าคิวก่อน แล้วค่อย process เข้า database ตามอัตราที่ database รับไหว) แทนที่จะให้ traffic ยิงเข้า database ตรง ๆ

### 992.4 Scenario 3: ออกแบบระบบ Chat / Messaging

**Requirements ที่ clarify:** รองรับ 1-on-1 chat และ group chat, ต้องการ pagination ย้อนดูข้อความเก่า, ต้องการ read receipt, latency การส่งข้อความต้องต่ำ (< 200ms)

**Schema design:**

```sql
CREATE TABLE conversations (
    id          BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    type        TEXT NOT NULL CHECK (type IN ('direct', 'group')),
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE conversation_members (
    conversation_id  BIGINT NOT NULL REFERENCES conversations(id),
    user_id          BIGINT NOT NULL REFERENCES users(id),
    last_read_msg_id BIGINT,
    joined_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (conversation_id, user_id)
);

CREATE TABLE messages (
    id               BIGINT GENERATED ALWAYS AS IDENTITY,
    conversation_id  BIGINT NOT NULL REFERENCES conversations(id),
    sender_id        BIGINT NOT NULL REFERENCES users(id),
    body             TEXT NOT NULL,
    created_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (conversation_id, id)
) PARTITION BY HASH (conversation_id);
```

**ประเด็นสำคัญที่ต้องพูดถึง:**

1. **Pagination แบบ keyset (ไม่ใช้ `OFFSET`)** — การเลื่อนดูข้อความเก่าต้องใช้ `WHERE conversation_id = ? AND id < :last_seen_id ORDER BY id DESC LIMIT 50` แทนการใช้ `OFFSET` เพราะ `OFFSET` จะช้าลงเรื่อย ๆ เมื่อเลื่อนลึก และข้อมูล real-time ที่เพิ่มเข้ามาตลอดเวลาทำให้ `OFFSET` ได้ผลลัพธ์ซ้ำ/ขาดหายได้ (Part 012, 030)
2. **Composite primary key `(conversation_id, id)`** ทำให้ query ที่กรองด้วย `conversation_id` แล้วเรียงตาม `id` ใช้ index ได้อย่างมีประสิทธิภาพ ตรงกับ access pattern จริง (อ่านข้อความในห้องเดียวเป็นหลัก แทบไม่มี query ข้ามห้อง)
3. **Read receipt** — ไม่ควร insert แถวใหม่ทุกครั้งที่มีคนอ่านข้อความ (จะทำให้ตารางโตเร็วมากและ contention สูงในกลุ่มใหญ่) แนวทางที่ดีกว่าคือเก็บแค่ `last_read_msg_id` ต่อ (conversation, user) ใน `conversation_members` แล้วคำนวณ "unread count" จาก `messages.id > last_read_msg_id`
4. **Partitioning by HASH(conversation_id)** — กระจายข้อความของห้องแชทต่าง ๆ ไปยังหลาย partition เพื่อลด lock contention และช่วยให้ vacuum/index maintenance ทำงานได้เร็วขึ้นในแต่ละ partition ที่เล็กลง ต่างจาก URL shortener ที่ partition ตามเวลาเพราะ chat query มักกรองด้วย `conversation_id` เป็นหลัก ไม่ใช่ช่วงเวลา
5. **Real-time delivery** ไม่ควรพึ่งพา PostgreSQL `LISTEN/NOTIFY` เป็นกลไกหลักสำหรับระบบขนาดใหญ่ เพราะ payload ของ `NOTIFY` มีขนาดจำกัด (8000 bytes) และไม่มี durability/replay guarantee หากผู้ฟังหลุดการเชื่อมต่อ ระบบจริงมักใช้ message broker (Kafka/Redis Pub-Sub) หรือ WebSocket gateway แยกต่างหาก แล้วให้ PostgreSQL เป็น source of truth สำหรับ persisted message เท่านั้น

**Scaling strategy:** read replica สำหรับ query ประวัติแชทเก่าที่ไม่ต้อง real-time, cache รายชื่อ conversation ล่าสุดของผู้ใช้แต่ละคน, และเมื่อ scale ใหญ่มากจึง shard ตาม `conversation_id` หรือ `user_id` ข้าม database instance หลายตัว

### 992.5 Scenario 4: ออกแบบ News Feed (Social Media Feed)

**Requirements ที่ clarify:** ผู้ใช้ follow กันได้, ต้องการเห็น feed โพสต์ของคนที่ follow เรียงตามเวลา, มี celebrity user ที่มี follower หลักล้านคน (hot key problem)

**ประเด็นออกแบบหลัก — Fan-out on Write vs Fan-out on Read:**

| แนวทาง | วิธีการ | ข้อดี | ข้อเสีย |
|---|---|---|---|
| Fan-out on write | เมื่อโพสต์ใหม่ ให้ insert เข้า feed table ของ follower ทุกคนทันที | อ่าน feed เร็วมาก (query ตารางเดียว) | เขียนหนักมากเมื่อ user มี follower เป็นล้าน (celebrity problem) |
| Fan-out on read | ไม่ pre-compute feed, query แบบ join ตอนอ่าน (`posts JOIN follows`) | เขียนเบา | อ่านช้าลงเมื่อ follow เยอะ ต้อง aggregate/merge หลายแหล่งตอน query |
| Hybrid | user ทั่วไปใช้ fan-out on write, celebrity ใช้ fan-out on read แล้ว merge ตอน serve feed | สมดุลระหว่างสองแบบ | ระบบซับซ้อนขึ้น ต้องจัดการ 2 code path |

คำตอบที่ดีที่สุดในการสัมภาษณ์คือการเสนอ **แนวทาง hybrid** พร้อมอธิบายเหตุผล และ **schema** คร่าว ๆ:

```sql
CREATE TABLE follows (
    follower_id  BIGINT NOT NULL REFERENCES users(id),
    followee_id  BIGINT NOT NULL REFERENCES users(id),
    created_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (follower_id, followee_id)
);
CREATE INDEX idx_follows_followee ON follows (followee_id);

CREATE TABLE posts (
    id          BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    author_id   BIGINT NOT NULL REFERENCES users(id),
    body        TEXT,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- pre-computed feed สำหรับ fan-out on write (user ทั่วไป)
CREATE TABLE feed_items (
    user_id     BIGINT NOT NULL,
    post_id     BIGINT NOT NULL,
    created_at  TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (user_id, created_at, post_id)
);
```

พึงอธิบายว่า `feed_items` ใน PostgreSQL ล้วนอาจไม่ใช่ทางเลือกที่ scale ที่สุดในระยะยาว (ระบบจริงระดับ Twitter/Facebook มักใช้ Redis sorted set หรือ wide-column store สำหรับ feed cache) แต่สำหรับการสัมภาษณ์ระดับกลาง การเสนอ PostgreSQL + partition ตาม `user_id` (hash) + คำอธิบาย trade-off ที่ถูกต้องก็เพียงพอแล้ว สิ่งสำคัญคือแสดงให้เห็นว่าเข้าใจ **fundamental trade-off ของการ denormalize เพื่อความเร็วในการอ่าน แลกกับ write amplification**

---

## Step 993: Behavioral/Technical Deep-dive Interview สำหรับ DBA/Backend Engineer

### 993.1 หลักการตอบคำถาม Technical Deep-dive

คำถามกลุ่มนี้ไม่ได้ต้องการคำตอบท่องจำนิยาม แต่ต้องการเห็นว่าคุณ **เข้าใจกลไกภายใน** และ **เคยเจอปัญหาจริง** คำตอบที่ดีควรมีโครงสร้าง: (1) นิยามสั้น ๆ (2) กลไกการทำงาน (3) ตัวอย่างหรือประสบการณ์จริงถ้ามี (4) ข้อควรระวัง/trade-off

สำหรับคำถามเชิง behavioral ("เล่า incident ที่เคยเจอ") ให้ใช้โครงสร้าง **STAR** (Situation, Task, Action, Result) และเน้นที่ "สิ่งที่เรียนรู้/เปลี่ยนแปลงหลังเหตุการณ์" เพราะผู้สัมภาษณ์สนใจ growth mindset มากกว่าความสมบูรณ์แบบ

### 993.2 คำถามยอดนิยมพร้อมแนวทางตอบ

**คำถามที่ 1: อธิบาย MVCC (Multi-Version Concurrency Control) ทำงานอย่างไร**

แนวทางตอบ: MVCC คือกลไกที่ทำให้ transaction หลายตัวอ่าน-เขียนพร้อมกันได้โดยไม่ต้อง lock กันตลอดเวลา หัวใจคือทุกแถว (tuple) มี metadata `xmin` (transaction ที่สร้างแถวนี้) และ `xmax` (transaction ที่ลบ/แทนที่แถวนี้) เมื่อ `UPDATE` เกิดขึ้น PostgreSQL จะไม่แก้ไขแถวเดิม แต่สร้าง **tuple version ใหม่** และใส่ `xmax` ให้ tuple เก่า แต่ละ transaction จะเห็นเฉพาะ tuple version ที่ "valid" ตาม snapshot ของตัวเอง (ตาม xmin/xmax เทียบกับ transaction ที่กำลัง commit/active อยู่) ทำให้ reader ไม่ block writer และ writer ไม่ block reader ผลข้างเคียงคือเกิด **dead tuple** สะสม ต้องมี `VACUUM`/`autovacuum` มาเก็บกวาดเป็นระยะ ไม่เช่นนั้นตารางจะบวม (table bloat) และเสี่ยง transaction ID wraparound (เชื่อมโยงกับ Part 058, 060, 084)

**คำถามที่ 2: จะ debug slow query อย่างไร**

แนวทางตอบเป็นขั้นตอน:
1. ใช้ `pg_stat_statements` หา query ที่กิน total_time หรือ mean_time สูงสุด (Part 072, 074)
2. รัน `EXPLAIN (ANALYZE, BUFFERS)` กับ query นั้น เพื่อดู actual execution plan, เทียบ estimated rows กับ actual rows — ถ้าต่างกันมากแสดงว่า statistics ไม่ update (`ANALYZE` table)
3. เช็คว่ามี Seq Scan บนตารางใหญ่โดยไม่จำเป็นหรือไม่ → พิจารณาเพิ่ม index ที่เหมาะสมกับ WHERE/JOIN/ORDER BY
4. เช็ค Buffers: hit vs read สูงว่า query ต้องอ่านจาก disk มากผิดปกติหรือไม่ (อาจต้องเพิ่ม `shared_buffers` หรือ query กรองข้อมูลได้แคบกว่านี้)
5. เช็คว่ามี lock contention หรือไม่ผ่าน `pg_stat_activity` / `pg_locks`
6. เช็ค table/index bloat ด้วย extension เช่น `pgstattuple` — ถ้า autovacuum ตามไม่ทันอาจต้องปรับ `autovacuum_vacuum_scale_factor`
7. สุดท้ายพิจารณา query rewrite (เช่นเปลี่ยน subquery เป็น JOIN, ใช้ CTE อย่างระวังเรื่อง materialization, ตัดคอลัมน์ที่ไม่จำเป็นออกจาก SELECT)

**คำถามที่ 3: เล่า incident ที่เคยเจอ และแก้ปัญหาอย่างไร**

ตัวอย่างแนวทางตอบ (ใช้ STAR, ปรับให้ตรงกับประสบการณ์จริงของตัวเอง): *"ครั้งหนึ่ง production database มี connection ล้น (connection exhaustion) ทำให้ application เริ่ม error 'too many connections' (Situation) หน้าที่ของผมคือหาสาเหตุและแก้ไขโดยเร็วที่สุดเพื่อลด downtime (Task) ผมตรวจสอบ `pg_stat_activity` พบว่ามี connection จำนวนมากค้างอยู่ในสถานะ `idle in transaction` จาก service ตัวหนึ่งที่ลืมปิด transaction หลังเรียก external API ผมสั่ง terminate connection ที่ค้างนานผิดปกติชั่วคราวเพื่อคืนทรัพยากร แล้วเพิ่ม `idle_in_transaction_session_timeout` เป็นมาตรการป้องกันระยะสั้น และตั้ง PgBouncer เป็น transaction pooling mode เพื่อจำกัดจำนวน connection จริงที่ไปถึง PostgreSQL (Action) หลังแก้ไข ระบบกลับมาเสถียร และทีมได้เพิ่ม monitoring alert บนจำนวน idle-in-transaction connection เพื่อจับปัญหานี้ได้ก่อนที่จะกระทบผู้ใช้จริงในครั้งถัดไป (Result)"* — เชื่อมโยงกับ Part 066, 072

**คำถามที่ 4: ความแตกต่างระหว่าง B-tree, GIN, GiST, BRIN, Hash index และเลือกใช้เมื่อไหร่**

แนวทางตอบแบบตาราง (พูดปากเปล่าได้เช่นกัน): B-tree ใช้กับ equality/range ทั่วไปและเป็น default; Hash ใช้กับ equality เท่านั้น เร็วกว่า B-tree เล็กน้อยในบาง case แต่ใช้งานจำกัดกว่ามาก (ตั้งแต่ PG10 รองรับ WAL แล้วจึงใช้ใน production ได้); GIN เหมาะกับข้อมูล composite เช่น array/jsonb/full-text search ที่ต้องการ containment query; GiST เหมาะกับข้อมูลเชิงพื้นที่/ช่วง เช่น geometric data, range type, exclusion constraint; BRIN เหมาะกับตารางใหญ่มากที่ข้อมูล correlate กับลำดับ physical storage เช่น time-series log (Part 041-042)

**คำถามที่ 5: Isolation level ต่างกันอย่างไร และเลือกใช้เมื่อไหร่**

แนวทางตอบ: PostgreSQL รองรับ Read Committed (default), Repeatable Read, Serializable (Read Uncommitted ถูก map ไปเป็น Read Committed เพราะ PostgreSQL ไม่มี dirty read อยู่แล้ว) — Read Committed เหมาะกับงานทั่วไปส่วนใหญ่ที่ไม่ sensitive กับ non-repeatable read; Repeatable Read เหมาะกับ report/batch job ที่ต้องการเห็นข้อมูล ณ จุดเวลาเดียวกันตลอด transaction; Serializable เหมาะกับ logic ที่ sensitive มากต่อ concurrency anomaly เช่น การโอนเงิน/จองที่นั่ง แต่ต้องแลกกับโอกาสเกิด serialization failure ที่ต้อง retry (Part 038)

**คำถามที่ 6: VACUUM/autovacuum ทำงานอย่างไร และทำไมสำคัญ**

แนวทางตอบ: อธิบายเชื่อมโยงกับ MVCC (คำถามที่ 1) — เพราะ UPDATE/DELETE ไม่ได้ลบข้อมูลจริงทันทีแต่สร้าง dead tuple, VACUUM มีหน้าที่ (1) ทำเครื่องหมายพื้นที่ dead tuple ให้ reuse ได้ (2) freeze tuple เก่าเพื่อป้องกัน transaction ID wraparound (3) อัปเดต visibility map เพื่อให้ index-only scan ทำงานได้เร็วขึ้น ควรพูดถึงพารามิเตอร์สำคัญ เช่น `autovacuum_vacuum_scale_factor`, `autovacuum_vacuum_cost_delay` และอาการของปัญหาเมื่อ autovacuum ตามไม่ทัน (table bloat, query ช้าลงเรื่อย ๆ, index โต) (Part 060)

**คำถามที่ 7: Streaming replication vs Logical replication ต่างกันอย่างไร เลือกใช้เมื่อไหร่**

แนวทางตอบ: Streaming replication (physical) ส่ง WAL record ระดับ byte ทำให้ standby เป็นสำเนาที่เหมือน primary ทุกประการ (รวม index, system catalog) เหมาะกับ HA/DR และต้องเป็น PostgreSQL version เดียวกัน; Logical replication ส่งเฉพาะการเปลี่ยนแปลงข้อมูลระดับ row ผ่าน publication/subscription ทำให้เลือก replicate เฉพาะบางตารางได้ ข้าม major version ได้ และรองรับ multi-master แบบจำกัด แต่ไม่ replicate DDL อัตโนมัติ (Part 063-064)

**คำถามที่ 8: ออกแบบ schema สำหรับ multi-tenant SaaS อย่างไร**

แนวทางตอบ: อธิบาย 3 แนวทางหลัก — (1) **Shared schema + tenant_id column** ทุกตารางมีคอลัมน์ `tenant_id` และใช้ Row-Level Security (RLS) บังคับ filter อัตโนมัติ (Part 069) ง่ายต่อการ maintain แต่ noisy-neighbor risk สูง (2) **Schema-per-tenant** แยก PostgreSQL schema ต่อ tenant ในฐานข้อมูลเดียวกัน isolation ดีกว่าแต่ maintenance (migration ต้องรันซ้ำทุก schema) ซับซ้อนขึ้น (3) **Database-per-tenant** isolation สูงสุด เหมาะกับ enterprise tenant ที่ต้องการ compliance เข้มงวด แต่ operational overhead สูงมากเมื่อ tenant เยอะ ควรสรุปว่าการเลือกขึ้นกับจำนวน tenant, ความต้องการ isolation/compliance, และ operational capacity ของทีม

**คำถามที่ 9: Connection pooling สำคัญอย่างไร PgBouncer มีโหมดอะไรบ้าง**

แนวทางตอบ: PostgreSQL แต่ละ connection ใช้ process แยก (ไม่ใช่ thread) ทำให้มี overhead memory/CPU สูงเมื่อ connection เยอะ PgBouncer ช่วยลดจำนวน connection จริงที่ไปถึง PostgreSQL โดยทำ connection multiplexing มี 3 โหมด: session pooling (ปลอดภัยสุด รองรับทุก feature แต่ประหยัด connection น้อยสุด), transaction pooling (คืน connection กลับ pool ทันทีที่ transaction จบ ประหยัดมากแต่จำกัด session-level feature), statement pooling (คืนกลับทันทีหลังแต่ละ statement จำกัดสุด ไม่รองรับ multi-statement transaction) งานส่วนใหญ่นิยม transaction pooling เพื่อ balance ระหว่างประสิทธิภาพและความสามารถ (Part 066)

**คำถามที่ 10: Deadlock คืออะไร เกิดจากอะไร ป้องกันอย่างไร**

แนวทางตอบ: Deadlock เกิดเมื่อ transaction สองตัว (หรือมากกว่า) รอ lock ที่อีกฝ่ายถืออยู่แบบเป็นวงกลม (circular wait) PostgreSQL มี deadlock detector ที่จะตรวจจับและ abort transaction หนึ่งโดยอัตโนมัติพร้อม error `deadlock detected` วิธีป้องกันหลักคือ **lock ตามลำดับเดียวกันเสมอ** (เช่น เรียง `UPDATE` ตาม primary key เสมอ ไม่สลับลำดับ) ลด transaction ให้สั้นและเร็วที่สุด และพิจารณาใช้ `SELECT ... FOR UPDATE` อย่างระมัดระวังเรื่องลำดับการ lock (Part 059)

**คำถามที่ 11: จะวางแผน capacity/scaling อย่างไรเมื่อ traffic คาดว่าจะโต 10 เท่าในปีหน้า**

แนวทางตอบ: เริ่มจากวัด baseline ปัจจุบัน (QPS, CPU/memory/disk utilization, connection count) แล้ว project ตามอัตราการโต, ระบุ bottleneck ที่จะเจอก่อน (มักเป็น connection limit หรือ disk I/O ก่อน CPU), วางแผนเป็นลำดับขั้น: ปรับ config/index ให้ efficient ที่สุดก่อน (ถูกที่สุด) → เพิ่ม read replica แยก read/write traffic → เพิ่ม caching layer ลด load ที่ query ซ้ำ ๆ → พิจารณา partitioning ตารางใหญ่ → ท้ายสุดจึงพิจารณา sharding/microservice หากยังไม่พอ ควรเน้นย้ำว่าการทำ **load testing ล่วงหน้า** และ **monitoring/alerting ที่ดี** สำคัญกว่าการเดาล่วงหน้าเฉย ๆ (Part 075, 091-096)

**คำถามที่ 12: OLTP กับ OLAP ต่างกันอย่างไร PostgreSQL เหมาะกับงานแบบไหน**

แนวทางตอบ: OLTP (Online Transaction Processing) เน้น transaction สั้น จำนวนมาก อ่าน/เขียนแถวจำนวนน้อยต่อครั้ง (เช่น การสั่งซื้อสินค้า) ต้องการ ACID เข้มงวด; OLAP (Online Analytical Processing) เน้น query ที่ scan ข้อมูลจำนวนมาก aggregate ซับซ้อน (เช่น รายงานยอดขายรายเดือน) PostgreSQL ถูกออกแบบมาเพื่อ OLTP เป็นหลักและทำได้ดีมาก แต่ก็รองรับงาน OLAP ระดับกลางได้ด้วย (window functions, parallel query, partitioning, extension อย่าง TimescaleDB สำหรับ time-series) หากเป็น OLAP ขนาดใหญ่มากระดับ data warehouse (เทราไบต์-เพตะไบต์) มักพิจารณาระบบเฉพาะทาง เช่น column-store (ClickHouse, Redshift, BigQuery) ควบคู่ไปด้วย โดยใช้ PostgreSQL เป็น source-of-truth แล้ว ETL/CDC ข้อมูลไปยังระบบ analytics แยกต่างหาก

### 993.3 Checklist ทบทวนความรู้ทั้งหมดจากหลักสูตร (Part 001-100)

ใช้ตารางนี้เป็นแผนที่ทบทวนก่อนสอบสัมภาษณ์ ทำเครื่องหมาย ✓ ในใจ (หรือ print ออกมาติ๊ก) สำหรับหัวข้อที่มั่นใจแล้ว ส่วนที่ยังไม่มั่นใจให้กลับไปอ่าน Part นั้นซ้ำก่อนวันสัมภาษณ์จริง

| Part | หัวข้อ | ระดับ |
|---|---|---|
| 001 | Database พื้นฐาน / RDBMS คืออะไร | พื้นฐาน |
| 002 | การติดตั้ง PostgreSQL | พื้นฐาน |
| 003 | เครื่องมือ (psql, pgAdmin ฯลฯ) | พื้นฐาน |
| 004 | สถาปัตยกรรมของ PostgreSQL | พื้นฐาน |
| 005 | การจัดการฐานข้อมูล (CREATE/DROP DATABASE) | พื้นฐาน |
| 006 | Data types พื้นฐาน | พื้นฐาน |
| 007 | Data types วันที่/เวลา | พื้นฐาน |
| 008 | การออกแบบตาราง | พื้นฐาน |
| 009 | INSERT | พื้นฐาน |
| 010 | SELECT พื้นฐาน | พื้นฐาน |
| 011 | WHERE และ operators | พื้นฐาน |
| 012 | ORDER BY / LIMIT | พื้นฐาน |
| 013 | UPDATE | พื้นฐาน |
| 014 | DELETE / TRUNCATE | พื้นฐาน |
| 015 | การจัดการ NULL | พื้นฐาน |
| 016 | Primary Key / Unique | พื้นฐาน |
| 017 | Foreign Key | พื้นฐาน |
| 018 | CHECK / DEFAULT constraint | พื้นฐาน |
| 019 | ALTER TABLE | พื้นฐาน |
| 020 | Capstone: Library System | พื้นฐาน |
| 021 | INNER JOIN | กลาง |
| 022 | OUTER JOIN | กลาง |
| 023 | CROSS JOIN / SELF JOIN | กลาง |
| 024 | Subqueries | กลาง |
| 025 | CTE (WITH) | กลาง |
| 026 | Recursive CTE | กลาง |
| 027 | Aggregate functions | กลาง |
| 028 | GROUP BY / HAVING | กลาง |
| 029 | Window functions พื้นฐาน | กลาง |
| 030 | Window functions ขั้นสูง | กลาง |
| 031 | String functions | กลาง |
| 032 | Date/Time functions | กลาง |
| 033 | Numeric functions | กลาง |
| 034 | Conditional logic (CASE ฯลฯ) | กลาง |
| 035 | Views | กลาง |
| 036 | Materialized Views | กลาง |
| 037 | Transactions / ACID | กลาง |
| 038 | Isolation Levels | กลาง |
| 039 | Sequences / Identity | กลาง |
| 040 | Capstone: E-commerce | กลาง |
| 041 | B-tree Index | ขั้นสูง |
| 042 | Index ชนิดอื่น (Hash, GIN, GiST, SP-GiST, BRIN) | ขั้นสูง |
| 043 | Partial / Expression Index | ขั้นสูง |
| 044 | Query Planner และ EXPLAIN | ขั้นสูง |
| 045 | Query Optimization | ขั้นสูง |
| 046 | PL/pgSQL พื้นฐาน | ขั้นสูง |
| 047 | PL/pgSQL ขั้นสูง | ขั้นสูง |
| 048 | Triggers | ขั้นสูง |
| 049 | Custom Types / Domains | ขั้นสูง |
| 050 | Arrays | ขั้นสูง |
| 051 | JSON/JSONB พื้นฐาน | ขั้นสูง |
| 052 | JSONB ขั้นสูง | ขั้นสูง |
| 053 | Full-Text Search | ขั้นสูง |
| 054 | Table Partitioning | ขั้นสูง |
| 055 | Table Inheritance | ขั้นสูง |
| 056 | Foreign Data Wrappers | ขั้นสูง |
| 057 | Extensions | ขั้นสูง |
| 058 | MVCC Deep Dive | ขั้นสูง |
| 059 | Deadlocks | ขั้นสูง |
| 060 | VACUUM / Autovacuum | ขั้นสูง |
| 061 | Backup Strategies | มืออาชีพ |
| 062 | Point-in-Time Recovery | มืออาชีพ |
| 063 | Streaming Replication | มืออาชีพ |
| 064 | Logical Replication | มืออาชีพ |
| 065 | High Availability | มืออาชีพ |
| 066 | Connection Pooling | มืออาชีพ |
| 067 | Load Balancing | มืออาชีพ |
| 068 | Roles / Privileges | มืออาชีพ |
| 069 | Row-Level Security | มืออาชีพ |
| 070 | SSL / Encryption | มืออาชีพ |
| 071 | Auditing / Logging | มืออาชีพ |
| 072 | Monitoring | มืออาชีพ |
| 073 | Tuning postgresql.conf | มืออาชีพ |
| 074 | Query-level Tuning | มืออาชีพ |
| 075 | Capacity Planning | มืออาชีพ |
| 076 | PostGIS พื้นฐาน | มืออาชีพ |
| 077 | PostGIS ขั้นสูง | มืออาชีพ |
| 078 | TimescaleDB | มืออาชีพ |
| 079 | Migration Strategies | มืออาชีพ |
| 080 | Capstone: SaaS Platform | มืออาชีพ |
| 081 | Process Architecture ภายใน | World-Class |
| 082 | Storage Engine | World-Class |
| 083 | WAL Deep Dive | World-Class |
| 084 | MVCC Internals | World-Class |
| 085 | เขียน C Extension | World-Class |
| 086 | Query Planner Internals | World-Class |
| 087 | Node.js Integration | World-Class |
| 088 | Python Integration | World-Class |
| 089 | Go Integration | World-Class |
| 090 | Java Integration | World-Class |
| 091 | Microservices Patterns | World-Class |
| 092 | Event Sourcing / CQRS | World-Class |
| 093 | Docker / Kubernetes | World-Class |
| 094 | AWS RDS / Aurora | World-Class |
| 095 | GCP Cloud SQL / AlloyDB | World-Class |
| 096 | Azure Database | World-Class |
| 097-100 | หัวข้อ World-Class เพิ่มเติม (performance engineering, reliability/chaos, การมีส่วนร่วมกับ core, capstone ระดับโลก) | World-Class |
| 101 | (บทนี้) Certification & Interview Prep | World-Class |

**วิธีใช้ตารางนี้อย่างมีประสิทธิภาพก่อนสัมภาษณ์:**

- 1 สัปดาห์ก่อนสัมภาษณ์: ไล่อ่านหัวข้อ (heading) ของทุก Part ในตาราง ถามตัวเองว่า "ถ้าถูกถามเรื่องนี้ในห้องสัมภาษณ์ จะอธิบายได้ไหม"
- 2-3 วันก่อน: เจาะลึก Part ที่ตรงกับ job description ของตำแหน่งที่สมัคร (เช่น สาย Backend เน้น Part 021-053, 087-092; สาย DBA/Platform เน้น Part 058-075, 081-086, 093-096)
- 1 วันก่อน: ทำข้อสอบฝึกฝนใน Step 991 ซ้ำอีกครั้ง และซ้อมพูดคำตอบ Step 993 ออกเสียงจริง (ไม่ใช่แค่อ่านในใจ)

---

## สรุปท้ายบท

บทนี้เป็นบทสรุปเชิงปฏิบัติที่เชื่อมโยงความรู้ทั้งหมดของหลักสูตรเข้ากับสถานการณ์จริงสองแบบที่ผู้เรียนจะต้องเจอ: **การสอบ certification** และ **การสัมภาษณ์งาน** ประเด็นสำคัญที่ควรจำ:

- Certification อย่าง EDB Associate/Professional เป็นเป้าหมายที่จับต้องได้และช่วยบังคับให้ทบทวนความรู้อย่างเป็นระบบ ควรเลือก cert ให้ตรงกับสายอาชีพ (on-prem DBA vs cloud vs developer)
- คำถาม system design ที่เกี่ยวกับฐานข้อมูลไม่มีคำตอบตายตัว สิ่งที่ผู้สัมภาษณ์มองหาคือ**กระบวนการคิด**: clarify requirement → estimate scale → design schema → index → scale แบบเป็นขั้นตอน พร้อมพูดถึง trade-off อย่างตรงไปตรงมา
- คำถาม technical deep-dive (MVCC, debug slow query, deadlock ฯลฯ) ต้องตอบด้วยความเข้าใจกลไกจริง ไม่ใช่ท่องนิยาม และควรมีตัวอย่างประสบการณ์จริงประกอบเสมอถ้าเป็นไปได้
- Checklist ใน Step 993.3 คือแผนที่สรุปทั้งหลักสูตร ใช้เป็นเครื่องมือวางแผนอ่านทวนก่อนวันสำคัญ

ความรู้ทางเทคนิคที่แน่นเพียงอย่างเดียวไม่พอ — การสื่อสารความรู้นั้นออกมาให้ผู้ฟัง (กรรมการสอบ หรือผู้สัมภาษณ์) เข้าใจได้อย่างเป็นระบบ คือทักษะที่แยกคนที่ "รู้" ออกจากคนที่ "ได้งาน/ได้ certification"

---

## แบบฝึกหัด

<details>
<summary><strong>แบบฝึกหัดที่ 1:</strong> อธิบายความแตกต่างระหว่าง `REPEATABLE READ` และ `SERIALIZABLE` โดยยกตัวอย่าง anomaly ที่ `REPEATABLE READ` ยังป้องกันไม่ได้แต่ `SERIALIZABLE` ป้องกันได้</summary>

**เฉลย:** `REPEATABLE READ` ใน PostgreSQL ป้องกัน dirty read และ non-repeatable read ได้ (เพราะใช้ snapshot เดียวตลอด transaction) แต่ยังสามารถเกิด **serialization anomaly** ได้ ตัวอย่างคลาสสิกคือ "write skew": มีบัญชีสองบัญชี A และ B มี constraint ว่าผลรวมยอดเงินต้อง >= 0 เสมอ transaction 1 ตรวจสอบยอด A แล้วหักเงินจาก A (เพราะเห็นว่า B มีพอ), transaction 2 ตรวจสอบยอด B แล้วหักเงินจาก B (เพราะเห็นว่า A มีพอ) พร้อมกัน ทั้งสอง transaction อ่านเห็น snapshot ก่อนที่อีกฝ่ายจะหักเงิน ทำให้ทั้งคู่ commit ผ่านได้ทั้งที่รวมกันแล้วยอดติดลบ ซึ่งละเมิด constraint ทางธุรกิจ `SERIALIZABLE` (ผ่าน SSI) จะตรวจจับ dependency cycle นี้และ abort transaction หนึ่งด้วย serialization failure บังคับให้ retry

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 2:</strong> ในการออกแบบระบบจองตั๋วคอนเสิร์ต (ticket booking) ที่ต้องป้องกันการขายตั๋วซ้ำ (double booking) ควรใช้เทคนิคใดใน PostgreSQL และทำไม</summary>

**เฉลย:** แนวทางที่แนะนำคือใช้ `SELECT ... FOR UPDATE` ล็อกแถวที่นั่ง/ตั๋วก่อนตรวจสอบสถานะว่าง แล้วค่อย `UPDATE` สถานะเป็น "จองแล้ว" ภายใน transaction เดียวกัน วิธีนี้ป้องกันไม่ให้ request สองตัวจองที่นั่งเดียวกันพร้อมกันได้ เพราะ request ที่สองจะถูก block จนกว่า request แรก commit/rollback อีกทางเลือกคือใช้ unique constraint บนคอลัมน์ `(event_id, seat_number)` ร่วมกับ `INSERT ... ON CONFLICT DO NOTHING` แล้วเช็คว่า insert สำเร็จหรือไม่ (optimistic approach) ซึ่งเหมาะกับกรณีที่ไม่ต้องการ lock ค้างนาน ทั้งสองวิธีควรใช้ isolation level อย่างน้อย `READ COMMITTED` ร่วมกับ explicit locking หรือ unique constraint เพื่อความถูกต้อง ไม่ควรพึ่งพาการเช็คในฝั่ง application เพียงอย่างเดียว (check-then-act ที่ไม่มี lock/constraint จะมี race condition เสมอ)

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 3:</strong> Interviewer ถามว่า "ทำไมไม่ใช้ NoSQL สำหรับระบบนี้แทน PostgreSQL" ควรตอบอย่างไรให้แสดงความเข้าใจ trade-off อย่างสมดุล</summary>

**เฉลย:** คำตอบที่ดีไม่ควรฟันธงว่า PostgreSQL ดีกว่าเสมอ แต่ควรวิเคราะห์ตาม requirement: ถ้าระบบต้องการ **strong consistency, complex relational query, transaction ที่ครอบคลุมหลายตาราง** (เช่น ระบบการเงิน, inventory, order) PostgreSQL (RDBMS) เหมาะกว่าเพราะมี ACID เต็มรูปแบบและ join ที่มีประสิทธิภาพ ถ้าระบบต้องการ **horizontal scale แบบ massive, schema ที่ยืดหยุ่นสูงมาก, หรือ access pattern แบบ key-value/document ล้วน ๆ** ที่ไม่ต้องการ join ซับซ้อน NoSQL (เช่น DynamoDB, MongoDB) อาจเหมาะกว่า นอกจากนี้ควรเสริมว่า PostgreSQL สมัยใหม่รองรับ JSONB (Part 051-052) ทำให้ได้ความยืดหยุ่นแบบ document store บางส่วนโดยยังคง ACID และ join ได้ ทำให้หลายระบบเลือกใช้ PostgreSQL เป็น "ทางเลือกที่ยืดหยุ่นพอ" แทนการแยกใช้สองระบบตั้งแต่ต้น (polyglot persistence ควรเป็นทางเลือกท้าย ๆ เมื่อพิสูจน์แล้วว่าจำเป็นจริง ไม่ใช่ default)

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 4:</strong> อธิบายว่าทำไม index-only scan ถึงเร็วกว่า index scan ปกติ และเงื่อนไขอะไรที่ทำให้ index-only scan เกิดขึ้นได้</summary>

**เฉลย:** Index scan ปกติต้อง (1) ค้นหาใน index เพื่อหาตำแหน่งของ tuple แล้ว (2) กลับไปอ่าน heap (ตารางจริง) เพื่อดึงข้อมูลคอลัมน์ที่ต้องการและตรวจสอบ visibility ของ tuple นั้น (เพราะ index ไม่เก็บข้อมูล MVCC visibility) ส่วน index-only scan สามารถตอบคำตอบได้จาก index เพียงอย่างเดียวโดยไม่ต้องแตะ heap เลย ถ้าคอลัมน์ที่ query ต้องการทั้งหมดอยู่ใน index (covering index) **และ** page ของ heap ที่เกี่ยวข้องถูกมาร์คว่า "all visible" ใน visibility map (ซึ่งอัปเดตโดย VACUUM) เงื่อนไขสำคัญคือตารางต้องถูก vacuum เป็นประจำ ไม่เช่นนั้น visibility map จะไม่ up-to-date และ planner จะ fallback ไปใช้ index scan ปกติแทน (Part 041, 060)

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 5:</strong> ในการสัมภาษณ์ ถูกถามให้ประมาณ (back-of-envelope) ว่าตาราง `orders` ที่มี 10 ล้าน order ต่อวัน เก็บข้อมูลไว้ 2 ปี จะมีขนาดประมาณเท่าไหร่ ถ้าแต่ละแถวเฉลี่ย 200 bytes จะวางแผน partitioning อย่างไร</summary>

**เฉลย:** คำนวณ: 10,000,000 แถว/วัน × 365 วัน × 2 ปี ≈ 7,300 ล้านแถว ขนาดข้อมูลดิบ ≈ 7,300,000,000 × 200 bytes ≈ 1.46 TB (ยังไม่รวม index ซึ่งอาจเพิ่มอีก 30-50%+ รวมแล้วอาจแตะ 2 TB) ด้วยขนาดนี้ควรทำ **RANGE partitioning รายเดือนหรือรายสัปดาห์** บนคอลัมน์ `created_at` เพื่อ (1) ให้ query ที่กรองตามช่วงเวลาทำ partition pruning ได้ ไม่ต้อง scan ทั้ง 2 ปี (2) ลบข้อมูลเก่าที่พ้นระยะเก็บได้ด้วย `DETACH/DROP PARTITION` ซึ่งเร็วกว่า `DELETE` มหาศาล (3) แต่ละ partition มีขนาดเล็กลง ทำให้ VACUUM/index maintenance ทำงานได้เร็วขึ้นและ lock ช่วงเวลาสั้นลง ควรตั้ง automation (เช่น pg_partman หรือ cron job) เพื่อสร้าง partition ใหม่ล่วงหน้าและ drop partition เก่าโดยอัตโนมัติ (Part 054, 075)

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 6:</strong> "อธิบายว่า `shared_buffers` คืออะไร และทำไมค่าที่แนะนำทั่วไปคือประมาณ 25% ของ RAM ไม่ใช่ 80-90%"</summary>

**เฉลย:** `shared_buffers` คือหน่วยความจำที่ PostgreSQL จองไว้เพื่อ cache page ของข้อมูล/index ที่ใช้งานบ่อย ลดการอ่าน disk ซ้ำ เหตุผลที่ไม่แนะนำให้ตั้งสูงมาก (80-90% ของ RAM) เพราะ PostgreSQL พึ่งพา **OS page cache** เป็นชั้น cache ที่สองอยู่แล้ว (double buffering) — ข้อมูลที่ไม่ได้อยู่ใน `shared_buffers` มักจะยังอยู่ใน OS cache ทำให้การอ่านยังเร็วอยู่ ถ้าตั้ง `shared_buffers` สูงเกินไปจะเบียดพื้นที่ที่ OS ใช้ cache ไฟล์อื่น ๆ (WAL, temp file) และเพิ่มภาระของ checkpoint (ต้อง flush dirty page จำนวนมากกว่าลง disk) ค่าประมาณ 25% ของ RAM เป็นจุดสมดุลที่ community ทดสอบแล้วให้ผลลัพธ์ดีในเวิร์กโหลดส่วนใหญ่ แต่ก็ควร benchmark จริงกับ workload ของตัวเองเสมอ (Part 073)

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 7:</strong> ทำไมการใช้ `SELECT *` ในโค้ด production ถึงถือเป็น anti-pattern ทั้งในแง่ performance และ maintainability</summary>

**เฉลย:** ในแง่ **performance**: (1) ดึงคอลัมน์ที่ไม่จำเป็นเพิ่ม network I/O และ memory ที่ใช้; (2) ทำให้ query ไม่สามารถใช้ **index-only scan** ได้ถ้าคอลัมน์บางตัวไม่ได้อยู่ใน index (ต้องกลับไปอ่าน heap เสมอ); (3) ถ้ามี column ขนาดใหญ่ เช่น `TEXT`/`BYTEA`/`JSONB` ที่ query ไม่ได้ต้องการจริง ๆ จะทำให้ query ช้าลงโดยไม่จำเป็น ในแง่ **maintainability**: (1) ถ้ามีการเพิ่มคอลัมน์ใหม่ในอนาคต โค้ด application ที่คาดหวังจำนวน/ลำดับคอลัมน์คงที่อาจพังโดยไม่รู้ตัว; (2) ทำให้อ่านโค้ดยากขึ้นเพราะไม่ชัดเจนว่า query ต้องการข้อมูลอะไรจริง ๆ ควรระบุคอลัมน์ที่ต้องการอย่างชัดเจนเสมอ (Part 010, 041, 074)

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 8:</strong> ถูกถามในห้องสัมภาษณ์ว่า "เคยทำ zero-downtime schema migration บนตารางขนาดใหญ่ไหม มีขั้นตอนอย่างไร" ให้ร่างแนวทางตอบ</summary>

**เฉลย:** แนวทางตอบที่ดีควรยกตัวอย่างเช่นการเพิ่มคอลัมน์ `NOT NULL` บนตารางใหญ่แบบไม่ downtime: (1) เพิ่มคอลัมน์แบบ **nullable** ก่อน พร้อม default (ใช้ fast-default metadata-only ของ PG11+ ถ้าเป็นค่าคงที่) — ขั้นตอนนี้เร็วมาก ไม่ rewrite ตาราง; (2) backfill ข้อมูลให้คอลัมน์นั้นเป็น batch เล็ก ๆ (เช่นทีละ 10,000 แถว) เพื่อไม่ให้ transaction ยาวเกินไปและไม่ lock ตารางนาน; (3) เพิ่ม `CHECK` constraint แบบ `NOT VALID` ก่อน (ไม่ scan ตารางทันที ไม่ lock นาน) แล้วค่อยรัน `VALIDATE CONSTRAINT` แยกต่างหาก (ใช้ lock ระดับเบากว่า `ADD CONSTRAINT` ธรรมดา); (4) เมื่อมั่นใจว่าข้อมูลครบแล้วจึงเปลี่ยนเป็น `NOT NULL` จริง หลักการทั่วไปคือ**แยกขั้นตอนที่ต้อง lock นาน/scan ทั้งตารางออกเป็นหลาย step เล็ก ๆ** และใช้ `NOT VALID` + `VALIDATE` pattern เพื่อลด lock time ในแต่ละขั้นตอนให้สั้นที่สุด (Part 019, 059, 079)

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 9:</strong> อธิบายความแตกต่างระหว่าง horizontal scaling กับ vertical scaling ของ PostgreSQL พร้อมข้อจำกัดของแต่ละแบบ</summary>

**เฉลย:** **Vertical scaling** คือการเพิ่มทรัพยากรของเครื่องเดิม (CPU, RAM, disk I/O ที่เร็วขึ้น) ข้อดีคือทำได้ง่าย ไม่ต้องเปลี่ยน architecture ของ application เลย แต่มีเพดานจำกัด (เครื่องที่ใหญ่ที่สุดที่ cloud provider มีให้) และค่าใช้จ่ายมักโตแบบไม่เป็นเส้นตรง (เครื่องใหญ่ขึ้นเท่าตัวราคาอาจมากกว่าเท่าตัว) **Horizontal scaling** คือการกระจายโหลดออกไปหลายเครื่อง เช่น read replica (กระจาย read), partitioning (แบ่งข้อมูลภายในเครื่องเดียว), sharding (แบ่งข้อมูลข้ามเครื่องหลายตัว) ข้อดีคือ scale ได้แทบไม่จำกัดในทางทฤษฎี แต่เพิ่มความซับซ้อนมาก: ต้องจัดการ data consistency ข้าม node, cross-shard query/join ทำได้ยากหรือทำไม่ได้เลย, operational overhead สูงขึ้นมาก (deploy, backup, monitoring ต้องทำกับหลาย instance) แนวทางที่แนะนำคือ vertical scale ให้สุดทางที่คุ้มค่าก่อน แล้วค่อยขยับไป horizontal เมื่อจำเป็นจริง ไม่ใช่เริ่มจาก sharding ตั้งแต่ day 1 (Part 065, 075, 094-096)

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 10:</strong> ในการสัมภาษณ์ระดับ Senior DBA มักถูกถามว่า "ถ้า production database CPU พุ่งขึ้น 100% กะทันหัน จะ diagnose อย่างไรใน 5 นาทีแรก" ให้ร่างขั้นตอนที่กระชับและใช้งานได้จริง</summary>

**เฉลย:** ขั้นตอนที่แนะนำใน 5 นาทีแรก (เน้นความเร็วก่อนความละเอียด): (1) รัน `SELECT * FROM pg_stat_activity WHERE state != 'idle' ORDER BY query_start;` เพื่อดูว่ามี query ไหนที่รันค้างนานผิดปกติ หรือมี query จำนวนมากที่เหมือนกันรันพร้อมกัน (อาจเป็นสัญญาณของ connection storm หรือ missing index ที่ทำให้ query ที่ปกติเร็วกลับมา seq scan); (2) เช็คว่ามี query ใหม่ที่เพิ่ง deploy หรือ config เปลี่ยนแปลงล่าสุดหรือไม่ (เชื่อมโยงกับ deployment timeline ล่าสุดของทีม) (3) ถ้าพบ query ตัวเดียวที่กิน CPU สูงชัดเจน ให้รัน `EXPLAIN` (ไม่ต้อง ANALYZE ถ้า production กำลังหนักอยู่แล้ว เพื่อไม่เพิ่มภาระ) ดู plan คร่าว ๆ ว่าเปลี่ยนจาก index scan เป็น seq scan หรือไม่ (อาจเกิดจาก statistics เก่า → `ANALYZE` ตารางนั้นด่วน) (4) ถ้าเป็น query ที่จำเป็นต้อง kill ทันทีเพื่อกู้สถานการณ์ ใช้ `pg_cancel_backend(pid)` (ยกเลิก query แบบนุ่มนวล) หรือ `pg_terminate_backend(pid)` (ตัด connection ทิ้งถ้า cancel ไม่ได้ผล) (5) พิจารณาเช็ค autovacuum ที่กำลังรันอยู่ด้วย `pg_stat_progress_vacuum` เพราะบางครั้ง CPU พุ่งมาจาก vacuum ตารางใหญ่ที่ไม่เกี่ยวกับ query เลย หลังจากบรรเทาสถานการณ์เฉพาะหน้าแล้ว จึงค่อยไปวิเคราะห์ root cause เชิงลึกต่อ (Part 044, 060, 072, 074)

</details>

---

**บทถัดไป:** [Part 102: Open Source Community — การมีส่วนร่วมกับชุมชน PostgreSQL](./part-102-open-source-community.md)
