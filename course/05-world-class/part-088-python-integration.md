# Part 088: เชื่อมต่อ PostgreSQL กับ Python (psycopg, SQLAlchemy, Django ORM)

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับโลก/ผู้เชี่ยวชาญ | Part 088

---

## เป้าหมายการเรียนรู้

เมื่อจบบทนี้ ผู้เรียนจะสามารถ:

1. อธิบายภาพรวมของไลบรารีสาย Python ที่ใช้คุยกับ PostgreSQL ได้ครบทุกระดับ — ตั้งแต่ driver ระดับต่ำ (psycopg) ไปจนถึง ORM เต็มรูปแบบ (SQLAlchemy ORM, Django ORM) — และเลือกเครื่องมือให้เหมาะกับงานแต่ละแบบ
2. เขียนโปรแกรม Python เชื่อมต่อ PostgreSQL ด้วย **psycopg (v3)** ได้อย่างถูกต้องและปลอดภัย ด้วย parameterized query ป้องกัน SQL Injection
3. จัดการ **Transaction** ด้วย context manager (`with conn:`, `with conn.transaction():`) ให้ commit/rollback อัตโนมัติอย่างถูกต้อง
4. ใช้ **Connection Pool** (`psycopg_pool`) เพื่อจัดการ connection อย่างมีประสิทธิภาพในแอปพลิเคชันจริง และเขียนโค้ด **async** ด้วย `AsyncConnection`
5. แยกความแตกต่างระหว่าง **SQLAlchemy Core** และ **SQLAlchemy ORM** — และรู้ว่าจะเลือกใช้ระดับไหนเมื่อไหร่
6. ประกาศ **Model class** ด้วย SQLAlchemy ORM, ใช้ `Session` ทำ query พื้นฐาน (`select`, `filter`, `where`)
7. สร้างความสัมพันธ์ระหว่างตาราง (`relationship()`, `back_populates`) และ JOIN ผ่าน ORM
8. ใช้ **Alembic** จัดการ schema migration แบบมีเวอร์ชัน และเชื่อมโยงแนวคิด zero-downtime migration จาก Part 079 เข้ากับ workflow ของ Python
9. เข้าใจโครงสร้างพื้นฐานของ **Django ORM** — Model class, QuerySet, `makemigrations`/`migrate`
10. เขียนสคริปต์ ETL/รายงานง่าย ๆ จากฐานข้อมูล e-commerce ได้ทั้งแบบ psycopg ดิบ ๆ และแบบ SQLAlchemy ORM แล้วเปรียบเทียบข้อดีข้อเสีย

---

## เตรียมข้อมูล

บทนี้ใช้ schema อีคอมเมิร์ซแบบย่อ 3 ตาราง ซึ่งจะปรากฏซ้ำในโค้ดตัวอย่างตลอดทั้งบท:

```sql
CREATE TABLE products (
    product_id      SERIAL PRIMARY KEY,
    product_name    VARCHAR(150),
    unit_price      NUMERIC(10,2),
    stock_quantity  INTEGER
);

CREATE TABLE customers (
    customer_id     SERIAL PRIMARY KEY,
    first_name      VARCHAR(60),
    email           VARCHAR(150)
);

CREATE TABLE orders (
    order_id        SERIAL PRIMARY KEY,
    customer_id     INTEGER REFERENCES customers(customer_id),
    order_date      TIMESTAMPTZ DEFAULT now(),
    status          VARCHAR(20)
);
```

และข้อมูลตัวอย่างสำหรับทดลองรันโค้ดในบทนี้:

```sql
INSERT INTO customers (first_name, email) VALUES
    ('Somchai', 'somchai@example.com'),
    ('Malee',   'malee@example.com'),
    ('Anong',   'anong@example.com');

INSERT INTO products (product_name, unit_price, stock_quantity) VALUES
    ('Wireless Mouse',     390.00, 120),
    ('Mechanical Keyboard', 1590.00,  45),
    ('USB-C Hub',           690.00,  80),
    ('27-inch Monitor',    6990.00,  15);

INSERT INTO orders (customer_id, status) VALUES
    (1, 'completed'),
    (1, 'pending'),
    (2, 'completed'),
    (3, 'cancelled');
```

โครงสร้างโปรเจกต์ Python ที่แนะนำสำหรับบทนี้ (ใช้ virtual environment แยกต่างหาก):

```bash
python3 -m venv .venv
source .venv/bin/activate

pip install "psycopg[binary,pool]" psycopg_pool sqlalchemy alembic django
```

ตัวแปรเชื่อมต่อฐานข้อมูลที่ใช้ร่วมกันทุกตัวอย่างในบทนี้ (แนะนำให้เก็บผ่าน environment variable ไม่ hardcode รหัสผ่านในโค้ด):

```python
import os

DB_DSN = os.environ.get(
    "DATABASE_URL",
    "postgresql://ecommerce_app:secret@localhost:5432/ecommerce",
)
```

> **หมายเหตุเรื่องความปลอดภัย**: ตลอดทั้งบทนี้ เราจะเน้นย้ำเรื่อง parameterized query ซ้ำหลายครั้ง เพราะการต่อ string SQL เองด้วย f-string หรือ `%` เป็นช่องโหว่ SQL Injection ที่พบบ่อยที่สุดในโค้ด Python ที่คุยกับฐานข้อมูล

---

## Step 871: ภาพรวมการเชื่อมต่อ PostgreSQL จาก Python

ระบบนิเวศของ Python สำหรับคุยกับ PostgreSQL มีอยู่ 3 ระดับหลัก ซึ่งแต่ละระดับเหมาะกับสถานการณ์ต่างกัน:

### 1. Driver ระดับต่ำ: psycopg2 / psycopg (v3)

**psycopg** คือ DB-API driver ที่คุยกับ PostgreSQL โดยตรง ส่ง SQL แบบ raw string (พร้อม placeholder) และรับผลลัพธ์กลับมาเป็น tuple/dict ไม่มีการ "แปลง object" ให้อัตโนมัติ

- **psycopg2**: เวอร์ชันดั้งเดิม เสถียรมาก ใช้กันแพร่หลายมาเป็นสิบปี เป็น synchronous (blocking) ล้วน ๆ ไม่รองรับ async แบบเนทีฟ
- **psycopg (v3)**: เขียนใหม่ทั้งหมด (บางครั้งเรียก `psycopg3` เพื่อความชัดเจน) รองรับทั้ง synchronous และ **async/await** แบบเนทีฟ, มี connection pool ในตัว (`psycopg_pool`), รองรับ pipeline mode, มี type adapter ที่ยืดหยุ่นกว่า, และ API ที่ทันสมัยกว่า

เหมาะกับ: งานที่ต้องการควบคุม SQL อย่างละเอียด, งานที่ performance-critical, สคริปต์ ETL/batch job, งานที่ทีมถนัด SQL อยู่แล้วและไม่อยากให้ ORM "แปลง" query ที่เขียนไว้อย่างตั้งใจ, ระบบที่ต้องการ async I/O เต็มรูปแบบ (เช่น เขียนคู่กับ FastAPI)

### 2. Toolkit ระดับกลาง-สูง: SQLAlchemy

**SQLAlchemy** แบ่งเป็น 2 ชั้นการทำงานที่ใช้ร่วมกันได้:

- **SQLAlchemy Core**: เขียน SQL ผ่าน Python expression language (query builder) ยังคงคิดเป็น "ตารางและคอลัมน์" ไม่ใช่ "object" เหมาะกับงานที่ต้องการความยืดหยุ่นสูงแต่ไม่อยากต่อ string SQL เอง
- **SQLAlchemy ORM**: แมป class ของ Python เข้ากับตารางในฐานข้อมูล (Object-Relational Mapping) ทำงานกับ object ได้โดยตรง มี `Session` คอยติดตามการเปลี่ยนแปลง (Unit of Work pattern), รองรับ relationship ระหว่าง object

เหมาะกับ: แอปพลิเคชันขนาดกลาง-ใหญ่ที่ต้องการ portability ข้าม database, โปรเจกต์ที่ต้องการ type-safety และ IDE autocomplete, ทีมที่ต้องการ migration tool (Alembic) แยกจาก framework

### 3. Framework ORM: Django ORM

**Django ORM** เป็นส่วนหนึ่งของ Django framework ผูกกับวงจรชีวิตของ Django project ทั้งหมด (models → migrations → admin site → views) ออกแบบมาให้ "เริ่มเร็ว จบไว" สำหรับเว็บแอปพลิเคชัน

เหมาะกับ: โปรเจกต์ที่ใช้ Django ทั้ง stack อยู่แล้ว, ทีมที่ต้องการ Admin UI ฟรี, งานที่เน้นความเร็วในการพัฒนา (rapid development) มากกว่าการควบคุม SQL แบบละเอียด

### ตารางเปรียบเทียบเบื้องต้น

| มิติ | psycopg (v3) | SQLAlchemy Core | SQLAlchemy ORM | Django ORM |
|---|---|---|---|---|
| ระดับ abstraction | ต่ำสุด (ใกล้ SQL) | กลาง (query builder) | สูง (object) | สูง (object + framework) |
| ควบคุม SQL ได้ละเอียดแค่ไหน | สูงสุด | สูง | ปานกลาง | ต่ำ-ปานกลาง |
| ความเร็วในการพัฒนา | ช้าสุด (เขียนเองทุกอย่าง) | ปานกลาง | เร็ว | เร็วสุด |
| Async รองรับ | ใช่ (เนทีฟ) | ใช่ (ตั้งแต่ 1.4+/2.0) | ใช่ (ตั้งแต่ 2.0) | ใช่ (ตั้งแต่ Django 4.1, เพิ่มขึ้นเรื่อย ๆ) |
| ต้องพึ่ง framework | ไม่ | ไม่ | ไม่ | ใช่ (ต้องใช้ Django) |
| Migration tool | ไม่มีในตัว (เขียน SQL/ใช้ tool แยก) | Alembic (แยก package) | Alembic (แยก package) | มีในตัว (`makemigrations`) |
| เหมาะกับ | สคริปต์, ETL, microservice, งาน perf-critical | งานที่ต้องยืดหยุ่นแต่คุม schema เอง | เว็บแอปทั่วไป, domain model ซับซ้อน | เว็บแอปแบบ full-stack Django |

ในบทนี้เราจะครอบคลุมทั้ง 3 สาย โดยเริ่มจาก psycopg (ใกล้ SQL ที่สุด) ไล่ขึ้นไปจนถึง SQLAlchemy ORM และปิดท้ายด้วย Django ORM

---

## Step 872: psycopg (v3) พื้นฐาน — การติดตั้ง, connection, cursor, parameterized query

### การติดตั้ง

```bash
# แนะนำสำหรับ production: ใช้ binary wheel (ไม่ต้อง compile libpq เอง)
pip install "psycopg[binary]"

# สำหรับ production ที่ต้องการควบคุม libpq เอง (เช่นต้องการเวอร์ชันตรงกับระบบ) ให้ใช้:
# pip install psycopg
# แล้วติดตั้ง libpq-dev ของระบบปฏิบัติการเอง
```

### การเชื่อมต่อพื้นฐาน

```python
import psycopg

conn = psycopg.connect(
    "postgresql://ecommerce_app:secret@localhost:5432/ecommerce"
)

# หรือระบุเป็น keyword argument แยกทีละตัวก็ได้
conn = psycopg.connect(
    host="localhost",
    port=5432,
    dbname="ecommerce",
    user="ecommerce_app",
    password="secret",
)

conn.close()
```

โดยทั่วไปควรใช้ `with` เพื่อให้ connection ถูกปิดอัตโนมัติแม้เกิด exception:

```python
import psycopg

with psycopg.connect(DB_DSN) as conn:
    print("เชื่อมต่อสำเร็จ:", conn.info.dbname)
# ออกจาก with block แล้ว connection จะถูกปิดให้อัตโนมัติ
```

### Cursor และการดึงข้อมูล

`cursor` คือ object ที่ใช้ส่งคำสั่ง SQL และดึงผลลัพธ์กลับมา psycopg3 แนะนำให้ใช้ cursor ผ่าน `with` เช่นกัน:

```python
with psycopg.connect(DB_DSN) as conn:
    with conn.cursor() as cur:
        cur.execute("SELECT product_id, product_name, unit_price FROM products")
        rows = cur.fetchall()
        for row in rows:
            print(row)   # (1, 'Wireless Mouse', Decimal('390.00'))
```

ผลลัพธ์แถวหนึ่งจะเป็น `tuple` โดยดีฟอลต์ ถ้าต้องการผลลัพธ์เป็น `dict` (คีย์เป็นชื่อคอลัมน์) ใช้ `row_factory`:

```python
from psycopg.rows import dict_row

with psycopg.connect(DB_DSN) as conn:
    with conn.cursor(row_factory=dict_row) as cur:
        cur.execute("SELECT product_id, product_name, unit_price FROM products")
        for row in cur.fetchall():
            print(row["product_name"], row["unit_price"])
```

นอกจาก `dict_row` ยังมี `namedtuple_row` (คืนค่าเป็น namedtuple เข้าถึงด้วย `.attribute`) และ `class_row(SomeClass)` (แมปเป็น dataclass ของเราเอง) ให้เลือกใช้ตามสไตล์โค้ด

### วิธีดึงผลลัพธ์แบบต่าง ๆ

```python
with conn.cursor() as cur:
    cur.execute("SELECT product_id, product_name FROM products ORDER BY product_id")

    first_row = cur.fetchone()        # ดึงมา 1 แถว หรือ None ถ้าหมด
    next_three = cur.fetchmany(3)     # ดึงมา 3 แถวถัดไป
    the_rest = cur.fetchall()         # ดึงที่เหลือทั้งหมด

    # หรือวนลูปแบบ streaming (ประหยัดหน่วยความจำสำหรับผลลัพธ์จำนวนมาก)
    cur.execute("SELECT product_id, product_name FROM products")
    for row in cur:
        print(row)
```

### Parameterized Query — หัวใจของความปลอดภัย

**ห้ามต่อ string SQL เองเด็ดขาด** เพราะเปิดช่องให้เกิด SQL Injection ให้ใช้ placeholder `%s` เสมอ (แม้ค่าที่ส่งเข้าไปจะเป็นตัวเลขก็ตาม):

```python
# ผิด — อันตรายมาก ห้ามทำแบบนี้เด็ดขาด
customer_email = request_input  # สมมติมาจาก user input
cur.execute(f"SELECT * FROM customers WHERE email = '{customer_email}'")
# ถ้า customer_email = "' OR '1'='1" จะกลายเป็นช่องโหว่ SQL Injection ทันที

# ถูกต้อง — ใช้ placeholder %s แล้วส่งค่าผ่าน parameter แยก
cur.execute(
    "SELECT * FROM customers WHERE email = %s",
    (customer_email,),
)
```

**สำคัญ**: psycopg3 ใช้ `%s` เป็น placeholder เสมอ **ไม่ว่าประเภทข้อมูลของคอลัมน์จะเป็นอะไร** (ต่างจาก driver บางตัวที่ใช้ `%d` สำหรับตัวเลข) และพารามิเตอร์ต้องส่งเป็น `tuple` หรือ `dict` เสมอ ไม่ใช่ scalar เดี่ยว ๆ:

```python
# ส่งพารามิเตอร์เดียว ต้องมี comma ต่อท้ายเพื่อให้เป็น tuple 1 ตัว
cur.execute("SELECT * FROM customers WHERE customer_id = %s", (5,))

# ส่งหลายพารามิเตอร์
cur.execute(
    "SELECT * FROM orders WHERE customer_id = %s AND status = %s",
    (5, "completed"),
)

# หรือใช้ named parameter ด้วย dict (%(name)s) — อ่านง่ายขึ้นเมื่อพารามิเตอร์เยอะ
cur.execute(
    """
    SELECT * FROM orders
    WHERE customer_id = %(cust_id)s
      AND status = %(status)s
    """,
    {"cust_id": 5, "status": "completed"},
)
```

### INSERT / UPDATE / DELETE

```python
with psycopg.connect(DB_DSN) as conn:
    with conn.cursor() as cur:
        cur.execute(
            """
            INSERT INTO customers (first_name, email)
            VALUES (%s, %s)
            RETURNING customer_id
            """,
            ("Preecha", "preecha@example.com"),
        )
        new_id = cur.fetchone()[0]
        print("customer_id ใหม่คือ", new_id)

    conn.commit()  # ต้อง commit เอง ถ้าไม่ใช้ context manager ของ transaction
```

### Batch insert ด้วย `executemany`

```python
new_products = [
    ("Laptop Stand", 590.00, 60),
    ("Webcam 1080p", 890.00, 40),
    ("USB Microphone", 1290.00, 25),
]

with conn.cursor() as cur:
    cur.executemany(
        """
        INSERT INTO products (product_name, unit_price, stock_quantity)
        VALUES (%s, %s, %s)
        """,
        new_products,
    )
conn.commit()
```

> **หมายเหตุ**: สำหรับ batch insert จำนวนมาก (หลักหมื่นแถวขึ้นไป) `executemany` ไม่ใช่ทางเลือกที่เร็วที่สุด — ควรพิจารณาใช้ `copy` (ดูรายละเอียดเรื่อง `COPY` protocol ใน Part ก่อนหน้าที่พูดถึง bulk loading) ซึ่ง psycopg3 รองรับผ่าน `cursor.copy()`:

```python
with conn.cursor() as cur:
    with cur.copy(
        "COPY products (product_name, unit_price, stock_quantity) FROM STDIN"
    ) as copy:
        for product in new_products:
            copy.write_row(product)
conn.commit()
```

`COPY` เร็วกว่า `INSERT` ทีละแถวหรือ `executemany` มากสำหรับข้อมูลจำนวนมาก เพราะข้าม overhead ของการ parse SQL ทีละคำสั่ง

---

## Step 873: psycopg — Transaction handling ด้วย context manager

### PostgreSQL กับ Transaction โดยดีฟอลต์ของ psycopg3

psycopg3 เปิด transaction ให้อัตโนมัติทันทีที่มีการ execute คำสั่งแรกบน connection (autocommit = False เป็นค่าดีฟอลต์) คุณต้องเรียก `conn.commit()` หรือ `conn.rollback()` เองเสมอ มิฉะนั้นการเปลี่ยนแปลงจะไม่ถูกบันทึกจริง — และถ้าปิด connection โดยไม่ commit การเปลี่ยนแปลงทั้งหมดในระหว่าง transaction จะถูก rollback ทิ้ง

### วิธีที่ 1: `with conn:` — จัดการ commit/rollback ให้อัตโนมัติ

`with conn:` (ไม่ใช่ `with psycopg.connect(...) as conn:` ตอนเปิด — แต่คือการใช้ตัวแปร `conn` เป็น context manager ซ้ำได้เรื่อย ๆ) จะ **commit อัตโนมัติเมื่อ block จบโดยไม่มี exception** และ **rollback อัตโนมัติเมื่อเกิด exception**:

```python
import psycopg

with psycopg.connect(DB_DSN) as conn:
    with conn:  # เปิด transaction block: commit เมื่อจบปกติ, rollback เมื่อ exception
        with conn.cursor() as cur:
            cur.execute(
                "UPDATE products SET stock_quantity = stock_quantity - %s "
                "WHERE product_id = %s",
                (5, 1),
            )
            cur.execute(
                "INSERT INTO orders (customer_id, status) VALUES (%s, %s)",
                (1, "pending"),
            )
    # ณ จุดนี้ transaction ถูก commit เรียบร้อยแล้ว (ถ้าไม่มี exception เกิดขึ้น)
```

ถ้าเกิด exception ระหว่างทาง เช่น constraint violation หรือ error ทาง logic ของแอป ทั้ง `UPDATE` และ `INSERT` ด้านบนจะถูก **rollback ทั้งคู่** — ระบบจะไม่ตกอยู่ในสถานะ "อัปเดตสต็อกไปแล้วแต่สร้าง order ไม่สำเร็จ"

### วิธีที่ 2: `conn.transaction()` — สำหรับ nested transaction / savepoint

`conn.transaction()` ให้ความยืดหยุ่นมากกว่า โดยเฉพาะเมื่อต้องการ **savepoint** ซ้อนกันหลายชั้น (เช่น ต้องการ rollback เฉพาะบางส่วนของ transaction โดยไม่ยกเลิกทั้งหมด):

```python
with psycopg.connect(DB_DSN) as conn:
    with conn.transaction():
        with conn.cursor() as cur:
            cur.execute(
                "INSERT INTO customers (first_name, email) VALUES (%s, %s) "
                "RETURNING customer_id",
                ("Wichai", "wichai@example.com"),
            )
            customer_id = cur.fetchone()[0]

        # เปิด savepoint ซ้อนข้างใน — ถ้าส่วนนี้ fail จะ rollback แค่ส่วนนี้
        try:
            with conn.transaction():  # นี่คือ savepoint ภายใน transaction หลัก
                with conn.cursor() as cur:
                    cur.execute(
                        "INSERT INTO orders (customer_id, status) VALUES (%s, %s)",
                        (customer_id, "invalid_status_that_violates_check"),
                    )
        except psycopg.errors.CheckViolation:
            print("การสร้างออเดอร์ล้มเหลว แต่ลูกค้าที่สร้างไว้ก่อนหน้ายังคงอยู่")
        # transaction หลักยังดำเนินต่อได้ตามปกติ เพราะ error ถูกจำกัดอยู่ใน savepoint
```

เมื่อ `conn.transaction()` ตัวในสุดจบโดยไม่มี exception จะเทียบเท่ากับการ `RELEASE SAVEPOINT`; ถ้ามี exception จะเทียบเท่ากับ `ROLLBACK TO SAVEPOINT` แล้วปล่อย exception ต่อขึ้นไปให้โค้ดข้างนอกจัดการ (ในตัวอย่างข้างบนคือ `try/except` ที่ดักไว้)

### การ commit/rollback แบบ manual (ไม่ใช้ context manager ของ transaction)

บางครั้งอาจจำเป็นต้องควบคุม transaction เองแบบละเอียด:

```python
conn = psycopg.connect(DB_DSN)
try:
    with conn.cursor() as cur:
        cur.execute(
            "UPDATE products SET stock_quantity = stock_quantity - %s WHERE product_id = %s",
            (1, 4),
        )
    conn.commit()
except Exception:
    conn.rollback()
    raise
finally:
    conn.close()
```

โดยทั่วไปแนะนำให้ใช้ `with conn:` หรือ `with conn.transaction():` มากกว่าการเขียน `try/except/finally` เอง เพราะลดโอกาสลืม `rollback()` หรือลืม `close()` ในกรณี edge case

### Isolation Level

psycopg3 อนุญาตให้กำหนด isolation level ต่อ connection หรือ transaction:

```python
from psycopg import IsolationLevel

with psycopg.connect(DB_DSN) as conn:
    conn.isolation_level = IsolationLevel.SERIALIZABLE
    with conn.transaction():
        # transaction นี้จะรันภายใต้ SERIALIZABLE isolation
        with conn.cursor() as cur:
            cur.execute("SELECT stock_quantity FROM products WHERE product_id = %s", (1,))
            qty = cur.fetchone()[0]
            if qty > 0:
                cur.execute(
                    "UPDATE products SET stock_quantity = stock_quantity - 1 WHERE product_id = %s",
                    (1,),
                )
```

เมื่อใช้ `SERIALIZABLE` และเกิด serialization conflict PostgreSQL จะโยน error รหัส `40001` (`serialization_failure`) กลับมาเป็น `psycopg.errors.SerializationFailure` — โค้ดที่ใช้ `SERIALIZABLE` ควรมี retry loop ครอบไว้เสมอ (รายละเอียดเรื่อง isolation level และ MVCC มีอธิบายลึกในบทว่าด้วย Transaction Isolation ก่อนหน้านี้)

---

## Step 874: psycopg — Connection Pool ด้วย psycopg_pool และ Async support

### ทำไมต้องใช้ Connection Pool

การเปิด connection ใหม่ทุกครั้งที่มี request เข้ามา (เช่นในเว็บแอปพลิเคชัน) มีต้นทุนสูง — ต้องทำ TCP handshake, authentication, และ PostgreSQL ต้อง fork process ใหม่สำหรับแต่ละ connection **Connection Pool** แก้ปัญหานี้โดยเปิด connection ไว้ล่วงหน้าจำนวนหนึ่งแล้วนำกลับมาใช้ซ้ำ

### การติดตั้งและใช้งาน `psycopg_pool`

```bash
pip install "psycopg[binary,pool]"
```

```python
from psycopg_pool import ConnectionPool

pool = ConnectionPool(
    conninfo=DB_DSN,
    min_size=2,      # จำนวน connection ขั้นต่ำที่เปิดค้างไว้เสมอ
    max_size=10,     # จำนวน connection สูงสุดที่ pool จะเปิดพร้อมกัน
    max_idle=300,    # ปิด connection ที่ไม่ได้ใช้เกิน 300 วินาที (คืนกลับสู่ min_size)
    timeout=30,      # เวลารอสูงสุดถ้า pool เต็มและทุก connection ถูกยืมไปหมด (วินาที)
)

# ยืม connection จาก pool มาใช้งานผ่าน context manager
with pool.connection() as conn:
    with conn.cursor(row_factory=dict_row) as cur:
        cur.execute("SELECT product_name, unit_price FROM products WHERE stock_quantity > %s", (0,))
        for row in cur.fetchall():
            print(row)
# เมื่อออกจาก with block แล้ว connection จะถูก "คืน" กลับสู่ pool ไม่ใช่ปิดจริง

pool.close()  # ปิด pool เมื่อแอปพลิเคชัน shutdown
```

`ConnectionPool` ยังรองรับการเปิดแบบ context manager ทั้งก้อน ซึ่งเหมาะกับสคริปต์สั้น ๆ:

```python
with ConnectionPool(conninfo=DB_DSN, min_size=2, max_size=10) as pool:
    pool.wait()  # รอจน pool เปิด connection ขั้นต่ำสำเร็จก่อนใช้งานจริง (เช่นตอน startup ของแอป)
    with pool.connection() as conn:
        ...
```

### รูปแบบการใช้งานใน Web Framework (ตัวอย่างแนวคิด FastAPI)

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI
from psycopg_pool import ConnectionPool

pool: ConnectionPool | None = None

@asynccontextmanager
async def lifespan(app: FastAPI):
    global pool
    pool = ConnectionPool(conninfo=DB_DSN, min_size=2, max_size=20)
    pool.wait()
    yield
    pool.close()

app = FastAPI(lifespan=lifespan)

@app.get("/products/{product_id}")
def get_product(product_id: int):
    with pool.connection() as conn:
        with conn.cursor(row_factory=dict_row) as cur:
            cur.execute(
                "SELECT product_id, product_name, unit_price FROM products WHERE product_id = %s",
                (product_id,),
            )
            return cur.fetchone()
```

### Async support ด้วย `AsyncConnection`

psycopg3 รองรับ async แบบเนทีฟ เหมาะกับแอปพลิเคชันที่เขียนด้วย `asyncio` เช่น FastAPI แบบ async endpoint:

```python
import asyncio
import psycopg

async def fetch_products():
    async with await psycopg.AsyncConnection.connect(DB_DSN) as conn:
        async with conn.cursor(row_factory=dict_row) as cur:
            await cur.execute(
                "SELECT product_id, product_name, unit_price FROM products ORDER BY unit_price DESC"
            )
            rows = await cur.fetchall()
            return rows

async def main():
    products = await fetch_products()
    for p in products:
        print(p["product_name"], p["unit_price"])

asyncio.run(main())
```

สังเกตว่าเมธอดที่ทำ I/O (เช่น `execute`, `fetchall`, `commit`) ทั้งหมดกลายเป็น `async def` และต้อง `await` — และการเปิด connection ก็เปลี่ยนจาก `with` เป็น `async with`

### Async Connection Pool

```python
from psycopg_pool import AsyncConnectionPool

async def main():
    async with AsyncConnectionPool(conninfo=DB_DSN, min_size=2, max_size=10) as pool:
        await pool.wait()
        async with pool.connection() as conn:
            async with conn.cursor(row_factory=dict_row) as cur:
                await cur.execute("SELECT count(*) AS total FROM orders")
                row = await cur.fetchone()
                print("จำนวนออเดอร์ทั้งหมด:", row["total"])

asyncio.run(main())
```

### รูปแบบ concurrent query หลายตัวพร้อมกัน

จุดแข็งของ async คือสามารถยิงหลาย query พร้อมกันได้โดยไม่บล็อกกัน (แต่ต้องระวังว่าแต่ละ query ควรใช้ connection คนละตัวจาก pool เพราะ connection เดียวใช้พร้อมกันหลาย coroutine ไม่ได้):

```python
async def fetch_customer_order_count(pool, customer_id):
    async with pool.connection() as conn:
        async with conn.cursor() as cur:
            await cur.execute(
                "SELECT count(*) FROM orders WHERE customer_id = %s", (customer_id,)
            )
            (count,) = await cur.fetchone()
            return customer_id, count

async def main():
    async with AsyncConnectionPool(conninfo=DB_DSN, min_size=3, max_size=10) as pool:
        await pool.wait()
        results = await asyncio.gather(
            fetch_customer_order_count(pool, 1),
            fetch_customer_order_count(pool, 2),
            fetch_customer_order_count(pool, 3),
        )
        for customer_id, count in results:
            print(f"customer {customer_id} มี {count} ออเดอร์")

asyncio.run(main())
```

> **ข้อควรระวัง**: อย่าแชร์ `Connection` เดียวกันข้าม coroutine ที่ทำงานพร้อมกัน (concurrent) เพราะ protocol ของ PostgreSQL ไม่รองรับการส่งหลายคำสั่งพร้อมกันบน connection เดียว ให้ยืม connection แยกจาก pool เสมอสำหรับแต่ละงานที่ทำงานพร้อมกัน

---

## Step 875: SQLAlchemy Core เทียบกับ ORM

SQLAlchemy แบ่งการทำงานออกเป็น 2 ชั้นที่ **ใช้ engine เดียวกัน** แต่มีวิธีคิดต่างกันโดยสิ้นเชิง

### SQLAlchemy Core — คิดแบบ "ตารางและคอลัมน์"

Core ให้เราสร้าง `Table` object แทนตาราง แล้วเขียน query ผ่าน expression language ที่ผลลัพธ์เป็น SQL แต่ยังคงคิดเป็นแถว/คอลัมน์ ไม่ใช่ object ของภาษา:

```python
from sqlalchemy import create_engine, Table, Column, Integer, String, Numeric, MetaData, select

engine = create_engine(DB_DSN)
metadata = MetaData()

products = Table(
    "products",
    metadata,
    Column("product_id", Integer, primary_key=True),
    Column("product_name", String(150)),
    Column("unit_price", Numeric(10, 2)),
    Column("stock_quantity", Integer),
)

with engine.connect() as conn:
    stmt = select(products.c.product_name, products.c.unit_price).where(
        products.c.stock_quantity > 0
    )
    result = conn.execute(stmt)
    for row in result:
        print(row.product_name, row.unit_price)
```

Core เหมาะกับงานที่:
- ต้องการยืดหยุ่นสูงในการสร้าง query แบบ dynamic (เช่น สร้าง filter หลายเงื่อนไขตาม input จริง)
- ทำงานกับ bulk data / reporting query ที่ซับซ้อน ไม่ได้ต้องการ object mapping
- ต้องการควบคุม SQL ที่ generate ออกมาอย่างใกล้ชิด แต่ยังอยากได้ database portability และการป้องกัน SQL Injection อัตโนมัติ

### SQLAlchemy ORM — คิดแบบ "object และความสัมพันธ์"

ORM แมป class ของ Python เข้ากับตาราง แล้วทำงานผ่าน object โดยตรง (ตัวอย่างเต็มอยู่ใน Step 876):

```python
from sqlalchemy.orm import Session

with Session(engine) as session:
    stmt = select(Product).where(Product.stock_quantity > 0)
    for product in session.scalars(stmt):
        print(product.product_name, product.unit_price)
        # product คือ object ของ class Product ไม่ใช่ tuple/row ธรรมดา
```

### เปรียบเทียบโค้ดเดียวกันในสองระดับ

**งาน**: ดึงชื่อสินค้าและราคาของสินค้าที่ stock มากกว่า 0 เรียงราคามากไปน้อย

```python
# --- Core ---
stmt = (
    select(products.c.product_name, products.c.unit_price)
    .where(products.c.stock_quantity > 0)
    .order_by(products.c.unit_price.desc())
)
with engine.connect() as conn:
    rows = conn.execute(stmt).all()   # list of Row (คล้าย namedtuple)

# --- ORM ---
stmt = (
    select(Product)
    .where(Product.stock_quantity > 0)
    .order_by(Product.unit_price.desc())
)
with Session(engine) as session:
    products_list = session.scalars(stmt).all()   # list of Product object
```

### ควรเลือกแบบไหน

| สถานการณ์ | แนะนำ |
|---|---|
| ต้องการ business object ที่มี method/behavior ของตัวเอง | ORM |
| Reporting query ซับซ้อน, aggregate, join หลายตารางเพื่อสรุปผล | Core (หรือ raw SQL ผ่าน `text()`) |
| CRUD ของ entity ที่มี relationship ซับซ้อน (customer → orders → order_items) | ORM |
| Batch/ETL job ที่เน้นความเร็วมากกว่าความสะดวกของ object | Core หรือ psycopg โดยตรง |
| ทีมใหม่ต้องการ onboarding เร็ว เขียน business logic เป็นหลัก | ORM |

ทั้งสองระดับใช้ `Engine` เดียวกันและสามารถผสมกันได้ในโปรเจกต์เดียว — เป็นเรื่องปกติที่จะใช้ ORM สำหรับ CRUD ทั่วไป แต่สลับไปใช้ Core (หรือ raw SQL) สำหรับ report ที่ซับซ้อนซึ่ง ORM generate query ได้ไม่มีประสิทธิภาพเท่า

---

## Step 876: SQLAlchemy ORM — Model class, Session, query พื้นฐาน

### การประกาศ Model class (SQLAlchemy 2.0 style)

SQLAlchemy 2.0 แนะนำให้ประกาศ model ด้วย `DeclarativeBase` และ `Mapped`/`mapped_column` ซึ่งได้ type hint ที่ถูกต้องและ IDE autocomplete เต็มรูปแบบ:

```python
from datetime import datetime
from decimal import Decimal
from typing import Optional

from sqlalchemy import String, Numeric, Integer, ForeignKey, TIMESTAMP, func
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship


class Base(DeclarativeBase):
    pass


class Customer(Base):
    __tablename__ = "customers"

    customer_id: Mapped[int] = mapped_column(primary_key=True)
    first_name: Mapped[Optional[str]] = mapped_column(String(60))
    email: Mapped[Optional[str]] = mapped_column(String(150))

    def __repr__(self) -> str:
        return f"Customer(customer_id={self.customer_id!r}, email={self.email!r})"


class Product(Base):
    __tablename__ = "products"

    product_id: Mapped[int] = mapped_column(primary_key=True)
    product_name: Mapped[Optional[str]] = mapped_column(String(150))
    unit_price: Mapped[Optional[Decimal]] = mapped_column(Numeric(10, 2))
    stock_quantity: Mapped[Optional[int]] = mapped_column(Integer)


class Order(Base):
    __tablename__ = "orders"

    order_id: Mapped[int] = mapped_column(primary_key=True)
    customer_id: Mapped[Optional[int]] = mapped_column(ForeignKey("customers.customer_id"))
    order_date: Mapped[Optional[datetime]] = mapped_column(
        TIMESTAMP(timezone=True), server_default=func.now()
    )
    status: Mapped[Optional[str]] = mapped_column(String(20))
```

`mapped_column` คือฟังก์ชันที่แทน `Column` แบบเดิม แต่ทำงานร่วมกับ type hint ของ `Mapped[...]` เพื่อให้ IDE รู้ประเภทข้อมูลล่วงหน้า — `server_default=func.now()` หมายความว่าให้ PostgreSQL เป็นผู้กำหนดค่า default (`now()`) ไม่ใช่ Python ฝั่ง client (ตรงกับ `DEFAULT now()` ใน schema ต้นฉบับ)

### การสร้าง Engine และ Table (ถ้ายังไม่มีในฐานข้อมูล)

```python
from sqlalchemy import create_engine

engine = create_engine(DB_DSN, echo=False)  # echo=True เพื่อดู SQL ที่ generate ออกมา ใช้ตอน debug

# สร้างตารางทั้งหมดตาม model (ใช้เฉพาะตอนพัฒนา/ทดสอบ — production ควรใช้ Alembic แทน)
Base.metadata.create_all(engine)
```

### Session — หัวใจของการทำงานกับ ORM

`Session` คือตัวกลางที่:
1. เปิด connection (ยืมจาก pool ภายใน engine)
2. ติดตามการเปลี่ยนแปลงของ object ที่โหลดมา (Identity Map + Unit of Work)
3. แปล query เป็น SQL และแปลงผลลัพธ์กลับเป็น object
4. จัดการ transaction (commit/rollback)

```python
from sqlalchemy.orm import Session

# เพิ่มข้อมูลใหม่
with Session(engine) as session:
    new_customer = Customer(first_name="Kanya", email="kanya@example.com")
    session.add(new_customer)
    session.commit()
    print("customer_id ใหม่:", new_customer.customer_id)  # SQLAlchemy ดึงค่าที่ database generate ให้อัตโนมัติ
```

### Query พื้นฐานด้วย `select()`

SQLAlchemy 2.0 ใช้ `select()` แบบเดียวกับ Core แต่ส่งผ่าน `session.execute()`/`session.scalars()`:

```python
from sqlalchemy import select

with Session(engine) as session:
    # ดึงทุกแถว
    stmt = select(Product)
    all_products = session.scalars(stmt).all()

    # filter ด้วย .where()
    stmt = select(Product).where(Product.stock_quantity > 50)
    in_stock = session.scalars(stmt).all()

    # filter หลายเงื่อนไข
    stmt = select(Product).where(
        Product.stock_quantity > 0,
        Product.unit_price < 1000,
    )
    affordable = session.scalars(stmt).all()

    # เรียงลำดับและจำกัดจำนวน
    stmt = select(Product).order_by(Product.unit_price.desc()).limit(3)
    top3_expensive = session.scalars(stmt).all()

    # ดึงเรคคอร์ดเดียวด้วย primary key
    product = session.get(Product, 1)   # เทียบเท่า SELECT ... WHERE product_id = 1

    # นับจำนวน
    from sqlalchemy import func
    stmt = select(func.count()).select_from(Product)
    total_products = session.scalar(stmt)
```

### การอัปเดตและลบ object

```python
with Session(engine) as session:
    product = session.get(Product, 1)
    product.stock_quantity -= 1          # แก้ค่า attribute ตรง ๆ
    session.commit()                     # SQLAlchemy สร้าง UPDATE ให้อัตโนมัติ (เฉพาะคอลัมน์ที่เปลี่ยน)

with Session(engine) as session:
    product = session.get(Product, 4)
    if product is not None:
        session.delete(product)
        session.commit()
```

### `filter_by` เทียบกับ `where`

`Query.filter_by()` เป็น API แบบเก่า (SQLAlchemy 1.x style, ยังใช้ได้ใน 2.0) ส่วน `select().where()` เป็นสไตล์ที่แนะนำใน 2.0:

```python
# สไตล์เก่า (1.x, ยังใช้งานได้)
products = session.query(Product).filter_by(product_name="Wireless Mouse").all()

# สไตล์ใหม่ (2.0, แนะนำ)
stmt = select(Product).where(Product.product_name == "Wireless Mouse")
products = session.scalars(stmt).all()
```

บทนี้จะใช้สไตล์ 2.0 (`select()` + `session.scalars()`/`session.execute()`) เป็นหลักตลอดทั้งบท เพราะเป็นแนวทางที่ SQLAlchemy แนะนำอย่างเป็นทางการตั้งแต่เวอร์ชัน 1.4 เป็นต้นมา

---

## Step 877: SQLAlchemy ORM — Relationship และ JOIN ผ่าน ORM

### การประกาศ `relationship()`

เพิ่ม `relationship()` เข้าไปใน model เพื่อให้ ORM รู้ว่า `Customer` หนึ่งคนมีได้หลาย `Order` (one-to-many) และแต่ละ `Order` อ้างอิงกลับไปยัง `Customer` เดียว (many-to-one):

```python
from typing import List
from sqlalchemy.orm import Mapped, mapped_column, relationship


class Customer(Base):
    __tablename__ = "customers"

    customer_id: Mapped[int] = mapped_column(primary_key=True)
    first_name: Mapped[Optional[str]] = mapped_column(String(60))
    email: Mapped[Optional[str]] = mapped_column(String(150))

    # one-to-many: ลูกค้าหนึ่งคนมีหลายออเดอร์
    orders: Mapped[List["Order"]] = relationship(back_populates="customer")


class Order(Base):
    __tablename__ = "orders"

    order_id: Mapped[int] = mapped_column(primary_key=True)
    customer_id: Mapped[Optional[int]] = mapped_column(ForeignKey("customers.customer_id"))
    order_date: Mapped[Optional[datetime]] = mapped_column(
        TIMESTAMP(timezone=True), server_default=func.now()
    )
    status: Mapped[Optional[str]] = mapped_column(String(20))

    # many-to-one: ออเดอร์แต่ละใบอ้างอิงกลับไปยังลูกค้าคนเดียว
    customer: Mapped[Optional["Customer"]] = relationship(back_populates="orders")
```

`back_populates` ทำให้ทั้งสองฝั่งของความสัมพันธ์ "ซิงก์กัน" อัตโนมัติในหน่วยความจำ — เมื่อ set `order.customer = some_customer` ฝั่ง `some_customer.orders` ก็จะมี `order` นั้นรวมอยู่ด้วยทันที โดยไม่ต้อง query ฐานข้อมูลใหม่

### การใช้งาน relationship

```python
with Session(engine) as session:
    customer = session.get(Customer, 1)
    print(customer.first_name)
    for order in customer.orders:          # SQLAlchemy จะ lazy-load ออเดอร์ทั้งหมดของลูกค้าคนนี้
        print(order.order_id, order.status, order.order_date)

    # สร้างออเดอร์ใหม่ผ่าน relationship โดยตรง ไม่ต้องกำหนด customer_id เอง
    new_order = Order(status="pending")
    customer.orders.append(new_order)      # ORM จะกำหนด customer_id ให้อัตโนมัติตอน flush/commit
    session.commit()
```

### Lazy Loading เทียบกับ Eager Loading

ค่าดีฟอลต์ของ `relationship()` คือ **lazy loading** — query ไปดึงข้อมูลที่เกี่ยวข้อง (เช่น `customer.orders`) **เมื่อถูกเข้าถึงจริงเท่านั้น** ไม่ใช่ตอนโหลด `customer` มาตอนแรก

**ปัญหา N+1 query**: ถ้าวนลูปลูกค้าหลายคนแล้วเข้าถึง `.orders` ของแต่ละคน จะเกิด query แยกสำหรับลูกค้าแต่ละคน (1 query สำหรับดึงลูกค้าทั้งหมด + N query สำหรับดึง orders ของลูกค้าแต่ละคน):

```python
# มีปัญหา N+1 query — สมมติมีลูกค้า 1,000 คน จะยิง query ทั้งหมด 1,001 ครั้ง
with Session(engine) as session:
    customers = session.scalars(select(Customer)).all()
    for customer in customers:
        print(customer.first_name, len(customer.orders))   # แต่ละบรรทัดคือ 1 query แยก
```

แก้ด้วย **eager loading** ผ่าน `selectinload` หรือ `joinedload`:

```python
from sqlalchemy.orm import selectinload

with Session(engine) as session:
    stmt = select(Customer).options(selectinload(Customer.orders))
    customers = session.scalars(stmt).all()
    for customer in customers:
        print(customer.first_name, len(customer.orders))   # ไม่มี query เพิ่มเติมอีกแล้ว ข้อมูลถูกโหลดมาล่วงหน้าครบ
```

`selectinload` จะยิง query แยกอีก 1 ครั้ง (ใช้ `IN` clause ดึง order ของลูกค้าทั้งหมดในคราวเดียว) รวมเป็น 2 query ทั้งหมด แทนที่จะเป็น N+1 — เหมาะกับ one-to-many ที่มีจำนวนแถวลูกเยอะ

`joinedload` ใช้ `LEFT OUTER JOIN` รวมในคำสั่งเดียว เหมาะกับ many-to-one หรือ one-to-one ที่ผลลัพธ์ไม่ทำให้แถวบวมมาก:

```python
from sqlalchemy.orm import joinedload

with Session(engine) as session:
    stmt = select(Order).options(joinedload(Order.customer))
    orders = session.scalars(stmt).all()
    for order in orders:
        print(order.order_id, order.customer.first_name)   # customer ถูกโหลดมาพร้อมกันใน JOIN เดียว
```

### JOIN แบบ explicit ผ่าน ORM

บางครั้งต้องการ JOIN แบบระบุเงื่อนไขเอง (ไม่ใช่ตาม relationship) หรือดึงเฉพาะบางคอลัมน์จากหลายตาราง:

```python
stmt = (
    select(Customer.first_name, Order.order_id, Order.status)
    .join(Order, Order.customer_id == Customer.customer_id)
    .where(Order.status == "completed")
)

with Session(engine) as session:
    for row in session.execute(stmt):
        print(row.first_name, row.order_id, row.status)
```

หรือใช้ relationship ที่ประกาศไว้แล้วเป็นตัวช่วยให้ SQLAlchemy หา join condition ให้อัตโนมัติ:

```python
stmt = (
    select(Customer)
    .join(Customer.orders)
    .where(Order.status == "completed")
    .distinct()
)
```

### สรุปเปรียบเทียบ loading strategy

| Strategy | จำนวน query | เหมาะกับ |
|---|---|---|
| `lazy` (ดีฟอลต์) | 1 + N (ถ้าวนลูป) | เข้าถึง relationship เพียงบาง object ไม่ใช่ทั้งหมด |
| `selectinload` | 2 (คงที่) | one-to-many ที่ต้องใช้ relationship ของทุก object ที่โหลดมา |
| `joinedload` | 1 (JOIN เดียว) | many-to-one / one-to-one ที่ไม่ทำให้แถวบวม |
| `subqueryload` | 2 (ใช้ subquery แทน IN) | คล้าย selectinload แต่เหมาะกับกรณีที่มี composite primary key |

---

## Step 878: SQLAlchemy — Alembic สำหรับ Migration

### ทำไมต้องมี Migration Tool

`Base.metadata.create_all()` ใช้ได้แค่ตอนสร้างตารางครั้งแรกในสภาพแวดล้อม dev/test เท่านั้น เพราะมันไม่รู้วิธี "แก้ไข" ตารางที่มีข้อมูลอยู่แล้วใน production (เช่น เพิ่มคอลัมน์ใหม่, เปลี่ยนชนิดข้อมูล, สร้าง index) — **Alembic** คือ migration tool มาตรฐานของ SQLAlchemy ที่บันทึกการเปลี่ยนแปลง schema เป็นไฟล์เวอร์ชันต่อเนื่องกัน (คล้ายกับ git commit ของ schema)

### การติดตั้งและเริ่มต้นโปรเจกต์

```bash
pip install alembic
alembic init alembic
```

คำสั่งนี้จะสร้างโฟลเดอร์ `alembic/` พร้อมไฟล์ `alembic.ini` และ `alembic/env.py`

ตั้งค่า connection string ใน `alembic.ini`:

```ini
sqlalchemy.url = postgresql://ecommerce_app:secret@localhost:5432/ecommerce
```

หรือดึงจาก environment variable ใน `alembic/env.py` (แนะนำสำหรับ production เพื่อไม่ให้รหัสผ่านอยู่ในไฟล์ที่ commit เข้า git):

```python
# alembic/env.py
import os
from myapp.models import Base  # import Base ที่มี metadata ของ model ทั้งหมด

config.set_main_option("sqlalchemy.url", os.environ["DATABASE_URL"])
target_metadata = Base.metadata   # ให้ Alembic เทียบ model กับฐานข้อมูลจริงเพื่อ autogenerate ได้
```

### การสร้าง Migration ใหม่

**Autogenerate** — ให้ Alembic เทียบ model ปัจจุบันกับสถานะฐานข้อมูล แล้วสร้าง migration script ให้อัตโนมัติ (ควรตรวจสอบ script ที่สร้างมาเสมอ ไม่ควรเชื่อ autogenerate แบบไม่ตรวจสอบ):

```bash
alembic revision --autogenerate -m "create initial ecommerce tables"
```

ไฟล์ migration ที่ได้ (ตัวอย่าง):

```python
# alembic/versions/a1b2c3d4e5f6_create_initial_ecommerce_tables.py
from alembic import op
import sqlalchemy as sa

revision = "a1b2c3d4e5f6"
down_revision = None


def upgrade() -> None:
    op.create_table(
        "customers",
        sa.Column("customer_id", sa.Integer(), primary_key=True),
        sa.Column("first_name", sa.String(60)),
        sa.Column("email", sa.String(150)),
    )
    op.create_table(
        "products",
        sa.Column("product_id", sa.Integer(), primary_key=True),
        sa.Column("product_name", sa.String(150)),
        sa.Column("unit_price", sa.Numeric(10, 2)),
        sa.Column("stock_quantity", sa.Integer()),
    )
    op.create_table(
        "orders",
        sa.Column("order_id", sa.Integer(), primary_key=True),
        sa.Column("customer_id", sa.Integer(), sa.ForeignKey("customers.customer_id")),
        sa.Column("order_date", sa.TIMESTAMP(timezone=True), server_default=sa.func.now()),
        sa.Column("status", sa.String(20)),
    )


def downgrade() -> None:
    op.drop_table("orders")
    op.drop_table("products")
    op.drop_table("customers")
```

### การรัน Migration

```bash
alembic upgrade head        # อัปเกรดไปจนถึงเวอร์ชันล่าสุด
alembic downgrade -1        # ย้อนกลับ 1 ขั้น
alembic current             # ดูว่าฐานข้อมูลอยู่ที่เวอร์ชันไหน
alembic history              # ดูประวัติ migration ทั้งหมด
```

### เชื่อมโยงกับแนวคิด Zero-Downtime Migration (Part 079)

หลักการ **expand/contract** ที่กล่าวถึงในบทว่าด้วย Zero-Downtime Migration นำมาใช้กับ Alembic ได้โดยตรง โดยแบ่ง migration ที่มีความเสี่ยงออกเป็นหลายขั้นแยก revision กัน:

```python
# Migration 1: expand — เพิ่มคอลัมน์ใหม่แบบ nullable ก่อน (ไม่ lock ตารางนาน)
def upgrade() -> None:
    op.add_column("orders", sa.Column("shipping_status", sa.String(20), nullable=True))
```

```python
# Migration 2: backfill ข้อมูลเก่าเป็น batch (รันนอกเวลา peak แยกจาก schema change)
def upgrade() -> None:
    conn = op.get_bind()
    conn.execute(
        sa.text(
            "UPDATE orders SET shipping_status = 'unknown' "
            "WHERE shipping_status IS NULL AND order_id BETWEEN :lo AND :hi"
        ),
        {"lo": 1, "hi": 10000},
    )
```

```python
# Migration 3: contract — เมื่อแอปพลิเคชันทุกตัวขึ้นเวอร์ชันใหม่และไม่มี row ที่ NULL แล้ว
# ค่อยใส่ NOT NULL constraint (ควรใช้ NOT VALID + VALIDATE CONSTRAINT แยกขั้น ตามที่อธิบายใน Part 079)
def upgrade() -> None:
    op.execute(
        "ALTER TABLE orders ADD CONSTRAINT shipping_status_not_null "
        "CHECK (shipping_status IS NOT NULL) NOT VALID"
    )
    op.execute("ALTER TABLE orders VALIDATE CONSTRAINT shipping_status_not_null")
```

หลักการสำคัญที่ยกมาจาก Part 079 ที่ยังใช้ได้เต็มที่กับ Alembic:

- **แยก schema change ที่ risky ออกจาก deploy ของโค้ดแอปพลิเคชัน** — อย่าให้ migration ที่ lock ตารางนานรันพร้อมกับตอน deploy โค้ดใหม่
- **เพิ่มก่อน ลบทีหลัง** (expand ก่อน contract) เพื่อให้แอปพลิเคชันเวอร์ชันเก่าและใหม่ทำงานร่วมกับ schema ได้ในช่วงเปลี่ยนผ่าน
- **ใช้ `NOT VALID` + `VALIDATE CONSTRAINT`** แยกขั้นสำหรับ constraint ใหม่บนตารางใหญ่ เพื่อเลี่ยง `ACCESS EXCLUSIVE` lock ยาว
- Alembic เป็นเพียง "เครื่องมือบันทึกเวอร์ชันของ schema" — วินัยเรื่อง zero-downtime ยังต้องมาจากคนเขียน migration script เอง ไม่ใช่สิ่งที่ Alembic ทำให้อัตโนมัติ

### การรัน migration ในขั้นตอน deploy (CI/CD)

```bash
# ตัวอย่าง step ใน pipeline ก่อน deploy โค้ดแอปพลิเคชันเวอร์ชันใหม่
alembic upgrade head
```

ควรทดสอบ `alembic downgrade` ในสภาพแวดล้อม staging ด้วยเสมอ เพื่อให้มั่นใจว่าสามารถ rollback ได้จริงหากเกิดปัญหาหลัง deploy

---

## Step 879: Django ORM แนะนำตัว

### โครงสร้างพื้นฐานของ Django Project

```bash
pip install django
django-admin startproject ecommerce_site
cd ecommerce_site
python manage.py startapp shop
```

### การประกาศ Model

```python
# shop/models.py
from django.db import models


class Customer(models.Model):
    first_name = models.CharField(max_length=60, blank=True, null=True)
    email = models.CharField(max_length=150, blank=True, null=True)

    class Meta:
        db_table = "customers"

    def __str__(self) -> str:
        return f"{self.first_name} <{self.email}>"


class Product(models.Model):
    product_name = models.CharField(max_length=150, blank=True, null=True)
    unit_price = models.DecimalField(max_digits=10, decimal_places=2, null=True)
    stock_quantity = models.IntegerField(null=True)

    class Meta:
        db_table = "products"

    def __str__(self) -> str:
        return self.product_name


class Order(models.Model):
    customer = models.ForeignKey(
        Customer, on_delete=models.CASCADE, db_column="customer_id",
        related_name="orders", null=True,
    )
    order_date = models.DateTimeField(auto_now_add=True)
    status = models.CharField(max_length=20, blank=True, null=True)

    class Meta:
        db_table = "orders"
```

Django สร้าง primary key `id` (BIGSERIAL) ให้อัตโนมัติทุกตาราง เว้นแต่จะประกาศ primary key เอง — ในตัวอย่างนี้เราตั้ง `db_table` ให้ตรงกับชื่อตารางเดิม (`customers`, `products`, `orders`) เพื่อให้ตรงกับ schema ที่กำหนดไว้ตอนต้นบท (ในกรณีของ schema ต้นฉบับที่ใช้ `customer_id` เป็น PK ชื่อ column ไม่ใช่ `id` ทั่วไป จะต้องประกาศ `primary_key=True` เองแทนการปล่อยให้ Django สร้าง default)

### ตั้งค่าการเชื่อมต่อฐานข้อมูลใน `settings.py`

```python
# ecommerce_site/settings.py
import os

DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.postgresql",
        "NAME": os.environ.get("DB_NAME", "ecommerce"),
        "USER": os.environ.get("DB_USER", "ecommerce_app"),
        "PASSWORD": os.environ.get("DB_PASSWORD", "secret"),
        "HOST": os.environ.get("DB_HOST", "localhost"),
        "PORT": os.environ.get("DB_PORT", "5432"),
        "CONN_MAX_AGE": 60,   # persistent connection: ใช้ connection ซ้ำได้นานสูงสุด 60 วินาที
    }
}

INSTALLED_APPS = [
    # ...
    "shop",
]
```

### `makemigrations` และ `migrate`

Django มี migration system ในตัว ทำงานคล้าย Alembic แต่ผูกกับ Django framework โดยตรง:

```bash
python manage.py makemigrations shop   # สร้างไฟล์ migration จากการเปลี่ยนแปลงใน models.py
python manage.py migrate               # รัน migration ที่ยังไม่ได้ apply กับฐานข้อมูล
python manage.py showmigrations        # ดูสถานะ migration ทั้งหมด
python manage.py sqlmigrate shop 0001  # ดู SQL จริงที่ migration นี้จะรัน (ไม่รันจริง)
```

ไฟล์ migration ที่ Django สร้างให้ (ตัวอย่าง):

```python
# shop/migrations/0001_initial.py
from django.db import migrations, models
import django.db.models.deletion


class Migration(migrations.Migration):
    initial = True
    dependencies = []

    operations = [
        migrations.CreateModel(
            name="Customer",
            fields=[
                ("id", models.BigAutoField(auto_created=True, primary_key=True, serialize=False)),
                ("first_name", models.CharField(blank=True, max_length=60, null=True)),
                ("email", models.CharField(blank=True, max_length=150, null=True)),
            ],
            options={"db_table": "customers"},
        ),
        # ... Product, Order ตามลำดับ
    ]
```

### QuerySet พื้นฐาน

Django ORM ใช้ **QuerySet** เป็นตัวแทน query ที่ **lazy** เสมอ (ยังไม่ยิง SQL จริงจนกว่าจะถูก evaluate เช่นวนลูปหรือแปลงเป็น `list()`):

```python
from shop.models import Product, Customer, Order

# ดึงทั้งหมด
all_products = Product.objects.all()

# filter
in_stock = Product.objects.filter(stock_quantity__gt=0)
affordable_in_stock = Product.objects.filter(stock_quantity__gt=0, unit_price__lt=1000)

# เรียงลำดับ
top_expensive = Product.objects.order_by("-unit_price")[:3]

# ดึงแถวเดียว
product = Product.objects.get(pk=1)             # โยน exception ถ้าไม่เจอหรือเจอมากกว่า 1
product = Product.objects.filter(pk=1).first()  # คืน None ถ้าไม่เจอ ปลอดภัยกว่า

# นับจำนวน
total = Product.objects.count()

# สร้างใหม่
new_customer = Customer.objects.create(first_name="Araya", email="araya@example.com")

# อัปเดต
product = Product.objects.get(pk=1)
product.stock_quantity -= 1
product.save()

# หรืออัปเดตแบบ bulk โดยไม่ต้องโหลด object มาก่อน (มีประสิทธิภาพกว่า)
Product.objects.filter(pk=1).update(stock_quantity=models.F("stock_quantity") - 1)

# ลบ
Product.objects.filter(stock_quantity=0).delete()
```

`models.F("stock_quantity") - 1` คือการอ้างอิงค่าคอลัมน์ในฐานข้อมูลโดยตรง (แปลเป็น `SET stock_quantity = stock_quantity - 1` ใน SQL) หลีกเลี่ยง race condition ที่อาจเกิดจากการอ่านค่ามาลบใน Python แล้วค่อยเขียนกลับ

### การ JOIN ผ่าน relationship

```python
# many-to-one: จาก order ไปหา customer (ใช้ select_related เพื่อ JOIN ในคำสั่งเดียว ลด query)
orders = Order.objects.select_related("customer").filter(status="completed")
for order in orders:
    print(order.customer.first_name, order.status)

# one-to-many: จาก customer ไปหา orders ทั้งหมด (ใช้ prefetch_related เพื่อลดปัญหา N+1)
customers = Customer.objects.prefetch_related("orders")
for customer in customers:
    print(customer.first_name, customer.orders.count())
```

`select_related` ทำงานคล้าย `joinedload` ของ SQLAlchemy (ใช้ SQL JOIN) เหมาะกับ many-to-one/one-to-one ส่วน `prefetch_related` ทำงานคล้าย `selectinload` (ยิง query แยกแล้วรวมผลใน Python) เหมาะกับ one-to-many/many-to-many

### Django Admin — จุดแข็งที่ไลบรารีอื่นไม่มี

```python
# shop/admin.py
from django.contrib import admin
from .models import Customer, Product, Order

admin.site.register(Customer)
admin.site.register(Product)
admin.site.register(Order)
```

เพียงเท่านี้ Django จะสร้างหน้า Admin UI แบบเต็มรูปแบบ (list, filter, search, edit, delete) ให้อัตโนมัติโดยไม่ต้องเขียน frontend เอง — นี่คือเหตุผลสำคัญที่หลายทีมเลือก Django สำหรับ internal tool หรือ MVP ที่ต้องการความเร็วในการพัฒนา

---

## Step 880: แบบฝึกหัดรวม — เขียน ETL/รายงานด้วย psycopg และ SQLAlchemy ORM

โจทย์: เขียนสคริปต์ที่สรุป **"ยอดขายและจำนวนออเดอร์ต่อลูกค้า เฉพาะออเดอร์ที่ completed"** ออกมาเป็นรายงาน แล้วเปรียบเทียบวิธีการเขียนทั้งสองแบบ

### เวอร์ชัน 1: psycopg (v3) — ควบคุม SQL เองทั้งหมด

```python
"""
report_psycopg.py
สรุปยอดขายต่อลูกค้า (เฉพาะสถานะ completed) ด้วย psycopg
"""
import os
import psycopg
from psycopg.rows import dict_row

DB_DSN = os.environ["DATABASE_URL"]


def build_customer_sales_report(conn) -> list[dict]:
    query = """
        SELECT
            c.customer_id,
            c.first_name,
            c.email,
            count(o.order_id)                    AS completed_order_count,
            coalesce(sum(p.unit_price), 0)        AS total_estimated_value
        FROM customers c
        JOIN orders o
            ON o.customer_id = c.customer_id
           AND o.status = %s
        LEFT JOIN products p ON false   -- placeholder: schema นี้ไม่มี order_items จริง
        GROUP BY c.customer_id, c.first_name, c.email
        ORDER BY completed_order_count DESC, c.customer_id
    """
    with conn.cursor(row_factory=dict_row) as cur:
        cur.execute(query, ("completed",))
        return cur.fetchall()


def main() -> None:
    with psycopg.connect(DB_DSN) as conn:
        report = build_customer_sales_report(conn)

    print(f"{'ลูกค้า':<20}{'อีเมล':<28}{'จำนวนออเดอร์':>15}")
    print("-" * 63)
    for row in report:
        print(
            f"{row['first_name']:<20}{row['email']:<28}{row['completed_order_count']:>15}"
        )


if __name__ == "__main__":
    main()
```

> หมายเหตุ: schema ในบทนี้ไม่มีตาราง `order_items` (ไม่มีรายการสินค้าต่อออเดอร์) จึงสรุปได้แค่ "จำนวนออเดอร์ที่ completed ต่อคน" — ในระบบจริงที่มี `order_items` จะ JOIN เพิ่มเพื่อคำนวณยอดขายจริงต่อออเดอร์ได้ ตัวอย่างข้างบนปรับให้เหมาะกับ schema แบบย่อของบทนี้ ดังนี้:

```python
def build_customer_order_count_report(conn) -> list[dict]:
    query = """
        SELECT
            c.customer_id,
            c.first_name,
            c.email,
            count(o.order_id) AS completed_order_count
        FROM customers c
        JOIN orders o
            ON o.customer_id = c.customer_id
           AND o.status = %s
        GROUP BY c.customer_id, c.first_name, c.email
        ORDER BY completed_order_count DESC, c.customer_id
    """
    with conn.cursor(row_factory=dict_row) as cur:
        cur.execute(query, ("completed",))
        return cur.fetchall()
```

### เวอร์ชัน 2: SQLAlchemy ORM — เขียนผ่าน object และ relationship

```python
"""
report_sqlalchemy.py
สรุปยอดขายต่อลูกค้า (เฉพาะสถานะ completed) ด้วย SQLAlchemy ORM
"""
import os
from sqlalchemy import create_engine, select, func
from sqlalchemy.orm import Session

from models import Customer, Order   # สมมติว่า model ประกาศไว้ตาม Step 876-877

DB_DSN = os.environ["DATABASE_URL"]


def build_customer_order_count_report(session: Session) -> list[dict]:
    stmt = (
        select(
            Customer.customer_id,
            Customer.first_name,
            Customer.email,
            func.count(Order.order_id).label("completed_order_count"),
        )
        .join(Order, Order.customer_id == Customer.customer_id)
        .where(Order.status == "completed")
        .group_by(Customer.customer_id, Customer.first_name, Customer.email)
        .order_by(func.count(Order.order_id).desc(), Customer.customer_id)
    )
    result = session.execute(stmt)
    return [row._asdict() for row in result]


def main() -> None:
    engine = create_engine(DB_DSN)
    with Session(engine) as session:
        report = build_customer_order_count_report(session)

    print(f"{'ลูกค้า':<20}{'อีเมล':<28}{'จำนวนออเดอร์':>15}")
    print("-" * 63)
    for row in report:
        print(
            f"{row['first_name']:<20}{row['email']:<28}{row['completed_order_count']:>15}"
        )


if __name__ == "__main__":
    main()
```

### เวอร์ชันทางเลือก: ใช้ relationship object แทนการเขียน JOIN เอง

```python
def build_report_via_relationship(session: Session) -> list[dict]:
    """แนวทาง ORM เต็มรูปแบบ: โหลด object แล้วนับ orders ที่ completed ใน Python
       (เหมาะกับข้อมูลจำนวนไม่มาก ไม่เหมาะกับ dataset ใหญ่เพราะดึงข้อมูลมาเกินจำเป็น)"""
    from sqlalchemy.orm import selectinload

    stmt = select(Customer).options(selectinload(Customer.orders))
    customers = session.scalars(stmt).all()

    report = []
    for customer in customers:
        completed = [o for o in customer.orders if o.status == "completed"]
        if completed:
            report.append({
                "customer_id": customer.customer_id,
                "first_name": customer.first_name,
                "email": customer.email,
                "completed_order_count": len(completed),
            })
    report.sort(key=lambda r: r["completed_order_count"], reverse=True)
    return report
```

### เปรียบเทียบทั้ง 3 แนวทาง

| แนวทาง | จำนวน query | ข้อดี | ข้อเสีย |
|---|---|---|---|
| psycopg + raw SQL | 1 | เร็วที่สุด, ควบคุม SQL เต็มที่, เห็น execution plan ตรงไปตรงมา | ต้องเขียน SQL เอง, ต้อง map ผลลัพธ์เอง (แม้จะใช้ `dict_row` ช่วยได้) |
| SQLAlchemy Core-style query (`select()` + `join`) | 1 | ได้ SQL ที่มีประสิทธิภาพเท่า raw SQL, ยัง type-safe และ portable ข้าม database | ต้องเข้าใจ expression language ของ SQLAlchemy |
| SQLAlchemy ORM ผ่าน relationship (`selectinload`) | 2 | โค้ดอ่านง่ายที่สุด ใกล้เคียงภาษามนุษย์ | ดึงข้อมูลมาเกินจำเป็น (โหลดทุก order ไม่ใช่แค่ที่นับ), ไม่เหมาะกับรายงานที่ aggregate ซับซ้อนหรือ dataset ใหญ่ |

**บทเรียนสำคัญ**: สำหรับ **รายงาน/aggregate query** ควรให้ PostgreSQL เป็นผู้คำนวณ (`GROUP BY`, `count()`, `sum()`) เสมอ ไม่ว่าจะผ่าน raw SQL หรือ SQLAlchemy `select()` แบบ Core-style — การดึง object ทั้งหมดมาแล้วคำนวณใน Python (อย่างเวอร์ชัน `build_report_via_relationship`) ใช้ได้กับข้อมูลน้อย แต่ **ไม่ scale** เมื่อข้อมูลโตขึ้น เพราะดึงข้อมูลผ่าน network มากเกินความจำเป็นและใช้ CPU/memory ฝั่ง Python แทนที่จะปล่อยให้ query planner ของฐานข้อมูลทำงาน

---

## สรุปท้ายบท

บทนี้พาเดินทางจากการต่อ PostgreSQL ด้วย driver ดิบ ๆ (psycopg) ไปจนถึง ORM เต็มรูปแบบ (SQLAlchemy ORM, Django ORM) ประเด็นสำคัญที่ควรจำ:

- **psycopg (v3)** คือ driver ระดับต่ำที่ให้ควบคุม SQL ได้เต็มที่ มี transaction context manager (`with conn:`, `with conn.transaction()`) ที่ทำให้ commit/rollback ถูกต้องอัตโนมัติ, มี connection pool ในตัว (`psycopg_pool`) และรองรับ async แบบเนทีฟผ่าน `AsyncConnection`
- **ต้องใช้ parameterized query (`%s`) เสมอ** ไม่ว่าจะเขียนด้วย psycopg หรือ library ใดก็ตาม เพื่อป้องกัน SQL Injection — นี่คือกฎที่ใช้ได้กับทุกระดับ abstraction
- **SQLAlchemy** แบ่งเป็น Core (คิดแบบตาราง/คอลัมน์, ยืดหยุ่นสูง) และ ORM (คิดแบบ object/relationship, พัฒนาโค้ด business logic ได้เร็ว) ทั้งสองใช้ engine เดียวกันและผสมกันได้ในโปรเจกต์จริง
- **`relationship()` + `back_populates`** ทำให้ object สองฝั่งซิงก์กันอัตโนมัติ แต่ต้องระวังปัญหา **N+1 query** จาก lazy loading — แก้ด้วย `selectinload`/`joinedload` (SQLAlchemy) หรือ `prefetch_related`/`select_related` (Django)
- **Alembic** คือมาตรฐานสำหรับ migration ของ SQLAlchemy โดยยังต้องใช้หลักการ expand/contract และ `NOT VALID` + `VALIDATE CONSTRAINT` แบบเดียวกับที่กล่าวถึงในเรื่อง Zero-Downtime Migration เพื่อความปลอดภัยบน production
- **Django ORM** ให้ความเร็วในการพัฒนาสูงสุดพร้อม migration system และ Admin UI ในตัว แต่ผูกกับ Django framework ทั้งหมด
- สำหรับ **รายงาน/aggregate query** ให้ปล่อยให้ PostgreSQL ทำ `GROUP BY`/`count()`/`sum()` เสมอ ไม่ว่าจะเขียนผ่านชั้นไหนก็ตาม อย่าดึง object ทั้งหมดมาคำนวณใน Python

### ตารางเปรียบเทียบสรุป: psycopg vs SQLAlchemy vs Django ORM

| หัวข้อ | psycopg (v3) | SQLAlchemy (Core/ORM) | Django ORM |
|---|---|---|---|
| ระดับควบคุม SQL | เต็มที่ (เขียนเอง) | สูง (Core) / ปานกลาง (ORM) | ต่ำ-ปานกลาง |
| ความเร็วในการเริ่มโปรเจกต์ | ช้า (setup เอง) | ปานกลาง | เร็วที่สุด |
| Type safety / IDE support | ต้องเขียน type hint เอง | ดี (2.0 style กับ `Mapped`) | ดี (ผ่าน field type) |
| Async รองรับ | ใช่ (เนทีฟ, สมบูรณ์ที่สุด) | ใช่ (ตั้งแต่ 1.4/2.0) | ใช่ (ตั้งแต่ 4.1 เพิ่มขึ้นเรื่อย ๆ แต่ยังไม่ครบทุกส่วนเท่า) |
| Migration tool | ไม่มี (เขียน SQL/ผูกกับ tool ภายนอกเอง) | Alembic (แยก package, ยืดหยุ่นสูง) | มีในตัว (`makemigrations`/`migrate`) |
| Connection Pool | `psycopg_pool` (แยก, ยืดหยุ่นสูง) | ในตัว (`QueuePool` เป็นดีฟอลต์) | ในตัว (`CONN_MAX_AGE`, หรือใช้ pgbouncer ภายนอก) |
| Admin UI สำเร็จรูป | ไม่มี | ไม่มี | มี (Django Admin) |
| เหมาะกับ | สคริปต์, ETL, microservice, งาน perf-critical, ระบบที่ไม่ใช้ framework ใหญ่ | เว็บแอปที่ต้องการ portability, domain model ซับซ้อน, ทีมที่ต้องการควบคุมทั้ง Core และ ORM | เว็บแอปแบบ full-stack Django, MVP, internal tool ที่ต้องการ Admin UI ฟรี |
| ความเสี่ยงเรื่อง N+1 query | ไม่มี (เขียน SQL เองทุกครั้ง) | มี (ถ้าใช้ lazy relationship โดยไม่ระวัง) | มี (ถ้าไม่ใช้ `select_related`/`prefetch_related`) |

**แนวทางเลือกโดยสรุป**: ถ้าทำ microservice ขนาดเล็กหรือ ETL script ที่เน้น performance ให้เลือก **psycopg**; ถ้าทำเว็บแอปที่ต้องการ domain model ที่ยืดหยุ่นและอยากคุม migration เอง ให้เลือก **SQLAlchemy + Alembic**; ถ้าทำเว็บแอปแบบ full-stack ที่ต้องการความเร็วในการพัฒนาและ Admin UI ฟรี ให้เลือก **Django ORM** — ในโปรเจกต์ขนาดใหญ่จริงหลายทีมใช้ผสมกัน เช่น ใช้ psycopg สำหรับ batch job/report ที่ perf-critical ควบคู่กับ SQLAlchemy ORM หรือ Django ORM สำหรับส่วน CRUD ทั่วไปของแอปพลิเคชันหลัก

---

## แบบฝึกหัด

<details>
<summary><strong>แบบฝึกหัดที่ 1</strong>: อธิบายว่าเหตุใดโค้ดต่อไปนี้จึงเป็นอันตราย และควรแก้ไขอย่างไร</summary>

```python
status = input("กรอกสถานะออเดอร์ที่ต้องการค้นหา: ")
cur.execute(f"SELECT * FROM orders WHERE status = '{status}'")
```

**เฉลย**:

โค้ดนี้ต่อ string SQL ด้วย f-string โดยตรงจาก input ของผู้ใช้ ทำให้เกิดช่องโหว่ **SQL Injection** — ถ้าผู้ใช้กรอก `' OR '1'='1` จะกลายเป็น `WHERE status = '' OR '1'='1'` ซึ่งคืนค่าทุกแถวในตาราง หรือแย่กว่านั้นอาจกรอกคำสั่งที่ทำลายข้อมูลได้ (เช่น `'; DROP TABLE orders; --`)

วิธีแก้: ใช้ parameterized query เสมอ

```python
status = input("กรอกสถานะออเดอร์ที่ต้องการค้นหา: ")
cur.execute("SELECT * FROM orders WHERE status = %s", (status,))
```

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 2</strong>: เขียนฟังก์ชัน psycopg ที่ตัดสต็อกสินค้าและสร้างออเดอร์ในหนึ่ง transaction — ถ้าสต็อกไม่พอให้ rollback ทั้งหมด</summary>

**เฉลย**:

```python
import psycopg

class OutOfStockError(Exception):
    pass

def place_order(conn: psycopg.Connection, customer_id: int, product_id: int, quantity: int) -> int:
    with conn.transaction():
        with conn.cursor() as cur:
            cur.execute(
                "SELECT stock_quantity FROM products WHERE product_id = %s FOR UPDATE",
                (product_id,),
            )
            row = cur.fetchone()
            if row is None or row[0] < quantity:
                raise OutOfStockError(f"สินค้า product_id={product_id} สต็อกไม่พอ")

            cur.execute(
                "UPDATE products SET stock_quantity = stock_quantity - %s WHERE product_id = %s",
                (quantity, product_id),
            )
            cur.execute(
                "INSERT INTO orders (customer_id, status) VALUES (%s, %s) RETURNING order_id",
                (customer_id, "pending"),
            )
            order_id = cur.fetchone()[0]
    return order_id

# การใช้งาน
with psycopg.connect(DB_DSN) as conn:
    try:
        order_id = place_order(conn, customer_id=1, product_id=4, quantity=1)
        print("สร้างออเดอร์สำเร็จ:", order_id)
    except OutOfStockError as e:
        print("ล้มเหลว:", e)
```

`FOR UPDATE` ใช้ล็อกแถวสินค้าไว้ป้องกัน race condition ระหว่างการตรวจสอบสต็อกกับการอัปเดต (ป้องกันสองคำสั่งพร้อมกันเห็นสต็อกเท่ากันแล้วตัดสต็อกซ้ำจนติดลบ) `with conn.transaction()` รับประกันว่าถ้าเกิด exception ระหว่างทาง ทั้ง `UPDATE` และ `INSERT` จะถูก rollback ทั้งคู่

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 3</strong>: เขียนโค้ดเชื่อมต่อฐานข้อมูลผ่าน connection pool แล้วรัน query 5 ครั้งพร้อมกันด้วย thread (ไม่ใช่ async)</summary>

**เฉลย**:

```python
from concurrent.futures import ThreadPoolExecutor
from psycopg_pool import ConnectionPool
from psycopg.rows import dict_row

pool = ConnectionPool(conninfo=DB_DSN, min_size=3, max_size=10)
pool.wait()

def query_customer(customer_id: int) -> dict | None:
    with pool.connection() as conn:
        with conn.cursor(row_factory=dict_row) as cur:
            cur.execute(
                "SELECT customer_id, first_name, email FROM customers WHERE customer_id = %s",
                (customer_id,),
            )
            return cur.fetchone()

with ThreadPoolExecutor(max_workers=5) as executor:
    results = list(executor.map(query_customer, [1, 2, 3, 1, 2]))

for r in results:
    print(r)

pool.close()
```

`ConnectionPool` เป็น thread-safe อยู่แล้ว — แต่ละ thread ที่เรียก `pool.connection()` จะได้ connection คนละตัวยืมมาใช้ชั่วคราวแล้วคืนกลับ pool เมื่อจบ `with` block

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 4</strong>: อธิบายความแตกต่างระหว่าง `conn.commit()` แบบ manual กับ `with conn:` ของ psycopg3</summary>

**เฉลย**:

- `conn.commit()` แบบ manual ต้องเรียกเองทุกครั้งหลังจบชุดคำสั่งที่ต้องการบันทึก และถ้าเกิด exception ระหว่างทาง ต้องเขียน `try/except` ดัก `conn.rollback()` เองด้วย มิฉะนั้น transaction จะค้างอยู่ในสถานะ error (`InFailedSqlTransaction`) จนกว่าจะ rollback
- `with conn:` (ใช้ตัวแปร connection เป็น context manager) จะ **commit อัตโนมัติ** เมื่อ block จบโดยไม่มี exception และ **rollback อัตโนมัติ** เมื่อมี exception หลุดออกจาก block — ลดโอกาสลืม rollback และทำให้โค้ดกระชับขึ้น

```python
# Manual — เสี่ยงลืม rollback
try:
    cur.execute(...)
    conn.commit()
except Exception:
    conn.rollback()
    raise

# with conn: — จัดการให้อัตโนมัติ
with conn:
    cur.execute(...)
    # commit อัตโนมัติเมื่อจบ block, rollback อัตโนมัติถ้า exception
```

ข้อควรระวัง: `with conn:` ไม่ได้ปิด connection เมื่อจบ block (ต่างจาก `with psycopg.connect(...) as conn:` ตอนเปิดซึ่งปิด connection เมื่อจบ) — สามารถเปิด `with conn:` ซ้ำได้หลายรอบบน connection เดียวกันเพื่อเปิด transaction ใหม่แต่ละครั้ง

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 5</strong>: เขียนโค้ด async ด้วย psycopg3 ที่ดึงจำนวนสินค้าที่ stock_quantity = 0 (สินค้าหมดสต็อก)</summary>

**เฉลย**:

```python
import asyncio
import psycopg

async def count_out_of_stock() -> int:
    async with await psycopg.AsyncConnection.connect(DB_DSN) as conn:
        async with conn.cursor() as cur:
            await cur.execute(
                "SELECT count(*) FROM products WHERE stock_quantity = %s", (0,)
            )
            (count,) = await cur.fetchone()
            return count

async def main():
    total = await count_out_of_stock()
    print(f"สินค้าหมดสต็อกทั้งหมด {total} รายการ")

asyncio.run(main())
```

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 6</strong>: ใน SQLAlchemy ORM เขียน query ที่ดึงสินค้าทั้งหมดที่ยังไม่เคยถูกสั่งซื้อเลย (ไม่มีใน `orders` — สมมติว่ามีตาราง `order_items` เชื่อมระหว่าง `orders` กับ `products`) โดยใช้แนวคิด LEFT JOIN + IS NULL</summary>

**เฉลย**:

โจทย์นี้สมมติว่ามีตาราง `order_items(order_item_id, order_id, product_id, quantity)` เพิ่มเข้ามา (schema เต็มของอีคอมเมิร์ซที่ใช้ในบทอื่น ๆ ของหลักสูตร) วิธีเขียนด้วย SQLAlchemy Core-style `select()`:

```python
from sqlalchemy import select

stmt = (
    select(Product)
    .outerjoin(OrderItem, OrderItem.product_id == Product.product_id)
    .where(OrderItem.order_item_id.is_(None))
)

with Session(engine) as session:
    never_ordered = session.scalars(stmt).all()
    for product in never_ordered:
        print(product.product_name)
```

หลักการ: `outerjoin` (เทียบเท่า `LEFT JOIN`) จับคู่สินค้าทุกตัวกับ order_items ที่ตรงกัน ถ้าไม่มีคู่ที่ตรงกันเลย คอลัมน์ฝั่ง `OrderItem` จะเป็น `NULL` ทั้งหมด — การกรอง `OrderItem.order_item_id.is_(None)` จึงหมายถึง "สินค้าที่ไม่เคยมีใครสั่งเลย" (เทียบเท่า SQL: `SELECT p.* FROM products p LEFT JOIN order_items oi ON oi.product_id = p.product_id WHERE oi.order_item_id IS NULL`)

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 7</strong>: ปัญหา N+1 query คืออะไร ยกตัวอย่างโค้ดที่มีปัญหานี้ใน Django ORM แล้วแก้ไข</summary>

**เฉลย**:

**ปัญหา N+1 query** คือการที่โค้ดยิง query 1 ครั้งเพื่อดึง object หลัก (เช่นลูกค้าทั้งหมด N คน) แล้วยิง query เพิ่มอีก N ครั้งเพื่อดึงข้อมูลที่เกี่ยวข้องของแต่ละ object (เช่น orders ของลูกค้าแต่ละคน) รวมเป็น N+1 query ทั้งหมด แทนที่จะรวมเป็นไม่กี่ query

```python
# มีปัญหา N+1 — ถ้ามีลูกค้า 100 คน จะยิง query ทั้งหมด 101 ครั้ง
customers = Customer.objects.all()
for customer in customers:
    print(customer.first_name, customer.orders.count())   # แต่ละบรรทัดคือ query แยก

# แก้ด้วย prefetch_related — ยิง query แค่ 2 ครั้งรวม ไม่ว่าจะมีลูกค้ากี่คน
customers = Customer.objects.prefetch_related("orders")
for customer in customers:
    print(customer.first_name, customer.orders.count())   # ใช้ข้อมูลที่ prefetch มาแล้ว ไม่ query ซ้ำ
```

หมายเหตุ: `customer.orders.count()` หลัง `prefetch_related` ยังคง query ฐานข้อมูลจริงถ้าเขียนแบบตรงไปตรงมา (เพราะ `.count()` เป็น aggregate call ใหม่) วิธีที่ถูกต้องกว่าคือใช้ `len(customer.orders.all())` หลัง prefetch หรือ annotate จำนวนไว้ล่วงหน้าด้วย `annotate(order_count=Count("orders"))` เพื่อให้แน่ใจว่าไม่มี query เพิ่มเติมเลย:

```python
from django.db.models import Count

customers = Customer.objects.annotate(order_count=Count("orders"))
for customer in customers:
    print(customer.first_name, customer.order_count)   # 1 query เดียว รวม JOIN + GROUP BY
```

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 8</strong>: เขียน Alembic migration ที่เพิ่มคอลัมน์ `phone_number` ให้ตาราง `customers` แบบปลอดภัยสำหรับ production (ไม่ lock ตารางนาน)</summary>

**เฉลย**:

```python
"""add phone_number to customers"""
from alembic import op
import sqlalchemy as sa

revision = "b2c3d4e5f6a7"
down_revision = "a1b2c3d4e5f6"


def upgrade() -> None:
    # เพิ่มคอลัมน์แบบ nullable ก่อนเสมอ — การเพิ่มคอลัมน์ nullable ใน PostgreSQL สมัยใหม่ (11+)
    # เป็น metadata-only change ไม่ต้อง rewrite ตารางทั้งหมด จึงเร็วและ lock สั้นมาก
    op.add_column("customers", sa.Column("phone_number", sa.String(20), nullable=True))


def downgrade() -> None:
    op.drop_column("customers", "phone_number")
```

ถ้าต้องการบังคับ `NOT NULL` ในภายหลัง (หลัง backfill ข้อมูลเก่าเสร็จแล้ว) ให้แยกเป็น migration คนละไฟล์ตามหลัก expand/contract ที่อธิบายใน Step 878:

```python
def upgrade() -> None:
    op.execute(
        "ALTER TABLE customers ADD CONSTRAINT phone_number_not_null "
        "CHECK (phone_number IS NOT NULL) NOT VALID"
    )
    op.execute("ALTER TABLE customers VALIDATE CONSTRAINT phone_number_not_null")
```

การแยกเป็นหลาย migration แบบนี้ทำให้แต่ละขั้นตอน lock ตารางสั้นที่สุดเท่าที่จะทำได้ และสามารถ deploy แอปพลิเคชันคู่ขนานไปกับ migration ได้โดยไม่มี downtime

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 9</strong>: เปรียบเทียบโค้ดสร้างลูกค้าใหม่พร้อมออเดอร์แรกในหนึ่ง transaction ระหว่าง psycopg และ SQLAlchemy ORM</summary>

**เฉลย**:

```python
# --- psycopg ---
import psycopg

def create_customer_with_first_order(conn: psycopg.Connection, first_name: str, email: str) -> tuple[int, int]:
    with conn.transaction():
        with conn.cursor() as cur:
            cur.execute(
                "INSERT INTO customers (first_name, email) VALUES (%s, %s) RETURNING customer_id",
                (first_name, email),
            )
            customer_id = cur.fetchone()[0]

            cur.execute(
                "INSERT INTO orders (customer_id, status) VALUES (%s, %s) RETURNING order_id",
                (customer_id, "pending"),
            )
            order_id = cur.fetchone()[0]
    return customer_id, order_id
```

```python
# --- SQLAlchemy ORM ---
from sqlalchemy.orm import Session

def create_customer_with_first_order(session: Session, first_name: str, email: str) -> tuple[int, int]:
    customer = Customer(first_name=first_name, email=email)
    order = Order(status="pending")
    customer.orders.append(order)   # ORM ผูก customer_id ให้อัตโนมัติตอน flush

    session.add(customer)
    session.commit()   # commit เดียว บันทึกทั้ง customer และ order ใน transaction เดียวกัน

    return customer.customer_id, order.order_id
```

**ข้อสังเกต**: เวอร์ชัน psycopg ต้องเขียน SQL สองคำสั่งและจัดการ `RETURNING` เอง ส่วนเวอร์ชัน SQLAlchemy ORM ใช้ relationship (`customer.orders.append(order)`) ให้ ORM คำนวณและ generate INSERT สองคำสั่งให้อัตโนมัติ พร้อมผูก foreign key ให้เอง — ทั้งสองแบบยังคงอยู่ใน transaction เดียวกัน (atomic) เหมือนกัน เพียงแต่ต่างระดับของ abstraction

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 10</strong>: ทำไมการดึง object ทั้งหมดมาคำนวณสรุปยอดใน Python (เช่นเวอร์ชัน `build_report_via_relationship` ใน Step 880) จึงไม่เหมาะกับ dataset ขนาดใหญ่ ให้อธิบายอย่างน้อย 3 เหตุผล</summary>

**เฉลย**:

1. **ปริมาณข้อมูลที่ส่งผ่าน network มากเกินจำเป็น** — แทนที่จะให้ PostgreSQL คำนวณตัวเลขสรุป (`count()`, `sum()`) แล้วส่งกลับมาแค่ไม่กี่แถว โค้ดนี้ดึง **ทุกแถวของทุก order ของทุกลูกค้า** ผ่าน network มาที่ฝั่ง Python ก่อน แล้วค่อยกรองและนับทีหลัง — ถ้ามีออเดอร์หลักล้านแถว การส่งข้อมูลทั้งหมดนี้ผ่าน network คือคอขวดที่ใหญ่ที่สุด

2. **ใช้หน่วยความจำฝั่งแอปพลิเคชันสูงเกินจำเป็น** — object ของ Python (โดยเฉพาะ ORM object ที่มี overhead ของ identity map, tracking state ฯลฯ) กินหน่วยความจำมากกว่าข้อมูลดิบหลายเท่า การโหลด object นับล้านตัวเข้าหน่วยความจำเสี่ยงทำให้แอปพลิเคชัน OOM (out of memory) หรือ GC ทำงานหนักจนช้าลงมาก

3. **ไม่ได้ใช้ประโยชน์จาก index และ query planner ของฐานข้อมูล** — PostgreSQL มี index, statistics และ query planner ที่ออกแบบมาเพื่อคำนวณ aggregate อย่างมีประสิทธิภาพ (เช่นใช้ index-only scan, parallel aggregate) การย้ายงานคำนวณไปทำใน Python คือการทิ้งความสามารถเหล่านี้ไปโดยเปล่าประโยชน์ ทำให้ CPU ของเครื่อง Python ต้องทำงานหนักแทนที่จะให้ database engine ที่ optimize มาเพื่องานนี้โดยเฉพาะเป็นผู้ทำ

4. **(เพิ่มเติม) เสี่ยงต่อความไม่สอดคล้องของข้อมูลระหว่างทาง (race condition)** — ถ้าดึงข้อมูลมาคำนวณใน Python ใช้เวลานาน ข้อมูลในฐานข้อมูลอาจเปลี่ยนไปแล้วระหว่างที่ query ยังไม่จบ (โดยเฉพาะถ้าไม่ได้ query ภายใน transaction ที่มี isolation level เหมาะสม) ทำให้ผลสรุปที่ได้ไม่ตรงกับสถานะจริงของฐานข้อมูล ณ ขณะใดขณะหนึ่ง

**บทสรุป**: หลักการทั่วไปคือ "ให้ฐานข้อมูลทำงานที่ฐานข้อมูลถนัด (filter, join, aggregate) แล้วส่งเฉพาะผลลัพธ์ที่จำเป็นกลับมาที่แอปพลิเคชัน" — ใช้ object-relationship ของ ORM สำหรับกรณีที่ต้องการ business object จริง ๆ (เช่นแสดงรายละเอียดออเดอร์ 1 ใบ) แต่สำหรับรายงาน/aggregate ควรเขียนเป็น query ที่ให้ PostgreSQL คำนวณให้เสมอ

</details>

---

## บทถัดไป

เมื่อเข้าใจการเชื่อมต่อ PostgreSQL จาก Python ทั้งสามระดับแล้ว บทถัดไปจะพาไปดูการเชื่อมต่อ PostgreSQL จากภาษา **Go** ซึ่งมีแนวคิดเรื่อง driver, connection pool และ ORM ที่คล้ายกันแต่ต่างในรายละเอียดของ ecosystem — ไปต่อที่ [Part 089: เชื่อมต่อ PostgreSQL กับ Go](./part-089-go-integration.md)