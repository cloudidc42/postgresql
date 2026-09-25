# Part 020: โปรเจกต์รวมระดับพื้นฐาน — ระบบจัดการห้องสมุด (Library Management System)

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับพื้นฐาน | Part 020 (โปรเจกต์ปิดท้ายระดับพื้นฐาน)

Steps 191–200 ของหลักสูตร คือบทปิดท้ายของ **ระดับพื้นฐาน (Foundations)** ซึ่งครอบคลุม Part 001 ถึง Part 019 ที่ผ่านมา บทนี้จะไม่มีความรู้ใหม่ที่เป็นคำสั่ง SQL แปลกใหม่ แต่จะเป็น **โปรเจกต์แบบครบวงจร (capstone project)** ที่บังคับให้เราหยิบทุกอย่างที่เรียนมาตั้งแต่ Part 001 (Database คืออะไร) จนถึง Part 019 (ALTER TABLE) มาใช้งานร่วมกันในสถานการณ์จริงเพียงสถานการณ์เดียว นั่นคือการสร้าง **ระบบจัดการห้องสมุด (Library Management System)** ตั้งแต่ศูนย์

โปรเจกต์นี้เป็นฐานข้อมูลใหม่ทั้งหมด แยกออกจากฐานข้อมูลร้านกาแฟ (coffee shop) ที่เราอาจเคยเห็นตัวอย่างผ่าน ๆ ใน Part ก่อนหน้า เพื่อให้ผู้เรียนได้ฝึกออกแบบตั้งแต่การเก็บ requirement, วาด ER diagram, เขียน DDL (Data Definition Language), ใส่ข้อมูลตัวอย่าง, ไปจนถึงเขียน query เพื่อตอบคำถามทางธุรกิจจริง ๆ ของห้องสมุด

## เป้าหมายการเรียนรู้

เมื่อเรียนจบ Part นี้ ผู้เรียนจะสามารถ:

- อธิบายขั้นตอนการเก็บ requirement และแปลง requirement ทางธุรกิจให้เป็นแบบจำลองข้อมูล (data model) ได้
- วาดและอ่าน ER diagram แบบ ASCII เพื่อสื่อสารความสัมพันธ์ระหว่างตารางได้
- สร้างฐานข้อมูลและ schema ใหม่ตั้งแต่ต้นด้วย `CREATE DATABASE` และ `CREATE SCHEMA` พร้อมอธิบายเหตุผลของการแยก schema
- ออกแบบตารางที่มีความสัมพันธ์แบบ one-to-many และ many-to-one โดยใช้ `PRIMARY KEY`, `FOREIGN KEY`, `UNIQUE`, `CHECK`, `DEFAULT`, `NOT NULL` ได้อย่างเหมาะสมกับสถานการณ์จริง
- แยกแยะความแตกต่างระหว่าง "หนังสือ" (book — ข้อมูลทั่วไปของชื่อเรื่อง) กับ "สำเนาหนังสือ" (book copy — เล่มจริงที่จับต้องได้บนชั้น) ซึ่งเป็นรูปแบบการออกแบบที่พบได้บ่อยในระบบจริง
- เขียนคำสั่ง `INSERT` เพื่อใส่ข้อมูลตัวอย่างจำนวนมากอย่างสมจริงและสอดคล้องกัน (referential integrity)
- เขียน query ระดับ `SELECT` พร้อม `WHERE`, `ORDER BY`, `LIMIT` เพื่อตอบคำถามเชิงธุรกิจ เช่น หนังสือที่ว่างให้ยืม สมาชิกที่ค้างคืนหนังสือเกินกำหนด และการคำนวณค่าปรับ
- ทบทวนภาพรวมทั้งหมดของระดับพื้นฐาน และประเมินความพร้อมของตนเองก่อนขึ้นสู่ระดับกลาง (Intermediate) ที่จะเริ่มด้วยเรื่อง `JOIN`

> **หมายเหตุเรื่องเวอร์ชัน:** โค้ดทั้งหมดในบทนี้เขียนและทดสอบแนวคิดให้ใช้งานได้กับ PostgreSQL 16 และ 17 หากใช้เวอร์ชันเก่ากว่านี้ คำสั่งส่วนใหญ่ยังคงใช้ได้ปกติ เพราะเป็นคำสั่งพื้นฐานของ SQL/DDL ที่เสถียรมานาน

---

## Step 191: ภาพรวมโปรเจกต์และ Requirement Gathering

### 191.1 ทำไมต้อง "เก็บ requirement" ก่อนเขียนโค้ด

มือใหม่หลายคนพอเรียนคำสั่ง SQL ได้แล้วมักจะรีบเปิดโปรแกรมแล้วพิมพ์ `CREATE TABLE` ทันที แต่ในงานจริง ขั้นตอนที่สำคัญที่สุดก่อนออกแบบฐานข้อมูลคือการ **เก็บ requirement (ความต้องการ)** — คือการถามตัวเองหรือถาม "เจ้าของระบบ" ว่า

- ระบบนี้มีไว้เพื่อแก้ปัญหาอะไร
- ใครเป็นผู้ใช้งาน (user) ของระบบ
- ระบบต้อง "ทำอะไรได้บ้าง" (functional requirement)
- มีกฎเกณฑ์ทางธุรกิจ (business rule) อะไรบ้างที่ฐานข้อมูลต้องบังคับใช้

ถ้าข้ามขั้นตอนนี้ไป มักจะได้ผลลัพธ์เป็นตารางที่ขาดคอลัมน์สำคัญ หรือมีความสัมพันธ์ผิดตั้งแต่ต้น แล้วต้องมาตาม `ALTER TABLE` แก้ทีหลัง (ซึ่งเราเพิ่งเรียนเรื่องนี้ไปใน Part 019 — แก้ได้ แต่ควรเลี่ยงถ้าออกแบบตั้งแต่แรกได้ดีกว่า)

### 191.2 สมมติสถานการณ์: ห้องสมุดประชาชนแห่งหนึ่ง

สมมติว่าเราได้รับมอบหมายให้ออกแบบฐานข้อมูลให้ "ห้องสมุดประชาชนสมมติ" (Somtiwa Public Library) ซึ่งปัจจุบันยังใช้สมุดบันทึกด้วยมือ ทางห้องสมุดต้องการระบบที่ทำให้:

1. **ค้นหาหนังสือได้** — ค้นตามชื่อเรื่อง ผู้แต่ง หรือหมวดหมู่
2. **รู้ว่าหนังสือเล่มไหนว่าง เล่มไหนถูกยืมอยู่** — เพราะหนังสือชื่อเดียวกันอาจมีหลายเล่ม (หลายสำเนา)
3. **จัดการสมาชิก** — รับสมัครสมาชิกใหม่ เก็บข้อมูลติดต่อ และประเภทสมาชิก
4. **บันทึกการยืม-คืน** — วันที่ยืม วันครบกำหนดคืน วันที่คืนจริง
5. **คำนวณค่าปรับ** — เมื่อคืนหนังสือช้ากว่ากำหนด ต้องคิดค่าปรับเป็นรายวัน และต้องติดตามได้ว่าใครจ่ายค่าปรับแล้วหรือยัง
6. **ออกรายงาน** — เช่น หนังสือที่ได้รับความนิยมสูงสุด สมาชิกที่ค้างคืนหนังสือเกินกำหนด ประวัติการยืมของสมาชิกแต่ละคน

### 191.3 แปลง requirement เป็นรายการ entity เบื้องต้น

จากข้อความข้างต้น เราสามารถ "ขีดเส้นใต้คำนาม" ที่ดูเหมือนจะเป็นสิ่งที่ต้องเก็บข้อมูล (entity) ได้ดังนี้:

| คำนามที่พบใน requirement | แปลว่าเป็น entity ว่า |
|---|---|
| ผู้แต่ง | `authors` |
| หนังสือ, ชื่อเรื่อง | `books` |
| สำเนาหนังสือ, เล่มที่ยืมได้จริง | `book_copies` |
| หมวดหมู่ | `categories` |
| สมาชิก | `members` |
| การยืม-คืน | `loans` |
| ค่าปรับ | `fines` |

รายการ entity ทั้ง 7 ตัวนี้คือสิ่งที่เราจะนำไปออกแบบเป็น ER diagram ใน Step 192 และสร้างเป็นตารางจริงใน Step 194–196

### 191.4 กฎเกณฑ์ทางธุรกิจ (Business Rules) ที่ฐานข้อมูลต้องบังคับใช้

นอกจาก entity แล้ว requirement ยังบอกกฎเกณฑ์ที่เราควรบังคับด้วย constraint ต่าง ๆ ที่เรียนมาใน Part 016–018:

- หนังสือ 1 ชื่อเรื่องมีได้หลายสำเนา (copy) → ความสัมพันธ์ one-to-many ระหว่าง `books` กับ `book_copies`
- สำเนาหนังสือ 1 เล่ม ในเวลาหนึ่ง ๆ ถูกยืมได้โดยสมาชิกเดียวเท่านั้น (ไม่ให้ยืมซ้อนกัน) → จะบังคับด้วย logic ระดับแอปพลิเคชันและ `status` ของสำเนา
- วันครบกำหนดคืน (`due_date`) ต้องมาหลังวันที่ยืม (`loan_date`) เสมอ → ใช้ `CHECK (due_date > loan_date)`
- อีเมลสมาชิกต้องไม่ซ้ำกัน → ใช้ `UNIQUE`
- เลข ISBN ของหนังสือต้องไม่ซ้ำกัน → ใช้ `UNIQUE`
- ค่าปรับต้องไม่ติดลบ → ใช้ `CHECK (amount >= 0)`
- สถานะของสำเนาหนังสือต้องเป็นค่าที่กำหนดไว้เท่านั้น เช่น `available`, `borrowed`, `lost` → ใช้ `CHECK (status IN (...))`
- ทุกแถวควรมี default ที่สมเหตุสมผล เช่น วันที่ยืมควรเป็นวันนี้โดยอัตโนมัติถ้าไม่ระบุ → ใช้ `DEFAULT CURRENT_DATE`

การไล่เรียง requirement มาเป็นกฎเกณฑ์แบบนี้คือสิ่งที่ทำให้ฐานข้อมูลของเรา "ป้องกันข้อมูลผิดพลาดได้ตั้งแต่ชั้น database" ไม่ต้องหวังพึ่งแอปพลิเคชันฝั่งเดียว ซึ่งเป็นแนวคิดสำคัญที่เราเรียนมาตลอด Part 016–018

### 191.5 ขอบเขตของโปรเจกต์ (Scope)

เพื่อให้โปรเจกต์นี้อยู่ในระดับ "พื้นฐาน" (ยังไม่ใช้ JOIN, subquery ซับซ้อน, index ขั้นสูง, transaction ฯลฯ ซึ่งจะไปเรียนในระดับกลางและสูงต่อไป) เราจะ**จำกัดขอบเขต**ไว้ดังนี้:

**อยู่ในขอบเขต (in scope):**
- ตารางหลัก 7 ตาราง พร้อม constraint ครบถ้วน
- ข้อมูลตัวอย่างสมจริง
- query พื้นฐานที่ใช้ `WHERE`, `ORDER BY`, `LIMIT`, การคำนวณด้วยฟังก์ชันวันที่/ตัวเลขง่าย ๆ, `NULL` handling ด้วย `IS NULL` / `COALESCE`

**นอกขอบเขต (out of scope — จะเรียนในระดับกลาง/สูงต่อไป):**
- การเขียน query ด้วย `JOIN` หลายตาราง (เริ่มที่ Part 021)
- `GROUP BY`, aggregate function ขั้นสูง, window function
- Index การปรับแต่งประสิทธิภาพ (performance tuning)
- Transaction, concurrency, locking
- Stored procedure / trigger

ดังนั้น query บางส่วนใน Step 198–199 ที่ "ในอุดมคติ" ควรใช้ `JOIN` เราจะเขียนแบบง่ายที่สุดเท่าที่ทำได้ด้วยความรู้ระดับพื้นฐาน (เช่น query แยกทีละตาราง หรือใช้ subquery แบบง่ายเท่าที่จำเป็น) แล้วจะ "แนะนำไว้ล่วงหน้า" ว่าพอเรียน `JOIN` ใน Part 021 แล้วจะเขียน query เดียวกันนี้ได้กระชับและทรงพลังกว่ามาก

---

## Step 192: ออกแบบ ER Diagram

### 192.1 ER Diagram คืออะไร (ทบทวน)

ER Diagram (Entity-Relationship Diagram) คือแผนภาพที่แสดง entity (ซึ่งจะกลายเป็นตาราง) และความสัมพันธ์ระหว่าง entity เหล่านั้น สัญลักษณ์ที่ใช้บ่อยคือ:

- `||` หมายถึง "หนึ่งเดียวเท่านั้น (exactly one)"
- `o{` หรือ `}o` หมายถึง "ศูนย์หรือหลาย (zero or many)"
- `||--o{` อ่านว่า หนึ่งฝั่งซ้ายสัมพันธ์กับศูนย์-ถึง-หลายฝั่งขวา (one-to-many)

### 192.2 ER Diagram แบบ ASCII ของระบบห้องสมุด

```
┌───────────────┐          ┌───────────────┐          ┌───────────────┐
│    authors    │          │  categories   │          │     books     │
├───────────────┤          ├───────────────┤          ├───────────────┤
│ author_id PK  │──┐       │category_id PK │──┐       │ book_id    PK │
│ first_name    │  │       │category_name  │  │       │ isbn      UQ  │
│ last_name     │  │       │description    │  │       │ title         │
│ birth_year    │  │       └───────────────┘  │       │ author_id  FK │◄─┐
│ nationality   │  │                          │       │ category_id FK│◄─┼─┐
│ created_at    │  │                          │       │ publication_yr│  │ │
└───────────────┘  │                          │       │ publisher     │  │ │
                    │                          │       │ language      │  │ │
                    │                          │       │ page_count    │  │ │
                    │                          │       │ price         │  │ │
                    │                          │       │ created_at    │  │ │
                    │                          │       └───────┬───────┘  │ │
                    └──────────────────────────┼──────────────┘          │ │
                                                 └─────────────────────────┘ │
                    (author_id 1 คน เขียนได้หลายเล่ม)                        │
                    (category_id 1 หมวด มีได้หลายเล่ม) ───────────────────────┘

        books (1) ──────────< book_copies (many)
┌───────────────┐          ┌────────────────┐          ┌───────────────┐
│     books     │          │  book_copies    │          │    members    │
├───────────────┤          ├────────────────┤          ├───────────────┤
│ book_id    PK │◄─┐       │ copy_id     PK  │          │ member_id  PK │
│ ...           │  │       │ book_id     FK  │──┘       │ member_code UQ│
└───────────────┘  └───────│ copy_number     │          │ first_name    │
                            │ status  CHECK   │          │ last_name     │
                            │ shelf_location  │          │ email      UQ │
                            │ acquired_date   │          │ phone         │
                            │ created_at      │          │ membership_ty │
                            └────────┬────────┘          │ join_date     │
                                     │                    │ is_active     │
                                     │                    │ created_at    │
                                     │                    └───────┬───────┘
                                     │                            │
                                     │  book_copies (1) ──< loans │
                                     │  members (1) ──< loans     │
                                     ▼                            ▼
                              ┌─────────────────────────────────────┐
                              │                loans                 │
                              ├───────────────────────────────────────┤
                              │ loan_id     PK                        │
                              │ copy_id     FK ──► book_copies        │
                              │ member_id   FK ──► members            │
                              │ loan_date      DEFAULT CURRENT_DATE   │
                              │ due_date       CHECK > loan_date      │
                              │ return_date    NULL ได้ (ยังไม่คืน)   │
                              │ status         CHECK                  │
                              │ created_at                            │
                              └───────────────────┬────────────────────┘
                                                   │
                                    loans (1) ──< fines (many, ปกติ 0 หรือ 1)
                                                   ▼
                                          ┌───────────────────┐
                                          │       fines        │
                                          ├─────────────────────┤
                                          │ fine_id      PK     │
                                          │ loan_id      FK     │
                                          │ amount   CHECK >=0  │
                                          │ reason              │
                                          │ is_paid  DEFAULT f  │
                                          │ paid_date           │
                                          │ created_at          │
                                          └─────────────────────┘
```

### 192.3 สรุปความสัมพันธ์ทั้งหมด (Relationship Summary)

| ตารางต้นทาง | ความสัมพันธ์ | ตารางปลายทาง | คำอธิบาย |
|---|---|---|---|
| `authors` | 1 : many | `books` | ผู้แต่ง 1 คน เขียนหนังสือได้หลายเล่ม, หนังสือ 1 เล่มมีผู้แต่งหลักได้ 1 คน (โมเดลนี้ทำให้เรียบง่ายสุดในระดับพื้นฐาน) |
| `categories` | 1 : many | `books` | หมวดหมู่ 1 หมวด มีหนังสือได้หลายเล่ม |
| `books` | 1 : many | `book_copies` | หนังสือ 1 ชื่อเรื่อง มีสำเนาได้หลายเล่ม (เช่น "Sapiens" มี 3 เล่มบนชั้น) |
| `book_copies` | 1 : many | `loans` | สำเนาแต่ละเล่มถูกยืมได้หลายครั้งตลอดอายุการใช้งาน (แต่ในเวลาเดียวกันยืมได้แค่ครั้งเดียว) |
| `members` | 1 : many | `loans` | สมาชิก 1 คน ยืมหนังสือได้หลายครั้ง |
| `loans` | 1 : 0 หรือ 1 | `fines` | การยืมแต่ละครั้ง อาจมีค่าปรับหรือไม่มีก็ได้ (ถ้าคืนตรงเวลาจะไม่มี fine) |

### 192.4 ทำไมต้องแยก `books` กับ `book_copies`

นี่คือจุดออกแบบที่สำคัญที่สุดของโปรเจกต์นี้ และเป็นรูปแบบ (pattern) ที่พบได้ในระบบห้องสมุด ร้านเช่า หรือระบบสินค้าคงคลังทั่วไป:

- **`books`** เก็บข้อมูล "ชื่อเรื่อง" — เช่น ชื่อหนังสือ, ผู้แต่ง, ปีพิมพ์ ซึ่งเป็นข้อมูลที่ **เหมือนกันทุกเล่ม** ไม่ว่าจะมีกี่สำเนา
- **`book_copies`** เก็บข้อมูล "เล่มที่จับต้องได้จริง" บนชั้นหนังสือ — แต่ละเล่มมีสถานะของตัวเอง (ว่าง/ถูกยืม/สูญหาย) และมีตำแหน่งชั้นวางของตัวเอง

ถ้าเราออกแบบผิดโดยไม่แยกสองตารางนี้ (เช่น ใส่ `status` ไว้ในตาราง `books` เลย) จะเกิดปัญหาทันทีเมื่อห้องสมุดมีหนังสือชื่อเดียวกัน 3 เล่ม เพราะจะไม่สามารถบอกได้ว่า "เล่มไหน" ถูกยืมอยู่ เล่มไหนว่าง — นี่คือเหตุผลเชิงออกแบบที่ทำให้เราต้องมีทั้งสองตารางแยกกัน

---

## Step 193: สร้างฐานข้อมูลและ Schema

### 193.1 สร้างฐานข้อมูลใหม่

เราจะสร้างฐานข้อมูลใหม่ชื่อ `library_db` แยกต่างหากจากฐานข้อมูลอื่น ๆ ที่เคยสร้างในบทก่อนหน้า (เช่น coffee shop) เพื่อไม่ให้ตารางปนกัน:

```sql
-- รันคำสั่งนี้ขณะเชื่อมต่อกับฐานข้อมูลเริ่มต้น เช่น postgres
CREATE DATABASE library_db
    WITH
    ENCODING = 'UTF8'
    LC_COLLATE = 'en_US.UTF-8'
    LC_CTYPE = 'en_US.UTF-8'
    TEMPLATE = template0;

COMMENT ON DATABASE library_db IS 'ฐานข้อมูลระบบจัดการห้องสมุด - โปรเจกต์ปิดท้ายระดับพื้นฐาน Part 020';
```

จากนั้นให้เชื่อมต่อเข้าไปยังฐานข้อมูลใหม่นี้ (ใน `psql` ใช้คำสั่ง `\c library_db`):

```sql
\c library_db
```

### 193.2 ทำไมต้อง `CREATE SCHEMA` เพิ่มอีกชั้น

Schema คือ "เนมสเปซ (namespace)" ที่อยู่ภายในฐานข้อมูลหนึ่ง ๆ ใช้จัดกลุ่มตารางให้เป็นระเบียบ ประโยชน์ของการสร้าง schema เฉพาะสำหรับระบบห้องสมุด (แทนที่จะสร้างตารางทั้งหมดไว้ใน schema `public` เฉย ๆ) มีดังนี้:

1. **แยกกลุ่มข้อมูลอย่างชัดเจน** — ถ้าในอนาคตฐานข้อมูลนี้ต้องรวมกับระบบอื่น (เช่นระบบจองห้องประชุมของห้องสมุด) จะไม่มีตารางชนกัน
2. **ตั้งสิทธิ์ (permission) แยกกลุ่มได้ง่าย** — เช่นให้สิทธิ์ `library` schema กับทีมพัฒนาเฉพาะกลุ่ม
3. **เป็นธรรมเนียมปฏิบัติที่ดี (best practice)** ในการทำงานระดับองค์กร ที่ไม่ควรใช้ `public` schema แบบไม่เป็นระบบ

```sql
CREATE SCHEMA IF NOT EXISTS library
    AUTHORIZATION CURRENT_USER;

COMMENT ON SCHEMA library IS 'Schema หลักของระบบจัดการห้องสมุด เก็บตารางทั้งหมดของโปรเจกต์นี้';
```

### 193.3 ตั้งค่า `search_path` เพื่อความสะดวก

`search_path` คือลำดับที่ PostgreSQL จะค้นหาตาราง เมื่อเราไม่ได้ระบุชื่อ schema นำหน้าชื่อตาราง การตั้งค่านี้ทำให้เราพิมพ์คำสั่งสั้นลง (พิมพ์ `books` แทนที่จะต้องพิมพ์ `library.books` ทุกครั้ง):

```sql
SET search_path TO library, public;
```

> **ข้อควรระวัง:** คำสั่ง `SET search_path` แบบนี้มีผลแค่ session ปัจจุบันเท่านั้น หากต้องการให้มีผลถาวรสำหรับผู้ใช้คนหนึ่ง ๆ ทุกครั้งที่เชื่อมต่อ สามารถใช้คำสั่ง `ALTER ROLE your_user SET search_path TO library, public;` แทน (เนื้อหาการจัดการ role และสิทธิ์แบบละเอียดจะอยู่ใน Part ถัด ๆ ไปของระดับกลาง/สูง)

### 193.4 ตรวจสอบว่าสร้าง schema สำเร็จ

```sql
-- แสดงรายชื่อ schema ทั้งหมดในฐานข้อมูลนี้
SELECT schema_name
FROM information_schema.schemata
WHERE schema_name NOT LIKE 'pg_%'
  AND schema_name <> 'information_schema'
ORDER BY schema_name;
```

ผลลัพธ์ที่คาดหวัง:

```
 schema_name
-------------
 library
 public
(2 rows)
```

ถึงตรงนี้เราพร้อมแล้วที่จะสร้างตารางจริงในขั้นตอนถัดไป

---

## Step 194: สร้างตาราง `authors`, `categories`, `books`

เราจะเริ่มสร้างตารางตามลำดับที่ "ไม่มี dependency ก่อน" คือ `authors` และ `categories` ก่อน เพราะตาราง `books` ที่จะสร้างตามมาต้องอ้างอิง (FOREIGN KEY) ไปยังทั้งสองตารางนี้ — หลักการนี้สำคัญมาก: **ต้องสร้างตารางฝั่ง "หนึ่ง" (the "one" side) ก่อนตารางฝั่ง "หลาย" (the "many" side) เสมอ** เพราะ FOREIGN KEY ต้องอ้างอิงไปยังตารางที่มีอยู่แล้วเท่านั้น

### 194.1 ตาราง `authors`

```sql
CREATE TABLE authors (
    author_id       SERIAL PRIMARY KEY,
    first_name      VARCHAR(100) NOT NULL,
    last_name       VARCHAR(100) NOT NULL,
    birth_year      SMALLINT
                        CHECK (birth_year IS NULL OR birth_year BETWEEN 1 AND 2100),
    nationality     VARCHAR(80),
    biography       TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

COMMENT ON TABLE authors IS 'ข้อมูลผู้แต่งหนังสือ';
COMMENT ON COLUMN authors.birth_year IS 'ปี ค.ศ. ที่เกิด อนุญาตให้เป็น NULL ถ้าไม่ทราบข้อมูล';
```

**เหตุผลของแต่ละ constraint:**

- `author_id SERIAL PRIMARY KEY` — ใช้ `SERIAL` เพื่อให้ PostgreSQL สร้างเลขลำดับอัตโนมัติ (auto-increment) เป็น `PRIMARY KEY` ซึ่งบังคับทั้ง `NOT NULL` และค่าห้ามซ้ำในตัวเองอยู่แล้ว (ทบทวนจาก Part 016)
- `first_name`, `last_name NOT NULL` — ผู้แต่งทุกคนต้องมีชื่อ-นามสกุล ห้ามเว้นว่าง
- `birth_year CHECK` — ป้องกันค่าที่ผิดตรรกะ เช่น ปีเกิดติดลบ หรือปีเกิดเกินปัจจุบันไปมาก แต่ยังอนุญาตให้เป็น `NULL` ได้เพราะผู้แต่งบางคน (เช่นนักเขียนโบราณ) อาจไม่ทราบปีเกิดที่แน่ชัด — นี่คือตัวอย่างการใช้ `NULL` อย่างถูกต้อง (ทบทวนจาก Part 015): `NULL` หมายถึง "ไม่ทราบค่า" ไม่ใช่ "ค่าเป็นศูนย์"
- `biography TEXT` — ใช้ `TEXT` แทน `VARCHAR` เพราะประวัติผู้แต่งอาจยาวไม่จำกัด ไม่จำเป็นต้องกำหนดความยาวสูงสุด
- `created_at TIMESTAMPTZ DEFAULT NOW()` — บันทึกเวลาสร้างแถวอัตโนมัติ ใช้ `TIMESTAMPTZ` (timestamp with time zone) ตามคำแนะนำที่เคยกล่าวถึงใน Part 007 เพื่อเก็บเวลาอย่างถูกต้องไม่ว่าเซิร์ฟเวอร์จะตั้ง time zone ใด

### 194.2 ตาราง `categories`

```sql
CREATE TABLE categories (
    category_id     SERIAL PRIMARY KEY,
    category_name   VARCHAR(80) NOT NULL UNIQUE,
    description     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

COMMENT ON TABLE categories IS 'หมวดหมู่ของหนังสือ เช่น นิยาย, วิทยาศาสตร์, ประวัติศาสตร์';
```

**เหตุผล:** `category_name` ใส่ `UNIQUE` เพราะห้ามมีหมวดหมู่ชื่อซ้ำกัน (เช่นห้ามมี "นิยาย" สองแถว) — การมี `UNIQUE` ในคอลัมน์ที่ไม่ใช่ primary key คือสิ่งที่เรียนไปแล้วใน Part 016

### 194.3 ตาราง `books`

```sql
CREATE TABLE books (
    book_id             SERIAL PRIMARY KEY,
    isbn                VARCHAR(20) NOT NULL UNIQUE,
    title               VARCHAR(255) NOT NULL,
    author_id           INTEGER NOT NULL
                            REFERENCES authors (author_id)
                            ON UPDATE CASCADE
                            ON DELETE RESTRICT,
    category_id         INTEGER
                            REFERENCES categories (category_id)
                            ON UPDATE CASCADE
                            ON DELETE SET NULL,
    publication_year    SMALLINT
                            CHECK (publication_year BETWEEN 1400 AND 2100),
    publisher           VARCHAR(150),
    language             VARCHAR(40) NOT NULL DEFAULT 'Thai',
    page_count          INTEGER CHECK (page_count IS NULL OR page_count > 0),
    price               NUMERIC(8,2) CHECK (price IS NULL OR price >= 0),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

COMMENT ON TABLE books IS 'ข้อมูลชื่อเรื่องของหนังสือ (ไม่ใช่เล่มจริง - ดู book_copies สำหรับเล่มจริง)';
COMMENT ON COLUMN books.isbn IS 'International Standard Book Number ต้องไม่ซ้ำกัน';
```

**เหตุผลของแต่ละ constraint:**

- `isbn VARCHAR(20) UNIQUE` — ISBN เป็นรหัสมาตรฐานสากลที่ต้องไม่ซ้ำกัน ใช้ `VARCHAR` แทน `INTEGER` เพราะ ISBN มีเครื่องหมายขีด (`-`) ปนอยู่ เช่น `978-616-01-1234-5` และบางครั้งอาจมี `X` ต่อท้าย (ทบทวนหลักการเลือกชนิดข้อมูลจาก Part 006 — "รหัส" ที่ไม่ได้ใช้คำนวณควรเก็บเป็น text ไม่ใช่ตัวเลข)
- `author_id INTEGER NOT NULL REFERENCES authors(author_id)` — นี่คือ **FOREIGN KEY** (ทบทวนจาก Part 017) หนังสือทุกเล่มต้องมีผู้แต่ง จึงใส่ `NOT NULL`
  - `ON UPDATE CASCADE` — ถ้า `author_id` ของผู้แต่งในตาราง `authors` ถูกเปลี่ยน (พบได้น้อยเพราะเป็น auto-increment แต่ใส่ไว้เพื่อความสมบูรณ์) ให้ปรับค่าตามในตาราง `books` โดยอัตโนมัติ
  - `ON DELETE RESTRICT` — ห้ามลบผู้แต่งถ้ายังมีหนังสือของผู้แต่งคนนั้นอยู่ในระบบ (ป้องกันข้อมูลกำพร้า / orphaned data)
- `category_id INTEGER REFERENCES categories(category_id) ON DELETE SET NULL` — หนังสือ**ไม่บังคับ**ต้องมีหมวดหมู่ (อนุญาต `NULL`ได้ เช่นหนังสือที่ยังไม่จัดหมวดหมู่) และถ้าหมวดหมู่นั้นถูกลบไป ให้ตั้งค่าเป็น `NULL` แทนที่จะลบหนังสือทิ้งไปด้วย — นี่คือตัวอย่างการเลือกใช้ `ON DELETE` ที่ต่างกันตามความหมายทางธุรกิจของแต่ละความสัมพันธ์ (ทบทวนจาก Part 017)
- `publication_year CHECK (... BETWEEN 1400 AND 2100)` — ป้องกันปีพิมพ์ที่ผิดตรรกะ (เช่น ปีติดลบ หรือปีในอนาคตไกลเกินจริง) ตั้งขอบเขตกว้าง ๆ ให้ครอบคลุมหนังสือโบราณได้ด้วย
- `language VARCHAR(40) NOT NULL DEFAULT 'Thai'` — ใช้ `DEFAULT` (ทบทวนจาก Part 018) เพราะหนังสือส่วนใหญ่ในห้องสมุดนี้เป็นภาษาไทย การตั้งค่า default ช่วยลดภาระตอน `INSERT` ข้อมูล
- `page_count CHECK (page_count IS NULL OR page_count > 0)` — จำนวนหน้าต้องเป็นบวกเท่านั้นถ้ามีการระบุ แต่ยอมให้เป็น `NULL` ได้ (ยังไม่ทราบข้อมูล) — สังเกตรูปแบบ `CHECK (col IS NULL OR col > 0)` ซึ่งเป็นแพทเทิร์นมาตรฐานเวลาต้องการอนุญาต `NULL` ควบคู่กับการบังคับเงื่อนไขตัวเลข
- `price NUMERIC(8,2)` — ใช้ `NUMERIC` แทน `FLOAT`/`REAL` สำหรับค่าเงิน (ทบทวนจาก Part 006 — ห้ามใช้ floating point กับเงินเพราะมีปัญหาความคลาดเคลื่อนจากการปัดเศษ)

### 194.4 ตรวจสอบโครงสร้างตารางที่สร้าง

```sql
\d books
```

ผลลัพธ์โดยประมาณ:

```
                                        Table "library.books"
      Column       |           Type           | Collation | Nullable |             Default
--------------------+---------------------------+-----------+----------+----------------------------------
 book_id            | integer                   |           | not null | nextval('books_book_id_seq'::regclass)
 isbn               | character varying(20)     |           | not null |
 title              | character varying(255)    |           | not null |
 author_id          | integer                   |           | not null |
 category_id        | integer                   |           |          |
 publication_year   | smallint                  |           |          |
 publisher          | character varying(150)    |           |          |
 language           | character varying(40)     |           | not null | 'Thai'::character varying
 page_count         | integer                   |           |          |
 price              | numeric(8,2)              |           |          |
 created_at         | timestamp with time zone  |           | not null | now()
Indexes:
    "books_pkey" PRIMARY KEY, btree (book_id)
    "books_isbn_key" UNIQUE CONSTRAINT, btree (isbn)
Check constraints:
    "books_page_count_check" CHECK (page_count IS NULL OR page_count > 0)
    "books_price_check" CHECK (price IS NULL OR price >= 0::numeric)
    "books_publication_year_check" CHECK (publication_year BETWEEN 1400 AND 2100)
Foreign-key constraints:
    "books_author_id_fkey" FOREIGN KEY (author_id) REFERENCES authors(author_id) ON UPDATE CASCADE ON DELETE RESTRICT
    "books_category_id_fkey" FOREIGN KEY (category_id) REFERENCES categories(category_id) ON UPDATE CASCADE ON DELETE SET NULL
```

---

## Step 195: สร้างตาราง `book_copies` และ `members`

### 195.1 ตาราง `book_copies`

ตารางนี้คือหัวใจของการออกแบบที่กล่าวถึงใน Step 192.4 — แต่ละแถวในตารางนี้แทน "เล่มจริง 1 เล่ม" ที่วางอยู่บนชั้นหนังสือ

```sql
CREATE TABLE book_copies (
    copy_id         SERIAL PRIMARY KEY,
    book_id         INTEGER NOT NULL
                        REFERENCES books (book_id)
                        ON UPDATE CASCADE
                        ON DELETE CASCADE,
    copy_number     INTEGER NOT NULL CHECK (copy_number > 0),
    status          VARCHAR(20) NOT NULL DEFAULT 'available'
                        CHECK (status IN ('available', 'borrowed', 'lost', 'damaged', 'under_repair')),
    shelf_location  VARCHAR(30),
    acquired_date   DATE NOT NULL DEFAULT CURRENT_DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    -- ห้ามมีสำเนาหมายเลขซ้ำกันภายในหนังสือเล่มเดียวกัน เช่น book_id=5 จะมี copy_number=1 ได้แค่แถวเดียว
    CONSTRAINT uq_book_copy_number UNIQUE (book_id, copy_number)
);

COMMENT ON TABLE book_copies IS 'สำเนาหนังสือแต่ละเล่มที่จับต้องได้จริงบนชั้น หนึ่ง book_id มีได้หลาย copy';
COMMENT ON COLUMN book_copies.status IS 'สถานะปัจจุบันของสำเนาเล่มนี้: available, borrowed, lost, damaged, under_repair';
```

**เหตุผลของแต่ละ constraint:**

- `book_id ... ON DELETE CASCADE` — ต่างจาก `books.author_id` ที่ใช้ `RESTRICT` ในที่นี้เราเลือกใช้ `CASCADE` เพราะถ้าหนังสือชื่อเรื่องหนึ่งถูกลบออกจากระบบทั้งหมด (เช่นห้องสมุดเลิกให้บริการหนังสือเล่มนี้) ก็สมเหตุสมผลที่จะลบสำเนาทุกเล่มของมันไปด้วย — นี่คือตัวอย่างสำคัญที่แสดงให้เห็นว่า **การเลือก `ON DELETE` แต่ละแบบต้องพิจารณาความหมายทางธุรกิจของความสัมพันธ์นั้น ๆ ไม่ใช่ใช้แบบเดียวกันหมดทุกที่**
- `copy_number CHECK (copy_number > 0)` — หมายเลขสำเนาต้องเป็นจำนวนเต็มบวก (เล่มที่ 1, 2, 3, ...)
- `CONSTRAINT uq_book_copy_number UNIQUE (book_id, copy_number)` — นี่คือ **composite unique constraint** (ทบทวนจาก Part 016) ที่บังคับว่าคู่ `(book_id, copy_number)` ต้องไม่ซ้ำกัน กล่าวคือหนังสือเล่มเดียวกันจะมี "เล่มที่ 1" ได้แค่ครั้งเดียว แต่ "เล่มที่ 1" ของหนังสือคนละชื่อเรื่องซ้ำกันได้ (เพราะนับแยกกันตาม `book_id`)
- `status ... CHECK (status IN (...))` — นี่คือการจำลอง ENUM แบบง่ายด้วย `CHECK` (ทบทวนจาก Part 018) ซึ่งเหมาะกับกรณีที่มีค่าที่เป็นไปได้จำกัดตายตัว
- `acquired_date DATE DEFAULT CURRENT_DATE` — วันที่ห้องสมุดจัดซื้อเล่มนี้เข้ามา ถ้าไม่ระบุจะใช้วันที่ปัจจุบันเป็นค่าเริ่มต้น

### 195.2 ตาราง `members`

```sql
CREATE TABLE members (
    member_id           SERIAL PRIMARY KEY,
    member_code         VARCHAR(15) NOT NULL UNIQUE,
    first_name          VARCHAR(100) NOT NULL,
    last_name           VARCHAR(100) NOT NULL,
    email               VARCHAR(150) NOT NULL UNIQUE,
    phone               VARCHAR(20),
    membership_type     VARCHAR(20) NOT NULL DEFAULT 'standard'
                            CHECK (membership_type IN ('standard', 'student', 'senior', 'vip')),
    join_date           DATE NOT NULL DEFAULT CURRENT_DATE,
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

COMMENT ON TABLE members IS 'ข้อมูลสมาชิกห้องสมุด';
COMMENT ON COLUMN members.member_code IS 'รหัสสมาชิกที่พิมพ์บนบัตรสมาชิก เช่น LIB-2026-0001';
```

**เหตุผลของแต่ละ constraint:**

- `member_code VARCHAR(15) UNIQUE` — รหัสสมาชิกที่พิมพ์บนบัตร ต้องไม่ซ้ำกัน และเก็บเป็น text เพราะมีตัวอักษรปนตัวเลข
- `email VARCHAR(150) UNIQUE` — อีเมลใช้เป็นช่องทางติดต่อหลัก ต้องไม่ซ้ำกัน (สมาชิก 1 คน 1 อีเมล)
- `membership_type CHECK` — จำกัดประเภทสมาชิกไว้ 4 แบบ ซึ่งอาจมีผลต่อ "สิทธิ์การยืม" ในอนาคต (เช่น VIP ยืมได้มากกว่า) แต่ในระดับพื้นฐานนี้เราเก็บไว้เป็นข้อมูลอ้างอิงก่อน
- `is_active BOOLEAN DEFAULT TRUE` — ใช้ชนิดข้อมูล `BOOLEAN` (ทบทวนจาก Part 006) แทนการใช้ตัวเลข 0/1 หรือ string 'Y'/'N' ทำให้อ่านง่ายและใช้ query ได้เป็นธรรมชาติ เช่น `WHERE is_active` แทนที่จะต้องเขียน `WHERE is_active = 1`
- `join_date DATE DEFAULT CURRENT_DATE` — วันที่สมัครสมาชิก ถ้าไม่ระบุจะใช้วันนี้

### 195.3 ตรวจสอบภาพรวมตารางทั้งหมดที่สร้างไปแล้ว

```sql
SELECT table_name
FROM information_schema.tables
WHERE table_schema = 'library'
ORDER BY table_name;
```

```
 table_name
--------------
 authors
 book_copies
 books
 categories
 members
(5 rows)
```

ยังเหลืออีก 2 ตาราง (`loans` และ `fines`) ซึ่งเป็นตารางที่ซับซ้อนที่สุดในระบบ เพราะต้องอ้างอิงไปหลายตารางพร้อมกัน — เราจะสร้างในขั้นตอนถัดไป

---

## Step 196: สร้างตาราง `loans` และ `fines`

### 196.1 ตาราง `loans`

ตารางนี้คือ "หัวใจของธุรกรรม" ในระบบห้องสมุด บันทึกทุกครั้งที่มีการยืม-คืนหนังสือ และต้องอ้างอิงไปทั้ง `book_copies` (ยืมสำเนาเล่มไหน) และ `members` (ใครเป็นคนยืม) — นี่คือตัวอย่างของตารางที่มี **FOREIGN KEY มากกว่าหนึ่งคอลัมน์** ซึ่งพบได้บ่อยมากในตารางที่บันทึกธุรกรรม (transaction table)

```sql
CREATE TABLE loans (
    loan_id         SERIAL PRIMARY KEY,
    copy_id         INTEGER NOT NULL
                        REFERENCES book_copies (copy_id)
                        ON UPDATE CASCADE
                        ON DELETE RESTRICT,
    member_id       INTEGER NOT NULL
                        REFERENCES members (member_id)
                        ON UPDATE CASCADE
                        ON DELETE RESTRICT,
    loan_date       DATE NOT NULL DEFAULT CURRENT_DATE,
    due_date        DATE NOT NULL,
    return_date     DATE,
    status          VARCHAR(20) NOT NULL DEFAULT 'active'
                        CHECK (status IN ('active', 'returned', 'lost', 'overdue')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    -- กฎธุรกิจสำคัญ: วันครบกำหนดคืนต้องมาหลังวันที่ยืมเสมอ
    CONSTRAINT chk_due_after_loan CHECK (due_date > loan_date),

    -- ถ้าคืนแล้ว วันที่คืนต้องไม่มาก่อนวันที่ยืม
    CONSTRAINT chk_return_after_loan CHECK (return_date IS NULL OR return_date >= loan_date)
);

COMMENT ON TABLE loans IS 'บันทึกการยืม-คืนหนังสือแต่ละครั้ง';
COMMENT ON COLUMN loans.return_date IS 'วันที่คืนจริง เป็น NULL หมายถึงยังไม่ได้คืน';
```

**เหตุผลของแต่ละ constraint:**

- `copy_id ... ON DELETE RESTRICT` และ `member_id ... ON DELETE RESTRICT` — ห้ามลบสำเนาหนังสือหรือสมาชิกทิ้งถ้ายังมีประวัติการยืมอ้างอิงอยู่ เพราะนั่นจะทำให้เสียประวัติธุรกรรม (transaction history) ซึ่งมักต้องเก็บไว้เพื่อการตรวจสอบ (audit) เสมอ — หลักการนี้ต่างจาก `book_copies` ที่ยอม `CASCADE` เมื่อลบ `books` เพราะข้อมูลธุรกรรมทางการเงิน/ประวัติสำคัญกว่าข้อมูล master data ทั่วไป
- `due_date DATE NOT NULL` — ทุกการยืมต้องมีวันครบกำหนดคืน ไม่มีข้อยกเว้น
- `return_date DATE` (ไม่มี `NOT NULL`) — คอลัมน์นี้ **ต้องอนุญาตให้เป็น `NULL` ได้** เพราะขณะที่หนังสือยังไม่ถูกคืน เรายังไม่มีวันที่จะใส่ — นี่คือตัวอย่างการใช้ `NULL` ที่ถูกต้องตามความหมาย (สื่อความหมายว่า "ยังไม่เกิดเหตุการณ์นี้") ตามที่เรียนใน Part 015
- `CONSTRAINT chk_due_after_loan CHECK (due_date > loan_date)` — นี่คือ constraint ที่โจทย์กำหนดไว้ชัดเจนใน requirement ของ Step 196 บังคับว่าวันครบกำหนดคืนต้องมาหลังวันที่ยืมเสมอ (ห้ามยืมวันนี้แต่ครบกำหนดเมื่อวาน)
- `CONSTRAINT chk_return_after_loan CHECK (return_date IS NULL OR return_date >= loan_date)` — ถ้ามีการคืนแล้ว วันที่คืนต้องไม่ย้อนไปก่อนวันที่ยืม สังเกตแพทเทิร์น `col IS NULL OR condition` อีกครั้ง ซึ่งเป็นวิธีมาตรฐานในการเขียน `CHECK` ที่ยอมให้ `NULL` ผ่านได้ แต่ถ้ามีค่าแล้วต้องผ่านเงื่อนไข

> **ทำไมไม่ใช้ `ON DELETE CASCADE` กับ FOREIGN KEY ทั้งหมดไปเลย?** เพราะแต่ละความสัมพันธ์มีความหมายทางธุรกิจต่างกัน ตารางที่เก็บ "ประวัติ/ธุรกรรม" (loans, fines) ควรป้องกันการลบข้อมูลต้นทางโดยไม่ตั้งใจด้วย `RESTRICT` ส่วนตารางที่เป็น "รายละเอียดย่อย" ของอีกตาราง (เช่น book_copies เป็นรายละเอียดของ books) มักใช้ `CASCADE` ได้อย่างปลอดภัยกว่า — การเลือก `ON DELETE` ที่เหมาะสมคือทักษะการออกแบบที่สำคัญ ไม่มีค่าเริ่มต้นที่ถูกต้องเสมอไปสำหรับทุกกรณี

### 196.2 ตาราง `fines`

```sql
CREATE TABLE fines (
    fine_id         SERIAL PRIMARY KEY,
    loan_id         INTEGER NOT NULL
                        REFERENCES loans (loan_id)
                        ON UPDATE CASCADE
                        ON DELETE CASCADE,
    amount          NUMERIC(8,2) NOT NULL CHECK (amount >= 0),
    reason          VARCHAR(255) NOT NULL DEFAULT 'คืนหนังสือล่าช้ากว่ากำหนด',
    is_paid         BOOLEAN NOT NULL DEFAULT FALSE,
    paid_date       DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    -- ถ้าจ่ายแล้ว ต้องมีวันที่จ่าย และถ้ายังไม่จ่าย ต้องไม่มีวันที่จ่าย
    CONSTRAINT chk_paid_date_consistency CHECK (
        (is_paid = TRUE AND paid_date IS NOT NULL)
        OR
        (is_paid = FALSE AND paid_date IS NULL)
    )
);

COMMENT ON TABLE fines IS 'ค่าปรับที่เกิดจากการคืนหนังสือล่าช้าหรือทำหนังสือสูญหาย/เสียหาย';
```

**เหตุผลของแต่ละ constraint:**

- `loan_id ... ON DELETE CASCADE` — ค่าปรับผูกอยู่กับการยืมครั้งนั้น ๆ โดยตรง ถ้าลบประวัติการยืมทิ้ง (กรณีพิเศษ เช่นแก้ไขข้อมูลผิดพลาด) ค่าปรับที่เกี่ยวข้องก็ควรถูกลบตามไปด้วย
- `amount NUMERIC(8,2) CHECK (amount >= 0)` — ค่าปรับห้ามติดลบ ใช้ `NUMERIC` สำหรับค่าเงินตามหลักการเดียวกับตาราง `books.price`
- `reason VARCHAR(255) NOT NULL DEFAULT '...'` — ใส่ค่า default เป็นเหตุผลที่พบบ่อยที่สุด (คืนช้า) เพื่อลดภาระตอน insert แต่ยังสามารถระบุเหตุผลอื่นได้ เช่น "ทำหนังสือสูญหาย"
- `CONSTRAINT chk_paid_date_consistency` — นี่คือตัวอย่าง `CHECK` ที่ซับซ้อนขึ้น โดยใช้ `AND`/`OR` ร่วมกันเพื่อบังคับว่า **สถานะการจ่ายเงินกับวันที่จ่ายต้องสอดคล้องกันเสมอ** — ถ้า `is_paid = TRUE` จะต้องมี `paid_date` เสมอ และถ้า `is_paid = FALSE` จะต้องไม่มี `paid_date` (เป็น `NULL`) การเขียน `CHECK` แบบนี้ช่วยป้องกันข้อมูลที่ขัดแย้งกันเอง (inconsistent data) เช่น แถวที่บอกว่า "จ่ายแล้ว" แต่ไม่มีวันที่จ่ายเลย

### 196.3 ตรวจสอบภาพรวมตารางทั้งหมด (ครบทั้ง 7 ตาราง)

```sql
SELECT
    table_name,
    (SELECT COUNT(*) FROM information_schema.columns c
        WHERE c.table_schema = t.table_schema AND c.table_name = t.table_name) AS column_count
FROM information_schema.tables t
WHERE table_schema = 'library'
ORDER BY table_name;
```

```
 table_name  | column_count
--------------+---------------
 authors      |             6
 book_copies  |             7
 books        |            10
 categories   |             4
 fines        |             7
 loans        |             8
 members      |             9
(7 rows)
```

ครบทั้ง 7 ตารางตามที่ออกแบบไว้ใน ER diagram ของ Step 192 แล้ว ต่อไปเราจะใส่ข้อมูลตัวอย่างให้สมจริง

---

## Step 197: เพิ่มข้อมูลตัวอย่าง (INSERT)

ในหัวข้อนี้เราจะใส่ข้อมูลตัวอย่างให้ครบทุกตาราง **ตามลำดับ FOREIGN KEY** เสมอ (ตารางฝั่ง "หนึ่ง" ก่อนตารางฝั่ง "หลาย") มิฉะนั้นจะเจอ error `violates foreign key constraint` (ทบทวนจาก Part 017) ลำดับที่ถูกต้องคือ:

`categories` → `authors` → `books` → `book_copies` & `members` → `loans` → `fines`

> เพื่อให้ตัวอย่างในบทนี้ใช้งานได้จริงกับ query ใน Step 198–199 เราจะอิง "วันนี้" ตามปฏิทินของหลักสูตรคือ **25 กันยายน 2026** ในการออกแบบวันที่ยืม-คืนบางรายการ เพื่อให้เกิดกรณี "ค้างคืนเกินกำหนด" ที่สมจริงเมื่อเทียบกับ `CURRENT_DATE`

### 197.1 ใส่ข้อมูล `categories` (8 แถว)

```sql
INSERT INTO categories (category_name, description) VALUES
    ('นิยาย',                  'วรรณกรรมประเภทเรื่องแต่ง ทั้งไทยและแปล'),
    ('วิทยาศาสตร์',            'หนังสือให้ความรู้ด้านวิทยาศาสตร์ทั่วไป'),
    ('ประวัติศาสตร์',          'หนังสือเกี่ยวกับเหตุการณ์และยุคสมัยในอดีต'),
    ('ธุรกิจและการบริหาร',      'การบริหารจัดการ การเงิน และการพัฒนาตนเองเชิงอาชีพ'),
    ('จิตวิทยา',               'ความรู้ด้านจิตวิทยาและพฤติกรรมมนุษย์'),
    ('เทคโนโลยีสารสนเทศ',      'คอมพิวเตอร์ การเขียนโปรแกรม และเทคโนโลยี'),
    ('สุขภาพและการแพทย์',       'การดูแลสุขภาพกายและใจ'),
    ('ศิลปะและการออกแบบ',      'ศิลปะ การออกแบบ และความคิดสร้างสรรค์');
```

### 197.2 ใส่ข้อมูล `authors` (12 แถว)

```sql
INSERT INTO authors (first_name, last_name, birth_year, nationality) VALUES
    ('ทวีวัฒน์',    'วงศ์ธนากร',  1975, 'ไทย'),
    ('Yuval',      'Harari',     1976, 'อิสราเอล'),
    ('เพียงดาว',   'สุขใจ',      1988, 'ไทย'),
    ('James',      'Clear',      1986, 'อเมริกัน'),
    ('อรุณี',       'ศรีสมบัติ',   1965, 'ไทย'),
    ('Malcolm',    'Gladwell',   1963, 'แคนาดา'),
    ('ปกรณ์',       'พงษ์ไพบูลย์', 1980, 'ไทย'),
    ('Cal',        'Newport',    1982, 'อเมริกัน'),
    ('สมชาย',       'ใจดี',       1955, 'ไทย'),
    ('Daniel',     'Kahneman',   1934, 'อิสราเอล'),
    ('วิภาวี',       'ชัยมงคล',    NULL, 'ไทย'),   -- ไม่ทราบปีเกิดที่แน่ชัด
    ('Robert',     'Martin',     1952, 'อเมริกัน');
```

สังเกตแถวของ **วิภาวี ชัยมงคล** ที่ใส่ `NULL` ในคอลัมน์ `birth_year` โดยตั้งใจ เพื่อจำลองสถานการณ์จริงที่บางครั้งเราไม่ทราบข้อมูลผู้แต่งครบทุกด้าน (ทบทวนหลักการ `NULL` จาก Part 015)

### 197.3 ใส่ข้อมูล `books` (15 แถว)

```sql
INSERT INTO books (isbn, title, author_id, category_id, publication_year, publisher, language, page_count, price) VALUES
    ('978-0062316097', 'Sapiens: A Brief History of Humankind',   2,  3, 2011, 'Harvill Secker',      'English', 443, 450.00),
    ('978-616-01-1234-5', 'เพลิงพ่าย',                              1,  1, 2018, 'อมรินทร์',             'Thai',    320, 295.00),
    ('978-0735211292', 'Atomic Habits',                           4,  4, 2018, 'Avery',               'English', 320, 380.00),
    ('978-616-02-5678-1', 'นิทานพันดาว',                            3,  1, 2020, 'นานมีบุ๊คส์',           'Thai',    210, 220.00),
    ('978-0316017930', 'Outliers: The Story of Success',          6,  4, 2008, 'Little, Brown',       'English', 309, 350.00),
    ('978-616-03-1122-9', 'ประวัติศาสตร์ไทยฉบับย่อ',                  5,  3, 2015, 'มติชน',                'Thai',    480, 420.00),
    ('978-1455586691', 'Deep Work',                                8,  4, 2016, 'Grand Central',       'English', 296, 340.00),
    ('978-616-04-3344-6', 'คิดแบบวิทยาศาสตร์',                      7,  2, 2019, 'ซีเอ็ด',               'Thai',    256, 260.00),
    ('978-0374533557', 'Thinking, Fast and Slow',                 10, 5, 2011, 'Farrar, Straus and Giroux', 'English', 499, 480.00),
    ('978-616-05-7788-2', 'คลีนโค้ด (Clean Code ฉบับแปลไทย)',        12, 6, 2017, 'ซีเอ็ด',               'Thai',    464, 495.00),
    ('978-616-06-9900-3', 'ห้วงเวลาแห่งรัก',                         1,  1, 2021, 'อมรินทร์',             'Thai',    288, 265.00),
    ('978-616-07-2233-4', 'สุขภาพดีเริ่มที่ใจ',                       9,  7, 2012, 'หมอชาวบ้าน',           'Thai',    180, 195.00),
    ('978-0062464316', 'Homo Deus: A Brief History of Tomorrow',  2,  3, 2016, 'Harvill Secker',      'English', 450, 460.00),
    ('978-616-08-4455-6', 'ศิลปะแห่งการออกแบบ UX',                   11, 8, 2022, 'ซีเอ็ด',               'Thai',    240, 310.00),
    ('978-616-09-6677-8', 'การบริหารเวลาอย่างมืออาชีพ',               5,  4, 2014, 'มติชน',                'Thai',    200, 210.00);
```

### 197.4 ใส่ข้อมูล `book_copies` (29 แถว)

หนังสือแต่ละเล่มมีจำนวนสำเนาไม่เท่ากัน ขึ้นอยู่กับความนิยม บางเล่มมีแค่ 1 สำเนา บางเล่มมีถึง 3 สำเนา:

```sql
INSERT INTO book_copies (book_id, copy_number, status, shelf_location, acquired_date) VALUES
    -- Sapiens (book_id 1) มี 3 สำเนา
    (1, 1, 'available', 'A1-01', '2022-01-10'),
    (1, 2, 'borrowed',  'A1-01', '2022-01-10'),
    (1, 3, 'available', 'A1-01', '2023-05-20'),
    -- เพลิงพ่าย (book_id 2) มี 2 สำเนา
    (2, 1, 'available', 'B2-05', '2019-03-15'),
    (2, 2, 'available', 'B2-05', '2019-03-15'),
    -- Atomic Habits (book_id 3) มี 3 สำเนา
    (3, 1, 'borrowed',  'C1-12', '2019-01-05'),
    (3, 2, 'borrowed',  'C1-12', '2019-01-05'),
    (3, 3, 'damaged',   'C1-12', '2019-06-18'),
    -- นิทานพันดาว (book_id 4) มี 2 สำเนา
    (4, 1, 'borrowed',  'A2-08', '2020-11-02'),
    (4, 2, 'available', 'A2-08', '2020-11-02'),
    -- Outliers (book_id 5) มี 1 สำเนา
    (5, 1, 'borrowed',  'C1-14', '2015-07-01'),
    -- ประวัติศาสตร์ไทยฉบับย่อ (book_id 6) มี 2 สำเนา
    (6, 1, 'available', 'D3-01', '2016-02-14'),
    (6, 2, 'under_repair', 'D3-01', '2016-02-14'),
    -- Deep Work (book_id 7) มี 2 สำเนา
    (7, 1, 'available', 'C1-16', '2017-04-09'),
    (7, 2, 'available', 'C1-16', '2017-04-09'),
    -- คิดแบบวิทยาศาสตร์ (book_id 8) มี 3 สำเนา
    (8, 1, 'borrowed',  'B1-03', '2020-01-20'),
    (8, 2, 'borrowed',  'B1-03', '2020-01-20'),
    (8, 3, 'available', 'B1-03', '2021-08-11'),
    -- Thinking, Fast and Slow (book_id 9) มี 1 สำเนา
    (9, 1, 'borrowed',  'C2-02', '2012-05-01'),
    -- คลีนโค้ด (book_id 10) มี 2 สำเนา
    (10, 1, 'available', 'E1-07', '2018-01-15'),
    (10, 2, 'lost',      'E1-07', '2018-01-15'),
    -- ห้วงเวลาแห่งรัก (book_id 11) มี 2 สำเนา
    (11, 1, 'borrowed',  'B2-06', '2021-09-01'),
    (11, 2, 'available', 'B2-06', '2022-02-10'),
    -- สุขภาพดีเริ่มที่ใจ (book_id 12) มี 1 สำเนา
    (12, 1, 'borrowed',  'F1-01', '2013-01-01'),
    -- Homo Deus (book_id 13) มี 2 สำเนา
    (13, 1, 'available', 'A1-02', '2017-03-01'),
    (13, 2, 'available', 'A1-02', '2017-03-01'),
    -- ศิลปะแห่งการออกแบบ UX (book_id 14) มี 1 สำเนา
    (14, 1, 'borrowed',  'G1-01', '2022-09-30'),
    -- การบริหารเวลาอย่างมืออาชีพ (book_id 15) มี 2 สำเนา
    (15, 1, 'available', 'C1-20', '2014-06-01'),
    (15, 2, 'borrowed',  'C1-20', '2014-06-01');
```

**สังเกต:** เราตั้งใจใส่สถานะ `status` ให้ครบทุกค่าที่ `CHECK` อนุญาต (`available`, `borrowed`, `lost`, `damaged`, `under_repair`) เพื่อให้มีข้อมูลไว้ฝึก query ในหัวข้อถัดไปอย่างครบถ้วน

### 197.5 ใส่ข้อมูล `members` (15 แถว)

```sql
INSERT INTO members (member_code, first_name, last_name, email, phone, membership_type, join_date, is_active) VALUES
    ('LIB-2026-0001', 'สมหญิง',    'ใจงาม',      'somying.j@example.com',    '081-111-2222', 'standard', '2023-01-15', TRUE),
    ('LIB-2026-0002', 'ธนวัฒน์',   'ศรีสุข',      'thanawat.s@example.com',   '082-222-3333', 'student',  '2024-06-01', TRUE),
    ('LIB-2026-0003', 'ปิยะดา',    'แสงทอง',     'piyada.s@example.com',     '083-333-4444', 'vip',      '2022-03-10', TRUE),
    ('LIB-2026-0004', 'กิตติพงษ์', 'รุ่งเรือง',    'kittipong.r@example.com',  '084-444-5555', 'student',  '2025-02-20', TRUE),
    ('LIB-2026-0005', 'มณีรัตน์',  'พูลสวัสดิ์',   'maneerat.p@example.com',   '085-555-6666', 'senior',   '2021-11-05', TRUE),
    ('LIB-2026-0006', 'ณัฐพล',    'เจริญสุข',    'nattapon.c@example.com',   '086-666-7777', 'standard', '2024-09-18', TRUE),
    ('LIB-2026-0007', 'วรรณา',    'ทองดี',       'wanna.t@example.com',      '087-777-8888', 'standard', '2023-07-22', FALSE),
    ('LIB-2026-0008', 'อภิสิทธิ์', 'มั่นคง',       'apisit.m@example.com',     '088-888-9999', 'vip',      '2020-01-01', TRUE),
    ('LIB-2026-0009', 'ชลธิชา',   'วงศ์สวัสดิ์',  'chonticha.w@example.com',  '089-999-0000', 'student',  '2025-08-01', TRUE),
    ('LIB-2026-0010', 'ประเสริฐ',  'ศักดิ์สิทธิ์',  'prasert.s@example.com',    '090-000-1111', 'senior',   '2019-05-14', TRUE),
    ('LIB-2026-0011', 'สุพัตรา',   'เกษมสุข',     'supattra.k@example.com',   '091-111-2222', 'standard', '2024-12-01', TRUE),
    ('LIB-2026-0012', 'ธีรภัทร',   'บุญมี',       'teerapat.b@example.com',   '092-222-3333', 'student',  '2026-01-10', TRUE),
    ('LIB-2026-0013', 'กมลวรรณ',  'อินทร์แก้ว',   'kamonwan.i@example.com',   '093-333-4444', 'standard', '2022-08-08', FALSE),
    ('LIB-2026-0014', 'นพดล',     'ทรัพย์เจริญ',  'noppadol.s@example.com',   '094-444-5555', 'vip',      '2023-03-03', TRUE),
    ('LIB-2026-0015', 'รัตนาภรณ์', 'ชูเกียรติ',    'rattanaporn.c@example.com','095-555-6666', 'senior',   '2021-06-19', TRUE);
```

สังเกตว่าสมาชิกหมายเลข 7 (วรรณา) และ 13 (กมลวรรณ) มี `is_active = FALSE` หมายถึงยกเลิกสมาชิกภาพไปแล้ว แต่ประวัติการยืมเก่ายังต้องเก็บไว้ (ตามที่ออกแบบ `ON DELETE RESTRICT` ไว้ใน Step 196)

### 197.6 ใส่ข้อมูล `loans` (20 แถว)

นี่คือตารางที่ซับซ้อนที่สุด เพราะต้องออกแบบให้มีทั้งกรณี "คืนตรงเวลา", "คืนล่าช้า", "ยังไม่ครบกำหนด", "เลยกำหนดแล้วแต่ยังไม่คืน" และ "สูญหาย" เพื่อให้ query ในหัวข้อถัดไปมีข้อมูลจริงให้ฝึกครบทุกกรณี:

```sql
INSERT INTO loans (copy_id, member_id, loan_date, due_date, return_date, status) VALUES
    ( 1,  1, '2026-08-01', '2026-08-15', '2026-08-14', 'returned'),  -- คืนตรงเวลา
    ( 4,  2, '2026-08-05', '2026-08-19', '2026-08-25', 'returned'),  -- คืนล่าช้า 6 วัน
    ( 6,  3, '2026-09-01', '2026-09-15', NULL,          'overdue'),  -- ยังไม่คืน เลยกำหนดแล้ว
    ( 9,  4, '2026-09-10', '2026-09-24', NULL,          'overdue'),  -- เลยกำหนด 1 วัน
    (11,  5, '2026-09-20', '2026-10-04', NULL,          'active'),   -- ยังไม่ครบกำหนด
    (12,  6, '2026-07-01', '2026-07-15', '2026-07-15', 'returned'),  -- คืนวันสุดท้ายพอดี
    (14,  7, '2026-06-10', '2026-06-24', '2026-07-05', 'returned'),  -- คืนล่าช้า 11 วัน
    (16,  8, '2026-09-15', '2026-09-29', NULL,          'active'),
    (19,  9, '2026-08-20', '2026-09-03', NULL,          'overdue'),
    (20, 10, '2026-05-01', '2026-05-15', '2026-05-13', 'returned'),  -- คืนก่อนกำหนด
    (22, 11, '2026-09-22', '2026-10-06', NULL,          'active'),
    (24, 12, '2026-09-05', '2026-09-19', NULL,          'overdue'),
    (25, 13, '2026-04-01', '2026-04-15', '2026-04-20', 'returned'),  -- คืนล่าช้า 5 วัน
    (27, 14, '2026-09-12', '2026-09-26', NULL,          'active'),
    (28, 15, '2026-08-15', '2026-08-29', '2026-09-02', 'returned'),  -- คืนล่าช้า 4 วัน
    ( 2,  3, '2026-09-18', '2026-10-02', NULL,          'active'),
    ( 7,  6, '2026-07-20', '2026-08-03', '2026-08-10', 'returned'),  -- คืนล่าช้า 7 วัน
    (17,  9, '2026-09-01', '2026-09-15', NULL,          'overdue'),
    (21,  1, '2026-03-01', '2026-03-15', NULL,          'lost'),     -- ทำหนังสือสูญหาย
    (29,  5, '2026-09-23', '2026-10-07', NULL,          'active');
```

ลองสังเกต `CHECK (due_date > loan_date)` และ `CHECK (return_date IS NULL OR return_date >= loan_date)` ที่เราสร้างไว้ใน Step 196 — ถ้าลองใส่แถวที่ `due_date` เท่ากับหรือน้อยกว่า `loan_date` PostgreSQL จะปฏิเสธด้วย error ทันที ลองทดสอบดูได้:

```sql
-- ทดสอบว่า constraint ทำงานจริง (คำสั่งนี้ควรจะ error)
INSERT INTO loans (copy_id, member_id, loan_date, due_date)
VALUES (1, 1, '2026-09-25', '2026-09-20');
```

```
ERROR:  new row for relation "loans" violates check constraint "chk_due_after_loan"
DETAIL:  Failing row contains (21, 1, 1, 2026-09-25, 2026-09-20, null, active, ...).
```

นี่คือตัวอย่างที่แสดงให้เห็นพลังของการออกแบบ constraint ตั้งแต่ระดับฐานข้อมูล — ไม่ว่าแอปพลิเคชันฝั่งหน้าบ้านจะมีบั๊กหรือไม่ตรวจสอบข้อมูลก่อนส่งมา ฐานข้อมูลก็ยังคงปกป้องความถูกต้องของข้อมูลได้เสมอ

### 197.7 ใส่ข้อมูล `fines` (6 แถว)

ค่าปรับจะเกิดเฉพาะกับการยืมที่คืนล่าช้า (คำนวณคร่าว ๆ ที่ 5 บาทต่อวัน) และกรณีทำหนังสือสูญหาย (คิดตามราคาหนังสือ):

```sql
INSERT INTO fines (loan_id, amount, reason, is_paid, paid_date) VALUES
    (2,  30.00, 'คืนหนังสือล่าช้ากว่ากำหนด 6 วัน',  TRUE,  '2026-08-25'),
    (7,  55.00, 'คืนหนังสือล่าช้ากว่ากำหนด 11 วัน', FALSE, NULL),
    (13, 25.00, 'คืนหนังสือล่าช้ากว่ากำหนด 5 วัน',  TRUE,  '2026-04-21'),
    (15, 20.00, 'คืนหนังสือล่าช้ากว่ากำหนด 4 วัน',  FALSE, NULL),
    (17, 35.00, 'คืนหนังสือล่าช้ากว่ากำหนด 7 วัน',  TRUE,  '2026-08-11'),
    (19, 495.00,'ทำหนังสือสูญหาย ชดใช้ราคาหนังสือเต็มจำนวน', FALSE, NULL);
```

สังเกตว่า `CONSTRAINT chk_paid_date_consistency` ที่สร้างไว้ใน Step 196.2 บังคับให้แถวที่ `is_paid = TRUE` ต้องมี `paid_date` เสมอ และแถวที่ `is_paid = FALSE` ต้องไม่มี `paid_date` — ข้อมูลด้านบนถูกออกแบบให้สอดคล้องกับกฎนี้ทุกแถว

### 197.8 ตรวจนับจำนวนแถวทั้งหมดในทุกตาราง

```sql
SELECT 'authors' AS table_name, COUNT(*) AS row_count FROM authors
UNION ALL
SELECT 'categories',   COUNT(*) FROM categories
UNION ALL
SELECT 'books',        COUNT(*) FROM books
UNION ALL
SELECT 'book_copies',  COUNT(*) FROM book_copies
UNION ALL
SELECT 'members',      COUNT(*) FROM members
UNION ALL
SELECT 'loans',        COUNT(*) FROM loans
UNION ALL
SELECT 'fines',        COUNT(*) FROM fines
ORDER BY table_name;
```

```
 table_name  | row_count
--------------+-----------
 authors      |        12
 book_copies  |        29
 books        |        15
 categories   |         8
 fines        |         6
 loans        |        20
 members      |        15
(7 rows)
```

ข้อมูลตัวอย่างพร้อมแล้ว ต่อไปเราจะเขียน query เพื่อตอบคำถามทางธุรกิจจริงของห้องสมุด

---

## Step 198: Query สำหรับ Use Case จริง (1) — หนังสือว่าง, ค้างคืน, ค่าปรับ

> **หมายเหตุสำคัญ:** ในระดับพื้นฐานเรายังไม่ได้เรียนคำสั่ง `JOIN` (จะเริ่มเรียนใน Part 021) ดังนั้น query ในหัวข้อนี้จะพยายามตอบคำถามด้วยเครื่องมือที่มีอยู่แล้ว คือ `SELECT`, `WHERE`, `ORDER BY`, `LIMIT`, subquery แบบ `IN` และการคำนวณกับชนิดข้อมูลวันที่ ซึ่งเพียงพอสำหรับตอบคำถามธุรกิจส่วนใหญ่ในบทนี้ได้ — เมื่อเรียน `JOIN` แล้วจะกลับมาเขียน query ชุดเดียวกันนี้ใหม่ให้กระชับและอ่านง่ายขึ้นมาก

### 198.1 หาหนังสือที่ว่างให้ยืม

**คำถาม:** สำเนาหนังสือเล่มไหนบ้างที่ตอนนี้ว่าง พร้อมให้สมาชิกยืมได้?

```sql
SELECT copy_id, book_id, copy_number, shelf_location, acquired_date
FROM book_copies
WHERE status = 'available'
ORDER BY book_id, copy_number;
```

```
 copy_id | book_id | copy_number | shelf_location | acquired_date
---------+---------+-------------+-----------------+----------------
       1 |       1 |           1 | A1-01           | 2022-01-10
       3 |       1 |           3 | A1-01           | 2023-05-20
       4 |       2 |           1 | B2-05           | 2019-03-15
       5 |       2 |           2 | B2-05           | 2019-03-15
      10 |       4 |           2 | A2-08           | 2020-11-02
     ...
(14 rows)
```

**คำถามต่อยอด:** หนังสือ "ชื่อเรื่องไหนบ้าง" ที่มีสำเนาว่างอย่างน้อย 1 เล่ม (ต้องเช็กจากตาราง `books` โดยอ้างอิง `book_id` ที่พบในตาราง `book_copies`) — ใช้ subquery ร่วมกับ `IN` ที่เรียนหลักการ subquery เบื้องต้นมาจาก Part 011:

```sql
SELECT book_id, isbn, title, publisher
FROM books
WHERE book_id IN (
    SELECT book_id FROM book_copies WHERE status = 'available'
)
ORDER BY title;
```

```
 book_id |      isbn       |                title                | publisher
---------+------------------+--------------------------------------+-----------
       1 | 978-0062316097   | Sapiens: A Brief History of Humankind| Harvill Secker
       4 | 978-616-02-5678-1| นิทานพันดาว                          | นานมีบุ๊คส์
      13 | 978-0062464316   | Homo Deus: A Brief History of Tomorrow| Harvill Secker
     ...
(10 rows)
```

**หนังสือที่ "ไม่มี" สำเนาว่างเลยสักเล่ม** (ทุกสำเนาถูกยืมหมด หรือเสีย/หาย) — ใช้ `NOT IN` ร่วมกับความรู้เรื่อง `NULL` จาก Part 015 (ต้องระวัง: `NOT IN` กับ subquery ที่มีค่า `NULL` ปนอยู่จะให้ผลลัพธ์ว่างเปล่าโดยไม่มี error เตือน ในกรณีนี้ปลอดภัยเพราะ `book_id` เป็น `NOT NULL` เสมอ):

```sql
SELECT book_id, title
FROM books
WHERE book_id NOT IN (
    SELECT book_id FROM book_copies WHERE status = 'available'
)
ORDER BY title;
```

```
 book_id |          title
---------+---------------------------
       3 | Atomic Habits
       5 | Outliers: The Story of Success
       9 | Thinking, Fast and Slow
      12 | สุขภาพดีเริ่มที่ใจ
      14 | ศิลปะแห่งการออกแบบ UX
(5 rows)
```

### 198.2 หาสมาชิกที่มีหนังสือเลยกำหนดคืน

**คำถาม:** การยืมรายการไหนบ้างที่ "เลยกำหนดคืนแล้ว" แต่ยังไม่มีการคืน (`return_date IS NULL`)?

```sql
SELECT loan_id, copy_id, member_id, loan_date, due_date,
       (CURRENT_DATE - due_date) AS days_overdue
FROM loans
WHERE return_date IS NULL
  AND due_date < CURRENT_DATE
ORDER BY days_overdue DESC;
```

```
 loan_id | copy_id | member_id | loan_date  |  due_date  | days_overdue
---------+---------+-----------+------------+------------+---------------
      19 |      21 |         1 | 2026-03-01 | 2026-03-15 |           194
       9 |      19 |         9 | 2026-08-20 | 2026-09-03 |            22
      18 |      17 |         9 | 2026-09-01 | 2026-09-15 |            10
      12 |      24 |        12 | 2026-09-05 | 2026-09-19 |             6
       3 |       6 |         3 | 2026-09-01 | 2026-09-15 |            10
       4 |       9 |         4 | 2026-09-10 | 2026-09-24 |             1
(6 rows)
```

> สังเกตนิพจน์ `CURRENT_DATE - due_date` — เมื่อนำค่าชนิด `DATE` สองค่ามาลบกัน PostgreSQL จะคืนค่าเป็นจำนวนวัน (integer) โดยอัตโนมัติ ซึ่งเป็นความสามารถของชนิดข้อมูล `DATE` ที่เรียนไปแล้วใน Part 007

**เจาะจงเฉพาะรายชื่อสมาชิก** ที่มีรายการค้างคืนอย่างน้อย 1 รายการ (ใช้ `IN` แบบเดียวกับหัวข้อก่อนหน้า):

```sql
SELECT member_id, member_code, first_name, last_name, email, phone
FROM members
WHERE member_id IN (
    SELECT member_id
    FROM loans
    WHERE return_date IS NULL
      AND due_date < CURRENT_DATE
)
ORDER BY last_name, first_name;
```

```
 member_id | member_code   | first_name | last_name | email                     | phone
-----------+----------------+------------+-----------+---------------------------+---------------
         1 | LIB-2026-0001  | สมหญิง     | ใจงาม     | somying.j@example.com     | 081-111-2222
         9 | LIB-2026-0009  | ชลธิชา     | วงศ์สวัสดิ์| chonticha.w@example.com   | 089-999-0000
         3 | LIB-2026-0003  | ปิยะดา     | แสงทอง    | piyada.s@example.com      | 083-333-4444
         4 | LIB-2026-0004  | กิตติพงษ์  | รุ่งเรือง  | kittipong.r@example.com   | 084-444-5555
        12 | LIB-2026-0012  | ธีรภัทร    | บุญมี     | teerapat.b@example.com    | 092-222-3333
(5 rows)
```

**เจาะจงเฉพาะการค้างคืนที่ "ร้ายแรง" (เกิน 30 วัน)** — ผสาน `WHERE` หลายเงื่อนไขด้วย `AND` ที่เรียนจาก Part 011 และเรียงจากค้างนานสุดด้วย `ORDER BY ... DESC` ผสาน `LIMIT` จาก Part 012:

```sql
SELECT loan_id, member_id, due_date, (CURRENT_DATE - due_date) AS days_overdue
FROM loans
WHERE return_date IS NULL
  AND due_date < CURRENT_DATE - INTERVAL '30 days'
ORDER BY days_overdue DESC
LIMIT 5;
```

```
 loan_id | member_id |  due_date  | days_overdue
---------+-----------+------------+---------------
      19 |         1 | 2026-03-15 |           194
(1 row)
```

### 198.3 คำนวณค่าปรับ

**คำถาม:** ถ้าคิดค่าปรับเป็นอัตรา 5 บาทต่อวันที่ล่าช้า การยืมที่ยังไม่คืนและเลยกำหนดแล้วแต่ละรายการควรมีค่าปรับ "ประมาณการ" เท่าไร?

```sql
SELECT
    loan_id,
    member_id,
    due_date,
    (CURRENT_DATE - due_date)          AS days_late,
    (CURRENT_DATE - due_date) * 5.00   AS estimated_fine_baht
FROM loans
WHERE return_date IS NULL
  AND due_date < CURRENT_DATE
ORDER BY estimated_fine_baht DESC;
```

```
 loan_id | member_id |  due_date  | days_late | estimated_fine_baht
---------+-----------+------------+-----------+----------------------
      19 |         1 | 2026-03-15 |       194 |               970.00
       9 |         9 | 2026-09-03 |        22 |               110.00
      18 |         9 | 2026-09-15 |        10 |                50.00
       3 |         3 | 2026-09-15 |        10 |                50.00
      12 |        12 | 2026-09-19 |         6 |                30.00
       4 |         4 | 2026-09-24 |         1 |                 5.00
(6 rows)
```

**ตรวจสอบค่าปรับที่บันทึกไว้จริงแล้ว (จากการคืนหนังสือในอดีต) ที่ยัง "ไม่ได้จ่าย"**:

```sql
SELECT fine_id, loan_id, amount, reason, is_paid
FROM fines
WHERE is_paid = FALSE
ORDER BY amount DESC;
```

```
 fine_id | loan_id | amount |                    reason                      | is_paid
---------+---------+--------+--------------------------------------------------+----------
       6 |      19 | 495.00 | ทำหนังสือสูญหาย ชดใช้ราคาหนังสือเต็มจำนวน        | f
       2 |       7 |  55.00 | คืนหนังสือล่าช้ากว่ากำหนด 11 วัน                  | f
       4 |      15 |  20.00 | คืนหนังสือล่าช้ากว่ากำหนด 4 วัน                   | f
(3 rows)
```

> **แอบดูล่วงหน้า (sneak peek):** ถ้าอยากรู้ "ยอดรวมค่าปรับที่ยังไม่ได้จ่ายทั้งหมด" เป็นตัวเลขเดียว จะต้องใช้ **aggregate function** อย่าง `SUM()` เช่น `SELECT SUM(amount) FROM fines WHERE is_paid = FALSE;` ซึ่งเป็นเนื้อหาที่จะได้เรียนอย่างละเอียดในระดับกลางคู่กับ `GROUP BY` — ในระดับพื้นฐานนี้เราเขียน query แบบแสดงรายแถวไปก่อน เมื่อเรียน `SUM()` แล้วจะกลับมาเขียนคำสั่งนี้ในบรรทัดเดียวได้เลย

---

## Step 199: Query สำหรับ Use Case จริง (2) — หนังสือยอดนิยม, ประวัติการยืม, สถิติ

> **หมายเหตุ:** หัวข้อนี้จำเป็นต้อง "แอบยืม" ฟังก์ชัน `COUNT()` มาใช้แบบคร่าว ๆ ล่วงหน้า เพราะรายงานเชิงสถิติ (เช่น หนังสือยอดนิยม) ไม่สามารถทำได้เลยถ้าไม่มีการนับจำนวน เราจะใช้ `COUNT()` แบบ **correlated subquery** (ไม่ใช้ `GROUP BY`) ซึ่งอ่านและเข้าใจได้ไม่ยาก ส่วนรายละเอียดเชิงลึกของ aggregate function ทั้งหมดพร้อม `GROUP BY` และ `HAVING` จะได้เรียนอย่างเป็นระบบในระดับกลาง

### 199.1 รายงานหนังสือยอดนิยม

**คำถาม:** หนังสือเล่มไหนถูกยืมบ่อยที่สุด (นับจากจำนวนครั้งที่มีการยืมสำเนาของหนังสือเล่มนั้น ไม่ว่าจะเป็นสำเนาเล่มไหนก็ตาม)?

```sql
SELECT
    b.book_id,
    b.title,
    (
        SELECT COUNT(*)
        FROM loans l
        WHERE l.copy_id IN (
            SELECT copy_id FROM book_copies bc WHERE bc.book_id = b.book_id
        )
    ) AS total_times_borrowed
FROM books b
ORDER BY total_times_borrowed DESC, b.title
LIMIT 5;
```

```
 book_id |               title                    | total_times_borrowed
---------+------------------------------------------+------------------------
       3 | Atomic Habits                            |                     2
       1 | Sapiens: A Brief History of Humankind    |                     2
      15 | การบริหารเวลาอย่างมืออาชีพ                |                     2
      10 | คลีนโค้ด (Clean Code ฉบับแปลไทย)          |                     2
       8 | คิดแบบวิทยาศาสตร์                         |                     2
(5 rows)
```

**อธิบาย query:** เราใช้ **scalar subquery ที่มีการอ้างอิงกลับไปยังตารางหลัก** (correlated subquery) — สังเกตว่า subquery ชั้นในสุดอ้างถึง `b.book_id` ซึ่งเป็นคอลัมน์จากตารางหลักภายนอก (`books b`) ทำให้ PostgreSQL ต้องรัน subquery นี้ซ้ำสำหรับหนังสือทุกแถว เพื่อนับว่าหนังสือเล่มนั้นมีประวัติการยืมกี่ครั้ง เทคนิคนี้ใช้ได้ผลดีกับข้อมูลขนาดเล็กแบบในบทเรียนนี้ แต่เมื่อข้อมูลมีขนาดใหญ่ การใช้ `JOIN` ร่วมกับ `GROUP BY` (ซึ่งจะเรียนในระดับกลาง) จะมีประสิทธิภาพดีกว่ามาก

### 199.2 ประวัติการยืมของสมาชิกแต่ละคน

**คำถาม:** สมาชิกรหัส `LIB-2026-0001` (สมหญิง ใจงาม) เคยยืมหนังสืออะไรไปบ้าง?

```sql
SELECT
    l.loan_id,
    (
        SELECT title FROM books
        WHERE book_id = (SELECT book_id FROM book_copies WHERE copy_id = l.copy_id)
    ) AS book_title,
    l.loan_date,
    l.due_date,
    l.return_date,
    l.status
FROM loans l
WHERE l.member_id = (SELECT member_id FROM members WHERE member_code = 'LIB-2026-0001')
ORDER BY l.loan_date DESC;
```

```
 loan_id |                book_title                | loan_date  |  due_date  | return_date | status
---------+--------------------------------------------+------------+------------+--------------+---------
      19 | คลีนโค้ด (Clean Code ฉบับแปลไทย)           | 2026-03-01 | 2026-03-15 | NULL         | lost
       1 | Sapiens: A Brief History of Humankind       | 2026-08-01 | 2026-08-15 | 2026-08-14   | returned
(2 rows)
```

**สังเกต:** เราใช้ subquery ซ้อนกันถึง 2 ชั้น — ชั้นแรกหา `member_id` จาก `member_code` (เพราะ member_code เป็นสิ่งที่มนุษย์จำได้ง่ายกว่า แต่ `member_id` คือ key ที่ใช้เชื่อมข้อมูลจริง) และชั้นในของแต่ละแถวหา `book_title` จาก `copy_id` ผ่าน `book_copies` ไปยัง `books` — นี่คือรูปแบบการ "เดินตามความสัมพันธ์" ด้วย subquery ซึ่งทำงานได้ถูกต้อง แต่จะสั้นและเร็วกว่ามากเมื่อเปลี่ยนไปใช้ `JOIN` ในระดับถัดไป

**นับจำนวนครั้งที่แต่ละสมาชิกเคยยืมหนังสือ (Top 5 นักอ่านตัวยง):**

```sql
SELECT
    m.member_id,
    m.first_name,
    m.last_name,
    (SELECT COUNT(*) FROM loans l WHERE l.member_id = m.member_id) AS total_loans
FROM members m
ORDER BY total_loans DESC, m.last_name
LIMIT 5;
```

```
 member_id | first_name | last_name  | total_loans
-----------+------------+------------+---------------
         1 | สมหญิง     | ใจงาม      |             2
         3 | ปิยะดา     | แสงทอง     |             2
         5 | มณีรัตน์   | พูลสวัสดิ์  |             2
         6 | ณัฐพล     | เจริญสุข    |             2
         9 | ชลธิชา     | วงศ์สวัสดิ์ |             2
(5 rows)
```

### 199.3 สถิติต่าง ๆ ของห้องสมุด

**จำนวนสมาชิกทั้งหมด แยกตามสถานะ (active/inactive):**

```sql
SELECT COUNT(*) AS active_members
FROM members
WHERE is_active = TRUE;

SELECT COUNT(*) AS inactive_members
FROM members
WHERE is_active = FALSE;
```

```
 active_members
-----------------
             13
(1 row)

 inactive_members
--------------------
              2
(1 row)
```

**จำนวนหนังสือในแต่ละหมวดหมู่ เรียงจากมากไปน้อย:**

```sql
SELECT
    c.category_name,
    (SELECT COUNT(*) FROM books b WHERE b.category_id = c.category_id) AS book_count
FROM categories c
ORDER BY book_count DESC, c.category_name;
```

```
      category_name       | book_count
----------------------------+-------------
 ธุรกิจและการบริหาร         |           4
 นิยาย                     |           3
 ประวัติศาสตร์              |           3
 จิตวิทยา                  |           1
 วิทยาศาสตร์                |           1
 สุขภาพและการแพทย์          |           1
 เทคโนโลยีสารสนเทศ          |           1
 ศิลปะและการออกแบบ          |           1
(8 rows)
```

**จำนวนสำเนาหนังสือ แยกตามสถานะ** (เนื่องจากยังไม่เรียน `GROUP BY` เราจึงต้องเขียนแยกทีละสถานะ ซึ่งเป็นสาเหตุที่ทำให้เห็นชัดว่า `GROUP BY` จะช่วยลดงานซ้ำซ้อนแบบนี้ได้มากแค่ไหนเมื่อได้เรียนในระดับกลาง):

```sql
SELECT status, COUNT(*) AS copy_count
FROM book_copies
WHERE status = 'available'
GROUP BY status  -- ตัวอย่างนี้ใช้ GROUP BY แบบง่ายที่สุดเพียงเพื่อสาธิต จะอธิบายละเอียดใน Part ถัดไป
UNION ALL
SELECT 'borrowed', COUNT(*) FROM book_copies WHERE status = 'borrowed'
UNION ALL
SELECT 'lost', COUNT(*) FROM book_copies WHERE status = 'lost'
UNION ALL
SELECT 'damaged', COUNT(*) FROM book_copies WHERE status = 'damaged'
UNION ALL
SELECT 'under_repair', COUNT(*) FROM book_copies WHERE status = 'under_repair';
```

```
    status     | copy_count
---------------+-------------
 available      |         14
 borrowed       |         12
 lost           |          1
 damaged        |          1
 under_repair   |          1
(5 rows)
```

**สมาชิกที่ยังไม่เคยยืมหนังสือเลยสักครั้ง** (ใช้ `NOT IN` ร่วมกับความเข้าใจเรื่อง `NULL` จาก Part 015 อีกครั้ง) — เป็น query ที่มีประโยชน์มาก เพราะห้องสมุดสามารถใช้ผลลัพธ์นี้ส่งอีเมลแนะนำหนังสือกระตุ้นให้สมาชิกกลุ่มนี้เริ่มใช้บริการ:

```sql
SELECT member_id, first_name, last_name, join_date
FROM members
WHERE member_id NOT IN (SELECT DISTINCT member_id FROM loans)
ORDER BY join_date;
```

```
 member_id | first_name | last_name | join_date
-----------+------------+-----------+------------
(0 rows)
```

ในชุดข้อมูลตัวอย่างของบทนี้ผลลัพธ์ว่างเปล่า (0 แถว) เพราะเราจงใจออกแบบข้อมูลให้สมาชิกทั้ง 15 คนมีประวัติการยืมอย่างน้อยคนละ 1 ครั้งอยู่แล้ว (ดู Step 197.6) — แต่ในระบบจริงที่มีสมาชิกหลักพันหรือหลักหมื่นคน คำสั่งเดียวกันนี้จะทรงพลังมาก เพราะสามารถกรองหาสมาชิก "เงียบ" (dormant member) ออกมาได้ทันทีโดยไม่ต้องไล่ดูทีละคน

---

## Step 200: ทบทวนและสรุปทุกอย่างที่เรียนมาในระดับพื้นฐาน

### 200.1 เชื่อมโยงสิ่งที่ใช้ในโปรเจกต์นี้กับแต่ละ Part

ตารางด้านล่างนี้สรุปว่าแต่ละแนวคิดที่เราใช้ตลอดโปรเจกต์ระบบห้องสมุด ถูกสอนไว้ที่ Part ไหนของหลักสูตร:

| Part | หัวข้อ | ใช้ที่ไหนในโปรเจกต์นี้ |
|---|---|---|
| 001 | Database คืออะไร, RDBMS | พื้นฐานความเข้าใจก่อนออกแบบ `library_db` |
| 002 | การติดตั้ง PostgreSQL | สภาพแวดล้อมที่ใช้รันโค้ดทั้งบท |
| 003 | เครื่องมือ (psql, pgAdmin) | ใช้รันคำสั่ง `\c`, `\d` ตรวจสอบตาราง |
| 004 | สถาปัตยกรรมของ PostgreSQL | เข้าใจว่า database, schema, table สัมพันธ์กันอย่างไร |
| 005 | การจัดการฐานข้อมูล | `CREATE DATABASE library_db`, `CREATE SCHEMA library` (Step 193) |
| 006 | ชนิดข้อมูลพื้นฐาน | เลือก `VARCHAR`, `TEXT`, `INTEGER`, `NUMERIC`, `BOOLEAN` ให้เหมาะกับแต่ละคอลัมน์ (Step 194–196) |
| 007 | ชนิดข้อมูลวันที่-เวลา | `DATE`, `TIMESTAMPTZ`, การคำนวณวันค้างคืน (Step 198) |
| 008 | การออกแบบตาราง | โครงสร้างตารางทั้ง 7 ตาราง, การแยก `books`/`book_copies` (Step 192, 194–196) |
| 009 | INSERT | ใส่ข้อมูลตัวอย่างทุกตาราง (Step 197) |
| 010 | SELECT พื้นฐาน | ทุก query ใน Step 198–199 |
| 011 | WHERE และ operator | `WHERE status = 'available'`, `AND`, `IN`, `NOT IN` (Step 198–199) |
| 012 | ORDER BY, LIMIT | จัดอันดับหนังสือยอดนิยม, Top 5 นักอ่าน (Step 199) |
| 013 | UPDATE | ใช้ปรับสถานะสำเนาหนังสือ/สถานะการยืม (ฝึกเพิ่มในแบบฝึกหัด) |
| 014 | DELETE, TRUNCATE | เข้าใจผลกระทบร่วมกับ `ON DELETE CASCADE`/`RESTRICT` (Step 196) |
| 015 | การจัดการ NULL | `return_date IS NULL`, `birth_year` ที่ไม่ทราบค่า, `CHECK (col IS NULL OR ...)` (ตลอดบท) |
| 016 | PRIMARY KEY, UNIQUE | `author_id`, `isbn`, `email`, `uq_book_copy_number` (Step 194–196) |
| 017 | FOREIGN KEY | ความสัมพันธ์ทั้ง 6 คู่ระหว่างตาราง, `ON DELETE CASCADE/RESTRICT/SET NULL` (Step 194–196) |
| 018 | CHECK, DEFAULT | `chk_due_after_loan`, `chk_paid_date_consistency`, `DEFAULT CURRENT_DATE` ฯลฯ (Step 194–196) |
| 019 | ALTER TABLE | พร้อมใช้แก้ไขโครงสร้างในแบบฝึกหัดท้ายบท |

จะเห็นได้ว่าโปรเจกต์นี้ไม่ได้ใช้ความรู้ใหม่เลยแม้แต่หัวข้อเดียว แต่เป็นการ **นำทุกอย่างที่เรียนมาผสมผสานกันในสถานการณ์จริงหนึ่งเดียว** ซึ่งคือเป้าหมายที่แท้จริงของโปรเจกต์ capstone

### 200.2 บทเรียนเชิงออกแบบที่สำคัญจากโปรเจกต์นี้

1. **แยก entity ให้ถูกระดับความละเอียด** — บทเรียนสำคัญที่สุดคือการแยก `books` (ชื่อเรื่อง) ออกจาก `book_copies` (เล่มจริง) การไม่แยกสองสิ่งนี้เป็นความผิดพลาดในการออกแบบที่พบบ่อยมากในมือใหม่
2. **เลือก `ON DELETE` behavior ตามความหมายทางธุรกิจ** — ไม่มีค่าเริ่มต้นที่ถูกต้องเสมอไป ต้องถามว่า "ถ้าแถวต้นทางถูกลบ ควรเกิดอะไรขึ้นกับแถวปลายทาง" ทุกครั้ง
3. **ใช้ `CHECK` บังคับกฎธุรกิจตั้งแต่ชั้นฐานข้อมูล** — ไม่ควรฝากความหวังไว้กับโค้ดฝั่งแอปพลิเคชันอย่างเดียว
4. **`NULL` มีความหมาย ไม่ใช่แค่ "ค่าว่าง"** — `return_date IS NULL` แปลว่า "ยังไม่คืน" ซึ่งเป็นข้อมูลที่มีความหมายทางธุรกิจชัดเจน ไม่ใช่ข้อผิดพลาด
5. **ตั้งชื่อ constraint ให้สื่อความหมาย** — เช่น `chk_due_after_loan` อ่านแล้วเข้าใจทันทีว่าคืออะไร ช่วยเวลาต้อง debug error message ในอนาคต

### 200.3 Checklist ความพร้อมก่อนขึ้นระดับกลาง

ก่อนไปต่อยัง **ระดับกลาง (Intermediate)** ซึ่งเริ่มด้วยเรื่อง `JOIN` ให้ลองตรวจสอบตัวเองด้วยรายการต่อไปนี้ ถ้าตอบ "ได้" ครบทุกข้อ แปลว่าพร้อมแล้ว:

- [ ] อธิบายความแตกต่างระหว่าง `CHAR`, `VARCHAR`, `TEXT` และบอกได้ว่าควรใช้แบบไหนเมื่อไร
- [ ] อธิบายความแตกต่างระหว่าง `DATE`, `TIMESTAMP`, `TIMESTAMPTZ` ได้
- [ ] เขียน `CREATE TABLE` พร้อม `PRIMARY KEY`, `FOREIGN KEY`, `UNIQUE`, `CHECK`, `DEFAULT`, `NOT NULL` ได้เองโดยไม่ต้องเปิดตัวอย่าง
- [ ] อธิบายได้ว่า `ON DELETE CASCADE`, `RESTRICT`, `SET NULL` ต่างกันอย่างไร และเลือกใช้ให้เหมาะกับสถานการณ์ได้
- [ ] เขียน `INSERT` ใส่ข้อมูลหลายแถวพร้อมกัน (multi-row `VALUES`) ได้
- [ ] เขียน `SELECT` พร้อม `WHERE` ที่มีเงื่อนไขซับซ้อน (`AND`, `OR`, `IN`, `BETWEEN`, `LIKE`) ได้
- [ ] เขียน `ORDER BY` หลายคอลัมน์ พร้อม `ASC`/`DESC` และ `LIMIT`/`OFFSET` ได้
- [ ] อธิบายว่าทำไม `NULL` ถึงต้องใช้ `IS NULL`/`IS NOT NULL` แทน `= NULL`
- [ ] เขียน `UPDATE` และ `DELETE` พร้อม `WHERE` ที่แม่นยำ (ไม่ลบ/แก้ทั้งตารางโดยไม่ตั้งใจ) ได้
- [ ] เขียน `ALTER TABLE` เพื่อเพิ่ม/ลบ/แก้ไขคอลัมน์และ constraint บนตารางที่มีข้อมูลอยู่แล้วได้
- [ ] อ่าน ER diagram และแปลงเป็นตารางจริงได้ด้วยตัวเอง
- [ ] เขียน subquery แบบง่าย (`IN`, `NOT IN` กับ subquery) เพื่อเชื่อมข้อมูลระหว่างตาราง 2 ตารางได้ (แม้จะยังไม่ใช้ `JOIN`)

ถ้ายังตอบ "ไม่ได้" ในข้อใดข้อหนึ่ง แนะนำให้กลับไปทบทวน Part ที่เกี่ยวข้องในตารางที่ 200.1 ก่อน แล้วลองทำแบบฝึกหัดท้ายบทนี้ให้ครบทั้ง 10 ข้อ เพื่อเสริมความมั่นใจก่อนไปต่อ

---

## สรุปหลักสูตรระดับพื้นฐาน (Part 001–020)

เมื่อเดินทางมาถึงจุดนี้ ผู้เรียนได้ผ่านการเรียนรู้มาแล้ว 20 Part เต็ม ครอบคลุม 200 Steps ซึ่งวางรากฐานความเข้าใจ PostgreSQL ตั้งแต่ระดับแนวคิดไปจนถึงการลงมือปฏิบัติจริง ขอสรุปเส้นทางทั้งหมดเป็น 4 กลุ่มใหญ่ดังนี้:

### กลุ่มที่ 1: ทำความรู้จักและติดตั้ง (Part 001–005)

เริ่มต้นจากคำถามพื้นฐานที่สุด "ฐานข้อมูลคืออะไร" และ "ทำไมต้องใช้ RDBMS" (Part 001) ไปจนถึงการลงมือติดตั้ง PostgreSQL จริงบนเครื่อง (Part 002) ทำความรู้จักเครื่องมือที่ใช้ทำงานอย่าง `psql` และ pgAdmin (Part 003) เข้าใจสถาปัตยกรรมเบื้องหลังว่า PostgreSQL ทำงานอย่างไร มี process อะไรบ้าง (Part 004) และปิดท้ายด้วยการจัดการฐานข้อมูลระดับสูงสุดอย่าง `CREATE DATABASE`, `DROP DATABASE` (Part 005) — กลุ่มนี้คือ "การปูพื้น" ก่อนที่จะเริ่มเขียนโค้ดจริง

### กลุ่มที่ 2: ชนิดข้อมูลและการออกแบบตาราง (Part 006–008)

เจาะลึกชนิดข้อมูลพื้นฐานอย่าง `INTEGER`, `VARCHAR`, `NUMERIC`, `BOOLEAN` (Part 006) และชนิดข้อมูลวันที่-เวลาที่มีความซับซ้อนเรื่อง time zone (Part 007) ก่อนจะนำความรู้เรื่องชนิดข้อมูลไปใช้ออกแบบตารางอย่างเป็นระบบด้วย `CREATE TABLE` (Part 008) — กลุ่มนี้คือทักษะ "เลือกเครื่องมือให้ถูกกับงาน" ซึ่งเราได้ใช้ทุกคอลัมน์ในโปรเจกต์ห้องสมุดจริง ๆ

### กลุ่มที่ 3: การจัดการข้อมูล — CRUD (Part 009–015)

หัวใจของการใช้งานฐานข้อมูลในชีวิตประจำวัน เริ่มจาก `INSERT` เพิ่มข้อมูล (Part 009), `SELECT` ดึงข้อมูล (Part 010), กรองข้อมูลด้วย `WHERE` และ operator ต่าง ๆ (Part 011), จัดเรียงและจำกัดผลลัพธ์ด้วย `ORDER BY`/`LIMIT` (Part 012), แก้ไขข้อมูลด้วย `UPDATE` (Part 013), ลบข้อมูลด้วย `DELETE`/`TRUNCATE` (Part 014) และปิดท้ายด้วยเรื่องที่มือใหม่มักสับสนที่สุดคือการจัดการค่า `NULL` อย่างถูกต้อง (Part 015) — กลุ่มนี้คือทักษะ CRUD (Create, Read, Update, Delete) ที่ใช้งานบ่อยที่สุดในชีวิตการทำงานจริง

### กลุ่มที่ 4: ความถูกต้องของข้อมูลและการปรับโครงสร้าง (Part 016–019)

กลุ่มสุดท้ายก่อนโปรเจกต์ปิดท้าย ว่าด้วยการบังคับกฎเกณฑ์ให้ข้อมูลถูกต้องเสมอ: `PRIMARY KEY`/`UNIQUE` ป้องกันข้อมูลซ้ำ (Part 016), `FOREIGN KEY` เชื่อมความสัมพันธ์ระหว่างตารางและป้องกันข้อมูลกำพร้า (Part 017), `CHECK`/`DEFAULT` บังคับกฎธุรกิจและลดภาระตอนใส่ข้อมูล (Part 018) และปิดท้ายด้วย `ALTER TABLE` สำหรับแก้ไขโครงสร้างตารางที่มีข้อมูลอยู่แล้วโดยไม่ทำข้อมูลเสียหาย (Part 019) — กลุ่มนี้คือสิ่งที่แยก "ฐานข้อมูลของมือสมัครเล่น" ออกจาก "ฐานข้อมูลระดับมืออาชีพ"

### Part 020: การสังเคราะห์ทุกอย่างเข้าด้วยกัน

และสุดท้ายคือ Part นี้เอง ที่นำทั้ง 4 กลุ่มข้างต้นมาผสานรวมกันในโปรเจกต์เดียว — ระบบจัดการห้องสมุดที่มีตารางจริง 7 ตาราง ข้อมูลตัวอย่างที่สมจริงกว่า 100 แถว และ query ที่ตอบคำถามทางธุรกิจได้จริง นี่คือหลักฐานว่าผู้เรียนพร้อมแล้วที่จะก้าวไปสู่ระดับกลาง ซึ่งจะเปิดโลกของการเชื่อมข้อมูลหลายตารางเข้าด้วยกันอย่างมีประสิทธิภาพด้วย `JOIN`

---

## สรุปท้ายบท

บทนี้พาเราเดินทางผ่านกระบวนการสร้างระบบฐานข้อมูลแบบครบวงจรตั้งแต่ต้นจนจบ:

- เริ่มจากการ**เก็บ requirement** และแปลงเป็นรายการ entity (Step 191)
- **ออกแบบ ER diagram** เพื่อวางแผนความสัมพันธ์ก่อนลงมือเขียนโค้ด (Step 192)
- **สร้างฐานข้อมูลและ schema** อย่างเป็นระบบ (Step 193)
- **สร้างตารางทั้ง 7 ตาราง** พร้อม constraint ที่ครบถ้วนและมีเหตุผลรองรับทุกจุด (Step 194–196)
- **ใส่ข้อมูลตัวอย่าง** ที่สมจริงและสอดคล้องกันในทุกตาราง (Step 197)
- **เขียน query** เพื่อตอบคำถามทางธุรกิจจริง ทั้งการหาหนังสือว่าง การติดตามหนี้ค้างคืน การคำนวณค่าปรับ รายงานความนิยม และสถิติต่าง ๆ (Step 198–199)
- **ทบทวนภาพรวม** ทั้งหมดของระดับพื้นฐาน พร้อม checklist ความพร้อม (Step 200)

ทักษะที่สำคัญที่สุดที่ได้จากบทนี้ไม่ใช่แค่ "รู้คำสั่ง SQL" แต่คือ **ความสามารถในการคิดแบบนักออกแบบฐานข้อมูล** — รู้จักถามคำถามที่ถูกต้องก่อนเขียนโค้ด รู้จักแยก entity ให้ถูกระดับ รู้จักเลือก constraint ที่เหมาะสมกับกฎธุรกิจ และรู้จักเขียน query ที่ตอบคำถามได้ตรงจุด ทักษะเหล่านี้จะติดตัวผู้เรียนไปตลอด ไม่ว่าจะไปสร้างระบบแบบไหนในอนาคต

---

## แบบฝึกหัดขยายโปรเจกต์ (10 ข้อ)

แบบฝึกหัดชุดนี้ให้ผู้เรียนขยายระบบห้องสมุดที่สร้างไว้ในบทนี้ต่อไปอีกขั้น โดยใช้เฉพาะความรู้ที่เรียนมาแล้วในระดับพื้นฐานเท่านั้น (ยังไม่ใช้ `JOIN`) ลองทำเองก่อนเปิดดูเฉลยใน `<details>`

### ข้อที่ 1: เพิ่มคอลัมน์จำนวนวันยืมสูงสุดตามประเภทสมาชิก

ห้องสมุดต้องการกำหนดว่าสมาชิกแต่ละประเภทยืมหนังสือได้นานสูงสุดกี่วัน (standard = 14 วัน, student = 21 วัน, senior = 14 วัน, vip = 30 วัน) จงเพิ่มคอลัมน์ `max_loan_days` ในตาราง `members` แล้วอัปเดตค่าตามประเภทสมาชิกของแต่ละคน

<details>
<summary>เฉลยข้อที่ 1</summary>

```sql
ALTER TABLE members
    ADD COLUMN max_loan_days SMALLINT NOT NULL DEFAULT 14
        CHECK (max_loan_days > 0);

UPDATE members SET max_loan_days = 14 WHERE membership_type = 'standard';
UPDATE members SET max_loan_days = 21 WHERE membership_type = 'student';
UPDATE members SET max_loan_days = 14 WHERE membership_type = 'senior';
UPDATE members SET max_loan_days = 30 WHERE membership_type = 'vip';

-- ตรวจสอบผลลัพธ์
SELECT member_code, membership_type, max_loan_days
FROM members
ORDER BY membership_type;
```

เราใส่ `DEFAULT 14` ไว้ก่อนตอน `ALTER TABLE ADD COLUMN` (ทบทวนจาก Part 019) เพื่อให้แถวเดิมทั้งหมดมีค่าเริ่มต้นทันทีโดยไม่ต้อง `UPDATE` ทีละคอลัมน์ที่เป็น `NULL` จากนั้นค่อยใช้ `UPDATE ... WHERE` ปรับค่าตามประเภทสมาชิกจริงในภายหลัง
</details>

### ข้อที่ 2: สร้างตารางจองหนังสือ (Reservations)

บางครั้งสำเนาหนังสือทุกเล่มถูกยืมหมด สมาชิกจึงอยากจอง (reserve) ไว้ล่วงหน้า เพื่อให้ห้องสมุดแจ้งเตือนเมื่อมีเล่มว่าง จงออกแบบตาราง `reservations` ที่บันทึกว่า สมาชิกคนไหนจองหนังสือเล่มไหน (อ้างอิงระดับ `books` ไม่ใช่ `book_copies` เพราะสมาชิกไม่สนใจว่าจะได้สำเนาเล่มไหน ขอแค่ชื่อเรื่องเดียวกัน) วันที่จอง และสถานะการจอง

<details>
<summary>เฉลยข้อที่ 2</summary>

```sql
CREATE TABLE reservations (
    reservation_id  SERIAL PRIMARY KEY,
    book_id         INTEGER NOT NULL
                        REFERENCES books (book_id)
                        ON UPDATE CASCADE
                        ON DELETE CASCADE,
    member_id       INTEGER NOT NULL
                        REFERENCES members (member_id)
                        ON UPDATE CASCADE
                        ON DELETE CASCADE,
    reservation_date DATE NOT NULL DEFAULT CURRENT_DATE,
    status          VARCHAR(20) NOT NULL DEFAULT 'waiting'
                        CHECK (status IN ('waiting', 'ready', 'fulfilled', 'cancelled')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    -- สมาชิกคนเดียวกันจองหนังสือเล่มเดียวกันซ้ำในสถานะ 'waiting' ไม่ได้
    CONSTRAINT uq_active_reservation UNIQUE (book_id, member_id, status)
);

COMMENT ON TABLE reservations IS 'การจองหนังสือล่วงหน้าเมื่อทุกสำเนาถูกยืมหมด';

-- ตัวอย่างการใส่ข้อมูล
INSERT INTO reservations (book_id, member_id, status) VALUES
    (5, 2, 'waiting'),   -- สมาชิก 2 จองหนังสือ Outliers ที่ตอนนี้ถูกยืมหมด
    (9, 4, 'waiting');   -- สมาชิก 4 จองหนังสือ Thinking, Fast and Slow
```

สังเกตว่าเราอ้างอิงไปที่ `books.book_id` ไม่ใช่ `book_copies.copy_id` เพราะการจองคือการจอง "ชื่อเรื่อง" ไม่ใช่จองเล่มใดเล่มหนึ่งโดยเฉพาะ — นี่คือการฝึกแยกระดับความละเอียดของ entity เช่นเดียวกับที่เรียนใน Step 192.4
</details>

### ข้อที่ 3: จำกัดจำนวนหน้าหนังสือไม่ให้เกินความสมเหตุสมผล

ทีมงานพบว่ามีการกรอกข้อมูล `page_count` ผิดพลาดบางครั้ง (เช่น กรอก 50000 หน้าโดยไม่ตั้งใจ) จงเพิ่ม `CHECK` constraint ให้ตาราง `books` เพื่อจำกัดว่า `page_count` ต้องไม่เกิน 5000 หน้า โดยไม่ทำให้ข้อมูลเดิมที่มีอยู่แล้วเสียหาย

<details>
<summary>เฉลยข้อที่ 3</summary>

```sql
-- ตรวจสอบก่อนว่าข้อมูลปัจจุบันผ่านเงื่อนไขนี้หรือไม่ (ควรได้ 0 แถว)
SELECT book_id, title, page_count
FROM books
WHERE page_count > 5000;

-- เพิ่ม constraint ใหม่ (ทบทวนจาก Part 019: ALTER TABLE ... ADD CONSTRAINT)
ALTER TABLE books
    ADD CONSTRAINT chk_page_count_reasonable CHECK (page_count IS NULL OR page_count <= 5000);
```

ขั้นตอนการตรวจสอบข้อมูลเดิมก่อนเพิ่ม constraint เป็นแนวปฏิบัติที่สำคัญมาก เพราะถ้าข้อมูลเดิมมีแถวที่ไม่ผ่านเงื่อนไข คำสั่ง `ALTER TABLE ADD CONSTRAINT` จะ error ทันทีและไม่สามารถเพิ่ม constraint ได้จนกว่าจะแก้ไขข้อมูลที่ขัดแย้งกันก่อน
</details>

### ข้อที่ 4: จำลองการคืนหนังสือ

จากข้อมูลตัวอย่างใน Step 197.6 loan_id = 5 คือการยืมของสมาชิก 5 ที่ยืมสำเนา copy_id = 11 (Outliers) และยังไม่ครบกำหนดคืน (due_date 2026-10-04) สมมติว่าวันนี้สมาชิกนำหนังสือมาคืนแล้ว (วันที่คืนจริงคือ 2026-09-28) จงเขียนคำสั่งอัปเดตให้ถูกต้องครบทั้งสองตารางที่เกี่ยวข้อง

<details>
<summary>เฉลยข้อที่ 4</summary>

```sql
-- ขั้นที่ 1: อัปเดตตาราง loans ให้บันทึกวันที่คืนจริงและเปลี่ยนสถานะ
UPDATE loans
SET return_date = '2026-09-28',
    status = 'returned'
WHERE loan_id = 5;

-- ขั้นที่ 2: อัปเดตตาราง book_copies ให้สำเนานี้กลับมาเป็น 'available'
UPDATE book_copies
SET status = 'available'
WHERE copy_id = (SELECT copy_id FROM loans WHERE loan_id = 5);

-- ตรวจสอบผลลัพธ์
SELECT loan_id, copy_id, status, return_date FROM loans WHERE loan_id = 5;
SELECT copy_id, status FROM book_copies WHERE copy_id = (SELECT copy_id FROM loans WHERE loan_id = 5);
```

ข้อนี้แสดงให้เห็นข้อจำกัดสำคัญของระดับพื้นฐาน: การคืนหนังสือ 1 ครั้งต้อง `UPDATE` ถึง 2 ตารางแยกกัน และถ้าคำสั่งใดคำสั่งหนึ่งล้มเหลวระหว่างทาง ข้อมูลจะไม่สอดคล้องกัน (inconsistent) — นี่คือปัญหาที่ **transaction** (`BEGIN` / `COMMIT` / `ROLLBACK`) จะเข้ามาแก้ไขได้อย่างสมบูรณ์แบบ ซึ่งเป็นหัวข้อที่จะได้เรียนในระดับกลาง/สูงต่อไป
</details>

### ข้อที่ 5: หมวดหมู่หนังสือยอดนิยม 3 อันดับแรก

จงเขียน query หาหมวดหมู่หนังสือ (category) 3 อันดับแรกที่มีจำนวนครั้งการยืมสะสมมากที่สุด (นับจากทุกเล่มในหมวดนั้นรวมกัน)

<details>
<summary>เฉลยข้อที่ 5</summary>

```sql
SELECT
    c.category_id,
    c.category_name,
    (
        SELECT COUNT(*)
        FROM loans l
        WHERE l.copy_id IN (
            SELECT bc.copy_id
            FROM book_copies bc
            WHERE bc.book_id IN (
                SELECT b.book_id FROM books b WHERE b.category_id = c.category_id
            )
        )
    ) AS total_loans_in_category
FROM categories c
ORDER BY total_loans_in_category DESC, c.category_name
LIMIT 3;
```

ข้อนี้ฝึกการเขียน subquery ซ้อนกันถึง 3 ชั้น (`categories` → `books` → `book_copies` → `loans`) โดยไม่ใช้ `JOIN` เลย ซึ่งแสดงให้เห็นชัดเจนว่ายิ่งความสัมพันธ์ซับซ้อนขึ้นเท่าไร โค้ดแบบ subquery ซ้อนกันก็จะยิ่งอ่านยากขึ้นเท่านั้น — นี่คือเหตุผลที่แท้จริงว่าทำไมต้องเรียน `JOIN` ในระดับถัดไป
</details>

### ข้อที่ 6: เพิ่มการยืนยันอีเมลสมาชิก

ห้องสมุดต้องการเพิ่มระบบยืนยันอีเมล จงเพิ่มคอลัมน์ `email_verified` ในตาราง `members` เป็นชนิด `BOOLEAN` ค่าเริ่มต้นคือ `FALSE` และห้ามเป็น `NULL`

<details>
<summary>เฉลยข้อที่ 6</summary>

```sql
ALTER TABLE members
    ADD COLUMN email_verified BOOLEAN NOT NULL DEFAULT FALSE;

-- ตัวอย่าง: สมมติว่าสมาชิกที่สมัครก่อนปี 2024 ได้รับการยืนยันอีเมลไปแล้วในระบบเก่า
UPDATE members
SET email_verified = TRUE
WHERE join_date < '2024-01-01';

SELECT member_code, join_date, email_verified FROM members ORDER BY join_date;
```
</details>

### ข้อที่ 7: ทดสอบพฤติกรรมของ `ON DELETE RESTRICT`

จงพยายามลบผู้แต่ง "Yuval Harari" (author_id = 2) ออกจากระบบ แล้วอธิบายว่าเกิดอะไรขึ้นและเพราะเหตุใด พร้อมบอกวิธีที่ถูกต้องหากต้องการลบผู้แต่งคนนี้จริง ๆ

<details>
<summary>เฉลยข้อที่ 7</summary>

```sql
-- คำสั่งนี้จะ ERROR เพราะ Yuval Harari มีหนังสืออยู่ในตาราง books (Sapiens, Homo Deus)
DELETE FROM authors WHERE author_id = 2;
```

```
ERROR:  update or delete on table "authors" violates foreign key constraint "books_author_id_fkey" on table "books"
DETAIL:  Key (author_id)=(2) is still referenced from table "books".
```

สาเหตุคือเราตั้งค่า `ON DELETE RESTRICT` ไว้ในตาราง `books.author_id` (Step 194.3) ซึ่งเป็นการป้องกันโดยตั้งใจ เพื่อไม่ให้ผู้แต่งที่ยังมีหนังสืออยู่ในระบบถูกลบไปโดยไม่ได้ตั้งใจ (ป้องกันข้อมูลกำพร้า) วิธีที่ถูกต้องหากต้องการลบผู้แต่งคนนี้จริง ๆ มี 2 ทางเลือก:

```sql
-- ทางเลือกที่ 1: ลบหนังสือของผู้แต่งคนนั้นออกก่อน แล้วค่อยลบผู้แต่ง
DELETE FROM books WHERE author_id = 2;
DELETE FROM authors WHERE author_id = 2;

-- ทางเลือกที่ 2: ย้ายหนังสือไปเป็นของผู้แต่งคนอื่นก่อน (ถ้าข้อมูลเดิมผิดพลาด)
UPDATE books SET author_id = 1 WHERE author_id = 2;
DELETE FROM authors WHERE author_id = 2;
```

การที่ระบบบังคับให้เราตัดสินใจอย่างชัดเจนแบบนี้ (แทนที่จะลบทิ้งแบบเงียบ ๆ) คือคุณค่าที่แท้จริงของ `FOREIGN KEY` constraint
</details>

### ข้อที่ 8: บันทึกพนักงานผู้อนุมัติยกเว้นค่าปรับ

ห้องสมุดต้องการเก็บประวัติว่าพนักงานคนไหนเป็นผู้อนุมัติยกเว้นค่าปรับให้สมาชิก จงสร้างตาราง `staff` (เก็บข้อมูลพนักงาน) แล้วเพิ่มคอลัมน์ `waived_by` ในตาราง `fines` ที่อ้างอิงไปยังตาราง `staff` (อนุญาตให้เป็น `NULL` ได้ เพราะค่าปรับส่วนใหญ่ไม่ได้ถูกยกเว้น)

<details>
<summary>เฉลยข้อที่ 8</summary>

```sql
CREATE TABLE staff (
    staff_id        SERIAL PRIMARY KEY,
    employee_code   VARCHAR(15) NOT NULL UNIQUE,
    first_name      VARCHAR(100) NOT NULL,
    last_name       VARCHAR(100) NOT NULL,
    position        VARCHAR(50) NOT NULL DEFAULT 'librarian',
    hired_date      DATE NOT NULL DEFAULT CURRENT_DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

INSERT INTO staff (employee_code, first_name, last_name, position) VALUES
    ('EMP-001', 'สุนีย์', 'มากมี', 'head_librarian'),
    ('EMP-002', 'วิชัย', 'ทองแท้', 'librarian');

ALTER TABLE fines
    ADD COLUMN waived_by INTEGER
        REFERENCES staff (staff_id)
        ON UPDATE CASCADE
        ON DELETE SET NULL;

-- ตัวอย่าง: พนักงาน EMP-001 อนุมัติยกเว้นค่าปรับ fine_id = 4 ให้กับสมาชิก
UPDATE fines
SET is_paid = TRUE,
    paid_date = CURRENT_DATE,
    waived_by = (SELECT staff_id FROM staff WHERE employee_code = 'EMP-001')
WHERE fine_id = 4;
```

สังเกตว่า `waived_by` ใช้ `ON DELETE SET NULL` เพราะถ้าพนักงานคนนั้นลาออกและถูกลบออกจากระบบในอนาคต ประวัติค่าปรับที่เคยยกเว้นไปแล้วไม่ควรถูกลบตาม แค่ไม่ทราบว่าใครเป็นคนอนุมัติก็เพียงพอ
</details>

### ข้อที่ 9: หาสมาชิกที่มีค่าปรับค้างจ่ายมากกว่า 1 ครั้ง

จงเขียน query หาสมาชิกที่มีค่าปรับสถานะ `is_paid = FALSE` มากกว่า 1 รายการ (เป็นกลุ่มที่ห้องสมุดควรติดตามทวงถามเป็นพิเศษ)

<details>
<summary>เฉลยข้อที่ 9</summary>

```sql
SELECT
    m.member_id,
    m.first_name,
    m.last_name,
    (
        SELECT COUNT(*)
        FROM fines f
        WHERE f.is_paid = FALSE
          AND f.loan_id IN (SELECT loan_id FROM loans WHERE member_id = m.member_id)
    ) AS unpaid_fine_count
FROM members m
WHERE (
    SELECT COUNT(*)
    FROM fines f
    WHERE f.is_paid = FALSE
      AND f.loan_id IN (SELECT loan_id FROM loans WHERE member_id = m.member_id)
) > 1
ORDER BY unpaid_fine_count DESC;
```

จากข้อมูลตัวอย่างในบทนี้ สมาชิกหมายเลข 9 (ชลธิชา) มีทั้งสถานะค้างคืนเกินกำหนด (loan_id 9, 18) แต่ยังไม่มีค่าปรับบันทึกไว้จริงในตาราง `fines` (เพราะยังไม่ได้คืนหนังสือ ค่าปรับจึงยังเป็นแค่ "ค่าประมาณการ" ตาม Step 198.3) ดังนั้นผลลัพธ์ของ query นี้อาจว่างเปล่าในชุดข้อมูลตัวอย่าง — ผู้เรียนสามารถลองเพิ่มข้อมูลค่าปรับซ้อนของสมาชิกคนเดียวกันเพื่อทดสอบ query นี้ให้เห็นผลลัพธ์จริงได้
</details>

### ข้อที่ 10: ขยายสถานะของสำเนาหนังสือให้รองรับ "ถูกจองไว้"

หลังจากสร้างระบบจองหนังสือในข้อที่ 2 แล้ว ห้องสมุดต้องการให้สำเนาหนังสือมีสถานะใหม่คือ `reserved` (ถูกจองไว้ รอสมาชิกมารับ) เพิ่มเข้าไปในตาราง `book_copies` จงแก้ไข `CHECK` constraint เดิมให้รองรับค่าใหม่นี้ โดยไม่ทำให้ข้อมูลเดิมเสียหาย

<details>
<summary>เฉลยข้อที่ 10</summary>

```sql
-- ขั้นที่ 1: ลบ constraint เดิมออกก่อน (ทบทวนจาก Part 019)
ALTER TABLE book_copies
    DROP CONSTRAINT book_copies_status_check;

-- ขั้นที่ 2: เพิ่ม constraint ใหม่ที่มีค่า 'reserved' เพิ่มเข้ามา
ALTER TABLE book_copies
    ADD CONSTRAINT book_copies_status_check
        CHECK (status IN ('available', 'borrowed', 'reserved', 'lost', 'damaged', 'under_repair'));

-- ทดสอบใช้งานสถานะใหม่
UPDATE book_copies SET status = 'reserved' WHERE copy_id = 4;

SELECT copy_id, status FROM book_copies WHERE copy_id = 4;
```

> **หมายเหตุ:** ชื่อ constraint ที่ PostgreSQL ตั้งให้อัตโนมัติเมื่อไม่ได้ระบุชื่อเอง (เช่น `book_copies_status_check`) จะมีรูปแบบ `<table>_<column>_check` โดยทั่วไป สามารถตรวจสอบชื่อจริงได้ด้วยคำสั่ง `\d book_copies` ก่อนลบเสมอ เพื่อความแน่ใจว่าชื่อ constraint ตรงกับที่มีอยู่จริงในระบบ — และนี่คือเหตุผลที่ตลอดบทนี้เราตั้งชื่อ constraint เองอย่างชัดเจนด้วย `CONSTRAINT ชื่อ CHECK (...)` แทนที่จะปล่อยให้ระบบตั้งชื่อให้ เพราะทำให้แก้ไขในอนาคตได้ง่ายกว่ามาก
</details>

---

## ก้าวต่อไป: ระดับกลาง (Intermediate)

ยินดีด้วย — เมื่อทำแบบฝึกหัดทั้ง 10 ข้อและผ่าน checklist ใน Step 200.3 แล้ว ผู้เรียนได้ปิดฉากระดับพื้นฐานของหลักสูตรอย่างสมบูรณ์ ระบบห้องสมุดที่สร้างขึ้นในบทนี้ยังคงเป็นฐานข้อมูลที่จะใช้ต่อยอดในระดับกลาง ซึ่งจะเริ่มต้นด้วยหัวข้อที่หลายคนรอคอย นั่นคือการเชื่อมข้อมูลจากหลายตารางเข้าด้วยกันในคำสั่งเดียวด้วย `JOIN`

พาร์ทถัดไป: **[Part 021: INNER JOIN](../02-intermediate/part-021-inner-join.md)** — จุดเริ่มต้นของระดับกลาง ที่จะพาเรากลับมาเขียน query แบบ subquery ซ้อนกันหลายชั้นที่เราฝึกไว้ในบทนี้ ให้กลายเป็น query เดียวที่กระชับ อ่านง่าย และมีประสิทธิภาพดีกว่าเดิมมาก
