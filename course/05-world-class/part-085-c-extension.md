# Part 085: การเขียน Custom Extension ด้วยภาษา C

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับโลก/ผู้เชี่ยวชาญ | Part 085

## เป้าหมายการเรียนรู้

เมื่อจบบทนี้ ผู้เรียนจะสามารถ:

- อธิบายได้ว่าเมื่อไหร่ควรเขียน extension ด้วยภาษา C แทน PL/pgSQL หรือ PL/Python และเหตุผลเชิง performance/architecture
- เข้าใจโครงสร้างไฟล์พื้นฐานของ extension หนึ่งตัว ได้แก่ control file, SQL script และ shared library (`.so`)
- ติดตั้งและตั้งค่า development environment สำหรับ compile extension บน Linux (Debian/Ubuntu และ RHEL/Fedora)
- เขียนฟังก์ชัน C ตัวแรกโดยใช้ `PG_MODULE_MAGIC`, `PG_FUNCTION_INFO_V1`, และ `Datum` type ได้อย่างถูกต้อง
- จัดการ argument และ return value ด้วยมาโคร `PG_GETARG_*` / `PG_RETURN_*` รวมถึงจัดการค่า NULL ด้วย `PG_ARGISNULL`
- เขียน Makefile โดยใช้ PGXS (PostgreSQL Extension Building Infrastructure) เพื่อ compile และ install extension
- เขียน control file และ SQL definition file ที่ผูกฟังก์ชัน SQL เข้ากับฟังก์ชัน C ผ่าน `MODULE_PATHNAME`
- เข้าใจแนวคิด Memory Context และใช้ `palloc`/`pfree` อย่างถูกต้องปลอดภัย
- ใช้ `ereport()` และ `elog()` เพื่อรายงาน error/notice/warning ตามมาตรฐานของ PostgreSQL
- ลงมือเขียน extension สมบูรณ์ที่มีฟังก์ชันคำนวณธุรกิจจริง ตั้งแต่ source code จนถึง compile, install และเรียกใช้งานผ่าน SQL

---

## Step 841: ทำไมต้องเขียน Extension ด้วย C

ตลอดหลักสูตรที่ผ่านมา เราเขียนฟังก์ชันด้วย PL/pgSQL (Part ก่อนหน้า) และอาจเคยใช้ PL/Python, PL/Perl หรือภาษาอื่นที่ฝังอยู่ใน PostgreSQL ได้ คำถามคือ แล้วทำไมยังต้องลงลึกไปเขียนด้วยภาษา C อีก?

### 1. Performance สูงสุดที่เป็นไปได้

PostgreSQL เองเขียนด้วยภาษา C ทั้งหมด ฟังก์ชัน built-in เช่น `+`, `substring()`, `now()` ล้วนเป็นฟังก์ชัน C ที่ compile เป็น native machine code เมื่อเราเขียนฟังก์ชันด้วย C เราจะได้ประสิทธิภาพในระดับเดียวกับฟังก์ชัน built-in เพราะ:

- **ไม่มี interpreter overhead** — PL/pgSQL ต้อง parse และ interpret แต่ละ statement ทุกครั้งที่เรียก (แม้จะมี plan cache) ส่วนฟังก์ชัน C คือ compiled code ที่ถูกเรียกโดยตรงผ่าน function pointer
- **ไม่มีการแปลงชนิดข้อมูลซ้ำซ้อน (type conversion overhead)** — PL/pgSQL ต้องแปลงค่าระหว่าง SQL datum และ PL/pgSQL internal representation หลายรอบ ส่วน C function ทำงานกับ `Datum` โดยตรง
- **สามารถเขียน algorithm ที่ optimize ระดับ CPU ได้** — เช่น bit manipulation, SIMD-friendly loop, cache-friendly memory layout

จากการวัดผลในโปรเจกต์จริงหลายแห่ง ฟังก์ชันคำนวณหนัก ๆ (เช่น geospatial calculation, cryptographic hash, string parsing แบบซับซ้อน) ที่เขียนใหม่จาก PL/pgSQL เป็น C สามารถเร็วขึ้นได้ 10x–100x ขึ้นอยู่กับลักษณะงาน

### 2. เข้าถึง Internal API ได้เต็มที่

PL/pgSQL ถูกจำกัดให้ทำงานผ่าน SQL statement และ built-in function เท่านั้น แต่ฟังก์ชันที่เขียนด้วย C สามารถเรียกใช้ internal API ของ PostgreSQL ได้โดยตรง เช่น:

- เข้าถึง buffer manager, WAL, lock manager โดยตรง (ใช้ทำ custom index access method, background worker)
- เขียน custom data type พร้อม input/output function ของตัวเอง (`typmod`, binary I/O)
- เขียน custom aggregate function, custom operator, custom index (GIN/GiST opclass)
- เรียกใช้ syscache, relcache เพื่ออ่าน metadata ของตาราง/คอลัมน์แบบ low-level
- เขียน Foreign Data Wrapper (FDW) เพื่อเชื่อมต่อกับระบบภายนอก
- เขียน background worker process ที่ทำงานเป็น daemon ภายใน PostgreSQL cluster

สิ่งเหล่านี้ **ทำไม่ได้เลย** ด้วย PL/pgSQL หรือแม้แต่ PL/Python เพราะภาษาเหล่านั้นถูกจำกัดอยู่ใน sandbox ของ procedural language interpreter

### 3. เปรียบเทียบกับ PL/pgSQL อย่างเป็นระบบ

| ประเด็น | PL/pgSQL | C Extension |
|---|---|---|
| ความเร็วในการ execute | ปานกลาง (มี interpreter overhead) | สูงสุด (native compiled code) |
| ความยาก/เวลาในการพัฒนา | ง่าย เขียนเร็ว | ยาก ต้องเข้าใจ C, memory management, PostgreSQL internals |
| ความเสี่ยงต่อ server crash | ต่ำมาก (error จะถูก catch เป็น SQL exception) | สูง — bug เช่น segmentation fault หรือ memory corruption อาจทำให้ **ทั้ง backend process crash** |
| การเข้าถึง internal API | ไม่ได้เลย | เข้าถึงได้เต็มรูปแบบ |
| การ deploy | อยู่ใน database เอง ผ่าน `CREATE FUNCTION` | ต้อง compile เป็น `.so`/`.dll` และ install ในระดับ OS ก่อน |
| Portability ข้ามระบบปฏิบัติการ/สถาปัตยกรรม | สูง (source อยู่ใน DB) | ต้อง compile ใหม่ต่อแพลตฟอร์ม |
| เหมาะกับงานประเภท | business logic ทั่วไป, trigger, validation | hot-path ที่ต้องการความเร็วสูงสุด, custom type, custom index, ระบบระดับ infrastructure |

### 4. เมื่อไหร่ควรเลือกเขียนด้วย C

ควรพิจารณาเขียน extension ด้วย C เมื่อ:

- ฟังก์ชันนั้นถูกเรียกใช้งานบ่อยมากในระดับ hot path (เช่น เรียกหลายล้านครั้งต่อ query, ใช้ใน `WHERE` clause ของตารางขนาดใหญ่)
- ต้องการสร้าง data type ใหม่ที่ PostgreSQL ไม่มี (เช่น custom geometric type, custom encoding)
- ต้องการสร้าง index access method หรือ operator class ใหม่
- ต้องการทำ integration ระดับลึกกับ internal ของ PostgreSQL เช่น custom WAL resource manager, custom background worker
- Benchmark แสดงให้เห็นชัดเจนว่า PL/pgSQL/PL/Python เป็นคอขวด (bottleneck) จริง ๆ

ในทางกลับกัน หากงานนั้นไม่ได้ต้องการความเร็วระดับสูงสุด หรือทีมไม่มีความชำนาญด้าน C และ memory management การเขียน C extension อาจสร้างความเสี่ยงเกินความจำเป็น เพราะ bug ในฟังก์ชัน C อาจทำให้ **ทั้ง PostgreSQL server ล่ม** ในขณะที่ bug ใน PL/pgSQL อย่างเลวร้ายที่สุดก็แค่ query นั้น error ออกมา

> **หลักการสำคัญ**: "Premature optimization is the root of all evil" — อย่าเริ่มเขียน C extension ตั้งแต่แรก ให้เริ่มจาก PL/pgSQL หรือ SQL function ก่อน แล้ววัดผลจริงด้วย `EXPLAIN ANALYZE` และ profiling tools ถ้าพบว่าเป็นคอขวดจริง ๆ ค่อยพิจารณาเขียนใหม่ด้วย C

---

## Step 842: โครงสร้างพื้นฐานของ Extension

ก่อนจะเริ่มเขียนโค้ด เราต้องเข้าใจก่อนว่า extension ของ PostgreSQL ประกอบด้วยไฟล์กี่ประเภท และแต่ละไฟล์มีหน้าที่อะไร

Extension ที่สมบูรณ์หนึ่งตัวต้องมีไฟล์อย่างน้อย 3 ประเภท:

### 1. Control File (`.control`)

ไฟล์ metadata ที่บอก PostgreSQL ว่า extension นี้ชื่ออะไร, version เริ่มต้นคือเท่าไหร่, และไฟล์ shared library อยู่ที่ไหน ไฟล์นี้ต้องชื่อตรงกับชื่อ extension เช่น extension ชื่อ `pgtutorial` ต้องมีไฟล์ `pgtutorial.control`

ตัวอย่างโครงสร้าง:

```ini
# pgtutorial.control
comment = 'ฟังก์ชันตัวอย่างสำหรับเรียนรู้การเขียน extension ด้วยภาษา C'
default_version = '1.0'
module_pathname = '$libdir/pgtutorial'
relocatable = true
```

ไฟล์นี้จะถูกอ่านโดย PostgreSQL เมื่อมีคำสั่ง `CREATE EXTENSION pgtutorial;` และไฟล์ต้องถูกวางไว้ใน directory `SHAREDIR/extension/` (หาตำแหน่งได้ด้วยคำสั่ง `pg_config --sharedir`)

### 2. SQL Script File (`.sql`)

ไฟล์ที่มีคำสั่ง SQL สำหรับสร้าง object ต่าง ๆ ของ extension เช่น `CREATE FUNCTION`, `CREATE TYPE`, `CREATE OPERATOR` ชื่อไฟล์ต้องเป็นรูปแบบ `<extension_name>--<version>.sql` เช่น `pgtutorial--1.0.sql`

ไฟล์นี้จะถูก execute โดยอัตโนมัติเมื่อรัน `CREATE EXTENSION` — มันคือ "แผนผัง" ว่า extension นี้เพิ่ม object อะไรเข้าไปใน database บ้าง

### 3. Shared Library (`.so` บน Linux)

ไฟล์ binary ที่ได้จากการ compile source code ภาษา C เป็น shared object (บน Linux คือ `.so`, บน macOS คือ `.dylib`, บน Windows คือ `.dll`) ไฟล์นี้ต้องถูกวางไว้ใน directory `PKGLIBDIR` (หาตำแหน่งได้ด้วยคำสั่ง `pg_config --pkglibdir`)

ภายใน shared library นี้คือ machine code ของฟังก์ชันที่เราเขียนด้วยภาษา C ซึ่ง PostgreSQL จะโหลดเข้าไปใน process memory ด้วย `dlopen()` (บน Linux) ตอนที่มีการเรียกใช้ฟังก์ชันครั้งแรก

### ภาพรวมความสัมพันธ์ระหว่างไฟล์

```
CREATE EXTENSION pgtutorial;
        │
        ▼
1. PostgreSQL หา pgtutorial.control ใน SHAREDIR/extension/
        │  (อ่าน default_version, module_pathname)
        ▼
2. PostgreSQL หา pgtutorial--1.0.sql ใน SHAREDIR/extension/
        │  (execute SQL statements ทั้งหมดในไฟล์นี้)
        ▼
3. คำสั่ง CREATE FUNCTION ... AS 'MODULE_PATHNAME', 'add_one'
        │  จะ resolve MODULE_PATHNAME เป็น $libdir/pgtutorial
        ▼
4. เมื่อมีการเรียกใช้ฟังก์ชัน add_one() ครั้งแรก
        │  PostgreSQL จะ dlopen() ไฟล์ pgtutorial.so ใน PKGLIBDIR
        ▼
5. เรียก symbol "add_one" ใน .so และรันโค้ด C จริง ๆ
```

### ตำแหน่ง directory ที่สำคัญ

```bash
# ตรวจสอบตำแหน่ง directory ต่าง ๆ ของ PostgreSQL installation
pg_config --sharedir     # เช่น /usr/share/postgresql/16
pg_config --pkglibdir    # เช่น /usr/lib/postgresql/16/lib
pg_config --includedir-server  # เช่น /usr/include/postgresql/16/server
pg_config --pgxs         # path ไปยัง PGXS Makefile (ใช้ตอน build)
```

- `.control` และ `.sql` จะถูกวางไว้ใน `$(pg_config --sharedir)/extension/`
- `.so` จะถูกวางไว้ใน `$(pg_config --pkglibdir)/`

ในทางปฏิบัติ เราไม่ต้อง copy ไฟล์เหล่านี้ไปวางเองด้วยมือ เพราะ PGXS build system (ที่จะเรียนใน Step 843 และ 846) จะจัดการ `make install` ให้อัตโนมัติ

### Extension แบบมี C code เทียบกับแบบ SQL-only

ควรทราบไว้ว่า extension ของ PostgreSQL ไม่จำเป็นต้องมี C code เสมอไป — extension ที่มีแค่ `.control` และ `.sql` (ไม่มี `.so`) ก็เป็น extension ที่สมบูรณ์ได้ เช่น extension ที่รวบรวมฟังก์ชัน PL/pgSQL ไว้เป็นชุดเดียว แต่บทนี้เราจะโฟกัสที่ extension ที่มี C shared library ประกอบอยู่ด้วย เพราะเป็นรูปแบบที่ให้ประสิทธิภาพสูงสุดตามที่กล่าวใน Step 841

---

## Step 843: การตั้งค่า Development Environment

ก่อนเริ่มเขียนโค้ดจริง เราต้องติดตั้งเครื่องมือที่จำเป็นสำหรับการ compile extension ก่อน

### 1. ติดตั้ง PostgreSQL server development headers

ในการ compile C code ที่เรียกใช้ internal API ของ PostgreSQL (เช่น `postgres.h`, `fmgr.h`) เราต้องมีไฟล์ header ของ PostgreSQL server ซึ่ง**ไม่ได้ติดตั้งมาพร้อมกับ PostgreSQL client package ทั่วไป** ต้องติดตั้งแพ็กเกจ `-dev`/`-devel` เพิ่มเติม

**บน Debian/Ubuntu:**

```bash
# ติดตั้ง PostgreSQL server development files
# เปลี่ยนเลขเวอร์ชันให้ตรงกับ PostgreSQL ที่ใช้งานจริง เช่น 16
sudo apt-get update
sudo apt-get install postgresql-server-dev-16

# ติดตั้ง build essential tools (gcc, make, ฯลฯ) ถ้ายังไม่มี
sudo apt-get install build-essential
```

**บน RHEL/Fedora/Rocky Linux:**

```bash
# ติดตั้ง PostgreSQL server development files
sudo dnf install postgresql16-devel

# หรือถ้าใช้ PGDG repository
sudo dnf install postgresql16-devel gcc make

# ติดตั้ง development tools group
sudo dnf groupinstall "Development Tools"
```

**บน macOS (ใช้ Homebrew):**

```bash
brew install postgresql@16
# Homebrew PostgreSQL รวม header และ pg_config มาให้ในตัวอยู่แล้ว
```

### 2. ตรวจสอบว่า `pg_config` ใช้งานได้

`pg_config` เป็นเครื่องมือสำคัญที่สุดในการ build extension เพราะมันบอกตำแหน่งของทุกอย่างที่ build system ต้องใช้

```bash
# ตรวจสอบว่า pg_config มองเห็นและชี้ไปที่ PostgreSQL เวอร์ชันที่ถูกต้อง
which pg_config
pg_config --version
# ตัวอย่าง output: PostgreSQL 16.4
```

หากมี PostgreSQL หลายเวอร์ชันในเครื่อง (เช่นติดตั้งจาก OS package และจาก source พร้อมกัน) ต้องตรวจสอบให้แน่ใจว่า `pg_config` ตัวที่อยู่ใน `$PATH` เป็นตัวที่ตรงกับ server ที่เราจะ deploy extension เข้าไปจริง เพราะการ compile ด้วย header เวอร์ชันหนึ่ง แต่ install เข้า server อีกเวอร์ชันหนึ่ง จะทำให้เกิด error ตอน `CREATE EXTENSION` หรือแย่กว่านั้นคือ server crash

```bash
# ถ้ามีหลายเวอร์ชัน สามารถระบุ path แบบเต็มได้
/usr/lib/postgresql/16/bin/pg_config --version
```

### 3. ทำความรู้จัก PGXS (PostgreSQL Extension Building Infrastructure)

PGXS คือชุด Makefile rules ที่ PostgreSQL เตรียมไว้ให้ เพื่อให้เราเขียน extension โดยไม่ต้องเขียน Makefile ที่ซับซ้อนเอง (ไม่ต้องกังวลเรื่อง compiler flags, include paths, linker flags ที่แตกต่างกันในแต่ละแพลตฟอร์ม)

หลักการทำงานคือ PGXS Makefile จะถูก `include` เข้าไปใน Makefile ของ extension เรา และมันจะ:

- ตั้งค่า `CC`, `CFLAGS` ให้ตรงกับที่ PostgreSQL server ถูก compile มา (เพื่อความเข้ากันได้ของ ABI)
- เพิ่ม include path ไปที่ `$(pg_config --includedir-server)` โดยอัตโนมัติ
- สร้าง target `all`, `install`, `clean`, `installcheck` ให้อัตโนมัติ
- จัดการ copy ไฟล์ `.control`, `.sql`, `.so` ไปวางใน directory ที่ถูกต้องตอน `make install`

```bash
# ตรวจสอบว่าเครื่อง PGXS อยู่ที่ไหน
pg_config --pgxs
# ตัวอย่าง output: /usr/lib/postgresql/16/lib/pgxs/src/makefiles/pgxs.mk
```

เราจะมาดูวิธีเขียน Makefile ที่ใช้ PGXS อย่างละเอียดใน Step 846 แต่ตอนนี้ให้ทราบไว้ก่อนว่าเราไม่ต้องเขียน build system เองตั้งแต่ต้น — PGXS จัดการให้เกือบทั้งหมด

### 4. ตรวจสอบ permission สำหรับ `make install`

`make install` จะ copy ไฟล์ไปยัง `SHAREDIR` และ `PKGLIBDIR` ซึ่งมักเป็น directory ที่ต้องใช้สิทธิ์ root หรือ sudo (ยกเว้นกรณี compile PostgreSQL จาก source เองแบบ local user) หากพบ error ประเภท `Permission denied` ตอน `make install` ให้ลองใช้ `sudo make install`

### 5. โครงสร้าง directory แนะนำสำหรับโปรเจกต์

```bash
# สร้าง directory สำหรับ extension โปรเจกต์
mkdir -p ~/dev/pgtutorial
cd ~/dev/pgtutorial

# โครงสร้างไฟล์ที่เราจะสร้างในบทนี้
# pgtutorial/
# ├── pgtutorial.c          # source code ภาษา C
# ├── pgtutorial.control    # control file
# ├── pgtutorial--1.0.sql   # SQL definition file
# └── Makefile              # PGXS-based Makefile
```

เมื่อเตรียม environment เสร็จแล้ว เราพร้อมที่จะเขียนฟังก์ชัน C ตัวแรกใน Step ถัดไป

---

## Step 844: เขียนฟังก์ชัน C แรก

มาถึงจุดสำคัญ — การเขียนฟังก์ชัน C ตัวแรกที่ PostgreSQL สามารถเรียกใช้งานได้ เราจะสร้างฟังก์ชันง่าย ๆ ชื่อ `add_one` ที่รับตัวเลข integer หนึ่งตัวแล้วคืนค่าตัวเลขนั้นบวกหนึ่ง

### แนวคิดพื้นฐานที่ต้องเข้าใจก่อน

#### 1. `Datum` — ชนิดข้อมูลสากลของ PostgreSQL internal

PostgreSQL ใช้ชนิดข้อมูล `Datum` เป็น "กล่องสากล" สำหรับส่งค่าทุกชนิดไปมาระหว่างฟังก์ชัน ไม่ว่าจะเป็น integer, text, boolean หรือ struct ที่ซับซ้อน ทุกอย่างถูกห่อหุ้มเป็น `Datum` ทั้งหมด

```c
/* Datum ถูกนิยามใน postgres.h โดยประมาณดังนี้ (concept, ไม่ต้องเขียนเอง) */
typedef uintptr_t Datum;
```

`Datum` คือค่า pointer-sized integer (ปกติ 8 bytes บนระบบ 64-bit) ที่อาจเก็บค่าได้สองแบบ:

- **Pass-by-value**: สำหรับชนิดข้อมูลขนาดเล็ก เช่น `int4`, `bool` ค่าจริงถูกเก็บอยู่ใน `Datum` โดยตรง (bit pattern)
- **Pass-by-reference**: สำหรับชนิดข้อมูลขนาดใหญ่หรือความยาวไม่แน่นอน เช่น `text`, `numeric`, `array` — `Datum` เก็บเป็น pointer ที่ชี้ไปยังหน่วยความจำที่เก็บข้อมูลจริง

เราไม่จำเป็นต้องแปลง `Datum` ด้วยมือเอง เพราะ PostgreSQL เตรียมมาโครสำหรับแปลงค่าให้แล้ว (จะเรียนใน Step 845)

#### 2. `PG_MODULE_MAGIC` — magic number บังคับ

ทุก shared library ที่จะถูก PostgreSQL โหลด (นอกเหนือจากตัว backend เอง) **ต้อง** ประกาศ `PG_MODULE_MAGIC` ไว้ในไฟล์ source อย่างน้อยหนึ่งไฟล์ (ถ้ามีหลายไฟล์ .c ใน extension เดียวกัน ประกาศแค่ครั้งเดียวพอ)

```c
PG_MODULE_MAGIC;
```

มาโครนี้จะ generate struct ที่บรรจุข้อมูล เช่น เวอร์ชันของ PostgreSQL ที่ compile มาด้วย, ขนาดของ struct ต่าง ๆ (เช่น `sizeof(Datum)`) เพื่อให้ PostgreSQL ตรวจสอบตอนโหลด `.so` ว่า extension นี้ compile มาด้วย PostgreSQL เวอร์ชัน/ABI ที่เข้ากันได้กับ server ปัจจุบันหรือไม่ ถ้าไม่ตรงกัน (เช่น compile ด้วย PostgreSQL 15 header แต่ไปรันบน PostgreSQL 16 server) จะได้ error ทันทีแทนที่จะปล่อยให้ crash แบบไม่ทราบสาเหตุ

#### 3. `PG_FUNCTION_INFO_V1` — ประกาศ calling convention

PostgreSQL มี calling convention สำหรับฟังก์ชัน C สองแบบคือ version 0 (แบบเก่ามาก ไม่ใช้แล้ว) และ **version 1** ซึ่งเป็นมาตรฐานปัจจุบัน มาโคร `PG_FUNCTION_INFO_V1(funcname)` ใช้ประกาศว่าฟังก์ชันนี้ใช้ calling convention แบบ version 1

```c
PG_FUNCTION_INFO_V1(add_one);
```

มาโครนี้จะ generate wrapper function ที่ export symbol พิเศษให้ PostgreSQL รู้จักและเรียกใช้ได้อย่างถูกต้องผ่านกลไก `fmgr` (function manager) ของ PostgreSQL

#### 4. `PG_FUNCTION_ARGS` — signature มาตรฐานของฟังก์ชัน

ฟังก์ชันทุกตัวที่จะถูกเรียกจาก SQL ต้องมี signature แบบนี้เสมอ:

```c
Datum funcname(PG_FUNCTION_ARGS)
```

`PG_FUNCTION_ARGS` คือมาโครที่ expand เป็น `FunctionCallInfo fcinfo` — ตัวแปรที่เก็บ argument ทั้งหมด, ข้อมูลว่า argument ไหนเป็น NULL, และข้อมูล context อื่น ๆ ที่ PostgreSQL ส่งมาให้

### ตัวอย่างฟังก์ชันแรก: `add_one`

สร้างไฟล์ `pgtutorial.c`:

```c
/*
 * pgtutorial.c
 *      ฟังก์ชันตัวอย่างสำหรับสอนการเขียน PostgreSQL extension ด้วยภาษา C
 */

#include "postgres.h"   /* ต้อง include เป็นไฟล์แรกเสมอในทุก .c file ของ backend/extension */
#include "fmgr.h"        /* สำหรับ PG_FUNCTION_INFO_V1, PG_GETARG_*, PG_RETURN_* */

/* บังคับต้องมีในทุก shared library ที่ PostgreSQL จะโหลด */
PG_MODULE_MAGIC;

/*
 * add_one
 *      รับค่า integer หนึ่งตัว คืนค่า integer นั้นบวกหนึ่ง
 *      เทียบเท่ากับ SQL: CREATE FUNCTION add_one(integer) RETURNS integer
 */
PG_FUNCTION_INFO_V1(add_one);

Datum
add_one(PG_FUNCTION_ARGS)
{
    int32   arg = PG_GETARG_INT32(0);  /* ดึง argument ตัวที่ 0 (นับจาก 0) เป็น int32 */

    PG_RETURN_INT32(arg + 1);          /* คืนค่าเป็น int32 กลับไปยัง SQL */
}
```

### อธิบายทีละบรรทัด

1. `#include "postgres.h"` — **ต้องเป็นไฟล์แรกที่ include เสมอ** ในทุกไฟล์ source ที่เกี่ยวข้องกับ PostgreSQL backend เพราะมันกำหนด type พื้นฐาน (`int32`, `int64`, `bool`, `Datum` ฯลฯ) และ platform-specific macro ที่ header อื่น ๆ ต้องพึ่งพา
2. `#include "fmgr.h"` — ให้มาโครสำหรับ function manager เช่น `PG_FUNCTION_INFO_V1`, `PG_GETARG_INT32`, `PG_RETURN_INT32`
3. `PG_MODULE_MAGIC;` — ประกาศ magic block (เขียนครั้งเดียวต่อ shared library)
4. `PG_FUNCTION_INFO_V1(add_one);` — ประกาศว่า `add_one` เป็นฟังก์ชัน V1-calling-convention
5. `Datum add_one(PG_FUNCTION_ARGS)` — signature ของฟังก์ชัน ต้องคืนค่าเป็น `Datum` เสมอ ไม่ว่าฟังก์ชันจริง ๆ จะคืนค่าชนิดอะไรก็ตาม เพราะ `Datum` คือ "กล่องสากล" ตามที่อธิบายไปก่อนหน้า
6. `PG_GETARG_INT32(0)` — ดึง argument ตัวแรก (index 0) แปลงจาก `Datum` เป็น `int32`
7. `PG_RETURN_INT32(arg + 1)` — แปลงค่า `int32` กลับเป็น `Datum` และ `return` ออกจากฟังก์ชัน (มาโครนี้มี `return` ซ่อนอยู่ในตัว)

> **ข้อควรระวัง**: `PG_RETURN_INT32` คือมาโครที่มีการ `return` แฝงอยู่ ดังนั้นห้ามเขียน `return PG_RETURN_INT32(...)` เพราะจะกลายเป็น syntax ผิด — เขียนแค่ `PG_RETURN_INT32(...);` เฉย ๆ

เมื่อเขียนโค้ดนี้เสร็จ เรายังไม่สามารถใช้งานได้ทันที ต้องผ่านขั้นตอนการจัดการ argument/return แบบละเอียดขึ้น (Step 845), เขียน Makefile (Step 846), และเขียน control/SQL file (Step 847) ก่อน จึงจะ compile และเรียกใช้งานผ่าน SQL ได้จริง

---

## Step 845: การจัดการ Argument และ Return Value

ใน Step 844 เราเห็นตัวอย่างการใช้ `PG_GETARG_INT32` และ `PG_RETURN_INT32` ไปแล้ว ใน Step นี้เราจะมาดูมาโครแบบเต็มสำหรับชนิดข้อมูลต่าง ๆ และวิธีจัดการค่า NULL อย่างถูกต้อง

### ตารางมาโคร `PG_GETARG_*` และ `PG_RETURN_*` สำหรับชนิดข้อมูลที่ใช้บ่อย

| ชนิดข้อมูล SQL | C type | Getter macro | Return macro |
|---|---|---|---|
| `smallint` (int2) | `int16` | `PG_GETARG_INT16(n)` | `PG_RETURN_INT16(x)` |
| `integer` (int4) | `int32` | `PG_GETARG_INT32(n)` | `PG_RETURN_INT32(x)` |
| `bigint` (int8) | `int64` | `PG_GETARG_INT64(n)` | `PG_RETURN_INT64(x)` |
| `real` (float4) | `float4` | `PG_GETARG_FLOAT4(n)` | `PG_RETURN_FLOAT4(x)` |
| `double precision` (float8) | `float8` | `PG_GETARG_FLOAT8(n)` | `PG_RETURN_FLOAT8(x)` |
| `boolean` | `bool` | `PG_GETARG_BOOL(n)` | `PG_RETURN_BOOL(x)` |
| `text` / `varchar` | `text *` | `PG_GETARG_TEXT_PP(n)` | `PG_RETURN_TEXT_P(x)` |
| `cstring` | `char *` | `PG_GETARG_CSTRING(n)` | `PG_RETURN_CSTRING(x)` |
| `bytea` | `bytea *` | `PG_GETARG_BYTEA_PP(n)` | `PG_RETURN_BYTEA_P(x)` |
| `numeric` | `Numeric` | `PG_GETARG_NUMERIC(n)` | `PG_RETURN_NUMERIC(x)` |
| `timestamp` | `Timestamp` | `PG_GETARG_TIMESTAMP(n)` | `PG_RETURN_TIMESTAMP(x)` |
| `date` | `DateADT` | `PG_GETARG_DATEADT(n)` | `PG_RETURN_DATEADT(x)` |
| `oid` | `Oid` | `PG_GETARG_OID(n)` | `PG_RETURN_OID(x)` |
| `void` (ไม่คืนค่า) | — | — | `PG_RETURN_VOID()` |

โดยที่ `n` คือ index ของ argument นับจาก 0 (argument ตัวแรกคือ `0`, ตัวที่สองคือ `1` เป็นต้นไป)

> **หมายเหตุเรื่อง `TEXT_PP`**: มาโคร `PG_GETARG_TEXT_PP` (PP ย่อมาจาก "possibly packed/possibly not") เป็นวิธีที่แนะนำในการอ่านค่า `text` เพราะรองรับทั้งข้อมูลที่ถูกบีบอัด (compressed/TOASTed) และไม่ถูกบีบอัดได้อย่างถูกต้อง โดยไม่ต้อง copy ข้อมูลถ้าไม่จำเป็น (ต่างจากมาโครรุ่นเก่า `PG_GETARG_TEXT_P` ที่จะ detoast แบบ copy เสมอ)

### ตัวอย่าง: ฟังก์ชันที่รับหลาย argument ต่างชนิดกัน

```c
#include "postgres.h"
#include "fmgr.h"
#include "utils/builtins.h"   /* สำหรับ cstring_to_text, text_to_cstring */

PG_MODULE_MAGIC;

/*
 * repeat_char
 *      รับ character หนึ่งตัว (cstring) และจำนวนครั้ง (integer)
 *      คืนค่าเป็น text ที่เกิดจากการทำซ้ำตัวอักษรนั้น
 *      ตัวอย่าง: repeat_char('x', 5) -> 'xxxxx'
 */
PG_FUNCTION_INFO_V1(repeat_char);

Datum
repeat_char(PG_FUNCTION_ARGS)
{
    char   *ch    = PG_GETARG_CSTRING(0);
    int32   count = PG_GETARG_INT32(1);
    char   *buf;
    int32   i;

    if (count < 0)
        count = 0;

    buf = (char *) palloc(count + 1);

    for (i = 0; i < count; i++)
        buf[i] = ch[0];
    buf[count] = '\0';

    PG_RETURN_TEXT_P(cstring_to_text(buf));
}
```

### การจัดการค่า NULL

ค่า NULL ใน SQL เป็นแนวคิดที่ต้องจัดการอย่างระมัดระวังในฟังก์ชัน C มี 2 แนวทางหลัก:

#### แนวทางที่ 1: ประกาศฟังก์ชันเป็น `STRICT` (แนะนำสำหรับกรณีทั่วไป)

ถ้าฟังก์ชันถูกประกาศเป็น `STRICT` ใน SQL (`CREATE FUNCTION ... LANGUAGE C STRICT`) PostgreSQL จะ**ตรวจสอบ NULL ให้อัตโนมัติก่อนเรียกฟังก์ชันของเรา** — ถ้า argument ตัวใดตัวหนึ่งเป็น NULL ฟังก์ชันจะไม่ถูกเรียกเลย และคืนค่า NULL ให้ทันที นี่คือแนวทางที่ปลอดภัยและง่ายที่สุด และเป็นค่าที่แนะนำเป็นค่าเริ่มต้น เพราะเราไม่ต้องเขียนโค้ดตรวจสอบ NULL เองเลยในตัวฟังก์ชัน

#### แนวทางที่ 2: ตรวจสอบ NULL เองด้วย `PG_ARGISNULL`

ถ้าฟังก์ชันต้องการ "ควบคุม logic เอง" เมื่อเจอ NULL (เช่น ต้องการคืนค่าเฉพาะแทนที่จะคืน NULL เสมอ หรือฟังก์ชันมีหลาย argument ที่บาง argument เป็น NULL ได้แต่บาง argument ไม่ได้) ต้องประกาศฟังก์ชันแบบ **ไม่ STRICT** และตรวจสอบด้วยมาโคร `PG_ARGISNULL(n)`

```c
#include "postgres.h"
#include "fmgr.h"

PG_MODULE_MAGIC;

/*
 * add_one_or_default
 *      รับ integer หนึ่งตัว (อาจเป็น NULL ได้)
 *      ถ้าเป็น NULL ให้คืนค่า 0 แทนที่จะคืน NULL
 *      ถ้าไม่ใช่ NULL ให้คืนค่าบวกหนึ่งตามปกติ
 *
 *      ฟังก์ชันนี้ *ห้าม* ประกาศเป็น STRICT ใน SQL
 *      มิฉะนั้น PostgreSQL จะคืนค่า NULL ให้เองโดยไม่เรียกฟังก์ชันนี้เลย
 */
PG_FUNCTION_INFO_V1(add_one_or_default);

Datum
add_one_or_default(PG_FUNCTION_ARGS)
{
    int32 result;

    if (PG_ARGISNULL(0))
    {
        result = 0;
    }
    else
    {
        int32 arg = PG_GETARG_INT32(0);
        result = arg + 1;
    }

    PG_RETURN_INT32(result);
}
```

**กฎสำคัญที่ต้องจำ**: ถ้าฟังก์ชันไม่ใช่ STRICT และมีโอกาสที่ argument จะเป็น NULL **ห้ามเรียก `PG_GETARG_*` กับ argument นั้นก่อนตรวจสอบ `PG_ARGISNULL`** เพราะถ้า argument เป็น NULL จริง ค่า `Datum` ของมันอาจไม่ได้ถูกกำหนดไว้อย่างมีความหมาย (ไม่มี pointer ที่ถูกต้องให้ dereference) การเรียก `PG_GETARG_TEXT_PP` กับ argument ที่เป็น NULL อาจทำให้เกิด **segmentation fault** และทำให้ backend process ทั้งตัว crash ทันที

### การคืนค่า NULL จากฟังก์ชัน

ถ้าฟังก์ชันต้องการคืนค่า NULL กลับไปยัง SQL ให้ใช้มาโคร `PG_RETURN_NULL()`

```c
PG_FUNCTION_INFO_V1(safe_divide);

Datum
safe_divide(PG_FUNCTION_ARGS)
{
    float8 numerator   = PG_GETARG_FLOAT8(0);
    float8 denominator = PG_GETARG_FLOAT8(1);

    /* ป้องกันการหารด้วยศูนย์ โดยคืนค่า NULL แทนที่จะ error */
    if (denominator == 0.0)
        PG_RETURN_NULL();

    PG_RETURN_FLOAT8(numerator / denominator);
}
```

> ฟังก์ชันที่มีโอกาสคืนค่า `PG_RETURN_NULL()` ก็ต้องประกาศเป็น non-STRICT เช่นกัน (หรืออย่างน้อยต้องแน่ใจว่า return type รองรับ NULL ตามปกติของ SQL ซึ่งเป็นค่าเริ่มต้นอยู่แล้ว)

เราจะเห็นการใช้ทั้ง STRICT และ non-STRICT ในตัวอย่างจริงของ Step 850 ต่อไป

---

## Step 846: Makefile ด้วย PGXS

เมื่อมีไฟล์ `pgtutorial.c` พร้อมแล้ว ขั้นตอนถัดไปคือการเขียน Makefile ที่ใช้ PGXS เพื่อ compile และ install extension

### โครงสร้าง Makefile มาตรฐานสำหรับ PGXS

สร้างไฟล์ชื่อ `Makefile` (ไม่มีนามสกุล) ใน directory เดียวกับ `pgtutorial.c`:

```makefile
# Makefile สำหรับ extension pgtutorial

MODULES = pgtutorial
EXTENSION = pgtutorial
DATA = pgtutorial--1.0.sql

PG_CONFIG = pg_config
PGXS := $(shell $(PG_CONFIG) --pgxs)
include $(PGXS)
```

### อธิบายแต่ละบรรทัด

- `MODULES = pgtutorial` — บอกว่ามี source ไฟล์ `pgtutorial.c` ที่ต้อง compile เป็น `pgtutorial.so` (ถ้ามีหลาย module ในโปรเจกต์เดียวกัน สามารถระบุหลายชื่อคั่นด้วยช่องว่างได้ เช่น `MODULES = foo bar`) ใช้ตัวแปรนี้เมื่อแต่ละไฟล์ `.c` compile แยกเป็น `.so` คนละไฟล์
- `EXTENSION = pgtutorial` — บอกชื่อ extension เพื่อให้ PGXS รู้ว่าต้อง install control file `pgtutorial.control` ด้วย
- `DATA = pgtutorial--1.0.sql` — ระบุ SQL script file ที่ต้อง copy ไปยัง `SHAREDIR/extension/` ตอน `make install` (ถ้ามีหลาย version file เช่น upgrade script สามารถระบุหลายไฟล์คั่นด้วยช่องว่างได้)
- `PG_CONFIG = pg_config` — ระบุว่าจะใช้ `pg_config` ตัวไหน (ปกติปล่อยเป็นค่า default นี้ก็พอ แต่ถ้ามีหลายเวอร์ชัน สามารถ override เป็น full path เช่น `PG_CONFIG = /usr/lib/postgresql/16/bin/pg_config`)
- `PGXS := $(shell $(PG_CONFIG) --pgxs)` — เรียก `pg_config --pgxs` เพื่อหา path ของไฟล์ PGXS makefile
- `include $(PGXS)` — บรรทัดสุดท้ายเสมอ ต้อง include ไฟล์ PGXS เพื่อดึง rules ทั้งหมดมาใช้ (target `all`, `install`, `clean` ฯลฯ)

### ตัวแปรอื่น ๆ ที่ใช้บ่อยใน Makefile แบบ PGXS

| ตัวแปร | ความหมาย |
|---|---|
| `MODULES` | รายชื่อ `.c` ไฟล์ (ไม่ใส่นามสกุล) ที่ compile แยกเป็น `.so` คนละไฟล์ |
| `MODULE_big` | ใช้แทน `MODULES` เมื่อ module เดียวประกอบจาก `.c` หลายไฟล์ ร่วมกับตัวแปร `OBJS` |
| `OBJS` | รายชื่อ `.o` object files (ใช้คู่กับ `MODULE_big`) |
| `EXTENSION` | ชื่อ extension (สำหรับ install `.control` และ `.sql`) |
| `DATA` | รายชื่อ SQL script files ที่ต้อง install |
| `DOCS` | รายชื่อไฟล์เอกสารที่ต้อง install ไปยัง `doc/` directory |
| `REGRESS` | รายชื่อ regression test (สำหรับ `make installcheck`) |
| `PG_CPPFLAGS` | เพิ่ม preprocessor flags พิเศษ เช่น `-I` เพิ่ม include path |
| `SHLIB_LINK` | เพิ่ม linker flags พิเศษ เช่น `-lm` สำหรับ link math library |

### ตัวอย่าง Makefile สำหรับ extension ที่มีหลายไฟล์ `.c` รวมเป็น module เดียว

```makefile
# ตัวอย่างเมื่อมี pgtutorial.c และ pgtutorial_helpers.c รวมเป็น pgtutorial.so ตัวเดียว
MODULE_big = pgtutorial
OBJS = pgtutorial.o pgtutorial_helpers.o
EXTENSION = pgtutorial
DATA = pgtutorial--1.0.sql
SHLIB_LINK = -lm

PG_CONFIG = pg_config
PGXS := $(shell $(PG_CONFIG) --pgxs)
include $(PGXS)
```

### การ compile และ install

กลับมาที่ extension `pgtutorial` (ใช้ `MODULES` แบบเดียวไฟล์) รันคำสั่งดังนี้:

```bash
# อยู่ใน directory ที่มี pgtutorial.c, Makefile, pgtutorial.control, pgtutorial--1.0.sql
cd ~/dev/pgtutorial

# compile source code เป็น .so
make

# ตัวอย่าง output ที่คาดหวัง:
# gcc -Wall -Wmissing-prototypes ... -c -o pgtutorial.o pgtutorial.c
# gcc -Wall -Wmissing-prototypes ... -shared -o pgtutorial.so pgtutorial.o

# ล้างไฟล์ที่ compile ไว้ (ถ้าต้องการ build ใหม่ตั้งแต่ต้น)
make clean

# compile ใหม่แล้ว install ไฟล์ทั้งหมด (.so, .control, .sql)
# ไปยัง PKGLIBDIR และ SHAREDIR/extension/ ตามลำดับ
# มักต้องใช้ sudo เพราะ directory เหล่านี้เป็นของ root/postgres user
sudo make install

# ตัวอย่าง output ที่คาดหวัง:
# /usr/bin/mkdir -p '/usr/lib/postgresql/16/lib'
# /usr/bin/install -c -m 755 pgtutorial.so '/usr/lib/postgresql/16/lib/'
# /usr/bin/mkdir -p '/usr/share/postgresql/16/extension'
# /usr/bin/install -c -m 644 .../pgtutorial.control '.../extension/'
# /usr/bin/install -c -m 644 .../pgtutorial--1.0.sql '.../extension/'
```

### เปิดใช้งาน extension ใน database

หลังจาก `make install` สำเร็จ ไฟล์ทุกอย่างพร้อมแล้ว แต่ extension ยัง**ไม่ถูกโหลดเข้า database ใด ๆ** จนกว่าจะรันคำสั่ง SQL:

```sql
-- เชื่อมต่อกับ database ที่ต้องการใช้งาน extension แล้วรัน
CREATE EXTENSION pgtutorial;

-- ตรวจสอบว่า extension ถูกติดตั้งแล้ว
SELECT extname, extversion FROM pg_extension WHERE extname = 'pgtutorial';

-- ทดสอบเรียกใช้ฟังก์ชัน
SELECT add_one(41);
--  add_one
-- ---------
--       42
-- (1 row)
```

ถ้าเจอ error เช่น `could not open extension control file` หมายความว่า `.control` ไฟล์ยังไม่ได้ถูก install ไปยัง directory ที่ถูกต้อง หรือ `pg_config` ที่ใช้ตอน `make install` ไม่ตรงกับ PostgreSQL server ที่กำลังรันอยู่ — ให้ตรวจสอบด้วย `pg_config --sharedir` และ `SHOW data_directory;` ว่าชี้ไปยัง instance เดียวกันหรือไม่

---

## Step 847: Control File และ SQL Definition File

ใน Step นี้เราจะลงรายละเอียดของไฟล์ `.control` และ `.sql` ที่กล่าวถึงคร่าว ๆ ไปแล้วใน Step 842 และ 846

### Control File แบบละเอียด

```ini
# pgtutorial.control
comment = 'ฟังก์ชันตัวอย่างสำหรับเรียนรู้การเขียน extension ด้วยภาษา C'
default_version = '1.0'
module_pathname = '$libdir/pgtutorial'
relocatable = true
```

รายละเอียดของแต่ละ parameter:

- **`comment`** — คำอธิบายสั้น ๆ ของ extension แสดงในผลลัพธ์ของ `\dx` ใน psql และ `SELECT * FROM pg_available_extensions;`
- **`default_version`** — เวอร์ชันเริ่มต้นที่จะถูกติดตั้งเมื่อรัน `CREATE EXTENSION pgtutorial;` โดยไม่ระบุ `VERSION` ต้องตรงกับชื่อไฟล์ SQL เช่น `pgtutorial--1.0.sql`
- **`module_pathname`** — path ไปยัง shared library โดยใช้ตัวแปร `$libdir` แทนที่ hardcode absolute path (เพื่อให้ extension ย้าย server ได้โดยไม่ต้องแก้ path) ค่านี้จะถูกใช้แทนที่ placeholder `MODULE_PATHNAME` ในไฟล์ `.sql` โดยอัตโนมัติ
- **`relocatable`** — ถ้าเป็น `true` หมายความว่า extension สามารถถูกย้าย schema ได้หลัง install ด้วยคำสั่ง `ALTER EXTENSION ... SET SCHEMA ...` (extension ที่มี object ผูกกับ schema เฉพาะ เช่น operator ที่ reference schema ชื่อตรง ๆ ควรตั้งเป็น `false`)

### Parameter เพิ่มเติมที่พบได้บ่อย (ไม่บังคับ)

| Parameter | ความหมาย |
|---|---|
| `requires` | รายชื่อ extension อื่นที่ต้องติดตั้งก่อน เช่น `requires = 'plpgsql'` |
| `schema` | บังคับให้ extension ติดตั้งใน schema ที่กำหนดตายตัวเท่านั้น |
| `superuser` | ถ้า `false` อนุญาตให้ non-superuser ที่มีสิทธิ์ `CREATE` ติดตั้ง extension ได้ (ต้องใช้คู่กับ `trusted = true` ด้วย PostgreSQL รุ่นใหม่) |
| `trusted` | (PostgreSQL 13+) ถ้า `true` อนุญาตให้ non-superuser ที่มีสิทธิ์ `CREATE` บน database ติดตั้ง extension นี้ได้โดยไม่ต้องเป็น superuser — ปกติใช้กับ extension ที่ปลอดภัย ไม่กระทบระบบ |

ตัวอย่าง control file ที่ระบุ parameter เพิ่มเติม:

```ini
# pgtutorial.control
comment = 'ฟังก์ชันตัวอย่างสำหรับเรียนรู้การเขียน extension ด้วยภาษา C'
default_version = '1.0'
module_pathname = '$libdir/pgtutorial'
relocatable = true
requires = ''
```

### SQL Definition File แบบละเอียด

ไฟล์ `pgtutorial--1.0.sql` คือชุดคำสั่ง SQL ที่จะถูก execute เมื่อรัน `CREATE EXTENSION pgtutorial;`

```sql
-- pgtutorial--1.0.sql

-- บรรทัดนี้ป้องกันไม่ให้ผู้ใช้รัน SQL script นี้ตรง ๆ ด้วย psql \i
-- (ต้องผ่านคำสั่ง CREATE EXTENSION เท่านั้น)
\echo Use "CREATE EXTENSION pgtutorial" to load this file. \quit

CREATE FUNCTION add_one(integer)
RETURNS integer
AS 'MODULE_PATHNAME', 'add_one'
LANGUAGE C IMMUTABLE STRICT;

COMMENT ON FUNCTION add_one(integer) IS
    'บวกค่า integer ที่รับเข้ามาด้วยหนึ่ง แล้วคืนค่ากลับ';
```

### อธิบายส่วนประกอบของ `CREATE FUNCTION`

```sql
CREATE FUNCTION add_one(integer)
RETURNS integer
AS 'MODULE_PATHNAME', 'add_one'
LANGUAGE C IMMUTABLE STRICT;
```

- `add_one(integer)` — ชื่อฟังก์ชันและชนิด argument ตามที่จะเรียกใช้จาก SQL
- `RETURNS integer` — ชนิดข้อมูลที่ฟังก์ชันคืนค่า
- `AS 'MODULE_PATHNAME', 'add_one'` — บอก PostgreSQL ว่าให้โหลด shared library ตาม `MODULE_PATHNAME` (ซึ่งจะถูกแทนที่ด้วยค่า `module_pathname` จาก control file คือ `$libdir/pgtutorial`) แล้วเรียก symbol ชื่อ `add_one` ภายในนั้น — **ชื่อ symbol ต้องตรงกับชื่อฟังก์ชัน C ที่ประกาศด้วย `PG_FUNCTION_INFO_V1` เป๊ะ ๆ**
- `LANGUAGE C` — บอกว่าเป็นฟังก์ชันที่เขียนด้วยภาษา C (ตรงข้ามกับ `LANGUAGE plpgsql`, `LANGUAGE sql` ฯลฯ)
- `IMMUTABLE` — hint บอก planner ว่าฟังก์ชันนี้คืนค่าเดิมเสมอสำหรับ input เดิม (ช่วยให้ optimizer ทำ constant folding หรือ cache ผลลัพธ์ได้) ใช้ `STABLE` ถ้าฟังก์ชันขึ้นกับข้อมูลใน transaction เดียวกัน (เช่นอ่านตาราง) และใช้ `VOLATILE` (ค่า default ถ้าไม่ระบุ) ถ้าฟังก์ชันมี side effect หรือผลลัพธ์เปลี่ยนแปลงได้แม้ input เดิม (เช่น `random()`, `now()`)
- `STRICT` — ตามที่อธิบายใน Step 845 คือบอกให้ PostgreSQL คืนค่า NULL อัตโนมัติถ้ามี argument ตัวใดเป็น NULL โดยไม่ต้องเรียกฟังก์ชันจริง

### ชื่อ SQL function กับชื่อ C function ไม่จำเป็นต้องตรงกัน

จุดที่มักสร้างความสับสน คือชื่อฟังก์ชันใน SQL (`add_one`) ไม่จำเป็นต้องตรงกับชื่อ symbol ใน C เสมอไป เราสามารถตั้งชื่อฟังก์ชัน SQL คนละชื่อกับ C function ได้ เช่น:

```sql
CREATE FUNCTION plus_one(integer)
RETURNS integer
AS 'MODULE_PATHNAME', 'add_one'  -- ชื่อ symbol C ยังคงเป็น add_one
LANGUAGE C IMMUTABLE STRICT;
```

กรณีนี้เมื่อเรียก `SELECT plus_one(5);` จาก SQL จะไปเรียก C function ที่ชื่อ `add_one` ภายใน shared library จริง ๆ

### Function Overloading — argument ต่างชนิด ต้องมี C function แยกกัน

PostgreSQL รองรับ function overloading ในระดับ SQL (ฟังก์ชันชื่อเดียวกันแต่ signature ต่างกันได้) แต่ในระดับ C แต่ละ overload ต้องมี **C function แยกกันคนละตัว** (คนละ symbol name) เสมอ เพราะ C ไม่รองรับ overloading โดยตรง เช่น:

```sql
CREATE FUNCTION add_one(integer) RETURNS integer
AS 'MODULE_PATHNAME', 'add_one_int4'
LANGUAGE C IMMUTABLE STRICT;

CREATE FUNCTION add_one(bigint) RETURNS bigint
AS 'MODULE_PATHNAME', 'add_one_int8'
LANGUAGE C IMMUTABLE STRICT;
```

### Extension Upgrade Script

เมื่อ extension มีการพัฒนาต่อ (เพิ่มฟังก์ชันใหม่ แก้ signature เดิม) เราสร้างไฟล์ upgrade script ในรูปแบบ `<extension>--<old_version>--<new_version>.sql` เช่น `pgtutorial--1.0--1.1.sql`:

```sql
-- pgtutorial--1.0--1.1.sql

\echo Use "ALTER EXTENSION pgtutorial UPDATE TO '1.1'" to load this file. \quit

CREATE FUNCTION add_n(integer, integer)
RETURNS integer
AS 'MODULE_PATHNAME', 'add_n'
LANGUAGE C IMMUTABLE STRICT;
```

จากนั้นอัปเดต `default_version` ใน control file เป็น `1.1` และเพิ่ม `pgtutorial--1.0--1.1.sql` เข้าไปในตัวแปร `DATA` ของ Makefile ผู้ใช้ที่ install `pgtutorial 1.0` ไว้แล้วสามารถอัปเกรดด้วย:

```sql
ALTER EXTENSION pgtutorial UPDATE TO '1.1';
```

PostgreSQL จะหาเส้นทาง upgrade ที่สั้นที่สุดจาก version ปัจจุบันไปยัง version เป้าหมายโดยอัตโนมัติ (เช่นถ้ามี 1.0→1.1 และ 1.1→1.2 มันจะรันทั้งสองไฟล์ต่อกันถ้าขอ upgrade จาก 1.0 ไป 1.2 โดยตรง)

---

## Step 848: Memory Management ใน PostgreSQL C API

การจัดการหน่วยความจำเป็นหนึ่งในจุดที่อันตรายที่สุดเมื่อเขียน extension ด้วย C เพราะ error เรื่อง memory (เช่น memory leak, use-after-free, buffer overflow) อาจทำให้ backend process crash หรือทำให้ข้อมูลเสียหายได้ PostgreSQL จึงมีระบบ memory management ของตัวเองที่**ต้อง**ใช้แทน `malloc`/`free` มาตรฐานของ C

### ทำไมต้องใช้ `palloc`/`pfree` แทน `malloc`/`free`

PostgreSQL ใช้แนวคิด **Memory Context** — คือ "ถัง" หน่วยความจำที่มีอายุการใช้งาน (lifetime) ผูกกับ scope การทำงานบางอย่าง เช่น:

- `CurrentMemoryContext` — memory context ที่กำลัง active อยู่ ณ ขณะนั้น (มักผูกกับช่วงการเรียกฟังก์ชัน หรือ query execution)
- `TopMemoryContext` — context ที่มีอายุยาวที่สุด อยู่ตลอด lifetime ของ backend process
- `MessageContext` — context ที่ถูกล้างทุกครั้งที่ backend รับคำสั่งใหม่จาก client
- `ExprContext` (per-tuple context) — context ที่ถูกล้างทุก ๆ row ที่ query ประมวลผล เหมาะกับหน่วยความจำชั่วคราวที่ใช้แล้วทิ้งทันที

ข้อดีของระบบนี้คือ **เราไม่จำเป็นต้อง `pfree()` ทุกครั้งที่ palloc** เพราะเมื่อ memory context หมดอายุ (เช่น function call จบ, query จบ) หน่วยความจำทั้งหมดใน context นั้นจะถูก free อัตโนมัติทีเดียวทั้งก้อน — นี่คือกลไกที่ป้องกัน memory leak ในระดับ query โดยไม่ต้องพึ่งวินัยของโปรแกรมเมอร์ทั้งหมด

```c
#include "postgres.h"
#include "fmgr.h"
#include "utils/builtins.h"
#include "utils/palloc.h"

PG_MODULE_MAGIC;

/*
 * build_greeting
 *      ตัวอย่างการใช้ palloc เพื่อสร้าง buffer ชั่วคราว
 */
PG_FUNCTION_INFO_V1(build_greeting);

Datum
build_greeting(PG_FUNCTION_ARGS)
{
    text   *name_arg = PG_GETARG_TEXT_PP(0);
    char   *name     = text_to_cstring(name_arg);
    char   *result_buf;
    size_t  needed;
    text   *result;

    /* คำนวณขนาด buffer ที่ต้องการ แล้ว palloc */
    needed = strlen(name) + strlen("สวัสดีคุณ, !") + 1;
    result_buf = (char *) palloc(needed);

    snprintf(result_buf, needed, "สวัสดีคุณ%s!", name);

    result = cstring_to_text(result_buf);

    /* pfree เป็นทางเลือก ไม่บังคับ เพราะ memory context
     * จะเก็บกวาดให้เองเมื่อ function call จบ
     * แต่ถ้าฟังก์ชันมี loop ขนาดใหญ่และ allocate ซ้ำ ๆ จำนวนมาก
     * ควร pfree buffer ชั่วคราวที่ไม่ใช้แล้วเพื่อลด peak memory usage
     */
    pfree(result_buf);
    pfree(name);

    PG_RETURN_TEXT_P(result);
}
```

### ฟังก์ชันสำคัญในตระกูล `palloc`

| ฟังก์ชัน | ความหมาย |
|---|---|
| `palloc(size)` | จอง memory ขนาด `size` bytes ใน `CurrentMemoryContext` (คล้าย `malloc` แต่ error ทันทีถ้าจองไม่สำเร็จ ไม่คืนค่า NULL แบบ `malloc`) |
| `palloc0(size)` | เหมือน `palloc` แต่ zero-initialize หน่วยความจำทั้งหมด (คล้าย `calloc`) |
| `repalloc(ptr, size)` | ขยาย/ย่อขนาด block ที่จองไว้แล้ว (คล้าย `realloc`) |
| `pfree(ptr)` | คืนหน่วยความจำกลับให้ memory context (คล้าย `free`) — เป็น**ทางเลือก** ไม่บังคับเรียกเสมอ |
| `MemoryContextAlloc(context, size)` | จองหน่วยความจำใน memory context ที่ระบุเจาะจง (ไม่ใช่ `CurrentMemoryContext`) |

### กฎเหล็กที่ต้องจำ

> **ห้ามใช้ `malloc()`/`free()`/`strdup()` มาตรฐานของ C ในโค้ดที่ทำงานกับข้อมูลที่จะถูกส่งกลับให้ PostgreSQL (เช่น return value, ข้อมูลที่ palloc ต่อ) ให้ใช้ `palloc`/`pfree` เสมอ**

เหตุผลคือ:

1. หน่วยความจำที่ `malloc()` จองมา จะ**ไม่ถูก track โดย memory context ของ PostgreSQL** ทำให้ถ้าลืม `free()` จะเกิด memory leak ถาวรตลอดอายุ backend process (ต่างจาก `palloc` ที่ถูกเก็บกวาดอัตโนมัติเมื่อ context หมดอายุ)
2. ถ้าเกิด error ระหว่างทาง (เช่น `ereport(ERROR, ...)`) PostgreSQL ใช้กลไก `longjmp` เพื่อ unwind กลับไปที่ transaction/query level ซึ่งจะข้ามโค้ดที่ควรจะเรียก `free()` ไป — แต่ memory context จะถูกล้างให้เองโดยอัตโนมัติไม่ว่าจะ error หรือไม่ก็ตาม นี่คือเหตุผลสำคัญที่สุดที่ควรใช้ `palloc` เพราะมันทำงานร่วมกับ error handling ได้อย่างปลอดภัย ในขณะที่ `malloc` จะรั่วไหลทันทีถ้า error เกิดขึ้นก่อนถึงบรรทัด `free()`

### แนวคิด Memory Context แบบเจาะลึก

```c
#include "utils/memutils.h"

/* ตัวอย่าง: สลับไปทำงานใน memory context อื่นชั่วคราว
 * ใช้เมื่อต้องการให้หน่วยความจำที่ palloc มีอายุยาวกว่า
 * function call ปัจจุบัน เช่น ต้องเก็บ cache ข้าม query
 */
MemoryContext oldcontext;
MemoryContext my_long_lived_context;

/* สร้าง context ใหม่ที่ลูกของ TopMemoryContext (อายุยาวตลอด backend) */
my_long_lived_context = AllocSetContextCreate(TopMemoryContext,
                                               "MyExtensionCache",
                                               ALLOCSET_DEFAULT_SIZES);

/* สลับ CurrentMemoryContext ไปเป็น context ใหม่ */
oldcontext = MemoryContextSwitchTo(my_long_lived_context);

/* ทุกอย่างที่ palloc ในช่วงนี้ จะอยู่ใน my_long_lived_context
 * ซึ่งจะไม่ถูกล้างจนกว่าจะ MemoryContextDelete() เอง หรือ backend หยุดทำงาน
 */
char *cached_data = palloc(1024);

/* สลับกลับไปยัง context เดิม ก่อน function จะ return */
MemoryContextSwitchTo(oldcontext);
```

การใช้ memory context แบบกำหนดเองนี้ ส่วนใหญ่จำเป็นเฉพาะกรณีที่ซับซ้อน เช่น เขียน custom cache, background worker, หรือ Set-Returning Function (SRF) ที่ต้องรักษา state ข้ามการเรียกหลายครั้ง สำหรับฟังก์ชันทั่วไปที่เราเขียนในบทนี้ การใช้ `palloc`/`pfree` ใน `CurrentMemoryContext` แบบธรรมดาก็เพียงพอแล้ว

### ข้อควรระวังเรื่อง TOAST และ detoast

เมื่อรับ argument ที่เป็น `text`, `bytea`, `numeric` ฯลฯ (varlena types) ค่าที่ได้จาก `PG_GETARG_TEXT_PP` อาจเป็นข้อมูลที่ถูก **TOAST** (บีบอัดหรือเก็บแยก out-of-line เนื่องจากมีขนาดใหญ่) มาโคร `_PP` (packed) จะจัดการ detoast ให้อัตโนมัติเมื่อจำเป็น แต่หน่วยความจำที่ได้จากการ detoast ก็ยังเป็นหน่วยความจำที่ palloc มาเช่นกัน จึงอยู่ภายใต้กฎ memory context เดียวกันทั้งหมด ไม่ต้องจัดการเป็นพิเศษ

---

## Step 849: Error Handling

การรายงาน error อย่างถูกวิธีเป็นสิ่งสำคัญมากในฟังก์ชัน C เพราะถ้าเราใช้ `printf()` หรือปล่อยให้ค่า error condition หลุดไปโดยไม่จัดการ อาจทำให้พฤติกรรมของ PostgreSQL ผิดเพี้ยนหรือ crash ได้ PostgreSQL มีระบบ error reporting มาตรฐานผ่านฟังก์ชัน `ereport()` และ `elog()`

### `ereport()` — วิธีมาตรฐานในการรายงาน error/notice/warning

```c
ereport(ERROR,
        (errcode(ERRCODE_INVALID_PARAMETER_VALUE),
         errmsg("ค่าที่ป้อนต้องมากกว่าศูนย์"),
         errdetail("ค่าที่ได้รับคือ %d", value),
         errhint("กรุณาตรวจสอบค่า input ก่อนเรียกฟังก์ชันนี้")));
```

โครงสร้างของ `ereport()`:

- **argument แรก** คือ error level (ดูตารางด้านล่าง)
- **argument ที่สอง** คือ list ของฟังก์ชันย่อยที่คั่นด้วยวงเล็บ (สังเกตว่าทั้งหมดอยู่ใน parenthesis เดียวกัน — นี่คือ syntax เฉพาะของมาโครนี้ ห้ามลืมวงเล็บชั้นนอก)
  - `errcode(...)` — ระบุ SQLSTATE error code (ใช้ constant ที่ PostgreSQL เตรียมไว้ใน `utils/errcodes.h` เช่น `ERRCODE_INVALID_PARAMETER_VALUE`, `ERRCODE_DIVISION_BY_ZERO`, `ERRCODE_NUMERIC_VALUE_OUT_OF_RANGE`)
  - `errmsg(...)` — ข้อความหลักของ error (รองรับ `printf`-style format string)
  - `errdetail(...)` — ข้อความรายละเอียดเพิ่มเติม (ไม่บังคับ)
  - `errhint(...)` — คำแนะนำสำหรับผู้ใช้ว่าควรแก้ไขอย่างไร (ไม่บังคับ)

### Error Level ต่าง ๆ

| Level | ความหมาย | ผลลัพธ์ |
|---|---|---|
| `DEBUG1`–`DEBUG5` | ข้อความ debug สำหรับ developer | แสดงเฉพาะเมื่อตั้งค่า `log_min_messages` ต่ำมาก ไม่กระทบการทำงาน |
| `NOTICE` | ข้อความแจ้งเตือนทั่วไป | แสดงให้ client เห็น แต่**ไม่หยุดการทำงาน** ของ statement |
| `WARNING` | ข้อความเตือนที่สำคัญกว่า NOTICE | แสดงให้ client เห็น แต่**ไม่หยุดการทำงาน** ของ statement เช่นกัน |
| `ERROR` | ข้อผิดพลาดที่ทำให้ statement/transaction ปัจจุบันต้อง**ยกเลิก** | ใช้กลไก `longjmp` เพื่อ unwind stack กลับไปยัง transaction control ระดับบนสุด ยกเลิก query ปัจจุบันทันที (แต่ **ไม่ทำให้ backend process ตาย** — connection ยังคงอยู่ พร้อมรับคำสั่งถัดไป) |
| `FATAL` | ข้อผิดพลาดร้ายแรงระดับ session | ตัดการเชื่อมต่อ (disconnect) ของ session ปัจจุบันทันที |
| `PANIC` | ข้อผิดพลาดร้ายแรงระดับระบบ | ทำให้ **PostgreSQL server ทั้งคลัสเตอร์ restart** (ใช้เฉพาะกรณีข้อมูล corrupt ระดับ critical เท่านั้น extension ทั่วไปไม่ควรใช้ level นี้) |

> **ข้อควรจำ**: ในโค้ด extension ทั่วไป เราแทบไม่มีเหตุผลต้องใช้ `FATAL` หรือ `PANIC` เลย — ควรใช้ `ERROR` สำหรับกรณีข้อมูล input ผิดพลาด, `WARNING`/`NOTICE` สำหรับแจ้งเตือนที่ไม่ต้องหยุดการทำงาน

### ตัวอย่างการใช้ `ereport(ERROR, ...)` ตรวจสอบ input

```c
#include "postgres.h"
#include "fmgr.h"
#include "utils/builtins.h"
#include "utils/elog.h"

PG_MODULE_MAGIC;

/*
 * safe_sqrt
 *      คำนวณ square root แบบตรวจสอบ input ก่อน
 *      ถ้า input ติดลบ ให้ raise ERROR แทนที่จะคืนค่า NaN แบบเงียบ ๆ
 */
PG_FUNCTION_INFO_V1(safe_sqrt);

Datum
safe_sqrt(PG_FUNCTION_ARGS)
{
    float8 value = PG_GETARG_FLOAT8(0);

    if (value < 0.0)
        ereport(ERROR,
                (errcode(ERRCODE_INVALID_ARGUMENT_FOR_POWER_FUNCTION),
                 errmsg("ไม่สามารถหา square root ของค่าติดลบได้: %g", value),
                 errhint("กรุณาตรวจสอบว่าค่าที่ส่งเข้ามาเป็นค่าที่ไม่ติดลบ")));

    PG_RETURN_FLOAT8(sqrt(value));
}
```

เมื่อเรียกใช้จาก SQL:

```sql
SELECT safe_sqrt(-4.0);
-- ERROR:  ไม่สามารถหา square root ของค่าติดลบได้: -4
-- HINT:  กรุณาตรวจสอบว่าค่าที่ส่งเข้ามาเป็นค่าที่ไม่ติดลบ

SELECT safe_sqrt(16.0);
--  safe_sqrt
-- -----------
--          4
-- (1 row)
```

สังเกตว่าเมื่อเจอ `ERROR` เฉพาะ statement นั้น fail แต่ session/connection ยังใช้งานต่อได้ตามปกติทันที (ถ้าไม่ได้อยู่ใน explicit transaction ที่ยังไม่ `ROLLBACK`)

### `elog()` — เวอร์ชันย่อของ `ereport()`

`elog()` เป็นมาโครที่ง่ายกว่า ใช้เมื่อไม่ต้องการระบุ SQLSTATE code หรือ detail/hint เพิ่มเติม เหมาะสำหรับข้อความ debug ภายในของ developer มากกว่าข้อความที่จะแสดงให้ end user เห็น

```c
elog(NOTICE, "กำลังประมวลผล record ที่ %d จากทั้งหมด %d", i, total);
elog(WARNING, "พบค่าที่ผิดปกติ แต่ยังคงดำเนินการต่อ: %s", value);
elog(DEBUG1, "เข้าสู่ฟังก์ชัน my_function ด้วย argument = %d", arg);
```

`elog(level, format, ...)` เทียบเท่ากับ `ereport(level, (errmsg_internal(format, ...)))` โดยประมาณ — ข้อแตกต่างสำคัญคือ `elog()` ใช้ `errmsg_internal` ซึ่ง**ไม่ถูกแปลภาษา (not translated)** และควรใช้กับข้อความสำหรับ developer/internal debugging เท่านั้น ส่วนข้อความที่ end user จะเห็น (เช่น error จาก invalid input) ควรใช้ `ereport()` กับ `errmsg()` เพราะรองรับการแปลภาษาผ่านระบบ gettext ของ PostgreSQL

### กฎการเขียนโค้ดหลัง `ereport(ERROR, ...)`

Compiler และ PostgreSQL รู้ว่า `ereport(ERROR, ...)` จะไม่ return กลับมา (มันถูก mark ด้วย `pg_attribute_noreturn()` ภายใน) ดังนั้นโค้ดหลังจากนั้นจะไม่ถูก execute เลย — เราสามารถเขียนโค้ดแบบ early-return pattern ได้อย่างปลอดภัยโดยไม่ต้องมี `return` หลัง `ereport(ERROR, ...)`:

```c
if (input_is_invalid)
    ereport(ERROR, (errmsg("ข้อมูล input ไม่ถูกต้อง")));
    /* ไม่ต้องมี return ตรงนี้ เพราะ ereport(ERROR, ...) ไม่ return กลับมา */

/* โค้ดส่วนนี้จะทำงานเฉพาะเมื่อ input ถูกต้องเท่านั้น */
```

### PG_TRY / PG_CATCH — ดักจับ error ภายในฟังก์ชัน (ใช้เท่าที่จำเป็น)

ในบางกรณีที่ต้องการ cleanup resource บางอย่าง (เช่น ปิด external file handle, คืนค่า lock พิเศษ) ก่อนที่ error จะ propagate ต่อไป สามารถใช้ `PG_TRY()`/`PG_CATCH()` ได้:

```c
PG_TRY();
{
    /* โค้ดที่อาจเกิด error */
    do_risky_operation();
}
PG_CATCH();
{
    /* cleanup ที่จำเป็น เช่น ปิด external resource */
    cleanup_external_resource();

    /* ส่ง error ต่อไปตามปกติ (re-throw) เกือบทุกกรณีควรทำแบบนี้ */
    PG_RE_THROW();
}
PG_END_TRY();
```

> **ข้อควรระวัง**: `PG_TRY`/`PG_CATCH` มีค่าใช้จ่าย (overhead) และความซับซ้อนสูงกว่าโค้ดปกติ ควรใช้เฉพาะเมื่อจำเป็นต้อง cleanup resource ที่อยู่นอกเหนือการดูแลของ memory context (เช่น file descriptor, external library handle) — สำหรับ memory ที่ palloc ไว้ ไม่จำเป็นต้องดักจับ error เพื่อ pfree เพราะ memory context จัดการให้อัตโนมัติอยู่แล้วตามที่อธิบายใน Step 848

---

## Step 850: แบบฝึกหัดรวม — เขียน Extension สมบูรณ์

ถึงเวลารวบยอดทุกอย่างที่เรียนมา เราจะสร้าง extension ชื่อ `thaibiz` ที่มีฟังก์ชันคำนวณธุรกิจจริง 2 ตัว:

1. **`haversine_km(lat1, lon1, lat2, lon2)`** — คำนวณระยะทางระหว่างพิกัด GPS สองจุดด้วยสูตร Haversine (หน่วยกิโลเมตร) ใช้ในระบบ logistics, delivery app, ride-hailing
2. **`thai_id_checksum_valid(text)`** — ตรวจสอบความถูกต้องของเลขบัตรประชาชนไทย 13 หลักด้วยสูตร checksum มาตรฐาน ใช้ในระบบ KYC, HR, e-commerce

### โครงสร้างไฟล์ทั้งหมด

```
thaibiz/
├── thaibiz.c            # source code หลัก
├── thaibiz.control      # control file
├── thaibiz--1.0.sql     # SQL definition
└── Makefile             # PGXS build script
```

### ไฟล์ 1: `thaibiz.c`

```c
/*
 * thaibiz.c
 *      Extension รวมฟังก์ชันคำนวณธุรกิจสำหรับตลาดไทย
 *
 *      1. haversine_km          - คำนวณระยะทางระหว่างพิกัด GPS (สูตร Haversine)
 *      2. thai_id_checksum_valid - ตรวจสอบ checksum เลขบัตรประชาชนไทย 13 หลัก
 */

#include "postgres.h"
#include "fmgr.h"
#include "utils/builtins.h"
#include "varatt.h"

#include <math.h>
#include <ctype.h>

PG_MODULE_MAGIC;

/* รัศมีเฉลี่ยของโลกเป็นกิโลเมตร ตามมาตรฐาน WGS84 mean radius */
#define EARTH_RADIUS_KM 6371.0088

/* แปลงองศาเป็นเรเดียน */
static inline double
deg2rad(double deg)
{
    return deg * (M_PI / 180.0);
}

/*
 * haversine_km
 *      รับพิกัด (latitude, longitude) ของจุดที่ 1 และจุดที่ 2 หน่วยองศา
 *      คืนค่าระยะทางเป็นเส้นตรงบนผิวโลก (great-circle distance) หน่วยกิโลเมตร
 *
 *      SQL: haversine_km(lat1 float8, lon1 float8, lat2 float8, lon2 float8)
 *           RETURNS float8
 */
PG_FUNCTION_INFO_V1(haversine_km);

Datum
haversine_km(PG_FUNCTION_ARGS)
{
    float8 lat1 = PG_GETARG_FLOAT8(0);
    float8 lon1 = PG_GETARG_FLOAT8(1);
    float8 lat2 = PG_GETARG_FLOAT8(2);
    float8 lon2 = PG_GETARG_FLOAT8(3);

    double  lat1_rad, lat2_rad;
    double  dlat_rad, dlon_rad;
    double  a, c, distance;

    /* ตรวจสอบขอบเขตของค่าละติจูด/ลองจิจูดที่ถูกต้องตามหลักภูมิศาสตร์ */
    if (lat1 < -90.0 || lat1 > 90.0 || lat2 < -90.0 || lat2 > 90.0)
        ereport(ERROR,
                (errcode(ERRCODE_INVALID_PARAMETER_VALUE),
                 errmsg("ค่า latitude ต้องอยู่ระหว่าง -90 ถึง 90 องศา"),
                 errdetail("ได้รับ lat1=%g, lat2=%g", lat1, lat2)));

    if (lon1 < -180.0 || lon1 > 180.0 || lon2 < -180.0 || lon2 > 180.0)
        ereport(ERROR,
                (errcode(ERRCODE_INVALID_PARAMETER_VALUE),
                 errmsg("ค่า longitude ต้องอยู่ระหว่าง -180 ถึง 180 องศา"),
                 errdetail("ได้รับ lon1=%g, lon2=%g", lon1, lon2)));

    lat1_rad = deg2rad(lat1);
    lat2_rad = deg2rad(lat2);
    dlat_rad = deg2rad(lat2 - lat1);
    dlon_rad = deg2rad(lon2 - lon1);

    /* สูตร Haversine มาตรฐาน */
    a = sin(dlat_rad / 2.0) * sin(dlat_rad / 2.0) +
        cos(lat1_rad) * cos(lat2_rad) *
        sin(dlon_rad / 2.0) * sin(dlon_rad / 2.0);

    c = 2.0 * atan2(sqrt(a), sqrt(1.0 - a));

    distance = EARTH_RADIUS_KM * c;

    PG_RETURN_FLOAT8(distance);
}

/*
 * thai_id_checksum_valid
 *      ตรวจสอบเลขบัตรประชาชนไทย 13 หลัก ด้วยสูตร checksum มาตรฐาน:
 *
 *      1. นำเลข 12 หลักแรกคูณด้วยตัวเลขน้ำหนัก 13, 12, 11, ..., 2 ตามลำดับ
 *      2. บวกผลคูณทั้งหมด แล้วหารด้วย 11 เอาเศษ (mod 11)
 *      3. นำ 11 ลบเศษที่ได้ แล้ว mod 10 อีกครั้ง จะได้ "เลขหลักที่ 13 ที่ถูกต้อง"
 *      4. เปรียบเทียบกับหลักที่ 13 จริงของ input ถ้าตรงกันคือถูกต้อง
 *
 *      ฟังก์ชันนี้ไม่ใช่ STRICT เพราะต้องการคืนค่า NULL แบบชัดเจน
 *      เมื่อ input เป็น NULL (แทนที่จะ error)
 *
 *      SQL: thai_id_checksum_valid(id text) RETURNS boolean
 */
PG_FUNCTION_INFO_V1(thai_id_checksum_valid);

Datum
thai_id_checksum_valid(PG_FUNCTION_ARGS)
{
    text   *id_text;
    char   *data;
    int     len;
    int     i;
    int     sum;
    int     expected_check_digit;
    int     actual_check_digit;

    /* จัดการ NULL input: คืนค่า NULL กลับไปตามธรรมเนียมของฟังก์ชันตรวจสอบข้อมูล */
    if (PG_ARGISNULL(0))
        PG_RETURN_NULL();

    id_text = PG_GETARG_TEXT_PP(0);
    data = VARDATA_ANY(id_text);
    len  = VARSIZE_ANY_EXHDR(id_text);

    /* เลขบัตรประชาชนไทยต้องมีความยาว 13 หลักพอดี */
    if (len != 13)
        PG_RETURN_BOOL(false);

    /* ตรวจสอบว่าทุกตัวอักษรเป็นตัวเลข 0-9 เท่านั้น */
    for (i = 0; i < 13; i++)
    {
        if (!isdigit((unsigned char) data[i]))
            PG_RETURN_BOOL(false);
    }

    /* คำนวณ checksum จาก 12 หลักแรก โดยใช้น้ำหนัก 13 ลงมาถึง 2 */
    sum = 0;
    for (i = 0; i < 12; i++)
    {
        int digit  = data[i] - '0';
        int weight = 13 - i;

        sum += digit * weight;
    }

    expected_check_digit = (11 - (sum % 11)) % 10;
    actual_check_digit   = data[12] - '0';

    PG_RETURN_BOOL(expected_check_digit == actual_check_digit);
}
```

### ไฟล์ 2: `thaibiz.control`

```ini
# thaibiz.control
comment = 'ฟังก์ชันคำนวณธุรกิจสำหรับตลาดไทย: ระยะทาง GPS (Haversine) และตรวจสอบเลขบัตรประชาชน'
default_version = '1.0'
module_pathname = '$libdir/thaibiz'
relocatable = true
```

### ไฟล์ 3: `thaibiz--1.0.sql`

```sql
-- thaibiz--1.0.sql

\echo Use "CREATE EXTENSION thaibiz" to load this file. \quit

--
-- haversine_km: คำนวณระยะทางระหว่างพิกัด GPS สองจุด หน่วยกิโลเมตร
--
CREATE FUNCTION haversine_km(
    lat1 float8,
    lon1 float8,
    lat2 float8,
    lon2 float8
) RETURNS float8
AS 'MODULE_PATHNAME', 'haversine_km'
LANGUAGE C IMMUTABLE STRICT PARALLEL SAFE;

COMMENT ON FUNCTION haversine_km(float8, float8, float8, float8) IS
    'คำนวณระยะทางแบบเส้นตรงบนผิวโลก (great-circle distance) ระหว่างสองพิกัด GPS ด้วยสูตร Haversine หน่วยกิโลเมตร';

--
-- thai_id_checksum_valid: ตรวจสอบความถูกต้องของเลขบัตรประชาชนไทย 13 หลัก
--
CREATE FUNCTION thai_id_checksum_valid(id text)
RETURNS boolean
AS 'MODULE_PATHNAME', 'thai_id_checksum_valid'
LANGUAGE C IMMUTABLE PARALLEL SAFE;
-- หมายเหตุ: ไม่ใส่ STRICT เพราะฟังก์ชันนี้ตรวจสอบ NULL เองภายใน
-- และตั้งใจคืนค่า NULL เมื่อ input เป็น NULL (ผ่าน PG_ARGISNULL)

COMMENT ON FUNCTION thai_id_checksum_valid(text) IS
    'ตรวจสอบเลขบัตรประชาชนไทย 13 หลักด้วยสูตร checksum มาตรฐาน คืนค่า true หากถูกต้อง, false หากไม่ถูกต้องหรือรูปแบบผิด, NULL หาก input เป็น NULL';
```

> **หมายเหตุเรื่อง `PARALLEL SAFE`**: เนื่องจากทั้งสองฟังก์ชันไม่ได้เข้าถึง shared state ใด ๆ ของ session (ไม่มี global variable, ไม่มีการเขียนตาราง) จึงปลอดภัยที่จะประกาศเป็น `PARALLEL SAFE` เพื่อให้ query planner สามารถใช้ parallel worker เรียกใช้ฟังก์ชันเหล่านี้พร้อมกันได้เมื่อ query มีการทำ parallel sequential scan

### ไฟล์ 4: `Makefile`

```makefile
# Makefile สำหรับ extension thaibiz

MODULES = thaibiz
EXTENSION = thaibiz
DATA = thaibiz--1.0.sql
SHLIB_LINK = -lm

PG_CONFIG = pg_config
PGXS := $(shell $(PG_CONFIG) --pgxs)
include $(PGXS)
```

สังเกตว่าเราเพิ่ม `SHLIB_LINK = -lm` เพราะ `haversine_km` เรียกใช้ฟังก์ชันจาก `<math.h>` (`sin`, `cos`, `atan2`, `sqrt`) ซึ่งบนบาง distro/linker ต้อง link กับ math library (`libm`) โดยชัดเจน

### ขั้นตอนการ compile, install และทดสอบทั้งหมด

```bash
# 1. สร้าง directory และไฟล์ทั้งหมดตามที่ระบุข้างต้น
mkdir -p ~/dev/thaibiz && cd ~/dev/thaibiz
# (สร้างไฟล์ thaibiz.c, thaibiz.control, thaibiz--1.0.sql, Makefile ตามเนื้อหาด้านบน)

# 2. compile
make

# ตรวจสอบว่าได้ thaibiz.so ออกมา
ls -la thaibiz.so

# 3. install (ต้องใช้สิทธิ์ที่เขียนเข้า PKGLIBDIR/SHAREDIR ได้)
sudo make install

# 4. เข้า psql แล้วเปิดใช้งาน extension
psql -d mydb
```

```sql
-- ภายใน psql
CREATE EXTENSION thaibiz;

-- ตรวจสอบว่า extension ถูกติดตั้ง พร้อมดูฟังก์ชันที่มีในนั้น
\dx thaibiz
\df thaibiz.*

-- ทดสอบ haversine_km: ระยะทางระหว่างกรุงเทพฯ (13.7563, 100.5018)
-- กับเชียงใหม่ (18.7883, 98.9853) — ระยะทางจริงประมาณ 580-590 กม.
SELECT haversine_km(13.7563, 100.5018, 18.7883, 98.9853) AS bangkok_to_chiangmai_km;
--  bangkok_to_chiangmai_km
-- -------------------------
--        584.67...
-- (1 row)

-- ทดสอบ input ที่ไม่ถูกต้อง (ค่า latitude เกินขอบเขต)
SELECT haversine_km(999, 100.5018, 18.7883, 98.9853);
-- ERROR:  ค่า latitude ต้องอยู่ระหว่าง -90 ถึง 90 องศา
-- DETAIL:  ได้รับ lat1=999, lat2=18.7883

-- ทดสอบ thai_id_checksum_valid ด้วยเลขบัตรประชาชนตัวอย่าง (สูตร checksum ถูกต้อง)
SELECT thai_id_checksum_valid('1101700210resolved') AS test_invalid_format;
-- false (เพราะมีตัวอักษรที่ไม่ใช่ตัวเลขปน)

SELECT thai_id_checksum_valid('1234567890121') AS example_check;
--  example_check
-- ---------------
--  f
-- (ตัวอย่างนี้เป็นเลขสุ่มเพื่อสาธิตรูปแบบ อาจไม่ผ่าน checksum จริง)

-- ทดสอบ NULL handling
SELECT thai_id_checksum_valid(NULL) AS test_null;
--  test_null
-- -----------
--
-- (1 row)  -- ได้ผลลัพธ์เป็น NULL ตามที่ออกแบบไว้

-- ทดสอบความยาวผิด
SELECT thai_id_checksum_valid('12345') AS test_wrong_length;
--  test_wrong_length
-- --------------------
--  f
-- (1 row)
```

### ตัวอย่างการใช้งานจริงในธุรกิจ: หาสาขาที่ใกล้ที่สุด

เมื่อ extension ติดตั้งแล้ว เราสามารถใช้ `haversine_km` ร่วมกับ query ปกติได้ทันที เช่น หาสาขาร้านค้าที่ใกล้กับตำแหน่งลูกค้าที่สุด:

```sql
CREATE TABLE store_branches (
    branch_id   serial PRIMARY KEY,
    branch_name text NOT NULL,
    latitude    float8 NOT NULL,
    longitude   float8 NOT NULL
);

INSERT INTO store_branches (branch_name, latitude, longitude) VALUES
    ('สาขาสยามพารากอน', 13.7460, 100.5340),
    ('สาขาเซ็นทรัลเวิลด์', 13.7466, 100.5393),
    ('สาขาเชียงใหม่', 18.7883, 98.9853);

-- หาสาขาที่ใกล้กับตำแหน่งลูกค้าที่สุด (สมมติลูกค้าอยู่ที่ 13.7500, 100.5350)
SELECT
    branch_name,
    round(haversine_km(13.7500, 100.5350, latitude, longitude)::numeric, 2) AS distance_km
FROM store_branches
ORDER BY haversine_km(13.7500, 100.5350, latitude, longitude)
LIMIT 3;

--        branch_name        | distance_km
-- ----------------------------+-------------
--  สาขาสยามพารากอน            |        0.53
--  สาขาเซ็นทรัลเวิลด์          |        0.68
--  สาขาเชียงใหม่               |      584.xx
-- (3 rows)
```

### การถอนการติดตั้ง (Uninstall)

```sql
-- ลบ extension ออกจาก database (ฟังก์ชันทั้งหมดที่สร้างโดย extension จะถูกลบไปด้วย)
DROP EXTENSION thaibiz;

-- ถ้ามี object อื่นอ้างอิงถึงฟังก์ชันในนี้ (เช่นใช้ใน view, index expression)
-- ต้องใช้ CASCADE หรือลบ object ที่อ้างอิงก่อน
DROP EXTENSION thaibiz CASCADE;
```

```bash
# ลบไฟล์ .so, .control, .sql ออกจากระบบไฟล์ (ถ้าต้องการล้างสมบูรณ์)
sudo make uninstall
```

extension `thaibiz` นี้แสดงให้เห็นครบทุกองค์ประกอบที่เรียนมาในบทนี้: การเขียนฟังก์ชัน C สองตัวที่มี signature ต่างกัน, การจัดการ argument หลายชนิด (float8, text), การจัดการ NULL ทั้งแบบ STRICT และแบบตรวจสอบเองด้วย `PG_ARGISNULL`, การใช้ `ereport(ERROR, ...)` เพื่อ validate input, การใช้ VARDATA/VARSIZE macro เพื่ออ่านข้อมูล text แบบ low-level, Makefile ที่ link กับ external library (`-lm`), และการเชื่อมโยงทุกอย่างเข้าด้วยกันผ่าน control file และ SQL definition file

---

## สรุปท้ายบท

ในบทนี้เราได้เรียนรู้การเขียน PostgreSQL extension ด้วยภาษา C ตั้งแต่พื้นฐานจนถึงตัวอย่างที่ใช้งานได้จริง:

- **เหตุผลในการเลือก C** เหนือ PL/pgSQL คือ performance สูงสุดและการเข้าถึง internal API แต่ต้องแลกกับความเสี่ยงที่สูงขึ้น (bug อาจทำให้ backend crash) และความซับซ้อนในการพัฒนา ควรเลือกใช้เมื่อวัดผลแล้วพบคอขวดจริง ไม่ใช่ optimize ล่วงหน้า
- **โครงสร้าง extension** ประกอบด้วย 3 ส่วนหลัก: control file (`.control`) สำหรับ metadata, SQL script (`.sql`) สำหรับนิยาม object, และ shared library (`.so`) สำหรับ compiled code
- **Development environment** ต้องมี PostgreSQL server headers (`postgresql-server-dev-XX` หรือ `postgresqlXX-devel`) และใช้ `pg_config` เพื่อตรวจสอบ path ต่าง ๆ
- **โครงสร้างฟังก์ชัน C พื้นฐาน** ต้องมี `PG_MODULE_MAGIC`, `PG_FUNCTION_INFO_V1(funcname)`, และ signature `Datum funcname(PG_FUNCTION_ARGS)` เสมอ
- **การจัดการ argument/return** ใช้มาโคร `PG_GETARG_*`/`PG_RETURN_*` ตามชนิดข้อมูล และต้องระวังเรื่อง NULL ด้วย `PG_ARGISNULL` หรือใช้ `STRICT` เพื่อให้ PostgreSQL จัดการ NULL ให้อัตโนมัติ
- **PGXS** เป็นระบบ build ที่ทำให้เขียน Makefile ได้ง่ายและ portable ข้ามแพลตฟอร์ม โดยแค่ `include $(PGXS)` เป็นบรรทัดสุดท้าย
- **Memory management** ต้องใช้ `palloc`/`pfree` แทน `malloc`/`free` เพราะทำงานร่วมกับ Memory Context และระบบ error handling ของ PostgreSQL ได้อย่างปลอดภัย
- **Error handling** ใช้ `ereport()` พร้อม error level ที่เหมาะสม (`NOTICE`, `WARNING`, `ERROR`) และให้ `errcode`/`errmsg`/`errdetail`/`errhint` เพื่อสื่อสารกับผู้ใช้อย่างชัดเจน
- **แบบฝึกหัดรวม** สาธิตการสร้าง extension `thaibiz` ที่มีฟังก์ชันคำนวณระยะทาง GPS ด้วยสูตร Haversine และฟังก์ชันตรวจสอบเลขบัตรประชาชนไทยด้วย checksum ซึ่งครอบคลุมทุกแนวคิดที่เรียนมาในบทนี้แบบครบวงจร

การเขียน extension ด้วย C เป็นทักษะระดับ expert ที่เปิดประตูสู่การพัฒนา PostgreSQL ในระดับลึกที่สุด ไม่ว่าจะเป็น custom data type, custom index access method, หรือแม้แต่การมีส่วนร่วมพัฒนา PostgreSQL core เอง ในบทถัดไปเราจะเจาะลึกกลไกภายในของ Query Planner เพื่อทำความเข้าใจว่า PostgreSQL ตัดสินใจเลือก execution plan อย่างไร ซึ่งเป็นความรู้พื้นฐานสำคัญสำหรับการ optimize query และการพัฒนา extension ที่โต้ตอบกับ planner (เช่น custom cost function, extended statistics)

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1

เพราะเหตุใด `Datum` จึงถูกออกแบบให้เป็น "กล่องสากล" (universal container) แทนที่จะให้แต่ละฟังก์ชัน C คืนค่าเป็น native C type ตรง ๆ (เช่น `int`, `char *`) เหมือนฟังก์ชัน C ทั่วไป?

<details>
<summary>เฉลย</summary>

เพราะ PostgreSQL ต้องมี calling convention ที่เป็นมาตรฐานเดียวกันสำหรับฟังก์ชันทุกตัว ไม่ว่าจะคืนค่าชนิดใด (`integer`, `text`, `numeric`, custom type ที่ผู้ใช้สร้างเอง ฯลฯ) function manager (`fmgr`) ของ PostgreSQL จึงสามารถเรียกฟังก์ชันทุกตัวผ่าน function pointer แบบเดียวกันได้ทั้งหมด (`Datum (*func)(PG_FUNCTION_ARGS)`) โดยไม่ต้องรู้ล่วงหน้าว่าฟังก์ชันนั้นคืนค่าชนิดอะไร ถ้าแต่ละฟังก์ชันคืนค่าเป็น native type ที่ต่างกัน (บาง function คืน `int`, บาง function คืน `struct` ที่ซับซ้อน) ระบบ dynamic dispatch แบบนี้จะทำไม่ได้เลยในภาษา C ซึ่งเป็น statically-typed language ที่ไม่มี polymorphism ในตัว `Datum` (ซึ่งมีขนาดคงที่เท่ากับ pointer) จึงเป็นวิธีแก้ปัญหานี้ — ค่าที่มีขนาดเล็กพอ (pass-by-value type) ถูกเก็บ bit pattern ตรงใน `Datum` เอง ส่วนค่าที่ใหญ่กว่า (pass-by-reference type) `Datum` จะเก็บเป็น pointer ชี้ไปยังหน่วยความจำจริง

</details>

### แบบฝึกหัดที่ 2

จงอธิบายว่าทำไม extension ที่ไม่มีไฟล์ `.so` เลย (มีแค่ `.control` และ `.sql`) ก็ยังถือว่าเป็น extension ที่สมบูรณ์ได้

<details>
<summary>เฉลย</summary>

เพราะ "extension" ในความหมายของ PostgreSQL คือกลไกการ packaging กลุ่มของ database object (function, type, operator, table ฯลฯ) ให้ install/upgrade/uninstall ได้เป็นหน่วยเดียวผ่านคำสั่ง `CREATE EXTENSION`/`ALTER EXTENSION`/`DROP EXTENSION` ไม่ได้จำกัดว่า object เหล่านั้นต้องมาจากโค้ด C เท่านั้น extension ที่มีแค่ SQL script ที่สร้างฟังก์ชัน PL/pgSQL, view, หรือตารางล้วน ๆ ก็เป็น extension ที่ใช้งานได้ปกติ (ตัวอย่างจริงคือ extension จำนวนมากใน PGXN ที่เป็นชุด utility function ที่เขียนด้วย PL/pgSQL ทั้งหมด) shared library `.so` จำเป็นเฉพาะเมื่อ extension นั้นมีฟังก์ชันหรือ object ที่ implement ด้วยภาษา C เท่านั้น

</details>

### แบบฝึกหัดที่ 3

ในไฟล์ Makefile แบบ PGXS จงอธิบายความแตกต่างระหว่างตัวแปร `MODULES` กับ `MODULE_big` ว่าใช้ในสถานการณ์ใดต่างกันอย่างไร

<details>
<summary>เฉลย</summary>

`MODULES` ใช้เมื่อแต่ละไฟล์ `.c` ใน extension compile แยกเป็น shared library `.so` คนละไฟล์ต่างหากกัน (เช่นมี `foo.c` และ `bar.c` ก็จะได้ `foo.so` และ `bar.so` แยกกัน) เหมาะกับ extension ที่มีฟังก์ชันอิสระจากกันหลายกลุ่ม ส่วน `MODULE_big` ใช้เมื่อต้องการรวมหลายไฟล์ `.c` เข้าเป็น shared library **เดียว** (เช่น `pgtutorial.c` และ `pgtutorial_helpers.c` compile แยกเป็น `.o` ก่อน แล้วรวมเป็น `pgtutorial.so` ไฟล์เดียว) ซึ่งต้องใช้คู่กับตัวแปร `OBJS` ที่ระบุรายชื่อ `.o` files ที่จะถูก link เข้าด้วยกัน กรณีนี้เหมาะกับ extension ขนาดใหญ่ที่แบ่งโค้ดเป็นหลายไฟล์เพื่อความเป็นระเบียบ แต่ต้องการให้ symbol ทั้งหมดอยู่ใน `.so` เดียวกัน (เช่นแชร์ static helper function ระหว่างไฟล์ได้)

</details>

### แบบฝึกหัดที่ 4

ฟังก์ชัน C ต่อไปนี้มีข้อผิดพลาดร้ายแรงอยู่ จงระบุปัญหาและอธิบายผลลัพธ์ที่อาจเกิดขึ้น

```c
PG_FUNCTION_INFO_V1(get_upper);

Datum
get_upper(PG_FUNCTION_ARGS)
{
    text *input = PG_GETARG_TEXT_PP(0);
    char *str = text_to_cstring(input);
    /* ... โค้ดแปลงเป็นตัวพิมพ์ใหญ่ ... */
    PG_RETURN_TEXT_P(cstring_to_text(str));
}
```

โดยที่ฟังก์ชันนี้ถูกประกาศใน SQL ว่า:

```sql
CREATE FUNCTION get_upper(text) RETURNS text
AS 'MODULE_PATHNAME', 'get_upper'
LANGUAGE C IMMUTABLE;  -- สังเกตว่าไม่มี STRICT
```

<details>
<summary>เฉลย</summary>

ปัญหาคือฟังก์ชันถูกประกาศเป็น **non-STRICT** (ไม่มีคำว่า `STRICT` ต่อท้าย) แต่โค้ดภายในฟังก์ชันกลับไม่ได้ตรวจสอบ `PG_ARGISNULL(0)` ก่อนเรียก `PG_GETARG_TEXT_PP(0)` เลย

ถ้ามีการเรียก `SELECT get_upper(NULL);` PostgreSQL จะเรียกฟังก์ชันนี้ตามปกติ (เพราะไม่ใช่ STRICT จึงไม่มีการเช็ค NULL อัตโนมัติให้) และ `PG_GETARG_TEXT_PP(0)` จะพยายามอ่านค่า argument ที่ไม่มีความหมาย (ไม่ใช่ pointer ที่ถูกต้อง) ไปยังหน่วยความจำ ทำให้เกิด **undefined behavior** ซึ่งในทางปฏิบัติมักแสดงออกมาเป็น **segmentation fault** ทำให้ backend process ทั้งตัว crash ทันที (ไม่ใช่แค่ query fail แบบ `ERROR` ธรรมดา) ผู้ใช้ทุกคนที่เชื่อมต่อผ่าน connection เดียวกันจะถูกตัดการเชื่อมต่อทันที

วิธีแก้: เพิ่ม `STRICT` ในการประกาศ SQL (ถ้าต้องการให้คืนค่า NULL เมื่อ input เป็น NULL แบบอัตโนมัติ) หรือถ้าต้องการ non-STRICT behavior จริง ๆ (เช่นต้องการ custom logic เมื่อเจอ NULL) ต้องเพิ่มการตรวจสอบ `if (PG_ARGISNULL(0)) PG_RETURN_NULL();` เป็นบรรทัดแรกสุดของฟังก์ชันก่อนเรียก `PG_GETARG_TEXT_PP` ใด ๆ

</details>

### แบบฝึกหัดที่ 5

เพราะเหตุใดการใช้ `malloc()`/`free()` มาตรฐานของภาษา C จึงเป็นแนวทางที่ไม่ปลอดภัยในฟังก์ชัน PostgreSQL extension โดยเฉพาะเมื่อฟังก์ชันนั้นอาจเรียก `ereport(ERROR, ...)`?

<details>
<summary>เฉลย</summary>

เพราะเมื่อ `ereport(ERROR, ...)` ถูกเรียก PostgreSQL จะใช้กลไก `longjmp` เพื่อ unwind stack กลับไปยัง error-handling point ระดับบนสุด (ปกติคือระดับ transaction/query) ทันที การ unwind แบบนี้จะ**ข้ามโค้ดที่เหลือในฟังก์ชันไปทั้งหมด** รวมถึงบรรทัดที่ควรจะเรียก `free()` เพื่อคืนหน่วยความจำที่ `malloc()` ไว้ก่อนหน้า ถ้าใช้ `malloc()` แล้วเกิด error ก่อนถึงบรรทัด `free()` หน่วยความจำนั้นจะ**รั่วไหลถาวร**ไปตลอดอายุของ backend process นั้น (เพราะไม่มีกลไกอื่นมา track และ cleanup ให้)

ในทางกลับกัน หน่วยความจำที่ `palloc()` ไว้จะถูกผูกกับ Memory Context ที่ active อยู่ ณ ขณะนั้น (เช่น per-query context) และเมื่อเกิด error หรือ query จบการทำงานไม่ว่าจะด้วยเหตุผลใด PostgreSQL จะล้าง (reset/delete) memory context นั้นทั้งก้อนโดยอัตโนมัติ ทำให้หน่วยความจำทั้งหมดที่ palloc ไว้ในนั้นถูกคืนกลับให้ระบบเสมอ ไม่ว่าโค้ดจะ error กลางทางหรือไม่ก็ตาม — นี่คือเหตุผลที่กฎเหล็กของการเขียน PostgreSQL C extension คือ "ใช้ palloc/pfree เสมอ ห้ามใช้ malloc/free"

</details>

### แบบฝึกหัดที่ 6

จงเขียนฟังก์ชัน C ชื่อ `clamp_int(value integer, min_val integer, max_val integer)` ที่คืนค่า `value` ถ้าอยู่ในช่วง `[min_val, max_val]` แต่ถ้าน้อยกว่า `min_val` ให้คืน `min_val` และถ้ามากกว่า `max_val` ให้คืน `max_val` (ฟังก์ชันนี้ไม่ต้องรองรับ NULL เป็นพิเศษ ใช้ STRICT ได้เลย)

<details>
<summary>เฉลย</summary>

```c
#include "postgres.h"
#include "fmgr.h"

PG_MODULE_MAGIC;

PG_FUNCTION_INFO_V1(clamp_int);

Datum
clamp_int(PG_FUNCTION_ARGS)
{
    int32 value   = PG_GETARG_INT32(0);
    int32 min_val = PG_GETARG_INT32(1);
    int32 max_val = PG_GETARG_INT32(2);
    int32 result;

    if (min_val > max_val)
        ereport(ERROR,
                (errcode(ERRCODE_INVALID_PARAMETER_VALUE),
                 errmsg("min_val (%d) ต้องไม่มากกว่า max_val (%d)",
                        min_val, max_val)));

    if (value < min_val)
        result = min_val;
    else if (value > max_val)
        result = max_val;
    else
        result = value;

    PG_RETURN_INT32(result);
}
```

และ SQL definition:

```sql
CREATE FUNCTION clamp_int(value integer, min_val integer, max_val integer)
RETURNS integer
AS 'MODULE_PATHNAME', 'clamp_int'
LANGUAGE C IMMUTABLE STRICT PARALLEL SAFE;
```

เนื่องจากฟังก์ชันนี้ไม่ต้องการ custom behavior เมื่อเจอ NULL (การคืนค่า NULL โดยอัตโนมัติเมื่อ argument ใดเป็น NULL ถือเป็นพฤติกรรมที่สมเหตุสมผลอยู่แล้วสำหรับฟังก์ชันประเภทนี้) จึงประกาศเป็น `STRICT` ได้เลยโดยไม่ต้องเขียนโค้ดตรวจสอบ `PG_ARGISNULL` เอง

</details>

### แบบฝึกหัดที่ 7

Error level ใดใน PostgreSQL ที่จะทำให้ **connection ของ client ถูกตัดทันที** แต่ **server หลักยังทำงานต่อได้ปกติ** (ไม่กระทบ session อื่น)? และ error level ใดที่ทำให้ query ปัจจุบัน fail แต่ connection ยังใช้งานต่อได้?

<details>
<summary>เฉลย</summary>

- **`FATAL`** — ทำให้ connection/session ปัจจุบันถูกตัดทันที (client ต้องเชื่อมต่อใหม่) แต่ server process หลักและ session อื่น ๆ ยังทำงานต่อได้ตามปกติ ไม่กระทบกัน
- **`ERROR`** — ทำให้เฉพาะ statement/transaction ปัจจุบัน fail และถูก rollback แต่ connection เดิมยังคงเปิดอยู่ พร้อมรับคำสั่ง SQL ถัดไปได้ทันที (ถ้าไม่ได้อยู่ใน explicit transaction ที่ค้างอยู่ในสถานะ aborted)

ส่วน `PANIC` จะร้ายแรงกว่า `FATAL` อีกขั้น คือทำให้ **PostgreSQL server ทั้งคลัสเตอร์ต้อง restart** (กระทบทุก session ไม่ใช่แค่ session เดียว) ซึ่งไม่ควรใช้ใน extension ทั่วไปเลย เว้นแต่กรณีตรวจพบข้อมูล internal corrupt ที่ร้ายแรงจริง ๆ ซึ่งเป็นสถานการณ์ที่แทบไม่เกิดขึ้นในโค้ด extension ระดับ application

</details>

### แบบฝึกหัดที่ 8

ในไฟล์ SQL definition (`--1.0.sql`) ของ extension เราเห็นบรรทัด `\echo Use "CREATE EXTENSION xxx" to load this file. \quit` อยู่ตอนต้นไฟล์เสมอ บรรทัดนี้ทำหน้าที่อะไร และจะเกิดอะไรขึ้นถ้าลบบรรทัดนี้ออก?

<details>
<summary>เฉลย</summary>

บรรทัดนี้เป็นกลไกป้องกันไม่ให้ผู้ใช้รัน SQL script ของ extension โดยตรงผ่าน `psql -f xxx--1.0.sql` หรือคำสั่ง `\i` ใน psql เพราะการรันแบบนั้นจะสร้าง object ทั้งหมดในไฟล์ (function, type ฯลฯ) เข้าไปใน database โดยที่ **PostgreSQL ไม่รู้ว่า object เหล่านี้เป็นส่วนหนึ่งของ extension** — จะไม่มีการบันทึกใน `pg_extension` catalog ทำให้ `DROP EXTENSION`, `ALTER EXTENSION UPDATE`, และการ track dependency ทั้งหมดใช้งานไม่ได้ (object เหล่านั้นจะกลายเป็น "loose" object ที่ต้องลบทีละตัวด้วยมือ)

เมื่อไฟล์นี้ถูก execute ผ่าน `\i` หรือ `psql -f` โดยตรง คำสั่ง `\echo` จะพิมพ์ข้อความแจ้งเตือน แล้วคำสั่ง `\quit` จะสั่งให้ psql ออกจากโปรแกรมทันที ทำให้ statement ที่เหลือในไฟล์ (เช่น `CREATE FUNCTION`) ไม่ถูก execute เลย — เป็นการ "บล็อก" การใช้งานผิดวิธีอย่างสุภาพ

ถ้าลบบรรทัดนี้ออก ไฟล์ยังคงทำงานถูกต้องเหมือนเดิมเมื่อถูกเรียกผ่าน `CREATE EXTENSION` ตามปกติ (เพราะกลไก `CREATE EXTENSION` internal ของ PostgreSQL ไม่ได้พึ่งพาบรรทัดนี้เลย) แต่จะสูญเสียการป้องกันผู้ใช้ที่พยายามรันไฟล์ตรง ๆ โดยไม่ตั้งใจ ซึ่งเป็นแนวทางปฏิบัติที่ดี (best practice) ที่ extension มาตรฐานเกือบทั้งหมดใน PostgreSQL core และ PGXN ใช้กัน แม้จะไม่ใช่ requirement บังคับก็ตาม

</details>

### แบบฝึกหัดที่ 9

พิจารณาฟังก์ชัน `thai_id_checksum_valid` ในแบบฝึกหัดรวม (Step 850) ทำไมจึงเลือกใช้ `PG_GETARG_TEXT_PP` แทน `PG_GETARG_TEXT_P` และทำไมจึงเลือกใช้ `VARDATA_ANY`/`VARSIZE_ANY_EXHDR` แทน `VARDATA`/`VARSIZE` ธรรมดา?

<details>
<summary>เฉลย</summary>

`PG_GETARG_TEXT_PP` (packed) เป็นเวอร์ชันที่ทันสมัยและมีประสิทธิภาพดีกว่า `PG_GETARG_TEXT_P` เพราะมันสามารถจัดการกับข้อมูลที่ยังอยู่ในรูปแบบ**บีบอัด (compressed)** ได้โดยตรงในบางกรณี (ไม่ต้อง detoast แบบเต็มรูปแบบเสมอไป) และหลีกเลี่ยงการ copy ข้อมูลที่ไม่จำเป็นเมื่อข้อมูลนั้นไม่ได้ถูก TOAST อยู่แล้ว (in-line, ไม่ compressed) ในขณะที่ `PG_GETARG_TEXT_P` แบบเก่าจะ detoast และ copy ข้อมูลเสมอไม่ว่าจำเป็นหรือไม่ ทำให้สิ้นเปลืองทรัพยากรโดยไม่จำเป็นสำหรับ text สั้น ๆ ทั่วไป (เช่น เลขบัตรประชาชน 13 ตัวอักษร ซึ่งไม่มีทาง TOAST อยู่แล้วเพราะสั้นกว่า threshold มาก)

เนื่องจากค่าที่ได้จาก `PG_GETARG_TEXT_PP` อาจอยู่ในรูปแบบพิเศษ (short varlena header หรือ compressed) มาโคร `VARDATA`/`VARSIZE` แบบดั้งเดิม (ที่สมมติว่าข้อมูลเป็น "uncompressed 4-byte header" เสมอ) จึงใช้ไม่ได้อย่างปลอดภัยกับค่าที่ได้จาก `_PP` ต้องใช้คู่มาโคร `VARDATA_ANY`/`VARSIZE_ANY_EXHDR` แทน ซึ่งถูกออกแบบมาให้รองรับได้ทุกรูปแบบของ varlena header โดยอัตโนมัติ (คำว่า "ANY" หมายถึงรองรับได้ทุก representation) และ `VARSIZE_ANY_EXHDR` จะคืนค่าความยาวข้อมูลจริงโดยไม่รวม header ให้เลย (EXHDR = excluding header) ซึ่งตรงกับความต้องการในการตรวจสอบว่า input มีความยาว 13 ตัวอักษรพอดีหรือไม่

</details>

### แบบฝึกหัดที่ 10

จงอธิบายเหตุผลที่ควรประกาศ `haversine_km` เป็น `IMMUTABLE` และประกาศ `thai_id_checksum_valid` เป็น `IMMUTABLE` เช่นกัน (ไม่ใช่ `STABLE` หรือ `VOLATILE`) แล้วยกตัวอย่างฟังก์ชันประเภทที่ **ไม่ควร** ประกาศเป็น `IMMUTABLE`

<details>
<summary>เฉลย</summary>

ทั้งสองฟังก์ชันควรประกาศเป็น `IMMUTABLE` เพราะผลลัพธ์ของฟังก์ชันขึ้นอยู่กับค่า argument ที่ส่งเข้ามาเท่านั้น ไม่ได้พึ่งพาข้อมูลใด ๆ ในฐานข้อมูล (ไม่อ่านตาราง ไม่ดู session variable ไม่ดูเวลาปัจจุบัน) และไม่มี side effect ใด ๆ — เมื่อ input เดิม จะได้ output เดิมเสมอทุกครั้งไม่ว่าจะเรียกกี่ครั้งก็ตาม, กี่ transaction ก็ตาม

การประกาศ `IMMUTABLE` ให้ประโยชน์ทาง performance สำคัญกับ query planner:

1. **Constant folding**: ถ้าเรียกฟังก์ชันด้วยค่าคงที่ (เช่น `WHERE haversine_km(13.7, 100.5, lat, lon) < 5`) planner ไม่สามารถ fold ส่วนที่เป็น constant ล่วงหน้าได้ในกรณีนี้เพราะ `lat`/`lon` เป็น column แต่ถ้าทุก argument เป็นค่าคงที่ทั้งหมด (`SELECT haversine_km(13.0, 100.0, 14.0, 101.0)`) planner จะคำนวณผลลัพธ์แค่ครั้งเดียวตอน planning แทนที่จะคำนวณซ้ำทุก row
2. **Index บน expression**: สามารถสร้าง functional index บนฟังก์ชันนี้ได้ (เช่น `CREATE INDEX ON table (thai_id_checksum_valid(id_column))`) เฉพาะฟังก์ชันที่เป็น `IMMUTABLE` เท่านั้นที่ PostgreSQL อนุญาตให้ใช้ใน index expression ได้ เพราะดัชนีต้องมีผลลัพธ์ที่ deterministic ตลอดไป
3. **Query result caching และ materialized view**: ระบบ optimize ต่าง ๆ ที่พึ่งพาความ deterministic ของฟังก์ชันสามารถทำงานได้อย่างถูกต้อง

ตัวอย่างฟังก์ชันที่ **ไม่ควร** ประกาศเป็น `IMMUTABLE`:

- ฟังก์ชันที่อ่านค่าจากตารางในฐานข้อมูล เช่น ฟังก์ชันที่ lookup ราคาสินค้าจากตาราง `products` — ควรเป็น `STABLE` (ผลลัพธ์คงที่ภายใน transaction เดียวกัน แต่เปลี่ยนได้ข้าม transaction ถ้าข้อมูลในตารางถูกแก้)
- ฟังก์ชันอย่าง `now()`, `random()`, `nextval()` ที่คืนค่าต่างกันทุกครั้งที่เรียกแม้ input เดิม (หรือไม่มี input เลย) — ต้องเป็น `VOLATILE` (ค่า default) เพราะถ้าประกาศผิดเป็น `IMMUTABLE` planner อาจ cache ผลลัพธ์แล้วนำไปใช้ซ้ำอย่างผิดพลาด ทำให้ได้ผลลัพธ์ที่ไม่ถูกต้องทาง logic (เช่น แทนที่จะสุ่มเลขใหม่ทุก row กลับได้ค่าเดิมซ้ำทุก row)

</details>

---

**บทถัดไป**: [Part 086 — Query Planner Internals](./part-086-query-planner-internals.md)
