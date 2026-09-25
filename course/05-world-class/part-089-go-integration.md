# Part 089: เชื่อมต่อ PostgreSQL กับ Go (pgx, GORM)

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับโลก/ผู้เชี่ยวชาญ | Part 089

---

## เป้าหมายการเรียนรู้

เมื่อจบบทนี้ ผู้เรียนจะสามารถ:

1. เข้าใจความแตกต่างระหว่าง `database/sql` interface มาตรฐานของ Go กับ driver `pgx` ที่เชื่อมต่อ PostgreSQL โดยตรง และเลือกใช้ได้อย่างเหมาะสม
2. ติดตั้งและใช้งาน `pgx` เพื่อเชื่อมต่อฐานข้อมูล, รัน query แบบ parameterized ด้วย `Query`, `QueryRow`, และ `Exec`
3. ตั้งค่าและใช้งาน `pgxpool` สำหรับ connection pooling ในแอปพลิเคชันจริงที่ต้องรองรับ concurrent request จำนวนมาก
4. Scan ผลลัพธ์จาก query เข้าสู่ Go struct ได้อย่างถูกต้อง รวมถึงใช้ `pgx.CollectRows` และ `pgx.RowToStructByName` ใน pgx v5
5. เขียน transaction ด้วย `pool.Begin()`, `tx.Commit()`, `tx.Rollback()` พร้อม pattern การจัดการ error แบบ idiomatic Go
6. เข้าใจว่าเมื่อไหร่ควรใช้ `database/sql` แทน pgx โดยตรง โดยเฉพาะกรณีที่ต้องการ compatibility กับ ORM หรือเครื่องมืออื่น
7. ใช้งาน GORM เพื่อ mapping struct เป็นตาราง, ทำ `AutoMigrate`, และเชื่อมต่อผ่าน postgres driver ของ GORM
8. ทำ CRUD operations พื้นฐานด้วย GORM (Create, Find, Update, Delete) และเปรียบเทียบกับการเขียน raw SQL ด้วย pgx
9. ใช้ GORM Association (`belongs to`, `has many`) และ `Preload` สำหรับ eager loading ข้อมูลที่มีความสัมพันธ์กัน
10. ประกอบร่างความรู้ทั้งหมดเขียน HTTP API เล็ก ๆ ด้วย `net/http` + `pgx` สำหรับจัดการสินค้าและคำสั่งซื้อของระบบ e-commerce

---

## เตรียมข้อมูล

ตลอดบทนี้เราจะใช้ schema ของระบบ e-commerce ชุดเดิมที่ใช้มาตลอดหลักสูตร ประกอบด้วย 3 ตาราง คือ `products`, `customers`, และ `orders`

```sql
CREATE TABLE products (
    product_id     SERIAL PRIMARY KEY,
    product_name   VARCHAR(150),
    unit_price     NUMERIC(10,2),
    stock_quantity INTEGER
);

CREATE TABLE customers (
    customer_id  SERIAL PRIMARY KEY,
    first_name   VARCHAR(60),
    email        VARCHAR(150)
);

CREATE TABLE orders (
    order_id     SERIAL PRIMARY KEY,
    customer_id  INTEGER REFERENCES customers(customer_id),
    order_date   TIMESTAMPTZ DEFAULT now(),
    status       VARCHAR(20)
);
```

ข้อมูลตัวอย่างสำหรับทดสอบโค้ดในบทนี้:

```sql
INSERT INTO customers (first_name, email) VALUES
    ('สมชาย', 'somchai@example.com'),
    ('สมหญิง', 'somying@example.com'),
    ('วิชัย', 'wichai@example.com');

INSERT INTO products (product_name, unit_price, stock_quantity) VALUES
    ('เมาส์ไร้สาย', 290.00, 150),
    ('คีย์บอร์ดกลไก', 1290.00, 80),
    ('จอมอนิเตอร์ 27 นิ้ว', 6990.00, 25),
    ('เว็บแคม HD', 890.00, 60);

INSERT INTO orders (customer_id, status) VALUES
    (1, 'pending'),
    (2, 'completed'),
    (1, 'shipped');
```

ก่อนเริ่มต้น สมมติว่าเรามี Go module ที่ชื่อ `ecommerce-api` และมีการตั้งค่า environment variable สำหรับ connection string ไว้แล้ว:

```bash
go mod init ecommerce-api
export DATABASE_URL="postgres://app_user:secret@localhost:5432/ecommerce?sslmode=disable"
```

---

## Step 881: ภาพรวมการเชื่อมต่อ PostgreSQL จาก Go — database/sql เทียบกับ pgx

Go มีวิธีเชื่อมต่อฐานข้อมูลอยู่สองแนวทางหลัก ๆ ที่ทุกคนที่เขียน Go เชื่อมต่อ PostgreSQL ต้องเข้าใจความแตกต่างให้ชัดเจนก่อน

### แนวทางที่ 1: `database/sql` (standard library)

`database/sql` เป็น package มาตรฐานที่มากับ Go เอง ออกแบบมาให้เป็น **generic interface** สำหรับฐานข้อมูลเชิงสัมพันธ์ทุกชนิด ไม่ผูกติดกับ PostgreSQL, MySQL, SQLite หรือฐานข้อมูลใดโดยเฉพาะ แนวคิดคือ `database/sql` กำหนด interface กลาง (`sql.DB`, `sql.Rows`, `sql.Stmt` ฯลฯ) ส่วนการเชื่อมต่อจริงกับฐานข้อมูลแต่ละชนิดทำผ่าน "driver" ที่ implement interface `database/sql/driver`

สำหรับ PostgreSQL มี driver ยอดนิยมสองตัวคือ:

- `github.com/lib/pq` — driver รุ่นเก่า เขียนด้วย pure Go ทั้งหมด ปัจจุบันอยู่ในสถานะ maintenance mode (ไม่มีฟีเจอร์ใหม่)
- `github.com/jackc/pgx/v5/stdlib` — ใช้ pgx เป็น engine เบื้องหลัง แต่ครอบด้วย `database/sql` interface ให้ compatibility กับโค้ดที่คาดหวัง `*sql.DB`

```go
package main

import (
    "database/sql"
    "log"

    _ "github.com/jackc/pgx/v5/stdlib" // ลงทะเบียน driver ชื่อ "pgx"
)

func main() {
    db, err := sql.Open("pgx", "postgres://app_user:secret@localhost:5432/ecommerce")
    if err != nil {
        log.Fatalf("sql.Open ล้มเหลว: %v", err)
    }
    defer db.Close()

    if err := db.Ping(); err != nil {
        log.Fatalf("เชื่อมต่อฐานข้อมูลไม่สำเร็จ: %v", err)
    }
    log.Println("เชื่อมต่อฐานข้อมูลสำเร็จผ่าน database/sql")
}
```

สังเกตว่า `sql.Open` **ไม่ได้เชื่อมต่อจริงทันที** — มันแค่เตรียม `*sql.DB` object ไว้ (ซึ่งจริง ๆ แล้วคือ connection pool) การเชื่อมต่อจริงจะเกิดขึ้นเมื่อมีการเรียกใช้งานครั้งแรก เช่น `db.Ping()` หรือ `db.QueryContext()`

### แนวทางที่ 2: pgx โดยตรง (native driver)

`pgx` (`github.com/jackc/pgx/v5`) คือ driver ที่เขียนขึ้นมาเฉพาะสำหรับ PostgreSQL โดยไม่ผ่าน `database/sql` interface เลย ข้อดีคือสามารถใช้ฟีเจอร์เฉพาะของ PostgreSQL ได้เต็มที่ เช่น:

- **Binary protocol** — pgx สื่อสารกับ PostgreSQL ด้วย binary wire protocol ทำให้เร็วกว่าการแปลงเป็น text แบบที่ `database/sql` ต้องทำ
- **รองรับ native type ของ PostgreSQL** เช่น arrays, JSON/JSONB, ranges, composite types, `numeric` แบบไม่สูญเสีย precision
- **Batch queries** ผ่าน `pgx.Batch` เพื่อส่งหลาย query ในรอบ network เดียว
- **Copy protocol** (`pgx.CopyFrom`) สำหรับ bulk insert ที่เร็วมาก
- **Listen/Notify** สำหรับ PostgreSQL pub/sub
- **Connection pooling ในตัว** ผ่าน `pgxpool` (ไม่ต้องพึ่ง `database/sql`'s pool ที่ค่อนข้างพื้นฐาน)

```go
package main

import (
    "context"
    "log"

    "github.com/jackc/pgx/v5"
)

func main() {
    ctx := context.Background()
    conn, err := pgx.Connect(ctx, "postgres://app_user:secret@localhost:5432/ecommerce")
    if err != nil {
        log.Fatalf("pgx.Connect ล้มเหลว: %v", err)
    }
    defer conn.Close(ctx)

    log.Println("เชื่อมต่อฐานข้อมูลสำเร็จผ่าน pgx โดยตรง")
}
```

### ตารางเปรียบเทียบเบื้องต้น

| ประเด็น | `database/sql` | `pgx` โดยตรง |
|---|---|---|
| ความเร็ว | ช้ากว่าเล็กน้อย (overhead ของ generic interface) | เร็วกว่า เพราะใช้ binary protocol เต็มรูปแบบ |
| ฟีเจอร์เฉพาะ PostgreSQL | จำกัด (ผ่าน type assertion เท่านั้น) | ครบถ้วน (arrays, JSONB, LISTEN/NOTIFY, COPY) |
| Compatibility กับ ORM/เครื่องมืออื่น | สูง (ORM ส่วนใหญ่คาดหวัง `*sql.DB`) | ต่ำกว่า (ต้องพึ่ง adapter) |
| ความซับซ้อนในการเรียนรู้ | ต่ำ (interface มาตรฐาน) | สูงกว่าเล็กน้อย (API เฉพาะตัว) |
| Connection pooling | มีในตัว (`sql.DB`) แต่ค่อนข้างพื้นฐาน | `pgxpool` ที่ยืดหยุ่นและปรับแต่งได้มากกว่า |

**คำแนะนำในทางปฏิบัติ**: ถ้าโปรเจกต์ใหม่ที่ใช้ PostgreSQL เท่านั้น และต้องการประสิทธิภาพสูงสุดพร้อมฟีเจอร์ครบ ให้ใช้ `pgx` โดยตรงผ่าน `pgxpool` ถ้าต้องการใช้ร่วมกับ ORM อื่น หรือเครื่องมือที่คาดหวัง `*sql.DB` (เช่น `sqlx`, บาง migration tool) ให้ใช้ `pgx/v5/stdlib` เป็น adapter ซึ่งจะได้ทั้งความเร็วของ pgx และ compatibility ของ `database/sql` ไปพร้อมกัน

---

## Step 882: pgx พื้นฐาน — การติดตั้ง, pgx.Connect, Query/QueryRow/Exec

### การติดตั้ง

```bash
go get github.com/jackc/pgx/v5
```

pgx v5 ต้องการ Go 1.19 ขึ้นไป และใช้ `context.Context` ในทุกฟังก์ชันที่คุยกับฐานข้อมูล ซึ่งเป็นแนวปฏิบัติมาตรฐานของ Go สมัยใหม่ (รองรับ cancellation และ timeout)

### `pgx.Connect` — การเชื่อมต่อเดี่ยว

```go
package main

import (
    "context"
    "fmt"
    "log"
    "os"

    "github.com/jackc/pgx/v5"
)

func main() {
    ctx := context.Background()

    connString := os.Getenv("DATABASE_URL")
    conn, err := pgx.Connect(ctx, connString)
    if err != nil {
        log.Fatalf("ไม่สามารถเชื่อมต่อฐานข้อมูล: %v", err)
    }
    defer conn.Close(ctx)

    var version string
    err = conn.QueryRow(ctx, "SELECT version()").Scan(&version)
    if err != nil {
        log.Fatalf("query ล้มเหลว: %v", err)
    }
    fmt.Println("PostgreSQL version:", version)
}
```

`pgx.Connect` เหมาะสำหรับสคริปต์เล็ก ๆ หรือ CLI tool ที่ใช้ connection เดียวตลอดโปรแกรม แต่ **ไม่เหมาะกับเว็บแอปพลิเคชันที่ต้องรองรับหลาย request พร้อมกัน** เพราะ connection เดียวใช้ query พร้อมกันหลาย goroutine ไม่ได้ (ต้องใช้ pool แทน ซึ่งจะพูดถึงใน Step 883)

### `Exec` — สำหรับคำสั่งที่ไม่คืนผลลัพธ์เป็นแถว (INSERT/UPDATE/DELETE)

```go
func insertProduct(ctx context.Context, conn *pgx.Conn, name string, price float64, qty int) error {
    tag, err := conn.Exec(ctx,
        `INSERT INTO products (product_name, unit_price, stock_quantity)
         VALUES ($1, $2, $3)`,
        name, price, qty,
    )
    if err != nil {
        return fmt.Errorf("insertProduct: %w", err)
    }
    log.Printf("insert สำเร็จ, จำนวนแถวที่ได้รับผลกระทบ: %d\n", tag.RowsAffected())
    return nil
}
```

สังเกตว่า pgx ใช้ **positional parameter** แบบ `$1`, `$2`, `$3` ตามธรรมชาติของ PostgreSQL (ต่างจาก MySQL ที่ใช้ `?`) การส่ง parameter แบบนี้คือ **parameterized query** ที่ป้องกัน SQL injection ได้อย่างสมบูรณ์ เพราะค่าพารามิเตอร์จะถูกส่งแยกจาก SQL text ไปยัง PostgreSQL โดยตรง ไม่มีการต่อ string ใด ๆ

```go
// ห้ามทำแบบนี้เด็ดขาด — เสี่ยงต่อ SQL injection
query := fmt.Sprintf("SELECT * FROM products WHERE product_name = '%s'", userInput)

// ทำแบบนี้เสมอ
rows, err := conn.Query(ctx, "SELECT * FROM products WHERE product_name = $1", userInput)
```

### `QueryRow` — สำหรับผลลัพธ์ที่คาดหวังแค่ 1 แถว

```go
func getProductByID(ctx context.Context, conn *pgx.Conn, id int) (string, float64, error) {
    var name string
    var price float64

    err := conn.QueryRow(ctx,
        "SELECT product_name, unit_price FROM products WHERE product_id = $1",
        id,
    ).Scan(&name, &price)

    if err != nil {
        if errors.Is(err, pgx.ErrNoRows) {
            return "", 0, fmt.Errorf("ไม่พบสินค้ารหัส %d", id)
        }
        return "", 0, fmt.Errorf("getProductByID: %w", err)
    }
    return name, price, nil
}
```

จุดสำคัญคือ `QueryRow` จะไม่คืน error ทันทีแม้ query จะมีปัญหา — error จะถูกเก็บไว้และคืนออกมาตอนเรียก `.Scan()` เท่านั้น และถ้าไม่พบแถวใดเลย จะได้ error พิเศษคือ `pgx.ErrNoRows` ซึ่งต้องตรวจสอบด้วย `errors.Is` เสมอ

### `Query` — สำหรับผลลัพธ์หลายแถว

```go
func listProducts(ctx context.Context, conn *pgx.Conn) error {
    rows, err := conn.Query(ctx,
        "SELECT product_id, product_name, unit_price, stock_quantity FROM products ORDER BY product_id",
    )
    if err != nil {
        return fmt.Errorf("listProducts query: %w", err)
    }
    defer rows.Close() // สำคัญมาก: ต้องปิด rows เสมอ ไม่ว่าจะอ่านจบหรือไม่

    for rows.Next() {
        var id, qty int
        var name string
        var price float64

        if err := rows.Scan(&id, &name, &price, &qty); err != nil {
            return fmt.Errorf("scan แถวล้มเหลว: %w", err)
        }
        fmt.Printf("#%d %s ราคา %.2f คงเหลือ %d ชิ้น\n", id, name, price, qty)
    }

    // ต้องเช็ค error หลังวน loop เสมอ — rows.Next() คืน false ได้ทั้งจากอ่านจบ
    // และจาก error ระหว่างทาง (เช่น connection หลุด)
    if err := rows.Err(); err != nil {
        return fmt.Errorf("เกิดข้อผิดพลาดระหว่างอ่านแถว: %w", err)
    }
    return nil
}
```

**ข้อควรระวังที่พบบ่อย**: การลืม `defer rows.Close()` เป็นสาเหตุอันดับหนึ่งของ connection leak ในแอปพลิเคชัน Go ที่ใช้ pgx เมื่อ `rows` ยังไม่ถูกปิด connection ที่ใช้อยู่จะไม่ถูกคืนกลับ pool (ในกรณีที่ใช้ pool) ทำให้ pool ค่อย ๆ หมด connection จนแอปหยุดทำงาน

---

## Step 883: pgxpool สำหรับ Connection Pooling

### ทำไมต้องใช้ pool

การสร้าง connection ใหม่ทุกครั้งที่มี HTTP request เข้ามาเป็นการสิ้นเปลืองทรัพยากรมาก เพราะการทำ TCP handshake + TLS handshake (ถ้ามี) + PostgreSQL authentication ใช้เวลาหลายมิลลิวินาที ในขณะที่ query จริงอาจใช้เวลาแค่เศษเสี้ยวมิลลิวินาที **Connection pool** คือชุด connection ที่เปิดค้างไว้ล่วงหน้า แล้วให้แต่ละ request "ยืม" connection จาก pool ไปใช้ชั่วคราว แล้วคืนกลับเมื่อใช้เสร็จ

`pgxpool` (`github.com/jackc/pgx/v5/pgxpool`) คือ pool implementation ที่มากับ pgx โดยตรง

```bash
go get github.com/jackc/pgx/v5/pgxpool
```

### การสร้างและตั้งค่า pool

```go
package db

import (
    "context"
    "fmt"
    "time"

    "github.com/jackc/pgx/v5/pgxpool"
)

func NewPool(ctx context.Context, connString string) (*pgxpool.Pool, error) {
    config, err := pgxpool.ParseConfig(connString)
    if err != nil {
        return nil, fmt.Errorf("parse config ล้มเหลว: %w", err)
    }

    // ตั้งค่าขนาด pool
    config.MaxConns = 25                       // จำนวน connection สูงสุดที่ pool เปิดพร้อมกันได้
    config.MinConns = 5                        // จำนวน connection ขั้นต่ำที่คงไว้เสมอ (warm pool)
    config.MaxConnLifetime = time.Hour          // connection แต่ละตัวมีอายุสูงสุดเท่าใด ก่อนถูกปิดแล้วสร้างใหม่
    config.MaxConnIdleTime = 30 * time.Minute   // connection ที่ไม่ได้ใช้งานนานแค่ไหนจึงถูกปิดทิ้ง
    config.HealthCheckPeriod = time.Minute      // ความถี่ในการตรวจสอบสุขภาพ connection ที่ idle

    pool, err := pgxpool.NewWithConfig(ctx, config)
    if err != nil {
        return nil, fmt.Errorf("สร้าง pool ล้มเหลว: %w", err)
    }

    // ตรวจสอบว่า pool เชื่อมต่อได้จริง ไม่ใช่แค่ parse config ผ่าน
    if err := pool.Ping(ctx); err != nil {
        pool.Close()
        return nil, fmt.Errorf("ping pool ล้มเหลว: %w", err)
    }

    return pool, nil
}
```

หรือถ้าไม่ต้องการปรับแต่งละเอียด สามารถใช้ `pgxpool.New` ตรง ๆ ได้:

```go
pool, err := pgxpool.New(ctx, connString)
```

### การกำหนดขนาด pool ให้เหมาะสม

การตั้งค่า `MaxConns` เป็นเรื่องสำคัญที่ต้องคำนึงถึงทั้งฝั่ง Go application และฝั่ง PostgreSQL:

- PostgreSQL มี `max_connections` จำกัด (default 100) และแต่ละ connection ใช้ RAM ประมาณ 5-10 MB บน PostgreSQL server ดังนั้นถ้ามีหลาย instance ของแอป (เช่น 10 pods) แต่ละตัวตั้ง `MaxConns = 25` รวมแล้วจะใช้ถึง 250 connections ซึ่งอาจเกิน `max_connections` ของ server
- สูตรคร่าว ๆ ที่นิยมใช้คือ `connections = ((core_count * 2) + effective_spindle_count)` ของ PostgreSQL server (มาจากแนวทางของ PgBouncer/HikariCP) แต่ในทางปฏิบัติควรวัดผลจริงจาก load testing
- ถ้ามีหลาย instance ของแอปพลิเคชัน ควรพิจารณาใช้ **PgBouncer** เป็นตัวกลางทำ connection pooling อีกชั้น เพื่อจำกัดจำนวน connection จริงที่ไปถึง PostgreSQL

```go
// ตัวอย่างการคำนวณ pool size แบบง่าย สำหรับแอปที่รันบน container เดียว
// สมมติ PostgreSQL server มี max_connections = 100
// และมี service อื่นที่ใช้ฐานข้อมูลเดียวกันอยู่แล้วประมาณ 20 connections
// เหลือให้แอปนี้ใช้ได้ประมาณ 20-30 connections ต่อ instance
config.MaxConns = 20
```

### การใช้ pool ในการ query

รูปแบบการเรียกใช้งาน pool แทบจะเหมือนกับ `pgx.Conn` ทุกประการ เพราะ `pgxpool.Pool` implement interface (`Query`, `QueryRow`, `Exec`, `Begin`) เหมือนกัน — ข้อแตกต่างคือ pool จะจัดการ acquire/release connection ให้อัตโนมัติในทุกเมธอด

```go
func GetProductStock(ctx context.Context, pool *pgxpool.Pool, productID int) (int, error) {
    var qty int
    err := pool.QueryRow(ctx,
        "SELECT stock_quantity FROM products WHERE product_id = $1",
        productID,
    ).Scan(&qty)
    if err != nil {
        return 0, fmt.Errorf("GetProductStock: %w", err)
    }
    return qty, nil
}
```

ถ้าต้องการควบคุม connection เอง (เช่น ต้องรันหลาย statement บน connection เดียวกันโดยไม่ผ่าน transaction) สามารถใช้ `pool.Acquire()`:

```go
func withDedicatedConnection(ctx context.Context, pool *pgxpool.Pool) error {
    conn, err := pool.Acquire(ctx)
    if err != nil {
        return fmt.Errorf("acquire connection ล้มเหลว: %w", err)
    }
    defer conn.Release() // คืน connection กลับ pool เสมอ

    _, err = conn.Exec(ctx, "SET LOCAL statement_timeout = '5s'")
    if err != nil {
        return err
    }
    // ... ใช้งาน conn ต่อ
    return nil
}
```

### การปิด pool ตอน application shutdown

```go
func main() {
    ctx := context.Background()
    pool, err := NewPool(ctx, os.Getenv("DATABASE_URL"))
    if err != nil {
        log.Fatal(err)
    }
    defer pool.Close() // ปิด connection ทั้งหมดใน pool อย่างสุภาพตอนโปรแกรมจบ

    // ... start HTTP server ฯลฯ
}
```

`pool.Close()` จะรอให้ connection ที่กำลังถูกใช้งานอยู่คืนกลับมาก่อน แล้วจึงปิดทั้งหมด เหมาะสำหรับใช้คู่กับ graceful shutdown pattern ของ HTTP server

---

## Step 884: Row Scanning เข้า struct, pgx.CollectRows, และ Error Handling แบบ Go Idiomatic

### การ scan เข้า struct ทีละ field (แบบดั้งเดิม)

```go
type Product struct {
    ProductID     int
    ProductName   string
    UnitPrice     float64
    StockQuantity int
}

func getProduct(ctx context.Context, pool *pgxpool.Pool, id int) (*Product, error) {
    var p Product
    err := pool.QueryRow(ctx,
        `SELECT product_id, product_name, unit_price, stock_quantity
         FROM products WHERE product_id = $1`,
        id,
    ).Scan(&p.ProductID, &p.ProductName, &p.UnitPrice, &p.StockQuantity)

    if err != nil {
        if errors.Is(err, pgx.ErrNoRows) {
            return nil, nil // ไม่พบข้อมูล — คืน nil โดยไม่ error (แล้วแต่ convention ของทีม)
        }
        return nil, fmt.Errorf("getProduct: %w", err)
    }
    return &p, nil
}
```

วิธีนี้ใช้ได้ผล แต่เมื่อ struct มี field เยอะขึ้น การ list `&p.Field1, &p.Field2, ...` ตามลำดับคอลัมน์ใน SELECT จะเสี่ยงต่อการเขียนผิดลำดับ และแก้ไขยากเมื่อ schema เปลี่ยน

### `pgx.CollectRows` และ `pgx.RowToStructByName` (pgx v5)

pgx v5 เพิ่มชุดฟังก์ชันใน package `github.com/jackc/pgx/v5` ที่ช่วยลดโค้ด boilerplate ลงมาก โดยใช้ **struct tag** `db` เพื่อ map ชื่อคอลัมน์กับ field:

```go
type Product struct {
    ProductID     int     `db:"product_id"`
    ProductName   string  `db:"product_name"`
    UnitPrice     float64 `db:"unit_price"`
    StockQuantity int     `db:"stock_quantity"`
}

func ListProducts(ctx context.Context, pool *pgxpool.Pool) ([]Product, error) {
    rows, err := pool.Query(ctx,
        `SELECT product_id, product_name, unit_price, stock_quantity
         FROM products ORDER BY product_id`,
    )
    if err != nil {
        return nil, fmt.Errorf("ListProducts query: %w", err)
    }
    // ไม่ต้อง defer rows.Close() เอง — pgx.CollectRows จัดการปิดให้อัตโนมัติ

    products, err := pgx.CollectRows(rows, pgx.RowToStructByName[Product])
    if err != nil {
        return nil, fmt.Errorf("collect rows ล้มเหลว: %w", err)
    }
    return products, nil
}
```

`pgx.RowToStructByName[T]` จะ match ชื่อคอลัมน์จาก SELECT กับ struct tag `db:"..."` โดยอัตโนมัติ ทำให้ **ลำดับคอลัมน์ใน SELECT ไม่จำเป็นต้องตรงกับลำดับ field ใน struct อีกต่อไป** ซึ่งลดโอกาสเกิด bug ได้มาก

ถ้าต้องการดึงแค่ 1 แถวสามารถใช้ `pgx.CollectExactlyOneRow`:

```go
func GetProductByID(ctx context.Context, pool *pgxpool.Pool, id int) (Product, error) {
    rows, err := pool.Query(ctx,
        `SELECT product_id, product_name, unit_price, stock_quantity
         FROM products WHERE product_id = $1`,
        id,
    )
    if err != nil {
        return Product{}, fmt.Errorf("GetProductByID query: %w", err)
    }

    product, err := pgx.CollectExactlyOneRow(rows, pgx.RowToStructByName[Product])
    if err != nil {
        if errors.Is(err, pgx.ErrNoRows) {
            return Product{}, fmt.Errorf("ไม่พบสินค้ารหัส %d: %w", id, err)
        }
        return Product{}, fmt.Errorf("GetProductByID collect: %w", err)
    }
    return product, nil
}
```

`RowToStructByNameLax` เป็นอีกทางเลือกหนึ่งเมื่อ struct มี field ที่ไม่ได้อยู่ใน SELECT (ปกติ `RowToStructByName` จะ error ถ้า field ใด map ไม่ได้ แต่ `Lax` จะปล่อยผ่าน):

```go
products, err := pgx.CollectRows(rows, pgx.RowToStructByNameLax[Product])
```

### Error Handling แบบ Go Idiomatic

หัวใจของการเขียน Go ที่ดีคือรูปแบบ `if err != nil { return ... }` ที่ปรากฏซ้ำ ๆ ทุกจุดที่มีโอกาส error หลักการสำคัญที่ต้องยึดถือ:

1. **ตรวจสอบ error ทันทีหลังเรียกฟังก์ชันที่คืน error เสมอ** ไม่มีข้อยกเว้น
2. **ห่อ error ด้วย context เพิ่มเติมโดยใช้ `fmt.Errorf` กับ `%w`** เพื่อให้ error chain สามารถ unwrap กลับไปดู error ต้นตอได้ด้วย `errors.Is` / `errors.As`
3. **แยกแยะ error ที่คาดการณ์ได้ (เช่น ไม่พบข้อมูล) ออกจาก error ที่ไม่คาดคิด (เช่น connection หลุด)**

```go
import (
    "errors"

    "github.com/jackc/pgx/v5"
    "github.com/jackc/pgx/v5/pgconn"
)

func CreateProduct(ctx context.Context, pool *pgxpool.Pool, p Product) (int, error) {
    var newID int
    err := pool.QueryRow(ctx,
        `INSERT INTO products (product_name, unit_price, stock_quantity)
         VALUES ($1, $2, $3) RETURNING product_id`,
        p.ProductName, p.UnitPrice, p.StockQuantity,
    ).Scan(&newID)

    if err != nil {
        var pgErr *pgconn.PgError
        if errors.As(err, &pgErr) {
            switch pgErr.Code {
            case "23505": // unique_violation
                return 0, fmt.Errorf("สินค้านี้มีอยู่แล้วในระบบ: %w", err)
            case "23514": // check_violation
                return 0, fmt.Errorf("ข้อมูลไม่ผ่านเงื่อนไข check constraint: %w", err)
            }
        }
        return 0, fmt.Errorf("CreateProduct: %w", err)
    }
    return newID, nil
}
```

การใช้ `errors.As(err, &pgErr)` ช่วยให้เข้าถึง `pgconn.PgError` ซึ่งมี field เช่น `Code` (SQLSTATE), `Message`, `Detail`, `ConstraintName` ทำให้สามารถแยกแยะสาเหตุของ error ในระดับ PostgreSQL ได้ ไม่ใช่แค่ error message ทั่วไป

**ตารางรหัส error ที่พบบ่อย:**

| SQLSTATE | ความหมาย |
|---|---|
| `23505` | unique_violation — ข้อมูลซ้ำกับ unique constraint |
| `23503` | foreign_key_violation — ละเมิด foreign key |
| `23502` | not_null_violation — ค่าเป็น NULL ในคอลัมน์ที่ห้าม NULL |
| `23514` | check_violation — ไม่ผ่าน CHECK constraint |
| `40001` | serialization_failure — เกิด conflict ใน serializable transaction |
| `40P01` | deadlock_detected — เกิด deadlock |

---

## Step 885: Transaction ด้วย pool.Begin()...tx.Commit()/tx.Rollback()

### รูปแบบพื้นฐานของ transaction ด้วย defer pattern

Transaction ใน pgx เริ่มต้นด้วย `pool.Begin(ctx)` ซึ่งคืน `pgx.Tx` แล้วจบด้วย `tx.Commit(ctx)` เมื่อสำเร็จ หรือ `tx.Rollback(ctx)` เมื่อเกิด error รูปแบบมาตรฐานที่ปลอดภัยที่สุดใน Go คือการใช้ `defer` ร่วมกับ named return value หรือ flag บอกสถานะ:

```go
func TransferStock(ctx context.Context, pool *pgxpool.Pool, fromID, toID, qty int) error {
    tx, err := pool.Begin(ctx)
    if err != nil {
        return fmt.Errorf("begin transaction ล้มเหลว: %w", err)
    }
    // defer Rollback ไว้เสมอ — ถ้า Commit สำเร็จไปแล้ว Rollback ที่ตามมาจะไม่มีผลใด ๆ
    // (pgx จัดการเรื่องนี้ให้อัตโนมัติ ปลอดภัยที่จะเรียกซ้ำ)
    defer tx.Rollback(ctx)

    var currentStock int
    err = tx.QueryRow(ctx,
        "SELECT stock_quantity FROM products WHERE product_id = $1 FOR UPDATE",
        fromID,
    ).Scan(&currentStock)
    if err != nil {
        return fmt.Errorf("ตรวจสอบสต็อกต้นทางล้มเหลว: %w", err)
    }

    if currentStock < qty {
        return fmt.Errorf("สต็อกสินค้ารหัส %d ไม่เพียงพอ (มี %d ต้องการ %d)", fromID, currentStock, qty)
    }

    _, err = tx.Exec(ctx,
        "UPDATE products SET stock_quantity = stock_quantity - $1 WHERE product_id = $2",
        qty, fromID,
    )
    if err != nil {
        return fmt.Errorf("ลดสต็อกต้นทางล้มเหลว: %w", err)
    }

    _, err = tx.Exec(ctx,
        "UPDATE products SET stock_quantity = stock_quantity + $1 WHERE product_id = $2",
        qty, toID,
    )
    if err != nil {
        return fmt.Errorf("เพิ่มสต็อกปลายทางล้มเหลว: %w", err)
    }

    if err := tx.Commit(ctx); err != nil {
        return fmt.Errorf("commit transaction ล้มเหลว: %w", err)
    }
    return nil
}
```

**สังเกต pattern สำคัญ**: `defer tx.Rollback(ctx)` ถูกวางไว้ทันทีหลัง `Begin` สำเร็จ นี่คือ idiom มาตรฐานของ pgx (และ `database/sql`) เพราะ:

- ถ้าเกิด error ระหว่างทางแล้ว `return` ออกไปก่อนถึง `Commit` → `Rollback` จะถูกเรียกโดย defer อัตโนมัติ ย้อนกลับการเปลี่ยนแปลงทั้งหมด
- ถ้าทุกอย่างสำเร็จและเรียก `tx.Commit(ctx)` ไปแล้ว → การเรียก `tx.Rollback(ctx)` ซ้ำใน defer จะคืน error `pgx.ErrTxClosed` เงียบ ๆ โดยไม่กระทบอะไร (ไม่จำเป็นต้อง handle error ตัวนี้)

### การสร้างคำสั่งซื้อพร้อมตรวจสอบสต็อก (ตัวอย่างที่ใช้งานจริงกับ schema e-commerce)

```go
type OrderItem struct {
    ProductID int
    Quantity  int
}

func PlaceOrder(ctx context.Context, pool *pgxpool.Pool, customerID int, items []OrderItem) (int, error) {
    tx, err := pool.Begin(ctx)
    if err != nil {
        return 0, fmt.Errorf("begin transaction ล้มเหลว: %w", err)
    }
    defer tx.Rollback(ctx)

    // 1. สร้าง order หลัก
    var orderID int
    err = tx.QueryRow(ctx,
        `INSERT INTO orders (customer_id, status) VALUES ($1, 'pending') RETURNING order_id`,
        customerID,
    ).Scan(&orderID)
    if err != nil {
        return 0, fmt.Errorf("สร้าง order ล้มเหลว: %w", err)
    }

    // 2. ตรวจสอบและตัดสต็อกสินค้าแต่ละรายการ
    for _, item := range items {
        tag, err := tx.Exec(ctx,
            `UPDATE products
             SET stock_quantity = stock_quantity - $1
             WHERE product_id = $2 AND stock_quantity >= $1`,
            item.Quantity, item.ProductID,
        )
        if err != nil {
            return 0, fmt.Errorf("ตัดสต็อกสินค้ารหัส %d ล้มเหลว: %w", item.ProductID, err)
        }
        if tag.RowsAffected() == 0 {
            // ไม่มีแถวถูก update แปลว่าสต็อกไม่พอ หรือไม่มีสินค้านี้ — ทำให้ transaction ล้มเหลวทั้งหมด
            return 0, fmt.Errorf("สินค้ารหัส %d มีสต็อกไม่เพียงพอ", item.ProductID)
        }
    }

    if err := tx.Commit(ctx); err != nil {
        return 0, fmt.Errorf("commit order ล้มเหลว: %w", err)
    }
    return orderID, nil
}
```

Pattern การใช้เงื่อนไข `AND stock_quantity >= $1` ใน `UPDATE` แล้วเช็ค `RowsAffected()` เป็นเทคนิคที่หลีกเลี่ยง race condition ได้ดีกว่าการ `SELECT` ตรวจสอบก่อนแล้วค่อย `UPDATE` แยกกัน (ซึ่งมีช่องว่างให้ transaction อื่นแทรกเข้ามาระหว่างนั้นได้) เพราะ `UPDATE` ใน PostgreSQL จะ lock แถวโดยอัตโนมัติระหว่างประมวลผล

### การกำหนด isolation level

```go
tx, err := pool.BeginTx(ctx, pgx.TxOptions{
    IsoLevel:   pgx.Serializable,
    AccessMode: pgx.ReadWrite,
})
if err != nil {
    return fmt.Errorf("begin serializable transaction ล้มเหลว: %w", err)
}
defer tx.Rollback(ctx)
```

เมื่อใช้ `pgx.Serializable` ต้องเตรียมโค้ดสำหรับ **retry** เมื่อเกิด serialization failure (SQLSTATE `40001`) เพราะ PostgreSQL อาจ abort transaction เพื่อรักษาความถูกต้องของข้อมูล:

```go
func withSerializableRetry(ctx context.Context, pool *pgxpool.Pool, fn func(pgx.Tx) error) error {
    const maxRetries = 3
    for attempt := 0; attempt < maxRetries; attempt++ {
        tx, err := pool.BeginTx(ctx, pgx.TxOptions{IsoLevel: pgx.Serializable})
        if err != nil {
            return fmt.Errorf("begin tx ล้มเหลว: %w", err)
        }

        err = fn(tx)
        if err == nil {
            if commitErr := tx.Commit(ctx); commitErr == nil {
                return nil
            } else {
                err = commitErr
            }
        }
        tx.Rollback(ctx)

        var pgErr *pgconn.PgError
        if errors.As(err, &pgErr) && pgErr.Code == "40001" {
            log.Printf("serialization failure, ลองใหม่ครั้งที่ %d", attempt+1)
            continue
        }
        return err
    }
    return fmt.Errorf("transaction ล้มเหลวหลังลอง %d ครั้ง", maxRetries)
}
```

### Nested logical block ด้วย `pgx.Tx.Begin` (savepoint)

pgx รองรับ savepoint ผ่านการเรียก `Begin()` ซ้ำบน `pgx.Tx` ที่มีอยู่แล้ว:

```go
func processWithSavepoint(ctx context.Context, tx pgx.Tx) error {
    sp, err := tx.Begin(ctx) // สร้าง savepoint ภายใน transaction หลัก
    if err != nil {
        return err
    }
    defer sp.Rollback(ctx)

    _, err = sp.Exec(ctx, "UPDATE products SET stock_quantity = 0 WHERE product_id = 999")
    if err != nil {
        return err
    }

    return sp.Commit(ctx) // release savepoint (ไม่ commit transaction หลักจริง ๆ)
}
```

---

## Step 886: database/sql มาตรฐาน — เมื่อไหร่ควรใช้แทน pgx โดยตรง

แม้ pgx จะให้ประสิทธิภาพและฟีเจอร์ที่ดีกว่า แต่ก็มีหลายสถานการณ์ที่การใช้ `database/sql` (ผ่าน `pgx/v5/stdlib` เป็น driver) เป็นทางเลือกที่เหมาะสมกว่า

### สถานการณ์ที่ควรใช้ database/sql

**1. ต้องการใช้ร่วมกับ library ที่ออกแบบมาสำหรับ `*sql.DB` โดยเฉพาะ**

เช่น `sqlx` (extension ของ `database/sql` ที่เพิ่มความสะดวกในการ scan struct โดยยังคง API compatible), หรือ migration tool อย่าง `golang-migrate`, `goose` ที่ทำงานผ่าน `*sql.DB`

```go
import (
    "database/sql"

    _ "github.com/jackc/pgx/v5/stdlib"
    "github.com/jmoiron/sqlx"
)

func NewSqlxDB(connString string) (*sqlx.DB, error) {
    db, err := sqlx.Open("pgx", connString)
    if err != nil {
        return nil, err
    }
    return db, nil
}

type Product struct {
    ProductID     int     `db:"product_id"`
    ProductName   string  `db:"product_name"`
    UnitPrice     float64 `db:"unit_price"`
    StockQuantity int     `db:"stock_quantity"`
}

func GetAllProducts(db *sqlx.DB) ([]Product, error) {
    var products []Product
    err := db.Select(&products, "SELECT * FROM products ORDER BY product_id")
    return products, err
}
```

**2. โปรเจกต์ที่ต้องรองรับฐานข้อมูลหลายชนิด (database-agnostic)**

ถ้าแอปพลิเคชันต้องรองรับทั้ง PostgreSQL, MySQL, SQLite (เช่น เป็น open-source tool ที่ผู้ใช้เลือกฐานข้อมูลเองได้) การเขียนโค้ดผ่าน `database/sql` interface ทำให้สลับ driver ได้โดยแก้แค่บรรทัด `sql.Open()` เท่านั้น ส่วน pgx ผูกกับ PostgreSQL เพียงอย่างเดียว

**3. GORM และ ORM ส่วนใหญ่ในระบบนิเวศ Go ใช้ database/sql เป็นฐาน**

GORM เองก็ใช้ driver ที่ครอบ `database/sql` อยู่ภายใน (จะกล่าวถึงในหัวข้อถัดไป)

### วิธีตั้งค่า connection pool ของ database/sql เมื่อใช้ pgx/stdlib

```go
package main

import (
    "database/sql"
    "time"

    _ "github.com/jackc/pgx/v5/stdlib"
)

func NewStdlibDB(connString string) (*sql.DB, error) {
    db, err := sql.Open("pgx", connString)
    if err != nil {
        return nil, err
    }

    db.SetMaxOpenConns(25)                  // เทียบเท่า pgxpool.MaxConns
    db.SetMaxIdleConns(5)                   // เทียบเท่า pgxpool.MinConns (คร่าว ๆ)
    db.SetConnMaxLifetime(time.Hour)        // เทียบเท่า pgxpool.MaxConnLifetime
    db.SetConnMaxIdleTime(30 * time.Minute) // เทียบเท่า pgxpool.MaxConnIdleTime

    return db, nil
}
```

### ตารางสรุปว่าควรเลือกแบบไหน

| ต้องการ... | แนะนำให้ใช้ |
|---|---|
| ประสิทธิภาพสูงสุด, ใช้ฟีเจอร์เฉพาะ PostgreSQL (arrays, JSONB, LISTEN/NOTIFY, COPY) | pgx โดยตรง (`pgxpool`) |
| ใช้ร่วมกับ `sqlx`, migration tools, หรือ library ที่ต้องการ `*sql.DB` | `database/sql` ผ่าน `pgx/v5/stdlib` |
| ใช้ GORM หรือ ORM อื่น | ตาม driver ที่ ORM นั้นรองรับ (มักครอบ `database/sql`) |
| รองรับหลายฐานข้อมูล (database-agnostic) | `database/sql` |
| เขียนสคริปต์เล็ก ๆ ใช้ connection เดียว | `pgx.Connect` |

ในทางปฏิบัติ ทีมจำนวนมากเลือก "ทางสายกลาง" คือใช้ `pgxpool` เป็นหลักสำหรับ business logic ที่ perf-critical และใช้ `pgx/v5/stdlib` เฉพาะจุดที่ต้องพึ่ง tool ภายนอก เช่น migration

---

## Step 887: GORM แนะนำตัว — struct tag, AutoMigrate, การเชื่อมต่อผ่าน postgres driver

### GORM คืออะไร

GORM (`gorm.io/gorm`) เป็น ORM (Object-Relational Mapping) ที่ได้รับความนิยมสูงสุดในระบบนิเวศ Go แนวคิดหลักคือให้นักพัฒนาทำงานกับ Go struct แทนการเขียน SQL ตรง ๆ โดย GORM จะแปลง operation บน struct เป็น SQL ให้อัตโนมัติ เหมาะสำหรับโปรเจกต์ที่ต้องการความเร็วในการพัฒนา (developer productivity) มากกว่าประสิทธิภาพสูงสุดแบบ raw SQL

### การติดตั้ง

```bash
go get -u gorm.io/gorm
go get -u gorm.io/driver/postgres
```

`gorm.io/driver/postgres` ภายในใช้ `pgx` เป็น driver เบื้องหลัง (ผ่าน `database/sql`) ทำให้ได้ประสิทธิภาพที่ดีในระดับหนึ่ง

### การเชื่อมต่อฐานข้อมูล

```go
package main

import (
    "log"

    "gorm.io/driver/postgres"
    "gorm.io/gorm"
    "gorm.io/gorm/logger"
)

func NewGormDB(dsn string) (*gorm.DB, error) {
    db, err := gorm.Open(postgres.Open(dsn), &gorm.Config{
        Logger: logger.Default.LogMode(logger.Info), // log ทุก SQL ที่ GORM รันออก (มีประโยชน์ตอน dev)
    })
    if err != nil {
        return nil, err
    }
    return db, nil
}

func main() {
    dsn := "host=localhost user=app_user password=secret dbname=ecommerce port=5432 sslmode=disable TimeZone=Asia/Bangkok"
    db, err := NewGormDB(dsn)
    if err != nil {
        log.Fatalf("เชื่อมต่อฐานข้อมูลผ่าน GORM ล้มเหลว: %v", err)
    }

    sqlDB, err := db.DB() // ดึง *sql.DB ที่อยู่เบื้องหลัง เพื่อตั้งค่า pool
    if err != nil {
        log.Fatal(err)
    }
    sqlDB.SetMaxOpenConns(25)
    sqlDB.SetMaxIdleConns(5)

    log.Println("เชื่อมต่อฐานข้อมูลผ่าน GORM สำเร็จ")
}
```

**ข้อสังเกตสำคัญ**: `gorm.Open` คืน `*gorm.DB` ซึ่งไม่ใช่ connection pool โดยตรง แต่เป็น "session/context" object การตั้งค่า connection pool จริง ๆ ต้องดึง `*sql.DB` ผ่าน `db.DB()` แล้วเรียก `SetMaxOpenConns` เหมือนกับ `database/sql` ทั่วไป

### การนิยาม struct model และ struct tag

```go
package model

import "time"

type Product struct {
    ProductID     uint    `gorm:"column:product_id;primaryKey"`
    ProductName   string  `gorm:"column:product_name;size:150"`
    UnitPrice     float64 `gorm:"column:unit_price;type:numeric(10,2)"`
    StockQuantity int     `gorm:"column:stock_quantity"`
}

// TableName บอก GORM ว่า struct นี้ผูกกับตารางชื่ออะไร
// (ปกติ GORM จะเดาชื่อตารางเป็นพหูพจน์ของชื่อ struct แบบ snake_case โดยอัตโนมัติ
// เช่น Product -> "products" แต่การระบุชัดเจนช่วยลดความสับสน)
func (Product) TableName() string {
    return "products"
}

type Customer struct {
    CustomerID uint   `gorm:"column:customer_id;primaryKey"`
    FirstName  string `gorm:"column:first_name;size:60"`
    Email      string `gorm:"column:email;size:150;uniqueIndex"`
    Orders     []Order `gorm:"foreignKey:CustomerID"` // ความสัมพันธ์ has many (ดู Step 889)
}

func (Customer) TableName() string {
    return "customers"
}

type Order struct {
    OrderID    uint      `gorm:"column:order_id;primaryKey"`
    CustomerID uint      `gorm:"column:customer_id"`
    OrderDate  time.Time `gorm:"column:order_date;default:now()"`
    Status     string    `gorm:"column:status;size:20"`
    Customer   Customer  `gorm:"foreignKey:CustomerID"` // ความสัมพันธ์ belongs to (ดู Step 889)
}

func (Order) TableName() string {
    return "orders"
}
```

Struct tag `gorm:"..."` ที่ใช้บ่อย:

| Tag | ความหมาย |
|---|---|
| `column:name` | ระบุชื่อคอลัมน์ในฐานข้อมูล (ถ้าไม่ระบุ GORM แปลง field name เป็น snake_case อัตโนมัติ) |
| `primaryKey` | กำหนดให้ field นี้เป็น primary key |
| `size:150` | กำหนดความยาวของ `VARCHAR` |
| `type:numeric(10,2)` | กำหนด PostgreSQL type ที่ต้องการชัดเจน |
| `default:now()` | ค่า default ระดับฐานข้อมูล |
| `uniqueIndex` | สร้าง unique index ให้คอลัมน์นี้ |
| `not null` | กำหนด `NOT NULL` constraint |
| `foreignKey:CustomerID` | ระบุ field ที่ใช้เป็น foreign key ในความสัมพันธ์ |
| `-` | บอกให้ GORM ไม่ต้อง map field นี้เข้าฐานข้อมูลเลย |

### AutoMigrate

`AutoMigrate` คือฟีเจอร์ของ GORM ที่สร้างหรือปรับปรุงโครงสร้างตารางให้ตรงกับ struct definition โดยอัตโนมัติ

```go
func RunMigrations(db *gorm.DB) error {
    return db.AutoMigrate(&model.Customer{}, &model.Product{}, &model.Order{})
}
```

**ข้อควรระวังเรื่อง AutoMigrate ในสภาพแวดล้อม production**:

1. `AutoMigrate` จะสร้างตารางที่ยังไม่มี, เพิ่มคอลัมน์ที่ขาด, เพิ่ม index ที่ขาด — แต่ **จะไม่ลบคอลัมน์หรือ index ที่ไม่ได้ใช้แล้วออก** (เพื่อความปลอดภัย ป้องกันข้อมูลสูญหายโดยไม่ตั้งใจ)
2. ไม่เหมาะกับการเปลี่ยนแปลง schema ที่ซับซ้อน เช่น การเปลี่ยน type ของคอลัมน์ที่มีข้อมูลอยู่แล้ว, การ rename คอลัมน์ (AutoMigrate จะมองว่าเป็นการสร้างคอลัมน์ใหม่ ไม่ใช่ rename)
3. สำหรับโปรเจกต์ระดับ production ควรใช้ dedicated migration tool เช่น `golang-migrate`, `goose`, หรือ `atlas` ที่เขียน migration file แบบ versioned และ reviewable ผ่าน code review แทนการพึ่ง `AutoMigrate` เพียงอย่างเดียว
4. `AutoMigrate` เหมาะสำหรับช่วง prototype, การพัฒนาเบื้องต้น, หรือ test environment ที่ต้องการความรวดเร็วในการ setup

```go
// ตัวอย่างการใช้ AutoMigrate ร่วมกับ raw migration สำหรับ production
func SetupDatabase(db *gorm.DB, isDev bool) error {
    if isDev {
        // dev/test: ใช้ AutoMigrate เพื่อความสะดวกรวดเร็ว
        return db.AutoMigrate(&model.Customer{}, &model.Product{}, &model.Order{})
    }
    // production: ใช้ migration tool ภายนอกจัดการแทน ที่นี่แค่ verify ว่า schema พร้อมใช้งาน
    return db.Exec("SELECT 1 FROM products LIMIT 1").Error
}
```

---

## Step 888: GORM — CRUD Operations พื้นฐาน เทียบเคียงกับ Raw SQL

ในหัวข้อนี้จะแสดงการทำ CRUD (Create, Read, Update, Delete) ด้วย GORM คู่กับ raw SQL ที่เขียนด้วย pgx เพื่อให้เห็นภาพความแตกต่างชัดเจน

### Create

**ด้วย GORM:**

```go
func CreateProductGorm(db *gorm.DB, p *model.Product) error {
    result := db.Create(p) // GORM จะ INSERT และเติม p.ProductID ที่ generate กลับมาให้อัตโนมัติ
    if result.Error != nil {
        return fmt.Errorf("CreateProductGorm: %w", result.Error)
    }
    return nil
}

// เรียกใช้งาน
newProduct := &model.Product{
    ProductName:   "หูฟังบลูทูธ",
    UnitPrice:     1590.00,
    StockQuantity: 40,
}
if err := CreateProductGorm(db, newProduct); err != nil {
    log.Fatal(err)
}
fmt.Println("สร้างสินค้าใหม่รหัส:", newProduct.ProductID)
```

**เทียบเท่าด้วย raw SQL (pgx):**

```go
func CreateProductPgx(ctx context.Context, pool *pgxpool.Pool, p *Product) error {
    return pool.QueryRow(ctx,
        `INSERT INTO products (product_name, unit_price, stock_quantity)
         VALUES ($1, $2, $3) RETURNING product_id`,
        p.ProductName, p.UnitPrice, p.StockQuantity,
    ).Scan(&p.ProductID)
}
```

### Read

**ด้วย GORM — ดึงแถวเดียวด้วย primary key:**

```go
func GetProductGorm(db *gorm.DB, id uint) (*model.Product, error) {
    var p model.Product
    result := db.First(&p, id) // SELECT * FROM products WHERE product_id = $1 ORDER BY product_id LIMIT 1
    if result.Error != nil {
        if errors.Is(result.Error, gorm.ErrRecordNotFound) {
            return nil, fmt.Errorf("ไม่พบสินค้ารหัส %d", id)
        }
        return nil, result.Error
    }
    return &p, nil
}
```

**ด้วย GORM — ดึงหลายแถวพร้อมเงื่อนไข:**

```go
func FindLowStockProducts(db *gorm.DB, threshold int) ([]model.Product, error) {
    var products []model.Product
    result := db.Where("stock_quantity < ?", threshold).
        Order("stock_quantity ASC").
        Find(&products)
    if result.Error != nil {
        return nil, fmt.Errorf("FindLowStockProducts: %w", result.Error)
    }
    return products, nil
}
```

**เทียบเท่าด้วย raw SQL (pgx):**

```go
func FindLowStockProductsPgx(ctx context.Context, pool *pgxpool.Pool, threshold int) ([]Product, error) {
    rows, err := pool.Query(ctx,
        `SELECT product_id, product_name, unit_price, stock_quantity
         FROM products WHERE stock_quantity < $1 ORDER BY stock_quantity ASC`,
        threshold,
    )
    if err != nil {
        return nil, fmt.Errorf("FindLowStockProductsPgx query: %w", err)
    }
    return pgx.CollectRows(rows, pgx.RowToStructByName[Product])
}
```

สังเกตว่า GORM ใช้ `?` เป็น placeholder ใน method chain (ซึ่ง GORM จะแปลงเป็น `$1`, `$2` ให้เองภายใต้ PostgreSQL dialect) ในขณะที่ pgx ใช้ `$1`, `$2` ตรง ๆ

### Update

**ด้วย GORM — update field เดียว:**

```go
func UpdateStockGorm(db *gorm.DB, productID uint, newQty int) error {
    result := db.Model(&model.Product{}).
        Where("product_id = ?", productID).
        Update("stock_quantity", newQty)
    if result.Error != nil {
        return fmt.Errorf("UpdateStockGorm: %w", result.Error)
    }
    if result.RowsAffected == 0 {
        return fmt.Errorf("ไม่พบสินค้ารหัส %d ให้อัปเดต", productID)
    }
    return nil
}
```

**ด้วย GORM — update หลาย field พร้อมกันด้วย struct (เฉพาะ field ที่ไม่ใช่ zero value):**

```go
func UpdateProductGorm(db *gorm.DB, productID uint, updates model.Product) error {
    // db.Updates ด้วย struct จะอัปเดตเฉพาะ field ที่ไม่ใช่ zero value เท่านั้น
    // ถ้าต้องการอัปเดตแม้เป็น zero value (เช่น ตั้งราคาเป็น 0) ให้ใช้ map[string]interface{} แทน
    result := db.Model(&model.Product{}).Where("product_id = ?", productID).Updates(updates)
    return result.Error
}

func UpdateProductGormMap(db *gorm.DB, productID uint, updates map[string]interface{}) error {
    result := db.Model(&model.Product{}).Where("product_id = ?", productID).Updates(updates)
    return result.Error
}
```

**เทียบเท่าด้วย raw SQL (pgx):**

```go
func UpdateStockPgx(ctx context.Context, pool *pgxpool.Pool, productID, newQty int) error {
    tag, err := pool.Exec(ctx,
        "UPDATE products SET stock_quantity = $1 WHERE product_id = $2",
        newQty, productID,
    )
    if err != nil {
        return fmt.Errorf("UpdateStockPgx: %w", err)
    }
    if tag.RowsAffected() == 0 {
        return fmt.Errorf("ไม่พบสินค้ารหัส %d ให้อัปเดต", productID)
    }
    return nil
}
```

### Delete

**ด้วย GORM:**

```go
func DeleteProductGorm(db *gorm.DB, productID uint) error {
    result := db.Delete(&model.Product{}, productID)
    if result.Error != nil {
        return fmt.Errorf("DeleteProductGorm: %w", result.Error)
    }
    if result.RowsAffected == 0 {
        return fmt.Errorf("ไม่พบสินค้ารหัส %d ให้ลบ", productID)
    }
    return nil
}
```

**เทียบเท่าด้วย raw SQL (pgx):**

```go
func DeleteProductPgx(ctx context.Context, pool *pgxpool.Pool, productID int) error {
    tag, err := pool.Exec(ctx, "DELETE FROM products WHERE product_id = $1", productID)
    if err != nil {
        return fmt.Errorf("DeleteProductPgx: %w", err)
    }
    if tag.RowsAffected() == 0 {
        return fmt.Errorf("ไม่พบสินค้ารหัส %d ให้ลบ", productID)
    }
    return nil
}
```

> **หมายเหตุเรื่อง Soft Delete**: ถ้า struct มี field ชื่อ `DeletedAt gorm.DeletedAt` GORM จะเปลี่ยนพฤติกรรม `Delete` เป็น **soft delete** โดยอัตโนมัติ (คือ `UPDATE ... SET deleted_at = now()` แทนการ `DELETE` จริง) และ query ปกติ (`Find`, `First`) จะกรองแถวที่ `deleted_at IS NOT NULL` ออกให้อัตโนมัติด้วย ถ้าต้องการลบจริง (hard delete) ต้องใช้ `db.Unscoped().Delete(...)`

### ตารางเปรียบเทียบ syntax GORM vs raw SQL (pgx)

| Operation | GORM | pgx (raw SQL) |
|---|---|---|
| Insert | `db.Create(&p)` | `INSERT ... RETURNING id` + `Scan` |
| Select by PK | `db.First(&p, id)` | `SELECT ... WHERE id = $1` + `QueryRow.Scan` |
| Select with condition | `db.Where(...).Find(&list)` | `SELECT ... WHERE ...` + `pgx.CollectRows` |
| Update field | `db.Model(&p).Update(...)` | `UPDATE ... SET ... WHERE ...` |
| Delete | `db.Delete(&p, id)` | `DELETE FROM ... WHERE id = $1` |
| Raw SQL เมื่อจำเป็น | `db.Raw("SELECT ...").Scan(&result)` | (ใช้ได้เต็มรูปแบบเสมอ) |

---

## Step 889: GORM — Association (belongs to, has many), Preload สำหรับ Eager Loading

### ความสัมพันธ์ belongs to และ has many

จาก schema ของเรา: `orders.customer_id` อ้างอิงไปยัง `customers.customer_id` ความสัมพันธ์นี้มองได้สองมุม:

- จากมุมของ `Order` → หนึ่ง order **belongs to** หนึ่ง customer
- จากมุมของ `Customer` → หนึ่ง customer **has many** orders

```go
type Customer struct {
    CustomerID uint    `gorm:"column:customer_id;primaryKey"`
    FirstName  string  `gorm:"column:first_name;size:60"`
    Email      string  `gorm:"column:email;size:150;uniqueIndex"`
    Orders     []Order `gorm:"foreignKey:CustomerID"` // has many
}

func (Customer) TableName() string { return "customers" }

type Order struct {
    OrderID    uint     `gorm:"column:order_id;primaryKey"`
    CustomerID uint     `gorm:"column:customer_id"`
    OrderDate  time.Time `gorm:"column:order_date"`
    Status     string   `gorm:"column:status;size:20"`
    Customer   Customer `gorm:"foreignKey:CustomerID"` // belongs to
}

func (Order) TableName() string { return "orders" }
```

GORM เดา foreign key โดยอัตโนมัติจากชื่อ struct + primary key (เช่น `CustomerID` จับคู่กับ `Customer.CustomerID`) แต่การระบุ `gorm:"foreignKey:CustomerID"` อย่างชัดเจนช่วยลดความกำกวม โดยเฉพาะเมื่อชื่อ field ไม่ตรงตาม convention มาตรฐาน

### ปัญหา N+1 Query

ถ้าไม่ระวัง การดึงข้อมูล order พร้อมข้อมูลลูกค้าของแต่ละ order แบบวน loop จะเกิดปัญหา **N+1 query** ซึ่งเป็นปัญหาประสิทธิภาพคลาสสิกที่พบบ่อยมากเมื่อใช้ ORM:

```go
// ตัวอย่างโค้ดที่มีปัญหา N+1 — ห้ามทำแบบนี้!
var orders []model.Order
db.Find(&orders) // query 1 ครั้งเพื่อดึง orders ทั้งหมด

for _, o := range orders {
    var customer model.Customer
    db.First(&customer, o.CustomerID) // query แยกสำหรับ "แต่ละ" order — ถ้ามี 1000 orders คือ 1000 query!
    fmt.Println(customer.FirstName, "->", o.Status)
}
```

ถ้ามี 1,000 orders โค้ดข้างต้นจะยิง query ไปที่ฐานข้อมูลถึง 1,001 ครั้ง (1 ครั้งสำหรับ orders + 1,000 ครั้งสำหรับ customer แต่ละราย) ซึ่งช้ามากเมื่อเทียบกับการ join ครั้งเดียว

### Preload — วิธีแก้ปัญหา N+1 ด้วย eager loading

```go
func ListOrdersWithCustomer(db *gorm.DB) ([]model.Order, error) {
    var orders []model.Order
    result := db.Preload("Customer").Find(&orders)
    if result.Error != nil {
        return nil, fmt.Errorf("ListOrdersWithCustomer: %w", result.Error)
    }
    return orders, nil
}
```

`Preload("Customer")` บอกให้ GORM ดึงข้อมูล `Customer` ที่เกี่ยวข้องมาด้วย แต่ **ไม่ได้ทำเป็น JOIN เดียว** — เบื้องหลัง GORM จะยิง 2 query แยกกัน:

```sql
-- query 1: ดึง orders ทั้งหมด
SELECT * FROM orders;

-- query 2: ดึง customers ทั้งหมดที่ id อยู่ใน list ของ customer_id จาก orders
SELECT * FROM customers WHERE customer_id IN (1, 2, 3, ...);
```

แล้ว GORM จะ "ประกอบ" ผลลัพธ์เข้าด้วยกันใน memory ให้อัตโนมัติ วิธีนี้ยังคงมีจำนวน query คงที่ (แค่ 2 ครั้ง ไม่ว่าจะมีกี่ orders) แทนที่จะเป็น N+1

### Preload สำหรับ has many — ดึงลูกค้าพร้อมรายการ orders ทั้งหมด

```go
func GetCustomerWithOrders(db *gorm.DB, customerID uint) (*model.Customer, error) {
    var customer model.Customer
    result := db.Preload("Orders").First(&customer, customerID)
    if result.Error != nil {
        if errors.Is(result.Error, gorm.ErrRecordNotFound) {
            return nil, fmt.Errorf("ไม่พบลูกค้ารหัส %d", customerID)
        }
        return nil, result.Error
    }
    return &customer, nil
}

// ใช้งาน
customer, err := GetCustomerWithOrders(db, 1)
if err != nil {
    log.Fatal(err)
}
fmt.Printf("ลูกค้า %s มีคำสั่งซื้อทั้งหมด %d รายการ\n", customer.FirstName, len(customer.Orders))
for _, o := range customer.Orders {
    fmt.Printf("  - order #%d สถานะ: %s\n", o.OrderID, o.Status)
}
```

### Preload พร้อมเงื่อนไข (conditional preload)

```go
func GetCustomerWithPendingOrders(db *gorm.DB, customerID uint) (*model.Customer, error) {
    var customer model.Customer
    result := db.Preload("Orders", "status = ?", "pending").First(&customer, customerID)
    if result.Error != nil {
        return nil, result.Error
    }
    return &customer, nil
}
```

### Preload หลายชั้น (nested preload)

ถ้ามี struct เพิ่มเติมที่เชื่อม order กับ order items สามารถ preload ต่อกันเป็นลูกโซ่ได้:

```go
// สมมติมี OrderItem ที่เชื่อมกับ Order และ Product
db.Preload("Orders.OrderItems.Product").First(&customer, customerID)
```

### เปรียบเทียบกับการ JOIN ด้วย raw SQL

ในบางกรณีการทำ single JOIN query ด้วย raw SQL อาจมีประสิทธิภาพดีกว่า 2-query approach ของ `Preload` โดยเฉพาะเมื่อผลลัพธ์มีขนาดไม่ใหญ่มาก:

```go
type OrderWithCustomer struct {
    OrderID    int       `db:"order_id"`
    Status     string    `db:"status"`
    OrderDate  time.Time `db:"order_date"`
    CustomerName string  `db:"customer_name"`
}

func ListOrdersWithCustomerPgx(ctx context.Context, pool *pgxpool.Pool) ([]OrderWithCustomer, error) {
    rows, err := pool.Query(ctx, `
        SELECT o.order_id, o.status, o.order_date, c.first_name AS customer_name
        FROM orders o
        JOIN customers c ON c.customer_id = o.customer_id
        ORDER BY o.order_id
    `)
    if err != nil {
        return nil, fmt.Errorf("ListOrdersWithCustomerPgx: %w", err)
    }
    return pgx.CollectRows(rows, pgx.RowToStructByName[OrderWithCustomer])
}
```

GORM เองก็รองรับการทำ JOIN ตรง ๆ ผ่าน `Joins()` ได้เช่นกัน หากต้องการควบคุมมากกว่า `Preload`:

```go
func ListOrdersJoinGorm(db *gorm.DB) ([]model.Order, error) {
    var orders []model.Order
    result := db.Joins("Customer").Find(&orders) // ทำ single JOIN แทนที่จะแยก query
    return orders, result.Error
}
```

**หลักการเลือกใช้**: `Preload` เหมาะกับกรณีที่ parent มีจำนวนไม่มากแต่ child มีจำนวนมาก (ลด duplication ของข้อมูล parent ที่ join ซ้ำ) ส่วน `Joins` เหมาะกับกรณีต้องการ filter บนคอลัมน์ของตารางที่ join หรือธุรกรรมที่ query ไม่ซับซ้อนมาก

---

## Step 890: แบบฝึกหัดรวม — เขียน HTTP API ด้วย net/http + pgx สำหรับจัดการสินค้าและคำสั่งซื้อ

หัวข้อนี้เป็นแบบฝึกหัดใหญ่ที่รวบรวมความรู้ทั้งหมดของบทนี้เข้าด้วยกัน โดยจะสร้าง HTTP API เล็ก ๆ ด้วย package `net/http` มาตรฐาน (ไม่ใช้ web framework ภายนอก) ร่วมกับ `pgx` สำหรับจัดการสินค้าและคำสั่งซื้อ

### โครงสร้างโปรเจกต์

```
ecommerce-api/
├── go.mod
├── main.go
├── db/
│   └── pool.go
├── model/
│   └── model.go
└── handler/
    ├── product.go
    └── order.go
```

### `model/model.go`

```go
package model

import "time"

type Product struct {
    ProductID     int     `json:"product_id" db:"product_id"`
    ProductName   string  `json:"product_name" db:"product_name"`
    UnitPrice     float64 `json:"unit_price" db:"unit_price"`
    StockQuantity int     `json:"stock_quantity" db:"stock_quantity"`
}

type Customer struct {
    CustomerID int    `json:"customer_id" db:"customer_id"`
    FirstName  string `json:"first_name" db:"first_name"`
    Email      string `json:"email" db:"email"`
}

type Order struct {
    OrderID    int       `json:"order_id" db:"order_id"`
    CustomerID int       `json:"customer_id" db:"customer_id"`
    OrderDate  time.Time `json:"order_date" db:"order_date"`
    Status     string    `json:"status" db:"status"`
}

type CreateOrderRequest struct {
    CustomerID int         `json:"customer_id"`
    Items      []OrderItem `json:"items"`
}

type OrderItem struct {
    ProductID int `json:"product_id"`
    Quantity  int `json:"quantity"`
}
```

### `db/pool.go`

```go
package db

import (
    "context"
    "fmt"
    "time"

    "github.com/jackc/pgx/v5/pgxpool"
)

func NewPool(ctx context.Context, connString string) (*pgxpool.Pool, error) {
    config, err := pgxpool.ParseConfig(connString)
    if err != nil {
        return nil, fmt.Errorf("parse config ล้มเหลว: %w", err)
    }
    config.MaxConns = 20
    config.MinConns = 2
    config.MaxConnLifetime = time.Hour
    config.MaxConnIdleTime = 30 * time.Minute

    pool, err := pgxpool.NewWithConfig(ctx, config)
    if err != nil {
        return nil, fmt.Errorf("สร้าง pool ล้มเหลว: %w", err)
    }
    if err := pool.Ping(ctx); err != nil {
        pool.Close()
        return nil, fmt.Errorf("ping ล้มเหลว: %w", err)
    }
    return pool, nil
}
```

### `handler/product.go`

```go
package handler

import (
    "encoding/json"
    "errors"
    "fmt"
    "net/http"
    "strconv"

    "github.com/jackc/pgx/v5"
    "github.com/jackc/pgx/v5/pgxpool"

    "ecommerce-api/model"
)

type ProductHandler struct {
    Pool *pgxpool.Pool
}

func NewProductHandler(pool *pgxpool.Pool) *ProductHandler {
    return &ProductHandler{Pool: pool}
}

// GET /products — คืนรายการสินค้าทั้งหมด
func (h *ProductHandler) List(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()

    rows, err := h.Pool.Query(ctx,
        `SELECT product_id, product_name, unit_price, stock_quantity
         FROM products ORDER BY product_id`,
    )
    if err != nil {
        writeError(w, http.StatusInternalServerError, fmt.Sprintf("query ล้มเหลว: %v", err))
        return
    }

    products, err := pgx.CollectRows(rows, pgx.RowToStructByName[model.Product])
    if err != nil {
        writeError(w, http.StatusInternalServerError, fmt.Sprintf("collect rows ล้มเหลว: %v", err))
        return
    }

    writeJSON(w, http.StatusOK, products)
}

// GET /products/{id} — คืนข้อมูลสินค้ารายชิ้น
func (h *ProductHandler) Get(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()

    id, err := strconv.Atoi(r.PathValue("id"))
    if err != nil {
        writeError(w, http.StatusBadRequest, "product id ไม่ถูกต้อง")
        return
    }

    var p model.Product
    err = h.Pool.QueryRow(ctx,
        `SELECT product_id, product_name, unit_price, stock_quantity
         FROM products WHERE product_id = $1`,
        id,
    ).Scan(&p.ProductID, &p.ProductName, &p.UnitPrice, &p.StockQuantity)

    if err != nil {
        if errors.Is(err, pgx.ErrNoRows) {
            writeError(w, http.StatusNotFound, fmt.Sprintf("ไม่พบสินค้ารหัส %d", id))
            return
        }
        writeError(w, http.StatusInternalServerError, fmt.Sprintf("query ล้มเหลว: %v", err))
        return
    }

    writeJSON(w, http.StatusOK, p)
}

// POST /products — สร้างสินค้าใหม่
func (h *ProductHandler) Create(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()

    var input model.Product
    if err := json.NewDecoder(r.Body).Decode(&input); err != nil {
        writeError(w, http.StatusBadRequest, "รูปแบบ JSON ไม่ถูกต้อง")
        return
    }

    if input.ProductName == "" {
        writeError(w, http.StatusBadRequest, "product_name ห้ามว่าง")
        return
    }
    if input.UnitPrice < 0 {
        writeError(w, http.StatusBadRequest, "unit_price ต้องไม่ติดลบ")
        return
    }

    err := h.Pool.QueryRow(ctx,
        `INSERT INTO products (product_name, unit_price, stock_quantity)
         VALUES ($1, $2, $3) RETURNING product_id`,
        input.ProductName, input.UnitPrice, input.StockQuantity,
    ).Scan(&input.ProductID)

    if err != nil {
        writeError(w, http.StatusInternalServerError, fmt.Sprintf("สร้างสินค้าล้มเหลว: %v", err))
        return
    }

    writeJSON(w, http.StatusCreated, input)
}

// PATCH /products/{id}/stock — ปรับปรุงจำนวนสต็อก
func (h *ProductHandler) UpdateStock(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()

    id, err := strconv.Atoi(r.PathValue("id"))
    if err != nil {
        writeError(w, http.StatusBadRequest, "product id ไม่ถูกต้อง")
        return
    }

    var body struct {
        StockQuantity int `json:"stock_quantity"`
    }
    if err := json.NewDecoder(r.Body).Decode(&body); err != nil {
        writeError(w, http.StatusBadRequest, "รูปแบบ JSON ไม่ถูกต้อง")
        return
    }
    if body.StockQuantity < 0 {
        writeError(w, http.StatusBadRequest, "stock_quantity ต้องไม่ติดลบ")
        return
    }

    tag, err := h.Pool.Exec(ctx,
        "UPDATE products SET stock_quantity = $1 WHERE product_id = $2",
        body.StockQuantity, id,
    )
    if err != nil {
        writeError(w, http.StatusInternalServerError, fmt.Sprintf("อัปเดตล้มเหลว: %v", err))
        return
    }
    if tag.RowsAffected() == 0 {
        writeError(w, http.StatusNotFound, fmt.Sprintf("ไม่พบสินค้ารหัส %d", id))
        return
    }

    w.WriteHeader(http.StatusNoContent)
}

// DELETE /products/{id} — ลบสินค้า
func (h *ProductHandler) Delete(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()

    id, err := strconv.Atoi(r.PathValue("id"))
    if err != nil {
        writeError(w, http.StatusBadRequest, "product id ไม่ถูกต้อง")
        return
    }

    tag, err := h.Pool.Exec(ctx, "DELETE FROM products WHERE product_id = $1", id)
    if err != nil {
        writeError(w, http.StatusInternalServerError, fmt.Sprintf("ลบล้มเหลว: %v", err))
        return
    }
    if tag.RowsAffected() == 0 {
        writeError(w, http.StatusNotFound, fmt.Sprintf("ไม่พบสินค้ารหัส %d", id))
        return
    }

    w.WriteHeader(http.StatusNoContent)
}
```

### `handler/order.go`

```go
package handler

import (
    "encoding/json"
    "errors"
    "fmt"
    "net/http"
    "strconv"

    "github.com/jackc/pgx/v5"
    "github.com/jackc/pgx/v5/pgxpool"

    "ecommerce-api/model"
)

type OrderHandler struct {
    Pool *pgxpool.Pool
}

func NewOrderHandler(pool *pgxpool.Pool) *OrderHandler {
    return &OrderHandler{Pool: pool}
}

// POST /orders — สร้างคำสั่งซื้อใหม่ พร้อมตัดสต็อกสินค้าใน transaction เดียว
func (h *OrderHandler) Create(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()

    var req model.CreateOrderRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        writeError(w, http.StatusBadRequest, "รูปแบบ JSON ไม่ถูกต้อง")
        return
    }
    if len(req.Items) == 0 {
        writeError(w, http.StatusBadRequest, "ต้องมีอย่างน้อย 1 รายการสินค้า")
        return
    }

    tx, err := h.Pool.Begin(ctx)
    if err != nil {
        writeError(w, http.StatusInternalServerError, fmt.Sprintf("begin transaction ล้มเหลว: %v", err))
        return
    }
    defer tx.Rollback(ctx)

    var orderID int
    err = tx.QueryRow(ctx,
        `INSERT INTO orders (customer_id, status) VALUES ($1, 'pending') RETURNING order_id`,
        req.CustomerID,
    ).Scan(&orderID)
    if err != nil {
        var pgErr *pgxPgError
        if errors.As(err, &pgErr) && pgErr.Code == "23503" {
            writeError(w, http.StatusBadRequest, fmt.Sprintf("ไม่พบลูกค้ารหัส %d", req.CustomerID))
            return
        }
        writeError(w, http.StatusInternalServerError, fmt.Sprintf("สร้าง order ล้มเหลว: %v", err))
        return
    }

    for _, item := range req.Items {
        if item.Quantity <= 0 {
            writeError(w, http.StatusBadRequest, "quantity ต้องมากกว่า 0")
            return
        }
        tag, err := tx.Exec(ctx,
            `UPDATE products
             SET stock_quantity = stock_quantity - $1
             WHERE product_id = $2 AND stock_quantity >= $1`,
            item.Quantity, item.ProductID,
        )
        if err != nil {
            writeError(w, http.StatusInternalServerError, fmt.Sprintf("ตัดสต็อกล้มเหลว: %v", err))
            return
        }
        if tag.RowsAffected() == 0 {
            writeError(w, http.StatusConflict,
                fmt.Sprintf("สินค้ารหัส %d มีสต็อกไม่เพียงพอ หรือไม่พบสินค้า", item.ProductID))
            return
        }
    }

    if err := tx.Commit(ctx); err != nil {
        writeError(w, http.StatusInternalServerError, fmt.Sprintf("commit ล้มเหลว: %v", err))
        return
    }

    writeJSON(w, http.StatusCreated, map[string]int{"order_id": orderID})
}

// GET /orders/{id} — ดูรายละเอียดคำสั่งซื้อ
func (h *OrderHandler) Get(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()

    id, err := strconv.Atoi(r.PathValue("id"))
    if err != nil {
        writeError(w, http.StatusBadRequest, "order id ไม่ถูกต้อง")
        return
    }

    var o model.Order
    err = h.Pool.QueryRow(ctx,
        `SELECT order_id, customer_id, order_date, status
         FROM orders WHERE order_id = $1`,
        id,
    ).Scan(&o.OrderID, &o.CustomerID, &o.OrderDate, &o.Status)

    if err != nil {
        if errors.Is(err, pgx.ErrNoRows) {
            writeError(w, http.StatusNotFound, fmt.Sprintf("ไม่พบคำสั่งซื้อรหัส %d", id))
            return
        }
        writeError(w, http.StatusInternalServerError, fmt.Sprintf("query ล้มเหลว: %v", err))
        return
    }

    writeJSON(w, http.StatusOK, o)
}

// PATCH /orders/{id}/status — เปลี่ยนสถานะคำสั่งซื้อ
func (h *OrderHandler) UpdateStatus(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()

    id, err := strconv.Atoi(r.PathValue("id"))
    if err != nil {
        writeError(w, http.StatusBadRequest, "order id ไม่ถูกต้อง")
        return
    }

    var body struct {
        Status string `json:"status"`
    }
    if err := json.NewDecoder(r.Body).Decode(&body); err != nil {
        writeError(w, http.StatusBadRequest, "รูปแบบ JSON ไม่ถูกต้อง")
        return
    }

    validStatuses := map[string]bool{"pending": true, "shipped": true, "completed": true, "cancelled": true}
    if !validStatuses[body.Status] {
        writeError(w, http.StatusBadRequest, "status ไม่ถูกต้อง (pending, shipped, completed, cancelled)")
        return
    }

    tag, err := h.Pool.Exec(ctx,
        "UPDATE orders SET status = $1 WHERE order_id = $2",
        body.Status, id,
    )
    if err != nil {
        writeError(w, http.StatusInternalServerError, fmt.Sprintf("อัปเดตล้มเหลว: %v", err))
        return
    }
    if tag.RowsAffected() == 0 {
        writeError(w, http.StatusNotFound, fmt.Sprintf("ไม่พบคำสั่งซื้อรหัส %d", id))
        return
    }

    w.WriteHeader(http.StatusNoContent)
}
```

### helper functions (เก็บไว้ใน `handler/response.go`)

```go
package handler

import (
    "encoding/json"
    "net/http"

    "github.com/jackc/pgx/v5/pgconn"
)

// alias เพื่อความสะดวกในการอ้างอิงในไฟล์อื่น
type pgxPgError = pgconn.PgError

func writeJSON(w http.ResponseWriter, status int, data interface{}) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    json.NewEncoder(w).Encode(data)
}

func writeError(w http.ResponseWriter, status int, message string) {
    writeJSON(w, status, map[string]string{"error": message})
}
```

### `main.go` — ประกอบร่างทั้งหมด

```go
package main

import (
    "context"
    "log"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"

    "ecommerce-api/db"
    "ecommerce-api/handler"
)

func main() {
    ctx := context.Background()

    connString := os.Getenv("DATABASE_URL")
    if connString == "" {
        log.Fatal("ต้องกำหนด environment variable DATABASE_URL")
    }

    pool, err := db.NewPool(ctx, connString)
    if err != nil {
        log.Fatalf("เชื่อมต่อฐานข้อมูลล้มเหลว: %v", err)
    }
    defer pool.Close()

    productHandler := handler.NewProductHandler(pool)
    orderHandler := handler.NewOrderHandler(pool)

    mux := http.NewServeMux()

    // Go 1.22+ รองรับ path parameter ใน ServeMux ได้โดยตรง (เช่น {id})
    mux.HandleFunc("GET /products", productHandler.List)
    mux.HandleFunc("GET /products/{id}", productHandler.Get)
    mux.HandleFunc("POST /products", productHandler.Create)
    mux.HandleFunc("PATCH /products/{id}/stock", productHandler.UpdateStock)
    mux.HandleFunc("DELETE /products/{id}", productHandler.Delete)

    mux.HandleFunc("POST /orders", orderHandler.Create)
    mux.HandleFunc("GET /orders/{id}", orderHandler.Get)
    mux.HandleFunc("PATCH /orders/{id}/status", orderHandler.UpdateStatus)

    srv := &http.Server{
        Addr:         ":8080",
        Handler:      mux,
        ReadTimeout:  5 * time.Second,
        WriteTimeout: 10 * time.Second,
    }

    // รัน server ใน goroutine แยก เพื่อให้ main goroutine รอสัญญาณ shutdown ได้
    go func() {
        log.Println("HTTP server กำลังรันที่ :8080")
        if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            log.Fatalf("server ล้มเหลว: %v", err)
        }
    }()

    // Graceful shutdown: รอสัญญาณ SIGINT/SIGTERM แล้วปิด server อย่างสุภาพ
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit

    log.Println("กำลังปิด server...")
    shutdownCtx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()

    if err := srv.Shutdown(shutdownCtx); err != nil {
        log.Fatalf("shutdown ล้มเหลว: %v", err)
    }
    log.Println("ปิด server สำเร็จ")
}
```

### การทดสอบ API ด้วย curl

```bash
# ดูรายการสินค้าทั้งหมด
curl http://localhost:8080/products

# สร้างสินค้าใหม่
curl -X POST http://localhost:8080/products \
  -H "Content-Type: application/json" \
  -d '{"product_name": "แท่นชาร์จไร้สาย", "unit_price": 590.00, "stock_quantity": 100}'

# สร้างคำสั่งซื้อ
curl -X POST http://localhost:8080/orders \
  -H "Content-Type: application/json" \
  -d '{"customer_id": 1, "items": [{"product_id": 1, "quantity": 2}, {"product_id": 3, "quantity": 1}]}'

# อัปเดตสถานะคำสั่งซื้อ
curl -X PATCH http://localhost:8080/orders/1/status \
  -H "Content-Type: application/json" \
  -d '{"status": "shipped"}'
```

API ตัวอย่างนี้แสดงให้เห็นการนำความรู้จากทุก Step ในบทนี้มาประกอบกัน: การใช้ `pgxpool` เป็น connection pool ระดับแอปพลิเคชัน (Step 883), การ scan ด้วย `pgx.CollectRows` (Step 884), การใช้ transaction พร้อม `defer tx.Rollback` เพื่อรับประกันความถูกต้องของข้อมูลตอนสร้าง order (Step 885), และการจัดการ error แบบ Go idiomatic ที่แยกแยะ PostgreSQL error code เพื่อคืน HTTP status code ที่เหมาะสม (Step 884)

---

## สรุปท้ายบท

### ตารางเปรียบเทียบ pgx vs database/sql vs GORM

| ประเด็น | pgx (โดยตรง) | database/sql | GORM |
|---|---|---|---|
| ระดับ abstraction | ต่ำ (ใกล้ SQL) | ต่ำ-กลาง (generic interface) | สูง (ORM) |
| ประสิทธิภาพ | สูงสุด (binary protocol) | ปานกลาง | ปานกลาง-ต่ำ (overhead ของ reflection/query builder) |
| ความเร็วในการพัฒนา | ปานกลาง (ต้องเขียน SQL เอง) | ปานกลาง | สูง (ลด boilerplate มาก) |
| ฟีเจอร์เฉพาะ PostgreSQL | ครบถ้วนที่สุด | จำกัด | จำกัดกว่า pgx (ต้องพึ่ง raw SQL เสริม) |
| Connection pooling | `pgxpool` (ยืดหยุ่นสูง) | มีในตัว (`sql.DB`) | ผ่าน `*sql.DB` เบื้องหลัง |
| Type safety ตอน compile | สูง (ผ่าน struct scan) | ปานกลาง | สูง (struct-based) |
| การ debug SQL ที่ generate | ไม่มีปัญหา (เขียนเอง) | ไม่มีปัญหา (เขียนเอง) | ต้องเปิด logger เพื่อดู SQL ที่ GORM สร้าง |
| เหมาะกับโปรเจกต์ | Perf-critical, ใช้ PostgreSQL เท่านั้น | Database-agnostic, ใช้ร่วมกับ tool อื่น | Rapid prototyping, CRUD-heavy application |
| Learning curve | ปานกลาง | ต่ำ | ต่ำ-ปานกลาง (ต้องเข้าใจ convention ของ ORM) |
| Migration | ต้องใช้ tool แยก (เช่น golang-migrate) | ต้องใช้ tool แยก | มี `AutoMigrate` ในตัว (แต่จำกัดความสามารถ) |

### แนวทางเลือกใช้ในทางปฏิบัติ

- **ทีมที่ต้องการประสิทธิภาพสูงสุดและควบคุม SQL เต็มที่** → ใช้ `pgx` + `pgxpool` โดยตรง เหมาะกับระบบที่มี traffic สูง หรือ query ที่ซับซ้อนต้องการ tuning ละเอียด
- **ทีมที่ต้องการความเร็วในการพัฒนา และมี CRUD operation เป็นส่วนใหญ่** → ใช้ GORM ช่วยลดเวลาพัฒนา แต่ควรเตรียมพร้อมเขียน raw SQL (ผ่าน `db.Raw()`) สำหรับ query ที่ซับซ้อนที่ GORM ทำได้ไม่ดีนัก
- **ทีมที่ต้องพึ่งพา tooling ที่คาดหวัง `*sql.DB`** → ใช้ `pgx/v5/stdlib` เป็น adapter เพื่อได้ทั้งความเร็วของ pgx และ compatibility ของ `database/sql`
- **แนวทางผสมผสานที่นิยมในทีมระดับ production** → ใช้ GORM สำหรับ CRUD พื้นฐานส่วนใหญ่ของแอป และสลับไปใช้ raw SQL ผ่าน pgx (หรือ `db.Raw()` ของ GORM) เฉพาะจุดที่ query ซับซ้อนหรือ performance-critical

### สิ่งสำคัญที่ต้องจำ

1. **parameterized query เสมอ** ไม่ว่าจะใช้ driver หรือ ORM ใด ห้ามต่อ string SQL จาก user input โดยตรงเด็ดขาด
2. **ปิด `rows` เสมอ** ด้วย `defer rows.Close()` เมื่อใช้ pgx โดยตรง (ยกเว้นเมื่อใช้ `pgx.CollectRows` ที่จัดการให้อัตโนมัติ)
3. **`defer tx.Rollback(ctx)`** ทันทีหลัง `Begin()` สำเร็จ เป็น pattern มาตรฐานที่ปลอดภัยเสมอ
4. **ระวังปัญหา N+1 query** เมื่อใช้ GORM โดยใช้ `Preload` หรือ `Joins` แทนการวน loop query ทีละแถว
5. **ตั้งค่า connection pool ให้เหมาะสม** กับจำนวน instance ของแอปพลิเคชันและ `max_connections` ของ PostgreSQL server
6. **ใช้ migration tool ที่เหมาะสมกับ production** แทนการพึ่ง `AutoMigrate` เพียงอย่างเดียวในระบบจริง

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1

จงอธิบายความแตกต่างระหว่าง `pgx.Connect` กับ `pgxpool.New` และบอกว่าแต่ละแบบเหมาะกับสถานการณ์ใด

<details>
<summary>เฉลย</summary>

`pgx.Connect` สร้าง connection เดี่ยว (single connection) ที่เชื่อมต่อกับฐานข้อมูลโดยตรง เหมาะสำหรับสคริปต์เล็ก ๆ, CLI tool, หรือโปรแกรมที่ทำงานแบบ sequential ไม่มีการใช้งานพร้อมกันหลาย goroutine ข้อจำกัดคือ connection เดียวไม่สามารถรับ query จากหลาย goroutine พร้อมกันได้อย่างปลอดภัย

`pgxpool.New` สร้าง connection pool ที่จัดการหลาย connection พร้อมกัน เหมาะสำหรับเว็บแอปพลิเคชันหรือ service ที่ต้องรองรับหลาย HTTP request พร้อมกัน (concurrent) เพราะแต่ละ request สามารถ acquire connection จาก pool มาใช้ได้โดยไม่ต้องรอ connection อื่นว่าง (ตราบใดที่ยังไม่เกิน `MaxConns`) และ pool ยังช่วยลด overhead ของการสร้าง connection ใหม่ทุกครั้งด้วย

</details>

### แบบฝึกหัดที่ 2

โค้ดต่อไปนี้มีปัญหาอะไร และควรแก้ไขอย่างไร

```go
func listAllOrders(ctx context.Context, pool *pgxpool.Pool) ([]Order, error) {
    rows, err := pool.Query(ctx, "SELECT order_id, customer_id, status FROM orders")
    if err != nil {
        return nil, err
    }

    var orders []Order
    for rows.Next() {
        var o Order
        rows.Scan(&o.OrderID, &o.CustomerID, &o.Status)
        orders = append(orders, o)
    }
    return orders, nil
}
```

<details>
<summary>เฉลย</summary>

โค้ดนี้มีปัญหาหลายจุด:

1. **ไม่มี `defer rows.Close()`** — แม้ pgx จะปิด rows ให้อัตโนมัติเมื่อ `rows.Next()` คืน false (อ่านจบทุกแถวหรือ error) แต่ถ้าฟังก์ชัน return ก่อนอ่านจบ (เช่นกรณี error กลางทาง) จะเกิด connection leak การใส่ `defer rows.Close()` ทันทีหลัง query สำเร็จเป็น best practice ที่ปลอดภัยเสมอ
2. **ไม่ได้ตรวจสอบ error จาก `rows.Scan()`** — โค้ดเพิกเฉยต่อ error ที่อาจเกิดขึ้นระหว่าง scan แต่ละแถว ถ้า scan ล้มเหลว (เช่น type ไม่ตรงกัน) โปรแกรมจะไม่รู้ตัวและอาจได้ข้อมูลที่ไม่สมบูรณ์
3. **ไม่ได้เช็ค `rows.Err()` หลังจบ loop** — `rows.Next()` คืน `false` ได้ทั้งจากอ่านจบปกติและจาก error ระหว่างทาง (เช่น connection หลุดกลางคัน) การไม่เช็ค `rows.Err()` ทำให้พลาดการตรวจจับ error เหล่านี้

โค้ดที่แก้ไขแล้ว:

```go
func listAllOrders(ctx context.Context, pool *pgxpool.Pool) ([]Order, error) {
    rows, err := pool.Query(ctx, "SELECT order_id, customer_id, status FROM orders")
    if err != nil {
        return nil, fmt.Errorf("listAllOrders query: %w", err)
    }
    defer rows.Close()

    var orders []Order
    for rows.Next() {
        var o Order
        if err := rows.Scan(&o.OrderID, &o.CustomerID, &o.Status); err != nil {
            return nil, fmt.Errorf("scan แถวล้มเหลว: %w", err)
        }
        orders = append(orders, o)
    }
    if err := rows.Err(); err != nil {
        return nil, fmt.Errorf("เกิดข้อผิดพลาดระหว่างอ่านแถว: %w", err)
    }
    return orders, nil
}
```

หรือใช้ `pgx.CollectRows` เพื่อลดโค้ดและความเสี่ยงเหล่านี้ไปเลย:

```go
func listAllOrders(ctx context.Context, pool *pgxpool.Pool) ([]Order, error) {
    rows, err := pool.Query(ctx, "SELECT order_id, customer_id, status FROM orders")
    if err != nil {
        return nil, fmt.Errorf("listAllOrders query: %w", err)
    }
    return pgx.CollectRows(rows, pgx.RowToStructByName[Order])
}
```

</details>

### แบบฝึกหัดที่ 3

จงเขียนฟังก์ชัน `CancelOrder` ที่ยกเลิกคำสั่งซื้อและคืนสต็อกสินค้ากลับเข้าคลัง โดยต้องทำภายใน transaction เดียว (สมมติว่ามีตาราง `order_items(order_id, product_id, quantity)` เพิ่มเติมจาก schema เดิม)

<details>
<summary>เฉลย</summary>

```go
func CancelOrder(ctx context.Context, pool *pgxpool.Pool, orderID int) error {
    tx, err := pool.Begin(ctx)
    if err != nil {
        return fmt.Errorf("begin transaction ล้มเหลว: %w", err)
    }
    defer tx.Rollback(ctx)

    // 1. ตรวจสอบสถานะปัจจุบันของ order ก่อน (ป้องกันการยกเลิกซ้ำ)
    var currentStatus string
    err = tx.QueryRow(ctx,
        "SELECT status FROM orders WHERE order_id = $1 FOR UPDATE",
        orderID,
    ).Scan(&currentStatus)
    if err != nil {
        if errors.Is(err, pgx.ErrNoRows) {
            return fmt.Errorf("ไม่พบคำสั่งซื้อรหัส %d", orderID)
        }
        return fmt.Errorf("ตรวจสอบสถานะ order ล้มเหลว: %w", err)
    }
    if currentStatus == "cancelled" {
        return fmt.Errorf("คำสั่งซื้อรหัส %d ถูกยกเลิกไปแล้ว", orderID)
    }
    if currentStatus == "completed" {
        return fmt.Errorf("ไม่สามารถยกเลิกคำสั่งซื้อที่เสร็จสมบูรณ์แล้ว")
    }

    // 2. คืนสต็อกสินค้าทั้งหมดในคำสั่งซื้อนี้กลับเข้าคลัง
    rows, err := tx.Query(ctx,
        "SELECT product_id, quantity FROM order_items WHERE order_id = $1",
        orderID,
    )
    if err != nil {
        return fmt.Errorf("ดึงรายการสินค้าล้มเหลว: %w", err)
    }

    type item struct {
        ProductID int
        Quantity  int
    }
    items, err := pgx.CollectRows(rows, pgx.RowToStructByPos[item])
    if err != nil {
        return fmt.Errorf("collect order items ล้มเหลว: %w", err)
    }

    for _, it := range items {
        _, err := tx.Exec(ctx,
            "UPDATE products SET stock_quantity = stock_quantity + $1 WHERE product_id = $2",
            it.Quantity, it.ProductID,
        )
        if err != nil {
            return fmt.Errorf("คืนสต็อกสินค้ารหัส %d ล้มเหลว: %w", it.ProductID, err)
        }
    }

    // 3. เปลี่ยนสถานะ order เป็น cancelled
    _, err = tx.Exec(ctx,
        "UPDATE orders SET status = 'cancelled' WHERE order_id = $1",
        orderID,
    )
    if err != nil {
        return fmt.Errorf("อัปเดตสถานะ order ล้มเหลว: %w", err)
    }

    if err := tx.Commit(ctx); err != nil {
        return fmt.Errorf("commit ล้มเหลว: %w", err)
    }
    return nil
}
```

จุดสำคัญคือการใช้ `FOR UPDATE` เพื่อ lock แถวของ order ป้องกันไม่ให้มีการยกเลิกคำสั่งซื้อเดียวกันพร้อมกันจากสอง request (race condition) และการทำทุกอย่างภายใน transaction เดียวเพื่อรับประกันว่าถ้าขั้นตอนใดล้มเหลว ทุกอย่างจะถูก rollback กลับ

</details>

### แบบฝึกหัดที่ 4

เมื่อใช้ `pgx.RowToStructByName` แล้วเกิด error ว่า "cannot find field" ควรตรวจสอบอะไรบ้าง

<details>
<summary>เฉลย</summary>

ควรตรวจสอบตามลำดับ:

1. **struct tag `db:"..."` ตรงกับชื่อคอลัมน์ที่ SELECT หรือไม่** — `pgx.RowToStructByName` match ชื่อคอลัมน์กับ struct tag `db` แบบตรงตัว (case-sensitive ตามที่ PostgreSQL คืนมา ซึ่งปกติเป็นตัวพิมพ์เล็ก) ถ้าคอลัมน์ชื่อ `product_name` แต่ struct tag เขียนว่า `db:"productName"` จะ match ไม่ได้
2. **มี alias ใน SELECT ที่ไม่ตรงกับ struct tag หรือไม่** — เช่น `SELECT first_name AS customer_name` แต่ struct มี tag `db:"first_name"` ต้องแก้ไข alias หรือ tag ให้ตรงกัน
3. **struct มี field ที่ไม่ได้อยู่ใน SELECT** — ถ้าต้องการอนุญาตให้มี field เกินได้ ให้ใช้ `pgx.RowToStructByNameLax` แทน `RowToStructByName` (ตัวปกติจะ error ทันทีถ้า mapping ไม่ครบ)
4. **field เป็น unexported (ตัวพิมพ์เล็กนำหน้า)** — pgx (เหมือน package อื่น ๆ ที่ใช้ reflection) เข้าถึงได้เฉพาะ exported field (ขึ้นต้นด้วยตัวพิมพ์ใหญ่) เท่านั้น

</details>

### แบบฝึกหัดที่ 5

จงอธิบายว่าทำไม `defer tx.Rollback(ctx)` ที่วางไว้หลัง `tx.Commit(ctx)` สำเร็จแล้ว ถึงไม่ทำให้เกิดปัญหา

<details>
<summary>เฉลย</summary>

เมื่อ `tx.Commit(ctx)` ถูกเรียกและสำเร็จ transaction จะถูกปิดไปแล้วในระดับ PostgreSQL connection เมื่อ `defer tx.Rollback(ctx)` ถูกเรียกตามมา pgx จะตรวจพบว่า transaction ปิดไปแล้ว (สถานะภายในของ `pgx.Tx` จะถูก mark ว่า closed) และคืน error พิเศษคือ `pgx.ErrTxClosed` กลับมาเงียบ ๆ โดยไม่มีผลกระทบใด ๆ ต่อข้อมูลที่ commit ไปแล้ว เพราะไม่มี operation ระดับฐานข้อมูลเกิดขึ้นจริงอีก

นี่คือเหตุผลที่ pattern `defer tx.Rollback(ctx)` ทันทีหลัง `Begin()` เป็นแนวทางที่ปลอดภัยและเป็นที่นิยมใน pgx — โค้ดไม่จำเป็นต้อง handle error จาก `Rollback` ใน defer เพราะในกรณีปกติ (commit สำเร็จ) มันจะไม่ทำอะไรเลย และในกรณีที่ error เกิดขึ้นก่อนถึง commit มันจะทำหน้าที่ rollback จริง ๆ ให้อัตโนมัติ

</details>

### แบบฝึกหัดที่ 6

จงเขียน GORM model และฟังก์ชันสำหรับดึงคำสั่งซื้อทั้งหมดของลูกค้าคนหนึ่ง พร้อมกับข้อมูลลูกค้า โดยหลีกเลี่ยงปัญหา N+1 query

<details>
<summary>เฉลย</summary>

```go
type Customer struct {
    CustomerID uint    `gorm:"column:customer_id;primaryKey"`
    FirstName  string  `gorm:"column:first_name;size:60"`
    Email      string  `gorm:"column:email;size:150"`
    Orders     []Order `gorm:"foreignKey:CustomerID"`
}

func (Customer) TableName() string { return "customers" }

type Order struct {
    OrderID    uint      `gorm:"column:order_id;primaryKey"`
    CustomerID uint      `gorm:"column:customer_id"`
    OrderDate  time.Time `gorm:"column:order_date"`
    Status     string    `gorm:"column:status;size:20"`
}

func (Order) TableName() string { return "orders" }

func GetCustomerOrders(db *gorm.DB, customerID uint) (*Customer, error) {
    var customer Customer
    // Preload("Orders") ทำให้ GORM ดึงข้อมูล orders ทั้งหมดของ customer นี้ด้วย 1 query เพิ่มเติม
    // แทนที่จะวน loop query ทีละ order (ซึ่งจะเป็นปัญหา N+1)
    result := db.Preload("Orders").First(&customer, customerID)
    if result.Error != nil {
        if errors.Is(result.Error, gorm.ErrRecordNotFound) {
            return nil, fmt.Errorf("ไม่พบลูกค้ารหัส %d", customerID)
        }
        return nil, result.Error
    }
    return &customer, nil
}
```

การใช้ `Preload("Orders")` ทำให้ GORM ยิง query เพียง 2 ครั้ง: ครั้งแรกดึงข้อมูล customer และครั้งที่สองดึง orders ทั้งหมดที่ `customer_id` ตรงกัน แทนที่จะยิง query แยกสำหรับ order แต่ละรายการ

</details>

### แบบฝึกหัดที่ 7

โค้ดต่อไปนี้ตั้งค่า pool ผิดพลาดอย่างไร และจะเกิดปัญหาอะไรถ้านำไปใช้จริงในระบบที่มีการ deploy 10 instance

```go
config.MaxConns = 100
pool, _ := pgxpool.NewWithConfig(ctx, config)
```

<details>
<summary>เฉลย</summary>

ปัญหาคือการตั้ง `MaxConns = 100` โดยไม่คำนึงถึงจำนวน instance ของแอปพลิเคชันที่จะ deploy พร้อมกัน ถ้าระบบมี 10 instance และแต่ละ instance ตั้ง `MaxConns = 100` ในสถานการณ์ที่ทุก instance ใช้ connection เต็ม pool พร้อมกัน จะมีการพยายามเปิด connection ไปยัง PostgreSQL รวมทั้งหมดถึง 1,000 connections ซึ่งเกินกว่าค่า default ของ `max_connections` ใน PostgreSQL (โดยทั่วไปตั้งไว้ที่ 100) มาก

ผลกระทบที่จะเกิดขึ้น:
- Instance ที่เชื่อมต่อไม่ทันจะได้รับ error `FATAL: too many connections`
- PostgreSQL server อาจใช้ RAM สูงเกินความจำเป็น เพราะแต่ละ connection ใช้ RAM หลาย MB
- Performance โดยรวมของระบบตกลง เพราะ PostgreSQL ต้องจัดการ context switch ระหว่าง connection จำนวนมากเกินความจำเป็น

วิธีแก้ไข:
1. คำนวณ `MaxConns` ให้เหมาะสมกับจำนวน instance ที่จะ deploy เช่น ถ้า PostgreSQL รองรับได้ 100 connections และมี 10 instance ควรตั้ง `MaxConns` ไว้ที่ประมาณ 8-10 ต่อ instance (เผื่อพื้นที่ให้ service อื่นและ admin connection ด้วย)
2. พิจารณาใช้ PgBouncer เป็นตัวกลางทำ connection pooling อีกชั้น เพื่อให้แอปทุก instance เชื่อมต่อผ่าน PgBouncer แทนที่จะเชื่อมตรงไปยัง PostgreSQL แต่ละตัว ซึ่งช่วยจำกัดจำนวน connection จริงที่ไปถึง PostgreSQL ได้อย่างมีประสิทธิภาพกว่า

</details>

### แบบฝึกหัดที่ 8

จงเขียนฟังก์ชันที่ใช้ `errors.As` เพื่อตรวจสอบว่า error ที่เกิดจากการ insert ลูกค้าใหม่เป็น unique_violation (email ซ้ำ) หรือไม่ และคืน HTTP status code ที่เหมาะสม

<details>
<summary>เฉลย</summary>

```go
func CreateCustomerHandler(pool *pgxpool.Pool) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        ctx := r.Context()

        var input struct {
            FirstName string `json:"first_name"`
            Email     string `json:"email"`
        }
        if err := json.NewDecoder(r.Body).Decode(&input); err != nil {
            writeError(w, http.StatusBadRequest, "รูปแบบ JSON ไม่ถูกต้อง")
            return
        }

        var customerID int
        err := pool.QueryRow(ctx,
            "INSERT INTO customers (first_name, email) VALUES ($1, $2) RETURNING customer_id",
            input.FirstName, input.Email,
        ).Scan(&customerID)

        if err != nil {
            var pgErr *pgconn.PgError
            if errors.As(err, &pgErr) && pgErr.Code == "23505" {
                // unique_violation — email ซ้ำ ควรคืน 409 Conflict ไม่ใช่ 500
                writeError(w, http.StatusConflict,
                    fmt.Sprintf("อีเมล %s ถูกใช้งานแล้วในระบบ", input.Email))
                return
            }
            writeError(w, http.StatusInternalServerError, fmt.Sprintf("สร้างลูกค้าล้มเหลว: %v", err))
            return
        }

        writeJSON(w, http.StatusCreated, map[string]int{"customer_id": customerID})
    }
}
```

หลักการสำคัญคือแยกแยะระหว่าง error ที่เกิดจาก "ผู้ใช้ทำผิดกติกา" (เช่น ข้อมูลซ้ำ ควรคืน `409 Conflict` หรือ `400 Bad Request`) กับ error ที่เกิดจาก "ปัญหาของระบบ" (เช่น connection หลุด ควรคืน `500 Internal Server Error`) การใช้ `errors.As` กับ `*pgconn.PgError` และตรวจ SQLSTATE code ช่วยให้แยกแยะได้แม่นยำ

</details>

### แบบฝึกหัดที่ 9

ข้อใดคือความแตกต่างสำคัญระหว่าง `db.Updates(structValue)` กับ `db.Updates(mapValue)` ใน GORM

<details>
<summary>เฉลย</summary>

เมื่อใช้ `db.Updates()` กับ **struct** GORM จะอัปเดตเฉพาะ field ที่มีค่า**ไม่ใช่ zero value** เท่านั้น (zero value ของแต่ละ type เช่น `0` สำหรับ int, `""` สำหรับ string, `false` สำหรับ bool) หมายความว่าถ้าต้องการอัปเดต `stock_quantity` เป็น `0` โดยใช้ struct จะไม่เกิดผลใด ๆ เพราะ GORM มองว่า `0` คือค่าที่ "ไม่ได้ตั้งใจจะอัปเดต"

```go
db.Model(&Product{}).Where("product_id = ?", 1).Updates(Product{StockQuantity: 0})
// ไม่มีผล! เพราะ 0 เป็น zero value ของ int
```

เมื่อใช้ `db.Updates()` กับ **`map[string]interface{}`** GORM จะอัปเดตทุก key-value ที่ระบุมาตรง ๆ โดยไม่สนใจว่าเป็น zero value หรือไม่ ทำให้สามารถตั้งค่าเป็น `0`, `""`, หรือ `false` ได้อย่างถูกต้อง

```go
db.Model(&Product{}).Where("product_id = ?", 1).Updates(map[string]interface{}{
    "stock_quantity": 0,
})
// ทำงานถูกต้อง — stock_quantity ถูกตั้งเป็น 0 จริง
```

ดังนั้นเมื่อต้องการอัปเดตฟิลด์ที่อาจมีค่าเป็น zero value ได้ตามธรรมชาติของ business logic (เช่น สต็อกเหลือ 0, ราคาลดเหลือ 0 ชั่วคราว) ควรใช้ `map[string]interface{}` แทน struct เสมอ

</details>

### แบบฝึกหัดที่ 10

จงเขียนฟังก์ชัน HTTP handler ด้วย `net/http` และ `pgx` ที่คืนรายการสินค้าที่มี `stock_quantity` ต่ำกว่าค่าที่กำหนดผ่าน query parameter `?threshold=10` พร้อมจัดการ error กรณี parameter ไม่ถูกต้อง

<details>
<summary>เฉลย</summary>

```go
func LowStockHandler(pool *pgxpool.Pool) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        ctx := r.Context()

        thresholdStr := r.URL.Query().Get("threshold")
        if thresholdStr == "" {
            thresholdStr = "10" // ค่า default ถ้าไม่ระบุ
        }

        threshold, err := strconv.Atoi(thresholdStr)
        if err != nil {
            writeError(w, http.StatusBadRequest, "threshold ต้องเป็นตัวเลขจำนวนเต็ม")
            return
        }
        if threshold < 0 {
            writeError(w, http.StatusBadRequest, "threshold ต้องไม่ติดลบ")
            return
        }

        rows, err := pool.Query(ctx,
            `SELECT product_id, product_name, unit_price, stock_quantity
             FROM products
             WHERE stock_quantity < $1
             ORDER BY stock_quantity ASC`,
            threshold,
        )
        if err != nil {
            writeError(w, http.StatusInternalServerError, fmt.Sprintf("query ล้มเหลว: %v", err))
            return
        }

        products, err := pgx.CollectRows(rows, pgx.RowToStructByName[model.Product])
        if err != nil {
            writeError(w, http.StatusInternalServerError, fmt.Sprintf("collect rows ล้มเหลว: %v", err))
            return
        }

        writeJSON(w, http.StatusOK, map[string]interface{}{
            "threshold": threshold,
            "count":     len(products),
            "products":  products,
        })
    }
}
```

จุดสำคัญของเฉลยนี้คือ: การตรวจสอบและแปลง query parameter อย่างปลอดภัยด้วย `strconv.Atoi` พร้อม error handling, การตั้งค่า default เมื่อไม่มี parameter ส่งมา, การใช้ parameterized query (`$1`) แทนการต่อ string, และการใช้ `pgx.CollectRows` เพื่อลด boilerplate ในการ scan ผลลัพธ์

</details>

---

## บทถัดไป

ในบทถัดไป เราจะย้ายไปดูการเชื่อมต่อ PostgreSQL จากภาษา Java ผ่าน JDBC driver, HikariCP connection pool, และ JPA/Hibernate ORM ซึ่งเป็นระบบนิเวศที่ใช้กันอย่างแพร่หลายในองค์กรขนาดใหญ่

อ่านต่อได้ที่ [Part 090: เชื่อมต่อ PostgreSQL กับ Java (JDBC, Hibernate)](./part-090-java-integration.md)
