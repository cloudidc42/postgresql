# Data Types วันที่และเวลา (date, time, timestamp, interval)

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับพื้นฐาน | Part 007

---

## เป้าหมายการเรียนรู้

หลังจากเรียนจบ Part นี้ ผู้เรียนจะสามารถ:

- อธิบายความแตกต่างระหว่าง `date`, `time`, `timestamp`, `timestamptz` และ `interval` ได้อย่างถูกต้อง
- เลือกใช้ data type วันที่/เวลาที่เหมาะสมกับสถานการณ์จริงในงาน production
- เข้าใจกลไกการทำงานของ time zone ใน PostgreSQL อย่างลึกซึ้ง โดยเฉพาะความแตกต่างระหว่าง `timestamp` กับ `timestamptz` ซึ่งเป็นจุดที่มือใหม่เข้าใจผิดมากที่สุด
- ใช้คำสั่ง `SET TIME ZONE` และ `AT TIME ZONE` เพื่อแปลงเวลาข้าม time zone ได้อย่างถูกต้อง
- สร้างและคำนวณค่า `interval` เพื่อบวกลบวันที่/เวลาได้
- ใช้ฟังก์ชัน `AGE()`, `EXTRACT()` เพื่อดึงข้อมูลย่อยจากวันที่/เวลา
- แปลงรูปแบบวันที่ไปมาระหว่าง string กับ date/timestamp ด้วย `TO_CHAR`, `TO_DATE`, `TO_TIMESTAMP`
- รู้จักปัญหาที่พบบ่อยเกี่ยวกับ DST, leap year และการเก็บวันเกิดข้าม time zone พร้อมวิธีป้องกัน
- นำ best practices ของการจัดการเวลาในแอปพลิเคชันจริงไปใช้งานได้ทันที (เก็บเป็น UTC เสมอ แปลงตอนแสดงผล)

---

## บทนำ: ทำไม Data Type วันที่/เวลาถึงสำคัญมาก

ในระบบฐานข้อมูลของงานจริงแทบทุกระบบ ไม่ว่าจะเป็นระบบ e-commerce, ระบบจองตั๋ว, ระบบบัญชี หรือระบบ log เหตุการณ์ ล้วนต้องเก็บ "เวลา" ไม่ทางใดก็ทางหนึ่ง เช่น วันที่สั่งซื้อ, เวลาที่ล็อกอิน, วันเกิดของลูกค้า, เวลานัดหมาย ฯลฯ

ปัญหาที่พบบ่อยที่สุดในระบบที่ทำงานข้ามหลาย time zone (เช่นระบบที่มีผู้ใช้ทั้งในไทย สหรัฐฯ และยุโรป) คือการเลือก data type ผิดตั้งแต่ต้น แล้วต้องมาแก้ไขทีหลังซึ่งยากและเสี่ยงต่อข้อมูลผิดพลาด PostgreSQL ให้เครื่องมือที่ครบถ้วนและถูกต้องมากสำหรับจัดการเรื่องนี้ แต่ผู้ใช้ต้องเข้าใจกลไกเบื้องหลังก่อนจึงจะใช้ได้อย่างปลอดภัย

Part นี้จะพาไปทำความเข้าใจตั้งแต่พื้นฐานไปจนถึงกับดักที่แม้แต่นักพัฒนาที่มีประสบการณ์ก็ยังพลาดบ่อย

---

## Step 61: ภาพรวม Date/Time Types ใน PostgreSQL

PostgreSQL มี data type สำหรับวันที่และเวลาหลักๆ 5 ชนิด แต่ละชนิดเหมาะกับงานที่ต่างกัน

| Type | ขนาด | ช่วงค่า | ความละเอียด | ตัวอย่าง |
|---|---|---|---|---|
| `date` | 4 bytes | 4713 BC – 5874897 AD | 1 วัน | `2026-09-25` |
| `time [without time zone]` | 8 bytes | 00:00:00 – 24:00:00 | 1 microsecond | `14:30:00` |
| `time with time zone` | 12 bytes | 00:00:00+1559 – 24:00:00-1559 | 1 microsecond | `14:30:00+07` |
| `timestamp [without time zone]` | 8 bytes | 4713 BC – 294276 AD | 1 microsecond | `2026-09-25 14:30:00` |
| `timestamp with time zone` (`timestamptz`) | 8 bytes | 4713 BC – 294276 AD | 1 microsecond | `2026-09-25 14:30:00+07` |
| `interval` | 16 bytes | ±178,000,000 ปี | 1 microsecond | `1 day 02:30:00` |

### สรุปสั้นๆ ว่าแต่ละตัวใช้ทำอะไร

- **`date`** — เก็บเฉพาะ "วันที่" ไม่มีเวลา เหมาะกับวันเกิด, วันหมดอายุสัญญา, วันที่ในใบเสร็จ (ถ้าไม่สนใจเวลาที่แน่นอน)
- **`time`** — เก็บเฉพาะ "เวลาในหนึ่งวัน" ไม่มีวันที่ เหมาะกับเวลาเปิด-ปิดร้าน, เวลานัดหมายประจำวัน
- **`timestamp`** — เก็บทั้งวันที่และเวลา แต่ **ไม่ผูกกับ time zone ใดๆ** (เป็นแค่ตัวเลข "หน้าปัดนาฬิกา" เฉยๆ)
- **`timestamptz`** — เก็บทั้งวันที่และเวลา และ **ผูกกับจุดเวลาที่แน่นอนบนโลก (absolute point in time)** — นี่คือตัวที่ควรใช้เป็นค่าเริ่มต้นสำหรับแอปพลิเคชันเกือบทุกกรณี
- **`interval`** — เก็บ "ช่วงเวลา" หรือ "ระยะเวลา" ไม่ใช่จุดเวลา เช่น "3 วัน", "2 ชั่วโมง 30 นาที"

ลองดูตัวอย่างการสร้างตารางที่ใช้ type เหล่านี้ทั้งหมด:

```sql
CREATE TABLE event_log (
    id              BIGSERIAL PRIMARY KEY,
    event_name      TEXT NOT NULL,
    event_date      DATE,               -- วันที่จัดงาน (ไม่สนใจเวลา)
    open_time       TIME,               -- เวลาเปิดรับลงทะเบียนในแต่ละวัน
    created_at      TIMESTAMP,          -- (ตัวอย่างแบบไม่แนะนำ ดู Step 64)
    starts_at       TIMESTAMPTZ NOT NULL DEFAULT now(),  -- เวลาเริ่มงานจริง (แนะนำ)
    duration        INTERVAL            -- ระยะเวลาที่ใช้จัดงาน
);
```

```
CREATE TABLE
```

ลองใส่ข้อมูลตัวอย่างและดูว่า PostgreSQL แสดงผลแต่ละ type อย่างไร:

```sql
INSERT INTO event_log (event_name, event_date, open_time, starts_at, duration)
VALUES ('PostgreSQL Meetup Bangkok', '2026-10-15', '08:30:00',
        '2026-10-15 18:00:00+07', INTERVAL '3 hours');

SELECT event_name, event_date, open_time, starts_at, duration
FROM event_log;
```

```
        event_name         | event_date |  open_time  |        starts_at         | duration
----------------------------+------------+-------------+---------------------------+----------
 PostgreSQL Meetup Bangkok  | 2026-10-15 | 08:30:00    | 2026-10-15 18:00:00+07    | 03:00:00
(1 row)
```

### ตรวจสอบ type จริงในระบบ

```sql
SELECT pg_typeof('2026-09-25'::date)                 AS t1,
       pg_typeof('14:30:00'::time)                    AS t2,
       pg_typeof('2026-09-25 14:30:00'::timestamp)    AS t3,
       pg_typeof(now())                               AS t4,
       pg_typeof(INTERVAL '1 day')                    AS t5;
```

```
    t1    |   t2   |          t3           |            t4            |   t5
----------+--------+------------------------+---------------------------+----------
 date     | time   | timestamp without ... | timestamp with time zone | interval
(1 row)
```

สังเกตว่า `now()` คืนค่าเป็น `timestamptz` เสมอ — นี่คือฟังก์ชันมาตรฐานที่ PostgreSQL แนะนำให้ใช้แทนการใช้ `timestamp` เฉยๆ

---

## Step 62: DATE Type — Format การป้อนค่าและฟังก์ชัน CURRENT_DATE

### รูปแบบการป้อนค่า (input formats)

PostgreSQL ยอมรับ format วันที่ได้หลายรูปแบบ แต่รูปแบบที่ปลอดภัยและแนะนำที่สุดคือ **ISO 8601** (`YYYY-MM-DD`) เพราะไม่กำกวมและตีความได้เหมือนกันทุก locale

```sql
SELECT '2026-09-25'::date  AS iso_format,
       'September 25, 2026'::date AS text_format,
       '09/25/2026'::date AS us_format,   -- MM/DD/YYYY (ขึ้นกับ DateStyle)
       '25-09-2026'::date AS dmy_format;  -- ตีความตาม DateStyle setting
```

```
 iso_format | text_format | us_format  | dmy_format
------------+-------------+------------+------------
 2026-09-25 | 2026-09-25  | 2026-09-25 | 2026-09-25
(1 row)
```

> **คำเตือนสำคัญ:** รูปแบบ `MM/DD/YYYY` กับ `DD/MM/YYYY` จะตีความต่างกันขึ้นอยู่กับค่า `DateStyle` ของ session ซึ่งเป็นสาเหตุของบั๊กที่พบบ่อยมาก เช่น `'03/04/2026'` อาจถูกตีความเป็น 3 เมษายน หรือ 4 มีนาคม ก็ได้ ขึ้นอยู่กับการตั้งค่า **ควรใช้ ISO format (`YYYY-MM-DD`) เสมอในโค้ด production เพื่อหลีกเลี่ยงความกำกวมนี้โดยสิ้นเชิง**

ตรวจสอบค่า DateStyle ปัจจุบัน:

```sql
SHOW DateStyle;
```

```
 DateStyle
------------
 ISO, MDY
(1 row)
```

### CURRENT_DATE และฟังก์ชันที่เกี่ยวข้อง

```sql
SELECT CURRENT_DATE,
       CURRENT_DATE - 1 AS yesterday,
       CURRENT_DATE + 7 AS next_week,
       CURRENT_DATE + INTERVAL '1 month' AS next_month;
```

```
 current_date | yesterday  | next_week  |      next_month
--------------+------------+------------+------------------------
 2026-09-25   | 2026-09-24 | 2026-10-02 | 2026-10-25 00:00:00
(1 row)
```

สังเกตว่า `date + integer` จะได้ `date` กลับมา แต่ `date + interval` จะกลายเป็น `timestamp` เพราะ interval สามารถมีเศษเวลาที่ไม่ใช่จำนวนวันเต็มได้

### การใช้งานจริง: หาคนที่เกิดในเดือนนี้

```sql
CREATE TABLE customer (
    id         SERIAL PRIMARY KEY,
    full_name  TEXT NOT NULL,
    birth_date DATE
);

INSERT INTO customer (full_name, birth_date) VALUES
    ('สมชาย ใจดี',   '1990-09-12'),
    ('สมหญิง รักเรียน', '1985-03-04'),
    ('วิชัย มั่นคง',   '1992-09-30');

SELECT full_name, birth_date
FROM customer
WHERE EXTRACT(MONTH FROM birth_date) = EXTRACT(MONTH FROM CURRENT_DATE);
```

```
   full_name   | birth_date
----------------+------------
 สมชาย ใจดี      | 1990-09-12
 วิชัย มั่นคง     | 1992-09-30
(2 rows)
```

### ค่าพิเศษที่ใช้ได้กับ date

```sql
SELECT 'today'::date, 'tomorrow'::date, 'yesterday'::date, 'epoch'::date, 'infinity'::date;
```

```
   date     |   date     |   date     |   date     |   date
------------+------------+------------+------------+----------
 2026-09-25 | 2026-09-26 | 2026-09-24 | 1970-01-01 | infinity
(1 row)
```

`infinity` และ `-infinity` มีประโยชน์มากสำหรับแทนค่า "ไม่มีวันสิ้นสุด" เช่นในตาราง history ที่ใช้ range `[valid_from, valid_to)`

---

## Step 63: TIME และ TIME WITH TIME ZONE

### TIME พื้นฐาน

`time` เก็บเฉพาะเวลาในหนึ่งวัน (00:00:00 ถึง 24:00:00) โดยไม่มีข้อมูลวันที่หรือ time zone

```sql
SELECT '14:30:00'::time,
       '14:30:00.123456'::time,
       '2:30 PM'::time,
       CURRENT_TIME::time AS now_time;
```

```
   time   |      time       |   time   |  now_time
----------+-----------------+----------+-------------
 14:30:00 | 14:30:00.123456 | 14:30:00 | 09:12:33.482
(1 row)
```

### TIME WITH TIME ZONE คืออะไร

`time with time zone` (`timetz`) เก็บเวลาพร้อม offset ของ time zone เช่น `14:30:00+07`

```sql
SELECT '14:30:00+07'::timetz,
       '14:30:00-05'::timetz;
```

```
   timetz    |   timetz
-------------+-------------
 14:30:00+07 | 14:30:00-05
(1 row)
```

### เมื่อไหร่ควรใช้ / ไม่ควรใช้ TIME WITH TIME ZONE

มาตรฐาน SQL กำหนดให้มี `time with time zone` แต่ในทางปฏิบัติ **PostgreSQL official documentation แนะนำอย่างชัดเจนว่าไม่ควรใช้ type นี้** เพราะมีปัญหาเชิงแนวคิดที่แก้ไม่ได้:

1. **การเปรียบเทียบทำได้ไม่สมบูรณ์** — เวลา `08:00+07` (ประเทศไทย) กับ `01:00+00` (UTC) เป็นเวลาเดียวกันจริง แต่การเปรียบเทียบ `timetz` ไม่ได้คำนึงถึง DST หรือการเปลี่ยนแปลงกฎ time zone เพราะไม่มีข้อมูล "วันที่" มาประกอบการคำนวณ
2. **ไม่รู้ว่าเป็นวันไหน** — DST (Daylight Saving Time) เปลี่ยนแปลงตามวันที่ ถ้าไม่มีวันที่กำกับ ก็ไม่สามารถรู้ได้ว่า offset ที่ถูกต้องคือเท่าไหร่ในบาง time zone
3. **ใช้พื้นที่เก็บข้อมูลมากกว่า** (12 bytes เทียบกับ 8 bytes ของ `time`)

```sql
-- ตัวอย่างปัญหา: การเรียงลำดับที่ดูขัดกับสามัญสำนึก
SELECT t, t AT TIME ZONE 'UTC' AS utc_equivalent
FROM (VALUES ('23:00+07'::timetz), ('01:00-05'::timetz)) AS x(t)
ORDER BY t;
```

```
    t     | utc_equivalent
----------+-----------------
 23:00+07 | 16:00:00
 01:00-05 | 06:00:00
(2 rows)
```

จะเห็นว่าเมื่อเรียงตาม `timetz` โดยตรง ผลลัพธ์อาจดูสับสนเพราะการเปรียบเทียบใช้ทั้งเวลาและ offset รวมกันแบบที่ไม่สอดคล้องกับ "เวลาจริงในโลก" เสมอไป

**ข้อสรุป:** ใช้ `time` (without time zone) เพื่อเก็บ "เวลาในหนึ่งวันแบบ local" เช่น เวลาเปิดร้านทุกวัน (`09:00`) โดยเก็บ time zone ของสถานที่แยกไว้ในคอลัมน์อื่น (เช่น `Asia/Bangkok`) แทนที่จะใช้ `timetz` เลย

```sql
CREATE TABLE store_hours (
    store_id   INT,
    open_time  TIME NOT NULL,
    close_time TIME NOT NULL,
    tz_name    TEXT NOT NULL DEFAULT 'Asia/Bangkok'  -- แยก timezone ออกมาเป็นคอลัมน์ชัดเจน
);

INSERT INTO store_hours VALUES (1, '09:00', '21:00', 'Asia/Bangkok');

SELECT * FROM store_hours;
```

```
 store_id | open_time | close_time |   tz_name
----------+-----------+------------+---------------
        1 | 09:00:00  | 21:00:00   | Asia/Bangkok
(1 row)
```

วิธีนี้ยืดหยุ่นและชัดเจนกว่า `timetz` มาก และเป็นแนวทางที่แนะนำในงาน production จริง

---

## Step 64: TIMESTAMP vs TIMESTAMPTZ — จุดที่มือใหม่เข้าใจผิดมากที่สุด

นี่คือหัวข้อที่ **สำคัญที่สุด** ใน Part นี้ เพราะเป็นความเข้าใจผิดที่พบบ่อยที่สุดของผู้เริ่มต้น และเป็นสาเหตุของบั๊กร้ายแรงในระบบจริงจำนวนมาก

### แนวคิดพื้นฐาน

- **`timestamp` (without time zone)** — เก็บ "ตัวเลขวันที่+เวลา" เฉยๆ โดยไม่มีข้อมูลว่ามันคือเวลาโซนไหน เปรียบเสมือนการเขียนบนกระดาษว่า "14:30" โดยไม่บอกว่าเป็นเวลาประเทศไหน
- **`timestamptz` (with time zone)** — เก็บ "จุดเวลาที่แน่นอนบนเส้นเวลาของจักรวาล" (absolute instant) โดยภายในจะถูกแปลงและเก็บเป็น **UTC เสมอ** ไม่ว่าจะป้อนด้วย time zone ใดก็ตาม แล้วเมื่อแสดงผลจะแปลงกลับเป็น time zone ของ session ที่กำลังอ่านอยู่

> ชื่อ `timestamptz` อาจทำให้เข้าใจผิดว่า "เก็บ time zone ไว้ด้วย" แต่ในความเป็นจริง **PostgreSQL ไม่ได้เก็บ time zone ที่คุณป้อนไว้เลย** มันแค่ใช้ time zone ที่ป้อนมาเพื่อคำนวณแปลงเป็น UTC ตอน insert เท่านั้น แล้วข้อมูลจริงที่เก็บใน disk คือ UTC timestamp (Unix-like) ล้วนๆ

### พิสูจน์ด้วยตัวอย่างจริง

ลองสร้างตารางเปรียบเทียบทั้งสอง type:

```sql
SET TIME ZONE 'Asia/Bangkok';  -- UTC+7

CREATE TABLE tz_demo (
    label TEXT,
    ts_no_tz   TIMESTAMP,
    ts_with_tz TIMESTAMPTZ
);

INSERT INTO tz_demo VALUES
    ('insert เวลาไทย 14:30', '2026-09-25 14:30:00', '2026-09-25 14:30:00');

SELECT * FROM tz_demo;
```

```
        label         |      ts_no_tz       |        ts_with_tz
-----------------------+---------------------+---------------------------
 insert เวลาไทย 14:30  | 2026-09-25 14:30:00 | 2026-09-25 14:30:00+07
(1 row)
```

ตอนนี้ดูยังไม่ต่างกันมาก แต่ **จุดสำคัญคือเมื่อเปลี่ยน time zone ของ session** แล้วอ่านค่าเดิมอีกครั้ง:

```sql
SET TIME ZONE 'UTC';

SELECT * FROM tz_demo;
```

```
        label         |      ts_no_tz       |        ts_with_tz
-----------------------+---------------------+---------------------------
 insert เวลาไทย 14:30  | 2026-09-25 14:30:00 | 2026-09-25 07:30:00+00
(1 row)
```

**นี่คือความแตกต่างที่สำคัญที่สุด:**

- `ts_no_tz` (`timestamp`) — **ค่าเหมือนเดิมทุกประการ** (`14:30:00`) ไม่ว่าจะดูจาก session ไหน เพราะมันไม่รู้จัก time zone เลย มันเป็นแค่ "ตัวเลข" ที่ตายตัว
- `ts_with_tz` (`timestamptz`) — **ค่าที่แสดงผลเปลี่ยนไปเป็น `07:30:00+00`** เพราะภายในเก็บเป็น UTC (`2026-09-25T07:30:00Z`) แล้วแปลงเป็น time zone ของ session ปัจจุบัน (ตอนนี้คือ UTC) ให้อัตโนมัติ — ทั้งสองค่านี้แท้จริงแล้ว **คือเวลาเดียวกัน** เพียงแต่แสดงผลต่างมุมมอง

ลองดูอีกมุมหนึ่ง สมมติมีผู้ใช้จาก New York (UTC-4 ในช่วง DST) อ่านค่าเดียวกัน:

```sql
SET TIME ZONE 'America/New_York';

SELECT label, ts_no_tz, ts_with_tz FROM tz_demo;
```

```
        label         |      ts_no_tz       |         ts_with_tz
-----------------------+---------------------+----------------------------
 insert เวลาไทย 14:30  | 2026-09-25 14:30:00 | 2026-09-25 03:30:00-04
(1 row)
```

`ts_with_tz` แปลงเป็น `03:30:00-04` ของ New York โดยอัตโนมัติ แต่ยังคงเป็น **เวลาเดียวกันในโลกแห่งความจริง** กับ `14:30:00+07` ที่ Bangkok และ `07:30:00+00` ที่ UTC ทุกประการ (ตรวจสอบได้ด้วยการแปลงเป็น epoch):

```sql
SELECT ts_with_tz, extract(epoch FROM ts_with_tz) AS unix_epoch
FROM tz_demo;
```

```
         ts_with_tz          | unix_epoch
------------------------------+------------
 2026-09-25 03:30:00-04       | 1758785400
(1 row)
```

ค่า `unix_epoch` จะเหมือนกันเป๊ะไม่ว่าจะตั้ง `TIME ZONE` เป็นอะไรก็ตาม เพราะมันคือจุดเวลาเดียวกันบนเส้นเวลาจริง

### ทำไมนี่คือกับดักของมือใหม่

ปัญหาที่พบบ่อยที่สุดคือ นักพัฒนาใช้ `timestamp` (ไม่มี tz) เก็บเวลาที่ "ควรจะเป็น timestamptz" เช่น `created_at`, `updated_at`, เวลาที่เกิด transaction ทางการเงิน ฯลฯ แล้วเมื่อระบบขยายไปให้บริการผู้ใช้หลาย time zone หรือย้าย server ไปอยู่ใน region อื่น (เช่นเปลี่ยนจาก server ที่ตั้ง timezone เป็น Asia/Bangkok ไปเป็น server ที่ตั้งเป็น UTC) **ข้อมูลเดิมทั้งหมดจะตีความผิดทันที** เพราะ `timestamp` ไม่มีข้อมูล time zone กำกับไว้เลยว่าตอน insert เวลานั้นหมายถึง time zone ไหน

```sql
-- ตัวอย่างปัญหาจริง: สมมติแอปนี้ตั้งใจเก็บ "เวลาไทย" ด้วย timestamp ธรรมดา
CREATE TABLE orders_bad (
    id          SERIAL PRIMARY KEY,
    order_time  TIMESTAMP DEFAULT CURRENT_TIMESTAMP  -- ผิดพลาด! ไม่ควรใช้แบบนี้
);

-- server ตั้ง timezone Asia/Bangkok ตอน insert
SET TIME ZONE 'Asia/Bangkok';
INSERT INTO orders_bad (order_time) VALUES ('2026-09-25 20:00:00');

-- ภายหลัง server ย้ายไป region อื่น เปลี่ยน timezone เป็น UTC
SET TIME ZONE 'UTC';
SELECT * FROM orders_bad;
```

```
 id |     order_time
----+---------------------
  1 | 2026-09-25 20:00:00
(1 row)
```

ค่ายัง "ดูเหมือนเดิม" (`20:00:00`) แต่ตอนนี้แอปพลิเคชันที่ทำงานบน UTC จะตีความว่านี่คือ 20:00 UTC ทั้งที่จริงแล้วมันคือ 20:00 เวลาไทย (= 13:00 UTC) — **เกิดความผิดพลาดของเวลาไป 7 ชั่วโมงโดยไม่มีทางรู้ได้เลยจากข้อมูลเพียงอย่างเดียว**

### ทำไมควรใช้ TIMESTAMPTZ แทบทุกกรณี

```sql
-- แบบที่ถูกต้อง
CREATE TABLE orders_good (
    id          SERIAL PRIMARY KEY,
    order_time  TIMESTAMPTZ NOT NULL DEFAULT now()
);

SET TIME ZONE 'Asia/Bangkok';
INSERT INTO orders_good (order_time) VALUES ('2026-09-25 20:00:00+07');

SET TIME ZONE 'UTC';
SELECT * FROM orders_good;
```

```
 id |       order_time
----+---------------------------
  1 | 2026-09-25 13:00:00+00
(1 row)
```

ค่านี้ **ถูกต้องเสมอ** ไม่ว่าจะดูจาก timezone ไหน เพราะ PostgreSQL รู้ว่าจุดเวลาจริงคือเมื่อไหร่ (เก็บเป็น UTC ภายใน) แล้วแปลงให้ดูตาม session timezone โดยอัตโนมัติ

**กฎทองคำ:** ให้ใช้ `timestamptz` เป็นค่าเริ่มต้นสำหรับคอลัมน์เวลาแทบทุกกรณีในระบบจริง ยกเว้นกรณีพิเศษที่ต้องการเก็บ "wall-clock time" แบบไม่ผูกกับจุดเวลาจริง เช่น เวลานัดหมายที่ยังไม่รู้ timezone ที่แน่นอน (ดูรายละเอียดใน Step 69–70)

---

## Step 65: Time Zone Handling — SET TIME ZONE และ AT TIME ZONE

### การตั้งค่า time zone ของ session

```sql
SHOW TIME ZONE;
```

```
 TimeZone
------------
 Asia/Bangkok
(1 row)
```

```sql
SET TIME ZONE 'Asia/Bangkok';
SET TIME ZONE 'UTC';
SET TIME ZONE 'America/New_York';
SET TIME ZONE '+07';           -- offset แบบตายตัว (ไม่รองรับ DST)
SET TIME ZONE DEFAULT;         -- กลับไปใช้ค่าจาก postgresql.conf
```

> **แนะนำ:** ให้ใช้ชื่อ time zone แบบ IANA (เช่น `Asia/Bangkok`, `America/New_York`, `Europe/London`) แทนการใช้ offset ตัวเลข (`+07`) เพราะชื่อ IANA จะจัดการเรื่อง DST ให้อัตโนมัติ ในขณะที่ offset ตัวเลขเป็นค่าคงที่ตายตัวซึ่งจะผิดพลาดในช่วง DST

ดูรายชื่อ time zone ทั้งหมดที่ PostgreSQL รองรับ:

```sql
SELECT name, abbrev, utc_offset, is_dst
FROM pg_timezone_names
WHERE name LIKE 'Asia/%'
ORDER BY name
LIMIT 5;
```

```
      name       | abbrev | utc_offset | is_dst
------------------+--------+------------+--------
 Asia/Aden        | +03    | 03:00:00   | f
 Asia/Almaty      | +06    | 06:00:00   | f
 Asia/Amman       | +03    | 03:00:00   | f
 Asia/Anyang      | KST    | 09:00:00   | f
 Asia/Aqtau       | +05    | 05:00:00   | f
(5 rows)
```

### AT TIME ZONE operator

`AT TIME ZONE` เป็น operator ที่ใช้แปลงค่าเวลาไปมาระหว่าง `timestamp` และ `timestamptz` โดยมีพฤติกรรม 2 แบบขึ้นอยู่กับ type ของ input:

**1. แปลงจาก `timestamptz` เป็น `timestamp`** (ดูว่าเวลานั้นคือกี่โมงใน timezone ที่ระบุ)

```sql
SELECT '2026-09-25 07:30:00+00'::timestamptz AT TIME ZONE 'Asia/Bangkok' AS bangkok_local_time;
```

```
   bangkok_local_time
------------------------
 2026-09-25 14:30:00
(1 row)
```

ผลลัพธ์เป็น `timestamp` (ไม่มี tz) เพราะเป็นการ "แปลใจความ" ว่าเวลา UTC 07:30 คือกี่โมงที่กรุงเทพฯ (คำตอบ: 14:30)

**2. แปลงจาก `timestamp` เป็น `timestamptz`** (บอกว่าตัวเลขที่ป้อนมาคือเวลาใน timezone ที่ระบุ แล้วแปลงเป็นจุดเวลาจริง)

```sql
SELECT '2026-09-25 14:30:00'::timestamp AT TIME ZONE 'Asia/Bangkok' AS actual_instant;
```

```
       actual_instant
------------------------
 2026-09-25 07:30:00+00
(1 row)
```

ผลลัพธ์เป็น `timestamptz` เพราะเป็นการ "ระบุ" ว่า 14:30 (wall clock) ที่กรุงเทพฯ คือจุดเวลาจริงอะไร (คำตอบ: UTC 07:30)

### ตัวอย่างใช้งานจริง: แสดงเวลาให้ผู้ใช้แต่ละ timezone

```sql
CREATE TABLE user_session (
    user_id     INT,
    login_at    TIMESTAMPTZ NOT NULL,
    user_tz     TEXT NOT NULL
);

INSERT INTO user_session VALUES
    (1, '2026-09-25 07:30:00+00', 'Asia/Bangkok'),
    (2, '2026-09-25 07:30:00+00', 'America/New_York'),
    (3, '2026-09-25 07:30:00+00', 'Europe/London');

SELECT user_id,
       login_at,
       login_at AT TIME ZONE user_tz AS local_login_time
FROM user_session;
```

```
 user_id |        login_at        |  local_login_time
---------+-------------------------+---------------------
       1 | 2026-09-25 07:30:00+00 | 2026-09-25 14:30:00
       2 | 2026-09-25 07:30:00+00 | 2026-09-25 03:30:00
       3 | 2026-09-25 07:30:00+00 | 2026-09-25 08:30:00
(3 rows)
```

ทั้ง 3 แถวคือเวลาเดียวกันจริง (07:30 UTC) แต่แสดงผลตาม local time ของแต่ละผู้ใช้ได้อย่างถูกต้อง นี่คือรูปแบบการใช้งานที่แนะนำที่สุดสำหรับระบบที่มีผู้ใช้หลาย timezone

### สลับ chain แปลง 2 ครั้ง (แปลง timezone หนึ่งไปอีก timezone)

```sql
SELECT '2026-09-25 14:30:00'::timestamp
       AT TIME ZONE 'Asia/Bangkok'   -- ตีความว่าเป็นเวลากรุงเทพฯ -> ได้ timestamptz
       AT TIME ZONE 'America/New_York'  -- แปลงเป็น local time ของนิวยอร์ก -> ได้ timestamp
       AS bangkok_to_newyork;
```

```
 bangkok_to_newyork
----------------------
 2026-09-25 03:30:00
(1 row)
```

---

## Step 66: INTERVAL Type — การสร้างและคำนวณช่วงเวลา

### การสร้างค่า interval

```sql
SELECT INTERVAL '1 day',
       INTERVAL '2 hours 30 minutes',
       INTERVAL '1 year 2 months 3 days',
       INTERVAL '90 minutes',
       INTERVAL '1 week',
       INTERVAL '-3 days';
```

```
 interval | interval |   interval    | interval | interval | interval
----------+----------+---------------+----------+----------+----------
 1 day    | 02:30:00 | 1 year 2 mons | 01:30:00 | 7 days   | -3 days
 3 days
(1 rows)
```

รูปแบบย่อแบบ ISO 8601 ก็ใช้ได้เช่นกัน:

```sql
SELECT INTERVAL 'P1Y2M3D',        -- 1 ปี 2 เดือน 3 วัน
       INTERVAL 'P1DT2H30M';      -- 1 วัน 2 ชั่วโมง 30 นาที
```

```
   interval    |     interval
----------------+-------------------
 1 year 2 mons  | 1 day 02:30:00
 3 days
(1 row)
```

หรือสร้างด้วยฟังก์ชัน `make_interval()`:

```sql
SELECT make_interval(years => 1, months => 2, days => 3, hours => 4);
```

```
     make_interval
------------------------
 1 year 2 mons 3 days 04:00:00
(1 row)
```

### โครงสร้างภายในของ interval

`interval` เก็บข้อมูลเป็น 3 ส่วนแยกกัน: **months**, **days**, **microseconds** — ซึ่งเป็นการออกแบบที่ชาญฉลาดมาก เพราะ "1 เดือน" มีจำนวนวันไม่เท่ากันเสมอ (28-31 วัน) และ "1 วัน" อาจไม่เท่ากับ 24 ชั่วโมงเป๊ะเมื่อมี DST เข้ามาเกี่ยวข้อง การแยกเก็บทำให้การคำนวณแม่นยำตามบริบท

```sql
SELECT EXTRACT(MONTH FROM INTERVAL '1 year 2 months') AS months_part,
       justify_interval(INTERVAL '1 year 13 months') AS justified;
```

```
 months_part |     justified
--------------+--------------------
            2 | 2 years 1 mon
(1 row)
```

### การคำนวณ interval

```sql
SELECT INTERVAL '1 day' + INTERVAL '2 hours',
       INTERVAL '2 days' * 3,
       INTERVAL '1 hour' / 2,
       INTERVAL '3 days' - INTERVAL '1 day 5 hours';
```

```
    ?column?    | ?column? | ?column? |    ?column?
-----------------+----------+----------+-----------------
 1 day 02:00:00  | 6 days   | 00:30:00 | 1 day 19:00:00
(1 row)
```

### ตัวอย่างใช้งานจริง: คำนวณระยะเวลาโปรโมชั่น

```sql
CREATE TABLE promotion (
    id          SERIAL PRIMARY KEY,
    name        TEXT,
    starts_at   TIMESTAMPTZ NOT NULL,
    duration    INTERVAL NOT NULL
);

INSERT INTO promotion (name, starts_at, duration) VALUES
    ('Flash Sale 10.10', '2026-10-10 00:00:00+07', INTERVAL '24 hours'),
    ('Black Friday', '2026-11-27 00:00:00+07', INTERVAL '3 days');

SELECT name,
       starts_at,
       starts_at + duration AS ends_at,
       duration
FROM promotion;
```

```
        name        |        starts_at        |          ends_at          | duration
---------------------+--------------------------+----------------------------+----------
 Flash Sale 10.10    | 2026-10-10 00:00:00+07   | 2026-10-11 00:00:00+07    | 1 day
 Black Friday        | 2026-11-27 00:00:00+07   | 2026-11-30 00:00:00+07    | 3 days
(2 rows)
```

---

## Step 67: การคำนวณวันที่/เวลา — บวกลบ, AGE(), EXTRACT()

### บวกลบ date/timestamp กับ interval

```sql
SELECT CURRENT_DATE + INTERVAL '1 month' AS one_month_later,
       now() - INTERVAL '7 days' AS one_week_ago,
       '2026-01-31'::date + INTERVAL '1 month' AS jan31_plus_1mo;
```

```
     one_month_later     |         one_week_ago          | jan31_plus_1mo
--------------------------+--------------------------------+------------------------
 2026-10-25 00:00:00      | 2026-09-18 10:15:22.481203+07 | 2026-02-28 00:00:00
(1 row)
```

สังเกตกรณีพิเศษ: `2026-01-31 + 1 month` จะได้ `2026-02-28` (ไม่ใช่ `2026-03-03`) เพราะ PostgreSQL จะปรับให้เป็นวันสุดท้ายของเดือนกุมภาพันธ์ถ้าวันที่เกินขอบเขต — เป็นพฤติกรรมที่ควรทราบไว้เมื่อคำนวณข้ามเดือน

### การลบ timestamp สองค่า

```sql
SELECT '2026-12-25'::date - '2026-09-25'::date AS days_diff,
       '2026-09-25 18:00:00'::timestamptz - '2026-09-25 09:00:00'::timestamptz AS hours_diff;
```

```
 days_diff | hours_diff
-----------+------------
        91 | 09:00:00
(1 row)
```

`date - date` ได้ผลลัพธ์เป็น **integer** (จำนวนวัน) ในขณะที่ `timestamp - timestamp` ได้ผลลัพธ์เป็น **interval**

### AGE() function

`AGE()` คำนวณช่วงเวลาระหว่างสองจุดเวลาในรูปแบบที่อ่านง่าย (ปี-เดือน-วัน) ต่างจากการลบตรงๆ ที่ได้แค่จำนวนวัน/ชั่วโมงรวม

```sql
SELECT AGE(CURRENT_DATE, '1990-09-12'::date) AS full_age,
       AGE('2026-09-25'::date, '1990-09-12'::date) AS age_on_specific_date;
```

```
      full_age       | age_on_specific_date
----------------------+------------------------
 36 years 13 days     | 36 years 13 days
(1 row)
```

```sql
-- AGE() แบบ 1 argument จะเทียบกับ CURRENT_DATE โดยอัตโนมัติ
SELECT full_name, birth_date, AGE(birth_date) AS current_age
FROM customer;
```

```
   full_name    | birth_date |      current_age
-----------------+------------+-------------------------
 สมชาย ใจดี       | 1990-09-12 | 36 years 13 days
 สมหญิง รักเรียน   | 1985-03-04 | 41 years 6 mons 21 days
 วิชัย มั่นคง      | 1992-09-30 | 33 years 11 mons 25 days
(3 rows)
```

### EXTRACT() function

`EXTRACT()` ดึงส่วนย่อยจาก date/time/timestamp/interval ออกมาเป็นตัวเลข

```sql
SELECT EXTRACT(YEAR FROM now())        AS yr,
       EXTRACT(MONTH FROM now())       AS mo,
       EXTRACT(DAY FROM now())         AS dy,
       EXTRACT(HOUR FROM now())        AS hr,
       EXTRACT(DOW FROM now())         AS day_of_week,   -- 0=Sunday .. 6=Saturday
       EXTRACT(DOY FROM now())         AS day_of_year,
       EXTRACT(QUARTER FROM now())     AS quarter,
       EXTRACT(WEEK FROM now())        AS iso_week,
       EXTRACT(EPOCH FROM now())       AS unix_epoch;
```

```
 yr   | mo | dy | hr | day_of_week | day_of_year | quarter | iso_week |  unix_epoch
------+----+----+----+-------------+--------------+---------+----------+---------------
 2026 |  9 | 25 | 10 |           5 |          268 |       3 |       39 | 1790665234.29
(1 row)
```

### ฟังก์ชัน date_trunc() คู่หูของ EXTRACT

`date_trunc()` ตัดค่าเวลาให้เหลือความละเอียดที่ต้องการ มีประโยชน์มากในการทำรายงานสรุปรายวัน/รายเดือน

```sql
SELECT date_trunc('hour', now())  AS truncated_hour,
       date_trunc('day', now())   AS truncated_day,
       date_trunc('month', now()) AS truncated_month,
       date_trunc('year', now())  AS truncated_year;
```

```
      truncated_hour      |      truncated_day        |    truncated_month     |     truncated_year
---------------------------+----------------------------+--------------------------+---------------------------
 2026-09-25 10:00:00+07    | 2026-09-25 00:00:00+07    | 2026-09-01 00:00:00+07  | 2026-01-01 00:00:00+07
(1 row)
```

**ตัวอย่างใช้งานจริง:** สรุปยอดขายรายวัน

```sql
CREATE TABLE sales (
    id         SERIAL PRIMARY KEY,
    amount     NUMERIC(12,2),
    sold_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

INSERT INTO sales (amount, sold_at) VALUES
    (1500.00, '2026-09-25 09:15:00+07'),
    (2300.00, '2026-09-25 14:30:00+07'),
    (900.00,  '2026-09-26 10:00:00+07');

SELECT date_trunc('day', sold_at) AS sale_day,
       SUM(amount) AS total_sales,
       COUNT(*) AS num_orders
FROM sales
GROUP BY sale_day
ORDER BY sale_day;
```

```
        sale_day         | total_sales | num_orders
--------------------------+-------------+------------
 2026-09-25 00:00:00+07   |     3800.00 |          2
 2026-09-26 00:00:00+07   |      900.00 |          1
(2 rows)
```

---

## Step 68: Formatting และ Parsing ด้วย TO_CHAR / TO_DATE / TO_TIMESTAMP

### TO_CHAR — แปลง date/timestamp เป็น string ตาม format ที่ต้องการ

```sql
SELECT to_char(now(), 'DD/MM/YYYY')             AS thai_date,
       to_char(now(), 'YYYY-MM-DD')             AS iso_date,
       to_char(now(), 'DD Month YYYY')          AS full_month,
       to_char(now(), 'Day, DD Mon YYYY HH24:MI:SS') AS full_format,
       to_char(now(), 'HH12:MI AM')             AS time_12h;
```

```
 thai_date  |  iso_date  |      full_month       |          full_format          | time_12h
------------+------------+-----------------------+--------------------------------+-----------
 25/09/2026 | 2026-09-25 | 25 September     2026 | Friday   , 25 Sep 2026 10:15:22 | 10:15 AM
(1 row)
```

### ตาราง Format Patterns ที่ใช้บ่อยที่สุด

| Pattern | ความหมาย | ตัวอย่าง |
|---|---|---|
| `YYYY` | ปี 4 หลัก | `2026` |
| `YY` | ปี 2 หลัก | `26` |
| `MM` | เดือนเป็นตัวเลข 2 หลัก | `09` |
| `Month` | ชื่อเดือนเต็ม | `September` |
| `Mon` | ชื่อเดือนย่อ | `Sep` |
| `DD` | วันที่ 2 หลัก | `25` |
| `Day` | ชื่อวันเต็ม | `Friday` |
| `Dy` | ชื่อวันย่อ | `Fri` |
| `HH24` | ชั่วโมงแบบ 24 ชม. | `14` |
| `HH12` / `HH` | ชั่วโมงแบบ 12 ชม. | `02` |
| `MI` | นาที | `30` |
| `SS` | วินาที | `45` |
| `MS` | มิลลิวินาที | `123` |
| `AM`/`PM` | ช่วงเวลา | `PM` |
| `TZ` | ชื่อ timezone | `ICT` |
| `OF` | UTC offset | `+07` |
| `Q` | ไตรมาส | `3` |
| `WW` | สัปดาห์ที่ของปี | `39` |

### TO_DATE — แปลง string เป็น date

```sql
SELECT to_date('25/09/2026', 'DD/MM/YYYY'),
       to_date('September 25, 2026', 'Month DD, YYYY'),
       to_date('2026-09-25', 'YYYY-MM-DD');
```

```
  to_date   |  to_date   |  to_date
------------+------------+------------
 2026-09-25 | 2026-09-25 | 2026-09-25
(1 row)
```

### TO_TIMESTAMP — แปลง string เป็น timestamp

```sql
SELECT to_timestamp('25/09/2026 14:30:00', 'DD/MM/YYYY HH24:MI:SS'),
       to_timestamp('2026-09-25T14:30:00+07:00', 'YYYY-MM-DD"T"HH24:MI:SSOF');
```

```
        to_timestamp        |        to_timestamp
------------------------------+---------------------------
 2026-09-25 14:30:00+07       | 2026-09-25 14:30:00+07
(1 row)
```

> **หมายเหตุ:** `to_timestamp(text, format)` คืนค่าเป็น `timestamptz` เสมอ โดยตีความ input ตาม timezone ของ session ปัจจุบัน (เว้นแต่จะระบุ offset ไว้ใน string เอง)

### กรณีใช้งานจริง: import ข้อมูลจากไฟล์ CSV ที่มี format วันที่แบบไทย

```sql
CREATE TABLE import_staging (
    raw_date_text TEXT,
    raw_amount    TEXT
);

INSERT INTO import_staging VALUES
    ('25/09/2026', '1500.50'),
    ('01/12/2026', '2300.00');

SELECT to_date(raw_date_text, 'DD/MM/YYYY') AS parsed_date,
       raw_amount::numeric AS parsed_amount
FROM import_staging;
```

```
 parsed_date | parsed_amount
--------------+---------------
 2026-09-25   |       1500.50
 2026-12-01   |       2300.00
(2 rows)
```

### TO_CHAR สำหรับตัวเลขและ interval ด้วย (bonus)

```sql
SELECT to_char(INTERVAL '2 years 3 months 10 days', 'YYYY-MM-DD'),
       to_char(1234567.891, 'FM999,999,999.00') AS formatted_number;
```

```
   to_char   | formatted_number
---------------+-------------------
 0002-03-10   | 1,234,567.89
(1 row)
```

---

## Step 69: ปัญหาที่พบบ่อย — DST, Leap Year, และการเก็บวันเกิดข้าม Time Zone

### ปัญหาที่ 1: Daylight Saving Time (DST)

ในบาง timezone (เช่น สหรัฐฯ, ยุโรป) มีการปรับเวลาสองครั้งต่อปี ทำให้เกิดเวลาที่ "ไม่มีอยู่จริง" หรือ "ซ้ำกัน" ในบางวัน

```sql
-- ตัวอย่าง: วันเปลี่ยน DST ของ New York ปี 2026 (spring forward, 8 มี.ค. 2026 เวลา 2:00 AM -> 3:00 AM)
SET TIME ZONE 'America/New_York';

SELECT '2026-03-08 02:30:00'::timestamptz AS nonexistent_time;
```

```
      nonexistent_time
-------------------------
 2026-03-08 03:30:00-04
(1 row)
```

เวลา `02:30:00` ในวันนั้น **ไม่มีอยู่จริง** เพราะนาฬิกากระโดดจาก 02:00 ไป 03:00 ทันที PostgreSQL จะพยายามตีความให้สมเหตุสมผลที่สุด (เลื่อนไปข้างหน้า) แต่ผลลัพธ์อาจไม่ตรงกับที่คาดหวังเสมอไป

```sql
-- วันที่ปรับ DST กลับ (fall back, 1 พ.ย. 2026 เวลา 2:00 AM -> 1:00 AM) เวลาซ้ำกัน
SELECT '2026-11-01 01:30:00'::timestamptz AS ambiguous_time;
```

```
     ambiguous_time
--------------------------
 2026-11-01 01:30:00-04
(1 row)
```

เวลา `01:30:00` เกิดขึ้น **สองครั้ง** ในวันนั้น (ครั้งแรกตอนยังเป็น DST คือ -04, ครั้งที่สองหลังปรับกลับคือ -05) ซึ่ง PostgreSQL จะเลือกตีความแบบใดแบบหนึ่งเป็นค่าเริ่มต้น (โดยทั่วไปคือช่วงก่อนปรับ)

**บทเรียน:** อย่าใช้ `timestamp` (ไม่มี tz) เก็บเวลาที่จะเอาไปคำนวณเทียบกับ timezone ที่มี DST เพราะจะเกิดความกำกวมแบบนี้ทุกครั้งที่มีการปรับ DST หากต้องเก็บ "เวลานัดหมายซ้ำทุกสัปดาห์" ที่ยึดตาม wall-clock ควรเก็บเป็น `timestamp` + `timezone name` แยกกัน แล้วคำนวณ instant จริงตอนต้องใช้งาน ไม่ใช่เก็บ instant ไว้ล่วงหน้า

### ปัญหาที่ 2: Leap Year (ปีอธิกสุรทิน)

```sql
SELECT '2024-02-29'::date AS valid_leap_day,
       '2026-02-29'::date AS invalid_date;  -- จะเกิด error
```

```
ERROR:  date/time field value out of range: "2026-02-29"
LINE 2: SELECT '2026-02-29'::date AS invalid_date;
```

PostgreSQL จะ reject วันที่ 29 กุมภาพันธ์ในปีที่ไม่ใช่ปีอธิกสุรทินโดยอัตโนมัติ ซึ่งเป็นเรื่องดี แต่ปัญหาที่แท้จริงมักเกิดตอน**คำนวณ** ไม่ใช่ตอน insert:

```sql
-- คำนวณวันครบรอบปีของคนที่เกิดวันที่ 29 กุมภาพันธ์
SELECT '2024-02-29'::date + INTERVAL '1 year' AS next_anniversary;
```

```
   next_anniversary
------------------------
 2025-02-28 00:00:00
(1 row)
```

PostgreSQL จะปรับให้เป็น 28 กุมภาพันธ์โดยอัตโนมัติ (ไม่ error) ซึ่งเป็นพฤติกรรมที่สมเหตุสมผล แต่นักพัฒนาต้องรู้และออกแบบ business logic ให้รองรับกรณีนี้ (เช่น จะฉลองวันเกิดวันที่ 28 ก.พ. หรือ 1 มี.ค. ในปีที่ไม่ใช่ปีอธิกสุรทิน)

### ปัญหาที่ 3: การเก็บวันเกิดข้าม time zone

คำถามคลาสสิก: **วันเกิดควรเก็บเป็น `date` หรือ `timestamptz`?**

```sql
-- สมมติผู้ใช้เกิดวันที่ 25 กันยายน (ไม่ว่าจะอยู่ที่ไหนในโลก วันเกิดคือ 25 กันยายนเสมอ)
-- ถ้าเก็บเป็น timestamptz จะเกิดปัญหา:
CREATE TABLE user_bad_birthday (
    id INT,
    birthday TIMESTAMPTZ  -- ผิด! วันเกิดไม่ใช่ "จุดเวลา" แต่เป็น "วันที่ตามปฏิทิน"
);

SET TIME ZONE 'Asia/Bangkok';
INSERT INTO user_bad_birthday VALUES (1, '1990-09-25 00:00:00+07');

-- ผู้ใช้เปิดแอปจากอเมริกา (session timezone เป็น New York)
SET TIME ZONE 'America/New_York';
SELECT id, birthday, birthday::date AS shown_date
FROM user_bad_birthday;
```

```
 id |         birthday         | shown_date
----+---------------------------+------------
  1 | 1990-09-24 13:00:00-04   | 1990-09-24
(1 row)
```

**นี่คือบั๊กจริง** — วันเกิดที่ควรจะเป็น 25 กันยายน กลับกลายเป็น 24 กันยายน เมื่อดูจาก timezone อื่น! เพราะ `timestamptz` ถูกออกแบบมาสำหรับ "จุดเวลาที่แน่นอน" ไม่ใช่ "วันที่ตามปฏิทินที่ไม่ผูกกับเวลา" อย่างวันเกิด

**วิธีแก้ที่ถูกต้อง:** ใช้ `date` (ไม่มีเวลา ไม่มี timezone) สำหรับวันเกิด เพราะวันเกิดคือแนวคิดเชิงปฏิทิน ไม่ใช่จุดเวลาจริง

```sql
CREATE TABLE user_good_birthday (
    id INT,
    birthday DATE  -- ถูกต้อง! ไม่ผูกกับ timezone
);

INSERT INTO user_good_birthday VALUES (1, '1990-09-25');

SET TIME ZONE 'America/New_York';
SELECT * FROM user_good_birthday;  -- ค่าคงที่เสมอ ไม่ว่า session timezone จะเป็นอะไร
```

```
 id |  birthday
----+------------
  1 | 1990-09-25
(1 row)
```

**หลักการเลือก type ให้ถูกต้อง:**

| ข้อมูล | Type ที่ควรใช้ | เหตุผล |
|---|---|---|
| วันเกิด, วันครบรอบ, วันหมดอายุบัตร | `date` | เป็นแนวคิดปฏิทิน ไม่ผูกกับจุดเวลาจริง |
| เวลาที่ transaction เกิดขึ้นจริง (`created_at`) | `timestamptz` | เป็นจุดเวลาจริงบนโลก ต้องแม่นยำไม่ว่าดูจากไหน |
| เวลาเปิด-ปิดร้านทุกวัน | `time` + timezone name แยก | เป็น wall-clock ที่ต้องรู้ location |
| เวลานัดหมายประจำสัปดาห์ (เช่น ประชุมทุกวันจันทร์ 9 โมง) | `timestamp` + timezone name แยก | ต้องคงเวลาตาม wall-clock local แม้ DST เปลี่ยน |

---

## Step 70: Best Practices สำหรับแอปพลิเคชันจริง

### หลักการที่ 1: เก็บเวลาเป็น UTC เสมอในฐานข้อมูล

ใช้ `timestamptz` เป็นค่าเริ่มต้นสำหรับคอลัมน์เวลาแทบทุกกรณี และตั้งค่า server/session timezone เป็น `UTC` เพื่อความสม่ำเสมอ

```sql
-- ตั้งค่าระดับ database ให้ default timezone เป็น UTC (แนะนำสำหรับ production)
ALTER DATABASE mydb SET timezone TO 'UTC';

-- หรือตั้งใน postgresql.conf
-- timezone = 'UTC'
```

ตรวจสอบค่า timezone ของ database:

```sql
SHOW timezone;
SELECT current_setting('timezone');
```

```
 timezone
----------
 UTC
(1 row)
```

### หลักการที่ 2: แปลงเป็น local time เฉพาะตอนแสดงผล (application layer)

```sql
-- ฐานข้อมูลเก็บและส่งค่า UTC เสมอ
CREATE TABLE app_events (
    id         BIGSERIAL PRIMARY KEY,
    event_name TEXT,
    occurred_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

INSERT INTO app_events (event_name) VALUES ('user_signup');

-- Query จาก application จะได้ค่า UTC (หรือ ISO 8601 with offset)
SELECT id, event_name, occurred_at
FROM app_events;
```

```
 id | event_name  |         occurred_at
----+-------------+-------------------------------
  1 | user_signup | 2026-09-25 10:15:22.481203+00
(1 row)
```

จากนั้น **แอปพลิเคชัน (backend/frontend) เป็นผู้รับผิดชอบแปลงเป็น local time ของผู้ใช้** ตามโปรไฟล์ผู้ใช้หรือ browser timezone ไม่ใช่ให้ database ทำหน้าที่นี้ เพราะ:

- Database ไม่รู้ว่าผู้ใช้ที่กำลังดูอยู่ใน timezone ไหน (session timezone อาจไม่ตรงกับ browser ของผู้ใช้จริง)
- การแปลงที่ application layer (เช่นด้วย JavaScript `Intl.DateTimeFormat`, Python `zoneinfo`) ยืดหยุ่นกว่าและทดสอบง่ายกว่า
- Layer นี้สามารถ cache/reuse ได้ดีกว่าการยิง query ใหม่ทุกครั้งที่ timezone เปลี่ยน

### หลักการที่ 3: อย่าใช้ hardcoded offset ให้ใช้ชื่อ IANA timezone

```sql
-- ไม่แนะนำ: offset ตายตัว ไม่รองรับ DST
SELECT now() AT TIME ZONE '-05';

-- แนะนำ: ชื่อ IANA จัดการ DST ให้อัตโนมัติ
SELECT now() AT TIME ZONE 'America/New_York';
```

### หลักการที่ 4: ใช้ NOT NULL + DEFAULT now() สำหรับ audit columns

```sql
CREATE TABLE orders (
    id          BIGSERIAL PRIMARY KEY,
    customer_id INT NOT NULL,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ใช้ trigger เพื่ออัปเดต updated_at อัตโนมัติ
CREATE OR REPLACE FUNCTION set_updated_at()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = now();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_orders_updated_at
    BEFORE UPDATE ON orders
    FOR EACH ROW
    EXECUTE FUNCTION set_updated_at();
```

### หลักการที่ 5: เลือก type ให้ตรงกับความหมายทางธุรกิจ ไม่ใช่แค่ "เก็บเวลา"

ทบทวนตารางการเลือก type จาก Step 69 อีกครั้งเสมอเมื่อออกแบบ schema ใหม่ — คำถามสำคัญที่ต้องถามตัวเองคือ **"ข้อมูลนี้คือจุดเวลาที่แน่นอนบนโลก หรือเป็นแค่แนวคิดเชิงปฏิทิน/wall-clock ที่ไม่ผูกกับ timezone?"**

### หลักการที่ 6: ระวังการเปรียบเทียบ timestamp กับ timestamptz

```sql
-- การเปรียบเทียบข้าม type จะถูกแปลงให้อัตโนมัติ แต่ควรเลี่ยงการปนกันตั้งแต่ต้น
SELECT '2026-09-25 10:00:00'::timestamp = '2026-09-25 10:00:00+00'::timestamptz;
```

```
 ?column?
----------
 t
(1 row)
```

ผลลัพธ์อาจดู "ถูก" ในตัวอย่างนี้ (เพราะ session timezone เป็น UTC พอดี) แต่ถ้า session timezone เปลี่ยน ผลลัพธ์การเปรียบเทียบจะเปลี่ยนตามไปด้วย ซึ่งเป็นพฤติกรรมที่ไม่ควรพึ่งพา — ให้ใช้ type เดียวกันตลอดทั้งระบบ (`timestamptz`) เพื่อหลีกเลี่ยงปัญหานี้

### สรุปเช็คลิสต์ก่อน deploy ระบบจริง

- [ ] ตั้งค่า database/server timezone เป็น `UTC`
- [ ] ใช้ `timestamptz` สำหรับคอลัมน์เวลาที่เป็น "เหตุการณ์" (event) ทั้งหมด
- [ ] ใช้ `date` สำหรับข้อมูลเชิงปฏิทินที่ไม่ผูก timezone เช่น วันเกิด
- [ ] ไม่ใช้ `time with time zone`
- [ ] ใช้ชื่อ IANA timezone แทน numeric offset เสมอ
- [ ] แปลงเป็น local time ที่ application layer ไม่ใช่ที่ database
- [ ] ทดสอบ edge case DST และ leap year ในระบบที่ใช้การคำนวณวันที่ซับซ้อน
- [ ] ใช้ ISO 8601 format (`YYYY-MM-DD`) ในการป้อนและแลกเปลี่ยนข้อมูลวันที่เสมอ

---

## สรุปท้ายบท

ใน Part นี้เราได้เรียนรู้ data type สำหรับวันที่และเวลาของ PostgreSQL อย่างละเอียด ตั้งแต่ภาพรวมของ `date`, `time`, `timestamp`, `timestamptz`, `interval` ไปจนถึงกลไกการทำงานเชิงลึกของ time zone

**ประเด็นที่สำคัญที่สุดที่ต้องจำ:**

1. **`timestamptz` ไม่ได้เก็บ timezone ไว้จริงๆ** — มันเก็บเป็น UTC เสมอภายใน แล้วแปลงแสดงผลตาม session timezone โดยอัตโนมัติ ในขณะที่ `timestamp` เป็นแค่ตัวเลขตายตัวที่ไม่รู้จัก timezone เลย
2. **ให้ใช้ `timestamptz` เป็นค่าเริ่มต้นแทบทุกกรณี** สำหรับข้อมูลที่เป็น "จุดเวลาจริงบนโลก" เช่น `created_at`, `order_time`, log ต่างๆ
3. **ใช้ `date` สำหรับข้อมูลเชิงปฏิทิน** ที่ไม่ควรผูกกับ timezone เช่น วันเกิด วันครบกำหนดสัญญา
4. **หลีกเลี่ยง `time with time zone`** เพราะมีปัญหาเชิงแนวคิดที่ไม่สามารถแก้ไขได้ ให้แยกเก็บ timezone เป็นคอลัมน์ต่างหากแทน
5. **`AT TIME ZONE`** เป็นเครื่องมือหลักในการแปลงเวลาไปมาระหว่าง `timestamp` และ `timestamptz`
6. **`interval`** เก็บช่วงเวลาโดยแยกเป็น months/days/microseconds เพื่อรองรับความไม่แน่นอนของปฏิทิน (จำนวนวันในเดือนไม่เท่ากัน)
7. **DST และ leap year** เป็นแหล่งบั๊กที่พบบ่อย ต้องเข้าใจกลไกและทดสอบให้ครอบคลุม
8. **Best practice หลัก:** เก็บเป็น UTC ในฐานข้อมูลเสมอ แปลงเป็น local time เฉพาะตอนแสดงผลที่ application layer

ความเข้าใจเรื่อง data type วันที่/เวลาอย่างถ่องแท้เป็นพื้นฐานสำคัญที่จะช่วยป้องกันบั๊กร้ายแรงในระบบจริง โดยเฉพาะระบบที่ต้องรองรับผู้ใช้จากหลาย time zone ซึ่งเป็นเรื่องปกติมากขึ้นเรื่อยๆ ในโลกปัจจุบัน

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1
จงเขียนคำสั่ง SQL เพื่อสร้างตาราง `appointment` ที่มีคอลัมน์ `id`, `doctor_name`, `appointment_date` (เก็บเฉพาะวันที่นัด), `appointment_time` (เก็บเฉพาะเวลานัดในแต่ละวัน) โดยไม่ใช้ `timestamptz`

**เฉลย:**
```sql
CREATE TABLE appointment (
    id                SERIAL PRIMARY KEY,
    doctor_name       TEXT NOT NULL,
    appointment_date  DATE NOT NULL,
    appointment_time  TIME NOT NULL
);
```
เนื่องจากนัดหมายที่โรงพยาบาลผูกกับวันที่และเวลาตามปฏิทิน ไม่ใช่จุดเวลาสากล จึงแยก `date` และ `time` เป็นสองคอลัมน์แทนการใช้ `timestamptz` เพียงตัวเดียว

---

### แบบฝึกหัดที่ 2
จากตาราง `orders_good` ในบทเรียน ให้เขียน query แสดงเวลา `order_time` เป็นเวลาท้องถิ่นของ 3 timezone พร้อมกัน: `Asia/Bangkok`, `Asia/Tokyo`, `Europe/London`

**เฉลย:**
```sql
SELECT id,
       order_time,
       order_time AT TIME ZONE 'Asia/Bangkok' AS bangkok_time,
       order_time AT TIME ZONE 'Asia/Tokyo'   AS tokyo_time,
       order_time AT TIME ZONE 'Europe/London' AS london_time
FROM orders_good;
```

---

### แบบฝึกหัดที่ 3
จงอธิบายว่าทำไมโค้ดต่อไปนี้อาจทำให้เกิดข้อมูลผิดพลาดเมื่อระบบขยายไปให้บริการผู้ใช้ในหลายประเทศ:

```sql
CREATE TABLE login_history (
    user_id INT,
    login_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**เฉลย:**
`CURRENT_TIMESTAMP` คืนค่าเป็น `timestamptz` แต่เมื่อ assign ให้คอลัมน์ type `TIMESTAMP` (ไม่มี tz) PostgreSQL จะแปลงเป็นเวลาท้องถิ่นตาม session timezone ปัจจุบันแล้ว "ตัด" ข้อมูล timezone ทิ้งไป ผลคือค่าที่เก็บไว้จะไม่มีทางรู้ได้อีกว่าตอน insert นั้น session timezone คืออะไร ถ้าในอนาคต server เปลี่ยน timezone หรือผู้ใช้ login จากหลาย timezone ข้อมูลเวลาที่เก็บไว้จะตีความผิดพลาดทันที ควรใช้ `TIMESTAMPTZ` แทน

---

### แบบฝึกหัดที่ 4
จงเขียน query คำนวณอายุเป็นปี-เดือน-วัน ของลูกค้าทุกคนในตาราง `customer` โดยใช้ `AGE()`

**เฉลย:**
```sql
SELECT full_name, birth_date, AGE(CURRENT_DATE, birth_date) AS age
FROM customer;
```

---

### แบบฝึกหัดที่ 5
มีค่า `INTERVAL '25 months'` จงเขียน query แปลงให้อยู่ในรูป "ปี เดือน" ที่อ่านง่าย (เช่น 2 years 1 mon)

**เฉลย:**
```sql
SELECT justify_interval(INTERVAL '25 months') AS result;
```
```
    result
---------------
 2 years 1 mon
(1 row)
```

---

### แบบฝึกหัดที่ 6
จงเขียน query แปลง string `'25-09-2026 14:30'` (format วันที่แบบไทย DD-MM-YYYY HH24:MI) ให้เป็น `timestamptz` โดยตีความว่าเป็นเวลาที่ `Asia/Bangkok`

**เฉลย:**
```sql
SELECT to_timestamp('25-09-2026 14:30', 'DD-MM-YYYY HH24:MI') AT TIME ZONE 'Asia/Bangkok' AS result;
```
คำอธิบาย: `to_timestamp()` จะคืนค่าเป็น `timestamptz` โดยตีความตาม session timezone ก่อน จากนั้นต้องใช้ `AT TIME ZONE` เพื่อบอกว่าค่าตัวเลขที่ parse ได้ควรตีความเป็นเวลาของ `Asia/Bangkok` (วิธีที่แม่นยำกว่าคือ `SET TIME ZONE 'Asia/Bangkok'` ก่อนแล้วค่อย parse หรือรวม offset ไว้ใน string ตั้งแต่ต้น)

---

### แบบฝึกหัดที่ 7
จงอธิบายว่าเพราะเหตุใด PostgreSQL จึงไม่แนะนำให้ใช้ type `TIME WITH TIME ZONE` และควรใช้แนวทางใดแทน

**เฉลย:**
เพราะ `TIME WITH TIME ZONE` มีแค่เวลาและ offset โดยไม่มีวันที่กำกับ จึงไม่สามารถคำนวณ DST ได้อย่างถูกต้อง (เนื่องจาก DST ขึ้นอยู่กับวันที่) และการเปรียบเทียบค่าระหว่างสอง `timetz` ก็ไม่สอดคล้องกับเวลาจริงบนโลกเสมอไป แนวทางที่แนะนำคือใช้ `TIME` (without time zone) เก็บเวลาแบบ local เพียวๆ แล้วเก็บชื่อ timezone (เช่น `Asia/Bangkok`) ไว้ในคอลัมน์แยกต่างหาก

---

### แบบฝึกหัดที่ 8
กำหนดตาราง `subscription`:
```sql
CREATE TABLE subscription (
    id SERIAL PRIMARY KEY,
    plan_name TEXT,
    started_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    plan_duration INTERVAL NOT NULL
);
```
จงเขียน query แสดงรายการ subscription ที่ **หมดอายุแล้ว** ณ เวลาปัจจุบัน (เทียบ `started_at + plan_duration` กับ `now()`)

**เฉลย:**
```sql
SELECT id, plan_name, started_at, plan_duration,
       started_at + plan_duration AS expires_at
FROM subscription
WHERE started_at + plan_duration < now();
```

---

### แบบฝึกหัดที่ 9
จงอธิบายความแตกต่างของผลลัพธ์ระหว่างสอง query นี้ และบอกว่าเมื่อไหร่ควรใช้แบบไหน:

```sql
-- Query A
SELECT '2026-09-25'::date - '2026-01-01'::date;

-- Query B
SELECT AGE('2026-09-25'::date, '2026-01-01'::date);
```

**เฉลย:**
- Query A ให้ผลลัพธ์เป็น **integer** คือจำนวนวันทั้งหมดระหว่างสองวันที่ (267 วัน) เหมาะกับกรณีที่ต้องการนับจำนวนวันรวมล้วนๆ เช่น คำนวณค่าบริการรายวัน
- Query B ให้ผลลัพธ์เป็น **interval** ในรูปแบบปี-เดือน-วันที่อ่านง่าย (8 mons 24 days) เหมาะกับการแสดงผลให้มนุษย์อ่านเข้าใจง่าย เช่น "อายุสมาชิกภาพ 8 เดือน 24 วัน"

---

### แบบฝึกหัดที่ 10
บริษัทแห่งหนึ่งต้องการเก็บ "เวลาประชุมประจำสัปดาห์" ที่จัดทุกวันจันทร์เวลา 9:00 น. ตามเวลาท้องถิ่นของสำนักงานนิวยอร์ก (ซึ่งมี DST) จงอธิบายว่าทำไมการเก็บเป็น `timestamptz` ค่าเดียวไว้ล่วงหน้าจึงไม่เหมาะสม และควรออกแบบ schema อย่างไร

**เฉลย:**
ถ้าเก็บเป็น `timestamptz` แบบคำนวณ instant ล่วงหน้าไว้ครั้งเดียว (เช่นคำนวณ UTC ของ "จันทร์หน้า 9 โมงเช้านิวยอร์ก") ค่านั้นจะถูกต้องเฉพาะสัปดาห์นั้นเท่านั้น เพราะเมื่อ New York เปลี่ยนจาก EST เป็น EDT (หรือกลับกัน) ตาม DST เวลา 9:00 น. ตามนาฬิกาท้องถิ่นจะสอดคล้องกับ UTC offset ที่ต่างไปจากเดิม การ hardcode instant ไว้ล่วงหน้าจะทำให้เวลาประชุมคลาดเคลื่อนไปหนึ่งชั่วโมงในบางสัปดาห์

วิธีที่ถูกต้องคือเก็บเป็น "wall-clock time" บวกชื่อ timezone แยกกัน แล้วคำนวณ instant จริงตอนที่ต้องใช้งาน:
```sql
CREATE TABLE recurring_meeting (
    id            SERIAL PRIMARY KEY,
    meeting_name  TEXT,
    weekday       INT,        -- 1 = Monday
    local_time    TIME NOT NULL,       -- '09:00:00'
    tz_name       TEXT NOT NULL        -- 'America/New_York'
);

-- ตอนต้องการ instant จริงของสัปดาห์นี้ ค่อยคำนวณด้วย AT TIME ZONE
SELECT (CURRENT_DATE + local_time) AT TIME ZONE tz_name AS next_meeting_utc
FROM recurring_meeting
WHERE meeting_name = 'Weekly Standup';
```
วิธีนี้ทำให้ระบบคำนวณ offset ที่ถูกต้องตาม DST ของแต่ละสัปดาห์โดยอัตโนมัติ แทนที่จะ freeze ค่าผิดพลาดไว้ล่วงหน้า

---

## บทถัดไป

เรียนรู้การออกแบบตารางอย่างมืออาชีพต่อได้ที่ [Part 008: Table Design](./part-008-table-design.md)
