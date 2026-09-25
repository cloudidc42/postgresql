# Part 094: PostgreSQL บน Cloud — AWS RDS และ Aurora PostgreSQL

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับโลก/ผู้เชี่ยวชาญ | Part 094

---

> **หมายเหตุสำคัญก่อนเริ่มบท**
>
> บทนี้เป็นเนื้อหาเชิงสถาปัตยกรรมและแนวคิด (architectural/conceptual) เกี่ยวกับ PostgreSQL บน AWS โดยเฉพาะ Amazon RDS for PostgreSQL และ Amazon Aurora PostgreSQL-Compatible Edition เนื่องจากสภาพแวดล้อมของบทเรียนนี้ไม่สามารถเชื่อมต่อไปยัง AWS Account จริง ไม่สามารถ provision ทรัพยากรจริง และไม่สามารถแคปภาพหน้าจอ AWS Console ได้ เนื้อหาทั้งหมดจึงอธิบายผ่าน:
>
> 1. **AWS CLI commands** ที่ถูกต้องตามไวยากรณ์จริง (syntax ที่ตรงกับ `aws rds` command reference) เพื่อให้นำไปรันบน AWS Account จริงของท่านได้ทันที
> 2. **ASCII Architecture Diagram** เพื่อแสดงโครงสร้างระบบแทนภาพหน้าจอ
> 3. **ตารางเปรียบเทียบและตารางตัดสินใจ (decision table)** ที่อ้างอิงจากเอกสารทางการของ AWS และพฤติกรรมที่ทราบกันดีในวงการ
> 4. **คำเตือนเรื่องราคา** — ตัวเลขราคาในบทนี้เป็น **ตัวอย่างประมาณการ** (illustrative estimate) เพื่อการเรียนรู้เรื่องการคิดต้นทุนเชิงสถาปัตยกรรมเท่านั้น ราคาจริงเปลี่ยนแปลงตาม region, ช่วงเวลา, และ AWS Pricing ปัจจุบัน **ท่านต้องตรวจสอบราคาจริงจาก [AWS Pricing Calculator](https://calculator.aws) และหน้า Pricing ของแต่ละบริการก่อนตัดสินใจใช้งานจริงเสมอ**
>
> ผู้เรียนควรมี AWS Account (Free Tier ก็เพียงพอสำหรับทดลองคำสั่งพื้นฐาน) เพื่อฝึกปฏิบัติจริงควบคู่ไปกับบทเรียนนี้

---

## เป้าหมายการเรียนรู้

เมื่อจบบทนี้ ผู้เรียนจะสามารถ:

1. อธิบายได้ว่า Amazon RDS for PostgreSQL คือ Managed Database Service แบบใด และ AWS รับผิดชอบส่วนใดของ stack แทนเรา
2. เลือก Instance Class และ Storage Type (gp3/io2) ที่เหมาะสมกับ workload พร้อมตั้งค่า Storage Autoscaling
3. ออกแบบและตั้งค่า Multi-AZ Deployment เพื่อความพร้อมใช้งานสูง และเข้าใจความแตกต่างจากการทำ Patroni cluster เอง (ตามที่เรียนใน Part 065)
4. สร้างและจัดการ Read Replica ทั้งแบบ same-region และ cross-region พร้อมเข้าใจเรื่อง replication lag
5. ตั้งค่า Automated Backup, Manual Snapshot และทำ Point-in-Time Recovery (PITR) แบบ managed เปรียบเทียบกับการทำ PITR เองใน Part 062
6. อธิบายสถาปัตยกรรมของ Amazon Aurora PostgreSQL ที่แยก compute ออกจาก storage และเข้าใจกลไก 6 copies across 3 AZ
7. ใช้งาน Aurora Cluster Endpoint, Reader Endpoint และตั้งค่า Aurora Auto Scaling สำหรับ Read Replica
8. ตัดสินใจได้ว่าเมื่อใดควรใช้ Aurora Serverless v2 แทน Provisioned Instance
9. ตั้งค่า DB Parameter Group, DB Cluster Parameter Group และ Security Group อย่างปลอดภัยภายใน VPC
10. ออกแบบสถาปัตยกรรม AWS สำหรับระบบ e-commerce จริง พร้อมประมาณการต้นทุนเบื้องต้น และให้เหตุผลในการเลือก RDS หรือ Aurora

---

## แผนที่บทเรียน (Steps 931–940)

| Step | หัวข้อ |
|------|--------|
| 931 | Amazon RDS for PostgreSQL คืออะไร |
| 932 | RDS Instance Class และ Storage |
| 933 | RDS Multi-AZ Deployment |
| 934 | RDS Read Replica |
| 935 | RDS Backup และ Snapshot |
| 936 | Amazon Aurora PostgreSQL คืออะไร |
| 937 | Aurora Reader/Writer Endpoint |
| 938 | Aurora Serverless v2 |
| 939 | Parameter Groups และ Security Groups |
| 940 | แบบฝึกหัดรวม: ออกแบบสถาปัตยกรรม e-commerce บน AWS |

---

## Step 931: Amazon RDS for PostgreSQL คืออะไร

### 931.1 นิยามของ Managed Database Service

**Amazon Relational Database Service (RDS)** คือบริการฐานข้อมูลเชิงสัมพันธ์แบบ **Managed** ของ AWS ที่รองรับ PostgreSQL เป็นหนึ่งใน database engine ที่เลือกใช้ได้ (นอกเหนือจาก MySQL, MariaDB, Oracle, SQL Server และ Aurora)

คำว่า "Managed" ในที่นี้หมายถึง AWS รับผิดชอบงานเชิงปฏิบัติการ (operational overhead) ที่ DBA ต้องทำเองในสภาพแวดล้อม self-managed แทนเรา โดยเราไม่มีสิทธิ์ `SSH` เข้าไปที่ตัวเครื่อง OS ที่รัน PostgreSQL ได้เลย — ทุกอย่างต้องทำผ่าน AWS API / Console / CLI

```
┌─────────────────────────────────────────────────────────────────┐
│                     ผู้ใช้งาน (Customer) รับผิดชอบ                  │
├─────────────────────────────────────────────────────────────────┤
│  - Schema Design, Query Optimization                              │
│  - Application-level Connection Pooling                           │
│  - Data (ความถูกต้อง, การเข้ารหัสระดับ column)                     │
│  - IAM Policy, Security Group Rules                                │
│  - เลือก Instance Class, Storage Type, Parameter Group             │
│  - Backup Retention Policy (กำหนดค่า ไม่ใช่ลงมือทำเอง)              │
├─────────────────────────────────────────────────────────────────┤
│                        AWS RDS จัดการให้                          │
├─────────────────────────────────────────────────────────────────┤
│  - OS Patching (Amazon Linux ใต้ RDS)                              │
│  - PostgreSQL Minor Version Patching                               │
│  - Automated Backup (Snapshot รายวัน + WAL Archiving)               │
│  - Point-in-Time Recovery Infrastructure                          │
│  - Multi-AZ Failover Orchestration                                │
│  - Hardware Provisioning, การ replace disk/node ที่เสีย            │
│  - Monitoring Infrastructure (CloudWatch, Enhanced Monitoring)     │
│  - Storage Scaling (เมื่อเปิด Storage Autoscaling)                  │
└─────────────────────────────────────────────────────────────────┘
```

### 931.2 สิ่งที่ AWS จัดการให้ (รายละเอียด)

| หมวด | สิ่งที่ AWS ทำให้อัตโนมัติ | ความถี่ / เงื่อนไข |
|------|---------------------------|---------------------|
| **OS Patching** | ปะแก้ kernel, security patch ของ underlying Linux | ตาม Maintenance Window ที่ตั้งไว้ |
| **Engine Patching** | อัปเดต PostgreSQL minor version (เช่น 16.3 → 16.4) | Auto minor version upgrade (เปิด/ปิดได้) |
| **Backup** | Automated snapshot รายวัน + continuous WAL archive ไปยัง S3 | ทุกวันตาม backup window, WAL ทุก 5 นาที |
| **Failover (Multi-AZ)** | ตรวจจับ failure และสลับไป standby อัตโนมัติ | ภายใน 60–120 วินาทีโดยทั่วไป |
| **Hardware Replacement** | เปลี่ยน physical host/disk เมื่อ hardware เสีย | อัตโนมัติ ไม่ต้องแจ้ง (Multi-AZ ลด downtime) |
| **Storage Scaling** | ขยาย storage เมื่อใกล้เต็ม (เมื่อเปิด autoscaling) | ตาม threshold ที่ตั้ง |
| **Monitoring** | CloudWatch metrics, Enhanced Monitoring (OS-level), Performance Insights | Real-time |

### 931.3 สิ่งที่ AWS **ไม่** จัดการให้

- **ไม่มี superuser OS access** — ไม่มีสิทธิ์ `sudo`, ไม่สามารถติดตั้ง extension ที่ไม่อยู่ใน allowlist ของ RDS ได้ (ต้องเช็ค `SHOW rds.extensions;` หรือ AWS documentation)
- **ไม่มีสิทธิ์เข้าถึง filesystem โดยตรง** — ไม่สามารถอ่าน/เขียนไฟล์บน disk ของ PostgreSQL data directory ได้เลย ทุกอย่างต้องผ่าน SQL หรือ RDS API
- **Major version upgrade** ต้องทำเองผ่านคำสั่ง (ไม่ auto) เพราะกระทบ compatibility ของ application
- **Query tuning, index design, schema migration** ยังเป็นหน้าที่ DBA/Developer เหมือนเดิม
- **การจัดการ connection pool ระดับ application** (เช่น PgBouncer) — RDS ไม่มี built-in connection pooler แบบเดียวกับ Aurora ที่มี RDS Proxy แยกต่างหาก (ต้องเปิดใช้เพิ่มเติม)

### 931.4 เปรียบเทียบ Self-Managed vs RDS

| มิติ | Self-Managed (บน EC2 หรือ On-Premise) | Amazon RDS for PostgreSQL |
|------|---------------------------------------|---------------------------|
| การติดตั้ง | ต้องติดตั้งเอง (apt/yum, compile) | Provision ผ่าน API/Console ภายในไม่กี่นาที |
| Patching | DBA ต้องวางแผนและทำเอง (ตามที่เรียนใน Part เรื่อง version upgrade) | AWS ทำให้ตาม maintenance window |
| High Availability | ต้องตั้ง Patroni + etcd/Consul เอง (Part 065) | เปิด Multi-AZ checkbox/flag เดียว |
| Backup | ต้องตั้ง pgBackRest/Barman เอง (Part 062) | Automated backup built-in |
| Extension ที่ใช้ได้ | ติดตั้งอะไรก็ได้ (full control) | จำกัดตาม RDS-supported extension list |
| Superuser | มี full `rds_superuser`-เทียบเท่า OS root | ไม่มี OS access, มีแค่ `rds_superuser` role (จำกัดกว่า native superuser) |
| ต้นทุนบุคลากร | ต้องมี DBA ดูแล infrastructure เต็มเวลา | ลด operational overhead ได้มาก |
| ความยืดหยุ่นการปรับแต่ง kernel/filesystem | ปรับได้เต็มที่ (เช่น `vm.overcommit_memory`, filesystem tuning) | ปรับได้เฉพาะผ่าน Parameter Group เท่านั้น |
| ต้นทุน compute/storage โดยตรง | อาจถูกกว่าถ้ามีทีมเก่งจัดการเอง | มี premium cost เทียบกับ raw EC2 แต่แลกกับ operational safety |

### 931.5 ตรวจสอบ Engine version ที่รองรับด้วย AWS CLI

```bash
# ดูรายการ PostgreSQL engine version ที่ RDS รองรับใน region ปัจจุบัน
aws rds describe-db-engine-versions \
    --engine postgres \
    --query "DBEngineVersions[].{Version:EngineVersion,Status:Status}" \
    --output table

# ดู extension ที่รองรับสำหรับ engine version หนึ่ง ๆ
aws rds describe-db-engine-versions \
    --engine postgres \
    --engine-version 16.4 \
    --query "DBEngineVersions[0].SupportedFeatureNames"
```

### 931.6 สร้าง RDS PostgreSQL Instance แรกด้วย AWS CLI

```bash
aws rds create-db-instance \
    --db-instance-identifier mycompany-prod-pg \
    --db-instance-class db.r6g.large \
    --engine postgres \
    --engine-version 16.4 \
    --master-username pgadmin \
    --master-user-password 'ChangeMe_UseSecretsManagerInstead!' \
    --allocated-storage 100 \
    --storage-type gp3 \
    --vpc-security-group-ids sg-0123456789abcdef0 \
    --db-subnet-group-name mycompany-db-subnet-group \
    --backup-retention-period 7 \
    --multi-az \
    --no-publicly-accessible \
    --storage-encrypted \
    --deletion-protection \
    --tags Key=Environment,Value=Production Key=Team,Value=Platform
```

> **หมายเหตุด้านความปลอดภัย**: ในทางปฏิบัติ **ห้าม** ฝังรหัสผ่านตรง ๆ ใน CLI command แบบนี้ ควรใช้ `--manage-master-user-password` เพื่อให้ RDS สร้างและเก็บ password ผ่าน **AWS Secrets Manager** ให้อัตโนมัติ:
>
> ```bash
> aws rds create-db-instance \
>     --db-instance-identifier mycompany-prod-pg \
>     --db-instance-class db.r6g.large \
>     --engine postgres \
>     --engine-version 16.4 \
>     --master-username pgadmin \
>     --manage-master-user-password \
>     --allocated-storage 100 \
>     --storage-type gp3 \
>     --multi-az \
>     --storage-encrypted
> ```

### 931.7 ตรวจสอบสถานะ instance

```bash
aws rds describe-db-instances \
    --db-instance-identifier mycompany-prod-pg \
    --query "DBInstances[0].{Status:DBInstanceStatus,Endpoint:Endpoint.Address,Port:Endpoint.Port,MultiAZ:MultiAZ}"

# รอจนกว่า instance จะพร้อมใช้งาน (available)
aws rds wait db-instance-available --db-instance-identifier mycompany-prod-pg
```

### 931.8 เมื่อไรควรใช้ RDS แทนการทำ self-managed

RDS เหมาะกับ:
- ทีมที่ไม่มี DBA เฉพาะทางเต็มเวลา หรือมี DBA แต่ต้องการลด operational load
- ระบบที่ต้องการ High Availability และ Disaster Recovery แบบ "เปิดใช้" ได้ทันทีโดยไม่ต้องสร้าง Patroni cluster เอง
- องค์กรที่ต้องผ่าน compliance audit ที่ต้องการ patching และ encryption ที่ตรวจสอบได้ง่าย

RDS อาจไม่เหมาะกับ:
- ระบบที่ต้องการ extension พิเศษที่ RDS ไม่รองรับ (เช่น extension บางตัวที่ต้องการ superuser เต็มรูปแบบ หรือ extension ที่ต้องแก้ระดับ OS)
- ระบบที่ต้อง tuning ระดับ kernel/filesystem ลึกมาก (เช่น การปรับ `io_uring`, custom filesystem)
- Workload ที่ cost-sensitive มากและมีทีมที่เก่งพอจะดูแล self-managed ได้อย่างมีประสิทธิภาพกว่า

---

## Step 932: RDS Instance Class และ Storage

### 932.1 Instance Class Family

RDS ใช้ตระกูล instance class เดียวกับ EC2 (มี suffix ต่างกันเล็กน้อย) แบ่งเป็นกลุ่มหลัก:

| Family | ลักษณะ | เหมาะกับ |
|--------|--------|----------|
| `db.t3` / `db.t4g` | Burstable performance, มี CPU credit | Dev/Test, workload เบา, ไม่ต่อเนื่อง |
| `db.m6g` / `db.m7g` | General purpose, balance CPU:RAM 1:4 | Workload ทั่วไปที่ไม่หนัก compute หรือ memory มากเป็นพิเศษ |
| `db.r6g` / `db.r7g` | Memory-optimized, balance CPU:RAM 1:8 | OLTP ที่ working set ใหญ่, ต้องการ cache buffer เยอะ |
| `db.x2g` | Memory-optimized ขนาดใหญ่มาก | In-memory analytics, workload ที่ต้องการ RAM มหาศาล |

> ตัวอักษร `g` ต่อท้าย (เช่น `db.r6g`) หมายถึงใช้ AWS Graviton (ARM-based processor) ซึ่งโดยทั่วไปให้ price-performance ดีกว่า Intel/AMD รุ่นเทียบเท่า ควรพิจารณาใช้ก่อนเสมอถ้า application ไม่มีข้อจำกัดเรื่อง architecture

### 932.2 หลักการเลือกขนาด Instance

```
ขั้นตอนการเลือก Instance Class:

1. ประเมิน working set size
   → SELECT pg_size_pretty(sum(pg_relation_size(oid))) FROM pg_class;
   → working set ควร fit ใน shared_buffers + OS page cache ได้เกือบหมด
     เพื่อลด disk I/O

2. ประเมิน concurrent connections ที่ต้องการ
   → connections มาก = ต้องการ RAM มากขึ้น (แต่ละ connection ใช้ ~5-10MB)
   → พิจารณาใช้ RDS Proxy หรือ PgBouncer แทนการเปิด instance ใหญ่เกินจำเป็น

3. ประเมิน CPU-bound หรือ Memory-bound
   → Query ซับซ้อน, sort/aggregate เยอะ = CPU-bound → m-family อาจพอ
   → Working set ใหญ่, ต้องการ cache hit ratio สูง = Memory-bound → r-family

4. เริ่มจากขนาดกลาง แล้ว monitor + scale ภายหลัง
   → RDS รองรับการเปลี่ยน instance class แบบ near-zero-downtime
     ถ้าเป็น Multi-AZ (AWS จะ patch standby ก่อนแล้ว failover)
```

### 932.3 ตารางอ้างอิง Instance Class (ตัวอย่าง)

| Instance Class | vCPU | RAM (GiB) | Network Performance | เหมาะกับ |
|----------------|------|-----------|----------------------|----------|
| `db.t4g.medium` | 2 | 4 | Up to 5 Gbps | Dev/Test, staging |
| `db.t4g.large` | 2 | 8 | Up to 5 Gbps | Dev/Test ขนาดกลาง |
| `db.m6g.large` | 2 | 8 | Up to 10 Gbps | Production ขนาดเล็ก-กลาง |
| `db.m6g.xlarge` | 4 | 16 | Up to 10 Gbps | Production ขนาดกลาง |
| `db.r6g.large` | 2 | 16 | Up to 10 Gbps | OLTP ที่ต้องการ cache เยอะ |
| `db.r6g.xlarge` | 4 | 32 | Up to 10 Gbps | OLTP/Analytics ขนาดกลาง-ใหญ่ |
| `db.r6g.2xlarge` | 8 | 64 | Up to 10 Gbps | Production ขนาดใหญ่ |
| `db.r6g.4xlarge` | 16 | 128 | 10 Gbps | Production ขนาดใหญ่มาก |

> ตัวเลขข้างต้นเป็นตัวอย่างอ้างอิงโดยประมาณ ควรตรวจสอบ spec ล่าสุดจาก AWS documentation เสมอ เนื่องจาก AWS เพิ่ม instance type ใหม่อยู่ตลอดเวลา

### 932.4 Storage Type: gp3 vs io2 vs io2 Block Express

| คุณสมบัติ | gp3 (General Purpose SSD) | io2 (Provisioned IOPS SSD) | io2 Block Express |
|-----------|---------------------------|------------------------------|---------------------|
| IOPS พื้นฐาน | 3,000 IOPS (รวมในราคา) | กำหนดเองได้ตั้งแต่ 100 | กำหนดเองได้สูงสุดมากกว่า io2 ปกติ |
| Throughput พื้นฐาน | 125 MiB/s (รวมในราคา) | ขึ้นกับ IOPS ที่ตั้ง | สูงกว่า io2 ปกติ |
| การปรับ IOPS/Throughput แยกจากขนาด storage | ทำได้ (ปรับได้อิสระในระดับหนึ่ง) | IOPS ปรับแยกได้ แต่ throughput ผูกกับ IOPS | ปรับได้อิสระมากกว่า |
| Durability | 99.8–99.9% annual | 99.999% annual | 99.999% annual |
| ความเหมาะสม | Workload ทั่วไป, cost-effective | Workload ที่ต้องการ IOPS สูงสม่ำเสมอ (มากกว่า 16,000 IOPS) | Workload ระดับ mission-critical ที่ต้องการ IOPS สูงสุดขีด |
| ราคา | ถูกกว่า | แพงกว่า (คิดตาม provisioned IOPS) | แพงที่สุด |

**คำแนะนำเชิงปฏิบัติ**:
- เริ่มต้นด้วย **gp3** เสมอ เว้นแต่มีการวัดผลแล้วว่า workload ต้องการ IOPS สูงกว่าที่ gp3 ให้ได้อย่างสม่ำเสมอ (16,000 IOPS max บน gp3 สำหรับ RDS)
- ใช้ **io2** เมื่อ workload เป็น OLTP ที่มี IOPS สูงมากและ **sensitivity ต่อ latency สูง** เช่น ระบบ payment/trading
- gp3 ให้ความยืดหยุ่นในการปรับ IOPS และ Throughput แยกจากขนาด storage ได้โดยไม่ต้องเพิ่มขนาด disk (ต่างจาก gp2 รุ่นเก่าที่ IOPS ผูกกับขนาด storage โดยตรง)

### 932.5 คำสั่งปรับแต่ง Storage

```bash
# ตรวจสอบ storage configuration ปัจจุบัน
aws rds describe-db-instances \
    --db-instance-identifier mycompany-prod-pg \
    --query "DBInstances[0].{StorageType:StorageType,AllocatedStorage:AllocatedStorage,IOPS:Iops,Throughput:StorageThroughput}"

# ปรับ storage เป็น gp3 พร้อมกำหนด IOPS และ throughput เอง (ต้องมากกว่า baseline)
aws rds modify-db-instance \
    --db-instance-identifier mycompany-prod-pg \
    --storage-type gp3 \
    --iops 6000 \
    --storage-throughput 250 \
    --apply-immediately
```

### 932.6 Storage Autoscaling

Storage Autoscaling ช่วยให้ RDS ขยายขนาด storage อัตโนมัติเมื่อพื้นที่ใกล้เต็ม โดยไม่ต้อง downtime

```bash
aws rds modify-db-instance \
    --db-instance-identifier mycompany-prod-pg \
    --max-allocated-storage 500 \
    --apply-immediately
```

กลไกการทำงาน:
- เมื่อ free storage เหลือน้อยกว่า **10%** ของ allocated storage และสถานการณ์นี้ต่อเนื่องอย่างน้อย **5 นาที** RDS จะเริ่มขยาย storage อัตโนมัติ
- ขยายทีละ "ก้อน" (increment) ตาม storage ปัจจุบันจนถึงเพดาน `max-allocated-storage` ที่ตั้งไว้
- มีระยะเวลา cooldown ระหว่างการขยายแต่ละครั้ง (ประมาณ 6 ชั่วโมง) เพื่อป้องกันการขยายถี่เกินไปจากค่าใช้จ่ายที่ไม่คาดคิด

> **ข้อควรระวัง**: ควรตั้งค่า **CloudWatch Alarm** บน metric `FreeStorageSpace` ควบคู่ไปด้วยเสมอ เพราะ autoscaling ไม่ใช่ "ไม่มีเพดาน" — ถ้าข้อมูลโตเร็วกว่าที่คาด อาจแตะเพดาน `max-allocated-storage` และเกิด storage full ได้

```bash
aws cloudwatch put-metric-alarm \
    --alarm-name "RDS-FreeStorageSpace-Low" \
    --namespace AWS/RDS \
    --metric-name FreeStorageSpace \
    --dimensions Name=DBInstanceIdentifier,Value=mycompany-prod-pg \
    --statistic Average \
    --period 300 \
    --threshold 5368709120 \
    --comparison-operator LessThanThreshold \
    --evaluation-periods 2 \
    --alarm-actions arn:aws:sns:ap-southeast-1:123456789012:dba-alerts
```

---

## Step 933: RDS Multi-AZ Deployment

### 933.1 สถาปัตยกรรม Multi-AZ (Instance-based)

RDS Multi-AZ แบบดั้งเดิม (Multi-AZ DB Instance) ทำงานโดยมี **standby replica** แบบ synchronous ในอีก Availability Zone (AZ) หนึ่ง โดยใช้กลไก **physical streaming replication ภายในที่ AWS จัดการให้ทั้งหมด** — เราไม่เห็นและไม่ต้องตั้งค่า replication เอง

```
                    Region: ap-southeast-1
   ┌───────────────────────┐        ┌───────────────────────┐
   │   AZ: ap-southeast-1a │        │   AZ: ap-southeast-1b │
   │                       │        │                       │
   │  ┌─────────────────┐  │        │  ┌─────────────────┐  │
   │  │  Primary (RW)    │──┼─sync───┼─▶│  Standby         │  │
   │  │  PostgreSQL      │  │ replic-│  │  PostgreSQL      │  │
   │  │                  │  │ ation  │  │  (ไม่รับ query    │  │
   │  └─────────────────┘  │        │  │   จาก client)     │  │
   │          │             │        │  └─────────────────┘  │
   └──────────┼─────────────┘        └───────────────────────┘
              │
      DNS Endpoint (CNAME)
   mycompany-prod-pg.xxxx.ap-southeast-1.rds.amazonaws.com
              │
              ▼
     ┌─────────────────┐
     │   Application     │   เมื่อ failover เกิดขึ้น DNS จะชี้ไป
     │   (เชื่อมต่อผ่าน    │   standby ตัวใหม่โดยอัตโนมัติ
     │    endpoint เดียว) │
     └─────────────────┘
```

### 933.2 จุดสำคัญของ Multi-AZ DB Instance

- **Standby ไม่รับ read query** — ต่างจาก Read Replica standby ใน Multi-AZ ไม่สามารถใช้เพื่อ offload read traffic ได้ (มีไว้เพื่อ HA เท่านั้น) — ถ้าต้องการ read scaling ต้องใช้ Read Replica แยกต่างหาก (Step 934)
- **Synchronous replication** — transaction จะ commit ก็ต่อเมื่อ WAL ถูกเขียนไปที่ standby แล้ว (ทำให้ RPO = 0 ในกรณีปกติ)
- **Automatic failover** — เมื่อ AWS ตรวจพบว่า primary มีปัญหา (เช่น AZ ล่ม, instance ล่ม, storage ล่ม) จะ promote standby ขึ้นเป็น primary อัตโนมัติ และเปลี่ยน DNS CNAME ให้ endpoint เดิมชี้ไปที่ instance ใหม่
- **Failover time** โดยทั่วไปอยู่ที่ **60-120 วินาที** (ขึ้นกับปัจจัย เช่น ต้องมีการ replay transaction log ก่อน)
- **Backup ทำที่ standby** — Multi-AZ ช่วยลด I/O suspension ระหว่าง backup เพราะ snapshot จะทำที่ standby แทน primary

### 933.3 Multi-AZ DB Cluster (รุ่นใหม่ — 2 readable standby)

AWS มี option ใหม่กว่าคือ **Multi-AZ DB Cluster** ซึ่งต่างจาก Multi-AZ DB Instance ตรงที่:

```
┌───────────────┐   ┌───────────────┐   ┌───────────────┐
│  AZ-a          │   │  AZ-b          │   │  AZ-c          │
│  Writer (RW)   │   │  Reader (RO)   │   │  Reader (RO)   │
│  Instance      │──▶│  Instance      │   │  Instance      │
│                │   │  (รับ read      │   │  (รับ read      │
│                │   │   query ได้)    │   │   query ได้)    │
└───────────────┘   └───────────────┘   └───────────────┘
        │  quorum-based semi-synchronous replication (2 จาก 3)
        ▼
   Writer Endpoint          Reader Endpoint (load balance ระหว่าง 2 readers)
```

| คุณสมบัติ | Multi-AZ DB **Instance** | Multi-AZ DB **Cluster** |
|-----------|---------------------------|----------------------------|
| จำนวน standby | 1 (ไม่ readable) | 2 (readable ทั้งคู่) |
| Replication mode | Fully synchronous | Semi-synchronous (quorum: primary + 1 ใน 2 reader) |
| Failover time | ~60-120 วินาที | เร็วกว่า ~ไม่ถึง 35 วินาทีโดยทั่วไป |
| Read scaling | ไม่มี (ต้องใช้ Read Replica แยก) | มี built-in ผ่าน Reader Endpoint |
| ราคา | Compute x2 (primary+standby) | Compute x3 |
| Storage | EBS แบบดั้งเดิม | EBS แบบดั้งเดิมเช่นกัน (คนละแบบกับ Aurora storage) |

### 933.4 เปรียบเทียบ RDS Multi-AZ กับ Patroni Cluster (Part 065)

| มิติ | Patroni + etcd/Consul (Self-Managed, Part 065) | RDS Multi-AZ (Managed) |
|------|--------------------------------------------------|---------------------------|
| ผู้ดูแล consensus/DCS | ทีมต้องติดตั้งและดูแล etcd/Consul/ZooKeeper เอง | AWS จัดการ internal mechanism ให้ทั้งหมด (ไม่เปิดเผยรายละเอียด) |
| การตั้งค่า failover threshold | ปรับ `ttl`, `loop_wait`, `retry_timeout` ได้ละเอียด | ควบคุมได้จำกัด (ผ่าน parameter บางตัวเท่านั้น) |
| Failover time | ปรับแต่งได้ให้เร็วมาก (วินาทีเดียวในบางเคส) | คงที่ตาม AWS engine (~60-120s หรือเร็วกว่าสำหรับ Cluster mode) |
| Custom fencing/STONITH | ทำได้เต็มที่ | ไม่สามารถเข้าถึง/ปรับแต่งได้เลย |
| ความรู้ที่ทีมต้องมี | ต้องเข้าใจ Patroni, DCS, PostgreSQL replication ลึกซึ้ง | ต้องเข้าใจ AWS RDS concept และ SQL เท่านั้น |
| ความยืดหยุ่นในการ deploy หลาย cloud/on-prem | ทำได้ (portable) | ผูกกับ AWS เท่านั้น (vendor lock-in) |
| Load Balancer ด้านหน้า | ต้องตั้งเอง (HAProxy/pgbouncer + Patroni REST API) | DNS CNAME แบบ built-in |

### 933.5 คำสั่งเปิด Multi-AZ

```bash
# เปิด Multi-AZ ให้ instance ที่มีอยู่แล้ว (จะมี brief outage ตอนสลับ)
aws rds modify-db-instance \
    --db-instance-identifier mycompany-prod-pg \
    --multi-az \
    --apply-immediately

# ทดสอบ failover ด้วยตนเอง (สำหรับ DR drill)
aws rds reboot-db-instance \
    --db-instance-identifier mycompany-prod-pg \
    --force-failover

# ตรวจสอบว่า instance ไหนเป็น writer / AZ ปัจจุบัน
aws rds describe-db-instances \
    --db-instance-identifier mycompany-prod-pg \
    --query "DBInstances[0].{AZ:AvailabilityZone,MultiAZ:MultiAZ,SecondaryAZ:SecondaryAvailabilityZone}"
```

> **แนวทางปฏิบัติที่ดี**: ควรทำ **DR Drill (`reboot-db-instance --force-failover`)** อย่างสม่ำเสมอในสภาพแวดล้อม non-production เพื่อยืนยันว่า application มี retry logic และ connection handling ที่รองรับ failover ได้จริง ไม่ใช่แค่ตั้งค่า Multi-AZ แล้ววางใจ

---

## Step 934: RDS Read Replica

### 934.1 หลักการทำงาน

Read Replica ใช้ **asynchronous physical streaming replication** (แตกต่างจาก Multi-AZ ที่เป็น synchronous) เพื่อสร้างสำเนาฐานข้อมูลที่ **รับ read query ได้จริง** ช่วย offload read traffic ออกจาก primary

```
                  Primary Region: ap-southeast-1
   ┌─────────────────────┐
   │  Primary (RW)         │
   │  mycompany-prod-pg    │
   └──────────┬────────────┘
              │ async replication (streaming WAL)
     ┌────────┼────────────────┐
     ▼        ▼                ▼
┌─────────┐┌─────────┐  ┌───────────────────────┐
│Replica-1 ││Replica-2 │  │  Cross-Region Replica   │
│(same     ││(same     │  │  Region: ap-northeast-1 │
│ region)  ││ region)  │  │  (สำหรับ DR / latency   │
│read-only ││read-only │  │   ลด สำหรับผู้ใช้ญี่ปุ่น)  │
└─────────┘└─────────┘  └───────────────────────┘
```

### 934.2 สร้าง Read Replica (Same-Region)

```bash
aws rds create-db-instance-read-replica \
    --db-instance-identifier mycompany-prod-pg-replica-1 \
    --source-db-instance-identifier mycompany-prod-pg \
    --db-instance-class db.r6g.large \
    --no-publicly-accessible \
    --tags Key=Purpose,Value=ReadScaling
```

### 934.3 สร้าง Read Replica ข้าม Region (Cross-Region)

```bash
# ต้อง reference ARN เต็มของ source instance (ไม่ใช่แค่ identifier)
aws rds create-db-instance-read-replica \
    --db-instance-identifier mycompany-prod-pg-replica-tokyo \
    --source-db-instance-identifier arn:aws:rds:ap-southeast-1:123456789012:db:mycompany-prod-pg \
    --db-instance-class db.r6g.large \
    --region ap-northeast-1 \
    --db-subnet-group-name mycompany-tokyo-subnet-group \
    --no-publicly-accessible \
    --kms-key-id arn:aws:kms:ap-northeast-1:123456789012:key/abcd-efgh-ijkl
```

> **ข้อควรทราบ**: ถ้า source instance เข้ารหัส (`storage-encrypted`) การสร้าง cross-region replica ต้องระบุ `--kms-key-id` ของ region ปลายทางด้วย เพราะ KMS key เป็นแบบ region-specific

### 934.4 Cross-Region Replication Latency

| ปัจจัย | ผลกระทบต่อ Replication Lag |
|--------|------------------------------|
| ระยะทางทาง network ระหว่าง region | ยิ่งไกล ยิ่ง latency สูง (เช่น Singapore ↔ Tokyo ~ 60-80ms RTT, Singapore ↔ US ~ 200ms+ RTT) |
| ปริมาณ WAL ที่ต้อง generate (write-heavy workload) | Workload ที่มี write เยอะ (bulk insert, batch job) → lag เพิ่มขึ้นชัดเจน |
| Instance class ของ replica | Replica ที่เล็กเกินไปอาจ apply WAL ไม่ทัน primary |
| Network bandwidth ระหว่าง region | AWS ใช้ backbone network ของตัวเอง แต่ยังมีข้อจำกัดตาม throughput |
| Long-running transaction บน replica | Query ที่ query นาน ๆ บน replica อาจถูก cancel ด้วย `max_standby_streaming_delay` หรือทำให้ lag เพิ่ม |

**วิธี monitor replication lag**:

```bash
aws cloudwatch get-metric-statistics \
    --namespace AWS/RDS \
    --metric-name ReplicaLag \
    --dimensions Name=DBInstanceIdentifier,Value=mycompany-prod-pg-replica-tokyo \
    --start-time 2026-09-25T00:00:00Z \
    --end-time 2026-09-25T06:00:00Z \
    --period 300 \
    --statistics Average Maximum
```

หรือตรวจสอบผ่าน SQL โดยตรงบน replica:

```sql
-- รันบน replica เพื่อดู lag เป็นวินาที
SELECT now() - pg_last_xact_replay_timestamp() AS replication_lag;
```

### 934.5 ข้อจำกัดของ Read Replica ที่ต้องรู้

- **Read-after-write inconsistency**: เนื่องจากเป็น async replication แอปพลิเคชันที่เขียนที่ primary แล้วอ่านทันทีที่ replica อาจไม่เห็นข้อมูลล่าสุด (eventual consistency) ต้องออกแบบ application ให้รองรับ (เช่น อ่านจาก primary สำหรับ flow ที่ต้องการ strong consistency)
- **Cross-region replica ไม่สามารถ promote เป็น Multi-AZ ได้ทันที** — ต้อง promote เป็น standalone instance ก่อน แล้วค่อยเปิด Multi-AZ ทีหลัง
- **จำนวน replica สูงสุด**: RDS PostgreSQL รองรับ read replica ได้สูงสุด 15 ตัวต่อ source instance (รวมทั้ง same-region และ cross-region) — ตรวจสอบ soft limit ปัจจุบันเสมอเพราะ AWS อาจปรับเปลี่ยน
- **Promote replica เป็น standalone**:

```bash
aws rds promote-read-replica \
    --db-instance-identifier mycompany-prod-pg-replica-tokyo \
    --backup-retention-period 7
```

### 934.6 Use Case ของ Cross-Region Replica

1. **Disaster Recovery (DR)**: หาก region หลักล่มทั้ง region (ไม่ใช่แค่ AZ) สามารถ promote cross-region replica ขึ้นเป็น primary ใหม่ได้
2. **Latency reduction สำหรับผู้ใช้ต่างภูมิภาค**: ผู้ใช้ในญี่ปุ่นอ่านข้อมูลจาก replica ที่ Tokyo แทนที่จะยิง query ข้าม region ไปที่ Singapore
3. **Data migration**: ใช้เป็นขั้นตอนกลางในการย้ายฐานข้อมูลไป region ใหม่แบบ near-zero-downtime

---

## Step 935: RDS Backup และ Snapshot

### 935.1 Automated Backup

RDS ทำ Automated Backup โดยอัตโนมัติเมื่อเปิด `backup-retention-period` มากกว่า 0 ประกอบด้วย:

1. **Full daily snapshot** ในช่วง backup window ที่กำหนด (หรือ AWS เลือกให้อัตโนมัติถ้าไม่ระบุ)
2. **Continuous transaction log (WAL) backup** ไปยัง Amazon S3 (จัดการโดย AWS ทั้งหมด ไม่ต้องตั้งค่าเหมือน `archive_command` ใน self-managed)

```
Backup Retention Window (เช่น 7 วัน)
├── Day 0: Full Snapshot (auto, ระหว่าง backup window)
├── Day 0 → Day 7: WAL continuous archiving ทุก ๆ ~5 นาที
├── Day 1: Full Snapshot
├── Day 2: Full Snapshot
├── ...
└── สามารถ Restore ไปยังจุดเวลาใดก็ได้ภายใน window นี้ (granularity ~5 นาที)
```

### 935.2 ตั้งค่า Backup Retention

```bash
aws rds modify-db-instance \
    --db-instance-identifier mycompany-prod-pg \
    --backup-retention-period 14 \
    --preferred-backup-window "17:00-17:30" \
    --preferred-maintenance-window "sun:18:00-sun:19:00" \
    --apply-immediately
```

> **ข้อควรทราบ**: `backup-retention-period = 0` จะ **ปิด automated backup ทั้งหมด** (รวมถึง PITR) — ห้ามตั้งเป็น 0 สำหรับ production เด็ดขาด

### 935.3 Manual Snapshot vs Automated Backup

| คุณสมบัติ | Automated Backup | Manual Snapshot |
|-----------|-------------------|-------------------|
| การสร้าง | อัตโนมัติตาม retention period | ผู้ใช้สั่งสร้างเอง (`create-db-snapshot`) |
| การลบเมื่อลบ instance | ถูกลบไปพร้อมกัน (เว้นแต่ Final Snapshot) | **คงอยู่ถาวร** จนกว่าจะลบเอง |
| อายุการเก็บ | ตาม retention period (สูงสุด 35 วัน) | ไม่จำกัดอายุ (เก็บได้ตลอดจนกว่าจะลบ) |
| ใช้สำหรับ | PITR (restore ไปจุดเวลาใดก็ได้) | Restore ไปจุดที่ snapshot ถูกสร้างเท่านั้น, หรือใช้ archive ระยะยาว |
| ค่าใช้จ่าย | รวมอยู่ในค่า storage backup (ฟรีถ้าไม่เกิน allocated storage) | คิดตามขนาด snapshot จริง |

### 935.4 สร้าง Manual Snapshot

```bash
aws rds create-db-snapshot \
    --db-instance-identifier mycompany-prod-pg \
    --db-snapshot-identifier mycompany-prod-pg-before-major-upgrade-20260925

aws rds wait db-snapshot-available \
    --db-snapshot-identifier mycompany-prod-pg-before-major-upgrade-20260925
```

### 935.5 Point-in-Time Recovery (PITR) แบบ Managed

```bash
# Restore ไปยังจุดเวลาที่ระบุ (สร้างเป็น instance ใหม่เสมอ ไม่ overwrite ของเดิม)
aws rds restore-db-instance-to-point-in-time \
    --source-db-instance-identifier mycompany-prod-pg \
    --target-db-instance-identifier mycompany-prod-pg-restored-20260925 \
    --restore-time "2026-09-25T14:30:00Z" \
    --db-instance-class db.r6g.large \
    --no-publicly-accessible \
    --db-subnet-group-name mycompany-db-subnet-group \
    --vpc-security-group-ids sg-0123456789abcdef0

# หรือ restore ไปยังจุดล่าสุดที่เป็นไปได้ (latest restorable time)
aws rds describe-db-instances \
    --db-instance-identifier mycompany-prod-pg \
    --query "DBInstances[0].LatestRestorableTime"

aws rds restore-db-instance-to-point-in-time \
    --source-db-instance-identifier mycompany-prod-pg \
    --target-db-instance-identifier mycompany-prod-pg-restored-latest \
    --use-latest-restorable-time
```

### 935.6 เปรียบเทียบ PITR แบบ Managed vs การทำเองด้วย pgBackRest/Barman (Part 062)

| มิติ | pgBackRest / Barman (Self-Managed, Part 062) | RDS PITR (Managed) |
|------|-------------------------------------------------|------------------------|
| การตั้งค่าเริ่มต้น | ต้องตั้ง `archive_command`, repository, retention policy เอง | เปิด flag เดียว (`backup-retention-period > 0`) |
| Storage backend | เลือกได้เอง (S3, local disk, NFS, Azure Blob ฯลฯ) | S3 เท่านั้น (จัดการโดย AWS, มองไม่เห็นโดยตรง) |
| Granularity ของ PITR | ปรับ WAL segment size / archive timeout ได้ละเอียด | ประมาณ 5 นาที (ตาม AWS internal) |
| Restore กลับที่เดิม (in-place) | ทำได้ (`pgbackrest restore`) | **ทำไม่ได้** — ต้อง restore เป็น instance ใหม่เสมอ แล้วค่อยสลับ endpoint/DNS เอง |
| การทดสอบ restore (DR drill) | ต้อง script เอง | สั่ง `restore-db-instance-to-point-in-time` ได้ตรงไปตรงมา แต่ต้องรอ instance ใหม่ provision (นาที) |
| Full/Diff/Incremental backup control | ควบคุมได้ละเอียด (เช่น incremental ทุกวัน, full ทุกสัปดาห์) | ไม่มีการควบคุมระดับนั้น เป็น black-box |
| ค่าใช้จ่ายเพิ่มเติม | ค่า storage ของ backup repository (S3 เอง) | รวมในราคา RDS (backup storage เท่ากับ allocated storage แรกฟรี ส่วนเกินคิดเพิ่ม) |
| Cross-region backup | ตั้งเองได้ (เช่น S3 replication) | ต้องใช้ `copy-db-snapshot` ข้าม region แยกต่างหาก |

### 935.7 Copy Snapshot ข้าม Region (สำหรับ DR)

```bash
aws rds copy-db-snapshot \
    --source-db-snapshot-identifier arn:aws:rds:ap-southeast-1:123456789012:snapshot:mycompany-prod-pg-before-major-upgrade-20260925 \
    --target-db-snapshot-identifier mycompany-prod-pg-dr-copy-20260925 \
    --region ap-northeast-1 \
    --kms-key-id arn:aws:kms:ap-northeast-1:123456789012:key/abcd-efgh-ijkl
```

### 935.8 Final Snapshot เมื่อลบ Instance

```bash
# เมื่อลบ instance ควรสร้าง final snapshot เสมอ (ยกเว้นตั้งใจลบถาวรจริง ๆ)
aws rds delete-db-instance \
    --db-instance-identifier mycompany-old-staging-pg \
    --final-db-snapshot-identifier mycompany-old-staging-pg-final-20260925 \
    --no-skip-final-snapshot
```

---

## Step 936: Amazon Aurora PostgreSQL คืออะไร

### 936.1 ความแตกต่างเชิงสถาปัตยกรรมจาก RDS ธรรมดา

จุดที่ทำให้ Aurora แตกต่างจาก RDS for PostgreSQL แบบ "ธรรมดา" อย่างสิ้นเชิงคือ **การแยก compute ออกจาก storage (storage-compute separation)**

**RDS PostgreSQL แบบดั้งเดิม**: storage คือ EBS volume ที่ผูกติดกับ instance เดียว (แม้ Multi-AZ จะ replicate ไปยัง standby แต่ก็ยังเป็น EBS แยกก้อนที่ sync กันผ่าน database-level replication)

**Aurora PostgreSQL**: storage เป็น **distributed storage layer** แยกออกมาเป็นบริการของตัวเอง ที่ทำงานคนละชั้นจาก compute instance โดยสิ้นเชิง

```
┌─────────────────────────────────────────────────────────────┐
│                     Aurora Compute Layer                        │
│                                                                  │
│   ┌──────────────┐        ┌──────────────┐  ┌──────────────┐ │
│   │  Writer        │        │  Reader-1      │  │  Reader-2      │ │
│   │  Instance      │        │  Instance      │  │  Instance      │ │
│   │  (RW)          │        │  (RO)          │  │  (RO)          │ │
│   └───────┬────────┘        └───────┬────────┘  └───────┬────────┘ │
│           │  ส่งเฉพาะ WAL log record (ไม่ใช่ data page)         │
└───────────┼────────────────────────┼────────────────────┼─────┘
            │                        │                    │
            ▼                        ▼                    ▼
┌──────────────────────────────────────────────────────────────┐
│              Aurora Distributed Storage Layer (แยกอิสระ)         │
│                                                                  │
│   AZ-a              AZ-b              AZ-c                      │
│  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐│
│  │Copy 1   │ │Copy 2   │ │Copy 3   │ │Copy 4   │ │Copy 5   │ │Copy 6   ││
│  └────────┘ └────────┘ └────────┘ └────────┘ └────────┘ └────────┘│
│                                                                  │
│   รวม 6 สำเนาข้อมูล กระจายใน 3 Availability Zone (2 copy/AZ)       │
│   Write ถือว่าสำเร็จเมื่อเขียนครบ 4 จาก 6 copies (quorum write)    │
│   Read ถือว่าถูกต้องเมื่ออ่านครบ 3 จาก 6 copies (quorum read)      │
└──────────────────────────────────────────────────────────────┘
```

### 936.2 กลไก 6 Copies across 3 AZ

- ข้อมูลถูกแบ่งเป็น "protection groups" ขนาด 10GB เรียกว่า **segment**
- แต่ละ segment มี **6 สำเนา** กระจายอยู่ใน **3 Availability Zone** (AZ ละ 2 สำเนา)
- **Write Quorum**: การเขียนจะถือว่าสำเร็จเมื่อเขียนสำเร็จอย่างน้อย **4 จาก 6** สำเนา (4/6 quorum)
- **Read Quorum**: การอ่านจะถือว่าถูกต้องเมื่ออ่านได้อย่างน้อย **3 จาก 6** สำเนา (3/6 quorum)
- เนื่องจาก 4 + 3 > 6 การันตีว่า read จะเห็นข้อมูลที่เขียนล่าสุดเสมอ (quorum overlap guarantee)
- Storage สามารถทนการสูญเสีย **1 ทั้ง AZ + 1 สำเนาเพิ่มเติม** โดยไม่กระทบการเขียน (เพราะยังเหลือ 4/6) และทน **1 ทั้ง AZ ล่ม** โดยไม่กระทบการอ่าน (ยังเหลือ 4/6 > 3/6 ที่ต้องการ)

### 936.3 ผลลัพธ์เชิงพฤติกรรมจากสถาปัตยกรรมนี้

| ผลลัพธ์ | คำอธิบาย |
|---------|----------|
| **Storage auto-scaling แบบ transparent** | Aurora storage ขยายอัตโนมัติทีละ 10GB จนถึงสูงสุด 128TiB (ตัวเลขอาจเปลี่ยนตาม AWS) โดยไม่ต้องตั้งค่า max-allocated-storage เหมือน RDS |
| **Crash recovery เร็วมาก** | เพราะไม่ต้อง replay WAL ทั้งหมดแบบ traditional PostgreSQL — storage layer ทำ redo แบบ distributed และ instance สามารถ restart ได้เร็วในไม่กี่วินาที |
| **Replica lag ต่ำกว่ามาก** | เพราะ reader instance ไม่ต้อง apply data page เหมือน physical replication — อ่านจาก shared storage layer เดียวกัน แล้วรับเฉพาะ log record มา apply ใน buffer cache — lag มักอยู่ในหลัก **สิบมิลลิวินาที** เทียบกับ RDS Read Replica ที่อาจเป็นวินาทีถึงหลักสิบวินาที |
| **ไม่มี "storage I/O" แบบเดิมที่ผูกกับ instance เดียว** | Write I/O กระจายไปยัง distributed storage nodes จึงลด bottleneck จาก single EBS volume |
| **Backtrack (เฉพาะ Aurora MySQL เท่านั้น)** | ฟีเจอร์นี้ไม่รองรับใน Aurora PostgreSQL — ต้องใช้ PITR แบบปกติแทน |

### 936.4 เปรียบเทียบ Aurora PostgreSQL vs RDS PostgreSQL

| มิติ | RDS for PostgreSQL | Aurora PostgreSQL-Compatible |
|------|----------------------|-------------------------------|
| Storage architecture | ผูกกับ instance (EBS) | แยกอิสระ, distributed, shared ระหว่าง writer/reader |
| จำนวนสำเนาข้อมูล | 1 (หรือ 2 ถ้า Multi-AZ) | 6 (across 3 AZ) เสมอ |
| Max storage | จำกัดตาม storage type (เช่น 64TiB) | สูงถึง 128TiB (auto-scale, ไม่ต้องตั้งค่า) |
| จำนวน read replica สูงสุด | 15 | 15 (Aurora Replica) + สามารถผสม Aurora Global Database ได้ |
| Replica lag | เป็นวินาทีถึงหลักสิบวินาที (async physical replication) | โดยทั่วไปต่ำกว่า 100ms (shared storage) |
| Failover time | ~60-120 วินาที (Instance), เร็วกว่าถ้าเป็น Multi-AZ Cluster | โดยทั่วไป ~30 วินาทีหรือเร็วกว่า (promote reader เป็น writer, ไม่ต้อง restart storage) |
| Engine compatibility | PostgreSQL เวอร์ชันมาตรฐาน 100% | "PostgreSQL-Compatible" — อิง PostgreSQL แต่มี storage engine ของ Aurora เอง, บาง extension/feature อาจต่างหรือมี lag ในการรองรับเวอร์ชันใหม่ |
| ราคา compute | ต่ำกว่า (ต่อ instance-hour) | สูงกว่า RDS instance ที่ spec เท่ากันโดยทั่วไป (~20% premium เป็นตัวเลขอ้างอิงคร่าว ๆ) |
| ราคา storage | คิดตาม provisioned storage (GB-month) | คิดตาม storage ที่ใช้จริง (auto-scale) + I/O request (เว้นแต่เลือก I/O-Optimized) |
| Serverless option | ไม่มี | มี (Aurora Serverless v2 — Step 938) |
| Global Database (multi-region, sub-second replication) | ไม่มี (ต้องใช้ cross-region read replica แบบ async ธรรมดา) | มี (Aurora Global Database) |
| Backtrack / Fast Clone | ไม่มี | Fast Database Cloning มี (copy-on-write, สร้าง clone ได้ในไม่กี่นาทีไม่ว่าขนาดข้อมูลใหญ่แค่ไหน) |

### 936.5 สร้าง Aurora PostgreSQL Cluster ด้วย AWS CLI

การสร้าง Aurora ต้องสร้าง **2 ระดับ**: DB Cluster (ระดับ storage + logical grouping) และ DB Instance (ระดับ compute ที่ join เข้า cluster)

```bash
# Step 1: สร้าง Aurora Cluster (storage layer)
aws rds create-db-cluster \
    --db-cluster-identifier mycompany-aurora-prod \
    --engine aurora-postgresql \
    --engine-version 16.4 \
    --master-username pgadmin \
    --manage-master-user-password \
    --db-subnet-group-name mycompany-db-subnet-group \
    --vpc-security-group-ids sg-0123456789abcdef0 \
    --backup-retention-period 14 \
    --storage-encrypted \
    --deletion-protection \
    --tags Key=Environment,Value=Production

# Step 2: สร้าง Writer Instance (compute แรกที่ join เข้า cluster)
aws rds create-db-instance \
    --db-instance-identifier mycompany-aurora-prod-writer \
    --db-cluster-identifier mycompany-aurora-prod \
    --engine aurora-postgresql \
    --db-instance-class db.r6g.xlarge \
    --no-publicly-accessible

# Step 3: สร้าง Reader Instance เพิ่ม (optional, สำหรับ read scaling)
aws rds create-db-instance \
    --db-instance-identifier mycompany-aurora-prod-reader-1 \
    --db-cluster-identifier mycompany-aurora-prod \
    --engine aurora-postgresql \
    --db-instance-class db.r6g.xlarge \
    --no-publicly-accessible
```

### 936.6 เมื่อไรควรใช้ Aurora แทน RDS ธรรมดา

| เลือก RDS PostgreSQL เมื่อ | เลือก Aurora PostgreSQL เมื่อ |
|------------------------------|----------------------------------|
| Workload เล็ก-กลาง, budget จำกัด | Workload ต้องการ HA/DR ระดับสูงมาก และ budget รองรับ premium |
| ต้องการความเข้ากันได้กับ PostgreSQL มาตรฐาน 100% (extension เฉพาะทาง) | ต้องการ read scaling ที่มี replica lag ต่ำมาก (หลาย reader, lag <100ms) |
| Predictable steady workload ที่ประเมิน instance size ได้ชัดเจน | Workload ผันผวนสูง (เหมาะกับ Aurora Serverless v2 — Step 938) |
| ทีมคุ้นเคยกับ RDS มาก่อน ไม่ต้องการความซับซ้อนเพิ่ม | ต้องการ Global Database สำหรับ multi-region active-passive |
| Dev/Test/Staging ที่ cost-sensitive | ต้องการ fast cloning สำหรับสร้าง environment ทดสอบจากข้อมูล production ขนาดใหญ่ |

---

## Step 937: Aurora Reader/Writer Endpoint

### 937.1 ประเภทของ Endpoint ใน Aurora

Aurora มี endpoint หลายแบบที่ทำหน้าที่ต่างกัน ต่างจาก RDS ที่ปกติมี endpoint เดียวต่อ instance

```
┌─────────────────────────────────────────────────────────────┐
│                    Aurora DB Cluster                            │
│                                                                  │
│   Writer Instance          Reader-1          Reader-2           │
│   (RW, primary)            (RO)              (RO)               │
│         ▲                    ▲                  ▲               │
│         │                    │                  │               │
│  ┌──────┴──────┐    ┌────────┴──────────────────┴────────┐     │
│  │  Cluster       │    │  Reader Endpoint                     │     │
│  │  (Writer)      │    │  (Load-balance ระหว่าง readers        │     │
│  │  Endpoint      │    │   ทั้งหมด, DNS round-robin)            │     │
│  └────────────────┘    └─────────────────────────────────────┘     │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Instance Endpoint (ต่อ instance โดยตรง, ใช้สำหรับ           │  │
│  │  monitoring/debug เฉพาะตัว ไม่ควรใช้ใน application ทั่วไป)      │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Custom Endpoint (กำหนด subset ของ instance เอง, เช่น         │  │
│  │  แยก reporting workload ไป instance เฉพาะ)                    │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

| Endpoint Type | หน้าที่ | ใช้เมื่อ |
|----------------|---------|----------|
| **Cluster Endpoint (Writer)** | ชี้ไปที่ instance ที่เป็น writer เสมอ (ไม่ว่าจะ failover กี่ครั้ง) | ทุก write operation, transaction ที่ต้องการ consistency |
| **Reader Endpoint** | Load-balance ระหว่าง reader instance ทั้งหมดแบบ DNS round-robin | Read-only query ที่ต้องการกระจายโหลด |
| **Instance Endpoint** | ชี้ไปที่ instance เฉพาะตัวตรง ๆ | Debug, monitoring, หรือกรณีต้องการควบคุม routing เอง |
| **Custom Endpoint** | กำหนด subset ของ instance เอง (เช่น เฉพาะ instance ขนาดใหญ่สำหรับ reporting) | แยก workload เฉพาะทาง (เช่น analytics query หนัก ๆ ไม่ให้กระทบ OLTP reader ตัวอื่น) |

### 937.2 พฤติกรรมสำคัญของ Reader Endpoint

- Reader Endpoint ใช้ **DNS-level load balancing** ซึ่งหมายความว่า **connection ที่เปิดค้างไว้จะไม่ถูกย้าย** — ต้องเปิด connection ใหม่เพื่อให้ได้ routing ใหม่ (สำคัญมากถ้าใช้ connection pooling ฝั่ง application ที่ hold connection นาน)
- TTL ของ DNS record ค่อนข้างสั้น (โดยทั่วไปไม่กี่วินาที) เพื่อให้ rebalance ได้เร็วเมื่อมี reader ใหม่เพิ่ม/ลด
- ถ้า cluster ไม่มี reader instance เลย (เช่น cluster ที่มีแค่ writer) Reader Endpoint จะ route ไปที่ writer แทนโดยอัตโนมัติ

### 937.3 การใช้งาน Endpoint ใน Connection String

```bash
# Writer Endpoint - ใช้สำหรับ write operations
export DB_WRITER="mycompany-aurora-prod.cluster-xxxxxxxxxx.ap-southeast-1.rds.amazonaws.com"

# Reader Endpoint - ใช้สำหรับ read-only operations
export DB_READER="mycompany-aurora-prod.cluster-ro-xxxxxxxxxx.ap-southeast-1.rds.amazonaws.com"

# ตัวอย่างการเชื่อมต่อด้วย psql
psql "host=$DB_WRITER port=5432 dbname=mydb user=pgadmin sslmode=require" -c "INSERT INTO orders ..."
psql "host=$DB_READER port=5432 dbname=mydb user=pgadmin sslmode=require" -c "SELECT ... FROM orders"
```

```bash
# ตรวจสอบ endpoint ทั้งหมดของ cluster
aws rds describe-db-clusters \
    --db-cluster-identifier mycompany-aurora-prod \
    --query "DBClusters[0].{Writer:Endpoint,Reader:ReaderEndpoint}"

# ดูว่า instance ไหนเป็น writer/reader ปัจจุบัน (สำคัญเพราะ failover อาจสลับ role)
aws rds describe-db-clusters \
    --db-cluster-identifier mycompany-aurora-prod \
    --query "DBClusters[0].DBClusterMembers[].{ID:DBInstanceIdentifier,IsWriter:IsClusterWriter}"
```

### 937.4 สร้าง Custom Endpoint สำหรับแยก Workload

```bash
aws rds create-db-cluster-endpoint \
    --db-cluster-identifier mycompany-aurora-prod \
    --db-cluster-endpoint-identifier mycompany-aurora-reporting \
    --endpoint-type READER \
    --static-members mycompany-aurora-prod-reader-2
```

Use case: มี reader 3 ตัว แต่ต้องการให้ reporting/analytics job ยิงเฉพาะไปที่ `reader-2` (instance class ใหญ่กว่า) เพื่อไม่ให้กระทบ `reader-1` ที่ serve traffic จาก application หลัก

### 937.5 Aurora Auto Scaling สำหรับ Read Replica

Aurora รองรับการเพิ่ม/ลด reader instance อัตโนมัติตามโหลด ผ่าน **Application Auto Scaling**

```bash
# ลงทะเบียน scalable target
aws application-autoscaling register-scalable-target \
    --service-namespace rds \
    --scalable-dimension rds:cluster:ReadReplicaCount \
    --resource-id cluster:mycompany-aurora-prod \
    --min-capacity 1 \
    --max-capacity 5

# สร้าง scaling policy ตาม CPU utilization เฉลี่ยของ reader
aws application-autoscaling put-scaling-policy \
    --service-namespace rds \
    --scalable-dimension rds:cluster:ReadReplicaCount \
    --resource-id cluster:mycompany-aurora-prod \
    --policy-name aurora-read-replica-cpu-scaling \
    --policy-type TargetTrackingScaling \
    --target-tracking-scaling-policy-configuration '{
        "TargetValue": 70.0,
        "PredefinedMetricSpecification": {
            "PredefinedMetricType": "RDSReaderAverageCPUUtilization"
        },
        "ScaleInCooldown": 300,
        "ScaleOutCooldown": 300
    }'
```

กลไกการทำงาน:
- เมื่อ average CPU ของ reader instance ทั้งหมดเกิน target value (70% ในตัวอย่าง) ติดต่อกันเกิน cooldown period → เพิ่ม reader instance ใหม่อัตโนมัติ
- เมื่อโหลดลดลง → ลด reader instance กลับ (ไม่ต่ำกว่า `min-capacity`)
- Metric ที่เลือกได้อื่น ๆ เช่น `RDSReaderAverageDatabaseConnections` (scale ตามจำนวน connection แทน CPU)

---

## Step 938: Aurora Serverless v2

### 938.1 แนวคิดของ Serverless v2

Aurora Serverless v2 คือโหมด compute ที่ **auto-scale ตาม workload จริงแบบ fine-grained** โดยไม่ต้องเลือก instance class คงที่ หน่วยวัดคือ **ACU (Aurora Capacity Unit)** ซึ่งประกอบด้วยสัดส่วนของ CPU และ memory ที่แน่นอน

```
Traffic Pattern ตลอดวัน
     │
ACU  │           ▄▄▄▄▄
16   │          ▄█████▄
     │         ▄███████▄            ▄▄▄▄
 8   │      ▄▄▄████████▄▄        ▄▄██████▄▄
     │  ▄▄▄▄██████████████▄▄▄▄▄▄▄██████████▄▄
 2   │▄▄████████████████████████████████████▄▄
 0.5 └──────────────────────────────────────────▶ เวลา
     00:00      08:00      12:00      18:00      24:00

Serverless v2 ปรับ ACU ขึ้น-ลงแบบ "fluid" (วินาทีต่อวินาที)
ตาม CPU/memory/connection ที่ใช้จริง ไม่ใช่การ scale แบบ
"เปลี่ยน instance class ทั้งก้อน" เหมือน Serverless v1
```

- 1 ACU ≈ 2 GiB RAM (พร้อม CPU และ networking ตามสัดส่วน)
- กำหนดช่วง scaling ได้ตั้งแต่ **0.5 ACU ถึง 256 ACU** (ตัวเลขสูงสุดอาจเปลี่ยนแปลงตาม region/เวอร์ชัน — ตรวจสอบ AWS docs ล่าสุดเสมอ)
- Scaling เกิดขึ้น **ภายในไม่กี่วินาที** ไม่ต้อง restart instance หรือมี downtime
- รองรับทั้ง Writer และ Reader ให้เป็น Serverless v2 ได้ (mixed cluster ก็ทำได้ เช่น Writer เป็น Provisioned, Reader เป็น Serverless v2)

### 938.2 สร้าง Aurora Serverless v2

```bash
# Step 1: สร้าง Cluster พร้อมกำหนด ServerlessV2ScalingConfiguration
aws rds create-db-cluster \
    --db-cluster-identifier mycompany-aurora-serverless \
    --engine aurora-postgresql \
    --engine-version 16.4 \
    --master-username pgadmin \
    --manage-master-user-password \
    --db-subnet-group-name mycompany-db-subnet-group \
    --vpc-security-group-ids sg-0123456789abcdef0 \
    --serverless-v2-scaling-configuration MinCapacity=0.5,MaxCapacity=16 \
    --storage-encrypted

# Step 2: สร้าง instance โดยระบุ db-instance-class เป็น db.serverless
aws rds create-db-instance \
    --db-instance-identifier mycompany-aurora-serverless-writer \
    --db-cluster-identifier mycompany-aurora-serverless \
    --engine aurora-postgresql \
    --db-instance-class db.serverless
```

### 938.3 เมื่อไรควรใช้ Serverless v2

| Use Case | เหตุผล |
|----------|--------|
| **Workload ที่มีช่วง peak/off-peak ชัดเจน** เช่น ระบบ HR ที่ใช้เฉพาะเวลาทำการ | ประหยัดค่าใช้จ่ายในช่วง idle โดยไม่ต้อง provision instance ใหญ่ตลอด 24 ชม. |
| **Multi-tenant SaaS ที่แต่ละ tenant มีขนาดต่างกันมาก** | สร้าง 1 cluster ต่อ tenant ได้โดยไม่ต้อง manage instance size ของแต่ละ tenant เอง |
| **Development/Testing environment** | Scale down เหลือ 0.5 ACU เมื่อไม่มีใครใช้งาน ประหยัดค่าใช้จ่ายได้มาก |
| **Workload ที่คาดการณ์ traffic ล่วงหน้าได้ยาก** (เช่น startup ที่เพิ่งเปิดตัว) | ไม่ต้องเดา instance size ล่วงหน้า ลด over-provisioning หรือ under-provisioning |
| **New application ที่ยังไม่รู้ pattern การใช้งาน** | เริ่มด้วย Serverless v2 แล้วดู CloudWatch metrics ก่อนตัดสินใจย้ายไป Provisioned ถ้าคุ้มกว่า |

### 938.4 เมื่อไรไม่ควรใช้ Serverless v2

- **Workload ที่มี CPU/Memory ใช้งานสม่ำเสมอสูงตลอด 24 ชม.** — Provisioned instance ที่ reserved capacity มักคุ้มค่ากว่าในระยะยาว (โดยเฉพาะถ้าซื้อ Reserved Instance)
- **Workload ที่ predictable มาก และต้องการควบคุมต้นทุนแบบตายตัว** — Serverless v2 คิดราคาตาม ACU-hour ที่ใช้จริง ซึ่งทำให้ bill ผันผวนตามการใช้งาน คาดเดาต้นทุนล่วงหน้าได้ยากกว่า
- **Batch job ที่ spike แรงมากในเวลาสั้น ๆ** (เช่น bulk load ข้อมูลขนาดใหญ่ครั้งเดียว) — ควร provision ล่วงหน้าให้เพียงพอ เพราะแม้ scaling จะเร็วแต่ก็ยังมี "ramp time" ที่อาจไม่ทันความต้องการแบบทันทีทันใด

### 938.5 เปรียบเทียบ Serverless v2 กับ Provisioned

| มิติ | Aurora Provisioned | Aurora Serverless v2 |
|------|----------------------|--------------------------|
| การเลือกขนาด | เลือก instance class ตายตัว (เช่น `db.r6g.xlarge`) | กำหนดช่วง Min/Max ACU แล้วปล่อยให้ scale เอง |
| ความเร็วในการ scale | ต้อง `modify-db-instance` (มี brief interruption ถ้าไม่ใช่ Multi-AZ) | Scale ภายในวินาที ไม่มี interruption |
| การคิดราคา | ต่อ instance-hour ตาม instance class คงที่ | ต่อ ACU-hour ตามการใช้งานจริง (billing แบบ per-second) |
| Reserved Instance (ส่วนลดระยะยาว) | รองรับ (ลดต้นทุนได้มากสำหรับ workload สม่ำเสมอ) | ไม่รองรับ Reserved pricing model แบบเดียวกัน |
| ความซับซ้อนในการ capacity planning | ต้องประเมิน peak load ล่วงหน้า | ลดภาระการวางแผน capacity ลงมาก |
| เหมาะกับ mixed cluster (writer provisioned + reader serverless) | ได้ | ได้ (Aurora รองรับ mixed configuration cluster เดียวกัน) |

---

## Step 939: Parameter Groups และ Security Groups

### 939.1 DB Parameter Group คืออะไร

เนื่องจากเราไม่มี OS-level access บน RDS/Aurora การปรับค่า `postgresql.conf` (เช่น `shared_buffers`, `work_mem`, `max_connections`) ต้องทำผ่าน **DB Parameter Group** (สำหรับ RDS instance) หรือ **DB Cluster Parameter Group** (สำหรับ Aurora cluster-level parameter เช่น `rds.logical_replication`)

```
┌──────────────────────────────────────────────────────────┐
│  DB Cluster Parameter Group (เฉพาะ Aurora, cluster-level)   │
│  - ควบคุมพารามิเตอร์ที่ apply ให้ทุก instance ใน cluster        │
│  - เช่น rds.logical_replication, timezone                    │
└─────────────────────┬────────────────────────────────────┘
                       │
      ┌────────────────┼────────────────┐
      ▼                ▼                ▼
┌──────────┐    ┌──────────┐    ┌──────────┐
│ DB Param   │    │ DB Param   │    │ DB Param   │
│ Group      │    │ Group      │    │ Group      │
│ (writer)   │    │ (reader-1) │    │ (reader-2) │
│ - instance-│    │            │    │            │
│   level    │    │            │    │            │
│   เช่น       │    │            │    │            │
│  work_mem  │    │            │    │            │
└──────────┘    └──────────┘    └──────────┘
```

### 939.2 ประเภทของ Parameter: Dynamic vs Static

| ประเภท | ผลของการเปลี่ยนแปลง | ตัวอย่าง |
|--------|------------------------|----------|
| **Dynamic parameter** | Apply ทันทีโดยไม่ต้อง reboot instance | `work_mem`, `random_page_cost`, `log_min_duration_statement` |
| **Static parameter** | ต้อง **reboot instance** ก่อนจึงจะมีผล | `shared_buffers`, `max_connections`, `shared_preload_libraries` |

```bash
# ตรวจสอบว่า parameter ไหนเป็น static/dynamic
aws rds describe-engine-default-parameters \
    --db-parameter-group-family postgres16 \
    --query "EngineDefaults.Parameters[?ParameterName=='shared_buffers' || ParameterName=='work_mem'].{Name:ParameterName,ApplyType:ApplyType,AllowedValues:AllowedValues}"
```

### 939.3 สร้างและตั้งค่า Custom Parameter Group

```bash
# สร้าง custom parameter group (ไม่ควรแก้ default parameter group ของ AWS โดยตรง)
aws rds create-db-parameter-group \
    --db-parameter-group-name mycompany-pg16-custom \
    --db-parameter-group-family postgres16 \
    --description "Custom tuning for mycompany production workload"

# ปรับค่า parameter (ตัวอย่างการ tuning ตามแนวทาง Part เรื่อง Memory Configuration)
aws rds modify-db-parameter-group \
    --db-parameter-group-name mycompany-pg16-custom \
    --parameters \
        "ParameterName=work_mem,ParameterValue=16384,ApplyMethod=immediate" \
        "ParameterName=random_page_cost,ParameterValue=1.1,ApplyMethod=immediate" \
        "ParameterName=log_min_duration_statement,ParameterValue=1000,ApplyMethod=immediate" \
        "ParameterName=shared_preload_libraries,ParameterValue=pg_stat_statements\,pg_cron,ApplyMethod=pending-reboot"

# ผูก parameter group เข้ากับ instance
aws rds modify-db-instance \
    --db-instance-identifier mycompany-prod-pg \
    --db-parameter-group-name mycompany-pg16-custom \
    --apply-immediately

# reboot เพื่อ apply static parameter (เช่น shared_preload_libraries)
aws rds reboot-db-instance --db-instance-identifier mycompany-prod-pg
```

### 939.4 พารามิเตอร์สำคัญที่มักปรับใน RDS/Aurora PostgreSQL

| Parameter | ความหมาย | คำแนะนำ |
|-----------|----------|----------|
| `shared_buffers` | Cache memory ของ PostgreSQL เอง | RDS ตั้งค่า default ตาม instance memory ให้อัตโนมัติ (สูตรประมาณ `{DBInstanceClassMemory/32768}` ปรับตาม engine) มักไม่ต้องแก้ |
| `max_connections` | จำนวน connection สูงสุด | RDS คำนวณ default จาก instance memory ให้ ถ้า connection ไม่พอควรพิจารณา RDS Proxy ก่อนเพิ่มค่านี้ตรง ๆ |
| `work_mem` | Memory ต่อ sort/hash operation | ปรับตาม workload query pattern (ระวัง memory pressure เพราะคูณด้วยจำนวน concurrent query) |
| `rds.force_ssl` | บังคับ SSL สำหรับทุก connection | ควรตั้งเป็น `1` (เปิด) เสมอสำหรับ production |
| `rds.logical_replication` | เปิดใช้ logical replication (สำหรับ CDC, Debezium ฯลฯ) | ต้องเปิดถ้าต้องการทำ logical replication ออกจาก RDS/Aurora — เป็น static parameter ต้อง reboot |
| `log_min_duration_statement` | Log query ที่ใช้เวลานานกว่าค่านี้ (ms) | ตั้งตาม SLA ของระบบ (เช่น 1000ms) เพื่อ debug slow query |
| `shared_preload_libraries` | Extension ที่ต้อง preload ตอน start | เช่น `pg_stat_statements`, `pg_cron` (ที่ RDS/Aurora รองรับ) |
| `rds.force_admin_logging_level` | ควบคุม logging สำหรับ admin operation | ใช้เพื่อ audit การเปลี่ยนแปลงระดับ admin |

### 939.5 Security Group และ VPC Design

RDS/Aurora ทำงานภายใน **VPC (Virtual Private Cloud)** เสมอ และควบคุม network access ผ่าน **Security Group** (ทำหน้าที่เหมือน stateful firewall)

```
┌────────────────────────────── VPC ──────────────────────────────┐
│                                                                    │
│  ┌──────── Public Subnet (AZ-a) ────────┐                          │
│  │   ┌───────────────┐                    │                          │
│  │   │  Bastion Host   │                    │                          │
│  │   │  (SSH เข้าได้    │                    │                          │
│  │   │   จาก IP office) │                    │                          │
│  │   └────────┬────────┘                    │                          │
│  └────────────┼─────────────────────────────┘                          │
│               │                                                        │
│  ┌────────────┼──── Private Subnet (AZ-a) ────┐  ┌── Private Subnet (AZ-b) ──┐│
│  │             ▼                                │  │                            ││
│  │   ┌──────────────────┐                        │  │  ┌──────────────────┐    ││
│  │   │  App Server (EC2)  │───┐                    │  │  │  Aurora Reader     │    ││
│  │   │  Security Group:    │   │                    │  │  │  (standby AZ)      │    ││
│  │   │  app-sg             │   │                    │  │  └──────────────────┘    ││
│  │   └──────────────────┘   │  SG rule: allow 5432   │  │                            ││
│  │                            │  from app-sg only     │  └────────────────────────┘│
│  │   ┌──────────────────┐   │                    │                              │
│  │   │  Aurora Writer      │◀──┘                    │                              │
│  │   │  Security Group:    │                        │                              │
│  │   │  db-sg              │                        │                              │
│  │   │  (ไม่มี public IP,    │                        │                              │
│  │   │   ไม่ publicly        │                        │                              │
│  │   │   accessible)        │                        │                              │
│  │   └──────────────────┘                        │                              │
│  └───────────────────────────────────────────────┘                              │
└────────────────────────────────────────────────────────────────┘
```

### 939.6 หลักการออกแบบ Security Group ที่ปลอดภัย

1. **ไม่เปิด `--publicly-accessible`** สำหรับ production database เด็ดขาด (ยกเว้นมีเหตุผลเฉพาะและควบคุม inbound rule เข้มงวดมาก)
2. **Reference Security Group แทน CIDR block** — อนุญาตให้ `app-sg` เข้าถึง `db-sg` แทนการเปิด CIDR range กว้าง ๆ ทำให้เพิ่ม/ลด EC2 instance ได้โดยไม่ต้องแก้ rule
3. **แยก subnet เป็น public/private** — database อยู่ใน private subnet เสมอ, มี NAT Gateway สำหรับ outbound เท่านั้น
4. **ใช้ IAM Database Authentication** แทน password-based authentication เมื่อเป็นไปได้ เพื่อลดการจัดการ credential

```bash
# สร้าง Security Group สำหรับ database
aws ec2 create-security-group \
    --group-name mycompany-db-sg \
    --description "Security group for Aurora PostgreSQL - production" \
    --vpc-id vpc-0123456789abcdef0

# อนุญาตเฉพาะจาก app-sg เท่านั้น (ไม่ใช่ CIDR)
aws ec2 authorize-security-group-ingress \
    --group-id sg-0db123456789 \
    --protocol tcp \
    --port 5432 \
    --source-group sg-0app123456789

# เปิดใช้ IAM Database Authentication บน cluster
aws rds modify-db-cluster \
    --db-cluster-identifier mycompany-aurora-prod \
    --enable-iam-database-authentication \
    --apply-immediately
```

```sql
-- สร้าง role ภายใน PostgreSQL ที่ผูกกับ IAM authentication
CREATE ROLE app_iam_user WITH LOGIN;
GRANT rds_iam TO app_iam_user;
```

```bash
# แอปพลิเคชันขอ auth token ชั่วคราว (อายุ 15 นาที) แทนการใช้ password ถาวร
aws rds generate-db-auth-token \
    --hostname mycompany-aurora-prod.cluster-xxxxxxxxxx.ap-southeast-1.rds.amazonaws.com \
    --port 5432 \
    --username app_iam_user
```

### 939.7 ตรวจสอบ Parameter Group และ Security Group ปัจจุบัน

```bash
aws rds describe-db-instances \
    --db-instance-identifier mycompany-prod-pg \
    --query "DBInstances[0].{ParamGroups:DBParameterGroups,SecurityGroups:VpcSecurityGroups}"

aws rds describe-db-parameters \
    --db-parameter-group-name mycompany-pg16-custom \
    --source user \
    --query "Parameters[].{Name:ParameterName,Value:ParameterValue,ApplyType:ApplyType}"
```

---

## สรุปท้ายบท

### ตารางเปรียบเทียบ RDS vs Aurora PostgreSQL (สรุปรวม)

| หัวข้อ | Amazon RDS for PostgreSQL | Amazon Aurora PostgreSQL |
|--------|------------------------------|------------------------------|
| Storage architecture | ผูกกับ instance (EBS gp3/io2) | แยกอิสระ, distributed, 6 copies/3 AZ |
| High Availability | Multi-AZ Instance (1 standby, ไม่ readable) หรือ Multi-AZ Cluster (2 readable standby) | Multi-AZ built-in โดยธรรมชาติของ storage layer + สามารถมี reader หลายตัว |
| Read scaling | Read Replica (async, lag เป็นวินาที-นาที) | Aurora Replica (shared storage, lag มักต่ำกว่า 100ms) |
| จำนวน replica สูงสุด | 15 | 15 + Global Database สำหรับ multi-region |
| Failover time | ~60-120s (Instance) / เร็วกว่าสำหรับ Cluster mode | โดยทั่วไปเร็วกว่า RDS Multi-AZ Instance |
| Serverless option | ไม่มี | Aurora Serverless v2 (auto-scale ACU) |
| Backup/PITR | Automated backup + snapshot (managed) | เช่นเดียวกัน + Fast Database Cloning (copy-on-write) |
| Compatibility กับ PostgreSQL มาตรฐาน | 100% | "Compatible" — อาจมีความต่างในบาง extension/edge case |
| ราคา compute | ถูกกว่า | แพงกว่า (premium ~20% โดยประมาณ, ตรวจสอบราคาจริงเสมอ) |
| ราคา storage | Provisioned (จ่ายตามที่จองไว้) | Pay-per-use (จ่ายตามที่ใช้จริง) + I/O request (เว้นแต่เลือก I/O-Optimized) |
| Extension ecosystem | กว้างกว่าเล็กน้อยในบางกรณี (ตรงกับ community PostgreSQL) | รองรับ extension หลัก ๆ ครบ แต่บางตัว/บางเวอร์ชันอาจตามหลัง |
| Vendor lock-in | ปานกลาง (ยังเป็น PostgreSQL มาตรฐาน, migrate ออกง่ายกว่า) | สูงกว่า (storage engine เฉพาะของ AWS, migrate ออกต้องผ่าน logical dump/restore) |
| เหมาะกับ | Workload ทั่วไป, budget จำกัด, ต้องการ portability | Workload ที่ต้องการ HA/read-scaling สูงสุด, งบประมาณรองรับ, ไม่กังวล lock-in |

### สิ่งที่ต้องจำ

1. **RDS และ Aurora ไม่ใช่สิ่งเดียวกัน** — RDS ใช้ storage แบบ instance-attached (EBS) ส่วน Aurora แยก storage ออกมาเป็น distributed layer ของตัวเอง นี่คือความแตกต่างเชิงสถาปัตยกรรมที่ลึกที่สุด ไม่ใช่แค่ "Aurora คือ RDS ที่แพงกว่า"
2. **Multi-AZ (RDS) ≠ Read Replica** — Multi-AZ standby ไม่รับ read traffic (ยกเว้น Multi-AZ DB Cluster รุ่นใหม่ที่ readable) ต้องแยกสร้าง Read Replica ถ้าต้องการ read scaling
3. **Managed ไม่ได้แปลว่าไม่ต้องคิด** — ยังต้องเลือก instance class, storage type, parameter tuning, security group design เอง; AWS จัดการแค่ operational overhead (patching, hardware, backup infrastructure)
4. **PITR แบบ managed restore เป็น instance ใหม่เสมอ** ไม่ใช่ restore in-place เหมือน pgBackRest — ต้องวางแผนเรื่อง DNS/endpoint switching ใน DR runbook
5. **Aurora Serverless v2 เหมาะกับ workload ผันผวน** ไม่ใช่ทุกกรณี — workload สม่ำเสมอสูงมักคุ้มค่ากว่าด้วย Provisioned + Reserved Instance
6. ราคาที่กล่าวถึงในบทนี้เป็นเพียงตัวอย่างเพื่อการเรียนรู้แนวคิดการประมาณต้นทุน **ต้องตรวจสอบราคาจริงจาก AWS Pricing Calculator ก่อนตัดสินใจใช้งานจริงเสมอ**

---

## Step 940: แบบฝึกหัดรวม — ออกแบบสถาปัตยกรรม AWS สำหรับระบบ e-commerce

### โจทย์

บริษัท "ShopThai" กำลังจะย้ายระบบ e-commerce จาก on-premise ไปยัง AWS มีลักษณะ workload ดังนี้:

- ฐานข้อมูลปัจจุบันขนาด **800 GB** เติบโตประมาณ **50 GB/เดือน**
- Traffic หลักคือ **read-heavy** (อัตราส่วน read:write ประมาณ 8:1) จาก catalog browsing, search
- มีช่วง **Flash Sale** 2-3 ครั้ง/เดือนที่ traffic พุ่งขึ้น **10 เท่า** เป็นเวลา 2-4 ชั่วโมง
- ต้องการ **RPO ใกล้ 0** (ห้ามข้อมูลสูญหายเกินไม่กี่วินาที) และ **RTO ไม่เกิน 5 นาที** สำหรับ failure ระดับ AZ
- ทีมมี DBA 1 คน ไม่มีทีม infrastructure เฉพาะทาง
- ผู้บริหารต้องการรายงานประมาณการต้นทุนรายเดือนเบื้องต้น

จงออกแบบสถาปัตยกรรม AWS ที่เหมาะสม พร้อมให้เหตุผลประกอบทุกการตัดสินใจ

---

### แนวทางการตอบ (Architecture Reference — ไม่ใช่คำตอบตายตัวเดียว)

```
┌───────────────────────────────────────────────────────────────┐
│                     Route 53 (DNS) + CloudFront (CDN)              │
└───────────────────────────────┬─────────────────────────────────┘
                                  │
┌───────────────────────────────▼─────────────────────────────────┐
│              Application Load Balancer (Multi-AZ)                   │
└───────────────────────────────┬─────────────────────────────────┘
                                  │
        ┌─────────────────────────┼─────────────────────────┐
        ▼                         ▼                         ▼
┌───────────────┐        ┌───────────────┐        ┌───────────────┐
│  ECS/EC2 App     │        │  ECS/EC2 App     │        │  ECS/EC2 App     │
│  (AZ-a)          │        │  (AZ-b)          │        │  (AZ-c)          │
│  Auto Scaling     │        │  Auto Scaling     │        │  Auto Scaling     │
└────────┬─────────┘        └────────┬─────────┘        └────────┬─────────┘
         │                            │                            │
         └────────────────────────────┼────────────────────────────┘
                                       │
                              ┌────────▼────────┐
                              │   RDS Proxy /     │  ← connection pooling
                              │   PgBouncer        │    (สำคัญมากช่วง Flash Sale)
                              └────────┬────────┘
                                       │
              ┌────────────────────────┼────────────────────────┐
              ▼                        ▼                        ▼
    ┌───────────────────┐   ┌───────────────────┐   ┌───────────────────┐
    │  Aurora Writer       │   │  Aurora Reader-1     │   │  Aurora Reader-2     │
    │  db.r6g.xlarge       │   │  db.r6g.large        │   │  db.r6g.large        │
    │  (AZ-a)              │   │  (AZ-b)              │   │  (AZ-c)              │
    └───────────────────┘   └───────────────────┘   └───────────────────┘
              │                        ▲                        ▲
              └── Aurora Distributed Storage (6 copies / 3 AZ) ──┘
                                       │
                    Aurora Auto Scaling: Reader 2 → 6 ตัว
                    (target CPU 60%, scale-out เร็วช่วง Flash Sale)

    DR: Aurora Global Database → Secondary Region (ap-northeast-1)
        (สำหรับ region-level disaster, RPO วินาที, RTO นาที)
```

### เหตุผลของการตัดสินใจ

**1. เลือก Aurora แทน RDS ธรรมดา**

| ปัจจัยจากโจทย์ | เหตุผลที่ชี้ไปทาง Aurora |
|-------------------|------------------------------|
| Read-heavy 8:1 + ต้องการ replica lag ต่ำ | Aurora Reader lag <100ms เทียบกับ RDS Read Replica ที่อาจ lag เป็นวินาที — สำคัญมากสำหรับ catalog browsing ที่ user คาดหวังเห็นข้อมูลล่าสุด |
| Flash Sale traffic พุ่ง 10 เท่า | Aurora Auto Scaling (Application Auto Scaling บน ReadReplicaCount) เพิ่ม reader อัตโนมัติ + failover เร็วกว่า |
| RTO ≤ 5 นาที | Aurora failover ปกติเร็วกว่า RDS Multi-AZ Instance (~30s เทียบกับ 60-120s) |
| ทีมมี DBA เพียง 1 คน | ลด operational overhead — ไม่ต้องจัดการ storage scaling เอง (Aurora auto-scale storage แบบ transparent ทีละ 10GB) |
| ข้อมูลโต 50GB/เดือน | Aurora storage ขยายอัตโนมัติไม่ต้องตั้ง `max-allocated-storage` เหมือน RDS |

**2. Writer instance class: `db.r6g.xlarge`**

- ข้อมูล 800GB + เติบโตต่อเนื่อง → เลือก r-family (memory-optimized) เพื่อให้ working set ของ catalog/product table fit ใน buffer cache ได้มากที่สุด
- เริ่มที่ `xlarge` (4 vCPU, 32 GiB) แล้ว monitor ผ่าน Performance Insights ก่อนตัดสินใจขยับขนาดจริง

**3. Reader instance + Aurora Auto Scaling**

- เริ่มด้วย reader 2 ตัว (`db.r6g.large`) สำหรับ traffic ปกติ เพื่อกระจาย read load และรองรับ AZ failure
- ตั้ง Application Auto Scaling ให้ scale-out ถึง 6 reader ในช่วง Flash Sale (target CPU 60% เพื่อให้มี headroom ก่อนถึง saturation)
- ใช้ **Custom Endpoint** แยก reader สำหรับ reporting/analytics query ไม่ให้แย่ง resource กับ reader ที่ serve traffic หลัก

**4. RDS Proxy (หรือ PgBouncer)**

- Flash Sale ทำให้ connection spike รุนแรง — RDS Proxy ช่วย pool connection และป้องกัน "connection storm" ที่อาจทำให้ database ปฏิเสธการเชื่อมต่อใหม่
- ช่วยลด failover impact เพราะ RDS Proxy จัดการ re-routing connection ให้แอปพลิเคชันโดยไม่ต้อง reconnect เอง

**5. Multi-AZ โดยธรรมชาติของ Aurora + พิจารณา Aurora Global Database**

- Aurora storage กระจาย 3 AZ อยู่แล้วโดยไม่ต้องตั้งค่าเพิ่ม (ตอบโจทย์ RTO/RPO ระดับ AZ)
- ถ้าต้องการป้องกันระดับ **region** (ไม่ใช่แค่ AZ) ควรพิจารณา **Aurora Global Database** ไปยัง secondary region (เช่น `ap-northeast-1`) — เป็นตัวเลือกเสริมถ้า budget และ RTO/RPO ระดับ region เข้มงวดพอ (ในโจทย์นี้เน้น RTO/RPO ระดับ AZ เป็นหลัก จึงอาจเริ่มจากไม่มี Global Database ก่อน แล้วประเมินเพิ่มภายหลัง)

**6. Backup Strategy**

- `backup-retention-period = 14` วัน (เกินขั้นต่ำ 7 วันเพื่อให้มี buffer สำหรับ incident ที่ตรวจพบช้า)
- Manual snapshot ก่อน major version upgrade หรือก่อน deploy schema change ใหญ่
- Copy snapshot ไป secondary region เป็นระยะสำหรับ DR ระดับ region (ถ้ายังไม่ใช้ Global Database)

### ประมาณการต้นทุนเบื้องต้น (ตัวอย่างประกอบการเรียนรู้เท่านั้น — ไม่ใช่ราคาจริง)

> **คำเตือนสำคัญ**: ตัวเลขด้านล่างเป็น **ตัวอย่างสมมติเพื่อสอนวิธีคิด** เท่านั้น ไม่ใช่ราคาปัจจุบันของ AWS ราคาจริงแตกต่างตาม region, reserved/on-demand pricing, และเปลี่ยนแปลงตลอดเวลา **ต้องคำนวณจาก AWS Pricing Calculator (calculator.aws) ก่อนใช้งานจริงเสมอ**

| รายการ | สเปกโดยประมาณ | หน่วยคิดราคา | ประมาณการ (USD/เดือน, สมมติ) |
|--------|-------------------|----------------|----------------------------------|
| Aurora Writer (`db.r6g.xlarge`) | 4 vCPU, 32GiB, On-Demand | instance-hour x 730 ชม. | ~$700 |
| Aurora Reader x 2 (`db.r6g.large`, baseline) | 2 vCPU, 16GiB x 2 | instance-hour x 730 ชม. x 2 | ~$700 |
| Aurora Reader Auto Scaling ส่วนเพิ่ม (Flash Sale, เฉลี่ย ~20 ชม./เดือน) | reader เพิ่ม 4 ตัวชั่วคราว | instance-hour เฉพาะช่วงใช้จริง | ~$120 |
| Aurora Storage (800GB → คาดว่าจะโต) | ~850GB เฉลี่ย | GB-month | ~$95 |
| Aurora I/O Requests | ประมาณการตาม read-heavy workload | ต่อ 1 ล้าน request | ~$150–300 (ผันผวนมากตาม traffic จริง) |
| Backup Storage เกิน allocated | ส่วนที่เกิน free allowance | GB-month | ~$20–40 |
| RDS Proxy | ตาม vCPU ของ instance ที่เชื่อมต่อ | proxy-instance-hour | ~$150 |
| Data Transfer (ภายใน AZ เดียวกันมักฟรี, ข้าม AZ/region มีค่าใช้จ่าย) | ประมาณการ | GB transferred | ~$50–150 |
| **รวมโดยประมาณ (สมมติ)** | | | **~$1,985–2,995 / เดือน** |

> หมายเหตุ: ถ้าเลือกใช้ **Aurora I/O-Optimized** (storage configuration ที่ไม่คิดค่า I/O request แยก แต่ compute/storage แพงขึ้น) อาจคุ้มค่ากว่าสำหรับ workload ที่มี I/O สูงมากแบบนี้ — ควรเปรียบเทียบทั้งสองแบบด้วย Pricing Calculator จริงก่อนตัดสินใจ

### ทางเลือกอื่นที่ควรพิจารณาเปรียบเทียบ

| ทางเลือก | ข้อดี | ข้อเสีย |
|----------|-------|---------|
| **RDS PostgreSQL + Multi-AZ Cluster + Read Replica** | ค่าใช้จ่ายต่ำกว่า Aurora ~20-30% | Replica lag สูงกว่า, ต้องจัดการ storage scaling เอง (max-allocated-storage) |
| **Aurora Serverless v2 (แทน Provisioned ทั้งหมด)** | ประหยัดช่วง off-peak, ไม่ต้อง provision fix สำหรับ Flash Sale (scale เร็วภายในวินาที) | คาดเดา bill รายเดือนยากกว่า, ต้องทดสอบว่า scale ทันจริงหรือไม่ในช่วง Flash Sale ที่ traffic พุ่งแรงมาก |
| **Aurora Provisioned (Writer) + Serverless v2 (Reader)** | ความสมดุลระหว่าง predictable cost (writer) กับ elastic scaling (reader ช่วง Flash Sale) | ความซับซ้อนในการ monitor 2 รูปแบบ billing |

---

## แบบฝึกหัด

<details>
<summary><strong>แบบฝึกหัดที่ 1</strong>: อธิบายความแตกต่างระหว่าง "Managed Database Service" กับ "Self-Managed Database" โดยยกตัวอย่างสิ่งที่ AWS RDS จัดการให้อย่างน้อย 4 อย่าง และสิ่งที่ยังเป็นหน้าที่ของผู้ใช้งานอย่างน้อย 3 อย่าง</summary>

**เฉลย**:

สิ่งที่ AWS RDS จัดการให้ (ตัวอย่าง 4 อย่าง):
1. OS Patching ของ underlying Linux instance
2. PostgreSQL Engine minor version patching
3. Automated Backup (daily snapshot + continuous WAL archiving)
4. Multi-AZ Failover Orchestration (ตรวจจับและสลับ standby อัตโนมัติ)
5. Hardware replacement เมื่อ physical host/disk เสีย

สิ่งที่ยังเป็นหน้าที่ผู้ใช้งาน (ตัวอย่าง 3 อย่าง):
1. Schema design และ query optimization
2. การเลือก Instance Class, Storage Type ให้เหมาะสมกับ workload
3. การตั้งค่า Security Group / VPC design
4. Major version upgrade (ต้องสั่งเองแม้ AWS จะจัดเตรียม process ให้)
5. Application-level connection pooling (ถ้าไม่ใช้ RDS Proxy)

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 2</strong>: บริษัทหนึ่งมี workload ที่ต้องการ IOPS สูงสม่ำเสมอถึง 20,000 IOPS ตลอดเวลา ควรเลือก storage type ใดระหว่าง gp3 กับ io2 และเพราะเหตุใด</summary>

**เฉลย**: ควรเลือก **io2** เพราะ gp3 บน RDS รองรับ IOPS สูงสุดที่ 16,000 IOPS ซึ่งไม่เพียงพอต่อความต้องการ 20,000 IOPS ที่ต้องการแบบสม่ำเสมอ ในขณะที่ io2 สามารถ provision IOPS ได้สูงกว่าอย่างชัดเจนและยังมี durability ที่สูงกว่า (99.999% เทียบกับ 99.8-99.9% ของ gp3) เหมาะกับ workload ที่ sensitivity ต่อ latency สูงและต้องการ IOPS คงที่ต่อเนื่อง

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 3</strong>: อธิบายว่าทำไม RDS Multi-AZ (DB Instance แบบดั้งเดิม) จึง "ไม่ช่วยเรื่อง read scaling" ทั้งที่มี standby อยู่แล้ว และต้องทำอย่างไรถ้าต้องการทั้ง HA และ read scaling</summary>

**เฉลย**: ใน RDS Multi-AZ DB Instance แบบดั้งเดิม standby ทำหน้าที่เพื่อ **High Availability เท่านั้น** — ไม่รับ read query จาก client โดยตรง (ไม่มี read-only endpoint ชี้ไปที่ standby) มีไว้เพื่อรอ promote เป็น primary กรณี failover เท่านั้น

ถ้าต้องการทั้ง HA และ read scaling พร้อมกัน มีทางเลือก:
1. เปิด **Multi-AZ DB Cluster** (รุ่นใหม่) ที่มี 2 readable standby พร้อม Reader Endpoint ในตัว
2. ใช้ **Multi-AZ DB Instance + สร้าง Read Replica แยกต่างหาก** (Read Replica คนละตัวกับ Multi-AZ standby)
3. ย้ายไปใช้ **Aurora** ที่มี Reader instance รับ read traffic ได้พร้อม HA ในตัวโดยธรรมชาติ

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 4</strong>: ระบบหนึ่งมี Read Replica ข้าม region จาก Singapore ไป Tokyo แอปพลิเคชันเขียนข้อมูลที่ Singapore แล้วอ่านทันทีที่ Tokyo replica พบว่าบางครั้งไม่เห็นข้อมูลที่เพิ่งเขียน ปัญหานี้คืออะไร และแก้ไขอย่างไร</summary>

**เฉลย**: ปัญหานี้คือ **Read-after-write inconsistency** ซึ่งเกิดจากธรรมชาติของ Read Replica ที่ใช้ **asynchronous replication** — ข้อมูลที่เขียนที่ primary (Singapore) ต้องใช้เวลาระยะหนึ่ง (replication lag) กว่าจะไปถึง replica (Tokyo) โดยเฉพาะ cross-region ที่มี network latency สูงกว่า same-region

วิธีแก้ไข:
1. สำหรับ flow ที่ต้องการ **strong consistency ทันทีหลังเขียน** (เช่น หน้า order confirmation หลัง checkout) ให้อ่านจาก **primary endpoint** แทนที่จะอ่านจาก replica
2. Monitor `ReplicaLag` metric และตั้ง threshold เพื่อ fallback ไปอ่าน primary ถ้า lag สูงเกินไป
3. ออกแบบ application ให้แยก flow ที่ tolerant ต่อ eventual consistency (เช่น catalog browsing) ออกจาก flow ที่ต้องการความถูกต้องทันที
4. พิจารณาย้ายไปใช้ Aurora ที่มี replica lag ต่ำกว่ามาก (แต่ยังคงเป็น async เช่นกัน ไม่ใช่ synchronous)

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 5</strong>: อธิบายความแตกต่างระหว่าง Automated Backup กับ Manual Snapshot ใน RDS โดยเฉพาะประเด็นเรื่องการลบ instance</summary>

**เฉลย**:

| คุณสมบัติ | Automated Backup | Manual Snapshot |
|-----------|-------------------|-------------------|
| สร้างโดย | อัตโนมัติตาม retention period | ผู้ใช้สั่งสร้างเอง |
| เมื่อลบ instance | **ถูกลบไปพร้อมกัน** (เว้นแต่สร้าง Final Snapshot ตอนลบ) | **คงอยู่ถาวร** ไม่ถูกลบแม้ instance หลักจะถูกลบไปแล้ว |
| อายุการเก็บ | จำกัดตาม retention period (สูงสุด 35 วัน) | ไม่จำกัดอายุ จนกว่าจะลบเอง |
| ใช้สำหรับ | PITR granular ถึงระดับ 5 นาที | Restore ไปจุดที่ snapshot ถูกสร้าง หรือเก็บ archive ระยะยาว |

ข้อควรระวัง: หากต้องการลบ instance แต่ยังต้องการเก็บข้อมูลไว้ **ต้องสร้าง Final Snapshot** ด้วย flag `--final-db-snapshot-identifier` (ไม่ใช้ `--skip-final-snapshot`) มิฉะนั้นข้อมูล backup ทั้งหมดจะหายไปพร้อมกับ instance

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 6</strong>: อธิบายกลไก Write Quorum และ Read Quorum ของ Aurora ว่าทำงานอย่างไร และเพราะเหตุใดการตั้งค่าที่ 4/6 write, 3/6 read จึงรับประกันความถูกต้องของข้อมูล (consistency)</summary>

**เฉลย**: Aurora เก็บข้อมูลเป็น 6 สำเนา กระจายใน 3 AZ (AZ ละ 2 สำเนา)

- **Write Quorum = 4/6**: การเขียนถือว่าสำเร็จเมื่อเขียนได้อย่างน้อย 4 จาก 6 สำเนา
- **Read Quorum = 3/6**: การอ่านถือว่าถูกต้องเมื่ออ่านได้อย่างน้อย 3 จาก 6 สำเนา

เหตุผลที่รับประกัน consistency: เพราะ **4 + 3 = 7 > 6** (จำนวนสำเนาทั้งหมด) ดังนั้นไม่ว่า write quorum (4 สำเนา) จะไปลงที่ชุดใดก็ตาม และ read quorum (3 สำเนา) จะไปอ่านชุดใดก็ตาม **จะต้องมีอย่างน้อย 1 สำเนาที่ overlap กันเสมอ** (pigeonhole principle) ทำให้การอ่านจะเจอข้อมูลเวอร์ชันล่าสุดที่เขียนสำเร็จแล้วเสมอ ไม่มีทางอ่านได้ข้อมูลเก่าที่ยังไม่ commit

นอกจากนี้การกระจาย 2 สำเนา/AZ ทำให้ระบบทนต่อการสูญเสียทั้ง 1 AZ (เหลือ 4 สำเนาจาก 2 AZ ที่เหลือ ยังคงเขียนได้) และทนอ่านได้แม้ AZ ล่ม 1 ตัว (เหลือ 4 สำเนา ≥ 3 ที่ต้องการ)

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 7</strong>: Aurora Reader Endpoint ใช้กลไก DNS round-robin ในการกระจายโหลด สิ่งนี้ส่งผลอย่างไรต่อแอปพลิเคชันที่ใช้ connection pooling แบบเปิด connection ค้างไว้นาน (long-lived connection pool) และควรแก้ปัญหาอย่างไร</summary>

**เฉลย**: เนื่องจาก Reader Endpoint ทำ load balancing ที่ **ระดับ DNS resolution** เท่านั้น (ไม่ใช่ระดับ connection/query) connection ที่เปิดค้างไว้นาน ๆ จะ **"ติด" อยู่กับ reader instance ตัวเดิม** ตลอดอายุของ connection นั้น แม้ reader ตัวอื่นจะว่างกว่าก็ตาม ทำให้เกิด **load imbalance** ระหว่าง reader โดยเฉพาะเมื่อมีการเพิ่ม reader ใหม่เข้าไปใน cluster (connection เก่าจะไม่ถูกย้ายไปหา reader ใหม่โดยอัตโนมัติ)

วิธีแก้ไข:
1. ตั้งค่า **connection pool ให้มี maximum connection lifetime** (เช่น recycle connection ทุก 5-10 นาที) เพื่อบังคับให้ re-resolve DNS และกระจายโหลดใหม่เป็นระยะ
2. ใช้ **RDS Proxy** ซึ่งจัดการ connection routing ที่ฉลาดกว่า DNS round-robin ธรรมดา
3. ลด TTL ของ application-level DNS cache ให้สอดคล้องกับ TTL ของ Aurora Reader Endpoint record

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 8</strong>: ระบบ Dev/Test environment มีการใช้งานเฉพาะเวลา 09:00-18:00 วันจันทร์-ศุกร์เท่านั้น ควรพิจารณาใช้ Aurora Provisioned หรือ Aurora Serverless v2 เพราะเหตุใด</summary>

**เฉลย**: ควรพิจารณา **Aurora Serverless v2** เพราะ:
1. Workload มีช่วง idle ชัดเจน (นอกเวลาทำการ, วันหยุดสุดสัปดาห์) — Serverless v2 สามารถ scale down เหลือ minimum ACU (เช่น 0.5 ACU) ในช่วงที่ไม่มีใครใช้งาน ช่วยประหยัดค่าใช้จ่ายอย่างมีนัยสำคัญเทียบกับการเปิด Provisioned instance ทิ้งไว้ 24 ชม. โดยไม่ได้ใช้
2. Dev/Test ไม่ต้องการ SLA เข้มงวดเท่า production จึงยอมรับความผันผวนของ scaling ได้มากกว่า
3. ลดภาระการต้อง manual start/stop instance เพื่อประหยัดเงิน (ซึ่งเป็นแนวทางที่ทีมมักทำกับ Provisioned instance แต่เพิ่มความซับซ้อนในการดำเนินงาน)

ข้อควรระวัง: ต้องตรวจสอบว่า cost ที่เกิดจาก billing แบบ ACU-hour (per-second) ยังคงถูกกว่า Provisioned instance ขนาดเล็กที่สุดที่เพียงพอ เพราะถ้า workload ใช้งานเกือบตลอดเวลาทำการทุกวัน Provisioned อาจคุ้มกว่าในบางกรณี ควรคำนวณเปรียบเทียบทั้งสองแบบด้วย Pricing Calculator จริง

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 9</strong>: parameter `shared_preload_libraries` เป็น static หรือ dynamic parameter และมีผลอย่างไรต่อขั้นตอนการเปิดใช้ extension เช่น `pg_cron` บน RDS/Aurora</summary>

**เฉลย**: `shared_preload_libraries` เป็น **static parameter** ซึ่งหมายความว่าการเปลี่ยนแปลงค่านี้จะ **ไม่มีผลทันที** ต้องผ่านขั้นตอน:

1. แก้ไขค่าใน DB Parameter Group (หรือ DB Cluster Parameter Group สำหรับ Aurora) ด้วย `ApplyMethod=pending-reboot`
2. **Reboot instance** (หรือ cluster) เพื่อให้ค่าที่เปลี่ยนมีผลจริง
3. หลัง reboot แล้วจึงสามารถรัน `CREATE EXTENSION pg_cron;` ใน SQL ได้สำเร็จ (ถ้า reboot ยังไม่เกิดขึ้น การสร้าง extension จะ error เพราะ library ยังไม่ถูก preload)

ข้อควรระวังเชิงปฏิบัติการ: เนื่องจากต้อง reboot ทั้ง instance การเปลี่ยนค่า static parameter ในระบบ production ควรวางแผนช่วง maintenance window และแจ้งทีมล่วงหน้า เพราะจะมี brief downtime (หรือ failover ถ้าเป็น Multi-AZ)

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 10</strong>: ทีมหนึ่งกำลังออกแบบ Security Group สำหรับ Aurora cluster และมีข้อเสนอ 2 แบบ: (A) เปิด inbound rule อนุญาต CIDR ของทั้ง VPC (เช่น 10.0.0.0/16) เข้าถึง port 5432 ได้ทั้งหมด (B) อนุญาตเฉพาะจาก Security Group ของ application server เท่านั้น ข้อใดเป็นแนวทางที่ดีกว่า และเพราะเหตุใด</summary>

**เฉลย**: แนวทาง **(B) อนุญาตเฉพาะจาก Security Group ของ application server** เป็นแนวทางที่ดีกว่าอย่างชัดเจน ด้วยเหตุผล:

1. **หลักการ Least Privilege**: การเปิด CIDR ทั้ง VPC (10.0.0.0/16) หมายความว่า **ทุกทรัพยากรใน VPC นั้น** (รวมถึง EC2 instance อื่น ๆ ที่ไม่เกี่ยวข้อง, Lambda ที่รันใน VPC, หรือแม้แต่ resource ที่ถูก compromise ในอนาคต) จะเข้าถึง database ได้หมด ซึ่งขยาย attack surface โดยไม่จำเป็น
2. **การ Reference Security Group** (แบบ B) ทำให้ rule ผูกกับ "บทบาท" ของ resource ไม่ใช่ "ตำแหน่ง IP" — เมื่อ Auto Scaling เพิ่ม/ลด EC2 instance ของ application server, instance ใหม่ที่อยู่ใน `app-sg` จะเข้าถึง database ได้ทันทีโดยไม่ต้องแก้ rule ใด ๆ เพิ่มเติม ในขณะที่ instance อื่นที่ไม่ได้อยู่ใน `app-sg` จะเข้าไม่ได้แม้จะอยู่ใน VPC เดียวกัน
3. **Audit และ Compliance ง่ายกว่า**: เมื่อ security team ตรวจสอบ rule จะเห็นชัดเจนว่า "เฉพาะ application tier เท่านั้นที่คุยกับ database ได้" ซึ่งสื่อความหมายเชิงสถาปัตยกรรมชัดเจนกว่าการเห็นแค่ CIDR range
4. หากมีการเพิ่ม resource ใหม่ใน VPC ในอนาคต (เช่น analytics tool, third-party service) จะไม่มีสิทธิ์เข้าถึง database โดยอัตโนมัติ ต้องถูกเพิ่มเข้า `app-sg` หรือสร้าง rule ใหม่อย่างจงใจเท่านั้น ลดความเสี่ยงจาก "การเข้าถึงโดยไม่ตั้งใจ" (implicit over-permissioning)

สรุป: ควรใช้ **Security Group-to-Security-Group reference** (`--source-group`) แทนการเปิด CIDR block กว้าง ๆ เสมอ เว้นแต่มีเหตุผลเฉพาะเจาะจงจริง ๆ ที่ทำให้ reference security group ทำไม่ได้ (เช่น การเชื่อมต่อจากภายนอก VPC ที่ไม่มี Security Group ให้ reference)

</details>

---

## บทถัดไป

บทถัดไปจะพาไปสำรวจ PostgreSQL บน Google Cloud Platform — Cloud SQL for PostgreSQL และ AlloyDB ซึ่งมีแนวคิดคล้ายกับ RDS/Aurora แต่มีรายละเอียดเชิงสถาปัตยกรรมและกลไกภายในที่แตกต่างกันอย่างน่าสนใจ

**อ่านต่อ**: [Part 095 — PostgreSQL บน GCP: Cloud SQL และ AlloyDB](./part-095-gcp-cloudsql-alloydb.md)