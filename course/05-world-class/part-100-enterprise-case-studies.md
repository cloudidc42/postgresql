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

**วิธีอ่านบทนี้ให้ได้ประโยชน์สูงสุด:** แต่ละ case study (Step 986-989) มีโครงสร้างเดียวกันเสมอ เริ่มจาก "โจทย์ทางธุรกิจ" (เพื่อฝึกอ่านและตีความ requirement ก่อนคิดถึง solution — ทักษะที่สำคัญกว่าการรู้จักเทคนิคเสียอีก) ตามด้วย "สถาปัตยกรรมภาพรวม" ในรูปแบบ ASCII diagram (เพื่อฝึกอ่านและวาด system diagram ซึ่งเป็นภาษากลางที่วิศวกรระดับ senior ใช้สื่อสารกัน) ตามด้วย "การตัดสินใจสำคัญ" ที่อธิบายเหตุผลเบื้องหลังทุกทางเลือก และปิดท้ายด้วย "trade-off" กับ "การเชื่อมโยงกับบทก่อนหน้า" แนะนำให้อ่านแต่ละ case study แบบช้า ๆ พร้อมลองปิดหนังสือแล้วอธิบายสถาปัตยกรรมนั้นด้วยคำพูดของตัวเองก่อนเปิดอ่านต่อ — นี่คือวิธีฝึกที่ใช้ได้ผลจริงสำหรับการเตรียมตัวสัมภาษณ์งานสาย system design

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

### 986.6 วิวัฒนาการของสถาปัตยกรรมตามสเกล (Evolution Timeline)

สิ่งสำคัญที่มักถูกมองข้ามคือ FeedFlow **ไม่ได้เกิดมาพร้อมสถาปัตยกรรมข้างต้นตั้งแต่วันแรก** ระบบทุกระบบเติบโตเป็นขั้นบันได และการลงทุนสร้างความซับซ้อนของ sharding/fan-out ตั้งแต่ผู้ใช้ยังมีไม่กี่พันคนคือ **premature optimization** ที่สิ้นเปลืองโดยใช่เหตุ ตารางด้านล่างแสดงเส้นทางวิวัฒนาการที่สมเหตุสมผล:

| ระยะ | ขนาดผู้ใช้ (DAU) | สถาปัตยกรรมที่เหมาะสม | สัญญาณที่บอกว่าต้องขยับไประยะถัดไป |
|---|---|---|---|
| เริ่มต้น | < 10,000 | PostgreSQL เครื่องเดียว, ไม่มี cache, query ตรง | latency p99 เริ่มเกิน 200ms, CPU primary สูงต่อเนื่อง |
| เติบโต | 10,000 – 1 ล้าน | เพิ่ม read replica 1-2 ตัว + cache layer (Redis) สำหรับ query ที่ query ซ้ำบ่อย | replica lag เริ่มกระทบ UX, cache hit ratio ต่ำกว่า 80% |
| ขยายใหญ่ | 1 ล้าน – 20 ล้าน | เพิ่ม read replica เป็นสิบตัว, cache แบบ tiered, เริ่มแยก service ตามโดเมน (feed/post/graph) | primary รับ write throughput ไม่ไหว, storage เดี่ยวเข้าใกล้ขีดจำกัด disk |
| ระดับ Enterprise | 20 ล้าน+ | Sharding ตาม user_id, hybrid fan-out, geo-distributed replica | เมื่อถึงจุดนี้แล้วการไม่ shard หมายถึง single point of failure ทางธุรกิจทั้งหมด |

หลักคิดสำคัญคือ **อย่า shard ก่อนที่จะจำเป็นจริง ๆ** เพราะ sharding เพิ่ม operational complexity มหาศาล (schema migration ต้องรันทุก shard, cross-shard query ทำไม่ได้ตรง ๆ, debugging ยากขึ้น) — ทีมวิศวกรรมที่ดีจะ **ยืดอายุของสถาปัตยกรรมแบบง่ายให้นานที่สุดเท่าที่ทำได้** โดยใช้ vertical scaling, indexing ที่ดี, และ caching ให้ถึงขีดสุดก่อน แล้วจึงขยับไปสู่ความซับซ้อนที่สูงขึ้นเมื่อมีหลักฐานชัดเจนว่าจำเป็นจริง ๆ เท่านั้น (data-driven scaling decision ไม่ใช่ scaling ตามความกลัว)

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

### 987.6 กรณีศึกษาย่อย: เมื่อ Reconciliation Job พบยอดไม่ตรงกัน

เพื่อให้เห็นภาพว่า "defense in depth" ทำงานอย่างไรในทางปฏิบัติ ลองพิจารณาสถานการณ์ incident สมมติ: ทุกคืนเวลาตี 2 ระบบ PayCore จะรัน **reconciliation job** ที่ตรวจสอบว่าผลรวมของทุก ledger entry ในแต่ละบัญชีเท่ากับยอดคงเหลือที่ cache ไว้หรือไม่ (SUM ของ debit/credit ทั้งหมดต้องเท่ากับ balance snapshot) คืนหนึ่ง job พบว่ามีบัญชีหนึ่งยอดไม่ตรงกันเป็นจำนวนเล็กน้อย

**ขั้นตอนที่ระบบและทีมงานควรทำ (runbook แบบย่อ):**

1. **Alert ทันทีระดับ Sev-1** — ต่างจากระบบอื่นที่ยอมรับ error rate เล็กน้อยได้ ระบบการเงินต้อง alert ทุกครั้งที่ reconciliation ไม่ balance แม้แต่สตางค์เดียว เพราะนี่คือสัญญาณของบั๊กที่อาจกระทบบัญชีอื่นด้วย
2. **หยุดการประมวลผลธุรกรรมใหม่ของบัญชีที่มีปัญหาชั่วคราว (freeze)** โดยไม่กระทบบัญชีอื่นในระบบ (ตัวอย่างของ blast radius containment)
3. **ตรวจสอบ audit log แบบ append-only** ย้อนหลังเพื่อดูว่า ledger entry ใดที่ทำให้ยอดไม่ตรง เนื่องจากตาราง ledger เป็น append-only การสืบสวนจึงทำได้แม่นยำ 100% (ไม่มีข้อมูลถูกเขียนทับ)
4. **แก้ไขด้วยการเพิ่ม correcting entry ใหม่ ไม่ใช่แก้ไขข้อมูลเดิม** — หลักการบัญชีคือห้ามลบ/แก้ entry เก่าเด็ดขาด แม้จะพบว่าผิดพลาด ต้องเพิ่ม entry ใหม่ที่ "แก้ไขผล" แทน เพื่อรักษาความสมบูรณ์ของ audit trail
5. **Post-incident review** เพื่อหาสาเหตุที่รากเหง้า (root cause) และปรับปรุงกลไกป้องกันชั้นที่เกี่ยวข้อง (เช่น ถ้าสาเหตุมาจาก retry ที่ idempotency key ตรวจจับไม่ทัน ต้องปรับ logic การเช็ค key)

กรณีศึกษาย่อยนี้แสดงให้เห็นว่า "audit trail ที่แก้ไขไม่ได้" (Step 987.3 ข้อ 1) ไม่ใช่แค่ข้อกำหนด compliance บนกระดาษ แต่เป็นเครื่องมือสืบสวนที่ใช้งานจริงเมื่อเกิดปัญหา และการ "ห้ามแก้ข้อมูลเก่า" คือหลักการเดียวกับที่นักบัญชีใช้มาเป็นร้อยปีก่อนที่จะมีฐานข้อมูล เพียงแต่ตอนนี้ PostgreSQL เป็นผู้บังคับใช้กฎนี้ผ่าน schema design แทนวินัยของมนุษย์

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

### 988.6 Timeline เหตุการณ์จริงในคืน Black Friday (นาทีต่อนาที)

เพื่อให้เห็นภาพว่าการตัดสินใจทางสถาปัตยกรรมทั้งหมดถูกนำมาใช้จริงอย่างไรในสถานการณ์กดดัน ลองไล่ timeline สมมติของทีม on-call ในคืนเปิดดีลใหญ่:

- **T-30 นาที:** ทีมเปิด virtual waiting room ที่ edge layer ล่วงหน้า, เพิ่มจำนวน read replica ชั่วคราว (pre-scaling ตามที่คาดการณ์ไว้ล่วงหน้า แทนที่จะรอ auto-scaling ตอบสนอง), แจ้งเตือนทีม on-call ทั้งหมดให้พร้อมเฝ้าระวัง
- **T-0 (เวลาเปิดดีล):** traffic พุ่งขึ้นทันที 40 เท่าภายใน 60 วินาที — virtual waiting room เริ่มทำงาน ปล่อย request เข้าระบบเป็นอัตราที่ควบคุมได้ แทนที่จะปล่อยทั้งหมดพร้อมกัน
- **T+2 นาที:** dashboard แสดง lock wait time บนสินค้า flash-sale บางรายการเริ่มสูงขึ้น (มากกว่า 500ms) — Redis pre-check layer เริ่มกรอง request ที่ "ไม่มีทางสำเร็จ" ออกไปได้ตามคาด ลด transaction ที่ไปถึง PostgreSQL ลงกว่า 90%
- **T+5 นาที:** สินค้า flash-sale บางรายการหมดสต็อกจริง — atomic conditional UPDATE ทำงานถูกต้อง ไม่มี oversell แม้แต่ชิ้นเดียวตามการตรวจสอบหลัง event
- **T+8 นาที:** CPU ของ read replica pool เริ่มสูงเกิน 85% จาก traffic การ browse สินค้าที่ไม่เกี่ยวกับ flash-sale — ระบบเปิด feature flag graceful degradation อัตโนมัติ ปิด recommendation engine ชั่วคราวเพื่อสงวน capacity ให้ checkout path
- **T+15 นาที:** traffic เริ่มคงที่ในระดับสูงแต่ควบคุมได้ ทีม on-call ค่อย ๆ ผ่อนคลาย rate limit ที่ virtual waiting room ให้ผู้ใช้เข้าระบบได้มากขึ้น
- **T+60 นาที:** เปิด recommendation engine กลับมาเมื่อโหลดลดลงสู่ระดับปกติ, เริ่มกระบวนการ post-event review

Timeline นี้แสดงให้เห็นว่า **สถาปัตยกรรมที่ดีไม่ได้ทำงานแบบ all-or-nothing** แต่เป็นระบบของกลไกหลายชั้นที่ทำงานร่วมกันแบบ graduated response — แต่ละชั้นเปิดใช้งานตามระดับความรุนแรงของสถานการณ์ ไม่ใช่ทุกกลไกทำงานเต็มกำลังตลอดเวลา

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

**ตัวอย่าง RLS policy ที่ TenantHub ใช้จริงสำหรับ tenant ในระดับ shared pool:**

```sql
-- เปิด RLS และบังคับใช้แม้กับเจ้าของตาราง (สำคัญมาก มิฉะนั้น superuser/owner จะ bypass ได้)
ALTER TABLE projects ENABLE ROW LEVEL SECURITY;
ALTER TABLE projects FORCE ROW LEVEL SECURITY;

-- Policy: มองเห็นได้เฉพาะแถวของ tenant ตัวเองเท่านั้น
CREATE POLICY tenant_isolation_policy ON projects
    USING (tenant_id = current_setting('app.current_tenant')::uuid);

-- Application ต้อง set ค่านี้ทุกครั้งที่หยิบ connection จาก pool มาใช้งาน
-- (สำคัญเป็นพิเศษเมื่อใช้ transaction pooling ที่ connection ถูกใช้ซ้ำข้าม request)
SET app.current_tenant = '3f29c1a4-...';

-- Index ที่ต้องมีคู่กันเสมอ เพื่อให้ query planner narrow ลงมาที่ tenant เดียวได้เร็ว
CREATE INDEX idx_projects_tenant_created
    ON projects (tenant_id, created_at DESC);
```

จุดที่ต้องระวังที่สุดในโค้ดข้างต้นคือบรรทัด `FORCE ROW LEVEL SECURITY` — ถ้าลืมบรรทัดนี้ role ที่เป็นเจ้าของตาราง (table owner) จะสามารถมองเห็นข้อมูลของทุก tenant ได้โดยไม่ผ่าน policy เลย ซึ่งเป็นข้อผิดพลาดด้าน configuration ที่พบบ่อยที่สุดในระบบ multi-tenant ที่ใช้ RLS และควรมีการ audit ตั้งค่านี้เป็นประจำ (automated compliance check) ไม่ใช่ตรวจสอบแค่ตอน setup ครั้งแรก

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

### 989.6 เกณฑ์การตัดสินใจ "เมื่อไหร่ควรย้าย tenant ออกจาก shared pool"

คำถามที่ทีมวิศวกรรมของ TenantHub ต้องตอบซ้ำ ๆ ตลอดการเติบโตคือ "tenant รายนี้ควรอยู่บน shared RLS pool ต่อไป หรือควรย้ายไป dedicated resource แล้ว" ตารางด้านล่างสรุปสัญญาณที่ใช้ประกอบการตัดสินใจ ซึ่งเป็นตัวอย่างของการนำหลักการ **observability-first** (Step 990.3) มาใช้ตัดสินใจเชิงสถาปัตยกรรมโดยตรง:

| สัญญาณที่วัดได้ | ความหมาย | การตอบสนอง |
|---|---|---|
| Tenant คิดเป็นสัดส่วน CPU/IO เกิน 5-10% ของทั้ง cluster อย่างต่อเนื่อง | สัญญาณ noisy neighbor ชัดเจน | พิจารณาย้ายไป dedicated instance |
| สัญญาแจ้งข้อกำหนด data residency หรือ compliance เฉพาะ (เช่น HIPAA, GDPR เข้มงวด) | ข้อกำหนดทางกฎหมาย ไม่ใช่แค่ performance | ต้องย้ายทันทีไม่ว่าขนาด tenant จะเล็กแค่ไหน |
| จำนวน query ต่อวินาทีของ tenant สูงกว่าค่าเฉลี่ยของ tenant อื่น 100 เท่าขึ้นไป | ขนาดต่างจาก tenant ทั่วไปมาก | ประเมินต้นทุน dedicated resource เทียบกับรายได้จาก tenant นั้น |
| ลูกค้าจ่ายค่าบริการในระดับ enterprise tier ที่มี SLA สูง (เช่น 99.99%) | ธุรกิจต้องการรับประกัน isolation ระดับสัญญา | ย้ายเป็นส่วนหนึ่งของ onboarding process สำหรับ tier นี้โดยอัตโนมัติ |

หลักคิดคือ **การย้าย tenant ไม่ใช่การแก้ปัญหาเฉพาะหน้าแบบ ad-hoc แต่ควรเป็นกระบวนการที่มี threshold ชัดเจนและอัตโนมัติได้บางส่วน** เพื่อให้ทีมวิศวกรรมไม่ต้องตัดสินใจแบบ "ดับไฟ" ทุกครั้งที่มี tenant ใหญ่รายใหม่เข้ามา แต่มีกรอบการตัดสินใจที่ทำซ้ำได้และปรับตาม data จริง

---

## Step 990: บทเรียนร่วมที่พบในทุก Case Study — Pattern ที่ซ้ำในระบบระดับ Enterprise

เมื่อมองทั้ง 4 case study พร้อมกัน จะเห็นว่าแม้โจทย์ทางธุรกิจจะต่างกันโดยสิ้นเชิง (feed, การเงิน, e-commerce, SaaS) แต่มี **pattern ทางสถาปัตยกรรมร่วม** ที่ปรากฏซ้ำเสมอ นี่คือสิ่งที่แยก "วิศวกรที่ท่องจำเทคนิค" ออกจาก "สถาปนิกระบบที่แท้จริง"

### 990.0 ตารางเปรียบเทียบทั้ง 4 Case Study ในมุมเดียวกัน

ก่อนสรุป pattern ร่วม ลองเปรียบเทียบ 4 ระบบผ่านมิติเดียวกันเพื่อเห็นความแตกต่างและความเหมือนอย่างชัดเจน:

| มิติ | FeedFlow (Social) | PayCore (Fintech) | ShopScale (E-commerce) | TenantHub (SaaS) |
|---|---|---|---|---|
| ตัวชี้วัดสำคัญที่สุด | Latency / Freshness | Correctness / RPO=0 | Throughput ช่วง spike | Isolation / ต้นทุนต่อ tenant |
| Consistency model หลัก | Eventual consistency (ส่วนใหญ่) | Strong consistency (ทุกที่) | Strong เฉพาะ write path, eventual สำหรับ read | ขึ้นกับ tier ของ tenant |
| กลไกป้องกันหลัก | Cache + replica + rate limit | Locking + sync replication + audit | Atomic UPDATE + pre-check + traffic shaping | RLS + routing layer |
| จุดคอขวดหลัก | Read throughput | Write correctness ภายใต้ failure | Lock contention บน hot row | Noisy neighbor ระหว่าง tenant |
| สิ่งที่ยอมสละเพื่อ scale | ความสด (freshness) เล็กน้อย | Latency (ยอมช้าลง) | UX บางส่วน (ต้องรอคิว) | Customization ต่อ tenant (สำหรับ tier ต่ำ) |

ตารางนี้แสดงให้เห็นชัดเจนว่า **ไม่มีระบบใดที่ "ดีที่สุดในทุกมิติ" ได้พร้อมกัน** — ทุกสถาปัตยกรรมคือการเลือกว่าจะยอมสละอะไรเพื่อให้ได้สิ่งที่สำคัญที่สุดสำหรับธุรกิจนั้น ๆ มา ซึ่งเป็นแก่นของทฤษฎี **CAP theorem** และหลักการ trade-off ที่เรียนมาตลอดหลักสูตร

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

### 990.7 ต้นทุน (Cost) เป็น Constraint ทางสถาปัตยกรรมเสมอ ไม่ใช่แค่ปัจจัยรอง

บทเรียนสุดท้ายที่มักถูกมองข้ามในห้องเรียนแต่เป็นความจริงข้อแรกในที่ทำงานจริงคือ **ทุกการตัดสินใจทางสถาปัตยกรรมมีป้ายราคากำกับอยู่เสมอ**:

- FeedFlow เลือก async replication ไม่ใช่แค่เพราะ "เร็วกว่า" แต่เพราะ synchronous replication ในระดับ traffic ของมันจะหมายถึงต้นทุน infrastructure ที่สูงขึ้นมหาศาลโดยไม่คุ้มกับความเสี่ยงทางธุรกิจที่ลดลง
- PayCore ยอมจ่ายต้นทุนที่สูงกว่า (synchronous replication, quorum commit, dedicated compliance tooling) เพราะความเสี่ยงของการสูญเสียเงินแม้เพียงครั้งเดียวมีมูลค่าสูงกว่าต้นทุน infrastructure ที่เพิ่มขึ้นมากในเชิงเปรียบเทียบ
- TenantHub เลือก RLS เป็นค่าเริ่มต้นสำหรับ tenant ส่วนใหญ่ เพราะ database-per-tenant สำหรับลูกค้าทุกรายจะทำให้ต้นทุนต่อลูกค้ารายเล็กสูงกว่ารายได้ที่ลูกค้ารายนั้นจ่าย (unit economics ไม่คุ้มค่า) และสงวนต้นทุนที่สูงกว่าไว้เฉพาะ tenant ที่จ่ายเงินคุ้มค่าจริง ๆ

ข้อสรุปคือ วิศวกรระดับ world-class ไม่ได้ถามแค่ "เทคนิคนี้ทำให้ระบบดีขึ้นไหม" แต่ถามว่า **"ความดีขึ้นนั้นคุ้มกับต้นทุน (เงิน, เวลาพัฒนา, operational complexity) ที่ต้องจ่ายหรือไม่ เมื่อเทียบกับความเสี่ยงทางธุรกิจที่กำลังป้องกันอยู่"** — นี่คือเหตุผลที่ไม่มีคำตอบสำเร็จรูปในงานสถาปัตยกรรมระบบ มีแต่การวิเคราะห์ trade-off ที่เหมาะสมกับบริบทเฉพาะของแต่ละองค์กรเท่านั้น

---

## สรุปท้ายบท

บทนี้พาเราเดินผ่าน 4 สถาปัตยกรรมต้นแบบที่แม้โจทย์ทางธุรกิจจะต่างกันสุดขั้ว — จาก feed ที่อ่านหนักสุดขีด, ระบบการเงินที่ต้องแม่นยำ 100%, e-commerce ที่ต้องรับ traffic spike รุนแรง, ไปจนถึง SaaS ที่ต้องขยายจากลูกค้าไม่กี่รายไปหลักพันราย — แต่ทั้งหมดล้วนสร้างขึ้นจาก **เทคนิคชุดเดียวกัน** ที่เราเรียนมาตลอดหลักสูตรนี้: indexing, replication, partitioning/sharding, locking/concurrency control, row-level security, connection pooling, และ observability

สิ่งที่แยกวิศวกรระดับ world-class ออกจากวิศวกรทั่วไปไม่ใช่การรู้จักเทคนิคเหล่านี้แยกกัน แต่คือความสามารถในการ **วิเคราะห์ constraint ทางธุรกิจ แล้วเลือกประกอบเทคนิคเหล่านั้นเข้าด้วยกันอย่างมีเหตุผล** โดยเข้าใจ trade-off ของทุกการตัดสินใจอย่างถ่องแท้ — ไม่มีสถาปัตยกรรมใดที่ "ถูกต้องที่สุด" ในสุญญากาศ มีแต่สถาปัตยกรรมที่ "เหมาะสมที่สุดกับ constraint ที่กำหนด" เท่านั้น

จำไว้เสมอว่า case study ในบทนี้เป็นสถาปัตยกรรมต้นแบบเพื่อการเรียนรู้ ไม่ใช่พิมพ์เขียวที่นำไปใช้ได้ตรง ๆ กับทุกระบบ — งานจริงของสถาปนิกระบบคือการวิเคราะห์โจทย์เฉพาะหน้าใหม่ทุกครั้ง โดยใช้กรอบความคิดที่ได้จากบทนี้เป็นจุดตั้งต้น

---

## Checklist: คำถามที่ควรถามตัวเองก่อนออกแบบระบบ Enterprise-Scale

ก่อนลงมือทำแบบฝึกหัด ลองใช้ checklist นี้เป็นกรอบตรวจสอบทุกครั้งที่ต้องออกแบบหรือประเมินสถาปัตยกรรมระบบขนาดใหญ่ ไม่ว่าจะเป็นระบบประเภทใดก็ตาม:

1. **Workload shape:** อัตราส่วน read:write เป็นเท่าไหร่? มี hot key/hot row ที่ถูก access ถี่ผิดปกติหรือไม่?
2. **Consistency requirement:** ส่วนไหนของระบบต้องการ strong consistency จริง ๆ และส่วนไหนยอมรับ eventual consistency ได้ (แยกให้ชัดเจน อย่าตัดสินทั้งระบบด้วยมาตรฐานเดียว)
3. **Failure mode:** ถ้า network partition เกิดขึ้นตอนนี้ ระบบจะทำอะไร? ข้อมูลจะหายได้หรือไม่? RPO/RTO ที่ธุรกิจยอมรับได้คือเท่าไหร่?
4. **Growth trajectory:** สถาปัตยกรรมปัจจุบันจะพังที่ scale เท่าไหร่? มีสัญญาณเตือนล่วงหน้า (leading indicator) อะไรที่บอกว่าใกล้ถึงจุดนั้นแล้ว?
5. **Blast radius:** ถ้าส่วนหนึ่งของระบบล้มเหลว จะกระทบส่วนอื่นแค่ไหน? มีกลไก isolate ความเสียหายหรือไม่ (bulkhead pattern)?
6. **Compliance/legal constraint:** มีข้อกำหนดทางกฎหมายที่บังคับการออกแบบหรือไม่ (data residency, retention, audit trail)?
7. **Cost per unit:** ต้นทุนต่อ user/transaction/tenant เป็นเท่าไหร่ และ scale ตามสัดส่วนกับรายได้หรือไม่?
8. **Observability:** ถ้าระบบมีปัญหาตอนตี 3 ทีม on-call จะรู้ได้อย่างไรก่อนที่ลูกค้าจะร้องเรียน? มี metric ที่ตรงกับความเสี่ยงเฉพาะของระบบนี้หรือไม่?
9. **Degradation path:** ถ้าโหลดเกินขีดจำกัดจริง ๆ อะไรคือ critical path ที่ต้องรักษาไว้เสมอ และอะไรที่ยอมเสียสละได้ก่อน?
10. **Reversibility:** การตัดสินใจนี้ย้อนกลับได้ง่ายแค่ไหนถ้าผิดพลาด? (เช่น เลือก sharding key ผิดมักย้อนกลับยากกว่าเลือก cache TTL ผิด)

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
<summary>เฉลย</summary>

นี่คือสถานการณ์คลาสสิกที่นำไปสู่กลยุทธ์ hybrid ตามที่อธิบายในหัวข้อ 989.3(4) ทีมควร "ย้าย" (migrate) เฉพาะ tenant enterprise รายนี้ออกจาก shared RLS database ไปยัง **database-per-tenant แยกต่างหากที่ตั้งอยู่ใน region สหภาพยุโรป (EU)** โดยมีขั้นตอนคร่าว ๆ ดังนี้:

1. **Provision database instance ใหม่ใน EU region** ผ่าน control plane / Infrastructure as Code ที่มี schema เดียวกันกับ shared database (ใช้ migration script ชุดเดียวกัน เพื่อรับประกันความสอดคล้องของ schema)
2. **Export ข้อมูลของ tenant นั้นออกจาก shared database** โดยกรองด้วย `tenant_id` (ใช้ `pg_dump` พร้อม `--where` หรือ custom export script) แล้ว import เข้า instance ใหม่
3. **อัปเดต routing/control plane layer** ให้รู้ว่า tenant นี้ต้อง route ไปยัง connection string ของ database ใหม่แทนที่จะผ่าน RLS-shared pool (คล้ายกับ routing layer ที่ใช้ใน sharding ของ Case Study 1)
4. **ลบข้อมูลของ tenant นั้นออกจาก shared database** หลังยืนยันว่าการย้ายสำเร็จและ verify ความถูกต้องของข้อมูลแล้ว (reconciliation check)
5. **ทดสอบ cutover แบบมี rollback plan** เช่น เปิด dual-write ชั่วคราว หรือมี maintenance window สั้น ๆ เพื่อป้องกัน data loss ระหว่างการย้าย

จุดสำคัญคือ tenant รายอื่นที่ยังอยู่บน shared RLS model **จะไม่ได้รับผลกระทบใด ๆ** เพราะ RLS policy, schema, และ application code ส่วนกลางไม่ต้องเปลี่ยนแปลง — สิ่งที่เปลี่ยนคือ routing layer ที่รู้จักเพิ่ม "ปลายทาง" ใหม่สำหรับ tenant พิเศษรายนี้เท่านั้น นี่คือเหตุผลที่การออกแบบ **control plane ที่รองรับ hybrid tenancy ตั้งแต่ต้น** (แม้ในช่วงแรกจะยังไม่มี tenant ที่ต้อง dedicate จริง) เป็นการลงทุนที่คุ้มค่าระยะยาว เพราะช่วยให้การขยายสถาปัตยกรรมในอนาคตไม่ต้อง refactor ระบบทั้งหมด
</details>

### แบบฝึกหัดที่ 5

เปรียบเทียบเหตุผลที่ FeedFlow (Case Study 1) เลือกใช้ asynchronous replication เป็นหลัก ในขณะที่ PayCore (Case Study 2) เลือกใช้ synchronous replication จงอธิบายว่าถ้าสลับกลยุทธ์ replication ของทั้งสองระบบ (เอา FeedFlow ไปใช้ synchronous, เอา PayCore ไปใช้ asynchronous) จะเกิดผลเสียอะไรกับแต่ละระบบ

<details>
<summary>เฉลย</summary>

**ถ้า FeedFlow ใช้ synchronous replication:** ทุก write (โพสต์, like, comment) ที่มีปริมาณสูงถึง 50,000-80,000 ครั้งต่อวินาทีในช่วง peak จะต้องรอให้ standby (อาจอยู่คนละภูมิภาค) ยืนยันรับ WAL ก่อนตอบ client ว่าสำเร็จ ด้วยจำนวน write ที่สูงมากและ standby ที่กระจายตามภูมิภาคเพื่อ availability ผลคือ **latency ของทุก write เพิ่มขึ้นอย่างมีนัยสำคัญ** (อาจเพิ่มจากไม่กี่ ms เป็นหลักสิบหรือหลักร้อย ms ถ้า standby อยู่ไกล) และที่แย่กว่านั้นคือ **throughput สูงสุดของระบบทั้งหมดจะถูกจำกัดด้วยความเร็วของ standby ที่ช้าที่สุด** ซึ่งขัดกับเป้าหมายหลักของ FeedFlow ที่ต้องการ scale การเขียนให้สูงที่สุดเท่าที่ทำได้ ในขณะที่ธุรกิจไม่ได้ต้องการ RPO=0 ขนาดนั้น (สูญเสีย like หนึ่งครั้งไม่ใช่หายนะ) จึงเป็นการลงทุนที่ไม่คุ้มค่าอย่างชัดเจน

**ถ้า PayCore ใช้ asynchronous replication:** ระบบจะได้ latency ต่ำลงและ throughput สูงขึ้น แต่จะสูญเสีย **การรับประกัน RPO=0** ทันที — หาก primary ล่มก่อนที่ WAL จะถูกส่งไปถึง standby (สถานการณ์ที่เกิดขึ้นได้จริงเสมอ ไม่ว่าความน่าจะเป็นจะต่ำแค่ไหน) transaction ทางการเงินที่เพิ่ง COMMIT สำเร็จจะหายไปพร้อมกับ primary หมายถึงเงินของลูกค้าหายไปโดยไม่มีทางกู้คืน ซึ่งเป็นความเสี่ยงที่ยอมรับไม่ได้ในธุรกิจการเงิน ทั้งในแง่ความเชื่อถือของลูกค้าและในแง่กฎหมาย/compliance ที่มักกำหนดให้ระบบการเงินต้องพิสูจน์ได้ว่าไม่มี transaction ใดสูญหาย

บทเรียนคือ **การเลือก replication mode ไม่ใช่การเลือก "แบบที่ดีที่สุด" แบบสัมบูรณ์ แต่ต้องเลือกให้สอดคล้องกับ RPO/RTO ที่ธุรกิจกำหนด** ซึ่งต่างกันไปตามความเสี่ยงที่แต่ละระบบยอมรับได้
</details>

### แบบฝึกหัดที่ 6

ในช่วง Black Friday ทีมงาน ShopScale สังเกตว่า `pg_stat_activity` แสดง transaction จำนวนมากอยู่ในสถานะ `active` แต่ค้างนานผิดปกติ (waiting) โดยเฉพาะที่พยายาม UPDATE แถวสินค้า flash-sale เดียวกัน จงอธิบายว่าเกิดอะไรขึ้นในเชิงเทคนิค และเสนอวิธีแก้ 2 แนวทางที่ไม่ต้องเปลี่ยน schema

<details>
<summary>เฉลย</summary>

สิ่งที่เกิดขึ้นคือ **row-level lock contention บน hot row** — เมื่อ transaction จำนวนมากพยายาม `UPDATE` แถวสินค้าเดียวกันพร้อมกัน PostgreSQL จะอนุญาตให้เพียง transaction เดียวถือ lock บนแถวนั้นได้ในแต่ละขณะ (ตาม MVCC/row-level locking ตามที่เรียนใน Part 058) transaction อื่นที่พยายาม UPDATE แถวเดียวกันจะต้อง **รอ (block)** จนกว่า transaction ก่อนหน้าจะ commit หรือ rollback เมื่อมีคนแย่งซื้อพร้อมกันหลักหมื่นคน จำนวน transaction ที่ต่อคิวรอจะสะสมเป็นแถวยาว (queueing) ทำให้ transaction ที่มาทีหลังต้องรอนานขึ้นเรื่อย ๆ (คล้ายปรากฏการณ์ traffic jam) และหาก connection pool มีจำนวนจำกัด transaction ที่ค้างรอเหล่านี้จะกิน connection slot ไปเรื่อย ๆ จนอาจทำให้ request อื่นที่ไม่เกี่ยวข้องกับสินค้าตัวนี้ได้รับผลกระทบไปด้วย (connection starvation)

วิธีแก้ที่ไม่ต้องเปลี่ยน schema:

1. **เพิ่ม pre-check layer ด้วย Redis atomic counter ก่อนแตะ PostgreSQL** — ใช้ `DECR` แบบ atomic ใน Redis เพื่อกรอง request ที่ "ไม่มีทางสำเร็จ" ออกไปตั้งแต่ต้นทาง (เช่นถ้า counter ติดลบแล้วให้ reject ทันทีโดยไม่ยิง query เข้า DB เลย) ลดจำนวน transaction ที่มาแย่ง row lock ที่ PostgreSQL ลงอย่างมาก แม้จะไม่ได้แก้ที่ต้นเหตุ (ยังมี hot row เดิม) แต่ลดปริมาณที่มาถึง DB ได้มหาศาล
2. **ตั้ง statement_timeout หรือ lock_timeout ที่เหมาะสมสำหรับ transaction เหล่านี้** เพื่อไม่ให้ transaction ค้างรอ lock นานเกินไปจนกิน connection slot — ถ้า transaction รอ lock เกินเวลาที่กำหนด ให้ error กลับไปทันที (fail fast) แทนที่จะปล่อยให้ connection ถูกยึดครองอยู่เรื่อย ๆ ทำให้ connection pool ยังมี slot ว่างสำหรับ request อื่น ๆ ที่ไม่เกี่ยวข้องกับสินค้าตัวนี้

ทั้งสองวิธีนี้ไม่ต้องแก้ schema ของตาราง inventory แต่จัดการที่ระดับ application/infrastructure layer ซึ่งเป็นแนวทางที่ปลอดภัยกว่าในสถานการณ์ที่กำลังเกิด incident จริง (deploy schema change ระหว่าง peak traffic มีความเสี่ยงสูงกว่าการปรับ config)
</details>

### แบบฝึกหัดที่ 7

TenantHub มีลูกค้ารายหนึ่งร้องขอ (ตามสิทธิ์ GDPR "right to be forgotten") ให้ลบข้อมูลทั้งหมดของบริษัทตนออกจากระบบภายใน 30 วัน จงเปรียบเทียบว่าคำขอนี้ทำได้ยากง่ายต่างกันอย่างไร ถ้าระบบใช้ (ก) RLS shared schema (ข) schema-per-tenant (ค) database-per-tenant

<details>
<summary>เฉลย</summary>

**(ก) RLS shared schema:** ยากที่สุด เพราะข้อมูลของ tenant นี้กระจายปะปนอยู่กับ tenant อื่นในทุกตารางของ shared schema การลบต้องเขียน script ที่ `DELETE FROM <table> WHERE tenant_id = X` วนทุกตารางในระบบอย่างระมัดระวัง (ต้องครบทุกตารางจริง ๆ ไม่ตกหล่นแม้แต่ตารางเดียว มิฉะนั้นถือว่าไม่ compliant) ต้องคำนึงถึง foreign key constraint และลำดับการลบที่ถูกต้อง อีกทั้งข้อมูลใน backup/WAL archive เก่ายังอาจมีร่องรอยของ tenant นี้หลงเหลืออยู่ (ต้องมีนโยบาย backup retention/purge แยกต่างหากเพื่อจัดการส่วนนี้)

**(ข) Schema-per-tenant:** ง่ายกว่ามาก เพราะสามารถใช้คำสั่งเดียว `DROP SCHEMA tenant_x CASCADE` เพื่อลบข้อมูลทั้งหมดของ tenant นั้นได้ในทีเดียว โดยไม่กระทบ schema ของ tenant อื่นเลย ยังคงต้องจัดการเรื่อง backup/archive แยกต่างหากเช่นเดิม แต่ส่วนของ live database ทำได้สะดวกและรวดเร็ว

**(ค) Database-per-tenant:** ง่ายที่สุด เพียงแค่ `DROP DATABASE` (หรือ terminate instance ทั้งหมด) ก็ลบข้อมูลของ tenant นั้นได้ครบถ้วนในคำสั่งเดียว รวมถึง backup ที่แยกต่างหากต่อ instance ก็จัดการ/purge ได้ตรงไปตรงมากว่า เพราะไม่ปะปนกับ tenant อื่นตั้งแต่ต้น

ข้อสรุปคือ **ระดับของ data isolation ทางสถาปัตยกรรมส่งผลโดยตรงต่อความง่ายในการปฏิบัติตามข้อกำหนดทางกฎหมายเกี่ยวกับข้อมูลส่วนบุคคล** ซึ่งเป็นอีกหนึ่งปัจจัยที่ควรนำมาชั่งน้ำหนักตั้งแต่ตอนเลือกกลยุทธ์ multi-tenancy ไม่ใช่พิจารณาแค่เรื่องต้นทุนและ performance เท่านั้น
</details>

### แบบฝึกหัดที่ 8

จงยกตัวอย่างการนำหลักการ "Defense in Depth" (จาก Step 990.1) มาประยุกต์ใช้กับระบบใดระบบหนึ่งที่ไม่ใช่ 4 case study ในบทนี้ (เช่น ระบบ booking ตั๋ว, ระบบ inventory คลังสินค้าทั่วไป, หรือระบบอื่นที่คุณคุ้นเคย) โดยระบุอย่างน้อย 3 ชั้นการป้องกัน

<details>
<summary>เฉลย</summary>

ตัวอย่างคำตอบ (ระบบจองตั๋วภาพยนตร์/ที่นั่ง ซึ่งมีปัญหาคล้าย flash-sale ของ ShopScale):

1. **ชั้นที่ 1 — Application/UX layer:** แสดงที่นั่งที่ "กำลังถูกคนอื่นจอง" แบบ real-time (soft lock ชั่วคราวด้วย TTL สั้น ๆ ใน Redis) เพื่อลดโอกาสที่คนสองคนจะพยายามจองที่นั่งเดียวกันตั้งแต่ต้น แม้จะไม่ใช่การป้องกันที่รับประกัน 100% (เป็นแค่ UX hint) แต่ช่วยลด traffic ที่จะไปถึงชั้นถัดไป
2. **ชั้นที่ 2 — Database constraint layer:** ใช้ `UNIQUE` constraint บนคู่ `(showtime_id, seat_number)` ในตาราง bookings ร่วมกับ atomic `INSERT ... ON CONFLICT DO NOTHING` — ถ้ามีสอง transaction พยายาม insert การจองที่นั่งเดียวกันพร้อมกัน PostgreSQL จะรับประกันด้วย unique index ว่ามีเพียง insert เดียวเท่านั้นที่สำเร็จ (เป็นชั้นการป้องกันที่รับประกันความถูกต้อง 100% ไม่ว่า application layer ด้านบนจะมี bug หรือไม่)
3. **ชั้นที่ 3 — Transaction/timeout layer:** กำหนด `lock_timeout` และใช้ transaction ที่ commit เร็ว (short transaction) เพื่อไม่ให้การจองที่นั่งหนึ่งค้างนานจนกระทบระบบโดยรวม พร้อมมี background job ที่ยกเลิกการจองที่ "ค้าง" เกินเวลาที่กำหนด (เช่น จองไว้แต่ยังไม่จ่ายเงินภายใน 10 นาที) เพื่อปลดปล่อยที่นั่งกลับคืนสู่ inventory

แต่ละชั้นทำหน้าที่ต่างกัน — ชั้นแรกลดโอกาสเกิดปัญหา (prevention), ชั้นที่สองรับประกันความถูกต้องแม้ชั้นแรกล้มเหลว (guarantee), ชั้นที่สามจัดการผลข้างเคียงระยะยาว (cleanup/recovery) นี่คือแก่นของ defense in depth — แต่ละชั้นไม่ต้องสมบูรณ์แบบ แต่รวมกันแล้วปิดช่องโหว่ซึ่งกันและกัน
</details>

### แบบฝึกหัดที่ 9

หากคุณเป็น Staff Engineer และถูกถามในการสัมภาษณ์งานว่า "ออกแบบระบบ inventory ของ e-commerce ที่รองรับ Black Friday ให้หน่อย" จงร่างคำตอบสั้น ๆ (5-8 ประโยค) ที่แสดงกระบวนการคิดแบบมีลำดับขั้น (ไม่ใช่แค่ท่องเทคนิค) โดยอ้างอิงแนวทางจาก Case Study 3

<details>
<summary>เฉลย</summary>

ตัวอย่างแนวคำตอบ: "ก่อนอื่นผมจะถามกลับก่อนว่า traffic pattern เป็นแบบ spike ฉับพลันหรือค่อย ๆ เพิ่ม และสินค้าที่ concern คือสินค้าทั่วไปหรือสินค้า flash-sale จำนวนจำกัดที่มีคนแย่งกันซื้อพร้อมกันมาก เพราะคำตอบจะต่างกัน สำหรับสินค้าทั่วไป ผมจะแยก read path (แสดงข้อมูลสินค้า, browse) ออกจาก write path (checkout) โดย read ใช้ cache+replica ได้เต็มที่เพราะยอมรับความคลาดเคลื่อนเล็กน้อยได้ ส่วน write ต้องยิงไปที่ primary เท่านั้น สำหรับสินค้า flash-sale ที่เป็น hot row ผมจะใช้ atomic conditional UPDATE (`UPDATE ... WHERE stock > 0 RETURNING stock`) เป็นกลไกป้องกัน oversell ระดับ database ซึ่งเป็นด่านสุดท้ายที่รับประกันความถูกต้องเสมอ แล้วเสริมด้วย pre-check layer ที่ Redis เพื่อกรอง request ที่ไม่มีทางสำเร็จออกก่อนถึง DB ลด lock contention ผมจะเสริมด้วย traffic shaping ที่ edge เช่น virtual waiting room เพื่อไม่ให้ spike ทั้งหมดถล่มระบบพร้อมกัน และสุดท้ายผมจะออกแบบ graceful degradation ไว้ล่วงหน้า เช่นปิดฟีเจอร์ recommendation ที่ไม่จำเป็นเมื่อโหลดสูงเกินขีด เพื่อรักษา checkout path ซึ่งเป็นหัวใจของธุรกิจให้ทำงานได้เสมอ"

คำตอบแบบนี้แสดงให้เห็นว่าผู้ตอบคิดเป็นลำดับ (clarify requirement → แยกปัญหาตาม pattern การใช้งาน → เลือกกลไกป้องกันที่ database layer ก่อนเป็นอันดับแรกเพราะเชื่อถือได้สุด → เสริมชั้นป้องกันอื่นเพื่อลดโหลด → เตรียมแผนสำรองเมื่อระบบเกินขีดจำกัด) ซึ่งเป็นสิ่งที่ผู้สัมภาษณ์ระดับ Staff/Principal มองหา มากกว่าการท่องชื่อเทคนิคโดด ๆ
</details>

### แบบฝึกหัดที่ 10

พิจารณาสถานการณ์สมมติ: TenantHub ตัดสินใจใช้ RLS shared schema กับ tenant ทั้งหมด 8,000 ราย และเริ่มพบว่า query บางตัวช้าลงเรื่อย ๆ เมื่อ dataset โตขึ้น จงวิเคราะห์ว่าอะไรคือสาเหตุที่เป็นไปได้มากที่สุด (2 ข้อ) และแต่ละข้อแก้ไขอย่างไรโดยยังคงใช้ RLS shared schema ต่อไป (ไม่ต้องย้ายไป schema/database-per-tenant)

<details>
<summary>เฉลย</summary>

**สาเหตุที่เป็นไปได้ข้อที่ 1 — Index ไม่มี `tenant_id` เป็น leading column:** ถ้า index ถูกสร้างโดยไม่ได้คำนึงถึง RLS filter เช่น สร้าง index บน `(created_at)` เฉยๆ แทนที่จะเป็น `(tenant_id, created_at)` query planner แม้จะได้ผลลัพธ์ถูกต้องจาก RLS policy ที่กรอง `tenant_id` ให้ แต่จะต้อง scan ข้อมูลของหลายพัน tenant ปนกันก่อนกรองออก (index ไม่ selective พอสำหรับ query ที่ scope อยู่แค่ tenant เดียว) วิธีแก้คือ **สร้าง composite index ที่มี `tenant_id` เป็นคอลัมน์แรกเสมอ** สำหรับทุกตารางและทุก query pattern ที่ใช้บ่อย เพื่อให้ query planner สามารถ narrow ลงมาที่ข้อมูลของ tenant เดียวได้อย่างมีประสิทธิภาพตั้งแต่ต้น

**สาเหตุที่เป็นไปได้ข้อที่ 2 — Data skew ระหว่าง tenant (บาง tenant มีข้อมูลมากกว่า tenant อื่นมหาศาล):** ถ้ามี tenant รายใหญ่บางรายที่มีข้อมูลมากกว่า tenant เล็กหลายร้อยเท่า (เช่น tenant องค์กรใหญ่ที่มีพนักงานหลายพันคนเทียบกับ tenant startup 5 คน) table statistics ที่ query planner ใช้ตัดสินใจ (ผ่าน `ANALYZE`) อาจไม่สะท้อนความจริงสำหรับ tenant ที่ query อยู่ ณ ขณะนั้น ทำให้ planner เลือก execution plan ที่ไม่เหมาะสม (เช่น เลือก seq scan ทั้งที่ควร index scan หรือกลับกัน) วิธีแก้คือการปรับ `default_statistics_target` ให้สูงขึ้นสำหรับคอลัมน์ที่มี skew สูง เพื่อให้ planner เก็บสถิติละเอียดขึ้น และพิจารณาใช้ **partial index หรือ partitioning ภายในตาราง** (เช่น partition ตามช่วงของ tenant_id หรือตามขนาด tenant) เพื่อแยก tenant ใหญ่ออกจาก tenant เล็กในระดับ physical storage แม้จะยังอยู่ใน logical shared schema เดียวกันก็ตาม รวมถึงพิจารณาย้าย tenant ใหญ่ที่สุดไม่กี่รายไปยัง dedicated resource ตามแนวทาง hybrid ที่กล่าวถึงในหัวข้อ 989.3(4) หากปัญหายังไม่คลี่คลายด้วยวิธีข้างต้น
</details>

---

## แหล่งอ้างอิงแนวคิด (สำหรับผู้ต้องการศึกษาเพิ่มเติมด้วยตนเอง)

Case study ในบทนี้สังเคราะห์ขึ้นจากหลักการทางวิศวกรรมที่เป็นสาธารณะและเป็นที่ยอมรับกว้างขวางในอุตสาหกรรม เช่น แนวคิด fan-out pattern ในระบบ feed, double-entry bookkeeping ในระบบบัญชี, optimistic/pessimistic concurrency control ในระบบ inventory, และรูปแบบ multi-tenancy ที่ใช้กันทั่วไปในงาน SaaS ผู้เรียนที่ต้องการศึกษาเชิงลึกเพิ่มเติมควรค้นหาคำสำคัญเหล่านี้ประกอบกับคำว่า "engineering blog" หรือ "system design" เพื่อดูกรณีศึกษาจากบริษัทเทคโนโลยีจริงที่เผยแพร่ต่อสาธารณะ และนำมาเทียบเคียงกับกรอบความคิดที่ได้จากบทนี้

---

## บทถัดไป

บทนี้เป็นการสังเคราะห์เทคนิคทั้งหมดของหลักสูตรผ่านมุมมองสถาปัตยกรรมระดับ enterprise บทถัดไปจะพาไปสู่การเตรียมตัวสำหรับการรับรองความเชี่ยวชาญและการสัมภาษณ์งานในระดับที่สูงขึ้น อ่านต่อได้ที่ [Part 101: Certification & Interview Preparation](./part-101-certification-interview-prep.md)