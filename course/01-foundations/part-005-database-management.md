# Part 005: การสร้างและจัดการฐานข้อมูล (Database & Schema Management)

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับพื้นฐาน | Part 005

---

## เป้าหมายการเรียนรู้

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

- สร้างฐานข้อมูล (database) ด้วย `CREATE DATABASE` พร้อมกำหนด options ที่สำคัญ เช่น `OWNER`, `TEMPLATE`, `ENCODING`, `LOCALE`, `TABLESPACE` ได้อย่างถูกต้อง
- เข้าใจเรื่อง Encoding และ Locale ว่าทำไมต้องเลือกให้ถูกตั้งแต่แรก และรู้จักปัญหา collation ที่พบบ่อย
- เข้าใจความแตกต่างระหว่าง `template0` และ `template1` และสร้าง custom template database ของตัวเองได้
- ใช้ `ALTER DATABASE` เพื่อเปลี่ยนชื่อ เปลี่ยนเจ้าของ ตั้งค่า connection limit และตั้งค่า parameter เฉพาะฐานข้อมูล
- ลบฐานข้อมูลด้วย `DROP DATABASE` อย่างปลอดภัย รวมถึงเข้าใจการใช้ `WITH (FORCE)` ใน PostgreSQL 13+
- สร้าง จัดการ และลบ Schema ด้วย `CREATE SCHEMA`, `ALTER SCHEMA`, `DROP SCHEMA`
- ใช้คำสั่ง psql meta-command และ SQL query เพื่อดูรายการ databases/schemas พร้อมขนาดของมัน
- เข้าใจสิทธิ์พื้นฐานในการสร้าง database/schema ระหว่าง superuser กับ role ที่มี `CREATEDB` attribute
- เข้าใจแนวคิดของ Tablespace เบื้องต้น และสามารถสร้าง/ย้ายข้อมูลไป tablespace อื่นได้
- นำ best practices เรื่องการตั้งชื่อและการวางแผนโครงสร้างฐานข้อมูลไปใช้กับโปรเจกต์จริงที่มีหลายทีม หลาย environment

---

## ก่อนเริ่ม: บริบทของ Part นี้

ใน Part ที่ผ่านมาเราได้ติดตั้ง PostgreSQL และเชื่อมต่อกับ server ผ่าน `psql` กันไปแล้ว ใน Part นี้เราจะเริ่มทำงานกับ "ฐานข้อมูล" (database) และ "schema" ซึ่งเป็นหน่วยจัดระเบียบระดับบนสุดของ PostgreSQL

โครงสร้างการจัดระเบียบของ PostgreSQL เรียงจากใหญ่ไปเล็กมีลักษณะดังนี้

```
PostgreSQL Server (instance / cluster)
 └── Database (เช่น shop_db, hr_db)
      └── Schema (เช่น public, sales, audit)
           └── Objects (table, view, function, sequence, index, ...)
```

ข้อควรรู้ที่สำคัญคือ **1 การเชื่อมต่อ (connection) จะอยู่ใน 1 database เท่านั้น** เราไม่สามารถ query ข้ามฐานข้อมูลได้โดยตรงด้วย SQL ธรรมดา (ต้องใช้ `dblink` หรือ `postgres_fdw` หรือ foreign data wrapper อื่น ๆ ซึ่งจะพูดถึงในบทที่สูงขึ้น) ในทางกลับกัน **schema อยู่ภายใน database เดียวกัน และ query ข้าม schema ได้ตามปกติ** เพียงระบุชื่อเต็มแบบ `schema_name.table_name`

ทำความเข้าใจจุดนี้ให้แม่นเพราะจะมีผลต่อการออกแบบสถาปัตยกรรมในทุกระดับของหลักสูตรนี้

---

## Step 41: CREATE DATABASE — syntax เต็มพร้อม options

### Syntax พื้นฐาน

รูปแบบคำสั่งแบบง่ายที่สุดคือ

```sql
CREATE DATABASE dbname;
```

ตัวอย่างเช่น

```sql
CREATE DATABASE shop_db;
```

ผลลัพธ์ที่ psql แสดง:

```
CREATE DATABASE
```

### Syntax เต็มพร้อม options ทั้งหมด

```sql
CREATE DATABASE dbname
    [ WITH ]
    [ OWNER [=] user_name ]
    [ TEMPLATE [=] template ]
    [ ENCODING [=] encoding ]
    [ LOCALE [=] locale ]
    [ LC_COLLATE [=] lc_collate ]
    [ LC_CTYPE [=] lc_ctype ]
    [ ICU_LOCALE [=] icu_locale ]
    [ LOCALE_PROVIDER [=] locale_provider ]
    [ TABLESPACE [=] tablespace_name ]
    [ ALLOW_CONNECTIONS [=] allowconn ]
    [ CONNECTION LIMIT [=] connlimit ]
    [ IS_TEMPLATE [=] istemplate ]
    [ STRATEGY [=] strategy ];
```

มาดูความหมายของแต่ละ option ทีละตัว

| Option | ความหมาย | ค่า default |
|---|---|---|
| `OWNER` | ผู้เป็นเจ้าของฐานข้อมูล (role) | role ที่รันคำสั่ง |
| `TEMPLATE` | ฐานข้อมูลต้นแบบที่ใช้ copy โครงสร้าง | `template1` |
| `ENCODING` | character encoding เช่น `UTF8`, `LATIN1` | ตาม template |
| `LOCALE` | กำหนดทั้ง `LC_COLLATE` และ `LC_CTYPE` พร้อมกัน | ตาม template |
| `LC_COLLATE` | กฎการเรียงลำดับ (sort order) ของ string | ตาม template |
| `LC_CTYPE` | กฎการจำแนกชนิดตัวอักษร (upper/lower, ตัวเลข ฯลฯ) | ตาม template |
| `TABLESPACE` | พื้นที่จัดเก็บไฟล์ของฐานข้อมูลนี้ | tablespace ของ template |
| `CONNECTION LIMIT` | จำนวน connection สูงสุดที่อนุญาต (-1 = ไม่จำกัด) | -1 |
| `IS_TEMPLATE` | ทำให้ฐานข้อมูลนี้เป็น template ได้ด้วยหรือไม่ | false |

### ตัวอย่างการใช้งานจริง

**ตัวอย่างที่ 1: สร้างฐานข้อมูลพร้อมกำหนด owner**

```sql
-- สร้าง role ก่อน (ถ้ายังไม่มี)
CREATE ROLE app_shop WITH LOGIN PASSWORD 'S3cur3P@ss';

-- สร้าง database โดยกำหนดให้ app_shop เป็นเจ้าของ
CREATE DATABASE shop_db
    OWNER app_shop
    ENCODING 'UTF8'
    TEMPLATE template0
    LC_COLLATE 'en_US.UTF-8'
    LC_CTYPE 'en_US.UTF-8'
    CONNECTION LIMIT 100;
```

```
CREATE ROLE
CREATE DATABASE
```

**ตัวอย่างที่ 2: สร้างฐานข้อมูลแบบระบุ tablespace**

```sql
CREATE DATABASE analytics_db
    OWNER analyst_team
    TABLESPACE fast_ssd_ts
    ENCODING 'UTF8';
```

> หมายเหตุ: การใช้ `TABLESPACE` จะพูดถึงรายละเอียดเพิ่มเติมใน Step 49

**ตัวอย่างที่ 3: สร้างฐานข้อมูลด้วยคำสั่ง shell (ไม่ต้องเข้า psql ก่อน)**

นอกจาก SQL แล้ว PostgreSQL ยังมีคำสั่งระดับ command line ชื่อ `createdb` ที่เป็น wrapper ของ `CREATE DATABASE`

```bash
createdb -O app_shop -E UTF8 shop_db
```

คำสั่งนี้เทียบเท่ากับการรัน `CREATE DATABASE shop_db OWNER app_shop ENCODING 'UTF8';` ผ่าน `psql`

### กฎเกี่ยวกับ IF NOT EXISTS

ข้อควรรู้: `CREATE DATABASE` **ไม่รองรับ** `IF NOT EXISTS` (ต่างจาก `CREATE TABLE` หรือ `CREATE SCHEMA`) เหตุผลคือ `CREATE DATABASE` ทำงานนอก transaction block (ไม่สามารถ rollback ได้) การเช็คว่ามีอยู่แล้วหรือไม่จึงมักทำผ่าน script ภายนอก เช่น

```bash
psql -tc "SELECT 1 FROM pg_database WHERE datname = 'shop_db'" | grep -q 1 || createdb shop_db
```

หรือใน SQL script ที่ซับซ้อนขึ้นอาจใช้ `DO` block ร่วมกับ `dblink` แต่โดยทั่วไปแนะนำให้จัดการผ่าน deployment/migration tool แทน

### ข้อจำกัดสำคัญของ CREATE DATABASE

1. **ต้องไม่มี transaction block ครอบ** — รันคำสั่งนี้เดี่ยว ๆ ไม่สามารถใส่ใน `BEGIN...COMMIT` ได้
2. **ต้องมีสิทธิ์ `CREATEDB`** — เฉพาะ superuser หรือ role ที่มี attribute `CREATEDB` เท่านั้นที่ทำได้ (ดูรายละเอียดใน Step 48)
3. **ฐานข้อมูลใหม่จะถูก copy จาก template** ไม่ได้สร้างจากศูนย์ (ดูรายละเอียดใน Step 43)

---

## Step 42: Encoding และ Locale — UTF8 คืออะไร ทำไมสำคัญ

### Character Encoding คืออะไร

Character encoding คือกฎที่ใช้แปลง "ตัวอักษร" ให้เป็น "bytes" ที่คอมพิวเตอร์เก็บได้ PostgreSQL รองรับ encoding หลายแบบ แต่ตัวที่แนะนำและใช้เป็นมาตรฐานในโลกปัจจุบันคือ **UTF8** (Unicode Transformation Format 8-bit)

ทำไมต้อง UTF8:

- รองรับตัวอักษรแทบทุกภาษาในโลกในระบบเดียว รวมถึงภาษาไทย, จีน, ญี่ปุ่น, อาหรับ, อิโมจิ ฯลฯ
- เป็นมาตรฐานสากลที่ใช้กันแพร่หลายที่สุด เข้ากันได้กับ framework และ library สมัยใหม่แทบทั้งหมด
- ป้องกันปัญหา "ตัวอักษรเพี้ยน" (mojibake) เมื่อข้อมูลเดินทางระหว่างระบบต่าง encoding

ตรวจสอบ encoding ของฐานข้อมูลปัจจุบัน:

```sql
SHOW SERVER_ENCODING;
SHOW CLIENT_ENCODING;
```

```
 server_encoding
------------------
 UTF8
(1 row)

 client_encoding
------------------
 UTF8
(1 row)
```

หรือดูค่า encoding ของทุกฐานข้อมูลในระบบ:

```sql
SELECT datname, pg_encoding_to_char(encoding) AS encoding
FROM pg_database
ORDER BY datname;
```

```
   datname    | encoding
--------------+----------
 postgres     | UTF8
 shop_db      | UTF8
 template0    | UTF8
 template1    | UTF8
(4 rows)
```

> **ข้อควรจำ**: encoding ของฐานข้อมูลกำหนดตอนสร้างเท่านั้น **เปลี่ยนภายหลังไม่ได้** (ต้อง dump ข้อมูลแล้ว restore ลงฐานข้อมูลใหม่ที่มี encoding ที่ถูกต้อง) ดังนั้นต้องเลือกให้ถูกตั้งแต่ตอน `CREATE DATABASE`

### Locale คืออะไร

Locale คือชุดกฎที่กำหนดพฤติกรรมเกี่ยวกับภาษาและวัฒนธรรม เช่น

- **LC_COLLATE** — กฎการเรียงลำดับ string (sort order) เช่น `ORDER BY name` จะเรียง ก-ฮ หรือ A-Z อย่างไร
- **LC_CTYPE** — กฎการจำแนกตัวอักษร เช่น อะไรคือตัวพิมพ์ใหญ่-เล็ก, อะไรคือตัวเลข, ใช้กับฟังก์ชันอย่าง `upper()`, `lower()`, regex `\w`

รูปแบบ locale ทั่วไปคือ `language_TERRITORY.encoding` เช่น

- `en_US.UTF-8` — ภาษาอังกฤษ สหรัฐอเมริกา
- `th_TH.UTF-8` — ภาษาไทย ประเทศไทย
- `C` หรือ `POSIX` — locale แบบ "byte order" ล้วน ๆ ไม่สนใจกฎภาษาใด ๆ เรียงตามค่า byte

ตรวจสอบ locale list ที่ OS รองรับ:

```bash
locale -a
```

```
C
C.UTF-8
en_US.utf8
th_TH.utf8
POSIX
```

### ปัญหา collation ที่พบบ่อย

**ปัญหาที่ 1: การเรียงลำดับข้อมูลไม่ตรงกับที่คาดหวัง**

```sql
CREATE TABLE t_demo (name text);
INSERT INTO t_demo VALUES ('apple'), ('Banana'), ('cherry'), ('Apple');

SELECT name FROM t_demo ORDER BY name;
```

ถ้าใช้ locale `C` ผลลัพธ์จะเรียงตามค่า byte (ตัวพิมพ์ใหญ่มาก่อนตัวพิมพ์เล็กเสมอ):

```
 name
--------
 Apple
 Banana
 apple
 cherry
```

แต่ถ้าใช้ locale `en_US.UTF-8` ผลลัพธ์จะเรียงแบบ "natural" มากกว่า:

```
  name
--------
 apple
 Apple
 Banana
 cherry
```

**ปัญหาที่ 2: Performance ของ index ช้าลง**

Locale ที่ไม่ใช่ `C` จะทำให้การเปรียบเทียบ string ช้ากว่า `C` locale พอสมควร เพราะต้องคำนวณกฎภาษาซับซ้อนกว่าการเทียบ byte ตรง ๆ สำหรับระบบที่เน้น performance สูงมาก บางทีมเลือกใช้ locale `C` แล้วจัดการเรื่องการเรียงลำดับที่ application layer แทน หรือใช้ `COLLATE` ระบุเฉพาะ column ที่จำเป็น

**ปัญหาที่ 3: ย้ายฐานข้อมูลข้าม OS แล้ว index เพี้ยน**

นี่คือปัญหาที่อันตรายที่สุด — ถ้า `glibc` เวอร์ชันบน server ต้นทางกับปลายทางต่างกัน กฎ collation ของ locale เดียวกัน (เช่น `en_US.UTF-8`) อาจตีความไม่เหมือนกันเป๊ะ ทำให้ **index ที่สร้างจาก collation นั้นอาจให้ผลลัพธ์ผิดพลาด** (query ที่ควรเจอแถวข้อมูลกลับไม่เจอ) วิธีแก้คือ

1. รัน `REINDEX` ทุก index ที่เกี่ยวกับ collation หลังจากย้าย server หรืออัปเกรด OS/glibc
2. ใช้ **ICU collation** แทน libc collation เพราะ ICU มีเวอร์ชันควบคุมชัดเจนกว่าและไม่ผูกกับ OS
3. ตรวจสอบด้วย `pg_collation` และ column `collversion`

```sql
SELECT collname, collprovider, collversion
FROM pg_collation
WHERE collname IN ('en_US.UTF-8', 'th_TH.UTF-8');
```

**ตัวอย่างการใช้ ICU locale (แนะนำสำหรับระบบใหม่ตั้งแต่ PostgreSQL 15+)**

```sql
CREATE DATABASE shop_db_icu
    TEMPLATE template0
    ENCODING 'UTF8'
    LOCALE_PROVIDER icu
    ICU_LOCALE 'en-US'
    LC_COLLATE 'C'
    LC_CTYPE 'C';
```

การใช้ ICU locale ทำให้ collation behavior คงที่ไม่ขึ้นกับ glibc ของ OS และยังรองรับ collation แบบละเอียด เช่น case-insensitive หรือ numeric sort ผ่าน ICU custom rules ได้ด้วย

### แนวทางเลือก Encoding/Locale สำหรับโปรเจกต์ใหม่

| สถานการณ์ | แนะนำ |
|---|---|
| ระบบทั่วไป รองรับหลายภาษารวมภาษาไทย | `ENCODING UTF8`, `LC_COLLATE/LC_CTYPE en_US.UTF-8` หรือ ICU `en-US` |
| ต้องการ performance สูงสุด ไม่สนใจการเรียงแบบภาษา | `ENCODING UTF8`, locale `C` แล้วใช้ `COLLATE "th-TH-x-icu"` เฉพาะ column ที่ต้องเรียงแบบภาษา |
| ระบบที่ย้าย server บ่อย ข้าม OS | ใช้ ICU provider แทน libc |

---

## Step 43: Template Databases (template0 vs template1)

### ทำไมต้องมี Template

ทุกครั้งที่รัน `CREATE DATABASE` PostgreSQL **ไม่ได้สร้างฐานข้อมูลเปล่าจากศูนย์** แต่จะ "copy" ฐานข้อมูลต้นแบบ (template) มาทั้งหมด รวมถึง system catalog, encoding, locale และ object ใด ๆ ที่มีอยู่ใน template นั้น

โดย default หากไม่ระบุ `TEMPLATE` จะใช้ `template1` เสมอ

### template0 คืออะไร

`template0` คือฐานข้อมูล "ต้นแบบดั้งเดิม" ที่ PostgreSQL สร้างไว้ตอน initdb เก็บเฉพาะ system object มาตรฐาน **ไม่มี** object ใด ๆ ที่ผู้ใช้เพิ่มเข้ามาภายหลังเลย และถูกตั้งค่าให้ **ห้าม connect เข้าไปแก้ไขโดยตรง** (`datallowconn = false`)

จุดประสงค์ของ `template0`:

1. ใช้เป็นฐานสำหรับสร้างฐานข้อมูลใหม่ที่ต้องการ **encoding/locale ต่างจาก template1**
2. ใช้เป็นฐานสำหรับ `pg_restore` แบบ full restore (เพราะรับประกันว่าไม่มี object แปลกปลอมติดมา)

### template1 คืออะไร

`template1` คือฐานข้อมูลต้นแบบที่ **ใช้เป็นค่า default** เมื่อสร้างฐานข้อมูลใหม่โดยไม่ระบุ `TEMPLATE` จุดสำคัญคือ **ผู้ดูแลระบบสามารถแก้ไข `template1` ได้** เช่น ติดตั้ง extension หรือสร้าง function/role มาตรฐานที่อยากให้ทุกฐานข้อมูลใหม่มีติดมาด้วย

```sql
-- ตัวอย่าง: ติดตั้ง extension ที่ต้องการให้ database ใหม่ทุกตัวมีอัตโนมัติ
\c template1
CREATE EXTENSION IF NOT EXISTS pgcrypto;
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
```

หลังจากนี้ ทุกครั้งที่สร้างฐานข้อมูลใหม่ (โดยไม่ระบุ template อื่น) จะมี extension `pgcrypto` และ `uuid-ossp` ติดมาให้อัตโนมัติ

### ตารางเปรียบเทียบ

| คุณสมบัติ | template0 | template1 |
|---|---|---|
| แก้ไขได้ไหม | ไม่ได้ (allowconn = false) | ได้ |
| มี object ที่ผู้ใช้เพิ่มไหม | ไม่มี (สะอาดเสมอ) | อาจมี ถ้าเคยแก้ไข |
| ใช้เมื่อไหร่ | ต้องการ encoding/locale ใหม่ หรือ full restore | สร้างฐานข้อมูลทั่วไปแบบ default |
| ตัวอย่างคำสั่ง | `CREATE DATABASE x TEMPLATE template0` | `CREATE DATABASE x` (default) |

### เหตุผลที่ต้องใช้ template0 เมื่อเปลี่ยน encoding/locale

ถ้าลองสร้างฐานข้อมูลใหม่โดยระบุ encoding ต่างจาก `template1` โดยไม่ระบุ `TEMPLATE template0` จะเกิด error

```sql
CREATE DATABASE latin_db ENCODING 'LATIN1';
```

```
ERROR:  new encoding (LATIN1) is incompatible with the encoding of the template database (UTF8)
HINT:  Use the same encoding as in the template database, or use template0 as template.
```

วิธีแก้:

```sql
CREATE DATABASE latin_db
    ENCODING 'LATIN1'
    TEMPLATE template0
    LC_COLLATE 'C'
    LC_CTYPE 'C';
```

```
CREATE DATABASE
```

### การสร้าง Custom Template Database ของตัวเอง

ในโปรเจกต์จริง หลายทีมสร้าง "custom template" ที่มี schema, extension, role มาตรฐานติดตั้งไว้ล่วงหน้า เพื่อให้สร้างฐานข้อมูลใหม่สำหรับแต่ละลูกค้า (multi-tenant) หรือแต่ละ environment ได้เร็วและสม่ำเสมอ

**ขั้นตอนที่ 1: สร้างฐานข้อมูลต้นแบบตามปกติ**

```sql
CREATE DATABASE company_template
    TEMPLATE template0
    ENCODING 'UTF8'
    LC_COLLATE 'en_US.UTF-8'
    LC_CTYPE 'en_US.UTF-8';
```

**ขั้นตอนที่ 2: เชื่อมต่อเข้าไปตั้งค่ามาตรฐานที่ต้องการ**

```sql
\c company_template

CREATE EXTENSION IF NOT EXISTS pgcrypto;

CREATE SCHEMA audit;

CREATE TABLE audit.change_log (
    id          bigserial PRIMARY KEY,
    table_name  text NOT NULL,
    changed_at  timestamptz NOT NULL DEFAULT now(),
    changed_by  text NOT NULL
);

CREATE ROLE readonly_user NOLOGIN;
GRANT USAGE ON SCHEMA public, audit TO readonly_user;
```

**ขั้นตอนที่ 3: ทำเครื่องหมายให้เป็น template และป้องกันการแก้ไข**

```sql
UPDATE pg_database
SET datistemplate = true
WHERE datname = 'company_template';

-- (แนะนำ) ปิดการเชื่อมต่อปกติเพื่อป้องกันการแก้ไขโดยไม่ตั้งใจ
UPDATE pg_database
SET datallowconn = false
WHERE datname = 'company_template';
```

**ขั้นตอนที่ 4: ใช้งาน template สำหรับสร้างฐานข้อมูลลูกค้าใหม่**

```sql
CREATE DATABASE tenant_acme
    TEMPLATE company_template
    OWNER app_shop;

CREATE DATABASE tenant_globex
    TEMPLATE company_template
    OWNER app_shop;
```

ทุกฐานข้อมูลใหม่ที่สร้างจาก `company_template` จะมี schema `audit`, table `audit.change_log`, extension `pgcrypto` และ role `readonly_user` ติดตั้งมาให้พร้อมใช้งานทันที ไม่ต้องรัน migration script ซ้ำ ๆ

> **ข้อควรระวัง**: ห้ามมี connection ค้างอยู่ในฐานข้อมูล template ขณะที่กำลังใช้เป็นต้นแบบสร้างฐานข้อมูลใหม่ ไม่เช่นนั้นจะเกิด error แบบเดียวกับตอน `DROP DATABASE` (ดู Step 45)

---

## Step 44: ALTER DATABASE — เปลี่ยนชื่อ, เจ้าของ, connection limit, parameter

### Syntax ภาพรวม

```sql
ALTER DATABASE name RENAME TO new_name;
ALTER DATABASE name OWNER TO { new_owner | CURRENT_ROLE | CURRENT_USER | SESSION_USER };
ALTER DATABASE name SET TABLESPACE new_tablespace;
ALTER DATABASE name WITH option [ ... ];
ALTER DATABASE name SET configuration_parameter { TO | = } value | DEFAULT;
ALTER DATABASE name SET configuration_parameter FROM CURRENT;
ALTER DATABASE name RESET configuration_parameter;
ALTER DATABASE name RESET ALL;
```

### 1. เปลี่ยนชื่อฐานข้อมูล (RENAME)

```sql
ALTER DATABASE shop_db RENAME TO ecommerce_db;
```

```
ALTER DATABASE
```

> **ข้อจำกัด**: ห้ามมี connection ใด ๆ เชื่อมต่ออยู่ในฐานข้อมูลนั้น (รวมถึง connection ของตัวเองที่กำลัง `\c` เข้าไปอยู่) ต้อง `\c` ไปยังฐานข้อมูลอื่นก่อนแล้วค่อยสั่ง rename

ถ้ามี connection ค้างอยู่จะได้ error:

```
ERROR:  database "shop_db" is being accessed by other users
DETAIL:  There is 1 other session using the database.
```

### 2. เปลี่ยนเจ้าของฐานข้อมูล (OWNER)

```sql
ALTER DATABASE ecommerce_db OWNER TO new_app_role;
```

```
ALTER DATABASE
```

การเปลี่ยน owner มีผลต่อ default privileges และการควบคุมสิทธิ์ระดับฐานข้อมูล (แต่ไม่ได้เปลี่ยน owner ของ object ภายในโดยอัตโนมัติ — table/schema ที่มีอยู่แล้วยังเป็นของ owner เดิม ต้อง `REASSIGN OWNED` แยกต่างหาก)

```sql
-- เปลี่ยน owner ของ object ทั้งหมดที่เป็นของ role เดิม ให้เป็นของ role ใหม่
REASSIGN OWNED BY old_app_role TO new_app_role;
```

### 3. ตั้งค่า Connection Limit

จำกัดจำนวน connection พร้อมกันสูงสุดที่ฐานข้อมูลนี้จะรับได้ (ไม่นับ superuser ซึ่งไม่ถูกจำกัด)

```sql
ALTER DATABASE ecommerce_db WITH CONNECTION LIMIT 50;
```

```
ALTER DATABASE
```

ตรวจสอบค่าปัจจุบัน:

```sql
SELECT datname, datconnlimit
FROM pg_database
WHERE datname = 'ecommerce_db';
```

```
    datname    | datconnlimit
---------------+--------------
 ecommerce_db  |           50
```

ยกเลิกการจำกัด (กลับเป็นไม่จำกัด):

```sql
ALTER DATABASE ecommerce_db WITH CONNECTION LIMIT -1;
```

> **กรณีใช้งานจริง**: เหมาะกับการป้องกันไม่ให้ฐานข้อมูล staging/dev ที่มีทรัพยากรจำกัด ถูก connection pool ของ application ใช้จนหมด หรือใช้จำกัดฐานข้อมูลของทีมที่ยังไม่ optimize connection pooling ให้ดี

### 4. ตั้งค่า Parameter เฉพาะฐานข้อมูล (Per-Database Configuration)

PostgreSQL อนุญาตให้ override ค่า configuration parameter (ที่ปกติตั้งใน `postgresql.conf`) ให้ใช้เฉพาะฐานข้อมูลใดฐานข้อมูลหนึ่งได้ โดยไม่กระทบฐานข้อมูลอื่นในเครื่องเดียวกัน

```sql
-- ตั้งค่า statement_timeout เฉพาะฐานข้อมูล analytics_db ให้ยาวขึ้น (query วิเคราะห์ข้อมูลหนัก)
ALTER DATABASE analytics_db SET statement_timeout = '30min';

-- ตั้งค่า work_mem ให้สูงขึ้นเฉพาะฐานข้อมูลนี้
ALTER DATABASE analytics_db SET work_mem = '256MB';

-- ตั้งค่า search_path default ให้รวม schema เฉพาะ
ALTER DATABASE ecommerce_db SET search_path = "$user", public, sales;

-- ปิด JIT compilation เฉพาะฐานข้อมูล staging ที่ query เล็ก ๆ บ่อย
ALTER DATABASE staging_db SET jit = off;
```

```
ALTER DATABASE
```

ตรวจสอบค่าที่ตั้งไว้ทั้งหมด:

```sql
SELECT datname, setconfig
FROM pg_db_role_setting drs
JOIN pg_database d ON drs.setdatabase = d.oid
WHERE d.datname = 'analytics_db';
```

```
   datname    |                setconfig
--------------+-------------------------------------------
 analytics_db | {statement_timeout=30min,work_mem=256MB}
```

หรือใช้ psql meta-command ที่แสดงคอลัมน์นี้ให้อ่านง่ายกว่า:

```sql
\l+ analytics_db
```

ยกเลิกการตั้งค่าเฉพาะ parameter:

```sql
ALTER DATABASE analytics_db RESET work_mem;
```

ยกเลิกทั้งหมด (กลับไปใช้ค่าจาก `postgresql.conf`):

```sql
ALTER DATABASE analytics_db RESET ALL;
```

### 5. เปลี่ยน Tablespace ของฐานข้อมูล

```sql
ALTER DATABASE ecommerce_db SET TABLESPACE fast_ssd_ts;
```

> คำสั่งนี้จะย้ายไฟล์ข้อมูลจริงทั้งหมดของฐานข้อมูลไปยัง tablespace ใหม่ ใช้เวลาตามขนาดข้อมูล และ **ต้องไม่มี connection อื่นใช้งานฐานข้อมูลนั้นอยู่** (รายละเอียดเพิ่มเติมใน Step 49)

---

## Step 45: DROP DATABASE และข้อควรระวัง

### Syntax พื้นฐาน

```sql
DROP DATABASE [ IF EXISTS ] name [ WITH ( FORCE ) ];
```

ตัวอย่าง:

```sql
DROP DATABASE staging_old_db;
```

```
DROP DATABASE
```

### คำเตือนสำคัญที่สุด: DROP DATABASE ทำลายข้อมูลถาวร ไม่มี Rollback

`DROP DATABASE` **ไม่สามารถรันภายใน transaction block ได้** และเมื่อรันสำเร็จ **ข้อมูลและไฟล์ทั้งหมดจะถูกลบทิ้งทันทีแบบกู้คืนไม่ได้** (ไม่มี recycle bin) ดังนั้นก่อนรันคำสั่งนี้ในระบบ production ควรมี checklist เสมอ:

1. ตรวจสอบชื่อฐานข้อมูลซ้ำแล้วซ้ำเล่า (พิมพ์ผิดเป็นเรื่องที่เกิดขึ้นได้ง่ายมาก)
2. มี backup ล่าสุดหรือยัง (`pg_dump` หรือ snapshot)
3. ไม่มีระบบ production ใดอ้างอิงถึงฐานข้อมูลนี้อยู่แล้ว
4. ได้รับการอนุมัติตามขั้นตอนของทีม (change management)

### ป้องกันการพิมพ์ผิดด้วย IF EXISTS

```sql
DROP DATABASE IF EXISTS temp_test_db;
```

ถ้าไม่มีฐานข้อมูลนี้อยู่จริง จะได้ notice แทน error:

```
NOTICE:  database "temp_test_db" does not exist, skipping
DROP DATABASE
```

### ปัญหาที่พบบ่อยที่สุด: มี Connection ค้างอยู่

```sql
DROP DATABASE ecommerce_db;
```

```
ERROR:  database "ecommerce_db" is being accessed by other users
DETAIL:  There are 3 other sessions using the database.
```

นี่คือ error ที่เจอบ่อยมาก เพราะมี application, connection pool (เช่น PgBouncer), หรือ session ของ psql เองที่ยังเชื่อมต่อค้างอยู่

**วิธีที่ 1: ตรวจสอบและตัด connection ด้วยมือ (แนะนำสำหรับ production)**

```sql
-- ดู session ที่เชื่อมต่ออยู่ในฐานข้อมูลเป้าหมาย
SELECT pid, usename, application_name, client_addr, state
FROM pg_stat_activity
WHERE datname = 'ecommerce_db';
```

```
  pid  | usename  | application_name | client_addr |  state
-------+----------+-------------------+-------------+--------
 12345 | app_shop | node-app          | 10.0.0.12   | idle
 12346 | app_shop | node-app          | 10.0.0.13   | active
```

```sql
-- ตัด connection ทีละตัว (ใช้ pid จริงจากผลลัพธ์ข้างต้น)
SELECT pg_terminate_backend(12345);
SELECT pg_terminate_backend(12346);
```

หรือตัดทุก connection ของฐานข้อมูลนั้นในคำสั่งเดียว (ยกเว้น connection ปัจจุบันของตัวเอง):

```sql
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE datname = 'ecommerce_db'
  AND pid <> pg_backend_pid();
```

จากนั้นค่อยสั่ง `DROP DATABASE`

**วิธีที่ 2: ใช้ WITH (FORCE) — ใหม่ใน PostgreSQL 13+**

PostgreSQL 13 ขึ้นไปมี option `FORCE` ที่ตัด connection ทั้งหมดให้อัตโนมัติในคำสั่งเดียว

```sql
DROP DATABASE ecommerce_db WITH (FORCE);
```

```
DROP DATABASE
```

> **ข้อควรระวังของ FORCE**: คำสั่งนี้จะ **ตัด connection ของทุกคนทันที** โดยไม่มีการเตือนล่วงหน้า ผู้ใช้ที่กำลังทำงานอยู่จะเจอ connection error ทันที ควรใช้เฉพาะกรณีที่แน่ใจแล้วว่าฐานข้อมูลนี้ไม่มีใครใช้งานจริงจัง (เช่น ฐานข้อมูล test/dev) หรือใช้ในกระบวนการที่วางแผน downtime ไว้แล้วเท่านั้น **ไม่ควรใช้กับฐานข้อมูล production โดยไม่ได้วางแผนล่วงหน้า**

### ข้อจำกัดอื่น ๆ ของ DROP DATABASE

1. **ลบตัวเองไม่ได้**: ไม่สามารถ `DROP DATABASE` ฐานข้อมูลที่ตัวเองกำลังเชื่อมต่ออยู่ ต้อง `\c` ไปฐานข้อมูลอื่นก่อน (เช่น `\c postgres`)
2. **ลบ template ไม่ได้โดยตรง**: ฐานข้อมูลที่ `datistemplate = true` ต้อง `UPDATE pg_database SET datistemplate = false` ก่อนถึงจะลบได้
3. **ต้องเป็น owner หรือ superuser**: เฉพาะเจ้าของฐานข้อมูลหรือ superuser เท่านั้นที่ลบได้

```sql
-- ตัวอย่าง: ลบ template database ที่ไม่ใช้แล้ว
UPDATE pg_database SET datistemplate = false WHERE datname = 'company_template_old';
DROP DATABASE company_template_old;
```

### Checklist สรุปก่อน DROP DATABASE บน Production

```
[ ] ยืนยันชื่อฐานข้อมูลถูกต้อง 100%
[ ] มี backup ล่าสุด (pg_dump / snapshot) และทดสอบ restore ได้จริง
[ ] แจ้งทีมที่เกี่ยวข้องและได้รับอนุมัติ
[ ] ตรวจสอบ pg_stat_activity ว่าไม่มี active connection ที่สำคัญ
[ ] พิจารณาใช้ RENAME แทน DROP ก่อน แล้วค่อยลบจริงหลังผ่านช่วง grace period
```

> **เทคนิคที่ทีมมืออาชีพนิยมใช้**: แทนที่จะ `DROP DATABASE` ทันที มักจะ `ALTER DATABASE ... RENAME TO xxx_pending_deletion_20260925` แล้วปิด connection limit เหลือ 0 เพื่อดูว่ามีใครพยายามเชื่อมต่อเข้ามาอีกหรือไม่ ก่อนจะลบจริงหลังผ่านไปสัก 1-2 สัปดาห์

---

## Step 46: การสร้าง Schema (CREATE SCHEMA, ALTER SCHEMA, DROP SCHEMA)

### Schema คืออะไร

Schema คือ "namespace" ภายในฐานข้อมูลเดียว ใช้จัดกลุ่ม object เช่น table, view, function ให้เป็นระเบียบ และช่วยแยกสิทธิ์การเข้าถึงในระดับที่ละเอียดกว่าฐานข้อมูล

ทุกฐานข้อมูลที่สร้างใหม่จะมี schema ชื่อ `public` มาให้อัตโนมัติ (ใน PostgreSQL 15+ ค่า default privilege ของ `public` schema เปลี่ยนไป — ผู้ใช้ทั่วไปจะไม่มีสิทธิ์ `CREATE` ใน `public` โดยอัตโนมัติอีกต่อไป ต่างจาก PostgreSQL เวอร์ชันเก่า)

### CREATE SCHEMA

Syntax:

```sql
CREATE SCHEMA [ IF NOT EXISTS ] schema_name [ AUTHORIZATION role_name ];
CREATE SCHEMA AUTHORIZATION role_name;  -- ใช้ role_name เป็นชื่อ schema ด้วย
```

ตัวอย่างพื้นฐาน:

```sql
CREATE SCHEMA sales;
CREATE SCHEMA inventory;
CREATE SCHEMA audit AUTHORIZATION audit_team;
```

```
CREATE SCHEMA
CREATE SCHEMA
CREATE SCHEMA
```

**สร้าง schema พร้อม object ภายในในคำสั่งเดียว** (schema element):

```sql
CREATE SCHEMA hr
    CREATE TABLE employees (
        id serial PRIMARY KEY,
        full_name text NOT NULL,
        department text
    )
    CREATE VIEW active_employees AS
        SELECT id, full_name, department
        FROM hr.employees;
```

```
CREATE SCHEMA
```

### การใช้งาน Schema — search_path

เมื่อ query โดยไม่ระบุชื่อ schema เช่น `SELECT * FROM employees;` PostgreSQL จะค้นหาตามลำดับใน `search_path`

```sql
SHOW search_path;
```

```
   search_path
-----------------
 "$user", public
```

```sql
-- เพิ่ม schema hr และ sales เข้าไปใน search_path ของ session ปัจจุบัน
SET search_path TO hr, sales, public;

-- ตอนนี้เรียก table โดยไม่ต้องใส่ prefix ก็ได้
SELECT * FROM employees;   -- เท่ากับ hr.employees

-- แต่ระบุ schema เต็มก็ยังทำได้เสมอ ไม่ว่า search_path จะเป็นอะไร
SELECT * FROM sales.orders;
```

### ALTER SCHEMA

```sql
-- เปลี่ยนชื่อ schema
ALTER SCHEMA sales RENAME TO sales_v2;

-- เปลี่ยนเจ้าของ schema
ALTER SCHEMA inventory OWNER TO warehouse_team;
```

```
ALTER SCHEMA
ALTER SCHEMA
```

### DROP SCHEMA

```sql
-- ลบ schema ที่ว่างเปล่า (ไม่มี object อยู่ข้างใน)
DROP SCHEMA old_reports;
```

ถ้า schema มี object อยู่ข้างในจะเจอ error:

```
ERROR:  cannot drop schema old_reports because other objects depend on it
DETAIL:  table old_reports.summary_2023 depends on schema old_reports
HINT:  Use DROP ... CASCADE to drop the dependent objects too.
```

**ใช้ CASCADE เพื่อลบทุกอย่างในนั้นไปด้วย** (ต้องระวังอย่างมาก — ลบ table/view/function ทุกตัวในนั้นทิ้งหมด):

```sql
DROP SCHEMA old_reports CASCADE;
```

```
DROP SCHEMA
```

**ใช้ RESTRICT** (เป็นค่า default อยู่แล้ว แต่เขียนเพื่อความชัดเจนได้) เพื่อป้องกันการลบโดยไม่ตั้งใจถ้ายังมี object อยู่:

```sql
DROP SCHEMA old_reports RESTRICT;
```

> **แนวทางปฏิบัติที่ปลอดภัย**: ก่อนใช้ `CASCADE` ให้ query ดูก่อนเสมอว่ามี object อะไรอยู่ใน schema นั้นบ้าง (ดู Step 47) เพื่อประเมินผลกระทบก่อนลบจริง

---

## Step 47: คำสั่งดูรายการ Databases/Schemas พร้อมขนาด

### ดูรายการฐานข้อมูลด้วย psql meta-command

```
\l
```

```
                                   List of databases
    Name     |  Owner   | Encoding | Locale Provider |  Collate   |   Ctype    | Access privileges
--------------+----------+----------+------------------+------------+------------+-------------------
 analytics_db | analyst  | UTF8     | libc             | en_US.UTF-8| en_US.UTF-8|
 ecommerce_db | app_shop | UTF8     | libc             | en_US.UTF-8| en_US.UTF-8|
 postgres     | postgres | UTF8     | libc             | en_US.UTF-8| en_US.UTF-8|
 template0    | postgres | UTF8     | libc             | en_US.UTF-8| en_US.UTF-8| =c/postgres +
              |          |          |                  |            |            | postgres=CTc/postgres
 template1    | postgres | UTF8     | libc             | en_US.UTF-8| en_US.UTF-8| =c/postgres +
              |          |          |                  |            |            | postgres=CTc/postgres
```

**เพิ่มขนาดฐานข้อมูลด้วย `\l+`:**

```
\l+
```

```
                                                     List of databases
    Name     |  Owner   | Encoding |  Collate   |   Ctype    | Size    | Tablespace | Description
--------------+----------+----------+------------+------------+---------+------------+-------------
 analytics_db | analyst  | UTF8     | en_US.UTF-8| en_US.UTF-8| 1024 MB | pg_default |
 ecommerce_db | app_shop | UTF8     | en_US.UTF-8| en_US.UTF-8| 256 MB  | pg_default |
 postgres     | postgres | UTF8     | en_US.UTF-8| en_US.UTF-8| 8617 kB | pg_default | default admin connection database
```

### ดูขนาดฐานข้อมูลด้วย SQL โดยตรง

ฟังก์ชันสำคัญคือ `pg_database_size()`

```sql
SELECT pg_size_pretty(pg_database_size('ecommerce_db')) AS size;
```

```
  size
--------
 256 MB
```

**ดูขนาดของทุกฐานข้อมูลพร้อมเรียงจากใหญ่ไปเล็ก:**

```sql
SELECT
    datname,
    pg_size_pretty(pg_database_size(datname)) AS size,
    pg_database_size(datname) AS size_bytes
FROM pg_database
WHERE datistemplate = false
ORDER BY pg_database_size(datname) DESC;
```

```
    datname    |  size   | size_bytes
---------------+---------+------------
 analytics_db  | 1024 MB | 1073741824
 ecommerce_db  | 256 MB  |  268435456
 postgres      | 8617 kB |    8823296
```

### ดูรายการ Schema

```
\dn
```

```
       List of schemas
    Name     |    Owner
--------------+-------------
 audit        | audit_team
 hr           | app_shop
 public       | pg_database_owner
 sales        | app_shop
```

**เพิ่มรายละเอียดสิทธิ์และคำอธิบายด้วย `\dn+`:**

```
\dn+
```

```
                            List of schemas
    Name     |    Owner    |          Access privileges          | Description
--------------+-------------+--------------------------------------+-------------
 audit        | audit_team  | audit_team=UC/audit_team            |
 hr           | app_shop    | app_shop=UC/app_shop                |
 public       | pg_database_owner | pg_database_owner=UC/pg_database_owner+|
              |             | =U/pg_database_owner                 | standard public schema
 sales        | app_shop    | app_shop=UC/app_shop                |
```

### ดูขนาดของแต่ละ Schema (รวมทุก table ในนั้น)

PostgreSQL ไม่มีฟังก์ชันสำเร็จรูปสำหรับขนาดของ schema โดยตรง ต้อง query รวม `pg_total_relation_size()` ของทุก table ในนั้น

```sql
SELECT
    schemaname,
    pg_size_pretty(SUM(pg_total_relation_size(schemaname || '.' || tablename))::bigint) AS total_size
FROM pg_tables
WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
GROUP BY schemaname
ORDER BY SUM(pg_total_relation_size(schemaname || '.' || tablename)) DESC;
```

```
 schemaname |  total_size
------------+---------------
 sales      | 180 MB
 hr         | 45 MB
 audit      | 12 MB
```

### ดูรายการ object ทั้งหมดใน Schema หนึ่ง ๆ

```sql
\dt sales.*
```

```
             List of relations
 Schema |    Name    | Type  |  Owner
--------+------------+-------+----------
 sales  | customers  | table | app_shop
 sales  | orders     | table | app_shop
 sales  | order_items| table | app_shop
```

หรือดูทุกชนิด object (table, view, sequence, ...) ด้วย SQL จาก `information_schema` / `pg_catalog`:

```sql
SELECT n.nspname AS schema_name, c.relname AS object_name,
       CASE c.relkind
           WHEN 'r' THEN 'table'
           WHEN 'v' THEN 'view'
           WHEN 'S' THEN 'sequence'
           WHEN 'i' THEN 'index'
           WHEN 'm' THEN 'materialized view'
       END AS object_type
FROM pg_class c
JOIN pg_namespace n ON n.oid = c.relnamespace
WHERE n.nspname = 'sales'
ORDER BY object_type, object_name;
```

### ดูฐานข้อมูลปัจจุบันและ schema ปัจจุบัน

```sql
SELECT current_database(), current_schema();
```

```
 current_database | current_schema
-------------------+-----------------
 ecommerce_db      | public
```

---

## Step 48: CREATE SCHEMA IF NOT EXISTS และสิทธิ์ในการสร้าง Database/Schema

### CREATE SCHEMA IF NOT EXISTS

ต่างจาก `CREATE DATABASE` ตรงที่ `CREATE SCHEMA` **รองรับ** `IF NOT EXISTS` ได้ (เพราะทำงานภายใน transaction ได้ตามปกติ)

```sql
CREATE SCHEMA IF NOT EXISTS sales;
```

ถ้ามี schema นี้อยู่แล้ว:

```
NOTICE:  schema "sales" already exists, skipping
CREATE SCHEMA
```

รูปแบบนี้มีประโยชน์มากในการเขียน migration script ที่ต้อง idempotent (รันซ้ำกี่ครั้งก็ได้ผลลัพธ์เดิม ไม่ error)

```sql
-- ตัวอย่างใน migration script
CREATE SCHEMA IF NOT EXISTS sales;
CREATE SCHEMA IF NOT EXISTS inventory;
CREATE SCHEMA IF NOT EXISTS audit;

CREATE TABLE IF NOT EXISTS sales.orders (
    id bigserial PRIMARY KEY,
    created_at timestamptz NOT NULL DEFAULT now()
);
```

### สิทธิ์ในการสร้าง Database: Superuser vs CREATEDB Role

ในการจะรัน `CREATE DATABASE` ได้ role นั้นต้องมีคุณสมบัติอย่างใดอย่างหนึ่งต่อไปนี้:

1. เป็น **superuser** — มีสิทธิ์ทำได้ทุกอย่างในระบบ ไม่มีข้อจำกัด
2. มี **role attribute `CREATEDB`** — สิทธิ์จำกัดเฉพาะการสร้าง/ลบฐานข้อมูล

ตรวจสอบว่า role ไหนมีสิทธิ์อะไรบ้าง:

```sql
SELECT rolname, rolsuper, rolcreatedb, rolcreaterole, rolcanlogin
FROM pg_roles
ORDER BY rolname;
```

```
   rolname   | rolsuper | rolcreatedb | rolcreaterole | rolcanlogin
--------------+----------+-------------+----------------+-------------
 app_shop     | f        | f           | f              | t
 analyst_team | f        | t           | f              | t
 postgres     | t        | t           | t              | t
```

จากตัวอย่างข้างบน: `postgres` เป็น superuser ทำได้ทุกอย่าง, `analyst_team` มีสิทธิ์ `CREATEDB` สามารถสร้างฐานข้อมูลเองได้แต่ไม่ใช่ superuser, ส่วน `app_shop` สร้างฐานข้อมูลไม่ได้เลย

**ให้สิทธิ์ CREATEDB กับ role ที่มีอยู่แล้ว:**

```sql
ALTER ROLE analyst_team CREATEDB;
```

```
ALTER ROLE
```

**ถอดสิทธิ์ CREATEDB:**

```sql
ALTER ROLE analyst_team NOCREATEDB;
```

**สร้าง role พร้อมสิทธิ์ CREATEDB ตั้งแต่แรก:**

```sql
CREATE ROLE devops_team WITH LOGIN PASSWORD 'xxx' CREATEDB;
```

### ข้อจำกัดของ role ที่มี CREATEDB (ไม่ใช่ superuser)

แม้ role จะมี `CREATEDB` แต่ก็ **ไม่ใช่** superuser ดังนั้นยังมีข้อจำกัดสำคัญ เช่น

- ไม่สามารถสร้างฐานข้อมูลโดยกำหนด `OWNER` เป็น role อื่นได้ (ทำได้เฉพาะให้ตัวเองเป็น owner หรือถ้า role นั้นเป็นสมาชิกของตน)
- ไม่สามารถ bypass row-level security ได้
- ยังคงต้องมีสิทธิ์ที่เหมาะสมในการติดตั้ง extension บางตัวที่ต้องการ superuser (เช่น extension ที่เข้าถึงระบบไฟล์)

### สิทธิ์ในการสร้าง Schema

การสร้าง schema ต้องการสิทธิ์ `CREATE` บนฐานข้อมูลนั้น (ไม่ใช่ role attribute แบบ `CREATEDB`) ตรวจสอบสิทธิ์ปัจจุบันของฐานข้อมูล:

```sql
SELECT datname, datacl FROM pg_database WHERE datname = current_database();
```

**ให้สิทธิ์สร้าง schema กับ role หนึ่ง:**

```sql
GRANT CREATE ON DATABASE ecommerce_db TO analyst_team;
```

```
GRANT
```

**ถอดสิทธิ์:**

```sql
REVOKE CREATE ON DATABASE ecommerce_db FROM analyst_team;
```

### สรุปตารางสิทธิ์

| การกระทำ | ต้องมีสิทธิ์ |
|---|---|
| `CREATE DATABASE` | superuser หรือ role ที่มี `CREATEDB` |
| `DROP DATABASE` | superuser หรือ owner ของฐานข้อมูลนั้น (และมี `CREATEDB`) |
| `CREATE SCHEMA` | สิทธิ์ `CREATE` บนฐานข้อมูลนั้น |
| `DROP SCHEMA` | superuser หรือ owner ของ schema นั้น |
| `ALTER DATABASE ... OWNER TO` | superuser หรือ owner เดิมที่เป็นสมาชิกของ role ใหม่ |

---

## Step 49: การใช้ Tablespace เบื้องต้น

### Tablespace คืออะไร

Tablespace คือการกำหนดว่า "ไฟล์ข้อมูลจริง" ของ object (table, index, database) จะถูกเก็บไว้ที่ไดเรกทอรีไหนบนดิสก์ โดย default ทุกอย่างจะถูกเก็บใน tablespace ชื่อ `pg_default` ซึ่งอยู่ในไดเรกทอรีข้อมูลหลักของ PostgreSQL (`PGDATA`)

ประโยชน์ของการใช้ tablespace หลายตัว:

- แยกข้อมูลที่เข้าถึงบ่อย (hot data) ไปไว้บน SSD ที่เร็ว และข้อมูลเก่าที่เข้าถึงน้อย (cold data / archive) ไปไว้บน HDD ที่ถูกกว่า
- แยก index ออกจาก table เพื่อกระจาย I/O
- จัดการพื้นที่ดิสก์เมื่อดิสก์หลักเต็ม โดยไม่ต้องย้ายทั้งระบบ

### สร้าง Tablespace

ก่อนสร้าง ต้องมีไดเรกทอรีอยู่บนเครื่อง server จริงและ PostgreSQL ต้องมีสิทธิ์เขียนไฟล์ในนั้น (เป็น owner ของโฟลเดอร์)

```bash
# รันบนเครื่อง server (OS level) — สร้างโฟลเดอร์และให้สิทธิ์ user ที่รัน postgres
sudo mkdir -p /mnt/fast_ssd/pg_tablespaces/fast_ssd_ts
sudo chown postgres:postgres /mnt/fast_ssd/pg_tablespaces/fast_ssd_ts
```

จากนั้นสร้าง tablespace ใน PostgreSQL:

```sql
CREATE TABLESPACE fast_ssd_ts
    OWNER app_shop
    LOCATION '/mnt/fast_ssd/pg_tablespaces/fast_ssd_ts';
```

```
CREATE TABLESPACE
```

สร้าง tablespace สำหรับข้อมูลเก่า (archive):

```sql
CREATE TABLESPACE archive_hdd_ts
    LOCATION '/mnt/archive_hdd/pg_tablespaces/archive_hdd_ts';
```

### ดูรายการ Tablespace

```
\db+
```

```
                                    List of tablespaces
     Name      |  Owner   |             Location              | Access privileges | Size
----------------+----------+------------------------------------+--------------------+--------
 archive_hdd_ts | postgres | /mnt/archive_hdd/pg_tablespaces/... |                    | 4 GB
 fast_ssd_ts    | app_shop | /mnt/fast_ssd/pg_tablespaces/...    |                    | 512 MB
 pg_default     | postgres |                                    |                    | 2048 MB
 pg_global      | postgres |                                    |                    | 1024 kB
```

หรือดูด้วย SQL:

```sql
SELECT spcname, pg_tablespace_location(oid), pg_size_pretty(pg_tablespace_size(spcname))
FROM pg_tablespace;
```

### สร้าง Table ลงใน Tablespace ที่กำหนด

```sql
CREATE TABLE sales.orders_2026 (
    id bigserial PRIMARY KEY,
    order_date date NOT NULL,
    total_amount numeric(12,2)
) TABLESPACE fast_ssd_ts;
```

```
CREATE TABLE
```

### ย้าย Table ที่มีอยู่แล้วไป Tablespace อื่น

```sql
ALTER TABLE sales.orders_2020 SET TABLESPACE archive_hdd_ts;
```

```
ALTER TABLE
```

> คำสั่งนี้จะ lock table ระหว่างย้ายข้อมูล (เขียนไฟล์ใหม่ทั้งหมด) สำหรับ table ขนาดใหญ่ควรวางแผนช่วงเวลาที่ traffic น้อย

### ย้าย Index ไป Tablespace อื่น

```sql
ALTER INDEX sales.idx_orders_2020_date SET TABLESPACE archive_hdd_ts;
```

### ย้ายทุก Table ในฐานข้อมูลไป Tablespace ใหม่ในคำสั่งเดียว

```sql
ALTER DATABASE ecommerce_db SET TABLESPACE fast_ssd_ts;
```

> คำสั่งนี้ย้าย "ทั้งฐานข้อมูล" (ทุก object ที่อยู่ใน `pg_default` เดิม) ไปยัง tablespace ใหม่ ต้องไม่มี connection อื่นใช้งานฐานข้อมูลนั้นอยู่ระหว่างดำเนินการ

### กำหนด Tablespace เป็นค่า default สำหรับ table ใหม่ทั้งหมดใน session

```sql
SET default_tablespace = fast_ssd_ts;

CREATE TABLE sales.new_orders (id bigserial PRIMARY KEY);
-- table นี้จะถูกสร้างใน fast_ssd_ts โดยอัตโนมัติ

RESET default_tablespace;
```

### ลบ Tablespace

Tablespace ต้อง **ว่างเปล่า** (ไม่มี object ใดใช้งานอยู่) ก่อนจึงจะลบได้

```sql
DROP TABLESPACE archive_hdd_ts;
```

ถ้ายังมี object ใช้งานอยู่:

```
ERROR:  tablespace "archive_hdd_ts" is not empty
```

---

## Step 50: Best Practices — การตั้งชื่อและวางแผนโครงสร้างฐานข้อมูล

### หลักการตั้งชื่อ Database

1. **ใช้ตัวพิมพ์เล็กและ underscore เสมอ** (`snake_case`) — เพราะ PostgreSQL แปลงชื่อ identifier ที่ไม่ได้ใส่ quote เป็นตัวพิมพ์เล็กอัตโนมัติ การผสมตัวพิมพ์ใหญ่-เล็กจะสร้างความสับสนและบังคับให้ต้อง quote ทุกครั้ง

   ```
   ดี:    shop_db, hr_system, analytics_dw
   ไม่ดี: ShopDB, HR-System, "Analytics DW"
   ```

2. **สื่อความหมายของระบบ/โปรเจกต์อย่างชัดเจน** ไม่ใช้ชื่อกำกวมอย่าง `db1`, `test`, `mydb`

3. **ระบุ environment แยกให้ชัดเจน** เมื่อจำเป็นต้องแยกฐานข้อมูลจริงต่างเครื่อง/ต่าง instance

   ```
   shop_dev, shop_staging, shop_prod
   ```

   (แต่โดยทั่วไป แนวทางที่ดีกว่าคือแยก environment ด้วย **server/instance คนละตัว** ไม่ใช่แค่ต่อชื่อฐานข้อมูลในเครื่องเดียวกัน เพื่อป้องกันความเสี่ยงที่ script รันผิด environment โดยไม่ตั้งใจ)

4. **หลีกเลี่ยง reserved words** เช่น `user`, `order`, `group` เป็นชื่อฐานข้อมูลหรือ schema แม้จะใช้ได้ถ้า quote แต่จะสร้างความยุ่งยากในระยะยาว

### หลักการตั้งชื่อ Schema

1. **ใช้ schema แยกตาม domain/business area** ไม่ใช่ตามทีมพัฒนา เช่น

   ```
   ดี:    sales, inventory, hr, audit, reporting
   ไม่ดี: team_a_schema, backend_schema
   ```

2. **แยก schema สำหรับข้อมูล audit/log ออกจาก business data เสมอ** ทำให้ backup/retention policy ตั้งค่าต่างกันได้ง่าย เช่น

   ```sql
   CREATE SCHEMA audit;
   CREATE SCHEMA reporting;   -- สำหรับ materialized view / denormalized data สำหรับ BI
   ```

3. **หลีกเลี่ยงการสร้างทุกอย่างใน `public` schema** โดยเฉพาะระบบขนาดใหญ่ที่มีหลาย module เพราะจะทำให้ยากต่อการจัดการสิทธิ์และมองภาพรวมของระบบ

4. **ใช้ prefix ที่สื่อความหมายเมื่อจำเป็นต้อง versioning schema** เช่นระหว่าง migration ใหญ่

   ```
   sales_v1, sales_v2   (ชั่วคราวระหว่าง migration เท่านั้น ไม่ใช่ pattern ถาวร)
   ```

### การวางแผนโครงสร้างสำหรับโปรเจกต์หลายทีม/หลาย Environment

**รูปแบบที่ 1: Single Database, Multiple Schemas (แนะนำสำหรับทีมขนาดกลาง ระบบที่มี module สัมพันธ์กัน)**

```
ecommerce_prod (database)
 ├── sales       (schema — ทีม Sales/Order)
 ├── inventory   (schema — ทีม Warehouse)
 ├── customer    (schema — ทีม CRM)
 ├── audit       (schema — ทีม Compliance, read-only สำหรับคนอื่น)
 └── reporting   (schema — ทีม Data/BI, materialized views)
```

ข้อดี: query ข้าม module ทำ join ได้ตรง ๆ ไม่ต้องพึ่ง foreign data wrapper, การจัดการ backup/restore ทำเป็นก้อนเดียว

ข้อควรระวัง: ต้องวาง permission ต่อ schema ให้รัดกุม ไม่ให้ทีมหนึ่งเขียนข้อมูลของอีกทีมโดยไม่ได้ตั้งใจ

```sql
-- ตัวอย่างการจำกัดสิทธิ์ตามทีม
GRANT USAGE ON SCHEMA sales TO sales_team;
GRANT SELECT, INSERT, UPDATE ON ALL TABLES IN SCHEMA sales TO sales_team;

GRANT USAGE ON SCHEMA sales TO inventory_team;
GRANT SELECT ON ALL TABLES IN SCHEMA sales TO inventory_team;  -- อ่านได้อย่างเดียว
```

**รูปแบบที่ 2: Multiple Databases (แนะนำเมื่อ module เป็นอิสระจากกันจริง ๆ หรือใช้ microservice architecture)**

```
PostgreSQL Server (instance)
 ├── sales_db      (แยกอิสระ เข้าถึงได้เฉพาะ sales-service)
 ├── inventory_db  (แยกอิสระ เข้าถึงได้เฉพาะ inventory-service)
 └── customer_db   (แยกอิสระ เข้าถึงได้เฉพาะ customer-service)
```

ข้อดี: isolation สมบูรณ์ทั้งในแง่สิทธิ์และ resource, สอดคล้องกับสถาปัตยกรรม microservice ที่แต่ละ service เป็นเจ้าของข้อมูลตัวเอง

ข้อควรระวัง: join ข้าม database ทำไม่ได้โดยตรง ต้องพึ่ง API เรียกข้าม service หรือ `postgres_fdw`/ETL หากต้องวิเคราะห์ข้อมูลรวม

**การแยก Environment (dev/staging/prod)**

แนวทางที่แนะนำที่สุดคือ **แยกด้วย PostgreSQL instance/server คนละตัว** (คนละเครื่อง คนละ connection string) ไม่ใช่แค่คนละฐานข้อมูลในเครื่องเดียวกัน เหตุผลหลัก:

1. ป้องกันความเสี่ยงร้ายแรงที่สุด — engineer เผลอรัน migration หรือ script ทำลายข้อมูลผิด environment เพราะเชื่อมต่อผิดฐานข้อมูลในเครื่องเดียวกัน
2. resource (CPU/RAM/disk) ของ production ไม่ถูกกระทบจาก workload ทดสอบของ dev/staging
3. ควบคุมสิทธิ์และ network access ได้ง่ายกว่า (เช่น prod อยู่หลัง VPN/firewall ที่เข้มงวดกว่า dev)

```
Dev instance:      postgres://dev-db.internal:5432/ecommerce
Staging instance:  postgres://staging-db.internal:5432/ecommerce
Prod instance:     postgres://prod-db.internal:5432/ecommerce
```

ในแต่ละ instance ใช้ชื่อฐานข้อมูล/schema **เหมือนกันทุกประการ** (`ecommerce` ไม่ใช่ `ecommerce_dev` / `ecommerce_prod`) เพื่อให้ connection string ต่างกันแค่ host เท่านั้น ลด configuration drift และลดโอกาสที่ migration script จะทำงานต่างกันระหว่าง environment

### Checklist สรุป Best Practices

```
[ ] ตั้งชื่อ database/schema เป็น snake_case สื่อความหมาย ไม่ใช้ reserved word
[ ] เลือก ENCODING UTF8 เสมอสำหรับระบบใหม่ เว้นแต่มีเหตุผลจำเพาะ
[ ] เลือก locale ให้เหมาะกับความต้องการ (ICU แนะนำสำหรับระบบใหม่ที่ต้อง portable)
[ ] วางแผน schema แยกตาม business domain ไม่ใช่ตามทีมพัฒนา
[ ] ใช้ CREATEDB role attribute แทน superuser สำหรับ automation/CI
[ ] แยก environment (dev/staging/prod) ด้วยคนละ instance เสมอ
[ ] ตั้ง CONNECTION LIMIT และ per-database parameter ให้เหมาะกับ workload
[ ] มี checklist และ backup ก่อน DROP DATABASE ทุกครั้งบน production
[ ] พิจารณา tablespace แยกเมื่อข้อมูลมีขนาดใหญ่และมี hot/cold data ชัดเจน
```

---

## สรุปท้ายบท

ใน Part นี้เราได้เรียนรู้การจัดการฐานข้อมูลระดับบนสุดของ PostgreSQL อย่างครบถ้วน ตั้งแต่:

- **CREATE DATABASE** พร้อม options เต็มรูปแบบ ทั้ง `OWNER`, `TEMPLATE`, `ENCODING`, `LOCALE`, `TABLESPACE`
- **Encoding และ Locale** — เข้าใจว่าทำไม UTF8 คือมาตรฐาน และปัญหา collation ที่ต้องระวังเมื่อย้าย server ข้าม OS
- **Template databases** — `template0` สะอาดเสมอใช้สำหรับ restore/เปลี่ยน encoding, `template1` แก้ไขได้ใช้เป็น default และสร้าง custom template ของตัวเองได้
- **ALTER DATABASE** — เปลี่ยนชื่อ เปลี่ยนเจ้าของ ตั้ง connection limit และ override parameter เฉพาะฐานข้อมูล
- **DROP DATABASE** — ต้องไม่มี connection ค้าง และใช้ `WITH (FORCE)` อย่างระมัดระวังใน PostgreSQL 13+
- **Schema management** — `CREATE/ALTER/DROP SCHEMA` และการจัดระเบียบ object ภายในฐานข้อมูลเดียว
- คำสั่งตรวจสอบขนาดและรายการ database/schema ด้วย `\l+`, `\dn+`, `pg_database_size()`
- สิทธิ์การสร้าง database/schema ผ่าน superuser หรือ `CREATEDB` role attribute
- **Tablespace เบื้องต้น** — แยกข้อมูลตามความเร็วดิสก์ และย้าย table/database ระหว่าง tablespace
- **Best practices** การตั้งชื่อและวางแผนโครงสร้างสำหรับทีมและหลาย environment

ความรู้ใน Part นี้เป็นพื้นฐานสำคัญที่จะใช้ตลอดทั้งหลักสูตร เพราะทุก object ที่เราจะสร้างต่อจากนี้ (table, index, function, ฯลฯ) ล้วนต้องอยู่ภายใน database และ schema ที่วางแผนมาอย่างดีตั้งแต่ต้น

---

## แบบฝึกหัด

**1.** จงเขียนคำสั่ง `CREATE DATABASE` สำหรับสร้างฐานข้อมูลชื่อ `library_db` โดยกำหนดให้ owner เป็น role ชื่อ `librarian`, encoding เป็น `UTF8`, ใช้ `template0`, และ `LC_COLLATE`/`LC_CTYPE` เป็น `en_US.UTF-8`

**2.** อธิบายความแตกต่างระหว่าง `template0` และ `template1` และบอกว่าในสถานการณ์ใดที่จำเป็นต้องใช้ `template0` แทน `template1`

**3.** ทีมของคุณต้องการสร้างฐานข้อมูลใหม่ที่ใช้ `ENCODING 'LATIN1'` แต่ `template1` ปัจจุบันเป็น `UTF8` จงเขียนคำสั่งที่ถูกต้องเพื่อสร้างฐานข้อมูลนี้โดยไม่เกิด error

**4.** จงเขียนคำสั่งเพื่อจำกัด connection limit ของฐานข้อมูล `reporting_db` ให้ไม่เกิน 20 connections พร้อมกัน และเขียนคำสั่ง SQL เพื่อตรวจสอบค่าที่ตั้งไว้

**5.** เมื่อพยายาม `DROP DATABASE old_project_db` แล้วเจอ error ว่า `database "old_project_db" is being accessed by other users` จงอธิบาย 2 วิธีในการแก้ปัญหานี้ พร้อมข้อควรระวังของแต่ละวิธี

**6.** จงเขียนคำสั่งสร้าง schema ชื่อ `finance` โดยกำหนดให้ role ชื่อ `finance_team` เป็นเจ้าของ แล้วเขียนคำสั่งลบ schema นี้แบบปลอดภัย (กรณีที่อาจมี table อยู่ข้างในแล้ว)

**7.** จงเขียน SQL query เพื่อแสดงรายชื่อฐานข้อมูลทั้งหมดในระบบ (ไม่รวม template) พร้อมขนาด เรียงจากฐานข้อมูลที่มีขนาดใหญ่ที่สุดไปเล็กที่สุด

**8.** role ชื่อ `app_backend` ต้องการสิทธิ์สร้างฐานข้อมูลได้เอง (สำหรับใช้ใน CI/CD pipeline ที่สร้างฐานข้อมูลทดสอบชั่วคราว) แต่ทีม security ไม่ต้องการให้ role นี้เป็น superuser จงเขียนคำสั่งที่เหมาะสม

**9.** จงอธิบายว่าทำไมการแยก environment dev/staging/prod ด้วยคนละ PostgreSQL instance จึงปลอดภัยกว่าการแยกด้วยชื่อฐานข้อมูลในเครื่องเดียวกัน (เช่น `shop_dev`, `shop_prod` อยู่ใน server เดียวกัน)

**10.** จงเขียนคำสั่งสร้าง tablespace ชื่อ `cold_storage_ts` ที่ location `/mnt/hdd/pg_cold` แล้วย้าย table ชื่อ `logs.access_log_2023` (ที่มีอยู่แล้ว) ไปยัง tablespace นี้

---

### เฉลย

**1.**
```sql
CREATE DATABASE library_db
    OWNER librarian
    ENCODING 'UTF8'
    TEMPLATE template0
    LC_COLLATE 'en_US.UTF-8'
    LC_CTYPE 'en_US.UTF-8';
```

**2.**
`template0` เป็นฐานข้อมูลต้นแบบดั้งเดิมที่ห้ามแก้ไข (allowconn = false) รับประกันว่าสะอาด ไม่มี object ใด ๆ ที่ผู้ใช้เพิ่มเข้ามาภายหลัง ส่วน `template1` เป็นฐานข้อมูลต้นแบบที่ใช้เป็น default เมื่อสร้างฐานข้อมูลใหม่ และสามารถแก้ไขเพิ่ม object ลงไปได้ (เช่น ติดตั้ง extension ที่ต้องการให้ทุกฐานข้อมูลใหม่มี)

ต้องใช้ `template0` เมื่อ: (ก) ต้องการสร้างฐานข้อมูลที่มี encoding หรือ locale ต่างจาก `template1` เพราะ PostgreSQL จะปฏิเสธถ้า encoding ไม่ตรงกับ template ที่ไม่ใช่ template0 หรือ (ข) ทำ full restore ด้วย `pg_restore` ที่ต้องการฐานข้อมูลที่สะอาดแน่นอน ไม่มี object แปลกปลอมติดมาจาก template1 ที่อาจถูกแก้ไขไว้

**3.**
```sql
CREATE DATABASE legacy_db
    ENCODING 'LATIN1'
    TEMPLATE template0
    LC_COLLATE 'C'
    LC_CTYPE 'C';
```

**4.**
```sql
ALTER DATABASE reporting_db WITH CONNECTION LIMIT 20;

SELECT datname, datconnlimit
FROM pg_database
WHERE datname = 'reporting_db';
```

**5.**
วิธีที่ 1: ตรวจสอบ session ที่ค้างอยู่ด้วย `SELECT * FROM pg_stat_activity WHERE datname = 'old_project_db';` แล้วตัด connection ทีละตัวด้วย `pg_terminate_backend(pid)` — ข้อควรระวังคือต้องแน่ใจว่า session ที่ตัดไม่ได้กำลังทำ transaction สำคัญค้างอยู่ (อาจทำให้งานที่ยังไม่เสร็จหายไป)

วิธีที่ 2: ใช้ `DROP DATABASE old_project_db WITH (FORCE);` (PostgreSQL 13+) ซึ่งจะตัด connection ทั้งหมดให้อัตโนมัติในคำสั่งเดียว — ข้อควรระวังคือจะตัด connection ของทุกคนทันทีโดยไม่เตือนล่วงหน้า ห้ามใช้กับฐานข้อมูล production ที่ยังมีคนใช้งานจริงอยู่โดยไม่ได้วางแผน downtime ก่อน

**6.**
```sql
CREATE SCHEMA finance AUTHORIZATION finance_team;

-- ลบแบบปลอดภัย ตรวจสอบก่อนว่ามี object อะไรอยู่ข้างในบ้าง
\dt finance.*

-- ถ้าต้องการลบพร้อม object ทั้งหมดข้างในจริง ๆ
DROP SCHEMA finance CASCADE;
```

**7.**
```sql
SELECT
    datname,
    pg_size_pretty(pg_database_size(datname)) AS size
FROM pg_database
WHERE datistemplate = false
ORDER BY pg_database_size(datname) DESC;
```

**8.**
```sql
ALTER ROLE app_backend CREATEDB;
```
วิธีนี้ให้สิทธิ์สร้าง/ลบฐานข้อมูลได้เฉพาะเจาะจง โดยไม่ทำให้ `app_backend` เป็น superuser จึงไม่มีสิทธิ์ bypass ระบบความปลอดภัยอื่น ๆ เช่น row-level security หรือแก้ไข system catalog โดยตรง

**9.**
การแยกด้วยคนละ instance ปลอดภัยกว่าเพราะ: (1) connection string/host ต่างกันโดยสิ้นเชิง ลดโอกาสที่ engineer จะเผลอเชื่อมต่อผิด environment เพราะพิมพ์ชื่อฐานข้อมูลผิดหรือใช้ config ผิดไฟล์ (2) resource การประมวลผลของ production ไม่ถูกกระทบจาก workload ทดสอบหนัก ๆ ใน dev/staging ที่รันบนเครื่องเดียวกัน (3) สามารถกำหนด network security (firewall, VPN) ที่เข้มงวดกว่าให้ prod ได้ โดยไม่กระทบการเข้าถึง dev/staging ที่ทีมพัฒนาต้องการความสะดวกมากกว่า (4) ความผิดพลาดร้ายแรงที่สุดคือการรัน `DROP DATABASE` หรือ migration script ทำลายข้อมูลผิด environment ซึ่งเกิดได้ยากกว่ามากเมื่อต้องสลับ host/connection ทั้งหมด ไม่ใช่แค่สลับชื่อฐานข้อมูล

**10.**
```sql
CREATE TABLESPACE cold_storage_ts
    LOCATION '/mnt/hdd/pg_cold';

ALTER TABLE logs.access_log_2023 SET TABLESPACE cold_storage_ts;
```

---

**บทต่อไป**: [Part 006 — Data Types พื้นฐาน](./part-006-data-types-basic.md)
