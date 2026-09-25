# Part 099: Disaster Recovery และ Business Continuity Planning

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับโลก/ผู้เชี่ยวชาญ | Part 099

---

## เป้าหมายการเรียนรู้

เมื่อจบบทนี้ ผู้เรียนจะสามารถ:

1. อธิบายความแตกต่างระหว่าง **Disaster Recovery (DR)**, **High Availability (HA)**, และ **Business Continuity Planning (BCP)** ได้อย่างชัดเจน พร้อมยกตัวอย่างสถานการณ์ที่แต่ละแนวคิดครอบคลุม
2. คำนวณและกำหนด **RPO (Recovery Point Objective)** และ **RTO (Recovery Time Objective)** ให้สอดคล้องกับความต้องการทางธุรกิจจริง ไม่ใช่ตัวเลขที่เดาสุ่ม
3. เปรียบเทียบ **DR Tier** ทั้งสี่แบบ (Backup/Restore, Pilot Light, Warm Standby, Multi-Site Active-Active) และเลือกใช้ให้เหมาะกับงบประมาณและความเสี่ยงที่ยอมรับได้
4. ออกแบบ **Multi-Region Strategy** โดยใช้ physical replication ข้าม region สำหรับ PostgreSQL
5. เขียน **DR Runbook** ที่เป็นขั้นตอนชัดเจน ไม่กำกวม สามารถให้คนที่ไม่เคยเห็นระบบมาก่อนทำตามได้ในสถานการณ์วิกฤต
6. วางแผนและดำเนินการ **DR Testing** รวมถึง Game Day และแนวคิดพื้นฐานของ Chaos Engineering
7. เข้าใจภาพรวมของ **BCP** ที่กว้างกว่าระบบ IT ครอบคลุมทั้งองค์กร
8. ออกแบบ **Communication Plan** สำหรับสื่อสารกับผู้มีส่วนได้ส่วนเสียระหว่างเกิดเหตุการณ์วิกฤต
9. จัดทำ **Postmortem** แบบ blameless เพื่อเรียนรู้จากเหตุการณ์จริงและป้องกันไม่ให้เกิดซ้ำ
10. ประยุกต์ความรู้ทั้งหมดเขียน DR Runbook และ BCP ฉบับเต็มสำหรับระบบ e-commerce SaaS จริง

---

## บทนำ: ทำไมต้องมี DR ทั้งที่มี HA อยู่แล้ว

ในบท Part 061-065 เราได้เรียนรู้เรื่อง Backup Strategy, PITR, Physical Replication, Logical Replication และ High Availability (HA) กับ Failover มาอย่างละเอียดแล้ว หลายคนอาจคิดว่า "ถ้ามี HA ที่ failover อัตโนมัติได้ใน 30 วินาที ก็ไม่จำเป็นต้องมี DR plan อีกแล้ว" ซึ่งเป็นความเข้าใจผิดที่พบบ่อยมากและเป็นสาเหตุให้หลายองค์กรล่มสลายเมื่อเจอภัยพิบัติจริง

**High Availability (HA)** ออกแบบมาเพื่อรับมือกับ **ความล้มเหลวระดับ component** เช่น server ตัวหนึ่งเสีย, disk พัง, process crash โดยระบบ standby ที่อยู่ใกล้ๆ (มักอยู่ใน datacenter เดียวกันหรือ availability zone เดียวกัน) จะ failover เข้ามาทำงานแทนอย่างรวดเร็ว

**Disaster Recovery (DR)** ออกแบบมาเพื่อรับมือกับ **ความล้มเหลวระดับใหญ่กว่านั้นมาก** — เหตุการณ์ที่ทำให้ "ทั้งระบบ" หรือ "ทั้ง site" ใช้งานไม่ได้พร้อมกัน เช่น:

- Datacenter ทั้งหลังไฟดับ, น้ำท่วม, ไฟไหม้
- Cloud region ทั้ง region ล่ม (เกิดขึ้นจริงกับ AWS, GCP, Azure มาแล้วหลายครั้ง)
- Ransomware attack ที่เข้ารหัสข้อมูลทั้ง production และ backup ที่เชื่อมต่ออยู่
- Human error ร้ายแรง เช่น `DROP TABLE` หรือ `DELETE FROM orders` โดยไม่มี `WHERE` บน production
- ภัยธรรมชาติ, สงคราม, การก่อการร้าย, หรือเหตุการณ์ที่กระทบทั้งภูมิภาค

ตารางเปรียบเทียบ HA กับ DR:

| มิติ | High Availability (HA) | Disaster Recovery (DR) |
|---|---|---|
| ขอบเขตความล้มเหลว | Component/Node เดียว | Site/Region/องค์กรทั้งหมด |
| ระยะทางระหว่าง node | มักอยู่ใกล้กัน (same DC/AZ) | อยู่ห่างกันมาก (ข้าม region/ประเทศ) |
| ความถี่ในการเกิดเหตุ | บ่อย (hardware failure เกิดได้ทุกสัปดาห์) | น้อยมาก (อาจเกิดปีละครั้งหรือน้อยกว่า) |
| เวลาที่ใช้กู้คืน | วินาทีถึงนาที | นาทีถึงชั่วโมง (หรือวัน หากไม่มีแผน) |
| ต้นทุน | ปานกลาง | สูง (ต้องมี infrastructure คู่ขนาน) |
| เป้าหมาย | Continuity ระหว่างวันทำงานปกติ | Survival ขององค์กรเมื่อเกิดหายนะ |
| ตัวอย่างเทคโนโลยี | Patroni, repmgr, pg_auto_failover | Cross-region replication, offsite backup, runbook |

DR ไม่ใช่แค่เรื่อง "database" เดียว แต่เป็นแผนที่ครอบคลุม **ทั้ง stack**: application servers, load balancers, DNS, message queues, object storage, secrets management, และแน่นอนว่ารวมถึง database ด้วย บทนี้จะเน้นมุมมองของ PostgreSQL เป็นหลัก แต่จะเชื่อมโยงให้เห็นภาพรวมขององค์กรทั้งหมดเสมอ เพราะ DBA หรือ Database Engineer ระดับ world-class ต้องเข้าใจบริบททางธุรกิจ ไม่ใช่แค่คำสั่ง SQL

---

## Step 976: Disaster Recovery (DR) คืออะไร

### 976.1 นิยามที่ถูกต้อง

**Disaster Recovery (DR)** คือชุดของนโยบาย เครื่องมือ และขั้นตอนที่ช่วยให้องค์กรสามารถกู้คืนหรือทำงานต่อได้หลังจากเกิด **ภัยพิบัติ (disaster)** ซึ่งหมายถึงเหตุการณ์ที่ทำให้ infrastructure หลักไม่สามารถใช้งานได้ทั้งหมดหรือเสียหายรุนแรงจนไม่สามารถกู้คืนในสถานที่เดิมได้ทันที

จุดสำคัญที่ต้องเข้าใจ:

1. **DR ไม่ใช่ backup อย่างเดียว** — backup เป็นเพียง "เครื่องมือชิ้นหนึ่ง" ที่ใช้ใน DR แต่ DR คือ "กระบวนการทั้งหมด" ตั้งแต่การตรวจจับภัยพิบัติ การตัดสินใจประกาศภาวะฉุกเฉิน การ failover ไปยัง site สำรอง การสื่อสารกับทีมงานและลูกค้า จนถึงการกลับมาทำงานปกติ (failback)
2. **DR ต้องมีแผนเป็นลายลักษณ์อักษร** — ถ้าแผนอยู่แค่ในหัวของวิศวกรคนใดคนหนึ่ง นั่นไม่ใช่ DR plan แต่เป็น "ความเสี่ยง" เพราะถ้าคนนั้นลาพักร้อนหรือเป็นคนที่ได้รับผลกระทบจากภัยพิบัติเอง (เช่น ไฟดับทั้งเมือง) องค์กรจะไม่มีใครกู้คืนระบบได้
3. **DR ต้องผ่านการทดสอบ** — แผนที่ไม่เคยทดสอบ คือแผนที่ใช้ไม่ได้ (รายละเอียดใน Step 981)
4. **DR มีต้นทุนเสมอ** — ยิ่งต้องการกู้คืนเร็วและสูญเสียข้อมูลน้อย ยิ่งต้องจ่ายแพงขึ้น องค์กรต้องตัดสินใจอย่างมีเหตุผลว่าคุ้มค่าหรือไม่ (รายละเอียดใน Step 978)

### 976.2 วงจรชีวิตของ DR (DR Lifecycle)

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Prevent   │ --> │   Detect    │ --> │   Respond   │ --> │   Recover   │
│ (ป้องกัน)    │     │ (ตรวจจับ)    │     │ (ตอบสนอง)    │     │ (กู้คืน)     │
└─────────────┘     └─────────────┘     └─────────────┘     └─────────────┘
       ^                                                            |
       |                                                            v
       |                     ┌─────────────┐                       |
       └-------------------- │   Review    │ <---------------------┘
                              │ (ทบทวน)      │
                              └─────────────┘
```

- **Prevent**: ลดโอกาสเกิดภัยพิบัติ เช่น deploy หลาย availability zone, ใช้ RAID, มี UPS, ตั้ง permission ให้รัดกุมเพื่อลด human error
- **Detect**: ระบบ monitoring/alerting ที่ตรวจจับความผิดปกติได้เร็ว (เชื่อมโยง Part 062 เรื่อง monitoring replication lag)
- **Respond**: ทีมงานทำตาม runbook ประกาศภาวะฉุกเฉิน และเริ่มกระบวนการกู้คืน
- **Recover**: ระบบกลับมาทำงานได้ (อาจอยู่ที่ site สำรองก่อน แล้วค่อย failback)
- **Review**: ทำ postmortem เพื่อปรับปรุงแผนสำหรับครั้งต่อไป (Step 984)

วงจรนี้วนซ้ำตลอดเวลา ไม่ใช่ทำครั้งเดียวจบ

### 976.3 ประเภทของภัยพิบัติที่ DR ต้องรับมือ

| ประเภท | ตัวอย่าง | ผลกระทบต่อ PostgreSQL |
|---|---|---|
| **Infrastructure failure** | Datacenter ไฟดับ, network switch เสียทั้งชุด | Primary + Standby ใน DC เดียวกันล่มพร้อมกัน |
| **Cloud provider outage** | AWS region ล่ม (เช่น us-east-1 incident ปี 2021, 2023) | ทุก instance ใน region นั้นเข้าถึงไม่ได้ |
| **Natural disaster** | น้ำท่วม, แผ่นดินไหว, พายุ | DC ทางกายภาพเสียหาย |
| **Cyber attack** | Ransomware, data breach | ข้อมูลถูกเข้ารหัสหรือขโมย รวมถึง backup ที่เชื่อมต่อ online |
| **Human error** | `DROP DATABASE production;`, deploy migration ผิด | ข้อมูลสูญหายหรือเสียหายแบบ logical (ไม่ใช่ physical) |
| **Supply chain / vendor failure** | ผู้ให้บริการ backup ล้มละลายหรือปิดบริการกะทันหัน | ไม่มี backup สำรองให้กู้คืน |
| **Software bug / data corruption** | Bug ทำให้ WAL เสียหาย, page checksum fail | ต้องใช้ backup ที่เก่ากว่าจุดที่เกิด corruption |

สังเกตว่า human error และ ransomware เป็นกรณีที่ **HA ช่วยไม่ได้เลย** เพราะ standby จะ replicate ความผิดพลาดนั้นตามไปด้วยทันที (โดยเฉพาะ synchronous replication) นี่คือเหตุผลสำคัญที่ต้องมี DR ที่แยกจาก HA อย่างชัดเจน — ต้องมี backup ที่ **immutable** และ **มี time delay** เพียงพอให้ตรวจพบความผิดปกติก่อนที่ backup ทุกชุดจะถูกทำลายตาม

### 976.4 DR กับ PostgreSQL ecosystem: เชื่อมโยงกับ Part ที่ผ่านมา

| หัวข้อ | Part ที่สอนไว้ | บทบาทใน DR |
|---|---|---|
| `pg_basebackup`, `pg_dump` | Part 061 | เป็นฐานของ Tier 1 (Backup/Restore) |
| PITR (Point-in-Time Recovery) | Part 061 | กู้คืนข้อมูลก่อนเกิด human error/corruption |
| WAL archiving | Part 061, 083 | ทำให้ RPO ต่ำกว่า backup แบบ full snapshot |
| Physical Streaming Replication | Part 063 | เป็นฐานของ Warm Standby และ Multi-Region |
| Logical Replication | Part 064 | ใช้ทำ selective replication ข้าม region หรือข้าม version |
| HA/Failover (Patroni ฯลฯ) | Part 065 | จัดการ automatic failover ภายใน site เดียว ไม่ใช่ DR โดยตรง |

บทนี้ (Part 099) จะนำองค์ประกอบเหล่านี้มา "สังเคราะห์" เป็นกลยุทธ์ DR ระดับองค์กรที่เป็นทางการ

---

## Step 977: RPO และ RTO เจาะลึก

### 977.1 นิยาม

- **RPO (Recovery Point Objective)**: ปริมาณข้อมูลสูงสุดที่องค์กรยอมรับได้ว่าจะ "สูญหาย" เมื่อเกิดภัยพิบัติ วัดเป็นหน่วยเวลา เช่น "RPO = 15 นาที" หมายความว่าถ้าระบบล่มตอน 10:00 น. ข้อมูลที่กู้คืนได้จะย้อนไปได้ไม่เกิน 09:45 น. (ข้อมูลระหว่าง 09:45-10:00 อาจสูญหาย)
- **RTO (Recovery Time Objective)**: ระยะเวลาสูงสุดที่องค์กรยอมรับได้ในการทำให้ระบบกลับมาใช้งานได้อีกครั้งหลังเกิดภัยพิบัติ เช่น "RTO = 1 ชั่วโมง" หมายความว่าตั้งแต่เริ่มเกิดเหตุจนถึงระบบใช้งานได้ปกติต้องไม่เกิน 1 ชั่วโมง

แผนภาพเส้นเวลา:

```
เวลา:    ... -30min  -15min  Disaster   +T1(detect)  +T2(decide)  +T3(recover) ...
                        │        │            │             │            │
Data:    [============ Last good backup/WAL ] X (data loss zone) X
                        └─────── RPO ─────────┘
Time:                            └──────────────── RTO ─────────────────┘
```

- **RPO** วัดในแนว "ข้อมูล" ย้อนไปทางซ้ายของเวลาเกิดเหตุ
- **RTO** วัดในแนว "เวลา" เดินไปข้างหน้าจากจุดเกิดเหตุจนกว่าระบบจะพร้อมใช้งาน

### 977.2 ทบทวนเชื่อมโยง Part 061

ใน Part 061 เราพูดถึง RPO/RTO ในบริบทของ backup strategy เบื้องต้น (Step 608 พูดถึงการทดสอบ restore) บทนี้จะยกระดับแนวคิดนั้นให้เป็น **กระบวนการทางธุรกิจอย่างเป็นทางการ** ไม่ใช่แค่ตัวเลขทางเทคนิค

สิ่งที่ต้องเข้าใจให้ลึกขึ้น:

1. **RPO/RTO ไม่ได้กำหนดโดยทีม engineering เพียงฝ่ายเดียว** — ต้องมาจากการเจรจาระหว่าง Business (CEO, COO, Product Owner), Finance (ต้นทุนที่ยอมจ่ายได้), Legal/Compliance (ข้อบังคับทางกฎหมาย เช่น PCI-DSS, GDPR), และ Engineering (ความเป็นไปได้ทางเทคนิค)
2. **RPO/RTO ต่างกันได้ตาม "ชนิดของข้อมูล/บริการ"** — เช่น ข้อมูล transaction ทางการเงินอาจต้องการ RPO เกือบ 0 แต่ log สำหรับ analytics อาจยอมรับ RPO เป็นชั่วโมงได้
3. **RTO ต้องนับรวมทุกขั้นตอน** ไม่ใช่แค่เวลา restore database แต่รวมถึงเวลาตรวจจับ (detection time), เวลาตัดสินใจ (decision time), เวลาสื่อสารทีม, เวลา provision infrastructure ใหม่, เวลาตรวจสอบความถูกต้องของข้อมูลก่อนเปิดใช้งานจริง (validation time)

### 977.3 การคำนวณ RPO/RTO จาก Business Impact Analysis (BIA)

ขั้นตอนที่ถูกต้องคือทำ **Business Impact Analysis** ก่อนกำหนด RPO/RTO:

**สูตรประเมินต้นทุนความเสียหายต่อชั่วโมง (Cost of Downtime):**

```
Cost of Downtime per Hour =
    (Lost Revenue per Hour)
  + (SLA Penalty per Hour)
  + (Productivity Loss: จำนวนพนักงาน × ค่าแรงเฉลี่ยต่อชั่วโมง × % ที่ทำงานไม่ได้)
  + (Reputation/Customer Churn Cost - ประมาณการ)
  + (Regulatory Fine Risk - ถ้ามี)
```

**ตัวอย่างการคำนวณจริง — ระบบ e-commerce SaaS (เชื่อมโยง Part 080):**

สมมติระบบ e-commerce SaaS มีข้อมูลดังนี้:
- รายได้เฉลี่ย 120,000,000 บาทต่อเดือน → เฉลี่ย 166,667 บาทต่อชั่วโมง (คำนวณจาก 30 วัน × 24 ชม.)
- ช่วง peak (11:00-13:00, 19:00-22:00) รายได้สูงกว่าค่าเฉลี่ย 3 เท่า
- มี SLA กับลูกค้า enterprise tier สัญญา 99.9% uptime ปรับ 5,000 บาท/ชม.ที่ downtime ต่อสัญญา (มี 20 สัญญา)
- พนักงาน support/ops 15 คน เงินเดือนเฉลี่ย 40,000 บาท/เดือน (250 บาท/ชม.)

คำนวณ Cost of Downtime ต่อชั่วโมงในช่วง peak:

```
Lost Revenue      = 166,667 × 3           = 500,000 บาท
SLA Penalty       = 5,000 × 20             = 100,000 บาท
Productivity Loss = 15 × 250 × 0.8         =   3,000 บาท
Reputation (est.) ≈ 5% of Lost Revenue     =  25,000 บาท
------------------------------------------------------
รวมโดยประมาณ                                = 628,000 บาท/ชั่วโมง (ช่วง peak)
```

เมื่อเทียบกับต้นทุนของแต่ละ DR Tier (ดู Step 978) องค์กรจะสามารถตัดสินใจได้อย่างมีเหตุผลว่าควรลงทุนเท่าไหร่ ถ้าต้นทุนของ Warm Standby อยู่ที่ 200,000 บาท/เดือน แต่ช่วยลด RTO จาก 4 ชั่วโมง เหลือ 15 นาที นั่นหมายถึงลดความเสียหายลงได้ประมาณ `628,000 × (4 - 0.25) = 2,355,000` บาทต่อครั้งที่เกิดเหตุ ซึ่งคุ้มค่ามากถ้าประเมินว่าจะเกิดเหตุการณ์ระดับ region failure อย่างน้อยปีละ 1 ครั้ง

### 977.4 ตัวอย่างการกำหนด RPO/RTO ตามระดับความสำคัญของระบบ

| Service Tier | ตัวอย่างระบบ | RPO เป้าหมาย | RTO เป้าหมาย | เหตุผล |
|---|---|---|---|---|
| **Tier 0 - Mission Critical** | Payment processing, Order database | ≤ 5 วินาที | ≤ 5 นาที | กระทบรายได้โดยตรงทันที, มี SLA เข้มงวด |
| **Tier 1 - Business Critical** | User authentication, Inventory | ≤ 1 นาที | ≤ 30 นาที | กระทบ user experience รุนแรงแต่ไม่ถึงขั้นเสียรายได้ทันที |
| **Tier 2 - Important** | Notification service, Search | ≤ 15 นาที | ≤ 2 ชั่วโมง | ผู้ใช้พอทนได้ระยะสั้น มี workaround |
| **Tier 3 - Non-critical** | Analytics, Reporting, Data warehouse | ≤ 24 ชั่วโมง | ≤ 24 ชั่วโมง | ไม่กระทบ user-facing operation โดยตรง |

ข้อควรระวัง: **PostgreSQL หนึ่ง instance อาจรองรับหลาย service ที่มี Tier ต่างกัน** ในกรณีนี้ RPO/RTO ของ database ต้องยึดตาม tier ที่สูงสุด (เข้มงวดที่สุด) ของ workload ที่มันรองรับ ซึ่งเป็นเหตุผลหนึ่งที่การแยก database ตาม criticality (database-per-service ใน microservices, เชื่อมโยง Part 091) ช่วยให้ออกแบบ DR ได้คุ้มค่ากว่า ไม่ต้องจ่ายแพงสุดสำหรับทุกอย่าง

### 977.5 RPO ที่แท้จริงของแต่ละกลไก PostgreSQL

| กลไก | RPO ทางทฤษฎี | RPO ในทางปฏิบัติ (มีความหน่วง) |
|---|---|---|
| `pg_dump` รายวัน | 24 ชั่วโมง | 24 ชั่วโมง (คงที่ ไม่มี WAL ป้องกัน) |
| `pg_basebackup` + WAL archiving (PITR) | ตาม `archive_timeout` | มักตั้ง 60 วินาที - 5 นาที |
| Asynchronous Streaming Replication | ใกล้ 0 แต่ไม่การันตี | ขึ้นกับ replication lag จริง (network, load) |
| Synchronous Streaming Replication | 0 (การันตี) | 0 แต่แลกกับ write latency ที่สูงขึ้น |
| Logical Replication | ใกล้ 0 | คล้าย async physical แต่มี overhead เพิ่มจาก decoding |

จุดสำคัญ: RPO ที่ "การันตีได้จริง" มีเพียง synchronous replication เท่านั้น กลไกอื่นทั้งหมดเป็นเพียง "best effort" ซึ่งต้องระบุให้ชัดเจนใน SLA ภายในว่าเป็น RPO แบบ target ไม่ใช่ guarantee

---

## Step 978: DR Tier ต่างๆ

องค์กรส่วนใหญ่อ้างอิงโมเดล 4 tier ซึ่งเป็นมาตรฐานที่ใช้กันแพร่หลายในอุตสาหกรรม (มีที่มาจากแนวคิดของ AWS Well-Architected Framework และ Disaster Recovery Institute International) เรียงจากต้นทุนต่ำสุด/RPO-RTO สูงสุด ไปจนถึงต้นทุนสูงสุด/RPO-RTO ต่ำสุด

### 978.1 ภาพรวมทั้ง 4 Tier

```
ต้นทุนต่ำ, RPO/RTO สูง                                    ต้นทุนสูง, RPO/RTO ต่ำ
     │                                                              │
     ▼                                                              ▼
┌─────────────┐   ┌─────────────┐   ┌─────────────┐   ┌─────────────────────┐
│  Backup &   │   │   Pilot     │   │    Warm     │   │   Multi-Site         │
│  Restore    │ → │   Light     │ → │  Standby    │ → │   Active-Active       │
└─────────────┘   └─────────────┘   └─────────────┘   └─────────────────────┘
  RTO: ชม.-วัน       RTO: 10-30 นาที   RTO: นาที         RTO: วินาที (~0)
  RPO: ชม.-วัน       RPO: นาที          RPO: วินาที       RPO: วินาที-0
```

### 978.2 ตารางเปรียบเทียบรายละเอียด

| คุณสมบัติ | Backup & Restore | Pilot Light | Warm Standby | Multi-Site Active-Active |
|---|---|---|---|---|
| **แนวคิดหลัก** | เก็บ backup ไว้ที่อื่น กู้คืนเมื่อจำเป็น | มี infrastructure ขั้นต่ำ (เช่น DB replica) รันตลอดเวลา ส่วน app servers ปิดไว้ | มี environment เต็มรูปแบบขนาดเล็กรันตลอดเวลา พร้อม scale up เมื่อ failover | มี environment เต็มรูปแบบรันพร้อมกันทั้งสอง (หรือมากกว่า) site และรับ traffic จริง |
| **PostgreSQL setup** | `pg_dump`/`pg_basebackup` + WAL archive ไปยัง object storage ต่าง region | Physical streaming replica (async) ที่ region สำรอง ไม่มี app connect | Physical streaming replica (async/sync) ขนาดเล็กกว่า primary แต่พร้อม promote | Multi-master หรือ Logical replication แบบ bi-directional, หรือใช้ sharding ข้าม region |
| **RPO** | ชั่วโมง - วัน | นาที (ตาม replication lag) | วินาที - นาที | วินาที - 0 (ถ้า sync) |
| **RTO** | ชั่วโมง - วัน (ต้อง provision ใหม่ทั้งหมด) | 10-30 นาที (ต้อง scale app servers ขึ้นมา) | นาที (promote replica + reroute traffic) | เกือบ 0 (traffic switch ที่ DNS/LB level) |
| **ต้นทุนโครงสร้าง** | ต่ำมาก (แค่ storage) | ต่ำ-ปานกลาง (DB instance ขนาดเล็กรันตลอด) | ปานกลาง-สูง (compute รันตลอดแม้ scale ต่ำ) | สูงมาก (compute เต็มรูปแบบ 2 ชุดขึ้นไป + ความซับซ้อนของ conflict resolution) |
| **ความซับซ้อนในการดูแล** | ต่ำ | ปานกลาง | สูง | สูงมาก |
| **เหมาะกับ** | ระบบที่ไม่ critical, งบจำกัด, บริษัทขนาดเล็ก | ระบบ business-critical ที่งบปานกลาง | ระบบ Tier 0-1 ที่ต้องการ RTO ต่ำแต่ยังจ่ายเต็มรูปแบบไม่ไหว | ระบบระดับ global scale ที่ downtime ไม่สามารถยอมรับได้เลย (fintech, healthcare) |
| **ตัวอย่างองค์กรที่ใช้** | Startup, internal tools | SME, SaaS ระดับกลาง | Enterprise, e-commerce ขนาดใหญ่ | Global bank, ตลาดหุ้น, healthcare ระดับชาติ |

### 978.3 รายละเอียดเชิงเทคนิคของแต่ละ Tier

#### Tier 1: Backup & Restore

สถาปัตยกรรม:

```
[Primary DB] --WAL archive--> [Object Storage (S3/GCS) - Cross Region]
                                          |
                                 (เมื่อเกิดภัยพิบัติ)
                                          v
                              [Provision new server]
                                          |
                                          v
                              [Restore from base backup + replay WAL]
                                          |
                                          v
                              [New Primary พร้อมใช้งาน]
```

ข้อดี: ต้นทุนต่ำที่สุด, ดูแลง่าย
ข้อเสีย: RTO สูงมาก เพราะต้องเริ่มจาก 0 ทุกครั้ง (provision hardware, restore ข้อมูลหลาย GB/TB ซึ่งอาจใช้เวลาหลายชั่วโมง)

คำสั่งตัวอย่าง (สรุปจาก Part 061 มาปรับใช้ใน DR context):

```bash
# Restore ที่ region สำรอง
pg_basebackup -h <อ่านจาก object storage ผ่าน WAL-G / pgBackRest> \
  -D /var/lib/postgresql/16/main \
  --progress

# กำหนด recovery target เป็นเวลาก่อนเกิด incident (สำหรับกรณี human error)
cat > /var/lib/postgresql/16/main/postgresql.auto.conf <<EOF
restore_command = 'wal-g wal-fetch %f %p'
recovery_target_time = '2026-09-25 09:44:00+07'
recovery_target_action = 'promote'
EOF

touch /var/lib/postgresql/16/main/recovery.signal
systemctl start postgresql
```

#### Tier 2: Pilot Light

แนวคิด "pilot light" มาจากเตาแก๊สที่มีเปลวไฟเล็กๆ ติดอยู่ตลอดเวลา พร้อมจุดไฟใหญ่ได้ทันทีเมื่อต้องการ

สถาปัตยกรรม:

```
Region A (Primary)                    Region B (DR - Pilot Light)
┌─────────────────┐                   ┌─────────────────┐
│  App Servers x10 │                   │  App Servers x0   │ (AMI/Image พร้อม แต่ไม่รัน)
│  Load Balancer    │                   │  Load Balancer     │ (config พร้อม แต่ไม่ active)
│  PostgreSQL       │--streaming repl-->│  PostgreSQL Replica │ (รันตลอดเวลา, async)
│  Redis Cache       │                   │  Redis (empty)      │
└─────────────────┘                   └─────────────────┘
```

เมื่อเกิดภัยพิบัติ: promote replica เป็น primary → scale app servers ขึ้นจาก 0 เป็น 10 (ใช้ Infrastructure as Code เช่น Terraform + Auto Scaling Group) → เปลี่ยน DNS/LB ชี้มาที่ region B

```bash
# ขั้นตอน promote บน replica (คล้าย Part 065)
psql -c "SELECT pg_promote();"
# หรือ
pg_ctl promote -D /var/lib/postgresql/16/main
```

#### Tier 3: Warm Standby

คล้าย Pilot Light แต่ app servers ที่ region สำรองรันอยู่ตลอดเวลาด้วยขนาดที่เล็กกว่า (เช่น 2 instances แทนที่จะเป็น 10) เมื่อ failover จะ scale ขึ้นอย่างรวดเร็ว database replica มักตั้งเป็น synchronous หรือ semi-synchronous เพื่อลด RPO ให้ใกล้ 0 มากที่สุด

```conf
# postgresql.conf ที่ Primary (Region A)
synchronous_standby_names = 'ANY 1 (region_b_replica)'
synchronous_commit = remote_apply
```

ข้อควรระวังสำคัญ: **synchronous replication ข้าม region มีผลกระทบต่อ write latency โดยตรง** เพราะทุก commit ต้องรอ acknowledge จาก replica ที่อยู่ไกล (เช่น ข้าม continent อาจมี network latency 100-200ms) ซึ่งจะทำให้ throughput ลดลงมาก ในทางปฏิบัติจึงมักใช้ `remote_write` แทน `remote_apply` หรือใช้ **quorum-based synchronous** กับ replica หลายตัวที่ region ต่างๆ เพื่อ balance ระหว่างความปลอดภัยของข้อมูลกับ performance

#### Tier 4: Multi-Site Active-Active

ทุก site รับ traffic จริงพร้อมกัน ต้องแก้ปัญหา **write conflict** เพราะ PostgreSQL โดยธรรมชาติไม่ใช่ multi-master database เทคนิคที่ใช้ได้แก่:

1. **Application-level partitioning** — แบ่ง write ตาม region เช่น user ในเอเชียเขียนที่ region เอเชีย, user ในยุโรปเขียนที่ region ยุโรป (sharding ตาม geography) แล้วใช้ logical replication แบบ bi-directional (BDR - Bi-Directional Replication, หรือ extension เช่น pglogical) เพื่อ sync ข้อมูลข้าม region สำหรับ read
2. **Conflict-free replicated data types (CRDTs)** — สำหรับ use case เฉพาะที่ conflict resolution ทำได้แบบ deterministic
3. **Distributed SQL layer** — ใช้ PostgreSQL-compatible distributed database เช่น CockroachDB, YugabyteDB, หรือ Citus ที่ออกแบบมาสำหรับ multi-region เขียนพร้อมกันโดยเฉพาะ (นอกเหนือขอบเขต vanilla PostgreSQL)

ต้นทุนและความซับซ้อนสูงมาก จึงเหมาะกับองค์กรระดับ global เท่านั้น ส่วนใหญ่ของระบบ e-commerce SaaS ระดับกลาง-ใหญ่ (เช่นตัวอย่างใน Part 080) จะเลือกใช้ **Warm Standby** เป็นจุดสมดุลที่คุ้มค่าที่สุด

### 978.4 กรอบการตัดสินใจเลือก Tier

```
                        Cost of Downtime สูงมาก?
                              │
                ┌─────────No──┴──Yes────┐
                │                        │
       RPO/RTO ยอมรับได้                มี budget สำหรับ
       เป็นชั่วโมง/วัน?                   compute คู่ขนานเต็มรูปแบบ?
                │                        │
        ┌───Yes─┴─No───┐         ┌───No──┴──Yes───┐
        │              │         │                │
   Backup &        Pilot     Warm Standby    Multi-Site
   Restore         Light                    Active-Active
```

---

## Step 979: Multi-Region Strategy

### 979.1 ทำไมต้อง Multi-Region

Cloud region คือหน่วยความล้มเหลวที่ใหญ่ที่สุดหน่วยหนึ่งที่ provider รับประกันความเป็นอิสระจากกัน (แต่ไม่ 100%) หากระบบทั้งหมดอยู่ใน region เดียว ไม่ว่าจะกระจาย availability zone (AZ) กี่ zone ก็ตาม ก็ยังมีความเสี่ยงที่ region นั้นทั้งหมดล่ม (เกิดขึ้นจริงหลายครั้ง เช่น เหตุการณ์ network configuration ผิดพลาดที่ทำให้ทั้ง region ใช้งานไม่ได้)

### 979.2 สถาปัตยกรรม Multi-Region สำหรับ PostgreSQL

เชื่อมโยงกับ Part 063 (Physical Replication) โดยตรง — เราใช้หลักการเดียวกัน แต่ปรับ configuration ให้เหมาะกับ network ข้าม region ที่มี latency สูงกว่าและไม่เสถียรเท่าภายใน DC เดียวกัน

```
   Region: ap-southeast-1 (Primary)          Region: ap-northeast-1 (DR Standby)
┌─────────────────────────────┐          ┌─────────────────────────────┐
│  AZ-1a          AZ-1b         │          │  AZ-1a          AZ-1b         │
│ ┌─────────┐  ┌─────────┐      │          │ ┌─────────┐  ┌─────────┐      │
│ │ Primary  │  │Sync      │      │  WAL     │ │ Async     │  │ (standby   │      │
│ │  DB      │─>│Standby   │──────┼────────>│ │ Standby   │  │ of standby)│      │
│ └─────────┘  └─────────┘      │ streaming│ └─────────┘  └─────────┘      │
│    (HA - local failover)       │          │    (DR - regional failover)   │
└─────────────────────────────┘          └─────────────────────────────┘
```

หลักการออกแบบสำคัญ:

1. **แยกชั้นความรับผิดชอบ**: local synchronous standby (ภายใน region เดียวกัน) รับผิดชอบ HA (failover เร็ว, zero data loss) ส่วน cross-region standby รับผิดชอบ DR (failover ช้ากว่าได้ แต่ป้องกัน region-level failure)
2. **ใช้ asynchronous replication ข้าม region เกือบทุกกรณี** เนื่องจาก synchronous ข้าม region จะทำให้ write latency สูงเกินยอมรับได้ (ยกเว้นกรณี Tier 0 ที่ยอมจ่ายราคานี้จริงๆ)
3. **ตั้ง cascading replication** เพื่อลด load บน primary — standby ที่ region ปลายทางสามารถมี standby ของตัวเองต่อได้อีกชั้น (feature ที่ PostgreSQL รองรับมาตั้งแต่ 9.2)

### 979.3 ตัวอย่าง Configuration

**Primary (Region A) — `postgresql.conf`:**

```conf
wal_level = replica
max_wal_senders = 10
wal_keep_size = 4GB          # กันไว้เผื่อ standby ข้าม region หลุด connection ชั่วคราว
archive_mode = on
archive_command = 'wal-g wal-push %p'   # archive ไว้ต่างหากด้วย เพื่อเป็น Tier-1 backup

# synchronous กับ local standby เท่านั้น
synchronous_standby_names = 'FIRST 1 (local_standby_az1b)'
```

**Standby ข้าม Region (Region B) — `postgresql.conf` / `primary_conninfo`:**

```conf
primary_conninfo = 'host=primary-region-a.internal port=5432 user=replicator
                     password=... sslmode=verify-full
                     application_name=dr_standby_region_b
                     connect_timeout=10
                     keepalives=1 keepalives_idle=30 keepalives_interval=10 keepalives_count=5'

# เผื่อ WAL streaming หลุด ให้ fallback ไป fetch จาก archive
restore_command = 'wal-g wal-fetch %f %p'

primary_slot_name = 'dr_standby_region_b_slot'
hot_standby = on
hot_standby_feedback = off   # ปิดไว้เพื่อไม่ให้ standby ที่ไกลกระทบ vacuum บน primary
```

ข้อควรระวังเรื่อง `hot_standby_feedback`: ถ้าเปิดไว้ standby ที่มี lag สูง (เพราะ network ข้าม region) จะทำให้ primary ไม่สามารถ vacuum ทำความสะอาด dead tuple ได้ เกิด table bloat สะสม ควรปิดสำหรับ cross-region DR standby และใช้ `max_standby_streaming_delay` ควบคุม query conflict แทน

### 979.4 Replication Slot กับความเสี่ยงข้ามภูมิภาค

การใช้ physical replication slot ช่วยป้องกันไม่ให้ primary ลบ WAL ที่ standby ยังไม่ได้รับไป แต่มีความเสี่ยงตรงกันข้าม: ถ้า cross-region standby หลุดการเชื่อมต่อเป็นเวลานาน (เช่น network ข้าม region มีปัญหาหลายชั่วโมง) WAL จะสะสมค้างอยู่บน primary จนอาจทำให้ disk เต็มและ **primary ล่มไปด้วย** — นี่คือความเสี่ยงร้ายแรงที่ DR standby ไปกระทบ production เสียเอง

แนวทางแก้ไข:

```sql
-- ตรวจสอบ WAL ที่ค้างอยู่เพราะ slot ไม่ถูกใช้
SELECT slot_name, active,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS retained_wal
FROM pg_replication_slots
ORDER BY retained_wal DESC;
```

ตั้ง alert เมื่อ `retained_wal` เกิน threshold (เช่น 10GB) และมีนโยบายชัดเจนว่าถ้า cross-region standby หลุดเกิน N ชั่วโมง จะ **drop slot และ re-seed ใหม่จาก backup** แทนที่จะปล่อยให้ primary เสี่ยง disk เต็ม — สลับมาใช้ `restore_command` จาก archive แทนในระหว่างนั้น (นี่คือเหตุผลที่ควรมีทั้ง streaming replication และ WAL archiving คู่กันเสมอสำหรับ DR standby)

### 979.5 การเลือก Region คู่ (Region Pairing)

หลักเกณฑ์การเลือก region สำรอง:

| ปัจจัย | คำแนะนำ |
|---|---|
| **ระยะทาง/ภูมิศาสตร์** | ควรอยู่ต่าง fault domain กันจริง (ไม่ใช่แค่ต่างชื่อ region แต่อยู่ grid ไฟฟ้าเดียวกัน) |
| **Latency** | ยิ่งไกลยิ่ง lag สูง ต้องประเมิน RPO ที่ยอมรับได้จริงจาก latency นั้น |
| **กฎหมาย/Data residency** | ข้อมูลลูกค้าบางประเภท (เช่น PII ของ EU ตาม GDPR) อาจห้ามเก็บนอกภูมิภาค ต้องเลือก region สำรองที่ยังอยู่ใน jurisdiction ที่อนุญาต |
| **ต้นทุนโอนข้อมูลข้าม region** | Cloud provider คิดค่า data transfer ข้าม region ซึ่งอาจสูงมากสำหรับ database ขนาดใหญ่ที่มี write throughput สูง |
| **ความพร้อมของบริการ** | Region สำรองต้องมีบริการที่จำเป็นครบ (managed PostgreSQL, object storage, CDN ฯลฯ) |

---

## Step 980: DR Runbook

### 980.1 Runbook คืออะไร และทำไมต้อง "ไม่กำกวม"

**DR Runbook** คือเอกสารขั้นตอนปฏิบัติที่ละเอียดพอให้ **วิศวกรที่ไม่เคยเห็นระบบนี้มาก่อน** สามารถทำตามได้สำเร็จภายใต้ความกดดันสูงและเวลาตี 3 ของเช้าวันหยุด หลักการเขียน runbook ที่ดี:

1. **ไม่กำกวม (Unambiguous)**: ทุกขั้นตอนต้องมีคำสั่งที่ copy-paste ได้จริง ไม่ใช่คำอธิบายเชิงแนวคิด เช่นไม่เขียนว่า "restore database จาก backup" แต่ต้องเขียนคำสั่งเต็มพร้อม parameter จริง
2. **มีเงื่อนไขการตัดสินใจชัดเจน (Decision Tree)**: ระบุว่าถ้าเจอสถานการณ์ A ให้ทำ B ถ้าเจอสถานการณ์ C ให้ทำ D
3. **ระบุผู้รับผิดชอบแต่ละขั้นตอน**: ใครเป็นคนกด, ใครเป็นคนอนุมัติ (โดยเฉพาะขั้นตอนที่ทำลายข้อมูลเก่าหรือเปลี่ยน DNS production)
4. **มี rollback plan**: ถ้าขั้นตอนใดล้มเหลวกลางทาง ต้องรู้ว่าจะย้อนกลับอย่างไร
5. **Version control**: runbook ต้องเก็บใน git และปรับปรุงทุกครั้งที่ระบบเปลี่ยน ไม่ใช่เอกสารที่เขียนครั้งเดียวแล้วทิ้งไว้
6. **เข้าถึงได้แม้ระบบหลักล่ม**: ห้ามเก็บ runbook ไว้บนระบบเดียวกับที่กำลังจะล่ม (เช่น wiki internal ที่ host อยู่บน infrastructure เดียวกับ production) ต้องมีสำเนาที่เข้าถึงได้จากภายนอกเสมอ (เช่น พิมพ์เป็น PDF เก็บใน password manager ที่แยก infrastructure, หรือ static site บน CDN คนละ provider)

### 980.2 ตัวอย่าง DR Runbook ฉบับเต็ม

ด้านล่างคือตัวอย่าง runbook จริงสำหรับสถานการณ์ **"Primary Region ล่มทั้ง region"** ของระบบ e-commerce SaaS (อ้างอิงสถาปัตยกรรมจาก Part 080)

```markdown
# DR RUNBOOK: Primary Region Failure (ap-southeast-1 Total Outage)

Document ID: DR-RB-001
Version: 3.2
Last Updated: 2026-09-01
Last Tested: 2026-08-15 (Game Day #7 - PASSED, RTO actual = 22 min)
Owner: Database Reliability Team (DRI: on-call SRE)
Approval Required From: Incident Commander + VP Engineering (for step 6 only)

## 1. ACTIVATION CRITERIA (เมื่อไหร่ต้องใช้ runbook นี้)

ให้เริ่ม runbook นี้เมื่อเข้าเงื่อนไขข้อใดข้อหนึ่ง:

- [ ] Health check ของ ap-southeast-1 ล้มเหลวติดต่อกัน > 5 นาที จาก 3 monitoring
      location ที่แตกต่างกัน (ป้องกัน false positive จาก network ฝั่งเรา)
- [ ] Cloud provider status page ประกาศ "Service Disruption" หรือสูงกว่า
      สำหรับ ap-southeast-1
- [ ] ทีม on-call ยืนยันด้วยตนเองว่าเชื่อมต่อ primary database และ app
      servers ทั้งหมดใน region ไม่ได้ ผ่าน 2 ช่องทางขึ้นไป (SSH, cloud console)

## 2. ทีมที่ต้องแจ้งทันที (ดู Step 983 - Communication Plan สำหรับรายละเอียด)

| ลำดับ | บุคคล/ทีม | ช่องทาง | เวลาที่ต้องแจ้งภายใน |
|---|---|---|---|
| 1 | Incident Commander (on-call) | PagerDuty escalation | ทันที (auto) |
| 2 | Database Reliability Team | Slack #incident-dr + phone call | ภายใน 2 นาที |
| 3 | VP Engineering | Phone call โดยตรง | ภายใน 5 นาที |
| 4 | Customer Support Lead | Slack #incident-dr | ภายใน 5 นาที |
| 5 | ลูกค้า (ผ่าน status page) | statuspage.io auto-update | ภายใน 10 นาที |

## 3. PRE-FLIGHT CHECKS (ก่อนเริ่ม failover จริง - ใช้เวลา ~3 นาที)

```bash
# 3.1 ยืนยันสถานะ DR standby ที่ ap-northeast-1 ว่ายังทำงานปกติ
ssh dr-standby.ap-northeast-1.internal
psql -U postgres -c "SELECT pg_is_in_recovery();"
# คาดหวังผลลัพธ์: t (true = ยังเป็น standby อยู่)

# 3.2 ตรวจสอบ replication lag ล่าสุดก่อนเกิดเหตุ (จาก monitoring dashboard
#     ที่แยก infrastructure - Grafana Cloud)
#     เปิด: https://<org>.grafana.net/d/dr-lag-dashboard
#     บันทึกค่า lag ล่าสุดที่เห็น ลงใน incident doc เพื่อประเมิน data loss

# 3.3 ตรวจสอบว่า standby รับ WAL ล่าสุดถึงเมื่อไหร่
psql -U postgres -c "SELECT pg_last_wal_receive_lsn(), pg_last_wal_replay_lsn(),
  pg_last_xact_replay_timestamp();"
```

**Decision point**: ถ้า `pg_last_xact_replay_timestamp()` เก่ากว่าเวลาปัจจุบันเกิน
15 นาที (RPO เป้าหมายของ Tier 0) → แจ้ง Incident Commander ทันทีว่า data loss
จะเกินเป้าหมาย RPO ก่อนดำเนินการต่อ เพื่อให้ผู้บริหารรับทราบความเสี่ยงล่วงหน้า

## 4. PROMOTE DR STANDBY เป็น PRIMARY

```bash
# 4.1 บน dr-standby.ap-northeast-1
sudo -u postgres pg_ctl promote -D /var/lib/postgresql/16/main

# 4.2 ยืนยันว่า promote สำเร็จ (รอไม่เกิน 30 วินาที)
watch -n2 'psql -U postgres -c "SELECT pg_is_in_recovery();"'
# ต้องเปลี่ยนจาก t เป็น f

# 4.3 ตรวจสอบว่า database รับ write ได้จริง
psql -U postgres -d ecommerce_prod -c \
  "INSERT INTO dr_test_log(tested_at, note) VALUES (now(), 'promote verified');"
```

## 5. อัปเดต APPLICATION LAYER ให้ชี้ไปที่ region ใหม่

```bash
# 5.1 อัปเดต connection string ใน secrets manager
aws secretsmanager update-secret \
  --secret-id prod/database/primary-endpoint \
  --secret-string '{"host":"dr-standby.ap-northeast-1.internal","port":5432}' \
  --region ap-northeast-1

# 5.2 Scale app servers ที่ region สำรองจาก pilot (2 instances) ขึ้นเป็นเต็มรูปแบบ
aws autoscaling set-desired-capacity \
  --auto-scaling-group-name app-servers-ap-northeast-1 \
  --desired-capacity 10 --region ap-northeast-1

# 5.3 รอ health check ของ app servers ผ่าน (ประมาณ 3-5 นาที)
aws elbv2 describe-target-health \
  --target-group-arn <arn> --region ap-northeast-1
```

## 6. เปลี่ยนเส้นทาง TRAFFIC (ต้องได้รับอนุมัติจาก Incident Commander)

**ขั้นตอนนี้เป็น point of no return ทางธุรกิจ (ลูกค้าจะเห็นผลกระทบ) ต้องได้
รับการยืนยันด้วยวาจาจาก Incident Commander ก่อนรันคำสั่งนี้เท่านั้น**

```bash
# 6.1 เปลี่ยน DNS record ให้ชี้ไปที่ load balancer ของ region ใหม่
#     (TTL ถูกตั้งไว้ล่วงหน้าที่ 60 วินาทีเพื่อให้ propagate เร็ว - ดู Step 980.4)
aws route53 change-resource-record-sets \
  --hosted-zone-id <ZONE_ID> \
  --change-batch file://failover-dns-changeset.json

# 6.2 ยืนยันการ propagate จากหลาย location
dig +short api.example.com @8.8.8.8
dig +short api.example.com @1.1.1.1
```

## 7. VALIDATION (ตรวจสอบว่าระบบใช้งานได้จริงก่อนประกาศ resolved)

- [ ] ทดสอบ end-to-end: สร้าง order ทดสอบผ่าน API จริง แล้วตรวจสอบว่าบันทึกใน
      database ใหม่สำเร็จ
- [ ] ตรวจสอบ error rate ใน APM (Datadog/New Relic) ว่ากลับสู่ระดับปกติ
- [ ] ตรวจสอบว่า background jobs / queue workers เชื่อมต่อ database ใหม่ได้
- [ ] แจ้ง Customer Support Lead ว่าระบบกลับมาใช้งานได้แล้ว เพื่ออัปเดต
      status page เป็น "Resolved" (แต่ยังคง monitor ต่อ)

## 8. POST-FAILOVER TASKS (ภายใน 24 ชั่วโมงถัดไป)

- [ ] ตั้งค่า region เดิม (ap-southeast-1) เป็น standby ตัวใหม่เมื่อกลับมา
      ใช้งานได้ (เตรียมสำหรับ failback ในอนาคต)
- [ ] เริ่มกระบวนการ Postmortem (ดู Step 984) ภายใน 48 ชั่วโมง
- [ ] อัปเดต runbook นี้หากพบขั้นตอนที่ไม่ถูกต้องหรือขาดหายระหว่างทำจริง

## 9. FAILBACK PLAN (เมื่อ region เดิมกลับมาใช้งานได้ปกติ)

การ failback ต้องวางแผนล่วงหน้าและทำในช่วง low-traffic window เท่านั้น
ไม่ทำแบบเร่งด่วนเหมือนตอน failover เพราะระบบใช้งานได้อยู่แล้ว ไม่มีความเร่งด่วน
ดูเอกสารแยก: DR-RB-002 "Planned Failback Procedure"
```

### 980.3 หลักการสำคัญที่สังเกตได้จาก Runbook ตัวอย่าง

1. **มี Activation Criteria ชัดเจน** — ป้องกันการ "over-react" กับปัญหาเล็กๆ ที่ไม่ใช่ region failure จริง
2. **มี Decision Point ที่ระบุเงื่อนไขตัวเลขชัดเจน** (เช่น "เกิน 15 นาที") ไม่ใช่คำพูดกำกวมเช่น "ถ้า lag เยอะเกินไป"
3. **แยกขั้นตอนที่ต้องขออนุมัติ** ออกจากขั้นตอนที่ on-call ทำได้ทันที — โดยเฉพาะขั้นตอนเปลี่ยน DNS ที่กระทบลูกค้าโดยตรงและอาจเป็น point of no return
4. **มี validation step ก่อนประกาศ resolved** — ป้องกันการประกาศเร็วเกินไปว่าปัญหาหมดแล้วทั้งที่ระบบยังไม่เสถียรจริง
5. **มี failback plan แยกต่างหาก** — เพราะ failback ไม่เร่งด่วนเท่า failover จึงควรทำอย่างระมัดระวังกว่า ไม่ใช้ runbook เดียวกัน

### 980.4 รายละเอียดทางเทคนิคที่มักถูกมองข้ามใน Runbook

**DNS TTL**: ต้องตั้ง TTL ของ record ที่จะใช้ตอน failover ให้ต่ำ (เช่น 60 วินาที) **ล่วงหน้า** ไม่ใช่มาตั้งตอนเกิดเหตุ เพราะ TTL เดิมอาจสูง (เช่น 24 ชั่วโมง) ทำให้ client บางส่วนยัง cache DNS เดิมอยู่นานหลังจาก failover สำเร็จแล้ว

**Connection pooler**: ถ้าใช้ PgBouncer/Pgpool (เชื่อมโยง Part 062) ต้องระบุใน runbook ว่าต้อง restart หรือ reload pooler config ด้วยหลัง endpoint เปลี่ยน ไม่เช่นนั้น connection เก่าที่ pool ไว้จะยังชี้ไปที่ database เดิมที่ล่มอยู่

**Sequence/Identity สำหรับ write conflict**: ถ้ามีความเป็นไปได้ที่ region เดิมจะกลับมา online และมี write หลุดเข้าไปโดยไม่ได้ตั้งใจ (split-brain) ต้องมีกลไกป้องกัน เช่น STONITH (Shoot The Other Node In The Head) หรือ fencing เพื่อปิดไม่ให้ region เดิมรับ write ได้จนกว่าจะยืนยันว่าปลอดภัย — นี่คือประเด็นเดียวกับที่พูดใน Part 065 เรื่อง split-brain prevention

---

## Step 981: DR Testing

### 981.1 ทำไมแผนที่ไม่เคยทดสอบเชื่อถือไม่ได้

ใน Part 061 Step 608 เราได้เรียนหลักการสำคัญไว้ว่า **"Backup ที่ไม่เคยทดสอบ restore ไม่ใช่ backup"** หลักการเดียวกันนี้ขยายไปสู่ DR ทั้งระบบ: **"DR plan ที่ไม่เคยทดสอบจริง ไม่ใช่ DR plan แต่เป็นเอกสารสมมติฐาน"**

เหตุผลที่ runbook ที่เขียนไว้เฉยๆ มักใช้ไม่ได้จริงเมื่อถึงเวลาจริง:

1. **Infrastructure เปลี่ยนแปลงตลอดเวลา** — server ใหม่ถูกเพิ่ม, config เปลี่ยน, แต่ runbook ไม่ได้อัปเดตตาม
2. **คำสั่งที่เขียนไว้อาจมี syntax error หรือ parameter ผิด** ที่ไม่มีใครสังเกตจนกว่าจะรันจริง
3. **สมมติฐานที่ผิด** เช่น คิดว่า standby มี data ล่าสุดเสมอ แต่จริงๆ replication หยุดทำงานไปแล้วหลายวันโดยไม่มีใครสังเกต (alert เสีย)
4. **ทีมงานไม่คุ้นเคยกับขั้นตอน** — ภายใต้ความกดดันสูง คนที่ไม่เคยฝึกซ้อมมักทำผิดพลาดหรือช้ากว่าที่ควร แม้จะมี runbook อยู่ตรงหน้า

### 981.2 Game Day คืออะไร

**Game Day** คือการจำลองสถานการณ์ภัยพิบัติแบบควบคุม (controlled simulation) โดยทีมงานจริงทำตาม runbook จริงบน environment ที่ใกล้เคียง production มากที่สุด (หรือใน production เองในบางกรณีที่องค์กรมีความเชื่อมั่นสูง เช่น Netflix)

**ขั้นตอนการจัด Game Day:**

1. **วางแผนล่วงหน้า** (1-2 สัปดาห์ก่อน): กำหนดวันเวลา, สถานการณ์ที่จะจำลอง, ผู้เข้าร่วม, ขอบเขต (staging หรือ production), เกณฑ์วัดความสำเร็จ
2. **แจ้งผู้ที่เกี่ยวข้อง** (ยกเว้นในกรณี chaos engineering แบบไม่แจ้งล่วงหน้า ซึ่งเป็นระดับที่สูงกว่า)
3. **ดำเนินการจำลองเหตุการณ์**: เช่น ปิด region จริง (ใน staging), หรือ block network traffic ไปยัง primary database
4. **ทีม on-call ทำตาม runbook จริง** โดยมีผู้สังเกตการณ์ (observer) จับเวลาและบันทึกทุกขั้นตอน ไม่ช่วยเหลือเว้นแต่จะติดขัดจริงๆ
5. **วัดผล**: RTO จริงที่ทำได้เทียบกับเป้าหมาย, RPO จริง (data loss ที่เกิดขึ้นจากการจำลอง), จุดที่ runbook ผิดพลาดหรือขาดหาย
6. **สรุปและปรับปรุง runbook** ทันทีหลังจบ Game Day ขณะที่ความจำยังสด

**ตารางตัวอย่างการวางแผน Game Day:**

| หัวข้อ | รายละเอียด |
|---|---|
| ชื่อสถานการณ์ | GD-2026-08: Primary Region Total Failure |
| Environment | Staging (จำลองสถาปัตยกรรมเหมือน production 100%) |
| วันเวลา | วันเสาร์ 10:00-12:00 น. (นอกเวลาทำการเพื่อลดผลกระทบถ้าเกิดปัญหาจริง) |
| ผู้เข้าร่วม | On-call SRE (ผู้ปฏิบัติ), DBA Lead (observer), Incident Commander (ทดสอบการตัดสินใจ) |
| วิธีจำลอง | Security Group ปิดกั้น network traffic ทั้งหมดไปยัง primary region |
| เป้าหมาย RTO | ≤ 30 นาที |
| เกณฑ์ผ่าน | RTO จริง ≤ 30 นาที และ data loss ≤ RPO เป้าหมาย (15 นาที) และไม่มี manual step ที่ runbook ไม่ได้ระบุไว้ |

### 981.3 Chaos Engineering เบื้องต้น

**Chaos Engineering** คือแนวคิดที่ก้าวไปอีกขั้นจาก Game Day แบบวางแผนล่วงหน้า โดยจงใจสร้างความล้มเหลวแบบสุ่มหรือกึ่งสุ่มใน production (หรือ production-like environment) อย่างสม่ำเสมอ เพื่อค้นหาจุดอ่อนที่ทีมงานไม่ทันคาดคิดว่าจะมีปัญหา แนวคิดนี้ริเริ่มโดย Netflix ผ่านเครื่องมือชื่อ "Chaos Monkey"

**หลักการพื้นฐาน 4 ข้อของ Chaos Engineering:**

1. **ตั้งสมมติฐานเกี่ยวกับสภาวะปกติ (steady state)** ของระบบก่อน เช่น "error rate ต้องต่ำกว่า 0.1%"
2. **ตั้งสมมติฐานว่าระบบจะรักษาสภาวะปกตินั้นได้แม้เกิดความล้มเหลว** เช่น "ถ้า replica ตัวหนึ่งหายไป error rate จะยังต่ำกว่า 0.1%"
3. **จำลองเหตุการณ์ในโลกจริง**: kill process, ตัด network, เพิ่ม latency เทียม, ทำให้ disk เต็ม
4. **พยายามพิสูจน์ว่าสมมติฐานผิด** — ถ้าเจอว่าระบบพังจริงเมื่อจำลองเหตุการณ์ นั่นคือจุดที่ต้องแก้ไขก่อนเกิดเหตุจริง

**ตัวอย่างการทดลอง Chaos เบื้องต้นสำหรับ PostgreSQL:**

```bash
# ทดลองที่ 1: จำลอง replica lag สูงกะทันหัน (network throttling)
# ใช้ tc (traffic control) บน Linux เพื่อจำลอง latency สูงระหว่าง
# primary กับ standby
tc qdisc add dev eth0 root netem delay 2000ms

# สังเกต: application ที่อ่านจาก replica (read replica pattern จาก
# Part 062) ยัง fallback ไปอ่านจาก primary ได้หรือไม่? หรือ error ออกมา?

# ยกเลิกการจำลอง
tc qdisc del dev eth0 root netem

# ทดลองที่ 2: จำลอง primary connection หายกะทันหัน
sudo iptables -A INPUT -p tcp --dport 5432 -j DROP
# สังเกต: connection pooler (PgBouncer) จัดการ reconnect หรือ fail
# gracefully หรือไม่? Patroni ตรวจจับและ failover ตามเวลาที่ตั้งไว้จริงหรือไม่?
sudo iptables -D INPUT -p tcp --dport 5432 -j DROP

# ทดลองที่ 3: จำลอง disk เต็มบน standby (ทดสอบผลกระทบต่อ replication)
fallocate -l 95% /var/lib/postgresql/16/main/dummy_fill_file
# สังเกต: alert ทำงานทันหรือไม่? replication หยุดแบบ graceful หรือ
# corrupt ข้อมูล?
rm /var/lib/postgresql/16/main/dummy_fill_file
```

**ข้อควรระวังสำคัญสำหรับมือใหม่**: ห้ามเริ่ม chaos engineering บน production โดยตรงทันที ต้องเริ่มจาก staging ก่อนเสมอ, มี "kill switch" ที่หยุดการทดลองได้ทันทีถ้าเกิดผลกระทบเกินคาด, และต้องมี blast radius ที่จำกัด (เช่น ทดลองกับ traffic 1% ก่อน ไม่ใช่ 100% ทันที)

### 981.4 ความถี่ในการทดสอบ DR

| ประเภทการทดสอบ | ความถี่แนะนำ |
|---|---|
| Backup restore verification (Part 061 Step 608) | อัตโนมัติทุกวัน |
| Tabletop exercise (พูดคุยทบทวน runbook บนกระดาษ ไม่รันจริง) | ทุกไตรมาส |
| Game Day บน staging | ทุก 6 เดือน |
| Game Day บน production (สำหรับองค์กรที่พร้อม) | ปีละ 1 ครั้ง |
| Chaos Engineering experiments เล็กๆ | ทุกสัปดาห์/เดือน (อัตโนมัติ) |
| ทบทวนและอัปเดต runbook | ทุกครั้งที่ infrastructure เปลี่ยนแปลงสำคัญ + ทุกไตรมาสเป็นอย่างน้อย |

---

## Step 982: Business Continuity Planning (BCP)

### 982.1 BCP กับ DR ต่างกันอย่างไร

**Disaster Recovery (DR)** เน้นที่การกู้คืน **ระบบ IT** ให้กลับมาทำงานได้

**Business Continuity Planning (BCP)** เป็นแนวคิดที่กว้างกว่ามาก ครอบคลุม **การดำเนินธุรกิจต่อไปได้** แม้ระบบ IT (หรือแม้แต่ office, พนักงาน, supplier) จะได้รับผลกระทบ DR เป็นเพียง "ส่วนประกอบหนึ่ง" ของ BCP เท่านั้น

```
┌──────────────────────────────────────────────────────────┐
│              Business Continuity Planning (BCP)             │
│  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐  │
│  │  IT Disaster   │  │   Workforce    │  │   Vendor/      │  │
│  │  Recovery (DR)  │  │  Continuity    │  │   Supply Chain │  │
│  │                 │  │ (คนทำงานที่ไหน  │  │   Continuity   │  │
│  │ (ระบบ database, │  │  ได้บ้างถ้า     │  │ (ถ้า vendor    │  │
│  │  server, network)│  │  office ใช้ไม่ได้)│  │  หลักมีปัญหา)  │  │
│  └───────────────┘  └───────────────┘  └───────────────┘  │
│  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐  │
│  │   Financial     │  │  Legal &        │  │  Crisis         │  │
│  │   Continuity    │  │  Compliance     │  │  Communication  │  │
│  │ (เงินสำรอง,      │  │ (ภาระผูกพัน      │  │ (แจ้งลูกค้า,     │  │
│  │  ประกันภัย)       │  │  ทางกฎหมาย)     │  │  สื่อ, นักลงทุน) │  │
│  └───────────────┘  └───────────────┘  └───────────────┘  │
└──────────────────────────────────────────────────────────┘
```

### 982.2 องค์ประกอบหลักของ BCP

1. **Business Impact Analysis (BIA)** — วิเคราะห์ว่าฟังก์ชันทางธุรกิจไหนสำคัญที่สุด และผลกระทบทางการเงิน/ชื่อเสียงหากหยุดชะงัก (เชื่อมโยงกับการคำนวณ RPO/RTO ใน Step 977.3)
2. **Risk Assessment** — ระบุความเสี่ยงทั้งหมดที่อาจกระทบธุรกิจ ไม่ใช่แค่ความเสี่ยงด้าน IT เช่น ความเสี่ยงจากการที่พนักงานหลักลาออกพร้อมกัน, ความเสี่ยงจาก supplier รายเดียวที่ผูกขาด
3. **Continuity Strategies** — แผนสำรองสำหรับแต่ละความเสี่ยง เช่น ถ้า office หลักใช้งานไม่ได้ พนักงานจะ work from home หรือย้ายไป office สำรองได้ทันที
4. **Plan Development** — เอกสารแผนที่เป็นทางการ รวม DR Runbook เป็นส่วนหนึ่ง
5. **Testing & Maintenance** — ทดสอบสม่ำเสมอเหมือนที่อธิบายใน Step 981 แต่ขยายขอบเขตไปถึงการซ้อมอพยพ, การซ้อมสื่อสารในภาวะวิกฤต

### 982.3 ตัวอย่างสถานการณ์ที่ DR อย่างเดียวไม่พอ แต่ต้องใช้ BCP

| สถานการณ์ | ทำไม DR อย่างเดียวไม่พอ | สิ่งที่ BCP ต้องครอบคลุมเพิ่ม |
|---|---|---|
| พนักงาน DBA หลักทั้งทีมติดโรคระบาดพร้อมกัน | ระบบ IT ยังทำงานได้ปกติ (ไม่มีปัญหาทาง technical) แต่ไม่มีคนดูแล | แผน cross-training, สัญญากับ managed service provider สำรอง, เอกสาร runbook ที่ทีมอื่นทำตามได้ |
| Payment gateway partner ล้มละลายกะทันหัน | Database เราไม่มีปัญหาเลย แต่ธุรกิจหยุดรับเงินไม่ได้ | แผนสำรอง payment provider ตัวที่สอง, สัญญา SLA กับ vendor หลายราย |
| สำนักงานใหญ่ถูกน้ำท่วม พนักงานเข้าออฟฟิศไม่ได้ | Cloud infrastructure (รวม DR site) ยังทำงานได้ แต่ทีมงานไม่มีที่นั่งทำงาน | นโยบาย remote work ฉุกเฉิน, งบสำรองสำหรับอุปกรณ์/internet ที่บ้าน |
| Ransomware เข้ารหัสทั้ง production และ office network (ไม่ใช่แค่ database) | DR restore database ได้ แต่ email, HR system, การสื่อสารภายในหยุดหมด | แผนสื่อสารสำรองที่ไม่พึ่ง infrastructure เดิม (เช่น Signal group แยกต่างหาก), incident response ระดับองค์กร |

### 982.4 ใครควรเป็นเจ้าของ BCP ในองค์กร

ต่างจาก DR ที่มักเป็นความรับผิดชอบของทีม Engineering/SRE/DBA โดยตรง **BCP ควรมีเจ้าของระดับ C-level หรือมีหน่วยงานเฉพาะ** (เช่น Chief Risk Officer, Business Continuity Manager) เพราะครอบคลุมทุกแผนกไม่ใช่แค่ IT ทีม Database/Engineering ทำหน้าที่เป็น **ผู้สนับสนุนข้อมูลทางเทคนิค** ให้กับแผน BCP ขององค์กร โดยเฉพาะส่วน DR ที่เป็นความเชี่ยวชาญโดยตรง

---

## Step 983: Communication Plan ระหว่างเกิดเหตุ

### 983.1 ทำไม Communication Plan สำคัญพอๆ กับ Technical Plan

จากประสบการณ์ของหลายองค์กรที่ผ่านเหตุการณ์ major incident มา ปัญหาที่พบบ่อยไม่แพ้ปัญหาทางเทคนิคคือ **ความสับสนในการสื่อสาร**: ทีมงานหลายทีมพยายามแก้ปัญหาเดียวกันโดยไม่รู้ตัว, ผู้บริหารไม่ทราบสถานะจนกว่าจะถูกถามจากลูกค้า, ลูกค้าโกรธเพราะไม่มีใครแจ้งความคืบหน้า, สื่อมวลชนได้ข้อมูลผิดพลาดเพราะไม่มีช่องทางการสื่อสารที่เป็นทางการ

### 983.2 RACI Matrix สำหรับ Incident Communication

| บทบาท | Responsible (ทำงานจริง) | Accountable (รับผิดชอบผลลัพธ์) | Consulted (ให้คำปรึกษา) | Informed (แจ้งให้ทราบ) |
|---|---|---|---|---|
| **Incident Commander** | ตัดสินใจ, ประสานงานทีม | ✓ | | |
| **On-call SRE/DBA** | รัน runbook, กู้คืนระบบ | | ✓ | |
| **Communication Lead** | ร่างและส่งข้อความสื่อสาร | ✓ | | |
| **VP Engineering** | | | ✓ | |
| **CEO/Executive Team** | | | | ✓ (สำหรับ major incident) |
| **Customer Support** | ตอบคำถามลูกค้าปลายทาง | ✓ | ✓ | |
| **Legal/Compliance** | | | ✓ (ถ้ามีเรื่อง data breach) | |
| **ลูกค้า** | | | | ✓ |

### 983.3 Timeline การสื่อสารตามระดับความรุนแรง (Severity)

| Severity | นิยาม | แจ้งภายใน | ช่องทาง | ความถี่อัปเดต |
|---|---|---|---|---|
| **SEV-1** | ระบบล่มทั้งหมด กระทบลูกค้าทุกราย | ทันที (5 นาที) | PagerDuty, Slack #incident, Status Page, Email ผู้บริหาร | ทุก 15-30 นาที |
| **SEV-2** | ระบบบางส่วนล่ม กระทบลูกค้าบางกลุ่ม | 15 นาที | Slack #incident, Status Page | ทุก 1 ชั่วโมง |
| **SEV-3** | ประสิทธิภาพลดลง ไม่กระทบการใช้งานหลัก | 1 ชั่วโมง | Slack #incident (internal เท่านั้น) | เมื่อมีความคืบหน้า |

### 983.4 ตัวอย่าง Template การสื่อสารแต่ละกลุ่มเป้าหมาย

**ข้อความแจ้ง Internal Team (ทันทีที่ประกาศ incident):**

```
🔴 SEV-1 INCIDENT DECLARED — INC-2026-0847

Summary: Primary region (ap-southeast-1) unreachable since 09:52 ICT.
DR failover procedure DR-RB-001 initiated.

Incident Commander: @jane.doe
Status: Failover in progress (Step 4/9 in runbook)
ETA to recovery: ~20 minutes (based on last Game Day timing)

War room: #incident-2026-0847 (Slack)
Bridge: https://meet.example.com/incident-bridge

DO NOT deploy any changes to production until incident is resolved.
Next update in 15 minutes.
```

**ข้อความแจ้งลูกค้า (Status Page — เวอร์ชันแรก):**

```
[Investigating] We are currently investigating reports of service
disruption. Some users may experience errors when accessing the
platform. We will provide an update within 15 minutes.

Posted at 09:58 ICT
```

**ข้อความแจ้งลูกค้า (Status Page — เวอร์ชันอัปเดตระหว่างกู้คืน):**

```
[Identified] We have identified the issue as a regional infrastructure
outage affecting our primary data center. Our team has activated our
disaster recovery procedure and service is being restored via our
backup region. We expect full recovery within 30 minutes.

Posted at 10:05 ICT
```

**ข้อความแจ้งลูกค้า (Status Page — เวอร์ชัน resolved):**

```
[Resolved] Service has been fully restored via our backup region as
of 10:22 ICT. All systems are operating normally. We apologize for
the disruption. A detailed post-incident report will be published
within 5 business days.

Posted at 10:25 ICT
```

**ข้อความแจ้งลูกค้า Enterprise ที่มี SLA (Email โดยตรง จาก Customer Success):**

```
Subject: [Action Required] Service Disruption Notice - INC-2026-0847

Dear [Customer Name],

We experienced a service disruption from 09:52 to 10:22 ICT (30 minutes)
today due to a regional infrastructure outage. Our disaster recovery
systems activated automatically and service has been fully restored.

Per our SLA agreement, this incident will be reflected in your monthly
service credit calculation. Our Customer Success team will follow up
with specific details within 2 business days.

A full post-incident report will be shared by [date].

We take service reliability seriously and apologize for any
inconvenience this may have caused to your operations.

[Customer Success Manager Name]
```

### 983.5 ข้อผิดพลาดที่พบบ่อยในการสื่อสารระหว่างเกิดเหตุ

1. **เงียบนานเกินไป** — ไม่มีการอัปเดตแม้จะยังไม่มีความคืบหน้าใหม่ ทำให้ลูกค้า/ผู้บริหารคิดว่าไม่มีใครทำอะไรอยู่ ควรอัปเดตแม้เพียง "ยังอยู่ระหว่างดำเนินการ คาดว่าจะทราบผลใน X นาที" ก็ยังดีกว่าเงียบ
2. **สัญญาเวลาที่ทำไม่ได้จริง** — บอกลูกค้าว่า "จะเสร็จใน 10 นาที" แล้วเกินเวลาซ้ำแล้วซ้ำเล่า ทำให้เสียความน่าเชื่อถือมากกว่าบอกช่วงเวลาที่กว้างแต่แม่นยำ
3. **ให้ข้อมูลทางเทคนิคเกินความจำเป็นกับลูกค้าทั่วไป** — ลูกค้าส่วนใหญ่ไม่ต้องการรู้ว่า "WAL replication lag เกิน threshold" แค่ต้องการรู้ว่า "กำลังแก้ไข จะเสร็จเมื่อไหร่"
4. **ไม่มีคนเดียวที่รับผิดชอบการสื่อสาร** — ถ้าหลายคนพูดกับลูกค้าพร้อมกันโดยไม่ประสานกัน อาจให้ข้อมูลขัดแย้งกัน ต้องมี Communication Lead เพียงคนเดียวเป็นจุดศูนย์กลาง

---

## Step 984: Post-Incident Review (Postmortem)

### 984.1 Blameless Postmortem คืออะไร

**Postmortem** คือกระบวนการทบทวนเหตุการณ์หลังเกิด incident อย่างเป็นระบบ เพื่อหาสาเหตุที่แท้จริง (root cause) และวางแผนป้องกันไม่ให้เกิดซ้ำ

**Blameless** หมายถึงหลักการที่ว่า **เป้าหมายของ postmortem คือการเรียนรู้ ไม่ใช่การหาคนผิด** เพราะในระบบที่ซับซ้อน ความผิดพลาดมักเกิดจาก **หลายปัจจัยร่วมกัน** (process ที่ไม่รัดกุม, เครื่องมือที่ไม่มี safeguard, documentation ที่ขาดหาย) ไม่ใช่ความผิดของบุคคลใดบุคคลหนึ่งเพียงคนเดียว

**เหตุผลที่ blameless postmortem สำคัญ:**

1. **ถ้ากลัวถูกตำหนิ คนจะปิดบังข้อมูล** — วิศวกรที่กดคำสั่งผิดจะไม่กล้าบอกความจริงทั้งหมดถ้ารู้ว่าจะถูกลงโทษ ทำให้ root cause ที่แท้จริงไม่ถูกค้นพบ
2. **ความผิดพลาดของมนุษย์เป็นเรื่องธรรมดา** — จุดสำคัญคือทำไม "ระบบ" ถึงยอมให้ความผิดพลาดของมนุษย์คนเดียวสร้างความเสียหายระดับ incident ได้ (เช่น ทำไมไม่มี confirmation step ก่อนรัน `DROP TABLE` บน production)
3. **สร้างวัฒนธรรมความปลอดภัยทางจิตใจ (psychological safety)** — ทีมที่รู้สึกปลอดภัยจะรายงานปัญหาเล็กๆ ก่อนที่จะบานปลายเป็นเรื่องใหญ่

### 984.2 โครงสร้าง Postmortem Document

```markdown
# Postmortem: INC-2026-0847 — Primary Region Outage

## Metadata
- Incident Date: 2026-09-25
- Severity: SEV-1
- Duration: 30 minutes (09:52 - 10:22 ICT)
- Author: [ชื่อผู้เขียน]
- Status: Draft / Reviewed / Published
- Participants in review: [รายชื่อทุกคนที่เกี่ยวข้อง]

## Summary (สรุปสั้น 2-3 ประโยคสำหรับผู้บริหารที่ไม่มีเวลาอ่านทั้งหมด)

Primary region ap-southeast-1 experienced a total network outage due to
a cloud provider incident. Our DR failover procedure was activated and
service was restored in region ap-northeast-1 within 30 minutes,
meeting our RTO target. No customer data was lost (RPO target met).

## Impact

- ลูกค้าที่ได้รับผลกระทบ: 100% ของ active users ในช่วง 30 นาที
- Revenue impact โดยประมาณ: 314,000 บาท (คำนวณจาก Cost of Downtime
  formula ใน Step 977.3)
- SLA breach: 12 enterprise contracts เข้าเงื่อนไข service credit
- Data loss: 0 (failover สำเร็จก่อนถึง RPO threshold)

## Timeline (เวลาทั้งหมดเป็น ICT, ละเอียดถึงนาที)

| เวลา | เหตุการณ์ |
|---|---|
| 09:52 | Cloud provider เริ่มมีปัญหา network ใน ap-southeast-1 (ตามที่ provider ประกาศย้อนหลัง) |
| 09:55 | Monitoring alert แจ้งเตือน "primary database unreachable" |
| 09:57 | On-call SRE รับ alert, เริ่มตรวจสอบ |
| 09:59 | ยืนยันว่าเป็น region-level outage (ไม่ใช่ปัญหาเฉพาะ database) |
| 10:00 | ประกาศ SEV-1, เริ่ม DR-RB-001 |
| 10:03 | Pre-flight checks เสร็จสิ้น ยืนยัน replication lag ล่าสุด = 8 วินาที |
| 10:05 | Promote DR standby สำเร็จ |
| 10:12 | App servers scale up เสร็จสมบูรณ์ |
| 10:15 | Incident Commander อนุมัติ DNS failover |
| 10:16 | DNS เปลี่ยนเสร็จ เริ่ม propagate |
| 10:20 | Validation ผ่านทั้งหมด |
| 10:22 | ประกาศ Resolved บน status page |

## Root Cause

Cloud provider network outage (external, ไม่สามารถป้องกันได้จากฝั่งเรา)
สาเหตุนี้ยืนยันจาก cloud provider's own postmortem report ที่เผยแพร่
ภายหลัง 3 วัน

## What Went Well

1. Monitoring ตรวจจับปัญหาได้ภายใน 3 นาที ตรงตามเป้าหมาย detection time
2. Runbook DR-RB-001 ทำตามได้ครบทุกขั้นตอนโดยไม่ต้อง improvise
3. RTO จริง (30 นาที) ใกล้เคียงกับที่ทดสอบใน Game Day ครั้งล่าสุด (22 นาที)
   ส่วนต่างมาจากขั้นตอนขออนุมัติที่ใช้เวลานานกว่าปกติเพราะ Incident
   Commander อยู่ระหว่างเดินทาง
4. Zero data loss - RPO target สำเร็จ

## What Went Wrong / Contributing Factors

1. Incident Commander หลักติดต่อไม่ได้ทันทีเพราะอยู่บนเครื่องบิน ทำให้
   ขั้นตอนที่ 6 (DNS failover) ล่าช้าไป 8 นาที จากที่ควรจะเป็น
2. Status page อัปเดตช้ากว่าที่ตั้งเป้าไว้ (10 นาทีแรกไม่มีการอัปเดตใดๆ)
   เพราะ Communication Lead ไม่ทราบว่าต้องเป็นคนกดปุ่มอัปเดตเอง
   (เข้าใจผิดว่าเป็น automation)
3. ทีม Customer Support ได้รับคำถามจากลูกค้าก่อนที่จะได้รับแจ้งจาก
   incident channel ภายใน

## Action Items (ต้องมี owner และ deadline ชัดเจนทุกข้อ)

| # | Action | Owner | Priority | Deadline | Status |
|---|---|---|---|---|---|
| 1 | กำหนด Incident Commander สำรอง (deputy) อย่างน้อย 2 คนเสมอ ในทุก on-call rotation | SRE Lead | High | 2026-10-05 | Open |
| 2 | เพิ่ม automation ให้ status page อัปเดตอัตโนมัติทันทีที่ SEV-1 ถูกประกาศ (ข้อความ generic ก่อน แล้วให้คนแก้ไขรายละเอียดภายหลัง) | Platform Team | High | 2026-10-15 | Open |
| 3 | เพิ่ม Customer Support เข้า incident notification list ให้ได้รับแจ้งพร้อมทีม engineering ไม่ใช่หลังจากนั้น | Communication Lead | Medium | 2026-10-10 | Open |
| 4 | จัด Game Day เพิ่มเติมที่จำลองสถานการณ์ Incident Commander ติดต่อไม่ได้ เพื่อทดสอบ escalation path สำรอง | SRE Lead | Medium | 2026-11-01 | Open |

## Lessons Learned (สรุปบทเรียนเชิงกว้างสำหรับองค์กร ไม่ใช่แค่ incident นี้)

DR ทางเทคนิคทำงานได้ดีตามที่ออกแบบไว้ แต่จุดอ่อนที่แท้จริงอยู่ที่ "คน
และกระบวนการ" ไม่ใช่ "เทคโนโลยี" — นี่เป็นรูปแบบที่พบบ่อยในหลาย incident
ทั่วอุตสาหกรรม ควรลงทุนใน redundancy ของบทบาทมนุษย์ (เช่น Incident
Commander) เทียบเท่ากับที่ลงทุนใน redundancy ของระบบ (เช่น DR standby)
```

### 984.3 หลักการเขียน Action Item ที่ดี

Action item ที่ดีต้องเป็นไปตามหลัก **SMART**: Specific, Measurable, Assignable, Realistic, Time-bound ตัวอย่างการเปรียบเทียบ:

| Action Item แบบไม่ดี | Action Item แบบดี |
|---|---|
| "ปรับปรุงการ monitoring" | "เพิ่ม alert สำหรับ replication lag > 60 วินาที ส่งไปยัง PagerDuty ให้ @sre-team ภายในวันที่ 10 ต.ค." |
| "ทำให้ failover เร็วขึ้น" | "ลดเวลาขั้นตอนที่ 5 (scale app servers) จาก 5 นาทีเหลือ 2 นาที โดยเปลี่ยนจาก cold start เป็น warm pool ที่มี instance รันอยู่แล้ว 2 ตัว — owner: Platform Team, deadline: 2026-10-20" |
| "สื่อสารให้ดีขึ้น" | "สร้าง automation ที่ post ข้อความ generic ไปยัง status page ทันทีที่ PagerDuty ประกาศ SEV-1 — owner: Communication Lead, deadline: 2026-10-15" |

### 984.4 การติดตาม Action Item

Postmortem ที่ไม่มีการติดตาม action item ก็ไร้ประโยชน์เท่ากับไม่ได้ทำ ควรมี:

- Dashboard ที่แสดง action item ทั้งหมดจากทุก postmortem พร้อมสถานะ (Open/In Progress/Done)
- Review รายเดือนในที่ประชุมทีม engineering เพื่อติดตามความคืบหน้า
- Escalate ให้ผู้บริหารทราบถ้า action item สำคัญค้างเกิน deadline โดยไม่มีเหตุผล
- เชื่อมโยง action item กลับไปที่ postmortem ครั้งก่อนหน้า เพื่อตรวจสอบว่าปัญหาเดิมไม่ได้เกิดซ้ำ (recurring issue คือสัญญาณอันตรายว่า root cause แท้จริงยังไม่ถูกแก้)

---

## สรุปท้ายบท

บทนี้ได้สังเคราะห์ความรู้จาก Part 061-065 (Backup, PITR, Physical Replication, Logical Replication, HA/Failover) เข้าสู่กรอบคิดของ **Disaster Recovery และ Business Continuity Planning** ระดับองค์กร ประเด็นสำคัญที่ต้องจำ:

1. **DR ≠ Backup และ DR ≠ HA** — DR คือกระบวนการทั้งหมดที่รับมือกับความล้มเหลวระดับ site/region ไม่ใช่แค่เทคโนโลยีชิ้นเดียว
2. **RPO/RTO ต้องมาจากการวิเคราะห์ผลกระทบทางธุรกิจ (BIA)** ไม่ใช่ตัวเลขที่วิศวกรกำหนดเอง และต้องคำนวณ Cost of Downtime เพื่อตัดสินใจลงทุนอย่างมีเหตุผล
3. **DR Tier ทั้ง 4 แบบ** (Backup & Restore, Pilot Light, Warm Standby, Multi-Site Active-Active) มี trade-off ระหว่างต้นทุนกับ RPO/RTO ที่ชัดเจน ต้องเลือกให้เหมาะกับแต่ละ service tier
4. **Multi-Region Strategy** ต้องแยกความรับผิดชอบระหว่าง HA (local, synchronous) กับ DR (cross-region, asynchronous) อย่างชัดเจน และระวังความเสี่ยงของ replication slot ที่ค้างเมื่อ cross-region standby หลุดการเชื่อมต่อ
5. **DR Runbook ต้องไม่กำกวม** มีคำสั่งที่รันได้จริง มี decision tree ชัดเจน และเก็บไว้ในที่ที่เข้าถึงได้แม้ production จะล่ม
6. **DR ที่ไม่เคยทดสอบเชื่อถือไม่ได้** — ต้องมี Game Day สม่ำเสมอ และควรเริ่มศึกษา Chaos Engineering เพื่อค้นหาจุดอ่อนที่ไม่มีใครคาดคิด
7. **BCP กว้างกว่า DR** ครอบคลุมทั้งองค์กร ไม่ใช่แค่ระบบ IT — DR เป็นเพียงส่วนประกอบหนึ่งของ BCP
8. **Communication Plan สำคัญพอๆ กับแผนทางเทคนิค** ต้องมี RACI ชัดเจน และ template ข้อความสำหรับผู้มีส่วนได้ส่วนเสียแต่ละกลุ่ม
9. **Blameless Postmortem** คือเครื่องมือสำคัญในการเรียนรู้จากเหตุการณ์จริง เน้นหา root cause และปรับปรุงระบบ/กระบวนการ ไม่ใช่การหาคนผิด

องค์กรระดับ world-class ไม่ใช่องค์กรที่ไม่เคยเจอภัยพิบัติ — เพราะภัยพิบัติเกิดขึ้นกับทุกองค์กรไม่ช้าก็เร็ว องค์กรระดับ world-class คือองค์กรที่ **เตรียมพร้อม ทดสอบสม่ำเสมอ และเรียนรู้จากทุกเหตุการณ์** จนสามารถกู้คืนได้อย่างรวดเร็วและมั่นใจ แม้ในสถานการณ์ที่เลวร้ายที่สุด

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1

จงอธิบายความแตกต่างระหว่าง High Availability (HA) และ Disaster Recovery (DR) พร้อมยกตัวอย่างสถานการณ์ที่ HA ช่วยไม่ได้แต่ DR จำเป็นต้องใช้

<details>
<summary>เฉลย</summary>

HA ออกแบบมาเพื่อรับมือความล้มเหลวระดับ component/node เดียว (เช่น server เสีย) โดย standby ที่อยู่ใกล้กัน (มักอยู่ same DC/AZ) จะ failover เข้ามาแทนอย่างรวดเร็ว (วินาทีถึงนาที) ส่วน DR ออกแบบมาเพื่อรับมือความล้มเหลวระดับ site/region ทั้งหมด (เช่น ทั้ง datacenter หรือทั้ง cloud region ใช้งานไม่ได้)

ตัวอย่างสถานการณ์ที่ HA ช่วยไม่ได้: วิศวกรรันคำสั่ง `DROP TABLE orders;` บน production โดยไม่ได้ตั้งใจ เพราะ standby ที่ใช้สำหรับ HA จะ replicate คำสั่งนี้ (ผ่าน WAL) ไปยัง standby ทันที ทำให้ตารางหายไปทั้งใน primary และ standby พร้อมกัน กรณีนี้ต้องใช้กลไกของ DR เช่น PITR (Point-in-Time Recovery) จาก backup ที่แยกต่างหาก เพื่อกู้คืนข้อมูลไปยังจุดเวลาก่อนเกิดความผิดพลาด อีกตัวอย่างคือ ransomware attack ที่เข้ารหัสข้อมูลทั้ง primary และ standby พร้อมกันเพราะทั้งคู่เชื่อมต่อ network เดียวกันแบบ online

</details>

### แบบฝึกหัดที่ 2

บริษัท e-commerce แห่งหนึ่งมีรายได้เฉลี่ย 90,000,000 บาทต่อเดือน (คิดเป็น 24 ชั่วโมง x 30 วัน) มีสัญญา SLA กับลูกค้า enterprise 15 ราย ปรับ 4,000 บาทต่อชั่วโมงต่อสัญญาเมื่อเกิด downtime และมีพนักงาน support 10 คน เงินเดือนเฉลี่ย 35,000 บาท/เดือน (คิดเป็น 219 บาท/ชม. โดยประมาณ) จงคำนวณ Cost of Downtime ต่อชั่วโมงในช่วงเวลาปกติ (ไม่ใช่ peak) โดยไม่รวม reputation cost

<details>
<summary>เฉลย</summary>

```
Lost Revenue per Hour = 90,000,000 / (30 × 24) = 125,000 บาท/ชม.
SLA Penalty           = 4,000 × 15               =  60,000 บาท/ชม.
Productivity Loss     = 10 × 219 × 0.8 (สมมติ 80% ทำงานไม่ได้)
                                                  ≈   1,752 บาท/ชม.
--------------------------------------------------------------
รวม Cost of Downtime ≈ 186,752 บาท/ชั่วโมง
```

ตัวเลขนี้ใช้เป็นฐานในการตัดสินใจว่าจะลงทุนใน DR Tier ไหนคุ้มค่า เช่น ถ้า Warm Standby มีต้นทุนเพิ่ม 150,000 บาท/เดือน แต่ช่วยลด RTO จาก 3 ชั่วโมงเหลือ 20 นาที จะประหยัดความเสียหายได้ประมาณ `186,752 × (3 - 0.33) ≈ 498,600` บาทต่อครั้งที่เกิดเหตุการณ์ระดับ region failure

</details>

### แบบฝึกหัดที่ 3

จงจับคู่ DR Tier กับลักษณะสถาปัตยกรรมที่ถูกต้อง: (ก) Backup & Restore (ข) Pilot Light (ค) Warm Standby (ง) Multi-Site Active-Active กับคำอธิบาย: (1) มี replica รันตลอดเวลาแต่ app servers ปิดอยู่ (2) ทุก site รับ traffic จริงพร้อมกัน (3) เก็บเฉพาะ backup ไว้ต่าง region ไม่มี infrastructure รันตลอดเวลา (4) มี environment เต็มรูปแบบขนาดเล็กรันตลอดเวลาพร้อม scale ขึ้น

<details>
<summary>เฉลย</summary>

(ก)-(3), (ข)-(1), (ค)-(4), (ง)-(2)

- Backup & Restore: เก็บ backup (base backup + WAL) ไว้ต่าง region ไม่มี compute รันตลอดเวลา ต้อง provision ใหม่ทั้งหมดเมื่อเกิดเหตุ
- Pilot Light: มี PostgreSQL replica (streaming replication) รันตลอดเวลาที่ region สำรอง แต่ app servers ปิดไว้ (desired capacity = 0) รอ scale ขึ้นเมื่อ failover
- Warm Standby: มีทั้ง database replica และ app servers ขนาดเล็กรันตลอดเวลา พร้อม scale ขึ้นเต็มรูปแบบเมื่อ failover
- Multi-Site Active-Active: ทุก site รับ traffic จริงพร้อมกัน ต้องแก้ปัญหา write conflict ด้วยเทคนิคเช่น geo-partitioning หรือ distributed SQL

</details>

### แบบฝึกหัดที่ 4

เพราะเหตุใดการเปิดใช้ `hot_standby_feedback = on` บน cross-region DR standby จึงเป็นความเสี่ยงต่อ primary database มากกว่าประโยชน์ที่ได้ และควรใช้อะไรควบคุม query conflict แทน

<details>
<summary>เฉลย</summary>

`hot_standby_feedback = on` ทำให้ standby ส่งข้อมูล transaction ID ที่กำลังทำงานอยู่กลับไปยัง primary เพื่อป้องกันไม่ให้ primary vacuum ลบ dead tuple ที่ query บน standby ยังต้องใช้อยู่ (ป้องกัน query cancellation) แต่ถ้า standby อยู่ cross-region ซึ่งมักมี replication lag สูงกว่าปกติ (เพราะ network latency ไกล หรือหลุด connection บ่อย) การเปิด feedback นี้จะทำให้ primary **ไม่สามารถ vacuum ทำความสะอาด dead tuple ได้เป็นเวลานาน** เกิด table bloat สะสมบน primary ซึ่งกระทบ performance ของ production โดยตรง ทั้งที่ standby ตัวนั้นมีไว้สำหรับ DR ไม่ใช่สำหรับ query จริงตลอดเวลา

ควรปิด `hot_standby_feedback` สำหรับ cross-region DR standby และใช้ `max_standby_streaming_delay` ควบคุมแทน ซึ่งจะยอมให้ query บน standby ถูก cancel หากมี conflict กับ WAL replay แทนที่จะไปกระทบ vacuum บน primary

</details>

### แบบฝึกหัดที่ 5

DR Runbook ที่ดีต้องมีคุณสมบัติอะไรบ้าง อย่างน้อย 5 ข้อ พร้อมอธิบายเหตุผลสั้นๆ ของแต่ละข้อ

<details>
<summary>เฉลย</summary>

1. **ไม่กำกวม (Unambiguous)**: มีคำสั่งที่ copy-paste รันได้จริง เพราะภายใต้ความกดดันสูง วิศวกรไม่มีเวลาตีความคำอธิบายเชิงแนวคิด
2. **มี Decision Tree ชัดเจน**: ระบุเงื่อนไขตัวเลขที่ชัดเจน (เช่น "ถ้า lag เกิน 15 นาที") ไม่ใช่คำพูดกำกวม เพื่อลดการตัดสินใจแบบอัตวิสัยในสถานการณ์วิกฤต
3. **ระบุผู้รับผิดชอบและผู้อนุมัติแต่ละขั้นตอน**: โดยเฉพาะขั้นตอนที่กระทบลูกค้าโดยตรง (เช่น DNS failover) ต้องมีการอนุมัติก่อนเพื่อป้องกันการตัดสินใจผิดพลาดที่ไม่สามารถย้อนกลับได้
4. **มี Rollback/Failback Plan**: เพื่อรองรับกรณีที่ขั้นตอนล้มเหลวกลางทางหรือต้องการกลับมาที่ site เดิมภายหลัง
5. **Version control และอัปเดตสม่ำเสมอ**: เพราะ infrastructure เปลี่ยนแปลงตลอดเวลา runbook เก่าจะใช้ไม่ได้จริงถ้าไม่ปรับปรุงตาม
6. **เข้าถึงได้แม้ระบบหลักล่ม**: ห้ามเก็บ runbook ไว้บน infrastructure เดียวกับที่กำลังจะล่ม ต้องมีสำเนาภายนอกที่เข้าถึงได้เสมอ

(ตอบครบ 5 ข้อใดก็ได้จากรายการข้างต้นถือว่าถูกต้อง)

</details>

### แบบฝึกหัดที่ 6

จงอธิบายความแตกต่างระหว่าง Game Day และ Chaos Engineering และเพราะเหตุใดองค์กรที่เพิ่งเริ่มต้นควรเริ่มจาก Game Day ก่อน

<details>
<summary>เฉลย</summary>

**Game Day** คือการจำลองสถานการณ์ภัยพิบัติแบบมีการวางแผนล่วงหน้า แจ้งผู้เกี่ยวข้องทราบ กำหนดวันเวลาชัดเจน และทีมงานทำตาม runbook จริงภายใต้การสังเกตการณ์ มีเป้าหมายชัดเจนที่วัดผลได้ (เช่น RTO เป้าหมาย)

**Chaos Engineering** คือแนวคิดที่จงใจสร้างความล้มเหลวแบบสุ่มหรือกึ่งสุ่มใน production อย่างสม่ำเสมอ (มักไม่แจ้งล่วงหน้าหรือแจ้งเพียงบางส่วน) เพื่อค้นหาจุดอ่อนที่ไม่มีใครคาดคิดว่าจะมีปัญหา โดยตั้งสมมติฐานเกี่ยวกับ steady state ของระบบและพยายามพิสูจน์ว่าสมมติฐานนั้นผิด

องค์กรที่เพิ่งเริ่มต้นควรเริ่มจาก Game Day ก่อนเพราะ: (1) เป็นการทดสอบแบบควบคุมความเสี่ยงได้ มีการวางแผนและจำกัดขอบเขตชัดเจน (2) ช่วยฝึกทีมงานให้คุ้นเคยกับ runbook ก่อน (3) Chaos Engineering ต้องการความเชื่อมั่นในระบบ monitoring, rollback mechanism, และ blast radius control ที่ดีมากก่อนจะทำใน production ซึ่งองค์กรที่เพิ่งเริ่มต้นมักยังไม่มีความพร้อมเพียงพอ การกระโดดไปทำ chaos engineering ก่อนอาจสร้างความเสียหายจริงโดยไม่ได้ตั้งใจ

</details>

### แบบฝึกหัดที่ 7

Business Continuity Planning (BCP) ครอบคลุมมากกว่า Disaster Recovery (DR) อย่างไร จงยกตัวอย่างสถานการณ์ที่ DR (ระบบ IT) ไม่มีปัญหาเลย แต่ธุรกิจยังคงหยุดชะงักอยู่

<details>
<summary>เฉลย</summary>

BCP ครอบคลุมความต่อเนื่องของธุรกิจทั้งองค์กร ไม่ใช่แค่ระบบ IT รวมถึง workforce continuity (คนทำงานที่ไหนได้บ้าง), vendor/supply chain continuity, financial continuity, legal/compliance และ crisis communication ส่วน DR เป็นเพียงองค์ประกอบหนึ่งของ BCP ที่เน้นเฉพาะการกู้คืนระบบ IT

ตัวอย่างสถานการณ์: payment gateway partner ที่บริษัทใช้อยู่ล้มละลายหรือหยุดให้บริการกะทันหัน ในกรณีนี้ database และระบบ IT ทั้งหมดของบริษัทยังทำงานได้ปกติ 100% (ไม่มีปัญหาทาง technical เลย) แต่ธุรกิจไม่สามารถรับชำระเงินจากลูกค้าได้ ทำให้ธุรกิจหยุดชะงักอยู่ดี กรณีนี้ต้องอาศัยแผน BCP ที่มี payment provider สำรองไว้ล่วงหน้า ไม่ใช่เรื่องที่ DR ของ database จะช่วยแก้ปัญหาได้

</details>

### แบบฝึกหัดที่ 8

ในระหว่างเกิด SEV-1 incident เพราะเหตุใดการมี "Communication Lead" เพียงคนเดียวที่รับผิดชอบสื่อสารกับภายนอกจึงสำคัญ และจะเกิดอะไรขึ้นถ้าไม่มีบทบาทนี้ชัดเจน

<details>
<summary>เฉลย</summary>

การมี Communication Lead เพียงคนเดียวช่วยให้ข้อความที่สื่อสารออกไปสู่ลูกค้า สื่อมวลชน หรือผู้บริหาร มีความสอดคล้องกันเป็นหนึ่งเดียว ป้องกันความสับสนที่เกิดจากหลายคนให้ข้อมูลที่แตกต่างกันหรือขัดแย้งกันในเวลาเดียวกัน

ถ้าไม่มีบทบาทนี้ชัดเจน อาจเกิดปัญหา เช่น: วิศวกรที่กำลังแก้ปัญหาต้องหยุดงานเพื่อตอบคำถามสื่อหรือลูกค้าเอง ทำให้การกู้คืนระบบล่าช้า, ทีม Customer Support ให้ข้อมูลที่ไม่ตรงกับสิ่งที่ทีมเทคนิครายงานภายใน (เช่น สัญญาเวลาที่ทำไม่ได้จริง) ทำให้เสียความน่าเชื่อถือ, หรือไม่มีใครอัปเดต status page เลยเพราะทุกคนคิดว่าเป็นหน้าที่ของคนอื่น ทำให้ลูกค้ารู้สึกว่าไม่มีใครดูแลปัญหาอยู่

</details>

### แบบฝึกหัดที่ 9

จงอธิบายหลักการของ "Blameless Postmortem" และยกตัวอย่าง Action Item ที่เขียนตามหลัก SMART เทียบกับ Action Item ที่เขียนไม่ดี

<details>
<summary>เฉลย</summary>

Blameless Postmortem คือหลักการที่เป้าหมายของการทบทวนเหตุการณ์คือการเรียนรู้และปรับปรุงระบบ/กระบวนการ ไม่ใช่การหาคนมารับผิดชอบหรือลงโทษ เพราะความผิดพลาดในระบบที่ซับซ้อนมักเกิดจากหลายปัจจัยร่วมกัน (process, tooling, documentation) ไม่ใช่ความผิดของบุคคลเดียว การสร้างบรรยากาศที่ปลอดภัยทางจิตใจ (psychological safety) ทำให้ทีมงานกล้ารายงานข้อเท็จจริงทั้งหมดอย่างตรงไปตรงมา ซึ่งจำเป็นต่อการค้นหา root cause ที่แท้จริง

ตัวอย่าง Action Item ที่ไม่ดี: "ปรับปรุงการ monitoring ให้ดีขึ้น" — กำกวม ไม่มี owner ไม่มี deadline วัดผลไม่ได้ว่าทำสำเร็จหรือไม่

ตัวอย่าง Action Item แบบ SMART: "เพิ่ม alert สำหรับ replication lag ที่เกิน 60 วินาที ส่งแจ้งเตือนไปยัง PagerDuty — Owner: ทีม SRE, Deadline: 10 ตุลาคม 2026" — ระบุสิ่งที่ต้องทำชัดเจน (Specific), วัดผลได้ (Measurable ผ่านค่า threshold), มีผู้รับผิดชอบ (Assignable), ทำได้จริง (Realistic), และมีกำหนดเวลา (Time-bound)

</details>

### แบบฝึกหัดที่ 10 (แบบฝึกหัดรวม)

จงเขียน DR Runbook และ BCP แบบย่อสำหรับระบบ e-commerce SaaS (เชื่อมโยง Part 080) ที่ครอบคลุม 3 สถานการณ์ต่อไปนี้: (ก) Region ล่มทั้ง region (ข) ข้อมูลถูกลบผิดพลาดจาก human error (เช่น `DELETE FROM orders` โดยไม่มี WHERE) (ค) Ransomware attack ที่เข้ารหัสทั้ง production database และ backup ที่เชื่อมต่อ online โดยระบุ RPO/RTO เป้าหมาย, DR Tier ที่เหมาะสม, และขั้นตอนหลักสำหรับแต่ละสถานการณ์

<details>
<summary>เฉลย (แนวทางคำตอบ)</summary>

**ภาพรวม Service Tier และ RPO/RTO เป้าหมาย:**

| Component | Tier | RPO เป้าหมาย | RTO เป้าหมาย |
|---|---|---|---|
| Order/Payment Database | Tier 0 | ≤ 10 วินาที | ≤ 15 นาที |
| User/Inventory Database | Tier 1 | ≤ 1 นาที | ≤ 30 นาที |
| Analytics/Reporting | Tier 3 | ≤ 24 ชั่วโมง | ≤ 24 ชั่วโมง |

DR Tier ที่เหมาะสมโดยรวม: **Warm Standby** สำหรับ Tier 0-1 (คุ้มค่าที่สุดเมื่อเทียบกับ Cost of Downtime ที่คำนวณได้) และ **Backup & Restore** ก็เพียงพอสำหรับ Tier 3

---

**(ก) Region ล่มทั้ง region**

- ใช้ DR Runbook DR-RB-001 ตามตัวอย่างเต็มใน Step 980.2: ตรวจจับผ่าน monitoring 3 จุด → ประกาศ SEV-1 → promote cross-region warm standby → scale app servers → ขออนุมัติ Incident Commander ก่อนเปลี่ยน DNS → validate → ประกาศ resolved
- ป้องกัน split-brain ด้วยการ fence region เดิมไม่ให้รับ write จนกว่าจะยืนยันปลอดภัย
- Failback ทำแยกต่างหากในช่วง low-traffic window เท่านั้น

**(ข) ข้อมูลถูกลบผิดพลาดจาก human error**

สถานการณ์นี้ **DR แบบ region failover ช่วยไม่ได้เลย** เพราะ standby จะ replicate คำสั่ง DELETE ตามไปด้วยทันที ต้องใช้ PITR แทน:

1. ทันทีที่ตรวจพบ (เช่นจาก alert ที่ row count ลดผิดปกติ หรือ report จากทีม): **สั่งหยุด application ไม่ให้เขียนข้อมูลเพิ่ม** เพื่อป้องกัน WAL ทับข้อมูลที่จำเป็นต่อการกู้คืน
2. ตรวจสอบเวลาที่แน่นอนที่คำสั่งผิดพลาดถูกรัน (จาก query log / `pg_stat_activity` history หรือ audit log)
3. Restore backup ล่าสุดไปยัง environment แยกต่างหาก (ไม่ใช่ production เดิม) พร้อม PITR ไปยังเวลาก่อนเกิดเหตุ 1 วินาที:
   ```
   recovery_target_time = '<เวลาก่อนเกิด DELETE>'
   recovery_target_action = 'promote'
   ```
4. ตรวจสอบความถูกต้องของข้อมูลใน environment ที่ restore แล้ว (validate row count, ตัวอย่างข้อมูลสำคัญ)
5. Export เฉพาะข้อมูลที่หายไป (เช่นใช้ `pg_dump` แบบ `--table=orders` หรือเขียน script เทียบ diff) แล้วนำกลับเข้า production เดิมแบบ selective (ไม่ใช่ overwrite ทั้ง database เพื่อไม่ให้สูญเสียข้อมูลที่ถูกต้องซึ่งเกิดขึ้นหลังจากเวลานั้น)
6. เปิด application กลับมาทำงานปกติ, เริ่ม postmortem

บทเรียนเชิงป้องกัน: เพิ่ม safeguard เช่น `ON DELETE` ต้องผ่าน confirmation, จำกัด permission ไม่ให้รัน raw DELETE บน production โดยตรง, ใช้ soft-delete pattern แทน hard-delete สำหรับตารางสำคัญ

**(ค) Ransomware attack**

สถานการณ์นี้อันตรายที่สุดเพราะกระทบทั้ง production และ backup ที่เชื่อมต่อ online พร้อมกัน:

1. **Isolate ทันที**: ตัดการเชื่อมต่อ network ของระบบที่ต้องสงสัยทั้งหมดออกจากกันและกัน เพื่อจำกัด blast radius ไม่ให้ ransomware แพร่กระจายต่อ (รวมถึงตัด access ของ backup storage ที่ online อยู่ทันที)
2. แจ้ง Legal/Compliance และพิจารณาแจ้งหน่วยงานที่เกี่ยวข้องตามกฎหมาย (data breach notification requirement)
3. ตรวจสอบว่ามี **immutable backup** (offline หรือ write-once storage ที่ ransomware เข้าถึงไม่ได้) หรือไม่ — นี่คือเหตุผลที่นโยบาย backup ต้องมี backup อย่างน้อย 1 ชุดที่ "air-gapped" หรือใช้ object storage ที่ตั้งค่า retention lock/object lock ไว้ (ป้องกันการลบหรือเข้ารหัสทับแม้ credential หลักจะถูกขโมย)
4. Restore จาก immutable backup ชุดล่าสุดก่อนที่ ransomware จะเริ่มทำงาน (ต้องสืบสวนหาจุดเริ่มต้นของการติดเชื้อก่อน ซึ่งอาจใช้เวลานานกว่าสถานการณ์อื่น เพราะต้อง forensic analysis ร่วมกับทีม security)
5. สร้าง infrastructure ใหม่ทั้งหมด (ไม่ reuse ของเดิมที่อาจยังมี backdoor หลงเหลือ) แล้ว restore ข้อมูลเข้าไป
6. หมุนเวียน (rotate) credentials ทั้งหมดที่เกี่ยวข้องก่อนเปิดระบบกลับมาใช้งาน
7. Communication Plan ระดับสูงสุด: ต้องแจ้งลูกค้า, อาจต้องแจ้งหน่วยงานกำกับดูแล, และเตรียมทีมกฎหมาย/PR รับมือ

บทเรียนเชิงป้องกันสำคัญที่สุด: **backup ที่เชื่อมต่อ online ตลอดเวลาไม่ใช่ DR ที่ปลอดภัยจาก ransomware** ต้องมี tier ของ backup ที่เป็น immutable/air-gapped เสมอ ต่อให้ RPO จะสูงกว่า real-time replication ก็ตาม เพราะเป็น "safety net สุดท้าย" ที่ไม่มีอะไรทำลายได้

---

**สรุปหลักการ BCP ที่ครอบคลุมทั้ง 3 สถานการณ์:**

- ทุกสถานการณ์ต้องมี Communication Plan ที่พร้อมใช้ทันที (status page, internal Slack, ลูกค้า enterprise)
- ทุกสถานการณ์ต้องจบด้วย Blameless Postmortem และ action item ที่ติดตามได้
- Runbook สำหรับแต่ละสถานการณ์ต้องแยกจากกันชัดเจน เพราะกลไกการกู้คืนต่างกันโดยสิ้นเชิง (region failover ใช้ promote standby, human error ใช้ PITR, ransomware ใช้ immutable backup + rebuild ใหม่ทั้งหมด)
- ทั้งหมดนี้ต้องผ่านการทดสอบ (Game Day) อย่างน้อยปีละครั้งต่อสถานการณ์ เพื่อยืนยันว่า runbook ยังใช้งานได้จริง

</details>

---

## บทถัดไป

บทถัดไปจะนำกรณีศึกษาจากองค์กรระดับโลกมาวิเคราะห์เชิงลึก ครอบคลุมสถาปัตยกรรม PostgreSQL ในระดับ enterprise จริง

→ [Part 100: Enterprise Case Studies](./part-100-enterprise-case-studies.md)
</content>
