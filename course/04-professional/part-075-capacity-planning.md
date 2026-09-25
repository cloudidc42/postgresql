# Capacity Planning และ Horizontal/Vertical Scaling

หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับมืออาชีพ | Part 075

---

## เป้าหมายการเรียนรู้

เมื่อเรียนจบบทนี้ ผู้เรียนจะสามารถ:

1. อธิบายได้ว่า Capacity Planning คืออะไร และเหตุใดจึงต้องทำล่วงหน้าก่อนระบบมีปัญหา ไม่ใช่ทำหลังระบบล่มไปแล้ว
2. วัดและติดตาม Growth Rate ของฐานข้อมูล ทั้งขนาดข้อมูล จำนวน transaction และ connection ด้วย SQL query ที่รันได้จริง
3. เข้าใจข้อดี-ข้อจำกัดของ Vertical Scaling (Scale Up) และรู้ว่าเมื่อไหร่จะ "ชนเพดาน" ของฮาร์ดแวร์
4. เข้าใจแนวคิด Horizontal Scaling (Scale Out) และความท้าทายเฉพาะตัวของฐานข้อมูลเชิงสัมพันธ์ (RDBMS) แบบดั้งเดิมเมื่อพยายามกระจายโหลด
5. ออกแบบกลยุทธ์ Read Scaling ด้วย Read Replica และรู้ข้อจำกัดว่าอะไรขยายได้ อะไรขยายไม่ได้
6. เข้าใจว่าทำไม Write Scaling ถึงยากกว่า Read Scaling มาก และรู้จักแนวคิดเบื้องต้นของ Sharding
7. วางแผน Storage Capacity ล่วงหน้า พร้อมกลยุทธ์ archive ข้อมูลเก่าโดยใช้ partitioning
8. คำนวณ Connection Capacity ที่ระบบต้องรองรับ และออกแบบ connection pooling ให้เหมาะสม
9. ทำ Load Testing ก่อน scale จริง เพื่อค้นหาคอขวด (bottleneck) ก่อนที่จะเจอปัญหาในสถานการณ์จริง
10. จัดทำแผน Capacity Planning ระยะ 12 เดือนสำหรับระบบที่คาดว่าจะเติบโตอย่างก้าวกระโดด

---

## บทนำ: ทำไม Capacity Planning ถึงสำคัญ

ลองจินตนาการสถานการณ์นี้: ทีมของคุณ deploy ระบบ e-commerce ขึ้น production สำเร็จ ทุกอย่างทำงานได้ดี response time เฉลี่ย 50ms CPU ใช้งานแค่ 20% ทุกคนพอใจ แต่ผ่านไป 8 เดือน ยอดขายเติบโต 5 เท่า และวันหนึ่งในช่วง campaign ลดราคาใหญ่ ระบบก็ล่ม — connection เต็ม, query ช้าลงเป็นวินาที, CPU พุ่งไป 100% ตลอดเวลา

นี่คือสิ่งที่ Capacity Planning มีไว้ป้องกัน มันคือกระบวนการ **คาดการณ์ทรัพยากรที่ระบบจะต้องการในอนาคต** โดยอิงจากข้อมูลการเติบโตในอดีตและแผนธุรกิจ แล้ววางแผนล่วงหน้าว่าจะขยายระบบอย่างไร เมื่อไหร่ และด้วยวิธีใด แทนที่จะรอให้ระบบมีปัญหาก่อนแล้วค่อยแก้แบบเร่งด่วน (firefighting)

บทนี้เชื่อมโยงความรู้จากหลายบทก่อนหน้า:
- Streaming Replication (Part 063) และ Load Balancing (Part 067) — ใช้เป็นกลไก Read Scaling
- Connection Pooling (Part 066) — ใช้บริหารจัดการ Connection Capacity
- Partitioning (Part 054) — ใช้เป็นกลยุทธ์ Storage Management และ Archive
- Citus (จะเจาะลึกใน Part 097) — ใช้เป็นแนวทาง Write Scaling / Sharding ขั้นสูง
- pgbench (จะเจาะลึกใน Part 098) — เครื่องมือหลักสำหรับ Load Testing

มาเริ่มต้นกันทีละ Step

---

## Step 741: Capacity Planning คืออะไร

### นิยาม

**Capacity Planning** คือกระบวนการเชิงระบบในการกำหนดว่าโครงสร้างพื้นฐาน (infrastructure) ที่มีอยู่ในปัจจุบันจะรองรับความต้องการ (demand) ในอนาคตได้หรือไม่ และถ้าไม่ได้ ต้องเพิ่มทรัพยากรอะไร เท่าไหร่ และเมื่อไหร่

หัวใจสำคัญคือคำว่า **ล่วงหน้า (proactive)** ไม่ใช่ **ตอบสนอง (reactive)** — Capacity Planning ที่ดีจะทำให้ทีมงานรู้ล่วงหน้าเป็นเดือนๆ ว่า "ในอีก 3 เดือนข้างหน้า ถ้าอัตราการเติบโตยังเป็นแบบนี้ storage จะเต็ม" หรือ "ในอีก 2 เดือน connection pool จะไม่พอ" แทนที่จะมารู้ตัวตอนที่ alert แจ้งเตือนว่า disk เหลือ 5%

### องค์ประกอบหลักของ Capacity Planning

| องค์ประกอบ | คำถามที่ต้องตอบ | Metric ที่เกี่ยวข้อง |
|---|---|---|
| **Compute (CPU)** | CPU จะพอไหมเมื่อ load เพิ่มขึ้น? | CPU utilization, load average, active query count |
| **Memory (RAM)** | shared_buffers, work_mem จะพอไหม? | cache hit ratio, swap usage, OS page cache |
| **Storage** | disk จะเต็มเมื่อไหร่? | database size, table growth rate, WAL volume |
| **I/O Throughput** | disk I/O จะเป็นคอขวดไหม? | IOPS, read/write latency, disk queue depth |
| **Network** | bandwidth ระหว่าง app-db พอไหม? | network throughput, replication lag |
| **Connections** | concurrent connections จะเกิน max_connections ไหม? | active/idle connections, pool saturation |
| **Concurrency** | ระบบรองรับ concurrent transaction ได้กี่ตัว? | lock contention, deadlock rate |

### วงจรของ Capacity Planning (Capacity Planning Lifecycle)

Capacity Planning ไม่ใช่กิจกรรมที่ทำครั้งเดียวจบ แต่เป็นวงจรต่อเนื่อง:

```
1. Monitor (เก็บข้อมูล metric ปัจจุบัน)
        ↓
2. Analyze (วิเคราะห์ trend และอัตราการเติบโต)
        ↓
3. Forecast (คาดการณ์ความต้องการในอนาคต)
        ↓
4. Plan (วางแผนว่าจะขยายอะไร เมื่อไหร่)
        ↓
5. Implement (ดำเนินการขยายระบบ)
        ↓
6. Validate (ตรวจสอบว่าที่ขยายไปเพียงพอจริง)
        ↓
   กลับไปข้อ 1 (วนซ้ำต่อเนื่อง)
```

### ทำไมองค์กรจำนวนมากถึงข้าม Capacity Planning ไป

ในทางปฏิบัติ หลายทีมมองข้าม Capacity Planning เพราะ:

1. **มองว่าเป็นงาน "ไม่เร่งด่วน"** — เมื่อเทียบกับการแก้บั๊กหรือทำ feature ใหม่ Capacity Planning ดูเหมือนไม่มี deadline ชัดเจน จนกระทั่งวันที่ระบบล่มจริงๆ
2. **ขาดข้อมูลในอดีต (historical data)** — ถ้าไม่เคยเก็บ metric ไว้ ก็ไม่มีฐานในการคาดการณ์
3. **เข้าใจผิดว่า cloud แก้ปัญหาให้อัตโนมัติ** — แม้ cloud provider จะ scale ฮาร์ดแวร์ให้ง่ายขึ้น (เช่นเปลี่ยน instance type) แต่ PostgreSQL เองก็ยังต้องมีการวางแผนเรื่อง configuration, downtime ระหว่าง scale, และข้อจำกัดทางสถาปัตยกรรมที่ cloud แก้ให้ไม่ได้

### ตัวอย่างสถานการณ์ที่ Capacity Planning ป้องกันได้

- **Black Friday / 11.11 / Flash Sale**: หากไม่วางแผนล่วงหน้า traffic ที่เพิ่มขึ้น 20-50 เท่าในไม่กี่ชั่วโมงจะทำให้ระบบล่มทันที
- **การเติบโตของบริษัทแบบค่อยเป็นค่อยไป**: user เพิ่มขึ้น 10% ต่อเดือน หลังจาก 12 เดือนจะกลายเป็น 3.1 เท่า ถ้าไม่ได้เผื่อไว้ ระบบจะทำงานได้ช้าลงเรื่อยๆ จนถึงจุดวิกฤต
- **Data Retention ที่ไม่มีการล้าง**: log, audit trail, transaction history ที่เก็บไม่มีวันหมดอายุ จะทำให้ storage โตแบบไม่มีขีดจำกัด

### SQL เบื้องต้น: ดูภาพรวมทรัพยากรปัจจุบัน

ก่อนจะวางแผนอนาคต ต้องรู้จุดเริ่มต้นก่อน มาดู query พื้นฐานที่ใช้สำรวจสถานะปัจจุบันของระบบ

```sql
-- ขนาดฐานข้อมูลทั้งหมดในเครื่อง PostgreSQL server นี้
SELECT
    datname                                AS database_name,
    pg_size_pretty(pg_database_size(datname)) AS size_pretty,
    pg_database_size(datname)              AS size_bytes
FROM pg_database
WHERE datname NOT IN ('template0', 'template1')
ORDER BY pg_database_size(datname) DESC;
```

```sql
-- จำนวน connection ปัจจุบัน เทียบกับ max_connections
SELECT
    (SELECT count(*) FROM pg_stat_activity)         AS current_connections,
    (SELECT setting::int FROM pg_settings
        WHERE name = 'max_connections')             AS max_connections,
    round(
        (SELECT count(*)::numeric FROM pg_stat_activity)
        / (SELECT setting::numeric FROM pg_settings WHERE name = 'max_connections')
        * 100, 2
    ) AS pct_used;
```

```sql
-- สรุป transaction ต่อวินาทีโดยประมาณ (จาก pg_stat_database)
SELECT
    datname,
    xact_commit + xact_rollback           AS total_transactions,
    xact_commit,
    xact_rollback,
    round(
        xact_rollback::numeric
        / nullif(xact_commit + xact_rollback, 0) * 100, 2
    ) AS rollback_pct
FROM pg_stat_database
WHERE datname NOT IN ('template0', 'template1')
ORDER BY total_transactions DESC;
```

หมายเหตุ: ค่า `xact_commit`/`xact_rollback` เป็นค่าสะสม (cumulative counter) นับตั้งแต่ database เริ่มทำงานครั้งล่าสุด ไม่ใช่ค่าต่อวินาที การจะได้อัตราจริง ต้องเก็บค่านี้เป็น time-series แล้วคำนวณผลต่างระหว่างช่วงเวลา ซึ่งเราจะพูดถึงใน Step 742

---

## Step 742: การวัด Growth Rate

### แนวคิดหลัก

Capacity Planning ที่แม่นยำต้องอาศัย **ข้อมูลย้อนหลัง (historical trend)** ไม่ใช่การเดา สิ่งที่ต้องติดตามอย่างสม่ำเสมอ (เช่น เก็บทุกวันหรือทุกชั่วโมง) มีอย่างน้อย 3 กลุ่ม:

1. **ขนาดฐานข้อมูล (Database Size Growth)** — โตขึ้นกี่ GB ต่อเดือน
2. **จำนวน Transaction ต่อวัน (Transaction Volume Growth)** — commit/rollback ต่อวันเพิ่มขึ้นแค่ไหน
3. **จำนวน Connection และ Active Session** — เพิ่มขึ้นตามจำนวน user/microservice หรือไม่

### การสร้างตารางเก็บ Historical Metrics

เนื่องจาก PostgreSQL เก็บ statistics แบบสะสม (cumulative) และ reset เมื่อ restart หรือรัน `pg_stat_reset()` เราจึงควรสร้างตารางเก็บ snapshot รายวันไว้เอง

```sql
-- ตารางสำหรับเก็บ snapshot ของ metric รายวัน
CREATE TABLE IF NOT EXISTS capacity_metrics_history (
    snapshot_date       date PRIMARY KEY,
    database_size_bytes bigint,
    total_connections   integer,
    max_connections     integer,
    xact_commit         bigint,
    xact_rollback       bigint,
    tup_inserted        bigint,
    tup_updated         bigint,
    tup_deleted         bigint,
    blks_hit            bigint,
    blks_read           bigint,
    captured_at         timestamptz DEFAULT now()
);
```

```sql
-- Query สำหรับ capture snapshot ประจำวัน (เรียกผ่าน cron/pg_cron ทุกวันเที่ยงคืน)
INSERT INTO capacity_metrics_history (
    snapshot_date, database_size_bytes, total_connections,
    max_connections, xact_commit, xact_rollback,
    tup_inserted, tup_updated, tup_deleted,
    blks_hit, blks_read
)
SELECT
    current_date,
    pg_database_size(current_database()),
    (SELECT count(*) FROM pg_stat_activity),
    (SELECT setting::int FROM pg_settings WHERE name = 'max_connections'),
    xact_commit,
    xact_rollback,
    tup_inserted,
    tup_updated,
    tup_deleted,
    blks_hit,
    blks_read
FROM pg_stat_database
WHERE datname = current_database()
ON CONFLICT (snapshot_date) DO UPDATE SET
    database_size_bytes = EXCLUDED.database_size_bytes,
    total_connections   = EXCLUDED.total_connections,
    xact_commit          = EXCLUDED.xact_commit,
    xact_rollback        = EXCLUDED.xact_rollback,
    tup_inserted         = EXCLUDED.tup_inserted,
    tup_updated          = EXCLUDED.tup_updated,
    tup_deleted          = EXCLUDED.tup_deleted,
    blks_hit             = EXCLUDED.blks_hit,
    blks_read            = EXCLUDED.blks_read,
    captured_at          = now();
```

> เชื่อมโยง: การ schedule query นี้ให้รันอัตโนมัติทุกวัน สามารถทำได้ด้วย `pg_cron` extension หรือ cron job ภายนอกที่ยิงผ่าน `psql`

### การคำนวณ Growth Rate จากข้อมูลย้อนหลัง

เมื่อมีข้อมูลสะสมหลายวันแล้ว เราสามารถคำนวณอัตราการเติบโตต่อวันได้ด้วย window function

```sql
-- คำนวณการเติบโตของขนาดฐานข้อมูลต่อวัน (day-over-day growth)
SELECT
    snapshot_date,
    pg_size_pretty(database_size_bytes)                        AS size_now,
    pg_size_pretty(database_size_bytes - lag(database_size_bytes) OVER w) AS growth_from_yesterday,
    round(
        (database_size_bytes - lag(database_size_bytes) OVER w)::numeric
        / nullif(lag(database_size_bytes) OVER w, 0) * 100, 3
    ) AS growth_pct
FROM capacity_metrics_history
WINDOW w AS (ORDER BY snapshot_date)
ORDER BY snapshot_date;
```

```sql
-- อัตราการเติบโตเฉลี่ยต่อเดือน (compound monthly growth rate โดยประมาณ)
WITH first_last AS (
    SELECT
        min(snapshot_date) AS first_date,
        max(snapshot_date) AS last_date,
        (array_agg(database_size_bytes ORDER BY snapshot_date))[1] AS first_size,
        (array_agg(database_size_bytes ORDER BY snapshot_date DESC))[1] AS last_size
    FROM capacity_metrics_history
)
SELECT
    first_date,
    last_date,
    (last_date - first_date) AS days_measured,
    pg_size_pretty(first_size) AS size_at_start,
    pg_size_pretty(last_size)  AS size_at_end,
    round(
        (power(
            (last_size::numeric / nullif(first_size, 0)),
            30.0 / nullif(last_date - first_date, 0)
        ) - 1) * 100, 2
    ) AS estimated_monthly_growth_pct
FROM first_last;
```

### สูตรคาดการณ์อนาคต (Forecasting Formula)

เมื่อรู้อัตราการเติบโตต่อเดือนแล้ว (สมมติว่าเป็น **linear growth** หรือ **exponential growth**) เราใช้สูตรพื้นฐานคาดการณ์ล่วงหน้าได้:

**Linear Growth** (เติบโตคงที่ต่องวด เช่น +50GB ทุกเดือน):

```
Size(n เดือนข้างหน้า) = Size(ปัจจุบัน) + (Growth ต่อเดือน × n)
```

**Exponential Growth** (เติบโตเป็นเปอร์เซ็นต์ทบต้น เช่น +8% ต่อเดือน — พบบ่อยในระบบที่กำลังโตเร็ว เช่น startup):

```
Size(n เดือนข้างหน้า) = Size(ปัจจุบัน) × (1 + growth_rate)^n
```

ตัวอย่างการคำนวณด้วย SQL โดยตรง (สมมติ growth 8% ต่อเดือน คาดการณ์ 12 เดือนข้างหน้า):

```sql
-- คาดการณ์ขนาดฐานข้อมูลล่วงหน้า 12 เดือน ด้วย exponential growth model
WITH current_state AS (
    SELECT pg_database_size(current_database()) AS current_size_bytes
),
growth_params AS (
    SELECT 0.08::numeric AS monthly_growth_rate  -- ปรับตามข้อมูลจริงที่วัดได้
)
SELECT
    n_month,
    pg_size_pretty(
        (cs.current_size_bytes * power(1 + gp.monthly_growth_rate, n_month))::bigint
    ) AS projected_size
FROM current_state cs, growth_params gp,
     generate_series(0, 12) AS n_month
ORDER BY n_month;
```

query นี้มีประโยชน์มาก เพราะสามารถนำ `monthly_growth_rate` ที่คำนวณได้จาก query ก่อนหน้า มาแทนใน CTE `growth_params` เพื่อดูอนาคตแบบเป็นตารางชัดเจน ทีมงานสามารถนำไปทำกราฟหรือ dashboard ต่อได้ทันที

### ตารางเปรียบเทียบ: Linear vs Exponential Growth

| ลักษณะระบบ | Growth Model ที่เหมาะสม | ตัวอย่าง |
|---|---|---|
| ระบบเสถียร มี user คงที่ | Linear | Internal tool, back-office system |
| ระบบกำลังขยายตลาด | Exponential | Startup, new product launch |
| ระบบตามฤดูกาล (seasonal) | Cyclical + trend | E-commerce, ระบบจองตั๋ว |
| ระบบ log/audit ที่ไม่มีการลบ | Linear (มักคงที่ตาม transaction rate) | Audit log, transaction history |

> ข้อควรระวัง: การประมาณด้วยโมเดลเดียวตลอด 12 เดือนมีความเสี่ยง เพราะ growth rate จริงมักไม่คงที่ (อาจมี seasonal spike เช่น เทศกาล) ควรทบทวนตัวเลขทุก 1-3 เดือน และปรับ forecast ใหม่เสมอ (rolling forecast) ไม่ใช่คาดการณ์ครั้งเดียวแล้วจบ

---

## Step 743: Vertical Scaling (Scale Up)

### แนวคิด

**Vertical Scaling** หรือ **Scale Up** คือการเพิ่มทรัพยากรให้กับ server เดิม (เครื่องเดียว) เช่น:

- เพิ่มจำนวน CPU core (เช่นจาก 4 core เป็น 16 core)
- เพิ่ม RAM (เช่นจาก 16GB เป็น 64GB)
- เปลี่ยน storage เป็นแบบเร็วขึ้น (เช่นจาก HDD เป็น SSD หรือจาก SSD เป็น NVMe)
- เพิ่มขนาด storage (เช่นจาก 500GB เป็น 2TB)

Vertical Scaling เป็นวิธีการ scale ที่ **ง่ายที่สุด** สำหรับ PostgreSQL เพราะ PostgreSQL ถูกออกแบบมาให้ทำงานบนเครื่องเดียวเป็นหลัก (single-node architecture) — ไม่ต้องเปลี่ยนสถาปัตยกรรมของแอปพลิเคชันหรือ query ใดๆ เลย เพียงแค่เปลี่ยนขนาดเครื่องแล้ว restart หรือปรับ configuration ให้เข้ากับทรัพยากรใหม่

### ข้อดีของ Vertical Scaling

1. **ทำได้ง่ายและรวดเร็ว** — โดยเฉพาะบน cloud provider ที่สามารถเปลี่ยน instance type ได้ภายในไม่กี่นาที (แม้จะมี downtime สั้นๆ ระหว่าง restart)
2. **ไม่ต้องแก้โค้ดแอปพลิเคชัน** — connection string, query, transaction logic ยังเหมือนเดิมทั้งหมด
3. **ไม่มีปัญหาเรื่อง data consistency ข้าม node** — เพราะยังเป็น single-node เหมือนเดิม
4. **เหมาะกับ workload ที่ยังไม่ใหญ่มาก** — สำหรับระบบ SME หรือ mid-size, vertical scaling อาจเพียงพอไปอีกหลายปี

### ข้อจำกัดสูงสุด (Ceiling) ของ Vertical Scaling

แม้จะง่าย แต่ Vertical Scaling มี **เพดาน (ceiling)** ที่ชนได้เสมอ:

| ทรัพยากร | ข้อจำกัดในทางปฏิบัติ | ผลกระทบเมื่อชนเพดาน |
|---|---|---|
| CPU core | Cloud provider มักจำกัดที่ 96-128 vCPU ต่อ instance (ขึ้นกับ instance family) | Query ที่ใช้ CPU มาก (aggregation, sort) จะช้าลงเมื่อ concurrent query สูงขึ้นแม้ core จะเยอะ เพราะ contention บน lock/latch ภายใน PostgreSQL เอง |
| RAM | instance ใหญ่สุดในตลาดคลาวด์ปัจจุบันมักอยู่ราว 1-24TB | shared_buffers ที่ใหญ่เกินไปอาจไม่ได้ช่วยเสมอไป เพราะ OS page cache และ checkpoint overhead เพิ่มขึ้นตาม |
| Storage IOPS | disk เดี่ยวมี IOPS จำกัด แม้เป็น NVMe ก็มีเพดาน | เมื่อ concurrent write สูง จะเกิด I/O wait สูงขึ้น |
| Network bandwidth | NIC ของเครื่องมีขีดจำกัด (เช่น 25 Gbps) | จำกัดปริมาณ data ที่ streaming replica หรือ client ดึงออกได้พร้อมกัน |
| ค่าใช้จ่าย | ราคาต่อหน่วยประสิทธิภาพเพิ่มแบบไม่เป็นเส้นตรง (non-linear cost) | instance ที่ใหญ่ที่สุดอาจแพงกว่า instance กลางถึง 10 เท่า ทั้งที่ประสิทธิภาพเพิ่มแค่ 2-3 เท่า |
| Downtime ระหว่าง resize | การ resize มักต้อง restart instance | แม้จะสั้น (นาทีถึงสิบนาที) แต่ระบบ mission-critical อาจรับความเสี่ยงนี้ไม่ได้ |

### จุดที่ควรเริ่มพิจารณา Scale Out แทน Scale Up

Vertical Scaling ควรถูกพิจารณาเปลี่ยนแนวทางเมื่อ:

- CPU utilization สูงเกิน 70-80% อย่างต่อเนื่องแม้ upgrade ไปเป็น instance ขนาดใหญ่ที่สุดที่มีให้เลือกแล้ว
- ค่าใช้จ่ายในการ upgrade เพิ่มขึ้นแบบก้าวกระโดดเทียบกับประสิทธิภาพที่ได้ (diminishing returns)
- Business ต้องการ **High Availability** ที่แท้จริง (single node ยังไงก็มี single point of failure)
- ปริมาณ **write throughput** เกินกว่าที่เครื่องเดียวจะรองรับได้ในทางฟิสิกส์ (แม้จะเพิ่ม RAM/CPU เท่าไหร่ disk write ก็ยังจำกัดด้วย WAL sequential write)

### SQL ตรวจสอบว่าใกล้ชนเพดาน CPU/Memory หรือยัง

```sql
-- ตรวจสอบ cache hit ratio (ถ้าต่ำกว่า 99% อาจหมายความว่า RAM ไม่พอสำหรับ working set)
SELECT
    sum(blks_hit)                                   AS total_cache_hits,
    sum(blks_read)                                  AS total_disk_reads,
    round(
        sum(blks_hit)::numeric
        / nullif(sum(blks_hit) + sum(blks_read), 0) * 100, 2
    ) AS cache_hit_ratio_pct
FROM pg_stat_database
WHERE datname NOT IN ('template0', 'template1');
```

```sql
-- ดู query ที่ใช้ CPU/เวลารวมมากที่สุด (ต้องเปิด extension pg_stat_statements ก่อน)
SELECT
    round(total_exec_time::numeric, 2)   AS total_ms,
    calls,
    round(mean_exec_time::numeric, 2)    AS mean_ms,
    round((100 * total_exec_time / sum(total_exec_time) OVER ())::numeric, 2) AS pct_of_total,
    left(query, 80)                       AS query_snippet
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 15;
```

```sql
-- ตรวจสอบจำนวน active query ที่กำลังทำงานอยู่จริง ณ ขณะนี้ (สัญญาณของ CPU/lock saturation)
SELECT
    state,
    count(*) AS session_count
FROM pg_stat_activity
WHERE pid <> pg_backend_pid()
GROUP BY state
ORDER BY session_count DESC;
```

หาก query ข้างต้นแสดงว่ามี session จำนวนมากอยู่ในสถานะ `active` พร้อมกันตลอดเวลา (ไม่ใช่แค่ชั่วครู่) นั่นคือสัญญาณว่า CPU กำลังทำงานเต็มพิกัด และเป็นจุดเริ่มต้นที่ต้องประเมินว่าจะ scale up ต่อ หรือถึงเวลาพิจารณา scale out แล้ว

---

## Step 744: Horizontal Scaling (Scale Out)

### แนวคิด

**Horizontal Scaling** หรือ **Scale Out** คือการกระจายโหลดไปยังหลาย server แทนที่จะพึ่งเครื่องเดียว แนวคิดนี้เป็นหัวใจของระบบ distributed system สมัยใหม่ เช่น NoSQL database (Cassandra, MongoDB sharded cluster) หรือ web application ที่ scale ผ่าน load balancer + หลาย instance

```
Vertical Scaling:                Horizontal Scaling:

   [ Server ใหญ่ขึ้นเรื่อยๆ ]         [Server 1] [Server 2] [Server 3] [Server N]
        ↑ ↑ ↑                              ↑         ↑         ↑         ↑
   เพิ่ม CPU/RAM/Disk                  กระจายโหลดไปหลายเครื่องแทน
```

### ทำไม RDBMS แบบดั้งเดิม (รวมถึง PostgreSQL) ถึง Scale Out ยาก

ระบบ NoSQL จำนวนมากถูกออกแบบมาตั้งแต่ต้นให้กระจายข้อมูลได้ง่าย (design for distribution) แต่ PostgreSQL และ RDBMS ดั้งเดิมส่วนใหญ่ถูกออกแบบมาเป็น **single-node ACID-compliant system** ตั้งแต่แรก ทำให้การ scale out มีความท้าทายเชิงพื้นฐานหลายประการ:

**1. ACID Transaction ข้าม node**

PostgreSQL รับประกัน ACID (Atomicity, Consistency, Isolation, Durability) ภายในเครื่องเดียวได้อย่างสมบูรณ์แบบ แต่เมื่อข้อมูลกระจายไปหลายเครื่อง การทำ transaction ที่ต้องแก้ไขข้อมูลใน node หลายตัวพร้อมกันแบบ atomic (all-or-nothing) จะซับซ้อนขึ้นมาก ต้องใช้ protocol อย่าง **Two-Phase Commit (2PC)** ซึ่งมี overhead สูงและเสี่ยงต่อปัญหา blocking หาก node ใด node หนึ่งตอบสนองช้า

**2. Join ข้าม node**

ในเครื่องเดียว การทำ `JOIN` ระหว่างตารางเป็นเรื่องปกติและเร็ว เพราะข้อมูลอยู่ใน disk เดียวกัน แต่เมื่อข้อมูลถูกแบ่ง (shard) ไปคนละเครื่อง การ join ข้ามเครื่องต้องส่งข้อมูลผ่าน network ซึ่งช้ากว่ามาก และบางกรณีต้องดึงข้อมูลทั้งหมดมารวมที่เครื่องเดียวก่อน join (data shuffling) ทำให้เสียประสิทธิภาพ

**3. Foreign Key และ Referential Integrity ข้าม node**

PostgreSQL บังคับ foreign key constraint ได้อย่างเข้มงวดในเครื่องเดียว แต่ถ้าตารางแม่และตารางลูกอยู่คนละ node การบังคับ constraint นี้แบบ real-time ทำได้ยากหรือทำไม่ได้เลยในทางปฏิบัติ

**4. Global Unique Constraint / Sequence**

`SERIAL` หรือ `sequence` ที่ให้ค่า auto-increment แบบ unique ทั่วทั้งระบบ เมื่อกระจายไปหลาย node จะต้องมีกลไกพิเศษ (เช่น แบ่งช่วงเลขให้แต่ละ node หรือใช้ UUID แทน) เพราะแต่ละ node ไม่สามารถรู้ค่าล่าสุดของกันและกันได้แบบ real-time โดยไม่มี network overhead

**5. Distributed Query Planning**

Query planner ของ PostgreSQL ปกติวางแผนโดยรู้ข้อมูล statistics ของตารางในเครื่องเดียว แต่เมื่อข้อมูลกระจาย ต้องมี **distributed query planner** ที่ฉลาดพอจะรู้ว่าข้อมูลส่วนไหนอยู่ node ไหน และควรส่ง query ไปประมวลผลที่ node นั้นโดยตรง (query pushdown) แทนที่จะดึงข้อมูลทั้งหมดมา process ที่ส่วนกลาง

### ตารางเปรียบเทียบ Vertical vs Horizontal Scaling

| ประเด็น | Vertical Scaling | Horizontal Scaling |
|---|---|---|
| ความซับซ้อนในการ implement | ต่ำ | สูง |
| ต้องแก้โค้ดแอปพลิเคชันหรือไม่ | ไม่ต้อง | มักต้อง (เช่น เลือก shard key, เลี่ยง cross-shard join) |
| เพดานสูงสุด | มีจำกัด (ขนาดเครื่องใหญ่สุดที่หาซื้อได้) | ในทางทฤษฎีแทบไม่จำกัด (เพิ่มเครื่องได้เรื่อยๆ) |
| ความพร้อมใช้งาน (Availability) | Single point of failure | กระจายความเสี่ยง ถ้าออกแบบดี node ล่มได้บางส่วนโดยระบบยังทำงานต่อ |
| ต้นทุนต่อหน่วยประสิทธิภาพ | แพงขึ้นแบบไม่เป็นเส้นตรงเมื่อใหญ่มาก | ใช้ commodity hardware หลายเครื่องมักคุ้มกว่า |
| เหมาะกับ | ระบบขนาดเล็ก-กลาง, ระบบที่ยังไม่ถึงเพดานฮาร์ดแวร์ | ระบบขนาดใหญ่มาก, ต้องการ availability สูง, write throughput สูงมาก |
| Downtime ระหว่างขยาย | มักมี (restart) | ขึ้นกับสถาปัตยกรรม อาจทำ rolling ได้ |

### แนวทางที่ PostgreSQL Ecosystem ใช้แก้ปัญหา Scale Out

แม้ PostgreSQL core เองจะไม่รองรับ native sharding ในตัว (ต่างจาก NoSQL บางระบบ) แต่ ecosystem ได้พัฒนาแนวทางหลายแบบเพื่อให้ scale out ได้ในทางปฏิบัติ:

1. **Read Replica + Load Balancer** — แก้ปัญหาเฉพาะ read scaling (รายละเอียดใน Step 745)
2. **Citus Extension** — เปลี่ยน PostgreSQL ให้เป็น distributed database ที่ shard ข้อมูลอัตโนมัติ พร้อม distributed query planner (จะเจาะลึกใน Part 097)
3. **Application-level Sharding** — แอปพลิเคชันเองเป็นคนตัดสินใจว่าข้อมูลชุดไหนไปอยู่ database instance ไหน (manual sharding)
4. **Foreign Data Wrapper (FDW)** — เชื่อมข้อมูลจากหลาย PostgreSQL instance เข้าด้วยกันแบบ federated query
5. **Logical Replication แบบ Multi-master** (ผ่าน extension เช่น pglogical หรือ BDR) — ให้หลาย node รับ write ได้พร้อมกัน แต่ต้องจัดการ conflict resolution เอง

หัวใจสำคัญคือ: **การ scale out ฐานข้อมูลเชิงสัมพันธ์ไม่ใช่แค่ "เพิ่มเครื่อง" แต่ต้องออกแบบสถาปัตยกรรมข้อมูลใหม่** ซึ่งต่างจาก vertical scaling ที่แทบไม่ต้องเปลี่ยนอะไรเลย

---

## Step 745: Read Scaling ด้วย Read Replica

### ทบทวนแนวคิดจาก Part 067

ใน Part 067 (Load Balancing) เราได้พูดถึงการใช้ **Streaming Replication** (Part 063) ร่วมกับ load balancer เพื่อกระจาย read query ไปยัง replica หลายตัว หลักการคือ:

```
                       ┌──────────────┐
        Write ────────▶│   Primary    │
                       └──────┬───────┘
                              │ streaming replication (WAL)
                ┌─────────────┼─────────────┐
                ▼             ▼             ▼
         ┌───────────┐ ┌───────────┐ ┌───────────┐
         │ Replica 1 │ │ Replica 2 │ │ Replica 3 │
         └───────────┘ └───────────┘ └───────────┘
                ▲             ▲             ▲
                └─────────────┼─────────────┘
                              │
                     Load Balancer (Read)
                              ▲
                          Read Query
```

### สิ่งที่ Read Replica ขยายได้ และขยายไม่ได้

**ขยายได้ (Scale ได้จริง):**

- ปริมาณ **read throughput** — เพิ่ม replica เท่าไหร่ ก็กระจาย read query ได้มากขึ้นตามนั้น (เกือบเป็นเส้นตรง)
- ความพร้อมใช้งานสำหรับ read workload — หาก replica ตัวหนึ่งล่ม การอ่านยังไปที่ replica ตัวอื่นได้
- แยก workload ที่หนัก เช่น reporting/analytics query ออกจาก transactional workload (แยก replica เฉพาะสำหรับ BI/reporting)

**ขยายไม่ได้ (ข้อจำกัดสำคัญ):**

- **Write throughput** — ทุก write ยังคงต้องผ่าน Primary เท่านั้น เพิ่ม replica กี่ตัวก็ไม่ช่วยให้ write เร็วขึ้นหรือรองรับ write มากขึ้น
- **Storage capacity ต่อ node** — แต่ละ replica เก็บสำเนาข้อมูลทั้งหมดเหมือนกัน (full copy) ดังนั้นถ้าข้อมูลใหญ่ 10TB ทุก replica ก็ต้องมี disk อย่างน้อย 10TB เท่ากันหมด ไม่ได้ช่วยประหยัด storage เลย
- **Replication Lag** — ข้อมูลที่อ่านจาก replica อาจไม่ใช่ข้อมูลล่าสุด (eventual consistency) เพราะ WAL ต้อง apply ตามหลัง primary เสมอ

### SQL ตรวจสอบ Replication Lag เพื่อประกอบการวางแผน

```sql
-- รันที่ Primary: ดูสถานะและ lag ของแต่ละ replica ที่เชื่อมต่ออยู่
SELECT
    client_addr,
    application_name,
    state,
    pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) AS lag_bytes,
    pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn)) AS lag_pretty,
    write_lag,
    flush_lag,
    replay_lag
FROM pg_stat_replication
ORDER BY lag_bytes DESC;
```

```sql
-- รันที่ Replica: ดูว่า apply ข้อมูลล่าช้าไปกี่วินาที
SELECT
    CASE
        WHEN pg_is_in_recovery() THEN
            EXTRACT(EPOCH FROM (now() - pg_last_xact_replay_timestamp()))
        ELSE 0
    END AS replication_lag_seconds;
```

### การวางแผนจำนวน Replica ที่ต้องมี

สูตรง่ายๆ ในการประมาณจำนวน replica ที่ต้องการ:

```
จำนวน Replica ที่ต้องการ (โดยประมาณ)
    = ceil( Read Query Rate ที่คาดการณ์ / Read Query Rate ที่ Replica 1 ตัวรองรับได้สูงสุด )
      + Replica สำรองสำหรับ Failover (buffer)
```

ตัวอย่าง: หากระบบต้องรองรับ 15,000 read query/วินาที และ replica แต่ละตัว (ด้วยขนาด hardware เดียวกัน) รองรับได้สูงสุด 4,000 query/วินาทีก่อนที่ latency จะเริ่มแย่ลง:

```
จำนวน Replica ที่ต้องการ = ceil(15,000 / 4,000) = 4 ตัว
บวก buffer สำหรับ failover/maintenance อีก 1 ตัว = 5 ตัวทั้งหมด
```

### ตารางแนวทางการแบ่ง Workload ระหว่าง Primary กับ Replica

| ประเภท Workload | ควรไปที่ | เหตุผล |
|---|---|---|
| INSERT / UPDATE / DELETE | Primary เท่านั้น | Replica เป็น read-only เสมอ (standby) |
| Read ที่ต้องการข้อมูล real-time แบบเป๊ะ (เช่น อ่านทันทีหลัง write ในธุรกรรมเดียวกัน) | Primary | เลี่ยงปัญหา read-after-write inconsistency จาก replication lag |
| Read ทั่วไปที่ยอมรับ lag ได้ (เช่น หน้ารายการสินค้า) | Replica | กระจายโหลดออกจาก Primary |
| Reporting / Analytics / Batch export | Replica เฉพาะ (แยก replica ต่างหาก) | กัน query หนักไม่ให้กระทบ transactional replica ตัวอื่น |
| Backup (pg_basebackup, pg_dump) | Replica | ลดภาระบน Primary ระหว่าง backup |

---

## Step 746: Write Scaling

### ทำไม Write Scaling ถึงยากกว่า Read Scaling มาก

Read Scaling ทำได้ง่ายเพราะการอ่านข้อมูล **ไม่เปลี่ยนแปลงสถานะ (state)** ของข้อมูล เราจึงคัดลอกข้อมูลไปหลายที่แล้วให้อ่านจากที่ไหนก็ได้ที่ตรงกัน (eventual consistency ยอมรับได้ในหลายกรณี)

แต่ Write เปลี่ยนสถานะของข้อมูล และ **ทุก node ที่เกี่ยวข้องต้องเห็นข้อมูลตรงกันในที่สุด** ปัญหาหลักที่ทำให้ write scaling ยาก:

1. **Single Writer Constraint** — ในสถาปัตยกรรม Primary-Replica แบบดั้งเดิม มีเพียง Primary node เดียวเท่านั้นที่รับ write ได้ (เพื่อป้องกัน conflict) การเพิ่ม node ไม่ได้ช่วยเพิ่ม write capacity เลย
2. **WAL เป็น Sequential Write** — ทุก transaction ต้องเขียนลง Write-Ahead Log ตามลำดับ (sequential) เพื่อรับประกัน durability ทำให้ write throughput ถูกจำกัดด้วยความเร็วของ disk WAL เป็นคอขวดตามธรรมชาติ
3. **Lock Contention** — เมื่อหลาย transaction พยายามแก้ไข row หรือ table เดียวกันพร้อมกัน จะเกิดการรอ lock ซึ่งยิ่งมี concurrent write มาก การรอคอยยิ่งสูง
4. **Global Consistency ข้าม Node** — หากต้องการให้หลาย node รับ write ได้พร้อมกัน (multi-master) ต้องแก้ปัญหา conflict resolution: ถ้าสอง node แก้ไข row เดียวกันพร้อมกัน ใครชนะ? และต้อง sync กันอย่างไรโดยไม่กระทบ latency

### แนวทางเบื้องต้นของ Sharding

**Sharding** คือการแบ่งข้อมูลออกเป็นส่วนย่อย (shard) ตาม key บางอย่าง แล้วกระจายแต่ละ shard ไปอยู่คนละ node เพื่อให้แต่ละ node รับ write เฉพาะข้อมูลของตัวเองเท่านั้น ทำให้ write throughput รวมของทั้งระบบเพิ่มขึ้นตามจำนวน node (แทนที่จะจำกัดที่ node เดียว)

```
ตัวอย่างการ Shard ตาม customer_id (hash-based sharding):

customer_id ที่มี hash % 4 = 0  ──▶ Shard 1 (Node A)
customer_id ที่มี hash % 4 = 1  ──▶ Shard 2 (Node B)
customer_id ที่มี hash % 4 = 2  ──▶ Shard 3 (Node C)
customer_id ที่มี hash % 4 = 3  ──▶ Shard 4 (Node D)
```

### กลยุทธ์การเลือก Shard Key

การเลือก **shard key** (คีย์ที่ใช้ตัดสินว่าข้อมูลแถวไหนไปอยู่ shard ไหน) เป็นการตัดสินใจที่สำคัญที่สุดในการออกแบบระบบ sharded และแก้ไขทีหลังได้ยากมาก:

| กลยุทธ์ | วิธีการ | ข้อดี | ข้อเสีย |
|---|---|---|---|
| **Hash-based** | ใช้ hash function กับ key แล้ว mod ด้วยจำนวน shard | กระจายข้อมูลสม่ำเสมอ (even distribution) | Range query (เช่น "ดึง order ทั้งหมดของเดือนนี้") ต้องกระจายไปทุก shard |
| **Range-based** | แบ่งตามช่วงค่า (เช่น customer_id 1-1000000 ไป shard 1) | Range query มีประสิทธิภาพดี | เสี่ยง hotspot ถ้าข้อมูลใหม่กระจุกที่ range เดียว |
| **Directory-based** | มีตาราง lookup บอกว่า key ไหนอยู่ shard ไหน | ยืดหยุ่นสูง ย้ายข้อมูลได้ง่าย | ต้อง query lookup table ก่อนเสมอ เพิ่ม latency และเป็น single point of failure ถ้าไม่ดูแลดี |
| **Geographic/Tenant-based** | แบ่งตามภูมิภาคหรือ tenant (เช่น multi-tenant SaaS) | ตรงกับความต้องการทางธุรกิจ (data residency) | ขนาดแต่ละ shard ไม่เท่ากัน อาจมี tenant ใหญ่ผิดปกติ (noisy neighbor) |

### ข้อควรพิจารณาก่อนตัดสินใจ Shard

Sharding เป็นการตัดสินใจสถาปัตยกรรมที่ **ย้อนกลับยาก** เมื่อทำแล้ว การเปลี่ยน shard key ในภายหลังต้องทำ data migration ครั้งใหญ่ ดังนั้นก่อนตัดสินใจควรถามคำถามเหล่านี้ก่อนเสมอ:

1. เราถึงเพดานของ Vertical Scaling จริงหรือยัง? (ลอง upgrade instance ที่ใหญ่ที่สุดที่มีก่อนหรือยัง)
2. Workload ของเราสามารถแบ่งตาม key ที่ชัดเจนได้หรือไม่ (เช่น tenant_id, customer_id, region)?
3. Query ส่วนใหญ่ของเราเป็น single-shard query หรือ cross-shard query? (ถ้า cross-shard join เยอะ sharding อาจทำให้ระบบช้าลงแทนที่จะเร็วขึ้น)
4. ทีมมีความสามารถดูแลระบบ distributed ที่ซับซ้อนขึ้นหรือไม่?

> **หมายเหตุสำคัญ**: บทนี้เพียงเกริ่นแนวคิดพื้นฐานของ Sharding เพื่อให้เข้าใจภาพรวมก่อน ในทางปฏิบัติ PostgreSQL ไม่มีกลไก native sharding ในตัว การ implement sharding แบบจริงจังในโลกความเป็นจริงมักใช้ extension อย่าง **Citus** ซึ่งจัดการเรื่อง distributed query planning, shard rebalancing, และ cross-shard transaction ให้อัตโนมัติ เราจะเจาะลึกเรื่องนี้แบบเต็มรูปแบบใน **Part 097**

### SQL ตรวจสอบว่า Write Load กำลังเป็นคอขวดหรือยัง

```sql
-- ตรวจสอบอัตราส่วน write กับ read และดู volume ของ WAL ที่ generate
SELECT
    datname,
    tup_inserted + tup_updated + tup_deleted   AS total_writes,
    tup_returned + tup_fetched                 AS total_reads,
    round(
        (tup_inserted + tup_updated + tup_deleted)::numeric
        / nullif(tup_returned + tup_fetched, 0) * 100, 4
    ) AS write_to_read_ratio_pct
FROM pg_stat_database
WHERE datname NOT IN ('template0', 'template1');
```

```sql
-- ตรวจสอบ lock contention ปัจจุบัน (สัญญาณว่า concurrent write ชนกันมากแค่ไหน)
SELECT
    locktype,
    mode,
    granted,
    count(*) AS lock_count
FROM pg_locks
GROUP BY locktype, mode, granted
ORDER BY lock_count DESC;
```

```sql
-- ดูว่ามี query ไหนกำลังรอ lock อยู่นาน (blocking chains)
SELECT
    blocked.pid          AS blocked_pid,
    blocked.query        AS blocked_query,
    blocking.pid         AS blocking_pid,
    blocking.query       AS blocking_query,
    now() - blocked.query_start AS blocked_duration
FROM pg_stat_activity AS blocked
JOIN pg_locks AS bl ON bl.pid = blocked.pid AND NOT bl.granted
JOIN pg_locks AS kl ON kl.locktype = bl.locktype
    AND kl.database IS NOT DISTINCT FROM bl.database
    AND kl.relation IS NOT DISTINCT FROM bl.relation
    AND kl.granted
JOIN pg_stat_activity AS blocking ON blocking.pid = kl.pid
WHERE blocked.pid <> blocking.pid
ORDER BY blocked_duration DESC;
```

หากพบว่ามี blocking chain ยาวและถี่ขึ้นเรื่อยๆ ตามจำนวน concurrent write ที่เพิ่มขึ้น นั่นเป็นสัญญาณว่ากำลังเข้าใกล้เพดานของ write capacity บนสถาปัตยกรรม single-primary แบบเดิม และควรเริ่มพิจารณาแนวทาง sharding หรือ Citus ในระยะยาว

---

## Step 747: Storage Capacity Planning

### การคาดการณ์การเติบโตของข้อมูล

Storage เป็นหนึ่งใน resource ที่ **คาดการณ์ได้ง่ายที่สุด** เมื่อเทียบกับ CPU/Memory เพราะข้อมูลมักโตแบบสม่ำเสมอตามจำนวน transaction ที่เกิดขึ้น (ไม่เหมือน CPU ที่ผันผวนตาม query pattern)

สูตรพื้นฐานในการคาดการณ์:

```
Storage ที่ต้องการในอนาคต
    = Storage ปัจจุบัน
    + (อัตราการเติบโตของข้อมูลต่อวัน × จำนวนวันที่คาดการณ์)
    + พื้นที่สำหรับ Index (โดยเฉลี่ย 20-40% ของขนาดตาราง)
    + พื้นที่สำหรับ WAL และ temporary file
    + Safety Margin (โดยทั่วไป 20-30% ของยอดรวม)
```

### SQL วิเคราะห์ขนาดตารางแต่ละตัวและอัตราการเติบโต

```sql
-- ขนาดของแต่ละตาราง แยก table data กับ index อย่างชัดเจน เรียงจากใหญ่สุด
SELECT
    schemaname,
    relname                                          AS table_name,
    pg_size_pretty(pg_table_size(c.oid))             AS table_size,
    pg_size_pretty(pg_indexes_size(c.oid))           AS indexes_size,
    pg_size_pretty(pg_total_relation_size(c.oid))    AS total_size,
    n_live_tup                                       AS estimated_rows
FROM pg_stat_user_tables s
JOIN pg_class c ON c.oid = s.relid
ORDER BY pg_total_relation_size(c.oid) DESC
LIMIT 20;
```

```sql
-- ประมาณขนาดเฉลี่ยต่อแถว เพื่อคาดการณ์ว่าถ้าแถวเพิ่มขึ้น N แถว จะใช้ storage เพิ่มเท่าไหร่
SELECT
    relname                                                  AS table_name,
    n_live_tup                                               AS current_rows,
    pg_total_relation_size(c.oid)                            AS total_bytes,
    round(
        pg_total_relation_size(c.oid)::numeric
        / nullif(n_live_tup, 0), 2
    ) AS avg_bytes_per_row
FROM pg_stat_user_tables s
JOIN pg_class c ON c.oid = s.relid
WHERE n_live_tup > 0
ORDER BY total_bytes DESC
LIMIT 20;
```

```sql
-- คาดการณ์ขนาดตารางล่วงหน้า เมื่อรู้อัตราการเพิ่มแถวต่อวัน (rows_per_day ต้องแทนค่าจริงจากธุรกิจ)
WITH table_stats AS (
    SELECT
        relname                                    AS table_name,
        n_live_tup                                  AS current_rows,
        pg_total_relation_size(c.oid)::numeric
            / nullif(n_live_tup, 0)                AS avg_bytes_per_row
    FROM pg_stat_user_tables s
    JOIN pg_class c ON c.oid = s.relid
    WHERE relname = 'orders'   -- เปลี่ยนชื่อตารางตามจริง
),
growth_assumption AS (
    SELECT 50000::bigint AS rows_added_per_day       -- ปรับตามข้อมูลจริงที่วัดได้จาก Step 742
)
SELECT
    table_name,
    current_rows,
    n_days,
    current_rows + (ga.rows_added_per_day * n_days)              AS projected_rows,
    pg_size_pretty(
        ((current_rows + (ga.rows_added_per_day * n_days))
            * avg_bytes_per_row)::bigint
    ) AS projected_size
FROM table_stats, growth_assumption ga,
     generate_series(0, 365, 30) AS n_days
ORDER BY n_days;
```

### กลยุทธ์ Archive ข้อมูลเก่า (เชื่อมโยง Partitioning จาก Part 054)

การคาดการณ์ storage ไม่ได้จบแค่ "จะต้องซื้อ disk เพิ่มเท่าไหร่" แต่ต้องพิจารณาคู่กับ **กลยุทธ์การลดขนาดข้อมูล** ด้วย เพราะข้อมูลจำนวนมาก โดยเฉพาะ transactional data ที่เก่าเกิน 1-2 ปี มักไม่ถูก query บ่อยแล้ว แต่ยังกิน storage เท่าเดิม

ดังที่ได้เรียนใน Part 054 (Table Partitioning) การแบ่งตารางตามช่วงเวลา (range partitioning ตามเดือนหรือปี) ทำให้เรา **archive หรือลบ partition เก่าได้ง่ายมาก** โดยไม่ต้องสแกนทั้งตาราง

```sql
-- ตัวอย่าง: ตรวจสอบว่า partition ไหนของตาราง orders เก่าเกิน 24 เดือนแล้วบ้าง
-- (สมมติว่า orders ถูก partition แบบ RANGE ตาม created_at เป็นรายเดือน ตามที่สอนใน Part 054)
SELECT
    child.relname                                          AS partition_name,
    pg_size_pretty(pg_total_relation_size(child.oid))      AS partition_size,
    pg_get_expr(child.relpartbound, child.oid)             AS partition_range
FROM pg_inherits
JOIN pg_class parent ON pg_inherits.inhparent = parent.oid
JOIN pg_class child  ON pg_inherits.inhrelid  = child.oid
WHERE parent.relname = 'orders'
ORDER BY child.relname;
```

```sql
-- ตัวอย่าง: ย้าย partition เก่าไปเก็บใน archive tablespace (แทนที่จะลบทิ้งเลย)
-- สมมติสร้าง tablespace 'archive_storage' ไว้บน disk ที่ถูกกว่า (เช่น HDD/cold storage)
ALTER TABLE orders_2023_01 SET TABLESPACE archive_storage;
```

```sql
-- ตัวอย่าง: detach partition เก่าออกจากตารางหลัก แล้วค่อยตัดสินใจว่าจะ archive เป็นไฟล์แยก หรือลบทิ้ง
ALTER TABLE orders DETACH PARTITION orders_2022_01;

-- จากนั้นอาจ export เป็นไฟล์ archive (เช่น ผ่าน COPY หรือ pg_dump เฉพาะตาราง) ก่อนจะ DROP
-- COPY orders_2022_01 TO '/archive/orders_2022_01.csv' WITH CSV HEADER;
-- DROP TABLE orders_2022_01;
```

### ตารางกลยุทธ์ Archive ตามลักษณะข้อมูล

| ประเภทข้อมูล | กลยุทธ์แนะนำ | เหตุผล |
|---|---|---|
| Transactional data ที่ยังต้อง query บ่อย (0-3 เดือน) | เก็บใน primary storage (SSD/NVMe) | ต้องการ performance สูงสุด |
| Transactional data ที่ query น้อยลง (3-24 เดือน) | เก็บใน primary storage แต่ partition แยกไว้ | ยังต้อง query ได้ แต่ไม่บ่อย |
| ข้อมูลเก่า (มากกว่า 24 เดือน) ที่ต้องเก็บตามกฎหมาย/compliance | ย้ายไป cold storage tablespace หรือ export เป็น cold storage (S3/Glacier) | ประหยัดค่าใช้จ่าย storage หลักได้มาก |
| ข้อมูลที่ไม่มีความจำเป็นต้องเก็บแล้ว (ตาม data retention policy) | ลบทิ้งตาม schedule (partition drop) | ลด storage และ backup size โดยตรง |
| Log/Audit trail | Partition รายวัน/สัปดาห์ + retention policy อัตโนมัติ | ข้อมูลประเภทนี้โตเร็วมากถ้าไม่มี retention policy |

> ข้อดีสำคัญของการใช้ partitioning ร่วมกับ archive strategy คือ **การลบ/ย้ายข้อมูลทำได้แบบ metadata operation** (DROP/DETACH PARTITION) ซึ่งเร็วมากและไม่ต้อง scan/lock ทั้งตารางเหมือนการ `DELETE FROM orders WHERE created_at < ...` แบบธรรมดา

---

## Step 748: Connection Capacity

### ทำไมต้องวางแผน Connection Capacity

`max_connections` ใน PostgreSQL เป็นค่าคงที่ที่ต้องกำหนดล่วงหน้า (ต้อง restart หากจะเปลี่ยน) และแต่ละ connection ที่เปิดอยู่จะกิน RAM ของ server (แม้จะ idle ก็ตาม เพราะแต่ละ connection คือ 1 OS process ใน PostgreSQL) การประเมิน connection capacity ผิดพลาดจะนำไปสู่ปัญหา 2 แบบ:

1. **ตั้ง `max_connections` ต่ำเกินไป** → แอปพลิเคชันได้ error "too many connections" เมื่อ traffic สูง
2. **ตั้ง `max_connections` สูงเกินไปโดยไม่มี pooling** → RAM ถูกใช้ไปกับ connection overhead จำนวนมาก และ context switching ระหว่าง process สูงจนทำให้ throughput โดยรวมลดลง (ทั้งที่ดูเหมือนน่าจะรองรับได้มากขึ้น)

### สูตรคำนวณ Connection ที่ต้องรองรับ

```
Concurrent Connections ที่ต้องรองรับ
    = จำนวน Application Instance × Pool Size ต่อ Instance
```

ตัวอย่าง: มี microservice 10 ตัว แต่ละตัว deploy 5 replica (pod) และแต่ละ pod เปิด connection pool ขนาด 20:

```
Concurrent Connections = 10 services × 5 replicas × 20 connections/pool = 1,000 connections
```

หากไม่มีการทำ connection pooling ที่ระดับ database (เช่น PgBouncer) นี่หมายความว่า `max_connections` ต้องตั้งอย่างน้อย 1,000 + ค่า reserve สำหรับ superuser/maintenance ซึ่งอาจสูงเกินกว่าที่ RAM ของเครื่องจะรองรับได้อย่างมีประสิทธิภาพ

### บทบาทของ Connection Pooling (ทบทวนจาก Part 066)

ดังที่ได้เรียนใน Part 066 (Connection Pooling) เครื่องมืออย่าง **PgBouncer** หรือ **pgpool-II** ช่วยแก้ปัญหานี้โดยทำหน้าที่เป็นตัวกลาง (proxy) ระหว่างแอปพลิเคชันกับ PostgreSQL:

```
[App 1] [App 2] ... [App 1000 pods]
     │      │              │
     └──────┼──────────────┘
            ▼
      ┌───────────┐
      │ PgBouncer │   ← รับ connection นับพันจาก app
      └─────┬─────┘
            │  แต่ส่งต่อไปที่ PostgreSQL เพียงไม่กี่สิบ connection จริง
            ▼
      ┌──────────────┐
      │ PostgreSQL   │   max_connections = 100 (เพียงพอ)
      └──────────────┘
```

ด้วยโหมด `transaction pooling` ของ PgBouncer แอปพลิเคชันหลายพัน connection สามารถแชร์ physical connection ไปยัง PostgreSQL จริงเพียงหลักสิบถึงหลักร้อยได้ เพราะ connection จริงจะถูกยืม-คืนกันในระดับ transaction แทนที่จะจับจองค้างไว้ตลอด session

### สูตรประมาณค่า `max_connections` ที่เหมาะสมเมื่อใช้ Pooling

```
max_connections (ที่ PostgreSQL) ที่แนะนำ
    ≈ (จำนวน CPU core × 2 ถึง 4)
      + จำนวน connection สำหรับ background process (autovacuum, replication, maintenance)
      + reserved_connections (สำหรับ superuser)
```

ตัวอย่าง: server มี 16 CPU core

```
max_connections ≈ (16 × 3) + 10 (background) + 5 (reserved) = 63 ≈ ตั้งไว้ 100 เผื่อ margin
```

### SQL วิเคราะห์การใช้งาน Connection ปัจจุบัน

```sql
-- แยกจำนวน connection ตามสถานะ เพื่อดูว่ามี idle connection ค้างเยอะแค่ไหน
SELECT
    state,
    count(*) AS connection_count,
    round(avg(extract(epoch FROM (now() - state_change)))::numeric, 1) AS avg_seconds_in_state
FROM pg_stat_activity
WHERE pid <> pg_backend_pid()
GROUP BY state
ORDER BY connection_count DESC;
```

```sql
-- ดู connection แยกตาม application_name และ client_addr เพื่อหาว่า service ไหนเปิด connection เยอะสุด
SELECT
    coalesce(application_name, 'unknown') AS application_name,
    client_addr,
    count(*)                              AS connection_count,
    count(*) FILTER (WHERE state = 'active') AS active_count,
    count(*) FILTER (WHERE state = 'idle')   AS idle_count
FROM pg_stat_activity
WHERE pid <> pg_backend_pid()
GROUP BY application_name, client_addr
ORDER BY connection_count DESC;
```

```sql
-- ตรวจจับ "idle in transaction" ซึ่งเป็นสัญญาณอันตราย (connection ถูกจับจองไว้แต่ไม่ทำงาน)
SELECT
    pid,
    application_name,
    state,
    now() - state_change AS idle_duration,
    query
FROM pg_stat_activity
WHERE state = 'idle in transaction'
ORDER BY idle_duration DESC;
```

`idle in transaction` ที่ค้างนานเป็นสาเหตุอันดับต้นๆ ของปัญหา connection exhaustion เพราะ connection เหล่านี้จับจอง slot ไว้โดยไม่ได้ทำงานจริง ทำให้ pool เต็มเร็วกว่าที่ควร ควรตั้งค่า `idle_in_transaction_session_timeout` เพื่อตัด session ประเภทนี้อัตโนมัติ

### ตารางสรุปแนวทางวางแผน Connection Capacity

| สถานการณ์ | คำแนะนำ |
|---|---|
| Microservice จำนวนมาก แต่ละตัวเปิด pool เอง | ใช้ PgBouncer เป็น central proxy ตัวเดียว (หรือหลายตัวแบบ HA) แทนที่แต่ละ service ต่อ PostgreSQL โดยตรง |
| Traffic มี burst สูงเป็นบางช่วง (เช่น flash sale) | ตั้ง pool size ให้รองรับ peak แต่เผื่อ queue/timeout ที่เหมาะสมเมื่อ pool เต็ม แทนที่จะปฏิเสธทันที |
| มี batch job ที่เปิด connection จำนวนมากพร้อมกัน | แยก pool/connection budget ต่างหากจาก transactional workload เพื่อไม่ให้แย่ง resource กัน |
| Query บางตัวใช้เวลานาน (long-running) | จำกัดด้วย `statement_timeout` เพื่อไม่ให้ connection ถูกจับจองนานเกินจำเป็น |

---

## Step 749: Load Testing ก่อน Scale จริง

### ทำไมต้อง Load Test ก่อนวันงานจริง

Capacity Planning ที่อาศัยแค่การคำนวณบนกระดาษ (theoretical calculation) มีความเสี่ยงเสมอ เพราะระบบจริงมีปัจจัยที่คาดเดายากหลายอย่าง เช่น query plan ที่เปลี่ยนไปเมื่อข้อมูลโตขึ้น, lock contention ที่ไม่ปรากฏตอน traffic น้อย, หรือ cache hit ratio ที่ตกลงเมื่อ working set ใหญ่กว่า RAM

**Load Testing** คือการจำลอง traffic ในระดับที่ใกล้เคียงหรือสูงกว่าที่คาดว่าจะเกิดขึ้นจริง (เช่น จำลอง traffic 10 เท่าของปัจจุบันสำหรับ campaign ใหญ่) เพื่อค้นหาคอขวด (bottleneck) **ก่อน** ที่จะไปเจอในสถานการณ์จริงที่แก้ไขไม่ทัน

### หลักการออกแบบ Load Test ที่ดี

1. **จำลอง workload ให้ใกล้เคียงของจริงที่สุด** — สัดส่วน read:write ต้องใกล้เคียงกับ production จริง ไม่ใช่แค่ยิง `SELECT 1` ซ้ำๆ
2. **ทดสอบแบบค่อยๆ เพิ่มโหลด (ramp-up)** — เพื่อดูว่า performance เริ่มแย่ลงที่จุดไหน (ไม่ใช่ยิงโหลดสูงสุดทันทีตั้งแต่วินาทีแรก)
3. **วัดทั้ง throughput และ latency** — throughput สูงแต่ latency (p95, p99) แย่ลงมาก ก็ถือว่าไม่ผ่าน
4. **ทดสอบบน environment ที่ใกล้เคียง production** — ทดสอบบนเครื่อง spec เล็กกว่ามากจะได้ผลลัพธ์ที่ไม่น่าเชื่อถือ
5. **Monitor ทุก layer ระหว่างทดสอบ** — ดู CPU, memory, disk I/O, lock, replication lag พร้อมกันทั้งหมด ไม่ใช่ดูแค่ "ผ่านหรือไม่ผ่าน"

### เกริ่นนำ pgbench (รายละเอียดเต็มใน Part 098)

PostgreSQL มีเครื่องมือ Load Testing ในตัวชื่อ **pgbench** ซึ่งสามารถจำลอง transactional workload (คล้าย TPC-B benchmark) หรือกำหนด custom script ของตัวเองก็ได้ ในบทนี้จะแนะนำแนวคิดพื้นฐานเพื่อเชื่อมโยงกับ Capacity Planning เท่านั้น ส่วนรายละเอียดการใช้งานเชิงลึก การตีความผลลัพธ์ และการเขียน custom benchmark script จะอยู่ใน **Part 098**

```bash
# ตัวอย่างแนวคิดเบื้องต้น (รายละเอียดเต็มใน Part 098):
# เตรียมข้อมูลทดสอบด้วย scale factor ที่เหมาะสมกับขนาดข้อมูลจริง
pgbench -i -s 100 mydb

# จำลอง 50 client พร้อมกัน รันต่อเนื่อง 300 วินาที
pgbench -c 50 -j 4 -T 300 mydb
```

ผลลัพธ์หลักที่ต้องสนใจคือ **TPS (Transactions Per Second)** และ **latency average/percentile** ซึ่งนำไปเทียบกับเป้าหมายที่คาดการณ์ไว้จาก Step 742 ได้โดยตรง

### กระบวนการ Load Testing ที่เชื่อมกับ Capacity Planning

```
1. กำหนดเป้าหมาย    →  "ต้องรองรับ 5,000 TPS ในช่วง campaign"
        ↓
2. จำลอง Workload   →  ใช้ pgbench หรือ custom script จำลอง traffic ใกล้เคียงจริง
        ↓
3. Ramp-up ทดสอบ    →  ค่อยๆ เพิ่ม concurrent client จาก 10 → 50 → 200 → 500 ...
        ↓
4. บันทึกจุดที่ latency เริ่มแย่ลงอย่างมีนัยสำคัญ (knee point)
        ↓
5. เทียบกับเป้าหมาย →  ถ้า knee point ต่ำกว่าเป้าหมาย = ต้อง scale เพิ่ม/ optimize query
        ↓
6. ปรับปรุงและทดสอบซ้ำ (index, query tuning, configuration, scale up/out)
        ↓
7. Sign-off          →  ยืนยันว่าระบบรองรับเป้าหมายได้จริงก่อนวันงาน
```

### SQL สำหรับ Monitor ระหว่าง Load Test

```sql
-- ระหว่าง load test ให้ monitor คอขวดที่พบบ่อยที่สุด: checkpoint และ WAL
SELECT
    now()                                            AS checked_at,
    checkpoints_timed,
    checkpoints_req,
    checkpoint_write_time,
    checkpoint_sync_time,
    buffers_checkpoint,
    buffers_backend
FROM pg_stat_bgwriter;
```

```sql
-- ดู top query ที่ตอบสนองช้าที่สุดระหว่าง load test (ต้องรีเซ็ต pg_stat_statements ก่อนเริ่มทดสอบทุกครั้ง)
-- SELECT pg_stat_statements_reset();  -- รันก่อนเริ่ม load test
SELECT
    round(mean_exec_time::numeric, 2)  AS mean_ms,
    round(max_exec_time::numeric, 2)   AS max_ms,
    calls,
    left(query, 100)                   AS query_snippet
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 15;
```

```sql
-- ตรวจ temp file usage ระหว่าง load test (สัญญาณว่า work_mem ไม่พอ)
SELECT
    datname,
    temp_files,
    pg_size_pretty(temp_bytes) AS temp_bytes_pretty
FROM pg_stat_database
WHERE datname NOT IN ('template0', 'template1')
ORDER BY temp_bytes DESC;
```

### ข้อควรระวังในการทำ Load Test

- **อย่ารันบน Production โดยตรงโดยไม่แจ้งเตือน** — ควรใช้ replica หรือ staging environment ที่มี spec และข้อมูลใกล้เคียง production จริง
- **ระวัง Cache Warm-up** — ผลลัพธ์รอบแรกของการทดสอบมักช้ากว่าปกติเพราะ cache ยังไม่ได้ warm ควรทดสอบซ้ำหลายรอบและดู trend
- **ทดสอบ Failure Scenario ด้วย** ไม่ใช่แค่ happy path — เช่น ทดสอบว่าเมื่อ replica ตัวหนึ่งล่มระหว่าง peak load ระบบยังรับโหลดได้ไหม
- **บันทึกผลทุกครั้งเป็น baseline** — เพื่อเปรียบเทียบเมื่อ scale หรือเปลี่ยน configuration ในอนาคต จะได้รู้ว่าดีขึ้นหรือแย่ลงจริง

---

## Step 750: แบบฝึกหัดรวม — จัดทำแผน Capacity Planning 12 เดือน

### โจทย์สถานการณ์

บริษัท "ShopVery" ดำเนินธุรกิจ e-commerce และคาดการณ์ว่าจะเติบโต **10 เท่า** ภายใน 12 เดือนข้างหน้า เนื่องจากเพิ่งได้รับเงินลงทุนรอบใหม่และวางแผนขยายตลาดไปยัง 3 ประเทศเพิ่มเติม

**สถานะปัจจุบัน (Baseline):**

| Metric | ค่าปัจจุบัน |
|---|---|
| ขนาดฐานข้อมูล | 200 GB |
| Transaction ต่อวัน | 500,000 |
| Peak concurrent connections | 150 |
| Peak read query/วินาที | 800 |
| Peak write query/วินาที | 120 |
| จำนวน order ต่อวัน | 20,000 |
| Server ปัจจุบัน | 8 vCPU, 32GB RAM, SSD 1TB |
| max_connections | 200 |

**เป้าหมาย 12 เดือนข้างหน้า:** เติบโต 10 เท่าในทุกมิติ (แบบ conservative estimate)

### แนวทางการจัดทำแผน (ให้ผู้เรียนลองทำก่อนดูตัวอย่างเฉลย)

ให้ผู้เรียนจัดทำแผน capacity planning โดยตอบคำถามต่อไปนี้ให้ครบถ้วน:

1. คาดการณ์ storage ที่ต้องการในเดือนที่ 3, 6, 9, 12
2. ประเมินว่าควรใช้ Vertical Scaling, Horizontal Scaling (Read Replica), หรือทั้งสองอย่าง
3. คำนวณจำนวน Read Replica ที่ต้องการ
4. ประเมินว่าจะเจอปัญหา Write Scaling หรือไม่ และถ้าเจอ จะแก้อย่างไรในระยะสั้น (ก่อนที่จะพิจารณา sharding เต็มรูปแบบ)
5. คำนวณ `max_connections` ใหม่ และประเมินว่าต้องใช้ connection pooling หรือไม่
6. วางแผนกลยุทธ์ archive ข้อมูลเก่า
7. วางแผน Load Testing ก่อนถึงจุดที่ต้อง scale จริง
8. สรุป timeline การขยายระบบเป็นตาราง 12 เดือน

### ตัวอย่างแนวทางเฉลย (Reference Solution)

**1. คาดการณ์ Storage**

ด้วย exponential growth model ที่ปลายทาง 12 เดือนต้องถึง 10 เท่า (2,000 GB) จากฐาน 200 GB:

```
monthly_growth_rate ที่ทำให้ 200 GB × (1+r)^12 = 2,000 GB
=> (1+r)^12 = 10
=> r ≈ 21.2% ต่อเดือน (compound)
```

```sql
-- คาดการณ์ storage แบบละเอียดทุก 3 เดือน โดยใช้อัตราการเติบโตทบต้นที่คำนวณได้
WITH growth_params AS (
    SELECT
        200.0::numeric AS current_gb,
        0.212::numeric AS monthly_growth_rate
)
SELECT
    n_month,
    round(current_gb * power(1 + monthly_growth_rate, n_month), 1) AS projected_gb
FROM growth_params, generate_series(0, 12, 3) AS n_month
ORDER BY n_month;
```

ผลลัพธ์โดยประมาณ: เดือน 3 ≈ 358 GB, เดือน 6 ≈ 641 GB, เดือน 9 ≈ 1,148 GB, เดือน 12 ≈ 2,000 GB

เผื่อ safety margin 30% และพื้นที่สำหรับ index/WAL/backup: ที่เดือน 12 ควรเตรียม storage อย่างน้อย **~3.5-4 TB**

**2. Vertical vs Horizontal**

- Storage และ CPU: เริ่มจาก **Vertical Scaling** ก่อน (เปลี่ยนเป็น 32 vCPU, 128GB RAM, NVMe 4TB) เพราะ workload ยังไม่ถึงเพดานฮาร์ดแวร์จริง และการเปลี่ยนโครงสร้างแอปพลิเคชันยังไม่จำเป็นในเฟสนี้
- Read scaling: จำเป็นต้องเสริมด้วย **Read Replica** เพราะ read query จาก 800/วินาที จะเพิ่มเป็น 8,000/วินาที ซึ่งเกินกำลังของเครื่องเดียวแม้จะ upgrade แล้ว

**3. คำนวณจำนวน Read Replica**

สมมติ replica แต่ละตัว (spec เดียวกับ primary ที่ upgrade แล้ว) รองรับได้ 3,000 read query/วินาทีก่อน latency แย่ลง:

```
จำนวน Replica = ceil(8,000 / 3,000) = 3 ตัว
บวก buffer สำหรับ failover/maintenance = 4 ตัวทั้งหมด
```

**4. Write Scaling**

Write query จะเพิ่มจาก 120 เป็น 1,200/วินาที ซึ่งยังอยู่ในขอบเขตที่ single Primary node (หลัง upgrade เป็น NVMe + RAM มากขึ้น) น่าจะรองรับได้ หากมีการ optimize query, index ที่เหมาะสม, และ batch write ในจุดที่ทำได้ (เช่น bulk insert แทน insert ทีละแถว) — **ยังไม่จำเป็นต้อง shard ในเฟส 12 เดือนแรก** แต่ควรเริ่มออกแบบ schema ให้รองรับ shard key (เช่น ใช้ `region_id` หรือ `customer_id` เป็นตัวอ้างอิงหลัก) ไว้ล่วงหน้า เผื่ออนาคตต้อง shard จริงเมื่อโตเกิน 10 เท่านี้ไปอีก

**5. Connection Capacity**

Peak connection จะเพิ่มจาก 150 เป็น 1,500 (ตามสัดส่วน 10 เท่า) แต่ด้วยจำนวน CPU ที่เพิ่มเป็น 32 core บนเครื่องใหม่ ค่า `max_connections` ที่เหมาะสมทางฟิสิกส์ยังคงอยู่ราว 100-150 เท่านั้น (ตามสูตรใน Step 748) ดังนั้น **จำเป็นต้องติดตั้ง PgBouncer** (หรือขยาย PgBouncer ที่มีอยู่ให้รองรับโหลดมากขึ้น) เพื่อรองรับ 1,500 connection จาก application โดยส่งต่อ physical connection ไปยัง PostgreSQL เพียง ~150 connection

**6. กลยุทธ์ Archive**

Partition ตาราง `orders`, `order_items`, และ log/audit table ตามเดือน (range partitioning) และตั้ง policy:
- ข้อมูลอายุ 0-6 เดือน: เก็บใน primary NVMe storage
- ข้อมูลอายุ 6-24 เดือน: ย้ายไป tablespace บน storage ที่ถูกกว่า (archive_storage)
- ข้อมูลอายุเกิน 24 เดือน (ตามนโยบายบริษัท/กฎหมาย): export เป็น cold storage แล้ว DROP partition ออกจากฐานข้อมูลหลัก

**7. Load Testing Plan**

ก่อนแต่ละ milestone (เดือนที่ 3, 6, 9, 12) ให้ pgbench หรือ custom benchmark จำลอง traffic ที่ระดับเป้าหมายของเดือนนั้นบน staging environment ที่ spec เดียวกับ production ที่วางแผนไว้ ก่อนจะ migrate จริง เพื่อยืนยันว่าตัวเลขที่คำนวณบนกระดาษตรงกับพฤติกรรมจริงของระบบ

**8. Timeline สรุป**

| เดือน | สิ่งที่ต้องดำเนินการ | เป้าหมาย |
|---|---|---|
| เดือน 1 | เก็บ baseline metric, ตั้งค่า capacity_metrics_history, วางแผน schema สำหรับอนาคต | มีข้อมูลพื้นฐานครบ |
| เดือน 2 | Load test บน staging spec ใหม่ (32 vCPU/128GB), ตั้ง PgBouncer | ยืนยัน spec ใหม่รองรับ 2-3 เท่าได้ |
| เดือน 3 | Vertical scale production เป็น 32 vCPU/128GB/NVMe 4TB | Storage ~360 GB, รองรับ traffic 2 เท่า |
| เดือน 4-5 | เพิ่ม Read Replica ตัวที่ 1-2, ปรับ query ที่หนักให้ไปที่ replica | Read scaling พร้อมสำหรับ 4-5 เท่า |
| เดือน 6 | ตั้ง Partitioning + Archive policy สำหรับตารางหลัก | Storage ~640 GB ควบคุมได้ |
| เดือน 7-8 | เพิ่ม Read Replica ตัวที่ 3-4, Load test ที่ระดับ 6-8 เท่า | รองรับ read query ระดับ 6,000-6,500/วินาที |
| เดือน 9 | ทบทวน write capacity, optimize query/index รอบใหญ่ | Storage ~1,150 GB, write ยังอยู่ในเพดาน |
| เดือน 10-11 | Load test เต็มรูปแบบที่ระดับ 10 เท่า, ซ้อม failover scenario | ยืนยันระบบพร้อมรับ peak จริง |
| เดือน 12 | Go-live เต็มรูปแบบ, monitor ใกล้ชิด, เตรียมแผน sharding สำรองถ้าจำเป็นในอนาคต | Storage ~2,000 GB (+ margin เป็น ~3.5-4TB), ระบบเสถียรที่ 10 เท่า |

---

## สรุปท้ายบท

Capacity Planning เป็นกระบวนการเชิงรุก (proactive) ที่ช่วยให้ทีมงานเตรียมทรัพยากรล่วงหน้าก่อนที่ระบบจะประสบปัญหา แทนที่จะแก้ปัญหาแบบเร่งด่วนหลังเกิดเหตุ หัวใจสำคัญของบทนี้สรุปได้ดังนี้:

1. **Capacity Planning ต้องอาศัยข้อมูลย้อนหลัง** — การเก็บ metric อย่างสม่ำเสมอ (database size, transaction rate, connection count) เป็นรากฐานของการคาดการณ์ที่แม่นยำ
2. **Vertical Scaling เป็นทางเลือกแรกที่ง่ายที่สุด** แต่มีเพดานที่ต้องรู้ล่วงหน้าว่าจะชนเมื่อไหร่
3. **Horizontal Scaling มีความท้าทายเฉพาะตัวของ RDBMS** เพราะ ACID transaction, join, และ referential integrity ข้าม node ทำได้ยากกว่าระบบ NoSQL
4. **Read Scaling ทำได้ง่ายกว่า Write Scaling มาก** เพราะการอ่านไม่เปลี่ยน state แต่ write ต้องรักษาความสอดคล้องของข้อมูลทั่วทั้งระบบ
5. **Sharding เป็นทางออกสุดท้ายสำหรับ Write Scaling** แต่เป็นการตัดสินใจสถาปัตยกรรมที่ย้อนกลับยาก ควรพิจารณาอย่างรอบคอบก่อนตัดสินใจ (รายละเอียดเต็มรออยู่ที่ Part 097 ผ่าน Citus)
6. **Storage Capacity Planning ควบคู่กับ Partitioning และ Archive Strategy** ช่วยให้ระบบไม่จมอยู่กับข้อมูลเก่าที่ไม่จำเป็น
7. **Connection Capacity ต้องคำนวณคู่กับ Connection Pooling** เพราะ `max_connections` ที่สูงเกินไปโดยไม่มี pooling จะทำให้ประสิทธิภาพแย่ลงแทนที่จะดีขึ้น
8. **Load Testing คือขั้นตอนตรวจสอบความจริง** ก่อนจะเชื่อตัวเลขที่คำนวณบนกระดาษ ควรทดสอบด้วย workload ที่ใกล้เคียงจริงที่สุดก่อนถึงวันงานจริงเสมอ

บทถัดไปจะเปลี่ยนแนวทางไปสำรวจโลกของข้อมูลเชิงพื้นที่ (spatial data) ด้วย **PostGIS** ซึ่งเป็น extension ที่ทรงพลังที่สุดตัวหนึ่งของ PostgreSQL สำหรับการทำงานกับข้อมูลแผนที่และพิกัดภูมิศาสตร์

---

## แบบฝึกหัด

<details>
<summary><strong>แบบฝึกหัดที่ 1:</strong> อธิบายความแตกต่างระหว่าง Capacity Planning แบบ Proactive กับ Reactive พร้อมยกตัวอย่างสถานการณ์ที่แสดงผลลัพธ์ต่างกันของทั้งสองแนวทาง</summary>

**เฉลย:**

- **Proactive Capacity Planning** คือการคาดการณ์และเตรียมทรัพยากรล่วงหน้าก่อนที่ปัญหาจะเกิดขึ้นจริง โดยอาศัยการติดตาม trend และ growth rate อย่างต่อเนื่อง
- **Reactive Capacity Planning** คือการแก้ปัญหาหลังจากที่ระบบมีปัญหาแล้ว เช่น รอจนกว่า disk เต็มหรือ connection หมดก่อนถึงจะขยายระบบ

ตัวอย่างสถานการณ์: บริษัท A ติดตาม database growth rate ทุกเดือน พบว่าจะเต็ม storage ใน 2 เดือนข้างหน้า จึงสั่งซื้อ storage เพิ่มล่วงหน้า 1 เดือนก่อนเต็มจริง (Proactive) — เทียบกับบริษัท B ที่ไม่ได้ติดตามอะไรเลย จนกระทั่ง alert แจ้งว่า disk เหลือ 2% ตอนตี 2 ของคืนวันเสาร์ ทำให้ทีม on-call ต้องรีบแก้ไขแบบฉุกเฉิน และระบบอาจ down ระหว่างรอการขยาย storage (Reactive) — ผลลัพธ์ของ Proactive คือไม่มี downtime และวางแผนงบประมาณล่วงหน้าได้ ในขณะที่ Reactive มักนำไปสู่ downtime และการตัดสินใจที่เร่งรีบซึ่งอาจไม่ใช่ทางออกที่ดีที่สุดในระยะยาว

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 2:</strong> เขียน SQL query เพื่อคำนวณ cache hit ratio ของฐานข้อมูลปัจจุบัน และอธิบายว่าค่านี้บอกอะไรเกี่ยวกับความเพียงพอของ RAM</summary>

**เฉลย:**

```sql
SELECT
    sum(blks_hit)  AS total_cache_hits,
    sum(blks_read) AS total_disk_reads,
    round(
        sum(blks_hit)::numeric
        / nullif(sum(blks_hit) + sum(blks_read), 0) * 100, 2
    ) AS cache_hit_ratio_pct
FROM pg_stat_database
WHERE datname NOT IN ('template0', 'template1');
```

ค่า cache hit ratio บอกสัดส่วนของการอ่านข้อมูลที่พบใน shared_buffers (RAM) เทียบกับที่ต้องอ่านจาก disk โดยตรง โดยทั่วไปค่าที่ดีควรอยู่ที่ 99% ขึ้นไปสำหรับระบบ OLTP หากค่านี้ต่ำลงเรื่อยๆ เมื่อข้อมูลโตขึ้น แสดงว่า working set ของข้อมูลเริ่มใหญ่กว่า RAM ที่มี ทำให้ต้องอ่าน disk บ่อยขึ้น ซึ่งเป็นสัญญาณเตือนว่าอาจถึงเวลาต้องเพิ่ม RAM (vertical scaling) หรือพิจารณา partition/archive ข้อมูลเก่าออกเพื่อลด working set

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 3:</strong> ทำไม Read Replica ถึงไม่ช่วยแก้ปัญหาเรื่อง Storage Capacity? อธิบายพร้อมเหตุผลเชิงสถาปัตยกรรม</summary>

**เฉลย:**

Read Replica ทำงานผ่านกลไก Streaming Replication ซึ่งคัดลอกข้อมูลทั้งหมดจาก Primary มาที่ Replica แบบ full copy (replica เป็นสำเนาที่สมบูรณ์ของ primary ณ ทุกช่วงเวลา) ดังนั้นหาก Primary มีขนาดข้อมูล 5TB ทุก Replica ที่เชื่อมต่อก็ต้องมี storage อย่างน้อย 5TB เท่ากันด้วย การเพิ่ม Replica จึงไม่ได้ช่วยลดหรือกระจาย storage requirement เลย ตรงกันข้ามกลับเพิ่ม total storage footprint ของทั้งระบบ (เพราะแต่ละ replica ก็ต้องมี disk เท่า primary) สิ่งที่ Read Replica ช่วยได้คือกระจาย **read query load** เท่านั้น ไม่ใช่กระจาย **storage** การแก้ปัญหา storage capacity ต้องอาศัยกลยุทธ์อื่น เช่น partitioning, archiving ข้อมูลเก่า หรือ sharding ที่แท้จริง (แบ่งข้อมูลไปคนละ node แทนที่จะสำเนาทั้งหมด)

</details>

<details>
<summary><strong>แบบฝึกหัดที่ 4:</strong> กำหนดให้ระบบมี transaction 200,000 ครั้งต่อวันในปัจจุบัน และเติบโตแบบ compound 15% ต่อเดือน จงคำนวณว่าอีก 6 เดือนข้างหน้า transaction ต่อวันจะเป็นเท่าไหร่ พร้อมเขียน SQL คำนวณ