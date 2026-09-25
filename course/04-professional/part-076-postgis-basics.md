# Part 076: PostGIS เบื้องต้น — Geospatial Database

> **หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับมืออาชีพ | Part 076**

---

## เป้าหมายการเรียนรู้

เมื่อเรียนจบบทนี้ ผู้เรียนจะสามารถ:

1. อธิบายได้ว่า PostGIS คืออะไร และเหตุใด PostgreSQL จึงเป็นฐานข้อมูลที่เหมาะกับงาน geospatial มากที่สุดตัวหนึ่ง
2. ติดตั้งและเปิดใช้งาน extension `postgis` ได้อย่างถูกต้อง
3. เข้าใจแนวคิดเรื่องระบบพิกัด (Spatial Reference System) และ SRID โดยเฉพาะ WGS84 (SRID 4326)
4. แยกความแตกต่างระหว่าง `geometry` type กับ `geography` type และเลือกใช้ให้เหมาะกับงาน
5. สร้างและจัดเก็บข้อมูลเชิงพื้นที่แบบ `POINT`, `LINESTRING`, และ `POLYGON`
6. คำนวณระยะทางระหว่างจุดสองจุดด้วย `ST_Distance` เพื่อหาสาขาที่ใกล้ลูกค้าที่สุด
7. ค้นหาจุดทั้งหมดที่อยู่ในรัศมีที่กำหนดด้วย `ST_DWithin`
8. ตรวจสอบว่าจุดอยู่ในพื้นที่ (polygon) หรือไม่ด้วย `ST_Contains` และ `ST_Intersects`
9. แปลงข้อมูลพิกัดเป็นรูปแบบข้อความหรือ GeoJSON เพื่อส่งไปแสดงผลบนแผนที่ (เช่น Leaflet, Mapbox, Google Maps)
10. ประยุกต์ใช้ความรู้ทั้งหมดสร้างระบบ "หาสาขาใกล้ที่สุด" และ "ตรวจสอบพื้นที่ให้บริการจัดส่ง" สำหรับระบบ e-commerce จริง

> **หมายเหตุสำคัญก่อนเริ่มบทเรียน**
>
> PostGIS เป็น **extension** ที่ต้องติดตั้งแยกต่างหากจาก PostgreSQL หลัก ไม่ใช่ทุกเซิร์ฟเวอร์หรือทุก environment (เช่น บาง managed service ระดับ free tier, บาง container image แบบ slim, หรือ sandbox บางตัว) จะมี PostGIS ติดตั้งไว้ให้ล่วงหน้า ก่อนรันโค้ดในบทนี้ ผู้เรียน**ต้อง**ตรวจสอบและติดตั้งให้พร้อมก่อน (วิธีติดตั้งอยู่ใน Step 751)
>
> โค้ด SQL ทุกตัวอย่างในบทนี้เขียนให้ **ถูกต้องและรันได้จริง** บนเซิร์ฟเวอร์ที่ติดตั้ง PostGIS แล้ว (PostgreSQL 16/17 + PostGIS 3.4/3.5) — เขียนขึ้นในลักษณะเดียวกับที่จะใช้งานจริงบน production server ที่ทำตามขั้นตอนติดตั้งในบทนี้แล้ว หากผู้เรียนรันบนเครื่องที่ยังไม่มี PostGIS จะได้รับ error ประเภท `extension "postgis" is not available` หรือ `type "geography" does not exist` — แก้ไขโดยติดตั้ง PostGIS ให้กับระบบปฏิบัติการก่อน (ดู Step 751)

---

## เตรียมข้อมูล

เนื่องจากบทนี้เป็นเรื่องใหม่ที่ไม่เกี่ยวข้องโดยตรงกับ schema เดิมของระบบ e-commerce ที่เราใช้มาตลอดหลักสูตร (orders, products, customers ฯลฯ) เราจะสร้าง **schema ใหม่** สำหรับโจทย์เชิงภูมิศาสตร์โดยเฉพาะ คือระบบ "สาขาหน้าร้าน" (physical stores) และ "ที่อยู่จัดส่ง" (delivery addresses) ของธุรกิจ e-commerce เดิมของเรา ที่ตอนนี้ขยายกิจการมาเปิดหน้าร้านจริงในกรุงเทพฯ ด้วย

### ขั้นตอนที่ 1: ติดตั้งและเปิดใช้งาน PostGIS

```sql
-- ต้องมีสิทธิ์ superuser หรือสิทธิ์ที่ได้รับอนุญาตให้สร้าง extension
CREATE EXTENSION IF NOT EXISTS postgis;

-- ตรวจสอบเวอร์ชันที่ติดตั้ง
SELECT PostGIS_Full_Version();
```

**ผลลัพธ์ตัวอย่าง:**

```
                                                          postgis_full_version
------------------------------------------------------------------------------------------------------------------------------
 POSTGIS="3.4.2" [EXTENSION] PGSQL="160" GEOS="3.12.1-CAPI-1.18.1" PROJ="9.3.1" LIBXML="2.9.14" LIBJSON="0.17" LIBPROTOBUF="1.4.1"
(1 row)
```

### ขั้นตอนที่ 2: สร้างตารางสำหรับสาขาและที่อยู่จัดส่ง

```sql
CREATE TABLE stores (
    store_id     SERIAL PRIMARY KEY,
    store_name   VARCHAR(150) NOT NULL,
    address      TEXT,
    phone        VARCHAR(20),
    location     GEOGRAPHY(POINT, 4326)
);

CREATE TABLE delivery_addresses (
    address_id     SERIAL PRIMARY KEY,
    customer_id    INTEGER,
    address_line   TEXT,
    recipient_name VARCHAR(150),
    location       GEOGRAPHY(POINT, 4326)
);

-- ตารางเขตพื้นที่ให้บริการจัดส่ง (จะใช้ในหัวข้อ POLYGON)
CREATE TABLE delivery_zones (
    zone_id      SERIAL PRIMARY KEY,
    zone_name    VARCHAR(150) NOT NULL,
    coverage     GEOGRAPHY(POLYGON, 4326)
);
```

### ขั้นตอนที่ 3: ใส่ข้อมูลสาขาจริงในกรุงเทพฯ

สมมติธุรกิจของเรามีสาขาหน้าร้าน 5 แห่งในกรุงเทพฯ (พิกัดใกล้เคียงของจริง เพื่อให้แบบฝึกหัดสมจริง):

```sql
INSERT INTO stores (store_name, address, phone, location) VALUES
('สาขาสยามพารากอน',
 '991 ถนนพระราม 1 แขวงปทุมวัน เขตปทุมวัน กรุงเทพฯ 10330',
 '02-610-8000',
 ST_SetSRID(ST_MakePoint(100.5347, 13.7466), 4326)::geography),

('สาขาเซ็นทรัลเวิลด์',
 '4-4/5 ถนนราชดำริ แขวงปทุมวัน เขตปทุมวัน กรุงเทพฯ 10330',
 '02-640-7000',
 ST_SetSRID(ST_MakePoint(100.5393, 13.7466), 4326)::geography),

('สาขาไอคอนสยาม',
 '299 ถนนเจริญนคร แขวงคลองต้นไทร เขตคลองสาน กรุงเทพฯ 10600',
 '02-495-7000',
 ST_SetSRID(ST_MakePoint(100.5099, 13.7262), 4326)::geography),

('สาขาเทอร์มินอล 21 อโศก',
 '88 ถนนสุขุมวิท แขวงคลองเตยเหนือ เขตวัฒนา กรุงเทพฯ 10110',
 '02-108-0888',
 ST_SetSRID(ST_MakePoint(100.5602, 13.7373), 4326)::geography),

('สาขาเอ็มบีเค เซ็นเตอร์',
 '444 ถนนพญาไท แขวงวังใหม่ เขตปทุมวัน กรุงเทพฯ 10330',
 '02-620-9000',
 ST_SetSRID(ST_MakePoint(100.5300, 13.7447), 4326)::geography);
```

### ขั้นตอนที่ 4: ใส่ข้อมูลที่อยู่จัดส่งของลูกค้า

```sql
INSERT INTO delivery_addresses (customer_id, address_line, recipient_name, location) VALUES
(101, '123/45 ซอยสุขุมวิท 24 แขวงคลองตัน เขตคลองเตย กรุงเทพฯ 10110', 'คุณสมชาย ใจดี',
 ST_SetSRID(ST_MakePoint(100.5650, 13.7280), 4326)::geography),

(102, '78 ถนนสีลม แขวงสีลม เขตบางรัก กรุงเทพฯ 10500', 'คุณสมหญิง รักเรียน',
 ST_SetSRID(ST_MakePoint(100.5341, 13.7245), 4326)::geography),

(103, '999 หมู่บ้านสายไหม ถนนสายไหม เขตสายไหม กรุงเทพฯ 10220', 'คุณวิชัย มั่นคง',
 ST_SetSRID(ST_MakePoint(100.6540, 13.9150), 4326)::geography),

(104, '55/2 ซอยทองหล่อ 10 แขวงคลองตันเหนือ เขตวัฒนา กรุงเทพฯ 10110', 'คุณนภา สว่างใจ',
 ST_SetSRID(ST_MakePoint(100.5789, 13.7320), 4326)::geography),

(105, '10 ถนนรัชดาภิเษก แขวงห้วยขวาง เขตห้วยขวาง กรุงเทพฯ 10310', 'คุณอนันต์ พูนทรัพย์',
 ST_SetSRID(ST_MakePoint(100.5735, 13.7690), 4326)::geography);
```

จากนี้ไปในบทเรียน เราจะใช้ schema นี้ (`stores`, `delivery_addresses`, `delivery_zones`) เป็นฐานในการอธิบายทุกคำสั่งของ PostGIS

---

## Step 751: PostGIS คืออะไร

**PostGIS** คือ extension (ส่วนเสริม) ของ PostgreSQL ที่เพิ่มความสามารถด้าน **Geospatial Database** หรือฐานข้อมูลเชิงพื้นที่ให้กับ PostgreSQL ทำให้ PostgreSQL สามารถ:

- จัดเก็บข้อมูลตำแหน่งทางภูมิศาสตร์ (จุด เส้น พื้นที่) ในรูปแบบมาตรฐานสากล
- คำนวณระยะทาง พื้นที่ ความยาว ระหว่างวัตถุเชิงพื้นที่
- ตรวจสอบความสัมพันธ์เชิงพื้นที่ เช่น จุดนี้อยู่ในพื้นที่นี้หรือไม่ เส้นสองเส้นตัดกันหรือไม่
- สร้าง spatial index (GiST) เพื่อค้นหาข้อมูลเชิงพื้นที่ได้อย่างรวดเร็วแม้มีข้อมูลนับล้านแถว
- แปลงข้อมูลไปมาระหว่างรูปแบบมาตรฐาน เช่น GeoJSON, WKT (Well-Known Text), WKB (Well-Known Binary), KML

PostGIS ถูกใช้งานอย่างแพร่หลายในอุตสาหกรรมที่ต้องพึ่งพาข้อมูลตำแหน่ง เช่น ระบบขนส่ง (logistics), แอปพลิเคชันแผนที่, ระบบ ride-hailing (Grab, Uber), การวิเคราะห์ผังเมือง, การเกษตรแม่นยำ (precision agriculture), และแน่นอนว่ารวมถึงระบบ e-commerce ที่ต้องการหาสาขาใกล้ที่สุด หรือกำหนดพื้นที่ให้บริการจัดส่ง ซึ่งเป็นโจทย์หลักของบทนี้

### ทำไมต้องใช้ PostGIS แทนที่จะเก็บ latitude/longitude เป็นตัวเลขธรรมดา

หลายคนอาจสงสัยว่า ในเมื่อพิกัด GPS ก็เป็นแค่ตัวเลขคู่หนึ่ง (`latitude`, `longitude`) ทำไมไม่เก็บเป็นคอลัมน์ `NUMERIC` สองคอลัมน์ธรรมดาแล้วคำนวณระยะทางด้วยสูตรคณิตศาสตร์เอง? คำตอบคือ:

1. **ความถูกต้องของการคำนวณระยะทาง** — โลกไม่ใช่พื้นราบ การคำนวณระยะทางระหว่างสองพิกัด GPS ต้องใช้สูตรทรงกลม (เช่น Haversine formula) ซึ่งซับซ้อนและมีโอกาสเขียนผิดสูง PostGIS มีฟังก์ชันที่ผ่านการทดสอบมาอย่างดีให้ใช้ทันที
2. **ประสิทธิภาพ** — PostGIS รองรับ spatial index (GiST) ทำให้การค้นหา "จุดที่อยู่ใกล้จุดหนึ่งในรัศมี X กม." เร็วกว่าการคำนวณ brute-force ทุกแถวมาก
3. **ฟังก์ชันสำเร็จรูปจำนวนมาก** — ตรวจสอบว่าจุดอยู่ในพื้นที่หรือไม่ (`ST_Contains`), หาจุดตัดของเส้นทาง, คำนวณพื้นที่ของ polygon ฯลฯ ล้วนมีให้พร้อมใช้
4. **มาตรฐานสากล** — ข้อมูลที่เก็บด้วย PostGIS สามารถแลกเปลี่ยนกับระบบ GIS อื่น ๆ ได้ทันที (QGIS, ArcGIS, Mapbox, Leaflet) ผ่านรูปแบบมาตรฐานอย่าง GeoJSON หรือ WKT

### วิธีติดตั้ง PostGIS

การติดตั้งแบ่งเป็น 2 ขั้นตอน คือ (1) ติดตั้ง package ของ PostGIS ที่ระดับระบบปฏิบัติการ และ (2) เปิดใช้งาน extension ที่ระดับฐานข้อมูล

**ขั้นตอนที่ 1 — ติดตั้ง package ที่ระดับ OS**

บน Ubuntu/Debian:

```bash
# ตรวจสอบเวอร์ชัน PostgreSQL ที่ใช้อยู่ก่อน เช่น 16
sudo apt update
sudo apt install postgresql-16-postgis-3
```

บน RHEL/Rocky/AlmaLinux (ผ่าน PGDG repository):

```bash
sudo dnf install postgis34_16
```

บน macOS ผ่าน Homebrew:

```bash
brew install postgis
```

บน Docker (วิธีที่นิยมมากสำหรับการพัฒนา/ทดสอบ) ให้ใช้ image ที่มี PostGIS มาพร้อมแล้วแทนที่จะใช้ image `postgres` เปล่า ๆ:

```bash
docker run -d --name pg-postgis \
  -e POSTGRES_PASSWORD=secret \
  -p 5432:5432 \
  postgis/postgis:16-3.4
```

> หากใช้บริการ managed database (เช่น AWS RDS, Google Cloud SQL, Azure Database for PostgreSQL) ผู้ให้บริการส่วนใหญ่รองรับ PostGIS อยู่แล้ว แต่บาง tier หรือบาง instance class อาจต้องเปิดใช้งานผ่านหน้า console ก่อน จึงจะรัน `CREATE EXTENSION postgis;` สำเร็จ

**ขั้นตอนที่ 2 — เปิดใช้งาน extension ในฐานข้อมูล**

หลังติดตั้ง package แล้ว ให้เชื่อมต่อเข้าฐานข้อมูลที่ต้องการใช้งาน แล้วรัน:

```sql
CREATE EXTENSION IF NOT EXISTS postgis;
```

ตรวจสอบว่าติดตั้งสำเร็จด้วยการดูรายชื่อ extension ที่เปิดใช้งานอยู่:

```sql
SELECT extname, extversion
FROM pg_extension
WHERE extname = 'postgis';
```

**ผลลัพธ์ตัวอย่าง:**

```
 extname | extversion
---------+------------
 postgis | 3.4.2
(1 row)
```

หากคำสั่ง `CREATE EXTENSION postgis;` แจ้ง error ว่า `could not open extension control file "...postgis.control": No such file or directory` แปลว่ายังไม่ได้ติดตั้ง package ที่ระดับ OS (ขั้นตอนที่ 1) ให้ย้อนกลับไปติดตั้งก่อน

> **ข้อควรระวัง:** extension นี้ต้องติดตั้ง**แยกในแต่ละฐานข้อมูล** (database) ที่ต้องการใช้งาน ไม่ใช่ติดตั้งครั้งเดียวใช้ได้ทั้ง PostgreSQL server หากมีหลายฐานข้อมูล ต้องรัน `CREATE EXTENSION postgis;` ในแต่ละฐานข้อมูลที่ต้องการ

---

## Step 752: ระบบพิกัดและ SRID

### Spatial Reference System (SRS) คืออะไร

การระบุตำแหน่งบนโลกไม่ได้มีมาตรฐานเดียว มีระบบพิกัด (Coordinate Reference System) หลายแบบ ขึ้นอยู่กับวัตถุประสงค์การใช้งาน เช่น บางระบบใช้หน่วยเป็นองศา (degree) บางระบบใช้หน่วยเป็นเมตร บางระบบออกแบบมาเฉพาะสำหรับพื้นที่หนึ่งประเทศ

**SRID (Spatial Reference System Identifier)** คือรหัสตัวเลขที่ใช้ระบุว่าพิกัดที่เก็บอยู่นั้นอ้างอิงระบบพิกัดแบบใด เป็นมาตรฐานที่ดูแลโดยองค์กร EPSG (European Petroleum Survey Group)

### WGS84 (SRID 4326) — มาตรฐานที่ใช้บ่อยที่สุด

**SRID 4326** หรือ **WGS84 (World Geodetic System 1984)** คือระบบพิกัดที่ใช้กันแพร่หลายที่สุดในโลก เพราะเป็นระบบที่ **GPS ใช้งานจริง** พิกัดในรูปแบบนี้เก็บเป็นคู่ **latitude (ละติจูด)** และ **longitude (ลองจิจูด)** มีหน่วยเป็นองศา (degrees)

- **Longitude** (ลองจิจูด) — แกนแนวนอน (แกน X) ค่าอยู่ระหว่าง -180 ถึง 180 (ประเทศไทยอยู่ประมาณ 97-106)
- **Latitude** (ละติจูด) — แกนแนวตั้ง (แกน Y) ค่าอยู่ระหว่าง -90 ถึง 90 (ประเทศไทยอยู่ประมาณ 5-21)

> **ข้อควรระวังที่พบบ่อยที่สุด:** ฟังก์ชัน `ST_MakePoint(x, y)` ของ PostGIS รับค่าตามลำดับ **(longitude, latitude)** ไม่ใช่ (latitude, longitude) ตามที่คนทั่วไปมักพูดถึง GPS ว่า "lat, long" — สลับลำดับผิดเป็นสาเหตุของบั๊กที่พบบ่อยมากในงาน PostGIS เพราะพิกัดจะไปโผล่อยู่คนละซีกโลก แต่บางครั้งอาจดูเหมือนถูกต้องโดยบังเอิญถ้าอยู่ใกล้เส้นศูนย์สูตรและ prime meridian

ตัวอย่างการตรวจสอบ SRID ของข้อมูลที่เก็บไว้:

```sql
SELECT store_name, ST_SRID(location) AS srid
FROM stores
LIMIT 3;
```

**ผลลัพธ์ตัวอย่าง:**

```
    store_name       | srid
----------------------+------
 สาขาสยามพารากอน       | 4326
 สาขาเซ็นทรัลเวิลด์     | 4326
 สาขาไอคอนสยาม         | 4326
(3 rows)
```

### ระบบพิกัดอื่นที่ควรรู้จัก

| SRID | ชื่อ | หน่วย | ใช้งานเมื่อ |
|------|------|-------|-------------|
| 4326 | WGS84 | องศา (degrees) | มาตรฐาน GPS ทั่วไป ใช้เก็บข้อมูลดิบ |
| 3857 | Web Mercator | เมตร | ใช้แสดงผลบนแผนที่เว็บ (Google Maps, Leaflet, OpenStreetMap tiles) |
| 32647 | UTM Zone 47N | เมตร | ใช้คำนวณระยะทาง/พื้นที่แบบ planar ในประเทศไทยตอนกลาง-ตะวันตก |
| 32648 | UTM Zone 48N | เมตร | ใช้คำนวณระยะทาง/พื้นที่แบบ planar ในประเทศไทยตอนตะวันออก |

การแปลงระหว่างระบบพิกัดทำได้ด้วยฟังก์ชัน `ST_Transform`:

```sql
SELECT
    store_name,
    ST_AsText(location::geometry) AS wgs84_point,
    ST_AsText(ST_Transform(location::geometry, 32647)) AS utm_point
FROM stores
WHERE store_name = 'สาขาสยามพารากอน';
```

**ผลลัพธ์ตัวอย่าง:**

```
    store_name      |            wgs84_point            |               utm_point
---------------------+------------------------------------+-----------------------------------------
 สาขาสยามพารากอน      | POINT(100.5347 13.7466)            | POINT(665926.xxxxx 1520273.xxxxx)
(1 row)
```

ในบทเรียนนี้เราจะยึด **SRID 4326 (WGS84)** เป็นหลักตลอด เพราะเป็นมาตรฐานที่ข้อมูล GPS ทุกแหล่ง (มือถือ, Google Maps API, ระบบนำทาง) ใช้ตรงกัน ทำให้ไม่ต้องแปลงไปมาเมื่อรับ-ส่งข้อมูลกับระบบภายนอก

---

## Step 753: Geometry Type เทียบกับ Geography Type

PostGIS มีชนิดข้อมูล (data type) หลักสำหรับเก็บข้อมูลเชิงพื้นที่อยู่ 2 แบบ คือ `geometry` และ `geography` ความแตกต่างนี้**สำคัญมาก**และเป็นจุดที่ผู้เริ่มต้นมักสับสน

### ความแตกต่างพื้นฐาน

| หัวข้อ | `geometry` | `geography` |
|--------|-----------|--------------|
| การคำนวณ | **Planar** (ระนาบแบน สมมติโลกแบนราบ) | **Spherical/Ellipsoidal** (ทรงกลม คำนึงถึงความโค้งของโลก) |
| ความเร็ว | เร็วกว่า | ช้ากว่าเล็กน้อย (คำนวณซับซ้อนกว่า) |
| ความแม่นยำสำหรับพิกัดโลกจริง | ผิดพลาดมากขึ้นเมื่อระยะทางไกล หรืออยู่ใกล้ขั้วโลก | แม่นยำสำหรับระยะทางจริงบนพื้นผิวโลก |
| หน่วยของระยะทาง (ผลลัพธ์ `ST_Distance`) | หน่วยตาม SRID (เช่น องศา หากใช้ 4326) | **เมตร** เสมอ (ไม่ว่า SRID จะเป็นอะไร) |
| ใช้กับพื้นที่ขนาดเล็ก/local (เมือง, ผังเมือง) | เหมาะสม โดยเฉพาะเมื่อ `ST_Transform` เป็น projected CRS เช่น UTM | ใช้ได้เช่นกัน แต่ overhead มากกว่าที่จำเป็น |
| ใช้กับข้อมูลกระจายทั่วโลก (global, GPS) | ต้องระวังเรื่องความโค้งของโลก | **เหมาะสมที่สุด** |
| ฟังก์ชันที่รองรับ | รองรับฟังก์ชันครบทุกตัวของ PostGIS | รองรับฟังก์ชันหลัก ๆ (เพิ่มขึ้นเรื่อย ๆ ในเวอร์ชันใหม่) |

### ตัวอย่างที่แสดงความแตกต่างชัดเจน

ลองคำนวณระยะทางระหว่างจุดสองจุดในสองแบบ:

```sql
-- สร้างจุดสองจุดแบบ geometry (SRID 4326 คือหน่วยองศา)
WITH points AS (
    SELECT
        ST_SetSRID(ST_MakePoint(100.5347, 13.7466), 4326) AS geom_a,  -- สยามพารากอน
        ST_SetSRID(ST_MakePoint(100.5602, 13.7373), 4326) AS geom_b   -- เทอร์มินอล 21
)
SELECT
    ST_Distance(geom_a, geom_b) AS distance_geometry_degrees,
    ST_Distance(geom_a::geography, geom_b::geography) AS distance_geography_meters
FROM points;
```

**ผลลัพธ์ตัวอย่าง:**

```
 distance_geometry_degrees | distance_geography_meters
----------------------------+---------------------------
        0.02861234567890    |          3105.42
(1 row)
```

จะเห็นว่า:
- คำนวณด้วย `geometry` โดยตรง (ไม่แปลง projection) จะได้ค่าเป็น**หน่วยองศา** ซึ่งไม่มีความหมายในเชิงระยะทางจริง เว้นแต่จะนำไปแปลงอีกที
- คำนวณด้วย `geography` (แคสต์ `::geography`) จะได้ค่าเป็น**เมตร**ทันที ซึ่งตีความได้ทันทีว่าห่างกันประมาณ 3.1 กิโลเมตร

### เมื่อไหร่ควรใช้ `geometry` เมื่อไหร่ควรใช้ `geography`

**ใช้ `geography` เมื่อ:**
- ข้อมูลเป็นพิกัด GPS จริง (SRID 4326) และต้องการคำนวณระยะทาง/พื้นที่ที่ถูกต้องตามความเป็นจริงบนผิวโลกทันที โดยไม่ต้องกังวลเรื่อง projection
- แอปพลิเคชันกระจายอยู่ในพื้นที่กว้าง (ข้ามจังหวัด ข้ามประเทศ) เช่น ระบบหาสาขาใกล้ที่สุดของธุรกิจที่มีสาขาทั่วประเทศ
- ต้องการความง่ายในการพัฒนา ไม่ต้องคิดเรื่อง `ST_Transform` เอง (นี่คือเหตุผลที่บทนี้เลือกใช้ `GEOGRAPHY(POINT, 4326)` เป็นชนิดข้อมูลหลักสำหรับ `stores` และ `delivery_addresses`)

**ใช้ `geometry` เมื่อ:**
- ต้องการประสิทธิภาพสูงสุด และพื้นที่ทำงานมีขอบเขตจำกัด (เช่น ผังเมืองเดียว, พื้นที่ก่อสร้างเดียว) สามารถเลือก projected CRS ที่เหมาะสม (เช่น UTM) แล้วคำนวณแบบ planar ได้แม่นยำและเร็ว
- ต้องใช้ฟังก์ชันขั้นสูงที่ยังไม่รองรับ `geography` (พบน้อยมากในเวอร์ชันปัจจุบัน)
- งาน GIS วิเคราะห์เชิงลึก เช่น การซ้อนทับ (overlay analysis), การสร้าง buffer ที่ซับซ้อน

ในบทเรียนนี้ เราเลือกใช้ **`GEOGRAPHY(POINT, 4326)`** สำหรับ `stores` และ `delivery_addresses` เพราะเหมาะกับโจทย์ "หาสาขาใกล้ลูกค้าที่สุดในกรุงเทพฯ ทั้งเมือง" มากที่สุด — ได้ผลลัพธ์ระยะทางเป็นเมตรทันที ไม่ต้องแปลง projection เอง

---

## Step 754: POINT Type — เก็บพิกัดตำแหน่ง

### โครงสร้างของ POINT

`POINT` เป็นชนิดข้อมูลเชิงพื้นที่ที่ง่ายที่สุด เก็บพิกัดเพียงจุดเดียว (x, y) เหมาะกับข้อมูลอย่างตำแหน่งร้านค้า ตำแหน่งลูกค้า ตำแหน่ง GPS ของรถขนส่ง

### ฟังก์ชันสำคัญในการสร้างจุด

**`ST_MakePoint(x, y)`** — สร้างจุดจากค่า x (longitude) และ y (latitude) แต่**ยังไม่มี SRID กำกับ**

```sql
SELECT ST_MakePoint(100.5347, 13.7466);
```

**ผลลัพธ์:**

```
                st_makepoint
---------------------------------------
 0101000000...  (แสดงเป็น hex ของ WKB)
```

จะเห็นว่าค่าที่ได้เป็น binary format (WKB) ซึ่งอ่านไม่รู้เรื่องด้วยตาเปล่า ต้องใช้ `ST_AsText` ในการอ่าน (จะกล่าวถึงใน Step 759)

**`ST_SetSRID(geom, srid)`** — กำหนด SRID ให้กับ geometry ที่สร้างไว้ (จำเป็นเสมอ เพราะถ้าไม่กำหนด SRID จะเป็น 0 ซึ่งหมายถึง "ไม่ทราบระบบพิกัด")

รูปแบบมาตรฐานที่ใช้สร้างจุดพิกัดคือการเชื่อมสองฟังก์ชันนี้เข้าด้วยกัน:

```sql
SELECT ST_AsText(
    ST_SetSRID(ST_MakePoint(100.5347, 13.7466), 4326)
) AS point_wkt;
```

**ผลลัพธ์:**

```
        point_wkt
--------------------------
 POINT(100.5347 13.7466)
(1 row)
```

### การ Insert ข้อมูลพิกัดจริง

เมื่อคอลัมน์เป็นชนิด `GEOGRAPHY(POINT, 4326)` เราต้องแคสต์ผลลัพธ์จาก `geometry` เป็น `geography` ด้วย `::geography`:

```sql
INSERT INTO stores (store_name, address, location)
VALUES (
    'สาขาเซ็นทรัลลาดพร้าว',
    '1697 ถนนพหลโยธิน แขวงจตุจักร เขตจตุจักร กรุงเทพฯ 10900',
    ST_SetSRID(ST_MakePoint(100.5606, 13.8154), 4326)::geography
);
```

### วิธีอื่นในการสร้าง POINT — ผ่าน WKT (Well-Known Text)

อีกวิธีที่นิยมใช้ (โดยเฉพาะเมื่อรับข้อมูลจากระบบภายนอกที่ส่งมาเป็นข้อความ) คือใช้ `ST_GeomFromText`:

```sql
SELECT ST_AsText(
    ST_GeomFromText('POINT(100.5347 13.7466)', 4326)
);
```

หรือใช้ literal แบบสั้น `SRID=4326;POINT(...)` ผ่าน cast โดยตรง:

```sql
SELECT 'SRID=4326;POINT(100.5347 13.7466)'::geography;
```

ทั้งสองวิธีนี้ให้ผลลัพธ์เทียบเท่ากับการใช้ `ST_MakePoint` + `ST_SetSRID`

### การดึงค่า longitude/latitude กลับออกมา

เมื่อมีข้อมูล `geography` แล้ว หากต้องการดึงค่า longitude/latitude แยกออกมา (เช่น เพื่อส่งให้ frontend แสดงผล) ใช้ `ST_X` และ `ST_Y` (ต้องแคสต์กลับเป็น `geometry` ก่อน):

```sql
SELECT
    store_name,
    ST_X(location::geometry) AS longitude,
    ST_Y(location::geometry) AS latitude
FROM stores
ORDER BY store_id;
```

**ผลลัพธ์ตัวอย่าง:**

```
       store_name        | longitude |  latitude
--------------------------+-----------+------------
 สาขาสยามพารากอน           | 100.5347  | 13.7466
 สาขาเซ็นทรัลเวิลด์         | 100.5393  | 13.7466
 สาขาไอคอนสยาม             | 100.5099  | 13.7262
 สาขาเทอร์มินอล 21 อโศก     | 100.5602  | 13.7373
 สาขาเอ็มบีเค เซ็นเตอร์      | 100.5300  | 13.7447
 สาขาเซ็นทรัลลาดพร้าว       | 100.5606  | 13.8154
(6 rows)
```

### สร้าง Spatial Index เพื่อประสิทธิภาพ

เมื่อข้อมูลมีจำนวนมาก การค้นหาเชิงพื้นที่ต้องพึ่งพา **GiST index** เพื่อให้ query เร็ว:

```sql
CREATE INDEX idx_stores_location ON stores USING GIST (location);
CREATE INDEX idx_delivery_addresses_location ON delivery_addresses USING GIST (location);
```

> ควรสร้าง index นี้เสมอสำหรับคอลัมน์ `geometry`/`geography` ที่จะถูกใช้ในเงื่อนไข `WHERE` ที่เกี่ยวกับระยะทางหรือความสัมพันธ์เชิงพื้นที่ (`ST_DWithin`, `ST_Contains`, `ST_Intersects` เป็นต้น) มิฉะนั้น PostgreSQL จะต้อง sequential scan ตรวจสอบทุกแถว ซึ่งช้ามากเมื่อข้อมูลมีขนาดใหญ่

---

## Step 755: LINESTRING และ POLYGON — เส้นทางและพื้นที่

นอกจาก `POINT` แล้ว PostGIS ยังรองรับชนิดข้อมูลอื่นที่ซับซ้อนกว่า สองแบบที่ใช้บ่อยที่สุดคือ `LINESTRING` (เส้น) และ `POLYGON` (พื้นที่ปิด)

### LINESTRING — เก็บเส้นทาง

`LINESTRING` เก็บลำดับของจุดที่เชื่อมต่อกันเป็นเส้น เหมาะกับการเก็บเส้นทางการเดินรถขนส่ง เส้นทางการเดินทาง หรือถนน

```sql
-- ตัวอย่างเส้นทางจัดส่งจากสาขาสยามพารากอนไปยังลูกค้าที่ทองหล่อ (จำลองเป็นเส้นตรง 3 จุด)
SELECT ST_AsText(
    ST_SetSRID(
        ST_MakeLine(ARRAY[
            ST_MakePoint(100.5347, 13.7466),  -- จุดเริ่มต้น: สยามพารากอน
            ST_MakePoint(100.5500, 13.7400),  -- จุดผ่านทาง
            ST_MakePoint(100.5789, 13.7320)   -- จุดหมาย: ทองหล่อ
        ]),
        4326
    )
) AS delivery_route;
```

**ผลลัพธ์ตัวอย่าง:**

```
                              delivery_route
---------------------------------------------------------------------------
 LINESTRING(100.5347 13.7466,100.55 13.74,100.5789 13.732)
(1 row)
```

การคำนวณความยาวของเส้นทางใช้ `ST_Length` (สำหรับ `geography` จะได้หน่วยเป็นเมตร):

```sql
SELECT ST_Length(
    ST_SetSRID(
        ST_MakeLine(ARRAY[
            ST_MakePoint(100.5347, 13.7466),
            ST_MakePoint(100.5500, 13.7400),
            ST_MakePoint(100.5789, 13.7320)
        ]),
        4326
    )::geography
) AS route_length_meters;
```

**ผลลัพธ์ตัวอย่าง:**

```
 route_length_meters
----------------------
           5389.71
(1 row)
```

### POLYGON — เก็บพื้นที่ (เช่น เขตการจัดส่ง)

`POLYGON` เก็บพื้นที่ปิด กำหนดโดยลำดับจุดที่ล้อมรอบพื้นที่ **จุดแรกและจุดสุดท้ายต้องเป็นจุดเดียวกัน** (เพื่อปิดรูปทรง)

ลองสร้างเขตการจัดส่งครอบคลุมพื้นที่รอบสาขาสยามพารากอนแบบง่าย ๆ (สี่เหลี่ยมคร่าว ๆ):

```sql
INSERT INTO delivery_zones (zone_name, coverage)
VALUES (
    'เขตจัดส่งปทุมวัน-บางรัก',
    ST_SetSRID(
        ST_GeomFromText(
            'POLYGON((
                100.5200 13.7150,
                100.5600 13.7150,
                100.5600 13.7550,
                100.5200 13.7550,
                100.5200 13.7150
            ))'
        ),
        4326
    )::geography
);
```

ตรวจสอบข้อมูลที่เก็บ:

```sql
SELECT zone_name, ST_AsText(coverage::geometry) AS polygon_wkt
FROM delivery_zones;
```

**ผลลัพธ์ตัวอย่าง:**

```
        zone_name           |                                     polygon_wkt
------------------------------+-------------------------------------------------------------------------------------
 เขตจัดส่งปทุมวัน-บางรัก        | POLYGON((100.52 13.715,100.56 13.715,100.56 13.755,100.52 13.755,100.52 13.715))
(1 row)
```

คำนวณพื้นที่ของเขตจัดส่งนี้ (หน่วยตารางเมตร เมื่อใช้ `geography`):

```sql
SELECT
    zone_name,
    ST_Area(coverage) AS area_sq_meters,
    ROUND((ST_Area(coverage) / 1000000)::numeric, 2) AS area_sq_km
FROM delivery_zones;
```

**ผลลัพธ์ตัวอย่าง:**

```
        zone_name           | area_sq_meters | area_sq_km
------------------------------+----------------+------------
 เขตจัดส่งปทุมวัน-บางรัก        |    19842731.5  |      19.84
(1 row)
```

เพิ่มเขตจัดส่งอีกโซนหนึ่งครอบคลุมพื้นที่สุขุมวิท-วัฒนา เพื่อใช้ในแบบฝึกหัดถัดไป:

```sql
INSERT INTO delivery_zones (zone_name, coverage)
VALUES (
    'เขตจัดส่งสุขุมวิท-วัฒนา',
    ST_SetSRID(
        ST_GeomFromText(
            'POLYGON((
                100.5400 13.7100,
                100.5900 13.7100,
                100.5900 13.7500,
                100.5400 13.7500,
                100.5400 13.7100
            ))'
        ),
        4326
    )::geography
);

-- สร้าง spatial index ให้ตาราง delivery_zones ด้วย
CREATE INDEX idx_delivery_zones_coverage ON delivery_zones USING GIST (coverage);
```

---

## Step 756: ST_Distance — คำนวณระยะทางระหว่างจุดสองจุด

`ST_Distance(geog_a, geog_b)` คือฟังก์ชันหลักในการคำนวณระยะทางระหว่างจุดสองจุด เมื่อใช้กับ `geography` ผลลัพธ์จะเป็น **เมตรเสมอ**

### ตัวอย่างพื้นฐาน: ระยะทางระหว่างสาขาสองสาขา

```sql
SELECT
    a.store_name AS store_a,
    b.store_name AS store_b,
    ROUND(ST_Distance(a.location, b.location)::numeric, 2) AS distance_meters
FROM stores a, stores b
WHERE a.store_name = 'สาขาสยามพารากอน'
  AND b.store_name = 'สาขาไอคอนสยาม';
```

**ผลลัพธ์ตัวอย่าง:**

```
      store_a       |    store_b     | distance_meters
----------------------+----------------+------------------
 สาขาสยามพารากอน       | สาขาไอคอนสยาม   |          2887.43
(1 row)
```

### โจทย์จริง: หาสาขาที่ใกล้ลูกค้าที่สุด

นี่คือหนึ่งใน use case ที่พบบ่อยที่สุดของ PostGIS ในระบบ e-commerce — เมื่อลูกค้าสั่งซื้อสินค้าและต้องการรับที่สาขา (click & collect) หรือระบบต้องการ route คำสั่งซื้อไปยังสาขาที่ใกล้ที่สุดเพื่อลดเวลาจัดส่ง

```sql
-- หาสาขาที่ใกล้ลูกค้า customer_id = 104 (คุณนภา สว่างใจ แถวทองหล่อ) มากที่สุด
SELECT
    s.store_name,
    s.address,
    ROUND(ST_Distance(s.location, d.location)::numeric, 2) AS distance_meters
FROM stores s
CROSS JOIN delivery_addresses d
WHERE d.customer_id = 104
ORDER BY ST_Distance(s.location, d.location)
LIMIT 3;
```

**ผลลัพธ์ตัวอย่าง:**

```
       store_name        |                        address                        | distance_meters
--------------------------+---------------------------------------------------------+------------------
 สาขาเทอร์มินอล 21 อโศก    | 88 ถนนสุขุมวิท แขวงคลองเตยเหนือ เขตวัฒนา...              |          2589.11
 สาขาสยามพารากอน           | 991 ถนนพระราม 1 แขวงปทุมวัน...                          |          4741.02
 สาขาเซ็นทรัลเวิลด์         | 4-4/5 ถนนราชดำริ แขวงปทุมวัน...                         |          4972.85
(3 rows)
```

จะเห็นว่าสาขาเทอร์มินอล 21 อโศก ใกล้ลูกค้าคนนี้ที่สุด (2.59 กม.) ซึ่งสมเหตุสมผลเพราะทั้งคู่อยู่แถวสุขุมวิท

### ใช้ Operator `<->` เพื่อประสิทธิภาพที่ดีกว่า (KNN Search)

แทนที่จะคำนวณ `ST_Distance` แล้วเรียง `ORDER BY` (ซึ่งต้องคำนวณระยะทางทุกแถวก่อนเรียง) PostGIS มี **KNN (K-Nearest Neighbor) operator** คือ `<->` ที่สามารถใช้ร่วมกับ GiST index เพื่อค้นหาแบบ "ใกล้ที่สุด N อันดับ" ได้เร็วกว่ามาก โดยเฉพาะเมื่อข้อมูลมีจำนวนมาก:

```sql
SELECT
    s.store_name,
    s.address,
    ROUND((s.location <-> d.location)::numeric, 2) AS distance_meters
FROM stores s
CROSS JOIN delivery_addresses d
WHERE d.customer_id = 104
ORDER BY s.location <-> d.location
LIMIT 3;
```

**ผลลัพธ์:** เหมือนกับตัวอย่างก่อนหน้า แต่เมื่อมีสาขาเป็นพันเป็นหมื่นแห่ง การใช้ `<->` ร่วมกับ GiST index จะเร็วกว่า `ORDER BY ST_Distance(...)` มาก เพราะ index สามารถนำทางการค้นหาแบบ nearest-neighbor ได้โดยตรง โดยไม่ต้องคำนวณระยะทางของทุกแถวก่อน

> **ข้อสังเกต:** สำหรับ query ที่ใช้ `ORDER BY ... LIMIT n` แบบหาที่ใกล้ที่สุด ควรใช้ operator `<->` เสมอเมื่อมี GiST index อยู่ เพราะ PostgreSQL planner จะเลือกใช้ index scan แบบ nearest-neighbor แทน sequential scan ทำให้ประหยัด I/O อย่างมาก

---

## Step 757: ST_DWithin — หาจุดในรัศมีที่กำหนด

`ST_DWithin(geog_a, geog_b, distance_in_meters)` คือฟังก์ชันตรวจสอบว่าจุดสองจุด (หรือ geometry สองชิ้น) อยู่ห่างกันไม่เกินระยะที่กำหนดหรือไม่ คืนค่าเป็น `boolean` (`TRUE`/`FALSE`)

### ทำไมต้องใช้ `ST_DWithin` แทน `ST_Distance(...) < ...`

ในทางทฤษฎี เราสามารถเขียน `WHERE ST_Distance(a, b) < 5000` เพื่อหาจุดที่อยู่ในรัศมี 5 กม. ได้เหมือนกัน แต่ `ST_DWithin` **ดีกว่ามาก**ในทางปฏิบัติ เพราะ:

1. **ใช้ spatial index ได้** — `ST_DWithin` ถูกออกแบบมาให้ PostgreSQL planner ใช้ GiST index กรองข้อมูลตั้งแต่ต้น (index-assisted) ในขณะที่ `ST_Distance(a,b) < X` ต้องคำนวณระยะทางที่แม่นยำของทุกแถวก่อนแล้วค่อยกรอง (ช้ากว่ามากเมื่อข้อมูลใหญ่)
2. **สั้นและอ่านง่ายกว่า**

### โจทย์จริง: หาสาขาทั้งหมดที่อยู่ในระยะ 5 กม. จากลูกค้า

```sql
-- ลูกค้า customer_id = 101 (คุณสมชาย ใจดี แถวสุขุมวิท 24)
SELECT
    s.store_name,
    s.address,
    ROUND(ST_Distance(s.location, d.location)::numeric, 2) AS distance_meters
FROM stores s
CROSS JOIN delivery_addresses d
WHERE d.customer_id = 101
  AND ST_DWithin(s.location, d.location, 5000)   -- รัศมี 5,000 เมตร = 5 กม.
ORDER BY distance_meters;
```

**ผลลัพธ์ตัวอย่าง:**

```
       store_name         |                    address                     | distance_meters
---------------------------+-------------------------------------------------+------------------
 สาขาเทอร์มินอล 21 อโศก     | 88 ถนนสุขุมวิท แขวงคลองเตยเหนือ...               |          2148.67
 สาขาสยามพารากอน            | 991 ถนนพระราม 1 แขวงปทุมวัน...                  |          4523.90
(2 rows)
```

จะเห็นว่ามีเพียง 2 สาขาที่อยู่ในระยะ 5 กม. จากลูกค้ารายนี้ (สาขาอื่น ๆ อยู่ไกลเกิน 5 กม. จึงถูกกรองออก)

### โจทย์ขยาย: นับจำนวนลูกค้าที่มีสาขาให้บริการภายใน 5 กม. (สำหรับวางแผนขยายสาขา)

```sql
SELECT
    d.customer_id,
    d.recipient_name,
    COUNT(s.store_id) AS stores_within_5km
FROM delivery_addresses d
LEFT JOIN stores s
    ON ST_DWithin(s.location, d.location, 5000)
GROUP BY d.customer_id, d.recipient_name
ORDER BY stores_within_5km ASC;
```

**ผลลัพธ์ตัวอย่าง:**

```
 customer_id | recipient_name          | stores_within_5km
-------------+--------------------------+--------------------
         103 | คุณวิชัย มั่นคง          |                  0
         105 | คุณอนันต์ พูนทรัพย์      |                  1
         104 | คุณนภา สว่างใจ           |                  2
         101 | คุณสมชาย ใจดี            |                  2
         102 | คุณสมหญิง รักเรียน       |                  4
(5 rows)
```

ผลลัพธ์นี้มีประโยชน์มากในเชิงธุรกิจ — จะเห็นว่าลูกค้า `customer_id = 103` (แถวสายไหม) ไม่มีสาขาใดอยู่ในระยะ 5 กม.เลย ซึ่งเป็นข้อมูลสำคัญสำหรับทีมวางแผนขยายสาขาใหม่

---

## Step 758: ST_Contains, ST_Intersects — ตรวจสอบพื้นที่

เมื่อมี `POLYGON` แทนขอบเขตพื้นที่ (เช่น เขตการจัดส่ง) เราจำเป็นต้องตรวจสอบว่าจุดหนึ่งอยู่ **ภายใน** พื้นที่นั้นหรือไม่ ใช้สองฟังก์ชันหลักคือ `ST_Contains` และ `ST_Intersects`

### ST_Contains(polygon, point) — polygon "บรรจุ" จุดนี้ไว้หรือไม่

```sql
SELECT
    dz.zone_name,
    d.recipient_name,
    ST_Contains(dz.coverage::geometry, d.location::geometry) AS is_inside
FROM delivery_zones dz
CROSS JOIN delivery_addresses d
WHERE dz.zone_name = 'เขตจัดส่งปทุมวัน-บางรัก'
ORDER BY d.customer_id;
```

**ผลลัพธ์ตัวอย่าง:**

```
          zone_name           | recipient_name          | is_inside
--------------------------------+--------------------------+-----------
 เขตจัดส่งปทุมวัน-บางรัก        | คุณสมชาย ใจดี             | f
 เขตจัดส่งปทุมวัน-บางรัก        | คุณสมหญิง รักเรียน        | t
 เขตจัดส่งปทุมวัน-บางรัก        | คุณวิชัย มั่นคง           | f
 เขตจัดส่งปทุมวัน-บางรัก        | คุณนภา สว่างใจ            | f
 เขตจัดส่งปทุมวัน-บางรัก        | คุณอนันต์ พูนทรัพย์       | f
(5 rows)
```

> **หมายเหตุทางเทคนิค:** ในเวอร์ชันปัจจุบันของ PostGIS ฟังก์ชัน `ST_Contains` ยังรองรับเฉพาะชนิดข้อมูล `geometry` เท่านั้น (ไม่รองรับ `geography` โดยตรง) จึงต้อง cast ทั้งสองฝั่งเป็น `::geometry` ก่อนเรียกใช้เสมอ ต่างจาก `ST_Distance` และ `ST_DWithin` ที่รองรับ `geography` โดยตรง

### ST_Intersects — ตรวจสอบว่าสองรูปทรง "ตัดกัน" หรือไม่ (รวมถึงกรณีสัมผัสขอบ)

`ST_Intersects` มีความหมายกว้างกว่า `ST_Contains` เล็กน้อย — คืนค่า `TRUE` เมื่อรูปทรงสองชิ้นมีจุดร่วมกันไม่ว่ากรณีใด (ซ้อนทับ สัมผัสขอบ อยู่ภายใน) ในขณะที่ `ST_Contains` ต้องการให้จุดนั้นอยู่ **ภายใน** พื้นที่อย่างเคร่งครัด (ไม่นับกรณีอยู่บนขอบพอดี)

ในทางปฏิบัติ สำหรับ "จุดอยู่ในพื้นที่หรือไม่" ทั้งสองฟังก์ชันมักให้ผลเหมือนกัน แต่ `ST_Intersects` นิยมใช้มากกว่าเมื่อเปรียบเทียบ polygon กับ polygon (เช่น ตรวจสอบว่าเขตจัดส่งสองเขตทับซ้อนกันหรือไม่):

```sql
-- ตรวจสอบว่าเขตจัดส่งสองเขตทับซ้อนกันหรือไม่ (สำคัญมากสำหรับวางแผนไม่ให้เขตซ้อนกัน)
SELECT
    z1.zone_name AS zone_a,
    z2.zone_name AS zone_b,
    ST_Intersects(z1.coverage::geometry, z2.coverage::geometry) AS zones_overlap
FROM delivery_zones z1
JOIN delivery_zones z2 ON z1.zone_id < z2.zone_id;
```

**ผลลัพธ์ตัวอย่าง:**

```
           zone_a             |          zone_b            | zones_overlap
--------------------------------+------------------------------+----------------
 เขตจัดส่งปทุมวัน-บางรัก        | เขตจัดส่งสุขุมวิท-วัฒนา       | t
(1 row)
```

ผลลัพธ์แสดงว่าสองเขตนี้ทับซ้อนกันอยู่บางส่วน (`t` = true) ซึ่งเป็นข้อมูลสำคัญที่ทีมปฏิบัติการต้องรู้ เพื่อกำหนดกฎว่าเมื่อที่อยู่ตกอยู่ในพื้นที่ทับซ้อน จะให้สาขาใดรับผิดชอบก่อน

### ใช้ ST_Intersects ตรวจสอบว่าที่อยู่จัดส่งอยู่ในเขตให้บริการหรือไม่ (โจทย์หลักของธุรกิจ)

```sql
-- สำหรับที่อยู่ใหม่ที่ลูกค้าเพิ่งกรอก ตรวจสอบว่าอยู่ในเขตให้บริการจัดส่งใดบ้าง
SELECT
    d.recipient_name,
    dz.zone_name
FROM delivery_addresses d
JOIN delivery_zones dz
    ON ST_Intersects(dz.coverage::geometry, d.location::geometry)
ORDER BY d.customer_id, dz.zone_name;
```

**ผลลัพธ์ตัวอย่าง:**

```
     recipient_name         |            zone_name
------------------------------+------------------------------------
 คุณสมชาย ใจดี                | เขตจัดส่งสุขุมวิท-วัฒนา
 คุณสมหญิง รักเรียน           | เขตจัดส่งปทุมวัน-บางรัก
 คุณนภา สว่างใจ               | เขตจัดส่งสุขุมวิท-วัฒนา
(3 rows)
```

จะสังเกตว่า `คุณวิชัย มั่นคง` (แถวสายไหม) และ `คุณอนันต์ พูนทรัพย์` (แถวรัชดาภิเษก) ไม่ปรากฏในผลลัพธ์เลย เพราะที่อยู่ของทั้งสองคนอยู่นอกเขตจัดส่งทั้งสองเขตที่เรากำหนดไว้ — ระบบสามารถใช้ข้อมูลนี้แจ้งลูกค้าได้ทันทีว่า "ที่อยู่นี้อยู่นอกพื้นที่ให้บริการจัดส่ง"

---

## Step 759: การแสดงผลพิกัดในรูปแบบต่างๆ

เมื่อต้องส่งข้อมูลพิกัดไปแสดงผลบนแผนที่ (เช่น หน้าเว็บที่ใช้ Leaflet, Mapbox GL JS, Google Maps JavaScript API) หรือส่งผ่าน REST API เราต้องแปลงข้อมูล geometry/geography จากรูปแบบ binary ภายในของ PostgreSQL ให้เป็นรูปแบบข้อความมาตรฐาน

### ST_AsText — แปลงเป็น WKT (Well-Known Text)

```sql
SELECT
    store_name,
    ST_AsText(location::geometry) AS location_wkt
FROM stores
ORDER BY store_id
LIMIT 3;
```

**ผลลัพธ์ตัวอย่าง:**

```
       store_name       |         location_wkt
--------------------------+--------------------------------
 สาขาสยามพารากอน           | POINT(100.5347 13.7466)
 สาขาเซ็นทรัลเวิลด์         | POINT(100.5393 13.7466)
 สาขาไอคอนสยาม             | POINT(100.5099 13.7262)
(3 rows)
```

WKT เป็นรูปแบบที่มนุษย์อ่านง่าย เหมาะสำหรับ debug หรือ log แต่ **ไม่ใช่รูปแบบที่นิยมส่งให้ frontend ใช้งานโดยตรง** เพราะ frontend ส่วนใหญ่ (โดยเฉพาะ JavaScript mapping library) ต้องการรูปแบบ **GeoJSON**

### ST_AsGeoJSON — แปลงเป็น GeoJSON (มาตรฐานสำหรับส่งขึ้นแผนที่)

```sql
SELECT
    store_name,
    ST_AsGeoJSON(location::geometry) AS location_geojson
FROM stores
WHERE store_name = 'สาขาสยามพารากอน';
```

**ผลลัพธ์ตัวอย่าง:**

```
    store_name      |                     location_geojson
----------------------+------------------------------------------------------------
 สาขาสยามพารากอน       | {"type":"Point","coordinates":[100.5347,13.7466]}
(1 row)
```

รูปแบบนี้สามารถนำไป `JSON.parse()` แล้วส่งตรงให้ library อย่าง Leaflet ใช้งานได้ทันที เช่น:

```javascript
// ตัวอย่างการใช้งานฝั่ง frontend (JavaScript)
const geojson = JSON.parse(row.location_geojson);
L.geoJSON(geojson).addTo(map);
```

### สร้าง GeoJSON แบบเต็มรูปแบบ (FeatureCollection) ด้วย SQL โดยตรง

ในการทำงานจริง เรามักต้องการส่งข้อมูลหลายแถวพร้อมกันในรูปแบบ GeoJSON `FeatureCollection` เพื่อให้ frontend วาดหมุดทั้งหมดบนแผนที่ในครั้งเดียว สามารถประกอบสร้างได้ด้วย `json_build_object` ร่วมกับ `ST_AsGeoJSON`:

```sql
SELECT json_build_object(
    'type', 'FeatureCollection',
    'features', json_agg(
        json_build_object(
            'type', 'Feature',
            'geometry', ST_AsGeoJSON(location::geometry)::json,
            'properties', json_build_object(
                'store_id', store_id,
                'store_name', store_name,
                'address', address
            )
        )
    )
) AS stores_geojson
FROM stores;
```

**ผลลัพธ์ตัวอย่าง (ย่อ):**

```json
{
  "type": "FeatureCollection",
  "features": [
    {
      "type": "Feature",
      "geometry": {"type": "Point", "coordinates": [100.5347, 13.7466]},
      "properties": {"store_id": 1, "store_name": "สาขาสยามพารากอน", "address": "991 ถนนพระราม 1..."}
    },
    {
      "type": "Feature",
      "geometry": {"type": "Point", "coordinates": [100.5393, 13.7466]},
      "properties": {"store_id": 2, "store_name": "สาขาเซ็นทรัลเวิลด์", "address": "4-4/5 ถนนราชดำริ..."}
    }
  ]
}
```

ผลลัพธ์นี้เป็น GeoJSON `FeatureCollection` มาตรฐานที่ frontend framework ส่วนใหญ่รับได้ทันที (Leaflet: `L.geoJSON(data).addTo(map)`, Mapbox: `map.addSource('stores', {type: 'geojson', data: data})`)

### รูปแบบอื่นที่ควรรู้จัก

| ฟังก์ชัน | รูปแบบผลลัพธ์ | ใช้งานเมื่อ |
|----------|---------------|--------------|
| `ST_AsText` | WKT (ข้อความอ่านง่าย) | debug, log, แสดงผลใน admin panel |
| `ST_AsGeoJSON` | GeoJSON | ส่งให้ web map library (Leaflet, Mapbox) |
| `ST_AsBinary` / `ST_AsEWKB` | WKB (binary) | ส่งข้อมูลระหว่างระบบที่รองรับ binary format โดยตรง |
| `ST_AsKML` | KML | ใช้กับ Google Earth หรือระบบที่รองรับ KML |
| `ST_AsLatLonText` | ข้อความพิกัดแบบองศา-ลิปดา-ฟิลิปดา (DMS) | แสดงผลรูปแบบพิกัดดั้งเดิมสำหรับงานสำรวจ |

```sql
-- ตัวอย่าง ST_AsLatLonText
SELECT ST_AsLatLonText(location::geometry) AS dms_format
FROM stores
WHERE store_name = 'สาขาสยามพารากอน';
```

**ผลลัพธ์ตัวอย่าง:**

```
              dms_format
----------------------------------------
 13°44'47.760"N 100°32'4.920"E
(1 row)
```

---

## Step 760: แบบฝึกหัดรวม — ระบบหาสาขาใกล้ที่สุดและตรวจสอบพื้นที่ให้บริการจัดส่ง

มาถึงจุดนี้ เราได้เรียนรู้พื้นฐานของ PostGIS ครบทุกส่วนที่จำเป็นสำหรับการสร้างระบบ geospatial จริงในธุรกิจ e-commerce แล้ว ในหัวข้อนี้เราจะรวมทุกอย่างเข้าด้วยกันเป็นฟังก์ชันและ view ที่ใช้งานได้จริง

### 1. สร้างฟังก์ชันหาสาขาใกล้ที่สุด N อันดับ

```sql
CREATE OR REPLACE FUNCTION find_nearest_stores(
    p_longitude DOUBLE PRECISION,
    p_latitude DOUBLE PRECISION,
    p_limit INTEGER DEFAULT 3
)
RETURNS TABLE (
    store_id INTEGER,
    store_name VARCHAR,
    address TEXT,
    distance_meters NUMERIC
) AS $$
DECLARE
    v_point GEOGRAPHY;
BEGIN
    v_point := ST_SetSRID(ST_MakePoint(p_longitude, p_latitude), 4326)::geography;

    RETURN QUERY
    SELECT
        s.store_id,
        s.store_name,
        s.address,
        ROUND(ST_Distance(s.location, v_point)::numeric, 2) AS distance_meters
    FROM stores s
    ORDER BY s.location <-> v_point
    LIMIT p_limit;
END;
$$ LANGUAGE plpgsql STABLE;
```

ทดสอบใช้งาน (สมมติลูกค้าอยู่แถวสีลม พิกัดประมาณ 100.5290, 13.7230):

```sql
SELECT * FROM find_nearest_stores(100.5290, 13.7230, 3);
```

**ผลลัพธ์ตัวอย่าง:**

```
 store_id |       store_name        |                     address                     | distance_meters
----------+---------------------------+---------------------------------------------------+------------------
        1 | สาขาสยามพารากอน           | 991 ถนนพระราม 1 แขวงปทุมวัน...                   |          2698.34
        5 | สาขาเอ็มบีเค เซ็นเตอร์     | 444 ถนนพญาไท แขวงวังใหม่...                       |          2810.55
        2 | สาขาเซ็นทรัลเวิลด์         | 4-4/5 ถนนราชดำริ แขวงปทุมวัน...                  |          3195.12
(3 rows)
```

### 2. สร้างฟังก์ชันตรวจสอบพื้นที่ให้บริการจัดส่ง

```sql
CREATE OR REPLACE FUNCTION check_delivery_coverage(
    p_longitude DOUBLE PRECISION,
    p_latitude DOUBLE PRECISION
)
RETURNS TABLE (
    zone_id INTEGER,
    zone_name VARCHAR,
    is_covered BOOLEAN
) AS $$
DECLARE
    v_point GEOGRAPHY;
BEGIN
    v_point := ST_SetSRID(ST_MakePoint(p_longitude, p_latitude), 4326)::geography;

    RETURN QUERY
    SELECT
        dz.zone_id,
        dz.zone_name,
        ST_Intersects(dz.coverage::geometry, v_point::geometry) AS is_covered
    FROM delivery_zones dz
    WHERE ST_Intersects(dz.coverage::geometry, v_point::geometry);
END;
$$ LANGUAGE plpgsql STABLE;
```

ทดสอบด้วยที่อยู่ในเขตให้บริการ (สีลม):

```sql
SELECT * FROM check_delivery_coverage(100.5290, 13.7230);
```

**ผลลัพธ์ตัวอย่าง:**

```
 zone_id |          zone_name           | is_covered
---------+---------------------------------+-------------
       1 | เขตจัดส่งปทุมวัน-บางรัก        | t
(1 row)
```

ทดสอบด้วยที่อยู่นอกเขตให้บริการ (สายไหม):

```sql
SELECT * FROM check_delivery_coverage(100.6540, 13.9150);
```

**ผลลัพธ์ตัวอย่าง:**

```
 zone_id | zone_name | is_covered
---------+-----------+------------
(0 rows)
```

`(0 rows)` หมายความว่าที่อยู่นี้อยู่นอกทุกเขตการให้บริการจัดส่ง — ระบบ e-commerce สามารถใช้ผลลัพธ์นี้ปฏิเสธคำสั่งซื้อแบบจัดส่งด่วน หรือแจ้งเตือนลูกค้าได้ทันที

### 3. สร้าง View รวมสำหรับ Dashboard ผู้ดูแลระบบ

```sql
CREATE OR REPLACE VIEW v_delivery_assignment AS
SELECT
    d.address_id,
    d.customer_id,
    d.recipient_name,
    d.address_line,
    nearest.store_name AS assigned_store,
    nearest.distance_meters,
    CASE
        WHEN EXISTS (
            SELECT 1 FROM delivery_zones dz
            WHERE ST_Intersects(dz.coverage::geometry, d.location::geometry)
        ) THEN TRUE
        ELSE FALSE
    END AS within_service_area
FROM delivery_addresses d
CROSS JOIN LATERAL (
    SELECT s.store_name, ROUND(ST_Distance(s.location, d.location)::numeric, 2) AS distance_meters
    FROM stores s
    ORDER BY s.location <-> d.location
    LIMIT 1
) nearest;
```

```sql
SELECT * FROM v_delivery_assignment ORDER BY customer_id;
```

**ผลลัพธ์ตัวอย่าง:**

```
 address_id | customer_id | recipient_name          | assigned_store           | distance_meters | within_service_area
------------+-------------+--------------------------+---------------------------+------------------+----------------------
          1 |         101 | คุณสมชาย ใจดี            | สาขาเทอร์มินอล 21 อโศก    |          2148.67 | t
          2 |         102 | คุณสมหญิง รักเรียน       | สาขาไอคอนสยาม             |          2701.90 | t
          3 |         103 | คุณวิชัย มั่นคง          | สาขาเซ็นทรัลลาดพร้าว      |         11245.33 | f
          4 |         104 | คุณนภา สว่างใจ           | สาขาเทอร์มินอล 21 อโศก    |          2589.11 | t
          5 |         105 | คุณอนันต์ พูนทรัพย์      | สาขาเซ็นทรัลลาดพร้าว      |          4890.22 | f
(5 rows)
```

View นี้แสดงให้เห็นภาพรวมทั้งหมด: สาขาที่ใกล้ที่สุดสำหรับแต่ละที่อยู่ ระยะทาง และสถานะว่าอยู่ในพื้นที่ให้บริการจัดส่งหรือไม่ ใน query เดียว — เป็นตัวอย่างของการนำแนวคิดทั้งหมดในบทนี้ (POINT, ST_Distance, `<->` KNN operator, ST_Intersects, LATERAL join) มาประกอบกันแก้โจทย์ธุรกิจจริง

---

## สรุปท้ายบท

ในบทนี้เราได้เรียนรู้พื้นฐานของ PostGIS ซึ่งเป็นเครื่องมือสำคัญที่ทำให้ PostgreSQL กลายเป็นฐานข้อมูลเชิงพื้นที่ (Geospatial Database) ระดับโลก ประเด็นสำคัญที่ควรจดจำ:

1. **PostGIS เป็น extension** ที่ต้องติดตั้งแยก ทั้งที่ระดับ OS package และเปิดใช้งานด้วย `CREATE EXTENSION postgis;` ในแต่ละฐานข้อมูล
2. **SRID 4326 (WGS84)** คือระบบพิกัดมาตรฐานที่ GPS ใช้จริง ลำดับพารามิเตอร์ใน `ST_MakePoint` คือ **(longitude, latitude)** เสมอ อย่าสลับลำดับ
3. **`geography` เหมาะกับข้อมูล GPS จริง** เพราะคำนวณแบบทรงกลม (spherical) ให้ผลลัพธ์ระยะทางเป็นเมตรทันที ในขณะที่ `geometry` เหมาะกับงานที่มีขอบเขตจำกัดและต้องการความเร็วสูงสุด (มักใช้ร่วมกับ projected CRS เช่น UTM)
4. ชนิดข้อมูลหลักที่ใช้บ่อย ได้แก่ `POINT` (ตำแหน่ง), `LINESTRING` (เส้นทาง), `POLYGON` (พื้นที่/เขตบริการ)
5. **`ST_Distance`** คำนวณระยะทางระหว่างจุดสองจุด ส่วน **`ST_DWithin`** ใช้ค้นหาจุดในรัศมีที่กำหนดได้อย่างมีประสิทธิภาพ (ใช้ spatial index ได้)
6. **`ST_Contains`/`ST_Intersects`** ใช้ตรวจสอบว่าจุดอยู่ในพื้นที่หรือรูปทรงสองชิ้นทับซ้อนกันหรือไม่ (ต้อง cast เป็น `::geometry` เสมอ)
7. **`ST_AsText`** ใช้ debug/แสดงผลแบบข้อความ ส่วน **`ST_AsGeoJSON`** คือรูปแบบมาตรฐานสำหรับส่งข้อมูลไปแสดงบนแผนที่ (Leaflet, Mapbox, Google Maps)
8. ควรสร้าง **GiST index** บนคอลัมน์ geospatial เสมอ (`CREATE INDEX ... USING GIST (...)`) และใช้ operator **`<->`** สำหรับ nearest-neighbor search เพื่อประสิทธิภาพสูงสุด
9. โจทย์ธุรกิจจริงอย่าง "หาสาขาใกล้ที่สุด" และ "ตรวจสอบพื้นที่ให้บริการจัดส่ง" สามารถแก้ได้ด้วยการผสมผสานฟังก์ชันเหล่านี้เข้าด้วยกันในรูปแบบ function, view หรือ query ตรง

ในบทถัดไป (Part 077) เราจะเจาะลึก PostGIS ในระดับที่สูงขึ้น เช่น spatial join ขั้นสูง, การทำ geocoding/reverse geocoding, การจัดการ raster data, การ optimize query เชิงพื้นที่สำหรับข้อมูลขนาดใหญ่ระดับล้านแถว และการเชื่อมต่อกับระบบแผนที่ระดับ production

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1

จงเขียนคำสั่ง SQL เพื่อติดตั้งและตรวจสอบว่า extension PostGIS ถูกเปิดใช้งานในฐานข้อมูลปัจจุบันเรียบร้อยแล้ว

<details>
<summary>เฉลยแบบฝึกหัดที่ 1</summary>

```sql
CREATE EXTENSION IF NOT EXISTS postgis;

SELECT extname, extversion
FROM pg_extension
WHERE extname = 'postgis';
```

หากมีแถวผลลัพธ์แสดงชื่อ `postgis` พร้อมเลขเวอร์ชัน แปลว่าเปิดใช้งานสำเร็จแล้ว
</details>

---

### แบบฝึกหัดที่ 2

จงเพิ่มสาขาใหม่ชื่อ "สาขาฟิวเจอร์พาร์ค รังสิต" ที่พิกัด longitude 100.6120, latitude 13.9967 เข้าไปในตาราง `stores`

<details>
<summary>เฉลยแบบฝึกหัดที่ 2</summary>

```sql
INSERT INTO stores (store_name, address, location)
VALUES (
    'สาขาฟิวเจอร์พาร์ค รังสิต',
    '94 ถนนพหลโยธิน ตำบลประชาธิปัตย์ อำเภอธัญบุรี ปทุมธานี 12130',
    ST_SetSRID(ST_MakePoint(100.6120, 13.9967), 4326)::geography
);
```

**ข้อควรระวัง:** ต้องใส่ longitude ก่อน latitude ใน `ST_MakePoint` และต้อง cast เป็น `::geography` ให้ตรงกับชนิดข้อมูลของคอลัมน์ `location`
</details>

---

### แบบฝึกหัดที่ 3

จงอธิบายความแตกต่างระหว่าง `geometry` และ `geography` type และยกตัวอย่างว่าในกรณีใดควรเลือกใช้แบบใด

<details>
<summary>เฉลยแบบฝึกหัดที่ 3</summary>

`geometry` คำนวณแบบ **planar** (ระนาบแบน) เหมาะกับพื้นที่ขอบเขตจำกัดที่แปลงเป็น projected CRS แล้ว (เช่น UTM) ให้ผลลัพธ์เร็วกว่าและแม่นยำในพื้นที่จำกัด ส่วน `geography` คำนวณแบบ **spherical** (ทรงกลม) คำนึงถึงความโค้งของโลก เหมาะกับข้อมูล GPS จริงที่กระจายในพื้นที่กว้าง (ข้ามจังหวัด/ประเทศ) ให้ผลลัพธ์ระยะทางเป็นเมตรทันทีโดยไม่ต้องแปลง projection เอง

ตัวอย่าง: ระบบหาสาขาใกล้ที่สุดของธุรกิจ e-commerce ที่มีสาขาทั่วประเทศไทย ควรใช้ `geography` เพราะครอบคลุมพื้นที่กว้างและต้องการความถูกต้องของระยะทางจริงทันที ในขณะที่งานวิเคราะห์ผังเมืองในเขตเดียวที่ต้องการความเร็วสูงสุดและควบคุม projection เอง ควรใช้ `geometry` ร่วมกับ UTM
</details>

---

### แบบฝึกหัดที่ 4

จงเขียนคำสั่ง SQL คำนวณระยะทาง (เป็นเมตร) ระหว่างสาขา "สาขาเซ็นทรัลลาดพร้าว" กับสาขา "สาขาเทอร์มินอล 21 อโศก"

<details>
<summary>เฉลยแบบฝึกหัดที่ 4</summary>

```sql
SELECT
    a.store_name AS store_a,
    b.store_name AS store_b,
    ROUND(ST_Distance(a.location, b.location)::numeric, 2) AS distance_meters
FROM stores a, stores b
WHERE a.store_name = 'สาขาเซ็นทรัลลาดพร้าว'
  AND b.store_name = 'สาขาเทอร์มินอล 21 อโศก';
```

เนื่องจากทั้งสองคอลัมน์เป็นชนิด `geography` อยู่แล้ว `ST_Distance` จะคืนค่าเป็นเมตรโดยอัตโนมัติ ไม่ต้องแคสต์เพิ่ม
</details>

---

### แบบฝึกหัดที่ 5

จงเขียนคำสั่ง SQL หาสาขาทั้งหมดที่อยู่ในระยะ 3 กิโลเมตรจากที่อยู่จัดส่งของลูกค้า `customer_id = 102`

<details>
<summary>เฉลยแบบฝึกหัดที่ 5</summary>

```sql
SELECT
    s.store_name,
    ROUND(ST_Distance(s.location, d.location)::numeric, 2) AS distance_meters
FROM stores s
CROSS JOIN delivery_addresses d
WHERE d.customer_id = 102
  AND ST_DWithin(s.location, d.location, 3000)
ORDER BY distance_meters;
```

ใช้ `ST_DWithin` แทน `ST_Distance(...) < 3000` เพราะสามารถใช้ประโยชน์จาก GiST index ได้ ทำให้เร็วกว่าเมื่อข้อมูลมีขนาดใหญ่
</details>

---

### แบบฝึกหัดที่ 6

จงสร้าง `LINESTRING` แทนเส้นทางจากสาขาไอคอนสยาม (100.5099, 13.7262) ไปยังสาขาเซ็นทรัลเวิลด์ (100.5393, 13.7466) โดยผ่านจุดกึ่งกลาง (100.5250, 13.7350) แล้วคำนวณความยาวเส้นทางเป็นเมตร

<details>
<summary>เฉลยแบบฝึกหัดที่ 6</summary>

```sql
SELECT ST_Length(
    ST_SetSRID(
        ST_MakeLine(ARRAY[
            ST_MakePoint(100.5099, 13.7262),
            ST_MakePoint(100.5250, 13.7350),
            ST_MakePoint(100.5393, 13.7466)
        ]),
        4326
    )::geography
) AS route_length_meters;
```

ผลลัพธ์จะเป็นความยาวรวมของเส้นทางเป็นเมตร (ประมาณ 3,900 - 4,200 เมตร ขึ้นอยู่กับการคำนวณจริง)
</details>

---

### แบบฝึกหัดที่ 7

จงเขียนคำสั่ง SQL ตรวจสอบว่าที่อยู่จัดส่งของลูกค้า `customer_id = 105` (แถวรัชดาภิเษก) อยู่ในเขตจัดส่งใดบ้าง (ถ้ามี) โดยใช้ `ST_Intersects`

<details>
<summary>เฉลยแบบฝึกหัดที่ 7</summary>

```sql
SELECT
    d.recipient_name,
    dz.zone_name
FROM delivery_addresses d
LEFT JOIN delivery_zones dz
    ON ST_Intersects(dz.coverage::geometry, d.location::geometry)
WHERE d.customer_id = 105;
```

ใช้ `LEFT JOIN` เพื่อให้เห็นผลลัพธ์แม้ที่อยู่นี้จะไม่อยู่ในเขตใดเลย (จะได้ `NULL` ในคอลัมน์ `zone_name`) ซึ่งจากข้อมูลตัวอย่างในบทเรียน ลูกค้ารายนี้อยู่นอกทั้งสองเขตที่กำหนดไว้ จึงได้ `zone_name = NULL`
</details>

---

### แบบฝึกหัดที่ 8

จงเขียนคำสั่ง SQL แปลงพิกัดของสาขาทั้งหมดเป็นรูปแบบ GeoJSON `FeatureCollection` เดียว โดยแต่ละ feature มี properties เป็นชื่อสาขาและเบอร์โทร

<details>
<summary>เฉลยแบบฝึกหัดที่ 8</summary>

```sql
SELECT json_build_object(
    'type', 'FeatureCollection',
    'features', json_agg(
        json_build_object(
            'type', 'Feature',
            'geometry', ST_AsGeoJSON(location::geometry)::json,
            'properties', json_build_object(
                'store_name', store_name,
                'phone', phone
            )
        )
    )
) AS geojson
FROM stores;
```
</details>

---

### แบบฝึกหัดที่ 9

จงสร้าง GiST index บนคอลัมน์ `location` ของตาราง `delivery_addresses` (หากยังไม่ได้สร้าง) และอธิบายว่าทำไม index ประเภทนี้จึงจำเป็นสำหรับคอลัมน์เชิงพื้นที่

<details>
<summary>เฉลยแบบฝึกหัดที่ 9</summary>

```sql
CREATE INDEX IF NOT EXISTS idx_delivery_addresses_location
    ON delivery_addresses USING GIST (location);
```

**คำอธิบาย:** B-tree index (ค่าเริ่มต้นของ PostgreSQL) เหมาะกับข้อมูลที่เรียงลำดับเชิงเส้นได้ (ตัวเลข, ตัวอักษร) แต่ข้อมูลเชิงพื้นที่ (geometry/geography) เป็นข้อมูลหลายมิติที่ไม่มีลำดับเชิงเส้นตามธรรมชาติ **GiST (Generalized Search Tree)** คือโครงสร้าง index ที่ออกแบบมาให้รองรับการค้นหาแบบหลายมิติ เช่น "มีจุดใดอยู่ในกรอบสี่เหลี่ยมนี้บ้าง" หรือ "จุดใดใกล้จุดนี้ที่สุด" ได้อย่างมีประสิทธิภาพ ทำให้ฟังก์ชันอย่าง `ST_DWithin`, `ST_Intersects`, และ operator `<->` สามารถใช้ index ช่วยกรองข้อมูลได้ตั้งแต่ต้น แทนที่จะต้องสแกนทุกแถวแล้วคำนวณทีละแถว
</details>

---

### แบบฝึกหัดที่ 10 (แบบฝึกหัดรวม)

จงเขียนฟังก์ชัน SQL ชื่อ `get_store_delivery_summary(p_store_id INTEGER)` ที่รับ `store_id` แล้วคืนค่าจำนวนที่อยู่จัดส่งทั้งหมดที่มีสาขานี้เป็นสาขาที่ใกล้ที่สุด พร้อมระยะทางเฉลี่ย (เมตร) ไปยังที่อยู่เหล่านั้น

<details>
<summary>เฉลยแบบฝึกหัดที่ 10</summary>

```sql
CREATE OR REPLACE FUNCTION get_store_delivery_summary(p_store_id INTEGER)
RETURNS TABLE (
    store_name VARCHAR,
    total_nearest_addresses BIGINT,
    avg_distance_meters NUMERIC
) AS $$
BEGIN
    RETURN QUERY
    WITH nearest_store_per_address AS (
        SELECT
            d.address_id,
            (
                SELECT s.store_id
                FROM stores s
                ORDER BY s.location <-> d.location
                LIMIT 1
            ) AS nearest_store_id,
            (
                SELECT ST_Distance(s.location, d.location)
                FROM stores s
                ORDER BY s.location <-> d.location
                LIMIT 1
            ) AS distance_to_nearest
        FROM delivery_addresses d
    )
    SELECT
        s.store_name,
        COUNT(nsa.address_id) AS total_nearest_addresses,
        ROUND(AVG(nsa.distance_to_nearest)::numeric, 2) AS avg_distance_meters
    FROM stores s
    LEFT JOIN nearest_store_per_address nsa
        ON nsa.nearest_store_id = s.store_id
    WHERE s.store_id = p_store_id
    GROUP BY s.store_name;
END;
$$ LANGUAGE plpgsql STABLE;
```

ทดสอบใช้งาน:

```sql
SELECT * FROM get_store_delivery_summary(4);  -- สาขาเทอร์มินอล 21 อโศก
```

**ผลลัพธ์ตัวอย่าง:**

```
       store_name        | total_nearest_addresses | avg_distance_meters
--------------------------+---------------------------+-----------------------
 สาขาเทอร์มินอล 21 อโศก    |                         2 |              2368.89
(1 row)
```

ฟังก์ชันนี้แสดงให้เห็นการรวมแนวคิดของทั้งบท: การใช้ `<->` KNN operator ค้นหาสาขาที่ใกล้ที่สุดของแต่ละที่อยู่ผ่าน correlated subquery, การจัดกลุ่มด้วย `GROUP BY`, และการคำนวณค่าเฉลี่ยระยะทางด้วย `AVG` — เป็นรูปแบบ query ที่ใช้งานได้จริงสำหรับ dashboard วิเคราะห์ภาระงานของแต่ละสาขาในระบบ e-commerce
</details>

---

## บทถัดไป

เมื่อเข้าใจพื้นฐานของ PostGIS ครบถ้วนแล้ว ในบทถัดไปเราจะเจาะลึกฟีเจอร์ขั้นสูงของ PostGIS สำหรับงานระดับ production เช่น spatial join ที่ซับซ้อน, geocoding, raster data และการ optimize สำหรับข้อมูลขนาดใหญ่

**อ่านต่อ:** [Part 077: PostGIS ขั้นสูง](./part-077-postgis-advanced.md)
