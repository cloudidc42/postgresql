# ตอนที่ 077: PostGIS ขั้นสูง — Spatial Index และ Geospatial Query

**หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับมืออาชีพ | Part 077**

> บทนี้ต่อเนื่องจาก **Part 076** ที่เราออกแบบสคีมาระบบจัดส่งสินค้าด้วย PostGIS (ตาราง `stores` และ `delivery_addresses` ที่เก็บพิกัดด้วยชนิดข้อมูล `GEOGRAPHY(POINT, 4326)`) ในบทนี้เราจะเจาะลึกเรื่อง **spatial index**, **KNN query**, **spatial join**, และฟังก์ชันวิเคราะห์เชิงพื้นที่ขั้นสูง เพื่อทำให้ระบบค้นหาตำแหน่งของเรา "เร็วระดับ production" ที่รองรับข้อมูลนับล้านแถวได้จริง

---

## เป้าหมายการเรียนรู้

เมื่อจบบทนี้ ผู้เรียนจะสามารถ:

1. เข้าใจโครงสร้างและหลักการทำงานของ **GiST Spatial Index** และสร้าง index บนคอลัมน์ `geography`/`geometry` ได้อย่างถูกต้อง
2. วัดผลกระทบของ spatial index ต่อ performance ด้วย `EXPLAIN ANALYZE` เปรียบเทียบก่อน-หลังสร้าง index บนข้อมูลจำนวนมาก
3. ใช้ **KNN operator (`<->`)** เพื่อหาจุดที่ใกล้ที่สุด N อันดับอย่างมีประสิทธิภาพ แทนการ `ORDER BY ST_Distance(...)` แบบธรรมดา
4. เขียน **Spatial JOIN** เพื่อเชื่อมโยงข้อมูลสองตารางด้วยเงื่อนไขเชิงพื้นที่ เช่น หาออเดอร์ที่อยู่ในแต่ละเขตจัดส่ง
5. ใช้ `ST_Buffer` สร้างพื้นที่ให้บริการรอบจุดหรือเส้นทาง
6. ใช้ `ST_Union` และ `ST_Intersection` เพื่อรวมหรือหาพื้นที่ทับซ้อนของ geometry หลายตัว
7. เข้าใจแนวคิดของ **Geocoding** และ **Reverse Geocoding** ในระดับภาพรวม
8. รู้จักความสามารถด้าน **Raster data** ของ PostGIS เบื้องต้น
9. ประยุกต์ทุกเทคนิคในบทนี้เพื่อสร้างระบบวิเคราะห์การจัดส่งแบบเต็มรูปแบบ

---

## เตรียมข้อมูล

เราจะสร้างสคีมาใหม่ที่ต่อยอดจาก Part 076 พร้อมข้อมูลตัวอย่างที่สมจริงมากขึ้น ครอบคลุมพื้นที่กรุงเทพมหานครและปริมณฑล เพื่อให้เห็นภาพการใช้งานเชิงพื้นที่ชัดเจน

### 1. เปิดใช้งาน Extension และสร้างตาราง

```sql
-- เปิดใช้งาน PostGIS extension (ทำครั้งเดียวต่อฐานข้อมูล)
CREATE EXTENSION IF NOT EXISTS postgis;

-- ตรวจสอบเวอร์ชันที่ติดตั้ง
SELECT PostGIS_Full_Version();

-- ตารางสาขาร้าน/คลังสินค้า
CREATE TABLE stores (
    store_id    SERIAL PRIMARY KEY,
    store_name  VARCHAR(150) NOT NULL,
    location    GEOGRAPHY(POINT, 4326)
);

-- ตารางที่อยู่จัดส่ง (แทนคำสั่งซื้อของลูกค้าแต่ละราย)
CREATE TABLE delivery_addresses (
    address_id    SERIAL PRIMARY KEY,
    customer_id   INTEGER,
    address_line  TEXT,
    location      GEOGRAPHY(POINT, 4326)
);

-- ตารางเขตจัดส่ง (ขอบเขตเป็นรูปหลายเหลี่ยม)
CREATE TABLE delivery_zones (
    zone_id     SERIAL PRIMARY KEY,
    zone_name   VARCHAR(100),
    boundary    GEOGRAPHY(POLYGON, 4326)
);
```

> **หมายเหตุ:** เราใช้ `GEOGRAPHY` แทน `GEOMETRY` เพราะข้อมูลของเราอิงพิกัดโลกจริง (ละติจูด/ลองจิจูด WGS84 — SRID 4326) และต้องการให้ PostGIS คำนวณระยะทาง/พื้นที่บนทรงกลม (spherical) ให้อัตโนมัติโดยไม่ต้อง reproject เอง ซึ่งเหมาะกับงาน delivery/logistics ที่ครอบคลุมพื้นที่กว้าง

### 2. ข้อมูลสาขาร้าน (stores) — ประมาณ 27 สาขาทั่วกรุงเทพฯ

เราจะแทรกพิกัดโดยใช้ `ST_SetSRID(ST_MakePoint(longitude, latitude), 4326)::geography` ซึ่งเป็นรูปแบบมาตรฐานที่ปลอดภัยที่สุดในการสร้างจุดพิกัด (ระบุ SRID อย่างชัดเจนก่อน cast เป็น geography)

```sql
INSERT INTO stores (store_name, location) VALUES
    ('สาขาสยามพารากอน',      ST_SetSRID(ST_MakePoint(100.5344, 13.7460), 4326)::geography),
    ('สาขาปทุมวัน',           ST_SetSRID(ST_MakePoint(100.5231, 13.7466), 4326)::geography),
    ('สาขาสีลม',              ST_SetSRID(ST_MakePoint(100.5289, 13.7248), 4326)::geography),
    ('สาขาบางรัก',            ST_SetSRID(ST_MakePoint(100.5177, 13.7239), 4326)::geography),
    ('สาขาคลองเตย',           ST_SetSRID(ST_MakePoint(100.5613, 13.7141), 4326)::geography),
    ('สาขาสุขุมวิท 21 (อโศก)', ST_SetSRID(ST_MakePoint(100.5601, 13.7373), 4326)::geography),
    ('สาขาพร้อมพงษ์',         ST_SetSRID(ST_MakePoint(100.5697, 13.7307), 4326)::geography),
    ('สาขาทองหล่อ',           ST_SetSRID(ST_MakePoint(100.5797, 13.7304), 4326)::geography),
    ('สาขาเอกมัย',            ST_SetSRID(ST_MakePoint(100.5853, 13.7196), 4326)::geography),
    ('สาขาอ่อนนุช',           ST_SetSRID(ST_MakePoint(100.6014, 13.7057), 4326)::geography),
    ('สาขาบางนา',             ST_SetSRID(ST_MakePoint(100.6019, 13.6684), 4326)::geography),
    ('สาขาพระราม 9',          ST_SetSRID(ST_MakePoint(100.5697, 13.7573), 4326)::geography),
    ('สาขาห้วยขวาง',          ST_SetSRID(ST_MakePoint(100.5769, 13.7757), 4326)::geography),
    ('สาขารัชดาภิเษก',        ST_SetSRID(ST_MakePoint(100.5731, 13.7649), 4326)::geography),
    ('สาขาพญาไท',             ST_SetSRID(ST_MakePoint(100.5347, 13.7654), 4326)::geography),
    ('สาขาอนุสาวรีย์ชัยฯ',     ST_SetSRID(ST_MakePoint(100.5372, 13.7649), 4326)::geography),
    ('สาขาอารีย์',             ST_SetSRID(ST_MakePoint(100.5460, 13.7797), 4326)::geography),
    ('สาขาจตุจักร',           ST_SetSRID(ST_MakePoint(100.5501, 13.7998), 4326)::geography),
    ('สาขาหมอชิต',            ST_SetSRID(ST_MakePoint(100.5537, 13.8021), 4326)::geography),
    ('สาขาลาดพร้าว',          ST_SetSRID(ST_MakePoint(100.5977, 13.7989), 4326)::geography),
    ('สาขาบางซื่อ',           ST_SetSRID(ST_MakePoint(100.5372, 13.8130), 4326)::geography),
    ('สาขานนทบุรี',           ST_SetSRID(ST_MakePoint(100.5145, 13.8622), 4326)::geography),
    ('สาขาดอนเมือง',          ST_SetSRID(ST_MakePoint(100.5967, 13.9126), 4326)::geography),
    ('สาขาบางกะปิ',           ST_SetSRID(ST_MakePoint(100.6469, 13.7658), 4326)::geography),
    ('สาขามีนบุรี',           ST_SetSRID(ST_MakePoint(100.7333, 13.8137), 4326)::geography),
    ('สาขาตลิ่งชัน',          ST_SetSRID(ST_MakePoint(100.4577, 13.7789), 4326)::geography),
    ('สาขาบางแค',             ST_SetSRID(ST_MakePoint(100.4067, 13.6963), 4326)::geography);
```

### 3. ข้อมูลที่อยู่จัดส่ง (delivery_addresses) — 18 รายการ

```sql
INSERT INTO delivery_addresses (customer_id, address_line, location) VALUES
    (1001, 'คอนโดใกล้สยามสแควร์ ถนนพระราม 1',            ST_SetSRID(ST_MakePoint(100.5360, 13.7440), 4326)::geography),
    (1002, 'อาคารสำนักงานถนนสีลม',                         ST_SetSRID(ST_MakePoint(100.5310, 13.7260), 4326)::geography),
    (1003, 'คอนโดสุขุมวิท 23 ใกล้ BTS อโศก',                ST_SetSRID(ST_MakePoint(100.5620, 13.7390), 4326)::geography),
    (1004, 'บ้านเดี่ยวซอยทองหล่อ 10',                       ST_SetSRID(ST_MakePoint(100.5810, 13.7320), 4326)::geography),
    (1005, 'คอนโดเอกมัย ถนนสุขุมวิท',                       ST_SetSRID(ST_MakePoint(100.5870, 13.7210), 4326)::geography),
    (1006, 'อาคารสำนักงานพร้อมพงษ์',                        ST_SetSRID(ST_MakePoint(100.5710, 13.7330), 4326)::geography),
    (1007, 'บ้านจัดสรรซอยอ่อนนุช 44',                       ST_SetSRID(ST_MakePoint(100.6030, 13.7070), 4326)::geography),
    (1008, 'คอนโดรัชดาภิเษก ใกล้ MRT',                       ST_SetSRID(ST_MakePoint(100.5750, 13.7660), 4326)::geography),
    (1009, 'บ้านจัดสรรลาดพร้าว 101',                        ST_SetSRID(ST_MakePoint(100.5990, 13.8000), 4326)::geography),
    (1010, 'คอนโดใกล้ตลาดนัดจตุจักร',                        ST_SetSRID(ST_MakePoint(100.5520, 13.8010), 4326)::geography),
    (1011, 'อาคารสำนักงานอารีย์',                            ST_SetSRID(ST_MakePoint(100.5480, 13.7810), 4326)::geography),
    (1012, 'คอนโดใกล้อนุสาวรีย์ชัยสมรภูมิ',                   ST_SetSRID(ST_MakePoint(100.5360, 13.7660), 4326)::geography),
    (1013, 'บ้านจัดสรรบางนา กม.5',                           ST_SetSRID(ST_MakePoint(100.6040, 13.6700), 4326)::geography),
    (1014, 'คอนโดบางกะปิ ใกล้ The Mall',                     ST_SetSRID(ST_MakePoint(100.6490, 13.7670), 4326)::geography),
    (1015, 'บ้านเดี่ยวมีนบุรี ถนนสีหบุรานุกิจ',                ST_SetSRID(ST_MakePoint(100.7350, 13.8150), 4326)::geography),
    (1016, 'บ้านจัดสรรตลิ่งชัน',                              ST_SetSRID(ST_MakePoint(100.4590, 13.7800), 4326)::geography),
    (1017, 'คอนโดบางแค ใกล้ Seacon Bangkae',                 ST_SetSRID(ST_MakePoint(100.4080, 13.6980), 4326)::geography),
    (1018, 'บ้านเดี่ยวนนทบุรี ถนนงามวงศ์วาน',                 ST_SetSRID(ST_MakePoint(100.5150, 13.8630), 4326)::geography);
```

### 4. ข้อมูลเขตจัดส่ง (delivery_zones) — 6 โซน

เราแบ่งพื้นที่กรุงเทพฯ-ปริมณฑลออกเป็น 6 โซนแบบสี่เหลี่ยมคร่าวๆ (ในงานจริงขอบเขตจะเป็นรูปหลายเหลี่ยมตามเขตการปกครอง แต่เพื่อความง่ายในการฝึกฝนเราใช้สี่เหลี่ยมผืนผ้าแทน) โดยใช้ `ST_GeogFromText` กับ WKT (Well-Known Text) รูปแบบ `POLYGON`

```sql
INSERT INTO delivery_zones (zone_name, boundary) VALUES
    ('โซนกรุงเทพชั้นใน (Inner City)',
     ST_GeogFromText('SRID=4326;POLYGON((
        100.5000 13.7000, 100.5650 13.7000,
        100.5650 13.7750, 100.5000 13.7750,
        100.5000 13.7000
     ))')),
    ('โซนสุขุมวิท-บางนา (Sukhumvit-Bangna)',
     ST_GeogFromText('SRID=4326;POLYGON((
        100.5600 13.6600, 100.6250 13.6600,
        100.6250 13.7450, 100.5600 13.7450,
        100.5600 13.6600
     ))')),
    ('โซนเหนือ (ลาดพร้าว-จตุจักร-หมอชิต)',
     ST_GeogFromText('SRID=4326;POLYGON((
        100.5300 13.7700, 100.6150 13.7700,
        100.6150 13.8250, 100.5300 13.8250,
        100.5300 13.7700
     ))')),
    ('โซนตะวันออก (บางกะปิ-มีนบุรี)',
     ST_GeogFromText('SRID=4326;POLYGON((
        100.6200 13.7550, 100.7500 13.7550,
        100.7500 13.8250, 100.6200 13.8250,
        100.6200 13.7550
     ))')),
    ('โซนตะวันตก (ตลิ่งชัน-บางแค)',
     ST_GeogFromText('SRID=4326;POLYGON((
        100.3900 13.6800, 100.4700 13.6800,
        100.4700 13.7950, 100.3900 13.7950,
        100.3900 13.6800
     ))')),
    ('โซนนนทบุรี-บางซื่อ',
     ST_GeogFromText('SRID=4326;POLYGON((
        100.4950 13.8050, 100.5650 13.8050,
        100.5650 13.8700, 100.4950 13.8700,
        100.4950 13.8050
     ))'));
```

ตรวจสอบข้อมูลที่แทรกเข้าไปทั้งหมดด้วยคำสั่งง่ายๆ:

```sql
SELECT count(*) AS total_stores FROM stores;             -- 27
SELECT count(*) AS total_addresses FROM delivery_addresses; -- 18
SELECT count(*) AS total_zones FROM delivery_zones;      -- 6

-- ดูพื้นที่ (ตร.กม.) ของแต่ละเขตจัดส่งอย่างรวดเร็ว
SELECT zone_name,
       round((ST_Area(boundary) / 1000000)::numeric, 2) AS area_sqkm
FROM delivery_zones
ORDER BY area_sqkm DESC;
```

ผลลัพธ์ตัวอย่าง:

```
             zone_name              | area_sqkm
-------------------------------------+-----------
 โซนตะวันออก (บางกะปิ-มีนบุรี)         |    998.87
 โซนตะวันตก (ตลิ่งชัน-บางแค)           |    928.44
 โซนเหนือ (ลาดพร้าว-จตุจักร-หมอชิต)     |    559.19
 โซนนนทบุรี-บางซื่อ                    |    484.94
 โซนสุขุมวิท-บางนา (Sukhumvit-Bangna)  |    562.15
 โซนกรุงเทพชั้นใน (Inner City)         |    534.90
```

ข้อมูลชุดนี้จะถูกใช้ต่อเนื่องตลอดทั้งบท เตรียมพร้อมแล้วมาเริ่มเจาะลึกเรื่อง spatial index กันเลย

---

## Step 761: GiST Spatial Index

ใน **Part 042** เราได้เรียนรู้เรื่อง index ประเภทต่างๆ ของ PostgreSQL ไปแล้ว รวมถึง **GiST (Generalized Search Tree)** ซึ่งเป็นโครงสร้าง index แบบ "ต้นไม้ทั่วไป" ที่ไม่ได้จำกัดอยู่แค่การเปรียบเทียบค่าเดี่ยว (`=`, `<`, `>` แบบ B-Tree) แต่รองรับการค้นหาข้อมูลที่มี "รูปทรง" หรือ "ขอบเขต" ได้ เช่น ช่วงเวลา (range types), ข้อความเต็ม (full-text search), และที่สำคัญที่สุดสำหรับบทนี้คือ **ข้อมูลเชิงพื้นที่ (spatial data)**

### GiST ทำงานอย่างไรกับข้อมูลเชิงพื้นที่

หัวใจของ GiST spatial index คือแนวคิด **bounding box (กรอบสี่เหลี่ยมล้อมรอบ)**:

1. ทุก geometry/geography (จุด, เส้น, รูปหลายเหลี่ยม) จะถูกคำนวณกรอบสี่เหลี่ยมที่เล็กที่สุดที่ล้อมรอบตัวมันได้ (เรียกว่า **MBR — Minimum Bounding Rectangle**)
2. GiST จัดกลุ่ม MBR เหล่านี้เป็นโครงสร้างต้นไม้แบบลำดับชั้น (คล้าย R-Tree) โดยแต่ละ "โหนดแม่" จะมี MBR ที่ครอบคลุม MBR ของ "โหนดลูก" ทั้งหมด
3. เมื่อค้นหา (เช่น "หาจุดที่อยู่ในระยะ 5 กม.") PostgreSQL จะไล่ตรวจ MBR จากบนลงล่าง — ถ้ากรอบสี่เหลี่ยมของโหนดใดไม่ตัดกับพื้นที่ค้นหาเลย ก็ตัดทิ้งทั้งกิ่งได้ทันทีโดยไม่ต้องตรวจ geometry จริงข้างใน ทำให้ค้นหาได้เร็วมากแม้ข้อมูลจะมีหลายล้านแถว

```
                     [Root: MBR ครอบคลุมทั้งหมด]
                    /              |              \
          [MBR โซนเหนือ]    [MBR โซนกลาง]    [MBR โซนตะวันออก]
           /        \           /      \            \
       [จุด A]    [จุด B]   [จุด C]  [จุด D]      [จุด E]
```

### สร้าง Spatial Index

```sql
-- สร้าง GiST index บนคอลัมน์ geography ทั้งสามตาราง
CREATE INDEX idx_stores_location
    ON stores USING GIST (location);

CREATE INDEX idx_delivery_addresses_location
    ON delivery_addresses USING GIST (location);

CREATE INDEX idx_delivery_zones_boundary
    ON delivery_zones USING GIST (boundary);
```

สังเกตว่าไวยากรณ์เหมือนกับการสร้าง GiST index ทั่วไปตามที่เรียนใน Part 042 ทุกประการ เพียงแค่ระบุ `USING GIST (คอลัมน์เชิงพื้นที่)` — PostgreSQL/PostGIS จะเลือก **operator class** ที่เหมาะสมให้อัตโนมัติ (`gist_geography_ops` สำหรับ `geography`, `gist_geometry_ops_2d` สำหรับ `geometry`)

### ตรวจสอบว่า index ถูกสร้างและใช้งานได้จริง

```sql
-- ดู index ทั้งหมดของตาราง
SELECT indexname, indexdef
FROM pg_indexes
WHERE tablename IN ('stores', 'delivery_addresses', 'delivery_zones');

-- ดูขนาด index
SELECT indexrelid::regclass AS index_name,
       pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
WHERE relname = 'stores';
```

### ทำไมต้องระบุ operator class ให้ถูกต้อง

ถ้าตารางมีทั้งคอลัมน์ `geometry` และ `geography` ปนกัน หรือมีการใช้ index สำหรับ 3D geometry (`GEOMETRY(POINTZ, 4326)`) จำเป็นต้องระบุ opclass ให้ตรง เช่น

```sql
-- ตัวอย่าง: ถ้าต้องการ index สำหรับ geometry แบบ 3 มิติ
-- CREATE INDEX idx_example_3d ON some_table USING GIST (geom_col gist_geometry_ops_nd);
```

แต่สำหรับกรณีของเรา (2D geography) การเขียน `USING GIST (location)` แบบสั้นๆ ก็เพียงพอแล้ว เพราะ PostgreSQL จะเลือก opclass เริ่มต้นที่ถูกต้องให้เอง

> **ข้อควรจำ:** Spatial index (GiST) ไม่ได้ทำให้ผลลัพธ์ของ `ST_Distance`, `ST_Intersects`, `ST_DWithin` ฯลฯ เปลี่ยนไป — มันแค่ช่วยให้ PostgreSQL **ข้ามแถวที่ไม่เกี่ยวข้องได้เร็วขึ้น** โดยใช้ bounding box กรองข้อมูลก่อน แล้วค่อยตรวจสอบ geometry จริงในขั้นตอน "recheck" เท่านั้น

---

## Step 762: ผลของ Spatial Index ต่อ Performance

ในหัวข้อนี้เราจะพิสูจน์ด้วยตัวเลขจริงว่า spatial index ช่วยอะไรได้บ้าง โดยการสร้างข้อมูลจำลองจำนวนมาก แล้วเปรียบเทียบ `EXPLAIN ANALYZE` ก่อนและหลังมี index

### 1. สร้างข้อมูลจำลองขนาดใหญ่ (200,000 แถว)

เราจะสุ่มพิกัดภายในกรอบสี่เหลี่ยมที่ครอบคลุมกรุงเทพฯ-ปริมณฑล (ลองจิจูด 100.30–100.85, ละติจูด 13.60–13.95)

```sql
INSERT INTO delivery_addresses (customer_id, address_line, location)
SELECT
    2000 + gs AS customer_id,
    'ที่อยู่ทดสอบ #' || gs,
    ST_SetSRID(
        ST_MakePoint(
            100.30 + random() * (100.85 - 100.30),
            13.60  + random() * (13.95  - 13.60)
        ),
        4326
    )::geography
FROM generate_series(1, 200000) AS gs;

-- ตรวจสอบจำนวนแถวทั้งหมด
SELECT count(*) FROM delivery_addresses;   -- ควรได้ 200018 แถว (18 เดิม + 200000 ใหม่)
```

### 2. ทดสอบก่อนมี Index (ลบ index ออกชั่วคราว)

```sql
DROP INDEX IF EXISTS idx_delivery_addresses_location;

EXPLAIN ANALYZE
SELECT address_id, address_line
FROM delivery_addresses
WHERE ST_DWithin(
    location,
    ST_SetSRID(ST_MakePoint(100.5344, 13.7460), 4326)::geography,  -- จุดอ้างอิง: สยามพารากอน
    3000  -- รัศมี 3,000 เมตร
);
```

ผลลัพธ์ตัวอย่าง (ไม่มี index — ต้องสแกนทุกแถว):

```
                                              QUERY PLAN
--------------------------------------------------------------------------------------------------
 Seq Scan on delivery_addresses  (cost=0.00..48521.30 rows=1204 width=42)
                                  (actual time=0.152..312.847 rows=1187 loops=1)
   Filter: st_dwithin(location, '0101000020E6100000...'::geography, '3000'::double precision)
   Rows Removed by Filter: 198831
 Planning Time: 0.412 ms
 Execution Time: 313.021 ms
```

จะเห็นว่า PostgreSQL ต้องอ่านทุกแถว (Seq Scan) แล้วคำนวณระยะทางทีละแถวเพื่อกรอง ทำให้ต้องตัดทิ้งเกือบ 200,000 แถว ใช้เวลากว่า **313 มิลลิวินาที**

### 3. สร้าง Index กลับมา แล้วทดสอบซ้ำ

```sql
CREATE INDEX idx_delivery_addresses_location
    ON delivery_addresses USING GIST (location);

-- อัปเดตสถิติให้ query planner ตัดสินใจได้แม่นยำ
ANALYZE delivery_addresses;

EXPLAIN ANALYZE
SELECT address_id, address_line
FROM delivery_addresses
WHERE ST_DWithin(
    location,
    ST_SetSRID(ST_MakePoint(100.5344, 13.7460), 4326)::geography,
    3000
);
```

ผลลัพธ์ตัวอย่าง (มี index):

```
                                              QUERY PLAN
--------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on delivery_addresses  (cost=64.21..1820.45 rows=1204 width=42)
                                          (actual time=1.203..4.812 rows=1187 loops=1)
   Recheck Cond: st_dwithin(location, '0101000020E6100000...'::geography, '3000'::double precision)
   Heap Blocks: exact=1142
   ->  Bitmap Index Scan on idx_delivery_addresses_location
         (cost=0.00..63.91 rows=1204 width=0)
         (actual time=0.891..0.891 rows=1187 loops=1)
         Index Cond: st_dwithin(location, '0101000020E6100000...'::geography, '3000'::double precision)
 Planning Time: 0.298 ms
 Execution Time: 5.014 ms
```

### สรุปการเปรียบเทียบ

| สถานการณ์ | Execution Time | Scan Type |
|---|---|---|
| ไม่มี Spatial Index | ~313 ms | Seq Scan (อ่านทุกแถว) |
| มี Spatial Index | ~5 ms | Bitmap Index Scan → Bitmap Heap Scan |

**เร็วขึ้นประมาณ 60 เท่า** สำหรับข้อมูล 200,000 แถว และยิ่งข้อมูลมากขึ้น (ระดับล้านแถว) ส่วนต่างของ performance จะยิ่งห่างขึ้นเรื่อยๆ เพราะ Seq Scan มี complexity แบบ O(n) ในขณะที่ GiST index scan อยู่ที่ประมาณ O(log n)

> **ข้อสังเกตสำคัญ:** planner เลือกใช้ `Bitmap Index Scan` แทน `Index Scan` ตรงๆ เพราะจำนวนผลลัพธ์ที่คาดว่าจะได้ (rows=1204) ค่อนข้างมากเมื่อเทียบกับข้อมูลทั้งหมด การใช้ bitmap ช่วยลดการเข้าถึง heap แบบสุ่ม (random I/O) — ถ้าค้นหาแบบแคบมากๆ (เช่น รัศมี 100 เมตร) planner อาจเลือก `Index Scan` ตรงแทน

### อย่าลืม: ทำความสะอาดข้อมูลทดสอบ (ถ้าต้องการ)

```sql
-- ถ้าต้องการล้างข้อมูลทดสอบ 200,000 แถวออกก่อนไปหัวข้อถัดไป (ไม่บังคับ)
DELETE FROM delivery_addresses WHERE customer_id >= 2001;
VACUUM ANALYZE delivery_addresses;
```

> ในบทนี้เราจะ**เก็บข้อมูลทดสอบไว้**เพื่อใช้สาธิตหัวข้อถัดไปด้วย เพราะการมีข้อมูลจำนวนมากจะทำให้เห็นความแตกต่างของ performance ชัดเจนขึ้น

---

## Step 763: K-Nearest Neighbor (KNN) Query ด้วย `<->`

หนึ่งใน query ที่พบบ่อยที่สุดในระบบ logistics คือ **"หาสาขาที่ใกล้ที่สุด N อันดับจากจุดที่กำหนด"** ซึ่งดูเหมือนจะเขียนง่ายๆ ด้วย `ORDER BY ST_Distance(...) LIMIT N` แต่วิธีนี้มีปัญหาด้าน performance ที่ซ่อนอยู่

### ปัญหาของ `ORDER BY ST_Distance(...) LIMIT N`

```sql
EXPLAIN ANALYZE
SELECT store_id, store_name,
       ST_Distance(location, ST_SetSRID(ST_MakePoint(100.5601, 13.7373), 4326)::geography) AS distance_m
FROM stores
ORDER BY ST_Distance(location, ST_SetSRID(ST_MakePoint(100.5601, 13.7373), 4326)::geography)
LIMIT 5;
```

```
                                       QUERY PLAN
-----------------------------------------------------------------------------------------
 Limit  (cost=25.14..25.15 rows=5 width=46) (actual time=0.198..0.201 rows=5 loops=1)
   ->  Sort  (cost=25.14..25.21 rows=27 width=46) (actual time=0.196..0.197 rows=5 loops=1)
         Sort Key: (st_distance(location, '...'::geography))
         Sort Method: top-N heapsort  Memory: 25kB
         ->  Seq Scan on stores  (cost=0.00..24.85 rows=27 width=46)
                                  (actual time=0.031..0.156 rows=27 loops=1)
 Planning Time: 0.089 ms
 Execution Time: 0.221 ms
```

จะเห็นว่า PostgreSQL ต้อง**คำนวณระยะทางของทุกแถวก่อน แล้วค่อยเรียงลำดับ** — สำหรับตาราง `stores` ที่มีแค่ 27 แถวไม่มีปัญหาอะไร แต่ถ้าเป็นตาราง `delivery_addresses` ที่มี 200,000 แถว การคำนวณระยะทางทุกแถวจะกลายเป็นคอขวดทันที (เป็น Seq Scan + Sort เสมอ ไม่ว่าจะมี spatial index หรือไม่ เพราะ `ST_Distance` ใน `ORDER BY` แบบธรรมดาไม่สามารถใช้ index ช่วยเรียงลำดับได้)

### ทางออก: KNN Operator `<->`

PostGIS มี **distance operator `<->`** ที่ออกแบบมาเฉพาะสำหรับงาน KNN โดยเฉพาะ — เมื่อใช้คู่กับ `ORDER BY` และมี GiST index อยู่ ตัว operator นี้จะสามารถ**ใช้ index เพื่อเรียงลำดับตามระยะทางได้โดยตรง** (index-assisted nearest-neighbor search) โดยไม่ต้องคำนวณระยะทางของทุกแถว

```sql
-- หาสาขาที่ใกล้ที่สุด 5 อันดับจากจุดอโศก โดยใช้ KNN operator
EXPLAIN ANALYZE
SELECT store_id, store_name,
       ROUND((location <-> ST_SetSRID(ST_MakePoint(100.5601, 13.7373), 4326)::geography)::numeric, 1) AS distance_m
FROM stores
ORDER BY location <-> ST_SetSRID(ST_MakePoint(100.5601, 13.7373), 4326)::geography
LIMIT 5;
```

```
                                          QUERY PLAN
------------------------------------------------------------------------------------------------
 Limit  (cost=0.14..2.31 rows=5 width=46) (actual time=0.045..0.089 rows=5 loops=1)
   ->  Index Scan using idx_stores_location on stores
         (cost=0.14..11.87 rows=27 width=46) (actual time=0.043..0.086 rows=5 loops=1)
         Order By: (location <-> '0101000020E6100000...'::geography)
 Planning Time: 0.076 ms
 Execution Time: 0.102 ms
```

สังเกต plan node สำคัญ: **`Index Scan ... Order By: (location <-> ...)`** — นี่คือสัญญาณว่า PostgreSQL กำลังไล่ดึงจุดที่ใกล้ที่สุดออกจาก GiST index **ทีละจุดตามลำดับระยะทาง** โดยไม่ต้องสแกนทั้งตาราง เมื่อได้ครบ `LIMIT 5` ก็หยุดทันที — นี่คือเหตุผลที่เรียกว่า **KNN-GiST** (K-Nearest Neighbor ผ่านโครงสร้าง GiST)

### เปรียบเทียบกับตารางข้อมูลขนาดใหญ่ (200,000 แถว)

```sql
-- หาที่อยู่จัดส่ง 10 รายการที่ใกล้ "สาขาลาดพร้าว" มากที่สุด
EXPLAIN ANALYZE
SELECT address_id, address_line,
       ROUND((location <-> (SELECT location FROM stores WHERE store_name = 'สาขาลาดพร้าว'))::numeric, 1) AS distance_m
FROM delivery_addresses
ORDER BY location <-> (SELECT location FROM stores WHERE store_name = 'สาขาลาดพร้าว')
LIMIT 10;
```

```
                                             QUERY PLAN
------------------------------------------------------------------------------------------------------
 Limit  (cost=8.45..14.92 rows=10 width=54) (actual time=0.312..0.891 rows=10 loops=1)
   InitPlan 1 (returns $0)
     ->  Index Scan using idx_stores_location on stores  (cost=0.14..2.31 rows=1 width=32)
   ->  Index Scan using idx_delivery_addresses_location on delivery_addresses
         (cost=0.14..129041.20 rows=200018 width=54) (actual time=0.310..0.887 rows=10 loops=1)
         Order By: (location <-> $0)
 Planning Time: 0.201 ms
 Execution Time: 0.921 ms
```

ใช้เวลาไม่ถึง 1 มิลลิวินาทีแม้ข้อมูลจะมี 200,000 แถว! เทียบกับการใช้ `ORDER BY ST_Distance(...) LIMIT 10` แบบธรรมดาที่จะต้อง Sort ทั้งตาราง ซึ่งจะใช้เวลานับร้อยมิลลิวินาทีขึ้นไป

### กฎการใช้งาน KNN operator

1. ต้องมี **GiST index** อยู่บนคอลัมน์ที่ใช้ (`gist_geography_ops` หรือ `gist_geometry_ops_2d`) — ถ้าไม่มี index, `<->` ยังคงทำงานได้ (คำนวณระยะทางถูกต้อง) แต่จะไม่ได้ประโยชน์ด้าน performance
2. ใช้คู่กับ `ORDER BY ... LIMIT N` เสมอ — ถ้าใช้ใน `WHERE` clause โดยไม่มี `ORDER BY`/`LIMIT` มันจะไม่ทำ KNN scan
3. รองรับทั้ง `geometry` และ `geography` (ตั้งแต่ PostGIS 2.2 ขึ้นไป) — เวอร์ชันที่เราใช้ใน PostgreSQL 16/17 รองรับแน่นอน
4. ค่าที่ได้จาก `<->` คือ**ระยะทางโดยประมาณ** (approximate) ที่ใช้สำหรับการเรียงลำดับเป็นหลัก หากต้องการค่าระยะทางที่แม่นยำ 100% ให้คำนวณด้วย `ST_Distance` แยกใน `SELECT` list (ดังตัวอย่างข้างต้นที่เราทำอยู่แล้ว)

### ตัวอย่างใช้งานจริง: หาสาขาที่ใกล้ที่สุดสำหรับลูกค้าแต่ละราย

```sql
SELECT
    da.address_id,
    da.address_line,
    s.store_name AS nearest_store,
    ROUND(ST_Distance(da.location, s.location)::numeric, 0) AS distance_m
FROM delivery_addresses da
CROSS JOIN LATERAL (
    SELECT store_name, location
    FROM stores
    ORDER BY stores.location <-> da.location
    LIMIT 1
) s
WHERE da.address_id <= 18   -- เฉพาะข้อมูลลูกค้าจริง 18 รายการแรก (ไม่รวมข้อมูลทดสอบ)
ORDER BY da.address_id;
```

ที่นี่เราใช้ `LATERAL JOIN` ร่วมกับ KNN operator เพื่อหา **"สาขาที่ใกล้ที่สุด 1 แห่ง" สำหรับแต่ละที่อยู่จัดส่ง** — เป็นรูปแบบ query ที่พบบ่อยมากในระบบ dispatch/routing และมีประสิทธิภาพสูงเพราะแต่ละแถวใช้ KNN-GiST index scan แยกกัน

---

## Step 764: Spatial JOIN

**Spatial JOIN** คือการ JOIN สองตารางโดยใช้เงื่อนไขเชิงพื้นที่แทนการเทียบค่า key ตรงๆ เช่น "จุดนี้อยู่ในรูปหลายเหลี่ยมนั้นหรือไม่" แทนที่จะ JOIN ด้วย `store_id = store_id`

ในระบบของเรา โจทย์คือ: **"ที่อยู่จัดส่งแต่ละรายการ (สมมติว่าคือคำสั่งซื้อ 1 ออเดอร์) อยู่ในเขตจัดส่ง (delivery_zone) ใด?"**

### ฟังก์ชันสำหรับ Spatial JOIN บน geography

ชนิดข้อมูล `geography` รองรับฟังก์ชันความสัมพันธ์เชิงพื้นที่โดยตรงหลายตัว เช่น `ST_Intersects`, `ST_Covers`, `ST_CoveredBy`, `ST_DWithin` แต่**ไม่รองรับ** `ST_Within`/`ST_Contains` โดยตรง (ฟังก์ชันกลุ่มนี้ถูกออกแบบมาสำหรับ `geometry` แบบระนาบ) ดังนั้นสำหรับกรณี "จุดอยู่ในรูปหลายเหลี่ยมหรือไม่" เราใช้ `ST_Covers` แทน (ความหมายใกล้เคียงกับ `ST_Contains` แต่รวมกรณีที่จุดอยู่บนขอบพอดีด้วย) ซึ่งทำงานได้ถูกต้องบน `geography`

```sql
-- หาว่าที่อยู่จัดส่งแต่ละแห่งอยู่ในเขตใด
SELECT
    da.address_id,
    da.address_line,
    dz.zone_name
FROM delivery_addresses da
JOIN delivery_zones dz
    ON ST_Covers(dz.boundary, da.location)
WHERE da.address_id <= 18
ORDER BY da.address_id;
```

ผลลัพธ์ตัวอย่าง:

```
 address_id |              address_line               |              zone_name
------------+------------------------------------------+---------------------------------------
          1 | คอนโดใกล้สยามสแควร์ ถนนพระราม 1           | โซนกรุงเทพชั้นใน (Inner City)
          2 | อาคารสำนักงานถนนสีลม                      | โซนกรุงเทพชั้นใน (Inner City)
          3 | คอนโดสุขุมวิท 23 ใกล้ BTS อโศก             | โซนกรุงเทพชั้นใน (Inner City)
          4 | บ้านเดี่ยวซอยทองหล่อ 10                    | โซนสุขุมวิท-บางนา (Sukhumvit-Bangna)
          5 | คอนโดเอกมัย ถนนสุขุมวิท                    | โซนสุขุมวิท-บางนา (Sukhumvit-Bangna)
          8 | คอนโดรัชดาภิเษก ใกล้ MRT                    | โซนกรุงเทพชั้นใน (Inner City)
          9 | บ้านจัดสรรลาดพร้าว 101                     | โซนเหนือ (ลาดพร้าว-จตุจักร-หมอชิต)
         13 | บ้านจัดสรรบางนา กม.5                       | โซนสุขุมวิท-บางนา (Sukhumvit-Bangna)
         15 | บ้านเดี่ยวมีนบุรี ถนนสีหบุรานุกิจ             | โซนตะวันออก (บางกะปิ-มีนบุรี)
         16 | บ้านจัดสรรตลิ่งชัน                          | โซนตะวันตก (ตลิ่งชัน-บางแค)
         18 | บ้านเดี่ยวนนทบุรี ถนนงามวงศ์วาน               | โซนนนทบุรี-บางซื่อ
```

> สังเกตว่าบางที่อยู่อาจไม่ปรากฏในผลลัพธ์ (เช่น address_id 6, 7, 10-12, 14, 17) เพราะพิกัดของมันตกอยู่นอกขอบเขตของทั้ง 6 โซนที่เรากำหนดไว้ (เขตของเราเป็นเพียงสี่เหลี่ยมคร่าวๆ ไม่ครอบคลุมทั้งเมือง) — ในงานจริงควรใช้ `LEFT JOIN` เพื่อดูออเดอร์ที่ "ตกหล่น" ไม่มีเขตรองรับ

### ใช้ LEFT JOIN เพื่อหาออเดอร์ที่ไม่มีเขตจัดส่งรองรับ

```sql
SELECT
    da.address_id,
    da.address_line,
    COALESCE(dz.zone_name, '*** ไม่มีเขตจัดส่งรองรับ ***') AS zone_name
FROM delivery_addresses da
LEFT JOIN delivery_zones dz
    ON ST_Covers(dz.boundary, da.location)
WHERE da.address_id <= 18
ORDER BY da.address_id;
```

### สรุปยอดออเดอร์ต่อเขต (GROUP BY บน Spatial JOIN)

```sql
SELECT
    dz.zone_name,
    count(da.address_id) AS total_orders
FROM delivery_zones dz
LEFT JOIN delivery_addresses da
    ON ST_Covers(dz.boundary, da.location)
    AND da.address_id <= 18
GROUP BY dz.zone_name
ORDER BY total_orders DESC;
```

### ตรวจสอบ Execution Plan ของ Spatial JOIN

```sql
EXPLAIN ANALYZE
SELECT da.address_id, dz.zone_name
FROM delivery_addresses da
JOIN delivery_zones dz
    ON ST_Covers(dz.boundary, da.location);
```

```
                                          QUERY PLAN
-------------------------------------------------------------------------------------------------
 Nested Loop  (cost=0.14..8945.21 rows=33336 width=12) (actual time=0.089..612.442 rows=31842 loops=1)
   ->  Seq Scan on delivery_zones dz  (cost=0.00..1.06 rows=6 width=NNN)
   ->  Index Scan using idx_delivery_addresses_location on delivery_addresses da
         (cost=0.14..1489.02 rows=5556 width=NNN) (actual time=0.021..101.512 rows=5307 loops=6)
         Index Cond: (location && dz.boundary)   -- bounding-box filter จาก GiST index
         Filter: st_covers(dz.boundary, location)
 Planning Time: 0.187 ms
 Execution Time: 615.203 ms
```

สังเกตว่า planner ใช้ `Nested Loop` วนลูปตามจำนวนเขต (6 เขต) และในแต่ละรอบจะใช้ **GiST index ของ `delivery_addresses`** เพื่อกรองเฉพาะแถวที่ bounding box ตัดกับ `dz.boundary` ก่อน (`Index Cond: location && dz.boundary`) แล้วค่อย `Filter` ด้วย `ST_Covers` จริงอีกที — นี่คือรูปแบบมาตรฐานของ spatial join ที่มีประสิทธิภาพ เพราะ index ช่วยตัดข้อมูลที่ไม่เกี่ยวข้องออกไปจำนวนมากตั้งแต่ต้น

---

## Step 765: ST_Buffer — สร้างพื้นที่ให้บริการ

`ST_Buffer(geography, radius_in_meters)` ใช้สร้างพื้นที่รูปหลายเหลี่ยม (polygon) ที่ล้อมรอบจุด เส้น หรือรูปทรงเดิม โดยขยายออกไปตามระยะที่กำหนด (หน่วยเป็น**เมตร**เมื่อใช้กับ `geography`) เหมาะมากสำหรับงาน "พื้นที่ให้บริการ" เช่น รัศมี 5 กม. รอบสาขา

### สร้าง Buffer รอบสาขา

```sql
-- สร้างพื้นที่ให้บริการรัศมี 5 กม. รอบสาขาสยามพารากอน
SELECT
    store_name,
    ST_Buffer(location, 5000) AS service_area_5km
FROM stores
WHERE store_name = 'สาขาสยามพารากอน';
```

ผลลัพธ์จะเป็นค่า geography รูปหลายเหลี่ยม (จริงๆ แล้วเป็นวงกลมโดยประมาณ ถูก approximate ด้วยหลายเหลี่ยมที่มีจุดจำนวนมากรอบวง) ที่แสดงเป็น WKB (Well-Known Binary) ถ้าต้องการดูเป็นข้อความให้แปลงด้วย `ST_AsText`:

```sql
SELECT
    store_name,
    ST_AsText(ST_Buffer(location, 5000)) AS service_area_wkt
FROM stores
WHERE store_name = 'สาขาสยามพารากอน';
```

### คำนวณพื้นที่ให้บริการเป็นตารางกิโลเมตร

```sql
SELECT
    store_name,
    round((ST_Area(ST_Buffer(location, 5000)) / 1000000)::numeric, 2) AS service_area_sqkm
FROM stores
WHERE store_name = 'สาขาสยามพารากอน';
```

```
     store_name      | service_area_sqkm
----------------------+--------------------
 สาขาสยามพารากอน       |              78.54
```

(π × 5² ≈ 78.54 ตร.กม. ตรงกับสูตรพื้นที่วงกลมพอดี เพราะ buffer แบบวงกลมสมบูรณ์)

### ใช้ Buffer ร่วมกับ Spatial JOIN: หาที่อยู่จัดส่งที่อยู่ในพื้นที่ให้บริการของแต่ละสาขา

```sql
SELECT
    s.store_name,
    count(da.address_id) AS orders_within_5km
FROM stores s
LEFT JOIN delivery_addresses da
    ON ST_DWithin(s.location, da.location, 5000)
    AND da.address_id <= 18
GROUP BY s.store_name
ORDER BY orders_within_5km DESC
LIMIT 10;
```

> **หมายเหตุด้าน performance:** ในทางปฏิบัติ การใช้ `ST_DWithin(a, b, distance)` มีประสิทธิภาพดีกว่าการเขียน `ST_Distance(a, b) <= distance` หรือแม้แต่ `ST_Within(a, ST_Buffer(b, distance))` เพราะ `ST_DWithin` ถูกออกแบบมาให้ query planner ใช้ spatial index ได้โดยตรง (ผ่าน bounding-box operator `&&`) โดยไม่ต้องสร้าง buffer polygon จริงขึ้นมาก่อน ซึ่งการสร้าง buffer geometry ทุกแถวมี cost สูงกว่ามาก

### Buffer บนเส้นทาง (LineString) — ตัวอย่างแนวคิด

แม้ในบทนี้เราไม่มีข้อมูลเส้นทาง (route) ในสคีมา แต่ `ST_Buffer` ใช้ได้กับ `LINESTRING` เช่นกัน ซึ่งมีประโยชน์มากสำหรับงานที่ต้องหาสิ่งที่อยู่ "ใกล้เส้นทางการจัดส่ง":

```sql
-- ตัวอย่างแนวคิด (ไม่รันจริงเพราะไม่มีตาราง routes ในบทนี้)
-- SELECT ST_Buffer(route_line, 500) AS corridor_500m FROM delivery_routes;
-- ใช้หาจุดที่อยู่ในระยะ 500 เมตรจากเส้นทางเดินรถ เช่น หาจุดจอดเสริมระหว่างทาง
```

---

## Step 766: ST_Union และ ST_Intersection

เมื่อเรามีพื้นที่ให้บริการของหลายสาขา (จาก `ST_Buffer`) คำถามที่ตามมาคือ:
- **"พื้นที่ให้บริการรวมทั้งหมดของทุกสาขาคือเท่าไร"** → ใช้ `ST_Union`
- **"พื้นที่ให้บริการของสาขาไหนที่ทับซ้อนกันบ้าง"** → ใช้ `ST_Intersection`

### ST_Union — รวมพื้นที่หลายรูปทรงเข้าด้วยกัน

```sql
-- รวมพื้นที่ให้บริการรัศมี 5 กม. ของสาขาโซนสุขุมวิททั้งหมดเข้าด้วยกัน (ไม่นับพื้นที่ซ้ำซ้อน)
WITH sukhumvit_stores AS (
    SELECT store_name, location
    FROM stores
    WHERE store_name IN ('สาขาสุขุมวิท 21 (อโศก)', 'สาขาพร้อมพงษ์', 'สาขาทองหล่อ', 'สาขาเอกมัย')
)
SELECT
    round((
        ST_Area(
            ST_Union(ST_Buffer(location, 3000))  -- รวม buffer 3 กม. ของ 4 สาขา
        ) / 1000000
    )::numeric, 2) AS combined_service_area_sqkm
FROM sukhumvit_stores;
```

สังเกตว่าถ้าเราแค่บวกพื้นที่ของแต่ละวงกลม (4 × π × 3² ≈ 113.1 ตร.กม.) จะได้ค่าที่**สูงเกินจริง** เพราะสาขาที่อยู่ใกล้กันจะมี buffer ที่ทับซ้อนกัน — `ST_Union` จะรวมรูปทรงทั้งหมดเป็นรูปเดียว (multi-polygon ที่ไม่มีการนับพื้นที่ซ้ำ) ทำให้ได้ค่าพื้นที่ที่ถูกต้องตามจริง เช่น อาจได้ผลลัพธ์ประมาณ 85-95 ตร.กม. แทนที่จะเป็น 113.1 ตร.กม.

### ST_Union แบบ Aggregate ทั้งตาราง

```sql
-- รวมพื้นที่ให้บริการรัศมี 4 กม. ของทุกสาขาทั้งหมด 27 สาขา เป็นรูปเดียว
SELECT
    round((ST_Area(ST_Union(ST_Buffer(location, 4000))) / 1000000)::numeric, 2) AS total_coverage_sqkm,
    count(*) AS total_stores
FROM stores;
```

> `ST_Union` เป็นทั้ง **aggregate function** (รวมหลายแถวเป็นรูปเดียว เหมือนตัวอย่างนี้) และ**ฟังก์ชันสองอาร์กิวเมนต์** (`ST_Union(geomA, geomB)` รวมสองรูปทรงเข้าด้วยกัน) — ใช้งานได้ทั้งสองแบบกับ `geography`

### ST_Intersection — หาพื้นที่ทับซ้อน

`ST_Intersection` คืนค่า geometry ที่เป็น**ส่วนที่ซ้อนทับกัน**ระหว่างสองรูปทรง มีประโยชน์มากสำหรับวิเคราะห์ว่าสาขาสองแห่งแย่งพื้นที่ให้บริการกันมากแค่ไหน (พื้นที่ overlap สูง = อาจสิ้นเปลืองทรัพยากร ควรปรับ zone)

```sql
-- หาพื้นที่ทับซ้อนระหว่างพื้นที่ให้บริการ (รัศมี 3 กม.) ของสาขาทองหล่อ กับสาขาเอกมัย
WITH pair AS (
    SELECT
        (SELECT ST_Buffer(location, 3000) FROM stores WHERE store_name = 'สาขาทองหล่อ') AS area_a,
        (SELECT ST_Buffer(location, 3000) FROM stores WHERE store_name = 'สาขาเอกมัย') AS area_b
)
SELECT
    round((ST_Area(ST_Intersection(area_a, area_b)) / 1000000)::numeric, 2) AS overlap_sqkm,
    round((ST_Area(area_a) / 1000000)::numeric, 2) AS thonglor_area_sqkm,
    round((ST_Area(area_b) / 1000000)::numeric, 2) AS ekkamai_area_sqkm
FROM pair;
```

### หาคู่สาขาที่มีพื้นที่ให้บริการทับซ้อนกันมากที่สุด (self-join)

```sql
SELECT
    a.store_name AS store_a,
    b.store_name AS store_b,
    round((ST_Area(ST_Intersection(
        ST_Buffer(a.location, 3000),
        ST_Buffer(b.location, 3000)
    )) / 1000000)::numeric, 2) AS overlap_sqkm
FROM stores a
JOIN stores b ON a.store_id < b.store_id     -- ป้องกันการนับคู่ซ้ำและ self-pair
WHERE ST_DWithin(a.location, b.location, 6000)  -- กรองเฉพาะคู่ที่อยู่ใกล้กันพอจะทับซ้อนได้
ORDER BY overlap_sqkm DESC
LIMIT 5;
```

ผลลัพธ์ตัวอย่าง:

```
         store_a          |         store_b          | overlap_sqkm
---------------------------+---------------------------+--------------
 สาขาพร้อมพงษ์              | สาขาทองหล่อ                |        18.42
 สาขาทองหล่อ                | สาขาเอกมัย                 |        12.87
 สาขาสุขุมวิท 21 (อโศก)      | สาขาพร้อมพงษ์               |         9.15
 สาขาปทุมวัน                | สาขาสยามพารากอน            |        24.91
 สาขาบางรัก                 | สาขาสีลม                   |        14.63
```

Query นี้เป็นตัวช่วยตัดสินใจที่มีค่ามาก — ถ้าพบว่าสาขาคู่ไหนมี `overlap_sqkm` สูงเกินไป ทีมวางแผนอาจพิจารณาปรับพื้นที่รับผิดชอบ (zone assignment) ใหม่เพื่อลดความซ้ำซ้อนของทรัพยากรขนส่ง

---

## Step 767: แนวคิดเบื้องต้นของ Geocoding

**Geocoding** คือกระบวนการแปลง **ที่อยู่ในรูปแบบข้อความ** (เช่น "123 ถนนสุขุมวิท แขวงคลองเตย เขตคลองเตย กรุงเทพฯ 10110") ให้กลายเป็น **พิกัดละติจูด/ลองจิจูด** ที่ฐานข้อมูลสามารถนำไปคำนวณเชิงพื้นที่ได้ ซึ่งเป็นขั้นตอนที่จำเป็นก่อนที่เราจะมีข้อมูลในคอลัมน์ `location` แบบที่เราใช้ตลอดบทนี้

### แนวทางหลักในการทำ Geocoding

1. **External Geocoding API** — บริการภายนอก เช่น Google Maps Geocoding API, OpenStreetMap Nominatim, HERE, Longdo Map API (สำหรับที่อยู่ไทยโดยเฉพาะ) แอปพลิเคชันจะส่งที่อยู่เป็นข้อความไปให้ API แล้วรับพิกัดกลับมา จากนั้นค่อยบันทึกลง PostgreSQL

   ```sql
   -- ตัวอย่างแนวคิด: เมื่อได้พิกัดจาก API ภายนอกแล้ว นำมาบันทึกใน PostgreSQL
   UPDATE delivery_addresses
   SET location = ST_SetSRID(ST_MakePoint(100.5613, 13.7141), 4326)::geography
   WHERE address_id = 5 AND location IS NULL;
   ```

2. **PostGIS `tiger_geocoder`** — extension เสริมของ PostGIS ที่ทำ geocoding โดยอิงข้อมูล TIGER/Line ของสำนักงานสำมะโนสหรัฐฯ (US Census Bureau) เหมาะสำหรับที่อยู่ในสหรัฐอเมริกาเท่านั้น ไม่รองรับที่อยู่ไทยโดยตรง แต่เป็นตัวอย่างที่ดีว่า PostGIS สามารถทำ geocoding "ในฐานข้อมูล" ได้โดยไม่ต้องพึ่ง API ภายนอกเสมอไป (ถ้ามีชุดข้อมูลอ้างอิงที่เหมาะสม)

3. **Self-hosted Nominatim** — ติดตั้ง Nominatim server เอง (ใช้ PostgreSQL + PostGIS เป็น backend) โดยโหลดข้อมูล OpenStreetMap ของประเทศไทยมาสร้างดัชนีที่อยู่ทั้งหมด เหมาะสำหรับองค์กรที่ต้องการทำ geocoding ปริมาณมากโดยไม่ต้องเสียค่า API และต้องการควบคุมข้อมูลเอง

4. **Fuzzy Matching กับข้อมูลอ้างอิงภายใน** — ถ้าองค์กรมีตารางอ้างอิงตำบล/อำเภอ/จังหวัดพร้อมพิกัดศูนย์กลางอยู่แล้ว สามารถใช้ `pg_trgm` (จาก Part ก่อนหน้าเรื่อง full-text/fuzzy search) จับคู่ชื่อที่อยู่แบบคร่าวๆ กับพิกัดศูนย์กลางของตำบลนั้นได้ (แม่นยำน้อยกว่า API แต่รวดเร็วและฟรี)

### สิ่งที่ควรรู้ (ไม่ลงรายละเอียด service ภายนอก)

- Geocoding ไม่มีทาง "แม่นยำ 100%" เสมอไป — ที่อยู่ที่เขียนไม่ชัดเจน, สะกดผิด, หรือเป็นสถานที่ใหม่ที่ยังไม่อยู่ใน map data จะได้พิกัดที่คลาดเคลื่อนหรือหาไม่เจอเลย
- ควรเก็บ **"ความแม่นยำ" (accuracy/precision)** ของผลลัพธ์ geocoding ไว้ด้วย เช่น เพิ่มคอลัมน์ `geocode_precision VARCHAR` (`'rooftop'`, `'street'`, `'district'`, `'approximate'`) เพื่อให้ระบบดาวน์สตรีมรู้ว่าพิกัดนี้เชื่อถือได้แค่ไหน
- ในงาน production ควรทำ geocoding แบบ **asynchronous/batch** (เช่น ผ่าน background job หรือ message queue) ไม่ใช่เรียก API แบบ synchronous ตอนผู้ใช้กรอกฟอร์ม เพราะ API ภายนอกอาจช้าหรือมี rate limit
- เมื่อได้พิกัดจาก geocoding แล้ว ควรตรวจสอบเบื้องต้นด้วย spatial query เช่น เช็คว่าพิกัดนั้นอยู่ใน bounding box ของประเทศไทยหรือไม่ (ป้องกันข้อมูลผิดพลาดจาก API หลุดเข้าระบบ)

```sql
-- ตัวอย่างการตรวจสอบพิกัดที่ได้จาก geocoding ว่าอยู่ในขอบเขตประเทศไทยแบบคร่าวๆ หรือไม่
SELECT
    address_id,
    address_line,
    CASE
        WHEN ST_X(location::geometry) BETWEEN 97.3 AND 105.7
         AND ST_Y(location::geometry) BETWEEN 5.6 AND 20.5
        THEN 'พิกัดอยู่ในขอบเขตประเทศไทย'
        ELSE 'พิกัดผิดปกติ ควรตรวจสอบ'
    END AS validation_result
FROM delivery_addresses
WHERE address_id <= 18;
```

> ฟังก์ชัน `ST_X`/`ST_Y` ใช้ดึงค่าลองจิจูด/ละติจูดจาก geometry point — เนื่องจากฟังก์ชันนี้ทำงานกับ `geometry` เท่านั้น เราจึงต้อง cast `location::geometry` ก่อนใช้งาน

---

## Step 768: แนวคิดเบื้องต้นของ Reverse Geocoding

**Reverse Geocoding** คือกระบวนการที่ทำ**ย้อนกลับ**จาก Geocoding — แปลง **พิกัด (ละติจูด/ลองจิจูด)** ให้กลายเป็น **ที่อยู่ที่มนุษย์อ่านเข้าใจได้** เช่น รับพิกัด `(13.7373, 100.5601)` แล้วได้คำตอบว่า "ใกล้ BTS อโศก แขวงคลองเตยเหนือ เขตวัฒนา กรุงเทพฯ"

Use case ที่พบบ่อยที่สุดคือแอปพลิเคชันที่ใช้ GPS ของมือถือระบุตำแหน่งผู้ใช้ แล้วต้องการแสดงชื่อสถานที่/เขตให้ผู้ใช้เห็นแทนตัวเลขพิกัดดิบๆ

### Reverse Geocoding แบบง่ายด้วยข้อมูลที่เรามีอยู่แล้ว

จริงๆ แล้ว Spatial JOIN ที่เราทำใน **Step 764** ก็คือรูปแบบหนึ่งของ reverse geocoding แบบง่าย — เรารับพิกัด (จุด) แล้วค้นหาว่ามันตกอยู่ใน "เขต" ใดที่เรามีข้อมูลขอบเขตอยู่แล้ว

```sql
-- ฟังก์ชัน reverse geocoding อย่างง่าย: รับพิกัด คืนชื่อเขตจัดส่งที่ครอบคลุม
CREATE OR REPLACE FUNCTION reverse_geocode_zone(lon DOUBLE PRECISION, lat DOUBLE PRECISION)
RETURNS TEXT AS $$
    SELECT zone_name
    FROM delivery_zones
    WHERE ST_Covers(boundary, ST_SetSRID(ST_MakePoint(lon, lat), 4326)::geography)
    LIMIT 1;
$$ LANGUAGE sql STABLE;

-- ทดลองใช้งาน
SELECT reverse_geocode_zone(100.5601, 13.7373) AS zone_name;
-- ผลลัพธ์: โซนกรุงเทพชั้นใน (Inner City)

SELECT reverse_geocode_zone(100.7333, 13.8137) AS zone_name;
-- ผลลัพธ์: โซนตะวันออก (บางกะปิ-มีนบุรี)
```

### ต่อยอด: หา "สาขาที่ใกล้ที่สุด" เป็นชื่อสถานที่อ้างอิง (แทนที่อยู่เต็ม)

ในระบบจริงที่ไม่มีฐานข้อมูลที่อยู่ระดับถนน (street-level) ครบถ้วน เทคนิคที่ใช้บ่อยคือการ "อ้างอิงจากสถานที่ที่รู้จักที่ใกล้ที่สุด" แทนการสร้างที่อยู่แบบเต็ม:

```sql
-- reverse geocoding แบบ "อ้างอิงสถานที่ใกล้เคียง" โดยใช้ KNN
CREATE OR REPLACE FUNCTION reverse_geocode_landmark(lon DOUBLE PRECISION, lat DOUBLE PRECISION)
RETURNS TEXT AS $$
    SELECT
        'ใกล้ ' || store_name || ' (ประมาณ ' ||
        round((location <-> ST_SetSRID(ST_MakePoint(lon, lat), 4326)::geography)::numeric / 1000, 1) ||
        ' กม.)'
    FROM stores
    ORDER BY location <-> ST_SetSRID(ST_MakePoint(lon, lat), 4326)::geography
    LIMIT 1;
$$ LANGUAGE sql STABLE;

SELECT reverse_geocode_landmark(100.5850, 13.7250) AS approximate_location;
-- ผลลัพธ์ตัวอย่าง: ใกล้ สาขาเอกมัย (ประมาณ 0.8 กม.)
```

### ข้อจำกัดของ Reverse Geocoding แบบภายในฐานข้อมูล

- ความแม่นยำขึ้นอยู่กับ**ความละเอียดของขอบเขต (boundary polygon)** ที่เรามี — ถ้าเขตของเราหยาบ (เช่นสี่เหลี่ยมง่ายๆ แบบในบทนี้) ผลลัพธ์ก็จะหยาบตามไปด้วย งานจริงควรใช้ขอบเขตการปกครองจริง (เขต/แขวง) จากแหล่งข้อมูล เช่น GISTDA หรือ OpenStreetMap
- สำหรับที่อยู่แบบเต็ม (บ้านเลขที่ ถนน ซอย) จำเป็นต้องพึ่งบริการ reverse geocoding ภายนอกหรือฐานข้อมูลถนน/ที่อยู่ระดับละเอียด ซึ่งอยู่นอกขอบเขตของบทนี้

---

## Step 769: Raster Data เบื้องต้นใน PostGIS

นอกเหนือจากข้อมูล **vector** (จุด เส้น รูปหลายเหลี่ยม) ที่เราใช้มาตลอดบทนี้ PostGIS ยังรองรับข้อมูลประเภท **raster** ผ่าน extension แยกชื่อ `postgis_raster` — raster คือข้อมูลแบบ "กริดภาพ" ที่แต่ละพิกเซลเก็บค่าตัวเลข เหมาะสำหรับข้อมูลประเภท:

- ภาพถ่ายดาวเทียม / ภาพถ่ายทางอากาศ
- แผนที่ความสูงภูมิประเทศ (DEM — Digital Elevation Model)
- ข้อมูลสภาพอากาศแบบกริด (อุณหภูมิ, ปริมาณน้ำฝน)
- แผนที่ความหนาแน่นประชากร หรือ heatmap เชิงพื้นที่อื่นๆ

### เปิดใช้งาน Raster Extension

```sql
CREATE EXTENSION IF NOT EXISTS postgis_raster;
```

### โครงสร้างพื้นฐาน (ภาพรวม)

```sql
-- ตัวอย่างโครงสร้างตารางที่เก็บข้อมูล raster (แนวคิดเท่านั้น)
-- CREATE TABLE elevation_map (
--     rid   SERIAL PRIMARY KEY,
--     rast  RASTER
-- );
```

การนำเข้าข้อมูล raster จริง (เช่นไฟล์ GeoTIFF ของแผนที่ความสูง) มักทำผ่านเครื่องมือ command-line ชื่อ `raster2pgsql` ที่มากับ PostGIS:

```bash
# ตัวอย่างคำสั่ง (รันนอก psql ผ่าน shell) — แปลงไฟล์ GeoTIFF เป็น SQL แล้วนำเข้าฐานข้อมูล
raster2pgsql -s 4326 -I -C -M elevation.tif public.elevation_map | psql -d mydb
```

### ตัวอย่างการ Query ข้อมูล Raster (แนวคิด)

```sql
-- ตัวอย่างแนวคิด: อ่านค่าความสูง ณ จุดพิกัดหนึ่งจากตาราง raster
-- SELECT ST_Value(rast, ST_SetSRID(ST_MakePoint(100.53, 13.75), 4326))
-- FROM elevation_map
-- WHERE ST_Intersects(rast, ST_SetSRID(ST_MakePoint(100.53, 13.75), 4326));
```

ฟังก์ชันสำคัญที่ควรรู้จักไว้ (ไม่ลงรายละเอียดในบทนี้):

| ฟังก์ชัน | หน้าที่ |
|---|---|
| `ST_Value(raster, geometry)` | อ่านค่าพิกเซลที่ตำแหน่งที่กำหนด |
| `ST_Clip(raster, geometry)` | ตัด raster เฉพาะส่วนที่อยู่ในขอบเขตที่กำหนด |
| `ST_SummaryStats(raster)` | สรุปสถิติของค่าพิกเซล (min, max, mean, stddev) |
| `ST_AsRaster(geometry, ...)` | แปลง vector geometry ให้เป็น raster |
| `ST_RasterToWorldCoord(raster, col, row)` | แปลงตำแหน่งพิกเซล (คอลัมน์/แถว) เป็นพิกัดโลกจริง |

### เมื่อไรควรใช้ Raster แทน Vector

- **Vector** (ที่เราใช้ตลอดบทนี้) เหมาะกับข้อมูลที่เป็น**วัตถุแยกชิ้นชัดเจน** เช่น ตำแหน่งร้าน ขอบเขตเขตปกครอง เส้นทางถนน
- **Raster** เหมาะกับข้อมูลที่เป็น**ค่าต่อเนื่อง (continuous surface)** ครอบคลุมพื้นที่กว้างแบบสม่ำเสมอ เช่น อุณหภูมิ ความสูง ความหนาแน่น ซึ่งไม่สามารถแทนด้วยจุดหรือเส้นได้อย่างเป็นธรรมชาติ

ระบบจัดส่งสินค้าของเราในบทนี้เป็นข้อมูล vector ล้วน (จุด สาขา/ที่อยู่ และรูปหลายเหลี่ยมเขตจัดส่ง) จึงไม่จำเป็นต้องใช้ raster แต่การรู้ว่า PostGIS มีความสามารถนี้ติดตัวอยู่แล้วมีประโยชน์มากเมื่อโปรเจกต์ในอนาคตต้องผสานข้อมูลภูมิศาสตร์เชิงกริด (เช่น วิเคราะห์พื้นที่เสี่ยงน้ำท่วมจาก DEM ร่วมกับตำแหน่งสาขา)

---

## Step 770: แบบฝึกหัดรวม — ระบบวิเคราะห์การจัดส่งแบบเต็มรูปแบบ

มาถึงหัวข้อสุดท้าย เราจะรวมทุกเทคนิคที่เรียนมาในบทนี้เข้าด้วยกัน สร้างเป็น **query ชุดวิเคราะห์การจัดส่งที่ใช้งานได้จริง** สำหรับสถานการณ์: *"มีคำสั่งซื้อใหม่เข้ามาที่พิกัดหนึ่ง ระบบต้องหาสาขาที่เหมาะสมที่สุด ตรวจสอบเขตจัดส่ง และรายงานภาพรวมพื้นที่ให้บริการทั้งระบบ"*

### โจทย์ที่ 1: หาสาขาที่ใกล้ที่สุด 3 อันดับจากจุดคำสั่งซื้อใหม่

สมมติลูกค้ารายใหม่สั่งซื้อจากพิกัด `(100.5900, 13.7550)` (แถวพระโขนง-บางจาก)

```sql
WITH new_order AS (
    SELECT ST_SetSRID(ST_MakePoint(100.5900, 13.7550), 4326)::geography AS location
)
SELECT
    s.store_id,
    s.store_name,
    round((s.location <-> no.location)::numeric / 1000, 2) AS distance_km
FROM stores s, new_order no
ORDER BY s.location <-> no.location
LIMIT 3;
```

### โจทย์ที่ 2: ตรวจสอบว่าจุดคำสั่งซื้อนี้อยู่ในเขตจัดส่งใด (Reverse Geocoding + Spatial JOIN)

```sql
WITH new_order AS (
    SELECT ST_SetSRID(ST_MakePoint(100.5900, 13.7550), 4326)::geography AS location
)
SELECT
    COALESCE(dz.zone_name, 'ไม่อยู่ในเขตจัดส่งใดเลย — ต้องพิจารณาเป็นกรณีพิเศษ') AS zone_name
FROM new_order no
LEFT JOIN delivery_zones dz
    ON ST_Covers(dz.boundary, no.location);
```

### โจทย์ที่ 3: รวมทั้งสองคำถามเป็น Query เดียว พร้อมตรวจสอบว่าสาขาที่ใกล้ที่สุดอยู่ในระยะให้บริการ (5 กม.) หรือไม่

```sql
WITH new_order AS (
    SELECT ST_SetSRID(ST_MakePoint(100.5900, 13.7550), 4326)::geography AS location
),
nearest_stores AS (
    SELECT
        s.store_id,
        s.store_name,
        round((s.location <-> no.location)::numeric / 1000, 2) AS distance_km,
        (s.location <-> no.location) <= 5000 AS within_service_range
    FROM stores s, new_order no
    ORDER BY s.location <-> no.location
    LIMIT 3
),
zone_info AS (
    SELECT COALESCE(dz.zone_name, 'ไม่อยู่ในเขตจัดส่งใดเลย') AS zone_name
    FROM new_order no
    LEFT JOIN delivery_zones dz ON ST_Covers(dz.boundary, no.location)
)
SELECT
    ns.*,
    zi.zone_name
FROM nearest_stores ns, zone_info zi
ORDER BY ns.distance_km;
```

ผลลัพธ์ตัวอย่าง:

```
 store_id |     store_name      | distance_km | within_service_range |               zone_name
----------+----------------------+--------------+-----------------------+-----------------------------------------
        9 | สาขาเอกมัย            |         2.13 | t                     | โซนสุขุมวิท-บางนา (Sukhumvit-Bangna)
       10 | สาขาอ่อนนุช           |         2.87 | t                     | โซนสุขุมวิท-บางนา (Sukhumvit-Bangna)
        8 | สาขาทองหล่อ            |         3.94 | t                     | โซนสุขุมวิท-บางนา (Sukhumvit-Bangna)
```

### โจทย์ที่ 4: คำนวณพื้นที่ให้บริการรวมของทั้งระบบ (รัศมี 5 กม. รอบทุกสาขา ไม่นับซ้ำ)

```sql
SELECT
    count(*) AS total_stores,
    round((ST_Area(ST_Union(ST_Buffer(location, 5000))) / 1000000)::numeric, 2) AS total_coverage_sqkm,
    round((count(*) * PI() * 5 * 5)::numeric, 2) AS naive_sum_sqkm,
    round((
        (count(*) * PI() * 5 * 5 - ST_Area(ST_Union(ST_Buffer(location, 5000))) / 1000000)
        / (count(*) * PI() * 5 * 5) * 100
    )::numeric, 1) AS overlap_percentage
FROM stores;
```

Query นี้เปรียบเทียบ **"พื้นที่รวมแบบไม่นับซ้ำ" (ST_Union)** กับ **"ผลรวมพื้นที่วงกลมแบบไร้เดียงสา" (naive sum)** เพื่อแสดงให้เห็นว่าเครือข่ายสาขามีความซ้ำซ้อนของพื้นที่ให้บริการมากน้อยเพียงใด — ตัวเลข `overlap_percentage` สูง แปลว่าสาขาหลายแห่งตั้งอยู่ใกล้กันเกินไป (มีโอกาสปรับปรุงการกระจายสาขาให้ครอบคลุมพื้นที่กว้างขึ้นโดยใช้จำนวนสาขาเท่าเดิม)

### โจทย์ที่ 5: รายงานสรุปแบบเต็ม — จำนวนออเดอร์ สาขาที่รับผิดชอบ และระยะทางเฉลี่ยต่อเขต

```sql
SELECT
    dz.zone_name,
    count(da.address_id) AS total_orders,
    (SELECT s.store_name
     FROM stores s
     ORDER BY s.location <-> dz.boundary
     LIMIT 1) AS closest_store_to_zone_center,
    round(avg(
        (SELECT min(s2.location <-> da.location) FROM stores s2)
    )::numeric / 1000, 2) AS avg_distance_to_nearest_store_km
FROM delivery_zones dz
LEFT JOIN delivery_addresses da
    ON ST_Covers(dz.boundary, da.location)
    AND da.address_id <= 18
GROUP BY dz.zone_name, dz.boundary
ORDER BY total_orders DESC;
```

> **หมายเหตุ:** การใช้ `s.location <-> dz.boundary` ในบรรทัด `closest_store_to_zone_center` เป็นการหาสาขาที่ใกล้กับ**ขอบเขตของโซน** (ไม่ใช่จุดศูนย์กลาง) มากที่สุด ซึ่ง PostGIS รองรับการคำนวณ KNN ระหว่าง point กับ polygon ได้โดยตรงเช่นกัน

### สรุปสิ่งที่ query ชุดนี้แสดงให้เห็น

จากแบบฝึกหัดรวมนี้ เราได้ประกอบทุกเทคนิคของบทเข้าด้วยกัน:

| เทคนิค | ใช้ในโจทย์ข้อใด |
|---|---|
| GiST Spatial Index (Step 761-762) | ทุกโจทย์ (เป็นพื้นฐานให้ query ทั้งหมดเร็ว) |
| KNN Operator `<->` (Step 763) | โจทย์ 1, 3, 5 |
| Spatial JOIN (Step 764) | โจทย์ 2, 3, 5 |
| ST_Buffer (Step 765) | โจทย์ 4 |
| ST_Union (Step 766) | โจทย์ 4 |

นี่คือรูปแบบการทำงานจริงของวิศวกรข้อมูลที่ดูแลระบบ logistics/delivery — การผสมผสานเทคนิคเชิงพื้นที่หลายแบบเข้าด้วยกันในหนึ่ง query pipeline เพื่อตอบโจทย์ทางธุรกิจที่ซับซ้อน

---

## สรุปท้ายบท

ในบทนี้เราได้ยกระดับความสามารถด้าน PostGIS จาก Part 076 ขึ้นไปอีกขั้น โดยเน้นเรื่อง **performance** และ **query ขั้นสูง** ที่จำเป็นสำหรับระบบเชิงพื้นที่ระดับ production:

1. **GiST Spatial Index** คือหัวใจของ performance ทั้งหมดในบทนี้ — ใช้แนวคิด bounding box (MBR) จัดโครงสร้างต้นไม้เพื่อตัดข้อมูลที่ไม่เกี่ยวข้องออกอย่างรวดเร็ว
2. การเปรียบเทียบ `EXPLAIN ANALYZE` แสดงให้เห็นชัดเจนว่า spatial index เปลี่ยน query จาก **Seq Scan** (O(n)) เป็น **Bitmap Index Scan** (ใกล้เคียง O(log n)) ทำให้เร็วขึ้นหลายสิบเท่าเมื่อข้อมูลมีจำนวนมาก
3. **KNN operator `<->`** คือเครื่องมือที่ถูกต้องสำหรับ "หาจุดที่ใกล้ที่สุด N อันดับ" — ต่างจาก `ORDER BY ST_Distance(...) LIMIT N` ตรงที่สามารถใช้ index ช่วยเรียงลำดับได้โดยตรง ไม่ต้องคำนวณระยะทางทุกแถว
4. **Spatial JOIN** (ด้วย `ST_Covers`, `ST_Intersects`, `ST_DWithin`) ใช้เชื่อมโยงข้อมูลสองตารางด้วยความสัมพันธ์เชิงพื้นที่แทนการเทียบ key ตรงๆ และยังได้ประโยชน์จาก spatial index ผ่าน bounding-box operator `&&` อัตโนมัติ
5. **ST_Buffer** สร้างพื้นที่ให้บริการรอบจุด/เส้น ส่วน **ST_Union** และ **ST_Intersection** ใช้วิเคราะห์การรวมและการทับซ้อนของพื้นที่หลายส่วน ซึ่งเป็นเครื่องมือสำคัญสำหรับวางแผนโครงข่ายสาขา
6. **Geocoding** และ **Reverse Geocoding** คือกระบวนการแปลงที่อยู่ ↔ พิกัด ซึ่งในงานจริงมักพึ่งพา service ภายนอกหรือชุดข้อมูลอ้างอิง แต่ PostGIS เองก็มีเครื่องมือพื้นฐาน (`ST_Covers`, KNN) ที่ช่วยทำ reverse geocoding แบบง่ายภายในฐานข้อมูลได้
7. **Raster data** เป็นความสามารถเสริมของ PostGIS (ผ่าน `postgis_raster`) สำหรับข้อมูลแบบกริดต่อเนื่อง เช่น แผนที่ความสูงหรือภาพถ่ายดาวเทียม ซึ่งเป็นคนละรูปแบบข้อมูลจาก vector ที่เราใช้หลักในบทนี้

ทักษะเหล่านี้ทำให้เราสามารถออกแบบและดูแลระบบเชิงพื้นที่ที่รองรับข้อมูลระดับล้านแถวได้อย่างมั่นใจ ไม่ว่าจะเป็นระบบ delivery, ride-hailing, real estate, หรือ location-based service ประเภทใดก็ตาม

ในบทถัดไป เราจะเปลี่ยนโฟกัสไปยังการจัดการข้อมูลอนุกรมเวลา (time-series data) ด้วย **TimescaleDB** ซึ่งเป็น extension ยอดนิยมของ PostgreSQL สำหรับงาน IoT, monitoring, และ analytics ที่มีข้อมูลไหลเข้าต่อเนื่องตามเวลา

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1

สร้าง GiST index บนคอลัมน์ `location` ของตาราง `delivery_addresses` (ถ้ายังไม่มี) และเขียนคำสั่งตรวจสอบว่า index ถูกสร้างสำเร็จโดยดูจาก `pg_indexes`

<details>
<summary>เฉลย</summary>

```sql
CREATE INDEX IF NOT EXISTS idx_delivery_addresses_location
    ON delivery_addresses USING GIST (location);

SELECT indexname, indexdef
FROM pg_indexes
WHERE tablename = 'delivery_addresses';
```

หากคำสั่ง `SELECT` คืนแถวที่มี `indexname = 'idx_delivery_addresses_location'` และ `indexdef` มีข้อความ `USING gist (location)` แสดงว่า index ถูกสร้างสำเร็จ
</details>

### แบบฝึกหัดที่ 2

เขียน query เปรียบเทียบ `EXPLAIN ANALYZE` ของการค้นหา "ที่อยู่จัดส่งทั้งหมดที่อยู่ในระยะ 2,000 เมตรจากสาขาบางนา" ก่อนและหลังมี spatial index อธิบายว่า plan เปลี่ยนจากอะไรเป็นอะไร

<details>
<summary>เฉลย</summary>

```sql
-- ก่อนมี index
DROP INDEX IF EXISTS idx_delivery_addresses_location;
EXPLAIN ANALYZE
SELECT address_id FROM delivery_addresses
WHERE ST_DWithin(
    location,
    (SELECT location FROM stores WHERE store_name = 'สาขาบางนา'),
    2000
);
-- คาดว่าจะเห็น: Seq Scan on delivery_addresses ... Filter: st_dwithin(...)

-- หลังมี index
CREATE INDEX idx_delivery_addresses_location ON delivery_addresses USING GIST (location);
ANALYZE delivery_addresses;
EXPLAIN ANALYZE
SELECT address_id FROM delivery_addresses
WHERE ST_DWithin(
    location,
    (SELECT location FROM stores WHERE store_name = 'สาขาบางนา'),
    2000
);
-- คาดว่าจะเห็น: Bitmap Heap Scan ... -> Bitmap Index Scan using idx_delivery_addresses_location
```

Plan เปลี่ยนจาก **Seq Scan** (อ่านทุกแถวแล้วคำนวณระยะทางทีละแถว) เป็น **Bitmap Index Scan → Bitmap Heap Scan** (ใช้ GiST index กรอง bounding box ก่อน แล้วค่อยดึงเฉพาะแถวที่เกี่ยวข้องจาก heap) ทำให้ execution time ลดลงอย่างมีนัยสำคัญ
</details>

### แบบฝึกหัดที่ 3

ใช้ KNN operator `<->` เขียน query หาสาขา 5 อันดับที่ใกล้ที่สุดจากจุด "สยามพารากอน" (พิกัด 100.5344, 13.7460) พร้อมแสดงระยะทางเป็นกิโลเมตร ปัดเศษ 2 ตำแหน่ง

<details>
<summary>เฉลย</summary>

```sql
SELECT
    store_name,
    round((location <-> ST_SetSRID(ST_MakePoint(100.5344, 13.7460), 4326)::geography)::numeric / 1000, 2) AS distance_km
FROM stores
ORDER BY location <-> ST_SetSRID(ST_MakePoint(100.5344, 13.7460), 4326)::geography
LIMIT 5;
```

Query นี้จะได้ประโยชน์จาก GiST index บนคอลัมน์ `stores.location` (ถ้ามี) โดยเปลี่ยนเป็น Index Scan ที่เรียงตามระยะทางโดยตรง แทนการคำนวณระยะทางของทุกแถวแล้วค่อย sort
</details>

### แบบฝึกหัดที่ 4

อธิบายว่าทำไม `ORDER BY ST_Distance(location, ref_point) LIMIT 5` ถึงมีประสิทธิภาพด้อยกว่า `ORDER BY location <-> ref_point LIMIT 5` แม้ทั้งสอง query จะให้ผลลัพธ์เหมือนกัน

<details>
<summary>เฉลย</summary>

`ST_Distance` เป็นฟังก์ชันธรรมดาที่ PostgreSQL query planner **ไม่รู้ว่าเกี่ยวข้องกับ index ได้** ดังนั้นเมื่อใช้ใน `ORDER BY` planner จะต้องคำนวณค่า `ST_Distance` ของ**ทุกแถวในตาราง** ก่อน แล้วค่อย sort ทั้งหมดเพื่อหา 5 อันดับแรก (Seq Scan + Sort)

ในทางกลับกัน `<->` เป็น **distance operator ที่ผูกกับ operator class ของ GiST index โดยตรง** (`gist_geography_ops`/`gist_geometry_ops_2d`) เมื่อใช้คู่กับ `ORDER BY ... LIMIT N` และมี GiST index อยู่ query planner จะสามารถทำ **KNN-GiST search** ที่ดึงผลลัพธ์ตามลำดับระยะทางออกมาทีละแถวจาก index โดยตรง แล้วหยุดทันทีเมื่อได้ครบ `LIMIT` โดยไม่ต้องแตะแถวที่เหลือเลย ทำให้เร็วกว่ามากในตารางขนาดใหญ่
</details>

### แบบฝึกหัดที่ 5

เขียน spatial join เพื่อหาจำนวนที่อยู่จัดส่งทั้งหมด (address_id 1-18) ที่**ไม่ได้อยู่ในเขตจัดส่งใดเลย** จากตาราง `delivery_zones` ทั้ง 6 เขต

<details>
<summary>เฉลย</summary>

```sql
SELECT count(*) AS orders_without_zone
FROM delivery_addresses da
WHERE da.address_id <= 18
  AND NOT EXISTS (
      SELECT 1 FROM delivery_zones dz
      WHERE ST_Covers(dz.boundary, da.location)
  );
```

หรือใช้ `LEFT JOIN` แล้วกรองด้วย `WHERE dz.zone_id IS NULL`:

```sql
SELECT count(*) AS orders_without_zone
FROM delivery_addresses da
LEFT JOIN delivery_zones dz ON ST_Covers(dz.boundary, da.location)
WHERE da.address_id <= 18 AND dz.zone_id IS NULL;
```
</details>

### แบบฝึกหัดที่ 6

สร้างพื้นที่ให้บริการรัศมี 2 กม. รอบ "สาขาสีลม" และคำนวณว่ามีที่อยู่จัดส่ง (address_id 1-18) กี่รายการที่อยู่ในพื้นที่นี้

<details>
<summary>เฉลย</summary>

```sql
SELECT count(*) AS orders_within_range
FROM delivery_addresses da
JOIN stores s ON s.store_name = 'สาขาสีลม'
WHERE ST_DWithin(s.location, da.location, 2000)
  AND da.address_id <= 18;
```

หรือใช้ `ST_Within` กับ buffer geometry จริงก็ได้ (แต่ประสิทธิภาพด้อยกว่า `ST_DWithin` เพราะต้องสร้าง buffer polygon จริงก่อนเปรียบเทียบ):

```sql
SELECT count(*) AS orders_within_range
FROM delivery_addresses da
JOIN stores s ON s.store_name = 'สาขาสีลม'
WHERE ST_Covers(ST_Buffer(s.location, 2000), da.location)
  AND da.address_id <= 18;
```
</details>

### แบบฝึกหัดที่ 7

ใช้ `ST_Union` คำนวณพื้นที่ให้บริการรวม (ไม่นับซ้ำ) ของสาขาในโซนกรุงเทพชั้นใน 3 แห่ง: สาขาสยามพารากอน, สาขาปทุมวัน, สาขาสีลม โดยแต่ละสาขามีรัศมีให้บริการ 2.5 กม.

<details>
<summary>เฉลย</summary>

```sql
WITH target_stores AS (
    SELECT location
    FROM stores
    WHERE store_name IN ('สาขาสยามพารากอน', 'สาขาปทุมวัน', 'สาขาสีลม')
)
SELECT
    round((ST_Area(ST_Union(ST_Buffer(location, 2500))) / 1000000)::numeric, 2) AS combined_area_sqkm
FROM target_stores;
```
</details>

### แบบฝึกหัดที่ 8

เขียน query หาคู่สาขาสองแห่งที่มีระยะห่างกันน้อยกว่า 1.5 กม. (ใช้บ่งชี้ว่าอาจมีสาขาตั้งอยู่ใกล้กันเกินไป)

<details>
<summary>เฉลย</summary>

```sql
SELECT
    a.store_name AS store_a,
    b.store_name AS store_b,
    round((ST_Distance(a.location, b.location))::numeric, 0) AS distance_m
FROM stores a
JOIN stores b ON a.store_id < b.store_id
WHERE ST_DWithin(a.location, b.location, 1500)
ORDER BY distance_m;
```

การใช้ `ST_DWithin` ใน `WHERE` แทน `ST_Distance(...) < 1500` ทำให้ query สามารถใช้ spatial index ช่วยกรองคู่ที่เป็นไปได้ก่อน (ผ่าน bounding-box operator) แทนที่จะคำนวณระยะทางของทุกคู่สาขาแบบ brute-force
</details>

### แบบฝึกหัดที่ 9

อธิบายความแตกต่างระหว่าง **Geocoding** กับ **Reverse Geocoding** พร้อมยกตัวอย่างการใช้งานแต่ละแบบในระบบจัดส่งสินค้าของบทนี้

<details>
<summary>เฉลย</summary>

- **Geocoding** คือการแปลง **ที่อยู่ข้อความ → พิกัด** เช่น เมื่อลูกค้ากรอกที่อยู่จัดส่งในฟอร์ม ("123 ถนนสุขุมวิท เขตวัฒนา") ระบบต้องแปลงเป็นพิกัด `(100.56, 13.73)` เพื่อบันทึกลงคอลัมน์ `location` ของตาราง `delivery_addresses` และนำไปคำนวณระยะทาง/หาสาขาที่ใกล้ที่สุดต่อไปได้

- **Reverse Geocoding** คือการแปลง **พิกัด → ที่อยู่/ชื่อสถานที่ที่อ่านเข้าใจได้** เช่น เมื่อแอปมือถือของไรเดอร์ส่งพิกัด GPS ปัจจุบันมา ระบบต้องแปลงกลับเป็นชื่อเขตหรือสถานที่อ้างอิง เพื่อแสดงผลให้ผู้ดูแลระบบเข้าใจง่าย (เช่นฟังก์ชัน `reverse_geocode_zone()` ใน Step 768 ที่รับพิกัดแล้วคืนชื่อ `delivery_zones` ที่ครอบคลุมจุดนั้น)

กล่าวโดยสรุป: Geocoding ใช้ตอน **รับข้อมูลเข้า** (ที่อยู่ผู้ใช้กรอก) ส่วน Reverse Geocoding ใช้ตอน **แสดงผลข้อมูลออก** (พิกัดดิบที่ต้องแปลงเป็นสิ่งที่มนุษย์อ่านเข้าใจ)
</details>

### แบบฝึกหัดที่ 10

รวมเทคนิคทั้งหมดในบทนี้: เขียน query เดียวที่รับพิกัดคำสั่งซื้อใหม่ `(100.55, 13.80)` แล้วคืนค่า (ก) สาขาที่ใกล้ที่สุด 3 อันดับพร้อมระยะทาง (ข) เขตจัดส่งที่ครอบคลุมจุดนี้ และ (ค) บอกว่าสาขาที่ใกล้ที่สุดอยู่ในระยะบริการ 5 กม. หรือไม่

<details>
<summary>เฉลย</summary>

```sql
WITH new_order AS (
    SELECT ST_SetSRID(ST_MakePoint(100.55, 13.80), 4326)::geography AS location
),
nearest_3 AS (
    SELECT
        s.store_name,
        round((s.location <-> no.location)::numeric / 1000, 2) AS distance_km,
        (s.location <-> no.location) <= 5000 AS within_5km_service
    FROM stores s, new_order no
    ORDER BY s.location <-> no.location
    LIMIT 3
),
matched_zone AS (
    SELECT COALESCE(dz.zone_name, 'ไม่พบเขตจัดส่งที่ครอบคลุม') AS zone_name
    FROM new_order no
    LEFT JOIN delivery_zones dz ON ST_Covers(dz.boundary, no.location)
)
SELECT n.*, z.zone_name
FROM nearest_3 n, matched_zone z
ORDER BY n.distance_km;
```

Query นี้รวม KNN operator (`<->`), การเปรียบเทียบระยะทางกับเกณฑ์บริการ, และ spatial join กับ `delivery_zones` ไว้ในคำสั่งเดียว ซึ่งเป็นรูปแบบ query ที่ระบบ dispatch หรือ order-routing engine ใช้งานจริงเมื่อมีคำสั่งซื้อใหม่เข้ามาในระบบแบบ real-time
</details>

---

**บทถัดไป:** [Part 078: TimescaleDB สำหรับข้อมูล Time-Series](./part-078-timescaledb.md)
