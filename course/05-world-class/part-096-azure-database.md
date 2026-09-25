# Part 096: PostgreSQL บน Cloud — Azure Database for PostgreSQL

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับโลก/ผู้เชี่ยวชาญ | Part 096

---

## เกริ่นนำ

หลังจากที่เราเรียนรู้ PostgreSQL บน AWS (RDS/Aurora — Part 094) และบน Google Cloud (Cloud SQL/AlloyDB — Part 095) ไปแล้ว บทนี้จะพาไปทำความรู้จักกับผู้เล่นรายที่สามในตลาด Cloud รายใหญ่ นั่นคือ **Microsoft Azure** ผ่านบริการ **Azure Database for PostgreSQL** โดยเฉพาะรุ่นปัจจุบันที่ใช้งานจริงคือ **Flexible Server**

บทนี้มีจำนวน Step น้อยกว่าบทอื่น ๆ ในหลักสูตร (มีเพียง 5 Step คือ Step 951–955) เพราะเนื้อหาเรื่อง PostgreSQL บน Cloud ผู้ให้บริการต่าง ๆ มีโครงสร้างแนวคิดที่คล้ายกันมาก (compute tier, HA, backup, networking) ซึ่งเราได้ปูพื้นฐานแนวคิดเหล่านี้ไปแล้วใน Part 094-095 ดังนั้นบทนี้จะเน้น **สิ่งที่ Azure ทำต่างออกไป** และจบด้วยแบบฝึกหัดเปรียบเทียบทั้งสาม Cloud อย่างครบวงจร

> **หมายเหตุสำคัญเกี่ยวกับสภาพแวดล้อมการเรียน**
>
> สภาพแวดล้อมที่ใช้เรียนหลักสูตรนี้ **ไม่มีสิทธิ์เข้าถึง Azure Subscription จริง** ดังนั้นคำสั่ง `az` CLI ทั้งหมดในบทนี้เป็น **แนวทางเชิงสถาปัตยกรรมและแนวคิด (architectural/conceptual guidance)** ที่เขียนให้ถูกต้องตรงตาม syntax และพารามิเตอร์จริงของ Azure CLI (ตรวจสอบกับเอกสารทางการของ Microsoft ก่อนนำไปใช้งานจริงเสมอ เพราะ Azure มีการปรับปรุงพารามิเตอร์และ default value อยู่เรื่อย ๆ) เป้าหมายของบทนี้คือให้ผู้เรียน **เข้าใจโครงสร้าง การตัดสินใจเชิงสถาปัตยกรรม และคำศัพท์เฉพาะของ Azure** เพื่อสามารถอ่าน design document, ประเมินข้อเสนอโครงการ หรือสื่อสารกับทีม Cloud/DevOps ที่ใช้ Azure ได้อย่างมั่นใจ แม้จะไม่ได้ลงมือรันคำสั่งจริงในบทเรียนนี้ก็ตาม

---

## เป้าหมายการเรียนรู้

เมื่อเรียนจบบทนี้ ผู้เรียนจะสามารถ:

1. อธิบายความแตกต่างระหว่าง **Azure Database for PostgreSQL Flexible Server** กับ **Single Server** (รุ่นเก่าที่เลิกใช้แล้ว) และเทียบเคียงกับ AWS RDS/Aurora และ GCP Cloud SQL/AlloyDB ได้
2. อธิบายสถาปัตยกรรมของ Flexible Server รวมถึง Zone-redundant High Availability และตัวเลือก Compute Tier (Burstable, General Purpose, Memory Optimized)
3. ออกแบบกลยุทธ์ Backup แบบ Geo-redundant และตั้งค่า Point-in-Time Recovery (PITR) แบบ managed บน Azure
4. ออกแบบ Network Topology ที่ปลอดภัยด้วย VNet Integration และ Private Endpoint
5. เปรียบเทียบและเลือกผู้ให้บริการ Cloud Database ที่เหมาะสม (AWS/GCP/Azure) ตามสถานการณ์ทางธุรกิจของระบบ e-commerce จริง

---

## Step 951: Azure Database for PostgreSQL คืออะไร

### 951.1 ภาพรวมบริการ

**Azure Database for PostgreSQL** คือบริการ Database-as-a-Service (DBaaS) ของ Microsoft Azure ที่รัน PostgreSQL แบบ fully-managed คล้ายกับ RDS ของ AWS และ Cloud SQL ของ GCP โดยมีให้เลือก 2 deployment option หลักในประวัติศาสตร์ของบริการนี้:

| รุ่น | สถานะ | หมายเหตุ |
|---|---|---|
| **Single Server** | **Deprecated** — Microsoft ประกาศเลิกใช้งาน (retirement) และหยุดรองรับแล้ว | สถาปัตยกรรมเก่า ใช้ gateway-based connection, จำกัด extension, ไม่มี Availability Zone control |
| **Flexible Server** | **ใช้งานจริง (Generally Available)** — เป็นทางเลือกมาตรฐานสำหรับ deployment ใหม่ทั้งหมด | ควบคุมได้ละเอียดกว่า รองรับ zone placement, custom maintenance window, burstable compute, HA แบบ zone-redundant |

> เนื่องจาก Single Server ถูก retire ไปแล้วในทางปฏิบัติ บทนี้จะโฟกัสที่ **Flexible Server** เท่านั้น และเมื่อพูดถึง "Azure Database for PostgreSQL" ต่อจากนี้ในบทนี้ จะหมายถึง Flexible Server โดยปริยาย

### 951.2 ทำไมต้องมี "Flexible" Server

ชื่อ "Flexible" สะท้อนถึงจุดขายหลักเมื่อเทียบกับ Single Server รุ่นเก่า:

- **ควบคุม Availability Zone ได้โดยตรง** — เลือกได้ว่าจะ deploy ใน AZ ไหน และเลือกได้ว่า Standby จะอยู่ AZ เดียวกันหรือคนละ AZ
- **Stop/Start ได้** — สามารถหยุด instance ชั่วคราว (สูงสุด 7 วันต่อครั้งตามข้อจำกัดของบริการ) เพื่อประหยัดค่า compute ในสภาพแวดล้อม dev/test ซึ่ง Single Server ทำไม่ได้
- **ควบคุม Maintenance Window ได้ละเอียดกว่า** — กำหนดวันและช่วงเวลาที่ต้องการให้ patch ได้
- **รองรับ Burstable compute tier** — เหมาะกับ workload ที่ใช้ CPU ไม่สม่ำเสมอ (คล้าย AWS RDS T-series instance)
- **เชื่อมต่อกับ VNet ได้โดยตรงแบบ native (VNet Integration)** — ไม่ต้องผ่าน public gateway เหมือน Single Server
- **รองรับ PostgreSQL extension ที่หลากหลายกว่า** เช่น `pg_partman`, `pg_cron`, `pg_stat_statements`, `postgis`, `pgvector` (สำหรับงาน AI/vector search) เป็นต้น

### 951.3 การสร้าง Flexible Server ด้วย Azure CLI

```bash
# ตรวจสอบ version ของ PostgreSQL ที่รองรับในปัจจุบัน
az postgres flexible-server list-skus \
  --location southeastasia \
  --output table

# สร้าง Resource Group (ถ้ายังไม่มี)
az group create \
  --name rg-ecommerce-prod \
  --location southeastasia

# สร้าง Flexible Server
az postgres flexible-server create \
  --resource-group rg-ecommerce-prod \
  --name pg-ecommerce-prod-th \
  --location southeastasia \
  --admin-user pgadmin_ecom \
  --admin-password 'Str0ngP@ssw0rd!ChangeMe' \
  --sku-name Standard_D4ds_v5 \
  --tier GeneralPurpose \
  --storage-size 256 \
  --storage-type PremiumV2_LRS \
  --version 16 \
  --high-availability ZoneRedundant \
  --zone 1 \
  --standby-zone 2 \
  --backup-retention 14 \
  --geo-redundant-backup Enabled \
  --public-access None \
  --vnet vnet-ecommerce-prod \
  --subnet snet-db-tier \
  --tags environment=production team=platform

# ตรวจสอบสถานะหลังสร้าง
az postgres flexible-server show \
  --resource-group rg-ecommerce-prod \
  --name pg-ecommerce-prod-th \
  --output table
```

**สังเกตพารามิเตอร์สำคัญ:**

- `--tier` รับค่า `Burstable`, `GeneralPurpose`, หรือ `MemoryOptimized` (รายละเอียดใน Step 952)
- `--sku-name` เป็นชื่อ VM series ของ Azure (เช่น `Standard_B2s` สำหรับ Burstable, `Standard_D4ds_v5` สำหรับ General Purpose, `Standard_E4ds_v5` สำหรับ Memory Optimized)
- `--high-availability` รับค่า `ZoneRedundant`, `SameZone`, หรือ `Disabled`
- `--public-access None` หมายถึงปิด public endpoint ทั้งหมด บังคับให้เข้าถึงผ่าน VNet เท่านั้น (best practice สำหรับ production)

### 951.4 เทียบเคียงกับสิ่งที่เรียนมาแล้ว (AWS RDS / GCP Cloud SQL)

| แนวคิด | AWS RDS/Aurora (Part 094) | GCP Cloud SQL/AlloyDB (Part 095) | Azure Flexible Server |
|---|---|---|---|
| หน่วยคำนวณ | DB Instance Class (เช่น `db.r6g.xlarge`) | Machine Type (เช่น `db-custom-4-16384`) | Compute Tier + SKU (เช่น `GeneralPurpose` / `Standard_D4ds_v5`) |
| ตัวเลือก Deployment รุ่นเก่า | RDS Classic (ยังใช้งานได้) | Cloud SQL 1st gen (retired) | Single Server (retired) |
| CLI หลัก | `aws rds` | `gcloud sql` | `az postgres flexible-server` |
| High-end variant | Aurora PostgreSQL (storage-compute แยกกัน) | AlloyDB (columnar engine + AI) | ยังไม่มี variant แยกต่างหาก (ใช้ Flexible Server เดียว ปรับ tier แทน) |
| การจ่ายเงินแบบ Serverless | Aurora Serverless v2 | Cloud SQL ไม่มี (AlloyDB มี auto-scaling storage) | Azure Database for PostgreSQL **ไม่มี Serverless tier แยก** ในปัจจุบัน (ต้องเลือก sku คงที่ หรือใช้ Burstable ที่ยืดหยุ่นระดับหนึ่ง) |

> **ข้อสังเกตเชิงกลยุทธ์:** จุดที่ Azure "ขาด" เมื่อเทียบกับคู่แข่งคือยังไม่มี serverless/high-performance variant เทียบเท่า Aurora หรือ AlloyDB โดยตรง — Azure เลือกใช้แนวทาง "single product line, multiple tiers" แทนที่จะแยกผลิตภัณฑ์ premium ออกมาต่างหาก ซึ่งเป็นจุดสำคัญที่ต้องพิจารณาเมื่อ workload ต้องการ auto-scaling แบบ Aurora Serverless หรือ analytical query แบบ AlloyDB

---

## Step 952: Flexible Server Architecture

### 952.1 สถาปัตยกรรม Zone-Redundant High Availability

Azure Flexible Server รองรับ HA 3 โหมด:

```
โหมดที่ 1: Disabled (ไม่มี HA)
┌─────────────────────────┐
│   Availability Zone 1   │
│  ┌────────────────────┐ │
│  │  Primary Instance   │ │
│  └────────────────────┘ │
└─────────────────────────┘
  * ไม่มี automatic failover
  * เหมาะกับ dev/test เท่านั้น

โหมดที่ 2: SameZone HA
┌─────────────────────────────────────────┐
│           Availability Zone 1            │
│  ┌────────────────────┐ ┌──────────────┐ │
│  │  Primary Instance   │→│   Standby    │ │
│  │  (Synchronous WAL)  │←│   Instance   │ │
│  └────────────────────┘ └──────────────┘ │
└─────────────────────────────────────────┘
  * ป้องกันได้เฉพาะ instance failure
  * ไม่ป้องกัน Zone-wide outage
  * Latency ระหว่าง Primary-Standby ต่ำมาก (อยู่ AZ เดียวกัน)

โหมดที่ 3: ZoneRedundant HA (แนะนำสำหรับ Production)
┌─────────────────────┐         ┌─────────────────────┐
│  Availability Zone 1 │         │  Availability Zone 2 │
│ ┌──────────────────┐ │  Sync   │ ┌──────────────────┐ │
│ │ Primary Instance  │─┼─WAL────┼→│ Standby Instance │ │
│ │                   │←┼─────────┼─│                   │ │
│ └──────────────────┘ │         │ └──────────────────┘ │
└─────────────────────┘         └─────────────────────┘
  * ป้องกันได้ทั้ง instance failure และ Zone-wide outage
  * Failover อัตโนมัติ (โดยทั่วไปภายใน 60-120 วินาที)
  * ใช้ synchronous replication → RPO = 0 (ไม่มี data loss)
```

**คำสั่งเปิดใช้งาน/ปรับเปลี่ยน HA:**

```bash
# เปิด Zone-redundant HA บน server ที่มีอยู่แล้ว
az postgres flexible-server update \
  --resource-group rg-ecommerce-prod \
  --name pg-ecommerce-prod-th \
  --high-availability ZoneRedundant \
  --standby-zone 2

# ตรวจสอบสถานะ HA
az postgres flexible-server show \
  --resource-group rg-ecommerce-prod \
  --name pg-ecommerce-prod-th \
  --query "{HAState:highAvailability.state, HAMode:highAvailability.mode, StandbyZone:highAvailability.standbyAvailabilityZone}" \
  --output table
```

เปรียบเทียบกับสิ่งที่เคยเรียน: แนวคิด ZoneRedundant HA ของ Azure เทียบเท่ากับ **Multi-AZ deployment** ของ AWS RDS และ **Regional (HA) configuration** ของ GCP Cloud SQL — ทั้งสามใช้หลักการเดียวกันคือ synchronous replication ข้าม Availability Zone พร้อม automatic failover

### 952.2 Compute Tier ทั้งสามแบบ

| Compute Tier | เหมาะกับ | ลักษณะ vCPU | ตัวอย่าง SKU | เทียบเคียง AWS/GCP |
|---|---|---|---|---|
| **Burstable (B-series)** | Dev/Test, workload ที่ใช้ CPU ไม่สม่ำเสมอ, ระบบขนาดเล็ก | ใช้ CPU credit สะสม บาง SKU มี baseline ต่ำแต่ burst ได้เมื่อจำเป็น | `Standard_B1ms`, `Standard_B2s`, `Standard_B2ms` | เทียบเท่า AWS `db.t3`/`db.t4g` และ GCP `db-custom` แบบ shared-core |
| **General Purpose (D-series)** | Production workload ทั่วไป, balance ระหว่าง compute/memory | CPU:Memory ratio 1:4 (โดยประมาณ) | `Standard_D2ds_v5` ถึง `Standard_D64ds_v5` | เทียบเท่า AWS `db.m6g`/`db.r6g` (general purpose) และ GCP `db-custom` มาตรฐาน |
| **Memory Optimized (E-series)** | Workload ที่ต้องการ RAM สูง เช่น cache-heavy query, large working set, analytical query | CPU:Memory ratio 1:8 | `Standard_E2ds_v5` ถึง `Standard_E96ds_v5` | เทียบเท่า AWS `db.r6g`/`db.x2g` และ GCP high-memory machine type |

```bash
# เปลี่ยน compute tier จาก GeneralPurpose เป็น MemoryOptimized
az postgres flexible-server update \
  --resource-group rg-ecommerce-prod \
  --name pg-ecommerce-prod-th \
  --tier MemoryOptimized \
  --sku-name Standard_E8ds_v5

# ปรับ storage IOPS แบบอิสระ (สำหรับ storage type ที่รองรับ provisioned IOPS)
az postgres flexible-server update \
  --resource-group rg-ecommerce-prod \
  --name pg-ecommerce-prod-th \
  --performance-tier P30 \
  --iops 7500 \
  --throughput 250
```

> **หมายเหตุ:** การเปลี่ยน compute tier หรือ SKU มักทำให้เกิด **restart** ของ server (downtime ช่วงสั้น ๆ) ควรวางแผนทำใน Maintenance Window หรือใช้ HA standby ช่วยลด downtime ในบาง scenario

### 952.3 Storage Auto-Grow

Flexible Server รองรับ **Storage Auto-grow** ซึ่งช่วยขยายพื้นที่ storage อัตโนมัติเมื่อใช้งานใกล้เต็ม เพื่อป้องกัน downtime จาก disk full:

```bash
az postgres flexible-server update \
  --resource-group rg-ecommerce-prod \
  --name pg-ecommerce-prod-th \
  --storage-auto-grow Enabled
```

แนวคิดนี้เทียบเท่ากับ **Storage Autoscaling** ของ AWS RDS และ **Automatic storage increase** ของ GCP Cloud SQL — ทั้งสาม Cloud มีกลไกป้องกันปัญหา "database หยุดทำงานเพราะ disk เต็ม" คล้ายกัน แต่รายละเอียด threshold และ increment size ต่างกันไปตามผู้ให้บริการ

### 952.4 Connection Pooling บน Azure Flexible Server

เช่นเดียวกับ AWS RDS/Aurora (RDS Proxy) และ GCP Cloud SQL (ผ่าน PgBouncer แบบ sidecar หรือ AlloyDB's built-in connection pooling) Azure Flexible Server ก็มีปัญหาเดียวกันเรื่อง **จำนวน connection สูงสุดที่จำกัดตาม RAM ของ SKU** — ยิ่ง SKU เล็ก `max_connections` ก็ยิ่งน้อย ซึ่งเป็นปัญหาคลาสสิกของแอปพลิเคชันที่เปิด connection จำนวนมาก (เช่น serverless function ที่ scale out เร็ว หรือ microservice หลายตัว)

Azure แก้ปัญหานี้ด้วย **Built-in PgBouncer** ที่สามารถเปิดใช้งานได้ในตัว Flexible Server เอง (รันอยู่ข้าง ๆ PostgreSQL engine บน compute เดียวกัน) โดยไม่ต้องติดตั้ง connection pooler แยกเป็น service ต่างหาก:

```bash
# เปิดใช้งาน built-in PgBouncer
az postgres flexible-server parameter set \
  --resource-group rg-ecommerce-prod \
  --server-name pg-ecommerce-prod-th \
  --name pgbouncer.enabled \
  --value true

# ปรับค่า pool mode (transaction / session / statement)
az postgres flexible-server parameter set \
  --resource-group rg-ecommerce-prod \
  --server-name pg-ecommerce-prod-th \
  --name pgbouncer.pool_mode \
  --value transaction

# เชื่อมต่อผ่าน PgBouncer โดยใช้ port 6432 แทน 5432 (โดยทั่วไป)
psql "host=pg-ecommerce-prod-th.postgres.database.azure.com port=6432 dbname=ecommerce user=pgadmin_ecom sslmode=require"
```

> **ข้อควรระวัง:** เนื่องจาก built-in PgBouncer แชร์ compute resource เดียวกับตัว PostgreSQL engine เอง หาก workload ที่ pooling ต้องรองรับหนักมาก อาจพิจารณาแยก connection pooler ออกไปรันเป็น service ต่างหาก (เช่นบน Azure Container Apps หรือ AKS) เพื่อไม่ให้แย่ง CPU/Memory กับตัว database engine เอง — หลักการเดียวกันกับที่ควรพิจารณาเมื่อใช้ RDS Proxy หรือ PgBouncer sidecar บน GCP

### 952.5 Azure Monitor และ Query Performance Insight

Flexible Server เชื่อมต่อกับ **Azure Monitor** โดยอัตโนมัติ ทำให้ดู metric สำคัญ เช่น CPU percent, Memory percent, Storage percent, Active Connections, IOPS ได้ผ่าน Azure Portal หรือดึงผ่าน CLI:

```bash
# ดึง metric CPU utilization ย้อนหลัง 1 ชั่วโมง
az monitor metrics list \
  --resource "/subscriptions/<sub-id>/resourceGroups/rg-ecommerce-prod/providers/Microsoft.DBforPostgreSQL/flexibleServers/pg-ecommerce-prod-th" \
  --metric "cpu_percent" \
  --interval PT5M \
  --output table

# เปิดใช้ Query Performance Insight (วิเคราะห์ query ที่ช้าที่สุด — ต้องเปิด pg_stat_statements ก่อน)
az postgres flexible-server parameter set \
  --resource-group rg-ecommerce-prod \
  --server-name pg-ecommerce-prod-th \
  --name shared_preload_libraries \
  --value pg_stat_statements
```

Query Performance Insight ของ Azure ทำหน้าที่คล้ายกับ **Performance Insights** ของ AWS RDS และ **Query Insights** ของ GCP Cloud SQL — ทั้งสามใช้หลักการเดียวกันคือต่อยอดจาก `pg_stat_statements` แล้วแสดงผลผ่าน dashboard ที่ใช้งานง่ายกว่าการ query ตารางระบบเอง

---

## Step 953: Azure Backup และ Geo-redundant Backup

### 953.1 กลไก Backup พื้นฐานของ Flexible Server

Azure Database for PostgreSQL Flexible Server ทำ **Automated Backup** แบบ managed โดยอัตโนมัติ ประกอบด้วย:

1. **Full snapshot backup** — ทำเป็นระยะตาม schedule ภายใน (โดยทั่วไปวันละครั้ง)
2. **Differential backup** — เก็บการเปลี่ยนแปลงระหว่าง full backup
3. **Transaction log (WAL) backup** — เก็บต่อเนื่องเพื่อรองรับ Point-in-Time Recovery (PITR)

```
Timeline การทำ Backup ของ Flexible Server
────────────────────────────────────────────────────────►  เวลา
   Day 1        Day 2        Day 3        Day 4       ...
    │Full         │Diff         │Diff         │Full
    │Backup       │Backup       │Backup       │Backup
    ▼             ▼             ▼             ▼
════╪═════════════╪═════════════╪═════════════╪════════
    WAL ──────────────────────────────────────────────►
    (ต่อเนื่องตลอดเวลา ใช้สำหรับ PITR แบบ second-level granularity)
```

### 953.2 การตั้งค่า Backup Retention Period

```bash
# ตั้งค่า backup retention period (รองรับ 1-35 วัน)
az postgres flexible-server update \
  --resource-group rg-ecommerce-prod \
  --name pg-ecommerce-prod-th \
  --backup-retention 35

# ตรวจสอบค่า backup retention ปัจจุบัน
az postgres flexible-server show \
  --resource-group rg-ecommerce-prod \
  --name pg-ecommerce-prod-th \
  --query "backup" \
  --output jsonc
```

**ข้อจำกัดสำคัญ:** Retention period ของ Flexible Server รองรับสูงสุด **35 วัน** (แตกต่างจาก AWS RDS ที่รองรับสูงสุด 35 วันเช่นกันสำหรับ automated backup แต่ Aurora รองรับได้ถึง 35 วันเช่นกัน โดยทั้งสามผู้ให้บริการมักมีเพดานใกล้เคียงกันสำหรับ automated backup — หากต้องการเก็บย้อนหลังนานกว่านั้นต้องใช้ long-term retention หรือ manual snapshot/export แยกต่างหาก)

### 953.3 Geo-redundant Backup

จุดเด่นสำคัญของ Azure Flexible Server ที่ควรเข้าใจให้ชัดคือ **Geo-redundant Backup** — ฟีเจอร์นี้จะคัดลอก backup ไปเก็บไว้ใน **paired region** อีกแห่งหนึ่งโดยอัตโนมัติ ทำให้สามารถกู้คืนฐานข้อมูลไปยัง region อื่นได้ในกรณีที่ region หลักเกิดภัยพิบัติระดับภูมิภาค (regional disaster)

```
┌───────────────────────────────┐        ┌───────────────────────────────┐
│   Primary Region               │        │   Paired Region (Geo-backup)   │
│   Southeast Asia (Singapore)   │        │   East Asia (Hong Kong)        │
│                                 │        │                                 │
│  ┌───────────────────────────┐ │  Async │  ┌───────────────────────────┐  │
│  │  Flexible Server (Live)   │ │ Backup │  │   Backup Storage (RA-GRS) │  │
│  │  Primary + HA Standby      │─┼───Copy─┼─→│   (Read-Access enabled)   │  │
│  └───────────────────────────┘ │        │  └───────────────────────────┘  │
└───────────────────────────────┘        └───────────────────────────────┘
        ปกติใช้งานที่นี่                         Restore มาที่นี่เมื่อเกิด
                                                  Regional Disaster
```

**สำคัญมาก:** Geo-redundant Backup ต้อง **เปิดใช้ตอนสร้าง server เท่านั้น** — Azure **ไม่อนุญาตให้เปลี่ยนจาก Locally-redundant เป็น Geo-redundant ภายหลัง** (ต้องสร้าง server ใหม่หากลืมเปิดตั้งแต่แรก) นี่คือข้อควรระวังเชิงสถาปัตยกรรมที่สำคัญที่สุดข้อหนึ่งในบทนี้

```bash
# ต้องกำหนดตอนสร้าง server เท่านั้น — แก้ไขภายหลังไม่ได้
az postgres flexible-server create \
  --resource-group rg-ecommerce-prod \
  --name pg-ecommerce-prod-th \
  --geo-redundant-backup Enabled \
  --location southeastasia \
  # ... พารามิเตอร์อื่น ๆ ตามความเหมาะสม
```

### 953.4 Point-in-Time Recovery (PITR) แบบ Managed

การกู้คืนข้อมูล ณ จุดเวลาใด ๆ ภายใน retention window ทำได้ผ่านคำสั่งเดียว โดย Azure จะสร้าง **server ใหม่** จาก backup + WAL replay ให้อัตโนมัติ (ไม่ overwrite server เดิม เพื่อความปลอดภัย เช่นเดียวกับหลักการ PITR ของ AWS RDS และ GCP Cloud SQL ที่เคยเรียนมา):

```bash
# กู้คืนไปยังจุดเวลาที่กำหนด (in-region restore)
az postgres flexible-server restore \
  --resource-group rg-ecommerce-prod \
  --name pg-ecommerce-prod-th-restored \
  --source-server pg-ecommerce-prod-th \
  --restore-time "2026-09-20T03:15:00Z"

# กู้คืนข้าม region โดยใช้ geo-redundant backup (Disaster Recovery scenario)
az postgres flexible-server geo-restore \
  --resource-group rg-ecommerce-dr \
  --name pg-ecommerce-dr-restored \
  --source-server pg-ecommerce-prod-th \
  --location eastasia
```

| ประเภทการกู้คืน | คำสั่ง | ใช้เมื่อ |
|---|---|---|
| In-region PITR | `az postgres flexible-server restore` | ข้อมูลเสียหายจาก human error (ลบ/แก้ข้อมูลผิด) แต่ region ยังปกติ |
| Geo-restore | `az postgres flexible-server geo-restore` | Primary region ล่มทั้งภูมิภาค ต้องกู้คืนไปยัง region สำรอง (ต้องเปิด geo-redundant backup ไว้ล่วงหน้า) |

> **เทียบเคียงกับที่เรียนมา:** แนวคิดนี้เหมือนกับ AWS RDS Cross-Region Automated Backup + PITR และ GCP Cloud SQL Cross-region backup replication ที่เคยเรียนใน Part 094-095 — หลักการพื้นฐานคือ "จะกู้คืนข้าม region ได้ ต้องเปิดฟีเจอร์ replicate backup ข้าม region ไว้ล่วงหน้าเสมอ" เป็นหลักการร่วมของทั้งสาม Cloud

---

## Step 954: Azure Database Networking

### 954.1 สองรูปแบบการเชื่อมต่อเครือข่าย

Azure Flexible Server รองรับ networking 2 รูปแบบหลัก ซึ่งต้อง **เลือกตอนสร้าง server และเปลี่ยนภายหลังได้ยาก/มีข้อจำกัด**:

1. **Public access (allowed IP addresses)** — server มี public endpoint พร้อม firewall rule จำกัด IP ที่อนุญาต
2. **Private access (VNet Integration)** — server ถูกฝังอยู่ใน Virtual Network โดยตรง ไม่มี public endpoint เลย

```
รูปแบบที่ 1: Public Access + Firewall Rules
┌─────────────┐        Internet         ┌────────────────────────┐
│ Application  │ ──────────────────────→ │  Flexible Server        │
│ (on-premise  │   (ผ่าน Firewall Rule    │  (Public Endpoint)      │
│  หรือที่อื่น) │    จำกัดเฉพาะ IP ที่      │                          │
│              │    อนุญาต)               │                          │
└─────────────┘                         └────────────────────────┘
  * ตั้งค่าง่าย เหมาะกับ dev/test หรือเชื่อมจากภายนอก Azure
  * ความเสี่ยงด้านความปลอดภัยสูงกว่า (attack surface กว้างกว่า)

รูปแบบที่ 2: VNet Integration (Private Access)
┌───────────────────────────────────────────────────────────┐
│                  Virtual Network (VNet)                    │
│  ┌─────────────────────┐      ┌────────────────────────┐  │
│  │  App Subnet          │      │  Database Subnet        │  │
│  │  (snet-app-tier)     │      │  (snet-db-tier,         │  │
│  │  ┌─────────────────┐ │      │   delegated to          │  │
│  │  │  App Server /    │─┼─────→│  Microsoft.DBforPostgreSQL) │
│  │  │  AKS Pod         │ │      │  ┌────────────────────┐ │  │
│  │  └─────────────────┘ │      │  │ Flexible Server     │ │  │
│  │                       │      │  │ (Private IP only)   │ │  │
│  └─────────────────────┘      │  └────────────────────┘ │  │
│                                 └────────────────────────┘  │
└───────────────────────────────────────────────────────────┘
  * ไม่มี public endpoint เลย — ปลอดภัยที่สุด
  * ต้องใช้ Private DNS Zone เพื่อ resolve ชื่อ server
  * เหมาะสำหรับ Production
```

### 954.2 การสร้าง Flexible Server แบบ VNet Integration

```bash
# 1) สร้าง Virtual Network และ Subnet สำหรับ database (ต้อง delegate ให้ PostgreSQL)
az network vnet create \
  --resource-group rg-ecommerce-prod \
  --name vnet-ecommerce-prod \
  --address-prefix 10.10.0.0/16 \
  --subnet-name snet-app-tier \
  --subnet-prefix 10.10.1.0/24

az network vnet subnet create \
  --resource-group rg-ecommerce-prod \
  --vnet-name vnet-ecommerce-prod \
  --name snet-db-tier \
  --address-prefix 10.10.2.0/24 \
  --delegations Microsoft.DBforPostgreSQL/flexibleServers

# 2) สร้าง Private DNS Zone สำหรับ resolve ชื่อ server ภายใน VNet
az network private-dns zone create \
  --resource-group rg-ecommerce-prod \
  --name pg-ecommerce-prod-th.private.postgres.database.azure.com

az network private-dns link vnet create \
  --resource-group rg-ecommerce-prod \
  --zone-name pg-ecommerce-prod-th.private.postgres.database.azure.com \
  --name link-ecommerce-prod \
  --virtual-network vnet-ecommerce-prod \
  --registration-enabled false

# 3) สร้าง Flexible Server แบบ VNet-integrated (ไม่มี public access เลย)
az postgres flexible-server create \
  --resource-group rg-ecommerce-prod \
  --name pg-ecommerce-prod-th \
  --location southeastasia \
  --vnet vnet-ecommerce-prod \
  --subnet snet-db-tier \
  --private-dns-zone pg-ecommerce-prod-th.private.postgres.database.azure.com \
  --admin-user pgadmin_ecom \
  --admin-password 'Str0ngP@ssw0rd!ChangeMe' \
  --sku-name Standard_D4ds_v5 \
  --tier GeneralPurpose \
  --version 16
```

### 954.3 Private Endpoint vs VNet Integration — ความแตกต่างที่ต้องเข้าใจ

จุดที่ผู้เรียนมักสับสน คือ Azure มีสองกลไกที่คล้ายกันแต่ต่างกัน:

| กลไก | ลักษณะการทำงาน | ใช้เมื่อ |
|---|---|---|
| **VNet Integration** (delegated subnet) | Flexible Server ถูก **ฝัง** อยู่ใน subnet ของ VNet โดยตรง มี private IP จาก subnet นั้นเลย | วิธีมาตรฐานสำหรับ Flexible Server — ใช้ตอนสร้าง server ตั้งแต่แรก |
| **Private Endpoint** | สร้าง NIC (Network Interface) เพิ่มเข้าไปใน VNet อื่น ที่ "ชี้" กลับไปยัง resource ที่อยู่นอก VNet นั้น (ผ่าน Azure Private Link) | ใช้เมื่อต้องการให้ VNet อื่น (เช่น VNet ของทีมอื่น หรือ VNet ต่าง subscription) เข้าถึง resource โดยไม่ผ่าน public internet — พบมากกับ PaaS บางตัว เช่น Azure Storage, Cosmos DB |

> **ข้อควรรู้:** สำหรับ Azure Database for PostgreSQL **Flexible Server** วิธีหลักที่ใช้คือ **VNet Integration (delegated subnet)** ไม่ใช่ Private Endpoint แบบที่ใช้กับบริการ PaaS อื่น ๆ ของ Azure — นี่คือจุดที่มักทำให้สับสนเวลาข้ามไปข้ามมาระหว่างเอกสารของบริการต่าง ๆ ใน Azure ควรตรวจสอบเอกสารทางการเฉพาะบริการเสมอ

### 954.4 เทียบเคียงกับ Networking ของ AWS/GCP

| แนวคิด | AWS RDS/Aurora | GCP Cloud SQL/AlloyDB | Azure Flexible Server |
|---|---|---|---|
| การฝังใน private network | VPC + DB Subnet Group | VPC + Private Service Access / Private Service Connect | VNet + Delegated Subnet (VNet Integration) |
| Firewall/Access control | Security Group | Authorized Networks / Firewall Rule | Firewall Rule (สำหรับ public access) |
| DNS ภายใน | Route 53 Private Hosted Zone (ถ้าใช้ custom) | Cloud DNS Private Zone | Azure Private DNS Zone |
| Cross-VNet/VPC access | VPC Peering / Transit Gateway | VPC Peering / Private Service Connect | VNet Peering |

หลักการพื้นฐานเหมือนกันทั้งสาม Cloud: **"Production database ไม่ควรมี public endpoint"** — ความแตกต่างอยู่ที่ terminology และรายละเอียดการตั้งค่าเท่านั้น

---

## สรุปท้ายบท

### ตารางเปรียบเทียบ Cloud PostgreSQL ทั้งสามผู้ให้บริการอย่างครบวงจร

| มิติเปรียบเทียบ | **AWS RDS PostgreSQL / Aurora PostgreSQL** | **GCP Cloud SQL PostgreSQL / AlloyDB** | **Azure Database for PostgreSQL (Flexible Server)** |
|---|---|---|---|
| **Pricing Model** | Pay-as-you-go ตาม instance class + storage (GP2/GP3/io1/io2) + I/O (RDS) หรือ ACU-based (Aurora Serverless v2); มี Reserved Instance ลดราคาได้ถึง ~60% | Pay-as-you-go ตาม machine type + storage; มี Committed Use Discount (CUD) ลดราคาได้ถึง ~50-60%; AlloyDB คิดราคาแยกสำหรับ compute/storage/columnar cache | Pay-as-you-go ตาม compute tier/SKU + storage; มี Reserved Capacity (1 หรือ 3 ปี) ลดราคาได้ใกล้เคียงคู่แข่ง |
| **High Availability** | Multi-AZ (synchronous standby ข้าม AZ); Aurora มี 6-way replication ข้าม 3 AZ ในตัว storage layer | Regional configuration (synchronous standby ข้าม zone); AlloyDB มี built-in HA แบบ multi-zone อัตโนมัติ | ZoneRedundant HA (synchronous standby ข้าม zone) หรือ SameZone HA |
| **Storage สูงสุด** | RDS PostgreSQL: สูงสุด 64 TiB (gp3/io1/io2); Aurora: auto-scaling สูงสุด 128 TiB | Cloud SQL: สูงสุด 64 TiB; AlloyDB: auto-scaling ไม่จำกัดตายตัว (จัดการโดยระบบ) | Flexible Server: สูงสุด 32 TiB (ขึ้นกับ storage type ที่เลือก) |
| **Read Replica** | รองรับ (สูงสุด 5 สำหรับ RDS, สูงสุด 15 สำหรับ Aurora) รวมถึง Cross-Region Read Replica | รองรับ (Cloud SQL สูงสุด 10 read replica ต่อ instance รวม cross-region); AlloyDB รองรับ read pool แบบ auto-scaling | รองรับ Read Replica (รวม cross-region read replica) แต่จำนวนสูงสุดต่อ primary น้อยกว่า Aurora โดยเปรียบเทียบ |
| **Serverless/Auto-scaling compute** | มี — Aurora Serverless v2 (scale ACU อัตโนมัติแบบ near-real-time) | ไม่มีสำหรับ Cloud SQL โดยตรง (แต่ AlloyDB มี auto-scaling storage); ไม่มี compute serverless เทียบเท่า Aurora | ไม่มี — ต้องเลือก SKU คงที่ หรือใช้ Burstable tier เพื่อความยืดหยุ่นระดับหนึ่ง |
| **Point-in-Time Recovery** | รองรับ ภายใน retention window (สูงสุด 35 วัน) | รองรับ ภายใน retention window (สูงสุด 365 วันสำหรับบาง config) | รองรับ ภายใน retention window (สูงสุด 35 วัน) |
| **Cross-region DR ผ่าน Backup** | Cross-Region Automated Backup Replication | Cross-region backup location | Geo-redundant Backup (ต้องเปิดตอนสร้าง server เท่านั้น) |
| **จุดเด่นที่โดดเด่นที่สุด (Standout Feature)** | Aurora: storage-compute แยกกัน, replication ระดับ storage layer ที่เร็วมาก, Aurora Serverless v2, ระบบนิเวศ AWS ที่ใหญ่ที่สุด | AlloyDB: columnar engine สำหรับ analytical query แบบ HTAP, AI/ML integration แน่นแฟ้นกับ Vertex AI, ราคาแข่งขันสูง | ความง่ายในการ integrate กับ Azure AD (Microsoft Entra ID) สำหรับ authentication, Zone-redundant HA ตั้งค่าง่าย, เหมาะมากสำหรับองค์กรที่ใช้ Microsoft ecosystem (Active Directory, .NET, Power BI) อยู่แล้ว |
| **Extension ที่โดดเด่น** | รองรับ extension หลากหลายผ่าน parameter group allowlist | รองรับ `pgvector`, PostGIS เต็มรูปแบบ | รองรับ `pgvector`, `pg_cron`, `pg_partman`, PostGIS ผ่าน allowlist ของ Azure |
| **Authentication แบบ Cloud-native** | IAM Database Authentication | Cloud IAM Database Authentication | Microsoft Entra ID (Azure AD) Authentication |

### ประเด็นสำคัญที่ควรจำ

1. **Flexible Server คือมาตรฐานปัจจุบัน** — อย่าเลือก Single Server สำหรับ deployment ใหม่เด็ดขาด เพราะถูก retire แล้ว
2. **Geo-redundant Backup ต้องเปิดตั้งแต่สร้าง server** — เป็นการตัดสินใจที่เปลี่ยนใจภายหลังไม่ได้ (ต้องสร้าง server ใหม่) จึงควรวางแผนเรื่อง Disaster Recovery **ก่อน** provisioning เสมอ
3. **Azure ไม่มี Serverless compute tier เทียบเท่า Aurora Serverless v2** — หาก workload ต้องการ auto-scaling compute แบบ real-time Azure อาจไม่ใช่ตัวเลือกที่ดีที่สุด
4. **VNet Integration (ไม่ใช่ Private Endpoint) คือวิธีมาตรฐาน** ในการทำให้ Flexible Server ไม่มี public exposure
5. **จุดแข็งที่แท้จริงของ Azure** อยู่ที่การ integrate เข้ากับระบบนิเวศ Microsoft (Entra ID, Power BI, Azure Synapse, .NET ecosystem) มากกว่าการแข่งขันด้านฟีเจอร์ database engine ล้วน ๆ
6. หลักการพื้นฐานของ Cloud Database ทั้งสามเจ้า (HA ข้าม Availability Zone, PITR ผ่าน WAL, Private networking, Compute/Storage แยกอิสระ) **เหมือนกันในระดับแนวคิด** ต่างกันแค่ terminology และรายละเอียดปลีกย่อย — เมื่อเข้าใจแนวคิดร่วมนี้แล้ว การย้ายความรู้ข้าม Cloud provider จะทำได้ง่ายขึ้นมาก

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: เลือก Deployment รุ่น

บริษัทกำลังจะเริ่มโปรเจกต์ใหม่และกำลังดูตัวอย่างสถาปัตยกรรมเก่าที่เขียนไว้เมื่อหลายปีก่อน ซึ่งระบุให้ใช้ "Azure Database for PostgreSQL — Single Server" จงอธิบายว่าทำไมจึงไม่ควรใช้ตามเอกสารเก่านี้ และควรแก้ไขเป็นอะไร

<details>
<summary>เฉลย</summary>

Single Server เป็นรุ่นเก่าที่ Microsoft ประกาศ retirement และเลิกรองรับแล้ว ไม่สามารถสร้าง instance ใหม่ได้อีกต่อไป (หรือหากยังสร้างได้ก็ไม่ควรใช้เพราะไม่มี security patch/feature update ในระยะยาว) ควรแก้ไขเอกสารสถาปัตยกรรมให้ระบุ **Flexible Server** แทน ซึ่งเป็นรุ่นมาตรฐานปัจจุบันที่รองรับ Zone-redundant HA, VNet Integration แบบ native, Stop/Start, Burstable compute tier และ extension ที่หลากหลายกว่า

</details>

---

### แบบฝึกหัดที่ 2: เลือก Compute Tier

ระบบวิเคราะห์ข้อมูล (analytics dashboard) ภายในองค์กรที่ query ข้อมูลปริมาณมากและต้องการ working set ขนาดใหญ่ใน memory เพื่อลด disk I/O ควรเลือก Compute Tier ใดของ Azure Flexible Server? และ SKU ตระกูลใด?

<details>
<summary>เฉลย</summary>

ควรเลือก **Memory Optimized (E-series)** เช่น `Standard_E8ds_v5` หรือสูงกว่า เนื่องจาก workload ประเภท analytics ที่มี working set ขนาดใหญ่จะได้ประโยชน์จาก CPU:Memory ratio ที่สูง (1:8) ทำให้ shared_buffers และ OS page cache สามารถเก็บข้อมูลที่ใช้บ่อยไว้ใน RAM ได้มากขึ้น ลดการอ่านจาก disk ลงอย่างมีนัยสำคัญ ซึ่งต่างจาก General Purpose (D-series) ที่มี ratio 1:4 และ Burstable (B-series) ที่เหมาะกับ workload เบาและไม่สม่ำเสมอมากกว่า

</details>

---

### แบบฝึกหัดที่ 3: ออกแบบ HA

ระบบ e-commerce ของบริษัทต้องการ RPO = 0 (ไม่ยอมให้ข้อมูลสูญหายเลยแม้แต่ transaction เดียว) เมื่อ instance ปัจจุบันล่ม จะต้อง config Azure Flexible Server อย่างไร และมีข้อจำกัดอะไรที่ต้องระวัง?

<details>
<summary>เฉลย</summary>

ต้องเปิดใช้ **ZoneRedundant HA** (`--high-availability ZoneRedundant`) ซึ่งใช้ **synchronous replication** ระหว่าง Primary กับ Standby ทำให้ transaction จะ commit สำเร็จก็ต่อเมื่อถูกเขียนไปยังทั้งสอง instance แล้ว จึงได้ RPO = 0 เมื่อเกิด failover

ข้อจำกัดที่ต้องระวัง:
- Synchronous replication เพิ่ม write latency เล็กน้อยเนื่องจากต้องรอ acknowledge จาก standby ก่อน commit
- Failover ใช้เวลาประมาณ 60-120 วินาที (ไม่ใช่ instantaneous) — แอปพลิเคชันต้องมี retry logic ที่รองรับ connection drop ช่วงสั้น ๆ
- ZoneRedundant HA ป้องกันได้เฉพาะ instance/zone failure เท่านั้น หากต้องการป้องกัน regional disaster ด้วย ต้องเพิ่ม Geo-redundant Backup หรือ cross-region read replica แยกต่างหาก (ซึ่งไม่ใช่ synchronous จึงไม่การันตี RPO=0 ในกรณี regional disaster)

</details>

---

### แบบฝึกหัดที่ 4: Geo-redundant Backup

ทีม DevOps สร้าง Flexible Server ไปแล้วโดยไม่ได้เปิด Geo-redundant Backup ตั้งแต่แรก ภายหลังพบว่าต้องการ Disaster Recovery ข้าม region จะแก้ไขปัญหานี้อย่างไร?

<details>
<summary>เฉลย</summary>

Geo-redundant Backup เป็นค่าที่กำหนดได้เฉพาะตอนสร้าง server เท่านั้น (immutable setting) ไม่สามารถเปิดใช้งานภายหลังได้ ทางแก้คือ:

1. สร้าง Flexible Server ใหม่โดยเปิด `--geo-redundant-backup Enabled` ตั้งแต่ตอนสร้าง
2. ย้ายข้อมูลจาก server เดิมไปยัง server ใหม่ (เช่นใช้ `pg_dump`/`pg_restore`, Azure Database Migration Service, หรือ logical replication เพื่อลด downtime)
3. สลับ connection string ของแอปพลิเคชันไปยัง server ใหม่
4. ปิด/ลบ server เดิมหลังยืนยันว่าระบบทำงานปกติแล้ว

บทเรียนสำคัญ: ควรวางแผนเรื่อง DR requirement **ก่อน** provisioning เสมอ ไม่ใช่ทำเป็น afterthought

</details>

---

### แบบฝึกหัดที่ 5: Networking

จงอธิบายความแตกต่างระหว่าง "Public access with firewall rules" กับ "VNet Integration" ของ Azure Flexible Server และควรเลือกแบบไหนสำหรับระบบ production ของ e-commerce ที่ต้องเชื่อมกับ application ที่รันบน AKS (Azure Kubernetes Service) ในองค์กรเดียวกัน

<details>
<summary>เฉลย</summary>

**Public access** ให้ server มี public endpoint บน internet และใช้ Firewall Rule จำกัด IP ที่อนุญาตเชื่อมต่อ ตั้งค่าง่ายแต่มี attack surface กว้างกว่า (เสี่ยงต่อการถูกโจมตีจากภายนอกมากกว่า แม้จะมี firewall กรองอยู่ก็ตาม)

**VNet Integration** ฝัง Flexible Server เข้าไปใน subnet ของ Virtual Network โดยตรง (ผ่าน delegated subnet) ทำให้ไม่มี public endpoint เลย เข้าถึงได้เฉพาะจากภายใน VNet เดียวกันหรือ VNet ที่ peer กันเท่านั้น ปลอดภัยกว่ามาก

สำหรับระบบ production ที่เชื่อมกับ AKS ในองค์กรเดียวกัน ควรเลือก **VNet Integration** เสมอ โดยวาง AKS cluster และ Flexible Server ไว้ใน VNet เดียวกัน (คนละ subnet) เพื่อให้การสื่อสารเกิดขึ้นผ่าน private network ทั้งหมด ไม่ผ่าน public internet เลย ลดความเสี่ยงด้านความปลอดภัยและมักได้ latency ที่ดีกว่าด้วย

</details>

---

### แบบฝึกหัดที่ 6: เปรียบเทียบ Storage สูงสุด

ระบบ e-commerce ขนาดใหญ่คาดการณ์ว่าข้อมูล transaction และ log จะเติบโตถึง 40 TiB ภายใน 3 ปีข้างหน้า จงเปรียบเทียบว่า AWS Aurora, GCP AlloyDB และ Azure Flexible Server รองรับขนาดนี้ได้หรือไม่ และมีข้อพิจารณาอะไรเพิ่มเติม

<details>
<summary>เฉลย</summary>

- **AWS Aurora PostgreSQL**: รองรับสบาย ๆ เพราะ auto-scaling storage สูงสุดถึง 128 TiB
- **GCP AlloyDB**: รองรับได้เช่นกัน เพราะมี auto-scaling storage ที่จัดการโดยระบบ ไม่มีเพดานตายตัวที่ต่ำกว่าความต้องการนี้
- **Azure Flexible Server**: เพดานสูงสุดของ Flexible Server อยู่ที่ **32 TiB** ซึ่ง **ไม่เพียงพอ** สำหรับ 40 TiB ที่คาดการณ์ไว้

ข้อพิจารณาเพิ่มเติม: หากองค์กรมีแผนใช้ Azure เป็นหลักและคาดว่าข้อมูลจะโตเกิน 32 TiB จำเป็นต้องวางแผน **data partitioning/archiving strategy** ล่วงหน้า (เช่น ย้ายข้อมูลเก่าไป Azure Data Lake หรือ Synapse Analytics, ทำ table partitioning + archival job) หรือพิจารณาเปลี่ยนไปใช้ AWS/GCP หากขนาดข้อมูลเป็นปัจจัยตัดสินใจหลัก ควรประเมิน growth trajectory ให้แม่นยำที่สุดก่อนเลือกผู้ให้บริการตั้งแต่ต้น เพราะการย้าย Cloud provภายหลังมีต้นทุนสูงมาก

</details>

---

### แบบฝึกหัดที่ 7: สถานการณ์ธุรกิจ — Startup ขนาดเล็ก

Startup e-commerce ขนาดเล็กเพิ่งเริ่มต้น มีงบประมาณจำกัด ทีมมีขนาดเล็ก (ไม่มี DBA เฉพาะทาง) และ traffic ไม่สม่ำเสมอ (peak เฉพาะช่วงโปรโมชัน) ผู้ให้บริการ Cloud ใดและ configuration แบบใดที่เหมาะสมที่สุด?

<details>
<summary>เฉลย</summary>

สถานการณ์นี้เหมาะกับ **AWS Aurora Serverless v2** มากที่สุด เพราะสามารถ auto-scale ACU (compute) ตาม load แบบ near-real-time โดยไม่ต้องมีทีม DBA คอยปรับขนาด instance เอง จ่ายตามการใช้งานจริง ประหยัดค่าใช้จ่ายช่วง traffic ต่ำ และรองรับ peak ช่วงโปรโมชันได้อัตโนมัติ

ทางเลือกรองลงมา: หากทีมต้องการความเรียบง่ายและงบจำกัดมาก อาจพิจารณา **Azure Flexible Server แบบ Burstable tier** (เช่น `Standard_B2s`) หรือ **GCP Cloud SQL** ขนาดเล็กที่ scale ขึ้น-ลงด้วยมือ (manual scaling) ร่วมกับ monitoring/alert เพื่อปรับขนาดล่วงหน้าก่อนช่วงโปรโมชัน แต่จะต้องมีคนคอยเฝ้าและปรับ config เอง ซึ่งเพิ่มภาระให้ทีมเล็ก ๆ

</details>

---

### แบบฝึกหัดที่ 8: สถานการณ์ธุรกิจ — องค์กรใหญ่ใช้ Microsoft Ecosystem

บริษัทค้าปลีกขนาดใหญ่ใช้ Microsoft 365, Active Directory (Entra ID), Power BI สำหรับ reporting และมีนโยบายบริษัทให้ authentication ทุกระบบต้องผ่าน Entra ID แบบ Single Sign-On จะเลือก Cloud provider ใดสำหรับ PostgreSQL database ของระบบ e-commerce?

<details>
<summary>เฉลย</summary>

**Azure Database for PostgreSQL (Flexible Server)** เป็นตัวเลือกที่เหมาะสมที่สุดในสถานการณ์นี้ เพราะ:

1. รองรับ **Microsoft Entra ID Authentication** แบบ native ทำให้ integrate เข้ากับนโยบาย SSO ขององค์กรได้โดยตรง ไม่ต้องสร้างระบบ credential แยกต่างหาก
2. เชื่อมต่อกับ **Power BI** สำหรับ reporting ได้อย่างราบรื่น เพราะเป็น ecosystem เดียวกัน
3. ทีม IT ที่คุ้นเคยกับ Azure Portal และ Azure Active Directory อยู่แล้ว จะลด learning curve และภาระด้าน operations ลงมาก
4. Compliance/Governance tools ขององค์กร (เช่น Azure Policy, Microsoft Defender for Cloud) ทำงานร่วมกับ Azure resource ได้แนบแน่นกว่า

แม้ AWS หรือ GCP จะมีฟีเจอร์ database engine ที่แข่งขันได้ (หรือดีกว่าในบางมิติ เช่น Aurora Serverless) แต่ต้นทุนรวม (integration effort, การจัดการ identity แยกระบบ, training ทีม) มักสูงกว่าเมื่อองค์กรผูกกับ Microsoft ecosystem อยู่แล้ว ในกรณีนี้ "fit กับระบบนิเวศที่มีอยู่" มีน้ำหนักมากกว่าฟีเจอร์ database เดี่ยว ๆ

</details>

---

### แบบฝึกหัดที่ 9: สถานการณ์ธุรกิจ — ระบบต้องการ Analytics แบบ Real-time

ทีมข้อมูล (Data team) ต้องการรัน analytical query ที่ซับซ้อน (aggregation ข้ามตารางขนาดใหญ่, JOIN หลายตาราง) บนข้อมูล transaction สด ๆ ของระบบ e-commerce แบบ near real-time โดยไม่ต้องการ ETL pipeline แยกไปยัง data warehouse ต่างหาก ผู้ให้บริการ Cloud ใดตอบโจทย์นี้ได้ดีที่สุด?

<details>
<summary>เฉลย</summary>

**GCP AlloyDB for PostgreSQL** ตอบโจทย์นี้ได้ดีที่สุด เพราะมี **columnar engine** ในตัวที่ทำงานร่วมกับ storage แบบ row-based ปกติ (HTAP — Hybrid Transactional/Analytical Processing) ทำให้สามารถรัน analytical query ที่ซับซ้อนบนข้อมูล transactional สด ๆ ได้เร็วกว่า PostgreSQL มาตรฐานหลายเท่า โดยไม่ต้องทำ ETL ไปยัง data warehouse แยกต่างหาก

เปรียบเทียบ:
- **AWS Aurora PostgreSQL** และ **Azure Flexible Server** เป็น row-based storage แบบมาตรฐาน หากต้องการ analytical query ที่ซับซ้อนมาก มักต้องพึ่ง read replica แยก + ETL ไปยัง data warehouse (Redshift สำหรับ AWS, Synapse Analytics สำหรับ Azure) ซึ่งมี latency สูงกว่าและซับซ้อนกว่า
- **AlloyDB** จึงเป็นตัวเลือกที่ตรงโจทย์ที่สุดสำหรับ use case แบบ HTAP โดยเฉพาะ

</details>

---

### แบบฝึกหัดที่ 10: สถานการณ์ธุรกิจ — Multi-cloud Disaster Recovery (โจทย์รวบยอด)

บริษัท e-commerce ขนาดกลาง-ใหญ่ตัดสินใจใช้ Azure เป็น primary cloud (เพราะ integrate กับระบบ ERP ที่ใช้ Microsoft Dynamics 365 อยู่แล้ว) แต่ฝ่ายบริหารความเสี่ยงกำหนดนโยบายว่าต้องมีแผน Business Continuity ที่ไม่ผูกกับผู้ให้บริการ Cloud รายเดียว (avoid vendor lock-in สำหรับ disaster recovery) จงออกแบบสถาปัตยกรรมระดับสูง (high-level) ที่ตอบโจทย์นี้ โดยอธิบายการตัดสินใจสำคัญอย่างน้อย 4 ข้อ

<details>
<summary>เฉลย</summary>

สถาปัตยกรรมระดับสูงที่แนะนำ:

```
┌─────────────────────────────┐         ┌─────────────────────────────┐
│   Azure (Primary)             │         │   AWS หรือ GCP (DR Site)      │
│                                │         │                                │
│  ┌──────────────────────────┐ │         │  ┌──────────────────────────┐  │
│  │ Flexible Server            │ │  Logical │  │ RDS PostgreSQL / Cloud SQL │  │
│  │ (ZoneRedundant HA)         │─┼─Replic-─┼─→│ (Standby, read-only)       │  │
│  │                            │ │  ation   │  │                            │  │
│  └──────────────────────────┘ │         │  └──────────────────────────┘  │
│  + Geo-redundant Backup        │         │                                │
│    (ภายใน Azure region คู่)     │         │                                │
└─────────────────────────────┘         └─────────────────────────────┘
```

การตัดสินใจสำคัญ 4 ข้อ:

1. **ใช้ Logical Replication (pglogical / native logical replication) แทน storage-level replication** — เพราะ storage-level HA/DR (เช่น ZoneRedundant HA, Geo-redundant Backup) ผูกอยู่กับ Azure เท่านั้น ไม่สามารถ replicate ข้ามไปยัง AWS/GCP ได้โดยตรง จึงต้องใช้ PostgreSQL logical replication ที่ทำงานได้ข้าม Cloud provider เพราะเป็นกลไกระดับ PostgreSQL engine เอง ไม่ใช่ฟีเจอร์เฉพาะของ Cloud provider ใดผู้ให้บริการหนึ่ง

2. **ยังคงใช้ Azure Zone-redundant HA + Geo-redundant Backup เป็นชั้นป้องกันแรก** — สำหรับ incident ระดับ instance/zone/region เพราะเร็วกว่าและง่ายกว่าการ failover ข้าม cloud provider ซึ่งควรสงวนไว้สำหรับ catastrophic scenario เท่านั้น (defense in depth — มีหลายชั้นการป้องกัน)

3. **เลือก DR site บน Cloud provider ที่ต่างจาก Azure โดยสิ้นเชิง** (เช่น AWS RDS PostgreSQL หรือ GCP Cloud SQL) เพื่อให้ตอบโจทย์ "ไม่ผูกกับผู้ให้บริการรายเดียว" ตามนโยบายบริหารความเสี่ยง — เลือกใช้บริการ PostgreSQL มาตรฐาน (ไม่ใช่ Aurora/AlloyDB ที่มี proprietary extension) เพื่อลดความซับซ้อนในการ sync schema/extension ระหว่างสอง engine

4. **วางแผน DNS/Traffic routing และ Runbook สำหรับ failover ข้าม Cloud** ล่วงหน้า (เช่นใช้ DNS-based traffic manager ที่ไม่ผูกกับ Cloud provider ใดรายหนึ่ง, เตรียม connection string / secret ของทั้งสอง environment ไว้ใน secret manager ที่เข้าถึงได้จากทั้งสองฝั่ง) พร้อมทั้งทดสอบ DR drill เป็นระยะ เพราะแผน DR ที่ไม่เคยทดสอบจริง มักใช้งานไม่ได้จริงเมื่อเกิดเหตุการณ์จริง

ข้อควรระวังเพิ่มเติม: ต้นทุนการดูแล multi-cloud DR สูงกว่าการอยู่ใน Cloud provider เดียวมาก (ทั้งค่าใช้จ่ายและความซับซ้อนในการดำเนินงาน) จึงควรประเมินร่วมกับฝ่ายบริหารว่าความเสี่ยงที่ต้องการป้องกันนั้นคุ้มค่ากับต้นทุนที่เพิ่มขึ้นหรือไม่ ก่อนตัดสินใจลงทุนสร้างสถาปัตยกรรมแบบนี้เต็มรูปแบบ

</details>

---

## บทถัดไป

บทถัดไปจะพาไปสำรวจการทำ **Distributed PostgreSQL** ด้วย **Citus** ซึ่งเป็น extension ที่เปลี่ยน PostgreSQL ให้กลายเป็นฐานข้อมูลแบบ distributed ที่ scale ออกในแนวนอน (horizontal scaling) ได้ — เทคนิคที่สำคัญมากสำหรับระบบที่มีข้อมูลขนาดใหญ่เกินกว่าที่ single-node PostgreSQL จะรองรับได้อย่างมีประสิทธิภาพ

**อ่านต่อ:** [Part 097: Citus และ Distributed PostgreSQL](./part-097-citus-distributed.md)
