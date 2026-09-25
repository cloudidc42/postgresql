# PostgreSQL บน Cloud: GCP Cloud SQL และ AlloyDB

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับโลก/ผู้เชี่ยวชาญ | Part 095

---

## เกี่ยวกับบทนี้

บทนี้เป็นภาคต่อของ Part 094 (PostgreSQL บน AWS: RDS และ Aurora) โดยย้ายมาฝั่ง **Google Cloud Platform (GCP)** ซึ่งมีบริการ managed PostgreSQL สองตัวหลักที่นักพัฒนาและสถาปนิกระบบต้องรู้จัก คือ **Cloud SQL for PostgreSQL** (บริการ managed database แบบดั้งเดิม เทียบเท่า RDS) และ **AlloyDB for PostgreSQL** (บริการรุ่นใหม่ที่ Google ออกแบบมาเพื่อประสิทธิภาพสูงกว่า เทียบเท่า Aurora)

> **หมายเหตุสำคัญเกี่ยวกับลักษณะเนื้อหา**
>
> สภาพแวดล้อมที่ใช้เขียนบทเรียนนี้ไม่มีสิทธิ์เข้าถึง GCP Console หรือบัญชี GCP จริง ดังนั้นคำสั่ง `gcloud` และตัวอย่าง configuration ทั้งหมดในบทนี้ **เป็นแนวทางเชิงสถาปัตยกรรมและแนวคิด (architectural / conceptual guidance)** ที่เขียนขึ้นให้ตรงกับ syntax และพฤติกรรมจริงของบริการ Cloud SQL และ AlloyDB ตามเอกสารทางการของ Google Cloud มากที่สุด แต่ผู้เรียนควรทดสอบคำสั่งจริงในโปรเจกต์ GCP ของตนเอง (แนะนำให้ใช้ GCP Free Trial หรือ sandbox project) และตรวจสอบราคา/พฤติกรรมล่าสุดจากเอกสารทางการ (`cloud.google.com/sql/docs`, `cloud.google.com/alloydb/docs`) ก่อนนำไปใช้งานจริงหรือใช้ในการตัดสินใจเชิงธุรกิจ เนื่องจากบริการ cloud มีการเปลี่ยนแปลง feature และราคาอยู่ตลอดเวลา

---

## เป้าหมายการเรียนรู้

เมื่อจบบทนี้ ผู้เรียนจะสามารถ:

1. อธิบายว่า Cloud SQL for PostgreSQL คืออะไร และเปรียบเทียบกับ AWS RDS for PostgreSQL ได้
2. เลือก machine type, storage type และ region/zone ที่เหมาะสมสำหรับ Cloud SQL instance
3. อธิบายกลไก High Availability (Regional instance) ของ Cloud SQL และเปรียบเทียบกับ Multi-AZ ของ RDS
4. ออกแบบ Read Replica (รวมถึง cross-region replica) เพื่อ scale การอ่านและรองรับ disaster recovery
5. อธิบายกลไก Automated Backup และ Point-in-Time Recovery (PITR) ของ Cloud SQL
6. ตั้งค่าและใช้งาน Cloud SQL Auth Proxy เพื่อเชื่อมต่อฐานข้อมูลอย่างปลอดภัยโดยไม่ต้องเปิด public IP
7. อธิบายว่า AlloyDB for PostgreSQL คืออะไร และสถาปัตยกรรม disaggregated storage ที่ทำให้มันต่างจาก Cloud SQL
8. อธิบายการทำงานของ AlloyDB Columnar Engine และแนวคิด Hybrid Transactional/Analytical Processing (HTAP)
9. เข้าใจภาพรวมของ AlloyDB AI และการรองรับ pgvector สำหรับ vector search
10. เลือกใช้บริการที่เหมาะสม (Cloud SQL vs AlloyDB) สำหรับ use case ต่าง ๆ ของระบบ e-commerce จริง

---

## Step 941: Cloud SQL for PostgreSQL คืออะไร

### ภาพรวม

**Cloud SQL for PostgreSQL** คือบริการ **fully managed relational database service** ของ Google Cloud ที่รัน PostgreSQL เวอร์ชันมาตรฐาน (community edition ที่ Google ดูแล patch ให้) โดยจัดการงานดูแลระบบพื้นฐานให้ผู้ใช้แทบทั้งหมด ได้แก่:

- การติดตั้งและ provisioning เครื่อง (VM ที่รัน PostgreSQL)
- การทำ patch/update เวอร์ชัน PostgreSQL และระบบปฏิบัติการ
- การสำรองข้อมูล (backup) อัตโนมัติ
- การทำ replication สำหรับ High Availability
- การขยาย storage อัตโนมัติ (storage auto-increase)
- การตรวจสอบสุขภาพระบบ (monitoring) ผ่าน Cloud Monitoring

แนวคิดนี้เหมือนกับ AWS RDS for PostgreSQL ที่เราเรียนใน Part 094 ทุกประการในระดับ "promise" — คือผู้ใช้ไม่ต้อง SSH เข้าไปติดตั้ง PostgreSQL เอง ไม่ต้องจัดการ OS patching เอง

### Cloud SQL เทียบกับ RDS (ทบทวนจาก Part 094)

| แนวคิด | AWS RDS for PostgreSQL | GCP Cloud SQL for PostgreSQL |
|---|---|---|
| หน่วยพื้นฐาน | DB Instance | Cloud SQL Instance |
| การเลือกขนาดเครื่อง | DB Instance Class (เช่น `db.r6g.xlarge`) | Machine Type (เช่น `db-custom-4-16384` หรือ tier ที่กำหนดไว้ล่วงหน้า) |
| High Availability | Multi-AZ Deployment | Regional Instance (HA configuration) |
| Read Replica | Read Replica (same-region/cross-region) | Read Replica (same-region/cross-region) |
| การเชื่อมต่อปลอดภัยแบบไม่เปิด public IP | RDS Proxy / VPC + IAM auth | **Cloud SQL Auth Proxy** |
| Storage | EBS (gp3/io1/io2) แยกจาก compute | Persistent Disk (SSD/HDD) ผูกกับ instance โดยตรง |
| Automated Backup + PITR | Automated Backups + Transaction Logs | Automated Backups + Write-Ahead Log (WAL) สำหรับ PITR |
| บริการรุ่น "next-gen" ประสิทธิภาพสูง | Aurora PostgreSQL | AlloyDB for PostgreSQL |
| Serverless option | Aurora Serverless v2 | Cloud SQL ไม่มี serverless แท้ (แต่มี Cloud SQL Enterprise Plus edition ปรับ resource ได้ยืดหยุ่นขึ้น), ฝั่ง serverless จริงคือ AlloyDB Omni/Spanner ไม่ใช่ตัวเดียวกัน |
| Console | AWS RDS Console | Cloud SQL Console (ภายใต้ Google Cloud Console) |
| CLI | `aws rds ...` | `gcloud sql ...` |

**ข้อสังเกตสำคัญ**: สถาปัตยกรรมของ Cloud SQL (เหมือน RDS) คือ **compute + storage ผูกกันในระดับ instance เดียว** ต่างจาก Aurora/AlloyDB ที่แยก storage ออกเป็น distributed storage layer ต่างหาก (ดู Step 947) นี่คือความแตกต่างเชิงสถาปัตยกรรมพื้นฐานที่สุดที่ทำให้ Aurora/AlloyDB เร็วกว่าและ scale storage ได้ยืดหยุ่นกว่า Cloud SQL/RDS

### ภาพรวมสถาปัตยกรรม Cloud SQL (Single Zone)

```
                     ┌─────────────────────────────┐
                     │   Application (Compute       │
                     │   Engine / GKE / Cloud Run)   │
                     └───────────────┬───────────────┘
                                     │ private IP / public IP
                                     ▼
                     ┌─────────────────────────────┐
                     │      Cloud SQL Instance       │
                     │  ┌─────────────────────────┐  │
                     │  │   PostgreSQL Engine      │  │
                     │  │   (single VM, one zone)  │  │
                     │  └───────────┬─────────────┘  │
                     │              │                 │
                     │  ┌───────────▼─────────────┐  │
                     │  │  Persistent Disk (SSD)    │  │
                     │  │  (compute + storage       │  │
                     │  │   ผูกกันในโซนเดียว)         │  │
                     │  └─────────────────────────┘  │
                     └─────────────────────────────┘
                          zone: asia-southeast1-a
```

### การเปิดใช้งาน Cloud SQL Admin API และสร้าง instance เบื้องต้น

ก่อนใช้งาน Cloud SQL ต้องเปิดใช้ API ก่อน (ทำครั้งเดียวต่อโปรเจกต์):

```bash
# ตั้งค่า project ปัจจุบัน
gcloud config set project my-ecommerce-project

# เปิดใช้งาน Cloud SQL Admin API
gcloud services enable sqladmin.googleapis.com

# ตรวจสอบว่า API เปิดใช้งานแล้ว
gcloud services list --enabled --filter="name:sqladmin.googleapis.com"
```

การสร้าง Cloud SQL instance แบบพื้นฐานที่สุด (single zone, สำหรับ dev/test):

```bash
gcloud sql instances create ecommerce-dev-pg \
  --database-version=POSTGRES_16 \
  --tier=db-custom-2-8192 \
  --region=asia-southeast1 \
  --storage-type=SSD \
  --storage-size=20GB \
  --root-password="ChangeMe!StrongPassword123" \
  --availability-type=ZONAL
```

**คำอธิบายพารามิเตอร์สำคัญ**:

- `--database-version=POSTGRES_16` — เลือกเวอร์ชัน PostgreSQL (Cloud SQL รองรับตั้งแต่ PostgreSQL 12 ขึ้นไป ขึ้นอยู่กับช่วงเวลาที่อ่านบทเรียนนี้ ควรตรวจสอบเวอร์ชันล่าสุดที่รองรับจากเอกสารทางการ)
- `--tier` — ขนาดเครื่อง (จะอธิบายละเอียดใน Step 942)
- `--availability-type=ZONAL` — instance เดี่ยว ไม่มี HA (เทียบกับ `REGIONAL` ใน Step 943)
- `--root-password` — รหัสผ่านของ default superuser (`postgres`)

### กลุ่มบริการที่เกี่ยวข้อง

Cloud SQL ทำงานร่วมกับบริการ GCP อื่น ๆ ในระบบนิเวศ ได้แก่:

| บริการ | บทบาท |
|---|---|
| VPC | เครือข่ายส่วนตัวสำหรับเชื่อมต่อผ่าน Private IP |
| Cloud Monitoring / Cloud Logging | เฝ้าระวังและเก็บ log ของ instance |
| IAM | ควบคุมสิทธิ์เข้าถึง Cloud SQL Admin API และ database-level IAM authentication |
| Cloud SQL Auth Proxy / Cloud SQL connectors | เชื่อมต่อฐานข้อมูลอย่างปลอดภัยจาก client |
| Cloud KMS | เข้ารหัสข้อมูลด้วย customer-managed encryption key (CMEK) |
| Secret Manager | เก็บ credential ของฐานข้อมูลอย่างปลอดภัย |

---

## Step 942: Cloud SQL Instance Configuration

การกำหนดค่า instance ให้เหมาะสมกับ workload เป็นทักษะที่สำคัญที่สุดอย่างหนึ่งของสถาปนิกระบบ เพราะกระทบทั้งประสิทธิภาพและค่าใช้จ่ายโดยตรง

### Machine Type (Tier)

Cloud SQL แบ่ง machine type ออกเป็น 2 รูปแบบ:

**1. Shared-core tier (สำหรับ dev/test เท่านั้น ไม่แนะนำสำหรับ production)**

```
db-f1-micro   — 1 shared vCPU, 0.6 GB RAM
db-g1-small   — 1 shared vCPU, 1.7 GB RAM
```

**2. Custom machine type (แนะนำสำหรับ production)** — รูปแบบ `db-custom-<vCPU>-<RAM in MB>`

```bash
# ตัวอย่าง: 4 vCPU, 16 GB RAM
--tier=db-custom-4-16384

# ตัวอย่าง: 8 vCPU, 32 GB RAM
--tier=db-custom-8-32768

# ตัวอย่าง: 16 vCPU, 64 GB RAM
--tier=db-custom-16-65536
```

**กฎการกำหนดค่า custom machine type ที่ควรรู้**:

- อัตราส่วน RAM ต่อ vCPU ต้องอยู่ระหว่าง 0.9 GB ถึง 6.5 GB ต่อ vCPU (โดยประมาณ ขึ้นกับ generation ของเครื่อง)
- จำนวน vCPU ต้องเป็นเลขคู่ (ยกเว้น 1 vCPU)
- ควรเลือก **Enterprise Plus edition** สำหรับ workload ที่ต้องการประสิทธิภาพสูงสุดและ near-zero downtime maintenance เทียบกับ **Enterprise edition** ที่เป็นมาตรฐานทั่วไป

ตัวอย่างการสร้าง instance ที่ระบุ edition:

```bash
gcloud sql instances create ecommerce-prod-pg \
  --database-version=POSTGRES_16 \
  --edition=ENTERPRISE_PLUS \
  --tier=db-perf-optimized-N-8 \
  --region=asia-southeast1 \
  --storage-type=SSD \
  --storage-size=100GB \
  --storage-auto-increase \
  --availability-type=REGIONAL
```

> หมายเหตุ: `db-perf-optimized-N-*` เป็นตัวอย่าง tier รุ่นใหม่ที่ออกแบบมาสำหรับ Enterprise Plus edition โดยเฉพาะ (performance-optimized) ชื่อ tier ที่แน่นอนควรตรวจสอบจาก `gcloud sql tiers list --region=asia-southeast1` ณ เวลาที่ใช้งานจริง

### การเลือก Storage Type

| Storage type | ลักษณะ | เหมาะกับ |
|---|---|---|
| **SSD (Solid State Drive)** | latency ต่ำ, throughput สูง | ทุก production workload แทบทั้งหมด |
| **HDD (Hard Disk Drive)** | ราคาถูกกว่า แต่ latency สูงกว่ามาก | เฉพาะ archive/cold data ที่ไม่ sensitive ต่อ latency |

โดยทั่วไป **แนะนำ SSD เสมอสำหรับ production** เพราะ IOPS และ throughput scale ตาม storage size และ machine tier โดยอัตโนมัติ (คล้าย gp3 ของ AWS ที่ throughput ผูกกับขนาด disk)

```bash
# เปิด storage auto-increase เพื่อป้องกันปัญหา disk full
gcloud sql instances patch ecommerce-prod-pg \
  --storage-auto-increase \
  --storage-auto-increase-limit=500GB
```

`--storage-auto-increase-limit` ช่วยป้องกันไม่ให้ storage ขยายแบบไม่มีเพดาน (ซึ่งอาจทำให้ค่าใช้จ่ายบานปลายโดยไม่รู้ตัว)

### การเลือก Region และ Zone

หลักการเลือก region/zone สำหรับ Cloud SQL:

1. **Latency กับ application** — วาง Cloud SQL instance ให้อยู่ region เดียวกับ compute layer (GKE, Compute Engine, Cloud Run) เสมอ เพื่อลด network latency
2. **Data residency / กฎหมาย** — บาง workload ต้องเก็บข้อมูลในประเทศ (เช่น ข้อมูลลูกค้าไทยอาจต้องอยู่ region `asia-southeast1` สิงคโปร์ หรือถ้ามีข้อกำหนดเฉพาะเจาะจงกว่านั้นต้องตรวจสอบ region ที่ใกล้ที่สุดตามนโยบายองค์กร)
3. **Zone ภายใน region** — สำหรับ instance แบบ `ZONAL` ควรเลือก zone เดียวกับ compute layer หลัก แต่ปล่อยให้ `REGIONAL` (HA) จัดการ zone รองให้อัตโนมัติ

```bash
# ตรวจสอบ region/zone ที่รองรับ Cloud SQL
gcloud sql tiers list

# ระบุ zone เฉพาะเจาะจง (เมื่อใช้ ZONAL availability)
gcloud sql instances create ecommerce-dev-pg \
  --zone=asia-southeast1-a \
  --database-version=POSTGRES_16 \
  --tier=db-custom-2-8192
```

### ตารางสรุปการตัดสินใจ (Decision Table) สำหรับ Instance Sizing

| Workload | vCPU แนะนำ | RAM แนะนำ | Storage | Availability |
|---|---|---|---|---|
| Dev/Test | 1-2 | 4-8 GB | 20-50 GB SSD | ZONAL |
| Small production API | 2-4 | 8-16 GB | 50-100 GB SSD | REGIONAL |
| Medium e-commerce (ออเดอร์หลักแสน/วัน) | 4-8 | 16-32 GB | 100-500 GB SSD | REGIONAL |
| High-traffic e-commerce / reporting รวม | 16+ | 64+ GB | 500 GB+ SSD | REGIONAL + Read Replica |

---

## Step 943: Cloud SQL High Availability

### แนวคิด Regional Instance

Cloud SQL รองรับ High Availability ผ่านสิ่งที่เรียกว่า **Regional Instance** (บางเอกสารเรียกว่า HA configuration) โดยมีหลักการทำงานดังนี้:

- instance หลัก (**primary**) รันอยู่ใน zone หนึ่ง
- มี **standby instance** รันอยู่อีก zone หนึ่งภายใน region เดียวกัน
- ข้อมูลจาก primary จะถูก **replicate แบบ synchronous** ไปยัง standby ผ่านกลไกที่เรียกว่า **regional persistent disk** (บาง generation) หรือ physical streaming replication ขึ้นกับสถาปัตยกรรมรุ่นของ Cloud SQL
- เมื่อ primary ล้มเหลว (zone outage, hardware failure) Cloud SQL จะทำ **automatic failover** ไปยัง standby โดยอัตโนมัติ และเปลี่ยน DNS/IP ให้ชี้ไปยัง standby ที่กลายเป็น primary ใหม่

```
                    Region: asia-southeast1
        ┌───────────────────────────────────────────────┐
        │  Zone A (asia-southeast1-a)                     │
        │  ┌─────────────────────────┐                    │
        │  │  Primary Instance         │                    │
        │  │  (accepts read/write)     │                    │
        │  └────────────┬─────────────┘                    │
        │               │ synchronous replication           │
        │               │ (ข้าม zone)                        │
        │  Zone B (asia-southeast1-b)                       │
        │  ┌────────────▼─────────────┐                    │
        │  │  Standby Instance          │                    │
        │  │  (ไม่รับ traffic ปกติ)       │                    │
        │  └────────────────────────────┘                    │
        └───────────────────────────────────────────────┘
                        │
                        │  หาก Zone A ล่ม → automatic failover
                        ▼
              Standby กลายเป็น Primary ใหม่ทันที
              (application เชื่อมต่อผ่าน instance connection name เดิม)
```

### การสร้าง Regional Instance

```bash
gcloud sql instances create ecommerce-prod-pg \
  --database-version=POSTGRES_16 \
  --tier=db-custom-8-32768 \
  --region=asia-southeast1 \
  --availability-type=REGIONAL \
  --storage-type=SSD \
  --storage-size=200GB \
  --backup-start-time=18:00 \
  --maintenance-window-day=SUN \
  --maintenance-window-hour=3
```

การ **แปลง instance ที่มีอยู่แล้วจาก ZONAL เป็น REGIONAL** ทำได้ในภายหลังเช่นกัน (ทำให้เกิด restart สั้น ๆ):

```bash
gcloud sql instances patch ecommerce-prod-pg \
  --availability-type=REGIONAL
```

### การทดสอบ Failover ด้วยตนเอง

Cloud SQL อนุญาตให้ trigger manual failover เพื่อทดสอบความพร้อมของระบบ (แนะนำให้ทำในช่วง maintenance window หรือ low-traffic):

```bash
gcloud sql instances failover ecommerce-prod-pg
```

คำสั่งนี้จะบังคับให้ standby กลายเป็น primary ทันที ใช้สำหรับซ้อม Disaster Recovery (DR) หรือทดสอบว่า application reconnect ได้ถูกต้องเมื่อเกิด failover จริง

### เปรียบเทียบ Cloud SQL Regional HA กับ RDS Multi-AZ

| แง่มุม | AWS RDS Multi-AZ | GCP Cloud SQL Regional (HA) |
|---|---|---|
| กลไก replication | Synchronous physical replication | Synchronous replication ข้าม zone |
| Standby รับ read traffic ได้ไหม | ไม่ได้ (Multi-AZ แบบดั้งเดิม) / ได้ใน Multi-AZ DB Cluster รุ่นใหม่ | ไม่ได้ (standby สำรองไว้สำหรับ failover เท่านั้น) |
| RTO (Recovery Time Objective) โดยประมาณ | ~60-120 วินาที | ~60-120 วินาที (ขึ้นกับขนาด instance และ workload) |
| RPO (Recovery Point Objective) | ใกล้ 0 (synchronous) | ใกล้ 0 (synchronous) |
| การ trigger failover ทดสอบ | `aws rds reboot-db-instance --force-failover` | `gcloud sql instances failover` |
| ค่าใช้จ่ายเพิ่มเติม | ~2 เท่าของ Single-AZ | ~2 เท่าของ ZONAL |

**สรุปสำคัญ**: กลไกพื้นฐานของทั้งสองระบบเหมือนกันในทางแนวคิด คือ "standby ข้าม zone ที่ sync กันแบบ synchronous เพื่อ failover อัตโนมัติ" แต่ standby จะ**ไม่**ถูกใช้เพื่อรับ read traffic ทั้งใน Cloud SQL และ RDS Multi-AZ แบบดั้งเดิม — หากต้องการ scale การอ่าน ต้องใช้ Read Replica แยกต่างหาก (ดู Step 944)

---

## Step 944: Cloud SQL Read Replica

### วัตถุประสงค์ของ Read Replica

Read Replica ใน Cloud SQL ใช้สำหรับ:

1. **Read scaling** — กระจาย read query (เช่น รายงาน, dashboard, search) ออกจาก primary instance เพื่อลด load
2. **Disaster Recovery ข้าม region** — cross-region replica ทำหน้าที่เป็น DR site หาก region หลักล่มทั้ง region
3. **Analytical workload แยกจาก transactional** — รัน query หนัก ๆ (เช่น batch report, ETL) บน replica แทนที่จะกระทบ primary ที่รับ order transaction

Read Replica ใน Cloud SQL ใช้กลไก **asynchronous replication** (ต่างจาก HA standby ที่เป็น synchronous)

### ประเภทของ Read Replica

| ประเภท | คำอธิบาย |
|---|---|
| **Same-region read replica** | อยู่ region เดียวกับ primary, latency ต่ำ, เหมาะกับ read scaling |
| **Cross-region read replica** | อยู่คนละ region, เหมาะกับ DR และการลด latency ให้ผู้ใช้ในภูมิภาคอื่น |
| **Cascading replica** | replica ของ replica (รองรับในบาง configuration) เพื่อลด load การ replicate จาก primary โดยตรง |

### การสร้าง Read Replica

```bash
# สร้าง read replica ใน region เดียวกัน
gcloud sql instances create ecommerce-prod-pg-replica-1 \
  --master-instance-name=ecommerce-prod-pg \
  --tier=db-custom-4-16384 \
  --region=asia-southeast1 \
  --availability-type=ZONAL

# สร้าง cross-region read replica (เช่น เก็บ DR ไว้ที่ Tokyo)
gcloud sql instances create ecommerce-prod-pg-replica-dr \
  --master-instance-name=ecommerce-prod-pg \
  --tier=db-custom-4-16384 \
  --region=asia-northeast1 \
  --availability-type=ZONAL
```

```
        Region: asia-southeast1 (Singapore)          Region: asia-northeast1 (Tokyo)
        ┌───────────────────────────┐                ┌───────────────────────────┐
        │  Primary Instance           │  async         │  Cross-region              │
        │  ecommerce-prod-pg          │───────────────▶│  Read Replica (DR)         │
        │  (read/write)               │  replication   │  ecommerce-prod-pg-        │
        └──────────────┬─────────────┘                │  replica-dr                 │
                        │ async                        └───────────────────────────┘
                        ▼
        ┌───────────────────────────┐
        │  Same-region Read Replica   │  ← รับ traffic รายงาน/dashboard
        │  ecommerce-prod-pg-         │
        │  replica-1                  │
        └───────────────────────────┘
```

### การ Monitor ความล่าช้าของ Replication (Replica Lag)

```bash
# ดูสถานะและ replication lag ของ replica
gcloud sql instances describe ecommerce-prod-pg-replica-1 \
  --format="value(state, replicaConfiguration)"
```

ในทางปฏิบัติ ควรตั้ง Cloud Monitoring alert บน metric `cloudsql.googleapis.com/database/replication/replica_lag` เพื่อแจ้งเตือนเมื่อ replica lag เกินค่าที่ยอมรับได้ (เช่น > 30 วินาที)

### การ Promote Replica เป็น Standalone (สำหรับ DR จริง)

หาก primary region ล่มทั้ง region และต้องการ promote cross-region replica เป็น primary ใหม่ถาวร:

```bash
gcloud sql instances promote-replica ecommerce-prod-pg-replica-dr
```

**ข้อควรระวัง**: การ promote เป็นการดำเนินการที่ **ย้อนกลับไม่ได้** (irreversible) — instance ที่ถูก promote จะกลายเป็น standalone instance ที่รับ read/write เอง และตัดขาดจาก replication chain เดิมโดยสมบูรณ์ ต้องวางแผน DNS/connection string switching ให้ application ชี้มาที่ instance ใหม่ด้วย

### แนวทางการใช้งาน Read Replica สำหรับ e-commerce

| Use case | แนะนำ replica ประเภทใด |
|---|---|
| หน้า product catalog / search ที่ traffic สูง | Same-region read replica |
| รายงานยอดขายรายวัน/ETL เข้า data warehouse | Same-region read replica (แยกจาก replica ที่ serve production read) |
| รองรับผู้ใช้ในภูมิภาคอื่น (latency ต่ำ) | Cross-region read replica |
| Disaster Recovery ระดับ region | Cross-region read replica + runbook สำหรับ promote |

---

## Step 945: Cloud SQL Backup และ PITR

### Automated Backup

Cloud SQL รองรับการสำรองข้อมูลอัตโนมัติรายวัน (automated backup) โดยมีคุณสมบัติ:

- เก็บ backup แบบ **incremental** เพื่อประหยัดพื้นที่จัดเก็บ (แต่ restore ได้แบบ full backup)
- กำหนดเวลาที่ backup เริ่มทำงานได้ (`backup-start-time`) — ควรเลือกช่วง low-traffic
- ค่าเริ่มต้นเก็บ backup ย้อนหลังได้ (retention) ปรับได้สูงสุดถึง 365 วัน (ขึ้นกับ edition/plan)

```bash
# เปิดใช้งาน automated backup พร้อมกำหนดเวลา และจำนวนวันที่เก็บ
gcloud sql instances patch ecommerce-prod-pg \
  --backup-start-time=18:00 \
  --backup-location=asia-southeast1 \
  --retained-backups-count=30 \
  --retained-transaction-log-days=7
```

**พารามิเตอร์สำคัญ**:

- `--retained-backups-count` — จำนวน full backup ที่เก็บไว้ (เช่น 30 = เก็บย้อนหลัง 30 ครั้ง)
- `--retained-transaction-log-days` — จำนวนวันที่เก็บ WAL (transaction log) ไว้สำหรับทำ PITR

### Point-in-Time Recovery (PITR)

PITR ใช้กลไกเดียวกับที่เราเรียนใน Part เกี่ยวกับ WAL และ Continuous Archiving (ภาค Backup & Recovery ต้น ๆ ของหลักสูตร) คือการนำ **base backup + WAL ที่เก็บไว้ต่อเนื่อง** มา replay จนถึงจุดเวลาที่ต้องการ

**เงื่อนไขสำคัญ**: ต้องเปิด `point-in-time-recovery` ไว้ (โดยการเปิด transaction log retention ด้านบน) มิฉะนั้นจะกู้คืนได้เฉพาะจุด backup ที่มีอยู่เท่านั้น (ไม่สามารถกู้คืนไปยังเวลาใด ๆ ระหว่าง backup ได้)

```bash
# เปิดใช้งาน point-in-time recovery อย่างชัดเจน
gcloud sql instances patch ecommerce-prod-pg \
  --enable-point-in-time-recovery
```

### การกู้คืนข้อมูล ณ จุดเวลาที่กำหนด

```bash
# กู้คืนไปยัง instance ใหม่ ณ เวลาที่ระบุ (ISO 8601, UTC)
gcloud sql instances clone ecommerce-prod-pg ecommerce-prod-pg-restored \
  --point-in-time="2026-09-24T15:30:00.000Z"
```

หรือกู้คืนจาก backup ที่ระบุโดยตรง:

```bash
# แสดงรายการ backup ที่มีอยู่
gcloud sql backups list --instance=ecommerce-prod-pg

# กู้คืนจาก backup id ที่ระบุ ไปยัง instance ใหม่
gcloud sql backups restore BACKUP_ID \
  --restore-instance=ecommerce-prod-pg-restored \
  --backup-instance=ecommerce-prod-pg
```

> **ข้อสำคัญที่ต้องเน้นย้ำ**: การ restore ของ Cloud SQL (เหมือน RDS) จะสร้าง **instance ใหม่เสมอ** ไม่ใช่การ restore ทับ instance เดิม ดังนั้นต้องวางแผนเรื่อง connection string/DNS switching ทุกครั้งที่ทำ disaster recovery drill หรือ restore จริง

### On-demand Backup (Manual)

นอกจาก automated backup ยังสามารถสั่ง backup ทันทีได้ก่อนทำการเปลี่ยนแปลงเสี่ยง เช่น ก่อน deploy migration ใหญ่:

```bash
gcloud sql backups create \
  --instance=ecommerce-prod-pg \
  --description="ก่อน deploy migration v2.5.0"
```

### ตารางสรุปกลยุทธ์ Backup สำหรับ e-commerce

| สถานการณ์ | กลยุทธ์ |
|---|---|
| ป้องกันความผิดพลาดจาก deploy | On-demand backup ก่อน deploy ทุกครั้งที่มี schema migration |
| RPO ต้องน้อยกว่า 5 นาที | เปิด PITR + retained-transaction-log-days ≥ 7 |
| Compliance ต้องเก็บ backup ย้อนหลังนาน | เพิ่ม `retained-backups-count` และพิจารณา export ไปยัง Cloud Storage ด้วย `gcloud sql export` สำหรับ long-term archive |
| ต้องการ backup แบบ cross-region สำหรับ compliance | ใช้ `--backup-location` เพื่อกำหนดตำแหน่งจัดเก็บ backup แยกจาก region ของ instance |

---

## Step 946: Cloud SQL Auth Proxy

### ปัญหาที่ Cloud SQL Auth Proxy แก้

การเปิด **public IP** ให้ Cloud SQL instance แล้วเชื่อมต่อตรงจาก application เป็นแนวทางที่มีความเสี่ยงด้านความปลอดภัยสูง (exposed attack surface, ต้องจัดการ firewall/authorized networks เอง, ต้องจัดการ SSL certificate เอง) **Cloud SQL Auth Proxy** คือ client-side proxy ที่ Google จัดทำให้ เพื่อแก้ปัญหานี้โดย:

1. เข้ารหัสการเชื่อมต่อด้วย **TLS 1.3** โดยอัตโนมัติ (ไม่ต้องจัดการ certificate เอง)
2. ยืนยันตัวตนด้วย **IAM credentials** แทนการเปิด public IP + whitelist IP address
3. ทำงานเป็น **local proxy** ที่ application เชื่อมต่อผ่าน `localhost` (หรือ Unix socket) เสมือนฐานข้อมูลอยู่ในเครื่อง

### สถาปัตยกรรมการเชื่อมต่อผ่าน Auth Proxy

```
   ┌────────────────────────────────┐
   │  Application Container/VM        │
   │  ┌────────────────────────┐      │
   │  │  App connects to:        │      │
   │  │  127.0.0.1:5432          │      │
   │  └────────────┬─────────────┘     │
   │               │                    │
   │  ┌────────────▼─────────────┐     │
   │  │  Cloud SQL Auth Proxy      │     │
   │  │  (sidecar process)         │     │
   │  └────────────┬─────────────┘     │
   └───────────────┼────────────────────┘
                    │  Encrypted tunnel (mTLS)
                    │  ยืนยันตัวตนด้วย IAM Service Account
                    ▼
        ┌───────────────────────────┐
        │  Cloud SQL Instance          │
        │  (ไม่ต้องเปิด public IP เลย)   │
        └───────────────────────────┘
```

### การติดตั้งและใช้งาน (ตัวอย่างบน Linux/Compute Engine)

```bash
# ดาวน์โหลด Cloud SQL Auth Proxy binary (เวอร์ชัน v2)
curl -o cloud-sql-proxy \
  https://storage.googleapis.com/cloud-sql-connectors/cloud-sql-proxy/v2.13.0/cloud-sql-proxy.linux.amd64
chmod +x cloud-sql-proxy

# รัน proxy โดยระบุ instance connection name
# รูปแบบ instance connection name: PROJECT_ID:REGION:INSTANCE_ID
./cloud-sql-proxy my-ecommerce-project:asia-southeast1:ecommerce-prod-pg \
  --port=5432
```

จากนั้น application เชื่อมต่อเหมือนเชื่อมต่อ PostgreSQL ปกติที่ `localhost:5432`:

```bash
psql "host=127.0.0.1 port=5432 dbname=ecommerce user=app_user sslmode=disable"
```

> **หมายเหตุ**: เมื่อเชื่อมต่อผ่าน Auth Proxy ไม่จำเป็นต้องตั้งค่า `sslmode=require` ที่ฝั่ง client เพราะ Proxy ทำการเข้ารหัสให้ระหว่าง Proxy กับ Cloud SQL instance อยู่แล้ว (encrypted tunnel แยกชั้นจาก connection string ของ application)

### การใช้งานใน Kubernetes (GKE) แบบ Sidecar Container

รูปแบบที่นิยมที่สุดในการ deploy บน GKE คือรัน Auth Proxy เป็น **sidecar container** ในทุก pod ที่ต้องเชื่อมต่อฐานข้อมูล:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  template:
    spec:
      serviceAccountName: order-service-sa
      containers:
        - name: order-service
          image: asia-southeast1-docker.pkg.dev/my-ecommerce-project/app/order-service:latest
          env:
            - name: DB_HOST
              value: "127.0.0.1"
            - name: DB_PORT
              value: "5432"
        - name: cloud-sql-proxy
          image: gcr.io/cloud-sql-connectors/cloud-sql-proxy:2.13.0
          args:
            - "--structured-logs"
            - "--port=5432"
            - "my-ecommerce-project:asia-southeast1:ecommerce-prod-pg"
          securityContext:
            runAsNonRoot: true
```

### การให้สิทธิ์ IAM ที่จำเป็น

Service Account ที่ใช้รัน Auth Proxy ต้องมี role `roles/cloudsql.client` เป็นอย่างน้อย:

```bash
gcloud projects add-iam-policy-binding my-ecommerce-project \
  --member="serviceAccount:order-service-sa@my-ecommerce-project.iam.gserviceaccount.com" \
  --role="roles/cloudsql.client"
```

### เปรียบเทียบ Cloud SQL Auth Proxy กับ RDS Proxy

| แง่มุม | AWS RDS Proxy | GCP Cloud SQL Auth Proxy |
|---|---|---|
| ตำแหน่งการทำงาน | Managed proxy ฝั่ง AWS (server-side) | Client-side binary ที่รันข้าง application |
| จุดประสงค์หลัก | Connection pooling + failover handling | ความปลอดภัยของการเชื่อมต่อ (encryption + IAM auth) โดยไม่เปิด public IP |
| Connection pooling ในตัว | มี (RDS Proxy ทำ pooling ระดับ managed service) | ไม่มี pooling ในตัว (ควรใช้ร่วมกับ PgBouncer/pgpool หากต้องการ pooling) |
| ค่าใช้จ่ายเพิ่มเติม | มีค่าบริการแยกต่างหาก | ไม่มีค่าบริการเพิ่มเติม (เป็น client binary ฟรี) |

**ข้อสังเกต**: Cloud SQL Auth Proxy ไม่ใช่ connection pooler เหมือน RDS Proxy — มันแก้ปัญหาเรื่อง **security/encryption/authentication** เป็นหลัก หากต้องการ connection pooling ในระบบ e-commerce ที่มี concurrent connection สูง ควรใช้ PgBouncer ควบคู่กันไปด้วย (ตามที่เรียนใน Part เกี่ยวกับ Connection Pooling ก่อนหน้านี้ของหลักสูตร)

---

## Step 947: AlloyDB for PostgreSQL คืออะไร

### จุดกำเนิดและตำแหน่งของ AlloyDB

**AlloyDB for PostgreSQL** คือบริการ managed database ของ Google ที่เปิดตัวเพื่อเป็นคำตอบโดยตรงต่อ **Amazon Aurora PostgreSQL** — เป็นบริการที่ **100% compatible กับ PostgreSQL wire protocol และ SQL syntax มาตรฐาน** (application ที่เขียนสำหรับ PostgreSQL ทั่วไปสามารถใช้งานกับ AlloyDB ได้โดยแทบไม่ต้องแก้โค้ด) แต่ภายในมีสถาปัตยกรรมที่ถูกออกแบบใหม่ทั้งหมดเพื่อประสิทธิภาพที่สูงกว่า Cloud SQL อย่างมีนัยสำคัญ

Google อ้างอิงตัวเลขประสิทธิภาพ (จากเอกสาร marketing ของ Google เอง ควรตรวจสอบ benchmark ล่าสุดก่อนใช้ตัดสินใจ) ว่า AlloyDB สามารถทำ transactional throughput ได้เร็วกว่า PostgreSQL มาตรฐานหลายเท่า และ analytical query เร็วกว่ามาก เนื่องจาก columnar engine (ดู Step 948)

### สถาปัตยกรรม Disaggregated Storage

หัวใจของ AlloyDB คือการ**แยก compute ออกจาก storage อย่างสมบูรณ์** (disaggregated storage) ซึ่งเป็นแนวคิดเดียวกับที่ Aurora ใช้ (log-structured distributed storage) แต่ Google implement ด้วยเทคโนโลยีของตนเอง

```
                 ┌───────────────────────────────────────┐
                 │            Compute Layer                  │
                 │                                             │
                 │  ┌──────────────┐   ┌──────────────┐      │
                 │  │  Primary       │   │  Read Pool     │      │
                 │  │  Instance      │   │  Instance(s)   │      │
                 │  │  (read/write)  │   │  (read-only)   │      │
                 │  └───────┬────────┘   └───────┬────────┘      │
                 │          │ WAL only            │ ไม่ replicate │
                 │          │ (log records)        │ full data,    │
                 │          │                       │ อ่านจาก        │
                 └──────────┼───────────────────────┼─ storage ───┘
                            │                       │  layer โดยตรง
                            ▼                       ▼
                 ┌───────────────────────────────────────┐
                 │      Disaggregated Storage Layer          │
                 │      (distributed, log-structured,        │
                 │       auto-scaling, replicated ข้าม zone)  │
                 └───────────────────────────────────────┘
```

**ความแตกต่างสำคัญจาก Cloud SQL**:

1. **Primary instance ส่งเฉพาะ WAL (log record) ไปยัง storage layer** ไม่ต้องเขียน full page ลง disk เอง (คล้ายหลักการ "log is the database" ของ Aurora) ทำให้ write throughput สูงขึ้นและลด I/O amplification
2. **Read Pool instances อ่านข้อมูลจาก storage layer เดียวกันโดยตรง** แทนที่จะต้องรอ replication แบบ streaming เต็มรูปแบบเหมือน Cloud SQL read replica ทำให้ replica lag ต่ำกว่ามาก (มักอยู่ในหลักวินาทีหรือต่ำกว่า)
3. **Storage ขยายอัตโนมัติแบบไม่มีขีดจำกัดที่ต้องกำหนดล่วงหน้า** และมีการ replicate ข้าม zone อยู่ภายในตัวโดยไม่ต้องตั้งค่า HA แยกเหมือน Cloud SQL Regional instance (AlloyDB HA เป็นความสามารถพื้นฐานของสถาปัตยกรรม)

### การสร้าง AlloyDB Cluster และ Instance

AlloyDB มีหน่วยการทำงานเป็น 2 ระดับ: **Cluster** (ระดับ storage + configuration) และ **Instance** (ระดับ compute — primary หรือ read pool)

```bash
# เปิดใช้งาน AlloyDB API
gcloud services enable alloydb.googleapis.com

# สร้าง network peering ที่จำเป็น (AlloyDB ต้องใช้ Private Service Access)
gcloud compute addresses create alloydb-range \
  --global \
  --purpose=VPC_PEERING \
  --prefix-length=16 \
  --network=my-vpc

gcloud services vpc-peerings connect \
  --service=servicenetworking.googleapis.com \
  --ranges=alloydb-range \
  --network=my-vpc

# สร้าง AlloyDB cluster
gcloud alloydb clusters create ecommerce-alloydb-cluster \
  --region=asia-southeast1 \
  --network=my-vpc \
  --password="ChangeMe!StrongPassword123"

# สร้าง primary instance ภายใน cluster
gcloud alloydb instances create ecommerce-alloydb-primary \
  --cluster=ecommerce-alloydb-cluster \
  --region=asia-southeast1 \
  --instance-type=PRIMARY \
  --cpu-count=8 \
  --memory-size=64GiB
```

### การเพิ่ม Read Pool Instance

```bash
gcloud alloydb instances create ecommerce-alloydb-readpool \
  --cluster=ecommerce-alloydb-cluster \
  --region=asia-southeast1 \
  --instance-type=READ_POOL \
  --read-pool-node-count=3 \
  --cpu-count=4 \
  --memory-size=32GiB
```

`--read-pool-node-count=3` หมายถึง read pool นี้ประกอบด้วย 3 nodes ที่ทำงานแบบ load-balanced ภายใต้ endpoint เดียวกัน — application เชื่อมต่อผ่าน connection endpoint เดียว แล้ว AlloyDB จะกระจาย query ไปยัง node ที่ว่างที่สุดให้อัตโนมัติ (ต่างจาก Cloud SQL ที่ read replica แต่ละตัวมี endpoint แยกกันเอง ต้องทำ load balancing เองที่ฝั่ง application หรือใช้ HAProxy/PgBouncer เพิ่มเติม)

### เปรียบเทียบ AlloyDB กับ Cloud SQL และ Aurora

| แง่มุม | Cloud SQL | AlloyDB | AWS Aurora PostgreSQL |
|---|---|---|---|
| สถาปัตยกรรม storage | ผูกกับ compute (Persistent Disk) | Disaggregated (แยกจาก compute) | Disaggregated (distributed log storage) |
| Read replica lag | วินาที-นาที (async) | ต่ำมาก (millisecond-level ในหลายกรณี) | ต่ำมาก (millisecond-level) |
| Read scaling | Read Replica (endpoint แยก, ต้อง load balance เอง) | Read Pool (endpoint เดียว, auto load-balanced) | Reader endpoint (auto load-balanced ผ่าน Aurora reader endpoint) |
| Analytical acceleration ในตัว | ไม่มี | มี (Columnar Engine — ดู Step 948) | ไม่มีในตัว (ต้องพึ่ง extension/Redshift) |
| ความ compatible กับ PostgreSQL | 100% (เป็น PostgreSQL จริง) | 100% (wire-compatible) | 100% (wire-compatible) |
| เหมาะกับ | Workload ทั่วไป, งบจำกัด, ต้องการความเรียบง่าย | Workload ที่ต้องการประสิทธิภาพสูง, HTAP, mission-critical | Workload ที่ต้องการประสิทธิภาพสูงบน AWS |

---

## Step 948: AlloyDB Columnar Engine

### ปัญหาที่ Columnar Engine แก้

PostgreSQL มาตรฐาน (รวมถึง Cloud SQL) จัดเก็บข้อมูลแบบ **row-oriented storage** (เก็บข้อมูลทีละแถวต่อเนื่องกันใน page) ซึ่งเหมาะมากสำหรับ **OLTP** (Online Transaction Processing) เช่น การ insert/update ออเดอร์ทีละรายการ แต่**ไม่เหมาะกับ analytical query** ที่ต้อง scan และ aggregate คอลัมน์จำนวนมากจากหลายล้านแถว (เช่น `SUM(total_amount) GROUP BY region, month`) เพราะต้องอ่านทั้งแถวทั้งที่ใช้แค่ไม่กี่คอลัมน์

### แนวคิด Hybrid Transactional/Analytical Processing (HTAP)

**AlloyDB Columnar Engine** คือฟีเจอร์ที่เก็บ **สำเนาข้อมูลบางคอลัมน์ไว้ในหน่วยความจำ (in-memory) ในรูปแบบ columnar representation** ควบคู่ไปกับข้อมูล row-oriented ปกติบน disk โดย PostgreSQL query planner จะเลือกใช้ columnar representation โดยอัตโนมัติเมื่อพบว่า query นั้นเป็น analytical pattern (scan จำนวนมาก, aggregate, filter บนหลายคอลัมน์) นี่คือที่มาของคำว่า **HTAP** — ฐานข้อมูลเดียวรองรับทั้ง Transactional และ Analytical workload ได้ดีพร้อมกัน โดยไม่ต้อง ETL ข้อมูลออกไปยัง data warehouse แยกต่างหากสำหรับ query แบบ real-time analytics

```
                    ┌─────────────────────────────────────┐
                    │          AlloyDB Instance                │
                    │                                            │
     OLTP query     │  ┌──────────────────┐                     │
     (order insert)  │  │  Row-oriented       │  ← ใช้สำหรับ         │
     ──────────────▶│  │  storage (on disk)  │     transactional   │
                    │  └──────────────────┘     query (INSERT/    │
                    │                              UPDATE/point    │
                    │                              lookup)          │
                    │                                            │
     Analytical      │  ┌──────────────────┐                     │
     query (report,  │  │  Columnar Engine    │  ← ใช้สำหรับ         │
     dashboard)      │  │  (in-memory,        │     aggregate/scan  │
     ──────────────▶│  │   auto-synced       │     query จำนวนมาก   │
                    │  │   จาก row storage)  │                     │
                    │  └──────────────────┘                     │
                    │                                            │
                    │  Query Planner เลือก engine ที่เหมาะสม        │
                    │  ให้อัตโนมัติตาม query pattern                │
                    └─────────────────────────────────────┘
```

### การเปิดใช้งาน Columnar Engine

Columnar Engine ควบคุมผ่าน extension และ system flag บน AlloyDB instance:

```sql
-- ตรวจสอบว่า extension พร้อมใช้งาน
CREATE EXTENSION IF NOT EXISTS google_columnar_engine;

-- กำหนดให้ตารางที่ต้องการเร่งความเร็ว analytical query
-- ถูกโหลดเข้า columnar engine
CALL google_columnar_engine_add(
  relation => 'public.order_items',
  columns  => ARRAY['product_id', 'quantity', 'unit_price', 'order_date']
);

-- ตรวจสอบสถานะตารางที่อยู่ใน columnar engine
SELECT * FROM google_columnar_engine_tables();
```

การปรับขนาดหน่วยความจำที่จัดสรรให้ columnar engine ทำผ่าน database flag ของ instance:

```bash
gcloud alloydb instances update ecommerce-alloydb-primary \
  --cluster=ecommerce-alloydb-cluster \
  --region=asia-southeast1 \
  --database-flags=google_columnar_engine.enabled=on,google_columnar_engine.memory_size_in_mb=8192
```

### ตัวอย่างสถานการณ์ที่ Columnar Engine ช่วยได้จริง

สมมติระบบ e-commerce มีตาราง `order_items` ขนาดหลายร้อยล้านแถว และทีม analytics รัน query แบบนี้บ่อย:

```sql
-- Query แบบ analytical: aggregate ยอดขายตามสินค้าและช่วงเวลา
SELECT
    product_id,
    date_trunc('month', order_date) AS sales_month,
    SUM(quantity * unit_price) AS total_revenue,
    COUNT(*) AS order_count
FROM order_items
WHERE order_date >= '2026-01-01'
GROUP BY product_id, date_trunc('month', order_date)
ORDER BY total_revenue DESC
LIMIT 100;
```

บน row-oriented storage ปกติ query นี้ต้อง scan ทุกคอลัมน์ของทุกแถวที่ตรงเงื่อนไข `order_date` แม้จะใช้แค่ 4 คอลัมน์ในการคำนวณ แต่เมื่อตาราง `order_items` ถูกโหลดเข้า Columnar Engine, PostgreSQL query planner จะเลือกอ่านเฉพาะคอลัมน์ที่จำเป็นจาก in-memory columnar representation ทำให้ scan เร็วขึ้นอย่างมาก (ลด I/O และใช้ vectorized execution ในการ aggregate)

### ข้อควรพิจารณาเมื่อใช้ Columnar Engine

| ข้อพิจารณา | รายละเอียด |
|---|---|
| หน่วยความจำ | Columnar representation อยู่ใน RAM ต้องจัดสรร memory เพิ่มเติมนอกเหนือจาก shared_buffers ปกติ |
| ความสด (freshness) ของข้อมูล | AlloyDB sync ข้อมูลระหว่าง row storage กับ columnar representation โดยอัตโนมัติแบบ near real-time — ไม่ต้องทำ manual refresh เหมือน materialized view |
| ตารางที่เหมาะสม | ตารางขนาดใหญ่ที่มี analytical query บ่อย (fact table เช่น order_items, transaction logs) ไม่จำเป็นต้องใช้กับตารางเล็กหรือตารางที่มีแต่ point lookup |
| ค่าใช้จ่าย | การจัดสรร memory เพิ่มเติมมีผลต่อ instance sizing และค่าใช้จ่ายโดยตรง |

---

## Step 949: AlloyDB AI Integration เบื้องต้น

### ภาพรวม AlloyDB AI

Google วางตำแหน่ง AlloyDB ให้เป็นฐานข้อมูลที่รองรับ **generative AI workload** โดยตรง ภายใต้ชื่อรวมว่า **AlloyDB AI** ซึ่งครอบคลุมความสามารถหลายอย่าง แต่หัวใจสำคัญที่สุดที่ผู้เรียนควรรู้จักในระดับเบื้องต้นคือการรองรับ **pgvector extension** สำหรับงาน **vector search / similarity search** ซึ่งเป็นรากฐานของระบบ **RAG (Retrieval-Augmented Generation)** และ AI-powered recommendation ที่นิยมกันมากในปัจจุบัน

### pgvector คืออะไร (ทบทวนสั้น ๆ)

`pgvector` คือ extension ของ PostgreSQL ที่เพิ่ม data type `vector` และ operator สำหรับคำนวณระยะห่างระหว่าง vector (เช่น cosine distance, L2 distance, inner product) ใช้สำหรับเก็บ **embedding** ที่ได้จากโมเดล AI (เช่น text embedding, image embedding) แล้วค้นหา "ข้อมูลที่ใกล้เคียงกันทางความหมาย" (semantic similarity search)

```sql
-- เปิดใช้งาน pgvector บน AlloyDB
CREATE EXTENSION IF NOT EXISTS vector;

-- ตัวอย่างตารางเก็บ embedding ของคำอธิบายสินค้า (สำหรับ semantic product search)
CREATE TABLE product_embeddings (
    product_id   BIGINT PRIMARY KEY REFERENCES products(id),
    description  TEXT,
    embedding    VECTOR(768)  -- ขนาด dimension ขึ้นกับโมเดลที่ใช้สร้าง embedding
);

-- ค้นหาสินค้าที่มีคำอธิบายใกล้เคียงกับ query embedding มากที่สุด
SELECT product_id, description
FROM product_embeddings
ORDER BY embedding <=> '[0.012, -0.045, 0.089, ...]'::vector
LIMIT 10;
```

### จุดเด่นเฉพาะของ AlloyDB สำหรับ Vector Search

AlloyDB ไม่ได้แค่รองรับ pgvector เหมือน PostgreSQL ทั่วไป (ซึ่ง Cloud SQL ก็รองรับ pgvector เช่นกัน) แต่ AlloyDB มี **ScaNN index** (Scalable Nearest Neighbors — อัลกอริทึม vector search ประสิทธิภาพสูงที่ Google พัฒนาและใช้งานจริงในผลิตภัณฑ์ของตนเอง เช่น Google Search) เป็นทางเลือกเพิ่มเติมจาก index มาตรฐานของ pgvector (เช่น IVFFlat, HNSW) ซึ่งให้ประสิทธิภาพและความแม่นยำที่ดีกว่าในหลายกรณีเมื่อข้อมูลมีขนาดใหญ่มาก

```sql
-- ตัวอย่างการสร้าง ScaNN index (แนวคิดเชิงสถาปัตยกรรม
-- syntax ที่แน่นอนควรตรวจสอบจากเอกสาร AlloyDB ล่าสุด)
CREATE INDEX product_embedding_scann_idx
    ON product_embeddings
    USING scann (embedding cosine);
```

### ตัวอย่าง Use Case สำหรับ e-commerce

| Use case | การใช้ pgvector/AlloyDB AI |
|---|---|
| Semantic product search ("หารองเท้าวิ่งใส่สบาย ระบายอากาศดี") | เก็บ embedding ของคำอธิบายสินค้า ค้นหาด้วย similarity แทน keyword matching แบบเดิม |
| Product recommendation ("สินค้าที่คล้ายกับที่คุณเพิ่งดู") | คำนวณ nearest-neighbor ของ embedding สินค้าที่ผู้ใช้เพิ่งเปิดดู |
| Customer support chatbot (RAG) | เก็บ embedding ของเอกสาร FAQ/policy แล้วค้นหาเนื้อหาที่เกี่ยวข้องมาป้อนให้ LLM ตอบคำถาม |
| Fraud/anomaly detection เบื้องต้น | เปรียบเทียบ embedding ของ pattern การสั่งซื้อกับ pattern ปกติ |

> **ขอบเขตของบทนี้**: หัวข้อ vector search, embedding model, และการออกแบบระบบ RAG แบบละเอียด จะถูกกล่าวถึงเชิงลึกในบทที่ว่าด้วย PostgreSQL สำหรับ AI/ML workload ของหลักสูตรนี้โดยเฉพาะ ในที่นี้ขอเพียงให้ผู้เรียนเข้าใจว่า **AlloyDB มีข้อได้เปรียบเชิงสถาปัตยกรรมสำหรับงาน vector search ระดับ production** เมื่อเทียบกับ Cloud SQL ที่รองรับ pgvector ได้เช่นกันแต่ไม่มี ScaNN index และไม่มี Columnar Engine มาช่วยเสริมงาน analytical ที่มักมาคู่กับระบบ AI

---

## Step 950: แบบฝึกหัดรวม — เลือกระหว่าง Cloud SQL กับ AlloyDB สำหรับ e-commerce

### สถานการณ์

บริษัท e-commerce แห่งหนึ่งกำลังออกแบบสถาปัตยกรรมฐานข้อมูลใหม่บน GCP ระบบประกอบด้วยส่วนต่าง ๆ ดังนี้:

1. **Order Service** — รับคำสั่งซื้อ, ชำระเงิน, จัดการ inventory แบบ real-time (transactional หนัก, ต้องการ consistency สูง)
2. **Product Catalog Service** — แสดงรายการสินค้า, ค้นหาสินค้า (read-heavy, ต้องการ latency ต่ำ)
3. **Analytics/Reporting Dashboard** — รายงานยอดขายแบบ real-time สำหรับทีมผู้บริหาร (aggregate query หนักบนข้อมูลขนาดใหญ่)
4. **AI-powered Recommendation Engine** — แนะนำสินค้าด้วย semantic similarity จาก embedding
5. **Internal Admin Tool** — เครื่องมือ back-office ใช้งานภายใน traffic ต่ำมาก

### ตารางการตัดสินใจที่แนะนำ

| ส่วนประกอบ | ลักษณะ workload | บริการที่แนะนำ | เหตุผล |
|---|---|---|---|
| Order Service | OLTP หนัก, consistency สูง, write-heavy | **AlloyDB** (Primary + HA ในตัว) | ต้องการ write throughput สูงและ HA ระดับ enterprise, disaggregated storage ช่วยเรื่อง I/O amplification ตอน peak (เช่น sale event) |
| Product Catalog Service | Read-heavy, latency ต่ำ | **AlloyDB Read Pool** หรือ **Cloud SQL Read Replica** ขึ้นกับงบประมาณ | ถ้างบเพียงพอ AlloyDB Read Pool ให้ replica lag ต่ำกว่ามากและ auto load-balance ในตัว หากงบจำกัด Cloud SQL Read Replica ก็เพียงพอสำหรับ catalog ที่ไม่ต้อง real-time เป๊ะ |
| Analytics/Reporting Dashboard | Analytical aggregate หนัก | **AlloyDB + Columnar Engine** | ตรงกับจุดแข็งที่สุดของ AlloyDB คือ HTAP ทำรายงาน real-time ได้โดยไม่ต้อง ETL ออกไป data warehouse แยก |
| AI Recommendation Engine | Vector similarity search | **AlloyDB + pgvector + ScaNN index** | AlloyDB AI ออกแบบมาเพื่องานนี้โดยเฉพาะ ประสิทธิภาพดีกว่า pgvector บน Cloud SQL ทั่วไป |
| Internal Admin Tool | Traffic ต่ำมาก, ไม่ critical | **Cloud SQL (ZONAL, shared-core หรือ custom เล็ก)** | ไม่คุ้มค่าที่จะใช้ AlloyDB ซึ่งมีค่าใช้จ่ายต่อ vCPU สูงกว่า สำหรับงานที่ไม่ critical และ traffic ต่ำ |

### หลักการตัดสินใจโดยสรุป (Rule of Thumb)

```
                      เริ่มต้น: เลือกบริการ PostgreSQL บน GCP
                                    │
                    ┌───────────────┴───────────────┐
                    ▼                                 ▼
        Traffic ต่ำ / งบจำกัด /              Mission-critical /
        dev-test / internal tool             ต้องการประสิทธิภาพสูงสุด /
                    │                         มี analytical + transactional
                    ▼                         ร่วมกัน (HTAP) / ใช้ vector search
              Cloud SQL                                  │
                    │                                     ▼
                    │                              AlloyDB for PostgreSQL
                    │
        ┌───────────┴───────────┐
        ▼                         ▼
  ต้องการ HA               ไม่ต้องการ HA
  (production)             (dev/test)
        │                         │
        ▼                         ▼
  REGIONAL instance        ZONAL instance
  + Read Replica            (ไม่มี replica)
  (ถ้าต้อง read scaling)
```

### คำถามชวนคิดก่อนไปแบบฝึกหัดจริง

1. ถ้างบประมาณจำกัดมากในช่วงเริ่มต้นธุรกิจ (startup MVP) ควรเริ่มด้วย Cloud SQL หรือ AlloyDB ทั้งระบบ? เพราะเหตุใด?
2. หากทีม Analytics ต้องการรายงานที่ "สดที่สุดเท่าที่จะเป็นไปได้" (near real-time) โดยไม่ยอมให้มี ETL lag เลย สถาปัตยกรรมแบบใดตอบโจทย์ที่สุดระหว่าง (ก) Cloud SQL + export ไป BigQuery ทุกชั่วโมง (ข) AlloyDB + Columnar Engine?
3. เมื่อระบบเติบโตจน Order Service ต้องการ HA และ analytics ต้องการ HTAP พร้อมกัน จะสามารถรวมทั้งสองไว้ใน AlloyDB cluster เดียวกันได้หรือไม่ ข้อดีข้อเสียคืออะไร?

*(คำถามเหล่านี้ไม่มีคำตอบตายตัว ให้ผู้เรียนนำไปอภิปรายในบริบทของ trade-off ต้นทุนกับประสิทธิภาพ ซึ่งเป็นทักษะหลักของสถาปนิกระบบระดับ world-class)*

---

## สรุปท้ายบท

### ตารางเปรียบเทียบรวม: Cloud SQL vs AlloyDB vs AWS RDS/Aurora

| มิติ | GCP Cloud SQL | GCP AlloyDB | AWS RDS PostgreSQL | AWS Aurora PostgreSQL |
|---|---|---|---|---|
| ระดับบริการ | Managed database มาตรฐาน | Managed database ประสิทธิภาพสูง (next-gen) | Managed database มาตรฐาน | Managed database ประสิทธิภาพสูง (next-gen) |
| สถาปัตยกรรม storage | ผูกกับ compute (Persistent Disk) | Disaggregated storage | ผูกกับ compute (EBS) | Disaggregated storage (log-structured) |
| ความ compatible กับ PostgreSQL | 100% (PostgreSQL จริง) | 100% (wire-compatible) | 100% (PostgreSQL จริง) | 100% (wire-compatible) |
| HA | Regional instance (standby ข้าม zone, synchronous) | มีในตัวจากสถาปัตยกรรม disaggregated storage | Multi-AZ deployment | Multi-AZ ในตัวจากสถาปัตยกรรม distributed storage |
| Read scaling | Read Replica (endpoint แยก) | Read Pool (endpoint เดียว, auto load-balance) | Read Replica (endpoint แยก) | Reader endpoint (auto load-balance) |
| Replica lag | วินาที-นาที | ต่ำมาก (มักต่ำกว่าวินาที) | วินาที-นาที | ต่ำมาก (มักต่ำกว่าวินาที) |
| Analytical acceleration | ไม่มีในตัว | Columnar Engine (HTAP) | ไม่มีในตัว | ไม่มีในตัว (ต้องพึ่งบริการอื่น) |
| Vector search (AI) | รองรับ pgvector มาตรฐาน | pgvector + ScaNN index | รองรับ pgvector มาตรฐาน | รองรับ pgvector มาตรฐาน |
| การเชื่อมต่อปลอดภัยไม่เปิด public IP | Cloud SQL Auth Proxy | AlloyDB Auth Proxy (แนวคิดเดียวกัน) | RDS Proxy / IAM auth | RDS Proxy / IAM auth |
| Backup/PITR | Automated backup + WAL-based PITR | Automated backup + continuous backup (log-based) | Automated backup + transaction log PITR | Automated backup + continuous backup |
| ความเหมาะสมด้านต้นทุน | ต้นทุนต่ำกว่า เหมาะกับ workload ทั่วไป | ต้นทุนสูงกว่า แลกกับประสิทธิภาพและ HTAP | ต้นทุนต่ำกว่า เหมาะกับ workload ทั่วไป | ต้นทุนสูงกว่า แลกกับประสิทธิภาพ |
| กรณีใช้งานที่เหมาะสมที่สุด | Web app ทั่วไป, งบจำกัด, dev/test, internal tool | Mission-critical OLTP, HTAP, AI/vector workload | Web app ทั่วไปบน AWS | Mission-critical OLTP บน AWS |

### สิ่งที่ควรจำจากบทนี้

1. Cloud SQL คือคู่เทียบของ RDS — บริการ managed database มาตรฐานที่ compute และ storage ผูกกันในระดับ instance
2. AlloyDB คือคู่เทียบของ Aurora — สถาปัตยกรรม disaggregated storage ทำให้ write throughput สูงขึ้น, replica lag ต่ำลง, และเปิดทาง HTAP ผ่าน Columnar Engine
3. Cloud SQL Regional Instance ให้ HA แบบ synchronous replication ข้าม zone เหมือน RDS Multi-AZ แต่ standby ไม่รับ read traffic
4. Read Replica ใช้สำหรับ read scaling และ disaster recovery ข้าม region แต่เป็น asynchronous replication (มี lag)
5. Cloud SQL Auth Proxy คือเครื่องมือด้านความปลอดภัย ไม่ใช่ connection pooler — ควรใช้ร่วมกับ PgBouncer หากต้องการ pooling
6. AlloyDB Columnar Engine เปิดทางให้ทำ analytical query แบบ real-time บนฐานข้อมูล transactional เดียวกัน โดยไม่ต้อง ETL แยก
7. AlloyDB AI (pgvector + ScaNN) เหมาะสำหรับงาน vector search/semantic search ที่ต้องการประสิทธิภาพระดับ production
8. การเลือกระหว่าง Cloud SQL กับ AlloyDB ควรพิจารณาจาก workload characteristic (OLTP ล้วน vs HTAP), งบประมาณ, และ SLA ที่ต้องการ ไม่ใช่การเลือกแบบ "ตัวใหม่ต้องดีกว่าเสมอ"

---

## แบบฝึกหัดท้ายบท

### แบบฝึกหัดที่ 1

อธิบายความแตกต่างเชิงสถาปัตยกรรมพื้นฐานที่สุดระหว่าง Cloud SQL กับ AlloyDB ในหนึ่งประโยค

<details>
<summary>เฉลย</summary>

Cloud SQL ผูก compute กับ storage ไว้ในระดับ instance เดียว (เหมือน RDS) ในขณะที่ AlloyDB แยก compute ออกจาก storage อย่างสมบูรณ์ (disaggregated storage เหมือน Aurora) ทำให้ AlloyDB มี write throughput สูงกว่า, replica lag ต่ำกว่า, และ storage ขยายได้ยืดหยุ่นกว่าโดยไม่กระทบ instance ที่กำลังรันอยู่

</details>

### แบบฝึกหัดที่ 2

จงเขียนคำสั่ง `gcloud` เพื่อสร้าง Cloud SQL instance ชื่อ `shop-db` เวอร์ชัน PostgreSQL 16 ขนาด 4 vCPU 16 GB RAM ที่ region `asia-southeast1` พร้อมเปิดใช้งาน High Availability

<details>
<summary>เฉลย</summary>

```bash
gcloud sql instances create shop-db \
  --database-version=POSTGRES_16 \
  --tier=db-custom-4-16384 \
  --region=asia-southeast1 \
  --storage-type=SSD \
  --storage-size=100GB \
  --availability-type=REGIONAL
```

`--availability-type=REGIONAL` คือพารามิเตอร์ที่เปิดใช้งาน High Availability (standby ข้าม zone แบบ synchronous replication)

</details>

### แบบฝึกหัดที่ 3

เพราะเหตุใด standby instance ใน Cloud SQL Regional configuration จึงไม่สามารถรับ read traffic ได้ และหากต้องการ scale การอ่าน ควรใช้กลไกใดแทน

<details>
<summary>เฉลย</summary>

Standby instance ใน Regional (HA) configuration ถูกออกแบบมาเพื่อการ **failover เท่านั้น** ไม่ใช่เพื่อ read scaling — มันต้องพร้อม promote เป็น primary ได้ทันทีด้วยข้อมูลที่ sync กันแบบ synchronous เสมอ การเปิดให้รับ read traffic จะเพิ่มความซับซ้อนและความเสี่ยงต่อ RPO/RTO หากต้องการ scale การอ่าน ควรใช้ **Read Replica** ต่างหาก (asynchronous replication, endpoint แยก) แทน

</details>

### แบบฝึกหัดที่ 4

Cross-region Read Replica ใช้สำหรับวัตถุประสงค์ใดได้บ้าง (ระบุอย่างน้อย 2 ข้อ)

<details>
<summary>เฉลย</summary>

1. **Disaster Recovery** — เป็น DR site ที่สามารถ promote เป็น primary ได้หาก region หลักล่มทั้ง region
2. **ลด latency ให้ผู้ใช้ในภูมิภาคอื่น** — วาง replica ไว้ใกล้ผู้ใช้กลุ่มนั้นเพื่อ serve read query ด้วย latency ต่ำ

</details>

### แบบฝึกหัดที่ 5

การ promote read replica ให้กลายเป็น standalone instance มีผลกระทบอะไรบ้างที่ต้องระวัง

<details>
<summary>เฉลย</summary>

การ promote เป็นการดำเนินการที่ **ย้อนกลับไม่ได้ (irreversible)** — instance ที่ถูก promote จะตัดขาดจาก replication chain เดิมโดยสมบูรณ์และกลายเป็น standalone instance ที่รับ read/write เอง ต้องวางแผนเรื่อง connection string/DNS switching ของ application ให้ชี้มาที่ instance ใหม่ และต้องแน่ใจว่า primary เดิม (ถ้ายังอยู่) จะไม่เขียนข้อมูลทับซ้อนกับ instance ที่ถูก promote ไปแล้ว

</details>

### แบบฝึกหัดที่ 6

Cloud SQL Auth Proxy แก้ปัญหาด้านความปลอดภัยอะไร และมันทำหน้าที่เป็น connection pooler หรือไม่

<details>
<summary>เฉลย</summary>

Cloud SQL Auth Proxy แก้ปัญหาการเปิด public IP ให้ฐานข้อมูลโดยตรง โดยสร้าง encrypted tunnel (TLS) ระหว่าง client กับ Cloud SQL instance และยืนยันตัวตนด้วย IAM credentials แทนการเปิด public IP + whitelist IP address มันไม่ใช่ connection pooler — ไม่มีกลไก pooling ในตัว หากต้องการ connection pooling ต้องใช้ร่วมกับเครื่องมืออื่น เช่น PgBouncer

</details>

### แบบฝึกหัดที่ 7

อธิบายว่า AlloyDB Columnar Engine ทำงานอย่างไร และเหตุใดจึงช่วยให้ query แบบ aggregate เร็วขึ้น

<details>
<summary>เฉลย</summary>

Columnar Engine เก็บสำเนาข้อมูลบางคอลัมน์ไว้ใน**หน่วยความจำในรูปแบบ columnar representation** ควบคู่กับข้อมูล row-oriented บน disk โดย sync กันแบบอัตโนมัติ เมื่อ query planner พบว่า query เป็น analytical pattern (scan/aggregate จำนวนมาก) จะเลือกอ่านจาก columnar representation แทน ทำให้อ่านเฉพาะคอลัมน์ที่จำเป็น (แทนที่จะอ่านทั้งแถว) และใช้ vectorized execution ในการคำนวณ ส่งผลให้ query เร็วขึ้นอย่างมากเมื่อเทียบกับการ scan row storage แบบปกติ

</details>

### แบบฝึกหัดที่ 8

ระบบ recommendation engine ที่ต้องการค้นหาสินค้าด้วย semantic similarity ควรเลือกใช้บริการ GCP ใด และต้องเปิดใช้งาน extension/index ใดบ้าง

<details>
<summary>เฉลย</summary>

ควรเลือก **AlloyDB for PostgreSQL** เพราะรองรับ pgvector extension เช่นเดียวกับ Cloud SQL แต่มี **ScaNN index** เพิ่มเติมซึ่งให้ประสิทธิภาพ vector search ที่ดีกว่าในระดับ production โดยต้องเปิดใช้งาน `CREATE EXTENSION vector;` เพื่อใช้ data type `vector` และสร้าง index (เช่น ScaNN หรือ HNSW/IVFFlat ของ pgvector มาตรฐาน) บนคอลัมน์ embedding เพื่อเร่งความเร็วการค้นหา nearest neighbor

</details>

### แบบฝึกหัดที่ 9

ทีม Internal Admin Tool ของบริษัท e-commerce มี traffic ต่ำมากและงบประมาณจำกัด ควรเลือก Cloud SQL หรือ AlloyDB และควรตั้งค่า availability type แบบใด

<details>
<summary>เฉลย</summary>

ควรเลือก **Cloud SQL** เนื่องจาก AlloyDB มีค่าใช้จ่ายต่อ vCPU สูงกว่าและความสามารถพิเศษ (Columnar Engine, disaggregated storage) ไม่จำเป็นสำหรับงานที่ไม่ critical เช่นนี้ สำหรับ availability type สามารถใช้ **ZONAL** (ไม่ต้องมี HA) ได้ เนื่องจากเป็นเครื่องมือภายในที่ traffic ต่ำมากและ downtime สั้น ๆ ไม่กระทบต่อธุรกิจหลัก ซึ่งช่วยประหยัดค่าใช้จ่ายได้เพิ่มเติม (ไม่ต้องจ่ายสำหรับ standby instance)

</details>

### แบบฝึกหัดที่ 10

จงอธิบายว่าเหตุใด Order Service ที่เป็น OLTP หนักและ Analytics Dashboard ที่เป็น analytical หนัก จึงสามารถใช้ AlloyDB cluster เดียวกันได้ในเชิงแนวคิด และมีข้อควรระวังอะไรบ้างหากตัดสินใจทำเช่นนั้นจริง

<details>
<summary>เฉลย</summary>

ในเชิงแนวคิด AlloyDB ถูกออกแบบมาเพื่อรองรับ **HTAP (Hybrid Transactional/Analytical Processing)** โดย Columnar Engine ช่วยให้ analytical query ทำงานเร็วบนข้อมูลเดียวกับที่ Order Service เขียนแบบ transactional โดยไม่ต้อง ETL แยก ซึ่งในทางทฤษฎีสามารถใช้ cluster เดียวกันได้จริง อย่างไรก็ตาม ข้อควรระวังคือ:

1. **Resource contention** — หาก analytical query หนักเกินไป อาจแย่ง CPU/memory จาก transactional workload แม้ Columnar Engine จะช่วยลดผลกระทบได้มาก แต่ควรพิจารณาแยก Read Pool instance สำหรับ analytics โดยเฉพาะ เพื่อไม่ให้กระทบ primary ที่รับ write โดยตรง
2. **Blast radius** — หาก cluster เดียวมีปัญหา (เช่น bug, มนุษย์ทำผิดพลาดตอน maintenance) จะกระทบทั้ง transactional และ analytical workload พร้อมกัน
3. **การจัดการสิทธิ์และ workload isolation** — ทีม analytics และทีม engineering อาจต้องการ SLA/permission ที่ต่างกัน การแยก instance (แม้อยู่ cluster เดียวกัน) ช่วยให้ควบคุมสิทธิ์และ monitor แยกจากกันได้ชัดเจนกว่า

แนวทางที่สมดุลคือใช้ cluster เดียวกันเพื่อประโยชน์จาก HTAP แต่แยก **Read Pool instance ต่างหากสำหรับ analytics** เพื่อ isolate resource และความเสี่ยงจาก primary instance ที่รับ transactional traffic

</details>

---

## บทถัดไป

บทถัดไปจะพาไปสำรวจ PostgreSQL บนอีกหนึ่ง cloud provider หลักของโลก คือ Microsoft Azure — Azure Database for PostgreSQL Flexible Server และแนวคิดที่เกี่ยวข้อง

**อ่านต่อ**: [Part 096 — PostgreSQL บน Azure](./part-096-azure-database.md)
