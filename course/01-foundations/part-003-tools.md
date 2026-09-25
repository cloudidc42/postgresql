# เครื่องมือทำงาน: psql, pgAdmin, DBeaver, VS Code extensions

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับพื้นฐาน | Part 003

---

## เป้าหมายการเรียนรู้

เมื่อจบบทนี้ ผู้เรียนจะสามารถ:

1. ใช้งาน `psql` command-line client ได้อย่างคล่องแคล่ว ทั้งการเชื่อมต่อ การสำรวจฐานข้อมูล และการรันคำสั่ง SQL
2. จำและใช้ psql meta-commands ที่สำคัญที่สุดได้โดยไม่ต้องเปิดเอกสารทุกครั้ง (`\l`, `\c`, `\dt`, `\d`, `\du`, `\dn`, `\x`, `\timing`, `\q`)
3. รันไฟล์ `.sql` ผ่าน psql และ export ผลลัพธ์ query ออกไปเป็นไฟล์ได้ (`\i`, `\o`, `\copy`)
4. ปรับแต่งสภาพแวดล้อมการทำงานของ psql ด้วย `.psqlrc` และ custom prompt ให้เหมาะกับสไตล์ของตัวเอง
5. ติดตั้งและตั้งค่า pgAdmin 4 เชื่อมต่อกับ PostgreSQL server ได้ครั้งแรก พร้อมเข้าใจโครงสร้าง UI
6. ใช้ Query Tool, ER Diagram tool และดู execution plan แบบ visual ใน pgAdmin 4
7. ติดตั้งและเชื่อมต่อ DBeaver กับ PostgreSQL พร้อมเข้าใจจุดเด่นเมื่อเทียบกับ pgAdmin
8. ใช้ฟีเจอร์ขั้นสูงของ DBeaver เช่น data editor, ER diagram, SQL formatter, data export/import
9. ติดตั้งและตั้งค่า VS Code extensions สำหรับทำงานกับ PostgreSQL โดยตรงจาก editor
10. เลือกเครื่องมือให้เหมาะกับสถานการณ์การทำงานจริง ไม่ว่าจะเป็น developer หรือ DBA

---

## บทนำก่อนเริ่ม

ก่อนหน้านี้เราได้ติดตั้ง PostgreSQL และเข้าใจแนวคิดพื้นฐานไปแล้วใน Part 001–002 ในบทนี้เราจะมาทำความรู้จักกับ "เครื่องมือ" ที่ใช้คุยกับฐานข้อมูล PostgreSQL ในชีวิตประจำวัน

ทำไมต้องเรียนรู้หลายเครื่องมือ? เพราะในโลกการทำงานจริง คุณจะเจอสถานการณ์ที่แตกต่างกัน:

- บางครั้งต้อง SSH เข้าเซิร์ฟเวอร์ที่ไม่มี GUI เลย — ต้องพึ่ง `psql`
- บางครั้งต้องการดูข้อมูลแบบตารางสวยๆ พร้อมกราฟและ diagram — pgAdmin หรือ DBeaver ตอบโจทย์กว่า
- บางครั้งกำลังเขียนโค้ด และไม่อยากสลับหน้าต่างไปมา — VS Code extension ช่วยได้
- บางครั้งต้องเขียน automation script — psql แบบ non-interactive คือคำตอบ

เครื่องมือแต่ละตัวไม่ได้แข่งกัน แต่เสริมกัน นักพัฒนามืออาชีพมักใช้ทั้งสี่แบบสลับกันไปตามงาน

---

## Step 21: psql คืออะไร — การเข้าและออกจาก psql, basic syntax

### psql คืออะไร

`psql` คือ interactive terminal-based client ที่มาพร้อมกับ PostgreSQL ตั้งแต่ติดตั้ง ไม่ต้องลงโปรแกรมเพิ่ม เป็นเครื่องมือมาตรฐานที่ DBA และนักพัฒนาทุกคนควรใช้เป็นอย่างน้อยที่สุด เพราะ:

- มีอยู่แล้วแทบทุกที่ที่มี PostgreSQL server (โดยเฉพาะบนเซิร์ฟเวอร์ Linux ที่ไม่มี GUI)
- เบา เร็ว ไม่กิน resource
- รองรับการเขียน script และ automation ได้ดีเยี่ยม
- เป็นเครื่องมือที่ "ตรงไปตรงมาที่สุด" — ไม่มีชั้น abstraction ระหว่างคุณกับฐานข้อมูล

### การเข้าสู่ psql

รูปแบบพื้นฐานของคำสั่งเชื่อมต่อ:

```bash
psql -h <host> -p <port> -U <username> -d <database>
```

ตัวอย่างเช่น เชื่อมต่อไปยังฐานข้อมูล `mydb` บนเครื่อง localhost ด้วย user `postgres`:

```bash
$ psql -h localhost -p 5432 -U postgres -d mydb
Password for user postgres:
psql (16.4)
Type "help" for help.

mydb=#
```

ถ้าเป็นการเชื่อมต่อไปยังเครื่อง local ของตัวเอง สามารถย่อได้ เพราะค่า default ของ `-h` คือ `localhost`, `-p` คือ `5432`, `-U` คือ current OS user:

```bash
# เชื่อมต่อฐานข้อมูลชื่อเดียวกับ OS user ปัจจุบัน
$ psql

# ระบุแค่ชื่อฐานข้อมูล
$ psql mydb

# ระบุ user และ database
$ psql -U postgres -d mydb
```

ตารางสรุป flag ที่ใช้บ่อย:

| Flag | ความหมาย | ค่า default |
|---|---|---|
| `-h` / `--host` | โฮสต์ปลายทาง | `localhost` (หรือ Unix socket) |
| `-p` / `--port` | พอร์ต | `5432` |
| `-U` / `--username` | ชื่อผู้ใช้ | OS user ปัจจุบัน |
| `-d` / `--dbname` | ชื่อฐานข้อมูล | เท่ากับชื่อ user |
| `-W` | บังคับให้ถามรหัสผ่าน | - |
| `-f` | รันไฟล์ SQL แล้วออก | - |
| `-c` | รันคำสั่งเดียวแล้วออก | - |
| `-l` | list ฐานข้อมูลทั้งหมดแล้วออก | - |

ตัวอย่างการรันคำสั่งเดียวโดยไม่เข้าสู่ interactive mode (มีประโยชน์มากสำหรับ script):

```bash
$ psql -U postgres -d mydb -c "SELECT current_database(), current_user;"
 current_database | current_user
------------------+---------------
 mydb             | postgres
(1 row)
```

### การอ่าน prompt ของ psql

เมื่อเชื่อมต่อสำเร็จ prompt จะเปลี่ยนไปตามสถานะ:

```
mydb=#     -- เชื่อมต่อสำเร็จ, เป็น superuser, พร้อมรับคำสั่งใหม่
mydb=>     -- เชื่อมต่อสำเร็จ, เป็น user ธรรมดา, พร้อมรับคำสั่งใหม่
mydb-#     -- อยู่กลางการพิมพ์คำสั่งที่ยังไม่จบด้วย ;
mydb=*#    -- อยู่ใน transaction ที่ยังไม่ commit (หลัง BEGIN)
mydb=!#    -- transaction ล้มเหลว (failed transaction block)
```

ตัวอย่างที่เห็นเครื่องหมาย `-#` เพราะลืมใส่ `;`:

```
mydb=# SELECT *
mydb-# FROM customers
mydb-# WHERE id = 1;
```

### Basic syntax และการจบคำสั่ง

psql มีสองประเภทของสิ่งที่พิมพ์ได้:

1. **SQL commands** — ต้องจบด้วย semicolon `;` เสมอ ไม่งั้น psql จะรอบรรทัดถัดไปเรื่อยๆ
2. **psql meta-commands** — ขึ้นต้นด้วย backslash `\` ไม่ต้องมี semicolon

```
mydb=# SELECT version();
                                                version
---------------------------------------------------------------------------------------------------
 PostgreSQL 16.4 on x86_64-pc-linux-gnu, compiled by gcc (GCC) 12.2.0, 64-bit
(1 row)

mydb=# \conninfo
You are connected to database "mydb" as user "postgres" on host "localhost" (address "127.0.0.1") at port "5432".
```

### การออกจาก psql

ใช้ meta-command `\q` (quit) หรือกด `Ctrl+D`:

```
mydb=# \q
$
```

> **เกร็ดความรู้**: ถ้าพิมพ์คำสั่ง SQL ผิด แล้ว psql ค้างอยู่ที่ `mydb-#` รอ semicolon ไม่รู้จะจบยังไง ให้พิมพ์ `;` เปล่าๆ แล้ว Enter หรือกด `Ctrl+C` เพื่อยกเลิกบรรทัดที่พิมพ์ค้างไว้

### ทดสอบด้วยตัวเอง

```bash
$ psql -U postgres -l
                                  List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    | Access privileges
-----------+----------+----------+-------------+-------------+-------------------
 mydb      | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
 postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
 template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres      +
           |          |          |             |             | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres      +
           |          |          |             |             | postgres=CTc/postgres
(4 rows)
```

คำสั่งนี้มีประโยชน์มากตอนเริ่มงานใหม่ — ใช้ตรวจสอบว่าเชื่อมต่อ server ได้ถูกต้องหรือไม่ โดยไม่ต้องเข้าไปใน interactive session เลย

---

## Step 22: psql meta-commands ที่สำคัญ

Meta-commands คือคำสั่งพิเศษของ psql เอง (ไม่ใช่ SQL) ใช้สำหรับสำรวจโครงสร้างฐานข้อมูลอย่างรวดเร็ว ทุกคำสั่งขึ้นต้นด้วย backslash และไม่ต้องมี semicolon

### `\l` — list databases

```
mydb=# \l
                                  List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    | Access privileges
-----------+----------+----------+-------------+-------------+-------------------
 mydb      | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
 postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
 template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres      +
           |          |          |             |             | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres      +
           |          |          |             |             | postgres=CTc/postgres
(4 rows)
```

เพิ่ม `+` ต่อท้ายคำสั่งได้เกือบทุกตัว เพื่อขอข้อมูล "verbose" มากขึ้น เช่น ขนาดของแต่ละฐานข้อมูล:

```
mydb=# \l+
```

### `\c` — connect (เปลี่ยนฐานข้อมูลหรือ user)

```
mydb=# \c postgres
You are now connected to database "postgres" as user "postgres".
postgres=# \c mydb appuser
You are now connected to database "mydb" as user "appuser".
mydb=>
```

สังเกตว่า prompt เปลี่ยนจาก `=#` เป็น `=>` เพราะ `appuser` ไม่ใช่ superuser

### `\dt` — list tables

```
mydb=# \dt
             List of relations
 Schema |     Name     | Type  |  Owner
--------+--------------+-------+----------
 public | customers    | table | postgres
 public | orders       | table | postgres
 public | order_items  | table | postgres
 public | products     | table | postgres
(4 rows)
```

ถ้าต้องการดูขนาดตารางและ description ด้วย ใช้ `\dt+`:

```
mydb=# \dt+
                                     List of relations
 Schema |     Name     | Type  |  Owner   | Persistence | Access method |    Size    | Description
--------+--------------+-------+----------+-------------+---------------+------------+-------------
 public | customers    | table | postgres | permanent   | heap          | 128 kB     |
 public | orders       | table | postgres | permanent   | heap          | 256 kB     |
 public | order_items  | table | postgres | permanent   | heap          | 512 kB     |
 public | products     | table | postgres | permanent   | heap          | 96 kB      |
(4 rows)
```

ดูตารางเฉพาะที่ขึ้นต้นด้วยคำใดคำหนึ่งได้โดยใช้ pattern matching:

```
mydb=# \dt order*
           List of relations
 Schema |    Name     | Type  |  Owner
--------+-------------+-------+----------
 public | orders      | table | postgres
 public | order_items | table | postgres
(2 rows)
```

### `\d` — describe object (โครงสร้างตาราง/index/view)

`\d` เพียวๆ (ไม่มี argument) จะแสดงรายการ object ทั้งหมดในลักษณะเดียวกับ `\dt` แต่รวม view, sequence, ฯลฯ ด้วย ส่วน `\d <table_name>` จะแสดงโครงสร้างของตารางนั้นแบบละเอียด:

```
mydb=# \d customers
                                        Table "public.customers"
   Column   |          Type          | Collation | Nullable |               Default
------------+-------------------------+-----------+----------+---------------------------------------
 id         | integer                 |           | not null | nextval('customers_id_seq'::regclass)
 name       | character varying(100) |           | not null |
 email      | character varying(255) |           | not null |
 created_at | timestamp with time zone |          | not null | now()
Indexes:
    "customers_pkey" PRIMARY KEY, btree (id)
    "customers_email_key" UNIQUE CONSTRAINT, btree (email)
Referenced by:
    TABLE "orders" CONSTRAINT "orders_customer_id_fkey" FOREIGN KEY (customer_id) REFERENCES customers(id)
```

`\d+` เพิ่มข้อมูล storage type, statistics target และ description:

```
mydb=# \d+ customers
```

### `\du` — list roles (users)

```
mydb=# \du
                                    List of roles
 Role name |                         Attributes                         | Member of
-----------+-------------------------------------------------------------+-----------
 appuser   |                                                             | {}
 postgres  | Superuser, Create role, Create DB, Replication, Bypass RLS | {}
 readonly  | Cannot login                                                | {}
```

### `\dn` — list schemas

```
mydb=# \dn
  List of schemas
  Name  |  Owner
--------+----------
 public | postgres
 sales  | postgres
(2 rows)
```

### `\x` — expanded display (สลับโหมดแสดงผล)

เมื่อ column เยอะหรือข้อมูลยาว ตารางแบบปกติจะดูยากมาก `\x` ช่วยให้แสดงผลทีละแถว แนวตั้งแทน:

```
mydb=# \x
Expanded display is on.
mydb=# SELECT * FROM customers WHERE id = 1;
-[ RECORD 1 ]----------------------
id         | 1
name       | สมชาย ใจดี
email      | somchai@example.com
created_at | 2026-01-15 10:30:00+07
```

พิมพ์ `\x` อีกครั้งเพื่อปิดโหมดนี้ หรือใช้ `\x auto` ให้ psql เลือกอัตโนมัติตามความกว้างของ terminal

### `\timing` — วัดเวลาในการรัน query

```
mydb=# \timing
Timing is on.
mydb=# SELECT count(*) FROM orders;
 count
-------
 15420
(1 row)

Time: 12.481 ms
```

มีประโยชน์มากตอน debug performance เบื้องต้น ก่อนจะไปดู `EXPLAIN ANALYZE` แบบละเอียด (จะเรียนใน Part ถัดๆ ไปเรื่อง query optimization)

### `\q` — ออกจาก psql

ตามที่กล่าวไปใน Step 21

### ตารางสรุป meta-commands สำคัญที่ควรจำขึ้นใจ

| Meta-command | ความหมาย |
|---|---|
| `\l` | list ฐานข้อมูลทั้งหมด |
| `\c <db> [user]` | เปลี่ยนไปเชื่อมต่อฐานข้อมูล/user อื่น |
| `\dt` | list ตารางใน schema ปัจจุบัน (search_path) |
| `\d <name>` | ดูโครงสร้างของ table/view/index |
| `\du` | list roles/users |
| `\dn` | list schemas |
| `\dv` | list views |
| `\di` | list indexes |
| `\df` | list functions |
| `\x` | สลับโหมด expanded display |
| `\timing` | เปิด/ปิดการแสดงเวลาในการรัน query |
| `\q` | ออกจาก psql |
| `\?` | ดูรายการ meta-command ทั้งหมด |
| `\h <SQL command>` | ดู syntax help ของคำสั่ง SQL |

> **เคล็ดลับ**: พิมพ์ `\?` เมื่อไหร่ก็ได้ในระหว่างใช้งาน psql เพื่อดูรายการ meta-command ทั้งหมดพร้อมคำอธิบายสั้นๆ ไม่ต้องเปิดเบราว์เซอร์ค้นหา

---

## Step 23: การรันไฟล์ .sql ด้วย psql (`\i`) และการ export ผลลัพธ์ (`\o`, `\copy`)

### การรันไฟล์ SQL ด้วย `\i`

เมื่อมีชุดคำสั่ง SQL ที่ต้องรันซ้ำๆ (เช่น script สร้างตาราง, seed data) การพิมพ์ทีละบรรทัดไม่สะดวก เราเก็บไว้ในไฟล์ `.sql` แล้วสั่งรันด้วย `\i` (include)

สมมติมีไฟล์ `setup.sql`:

```sql
-- setup.sql
CREATE TABLE IF NOT EXISTS products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    price NUMERIC(10,2) NOT NULL
);

INSERT INTO products (name, price) VALUES
    ('Keyboard', 890.00),
    ('Mouse', 350.00),
    ('Monitor', 5900.00);
```

รันจากภายใน psql session:

```
mydb=# \i setup.sql
CREATE TABLE
INSERT 0 3
mydb=#
```

หรือรันจาก shell โดยตรงโดยไม่ต้องเข้า psql interactive mode เลย:

```bash
$ psql -U postgres -d mydb -f setup.sql
CREATE TABLE
INSERT 0 3
```

ทั้งสองวิธีให้ผลเหมือนกัน ต่างกันแค่ว่า `\i` ใช้เมื่ออยู่ใน session อยู่แล้ว ส่วน `-f` ใช้เรียกจาก shell/script

> **หมายเหตุ**: มี meta-command พี่น้องกันคือ `\ir` ซึ่งเหมือน `\i` แต่ resolve path แบบ relative จากตำแหน่งของไฟล์ script ที่กำลังรันอยู่ มีประโยชน์เวลาเขียน script ที่ include ไฟล์อื่นซ้อนกันหลายชั้น

### การ export ผลลัพธ์ด้วย `\o`

`\o` ใช้เปลี่ยนปลายทางของ output จากหน้าจอไปเป็นไฟล์ ทุกอย่างที่ psql จะพิมพ์ (รวมผลลัพธ์ query) จะถูกเขียนลงไฟล์แทน

```
mydb=# \o /tmp/report.txt
mydb=# SELECT name, price FROM products ORDER BY price DESC;
mydb=# \o
mydb=# \! cat /tmp/report.txt
    name    | price
------------+---------
 Monitor    | 5900.00
 Keyboard   |  890.00
 Mouse      |  350.00
(3 rows)
```

สังเกตว่าเมื่อพิมพ์ `\o` ครั้งที่สองโดยไม่ใส่ argument จะเป็นการ "ปิด" การ redirect กลับมาแสดงผลบนหน้าจอตามเดิม และ `\!` คือการรันคำสั่ง shell จากภายใน psql

### `\copy` — export/import ข้อมูลแบบยืดหยุ่น

`\copy` เป็น client-side version ของคำสั่ง SQL `COPY` ต่างจาก `COPY` ตรงที่ `\copy` รันฝั่ง client (เครื่องที่รัน psql) จึงไม่ต้องมีสิทธิ์ superuser และไฟล์ปลายทางอยู่บนเครื่อง client ไม่ใช่เครื่อง server

**Export ข้อมูลเป็น CSV:**

```
mydb=# \copy (SELECT * FROM products ORDER BY id) TO '/tmp/products.csv' WITH (FORMAT csv, HEADER true)
COPY 3
```

ตรวจสอบไฟล์ที่ได้:

```bash
$ cat /tmp/products.csv
id,name,price
1,Keyboard,890.00
2,Mouse,350.00
3,Monitor,5900.00
```

**Import ข้อมูลจาก CSV กลับเข้าตาราง:**

```
mydb=# \copy products(name, price) FROM '/tmp/new_products.csv' WITH (FORMAT csv, HEADER true)
COPY 25
```

**Export ทั้งตารางแบบง่ายๆ ไม่ต้องเขียน query:**

```
mydb=# \copy products TO '/tmp/products_full.csv' CSV HEADER
COPY 3
```

### เปรียบเทียบ `COPY` กับ `\copy`

| ประเด็น | `COPY` (SQL command) | `\copy` (psql meta-command) |
|---|---|---|
| รันที่ไหน | ฝั่ง server | ฝั่ง client |
| ไฟล์อยู่ที่ไหน | บนเครื่อง server | บนเครื่อง client (เครื่องที่รัน psql) |
| ต้องการสิทธิ์ | ปกติต้องเป็น superuser หรือมี role `pg_read_server_files`/`pg_write_server_files` | ใช้สิทธิ์ของ user ปกติที่เชื่อมต่อได้ |
| ใช้งานเมื่อ | server และ client เป็นเครื่องเดียวกัน หรือมีสิทธิ์เข้าถึงไฟล์ระบบของ server | ทำงานจากเครื่องตัวเองเชื่อมต่อ remote server (กรณีนี้พบบ่อยที่สุด) |

ในการทำงานจริง `\copy` คือคำสั่งที่ใช้บ่อยกว่ามาก เพราะส่วนใหญ่เราเชื่อมต่อไปยัง remote database server และไม่มีสิทธิ์เขียนไฟล์บนเครื่อง server นั้น

### ตัวอย่างการรวม `\i` และ `\o` ใน workflow จริง

```
mydb=# \o /tmp/daily_report.txt
mydb=# \i /home/user/scripts/daily_summary.sql
mydb=# \o
```

Workflow แบบนี้ใช้สร้างรายงานอัตโนมัติได้ง่ายๆ โดยไม่ต้องเขียนโปรแกรมภายนอกเลย

---

## Step 24: การปรับแต่ง psql ด้วย `.psqlrc` และ prompt customization

### `.psqlrc` คืออะไร

`.psqlrc` เป็นไฟล์ configuration ที่ psql จะอ่านทุกครั้งที่เริ่ม session (ยกเว้นเรียกด้วย flag `-X`) วางไว้ที่ home directory ของ user:

```bash
~/.psqlrc          # Linux / macOS
%APPDATA%\postgresql\psqlrc.conf   # Windows
```

ไฟล์นี้ใส่ได้ทั้ง meta-commands และ SQL commands ที่ต้องการให้รันอัตโนมัติทุกครั้งที่เข้า psql

### ตัวอย่าง `.psqlrc` พื้นฐานที่มีประโยชน์

```sql
-- ~/.psqlrc

-- เปิดการวัดเวลา query อัตโนมัติ
\timing on

-- แสดง NULL เป็นคำว่า [NULL] แทนที่จะเป็นช่องว่าง จะได้แยกออกจาก empty string
\pset null '[NULL]'

-- จำกัดจำนวนแถวที่แสดงผ่าน pager (less) ถ้าผลลัพธ์เยอะ
\pset pager always

-- เปิด autocommit (ค่า default อยู่แล้ว แต่ระบุชัดเจนไว้)
\set AUTOCOMMIT on

-- เก็บ history ของคำสั่งไว้แยกตามฐานข้อมูล
\set HISTFILE ~/.psql_history- :DBNAME
\set HISTSIZE 2000
\set HISTCONTROL ignoredups

-- เตือนก่อนรัน DELETE/UPDATE โดยไม่มี WHERE (ป้องกันความผิดพลาด)
\set ON_ERROR_ROLLBACK interactive

-- ตั้งค่า editor ที่ใช้เปิดตอนพิมพ์ \e
\setenv EDITOR vim

-- ข้อความต้อนรับ
\echo 'ยินดีต้อนรับเข้าสู่ psql :DBNAME (connected as :USER)'
```

หลังจากบันทึกไฟล์แล้ว ลองเปิด psql ใหม่:

```bash
$ psql -U postgres -d mydb
ยินดีต้อนรับเข้าสู่ psql mydb (connected as postgres)
psql (16.4)
Type "help" for help.

Timing is on.
mydb=#
```

### การปรับแต่ง prompt ด้วย `PROMPT1` / `PROMPT2` / `PROMPT3`

psql มี prompt สามระดับ:

- `PROMPT1` — prompt ปกติเมื่อพร้อมรับคำสั่งใหม่
- `PROMPT2` — prompt เมื่อกำลังพิมพ์คำสั่งที่ยังไม่จบ (รอ `;`)
- `PROMPT3` — prompt ระหว่างกระบวนการ `COPY ... FROM STDIN`

ตัวแปรพิเศษที่ใช้ในการประกอบ prompt ได้แก่:

| ตัวแปร | ความหมาย |
|---|---|
| `%n` | ชื่อ session user |
| `%/` | ชื่อฐานข้อมูลปัจจุบัน |
| `%~` | ชื่อฐานข้อมูล (แบบย่อ แสดง `~` ถ้าตรงกับ user) |
| `%R` | สัญลักษณ์บอกสถานะ (เช่น `#` สำหรับ superuser, `>` สำหรับ user ปกติ) |
| `%m` | hostname (แบบสั้น) |
| `%>` | หมายเลขพอร์ต |
| `%[...%]` | ห่อ ANSI color code เพื่อไม่ให้ psql นับความยาวผิด |

ตัวอย่างการตั้งค่า prompt ให้แสดง user, host, และชื่อฐานข้อมูลพร้อมสีสัน ใส่ใน `.psqlrc`:

```sql
-- Prompt แบบมีสี: เขียว = user, ฟ้า = database, แดง = # สำหรับ superuser
\set PROMPT1 '%[%033[1;32m%]%n%[%033[0m%]@%[%033[1;34m%]%~%[%033[0m%]%R%# '
\set PROMPT2 '%[%033[1;33m%]-> %[%033[0m%]'
```

ผลลัพธ์ที่ได้จะหน้าตาประมาณนี้ (สีจะแสดงเฉพาะใน terminal จริง):

```
postgres@mydb=# SELECT 1
-> ;
```

ตัวอย่างที่ใช้งานได้จริงและเรียบง่ายกว่า สำหรับคนที่อยากเห็นว่าตัวเองต่อ production หรือ dev อยู่ (ป้องกันความผิดพลาดร้ายแรง):

```sql
-- แสดงชื่อ host เด่นชัด เตือนเมื่อต่อ production
\set PROMPT1 '[%m] %n@%/%R%# '
```

```
[prod-db-01] postgres@mydb=#
```

การเห็นชื่อ host ชัดเจนแบบนี้ช่วยลดความเสี่ยงจากการรัน `DROP TABLE` หรือ `DELETE` ผิดเครื่องได้มาก ซึ่งเป็นความผิดพลาดที่เกิดขึ้นบ่อยกับมือใหม่ (และบางครั้งกับมือเก๋าด้วย!)

### คำสั่ง `\set` และ `\pset` ที่มีประโยชน์อื่นๆ

```sql
-- แสดงผลลัพธ์เป็นตาราง unicode สวยงามแทน ASCII ธรรมดา
\pset border 2
\pset linestyle unicode

-- จำกัด format ของตัวเลข ทศนิยมให้อ่านง่าย
\pset numericlocale on
```

### เพิ่มเติม: `.psql_history`

psql เก็บประวัติคำสั่งไว้ที่ `~/.psql_history` โดย default สามารถเรียกดูย้อนหลังด้วยปุ่มลูกศรขึ้น-ลง หรือค้นหาแบบ reverse search ด้วย `Ctrl+R` เหมือนใน bash shell ทั่วไป และถ้าตั้งค่า `HISTFILE` แยกตาม `:DBNAME` ตามตัวอย่างข้างต้น จะช่วยไม่ให้ประวัติคำสั่งของฐานข้อมูลต่างๆ ปนกัน

---

## Step 25: pgAdmin 4 — ติดตั้งและตั้งค่าเชื่อมต่อ server ครั้งแรก, tour ของ UI

### pgAdmin 4 คืออะไร

pgAdmin 4 คือ GUI (Graphical User Interface) client อย่างเป็นทางการสำหรับ PostgreSQL พัฒนาและดูแลโดยชุมชนที่ใกล้ชิดกับทีม PostgreSQL core มากที่สุด รองรับการทำงานได้ทั้งในรูปแบบ desktop application และ web application (deploy เป็น container ให้ทีมใช้งานร่วมกันผ่านเบราว์เซอร์ได้)

### การติดตั้ง

**Windows / macOS:** ดาวน์โหลด installer จาก `https://www.pgadmin.org/download/` แล้วติดตั้งตามขั้นตอนปกติ (Next > Next > Finish) เมื่อเปิดครั้งแรกจะขอให้ตั้ง master password สำหรับเข้ารหัสข้อมูล connection ที่บันทึกไว้

**Linux (Ubuntu/Debian) ผ่าน apt:**

```bash
# เพิ่ม repository ของ pgAdmin
curl -fsS https://www.pgadmin.org/static/packages_pgadmin_org.pub | sudo gpg --dearmor -o /usr/share/keyrings/packages-pgadmin-org.gpg
sudo sh -c 'echo "deb [signed-by=/usr/share/keyrings/packages-pgadmin-org.gpg] https://ftp.postgresql.org/pub/pgadmin/pgadmin4/apt/$(lsb_release -cs) pgadmin4 main" > /etc/apt/sources.list.d/pgadmin4.list'
sudo apt update

# ติดตั้งแบบ desktop mode
sudo apt install pgadmin4-desktop

# หรือแบบ web mode (สำหรับใช้งานผ่านเบราว์เซอร์บนเครื่อง server)
sudo apt install pgadmin4-web
sudo /usr/pgadmin4/bin/setup-web.sh
```

**รันผ่าน Docker (สะดวกที่สุดสำหรับทดลองใช้ชั่วคราว):**

```bash
docker run -p 8080:80 \
  -e 'PGADMIN_DEFAULT_EMAIL=admin@example.com' \
  -e 'PGADMIN_DEFAULT_PASSWORD=SecurePass123' \
  -d dpage/pgadmin4
```

จากนั้นเปิดเบราว์เซอร์ไปที่ `http://localhost:8080`

### การตั้งค่าเชื่อมต่อ server ครั้งแรก

เมื่อเปิด pgAdmin 4 ขึ้นมาครั้งแรก จะเห็นหน้าจอหลักที่มีแถบซ้ายชื่อ **Browser** ว่างเปล่า (ยังไม่มี server ใดๆ) ทำตามขั้นตอนนี้:

1. คลิกขวาที่ **Servers** ในแถบ Browser ด้านซ้าย > เลือก **Register > Server...**
2. หน้าต่าง "Register - Server" จะเปิดขึ้น มีสองแท็บหลักที่ต้องกรอก:
   - แท็บ **General**: ใส่ **Name** เป็นชื่อที่อยากเรียก connection นี้ (เช่น `Local Dev`, `Production - Tokyo`) — ชื่อนี้เป็นแค่ label ไม่กระทบการเชื่อมต่อจริง
   - แท็บ **Connection**: กรอกข้อมูลจริงของ server
     - **Host name/address**: `localhost` หรือ IP/hostname ของ server
     - **Port**: `5432`
     - **Maintenance database**: `postgres` (ฐานข้อมูล default สำหรับเชื่อมต่อครั้งแรก)
     - **Username**: `postgres` หรือ user ที่ต้องการ
     - **Password**: รหัสผ่าน (มี checkbox "Save password?" ให้ pgAdmin จำรหัสผ่านไว้)
3. กด **Save**

หากเชื่อมต่อสำเร็จ จะเห็น server ปรากฏใน Browser panel พร้อมไอคอนช้างสีฟ้า (โลโก้ PostgreSQL) และเมื่อขยาย (คลิกลูกศรข้างหน้า) จะเห็นโครงสร้างแบบ tree:

```
Servers
└── Local Dev
    └── Databases
        ├── mydb
        │   ├── Schemas
        │   │   └── public
        │   │       ├── Tables
        │   │       ├── Views
        │   │       ├── Functions
        │   │       └── Sequences
        │   └── Extensions
        └── postgres
    └── Login/Group Roles
    └── Tablespaces
```

### Tour ของ UI หลัก

pgAdmin 4 แบ่งพื้นที่หน้าจอออกเป็นส่วนหลักๆ ดังนี้:

1. **Browser panel (ซ้ายมือ)** — แสดง object tree ของทุก server/database/schema/table ที่เชื่อมต่อไว้ คลิกขวาที่ object ใดๆ จะมีเมนู context เช่น "Create", "Properties", "Delete/Drop", "View/Edit Data"

2. **Dashboard tab** — เมื่อคลิกเลือก server หรือ database จะเห็นแท็บ Dashboard แสดงกราฟ real-time ของ:
   - Server activity (จำนวน sessions, transactions per second)
   - Database sessions (active/idle connections)
   - Locks ที่กำลังเกิดขึ้น
   - Query ที่กำลังรันอยู่ (สามารถดูและ cancel ได้จากตรงนี้)

3. **Properties tab** — เมื่อเลือก object ใน tree จะแสดงรายละเอียดของ object นั้น เช่น columns, constraints, indexes ของตาราง

4. **Query Tool** — เปิดผ่านเมนู Tools > Query Tool หรือคลิกไอคอนรูปฟ้าผ่า พื้นที่สำหรับเขียนและรัน SQL (รายละเอียดใน Step 26)

5. **Statistics / SQL / Dependencies tab** — แสดงสถิติการใช้งานของ object, SQL ที่ใช้สร้าง object นั้น (DDL), และ dependency ระหว่าง object

6. **Preferences (File > Preferences)** — ปรับแต่งได้ละเอียดมาก เช่น theme (light/dark), ขนาด font ใน Query Tool, การแสดงผลตัวเลข, keyboard shortcuts

### ตัวอย่างการสำรวจข้อมูลผ่าน UI แบบไม่ต้องเขียน SQL

1. ขยาย tree ไปที่ `Servers > Local Dev > Databases > mydb > Schemas > public > Tables`
2. คลิกขวาที่ตาราง `customers` > เลือก **View/Edit Data > All Rows**
3. จะเห็นข้อมูลแสดงเป็นตาราง แก้ไขค่าในเซลล์ได้โดยตรง (double-click) แล้วกด disk icon เพื่อ save การเปลี่ยนแปลงกลับเข้าฐานข้อมูล — เหมาะสำหรับแก้ข้อมูลเล็กๆ น้อยๆ อย่างรวดเร็วโดยไม่ต้องเขียน `UPDATE` เอง

> **ข้อควรระวัง**: การแก้ไขข้อมูลผ่าน grid ของ pgAdmin โดยตรงสะดวกก็จริง แต่ไม่มี audit trail หรือ confirmation แบบเดียวกับการรัน SQL ที่เขียนไว้ชัดเจน จึงควรใช้เฉพาะฐานข้อมูล dev/test หรือกรณีจำเป็นจริงๆ เท่านั้น

---

## Step 26: pgAdmin 4 — Query Tool, ER Diagram tool, และการดู execution plan แบบ visual

### การเปิดและใช้งาน Query Tool

เปิด Query Tool ได้สามวิธี:

- คลิกขวาที่ฐานข้อมูล (หรือ schema) > **Query Tool**
- คลิกไอคอนรูปฟ้าผ่าบน toolbar เมื่อเลือกฐานข้อมูลไว้แล้ว
- กด keyboard shortcut `Alt+Shift+Q` (Windows/Linux) หรือ `Option+Shift+Q` (macOS)

หน้าต่าง Query Tool แบ่งเป็นสองส่วนหลัก: **editor ด้านบน** สำหรับพิมพ์ SQL และ **panel ผลลัพธ์ด้านล่าง** ที่จะแสดง data grid, messages, หรือ execution plan สลับกันตามแท็บ

ตัวอย่างการรัน query:

```sql
SELECT c.name, COUNT(o.id) AS total_orders, SUM(o.total_amount) AS total_spent
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id
GROUP BY c.name
ORDER BY total_spent DESC NULLS LAST
LIMIT 10;
```

พิมพ์ query แล้วกด **F5** (หรือปุ่มสามเหลี่ยม Play สีเขียวบน toolbar) เพื่อรันทั้ง statement หรือกด **F5** ขณะ cursor อยู่ในเฉพาะ statement เดียวเพื่อรันแค่ statement นั้นถ้ามีหลาย statement คั่นด้วย `;`

ฟีเจอร์ที่มีประโยชน์ใน Query Tool:

- **Auto-complete**: พิมพ์ชื่อตาราง/คอลัมน์แล้วกด `Ctrl+Space` เพื่อขอคำแนะนำ
- **Format SQL** (ไอคอนรูปไม้บรรทัด): จัดรูปแบบ SQL ให้อ่านง่ายอัตโนมัติ
- **History panel**: แท็บด้านล่างเก็บประวัติ query ที่เคยรันในเซสชันนี้ พร้อมเวลาที่ใช้
- **Explain options**: ปุ่มลูกศรข้าง Explain ให้เลือกว่าจะ `EXPLAIN`, `EXPLAIN ANALYZE` และเปิด `BUFFERS`/`COSTS`/`VERBOSE` เพิ่มได้
- **Download as CSV**: บน result grid มีปุ่ม download ไอคอนลูกศรชี้ลง ให้ export ผลลัพธ์เป็นไฟล์ CSV ได้ทันที

### ER Diagram Tool

pgAdmin 4 มีเครื่องมือสร้าง Entity-Relationship diagram อัตโนมัติจากโครงสร้างฐานข้อมูลจริง วิธีเปิดใช้งาน:

1. คลิกขวาที่ฐานข้อมูล (เช่น `mydb`) ใน Browser panel
2. เลือก **Generate ER Diagram**
3. pgAdmin จะสร้าง canvas แสดงตารางทั้งหมดในฐานข้อมูล พร้อมเส้นเชื่อมแสดงความสัมพันธ์ foreign key ระหว่างตาราง โดยอัตโนมัติ

ใน ER Diagram canvas สามารถ:

- ลากตารางไปวางตำแหน่งใหม่เพื่อจัดผังให้อ่านง่าย
- คลิกที่ตารางเพื่อดู/แก้ไข columns โดยตรงบน diagram
- เพิ่มตารางใหม่ วาด relationship ใหม่ แล้วให้ pgAdmin generate SQL DDL ให้อัตโนมัติ (ปุ่ม **Generate SQL**)
- Export diagram เป็นไฟล์ภาพ PNG หรือบันทึกเป็นไฟล์ `.pgerd` เพื่อแก้ไขต่อภายหลัง

ER Diagram tool มีประโยชน์มากในการ:
- ทำความเข้าใจโครงสร้างฐานข้อมูลที่คนอื่นสร้างไว้ (เช่น เข้าโปรเจกต์ใหม่)
- นำเสนอโครงสร้างฐานข้อมูลให้ทีมหรือผู้บริหารเข้าใจง่าย
- ออกแบบฐานข้อมูลใหม่แบบ visual ก่อนเขียน DDL จริง

### การดู Execution Plan แบบ Visual

เมื่อรัน query ใน Query Tool แล้วต้องการดูว่า PostgreSQL planner เลือกวิธีประมวลผลอย่างไร ให้ใช้ปุ่ม **Explain** (ไอคอนรูปเข็มทิศ) แทนการกด F5 ปกติ

ขั้นตอน:

1. พิมพ์ query ที่ต้องการวิเคราะห์
2. คลิกลูกศรข้างปุ่ม Explain > เลือก **Analyze** (เพื่อให้รันจริงและวัดเวลาจริง ไม่ใช่แค่ประมาณการ) และเปิด **Buffers** ด้วยถ้าต้องการดูสถิติการอ่าน I/O
3. กดปุ่ม Explain (หรือ `F7`)
4. ผลลัพธ์จะแสดงเป็นแท็บ **Explain** ในรูปแบบ **กราฟ diagram** ที่มีไอคอนแทนแต่ละ operation (Seq Scan, Index Scan, Nested Loop, Hash Join, Sort ฯลฯ) เชื่อมต่อกันเป็นต้นไม้ (tree) จากล่างขึ้นบน

จุดเด่นของ visual plan:

- ขนาดของเส้นเชื่อม (edge) ระหว่าง node จะหนา-บางตามจำนวนแถวที่ไหลผ่าน ทำให้มองเห็นได้ทันทีว่า operation ไหน "หนัก" ที่สุด
- วางเมาส์ (hover) ที่แต่ละ node จะแสดง tooltip รายละเอียด เช่น `cost`, `actual time`, `rows`, `loops`, `Filter`
- มีแท็บ **Statistics** ควบคู่กันแสดงตัวเลขดิบในรูปตาราง เรียงตาม self-time เพื่อหา bottleneck ได้เร็ว
- แท็บ **Explain (text)** ยังคงมีให้ดู raw output แบบข้อความเหมือน `EXPLAIN ANALYZE` ปกติสำหรับคนที่ถนัดอ่านแบบข้อความ

ตัวอย่าง raw text ที่ปรากฏคู่กับ diagram (ย่อ):

```
Sort  (cost=245.32..247.87 rows=1020 width=48) (actual time=3.201..3.245 rows=1020 loops=1)
  Sort Key: total_spent DESC NULLS LAST
  ->  Hash Left Join  (cost=15.00..190.45 rows=1020 width=48) (actual time=0.512..2.890 rows=1020 loops=1)
        Hash Cond: (o.customer_id = c.id)
        ->  Seq Scan on orders o  (cost=0.00..150.20 rows=5020 width=16) (actual time=0.010..1.203 rows=5020 loops=1)
        ->  Hash  (cost=12.50..12.50 rows=200 width=36) (actual time=0.480..0.481 rows=200 loops=1)
              ->  Seq Scan on customers c  (cost=0.00..12.50 rows=200 width=36) (actual time=0.005..0.201 rows=200 loops=1)
Planning Time: 0.412 ms
Execution Time: 3.501 ms
```

การอ่าน plan อย่างละเอียด (การเลือก index, cost-based optimizer ฯลฯ) จะเป็นหัวข้อเจาะลึกในบทที่ว่าด้วย Query Optimization ในระดับ intermediate/advanced ของหลักสูตรนี้ ในบทนี้ให้เข้าใจแค่ว่า pgAdmin ช่วยให้การ "มองเห็น" plan ทำได้ง่ายกว่าการอ่านข้อความล้วนมาก

---

## Step 27: DBeaver — ติดตั้งและเชื่อมต่อ PostgreSQL, ข้อดีเทียบกับ pgAdmin

### DBeaver คืออะไร

DBeaver เป็น universal database tool แบบ GUI ที่ไม่ได้ทำมาเพื่อ PostgreSQL อย่างเดียว แต่รองรับฐานข้อมูลได้หลากหลายชนิดมาก (MySQL, SQL Server, Oracle, SQLite, MongoDB, Redis ฯลฯ) ผ่านกลไก JDBC driver มีทั้งเวอร์ชัน **Community Edition (ฟรี, open source)** และ **Enterprise Edition (เสียเงิน มีฟีเจอร์เพิ่ม เช่น NoSQL, ER diagram ขั้นสูง, collaboration)**

### การติดตั้ง

**Windows/macOS/Linux:** ดาวน์โหลดจาก `https://dbeaver.io/download/` เลือก installer ตาม OS

**macOS ผ่าน Homebrew:**

```bash
brew install --cask dbeaver-community
```

**Linux ผ่าน Snap:**

```bash
sudo snap install dbeaver-ce
```

**Linux ผ่าน .deb (Ubuntu/Debian):**

```bash
wget -O dbeaver.deb https://dbeaver.io/files/dbeaver-ce_latest_amd64.deb
sudo dpkg -i dbeaver.deb
```

### การเชื่อมต่อ PostgreSQL ครั้งแรก

1. เปิด DBeaver ขึ้นมา ครั้งแรกจะเห็นหน้าจอ Database Navigator ว่างเปล่าทางซ้าย
2. คลิกไอคอน **New Database Connection** (ปลั๊กไฟฟ้าสีเขียว) หรือเมนู **Database > New Database Connection**
3. หน้าต่างเลือกชนิดฐานข้อมูลจะเปิดขึ้น (มีไอคอนให้เลือกเป็นสิบๆ ชนิด) เลือก **PostgreSQL** แล้วกด **Next**
4. กรอกข้อมูลการเชื่อมต่อในแท็บ **Main**:
   - **Host**: `localhost`
   - **Port**: `5432`
   - **Database**: `mydb`
   - **Username**: `postgres`
   - **Password**: กรอกรหัสผ่าน และติ๊ก **Save password**
5. กดปุ่ม **Test Connection...** ที่มุมล่างซ้าย — ถ้าเป็นครั้งแรกที่เชื่อมต่อ PostgreSQL DBeaver จะถามว่าต้องการ **download PostgreSQL JDBC driver** หรือไม่ ให้กด **Download** (ทำครั้งเดียว ใช้ได้ตลอดไป)
6. เมื่อ test สำเร็จจะเห็นข้อความ `Connected` พร้อมเวอร์ชัน PostgreSQL ที่เจอ
7. กด **Finish**

Connection จะปรากฏใน Database Navigator ทางซ้าย ขยาย tree ดูได้ทันที:

```
mydb
└── PostgreSQL
    └── Databases
        └── mydb
            └── Schemas
                └── public
                    ├── Tables
                    ├── Views
                    ├── Materialized Views
                    ├── Procedures
                    └── Sequences
```

### ข้อดีของ DBeaver เมื่อเทียบกับ pgAdmin

| ประเด็น | DBeaver | pgAdmin 4 |
|---|---|---|
| รองรับฐานข้อมูลชนิดอื่น | รองรับหลายสิบชนิด (MySQL, Oracle, SQL Server, MongoDB, SQLite ฯลฯ) ในโปรแกรมเดียว | เฉพาะ PostgreSQL และ fork ที่ compatible เท่านั้น |
| ความเร็วของ UI | เร็วกว่า โดยเฉพาะเมื่อ browse ตารางข้อมูลขนาดใหญ่ | บางครั้งช้าเมื่อข้อมูลเยอะมาก (โดยเฉพาะ web mode) |
| Data editor | รองรับ filter/sort/group แบบ spreadsheet ในตัว, แก้ค่าแล้ว preview SQL ก่อน commit ได้ | แก้ไขได้ แต่ฟีเจอร์ filter ในตัว grid มีจำกัดกว่า |
| SQL auto-complete | ฉลาดกว่า จดจำ alias, join แนะนำ column ตาม context ได้ดี | มี auto-complete แต่ basic กว่า |
| ธีมและ UI | ปรับแต่งได้เยอะ รองรับ dark mode สวยงาม | ปรับแต่งได้เช่นกัน แต่ theme มีจำกัดกว่า |
| Native/ผูกกับ PostgreSQL | เป็น third-party tool ไม่ได้พัฒนาโดยทีม PostgreSQL | พัฒนาและดูแลใกล้ชิดกับ PostgreSQL project ฟีเจอร์ใหม่ของ PostgreSQL มักรองรับเร็วกว่า |
| ฟรีทั้งหมดหรือไม่ | Community Edition ฟรี, Enterprise เสียเงิน | ฟรีทั้งหมด 100% (open source) |
| เหมาะกับ | คนที่ทำงานกับฐานข้อมูลหลายชนิด, นักพัฒนาที่ต้องการ productivity สูง | DBA ที่โฟกัส PostgreSQL ล้วนๆ, ต้องการฟีเจอร์ admin เฉพาะทางล่าสุด |

โดยสรุป DBeaver มักถูกเลือกโดยนักพัฒนา (developer) ที่ต้องสลับไปมาระหว่างฐานข้อมูลหลายชนิดในงานเดียวกัน ส่วน pgAdmin มักถูกเลือกโดย DBA ที่ทำงานกับ PostgreSQL เป็นหลักและต้องการเครื่องมือ admin เฉพาะทาง เช่น การจัดการ replication, tablespace หรือฟีเจอร์ใหม่ล่าสุดของ PostgreSQL

---

## Step 28: DBeaver — ฟีเจอร์ที่มีประโยชน์: data editor, ER diagram, SQL formatter, data export/import

### Data Editor

Double-click ที่ตารางใน Database Navigator จะเปิดแท็บ editor ของตารางนั้น ซึ่งมีแท็บย่อยหลายแท็บ เช่น **Properties**, **Data**, **ER Diagram**, **DDL**

ในแท็บ **Data** จะเห็นข้อมูลแบบ grid คล้าย spreadsheet พร้อมความสามารถ:

- **Filter แบบ inline**: คลิกที่ header ของคอลัมน์ กด icon กรวย (filter) แล้วพิมพ์เงื่อนไข เช่น `price > 1000` โดยไม่ต้องเขียน SQL เอง
- **Sort**: คลิก header คอลัมน์เพื่อ sort ตามคอลัมน์นั้น (คลิกซ้ำสลับ ascending/descending)
- **Group by panel**: ลากคอลัมน์ไปวางใน grouping panel เพื่อสรุปข้อมูลแบบ pivot คร่าวๆ ได้ทันที
- **แก้ไขค่าในเซลล์**: double-click เพื่อแก้ไข แล้วเซลล์ที่เปลี่ยนจะไฮไลต์สีส้ม กด `Ctrl+S` เพื่อ save — DBeaver จะแสดง **preview SQL** (`UPDATE ... SET ... WHERE ...`) ที่กำลังจะรันให้ยืนยันก่อนเสมอ ทำให้ปลอดภัยกว่าการแก้ตรงๆ แบบ silent
- **เพิ่ม/ลบแถว**: ปุ่ม `+` และ `-` บน toolbar ของ grid

ตัวอย่าง SQL preview ที่ DBeaver แสดงก่อน save การแก้ไข:

```sql
UPDATE public.products SET price=990.00 WHERE id=1;
```

### ER Diagram

DBeaver สร้าง ER diagram ได้สองระดับ:

1. **ระดับ schema ทั้งหมด**: คลิกขวาที่ schema (เช่น `public`) > **View Diagram** จะแสดงทุกตารางในสคีมาพร้อมความสัมพันธ์ foreign key
2. **ระดับตารางเดียว**: เปิดตาราง แล้วไปที่แท็บ **ER Diagram** จะแสดงเฉพาะตารางนั้นกับตารางที่เกี่ยวข้องโดยตรง (ตารางที่มี FK ชี้เข้า-ออก)

Diagram รองรับการ:
- Zoom in/out, ลาก node จัดตำแหน่งใหม่
- เปลี่ยน layout อัตโนมัติ (Tree, Directed graph, Orthogonal)
- Export เป็นไฟล์ภาพ (PNG, SVG) หรือ print ออกมาโดยตรง

### SQL Formatter

เมื่อเขียน SQL Editor แล้วโค้ดยุ่งเหยิง (query ยาวๆ join หลายตารางจนอ่านยาก) กด `Ctrl+Shift+F` (Format SQL) DBeaver จะจัดรูปแบบให้อ่านง่ายอัตโนมัติทันที

ตัวอย่างก่อน format:

```sql
select c.name, o.id, o.total_amount from customers c join orders o on o.customer_id=c.id where o.total_amount>1000 order by o.total_amount desc;
```

หลัง format (`Ctrl+Shift+F`):

```sql
SELECT c.name,
       o.id,
       o.total_amount
FROM customers c
    JOIN orders o ON o.customer_id = c.id
WHERE o.total_amount > 1000
ORDER BY o.total_amount DESC;
```

สไตล์การ format ปรับแต่งได้ที่ **Preferences > Editors > SQL Editor > Formatting** เช่น ให้ keyword เป็นตัวพิมพ์ใหญ่อัตโนมัติ, ความกว้างของ indent ฯลฯ

### Data Export / Import

DBeaver มีตัวช่วย export/import ที่ยืดหยุ่นมาก รองรับหลายรูปแบบไฟล์ปลายทาง

**การ export ข้อมูล:**

1. คลิกขวาที่ตาราง (หรือผลลัพธ์ query) > **Export Data...**
2. เลือกรูปแบบปลายทาง: `Database table` (คัดลอกข้อมูลไปอีกฐานข้อมูล/ตาราง), `CSV`, `JSON`, `XML`, `SQL INSERT statements`, `XLSX (Excel)` ฯลฯ
3. เลือกไฟล์ปลายทางและตัวเลือกเพิ่มเติม (เช่น delimiter, encoding, whether to include header)
4. กด **Proceed** — DBeaver จะแสดง progress bar และสรุปจำนวนแถวที่ export สำเร็จ

**การ import ข้อมูล:**

1. คลิกขวาที่ตาราง > **Import Data...**
2. เลือกไฟล์ต้นทาง (เช่น CSV)
3. Map คอลัมน์ในไฟล์กับคอลัมน์ในตาราง (DBeaver เดาให้อัตโนมัติถ้าชื่อตรงกัน แต่ปรับ mapping มือได้)
4. เลือกโหมด: insert ใหม่ทั้งหมด, หรือ update ถ้ามี key ตรงกัน
5. กด **Proceed**

ฟีเจอร์นี้มีประโยชน์มากเวลาต้อง migrate ข้อมูลจาก Excel หรือระบบเก่าเข้าสู่ PostgreSQL โดยไม่ต้องเขียน script เอง

### ฟีเจอร์เสริมอื่นๆ ที่ควรรู้จัก

- **ER navigation ผ่าน double-click ที่ foreign key value**: กด `Ctrl+Click` ที่ค่าในคอลัมน์ FK จะพาไปยังแถวต้นทางในตารางที่ถูกอ้างอิงทันที
- **SQL Editor แยกตาม connection**: เปิดหลาย SQL editor พร้อมกัน แต่ละอันผูกกับ connection/database คนละตัวได้ ลด error สลับฐานข้อมูลผิด
- **Execution plan แบบ visual**: กด `Ctrl+Shift+E` เพื่อดู explain plan แบบ diagram คล้ายกับ pgAdmin
- **Task automation**: ตั้ง scheduled task ให้ export ข้อมูลอัตโนมัติตามรอบเวลาได้ (ฟีเจอร์นี้มีจำกัดใน Community Edition)

---

## Step 29: VS Code extensions สำหรับ PostgreSQL และการตั้งค่า

หลายคนทำงานเขียนโค้ดหลักอยู่บน VS Code อยู่แล้ว การมี extension ที่ให้คุยกับฐานข้อมูลได้โดยตรงจาก editor ช่วยลดการสลับหน้าต่างไปมา และทำให้เขียน SQL คู่กับโค้ด application ได้ลื่นไหลขึ้น

### ตัวเลือกหลักที่นิยมใช้

**1. PostgreSQL (โดย Microsoft / Chris Kolkman - ปัจจุบันดูแลโดยทีม Microsoft/Azure Data Studio)**

extension นี้เดิมเขียนโดย Chris Kolkman แล้วภายหลังถูก Microsoft รับช่วงพัฒนาต่อ (เกี่ยวข้องกับโปรเจกต์ Azure Data Studio) ให้ความสามารถพื้นฐานที่จำเป็น:

- เชื่อมต่อ PostgreSQL server ผ่าน sidebar
- Browse โครงสร้างฐานข้อมูล (databases, schemas, tables, views, functions)
- รัน query ใน editor แล้วดูผลลัพธ์แบบ grid ในแท็บใหม่
- IntelliSense (auto-complete) สำหรับชื่อตาราง/คอลัมน์เมื่อเขียน SQL ในไฟล์ `.sql`

**2. SQLTools + SQLTools PostgreSQL/Redshift Driver (โดย Matheus Teixeira)**

เป็นชุด extension ที่ได้รับความนิยมสูงมาก เพราะ:

- รองรับหลายฐานข้อมูล (ต้องติดตั้ง driver แยกตามชนิดฐานข้อมูล)
- UI สะอาด ใช้งานง่าย มี connection explorer ในตัว
- รองรับการเขียน query แล้วรันเฉพาะบางส่วน (bookmark), เก็บ query โปรดไว้ (saved queries/snippets)
- แสดงผลลัพธ์เป็นตารางใน panel ด้านล่าง พร้อมปุ่ม export CSV/JSON ในตัว
- Format SQL ในตัว

### การติดตั้งและตั้งค่า SQLTools (ตัวอย่างละเอียด)

1. เปิด VS Code > กดไอคอน Extensions (`Ctrl+Shift+X`)
2. ค้นหา `SQLTools` ติดตั้ง extension หลัก (publisher: Matheus Teixeira)
3. ค้นหา `SQLTools PostgreSQL/Redshift Driver` ติดตั้งเพิ่ม (จำเป็นต้องมีคู่กัน ไม่งั้นเชื่อมต่อ PostgreSQL ไม่ได้)
4. คลิกไอคอน SQLTools ที่ Activity Bar ด้านซ้าย (รูปปลั๊กเสียบ/ฐานข้อมูล)
5. คลิก **Add New Connection**
6. เลือก **PostgreSQL** จากรายการชนิดฐานข้อมูล
7. กรอกฟอร์ม connection:
   - **Connection name**: `Local Dev DB`
   - **Server / Host**: `localhost`
   - **Port**: `5432`
   - **Database**: `mydb`
   - **Username**: `postgres`
   - **Password**: เลือก "Ask on connect" (ปลอดภัยกว่า ไม่เก็บรหัสผ่านในไฟล์) หรือกรอกตรงเพื่อความสะดวก (ไม่แนะนำสำหรับ production)
8. กด **Test Connection** แล้ว **Save Connection**

หลังบันทึก connection จะปรากฏใน sidebar ของ SQLTools ขยาย tree ดู tables/columns ได้เหมือน DBeaver/pgAdmin

### ตัวอย่างไฟล์ config ที่ SQLTools สร้างไว้ใน `.vscode/settings.json`

เมื่อบันทึก connection ผ่าน UI แล้ว SQLTools จะเขียนค่าไว้ในไฟล์ settings ของ VS Code (ระดับ workspace หรือ user) โดยอัตโนมัติ ตัวอย่าง:

```json
{
  "sqltools.connections": [
    {
      "previewLimit": 50,
      "server": "localhost",
      "port": 5432,
      "driver": "PostgreSQL",
      "name": "Local Dev DB",
      "database": "mydb",
      "username": "postgres",
      "askForPassword": true
    }
  ],
  "sqltools.format": {
    "language": "sql",
    "indentSize": 2,
    "reservedWordCase": "upper"
  }
}
```

การมีไฟล์นี้อยู่ใน workspace settings (`.vscode/settings.json`) มีประโยชน์มากสำหรับทีม เพราะทุกคนที่ clone โปรเจกต์แล้วเปิดด้วย VS Code จะเห็น connection ที่ตั้งไว้ล่วงหน้าทันที (ไม่รวมรหัสผ่านถ้าตั้งเป็น `askForPassword: true`)

### การใช้งานพื้นฐาน

สร้างไฟล์ `query.sql` แล้วพิมพ์:

```sql
SELECT id, name, email FROM customers WHERE created_at >= '2026-01-01';
```

วาง cursor ไว้ในบรรทัด statement นั้น แล้วกด:

- `Ctrl+E Ctrl+E` (Windows/Linux) หรือ `Cmd+E Cmd+E` (macOS) เพื่อรัน query
- ผลลัพธ์จะเปิดในแท็บใหม่ชื่อ "SQLTools Results" แสดงเป็นตาราง มีปุ่ม export เป็น CSV/JSON ที่มุมขวาบน

### เกร็ดเพิ่มเติม: extension อื่นที่ช่วยงานเสริม

| Extension | ประโยชน์ |
|---|---|
| **PostgreSQL Formatter** | จัดรูปแบบ SQL แบบเฉพาะทาง PostgreSQL syntax |
| **Rainbow CSV** | เปิดไฟล์ CSV/TSV ที่ export มาแล้วให้อ่านง่าย แยกสีตามคอลัมน์ |
| **DotENV** | ช่วย syntax highlight ไฟล์ `.env` ที่เก็บ connection string |
| **GitLens** | ไม่เกี่ยวกับฐานข้อมูลโดยตรง แต่มีประโยชน์เมื่อทำงานร่วมกับไฟล์ migration SQL ที่เก็บใน git |

### ข้อจำกัดที่ควรรู้

VS Code extension เหมาะกับงาน "เขียนโค้ด + รัน query เร็วๆ ประกอบการพัฒนา" มากกว่างาน admin เชิงลึก เช่น การจัดการ role/permission แบบ visual, การดู execution plan แบบ diagram สวยๆ, หรือ ER diagram — งานเหล่านี้ pgAdmin หรือ DBeaver ยังทำได้ดีกว่า

---

## Step 30: เลือกเครื่องมือให้เหมาะกับงาน

ถึงจุดนี้เราได้รู้จักเครื่องมือหลักทั้งสี่แล้ว คำถามคือ "แล้วควรใช้ตัวไหน" คำตอบคือ **ไม่มีเครื่องมือใดเครื่องมือเดียวที่ตอบโจทย์ทุกสถานการณ์** มืออาชีพมักมีทั้งสี่ตัวติดเครื่องไว้ และสลับใช้ตามงาน

### ตารางเปรียบเทียบสรุป

| คุณสมบัติ | psql | pgAdmin 4 | DBeaver | VS Code Extension |
|---|---|---|---|---|
| ประเภท | CLI | GUI (desktop/web) | GUI (desktop) | Editor integration |
| ติดตั้งเพิ่มหรือไม่ | มาพร้อม PostgreSQL อยู่แล้ว | ต้องติดตั้งแยก | ต้องติดตั้งแยก | ต้องติดตั้งแยกใน VS Code |
| ใช้งานบน server ไม่มี GUI ได้ | ได้ (เป็นตัวเลือกหลัก) | ได้เฉพาะ web mode | ไม่ได้โดยตรง | ไม่ได้โดยตรง (ต้องมี VS Code) |
| เหมาะกับ automation/script | ดีเยี่ยม (`-f`, `-c`, non-interactive) | ไม่เหมาะ | ไม่เหมาะ | ไม่เหมาะ |
| ดูโครงสร้างฐานข้อมูลแบบ visual | จำกัด (text-based) | ดีมาก | ดีมาก | ปานกลาง |
| ER Diagram | ไม่มี | มี | มี | ไม่มี (ปกติ) |
| Execution plan แบบ visual | ไม่มี (text only) | มี | มี | ไม่มี (บาง extension มี) |
| Data editor (แก้ข้อมูลผ่าน grid) | ไม่มี | มี | มี (ดีที่สุด มี preview SQL) | มีจำกัด |
| รองรับฐานข้อมูลอื่นนอก PostgreSQL | ไม่ (เฉพาะ PostgreSQL-family) | ไม่ (เฉพาะ PostgreSQL-family) | มี (หลายสิบชนิด) | ขึ้นกับ extension ที่ติดตั้ง |
| น้ำหนัก/ความเร็วในการเปิด | เบามาก เปิดเร็วที่สุด | ปานกลาง-หนัก | ปานกลาง | เบา (เพราะใช้ editor ที่เปิดอยู่แล้ว) |
| ค่าใช้จ่าย | ฟรี | ฟรี | ฟรี (Community) / เสียเงิน (Enterprise) | ฟรี |
| เหมาะกับ automation ผ่าน SSH/cron | ดีเยี่ยม | ไม่เหมาะ | ไม่เหมาะ | ไม่เหมาะ |

### Workflow แนะนำสำหรับ Developer

นักพัฒนาแอปพลิเคชันที่ทำงานกับฐานข้อมูลเป็นส่วนหนึ่งของงาน (ไม่ใช่งานหลัก) มักได้ประโยชน์สูงสุดจาก:

1. **VS Code + SQLTools/PostgreSQL extension** เป็นเครื่องมือหลักระหว่างเขียนโค้ด — เขียน query ทดสอบคู่กับโค้ด, รันดูผลเร็วๆ โดยไม่ต้องสลับโปรแกรม
2. **DBeaver** เมื่อต้องสำรวจข้อมูลเชิงลึก, ทำ ER diagram เพื่อเข้าใจ schema ของโปรเจกต์ใหม่, หรือ export/import ข้อมูลทดสอบ (test data)
3. **psql** เมื่อต้อง SSH เข้า server หรือเขียน migration script / seed script ที่ต้องรันซ้ำได้แน่นอนผ่าน CI/CD pipeline

ตัวอย่าง workflow ในทางปฏิบัติ: เขียน SQL migration ไฟล์ `V003__add_index.sql` ใน VS Code (ได้ syntax highlight, auto-complete) → ทดสอบรันผ่าน SQLTools ในฐานข้อมูล dev บนเครื่อง local → เมื่อพร้อม deploy จริง รันผ่าน `psql -f V003__add_index.sql` ใน CI/CD pipeline หรือ deployment script (ซึ่งเป็นวิธีที่ reproducible และ automate ได้)

### Workflow แนะนำสำหรับ DBA

DBA (Database Administrator) ที่ดูแลความเสถียรและ performance ของฐานข้อมูลเป็นงานหลัก มักได้ประโยชน์สูงสุดจาก:

1. **psql** เป็นเครื่องมือหลักที่ใช้เกือบตลอดเวลา โดยเฉพาะเมื่อ SSH เข้า production server ที่มักไม่มี GUI เลย ใช้ทำทุกอย่างตั้งแต่ตรวจสอบ health, รัน `VACUUM`/`ANALYZE` ด้วยมือ, ไปจนถึง emergency troubleshooting ตอนดึก
2. **pgAdmin 4** สำหรับงาน admin เชิง visual — ดู Dashboard เพื่อ monitor real-time activity/locks, จัดการ role และ permission ผ่าน GUI, ดู execution plan แบบ diagram เพื่อวิเคราะห์ query ที่ช้าอย่างละเอียด, และใช้ฟีเจอร์เฉพาะทาง PostgreSQL ล่าสุดที่ pgAdmin มักรองรับก่อนเครื่องมืออื่น
3. **Shell script ที่ครอบ psql ไว้** สำหรับงาน automation ประจำวัน เช่น backup verification, health check report ที่ต้องรันผ่าน cron

DBA มักใช้ DBeaver และ VS Code extension น้อยกว่า developer เพราะงานหลักไม่ใช่การพัฒนาโค้ดแอปพลิเคชัน แต่ถ้าต้อง manage ฐานข้อมูลหลายชนิด (เช่น ดูแลทั้ง PostgreSQL และ MySQL ในองค์กรเดียวกัน) DBeaver ก็เป็นตัวเลือกที่คุ้มค่ามาก เพราะไม่ต้องสลับโปรแกรมไปมา

### หลักการเลือกอย่างง่าย

ถามตัวเองสามคำถามนี้เพื่อช่วยตัดสินใจ:

1. **มี GUI บนเครื่องที่ทำงานอยู่หรือไม่?** ถ้าไม่มี (SSH เข้า server) → ใช้ `psql` เท่านั้น
2. **ต้องการทำ automation/script ที่ต้อง reproducible หรือไม่?** ถ้าใช่ → `psql` เสมอ (ไม่ใช่ GUI เครื่องมือไหนเลย)
3. **ต้องการ visual (diagram, grid แก้ข้อมูล, execution plan graph) เพื่อทำความเข้าใจหรือ debug หรือไม่?** ถ้าใช่ และทำงานกับ PostgreSQL ล้วนๆ → pgAdmin; ถ้าทำงานกับหลายฐานข้อมูล → DBeaver; ถ้าต้องการแค่ query เร็วๆ ระหว่างเขียนโค้ด → VS Code extension

> **คำแนะนำสำหรับผู้เรียน**: ในหลักสูตรนี้ เราจะใช้ **psql เป็นหลัก** สำหรับตัวอย่างคำสั่งและ transcript ต่างๆ เพราะเป็นเครื่องมือที่ universal ที่สุด ทำงานได้ทุกที่ และช่วยให้เข้าใจ PostgreSQL "แบบตรงไปตรงมา" โดยไม่มีชั้น GUI มาบัง อย่างไรก็ตาม ขอแนะนำให้ติดตั้ง pgAdmin หรือ DBeaver ไว้ควบคู่กันด้วย เพื่อใช้ตรวจสอบและทำความเข้าใจโครงสร้างข้อมูลแบบ visual ประกอบการเรียน

---

## สรุปท้ายบท

ในบทนี้เราได้ทำความรู้จักกับเครื่องมือหลักสี่ตัวที่ใช้ทำงานกับ PostgreSQL ในโลกจริง:

- **psql** — CLI client ที่มาพร้อม PostgreSQL เรียนรู้การเข้า-ออก, meta-commands สำคัญ (`\l`, `\c`, `\dt`, `\d`, `\du`, `\dn`, `\x`, `\timing`), การรันไฟล์ script (`\i`), การ export ข้อมูล (`\o`, `\copy`), และการปรับแต่งด้วย `.psqlrc`
- **pgAdmin 4** — GUI มาตรฐานที่ผูกกับ PostgreSQL โดยตรง มี Query Tool, ER Diagram generator, และ visual execution plan
- **DBeaver** — universal database tool ที่รองรับหลายฐานข้อมูล มี data editor ที่ยืดหยุ่น, SQL formatter, และระบบ export/import ที่หลากหลายรูปแบบ
- **VS Code extensions** (PostgreSQL extension, SQLTools) — ทำให้เขียน SQL คู่กับโค้ด application ได้โดยไม่ต้องสลับโปรแกรม

หลักสำคัญที่สุดที่ควรจำจากบทนี้คือ: **ไม่มีเครื่องมือใดเครื่องมือหนึ่งที่ดีที่สุดสำหรับทุกงาน** นักพัฒนามืออาชีพและ DBA ที่เก่งมักคล่องกับเครื่องมือหลายตัว และเลือกใช้ให้เหมาะกับบริบท — โดยเฉพาะ `psql` ที่ควรเป็นทักษะพื้นฐานที่ทุกคนต้องมี เพราะเป็นเครื่องมือเดียวที่รับประกันว่าจะมีอยู่ในทุกสภาพแวดล้อมที่มี PostgreSQL ติดตั้งอยู่

ในบทถัดไป เราจะเจาะลึกเข้าไปใน **สถาปัตยกรรมของ PostgreSQL** (process model, memory architecture, WAL, MVCC เบื้องต้น) ซึ่งเป็นความรู้พื้นฐานที่จำเป็นก่อนจะไปเรียนรู้เรื่อง SQL และการออกแบบฐานข้อมูลในเชิงลึกต่อไป

---

## แบบฝึกหัด

**แบบฝึกหัดที่ 1**

จงเขียนคำสั่ง shell ที่ใช้เชื่อมต่อไปยัง PostgreSQL server ที่ host `db.example.com` พอร์ต `5433` ด้วย user `analyst` เข้าฐานข้อมูล `analytics`

<details>
<summary>เฉลย</summary>

```bash
psql -h db.example.com -p 5433 -U analyst -d analytics
```

</details>

---

**แบบฝึกหัดที่ 2**

ภายใน psql session ต้องการดูรายชื่อตารางทั้งหมดในฐานข้อมูลปัจจุบัน พร้อมขนาดของแต่ละตาราง ต้องใช้ meta-command ใด

<details>
<summary>เฉลย</summary>

```
\dt+
```

การเติม `+` ต่อท้าย `\dt` จะแสดงคอลัมน์เพิ่มเติมรวมถึง Size และ Description ของแต่ละตาราง

</details>

---

**แบบฝึกหัดที่ 3**

ผู้เรียนพิมพ์คำสั่งนี้ใน psql แต่ prompt กลายเป็น `mydb-#` ค้างอยู่ ไม่ยอมกลับไปเป็น `mydb=#`:

```
mydb=# SELECT * FROM customers
```

เกิดอะไรขึ้น และจะแก้ไขอย่างไร

<details>
<summary>เฉลย</summary>

เกิดขึ้นเพราะลืมใส่ semicolon (`;`) ปิดท้ายคำสั่ง SQL ทำให้ psql คิดว่าคำสั่งยังไม่จบ และรอบรรทัดถัดไปต่อ (prompt เปลี่ยนเป็น `-#`) วิธีแก้คือพิมพ์ `;` แล้วกด Enter เพื่อจบคำสั่ง:

```
mydb-# ;
```

หรือกด `Ctrl+C` เพื่อยกเลิกคำสั่งที่พิมพ์ค้างไว้ทั้งหมด

</details>

---

**แบบฝึกหัดที่ 4**

จงเขียนคำสั่ง psql meta-command สำหรับ export ผลลัพธ์ของ query `SELECT * FROM products WHERE price > 1000` ไปเป็นไฟล์ CSV ชื่อ `expensive_products.csv` พร้อม header โดยไฟล์ต้องถูกเขียนที่เครื่อง client (ไม่ใช่เครื่อง server)

<details>
<summary>เฉลย</summary>

```
\copy (SELECT * FROM products WHERE price > 1000) TO 'expensive_products.csv' WITH (FORMAT csv, HEADER true)
```

ใช้ `\copy` แทน `COPY` (SQL command) เพราะ `\copy` รันฝั่ง client ทำให้ไฟล์ถูกเขียนบนเครื่องที่รัน psql อยู่ และไม่ต้องการสิทธิ์ superuser

</details>

---

**แบบฝึกหัดที่ 5**

ต้องการให้ psql แสดงเวลาที่ใช้รัน query ทุกครั้งโดยอัตโนมัติทุกครั้งที่เปิด session ใหม่ โดยไม่ต้องพิมพ์ `\timing` เอง ควรตั้งค่าที่ไหน และตั้งอย่างไร

<details>
<summary>เฉลย</summary>

ตั้งค่าในไฟล์ `~/.psqlrc` โดยเพิ่มบรรทัด:

```sql
\timing on
```

ไฟล์นี้จะถูกอ่านและรันคำสั่งข้างในโดยอัตโนมัติทุกครั้งที่เปิด psql session ใหม่

</details>

---

**แบบฝึกหัดที่ 6**

เมื่อเชื่อมต่อ pgAdmin 4 เข้ากับ PostgreSQL server ครั้งแรก ผู้เรียนต้องกรอกข้อมูลอะไรบ้างในแท็บ **Connection** ของหน้าต่าง Register Server จงระบุอย่างน้อย 4 อย่าง

<details>
<summary>เฉลย</summary>

1. Host name/address (เช่น `localhost`)
2. Port (เช่น `5432`)
3. Maintenance database (เช่น `postgres`)
4. Username (เช่น `postgres`)
5. Password (พร้อมตัวเลือกให้บันทึกรหัสผ่านหรือไม่)

</details>

---

**แบบฝึกหัดที่ 7**

จงอธิบายความแตกต่างหลักระหว่าง DBeaver กับ pgAdmin 4 อย่างน้อย 3 ประเด็น และบอกว่ากรณีใดควรเลือกใช้ DBeaver แทน pgAdmin

<details>
<summary>เฉลย</summary>

ความแตกต่างหลัก:

1. DBeaver รองรับฐานข้อมูลหลายชนิด (MySQL, Oracle, SQL Server ฯลฯ) ในโปรแกรมเดียว ส่วน pgAdmin รองรับเฉพาะ PostgreSQL-family
2. DBeaver มี data editor ที่ยืดหยุ่นกว่า มี filter/sort/group ในตัว และแสดง SQL preview ก่อน commit การแก้ไขข้อมูล
3. pgAdmin พัฒนาใกล้ชิดกับทีม PostgreSQL core จึงมักรองรับฟีเจอร์ใหม่ล่าสุดของ PostgreSQL เร็วกว่า และฟรี 100% ในขณะที่ DBeaver มีทั้งรุ่นฟรี (Community) และเสียเงิน (Enterprise)

ควรเลือก DBeaver เมื่อทำงานกับฐานข้อมูลหลายชนิดในงานเดียวกัน (เช่น ดูแลทั้ง PostgreSQL และ MySQL) หรือต้องการ data editor ที่ทรงพลังกว่าสำหรับแก้ไขข้อมูลจำนวนมาก

</details>

---

**แบบฝึกหัดที่ 8**

ผู้เรียนกำลังเขียนโค้ด backend อยู่ใน VS Code และต้องการทดสอบ query สั้นๆ บ่อยครั้งโดยไม่อยากสลับไปเปิดโปรแกรมอื่น ควรติดตั้ง extension อะไร และตั้งค่าอย่างไรในเบื้องต้น

<details>
<summary>เฉลย</summary>

ควรติดตั้ง **SQLTools** พร้อม **SQLTools PostgreSQL/Redshift Driver** (หรือใช้ extension **PostgreSQL** โดย Microsoft ก็ได้) จากนั้นเปิด sidebar ของ SQLTools > Add New Connection > เลือก PostgreSQL > กรอก host, port, database, username, password > Test Connection > Save จากนั้นสามารถเขียน query ในไฟล์ `.sql` แล้วรันด้วย `Ctrl+E Ctrl+E` เพื่อดูผลลัพธ์โดยไม่ต้องออกจาก VS Code

</details>

---

**แบบฝึกหัดที่ 9**

DBA คนหนึ่งต้อง SSH เข้า production server ที่ไม่มี GUI ใดๆ เลย เพื่อตรวจสอบว่าฐานข้อมูล `orders_db` มีอยู่จริงหรือไม่ และมีตารางอะไรบ้าง จงเขียนขั้นตอนคำสั่งที่ใช้ตรวจสอบโดยใช้เครื่องมือที่เหมาะสมที่สุด

<details>
<summary>เฉลย</summary>

เครื่องมือที่เหมาะสมที่สุดคือ `psql` เพราะเป็น CLI ที่ใช้งานได้แม้ไม่มี GUI

```bash
# ตรวจสอบว่ามีฐานข้อมูลนี้จริงหรือไม่
psql -U postgres -l

# เชื่อมต่อเข้าฐานข้อมูลนั้น
psql -U postgres -d orders_db

# ดูรายชื่อตารางทั้งหมด
orders_db=# \dt
```

</details>

---

**แบบฝึกหัดที่ 10**

จงอธิบายว่าทำไม `psql` ยังคงเป็นเครื่องมือที่ "ขาดไม่ได้" แม้จะมี GUI ที่ทันสมัยกว่าอย่าง pgAdmin และ DBeaver ให้ใช้งาน

<details>
<summary>เฉลย</summary>

เหตุผลหลักได้แก่:

1. **มีอยู่แล้วในทุกที่ที่ติดตั้ง PostgreSQL** ไม่ต้องติดตั้งเพิ่ม โดยเฉพาะบน production server ที่มักไม่มี GUI
2. **เหมาะกับ automation** — ใช้ใน shell script, cron job, CI/CD pipeline ได้อย่างเป็นธรรมชาติ ผ่าน flag `-f` และ `-c` ซึ่ง GUI ทำไม่ได้ (หรือทำได้ยากกว่ามาก)
3. **เบาและเร็ว** — ไม่กิน resource ของเครื่อง เหมาะกับสถานการณ์ฉุกเฉินที่ต้อง troubleshoot อย่างรวดเร็ว
4. **สอนให้เข้าใจ PostgreSQL แบบตรงไปตรงมา** ไม่มีชั้น GUI บดบัง ทำให้เข้าใจพฤติกรรมจริงของฐานข้อมูลได้ดีกว่า

ด้วยเหตุนี้ psql จึงยังเป็นทักษะพื้นฐานที่จำเป็นสำหรับทุกคนที่ทำงานกับ PostgreSQL อย่างจริงจัง ไม่ว่าจะถนัด GUI มากแค่ไหนก็ตาม

</details>

---

## บทถัดไป

[Part 004: สถาปัตยกรรมของ PostgreSQL](./part-004-architecture.md) — เจาะลึก process model, memory architecture, WAL และแนวคิด MVCC เบื้องต้น ซึ่งเป็นรากฐานสำคัญก่อนเข้าสู่การเขียน SQL และออกแบบฐานข้อมูลในบทถัดๆ ไป
