# Part 071: Auditing และ Logging

**หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับมืออาชีพ | Part 071**

---

## เป้าหมายการเรียนรู้

เมื่อจบบทนี้ ผู้เรียนจะสามารถ:

- อธิบายได้ว่าทำไม "การ audit" จึงเป็นเรื่องจำเป็นสำหรับระบบฐานข้อมูลระดับองค์กร ทั้งในมุมของ compliance (PCI-DSS, GDPR, SOX, HIPAA) และในมุมของการสืบสวนเหตุการณ์ด้านความปลอดภัย (security incident investigation)
- ตั้งค่าระบบ logging พื้นฐานของ PostgreSQL ได้อย่างถูกต้อง ทั้ง `log_destination`, `logging_collector`, `log_directory`, `log_filename`
- เลือกใช้พารามิเตอร์ log ที่สำคัญ เช่น `log_min_duration_statement`, `log_statement`, `log_connections`, `log_disconnections` ให้เหมาะกับสถานการณ์จริงโดยไม่ทำให้ระบบช้าลงเกินจำเป็น
- ปรับแต่ง `log_line_prefix` เพื่อให้ log แต่ละบรรทัดมีข้อมูลครบถ้วนพอสำหรับการสืบสวนย้อนหลัง
- ติดตั้งและตั้งค่า extension **pgAudit** เพื่อทำ audit logging ระดับ object ที่ log_statement ธรรมดาทำไม่ได้
- แยกความแตกต่างระหว่าง **Session Audit Logging** กับ **Object Audit Logging** ใน pgAudit และเลือกใช้ให้เหมาะกับสถานการณ์
- อ่านและตีความ audit log entry จริง เพื่อดึงข้อมูลที่จำเป็นสำหรับรายงานการตรวจสอบ (audit report)
- วางแผนเรื่อง log rotation และการจัดการพื้นที่ดิสก์ รวมถึงแนวคิดการส่ง log ไปยังระบบ centralized logging เช่น ELK Stack หรือ Grafana Loki
- ออกแบบระบบ audit logging ที่ครบวงจรสำหรับระบบ e-commerce ที่ต้องปฏิบัติตามมาตรฐานความปลอดภัยของข้อมูลการชำระเงิน (PCI-DSS)

---

## เตรียมข้อมูล

ตลอดบทนี้เราจะใช้ schema ระบบ e-commerce ชุดเดิมที่ใช้มาตลอดทั้งหลักสูตร เพื่อสาธิตการตั้งค่า logging และ pgAudit บนตารางที่มีข้อมูลอ่อนไหว (sensitive data) เช่น อีเมลลูกค้า และราคาสินค้า

```sql
-- ตารางสินค้า
CREATE TABLE products (
    product_id   SERIAL PRIMARY KEY,
    product_name VARCHAR(150),
    unit_price   NUMERIC(10,2)
);

-- ตารางลูกค้า (มีข้อมูลส่วนบุคคล - PII)
CREATE TABLE customers (
    customer_id  SERIAL PRIMARY KEY,
    first_name   VARCHAR(60),
    email        VARCHAR(150)
);

-- ตารางคำสั่งซื้อ
CREATE TABLE orders (
    order_id     SERIAL PRIMARY KEY,
    customer_id  INTEGER REFERENCES customers(customer_id),
    status       VARCHAR(20)
);

-- ข้อมูลตัวอย่างสำหรับทดลอง
INSERT INTO products (product_name, unit_price) VALUES
    ('Wireless Mouse', 590.00),
    ('Mechanical Keyboard', 2490.00),
    ('USB-C Hub', 890.00);

INSERT INTO customers (first_name, email) VALUES
    ('Suda', 'suda@example.com'),
    ('Anan', 'anan@example.com');

INSERT INTO orders (customer_id, status) VALUES
    (1, 'PENDING'),
    (2, 'SHIPPED');
```

ตารางนี้จะถูกใช้เป็นตัวอย่างประกอบตลอดบท ทั้งเวลาสาธิต `log_statement`, การ query แบบช้า (slow query) เพื่อทดสอบ `log_min_duration_statement`, และการตั้งค่า pgAudit เพื่อดักจับการเข้าถึงตาราง `customers` ซึ่งเป็นข้อมูล PII ที่ต้องถูก audit อย่างเข้มงวด

> **หมายเหตุ**: คำสั่งในบทนี้ส่วนใหญ่รันได้จริงบน PostgreSQL local instance ของผู้เรียน แต่การแก้ไข `postgresql.conf` และการ restart/reload service ต้องมีสิทธิ์ superuser หรือสิทธิ์ระดับ OS ในการแก้ไขไฟล์ config และควรทดสอบในสภาพแวดล้อม dev/staging ก่อนนำไปใช้ใน production เสมอ

---

## Step 701: ทำไมต้อง Audit

### Audit logging คืออะไร และต่างจาก logging ทั่วไปอย่างไร

คำว่า **logging** ในความหมายทั่วไปของระบบฐานข้อมูล หมายถึงการบันทึกเหตุการณ์ต่าง ๆ ที่เกิดขึ้นในระบบ เช่น error, warning, slow query เพื่อช่วยให้ DBA (Database Administrator) ทำการ debug หรือ tuning performance ได้ง่ายขึ้น

แต่ **auditing** มีเป้าหมายที่ต่างออกไปโดยสิ้นเชิง: auditing คือการบันทึก **"ใคร ทำอะไร กับข้อมูลไหน เมื่อไหร่ และผ่านช่องทางใด"** อย่างเป็นระบบและเชื่อถือได้ (tamper-evident) เพื่อวัตถุประสงค์ 3 ประการหลัก:

1. **Compliance** — การปฏิบัติตามกฎหมายและมาตรฐานอุตสาหกรรมที่บังคับให้องค์กรต้องเก็บหลักฐานการเข้าถึงข้อมูล
2. **Security investigation** — เมื่อเกิดเหตุการณ์ผิดปกติ (เช่น ข้อมูลรั่วไหล, การเข้าถึงโดยไม่ได้รับอนุญาต) ทีม security ต้องสามารถย้อนดูได้ว่าเกิดอะไรขึ้นจริง
3. **Accountability** — สร้างความรับผิดชอบให้กับผู้ใช้งานทุกคนในระบบ (รวมถึง DBA เอง) ว่าการกระทำทุกอย่างถูกบันทึกไว้

### มาตรฐาน compliance ที่เกี่ยวข้องกับฐานข้อมูล

| มาตรฐาน | ขอบเขต | สิ่งที่เกี่ยวกับ audit logging |
|---|---|---|
| **PCI-DSS** (Payment Card Industry Data Security Standard) | ระบบที่ประมวลผล/จัดเก็บข้อมูลบัตรเครดิต | Requirement 10: ต้อง log การเข้าถึงข้อมูล cardholder ทุกครั้ง ต้องเก็บ log อย่างน้อย 1 ปี (online 3 เดือน) และต้องมี log ที่ป้องกันการแก้ไข |
| **GDPR** (General Data Protection Regulation) | ข้อมูลส่วนบุคคลของพลเมือง EU | ต้องพิสูจน์ได้ว่าใครเข้าถึงข้อมูลส่วนบุคคล (PII) เมื่อไหร่ เพื่อรองรับสิทธิ "Right to be forgotten" และการแจ้งเหตุข้อมูลรั่วไหลภายใน 72 ชั่วโมง |
| **SOX** (Sarbanes-Oxley Act) | บริษัทมหาชนในสหรัฐฯ | ต้องมี audit trail สำหรับข้อมูลทางการเงิน เพื่อป้องกันการปลอมแปลงรายงานการเงิน |
| **HIPAA** (Health Insurance Portability and Accountability Act) | ข้อมูลสุขภาพในสหรัฐฯ | ต้อง log การเข้าถึงข้อมูลผู้ป่วย (PHI - Protected Health Information) ทุกครั้ง |
| **PDPA** (Personal Data Protection Act - ไทย) | ข้อมูลส่วนบุคคลในประเทศไทย | คล้าย GDPR กำหนดให้ต้องมีมาตรการรักษาความปลอดภัยและสามารถตรวจสอบการเข้าถึงข้อมูลได้ |

สำหรับระบบ e-commerce ของเรา ตาราง `customers` ที่มีคอลัมน์ `email` ถือเป็นข้อมูล PII ที่อยู่ภายใต้ GDPR/PDPA และหากในอนาคตมีการเก็บข้อมูลบัตรเครดิต (แม้จะผ่าน payment gateway ก็ตาม) ระบบก็จะต้องปฏิบัติตาม PCI-DSS ด้วย

### สถานการณ์ที่ audit log ช่วยชีวิต DBA

ลองจินตนาการสถานการณ์ต่อไปนี้:

**สถานการณ์ที่ 1**: เช้าวันจันทร์ ทีม customer support แจ้งว่าลูกค้าหลายคนบ่นว่าคำสั่งซื้อของตนหายไป โดยไม่มีใคร "ยอมรับ" ว่าเป็นคนลบ หากไม่มี audit log จะไม่มีทางรู้เลยว่าใครรัน `DELETE FROM orders ...` เมื่อไหร่ ด้วย connection จากที่ไหน

**สถานการณ์ที่ 2**: ฝ่ายกฎหมายต้องการทราบว่ามีใครเข้าถึงอีเมลของลูกค้ารายหนึ่งในช่วง 6 เดือนที่ผ่านมาบ้าง เนื่องจากลูกค้ารายนั้นยื่นเรื่องร้องเรียนว่าถูกส่ง spam จากบุคคลที่สาม

**สถานการณ์ที่ 3**: ทีม security สงสัยว่ามี insider threat — พนักงานคนหนึ่งอาจ export ข้อมูลลูกค้าทั้งหมดออกไปก่อนลาออก การมี audit log ที่บันทึก `SELECT` จำนวนมากผิดปกติจากบัญชีเดียวจะช่วยยืนยันหรือปฏิเสธข้อสงสัยนี้ได้

ทั้งสามสถานการณ์นี้แสดงให้เห็นว่า audit logging ไม่ใช่แค่ "nice to have" แต่เป็น **security control** ที่จำเป็นสำหรับระบบที่จัดการข้อมูลสำคัญ

### หลักการสำคัญ 3 ข้อของการออกแบบระบบ audit

1. **Completeness (ความครบถ้วน)** — ต้องจับเหตุการณ์ที่สำคัญทั้งหมด ไม่ตกหล่น โดยเฉพาะการเข้าถึงข้อมูลอ่อนไหว
2. **Integrity (ความสมบูรณ์)** — log ต้องไม่ถูกแก้ไขหรือลบได้โดยผู้ใช้งานทั่วไป (แม้แต่ DBA ควรมีข้อจำกัดในการแก้ไข log ย้อนหลัง)
3. **Reviewability (สามารถตรวจสอบได้)** — log ต้องอยู่ในรูปแบบที่มนุษย์หรือระบบอัตโนมัติสามารถวิเคราะห์ได้ ไม่ใช่แค่เก็บไว้เฉย ๆ โดยไม่มีใครดู

ในบทนี้เราจะเรียนรู้เครื่องมือ 2 ระดับที่ PostgreSQL มีให้:

- **PostgreSQL native logging** (Step 702-704) — ระบบ logging พื้นฐานที่มากับ core ของ PostgreSQL
- **pgAudit extension** (Step 705-708) — extension ที่ออกแบบมาเฉพาะสำหรับ audit ระดับ enterprise/compliance

---

## Step 702: PostgreSQL Logging พื้นฐาน

### ระบบ logging ของ PostgreSQL ทำงานอย่างไร

โดย default แล้ว PostgreSQL process แต่ละตัว (postmaster, backend process สำหรับแต่ละ connection, background worker) จะเขียน log ออกทาง `stderr` เมื่อเกิดเหตุการณ์ต่าง ๆ เช่น error, warning หรือ statement ที่ตรงกับเงื่อนไขที่ตั้งไว้

การตั้งค่าพื้นฐานที่ต้องเข้าใจก่อนคือพารามิเตอร์ 4 ตัวนี้ ซึ่งอยู่ในไฟล์ `postgresql.conf`:

#### 1. `log_destination`

กำหนดว่า log จะถูกส่งไปที่ไหน สามารถระบุได้หลายค่าพร้อมกัน (comma-separated):

```ini
# postgresql.conf
log_destination = 'stderr'          # ค่า default - เขียนไป stderr
# log_destination = 'csvlog'        # เขียนเป็น CSV format (ต้องเปิด logging_collector)
# log_destination = 'jsonlog'       # เขียนเป็น JSON format (PostgreSQL 15+)
# log_destination = 'syslog'        # ส่งไป syslog ของ OS
# log_destination = 'stderr,csvlog' # เขียนทั้งสองแบบพร้อมกัน
```

ค่าที่เป็นไปได้:

| ค่า | คำอธิบาย | เหมาะกับ |
|---|---|---|
| `stderr` | เขียนไปยัง standard error (default) | ระบบทั่วไป, ใช้ร่วมกับ logging_collector |
| `csvlog` | เขียนเป็นไฟล์ CSV ที่ parse ง่าย มีคอลัมน์ครบ 26 ฟิลด์ | ระบบที่ต้องการ import log เข้า analysis tool |
| `jsonlog` | เขียนเป็น JSON บรรทัดละ 1 record (PostgreSQL 15+) | ระบบที่ใช้ centralized logging สมัยใหม่ เช่น ELK, Loki |
| `syslog` | ส่งผ่าน syslog protocol ของระบบปฏิบัติการ | องค์กรที่มีระบบ syslog รวมศูนย์อยู่แล้ว |
| `eventlog` | สำหรับ Windows เท่านั้น | Windows Server |

สำหรับงาน audit ระดับ enterprise แนะนำให้ใช้ `csvlog` หรือ `jsonlog` ควบคู่กับ `stderr` เพราะ format ที่มีโครงสร้างชัดเจนจะทำให้การ parse และวิเคราะห์ log ทำได้ง่ายกว่ามาก เมื่อเทียบกับข้อความ plain text

#### 2. `logging_collector`

เป็น process พิเศษที่ทำหน้าที่รวบรวม log จาก `stderr` ของทุก backend process แล้วเขียนลงไฟล์อย่างเป็นระเบียบ **จำเป็นต้องเปิดใช้งานถ้าต้องการใช้ `csvlog` หรือ `jsonlog`**

```ini
# postgresql.conf
logging_collector = on   # ต้องเปิดเพื่อให้ระบบไฟล์ log ทำงานอย่างเป็นระบบ
```

> **ข้อควรระวัง**: การเปลี่ยนค่า `logging_collector` ต้อง **restart** PostgreSQL server (ไม่ใช่แค่ `reload`) เนื่องจากเป็นพารามิเตอร์ประเภท `postmaster context` ที่มีผลตั้งแต่ตอน process หลักเริ่มทำงาน

#### 3. `log_directory`

กำหนด directory ที่จะเก็บไฟล์ log เมื่อเปิด `logging_collector`

```ini
# postgresql.conf
log_directory = 'log'    # relative path จาก data directory (default)
# log_directory = '/var/log/postgresql'   # absolute path แนะนำสำหรับ production
```

ใน production ควรใช้ absolute path และแยก disk/partition ต่างหากจาก data directory เพื่อป้องกันไม่ให้ log โตจนพื้นที่ disk ของข้อมูลจริงเต็ม

#### 4. `log_filename`

กำหนดรูปแบบชื่อไฟล์ log โดยใช้ strftime-style escape sequences

```ini
# postgresql.conf
log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'
```

ตัวอย่างผลลัพธ์: `postgresql-2026-09-25_140000.log`

Escape sequences ที่ใช้บ่อย:

| Escape | ความหมาย |
|---|---|
| `%Y` | ปี ค.ศ. 4 หลัก |
| `%m` | เดือน 2 หลัก |
| `%d` | วันที่ 2 หลัก |
| `%H` | ชั่วโมง 24 ชม. |
| `%M` | นาที |
| `%S` | วินาที |

### การตั้งค่าตัวอย่างสำหรับระบบ e-commerce ที่ต้อง audit

```ini
# postgresql.conf - ตั้งค่าพื้นฐานสำหรับระบบที่ต้องเก็บ audit trail

log_destination = 'csvlog'
logging_collector = on
log_directory = '/var/log/postgresql'
log_filename = 'postgresql-%Y-%m-%d.log'
log_file_mode = 0600          # จำกัดสิทธิ์อ่านเฉพาะ owner (postgres user)
```

หลังแก้ไขค่าที่เป็น `postmaster context` (เช่น `logging_collector`) ต้อง restart service:

```bash
# Debian/Ubuntu
sudo systemctl restart postgresql

# ตรวจสอบว่า log ทำงานถูกต้อง
sudo tail -f /var/log/postgresql/postgresql-2026-09-25.log
```

ตรวจสอบค่าปัจจุบันของพารามิเตอร์เหล่านี้ผ่าน SQL ได้เช่นกัน:

```sql
SELECT name, setting, context
FROM pg_settings
WHERE name IN ('log_destination', 'logging_collector', 'log_directory', 'log_filename')
ORDER BY name;
```

```
       name        |        setting        |  context
--------------------+------------------------+------------
 log_destination    | csvlog                 | sighup
 log_directory      | /var/log/postgresql    | sighup
 log_filename       | postgresql-%Y-%m-%d.log| sighup
 logging_collector  | on                     | postmaster
```

สังเกตคอลัมน์ `context`:
- `sighup` = แก้ค่าใน config แล้ว `reload` ได้เลย ไม่ต้อง restart
- `postmaster` = ต้อง restart service เท่านั้นจึงจะมีผล

```sql
-- reload config โดยไม่ต้อง restart (สำหรับพารามิเตอร์ที่เป็น sighup context)
SELECT pg_reload_conf();
```

---

## Step 703: พารามิเตอร์ log ที่สำคัญ

เมื่อระบบ logging พื้นฐานพร้อมแล้ว ขั้นต่อไปคือการกำหนดว่า "อะไรบ้าง" ที่จะถูกบันทึกลง log พารามิเตอร์ 3 กลุ่มนี้คือหัวใจของการตัดสินใจนั้น

### 1. `log_min_duration_statement` — Slow Query Log

พารามิเตอร์นี้กำหนดว่า statement ใดที่ใช้เวลาทำงานนานเกินค่าที่กำหนด (หน่วยเป็นมิลลิวินาที) จะถูกบันทึกลง log พร้อมระยะเวลาที่ใช้จริง

```ini
# postgresql.conf
log_min_duration_statement = 1000   # log statement ที่ใช้เวลา >= 1000 ms (1 วินาที)
```

ค่าพิเศษ:
- `-1` = ปิดการใช้งาน (default)
- `0` = log **ทุก** statement พร้อมเวลาที่ใช้ (ใช้เฉพาะตอน debug เท่านั้น เพราะจะทำให้ log โตเร็วมากและกระทบ performance)

ทดลองจริง:

```sql
-- ตั้งค่าระดับ session เพื่อทดสอบ (ไม่กระทบ production)
SET log_min_duration_statement = 200;  -- 200 ms

-- จำลอง query ที่ช้า ด้วยการ join ซ้ำหลายรอบ (ตัวอย่างเพื่อการสาธิต)
SELECT o.order_id, c.first_name, c.email, p.product_name
FROM orders o
JOIN customers c ON c.customer_id = o.customer_id
CROSS JOIN products p
CROSS JOIN generate_series(1, 500000) AS gs;
```

ตัวอย่าง log entry ที่ได้:

```
2026-09-25 14:05:12.331 +07 [21044] LOG:  duration: 842.117 ms  statement: SELECT o.order_id, c.first_name, c.email, p.product_name
        FROM orders o
        JOIN customers c ON c.customer_id = o.customer_id
        CROSS JOIN products p
        CROSS JOIN generate_series(1, 500000) AS gs;
```

การตั้งค่านี้เป็นเครื่องมือสำคัญสำหรับ **performance tuning** (จะกล่าวถึงเพิ่มเติมในบท Performance Tuning) แต่ก็มีมิติด้าน audit เช่นกัน — บาง query ที่ทำงานช้าผิดปกติอาจเป็นสัญญาณของการโจมตีแบบ SQL injection ที่พยายาม brute-force ข้อมูล

### 2. `log_statement` — บันทึกประเภทคำสั่งที่รัน

พารามิเตอร์นี้กำหนดว่า SQL statement ประเภทไหนบ้างที่จะถูก log โดยไม่สนใจว่าจะใช้เวลานานเท่าไหร่

```ini
# postgresql.conf
log_statement = 'none'    # ไม่ log statement เลย (default)
# log_statement = 'ddl'    # log เฉพาะ DDL (CREATE, ALTER, DROP)
# log_statement = 'mod'    # log DDL + DML ที่แก้ไขข้อมูล (INSERT, UPDATE, DELETE, TRUNCATE)
# log_statement = 'all'    # log ทุก statement (มี overhead สูงมาก ใช้เฉพาะ debug)
```

ตารางเปรียบเทียบระดับการ log:

| ค่า | DDL (CREATE/ALTER/DROP) | DML ที่แก้ไข (INSERT/UPDATE/DELETE) | SELECT | Overhead |
|---|---|---|---|---|
| `none` | ไม่ | ไม่ | ไม่ | ไม่มี |
| `ddl` | ✅ | ไม่ | ไม่ | ต่ำมาก |
| `mod` | ✅ | ✅ | ไม่ | ปานกลาง |
| `all` | ✅ | ✅ | ✅ | สูงมาก |

ทดลองตั้งค่าระดับ session:

```sql
SET log_statement = 'mod';

-- statement นี้จะถูก log เพราะเป็น DML ที่แก้ไขข้อมูล
UPDATE orders SET status = 'SHIPPED' WHERE order_id = 1;

-- statement นี้จะ "ไม่" ถูก log เพราะเป็น SELECT
SELECT * FROM orders WHERE order_id = 1;
```

ตัวอย่าง log entry:

```
2026-09-25 14:12:03.552 +07 [21099] LOG:  statement: UPDATE orders SET status = 'SHIPPED' WHERE order_id = 1;
```

> **ข้อสังเกตสำคัญ**: `log_statement = 'all'` หรือ `'mod'` **ไม่เพียงพอสำหรับงาน audit ระดับ compliance** เพราะมันไม่ได้บอกว่า "ใคร" เข้าถึง "แถวข้อมูล (row)" ไหนจริง ๆ มันบันทึกแค่ตัวคำสั่ง SQL เท่านั้น ถ้าเป็น `SELECT * FROM customers` ก็จะไม่รู้เลยว่าจริง ๆ แล้วมีกี่แถวที่ถูกอ่าน และแถวไหนบ้าง — นี่คือเหตุผลที่เราต้องมี pgAudit (Step 705 เป็นต้นไป)

### 3. `log_connections` และ `log_disconnections`

สองพารามิเตอร์นี้บันทึกการเชื่อมต่อและตัดการเชื่อมต่อของทุก session ซึ่งเป็นข้อมูลพื้นฐานที่สุดของ audit trail — "ใครล็อกอินเข้ามาเมื่อไหร่ และออกไปเมื่อไหร่"

```ini
# postgresql.conf
log_connections = on
log_disconnections = on
```

```sql
-- ตรวจสอบค่าปัจจุบัน
SHOW log_connections;
SHOW log_disconnections;
```

ตัวอย่าง log เมื่อมี client เชื่อมต่อเข้ามาใหม่:

```
2026-09-25 14:20:01.001 +07 [21150] LOG:  connection received: host=10.0.1.55 port=54322
2026-09-25 14:20:01.015 +07 [21150] LOG:  connection authorized: user=app_user database=ecommerce SSL enabled (protocol=TLSv1.3, cipher=TLS_AES_256_GCM_SHA384)
```

และเมื่อตัดการเชื่อมต่อ:

```
2026-09-25 14:35:42.882 +07 [21150] LOG:  disconnection: session time: 0:15:41.867 user=app_user database=ecommerce host=10.0.1.55 port=54322
```

ข้อมูลจากบรรทัด disconnection มีประโยชน์มากในเชิง audit เพราะบอก **session time รวม** ซึ่งช่วยตรวจจับ session ที่เปิดค้างไว้นานผิดปกติ (อาจเป็นสัญญาณของ session hijacking หรือ connection ที่ไม่ได้ปิดอย่างถูกต้องจาก connection pool)

### ตารางสรุปคำแนะนำการตั้งค่าตาม environment

| พารามิเตอร์ | Development | Staging | Production (ทั่วไป) | Production (compliance-heavy) |
|---|---|---|---|---|
| `log_min_duration_statement` | `0` (log ทุก query) | `500` ms | `1000` ms | `1000` ms |
| `log_statement` | `all` | `mod` | `ddl` | `ddl` (ใช้ pgAudit สำหรับ DML แทน) |
| `log_connections` | `on` | `on` | `on` | `on` |
| `log_disconnections` | `on` | `on` | `on` | `on` |

สาเหตุที่ production ที่ต้อง compliance-heavy ไม่ควรใช้ `log_statement = 'all'` หรือ `'mod'` คือ **overhead** และ **ความไม่ยืดหยุ่นในการกรอง** — pgAudit ให้ granularity ที่ดีกว่ามาก (จะอธิบายใน Step 705-706)

---

## Step 704: log_line_prefix — การปรับแต่ง format ของแต่ละบรรทัด log

### ทำไม log_line_prefix จึงสำคัญที่สุดสำหรับงาน audit

ลองดู log entry ที่ไม่ได้ปรับแต่ง `log_line_prefix` เลย (ใช้ default ที่ค่อนข้างเรียบง่าย):

```
LOG:  duration: 842.117 ms  statement: SELECT * FROM customers;
```

จะเห็นว่า **ไม่มีข้อมูลว่าใครรัน คำสั่งนี้เมื่อไหร่ จาก database ไหน หรือจาก process ID ไหน** — ข้อมูลแบบนี้ไร้ประโยชน์อย่างสิ้นเชิงสำหรับงาน audit เพราะไม่สามารถระบุตัวตนผู้กระทำได้

`log_line_prefix` คือพารามิเตอร์ที่กำหนด "ส่วนหัว" ของทุกบรรทัด log โดยใช้ escape sequences (`%` ตามด้วยตัวอักษร) เพื่อแทรกข้อมูล context ต่าง ๆ

### Escape sequences ที่สำคัญ

| Escape | ความหมาย | ตัวอย่างผลลัพธ์ |
|---|---|---|
| `%m` | timestamp พร้อม milliseconds | `2026-09-25 14:20:01.015 +07` |
| `%t` | timestamp ไม่มี milliseconds | `2026-09-25 14:20:01 +07` |
| `%u` | username ที่เชื่อมต่อ | `app_user` |
| `%d` | ชื่อ database | `ecommerce` |
| `%r` | remote host + port | `10.0.1.55(54322)` |
| `%h` | remote host เท่านั้น | `10.0.1.55` |
| `%p` | process ID | `21150` |
| `%c` | session ID (unique ต่อ session) | `66f3a1e2.5296` |
| `%l` | หมายเลขบรรทัด log ภายใน session นี้ | `42` |
| `%a` | application_name ที่ client ส่งมา | `psql`, `pgAdmin`, ชื่อ app |
| `%v` | virtual transaction ID | `3/145` |
| `%x` | transaction ID (ถ้ามี) | `0` ถ้ายังไม่ assign |
| `%e` | SQLSTATE error code | `42601` |
| `%q` | (ไม่แสดงผลลัพธ์ใด ๆ - หยุดพิมพ์ prefix สำหรับ non-session process) | - |
| `%%` | เครื่องหมาย `%` ตัวจริง | `%` |

### รูปแบบ log_line_prefix ที่แนะนำสำหรับงาน audit

```ini
# postgresql.conf
log_line_prefix = '%m [%p] user=%u,db=%d,app=%a,client=%h '
```

ผลลัพธ์ที่ได้:

```
2026-09-25 14:20:01.015 +07 [21150] user=app_user,db=ecommerce,app=psql,client=10.0.1.55 LOG:  statement: UPDATE orders SET status = 'SHIPPED' WHERE order_id = 1;
```

เทียบกับรูปแบบที่ครบถ้วนยิ่งขึ้นสำหรับ environment ที่ต้อง compliance สูง เช่น PCI-DSS ที่ต้องระบุตัวตนผู้ใช้อย่างชัดเจนทุกครั้ง:

```ini
# postgresql.conf - รูปแบบสำหรับ compliance-heavy environment
log_line_prefix = '%m [%p:%l] user=%u,db=%d,app=%a,client=%h,vtid=%v,txid=%x '
```

ผลลัพธ์:

```
2026-09-25 14:20:01.015 +07 [21150:3] user=app_user,db=ecommerce,app=payment_service,client=10.0.1.55,vtid=3/145,txid=0 LOG:  statement: SELECT email FROM customers WHERE customer_id = 42;
```

จาก log บรรทัดเดียวนี้ เราสามารถตอบคำถามสืบสวนได้ทันทีว่า:
- **เมื่อไหร่**: 25 กันยายน 2026 เวลา 14:20:01.015 (+07 timezone)
- **ใคร**: user `app_user`
- **จากที่ไหน**: IP address `10.0.1.55`
- **ผ่าน application อะไร**: `payment_service`
- **ทำอะไร**: อ่าน email ของ customer_id = 42
- **Process ID**: 21150 (ใช้ correlate กับ log บรรทัดอื่นในธุรกรรมเดียวกัน)

### ทดลองปรับ log_line_prefix แบบ runtime

```sql
-- ดูค่าปัจจุบัน
SHOW log_line_prefix;

-- log_line_prefix เป็น postmaster-context บางส่วน แต่โดยทั่วไปแก้ผ่าน config แล้ว reload ได้
-- (เป็น context = sighup)
SELECT name, context FROM pg_settings WHERE name = 'log_line_prefix';
```

```
       name       | context
-------------------+---------
 log_line_prefix   | sighup
```

เนื่องจากเป็น `sighup` context จึงแก้ในไฟล์ `postgresql.conf` แล้วสั่ง reload ได้โดยไม่ต้อง restart:

```bash
sudo -u postgres psql -c "SELECT pg_reload_conf();"
```

### ข้อควรระวัง: log_line_prefix ไม่สามารถตั้งเป็น session-level ได้อย่างมีความหมาย

แม้ว่า `log_line_prefix` จะปรากฏใน `pg_settings` แบบที่ดูเหมือนจะ `SET` ได้ แต่ในทางปฏิบัติควรตั้งค่านี้ที่ระดับ **server (postgresql.conf)** เท่านั้น เพราะเป้าหมายของมันคือการทำให้ **ทุกบรรทัด log มีรูปแบบสม่ำเสมอ** — ถ้าแต่ละ session ตั้งค่า prefix ต่างกัน จะทำให้การ parse log ด้วยเครื่องมืออัตโนมัติ (เช่น `pgbadger`, Logstash) ทำงานผิดพลาดได้

---

## Step 705: pgAudit extension คืออะไร

### ข้อจำกัดของ log_statement ที่ pgAudit แก้ไข

จาก Step 703 เราเห็นแล้วว่า `log_statement = 'all'` หรือ `'mod'` มีข้อจำกัดสำคัญหลายประการสำหรับงาน audit ระดับ compliance:

1. **ไม่แยกประเภทคำสั่งอย่างละเอียด** — `log_statement` แยกได้แค่ 4 ระดับ (`none`, `ddl`, `mod`, `all`) แต่ไม่สามารถบอกได้ว่า "log เฉพาะ SELECT บนตาราง customers แต่ไม่ log SELECT บนตาราง products"
3. **ไม่บันทึกระดับ object** — ไม่รู้ว่า statement หนึ่งเข้าถึง **ตารางไหนบ้าง** โดยเฉพาะ query ที่ join หลายตาราง หรือใช้ view/function ซ้อนกันหลายชั้น
3. **ไม่รองรับ role-based auditing** — ไม่สามารถบอกได้ง่าย ๆ ว่า role ไหนถูกใช้เพื่อ escalate สิทธิ์ผ่าน `SET ROLE` หรือ `GRANT`/`REVOKE`
4. **prepared statement placeholders** — `log_statement` มักจะ log statement text ดิบ ซึ่งบางครั้งไม่ครบถ้วนสำหรับการวิเคราะห์เชิงลึก

**pgAudit** (PostgreSQL Audit Extension) คือ extension แบบ open-source ที่พัฒนาโดยชุมชน PostgreSQL (เดิมเริ่มจาก 2ndQuadrant ปัจจุบันดูแลโดยหลายองค์กรรวมถึง pgAudit community) ออกแบบมาเพื่อตอบโจทย์การ audit ระดับ enterprise โดยเฉพาะ โดยอิงตามมาตรฐาน audit ของฐานข้อมูลองค์กรขนาดใหญ่ เช่น Oracle's Fine Grained Auditing

### pgAudit ทำอะไรได้มากกว่า log_statement

| ความสามารถ | log_statement | pgAudit |
|---|---|---|
| แยกประเภท statement (READ/WRITE/DDL/ROLE/FUNCTION/MISC) | ❌ (แยกได้แค่ ddl/mod/all) | ✅ แยกได้ละเอียด 7 class |
| บันทึกระดับ object (ตาราง/คอลัมน์ที่ถูกเข้าถึง) | ❌ | ✅ ผ่าน `pgaudit.log_relation` |
| แยก session-level กับ object-level auditing | ❌ | ✅ (ดู Step 707) |
| ระบุ parameter values ที่ bind เข้ากับ prepared statement | บางส่วน | ✅ ผ่าน `pgaudit.log_parameter` |
| รองรับ role-based filtering (audit เฉพาะบาง role) | ❌ | ✅ ผ่าน `pgaudit.role` |
| Log format ที่เป็นมาตรฐานสำหรับ SIEM/compliance tool | ไม่เป็นมาตรฐาน | ✅ format คงที่ ใช้ CSV/JSON parse ง่าย |

### สถาปัตยกรรมของ pgAudit

pgAudit เป็น extension ที่ทำงานผ่านกลไก **hook** ของ PostgreSQL (โดยเฉพาะ `ProcessUtility_hook` และ hook อื่น ๆ ที่เกี่ยวกับ executor) เมื่อมี statement ใดที่ตรงกับเงื่อนไขที่ตั้งค่าไว้ pgAudit จะเขียน log entry เพิ่มเติมที่มี **prefix พิเศษคือ `AUDIT:`** ต่อท้าย `LOG:` ปกติของ PostgreSQL ทำให้สามารถแยก audit log ออกจาก log ทั่วไปได้ง่ายด้วยการ grep

```
2026-09-25 14:40:11.220 +07 [21200] user=app_user,db=ecommerce LOG:  AUDIT: SESSION,1,1,READ,SELECT,,,SELECT email FROM customers WHERE customer_id = 42;,<not logged>
```

ข้อความ log ของ pgAudit จะอยู่ในรูปแบบ **comma-separated fields** ที่มีโครงสร้างตายตัว (เราจะเจาะลึกการอ่าน log format นี้ใน Step 708)

### การติดตั้ง pgAudit

pgAudit เป็น extension ที่ต้องติดตั้งแยกจาก PostgreSQL core (ไม่ได้ bundle มาให้อัตโนมัติ) ขั้นตอนติดตั้งบน Linux (Debian/Ubuntu ตัวอย่าง):

```bash
# ติดตั้ง package pgaudit ให้ตรงกับ major version ของ PostgreSQL
# ตัวอย่างสำหรับ PostgreSQL 16
sudo apt-get install postgresql-16-pgaudit
```

หลังติดตั้ง package แล้ว ต้องเพิ่ม `pgaudit` เข้าไปใน `shared_preload_libraries` ซึ่งเป็นพารามิเตอร์ประเภท **postmaster context** จึงต้อง restart server:

```ini
# postgresql.conf
shared_preload_libraries = 'pgaudit'
```

```bash
sudo systemctl restart postgresql
```

จากนั้นสร้าง extension ในแต่ละ database ที่ต้องการเปิดใช้งาน audit:

```sql
CREATE EXTENSION IF NOT EXISTS pgaudit;

-- ตรวจสอบว่าติดตั้งสำเร็จ
SELECT extname, extversion FROM pg_extension WHERE extname = 'pgaudit';
```

```
 extname | extversion
---------+------------
 pgaudit | 1.7
```

ตรวจสอบว่า `shared_preload_libraries` โหลด pgAudit สำเร็จหรือไม่:

```sql
SHOW shared_preload_libraries;
```

```
 shared_preload_libraries
---------------------------
 pgaudit
```

> **ข้อควรทราบ**: หากลืมเพิ่ม `pgaudit` ใน `shared_preload_libraries` แล้วรัน `CREATE EXTENSION pgaudit;` เลย จะสร้าง extension สำเร็จ แต่ pgAudit จะ **ไม่ทำงานจริง** เพราะ hook ยังไม่ถูกลงทะเบียนตั้งแต่ process เริ่มทำงาน — เป็นข้อผิดพลาดที่พบบ่อยที่สุดในการติดตั้ง pgAudit

---

## Step 706: การตั้งค่า pgAudit

### พารามิเตอร์หลัก: pgaudit.log

พารามิเตอร์ที่สำคัญที่สุดของ pgAudit คือ `pgaudit.log` ซึ่งกำหนดว่า statement **class** ไหนบ้างที่จะถูก audit โดยแบ่งเป็น 7 class หลัก:

| Class | คำอธิบาย | ตัวอย่างคำสั่งที่ครอบคลุม |
|---|---|---|
| `READ` | คำสั่งอ่านข้อมูล | `SELECT`, `COPY` (เมื่อใช้ source เป็น table/query) |
| `WRITE` | คำสั่งแก้ไขข้อมูล | `INSERT`, `UPDATE`, `DELETE`, `TRUNCATE`, `COPY` (เมื่อ target เป็น table) |
| `FUNCTION` | การเรียกใช้ function/procedure | `EXECUTE`, `CALL`, การเรียก function ใด ๆ |
| `ROLE` | คำสั่งเกี่ยวกับสิทธิ์และผู้ใช้ | `GRANT`, `REVOKE`, `CREATE ROLE`, `ALTER ROLE`, `DROP ROLE` |
| `DDL` | คำสั่งเปลี่ยนแปลง schema | `CREATE`, `ALTER`, `DROP` (ยกเว้นที่เข้าข่าย ROLE) |
| `MISC` | คำสั่งเบ็ดเตล็ด | `SET`, `SHOW`, `DISCARD`, `LOCK`, `NOTIFY` |
| `MISC_SET` | คำสั่ง SET/SHOW config (แยกจาก MISC ใน pgAudit เวอร์ชันใหม่) | `SET`, `RESET` |

การตั้งค่า `pgaudit.log` เป็น comma-separated list ของ class ที่ต้องการ audit สามารถใช้ `-` นำหน้าเพื่อ **ยกเว้น** class นั้นได้เช่นกัน

```sql
-- ตัวอย่าง 1: audit เฉพาะการอ่านและเขียนข้อมูล (ไม่รวม DDL/ROLE/MISC)
SET pgaudit.log = 'READ, WRITE';

-- ตัวอย่าง 2: audit ทุกอย่างยกเว้น MISC (ลด noise จากคำสั่ง SET/SHOW ที่ client ส่งมาบ่อย)
SET pgaudit.log = 'ALL, -MISC';

-- ตัวอย่าง 3: การตั้งค่าที่แนะนำสำหรับระบบ e-commerce ที่ต้อง PCI-DSS compliance
SET pgaudit.log = 'READ, WRITE, DDL, ROLE';
```

ทดลองจริงและดูผลลัพธ์:

```sql
SET pgaudit.log = 'READ, WRITE';

SELECT email FROM customers WHERE customer_id = 1;
UPDATE customers SET email = 'suda.new@example.com' WHERE customer_id = 1;
```

ตัวอย่าง log entry ที่ pgAudit สร้างขึ้น (บันทึกลง log file ตามปกติของ PostgreSQL):

```
2026-09-25 14:50:03.114 +07 [21230] user=app_user,db=ecommerce LOG:  AUDIT: SESSION,1,1,READ,SELECT,,,"SELECT email FROM customers WHERE customer_id = 1;",<not logged>
2026-09-25 14:50:05.220 +07 [21230] user=app_user,db=ecommerce LOG:  AUDIT: SESSION,2,1,WRITE,UPDATE,,,"UPDATE customers SET email = 'suda.new@example.com' WHERE customer_id = 1;",<not logged>
```

### พารามิเตอร์ pgaudit.log_relation

โดย default เมื่อ statement หนึ่งเข้าถึงหลายตาราง (เช่น `JOIN`) pgAudit จะสร้าง **audit entry เดียว** สำหรับทั้ง statement แต่ถ้าต้องการให้แยก entry ตามแต่ละ relation (table/view) ที่ถูกเข้าถึง ให้เปิด `pgaudit.log_relation`

```sql
SET pgaudit.log = 'READ';
SET pgaudit.log_relation = on;

SELECT o.order_id, c.email
FROM orders o
JOIN customers c ON c.customer_id = o.customer_id
WHERE o.status = 'PENDING';
```

ผลลัพธ์: pgAudit จะสร้าง log entry **แยกต่างหาก** สำหรับแต่ละตารางที่ query นี้เข้าถึง:

```
2026-09-25 14:55:10.001 +07 [21240] user=app_user,db=ecommerce LOG:  AUDIT: SESSION,3,1,READ,SELECT,,,"SELECT o.order_id, c.email FROM orders o JOIN customers c ON c.customer_id = o.customer_id WHERE o.status = 'PENDING';",<not logged>
2026-09-25 14:55:10.001 +07 [21240] user=app_user,db=ecommerce LOG:  AUDIT: SESSION,3,2,READ,SELECT,TABLE,public.orders,"SELECT o.order_id, c.email FROM orders o JOIN customers c ON c.customer_id = o.customer_id WHERE o.status = 'PENDING';",<not logged>
2026-09-25 14:55:10.001 +07 [21240] user=app_user,db=ecommerce LOG:  AUDIT: SESSION,3,3,READ,SELECT,TABLE,public.customers,"SELECT o.order_id, c.email FROM orders o JOIN customers c ON c.customer_id = o.customer_id WHERE o.status = 'PENDING';",<not logged>
```

สังเกตว่า field ที่ 6 (relation type) และ field ที่ 7 (relation name) ถูกเติมข้อมูล `TABLE,public.orders` และ `TABLE,public.customers` ตามลำดับ — ข้อมูลนี้สำคัญมากสำหรับการตอบคำถามว่า **"ใครเคยอ่านตาราง customers บ้าง"** โดยไม่ต้อง parse SQL text เอง

### พารามิเตอร์เสริมอื่น ๆ ที่ควรรู้

```sql
-- log ค่า parameter ที่ bind เข้า prepared statement (เช่น ค่าจาก application ที่ใช้ parameterized query)
SET pgaudit.log_parameter = on;

-- ไม่ log statement text ซ้ำในทุก entry ของ relation เดียวกัน (ลดขนาด log)
SET pgaudit.log_statement_once = on;

-- log ผลลัพธ์ว่าคำสั่งสำเร็จหรือ error (PostgreSQL 15+ ของ pgAudit)
SET pgaudit.log_client = on;
```

ตัวอย่างการตั้งค่าแบบถาวรใน `postgresql.conf` สำหรับระบบ e-commerce ที่ต้อง audit ตามมาตรฐาน PCI-DSS:

```ini
# postgresql.conf - การตั้งค่า pgAudit สำหรับระบบ e-commerce (PCI-DSS compliant)
shared_preload_libraries = 'pgaudit'

pgaudit.log = 'READ, WRITE, DDL, ROLE'
pgaudit.log_relation = on
pgaudit.log_parameter = on
pgaudit.log_statement_once = off
pgaudit.log_catalog = off          # ไม่ audit query ที่เข้าถึงเฉพาะ system catalog (pg_catalog) ลด noise
```

หลังแก้ไข `postgresql.conf` (พารามิเตอร์ของ pgAudit ส่วนใหญ่เป็น context ระดับ `superuser` หรือ `user` ซึ่งสามารถ `SET` ระดับ session/database ได้ ไม่จำเป็นต้อง restart ยกเว้น `shared_preload_libraries`):

```sql
SELECT pg_reload_conf();
```

### การตั้งค่าระดับ database หรือ role โดยเฉพาะ

ในทางปฏิบัติ มักไม่ต้องการเปิด audit แบบเดียวกันทุก session — เช่น อาจต้องการ audit เฉพาะ role ที่เข้าถึงตาราง `customers` อย่างเข้มงวด แต่ไม่ต้องการ audit ทุกคำสั่งของ role ที่ใช้สำหรับ reporting ทั่วไป สามารถตั้งค่าระดับ role ได้ด้วย `ALTER ROLE ... SET`:

```sql
-- ตั้งค่า audit เข้มงวดเฉพาะสำหรับ role ที่เข้าถึงข้อมูลลูกค้า
ALTER ROLE payment_service_role SET pgaudit.log = 'READ, WRITE, DDL, ROLE';
ALTER ROLE payment_service_role SET pgaudit.log_relation = on;

-- role สำหรับ reporting ทั่วไป อาจ audit แค่ DDL/ROLE (ลด overhead)
ALTER ROLE reporting_role SET pgaudit.log = 'DDL, ROLE';
```

หรือระดับ database:

```sql
ALTER DATABASE ecommerce SET pgaudit.log = 'READ, WRITE, DDL, ROLE';
```

---

## Step 707: Session Audit Logging เทียบกับ Object Audit Logging

pgAudit มีโหมดการทำงาน 2 แบบที่มีวัตถุประสงค์ต่างกันโดยสิ้นเชิง — การเข้าใจความแตกต่างนี้เป็นกุญแจสำคัญในการออกแบบระบบ audit ที่มีประสิทธิภาพและไม่สร้าง log จำนวนมหาศาลโดยไม่จำเป็น

### Session Audit Logging

**Session Audit Logging** คือสิ่งที่เราเห็นมาตลอดใน Step 706 — การตั้งค่า `pgaudit.log` จะทำให้ pgAudit บันทึกทุก statement ที่ตรงกับ class ที่กำหนด **สำหรับทุก session** ที่มีการตั้งค่านี้ (ไม่ว่าจะตั้งผ่าน server config, database, role, หรือ session)

จุดเด่น:
- ตั้งค่าง่าย ครอบคลุมกว้าง
- เหมาะกับการ audit ภาพรวมของกิจกรรมในระบบ

จุดด้อย:
- **ไม่สามารถระบุ object เฉพาะเจาะจงได้อย่างละเอียด** แม้จะเปิด `pgaudit.log_relation` ก็ยังคง audit "ทุกตาราง" ที่ query เข้าถึง ไม่สามารถเจาะจงว่า "audit เฉพาะตาราง customers เท่านั้น แต่ไม่ต้อง audit ตาราง products"
- อาจสร้าง log จำนวนมากถ้า workload มี query สูง

### Object Audit Logging

**Object Audit Logging** ใช้กลไกที่ต่างออกไปโดยสิ้นเชิง: มันใช้ **role พิเศษ** ชื่อ `pgaudit_object` (หรือชื่ออื่นที่กำหนดผ่าน `pgaudit.role`) ที่ทำหน้าที่เป็น "ตัวแทน" สำหรับ audit เฉพาะ object ที่ role นั้นถูก `GRANT` สิทธิ์ให้ — หลักการคือ **"audit เฉพาะ object ที่ audit role มีสิทธิ์เข้าถึง"** โดยไม่สนใจว่า session ที่รัน statement จะเป็น role ไหน

### วิธีตั้งค่า Object Audit Logging

```sql
-- ขั้นที่ 1: สร้าง role พิเศษสำหรับทำหน้าที่เป็น "audit role"
-- role นี้ไม่จำเป็นต้องใช้ login จริง เป็นเพียงตัวแทนสำหรับกำหนดขอบเขต audit
CREATE ROLE pgaudit_object NOLOGIN;

-- ขั้นที่ 2: กำหนดใน postgresql.conf (หรือ SET ระดับ database) ว่าใช้ role นี้เป็นตัวกำหนดขอบเขต object audit
-- pgaudit.role = 'pgaudit_object'

-- ขั้นที่ 3: GRANT สิทธิ์เฉพาะ object ที่ต้องการ audit ให้กับ audit role นี้
-- ต้องการ audit ทุกการ SELECT/UPDATE บนตาราง customers (ตารางที่มี PII)
GRANT SELECT, UPDATE ON customers TO pgaudit_object;
```

```ini
# postgresql.conf
pgaudit.role = 'pgaudit_object'
```

```sql
SELECT pg_reload_conf();
```

จากนั้นทดสอบ:

```sql
-- session ของ app_user (role ทั่วไป ไม่ใช่ pgaudit_object)
SELECT email FROM customers WHERE customer_id = 1;   -- ตารางที่ pgaudit_object มีสิทธิ์ SELECT -> ถูก audit
SELECT unit_price FROM products WHERE product_id = 1; -- ตารางที่ pgaudit_object ไม่มีสิทธิ์ -> ไม่ถูก audit (ในโหมด object-only)
```

ผลลัพธ์ log (เมื่อปิด session audit ไว้ และเปิดเฉพาะ object audit ผ่าน `pgaudit.role`):

```
2026-09-25 15:02:44.331 +07 [21260] user=app_user,db=ecommerce LOG:  AUDIT: OBJECT,1,1,READ,SELECT,TABLE,public.customers,"SELECT email FROM customers WHERE customer_id = 1;",<not logged>
```

สังเกตว่า field แรกของ log entry เปลี่ยนจาก `SESSION` เป็น **`OBJECT`** — นี่คือวิธีแยกแยะระหว่าง session audit log กับ object audit log เมื่อ parse ข้อมูล

statement ที่สองที่ query ตาราง `products` จะ **ไม่ปรากฏ** ใน audit log เลย เพราะ `pgaudit_object` role ไม่มีสิทธิ์ SELECT บนตารางนั้น

### ตารางเปรียบเทียบ Session vs Object Audit Logging

| มิติ | Session Audit Logging | Object Audit Logging |
|---|---|---|
| ขอบเขต | ทุก session ที่ตรงเงื่อนไข class | เฉพาะ object ที่ audit role มีสิทธิ์ |
| การตั้งค่า | `pgaudit.log` (READ/WRITE/DDL/...) | `GRANT` สิทธิ์ให้ `pgaudit.role` + `pgaudit.log = 'NONE'` (ปิด session) หรือใช้ร่วมกัน |
| Granularity | ระดับ statement class | ระดับ table/column แม่นยำ |
| เหมาะกับ | Audit ภาพรวมของกิจกรรมทั้งหมด (เช่น ทุก DDL, ทุก ROLE change) | Audit เฉพาะข้อมูลอ่อนไหว (PII, ข้อมูลการเงิน) |
| Log field ระบุ | `SESSION` | `OBJECT` |
| ตัวอย่างการใช้งาน | บันทึกทุกคำสั่ง DDL/GRANT ในระบบ เพื่อ SOX compliance | บันทึกทุกการเข้าถึงตาราง customers/payment_info เพื่อ GDPR/PCI-DSS |

### กลยุทธ์ที่แนะนำ: ใช้ทั้งสองแบบร่วมกัน

ในระบบจริงระดับ enterprise มักใช้ทั้งสองโหมดพร้อมกัน โดยแบ่งความรับผิดชอบดังนี้:

```ini
# postgresql.conf - กลยุทธ์ผสมผสานสำหรับระบบ e-commerce

# Session audit: จับภาพรวมของ DDL และการเปลี่ยนแปลงสิทธิ์ทั้งระบบ
pgaudit.log = 'DDL, ROLE'

# Object audit: จับการเข้าถึงข้อมูลอ่อนไหวแบบละเอียด (READ/WRITE) เฉพาะตารางที่ระบุ
pgaudit.role = 'pgaudit_object'
```

```sql
-- audit role มีสิทธิ์เฉพาะตารางที่มีข้อมูลอ่อนไหว
GRANT SELECT, INSERT, UPDATE, DELETE ON customers TO pgaudit_object;
GRANT SELECT, INSERT, UPDATE, DELETE ON orders TO pgaudit_object;
-- ไม่ GRANT สิทธิ์บนตาราง products เพราะไม่ใช่ข้อมูลอ่อนไหว ไม่จำเป็นต้อง audit ละเอียดระดับนี้
```

ด้วยกลยุทธ์นี้ ระบบจะได้ audit log ที่ **ครบถ้วนสำหรับ compliance แต่ไม่ท่วมท้นจนวิเคราะห์ไม่ไหว** — DDL/ROLE change ทั้งระบบถูกจับหมด ขณะที่การเข้าถึงข้อมูลระดับแถว (row) จะถูกเน้นเฉพาะตารางที่มีความเสี่ยงสูง

---

## Step 708: การอ่านและวิเคราะห์ audit log

### โครงสร้างฟิลด์ของ pgAudit log entry

ทุก log entry ของ pgAudit มีโครงสร้างเป็น comma-separated fields ตามลำดับดังนี้ (ต่อจาก prefix `AUDIT:`):

```
AUDIT_TYPE, STATEMENT_ID, SUBSTATEMENT_ID, CLASS, COMMAND, OBJECT_TYPE, OBJECT_NAME, STATEMENT, PARAMETER
```

| ตำแหน่ง | ชื่อฟิลด์ | ความหมาย |
|---|---|---|
| 1 | `AUDIT_TYPE` | `SESSION` หรือ `OBJECT` |
| 2 | `STATEMENT_ID` | หมายเลขลำดับ statement ภายใน session (เพิ่มขึ้นทุกครั้งที่มี statement ใหม่) |
| 3 | `SUBSTATEMENT_ID` | หมายเลขลำดับ sub-statement (เมื่อ statement เดียวมีหลาย entry เช่นตอนเปิด `log_relation`) |
| 4 | `CLASS` | READ / WRITE / FUNCTION / ROLE / DDL / MISC |
| 5 | `COMMAND` | คำสั่งจริง เช่น SELECT, UPDATE, GRANT, CREATE TABLE |
| 6 | `OBJECT_TYPE` | ประเภท object เช่น TABLE, VIEW, FUNCTION (ว่างถ้าไม่ระบุ relation) |
| 7 | `OBJECT_NAME` | ชื่อเต็มของ object รวม schema เช่น `public.customers` |
| 8 | `STATEMENT` | ข้อความ SQL statement เต็ม |
| 9 | `PARAMETER` | ค่า parameter ที่ bind (ถ้าเปิด `pgaudit.log_parameter`) หรือ `<not logged>` |

### ตัวอย่างการวิเคราะห์ log entry จริงทีละฟิลด์

ลองดู log entry ตัวอย่างนี้ที่ถูกบันทึกเมื่อมีคนพยายามลบคำสั่งซื้อ:

```
2026-09-25 15:20:33.884 +07 [21301] user=admin_user,db=ecommerce,client=192.168.1.20 LOG:  AUDIT: SESSION,5,1,WRITE,DELETE,TABLE,public.orders,"DELETE FROM orders WHERE order_id = $1;",'99'
```

การตีความทีละส่วน:

**ส่วน log_line_prefix (นอกเหนือจาก pgAudit):**
- `2026-09-25 15:20:33.884 +07` = เวลาที่เกิดเหตุการณ์
- `[21301]` = process ID
- `user=admin_user` = ผู้ใช้ที่รันคำสั่ง — **ข้อมูลสำคัญที่สุด**
- `db=ecommerce` = database ที่ทำงานอยู่
- `client=192.168.1.20` = IP ต้นทางของ connection

**ส่วน pgAudit fields:**
- `SESSION` = เป็น session-level audit (ไม่ใช่ object-only)
- `5` = statement ลำดับที่ 5 ใน session นี้
- `1` = sub-statement ลำดับที่ 1 (ไม่มี sub-entry เพิ่มเติม)
- `WRITE` = class เป็นการเขียนข้อมูล
- `DELETE` = คำสั่งจริงคือ DELETE
- `TABLE` / `public.orders` = เข้าถึงตาราง orders ใน schema public
- `"DELETE FROM orders WHERE order_id = $1;"` = SQL text (ใช้ placeholder เพราะเป็น prepared statement)
- `'99'` = ค่าพารามิเตอร์จริงที่ bind เข้า `$1` คือ `99`

**สรุปจาก log บรรทัดนี้**: ผู้ใช้ `admin_user` จาก IP `192.168.1.20` ได้ลบคำสั่งซื้อที่มี `order_id = 99` เมื่อเวลา 15:20:33.884 น. ของวันที่ 25 กันยายน 2026 — ข้อมูลนี้เพียงพอสำหรับการตอบคำถามในสถานการณ์ที่ 1 ที่ยกตัวอย่างไว้ใน Step 701

### ตัวอย่างการ query วิเคราะห์ audit log ด้วย SQL (ผ่าน foreign data wrapper หรือ external table)

ในทางปฏิบัติ DBA มักจะโหลด CSV/JSON log เข้าไปเป็นตารางชั่วคราวเพื่อ query วิเคราะห์ได้สะดวก ตัวอย่างการสร้างตารางรับ csvlog:

```sql
-- โครงสร้างตารางที่ตรงกับ csvlog format ของ PostgreSQL (26 คอลัมน์มาตรฐาน)
CREATE TABLE postgres_log (
    log_time             TIMESTAMP(3) WITH TIME ZONE,
    user_name            TEXT,
    database_name        TEXT,
    process_id           INTEGER,
    connection_from      TEXT,
    session_id           TEXT,
    session_line_num     BIGINT,
    command_tag          TEXT,
    session_start_time   TIMESTAMP WITH TIME ZONE,
    virtual_transaction_id TEXT,
    transaction_id        BIGINT,
    error_severity        TEXT,
    sql_state_code         TEXT,
    message                TEXT,
    detail                 TEXT,
    hint                   TEXT,
    internal_query          TEXT,
    internal_query_pos      INTEGER,
    context                 TEXT,
    query                   TEXT,
    query_pos                INTEGER,
    location                  TEXT,
    application_name          TEXT,
    backend_type               TEXT,
    leader_pid                  INTEGER,
    query_id                     BIGINT
);

-- โหลดไฟล์ csvlog เข้าตารางด้วย COPY
COPY postgres_log FROM '/var/log/postgresql/postgresql-2026-09-25.csv' WITH (FORMAT csv);

-- Query หาทุกครั้งที่มีคนเข้าถึงตาราง customers ผ่าน pgAudit ในวันนี้
SELECT log_time, user_name, connection_from, message
FROM postgres_log
WHERE message LIKE 'AUDIT:%public.customers%'
ORDER BY log_time DESC;

-- Query สรุปจำนวนครั้งที่แต่ละ user เข้าถึงข้อมูลอ่อนไหว แยกตามวัน
SELECT
    date_trunc('day', log_time) AS audit_day,
    user_name,
    count(*) AS access_count
FROM postgres_log
WHERE message LIKE 'AUDIT:%READ%public.customers%'
GROUP BY 1, 2
ORDER BY 1 DESC, 3 DESC;
```

ผลลัพธ์ตัวอย่าง:

```
 audit_day  | user_name        | access_count
------------+-------------------+--------------
 2026-09-25 | payment_service   |          142
 2026-09-25 | admin_user        |            3
 2026-09-25 | reporting_role    |           27
```

รายงานแบบนี้เป็นพื้นฐานของ **audit report** ที่ต้องส่งให้ auditor ภายนอกตรวจสอบตามมาตรฐาน PCI-DSS หรือ SOC 2

### เครื่องมือสำเร็จรูปสำหรับวิเคราะห์ log

นอกจากการ query เองด้วย SQL แล้ว ยังมีเครื่องมือ open-source ที่ช่วยวิเคราะห์ PostgreSQL log ได้สะดวกกว่า เช่น:

- **pgBadger** — สร้างรายงาน HTML สรุปสถิติ query, slow query, error จาก log file โดยอัตโนมัติ (รองรับทั้ง `log_min_duration_statement` และบางส่วนของ pgAudit)
- **pgaudit_analyze** (community script) — สคริปต์ parse audit log format ของ pgAudit โดยเฉพาะ

```bash
# ตัวอย่างการใช้ pgBadger เพื่อสร้างรายงานจาก log file
pgbadger /var/log/postgresql/postgresql-2026-09-25.log -o report.html
```

---

## Step 709: Log Rotation และการจัดการพื้นที่ดิสก์

### ทำไมต้องมี log rotation

เมื่อเปิด audit logging ระดับละเอียด (โดยเฉพาะ `pgaudit.log = 'READ, WRITE'` บนระบบที่มี transaction สูง) ปริมาณ log ที่เกิดขึ้นจะมหาศาลมาก หากไม่มีการจัดการ log rotation ที่ดี พื้นที่ disk จะเต็มอย่างรวดเร็วและอาจทำให้ **PostgreSQL server หยุดทำงาน** เนื่องจากเขียนข้อมูลลง disk ไม่ได้

PostgreSQL มีกลไก log rotation ในตัวผ่าน `logging_collector` โดยควบคุมด้วยพารามิเตอร์ 2 ตัวหลัก

### log_rotation_age

กำหนดระยะเวลาสูงสุดก่อนที่จะสร้างไฟล์ log ใหม่ (rotate)

```ini
# postgresql.conf
log_rotation_age = 1d    # สร้างไฟล์ log ใหม่ทุก 1 วัน (ค่า default คือ 24h)
```

หน่วยที่ใช้ได้: `ms`, `s`, `min`, `h`, `d` (เช่น `1440min`, `24h`, `1d` มีค่าเท่ากัน)

ค่า `0` = ปิดการ rotate ตามเวลา (rotate ตามขนาดไฟล์อย่างเดียว หรือไม่ rotate เลยถ้าปิดทั้งคู่)

### log_rotation_size

กำหนดขนาดไฟล์สูงสุดก่อนที่จะสร้างไฟล์ log ใหม่

```ini
# postgresql.conf
log_rotation_size = 100MB   # rotate เมื่อไฟล์ log มีขนาดถึง 100MB (ค่า default)
```

ถ้าตั้งค่าทั้ง `log_rotation_age` และ `log_rotation_size` ระบบจะ rotate ทันทีที่เงื่อนไขใดเงื่อนไขหนึ่งถึงก่อน

### ตัวอย่างการตั้งค่าที่สมบูรณ์สำหรับระบบที่มี audit logging ปริมาณสูง

```ini
# postgresql.conf - การจัดการ log rotation สำหรับระบบ e-commerce ที่มี audit logging เปิดเต็มรูปแบบ

logging_collector = on
log_directory = '/var/log/postgresql'
log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'

log_rotation_age = 1d       # rotate ทุกวัน (สอดคล้องกับรอบการ archive log รายวัน)
log_rotation_size = 500MB   # rotate เมื่อไฟล์ใหญ่เกิน 500MB แม้ยังไม่ถึงเวลา

log_truncate_on_rotation = off   # ไม่ overwrite ไฟล์เดิม แม้ชื่อไฟล์จะซ้ำ (ป้องกันข้อมูล audit หาย)
```

> **ข้อควรระวังสำคัญเรื่อง `log_truncate_on_rotation`**: ถ้าตั้ง `log_filename` แบบที่ไม่มี timestamp ละเอียดพอ (เช่น `postgresql-%a.log` ซึ่งจะวนซ้ำทุกสัปดาห์) และเปิด `log_truncate_on_rotation = on` ระบบจะ **ลบเนื้อหาไฟล์เก่าทิ้งเมื่อ rotate มาถึงชื่อไฟล์เดิม** ซึ่งอันตรายมากสำหรับ audit log ที่ต้องเก็บรักษาไว้ตามกฎหมาย ควรตั้ง `log_filename` ให้มี timestamp ที่ไม่ซ้ำกัน (เช่นรวมปี-เดือน-วัน) และปิด truncate ไว้เสมอสำหรับ environment ที่ต้อง compliance

### การจัดการพื้นที่ดิสก์ระยะยาว: log archiving และ retention policy

Log rotation ของ PostgreSQL เองไม่ได้ "ลบ" ไฟล์เก่าให้อัตโนมัติ (ยกเว้นกรณี `log_truncate_on_rotation` ที่ overwrite ไฟล์ชื่อซ้ำ) ดังนั้นต้องมีกลไกภายนอกจัดการ retention เช่น cron job หรือ logrotate ของ Linux

ตัวอย่างการตั้งค่า Linux `logrotate` เพื่อบีบอัดและลบ log เก่าตาม retention policy ของ PCI-DSS (เก็บอย่างน้อย 1 ปี):

```
# /etc/logrotate.d/postgresql-audit
/var/log/postgresql/*.log {
    daily
    rotate 365
    compress
    delaycompress
    missingok
    notifempty
    create 0600 postgres postgres
    postrotate
        # ไม่ต้อง reload PostgreSQL เพราะ PostgreSQL จัดการไฟล์ของตัวเองอยู่แล้วผ่าน logging_collector
        # ส่วนนี้ใช้กรณีต้อง sync กับระบบภายนอก เช่น trigger การ backup log ไป S3
        true
    endscript
}
```

> **ข้อสังเกต**: ในทางปฏิบัติจริง หลายองค์กรจะปล่อยให้ PostgreSQL `logging_collector` จัดการ rotate เองตามขนาด/เวลา แล้วใช้ script แยกต่างหาก (cron job) ในการ **archive ไฟล์ log เก่าไปเก็บที่ cold storage** (เช่น S3 Glacier) หลังผ่านไป N วัน เพื่อประหยัดพื้นที่ disk หลักของ database server

### แนวคิดการส่ง log ไป Centralized Logging

สำหรับองค์กรที่มี PostgreSQL หลาย instance หรือมี microservices จำนวนมาก การเก็บ log แยกไฟล์ในแต่ละเครื่องไม่เพียงพอต่อการวิเคราะห์และตรวจสอบ จึงนิยมส่ง log ไปยังระบบ **centralized logging** ที่รวบรวม log จากทุกแหล่งไว้ที่เดียว ค้นหาและสร้าง dashboard ได้สะดวก

ระบบที่นิยมใช้กันในอุตสาหกรรมมี 2 กลุ่มหลัก:

**1. ELK Stack (Elasticsearch, Logstash, Kibana)**

- **Logstash** หรือ **Filebeat** จะอ่านไฟล์ log ของ PostgreSQL (แนะนำใช้ `csvlog` หรือ `jsonlog` เพราะ parse ง่ายกว่า plain text มาก)
- ส่งข้อมูลเข้า **Elasticsearch** เพื่อ index และค้นหา
- ใช้ **Kibana** สร้าง dashboard และค้นหา log แบบ full-text search

ตัวอย่างแนวคิด configuration ของ Filebeat (เพื่อความเข้าใจ ไม่ใช่ syntax ที่ทดสอบใน environment ของบทนี้):

```yaml
# filebeat.yml (แนวคิดคร่าว ๆ)
filebeat.inputs:
  - type: log
    paths:
      - /var/log/postgresql/*.csv
    fields:
      log_type: postgresql_audit
    fields_under_root: true

output.elasticsearch:
  hosts: ["elasticsearch.internal:9200"]
  index: "postgresql-audit-%{+yyyy.MM.dd}"
```

**2. Grafana Loki**

- ระบบ log aggregation ที่เบากว่า Elasticsearch เพราะไม่ทำ full-text indexing ของเนื้อหา log (index แค่ label) ทำให้ประหยัดทรัพยากรกว่า
- ใช้ **Promtail** เป็น agent อ่าน log แล้วส่งเข้า Loki
- ดูผลผ่าน **Grafana** ซึ่งมักใช้ dashboard เดียวกับที่ใช้ดู metrics (Prometheus) อยู่แล้ว ทำให้เห็นภาพรวมทั้ง log และ metrics ในที่เดียว

ตัวอย่างแนวคิด configuration ของ Promtail:

```yaml
# promtail-config.yaml (แนวคิดคร่าว ๆ)
scrape_configs:
  - job_name: postgresql_audit
    static_configs:
      - targets:
          - localhost
        labels:
          job: postgresql
          log_type: audit
          __path__: /var/log/postgresql/*.csv
```

### ทำไมควรใช้ jsonlog หรือ csvlog แทน plain text เมื่อส่งเข้า centralized logging

การใช้ `log_destination = 'jsonlog'` (PostgreSQL 15+) ทำให้แต่ละบรรทัด log เป็น JSON object ที่มี key-value ชัดเจน ซึ่งระบบอย่าง Logstash/Promtail สามารถ parse โดยไม่ต้องเขียน regex ที่ซับซ้อนเพื่อแยกฟิลด์จาก plain text อีกต่อไป

```ini
# postgresql.conf
log_destination = 'jsonlog'
logging_collector = on
```

ตัวอย่างผลลัพธ์ (ย่อ):

```json
{"timestamp":"2026-09-25 15:30:00.123 +07","pid":21400,"user":"app_user","dbname":"ecommerce","message":"AUDIT: SESSION,1,1,READ,SELECT,TABLE,public.customers,\"SELECT * FROM customers;\",<not logged>"}
```

รูปแบบนี้ทำให้การส่งเข้า Elasticsearch หรือ Loki ทำได้โดยตรงแทบไม่ต้องแปลง format เพิ่มเติม ซึ่งลด engineering effort และลดโอกาสเกิด parsing error ที่อาจทำให้ audit trail ขาดหายไปบางส่วน

---

## สรุปท้ายบท

ในบทนี้เราได้เรียนรู้การสร้างระบบ audit logging ที่ครบวงจรสำหรับ PostgreSQL ตั้งแต่พื้นฐานไปจนถึงระดับที่พร้อมใช้งานจริงในองค์กรที่ต้อง compliance:

1. **ทำไมต้อง audit** — audit logging ไม่ใช่แค่เรื่อง performance debugging แต่เป็น security control ที่จำเป็นสำหรับ compliance (PCI-DSS, GDPR, SOX, HIPAA, PDPA) และการสืบสวนเหตุการณ์ด้านความปลอดภัย
2. **PostgreSQL native logging** — `log_destination`, `logging_collector`, `log_directory`, `log_filename` เป็นรากฐานของระบบ log ที่ต้องตั้งค่าให้ถูกต้องก่อนเสมอ
3. **พารามิเตอร์ log สำคัญ** — `log_min_duration_statement` สำหรับ slow query, `log_statement` สำหรับบันทึกประเภทคำสั่ง, `log_connections`/`log_disconnections` สำหรับติดตาม session
4. **log_line_prefix** — กุญแจสำคัญที่ทำให้ log แต่ละบรรทัดมี context ครบถ้วนพอสำหรับการสืบสวน (user, database, client IP, timestamp)
5. **pgAudit extension** — เครื่องมือ audit ระดับ enterprise ที่ให้ granularity ที่ `log_statement` ธรรมดาทำไม่ได้ ผ่าน `pgaudit.log` (READ/WRITE/DDL/ROLE/...) และ `pgaudit.log_relation`
6. **Session vs Object Audit Logging** — สองโหมดที่เสริมกัน: session audit จับภาพรวมกว้าง ๆ (DDL/ROLE) ส่วน object audit เจาะจงเฉพาะตารางที่มีข้อมูลอ่อนไหว
7. **การอ่าน audit log** — เข้าใจโครงสร้างฟิลด์ของ pgAudit log entry (AUDIT_TYPE, CLASS, COMMAND, OBJECT_NAME, ...) เพื่อดึงข้อมูลออกมาวิเคราะห์และสร้างรายงาน
8. **Log rotation และ centralized logging** — การจัดการพื้นที่ดิสก์ด้วย `log_rotation_age`/`log_rotation_size` และการส่ง log ไปยังระบบรวมศูนย์อย่าง ELK Stack หรือ Grafana Loki เพื่อรองรับการวิเคราะห์และ retention ระยะยาว

การมีระบบ audit ที่ดีคือการสร้างสมดุลระหว่าง **ความครบถ้วนของข้อมูล** กับ **overhead ต่อ performance** — การ audit ทุกอย่างแบบ `pgaudit.log = 'ALL'` โดยไม่คิดหน้าคิดหลังจะทำให้ระบบช้าลงมากและสร้าง log จำนวนมหาศาลที่วิเคราะห์ไม่ทัน ในขณะที่การไม่ audit อะไรเลยก็ทำให้องค์กรเสี่ยงต่อการไม่ผ่าน compliance audit และไม่มีหลักฐานเมื่อเกิดเหตุการณ์ด้านความปลอดภัย กลยุทธ์ที่ดีที่สุดคือการระบุ **ข้อมูลอ่อนไหวที่แท้จริง** (เช่นตาราง `customers` ในระบบของเรา) แล้วใช้ Object Audit Logging เจาะจงเฉพาะจุดนั้น ควบคู่กับ Session Audit Logging ระดับ DDL/ROLE เพื่อครอบคลุมการเปลี่ยนแปลงโครงสร้างและสิทธิ์ของทั้งระบบ

ในบทถัดไป เราจะต่อยอดจากการ audit ไปสู่การ **monitoring** ระบบ PostgreSQL แบบ real-time เพื่อติดตามสุขภาพของระบบ ตรวจจับปัญหาก่อนที่จะลุกลาม และเชื่อมโยงกับเครื่องมืออย่าง Prometheus และ Grafana

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1

จงอธิบายความแตกต่างระหว่าง "logging" ทั่วไปกับ "auditing" ในบริบทของฐานข้อมูล พร้อมยกตัวอย่างสถานการณ์ที่ audit log จำเป็นแต่ log ทั่วไปไม่เพียงพอ

<details>
<summary>เฉลย</summary>

Logging ทั่วไปมีเป้าหมายเพื่อช่วย debug และ performance tuning เช่น บันทึก error, slow query โดยไม่จำเป็นต้องมีข้อมูลครบถ้วนเรื่องตัวตนผู้ใช้หรือ object ที่ถูกเข้าถึง ส่วน auditing มีเป้าหมายเพื่อตอบคำถาม "ใคร ทำอะไร กับข้อมูลไหน เมื่อไหร่" อย่างครบถ้วนและเชื่อถือได้ เพื่อรองรับ compliance และการสืบสวน

ตัวอย่างสถานการณ์: หากลูกค้าร้องเรียนว่าข้อมูลอีเมลของตนถูกเข้าถึงโดยไม่ได้รับอนุญาต การมี log ทั่วไปที่บันทึกแค่ error/slow query จะไม่สามารถตอบได้เลยว่าใครเคย `SELECT` อีเมลของลูกค้ารายนี้บ้าง แต่ audit log (เช่นจาก pgAudit) ที่บันทึกทุกครั้งที่มีการ SELECT บนตาราง customers พร้อมระบุ user และ timestamp จะสามารถตอบคำถามนี้ได้ทันที

</details>

---

### แบบฝึกหัดที่ 2

ระบบ e-commerce ของคุณต้องการ log ทุกคำสั่ง DDL (CREATE, ALTER, DROP) แต่ไม่ต้องการ log คำสั่ง SELECT หรือ INSERT/UPDATE/DELETE เพื่อลด overhead จงเขียนค่าพารามิเตอร์ `log_statement` ที่เหมาะสม

<details>
<summary>เฉลย</summary>

```ini
log_statement = 'ddl'
```

ค่า `ddl` จะทำให้ระบบ log เฉพาะคำสั่งที่เปลี่ยนแปลง schema (CREATE, ALTER, DROP) โดยไม่ log คำสั่ง DML อย่าง SELECT, INSERT, UPDATE, DELETE ซึ่งช่วยลด overhead ได้มากเมื่อเทียบกับ `mod` หรือ `all`

</details>

---

### แบบฝึกหัดที่ 3

จงเขียนค่า `log_line_prefix` ที่รวม timestamp พร้อม milliseconds, username, ชื่อ database, และ remote host เข้าไว้ในบรรทัดเดียว พร้อมอธิบายว่าแต่ละ escape sequence หมายถึงอะไร

<details>
<summary>เฉลย</summary>

```ini
log_line_prefix = '%m user=%u db=%d host=%h '
```

- `%m` = timestamp พร้อม milliseconds
- `%u` = username ที่เชื่อมต่อ
- `%d` = ชื่อ database
- `%h` = remote host ของ connection

ผลลัพธ์ตัวอย่าง: `2026-09-25 15:30:00.123 +07 user=app_user db=ecommerce host=10.0.1.55 `

</details>

---

### แบบฝึกหัดที่ 4

หลังจากแก้ไข `shared_preload_libraries = 'pgaudit'` ใน `postgresql.conf` แล้ว จำเป็นต้องทำขั้นตอนใดต่อไปจึงจะใช้งาน pgAudit ได้จริง

<details>
<summary>เฉลย</summary>

ต้องทำ 2 ขั้นตอน:

1. **Restart** PostgreSQL server (ไม่ใช่แค่ reload) เพราะ `shared_preload_libraries` เป็นพารามิเตอร์ประเภท postmaster context

```bash
sudo systemctl restart postgresql
```

2. รันคำสั่งสร้าง extension ใน database ที่ต้องการใช้งาน

```sql
CREATE EXTENSION IF NOT EXISTS pgaudit;
```

หากข้ามขั้นตอน restart แล้วสร้าง extension เลย จะสร้างสำเร็จแต่ pgAudit จะไม่ทำงานจริง เพราะ hook ยังไม่ถูกลงทะเบียนตอน process หลักเริ่มทำงาน

</details>

---

### แบบฝึกหัดที่ 5

จงตั้งค่า `pgaudit.log` เพื่อ audit เฉพาะคำสั่งที่แก้ไขข้อมูล (INSERT/UPDATE/DELETE) และคำสั่งที่เปลี่ยนแปลงสิทธิ์ (GRANT/REVOKE) แต่ไม่ต้องการ audit คำสั่งอ่านข้อมูล (SELECT)

<details>
<summary>เฉลย</summary>

```sql
SET pgaudit.log = 'WRITE, ROLE';
```

`WRITE` ครอบคลุม INSERT, UPDATE, DELETE, TRUNCATE และ `ROLE` ครอบคลุม GRANT, REVOKE, CREATE/ALTER/DROP ROLE โดยไม่รวม `READ` ซึ่งครอบคลุม SELECT

</details>

---

### แบบฝึกหัดที่ 6

จงอธิบายความแตกต่างระหว่าง Session Audit Logging กับ Object Audit Logging ใน pgAudit และยกตัวอย่างสถานการณ์ที่ควรใช้แต่ละแบบ

<details>
<summary>เฉลย</summary>

**Session Audit Logging**: ตั้งค่าผ่าน `pgaudit.log` (เช่น READ, WRITE, DDL) จะ audit statement ที่ตรงกับ class ที่กำหนดสำหรับ **ทุก session** โดยไม่จำกัดเฉพาะ object ใด เหมาะกับการ audit ภาพรวม เช่น ทุกคำสั่ง DDL หรือ ROLE change ในระบบ เพื่อ SOX compliance

**Object Audit Logging**: ตั้งค่าผ่าน `pgaudit.role` โดยสร้าง role พิเศษ (เช่น `pgaudit_object`) แล้ว `GRANT` สิทธิ์เฉพาะ object ที่ต้องการ audit ให้กับ role นั้น ทำให้ audit เจาะจงเฉพาะตาราง/คอลัมน์ที่สนใจ เหมาะกับการ audit ข้อมูลอ่อนไหวโดยเฉพาะ เช่น ตาราง customers ที่มี PII เพื่อ GDPR/PCI-DSS compliance โดยไม่ต้อง audit ตารางอื่นที่ไม่อ่อนไหว เช่น products

ในทางปฏิบัติมักใช้ทั้งสองแบบร่วมกัน: session audit สำหรับ DDL/ROLE ทั้งระบบ + object audit สำหรับตารางข้อมูลอ่อนไหวโดยเฉพาะ

</details>

---

### แบบฝึกหัดที่ 7

จากตัวอย่าง log entry ต่อไปนี้ จงระบุว่า user ใดเป็นผู้กระทำ, กระทำอะไร, กับ object ใด และค่าพารามิเตอร์ที่ใช้คืออะไร

```
2026-09-25 16:00:12.500 +07 [21500] user=support_agent,db=ecommerce,client=10.0.2.30 LOG:  AUDIT: OBJECT,7,1,WRITE,UPDATE,TABLE,public.orders,"UPDATE orders SET status = $1 WHERE order_id = $2;",'CANCELLED','105'
```

<details>
<summary>เฉลย</summary>

- **User**: `support_agent`
- **Database**: `ecommerce`
- **Client IP**: `10.0.2.30`
- **AUDIT_TYPE**: `OBJECT` (เป็น object audit logging ไม่ใช่ session audit)
- **Class**: `WRITE`
- **Command**: `UPDATE`
- **Object**: ตาราง `public.orders`
- **Statement**: `UPDATE orders SET status = $1 WHERE order_id = $2;`
- **Parameter values**: `$1 = 'CANCELLED'`, `$2 = '105'`

สรุป: ผู้ใช้ `support_agent` จาก IP `10.0.2.30` ได้เปลี่ยนสถานะคำสั่งซื้อ `order_id = 105` เป็น `CANCELLED` เมื่อเวลา 16:00:12.500 น.

</details>

---

### แบบฝึกหัดที่ 8

จงเขียนค่า `log_rotation_age` และ `log_rotation_size` ที่เหมาะสมสำหรับระบบที่มี audit logging ปริมาณสูง โดยต้องการให้สร้างไฟล์ log ใหม่ทุกวัน หรือเมื่อไฟล์มีขนาดเกิน 500MB (แล้วแต่อย่างใดถึงก่อน) พร้อมอธิบายว่าทำไมควรปิด `log_truncate_on_rotation`

<details>
<summary>เฉลย</summary>

```ini
log_rotation_age = 1d
log_rotation_size = 500MB
log_truncate_on_rotation = off
```

ควรปิด `log_truncate_on_rotation` (ตั้งเป็น `off`) เพราะถ้าชื่อไฟล์ log ที่กำหนดใน `log_filename` มีโอกาสซ้ำกัน (เช่นวนซ้ำทุกสัปดาห์) การเปิด truncate จะทำให้ระบบ **overwrite เนื้อหาไฟล์เก่าทิ้ง** เมื่อ rotate มาถึงชื่อไฟล์เดิม ซึ่งเป็นอันตรายมากสำหรับ audit log ที่ต้องเก็บรักษาตามกฎหมาย (เช่น PCI-DSS กำหนดให้เก็บอย่างน้อย 1 ปี) การปิด truncate จะทำให้ PostgreSQL เขียนต่อท้ายไฟล์แทน ป้องกันข้อมูล audit สูญหาย

</details>

---

### แบบฝึกหัดที่ 9

องค์กรของคุณมี PostgreSQL หลาย instance กระจายอยู่หลายเครื่อง และต้องการรวบรวม audit log ทั้งหมดไว้ที่เดียวเพื่อให้ทีม security ค้นหาและวิเคราะห์ได้สะดวก จงอธิบายแนวทางการแก้ปัญหานี้ พร้อมยกตัวอย่างเครื่องมือที่ใช้ได้ 2 ระบบ และเหตุผลที่ควรใช้ `jsonlog` หรือ `csvlog` แทน plain text log

<details>
<summary>เฉลย</summary>

แนวทางแก้ปัญหาคือการส่ง log จากทุก instance เข้าสู่ระบบ **centralized logging** ที่รวบรวม log ไว้ที่เดียว เครื่องมือที่นิยมใช้มี 2 กลุ่ม:

1. **ELK Stack** (Elasticsearch, Logstash/Filebeat, Kibana) — ใช้ Filebeat อ่านไฟล์ log แล้วส่งเข้า Elasticsearch เพื่อ index และค้นหา ใช้ Kibana สร้าง dashboard

2. **Grafana Loki** — ใช้ Promtail อ่าน log แล้วส่งเข้า Loki ซึ่งเบากว่า Elasticsearch เพราะ index เฉพาะ label ไม่ index เนื้อหาเต็ม ดูผลผ่าน Grafana ซึ่งมักใช้ dashboard เดียวกับ metrics อยู่แล้ว

ควรใช้ `jsonlog` หรือ `csvlog` แทน plain text เพราะมีโครงสร้างฟิลด์ชัดเจน (structured format) ทำให้เครื่องมือ Logstash/Promtail สามารถ parse ได้โดยตรงโดยไม่ต้องเขียน regex ซับซ้อนเพื่อแยกฟิลด์จากข้อความ ลด engineering effort และลดโอกาสเกิด parsing error ที่อาจทำให้ audit trail ขาดหายไปบางส่วน

</details>

---

### แบบฝึกหัดที่ 10 (แบบฝึกหัดรวม)

คุณเป็น DBA ของบริษัท e-commerce ที่เพิ่งเริ่มรับชำระเงินผ่านบัตรเครดิตโดยตรง (ผ่าน payment gateway) ทำให้ระบบต้องปฏิบัติตามมาตรฐาน **PCI-DSS**  จงออกแบบระบบ audit logging ที่ครบถ้วน โดยระบุ:

1. การตั้งค่า PostgreSQL native logging (`log_destination`, `log_line_prefix`, `log_connections`/`log_disconnections`)
2. การตั้งค่า pgAudit (`pgaudit.log`, session vs object audit) สำหรับตาราง `customers` และ `orders`
3. นโยบาย log rotation และ retention ที่สอดคล้องกับข้อกำหนด PCI-DSS Requirement 10 (เก็บ log อย่างน้อย 1 ปี)
4. แนวทางการส่ง log ไป centralized logging

<details>
<summary>เฉลย</summary>

**1. PostgreSQL native logging**

```ini
# postgresql.conf
log_destination = 'csvlog'
logging_collector = on
log_directory = '/var/log/postgresql'
log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'
log_file_mode = 0600

log_line_prefix = '%m [%p:%l] user=%u,db=%d,app=%a,client=%h,vtid=%v,txid=%x '

log_connections = on
log_disconnections = on
log_min_duration_statement = 1000
log_statement = 'ddl'
```

เหตุผล: ใช้ `csvlog` เพื่อให้ parse ง่าย, `log_line_prefix` ครบถ้วนพอสำหรับสืบสวน (user, database, client, transaction id), เปิด connection/disconnection logging เพื่อติดตาม session ทั้งหมด, ใช้ `log_statement = 'ddl'` แทนที่จะเป็น `mod`/`all` เพราะจะใช้ pgAudit จัดการ DML แทนซึ่งให้ granularity ดีกว่า

**2. pgAudit configuration**

```ini
# postgresql.conf
shared_preload_libraries = 'pgaudit'
pgaudit.log = 'DDL, ROLE'          # session audit สำหรับภาพรวมทั้งระบบ
pgaudit.role = 'pgaudit_object'    # object audit สำหรับข้อมูลอ่อนไหว
pgaudit.log_relation = on
pgaudit.log_parameter = on
pgaudit.log_catalog = off
```

```sql
CREATE EXTENSION IF NOT EXISTS pgaudit;

CREATE ROLE pgaudit_object NOLOGIN;

-- audit การเข้าถึงตาราง customers และ orders อย่างละเอียด (ข้อมูลลูกค้าและธุรกรรม)
GRANT SELECT, INSERT, UPDATE, DELETE ON customers TO pgaudit_object;
GRANT SELECT, INSERT, UPDATE, DELETE ON orders TO pgaudit_object;

SELECT pg_reload_conf();
```

เหตุผล: session audit จับ DDL/ROLE ทั้งระบบเพื่อความครบถ้วนตาม PCI-DSS Requirement 10.2 (การเปลี่ยนแปลงสิทธิ์และโครงสร้าง) ส่วน object audit เจาะจงตาราง `customers` (มี PII) และ `orders` (มีความเชื่อมโยงกับธุรกรรมการเงิน) เพื่อบันทึกทุกการเข้าถึงระดับแถวโดยไม่ต้อง audit ตาราง `products` ที่ไม่อ่อนไหวเท่า ลด noise ของ log

**3. Log rotation และ retention**

```ini
log_rotation_age = 1d
log_rotation_size = 500MB
log_truncate_on_rotation = off
```

```
# /etc/logrotate.d/postgresql-audit
/var/log/postgresql/*.log {
    daily
    rotate 365
    compress
    delaycompress
    missingok
    notifempty
    create 0600 postgres postgres
}
```

เหตุผล: PCI-DSS Requirement 10.7 กำหนดให้เก็บ audit log อย่างน้อย 1 ปี โดยข้อมูล 3 เดือนล่าสุดต้องพร้อมใช้งานได้ทันที (online) การตั้ง `rotate 365` ใน logrotate ครอบคลุมการเก็บย้อนหลัง 1 ปี ร่วมกับการ compress ไฟล์เก่าเพื่อประหยัดพื้นที่ และปิด `log_truncate_on_rotation` ป้องกันข้อมูล audit หายจากการ overwrite

**4. Centralized logging**

ใช้ Filebeat หรือ Promtail อ่านไฟล์ `csvlog` (หรือเปลี่ยนเป็น `jsonlog` ถ้าต้องการ integration ที่ง่ายกว่า) ส่งเข้า Elasticsearch (ELK Stack) หรือ Loki (Grafana) เพื่อให้ทีม security ค้นหาและสร้าง alert อัตโนมัติได้ เช่น alert เมื่อมีการ `DELETE` จำนวนมากผิดปกติบนตาราง `orders` ภายในเวลาสั้น ๆ ซึ่งอาจเป็นสัญญาณของการโจมตีหรือข้อผิดพลาดร้ายแรง พร้อมทั้งตั้งค่า retention policy บนระบบ centralized logging ให้สอดคล้องกับ 1 ปีตามข้อกำหนดเดียวกัน เพื่อให้มี audit trail สำรองแยกจาก local disk ของ database server ด้วย

ระบบที่ออกแบบมาทั้งหมดนี้ตอบโจทย์ทั้ง 3 เสาหลักของการ audit ที่ดี — **Completeness** (ครอบคลุมทั้ง DDL/ROLE ทั้งระบบและ DML บนข้อมูลอ่อนไหว), **Integrity** (ป้องกันการ overwrite/truncate, สิทธิ์ไฟล์จำกัดเฉพาะ postgres user), และ **Reviewability** (format ที่ parse ง่ายและส่งเข้า centralized logging ให้ทีม security ตรวจสอบได้สะดวก)

</details>

---

**บทถัดไป**: [Part 072: Monitoring](./part-072-monitoring.md)
