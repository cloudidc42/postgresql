# PostgreSQL Internals: Process Architecture (Postmaster, Backend)

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับโลก/ผู้เชี่ยวชาญ | Part 081

---

ยินดีต้อนรับสู่ระดับสุดท้ายของหลักสูตรนี้ — **ระดับโลก / ผู้เชี่ยวชาญ (World-Class / Expert)**

ตลอดหลักสูตรที่ผ่านมา เราเรียนรู้การ "ใช้" PostgreSQL ตั้งแต่คำสั่ง SQL พื้นฐาน ไปจนถึงการจูน performance, replication, การจัดการ transaction, indexing และ query optimization ในระดับสูง แต่คำถามที่แท้จริงของ "ผู้เชี่ยวชาญระดับโลก" ไม่ใช่แค่ "ทำอย่างไร" (How) แต่คือ "ทำไมมันถึงทำงานแบบนี้" (Why) — และคำตอบนั้นอยู่ใน **internals** ของ PostgreSQL เอง

นับจากบทนี้เป็นต้นไป เราจะ "ผ่าตัด" PostgreSQL ดูโครงสร้างภายในทีละชั้น เริ่มจากสิ่งที่พื้นฐานที่สุด — **สถาปัตยกรรมของ process** — ว่าเมื่อคุณพิมพ์ `psql` แล้วเชื่อมต่อไปยัง PostgreSQL server เกิดอะไรขึ้นจริง ๆ ในระบบปฏิบัติการ มี process อะไรเกิดขึ้นบ้าง แต่ละ process ทำหน้าที่อะไร และทำไมสถาปัตยกรรมแบบนี้ถึงเป็นรากฐานของทั้งความเสถียรและข้อจำกัดของ PostgreSQL ที่เราเจอมาตลอดหลักสูตร (เช่นเรื่อง connection pooling ใน Part 066)

การเข้าใจ process architecture คือกุญแจสำคัญที่จะทำให้คุณอ่าน log, วินิจฉัยปัญหา performance, และตัดสินใจเชิงสถาปัตยกรรมได้อย่างมีเหตุผล ไม่ใช่แค่ท่องจำคำสั่ง tuning

## เป้าหมายการเรียนรู้

เมื่อจบบทนี้ ผู้เรียนจะสามารถ:

1. อธิบายได้ว่า PostgreSQL ใช้สถาปัตยกรรมแบบ **process-based** ไม่ใช่ **thread-based** และเหตุผลเชิงประวัติศาสตร์/วิศวกรรมเบื้องหลังการตัดสินใจนี้
2. อธิบายบทบาทของ **Postmaster** ในฐานะ process แม่ที่ควบคุมวงจรชีวิตของทั้งระบบ
3. เข้าใจว่า 1 client connection ผูกกับ 1 **backend process** เสมอ และสามารถสำรวจ process เหล่านี้ได้จริงด้วยคำสั่งระบบปฏิบัติการ
4. อธิบายโครงสร้างและบทบาทของ **Shared Memory** ที่ทุก process ใช้ร่วมกัน
5. ระบุหน้าที่ของ background process หลักแต่ละตัว: Background Writer, Checkpointer, WAL Writer, Autovacuum Launcher/Worker, Stats Collector / cumulative statistics, Logical Replication Launcher
6. เข้าใจกลไกการสื่อสารระหว่าง process ผ่าน shared memory, semaphore และ signal
7. เปรียบเทียบข้อดี-ข้อเสียของสถาปัตยกรรม process-based กับ thread-based (MySQL, SQL Server) อย่างมีหลักการ
8. เชื่อมโยงความรู้ internals กลับไปอธิบาย "ทำไมต้องทำ connection pooling" ด้วยความเข้าใจเชิงกลไกจริง ไม่ใช่แค่จำ best practice
9. สำรวจ process ของ PostgreSQL บนเครื่องจริงได้ด้วยตนเอง และจับคู่แต่ละ process กับหน้าที่ที่ถูกต้อง

---

## Step 801: ภาพรวมสถาปัตยกรรม PostgreSQL — Process-based ไม่ใช่ Thread-based

### จุดเริ่มต้น: ฐานข้อมูลก็คือโปรแกรมหนึ่งที่รันบน OS

ก่อนจะลงลึกในรายละเอียด เราต้องปรับมุมมองก่อนว่า PostgreSQL Server ไม่ใช่ "กล่องดำ" ที่ลอยอยู่เฉย ๆ แต่มันคือ**ชุดของโปรแกรม (executable)** ที่รันอยู่บนระบบปฏิบัติการ (Linux, macOS, Windows) เหมือนโปรแกรมอื่น ๆ ทั่วไป เพียงแต่มันถูกออกแบบให้มีหลาย process ทำงานร่วมกันเป็นระบบเดียว

เมื่อคุณสั่ง `pg_ctl start` หรือ `systemctl start postgresql` สิ่งที่เกิดขึ้นคือ OS จะรัน executable ชื่อ `postgres` (บางระบบเก่าเรียก `postmaster`) ขึ้นมาเป็น process แรก แล้ว process นี้จะ **fork()** ตัวเองซ้ำ ๆ เพื่อสร้าง process ลูกจำนวนมาก แต่ละ process ลูกมีหน้าที่เฉพาะของตัวเอง

### Process-based vs Thread-based คืออะไร

ก่อนเข้าใจ PostgreSQL เราต้องเข้าใจความแตกต่างพื้นฐานระหว่างสอง model นี้ก่อน

**Thread-based model** (เช่น MySQL, SQL Server, Oracle บางส่วน):
- Database server เป็น **1 process เดียว**
- แต่ละ client connection ถูกจัดการโดย **thread** ที่รันอยู่ภายใน process เดียวกัน
- ทุก thread แชร์ **memory address space เดียวกัน** โดยธรรมชาติ (ไม่ต้องทำอะไรพิเศษ)
- การสลับ (context switch) ระหว่าง thread เบากว่าการสลับ process
- แต่ถ้า thread หนึ่งเขียน memory ผิดพลาด (buffer overflow, null pointer, segfault) มันสามารถทำให้**ทั้ง process ล่ม** ซึ่งหมายถึง**ทุก connection ล่มพร้อมกัน**

**Process-based model** (PostgreSQL):
- Database server ประกอบด้วย **หลาย OS process** ที่แยกจากกัน
- แต่ละ client connection ถูกจัดการโดย **process แยกต่างหาก** (1 connection = 1 process)
- แต่ละ process มี memory address space ของตัวเอง (isolated) — ถ้าต้องการแชร์ข้อมูลต้องใช้ **shared memory segment** ที่ตั้งใจสร้างขึ้นมาเฉพาะ
- การสร้าง process (fork) และสลับ process หนักกว่า thread
- แต่ถ้า process หนึ่งล่ม (crash) ผลกระทบจะถูก "กัน" ไว้ในระดับหนึ่ง ไม่ลามไปทุก connection ทันที (รายละเอียดใน Step 808)

```
┌─────────────────────────────────────────────────────────────┐
│                      Thread-based (MySQL)                     │
│                                                                 │
│   ┌───────────────────────────────────────────────────────┐  │
│   │              mysqld (1 OS Process)                     │  │
│   │                                                          │  │
│   │   ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐      │  │
│   │   │Thread 1│  │Thread 2│  │Thread 3│  │Thread N│      │  │
│   │   │(conn A)│  │(conn B)│  │(conn C)│  │(conn.. )│      │  │
│   │   └────────┘  └────────┘  └────────┘  └────────┘      │  │
│   │        │            │            │            │         │  │
│   │        └────────────┴────────────┴────────────┘         │  │
│   │                shared memory space (automatic)           │  │
│   └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                   Process-based (PostgreSQL)                  │
│                                                                 │
│   ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐    │
│   │ postgres │  │ postgres │  │ postgres │  │ postgres │    │
│   │(backend  │  │(backend  │  │(backend  │  │(bg writer│    │
│   │ conn A)  │  │ conn B)  │  │ conn C)  │  │ process) │    │
│   │ PID 1001 │  │ PID 1002 │  │ PID 1003 │  │ PID 1004 │    │
│   └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘    │
│        │             │             │             │           │
│        └─────────────┴──────┬──────┴─────────────┘           │
│                              │                                 │
│                 ┌────────────▼────────────┐                   │
│                 │   Shared Memory Segment   │                   │
│                 │  (ต้องสร้างขึ้นชัดเจนตั้งแต่ start)  │                   │
│                 └───────────────────────────┘                   │
└─────────────────────────────────────────────────────────────┘
```

### ทำไม PostgreSQL ถึงเลือก process-based

ต้องเข้าใจบริบทประวัติศาสตร์ก่อน: PostgreSQL มีรากมาจากโปรเจกต์ POSTGRES ที่ University of California, Berkeley เริ่มตั้งแต่ปี 1986 ซึ่งเป็นยุคที่ **thread support ในระบบปฏิบัติการ Unix ยังไม่เสถียรหรือไม่มีมาตรฐานเดียวกัน** (POSIX threads หรือ pthreads เพิ่งเป็นมาตรฐานจริงจังช่วงต้นทศวรรษ 1990) การใช้ `fork()` เพื่อสร้าง process ใหม่เป็นกลไกที่มีอยู่แล้วใน Unix ตั้งแต่ยุคแรกเริ่ม เสถียร และเป็นมาตรฐานข้ามแพลตฟอร์มมากกว่า

แต่เหตุผลที่ PostgreSQL "ยังคง" ใช้ process-based มาจนถึงทุกวันนี้ (แม้ thread จะพัฒนาไปมากแล้ว) ไม่ใช่แค่เพราะประวัติศาสตร์ แต่เพราะข้อดีเชิงวิศวกรรมที่แท้จริง:

1. **Isolation / Fault Tolerance**: ถ้า backend process หนึ่งเกิด segmentation fault จาก bug ใน extension หรือ edge case แปลก ๆ ระบบจะ "รู้ตัว" ผ่าน signal และสามารถ reset ตัวเองอย่างปลอดภัย โดยไม่ทำให้ connection อื่นที่กำลังทำงานอยู่หายไปทันที (ในกรณีส่วนใหญ่)
2. **Memory Protection โดย OS**: แต่ละ process มี virtual memory space แยกกัน การเขียนข้อมูลผิดพลาดของ backend หนึ่งจะไม่ไปทับ memory ของ backend อื่นโดยบังเอิญ เพราะ OS-level memory protection ป้องกันไว้อยู่แล้ว
3. **ความง่ายในการ debug**: สามารถใช้เครื่องมือระดับ OS อย่าง `ps`, `top`, `gdb`, `strace` แนบเข้ากับแต่ละ connection ได้โดยตรง เพราะแต่ละ connection คือ process จริง ๆ ที่มองเห็นได้จาก OS
4. **Security boundary ที่ชัดเจน**: OS-level permission (เช่น file descriptor, resource limit ต่อ process) สามารถนำมาใช้ควบคุมแต่ละ connection ได้

ข้อเสียคือ**ต้นทุนต่อ connection สูงกว่า** ทั้งในแง่ memory footprint และเวลาที่ใช้ fork process ใหม่ ซึ่งเป็นที่มาของปัญหาที่เราจะพูดถึงใน Step 808-809 (และเป็นเหตุผลที่ PgBouncer/PgPool ถูกพัฒนาขึ้นมา — ทบทวน Part 066)

> **สรุปหลักการ**: PostgreSQL แลก "ต้นทุนต่อ connection ที่สูงกว่า" เพื่อแลกกับ "ความเสถียรและความปลอดภัยของระบบโดยรวมที่สูงกว่า" — นี่คือ design trade-off ที่จงใจ ไม่ใช่ข้อจำกัดทางเทคนิคที่แก้ไม่ได้

### ตรวจสอบด้วยตัวเอง: PostgreSQL เป็น multi-process จริงหรือไม่

```bash
# บนเครื่อง Linux ที่รัน PostgreSQL อยู่
$ ps aux | grep postgres

postgres   1001  0.0  0.5 215000 21344 ?        Ss   08:00   0:00 /usr/lib/postgresql/16/bin/postgres -D /var/lib/postgresql/16/main
postgres   1005  0.0  0.1 215136  4520 ?        Ss   08:00   0:00 postgres: checkpointer
postgres   1006  0.0  0.1 215000  3212 ?        Ss   08:00   0:00 postgres: background writer
postgres   1007  0.0  0.2 215368  5680 ?        Ss   08:00   0:00 postgres: walwriter
postgres   1008  0.0  0.1 215636  4108 ?        Ss   08:00   0:00 postgres: autovacuum launcher
postgres   1009  0.0  0.1  69720  3016 ?        Ss   08:00   0:00 postgres: logical replication launcher
postgres   2101  0.0  0.3 216200  9840 ?        Ss   09:14   0:00 postgres: myapp app_user 10.0.1.5(53210) idle
postgres   2144  0.1  0.4 217120 11200 ?        Rs   09:15   0:00 postgres: myapp app_user 10.0.1.5(53298) SELECT
```

สังเกตว่ามี **หลาย process แยกกันจริง ๆ** ในระดับ OS แต่ละบรรทัดคือ 1 PID (Process ID) และหลังคำว่า `postgres:` จะบอกหน้าที่ของ process นั้นอย่างชัดเจน — นี่คือสิ่งที่เรียกว่า **process title** ที่ PostgreSQL ตั้งค่าให้ตัวเองเพื่อให้ admin มองเห็นสถานะได้ทันทีจาก `ps`

บทถัดไปเราจะเจาะลึกแต่ละ process เหล่านี้ทีละตัว เริ่มจาก process แรกสุด: **Postmaster**

---

## Step 802: Postmaster Process — process แม่ตัวแรกที่เริ่มทำงาน

### Postmaster คืออะไร

**Postmaster** คือชื่อดั้งเดิม (historical name) ของ process แรกสุดที่เริ่มทำงานเมื่อคุณ start PostgreSQL server ในเวอร์ชันปัจจุบัน executable ที่รันจริง ๆ ชื่อ `postgres` (ไฟล์เดียวกับที่ backend process ใช้) แต่เมื่อรันโดยไม่มี argument พิเศษที่บ่งบอกว่าเป็น backend มันจะทำหน้าที่เป็น postmaster — จึงมักเห็นคนเรียกมันว่า "the postmaster process" หรือ "the main server process" สลับกันไป

```bash
$ pg_ctl -D /var/lib/postgresql/16/main status
pg_ctl: server is running (PID: 1001)

$ ps -p 1001 -o pid,ppid,cmd
    PID    PPID CMD
   1001       1 /usr/lib/postgresql/16/bin/postgres -D /var/lib/postgresql/16/main -c config_file=/etc/postgresql/16/main/postgresql.conf
```

สังเกตว่า `PPID` (Parent PID) ของ postmaster คือ `1` (หรือ `init`/`systemd` บนระบบที่ใช้ systemd) — แปลว่า postmaster เป็น process ระดับบนสุดของ PostgreSQL ทั้งหมด ไม่มี PostgreSQL process อื่นใดเป็นพ่อของมัน

### วงจรชีวิตเมื่อเริ่มต้น (Startup Sequence)

เมื่อ postmaster เริ่มทำงาน มันทำงานตามลำดับต่อไปนี้:

```
┌──────────────────────────────────────────────────────────────┐
│                    Postmaster Startup Sequence                 │
└──────────────────────────────────────────────────────────────┘

 1. อ่านไฟล์ config
    postgresql.conf, pg_hba.conf, postgresql.auto.conf
              │
              ▼
 2. ตรวจสอบและ allocate Shared Memory + Semaphore
    (ขนาดตาม shared_buffers, max_connections, ฯลฯ)
              │
              ▼
 3. รัน Crash Recovery หากจำเป็น
    (ตรวจสอบ pg_control, WAL replay ถ้า shutdown ไม่ปกติครั้งก่อน)
              │
              ▼
 4. Fork background processes ที่จำเป็นต้องมีตลอดเวลา
    - checkpointer
    - background writer
    - walwriter
    - autovacuum launcher
    - logical replication launcher
    - stats collector process (หรือ cumulative stats ใน PG15+
      ซึ่งไม่มี process แยกอีกต่อไป — ดู Step 806)
              │
              ▼
 5. เปิด listening socket
    - TCP/IP socket (ตาม listen_addresses, port)
    - Unix domain socket (สำหรับ local connection)
              │
              ▼
 6. เข้าสู่ main loop: รอ connection request เข้ามา (accept loop)
```

### หน้าที่หลักของ Postmaster

Postmaster มีหน้าที่หลัก 4 อย่างตลอดอายุการทำงานของ server:

**1. Listen for incoming connections** — postmaster เปิด socket และ "ฟัง" (listen) การเชื่อมต่อเข้ามาตลอดเวลา ไม่ว่าจะเป็นทาง TCP/IP (port 5432 เป็นค่า default) หรือ Unix domain socket

**2. Spawn (fork) child process สำหรับแต่ละ connection ใหม่** — เมื่อมี client เชื่อมต่อเข้ามาสำเร็จ (ผ่านการตรวจสอบเบื้องต้นตาม `pg_hba.conf`) postmaster จะเรียก `fork()` เพื่อสร้าง **backend process ใหม่** ที่จะรับผิดชอบ connection นั้นโดยเฉพาะ แล้ว postmaster ก็กลับไป listen ต่อทันที — **postmaster เองไม่เคยประมวลผล SQL query ใด ๆ เลย**

**3. Monitor และจัดการ child process ทั้งหมด** — postmaster คอยตรวจสอบว่า background process และ backend process ทั้งหมดยังทำงานปกติหรือไม่ (ผ่านกลไก signal ที่จะพูดถึงใน Step 807) ถ้า process ใดล่มโดยไม่คาดคิด postmaster จะตัดสินใจว่าต้อง initiate crash recovery ทั้งระบบหรือไม่

**4. จัดการ shutdown/restart ของทั้งระบบ** — เมื่อได้รับ signal ให้ shutdown (เช่นจาก `pg_ctl stop`) postmaster จะส่ง signal ต่อไปยัง child process ทั้งหมดตามลำดับที่เหมาะสม (smart / fast / immediate shutdown mode)

```
┌───────────────────────────────────────────────────────────┐
│                         Postmaster                            │
│                        (PID 1001)                             │
│                                                                │
│   หน้าที่: Listen + Fork + Monitor + Shutdown coordination      │
│   ไม่เคยรัน SQL query ใด ๆ ด้วยตัวเอง                          │
└──────────────────────┬────────────────────────────────────┘
                        │ fork() เมื่อมี connection ใหม่
         ┌──────────────┼──────────────┬─────────────────┐
         ▼              ▼              ▼                 ▼
   ┌──────────┐   ┌──────────┐   ┌──────────┐    ┌──────────────┐
   │ Backend  │   │ Backend  │   │ Backend  │    │  Background   │
   │ (conn A) │   │ (conn B) │   │ (conn C) │    │  processes    │
   │ PID 2001 │   │ PID 2002 │   │ PID 2003 │    │ (checkpointer,│
   └──────────┘   └──────────┘   └──────────┘    │  bgwriter, ..)│
                                                     └──────────────┘
```

### ทดสอบดู: ทำไม postmaster ถึงสำคัญ

ลองสังเกต process tree ด้วยคำสั่ง `pstree` ซึ่งจะเห็นความสัมพันธ์แบบ parent-child ชัดเจน

```bash
$ pstree -p 1001

postgres(1001)-+-postgres(1005)   # checkpointer
                |-postgres(1006)   # background writer
                |-postgres(1007)   # walwriter
                |-postgres(1008)   # autovacuum launcher
                |-postgres(1009)   # logical replication launcher
                |-postgres(2101)   # backend: idle connection
                `-postgres(2144)   # backend: running SELECT
```

จะเห็นว่า**ทุก process ที่เกี่ยวข้องกับ PostgreSQL instance นี้เป็นลูกของ postmaster (PID 1001) ทั้งหมด** — นี่คือเหตุผลที่การ `kill -9` postmaster โดยตรงเป็นเรื่องอันตรายมาก เพราะระบบปฏิบัติการจะไม่มีใครคอย "รวบรวมซาก" (reap) child process หรือ coordinate การ shutdown อย่างปลอดภัยอีกต่อไป ในสถานการณ์เช่นนี้ backend process ที่เหลือมักจะ terminate ตัวเองเมื่อพบว่า postmaster หายไป (เพราะ PostgreSQL มีกลไกตรวจจับผ่าน shared memory) แต่ผลลัพธ์ที่ปลอดภัยที่สุดคือการใช้ `pg_ctl stop` หรือส่ง `SIGTERM` ไปที่ postmaster โดยตรงเสมอ

---

## Step 803: Backend Process — 1 connection = 1 OS process

### ความสัมพันธ์แบบ 1:1 ระหว่าง connection กับ process

นี่คือหัวใจของสถาปัตยกรรม PostgreSQL: **ทุกครั้งที่ client เชื่อมต่อสำเร็จ postmaster จะ fork() backend process ใหม่ขึ้นมาเฉพาะสำหรับ connection นั้น 1 ต่อ 1** ไม่มีการแชร์ backend process ระหว่างหลาย connection ไม่มี thread pool ภายใน backend เดียว

```
Client 1 ──── connect ────► Postmaster ──fork()──► Backend Process PID 2001
Client 2 ──── connect ────► Postmaster ──fork()──► Backend Process PID 2002
Client 3 ──── connect ────► Postmaster ──fork()──► Backend Process PID 2003
   ...                                                      ...
Client N ──── connect ────► Postmaster ──fork()──► Backend Process PID 200N
```

Backend process ที่ถูกสร้างขึ้นมานี้จะ**อยู่ตลอดอายุของ connection นั้น** — ตั้งแต่ authenticate เสร็จ ไปจนถึง client สั่ง disconnect หรือ connection ถูกตัด กระบวนการทั้งหมดที่เกิดขึ้นระหว่างนั้น (parse SQL, plan query, execute, ส่งผลลัพธ์กลับ, transaction management) เกิดขึ้นภายใน backend process เดียวนี้ทั้งหมด

เมื่อ connection ปิดลง backend process จะ**ถูกทำลาย (terminate)** ไปด้วย — มันไม่ถูกนำกลับมาใช้ใหม่สำหรับ connection ถัดไป (ต่างจาก thread pool ในบาง database ที่ reuse thread ได้) นี่คือเหตุผลสำคัญที่การเปิด-ปิด connection บ่อย ๆ มีต้นทุนสูงใน PostgreSQL (จะอธิบายเชิงลึกใน Step 809)

### สำรวจ backend process จริงด้วย ps aux

ลองเปิด 3 session พร้อมกันแล้วดูผลลัพธ์:

```bash
# Terminal 1
$ psql -h localhost -U app_user -d myapp -c "SELECT pg_sleep(60);" &

# Terminal 2
$ psql -h localhost -U app_user -d myapp -c "SELECT pg_backend_pid();"

# ดู process ทั้งหมด
$ ps aux | grep "postgres:"

postgres  3001  0.0  0.2 216204  9932 ?  Ss  10:02  0:00 postgres: myapp app_user 127.0.0.1(41200) idle
postgres  3005  0.0  0.2 216204  9944 ?  Ss  10:03  0:00 postgres: myapp app_user 127.0.0.1(41210) SELECT
postgres  3009  0.0  0.2 216100  9800 ?  Ss  10:03  0:00 postgres: myapp app_user 127.0.0.1(41220) idle
```

รูปแบบ process title ของ backend มีโครงสร้างคงที่คือ:

```
postgres: <database> <username> <client_address>(<client_port>) <state>
```

โดย `<state>` จะบอกว่า connection นั้นกำลังทำอะไรอยู่ เช่น:

| State ที่เห็นใน `ps` | ความหมาย |
|---|---|
| `idle` | connection เปิดอยู่ แต่ไม่มี transaction/query ทำงาน |
| `idle in transaction` | อยู่ใน transaction แต่ไม่มี statement กำลังรัน (อันตรายถ้าค้างนาน — ดู Part เรื่อง long-running transaction) |
| `SELECT`, `INSERT`, `UPDATE`, ... | กำลังรัน statement ประเภทนั้นอยู่ |
| `authentication` | กำลังอยู่ระหว่างขั้นตอน authenticate |

### เชื่อม PID กับ connection ผ่าน pg_stat_activity

การดู `ps aux` บอกเราแค่ระดับ OS แต่ PostgreSQL มี system view ที่ให้ข้อมูลละเอียดกว่ามาก คือ `pg_stat_activity` ซึ่งแต่ละแถวสอดคล้องกับ**backend process 1 ตัว** โดยตรง

```sql
SELECT pid, usename, datname, client_addr, state, query, backend_start
FROM pg_stat_activity
WHERE backend_type = 'client backend';
```

```
  pid  | usename  | datname |  client_addr  |        state        |          query          |         backend_start
-------+----------+---------+----------------+----------------------+---------------------------+-------------------------------
  3001 | app_user | myapp   | 127.0.0.1      | idle                 |                           | 2026-09-25 10:02:11.234+07
  3005 | app_user | myapp   | 127.0.0.1      | active               | SELECT pg_sleep(60);     | 2026-09-25 10:03:02.881+07
  3009 | app_user | myapp   | 127.0.0.1      | idle                 |                           | 2026-09-25 10:03:15.560+07
```

คอลัมน์ `pid` ในผลลัพธ์นี้**คือ PID เดียวกัน**กับที่เห็นใน `ps aux` — นี่คือหลักฐานเชิงประจักษ์ว่า `pg_stat_activity` ไม่ใช่แค่ "ตัวเลขนามธรรม" แต่สะท้อน process จริงในระบบปฏิบัติการ 1:1

ลองยืนยันด้วยตัวเอง:

```sql
-- รันคำสั่งนี้จาก session ที่กำลังจะ sleep
SELECT pg_backend_pid();
--  pg_backend_pid
-- ----------------
--            3005
```

```bash
# แล้วไปเช็คที่ OS ว่า PID นี้มีอยู่จริง
$ ps -p 3005 -o pid,ppid,stat,etime,cmd
    PID    PPID STAT     ELAPSED CMD
   3005    1001 Ss          0:12 postgres: myapp app_user 127.0.0.1(41210) SELECT
```

สังเกตคอลัมน์ `PPID` เท่ากับ `1001` ซึ่งคือ PID ของ postmaster ที่เราเห็นใน Step 802 — ยืนยันความสัมพันธ์แบบ parent-child อย่างชัดเจน

### ผลที่ตามมาจากสถาปัตยกรรมนี้

จากข้อเท็จจริงที่ว่า 1 connection = 1 process เราสามารถสรุปคุณสมบัติสำคัญได้ทันที:

- **การ kill connection ก็คือการ kill process จริง** — คำสั่ง `SELECT pg_terminate_backend(pid)` จริง ๆ แล้วคือการส่ง `SIGTERM` ไปยัง OS process นั้นโดยตรง
- **จำนวน connection สูงสุด (`max_connections`) จำกัดจำนวน process สูงสุดที่ postmaster จะ fork ได้** — นี่ไม่ใช่ arbitrary limit แต่สัมพันธ์กับ memory และ resource ของเครื่องจริง
- **แต่ละ connection ใช้ memory ของตัวเอง** สำหรับ local state (work_mem, temp buffers, query execution context) นอกเหนือจาก shared memory ที่ทุก process แชร์กัน — ซึ่งเป็นที่มาของต้นทุนต่อ connection ที่เราจะพูดถึงใน Step 808-809

```bash
# ตรวจสอบ memory ที่แต่ละ backend ใช้จริง (RSS = Resident Set Size)
$ ps -o pid,rss,vsz,cmd -p 3001,3005,3009

    PID    RSS    VSZ CMD
   3001   9932 216204 postgres: myapp app_user 127.0.0.1(41200) idle
   3005  12480 217120 postgres: myapp app_user 127.0.0.1(41210) SELECT
   3009   9800 216100 postgres: myapp app_user 127.0.0.1(41220) idle
```

จะเห็นว่าแม้ backend ที่ `idle` ก็ยังกิน memory หลายสิบ MB ของ RSS แล้ว (ส่วนหนึ่งเป็น shared memory ที่ถูก map เข้ามาซึ่งไม่ได้กินจริงต่อ process แต่ VSZ จะรวมไว้ ส่วน RSS จริงมักจะ 5-20 MB ต่อ idle connection ขึ้นกับ config) — นี่คือตัวเลขที่อธิบายว่าทำไม 1,000 connection พร้อมกันจึงเป็นภาระต่อ memory ของเครื่องอย่างมีนัยสำคัญ

---

## Step 804: Shared Memory — พื้นที่หน่วยความจำที่ทุก process เข้าถึงร่วมกัน

### ทำไมต้องมี Shared Memory

ในเมื่อ backend แต่ละตัวเป็น process แยกกัน มี memory address space ของตัวเอง แล้ว backend เหล่านี้จะรู้ได้อย่างไรว่า backend อื่นกำลังแก้ไข row เดียวกันอยู่? จะแชร์ cached data page ร่วมกันได้อย่างไร? จะรู้ transaction ID ล่าสุดร่วมกันได้อย่างไร?

คำตอบคือ **Shared Memory Segment** — พื้นที่หน่วยความจำก้อนหนึ่งที่ OS จัดสรรให้ และถูก **map เข้าไปใน address space ของทุก process ที่เกี่ยวข้อง** (postmaster และ child process ทั้งหมด) ตั้งแต่ตอน startup พื้นที่นี้จึงเป็น "พื้นที่กลาง" ที่ทุก process มองเห็นข้อมูลเดียวกัน อ่าน-เขียนข้อมูลเดียวกันได้แบบ real-time

```
        Process A                Process B                Process C
   (Backend conn 1)          (Backend conn 2)          (Checkpointer)
┌───────────────────┐    ┌───────────────────┐    ┌───────────────────┐
│  Private Memory     │    │  Private Memory     │    │  Private Memory     │
│  (local variables,  │    │  (local variables,  │    │  (local variables,  │
│   work_mem, etc.)   │    │   work_mem, etc.)   │    │   maintenance_work_mem)│
├─────────┬───────────┤    ├─────────┬───────────┤    ├─────────┬───────────┤
│         │  mapped   │    │         │  mapped   │    │         │  mapped   │
│         │  region   │    │         │  region   │    │         │  region   │
└─────────┼───────────┘    └─────────┼───────────┘    └─────────┼───────────┘
          │                          │                          │
          └──────────────────────────┼──────────────────────────┘
                                      ▼
                    ┌───────────────────────────────────────┐
                    │       SHARED MEMORY SEGMENT              │
                    │       (allocated once at server start)   │
                    │                                           │
                    │  ┌─────────────────────────────────┐    │
                    │  │  Shared Buffers (shared_buffers) │    │
                    │  │  (data page cache)                │    │
                    │  └─────────────────────────────────┘    │
                    │  ┌─────────────────────────────────┐    │
                    │  │  WAL Buffers (wal_buffers)        │    │
                    │  └─────────────────────────────────┘    │
                    │  ┌─────────────────────────────────┐    │
                    │  │  Lock Table                       │    │
                    │  └─────────────────────────────────┘    │
                    │  ┌─────────────────────────────────┐    │
                    │  │  Proc Array (all backend states) │    │
                    │  └─────────────────────────────────┘    │
                    │  ┌─────────────────────────────────┐    │
                    │  │  CLOG / commit status              │    │
                    │  └─────────────────────────────────┘    │
                    └───────────────────────────────────────┘
```

### ส่วนประกอบหลักของ Shared Memory

**1. Shared Buffers (`shared_buffers`)** — พื้นที่ที่ใหญ่ที่สุดโดยทั่วไป ใช้เก็บ cache ของ data page (8 KB block) ที่อ่านมาจาก disk เมื่อ backend หนึ่งอ่าน table page เข้ามาไว้ใน shared buffer แล้ว backend อื่นที่ต้องการ page เดียวกันจะอ่านจาก memory ได้เลยโดยไม่ต้องไป disk ซ้ำ — นี่คือกลไกพื้นฐานที่ทำให้ PostgreSQL เร็ว

**2. WAL Buffers (`wal_buffers`)** — buffer สำหรับเก็บ Write-Ahead Log record ก่อนที่จะถูก flush ลง disk จริง ทุก backend ที่ทำการเปลี่ยนแปลงข้อมูล (INSERT/UPDATE/DELETE) จะเขียน WAL record ลงในบัฟเฟอร์นี้ก่อนเสมอ

**3. Lock Table** — ตารางที่เก็บสถานะ lock ทั้งหมดของระบบ (row lock, table lock, advisory lock ฯลฯ) เมื่อ backend หนึ่งต้องการ lock resource ใด มันต้องเข้าไปตรวจสอบและจองใน lock table นี้ ซึ่งต้องเป็น shared เพราะ lock ต้องมองเห็นได้จากทุก process ที่แข่งขันกันเข้าถึง resource เดียวกัน

**4. Proc Array** — array ที่เก็บสถานะของ backend process ที่กำลัง active ทั้งหมด (transaction ID, snapshot, lock ที่ถืออยู่) ใช้สำหรับกลไก MVCC (Multi-Version Concurrency Control) — เมื่อ backend ต้องการรู้ว่า transaction ไหนกำลัง active อยู่บ้าง (เพื่อสร้าง visibility snapshot) มันจะอ่านจาก proc array นี้

**5. CLOG (Commit Log)** — เก็บสถานะว่าแต่ละ transaction commit สำเร็จ, abort หรือยังไม่เสร็จ ใช้ประกอบการตัดสินใจ visibility ของ row แต่ละ version

### ดูขนาด Shared Memory จริง

```sql
-- ดูค่า config ที่กำหนดขนาด shared memory
SHOW shared_buffers;
--  shared_buffers
-- ----------------
--  4GB

SHOW wal_buffers;
--  wal_buffers
-- -------------
--  16MB

SHOW max_connections;
--  max_connections
-- ------------------
--  200
```

```sql
-- ดู breakdown การใช้งาน shared buffer ปัจจุบัน (ต้องมี extension pg_buffercache)
CREATE EXTENSION IF NOT EXISTS pg_buffercache;

SELECT
    c.relname,
    count(*) AS buffers,
    pg_size_pretty(count(*) * 8192) AS size
FROM pg_buffercache b
JOIN pg_class c ON b.relfilenode = pg_relation_filenode(c.oid)
    AND b.reldatabase IN (0, (SELECT oid FROM pg_database WHERE datname = current_database()))
GROUP BY c.relname
ORDER BY buffers DESC
LIMIT 10;
```

```
     relname        | buffers |  size
---------------------+---------+---------
 orders               |   45210 | 353 MB
 orders_pkey          |   12300 |  96 MB
 order_items          |    8900 |  69 MB
 customers             |    3100 |  24 MB
 ...
```

```bash
# ระดับ OS: ดู shared memory segment ที่ PostgreSQL จองไว้ (Linux, System V shared memory
# หรือ POSIX shared memory ขึ้นกับ dynamic_shared_memory_type)
$ ipcs -m

------ Shared Memory Segments --------
key        shmid      owner      perms      bytes      nattch     status
0x0052e2c1 32768      postgres   600        56         6

# หรือดูผ่าน /proc สำหรับ POSIX shared memory (ค่า default สมัยใหม่คือ dynamic shared memory)
$ ls -la /dev/shm/ | grep PostgreSQL
-rw------- 1 postgres postgres 4398046511 Sep 25 08:00 PostgreSQL.2864135646
```

หมายเหตุ: ตั้งแต่ PostgreSQL 9.4 เป็นต้นมา ค่า default ของ `dynamic_shared_memory_type` เปลี่ยนมาใช้กลไกที่ยืดหยุ่นกว่า System V shared memory แบบดั้งเดิม (เช่น `posix` บน Linux) แต่หลักการพื้นฐาน "memory segment เดียวที่ทุก process map เข้าถึงร่วมกัน" ยังคงเหมือนเดิม

### ทำไมต้องล็อกเมื่อเข้าถึง shared memory

เมื่อหลาย process เข้าถึงข้อมูลเดียวกันพร้อมกัน (concurrent access) จำเป็นต้องมีกลไกป้องกัน race condition — PostgreSQL ใช้ **lightweight locks (LWLocks)** และ **spinlocks** ภายใน shared memory เพื่อป้องกันไม่ให้สอง process เขียนทับข้อมูลโครงสร้างเดียวกันพร้อมกันจนเกิดความเสียหาย เราจะพูดถึงกลไกนี้ละเอียดขึ้นใน Step 807

```sql
-- ดู LWLock ที่กำลังรอ (ถ้ามี contention)
SELECT wait_event_type, wait_event, count(*)
FROM pg_stat_activity
WHERE wait_event_type IS NOT NULL
GROUP BY 1, 2
ORDER BY 3 DESC;
```

```
 wait_event_type |      wait_event       | count
------------------+-------------------------+-------
 LWLock            | buffer_mapping          |     2
 Lock              | tuple                    |     1
```

---

## Step 805: Background Processes ที่สำคัญ — Background Writer, Checkpointer, WAL Writer

นอกจาก backend process ที่รับผิดชอบ connection ของ client แล้ว postmaster ยังดูแล **background process** อีกชุดหนึ่งที่ทำงานเบื้องหลังตลอดเวลา ไม่ได้ผูกกับ connection ใด ๆ แต่ทำหน้าที่ดูแลสุขภาพและประสิทธิภาพของทั้งระบบ

```bash
$ ps aux | grep postgres | grep -v grep

postgres  1001  0.0  0.5 215000 21344 ?  Ss  08:00  0:00 /usr/lib/postgresql/16/bin/postgres -D /var/lib/postgresql/16/main
postgres  1005  0.0  0.1 215136  4520 ?  Ss  08:00  0:00 postgres: checkpointer
postgres  1006  0.0  0.1 215000  3212 ?  Ss  08:00  0:00 postgres: background writer
postgres  1007  0.0  0.2 215368  5680 ?  Ss  08:00  0:00 postgres: walwriter
postgres  1008  0.0  0.1 215636  4108 ?  Ss  08:00  0:00 postgres: autovacuum launcher
postgres  1009  0.0  0.1  69720  3016 ?  Ss  08:00  0:00 postgres: logical replication launcher
```

### Background Writer (bgwriter)

**หน้าที่**: ค่อย ๆ เขียน (flush) "dirty page" (data page ใน shared buffer ที่ถูกแก้ไขแต่ยังไม่ถูกเขียนลง disk) ออกไปยัง disk อย่างสม่ำเสมอและ**ทีละน้อย** ในพื้นหลัง แทนที่จะรอให้ backend process ต้องเขียนเองตอนที่ shared buffer เต็มและต้องการ evict page เก่าออก

**ทำไมสำคัญ**: หากไม่มี background writer backend process ที่ต้องการ buffer ว่างจะต้องหยุดรอ (block) เพื่อเขียน dirty page ออกไปก่อนด้วยตัวเอง ซึ่งทำให้ query ของ client ช้าลงแบบ unpredictable Background writer ช่วย "เตรียมพื้นที่ว่างล่วงหน้า" ให้เสมอ ทำให้ backend process ไม่ต้องเจอ latency spike จากการ evict buffer

```
Timeline การทำงานของ Background Writer (แบบ round-robin ไล่ scan buffer):

 shared_buffers:
 ┌────┬────┬────┬────┬────┬────┬────┬────┬────┬────┐
 │clean│dirty│clean│dirty│dirty│clean│clean│dirty│clean│dirty│
 └────┴────┴────┴────┴────┴────┴────┴────┴────┴────┘
         │            │    │            │        │
         ▼            ▼    ▼            ▼        ▼
   bgwriter เขียน dirty page เหล่านี้ลง disk ทีละน้อย
   ตาม bgwriter_lru_maxpages ต่อรอบ (default 100 pages)
   ทุก bgwriter_delay (default 200ms)
```

```sql
-- config ที่ควบคุมพฤติกรรม background writer
SHOW bgwriter_delay;          -- 200ms (ความถี่ในการทำงาน)
SHOW bgwriter_lru_maxpages;   -- 100 (จำนวน page สูงสุดต่อรอบ)
SHOW bgwriter_lru_multiplier; -- 2.0 (ตัวคูณเพื่อคาดการณ์ความต้องการ buffer ล่วงหน้า)
```

```sql
-- สถิติการทำงานของ background writer
SELECT * FROM pg_stat_bgwriter;
```

```
 buffers_clean | maxwritten_clean | buffers_alloc | buffers_backend | buffers_checkpoint
----------------+--------------------+----------------+-------------------+----------------------
        1284392 |                 45 |        3821004 |            12088 |             892103
```

- `buffers_clean` — จำนวน buffer ที่ bgwriter เขียนออกไปเอง (นี่คือสิ่งที่เราต้องการเห็นสูง)
- `buffers_backend` — จำนวน buffer ที่ **backend process ต้องเขียนเอง** เพราะ bgwriter ตามไม่ทัน (ค่านี้สูงเป็นสัญญาณว่าควร tune bgwriter ให้ทำงานถี่ขึ้นหรือ aggressive ขึ้น)
- `maxwritten_clean` — จำนวนครั้งที่ bgwriter หยุดทำงานก่อนกำหนดเพราะเขียนครบ `bgwriter_lru_maxpages` แล้ว

> **หมายเหตุ**: ตั้งแต่ PostgreSQL 15 เป็นต้นไป คอลัมน์บางส่วนของ `pg_stat_bgwriter` (โดยเฉพาะที่เกี่ยวกับ checkpoint) ถูกย้ายไปอยู่ใน `pg_stat_checkpointer` ใน PostgreSQL 17 — ควรตรวจสอบ version ที่ใช้จริงก่อนอ้างอิงคอลัมน์เหล่านี้

### Checkpointer

**หน้าที่**: ทำ **checkpoint** เป็นระยะ — กระบวนการที่เขียน dirty page **ทั้งหมด**ที่ยังค้างอยู่ใน shared buffer ลง disk ให้เรียบร้อย แล้วบันทึกตำแหน่ง WAL ล่าสุดที่ checkpoint สำเร็จไว้ใน control file (`pg_control`)

**ทำไมสำคัญ**: checkpoint คือจุดอ้างอิงสำหรับ **crash recovery** — เมื่อ PostgreSQL restart หลังจาก crash มันจะเริ่ม replay WAL จากตำแหน่ง checkpoint ล่าสุดเท่านั้น (ไม่ใช่จากจุดเริ่มต้นของ WAL ทั้งหมด) ยิ่ง checkpoint ห่างกันนาน ยิ่งมี WAL ต้อง replay มากขึ้นเมื่อเกิด crash ทำให้เวลา recovery นานขึ้น แต่ถ้า checkpoint ถี่เกินไปก็จะสร้าง I/O load สูงบ่อยเกินไป — นี่คือ trade-off คลาสสิกที่ DBA ต้องจูน

```
Checkpointer ทำงานต่างจาก bgwriter อย่างไร:

Background Writer:  เขียนทีละน้อย, ต่อเนื่อง, กระจาย I/O ให้เรียบ
                     ┌─┐  ┌─┐  ┌─┐  ┌─┐  ┌─┐  ┌─┐  (small, frequent writes)

Checkpointer:        เขียน dirty page ทั้งหมดให้เสร็จภายในรอบเดียว
                     ┌───────────────────────┐   (large batch, spread over
                     │  checkpoint I/O burst  │    checkpoint_completion_target
                     └───────────────────────┘    เพื่อไม่ให้ spike แรงเกินไป)
```

```sql
-- config ที่ควบคุม checkpoint
SHOW checkpoint_timeout;             -- 5min (checkpoint ตามเวลา)
SHOW max_wal_size;                   -- 1GB (checkpoint ตามปริมาณ WAL ที่สร้าง)
SHOW checkpoint_completion_target;   -- 0.9 (กระจาย I/O ให้เสร็จภายใน 90% ของช่วงเวลา)
```

```sql
-- สถิติ checkpoint (PostgreSQL 17+ ใช้ pg_stat_checkpointer,
-- เวอร์ชันก่อนหน้าอยู่ใน pg_stat_bgwriter)
SELECT num_timed, num_requested, write_time, sync_time, buffers_written
FROM pg_stat_checkpointer;
```

```
 num_timed | num_requested | write_time | sync_time | buffers_written
------------+-----------------+-------------+------------+-------------------
       1220 |              38 |   842103.5 |   12044.2 |           2145890
```

- `num_timed` — checkpoint ที่เกิดจากครบเวลา (`checkpoint_timeout`) — เป็นปกติ
- `num_requested` — checkpoint ที่เกิดจากครบปริมาณ WAL (`max_wal_size`) ก่อนถึงเวลา — ถ้าค่านี้สูงมากเทียบกับ `num_timed` แปลว่าระบบเขียนข้อมูลเยอะจนต้อง checkpoint ถี่กว่าที่ตั้งใจ ควรพิจารณาเพิ่ม `max_wal_size`

### WAL Writer (walwriter)

**หน้าที่**: เขียน WAL record จาก WAL buffer (ใน shared memory) ออกไปยัง disk (WAL segment file ใน `pg_wal/`) อย่างสม่ำเสมอในพื้นหลัง

**ทำไมสำคัญ**: แม้ backend process ที่ commit transaction จะต้อง `fsync` WAL ของตัวเองอยู่แล้ว (เพื่อรับประกัน durability ตาม ACID) แต่ walwriter ช่วยลด **จำนวนรอบการเขียน WAL buffer to disk ที่ไม่จำเป็น** โดยการ flush เป็นระยะ (ทุก `wal_writer_delay`) ทำให้ transaction ที่ commit พร้อม ๆ กันได้ประโยชน์จากการ "รวมยอด" การเขียน WAL (group commit) แทนที่แต่ละ transaction จะต้อง fsync แยกกันเอง

```sql
SHOW wal_writer_delay;        -- 200ms
SHOW wal_writer_flush_after;  -- 1MB
```

```
┌──────────────────────────────────────────────────────────────┐
│                    3 Background Processes นี้ทำงานร่วมกัน            │
├──────────────────────────────────────────────────────────────┤
│                                                                  │
│  Background Writer:  ดูแล data page dirty buffer → เขียนออก      │
│                       ทีละน้อย ต่อเนื่อง                          │
│                                                                  │
│  Checkpointer:        ทำ full flush เป็นระยะ → สร้างจุด recovery    │
│                       ที่แน่นอน                                   │
│                                                                  │
│  WAL Writer:          ดูแล WAL buffer → เขียน WAL log ออก         │
│                       เพื่อความ durable และลด fsync ซ้ำซ้อน         │
│                                                                  │
│  ทั้ง 3 ตัวทำงานเป็น "แรงงานเบื้องหลัง" ที่ทำให้ backend process      │
│  ของ client ไม่ต้องแบกภาระ I/O เองโดยตรงทุกครั้ง                    │
└──────────────────────────────────────────────────────────────┘
```

---

## Step 806: Background Processes ต่อ — Autovacuum, Stats Collector/Cumulative Stats, Logical Replication Launcher

### Autovacuum Launcher และ Autovacuum Worker

**Autovacuum Launcher** เป็น background process ที่รันตลอดเวลา (คล้าย checkpointer) แต่**ไม่ได้ทำงาน vacuum เอง** — หน้าที่ของมันคือคอยตรวจสอบสถิติของแต่ละตารางในทุก database (จำนวน row ที่ตายแล้ว/dead tuple, transaction ID age) แล้วตัดสินใจว่าถึงเวลาต้อง vacuum ตารางไหนแล้ว จากนั้นจะสั่ง**fork Autovacuum Worker process ใหม่**ขึ้นมาทำงานจริง

```bash
$ ps aux | grep autovacuum

postgres  1008  0.0  0.1 215636  4108 ?  Ss  08:00  0:00 postgres: autovacuum launcher
postgres  4501  0.5  0.3 216800 10200 ?  Rs  10:20  0:01 postgres: autovacuum worker myapp
```

**ทำไมสำคัญ**: PostgreSQL ใช้ MVCC ซึ่งหมายความว่าเมื่อ UPDATE หรือ DELETE row จะไม่ถูกลบทิ้งทันทีแต่ถูกทำเครื่องหมายว่า "ตายแล้ว" (dead tuple) และ row version เก่ายังคงอยู่ในตารางจนกว่าจะไม่มี transaction ใดต้องการเห็นมันอีก VACUUM คือกระบวนการที่เข้ามาเก็บกวาด dead tuple เหล่านี้คืนพื้นที่ให้ระบบใช้ใหม่ได้ และยังทำหน้าที่ป้องกัน **transaction ID wraparound** ซึ่งเป็นปัญหาระดับร้ายแรงถ้าไม่ได้รับการจัดการ (รายละเอียดเชิงลึกจะอยู่ในบทถัดไปเรื่อง MVCC internals)

```sql
-- config ที่ควบคุม autovacuum
SHOW autovacuum;                        -- on
SHOW autovacuum_max_workers;            -- 3 (จำนวน worker พร้อมกันสูงสุด)
SHOW autovacuum_naptime;                -- 1min (ความถี่ที่ launcher ตรวจสอบ)
SHOW autovacuum_vacuum_scale_factor;    -- 0.2 (เกณฑ์ % dead tuple ที่ trigger vacuum)
```

```sql
-- ดูว่า table ไหนกำลังถูก autovacuum ทำงานอยู่ตอนนี้
SELECT pid, datname, relid::regclass AS table_name, phase,
       heap_blks_total, heap_blks_scanned, heap_blks_vacuumed
FROM pg_stat_progress_vacuum;
```

```
  pid  | datname | table_name |        phase        | heap_blks_total | heap_blks_scanned | heap_blks_vacuumed
--------+---------+------------+-----------------------+-------------------+----------------------+----------------------
  4501 | myapp   | orders     | vacuuming heap        |            52000 |               31200 |               28900
```

```sql
-- ดูว่า table ไหนถูก vacuum ล่าสุดเมื่อไร และมี dead tuple ค้างเท่าไร
SELECT relname, n_dead_tup, n_live_tup, last_autovacuum, autovacuum_count
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC
LIMIT 5;
```

```
   relname    | n_dead_tup | n_live_tup |       last_autovacuum        | autovacuum_count
---------------+-------------+-------------+--------------------------------+--------------------
 orders         |       48210 |     1204500 | 2026-09-25 09:55:12.100+07     |             1822
 order_items    |       12300 |     3402100 | 2026-09-25 08:40:03.552+07     |             1590
```

### Stats Collector (PostgreSQL 14 และเก่ากว่า) vs Cumulative Statistics (PostgreSQL 15+)

**ในเวอร์ชัน PostgreSQL 14 และก่อนหน้า** มี background process แยกต่างหากชื่อ **stats collector** ทำหน้าที่รวบรวมสถิติการใช้งาน (จำนวน row ที่ scan, จำนวน index ที่ใช้, cache hit ratio ฯลฯ) จากทุก backend process แล้วเขียนเก็บไว้ในไฟล์ชั่วคราว วิธีนี้มีข้อเสียคือ backend ต้องส่งข้อมูลผ่าน UDP socket ไปให้ stats collector และ stats collector ต้องเขียนไฟล์ลง disk เป็นระยะ (ค่า default ทุก `stats_temp_directory`) ซึ่งมี overhead และ latency ในการอัปเดตค่าที่เห็น

**ตั้งแต่ PostgreSQL 15 เป็นต้นไป** ระบบสถิติถูกออกแบบใหม่ทั้งหมดเป็น **cumulative statistics system** ที่เก็บข้อมูลสถิติไว้ **ใน shared memory โดยตรง** ไม่มี stats collector process แยกต่างหากอีกต่อไป และไม่มีการเขียนไฟล์ temp file เป็นระยะเหมือนเดิม (ยกเว้นตอน shutdown ที่จะ dump ลง disk เพื่อให้ restart แล้วยังใช้สถิติเก่าต่อได้)

```bash
# PostgreSQL 14 หรือเก่ากว่า: จะเห็น stats collector process
postgres  1010  0.0  0.1 215050  3800 ?  Ss  08:00  0:00 postgres: stats collector

# PostgreSQL 15 ขึ้นไป: ไม่มี process นี้อีกต่อไป
# (ตรวจสอบด้วย ps aux | grep postgres จะไม่พบ "stats collector")
```

```sql
-- ตรวจสอบ version เพื่อเข้าใจว่าระบบของเรามี stats collector process หรือไม่
SELECT version();
--  PostgreSQL 16.4 ...

-- pg_stat_activity ยังคงมองเห็นข้อมูลเหมือนเดิม เพราะ view ระดับ SQL ไม่เปลี่ยน
-- แต่ backend_type จะไม่มีแถวของ "stats collector" อีกต่อไปใน PG15+
SELECT DISTINCT backend_type FROM pg_stat_activity;
```

```
      backend_type
--------------------------
 client backend
 checkpointer
 background writer
 walwriter
 autovacuum launcher
 logical replication launcher
```

การเปลี่ยนแปลงนี้เป็นตัวอย่างที่ดีว่า internals ของ PostgreSQL ยังคงพัฒนาต่อเนื่อง — ผู้เชี่ยวชาญต้องรู้ทัน version-specific behavior เหล่านี้ ไม่ใช่จำสถาปัตยกรรมแบบตายตัวจากเวอร์ชันเดียว

### Logical Replication Launcher

**หน้าที่**: ดูแลและ fork **logical replication worker process** สำหรับ subscription แต่ละตัวที่ตั้งค่าไว้ในระบบ (เมื่อ database นี้เป็น subscriber ของ logical replication — ทบทวนเรื่อง replication ในบทก่อนหน้า) worker process แต่ละตัวจะเชื่อมต่อไปยัง publisher และรับ change stream มา apply กับ local table

```bash
$ ps aux | grep -E "logical|walsender|walreceiver"

postgres  1009  0.0  0.1  69720  3016 ?  Ss  08:00  0:00 postgres: logical replication launcher
postgres  5201  0.1  0.3 216900 11400 ?  Ss  09:00  0:00 postgres: logical replication worker for subscription 16401
postgres  5300  0.0  0.2 216500  8900 ?  Ss  09:00  0:00 postgres: walsender replicator_user 10.0.2.5(52100) streaming 0/3A2F1C0
```

```sql
-- ดู subscription ที่กำลังทำงาน (ต้องรันบน subscriber database)
SELECT subname, pid, received_lsn, latest_end_lsn, latest_end_time
FROM pg_stat_subscription;
```

```
        subname        |  pid  | received_lsn |  latest_end_lsn |        latest_end_time
-------------------------+-------+----------------+--------------------+---------------------------------
 sub_orders_to_analytics |  5201 |  0/3A2F1C0    |  0/3A2F1C0        | 2026-09-25 10:32:01.221+07
```

### สรุปตารางรวม Background Processes

| Process | หน้าที่หลัก | Process title ที่เห็นใน `ps` |
|---|---|---|
| Postmaster | Listen + fork + monitor + shutdown coordination | `postgres -D <datadir>` |
| Background Writer | เขียน dirty page ทีละน้อยอย่างต่อเนื่อง | `postgres: background writer` |
| Checkpointer | ทำ full checkpoint เป็นระยะเพื่อจุด recovery | `postgres: checkpointer` |
| WAL Writer | เขียน WAL buffer ลง disk เป็นระยะ | `postgres: walwriter` |
| Autovacuum Launcher | คอยตัดสินใจว่าต้อง vacuum table ไหน | `postgres: autovacuum launcher` |
| Autovacuum Worker | ทำ vacuum จริงบน table ที่ถูกเลือก (ชั่วคราว) | `postgres: autovacuum worker <db>` |
| Logical Replication Launcher | คอย fork worker สำหรับแต่ละ subscription | `postgres: logical replication launcher` |
| Stats Collector (PG ≤14 เท่านั้น) | รวบรวมสถิติการใช้งานจากทุก backend | `postgres: stats collector` |

---

## Step 807: การสื่อสารระหว่าง process — Shared Memory, Semaphore, Signal

เมื่อสถาปัตยกรรมเป็นแบบหลาย process แยกกัน (ต่างจาก thread ที่แชร์ address space โดยธรรมชาติ) PostgreSQL ต้องมีกลไกเฉพาะเพื่อให้ process เหล่านี้**สื่อสารและประสานงานกัน** กลไกหลักมี 3 อย่าง:

### 1. Shared Memory (ข้อมูลที่แชร์)

ตามที่อธิบายใน Step 804 — shared memory คือ "กระดานข้อมูลกลาง" ที่ทุก process อ่าน-เขียนร่วมกันได้ แต่การมี memory ที่แชร์กันเพียงอย่างเดียวยังไม่พอ เพราะถ้าสอง process เขียนพร้อมกันโดยไม่มีการประสานงาน ข้อมูลจะเสียหาย (race condition) — จึงต้องมีกลไกที่สอง

### 2. Locks: Spinlock และ LWLock (การป้องกันการเข้าถึงพร้อมกัน)

PostgreSQL ใช้ locking primitive สองระดับภายใน shared memory:

- **Spinlock**: ใช้ป้องกันโครงสร้างข้อมูลขนาดเล็กมากที่ต้องเข้าถึงเร็วมาก (เช่น การอัปเดตตัวแปรตัวเดียว) process ที่รอ spinlock จะ "วนรอ" (busy-wait) สั้น ๆ แทนที่จะ sleep เพราะคาดว่า lock จะถูกปล่อยเร็วมาก
- **LWLock (Lightweight Lock)**: ใช้ป้องกันโครงสร้างข้อมูลที่ใหญ่ขึ้นและอาจถือ lock นานกว่า เช่น buffer mapping table, WAL insertion lock — ถ้า process รอ LWLock นานเกินไปมันจะ sleep แทนที่จะ busy-wait เพื่อไม่ให้เปลือง CPU

```sql
-- สังเกต LWLock contention ผ่าน pg_stat_activity แบบ real-time
SELECT pid, wait_event_type, wait_event, state, query
FROM pg_stat_activity
WHERE wait_event_type = 'LWLock';
```

```
  pid  | wait_event_type |     wait_event      | state  |              query
--------+-------------------+------------------------+---------+-----------------------------------
  4102 | LWLock            | WALWrite               | active | INSERT INTO orders VALUES (...)
  4110 | LWLock            | buffer_mapping          | active | SELECT * FROM orders WHERE id = 5
```

### 3. Semaphore (การประสานจังหวะและ blocking)

**Semaphore** เป็นกลไกระดับ OS ที่ PostgreSQL ใช้เมื่อ process หนึ่งต้อง**รอ**เหตุการณ์จาก process อื่นแบบ blocking (คือหยุดทำงานจริง ไม่ใช่ busy-wait) เช่น เมื่อ backend process A ต้องการ row lock ที่ backend process B ถืออยู่ A จะไปรอที่ semaphore จน B ปล่อย lock แล้ว OS จะปลุก (wake up) A ให้ทำงานต่อ วิธีนี้ประหยัด CPU กว่าการวนรอตลอดเวลา เพราะ process ที่รออยู่จะไม่กิน CPU cycle เลยระหว่างรอ

```
┌─────────────────────────────────────────────────────────────┐
│         ตัวอย่าง: Backend A กำลังรอ Row Lock จาก Backend B         │
└─────────────────────────────────────────────────────────────┘

Backend B:  UPDATE orders SET status = 'shipped' WHERE id = 100;
            (ยังไม่ COMMIT — ถือ row lock ของ id = 100 อยู่)

Backend A:  UPDATE orders SET status = 'cancelled' WHERE id = 100;
            │
            ▼
       ตรวจสอบ Lock Table (shared memory) → พบว่า row 100 ถูกล็อกโดย B
            │
            ▼
       Backend A ไปรอที่ Semaphore (block, ไม่กิน CPU)
            │
            │    (Backend B ทำ COMMIT หรือ ROLLBACK)
            ▼
       OS ปลุก Backend A ผ่าน Semaphore signal
            │
            ▼
       Backend A ตรวจสอบ Lock Table อีกครั้ง → lock ว่างแล้ว → ดำเนินการต่อ
```

```sql
-- ดู lock ที่กำลังรอกันอยู่จริง (blocking chain)
SELECT
    blocked.pid AS blocked_pid,
    blocked.query AS blocked_query,
    blocking.pid AS blocking_pid,
    blocking.query AS blocking_query
FROM pg_stat_activity blocked
JOIN pg_locks bl ON bl.pid = blocked.pid AND NOT bl.granted
JOIN pg_locks kl ON kl.locktype = bl.locktype
    AND kl.database IS NOT DISTINCT FROM bl.database
    AND kl.relation IS NOT DISTINCT FROM bl.relation
    AND kl.page IS NOT DISTINCT FROM bl.page
    AND kl.tuple IS NOT DISTINCT FROM bl.tuple
    AND kl.granted
JOIN pg_stat_activity blocking ON blocking.pid = kl.pid
WHERE blocked.pid != blocking.pid;
```

```
 blocked_pid |               blocked_query                | blocking_pid |                blocking_query
--------------+-----------------------------------------------+---------------+-----------------------------------------
        4102 | UPDATE orders SET status='cancelled' WHERE... |          4098 | UPDATE orders SET status='shipped' WHERE...
```

### 4. Signal (การสื่อสาร event ระดับ OS)

**Signal** คือกลไกของ OS เองที่ใช้แจ้งเหตุการณ์ระดับ process (เช่น "จง terminate ตัวเอง", "จง reload config") ไม่เหมือน shared memory/lock/semaphore ที่ใช้ synchronize การเข้าถึงข้อมูล — signal ใช้สำหรับสั่งการ (control) process โดยตรงจากภายนอก postmaster ใช้ signal เพื่อสื่อสารกับ child process ทั้งหมด และ admin/DBA ก็ใช้ signal สั่ง postmaster ได้เช่นกัน

| Signal | ความหมายใน PostgreSQL |
|---|---|
| `SIGTERM` | Smart/graceful shutdown — รอ transaction ปัจจุบันเสร็จก่อนค่อยปิด |
| `SIGINT` | Fast shutdown — rollback transaction ที่ค้างอยู่แล้วปิดทันที |
| `SIGQUIT` | Immediate shutdown — ปิดทันทีโดยไม่ cleanup (เหมือน crash) |
| `SIGHUP` | Reload configuration file โดยไม่ต้อง restart process |
| `SIGUSR1` | ใช้ภายในเพื่อแจ้งเหตุการณ์ระหว่าง process (เช่น การแจ้ง background worker) |
| `SIGUSR2` | ใช้ภายในเช่นกัน เช่น แจ้ง promote สำหรับ standby server |

```bash
# ทดลองส่ง SIGHUP เพื่อ reload config โดยไม่ต้อง restart server
$ kill -HUP 1001    # 1001 คือ PID ของ postmaster

# หรือใช้คำสั่งที่ปลอดภัยกว่า (เทียบเท่ากัน)
$ pg_ctl reload -D /var/lib/postgresql/16/main
server signalled

# หรือสั่งผ่าน SQL โดยตรง (ต้องมีสิทธิ์ superuser หรือ pg_signal_backend)
$ psql -c "SELECT pg_reload_conf();"
```

```bash
# ทดลอง terminate backend process หนึ่งด้วย signal โดยตรงระดับ OS
# (ไม่แนะนำในการทำงานจริง ควรใช้ pg_terminate_backend() แทนเสมอ)
$ kill -TERM 4102

# วิธีที่ถูกต้องและปลอดภัยกว่า: ให้ PostgreSQL จัดการผ่าน SQL
$ psql -c "SELECT pg_terminate_backend(4102);"
```

> **ข้อควรระวัง**: การใช้ `kill -9` (`SIGKILL`) กับ backend process ใด ๆ ของ PostgreSQL **ไม่ควรทำเด็ดขาด** เพราะ `SIGKILL` ไม่สามารถถูก handle โดยโปรแกรมได้ — process จะถูกฆ่าทันทีโดยไม่มีโอกาส cleanup shared memory state ของตัวเอง เมื่อ postmaster ตรวจพบว่า process ถูกฆ่าแบบผิดปกติ (ไม่ใช่ exit อย่างสะอาด) มันจะสันนิษฐานว่า **shared memory อาจเสียหาย (corrupted)** และจะสั่งให้**ทุก backend process อื่น disconnect ทันที** แล้วเข้าสู่ crash recovery ทั้งระบบ — นี่คือเหตุผลสำคัญที่ทำให้เข้าใจว่าทำไมต้องใช้ `pg_terminate_backend()` หรือ `pg_cancel_backend()` แทนการ `kill -9` โดยตรงเสมอ

---

## Step 808: ข้อดีข้อเสียของ Process-based Architecture เทียบกับ Thread-based

หลังจากเข้าใจกลไกภายในแล้ว เรามาสรุปเปรียบเทียบเชิงวิศวกรรมระหว่างสองแนวทางนี้อย่างเป็นระบบ

### ตารางเปรียบเทียบ

| ประเด็น | Process-based (PostgreSQL) | Thread-based (MySQL, SQL Server) |
|---|---|---|
| Isolation เมื่อเกิด crash | ระดับหนึ่ง: backend ที่ crash ส่งสัญญาณผ่าน shared memory ทำให้ postmaster ต้อง reset connection ทั้งหมด (ไม่ล่มทั้ง server แต่กระทบทุก connection ชั่วขณะ) | Thread หนึ่ง crash → ทั้ง process ล่มได้ทันที (ทุก connection หายพร้อมกัน) |
| ต้นทุน memory ต่อ connection | สูงกว่า (ต้อง allocate memory ระดับ process, page table, ฯลฯ) | ต่ำกว่า (thread เบากว่า process มาก) |
| ความเร็วในการสร้าง connection ใหม่ | ช้ากว่า (ต้อง `fork()` process ใหม่ทั้งหมด) | เร็วกว่า (สร้าง thread ใหม่เบากว่ามาก) |
| การ debug ด้วยเครื่องมือ OS | ง่าย (`ps`, `gdb`, `strace` ต่อ PID ได้ตรงตัว) | ซับซ้อนกว่า (ต้องแยกแยะ thread ID ภายใน process เดียว) |
| การ scale จำนวน connection พร้อมกัน | จำกัดด้วย memory และ context-switch overhead ของ OS scheduler | ปรับตัวได้ดีกว่าในทางทฤษฎี เพราะ thread เบากว่า |
| ความปลอดภัยจาก memory corruption ข้าม connection | สูงกว่า (OS memory protection แยก address space จริง) | ต่ำกว่า (bug ใน thread หนึ่งอาจเขียนทับ memory ของ thread อื่นได้ถ้าไม่ระวัง) |
| ความซับซ้อนของโค้ดภายใน (สำหรับผู้พัฒนา DB เอง) | ต้องจัดการ IPC (shared memory, lock, semaphore) อย่างระมัดระวัง | ต้องจัดการ thread synchronization (mutex, condition variable) อย่างระมัดระวังเช่นกัน แต่การแชร์ข้อมูลง่ายกว่าโดยธรรมชาติ |

### กรณีศึกษา: 1 connection crash ไม่ล้มทั้งระบบ (ในทางปฏิบัติ)

สมมติว่า extension หนึ่งมี bug ที่ทำให้เกิด `segmentation fault` เมื่อ backend process รันฟังก์ชันบางอย่าง เกิดอะไรขึ้น?

```
1. Backend process (PID 4200) รัน query ที่ trigger bug → segfault
                    │
                    ▼
2. OS ส่ง SIGSEGV ให้ process 4200 → process นี้ terminate ทันที
                    │
                    ▼
3. Postmaster (parent) ตรวจพบผ่าน wait() syscall ว่า child process
   terminate ผิดปกติ (ไม่ใช่ exit code 0 แบบปกติ)
                    │
                    ▼
4. Postmaster สันนิษฐานว่า shared memory (เช่น lock table,
   shared buffer) อาจอยู่ในสถานะไม่สมบูรณ์ เพราะ process 4200
   อาจตายกลางคันขณะกำลังแก้ไขโครงสร้างข้อมูลที่แชร์อยู่
                    │
                    ▼
5. Postmaster สั่ง terminate ทุก backend process ที่เหลือ (SIGQUIT)
   เพื่อความปลอดภัย แล้วรัน crash recovery (WAL replay)
   จาก checkpoint ล่าสุด
                    │
                    ▼
6. Server กลับมาพร้อมใช้งานใหม่ ข้อมูลไม่เสียหาย (เพราะ WAL)
   แต่ทุก connection ที่เปิดค้างอยู่จะถูกตัดและต้องเชื่อมต่อใหม่
```

จะเห็นว่าข้อความ "1 connection crash ไม่ล้มทั้งระบบ" ต้องเข้าใจอย่างละเอียด — **ข้อมูลไม่หายและ server ฟื้นตัวได้เองโดยอัตโนมัติ** (ต่างจาก thread-based ที่อาจถึงขั้น process หลักล่มและต้อง restart ด้วยมือ หรือข้อมูลใน memory หายไปกับ thread ที่ crash) แต่ในทางปฏิบัติ**connection อื่นที่เปิดค้างอยู่ในขณะนั้นจะถูกตัดชั่วขณะเช่นกัน** เพื่อความปลอดภัยของ shared memory — นี่คือความละเอียดที่ผู้เชี่ยวชาญต้องเข้าใจ ไม่ใช่แค่ท่องจำประโยคว่า "process แยกกันจึงปลอดภัยกว่า" แบบผิวเผิน

ทดสอบดูใน log จริง:

```
2026-09-25 10:45:12.881 +07 [1001] LOG:  server process (PID 4200) was terminated by signal 11: Segmentation fault
2026-09-25 10:45:12.881 +07 [1001] DETAIL:  Failed process was running: SELECT broken_extension_function(id) FROM orders WHERE id = 100;
2026-09-25 10:45:12.882 +07 [1001] LOG:  terminating any other active server processes
2026-09-25 10:45:12.902 +07 [4102] WARNING:  terminating connection because of crash of another server process
2026-09-25 10:45:12.902 +07 [4102] DETAIL:  The postmaster has commanded this server process to roll back the current transaction and exit, because another server process exited abnormally and possibly corrupted shared memory.
2026-09-25 10:45:13.120 +07 [1001] LOG:  all server processes terminated; reinitializing
2026-09-25 10:45:13.310 +07 [1001] LOG:  database system was interrupted; last known up at 2026-09-25 10:40:00
2026-09-25 10:45:13.450 +07 [1001] LOG:  database system was not properly shut down; automatic recovery in progress
2026-09-25 10:45:13.520 +07 [1001] LOG:  redo starts at 0/3A100000
2026-09-25 10:45:13.610 +07 [1001] LOG:  redo done at 0/3A2F1C0
2026-09-25 10:45:13.650 +07 [1001] LOG:  database system is ready to accept connections
```

นี่คือหลักฐานที่แสดงกลไก "safety net" ของสถาปัตยกรรม process-based อย่างเป็นรูปธรรม

### ต้นทุนที่ต้องจ่าย: Memory Overhead ต่อ Connection

```sql
-- ประมาณการ memory overhead จากจำนวน connection สูงสุด
SHOW max_connections;
--  200

-- ตัวเลขคร่าว ๆ (ขึ้นกับ workload): แต่ละ backend ใช้ memory ส่วนตัวประมาณ
-- 5-10 MB (idle) ไปจนถึงหลักสิบ MB (กำลังรัน query ซับซ้อนที่ใช้ work_mem เต็ม)
-- 200 connections * 10 MB (average) = ~2 GB เฉพาะ overhead ของ backend process
-- (ไม่รวม shared_buffers ที่เป็นก้อนเดียวกันสำหรับทุก process)
```

```bash
# วัดจริง: รวม RSS memory ของทุก backend process
$ ps -C postgres -o rss= | awk '{sum+=$1} END {print sum/1024 " MB total RSS across all postgres processes"}'
1842.5 MB total RSS across all postgres processes
```

ตัวเลขนี้เองคือแรงผลักดันสำคัญที่ทำให้เกิดเครื่องมือ **connection pooling** ซึ่งเราจะเชื่อมโยงกันอย่างละเอียดใน Step ถัดไป

---

## Step 809: เชื่อมโยงกลับไปทำไมต้อง Connection Pooling

ใน Part 066 เราเคยเรียนเรื่อง Connection Pooling (PgBouncer, PgPool-II) ในมุมของ "แนวทางปฏิบัติที่ดี" — แต่ตอนนี้เรามีความรู้พอที่จะเข้าใจ**กลไกจริงเบื้องหลัง**ว่าทำไมมันถึงจำเป็นสำหรับ PostgreSQL โดยเฉพาะ (มากกว่า database บางตัวที่ thread-based)

### ทบทวน: การเปิด connection ใหม่แต่ละครั้งมีต้นทุนอะไรบ้าง

จากทุกสิ่งที่เราเรียนมาในบทนี้ ลองไล่ดูว่าเมื่อ client เปิด connection ใหม่ 1 ครั้ง ระบบต้องทำอะไรบ้าง:

```
┌─────────────────────────────────────────────────────────────┐
│              ต้นทุนที่แท้จริงของการเปิด Connection ใหม่ 1 ครั้ง       │
└─────────────────────────────────────────────────────────────┘

1. TCP handshake (network layer) — ต้นทุนนี้เหมือนกันทุก database

2. Postmaster เรียก fork() เพื่อสร้าง OS process ใหม่ทั้งหมด
   ├─ Copy process metadata (page table, file descriptor table)
   ├─ Allocate private memory เริ่มต้นสำหรับ process ใหม่
   └─ Map shared memory segment เข้ากับ address space ใหม่นี้
      (ต้นทุนนี้ "แพง" กว่าการสร้าง thread ใหม่หลายเท่า)

3. Backend process ใหม่ทำ authentication
   ├─ ตรวจสอบ pg_hba.conf
   ├─ ตรวจสอบ password/certificate/GSSAPI/ฯลฯ ตาม auth method
   └─ สร้าง session state เริ่มต้น

4. Backend process โหลด catalog cache เบื้องต้น
   ├─ อ่าน system catalog (pg_class, pg_attribute, ฯลฯ) บางส่วน
   │  เพื่อเตรียมสำหรับ query แรกที่จะมา
   └─ Cache นี้เป็น per-process (ไม่ shared!) ดังนั้นทุก connection
      ใหม่ต้องเริ่มสร้าง catalog cache ของตัวเองใหม่หมด แม้ query
      จะซ้ำกับ connection อื่นก็ตาม

5. Backend process พร้อมรับคำสั่งแรก
   (โดยรวมแล้ว fork() + authenticate + cache warmup มักใช้เวลา
    ระดับ millisecond ถึงหลักสิบ millisecond ต่อ connection —
    ฟังดูน้อย แต่ถ้าแอปพลิเคชันเปิด-ปิด connection บ่อย ๆ
    (เช่นทุก HTTP request) ต้นทุนนี้จะสะสมมหาศาล)
```

### เปรียบเทียบตัวเลขจริง

```bash
# ทดสอบง่าย ๆ: วัดเวลาที่ใช้ในการเปิด connection ใหม่แต่ละครั้ง
$ time psql -h localhost -U app_user -d myapp -c "SELECT 1;" > /dev/null

real    0m0.028s
user    0m0.008s
sys     0m0.004s
```

```bash
# เทียบกับการรัน query ผ่าน connection ที่เปิดค้างไว้แล้ว (ผ่าน pgbench persistent connection)
$ pgbench -h localhost -U app_user -d myapp -c 10 -j 2 -T 10 -S
starting vacuum...end.
transaction type: <builtin: select only>
scaling factor: 1
query mode: simple
number of clients: 10
number of threads: 2
duration: 10 s
number of transactions actually processed: 284210
latency average = 0.352 ms
tps = 28421.234567 (without initial connection time)
```

จาก 28 ms (เปิด connection ใหม่ทุกครั้ง) เทียบกับ 0.35 ms (ใช้ connection ที่เปิดอยู่แล้ว) — ความแตกต่างคือ **~80 เท่า** นี่คือหลักฐานเชิงตัวเลขที่ชัดเจนว่าทำไมแอปพลิเคชันที่เปิด-ปิด connection บ่อย (เช่น serverless function, short-lived script) จึงเจอปัญหา performance รุนแรงถ้าไม่มี connection pooling

### ทำไม Connection Pooler ถึงแก้ปัญหานี้ได้

```
┌──────────────────────────────────────────────────────────────┐
│              ไม่มี Connection Pooling                           │
│                                                                  │
│  App Request 1 ──► fork() backend ──► query ──► close ──► process ตาย │
│  App Request 2 ──► fork() backend ──► query ──► close ──► process ตาย │
│  App Request 3 ──► fork() backend ──► query ──► close ──► process ตาย │
│                                                                  │
│  ทุก request จ่ายต้นทุน fork() + auth + cache warmup ใหม่หมด        │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│              มี Connection Pooling (PgBouncer, transaction mode)  │
│                                                                  │
│  App Request 1 ──► PgBouncer ──► reuse backend (PID 4001) ──► query │
│  App Request 2 ──► PgBouncer ──► reuse backend (PID 4001) ──► query │
│  App Request 3 ──► PgBouncer ──► reuse backend (PID 4002) ──► query │
│                                                                  │
│  Backend process (PID 4001, 4002) ถูกสร้างขึ้นแค่ครั้งเดียว         │
│  แล้ว "ยืม-คืน" หมุนเวียนใช้ซ้ำ ไม่ต้อง fork() ใหม่ทุกครั้ง          │
│  → หลีกเลี่ยงต้นทุน fork() + auth ซ้ำ ๆ ได้เกือบทั้งหมด               │
└──────────────────────────────────────────────────────────────┘
```

PgBouncer เองก็เป็นโปรแกรมที่ **รักษา pool ของ backend connection ที่เปิดค้างไว้กับ PostgreSQL จริง** แล้วให้ client (application) เชื่อมต่อเข้ามาที่ PgBouncer แทน — PgBouncer จะ "ยืม" backend connection ที่มีอยู่แล้วให้ client ใช้ชั่วคราว (ตาม pooling mode: session/transaction/statement) แล้วคืนกลับเข้า pool เมื่อใช้เสร็จ โดยที่ **backend process ฝั่ง PostgreSQL แทบไม่ต้องถูกสร้างใหม่บ่อย ๆ เลย**

นี่คือจุดที่ความรู้ internals กับ best practice มาบรรจบกัน: PgBouncer ไม่ได้ "วิเศษ" อะไร มันแค่หลีกเลี่ยงขั้นตอนที่แพงที่สุดของสถาปัตยกรรม process-based คือ `fork()` process ใหม่ทุกครั้ง — เพราะเข้าใจ trade-off ของ architecture design ตั้งแต่ต้น

### ข้อควรระวัง: max_connections กับ memory ที่แท้จริง

```sql
-- คำนวณ memory overhead สูงสุดที่เป็นไปได้ถ้าเปิดทุก connection พร้อมกัน
SHOW max_connections;    -- 200
SHOW shared_buffers;     -- 4GB (shared, ไม่คูณตาม connection)
SHOW work_mem;           -- 64MB (ต่อ operation ต่อ connection, อาจคูณหลายเท่าถ้า query ซับซ้อน)

-- ประมาณการ worst-case memory (แบบคร่าว ๆ เพื่อความเข้าใจ ไม่ใช่สูตรแม่นยำ)
-- worst_case = shared_buffers + (max_connections * (per_backend_overhead + work_mem * concurrent_sort_ops))
```

การตั้ง `max_connections` สูงเกินความจำเป็น (เช่น 1000) โดยไม่มี connection pooler อยู่หน้า PostgreSQL จึงเป็นความเสี่ยงเชิง memory และ context-switch overhead อย่างมีนัยสำคัญ — นี่คือเหตุผลเชิงกลไกที่สนับสนุนคำแนะนำใน Part 066 ว่า **"ตั้ง max_connections ให้พอเหมาะ แล้วใช้ connection pooler จัดการ concurrency ของ client จำนวนมากแทน"**

---

## Step 810: แบบฝึกหัดรวม — สำรวจ Process ของ PostgreSQL ในเครื่องจริง

ในหัวข้อสุดท้ายนี้ เราจะรวบรวมทุกสิ่งที่เรียนมาเข้าด้วยกัน ด้วยการลงมือสำรวจ process จริงบนเครื่องของคุณเอง (หรือเครื่องทดสอบ) แล้วจับคู่แต่ละ process กับหน้าที่ที่ถูกต้อง

### ภารกิจที่ 1: สำรวจ process ทั้งหมดของ instance

```bash
# Step 1: หา PID ของ postmaster
$ pg_ctl -D /var/lib/postgresql/16/main status
pg_ctl: server is running (PID: 1001)

# Step 2: ดู process tree ทั้งหมดที่เป็นลูกของ postmaster
$ pstree -p 1001

# Step 3: ดู process แบบละเอียดพร้อม memory usage
$ ps -eo pid,ppid,rss,vsz,stat,etime,cmd | grep postgres
```

### ภารกิจที่ 2: จำลอง workload แล้วสังเกต backend process เกิดใหม่

```bash
# เปิด terminal ที่ 1: รัน query ที่ใช้เวลานาน
$ psql -h localhost -U app_user -d myapp -c "SELECT pg_sleep(30), pg_backend_pid();"

# เปิด terminal ที่ 2: ระหว่างที่ query ยังทำงานอยู่ ให้ดู process
$ ps aux | grep "postgres:.*SELECT"

# เปรียบเทียบ PID ที่เห็นใน terminal 1 (pg_backend_pid()) กับ PID ใน terminal 2
```

### ภารกิจที่ 3: เชื่อมโยง pg_stat_activity กับ ps aux

```sql
-- รันใน terminal ที่ 3 ระหว่าง query ใน terminal 1 ยังทำงานอยู่
SELECT pid, state, wait_event_type, wait_event, backend_type, query
FROM pg_stat_activity
ORDER BY backend_start;
```

จับคู่ทุกแถวในผลลัพธ์กับ process ที่เห็นจาก `ps aux` — ควรพบว่าจำนวนแถวที่ `backend_type = 'client backend'` เท่ากับจำนวน connection ที่คุณเปิดจริง และ background process แต่ละตัว (checkpointer, background writer ฯลฯ) ก็ควรปรากฏเป็นแถวเช่นกัน

### ภารกิจที่ 4: สังเกต background writer และ checkpointer ทำงานจริง

```sql
-- บันทึกค่าสถิติก่อน
SELECT buffers_clean, buffers_backend FROM pg_stat_bgwriter;

-- สร้าง workload ที่เขียนข้อมูลเยอะ ๆ
CREATE TABLE stress_test AS SELECT generate_series(1, 1000000) AS id, md5(random()::text) AS payload;

-- บันทึกค่าสถิติหลัง แล้วเทียบความต่าง
SELECT buffers_clean, buffers_backend FROM pg_stat_bgwriter;
```

### ภารกิจที่ 5: ทดสอบ SIGHUP reload config

```bash
$ psql -c "SHOW work_mem;"
#  work_mem
# -----------
#  4MB

$ sed -i "s/#work_mem = 4MB/work_mem = 8MB/" /etc/postgresql/16/main/postgresql.conf
$ pg_ctl reload -D /var/lib/postgresql/16/main

$ psql -c "SHOW work_mem;"
#  work_mem
# -----------
#  8MB
```

---

## สรุปท้ายบท

ในบทนี้เราได้เจาะลึกรากฐานที่สุดของ PostgreSQL — สถาปัตยกรรม process ที่เป็นตัวกำหนดพฤติกรรมของระบบในทุกระดับที่เราเคยเรียนมาตลอดหลักสูตร ประเด็นสำคัญที่ควรจดจำ:

1. **PostgreSQL เป็น process-based architecture** — ทุก connection คือ OS process แยกต่างหาก ไม่ใช่ thread ภายใน process เดียว ซึ่งเป็นการตัดสินใจเชิงวิศวกรรมที่แลกความเร็วในการสร้าง connection กับความเสถียรและ isolation ของระบบ

2. **Postmaster เป็น process แม่** ที่ทำหน้าที่ listen connection, fork backend process ใหม่สำหรับทุก client, monitor สถานะของทุก process และประสานงาน shutdown/restart — postmaster เองไม่เคยรัน SQL query

3. **Backend process ผูกกับ connection แบบ 1:1** — สามารถยืนยันได้จริงผ่าน `ps aux`, `pg_backend_pid()` และ `pg_stat_activity` ที่ PID ตรงกันเสมอ

4. **Shared Memory คือกระดานข้อมูลกลาง** ที่ประกอบด้วย shared_buffers, WAL buffers, lock table, proc array และ CLOG — เป็นสิ่งที่ทำให้ process ที่แยกกันสามารถทำงานร่วมกันเป็นระบบเดียวได้

5. **Background process หลัก 6 ตัว** — Background Writer (เขียน dirty page ทีละน้อยต่อเนื่อง), Checkpointer (full flush เป็นระยะเพื่อจุด recovery), WAL Writer (เขียน WAL buffer เป็นระยะ), Autovacuum Launcher/Worker (เก็บกวาด dead tuple และป้องกัน wraparound), Logical Replication Launcher (จัดการ subscription worker) — โดย Stats Collector มีอยู่เฉพาะใน PostgreSQL 14 และเก่ากว่า ส่วน PG15+ เปลี่ยนเป็น cumulative statistics ใน shared memory

6. **การสื่อสารระหว่าง process** ใช้ 3 กลไกร่วมกัน — shared memory (ข้อมูล), lock/semaphore (การประสานการเข้าถึงและ blocking), และ signal (การสั่งการระดับ OS) — ไม่ใช่ message passing แบบที่ thread ใช้กัน

7. **Process-based มีทั้งข้อดีและข้อเสียชัดเจน** — ปลอดภัยกว่าในแง่ isolation และ debug ง่ายกว่า แต่ต้นทุนต่อ connection สูงกว่า thread-based อย่างมีนัยสำคัญ ทั้งในแง่ memory และเวลาที่ใช้สร้าง connection ใหม่

8. **Connection Pooling ไม่ใช่แค่ best practice ลอย ๆ** แต่เป็นวิธีแก้ปัญหาเชิงกลไกที่ตรงจุด — หลีกเลี่ยงต้นทุน `fork()` ที่แพงที่สุดของสถาปัตยกรรมนี้ โดยการ "ยืม-คืน" backend process ที่มีอยู่แล้วแทนการสร้างใหม่ทุกครั้ง

ความเข้าใจในบทนี้จะเป็นรากฐานสำคัญสำหรับบทถัดไป ซึ่งเราจะเจาะลึกลงไปอีกชั้นหนึ่ง — วิธีที่ PostgreSQL จัดเก็บข้อมูลจริงบน disk (Storage Engine, Page Layout, Heap, TOAST) ซึ่งจะอธิบายว่าเมื่อ backend process เขียนข้อมูลลง shared buffer แล้ว มันถูกจัดรูปแบบและบันทึกลง disk อย่างไรในระดับ byte

---

## แบบฝึกหัด

<details>
<summary><strong>แบบฝึกหัดที่ 1:</strong> อธิบายความแตกต่างพื้นฐานระหว่าง process-based architecture และ thread-based architecture โดยเฉพาะในแง่ของ memory address space</summary>

**เฉลย:**

ใน **process-based architecture** (PostgreSQL) แต่ละ connection ถูกจัดการโดย OS process แยกต่างหาก แต่ละ process มี **memory address space เป็นของตัวเอง** ที่ OS ป้องกันไม่ให้ process อื่นเข้าถึงได้โดยตรง หากต้องการแชร์ข้อมูลระหว่าง process ต้องสร้าง **shared memory segment** ขึ้นมาเฉพาะและ map เข้าไปใน address space ของแต่ละ process ที่ต้องการเข้าถึง

ใน **thread-based architecture** (MySQL, SQL Server) แต่ละ connection ถูกจัดการโดย thread ที่รันอยู่ภายใน process เดียวกัน ทุก thread **แชร์ memory address space เดียวกันโดยธรรมชาติ** ไม่ต้องสร้างกลไกพิเศษเพื่อแชร์ข้อมูล แต่ก็หมายความว่า thread หนึ่งสามารถเขียนทับ memory ของ thread อื่นโดยไม่ได้ตั้งใจได้ (ถ้ามี bug) ซึ่งอาจทำให้ทั้ง process ล่มพร้อมกันทุก connection

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 2:</strong> เมื่อรันคำสั่ง <code>ps aux | grep postgres</code> แล้วเห็น process ที่มี PPID เท่ากับ 1001 (postmaster) จำนวนมาก คุณจะอธิบายความสัมพันธ์นี้อย่างไร และจะยืนยันได้อย่างไรว่า process แต่ละตัวทำหน้าที่อะไร</summary>

**เฉลย:**

ความสัมพันธ์นี้แสดงว่าทุก process ที่มี PPID = 1001 เป็น **child process ที่ถูก fork() โดย postmaster (PID 1001) โดยตรง** ทั้งหมด ไม่ว่าจะเป็น backend process ของ client connection หรือ background process ต่าง ๆ (checkpointer, background writer ฯลฯ)

วิธียืนยันหน้าที่ของแต่ละ process:
1. ดู **process title** ที่ต่อท้าย `postgres:` ใน `ps aux` เช่น `postgres: checkpointer`, `postgres: myapp app_user 127.0.0.1(41200) idle`
2. เทียบ PID กับผลลัพธ์จาก `SELECT pid, backend_type FROM pg_stat_activity;` ซึ่งจะบอกหน้าที่ของแต่ละ process ในรูปแบบ SQL
3. ใช้ `pstree -p 1001` เพื่อดูโครงสร้าง parent-child ทั้งหมดในมุมมองต้นไม้

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 3:</strong> Shared Memory ใน PostgreSQL ประกอบด้วยส่วนสำคัญอะไรบ้าง อธิบายหน้าที่ของแต่ละส่วนสั้น ๆ</summary>

**เฉลย:**

1. **Shared Buffers** — cache ของ data page (8 KB block) ที่อ่านจาก disk เพื่อลดการอ่าน disk ซ้ำ
2. **WAL Buffers** — buffer เก็บ WAL record ก่อน flush ลง disk จริง ใช้โดยทุก backend ที่แก้ไขข้อมูล
3. **Lock Table** — เก็บสถานะ lock ทั้งหมดของระบบ (row lock, table lock ฯลฯ) เพื่อให้ทุก process มองเห็น lock ที่ถูกถืออยู่ร่วมกัน
4. **Proc Array** — เก็บสถานะของ backend ที่ active ทั้งหมด (transaction ID, snapshot) ใช้สำหรับกลไก MVCC
5. **CLOG (Commit Log)** — เก็บสถานะ commit/abort ของแต่ละ transaction ใช้ตัดสิน visibility ของ row version

ทั้งหมดนี้ถูก allocate ครั้งเดียวตอน server start และ map เข้าไปใน address space ของทุก process ที่เกี่ยวข้อง

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 4:</strong> Background Writer และ Checkpointer ทำหน้าที่คล้ายกัน (เขียน dirty page ลง disk) แต่ต่างกันอย่างไร</summary>

**เฉลย:**

**Background Writer** เขียน dirty page **ทีละน้อยอย่างต่อเนื่อง** ในพื้นหลังตลอดเวลา (ตาม `bgwriter_delay`, `bgwriter_lru_maxpages`) เพื่อเตรียมพื้นที่ buffer ว่างล่วงหน้าให้ backend process ใช้ ไม่ต้อง block รอเขียนเอง

**Checkpointer** ทำ **full flush** ของ dirty page ทั้งหมดที่ค้างอยู่เป็นระยะ (ตาม `checkpoint_timeout` หรือ `max_wal_size`) แล้วบันทึกตำแหน่ง checkpoint ล่าสุดลง control file เพื่อเป็นจุดอ้างอิงสำหรับ **crash recovery** — ยิ่ง checkpoint ห่างกันนาน ยิ่งต้อง replay WAL มากขึ้นตอน recovery แต่ถ้าถี่เกินไปจะสร้าง I/O burst บ่อยเกินไป

สรุป: bgwriter ทำงานเบา ๆ ต่อเนื่องเพื่อ "ผ่อนภาระ" backend process ส่วน checkpointer ทำงานหนักเป็นช่วง ๆ เพื่อสร้าง "จุดกู้คืน" ที่ชัดเจน

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 5:</strong> ในเวอร์ชัน PostgreSQL 15 ขึ้นไป เกิดการเปลี่ยนแปลงอะไรกับระบบสถิติ (statistics) เมื่อเทียบกับ PostgreSQL 14 และเก่ากว่า</paragraph></summary>

**เฉลย:**

ใน PostgreSQL 14 และเก่ากว่า มี background process แยกต่างหากชื่อ **stats collector** ที่รับข้อมูลสถิติจากทุก backend ผ่าน UDP socket แล้วเขียนลงไฟล์ temp file เป็นระยะ ซึ่งมี overhead และ latency ในการอัปเดต

ตั้งแต่ PostgreSQL 15 เป็นต้นไป มีการปรับสถาปัตยกรรมเป็น **cumulative statistics system** ที่เก็บสถิติ**ไว้ใน shared memory โดยตรง** ไม่มี stats collector process แยกต่างหากอีกต่อไป และไม่มีการเขียน temp file เป็นระยะเหมือนเดิม (จะ dump ลง disk เฉพาะตอน shutdown เพื่อให้ restart แล้วยังใช้สถิติเก่าต่อได้) ทำให้การอัปเดตสถิติเร็วขึ้นและลด overhead ของระบบ

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 6:</strong> อธิบายความแตกต่างระหว่างกลไก "Lock (LWLock/Spinlock)" กับ "Semaphore" ในการประสานงานระหว่าง process ของ PostgreSQL</summary>

**เฉลย:**

**Lock (LWLock/Spinlock)** ใช้ป้องกันไม่ให้หลาย process เข้าถึง/แก้ไขโครงสร้างข้อมูลใน shared memory พร้อมกันจนเกิด race condition — Spinlock ใช้กับข้อมูลเล็กที่คาดว่าจะถือ lock สั้นมาก (busy-wait) ส่วน LWLock ใช้กับโครงสร้างที่ใหญ่กว่าและอาจถือนานกว่า (จะ sleep แทน busy-wait ถ้ารอนาน)

**Semaphore** เป็นกลไกระดับ OS ที่ใช้เมื่อ process ต้อง**บล็อกรอเหตุการณ์**จาก process อื่นแบบเต็มรูปแบบ เช่น รอ row lock ที่ transaction อื่นถืออยู่ — process ที่รอจะ sleep จริง ๆ ไม่กิน CPU cycle จนกว่า OS จะปลุกมันเมื่อเหตุการณ์ที่รอเกิดขึ้น (เช่น transaction อื่น commit/rollback แล้วปล่อย lock)

สรุปคือ Lock ใช้ป้องกันการเข้าถึงข้อมูลพร้อมกันระยะสั้น ๆ ส่วน Semaphore ใช้สำหรับ blocking รอเหตุการณ์ที่อาจใช้เวลานานกว่า

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 7:</strong> ทำไมการ <code>kill -9</code> (SIGKILL) backend process โดยตรงจึงเป็นอันตราย ควรใช้คำสั่งอะไรแทน</summary>

**เฉลย:**

`SIGKILL` ไม่สามารถถูก handle โดยโปรแกรมได้ — process จะถูกฆ่าทันทีโดยไม่มีโอกาส cleanup หรือแจ้งสถานะใด ๆ เมื่อ postmaster ตรวจพบว่า child process terminate ผิดปกติแบบนี้ มันจะสันนิษฐานว่า **shared memory อาจเสียหาย** (เพราะ process อาจตายกลางคันขณะกำลังแก้ไขโครงสร้างข้อมูลที่แชร์อยู่) และจะสั่งให้ **backend process อื่นทั้งหมด disconnect ทันที** แล้วเข้าสู่ crash recovery ทั้งระบบ — กระทบทุก connection ไม่ใช่แค่ connection ที่ถูก kill

ควรใช้ `SELECT pg_terminate_backend(pid);` หรือ `SELECT pg_cancel_backend(pid);` แทน ซึ่งเป็นการส่ง `SIGTERM`/`SIGINT` ที่ PostgreSQL ออกแบบมาให้ backend process จัดการ cleanup ตัวเองอย่างปลอดภัยก่อน exit โดยไม่กระทบ connection อื่น

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 8:</strong> เปรียบเทียบเวลาที่ใช้ในการเปิด connection ใหม่ (fork process) กับเวลาที่ใช้รัน query ผ่าน connection ที่เปิดค้างไว้แล้ว เพราะเหตุใดความแตกต่างนี้จึงสำคัญต่อการออกแบบแอปพลิเคชัน</summary>

**เฉลย:**

การเปิด connection ใหม่แต่ละครั้งต้องผ่านขั้นตอน `fork()` process ใหม่ทั้งหมด, authentication, และการสร้าง catalog cache เริ่มต้น ซึ่งมักใช้เวลาระดับหลักสิบ millisecond ในขณะที่การรัน query ผ่าน connection ที่เปิดค้างไว้แล้วใช้เวลาต่ำกว่า 1 millisecond (ตัวอย่างในบทนี้แสดงความต่างประมาณ 80 เท่า)

ความสำคัญ: แอปพลิเคชันที่เปิด-ปิด connection บ่อย ๆ (เช่น เปิด connection ใหม่ทุก HTTP request หรือทุก function call ใน serverless environment) จะเจอ latency overhead สะสมมหาศาลจากขั้นตอน `fork()` ที่แพง ทำให้ throughput โดยรวมของระบบต่ำกว่าที่ควรจะเป็นมาก การแก้ปัญหาคือใช้ **connection pooling** (เช่น PgBouncer) เพื่อ reuse backend process ที่มีอยู่แล้วแทนการสร้างใหม่ทุกครั้ง หรือออกแบบแอปพลิเคชันให้คง connection ไว้นาน ๆ (persistent connection) แทนการเปิด-ปิดถี่ ๆ

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 9:</strong> จงอธิบายด้วยคำพูดของตัวเองว่า PgBouncer แก้ปัญหาอะไรของสถาปัตยกรรม process-based ของ PostgreSQL โดยเฉพาะ</summary>

**เฉลย:**

PgBouncer แก้ปัญหาต้นทุนของการ **`fork()` backend process ใหม่ทุกครั้ง** ที่มี connection ใหม่เข้ามา ซึ่งเป็นขั้นตอนที่แพงที่สุดในสถาปัตยกรรม process-based ของ PostgreSQL

แทนที่ application จะเชื่อมต่อตรงไปยัง PostgreSQL (ซึ่งจะ trigger การ fork backend process ใหม่ทุกครั้ง) application จะเชื่อมต่อไปยัง PgBouncer แทน โดย PgBouncer จะรักษา pool ของ backend connection ที่เปิดค้างไว้กับ PostgreSQL อยู่แล้วจำนวนหนึ่ง แล้ว "ยืม" backend connection เหล่านั้นให้ application ใช้ชั่วคราว (ตาม pooling mode ที่ตั้งค่าไว้) จากนั้น "คืน" กลับเข้า pool เมื่อใช้งานเสร็จ

ผลลัพธ์คือ backend process ฝั่ง PostgreSQL จริง ๆ ถูกสร้างขึ้นเพียงจำนวนจำกัด (เท่ากับขนาด pool) และถูกใช้ซ้ำต่อเนื่อง แทนที่จะต้อง fork ใหม่ทุกครั้งที่มี client request เข้ามา ทำให้ลดต้นทุน `fork()` + authentication + cache warmup ที่ต้องเสียซ้ำ ๆ ได้เกือบทั้งหมด และยังช่วยจำกัดจำนวน backend process สูงสุดที่ PostgreSQL server ต้องรองรับพร้อมกัน ลดความเสี่ยงเรื่อง memory overhead จาก `max_connections` ที่สูงเกินไป

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 10:</strong> ลองรันคำสั่งต่อไปนี้บนเครื่องจริงของคุณ แล้วอธิบายว่าแต่ละแถวในผลลัพธ์สอดคล้องกับ background process ตัวใดที่เรียนมาในบทนี้

```sql
SELECT pid, backend_type, state
FROM pg_stat_activity
ORDER BY backend_type;
```
</summary>

**เฉลย:**

ผลลัพธ์ตัวอย่าง (จะแตกต่างกันไปตาม instance จริงและ PostgreSQL version):

```
  pid  |          backend_type           | state
--------+-----------------------------------+--------
  1005 | checkpointer                      |
  1006 | background writer                 |
  1007 | walwriter                         |
  1008 | autovacuum launcher               |
  1009 | logical replication launcher      |
  3001 | client backend                    | idle
  3005 | client backend                    | active
```

การจับคู่:
- `checkpointer` → process ที่ทำ full flush ของ dirty page เป็นระยะเพื่อสร้างจุด recovery (Step 805)
- `background writer` → process ที่เขียน dirty page ทีละน้อยต่อเนื่อง (Step 805)
- `walwriter` → process ที่เขียน WAL buffer ลง disk เป็นระยะ (Step 805)
- `autovacuum launcher` → process ที่คอยตัดสินใจว่าต้อง vacuum table ไหน แล้ว fork worker (Step 806)
- `logical replication launcher` → process ที่ดูแล worker สำหรับแต่ละ logical replication subscription (Step 806)
- `client backend` → process ที่ผูกกับ client connection แบบ 1:1 โดยตรง (Step 803) — สังเกตว่าแถวเหล่านี้เท่านั้นที่มีค่า `state` ที่มีความหมาย เช่น `idle`/`active` เพราะเป็น process เดียวที่รับและประมวลผล SQL statement จาก client จริง ๆ

หมายเหตุ: หากใช้ PostgreSQL 14 หรือเก่ากว่า จะเห็นแถวเพิ่มเติมที่มี `backend_type = 'stats collector'` ด้วย (ดู Step 806)

</details>

---

**บทถัดไป:** [Part 082 — PostgreSQL Storage Engine: Page Layout, Heap, TOAST](./part-082-storage-engine.md)
