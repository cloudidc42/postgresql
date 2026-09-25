# Part 102: การมีส่วนร่วมใน PostgreSQL Open Source Community

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับโลก/ผู้เชี่ยวชาญ | Part 102

## เป้าหมายการเรียนรู้

เมื่อเรียนจบบทนี้ คุณจะสามารถ:

- อธิบายโครงสร้างการปกครองของ PostgreSQL Global Development Group (PGDG) และเข้าใจว่าทำไม PostgreSQL จึงเป็น "community-driven" ไม่มีบริษัทเดียวเป็นเจ้าของ
- ใช้ mailing list หลักของโครงการ (`pgsql-hackers`, `pgsql-bugs`, `pgsql-general` ฯลฯ) ได้อย่างถูกมารยาทและมีประสิทธิภาพ
- เขียน bug report ที่มีคุณภาพระดับที่ core developer จะรับฟังและตอบสนอง
- เข้าใจขั้นตอนการส่ง patch ผ่านกระบวนการ Commitfest ตั้งแต่ต้นจนจบ
- รู้จักช่องทางเริ่มต้นมีส่วนร่วมที่ไม่ต้องเขียนโค้ด เช่น documentation, การตอบคำถามในชุมชน, การทดสอบ beta
- รู้จักชุมชน PostgreSQL ในประเทศไทยและภูมิภาคเอเชีย รวมถึง PGConf.Asia และประโยชน์เชิงอาชีพจากการมีส่วนร่วม

บทนี้มีเพียง 3 ขั้นตอน (Step 994–996) เพราะเนื้อหาเน้น "การลงมือทำจริง" มากกว่าทฤษฎี — อ่านจบแล้วควรจะสามารถไปเปิด mailing list, หา issue แรก และเริ่มมีส่วนร่วมได้ทันที

---

## Step 994: PostgreSQL Global Development Group (PGDG) — โครงสร้างชุมชนและการพัฒนาแบบ Consensus-Driven

### PostgreSQL ไม่มีเจ้าของ

สิ่งแรกที่ต้องเข้าใจก่อนจะ "มีส่วนร่วม" กับ PostgreSQL คือ PostgreSQL **ไม่มีบริษัทใดบริษัทหนึ่งเป็นเจ้าของ** ต่างจากซอฟต์แวร์โอเพนซอร์สหลายตัวที่มีบริษัทหลักคอยกำกับทิศทาง (เช่น MySQL ที่ Oracle เป็นเจ้าของ หรือ MongoDB ที่บริษัท MongoDB Inc. ควบคุม) PostgreSQL พัฒนาโดย **PostgreSQL Global Development Group (PGDG)** ซึ่งเป็นกลุ่มอาสาสมัครและตัวแทนจากหลายบริษัททั่วโลกที่ทำงานร่วมกันแบบกระจายอำนาจ (decentralized)

ลิขสิทธิ์ของ PostgreSQL อยู่ภายใต้ **PostgreSQL License** ซึ่งเป็น permissive license คล้าย MIT/BSD และถือครองโดย "The PostgreSQL Global Development Group" ไม่ใช่บริษัทใดบริษัทหนึ่ง — ทำให้ไม่มีความเสี่ยงเรื่อง license เปลี่ยนแปลงแบบที่เกิดขึ้นกับโปรเจกต์อื่น (เช่นกรณี MongoDB เปลี่ยนเป็น SSPL หรือ Elastic เปลี่ยนเป็น Elastic License)

### โครงสร้างการปกครอง (Governance Structure)

PostgreSQL มีโครงสร้างการตัดสินใจแบบหลายชั้น ที่ไม่มีใครคนเดียวมีอำนาจเบ็ดเสร็จ:

**1. Core Team (ทีมหลัก)**

Core Team ประกอบด้วยสมาชิกประมาณ 5-7 คน (ปัจจุบันรวมถึงบุคคลอย่าง Bruce Momjian, Tom Lane, Andres Freund, Magnus Hagander, Dave Page, Peter Eisentraut, Jonathan Katz — รายชื่อเปลี่ยนแปลงได้ตามกาลเวลา) ทำหน้าที่:

- ตัดสินใจเรื่องบริหารจัดการที่ไม่เกี่ยวกับเทคนิคโดยตรง เช่น การจัดการ trademark, การอนุมัติ release manager, การไกล่เกลี่ยข้อขัดแย้งเมื่อ committer ไม่สามารถตกลงกันเองได้
- **ไม่ได้** เป็นผู้ตัดสินใจเรื่องทิศทางเทคนิคของโค้ดโดยตรง — เรื่องนั้นเป็นหน้าที่ของ committers และการอภิปรายใน mailing list

**2. Committers**

Committer คือนักพัฒนาที่ได้รับสิทธิ์ push โค้ดเข้า official source tree ปัจจุบันมีประมาณ 40-50 คนทั่วโลก การจะเป็น committer ได้ต้องผ่านการพิสูจน์ตัวเองในชุมชนมาอย่างยาวนาน — ส่ง patch คุณภาพสูงอย่างต่อเนื่อง, ทำ code review ให้คนอื่นอย่างมีคุณภาพ, และได้รับความไว้วางใจจาก committer คนอื่น ๆ ผ่านฉันทามติ (consensus) ไม่มีกระบวนการสมัครแบบเป็นทางการ — เป็นเรื่องของ "ชุมชนสังเกตเห็นแล้วเชิญ"

**3. Contributors**

ใครก็ได้ที่ส่ง patch, รายงานบั๊ก, เขียนเอกสาร, หรือช่วยตอบคำถามในชุมชน — ไม่จำเป็นต้องมีสิทธิ์พิเศษใด ๆ นี่คือจุดเริ่มต้นของทุกคน รวมถึงคนที่ภายหลังกลายเป็น committer ด้วย

### หลักการ Consensus-Driven Development

จุดเด่นสำคัญของ PostgreSQL คือการพัฒนาแบบ **consensus-driven** ทุกการเปลี่ยนแปลงสำคัญต้องผ่านการอภิปรายอย่างเปิดเผยใน mailing list ก่อนเสมอ — ไม่มีทางลัดที่จะ "สั่งจากบนลงล่าง" ได้ แม้แต่ core team หรือ committer ที่มีชื่อเสียงที่สุดก็ต้องอธิบายเหตุผลและตอบข้อโต้แย้งใน thread สาธารณะ

ข้อดีของโมเดลนี้:

- **คุณภาพโค้ดสูง** — ทุก patch ผ่านการ review หลายรอบจากผู้เชี่ยวชาญหลายคนก่อนถูก merge
- **ความเสถียรระยะยาว** — ไม่มีการเปลี่ยนทิศทางกะทันหันเพราะการตัดสินใจของผู้บริหารบริษัทเดียว
- **ความน่าเชื่อถือสำหรับองค์กร** — บริษัทใหญ่ ๆ (เช่น ธนาคาร, รัฐบาล) มั่นใจได้ว่า PostgreSQL จะไม่ถูก "acquire" แล้วเปลี่ยน license หรือปิดฟีเจอร์ไปทำ SaaS แบบปิด
- **ความหลากหลายของมุมมอง** — Contributor มาจากหลายบริษัท (EDB, Microsoft, Amazon, Crunchy Data, Percona, Fujitsu, NTT, VMware/Broadcom ฯลฯ) และอาสาสมัครอิสระ ทำให้ฟีเจอร์ที่ถูกเพิ่มเข้ามาสะท้อนความต้องการที่หลากหลายจริง ๆ ไม่ใช่แค่ของบริษัทเดียว

ข้อเสียที่ต้องยอมรับ:

- **ช้า** — ฟีเจอร์ใหญ่ ๆ อาจใช้เวลาหลายปีกว่าจะถูกรวมเข้า core (เช่น logical replication ใช้เวลาพัฒนานานหลายปีก่อนเข้า PostgreSQL 10, หรือ built-in connection pooling ที่ยังไม่มีใน core จนถึงปัจจุบัน)
- **ต้องอดทนกับการโต้เถียง** — thread ใน pgsql-hackers บาง thread ยาวหลายร้อยอีเมลเพราะความเห็นไม่ตรงกัน
- **มาตรฐานสูงมาก** — patch ที่ถูก reject เพราะไม่ผ่านมาตรฐานคุณภาพเป็นเรื่องปกติ ไม่ใช่เรื่องส่วนตัว

### Mailing List — หัวใจของชุมชน

แม้ยุคนี้จะมี Slack, Discord, GitHub Issues แต่ PostgreSQL ยังคงใช้ **mailing list แบบดั้งเดิม** เป็นช่องทางหลักในการพัฒนา เพราะ:

1. เก็บบันทึกถาวร ค้นหาย้อนหลังได้ผ่าน archive
2. ไม่ผูกกับแพลตฟอร์มใดแพลตฟอร์มหนึ่ง (ไม่ต้องพึ่ง GitHub ซึ่งเป็นของ Microsoft)
3. ทำงานได้ดีกับ workflow แบบ patch-review ที่ใช้ email attachment เป็น diff

Mailing list สำคัญที่ควรรู้จัก (ทั้งหมดสมัครและอ่าน archive ได้ฟรีที่ `https://www.postgresql.org/list/`):

| Mailing List | ใช้สำหรับ |
|---|---|
| **pgsql-hackers** | mailing list หลักสำหรับการพัฒนา core — อภิปรายฟีเจอร์ใหม่, ส่ง/รีวิว patch, ถกเถียงสถาปัตยกรรม นี่คือ "ห้องประชุม" ที่แท้จริงของโครงการ |
| **pgsql-bugs** | รายงานบั๊ก — ทุก bug report ที่ส่งผ่านฟอร์มบนเว็บไซต์จะเข้า list นี้ |
| **pgsql-general** | คำถามทั่วไปเกี่ยวกับการใช้งาน PostgreSQL เหมาะสำหรับผู้ใช้งาน ไม่ใช่ผู้พัฒนา core |
| **pgsql-docs** | อภิปรายและปรับปรุงเอกสารอย่างเป็นทางการ (เหมาะมากสำหรับมือใหม่ที่อยากเริ่มมีส่วนร่วม) |
| **pgsql-novice** | คำถามสำหรับผู้เริ่มต้นใช้งาน PostgreSQL โดยเฉพาะ |
| **pgsql-performance** | ปัญหาด้าน performance และการ tuning |
| **pgsql-admin** | เรื่อง database administration |
| **pgsql-announce** | ประกาศ release ใหม่ ๆ เท่านั้น (low traffic) |

**มารยาทสำคัญของ pgsql-hackers ที่ต้องรู้ก่อนโพสต์:**

- ใช้ **plain text email** ไม่ใช่ HTML — client บางตัว (เช่น Gmail แบบ default) ต้องปรับ settings ก่อน
- **Reply แบบ inline/bottom-posting** ไม่ใช่ top-posting แบบที่คนไทยคุ้นเคยกับอีเมลออฟฟิศ — ตัด quote ส่วนที่ไม่เกี่ยวข้องออก ตอบใต้ประเด็นที่เกี่ยวข้อง
- ส่ง patch เป็น **attachment แบบ `.patch` หรือ `.diff`** ที่สร้างด้วย `git diff` หรือ `git format-patch` ไม่ใช่ paste โค้ดในเนื้ออีเมล และไม่ใช่ลิงก์ GitHub PR เฉย ๆ (แม้จะมี CommitFest app เชื่อม GitHub ได้ แต่การอภิปรายยังคงเกิดบน mailing list)
- ให้เกียรติผู้อื่นเสมอ แม้จะไม่เห็นด้วยอย่างรุนแรง — วัฒนธรรม PostgreSQL ขึ้นชื่อเรื่องความสุภาพเมื่อเทียบกับโครงการโอเพนซอร์สอื่น ๆ และมี Code of Conduct (`https://www.postgresql.org/about/policies/coc/`) บังคับใช้จริงจัง
- อ่าน thread ทั้งหมดก่อนตอบ อย่าถามคำถามที่มีคำตอบอยู่แล้วใน thread เดียวกัน

### Sponsoring Companies และ Non-Profit Foundation

แม้ PostgreSQL จะไม่มีบริษัทเดียวควบคุม แต่ก็มี **PostgreSQL Community Association of Canada (PgCAC)** ทำหน้าที่เป็นนิติบุคคลไม่แสวงหากำไรที่ถือครองทรัพย์สินของโครงการ (เช่น domain, infrastructure ของเว็บไซต์, เงินบริจาคสำหรับจัดงาน conference) แยกออกจากประเด็นเรื่อง governance เชิงเทคนิคโดยสิ้นเชิง เงินทุนส่วนใหญ่มาจากการสปอนเซอร์ของบริษัทที่ใช้ PostgreSQL เป็นธุรกิจหลัก เช่น EDB, Crunchy Data, Percona, Fujitsu, VMware/Broadcom, Amazon, Google, Microsoft — บริษัทเหล่านี้จ้างพนักงานของตัวเองให้ทำงาน full-time เป็น committer หรือ contributor แต่ **การจ้างงานไม่ได้ให้สิทธิพิเศษในการตัดสินใจของโครงการ** คนที่ทำงานให้ EDB หรือ Amazon ยังต้องโน้มน้าวชุมชนด้วยเหตุผลทางเทคนิคเหมือนอาสาสมัครอิสระทุกคน — นี่คือกลไกที่ทำให้ "corporate involvement" กับ "corporate control" แยกออกจากกันอย่างชัดเจน

### Feature Freeze และ Release Cycle

PostgreSQL ออก major version ใหม่ **ปีละ 1 ครั้ง** (เดือนกันยายน-ตุลาคม) ตามปฏิทินที่แน่นอน โดยมีขั้นตอนคร่าว ๆ ดังนี้:

1. **Development phase** (ประมาณเดือนกรกฎาคมปีก่อนหน้า ถึงมีนาคมปีที่ออก) — รับฟีเจอร์ใหม่ผ่าน Commitfest หลายรอบ
2. **Feature Freeze** (ปลายมีนาคม/ต้นเมษายน) — หยุดรับฟีเจอร์ใหม่ เข้าสู่ช่วง bug fixing และ beta testing เท่านั้น
3. **Beta releases** (พฤษภาคม-สิงหาคม) — ออก beta หลายรอบให้ชุมชนช่วยทดสอบ
4. **Release Candidate** (กันยายน)
5. **General Availability (GA)** (กันยายน-ตุลาคม)

การมีปฏิทินที่แน่นอนแบบนี้ทำให้ผู้ใช้งานวางแผน upgrade ได้ล่วงหน้า และทำให้ทุกคนในชุมชนรู้ deadline ที่ชัดเจนสำหรับการส่ง patch

นอกจากนี้แต่ละ major version ยังได้รับการซัพพอร์ต security/bug fix เป็นเวลา **5 ปี** นับจาก release แรก (ออก minor release ทุก ๆ ไตรมาสโดยประมาณ) ซึ่งเป็นอีกหนึ่งจุดแข็งของโมเดล community-driven — นโยบายซัพพอร์ตนี้ประกาศไว้ล่วงหน้าอย่างชัดเจนและไม่เคยถูกเปลี่ยนแปลงเพื่อผลประโยชน์ทางธุรกิจของฝ่ายใดฝ่ายหนึ่ง ต่างจากซอฟต์แวร์เชิงพาณิชย์บางตัวที่อาจ "end-of-life" เวอร์ชันเก่าเร็วขึ้นเพื่อผลักดันให้ลูกค้าอัปเกรดไปใช้ผลิตภัณฑ์ระดับ enterprise ที่มีค่าใช้จ่ายสูงกว่า

| หัวข้อ | นโยบายของ PostgreSQL |
|---|---|
| รอบ release major version | ปีละ 1 ครั้ง (ราวเดือนกันยายน-ตุลาคม) |
| ระยะเวลาซัพพอร์ต | 5 ปีต่อ major version |
| รอบ minor release | ทุกไตรมาส หรือเร็วกว่านั้นเมื่อพบช่องโหว่ความปลอดภัยร้ายแรง |
| ผู้ตัดสินใจ EOL | ประกาศตามตารางที่กำหนดไว้ล่วงหน้า ไม่ผูกกับผลประโยชน์เชิงพาณิชย์ของบริษัทใดบริษัทหนึ่ง |

---

## Step 995: วิธีเริ่มมีส่วนร่วม — จาก Bug Report แรกถึง Patch แรกผ่าน Commitfest

หัวข้อนี้คือหัวใจของบทนี้ — คำแนะนำที่ลงมือทำได้จริงทันที ไม่ใช่ทฤษฎีลอย ๆ

### เส้นทางที่แนะนำสำหรับผู้เริ่มต้น (จากง่ายไปยาก)

```
1. ใช้งาน PostgreSQL อย่างจริงจัง → สังเกตปัญหา
2. อ่าน mailing list / เข้าร่วม community เฉย ๆ ก่อน (lurking)
3. ตอบคำถามคนอื่นใน pgsql-general / Stack Overflow / Thai group
4. รายงานบั๊กที่เจอ (pgsql-bugs)
5. แก้ไขเอกสาร (documentation patch) — จุดเริ่มต้นที่ดีที่สุดสำหรับ patch แรก
6. รีวิว patch ของคนอื่นใน Commitfest
7. ส่ง patch เล็ก ๆ ของตัวเอง (bug fix เล็ก ๆ)
8. ส่ง patch ฟีเจอร์ใหม่ (ใช้เวลาเป็นปี ต้องอดทน)
```

ข้อสำคัญ: **ไม่จำเป็นต้องเขียนโค้ด C เก่ง ๆ ถึงจะมีส่วนร่วมได้** ชุมชนต้องการคนช่วยเรื่องเอกสาร, การแปล, การทดสอบ, การตอบคำถาม พอ ๆ กับต้องการคนเขียนโค้ด

### วิธีเขียน Bug Report ที่ดี

การรายงานบั๊กที่ดีคือ "ของขวัญ" ให้กับ maintainer เพราะมันประหยัดเวลาการสืบสวน bug report ที่แย่มักจะถูกขอข้อมูลเพิ่มแล้วเงียบหายไป ในขณะที่ bug report ที่ดีมักได้รับการตอบและแก้ไขอย่างรวดเร็ว

ส่งผ่านฟอร์มที่ `https://www.postgresql.org/account/submitbug/` (ต้องมี community account ฟรี) ซึ่งจะกระจายเข้า `pgsql-bugs` mailing list โดยอัตโนมัติ

**โครงสร้างของ bug report ที่ดี ต้องมี:**

1. **PostgreSQL version ที่แน่นอน** — ผลลัพธ์จาก `SELECT version();` ไม่ใช่แค่ "PostgreSQL 16"
2. **Operating system และสถาปัตยกรรม** — เช่น Ubuntu 24.04 LTS, x86_64
3. **วิธีการติดตั้ง** — compile เอง, apt/yum package, Docker image ตัวไหน
4. **ขั้นตอนที่ทำให้เกิดปัญหาซ้ำได้ (reproducible steps)** — นี่คือส่วนสำคัญที่สุด ต้องเป็น minimal reproducible example
5. **ผลลัพธ์ที่คาดหวัง (expected) เทียบกับผลลัพธ์จริง (actual)**
6. **Log ที่เกี่ยวข้อง** ถ้ามี error message หรือ crash

**ตัวอย่าง bug report ที่ดี:**

```
Subject: Incorrect result when using GROUP BY with parallel aggregate on partitioned table

PostgreSQL version:
PostgreSQL 16.3 on x86_64-pc-linux-gnu, compiled by gcc (Ubuntu 13.2.0) 13.2.0, 64-bit

Operating system: Ubuntu 24.04 LTS (running in Docker, official postgres:16 image)

Steps to reproduce:

    CREATE TABLE sales (id int, region text, amount numeric)
      PARTITION BY LIST (region);
    CREATE TABLE sales_th PARTITION OF sales FOR VALUES IN ('TH');
    CREATE TABLE sales_sg PARTITION OF sales FOR VALUES IN ('SG');

    INSERT INTO sales SELECT i, CASE WHEN i % 2 = 0 THEN 'TH' ELSE 'SG' END, i * 10
      FROM generate_series(1, 2000000) i;

    SET force_parallel_mode = on;
    SET max_parallel_workers_per_gather = 4;

    SELECT region, sum(amount) FROM sales GROUP BY region;

Expected output:
    region | sum
    -------+------------
    TH     | 10000000000
    SG     | 10000010000

Actual output:
    region | sum
    -------+------------
    TH     | 6234010500
    SG     | 6234020500

The result is consistently wrong when force_parallel_mode = on and the
table has more than 2 partitions with matching partition-wise aggregate
settings. With parallel disabled (SET max_parallel_workers_per_gather = 0)
the result is correct.

I have also attached the full EXPLAIN (ANALYZE, BUFFERS) output and the
postgresql.conf diff from default.
```

สังเกตว่า report นี้ให้ **ขั้นตอนที่ copy-paste รันได้ทันที**, ระบุ version ชัดเจน, เทียบ expected/actual, และบอกด้วยว่าลองปิด parallel แล้วปัญหาหายไป (isolation ปัญหาบางส่วนแล้ว) — นี่คือมาตรฐานที่ทำให้ maintainer อยากช่วยตอบ

**สิ่งที่ไม่ควรทำ:** ส่ง report แบบ "database ของฉันช้ามาก ช่วยด้วย" โดยไม่มี query, ไม่มี schema, ไม่มี version — รายงานแบบนี้จะถูกขอข้อมูลเพิ่มเสมอและมักไม่มีใครตามต่อ

### จุดเริ่มต้นที่ดีที่สุด: Documentation Patch

สำหรับคนที่อยากลองส่ง patch แรกแต่ยังไม่มั่นใจเรื่องโค้ด C การแก้ไขเอกสารคือจุดเริ่มต้นที่ดีที่สุด เพราะ:

- source ของเอกสารเป็น DocBook XML อยู่ใน `doc/src/sgml/` ใน source tree ไม่ต้อง compile ทั้งระบบก็แก้ได้
- ข้อผิดพลาดในเอกสารพบง่าย เช่น คำอธิบายที่คลุมเครือ, ตัวอย่างโค้ดที่ใช้ syntax เก่า, ลิงก์ที่เสีย, การสะกดผิด
- ยังคงผ่านกระบวนการ review เดียวกันกับ code patch ทำให้ได้เรียนรู้ workflow ทั้งหมดโดยความเสี่ยงต่ำ

ขั้นตอน:

```bash
git clone https://git.postgresql.org/git/postgresql.git
cd postgresql
git checkout -b fix-docs-typo

# แก้ไขไฟล์ที่เกี่ยวข้อง เช่น
# doc/src/sgml/ref/create_table.sgml

git diff > fix-docs-typo.patch
```

จากนั้นส่ง patch นี้เข้า `pgsql-hackers` (สำหรับเรื่องใหญ่) หรือ `pgsql-docs` (สำหรับเรื่องเอกสารเล็ก ๆ) พร้อมคำอธิบายว่าแก้อะไร ทำไม

### กระบวนการ Commitfest — วิธีที่ Patch เข้าสู่ Core อย่างละเอียด

**Commitfest คือกลไกหลักที่ PostgreSQL ใช้จัดการ patch review อย่างเป็นระบบ** เว็บไซต์คือ `https://commitfest.postgresql.org/`

ในหนึ่งปีมี Commitfest ประมาณ 4-5 รอบ (กรกฎาคม, กันยายน, พฤศจิกายน, มกราคม, มีนาคม) แต่ละรอบยาวประมาณ 1 เดือน โดยมี "Commitfest Manager" อาสาสมัครทำหน้าที่ประสานงานให้ทุก patch ได้รับการดูแล

**ขั้นตอนแบบละเอียด ตั้งแต่ต้นจนจบ:**

**ขั้นตอนที่ 1 — พัฒนา patch และอภิปรายบน pgsql-hackers**

ก่อนส่งเข้า Commitfest ควรเริ่ม thread ใหม่ใน pgsql-hackers อธิบายปัญหาหรือฟีเจอร์ที่ต้องการเพิ่ม แนบ patch เบื้องต้น (แม้จะยังไม่สมบูรณ์) เพื่อฟังความเห็นก่อน บางครั้งการอภิปรายในขั้นนี้อาจทำให้พบว่าแนวทางที่คิดไว้มีปัญหา ควรเปลี่ยนวิธี — ดีกว่าไปเสียเวลาทำเสร็จแล้วโดนปฏิเสธทั้งยวง

**ขั้นตอนที่ 2 — สมัคร entry ใน Commitfest app**

ไปที่ `commitfest.postgresql.org` → login ด้วย community account → สร้าง entry ใหม่ ใส่ลิงก์ email thread จาก archive (`https://www.postgresql.org/message-id/...`) ที่มี patch แนบอยู่ ระบบจะดึง patch ล่าสุดจาก thread นั้นมาแสดง

**ขั้นตอนที่ 3 — เข้าสู่คิว "Needs Review"**

Patch ใหม่ทุกตัวเริ่มต้นที่สถานะ `Needs Review` — ตอนนี้ใครก็ได้ในชุมชนสามารถอาสามาช่วยรีวิว **นี่คือจุดสำคัญมาก: การรีวิว patch ของคนอื่นคือหนึ่งในวิธีมีส่วนร่วมที่มีคุณค่าที่สุดและขาดแคลนที่สุด** — ชุมชน PostgreSQL มี patch เข้าใหม่มากกว่าจำนวน reviewer ที่มีเสมอ ถ้าคุณอยากช่วยแม้จะยังไม่มั่นใจจะเขียนโค้ดเอง การไปเลือก patch สัก 1-2 ตัวใน Commitfest ปัจจุบันมารีวิว (compile, apply patch, ทดสอบ, ให้ feedback) คือการช่วยเหลือที่มีคุณค่ามากและเป็นก้าวแรกที่ยอดเยี่ยมก่อนจะส่ง patch ของตัวเอง

**ขั้นตอนที่ 4 — วงจร Review ↔ Update**

Reviewer จะ apply patch, ทดสอบ, อ่านโค้ด, แล้วให้ feedback ใน email thread เดิม ผู้เขียน patch แก้ไขตามคำแนะนำแล้วส่งเวอร์ชันใหม่กลับเข้า thread วนซ้ำแบบนี้หลายรอบ — patch ฟีเจอร์ใหญ่บางตัวผ่าน 20-30 รอบก่อนถูกยอมรับ สถานะจะถูกเปลี่ยนเป็น `Waiting on Author` เมื่อรอผู้เขียนแก้ หรือกลับเป็น `Needs Review` เมื่อส่งเวอร์ชันใหม่แล้ว

**ขั้นตอนที่ 5 — Committer เข้ามารับผิดชอบ**

เมื่อ patch ผ่าน community review มาระดับหนึ่งแล้ว committer ที่สนใจหัวข้อนั้นจะเข้ามาอ่านอย่างละเอียดอีกชั้น (มักจะเข้มงวดกว่า reviewer ทั่วไป) นี่คือด่านสุดท้ายก่อนถูก merge

**ขั้นตอนที่ 6 — Commit หรือ Reject/Return**

ถ้าผ่านทุกด่าน committer จะ `git commit` เข้า official repository พร้อมให้เครดิตผู้เขียนและ reviewer ทุกคนใน commit message (PostgreSQL ให้เครดิตอย่างละเอียดมาก — ดู commit message จริงใน git log จะเห็นชื่อคนหลายคนเสมอ) ถ้า patch ไม่ผ่านมาตรฐานหรือไม่มีฉันทามติ อาจถูกทำเครื่องหมายเป็น `Rejected` หรือ `Returned with Feedback` (แปลว่ายังพอมีทางไปต่อได้ในอนาคตถ้าแก้ไขปัญหาที่ชี้ออกมา)

**สถานะที่เป็นไปได้ทั้งหมดใน Commitfest:**

| สถานะ | ความหมาย |
|---|---|
| Needs Review | รอ reviewer เข้ามาดู |
| Waiting on Author | reviewer ให้ feedback แล้ว รอผู้เขียนแก้ |
| Ready for Committer | ผ่าน review แล้ว รอ committer มา commit |
| Committed | เข้า core แล้ว |
| Returned with Feedback | ยังไม่พร้อม ต้องกลับไปคิดใหม่ อาจส่งกลับมาในรอบหน้า |
| Rejected | ไม่ยอมรับ (มักมีเหตุผลทางสถาปัตยกรรมที่ชัดเจน) |
| Withdrawn | ผู้เขียนถอนเอง |

### หา "First Issue" ได้จากที่ไหน

สำหรับคนที่ไม่รู้จะเริ่มจากอะไร มีหลายช่องทางที่ช่วยหา patch/บั๊กที่เหมาะกับมือใหม่:

1. **Commitfest ปัจจุบัน** — เข้าไปดู patch ที่มีสถานะ `Needs Review` และยังไม่มี reviewer คนไหนรับไปดู เลือก patch เล็ก ๆ (diff ไม่กี่สิบบรรทัด) มาลองรีวิวก่อน
2. **PostgreSQL Wiki — "Todo list"** ที่ `wiki.postgresql.org/wiki/Todo` รวบรวมไอเดียฟีเจอร์ที่ชุมชนอยากได้แต่ยังไม่มีใครทำ บางรายการระบุระดับความยาก
3. **Wiki — "First Patch"** ที่ `wiki.postgresql.org/wiki/Developer_FAQ` และหน้าที่เกี่ยวข้อง มีคำแนะนำสำหรับ contributor มือใหม่โดยเฉพาะ
4. **Recent bug reports ใน pgsql-bugs archive** — บั๊กที่รายงานเข้ามาแต่ยังไม่มีใครสืบสวนต่อ บางเคสเป็นแค่ documentation ผิด แก้ได้ไม่ยาก
5. **`git log --grep="Reported-by"`** ในซอร์สโค้ด — ดูว่าบั๊กที่ผ่านมาแก้กันแบบไหน เพื่อเรียนรู้ระดับของ fix ที่ยอมรับได้

### ตัวอย่างการสร้างและส่ง Patch ด้วย git

เมื่อแก้ไขโค้ดหรือเอกสารเสร็จแล้ว ควรสร้าง patch ด้วย `git format-patch` แทน `git diff` ธรรมดา เพราะจะรวม commit message ที่มีคำอธิบายเหตุผลของการแก้ไขไว้ในไฟล์ patch โดยอัตโนมัติ ซึ่งทำให้ reviewer เข้าใจบริบทได้ทันทีโดยไม่ต้องเปิดอ่าน email แยก:

```bash
git checkout -b fix-partition-pruning-doc
# ... แก้ไขไฟล์ ...
git add doc/src/sgml/ddl.sgml
git commit -m "doc: clarify partition pruning behavior with default partitions

The current wording implies pruning always applies to the default
partition, which is not accurate when the partition key involves
expressions. Add a clarifying example."

git format-patch -1 HEAD -o /tmp/patches/
# ได้ไฟล์ /tmp/patches/0001-doc-clarify-partition-pruning-behavior.patch
```

จากนั้นแนบไฟล์ `.patch` นี้เข้าไปในอีเมลที่ส่งเข้า mailing list โดยตรง (ไม่ใช่ paste เนื้อหาลงในตัวอีเมล) — รูปแบบนี้ทำให้ reviewer สามารถ `git am` ไฟล์ patch เข้า local branch ของตัวเองได้ทันทีโดยไม่ต้องแก้ไข format ใด ๆ

### สิ่งที่ต้องเตรียมก่อนเริ่มพัฒนา (Developer Setup)

- สมัคร community account ที่ `postgresql.org` (ใช้บัญชีเดียวกันได้ทั้ง mailing list, Commitfest, bug tracker)
- Clone source code: `git clone https://git.postgresql.org/git/postgresql.git`
- อ่านเอกสาร **"PostgreSQL Coding Conventions"** ใน source tree (`src/tools/pgindent/README` และหมวด Developer's FAQ ใน wiki) — PostgreSQL มีมาตรฐานการจัด format โค้ดที่เข้มงวดมาก ต้องรันผ่าน `pgindent` ก่อนส่ง patch
- เขียน commit message ตามรูปแบบที่ชุมชนใช้ — อธิบาย "ทำไม" ไม่ใช่แค่ "ทำอะไร"
- สร้าง regression test ใหม่คู่กับ patch เสมอ (ใน `src/test/regress/`) — patch ที่ไม่มี test แทบไม่มีทางถูกรับ

### วิธีอื่นที่ไม่ต้องเขียนโค้ดเลยก็มีคุณค่า

- **แปลเอกสารหรือ error message เป็นภาษาไทย** — PostgreSQL รองรับ i18n/l10n ผ่านไฟล์ `.po` มีทีมแปลสำหรับหลายภาษา
- **ทดสอบ beta release** — ทุกครั้งที่มี beta ออกใหม่ (พฤษภาคม-สิงหาคมของทุกปี) ลองเอาไปรันกับ workload จริงของตัวเอง (แบบ non-production) แล้วรายงานปัญหาที่เจอ
- **เขียน blog หรือบทความอธิบายฟีเจอร์ใหม่** — ช่วยกระจายความรู้ในชุมชน แม้ไม่ใช่ contribution ต่อ core โดยตรง แต่มีคุณค่าสูงมาก
- **ตอบคำถามใน pgsql-general, Stack Overflow (tag `postgresql`), DBA Stack Exchange** — การช่วยคนอื่นแก้ปัญหาเป็นการฝึกฝนตัวเองไปในตัว และสร้างชื่อเสียงในวงการ

---

## Step 996: ชุมชน PostgreSQL ในประเทศไทยและภูมิภาค

### PGConf.Asia — งานประชุมหลักของภูมิภาค

**PGConf.Asia** เป็นงานประชุม PostgreSQL ระดับภูมิภาคเอเชียที่ใหญ่ที่สุด จัดขึ้นเป็นประจำทุกปีตั้งแต่ปี 2016 หมุนเวียนจัดในเมืองต่าง ๆ ของเอเชีย (เคยจัดที่โตเกียว, สิงคโปร์, ไทเป, เกียวโต และอื่น ๆ) งานนี้รวมนักพัฒนา core, DBA, และผู้ใช้งานระดับองค์กรจากทั่วภูมิภาคมาแลกเปลี่ยนความรู้

สิ่งที่ได้จากการเข้าร่วม PGConf.Asia:

- **Session เชิงลึกจาก core contributor ตัวจริง** — ได้ฟังคนที่เขียนฟีเจอร์นั้น ๆ อธิบายด้วยตัวเอง ไม่ใช่แค่อ่านเอกสาร
- **Hallway track** — การพูดคุยนอกรอบระหว่าง session มักมีค่ามากกว่าตัว session เอง เพราะได้เจอคนที่แก้ปัญหาคล้ายกับที่เราเจอ
- **โอกาสสร้างเครือข่าย (networking)** กับวิศวกรจากบริษัทเทคโนโลยีใหญ่ ๆ ในภูมิภาคที่ใช้ PostgreSQL เป็น core infrastructure
- **CFP (Call for Papers)** — เปิดโอกาสให้ทุกคนส่งหัวข้อพูดเองได้ ไม่จำเป็นต้องเป็น core developer การพูดในงานระดับภูมิภาคเป็นก้าวสำคัญของอาชีพและสร้างความน่าเชื่อถือ

นอกจาก PGConf.Asia ยังมีงานประชุมระดับโลกอื่น ๆ ที่ควรรู้จัก: **PGConf.EU** (ยุโรป), **PGConf.US**, และ **PostgreSQL Conference Japan** ซึ่งแต่ละงานเปิด CFP และมักมี live stream/recording เผยแพร่ฟรีบน YouTube ภายหลัง สำหรับคนที่ไม่สามารถเดินทางไปร่วมงานได้

### ชุมชน PostgreSQL ในประเทศไทย

ชุมชน PostgreSQL ในไทยเติบโตขึ้นเรื่อย ๆ ตามการเติบโตของอุตสาหกรรมเทคโนโลยีและ fintech ที่นิยมใช้ PostgreSQL เป็น database หลัก ช่องทางที่ควรติดตาม:

- **กลุ่ม PostgreSQL Thailand** — กลุ่มพูดคุยออนไลน์ (Facebook Group / Discord / Meetup.com หรือช่องทางอื่นที่ยังใช้งานอยู่ในช่วงเวลานั้น ๆ) ที่รวมนักพัฒนาและ DBA ชาวไทยไว้พูดคุยแลกเปลี่ยนปัญหาการใช้งานจริง แนะนำให้ค้นหาชื่อกลุ่มปัจจุบันผ่าน search engine เพราะแพลตฟอร์มชุมชนมักเปลี่ยนไปตามยุคสมัย
- **Meetup และ Tech Talk ในกรุงเทพฯ** — บริษัทเทคโนโลยีไทยหลายแห่ง (ธนาคาร, fintech, e-commerce) จัด internal tech talk หรือเปิดให้บุคคลภายนอกเข้าร่วมเป็นครั้งคราว มักประกาศผ่าน Meetup.com หรือ LinkedIn
- **Bangkok DevOps/Database meetup groups** — แม้ไม่ได้เจาะจง PostgreSQL แต่มักมีหัวข้อเกี่ยวกับ PostgreSQL อยู่บ่อยครั้งเพราะเป็น database ยอดนิยมในสาย cloud-native
- **การเข้าร่วมแบบ remote** — ชุมชน PostgreSQL ระดับโลกส่วนใหญ่เปิดกว้างและใช้ภาษาอังกฤษเป็นหลัก ผู้ใช้งานไทยสามารถเข้าร่วม mailing list, Commitfest, หรือแม้แต่ virtual meetup ของภูมิภาคอื่นได้โดยไม่มีข้อจำกัดทางภูมิศาสตร์

| งาน/ช่องทาง | ขอบเขต | ความถี่ | เหมาะกับ |
|---|---|---|---|
| PGConf.Asia | ภูมิภาคเอเชีย | ปีละ 1 ครั้ง | ทุกระดับ ตั้งแต่ผู้ใช้งานถึง core contributor |
| PGConf.EU / PGConf.US | ระดับโลก (แยกทวีป) | ปีละ 1 ครั้งต่องาน | ผู้ที่ต้องการเจาะลึกระดับ core development |
| PostgreSQL Conference Japan | ประเทศญี่ปุ่น | ปีละ 1 ครั้ง | ผู้สนใจ ecosystem ของญี่ปุ่นซึ่งมี contributor จำนวนมาก |
| กลุ่ม/Meetup PostgreSQL Thailand | ประเทศไทย | ไม่แน่นอน ขึ้นกับผู้จัด | ผู้เริ่มต้นและคนที่ต้องการเครือข่ายท้องถิ่น |
| Mailing list / Commitfest | ทั่วโลก แบบ remote | ต่อเนื่องตลอดปี | ทุกระดับ ไม่มีข้อจำกัดด้านภูมิศาสตร์ |

**ข้อแนะนำเชิงปฏิบัติสำหรับผู้อ่านในไทย:** เริ่มจากการค้นหากลุ่มที่ยังเคลื่อนไหวอยู่จริง (active) ผ่าน search engine ด้วยคำว่า "PostgreSQL Thailand community" หรือถามในที่ทำงาน/เครือข่ายสายงาน IT เพราะกลุ่มชุมชนออนไลน์มีการเปลี่ยนแพลตฟอร์มบ่อย (จาก Facebook ไป Discord ไป Slack) การเข้าร่วมชุมชนท้องถิ่นแม้จะเล็กก็มีค่ามาก เพราะสามารถแลกเปลี่ยนปัญหาที่ตรงบริบทกับอุตสาหกรรมไทยได้ เช่น การปฏิบัติตาม PDPA, การเชื่อมต่อกับระบบธนาคารไทย, หรือ latency ระหว่าง region ที่ผู้ให้บริการ cloud มีในไทย

### ประโยชน์ของการมีส่วนร่วมต่อ Career

การมีส่วนร่วมกับ PostgreSQL community ไม่ใช่แค่เรื่อง "การให้กลับคืน" (giving back) แต่เป็นการลงทุนที่ให้ผลตอบแทนเชิงอาชีพที่จับต้องได้จริง:

**1. Portfolio ที่ตรวจสอบได้จริง (Verifiable Track Record)**

Commit history และ mailing list archive ของ PostgreSQL เป็นสาธารณะและถาวร เมื่อสมัครงานตำแหน่ง Senior DBA, Database Engineer หรือ Platform Engineer การชี้ให้เห็นว่า "ผมมี patch ที่ถูก commit เข้า PostgreSQL core" หรือ "ผมช่วย review patch ใน Commitfest มา 3 รอบ" มีน้ำหนักมากกว่าการเขียนใน resume ว่า "เชี่ยวชาญ PostgreSQL" เฉย ๆ เพราะเป็นหลักฐานที่ verify ได้จริงจากบุคคลที่สาม (committer ที่ merge patch)

**2. ความเข้าใจเชิงลึกที่หาไม่ได้จากที่อื่น**

การอ่านและ review patch ของคนอื่นสม่ำเสมอทำให้เข้าใจ internal ของ PostgreSQL ในระดับที่เอกสารทั่วไปไม่มีทางสอนได้ — เข้าใจว่าทำไม optimizer ตัดสินใจแบบนี้ เข้าใจ trade-off เบื้องหลังการออกแบบแต่ละฟีเจอร์ ความรู้ระดับนี้แปลงเป็นความสามารถในการ debug ปัญหา production ที่ซับซ้อนได้เร็วกว่าคนอื่นมาก

**3. เครือข่ายระดับโลก**

คนที่ active ใน mailing list หรือ Commitfest จะรู้จักและเป็นที่รู้จักของ core developer และ committer ทั่วโลก ซึ่งหลายคนทำงานอยู่ในบริษัทชั้นนำ (EDB, Microsoft, AWS, Crunchy Data, Google) เครือข่ายแบบนี้มักนำไปสู่โอกาสงานหรือ collaboration ที่ไม่เคยเปิดประกาศสาธารณะ

**4. เส้นทางสู่การเป็น Recognized Expert**

หลายคนที่เริ่มจากการตอบคำถามใน mailing list หรือรายงานบั๊กเล็ก ๆ ในที่สุดกลายเป็นวิทยากรงานประชุมระดับโลก, ผู้เขียนหนังสือ, หรือแม้แต่ committer เส้นทางนี้ไม่มีทางลัด แต่ทุกก้าวสร้างสะสมชื่อเสียงในวงการที่ยั่งยืนกว่าใบ certificate ทั่วไป

**5. มูลค่าต่อองค์กรที่คุณทำงานอยู่**

วิศวกรที่เข้าใจ internal ของ PostgreSQL ระดับที่ contribute กลับให้โครงการได้ มักเป็นคนที่องค์กรพึ่งพาในการตัดสินใจสถาปัตยกรรมสำคัญ เช่น การเลือก extension, การวางแผน upgrade major version, การ tuning ระดับที่ต้องเข้าใจ source code — ทักษะนี้ทำให้ตำแหน่งงานและค่าตอบแทนสูงขึ้นตามธรรมชาติ

**6. Certification ที่ EDB และผู้ให้บริการอื่นเสนอ**

นอกเหนือจาก contribution โดยตรง ยังมี certification เชิงพาณิชย์ เช่น EDB PostgreSQL certification ที่ช่วยยืนยันความรู้ในเชิง formal สำหรับตลาดงานที่ต้องการใบรับรองประกอบ แม้ certification เหล่านี้ไม่เท่ากับการ contribute จริงในแง่ความลึกของความรู้ แต่เป็นส่วนเสริมที่ดีสำหรับ resume โดยเฉพาะในตลาดองค์กรขนาดใหญ่ที่ต้องการเอกสารยืนยันอย่างเป็นทางการ

---

## คำศัพท์สำคัญประจำบท

| คำศัพท์ | ความหมายโดยย่อ |
|---|---|
| **PGDG** | PostgreSQL Global Development Group — กลุ่มผู้พัฒนาที่กระจายอำนาจ ไม่มีบริษัทเดียวควบคุม |
| **Committer** | ผู้มีสิทธิ์ push โค้ดเข้า official repository ได้รับความไว้วางใจผ่านฉันทามติของชุมชน |
| **Commitfest** | กระบวนการจัดคิว review patch อย่างเป็นระบบ จัดปีละ 4-5 รอบ |
| **pgsql-hackers** | mailing list หลักสำหรับอภิปรายและพัฒนา core ของ PostgreSQL |
| **Reproducible example** | ตัวอย่างขั้นตอนที่ทำให้ผู้อื่น reproduce บั๊กได้ตรงกับที่พบจริง หัวใจของ bug report ที่ดี |
| **PGConf.Asia** | งานประชุม PostgreSQL ระดับภูมิภาคเอเชียที่ใหญ่ที่สุด |
| **Returned with Feedback** | สถานะ patch ที่ยังไม่พร้อมเข้า core แต่มีทางกลับมาแก้ไขเพิ่มเติมได้ในอนาคต |

## สรุปท้ายบท

บทนี้พาไปสำรวจมิติที่มักถูกมองข้ามของการเป็นผู้เชี่ยวชาญ PostgreSQL ระดับโลก นั่นคือ **การมีส่วนร่วมกับชุมชนโอเพนซอร์ส** ซึ่งไม่ใช่แค่เรื่องอุดมการณ์ แต่เป็นเส้นทางที่จับต้องได้จริงสู่ความเชี่ยวชาญระดับสูงสุด

ประเด็นสำคัญที่ควรจำ:

1. **PostgreSQL ปกครองแบบ consensus-driven ผ่าน PGDG** — ไม่มีบริษัทเดียวควบคุม การตัดสินใจทุกอย่างเกิดขึ้นอย่างเปิดเผยบน mailing list โดยเฉพาะ `pgsql-hackers` ซึ่งเป็นศูนย์กลางการพัฒนาโครงการอย่างแท้จริง
2. **จุดเริ่มต้นที่ดีที่สุดไม่ใช่การเขียนโค้ด C** — การรายงานบั๊กที่มีคุณภาพ, การแก้เอกสาร, และการรีวิว patch ของคนอื่นใน Commitfest คือก้าวแรกที่มีคุณค่าและเข้าถึงได้ง่ายกว่าที่คิด
3. **Commitfest คือกลไกหลักในการส่ง patch เข้า core** — ผ่านวงจร Needs Review → Waiting on Author → Ready for Committer → Committed ซึ่งต้องใช้ความอดทนและเปิดรับ feedback
4. **ชุมชนในไทยและภูมิภาคกำลังเติบโต** — PGConf.Asia เป็นเวทีหลักของภูมิภาค และการเข้าร่วมชุมชนท้องถิ่นแม้เล็กก็มีค่าต่อการแลกเปลี่ยนความรู้ที่ตรงบริบท
5. **การมีส่วนร่วมคือการลงทุนเชิงอาชีพ** — สร้าง track record ที่ verify ได้จริง, เครือข่ายระดับโลก, และความเข้าใจเชิงลึกที่หาที่อื่นไม่ได้

การเดินทางสู่การเป็น "world-class PostgreSQL expert" ไม่ได้จบแค่การรู้เทคนิคเชิงลึก แต่รวมถึงการเป็นส่วนหนึ่งของชุมชนที่สร้างเทคโนโลยีนี้ขึ้นมา — บทถัดไปจะพาไปมองไปข้างหน้าสู่อนาคตของ PostgreSQL ว่าโครงการนี้กำลังมุ่งไปทางไหน

**บทต่อไป:** [Part 103: อนาคตของ PostgreSQL](./part-103-postgresql-future.md)

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: ทำความรู้จักโครงสร้างการปกครอง

จงอธิบายความแตกต่างระหว่าง Core Team, Committer และ Contributor ของ PostgreSQL และอธิบายว่าทำไมการที่ไม่มีบริษัทเดียวควบคุมโครงการจึงเป็นข้อดีสำหรับองค์กรที่นำ PostgreSQL ไปใช้งานจริง

<details>
<summary>เฉลย</summary>

- **Core Team** (~5-7 คน) ดูแลเรื่องบริหารจัดการที่ไม่ใช่เทคนิคโดยตรง เช่น trademark, การแต่งตั้ง release manager, การไกล่เกลี่ยข้อขัดแย้ง — ไม่ได้ตัดสินใจทิศทางเทคนิคของโค้ดโดยตรง
- **Committer** (~40-50 คน) มีสิทธิ์ push โค้ดเข้า official repository ได้จากการพิสูจน์ตัวเองผ่าน patch คุณภาพสูงและการ review ที่ดีอย่างต่อเนื่อง ได้รับความไว้วางใจผ่านฉันทามติ ไม่ใช่การแต่งตั้งจากบนลงล่าง
- **Contributor** คือใครก็ได้ที่ส่ง patch, รายงานบั๊ก, เขียนเอกสาร หรือช่วยตอบคำถาม — จุดเริ่มต้นของทุกคน

ข้อดีต่อองค์กร: ไม่มีความเสี่ยงเรื่องบริษัทเดียวถูก acquire แล้วเปลี่ยน license (แบบที่เกิดกับ MongoDB/Elastic), การตัดสินใจสถาปัตยกรรมผ่านการอภิปรายแบบเปิดเผยทำให้ฟีเจอร์สะท้อนความต้องการหลากหลาย ไม่ใช่ผลประโยชน์ของบริษัทเดียว และความเสถียรระยะยาวเพราะไม่มีการเปลี่ยนทิศทางกะทันหันจากการตัดสินใจของผู้บริหารคนเดียว

</details>

### แบบฝึกหัดที่ 2: เลือก Mailing List ให้ถูกต้อง

สถานการณ์ต่อไปนี้ควรส่งไปที่ mailing list ไหน: (ก) พบว่า query planner เลือกแผนที่ผิดอย่างสม่ำเสมอเมื่อใช้ partial index ร่วมกับ CTE (ข) มีคำถามว่าจะ config `shared_buffers` อย่างไรให้เหมาะกับ server 64GB RAM (ค) พบคำอธิบายฟังก์ชัน `pg_stat_statements` ในเอกสารทางการที่ไม่ตรงกับพฤติกรรมจริง

<details>
<summary>เฉลย</summary>

(ก) **pgsql-bugs** — เป็นการรายงานพฤติกรรมที่ผิดปกติของ planner ถือเป็นบั๊ก ควรแนบ minimal reproducible example

(ข) **pgsql-general** หรือ **pgsql-performance** — เป็นคำถามเรื่องการใช้งาน/tuning ไม่ใช่การพัฒนา core

(ค) **pgsql-docs** — เป็นเรื่องความไม่ตรงกันของเอกสาร เหมาะกับ list ที่เจาะจงเรื่องเอกสารโดยตรง แม้จะส่งเข้า pgsql-hackers ก็ยอมรับได้เช่นกันถ้าเกี่ยวข้องกับพฤติกรรมของโค้ดด้วย

</details>

### แบบฝึกหัดที่ 3: วิจารณ์ Bug Report

Bug report ต่อไปนี้มีปัญหาอะไรบ้าง และควรแก้ไขอย่างไรให้มีคุณภาพขึ้น:

> "Subject: postgres ช้ามาก
> ผม query ตารางใหญ่แล้วมันช้ามาก ช่วยดูให้หน่อยครับ ใช้ PostgreSQL"

<details>
<summary>เฉลย</summary>

ปัญหาของ report นี้:

1. ไม่ระบุ PostgreSQL version ที่แน่นอน (ควรใช้ผลจาก `SELECT version();`)
2. ไม่ระบุ operating system หรือวิธีการติดตั้ง
3. ไม่มี query ที่ใช้จริง ไม่มี schema ของตาราง
4. ไม่มีตัวเลข "ช้า" หมายถึงกี่วินาที เทียบกับที่คาดหวังกี่วินาที
5. ไม่มี `EXPLAIN (ANALYZE, BUFFERS)` output
6. "ช้า" อาจไม่ใช่บั๊กเลย อาจเป็นเรื่อง performance tuning ซึ่งควรไปที่ pgsql-performance หรือ pgsql-general ไม่ใช่ pgsql-bugs

วิธีแก้ไข: ระบุ version, OS, ให้ schema (`\d tablename`), query จริง (หรือตัวอย่างย่อที่ทำให้ปัญหาเกิดซ้ำได้), ผลจาก `EXPLAIN (ANALYZE, BUFFERS)`, ค่า config ที่เกี่ยวข้อง (เช่น `work_mem`, `shared_buffers`), และตัวเลข "ช้า" ที่ชัดเจนเทียบกับสิ่งที่คาดหวัง

</details>

### แบบฝึกหัดที่ 4: ลำดับสถานะ Commitfest

จงเรียงลำดับสถานะของ patch ใน Commitfest ตั้งแต่เริ่มส่งจนถูก merge เข้า core โดยสมมติว่า patch ผ่านการ review หนึ่งรอบแล้วผู้เขียนต้องแก้ไขก่อนจะผ่าน

<details>
<summary>เฉลย</summary>

`Needs Review` → (reviewer ให้ feedback) → `Waiting on Author` → (ผู้เขียนแก้ไขและส่งเวอร์ชันใหม่) → `Needs Review` อีกครั้ง → (ผ่าน review แล้ว) → `Ready for Committer` → (committer ตรวจสอบและ merge) → `Committed`

หมายเหตุ: วงจร Needs Review ↔ Waiting on Author อาจเกิดขึ้นหลายรอบก่อนจะไปถึง Ready for Committer โดยเฉพาะ patch ฟีเจอร์ใหญ่

</details>

### แบบฝึกหัดที่ 5: เลือก First Contribution ที่เหมาะกับตัวเอง

สมมติคุณเป็นนักพัฒนาที่ใช้ PostgreSQL มา 2 ปี เขียน SQL คล่อง แต่ไม่เคยเขียนโค้ด C มาก่อน จงเสนอแผน 3 ขั้นตอนที่เหมาะสมสำหรับการเริ่มมีส่วนร่วมกับชุมชน โดยไม่ต้องเริ่มจากการเขียนโค้ด C ทันที

<details>
<summary>เฉลย</summary>

ตัวอย่างแผนที่เหมาะสม:

1. **เดือนที่ 1-2**: สมัคร community account, subscribe pgsql-general และ pgsql-docs, อ่าน archive ย้อนหลังเพื่อทำความเข้าใจวัฒนธรรมและมารยาทของ list เริ่มตอบคำถามง่าย ๆ ใน pgsql-general ที่ตรงกับความเชี่ยวชาญ SQL ของตัวเอง
2. **เดือนที่ 3-4**: เริ่มอ่านเอกสารทางการอย่างละเอียด (`doc/src/sgml/`) หาจุดที่คำอธิบายคลุมเครือหรือตัวอย่างที่ล้าสมัย ส่ง documentation patch แรกเข้า pgsql-docs หรือ pgsql-hackers พร้อมคำอธิบายที่ชัดเจน
3. **เดือนที่ 5-6**: เข้าไปดู Commitfest ปัจจุบัน เลือก patch เล็ก ๆ (diff ไม่กี่สิบบรรทัด) ที่ยังไม่มี reviewer มารับ ลอง apply patch ทดสอบด้วยตัวเอง (ใช้แค่ SQL และการทดสอบพฤติกรรม ไม่ต้องอ่านโค้ด C ลึกก็ช่วยตรวจ behavior ได้) แล้วให้ feedback เป็นลายลักษณ์อักษรใน thread — นี่คือ contribution ที่มีค่าจริงแม้ไม่ได้เขียนโค้ด C เอง

หลังจากสะสมประสบการณ์และความมั่นใจจากขั้นตอนเหล่านี้ ค่อยพิจารณาศึกษา C และลองแก้บั๊กเล็ก ๆ ในโค้ดจริงต่อไป

</details>

### แบบฝึกหัดที่ 6: หา Open Issue จริงและวางแผนลงมือทำ

ให้ไปที่ `commitfest.postgresql.org` (หรือ wiki Todo list ถ้าเข้าถึงไม่ได้) มองหา patch หรือหัวข้อที่มีสถานะ "Needs Review" และยังไม่มี reviewer จดไว้ 1 รายการ แล้วเขียนแผนสั้น ๆ ว่าจะเริ่มรีวิวอย่างไร (จะ clone code ที่ไหน, จะ apply patch ยังไง, จะทดสอบอะไรบ้าง)

<details>
<summary>เฉลย</summary>

ตัวอย่างแผน (คำตอบจะแตกต่างกันไปตาม patch ที่เลือกจริง):

1. เข้าไปที่ entry ของ patch ใน Commitfest app, เปิด email thread ต้นทางเพื่ออ่านบริบทว่าปัญหาคืออะไร ทำไมถึงเสนอ patch นี้
2. `git clone https://git.postgresql.org/git/postgresql.git` แล้ว checkout branch ที่ตรงกับเวอร์ชัน development ปัจจุบัน
3. ดาวน์โหลด patch ล่าสุดจาก thread แล้ว `git apply patch-file.patch` หรือใช้ `patch -p1 < patch-file.patch`
4. `./configure && make && make install` เพื่อ build เวอร์ชันที่มี patch
5. รัน regression test ที่มากับ patch (ถ้ามี) ด้วย `make check`
6. ทดสอบเพิ่มเติมด้วยตัวเอง — ลอง edge case ที่ผู้เขียน patch อาจไม่ได้คิดถึง เช่น ค่า NULL, ตารางว่าง, ข้อมูลขนาดใหญ่
7. เขียน feedback กลับเข้า email thread เดิม (reply-all แบบ inline) บอกว่า apply ได้หรือไม่, test ผ่านหรือไม่, มีข้อสังเกตอะไรเกี่ยวกับโค้ดหรือพฤติกรรมบ้าง
8. อัปเดตสถานะใน Commitfest app ว่าตัวเองเป็น reviewer ของ patch นี้แล้ว

ข้อสำคัญ: ไม่จำเป็นต้องเข้าใจโค้ด C ทั้งหมดเพื่อเริ่มรีวิว การทดสอบพฤติกรรมและ regression test ก็มีค่ามากแล้ว

</details>

### แบบฝึกหัดที่ 7: วิเคราะห์ Trade-off ของ Consensus-Driven Development

จงวิเคราะห์ข้อดีและข้อเสียของการพัฒนาแบบ consensus-driven เปรียบเทียบกับโมเดลที่บริษัทเดียวควบคุมทิศทาง (เช่น MongoDB) โดยยกตัวอย่างสถานการณ์ที่แต่ละโมเดลจะได้เปรียบ

<details>
<summary>เฉลย</summary>

**ข้อดีของ consensus-driven (PostgreSQL):**
- คุณภาพโค้ดสูงเพราะผ่านการ review หลายชั้น
- ไม่มีความเสี่ยงเรื่อง license เปลี่ยนหรือฟีเจอร์ถูกดึงไปทำ SaaS ปิด
- ความเสถียรระยะยาว เหมาะกับองค์กรที่ต้องวางแผน 10-20 ปีข้างหน้า (ธนาคาร, รัฐบาล)
- ฟีเจอร์สะท้อนความต้องการหลากหลายจากหลายอุตสาหกรรม

**ข้อเสียของ consensus-driven:**
- ช้า — ฟีเจอร์ใหญ่ใช้เวลาหลายปี (เช่น built-in connection pooling ที่ core ยังไม่มี)
- ตอบสนอง trend ตลาดช้ากว่าบริษัทเดียวที่ตัดสินใจเองได้ทันที

**ข้อดีของโมเดลบริษัทเดียวควบคุม:**
- ตัดสินใจเร็ว ออกฟีเจอร์ตอบโจทย์ตลาดได้ทันเวลา (เช่น MongoDB ตอบสนอง trend NoSQL/document DB ได้เร็ว)
- มี roadmap ชัดเจนจากบริษัทเดียว ไม่ต้องรอฉันทามติ

**ข้อเสียของโมเดลบริษัทเดียวควบคุม:**
- ความเสี่ยงเรื่อง license เปลี่ยน (SSPL ของ MongoDB, Elastic License) กระทบผู้ใช้งานที่พึ่งพา open source license เดิม
- ทิศทางอาจเปลี่ยนกะทันหันตามผลประโยชน์ทางธุรกิจของบริษัทเดียว

สรุป: PostgreSQL เหมาะกับงานที่ต้องการความเสถียรระยะยาวและความน่าเชื่อถือสูงสุด ส่วนโมเดลบริษัทเดียวเหมาะกับตลาดที่ต้องการนวัตกรรมเร็วและยอมรับความเสี่ยงด้าน license ได้มากกว่า

</details>

### แบบฝึกหัดที่ 8: วางแผนเข้าร่วม PGConf.Asia

สมมติ PGConf.Asia ปีหน้าจะจัดในเมืองที่คุณสามารถเดินทางไปได้ จงวางแผนการเตรียมตัว 3 เดือนก่อนงาน ครอบคลุมทั้งการเตรียมความรู้และการใช้ประโยชน์จากงานให้คุ้มค่าที่สุด

<details>
<summary>เฉลย</summary>

ตัวอย่างแผน:

**เดือนที่ 1 (3 เดือนก่อนงาน):**
- ตรวจสอบ agenda และรายชื่อ speaker ที่ประกาศ เลือก session ที่ตรงกับงานปัจจุบันหรือปัญหาที่กำลังเจอ
- ถ้าสนใจพูด ให้เตรียมส่ง CFP (Call for Papers) ถ้ายังเปิดรับ — เลือกหัวข้อจากประสบการณ์จริงในการทำงาน เช่น การ migrate จาก database อื่นมา PostgreSQL, หรือ case study การ tuning

**เดือนที่ 2:**
- อ่าน release notes ของเวอร์ชันล่าสุดและ major feature ที่กำลังจะออก เพื่อให้เข้าใจบริบทของ session ต่าง ๆ
- เตรียมคำถามที่อยากถามในแต่ละ session ล่วงหน้า
- ติดต่อเพื่อนร่วมงานหรือคนในวงการที่จะไปงานเดียวกัน นัดเจอกันในงาน (hallway track มักมีค่ามากกว่า session)

**เดือนที่ 3 (ก่อนงาน):**
- เตรียม business card หรือ LinkedIn QR code สำหรับสร้างเครือข่าย
- ทบทวนปัญหาที่กำลังเจอในงานจริง เพื่อเอาไปถาม core developer หรือผู้เชี่ยวชาญในงานโดยตรง
- วางแผนเขียนสรุปสิ่งที่ได้เรียนรู้หลังงานจบ (blog post หรือแชร์ในทีม) เพื่อขยายคุณค่าของการเข้าร่วมให้คนอื่นในองค์กรได้ประโยชน์ด้วย

</details>

### แบบฝึกหัดที่ 9: ออกแบบ Contribution Roadmap ส่วนตัว 1 ปี

จงออกแบบแผนการมีส่วนร่วมกับ PostgreSQL community ของตัวเองในระยะเวลา 1 ปี โดยระบุเป้าหมายที่วัดผลได้ในแต่ละไตรมาส

<details>
<summary>เฉลย</summary>

ตัวอย่าง roadmap (ปรับตามบริบทของแต่ละคน):

**Q1:** สมัคร community account, subscribe mailing list ที่เกี่ยวข้อง, ตอบคำถามใน pgsql-general อย่างน้อย 5 ครั้ง, อ่าน Commitfest archive เพื่อทำความเข้าใจ workflow

**Q2:** ส่ง bug report คุณภาพสูงอย่างน้อย 1 ฉบับ (พร้อม reproducible example), ส่ง documentation patch แรก, เริ่มรีวิว patch ของคนอื่นใน Commitfest อย่างน้อย 2 ตัว

**Q3:** ส่ง patch bug fix เล็ก ๆ ของตัวเองเข้า Commitfest, เข้าร่วม PGConf.Asia หรือ local meetup อย่างน้อย 1 งาน, เขียนบทความสรุปสิ่งที่เรียนรู้

**Q4:** ติดตาม patch ของตัวเองผ่านกระบวนการ review จนจบ (ไม่ว่าจะ committed หรือ returned with feedback), ตั้งเป้าหมาย contribution ปีถัดไปโดยอิงจากประสบการณ์ที่สะสมมา เช่น เริ่มศึกษาโค้ด C เพื่อแก้บั๊กที่ซับซ้อนขึ้น

**ตัวชี้วัดรวมทั้งปี:** จำนวนอีเมลที่ตอบในชุมชน, จำนวน patch ที่ส่ง/รีวิว, จำนวนงานที่เข้าร่วม, และเครือข่ายใหม่ที่สร้างขึ้น

</details>

### แบบฝึกหัดที่ 10: สะท้อนคุณค่าต่ออาชีพของตัวเอง

จงเขียนย่อหน้าสั้น ๆ (3-5 ประโยค) อธิบายว่าการมีส่วนร่วมกับ PostgreSQL community จะช่วยเป้าหมายอาชีพของคุณอย่างไรโดยเฉพาะเจาะจง (ไม่ใช่คำตอบทั่วไป) — ระบุตำแหน่งงานเป้าหมาย ทักษะที่ต้องการพัฒนา และ contribution แบบไหนที่จะช่วยตอบโจทย์นั้นได้ดีที่สุด

<details>
<summary>เฉลย</summary>

ไม่มีคำตอบตายตัว เพราะขึ้นกับเป้าหมายอาชีพของแต่ละคน แต่คำตอบที่ดีควรมีองค์ประกอบ:

1. **ตำแหน่งเป้าหมายที่ชัดเจน** เช่น "Senior Database Engineer ที่ดูแล PostgreSQL cluster ขนาดใหญ่ระดับ petabyte" หรือ "Database consultant อิสระ"
2. **ทักษะเฉพาะที่ต้องการ** เช่น ความเข้าใจ query optimizer, ความสามารถ debug production incident ได้เร็ว, หรือความน่าเชื่อถือในการให้คำแนะนำสถาปัตยกรรม
3. **Contribution ที่ตรงจุด** เช่น ถ้าต้องการเป็นผู้เชี่ยวชาญด้าน performance การรีวิว patch ที่เกี่ยวกับ query planner ใน Commitfest จะช่วยได้ตรงกว่าการแก้ documentation ทั่วไป หรือถ้าต้องการเป็น consultant การพูดในงาน PGConf.Asia และเขียนบทความจะช่วยสร้างชื่อเสียงสาธารณะที่จำเป็นต่อสายอาชีพนี้โดยตรง

ตัวอย่างคำตอบ: "ผมต้องการเป็น Database Reliability Engineer ที่บริษัท fintech ซึ่งต้องเข้าใจ internal ของ PostgreSQL ระดับลึกเพื่อ debug ปัญหา production ที่ซับซ้อน การเริ่มรีวิว patch ใน Commitfest ที่เกี่ยวกับ WAL และ replication จะช่วยให้ผมเข้าใจกลไกเหล่านี้ในระดับที่เอกสารทั่วไปสอนไม่ได้ และ track record การ contribute จะเป็นหลักฐานที่ verify ได้จริงเมื่อสมัครงานตำแหน่งนี้ในอนาคต"

</details>

---

**บทต่อไป:** [Part 103: อนาคตของ PostgreSQL](./part-103-postgresql-future.md)
