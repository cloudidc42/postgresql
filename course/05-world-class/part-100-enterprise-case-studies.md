# Case Studies: สถาปัตยกรรมระบบ PostgreSQL ระดับ Enterprise จริง

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับโลก/ผู้เชี่ยวชาญ | Part 100

---

## คำชี้แจงสำคัญก่อนเริ่มบทเรียน (Disclaimer)

บทนี้แตกต่างจากบทอื่น ๆ ในหลักสูตร ตรงที่เนื้อหาไม่ได้เป็น "สูตรสำเร็จ" หรือ "ฟีเจอร์ของ PostgreSQL" แต่เป็น **case study เชิงสถาปัตยกรรม (architecture case study)** ที่สังเคราะห์ขึ้นจากเทคนิคทั้งหมดที่เราเรียนมาตลอดหลักสูตร เพื่อแสดงให้เห็นว่าในโลกจริง วิศวกรระดับ enterprise นำเทคนิคเหล่านั้นมา "ประกอบร่าง" เป็นระบบที่ใช้งานจริงได้อย่างไร

**สิ่งที่ต้องเข้าใจให้ชัดเจนก่อนอ่านต่อ:**

1. **นี่ไม่ใช่ข้อมูลภายในของบริษัทใดบริษัทหนึ่ง** — Case study ทั้ง 4 เรื่องในบทนี้เป็น **สถาปัตยกรรมต้นแบบ (archetype)** ที่ผมสร้างขึ้นเพื่อการเรียนรู้ โดยอ้างอิงจาก **pattern สาธารณะที่เป็นที่รู้จักกันดี** ในวงการวิศวกรรมระบบขนาดใหญ่ (public engineering blogs, conference talks, และหลักการทางวิศวกรรมฐานข้อมูลที่เป็นมาตรฐานอุตสาหกรรม) ไม่ใช่ข้อมูลลับ ไม่ใช่ตัวเลขจริงของบริษัทใด และไม่ควรนำตัวเลขในบทนี้ไปอ้างอิงว่าเป็น "ความจริง" เกี่ยวกับระบบของบริษัทที่มีชื่อคล้ายกัน
2. **ชื่อระบบเป็นชื่อสมมติ** ("FeedFlow", "PayCore", "ShopScale", "TenantHub") เพื่อให้เล่าเรื่องเป็นธรรมชาติ ไม่ได้อ้างอิงถึงบริษัทจริงใด ๆ
3. **ตัวเลขที่ปรากฏ** (เช่น "10 ล้าน QPS", "5,000 tenant") เป็นตัวเลขสมมติที่ **สมเหตุสมผลในเชิงวิศวกรรม (plausible order of magnitude)** เพื่อให้เห็นภาพขนาดของปัญหา ไม่ใช่ตัวเลขที่วัดได้จริงจากระบบใดระบบหนึ่ง
4. **เป้าหมายของบทนี้** คือฝึกให้ผู้เรียน "คิดแบบสถาปนิกระบบ" (systems architect) — เห็นภาพรวมว่าฟีเจอร์ที่เรียนแยกกันมา (indexing, replication, partitioning, locking, RLS, connection pooling, observability ฯลฯ) มาบรรจบกันเป็นการตัดสินใจทางสถาปัตยกรรมได้อย่างไร ภายใต้ constraint ทางธุรกิจจริง เช่น เงิน เวลา ความเสี่ยง และกฎหมาย

หากคุณกำลังสัมภาษณ์งานระดับ Staff/Principal Engineer หรือกำลังออกแบบระบบขนาดใหญ่จริง ให้ใช้บทนี้เป็น "กรอบความคิด (mental framework)" ไม่ใช่ "คำตอบสำเร็จรูป" เพราะระบบจริงทุกระบบมี constraint เฉพาะตัวที่ต้องวิเคราะห์ใหม่เสมอ

---

## เป้าหมายการเรียนรู้

เมื่อจบบทนี้ ผู้เรียนจะสามารถ:

1. วิเคราะห์ระบบ PostgreSQL ขนาดใหญ่แบบ end-to-end โดยเชื่อมโยงทุกเทคนิคที่เรียนมา (indexing, replication, partitioning, sharding, locking, security, observability) เข้าด้วยกันเป็นภาพเดียว
2. อธิบาย trade-off ของสถาปัตยกรรมสำคัญ 4 แบบ ได้แก่ read-heavy fan-out system, strict-ACID financial system, high-concurrency inventory system, และ multi-tenant SaaS system
3. เลือกกลยุทธ์ที่เหมาะสมระหว่าง caching layer, read replica, sharding, RLS, schema-per-tenant, และ database-per-tenant โดยพิจารณาจากขนาดและลักษณะของ workload
4. ระบุ "บทเรียนร่วม" (common patterns) ที่ปรากฏซ้ำในระบบ enterprise ทุกประเภท เช่น defense in depth, graceful degradation, observability-first design
5. วิเคราะห์และนำเสนอ trade-off ของสถาปัตยกรรมได้อย่างมีเหตุผล ในลักษณะที่ใช้ตอบคำถามสัมภาษณ์งานระดับ Senior/Staff/Principal Engineer ได้จริง

**ความรู้พื้นฐานที่ควรมีก่อนอ่านบทนี้:** บทนี้อ้างอิงเนื้อหาจากเกือบทุกส่วนของหลักสูตร โดยเฉพาะ indexing (ระดับ Intermediate), replication & high availability, partitioning, MVCC และ locking (Part 058), connection pooling, row-level security (RLS), และ observability/monitoring หากยังไม่คุ้นเคยกับหัวข้อเหล่านี้ แนะนำให้ย้อนกลับไปทบทวนก่อน

---

## Step 986: Case Study 1 — ระบบ Social Media / Feed ขนาดใหญ่: การออกแบบรับมือ Read-Heavy Workload มหาศาล

### 986.1 โจทย์ทางธุรกิจ

**"FeedFlow"** เป็นแพลตฟอร์ม social media สมมติที่มีผู้ใช้งาน 200 ล้านบัญชี โดยมี Daily Active Users (DAU) ราว 60 ล้านคน แต่ละคนเปิดแอปเฉลี่ย 15 ครั้งต่อวัน และแต่ละครั้งจะโหลด feed ประมาณ 20-40 โพสต์ ลักษณะ workload มีอัตราส่วนที่ชัดเจนมาก:

- **Read : Write ratio ≈ 1000 : 1** — ทุกครั้งที่มีคนโพสต์ 1 ครั้ง จะมีการอ่าน (แสดงในหน้า feed ของคนอื่น) นับพันครั้ง
- Peak read traffic: ~8-10 ล้าน query ต่อวินาที (รวมทุก service ที่คุยกับฐานข้อมูล ไม่ใช่ query ตรงไปที่ PostgreSQL ทั้งหมด — ส่วนใหญ่ถูกดูดซับโดย cache layer)
- Write traffic (โพสต์ใหม่ + like + comment): ~50,000-80,000 เขียนต่อวินาทีในช่วง peak
- ข้อมูลรวมเติบโตเร็วมาก: โพสต์ใหม่ ~2 พันล้านรายการต่อปี, ความสัมพันธ์ social graph (follow/follower) หลายแสนล้าน edge

**คำถามสถาปัตยกรรมหลัก:** จะออกแบบระบบอย่างไรให้ผู้ใช้ 60 ล้านคนเห็น feed ที่ "สด" (fresh) ภายในเวลาต่ำกว่า 200ms โดยที่ PostgreSQL เพียงเครื่องเดียวไม่มีทางรับ throughput ระดับนี้ได้โดยตรง?

### 986.2 สถาปัตยกรรมภาพรวม

```
                              ┌─────────────────────────┐
                              │      Client / Mobile     │
                              └────────────┬─────────────┘
                                           │
                              ┌────────────▼─────────────┐
                              │     Edge / CDN Layer      │  (static assets, images)
                              └────────────┬─────────────┘
                                           │
                              ┌────────────▼─────────────┐
                              │   API Gateway / LB         │
                              └────────────┬─────────────┘
                                           │
                     ┌─────────────────────┼─────────────────────┐
                     │                     │                     │
            ┌────────▼────────┐  ┌─────────▼────────┐  ┌─────────▼────────┐
            │  Feed Service     │  │  Post Service      │  │  Social Graph      │
            │  (read-heavy)     │  │  (write path)       │  │  Service           │
            └────────┬────────┘  └─────────┬────────┘  └─────────┬────────┘
                     │                     │                     │
        ┌────────────▼───────────┐        │          ┌───────────▼────────────┐
        │  Redis Cache Cluster     │        │          │  Redis / In-memory       │
        │  (feed timeline, hot     │        │          │  graph cache              │
        │   post metadata)         │        │          │  (follow/follower list)   │
        │  TTL + write-through     │        │          └───────────┬────────────┘
        └────────────┬───────────┘        │                      │
                     │ cache miss           │                      │
        ┌────────────▼───────────────────────▼──────────────────────▼────────────┐
        │                         Connection Pooler (PgBouncer)                    │
        │              transaction pooling, ~2,000-5,000 connections               │
        └────────────┬───────────────────────────────────────────────┬───────────┘
                     │                                              │
        ┌────────────▼────────────┐                    ┌────────────▼────────────┐
        │   Primary (Write) Node    │──── streaming ────▶│  Read Replica Pool        │
        │   - posts table            │    replication      │  (N=10-30 replicas,       │
        │   - social_graph table     │                      │   geo-distributed)        │
        │   - sharded by user_id     │                      │  - feed reads              │
        │     (Part: Sharding /      │                      │  - analytics reads         │
        │     Citus-style)           │                      └────────────────────────────┘
        └─────────────────────────┘
                     │
        ┌────────────▼────────────┐
        │  Fan-out Worker Queue     │  (Kafka / SQS)
        │  - push model for        │
        │    "celebrity" accounts   │
        │  - pull model for        │
        │    normal accounts        │
        └────────────────────────┘
```

### 986.3 การตัดสินใจสำคัญ (Key Design Decisions)

**(1) Caching Layer เป็นเกราะป้องกันด่านแรก ไม่ใช่ทางเลือก**

จุดที่สำคัญที่สุดของสถาปัตยกรรมนี้คือ PostgreSQL **ไม่เคยเห็น** query ส่วนใหญ่เลย เพราะ Redis cache layer ดูดซับ read traffic ไปแล้วกว่า 95-99% สำหรับ hot data (เช่น timeline ของผู้ใช้ที่ active, metadata ของโพสต์ยอดนิยม) หลักการคือ:

- **Cache-aside pattern** สำหรับข้อมูลที่ query ซับซ้อน (เช่น aggregated feed)
- **Write-through** สำหรับข้อมูล metadata ที่ต้อง consistent สูง เช่น like count โดยประมาณ (approximate counter — ยอมรับความคลาดเคลื่อนเล็กน้อยเพื่อแลกกับ throughput)
- TTL สั้น (วินาทีถึงนาที) สำหรับ feed timeline เพื่อสมดุลระหว่างความสดกับ cache hit rate

ผลคือ PostgreSQL รับ query จริงเพียงเศษเสี้ยวของ 8-10 ล้าน query/วินาที (อาจเหลือหลักหมื่นถึงแสน query/วินาทีที่กระทบฐานข้อมูลจริง) ซึ่งยังคงเป็นตัวเลขสูงมาก แต่จัดการได้ด้วยการกระจายโหลด

**(2) Read Replica Pool + Geo-distribution**

เนื่องจาก read:write ratio สูงมาก การเพิ่ม read replica คือการลงทุนที่คุ้มค่าที่สุด ระบบใช้ **streaming replication** (ดูเนื้อหาบท Replication & High Availability) พร้อม replica จำนวนมาก กระจายตามภูมิภาคเพื่อลด latency สำหรับผู้ใช้ทั่วโลก จุดที่ต้องระวังคือ **replication lag** — feed service ต้องออกแบบให้ "ทนต่อความล่าช้า" (eventual consistency) ได้ เช่น:

- โพสต์ใหม่ของตัวเองต้องเห็นทันที → อ่านจาก primary หรือ cache แบบ read-your-writes
- feed ของคนอื่นยอมรับความล่าช้าได้เป็นวินาที → อ่านจาก replica ได้สบาย

**(3) Sharding ตาม user_id**

เมื่อข้อมูล social graph และโพสต์เติบโตถึงระดับที่ replica เดียวไม่พอ (ทั้ง storage และ write throughput ที่ primary ต้องรับ) ระบบต้อง **shard ตาม user_id** (horizontal partitioning ข้าม node) เพื่อให้:

- Write ของแต่ละ shard กระจายกันไป ไม่ผูกกับ primary node เดียว
- Query ที่ query "feed ของ user X" หรือ "โพสต์ของ user X" เป็น query ที่ shard-local (ไม่ต้อง fan-out ข้าม shard)

จุดยากคือ query ที่ "ข้าม user" เช่น hashtag search หรือ trending topics ต้องพึ่งระบบแยกต่างหาก (search engine เช่น Elasticsearch/OpenSearch) แทนที่จะพยายามทำ cross-shard query บน PostgreSQL โดยตรง — นี่คือหลักการ **"เลือกเครื่องมือให้ตรงกับรูปแบบ query"** ที่เรียนในบท Partitioning/Sharding

**(4) Fan-out Pattern: Push vs Pull**

นี่คือหัวใจของระบบ feed ทุกระบบ มี 2 กลยุทธ์หลัก:

- **Push model (fan-out-on-write):** เมื่อมีคนโพสต์ ระบบจะเขียนโพสต์นั้นเข้า timeline cache ของ **ทุก follower ทันที** ข้อดีคืออ่านเร็วมาก (แค่ดึง timeline ที่เตรียมไว้แล้ว) แต่ข้อเสียคือถ้าบัญชีมี follower หลักล้าน (celebrity account) การเขียนครั้งเดียวจะกลายเป็นการเขียนหลักล้านครั้ง — เรียกว่า **"celebrity problem"**
- **Pull model (fan-out-on-read):** เมื่อผู้ใช้เปิด feed ระบบจะไป query โพสต์ล่าสุดจากทุกคนที่ follow แบบ real-time แล้วรวมกัน (merge) ข้อดีคือเขียนเบา แต่ข้อเสียคืออ่านหนักและ latency สูงถ้า follow เยอะ

**FeedFlow (สถาปัตยกรรมต้นแบบ) ใช้แบบ Hybrid:**
- บัญชีทั่วไป (follower < ~10,000) → push model, fan-out เขียนเข้า cache ทันทีผ่าน queue worker
- บัญชี celebrity (follower สูงมาก) → pull model, feed service จะ merge โพสต์จาก celebrity เข้ากับ timeline cache ของผู้ใช้ตอน "อ่าน" แทน

นี่คือตัวอย่างคลาสสิกของ **การประนีประนอมทางสถาปัตยกรรมตาม data distribution** ซึ่งเป็นแนวคิดเดียวกับที่เราเจอตอนเลือก partitioning key — ต้องวิเคราะห์ **การกระจายตัวของข้อมูล (data skew)** ก่อนเลือกกลยุทธ์

### 986.4 Trade-off ที่ต้องยอมรับ

| การตัดสินใจ | สิ่งที่ได้ | สิ่งที่เสีย |
|---|---|---|
| Cache-first design | latency ต่ำมาก, ลดโหลด DB มหาศาล | ความซับซ้อนเรื่อง cache invalidation, ข้อมูลอาจไม่ real-time 100% |
| Read replica จำนวนมาก | scale การอ่านได้เกือบไม่จำกัด | replication lag, ต้นทุน infra สูง, ต้องจัดการ failover ที่ซับซ้อนขึ้น |
| Sharding by user_id | write scale ได้, query เร็วสำหรับ single-user | cross-shard query (search, analytics) ต้องพึ่งระบบอื่น, operational complexity สูงมาก (schema migration ต้องรันทุก shard) |
| Hybrid fan-out | แก้ celebrity problem ได้ | โค้ด feed service ซับซ้อนขึ้นมาก ต้องมี logic แยก 2 เส้นทาง |

### 986.5 เชื่อมโยงกับบทก่อนหน้า

- **Indexing (ระดับ Intermediate):** composite index บน `(user_id, created_at DESC)` เป็นหัวใจของทุก query timeline
- **Partitioning:** ตาราง `posts` มักถูก partition ตามเวลา (time-based) ซ้อนกับการ shard ตาม user_id เพื่อให้ retention/archival ทำได้ง่าย (DROP PARTITION แทน DELETE)
- **Replication & HA:** streaming replication, replica lag monitoring, failover strategy
- **Connection Pooling:** PgBouncer transaction pooling จำเป็นมากเมื่อมี service instance หลักพันตัวที่ต้อง connect เข้า DB พร้อมกัน
- **Observability:** ต้อง monitor replication lag, cache hit ratio, และ query latency percentile (p99) แบบ real-time เพื่อจับปัญหาก่อนผู้ใช้รู้สึก

---

## Step 987: Case Study 2 — ระบบ Fintech / Payment: ACID เข้มงวด, Audit Trail และ Compliance

### 987.1 โจทย์ทางธุรกิจ

**"PayCore"** เป็นระบบประมวลผลการชำระเงินสมมติ ที่ต้องรองรับธุรกรรมทางการเงิน (transfer, payment capture, refund) มูลค่ารวมหลายพันล้านบาทต่อวัน ลักษณะ workload แตกต่างจาก Case Study 1 โดยสิ้นเชิง:

- **Throughput ต่ำกว่ามาก** (หลักพันถึงหลักหมื่น transaction/วินาทีที่ peak) แต่ **ความถูกต้องต้อง 100%**
- **Zero tolerance สำหรับ data loss** — เงิน 1 บาทหายไปคือปัญหาระดับ incident สูงสุด (Sev-1)
- ต้องมี **audit trail ที่แก้ไขไม่ได้ (immutable)** สำหรับทุกธุรกรรม เพื่อผ่าน compliance เช่น PCI-DSS, และการตรวจสอบบัญชี (audit)
- Consistency สำคัญกว่า latency: ยอมรับ latency สูงขึ้นเล็กน้อยเพื่อแลกกับความถูกต้อง

**คำถามสถาปัตยกรรมหลัก:** จะออกแบบระบบอย่างไรให้ **ไม่มีทางที่เงินจะหายหรือถูกนับซ้ำ (double-spend)** แม้ในสถานการณ์ที่เลวร้ายที่สุด เช่น network partition, process crash กลางธุรกรรม, หรือแม้แต่ operator พิมพ์คำสั่งผิด?

### 987.2 สถาปัตยกรรมภาพรวม

```
              ┌──────────────────────┐
              │   Client / Merchant    │
              │   API (idempotent)     │
              └───────────┬───────────┘
                          │  Idempotency-Key header
              ┌───────────▼───────────┐
              │   Payment Gateway       │
              │   Service (stateless)   │
              └───────────┬───────────┘
                          │
       ┌──────────────────┼──────────────────┐
       │                  │                  │
┌──────▼──────┐  ┌────────▼────────┐  ┌───────▼───────┐
│ Ledger        │  │ Fraud/Risk       │  │ Audit Log       │
│ Service        │  │ Engine (async)   │  │ Service (append │
│ (core, sync)   │  │                  │  │ -only)           │
└──────┬──────┘  └─────────────────┘  └───────┬───────┘
       │                                       │
       │           single writer, no           │
       │           read replica lag risk        │
       │           for financial reads          │
       │                                       │
┌──────▼───────────────────────────────────────▼───────┐
│              PostgreSQL Primary (synchronous)          │
│  ┌────────────────────────────────────────────────┐   │
│  │  accounts, ledger_entries (append-only,          │   │
│  │  double-entry bookkeeping), idempotency_keys      │   │
│  │  ┌──────────────────────────────────────────┐    │   │
│  │  │ SERIALIZABLE isolation for balance checks │    │   │
│  │  │ + row-level locking (SELECT ... FOR UPDATE│    │   │
│  │  │   NOWAIT) เพื่อกัน double-spend            │    │   │
│  │  └──────────────────────────────────────────┘    │   │
│  └────────────────────────────────────────────────┘   │
└───────────────┬──────────────────────────────────────┘
                │  synchronous_commit = on/remote_apply
   ┌────────────▼─────────────┐
   │  Synchronous Standby       │  (quorum commit — ต้อง ack
   │  (at least 1, often 2)     │   ก่อน primary ตอบ COMMIT สำเร็จ)
   └────────────┬─────────────┘
                │  streaming replication (async, extra copies)
   ┌────────────▼─────────────┐
   │  Async Replicas             │  (reporting, analytics, DR site
   │  (cross-region DR)          │   ต่างภูมิภาค)
   └───────────────────────────┘

   ┌───────────────────────────────────────────────────┐
   │  WAL Archiving → Point-in-Time Recovery (PITR)      │
   │  + off-site encrypted backup (retention ตาม          │
   │  ข้อกำหนดกฎหมาย เช่น 7 ปี)                            │
   └───────────────────────────────────────────────────┘
```

### 987.3 การตัดสินใจสำคัญ (Key Design Decisions)

**(1) Double-Entry Bookkeeping แบบ Append-Only**

หัวใจของระบบบัญชีการเงินทุกระบบคือ **ไม่มีการ UPDATE ยอดเงินโดยตรง** แต่ใช้หลัก double-entry ที่ทุกธุรกรรมสร้าง 2 บรรทัดขึ้นไปใน `ledger_entries` (เดบิต/เครดิต) ที่ผลรวมต้องเป็นศูนย์เสมอ ยอดคงเหลือ (balance) คำนวณจากผลรวมของ ledger entries (หรือ cache เป็น materialized balance ที่ reconcile เป็นระยะ) ข้อดีคือ:

- ตาราง ledger เป็น **immutable / append-only** → ไม่มีทางที่ประวัติจะถูกแก้ไขทับ (เทียบเท่า audit trail ในตัว)
- ตรวจสอบความถูกต้องได้ตลอดเวลาด้วยการ SUM และเปรียบเทียบ (reconciliation job รันเป็น cron ทุกคืน)
- รองรับการสืบสวนย้อนหลัง (forensic audit) ได้ 100% เพราะไม่มีข้อมูลถูกเขียนทับ

**(2) Isolation Level และ Locking ที่เข้มงวด**

การโอนเงินระหว่างบัญชีคือ classic race condition — ถ้าสองธุรกรรมพยายามหักบัญชีเดียวกันพร้อมกัน (เช่น double-spend attack) ต้องมีกลไกป้องกันแน่นหนา:

- ใช้ **`SELECT ... FOR UPDATE`** เพื่อ lock แถวบัญชีก่อนตรวจสอบยอดคงเหลือและหักเงิน (pessimistic locking) — ป้องกัน lost update
- สำหรับ operation ที่ซับซ้อนกว่า (เช่น ตรวจสอบ constraint ข้ามหลายบัญชี) อาจใช้ **`SERIALIZABLE` isolation level** เพื่อให้ PostgreSQL ตรวจจับ conflict อัตโนมัติ แล้ว retry transaction ที่ล้มเหลว (แนวคิดจาก Part 058 — Locking & Concurrency Control)
- ใช้ **`NOWAIT` หรือ `SKIP LOCKED`** อย่างระมัดระวังตามบริบท เพื่อป้องกัน deadlock แทนที่จะให้ transaction ค้างรอ

**(3) Idempotency Key — ป้องกัน Double-Processing**

เครือข่ายไม่น่าเชื่อถือ 100% เสมอ (timeout, retry จาก client) ระบบการเงินทุกระบบต้องมีตาราง `idempotency_keys` ที่บันทึกว่า request ID นี้เคยประมวลผลไปแล้วหรือยัง ก่อนจะดำเนินการซ้ำ — นี่คือ pattern ที่ป้องกัน "เรียก API ซ้ำเพราะ timeout แต่จริง ๆ transaction แรกสำเร็จแล้ว" ซึ่งถ้าไม่มีกลไกนี้ เงินอาจถูกโอนซ้ำสองครั้ง

**(4) Synchronous Replication สำหรับ Zero Data Loss**

ต่างจาก Case Study 1 ที่ใช้ async replication เพื่อความเร็ว ระบบการเงินต้องใช้ **synchronous_commit** (อย่างน้อย 1 standby, มักเป็น quorum commit กับ 2 standby ขึ้นไป) เพื่อรับประกันว่าธุรกรรมที่ COMMIT สำเร็จแล้ว **มีสำเนาอยู่อย่างน้อย 2 ที่** ก่อนตอบกลับลูกค้าว่าสำเร็จ — ยอมรับ latency ที่สูงขึ้น (เพิ่มไม่กี่ ms) เพื่อแลกกับการรับประกันว่าจะไม่มี transaction ใดหายไปแม้ primary จะล่มทันทีหลัง COMMIT

**(5) Audit Trail และ Compliance (PCI-DSS)**

- ข้อมูลบัตรเครดิตที่อ่อนไหว (PAN, CVV) **ไม่ถูกเก็บใน PostgreSQL โดยตรง** — ใช้ tokenization ผ่าน payment processor ภายนอกที่ผ่านการรับรอง PCI-DSS Level 1 แทน (halo scope reduction)
- ทุกการเข้าถึงข้อมูลอ่อนไหวถูกบันทึกผ่าน audit log แยกต่างหาก (append-only, ship ไปยัง SIEM แบบ near real-time)
- ใช้ **Row-Level Security (RLS)** ร่วมกับ role-based access control เพื่อจำกัดว่า service account ไหนเห็นข้อมูลลูกค้ารายใดได้บ้าง (defense in depth)
- Encryption at rest (TDE หรือ disk-level encryption) และ in transit (TLS) เป็นมาตรฐานบังคับ
- Backup ต้อง **encrypted** และมี retention ตามกฎหมาย (เช่น 7 ปีสำหรับข้อมูลทางบัญชีในหลายประเทศ) พร้อมทดสอบ restore เป็นระยะ (compliance มักกำหนดให้ต้องพิสูจน์ได้ว่า restore ได้จริง ไม่ใช่แค่มี backup)

### 987.4 Trade-off ที่ต้องยอมรับ

| การตัดสินใจ | สิ่งที่ได้ | สิ่งที่เสีย |
|---|---|---|
| Synchronous replication | รับประกัน zero data loss (RPO=0) | latency สูงขึ้น, availability ผูกกับ standby ที่ต้อง ack |
| Pessimistic locking (`FOR UPDATE`) | ป้องกัน race condition ชัดเจน เข้าใจง่าย | throughput ต่ำกว่า optimistic approach, เสี่ยง lock contention ถ้าออกแบบไม่ดี |
| Append-only ledger | audit trail สมบูรณ์ ตรวจสอบย้อนหลังได้ 100% | อ่าน "ยอดคงเหลือปัจจุบัน" ต้องคำนวณหรือ cache แยก เพิ่มความซับซ้อน |
| Tokenization แยกข้อมูลอ่อนไหวออกจาก DB หลัก | ลด compliance scope ของ PostgreSQL เอง | เพิ่ม dependency กับระบบภายนอก, latency เพิ่มขึ้นเล็กน้อยตอนเรียก token |

### 987.5 เชื่อมโยงกับบทก่อนหน้า

- **Transactions & ACID (ระดับ Foundations):** พื้นฐานของ atomicity/consistency ที่ทั้งระบบยืนอยู่บน
- **Locking & Concurrency Control (Part 058):** `SELECT FOR UPDATE`, `SERIALIZABLE`, deadlock detection คือแกนกลางของ ledger service
- **Replication & High Availability:** synchronous vs asynchronous replication, quorum commit
- **Backup & PITR:** WAL archiving, point-in-time recovery สำหรับ compliance และ disaster recovery
- **Security (RLS, Encryption):** การจำกัดสิทธิ์เข้าถึงข้อมูลตาม role และ tenant

---

## Step 988: Case Study 3 — ระบบ E-commerce ขนาดใหญ่ระดับ Black Friday: รับมือ Traffic Spike และ Inventory Consistency

### 988.1 โจทย์ทางธุรกิจ

**"ShopScale"** เป็นแพลตฟอร์ม e-commerce สมมติที่ปกติมี traffic ระดับปานกลาง แต่ในช่วง **Black Friday / 11.11 / เทศกาลลดราคาใหญ่** traffic พุ่งขึ้น **20-50 เท่า** ภายในเวลาไม่กี่นาที ลักษณะเฉพาะของโจทย์นี้คือ:

- Traffic ไม่ได้เพิ่มแบบค่อยเป็นค่อยไป แต่ **spike ทันที** ณ เวลาเปิดดีล (เช่น เที่ยงคืนตรง) — ระบบ auto-scaling ที่ scale ตาม CPU average อาจตามไม่ทัน
- สินค้า flash sale จำนวนจำกัด (เช่น มี 100 ชิ้น) มีคนแย่งซื้อพร้อมกันหลักหมื่นถึงหลักแสนคนในวินาทีเดียว → **high-concurrency write บนแถวเดียวกัน (hot row)**
- ต้องป้องกัน **overselling** (ขายเกินสต็อกที่มีจริง) อย่างเด็ดขาด แต่ในขณะเดียวกันต้องไม่ทำให้ระบบช้าจนผู้ใช้ทั้งหมด timeout
- ระบบต้อง **graceful degradation** ได้ — ถ้าโหลดเกินขีดจำกัด ควรลดฟีเจอร์บางอย่างลง (เช่น ปิด recommendation engine ชั่วคราว) แทนที่จะล่มทั้งระบบ

**คำถามสถาปัตยกรรมหลัก:** จะออกแบบระบบตรวจสอบและตัดสต็อกสินค้าอย่างไร ให้ **ไม่ oversell แม้แต่ชิ้นเดียว** ในขณะที่มีคนหลายหมื่นคนพยายามซื้อสินค้าชิ้นเดียวกันในวินาทีเดียวกัน โดยไม่ทำให้ database ล่มจาก lock contention?

### 988.2 สถาปัตยกรรมภาพรวม

```
        ┌──────────────────────────────────────────────┐
        │         Traffic Shaping / Queue at Edge          │
        │  (virtual waiting room, rate limiting ที่ CDN/    │
        │   API Gateway ก่อนเข้าระบบจริง — ลด thundering     │
        │   herd effect ตั้งแต่ต้นทาง)                       │
        └──────────────────────┬───────────────────────┘
                               │
        ┌──────────────────────▼───────────────────────┐
        │              Application Layer                  │
        │  - Circuit breaker ต่อ downstream service         │
        │  - Feature flag: ปิด non-critical feature         │
        │    อัตโนมัติเมื่อ load สูง (graceful degradation)    │
        └───────┬───────────────────────────┬───────────┘
               │                           │
   ┌───────────▼───────────┐   ┌───────────▼───────────┐
   │  Product Catalog        │   │  Order / Checkout        │
   │  Service (read-heavy)   │   │  Service (write-critical) │
   │  → read replica + cache │   │  → primary only            │
   └───────────────────────┘   └───────────┬───────────┘
                                           │
                          ┌─────────────────▼─────────────────┐
                          │   Inventory Reservation Layer        │
                          │  ┌─────────────────────────────┐   │
                          │  │ กลยุทธ์ที่เลือก (ดู 988.3):     │   │
                          │  │  - Redis atomic counter        │   │
                          │  │    (pre-check, fast reject)    │   │
                          │  │  - PostgreSQL เป็น source        │   │
                          │  │    of truth ตัวสุดท้าย            │   │
                          │  └─────────────────────────────┘   │
                          └─────────────────┬─────────────────┘
                                           │
                          ┌─────────────────▼─────────────────┐
                          │        PostgreSQL Primary            │
                          │  ┌───────────────────────────────┐ │
                          │  │ inventory table                 │ │
                          │  │ - UPDATE ... WHERE stock > 0     │ │
                          │  │   (atomic conditional update,   │ │
                          │  │    ไม่ใช้ SELECT แล้วค่อย UPDATE) │ │
                          │  │ - Advisory lock ต่อ product_id   │ │
                          │  │   สำหรับ flash-sale item          │ │
                          │  │   (ลด row-lock contention)        │ │
                          │  └───────────────────────────────┘ │
                          └─────────────────────────────────────┘
                                           │
                          ┌─────────────────▼─────────────────┐
                          │   Async Order Processing (Queue)     │
                          │   payment capture, notification,      │
                          │   ไม่บล็อก checkout path                │
                          └─────────────────────────────────────┘
```

### 988.3 การตัดสินใจสำคัญ (Key Design Decisions)

**(1) Atomic Conditional UPDATE แทน "Check-then-Act"**

ข้อผิดพลาดคลาสสิกที่สุดของนักพัฒนาที่ไม่คุ้นเคยกับ concurrency คือการเขียนโค้ดแบบ:

```sql
-- ผิด: race condition ชัดเจน (check-then-act)
SELECT stock FROM inventory WHERE product_id = 123;
-- ... ตรวจสอบใน application ว่า stock > 0 ...
UPDATE inventory SET stock = stock - 1 WHERE product_id = 123;
```

รูปแบบนี้มี **race window** ระหว่าง SELECT กับ UPDATE ที่ทำให้สอง transaction พร้อมกันอ่านเห็น stock เท่ากัน แล้วทั้งคู่หักสำเร็จ → oversell ทันที

รูปแบบที่ถูกต้อง (ตามหลักการที่เรียนใน Part 058 — Locking & Concurrency) คือทำให้การตรวจสอบและการหักเป็น **atomic operation เดียว**:

```sql
UPDATE inventory
   SET stock = stock - 1
 WHERE product_id = 123
   AND stock > 0
RETURNING stock;
-- ถ้า RETURNING ไม่มีแถว แปลว่าสินค้าหมดแล้ว (0 rows affected)
```

คำสั่งเดียวนี้ใช้ row-level lock ของ PostgreSQL โดยอัตโนมัติ (MVCC + row lock ระหว่าง UPDATE) ทำให้ไม่มี transaction ใดสามารถหักสต็อกที่ติดลบได้ ไม่ว่าจะมี concurrent request กี่พันตัวพร้อมกัน

**(2) Advisory Lock สำหรับ Hot Row (Flash Sale Item)**

เมื่อสินค้าชิ้นเดียวถูก UPDATE พร้อมกันหลักหมื่นครั้งในวินาทีเดียว แม้ atomic UPDATE จะถูกต้อง แต่จะเกิด **lock contention รุนแรง** บนแถวเดียว (แถวถูก lock แบบ serialize กันหมด ทำให้ throughput ของแถวนั้นถูกจำกัดที่ความเร็วของ disk fsync ต่อ transaction) กลยุทธ์บรรเทาปัญหานี้ ได้แก่:

- **Application-level rate limiting / queueing ก่อนถึง DB** — ใช้ Redis เป็น fast pre-check (atomic `DECR`) เพื่อ "กรอง" คนที่ไม่มีสิทธิ์ซื้อออกไปก่อนที่จะยิง query เข้า PostgreSQL เลย เหลือเฉพาะ request ที่ "อาจจะ" สำเร็จจริง ๆ เท่านั้นที่ไปแตะ DB — ลดจำนวน transaction ที่ชน hot row ลงมหาศาล
- **แบ่งสต็อกเป็น "shard ย่อย" ชั่วคราว (stock sharding)** — เช่น สินค้า 1,000 ชิ้น แบ่งเป็น 10 แถวย่อย ๆ ละ 100 ชิ้น กระจาย request ไปหักแถวใดแถวหนึ่งแบบสุ่ม ลด contention ต่อแถวลง 10 เท่า แล้วค่อย reconcile ผลรวมภายหลัง (แลกกับความซับซ้อนเพิ่มขึ้นและอาจมีบางแถว "หมด" เร็วกว่าแถวอื่นเล็กน้อย)
- **`FOR UPDATE SKIP LOCKED`** สำหรับกรณีที่ inventory ถูกจัดสรรจากหลาย batch/lot — อนุญาตให้ transaction ที่ชน lock ข้ามไปจับ batch อื่นแทนที่จะรอ

**(3) แยก Read Path ออกจาก Write Path อย่างชัดเจน**

การดู "รายละเอียดสินค้า" (browse) ต่างจากการ "กดสั่งซื้อ" (checkout) โดยสิ้นเชิงในแง่ consistency requirement — การ browse ใช้ read replica + cache ได้สบาย (ยอมรับ stock ที่แสดงคลาดเคลื่อนได้เล็กน้อย เช่น "เหลือน้อย" แทนตัวเลขแม่นยำ) แต่การ checkout ต้อง query ไปที่ **primary เท่านั้น** เพื่อความถูกต้อง 100% ณ เวลาที่ตัดสต็อกจริง นี่คือการประยุกต์หลักการ **CQRS (Command Query Responsibility Segregation)** ที่เรียนในบท Event Sourcing/CQRS

**(4) Traffic Shaping ที่ Edge — Virtual Waiting Room**

เมื่อ traffic คาดว่าจะ spike รุนแรงในเวลาที่รู้ล่วงหน้า (เช่น เที่ยงคืนวัน Black Friday) แนวทางที่ระบบระดับ enterprise ใช้กันคือ **ไม่ปล่อยให้ทุก request ไปถึง backend พร้อมกัน** แต่สร้าง "ห้องรอเสมือน" (virtual waiting room) ที่ edge layer เพื่อ throttle อัตราการปล่อย request เข้าระบบให้อยู่ในระดับที่ backend รับมือได้ นี่คือรูปแบบหนึ่งของ **backpressure** ที่ป้องกันไม่ให้ database ล่มจาก thundering herd ตั้งแต่ต้นทาง แทนที่จะพึ่ง auto-scaling อย่างเดียวซึ่งมักตามไม่ทัน spike ที่เกิดในเวลาไม่กี่วินาที

**(5) Graceful Degradation**

เมื่อโหลดเกินขีดจำกัดจริง ๆ ระบบควรมี feature flag ที่ปิดฟีเจอร์ที่ไม่จำเป็นต่อการทำธุรกรรมหลักโดยอัตโนมัติ เช่น:

- ปิด "สินค้าที่คุณอาจสนใจ" (recommendation, มักเป็น query ที่หนักและซับซ้อน)
- ลดความแม่นยำของตัวเลข "เหลือ X ชิ้น" เป็นแค่ "เหลือน้อย" (ลด query load)
- ปิด real-time analytics dashboard ชั่วคราว เพื่อสงวน replica capacity ไว้ให้ query ที่สำคัญกว่า

หลักการคือ **"รักษาเส้นทางหลัก (critical path) ให้ทำงานได้เสมอ แม้ต้องเสียสละฟีเจอร์รอง"**

### 988.4 Trade-off ที่ต้องยอมรับ

| การตัดสินใจ | สิ่งที่ได้ | สิ่งที่เสีย |
|---|---|---|
| Atomic conditional UPDATE | ป้องกัน oversell ได้แน่นอน 100% | ยังคงมี lock contention บน hot row ถ้าไม่มีมาตรการเสริม |
| Redis pre-check ก่อนแตะ DB | ลดโหลด DB มหาศาล ตอบสนองเร็ว | เพิ่มความซับซ้อน ต้อง sync ระหว่าง Redis กับ DB, เสี่ยง inconsistency ชั่วคราว |
| Stock sharding | ลด contention ต่อแถวลงอย่างมาก | เพิ่ม operational complexity, อาจมี edge case ที่ shard หนึ่งหมดเร็วกว่าที่ควร |
| Virtual waiting room ที่ edge | ป้องกัน thundering herd ตั้งแต่ต้นทาง | เพิ่ม latency ให้ผู้ใช้บางส่วน (ต้องรอคิว), UX ต้องออกแบบให้สื่อสารชัดเจน |

### 988.5 เชื่อมโยงกับบทก่อนหน้า

- **Locking & Concurrency Control (Part 058):** เป็นแกนกลางที่สุดของ case study นี้ — row-level lock, MVCC, advisory lock, `SKIP LOCKED`
- **Indexing:** ดัชนีที่แม่นยำบน `product_id` (primary key) ต้องเร็วมากเพราะถูก query/update ถี่สุดขั้ว
- **Connection Pooling:** ในช่วง spike จำนวน connection ที่พยายามเปิดพร้อมกันอาจทำให้ PostgreSQL ล่มจาก connection exhaustion หากไม่มี pooler กั้นไว้
- **CQRS/Event Sourcing:** การแยก read model กับ write model สำหรับ inventory
- **Observability:** ต้องมี dashboard ที่ monitor lock wait time และ transaction throughput แบบ real-time ในช่วง event สำคัญ พร้อมทีม on-call เฝ้าระวังเป็นพิเศษ

---

## Step 989: Case Study 4 — ระบบ SaaS Multi-tenant B2B: จากไม่กี่ Tenant สู่หลักพันราย

### 989.1 โจทย์ทางธุรกิจ

**"TenantHub"** เป็นแพลตฟอร์ม SaaS B2B สมมติ (เช่น ระบบ HR, CRM, หรือ project management) ที่เริ่มต้นด้วยลูกค้าไม่กี่ราย แต่เติบโตอย่างรวดเร็วจนมี **tenant (บริษัทลูกค้า) หลายพันราย** แต่ละ tenant มีขนาดต่างกันมาก — ตั้งแต่ startup เล็ก ๆ 5 คน ไปจนถึงองค์กรใหญ่หลายพันพนักงาน โจทย์ที่ยากที่สุดของระบบ multi-tenant คือ:

- **Data isolation** — tenant A ต้องไม่มีทางเห็นข้อมูลของ tenant B แม้แต่กรณี bug ในโค้ด (นี่คือความเสี่ยงทางธุรกิจระดับ "จบบริษัท" ถ้าเกิดขึ้น)
- **Noisy neighbor problem** — tenant ใหญ่ที่มี query หนักไม่ควรทำให้ tenant เล็กช้าไปด้วย
- **ต้นทุนต่อ tenant** ต้องต่ำพอที่จะทำกำไรจาก tenant เล็ก ในขณะที่ยังรองรับ tenant ใหญ่ที่ต้องการ SLA สูงได้
- **Operational overhead** — เมื่อ tenant มีหลักพันราย การ migrate schema ต้องทำได้โดยไม่ downtime ในทุก tenant พร้อมกัน

**คำถามสถาปัตยกรรมหลัก:** ระหว่าง **Row-Level Security (RLS)**, **Schema-per-tenant**, และ **Database-per-tenant** ควรเลือกแบบไหน และคำตอบเปลี่ยนไปอย่างไรเมื่อ scale จาก 10 tenant ไปเป็น 10,000 tenant?

### 989.2 สถาปัตยกรรมภาพรวมของทั้ง 3 กลยุทธ์

```
กลยุทธ์ที่ 1: Shared Schema + Row-Level Security (RLS)
┌─────────────────────────────────────────────────────┐
│                 PostgreSQL Database                    │
│  ┌────────────────────────────────────────────────┐  │
│  │  ตาราง users, orders, projects ฯลฯ (shared schema)│  │
│  │  ทุกแถวมีคอลัมน์ tenant_id                        │  │
│  │  POLICY: tenant_id = current_setting('app.tid')   │  │
│  │  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐             │  │
│  │  │tenant│ │tenant│ │tenant│ │tenant│  ... (พันแถว) │  │
│  │  │  1   │ │  2   │ │  3   │ │  N   │             │  │
│  │  └──────┘ └──────┘ └──────┘ └──────┘             │  │
│  └────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────┘
   Connection pool ใช้ร่วมกันได้เต็มที่ (เชื่อม pooling ง่ายสุด)


กลยุทธ์ที่ 2: Shared Database + Schema-per-tenant
┌─────────────────────────────────────────────────────┐
│                 PostgreSQL Database                    │
│  ┌───────────┐  ┌───────────┐  ┌───────────┐         │
│  │ schema:    │  │ schema:    │  │ schema:    │  ...   │
│  │ tenant_1   │  │ tenant_2   │  │ tenant_3   │        │
│  │ (users,    │  │ (users,    │  │ (users,    │        │
│  │  orders..) │  │  orders..) │  │  orders..) │        │
│  └───────────┘  └───────────┘  └───────────┘         │
└─────────────────────────────────────────────────────┘
   search_path เปลี่ยนตาม tenant ที่ connect เข้ามา


กลยุทธ์ที่ 3: Database-per-tenant
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│ PostgreSQL DB │  │ PostgreSQL DB │  │ PostgreSQL DB │  ...
│  (tenant 1)   │  │  (tenant 2)   │  │  (tenant 3)   │
│  ทั้ง instance   │  │  ทั้ง instance   │  │  ทั้ง instance   │
│  หรือ database  │  │  หรือ database  │  │  หรือ database  │
│  แยกกันเต็มที่   │  │  แยกกันเต็มที่   │  │  แยกกันเต็มที่   │
└──────────────┘  └──────────────┘  └──────────────┘
   Provisioning ผ่าน control plane อัตโนมัติ (IaC)
```

### 989.3 การวิเคราะห์เชิงเปรียบเทียบ

**(1) Row-Level Security (RLS) — เหมาะกับช่วงเริ่มต้นถึงกลาง (10 – หลักพัน tenant เล็ก-กลาง)**

RLS คือการฝัง policy ไว้ในระดับ PostgreSQL engine เอง (`CREATE POLICY ... USING (tenant_id = current_setting('app.current_tenant')::uuid)`) ทำให้ต่อให้ query จาก application layer มี bug ลืมใส่ `WHERE tenant_id = ...` ตัว database engine เองก็จะกรองให้อัตโนมัติ — นี่คือหลักการ **defense in depth** ที่สำคัญที่สุดของสถาปัตยกรรม multi-tenant

ข้อดี:
- ต้นทุนต่อ tenant ต่ำที่สุด (schema เดียว, connection pool ใช้ร่วมกันได้เต็มประสิทธิภาพ)
- Migration schema ทำครั้งเดียวใช้กับทุก tenant พร้อมกัน (operational overhead ต่ำสุด)
- เหมาะกับ tenant จำนวนมากที่แต่ละรายมีขนาดข้อมูลไม่ใหญ่มาก

ข้อเสีย:
- **Noisy neighbor problem รุนแรงที่สุด** ในบรรดา 3 กลยุทธ์ — tenant ใหญ่ที่ query หนักกระทบ connection pool, buffer cache, และ lock contention ที่ tenant อื่นใช้ร่วมกันโดยตรง
- Index ต้องมี `tenant_id` เป็น leading column แทบทุกตัว มิฉะนั้น query planner อาจ scan ข้าม tenant โดยไม่จำเป็น (แม้ RLS จะกรองผลลัพธ์ถูกต้อง แต่ประสิทธิภาพแย่)
- ความเสี่ยงจาก misconfiguration ของ RLS policy (เช่น ลืมเปิด `FORCE ROW LEVEL SECURITY`, หรือ superuser/BYPASSRLS role หลุดเข้ามาใน connection pool) เป็นความเสี่ยงเชิง security ที่ต้อง audit สม่ำเสมอ
- ทำ backup/restore เฉพาะ tenant เดียว (เช่น ตอนลูกค้าขอ "export ข้อมูลทั้งหมด" หรือ "ลบข้อมูลทั้งหมดตาม GDPR") ทำได้ยากกว่า ต้อง query กรองออกมาเอง

**(2) Schema-per-tenant — จุดกึ่งกลางที่มักถูกมองข้าม**

แต่ละ tenant มี schema ของตัวเองภายใน database เดียวกัน โดย connection จะ set `search_path` ตาม tenant ที่ login

ข้อดี:
- Data isolation ดีกว่า RLS อย่างมีนัยสำคัญ (ผิดพลาดยากกว่า เพราะ query ที่ไม่ได้ set search_path ให้ถูกจะ error แทนที่จะได้ผลลัพธ์ปนกัน)
- Backup/restore เฉพาะ tenant ทำได้ง่ายกว่า RLS (`pg_dump --schema=tenant_x`)
- ยังใช้ connection pool ร่วมกันได้ (แม้ต้องระวังเรื่อง pool ที่ผูกกับ search_path — มักต้องใช้ session pooling แทน transaction pooling หรือ SET search_path ทุกครั้งที่หยิบ connection จาก pool)

ข้อเสีย:
- **Schema migration กลายเป็นฝันร้ายเมื่อ tenant เยอะ** — ต้องรัน DDL migration ซ้ำ N ครั้ง (ครั้งละ schema) ถ้ามี 5,000 tenant การ deploy migration หนึ่งครั้งอาจใช้เวลานานและเสี่ยงล้มเหลวกลางทาง (partial migration state)
- `pg_class`, `pg_attribute` และ system catalog โตตามจำนวน schema × จำนวนตาราง — ที่ scale หลักหมื่น schema อาจเริ่มกระทบ performance ของ catalog lookup และเครื่องมือ tooling บางตัวที่ไม่ได้ออกแบบมาให้รองรับ schema จำนวนมหาศาล
- Connection pooler ต้องฉลาดขึ้น (routing ตาม tenant → schema)

โดยทั่วไป schema-per-tenant จะเริ่ม "เจ็บ" เมื่อจำนวน tenant ทะลุหลักพันขึ้นไป เพราะ operational overhead ของการ migrate และ monitor เพิ่มเป็นเส้นตรงตามจำนวน schema

**(3) Database-per-tenant — เหมาะกับ tenant ใหญ่/enterprise ที่ต้องการ isolation สูงสุด**

แต่ละ tenant ได้ database (หรือแม้แต่ instance) ของตัวเองทั้งหมด

ข้อดี:
- **Isolation สมบูรณ์แบบที่สุด** — ไม่มีทางที่ query ของ tenant หนึ่งจะกระทบอีก tenant ได้เลยในระดับ engine (คนละ process, คนละ buffer cache, คนละ disk I/O ถ้าแยก instance จริง)
- รองรับ **customization ต่อ tenant** ได้ง่าย (เช่น enterprise customer ต้องการ extension พิเศษ, ต้องการ region เฉพาะเพื่อ data residency ตามกฎหมายท้องถิ่น)
- Backup/restore, scaling, และ resource allocation ทำต่อ tenant ได้อย่างอิสระเต็มที่ — เหมาะมากกับลูกค้าองค์กรใหญ่ที่จ่ายเงินสูงและต้องการ SLA เข้มงวด หรือสัญญาที่ระบุ data residency เฉพาะประเทศ
- ปิดหรือ offboard tenant ทำได้ง่ายมาก (drop database เดียว)

ข้อเสีย:
- **ต้นทุนต่อ tenant สูงที่สุด** — ถ้าแยกเป็น instance จริง ต้นทุน infrastructure และ operational overhead (monitoring, patching, backup ต่อ instance) โตเป็นเส้นตรงตามจำนวน tenant อย่างชัดเจน ไม่เหมาะกับ tenant จำนวนมากที่แต่ละรายจ่ายเงินน้อย
- Connection pool ไม่สามารถแชร์ข้ามได้ (ต้องมี pool แยกต่อ tenant หรือใช้ pooler ระดับ proxy ที่ route ตาม tenant)
- ต้องมี **control plane** ที่ทำ provisioning/deprovisioning อัตโนมัติ (Infrastructure as Code) เพราะ manual provisioning ไม่ scale เมื่อมี tenant ใหม่สมัครทุกวัน

**(4) กลยุทธ์ Hybrid ที่ระบบระดับ enterprise จริงมักใช้**

ในทางปฏิบัติ TenantHub (สถาปัตยกรรมต้นแบบ) ที่เติบโตจาก 10 ไปหลักพัน tenant มักจบลงด้วยกลยุทธ์แบบ **tiered / hybrid**:

- **Tenant ขนาดเล็ก-กลาง (ส่วนใหญ่ของฐานลูกค้า):** ใช้ **RLS บน shared database** เพื่อควบคุมต้นทุนและ operational overhead ให้ต่ำที่สุด
- **Tenant ขนาดใหญ่ / enterprise tier:** "แยกออก" ไปอยู่ database-per-tenant (หรืออย่างน้อย shard แยกที่มี tenant ใหญ่ไม่กี่รายต่อ shard) เพื่อป้องกัน noisy neighbor และตอบโจทย์ compliance/data residency ที่ enterprise customer มักต้องการ
- ใช้ **shard/pool routing layer** ที่ control plane เก็บ mapping ว่า tenant ไหนอยู่ที่ database/shard ไหน (คล้ายหลักการ sharding ใน Case Study 1) ทำให้ระบบสามารถ "ย้าย tenant" จาก shared pool ไปยัง dedicated instance ได้เมื่อ tenant นั้นเติบโตขึ้น โดยไม่กระทบสถาปัตยกรรมโดยรวม

นี่คือตัวอย่างที่ชัดเจนของหลักการที่ว่า **"ไม่มีกลยุทธ์เดียวที่ถูกต้องเสมอ — คำตอบขึ้นกับขนาดและลักษณะของ workload ในแต่ละช่วงของการเติบโต"**

### 989.4 ตารางสรุปเปรียบเทียบ

| ปัจจัย | RLS (Shared Schema) | Schema-per-tenant | Database-per-tenant |
|---|---|---|---|
| ต้นทุนต่อ tenant | ต่ำที่สุด | ปานกลาง | สูงที่สุด |
| Data isolation | ต่ำสุด (ต้องพึ่ง policy ถูกต้อง) | ปานกลาง-ดี | สูงสุด |
| Noisy neighbor risk | สูงที่สุด | ปานกลาง | ต่ำสุด (แทบไม่มี) |
| ความง่ายของ schema migration | ง่ายที่สุด (ครั้งเดียวจบ) | ยาก (ต้องรัน N ครั้ง) | ยาก (ต้องรัน N ครั้ง + coordination) |
| เหมาะกับจำนวน tenant | มาก (หมื่น-แสน) | ปานกลาง (ร้อย-พัน) | น้อย-ปานกลาง (สิบ-ร้อย) หรือ tenant enterprise เฉพาะราย |
| Customization ต่อ tenant | ยากสุด | ปานกลาง | ง่ายสุด |
| รองรับ data residency ตามกฎหมาย | ยาก | ปานกลาง | ง่ายสุด |

### 989.5 เชื่อมโยงกับบทก่อนหน้า

- **Row-Level Security:** พื้นฐานสำคัญที่สุดของกลยุทธ์ RLS, รวมถึง `FORCE ROW LEVEL SECURITY` และการจัดการ role ที่ไม่มี `BYPASSRLS`
- **Indexing:** composite index ที่มี `tenant_id` เป็น leading column แทบทุกตารางในโมเดล shared schema
- **Connection Pooling:** ความซับซ้อนของการ route connection ตาม tenant ในทั้ง schema-per-tenant และ database-per-tenant
- **Sharding:** แนวคิด "control plane + routing layer" เดียวกับที่ใช้ใน Case Study 1 ถูกนำมาปรับใช้กับการ route tenant ไป shard/instance ที่ถูกต้อง
- **Backup & Compliance:** ความสามารถในการ export/delete ข้อมูลของ tenant เดียว (เช่นตาม GDPR "right to be forgotten") ที่ยากง่ายต่างกันตามกลยุทธ์ที่เลือก

---

## Step 990: บทเรียนร่วมที่พบในทุก Case Study — Pattern ที่ซ้ำในระบบระดับ Enterprise

เมื่อมองทั้ง 4 case study พร้อมกัน จะเห็นว่าแม้โจทย์ทางธุรกิจจะต่างกันโดยสิ้นเชิง (feed, การเงิน, e-commerce, SaaS) แต่มี **pattern ทางสถาปัตยกรรมร่วม** ที่ปรากฏซ้ำเสมอ นี่คือสิ่งที่แยก "วิศวกรที่ท่องจำเทคนิค" ออกจาก "สถาปนิกระบบที่แท้จริง"

### 990.1 Defense in Depth (การป้องกันหลายชั้น)

ไม่มีระบบไหนพึ่งกลไกป้องกันเพียงชั้นเดียว:

- **FeedFlow:** cache layer + read replica + rate limiting ที่ API gateway (หลายชั้นป้องกันไม่ให้ traffic ถล่ม DB)
- **PayCore:** idempotency key + `SELECT FOR UPDATE` + `SERIALIZABLE` + synchronous replication + audit log (หลายชั้นป้องกันการสูญเสียเงิน)
- **ShopScale:** virtual waiting room + Redis pre-check + atomic conditional UPDATE (หลายชั้นป้องกัน overselling)
- **TenantHub:** RLS policy + application-layer filtering + (สำหรับ tier สูง) database isolation ทางกายภาพ (หลายชั้นป้องกัน data leak ข้าม tenant)

หลักคิด: **อย่าไว้ใจว่าชั้นใดชั้นหนึ่งจะไม่มีวันล้มเหลว** — ทุกชั้นควรออกแบบให้ล้มเหลวได้อย่างปลอดภัย (fail-safe) โดยมีชั้นถัดไปรองรับ

### 990.2 Graceful Degradation (การเสื่อมสภาพอย่างมีการควบคุม)

ระบบระดับ enterprise ไม่พยายาม "ทำทุกอย่างให้สมบูรณ์แบบเสมอ" แต่ออกแบบให้รู้ว่า **อะไรคือ critical path ที่ต้องรักษาไว้เสมอ กับอะไรคือ feature รองที่ยอมเสียสละได้เมื่อจำเป็น**:

- FeedFlow ยอม feed ไม่ real-time 100% เพื่อแลกกับความเร็ว
- ShopScale ยอมปิด recommendation engine เพื่อรักษา checkout path
- PayCore ไม่ยอม degrade เรื่อง consistency แม้แต่น้อย (เพราะ critical path ของมันคือความถูกต้อง ไม่ใช่ความเร็ว) — นี่คือตัวอย่างว่า **"critical path" นิยามต่างกันตามลักษณะธุรกิจ**

### 990.3 Observability First (สังเกตการณ์ได้ก่อนพัง ไม่ใช่หลังพัง)

ทุกระบบต้อง monitor ตัวชี้วัดที่เฉพาะเจาะจงกับความเสี่ยงของตัวเอง **ก่อน** ที่จะเกิดปัญหาจริง:

- FeedFlow: cache hit ratio, replication lag, query latency p99
- PayCore: reconciliation mismatch alert (ยอด ledger ไม่ balance แม้เพียงสตางค์เดียวต้อง alert ทันที), synchronous replica ack latency
- ShopScale: lock wait time, transaction throughput ต่อวินาทีระหว่าง flash sale
- TenantHub: per-tenant resource usage (เพื่อจับ noisy neighbor ก่อนที่ tenant อื่นจะร้องเรียน), RLS policy coverage audit

หลักคิดร่วม: **"คุณไม่สามารถแก้ปัญหาที่คุณมองไม่เห็น"** — การลงทุนใน observability (metrics, logging, tracing, alerting) ไม่ใช่ทางเลือก แต่เป็นส่วนหนึ่งของสถาปัตยกรรมตั้งแต่วันแรก ไม่ใช่สิ่งที่ "เพิ่มทีหลังเมื่อมีปัญหา"

### 990.4 Read/Write Path Separation

ทุกระบบแยก "เส้นทางอ่าน" ออกจาก "เส้นทางเขียน" อย่างชัดเจน แม้จะใช้เทคนิคต่างกัน (cache+replica ใน FeedFlow, primary-only สำหรับ ledger ใน PayCore, CQRS ใน ShopScale) เพราะ **ความต้องการด้าน consistency ของการอ่านกับการเขียนแทบไม่เคยเท่ากัน** — การเขียนมักต้องการ strong consistency เสมอ ในขณะที่การอ่านส่วนใหญ่ยอมรับ eventual consistency ได้ถ้าแลกกับ scale ที่สูงกว่ามาก

### 990.5 Idempotency และการออกแบบเพื่อความล้มเหลว (Design for Failure)

เครือข่ายและระบบกระจาย (distributed system) ล้มเหลวเสมอ — ไม่ใช่ "ถ้า" แต่คือ "เมื่อไหร่" ทุกระบบข้างต้นออกแบบโดยสมมติว่า:

- Request อาจถูกส่งซ้ำ (retry) → ต้องมี idempotency key
- Replica อาจ lag → ต้องออกแบบ UX ให้ทนต่อความล่าช้าได้ หรือ route ไป primary เมื่อจำเป็น (read-your-writes)
- Node อาจล่มกลาง transaction → ต้องพึ่ง WAL, replication, และ atomic commit ของ PostgreSQL เอง

### 990.6 Data Locality และการเลือก Partition/Shard Key อย่างมีเหตุผล

ทั้ง sharding by `user_id` (FeedFlow), stock sharding (ShopScale), และ tenant routing (TenantHub) ล้วนสะท้อนหลักการเดียวกัน: **เลือก key ที่ทำให้ query ส่วนใหญ่เป็น "local" ต่อ shard/partition เดียว** เพื่อหลีกเลี่ยง cross-shard join หรือ cross-shard transaction ที่มีต้นทุนสูงมากและซับซ้อนในการรับประกัน consistency

---

## สรุปท้ายบท

บทนี้พาเราเดินผ่าน 4 สถาปัตยกรรมต้นแบบที่แม้โจทย์ทางธุรกิจจะต่างกันสุดขั้ว — จาก feed ที่อ่านหนักสุดขีด, ระบบการเงินที่ต้องแม่นยำ 100%, e-commerce ที่ต้องรับ traffic spike รุนแรง, ไปจนถึง SaaS ที่ต้องขยายจากลูกค้าไม่กี่รายไปหลักพันราย — แต่ทั้งหมดล้วนสร้างขึ้นจาก **เทคนิคชุดเดียวกัน** ที่เราเรียนมาตลอดหลักสูตรนี้: indexing, replication, partitioning/sharding, locking/concurrency control, row-level security, connection pooling, และ observability

สิ่งที่แยกวิศวกรระดับ world-class ออกจากวิศวกรทั่วไปไม่ใช่การรู้จักเทคนิคเหล่านี้แยกกัน แต่คือความสามารถในการ **วิเคราะห์ constraint ทางธุรกิจ แล้วเลือกประกอบเทคนิคเหล่านั้นเข้าด้วยกันอย่างมีเหตุผล** โดยเข้าใจ trade-off ของทุกการตัดสินใจอย่างถ่องแท้ — ไม่มีสถาปัตยกรรมใดที่ "ถูกต้องที่สุด" ในสุญญากาศ มีแต่สถาปัตยกรรมที่ "เหมาะสมที่สุดกับ constraint ที่กำหนด" เท่านั้น

จำไว้เสมอว่า case study ในบทนี้เป็นสถาปัตยกรรมต้นแบบเพื่อการเรียนรู้ ไม่ใช่พิมพ์เขียวที่นำไปใช้ได้ตรง ๆ กับทุกระบบ — งานจริงของสถาปนิกระบบคือการวิเคราะห์โจทย์เฉพาะหน้าใหม่ทุกครั้ง โดยใช้กรอบความคิดที่ได้จากบทนี้เป็นจุดตั้งต้น

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1

FeedFlow (Case Study 1) ตัดสินใจใช้ hybrid fan-out (push สำหรับบัญชีทั่วไป, pull สำหรับ celebrity account) จงอธิบายว่าทำไมการใช้ push model ล้วน ๆ กับบัญชีที่มี follower 50 ล้านคนถึงเป็นปัญหาในเชิงวิศวกรรม และเสนอเกณฑ์ (threshold) ที่สมเหตุสมผลสำหรับการสลับจาก push ไป pull

<details>
<summary>เฉลย</summary>

หากใช้ push model ล้วน ทุกครั้งที่ celebrity account (follower 50 ล้านคน) โพสต์ 1 ครั้ง ระบบต้องเขียนเข้า timeline cache ของผู้ติดตามทั้ง 50 ล้านคนทันที นั่นหมายถึง **write amplification 50 ล้านเท่า** จากการกระทำเดียว ถ้า celebrity โพสต์บ่อย (หลายครั้งต่อวัน) หรือมีหลาย celebrity โพสต์พร้อมกัน (เช่น ช่วง event ใหญ่) ระบบ fan-out worker queue จะถูกถล่มจน backlog สะสมมหาศาล ทำให้ follower เห็นโพสต์ล่าช้าไปหลายนาทีถึงหลายชั่วโมง อีกทั้งยังสิ้นเปลือง storage cache สำหรับข้อมูลที่ซ้ำซ้อนมหาศาล (เขียนโพสต์เดียวกันซ้ำ 50 ล้านชุด)

เกณฑ์ที่สมเหตุสมผลไม่ใช่ตัวเลขตายตัว แต่ควรพิจารณาจาก **ต้นทุนรวมของการ fan-out เทียบกับต้นทุนของการ pull ตอนอ่าน** เช่น ถ้า follower count เกินหลักหมื่นถึงแสน (ตัวเลขต้องปรับตาม throughput ของ queue และความถี่การโพสต์ของบัญชีนั้น) ต้นทุนของการ push (เขียนซ้ำจำนวนมหาศาลทุกครั้งที่โพสต์) จะแพงกว่าต้นทุนของการให้ follower query แบบ pull ตอนเปิด feed (ซึ่งเกิดขึ้นเพียงครั้งเดียวต่อการเปิดแอปของแต่ละคน ไม่ใช่ทุกครั้งที่ celebrity โพสต์) ระบบจริงมักกำหนด threshold แบบ dynamic และ monitor อัตราการโพสต์ร่วมด้วย ไม่ใช่แค่จำนวน follower อย่างเดียว
</details>

### แบบฝึกหัดที่ 2

ในสถาปัตยกรรม PayCore (Case Study 2) เหตุใดระบบจึงใช้ **synchronous replication** แทนที่จะใช้ asynchronous replication ที่มี latency ต่ำกว่าและ throughput สูงกว่าแบบที่ FeedFlow ใช้? อธิบายในมุมของ RPO (Recovery Point Objective)

<details>
<summary>เฉลย</summary>

RPO (Recovery Point Objective) คือ "ปริมาณข้อมูลสูงสุดที่ยอมสูญเสียได้เมื่อเกิดความล้มเหลว" วัดเป็นเวลา ระบบการเงินอย่าง PayCore ต้องการ **RPO = 0** อย่างเคร่งครัด หมายความว่าเมื่อ transaction ถูกตอบกลับว่า "COMMIT สำเร็จ" แล้ว จะต้องไม่มีทางที่ transaction นั้นจะหายไปแม้ primary node จะล่มทันทีในเสี้ยววินาทีถัดมา

ด้วย asynchronous replication แบบที่ FeedFlow ใช้ primary จะตอบ COMMIT สำเร็จทันทีที่เขียนลง WAL ของตัวเองโดยไม่รอ standby ยืนยัน — ถ้า primary ล่มก่อนที่ WAL จะถูกส่งไปถึง standby (ซึ่งเป็นไปได้เสมอในสภาวะจริง) transaction ที่เพิ่ง COMMIT ไปจะหายไปพร้อมกับ primary นั่นหมายถึง "เงินหาย" ซึ่งยอมรับไม่ได้ในระบบการเงิน

ในทางกลับกัน FeedFlow ยอมรับ asynchronous replication ได้เพราะการสูญเสียโพสต์หรือ like หนึ่งรายการช่วงเสี้ยววินาทีของการ failover ไม่ใช่ความเสียหายร้ายแรงในเชิงธุรกิจ (ยอมรับ RPO ที่ไม่ใช่ศูนย์ได้) แต่ได้ latency ที่ต่ำกว่าและ throughput ที่สูงกว่ามาก — นี่คือตัวอย่างชัดเจนของการที่ **ธุรกิจกำหนดค่า RPO/RTO ที่ยอมรับได้ และสถาปัตยกรรมต้องออกแบบตามนั้น ไม่ใช่เลือกเทคนิคที่ "เร็วที่สุด" เสมอไป**
</details>

### แบบฝึกหัดที่ 3

จงเขียน SQL statement ที่ถูกต้องสำหรับการหักสต็อกสินค้าใน ShopScale (Case Study 3) ที่ป้องกัน race condition ได้อย่างสมบูรณ์ พร้อมอธิบายว่าทำไมวิธีนี้จึงปลอดภัยกว่าการ SELECT แล้วค่อย UPDATE แยกกัน

<details>
<summary>เฉลย</summary>

```sql
UPDATE inventory
   SET stock = stock - 1
 WHERE product_id = 123
   AND stock > 0
RETURNING stock;
```

จากนั้นตรวจสอบใน application ว่า query คืนแถวกลับมาหรือไม่ (`rowCount > 0`) — ถ้าไม่มีแถวคืนกลับมา แปลว่าสต็อกหมดแล้ว ควรตอบผู้ใช้ว่า "สินค้าหมด" โดยไม่ต้องทำ transaction ต่อ

เหตุผลที่ปลอดภัยกว่าการ `SELECT` แล้ว `UPDATE` แยกคำสั่งกัน:

1. **Atomicity ระดับ statement เดียว:** PostgreSQL รับประกันว่า UPDATE หนึ่งคำสั่งเป็น atomic operation — ไม่มี transaction อื่นสามารถ "แทรก" เข้ามาระหว่างการตรวจสอบเงื่อนไข (`stock > 0`) กับการเขียนค่าใหม่ได้ เพราะทั้งสองเกิดขึ้นพร้อมกันภายใต้ row lock เดียว
2. **ไม่มี race window:** การแยก SELECT กับ UPDATE สร้างช่วงเวลา (window) ที่ transaction สองตัวสามารถอ่านค่า stock เดียวกันได้พร้อมกัน (เช่นทั้งคู่เห็น stock=1) แล้วทั้งคู่ตัดสินใจว่า "พอขายได้" และรันคำสั่ง UPDATE ทับกัน ทำให้ stock ติดลบ (oversell) — atomic UPDATE ไม่มีช่องให้เกิดสถานการณ์นี้
3. **ใช้ MVCC row lock โดยอัตโนมัติ:** เมื่อ transaction แรกกำลัง UPDATE แถวนั้น transaction ที่สองที่พยายาม UPDATE แถวเดียวกันจะถูก PostgreSQL บล็อกให้รอโดยอัตโนมัติ (ไม่ต้องเขียน explicit lock เอง) จนกว่า transaction แรกจะ commit หรือ rollback แล้วจึงเห็นค่าล่าสุดที่ถูกต้อง
</details>

### แบบฝึกหัดที่ 4

TenantHub (Case Study 4) เริ่มต้นด้วย RLS สำหรับทุก tenant แต่เมื่อมี tenant enterprise รายใหญ่รายหนึ่งเซ็นสัญญาที่กำหนดว่า **ข้อมูลต้องอยู่ในสหภาพยุโรปเท่านั้น (data residency)** และห้ามแชร์ infrastructure กับลูกค้ารายอื่น จงวิเคราะห์ว่าทีมควรทำอย่างไร โดยไม่กระทบ tenant รายอื่นที่ยังอยู่บน shared RLS model

<details>
<summary>เฉลย