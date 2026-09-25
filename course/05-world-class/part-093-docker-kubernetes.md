# Part 093: PostgreSQL บน Docker และ Kubernetes (Operators: Zalando, CloudNativePG)

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับโลก/ผู้เชี่ยวชาญ | Part 093

---

## เป้าหมายการเรียนรู้

เมื่อจบบทนี้ ผู้เรียนจะสามารถ:

1. รัน PostgreSQL บน Docker ได้อย่างถูกต้อง พร้อมจัดการ persistent volume ให้ข้อมูลไม่หายเมื่อ container restart หรือถูกลบ
2. ออกแบบ Docker Compose stack แบบเต็มรูปแบบที่มี PostgreSQL + PgBouncer + Redis + application เชื่อมต่อกันอย่างปลอดภัย
3. อธิบายได้ว่าทำไมการรัน stateful database อย่าง PostgreSQL บน Kubernetes จึงซับซ้อนกว่า stateless application ทั่วไป
4. เข้าใจและเขียน StatefulSet + PersistentVolumeClaim พื้นฐานสำหรับ PostgreSQL บน Kubernetes
5. อธิบาย Kubernetes Operator Pattern และบทบาทของ Custom Resource Definition (CRD) + Controller
6. อ่านและเขียน manifest ของ Zalando Postgres Operator (`postgresql` CRD) ได้
7. อ่านและเขียน manifest ของ CloudNativePG (CNPG) (`Cluster` CRD) ได้
8. เปรียบเทียบข้อดีข้อเสียของ Zalando Operator กับ CloudNativePG เพื่อเลือกใช้งานให้เหมาะกับองค์กร
9. เข้าใจว่า Operator จัดการ High Availability, failover, และ automated backup บน Kubernetes อย่างไร
10. ออกแบบและเขียน manifest ระดับ production สำหรับระบบ e-commerce ที่ใช้ PostgreSQL บน Kubernetes ด้วย CloudNativePG

> **หมายเหตุสำคัญเกี่ยวกับตัวอย่างในบทนี้**
> ตัวอย่าง **Docker / Docker Compose** ทั้งหมดในบทนี้เป็นโค้ดที่ **รันได้จริง** (runnable) บนเครื่องที่มี Docker และ Docker Compose ติดตั้งอยู่ ผู้เรียนสามารถคัดลอกไปรันทดสอบได้ทันที
>
> ตัวอย่าง **Kubernetes manifest** (รวมถึง Operator ต่าง ๆ) เป็น **ตัวอย่างอ้างอิงเชิงสถาปัตยกรรม** (architectural / conceptual reference) ที่เขียนตาม syntax และ field จริงของแต่ละ CRD ในเวอร์ชันที่นิยมใช้งาน ณ ปัจจุบัน เพื่อให้ผู้เรียนนำไปปรับใช้กับ cluster จริงได้ แต่ **ไม่ได้ถูกรันทดสอบจริงในสภาพแวดล้อมของบทเรียนนี้** เนื่องจากไม่มี Kubernetes cluster ให้ใช้งาน ผู้เรียนควรทดสอบบน cluster จริง (เช่น `kind`, `minikube`, GKE, EKS, AKS) ก่อนนำไปใช้งานจริงบน production เสมอ

---

## Step 921: ทบทวน PostgreSQL บน Docker เจาะลึก — Persistent Volume และการจัดการข้อมูลไม่ให้หาย

ใน Part 002 เราได้แนะนำการรัน PostgreSQL บน Docker แบบเบื้องต้นไปแล้ว ในบทนี้เราจะเจาะลึกในประเด็นที่สำคัญที่สุดสำหรับการใช้งานจริง นั่นคือ **การจัดการข้อมูลให้คงอยู่ (data persistence)**

### ปัญหาพื้นฐาน: Container คือของชั่วคราว (ephemeral)

Container ถูกออกแบบมาให้เป็น ephemeral โดยธรรมชาติ — เมื่อ container ถูกลบ (`docker rm`) ข้อมูลทั้งหมดที่เขียนอยู่ใน writable layer ของ container จะหายไปทันที หาก PostgreSQL เก็บข้อมูลไว้ใน layer นี้โดยไม่มีการ mount volume ใด ๆ การ restart หรือ recreate container แม้เพียงครั้งเดียวก็อาจทำให้ข้อมูลทั้งฐานหายไปอย่างถาวร

Image `postgres` อย่างเป็นทางการบน Docker Hub กำหนด environment variable `PGDATA` ไว้ที่ `/var/lib/postgresql/data` และประกาศ `VOLUME` ไว้ที่ตำแหน่งนี้ในทุก layer ของ Dockerfile แต่การประกาศ `VOLUME` ใน Dockerfile เพียงอย่างเดียว **ไม่ได้แปลว่าข้อมูลจะปลอดภัย** — ถ้าเราไม่ระบุ named volume หรือ bind mount เอง Docker จะสร้าง **anonymous volume** ให้อัตโนมัติ ซึ่งยังคงอยู่หลัง container ถูกลบก็จริง แต่จะหาชื่อ อ้างอิง หรือ backup ได้ยากมาก เพราะชื่อเป็น hash แบบสุ่ม

### แนวทางที่ถูกต้อง: Named Volume

```bash
# สร้าง named volume แยกต่างหาก
docker volume create pgdata_prod

# รัน PostgreSQL container โดย mount named volume เข้ากับ PGDATA
docker run -d \
  --name pg-primary \
  --restart unless-stopped \
  -e POSTGRES_PASSWORD=SuperSecretPass123 \
  -e POSTGRES_USER=appuser \
  -e POSTGRES_DB=appdb \
  -e PGDATA=/var/lib/postgresql/data/pgdata \
  -v pgdata_prod:/var/lib/postgresql/data \
  -p 5432:5432 \
  --shm-size=256mb \
  postgres:16.4
```

ประเด็นสำคัญที่ต้องอธิบาย:

1. **`-v pgdata_prod:/var/lib/postgresql/data`** — mount named volume `pgdata_prod` เข้ากับ directory หลักที่ PostgreSQL เก็บ data files, WAL, และ configuration ทั้งหมด ตราบใดที่ volume นี้ไม่ถูกลบ (`docker volume rm`) ข้อมูลจะอยู่รอดไม่ว่า container จะถูก stop, start, restart หรือแม้กระทั่ง `docker rm` แล้วสร้างใหม่
2. **`PGDATA=/var/lib/postgresql/data/pgdata`** — เป็นแนวปฏิบัติที่แนะนำจาก official image เพื่อให้ PostgreSQL เก็บข้อมูลไว้ใน subdirectory ของ mount point แทนที่จะเป็น mount point เอง เนื่องจากบาง filesystem จะสร้าง hidden directory `lost+found` ไว้ที่ root ของ mount ซึ่งจะรบกวนการทำงานของ `initdb` (initdb ต้องการ directory ที่ว่างเปล่าสนิท)
3. **`--shm-size=256mb`** — PostgreSQL ใช้ shared memory (`/dev/shm`) สำหรับ parallel query และ operation อื่น ๆ ค่า default ของ Docker คือ 64MB ซึ่งน้อยเกินไปสำหรับ workload จริง ควรปรับเพิ่มตามขนาด `work_mem` และจำนวน worker ที่คาดว่าจะใช้
4. **`--restart unless-stopped`** — ทำให้ container กลับมาทำงานอัตโนมัติเมื่อ Docker daemon หรือเครื่อง restart โดยไม่ต้องมีคนสั่ง start เอง (ยกเว้นถ้าเคยสั่ง `docker stop` ไว้ก่อนหน้า)

### ทดสอบว่าข้อมูลไม่หายจริง

```bash
# สร้างข้อมูลทดสอบ
docker exec -it pg-primary psql -U appuser -d appdb -c \
  "CREATE TABLE proof_of_life (id serial PRIMARY KEY, note text, created_at timestamptz default now());"
docker exec -it pg-primary psql -U appuser -d appdb -c \
  "INSERT INTO proof_of_life (note) VALUES ('ข้อมูลนี้ต้องไม่หาย');"

# ลบ container ทิ้งทั้งหมด (ไม่ใช่แค่ stop)
docker rm -f pg-primary

# สร้าง container ใหม่โดยอ้างอิง volume เดิม
docker run -d \
  --name pg-primary \
  -e POSTGRES_PASSWORD=SuperSecretPass123 \
  -e POSTGRES_USER=appuser \
  -e POSTGRES_DB=appdb \
  -e PGDATA=/var/lib/postgresql/data/pgdata \
  -v pgdata_prod:/var/lib/postgresql/data \
  -p 5432:5432 \
  postgres:16.4

# รอ container เริ่มทำงาน แล้วตรวจสอบข้อมูล
sleep 5
docker exec -it pg-primary psql -U appuser -d appdb -c "SELECT * FROM proof_of_life;"
```

ผลลัพธ์ที่ได้ต้องเห็นแถวข้อมูลที่บันทึกไว้ก่อนหน้ายังอยู่ครบ เพราะ container ตัวใหม่ mount volume `pgdata_prod` ตัวเดิม ซึ่งเก็บ physical data files ของ PostgreSQL ไว้จริง

### Bind Mount vs Named Volume

| คุณสมบัติ | Named Volume | Bind Mount |
|---|---|---|
| การจัดการโดย Docker | ใช่ (`docker volume ls/inspect/rm`) | ไม่ (เป็นแค่ path บน host) |
| Portability ระหว่างเครื่อง | ดีกว่า (Docker เลือก location ให้) | ต้องมี path เดียวกันทุกเครื่อง |
| Performance บน macOS/Windows | ดีกว่า (ผ่าน VM storage driver) | ช้ากว่าเมื่อ mount ผ่าน filesystem sharing |
| เหมาะสำหรับ | Production, CI/CD | Local dev ที่ต้องการเข้าถึงไฟล์จาก host โดยตรง (เช่น debug data files) |
| Backup/Migration | ต้อง `docker run --volumes-from` หรือ `docker cp` | เข้าถึงไฟล์ได้ตรง ๆ จาก host |

```bash
# ตัวอย่าง bind mount (เหมาะสำหรับ local development ที่ต้องการดู data files)
mkdir -p ~/pgdata-dev
docker run -d \
  --name pg-dev \
  -e POSTGRES_PASSWORD=devpass \
  -v ~/pgdata-dev:/var/lib/postgresql/data \
  -p 5433:5432 \
  postgres:16.4
```

**คำเตือน**: บน Linux การใช้ bind mount กับ PostgreSQL image อาจเจอปัญหา permission เพราะ process ภายใน container รันด้วย user `postgres` (UID 999 โดยปกติ) แต่ directory บน host เป็นของ user อื่น ทำให้ `initdb` ล้มเหลวด้วย permission denied ต้อง `chown -R 999:999 ~/pgdata-dev` ก่อน หรือใช้ named volume แทนเพื่อหลีกเลี่ยงปัญหานี้ทั้งหมด

### Backup Volume ด้วย `docker run --volumes-from`

```bash
# สำรอง volume ทั้งหมดเป็นไฟล์ tar
docker run --rm \
  -v pgdata_prod:/source:ro \
  -v "$(pwd)/backups":/backup \
  alpine \
  tar czf /backup/pgdata_prod_$(date +%Y%m%d_%H%M%S).tar.gz -C /source .
```

การ backup แบบนี้เหมาะสำหรับ disaster recovery ระดับ volume แต่ **ไม่แนะนำให้ใช้แทน `pg_dump` หรือ `pg_basebackup`** เพราะการ copy ไฟล์ data โดยตรงขณะ PostgreSQL กำลังทำงาน (ไม่ได้ stop container) อาจได้ snapshot ที่ไม่ consistent เนื่องจาก WAL และ data files อาจอยู่คนละจุดเวลากัน ควรใช้ `pg_basebackup` (ตามที่กล่าวถึงใน Part 065) หรืออย่างน้อยต้อง stop container ก่อน backup แบบ volume-level

### สรุปหลักการสำคัญของ Step นี้

- ห้ามรัน PostgreSQL container โดยไม่ mount volume — ข้อมูลจะอยู่ใน writable layer ที่หายได้ง่ายมาก
- ใช้ named volume เป็นค่าเริ่มต้นสำหรับ production
- ตั้งค่า `PGDATA` ให้ชี้ไปยัง subdirectory ของ mount point
- ปรับ `--shm-size` ให้เหมาะกับ workload
- การ backup ระดับ volume ไม่ใช่ทางเลือกแทน logical/physical backup ที่ PostgreSQL รองรับโดยตรง

---

## Step 922: Docker Compose สำหรับ PostgreSQL + Application Stack แบบเต็มรูปแบบ

ในการพัฒนาและ deploy ระบบจริง PostgreSQL แทบไม่เคยทำงานตัวเดียว แต่มักอยู่ร่วมกับ connection pooler, cache layer, และ application server เราจะประกอบ stack ที่สมจริงด้วย Docker Compose ประกอบด้วย:

- **PostgreSQL** — database หลัก
- **PgBouncer** — connection pooler (ตามที่กล่าวถึงใน Part 072-073)
- **Redis** — cache layer สำหรับ session และ query result caching
- **Application** — Node.js API server ตัวอย่าง

### โครงสร้างโปรเจกต์

```
myapp/
├── docker-compose.yml
├── .env
├── pgbouncer/
│   ├── pgbouncer.ini
│   └── userlist.txt
├── postgres/
│   └── init/
│       └── 01-init-schema.sql
└── app/
    ├── Dockerfile
    └── src/index.js
```

### `.env`

```bash
# .env — ไม่ควร commit ไฟล์นี้เข้า git จริง (ใช้ .gitignore)
POSTGRES_USER=appuser
POSTGRES_PASSWORD=ChangeMe_In_Production_2026
POSTGRES_DB=ecommerce
PGBOUNCER_AUTH_USER=appuser
PGBOUNCER_AUTH_PASSWORD=ChangeMe_In_Production_2026
REDIS_PASSWORD=RedisSecret2026
APP_PORT=3000
```

### `postgres/init/01-init-schema.sql`

```sql
-- ไฟล์ .sql ใน /docker-entrypoint-initdb.d จะถูกรันอัตโนมัติ
-- เมื่อ container เริ่มทำงานครั้งแรก (เฉพาะตอน data directory ว่างเปล่าเท่านั้น)
CREATE SCHEMA IF NOT EXISTS shop;

CREATE TABLE IF NOT EXISTS shop.products (
    id          bigserial PRIMARY KEY,
    sku         text NOT NULL UNIQUE,
    name        text NOT NULL,
    price_cents integer NOT NULL CHECK (price_cents >= 0),
    created_at  timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE IF NOT EXISTS shop.orders (
    id          bigserial PRIMARY KEY,
    customer_id bigint NOT NULL,
    status      text NOT NULL DEFAULT 'pending',
    total_cents integer NOT NULL DEFAULT 0,
    created_at  timestamptz NOT NULL DEFAULT now()
);

CREATE INDEX IF NOT EXISTS idx_orders_customer ON shop.orders (customer_id);
```

### `pgbouncer/pgbouncer.ini`

```ini
[databases]
ecommerce = host=postgres port=5432 dbname=ecommerce

[pgbouncer]
listen_addr = 0.0.0.0
listen_port = 6432
auth_type = md5
auth_file = /etc/pgbouncer/userlist.txt
pool_mode = transaction
max_client_conn = 500
default_pool_size = 25
min_pool_size = 5
reserve_pool_size = 5
reserve_pool_timeout = 3
server_idle_timeout = 600
log_connections = 1
log_disconnections = 1
admin_users = appuser
```

### `pgbouncer/userlist.txt`

```text
"appuser" "ChangeMe_In_Production_2026"
```

> ในสภาพแวดล้อมจริงควรใช้ `md5` hash ของรหัสผ่านแทนการใส่ plaintext (`SELECT 'md5' || md5('password' || 'username');`) และควรจัดการไฟล์นี้ผ่าน secret manager ไม่ใช่ commit เข้า repository

### `docker-compose.yml` (ไฟล์หลักของ Step นี้)

```yaml
version: "3.9"

services:
  postgres:
    image: postgres:16.4
    container_name: ecommerce-postgres
    restart: unless-stopped
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
      PGDATA: /var/lib/postgresql/data/pgdata
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./postgres/init:/docker-entrypoint-initdb.d:ro
    shm_size: "256mb"
    command:
      - "postgres"
      - "-c"
      - "max_connections=200"
      - "-c"
      - "shared_buffers=512MB"
      - "-c"
      - "effective_cache_size=1536MB"
      - "-c"
      - "work_mem=8MB"
      - "-c"
      - "log_min_duration_statement=500"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 5s
      timeout: 5s
      retries: 10
      start_period: 10s
    networks:
      - backend
    # ไม่ expose port 5432 ออกสู่ host โดยตรงในโปรดักชัน
    # (เปิดไว้เฉพาะตอนพัฒนา/debug เท่านั้น)
    ports:
      - "127.0.0.1:5432:5432"

  pgbouncer:
    image: edoburu/pgbouncer:1.21.0
    container_name: ecommerce-pgbouncer
    restart: unless-stopped
    depends_on:
      postgres:
        condition: service_healthy
    volumes:
      - ./pgbouncer/pgbouncer.ini:/etc/pgbouncer/pgbouncer.ini:ro
      - ./pgbouncer/userlist.txt:/etc/pgbouncer/userlist.txt:ro
    ports:
      - "6432:6432"
    networks:
      - backend

  redis:
    image: redis:7.4-alpine
    container_name: ecommerce-redis
    restart: unless-stopped
    command: ["redis-server", "--requirepass", "${REDIS_PASSWORD}", "--maxmemory", "256mb", "--maxmemory-policy", "allkeys-lru"]
    volumes:
      - redisdata:/data
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "${REDIS_PASSWORD}", "ping"]
      interval: 5s
      timeout: 3s
      retries: 10
    networks:
      - backend

  app:
    build:
      context: ./app
      dockerfile: Dockerfile
    container_name: ecommerce-app
    restart: unless-stopped
    depends_on:
      pgbouncer:
        condition: service_started
      redis:
        condition: service_healthy
    environment:
      DATABASE_URL: postgres://${POSTGRES_USER}:${POSTGRES_PASSWORD}@pgbouncer:6432/${POSTGRES_DB}
      REDIS_URL: redis://:${REDIS_PASSWORD}@redis:6379
      PORT: ${APP_PORT}
    ports:
      - "${APP_PORT}:3000"
    networks:
      - backend
    deploy:
      resources:
        limits:
          cpus: "1.0"
          memory: 512M

volumes:
  pgdata:
    name: ecommerce_pgdata
  redisdata:
    name: ecommerce_redisdata

networks:
  backend:
    driver: bridge
```

### จุดออกแบบที่ควรอธิบายให้ผู้เรียนเข้าใจ

1. **Application ไม่เชื่อมต่อ PostgreSQL โดยตรง แต่เชื่อมผ่าน PgBouncer (`pgbouncer:6432`)** — ทำให้ connection pool ถูกจัดการอย่างมีประสิทธิภาพ โดยเฉพาะเมื่อ scale จำนวน application instance ขึ้นหลายตัว
2. **`healthcheck` + `depends_on.condition: service_healthy`** — ทำให้ container ที่พึ่งพากัน (เช่น `pgbouncer` รอ `postgres`) เริ่มทำงานตามลำดับที่ถูกต้อง ไม่ใช่แค่รอให้ container start แต่รอให้ "พร้อมใช้งานจริง"
3. **`ports: - "127.0.0.1:5432:5432"`** — bind เฉพาะ localhost ของ host ไม่เปิดออก public เพื่อความปลอดภัย ในการใช้งานจริง production มักไม่ expose port ของ PostgreSQL ออกสู่ host เลยด้วยซ้ำ (ลบ `ports` block ออกทั้งหมด แล้วให้เข้าถึงผ่าน internal network เท่านั้น)
4. **`command` override ของ `postgres` image** — ใช้ตั้งค่า parameter สำคัญอย่าง `shared_buffers`, `work_mem` ผ่าน command-line flag แทนการแก้ `postgresql.conf` โดยตรง ซึ่งสะดวกสำหรับ container-based deployment
5. **`deploy.resources.limits`** — แม้จะทำงานเต็มรูปแบบเฉพาะบน Docker Swarm mode แต่การใส่ไว้ช่วยเป็นเอกสารอ้างอิง และหากย้ายไป Kubernetes ภายหลัง ค่าพวกนี้จะแปลงเป็น `resources.limits` ได้ตรงตัว

### คำสั่งใช้งาน

```bash
# เริ่ม stack ทั้งหมด
docker compose up -d

# ดู log ของ postgres
docker compose logs -f postgres

# เข้าไปสำรวจฐานข้อมูลผ่าน pgbouncer โดยตรง
docker compose exec postgres psql -U appuser -d ecommerce

# ทดสอบว่า pgbouncer ทำงานถูกต้อง (connect ผ่าน pgbouncer)
psql "postgresql://appuser:ChangeMe_In_Production_2026@localhost:6432/ecommerce" -c "SELECT 1;"

# ดูสถานะ pool ของ pgbouncer
psql "postgresql://appuser:ChangeMe_In_Production_2026@localhost:6432/pgbouncer" -c "SHOW POOLS;"

# ปิด stack แต่เก็บข้อมูลไว้ (ไม่ลบ volume)
docker compose down

# ปิด stack และลบข้อมูลทั้งหมด (ใช้เฉพาะตอนต้องการล้างข้อมูลจริง ๆ)
docker compose down -v
```

Stack นี้จำลองสภาพแวดล้อมการพัฒนาที่ใกล้เคียง production มากพอที่จะใช้ทดสอบ query performance, connection pooling behavior, และ caching strategy ได้จริงก่อน deploy ขึ้น environment จริง

---

## Step 923: ทำไม PostgreSQL บน Kubernetes ซับซ้อนกว่า Stateless Application

Kubernetes ถูกออกแบบมาสำหรับ **stateless workload** เป็นหลักตั้งแต่แรกเริ่ม — application server ที่ไม่มี state ติดตัว (state เก็บอยู่ที่ database ต่างหาก) เมื่อ Pod ตายหรือถูกย้ายไปโหนดอื่น เราสามารถสร้าง Pod ใหม่ทดแทนได้ทันทีโดยไม่ต้องสนใจว่า Pod เดิมคืออันไหน เพราะทุก instance เหมือนกันหมด (identical replicas)

แต่ PostgreSQL เป็น **stateful application** ที่มีลักษณะแตกต่างไปโดยพื้นฐาน:

### 1. Identity สำคัญ (Pod ไม่ใช่สิ่งที่ทดแทนกันได้ทันที)

ใน cluster PostgreSQL ที่มี primary 1 ตัวและ replica 2 ตัว แต่ละ instance **ไม่เหมือนกัน** — primary รับ write ได้ ส่วน replica รับได้แค่ read และต้อง stream WAL มาจาก primary ตัวใดตัวหนึ่งเจาะจง หาก Kubernetes สร้าง Pod ใหม่มาแทนที่ Pod ที่ตายไปโดยสุ่มเลือกว่าใครเป็น primary ใครเป็น replica ระบบจะพังทันที ต้องมีกลไกที่รู้ว่า "Pod ตัวไหนคือใคร" อย่างสม่ำเสมอ (stable network identity, stable storage identity)

### 2. Storage ต้องคงอยู่และผูกกับ Pod ที่ถูกต้อง

Deployment ทั่วไปใช้ ephemeral storage หรือ shared storage ที่ทุก replica เข้าถึงข้อมูลเดียวกันได้ (เช่น static file บน object storage) แต่ PostgreSQL แต่ละ instance ต้องมี **data directory ของตัวเอง** ที่ห้ามใช้ร่วมกัน (PostgreSQL ไม่รองรับ multiple instance เขียนทับ data directory เดียวกันพร้อมกัน) และเมื่อ Pod ถูกสร้างใหม่ (เช่นย้ายโหนด) จะต้องได้ **PersistentVolume (PV) เดิม** กลับมา ไม่ใช่ PV ใหม่ที่ว่างเปล่า

### 3. Network Identity ต้องเสถียร

Replica ต้องรู้ชัดเจนว่าจะ stream WAL จาก host ไหน (`primary_conninfo`) หากทุกครั้งที่ Pod restart แล้วได้ hostname/IP ใหม่แบบสุ่ม การตั้งค่า replication จะพังซ้ำ ๆ ต้องมี DNS name ที่คงที่ต่อ instance

### 4. Failover ต้องมีการตัดสินใจ ไม่ใช่แค่ restart

เมื่อ Pod ของ primary ตาย Kubernetes เพียงแค่ "restart Pod" ไม่เพียงพอ เพราะ:
- ต้องมีการเลือก **replica ตัวใดตัวหนึ่งขึ้นเป็น primary ใหม่** (leader election) โดยพิจารณาว่า replica ตัวไหนมี WAL ล่าสุด (LSN สูงสุด) เพื่อลด data loss
- ต้อง **promote** replica ตัวนั้นด้วยคำสั่งเฉพาะของ PostgreSQL (`pg_ctl promote` หรือเทียบเท่า)
- ต้อง **อัปเดต service/DNS routing** ให้ client ไปหา primary ใหม่ ไม่ใช่ Pod เดิมที่ตายไปแล้ว
- replica ตัวอื่น ๆ ต้องถูก **re-point** ให้ stream จาก primary ใหม่แทน
- ถ้า primary เดิมกลับมาออนไลน์อีกครั้ง (เช่นโหนดกลับมาทำงาน) ต้องป้องกันไม่ให้เกิด **split-brain** (มี "primary" สองตัวพร้อมกัน)

งานเหล่านี้คือ **domain-specific logic ที่ Kubernetes เองไม่รู้จัก** — Kubernetes ควบคุม container ได้ แต่ไม่รู้ว่า "PostgreSQL replica ตัวไหนมี LSN สูงสุด" หรือ "จะ promote replica อย่างไรให้ปลอดภัย" นี่คือเหตุผลที่ต้องมี **Operator** เข้ามาเสริมความสามารถของ Kubernetes ให้เข้าใจ domain logic ของ PostgreSQL โดยเฉพาะ (รายละเอียดใน Step 925 เป็นต้นไป)

### สรุปเปรียบเทียบ

| ประเด็น | Stateless App (เช่น web server) | PostgreSQL (Stateful) |
|---|---|---|
| Replica เหมือนกันหมดหรือไม่ | ใช่ ทดแทนกันได้ทันที | ไม่ — primary/replica มีบทบาทต่างกัน |
| Storage | มักไม่ต้องมี หรือ share storage ได้ | ต้องมี dedicated volume ต่อ instance ห้าม share |
| Network identity | ไม่สำคัญมาก | ต้องคงที่ (stable DNS) เพื่อ replication |
| การ scale out | เพิ่ม replica ใหม่ทันที | ต้อง clone/basebackup จาก primary ก่อน |
| Failure recovery | restart Pod ใหม่ก็พอ | ต้อง leader election + promotion + re-routing |
| ตัวช่วยหลักบน K8s | Deployment ธรรมดา | StatefulSet + Operator |

ด้วยเหตุผลเหล่านี้ Kubernetes จึงมี object พิเศษชื่อ **StatefulSet** ที่ออกแบบมาสำหรับ workload ที่ต้องการ identity คงที่และ storage เฉพาะตัว ซึ่งเราจะเรียนรู้ใน Step ถัดไป ก่อนที่จะไปถึง Operator ที่ทำงานซ้อนอยู่เหนือ StatefulSet อีกชั้นเพื่อจัดการ domain logic ของ PostgreSQL โดยเฉพาะ

---

## Step 924: StatefulSet พื้นฐานสำหรับ PostgreSQL

### ทำไมไม่ใช้ Deployment ธรรมดา

`Deployment` ใน Kubernetes ถูกออกแบบมาเพื่อจัดการ Pod ที่ **เหมือนกันทุกตัว (interchangeable)** — เมื่อ scale up/down Kubernetes จะสร้างหรือลบ Pod แบบสุ่มชื่อ (เช่น `myapp-7d9f8c6-x2k9p`) และเมื่อ Pod ถูกสร้างใหม่จะได้ PersistentVolumeClaim ใหม่ (หากใช้ `volumeClaimTemplates` ก็ยังทำไม่ได้ใน Deployment) ทำให้ไม่เหมาะกับ PostgreSQL ที่ต้องการ:

- ชื่อ Pod ที่คาดเดาได้และคงที่ (`postgres-0`, `postgres-1`, `postgres-2`)
- DNS name ที่เสถียรต่อ Pod แต่ละตัว
- PersistentVolumeClaim เฉพาะของแต่ละ Pod ที่ "ติดตาม" Pod นั้นไปทุกที่ (แม้ Pod จะถูกลบแล้วสร้างใหม่ ก็ยังได้ PVC เดิม)
- ลำดับการสร้าง/ลบที่แน่นอน (ordered, graceful deployment and scaling)

`StatefulSet` คือ object ของ Kubernetes ที่ตอบโจทย์ทั้งหมดนี้

### ตัวอย่าง StatefulSet พื้นฐานสำหรับ PostgreSQL (single primary, ไม่มี replication อัตโนมัติ)

> ตัวอย่างนี้เป็น **การสาธิตแนวคิดพื้นฐานของ StatefulSet** เพื่อความเข้าใจกลไกภายใน ในทางปฏิบัติจริงแทบไม่มีใครเขียน StatefulSet สำหรับ PostgreSQL มือเปล่าแบบนี้ในระดับ production เพราะขาด logic การจัดการ failover, backup, และ configuration ที่ซับซ้อน — งานนี้ควรปล่อยให้ Operator (Step 925 เป็นต้นไป) จัดการแทน

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres-headless
  labels:
    app: postgres
spec:
  clusterIP: None   # Headless Service — จำเป็นสำหรับ StatefulSet
  selector:
    app: postgres
  ports:
    - port: 5432
      name: postgresql
---
apiVersion: v1
kind: Secret
metadata:
  name: postgres-credentials
type: Opaque
stringData:
  POSTGRES_USER: appuser
  POSTGRES_PASSWORD: ChangeMe_In_Production_2026
  POSTGRES_DB: ecommerce
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres-headless   # ต้องตรงกับชื่อ headless service ด้านบน
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      terminationGracePeriodSeconds: 60
      containers:
        - name: postgres
          image: postgres:16.4
          ports:
            - containerPort: 5432
              name: postgresql
          envFrom:
            - secretRef:
                name: postgres-credentials
          env:
            - name: PGDATA
              value: /var/lib/postgresql/data/pgdata
          resources:
            requests:
              cpu: "500m"
              memory: "1Gi"
            limits:
              cpu: "2"
              memory: "2Gi"
          volumeMounts:
            - name: postgres-storage
              mountPath: /var/lib/postgresql/data
          readinessProbe:
            exec:
              command: ["pg_isready", "-U", "appuser", "-d", "ecommerce"]
            initialDelaySeconds: 10
            periodSeconds: 5
            timeoutSeconds: 3
          livenessProbe:
            exec:
              command: ["pg_isready", "-U", "appuser", "-d", "ecommerce"]
            initialDelaySeconds: 30
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 6
  volumeClaimTemplates:
    - metadata:
        name: postgres-storage
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: "fast-ssd"   # ขึ้นกับ storage class ที่ cluster รองรับ
        resources:
          requests:
            storage: 50Gi
```

### กลไกสำคัญของ StatefulSet ที่ควรอธิบาย

1. **`volumeClaimTemplates`** — จุดต่างสำคัญที่สุดจาก Deployment เมื่อ StatefulSet สร้าง Pod ชื่อ `postgres-0` มันจะสร้าง PVC ชื่อ `postgres-storage-postgres-0` โดยอัตโนมัติ และเมื่อ Pod `postgres-0` ถูกลบแล้วสร้างใหม่ (เช่น restart หรือ reschedule ไปโหนดอื่น) StatefulSet controller จะ **mount PVC เดิม** กลับเข้ากับ Pod ใหม่เสมอ ไม่สร้าง volume ว่างเปล่าใหม่
2. **`serviceName` + headless Service (`clusterIP: None`)** — ทำให้แต่ละ Pod มี stable DNS name ในรูปแบบ `<pod-name>.<service-name>.<namespace>.svc.cluster.local` เช่น `postgres-0.postgres-headless.default.svc.cluster.local` ซึ่งคงที่ไม่ว่า Pod จะถูกสร้างใหม่กี่ครั้งก็ตาม (ตราบใดที่ยังชื่อ `postgres-0` เดิม) — นี่คือกลไกที่ทำให้ replica สามารถ stream WAL จาก primary ด้วยชื่อที่แน่นอนได้
3. **Ordered creation/scaling** — เมื่อ scale StatefulSet จาก `replicas: 1` เป็น `replicas: 3` Kubernetes จะสร้าง `postgres-1` ก่อน แล้วรอให้ `Ready` ก่อนถึงจะสร้าง `postgres-2` ต่อ (ตามค่า default ของ `podManagementPolicy: OrderedReady`) ซึ่งสำคัญมากสำหรับ replication ที่ replica ตัวถัดไปมักต้องพึ่งพา primary ที่พร้อมทำงานแล้ว
4. **Readiness/Liveness Probe ด้วย `pg_isready`** — ใช้ตรวจสอบว่า PostgreSQL พร้อมรับ connection จริงหรือไม่ ไม่ใช่แค่ตรวจว่า process ยังรันอยู่

### ข้อจำกัดของการเขียน StatefulSet มือเปล่า

แม้ StatefulSet จะแก้ปัญหาเรื่อง identity และ storage ได้ แต่ยังขาดความสามารถสำคัญที่ต้องมีในการรัน PostgreSQL ระดับ production:

- **ไม่รู้จัก replication topology** — StatefulSet ไม่รู้ว่า Pod ตัวไหนคือ primary ตัวไหนคือ replica ต้องตั้งค่าเองผ่าน init script หรือ entrypoint ที่ซับซ้อน
- **ไม่มี automatic failover** — เมื่อ `postgres-0` (primary) ตาย StatefulSet จะแค่พยายาม restart Pod เดิม ไม่ promote replica ให้อัตโนมัติ
- **ไม่มี automated backup scheduling**
- **ไม่มี connection routing ที่ฉลาด** (ไม่รู้ว่าต้องส่ง write ไปที่ไหน read ไปที่ไหน)
- **การ scale ต้องมี custom logic เพิ่ม** เพื่อสั่งให้ replica ใหม่ทำ `pg_basebackup` จาก primary

ทั้งหมดนี้คือเหตุผลที่นำไปสู่ **Kubernetes Operator Pattern** ซึ่งเราจะอธิบายในรายละเอียดในบทถัดไป

---

## Step 925: Kubernetes Operator Pattern คืออะไร

### แนวคิดหลัก: Operator = Human Operational Knowledge ที่เขียนเป็นโค้ด

Kubernetes Operator คือ software pattern ที่ **encode ความรู้และขั้นตอนการดูแลระบบของมนุษย์ (human operational knowledge)** ให้กลายเป็นโค้ดที่ทำงานอัตโนมัติภายใน cluster แนวคิดนี้เริ่มต้นโดย CoreOS ในปี 2016 โดยมีเป้าหมายให้ระบบที่ซับซ้อนอย่าง database, message queue, หรือ monitoring stack สามารถ "ดูแลตัวเอง" ได้เหมือนที่ Site Reliability Engineer (SRE) หรือ Database Administrator (DBA) เคยทำด้วยมือ

Operator ประกอบด้วยสองส่วนหลัก:

### 1. Custom Resource Definition (CRD)

CRD คือการขยาย Kubernetes API ให้รู้จัก object type ใหม่ที่ไม่ได้มีมาให้ในตัว Kubernetes เอง เช่นแทนที่จะต้องเขียน StatefulSet + Service + Secret + ConfigMap แยกกันหลายไฟล์ทุกครั้งที่ต้องการ PostgreSQL cluster ใหม่ เราสามารถนิยาม object ใหม่ชื่อ `Cluster` หรือ `postgresql` ที่บอกความต้องการระดับสูง (declarative, high-level intent) เพียงไม่กี่บรรทัด เช่น "ฉันต้องการ PostgreSQL 16 จำนวน 3 instance พร้อม backup รายวัน"

```yaml
# ตัวอย่างแนวคิด CRD (ไม่ใช่ manifest ที่ต้อง apply เอง — CRD นี้ถูกติดตั้งมาพร้อม Operator แล้ว)
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: clusters.postgresql.cnpg.io
spec:
  group: postgresql.cnpg.io
  names:
    kind: Cluster
    plural: clusters
    singular: cluster
  scope: Namespaced
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                instances:
                  type: integer
                storage:
                  type: object
                  properties:
                    size:
                      type: string
```

### 2. Controller (Reconciliation Loop)

Controller คือโปรแกรม (มักรันเป็น Pod ภายใน cluster เอง) ที่คอย **watch** การเปลี่ยนแปลงของ custom resource แล้วพยายามทำให้สถานะจริงของระบบ (actual state) ตรงกับสถานะที่ต้องการ (desired state) ที่ประกาศไว้ใน YAML อย่างต่อเนื่องไม่มีที่สิ้นสุด — กระบวนการนี้เรียกว่า **reconciliation loop**

```
┌─────────────────────────────────────────────────────────┐
│                  Reconciliation Loop                     │
│                                                             │
│   1. Observe    →  อ่านสถานะปัจจุบันของ Cluster resource  │
│                     และ Pod/PVC/Service ที่เกี่ยวข้องจริง  │
│                                                             │
│   2. Diff       →  เปรียบเทียบ desired state (ใน YAML)     │
│                     กับ actual state (สิ่งที่มีอยู่จริง)   │
│                                                             │
│   3. Act        →  สร้าง/แก้ไข/ลบ Pod, PVC, Service,      │
│                     Secret, ConfigMap ให้ตรงกับ desired    │
│                                                             │
│   4. Repeat     →  วนลูปนี้ตลอดเวลา (เป็นวินาที)           │
└─────────────────────────────────────────────────────────┘
```

ตัวอย่างที่เป็นรูปธรรม: เมื่อ PostgreSQL Operator watch เห็นว่า Pod ที่ทำหน้าที่ primary หายไปจาก cluster (เช่นโหนดล่ม) มันจะ:

1. ตรวจสอบ replica ทั้งหมดที่เหลือ เปรียบเทียบค่า **LSN (Log Sequence Number)** ของแต่ละตัวเพื่อหาตัวที่มีข้อมูลล่าสุดที่สุด
2. สั่ง promote replica ตัวนั้นให้เป็น primary ใหม่ (ผ่านกลไกภายในของ PostgreSQL เช่น `pg_ctl promote`)
3. อัปเดต Service/label ให้ traffic ไหลไปยัง Pod ที่เป็น primary ใหม่
4. สั่งให้ replica ตัวอื่นที่เหลือ re-point การ stream WAL ไปยัง primary ใหม่
5. เมื่อโหนดเดิมกลับมา สร้าง Pod ใหม่ให้เข้าร่วม cluster ในฐานะ replica (ไม่ใช่ primary) เพื่อป้องกัน split-brain

งานทั้งหมดนี้เกิดขึ้น **โดยอัตโนมัติภายในไม่กี่วินาทีถึงนาที** โดยที่ DBA ไม่ต้องเข้ามาแทรกแซงเอง — นี่คือคุณค่าหลักของ Operator pattern

### เปรียบเทียบ: ไม่มี Operator vs มี Operator

| งาน | ไม่มี Operator (StatefulSet มือเปล่า) | มี Operator |
|---|---|---|
| สร้าง cluster ใหม่ | เขียน YAML หลายไฟล์ ตั้งค่า replication เอง | ประกาศ `instances: 3` บรรทัดเดียว |
| Failover เมื่อ primary ตาย | ต้องมีคนเข้ามาสั่ง promote เอง หรือเขียน script custom | อัตโนมัติภายในไม่กี่วินาที |
| Scale out (เพิ่ม replica) | ต้องเขียน init script ทำ `pg_basebackup` เอง | ประกาศ `instances: 5` แล้ว Operator จัดการ clone ให้ |
| Backup รายวัน | ต้องตั้ง CronJob แยกเขียนเอง | ประกาศ schedule ใน CRD |
| Minor version upgrade | rolling restart เอง ระวังลำดับ replica/primary | Operator จัดการ rolling upgrade ให้ |
| Connection routing (read/write split) | ต้องทำ Service แยกเอง | Operator สร้าง Service `-rw`/`-ro`/`-r` ให้อัตโนมัติ |

### Operator ที่นิยมใช้สำหรับ PostgreSQL

ในระบบนิเวศของ Kubernetes มี PostgreSQL Operator หลายตัว โดยที่ได้รับความนิยมสูงสุดในปัจจุบันคือ:

1. **Zalando Postgres Operator** — พัฒนาโดยทีม Zalando (บริษัท e-commerce ยุโรป) เปิดตัวตั้งแต่ปี 2017 ใช้ Patroni เป็นเครื่องมือจัดการ high availability ภายใน
2. **CloudNativePG (CNPG)** — พัฒนาโดย EDB (EnterpriseDB) เปิดตัวปี 2022 ปัจจุบันเป็น CNCF Sandbox project ออกแบบมาให้ "cloud native" ตั้งแต่ต้น ไม่พึ่งพา external tool อย่าง Patroni
3. **Crunchy Postgres Operator (PGO)** — พัฒนาโดย Crunchy Data เน้นความปลอดภัยและ compliance สูง
4. **StackGres** — เน้นความง่ายในการติดตั้งและมี UI จัดการ

บทนี้จะเน้นเจาะลึกที่ **Zalando Postgres Operator** และ **CloudNativePG** เนื่องจากเป็นสองตัวที่ถูกใช้งานอย่างแพร่หลายในองค์กรระดับโลก และมีปรัชญาการออกแบบที่ต่างกันอย่างน่าสนใจ

---

## Step 926: Zalando Postgres Operator — ภาพรวมและตัวอย่าง Manifest

### ประวัติและสถาปัตยกรรม

Zalando Postgres Operator เป็นหนึ่งใน Operator ตัวแรก ๆ สำหรับ PostgreSQL บน Kubernetes ถูกพัฒนาขึ้นเพื่อใช้งานภายในของ Zalando เองตั้งแต่ปี 2017 ก่อนจะ open source ให้ชุมชนใช้งาน จุดเด่นสำคัญของสถาปัตยกรรมคือการพึ่งพา **Patroni** ซึ่งเป็นเครื่องมือ HA (High Availability) template ที่ใช้ **DCS (Distributed Configuration Store)** อย่าง etcd, Consul หรือ Kubernetes API เองเป็นตัวกลางในการทำ leader election และเก็บสถานะของ cluster

```
┌───────────────────────────────────────────────────────────┐
│                 Zalando Postgres Operator                    │
│                                                                │
│   ┌──────────────┐     watches     ┌─────────────────────┐  │
│   │  postgresql   │ ─────────────▶ │  Operator Controller │  │
│   │  CRD instance │                 │  (Go, reconcile loop) │  │
│   └──────────────┘                 └──────────┬────────────┘  │
│                                                  │              │
│                                     สร้าง/จัดการ  ▼              │
│              ┌────────────────────────────────────────────┐  │
│              │           StatefulSet (Pod x N)              │  │
│              │  ┌──────────┐  ┌──────────┐  ┌──────────┐   │  │
│              │  │ Pod 0    │  │ Pod 1    │  │ Pod 2    │   │  │
│              │  │ Patroni  │  │ Patroni  │  │ Patroni  │   │  │
│              │  │ + PG     │  │ + PG     │  │ + PG     │   │  │
│              │  └────┬─────┘  └────┬─────┘  └────┬─────┘   │  │
│              └───────┼─────────────┼─────────────┼─────────┘  │
│                       │  leader election ผ่าน    │              │
│                       └──────── Kubernetes API/etcd ┘            │
└───────────────────────────────────────────────────────────┘
```

แต่ละ Pod ใน StatefulSet รัน **Patroni เป็น sidecar/wrapper process** ที่ควบคุม PostgreSQL อีกที Patroni เองเป็นคนตัดสินใจว่าใครเป็น leader (primary) โดยแข่งกันถือ "lock" ใน Kubernetes API (หรือ etcd) — ตัวที่ถือ lock ได้คือ primary ตัวที่เหลือกลายเป็น replica ที่ stream WAL มาจาก primary โดยอัตโนมัติ

### การติดตั้ง Operator (bash)

```bash
# ติดตั้งผ่าน Helm chart (แนวทางที่แนะนำ)
helm repo add postgres-operator-charts \
  https://opensource.zalando.com/postgres-operator/charts/postgres-operator
helm repo update

helm install postgres-operator postgres-operator-charts/postgres-operator \
  --namespace postgres-operator \
  --create-namespace \
  --version 1.13.0
```

### ตัวอย่าง Manifest: สร้าง PostgreSQL Cluster ด้วย `postgresql` CRD

```yaml
apiVersion: "acid.zalan.do/v1"
kind: postgresql
metadata:
  name: ecommerce-cluster
  namespace: production
  labels:
    team: platform
spec:
  teamId: "platform"
  postgresql:
    version: "16"
    parameters:
      shared_buffers: "1GB"
      max_connections: "200"
      work_mem: "16MB"
      log_min_duration_statement: "500"
      wal_level: "replica"

  numberOfInstances: 3   # 1 primary + 2 replica

  volume:
    size: 100Gi
    storageClass: fast-ssd

  resources:
    requests:
      cpu: "1"
      memory: "2Gi"
    limits:
      cpu: "4"
      memory: "4Gi"

  # Database และ user ที่ Operator จะสร้างอัตโนมัติ
  databases:
    ecommerce: appuser

  users:
    appuser:
      - superuser
      - createdb
    readonly_user:
      - login

  # ตั้งค่า connection pooler (pgbouncer) ให้ operator จัดการเองในตัว
  enableConnectionPooler: true
  connectionPooler:
    numberOfInstances: 2
    mode: "transaction"
    maxDBConnections: 60

  # ตั้งค่า automated backup ไปยัง S3-compatible storage (ผ่าน WAL-G ภายใน)
  # การตั้งค่า credential จริงมักอยู่ใน environment ของ Operator เอง
  # ผ่าน ConfigMap `pod_environment_configmap`

  patroni:
    synchronous_mode: false
    pg_hba:
      - "host all all 0.0.0.0/0 md5"
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 33554432   # 32MB

  maintenanceWindows:
    - "Sat:02:00-04:00"

  podAnnotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "9187"
```

### สิ่งที่ Operator สร้างให้อัตโนมัติเมื่อ apply manifest นี้

```bash
kubectl apply -f ecommerce-cluster.yaml -n production

# ตรวจสอบสถานะ cluster
kubectl get postgresql -n production

# ดู Pod ที่ถูกสร้าง (StatefulSet ภายใต้การจัดการของ Operator)
kubectl get pods -n production -l cluster-name=ecommerce-cluster

# ดู Service ที่ Operator สร้างให้อัตโนมัติ:
#   ecommerce-cluster       -> ชี้ไปยัง primary (read-write)
#   ecommerce-cluster-repl  -> ชี้ไปยัง replica (read-only, load balanced)
kubectl get svc -n production -l cluster-name=ecommerce-cluster

# ดู log ของ Patroni ภายใน Pod เพื่อตรวจสอบ leader election
kubectl logs ecommerce-cluster-0 -n production -c postgres | grep -i patroni
```

### ตรวจสอบว่าใครเป็น primary ปัจจุบัน (ผ่าน Patroni REST API)

```bash
# Patroni รัน REST API บน port 8008 ของทุก Pod
kubectl exec -it ecommerce-cluster-0 -n production -- \
  curl -s http://localhost:8008/cluster | jq .

# หรือใช้ patronictl ที่ built-in อยู่ใน container
kubectl exec -it ecommerce-cluster-0 -n production -- \
  patronictl -c /etc/patroni.yml list
```

### จุดเด่นของ Zalando Operator

- ใช้ **Patroni** ซึ่งเป็นเครื่องมือ HA สำหรับ PostgreSQL ที่ผ่านการพิสูจน์แล้วอย่างกว้างขวาง (ใช้แม้กระทั่งใน non-Kubernetes environment ก็ได้)
- รองรับ **connection pooler แบบ built-in** (pgbouncer sidecar) โดยไม่ต้องตั้งค่าแยกเอง
- มี **UI (postgres-operator-ui)** ให้ใช้จัดการ cluster ผ่านหน้าเว็บได้
- รองรับการทำ `databases` และ `users` เป็นส่วนหนึ่งของ CRD โดยตรง ลด manual step ในการสร้าง schema/role เบื้องต้น
- Community ใหญ่ ใช้งานจริงในองค์กรขนาดใหญ่มานานหลายปี (production-proven ตั้งแต่ปี 2017)

---

## Step 927: CloudNativePG (CNPG) — Operator ยอดนิยมสมัยใหม่

### ปรัชญาการออกแบบที่ต่างจาก Zalando

CloudNativePG (มักเรียกย่อว่า **CNPG**) พัฒนาโดย EDB เปิดตัวในปี 2022 และปัจจุบันเป็นโปรเจกต์ระดับ **CNCF Sandbox** จุดเด่นที่สำคัญที่สุดของ CNPG คือ **ไม่พึ่งพา Patroni หรือ external DCS (เช่น etcd) เลย** — CNPG เขียน logic การทำ leader election และ failover ทั้งหมดขึ้นมาใหม่โดยใช้ **Kubernetes API เป็นแหล่งความจริงเดียว (single source of truth)** ผ่านกลไก native ของ Kubernetes เอง เช่น การใช้ Kubernetes lease object เป็นกลไกคล้าย distributed lock

```
┌───────────────────────────────────────────────────────────┐
│                     CloudNativePG (CNPG)                     │
│                                                                │
│   ┌──────────────┐     watches     ┌─────────────────────┐  │
│   │   Cluster     │ ─────────────▶ │  CNPG Controller      │  │
│   │  CRD instance │                 │  (มี instance manager  │  │
│   └──────────────┘                 │   ฝังตัวใน Pod ด้วย)   │  │
│                                     └──────────┬────────────┘  │
│                                                  │              │
│                                     สร้าง/จัดการ  ▼              │
│              ┌────────────────────────────────────────────┐  │
│              │           Pod (ไม่ใช้ StatefulSet ดั้งเดิม     │  │
│              │            แต่ใช้กลไกคล้ายกันภายใน)           │  │
│              │  ┌──────────┐  ┌──────────┐  ┌──────────┐   │  │
│              │  │ Pod 0    │  │ Pod 1    │  │ Pod 2    │   │  │
│              │  │ instance │  │ instance │  │ instance │   │  │
│              │  │ manager  │  │ manager  │  │ manager  │   │  │
│              │  │ + PG     │  │ + PG     │  │ + PG     │   │  │
│              │  └────┬─────┘  └────┬─────┘  └────┬─────┘   │  │
│              └───────┼─────────────┼─────────────┼─────────┘  │
│                       │   leader election ผ่าน                │
│                       └─── Kubernetes native (lease object) ──┘│
└───────────────────────────────────────────────────────────┘
```

แต่ละ Pod รัน **instance manager** ซึ่งเป็น process ที่เขียนด้วย Go โดยทีม CNPG เอง ทำหน้าที่คล้าย Patroni แต่ built-in ไปกับ container image เลย ไม่ต้องพึ่ง external component ใด ๆ เพิ่มเติม จุดนี้ทำให้ CNPG มี **moving part น้อยกว่า** และ debug ง่ายกว่าในหลายกรณี

### การติดตั้ง Operator (bash)

```bash
# วิธีที่ 1: ติดตั้งผ่าน kubectl apply โดยตรง (แนะนำสำหรับทดสอบเร็ว)
kubectl apply --server-side -f \
  https://raw.githubusercontent.com/cloudnative-pg/cloudnative-pg/release-1.24/releases/cnpg-1.24.1.yaml

# วิธีที่ 2: ติดตั้งผ่าน Helm chart (แนะนำสำหรับ production)
helm repo add cnpg https://cloudnative-pg.github.io/charts
helm repo update

helm upgrade --install cnpg \
  --namespace cnpg-system \
  --create-namespace \
  cnpg/cloudnative-pg \
  --version 0.22.0

# ตรวจสอบว่า Operator ทำงานปกติ
kubectl get deployment -n cnpg-system cnpg-controller-manager
```

### ตัวอย่าง Manifest: สร้าง PostgreSQL Cluster ด้วย `Cluster` CRD

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: ecommerce-cluster
  namespace: production
spec:
  instances: 3   # 1 primary + 2 replica

  imageName: ghcr.io/cloudnative-pg/postgresql:16.4

  postgresql:
    parameters:
      shared_buffers: "1GB"
      max_connections: "200"
      work_mem: "16MB"
      log_min_duration_statement: "500"
      wal_level: "replica"
    pg_hba:
      - "host all all 10.0.0.0/8 md5"

  storage:
    size: 100Gi
    storageClass: fast-ssd

  resources:
    requests:
      cpu: "1"
      memory: "2Gi"
    limits:
      cpu: "4"
      memory: "4Gi"

  # การสร้าง database และ user เริ่มต้น
  bootstrap:
    initdb:
      database: ecommerce
      owner: appuser
      secret:
        name: ecommerce-app-credentials
      dataChecksums: true
      encoding: UTF8

  # Affinity: กระจาย Pod แต่ละตัวไปคนละโหนด/คนละ availability zone
  affinity:
    enablePodAntiAffinity: true
    topologyKey: topology.kubernetes.io/zone

  # Automated backup ไปยัง S3-compatible object storage ผ่าน Barman Cloud
  backup:
    barmanObjectStore:
      destinationPath: "s3://ecommerce-pg-backups/production"
      s3Credentials:
        accessKeyId:
          name: backup-s3-credentials
          key: ACCESS_KEY_ID
        secretAccessKey:
          name: backup-s3-credentials
          key: SECRET_ACCESS_KEY
      wal:
        compression: gzip
        maxParallel: 4
      data:
        compression: gzip
        jobs: 4
    retentionPolicy: "30d"

  # กำหนดตารางเวลาสำหรับ scheduled backup (แยก CRD ต่างหาก ดู manifest ถัดไป)

  monitoring:
    enablePodMonitor: true

  # การตั้งค่า connection pooler แยก CRD ต่างหาก (Pooler resource) ดูด้านล่าง
---
apiVersion: v1
kind: Secret
metadata:
  name: ecommerce-app-credentials
  namespace: production
type: kubernetes.io/basic-auth
stringData:
  username: appuser
  password: ChangeMe_In_Production_2026
---
apiVersion: postgresql.cnpg.io/v1
kind: Pooler
metadata:
  name: ecommerce-cluster-pooler
  namespace: production
spec:
  cluster:
    name: ecommerce-cluster
  instances: 2
  type: rw   # rw = ชี้ไปยัง primary, ro = ชี้ไปยัง replica
  pgbouncer:
    poolMode: transaction
    parameters:
      max_client_conn: "500"
      default_pool_size: "25"
---
apiVersion: postgresql.cnpg.io/v1
kind: ScheduledBackup
metadata:
  name: ecommerce-daily-backup
  namespace: production
spec:
  schedule: "0 2 * * *"   # cron format — ทุกวันตี 2
  backupOwnerReference: self
  cluster:
    name: ecommerce-cluster
```

### คำสั่งตรวจสอบและจัดการ

```bash
kubectl apply -f ecommerce-cluster.yaml -n production

# ตรวจสอบสถานะ cluster (แสดง role ของแต่ละ instance ชัดเจน)
kubectl get cluster ecommerce-cluster -n production

# ดูรายละเอียดแบบเจาะลึก รวมถึงว่า Pod ไหนเป็น primary
kubectl describe cluster ecommerce-cluster -n production

# CNPG มี plugin สำหรับ kubectl โดยเฉพาะ ช่วยดู status สวยงามอ่านง่าย
kubectl cnpg status ecommerce-cluster -n production

# ดู Service ที่ CNPG สร้างให้อัตโนมัติ
#   ecommerce-cluster-rw   -> primary เท่านั้น (read-write)
#   ecommerce-cluster-ro   -> replica เท่านั้น (read-only)
#   ecommerce-cluster-r    -> ทุก instance รวม primary (read, load-balanced)
kubectl get svc -n production -l cnpg.io/cluster=ecommerce-cluster

# สั่ง manual switchover (เปลี่ยน primary แบบวางแผนล่วงหน้า ไม่ใช่ failover ฉุกเฉิน)
kubectl cnpg promote ecommerce-cluster ecommerce-cluster-2 -n production
```

### จุดเด่นของ CloudNativePG

- **ไม่มี external dependency อย่าง etcd/Patroni** — ลดความซับซ้อนของสถาปัตยกรรมโดยรวม ใช้ Kubernetes native mechanism ทั้งหมด
- **CNCF Sandbox project** — อยู่ภายใต้การกำกับดูแลของ CNCF ซึ่งเป็นองค์กรกลางที่ดูแล Kubernetes เอง เพิ่มความน่าเชื่อถือระยะยาว
- **Barman Cloud integration แบบ built-in** สำหรับ continuous backup ไปยัง S3-compatible storage โดยไม่ต้องติดตั้ง tool เพิ่ม
- **`kubectl cnpg` plugin** ทำให้ operation งานประจำ (status, promote, backup, restore) สะดวกผ่าน command line
- **Declarative connection pooler** (`Pooler` CRD) แยกเป็น resource ต่างหาก ยืดหยุ่นกว่าในการปรับ scale ของ pooler อิสระจาก database instance
- ได้รับการพัฒนาอย่างต่อเนื่องและรวดเร็ว มี release cycle ถี่ รองรับ PostgreSQL เวอร์ชันใหม่ ๆ เร็ว

---

## Step 928: เปรียบเทียบ Zalando Operator กับ CloudNativePG

การเลือก Operator ที่เหมาะสมกับองค์กรควรพิจารณาจากหลายมิติ ไม่ใช่แค่ฟีเจอร์อย่างเดียว แต่รวมถึงความคุ้นเคยของทีม, ระบบนิเวศที่มีอยู่, และทิศทางระยะยาวของโปรเจกต์

### ตารางเปรียบเทียบละเอียด

| หัวข้อ | Zalando Postgres Operator | CloudNativePG (CNPG) |
|---|---|---|
| ผู้พัฒนาหลัก | Zalando SE | EDB (EnterpriseDB) |
| ปีเปิดตัว | 2017 | 2022 |
| สถานะโปรเจกต์ | Open source อิสระ | CNCF Sandbox project |
| กลไก HA/failover | พึ่งพา Patroni + DCS (K8s API/etcd/Consul) | Native Kubernetes mechanism (ไม่พึ่ง external tool) |
| Complexity ของสถาปัตยกรรม | สูงกว่า (มี moving part เพิ่ม คือ Patroni) | ต่ำกว่า (built-in ทั้งหมดใน instance manager) |
| Connection Pooler | pgbouncer sidecar แบบ built-in ผ่าน field เดียวใน CRD | แยกเป็น `Pooler` CRD ต่างหาก ยืดหยุ่นกว่า |
| Backup solution | WAL-E / WAL-G (ผ่าน environment config) | Barman Cloud (built-in, integration แน่นกว่า) |
| Scheduled Backup CRD | ไม่มี CRD เฉพาะ ตั้งผ่าน CronJob แยกหรือ config ภายนอก | มี `ScheduledBackup` CRD โดยตรง |
| Point-in-Time Recovery (PITR) | รองรับผ่าน WAL-G | รองรับผ่าน Barman Cloud พร้อม CRD `Cluster` ที่มี `bootstrap.recovery` |
| Web UI จัดการ | มี (postgres-operator-ui) | ไม่มี UI ในตัว (ใช้ `kubectl cnpg` plugin แทน) |
| Multi-cloud / Object storage รองรับ | S3, GCS, Azure Blob (ผ่าน WAL-G) | S3, GCS, Azure Blob (ผ่าน Barman Cloud) |
| Database/User provisioning | ระบุใน CRD `databases`/`users` ได้ตรง ๆ | ระบุผ่าน `bootstrap.initdb` และ CRD เสริม (`Database`, `Publication` ในเวอร์ชันใหม่) |
| Rolling upgrade (minor version) | รองรับ | รองรับ พร้อม `primaryUpdateStrategy` ที่ปรับ downtime ได้ละเอียด |
| Replication slot management | จัดการผ่าน Patroni | จัดการ native พร้อม feature sync replication slot สำหรับ HA replica |
| ความสามารถด้าน Tablespace | จำกัด | รองรับ declarative tablespace management (เวอร์ชันใหม่) |
| Community/adoption | ใหญ่ ใช้งานมานาน มี case study จากองค์กรใหญ่หลายแห่ง | เติบโตเร็วมาก ได้รับความนิยมสูงในโปรเจกต์ใหม่ตั้งแต่ปี 2023 เป็นต้นมา |
| Release cadence | ปานกลาง | เร็ว รองรับ PostgreSQL เวอร์ชันใหม่รวดเร็ว |
| เอกสารและ tutorial | ครบถ้วน มีมานาน | ครบถ้วน ทันสมัย เขียนดีมาก (จาก EDB) |
| เหมาะกับ | ทีมที่คุ้นเคยกับ Patroni อยู่แล้ว หรือใช้ Patroni นอก K8s ด้วย | ทีมที่ต้องการสถาปัตยกรรม cloud-native ล้วน ลด dependency ภายนอก |

### คำแนะนำในการเลือกใช้งาน

**เลือก Zalando Postgres Operator เมื่อ:**
- ทีมมีความคุ้นเคยกับ Patroni อยู่แล้ว (เช่นเคยใช้ Patroni บน VM หรือ bare-metal มาก่อน) และต้องการความสม่ำเสมอของ mental model
- ต้องการ Web UI สำหรับให้ทีมที่ไม่ถนัด command line จัดการ cluster ได้ง่าย
- มีระบบ backup ที่ใช้ WAL-G อยู่แล้วในองค์กร

**เลือก CloudNativePG เมื่อ:**
- เริ่มต้นโปรเจกต์ใหม่และต้องการสถาปัตยกรรมที่เรียบง่าย ลด external dependency ให้น้อยที่สุด
- ให้ความสำคัญกับการเป็นโปรเจกต์ที่อยู่ภายใต้ CNCF governance (ความมั่นใจระยะยาวสูงกว่าในมุมมององค์กร)
- ต้องการ built-in integration กับ object storage สำหรับ backup/PITR ที่ตั้งค่าได้ง่ายกว่า
- ต้องการ `kubectl cnpg` plugin ที่ช่วยงาน operation ประจำวันได้สะดวก
- เป็นโปรเจกต์ใหม่ที่เริ่มในปี 2023 เป็นต้นมา — เทรนด์ตลาดปัจจุบันมักนิยม CNPG มากกว่าสำหรับโปรเจกต์ใหม่ เนื่องจากความง่ายในการ maintain ระยะยาว

> ในบทเรียนถัดไปและแบบฝึกหัดของบทนี้ เราจะใช้ **CloudNativePG** เป็นหลักในการสาธิต เนื่องจากเป็นทิศทางที่อุตสาหกรรมกำลังมุ่งไปมากกว่าในช่วงปี 2024-2026

---

## Step 929: High Availability บน Kubernetes — Operator จัดการ Failover และ Backup อัตโนมัติอย่างไร

ใน Part 065 เราได้พูดถึงหลักการ High Availability, Streaming Replication, และ Point-in-Time Recovery (PITR) ของ PostgreSQL ไปแล้วในระดับ engine เอง ในบทนี้เราจะเชื่อมโยงหลักการเหล่านั้นเข้ากับสิ่งที่ Operator ทำให้อัตโนมัติบน Kubernetes

### กลไก Automatic Failover แบบละเอียด (อ้างอิงตามแนวทางของ CloudNativePG)

```
สถานการณ์: Pod ที่เป็น primary (ecommerce-cluster-1) ตายกะทันหัน (โหนดล่ม)

Timeline:
  T+0s   Kubelet บนโหนดที่ล่มหยุดตอบสนอง (node NotReady)
  T+0s   kube-controller-manager เริ่มนับ node-monitor-grace-period
         (ค่า default ประมาณ 40 วินาที ก่อนตัดสินว่าโหนด NotReady จริง)
  T+~40s Kubernetes ทำเครื่องหมาย Pod บนโหนดนั้นว่า Unknown/Terminating
  T+~40s CNPG instance manager บน replica ตัวอื่น (ผ่าน liveness
         probe ของตัวเอง) ตรวจพบว่า primary ไม่ตอบสนอง
  T+~41s CNPG controller เลือก replica ที่มี LSN ล่าสุดสูงสุดจากทั้งหมด
         (เปรียบเทียบผ่าน pg_stat_replication ที่บันทึกไว้ล่าสุด)
  T+~42s สั่ง promote replica ตัวที่เลือก (เช่น ecommerce-cluster-2)
         ด้วยคำสั่งภายในเทียบเท่า `pg_promote()`
  T+~43s อัปเดต Endpoint ของ Service `ecommerce-cluster-rw`
         ให้ชี้ไปยัง Pod ใหม่ (ecommerce-cluster-2) แทน
  T+~44s replica ที่เหลือ (ecommerce-cluster-3 เดิม) ถูกสั่งให้
         re-point primary_conninfo ไปยัง primary ใหม่
  T+~60s Kubernetes สร้าง Pod ทดแทนบนโหนดอื่น (เมื่อ storage
         attach ใหม่ได้) เข้าร่วม cluster เป็น replica ตัวใหม่
```

ประเด็นสำคัญคือ **ทุกขั้นตอนหลัง T+40s (การเลือก replica, promote, re-routing) เป็นงานของ Operator ล้วน ๆ ไม่ใช่ของ Kubernetes core** — Kubernetes ทำหน้าที่แค่แจ้งว่า "Pod นี้หายไปแล้ว" ส่วนที่เหลือทั้งหมดคือ domain logic ของ PostgreSQL ที่ Operator เขียนขึ้นมาเสริม

### เปรียบเทียบเวลาการทำ Failover: Synchronous vs Asynchronous Replication

| โหมด Replication | Data Loss เมื่อ Failover | ความเร็วในการเขียนปกติ | เหมาะกับ |
|---|---|---|---|
| Asynchronous (default) | อาจสูญเสีย transaction ล่าสุดที่ยังไม่ถูก stream ไปถึง replica | เร็วที่สุด | ระบบที่ยอมรับ data loss เล็กน้อยได้ |
| Synchronous (1 replica) | ไม่มี data loss สำหรับ transaction ที่ commit สำเร็จ | ช้าลง (ต้องรอ replica ยืนยัน) | ระบบการเงิน/ธุรกรรมสำคัญ |
| Synchronous Quorum | ไม่มี data loss พร้อมความทนทานสูงขึ้น | ช้าที่สุด | ระบบที่ต้องการ durability สูงสุด |

ตัวอย่างการตั้งค่า synchronous replication ใน CNPG:

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: ecommerce-cluster-ha
  namespace: production
spec:
  instances: 3
  postgresql:
    synchronous:
      method: any
      number: 1   # ต้องมี replica อย่างน้อย 1 ตัวยืนยัน commit ก่อนถือว่าสำเร็จ
  storage:
    size: 100Gi
    storageClass: fast-ssd
```

### Automated Backup — เชื่อมโยงกับ Part 065

ใน Part 065 เราพูดถึง `pg_basebackup`, WAL archiving, และ PITR ด้วยมือ ส่วนบนนี้ Operator ทำให้เป็นกระบวนการอัตโนมัติทั้งหมด:

1. **Continuous WAL Archiving** — Operator ตั้งค่า `archive_command` ให้ส่ง WAL segment ไปยัง object storage (S3/GCS/Azure Blob) โดยอัตโนมัติทันทีที่ WAL segment เต็ม ไม่ต้องเขียน script เอง
2. **Scheduled Base Backup** — ผ่าน `ScheduledBackup` CRD (CNPG) หรือการตั้งค่า cron ภายนอก (Zalando) ทำ full base backup ตามตารางเวลาที่กำหนด
3. **Point-in-Time Recovery** — เมื่อต้องการกู้คืนข้อมูล ณ เวลาใดเวลาหนึ่ง เพียงสร้าง `Cluster` ใหม่โดยระบุ `bootstrap.recovery` ชี้ไปยัง backup เดิมพร้อม `recoveryTarget`

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: ecommerce-cluster-restored
  namespace: production
spec:
  instances: 3
  storage:
    size: 100Gi
    storageClass: fast-ssd
  bootstrap:
    recovery:
      source: ecommerce-cluster-backup-source
      recoveryTarget:
        targetTime: "2026-09-20 14:30:00+07"   # กู้คืนข้อมูล ณ เวลานี้
  externalClusters:
    - name: ecommerce-cluster-backup-source
      barmanObjectStore:
        destinationPath: "s3://ecommerce-pg-backups/production"
        s3Credentials:
          accessKeyId:
            name: backup-s3-credentials
            key: ACCESS_KEY_ID
          secretAccessKey:
            name: backup-s3-credentials
            key: SECRET_ACCESS_KEY
```

การประกาศ manifest แบบนี้เพียงไฟล์เดียว Operator จะทำการ:
- ดึง base backup ล่าสุดก่อน `targetTime` จาก object storage
- Replay WAL segment ต่อจนถึงเวลาที่ระบุแม่นยำ
- สร้าง cluster ใหม่ที่มีข้อมูล ณ จุดเวลานั้นพร้อมใช้งาน พร้อม replica ตามจำนวนที่กำหนด

### ทดสอบ Failover ด้วยตนเองบนสภาพแวดล้อมทดสอบ (เชิงแนวคิด)

```bash
# ตรวจสอบ Pod ไหนเป็น primary ปัจจุบัน
kubectl cnpg status ecommerce-cluster-ha -n production

# จำลองความล้มเหลว โดยลบ Pod primary ทิ้งโดยตรง (บน test cluster เท่านั้น)
kubectl delete pod ecommerce-cluster-ha-1 -n production --grace-period=0 --force

# สังเกตว่า Operator promote replica ตัวใหม่ขึ้นมาแทนภายในไม่กี่วินาที
kubectl get pods -n production -l cnpg.io/cluster=ecommerce-cluster-ha -w

# ตรวจสอบ event log ของ Operator เพื่อดูรายละเอียดขั้นตอน failover
kubectl get events -n production --field-selector involvedObject.name=ecommerce-cluster-ha \
  --sort-by='.lastTimestamp'
```

**ข้อควรระวัง**: การทดสอบ failover แบบ force delete Pod ควรทำบน staging/test environment เท่านั้น ห้ามทดสอบบน production cluster โดยไม่วางแผน เพราะแม้ Operator จะจัดการ failover ได้อัตโนมัติ แต่ก็ยังมี window เวลาสั้น ๆ (มักไม่กี่วินาทีถึงหลักสิบวินาที) ที่ระบบไม่สามารถรับ write ได้ระหว่างกระบวนการ promote

---

## Step 930: แบบฝึกหัดรวม — Deploy ระบบ E-commerce แบบ Production-ready

โจทย์: ออกแบบและเขียน manifest สำหรับระบบ e-commerce ที่ต้องรองรับทั้งสภาพแวดล้อมการพัฒนา (local, ผ่าน Docker Compose) และสภาพแวดล้อม production (Kubernetes ผ่าน CloudNativePG) โดยมีความต้องการดังนี้:

- Database `ecommerce` พร้อม schema เริ่มต้นสำหรับ products/orders
- High Availability ระดับ production (3 instances, synchronous replication อย่างน้อย 1 replica)
- Automated backup รายวัน พร้อม retention 30 วัน ไปยัง S3
- Connection pooling แยกทั้ง read-write และ read-only
- Resource limit ที่เหมาะสมกับ workload ระดับกลาง (moderate traffic e-commerce)
- Monitoring ผ่าน Prometheus

### ส่วนที่ 1: Docker Compose สำหรับ Local Development (รันได้จริง)

```yaml
version: "3.9"

services:
  postgres:
    image: postgres:16.4
    container_name: ecommerce-dev-postgres
    restart: unless-stopped
    environment:
      POSTGRES_USER: appuser
      POSTGRES_PASSWORD: dev_password_only
      POSTGRES_DB: ecommerce
      PGDATA: /var/lib/postgresql/data/pgdata
    volumes:
      - pgdata_dev:/var/lib/postgresql/data
      - ./db/init:/docker-entrypoint-initdb.d:ro
    ports:
      - "127.0.0.1:5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U appuser -d ecommerce"]
      interval: 5s
      timeout: 5s
      retries: 10

  pgbouncer:
    image: edoburu/pgbouncer:1.21.0
    container_name: ecommerce-dev-pgbouncer
    restart: unless-stopped
    depends_on:
      postgres:
        condition: service_healthy
    environment:
      DATABASE_URL: postgres://appuser:dev_password_only@postgres:5432/ecommerce
      POOL_MODE: transaction
      MAX_CLIENT_CONN: 200
      DEFAULT_POOL_SIZE: 20
    ports:
      - "6432:6432"

  redis:
    image: redis:7.4-alpine
    container_name: ecommerce-dev-redis
    restart: unless-stopped
    command: ["redis-server", "--maxmemory", "128mb", "--maxmemory-policy", "allkeys-lru"]
    volumes:
      - redisdata_dev:/data

  app:
    build:
      context: ./app
    container_name: ecommerce-dev-app
    restart: unless-stopped
    depends_on:
      - pgbouncer
      - redis
    environment:
      DATABASE_URL: postgres://appuser:dev_password_only@pgbouncer:6432/ecommerce
      REDIS_URL: redis://redis:6379
    ports:
      - "3000:3000"

volumes:
  pgdata_dev:
  redisdata_dev:
```

### ส่วนที่ 2: Kubernetes Manifest ด้วย CloudNativePG (อ้างอิงเชิงสถาปัตยกรรม)

```yaml
# ── namespace.yaml ──────────────────────────────────────────
apiVersion: v1
kind: Namespace
metadata:
  name: ecommerce-prod
---
# ── secrets.yaml ────────────────────────────────────────────
apiVersion: v1
kind: Secret
metadata:
  name: ecommerce-db-credentials
  namespace: ecommerce-prod
type: kubernetes.io/basic-auth
stringData:
  username: appuser
  password: "USE_A_REAL_SECRET_MANAGER_IN_PRODUCTION"
---
apiVersion: v1
kind: Secret
metadata:
  name: ecommerce-backup-s3-credentials
  namespace: ecommerce-prod
type: Opaque
stringData:
  ACCESS_KEY_ID: "REPLACE_WITH_REAL_KEY"
  SECRET_ACCESS_KEY: "REPLACE_WITH_REAL_SECRET"
---
# ── cluster.yaml ────────────────────────────────────────────
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: ecommerce-pg
  namespace: ecommerce-prod
spec:
  description: "PostgreSQL cluster สำหรับระบบ e-commerce production"
  instances: 3

  imageName: ghcr.io/cloudnative-pg/postgresql:16.4

  postgresql:
    parameters:
      shared_buffers: "2GB"
      effective_cache_size: "6GB"
      max_connections: "300"
      work_mem: "16MB"
      maintenance_work_mem: "512MB"
      log_min_duration_statement: "500"
      wal_level: "replica"
      max_wal_size: "4GB"
      checkpoint_completion_target: "0.9"
      random_page_cost: "1.1"   # ใช้ SSD-based storage class
    synchronous:
      method: any
      number: 1

  bootstrap:
    initdb:
      database: ecommerce
      owner: appuser
      secret:
        name: ecommerce-db-credentials
      dataChecksums: true
      encoding: UTF8
      postInitApplicationSQL:
        - "CREATE SCHEMA IF NOT EXISTS shop;"
        - |
          CREATE TABLE IF NOT EXISTS shop.products (
              id          bigserial PRIMARY KEY,
              sku         text NOT NULL UNIQUE,
              name        text NOT NULL,
              price_cents integer NOT NULL CHECK (price_cents >= 0),
              created_at  timestamptz NOT NULL DEFAULT now()
          );
        - |
          CREATE TABLE IF NOT EXISTS shop.orders (
              id          bigserial PRIMARY KEY,
              customer_id bigint NOT NULL,
              status      text NOT NULL DEFAULT 'pending',
              total_cents integer NOT NULL DEFAULT 0,
              created_at  timestamptz NOT NULL DEFAULT now()
          );
        - "CREATE INDEX IF NOT EXISTS idx_orders_customer ON shop.orders (customer_id);"

  storage:
    size: 200Gi
    storageClass: fast-ssd

  resources:
    requests:
      cpu: "2"
      memory: "4Gi"
    limits:
      cpu: "8"
      memory: "8Gi"

  affinity:
    enablePodAntiAffinity: true
    topologyKey: topology.kubernetes.io/zone
    podAntiAffinityType: required   # บังคับให้ทุก Pod อยู่คนละ zone จริง ๆ

  primaryUpdateStrategy: unsupervised   # rolling upgrade อัตโนมัติเมื่อ image เปลี่ยน

  backup:
    barmanObjectStore:
      destinationPath: "s3://ecommerce-prod-pg-backups"
      s3Credentials:
        accessKeyId:
          name: ecommerce-backup-s3-credentials
          key: ACCESS_KEY_ID
        secretAccessKey:
          name: ecommerce-backup-s3-credentials
          key: SECRET_ACCESS_KEY
      wal:
        compression: gzip
        maxParallel: 8
      data:
        compression: gzip
        jobs: 8
    retentionPolicy: "30d"

  monitoring:
    enablePodMonitor: true
---
# ── scheduled-backup.yaml ───────────────────────────────────
apiVersion: postgresql.cnpg.io/v1
kind: ScheduledBackup
metadata:
  name: ecommerce-pg-daily-backup
  namespace: ecommerce-prod
spec:
  schedule: "0 2 * * *"
  backupOwnerReference: self
  cluster:
    name: ecommerce-pg
---
# ── pooler-rw.yaml ──────────────────────────────────────────
apiVersion: postgresql.cnpg.io/v1
kind: Pooler
metadata:
  name: ecommerce-pg-pooler-rw
  namespace: ecommerce-prod
spec:
  cluster:
    name: ecommerce-pg
  instances: 2
  type: rw
  pgbouncer:
    poolMode: transaction
    parameters:
      max_client_conn: "1000"
      default_pool_size: "40"
  template:
    spec:
      containers:
        - name: pgbouncer
          resources:
            requests:
              cpu: "200m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
---
# ── pooler-ro.yaml ──────────────────────────────────────────
apiVersion: postgresql.cnpg.io/v1
kind: Pooler
metadata:
  name: ecommerce-pg-pooler-ro
  namespace: ecommerce-prod
spec:
  cluster:
    name: ecommerce-pg
  instances: 2
  type: ro
  pgbouncer:
    poolMode: transaction
    parameters:
      max_client_conn: "1000"
      default_pool_size: "60"
---
# ── app-deployment.yaml (ตัวอย่าง application ฝั่ง stateless) ─
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ecommerce-app
  namespace: ecommerce-prod
spec:
  replicas: 4
  selector:
    matchLabels:
      app: ecommerce-app
  template:
    metadata:
      labels:
        app: ecommerce-app
    spec:
      containers:
        - name: app
          image: registry.example.com/ecommerce-app:1.4.2
          env:
            - name: DATABASE_URL_RW
              value: "postgres://appuser@ecommerce-pg-pooler-rw:5432/ecommerce"
            - name: DATABASE_URL_RO
              value: "postgres://appuser@ecommerce-pg-pooler-ro:5432/ecommerce"
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: ecommerce-db-credentials
                  key: password
          resources:
            requests:
              cpu: "250m"
              memory: "256Mi"
            limits:
              cpu: "1"
              memory: "512Mi"
          ports:
            - containerPort: 3000
---
apiVersion: v1
kind: Service
metadata:
  name: ecommerce-app-svc
  namespace: ecommerce-prod
spec:
  selector:
    app: ecommerce-app
  ports:
    - port: 80
      targetPort: 3000
  type: ClusterIP
```

### คำสั่ง Deploy ทั้งชุด (เชิงแนวคิด)

```bash
kubectl apply -f namespace.yaml
kubectl apply -f secrets.yaml
kubectl apply -f cluster.yaml
kubectl apply -f scheduled-backup.yaml
kubectl apply -f pooler-rw.yaml
kubectl apply -f pooler-ro.yaml
kubectl apply -f app-deployment.yaml

# ตรวจสอบสถานะทั้งหมด
kubectl cnpg status ecommerce-pg -n ecommerce-prod
kubectl get pods,svc,pooler -n ecommerce-prod
```

Manifest ชุดนี้แสดงให้เห็นภาพรวมของระบบ production-ready ที่ครบวงจร ตั้งแต่ database cluster ที่มี HA, automated backup, connection pooling แยก read/write, ไปจนถึง application layer ที่เชื่อมต่อผ่าน pooler อย่างถูกต้อง

---

## สรุปท้ายบท

บทนี้พาผู้เรียนเดินทางจากการรัน PostgreSQL บน Docker แบบพื้นฐาน ไปจนถึงการ deploy ระดับ production บน Kubernetes ด้วย Operator ประเด็นสำคัญที่ควรจดจำ:

1. **Docker**: ข้อมูลของ PostgreSQL ต้องอยู่บน named volume เสมอ container คือของชั่วคราว แต่ volume คือที่เก็บข้อมูลจริง
2. **Docker Compose**: การประกอบ stack ที่มี PostgreSQL + PgBouncer + Redis + App ควรใช้ healthcheck และ `depends_on.condition` เพื่อควบคุมลำดับการเริ่มทำงาน
3. **Kubernetes ไม่ใช่ที่ที่เหมาะกับ stateful workload โดยธรรมชาติ** — ต้องอาศัย StatefulSet เพื่อแก้ปัญหาเรื่อง identity และ storage เป็นพื้นฐาน
4. **StatefulSet เพียงอย่างเดียวไม่พอ** — ขาด domain logic สำหรับ failover, backup, และ replication management ของ PostgreSQL โดยเฉพาะ
5. **Operator Pattern** คือทางออกที่ encode ความรู้ของ DBA ให้เป็นโค้ดอัตโนมัติ ผ่าน CRD (ประกาศ desired state) และ Controller (reconciliation loop)
6. **Zalando Operator** พึ่งพา Patroni เป็นแกนหลักของ HA เหมาะกับทีมที่คุ้นเคยกับ Patroni อยู่แล้ว
7. **CloudNativePG** ออกแบบใหม่ทั้งหมดให้เป็น cloud-native ล้วน ไม่พึ่ง external dependency เป็นทิศทางที่ได้รับความนิยมมากขึ้นเรื่อย ๆ ในโปรเจกต์ใหม่
8. **Automated Failover และ Backup** บน Kubernetes คืองานของ Operator ไม่ใช่ Kubernetes core — Kubernetes แค่บอกว่า Pod หายไป ส่วนการตัดสินใจ promote/backup เป็นหน้าที่ของ Operator ทั้งหมด

### ตารางเปรียบเทียบสรุป Zalando vs CloudNativePG

| มิติ | Zalando Postgres Operator | CloudNativePG |
|---|---|---|
| กลไก HA | Patroni + DCS | Native Kubernetes (ไม่พึ่ง external tool) |
| ความซับซ้อนสถาปัตยกรรม | สูงกว่า | ต่ำกว่า |
| Backup engine | WAL-G | Barman Cloud (built-in) |
| Scheduled Backup CRD | ไม่มีโดยตรง | มี (`ScheduledBackup`) |
| Web UI | มี | ไม่มี (ใช้ `kubectl cnpg` แทน) |
| Governance | Open source อิสระ (Zalando) | CNCF Sandbox |
| เหมาะกับ | ทีมที่คุ้นเคย Patroni | โปรเจกต์ใหม่ที่ต้องการความเรียบง่าย cloud-native |
| แนวโน้มความนิยม (2024-2026) | คงที่/ค่อย ๆ ลดลง | เติบโตรวดเร็ว |

---

## แบบฝึกหัด

<details>
<summary><strong>แบบฝึกหัดที่ 1:</strong> เหตุใดการรัน `docker run postgres` โดยไม่ระบุ `-v` ใด ๆ เลยจึงเป็นความเสี่ยงร้ายแรง แม้ image จะประกาศ `VOLUME` ไว้ใน Dockerfile ก็ตาม?</summary>

แม้ image `postgres` จะประกาศ `VOLUME /var/lib/postgresql/data` ไว้ใน Dockerfile ซึ่งทำให้ Docker สร้าง **anonymous volume** ให้อัตโนมัติเมื่อไม่มีการ mount ใด ๆ ระบุไว้ ข้อมูลจึงไม่ได้หายไปทันทีที่ container หยุดทำงาน แต่ความเสี่ยงคือ:

1. Anonymous volume มีชื่อเป็น hash แบบสุ่ม ยากต่อการอ้างอิงหรือค้นหาในภายหลัง
2. หากรัน `docker rm -v <container>` (ซึ่งเป็นคำสั่งที่คนมักใช้เพื่อ "ล้าง" container โดยไม่ทันสังเกตว่ามี `-v`) anonymous volume ที่ผูกกับ container นั้นจะถูกลบไปพร้อมกันทันที
3. ไม่มีการตั้งชื่อที่จำง่ายสำหรับ backup/restore หรือ migrate ไปเครื่องอื่น
4. เมื่อรัน `docker-compose down -v` โดยไม่ได้ตั้งใจ หรือใช้ `docker system prune --volumes` ข้อมูลจะหายไปโดยไม่มีการเตือนที่ชัดเจนพอ

ทางออกที่ปลอดภัยคือใช้ **named volume** ที่ตั้งชื่อไว้ชัดเจน (เช่น `pgdata_prod`) เพื่อให้ควบคุมและติดตามได้ง่าย
</details>

<details>
<summary><strong>แบบฝึกหัดที่ 2:</strong> ในไฟล์ Docker Compose ของ Step 922 เหตุใด application service จึงเชื่อมต่อผ่าน PgBouncer (`pgbouncer:6432`) แทนที่จะเชื่อมต่อ PostgreSQL โดยตรง (`postgres:5432`)?</summary>

เหตุผลหลักคือการจัดการ connection อย่างมีประสิทธิภาพ:

1. PostgreSQL แต่ละ connection ใช้ process แยกต่างหาก (process-per-connection model) ซึ่งมี overhead ด้าน memory และ CPU สูงเมื่อมี connection จำนวนมาก
2. เมื่อ scale application ขึ้นหลาย instance (เช่นใน production ที่มี app หลาย replica) จำนวน connection ที่ยิงตรงไปยัง PostgreSQL จะเพิ่มขึ้นรวดเร็วจนอาจชน `max_connections`
3. PgBouncer ทำหน้าที่เป็น connection pooler กลาง รวบรวม connection จาก application จำนวนมากให้เหลือ connection จริงไปยัง PostgreSQL จำนวนน้อยกว่ามาก (ผ่าน `pool_mode: transaction`)
4. ทำให้ PostgreSQL server ไม่ต้องรับภาระจัดการ connection จำนวนมากโดยตรง ลด memory overhead และเพิ่มความเสถียรของระบบโดยรวม

การเชื่อมต่อผ่าน pooler จึงเป็น best practice มาตรฐานสำหรับ production workload ตามที่กล่าวถึงใน Part 072-073
</details>

<details>
<summary><strong>แบบฝึกหัดที่ 3:</strong> อธิบายว่าทำไม Deployment ธรรมดาใน Kubernetes ไม่เหมาะกับการรัน PostgreSQL cluster ที่มี primary และ replica</summary>

Deployment ถูกออกแบบมาสำหรับ Pod ที่ **เหมือนกันทุกตัวและทดแทนกันได้ทันที (interchangeable/stateless)**:

1. Deployment ไม่รับประกันชื่อ Pod ที่คงที่ — ชื่อ Pod จะเป็น random suffix ทุกครั้งที่สร้างใหม่ ทำให้ไม่สามารถระบุได้แน่ชัดว่า Pod ไหนคือ primary Pod ไหนคือ replica
2. Deployment ไม่มี `volumeClaimTemplates` — ทุก replica ที่ใช้ PVC เดียวกันจะพยายามเขียนทับ data directory เดียวกัน (ซึ่ง PostgreSQL ไม่รองรับหลาย instance เขียนพร้อมกัน) หรือถ้าใช้ PVC คนละตัว ก็ไม่มีกลไกให้ Pod ที่สร้างใหม่ได้ PVC เดิมกลับมาอย่างแน่นอน
3. ไม่มี stable network identity — replica ไม่สามารถระบุ hostname ที่แน่นอนของ primary เพื่อตั้งค่า `primary_conninfo` ได้อย่างเสถียร
4. ไม่มีลำดับการสร้าง/ลบที่แน่นอน (ordered creation) ซึ่งสำคัญเมื่อ replica ต้องรอให้ primary พร้อมก่อนจึงจะ clone ข้อมูลได้

ด้วยเหตุนี้ต้องใช้ **StatefulSet** ซึ่งให้ stable identity, dedicated storage ต่อ Pod ผ่าน `volumeClaimTemplates`, และ ordered deployment แทน
</details>

<details>
<summary><strong>แบบฝึกหัดที่ 4:</strong> Reconciliation loop ของ Kubernetes Operator ทำงานอย่างไร จงอธิบายเป็นขั้นตอน พร้อมยกตัวอย่างเหตุการณ์ที่กระตุ้นให้ loop นี้ทำงาน</summary>

Reconciliation loop ทำงานเป็นวงจรต่อเนื่อง 4 ขั้นตอนหลัก:

1. **Observe** — Controller watch (ผ่าน Kubernetes API watch mechanism) การเปลี่ยนแปลงของ Custom Resource (เช่น `Cluster`) และ object ที่เกี่ยวข้อง (Pod, PVC, Service)
2. **Diff** — เปรียบเทียบ desired state (สิ่งที่ประกาศไว้ใน YAML เช่น `instances: 3`) กับ actual state (สิ่งที่มีอยู่จริงใน cluster ขณะนั้น เช่นมี Pod ทำงานอยู่แค่ 2 ตัว)
3. **Act** — ดำเนินการเพื่อลด "ความต่าง" ระหว่าง desired กับ actual เช่นสร้าง Pod เพิ่มอีก 1 ตัว หรือสั่ง promote replica เมื่อ primary หาย
4. **Repeat** — วนกลับไปขั้นตอน Observe อีกครั้งอย่างต่อเนื่องไม่มีที่สิ้นสุด

ตัวอย่างเหตุการณ์ที่กระตุ้น loop:
- ผู้ใช้แก้ไข `instances: 3` เป็น `instances: 5` ใน manifest แล้ว `kubectl apply` — controller ตรวจพบ diff แล้วสร้าง Pod ใหม่ 2 ตัวพร้อม clone ข้อมูล
- Pod ของ primary ตายกะทันหัน — controller ตรวจพบว่า actual state ไม่มี primary แล้ว จึงเลือกและ promote replica ตัวใหม่
- PVC เต็มพื้นที่ (ในกรณีที่รองรับ auto-resize) — controller ตรวจพบและขยาย storage ให้อัตโนมัติ
</details>

<details>
<summary><strong>แบบฝึกหัดที่ 5:</strong> ในตัวอย่าง manifest ของ Zalando Postgres Operator (Step 926) field ใดที่ทำให้ Operator สร้าง pgbouncer sidecar ให้อัตโนมัติ และมีข้อดีอย่างไรเมื่อเทียบกับการตั้งค่า pooler เอง?</summary>

Field ที่เกี่ยวข้องคือ:

```yaml
enableConnectionPooler: true
connectionPooler:
  numberOfInstances: 2
  mode: "transaction"
  maxDBConnections: 60
```

เมื่อตั้งค่านี้ Operator จะสร้างและจัดการ pgbouncer deployment แยกต่างหากให้อัตโนมัติ พร้อมเชื่อมต่อกับ PostgreSQL cluster ที่ถูกต้องโดยไม่ต้องเขียน manifest ของ pgbouncer เองแยกต่างหาก (ต่างจาก Docker Compose ที่ต้องเขียน service `pgbouncer` เองทั้งหมด)

ข้อดีเมื่อเทียบกับการตั้งค่าเอง:
1. Operator จัดการ credential และ connection string ให้ตรงกับ cluster โดยอัตโนมัติ ลดโอกาสตั้งค่าผิดพลาด
2. เมื่อ cluster มีการ failover (primary เปลี่ยน) pooler จะถูกอัปเดตให้ชี้ไปยัง primary ใหม่โดย Operator เอง ไม่ต้องตั้งค่าใหม่ด้วยมือ
3. จำนวน instance ของ pooler ปรับ scale ได้ง่ายผ่าน field เดียว (`numberOfInstances`)
4. ลดจำนวนไฟล์ manifest ที่ต้องดูแลแยกต่างหาก
</details>

<details>
<summary><strong>แบบฝึกหัดที่ 6:</strong> CloudNativePG ต่างจาก Zalando Operator อย่างไรในเรื่องการทำ Leader Election เพราะเหตุใดจึงถือว่า CNPG มี "moving part" น้อยกว่า?</summary>

**Zalando Operator**: ใช้ Patroni เป็นตัวจัดการ leader election โดย Patroni แต่ละ instance จะแข่งกันถือ "lock" ผ่าน DCS (Distributed Configuration Store) ซึ่งอาจเป็น Kubernetes API เอง, etcd, หรือ Consul — Patroni เป็น external tool ที่ต้องรันเป็น process แยกอยู่ภายใน Pod แต่ละตัว (ทำงานร่วมกับ PostgreSQL แต่เป็นคนละ codebase)

**CloudNativePG**: เขียน logic การทำ leader election ขึ้นใหม่ทั้งหมดโดยทีม CNPG เอง ผ่าน **instance manager** ที่ built-in ไปกับ container image และใช้ Kubernetes native mechanism (เช่น lease object) เป็นกลไกกลางโดยตรง ไม่ต้องพึ่งพา external DCS อย่าง etcd หรือ Consul เพิ่มเติม

เหตุผลที่ CNPG มี moving part น้อยกว่า:
- ไม่ต้องติดตั้ง/ดูแล DCS แยกต่างหาก (ในกรณีที่ไม่ใช้ Kubernetes API เป็น DCS)
- ไม่มี codebase ของ Patroni เป็นชั้นเพิ่มเติมที่ต้อง debug เมื่อเกิดปัญหา — logic ทั้งหมดอยู่ใน instance manager ตัวเดียวที่ทีม CNPG ควบคุมและพัฒนาเอง
- ลดจำนวนจุดที่อาจเกิดความล้มเหลว (failure point) เนื่องจากระบบพึ่งพา component ภายนอกน้อยกว่า
</details>

<details>
<summary><strong>แบบฝึกหัดที่ 7:</strong> ในกรณีที่องค์กรต้องการทำ Point-in-Time Recovery (PITR) บน Kubernetes ด้วย CloudNativePG ต้องประกาศ field ใดใน `Cluster` CRD และมีขั้นตอนอย่างไร?</summary>

ต้องใช้ field `bootstrap.recovery` พร้อม `recoveryTarget` และประกาศ `externalClusters` เพื่อชี้ไปยังแหล่ง backup เดิม ดังตัวอย่างใน Step 929:

```yaml
spec:
  bootstrap:
    recovery:
      source: ecommerce-cluster-backup-source
      recoveryTarget:
        targetTime: "2026-09-20 14:30:00+07"
  externalClusters:
    - name: ecommerce-cluster-backup-source
      barmanObjectStore:
        destinationPath: "s3://ecommerce-pg-backups/production"
        s3Credentials: { ... }
```

ขั้นตอนที่เกิดขึ้นภายในเมื่อ apply manifest นี้:
1. Operator สร้าง `Cluster` ใหม่ (คนละชื่อกับ cluster เดิม)
2. ดึง base backup ล่าสุดที่ทำก่อน `targetTime` จาก object storage ที่ระบุใน `externalClusters`
3. Restore base backup ลง data directory ของ instance แรก
4. Replay WAL segment ที่เก็บไว้ต่อจนถึงเวลาที่ระบุใน `targetTime` อย่างแม่นยำ (ใช้กลไก `recovery_target_time` ของ PostgreSQL เอง)
5. เมื่อ recovery เสร็จสมบูรณ์ cluster ใหม่จะพร้อมใช้งานที่สถานะข้อมูล ณ เวลาที่ระบุ พร้อม replica ตามจำนวน `instances` ที่กำหนด

วิธีนี้ทำให้การกู้คืนข้อมูลที่เคยต้องทำด้วยมือหลายขั้นตอน (ตามที่อธิบายใน Part 065) กลายเป็นการประกาศ manifest เพียงไฟล์เดียว
</details>

<details>
<summary><strong>แบบฝึกหัดที่ 8:</strong> เหตุใดการทดสอบ Automatic Failover ด้วยการ `kubectl delete pod <primary> --force` จึงไม่ควรทำบน production cluster โดยไม่วางแผนล่วงหน้า?</summary>

แม้ Operator จะจัดการ failover ได้อัตโนมัติ แต่การทดสอบแบบ force delete บน production มีความเสี่ยงหลายประการ:

1. **มี downtime window จริง** — แม้กระบวนการ failover จะเร็ว (มักไม่กี่วินาทีถึงหลักสิบวินาที) แต่ในช่วงเวลานั้นระบบไม่สามารถรับ write ได้ ถ้าเป็น production ที่มี traffic จริง อาจทำให้ transaction ล้มเหลวหรือ request ของผู้ใช้จริงเกิด error
2. **ความเสี่ยงด้าน data loss** — หากใช้ asynchronous replication (ไม่ใช่ synchronous) transaction ล่าสุดที่ยังไม่ถูก stream ไปถึง replica ที่ถูกเลือกเป็น primary ใหม่อาจสูญหาย
3. **ผลกระทบต่อ connection pooler และ application** — application ที่เชื่อมต่อผ่าน connection pool อาจได้รับ error ชั่วคราวระหว่าง reconnect ไปยัง primary ใหม่ หากไม่มี retry logic ที่ดีพออาจกระทบผู้ใช้จริง
4. **ไม่สามารถคาดเดาผลกระทบข้างเคียงได้ทั้งหมด** — เช่น cascading effect กับระบบ monitoring, alerting, หรือ downstream service ที่ผูกกับ database

ควรทดสอบบน staging/test environment ที่จำลองสภาพใกล้เคียง production ก่อนเสมอ และหากต้องการทดสอบบน production จริง ควรวางแผน (planned failover / manual switchover) แจ้งทีมที่เกี่ยวข้อง และเลือกช่วงเวลาที่ traffic ต่ำ
</details>

<details>
<summary><strong>แบบฝึกหัดที่ 9:</strong> ในตัวอย่าง manifest แบบฝึกหัดรวม (Step 930) เหตุใดจึงแยก Service เป็น `ecommerce-pg-pooler-rw` และ `ecommerce-pg-pooler-ro` แทนที่จะใช้ Service เดียวสำหรับทุก operation?</summary>

การแยก pooler เป็น read-write (`rw`) และ read-only (`ro`) สะท้อนหลักการ **read/write splitting** ที่สำคัญมากสำหรับระบบที่มี replica หลายตัว:

1. **`ecommerce-pg-pooler-rw`** (type: `rw`) — ชี้ไปยัง primary instance เท่านั้น ใช้สำหรับ operation ที่ต้องเขียนข้อมูล (INSERT/UPDATE/DELETE) เพราะมีเพียง primary เท่านั้นที่รับ write ได้
2. **`ecommerce-pg-pooler-ro`** (type: `ro`) — ชี้ไปยัง replica เท่านั้น (load balance ระหว่าง replica ที่มีอยู่) ใช้สำหรับ query ที่อ่านข้อมูลอย่างเดียว เช่นการแสดงรายการสินค้า, รายงาน, dashboard

ข้อดีของการแยกแบบนี้:
- **กระจายภาระการอ่าน** ออกจาก primary ไปยัง replica หลายตัว ลด load ที่ primary ต้องรับ ทำให้ primary มีทรัพยากรเหลือสำหรับ write operation ที่สำคัญกว่า
- **Application สามารถเลือก connection string ตามประเภท operation** ได้อย่างชัดเจน (ดังที่เห็นใน `app-deployment.yaml` ที่มีทั้ง `DATABASE_URL_RW` และ `DATABASE_URL_RO`)
- **Scale การอ่านได้อิสระจากการเขียน** — หาก traffic การอ่านสูงมาก สามารถเพิ่มจำนวน replica (และ `instances` ของ pooler ro) โดยไม่กระทบ primary
- ป้องกันการเขียนข้อมูลผิดพลาดไปยัง replica โดยไม่ตั้งใจ เพราะ replica ปฏิเสธ write อยู่แล้วโดยธรรมชาติของ PostgreSQL แต่การแยก endpoint ชัดเจนช่วยให้ developer เข้าใจเจตนาของ query ได้ง่ายขึ้นด้วย
</details>

<details>
<summary><strong>แบบฝึกหัดที่ 10:</strong> จงอธิบายว่าเหตุใด `affinity.podAntiAffinityType: required` พร้อม `topologyKey: topology.kubernetes.io/zone` ใน manifest ของ CNPG (Step 930) จึงสำคัญต่อความทนทานของระบบต่อความล้มเหลวระดับ data center</summary>

การตั้งค่านี้บังคับให้ **แต่ละ Pod ของ PostgreSQL cluster (primary และ replica ทุกตัว) ต้องถูกจัดวางไว้คนละ availability zone กันเสมอ** (ไม่ใช่แค่คนละโหนด แต่คนละ zone ทางภูมิศาสตร์/โครงสร้างพื้นฐานที่แยกจากกัน)

เหตุผลที่สำคัญ:

1. **ความทนทานต่อความล้มเหลวระดับ zone** — หากทุก Pod ถูกจัดวางไว้ใน zone เดียวกันโดยบังเอิญ (เช่น scheduler เลือกโหนดในโซนเดียวกันทั้งหมดเพราะมีทรัพยากรว่างพอ) เมื่อ zone นั้นเกิดปัญหา (เช่น network partition, power outage ของ data center นั้น) ทั้ง primary และ replica ทุกตัวจะล่มพร้อมกัน ทำให้ cluster ทั้งหมดใช้งานไม่ได้ และไม่มี replica เหลือให้ promote เป็น primary ใหม่เลย
2. **`required` (แทนที่จะเป็น `preferred`)** — บังคับอย่างเข้มงวดว่า Kubernetes scheduler **ต้อง** จัดวาง Pod คนละ zone เท่านั้น หากไม่มีโหนดว่างในต่างโซนเพียงพอ Pod จะค้างสถานะ `Pending` แทนที่จะยอมจัดวางไว้ zone เดียวกัน (ซึ่งต่างจาก `preferred` ที่เป็นเพียงคำแนะนำ scheduler อาจฝ่าฝืนได้หากจำเป็น)
3. **สอดคล้องกับหลักการ Multi-AZ ที่กล่าวถึงใน Part 065** — การกระจาย replica ข้าม availability zone คือรากฐานสำคัญของสถาปัตยกรรม HA ระดับ cloud ไม่ว่าจะรันบน VM ตรง ๆ หรือบน Kubernetes ก็ตามหลักการเดียวกันนี้ยังคงสำคัญเสมอ

หากไม่ตั้งค่า anti-affinity นี้ไว้ ระบบอาจดูเหมือนมี HA ครบ (3 instances, automated failover) แต่ในความเป็นจริงมีจุดล้มเหลวร่วม (single point of failure) ซ่อนอยู่ที่ระดับ physical infrastructure ซึ่งขัดกับเจตนารมณ์ของการทำ High Availability ตั้งแต่ต้น
</details>

---

## บทถัดไป

บทถัดไปจะพาผู้เรียนไปสำรวจการใช้งาน PostgreSQL บน **AWS RDS และ Aurora** ซึ่งเป็นอีกแนวทางหนึ่งของการรัน PostgreSQL แบบ managed service บน cloud โดยไม่ต้องดูแล infrastructure ระดับ container/Kubernetes เอง — เหมาะสำหรับเปรียบเทียบข้อดีข้อเสียกับแนวทาง self-managed ที่เรียนไปในบทนี้

**อ่านต่อ:** [Part 094: AWS RDS และ Aurora](./part-094-aws-rds-aurora.md)
