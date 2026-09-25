# Part 065: High Availability — Patroni, repmgr, Failover Automation

หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับมืออาชีพ | Part 065

---

> **หมายเหตุสำคัญก่อนเริ่มบทเรียน**
>
> บทนี้เป็นบทที่ว่าด้วยสถาปัตยกรรมและเครื่องมือระดับ Production สำหรับทำ High Availability (HA) จริงบน PostgreSQL ซึ่งโดยธรรมชาติของ HA แล้วต้องการ **หลายเครื่อง (multi-node)** ทำงานร่วมกัน — อย่างน้อย 3 โหนด PostgreSQL บวกกับ Distributed Consensus Store อีก 3 โหนด (etcd/Consul) บวกกับ Load Balancer อีกอย่างน้อย 1-2 เครื่อง
>
> สภาพแวดล้อมที่ใช้เรียนหลักสูตรนี้เป็นเครื่องเดียว (single environment) จึง **ไม่สามารถสร้างคลัสเตอร์ multi-node จริงให้ทดลองรันได้** ในบทนี้ เนื้อหาส่วนที่เป็นสถาปัตยกรรม การออกแบบ และไฟล์คอนฟิกตัวอย่าง (`patroni.yml`, `repmgr.conf`, `haproxy.cfg`) จึงเป็น **เนื้อหาเชิงสถาปัตยกรรม/แนวคิด (conceptual & architectural)** ที่ถูกต้องตรงตาม syntax และ best practice ของเครื่องมือจริงในระดับ Production — คุณสามารถนำไปใช้อ้างอิงและปรับใช้ได้ทันทีเมื่อมีคลัสเตอร์จริง (on-prem, cloud VM, หรือ Kubernetes) แต่ **จะไม่มีคำสั่งให้รันตามในเครื่องเรียนนี้แบบ end-to-end** เหมือนบทก่อน ๆ ที่เป็น single-node
>
> จุดที่สามารถทดลองได้จริงในเครื่องเดียว (เช่น การตั้งค่า streaming replication แบบ 2 โหนดจำลองด้วย 2 พอร์ตบนเครื่องเดียวกัน, `pg_ctl promote`, trigger file) ได้เรียนไปแล้วใน Part 063 — บทนี้ต่อยอดจากความเข้าใจนั้นไปสู่ระดับที่ automate การ failover ด้วยซอฟต์แวร์ HA orchestration

---

## เป้าหมายการเรียนรู้

หลังจบบทเรียนนี้ คุณจะสามารถ:

1. อธิบายความหมายของ High Availability, คำนวณ downtime ต่อปีจากค่า SLA แบบ "nines" ได้อย่างแม่นยำ
2. อธิบายได้ว่าทำไม manual failover (ที่เรียนใน Part 063) ไม่เพียงพอสำหรับระบบ Production ที่ต้องการ SLA สูง
3. ระบุองค์ประกอบหลัก 4 อย่างของระบบ HA อัตโนมัติ: health check, consensus/leader election, automatic failover, client redirection
4. เข้าใจสถาปัตยกรรมและวิธีตั้งค่าเบื้องต้นของ repmgr และ repmgrd
5. เข้าใจสถาปัตยกรรมของ Patroni อย่างลึกซึ้ง รวมถึงบทบาทของ Distributed Consensus Store (DCS) เช่น etcd, Consul, ZooKeeper
6. อ่านและเขียนไฟล์ `patroni.yml` ได้ครบทุก section พร้อมเข้าใจความหมายของแต่ละ parameter
7. เข้าใจภาพรวมว่า HAProxy/PgBouncer เข้ามาช่วย route traffic ไปยัง leader โหนดอัตโนมัติร่วมกับ Patroni ได้อย่างไร (รายละเอียดเต็มใน Part 066-067)
8. อธิบายปัญหา split-brain ได้อย่างชัดเจน พร้อมวิธีป้องกันด้วยแนวคิด fencing และ STONITH
9. ออกแบบสถาปัตยกรรม HA แบบเต็มรูปแบบสำหรับระบบระดับ Production เช่น e-commerce ได้

---

## Step 641: High Availability (HA) คืออะไร

### นิยาม

**High Availability (HA)** คือคุณสมบัติของระบบที่ทำให้ระบบยังคง **ให้บริการได้ต่อเนื่อง** แม้ว่าจะมี component บางส่วนล้มเหลว (failure) ไม่ว่าจะเป็นฮาร์ดแวร์เสีย, เครือข่ายขาด, ซอฟต์แวร์ crash, หรือแม้แต่ดาต้าเซ็นเตอร์ทั้งแห่งไฟดับ

หัวใจของ HA ไม่ใช่การ "ป้องกันไม่ให้เกิดความล้มเหลว" (นั่นเป็นไปไม่ได้ในระยะยาว — ฮาร์ดแวร์ทุกชิ้นมีวันเสีย) แต่คือการ **ลดผลกระทบของความล้มเหลวให้เหลือน้อยที่สุด** ทั้งในแง่ของ:

- **Downtime** — เวลาที่ระบบใช้บริการไม่ได้
- **Data loss** — ข้อมูลที่อาจสูญหายไปในช่วงที่เกิดปัญหา

### สูตรคำนวณ Availability

```
Availability (%) = (Total Time - Downtime) / Total Time × 100
```

ตัวอย่างเช่น ถ้าระบบทำงาน 364 วันจาก 365 วันในหนึ่งปี (มี downtime รวม 1 วัน):

```
Availability = (365 - 1) / 365 × 100 = 99.726%
```

### SLA และแนวคิด "Nines"

ในสัญญาระดับบริการ (Service Level Agreement — SLA) มักระบุ availability เป็นเปอร์เซ็นต์ที่มีเลข 9 ต่อกัน เรียกกันในวงการว่า "nines" ยิ่งมี 9 เยอะ ยิ่งหมายถึง downtime ที่ยอมรับได้ต่อปีน้อยลงมาก (ไม่ใช่น้อยลงเป็นเส้นตรง แต่ลดลงแบบ **10 เท่า** ในแต่ละ nine ที่เพิ่มขึ้น)

| ระดับ | Availability | Downtime/ปี | Downtime/เดือน | Downtime/สัปดาห์ |
|---|---|---|---|---|
| 1 nine | 90% | 36.5 วัน | 72 ชั่วโมง | 16.8 ชั่วโมง |
| 2 nines | 99% | 3.65 วัน | 7.2 ชั่วโมง | 1.68 ชั่วโมง |
| 3 nines | 99.9% | 8.76 ชั่วโมง | 43.8 นาที | 10.1 นาที |
| 4 nines | 99.99% | 52.56 นาที | 4.38 นาที | 1.01 นาที |
| 5 nines | 99.999% | 5.26 นาที | 25.9 วินาที | 6.05 วินาที |
| 6 nines | 99.9999% | 31.5 วินาที | 2.59 วินาที | 0.605 วินาที |

ข้อสังเกตที่สำคัญมาก: **99.99% (4 nines) อนุญาตให้ downtime ได้เพียง ~52 นาทีต่อปีทั้งปี** ถ้าคุณทำ manual failover ที่ใช้เวลาเฉลี่ย 15-30 นาทีต่อครั้ง (ตรวจสอบปัญหา + ตัดสินใจ + สั่ง promote + reconfigure connection) แค่เหตุการณ์ล้มเหลว **2 ครั้ง** ต่อปีก็ทำให้ SLA 99.99% พังไม่เหลือ budget แล้ว

### RTO และ RPO

สองค่าที่ต้องเข้าใจคู่กับ HA เสมอ:

- **RTO (Recovery Time Objective)** — "ระบบต้องกลับมาใช้งานได้ภายในเวลาเท่าไรหลังเกิดปัญหา" เช่น RTO = 30 วินาที หมายถึงหลังจาก primary ล้มเหลว ระบบต้องมี node ใหม่ทำหน้าที่ primary และรับ write ได้ภายใน 30 วินาที
- **RPO (Recovery Point Objective)** — "ข้อมูลสูญหายได้มากที่สุดเท่าไร" วัดเป็นเวลาย้อนหลัง เช่น RPO = 0 หมายถึงห้ามข้อมูลหายแม้แต่ transaction เดียวที่ commit แล้ว (ต้องใช้ synchronous replication), RPO = 5 นาที หมายถึงยอมรับได้ที่ข้อมูล 5 นาทีสุดท้ายก่อนเกิดปัญหาอาจหายไป

ระบบ HA ที่ดีต้องถูกออกแบบโดยกำหนด RTO/RPO เป้าหมายให้ชัดเจนตั้งแต่แรก เพราะมันกำหนดว่าต้องเลือกใช้ synchronous หรือ asynchronous replication, ต้องมี automated failover หรือพอแค่ automated detection + notify, และต้องมีกี่ standby node

### Trade-off: ยิ่ง HA สูง ยิ่งซับซ้อนและแพงขึ้น

```
Availability เป้าหมาย    ต้องมี
─────────────────────    ──────────────────────────────────────
99%    (2 nines)         Single server + backup ปกติ, manual restore
99.9%  (3 nines)         Primary + 1 standby (async), manual failover
99.99% (4 nines)         Primary + ≥2 standby, automated failover (Patroni/repmgr)
                         + load balancer + monitoring
99.999%(5 nines)         ข้างต้น + multi-AZ/multi-datacenter + synchronous
                         replication + automated fencing + extensive testing
```

ทุก 9 ที่เพิ่มขึ้นมักหมายถึงต้นทุนที่เพิ่มขึ้นแบบไม่เป็นเส้นตรง (มากกว่า 2-3 เท่า) ดังนั้นการเลือกระดับ HA ต้องพิจารณาจาก **ผลกระทบทางธุรกิจของ downtime** เทียบกับ **ต้นทุนของโครงสร้างพื้นฐาน** เสมอ — ระบบ blog ส่วนตัวไม่จำเป็นต้องมี 5 nines แต่ระบบ payment gateway ของธนาคารอาจต้องการมันจริง ๆ

---

## Step 642: ทำไม Manual Failover ไม่พอสำหรับ Production

ใน Part 063 เราได้เรียนกระบวนการ manual failover ไปแล้ว: ตรวจพบว่า primary ล้มเหลว → ตัดสินใจ → สั่ง `pg_ctl promote` หรือสร้าง trigger file บน standby → รอให้ standby กลายเป็น primary → เปลี่ยนค่า connection string หรือ DNS ของแอปพลิเคชันให้ชี้ไปที่ primary ใหม่ → ตรวจสอบว่าระบบกลับมาทำงานปกติ

กระบวนการนี้ **ใช้งานได้จริงและถูกต้อง** แต่มีข้อจำกัดร้ายแรงหลายประการที่ทำให้ไม่เหมาะกับระบบ Production ที่ต้องการ SLA สูง:

### 1. Detection Delay — มนุษย์ตรวจจับปัญหาช้า

ถ้าไม่มีระบบ monitoring ที่ alert ทันที ปัญหาอาจไม่ถูกพบจนกว่าจะมีคนโทรมาแจ้งว่าเว็บใช้งานไม่ได้ หรือ DBA เข้ามาเช็คตามรอบ ในกรณีเลวร้าย (เช่น เที่ยงคืนวันหยุด) ปัญหาอาจไม่ถูกพบเป็นชั่วโมง

### 2. Decision Paralysis และ Human Error

เมื่อ DBA ถูกปลุกตอนตี 3 ด้วย alert ความเครียดและความง่วงทำให้การตัดสินใจช้าลงและเสี่ยงต่อการพิมพ์คำสั่งผิด เช่น สั่ง promote node ผิดตัว หรือลืมตรวจสอบว่า standby ตัวไหนมีข้อมูลล่าสุดที่สุดก่อน promote (อาจทำให้เลือก standby ที่ replication lag เยอะที่สุดมาเป็น primary ใหม่โดยไม่รู้ตัว)

### 3. เวลาที่ใช้ทั้งกระบวนการยาวเกินไป

เปรียบเทียบเวลาที่ใช้จริง:

```
Manual Failover Timeline (ทั่วไป)
──────────────────────────────────────────────────────────
 00:00   Primary ล้มเหลว
 00:00-05:00   Monitoring alert ส่งเข้า on-call (ถ้ามี)
 05:00-10:00   DBA รับสาย/เปิดแล็ปท็อป/VPN เข้าระบบ
 10:00-15:00   ตรวจสอบสถานการณ์ ยืนยันว่า primary ตายจริง (ไม่ใช่ network flap)
 15:00-20:00   ตรวจสอบว่า standby ตัวไหน replication lag น้อยที่สุด
 20:00-22:00   สั่ง promote standby
 22:00-25:00   เปลี่ยน DNS/connection string/VIP ให้ชี้ไป primary ใหม่
 25:00-28:00   Restart/reconnect application pool
 28:00-30:00   ยืนยันว่าระบบกลับมาทำงานปกติ
──────────────────────────────────────────────────────────
รวม downtime: ~25-30 นาที (ในกรณีที่ดี — ทีมมีประสบการณ์และตื่นทันที)
```

เทียบกับ automated failover ที่ทำทุกขั้นตอนข้างต้นภายใน **10-30 วินาที** เพราะไม่ต้องรอมนุษย์ตื่น ตรวจสอบ และพิมพ์คำสั่งเอง

### 4. ไม่ Scale เมื่อมีหลายคลัสเตอร์

องค์กรที่มี PostgreSQL cluster เดียวอาจพอจัดการ manual failover ได้ แต่องค์กรที่มี 50-100 cluster (เช่น SaaS ที่แยก database ต่อลูกค้า หรือมี microservices จำนวนมาก) การพึ่งพา DBA ตัดสินใจ manual ทุกครั้งเป็นไปไม่ได้ในทางปฏิบัติ

### 5. Business Impact ที่จับต้องได้

ตัวอย่างระบบ e-commerce ที่มียอดขายเฉลี่ย 100,000 บาท/นาทีในช่วง peak:

```
Manual failover downtime 25 นาที  → สูญเสียยอดขายที่อาจเกิดขึ้น ~2,500,000 บาท
Automated failover downtime 20 วินาที → สูญเสียยอดขายที่อาจเกิดขึ้น ~33,000 บาท
```

นี่ยังไม่นับความเสียหายทางชื่อเสียง (reputation), SLA penalty ที่ต้องจ่ายคืนลูกค้า, และ churn ของลูกค้าที่หันไปใช้คู่แข่ง

### สรุป

Manual failover เหมาะสำหรับ:
- ระบบ non-critical, dev/staging environment
- องค์กรขนาดเล็กที่ downtime ไม่กี่สิบนาทีไม่กระทบธุรกิจมาก
- สถานการณ์ที่ยังไม่มี automated tooling พร้อมใช้

Production ระดับ Enterprise ที่ต้องการ SLA 99.9% ขึ้นไป **จำเป็น** ต้องมีระบบ HA อัตโนมัติที่ตรวจจับปัญหาและ failover โดยไม่รอมนุษย์ ซึ่งนำเราไปสู่เครื่องมืออย่าง **repmgr** และ **Patroni** ที่จะอธิบายในบทนี้

---

## Step 643: องค์ประกอบของระบบ HA อัตโนมัติ

ระบบ HA อัตโนมัติสำหรับ PostgreSQL ไม่ว่าจะใช้เครื่องมือใด (repmgr, Patroni, หรือแม้แต่ cloud-managed service อย่าง RDS Multi-AZ) ล้วนต้องมีองค์ประกอบหลัก 4 อย่างทำงานร่วมกัน:

```
┌─────────────────────────────────────────────────────────────────┐
│                     HA System Components                         │
│                                                                    │
│   ┌──────────────┐      ┌───────────────────────┐               │
│   │ 1. Health     │─────▶│ 2. Consensus /         │               │
│   │    Check      │      │    Leader Election     │               │
│   └──────────────┘      └───────────┬───────────┘               │
│                                      │                             │
│                                      ▼                             │
│                          ┌───────────────────────┐               │
│                          │ 3. Automatic Failover  │               │
│                          └───────────┬───────────┘               │
│                                      │                             │
│                                      ▼                             │
│                          ┌───────────────────────┐               │
│                          │ 4. Client Redirection  │               │
│                          └───────────────────────┘               │
└─────────────────────────────────────────────────────────────────┘
```

### 1. Health Check (การตรวจสุขภาพ)

กระบวนการตรวจสอบว่าแต่ละ node ในคลัสเตอร์ยังทำงานปกติหรือไม่ โดยทั่วไปทำผ่าน:

- **Heartbeat / Ping** — ตรวจสอบว่า process หรือ network endpoint ยังตอบสนอง
- **SQL-level probe** — รัน query ง่าย ๆ เช่น `SELECT 1` หรือ `SELECT pg_is_in_recovery()` เพื่อตรวจสอบว่า PostgreSQL ยังทำงานได้จริง ไม่ใช่แค่ process ยังไม่ตาย
- **REST API check** — เครื่องมือ HA สมัยใหม่อย่าง Patroni มี REST API endpoint ให้ตรวจสอบสถานะ (`/health`, `/leader`, `/replica`)

การตรวจสอบต้องเกิดขึ้นถี่พอที่จะจับปัญหาได้เร็ว (เช่นทุก 1-2 วินาที) แต่ไม่ถี่เกินจนสร้างภาระให้ระบบ และต้องมี **timeout/retry logic** ที่เหมาะสมเพื่อไม่ให้ network เพี้ยนชั่วขณะ (transient network blip) ถูกตีความว่าเป็นปัญหาจริง — ประเด็นนี้เกี่ยวข้องโดยตรงกับปัญหา split-brain ที่จะอธิบายใน Step 649

### 2. Consensus / Leader Election (ฉันทามติและการเลือกผู้นำ)

เมื่อ health check พบว่า primary (leader) มีปัญหา ระบบต้องมีกลไกให้ node ที่เหลือ **ตกลงกันได้อย่างเป็นเอกฉันท์** ว่าใครจะเป็น leader คนใหม่ โดยไม่มี node สองตัวคิดว่าตัวเองเป็น leader พร้อมกัน (ซึ่งจะนำไปสู่ split-brain)

กลไกนี้อาศัยอัลกอริทึมทาง distributed systems เช่น **Raft** หรือ **Paxos** ซึ่งเป็นหัวใจของ Distributed Consensus Store อย่าง etcd, Consul, ZooKeeper — นี่คือเหตุผลที่ Patroni เลือกใช้เครื่องมือเหล่านี้แทนที่จะพยายามเขียนกลไก consensus เองใหม่ทั้งหมด

หลักการสำคัญคือ **Quorum** — ต้องมี node เสียงข้างมาก (majority) เห็นพ้องกันก่อนถึงจะยืนยันผลการเลือกตั้งได้ เช่น คลัสเตอร์ 3 node ต้องมีอย่างน้อย 2 node เห็นตรงกัน คลัสเตอร์ 5 node ต้องมีอย่างน้อย 3 node — นี่คือเหตุผลที่ DCS cluster (etcd/Consul) มักตั้งเป็นจำนวนคี่ (3, 5, 7) เพื่อให้คำนวณ quorum ได้ชัดเจนและทนต่อการสูญเสีย node ได้มากที่สุดโดยยังมี quorum

### 3. Automatic Failover (การสลับ Primary อัตโนมัติ)

เมื่อเลือก leader ใหม่ได้แล้ว ระบบต้องทำขั้นตอนต่อไปนี้โดยอัตโนมัติ:

1. Promote standby ที่ถูกเลือกให้เป็น primary ใหม่ (เทียบเท่า `pg_ctl promote` ที่เราทำ manual ใน Part 063 แต่ทำโดยซอฟต์แวร์)
2. Reconfigure standby ตัวอื่น ๆ ที่เหลือให้ replicate จาก primary ใหม่ (cascading reconfiguration)
3. Fence (ตัดขาด) primary เก่าไม่ให้กลับมารับ write ได้อีก แม้ว่ามันจะฟื้นตัวกลับมาในภายหลัง (ป้องกัน split-brain)
4. บันทึก event/history ของการ failover เพื่อการตรวจสอบย้อนหลัง (audit)

### 4. Client Redirection (การเปลี่ยนเส้นทาง Client)

หลังจากมี primary ใหม่แล้ว แอปพลิเคชันต้องสามารถ **หา primary ใหม่เจอโดยอัตโนมัติ** โดยไม่ต้องแก้ connection string เอง วิธีที่นิยมใช้:

- **Virtual IP (VIP)** — IP address ลอยที่ถูกย้ายไปผูกกับเครื่องที่เป็น primary ปัจจุบัน (ใช้ keepalived หรือ Pacemaker)
- **DNS-based** — อัปเดต DNS record ให้ชี้ไป primary ใหม่ (มักช้าเพราะ DNS caching/TTL)
- **Load Balancer / Proxy** — HAProxy หรือ PgBouncer ที่ query สถานะ node ผ่าน health check endpoint แล้ว route traffic ไปยัง node ที่เป็น primary ปัจจุบันเท่านั้น (วิธีที่นิยมที่สุดร่วมกับ Patroni — จะอธิบายเพิ่มใน Step 648 และเจาะลึกใน Part 066-067)
- **Client-side multi-host connection string** — libpq รองรับ `target_session_attrs=read-write` ร่วมกับ multi-host connection string ทำให้ driver ไล่ลองแต่ละ host จนเจอตัวที่รับ write ได้เอง

องค์ประกอบทั้ง 4 นี้ต้องทำงานร่วมกันอย่างแนบเนียน — ถ้าขาดตัวใดตัวหนึ่งไป ระบบ HA จะไม่สมบูรณ์ เช่น มี automatic failover แต่ไม่มี client redirection ก็ไร้ประโยชน์เพราะแอปยังต่อไปหา primary เก่าที่ตายไปแล้วอยู่ดี

---

## Step 644: repmgr — เครื่องมือ HA แบบดั้งเดิมสำหรับ PostgreSQL

### ภาพรวม

**repmgr** (Replication Manager) เป็นเครื่องมือ open-source ที่พัฒนาโดย 2ndQuadrant (ปัจจุบันเป็นส่วนหนึ่งของ EDB — EnterpriseDB) เป็นหนึ่งในเครื่องมือ HA ที่ **เก่าแก่และเป็นที่นิยมที่สุด** สำหรับ PostgreSQL มาตั้งแต่ยุค PostgreSQL 9.x

repmgr ทำงาน **บนฐานของ streaming replication ดั้งเดิม** ที่ PostgreSQL มีอยู่แล้ว (ที่เราเรียนใน Part 062-063) โดยเพิ่มชั้นการจัดการ (management layer) ครอบไว้ เพื่อ:

- ลงทะเบียนและติดตามสถานะของทุก node ในคลัสเตอร์
- ช่วย clone standby ใหม่จาก primary ได้ง่ายขึ้น (`repmgr standby clone`)
- ตรวจจับความล้มเหลวของ primary และทำ automatic failover (ผ่าน daemon `repmgrd`)
- รองรับ manual switchover แบบ zero-downtime

### สถาปัตยกรรมของ repmgr

repmgr ประกอบด้วย 2 ส่วนหลัก:

```
┌────────────────────────────────────────────────────────────┐
│                     repmgr Architecture                      │
│                                                                │
│   ┌─────────────┐         ┌─────────────┐                   │
│   │  repmgr CLI  │         │  repmgrd     │                   │
│   │  (คำสั่ง      │         │  (daemon     │                   │
│   │   manual)    │         │   ตรวจจับ+    │                   │
│   │              │         │   failover   │                   │
│   └──────┬───────┘         │   อัตโนมัติ)  │                   │
│          │                 └──────┬───────┘                   │
│          │                        │                            │
│          ▼                        ▼                            │
│   ┌──────────────────────────────────────┐                   │
│   │   repmgr metadata schema              │                   │
│   │   (เก็บใน database ชื่อ repmgr         │                   │
│   │    บน primary node — replicate        │                   │
│   │    ไปทุก standby ตามปกติ)              │                   │
│   └──────────────────────────────────────┘                   │
└────────────────────────────────────────────────────────────┘
```

1. **repmgr (CLI tool)** — คำสั่งที่ DBA ใช้ manual เช่น การ register node, clone standby, สั่ง switchover, หรือ promote — ทำหน้าที่คล้ายกับที่เราทำ manual ใน Part 063 แต่มีคำสั่งสำเร็จรูปให้ใช้แทนการเขียน script เอง

2. **repmgrd (daemon)** — ตัว daemon ที่รันค้างอยู่บนทุก node คอยตรวจสอบสถานะ primary ผ่านการ connect โดยตรง (ไม่ใช้ DCS แยกต่างหากแบบ Patroni) เมื่อพบว่า primary ไม่ตอบสนองเกินเวลาที่กำหนด จะเริ่มกระบวนการ election ในหมู่ node ที่เหลือเพื่อเลือก standby ที่เหมาะสมที่สุดมา promote

repmgr เก็บ metadata ของคลัสเตอร์ (รายชื่อ node, role, สถานะ) ไว้ใน schema พิเศษชื่อ `repmgr` ภายใน database ที่กำหนด (มักตั้งชื่อ `repmgr`) ซึ่งอยู่บน primary และถูก replicate ไปยัง standby ตามกลไก streaming replication ปกติ

### การตั้งค่าเบื้องต้น (Conceptual Example)

> เนื้อหาต่อไปนี้เป็นตัวอย่างไฟล์คอนฟิกที่ถูกต้องตาม syntax จริงของ repmgr แต่จำลองสถานการณ์คลัสเตอร์ 3 โหนด (`node1` = primary, `node2`/`node3` = standby) ซึ่งไม่สามารถรันจริงในสภาพแวดล้อมเรียนนี้ที่มีเครื่องเดียว

ไฟล์ `repmgr.conf` บน primary (`node1`):

```ini
# /etc/repmgr.conf บน node1 (primary)

node_id=1
node_name='node1'
conninfo='host=node1.internal dbname=repmgr user=repmgr connect_timeout=2'
data_directory='/var/lib/postgresql/16/main'

# ตำแหน่งไฟล์ binary ของ PostgreSQL
pg_bindir='/usr/lib/postgresql/16/bin'

# log
log_level=INFO
log_file='/var/log/repmgr/repmgr.log'

# failover mode: automatic ให้ repmgrd จัดการเอง
failover=automatic

# คำสั่งที่ repmgrd เรียกใช้เพื่อ promote node นี้เป็น primary
promote_command='repmgr standby promote -f /etc/repmgr.conf --log-to-file'

# คำสั่งที่ repmgrd เรียกใช้เพื่อให้ node นี้ follow primary ใหม่
follow_command='repmgr standby follow -f /etc/repmgr.conf --log-to-file --upstream-node-id=%n'

# ตั้งค่า monitoring interval (วินาที)
monitor_interval_secs=2

# จำนวนครั้งและ interval ที่ repmgrd retry ก่อนยืนยันว่า primary ตายจริง
reconnect_attempts=6
reconnect_interval=10

# ป้องกัน node ที่มี replication lag เยอะเกินไปถูกเลือกเป็น primary ใหม่
priority=100
```

ไฟล์ `repmgr.conf` บน standby (`node2`) จะคล้ายกัน แต่เปลี่ยน `node_id`, `node_name`, `conninfo` และอาจตั้ง `priority` ต่างกันเพื่อกำหนดลำดับความสำคัญในการถูกเลือกเป็น primary (ค่ามากกว่า = ความสำคัญสูงกว่า, ตั้ง `priority=0` เพื่อห้าม node นั้นถูก promote เลย เช่น node ที่อยู่ไกล region หรือใช้เพื่อ reporting เท่านั้น)

### Witness Node

ในคลัสเตอร์ที่มีเพียง 2 node (primary + 1 standby) repmgr แนะนำให้เพิ่ม **witness node** เป็น node ที่ 3 ที่ไม่มี PostgreSQL data จริง แต่ทำหน้าที่เป็นเสียงโหวตเพิ่มเติมเพื่อช่วยตัดสิน quorum ป้องกันปัญหา "2 node เห็นไม่ตรงกันแล้วตัดสินใจไม่ได้" (เพราะ 2 node ไม่มีทาง achieve majority ที่ชัดเจนถ้าแบ่งเป็นฝ่ายละ 1)

### คำสั่งพื้นฐานที่ใช้บ่อย (สรุปไว้เพื่ออ้างอิง)

```bash
# ลงทะเบียน primary node เข้าคลัสเตอร์ (รันบน node1)
repmgr primary register -f /etc/repmgr.conf

# clone standby จาก primary (รันบน node2 ก่อนเริ่ม PostgreSQL)
repmgr standby clone -h node1.internal -U repmgr -d repmgr -f /etc/repmgr.conf

# ลงทะเบียน standby เข้าคลัสเตอร์ (รันบน node2 หลัง PostgreSQL start แล้ว)
repmgr standby register -f /etc/repmgr.conf

# ตรวจสอบสถานะคลัสเตอร์ทั้งหมด
repmgr cluster show

# ทำ switchover แบบวางแผนล่วงหน้า (planned, zero-downtime)
repmgr standby switchover -f /etc/repmgr.conf

# เริ่ม/หยุด repmgrd daemon
repmgrd -f /etc/repmgr.conf -d
repmgr daemon stop -f /etc/repmgr.conf
```

### ข้อดี / ข้อจำกัดของ repmgr

**ข้อดี:**
- เรียนรู้ง่าย แนวคิดใกล้เคียงกับ streaming replication ดั้งเดิมที่เราเรียนมา
- Mature และผ่านการใช้งานจริงมานาน (production-proven)
- ไม่ต้องพึ่งพา external consensus store (ไม่ต้องติดตั้ง etcd/Consul เพิ่ม)
- เหมาะกับทีมที่ต้องการ HA แบบ "PostgreSQL-native" ไม่อยากเพิ่ม moving part มาก

**ข้อจำกัด:**
- Consensus mechanism ของ repmgrd เอง (แบบ voting ระหว่าง node โดยตรง) **ไม่แข็งแกร่งเท่า Raft/Paxos** ที่ etcd/Consul ใช้ — เสี่ยงต่อ split-brain มากกว่าถ้า network partition ซับซ้อน
- ไม่มี REST API ในตัวสำหรับให้ load balancer เช็คสถานะได้ง่ายเหมือน Patroni (ต้องพึ่ง script เสริมหรือ third-party integration)
- การจัดการ configuration แบบ dynamic (เปลี่ยนค่า PostgreSQL parameters พร้อมกันทั้งคลัสเตอร์) ทำได้จำกัดกว่า Patroni

---

## Step 645: Patroni — เครื่องมือ HA สมัยใหม่ที่ใช้ Distributed Consensus Store

### ภาพรวม

**Patroni** เป็นเครื่องมือ HA สำหรับ PostgreSQL ที่พัฒนาโดย Zalando (ภายหลังมีชุมชน open-source ขนาดใหญ่ดูแลต่อ) เขียนด้วยภาษา Python ปัจจุบันถือเป็น **มาตรฐานอุตสาหกรรม (de facto standard)** สำหรับการทำ PostgreSQL HA ในสภาพแวดล้อม cloud-native และ Kubernetes

ความแตกต่างที่สำคัญที่สุดระหว่าง Patroni กับ repmgr คือ **Patroni ไม่ได้เขียนกลไก consensus/leader-election ขึ้นมาเอง** แต่เลือกที่จะ **มอบหน้าที่นั้นให้กับ Distributed Consensus Store (DCS)** ที่มีอยู่แล้วและผ่านการพิสูจน์ทาง distributed systems theory มาอย่างเข้มงวด เช่น:

- **etcd** — พัฒนาโดย CoreOS (ปัจจุบันอยู่ใต้ CNCF) ใช้อัลกอริทึม **Raft consensus** เป็น DCS ที่ Patroni นิยมใช้มากที่สุด (และเป็นตัวเดียวกับที่ Kubernetes ใช้เก็บ cluster state)
- **Consul** — พัฒนาโดย HashiCorp ก็ใช้ Raft เช่นกัน มี feature เสริมเช่น service discovery ในตัว
- **ZooKeeper** — เครื่องมือ consensus ดั้งเดิมจากโลก Hadoop/Kafka ใช้อัลกอริทึม ZAB (คล้าย Paxos)
- **Kubernetes API** — เมื่อรันบน Kubernetes, Patroni สามารถใช้ Kubernetes Endpoints/ConfigMap เป็น DCS ได้เลยโดยไม่ต้องติดตั้ง etcd แยก (เพราะ Kubernetes เองก็ใช้ etcd อยู่แล้วภายใน)

### แนวคิดหลัก: "อย่าคิดค้น Consensus ใหม่"

ทีมพัฒนา Patroni มองว่าปัญหา distributed consensus (การทำให้ node หลายตัวเห็นพ้องกันเรื่อง leader ท่ามกลาง network partition) เป็นปัญหาที่ยากมากทาง computer science และมีการวิจัย/พิสูจน์ correctness มาอย่างละเอียดแล้วในอัลกอริทึมอย่าง Raft และ Paxos การพยายามเขียนกลไกนี้ขึ้นมาเองใหม่ (อย่างที่ repmgrd ทำในระดับหนึ่ง) มีความเสี่ยงสูงที่จะมี edge case ที่ไม่ถูกคิดถึง โดยเฉพาะกรณี network partition ที่ซับซ้อน

Patroni จึงเลือกทำหน้าที่เป็น **"ตัวจัดการ PostgreSQL instance ที่ฉลาด"** — คอยควบคุม start/stop/promote/reconfigure ของ PostgreSQL บนเครื่องนั้น ๆ ตามสถานะที่อ่านได้จาก DCS ในขณะที่ปัญหา "ใครคือ leader" ถูกมอบให้ DCS จัดการทั้งหมด

### หลักการทำงานคร่าว ๆ

```
┌──────────────────────────────────────────────────────────────┐
│                                                                  │
│   Node 1              Node 2              Node 3               │
│   ┌──────────┐        ┌──────────┐        ┌──────────┐         │
│   │ Patroni  │        │ Patroni  │        │ Patroni  │         │
│   │  wraps   │        │  wraps   │        │  wraps   │         │
│   │PostgreSQL│        │PostgreSQL│        │PostgreSQL│         │
│   └────┬─────┘        └────┬─────┘        └────┬─────┘         │
│        │                   │                    │               │
│        │  แข่งกันขอ leader lock (TTL-based)      │               │
│        └───────────┬───────┴────────────────────┘               │
│                     ▼                                            │
│         ┌───────────────────────────┐                           │
│         │  DCS Cluster (etcd/Consul) │                           │
│         │  เก็บ: leader key, config,  │                           │
│         │  cluster state, history    │                           │
│         │  (3-node, ใช้ Raft consensus)│                          │
│         └───────────────────────────┘                           │
└──────────────────────────────────────────────────────────────┘
```

หลักการคือแต่ละ Patroni instance จะพยายาม **สร้าง key พิเศษใน DCS ที่มีอายุจำกัด (TTL — Time To Live)** เรียกว่า "leader lock" หรือ "leader key" — node ที่ได้ key นี้ไปก่อนจะกลายเป็น leader และต้อง **renew (ต่ออายุ)** key นี้อย่างสม่ำเสมอ (ตามค่า `loop_wait`) ถ้า leader หยุด renew (เพราะ crash, network ขาด, หรือเหตุผลอื่น) key จะหมดอายุตาม TTL แล้ว node อื่นที่เหลือจะแข่งกันสร้าง key ใหม่ — ตัวที่ทำสำเร็จก่อนจะกลายเป็น leader คนใหม่โดยอัตโนมัติ

กลไก TTL-based lock นี้ทำงานถูกต้องได้เพราะ DCS (etcd/Consul) รับประกัน **atomicity และ consistency** ของการเขียน key ผ่านอัลกอริทึม Raft — ไม่มีทางที่ node สองตัวจะได้ key เดียวกันพร้อมกันได้ (ตราบใดที่ DCS cluster เองมี quorum สมบูรณ์)

รายละเอียดเชิงลึกของสถาปัตยกรรมนี้ — รวมถึง REST API, watch mechanism และ loop การทำงานภายใน — จะอธิบายต่อใน Step 646

---

## Step 646: สถาปัตยกรรม Patroni เชิงลึก

### ภาพรวมองค์ประกอบ

Patroni ที่รันบนแต่ละ node ประกอบด้วย 3 ส่วนที่ทำงานร่วมกัน:

```
┌───────────────────────────────────────────────────────────────┐
│                    Patroni Process (per node)                    │
│                                                                    │
│   ┌───────────────┐   ┌───────────────┐   ┌──────────────┐      │
│   │  REST API      │   │  State Machine │   │  PostgreSQL   │      │
│   │  Server        │◀─▶│  (main loop)   │◀─▶│  Controller   │      │
│   │  (port 8008)   │   │                │   │  (start/stop/ │      │
│   │                │   │                │   │   promote/    │      │
│   │                │   │                │   │   reload)     │      │
│   └───────┬───────┘   └───────┬───────┘   └──────┬───────┘      │
│           │                   │                    │              │
└───────────┼───────────────────┼────────────────────┼──────────────┘
            │                   │                    │
            ▼                   ▼                    ▼
     ┌─────────────┐    ┌──────────────┐    ┌──────────────┐
     │ HAProxy /    │    │     DCS       │    │  local        │
     │ PgBouncer    │    │ (etcd/Consul) │    │  PostgreSQL    │
     │ (health      │    │               │    │  instance      │
     │  check)      │    │               │    │                │
     └─────────────┘    └──────────────┘    └──────────────┘
```

### 1. REST API

Patroni แต่ละ node เปิด HTTP server (ค่าเริ่มต้น port 8008) ที่ให้ข้อมูลสถานะและรับคำสั่งควบคุมได้ endpoint ที่สำคัญ ได้แก่:

| Endpoint | Method | ความหมาย |
|---|---|---|
| `/health` หรือ `/` | GET | ตรวจสอบว่า Patroni ยังทำงานอยู่ (สำหรับ generic health check) |
| `/master` หรือ `/primary` | GET | ตอบ HTTP 200 ถ้า node นี้เป็น primary ปัจจุบัน (200 = ใช่, 503 = ไม่ใช่) — **นี่คือ endpoint ที่ HAProxy ใช้เช็คว่าจะ route traffic ไปหา node ไหน** |
| `/replica` | GET | ตอบ HTTP 200 ถ้า node นี้เป็น replica ที่พร้อมรับ read query |
| `/leader` | GET | คล้าย `/master` แต่ใช้คำศัพท์ leader (เป็นกลางไม่ผูกกับ terminology primary/master) |
| `/patroni` | GET | คืนข้อมูล state แบบละเอียด (JSON) ของ node นี้ |
| `/config` | GET/PATCH | อ่านหรือแก้ dynamic configuration ของคลัสเตอร์ |
| `/failover` | POST | สั่ง failover แบบ manual (unplanned scenario, ไม่สนใจว่า primary ปัจจุบันยัง healthy หรือไม่) |
| `/switchover` | POST | สั่ง switchover แบบ planned (zero-downtime, รอให้ primary เก่า sync เสร็จก่อน) |

การมี REST API ในตัวนี้เป็นจุดต่างสำคัญจาก repmgr — มันทำให้ Load Balancer เช่น HAProxy สามารถเช็คสถานะ node ได้ง่ายมาก โดยไม่ต้อง query SQL หรือ parse output อะไรซับซ้อน (รายละเอียดใน Step 648)

### 2. DCS (Distributed Configuration Store) — บทบาทและ Watch Mechanism

DCS ทำหน้าที่เป็น "แหล่งความจริงเดียว" (single source of truth) ของสถานะคลัสเตอร์ทั้งหมด เก็บข้อมูลสำคัญไว้เป็น key-value เช่น (ตัวอย่างโครงสร้าง key ใน etcd ที่ Patroni ใช้):

```
/service/<scope>/leader            → ชื่อ node ที่เป็น leader ปัจจุบัน (พร้อม TTL lease)
/service/<scope>/members/<node1>   → ข้อมูลสถานะของ node1 (role, timeline, lsn, api_url)
/service/<scope>/members/<node2>   → ข้อมูลสถานะของ node2
/service/<scope>/members/<node3>   → ข้อมูลสถานะของ node3
/service/<scope>/config            → dynamic configuration ของคลัสเตอร์ (ตั้งผ่าน /config API)
/service/<scope>/history            → ประวัติการ failover/switchover ทั้งหมด (timeline history)
/service/<scope>/initialize         → flag ว่าคลัสเตอร์ผ่านการ bootstrap แล้ว
```

`<scope>` คือชื่อคลัสเตอร์ที่กำหนดใน `patroni.yml` (parameter `scope`) — ทำให้ DCS ตัวเดียวสามารถเก็บสถานะของหลายคลัสเตอร์พร้อมกันได้โดยแยก namespace กันด้วย scope

**Watch Mechanism**: Patroni แต่ละ node ไม่ได้ poll DCS ซ้ำ ๆ แบบ busy-loop เพียงอย่างเดียว แต่ใช้กลไก **watch** ที่ etcd/Consul มีให้ในตัว — คือการ subscribe เพื่อรับ notification ทันทีเมื่อ key ที่สนใจ (เช่น `/leader`) มีการเปลี่ยนแปลง วิธีนี้ทำให้ node ที่เหลือรับรู้ได้เกือบจะทันทีเมื่อ leader key หายไป (หมดอายุหรือถูกลบ) แทนที่จะต้องรอรอบ poll ถัดไป ช่วยลด failover time ลงได้มาก

### 3. Main Loop (State Machine) — `loop_wait`

Patroni มี main loop ที่ทำงานซ้ำทุก ๆ `loop_wait` วินาที (ค่า default คือ 10 แต่ production มักตั้งไว้ต่ำกว่านั้น เช่น 5) ในแต่ละรอบจะ:

1. เช็คสถานะ local PostgreSQL (ทำงานอยู่ไหม, เป็น primary หรือ replica, ล่าสุด replicate ไปถึงไหน)
2. เช็คสถานะจาก DCS (ใครเป็น leader ตอนนี้, key leader ยังไม่หมดอายุใช่ไหม)
3. ถ้า node ตัวเองเป็น leader → renew (ต่ออายุ) leader key ใน DCS
4. ถ้า node ตัวเองไม่ใช่ leader → ตรวจสอบว่าควร follow leader ปัจจุบันต่อไป หรือถ้า leader key หายไป (ไม่มีใครถือ) → พยายามแข่งสร้าง leader key ใหม่ (เข้าสู่กระบวนการ election)
5. Apply configuration changes ถ้ามีการเปลี่ยน dynamic config ผ่าน DCS

พารามิเตอร์สำคัญที่ควบคุมความไวของกลไกนี้ (อยู่ใน `bootstrap.dcs` ของ `patroni.yml`):

- **`ttl`** (default 30 วินาที) — อายุของ leader lock; ถ้า leader ไม่ renew ภายในเวลานี้ ถือว่า leader ตาย
- **`loop_wait`** (default 10 วินาที) — ความถี่ของ main loop
- **`retry_timeout`** (default 10 วินาที) — เวลาที่ยอมให้ retry การเชื่อมต่อ DCS หรือ PostgreSQL ก่อนถือว่า fail

ความสัมพันธ์ของค่าทั้งสามนี้สำคัญมาก: **`ttl` ต้องมากกว่า `loop_wait` เสมอ** (โดยทั่วไปแนะนำ `ttl` ≥ 3×`loop_wait`) เพื่อให้มีโอกาส renew สำเร็จหลายครั้งก่อนหมดอายุจริง ถ้าตั้งค่าใกล้กันเกินไป ระบบอาจเกิด "false positive failover" คือคิดว่า leader ตายทั้งที่จริงแค่ network ช้าชั่วขณะ ซึ่งเป็นสาเหตุหนึ่งของ split-brain (ดู Step 649)

---

## Step 647: การตั้งค่า Patroni เบื้องต้น — patroni.yml

ไฟล์คอนฟิกหลักของ Patroni คือ `patroni.yml` (YAML format) ต่อไปนี้คือตัวอย่างที่ครบทุก section พร้อมคำอธิบายละเอียด — จำลองว่าเป็นการตั้งค่า `node1` ในคลัสเตอร์ 3 โหนดที่ใช้ etcd เป็น DCS

> **ย้ำอีกครั้ง**: นี่คือตัวอย่างไฟล์คอนฟิกที่ syntax ถูกต้องตรงตาม Patroni เวอร์ชันปัจจุบัน แต่เป็นเนื้อหาเชิงสถาปัตยกรรม — ไม่มีการรันจริงในสภาพแวดล้อมเรียนนี้เพราะต้องมี etcd cluster และ PostgreSQL หลายโหนดจริง

```yaml
# /etc/patroni/patroni.yml — ตัวอย่างสำหรับ node1

# ===== ค่าระบุตัวตนของคลัสเตอร์และ node =====
scope: ecommerce-cluster        # ชื่อคลัสเตอร์ (namespace ใน DCS) ทุกโหนดในคลัสเตอร์เดียวกันต้องตรงกัน
namespace: /db/                 # namespace เพิ่มเติมใน DCS (ทางเลือก, ใช้แยกหลาย environment)
name: node1                     # ชื่อเฉพาะของ node นี้ ต้องไม่ซ้ำกันในคลัสเตอร์

# ===== REST API ที่ Patroni เปิดให้ HAProxy/DCS เช็คสถานะ =====
restapi:
  listen: 10.0.1.11:8008        # IP:port ที่ REST API รับฟัง
  connect_address: 10.0.1.11:8008
  # (ทางเลือก) เปิด TLS สำหรับ REST API ใน production จริง
  # certfile: /etc/patroni/certs/server.crt
  # keyfile: /etc/patroni/certs/server.key
  # authentication:
  #   username: admin
  #   password: changeme_use_vault

# ===== การเชื่อมต่อ DCS (ในที่นี้ใช้ etcd v3) =====
etcd3:
  hosts:
    - 10.0.1.21:2379
    - 10.0.1.22:2379
    - 10.0.1.23:2379
  protocol: https
  cacert: /etc/patroni/certs/etcd-ca.crt

# ===== Bootstrap: ตั้งค่าตอนสร้างคลัสเตอร์ครั้งแรกเท่านั้น =====
bootstrap:
  # dcs: ค่าที่จะถูกเขียนลง DCS เป็น dynamic config ตอน bootstrap
  # หลังจากนั้นการแก้ไขค่าเหล่านี้ต้องทำผ่าน `patronictl edit-config` ไม่ใช่แก้ไฟล์นี้ตรง ๆ
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576   # 1 MB — ห้าม promote standby ที่ lag เกินนี้
    master_start_timeout: 300
    synchronous_mode: false             # true = บังคับ synchronous replication (ดู Step 649)
    postgresql:
      use_pg_rewind: true
      use_slots: true                   # ใช้ replication slots ป้องกัน WAL ถูกลบก่อน standby อ่านทัน
      parameters:
        max_connections: 200
        shared_buffers: 4GB
        wal_level: replica
        hot_standby: "on"
        wal_keep_size: 1GB
        max_wal_senders: 10
        max_replication_slots: 10
        checkpoint_completion_target: 0.9
        archive_mode: "on"
        archive_command: "wal-g wal-push %p"

  # initdb: options ที่ใช้ตอน initdb ครั้งแรก (เฉพาะ node ที่ bootstrap คลัสเตอร์ก่อนใคร)
  initdb:
    - encoding: UTF8
    - data-checksums
    - locale: en_US.UTF-8

  # pg_hba: กฎ pg_hba.conf ที่ Patroni จะ generate ให้อัตโนมัติ
  pg_hba:
    - host replication replicator 10.0.1.0/24 scram-sha-256
    - host all all 10.0.1.0/24 scram-sha-256
    - host all all 0.0.0.0/0 reject

# ===== การตั้งค่า PostgreSQL instance บนเครื่องนี้ =====
postgresql:
  listen: 10.0.1.11:5432
  connect_address: 10.0.1.11:5432
  data_dir: /var/lib/postgresql/16/main
  bin_dir: /usr/lib/postgresql/16/bin
  pgpass: /tmp/pgpass_patroni

  authentication:
    replication:
      username: replicator
      password: "${REPLICATION_PASSWORD}"   # แนะนำอ่านจาก env var / vault ไม่ hardcode
    superuser:
      username: postgres
      password: "${SUPERUSER_PASSWORD}"
    rewind:                                  # ใช้สำหรับ pg_rewind ตอน re-join คลัสเตอร์
      username: rewind_user
      password: "${REWIND_PASSWORD}"

  # parameters เฉพาะ instance นี้ (override ทับค่าจาก bootstrap.dcs.postgresql.parameters ได้)
  parameters:
    unix_socket_directories: '/var/run/postgresql'

  # callback scripts (ทางเลือก) — เรียกเมื่อมี event เช่น on_start, on_stop, on_role_change
  callbacks:
    on_role_change: /etc/patroni/callbacks/notify_role_change.sh

  # create_replica_methods: วิธีที่ standby ใหม่จะ clone ข้อมูลจาก leader
  create_replica_methods:
    - basebackup
  basebackup:
    max-rate: '100M'
    checkpoint: fast

# ===== Tags: กำหนดพฤติกรรมพิเศษของ node นี้ =====
tags:
  nofailover: false      # true = ห้าม node นี้ถูกเลือกเป็น primary ใหม่เด็ดขาด
  noloadbalance: false   # true = ห้าม load balancer route read query มาที่ node นี้
  clonefrom: true        # true = node นี้เป็นแหล่งอ้างอิงที่ standby ใหม่ควร clone จาก node นี้ก่อน
  nosync: false          # true = ห้ามเลือก node นี้เป็น synchronous standby
```

### อธิบายแต่ละ Section เพิ่มเติม

**`scope` / `name`** — เป็นค่าที่สำคัญที่สุด `scope` ต้องตรงกันทุก node เพื่อให้ Patroni มองว่าอยู่คลัสเตอร์เดียวกัน ส่วน `name` ต้องไม่ซ้ำ (เทียบได้กับ `node_id`/`node_name` ใน repmgr.conf)

**`restapi`** — Patroni บังคับให้ต้องเปิด REST API เสมอ (ไม่ใช่ optional feature) เพราะทั้ง Patroni node อื่น ๆ (เวลาทำ switchover/failover ผ่าน `patronictl`) และ HAProxy ต่างพึ่งพา endpoint นี้

**`etcd3`** (หรือ `consul`, `zookeeper`, `kubernetes` แล้วแต่เลือกใช้ DCS ตัวไหน) — ต้องระบุรายชื่อ endpoint ของ DCS cluster ทั้งหมด ไม่ใช่แค่ตัวเดียว เพื่อให้ Patroni ทนต่อการที่ DCS node บางตัวล่มได้ (fault tolerance ระดับ client)

**`bootstrap.dcs`** — นี่คือจุดที่มักสร้างความสับสน: ค่าพวกนี้ **ใช้แค่ตอน initialize คลัสเตอร์ครั้งแรกเท่านั้น** หลังจากนั้นค่าจะถูกเก็บเป็น dynamic config ใน DCS และการแก้ไขในอนาคตต้องทำผ่านคำสั่ง `patronictl edit-config` เท่านั้น (แก้ไฟล์ `patroni.yml` แล้ว restart Patroni **จะไม่มีผล** กับค่าพวกนี้อีกต่อไป)

**`postgresql.parameters`** — พารามิเตอร์ PostgreSQL ระดับ instance ที่ Patroni จะเขียนลง `postgresql.conf` ให้อัตโนมัติ ข้อดีคือทำให้ configuration ของทุก node ในคลัสเตอร์ sync กันได้ง่ายผ่าน dynamic config (แก้ที่เดียว กระจายไปทุก node)

**`tags`** — ใช้ควบคุมพฤติกรรมพิเศษราย node เช่น node ที่อยู่ region ไกล (DR site) อาจตั้ง `nofailover: true` เพื่อไม่ให้ถูกเลือกเป็น primary โดยไม่ตั้งใจ (เพราะ latency สูงจะกระทบ synchronous replication) หรือ node ที่ใช้สำหรับ analytical query หนัก ๆ อาจตั้ง `noloadbalance: true` เพื่อกันไม่ให้ traffic ปกติไปกองที่ node นั้น

### คำสั่งควบคุมที่ใช้บ่อย (patronictl)

```bash
# ดูสถานะคลัสเตอร์ทั้งหมด (แสดง role, lag, state ของทุก node)
patronictl -c /etc/patroni/patroni.yml list

# แก้ dynamic configuration (เปิด editor ให้แก้ YAML แล้ว apply ทั้งคลัสเตอร์)
patronictl -c /etc/patroni/patroni.yml edit-config

# สั่ง switchover แบบ planned (เลือก node ปลายทางได้)
patronictl -c /etc/patroni/patroni.yml switchover --master node1 --candidate node2

# สั่ง failover แบบ manual/emergency
patronictl -c /etc/patroni/patroni.yml failover

# ดูประวัติ timeline/failover history
patronictl -c /etc/patroni/patroni.yml history

# Pause/Resume — หยุด Patroni ไม่ให้ทำ automatic failover ชั่วคราว (เช่นตอน maintenance)
patronictl -c /etc/patroni/patroni.yml pause
patronictl -c /etc/patroni/patroni.yml resume
```

`patronictl pause` มีความสำคัญมากในทางปฏิบัติ — เวลาต้อง maintenance เช่น upgrade OS หรือทำ manual restart ของ PostgreSQL node ใดตัวหนึ่ง ถ้าไม่ pause ก่อน Patroni อาจตีความว่า node นั้น "ตาย" แล้วเริ่มกระบวนการ failover โดยไม่ตั้งใจ

---

## Step 648: HAProxy / PgBouncer ร่วมกับ Patroni สำหรับ Routing Traffic

เมื่อมี automatic failover แล้ว ปัญหาถัดมาคือ **แอปพลิเคชันจะรู้ได้อย่างไรว่า primary ตัวปัจจุบันคือใคร** เพราะ IP ของ primary เปลี่ยนไปทุกครั้งที่เกิด failover/switchover

นี่คือจุดที่ **HAProxy** เข้ามาเสริม Patroni ได้อย่างลงตัว เพราะ Patroni มี REST API endpoint `/master` (หรือ `/primary`) ที่ตอบ HTTP 200 เฉพาะ node ที่เป็น primary เท่านั้น — HAProxy ใช้กลไก `httpchk` เพื่อ probe endpoint นี้ทุก node แล้ว route traffic ไปยัง node ที่ตอบ 200 เท่านั้น

### ภาพสถาปัตยกรรมคร่าว ๆ

```
                        ┌──────────────┐
        Application ───▶│   HAProxy     │
                        │  (port 5000  │
                        │   = primary  │
                        │   port 5001  │
                        │   = replica) │
                        └──────┬───────┘
                               │ httpchk GET /master ทุก node ทุก 3 วินาที
              ┌────────────────┼────────────────┐
              ▼                ▼                ▼
      ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
      │ node1         │ │ node2         │ │ node3         │
      │ Patroni:8008  │ │ Patroni:8008  │ │ Patroni:8008  │
      │ /master → 200 │ │ /master → 503 │ │ /master → 503 │
      │ (เป็น primary) │ │ (เป็น replica)│ │ (เป็น replica)│
      │ PostgreSQL    │ │ PostgreSQL    │ │ PostgreSQL    │
      │ :5432         │ │ :5432         │ │ :5432         │
      └──────────────┘ └──────────────┘ └──────────────┘
```

ตัวอย่างส่วนสำคัญของ `haproxy.cfg` (แนวคิดเบื้องต้น — จะลงรายละเอียดเต็มใน Part 066-067):

```
# haproxy.cfg (แสดงเฉพาะส่วนที่เกี่ยวกับ Patroni backend)

listen postgres_primary
    bind *:5000
    option httpchk GET /master
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server node1 10.0.1.11:5432 maxconn 100 check port 8008
    server node2 10.0.1.12:5432 maxconn 100 check port 8008
    server node3 10.0.1.13:5432 maxconn 100 check port 8008

listen postgres_replicas
    bind *:5001
    option httpchk GET /replica
    http-check expect status 200
    balance roundrobin
    default-server inter 3s fall 3 rise 2
    server node1 10.0.1.11:5432 maxconn 100 check port 8008
    server node2 10.0.1.12:5432 maxconn 100 check port 8008
    server node3 10.0.1.13:5432 maxconn 100 check port 8008
```

หลักการสำคัญ: HAProxy เปิดสอง listener แยกกัน — **port 5000 สำหรับ write traffic** (route ไปเฉพาะ node ที่ `/master` ตอบ 200 คือ primary ปัจจุบันเท่านั้น ซึ่งจะมีแค่ node เดียวที่ผ่าน health check ณ เวลาใดเวลาหนึ่ง) และ **port 5001 สำหรับ read traffic** (route แบบ round-robin ไปยังทุก node ที่ `/replica` ตอบ 200 เพื่อกระจาย read load — เทคนิคนี้เรียกว่า read/write splitting ซึ่งจะอธิบายเจาะลึกในบทถัดไป)

เมื่อเกิด failover Patroni จะอัปเดตค่าที่ `/master` ตอบของแต่ละ node ให้ตรงกับสถานะใหม่โดยอัตโนมัติ (ภายในไม่กี่วินาที) ทำให้ HAProxy เปลี่ยนเส้นทาง traffic ตามไปโดยที่ **แอปพลิเคชันไม่ต้องรู้เรื่อง IP ของ primary จริงเลย** — แอปแค่ต่อไปที่ HAProxy port 5000 เสมอ

### PgBouncer ในบทบาทเสริม

**PgBouncer** ซึ่งเป็น connection pooler (จะเรียนเจาะลึกใน Part 066) มักถูกวางไว้ **ต่อจาก HAProxy** อีกชั้นหนึ่ง เพื่อทำ connection pooling ลดภาระจากการเปิด/ปิด connection บ่อย ๆ — บาง setup ก็ใช้ PgBouncer เป็นตัวที่แอปเชื่อมต่อโดยตรง แล้ว PgBouncer ค่อย forward ไปที่ HAProxy อีกที หรือบางองค์กรใช้ `pgbouncer` ร่วมกับ script ที่ query Patroni REST API เพื่อสลับ `[databases]` target แบบ dynamic เอง (ซับซ้อนกว่าการใช้ HAProxy ตรง ๆ)

รายละเอียดวิธีตั้งค่า PgBouncer แบบเต็มรูปแบบ, transaction pooling mode, connection limits ฯลฯ จะอยู่ใน **Part 066: Connection Pooling** และรายละเอียด HAProxy แบบเต็มรวมถึง keepalived สำหรับทำ VIP จะอยู่ใน **Part 067**

---

## Step 649: Split-Brain Problem — ป้องกันด้วย Fencing และ STONITH

### Split-Brain คืออะไร

**Split-brain** คือสถานการณ์อันตรายที่สุดอย่างหนึ่งในระบบ HA — เกิดขึ้นเมื่อ **มี node มากกว่าหนึ่งตัวเชื่อว่าตัวเองเป็น primary พร้อมกัน** และทั้งสองตัวต่างรับ write request จาก client โดยไม่รู้ตัวว่ามีอีกตัวทำงานคู่ขนานอยู่

```
                    ก่อนเกิด Network Partition
                    ┌──────────────────────┐
                    │   node1 (primary)     │
                    │   node2 (standby)     │◀── ทำงานปกติ
                    │   node3 (standby)     │
                    └──────────────────────┘

                    เกิด Network Partition
        ┌──────────────────┐        ┌──────────────────┐
        │  node1 (primary)  │   ╳    │  node2, node3      │
        │  แยกขาดจาก DCS      │        │  ยังเห็น DCS ปกติ   │
        │  majority          │        │  (มี quorum)        │
        └──────────────────┘        └──────────────────┘
                                              │
                                              ▼
                                    node2/node3 เลือก leader ใหม่
                                    (เช่น node2 กลายเป็น primary ใหม่)

        ผลลัพธ์อันตราย: ถ้า node1 ไม่ถูก "fence" ให้หยุดรับ write
        จะกลายเป็นมี primary 2 ตัวพร้อมกัน (node1 เดิม + node2 ใหม่)
        ═══════════ SPLIT BRAIN ═══════════
        แอปบางส่วนอาจยังต่อ node1 อยู่ → เขียนข้อมูลเข้า node1
        แอปบางส่วนต่อ node2 (ใหม่) → เขียนข้อมูลเข้า node2
        ข้อมูลสองฝั่ง "diverge" (แยกทางกัน) ไม่มีทาง merge กลับ
        อัตโนมัติได้ — ต้องแก้ด้วยมือ อาจสูญเสียข้อมูลถาวร
```

### ทำไม Split-Brain ถึงอันตรายมาก

- ข้อมูลที่เขียนเข้า node1 (primary เก่าที่ถูกตัดขาด) กับข้อมูลที่เขียนเข้า node2 (primary ใหม่) จะ **diverge** ออกจากกันในแง่ WAL timeline — ไม่สามารถ merge กลับเป็นสายเดียวกันโดยอัตโนมัติได้ เพราะ PostgreSQL replication เป็นแบบ single-writer, single-source-of-truth
- Transaction ที่ commit สำเร็จในทั้งสองฝั่งอาจขัดแย้งกัน (เช่น order เดียวกันถูกประมวลผลสองครั้ง, primary key ซ้ำ, ยอดเงินคลาดเคลื่อน)
- การกู้คืนต้องอาศัยการ **เลือกฝั่งใดฝั่งหนึ่งเป็น "ความจริง"** แล้วทิ้งอีกฝั่ง (หรือพยายาม manually reconcile ข้อมูลที่ขัดแย้งกัน) ซึ่งมักหมายถึง **สูญเสียข้อมูลถาวร** ของฝั่งที่ถูกทิ้ง

### สาเหตุที่ทำให้เกิด Split-Brain

1. **Network partition** — network ขาดเป็นสองฝั่ง (partition) แต่ทั้งสองฝั่งยังทำงานได้ปกติภายในตัวเอง (คลาสสิกที่สุด)
2. **False positive health check** — health check timeout สั้นเกินไป ทำให้ node ที่ยัง healthy จริงถูกเข้าใจผิดว่าตายแล้ว จึงเริ่ม failover ทั้งที่ node เดิมยังรับ write อยู่
3. **Clock skew / GC pause** — โปรแกรม (เช่น JVM-based tools) หยุดชั่วขณะจาก garbage collection pause นานผิดปกติ ทำให้พลาด renew lease ทั้งที่ process จริงยังไม่ตาย
4. **Split DCS quorum** — ตัว DCS เอง (etcd/Consul) ถ้าไม่มี quorum ที่ชัดเจน (เช่น deploy แค่ 2 node หรือ 4 node ที่แบ่งเท่ากันได้) ก็อาจตัดสินใจผิดพลาดได้เช่นกัน

### วิธีป้องกัน Split-Brain

#### 1. Quorum ที่ถูกต้องเสมอ (Odd Number of DCS Nodes)

ตั้ง DCS cluster เป็นจำนวนคี่เสมอ (3, 5, 7) เพื่อให้มี majority ที่ชัดเจนเสมอเมื่อเกิด partition — เช่น etcd 3 node แบ่งเป็น 1 vs 2 ฝั่งไหนมี 2 โหนดถือว่ามี quorum ฝั่งที่มีแค่ 1 โหนดจะ**ไม่สามารถ**เขียนอะไรลง DCS ได้เลย (รวมถึงไม่สามารถ renew leader lock ได้) — Patroni node ที่อยู่ฝั่ง minority (แม้จะยังคิดว่าตัวเองเป็น primary) จะตรวจพบว่าติดต่อ DCS ไม่ได้ แล้ว **ปฏิเสธที่จะรับ write เอง** (demote ตัวเองหรือปฏิเสธ connection) — นี่คือกลไกป้องกัน split-brain แบบพื้นฐานที่สุดของ Patroni

#### 2. Fencing และแนวคิด STONITH

**Fencing** คือการ "ตัดขาด" node ที่มีปัญหาออกจากระบบโดยสมบูรณ์ เพื่อรับประกันว่ามันจะไม่สามารถรับ write หรือส่งผลกระทบต่อข้อมูลได้อีก แม้ว่ามันจะยังทำงานอยู่ (แค่ network ขาด)

คำว่า **STONITH — "Shoot The Other Node In The Head"** เป็นศัพท์ที่มาจากโลก Linux HA (Pacemaker/Corosync) หมายถึงแนวทาง fencing แบบเข้มข้น: แทนที่จะพยายามสื่อสารกับ node ที่มีปัญหาอย่างสุภาพ ระบบจะ **สั่งปิดเครื่องนั้นทันทีแบบ hard power-off** ผ่านกลไกภายนอก (เช่น IPMI, cloud provider API สั่ง stop VM, PDU ตัดไฟปลั๊ก) เพื่อรับประกัน 100% ว่ามันจะหยุดทำงานจริง ไม่ใช่แค่ "ขอให้หยุด" ผ่าน network ที่อาจไม่ถึงมันอยู่แล้ว

```
แนวคิด STONITH:
┌─────────────┐         Power Off Command        ┌──────────────┐
│ Fencing      │ ─────────(ผ่าน IPMI/Cloud API)──▶│ node1 (ปัญหา) │
│ Controller   │         ไม่ใช่ผ่าน network เดียวกัน │              │
└─────────────┘         ที่ขาดอยู่                  └──────────────┘
```

จุดสำคัญคือ fencing ต้องทำผ่าน **out-of-band channel** ที่ไม่ใช่ network เส้นเดียวกับที่ขาด (เช่น ใช้ IPMI ที่เป็น network การ์ดแยกต่างหาก หรือใช้ cloud provider API ซึ่งเป็นคนละ path จาก application network) มิฉะนั้นถ้า network ที่ขาดเป็นเส้นเดียวกับที่ใช้สั่ง fence คำสั่งก็จะส่งไม่ถึงเช่นกัน

#### 3. Watchdog Device — กลไก Self-Fencing ที่ Patroni รองรับ

Patroni มีกลไกที่เรียกว่า **watchdog** ซึ่งเป็นแนวทาง "self-fencing" — คือให้ node ที่เป็น primary เอง ผูกตัวไว้กับ hardware/software watchdog timer ของ OS (เช่น Linux `softdog` kernel module หรือ hardware watchdog chip บนเมนบอร์ด) โดย Patroni ต้อง "เลี้ยง" (kick/ping) watchdog นี้อย่างสม่ำเสมอตราบใดที่มันยังเชื่อว่าตัวเองเป็น primary ที่ถูกต้องและติดต่อ DCS ได้ปกติ

ถ้า Patroni process หยุดทำงาน หรือติดต่อ DCS ไม่ได้นานเกินกำหนด (เช่นเพราะ network partition) มันจะ**หยุดเลี้ยง watchdog** — และเมื่อ watchdog timer หมดเวลา (โดยไม่ได้รับการ kick) **OS จะสั่ง reboot เครื่องนั้นทันทีโดยอัตโนมัติ** แม้ Patroni process เองจะค้างหรือตายไปแล้วก็ตาม รับประกันว่า primary เก่าที่ถูกตัดขาดจาก network จะถูก "ยิงตาย" ด้วยตัวมันเอง ไม่มีทางรับ write ต่อได้อีก

ตัวอย่างการเปิด watchdog ใน `patroni.yml`:

```yaml
watchdog:
  mode: required          # required = ถ้าไม่มี watchdog device จะไม่ยอม start เป็น primary เลย
  device: /dev/watchdog    # path ของ watchdog device บน Linux
  safety_margin: 5         # วินาทีสำรองที่เผื่อไว้ก่อน timeout จริง
```

การตั้งค่า `mode: required` เป็นแนวทางที่ production ระดับสูงแนะนำ เพราะมันบังคับให้ระบบ "ปลอดภัยไว้ก่อน" — ถ้าไม่มี watchdog hardware/kernel module พร้อมใช้งาน Patroni จะปฏิเสธไม่ยอมให้ node นั้นเป็น primary เลย ดีกว่าเสี่ยงปล่อยให้เกิด split-brain

#### 4. Synchronous Replication ลด Data Loss (ไม่ใช่ป้องกัน Split-Brain โดยตรง แต่เกี่ยวข้อง)

การตั้ง `synchronous_mode: true` ใน `bootstrap.dcs` ทำให้ Patroni บังคับให้ primary ต้องรอ standby อย่างน้อย 1 ตัว (synchronous standby) ยืนยันว่าได้รับ WAL แล้วก่อนจึง commit transaction สำเร็จ — วิธีนี้ทำให้ **RPO = 0** (ไม่มี transaction ไหนสูญหายแม้ primary จะตายกะทันหัน) แต่ **ไม่ได้ป้องกัน split-brain โดยตรง** — มันช่วยแค่ทำให้ข้อมูลของ standby ที่ถูกเลือกเป็น primary ใหม่ complete ที่สุดเท่าที่จะเป็นไปได้ Split-brain ยังคงต้องป้องกันด้วย quorum + fencing/watchdog ตามที่อธิบายข้างต้นเสมอ

Patroni ยังมีตัวช่วยเสริมคือ `maximum_lag_on_failover` (ที่เห็นในตัวอย่าง `patroni.yml` ข้างต้น = 1MB) ซึ่งห้าม standby ที่ replication lag เกินค่านี้ถูกเลือกเป็น primary ใหม่โดยอัตโนมัติ — ลดความเสี่ยงที่จะ promote node ที่ข้อมูลเก่าเกินไป

### สรุปแนวทางป้องกัน Split-Brain แบบครบวงจร

```
Layer 1: Odd-numbered DCS quorum (3/5/7 nodes)     ← ป้องกันพื้นฐานที่สุด
Layer 2: Watchdog self-fencing (softdog/hardware)   ← ป้องกันระดับ OS
Layer 3: External fencing/STONITH (IPMI/cloud API)  ← ป้องกันระดับ infrastructure
Layer 4: Synchronous replication (RPO=0)            ← ลด data loss ถ้าเกิด failover จริง
Layer 5: maximum_lag_on_failover                    ← ป้องกันเลือก standby เก่าเกินไป
```

ระบบ Production ระดับสูงมักใช้หลาย layer พร้อมกัน ไม่พึ่งพา layer เดียว เพราะแต่ละ layer มีจุดอ่อนต่างกัน (เช่น watchdog ป้องกันไม่ได้ถ้า kernel เองค้างทั้งระบบ, external fencing ป้องกันไม่ได้ถ้า fencing controller เองเข้าไม่ถึง node)

---

## Step 650: แบบฝึกหัดรวม — ออกแบบสถาปัตยกรรม HA สำหรับระบบ E-commerce

### โจทย์

บริษัท e-commerce แห่งหนึ่งมี requirement ดังนี้:

- ต้องการ SLA 99.99% (downtime ยอมรับได้ ~52 นาที/ปี)
- RPO ใกล้ 0 มากที่สุด (ห้ามข้อมูล order/payment สูญหาย)
- RTO เป้าหมาย < 30 วินาที
- แอปพลิเคชัน backend เชื่อมต่อผ่าน connection string เดียว ไม่ต้องการแก้โค้ดเวลาเกิด failover
- ต้องรองรับ read replica สำหรับ reporting/analytics แยกจาก transactional load

### แนวทางออกแบบสถาปัตยกรรม

```
┌─────────────────────────────────────────────────────────────────────┐
│                     E-COMMERCE HA ARCHITECTURE                        │
│                                                                         │
│                        ┌───────────────┐                              │
│                        │  Application    │                              │
│                        │  Backend        │                              │
│                        │  (stateless,    │                              │
│                        │   many pods)    │                              │
│                        └───────┬───────┘                              │
│                                │ connect ผ่าน VIP เดียว                 │
│                                ▼                                        │
│                   ┌──────────────────────┐                            │
│                   │  Keepalived VIP        │  ← Active/Passive VIP     │
│                   │  10.0.1.100            │     ลอยระหว่าง HAProxy 2  │
│                   └──────────┬───────────┘     เครื่อง (กันเอง HAProxy │
│                              │                   เป็น single point)    │
│           ┌──────────────────┴──────────────────┐                    │
│           ▼                                      ▼                    │
│   ┌───────────────┐                     ┌───────────────┐            │
│   │ HAProxy #1      │◀── keepalived ──▶│ HAProxy #2      │            │
│   │ (active)        │   VRRP heartbeat  │ (standby)       │            │
│   │ :5000 write      │                   │ :5000 write      │            │
│   │ :5001 read        │                   │ :5001 read        │            │
│   └───────┬───────┘                     └───────┬───────┘            │
│           │  httpchk /master, /replica ทุก node ทุก 3 วินาที           │
│   ┌────────┴──────────────────────────────────────┴───────┐          │
│   ▼                          ▼                          ▼             │
│ ┌────────────┐      ┌────────────┐      ┌────────────┐             │
│ │ pg-node1    │      │ pg-node2    │      │ pg-node3    │             │
│ │ (AZ-1)      │      │ (AZ-2)      │      │ (AZ-3)      │             │
│ │             │      │             │      │             │             │
│ │ Patroni     │◀────▶│ Patroni     │◀────▶│ Patroni     │             │
│ │ :8008       │      │ :8008       │      │ :8008       │             │
│ │ PostgreSQL  │      │ PostgreSQL  │      │ PostgreSQL  │             │
│ │ :5432       │      │ :5432       │      │ :5432       │             │
│ │ watchdog:   │      │ watchdog:   │      │ watchdog:   │             │
│ │  softdog    │      │  softdog    │      │  softdog    │             │
│ └──────┬─────┘      └──────┬─────┘      └──────┬─────┘             │
│        │  leader election / lease (Raft)          │                    │
│        └───────────────────┬───────────────────────┘                    │
│                             ▼                                          │
│               ┌───────────────────────────┐                          │
│               │   etcd cluster (3-node)     │                          │
│               │   etcd-1 (AZ-1)             │                          │
│               │   etcd-2 (AZ-2)             │                          │
│               │   etcd-3 (AZ-3)             │                          │
│               │   (Raft consensus, quorum=2) │                          │
│               └───────────────────────────┘                          │
│                                                                         │
│   WAL archive/backup: wal-g / pgBackRest → Object Storage (S3-compatible)│
│   (สำหรับ PITR และ bootstrap standby ใหม่ใน DR site)                    │
└─────────────────────────────────────────────────────────────────────┘
```

### เหตุผลเบื้องหลังการออกแบบแต่ละส่วน

**1. 3-node Patroni cluster กระจายคนละ Availability Zone (AZ)**
เพื่อทนต่อการที่ AZ ทั้งหมด (ไม่ใช่แค่เครื่องเดียว) ล่มพร้อมกัน (เช่น ไฟดับทั้ง data center) — ถ้าใช้แค่ 2 node ความเสี่ยง split-brain จะสูงขึ้นมาก และไม่มี quorum ที่ชัดเจน

**2. etcd 3-node แยก AZ เดียวกับ PostgreSQL node**
วาง etcd node คู่กับ PostgreSQL node ในแต่ละ AZ (ไม่จำเป็นต้องเป็นเครื่องเดียวกัน แต่อยู่ AZ เดียวกัน) เพื่อให้เมื่อ AZ หนึ่งหายไป ทั้ง Patroni node และ etcd node ของ AZ นั้นหายไปพร้อมกัน — ยังเหลือ etcd 2 node (quorum) และ Patroni 2 node ให้ทำงานต่อได้

**3. HAProxy คู่ + Keepalived VIP**
HAProxy เองก็เป็น single point of failure ได้ถ้ามีแค่เครื่องเดียว จึงต้องมี HAProxy 2 เครื่อง (active-passive) ผูกกับ Virtual IP ผ่าน `keepalived` (ใช้ VRRP protocol) — แอปพลิเคชันต่อไปที่ VIP เดียวเสมอ ไม่สนว่า HAProxy เครื่องไหน active อยู่ (รายละเอียดเต็มใน Part 067)

**4. synchronous_mode: true + watchdog required**
เพื่อให้ได้ RPO ≈ 0 ตาม requirement ต้องเปิด synchronous replication (อย่างน้อย 1 synchronous standby) ควบคู่กับ watchdog `mode: required` เพื่อป้องกัน split-brain อย่างเข้มงวด

**5. WAL archiving ไปยัง Object Storage**
แม้จะมี 3-node HA แล้ว ยังคงต้องมี physical backup + WAL archiving (เรียนไปแล้วใน Part 061-062) แยกต่างหาก เพื่อป้องกันกรณี catastrophic failure ที่กระทบทั้ง 3 node พร้อมกัน (เช่น ระดับภัยพิบัติจริง หรือ human error ลบข้อมูลผิด) และใช้เป็นแหล่ง bootstrap standby ใหม่หรือทำ Disaster Recovery site ในภูมิภาคอื่น

### วิเคราะห์ Failure Scenarios

| สถานการณ์ | ระบบตอบสนองอย่างไร |
|---|---|
| pg-node1 (primary) process crash | etcd lease หมดอายุใน ≤30s → node2/node3 แข่ง election → node ที่ lag น้อยกว่าและผ่าน `maximum_lag_on_failover` ถูก promote → HAProxy เห็น `/master` เปลี่ยนภายใน ≤3s → แอป reconnect อัตโนมัติ (RTO รวม ~15-35s) |
| AZ-1 ทั้งโซนไฟดับ (pg-node1 + etcd-1 หายพร้อมกัน) | etcd เหลือ 2 node ยังมี quorum (2/3) → Patroni node2/node3 ยัง election ได้ปกติ → failover สำเร็จเหมือนกรณีข้างต้น |
| Network partition แยก pg-node1 ออกจาก etcd-2, etcd-3 | pg-node1 ติดต่อ etcd ได้แค่ etcd-1 (minority, ไม่มี quorum) → Patroni บน node1 ตรวจพบว่าไม่มี quorum → self-demote (ปฏิเสธรับ write) + watchdog เตรียม reboot ถ้ายังไม่ demote ทัน → ไม่มี split-brain |
| HAProxy #1 (active) เครื่องเสีย | keepalived ตรวจพบ VRRP heartbeat ขาด → VIP ย้ายไป HAProxy #2 ภายในไม่กี่วินาที → แอปยังต่อ VIP เดิมได้ต่อเนื่อง |
| DBA ต้อง upgrade OS ของ pg-node2 (standby) | รัน `patronictl pause` ก่อน (กัน false failover) → maintenance node2 → `patronictl resume` |

---

## สรุปท้ายบท

ในบทนี้เราได้เรียนรู้แนวคิดและสถาปัตยกรรมของ High Availability สำหรับ PostgreSQL ระดับ Production:

1. **HA และ SLA** — เข้าใจว่า "nines" แต่ละตัวหมายถึง downtime ที่ยอมรับได้ต่อปีต่างกันแบบ 10 เท่า และการเพิ่ม nine แต่ละตัวมาพร้อมต้นทุนที่เพิ่มขึ้นไม่เป็นเส้นตรง
2. **Manual failover ไม่พอสำหรับ Production** เพราะ detection delay, human error, และเวลาที่ใช้นานเกินกว่า SLA สูง ๆ จะยอมรับได้
3. **องค์ประกอบ HA อัตโนมัติ 4 อย่าง**: health check, consensus/leader election, automatic failover, client redirection — ต้องทำงานครบทุกส่วนถึงจะเป็นระบบ HA ที่สมบูรณ์
4. **repmgr** — เครื่องมือ HA แบบดั้งเดิม ใช้ `repmgrd` daemon ตรวจจับและ failover เอง ไม่พึ่งพา external consensus store แต่ consensus mechanism ไม่แข็งแกร่งเท่า Raft/Paxos
5. **Patroni** — เครื่องมือ HA สมัยใหม่ มอบหน้าที่ consensus ให้ DCS (etcd/Consul/ZooKeeper) จัดการผ่าน Raft/Paxos ซึ่งเป็นมาตรฐานอุตสาหกรรมปัจจุบัน
6. **สถาปัตยกรรม Patroni** — REST API (port 8008), DCS สำหรับเก็บ leader lock/config/history, watch mechanism ที่ทำให้ node รับรู้การเปลี่ยนแปลงเร็ว, main loop ที่ควบคุมด้วย `ttl`/`loop_wait`/`retry_timeout`
7. **patroni.yml** ครบทุก section: scope/name, restapi, DCS connection, bootstrap.dcs (dynamic config), postgresql (instance config), tags (nofailover/noloadbalance/clonefrom/nosync)
8. **HAProxy ร่วมกับ Patroni** ใช้ `httpchk` เช็ค `/master`/`/replica` endpoint เพื่อ route traffic แยก write/read โดยอัตโนมัติ (รายละเอียดเต็มใน Part 066-067)
9. **Split-brain** คือความเสี่ยงร้ายแรงที่สุดของระบบ HA — ป้องกันด้วยหลาย layer: odd-numbered quorum, watchdog self-fencing, external fencing/STONITH, synchronous replication, maximum_lag_on_failover
10. **การออกแบบสถาปัตยกรรมเต็มรูปแบบ** สำหรับ e-commerce ต้องพิจารณาทั้ง PostgreSQL layer, DCS layer, Load Balancer layer และการกระจายข้าม Availability Zone

### ตารางเปรียบเทียบ repmgr vs Patroni

| คุณสมบัติ | repmgr | Patroni |
|---|---|---|
| Consensus mechanism | Voting โดยตรงระหว่าง node (custom) | มอบให้ DCS (etcd/Consul/ZooKeeper) ที่ใช้ Raft/Paxos |
| External dependency | ไม่ต้องมี (PostgreSQL-native) | ต้องมี DCS cluster แยกต่างหาก |
| REST API ในตัว | ไม่มี (ต้องพึ่ง script เสริม) | มี ครบถ้วน (`/master`, `/replica`, `/config` ฯลฯ) |
| Dynamic config management | จำกัด | รองรับดี ผ่าน `patronictl edit-config` |
| ความนิยมใน Kubernetes/Cloud-native | น้อยกว่า | สูงมาก (มาตรฐาน de facto) |
| ความซับซ้อนในการติดตั้ง | ต่ำกว่า | สูงกว่า (ต้องดูแล DCS เพิ่ม) |
| ความเป็นผู้ใหญ่/ประวัติการใช้งาน | เก่าแก่ ใช้มานาน | ใหม่กว่าแต่เติบโตเร็วมาก |

ในบทถัดไป (**Part 066: Connection Pooling**) เราจะเจาะลึกเรื่อง PgBouncer อย่างละเอียด — transaction/session/statement pooling mode, การตั้งค่า connection limits, และการผสานเข้ากับสถาปัตยกรรม Patroni + HAProxy ที่เราออกแบบไว้ในบทนี้ ตามด้วย **Part 067** ที่จะเจาะลึก HAProxy configuration แบบเต็มรูปแบบพร้อม keepalived สำหรับทำ Virtual IP

---

## แบบฝึกหัด

<details>
<summary><b>แบบฝึกหัดที่ 1:</b> ระบบมี SLA เป้าหมาย 99.95% จงคำนวณว่า downtime ที่ยอมรับได้ต่อปีคือกี่ชั่วโมง/นาที</summary>

**เฉลย:**

```
Downtime = (1 - 0.9995) × 365 × 24 ชั่วโมง
         = 0.0005 × 8760 ชั่วโมง
         = 4.38 ชั่วโมง/ปี
         ≈ 4 ชั่วโมง 22.8 นาที/ปี
```

99.95% อยู่ระหว่าง 3 nines (99.9% = 8.76 ชม./ปี) และ 4 nines (99.99% = 52.56 นาที/ปี) — downtime ที่ยอมรับได้คือประมาณ **4 ชั่วโมง 23 นาทีต่อปี**
</details>

<details>
<summary><b>แบบฝึกหัดที่ 2:</b> อธิบายว่าทำไม manual failover ที่ใช้เวลาเฉลี่ย 20 นาทีต่อครั้ง ไม่สามารถรองรับ SLA 99.99% ได้ แม้จะเกิดปัญหาเพียงปีละ 3 ครั้ง</summary>

**เฉลย:**

SLA 99.99% อนุญาตให้มี downtime ได้เพียง 52.56 นาทีต่อปีทั้งปี ถ้าเกิดปัญหา 3 ครั้ง/ปี และแต่ละครั้งใช้เวลา manual failover 20 นาที รวม downtime = 3 × 20 = 60 นาที ซึ่ง**เกิน budget ที่อนุญาต (52.56 นาที) ไปแล้ว** แม้จะยังไม่นับ downtime จากสาเหตุอื่น เช่น planned maintenance หรือ network issue เล็กน้อย — ดังนั้นต้องใช้ automated failover ที่ใช้เวลาระดับวินาทีแทน เพื่อให้เหลือ budget สำหรับเหตุการณ์อื่นด้วย
</details>

<details>
<summary><b>แบบฝึกหัดที่ 3:</b> องค์ประกอบ HA อัตโนมัติ 4 อย่างคืออะไรบ้าง จงอธิบายสั้น ๆ ทีละอย่างพร้อมยกตัวอย่างเครื่องมือที่ทำหน้าที่นั้นใน Patroni</summary>

**เฉลย:**

1. **Health check** — ตรวจสอบสถานะ node สม่ำเสมอ ใน Patroni คือ REST API `/health` และการเช็ค local PostgreSQL ในทุกรอบ `loop_wait`
2. **Consensus/Leader election** — ตกลงกันว่าใครเป็น leader โดยไม่มี node สองตัวคิดว่าตัวเองเป็น leader พร้อมกัน ใน Patroni คือ DCS (etcd/Consul) ที่ใช้ Raft consensus ผ่านกลไก leader lock/TTL
3. **Automatic failover** — promote standby ใหม่, reconfigure node ที่เหลือ, fence primary เก่า ใน Patroni คือ state machine ภายใน main loop ที่เรียก PostgreSQL controller
4. **Client redirection** — ทำให้แอปหา primary ใหม่เจอโดยอัตโนมัติ ใน Patroni มักใช้ HAProxy ที่เช็ค endpoint `/master` ผ่าน `httpchk`
</details>

<details>
<summary><b>แบบฝึกหัดที่ 4:</b> ในไฟล์ repmgr.conf พารามิเตอร์ `priority` ทำหน้าที่อะไร และถ้าต้องการห้ามไม่ให้ node หนึ่งถูกเลือกเป็น primary เด็ดขาด (เช่น node สำหรับ reporting เท่านั้น) ควรตั้งค่าอย่างไร</summary>

**เฉลย:**

`priority` กำหนดลำดับความสำคัญของ standby ในการถูกเลือกเป็น primary ใหม่ตอน failover — ค่ามากกว่าหมายถึงมีโอกาสถูกเลือกก่อน ถ้าต้องการห้ามไม่ให้ node ใดถูกเลือกเป็น primary เด็ดขาด ให้ตั้งค่า `priority=0` สำหรับ node นั้นใน `repmgr.conf` (เทียบเท่ากับการตั้ง `tags.nofailover: true` ใน Patroni)
</details>

<details>
<summary><b>แบบฝึกหัดที่ 5:</b> จงอธิบายความแตกต่างระหว่างแนวทาง consensus ของ repmgr กับ Patroni ว่าทำไม Patroni ถึงถือว่าแข็งแกร่งกว่าในแง่ทฤษฎี distributed systems</summary>

**เฉลย:**

repmgr ใช้กลไก voting ที่เขียนขึ้นเองโดยให้ node ต่าง ๆ ติดต่อกันโดยตรงเพื่อตัดสินว่า primary ตายหรือไม่ ซึ่งเป็น custom implementation ที่อาจมี edge case ที่ไม่ถูกคิดถึงในสถานการณ์ network partition ที่ซับซ้อน

Patroni มอบหน้าที่ consensus ทั้งหมดให้ DCS ภายนอก (etcd/Consul/ZooKeeper) ที่ใช้อัลกอริทึม Raft หรือ ZAB ซึ่งเป็นอัลกอริทึมที่ถูกพิสูจน์ทางคณิตศาสตร์ (formally proven) ว่าถูกต้องภายใต้เงื่อนไข quorum ที่กำหนด และผ่านการทดสอบอย่างเข้มข้นในระบบ production จำนวนมาก (รวมถึงเป็นแกนหลักของ Kubernetes เอง) จึงมีความน่าเชื่อถือทางทฤษฎีสูงกว่า custom voting mechanism
</details>

<details>
<summary><b>แบบฝึกหัดที่ 6:</b> ในไฟล์ patroni.yml ค่า `ttl`, `loop_wait`, และ `retry_timeout` ควรมีความสัมพันธ์กันอย่างไร และถ้าตั้งค่าใกล้กันเกินไปจะเกิดปัญหาอะไร</summary>

**เฉลย:**

`ttl` (อายุของ leader lock) ควรมากกว่า `loop_wait` (ความถี่ของ main loop) อย่างมีนัยสำคัญ โดยทั่วไปแนะนำ `ttl` ≥ 3×`loop_wait` เพื่อให้มีโอกาส renew lease ได้หลายครั้งก่อนหมดอายุจริง

ถ้าตั้งค่าใกล้กันเกินไป (เช่น `ttl=10`, `loop_wait=10`) แม้แค่ network delay เล็กน้อยหรือ GC pause สั้น ๆ ก็อาจทำให้ renew ไม่ทันเวลา ทำให้ leader lock หมดอายุทั้งที่ primary จริงยัง healthy อยู่ — เกิด "false positive failover" ซึ่งเป็นสาเหตุหนึ่งของความเสี่ยง split-brain และทำให้ระบบ failover บ่อยเกินความจำเป็น (flapping)
</details>

<details>
<summary><b>แบบฝึกหัดที่ 7:</b> HAProxy ใช้กลไกอะไรในการแยก route write traffic กับ read traffic เมื่อทำงานร่วมกับ Patroni และมันตรวจสอบสถานะ node ผ่านช่องทางใด</summary>

**เฉลย:**

HAProxy เปิด listener แยกกันสองพอร์ต โดยพอร์ตสำหรับ write ใช้ `option httpchk GET /master` เพื่อ route ไปเฉพาะ node ที่ Patroni REST API ตอบ HTTP 200 บน endpoint `/master` (คือ primary ปัจจุบัน) ส่วนพอร์ตสำหรับ read ใช้ `option httpchk GET /replica` พร้อม `balance roundrobin` เพื่อกระจาย read query ไปยังทุก node ที่เป็น replica ที่ตอบ 200 — การตรวจสอบทำผ่าน HTTP health check ไปยัง Patroni REST API (port 8008) ของแต่ละ node ไม่ใช่การ query SQL โดยตรง
</details>

<details>
<summary><b>แบบฝึกหัดที่ 8:</b> จงอธิบาย Split-brain ด้วยคำพูดของตัวเอง และยกตัวอย่างสาเหตุที่ทำให้เกิดขึ้นได้อย่างน้อย 2 สาเหตุ</summary>

**เฉลย:**

Split-brain คือสถานการณ์ที่มี node มากกว่าหนึ่งตัวในคลัสเตอร์เชื่อว่าตัวเองเป็น primary พร้อมกัน และรับ write จาก client โดยไม่รู้ตัวว่ามีอีก node ทำงานคู่ขนานเป็น primary เช่นกัน ทำให้ข้อมูลของทั้งสองฝั่ง diverge ออกจากกันจน merge กลับไม่ได้โดยอัตโนมัติ

สาเหตุที่ทำให้เกิดขึ้นได้ เช่น:
1. Network partition ที่แยก primary เดิมออกจากส่วนที่เหลือของคลัสเตอร์ แต่ primary เดิมยังทำงานได้ปกติภายในตัวเองและไม่รู้ว่าถูกตัดขาด
2. False positive health check — timeout สั้นเกินไปจนตีความว่า primary ตายทั้งที่ยัง healthy จริง ทำให้เริ่ม failover สร้าง primary ใหม่ทั้งที่ primary เดิมยังรับ write อยู่
</details>

<details>
<summary><b>แบบฝึกหัดที่ 9:</b> STONITH คืออะไร ต่างจากการสั่งปิดเครื่องผ่าน SSH ธรรมดาอย่างไร และทำไมต้องใช้ out-of-band channel</summary>

**เฉลย:**

STONITH ("Shoot The Other Node In The Head") คือแนวทาง fencing ที่สั่งปิดเครื่อง (power off) node ที่มีปัญหาทันทีแบบ hard ผ่านกลไกภายนอก เช่น IPMI, cloud provider API, หรือ PDU ตัดไฟ

ต่างจากการสั่งปิดผ่าน SSH ตรงที่ SSH ต้องอาศัย network เส้นเดียวกับที่แอปพลิเคชันใช้ ถ้า network เส้นนั้นขาดอยู่พอดี (ซึ่งมักเป็นสาเหตุของปัญหาตั้งแต่แรก) คำสั่ง SSH ก็จะส่งไม่ถึงเช่นกัน ทำให้ fencing ล้มเหลว

จึงต้องใช้ **out-of-band channel** ที่เป็นคนละ network path จากที่ขาด เช่น IPMI ที่มี network card แยกต่างหากบนเมนบอร์ด หรือ cloud provider API ที่ทำงานผ่าน control plane คนละเส้นทางจาก data plane เพื่อรับประกันว่าคำสั่ง fence จะส่งถึงและมีผลจริง แม้ network หลักของ node นั้นจะขาดอยู่ก็ตาม
</details>

<details>
<summary><b>แบบฝึกหัดที่ 10:</b> ในการออกแบบ e-commerce HA architecture ตาม Step 650 จงอธิบายว่าทำไมต้องวาง etcd node คนละ Availability Zone เดียวกับ PostgreSQL/Patroni node ที่สอดคล้องกัน แทนที่จะรวม etcd ทั้ง 3 node ไว้ที่ AZ เดียว</summary>

**เฉลย:**

ถ้ารวม etcd ทั้ง 3 node ไว้ที่ AZ เดียว เมื่อ AZ นั้นล่มทั้งหมด (เช่น ไฟดับทั้งดาต้าเซ็นเตอร์) etcd cluster ทั้งหมดจะหายไปพร้อมกัน ทำให้ไม่มี DCS เหลือให้ Patroni node ใน AZ อื่นใช้ตัดสิน leader election ได้เลย แม้ PostgreSQL node ใน AZ อื่นจะยัง healthy อยู่ก็ตาม — ระบบทั้งคลัสเตอร์จะหยุดทำงาน (ไม่มีใครเป็น primary ได้เพราะติดต่อ DCS ไม่ได้)

การกระจาย etcd node คนละ AZ ที่สอดคล้องกับ Patroni node (etcd-1 คู่กับ pg-node1 ใน AZ-1 เป็นต้น) ทำให้เมื่อ AZ ใดหายไปหนึ่งโซน ทั้ง PostgreSQL node และ etcd node ของโซนนั้นหายไปพร้อมกัน แต่ etcd ที่เหลืออีก 2 node (จาก AZ อื่น) ยังคงมี quorum (2 จาก 3) เพียงพอให้ Patroni node ที่เหลือดำเนินกระบวนการ leader election และ failover ต่อไปได้ตามปกติ — นี่คือหลักการ "align failure domains" ระหว่าง DCS layer และ database layer ในการออกแบบ HA ข้าม Availability Zone
</details>

---

**บทถัดไป:** [Part 066: Connection Pooling](./part-066-connection-pooling.md)
