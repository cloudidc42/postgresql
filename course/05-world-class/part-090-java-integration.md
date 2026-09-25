# Part 090: เชื่อมต่อ PostgreSQL กับ Java (JDBC, Hibernate/JPA)

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับโลก/ผู้เชี่ยวชาญ | Part 090

---

## เป้าหมายการเรียนรู้

เมื่อจบบทนี้ ผู้เรียนจะสามารถ:

1. เข้าใจภาพรวมของการเชื่อมต่อ PostgreSQL จากภาษา Java ทั้งแบบ JDBC ดิบ (raw JDBC) และผ่าน ORM อย่าง Hibernate/JPA
2. ใช้ `DriverManager`, `Connection`, `Statement`, `ResultSet` เพื่อรัน query พื้นฐานได้อย่างถูกต้อง
3. เขียน `PreparedStatement` แบบ parameterized query เพื่อป้องกัน SQL Injection ได้อย่างมืออาชีพ
4. ตั้งค่าและใช้งาน Connection Pooling ด้วย HikariCP ซึ่งเป็นมาตรฐานอุตสาหกรรมในปัจจุบัน
5. จัดการ Transaction ด้วย JDBC โดยใช้ `try-with-resources` ให้ปลอดภัยจาก resource leak
6. เข้าใจแนวคิดของ JPA/Hibernate การทำ Entity Mapping และการใช้ `EntityManager`
7. เขียน CRUD operations และ JPQL query ผ่าน JPA
8. เข้าใจ Relationship mapping (`@ManyToOne`, `@OneToMany`) และปัญหา N+1 query พร้อมวิธีแก้
9. ใช้งาน Spring Data JPA เพื่อลดโค้ด boilerplate ด้วย Repository interface และ query method naming convention
10. ประยุกต์ความรู้ทั้งหมดสร้าง REST API เล็ก ๆ สำหรับจัดการสินค้าและคำสั่งซื้อในระบบ e-commerce

---

## เตรียมข้อมูล

ตลอดบทนี้เราจะใช้ schema ของระบบ e-commerce ชุดเดียวกับบทก่อนหน้า เพื่อให้ตัวอย่างโค้ด Java ทั้งหมดเชื่อมโยงกับสถานการณ์จริงที่คุ้นเคย

```sql
-- ตารางสินค้า
CREATE TABLE products (
    product_id      SERIAL PRIMARY KEY,
    product_name    VARCHAR(150),
    unit_price      NUMERIC(10,2),
    stock_quantity  INTEGER
);

-- ตารางลูกค้า
CREATE TABLE customers (
    customer_id  SERIAL PRIMARY KEY,
    first_name   VARCHAR(60),
    email        VARCHAR(150)
);

-- ตารางคำสั่งซื้อ
CREATE TABLE orders (
    order_id     SERIAL PRIMARY KEY,
    customer_id  INTEGER REFERENCES customers(customer_id),
    order_date   TIMESTAMPTZ DEFAULT now(),
    status       VARCHAR(20)
);

-- ข้อมูลตัวอย่าง
INSERT INTO products (product_name, unit_price, stock_quantity) VALUES
    ('Mechanical Keyboard', 2590.00, 120),
    ('Wireless Mouse', 690.00, 300),
    ('4K Monitor 27"', 8900.00, 45),
    ('USB-C Hub', 990.00, 200),
    ('Laptop Stand', 550.00, 150);

INSERT INTO customers (first_name, email) VALUES
    ('สมชาย', 'somchai@example.com'),
    ('สุดา', 'suda@example.com'),
    ('วิชัย', 'wichai@example.com');

INSERT INTO orders (customer_id, status) VALUES
    (1, 'PENDING'),
    (2, 'SHIPPED'),
    (1, 'DELIVERED');
```

โปรเจกต์ Java ในบทนี้ใช้ **Maven** เป็นเครื่องมือจัดการ dependency (concept เดียวกันใช้ได้กับ Gradle) และใช้ **PostgreSQL JDBC Driver (pgJDBC)** เวอร์ชันล่าสุดในตระกูล 42.x เป็นตัวขับเคลื่อนการเชื่อมต่อทั้งหมด ไม่ว่าจะเรียกผ่าน JDBC ตรง ๆ หรือผ่าน Hibernate/JPA ก็ตาม เพราะ Hibernate เองก็ใช้ JDBC driver อยู่ข้างใต้เช่นกัน

โครงสร้าง dependency พื้นฐานใน `pom.xml` ที่จะใช้ตลอดบทนี้:

```xml
<dependencies>
    <!-- PostgreSQL JDBC Driver -->
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
        <version>42.7.3</version>
    </dependency>

    <!-- HikariCP Connection Pool -->
    <dependency>
        <groupId>com.zaxxer</groupId>
        <artifactId>HikariCP</artifactId>
        <version>5.1.0</version>
    </dependency>

    <!-- Hibernate ORM (JPA implementation) -->
    <dependency>
        <groupId>org.hibernate.orm</groupId>
        <artifactId>hibernate-core</artifactId>
        <version>6.5.2.Final</version>
    </dependency>

    <!-- Jakarta Persistence API -->
    <dependency>
        <groupId>jakarta.persistence</groupId>
        <artifactId>jakarta.persistence-api</artifactId>
        <version>3.1.0</version>
    </dependency>
</dependencies>
```

---

## Step 891: ภาพรวมการเชื่อมต่อ PostgreSQL จาก Java

### ทำไมต้องรู้หลายวิธี

การเชื่อมต่อ PostgreSQL จาก Java มีให้เลือกใช้หลายระดับ (layer) ซ้อนกันอยู่ ตั้งแต่ระดับต่ำสุดไปจนถึงระดับสูงสุด:

```
┌─────────────────────────────────────────────────────┐
│  Spring Data JPA (Repository interface)              │  ← ลด boilerplate มากที่สุด
├─────────────────────────────────────────────────────┤
│  JPA / Hibernate (EntityManager, @Entity)             │  ← ORM เต็มรูปแบบ
├─────────────────────────────────────────────────────┤
│  JDBC (Connection, Statement, ResultSet)              │  ← มาตรฐานพื้นฐานของ Java
├─────────────────────────────────────────────────────┤
│  PostgreSQL JDBC Driver (pgJDBC)                      │  ← driver ที่คุย protocol กับ PostgreSQL
├─────────────────────────────────────────────────────┤
│  PostgreSQL Wire Protocol / TCP Socket                │  ← ระดับเครือข่ายจริง
└─────────────────────────────────────────────────────┘
```

**JDBC (Java Database Connectivity)** คือ API มาตรฐานของ Java ที่กำหนดว่าโปรแกรม Java จะคุยกับฐานข้อมูลเชิงสัมพันธ์อย่างไร เป็น interface ที่อยู่ใน package `java.sql` และ `javax.sql` โดยตัว JDBC เองไม่รู้จัก PostgreSQL โดยตรง แต่ต้องอาศัย **driver** ที่ implement interface เหล่านั้นให้เข้าใจ wire protocol เฉพาะของแต่ละฐานข้อมูล — สำหรับ PostgreSQL คือไลบรารี `org.postgresql:postgresql` หรือที่เรียกกันว่า **pgJDBC**

จุดเด่นของ JDBC คือความเรียบง่ายและควบคุมได้เต็มที่ (full control) — เราเห็น SQL ทุกบรรทัดที่ถูกส่งไปจริง ๆ เหมาะกับงานที่ต้องการ performance สูงสุด หรือ query ที่ซับซ้อนมากจนการ generate SQL อัตโนมัติทำได้ไม่ดีพอ แต่ข้อเสียคือต้องเขียนโค้ด boilerplate เยอะ เช่น การแปลง `ResultSet` เป็น object เอง

**Hibernate / JPA (Jakarta Persistence API)** คือ ORM (Object-Relational Mapping) ที่ทำหน้าที่แปลง object ของ Java (class) ให้กลายเป็นแถวในตาราง และแปลงแถวในตารางกลับมาเป็น object โดยอัตโนมัติ ผู้พัฒนาไม่ต้องเขียน SQL เองในหลายกรณี เพียงแค่ประกาศ mapping ผ่าน annotation เช่น `@Entity`, `@Id`, `@Column` แล้ว Hibernate จะ generate SQL ให้เอง จุดเด่นคือลดโค้ด boilerplate ได้มหาศาล มี caching, dirty checking, lazy loading ในตัว แต่ก็มีค่าใช้จ่ายด้าน learning curve และถ้าใช้ไม่ถูกวิธี (เช่นปัญหา N+1 query) อาจทำให้ performance แย่กว่าการเขียน SQL เองเสียอีก

**Spring Data JPA** เป็นชั้นที่ซ้อนอยู่บน JPA/Hibernate อีกที ช่วยลด boilerplate ของ JPA ลงไปอีกขั้น โดยเราเพียงประกาศ interface ที่ extend `JpaRepository` แล้ว Spring จะสร้าง implementation ให้อัตโนมัติในขณะรัน (runtime proxy) รวมถึงสามารถ generate query จากชื่อ method ได้เลย เช่น `findByCustomerId(Integer id)`

### ตารางเปรียบเทียบเบื้องต้น

| คุณสมบัติ | JDBC | Hibernate/JPA | Spring Data JPA |
|---|---|---|---|
| ระดับ abstraction | ต่ำสุด (ใกล้ SQL) | กลาง (ORM) | สูงสุด (ลด boilerplate) |
| ควบคุม SQL ได้ละเอียด | สูงมาก | ปานกลาง (ผ่าน JPQL/native query) | ต่ำกว่า (แต่ยังปรับได้) |
| ความเร็วในการพัฒนา | ช้า | เร็ว | เร็วที่สุด |
| เหมาะกับ | งานเน้น performance, batch job, report | แอปพลิเคชันทั่วไปที่มี business object ซับซ้อน | REST API / microservice ที่ต้องการความเร็วในการพัฒนา |
| Learning curve | ต่ำ | สูง | ปานกลาง (ต้องเข้าใจ JPA ก่อน) |

เราจะไล่เรียนตั้งแต่ JDBC พื้นฐานไปจนถึง Spring Data JPA ในบทนี้ เพื่อให้เห็นภาพว่าแต่ละชั้นซ้อนทับกันอย่างไร และเมื่อไหร่ควรเลือกใช้แบบไหน

### แนวคิดสำคัญ: Driver คือสิ่งที่ทำให้ Java คุยกับ PostgreSQL ได้

pgJDBC ทำงานโดยการเปิด TCP connection ไปยัง PostgreSQL server (default port 5432) แล้วคุยกันด้วย PostgreSQL wire protocol เวอร์ชัน 3 (v3 protocol) ซึ่งรองรับทั้ง simple query protocol และ extended query protocol (ที่ใช้สำหรับ `PreparedStatement`) การเข้าใจว่ามี driver อยู่ตรงกลางนี้สำคัญมาก เพราะปัญหาหลายอย่างที่เจอในทางปฏิบัติ เช่น SSL handshake ล้มเหลว, timezone ไม่ตรงกัน, หรือ connection ค้าง มักมีต้นตอมาจากการตั้งค่า driver ไม่ถูกต้อง

URL การเชื่อมต่อพื้นฐานมีรูปแบบ:

```
jdbc:postgresql://<host>:<port>/<database>?param1=value1&param2=value2
```

ตัวอย่างเช่น:

```
jdbc:postgresql://localhost:5432/ecommerce?sslmode=prefer&ApplicationName=order-service
```

---

## Step 892: JDBC พื้นฐาน — DriverManager, Connection, Statement, ResultSet

### การเชื่อมต่อครั้งแรก

หัวใจของ JDBC คือ 3 interface นี้:

- **`Connection`** — ตัวแทนของการเชื่อมต่อหนึ่งเส้นไปยังฐานข้อมูล
- **`Statement`** — ตัวส่งคำสั่ง SQL ไปรันบนฐานข้อมูล
- **`ResultSet`** — ตัวแทนผลลัพธ์ที่ query กลับมา (เหมือน cursor ที่เลื่อนไปทีละแถว)

```java
package com.example.jdbc;

import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.ResultSet;
import java.sql.SQLException;
import java.sql.Statement;

public class BasicJdbcDemo {

    private static final String URL =
            "jdbc:postgresql://localhost:5432/ecommerce";
    private static final String USER = "app_user";
    private static final String PASSWORD = "secret_password";

    public static void main(String[] args) {
        // getConnection จะโหลด driver ผ่าน Java SPI (ServiceLoader) โดยอัตโนมัติ
        // ตั้งแต่ JDBC 4.0 เป็นต้นมา ไม่ต้องเรียก Class.forName() เองแล้ว
        try (Connection conn = DriverManager.getConnection(URL, USER, PASSWORD);
             Statement stmt = conn.createStatement()) {

            System.out.println("เชื่อมต่อสำเร็จ: " + conn.getMetaData().getURL());

            ResultSet rs = stmt.executeQuery(
                    "SELECT product_id, product_name, unit_price, stock_quantity " +
                    "FROM products ORDER BY product_id"
            );

            while (rs.next()) {
                int id = rs.getInt("product_id");
                String name = rs.getString("product_name");
                java.math.BigDecimal price = rs.getBigDecimal("unit_price");
                int stock = rs.getInt("stock_quantity");

                System.out.printf("#%d %-25s ราคา %.2f บาท คงเหลือ %d ชิ้น%n",
                        id, name, price, stock);
            }

        } catch (SQLException e) {
            // SQLException มี getSQLState() ที่บอกรหัส error มาตรฐาน (เช่น "23505" = unique_violation)
            System.err.println("SQL error [" + e.getSQLState() + "]: " + e.getMessage());
        }
    }
}
```

### จุดที่ต้องระวัง

1. **try-with-resources เป็นสิ่งจำเป็น ไม่ใช่ทางเลือก** — `Connection`, `Statement`, `ResultSet` ล้วน implement `AutoCloseable` การไม่ปิดจะทำให้เกิด connection leak และ resource leak ฝั่ง PostgreSQL server (เช่น process ค้างเป็น `idle` connection สะสมจนเต็ม `max_connections`)

2. **ลำดับการปิด** — try-with-resources จะปิดจากล่างขึ้นบนโดยอัตโนมัติ (`ResultSet` → `Statement` → `Connection`) ซึ่งถูกต้องตามลำดับที่ควรจะเป็น

3. **`Statement` vs `PreparedStatement`** — `Statement` ธรรมดาไม่รองรับ parameter binding และเสี่ยงต่อ SQL Injection หากมีการต่อ string จาก user input โดยตรง (จะกล่าวถึงใน Step 893)

4. **ResultSet เป็น forward-only cursor โดย default** — เรียก `rs.next()` เพื่อเลื่อนไปทีละแถว จะเลื่อนถอยหลังไม่ได้ ถ้าต้องการ scroll ได้ทั้งสองทิศทางต้องสร้าง Statement ด้วย `conn.createStatement(ResultSet.TYPE_SCROLL_INSENSITIVE, ResultSet.CONCUR_READ_ONLY)`

5. **การ map ชนิดข้อมูล PostgreSQL → Java** เป็นเรื่องสำคัญที่ต้องรู้:

| PostgreSQL type | Java type (แนะนำ) | Getter method |
|---|---|---|
| `integer` / `serial` | `int` / `Integer` | `getInt()` |
| `bigint` / `bigserial` | `long` / `Long` | `getLong()` |
| `numeric(p,s)` | `java.math.BigDecimal` | `getBigDecimal()` |
| `varchar` / `text` | `String` | `getString()` |
| `boolean` | `boolean` / `Boolean` | `getBoolean()` |
| `timestamp` | `java.time.LocalDateTime` | `getObject(col, LocalDateTime.class)` |
| `timestamptz` | `java.time.OffsetDateTime` | `getObject(col, OffsetDateTime.class)` |
| `date` | `java.time.LocalDate` | `getObject(col, LocalDate.class)` |
| `jsonb` / `json` | `String` (แล้วแปลงด้วย Jackson/Gson) | `getString()` |
| `uuid` | `java.util.UUID` | `getObject(col, UUID.class)` |

> **ข้อควรระวังเรื่อง `timestamptz`**: ให้ใช้ `getObject(columnLabel, OffsetDateTime.class)` แทน `getTimestamp()` แบบเก่า เพราะ pgJDBC เวอร์ชันใหม่รองรับ `java.time` API เต็มรูปแบบ และหลีกเลี่ยงปัญหาเรื่อง timezone ที่สับสนจาก `java.util.Date`/`Calendar` แบบเดิม

### ตัวอย่าง INSERT/UPDATE/DELETE ด้วย Statement

```java
try (Connection conn = DriverManager.getConnection(URL, USER, PASSWORD);
     Statement stmt = conn.createStatement()) {

    int rowsAffected = stmt.executeUpdate(
            "UPDATE products SET stock_quantity = stock_quantity - 1 " +
            "WHERE product_id = 1"
    );
    System.out.println("อัปเดตไป " + rowsAffected + " แถว");

} catch (SQLException e) {
    e.printStackTrace();
}
```

`executeUpdate()` ใช้กับ `INSERT`, `UPDATE`, `DELETE`, `DDL` และคืนค่าจำนวนแถวที่ได้รับผลกระทบ ส่วน `executeQuery()` ใช้กับ `SELECT` เท่านั้นและคืนค่า `ResultSet` หากไม่แน่ใจว่าจะเป็น query แบบไหน สามารถใช้ `execute()` ซึ่งคืนค่า `boolean` บอกว่ามี `ResultSet` หรือไม่

---

## Step 893: PreparedStatement — parameterized query ป้องกัน SQL Injection

### ปัญหาของการต่อ string SQL

โค้ดแบบนี้ **ห้ามเขียนเด็ดขาด** ในโปรเจกต์จริง:

```java
// ❌ อันตรายมาก — เสี่ยง SQL Injection
String email = request.getParameter("email"); // สมมติ user กรอกมา
String sql = "SELECT * FROM customers WHERE email = '" + email + "'";
ResultSet rs = stmt.executeQuery(sql);
```

หาก user ป้อนค่า `email` เป็น `' OR '1'='1` ค่า SQL ที่ได้จะกลายเป็น `SELECT * FROM customers WHERE email = '' OR '1'='1'` ซึ่งจะคืนลูกค้าทั้งหมดในระบบ หรือแย่กว่านั้นคือใช้เทคนิค stacked query เพื่อลบข้อมูลทั้งตารางได้

### PreparedStatement คือทางแก้มาตรฐาน

`PreparedStatement` ใช้ **extended query protocol** ของ PostgreSQL ซึ่งแยก SQL text ออกจาก parameter values อย่างชัดเจนตั้งแต่ต้นทาง ทำให้ driver ส่ง SQL แม่แบบ (พร้อม placeholder `?`) ไปให้ PostgreSQL parse และ plan ก่อน แล้วค่อยส่ง parameter ไปผูกทีหลัง — PostgreSQL จึงไม่มีทางตีความ parameter value เป็นส่วนหนึ่งของโครงสร้าง SQL ได้เลย

```java
package com.example.jdbc;

import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.sql.SQLException;
import java.math.BigDecimal;

public class PreparedStatementDemo {

    private static final String URL = "jdbc:postgresql://localhost:5432/ecommerce";
    private static final String USER = "app_user";
    private static final String PASSWORD = "secret_password";

    // ค้นหาสินค้าที่ราคาไม่เกินที่กำหนด และมีสต็อกมากกว่า 0
    public void findAffordableProducts(BigDecimal maxPrice) throws SQLException {
        String sql = """
                SELECT product_id, product_name, unit_price, stock_quantity
                FROM products
                WHERE unit_price <= ? AND stock_quantity > 0
                ORDER BY unit_price
                """;

        try (Connection conn = DriverManager.getConnection(URL, USER, PASSWORD);
             PreparedStatement ps = conn.prepareStatement(sql)) {

            ps.setBigDecimal(1, maxPrice); // parameter index เริ่มที่ 1 ไม่ใช่ 0

            try (ResultSet rs = ps.executeQuery()) {
                while (rs.next()) {
                    System.out.printf("%s - %.2f บาท (คงเหลือ %d)%n",
                            rs.getString("product_name"),
                            rs.getBigDecimal("unit_price"),
                            rs.getInt("stock_quantity"));
                }
            }
        }
    }

    // สร้างคำสั่งซื้อใหม่ พร้อมรับค่า generated key (order_id) กลับมา
    public int createOrder(int customerId, String status) throws SQLException {
        String sql = "INSERT INTO orders (customer_id, status) VALUES (?, ?)";

        try (Connection conn = DriverManager.getConnection(URL, USER, PASSWORD);
             PreparedStatement ps = conn.prepareStatement(sql,
                     PreparedStatement.RETURN_GENERATED_KEYS)) {

            ps.setInt(1, customerId);
            ps.setString(2, status);

            int affected = ps.executeUpdate();
            if (affected == 0) {
                throw new SQLException("สร้างคำสั่งซื้อไม่สำเร็จ ไม่มีแถวถูกเพิ่ม");
            }

            try (ResultSet generatedKeys = ps.getGeneratedKeys()) {
                if (generatedKeys.next()) {
                    return generatedKeys.getInt(1);
                } else {
                    throw new SQLException("สร้างคำสั่งซื้อสำเร็จ แต่ไม่ได้ order_id กลับมา");
                }
            }
        }
    }
}
```

### Batch update ด้วย PreparedStatement

เมื่อต้อง insert/update จำนวนมาก การใช้ batch จะลด round-trip ระหว่าง Java กับ PostgreSQL ได้อย่างมาก:

```java
public void bulkAdjustStock(Map<Integer, Integer> productIdToDelta) throws SQLException {
    String sql = "UPDATE products SET stock_quantity = stock_quantity + ? WHERE product_id = ?";

    try (Connection conn = DriverManager.getConnection(URL, USER, PASSWORD);
         PreparedStatement ps = conn.prepareStatement(sql)) {

        conn.setAutoCommit(false); // ปิด autocommit เพื่อรวมทั้งหมดเป็น 1 transaction

        for (Map.Entry<Integer, Integer> entry : productIdToDelta.entrySet()) {
            ps.setInt(1, entry.getValue());   // delta
            ps.setInt(2, entry.getKey());     // product_id
            ps.addBatch();
        }

        int[] results = ps.executeBatch(); // ส่งเป็นชุดเดียว
        conn.commit();

        System.out.println("อัปเดตสำเร็จทั้งหมด " + results.length + " รายการ");
    }
}
```

> **เทคนิคสำหรับ throughput สูง**: เพิ่ม `reWriteBatchedInserts=true` ต่อท้าย JDBC URL (เช่น `jdbc:postgresql://localhost:5432/ecommerce?reWriteBatchedInserts=true`) เพื่อให้ pgJDBC รวมหลาย `INSERT` เป็นคำสั่งเดียวแบบ multi-values (`INSERT INTO t VALUES (...), (...), (...)`) แทนที่จะส่งทีละคำสั่งใน batch protocol ซึ่งในหลายกรณีเร็วขึ้นหลายเท่าตัว

### ข้อควรจำเรื่อง PreparedStatement caching

pgJDBC มีกลไก **server-side prepared statement** โดย default จะเริ่ม cache ที่ฝั่ง server หลังจากเรียก statement เดิม (SQL text เดียวกัน) ซ้ำเกิน `prepareThreshold` ครั้ง (ค่า default คือ 5) ซึ่งช่วยลดเวลา parse/plan ซ้ำ ๆ ได้มาก แต่ในบางกรณี (เช่น query ที่ query plan ควรเปลี่ยนไปตาม parameter value การ cache plan ไว้อาจทำให้ plan ไม่เหมาะกับข้อมูลจริง) สามารถปิดหรือปรับค่านี้ผ่าน connection property `prepareThreshold=0`

---

## Step 894: Connection Pooling ด้วย HikariCP

### ทำไมต้องมี Connection Pool

การเปิด-ปิด TCP connection ไปยัง PostgreSQL ทุกครั้งที่ต้องการ query เป็นการทำงานที่มีต้นทุนสูงมาก เพราะ PostgreSQL ใช้สถาปัตยกรรม **process-per-connection** — ทุกครั้งที่มี connection ใหม่ PostgreSQL ต้อง fork process ใหม่ขึ้นมาทั้งหมด ซึ่งใช้เวลาหลัก millisecond และใช้หน่วยความจำเพิ่มขึ้นทุกครั้ง

**Connection Pool** คือกลไกที่เปิด connection ไว้ล่วงหน้าจำนวนหนึ่ง แล้วให้แอปพลิเคชัน "ยืม" ไปใช้และ "คืน" กลับมาแทนที่จะปิดจริง ทำให้ลด overhead การสร้าง connection ใหม่ไปได้เกือบทั้งหมด

**HikariCP** เป็น connection pool ที่เร็วที่สุดและได้รับความนิยมสูงสุดใน ecosystem ของ Java ปัจจุบัน (เป็น default pool ของ Spring Boot ตั้งแต่ Spring Boot 2 เป็นต้นมา) จุดเด่นคือ code base เล็ก, benchmark ดีที่สุดในกลุ่ม, และมี pool ที่ฉลาดในการจัดการ connection state

### ตั้งค่า HikariConfig

```java
package com.example.pool;

import com.zaxxer.hikari.HikariConfig;
import com.zaxxer.hikari.HikariDataSource;

import java.sql.Connection;
import java.sql.SQLException;

public class DataSourceProvider {

    private static final HikariDataSource dataSource;

    static {
        HikariConfig config = new HikariConfig();

        config.setJdbcUrl("jdbc:postgresql://localhost:5432/ecommerce");
        config.setUsername("app_user");
        config.setPassword("secret_password");
        config.setDriverClassName("org.postgresql.Driver");

        // ---- ขนาด pool ----
        // สูตรทั่วไป (Brian Goetz / HikariCP wiki): connections = ((core_count * 2) + effective_spindle_count)
        // สำหรับ web app ทั่วไปเริ่มที่ 10 แล้วปรับตาม load test จริง
        config.setMaximumPoolSize(10);
        config.setMinimumIdle(2); // จำนวน connection ขั้นต่ำที่เปิดค้างไว้เสมอ

        // ---- timeout ต่าง ๆ ----
        config.setConnectionTimeout(30_000);  // รอ connection ว่างสูงสุด 30 วินาที ก่อน throw exception
        config.setIdleTimeout(600_000);       // ปิด connection ที่ idle เกิน 10 นาที (ถ้าเกิน minimumIdle)
        config.setMaxLifetime(1_800_000);     // อายุสูงสุดของ connection คือ 30 นาที (ควรน้อยกว่า timeout ฝั่ง DB/firewall)
        config.setKeepaliveTime(300_000);     // ส่ง keepalive ทุก 5 นาที เพื่อกัน connection ถูกตัดโดย network device

        // ---- ตรวจสอบความถูกต้องของ connection ----
        config.setConnectionTestQuery("SELECT 1"); // ใช้เฉพาะ driver เก่าที่ไม่รองรับ JDBC4 isValid()
        // pgJDBC สมัยใหม่รองรับ Connection.isValid() อยู่แล้ว จึงมักไม่ต้องตั้งค่านี้

        // ---- ชื่อ pool (ช่วยตอน debug / อ่าน log / JMX) ----
        config.setPoolName("ecommerce-hikari-pool");

        // ---- ส่ง connection property เพิ่มเติมให้ pgJDBC ----
        config.addDataSourceProperty("ApplicationName", "ecommerce-order-service");
        config.addDataSourceProperty("reWriteBatchedInserts", "true");

        dataSource = new HikariDataSource(config);
    }

    public static Connection getConnection() throws SQLException {
        return dataSource.getConnection(); // ยืม connection จาก pool
    }

    public static HikariDataSource getDataSource() {
        return dataSource;
    }

    public static void shutdown() {
        dataSource.close(); // เรียกตอนแอปพลิเคชัน shutdown เท่านั้น
    }
}
```

การใช้งานหลังจากนี้จะเหมือนเดิมทุกประการ เพียงแค่เปลี่ยนจาก `DriverManager.getConnection(...)` มาเป็น `DataSourceProvider.getConnection()`:

```java
public List<String> listProductNames() throws SQLException {
    List<String> names = new ArrayList<>();

    // Connection นี้จริง ๆ แล้วเป็น proxy ของ HikariCP — เรียก close() แล้ว
    // connection จะถูก "คืน" กลับ pool ไม่ได้ถูกปิดจริง ๆ
    try (Connection conn = DataSourceProvider.getConnection();
         PreparedStatement ps = conn.prepareStatement("SELECT product_name FROM products");
         ResultSet rs = ps.executeQuery()) {

        while (rs.next()) {
            names.add(rs.getString(1));
        }
    }
    return names;
}
```

### ตารางพารามิเตอร์สำคัญของ HikariCP

| พารามิเตอร์ | ความหมาย | ค่าแนะนำเริ่มต้น |
|---|---|---|
| `maximumPoolSize` | จำนวน connection สูงสุดใน pool | 10 (ปรับตาม CPU core และ load test) |
| `minimumIdle` | จำนวน connection ขั้นต่ำที่เปิดค้างไว้ | เท่ากับ `maximumPoolSize` (แนะนำโดย HikariCP wiki เพื่อความเสถียร) |
| `connectionTimeout` | เวลารอ connection ว่างก่อน throw `SQLTransientConnectionException` | 30000 ms |
| `maxLifetime` | อายุสูงสุดของแต่ละ connection ก่อนถูกปิดและสร้างใหม่ | 1800000 ms (30 นาที) ควรน้อยกว่า `idle_in_transaction_session_timeout` ของ PostgreSQL |
| `idleTimeout` | เวลาที่ connection ว่างเกินแล้วจะถูกปิด (มีผลเมื่อจำนวน idle > minimumIdle) | 600000 ms |
| `leakDetectionThreshold` | ตรวจจับกรณี connection ถูกยืมไปนานผิดปกติ (ลืม close) แล้ว log warning | 0 (ปิด) แนะนำเปิดใน dev/staging เป็น 60000 ms |

> **ข้อควรระวังสำคัญ**: `maxLifetime` ของ HikariCP ควรตั้งให้ **น้อยกว่า** ค่า network timeout ใด ๆ ที่อยู่ระหว่างแอปกับ PostgreSQL เช่น load balancer idle timeout, firewall connection timeout หรือ PostgreSQL's `tcp_keepalives` ไม่เช่นนั้นอาจเกิด "connection ที่ดูเหมือนใช้ได้แต่จริง ๆ ถูกตัดไปแล้ว" (stale connection) ซึ่งทำให้เกิด error แบบสุ่มในการ query

### เปรียบเทียบขนาด pool กับ PostgreSQL `max_connections`

ต้องคำนึงเสมอว่า PostgreSQL มีขีดจำกัด `max_connections` (default 100) ในระดับ server หากมีแอปพลิเคชันหลาย instance แต่ละ instance เปิด pool ขนาด 20 connection และรันพร้อมกัน 10 instance จะใช้ไป 200 connections ทันที ซึ่งเกิน limit ของ PostgreSQL ในกรณีนี้ควรพิจารณาใช้ **PgBouncer** เป็น connection pooler ระดับกลาง (transaction pooling mode) เพื่อให้ connection จำนวนมากจากฝั่งแอปพลิเคชันถูก multiplex ไปยัง connection จำนวนน้อยกว่าที่ PostgreSQL จริง ๆ

---

## Step 895: Transaction ด้วย JDBC

### หลักการพื้นฐาน

โดย default, JDBC `Connection` จะอยู่ในโหมด `autoCommit = true` ซึ่งหมายความว่าทุกคำสั่ง SQL จะถูก commit ทันทีที่รันเสร็จ เหมาะกับ query เดี่ยว ๆ แต่ไม่เหมาะกับกรณีที่ต้องรันหลายคำสั่งร่วมกันเป็นหน่วยเดียว (atomic unit) เช่น การตัดสต็อกสินค้าพร้อมกับสร้างคำสั่งซื้อ — ถ้าขั้นตอนใดขั้นตอนหนึ่งล้มเหลว ต้อง rollback ทั้งหมด ไม่ให้ข้อมูลครึ่ง ๆ กลาง ๆ ถูกบันทึกลงไป

### ตัวอย่าง: สร้างคำสั่งซื้อพร้อมตัดสต็อกแบบ atomic

```java
package com.example.jdbc;

import java.sql.Connection;
import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.sql.SQLException;

public class OrderTransactionService {

    /**
     * สร้างคำสั่งซื้อและตัดสต็อกสินค้าในคราวเดียว
     * ถ้าสต็อกไม่พอ หรือขั้นตอนใดล้มเหลว จะ rollback ทั้งหมด
     */
    public int placeOrder(int customerId, int productId, int quantity) throws SQLException {

        String checkStockSql =
                "SELECT stock_quantity FROM products WHERE product_id = ? FOR UPDATE";
        String deductStockSql =
                "UPDATE products SET stock_quantity = stock_quantity - ? WHERE product_id = ?";
        String insertOrderSql =
                "INSERT INTO orders (customer_id, status) VALUES (?, 'PENDING') RETURNING order_id";

        // try-with-resources จะปิด Connection ให้อัตโนมัติเมื่อออกจาก block
        // ไม่ว่าจะสำเร็จหรือเกิด exception ก็ตาม
        try (Connection conn = DataSourceProvider.getConnection()) {

            conn.setAutoCommit(false); // เริ่ม transaction ด้วยตนเอง

            try {
                // 1) ล็อกแถวสินค้าด้วย FOR UPDATE เพื่อป้องกัน race condition
                //    ระหว่างการอ่านค่า stock กับการตัดสต็อกจริง
                int currentStock;
                try (PreparedStatement ps = conn.prepareStatement(checkStockSql)) {
                    ps.setInt(1, productId);
                    try (ResultSet rs = ps.executeQuery()) {
                        if (!rs.next()) {
                            throw new SQLException("ไม่พบสินค้า product_id=" + productId);
                        }
                        currentStock = rs.getInt("stock_quantity");
                    }
                }

                if (currentStock < quantity) {
                    throw new IllegalStateException(
                            "สต็อกไม่เพียงพอ (คงเหลือ " + currentStock + ", ต้องการ " + quantity + ")");
                }

                // 2) ตัดสต็อก
                try (PreparedStatement ps = conn.prepareStatement(deductStockSql)) {
                    ps.setInt(1, quantity);
                    ps.setInt(2, productId);
                    ps.executeUpdate();
                }

                // 3) สร้างคำสั่งซื้อ
                int orderId;
                try (PreparedStatement ps = conn.prepareStatement(insertOrderSql)) {
                    ps.setInt(1, customerId);
                    try (ResultSet rs = ps.executeQuery()) {
                        rs.next();
                        orderId = rs.getInt("order_id");
                    }
                }

                conn.commit(); // ทุกอย่างสำเร็จ ยืนยันการเปลี่ยนแปลงทั้งหมด
                return orderId;

            } catch (Exception e) {
                conn.rollback(); // ล้มเหลวขั้นตอนใดขั้นตอนหนึ่ง ย้อนกลับทั้งหมด
                throw new SQLException("สร้างคำสั่งซื้อล้มเหลว: " + e.getMessage(), e);
            } finally {
                conn.setAutoCommit(true); // คืนค่า default ก่อนคืน connection กลับ pool
            }
        }
    }
}
```

### จุดสำคัญที่ต้องเข้าใจ

1. **`SELECT ... FOR UPDATE`** — ล็อกแถวที่อ่านมาเพื่อป้องกันไม่ให้ transaction อื่นแก้ไขแถวเดียวกันพร้อมกัน (row-level lock) จำเป็นมากในสถานการณ์ตัดสต็อกที่มีการเข้าถึงพร้อมกันสูง (high concurrency)

2. **คืนค่า `autoCommit` เป็น `true` ก่อนคืน connection กลับ pool** — เพราะ connection ที่ยืมจาก HikariCP จะถูกใช้ซ้ำโดยโค้ดส่วนอื่น หากลืมคืนค่า อาจทำให้โค้ดส่วนถัดไปที่ยืม connection นี้ไปใช้ทำงานผิดพลาดโดยไม่รู้ตัว (เข้าใจผิดว่าอยู่ใน autocommit mode) — HikariCP มีกลไก `resetAutoCommit` ช่วยอยู่แล้วในบางกรณี แต่การเขียนโค้ดให้ชัดเจนเองเป็นแนวทางปฏิบัติที่ปลอดภัยกว่า

3. **Isolation Level** — JDBC อนุญาตให้กำหนด isolation level ต่อ connection ได้ผ่าน `conn.setTransactionIsolation(...)`:

```java
conn.setTransactionIsolation(Connection.TRANSACTION_READ_COMMITTED); // default ของ PostgreSQL
conn.setTransactionIsolation(Connection.TRANSACTION_REPEATABLE_READ);
conn.setTransactionIsolation(Connection.TRANSACTION_SERIALIZABLE);
```

PostgreSQL ไม่รองรับ `TRANSACTION_READ_UNCOMMITTED` จริง (จะถูก treat เป็น `READ_COMMITTED` โดยอัตโนมัติ) ต่างจากฐานข้อมูลบางตัว

4. **Savepoint** — สำหรับ transaction ที่ซับซ้อน สามารถใช้ savepoint เพื่อ rollback บางส่วนได้โดยไม่ต้อง rollback ทั้ง transaction:

```java
Savepoint sp = conn.setSavepoint("before_risky_operation");
try {
    // ทำงานที่เสี่ยง
} catch (SQLException e) {
    conn.rollback(sp); // ย้อนกลับเฉพาะส่วนหลัง savepoint เท่านั้น
}
```

5. **try-with-resources ครอบ Connection ด้วยเสมอ** — แม้จะมี try-catch-finally ซ้อนอยู่ข้างในสำหรับจัดการ transaction แต่ตัว `Connection` เองต้องอยู่ใน try-with-resources ชั้นนอกสุดเสมอ เพื่อรับประกันว่าไม่ว่าจะเกิดอะไรขึ้น connection จะถูกคืนกลับ pool อย่างแน่นอน

---

## Step 896: JPA/Hibernate แนะนำตัว — Entity, EntityManager

### แนวคิดของ ORM

**Hibernate** คือ ORM framework ที่ได้รับความนิยมสูงสุดใน Java ecosystem และเป็น implementation หลักของสเปก **JPA (Jakarta Persistence API)** — JPA เป็นเพียง "สเปก" (interface/annotation ที่กำหนดมาตรฐาน) ส่วน Hibernate คือ "ผู้ implement" สเปกนั้นจริง ๆ (คล้ายความสัมพันธ์ระหว่าง JDBC กับ pgJDBC) ทำให้เราเขียนโค้ดโดยอ้างอิง JPA API เป็นหลัก และสามารถเปลี่ยน ORM implementation ไปเป็นตัวอื่น (เช่น EclipseLink) ได้โดยกระทบโค้ดน้อยที่สุด — แม้ในทางปฏิบัติ Hibernate จะเป็นตัวเลือกหลักเกือบทุกโปรเจกต์

### การประกาศ Entity

```java
package com.example.entity;

import jakarta.persistence.*;
import java.math.BigDecimal;

@Entity
@Table(name = "products")
public class Product {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY) // map กับ SERIAL ของ PostgreSQL
    @Column(name = "product_id")
    private Integer productId;

    @Column(name = "product_name", length = 150)
    private String productName;

    @Column(name = "unit_price", precision = 10, scale = 2)
    private BigDecimal unitPrice;

    @Column(name = "stock_quantity")
    private Integer stockQuantity;

    // JPA ต้องการ no-arg constructor เสมอ (Hibernate ใช้ reflection สร้าง object)
    protected Product() {
    }

    public Product(String productName, BigDecimal unitPrice, Integer stockQuantity) {
        this.productName = productName;
        this.unitPrice = unitPrice;
        this.stockQuantity = stockQuantity;
    }

    // ---- getters / setters ----
    public Integer getProductId() { return productId; }
    public String getProductName() { return productName; }
    public void setProductName(String productName) { this.productName = productName; }
    public BigDecimal getUnitPrice() { return unitPrice; }
    public void setUnitPrice(BigDecimal unitPrice) { this.unitPrice = unitPrice; }
    public Integer getStockQuantity() { return stockQuantity; }
    public void setStockQuantity(Integer stockQuantity) { this.stockQuantity = stockQuantity; }
}
```

**Annotation หลักที่ต้องรู้จัก:**

| Annotation | หน้าที่ |
|---|---|
| `@Entity` | บอก Hibernate ว่า class นี้ map กับตารางในฐานข้อมูล |
| `@Table(name = "...")` | ระบุชื่อตารางจริง (ถ้าไม่ระบุจะใช้ชื่อ class เป็น default) |
| `@Id` | ระบุ field ที่เป็น primary key |
| `@GeneratedValue` | บอกกลยุทธ์การ generate ค่า primary key |
| `@Column` | ระบุรายละเอียดของ column (ชื่อ, ความยาว, nullable, ฯลฯ) |

**GenerationType ที่สำคัญสำหรับ PostgreSQL:**

- `IDENTITY` — ใช้กับ `SERIAL`/`BIGSERIAL`/`GENERATED ALWAYS AS IDENTITY` ให้ PostgreSQL เป็นผู้ generate ค่า จุดอ่อนคือ Hibernate ไม่สามารถทำ batch insert ได้เต็มประสิทธิภาพ (เพราะต้อง insert ทีละแถวเพื่อรู้ id ที่ generate ออกมา)
- `SEQUENCE` — ใช้กับ PostgreSQL sequence โดยตรง (แนะนำมากกว่าถ้าออกแบบ schema ใหม่ได้ เพราะรองรับ batch insert ได้ดีกว่า) เช่น `@GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "product_seq")`
- เนื่องจาก schema ของเราใช้ `SERIAL` (ซึ่งภายในคือ `INTEGER` + sequence + default) การใช้ `IDENTITY` จึงเหมาะสมและตรงไปตรงมาที่สุด

### การตั้งค่า EntityManagerFactory (persistence.xml)

Hibernate/JPA ต้องมีไฟล์ config `META-INF/persistence.xml` วางไว้ใน classpath:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<persistence xmlns="https://jakarta.ee/xml/ns/persistence"
             version="3.0">

    <persistence-unit name="ecommercePU" transaction-type="RESOURCE_LOCAL">
        <provider>org.hibernate.jpa.HibernatePersistenceProvider</provider>

        <class>com.example.entity.Product</class>
        <class>com.example.entity.Customer</class>
        <class>com.example.entity.Order</class>

        <properties>
            <property name="jakarta.persistence.jdbc.url"
                      value="jdbc:postgresql://localhost:5432/ecommerce"/>
            <property name="jakarta.persistence.jdbc.user" value="app_user"/>
            <property name="jakarta.persistence.jdbc.password" value="secret_password"/>
            <property name="jakarta.persistence.jdbc.driver" value="org.postgresql.Driver"/>

            <!-- ใช้ dialect ของ PostgreSQL เพื่อให้ Hibernate generate SQL ที่ถูกต้องกับ PostgreSQL -->
            <property name="hibernate.dialect" value="org.hibernate.dialect.PostgreSQLDialect"/>

            <!-- แสดง SQL ที่ Hibernate ยิงจริงใน console (เปิดเฉพาะตอน dev เท่านั้น) -->
            <property name="hibernate.show_sql" value="true"/>
            <property name="hibernate.format_sql" value="true"/>

            <!-- ห้ามใช้ update/create ใน production เด็ดขาด ให้ควบคุม schema ผ่าน migration tool
                 เช่น Flyway/Liquibase แทน -->
            <property name="hibernate.hbm2ddl.auto" value="validate"/>

            <!-- ใช้ HikariCP เป็น connection pool ให้ Hibernate -->
            <property name="hibernate.connection.provider_class"
                      value="org.hibernate.hikaricp.internal.HikariCPConnectionProvider"/>
            <property name="hibernate.hikari.maximumPoolSize" value="10"/>
        </properties>
    </persistence-unit>
</persistence>
```

### EntityManager — หัวใจของการทำงานกับ JPA

```java
package com.example.jpa;

import jakarta.persistence.EntityManager;
import jakarta.persistence.EntityManagerFactory;
import jakarta.persistence.Persistence;
import com.example.entity.Product;

import java.math.BigDecimal;

public class EntityManagerDemo {

    public static void main(String[] args) {
        // EntityManagerFactory สร้างครั้งเดียวตอน app เริ่มทำงาน (มีค่าใช้จ่ายสูงในการสร้าง)
        EntityManagerFactory emf = Persistence.createEntityManagerFactory("ecommercePU");

        // EntityManager สร้างใหม่ต่อ request/ต่อ transaction (ไม่ thread-safe, ราคาถูก)
        EntityManager em = emf.createEntityManager();

        try {
            em.getTransaction().begin();

            Product newProduct = new Product("Gaming Chair", new BigDecimal("5990.00"), 30);
            em.persist(newProduct); // INSERT จะถูกส่งไปจริงตอน flush/commit

            em.getTransaction().commit();

            System.out.println("บันทึกสินค้าใหม่ id = " + newProduct.getProductId());

        } catch (Exception e) {
            if (em.getTransaction().isActive()) {
                em.getTransaction().rollback();
            }
            throw e;
        } finally {
            em.close();
        }

        emf.close(); // ปิดตอน application shutdown เท่านั้น
    }
}
```

**สิ่งที่ต้องเข้าใจให้แม่นเรื่อง lifecycle:**

- **`EntityManagerFactory`** — เปรียบเสมือน `DataSource` สร้างครั้งเดียว ใช้ตลอดอายุแอปพลิเคชัน (thread-safe)
- **`EntityManager`** — เปรียบเสมือน `Connection` มี lifecycle สั้น ผูกกับ 1 transaction/1 request ไม่ thread-safe ห้าม share ข้าม thread
- **Persistence Context** — พื้นที่ความจำภายใน `EntityManager` ที่เก็บ entity ที่กำลังถูกติดตามอยู่ (managed state) เป็นที่มาของ first-level cache และ dirty checking

---

## Step 897: JPA — CRUD Operations และ JPQL

### CRUD พื้นฐานผ่าน EntityManager

```java
package com.example.jpa;

import jakarta.persistence.EntityManager;
import com.example.entity.Product;
import java.math.BigDecimal;
import java.util.List;

public class ProductJpaService {

    private final EntityManager em;

    public ProductJpaService(EntityManager em) {
        this.em = em;
    }

    // CREATE
    public Product create(String name, BigDecimal price, int stock) {
        em.getTransaction().begin();
        Product product = new Product(name, price, stock);
        em.persist(product);
        em.getTransaction().commit();
        return product;
    }

    // READ — find() ค้นหาด้วย primary key โดยตรง เร็วที่สุดเพราะเช็ค cache ก่อน
    public Product findById(Integer id) {
        return em.find(Product.class, id); // คืนค่า null ถ้าไม่พบ (ไม่ throw exception)
    }

    // UPDATE — ไม่ต้องเรียก save() เอง แค่แก้ field แล้ว commit
    // Hibernate จะตรวจจับการเปลี่ยนแปลง (dirty checking) แล้ว generate UPDATE ให้อัตโนมัติ
    public void updatePrice(Integer id, BigDecimal newPrice) {
        em.getTransaction().begin();
        Product product = em.find(Product.class, id);
        if (product != null) {
            product.setUnitPrice(newPrice); // ไม่ต้อง em.persist() ซ้ำ เพราะ entity นี้ managed อยู่แล้ว
        }
        em.getTransaction().commit();
    }

    // DELETE
    public void delete(Integer id) {
        em.getTransaction().begin();
        Product product = em.find(Product.class, id);
        if (product != null) {
            em.remove(product);
        }
        em.getTransaction().commit();
    }

    // LIST ALL — ใช้ JPQL
    public List<Product> findAll() {
        return em.createQuery("SELECT p FROM Product p ORDER BY p.productId", Product.class)
                 .getResultList();
    }
}
```

### JPQL (Jakarta Persistence Query Language)

JPQL คือภาษา query ที่คล้าย SQL แต่ทำงานกับ **entity/field ของ Java** แทนที่จะเป็น table/column ของฐานข้อมูลโดยตรง ทำให้ query แบบ portable ข้าม database vendor ได้ในระดับหนึ่ง

```java
// ค้นหาสินค้าที่ราคาน้อยกว่าค่าที่กำหนด
public List<Product> findCheaperThan(BigDecimal maxPrice) {
    return em.createQuery(
            "SELECT p FROM Product p WHERE p.unitPrice <= :maxPrice ORDER BY p.unitPrice",
            Product.class)
            .setParameter("maxPrice", maxPrice) // ใช้ named parameter เสมอ (ป้องกัน injection เช่นเดียวกับ PreparedStatement)
            .getResultList();
}

// นับจำนวนสินค้าที่สต็อกหมด
public long countOutOfStock() {
    return em.createQuery(
            "SELECT COUNT(p) FROM Product p WHERE p.stockQuantity = 0", Long.class)
            .getSingleResult();
}

// อัปเดตแบบ bulk (ไม่ผ่าน entity lifecycle, ยิง UPDATE ตรง ๆ)
public int discountAllProducts(BigDecimal percentOff) {
    em.getTransaction().begin();
    int updated = em.createQuery(
            "UPDATE Product p SET p.unitPrice = p.unitPrice * (1 - :pct) WHERE p.stockQuantity > 0")
            .setParameter("pct", percentOff)
            .executeUpdate();
    em.getTransaction().commit();
    return updated;
}

// JOIN ระหว่าง entity (Order กับ Customer)
public List<Object[]> listOrdersWithCustomerName() {
    return em.createQuery(
            "SELECT o.orderId, c.firstName, o.status " +
            "FROM Order o JOIN o.customer c " +
            "ORDER BY o.orderDate DESC", Object[].class)
            .getResultList();
}
```

### Native Query — เมื่อ JPQL ไม่พอ

บางครั้ง JPQL ไม่รองรับฟีเจอร์เฉพาะของ PostgreSQL (เช่น window function, `jsonb` operator, `ILIKE`) สามารถใช้ native SQL ได้โดยตรง:

```java
public List<Product> searchByNameNative(String keyword) {
    return em.createNativeQuery(
            "SELECT * FROM products WHERE product_name ILIKE :kw", Product.class)
            .setParameter("kw", "%" + keyword + "%")
            .getResultList();
}
```

### ตารางเปรียบเทียบ JPQL vs Native Query

| ประเด็น | JPQL | Native Query |
|---|---|---|
| Portability ข้าม database | สูง | ต่ำ (ผูกกับ PostgreSQL syntax) |
| รองรับฟีเจอร์เฉพาะของ PostgreSQL | ไม่รองรับโดยตรง | รองรับเต็มรูปแบบ |
| ทำงานกับ entity/field name | ใช่ | ไม่ใช่ (ใช้ table/column name จริง) |
| เหมาะกับ | query ทั่วไป, business logic มาตรฐาน | query ซับซ้อน, ใช้ฟีเจอร์เฉพาะ DB, tuning performance |

---

## Step 898: Relationship Mapping และปัญหา N+1

### การ map ความสัมพันธ์ระหว่าง Entity

จาก schema ของเรา `orders.customer_id REFERENCES customers(customer_id)` คือความสัมพันธ์แบบ **many-to-one** จากมุมมองของ `Order` (คำสั่งซื้อหลายรายการเป็นของลูกค้าคนเดียวกันได้) และเป็น **one-to-many** จากมุมมองของ `Customer` (ลูกค้าหนึ่งคนมีได้หลายคำสั่งซื้อ)

```java
package com.example.entity;

import jakarta.persistence.*;
import java.util.ArrayList;
import java.util.List;

@Entity
@Table(name = "customers")
public class Customer {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "customer_id")
    private Integer customerId;

    @Column(name = "first_name", length = 60)
    private String firstName;

    @Column(name = "email", length = 150)
    private String email;

    // ฝั่ง "หนึ่ง" ของความสัมพันธ์ — mappedBy ชี้ไปที่ field "customer" ใน Order
    // ไม่มี foreign key column ในตาราง customers เอง (FK อยู่ที่ orders.customer_id)
    @OneToMany(mappedBy = "customer", fetch = FetchType.LAZY, cascade = CascadeType.ALL)
    private List<Order> orders = new ArrayList<>();

    protected Customer() {}

    public Customer(String firstName, String email) {
        this.firstName = firstName;
        this.email = email;
    }

    public Integer getCustomerId() { return customerId; }
    public String getFirstName() { return firstName; }
    public void setFirstName(String firstName) { this.firstName = firstName; }
    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }
    public List<Order> getOrders() { return orders; }
}
```

```java
package com.example.entity;

import jakarta.persistence.*;
import java.time.OffsetDateTime;

@Entity
@Table(name = "orders")
public class Order {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "order_id")
    private Integer orderId;

    // ฝั่ง "หลาย" ของความสัมพันธ์ — มี foreign key column customer_id จริง ๆ ในตาราง
    @ManyToOne(fetch = FetchType.LAZY) // แนะนำให้ LAZY เสมอสำหรับ @ManyToOne
    @JoinColumn(name = "customer_id", referencedColumnName = "customer_id")
    private Customer customer;

    @Column(name = "order_date")
    private OffsetDateTime orderDate;

    @Column(name = "status", length = 20)
    private String status;

    protected Order() {}

    public Order(Customer customer, String status) {
        this.customer = customer;
        this.status = status;
    }

    public Integer getOrderId() { return orderId; }
    public Customer getCustomer() { return customer; }
    public OffsetDateTime getOrderDate() { return orderDate; }
    public String getStatus() { return status; }
    public void setStatus(String status) { this.status = status; }
}
```

### FetchType.LAZY vs FetchType.EAGER

| | `LAZY` | `EAGER` |
|---|---|---|
| ความหมาย | โหลดข้อมูลที่เกี่ยวข้องก็ต่อเมื่อถูกเรียกใช้จริง (ผ่าน proxy) | โหลดข้อมูลที่เกี่ยวข้องทันทีพร้อมกับ entity หลัก |
| Default ของ `@ManyToOne` / `@OneToOne` | `EAGER` (ค่า default ของสเปก) | — |
| Default ของ `@OneToMany` / `@ManyToMany` | `LAZY` (ค่า default ของสเปก) | — |
| ความเสี่ยง | `LazyInitializationException` ถ้าเข้าถึงหลัง `EntityManager` ปิดไปแล้ว | โหลดข้อมูลเกินความจำเป็น, กลายเป็นต้นตอของปัญหา N+1 |
| คำแนะนำ | **ควร override `@ManyToOne` ให้เป็น `LAZY` เสมอ** เพราะ default `EAGER` มักไม่ใช่สิ่งที่ต้องการจริง | ใช้เฉพาะกรณีรู้แน่ชัดว่าต้องใช้ข้อมูลนั้นทุกครั้ง |

### ปัญหา N+1 Query

นี่คือปัญหาที่พบบ่อยที่สุดในการใช้งาน Hibernate ในทางปฏิบัติ ลองดูโค้ดนี้:

```java
// ❌ โค้ดที่ก่อให้เกิดปัญหา N+1
List<Order> orders = em.createQuery("SELECT o FROM Order o", Order.class)
                        .getResultList(); // Query #1: ดึง orders ทั้งหมด (สมมติ 100 รายการ)

for (Order order : orders) {
    // ทุกครั้งที่เข้าถึง order.getCustomer().getFirstName() ถ้า customer ยังไม่ถูกโหลด
    // (LAZY proxy) Hibernate จะยิง SELECT ใหม่ไปหา customers ทันที
    System.out.println(order.getCustomer().getFirstName()); // Query #2 ถึง #101
}
```

โค้ดนี้ดูปกติมาก แต่จริง ๆ แล้วยิง SQL ไปยัง PostgreSQL ถึง **101 ครั้ง** (1 ครั้งสำหรับดึง orders + 100 ครั้งสำหรับดึง customer ของแต่ละ order ทีละตัว) แทนที่จะเป็นแค่ 1-2 ครั้ง นี่คือที่มาของชื่อ "N+1" (1 query หลัก + N query ย่อยตามจำนวนแถว)

### วิธีแก้ปัญหา N+1

**วิธีที่ 1: JOIN FETCH ใน JPQL** (แนะนำที่สุด สำหรับกรณีที่รู้ล่วงหน้าว่าต้องใช้ข้อมูลที่เกี่ยวข้อง)

```java
// ✅ ดึง Order พร้อม Customer มาในคำสั่งเดียว ด้วย JOIN FETCH
List<Order> orders = em.createQuery(
        "SELECT o FROM Order o JOIN FETCH o.customer ORDER BY o.orderDate DESC",
        Order.class)
        .getResultList(); // แค่ Query เดียว! Hibernate ยิง SQL แบบมี JOIN ให้เอง

for (Order order : orders) {
    // ไม่มี query เพิ่มเติม เพราะ customer ถูกโหลดมาพร้อมกันแล้ว
    System.out.println(order.getCustomer().getFirstName());
}
```

SQL ที่ Hibernate generate ออกมาจริง ๆ จะเป็นประมาณนี้:

```sql
SELECT o.order_id, o.customer_id, o.order_date, o.status,
       c.customer_id, c.first_name, c.email
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
ORDER BY o.order_date DESC;
```

**วิธีที่ 2: `@EntityGraph`** (เหมาะกับ Spring Data JPA — จะกล่าวถึงใน Step 899)

**วิธีที่ 3: Batch Fetching** — ตั้งค่า `@BatchSize` เพื่อให้ Hibernate โหลดข้อมูลที่เกี่ยวข้องเป็นชุด (เช่นทีละ 20 รายการ) แทนที่จะโหลดทีละ 1:

```java
@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "customer_id")
@org.hibernate.annotations.BatchSize(size = 20)
private Customer customer;
```

วิธีนี้จะลดจาก 101 query เหลือประมาณ 6 query (1 สำหรับ orders + 5 สำหรับ customers เป็นชุด ชุดละ 20)

### วิธีตรวจจับปัญหา N+1 ในทางปฏิบัติ

- เปิด `hibernate.show_sql=true` และ `hibernate.format_sql=true` ระหว่าง dev เพื่อดูจำนวน query จริงที่ถูกยิงออกไป
- ใช้เครื่องมือเช่น **p6spy** หรือ **datasource-proxy** เพื่อ log SQL statement พร้อม stack trace ต้นทาง
- ใน production ใช้ APM tool (เช่น New Relic, Datadog) ที่รายงานจำนวน query ต่อ HTTP request — ถ้าจำนวน query แปรผันตามจำนวนแถวที่ดึงมา มักเป็นสัญญาณของ N+1
- ไลบรารี **db-util** มี `assertSelectCount()` ที่ใช้ใน unit test เพื่อยืนยันว่าจำนวน query ไม่เกินที่คาดไว้

---

## Step 899: Spring Data JPA — Repository และ Query Method

### แนวคิด

**Spring Data JPA** เป็นส่วนหนึ่งของ Spring Data project ที่ทำหน้าที่ generate implementation ของ Repository interface ให้อัตโนมัติในขณะรันไทม์ ผ่านกลไก dynamic proxy โดยผู้พัฒนาเพียงประกาศ interface เปล่า ๆ ที่ extend จาก `JpaRepository<T, ID>` แล้ว Spring จะสร้าง class ที่ implement CRUD operation พื้นฐานทั้งหมดให้ทันที (`save`, `findById`, `findAll`, `delete`, ฯลฯ) โดยไม่ต้องเขียนโค้ดเอง

### ตั้งค่า Spring Boot กับ PostgreSQL

`application.yml`:

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/ecommerce
    username: app_user
    password: secret_password
    driver-class-name: org.postgresql.Driver
    hikari:
      maximum-pool-size: 10
      minimum-idle: 2
      connection-timeout: 30000
      max-lifetime: 1800000
      pool-name: ecommerce-hikari-pool

  jpa:
    hibernate:
      ddl-auto: validate   # ห้ามใช้ update/create ใน production
    properties:
      hibernate:
        dialect: org.hibernate.dialect.PostgreSQLDialect
        format_sql: true
    show-sql: false          # เปิดเฉพาะ dev
    open-in-view: false      # ปิด OSIV เพื่อป้องกันปัญหาแอบยิง query นอก transaction (ดูรายละเอียดด้านล่าง)
```

> **เรื่อง `open-in-view: false`**: Spring Boot เปิด `spring.jpa.open-in-view=true` เป็น default ซึ่งจะเปิด `EntityManager`/session ค้างไว้ตลอด HTTP request รวมถึงตอน render view ด้วย ทำให้ lazy-loading ทำงานได้แม้อยู่นอก service layer แต่ผลข้างเคียงคือซ่อนปัญหา N+1 ไว้ไม่ให้เห็นชัดเจน (เพราะไม่ throw `LazyInitializationException` แต่กลับไปยิง query เพิ่มเงียบ ๆ) แนวทางปฏิบัติที่ดีในระดับ production คือปิดค่านี้ แล้วจัดการการโหลดข้อมูลที่เกี่ยวข้องให้ครบถ้วนภายใน `@Transactional` service layer อย่างชัดเจน

### Repository Interface

```java
package com.example.repository;

import com.example.entity.Product;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;

import java.math.BigDecimal;
import java.util.List;
import java.util.Optional;

public interface ProductRepository extends JpaRepository<Product, Integer> {

    // ---- Query Method จากชื่อ method (Spring วิเคราะห์ชื่อแล้ว generate query ให้เอง) ----

    // SELECT * FROM products WHERE product_name = ?
    Optional<Product> findByProductName(String productName);

    // SELECT * FROM products WHERE unit_price <= ?
    List<Product> findByUnitPriceLessThanEqual(BigDecimal maxPrice);

    // SELECT * FROM products WHERE stock_quantity > 0 ORDER BY unit_price ASC
    List<Product> findByStockQuantityGreaterThanOrderByUnitPriceAsc(Integer minStock);

    // SELECT * FROM products WHERE product_name ILIKE '%keyword%'
    List<Product> findByProductNameContainingIgnoreCase(String keyword);

    // SELECT COUNT(*) FROM products WHERE stock_quantity = 0
    long countByStockQuantity(Integer stockQuantity);

    // DELETE FROM products WHERE stock_quantity = 0
    void deleteByStockQuantity(Integer stockQuantity);

    // ---- กรณี query ซับซ้อนกว่า ชื่อ method ไม่พอ ให้ใช้ @Query (JPQL) ----
    @Query("SELECT p FROM Product p WHERE p.stockQuantity BETWEEN :min AND :max")
    List<Product> findByStockRange(@Param("min") int min, @Param("max") int max);

    // ---- native query เมื่อจำเป็นต้องใช้ฟีเจอร์เฉพาะของ PostgreSQL ----
    @Query(value = "SELECT * FROM products WHERE product_name ILIKE %:kw%", nativeQuery = true)
    List<Product> searchNative(@Param("kw") String keyword);
}
```

### Query Method Naming Convention

Spring Data JPA วิเคราะห์ชื่อ method แล้วสร้าง query ให้อัตโนมัติ ตามรูปแบบ:

```
find...By<Property>[<Operator>][And/Or<Property>[<Operator>]]...[OrderBy<Property>[Asc/Desc]]
```

| Keyword | ความหมาย | ตัวอย่าง method | SQL ที่ generate |
|---|---|---|---|
| (ไม่ระบุ) | `=` | `findByStatus(String s)` | `WHERE status = ?` |
| `GreaterThan` | `>` | `findByUnitPriceGreaterThan(BigDecimal p)` | `WHERE unit_price > ?` |
| `LessThanEqual` | `<=` | `findByStockQuantityLessThanEqual(int n)` | `WHERE stock_quantity <= ?` |
| `Between` | `BETWEEN` | `findByUnitPriceBetween(BigDecimal min, BigDecimal max)` | `WHERE unit_price BETWEEN ? AND ?` |
| `Like` / `Containing` | `LIKE` / `ILIKE` | `findByProductNameContainingIgnoreCase(String kw)` | `WHERE product_name ILIKE '%'\|\|?\|\|'%'` |
| `In` | `IN` | `findByStatusIn(List<String> statuses)` | `WHERE status IN (...)` |
| `IsNull` / `IsNotNull` | `IS NULL` | `findByEmailIsNull()` | `WHERE email IS NULL` |
| `And` / `Or` | รวมเงื่อนไข | `findByCustomerIdAndStatus(int id, String s)` | `WHERE customer_id = ? AND status = ?` |
| `OrderBy...Asc/Desc` | เรียงลำดับ | `findByCustomerIdOrderByOrderDateDesc(int id)` | `WHERE customer_id = ? ORDER BY order_date DESC` |
| `First` / `Top<N>` | จำกัดจำนวน | `findFirst5ByOrderByUnitPriceDesc()` | `ORDER BY unit_price DESC LIMIT 5` |

### Repository สำหรับ Order (ใช้แก้ปัญหา N+1 ด้วย `@EntityGraph`)

```java
package com.example.repository;

import com.example.entity.Order;
import org.springframework.data.jpa.repository.EntityGraph;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;

import java.util.List;

public interface OrderRepository extends JpaRepository<Order, Integer> {

    // Query method ธรรมดา — findByCustomerId คือ naming convention ที่ตรงตาม spec ของโจทย์
    List<Order> findByCustomerId(Integer customerId);

    // @EntityGraph บอกให้ Spring JOIN FETCH "customer" มาด้วยเสมอ แก้ปัญหา N+1
    // โดยไม่ต้องเขียน JPQL เอง
    @EntityGraph(attributePaths = {"customer"})
    List<Order> findByStatus(String status);

    // ตัวอย่างการเขียน JPQL เองพร้อม JOIN FETCH เมื่อ EntityGraph ไม่ยืดหยุ่นพอ
    @Query("SELECT o FROM Order o JOIN FETCH o.customer WHERE o.orderDate >= :since")
    List<Order> findRecentOrdersWithCustomer(@Param("since") java.time.OffsetDateTime since);
}
```

### การใช้งาน Repository ใน Service Layer

```java
package com.example.service;

import com.example.entity.Product;
import com.example.repository.ProductRepository;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.math.BigDecimal;
import java.util.List;
import java.util.NoSuchElementException;

@Service
public class ProductService {

    private final ProductRepository productRepository;

    // Constructor injection คือแนวทางที่แนะนำ (ไม่ใช้ @Autowired บน field โดยตรง)
    public ProductService(ProductRepository productRepository) {
        this.productRepository = productRepository;
    }

    @Transactional(readOnly = true)
    public List<Product> listAffordable(BigDecimal maxPrice) {
        return productRepository.findByUnitPriceLessThanEqual(maxPrice);
    }

    @Transactional
    public Product create(String name, BigDecimal price, int stock) {
        Product product = new Product(name, price, stock);
        return productRepository.save(product); // save() ทำหน้าที่ทั้ง insert และ update
    }

    @Transactional
    public Product adjustStock(Integer productId, int delta) {
        Product product = productRepository.findById(productId)
                .orElseThrow(() -> new NoSuchElementException("ไม่พบสินค้า id=" + productId));

        int newStock = product.getStockQuantity() + delta;
        if (newStock < 0) {
            throw new IllegalStateException("สต็อกไม่พอสำหรับปรับลด " + Math.abs(delta) + " ชิ้น");
        }
        product.setStockQuantity(newStock); // dirty checking: ไม่ต้องเรียก save() ซ้ำก็ได้ เพราะอยู่ใน @Transactional
        return product;
    }
}
```

`@Transactional` ของ Spring คือ annotation สำคัญที่ทำให้ method ทั้งหมดรันอยู่ภายใน database transaction เดียวกัน — ถ้า method จบโดยไม่มี exception จะ commit อัตโนมัติ ถ้ามี unchecked exception (RuntimeException) หลุดออกมาจะ rollback อัตโนมัติ ค่า `readOnly = true` เป็น hint บอก Hibernate และ driver ว่าไม่มีการเขียนข้อมูล ช่วยเพิ่ม performance ได้เล็กน้อย (เช่น skip dirty checking บาง flush)

---

## Step 900: แบบฝึกหัดรวม — Spring Boot REST API สำหรับจัดการสินค้าและคำสั่งซื้อ

ในหัวข้อสุดท้ายนี้ เราจะประกอบทุกอย่างที่เรียนมาเข้าด้วยกัน สร้าง REST API เล็ก ๆ ที่สมบูรณ์สำหรับระบบ e-commerce โดยใช้ Spring Boot + Spring Data JPA + HikariCP (ซึ่งมากับ Spring Boot อยู่แล้วเป็น default)

### โครงสร้างโปรเจกต์

```
src/main/java/com/example/ecommerce/
├── EcommerceApplication.java
├── entity/
│   ├── Product.java
│   ├── Customer.java
│   └── Order.java
├── repository/
│   ├── ProductRepository.java
│   ├── CustomerRepository.java
│   └── OrderRepository.java
├── service/
│   ├── ProductService.java
│   └── OrderService.java
├── controller/
│   ├── ProductController.java
│   └── OrderController.java
└── dto/
    ├── ProductRequest.java
    ├── ProductResponse.java
    ├── OrderRequest.java
    └── OrderResponse.java
```

### Main Application Class

```java
package com.example.ecommerce;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class EcommerceApplication {
    public static void main(String[] args) {
        SpringApplication.run(EcommerceApplication.class, args);
    }
}
```

### DTO Classes (แยกจาก Entity เสมอ เพื่อไม่ให้ internal model รั่วไหลออกสู่ API ภายนอก)

```java
package com.example.ecommerce.dto;

import java.math.BigDecimal;

public record ProductRequest(String productName, BigDecimal unitPrice, Integer stockQuantity) {}
```

```java
package com.example.ecommerce.dto;

import java.math.BigDecimal;

public record ProductResponse(Integer productId, String productName,
                               BigDecimal unitPrice, Integer stockQuantity) {

    public static ProductResponse from(com.example.ecommerce.entity.Product p) {
        return new ProductResponse(p.getProductId(), p.getProductName(),
                p.getUnitPrice(), p.getStockQuantity());
    }
}
```

```java
package com.example.ecommerce.dto;

public record OrderRequest(Integer customerId, Integer productId, Integer quantity) {}
```

```java
package com.example.ecommerce.dto;

import java.time.OffsetDateTime;

public record OrderResponse(Integer orderId, Integer customerId, String customerName,
                             String status, OffsetDateTime orderDate) {

    public static OrderResponse from(com.example.ecommerce.entity.Order o) {
        return new OrderResponse(
                o.getOrderId(),
                o.getCustomer().getCustomerId(),
                o.getCustomer().getFirstName(),
                o.getStatus(),
                o.getOrderDate());
    }
}
```

### Product Controller

```java
package com.example.ecommerce.controller;

import com.example.ecommerce.dto.ProductRequest;
import com.example.ecommerce.dto.ProductResponse;
import com.example.ecommerce.service.ProductService;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.math.BigDecimal;
import java.util.List;

@RestController
@RequestMapping("/api/products")
public class ProductController {

    private final ProductService productService;

    public ProductController(ProductService productService) {
        this.productService = productService;
    }

    @GetMapping
    public List<ProductResponse> list(
            @RequestParam(required = false) BigDecimal maxPrice) {

        if (maxPrice != null) {
            return productService.listAffordable(maxPrice)
                    .stream().map(ProductResponse::from).toList();
        }
        return productService.listAll()
                .stream().map(ProductResponse::from).toList();
    }

    @GetMapping("/{id}")
    public ResponseEntity<ProductResponse> getById(@PathVariable Integer id) {
        return productService.findById(id)
                .map(ProductResponse::from)
                .map(ResponseEntity::ok)
                .orElseGet(() -> ResponseEntity.notFound().build());
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public ProductResponse create(@RequestBody ProductRequest request) {
        var product = productService.create(
                request.productName(), request.unitPrice(), request.stockQuantity());
        return ProductResponse.from(product);
    }

    @PatchMapping("/{id}/stock")
    public ProductResponse adjustStock(@PathVariable Integer id,
                                        @RequestParam int delta) {
        var product = productService.adjustStock(id, delta);
        return ProductResponse.from(product);
    }

    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void delete(@PathVariable Integer id) {
        productService.delete(id);
    }

    // ---- จัดการ error แบบรวมศูนย์สำหรับ controller นี้ ----
    @ExceptionHandler(java.util.NoSuchElementException.class)
    public ResponseEntity<String> handleNotFound(java.util.NoSuchElementException e) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(e.getMessage());
    }

    @ExceptionHandler(IllegalStateException.class)
    public ResponseEntity<String> handleBadState(IllegalStateException e) {
        return ResponseEntity.status(HttpStatus.CONFLICT).body(e.getMessage());
    }
}
```

### Order Service — ประกอบ transaction + relationship + pessimistic lock

```java
package com.example.ecommerce.service;

import com.example.ecommerce.entity.Customer;
import com.example.ecommerce.entity.Order;
import com.example.ecommerce.entity.Product;
import com.example.ecommerce.repository.CustomerRepository;
import com.example.ecommerce.repository.OrderRepository;
import com.example.ecommerce.repository.ProductRepository;
import jakarta.persistence.LockModeType;
import org.springframework.data.jpa.repository.Lock;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;
import java.util.NoSuchElementException;

@Service
public class OrderService {

    private final OrderRepository orderRepository;
    private final ProductRepository productRepository;
    private final CustomerRepository customerRepository;

    public OrderService(OrderRepository orderRepository,
                         ProductRepository productRepository,
                         CustomerRepository customerRepository) {
        this.orderRepository = orderRepository;
        this.productRepository = productRepository;
        this.customerRepository = customerRepository;
    }

    @Transactional
    public Order placeOrder(Integer customerId, Integer productId, int quantity) {

        Customer customer = customerRepository.findById(customerId)
                .orElseThrow(() -> new NoSuchElementException("ไม่พบลูกค้า id=" + customerId));

        // ใช้ pessimistic lock (SELECT ... FOR UPDATE) เพื่อป้องกัน race condition
        // เมื่อมีหลาย request แข่งกันซื้อสินค้าชิ้นสุดท้ายพร้อมกัน
        Product product = productRepository.findByIdForUpdate(productId)
                .orElseThrow(() -> new NoSuchElementException("ไม่พบสินค้า id=" + productId));

        if (product.getStockQuantity() < quantity) {
            throw new IllegalStateException(
                    "สต็อกไม่เพียงพอ (คงเหลือ " + product.getStockQuantity() + ")");
        }

        product.setStockQuantity(product.getStockQuantity() - quantity);

        Order order = new Order(customer, "PENDING");
        return orderRepository.save(order);
    }

    @Transactional(readOnly = true)
    public List<Order> listByCustomer(Integer customerId) {
        return orderRepository.findByCustomerId(customerId);
    }

    @Transactional
    public Order updateStatus(Integer orderId, String newStatus) {
        Order order = orderRepository.findById(orderId)
                .orElseThrow(() -> new NoSuchElementException("ไม่พบคำสั่งซื้อ id=" + orderId));
        order.setStatus(newStatus);
        return order;
    }
}
```

เพิ่ม method ล็อกแถวใน `ProductRepository`:

```java
public interface ProductRepository extends JpaRepository<Product, Integer> {

    // ... query methods เดิม ...

    @Lock(LockModeType.PESSIMISTIC_WRITE) // แปลงเป็น SELECT ... FOR UPDATE โดยอัตโนมัติ
    @Query("SELECT p FROM Product p WHERE p.productId = :id")
    java.util.Optional<Product> findByIdForUpdate(@Param("id") Integer id);
}
```

### Order Controller

```java
package com.example.ecommerce.controller;

import com.example.ecommerce.dto.OrderRequest;
import com.example.ecommerce.dto.OrderResponse;
import com.example.ecommerce.service.OrderService;
import org.springframework.http.HttpStatus;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/api/orders")
public class OrderController {

    private final OrderService orderService;

    public OrderController(OrderService orderService) {
        this.orderService = orderService;
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public OrderResponse create(@RequestBody OrderRequest request) {
        var order = orderService.placeOrder(
                request.customerId(), request.productId(), request.quantity());
        return OrderResponse.from(order);
    }

    @GetMapping
    public List<OrderResponse> listByCustomer(@RequestParam Integer customerId) {
        return orderService.listByCustomer(customerId)
                .stream().map(OrderResponse::from).toList();
    }

    @PatchMapping("/{id}/status")
    public OrderResponse updateStatus(@PathVariable Integer id,
                                       @RequestParam String status) {
        return OrderResponse.from(orderService.updateStatus(id, status));
    }
}
```

### ทดสอบด้วย curl

```bash
# สร้างสินค้าใหม่
curl -X POST http://localhost:8080/api/products \
  -H "Content-Type: application/json" \
  -d '{"productName":"Webcam 1080p","unitPrice":1290.00,"stockQuantity":80}'

# ดูสินค้าที่ราคาไม่เกิน 1000 บาท
curl "http://localhost:8080/api/products?maxPrice=1000"

# สั่งซื้อสินค้า
curl -X POST http://localhost:8080/api/orders \
  -H "Content-Type: application/json" \
  -d '{"customerId":1,"productId":2,"quantity":3}'

# ดูคำสั่งซื้อของลูกค้า id=1
curl "http://localhost:8080/api/orders?customerId=1"

# อัปเดตสถานะคำสั่งซื้อ
curl -X PATCH "http://localhost:8080/api/orders/1/status?status=SHIPPED"
```

### สิ่งที่โค้ดชุดนี้แสดงให้เห็นครบทุกแนวคิดของบทนี้

1. **Connection Pooling** — Spring Boot ใช้ HikariCP เป็น default อัตโนมัติ ผ่านการตั้งค่าใน `application.yml`
2. **Transaction Management** — `@Transactional` ครอบทุก business operation แทนการเขียน `conn.commit()`/`rollback()` เอง
3. **Entity Relationship** — `Order` ↔ `Customer` ผ่าน `@ManyToOne`/`@OneToMany`
4. **ป้องกัน race condition** — ใช้ pessimistic lock (`@Lock(PESSIMISTIC_WRITE)`) เทียบเท่ากับ `SELECT ... FOR UPDATE` ใน raw JDBC
5. **Query Method Naming Convention** — `findByCustomerId` ที่โจทย์กำหนด สร้าง query อัตโนมัติโดยไม่ต้องเขียน SQL
6. **DTO Pattern** — แยก Entity (internal) ออกจาก Request/Response (external contract) ป้องกันปัญหา over-fetching และ serialization loop จาก lazy-loaded field

---

## สรุปท้ายบท

บทนี้พาเราเดินทางจากระดับต่ำสุด (raw JDBC ที่คุยกับ PostgreSQL โดยตรง) ไปจนถึงระดับสูงสุด (Spring Data JPA ที่ generate โค้ดให้อัตโนมัติ) แต่ละชั้นไม่ได้แทนที่กันโดยสมบูรณ์ — ทุกชั้นยังคงพึ่งพา pgJDBC driver อยู่ข้างใต้เสมอ และแต่ละชั้นก็มีที่ทางของตัวเองในสถานการณ์ที่ต่างกัน

### ตารางเปรียบเทียบสรุป: JDBC vs Hibernate/JPA vs Spring Data JPA

| หัวข้อ | JDBC (raw) | Hibernate/JPA | Spring Data JPA |
|---|---|---|---|
| **ปริมาณโค้ด boilerplate** | มากที่สุด (ResultSet mapping เอง) | ปานกลาง (ยังต้องเขียน EntityManager code) | น้อยที่สุด (Repository interface เปล่า) |
| **การควบคุม SQL** | ควบคุมได้ 100% เห็น SQL ทุกตัวอักษร | ควบคุมผ่าน JPQL/native query ได้ แต่ Hibernate generate SQL บางส่วนเอง | ควบคุมน้อยที่สุด แต่ override ด้วย `@Query` ได้เสมอ |
| **Transaction management** | เขียนเอง (`setAutoCommit`, `commit`, `rollback`) | เขียนเอง (`EntityTransaction`) หรือใช้ container-managed | ประกาศผ่าน `@Transactional` เท่านั้น |
| **Connection Pooling** | ต้องตั้งเอง (เช่น HikariCP โดยตรง) | ตั้งผ่าน persistence.xml properties | มาพร้อม Spring Boot auto-configuration |
| **Caching** | ไม่มีในตัว ต้องทำเอง | มี first-level cache (persistence context) และ second-level cache (เสริม) | สืบทอดจาก Hibernate ทั้งหมด |
| **ความเสี่ยงเรื่อง N+1** | ไม่มี (เพราะเขียน SQL เองทุกครั้ง) | มีความเสี่ยงสูงถ้าใช้ LAZY loading ไม่ระวัง | มีความเสี่ยงเช่นเดียวกับ Hibernate แต่มี `@EntityGraph` ช่วยจัดการง่ายขึ้น |
| **Performance งาน batch/report ขนาดใหญ่** | ดีที่สุด (ควบคุม SQL, batch ได้เต็มที่) | รองลงมา (มี overhead ของ ORM) | เทียบเท่า Hibernate |
| **ความเร็วในการพัฒนา feature ใหม่** | ช้าที่สุด | เร็วขึ้นมาก | เร็วที่สุด |
| **Learning curve** | ต่ำ (แค่รู้ JDBC API) | สูง (ต้องเข้าใจ entity lifecycle, cache, lazy loading) | ปานกลาง (ต้องเข้าใจ JPA ก่อน แล้วค่อยเรียนรู้ convention ของ Spring) |
| **เหมาะกับสถานการณ์** | Batch job, ETL, report ที่ query ซับซ้อนมาก, ระบบ legacy | แอปพลิเคชันขนาดกลาง-ใหญ่ ที่มี business object ซับซ้อนและต้องการความยืดหยุ่น | REST API / microservice ที่เน้นความเร็วในการพัฒนา CRUD เป็นหลัก |

### หลักการเลือกใช้ในทางปฏิบัติ

ในโปรเจกต์จริงระดับ production มักไม่ได้เลือกใช้เพียงแบบใดแบบหนึ่งเท่านั้น แต่ใช้ **ผสมกัน**:

- ใช้ **Spring Data JPA** เป็นค่าเริ่มต้นสำหรับ CRUD operation ทั่วไป (80-90% ของ use case)
- เมื่อเจอ query ที่ซับซ้อนเกินกว่า query method naming convention จะรองรับได้ ให้ใช้ **`@Query` แบบ JPQL** หรือ **native query**
- เมื่อเจองาน batch, report, หรือ query ที่ต้อง tune performance อย่างละเอียด (เช่น bulk export ข้อมูลหลายล้านแถว) ให้ตกลงมาใช้ **JDBC ดิบ** โดยตรง (Spring มี `JdbcTemplate` ที่ช่วยลด boilerplate ของ JDBC ได้โดยไม่ต้องผ่าน ORM เต็มรูปแบบ)
- เสมอ**ระวังปัญหา N+1** ไม่ว่าจะใช้ layer ไหนก็ตาม ด้วยการเปิด `show_sql` ระหว่าง dev และทดสอบด้วยข้อมูลปริมาณมากพอที่จะเห็นปัญหาจริง

---

## แบบฝึกหัด

<details>
<summary><b>แบบฝึกหัดที่ 1:</b> จงอธิบายว่าเหตุใดโค้ดต่อไปนี้จึงเป็นอันตราย และควรแก้ไขอย่างไร</summary>

```java
String status = request.getParameter("status");
Statement stmt = conn.createStatement();
ResultSet rs = stmt.executeQuery("SELECT * FROM orders WHERE status = '" + status + "'");
```

**เฉลย:**

โค้ดนี้เสี่ยงต่อ **SQL Injection** เพราะนำค่าที่รับมาจาก user (`request.getParameter`) ไปต่อ string เข้ากับคำสั่ง SQL โดยตรงโดยไม่มีการ escape หรือ validate ใด ๆ หาก user ส่งค่า `status` เป็น `' OR '1'='1` จะได้ query กลายเป็น `SELECT * FROM orders WHERE status = '' OR '1'='1'` ซึ่งคืนข้อมูลคำสั่งซื้อทั้งหมดในระบบ หรือแย่กว่านั้นสามารถใช้เทคนิค UNION-based injection เพื่อดึงข้อมูลจากตารางอื่น เช่น `customers` ได้

**วิธีแก้:** ใช้ `PreparedStatement` พร้อม parameter binding แทน:

```java
String sql = "SELECT * FROM orders WHERE status = ?";
try (PreparedStatement ps = conn.prepareStatement(sql)) {
    ps.setString(1, status);
    try (ResultSet rs = ps.executeQuery()) {
        // ...
    }
}
```

`PreparedStatement` ใช้ extended query protocol ที่แยก SQL structure ออกจาก parameter value ตั้งแต่ระดับ wire protocol ทำให้ PostgreSQL ไม่มีทางตีความค่า parameter เป็นส่วนหนึ่งของ SQL syntax ได้เลย

</details>

<details>
<summary><b>แบบฝึกหัดที่ 2:</b> เขียนเมธอด JDBC ที่ค้นหาลูกค้าจาก email แล้วคืนค่าเป็น <code>Optional&lt;Customer&gt;</code> (สมมติมี POJO <code>Customer</code> พร้อมแล้ว)</summary>

**เฉลย:**

```java
public Optional<Customer> findByEmail(String email) throws SQLException {
    String sql = "SELECT customer_id, first_name, email FROM customers WHERE email = ?";

    try (Connection conn = DataSourceProvider.getConnection();
         PreparedStatement ps = conn.prepareStatement(sql)) {

        ps.setString(1, email);

        try (ResultSet rs = ps.executeQuery()) {
            if (rs.next()) {
                Customer customer = new Customer(
                        rs.getInt("customer_id"),
                        rs.getString("first_name"),
                        rs.getString("email")
                );
                return Optional.of(customer);
            }
            return Optional.empty();
        }
    }
}
```

จุดสำคัญ: ใช้ `PreparedStatement` เสมอ, ตรวจ `rs.next()` ก่อนอ่านค่า (คืนค่าเป็น `true` ถ้ามีแถว), และห่อผลลัพธ์ด้วย `Optional` เพื่อบังคับให้ผู้เรียกใช้ตรวจสอบกรณีไม่พบข้อมูลอย่างชัดเจนแทนที่จะคืน `null` เฉย ๆ

</details>

<details>
<summary><b>แบบฝึกหัดที่ 3:</b> ตั้งค่า HikariCP ให้เหมาะสมกับแอปพลิเคชันที่รันบนเครื่อง 4 CPU core และต้องการรองรับ concurrent request สูง จงอธิบายเหตุผลของค่าที่เลือก</summary>

**เฉลย:**

```java
HikariConfig config = new HikariConfig();
config.setJdbcUrl("jdbc:postgresql://localhost:5432/ecommerce");
config.setUsername("app_user");
config.setPassword("secret_password");

// สูตรอ้างอิงจาก HikariCP wiki: connections = ((core_count * 2) + effective_spindle_count)
// สำหรับ SSD/cloud disk (spindle count ประมาณ 1): (4*2)+1 = 9 ปัดขึ้นเป็น 10
config.setMaximumPoolSize(10);
config.setMinimumIdle(10); // แนะนำให้เท่ากับ maximumPoolSize เพื่อความเสถียรของ pool

config.setConnectionTimeout(30_000);
config.setMaxLifetime(1_800_000);
```

**เหตุผล:** ขนาด pool ไม่ควรใหญ่เกินความจำเป็น เพราะ PostgreSQL ใช้ process-per-connection การมี connection มากเกินไปจะทำให้ context-switching ระหว่าง process สูงขึ้นและ throughput รวมกลับลดลง (Little's Law และงานวิจัยของทีม HikariCP ชี้ว่า pool ขนาดเล็กพอเหมาะมักให้ throughput สูงกว่า pool ขนาดใหญ่เกินจำเป็น) การตั้ง `minimumIdle` เท่ากับ `maximumPoolSize` ทำให้ pool คงที่ (fixed-size pool) ซึ่งเป็นคำแนะนำอย่างเป็นทางการจาก HikariCP wiki เพื่อหลีกเลี่ยงพฤติกรรมแปลกปลอมจากการสร้าง/ทำลาย connection ขึ้นลงตาม load

</details>

<details>
<summary><b>แบบฝึกหัดที่ 4:</b> เขียนโค้ด JDBC transaction ที่โอนสินค้าคงคลัง (stock) จากสินค้า A ไปยังสินค้า B จำนวน 10 ชิ้น โดยต้องรับประกันว่าถ้าขั้นตอนใดล้มเหลว จะไม่มีการเปลี่ยนแปลงข้อมูลเลย</summary>

**เฉลย:**

```java
public void transferStock(int fromProductId, int toProductId, int quantity) throws SQLException {
    String deductSql = "UPDATE products SET stock_quantity = stock_quantity - ? " +
                        "WHERE product_id = ? AND stock_quantity >= ?";
    String addSql = "UPDATE products SET stock_quantity = stock_quantity + ? WHERE product_id = ?";

    try (Connection conn = DataSourceProvider.getConnection()) {
        conn.setAutoCommit(false);

        try (PreparedStatement deductPs = conn.prepareStatement(deductSql);
             PreparedStatement addPs = conn.prepareStatement(addSql)) {

            deductPs.setInt(1, quantity);
            deductPs.setInt(2, fromProductId);
            deductPs.setInt(3, quantity); // เงื่อนไขกันสต็อกติดลบในระดับ SQL เอง
            int deducted = deductPs.executeUpdate();

            if (deducted == 0) {
                throw new SQLException("สินค้าต้นทาง id=" + fromProductId +
                        " มีสต็อกไม่พอสำหรับโอน " + quantity + " ชิ้น");
            }

            addPs.setInt(1, quantity);
            addPs.setInt(2, toProductId);
            int added = addPs.executeUpdate();

            if (added == 0) {
                throw new SQLException("ไม่พบสินค้าปลายทาง id=" + toProductId);
            }

            conn.commit();

        } catch (SQLException e) {
            conn.rollback();
            throw e;
        } finally {
            conn.setAutoCommit(true);
        }
    }
}
```

จุดสำคัญ: ใช้เงื่อนไข `stock_quantity >= ?` ใน `WHERE` clause แทนการอ่านค่าแล้วเช็คใน Java (ลด race condition โดยไม่ต้องใช้ `FOR UPDATE` ก็ได้ในกรณีนี้ เพราะ PostgreSQL รับประกัน atomicity ของ single-row UPDATE อยู่แล้ว) และตรวจค่าที่คืนจาก `executeUpdate()` เสมอเพื่อยืนยันว่ามีแถวถูกอัปเดตจริง

</details>

<details>
<summary><b>แบบฝึกหัดที่ 5:</b> เขียน Entity class <code>Order</code> ที่ map กับตาราง <code>orders</code> ให้ถูกต้อง โดยกำหนดว่า <code>order_date</code> เป็น <code>TIMESTAMPTZ</code> และมีค่า default เป็น <code>now()</code></summary>

**เฉลย:**

```java
@Entity
@Table(name = "orders")
public class Order {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "order_id")
    private Integer orderId;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "customer_id")
    private Customer customer;

    // insertable = false, updatable = false เพราะค่านี้ถูกกำหนดโดย DEFAULT now() ของฐานข้อมูล
    // ไม่ควรให้ Hibernate ส่งค่านี้ไปตอน INSERT/UPDATE เอง
    @Column(name = "order_date", insertable = false, updatable = false)
    private OffsetDateTime orderDate;

    @Column(name = "status", length = 20)
    private String status;

    protected Order() {}

    public Order(Customer customer, String status) {
        this.customer = customer;
        this.status = status;
    }

    // getters ...
}
```

จุดสำคัญ: ใช้ `OffsetDateTime` แทน `LocalDateTime` เพราะ `TIMESTAMPTZ` ของ PostgreSQL เก็บข้อมูลพร้อม timezone offset ส่วน `insertable = false, updatable = false` ป้องกันไม่ให้ Hibernate พยายามเขียนค่า Java-side ทับค่า `DEFAULT now()` ที่ฐานข้อมูลกำหนดไว้ ทำให้ค่า timestamp ถูกกำหนดโดย PostgreSQL server เสมอ (แม่นยำกว่าและสอดคล้องกันไม่ว่าจะเขียนผ่าน client ไหน)

</details>

<details>
<summary><b>แบบฝึกหัดที่ 6:</b> จงระบุว่าโค้ดต่อไปนี้มีปัญหา N+1 หรือไม่ พร้อมอธิบายเหตุผล และถ้ามีให้แก้ไข</summary>

```java
List<Customer> customers = em.createQuery("SELECT c FROM Customer c", Customer.class)
                              .getResultList();

int totalOrders = 0;
for (Customer c : customers) {
    totalOrders += c.getOrders().size(); // getOrders() คือ @OneToMany(fetch = LAZY)
}
```

**เฉลย:**

ใช่ โค้ดนี้มีปัญหา N+1 อย่างชัดเจน เพราะ query แรกดึง `Customer` มาทั้งหมด (1 query) แต่ทุกครั้งที่เรียก `c.getOrders()` ในลูป ถ้า `orders` เป็น `FetchType.LAZY` (ซึ่งเป็นค่า default ของ `@OneToMany`) Hibernate จะยิง query แยกไปดึง orders ของลูกค้าคนนั้นทันที ทำให้รวมแล้วเป็น 1 + N query (N = จำนวนลูกค้า)

**วิธีแก้ด้วย JOIN FETCH:**

```java
List<Customer> customers = em.createQuery(
        "SELECT DISTINCT c FROM Customer c LEFT JOIN FETCH c.orders",
        Customer.class)
        .getResultList();
```

ใช้ `DISTINCT` เพื่อป้องกัน Customer ซ้ำในผลลัพธ์ (เพราะ JOIN แบบ one-to-many จะทำให้แถวของ Customer ซ้ำตามจำนวน Order ที่ join มา) และใช้ `LEFT JOIN FETCH` แทน `JOIN FETCH` ธรรมดา เพื่อให้ลูกค้าที่ยังไม่มีคำสั่งซื้อเลยก็ยังปรากฏในผลลัพธ์ (ไม่ถูกตัดออกเหมือน inner join)

หากต้องการนับจำนวนเท่านั้น อีกทางเลือกที่มีประสิทธิภาพกว่าคือใช้ aggregate query โดยตรงแทนการโหลด entity ทั้งหมด:

```java
Long totalOrders = em.createQuery(
        "SELECT COUNT(o) FROM Order o", Long.class)
        .getSingleResult();
```

</details>

<details>
<summary><b>แบบฝึกหัดที่ 7:</b> เขียน Spring Data JPA Repository method (ใช้ query method naming convention) สำหรับความต้องการต่อไปนี้: "ค้นหาคำสั่งซื้อทั้งหมดของลูกค้าคนหนึ่ง ที่มีสถานะ SHIPPED หรือ DELIVERED เรียงจากวันที่ล่าสุดไปเก่าสุด"</summary>

**เฉลย:**

```java
public interface OrderRepository extends JpaRepository<Order, Integer> {

    List<Order> findByCustomerIdAndStatusInOrderByOrderDateDesc(
            Integer customerId, List<String> statuses);
}
```

เรียกใช้:

```java
List<Order> completed = orderRepository.findByCustomerIdAndStatusInOrderByOrderDateDesc(
        customerId, List.of("SHIPPED", "DELIVERED"));
```

การแยกส่วนของชื่อ method: `findBy` (คำเริ่มต้นมาตรฐาน) + `CustomerId` (`WHERE customer_id = ?`) + `And` (เชื่อมเงื่อนไข) + `StatusIn` (`AND status IN (?)`) + `OrderByOrderDateDesc` (`ORDER BY order_date DESC`) — Spring Data JPA จะ parse ชื่อนี้แล้ว generate JPQL ให้อัตโนมัติโดยไม่ต้องเขียน query เอง

หากชื่อ method เริ่มยาวและอ่านยากเกินไป (เกิน 3-4 เงื่อนไข) ควรเปลี่ยนไปใช้ `@Query` แทนเพื่อความชัดเจน

</details>

<details>
<summary><b>แบบฝึกหัดที่ 8:</b> อธิบายความแตกต่างระหว่าง <code>@Transactional(readOnly = true)</code> กับไม่ใส่ <code>readOnly</code> เลย และควรใช้เมื่อใด</summary>

**เฉลย:**

`@Transactional(readOnly = true)` เป็น hint ที่บอกทั้ง Spring และ Hibernate ว่า method นี้จะไม่มีการเขียนข้อมูลใด ๆ ลงฐานข้อมูล ผลที่เกิดขึ้นจริงมีหลายระดับ:

1. **ระดับ Hibernate**: session จะ skip dirty-checking (การตรวจสอบว่า entity ที่โหลดมามีการเปลี่ยนแปลงหรือไม่ก่อน flush) ทำให้ลด CPU overhead โดยเฉพาะเมื่อโหลด entity จำนวนมาก
2. **ระดับ Spring/driver**: ในบาง connection pool และบาง driver การตั้งค่านี้อาจถูกส่งต่อไปเป็น `Connection.setReadOnly(true)` ซึ่งบางระบบ (เช่น read replica routing) ใช้ค่านี้ในการตัดสินใจ route query ไปยัง read replica แทน primary
3. **ความชัดเจนของโค้ด**: เป็นเอกสารในตัวโค้ดเองว่า method นี้มีเจตนาเป็น read-only

**ควรใช้เมื่อ**: ทุก method ที่เป็นการ query อย่างเดียว (list, findById, search, report) ควรใส่ `readOnly = true` เสมอ เพราะไม่มีข้อเสีย และช่วย performance เล็กน้อยเสมอ

**ไม่ควรใช้เมื่อ**: method ที่มีการ `save`, `delete`, หรือแก้ไข field ของ managed entity (dirty checking ต้องทำงาน) — ถ้าใส่ `readOnly = true` ผิดที่ การเปลี่ยนแปลงข้อมูลจะไม่ถูก flush ลงฐานข้อมูลจริง (เงียบ ๆ โดยไม่มี error ชัดเจน) เป็นบั๊กที่ตรวจจับยากมาก

</details>

<details>
<summary><b>แบบฝึกหัดที่ 9:</b> ในสถานการณ์ที่มีหลาย request พยายามสั่งซื้อสินค้าชิ้นสุดท้ายพร้อมกัน (concurrent order) จงอธิบายว่าทำไมโค้ดต่อไปนี้จึงมีปัญหา race condition และเสนอวิธีแก้ด้วย pessimistic lock