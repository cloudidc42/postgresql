# Part 062: Point-in-Time Recovery (PITR) และ WAL Archiving

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับมืออาชีพ | Part 062

---

## เป้าหมายการเรียนรู้

เมื่อจบบทนี้ ผู้เรียนจะสามารถ:

- อธิบายแนวคิดของ Write-Ahead Log (WAL) และบทบาทของมันในการทำ Point-in-Time Recovery (PITR)
- อธิบายได้ว่าทำไมองค์กรที่ดูแลระบบฐานข้อมูล e-commerce ระดับ production จำเป็นต้องมี PITR ไม่ใช่แค่ backup แบบ full dump
- ตั้งค่า `wal_level`, `archive_mode`, `archive_command` ใน `postgresql.conf` เพื่อเปิดใช้งาน WAL archiving ได้ถูกต้อง
- ใช้ `pg_basebackup` สร้าง base backup ที่เป็นจุดเริ่มต้นของกระบวนการ PITR
- เขียน `archive_command` สำหรับปลายทางต่าง ๆ ได้แก่ local disk, NFS mount และแนวคิดการ archive ไปยัง S3
- เข้าใจกลไก `recovery.signal` และ `postgresql.auto.conf` ใน PostgreSQL 12 ขึ้นไป และความแตกต่างจาก `recovery.conf` ในเวอร์ชันเก่า
- ตั้งค่า `restore_command` และ `recovery_target_time` เพื่อกู้คืนข้อมูล ณ เวลาที่ต้องการได้อย่างแม่นยำ
- เลือกใช้ `recovery_target_xid` หรือ `recovery_target_name` (restore point) แทนการระบุเวลา เมื่อเหมาะสมกว่า
- ปฏิบัติตามขั้นตอน PITR แบบเต็มรูปแบบ ตั้งแต่ restore base backup, ตั้งค่า recovery, สั่ง start จนถึงการ verify ผลลัพธ์
- จำลองสถานการณ์ข้อมูลถูกลบผิดพลาด (accidental DELETE) บนฐานข้อมูล e-commerce แล้วกู้คืนด้วย PITR ได้จริงด้วยมือของตัวเอง

---

## บริบทของบทนี้

ในบทที่แล้ว (Part 061) เราเรียนรู้การสำรองข้อมูลด้วย `pg_dump`, `pg_dumpall` และ `pg_basebackup` ซึ่งเป็นการสำรองข้อมูล ณ จุดเวลาหนึ่ง (snapshot) — เช่น backup ทุกคืนเที่ยงคืน ถ้าเซิร์ฟเวอร์ล่มตอนบ่ายสามโมง เราจะกู้คืนได้แค่ข้อมูล ณ เที่ยงคืนของคืนก่อนหน้าเท่านั้น ข้อมูลของทั้งวันจะหายไป

บทนี้จะพาไปสู่เทคนิคที่ทรงพลังกว่า นั่นคือ **Point-in-Time Recovery (PITR)** ซึ่งอาศัย WAL (Write-Ahead Log) ที่ PostgreSQL บันทึกไว้อย่างต่อเนื่อง ทำให้เราสามารถกู้คืนฐานข้อมูลกลับไปยัง "จุดเวลาใดก็ได้" ในอดีต ไม่ใช่แค่จุดที่ทำ backup เท่านั้น — ตัวอย่างเช่น กู้คืนไปยังเวลา 14:59:30 น. ซึ่งเป็นวินาทีก่อนที่พนักงานจะรัน `DELETE FROM orders;` โดยไม่ใส่ `WHERE` clause โดยไม่ได้ตั้งใจ

เราจะใช้ฐานข้อมูล e-commerce ที่คุ้นเคยกันมาตลอดหลักสูตรเป็นกรณีศึกษา สมมติว่าเรามีตาราง `customers`, `orders`, `order_items`, `products` และมีเหตุการณ์ข้อมูลสูญหายเกิดขึ้นจริงในบทสุดท้าย (Step 620) เพื่อฝึกกู้คืนแบบครบวงจร

---

## Step 611: WAL (Write-Ahead Log) คืออะไรโดยสรุป

### แนวคิดพื้นฐาน

**Write-Ahead Log (WAL)** คือกลไกหลักที่ PostgreSQL ใช้รับประกันความทนทานของข้อมูล (durability) และความสอดคล้อง (consistency) หลักการสำคัญคือ

> **ทุกการเปลี่ยนแปลงข้อมูล (INSERT, UPDATE, DELETE, การเปลี่ยนโครงสร้างตาราง ฯลฯ) จะต้องถูกบันทึกลง WAL log ก่อน แล้วจึงค่อยเขียนการเปลี่ยนแปลงจริงลง data file (heap/index files) ภายหลัง**

พูดง่าย ๆ คือ PostgreSQL จะไม่มีวันเขียนข้อมูลลงตารางจริงบนดิสก์ก่อนที่จะมีบันทึก "สัญญา" ว่าจะเขียนอะไรลงใน WAL ก่อนเสมอ เหตุผลคือ:

1. **WAL record มีขนาดเล็กและเขียนแบบ sequential (append-only)** ทำให้เขียนได้เร็วกว่าการ flush หน้า data page ทั้งหน้าลงดิสก์แบบสุ่ม (random I/O)
2. เมื่อ WAL record ถูกเขียนและ `fsync` ลงดิสก์แล้ว ถือว่า transaction นั้น "ปลอดภัย" (durable) แม้ว่าการเปลี่ยนแปลงจริงในตาราง (data page) จะยังไม่ถูกเขียนลงดิสก์ก็ตาม เพราะหาก server ล่มกะทันหัน PostgreSQL จะใช้ WAL ในการ **replay** (เล่นซ้ำ) การเปลี่ยนแปลงเหล่านั้นตอน startup เพื่อทำให้ข้อมูลกลับมาสมบูรณ์ (crash recovery)

### WAL segment file

ไฟล์ WAL แต่ละไฟล์เรียกว่า **WAL segment** โดยค่าเริ่มต้นมีขนาด 16 MB ต่อไฟล์ (ปรับได้ตอน `initdb` ด้วย `--wal-segsize`) เก็บอยู่ในไดเรกทอรี `pg_wal/` (เดิมชื่อ `pg_xlog/` ก่อน PostgreSQL 10) ภายใต้ data directory

```bash
$ ls -la /var/lib/postgresql/16/main/pg_wal/
000000010000000000000001
000000010000000000000002
000000010000000000000003
archive_status/
```

ชื่อไฟล์ WAL เป็นเลขฐาน 16 หลัก 24 หลัก แบ่งเป็น 3 ส่วน: **Timeline ID** (8 หลักแรก) + **Log file ID** (8 หลักถัดมา) + **Segment ID** (8 หลักสุดท้าย) เช่น `000000010000000000000001` หมายถึง timeline 1, log 0, segment 1

### ความสัมพันธ์กับ PITR

เนื่องจาก WAL คือบันทึกการเปลี่ยนแปลงข้อมูล **ทุกอย่าง** ตามลำดับเวลา (sequential, ordered) หากเรามี:

1. **Base backup** — สำเนาข้อมูลทั้งหมด ณ จุดเวลาหนึ่ง (checkpoint)
2. **WAL segment files ทั้งหมด** ที่เกิดขึ้นหลังจาก base backup นั้น

เราก็สามารถ "เล่นซ้ำ" (replay) WAL segment เหล่านั้นทีละรายการ เริ่มจาก base backup จนถึงจุดเวลาใดเวลาหนึ่งที่เราต้องการ — นี่คือหัวใจของ Point-in-Time Recovery นั่นเอง

> **หมายเหตุ:** บทนี้จะพูดถึง WAL ในระดับที่จำเป็นสำหรับ PITR เท่านั้น กลไกเชิงลึกของ WAL เช่น โครงสร้าง WAL record, checkpoint, `full_page_writes`, WAL buffer และการปรับจูนประสิทธิภาพ WAL จะอธิบายอย่างละเอียดใน **Part 083** ของหลักสูตรนี้

### ตาราง: สรุปคุณสมบัติของ WAL

| คุณสมบัติ | รายละเอียด |
|---|---|
| ตำแหน่งไฟล์ | `$PGDATA/pg_wal/` |
| ขนาดไฟล์เริ่มต้น | 16 MB ต่อ segment |
| รูปแบบชื่อไฟล์ | Timeline (8 hex) + LogId (8 hex) + SegmentId (8 hex) |
| ลำดับการเขียน | Sequential / append-only |
| วัตถุประสงค์หลัก | Crash recovery, Replication, PITR |
| พารามิเตอร์ควบคุมระดับ | `wal_level` (minimal / replica / logical) |

---

## Step 612: ทำไมต้องมี PITR

### ข้อจำกัดของ backup แบบ snapshot

การสำรองข้อมูลแบบ `pg_dump` หรือ `pg_basebackup` เพียงอย่างเดียว (ไม่มี WAL archiving) ให้เราได้แค่ "จุดคงที่" (fixed point) ในอดีต สมมติสถานการณ์:

```
23:00 - pg_basebackup ทำงานเสร็จ (backup ล่าสุด)
...
09:15 - พนักงานฝ่ายบัญชีรัน DELETE FROM order_items; โดยลืมใส่ WHERE
09:16 - พบว่าข้อมูล order_items หายทั้งตาราง (มีผลกระทบต่อคำสั่งซื้อลูกค้ากว่าล้านรายการ)
```

ถ้ามีแค่ backup ตอน 23:00 การกู้คืนจะทำให้เราสูญเสียข้อมูลตั้งแต่ 23:00 ถึง 09:15 ไปทั้งหมด (คำสั่งซื้อใหม่ การชำระเงิน การอัปเดตสต็อกสินค้า ฯลฯ ของทั้งคืนและเช้าวันนั้น) ซึ่งสำหรับระบบ e-commerce ที่มีธุรกรรมเกิดขึ้นตลอดเวลา ถือเป็นความเสียหายที่ยอมรับไม่ได้

### PITR แก้ปัญหานี้อย่างไร

ด้วย PITR เราสามารถกู้คืนฐานข้อมูลไปยัง **วินาทีใดก็ได้** ระหว่างจุดที่ทำ base backup กับจุดปัจจุบัน (หรือจนถึงจุดที่ WAL ถูก archive ล่าสุด) ตัวอย่างสถานการณ์เดียวกัน:

```
23:00 - pg_basebackup (base backup)
...     - WAL segments ถูก archive ต่อเนื่องทุกครั้งที่ segment เต็มหรือ switch
09:14:58 - จุดสุดท้ายก่อนเกิดเหตุ DELETE ผิดพลาด
09:15:00 - DELETE FROM order_items; (ไม่มี WHERE) ← transaction นี้ถูกบันทึกใน WAL ด้วย
09:16 - พบปัญหา
```

เราสามารถสั่ง PostgreSQL ให้:

1. Restore base backup ของ 23:00
2. Replay WAL ทั้งหมดตั้งแต่ 23:00 มาเรื่อย ๆ
3. **หยุด replay ที่เวลา 09:14:59 น.** (ก่อน transaction DELETE ที่ผิดพลาดเพียงเสี้ยววินาที)

ผลลัพธ์คือฐานข้อมูลกลับมาสมบูรณ์ ณ เวลา 09:14:59 น. — มีข้อมูลคำสั่งซื้อของทั้งคืนและเช้าครบถ้วน และ **ไม่มี** DELETE ที่ผิดพลาดปนอยู่เลย นี่คือพลังของ PITR ที่ backup แบบ snapshot ทำไม่ได้

### กรณีการใช้งานทั่วไปของ PITR

| สถานการณ์ | ตัวอย่าง |
|---|---|
| Human error | `DELETE`/`UPDATE`/`DROP TABLE` โดยไม่ตั้งใจหรือไม่มี `WHERE` |
| Application bug | โค้ด batch job มี logic ผิดพลาด เขียนข้อมูลเสียหายจำนวนมาก |
| Malicious action | มีผู้ไม่หวังดีเข้าถึงฐานข้อมูลและทำลายข้อมูล |
| Disaster recovery | Data center ล่ม ต้องกู้คืนไปยังเครื่องใหม่ที่จุดเวลาล่าสุดที่มี WAL |
| Data audit / debugging | ต้องการดูสถานะฐานข้อมูล ณ เวลาใดเวลาหนึ่งในอดีตเพื่อตรวจสอบ (ทำใน environment แยก) |
| Compliance | องค์กรที่ต้องรักษา RPO (Recovery Point Objective) ให้ต่ำมาก เช่นไม่เกินไม่กี่วินาที |

### RPO และ RTO

เมื่อพูดถึง PITR จำเป็นต้องเข้าใจ 2 คำศัพท์สำคัญ:

- **RPO (Recovery Point Objective)** — ยอมรับการสูญเสียข้อมูลได้มากที่สุดกี่นาที/วินาที นับจากจุดเกิดเหตุย้อนกลับไป การทำ WAL archiving แบบต่อเนื่อง (continuous archiving) ทำให้ RPO ต่ำมาก (อาจเป็นวินาทีระดับเดียวกับ `archive_timeout`)
- **RTO (Recovery Time Objective)** — ต้องกู้คืนระบบให้กลับมาใช้งานได้ภายในเวลาเท่าไหร่ PITR ใช้เวลานานกว่าการ restore snapshot ธรรมดา เพราะต้อง replay WAL จำนวนมาก แต่แลกมาด้วย RPO ที่ดีกว่ามาก

> **ข้อคิดสำคัญ:** ระบบ e-commerce ระดับ production ที่ดี ควรมีทั้ง backup แบบ full (pg_basebackup รายวัน/รายสัปดาห์) **ร่วมกับ** WAL archiving แบบต่อเนื่อง เพื่อให้ได้ RPO ที่ต่ำที่สุดเท่าที่จะเป็นไปได้ ไม่ใช่เลือกใช้อย่างใดอย่างหนึ่ง

---

## Step 613: การเปิดใช้งาน WAL Archiving

การเปิดใช้ WAL Archiving ต้องตั้งค่า 3 พารามิเตอร์หลักใน `postgresql.conf` ได้แก่ `wal_level`, `archive_mode` และ `archive_command`

### 1. wal_level

พารามิเตอร์นี้กำหนดว่า WAL จะบันทึกข้อมูลละเอียดแค่ไหน มี 3 ค่าใน PostgreSQL 16/17:

| ค่า | คำอธิบาย |
|---|---|
| `minimal` | บันทึกข้อมูลขั้นต่ำสุดสำหรับ crash recovery เท่านั้น **ไม่รองรับ** WAL archiving หรือ replication |
| `replica` | (ค่าเริ่มต้น) บันทึกข้อมูลเพียงพอสำหรับ WAL archiving และ streaming replication รวมถึง read-only queries บน standby |
| `logical` | เพิ่มข้อมูลสำหรับ logical decoding/logical replication (จะกล่าวถึงใน Part 084) |

สำหรับ PITR เราต้องตั้งค่าอย่างน้อย `replica` (ซึ่งเป็นค่า default อยู่แล้วใน PostgreSQL สมัยใหม่ แต่ควรตรวจสอบให้แน่ใจ):

```ini
# postgresql.conf
wal_level = replica
```

> **หมายเหตุ:** การเปลี่ยน `wal_level` ต้อง **restart PostgreSQL service** เนื่องจากเป็นพารามิเตอร์ประเภท `postmaster` context (ไม่สามารถ reload ได้เฉย ๆ)

### 2. archive_mode

เปิดใช้งานฟีเจอร์ archiving:

```ini
archive_mode = on
```

ค่าที่เป็นไปได้:

| ค่า | ความหมาย |
|---|---|
| `off` | ปิดการ archive WAL (ค่าเริ่มต้น) |
| `on` | เปิด archiving เมื่อ server เป็น primary (read-write) |
| `always` | archive แม้ในขณะที่ server เป็น standby (ใช้ในสถาปัตยกรรม cascading replication บางกรณี) |

### 3. archive_command

คำสั่ง shell ที่ PostgreSQL จะเรียกใช้ทุกครั้งที่ WAL segment หนึ่งไฟล์ "เสร็จสมบูรณ์" (完成) และพร้อมถูกคัดลอกไปเก็บที่ archive storage:

```ini
archive_command = 'test ! -f /mnt/wal_archive/%f && cp %p /mnt/wal_archive/%f'
```

อธิบาย placeholder:

- `%p` — full path ของไฟล์ WAL ต้นทางที่จะถูก archive (relative ต่อ current working directory ของ server หรือ absolute path)
- `%f` — เฉพาะชื่อไฟล์ (filename) ของ WAL segment

**ข้อสำคัญมาก:** `archive_command` ต้อง return exit code `0` เมื่อสำเร็จเท่านั้น หาก return ค่าอื่น PostgreSQL จะถือว่า archive ล้มเหลว และจะ **พยายามเรียกซ้ำเรื่อย ๆ** จนกว่าจะสำเร็จ (WAL segment จะไม่ถูกลบออกจาก `pg_wal/` จนกว่าจะ archive สำเร็จ ซึ่งอาจทำให้ดิสก์เต็มได้หากปล่อยไว้นาน)

เหตุผลที่ต้องมี `test ! -f ... &&` คือเพื่อป้องกันไม่ให้ archive_command เขียนทับไฟล์ archive เดิมที่มีอยู่แล้วโดยไม่ตั้งใจ (ซึ่งอาจเกิดจาก timeline switch หรือการรัน server ผิดพลาด) — นี่เป็น best practice มาตรฐานตาม PostgreSQL documentation

### ตัวอย่างการตั้งค่าแบบสมบูรณ์

```ini
# =========================================================
# postgresql.conf - WAL Archiving Configuration
# =========================================================

# --- WAL level (จำเป็นสำหรับ archiving/replication) ---
wal_level = replica

# --- เปิดใช้งาน archiving ---
archive_mode = on

# --- คำสั่ง archive แต่ละ WAL segment ---
archive_command = 'test ! -f /mnt/wal_archive/%f && cp %p /mnt/wal_archive/%f'

# --- (แนะนำ) บังคับให้ WAL switch อย่างน้อยทุก 5 นาที แม้ traffic น้อย ---
archive_timeout = 300

# --- (แนะนำ) จำนวน WAL sender process สูงสุด สำหรับ replication/backup tools ---
max_wal_senders = 10

# --- (แนะนำ) เก็บ WAL ไว้ใน pg_wal อย่างน้อยเท่านี้ก่อนถูก recycle ---
wal_keep_size = 1GB
```

### archive_timeout คืออะไร

โดยปกติ WAL segment จะถูก archive ก็ต่อเมื่อไฟล์นั้น "เต็ม" (16 MB) เท่านั้น หากระบบมี traffic น้อยในบางช่วงเวลา (เช่น ตี 3 เช้า) อาจต้องรอนานกว่าจะมี WAL segment ใหม่ให้ archive ทำให้ RPO แย่ลง การตั้ง `archive_timeout = 300` (หน่วยเป็นวินาที) จะบังคับให้ PostgreSQL สลับ (switch) ไปยัง WAL segment ใหม่ทุก ๆ 5 นาทีเป็นอย่างน้อย แม้ segment จะยังไม่เต็ม เพื่อรับประกันว่าข้อมูลจะถูก archive อย่างสม่ำเสมอ

### การนำการตั้งค่าไปใช้งาน

```bash
# แก้ไขไฟล์ postgresql.conf
$ sudo vi /etc/postgresql/16/main/postgresql.conf

# ตรวจสอบ syntax ก่อน restart (PostgreSQL 16+)
$ sudo -u postgres /usr/lib/postgresql/16/bin/postgres \
    --config-file=/etc/postgresql/16/main/postgresql.conf -C wal_level

# สร้างไดเรกทอรีปลายทางสำหรับ archive และกำหนดสิทธิ์
$ sudo mkdir -p /mnt/wal_archive
$ sudo chown postgres:postgres /mnt/wal_archive
$ sudo chmod 700 /mnt/wal_archive

# เนื่องจาก wal_level เป็น postmaster context ต้อง restart ไม่ใช่แค่ reload
$ sudo systemctl restart postgresql@16-main
```

### ตรวจสอบว่า archiving ทำงานถูกต้อง

```sql
-- ตรวจสอบค่าพารามิเตอร์
SHOW wal_level;
SHOW archive_mode;
SHOW archive_command;

-- บังคับ switch WAL segment เพื่อทดสอบว่า archive_command ทำงาน
SELECT pg_switch_wal();

-- ดูสถิติการ archive (สำเร็จ/ล้มเหลวกี่ครั้ง, เวลาล่าสุด)
SELECT * FROM pg_stat_archiver;
```

ผลลัพธ์ตัวอย่างจาก `pg_stat_archiver`:

```
 archived_count | last_archived_wal      | last_archived_time    | failed_count | last_failed_wal | last_failed_time | stats_reset
----------------+-------------------------+------------------------+--------------+------------------+-------------------+-------------
             42 | 000000010000000000000029 | 2026-09-25 09:10:03+07 |            0 |                  |                   | 2026-09-20 00:00:00+07
```

หาก `failed_count` เพิ่มขึ้นเรื่อย ๆ แสดงว่า `archive_command` มีปัญหา (เช่น permission ผิด, ปลายทางเต็ม, path ไม่ถูกต้อง) ต้องตรวจสอบ log ของ PostgreSQL (`postgresql-*.log`) ทันที เพราะ WAL segments จะสะสมค้างอยู่ใน `pg_wal/` และอาจทำให้ดิสก์เต็มจนฐานข้อมูลหยุดทำงานได้

---

## Step 614: Base Backup สำหรับ PITR

### ทบทวนจาก Part 061

ใน Part 061 เราได้เรียนรู้การใช้ `pg_basebackup` เพื่อสำรองข้อมูลทั้ง data directory ของ PostgreSQL แบบ physical (ไม่ใช่ logical แบบ `pg_dump`) ในบริบทของ PITR **base backup คือจุดตั้งต้น (starting point)** ที่เราจะเอา WAL มา replay ทับต่อ ดังนั้น base backup จึงเป็นองค์ประกอบที่ขาดไม่ได้ของ PITR strategy

ความสัมพันธ์คือ:

```
[Base Backup ณ เวลา T0] + [WAL segments ตั้งแต่ T0 ถึง T-target] = ฐานข้อมูล ณ เวลา T-target
```

### การสร้าง base backup ด้วย pg_basebackup

```bash
# สร้าง base backup แบบ plain format พร้อม WAL ที่จำเป็น (streaming)
$ sudo -u postgres pg_basebackup \
    -D /var/backups/pgsql/base/$(date +%Y%m%d_%H%M%S) \
    -Fp \
    -Xs \
    -P \
    -c fast \
    -h localhost \
    -U replicator

# -D : ปลายทางที่จะเก็บ base backup
# -Fp: format plain (คัดลอกไฟล์ตรง ๆ ไม่ใช่ tar)
# -Xs: streaming WAL ระหว่าง backup พร้อมกันไปด้วย (แนะนำเสมอสำหรับ PITR)
# -P : แสดง progress
# -c fast: บังคับ checkpoint ทันที ไม่ต้องรอ checkpoint ตามรอบ (เริ่ม backup เร็วขึ้น)
```

ตัวอย่าง output:

```
30005/30005 kB (100%), 1/1 tablespace
```

### ทำไมต้องใช้ -X (WAL streaming) ตอน base backup

พารามิเตอร์ `-X` (หรือ `--wal-method`) มี 3 ค่า: `none`, `fetch`, `stream`

- `none` — ไม่รวม WAL ใด ๆ (ต้องพึ่ง `archive_command`/`restore_command` เพียงอย่างเดียวตอน recovery) — เหมาะกับ workflow ที่มั่นใจว่า WAL archive สมบูรณ์อยู่แล้ว
- `fetch` — ดึง WAL ที่จำเป็นหลัง backup เสร็จ (มีความเสี่ยงถ้า WAL segment ถูก recycle ไปก่อน)
- `stream` (`-Xs`) — เปิด connection คู่ขนานเพื่อ stream WAL ที่เกิดขึ้น **ระหว่าง** การ backup กำลังทำงานอยู่ ทำให้ base backup ที่ได้ "สมบูรณ์ในตัวเอง" (self-contained) ไม่ต้องพึ่งพา archive สำหรับช่วงเวลานั้น — **แนะนำอย่างยิ่งสำหรับ production**

### โครงสร้างไฟล์ base backup ที่ได้

```
/var/backups/pgsql/base/20260925_090000/
├── PG_VERSION
├── backup_label
├── base/
│   ├── 1/
│   ├── 16384/          <- OID ของฐานข้อมูล ecommerce
│   └── ...
├── global/
├── pg_wal/
│   └── 000000010000000000000030   <- WAL segments ที่ stream มาระหว่าง backup
├── pg_tblspc/
├── postgresql.conf
├── postgresql.auto.conf
├── pg_hba.conf
└── backup_manifest
```

ไฟล์สำคัญที่ต้องรู้จัก:

- **`backup_label`** — บันทึกว่า backup เริ่มที่ WAL location ใด (`START WAL LOCATION`) ซึ่ง PostgreSQL จะใช้อ้างอิงตอน recovery เพื่อรู้ว่าต้องเริ่ม replay WAL จากจุดไหน
- **`backup_manifest`** — (ตั้งแต่ PostgreSQL 13) เก็บ checksum ของทุกไฟล์ใน backup ใช้ตรวจสอบความสมบูรณ์ด้วย `pg_verifybackup`

### ตรวจสอบความสมบูรณ์ของ base backup

```bash
$ sudo -u postgres pg_verifybackup /var/backups/pgsql/base/20260925_090000

backup successfully verified
```

### ความถี่ในการทำ base backup

การทำ base backup บ่อยเกินไปสิ้นเปลือง I/O และพื้นที่จัดเก็บ แต่ถ้าห่างเกินไปจะทำให้ recovery time (RTO) นานขึ้น เพราะต้อง replay WAL จำนวนมากกว่า แนวทางทั่วไป:

| ขนาดฐานข้อมูล | ความถี่ base backup แนะนำ |
|---|---|
| เล็ก (< 10 GB) | ทุกวัน |
| กลาง (10-200 GB) | ทุก 1-3 วัน |
| ใหญ่ (> 200 GB) | ทุกสัปดาห์ + พิจารณา incremental backup (Part 061 อธิบาย `pg_basebackup` incremental ใน PG 17) |

> ระบบ e-commerce ของเราควรทำ base backup อย่างน้อยวันละ 1 ครั้ง (เช่น ตอนตี 2 ที่ traffic ต่ำสุด) ร่วมกับ WAL archiving ต่อเนื่องตลอด 24 ชั่วโมง

---

## Step 615: การตั้งค่า archive_command ตัวอย่างจริง

ในสภาพแวดล้อมจริง ปลายทางของ WAL archive มีได้หลายแบบ ขึ้นอยู่กับโครงสร้างพื้นฐานขององค์กร มาดูตัวอย่างที่ใช้งานได้จริง 3 รูปแบบ

### 1. Archive ไปยัง Local Disk (ดิสก์แยกจาก data directory)

เหมาะสำหรับระบบขนาดเล็กถึงกลาง หรือใช้เป็นชั้นป้องกันแรกก่อนจะ sync ไปที่อื่นต่อ **ข้อสำคัญ:** ต้องเป็นดิสก์คนละก้อนกับ data directory เพื่อไม่ให้ปัญหาฮาร์ดแวร์เดียวทำลายทั้ง data และ archive พร้อมกัน

```ini
archive_command = 'test ! -f /mnt/wal_archive/%f && cp %p /mnt/wal_archive/%f'
```

เวอร์ชันที่ทนทานขึ้น (ตรวจสอบ exit code ของแต่ละคำสั่งอย่างชัดเจน):

```bash
#!/bin/bash
# /usr/local/bin/pg_archive_local.sh
set -euo pipefail

WAL_PATH="$1"    # %p
WAL_FILE="$2"    # %f
ARCHIVE_DIR="/mnt/wal_archive"

DEST="${ARCHIVE_DIR}/${WAL_FILE}"

if [ -f "$DEST" ]; then
    echo "WARNING: ${DEST} already exists, refusing to overwrite" >&2
    exit 1
fi

cp "$WAL_PATH" "$DEST"
sync "$DEST"
exit 0
```

```ini
archive_command = '/usr/local/bin/pg_archive_local.sh %p %f'
```

### 2. Archive ไปยัง NFS Mount

หลายองค์กรใช้ NFS share เป็นปลายทาง archive เพราะสามารถ mount เข้าได้จากหลายเซิร์ฟเวอร์ (เช่น production และ DR site) สิ่งที่ต้องระวังคือ **NFS latency** และปัญหา network partition ที่อาจทำให้ archive ค้าง

```ini
archive_command = 'cp %p /mnt/nfs/pg_wal_archive/%f && sync'
```

ตัวอย่าง `/etc/fstab` สำหรับ mount NFS:

```
nfs-backup-server:/exports/pg_wal_archive  /mnt/nfs/pg_wal_archive  nfs  rw,hard,intr,timeo=30  0  0
```

> **แนะนำ:** ใช้ตัวเลือก `hard` (ไม่ใช่ `soft`) สำหรับ NFS mount ที่ใช้เก็บ WAL archive เพื่อให้แน่ใจว่าเมื่อ network มีปัญหาชั่วคราว คำสั่ง `archive_command` จะรอ (block) แทนที่จะ return error และทำให้ PostgreSQL คิดว่า archive สำเร็จทั้งที่จริง ๆ ไม่สำเร็จ

### 3. Archive ไปยัง Object Storage เช่น Amazon S3 (แนวคิด)

สำหรับ production ระดับ enterprise นิยม archive WAL ไปยัง object storage เช่น S3, Google Cloud Storage หรือ MinIO เพราะมีความทนทานสูง (durability) และ scale ได้ไม่จำกัด แนวคิดคือใช้เครื่องมือ เช่น `wal-g` หรือ `pgBackRest` (ซึ่งจะกล่าวถึงรายละเอียดใน Part 063-064 และบทที่เกี่ยวกับ backup tools ขั้นสูง) หรือเขียน script ห่อ AWS CLI เอง:

```bash
#!/bin/bash
# /usr/local/bin/pg_archive_s3.sh
set -euo pipefail

WAL_PATH="$1"
WAL_FILE="$2"
S3_BUCKET="s3://ecommerce-pg-wal-archive/prod"

aws s3 cp "$WAL_PATH" "${S3_BUCKET}/${WAL_FILE}" \
    --only-show-errors \
    --storage-class STANDARD_IA

exit $?
```

```ini
archive_command = '/usr/local/bin/pg_archive_s3.sh %p %f'
```

**ข้อควรพิจารณาเมื่อใช้ S3 (หรือ object storage อื่น ๆ) เป็นปลายทาง archive:**

| ประเด็น | คำอธิบาย |
|---|---|
| Latency | การอัปโหลดผ่านเครือข่ายช้ากว่า local disk มาก อาจทำให้ WAL ค้างสะสมใน `pg_wal/` ถ้า traffic เขียนข้อมูลสูงมาก |
| Cost | ค่าใช้จ่าย API request (PUT/GET) และค่าจัดเก็บ ควรตั้ง lifecycle policy ลบ WAL เก่าที่เกิน retention period |
| Throughput | ควรใช้เครื่องมือที่รองรับ parallel upload/compression เช่น `wal-g` หรือ `pgBackRest` แทนการเขียน script เองสำหรับ production จริง เพราะมี retry logic, compression, encryption ในตัว |
| IAM Permission | ต้องจำกัดสิทธิ์ IAM ให้ archive ได้แค่ `PutObject`/`GetObject` เฉพาะ bucket/path ที่เกี่ยวข้อง ไม่ควรให้สิทธิ์ delete แบบ unrestricted |

> ในบทนี้เราจะใช้ **local disk** เป็นตัวอย่างหลักในการฝึกปฏิบัติ (Step 620) เพื่อให้ทำตามได้ง่ายโดยไม่ต้องพึ่งพา cloud account แต่แนวคิดและขั้นตอน recovery จะเหมือนกันทุกประการไม่ว่าปลายทาง archive จะเป็นอะไร

---

## Step 616: recovery.signal และ postgresql.auto.conf (PostgreSQL 12+)

### การเปลี่ยนแปลงสำคัญใน PostgreSQL 12

ก่อน PostgreSQL 12 การเข้าสู่ recovery mode ต้องสร้างไฟล์ `recovery.conf` ในลักษณะไฟล์ config แยกต่างหากที่ใส่ `restore_command`, `recovery_target_time` และค่าอื่น ๆ ทั้งหมดไว้ด้วยกัน

**ตั้งแต่ PostgreSQL 12 เป็นต้นไป (รวมถึง PostgreSQL 16 และ 17 ที่หลักสูตรนี้ใช้) ไฟล์ `recovery.conf` ถูก "ยกเลิก" ไปโดยสิ้นเชิง** หาก PostgreSQL เจอไฟล์ `recovery.conf` ใน data directory จะ **ไม่ยอม start** และแจ้ง error ทันที

แทนที่ด้วยกลไกใหม่ 2 ส่วน:

1. **Signal file** — ไฟล์เปล่า ๆ (empty file) ที่บอก PostgreSQL ว่าต้องเข้าสู่โหมดไหนตอน startup
2. **Recovery parameters** — ย้ายไปอยู่รวมกับพารามิเตอร์ config ทั่วไป ใส่ได้ทั้งใน `postgresql.conf` หรือ `postgresql.auto.conf`

### Signal files ที่เกี่ยวข้อง

| ไฟล์ | ใช้เมื่อ |
|---|---|
| `recovery.signal` | ต้องการเข้าสู่ **archive recovery mode** (เช่น การทำ PITR) — server จะ replay WAL จนถึง target ที่กำหนด แล้ว **promote เป็น primary** ปกติ (เว้นแต่ตั้ง `pause_at_recovery_target`) |
| `standby.signal` | ต้องการ start เป็น **standby server** สำหรับ streaming replication (server จะรอรับ WAL ต่อเนื่องและไม่ promote เอง จนกว่าจะสั่ง `pg_ctl promote`) — รายละเอียดเต็มใน Part 063 |

**ข้อควรระวัง:** ห้ามมีทั้ง `recovery.signal` และ `standby.signal` พร้อมกันใน data directory เดียว หากมีทั้งคู่ PostgreSQL จะให้ความสำคัญกับ `standby.signal` (คือ server จะทำงานเป็น standby) — สำหรับ PITR ในบทนี้เราใช้แค่ `recovery.signal`

### การสร้าง signal file

```bash
$ sudo -u postgres touch /var/lib/postgresql/16/main/recovery.signal
```

เพียงเท่านี้ ไฟล์นี้เป็นไฟล์เปล่า ไม่ต้องมีเนื้อหาใด ๆ ข้างใน — PostgreSQL เช็คแค่ "การมีอยู่" ของไฟล์เท่านั้น เมื่อ recovery เสร็จสมบูรณ์แล้ว **PostgreSQL จะลบไฟล์นี้ทิ้งโดยอัตโนมัติ** (rename เป็น หรือลบไปเลยแล้วแต่เวอร์ชัน) เพื่อไม่ให้ recovery เกิดซ้ำตอน restart ครั้งถัดไป

### พารามิเตอร์ recovery ย้ายไปไว้ที่ไหน

พารามิเตอร์ recovery เช่น `restore_command`, `recovery_target_time` ฯลฯ สามารถใส่ได้ใน:

- `postgresql.conf` — ไฟล์ config หลัก แก้ไขด้วยมือได้ตามปกติ
- `postgresql.auto.conf` — ไฟล์ที่ PostgreSQL ใช้เก็บค่าที่ถูกตั้งผ่านคำสั่ง `ALTER SYSTEM SET ...` (มี priority สูงกว่า `postgresql.conf` เสมอ เพราะถูกอ่านทีหลัง)

ในทางปฏิบัติสำหรับงาน PITR เรามักจะสร้างหรือแก้ไฟล์ `postgresql.auto.conf` ของ data directory ที่ restore มาโดยตรง หรือเขียนค่าลง `postgresql.conf` ก็ได้ทั้งสองแบบใช้งานได้จริง แต่นิยมแยกใส่ `postgresql.auto.conf` เพื่อไม่ปนกับ config การทำงานปกติ

```bash
# ตัวอย่างการเขียน recovery parameters ลง postgresql.auto.conf โดยตรง
$ sudo -u postgres bash -c "cat >> /var/lib/postgresql/16/main/postgresql.auto.conf" <<'EOF'
restore_command = 'cp /mnt/wal_archive/%f %p'
recovery_target_time = '2026-09-25 09:14:59+07'
recovery_target_action = 'promote'
EOF
```

### ลำดับการอ่านไฟล์ config ที่เกี่ยวข้องกับ recovery

```
1. postgresql.conf              (ค่าพื้นฐานทั่วไป)
2. postgresql.auto.conf         (ค่าจาก ALTER SYSTEM หรือใส่เองสำหรับ recovery — override ค่าจาก postgresql.conf)
3. ตรวจสอบว่ามี recovery.signal หรือ standby.signal หรือไม่
4. ถ้ามี recovery.signal → เข้าสู่ archive recovery mode
```

### ตารางเปรียบเทียบ: PostgreSQL รุ่นเก่า vs PostgreSQL 12+

| หัวข้อ | ก่อน PostgreSQL 12 | PostgreSQL 12 ขึ้นไป (16/17) |
|---|---|---|
| ไฟล์บอกโหมด recovery | `recovery.conf` (ไฟล์เดียว รวมทุกอย่าง) | `recovery.signal` หรือ `standby.signal` (ไฟล์เปล่า) |
| พารามิเตอร์ recovery | อยู่ใน `recovery.conf` | อยู่ใน `postgresql.conf` / `postgresql.auto.conf` |
| ถ้ามีไฟล์ `recovery.conf` | ทำงานปกติ | **PostgreSQL ปฏิเสธการ start ทันที** (error) |
| การจบ recovery | ไฟล์ `recovery.conf` ถูก rename เป็น `recovery.done` | ไฟล์ signal ถูกลบออกไปเลย |

> **คำเตือนสำหรับผู้ที่คุ้นเคยกับเอกสารเก่า:** บทความหรือบทเรียนออนไลน์จำนวนมากที่เขียนก่อนปี 2019 ยังอ้างอิง `recovery.conf` อยู่ ถ้าเจอเนื้อหาลักษณะนี้ ให้เข้าใจว่าเป็นวิธีการของ PostgreSQL เวอร์ชัน 11 หรือต่ำกว่าเท่านั้น สำหรับ PostgreSQL 16/17 ที่หลักสูตรนี้สอน ต้องใช้ `recovery.signal` เสมอ

---

## Step 617: restore_command และ recovery_target_time

### restore_command

`restore_command` คือคำสั่งย้อนกลับของ `archive_command` — ในขณะที่ `archive_command` ใช้ **คัดลอก WAL segment ออกไปเก็บ** ที่ archive storage, `restore_command` ใช้ **ดึง WAL segment กลับมา** จาก archive storage เข้าสู่ `pg_wal/` ระหว่างกระบวนการ recovery

```ini
restore_command = 'cp /mnt/wal_archive/%f %p'
```

Placeholder มีความหมายเหมือนเดิม:

- `%f` — ชื่อไฟล์ WAL segment ที่ PostgreSQL ต้องการ (ระบุโดย recovery process)
- `%p` — path ปลายทางใน `pg_wal/` ที่ต้องการให้คัดลอกไฟล์ไปวาง

`restore_command` จะถูกเรียกซ้ำ ๆ ทีละไฟล์ WAL ไปเรื่อย ๆ จนกว่า:

1. จะถึงจุด `recovery_target_time` (หรือ target อื่นที่กำหนด) หรือ
2. ไม่พบไฟล์ WAL ถัดไปใน archive แล้ว (แปลว่า replay มาถึง WAL ล่าสุดที่มี)

**ข้อสำคัญ:** `restore_command` ต้อง return exit code ที่ไม่ใช่ 0 เมื่อไม่พบไฟล์ (เพื่อบอก PostgreSQL ว่า "หมดแล้ว ไม่มี WAL segment นี้") ซึ่งคำสั่ง `cp` มาตรฐานทำแบบนี้อยู่แล้วโดยอัตโนมัติเมื่อไฟล์ต้นทางไม่มีอยู่จริง

### recovery_target_time

พารามิเตอร์นี้กำหนด **จุดเวลาที่ต้องการให้ recovery หยุด replay WAL** — คือหัวใจของ PITR แบบระบุเวลา

```ini
recovery_target_time = '2026-09-25 09:14:59+07'
```

รูปแบบเวลาต้องเป็น timestamp ที่ PostgreSQL เข้าใจได้ (รองรับ timezone offset ได้ด้วย) เมื่อ replay WAL มาถึง transaction ที่ commit หลังเวลานี้ PostgreSQL จะหยุดทันที และ **ไม่ apply transaction ที่เกิดหลังจากเวลานี้**

### recovery_target_inclusive

ควบคุมว่า transaction ที่ commit **พอดี** ณ `recovery_target_time` จะถูกรวมด้วยหรือไม่:

```ini
recovery_target_inclusive = false   # ค่าเริ่มต้นคือ true
```

- `true` (default) — รวม transaction ที่ commit ตรงเวลานั้นพอดีด้วย
- `false` — หยุด **ก่อน** transaction นั้น (ไม่รวม) — มีประโยชน์เมื่อรู้เวลาที่แน่นอนของ transaction ที่ผิดพลาด และต้องการกู้คืนไปยังจุด "ก่อนหน้า" transaction นั้นเป๊ะ ๆ

### recovery_target_action

กำหนดว่าเมื่อ recovery มาถึง target แล้ว PostgreSQL ควรทำอะไรต่อ:

```ini
recovery_target_action = 'promote'
```

| ค่า | พฤติกรรม |
|---|---|
| `pause` | (ค่าเริ่มต้น) หยุด replay แล้ว **หยุดรอ** อยู่ในสถานะ paused ให้ DBA ตรวจสอบข้อมูลก่อน จากนั้นค่อยสั่ง `pg_wal_replay_resume()` หรือ promote ด้วยมือ |
| `promote` | เมื่อถึง target แล้ว promote เป็น read-write server ทันทีโดยอัตโนมัติ |
| `shutdown` | shutdown server ทันทีเมื่อถึง target (เหมาะสำหรับ script อัตโนมัติที่ต้องการตรวจสอบไฟล์ก่อน promote เอง) |

> **แนวทางที่แนะนำสำหรับ production:** ใช้ `recovery_target_action = 'pause'` (หรือปล่อยเป็นค่า default) เพื่อให้มีโอกาส **ตรวจสอบข้อมูลก่อน** ว่ากู้คืนมาถูกจุดจริงหรือไม่ ก่อนจะ promote ให้ระบบกลับมารับงานจริง เพราะการ promote คือจุดที่ **ไม่สามารถย้อนกลับได้** (server จะเริ่ม WAL timeline ใหม่)

### ตัวอย่างการตั้งค่าที่สมบูรณ์สำหรับ PITR ตามเวลา

```ini
# postgresql.auto.conf (หรือ postgresql.conf) ของ data directory ที่ restore มา
restore_command = 'cp /mnt/wal_archive/%f %p'
recovery_target_time = '2026-09-25 09:14:59+07'
recovery_target_inclusive = false
recovery_target_action = 'pause'
```

### ตรวจสอบสถานะ recovery ระหว่างที่ paused

```sql
-- เช็คว่ากำลังอยู่ในสถานะ recovery หรือไม่
SELECT pg_is_in_recovery();

-- เช็คว่า recovery ถูก pause อยู่หรือไม่ (เมื่อ recovery_target_action = 'pause')
SELECT pg_get_wal_replay_pause_state();

-- ดูตำแหน่ง WAL ล่าสุดที่ replay ไปแล้ว
SELECT pg_last_wal_replay_lsn();
SELECT pg_last_xact_replay_timestamp();
```

เมื่อตรวจสอบข้อมูลแล้วพบว่าถูกต้อง สามารถสั่งให้ replay ทำงานต่อ (resume) หรือ promote ได้:

```sql
-- ให้ replay ทำงานต่อจนกว่าจะหมด WAL ที่มี (ถ้าต้องการเลื่อน target ออกไป)
SELECT pg_wal_replay_resume();
```

```bash
# หรือ promote ให้เป็น read-write server ทันที (คำสั่งระดับ OS)
$ sudo -u postgres pg_ctl promote -D /var/lib/postgresql/16/main
```

---

## Step 618: recovery_target_xid, recovery_target_name (restore point)

การระบุเวลา (`recovery_target_time`) เป็นวิธีที่ใช้บ่อยที่สุด แต่ในบางสถานการณ์ การระบุด้วยวิธีอื่นแม่นยำและปลอดภัยกว่า PostgreSQL มี recovery target อีก 3 แบบให้เลือกใช้แทน (หรือร่วมกับ) เวลา

### 1. recovery_target_xid — ระบุด้วย Transaction ID

ใช้เมื่อเรารู้ **transaction ID ที่แน่ชัด** ของ transaction ที่ทำให้เกิดปัญหา (เช่น ดึงมาจาก log หรือจากการตรวจสอบด้วย `pg_waldump`)

```ini
recovery_target_xid = '4832917'
recovery_target_inclusive = false
```

วิธีหา XID ของ transaction ที่ต้องการ เช่น ค้นจาก PostgreSQL log ที่มักบันทึก XID ของแต่ละ statement ไว้ (ถ้าตั้งค่า `log_line_prefix` ให้รวม `%x`):

```ini
# ตัวอย่างการตั้งค่า log_line_prefix ให้แสดง XID (ตั้งไว้ล่วงหน้าก่อนเกิดเหตุ)
log_line_prefix = '%m [%p] user=%u,db=%d,xid=%x '
```

```
2026-09-25 09:15:00.123 +07 [21044] user=app_user,db=ecommerce,xid=4832917 LOG:  statement: DELETE FROM order_items;
```

การระบุด้วย XID **แม่นยำระดับ transaction เดียว** ซึ่งแม่นยำกว่าการระบุเวลา (เพราะในหนึ่งวินาทีอาจมีหลาย transaction commit พร้อมกัน) เหมาะสำหรับกรณีที่ต้องการความแม่นยำสูงสุดและมี XID ที่ชัดเจนอยู่แล้ว

อีกวิธีในการหา XID คือใช้ `pg_waldump` ตรวจสอบเนื้อหาของ WAL โดยตรง:

```bash
$ sudo -u postgres pg_waldump -p /mnt/wal_archive \
    000000010000000000000030 000000010000000000000031 \
    | grep -i "COMMIT" | head -20
```

### 2. recovery_target_name — Restore Point

**Restore point** คือ "ป้ายบอกทาง" (bookmark) ที่ DBA สร้างขึ้นไว้ล่วงหน้าโดยตั้งใจ ณ จุดเวลาที่สำคัญ เช่น ก่อนรัน migration ใหญ่ ก่อน deploy โค้ดเวอร์ชันใหม่ หรือก่อนทำ maintenance ที่มีความเสี่ยง

**การสร้าง restore point ล่วงหน้า** (รันบน primary server ก่อนทำงานที่มีความเสี่ยง):

```sql
-- สร้าง restore point ชื่อ 'before_black_friday_migration' ก่อนรัน schema migration
SELECT pg_create_restore_point('before_black_friday_migration');
```

Restore point นี้จะถูกบันทึกลง WAL ทันที ทำให้ในอนาคตสามารถกู้คืนกลับมายังจุดนี้ได้อย่างแม่นยำโดยไม่ต้องจำเวลาที่แน่นอน:

```ini
recovery_target_name = 'before_black_friday_migration'
```

### ตัวอย่างการใช้งาน restore point ในทางปฏิบัติจริง (e-commerce)

```sql
-- ก่อนรัน migration เพิ่มคอลัมน์ discount_code ในตาราง orders สำหรับแคมเปญ Black Friday
SELECT pg_create_restore_point('pre_migration_add_discount_code');

-- จากนั้นจึงรัน migration จริง
ALTER TABLE orders ADD COLUMN discount_code VARCHAR(20);
```

ถ้าหลัง migration พบว่าเกิดปัญหา (เช่น application มี bug จน insert ข้อมูลเสียหายจำนวนมาก) DBA สามารถ PITR กลับไปยัง restore point `pre_migration_add_discount_code` ได้ทันทีโดยไม่ต้องนั่งไล่หาเวลาที่แน่นอนจาก log

### 3. recovery_target_lsn — ระบุด้วย LSN (Log Sequence Number)

อีกทางเลือกหนึ่งที่แม่นยำระดับต่ำสุด คือระบุด้วย **LSN** (ตำแหน่งไบต์ที่แน่นอนใน WAL stream):

```ini
recovery_target_lsn = '0/3000A458'
```

มักใช้ในกรณีขั้นสูง เช่น การ debug ปัญหาระดับ WAL หรือประสาน recovery กับระบบ replication ที่ report ตำแหน่งเป็น LSN อยู่แล้ว

### ตารางสรุปเปรียบเทียบ recovery target ทั้ง 4 แบบ

| พารามิเตอร์ | ความแม่นยำ | ใช้เมื่อ |
|---|---|---|
| `recovery_target_time` | ระดับวินาที (อาจมีหลาย transaction commit เวลาเดียวกัน) | รู้เวลาโดยประมาณของเหตุการณ์ (กรณีทั่วไปที่พบบ่อยที่สุด) |
| `recovery_target_xid` | ระดับ transaction เดียว | รู้ XID ที่แน่ชัดจาก log หรือ `pg_waldump` |
| `recovery_target_name` | ระดับ transaction เดียว (ตาม restore point) | มีการสร้าง restore point ไว้ล่วงหน้าก่อนงานเสี่ยง (best practice) |
| `recovery_target_lsn` | ระดับไบต์ (แม่นยำที่สุด) | งาน debug ขั้นสูง หรือประสานกับระบบ replication |

> **ข้อควรทราบ:** ใส่ได้เพียง **หนึ่งเดียว** ในสี่พารามิเตอร์นี้ต่อการ recovery หนึ่งครั้งเท่านั้น (ห้ามใส่ `recovery_target_time` พร้อมกับ `recovery_target_xid` เพราะจะกำกวมว่าต้องการหยุดที่จุดไหนกันแน่ — PostgreSQL จะ error หากตั้งมากกว่าหนึ่งค่าพร้อมกัน)

### recovery_target (พิเศษ)

มีพารามิเตอร์เสริมชื่อ `recovery_target` ที่รับค่าเดียวคือ `'immediate'` ใช้เมื่อไม่ต้องการระบุ target ใด ๆ เลย แค่ต้องการหยุด replay WAL ทันทีที่ข้อมูลถึงสถานะ **consistent** (สอดคล้องกันในระดับ transaction) เร็วที่สุดเท่าที่จะทำได้ — มีประโยชน์เมื่อต้องการกู้คืนให้เร็วที่สุดโดยไม่สนใจว่าจะได้ข้อมูลล่าสุดแค่ไหน (เช่น การสร้าง standby server อย่างรวดเร็ว)

```ini
recovery_target = 'immediate'
```

---

## Step 619: ขั้นตอนการทำ PITR แบบเต็มรูปแบบทีละขั้นตอน

หัวข้อนี้จะรวบรวมทุกอย่างที่เรียนมาให้เป็น **ขั้นตอนมาตรฐาน (standard procedure)** ที่ใช้ได้จริงในสถานการณ์ production เพื่อใช้เป็น checklist อ้างอิงเมื่อต้องกู้คืนข้อมูลจริง

### ภาพรวมของกระบวนการทั้งหมด

```
┌─────────────────────────────────────────────────────────────────┐
│  1. หยุด PostgreSQL server ที่มีปัญหา (ถ้ายังรันอยู่)              │
│  2. สำรอง data directory เดิมไว้ก่อน (เผื่อจำเป็นต้องกลับไปดู)      │
│  3. ลบ/ย้าย data directory เดิมออก เตรียมพื้นที่ว่าง                │
│  4. Restore base backup ล่าสุด (ก่อนจุดเวลาเป้าหมาย) ลงในตำแหน่งใหม่ │
│  5. สร้างไฟล์ recovery.signal                                     │
│  6. ตั้งค่า restore_command + recovery_target_* ใน .auto.conf      │
│  7. เริ่ม (start) PostgreSQL server                                │
│  8. PostgreSQL จะ replay WAL อัตโนมัติจนถึง target ที่กำหนด          │
│  9. ตรวจสอบผลลัพธ์ (verify) ว่าข้อมูลถูกต้องตามต้องการ              │
│ 10. Promote server ให้กลับมาเป็น read-write ตามปกติ                │
│ 11. ทำ base backup ใหม่ทันที (timeline เปลี่ยนไปแล้ว)               │
└─────────────────────────────────────────────────────────────────┘
```

### ขั้นตอนที่ 1-3: เตรียมการ

```bash
# 1. หยุด PostgreSQL server (ถ้ายังรันอยู่และเป็นสาเหตุของปัญหา)
$ sudo systemctl stop postgresql@16-main

# 2. สำรอง data directory เดิมไว้ก่อนเสมอ (safety net)
#    อย่าเพิ่งลบทิ้ง! เผื่อ base backup ที่จะ restore มีปัญหา
$ sudo mv /var/lib/postgresql/16/main /var/lib/postgresql/16/main.broken_20260925

# 3. สร้างไดเรกทอรีใหม่สำหรับ data directory ที่กำลังจะ restore
$ sudo mkdir -p /var/lib/postgresql/16/main
$ sudo chown postgres:postgres /var/lib/postgresql/16/main
$ sudo chmod 700 /var/lib/postgresql/16/main
```

### ขั้นตอนที่ 4: Restore base backup

```bash
# 4. คัดลอก base backup ล่าสุด (ที่ทำก่อนเวลา target) เข้าไปในตำแหน่ง data directory
$ sudo -u postgres cp -a /var/backups/pgsql/base/20260925_020000/. \
    /var/lib/postgresql/16/main/

# ตรวจสอบว่าไฟล์สำคัญครบถ้วน
$ sudo -u postgres ls /var/lib/postgresql/16/main/
PG_VERSION  backup_label  base  global  pg_wal  pg_tblspc  postgresql.conf ...
```

> **ข้อสำคัญ:** เลือกใช้ base backup ที่ **เก่าที่สุดเท่าที่จำเป็น แต่ยังเก่ากว่า** recovery target ที่ต้องการ ไม่ใช่ base backup ที่ใหม่ที่สุดเสมอไป — เช่นถ้า target คือ 09:14:59 น. และมี base backup ตอน 02:00 น. กับ 10:00 น. (วันเดียวกัน) ต้องใช้ backup ตอน 02:00 น. เท่านั้น เพราะ backup ตอน 10:00 น. เกิด**หลัง**จุดที่ต้องการกู้คืนไปแล้ว จึงใช้ไม่ได้

### ขั้นตอนที่ 5-6: ตั้งค่า recovery

```bash
# 5. สร้าง recovery.signal (ไฟล์เปล่า)
$ sudo -u postgres touch /var/lib/postgresql/16/main/recovery.signal

# 6. เพิ่ม recovery parameters ลง postgresql.auto.conf
$ sudo -u postgres bash -c "cat >> /var/lib/postgresql/16/main/postgresql.auto.conf" <<'EOF'
restore_command = 'cp /mnt/wal_archive/%f %p'
recovery_target_time = '2026-09-25 09:14:59+07'
recovery_target_inclusive = false
recovery_target_action = 'pause'
EOF
```

ตรวจสอบไฟล์ที่แก้ไข:

```bash
$ sudo -u postgres cat /var/lib/postgresql/16/main/postgresql.auto.conf

# Do not edit this file manually!
# It will be overwritten by the ALTER SYSTEM command.
restore_command = 'cp /mnt/wal_archive/%f %p'
recovery_target_time = '2026-09-25 09:14:59+07'
recovery_target_inclusive = false
recovery_target_action = 'pause'
```

### ขั้นตอนที่ 7-8: Start server และให้ replay ทำงาน

```bash
# 7. เริ่ม PostgreSQL — จะเข้าสู่ recovery mode อัตโนมัติ เพราะเจอ recovery.signal
$ sudo systemctl start postgresql@16-main

# 8. ติดตาม log เพื่อดูความคืบหน้าของการ replay WAL
$ sudo tail -f /var/log/postgresql/postgresql-16-main.log
```

Log ที่ควรเห็นระหว่าง recovery:

```
2026-09-25 09:30:01 LOG:  starting point-in-time recovery to 2026-09-25 09:14:59+07
2026-09-25 09:30:01 LOG:  restored log file "000000010000000000000030" from archive
2026-09-25 09:30:02 LOG:  restored log file "000000010000000000000031" from archive
2026-09-25 09:30:02 LOG:  redo starts at 0/30000028
...
2026-09-25 09:30:05 LOG:  restored log file "000000010000000000000034" from archive
2026-09-25 09:30:05 LOG:  recovery stopping before commit of transaction 4832917, time 2026-09-25 09:15:00.123456+07
2026-09-25 09:30:05 LOG:  recovery has paused
2026-09-25 09:30:05 HINT:  Execute pg_wal_replay_resume() to continue.
```

สังเกตว่า log บอกชัดเจนว่า "recovery stopping **before** commit of transaction 4832917" ซึ่งตรงกับ transaction DELETE ที่ผิดพลาด — แปลว่า PostgreSQL หยุด replay ไว้ **ก่อน** transaction นั้นพอดีตามที่เราตั้งใจ

### ขั้นตอนที่ 9: Verify ข้อมูล

ในขณะที่ recovery อยู่ในสถานะ paused เราสามารถ **query ข้อมูลได้แบบ read-only** เพื่อตรวจสอบความถูกต้องก่อนตัดสินใจ promote:

```sql
-- ตรวจสอบสถานะ
SELECT pg_is_in_recovery();          -- ควรได้ t (true)
SELECT pg_get_wal_replay_pause_state();  -- ควรได้ 'paused'

-- ตรวจสอบว่าข้อมูลที่ต้องการยังอยู่ครบ
SELECT count(*) FROM order_items;
-- ควรได้จำนวนแถวที่ถูกต้อง (ไม่ใช่ 0 แบบที่เกิดจาก DELETE ผิดพลาด)

-- ตรวจสอบคำสั่งซื้อล่าสุดก่อนเวลา 09:15
SELECT order_id, customer_id, created_at, status
FROM orders
ORDER BY created_at DESC
LIMIT 10;

-- ตรวจสอบว่าไม่มีข้อมูลหลังจุด target ปนอยู่ (ควรไม่มีแถวใดหลัง 09:14:59)
SELECT count(*) FROM orders WHERE created_at > '2026-09-25 09:14:59+07';
-- คาดหวังผลลัพธ์: 0
```

หากตรวจสอบแล้วพบว่า **ยังไม่ใช่จุดที่ต้องการ** (เช่น ต้องการเวลาที่ต่างไปเล็กน้อย) สามารถแก้ไข `recovery_target_time` ใหม่แล้ว restart กระบวนการทั้งหมดได้ (กลับไปเริ่มจาก base backup ใหม่อีกครั้ง เพราะเมื่อ pause แล้วจะเลื่อน target ย้อนกลับไปในอดีตไม่ได้ ทำได้แค่เลื่อนไปข้างหน้าด้วย `pg_wal_replay_resume()`)

### ขั้นตอนที่ 10: Promote

เมื่อมั่นใจว่าข้อมูลถูกต้องแล้ว สั่ง promote เพื่อให้ server กลับมาเป็น read-write ตามปกติ:

```bash
$ sudo -u postgres pg_ctl promote -D /var/lib/postgresql/16/main
```

หรือถ้าใช้ SQL function (ใน PostgreSQL 16+ ):

```sql
SELECT pg_promote();
```

Log ที่ควรเห็นหลัง promote:

```
2026-09-25 09:32:10 LOG:  received promote request
2026-09-25 09:32:10 LOG:  redo done at 0/34000A58
2026-09-25 09:32:10 LOG:  selected new timeline ID: 2
2026-09-25 09:32:10 LOG:  archive recovery complete
2026-09-25 09:32:10 LOG:  database system is ready to accept connections
```

สังเกตบรรทัด **"selected new timeline ID: 2"** — นี่คือจุดสำคัญที่ต้องเข้าใจในหัวข้อถัดไป

### ขั้นตอนที่ 11: ทำความเข้าใจเรื่อง Timeline และทำ base backup ใหม่

เมื่อ PostgreSQL promote จาก recovery mode มันจะสร้าง **timeline ใหม่** เสมอ (ในตัวอย่างนี้ ID เปลี่ยนจาก 1 เป็น 2) เหตุผลคือ WAL หลังจากจุด recovery target เดิม (เช่น transaction DELETE ที่ผิดพลาด) **ไม่ถูกนำกลับมาใช้อีก** — ถ้า PostgreSQL ยังใช้ timeline เดิมต่อ อาจเกิดความสับสนว่า WAL segment เดิมกับ WAL ใหม่ที่เขียนหลัง promote นั้นเป็นสาย (history) เดียวกันหรือไม่

Timeline ทำหน้าที่เหมือน "เส้นแขนงประวัติศาสตร์" (history branch) ของฐานข้อมูล คล้ายกับแนวคิด branch ใน Git — เมื่อ PITR แล้ว promote จะเหมือนการสร้าง branch ใหม่จากจุดที่เลือก และ WAL ทั้งหมดที่เขียนใหม่หลังจากนี้จะอยู่บน timeline ใหม่นี้

```bash
# ตรวจสอบ timeline history file ที่ PostgreSQL สร้างขึ้นใน pg_wal/
$ sudo -u postgres ls /var/lib/postgresql/16/main/pg_wal/*.history
00000002.history
```

```bash
$ sudo -u postgres cat /var/lib/postgresql/16/main/pg_wal/00000002.history
1	000000010000000000000034	before 2026-09-25 09:14:59.000000+07
```

ไฟล์นี้บอกว่า timeline 2 แยกออกมาจาก timeline 1 ที่ WAL segment `000000010000000000000034` ณ เวลาก่อน `09:14:59`

**สิ่งที่ต้องทำทันทีหลัง promote:**

```bash
# ทำ base backup ใหม่ทันที เพราะ base backup เก่าอ้างอิง timeline เก่า
# หาก PITR ครั้งต่อไปต้องใช้ base backup เก่า + WAL ข้าม timeline จะซับซ้อนมาก
$ sudo -u postgres pg_basebackup \
    -D /var/backups/pgsql/base/$(date +%Y%m%d_%H%M%S)_after_pitr \
    -Fp -Xs -P -c fast \
    -h localhost -U replicator
```

> **บทเรียนสำคัญ:** ทุกครั้งที่ทำ PITR และ promote สำเร็จ ให้ถือว่าเป็นจุดเริ่มต้นใหม่ของ backup chain เสมอ ควรทำ base backup ใหม่ทันที และตรวจสอบว่า monitoring/alerting เรื่อง `pg_stat_archiver` ยังทำงานถูกต้องบน timeline ใหม่

### สรุปคำสั่งทั้งหมดในรูปแบบ checklist เดียว

```bash
# ===== PITR Full Procedure Checklist =====

# 1) หยุด service เดิม (ถ้ายังรัน)
sudo systemctl stop postgresql@16-main

# 2) สำรอง data dir เดิมไว้ก่อน
sudo mv /var/lib/postgresql/16/main /var/lib/postgresql/16/main.broken_$(date +%Y%m%d)

# 3) เตรียม data dir ใหม่
sudo mkdir -p /var/lib/postgresql/16/main
sudo chown postgres:postgres /var/lib/postgresql/16/main
sudo chmod 700 /var/lib/postgresql/16/main

# 4) restore base backup ที่เหมาะสม (เก่ากว่า target)
sudo -u postgres cp -a /var/backups/pgsql/base/<TIMESTAMP>/. /var/lib/postgresql/16/main/

# 5) สร้าง signal file
sudo -u postgres touch /var/lib/postgresql/16/main/recovery.signal

# 6) ตั้งค่า recovery parameters
sudo -u postgres tee -a /var/lib/postgresql/16/main/postgresql.auto.conf <<'EOF'
restore_command = 'cp /mnt/wal_archive/%f %p'
recovery_target_time = '<TARGET_TIMESTAMP>'
recovery_target_inclusive = false
recovery_target_action = 'pause'
EOF

# 7) start
sudo systemctl start postgresql@16-main

# 8) ตรวจสอบ log จนเจอ "recovery has paused"
sudo tail -f /var/log/postgresql/postgresql-16-main.log

# 9) verify ข้อมูล (psql, read-only queries)

# 10) promote
sudo -u postgres pg_ctl promote -D /var/lib/postgresql/16/main

# 11) base backup ใหม่ทันที
sudo -u postgres pg_basebackup -D /var/backups/pgsql/base/$(date +%Y%m%d_%H%M%S)_after_pitr -Fp -Xs -P -c fast -h localhost -U replicator
```

---

## Step 620: แบบฝึกหัดรวม — จำลองสถานการณ์ข้อมูลถูกลบผิดพลาดและกู้คืนด้วย PITR

หัวข้อนี้เป็น **hands-on lab** แบบครบวงจร ให้ผู้เรียนลงมือทำจริงบนเครื่องทดสอบ (ไม่ใช่ production!) เพื่อประสบการณ์ตรงในการทำ PITR ตั้งแต่ต้นจนจบ สมมติว่าเรากำลังทำงานกับฐานข้อมูล `ecommerce` บนเครื่องทดสอบ

### เตรียมสภาพแวดล้อมสำหรับ Lab

```bash
# 1. ตั้งค่า WAL archiving (ตาม Step 613)
sudo mkdir -p /mnt/wal_archive
sudo chown postgres:postgres /mnt/wal_archive
```

```ini
# postgresql.conf
wal_level = replica
archive_mode = on
archive_command = 'test ! -f /mnt/wal_archive/%f && cp %p /mnt/wal_archive/%f'
archive_timeout = 60
```

```bash
sudo systemctl restart postgresql@16-main
```

```sql
-- ตรวจสอบว่า archiving ทำงาน
SHOW archive_mode;
SELECT pg_switch_wal();
SELECT * FROM pg_stat_archiver;
```

### ขั้นที่ 1: เตรียมข้อมูลทดสอบ

```sql
-- สร้าง schema จำลองแบบง่าย (สมมติว่ามีอยู่แล้วจาก Part ก่อนหน้า)
CREATE TABLE IF NOT EXISTS order_items (
    item_id     SERIAL PRIMARY KEY,
    order_id    INT NOT NULL,
    product_id  INT NOT NULL,
    quantity    INT NOT NULL,
    unit_price  NUMERIC(10,2) NOT NULL,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ใส่ข้อมูลตั้งต้น
INSERT INTO order_items (order_id, product_id, quantity, unit_price)
SELECT
    (random() * 10000)::int,
    (random() * 500)::int,
    (random() * 5 + 1)::int,
    (random() * 1000)::numeric(10,2)
FROM generate_series(1, 1000);

SELECT count(*) FROM order_items;  -- ควรได้ 1000
```

### ขั้นที่ 2: ทำ base backup

```bash
sudo -u postgres pg_basebackup \
    -D /var/backups/pgsql/base/lab_$(date +%Y%m%d_%H%M%S) \
    -Fp -Xs -P -c fast \
    -h localhost -U replicator
```

จดเวลาที่ base backup เสร็จไว้ (สมมติเสร็จตอน `10:00:00`)

### ขั้นที่ 3: จำลอง transaction ปกติที่เกิดขึ้นหลัง backup

```sql
-- เวลา 10:05:00 - มีคำสั่งซื้อใหม่เข้ามาปกติ
INSERT INTO order_items (order_id, product_id, quantity, unit_price)
VALUES (20001, 305, 2, 459.00);

-- ตรวจสอบเวลาปัจจุบันของ server เพื่อใช้อ้างอิง
SELECT now();
```

```sql
-- เวลา 10:06:00 - อีกคำสั่งซื้อหนึ่ง
INSERT INTO order_items (order_id, product_id, quantity, unit_price)
VALUES (20002, 118, 1, 199.50);

SELECT now();  -- จด timestamp นี้ไว้ให้แม่นยำ เช่น '2026-09-25 10:06:03.512+07'
```

**จดเวลานี้ไว้เป็น "จุดปลอดภัยสุดท้าย" (last known good time)** — ในตัวอย่างนี้สมมติว่าได้ `2026-09-25 10:06:03+07`

### ขั้นที่ 4: จำลองความผิดพลาด (accidental DELETE)

```sql
-- เวลา 10:07:00 - เกิดเหตุการณ์! พนักงานรันคำสั่งผิดพลาด
DELETE FROM order_items;

SELECT count(*) FROM order_items;  -- ได้ 0 แถว! ข้อมูลหายทั้งหมด
```

```sql
-- บังคับ switch WAL เพื่อให้แน่ใจว่า transaction นี้ถูก archive แล้ว (สำหรับ lab เท่านั้น)
SELECT pg_switch_wal();
```

ตอนนี้เราอยู่ในสถานการณ์วิกฤต: ตาราง `order_items` ว่างเปล่า ต้องกู้คืนด้วย PITR ไปยังเวลาก่อน `10:07:00`

### ขั้นที่ 5: ดำเนินการ PITR

```bash
# 5.1 หยุด server
sudo systemctl stop postgresql@16-main

# 5.2 สำรอง data dir เดิมไว้ก่อน (แม้จะเป็น lab ก็ควรฝึกวินัยนี้)
sudo mv /var/lib/postgresql/16/main /var/lib/postgresql/16/main.lab_broken

# 5.3 เตรียม data dir ใหม่
sudo mkdir -p /var/lib/postgresql/16/main
sudo chown postgres:postgres /var/lib/postgresql/16/main
sudo chmod 700 /var/lib/postgresql/16/main

# 5.4 restore base backup ที่ทำไว้ในขั้นที่ 2
sudo -u postgres cp -a /var/backups/pgsql/base/lab_20260925_100000/. \
    /var/lib/postgresql/16/main/

# 5.5 สร้าง signal file
sudo -u postgres touch /var/lib/postgresql/16/main/recovery.signal

# 5.6 ตั้งค่า recovery target เป็นเวลาก่อน DELETE (จากขั้นที่ 3)
sudo -u postgres tee -a /var/lib/postgresql/16/main/postgresql.auto.conf <<'EOF'
restore_command = 'cp /mnt/wal_archive/%f %p'
recovery_target_time = '2026-09-25 10:06:03+07'
recovery_target_inclusive = true
recovery_target_action = 'pause'
EOF

# 5.7 start server
sudo systemctl start postgresql@16-main
```

### ขั้นที่ 6: Verify และ Promote

```sql
-- ตรวจสอบว่าอยู่ในสถานะ paused
SELECT pg_is_in_recovery();
SELECT pg_get_wal_replay_pause_state();

-- ตรวจสอบว่าข้อมูลกลับมาครบ (ควรได้ 1002 แถว = 1000 เดิม + 2 คำสั่งซื้อใหม่)
SELECT count(*) FROM order_items;

-- ตรวจสอบว่า order_id 20001 และ 20002 (ที่ insert ก่อน DELETE) ยังอยู่
SELECT * FROM order_items WHERE order_id IN (20001, 20002);
```

หากผลลัพธ์ถูกต้อง (นับได้ 1002 แถว และเจอ order 20001, 20002):

```bash
# promote
sudo -u postgres pg_ctl promote -D /var/lib/postgresql/16/main
```

```sql
-- ตรวจสอบอีกครั้งหลัง promote ว่า server กลับมาเป็น read-write ปกติ
SELECT pg_is_in_recovery();  -- ควรได้ f (false)

-- ทดสอบเขียนข้อมูลใหม่ได้
INSERT INTO order_items (order_id, product_id, quantity, unit_price)
VALUES (20003, 77, 3, 89.00);
```

### ขั้นที่ 7: ทำความสะอาดหลังจบ Lab

```bash
# ทำ base backup ใหม่ (best practice หลัง PITR ทุกครั้ง)
sudo -u postgres pg_basebackup \
    -D /var/backups/pgsql/base/lab_after_pitr_$(date +%Y%m%d_%H%M%S) \
    -Fp -Xs -P -c fast \
    -h localhost -U replicator

# ลบ data dir เก่าที่สำรองไว้ (เมื่อมั่นใจแล้วว่าไม่ต้องใช้)
sudo rm -rf /var/lib/postgresql/16/main.lab_broken
```

### สรุปสิ่งที่ Lab นี้พิสูจน์ให้เห็น

1. WAL archiving ที่ตั้งค่าไว้ล่วงหน้า ทำให้เรามี "เทปบันทึกเหตุการณ์" ของทุก transaction
2. Base backup + WAL archive ทำให้กู้คืนไปยังวินาทีที่ต้องการได้อย่างแม่นยำ
3. ข้อมูลที่เกิดขึ้น **หลัง** base backup แต่ **ก่อน** เหตุการณ์ผิดพลาด (คำสั่งซื้อ 20001, 20002) ถูกกู้คืนกลับมาได้ครบถ้วน — นี่คือสิ่งที่ backup แบบ snapshot อย่างเดียวทำไม่ได้
4. ข้อมูลที่เกิด **หลัง** จุดเป้าหมาย (การ DELETE ที่ผิดพลาด) ไม่ถูกนำกลับมา — ระบบสะอาดจากความผิดพลาดนั้นโดยสมบูรณ์
5. หลัง promote ต้องทำ base backup ใหม่เสมอ เพราะ timeline เปลี่ยนไปแล้ว

---

## สรุปท้ายบท

ในบทนี้เราได้เรียนรู้กลไกที่สำคัญที่สุดอย่างหนึ่งของการดูแลฐานข้อมูล PostgreSQL ระดับมืออาชีพ นั่นคือ **Point-in-Time Recovery (PITR)** ซึ่งอาศัยรากฐานจาก **Write-Ahead Log (WAL)** — บันทึกทุกการเปลี่ยนแปลงข้อมูลที่เกิดขึ้นก่อนจะเขียนลงตารางจริง

ประเด็นสำคัญที่ควรจำ:

- **WAL** คือบันทึกการเปลี่ยนแปลงแบบต่อเนื่อง ที่เมื่อรวมกับ **base backup** จะทำให้กู้คืนข้อมูลไปยังจุดเวลาใดก็ได้ในอดีต
- การเปิดใช้ WAL archiving ต้องตั้งค่า `wal_level = replica`, `archive_mode = on` และ `archive_command` ที่ return exit code 0 เมื่อสำเร็จเท่านั้น
- `pg_basebackup` พร้อม `-Xs` (WAL streaming) คือวิธีสร้าง base backup ที่สมบูรณ์ในตัวเองสำหรับ PITR
- ปลายทางของ archive มีได้หลายแบบ ตั้งแต่ local disk, NFS จนถึง object storage เช่น S3 — หลักการเดียวกันหมด
- ตั้งแต่ PostgreSQL 12 ขึ้นไป ใช้ **`recovery.signal`** (ไฟล์เปล่า) แทน `recovery.conf` และย้ายพารามิเตอร์ recovery ไปไว้ใน `postgresql.conf`/`postgresql.auto.conf`
- `restore_command` และ `recovery_target_time` คือคู่พารามิเตอร์หลักที่ใช้กำหนดว่าจะดึง WAL จากไหน และหยุด replay ที่เวลาใด
- `recovery_target_xid`, `recovery_target_name` (restore point) และ `recovery_target_lsn` เป็นทางเลือกที่แม่นยำกว่าเวลาในบางสถานการณ์ — โดยเฉพาะ restore point ที่ควรสร้างไว้ล่วงหน้าก่อนงานเสี่ยงเสมอ
- ขั้นตอน PITR แบบเต็มรูปแบบมี 11 ขั้นตอนหลัก ตั้งแต่หยุด server, restore base backup, ตั้งค่า recovery, start, verify จนถึง promote และทำ base backup ใหม่
- หลัง promote ทุกครั้ง PostgreSQL จะสร้าง **timeline ใหม่** เสมอ ซึ่งเป็นสัญญาณว่าต้องทำ base backup รอบใหม่ทันที

PITR เป็นทักษะที่ DBA ทุกคนต้อง **ฝึกซ้อมจริง** (ไม่ใช่แค่อ่านทฤษฎี) เพราะในสถานการณ์วิกฤตจริง ความเร็วและความแม่นยำในการกู้คืนคือสิ่งที่ตัดสินว่าองค์กรจะสูญเสียข้อมูลไปมากน้อยแค่ไหน ในบทถัดไป เราจะขยายแนวคิดนี้ไปสู่ **Streaming Replication** ซึ่งใช้กลไก WAL แบบเดียวกันนี้ แต่ส่งข้อมูลแบบ real-time ไปยัง standby server แทนที่จะ archive ไว้กู้คืนภายหลัง

---

## แบบฝึกหัด

<details>
<summary><strong>แบบฝึกหัดที่ 1:</strong> อธิบายความแตกต่างระหว่าง backup แบบ snapshot (เช่น pg_dump รายวัน) กับ PITR ในแง่ของ RPO (Recovery Point Objective)</summary>

**เฉลย:**

Backup แบบ snapshot ให้ RPO เท่ากับ "ช่วงเวลาระหว่างรอบการ backup" เช่น ถ้า backup ทุกเที่ยงคืน RPO สูงสุดคือ 24 ชั่วโมง (ข้อมูลของทั้งวันอาจสูญหายถ้าเกิดปัญหาก่อนถึง backup รอบถัดไป)

PITR ที่ใช้ WAL archiving แบบต่อเนื่อง ให้ RPO ต่ำกว่ามาก — ขึ้นอยู่กับความถี่ที่ WAL segment ถูก archive (ควบคุมด้วย `archive_timeout` และขนาด traffic) โดยทั่วไป RPO อาจอยู่ที่ระดับวินาทีถึงไม่กี่นาที เพราะ WAL ถูกเขียนและ archive อย่างต่อเนื่องตลอดเวลา ไม่ใช่แค่ตอน backup รอบใหญ่เท่านั้น

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 2:</strong> ตั้งค่า postgresql.conf เพื่อเปิดใช้งาน WAL archiving ไปยัง /backup/wal_archive โดยต้องไม่เขียนทับไฟล์เดิมที่มีอยู่แล้ว</summary>

**เฉลย:**

```ini
wal_level = replica
archive_mode = on
archive_command = 'test ! -f /backup/wal_archive/%f && cp %p /backup/wal_archive/%f'
```

จากนั้นต้อง restart PostgreSQL (ไม่ใช่แค่ reload) เพราะ `wal_level` เป็นพารามิเตอร์ระดับ `postmaster` context:

```bash
sudo systemctl restart postgresql@16-main
```

ส่วน `test ! -f ... &&` คือกลไกป้องกันการเขียนทับไฟล์เดิม หากไฟล์ปลายทางมีอยู่แล้ว คำสั่ง `cp` จะไม่ถูกเรียก และ `archive_command` จะ return exit code ที่ไม่ใช่ 0 (fail) ซึ่งเป็นพฤติกรรมที่ถูกต้อง เพราะไม่ควรมี WAL segment สองไฟล์ชื่อเดียวกันเนื้อหาต่างกัน

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 3:</strong> เหตุใด pg_basebackup ควรใช้ -Xs (stream) แทนที่จะไม่ระบุ -X เลย เมื่อใช้สำหรับ PITR</summary>

**เฉลย:**

ถ้าไม่ระบุ `-X` เลย ค่าเริ่มต้นจะไม่รวม WAL ใด ๆ ไปกับ base backup (เทียบเท่า `--wal-method=none` ในบางเวอร์ชัน หรือพฤติกรรม default ที่ต้องพึ่ง archiving ภายนอกทั้งหมด) ซึ่งหมายความว่า base backup ที่ได้จะไม่ "สมบูรณ์ในตัวเอง" — ต้องพึ่งพา `archive_command`/`restore_command` ให้ทำงานถูกต้อง 100% สำหรับ WAL ที่เกิดขึ้น **ระหว่าง** การ backup กำลังทำงานอยู่ หาก WAL segment เหล่านั้นถูก recycle ไปก่อนที่จะ archive ได้ทัน (เช่น archive_command ล้มเหลวชั่วคราว) การ restore ในอนาคตจะขาด WAL ช่วงนั้นไป ทำให้ backup ใช้งานไม่ได้

การใช้ `-Xs` (stream) เปิด connection คู่ขนานเพื่อ stream WAL ที่เกิดขึ้นระหว่าง backup มาเก็บไว้ใน base backup โดยตรง ทำให้ base backup ที่ได้สมบูรณ์และปลอดภัยกว่า ไม่ต้องพึ่งพา archive_command สำหรับช่วงเวลานั้นเลย จึงเป็นตัวเลือกที่แนะนำสำหรับ production เสมอ

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 4:</strong> อธิบายว่าทำไม PostgreSQL 16 จึงปฏิเสธการ start ถ้าพบไฟล์ recovery.conf ใน data directory และต้องทำอย่างไรแทน</summary>

**เฉลย:**

ตั้งแต่ PostgreSQL 12 เป็นต้นไป ไฟล์ `recovery.conf` ถูกยกเลิกไปโดยสิ้นเชิง กลไกการเข้าสู่ recovery mode ถูกเปลี่ยนไปใช้ signal file (`recovery.signal` หรือ `standby.signal`) ร่วมกับพารามิเตอร์ recovery ที่ย้ายไปอยู่ใน `postgresql.conf`/`postgresql.auto.conf` แทน

หาก PostgreSQL 16 พบไฟล์ `recovery.conf` มันจะถือว่าเป็นการตั้งค่าที่ไม่ถูกต้อง (ค้างมาจาก config รุ่นเก่าที่ไม่รองรับแล้ว) และปฏิเสธการ start ทันทีพร้อม error message เพื่อป้องกันไม่ให้ผู้ใช้สับสนว่าทำไม config เก่าไม่มีผล

วิธีที่ถูกต้องคือ:
1. ลบไฟล์ `recovery.conf` (ถ้ามี) ออกจาก data directory
2. สร้างไฟล์ `recovery.signal` (ไฟล์เปล่า) แทน
3. ใส่พารามิเตอร์ recovery เช่น `restore_command`, `recovery_target_time` ลงใน `postgresql.conf` หรือ `postgresql.auto.conf`

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 5:</strong> ตั้งค่า recovery ให้กู้คืนข้อมูลไปยังเวลา 2026-09-25 14:30:00 (เวลาไทย +07) โดยไม่รวม transaction ที่ commit พอดีเวลานั้น และให้หยุดรอ (ไม่ promote อัตโนมัติ) เพื่อให้ตรวจสอบก่อน</summary>

**เฉลย:**

```ini
restore_command = 'cp /mnt/wal_archive/%f %p'
recovery_target_time = '2026-09-25 14:30:00+07'
recovery_target_inclusive = false
recovery_target_action = 'pause'
```

`recovery_target_inclusive = false` ทำให้ transaction ที่ commit พอดี ณ เวลา 14:30:00 **ไม่ถูกรวม** เข้ามา (หยุดก่อนหน้านั้น) ส่วน `recovery_target_action = 'pause'` (ซึ่งเป็นค่าเริ่มต้นอยู่แล้ว แต่การระบุชัดเจนช่วยให้อ่านง่าย) ทำให้ server หยุดรอในสถานะ paused เมื่อถึง target แทนที่จะ promote ให้ทันที เปิดโอกาสให้ตรวจสอบข้อมูลด้วย read-only query ก่อนตัดสินใจ

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 6:</strong> restore point คืออะไร และมีข้อดีอย่างไรเมื่อเทียบกับการระบุ recovery_target_time</summary>

**เฉลย:**

Restore point คือ "ป้ายบอกทาง" ที่ DBA สร้างขึ้นไว้ล่วงหน้าโดยเจตนา ณ จุดเวลาที่สำคัญ ด้วยคำสั่ง `SELECT pg_create_restore_point('ชื่อ');` ซึ่งจะถูกบันทึกลง WAL ทันที

ข้อดีเมื่อเทียบกับการระบุเวลา:
1. **ไม่ต้องจำเวลาที่แน่นอน** — แค่จำชื่อ restore point เช่น `before_migration_v2` ก็สามารถ recovery กลับไปได้แม่นยำ
2. **แม่นยำระดับ transaction เดียว** ไม่มีความกำกวมเหมือนการระบุเวลาที่อาจมีหลาย transaction commit ในวินาทีเดียวกัน
3. **เหมาะกับ workflow ที่วางแผนล่วงหน้า** เช่น ก่อนรัน schema migration หรือ deploy ที่มีความเสี่ยง ทำให้มี "จุด rollback" ที่ชัดเจนพร้อมใช้ทันทีโดยไม่ต้องมานั่งไล่ log หา timestamp ภายหลัง

ใช้งานผ่าน `recovery_target_name = 'ชื่อ restore point'` ใน recovery configuration

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 7:</strong> ในกระบวนการ PITR แบบเต็มรูปแบบ ทำไมจึงต้องเลือก base backup ที่ "เก่าที่สุดเท่าที่จำเป็น แต่ยังเก่ากว่า" recovery target เสมอ ไม่ใช่ base backup ที่ใหม่ที่สุด</summary>

**เฉลย:**

เพราะ base backup คือ "จุดเริ่มต้น" ของกระบวนการ replay WAL — PostgreSQL จะเริ่ม apply WAL จากจุดที่ base backup ถูกสร้างขึ้น (ตามที่ระบุใน `backup_label`) ไปข้างหน้าเท่านั้น ไม่สามารถ replay WAL ย้อนหลังได้

หากเลือก base backup ที่ถูกสร้างขึ้น **หลัง** จุดเวลาเป้าหมายที่ต้องการกู้คืน (เช่น target คือ 09:14:59 แต่ base backup ทำตอน 10:00:00) ข้อมูลในฐานข้อมูลตอนที่ base backup ถูกสร้างจะ**เลยจุด target ไปแล้ว** — ไม่มีทางย้อนกลับไปยังเวลา 09:14:59 ได้อีก เพราะ base backup นั้นได้ "บันทึก" สถานะข้อมูล ณ เวลาที่มากกว่า target ไปเรียบร้อยแล้ว รวมถึงอาจมีการ DELETE ที่ผิดพลาดรวมอยู่ในนั้นด้วย

ดังนั้นต้องเลือก base backup ที่เก่ากว่า target เสมอ เพื่อให้มี "ระยะ" ของ WAL ที่จะ replay ไปสู่จุดที่ต้องการได้จริง

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 8:</strong> หลัง promote server จาก PITR เสร็จแล้ว เหตุใดจึงจำเป็นต้องทำ base backup ใหม่ทันที และคำว่า "timeline" ในบริบทนี้หมายถึงอะไร</summary>

**เฉลย:**

เมื่อ PostgreSQL promote จาก recovery mode มันจะสร้าง **timeline ID ใหม่** เสมอ (เช่นจาก 1 เป็น 2) ซึ่งเปรียบเสมือนการสร้าง "แขนงประวัติศาสตร์" (history branch) ใหม่ของข้อมูล คล้ายกับ branch ใน Git — WAL ทั้งหมดที่เขียนขึ้นหลัง promote จะอยู่บน timeline ใหม่นี้ ในขณะที่ WAL หลังจุด recovery target บน timeline เดิม (ซึ่งรวมถึงเหตุการณ์ผิดพลาดที่เราต้องการหลีกเลี่ยง) จะถูกละทิ้งไปอย่างถาวร

เหตุผลที่ต้องทำ base backup ใหม่ทันทีคือ:
1. Base backup chain เดิม + WAL archive เดิม อ้างอิงกับ timeline เก่า หาก PITR ครั้งต่อไปในอนาคตต้องใช้ base backup เก่านี้ ร่วมกับ WAL ที่เขียนหลัง promote (ซึ่งอยู่คนละ timeline) กระบวนการจะซับซ้อนมาก (ต้องอาศัย `.history` file ในการข้าม timeline ซึ่งเสี่ยงต่อความผิดพลาด)
2. การมี base backup ใหม่บน timeline ใหม่ทำให้ backup chain สะอาดและง่ายต่อการกู้คืนในอนาคต
3. เป็นการรับประกันว่าถ้าเกิดปัญหาซ้ำในอนาคตอันใกล้ เรามีจุดเริ่มต้น (base backup) ที่ใหม่และตรงกับ timeline ปัจจุบันจริง ๆ

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 9:</strong> สมมติ archive_command ล้มเหลวต่อเนื่องเป็นเวลาหลายชั่วโมง (เช่น ปลายทาง NFS mount หลุดการเชื่อมต่อ) จะเกิดผลกระทบอะไรกับ PostgreSQL server และควรตรวจสอบด้วยคำสั่งอะไร</summary>

**เฉลย:**

เมื่อ `archive_command` ล้มเหลว (return exit code ที่ไม่ใช่ 0) PostgreSQL จะ **ไม่ลบ** WAL segment ไฟล์นั้นออกจาก `pg_wal/` และจะพยายามเรียก `archive_command` ซ้ำไปเรื่อย ๆ จนกว่าจะสำเร็จ ผลกระทบที่ตามมาคือ:

1. **WAL segment สะสมเพิ่มขึ้นเรื่อย ๆ** ใน `pg_wal/` เพราะไฟล์ที่ยังไม่ถูก archive สำเร็จจะไม่ถูก recycle หรือลบทิ้ง
2. หากปล่อยไว้นานพอ (หลายชั่วโมงในระบบที่มี write traffic สูง) **ดิสก์ที่เก็บ `pg_wal/` อาจเต็ม** ซึ่งจะทำให้ PostgreSQL ไม่สามารถเขียน WAL ใหม่ได้อีก และ **server จะ crash หรือปฏิเสธ transaction ใหม่ทั้งหมด** — เป็นปัญหาระดับวิกฤต (Denial of Service ของฐานข้อมูลทั้งระบบ)
3. **RPO แย่ลงทันที** เพราะ WAL ที่ค้างอยู่ยังไม่ถูก archive ไปยังปลายทางที่ปลอดภัย หากเซิร์ฟเวอร์หลักล่มในช่วงนี้ (ก่อนที่ archive จะสำเร็จ) ข้อมูลในช่วงเวลาที่ archive ล้มเหลวอาจสูญหายได้จริง เพราะ WAL ยังอยู่แค่ในเครื่องเดียว

คำสั่งที่ควรใช้ตรวจสอบ:

```sql
-- ดูสถิติการ archive ว่ามี failed_count เพิ่มขึ้นหรือไม่ และเวลาที่ archive สำเร็จล่าสุดคือเมื่อไหร่
SELECT * FROM pg_stat_archiver;
```

```bash
# ตรวจสอบ error message จาก log ของ PostgreSQL โดยตรง
sudo tail -100 /var/log/postgresql/postgresql-16-main.log | grep -i archive

# ตรวจสอบพื้นที่ดิสก์ของ pg_wal/ ว่าใกล้เต็มหรือยัง
df -h $(sudo -u postgres psql -tAc "SHOW data_directory")/pg_wal

# ตรวจสอบจำนวนไฟล์ WAL ที่ค้างอยู่ใน pg_wal/
sudo -u postgres ls /var/lib/postgresql/16/main/pg_wal/ | grep -E '^[0-9A-F]{24}$' | wc -l
```

การแก้ปัญหาเร่งด่วนคือแก้ไขสาเหตุที่ทำให้ `archive_command` ล้มเหลว (เช่น mount NFS กลับมาใหม่, แก้ permission, เพิ่มพื้นที่ปลายทาง) ทันทีที่แก้ไขเสร็จ WAL ที่ค้างอยู่จะถูก archive ตามไปเรื่อย ๆ จนหมดคิวโดยอัตโนมัติ ไม่ต้องทำอะไรเพิ่มเติม

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 10:</strong> เรียงลำดับขั้นตอนต่อไปนี้ให้ถูกต้องตามกระบวนการ PITR แบบเต็มรูปแบบ: (ก) promote server, (ข) สร้าง recovery.signal, (ค) restore base backup, (ง) verify ข้อมูลด้วย read-only query, (จ) หยุด PostgreSQL server เดิม, (ฉ) ทำ base backup ใหม่, (ช) ตั้งค่า restore_command และ recovery_target_time, (ซ) start PostgreSQL แล้วรอ recovery ทำงาน</summary>

**เฉลย:**

ลำดับที่ถูกต้องคือ:

1. **(จ)** หยุด PostgreSQL server เดิม
2. **(ค)** restore base backup (คัดลอก base backup ที่เหมาะสมลงใน data directory ใหม่)
3. **(ข)** สร้าง `recovery.signal`
4. **(ช)** ตั้งค่า `restore_command` และ `recovery_target_time` ใน `postgresql.auto.conf`
5. **(ซ)** start PostgreSQL แล้วรอ recovery ทำงาน (replay WAL จนถึง target)
6. **(ง)** verify ข้อมูลด้วย read-only query ขณะที่ recovery อยู่ในสถานะ paused
7. **(ก)** promote server ให้กลับมาเป็น read-write
8. **(ฉ)** ทำ base backup ใหม่ทันที (เพราะ timeline เปลี่ยนไปแล้วหลัง promote)

หลักการสำคัญที่ต้องจำคือ ต้อง **เตรียมทุกอย่างให้พร้อมก่อน start server** (base backup + signal file + recovery parameters) เพราะเมื่อ start แล้ว PostgreSQL จะเข้าสู่กระบวนการ recovery ทันทีโดยอัตโนมัติ และควร **verify ก่อน promote เสมอ** เนื่องจากการ promote เป็นจุดที่ย้อนกลับไม่ได้ (สร้าง timeline ใหม่ทันที) หากพบว่าเลือกจุดเวลาไม่ถูกต้อง จะต้องเริ่มกระบวนการทั้งหมดใหม่ตั้งแต่ restore base backup อีกครั้ง

</details>

---

## บทถัดไป

บทนี้เน้นการกู้คืนข้อมูลแบบ "ตอบสนองเมื่อเกิดปัญหา" (reactive) ด้วยการ replay WAL จาก archive ในบทถัดไป เราจะขยายแนวคิด WAL ไปสู่การป้องกันปัญหาเชิงรุก (proactive) ด้วยการส่งสำเนาข้อมูลแบบ real-time ไปยังเซิร์ฟเวอร์สำรองที่พร้อมทำงานแทนได้ทันที

**บทถัดไป:** [Part 063 — Streaming Replication](./part-063-streaming-replication.md)