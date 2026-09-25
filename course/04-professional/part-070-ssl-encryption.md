# Part 070: SSL/TLS Configuration และ Encryption

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับมืออาชีพ | Part 070

---

## เป้าหมายการเรียนรู้

เมื่อจบบทนี้ ผู้เรียนจะสามารถ:

- อธิบายความแตกต่างระหว่าง **Encryption in Transit** และ **Encryption at Rest** และเข้าใจว่าทำไมระบบ e-commerce ต้องใช้ทั้งสองแบบร่วมกัน
- เปิดใช้งาน SSL/TLS บน PostgreSQL server ได้ด้วยตนเอง ทั้งบน production และ environment สำหรับทดสอบ
- สร้าง self-signed certificate ด้วย `openssl` เพื่อทดสอบ และเข้าใจความแตกต่างจาก CA-signed certificate ที่ใช้ใน production จริง
- กำหนดค่า `pg_hba.conf` ให้บังคับการเชื่อมต่อผ่าน SSL เท่านั้น ด้วย `hostssl` และปฏิเสธการเชื่อมต่อแบบไม่เข้ารหัสด้วย `hostnossl`
- เลือกระดับ `sslmode` ที่เหมาะสมฝั่ง client ตั้งแต่ `disable` จนถึง `verify-full` และเข้าใจว่าระดับไหนป้องกัน MITM (Man-in-the-Middle) ได้จริง
- ตั้งค่า **Client Certificate Authentication** เพื่อยืนยันตัวตนด้วย certificate แทนหรือเสริม password
- เข้าใจแนวทาง **Encryption at Rest** ใน PostgreSQL ว่าทำไมไม่มี TDE (Transparent Data Encryption) built-in และมีทางเลือกอะไรบ้าง (LUKS, cloud-managed encryption)
- ใช้ `pgcrypto` ทำ **Column-level Encryption** สำหรับข้อมูลอ่อนไหวเฉพาะจุด เช่น เลขบัตรเครดิต หรือเลขบัตรประชาชน
- ออกแบบกลยุทธ์ encryption แบบครบวงจรสำหรับระบบ e-commerce ที่ต้องปกป้องทั้งข้อมูลลูกค้า (PII) และข้อมูลการชำระเงิน (payment data)

### บริบทของบทนี้

ร้านค้าออนไลน์ที่เราใช้เป็นตัวอย่างตลอดหลักสูตรมีข้อมูลอ่อนไหวอยู่หลายชั้น: ข้อมูลลูกค้า (`customers`) ที่มีอีเมล เบอร์โทร ที่อยู่จัดส่ง, ข้อมูลการชำระเงิน (`payments`, `payment_methods`) ที่อาจมีเลขบัตรบางส่วนหรือ token จาก payment gateway, และ session/token การเข้าสู่ระบบ หากข้อมูลเหล่านี้รั่วไหลระหว่างทาง (เช่น แอปพลิเคชันเชื่อมต่อฐานข้อมูลผ่านเครือข่ายสาธารณะ) หรือรั่วไหลจาก disk/backup ที่ถูกขโมย ความเสียหายทั้งทางกฎหมาย (PDPA, PCI-DSS) และความเชื่อมั่นของลูกค้าจะรุนแรงมาก บทนี้จะพาไปตั้งค่าการป้องกันทั้งสองแนวรบ

---

## Step 691: Encryption in Transit คืออะไร

**Encryption in Transit** (หรือบางครั้งเรียกว่า Encryption in Motion) คือการเข้ารหัสข้อมูลระหว่างที่มันเดินทางผ่านเครือข่าย จาก client ไปยัง server หรือระหว่าง server ต่อ server เป้าหมายคือป้องกันไม่ให้ผู้ไม่หวังดีที่ดักฟังการจราจรบนเครือข่าย (packet sniffing) อ่านเนื้อหาที่แท้จริงได้

### ทำไมมันสำคัญกับฐานข้อมูล

โดย default การเชื่อมต่อ PostgreSQL ผ่าน TCP/IP จะส่งข้อมูล**แบบข้อความเปล่า (plaintext)** หมายความว่า:

- Query SQL ที่ส่งไป เช่น `SELECT credit_card_number FROM payment_methods WHERE customer_id = 123`
- ผลลัพธ์ที่ส่งกลับมา รวมถึงข้อมูลลูกค้าทั้งแถว
- แม้กระทั่ง **password** ที่ใช้ authenticate (ถ้าไม่ได้ใช้ SCRAM ที่มีการ hash ฝั่ง client)

ทั้งหมดนี้จะเดินทางเป็น plaintext บนเครือข่าย ถ้ามีใครสามารถดักฟัง traffic ได้ — ไม่ว่าจะเป็นคนในองค์กรเดียวกันที่ทำ ARP spoofing, ผู้ให้บริการ cloud ที่แชร์ network segment, หรือแฮกเกอร์ที่แทรกตัวเข้ามาในเครือข่ายภายใน — ก็สามารถอ่านข้อมูลอ่อนไหวได้ทันที

### สถานการณ์เสี่ยงในระบบ e-commerce

```
[Web Application Server] ------ plaintext TCP/5432 ------ [PostgreSQL Server]
        (เชื่อมผ่าน VPC/Network เดียวกัน หรือข้าม Availability Zone)
```

ตัวอย่างสถานการณ์ที่มีความเสี่ยงจริง:

1. **Multi-tier architecture ข้าม network segment** — Application server กับ Database server อยู่คนละ subnet หรือคนละ data center ทำให้ traffic ต้องวิ่งผ่าน network hop หลายจุด
2. **Cloud environment แบบ shared tenancy** — แม้จะอยู่ใน VPC เดียวกัน แต่ยังต้องป้องกัน insider threat และการตั้งค่า security group ผิดพลาด
3. **Read replica หรือ reporting server ที่อยู่ต่างภูมิภาค** — ข้อมูลลูกค้าและคำสั่งซื้อถูก replicate ข้าม region ผ่านเครือข่ายสาธารณะบางส่วน
4. **DBA เชื่อมต่อจากเครื่องส่วนตัวผ่าน VPN หรือ bastion host** — traffic การ query ข้อมูล production วิ่งผ่านหลาย hop

### สิ่งที่ SSL/TLS ป้องกันได้ และป้องกันไม่ได้

| ป้องกันได้ | ป้องกันไม่ได้ |
|---|---|
| การดักฟัง (eavesdropping) ข้อมูลระหว่างทาง | SQL Injection |
| การปลอมแปลงข้อมูลระหว่างทาง (tampering) เมื่อใช้ verify mode ที่เหมาะสม | สิทธิ์การเข้าถึงที่ตั้งไว้ผิดพลาด (over-privileged roles) |
| Man-in-the-Middle attack (เมื่อ verify certificate อย่างถูกต้อง) | ข้อมูลที่ถูกขโมยจาก disk/backup (ต้องใช้ Encryption at Rest) |
| การขโมย credential จากการดักฟัง traffic ตอน authenticate | Insider ที่มีสิทธิ์เข้าถึงฐานข้อมูลโดยตรงอยู่แล้ว |

> **ข้อควรจำ:** PostgreSQL ใช้ SCRAM-SHA-256 (ตั้งแต่ PostgreSQL 10+) เป็นวิธี authenticate ที่ไม่ส่ง password แบบ plaintext อยู่แล้ว แต่ SSL/TLS ยังจำเป็น เพราะ**ข้อมูลในตัว query และผลลัพธ์**ยังคงเป็น plaintext ถ้าไม่เปิด SSL — นี่คือสิ่งที่ระบบ e-commerce ต้องกังวลมากที่สุด เพราะ query ที่ส่งอาจมีเลขบัตร เลขบัญชี หรือ PII อื่น ๆ อยู่ในนั้น

---

## Step 692: การเปิดใช้งาน SSL ใน PostgreSQL

การเปิด SSL บน PostgreSQL server ทำผ่านไฟล์ `postgresql.conf` และต้องเตรียมไฟล์ certificate/private key ให้พร้อมก่อน

### พารามิเตอร์หลักใน postgresql.conf

```ini
# postgresql.conf

# เปิดใช้งาน SSL
ssl = on

# ตำแหน่งไฟล์ certificate (public) และ private key ของ server
ssl_cert_file = 'server.crt'
ssl_key_file  = 'server.key'

# (แนะนำ) Certificate Authority ที่ server เชื่อถือ สำหรับ verify client certificate
ssl_ca_file = 'root.crt'

# (แนะนำ) Certificate Revocation List
ssl_crl_file = ''

# ควบคุมชุด cipher ที่อนุญาต (ปฏิเสธ cipher เก่าที่ไม่ปลอดภัย)
ssl_ciphers = 'HIGH:MEDIUM:+3DES:!aNULL'

# บังคับ TLS version ขั้นต่ำ (แนะนำอย่างยิ่งสำหรับ production)
ssl_min_protocol_version = 'TLSv1.2'
ssl_max_protocol_version = ''

# ให้ server เลือก cipher ที่ต้องการก่อน แทนที่จะให้ client เลือก
ssl_prefer_server_ciphers = on

# เปิดใช้ Perfect Forward Secrecy ด้วย Diffie-Hellman parameters
ssl_dh_params_file = ''
```

### ข้อกำหนดของไฟล์ private key

PostgreSQL จะปฏิเสธการ start ถ้า `ssl_key_file` มี permission ที่หย่อนเกินไป เพราะ private key ต้องเข้าถึงได้เฉพาะ owner (ผู้ใช้ที่รัน postgres process) เท่านั้น

```bash
# ตรวจสอบและตั้งค่า permission ของ private key ให้ถูกต้อง
chmod 600 /etc/postgresql/16/main/server.key
chown postgres:postgres /etc/postgresql/16/main/server.key

# ถ้า permission ผิด PostgreSQL จะ error ตอน start ประมาณนี้:
# FATAL: private key file "server.key" has group or world access
# DETAIL: File must have permissions u=rw (0600) or less if owned by the database user.
```

### ตำแหน่งไฟล์ certificate

ค่า default ของ PostgreSQL จะมองหาไฟล์ในโฟลเดอร์ data directory (`PGDATA`) แต่สามารถระบุ path แบบเต็มได้เช่นกัน:

```ini
# วิธีที่ 1: ใช้ path แบบเต็ม (แนะนำสำหรับ production เพื่อความชัดเจน)
ssl_cert_file = '/etc/postgresql/certs/server.crt'
ssl_key_file  = '/etc/postgresql/certs/server.key'
ssl_ca_file   = '/etc/postgresql/certs/root.crt'

# วิธีที่ 2: ใช้ path สัมพัทธ์กับ PGDATA (ค่า default)
# server.crt, server.key จะถูกวางไว้ใน $PGDATA โดยตรง
```

### การ reload configuration หลังตั้งค่า

การเปลี่ยนค่า SSL ส่วนใหญ่ (`ssl`, `ssl_cert_file`, `ssl_key_file`, `ssl_ciphers`) เป็นพารามิเตอร์ประเภท `SIGHUP` — สามารถ reload ได้โดยไม่ต้อง restart server:

```bash
# วิธีที่ 1: ใช้ pg_ctl
pg_ctl reload -D /var/lib/postgresql/16/main

# วิธีที่ 2: ใช้ systemd
sudo systemctl reload postgresql

# วิธีที่ 3: ใช้ SQL function (ต้องเชื่อมต่อได้ก่อนอยู่แล้ว)
```

```sql
SELECT pg_reload_conf();
```

### ตรวจสอบว่า SSL เปิดใช้งานสำเร็จ

```sql
-- ตรวจสอบค่าปัจจุบันของ SSL parameters
SHOW ssl;
SHOW ssl_cert_file;
SHOW ssl_min_protocol_version;

-- ตรวจสอบว่า connection ปัจจุบันของเราใช้ SSL หรือไม่
SELECT
    pid,
    usename,
    ssl,
    ssl_version,
    ssl_cipher,
    client_addr
FROM pg_stat_ssl
JOIN pg_stat_activity USING (pid)
WHERE pid = pg_backend_pid();
```

ผลลัพธ์ตัวอย่างเมื่อเชื่อมต่อผ่าน SSL สำเร็จ:

```
 pid  | usename  | ssl | ssl_version |     ssl_cipher      | client_addr
------+----------+-----+-------------+----------------------+-------------
 8421 | app_user | t   | TLSv1.3     | TLS_AES_256_GCM_SHA384 | 10.0.1.15
```

ถ้า `ssl` เป็น `f` (false) แสดงว่า connection นั้นยังคงเป็น plaintext แม้ server จะเปิด SSL ไว้แล้วก็ตาม — เพราะ `ssl = on` เป็นเพียงการ**อนุญาตให้ใช้ SSL ได้** ไม่ได้บังคับ ต้องไปกำหนดที่ `pg_hba.conf` เพิ่มเติม (Step 694)

---

## Step 693: การสร้าง Certificate — Self-signed สำหรับทดสอบ เทียบกับ CA-signed สำหรับ Production

### แนวคิดพื้นฐานของ Certificate

Certificate ทำหน้าที่สองอย่างพร้อมกัน:

1. **เข้ารหัสข้อมูล** ระหว่างทาง (encryption)
2. **ยืนยันตัวตน** ของฝ่ายที่เราคุยด้วย (authentication) — เพื่อป้องกัน Man-in-the-Middle attack

Self-signed certificate ทำข้อ 1 ได้ดี แต่ทำข้อ 2 ได้ไม่สมบูรณ์ เพราะไม่มีบุคคลที่สาม (Certificate Authority) มารับรองว่า certificate นี้เป็นของ server ตัวจริง — client ต้อง "เชื่อ" certificate นั้นเองโดยตรง (trust on first use) ซึ่งเหมาะสำหรับ development/testing เท่านั้น

### การสร้าง Self-signed Certificate ด้วย openssl (สำหรับทดสอบ)

```bash
# Step 1: สร้าง private key และ self-signed certificate ในคำสั่งเดียว
# -x509 หมายถึงสร้าง self-signed certificate โดยตรง (ไม่ผ่าน CSR)
# -days 365 คืออายุของ certificate
# -newkey rsa:4096 สร้าง RSA key ขนาด 4096 bit
openssl req -new -x509 -days 365 -nodes \
    -out server.crt \
    -keyout server.key \
    -newkey rsa:4096 \
    -subj "/CN=db.ecommerce.internal"

# Step 2: ตั้ง permission ให้ private key
chmod 600 server.key

# Step 3: ย้ายไฟล์ไปยัง data directory หรือตำแหน่งที่กำหนดใน postgresql.conf
sudo mv server.crt server.key /etc/postgresql/certs/
sudo chown postgres:postgres /etc/postgresql/certs/server.*
```

> **สำคัญ:** ค่า `CN` (Common Name) ต้องตรงกับ hostname ที่ client ใช้เชื่อมต่อ เช่นถ้า client เชื่อมต่อด้วย `db.ecommerce.internal` แต่ certificate ออกให้ `CN=localhost` การ verify แบบ `verify-full` (Step 695) จะล้มเหลวทันที

สำหรับ certificate ที่รองรับหลาย hostname หรือ IP ควรใช้ **Subject Alternative Name (SAN)** แทนการพึ่ง CN อย่างเดียว (CN ถูกลดความสำคัญลงในมาตรฐานสมัยใหม่):

```bash
# สร้างไฟล์ config เพิ่ม SAN
cat > openssl_san.cnf <<EOF
[req]
distinguished_name = req_distinguished_name
x509_extensions = v3_req
prompt = no

[req_distinguished_name]
CN = db.ecommerce.internal

[v3_req]
subjectAltName = @alt_names

[alt_names]
DNS.1 = db.ecommerce.internal
DNS.2 = db-replica.ecommerce.internal
IP.1  = 10.0.1.20
EOF

openssl req -new -x509 -days 365 -nodes \
    -out server.crt \
    -keyout server.key \
    -newkey rsa:4096 \
    -config openssl_san.cnf \
    -extensions v3_req
```

### การสร้าง CA-signed Certificate (สำหรับ Production)

ใน production ที่ต้องการความน่าเชื่อถือระดับสูง (เช่นระบบชำระเงินที่ต้องผ่าน PCI-DSS) ควรใช้ certificate ที่ออกโดย Certificate Authority ที่เชื่อถือได้ ซึ่งมี 2 แนวทางหลัก:

**แนวทางที่ 1: ใช้ Public CA** เช่น Let's Encrypt, DigiCert — เหมาะสำหรับ endpoint ที่เข้าถึงผ่าน public internet

```bash
# ตัวอย่างด้วย certbot (Let's Encrypt) — สำหรับ server ที่มี public domain
sudo certbot certonly --standalone -d db.ecommerce.com

# ไฟล์ certificate จะอยู่ที่
# /etc/letsencrypt/live/db.ecommerce.com/fullchain.pem
# /etc/letsencrypt/live/db.ecommerce.com/privkey.pem
```

**แนวทางที่ 2: ใช้ Internal/Private CA** — เหมาะสำหรับฐานข้อมูล internal ที่ไม่ควรเปิดสู่ public internet เช่นระบบ e-commerce ที่ database อยู่หลัง VPC เท่านั้น

```bash
# ------ ขั้นตอนที่ 1: สร้าง Root CA (ทำครั้งเดียว เก็บ private key อย่างปลอดภัยที่สุด) ------
openssl genrsa -out root.key 4096
openssl req -x509 -new -nodes -key root.key -days 3650 \
    -out root.crt \
    -subj "/CN=Ecommerce-Internal-Root-CA"

# ------ ขั้นตอนที่ 2: สร้าง CSR (Certificate Signing Request) สำหรับ database server ------
openssl genrsa -out server.key 4096
openssl req -new -key server.key \
    -out server.csr \
    -subj "/CN=db.ecommerce.internal"

# ------ ขั้นตอนที่ 3: ใช้ Root CA เซ็นรับรอง server certificate ------
openssl x509 -req -in server.csr \
    -CA root.crt -CAkey root.key -CAcreateserial \
    -out server.crt -days 825 \
    -extfile openssl_san.cnf -extensions v3_req

# ------ ขั้นตอนที่ 4: แจกจ่ายไฟล์ ------
# server.crt, server.key -> เก็บไว้ที่ database server เท่านั้น
# root.crt               -> แจกจ่ายให้ client ทุกตัวที่ต้อง verify server (sslmode=verify-full)
```

### สรุปเปรียบเทียบ

| หัวข้อ | Self-signed | CA-signed (Internal CA) | CA-signed (Public CA) |
|---|---|---|---|
| เหมาะกับ | Dev / Test | Production internal (VPC) | Production ที่เข้าถึงผ่าน internet |
| ค่าใช้จ่าย | ฟรี | ฟรี (ดูแลเอง) | ฟรี (Let's Encrypt) ถึงมีค่าใช้จ่าย (DigiCert ฯลฯ) |
| การจัดการความน่าเชื่อถือ | client ต้อง trust เอง (เสี่ยงต่อ MITM ถ้าไม่ verify ให้ดี) | Root CA แจกจ่ายให้ client ภายในองค์กรเท่านั้น | Browser/OS trust store รู้จัก CA อยู่แล้ว |
| การต่ออายุ (renewal) | ทำเอง | ทำเองผ่าน internal PKI | Certbot/ACME auto-renew ได้ |
| รองรับ `sslmode=verify-full` อย่างมีความหมาย | ได้ แต่ต้อง distribute cert เอง | ได้ดี | ได้ดีที่สุด |

---

## Step 694: pg_hba.conf กับ SSL — hostssl เทียบกับ hostnossl

การเปิด `ssl = on` เพียงอย่างเดียวไม่ได้บังคับให้ client ต้องใช้ SSL — client ยังสามารถเลือกเชื่อมต่อแบบ plaintext ได้อยู่ ถ้าต้องการ**บังคับ**ว่าการเชื่อมต่อบางประเภท (หรือทั้งหมด) ต้องผ่าน SSL เท่านั้น ต้องกำหนดใน `pg_hba.conf`

### รูปแบบ record ใน pg_hba.conf

```
# TYPE      DATABASE  USER    ADDRESS         METHOD
```

`TYPE` มีค่าที่เกี่ยวข้องกับ SSL ดังนี้:

| TYPE | ความหมาย |
|---|---|
| `host` | อนุญาตทั้ง SSL และ non-SSL (TCP/IP ปกติ) |
| `hostssl` | อนุญาต**เฉพาะ**การเชื่อมต่อที่ใช้ SSL เท่านั้น |
| `hostnossl` | อนุญาต**เฉพาะ**การเชื่อมต่อที่**ไม่**ใช้ SSL เท่านั้น |
| `hostgssenc` | อนุญาตเฉพาะการเชื่อมต่อที่เข้ารหัสด้วย GSSAPI |

### ตัวอย่างการตั้งค่าสำหรับระบบ e-commerce

**สถานการณ์ที่ 1: บังคับให้ทุก connection จาก application server ต้องใช้ SSL**

```
# pg_hba.conf

# TYPE       DATABASE        USER          ADDRESS          METHOD
# ปฏิเสธ connection ที่ไม่ใช้ SSL จาก application subnet โดยตรง
hostnossl    ecommerce_db    app_user      10.0.1.0/24      reject

# บังคับ SSL สำหรับ application server ที่เชื่อมต่อฐานข้อมูล
hostssl      ecommerce_db    app_user      10.0.1.0/24      scram-sha-256

# บังคับ SSL แบบเข้มขึ้นสำหรับ payment service (ใช้ client certificate ด้วย)
hostssl      ecommerce_db    payment_svc   10.0.2.0/24      scram-sha-256 clientcert=verify-full

# DBA เชื่อมต่อผ่าน bastion host ต้องใช้ SSL เสมอ
hostssl      all             dba_user      10.0.9.5/32      scram-sha-256

# ปฏิเสธทุกอย่างที่เหลือ (deny by default)
host         all             all           0.0.0.0/0        reject
```

**สถานการณ์ที่ 2: อนุญาต local socket โดยไม่ต้องใช้ SSL (เพราะไม่ผ่านเครือข่าย)**

```
# TYPE       DATABASE   USER    ADDRESS       METHOD
# Unix domain socket ไม่ต้องเข้ารหัส เพราะไม่ได้วิ่งผ่านเครือข่ายจริง
local        all        all                   peer

# Loopback interface (localhost) อาจผ่อนปรนได้ในบาง environment
host         all        postgres  127.0.0.1/32  scram-sha-256
```

### ลำดับการอ่าน pg_hba.conf สำคัญมาก

PostgreSQL อ่าน `pg_hba.conf` **จากบนลงล่าง** และใช้ record**แรก**ที่ match — ดังนั้นถ้าเขียน `hostssl` ไว้หลัง record ที่กว้างกว่าและ match ไปแล้ว (เช่น `host all all 0.0.0.0/0 trust`) การบังคับ SSL จะไม่มีผลเลย ต้องเรียงลำดับจาก**เฉพาะเจาะจงที่สุดไปกว้างที่สุด**เสมอ

```bash
# หลังแก้ pg_hba.conf ต้อง reload เสมอ (ไม่ต้อง restart)
sudo -u postgres pg_ctl reload -D /var/lib/postgresql/16/main
```

```sql
-- ตรวจสอบว่า config ที่ reload ไปถูกต้อง ไม่มี error
SELECT * FROM pg_hba_file_rules WHERE error IS NOT NULL;
```

### ทดสอบว่าการบังคับ SSL ทำงานจริง

```bash
# พยายามเชื่อมต่อแบบปิด SSL — ควรถูกปฏิเสธถ้า pg_hba.conf บังคับ hostssl ไว้
psql "host=db.ecommerce.internal dbname=ecommerce_db user=app_user sslmode=disable"
# คาดหวัง error:
# psql: error: connection to server at "db.ecommerce.internal" failed:
# FATAL: no pg_hba.conf entry for host "10.0.1.15", user "app_user",
#        database "ecommerce_db", no encryption
```

---

## Step 695: sslmode ฝั่ง Client — ระดับความปลอดภัยที่แตกต่างกัน

`sslmode` เป็นพารามิเตอร์ที่ควบคุมพฤติกรรมการเชื่อมต่อ**จากฝั่ง client** (libpq, psql, connection pooler, driver ต่าง ๆ) ว่าจะใช้ SSL อย่างไรและตรวจสอบความน่าเชื่อถือระดับไหน มีทั้งหมด 6 ระดับ เรียงจากปลอดภัยน้อยที่สุดไปมากที่สุด

### ตารางสรุประดับ sslmode

| ระดับ | เข้ารหัสข้อมูล | ตรวจสอบว่า CA ที่ออก cert เชื่อถือได้ | ตรวจสอบว่า hostname ตรงกับ certificate | ป้องกัน MITM ได้จริงหรือไม่ |
|---|---|---|---|---|
| `disable` | ไม่ | ไม่ | ไม่ | ไม่ |
| `allow` | อาจจะ (ลอง non-SSL ก่อน ถ้าปฏิเสธค่อยลอง SSL) | ไม่ | ไม่ | ไม่ |
| `prefer` (ค่า default) | อาจจะ (ลอง SSL ก่อน ถ้าไม่ได้ fallback เป็น non-SSL) | ไม่ | ไม่ | ไม่ |
| `require` | ใช่ (บังคับ SSL) | ไม่ | ไม่ | ไม่ (เข้ารหัสอย่างเดียว ยัง MITM ได้ด้วย fake cert) |
| `verify-ca` | ใช่ | ใช่ | ไม่ | บางส่วน |
| `verify-full` | ใช่ | ใช่ | ใช่ | **ใช่ ป้องกันได้เต็มรูปแบบ** |

### อธิบายรายละเอียดแต่ละระดับ

**1. `disable`** — ไม่ใช้ SSL เลย แม้ server จะรองรับก็ตาม

```bash
psql "host=db.ecommerce.internal dbname=ecommerce_db user=app_user sslmode=disable"
```

ใช้ได้เฉพาะในการเชื่อมต่อผ่าน trusted network ภายในเท่านั้น (เช่น local Docker network สำหรับทดสอบ) **ห้ามใช้กับข้อมูลจริงในระบบ production เด็ดขาด**

**2. `allow`** — client จะลองเชื่อมต่อแบบ non-SSL ก่อน ถ้า server ปฏิเสธค่อยลองใหม่ด้วย SSL

```bash
psql "host=db.ecommerce.internal dbname=ecommerce_db user=app_user sslmode=allow"
```

แทบไม่มีประโยชน์ในทางปฏิบัติ เพราะยัง "ยอม" เชื่อมต่อแบบ plaintext ได้ถ้า server อนุญาต — เหมาะกับกรณี migration ชั่วคราวเท่านั้น

**3. `prefer`** — ค่า default ของ libpq ลอง SSL ก่อน ถ้าเชื่อมต่อ SSL ไม่ได้ (เช่น server ไม่รองรับ) จะ fallback กลับไปเป็น plaintext โดยอัตโนมัติ**โดยไม่แจ้งเตือน**

```bash
psql "host=db.ecommerce.internal dbname=ecommerce_db user=app_user sslmode=prefer"
```

> **คำเตือนสำคัญ:** `prefer` คือค่า default แต่**ไม่ปลอดภัยพอสำหรับข้อมูลอ่อนไหว** เพราะ attacker ที่ทำ MITM สามารถบล็อก SSL handshake เพื่อบังคับให้ client fallback เป็น plaintext ได้ (downgrade attack) — สำหรับระบบ e-commerce ที่จัดการข้อมูลลูกค้าและการชำระเงิน**ต้องไม่ใช้ค่า default นี้**

**4. `require`** — บังคับต้องใช้ SSL เท่านั้น ถ้าเชื่อมต่อ SSL ไม่ได้จะ fail ทันที แต่**ไม่ตรวจสอบ**ว่า certificate ของ server น่าเชื่อถือหรือไม่

```bash
psql "host=db.ecommerce.internal dbname=ecommerce_db user=app_user sslmode=require"
```

ข้อมูลถูกเข้ารหัสแน่นอน แต่ client ยังคงเชื่อ certificate อะไรก็ได้ที่ server ส่งมา (รวมถึง self-signed certificate ปลอมที่ attacker สร้างขึ้นระหว่างทำ MITM) — เหมาะกับกรณีที่ network ภายในค่อนข้างเชื่อถือได้อยู่แล้ว แต่ยังต้องการเข้ารหัส traffic เป็นชั้นป้องกันเพิ่ม

**5. `verify-ca`** — บังคับ SSL และตรวจสอบว่า certificate ของ server ถูกเซ็นรับรองโดย CA ที่ client เชื่อถือ (ระบุผ่าน `sslrootcert`) แต่**ไม่ตรวจสอบ**ว่า hostname ตรงกับ certificate

```bash
psql "host=db.ecommerce.internal dbname=ecommerce_db user=app_user \
      sslmode=verify-ca sslrootcert=/etc/ssl/certs/root.crt"
```

ป้องกัน attacker ที่ใช้ certificate ปลอมที่ไม่ได้เซ็นโดย CA ที่รู้จัก แต่ยังมีช่องโหว่ถ้า attacker มี certificate ที่ถูกต้องจาก CA เดียวกันสำหรับ domain อื่น (edge case ที่พบได้น้อยแต่มีจริง)

**6. `verify-full`** — ระดับปลอดภัยสูงสุด ตรวจสอบทั้ง CA และ hostname ต้องตรงกับ CN หรือ SAN ใน certificate

```bash
psql "host=db.ecommerce.internal dbname=ecommerce_db user=app_user \
      sslmode=verify-full sslrootcert=/etc/ssl/certs/root.crt"
```

ป้องกัน MITM ได้เต็มรูปแบบ เพราะแม้ attacker จะมี certificate ที่ CA เดียวกันเซ็นให้ แต่ต้องเป็น certificate ที่ระบุ hostname ตรงกับที่ client เรียกใช้เท่านั้น **นี่คือระดับที่แนะนำสำหรับการเชื่อมต่อกับข้อมูล production ของระบบ e-commerce โดยเฉพาะ service ที่แตะข้อมูลการชำระเงิน**

### ตัวอย่าง Connection String แบบเต็มสำหรับ production

```bash
# Environment variable แบบ URI (เหมาะกับ application config)
export DATABASE_URL="postgresql://app_user:${DB_PASSWORD}@db.ecommerce.internal:5432/ecommerce_db?sslmode=verify-full&sslrootcert=/etc/ssl/certs/ecommerce-root-ca.crt"
```

```ini
# ตัวอย่างใน application config (เช่น Python + psycopg2 / SQLAlchemy)
# postgresql://user:password@host:port/dbname?sslmode=verify-full&sslrootcert=/path/to/root.crt
```

```python
# ตัวอย่างการเชื่อมต่อด้วย psycopg2
import psycopg2

conn = psycopg2.connect(
    host="db.ecommerce.internal",
    port=5432,
    dbname="ecommerce_db",
    user="app_user",
    password=db_password,
    sslmode="verify-full",
    sslrootcert="/etc/ssl/certs/ecommerce-root-ca.crt",
)
```

---

## Step 696: Client Certificate Authentication

นอกจากใช้ SSL เพื่อเข้ารหัสข้อมูล เรายังสามารถใช้ **client certificate** เพื่อยืนยันตัวตนของ client แทนหรือเสริม password ได้ — วิธีนี้เรียกว่า **mutual TLS (mTLS)** เพราะทั้ง server และ client ต่างยืนยันตัวตนซึ่งกันและกันด้วย certificate

### แนวคิด

- Server มี certificate ของตัวเอง (จาก Step 693) เพื่อให้ client verify
- Client (เช่น payment service) มี certificate ของตัวเองที่เซ็นโดย CA เดียวกัน เพื่อให้ **server** verify ว่า client เป็นใคร
- Server ต้องมีไฟล์ `ssl_ca_file` เพื่อรู้ว่า CA ไหนที่ตนเองเชื่อถือสำหรับตรวจสอบ client certificate

### ขั้นตอนที่ 1: สร้าง Client Certificate

```bash
# สร้าง private key และ CSR สำหรับ payment service
openssl genrsa -out payment_svc.key 4096
openssl req -new -key payment_svc.key \
    -out payment_svc.csr \
    -subj "/CN=payment_svc"

# ใช้ Internal Root CA (จาก Step 693) เซ็นรับรอง client certificate
openssl x509 -req -in payment_svc.csr \
    -CA root.crt -CAkey root.key -CAcreateserial \
    -out payment_svc.crt -days 365

chmod 600 payment_svc.key
```

> **ข้อสังเกตสำคัญ:** ค่า `CN` ของ client certificate จะถูกใช้เทียบกับชื่อ PostgreSQL role โดยตรง — ในตัวอย่างนี้ `CN=payment_svc` ต้องตรงกับชื่อ role `payment_svc` ใน PostgreSQL (เว้นแต่จะตั้งค่า `clientcert` ร่วมกับวิธี authentication อื่นที่ไม่อิง CN โดยตรง)

### ขั้นตอนที่ 2: ตั้งค่า Server ให้รู้จัก CA สำหรับ verify client certificate

```ini
# postgresql.conf
ssl = on
ssl_cert_file = 'server.crt'
ssl_key_file = 'server.key'
ssl_ca_file = 'root.crt'    # CA ที่ใช้ verify client certificate
```

### ขั้นตอนที่ 3: ตั้งค่า pg_hba.conf ให้บังคับ client certificate

```
# TYPE       DATABASE        USER          ADDRESS          METHOD
# บังคับ client certificate + password (เข้มที่สุด — สำหรับ payment service)
hostssl      ecommerce_db    payment_svc   10.0.2.0/24      scram-sha-256 clientcert=verify-full

# หรือใช้ certificate เพียงอย่างเดียวแทน password (cert method)
hostssl      ecommerce_db    payment_svc   10.0.2.0/24      cert
```

ความแตกต่างของ 2 วิธี:

| Method | พฤติกรรม |
|---|---|
| `scram-sha-256 clientcert=verify-full` | ต้องมีทั้ง client certificate ที่ valid **และ** password ถูกต้อง (2 factors) |
| `cert` | ใช้ client certificate อย่างเดียวในการยืนยันตัวตน ไม่ต้องใส่ password เลย — CN ของ certificate ต้องตรงกับ PostgreSQL username เป๊ะ |

สำหรับระบบที่จัดการข้อมูลการชำระเงิน แนะนำให้ใช้ `clientcert=verify-full` ร่วมกับ password เพราะเป็นการยืนยันตัวตนแบบหลายปัจจัย (defense in depth) — แม้ private key ของ client certificate จะรั่วไหล ก็ยังต้องมี password ประกอบด้วย

### ขั้นตอนที่ 4: การเชื่อมต่อจากฝั่ง Client ด้วย Certificate

```bash
psql "host=db.ecommerce.internal dbname=ecommerce_db user=payment_svc \
      sslmode=verify-full \
      sslrootcert=/etc/ssl/certs/root.crt \
      sslcert=/etc/ssl/certs/payment_svc.crt \
      sslkey=/etc/ssl/private/payment_svc.key"
```

```python
# psycopg2 พร้อม client certificate authentication
import psycopg2

conn = psycopg2.connect(
    host="db.ecommerce.internal",
    dbname="ecommerce_db",
    user="payment_svc",
    sslmode="verify-full",
    sslrootcert="/etc/ssl/certs/root.crt",
    sslcert="/etc/ssl/certs/payment_svc.crt",
    sslkey="/etc/ssl/private/payment_svc.key",
)
```

> **หมายเหตุเรื่อง permission ของ client private key:** libpq (บน Linux/macOS) จะปฏิเสธการใช้ `sslkey` ถ้าไฟล์มี permission หย่อนเกิน `0600` เช่นเดียวกับฝั่ง server

### เมื่อไหร่ควรใช้ Client Certificate Authentication

- **Service-to-service communication** ภายใน backend เช่น payment service, order service ที่เชื่อมต่อฐานข้อมูลโดยไม่มี "คน" อยู่หน้าจอ — certificate จัดการง่ายกว่าการหมุนเวียน password
- **Compliance requirement** เช่น PCI-DSS ที่ต้องการ strong authentication สำหรับระบบที่แตะข้อมูลบัตรเครดิต
- **Zero-trust network** ที่ไม่เชื่อถือ network segment ใด ๆ โดย default แม้จะอยู่ใน VPC เดียวกัน

---

## Step 697: Encryption at Rest คืออะไร

**Encryption at Rest** คือการเข้ารหัสข้อมูล**ขณะที่มันถูกเก็บอยู่บน disk** (ไฟล์ data, WAL, backup) ต่างจาก Encryption in Transit ที่ปกป้องข้อมูลระหว่างเดินทางบนเครือข่ายเท่านั้น

### ทำไมสองอย่างนี้ไม่ทดแทนกัน

| สถานการณ์ | Encryption in Transit ช่วยได้ไหม | Encryption at Rest ช่วยได้ไหม |
|---|---|---|
| แฮกเกอร์ดักฟัง network traffic | ป้องกันได้ | ไม่เกี่ยวข้อง |
| Disk หรือ SSD ของ server ถูกขโมยทางกายภาพ | ไม่เกี่ยวข้อง | ป้องกันได้ |
| ไฟล์ backup (`.sql`, `.tar`, base backup) ถูกขโมยหรือรั่วไหล | ไม่เกี่ยวข้อง (backup ไม่ได้วิ่งผ่านเครือข่ายตลอดเวลา) | ป้องกันได้ (ถ้า backup ถูกเข้ารหัสด้วย) |
| Snapshot ของ cloud volume ถูกแชร์ผิดสิทธิ์ | ไม่เกี่ยวข้อง | ป้องกันได้ (ถ้า snapshot เข้ารหัส) |
| พนักงานที่มีสิทธิ์เข้าถึง filesystem โดยตรง (ไม่ผ่าน PostgreSQL) copy ไฟล์ data ออกไป | ไม่เกี่ยวข้อง | ป้องกันได้ (attacker ไม่มี key ก็อ่านไฟล์ raw ไม่ออก) |
| Attacker เจาะเข้ามาใน connection ที่ authenticate แล้ว (มีสิทธิ์ query ปกติ) | ไม่ช่วย | ไม่ช่วย (encryption at rest ไม่ป้องกันการ query ผ่าน SQL ปกติ) |

จะเห็นว่าทั้งสองแบบป้องกันภัยคุกคามคนละประเภท — ระบบ e-commerce ที่จริงจังเรื่องความปลอดภัยต้องใช้**ทั้งคู่ร่วมกัน** ไม่ใช่เลือกอย่างใดอย่างหนึ่ง

### ระดับของ Encryption at Rest

Encryption at Rest สามารถทำได้หลายระดับ แต่ละระดับป้องกันภัยคุกคามต่างกัน:

1. **Full-disk / Volume-level encryption** — เข้ารหัสทั้ง disk หรือ volume เช่น LUKS บน Linux, BitLocker บน Windows, EBS encryption บน AWS ป้องกันกรณี disk ถูกขโมยทางกายภาพหรือ snapshot รั่วไหล
2. **Filesystem-level encryption** — เข้ารหัสเฉพาะบางโฟลเดอร์ เช่น eCryptfs
3. **Database-level (Transparent Data Encryption / TDE)** — เข้ารหัสในระดับไฟล์ data ของฐานข้อมูลโดยตรง โปร่งใสต่อ application (ไม่ต้องแก้ query) — **PostgreSQL ไม่มีฟีเจอร์นี้ built-in** (รายละเอียดใน Step 698)
4. **Column-level / Application-level encryption** — เข้ารหัสเฉพาะคอลัมน์ที่อ่อนไหว เช่นด้วย `pgcrypto` — ควบคุมได้ละเอียดสุด แต่ต้องแก้ application logic (Step 699)

### ข้อจำกัดที่ต้องเข้าใจ

Encryption at Rest ป้องกัน**เฉพาะตอนที่ระบบปิดอยู่หรือ disk ถูกแยกออกจากระบบที่ทำงานอยู่**เท่านั้น — เมื่อ PostgreSQL server กำลังทำงานและมีคน authenticate เข้ามาได้ (ไม่ว่าจะถูกต้องหรือถูกขโมย credential มา) ข้อมูลจะถูก decrypt โดยอัตโนมัติเพื่อให้ query ทำงานได้ ดังนั้น Encryption at Rest **ไม่ทดแทน** การควบคุมสิทธิ์ (RBAC), การจัดการ credential ที่ดี, หรือ audit logging

---

## Step 698: แนวทาง Encryption at Rest ใน PostgreSQL

### ทำไม PostgreSQL ไม่มี TDE Built-in

Transparent Data Encryption (TDE) เป็นฟีเจอร์ที่พบใน Oracle, SQL Server, และ MySQL Enterprise ที่เข้ารหัสไฟล์ data ทั้งหมดในระดับ storage engine โดยที่ application มองไม่เห็นความแตกต่าง (โปร่งใส) PostgreSQL core**ไม่มีฟีเจอร์นี้ built-in** ด้วยเหตุผลหลายประการ:

- การออกแบบ WAL (Write-Ahead Log) ของ PostgreSQL ทำให้การเข้ารหัสระดับ storage ซับซ้อนกว่าฐานข้อมูลอื่น
- มีการพูดคุยและเสนอ patch ใน PostgreSQL community มาหลายปี (เช่น proposal เรื่อง "Transparent Data Encryption" ในปี 2020-2023) แต่ยังไม่ถูกรวมเข้า core เนื่องจากความซับซ้อนด้าน key management และ performance trade-off
- ทีม PostgreSQL core เน้นให้แก้ปัญหานี้ที่ชั้น**filesystem/OS** หรือ**cloud provider** แทน ซึ่งมีความน่าเชื่อถือและผ่านการพิสูจน์แล้ว

ดังนั้นสำหรับ PostgreSQL การทำ Encryption at Rest ต้องพึ่งพา**เครื่องมือภายนอก**เป็นหลัก

### แนวทางที่ 1: Filesystem-level Encryption ด้วย LUKS (Self-hosted)

LUKS (Linux Unified Key Setup) คือมาตรฐานการเข้ารหัส disk บน Linux ที่ทำงานในระดับ block device — เหมาะสำหรับ deployment ที่ self-host PostgreSQL เอง (on-premise หรือ VM ที่ควบคุมเอง)

```bash
# ตัวอย่างการตั้งค่า LUKS บน disk ที่จะใช้เก็บ PostgreSQL data directory
# (ทำก่อนติดตั้ง PostgreSQL หรือก่อน mount data directory)

# 1. เข้ารหัส partition ด้วย LUKS
sudo cryptsetup luksFormat /dev/sdb1

# 2. เปิด (unlock) partition ที่เข้ารหัสไว้
sudo cryptsetup luksOpen /dev/sdb1 pgdata_encrypted

# 3. สร้าง filesystem บน mapped device ที่ decrypt แล้ว
sudo mkfs.ext4 /dev/mapper/pgdata_encrypted

# 4. Mount ไปยังตำแหน่งที่จะใช้เป็น PostgreSQL data directory
sudo mount /dev/mapper/pgdata_encrypted /var/lib/postgresql/16/main

# 5. ตั้งค่าให้ unlock อัตโนมัติตอน boot (ต้องเก็บ key file อย่างปลอดภัย เช่นใน HSM หรือ secret manager)
# /etc/crypttab
# pgdata_encrypted  /dev/sdb1  /root/keys/pgdata.key  luks
```

**ข้อดี:** โปร่งใสต่อ PostgreSQL อย่างสมบูรณ์ (PostgreSQL ไม่รู้ตัวเลยว่ากำลังทำงานบน disk ที่เข้ารหัส), ป้องกันได้ทั้ง data files, WAL, และ temporary files, performance overhead ต่ำ (ใช้ AES-NI hardware acceleration ได้)

**ข้อจำกัด:** ป้องกันเฉพาะตอน disk ถูกถอดออกจากระบบ (offline) — ขณะ server ทำงานอยู่และถูก unlock แล้ว ข้อมูลจะอยู่ในสถานะ decrypted เสมอ, ต้องจัดการ key management เอง (การเก็บ passphrase/keyfile อย่างปลอดภัย)

### แนวทางที่ 2: Cloud Provider Managed Encryption

สำหรับ Managed PostgreSQL บน cloud (AWS RDS/Aurora, Google Cloud SQL, Azure Database for PostgreSQL) มักมีตัวเลือกเข้ารหัส storage ให้แบบง่าย ๆ ผ่าน UI หรือ IaC

```bash
# ตัวอย่าง AWS CLI: สร้าง RDS PostgreSQL instance พร้อมเปิด encryption at rest
aws rds create-db-instance \
    --db-instance-identifier ecommerce-db-prod \
    --engine postgres \
    --engine-version 16.3 \
    --db-instance-class db.r6g.xlarge \
    --allocated-storage 500 \
    --storage-encrypted \
    --kms-key-id arn:aws:kms:ap-southeast-1:123456789012:key/abcd-1234-efgh-5678 \
    --master-username dbadmin \
    --master-user-password "${DB_MASTER_PASSWORD}"
```

```hcl
# ตัวอย่าง Terraform สำหรับ AWS RDS ที่เปิด encryption at rest
resource "aws_db_instance" "ecommerce_db" {
  identifier        = "ecommerce-db-prod"
  engine            = "postgres"
  engine_version    = "16.3"
  instance_class    = "db.r6g.xlarge"
  allocated_storage = 500

  storage_encrypted = true
  kms_key_id        = aws_kms_key.ecommerce_db_key.arn

  # บังคับให้ backup / snapshot ก็ถูกเข้ารหัสตามไปด้วยโดยอัตโนมัติ
  # (RDS จะเข้ารหัส automated backups และ manual snapshots ให้เองเมื่อ storage_encrypted = true)
}
```

**ข้อดีของแนวทาง cloud-managed:**

- เปิดใช้งานง่ายมาก มักเป็นแค่ checkbox หรือ flag เดียว
- Backup, snapshot, และ read replica จะถูกเข้ารหัสตามไปโดยอัตโนมัติ
- Key management ผ่าน KMS (Key Management Service) ที่มี audit trail, key rotation, และ access control ในตัว
- ไม่มี performance overhead ที่สังเกตเห็นได้ (ทำในระดับ storage layer ของ cloud provider)

**ข้อควรระวัง:** encryption at rest แบบนี้**ต้องเปิดตอนสร้าง instance** — RDS ส่วนใหญ่ไม่อนุญาตให้เปิดย้อนหลังกับ instance ที่มีอยู่แล้วโดยตรง ต้องสร้าง encrypted snapshot แล้ว restore เป็น instance ใหม่แทน

```bash
# กรณีต้องเข้ารหัส RDS instance ที่มีอยู่แล้วย้อนหลัง (ทำผ่าน snapshot)
# 1. สร้าง snapshot จาก instance เดิม (ไม่เข้ารหัส)
aws rds create-db-snapshot \
    --db-instance-identifier ecommerce-db-prod \
    --db-snapshot-identifier ecommerce-db-before-encryption

# 2. คัดลอก snapshot พร้อมเข้ารหัส
aws rds copy-db-snapshot \
    --source-db-snapshot-identifier ecommerce-db-before-encryption \
    --target-db-snapshot-identifier ecommerce-db-encrypted \
    --kms-key-id arn:aws:kms:ap-southeast-1:123456789012:key/abcd-1234-efgh-5678

# 3. Restore instance ใหม่จาก encrypted snapshot
aws rds restore-db-instance-from-db-snapshot \
    --db-instance-identifier ecommerce-db-prod-encrypted \
    --db-snapshot-identifier ecommerce-db-encrypted

# 4. Cutover application ไปยัง instance ใหม่ แล้วลบ instance เก่า
```

### แนวทางที่ 3: Extension สำหรับ Encryption (แนวทางเสริม ไม่ใช่ TDE เต็มรูปแบบ)

มี extension บางตัวในระบบนิเวศ PostgreSQL ที่พยายามเสริมเรื่อง encryption แต่ควรเข้าใจข้อจำกัด:

- **pgcrypto** — เข้ารหัสระดับ column/value ไม่ใช่ TDE เต็มรูปแบบ (รายละเอียดใน Step 699)
- **pg_tde** (extension จากบาง distribution เช่น Percona) — พยายามเลียนแบบ TDE โดยเข้ารหัสไฟล์ data บางส่วน แต่ยังไม่ใช่ core PostgreSQL feature และยังมีข้อจำกัดด้าน WAL encryption, compatibility กับ extension อื่น ๆ

### สรุปแนวทางเลือกตามสถานการณ์

| สถานการณ์ | แนวทางที่แนะนำ |
|---|---|
| Self-hosted PostgreSQL บน VM/bare-metal | LUKS filesystem-level encryption |
| Managed PostgreSQL บน AWS RDS/Aurora | `storage_encrypted = true` + KMS |
| Managed PostgreSQL บน Google Cloud SQL | Customer-Managed Encryption Keys (CMEK) |
| Managed PostgreSQL บน Azure | Transparent Data Encryption ของ Azure (ระดับ storage, ไม่ใช่ของ PostgreSQL) |
| ข้อมูลอ่อนไหวมากเป็นพิเศษ (เลขบัตรเครดิต) ที่ต้องการควบคุมละเอียด | เสริมด้วย Column-level Encryption (pgcrypto) ทับแนวทางข้างต้น |

---

## Step 699: Column-level Encryption ด้วย pgcrypto

จาก **Part 057** เราได้แนะนำ `pgcrypto` extension ไปแล้วในบริบทของการ hash password และการเข้ารหัสข้อมูลทั่วไป ในบทนี้เราจะเจาะลึกการใช้ `pgcrypto` เพื่อทำ **Column-level Encryption** โดยเฉพาะ ซึ่งเป็นทางเลือกที่เหมาะเมื่อ:

- ไม่ต้องการ (หรือไม่สามารถ) เข้ารหัสทั้ง disk เช่น environment แบบ shared hosting ที่ควบคุม infrastructure ไม่ได้เต็มที่
- ต้องการป้องกัน**เฉพาะบาง column ที่อ่อนไหวที่สุด** โดยไม่กระทบ performance ของ column อื่นที่ต้อง query บ่อย (เช่น index scan บน column ทั่วไป)
deriving ความปลอดภัยแบบ **defense in depth** — แม้ Encryption at Rest ระดับ disk จะถูกเจาะ (เช่น server ที่กำลังทำงานถูกแฮก) column ที่เข้ารหัสด้วย pgcrypto ยังต้องใช้ key แยกต่างหากถึงจะอ่านได้

> **หมายเหตุ:** ย่อหน้าด้านบนมี glitch การพิมพ์ ("deriving") ขอแก้ไขเป็นประโยคที่ถูกต้อง: **เพื่อให้ได้ความปลอดภัยแบบ defense in depth** — แม้ Encryption at Rest ระดับ disk จะถูกเจาะ (เช่น server ที่กำลังทำงานถูกแฮก) column ที่เข้ารหัสด้วย pgcrypto ยังต้องใช้ key แยกต่างหากถึงจะอ่านได้

### เตรียม pgcrypto

```sql
CREATE EXTENSION IF NOT EXISTS pgcrypto;
```

### ตัวอย่าง: เข้ารหัสเลขบัตรเครดิตในตาราง payment_methods

สมมติเรามีตาราง `payment_methods` ที่เก็บข้อมูลบัตรของลูกค้า (ในทางปฏิบัติจริง ระบบ e-commerce ส่วนใหญ่จะไม่เก็บเลขบัตรเต็มเลย แต่เก็บ token จาก payment gateway แทนเพื่อลดขอบเขต PCI-DSS — แต่ในกรณีที่จำเป็นต้องเก็บข้อมูลอ่อนไหวบางส่วน เช่น เลขบัตรประชาชนสำหรับยืนยันตัวตน หรือเลขที่บัญชีธนาคารสำหรับคืนเงิน ก็ใช้แนวทางเดียวกันนี้ได้):

```sql
CREATE TABLE payment_methods (
    payment_method_id  BIGSERIAL PRIMARY KEY,
    customer_id         BIGINT NOT NULL REFERENCES customers(customer_id),
    card_last4          CHAR(4) NOT NULL,           -- เก็บ plaintext ได้ เพราะไม่ระบุตัวตนโดยตรง
    card_number_encrypted BYTEA NOT NULL,            -- เข้ารหัสด้วย pgcrypto
    card_brand           VARCHAR(20) NOT NULL,
    created_at           TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

**การเข้ารหัสข้อมูลตอน insert** ด้วยฟังก์ชัน `pgp_sym_encrypt` (symmetric encryption ด้วย passphrase):

```sql
-- key การเข้ารหัสไม่ควร hardcode ใน SQL แบบนี้ในระบบจริง
-- ในทางปฏิบัติควรส่งผ่าน parameter จาก application ที่ดึง key จาก secret manager (เช่น AWS Secrets Manager, HashiCorp Vault)
INSERT INTO payment_methods (customer_id, card_last4, card_number_encrypted, card_brand)
VALUES (
    1024,
    '1234',
    pgp_sym_encrypt('4111111111111234', current_setting('app.encryption_key')),
    'VISA'
);
```

**การถอดรหัสตอน query** (เฉพาะเมื่อจำเป็นจริง ๆ เท่านั้น):

```sql
SELECT
    payment_method_id,
    card_last4,
    pgp_sym_decrypt(card_number_encrypted, current_setting('app.encryption_key')) AS card_number,
    card_brand
FROM payment_methods
WHERE customer_id = 1024;
```

การตั้งค่า `app.encryption_key` ผ่าน session parameter ที่ application ส่งเข้ามาตอนเชื่อมต่อ (ไม่ hardcode ใน SQL):

```sql
-- Application ตั้งค่า session parameter ก่อน query (ดึง key จาก secret manager มาก่อน)
SET app.encryption_key = 'retrieved-from-secret-manager-at-runtime';
```

### ใช้ Asymmetric Encryption สำหรับสถานการณ์ที่ต้องการแยกสิทธิ์ เขียน/อ่าน

pgcrypto รองรับ PGP asymmetric encryption ด้วย — เหมาะกับสถานการณ์ที่ต้องการให้ระบบหน้าบ้าน (เช่น checkout service) **เขียน**ข้อมูลเข้ารหัสได้ แต่**ไม่สามารถอ่าน**กลับได้ มีเพียงระบบหลังบ้านที่มี private key เท่านั้นที่ถอดรหัสได้:

```bash
# สร้างคู่กุญแจ PGP ด้วย gpg (ทำครั้งเดียว เก็บ private key อย่างปลอดภัยที่สุด)
gpg --batch --gen-key <<EOF
%no-protection
Key-Type: RSA
Key-Length: 4096
Name-Real: Ecommerce Payment Encryption
Expire-Date: 0
%commit
EOF

# Export public key (ใช้ฝั่งเขียนข้อมูล) และ private key (ใช้ฝั่งอ่านข้อมูลเท่านั้น)
gpg --export --armor "Ecommerce Payment Encryption" > payment_public.key
gpg --export-secret-key --armor "Ecommerce Payment Encryption" > payment_private.key
```

```sql
-- ฝั่งเขียน (checkout service): เข้ารหัสด้วย public key เท่านั้น ไม่สามารถถอดรหัสได้แม้จะมีสิทธิ์ INSERT
INSERT INTO payment_methods (customer_id, card_last4, card_number_encrypted, card_brand)
VALUES (
    1024,
    '1234',
    pgp_pub_encrypt('4111111111111234', dearmor(current_setting('app.public_key'))),
    'VISA'
);

-- ฝั่งอ่าน (payment reconciliation service ที่มีสิทธิ์สูงกว่า): ถอดรหัสด้วย private key
SELECT pgp_pub_decrypt(
    card_number_encrypted,
    dearmor(current_setting('app.private_key'))
) AS card_number
FROM payment_methods
WHERE payment_method_id = 501;
```

รูปแบบนี้มีประโยชน์มากในสถาปัตยกรรมที่แยก role ชัดเจน: service ที่รับข้อมูลจากลูกค้า (attack surface สูงเพราะเผชิญ internet โดยตรง) ไม่ควรมีความสามารถถอดรหัสข้อมูลได้เลย แม้จะถูกแฮกก็ตาม

### ข้อจำกัดของ Column-level Encryption ที่ต้องเข้าใจ

| ข้อจำกัด | รายละเอียด |
|---|---|
| **Query ได้ยากขึ้น** | ไม่สามารถทำ `WHERE card_number_encrypted = 'xxx'` ได้ตรง ๆ เพราะ ciphertext เปลี่ยนทุกครั้งที่เข้ารหัส (มี random IV) ต้อง decrypt ทั้งแถวก่อนเทียบ หรือเก็บ deterministic hash แยกไว้สำหรับ lookup |
| **ทำ Index ปกติไม่ได้อย่างมีประสิทธิภาพ** | Index บน encrypted column ใช้ค้นหาแบบ range หรือ equality ตรง ๆ ไม่ได้ ต้องพึ่ง column เสริม เช่น `card_last4` ที่เก็บ plaintext บางส่วน |
| **Performance overhead** | การ encrypt/decrypt ใช้ CPU เพิ่มขึ้นทุกครั้งที่ query โดยเฉพาะถ้า decrypt หลายแถวพร้อมกัน |
| **Key management เป็นความรับผิดชอบของ application** | pgcrypto ไม่มีระบบ key rotation หรือ key management ในตัว ต้องออกแบบเอง |
| **DBA ที่มีสิทธิ์ superuser ยังเห็น query และ session parameter ได้** | ถ้า key ถูกส่งผ่าน `SET` หรือปรากฏใน query log ก็ยังมีความเสี่ยงรั่วไหลผ่าน log ได้ ต้องระวังเรื่อง `log_statement` และ `pg_stat_statements` |

> **เชื่อมโยงกลับไป Part 057:** บทนั้นเราเน้นการใช้ `pgcrypto` สำหรับ hash password ด้วย `crypt()` และ `gen_salt()` ซึ่งเป็น **one-way hashing** (ถอดรหัสกลับไม่ได้ ใช้สำหรับเทียบค่าเท่านั้น) ในขณะที่บทนี้เน้น **two-way encryption** ด้วย `pgp_sym_encrypt`/`pgp_pub_encrypt` (ถอดรหัสกลับได้ ใช้เมื่อต้องการเรียกดูข้อมูลต้นฉบับ เช่น เลขบัตรที่ต้องแสดงให้ลูกค้าเห็น 4 หลักท้าย) ทั้งสองเทคนิคใช้ extension เดียวกันแต่ตอบโจทย์คนละสถานการณ์ — password ไม่จำเป็นต้อง "อ่านกลับ" ได้ แต่เลขบัตรบางกรณีจำเป็นต้อง decrypt ได้ (เช่นตอนเรียกเก็บเงินซ้ำ)

---

## Step 700: แบบฝึกหัดรวม — ออกแบบกลยุทธ์ Encryption แบบครบวงจร

โจทย์นี้เป็นแบบฝึกหัดสรุปรวมทั้งบท ให้ผู้เรียนออกแบบกลยุทธ์ encryption สำหรับระบบ e-commerce ที่มีองค์ประกอบดังนี้:

**สถาปัตยกรรมของระบบ:**

```
[Customer Browser]
        |  HTTPS (out of scope ของบทนี้ — เป็น web layer)
        v
[Web/API Application Server]  (subnet: 10.0.1.0/24)
        |
        | PostgreSQL connection
        v
[PostgreSQL Primary]  (subnet: 10.0.10.0/24)
        |
        | Streaming replication (ข้าม region)
        v
[PostgreSQL Read Replica - Reporting]  (region: ap-southeast-2)

[Payment Service]  (subnet: 10.0.2.0/24) --- เชื่อมต่อ PostgreSQL Primary โดยตรง
[Nightly Backup Job] --- ส่ง backup ไปเก็บที่ Object Storage
```

**ข้อมูลที่ต้องปกป้อง:**

1. ตาราง `customers` — ชื่อ, อีเมล, เบอร์โทร, ที่อยู่จัดส่ง (PII ทั่วไป)
2. ตาราง `payment_methods` — เลขบัตร (บางส่วน), token จาก payment gateway
3. ตาราง `orders`, `order_items` — ไม่มี PII โดยตรง แต่เชื่อมโยงกับ customer ได้
4. Backup files ที่เก็บบน object storage
5. Replication traffic ข้าม region

### โจทย์ให้ออกแบบ

ให้เขียนกลยุทธ์ครอบคลุม 3 ชั้น พร้อมเหตุผลประกอบ:

**ก) Encryption in Transit** — ต้องเข้ารหัส connection ไหนบ้าง ด้วย sslmode ระดับไหน และ pg_hba.conf ต้องตั้งอย่างไร

**ข) Encryption at Rest** — เลือกแนวทางไหนสำหรับ primary, replica, และ backup โดยพิจารณาว่าระบบนี้ deploy บน self-hosted VM หรือ managed cloud

**ค) Column-level Encryption** — column ไหนที่ควรเข้ารหัสเพิ่มด้วย pgcrypto และควรใช้ symmetric หรือ asymmetric encryption

<details>
<summary><strong>คำตอบตัวอย่าง (แนวทางที่แนะนำ — สามารถออกแบบต่างจากนี้ได้ถ้ามีเหตุผลรองรับ)</strong></summary>

**ก) Encryption in Transit**

```
# pg_hba.conf บน PostgreSQL Primary

# Web/API Application Server: บังคับ SSL + verify-full ฝั่ง client
hostssl    ecommerce_db    app_user       10.0.1.0/24     scram-sha-256

# Payment Service: บังคับ SSL + client certificate (mTLS) เพราะข้อมูลอ่อนไหวที่สุด
hostssl    ecommerce_db    payment_svc    10.0.2.0/24     scram-sha-256 clientcert=verify-full

# Streaming replication ไปยัง read replica ข้าม region: บังคับ SSL เสมอ
hostssl    replication     replicator     10.0.10.0/24    scram-sha-256

# ปฏิเสธการเชื่อมต่อแบบไม่เข้ารหัสทั้งหมดจาก subnet เหล่านี้
hostnossl  all             all            10.0.0.0/16     reject

# ปฏิเสธทุกอย่างที่ไม่ match (deny by default)
host       all             all            0.0.0.0/0       reject
```

Connection string ฝั่ง Application server:

```bash
DATABASE_URL="postgresql://app_user:${PW}@db-primary.internal:5432/ecommerce_db?sslmode=verify-full&sslrootcert=/etc/ssl/certs/ecommerce-root-ca.crt"
```

Connection string ฝั่ง Payment service (ใช้ mTLS):

```bash
DATABASE_URL="postgresql://payment_svc@db-primary.internal:5432/ecommerce_db?sslmode=verify-full&sslrootcert=/etc/ssl/certs/root.crt&sslcert=/etc/ssl/certs/payment_svc.crt&sslkey=/etc/ssl/private/payment_svc.key"
```

เหตุผล: ใช้ `verify-full` ทุกที่เพราะเป็นระบบที่จัดการข้อมูลการชำระเงิน จำเป็นต้องป้องกัน MITM อย่างเต็มรูปแบบ ไม่ใช่แค่เข้ารหัส (`require`) เท่านั้น Payment service ใช้ client certificate เพิ่มเพราะเป็น service-to-service ที่ต้องการ strong authentication แบบไม่พึ่ง password อย่างเดียว (defense in depth ตาม PCI-DSS)

**ข) Encryption at Rest**

สมมติระบบนี้ deploy บน **AWS RDS/Aurora (managed cloud)**:

```hcl
resource "aws_db_instance" "primary" {
  identifier         = "ecommerce-db-primary"
  engine             = "postgres"
  storage_encrypted  = true
  kms_key_id         = aws_kms_key.ecommerce.arn
  # ...
}

resource "aws_db_instance" "read_replica" {
  identifier             = "ecommerce-db-replica-au"
  replicate_source_db    = aws_db_instance.primary.identifier
  storage_encrypted      = true   # replica สืบทอด encryption จาก source โดยอัตโนมัติ
  # ...
}
```

- **Primary:** เปิด `storage_encrypted = true` ตั้งแต่สร้าง instance ครั้งแรก (encryption at rest เปิดย้อนหลังไม่ได้ใน RDS)
- **Read Replica:** เมื่อ primary เข้ารหัสแล้ว replica ที่สร้างจาก snapshot จะเข้ารหัสตามโดยอัตโนมัติ
- **Backup:** ใช้ automated snapshot ของ RDS ที่เข้ารหัสตาม `storage_encrypted` ของ instance ต้นทางอัตโนมัติ ถ้ามี custom backup script ที่ export เป็น `.sql`/`.tar` ไปยัง S3 ต้องเปิด **S3 default encryption (SSE-KMS)** บน bucket นั้นด้วย และควรเข้ารหัสไฟล์ backup ด้วย `gpg` ก่อน upload เป็นอีกชั้นหนึ่ง:

```bash
pg_dump -Fc ecommerce_db | gpg --encrypt --recipient backup-team@ecommerce.com \
    --output "ecommerce_backup_$(date +%Y%m%d).dump.gpg"

aws s3 cp "ecommerce_backup_$(date +%Y%m%d).dump.gpg" \
    s3://ecommerce-backups/postgresql/ --sse aws:kms
```

หากเป็น **self-hosted บน VM** แทน: ใช้ LUKS เข้ารหัส volume ที่เก็บ `$PGDATA` ทั้งของ primary และ replica ตั้งแต่ก่อนติดตั้ง PostgreSQL และตั้งค่า key management ผ่าน secret manager หรือ HSM แยกจากตัว VM เอง

**ค) Column-level Encryption**

| Column | เข้ารหัสเพิ่มด้วย pgcrypto หรือไม่ | เหตุผล |
|---|---|---|
| `customers.email`, `customers.phone` | ไม่จำเป็น (พึ่ง encryption at rest + RBAC + audit log พอ) | ต้อง query บ่อย (login, search, JOIN) การเข้ารหัสจะทำให้ query ยากและช้าลงมาก โดยที่ RDS encryption ครอบคลุมความเสี่ยงหลักอยู่แล้ว |
| `customers.national_id` (ถ้ามีเก็บเลขบัตรประชาชนสำหรับ KYC) | **ควรเข้ารหัส** ด้วย `pgp_sym_encrypt` | เป็นข้อมูลอ่อนไหวสูงตามกฎหมาย (PDPA) ไม่ต้องใช้ query แบบ range หรือ JOIN บ่อย เข้าถึงเฉพาะกรณีตรวจสอบตัวตนเท่านั้น |
| `payment_methods.card_number_encrypted` | **เข้ารหัสอยู่แล้วตั้งแต่ออกแบบ** ด้วย `pgp_pub_encrypt` (asymmetric) | ใช้ asymmetric เพื่อให้ checkout service (attack surface สูงสุด) เขียนได้อย่างเดียว ถอดรหัสไม่ได้ มีเพียง payment reconciliation service ที่มี private key เท่านั้นที่อ่านได้ |
| `payment_methods.card_last4` | ไม่เข้ารหัส (เก็บ plaintext) | ใช้แสดงผลให้ลูกค้าเห็น ("บัตรลงท้าย 1234") ไม่ระบุตัวตนโดยตรงหากหลุดไปเพียงอย่างเดียว |
| `orders`, `order_items` | ไม่เข้ารหัสระดับ column | ไม่มี PII โดยตรง พึ่ง encryption at rest + การควบคุมสิทธิ์ระดับ row (RLS จาก Part 069) พอเพียง |

**สรุปภาพรวมกลยุทธ์:** ใช้ Encryption at Rest (RDS `storage_encrypted`) เป็นเกราะป้องกันชั้นนอกครอบคลุมข้อมูลทั้งหมดจากภัยคุกคามระดับ infrastructure, ใช้ Encryption in Transit (`verify-full` + mTLS สำหรับ payment service) ป้องกันการดักฟัง/MITM บนเครือข่าย, และเสริม Column-level Encryption เฉพาะจุดที่อ่อนไหวที่สุด (เลขบัตร, เลขบัตรประชาชน) เป็นเกราะชั้นในสุดที่ยังป้องกันได้แม้ชั้นนอกถูกเจาะ — เป็นแนวทาง **defense in depth** ที่สมดุลระหว่างความปลอดภัยกับ performance/ความซับซ้อนของระบบ

</details>

---

## สรุปท้ายบท

### ตารางเปรียบเทียบ sslmode ทุกระดับ (สรุปรวม)

| sslmode | เข้ารหัสข้อมูล | Verify CA | Verify Hostname | ป้องกัน MITM | เหมาะกับ |
|---|---|---|---|---|---|
| `disable` | ไม่ | ไม่ | ไม่ | ไม่ | Local dev เท่านั้น |
| `allow` | อาจจะ | ไม่ | ไม่ | ไม่ | แทบไม่มีการใช้งานจริง |
| `prefer` (default) | อาจจะ | ไม่ | ไม่ | ไม่ | ไม่แนะนำสำหรับข้อมูลอ่อนไหว |
| `require` | ใช่ | ไม่ | ไม่ | ไม่ | Internal network ที่เชื่อถือได้ในระดับหนึ่ง |
| `verify-ca` | ใช่ | ใช่ | ไม่ | บางส่วน | ต้องการยืนยัน CA แต่ยืดหยุ่นเรื่อง hostname |
| `verify-full` | ใช่ | ใช่ | ใช่ | **ใช่ เต็มรูปแบบ** | **Production ที่จัดการข้อมูลอ่อนไหว (แนะนำสำหรับ e-commerce)** |

### สิ่งที่ควรจำจากบทนี้

1. **Encryption in Transit** ปกป้องข้อมูลระหว่างทางบนเครือข่าย เปิดผ่าน `ssl = on` ใน postgresql.conf และบังคับด้วย `hostssl` ใน pg_hba.conf
2. Certificate มีสองแบบหลัก: **self-signed** สำหรับทดสอบ และ **CA-signed** (internal หรือ public CA) สำหรับ production
3. `sslmode` มี 6 ระดับ แต่มีเพียง **`verify-full`** เท่านั้นที่ป้องกัน MITM ได้อย่างสมบูรณ์ — ค่า default `prefer` ไม่ปลอดภัยพอสำหรับข้อมูลอ่อนไหว
4. **Client Certificate Authentication (mTLS)** เพิ่มการยืนยันตัวตนอีกชั้น เหมาะกับ service-to-service communication ที่ต้องการความปลอดภัยสูง เช่น payment service
5. **Encryption at Rest** ปกป้องข้อมูลบน disk — PostgreSQL ไม่มี TDE built-in ต้องพึ่ง LUKS (self-hosted) หรือ cloud-managed encryption (RDS, Cloud SQL ฯลฯ)
6. **Column-level Encryption** ด้วย `pgcrypto` เป็นเกราะป้องกันเสริมเฉพาะจุด เหมาะกับข้อมูลอ่อนไหวสูงสุดที่ไม่ต้องการ query บ่อย
7. กลยุทธ์ที่ดีที่สุดคือ **defense in depth** — ใช้ทั้ง 3 ชั้นร่วมกัน ไม่พึ่งพาชั้นใดชั้นหนึ่งเพียงอย่างเดียว

---

## แบบฝึกหัดท้ายบท

**1.** ข้อใดคือความแตกต่างหลักระหว่าง Encryption in Transit และ Encryption at Rest?

<details>
<summary>เฉลย</summary>

Encryption in Transit เข้ารหัสข้อมูลระหว่างที่เดินทางผ่านเครือข่าย (client ↔ server) ป้องกันการดักฟัง (eavesdropping) และ MITM ส่วน Encryption at Rest เข้ารหัสข้อมูลขณะที่จัดเก็บอยู่บน disk (data files, WAL, backup) ป้องกันกรณี disk/backup ถูกขโมยหรือรั่วไหล ทั้งสองอย่างป้องกันภัยคุกคามคนละประเภท และไม่สามารถทดแทนกันได้ ระบบที่ปลอดภัยจริงต้องใช้ทั้งคู่ร่วมกัน
</details>

**2.** ทำไม `sslmode=prefer` (ค่า default ของ libpq) จึงไม่เหมาะกับการเชื่อมต่อฐานข้อมูลที่มีข้อมูลการชำระเงิน?

<details>
<summary>เฉลย</summary>

เพราะ `prefer` จะลองเชื่อมต่อด้วย SSL ก่อน แต่ถ้าเชื่อมต่อ SSL ไม่สำเร็จ (ไม่ว่าด้วยเหตุผลใดก็ตาม รวมถึงถูก attacker บล็อก SSL handshake โดยเจตนา) จะ fallback กลับไปเป็นการเชื่อมต่อแบบ plaintext โดยอัตโนมัติโดยไม่แจ้งเตือนผู้ใช้ ทำให้เปิดช่องให้เกิด downgrade attack ได้ ควรใช้อย่างน้อย `require` หรือดีที่สุดคือ `verify-full` แทน
</details>

**3.** เขียนคำสั่ง `openssl` สำหรับสร้าง self-signed certificate ที่มีอายุ 365 วัน สำหรับทดสอบ PostgreSQL SSL บนเครื่อง local

<details>
<summary>เฉลย</summary>

```bash
openssl req -new -x509 -days 365 -nodes \
    -out server.crt \
    -keyout server.key \
    -newkey rsa:4096 \
    -subj "/CN=localhost"

chmod 600 server.key
```
</details>

**4.** ในไฟล์ `pg_hba.conf` ความแตกต่างระหว่าง `hostssl` และ `hostnossl` คืออะไร และถ้าต้องการบังคับให้ role `app_user` เชื่อมต่อผ่าน SSL เท่านั้นจาก subnet `10.0.1.0/24` ต้องเขียน record อย่างไร?

<details>
<summary>เฉลย</summary>

`hostssl` อนุญาตเฉพาะ connection ที่ใช้ SSL เท่านั้น ส่วน `hostnossl` อนุญาตเฉพาะ connection ที่**ไม่**ใช้ SSL เท่านั้น (ตรงข้ามกัน) หากต้องการบังคับ SSL สำหรับ `app_user` จาก subnet ที่กำหนด:

```
hostssl    ecommerce_db    app_user    10.0.1.0/24    scram-sha-256
```

และควรเพิ่ม record `hostnossl ... reject` ไว้ก่อนหน้าเพื่อปฏิเสธ non-SSL อย่างชัดเจน หรือปล่อยให้ default deny (ไม่มี record ที่ match แบบ `host` ธรรมดา) ก็เพียงพอ เพราะ PostgreSQL ปฏิเสธ connection ที่ไม่มี record match อยู่แล้ว
</details>

**5.** `sslmode=require` และ `sslmode=verify-full` ต่างกันอย่างไร และทำไม `require` เพียงอย่างเดียวจึงยังเสี่ยงต่อ MITM attack?

<details>
<summary>เฉลย</summary>

`require` บังคับให้ต้องใช้ SSL เท่านั้น (เข้ารหัสข้อมูลแน่นอน) แต่**ไม่ตรวจสอบ**ว่า certificate ของ server น่าเชื่อถือหรือไม่ — client จะยอมรับ certificate อะไรก็ได้ที่ server ส่งมา รวมถึง certificate ปลอมที่ attacker สร้างขึ้นระหว่างทำ MITM ส่วน `verify-full` ตรวจสอบทั้งว่า certificate ถูกเซ็นโดย CA ที่เชื่อถือได้ (`verify-ca`) และ hostname ต้องตรงกับ certificate ด้วย ทำให้ attacker ไม่สามารถใช้ certificate ปลอมหรือ certificate ของ domain อื่นมาสวมรอยได้ จึงป้องกัน MITM ได้อย่างสมบูรณ์
</details>

**6.** Client Certificate Authentication (mTLS) ทำงานอย่างไร และเหมาะกับสถานการณ์ใดในระบบ e-commerce มากที่สุด?

<details>
<summary>เฉลย</summary>

mTLS คือการที่ทั้ง server และ client ต่างมี certificate ของตัวเองและ verify ซึ่งกันและกัน — server verify client certificate ผ่าน `ssl_ca_file` และตั้งค่า `pg_hba.conf` ด้วย `clientcert=verify-full` หรือ method `cert` เหมาะกับ service-to-service communication ที่ไม่มี "คน" อยู่หน้าจอ เช่น payment service ที่เชื่อมต่อฐานข้อมูลโดยตรง เพราะให้ความปลอดภัยระดับสูงกว่าการใช้ password เพียงอย่างเดียว และตรงกับข้อกำหนดของมาตรฐาน เช่น PCI-DSS ที่ต้องการ strong authentication สำหรับระบบที่แตะข้อมูลบัตรเครดิต
</details>

**7.** ทำไม PostgreSQL จึงไม่มี Transparent Data Encryption (TDE) แบบ built-in เหมือน Oracle หรือ SQL Server และมีทางเลือกอะไรทดแทนได้บ้าง?

<details>
<summary>เฉลย</summary>

PostgreSQL ไม่มี TDE built-in ส่วนหนึ่งเพราะความซับซ้อนของการออกแบบ WAL และการจัดการ key management ที่ยังไม่ถูกรวมเข้า core แม้จะมีการเสนอ patch มาหลายครั้ง ทีม core เลือกให้แก้ปัญหานี้ที่ชั้น filesystem/OS หรือ cloud provider แทน ทางเลือกทดแทนได้แก่: (1) Filesystem-level encryption ด้วย LUKS สำหรับ self-hosted deployment (2) Cloud-managed encryption เช่น AWS RDS `storage_encrypted` หรือ Google Cloud SQL CMEK (3) Column-level encryption ด้วย pgcrypto สำหรับข้อมูลเฉพาะจุดที่อ่อนไหวสูงสุด
</details>

**8.** เขียน SQL สำหรับเข้ารหัสเลขบัตรเครดิตด้วย `pgcrypto` โดยใช้ symmetric encryption และเขียน SQL สำหรับถอดรหัสกลับ

<details>
<summary>เฉลย</summary>

```sql
-- เข้ารหัสตอน insert
INSERT INTO payment_methods (customer_id, card_last4, card_number_encrypted, card_brand)
VALUES (
    1024,
    '1234',
    pgp_sym_encrypt('4111111111111234', current_setting('app.encryption_key')),
    'VISA'
);

-- ถอดรหัสตอน select
SELECT
    payment_method_id,
    pgp_sym_decrypt(card_number_encrypted, current_setting('app.encryption_key')) AS card_number
FROM payment_methods
WHERE customer_id = 1024;
```
</details>

**9.** ข้อจำกัดของ Column-level Encryption ที่ทำให้ query ยากขึ้นคืออะไร และมีวิธีแก้ปัญหาเบื้องต้นอย่างไรสำหรับกรณีที่ต้องการค้นหาด้วยค่าที่เข้ารหัสอยู่?

<details>
<summary>เฉลย</summary>

เพราะ ciphertext ที่ได้จาก `pgp_sym_encrypt`/`pgp_pub_encrypt` จะไม่เหมือนกันทุกครั้งที่เข้ารหัสค่าเดียวกัน (เนื่องจากมี random IV ในกระบวนการเข้ารหัส) จึงไม่สามารถทำ `WHERE encrypted_column = 'xxx'` ได้ตรง ๆ และไม่สามารถสร้าง index ที่มีประสิทธิภาพบน column ที่เข้ารหัสได้ วิธีแก้เบื้องต้นคือเก็บ column เสริมที่เป็น plaintext บางส่วนที่ไม่ระบุตัวตนโดยตรง (เช่น `card_last4`) สำหรับ lookup ทั่วไป หรือถ้าจำเป็นต้องค้นหาด้วยค่าที่แม่นยำ อาจเก็บ deterministic hash (เช่น HMAC) แยกไว้อีก column หนึ่งสำหรับ equality lookup โดยไม่กระทบความปลอดภัยของ column เข้ารหัสหลัก
</details>

**10.** ในการออกแบบกลยุทธ์ encryption สำหรับระบบ e-commerce ควรใช้ symmetric encryption (`pgp_sym_encrypt`) หรือ asymmetric encryption (`pgp_pub_encrypt`) สำหรับ service ที่รับข้อมูลบัตรเครดิตจากลูกค้าโดยตรง (เช่น checkout service) และเพราะเหตุใด?

<details>
<summary>เฉลย</summary>

ควรใช้ **asymmetric encryption** (`pgp_pub_encrypt`) เพราะ checkout service เป็น service ที่เผชิญกับ internet โดยตรงและมี attack surface สูงที่สุดในระบบ การใช้ public key เข้ารหัสทำให้ service นี้สามารถ**เขียน**ข้อมูลเข้ารหัสได้ แต่**ไม่มีความสามารถถอดรหัสกลับได้เลย** แม้จะถูกแฮกหรือมี code ที่มีช่องโหว่ ผู้โจมตีก็ไม่สามารถดึงข้อมูลบัตรที่เข้ารหัสไว้ออกมาอ่านได้ ต้องมี private key ซึ่งเก็บแยกไว้ในระบบหลังบ้านที่มีการควบคุมเข้มงวดกว่า (เช่น payment reconciliation service) เท่านั้นถึงจะถอดรหัสได้ — นี่คือหลักการ defense in depth ที่จำกัดขอบเขตความเสียหายหาก service ใด service หนึ่งถูกเจาะ
</details>

---

**บทถัดไป:** [Part 071 — Auditing & Logging](./part-071-auditing-logging.md)
