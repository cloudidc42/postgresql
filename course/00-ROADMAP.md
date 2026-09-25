# สารบัญหลักสูตร PostgreSQL ฉบับสมบูรณ์ (Roadmap: Step 1–1000)

เอกสารนี้คือแผนที่หลักสูตรทั้งหมด แบ่งเป็น 5 ระดับ รวม 104 Part ครอบคลุม Step ที่ 1–1000 โดยแต่ละ Part จะมีไฟล์ markdown แยกของตัวเอง เนื้อหาใช้งานได้จริง 500–3000+ บรรทัดต่อ Part

**สถานะ:** ✅ เขียนเสร็จแล้ว | 🚧 กำลังเขียน | ⬜ ยังไม่ได้เริ่ม

---

## 🟢 ระดับ 1: พื้นฐาน (Foundations) — Step 1–200

โฟลเดอร์: `course/01-foundations/`

| Part | Steps | หัวข้อ | สถานะ |
|---|---|---|---|
| 001 | 1–10 | Database คืออะไร, RDBMS, ทำไมต้อง PostgreSQL, ประวัติและสถาปัตยกรรมโดยรวม | ✅ |
| 002 | 11–20 | การติดตั้ง PostgreSQL บน Windows/macOS/Linux และผ่าน Docker | ✅ |
| 003 | 21–30 | เครื่องมือทำงาน: psql, pgAdmin, DBeaver, VS Code extensions | ✅ |
| 004 | 31–40 | โครงสร้างฐานข้อมูล: Cluster, Database, Schema, Table, Catalog | ✅ |
| 005 | 41–50 | การสร้างและจัดการฐานข้อมูล (CREATE/ALTER/DROP DATABASE) | ✅ |
| 006 | 51–60 | Data Types พื้นฐาน: ตัวเลข, ข้อความ, boolean | ✅ |
| 007 | 61–70 | Data Types วันที่และเวลา (date, time, timestamp, interval) | ✅ |
| 008 | 71–80 | การสร้างตาราง CREATE TABLE และการออกแบบคอลัมน์ | ✅ |
| 009 | 81–90 | INSERT: เพิ่มข้อมูลเดี่ยว, หลายแถว, INSERT...SELECT | ✅ |
| 010 | 91–100 | SELECT พื้นฐาน: เลือกคอลัมน์, alias, DISTINCT | ✅ |
| 011 | 101–110 | WHERE clause และ Operators (comparison, logical, LIKE, IN, BETWEEN) | ✅ |
| 012 | 111–120 | ORDER BY, LIMIT, OFFSET, FETCH | ✅ |
| 013 | 121–130 | UPDATE: แก้ไขข้อมูลแบบมีเงื่อนไขและหลายคอลัมน์ | ✅ |
| 014 | 131–140 | DELETE และความแตกต่างจาก TRUNCATE | ✅ |
| 015 | 141–150 | NULL คืออะไร และการจัดการ NULL อย่างถูกต้อง | ✅ |
| 016 | 151–160 | Primary Key และ Unique Constraint | ✅ |
| 017 | 161–170 | Foreign Key และ Referential Integrity, ON DELETE/UPDATE | ✅ |
| 018 | 171–180 | Check Constraint, Default Value, Not Null | ✅ |
| 019 | 181–190 | ALTER TABLE: เพิ่ม/ลบ/แก้ไขคอลัมน์และ constraint | ✅ |
| 020 | 191–200 | โปรเจกต์รวมระดับพื้นฐาน: ระบบจัดการห้องสมุด (Library Management) | ✅ |

## 🔵 ระดับ 2: ระดับกลาง (Intermediate) — Step 201–400

โฟลเดอร์: `course/02-intermediate/`

| Part | Steps | หัวข้อ | สถานะ |
|---|---|---|---|
| 021 | 201–210 | JOIN พื้นฐาน: INNER JOIN และการเชื่อมหลายตาราง | ✅ |
| 022 | 211–220 | LEFT/RIGHT/FULL OUTER JOIN | ✅ |
| 023 | 221–230 | CROSS JOIN, SELF JOIN, NATURAL JOIN | ✅ |
| 024 | 231–240 | Subqueries: scalar, correlated, EXISTS/NOT EXISTS | ✅ |
| 025 | 241–250 | Common Table Expressions (WITH / CTE) | ✅ |
| 026 | 251–260 | Recursive CTE และการใช้งานกับข้อมูลแบบ hierarchy | ✅ |
| 027 | 261–270 | Aggregate Functions: COUNT, SUM, AVG, MIN, MAX | ✅ |
| 028 | 271–280 | GROUP BY, HAVING, GROUPING SETS, ROLLUP, CUBE | ✅ |
| 029 | 281–290 | Window Functions พื้นฐาน: OVER, PARTITION BY | ✅ |
| 030 | 291–300 | Window Functions ขั้นสูง: RANK, DENSE_RANK, ROW_NUMBER, LAG/LEAD, NTILE | ✅ |
| 031 | 301–310 | String Functions และ Pattern Matching (regex) | ✅ |
| 032 | 311–320 | Date/Time Functions และการคำนวณช่วงเวลา | ✅ |
| 033 | 321–330 | Numeric/Math Functions และ Type Casting | ✅ |
| 034 | 331–340 | CASE WHEN, COALESCE, NULLIF และ Conditional Logic | ✅ |
| 035 | 341–350 | Views: การสร้างและใช้งาน Virtual Table | ✅ |
| 036 | 351–360 | Materialized Views และการ REFRESH | ✅ |
| 037 | 361–370 | Transactions และหลักการ ACID | ✅ |
| 038 | 371–380 | Isolation Levels และปัญหา Concurrency (dirty read, phantom read) | ✅ |
| 039 | 381–390 | Sequences, SERIAL, IDENTITY Columns | ✅ |
| 040 | 391–400 | โปรเจกต์รวมระดับกลาง: ระบบร้านค้าออนไลน์ (E-Commerce Schema) | ✅ |

## 🟣 ระดับ 3: ระดับสูง (Advanced) — Step 401–600

โฟลเดอร์: `course/03-advanced/`

| Part | Steps | หัวข้อ | สถานะ |
|---|---|---|---|
| 041 | 401–410 | Index พื้นฐาน: B-Tree และหลักการทำงาน | ✅ |
| 042 | 411–420 | Index ขั้นสูง: Hash, GiST, GIN, BRIN, SP-GiST | ✅ |
| 043 | 421–430 | Partial Index, Expression Index, Multi-column Index | ✅ |
| 044 | 431–440 | Query Planner และการอ่าน EXPLAIN / EXPLAIN ANALYZE | ✅ |
| 045 | 441–450 | เทคนิค Query Optimization | ✅ |
| 046 | 451–460 | PL/pgSQL: Stored Procedures และ Functions เบื้องต้น | ✅ |
| 047 | 461–470 | PL/pgSQL ขั้นสูง: Loop, Exception Handling, Dynamic SQL | ✅ |
| 048 | 471–480 | Triggers และ Trigger Functions | ✅ |
| 049 | 481–490 | Custom Data Types, Domains, ENUM | ✅ |
| 050 | 491–500 | Arrays: การสร้าง จัดเก็บ และ Query | ✅ |
| 051 | 501–510 | JSON และ JSONB พื้นฐาน | ⬜ |
| 052 | 511–520 | JSONB Query ขั้นสูงและ Indexing (GIN) | ⬜ |
| 053 | 521–530 | Full Text Search (tsvector, tsquery, ranking) | ⬜ |
| 054 | 531–540 | Table Partitioning: Range, List, Hash Partitioning | ⬜ |
| 055 | 541–550 | Table Inheritance และเปรียบเทียบกับ Partitioning | ⬜ |
| 056 | 551–560 | Foreign Data Wrappers (postgres_fdw, file_fdw) | ⬜ |
| 057 | 561–570 | Extensions ที่สำคัญ: pg_stat_statements, pgcrypto, uuid-ossp, pg_trgm | ⬜ |
| 058 | 571–580 | MVCC เชิงลึก: Locking และ Concurrency Control | ⬜ |
| 059 | 581–590 | Deadlock: สาเหตุ การตรวจจับ และการป้องกัน | ⬜ |
| 060 | 591–600 | VACUUM, AUTOVACUUM และการจัดการ Bloat — พร้อมโปรเจกต์ Analytics Dashboard | ⬜ |

## 🟠 ระดับ 4: ระดับมืออาชีพ (Professional) — Step 601–800

โฟลเดอร์: `course/04-professional/`

| Part | Steps | หัวข้อ | สถานะ |
|---|---|---|---|
| 061 | 601–610 | Backup Strategies: pg_dump, pg_dumpall, pg_basebackup | ⬜ |
| 062 | 611–620 | Point-in-Time Recovery (PITR) และ WAL Archiving | ⬜ |
| 063 | 621–630 | Streaming Replication (Physical Replication) | ⬜ |
| 064 | 631–640 | Logical Replication และ Publication/Subscription | ⬜ |
| 065 | 641–650 | High Availability: Patroni, repmgr, Failover Automation | ⬜ |
| 066 | 651–660 | Connection Pooling: PgBouncer, Pgpool-II | ⬜ |
| 067 | 661–670 | Load Balancing และ Read Replica Strategy | ⬜ |
| 068 | 671–680 | Security: Roles, Privileges, GRANT/REVOKE | ⬜ |
| 069 | 681–690 | Row Level Security (RLS) และ Multi-tenant Security | ⬜ |
| 070 | 691–700 | SSL/TLS, Encryption at Rest/in Transit | ⬜ |
| 071 | 701–710 | Auditing และ Logging (pgAudit, log configuration) | ⬜ |
| 072 | 711–720 | Monitoring: pg_stat views, Prometheus + Grafana, pganalyze | ⬜ |
| 073 | 721–730 | Performance Tuning: postgresql.conf ระดับ Production | ⬜ |
| 074 | 731–740 | Performance Tuning: Query-Level Deep Dive | ⬜ |
| 075 | 741–750 | Capacity Planning และ Horizontal/Vertical Scaling | ⬜ |
| 076 | 751–760 | PostGIS เบื้องต้น: Geospatial Database | ⬜ |
| 077 | 761–770 | PostGIS ขั้นสูง: Spatial Index และ Geospatial Query | ⬜ |
| 078 | 771–780 | TimescaleDB สำหรับ Time-Series Data | ⬜ |
| 079 | 781–790 | Database Migration Strategies (Zero-downtime Migration) | ⬜ |
| 080 | 791–800 | โปรเจกต์มืออาชีพ: ระบบ Production-Grade Multi-tenant SaaS | ⬜ |

## 🔴 ระดับ 5: ระดับโลก / ผู้เชี่ยวชาญ (World-Class / Expert) — Step 801–1000+

โฟลเดอร์: `course/05-world-class/`

| Part | Steps | หัวข้อ | สถานะ |
|---|---|---|---|
| 081 | 801–810 | PostgreSQL Internals: Process Architecture (Postmaster, Backend) | ⬜ |
| 082 | 811–820 | Storage Engine: Heap, Page Layout, TOAST | ⬜ |
| 083 | 821–830 | Write-Ahead Logging (WAL) เชิงลึก | ⬜ |
| 084 | 831–840 | MVCC Internals เชิงลึก: Tuple Visibility, xmin/xmax | ⬜ |
| 085 | 841–850 | การเขียน Custom Extension ด้วยภาษา C | ⬜ |
| 086 | 851–860 | Query Planner Internals: Cost-based Optimization | ⬜ |
| 087 | 861–870 | เชื่อมต่อ PostgreSQL กับ Node.js (node-postgres, Prisma) | ⬜ |
| 088 | 871–880 | เชื่อมต่อ PostgreSQL กับ Python (psycopg, SQLAlchemy, Django ORM) | ⬜ |
| 089 | 881–890 | เชื่อมต่อ PostgreSQL กับ Go (pgx, GORM) | ⬜ |
| 090 | 891–900 | เชื่อมต่อ PostgreSQL กับ Java (JDBC, Hibernate/JPA) | ⬜ |
| 091 | 901–910 | Database Design Patterns สำหรับสถาปัตยกรรม Microservices | ⬜ |
| 092 | 911–920 | Event Sourcing และ CQRS กับ PostgreSQL | ⬜ |
| 093 | 921–930 | PostgreSQL บน Docker และ Kubernetes (Operators: Zalando, CloudNativePG) | ⬜ |
| 094 | 931–940 | PostgreSQL บน Cloud: AWS RDS และ Aurora PostgreSQL | ⬜ |
| 095 | 941–950 | PostgreSQL บน Cloud: GCP Cloud SQL และ AlloyDB | ⬜ |
| 096 | 951–955 | PostgreSQL บน Cloud: Azure Database for PostgreSQL | ⬜ |
| 097 | 956–965 | Distributed PostgreSQL: Citus และ Sharding Strategy | ⬜ |
| 098 | 966–975 | Benchmarking ด้วย pgbench และ Load Testing | ⬜ |
| 099 | 976–985 | Disaster Recovery และ Business Continuity Planning | ⬜ |
| 100 | 986–990 | Case Studies: สถาปัตยกรรมระบบระดับ Enterprise จริง | ⬜ |
| 101 | 991–993 | เตรียมสอบ PostgreSQL Certification และ System Design Interview | ⬜ |
| 102 | 994–996 | การมีส่วนร่วมใน PostgreSQL Open Source Community | ⬜ |
| 103 | 997–999 | อนาคตของ PostgreSQL: Roadmap และฟีเจอร์ใหม่ (PG16/17/18) | ⬜ |
| 104 | 1000 | Capstone Project: สร้างระบบ Full-Stack Production-Ready ด้วย PostgreSQL | ⬜ |

---

## หลักเกณฑ์การเขียนเนื้อหาแต่ละ Part

1. **ความยาว**: 500–3000+ บรรทัดต่อไฟล์ ขึ้นกับความซับซ้อนของหัวข้อ
2. **ภาษา**: ภาษาไทยเป็นหลัก คำศัพท์เทคนิคคงคำอังกฤษไว้เพื่อความถูกต้อง
3. **โครงสร้างมาตรฐานของแต่ละ Part**:
   - หัวข้อและเป้าหมายการเรียนรู้ (Learning Objectives)
   - เนื้อหาทฤษฎีสั้น กระชับ เข้าใจง่าย
   - ตัวอย่าง SQL ที่รันได้จริงพร้อมผลลัพธ์ตัวอย่าง
   - Use case จากสถานการณ์จริง
   - ข้อควรระวัง / Best Practices / Common Pitfalls
   - แบบฝึกหัดท้ายบท (5–10 ข้อ) พร้อมเฉลย
   - สรุปท้ายบทและลิงก์ไป Part ถัดไป
4. **ทุกตัวอย่างทดสอบได้กับ PostgreSQL เวอร์ชัน 16/17**
5. Dataset ตัวอย่างที่ใช้ร่วมกันหลาย Part เก็บไว้ที่ `course/datasets/`

## ลำดับการพัฒนา (Development Order)

เนื้อหาจะถูกพัฒนาไล่ตามลำดับ Part 001 → 104 โดยจะ commit และ push เข้า branch เป็นระยะทุกครั้งที่เขียนเสร็จเป็นชุด (batch) เพื่อให้ติดตามความคืบหน้าได้ตลอดเวลาผ่านตาราง checklist ด้านบน
