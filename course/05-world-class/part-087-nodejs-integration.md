# Part 087: เชื่อมต่อ PostgreSQL กับ Node.js (node-postgres, Prisma)

> หลักสูตร PostgreSQL ฉบับสมบูรณ์ | ระดับโลก/ผู้เชี่ยวชาญ | Part 087

---

## เป้าหมายการเรียนรู้

หลังจบบทนี้ ผู้เรียนจะสามารถ:

1. เข้าใจภาพรวมของตัวเลือกในการเชื่อมต่อ PostgreSQL จาก Node.js ทั้งแบบ driver ดิบ (`pg`) และ ORM (Prisma, Drizzle, Sequelize) พร้อมเลือกใช้ให้เหมาะกับสถานการณ์
2. ติดตั้งและตั้งค่า `node-postgres` (pg) ได้อย่างถูกต้อง เข้าใจความแตกต่างระหว่าง `Pool` และ `Client` และวิธีตั้ง connection string ที่ปลอดภัย
3. เขียน parameterized query ด้วย `$1, $2, ...` เพื่อป้องกัน SQL Injection แทนการต่อ string โดยตรง
4. จัดการ Transaction ด้วย `BEGIN`/`COMMIT`/`ROLLBACK` ผ่าน `client.query()` พร้อมรูปแบบ try/catch/finally ที่ปลอดภัยและคืน connection กลับสู่ pool เสมอ
5. เขียนโค้ดแบบ async/await ที่ถูกหลัก จัดการ connection pool อย่างมีประสิทธิภาพ และดักจับ error อย่างเป็นระบบ
6. เข้าใจแนวคิดของ Prisma ORM ตั้งแต่ `schema.prisma`, การ generate client ไปจนถึงข้อดีของ type-safety
7. เขียน CRUD operation พื้นฐานด้วย Prisma (`findMany`, `create`, `update`, `delete`) และเปรียบเทียบกับ raw SQL ที่เทียบเท่ากัน
8. ใช้ Prisma Migrate (`migrate dev` / `migrate deploy`) ในการจัดการ schema migration และเข้าใจแนวคิด zero-downtime migration ที่เชื่อมโยงกับ Part 079
9. เขียน relation query ด้วย Prisma (`include`, nested writes) และเข้าใจ SQL JOIN ที่ Prisma สร้างอยู่เบื้องหลัง
10. ประยุกต์ความรู้ทั้งหมดสร้าง REST API เล็กๆ ด้วย Express + node-postgres/Prisma สำหรับจัดการสินค้าและคำสั่งซื้อของระบบ e-commerce

---

## เตรียมข้อมูล

ตลอดบทนี้เราจะใช้ schema ของระบบ e-commerce ชุดเดิมที่ใช้มาตลอดหลักสูตร ประกอบด้วย 3 ตาราง:

```sql
CREATE TABLE products (
    product_id      SERIAL PRIMARY KEY,
    product_name    VARCHAR(150),
    unit_price      NUMERIC(10,2),
    stock_quantity  INTEGER
);

CREATE TABLE customers (
    customer_id     SERIAL PRIMARY KEY,
    first_name      VARCHAR(60),
    email           VARCHAR(150)
);

CREATE TABLE orders (
    order_id        SERIAL PRIMARY KEY,
    customer_id     INTEGER REFERENCES customers(customer_id),
    order_date      TIMESTAMPTZ DEFAULT now(),
    status          VARCHAR(20)
);
```

ข้อมูลตัวอย่างสำหรับทดสอบ:

```sql
INSERT INTO customers (first_name, email) VALUES
    ('สมชาย', 'somchai@example.com'),
    ('สมหญิง', 'somying@example.com'),
    ('วิชัย', 'wichai@example.com');

INSERT INTO products (product_name, unit_price, stock_quantity) VALUES
    ('คีย์บอร์ดกลไก', 1590.00, 25),
    ('เมาส์ไร้สาย', 690.00, 50),
    ('จอมอนิเตอร์ 27 นิ้ว', 6900.00, 10),
    ('หูฟังบลูทูธ', 1290.00, 30);

INSERT INTO orders (customer_id, status) VALUES
    (1, 'pending'),
    (2, 'completed'),
    (1, 'shipped');
```

สมมติว่า PostgreSQL รันอยู่บน `localhost:5432` มี database ชื่อ `ecommerce_db` และผู้ใช้ `app_user` รหัสผ่าน `secretpassword` (ในตัวอย่างจริงควรใช้ environment variable เสมอ ไม่ hardcode)

โปรเจกต์ Node.js เริ่มต้นด้วย:

```bash
mkdir ecommerce-api && cd ecommerce-api
npm init -y
npm install pg dotenv
npm install --save-dev @types/pg typescript ts-node nodemon
```

ไฟล์ `.env`:

```bash
DATABASE_URL=postgresql://app_user:secretpassword@localhost:5432/ecommerce_db
PGHOST=localhost
PGPORT=5432
PGDATABASE=ecommerce_db
PGUSER=app_user
PGPASSWORD=secretpassword
```

> **คำเตือนด้านความปลอดภัย**: ห้าม commit ไฟล์ `.env` ที่มีรหัสผ่านจริงเข้า git repository เด็ดขาด ให้เพิ่ม `.env` ใน `.gitignore` และใช้ `.env.example` เป็นแม่แบบแทน

---

## Step 861: ภาพรวมการเชื่อมต่อ PostgreSQL จาก Node.js

### ทำไมต้องรู้จักหลายตัวเลือก

ระบบนิเวศของ Node.js มีเครื่องมือเชื่อมต่อฐานข้อมูลหลายระดับ ตั้งแต่ driver ดิบที่คุยกับ PostgreSQL protocol โดยตรง ไปจนถึง ORM (Object-Relational Mapper) ที่แปลง object ใน JavaScript/TypeScript ให้กลายเป็น SQL โดยอัตโนมัติ การเลือกเครื่องมือที่เหมาะสมส่งผลต่อ performance, maintainability, และความเร็วในการพัฒนาอย่างมาก

### ตารางเปรียบเทียบตัวเลือกหลัก

| เครื่องมือ | ประเภท | จุดเด่น | จุดอ่อน | เหมาะกับ |
|---|---|---|---|---|
| **node-postgres (pg)** | Driver ดิบ | เร็วที่สุด, ควบคุม SQL เต็มรูปแบบ, mature (ใช้มานานกว่า 10 ปี), community ใหญ่ | ต้องเขียน SQL เอง, ไม่มี type-safety อัตโนมัติ, ต้องจัดการ mapping เอง | ระบบที่ต้องการ performance สูงสุด, query ซับซ้อน, ทีมที่ถนัด SQL |
| **Prisma** | ORM รุ่นใหม่ | Type-safe เต็มรูปแบบ, DX (developer experience) ดีเยี่ยม, migration tool ในตัว, auto-completion | Generate client เพิ่ม build step, บาง query ซับซ้อนต้องพึ่ง raw SQL, overhead เล็กน้อย | ทีมที่ต้องการความเร็วในการพัฒนา, TypeScript-first project |
| **Drizzle ORM** | Query builder แบบ type-safe | เบา, ใกล้เคียง SQL มาก (SQL-like syntax), type-safe, ไม่ต้อง generate client แยก | Ecosystem ใหม่กว่า Prisma, เอกสารน้อยกว่า | ทีมที่ต้องการ type-safety แต่ไม่อยากเสีย control เหนือ SQL |
| **Sequelize** | ORM แบบดั้งเดิม | เก่าแก่, รองรับหลายฐานข้อมูล (MySQL, PostgreSQL, SQLite, MSSQL), community ใหญ่ | Type-safety ไม่แน่นเท่า Prisma/Drizzle, API ค่อนข้างเก่า | โปรเจกต์เดิมที่ใช้อยู่แล้ว หรือทีมที่คุ้นเคย ActiveRecord pattern |
| **TypeORM** | ORM แบบ Decorator | รองรับ Active Record และ Data Mapper pattern, ใช้ decorator สวยงาม | Bug และ breaking change ค่อนข้างบ่อยในอดีต, เอกสารสับสนบางจุด | โปรเจกต์ NestJS (ผูกกันแน่น) |
| **Kysely** | Type-safe query builder | Type inference ยอดเยี่ยม, เบา, ไม่มี magic | ต้องเขียน query เองเกือบทั้งหมด (ไม่มี ORM layer) | ทีมที่ต้องการ SQL builder ที่ type-safe แท้จริง |

### แนวทางการเลือกใช้ในบทนี้

บทนี้จะเน้น 2 เครื่องมือหลักที่ครองส่วนแบ่งการใช้งานมากที่สุดในวงการ Node.js + PostgreSQL:

1. **node-postgres (pg)** — Step 862–865 เพื่อให้เข้าใจกลไกพื้นฐานของการเชื่อมต่อ, query, transaction ที่ ORM ทุกตัวใช้อยู่เบื้องหลังจริงๆ
2. **Prisma** — Step 866–869 เพื่อให้เห็นว่า ORM ช่วยยกระดับ developer experience และ type-safety ได้อย่างไร โดยไม่ทิ้งความเข้าใจ SQL ที่เรียนมาตลอดหลักสูตร

> **หลักคิดสำคัญ**: ไม่ว่าจะใช้เครื่องมือใด ความเข้าใจ SQL, index, transaction, และ execution plan ที่เราเรียนมาตลอดหลักสูตรนี้ยังคงเป็นพื้นฐานที่ขาดไม่ได้ ORM เพียงช่วยลดงานซ้ำซาก แต่ไม่ได้ทดแทนความเข้าใจฐานข้อมูลเชิงลึก

### เกริ่น Drizzle และ Sequelize สั้นๆ

**Drizzle ORM** ตัวอย่าง syntax ที่ใกล้เคียง SQL มาก:

```typescript
// Drizzle ORM — ตัวอย่างสั้นๆ เพื่อให้เห็นภาพ (ไม่ได้ลงลึกในบทนี้)
import { pgTable, serial, varchar, numeric, integer } from 'drizzle-orm/pg-core';

export const products = pgTable('products', {
  productId: serial('product_id').primaryKey(),
  productName: varchar('product_name', { length: 150 }),
  unitPrice: numeric('unit_price', { precision: 10, scale: 2 }),
  stockQuantity: integer('stock_quantity'),
});

// Query
const cheapProducts = await db
  .select()
  .from(products)
  .where(lt(products.unitPrice, 1000));
```

**Sequelize** ตัวอย่าง syntax แบบ Active Record:

```javascript
// Sequelize — ตัวอย่างสั้นๆ เพื่อให้เห็นภาพ (ไม่ได้ลงลึกในบทนี้)
const Product = sequelize.define('Product', {
  productName: DataTypes.STRING(150),
  unitPrice: DataTypes.DECIMAL(10, 2),
  stockQuantity: DataTypes.INTEGER,
}, { tableName: 'products', timestamps: false });

const cheapProducts = await Product.findAll({
  where: { unitPrice: { [Op.lt]: 1000 } },
});
```

ทั้งสองตัวนี้มีแนวคิดคล้าย Prisma แต่มี syntax และปรัชญาต่างกัน ผู้เรียนที่เข้าใจ Prisma และ node-postgres จากบทนี้จะสามารถต่อยอดไปเรียนรู้ Drizzle หรือ Sequelize ได้ไม่ยาก เพราะหลักการเชื่อมต่อฐานข้อมูล, connection pooling, และ transaction เป็นแนวคิดร่วมกันทั้งหมด

---

## Step 862: node-postgres (pg) พื้นฐาน — การติดตั้ง, Pool vs Client, connection string

### การติดตั้ง

```bash
npm install pg
npm install --save-dev @types/pg   # สำหรับ TypeScript
```

`pg` เป็น driver ระดับ protocol ที่คุยกับ PostgreSQL โดยตรงผ่าน TCP socket ไม่มี ORM layer ห่อหุ้ม ทำให้เป็น dependency พื้นฐานที่ ORM หลายตัว (Knex, Sequelize บางส่วน) ก็ใช้อยู่เบื้องหลังเช่นกัน

### Connection string

รูปแบบมาตรฐานของ PostgreSQL connection string (เรียกอีกชื่อว่า `libpq` connection URI):

```
postgresql://<user>:<password>@<host>:<port>/<database>?<options>
```

ตัวอย่าง:

```
postgresql://app_user:secretpassword@localhost:5432/ecommerce_db?sslmode=disable
```

ใน production ที่เชื่อมต่อผ่าน SSL (เช่น managed database บน cloud):

```
postgresql://app_user:secretpassword@db.example.com:5432/ecommerce_db?sslmode=require
```

### Client vs Pool: ความแตกต่างที่สำคัญ

`node-postgres` มี 2 คลาสหลักสำหรับเชื่อมต่อ:

| | `Client` | `Pool` |
|---|---|---|
| ความหมาย | connection เดี่ยว 1 เส้น | กลุ่มของ connection ที่ใช้ซ้ำได้ (connection pool) |
| การใช้งานทั่วไป | script สั้นๆ, migration, งานที่ต้องการ session state เฉพาะ (เช่น `SET search_path`) | web application/API ที่รับ request พร้อมกันหลายตัว |
| ต้อง `connect()`/`end()` เอง | ใช่ | ไม่จำเป็น (pool จัดการให้) |
| Concurrency | รองรับ query เดียวต่อครั้ง (ต้องรอ query ก่อนหน้าเสร็จ) | รองรับหลาย query พร้อมกันผ่านหลาย connection ใน pool |
| คำแนะนำ | ใช้เมื่อจำเป็นต้องคุมเส้น connection เดียว เช่น transaction | **ใช้เป็นค่าเริ่มต้นสำหรับ web application แทบทุกกรณี** |

### การใช้ Client (สำหรับกรณีพิเศษ)

```javascript
// client-example.js
const { Client } = require('pg');

async function main() {
  const client = new Client({
    connectionString: process.env.DATABASE_URL,
  });

  await client.connect();

  const result = await client.query('SELECT product_name, unit_price FROM products LIMIT 5');
  console.log(result.rows);

  await client.end(); // ต้องปิด connection เองเสมอ
}

main().catch((err) => {
  console.error('เกิดข้อผิดพลาด:', err);
  process.exit(1);
});
```

### การใช้ Pool (แนะนำสำหรับ production)

```javascript
// db/pool.js
const { Pool } = require('pg');

const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  max: 20,                     // จำนวน connection สูงสุดใน pool
  idleTimeoutMillis: 30000,    // ปิด connection ที่ไม่ได้ใช้เกิน 30 วินาที
  connectionTimeoutMillis: 5000, // รอ connection ใหม่ได้สูงสุด 5 วินาที ก่อน error
});

// ดักจับ error ที่เกิดกับ connection ที่ idle อยู่ใน pool
pool.on('error', (err) => {
  console.error('Unexpected error on idle client', err);
  process.exit(-1);
});

module.exports = pool;
```

การใช้งาน pool ไม่ต้องเรียก `connect()`/`end()` ทุกครั้ง เพราะ pool จะจัดการเปิด/ปิด connection ให้อัตโนมัติ:

```javascript
// index.js
const pool = require('./db/pool');

async function listProducts() {
  const result = await pool.query('SELECT product_name, unit_price FROM products ORDER BY unit_price');
  return result.rows;
}

listProducts()
  .then((rows) => console.table(rows))
  .catch((err) => console.error(err));
```

### ตั้งค่าผ่าน environment variables โดยไม่ระบุ config

`pg` รองรับการอ่านค่าจาก environment variable มาตรฐานของ PostgreSQL โดยอัตโนมัติ (`PGHOST`, `PGPORT`, `PGUSER`, `PGPASSWORD`, `PGDATABASE`) ถ้าไม่ส่ง config ใดๆ เข้าไปเลย:

```javascript
require('dotenv').config();
const { Pool } = require('pg');

// ถ้าตั้ง PGHOST, PGUSER, PGPASSWORD, PGDATABASE, PGPORT ใน .env ไว้แล้ว
// สามารถสร้าง pool แบบไม่ต้องส่ง config เลยก็ได้
const pool = new Pool();
```

### SSL configuration สำหรับ managed database

```javascript
const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  ssl: {
    rejectUnauthorized: true,   // ตรวจสอบ certificate ของเซิร์ฟเวอร์ (แนะนำใน production)
    // ca: fs.readFileSync('./certs/ca-certificate.crt').toString(), // ถ้าต้องใช้ custom CA
  },
});
```

> **ข้อควรระวัง**: อย่าตั้ง `rejectUnauthorized: false` ใน production เพราะเปิดช่องให้เกิด man-in-the-middle attack ควรใช้ certificate ที่ถูกต้องเสมอ

### สรุป pattern การจัดโครงสร้างโปรเจกต์

```
ecommerce-api/
├── .env
├── .gitignore
├── package.json
├── db/
│   └── pool.js          # สร้าง pool ที่ใช้ร่วมกันทั้งแอป (singleton)
├── routes/
│   ├── products.js
│   └── orders.js
└── index.js              # entry point
```

การสร้าง pool เป็น **singleton module** (สร้างครั้งเดียวแล้ว export ใช้ซ้ำ) เป็นแนวทางที่ถูกต้อง เพราะถ้าสร้าง `Pool` ใหม่ทุกครั้งที่มี request เข้ามา จะทำให้เปิด connection เกินความจำเป็นและ database อาจปฏิเสธการเชื่อมต่อเมื่อถึง `max_connections`

---

## Step 863: node-postgres — Query พื้นฐานพร้อม parameterized query

### รูปแบบ query พื้นฐาน

```javascript
const pool = require('./db/pool');

async function getAllProducts() {
  const result = await pool.query('SELECT * FROM products ORDER BY product_id');
  return result.rows; // array ของ object แต่ละแถว
}
```

Object ที่ได้จาก `pool.query()` มีโครงสร้างสำคัญ:

```javascript
{
  rows: [ { product_id: 1, product_name: 'คีย์บอร์ดกลไก', unit_price: '1590.00', stock_quantity: 25 }, ... ],
  rowCount: 4,
  command: 'SELECT',
  fields: [ /* metadata ของแต่ละ column เช่น ชื่อ, data type */ ],
}
```

> **สังเกต**: `unit_price` ที่เป็น `NUMERIC` ใน PostgreSQL จะถูกแปลงมาเป็น **string** ใน JavaScript (เช่น `'1590.00'`) ไม่ใช่ number โดยอัตโนมัติ เพราะ `NUMERIC` มีความแม่นยำสูงกว่าที่ JavaScript `number` (double-precision float) จะรองรับได้อย่างปลอดภัย หากต้องการแปลงเป็น number ต้องทำเองด้วย `parseFloat()` หรือใช้ library เช่น `pg-types` ปรับแต่ง type parser

### อันตรายของการต่อ string SQL โดยตรง (SQL Injection)

```javascript
// ❌ ห้ามทำแบบนี้เด็ดขาด — เสี่ยง SQL Injection
async function getProductByNameUnsafe(name) {
  const result = await pool.query(`SELECT * FROM products WHERE product_name = '${name}'`);
  return result.rows;
}

// ถ้าผู้ใช้ส่ง name = "x'; DROP TABLE products; --"
// query ที่ถูกสร้างขึ้นจริงจะกลายเป็น:
// SELECT * FROM products WHERE product_name = 'x'; DROP TABLE products; --'
// ซึ่งอาจลบตาราง products ทิ้งได้ทันที!
```

### วิธีที่ถูกต้อง: parameterized query ด้วย `$1, $2, ...`

```javascript
// ✅ ใช้ parameterized query เสมอ
async function getProductByName(name) {
  const result = await pool.query(
    'SELECT * FROM products WHERE product_name = $1',
    [name]
  );
  return result.rows;
}
```

`node-postgres` จะส่ง SQL text กับ parameter values แยกกันไปยัง PostgreSQL server ผ่าน **extended query protocol** ทำให้ PostgreSQL แยกแยะ "คำสั่ง SQL" กับ "ข้อมูล" ออกจากกันอย่างชัดเจนในระดับ protocol ไม่ใช่แค่การ escape string ข้อมูลที่ส่งเข้าไปจึงไม่มีทางถูกตีความเป็นส่วนหนึ่งของคำสั่ง SQL ได้เลย ไม่ว่าจะมีอักขระพิเศษอะไรอยู่ในนั้น

### ตัวอย่างหลาย parameter

```javascript
async function findProductsInPriceRange(minPrice, maxPrice) {
  const result = await pool.query(
    `SELECT product_id, product_name, unit_price
     FROM products
     WHERE unit_price BETWEEN $1 AND $2
     ORDER BY unit_price`,
    [minPrice, maxPrice]
  );
  return result.rows;
}

findProductsInPriceRange(500, 2000).then(console.table);
```

### INSERT พร้อม RETURNING

```javascript
async function createProduct({ productName, unitPrice, stockQuantity }) {
  const result = await pool.query(
    `INSERT INTO products (product_name, unit_price, stock_quantity)
     VALUES ($1, $2, $3)
     RETURNING product_id, product_name, unit_price, stock_quantity`,
    [productName, unitPrice, stockQuantity]
  );
  return result.rows[0]; // ได้ row ที่เพิ่งสร้างกลับมาทันที
}

createProduct({ productName: 'แผ่นรองเมาส์', unitPrice: 190.00, stockQuantity: 100 })
  .then((product) => console.log('สร้างสินค้าใหม่:', product));
```

### UPDATE ด้วย parameterized query

```javascript
async function updateStock(productId, newQuantity) {
  const result = await pool.query(
    `UPDATE products
     SET stock_quantity = $1
     WHERE product_id = $2
     RETURNING *`,
    [newQuantity, productId]
  );

  if (result.rowCount === 0) {
    throw new Error(`ไม่พบสินค้า product_id = ${productId}`);
  }
  return result.rows[0];
}
```

### DELETE

```javascript
async function deleteProduct(productId) {
  const result = await pool.query(
    'DELETE FROM products WHERE product_id = $1',
    [productId]
  );
  return result.rowCount; // จำนวนแถวที่ถูกลบ (0 หรือ 1)
}
```

### การใช้ named query object (เพื่อ readability และ prepared statement)

`pool.query()` รองรับการส่ง object แทน string+array ซึ่งช่วยให้โค้ดอ่านง่ายขึ้น และยังสามารถตั้งชื่อ (`name`) เพื่อให้ PostgreSQL cache execution plan ของ query นั้นไว้ (prepared statement):

```javascript
async function getProductById(productId) {
  const query = {
    name: 'get-product-by-id',   // ตั้งชื่อ prepared statement (ถูก cache ไว้ใน session)
    text: 'SELECT * FROM products WHERE product_id = $1',
    values: [productId],
  };
  const result = await pool.query(query);
  return result.rows[0] || null;
}
```

การตั้ง `name` มีประโยชน์เมื่อ query เดิมถูกเรียกซ้ำๆ บ่อยมากในระบบ เพราะ PostgreSQL จะไม่ต้อง parse และ plan ใหม่ทุกครั้ง แต่ต้องระวังว่า prepared statement จะผูกกับ connection เดียวเท่านั้น — ถ้าใช้กับ `Pool` ซึ่งสลับ connection ไปมา ควรตรวจสอบพฤติกรรมนี้ให้ดีในระบบที่มี connection pooler ภายนอก เช่น PgBouncer โหมด transaction pooling (ซึ่งไม่รองรับ prepared statement ข้าม transaction)

### ตัวอย่าง query ที่ join หลายตาราง

```javascript
async function getOrdersWithCustomerInfo() {
  const result = await pool.query(
    `SELECT
       o.order_id,
       o.order_date,
       o.status,
       c.first_name,
       c.email
     FROM orders o
     JOIN customers c ON c.customer_id = o.customer_id
     ORDER BY o.order_date DESC`
  );
  return result.rows;
}
```

### ข้อควรจำ: parameterized query ไม่ใช้ได้กับทุกส่วนของ SQL

`$1, $2, ...` ใช้แทนค่า (value) เท่านั้น **ไม่สามารถใช้แทนชื่อ table, column, หรือ keyword ของ SQL ได้** เช่น `ORDER BY $1` หรือ `FROM $1` จะไม่ทำงานตามที่คาดหวัง หากต้องการสร้าง dynamic column/table name ต้อง whitelist ค่าที่รับเข้ามาด้วยตนเองก่อนนำไปต่อ string:

```javascript
// การจัดการ dynamic ORDER BY อย่างปลอดภัย
const ALLOWED_SORT_COLUMNS = ['product_name', 'unit_price', 'stock_quantity'];

async function listProductsSorted(sortColumn = 'product_id', direction = 'ASC') {
  // whitelist ชื่อ column ก่อนนำไปต่อ string เพื่อป้องกัน injection
  if (!ALLOWED_SORT_COLUMNS.includes(sortColumn)) {
    sortColumn = 'product_id';
  }
  const safeDirection = direction.toUpperCase() === 'DESC' ? 'DESC' : 'ASC';

  // ปลอดภัยเพราะ sortColumn และ safeDirection ผ่านการ whitelist แล้ว ไม่ใช่ค่าจาก user ตรงๆ
  const result = await pool.query(
    `SELECT * FROM products ORDER BY ${sortColumn} ${safeDirection}`
  );
  return result.rows;
}
```

---

## Step 864: node-postgres — Transaction ด้วย BEGIN/COMMIT/ROLLBACK

### ทำไม `pool.query()` ใช้ทำ transaction ไม่ได้โดยตรง

`Pool` จะสลับ connection ให้กับแต่ละ `pool.query()` โดยอัตโนมัติ (อาจได้ connection คนละเส้นในแต่ละครั้ง) ดังนั้นถ้าเรียก `pool.query('BEGIN')` แล้วตามด้วย `pool.query('INSERT ...')` ทั้งสองคำสั่งอาจไปลงคนละ connection กัน ทำให้ transaction ไม่ทำงานตามที่คาดหวัง

วิธีที่ถูกต้องคือต้อง **ขอ connection เดียวออกมาจาก pool** ด้วย `pool.connect()` แล้วใช้ connection (client) เดียวนั้นรัน `BEGIN`, คำสั่งต่างๆ, และ `COMMIT`/`ROLLBACK` ทั้งหมด

### รูปแบบ transaction ที่ถูกต้อง

```javascript
const pool = require('./db/pool');

async function transferStock(fromProductId, toProductId, quantity) {
  const client = await pool.connect(); // ขอ connection เดี่ยวออกมาจาก pool

  try {
    await client.query('BEGIN');

    // ลดสต็อกสินค้าต้นทาง
    const decreaseResult = await client.query(
      `UPDATE products
       SET stock_quantity = stock_quantity - $1
       WHERE product_id = $2 AND stock_quantity >= $1
       RETURNING stock_quantity`,
      [quantity, fromProductId]
    );

    if (decreaseResult.rowCount === 0) {
      throw new Error('สต็อกสินค้าต้นทางไม่เพียงพอ หรือไม่พบสินค้า');
    }

    // เพิ่มสต็อกสินค้าปลายทาง
    await client.query(
      `UPDATE products
       SET stock_quantity = stock_quantity + $1
       WHERE product_id = $2`,
      [quantity, toProductId]
    );

    await client.query('COMMIT');
    return { success: true };

  } catch (err) {
    await client.query('ROLLBACK'); // ย้อนกลับทุกอย่างถ้ามีข้อผิดพลาดใดๆ เกิดขึ้น
    throw err; // ส่ง error ต่อให้ผู้เรียกจัดการ

  } finally {
    client.release(); // คืน connection กลับสู่ pool เสมอ ไม่ว่าจะสำเร็จหรือล้มเหลว
  }
}
```

### จุดที่ต้องระวังในรูปแบบ try/catch/finally

1. **`client.release()` ต้องอยู่ใน `finally`** เสมอ เพื่อรับประกันว่า connection จะถูกคืนกลับสู่ pool ไม่ว่า transaction จะสำเร็จหรือล้มเหลว หากลืม `release()` connection จะค้างอยู่และในที่สุด pool จะเต็ม (connection leak)
2. **`ROLLBACK` ต้องอยู่ใน `catch`** เพื่อยกเลิกการเปลี่ยนแปลงทั้งหมดที่ยังไม่ commit เมื่อเกิด error
3. ห้ามลืม `throw err` ใน catch หลัง rollback มิฉะนั้น error จะถูกกลืนหายไปเงียบๆ และผู้เรียกใช้ฟังก์ชันจะไม่รู้ว่า transaction ล้มเหลว

### ตัวอย่าง transaction ที่ซับซ้อนขึ้น: สร้างคำสั่งซื้อพร้อมตัดสต็อก

```javascript
async function createOrderWithStockDeduction(customerId, items) {
  // items = [{ productId: 1, quantity: 2 }, { productId: 3, quantity: 1 }]
  const client = await pool.connect();

  try {
    await client.query('BEGIN');

    // ตรวจสอบว่า customer มีอยู่จริง
    const customerCheck = await client.query(
      'SELECT customer_id FROM customers WHERE customer_id = $1',
      [customerId]
    );
    if (customerCheck.rowCount === 0) {
      throw new Error(`ไม่พบลูกค้า customer_id = ${customerId}`);
    }

    // สร้าง order
    const orderResult = await client.query(
      `INSERT INTO orders (customer_id, status)
       VALUES ($1, 'pending')
       RETURNING order_id`,
      [customerId]
    );
    const orderId = orderResult.rows[0].order_id;

    // ตัดสต็อกทีละสินค้า
    for (const item of items) {
      const stockResult = await client.query(
        `UPDATE products
         SET stock_quantity = stock_quantity - $1
         WHERE product_id = $2 AND stock_quantity >= $1
         RETURNING product_id, stock_quantity`,
        [item.quantity, item.productId]
      );

      if (stockResult.rowCount === 0) {
        // สต็อกไม่พอ หรือไม่พบสินค้า -> ยกเลิกทั้ง transaction
        throw new Error(`สต็อกสินค้า product_id=${item.productId} ไม่เพียงพอ`);
      }
    }

    await client.query('COMMIT');
    return { orderId, status: 'pending' };

  } catch (err) {
    await client.query('ROLLBACK');
    throw err;

  } finally {
    client.release();
  }
}
```

ในตัวอย่างนี้ ถ้าสินค้าชิ้นที่ 2 มีสต็อกไม่พอ ทั้ง order และการตัดสต็อกของสินค้าชิ้นที่ 1 จะถูก rollback ทั้งหมด — นี่คือคุณสมบัติ **atomicity** ของ transaction ที่เรียนมาใน Part เกี่ยวกับ ACID

### Savepoint สำหรับ transaction ที่ซับซ้อน

หากต้องการ rollback เพียงบางส่วนของ transaction โดยไม่ยกเลิกทั้งหมด สามารถใช้ `SAVEPOINT` ภายใน transaction เดียวกันได้ เช่น `await client.query('SAVEPOINT before_discount')` แล้วถ้าขั้นตอนถัดไปล้มเหลวก็สั่ง `await client.query('ROLLBACK TO SAVEPOINT before_discount')` เพื่อย้อนกลับเฉพาะส่วนนั้น โดยที่งานส่วนก่อนหน้า savepoint (เช่น การสร้าง order) ยังคงอยู่และรอ `COMMIT` ตามปกติเมื่อ transaction จบ

### Helper function สำหรับหุ้ม transaction pattern

เพื่อลดการเขียนโค้ด boilerplate ซ้ำๆ นิยมสร้างฟังก์ชัน helper ที่หุ้ม pattern `BEGIN/COMMIT/ROLLBACK/release` ไว้:

```javascript
// db/withTransaction.js
const pool = require('./pool');

async function withTransaction(callback) {
  const client = await pool.connect();
  try {
    await client.query('BEGIN');
    const result = await callback(client);
    await client.query('COMMIT');
    return result;
  } catch (err) {
    await client.query('ROLLBACK');
    throw err;
  } finally {
    client.release();
  }
}

module.exports = withTransaction;
```

ใช้งานได้กระชับขึ้นมาก:

```javascript
const withTransaction = require('./db/withTransaction');

async function transferStock(fromId, toId, qty) {
  return withTransaction(async (client) => {
    await client.query(
      'UPDATE products SET stock_quantity = stock_quantity - $1 WHERE product_id = $2',
      [qty, fromId]
    );
    await client.query(
      'UPDATE products SET stock_quantity = stock_quantity + $1 WHERE product_id = $2',
      [qty, toId]
    );
    return { success: true };
  });
}
```

---

## Step 865: node-postgres ขั้นสูง — async/await, connection pool management, error handling

### Pattern async/await ที่ถูกต้อง

เนื่องจาก `pool.query()` และ `client.query()` คืนค่าเป็น `Promise` เสมอ การใช้ `async/await` เป็นวิธีที่อ่านง่ายที่สุด แต่ต้องระวังจุดผิดพลาดที่พบบ่อย:

```javascript
// ❌ ผิด: ไม่ await ทำให้ error หลุดออกไปแบบ unhandled promise rejection
function getProduct(id) {
  pool.query('SELECT * FROM products WHERE product_id = $1', [id])
    .then((result) => result.rows[0]);
  // ฟังก์ชันนี้ return undefined เสมอ เพราะไม่ได้ return promise ออกไป!
}

// ✅ ถูก: return promise ออกไปให้ผู้เรียกใช้ await ได้
async function getProduct(id) {
  const result = await pool.query('SELECT * FROM products WHERE product_id = $1', [id]);
  return result.rows[0] || null;
}
```

### การจัดการ error อย่างเป็นระบบ

```javascript
async function getProduct(id) {
  try {
    const result = await pool.query(
      'SELECT * FROM products WHERE product_id = $1',
      [id]
    );
    return result.rows[0] || null;
  } catch (err) {
    // pg error object มี field พิเศษที่มีประโยชน์มาก
    console.error('Database error:', {
      message: err.message,
      code: err.code,        // PostgreSQL error code เช่น '23505' = unique_violation
      detail: err.detail,
      table: err.table,
      constraint: err.constraint,
    });
    throw err; // ส่งต่อให้ชั้นที่สูงกว่าตัดสินใจว่าจะทำอย่างไรต่อ
  }
}
```

### PostgreSQL error codes ที่พบบ่อย

| Error code | ความหมาย | ตัวอย่างสถานการณ์ |
|---|---|---|
| `23505` | `unique_violation` | ใส่ email ซ้ำในตาราง customers ที่มี UNIQUE constraint |
| `23503` | `foreign_key_violation` | สร้าง order ด้วย customer_id ที่ไม่มีอยู่จริง |
| `23502` | `not_null_violation` | ไม่ใส่ค่าให้ column ที่เป็น NOT NULL |
| `23514` | `check_violation` | ค่าที่ใส่ไม่ผ่าน CHECK constraint |
| `42P01` | `undefined_table` | อ้างถึงตารางที่ไม่มีอยู่ (มักเกิดจาก typo หรือ schema ไม่ตรง) |
| `28P01` | `invalid_password` | รหัสผ่านผิด |
| `08006` / `ECONNREFUSED` | connection failure | database ไม่ได้เปิด หรือ host/port ผิด |
| `57014` | `query_canceled` | query ถูกยกเลิกเพราะเกิน `statement_timeout` |

### แปลง error code ให้เป็นข้อความที่เข้าใจง่ายสำหรับผู้ใช้

```javascript
function toUserFriendlyError(err) {
  switch (err.code) {
    case '23505':
      return { statusCode: 409, message: `ข้อมูลซ้ำ: ${err.detail || 'มีข้อมูลนี้อยู่แล้ว'}` };
    case '23503':
      return { statusCode: 400, message: `ข้อมูลอ้างอิงไม่ถูกต้อง: ${err.detail || ''}` };
    case '23502':
      return { statusCode: 400, message: `กรุณากรอกข้อมูลให้ครบถ้วน` };
    case '23514':
      return { statusCode: 400, message: `ข้อมูลไม่ผ่านเงื่อนไขที่กำหนด` };
    default:
      return { statusCode: 500, message: 'เกิดข้อผิดพลาดภายในระบบ' };
  }
}

// ตัวอย่างการใช้งานใน Express route
app.post('/customers', async (req, res) => {
  try {
    const result = await pool.query(
      'INSERT INTO customers (first_name, email) VALUES ($1, $2) RETURNING *',
      [req.body.firstName, req.body.email]
    );
    res.status(201).json(result.rows[0]);
  } catch (err) {
    const { statusCode, message } = toUserFriendlyError(err);
    res.status(statusCode).json({ error: message });
  }
});
```

### Connection pool: การตั้งค่าที่เหมาะสมกับ production

```javascript
const pool = new Pool({
  connectionString: process.env.DATABASE_URL,

  // จำนวน connection สูงสุด — คำนวณจาก:
  // (จำนวน instance ของแอป) x (max ต่อ instance) ต้อง <= max_connections ของ PostgreSQL
  max: 20,

  // connection ที่ idle นานเกินนี้จะถูกปิดทิ้ง เพื่อคืนทรัพยากรให้ database
  idleTimeoutMillis: 30000,

  // เวลารอสูงสุดในการขอ connection ใหม่จาก pool ก่อนที่จะ throw error
  connectionTimeoutMillis: 5000,

  // เวลาสูงสุดที่ query หนึ่งจะรันได้ก่อนถูกยกเลิก (ป้องกัน query ค้างตลอดไป)
  statement_timeout: 10000,

  // ปิด application ทันทีถ้า query ทั้งหมด (รวม idle in transaction) ใช้เวลานานเกินนี้
  query_timeout: 10000,
});
```

### ตรวจสอบสถานะของ pool (สำหรับ monitoring / health check)

```javascript
function getPoolStats() {
  return {
    totalCount: pool.totalCount,   // จำนวน connection ทั้งหมดใน pool
    idleCount: pool.idleCount,     // จำนวน connection ที่ว่างอยู่
    waitingCount: pool.waitingCount, // จำนวน request ที่กำลังรอ connection ว่าง
  };
}

// ใช้ทำ health check endpoint
app.get('/health/db', async (req, res) => {
  try {
    await pool.query('SELECT 1');
    res.json({ status: 'ok', pool: getPoolStats() });
  } catch (err) {
    res.status(503).json({ status: 'unavailable', error: err.message });
  }
});
```

> **สัญญาณเตือน**: ถ้า `waitingCount` สูงอย่างต่อเนื่อง แปลว่า pool มีขนาดเล็กเกินไปเมื่อเทียบกับ load ควรพิจารณาเพิ่ม `max` หรือหาสาเหตุว่ามี query ที่ค้างนานผิดปกติ (long-running query ที่ไม่คืน connection)

### Graceful shutdown

การปิดแอปพลิเคชันอย่างเหมาะสมต้องปิด pool ให้เรียบร้อยก่อน เพื่อให้ query ที่ค้างอยู่ทำงานจบและ connection ถูกปิดอย่างถูกต้อง:

```javascript
// index.js
const server = app.listen(3000, () => console.log('Server running on port 3000'));

async function gracefulShutdown(signal) {
  console.log(`ได้รับสัญญาณ ${signal}, กำลังปิดระบบ...`);
  server.close(async () => {
    console.log('HTTP server ปิดแล้ว');
    await pool.end(); // รอ query ที่ค้างอยู่ทั้งหมดจบก่อน แล้วปิด connection ทั้งหมดใน pool
    console.log('Database pool ปิดแล้ว');
    process.exit(0);
  });
}

process.on('SIGTERM', () => gracefulShutdown('SIGTERM'));
process.on('SIGINT', () => gracefulShutdown('SIGINT'));
```

### รวม best practices ทั้งหมดในโมดูลเดียว (TypeScript)

```typescript
// db/pool.ts
import { Pool, PoolClient } from 'pg';
import dotenv from 'dotenv';

dotenv.config();

export const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  max: 20,
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 5000,
  statement_timeout: 10000,
});

pool.on('error', (err: Error) => {
  console.error('Unexpected error on idle client', err);
});

export async function withTransaction<T>(
  callback: (client: PoolClient) => Promise<T>
): Promise<T> {
  const client = await pool.connect();
  try {
    await client.query('BEGIN');
    const result = await callback(client);
    await client.query('COMMIT');
    return result;
  } catch (err) {
    await client.query('ROLLBACK');
    throw err;
  } finally {
    client.release();
  }
}

export async function closePool(): Promise<void> {
  await pool.end();
}
```

```typescript
// services/productService.ts
import { pool } from '../db/pool';

export interface Product {
  product_id: number;
  product_name: string;
  unit_price: string; // NUMERIC มาเป็น string เสมอจาก pg
  stock_quantity: number;
}

export async function getProductById(id: number): Promise<Product | null> {
  const result = await pool.query<Product>(
    'SELECT * FROM products WHERE product_id = $1',
    [id]
  );
  return result.rows[0] ?? null;
}
```

การใช้ generic type `pool.query<Product>()` ช่วยให้ TypeScript ทราบ shape ของ `result.rows` แต่ต้องระวังว่านี่เป็นเพียง **type assertion ที่นักพัฒนาประกาศเอง** ไม่มีการตรวจสอบจริงว่า column ที่ query กลับมาตรงกับ interface หรือไม่ — นี่คือข้อจำกัดสำคัญของ raw SQL driver ที่ ORM อย่าง Prisma แก้ปัญหานี้ได้ด้วยการ generate type จาก schema จริง ซึ่งเราจะไปดูกันในหัวข้อถัดไป

---

## Step 866: Prisma ORM แนะนำตัว — schema.prisma, generate client, type-safety

### Prisma คืออะไร

Prisma เป็น ORM รุ่นใหม่สำหรับ Node.js/TypeScript ที่มีปรัชญาแตกต่างจาก ORM ดั้งเดิม (เช่น Sequelize, TypeORM) ตรงที่ Prisma ใช้ **schema file เฉพาะของตัวเอง** (`schema.prisma`) เป็นแหล่งความจริงเดียว (single source of truth) แล้ว **generate โค้ด TypeScript ที่ type-safe เต็มรูปแบบ** ขึ้นมาจาก schema นั้นโดยอัตโนมัติ

ส่วนประกอบหลักของ Prisma มี 3 ส่วน:

1. **Prisma Schema** (`schema.prisma`) — ไฟล์ที่นิยาม data model, datasource, และ generator
2. **Prisma Client** — library ที่ auto-generate ขึ้นมาจาก schema เพื่อใช้ query ฐานข้อมูลแบบ type-safe
3. **Prisma Migrate** — เครื่องมือจัดการ schema migration (จะพูดถึงใน Step 868)

### การติดตั้ง

```bash
npm install prisma --save-dev
npm install @prisma/client
npx prisma init
```

คำสั่ง `prisma init` จะสร้างโครงสร้างไฟล์ดังนี้:

```
ecommerce-api/
├── prisma/
│   └── schema.prisma
├── .env
```

### เขียน schema.prisma สำหรับ e-commerce schema ของเรา

```prisma
// prisma/schema.prisma

generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model Product {
  productId     Int      @id @default(autoincrement()) @map("product_id")
  productName   String?  @map("product_name") @db.VarChar(150)
  unitPrice     Decimal? @map("unit_price") @db.Decimal(10, 2)
  stockQuantity Int?     @map("stock_quantity")

  @@map("products")
}

model Customer {
  customerId Int     @id @default(autoincrement()) @map("customer_id")
  firstName  String? @map("first_name") @db.VarChar(60)
  email      String? @db.VarChar(150)

  orders     Order[]

  @@map("customers")
}

model Order {
  orderId    Int       @id @default(autoincrement()) @map("order_id")
  customerId Int?      @map("customer_id")
  orderDate  DateTime? @default(now()) @map("order_date") @db.Timestamptz(6)
  status     String?   @db.VarChar(20)

  customer   Customer? @relation(fields: [customerId], references: [customerId])

  @@map("orders")
}
```

### อธิบายส่วนประกอบสำคัญของ schema

- **`generator client`** — บอก Prisma ว่าให้ generate Prisma Client สำหรับ JavaScript/TypeScript
- **`datasource db`** — ระบุว่าฐานข้อมูลเป็น PostgreSQL และดึง connection string จาก environment variable `DATABASE_URL`
- **`@id`** — กำหนดว่า field นี้เป็น primary key
- **`@default(autoincrement())`** — เทียบเท่ากับ `SERIAL` ใน PostgreSQL
- **`@map("product_id")`** — บอกว่าชื่อ field ใน Prisma (`productId`, camelCase ตามธรรมเนียม JavaScript) แมปกับชื่อ column จริงในฐานข้อมูล (`product_id`, snake_case ตามธรรมเนียม PostgreSQL) — สิ่งนี้ช่วยให้เราเขียน schema เดิมในฐานข้อมูลไว้ตามธรรมเนียม SQL แต่ใช้ชื่อ JavaScript-friendly ในโค้ด
- **`@@map("products")`** — แมปชื่อ model (`Product`, PascalCase เอกพจน์ตามธรรมเนียม Prisma) กับชื่อ table จริง (`products`, พหูพจน์)
- **`@db.VarChar(150)`** — native type attribute ที่ระบุชนิดข้อมูลจริงใน PostgreSQL อย่างละเอียด
- **`orders Order[]`** ใน model `Customer` และ **`customer Customer?`** ใน model `Order` — นิยาม relation แบบ one-to-many ระหว่างสองตาราง

### Introspection: ให้ Prisma อ่าน schema ที่มีอยู่แล้วในฐานข้อมูล

ถ้าฐานข้อมูล e-commerce ของเรามีอยู่แล้ว (สร้างด้วย SQL ตามที่ให้ไว้ตอนต้นบท) เราไม่จำเป็นต้องเขียน `schema.prisma` เองทั้งหมด สามารถให้ Prisma "ส่องกล้อง" เข้าไปอ่านโครงสร้างจริงแล้ว generate schema ให้อัตโนมัติ:

```bash
npx prisma db pull
```

คำสั่งนี้จะเชื่อมต่อไปยัง `DATABASE_URL` แล้วอ่านตาราง, column, constraint, foreign key ทั้งหมด มาเขียนเป็น `schema.prisma` ให้โดยอัตโนมัติ เหมาะมากสำหรับการนำ Prisma มาใช้กับฐานข้อมูลเดิมที่มีอยู่แล้ว (brownfield project)

### การ generate Prisma Client

หลังจากแก้ไข `schema.prisma` เสร็จ (ไม่ว่าจะเขียนเองหรือ pull มา) ต้อง generate client ทุกครั้ง:

```bash
npx prisma generate
```

คำสั่งนี้จะสร้างโค้ด TypeScript ขึ้นมาใน `node_modules/@prisma/client` ที่มี type ตรงกับ schema ทุกประการ — นี่คือหัวใจของ type-safety ใน Prisma: **ทุกครั้งที่ schema เปลี่ยน แล้วรัน `generate` ใหม่ TypeScript compiler จะรู้ทันทีถ้าโค้ดที่ query ฐานข้อมูลไม่ตรงกับ schema ปัจจุบัน**

### การสร้าง PrismaClient instance

```typescript
// db/prisma.ts
import { PrismaClient } from '@prisma/client';

// สร้าง singleton instance เพื่อไม่ให้เปิด connection pool ซ้ำซ้อน
const prisma = new PrismaClient({
  log: ['query', 'error', 'warn'], // log SQL ที่ Prisma สร้างขึ้น มีประโยชน์มากตอน debug
});

export default prisma;
```

> **ข้อควรระวังสำคัญ**: ในสภาพแวดล้อมที่มี hot-reload (เช่น dev server ที่ restart บ่อย หรือ serverless function) การสร้าง `new PrismaClient()` ซ้ำๆ จะทำให้เกิด connection pool จำนวนมากเกินความจำเป็น ควรใช้ pattern singleton ที่เก็บ instance ไว้ใน global object ดังตัวอย่างด้านล่าง

```typescript
// db/prisma.ts — pattern ที่ปลอดภัยสำหรับ dev server ที่ hot-reload บ่อย
import { PrismaClient } from '@prisma/client';

const globalForPrisma = globalThis as unknown as { prisma: PrismaClient };

const prisma = globalForPrisma.prisma || new PrismaClient();

if (process.env.NODE_ENV !== 'production') {
  globalForPrisma.prisma = prisma;
}

export default prisma;
```

### ทดสอบการเชื่อมต่อด้วย type-safety ที่เห็นได้ทันที

```typescript
import prisma from './db/prisma';

async function main() {
  const products = await prisma.product.findMany();
  //    ^ TypeScript รู้ทันทีว่า products มี type Product[]
  //      แต่ละ product มี .productId (number), .productName (string | null), ...

  console.log(products);

  // ตัวอย่าง error ที่ TypeScript จะจับได้ทันทีตอน compile
  // console.log(products[0].productTitle); // ❌ Error: property 'productTitle' does not exist on type 'Product'
}

main()
  .catch(console.error)
  .finally(() => prisma.$disconnect());
```

### เปรียบเทียบ type-safety: pg (raw) vs Prisma

| | node-postgres (pg) | Prisma |
|---|---|---|
| Type ของผลลัพธ์ query | `any` โดยปริยาย ต้องประกาศ interface เอง (ไม่มีการตรวจสอบจริง) | Generate type อัตโนมัติจาก schema จริง ตรงกันเสมอ |
| Autocomplete ตอนเขียนโค้ด | ไม่มี (เว้นแต่เขียน type เอง) | มีเต็มรูปแบบ รวมถึงชื่อ field, relation, filter operator |
| ตรวจจับ error เมื่อ schema เปลี่ยน | ไม่ตรวจจับอัตโนมัติ ต้องรอ runtime error | TypeScript compiler แจ้ง error ทันทีถ้าโค้ดไม่ตรงกับ schema (หลัง `prisma generate`) |
| ความเร็วในการ query | เร็วที่สุด (ไม่มี overhead ของ ORM) | มี overhead เล็กน้อยจากการแปลง query เป็น SQL และ map ผลลัพธ์กลับ |

### สรุป workflow การเริ่มต้นใช้ Prisma

```
1. npx prisma init                 → สร้างโครงสร้างโปรเจกต์
2. เขียน/pull schema.prisma        → นิยาม data model
3. npx prisma generate             → generate type-safe client
4. import PrismaClient มาใช้งาน    → เริ่ม query แบบ type-safe
```

---

## Step 867: Prisma — CRUD operations พื้นฐาน เทียบเคียงกับ raw SQL

หัวข้อนี้จะแสดงโค้ด Prisma คู่กับ raw SQL ที่เทียบเท่ากัน เพื่อให้เห็นความสัมพันธ์ระหว่างสิ่งที่ Prisma สร้างขึ้นเบื้องหลังกับ SQL ที่เราคุ้นเคยมาตลอดหลักสูตร

### READ: findMany

```typescript
// Prisma
const allProducts = await prisma.product.findMany();
```

```sql
-- SQL ที่ Prisma สร้างขึ้นเบื้องหลัง (โดยประมาณ)
SELECT "product_id", "product_name", "unit_price", "stock_quantity"
FROM "products";
```

**พร้อม filter, order, limit:**

```typescript
// Prisma
const affordableProducts = await prisma.product.findMany({
  where: {
    unitPrice: { lt: 2000 },
    stockQuantity: { gt: 0 },
  },
  orderBy: { unitPrice: 'asc' },
  take: 10,
  skip: 0,
});
```

```sql
-- SQL เทียบเท่า
SELECT * FROM products
WHERE unit_price < 2000 AND stock_quantity > 0
ORDER BY unit_price ASC
LIMIT 10 OFFSET 0;
```

**ค้นหาข้อความบางส่วน (LIKE):**

```typescript
const searchResult = await prisma.product.findMany({
  where: {
    productName: { contains: 'คีย์บอร์ด', mode: 'insensitive' },
  },
});
```

```sql
SELECT * FROM products
WHERE product_name ILIKE '%คีย์บอร์ด%';
```

### READ: findUnique / findFirst

```typescript
// findUnique ใช้กับ field ที่เป็น @id หรือ @unique เท่านั้น
const product = await prisma.product.findUnique({
  where: { productId: 1 },
});
```

```sql
SELECT * FROM products WHERE product_id = 1 LIMIT 1;
```

```typescript
// findFirst ใช้กับเงื่อนไขทั่วไปที่ไม่จำเป็นต้องเป็น unique
const cheapest = await prisma.product.findFirst({
  where: { stockQuantity: { gt: 0 } },
  orderBy: { unitPrice: 'asc' },
});
```

```sql
SELECT * FROM products
WHERE stock_quantity > 0
ORDER BY unit_price ASC
LIMIT 1;
```

### CREATE: create

```typescript
const newProduct = await prisma.product.create({
  data: {
    productName: 'แผ่นรองเมาส์ขนาดใหญ่',
    unitPrice: 290.00,
    stockQuantity: 75,
  },
});
```

```sql
INSERT INTO products (product_name, unit_price, stock_quantity)
VALUES ('แผ่นรองเมาส์ขนาดใหญ่', 290.00, 75)
RETURNING product_id, product_name, unit_price, stock_quantity;
```

**สร้างหลายรายการพร้อมกัน:**

```typescript
const result = await prisma.product.createMany({
  data: [
    { productName: 'สายชาร์จ USB-C', unitPrice: 190, stockQuantity: 200 },
    { productName: 'ที่ชาร์จไร้สาย', unitPrice: 590, stockQuantity: 60 },
  ],
  skipDuplicates: true, // ข้ามรายการที่ซ้ำกับ unique constraint แทนที่จะ error
});
console.log(`สร้างสำเร็จ ${result.count} รายการ`);
```

```sql
INSERT INTO products (product_name, unit_price, stock_quantity)
VALUES
    ('สายชาร์จ USB-C', 190, 200),
    ('ที่ชาร์จไร้สาย', 590, 60)
ON CONFLICT DO NOTHING;
```

### UPDATE: update

```typescript
const updated = await prisma.product.update({
  where: { productId: 1 },
  data: { stockQuantity: 40 },
});
```

```sql
UPDATE products SET stock_quantity = 40
WHERE product_id = 1
RETURNING *;
```

**Update แบบคำนวณจากค่าเดิม (increment/decrement) — เทียบเท่ากับ atomic update ใน SQL:**

```typescript
const afterSale = await prisma.product.update({
  where: { productId: 1 },
  data: {
    stockQuantity: { decrement: 3 }, // ลดค่าเดิมลง 3 แบบ atomic
  },
});
```

```sql
UPDATE products
SET stock_quantity = stock_quantity - 3
WHERE product_id = 1
RETURNING *;
```

การใช้ `decrement`/`increment` สำคัญมาก เพราะเป็นการ update แบบ atomic ที่ปลอดภัยจาก race condition (ต่างจากการ `findUnique` มาอ่านค่าก่อนแล้วคำนวณใน JavaScript แล้วค่อย `update` ทีหลัง ซึ่งเสี่ยงต่อ lost update เมื่อมีหลาย request พร้อมกัน — concept เดียวกับที่เรียนใน Part เกี่ยวกับ concurrency control)

**updateMany:**

```typescript
const result = await prisma.product.updateMany({
  where: { stockQuantity: { lt: 5 } },
  data: { status: 'low_stock' }, // สมมติมี column status เพิ่มเติม
});
```

```sql
UPDATE products SET status = 'low_stock'
WHERE stock_quantity < 5;
```

### DELETE: delete

```typescript
const deleted = await prisma.product.delete({
  where: { productId: 10 },
});
```

```sql
DELETE FROM products WHERE product_id = 10
RETURNING *;
```

**deleteMany:**

```typescript
const result = await prisma.product.deleteMany({
  where: { stockQuantity: 0 },
});
console.log(`ลบสินค้าที่หมดสต็อกไป ${result.count} รายการ`);
```

```sql
DELETE FROM products WHERE stock_quantity = 0;
```

### upsert: สร้างหรืออัปเดตในคำสั่งเดียว

```typescript
const product = await prisma.product.upsert({
  where: { productId: 5 },
  update: { stockQuantity: { increment: 20 } },
  create: {
    productId: 5,
    productName: 'สินค้าใหม่',
    unitPrice: 100,
    stockQuantity: 20,
  },
});
```

```sql
INSERT INTO products (product_id, product_name, unit_price, stock_quantity)
VALUES (5, 'สินค้าใหม่', 100, 20)
ON CONFLICT (product_id)
DO UPDATE SET stock_quantity = products.stock_quantity + 20
RETURNING *;
```

### Aggregate: COUNT, SUM, AVG

```typescript
const stats = await prisma.product.aggregate({
  _count: { productId: true },
  _sum: { stockQuantity: true },
  _avg: { unitPrice: true },
  where: { stockQuantity: { gt: 0 } },
});
console.log(stats);
// { _count: { productId: 4 }, _sum: { stockQuantity: 115 }, _avg: { unitPrice: 2617.5 } }
```

```sql
SELECT
  COUNT(product_id) AS count,
  SUM(stock_quantity) AS sum,
  AVG(unit_price) AS avg
FROM products
WHERE stock_quantity > 0;
```

### groupBy

```typescript
const ordersByStatus = await prisma.order.groupBy({
  by: ['status'],
  _count: { orderId: true },
});
```

```sql
SELECT status, COUNT(order_id) AS count
FROM orders
GROUP BY status;
```

### เมื่อไหร่ควรใช้ raw SQL แทน Prisma query builder

แม้ Prisma จะครอบคลุมการ query ส่วนใหญ่ได้ดี แต่บาง query ที่ซับซ้อนมาก (window function, recursive CTE, full-text search แบบละเอียด) การเขียน Prisma query builder อาจทำได้ยากหรือไม่รองรับเลย ในกรณีนี้ Prisma มี escape hatch ให้ใช้ raw SQL ได้โดยตรง:

```typescript
// $queryRaw — ใช้ tagged template literal ที่ escape parameter ให้อัตโนมัติ (ปลอดภัยจาก SQL Injection)
const minPrice = 500;
const topProducts = await prisma.$queryRaw<
  { product_name: string; unit_price: number; rank: bigint }[]
>`
  SELECT
    product_name,
    unit_price,
    RANK() OVER (ORDER BY unit_price DESC) AS rank
  FROM products
  WHERE unit_price >= ${minPrice}
`;
```

```typescript
// $executeRaw — สำหรับคำสั่งที่ไม่คืนค่าแถว เช่น UPDATE/DELETE แบบ raw
await prisma.$executeRaw`
  UPDATE products
  SET stock_quantity = stock_quantity - ${1}
  WHERE product_id = ${5} AND stock_quantity >= ${1}
`;
```

> **ข้อควรระวัง**: ใช้ tagged template literal (```` prisma.$queryRaw`...` ````) เท่านั้น ห้ามใช้ `prisma.$queryRawUnsafe()` กับข้อมูลจากผู้ใช้โดยตรง เพราะ `$queryRawUnsafe` **ไม่มีการป้องกัน SQL Injection ให้อัตโนมัติ** เหมือนการต่อ string ธรรมดา

---

## Step 868: Prisma — Migration workflow เทียบเคียงกับ zero-downtime migration

### Prisma Migrate คืออะไร

Prisma Migrate เป็นเครื่องมือจัดการ schema migration ที่ทำงานคล้ายกับเครื่องมือ migration อื่นๆ ที่เคยเรียนมา (เช่น Flyway, golang-migrate ใน Part ก่อนหน้า) แต่ผูกเข้ากับ `schema.prisma` โดยตรง — เมื่อแก้ schema แล้วรันคำสั่ง migrate Prisma จะ diff schema เดิมกับใหม่ แล้ว generate ไฟล์ SQL migration ให้อัตโนมัติ

### migrate dev: ใช้ระหว่างพัฒนา (development)

```bash
npx prisma migrate dev --name add_order_notes
```

คำสั่งนี้ทำสิ่งต่อไปนี้ตามลำดับ:

1. เปรียบเทียบ `schema.prisma` ปัจจุบันกับสถานะฐานข้อมูลจริง (ผ่าน shadow database)
2. สร้างไฟล์ SQL migration ใหม่ในโฟลเดอร์ `prisma/migrations/<timestamp>_add_order_notes/migration.sql`
3. รัน migration นั้นกับฐานข้อมูล development ทันที
4. เรียก `prisma generate` ให้อัตโนมัติเพื่ออัปเดต Prisma Client

ตัวอย่างการเพิ่ม column ใหม่ใน schema:

```prisma
model Order {
  orderId    Int       @id @default(autoincrement()) @map("order_id")
  customerId Int?      @map("customer_id")
  orderDate  DateTime? @default(now()) @map("order_date") @db.Timestamptz(6)
  status     String?   @db.VarChar(20)
  notes      String?   @db.Text  // <-- field ใหม่ที่เพิ่มเข้ามา

  customer   Customer? @relation(fields: [customerId], references: [customerId])

  @@map("orders")
}
```

หลังรัน `prisma migrate dev --name add_order_notes` จะได้ไฟล์:

```sql
-- prisma/migrations/20260925103000_add_order_notes/migration.sql
ALTER TABLE "orders" ADD COLUMN "notes" TEXT;
```

### migrate deploy: ใช้ใน production/CI

```bash
npx prisma migrate deploy
```

ต่างจาก `migrate dev` ตรงที่ `migrate deploy`:
- **ไม่สร้าง migration ใหม่** เพียงแค่รัน migration ที่มีอยู่แล้วในโฟลเดอร์ `prisma/migrations/` ที่ยังไม่เคยถูกรันกับฐานข้อมูลเป้าหมาย
- **ไม่ใช้ shadow database** จึงไม่มี prompt โต้ตอบใดๆ เหมาะสำหรับรันอัตโนมัติใน CI/CD pipeline
- **ไม่เรียก `prisma generate`** อัตโนมัติ (ต้องเรียกแยกต่างหากถ้าต้องการ)

Workflow ทั่วไปใน CI/CD:

```yaml
# ตัวอย่างขั้นตอนใน CI/CD pipeline (แนวคิด ไม่ใช่ syntax เฉพาะเจาะจง)
steps:
  - run: npm ci
  - run: npx prisma generate
  - run: npx prisma migrate deploy   # รัน migration กับ production database
  - run: npm run build
  - run: npm run deploy
```

### เปรียบเทียบ Prisma Migrate กับแนวคิด zero-downtime migration (ทบทวน Part 079)

ใน Part 079 เราได้เรียนหลักการ zero-downtime migration ด้วยเทคนิค **expand-contract pattern**: เพิ่ม column/schema ใหม่แบบ backward-compatible ก่อน (expand), ปรับโค้ดแอปให้ใช้ของใหม่, แล้วค่อยลบของเก่าทีหลัง (contract) เพื่อไม่ให้แอปที่กำลังรันอยู่พังระหว่าง deploy

**Prisma Migrate ไม่ได้ทำ expand-contract ให้อัตโนมัติ** — มันเพียงสร้าง SQL migration ตรงไปตรงมาจาก diff ของ schema เท่านั้น ดังนั้นนักพัฒนาต้อง **ออกแบบลำดับการเปลี่ยน schema ในหลายขั้นตอนด้วยตนเอง** เพื่อให้ได้ zero-downtime migration ที่แท้จริง

**ตัวอย่าง: เปลี่ยนชื่อ column `status` เป็น `order_status` แบบ zero-downtime**

```sql
-- ถ้าใช้ prisma migrate dev แบบตรงไปตรงมา (สร้างจาก diff เดียว)
-- Prisma อาจสร้าง SQL แบบนี้ ซึ่ง "อันตราย" ต่อ production:
ALTER TABLE "orders" RENAME COLUMN "status" TO "order_status";
-- ปัญหา: ถ้าแอปเวอร์ชันเก่ายังรันอยู่ระหว่าง deploy (rolling deployment)
-- แอปเวอร์ชันเก่าจะ query column "status" ที่ไม่มีอยู่แล้ว -> error ทันที
```

**แนวทาง expand-contract ที่ถูกต้องเมื่อใช้ Prisma Migrate (ทำเป็นหลายขั้นตอน):**

```
ขั้นที่ 1 (Expand): เพิ่ม column ใหม่ โดยยังคง column เก่าไว้
```

```prisma
model Order {
  // ...
  status      String? @db.VarChar(20)      // column เก่า ยังคงอยู่
  orderStatus String? @map("order_status") @db.VarChar(20) // column ใหม่
}
```

```bash
npx prisma migrate dev --name add_order_status_column
```

```
ขั้นที่ 2: deploy โค้ดที่เขียนข้อมูลลงทั้งสอง column พร้อมกัน (dual write)
           แล้วรัน backfill script คัดลอกข้อมูลจาก status -> order_status
```

```sql
-- backfill script (รันแยกจาก migration file เพื่อควบคุม batch size ได้)
UPDATE orders SET order_status = status WHERE order_status IS NULL;
```

```
ขั้นที่ 3: deploy โค้ดที่อ่าน/เขียนเฉพาะ order_status เท่านั้น (ไม่แตะ status อีกต่อไป)
ขั้นที่ 4 (Contract): ลบ column status เก่าทิ้ง เมื่อมั่นใจว่าไม่มีแอปเวอร์ชันไหนใช้แล้ว
```

```prisma
model Order {
  // ...
  orderStatus String? @map("order_status") @db.VarChar(20)
  // status ถูกลบออกจาก schema แล้ว
}
```

```bash
npx prisma migrate dev --name drop_old_status_column
```

### ตารางเปรียบเทียบ operation ที่ปลอดภัย/อันตรายต่อ zero-downtime ใน Prisma Migrate

| Schema change | ความเสี่ยงต่อ downtime | คำแนะนำ |
|---|---|---|
| เพิ่ม column ใหม่ที่ nullable หรือมี default | ต่ำ | ทำได้โดยตรง ไม่ต้อง expand-contract |
| เพิ่ม column NOT NULL โดยไม่มี default | สูง (ตาราง lock ระหว่างเติมค่า, แอปเก่า insert ไม่ผ่าน) | เพิ่มแบบ nullable ก่อน backfill แล้วค่อยเพิ่ม constraint ทีหลัง |
| ลบ column | สูง (แอปเก่ายังอ้างอิงอยู่) | ทำตาม expand-contract: หยุดใช้ใน code ก่อน แล้วค่อยลบ schema |
| เปลี่ยนชื่อ column/table | สูงมาก (Prisma มองเป็น DROP + CREATE โดยดีฟอลต์) | ใช้แนวทาง เพิ่มใหม่ → dual write → migrate ข้อมูล → เลิกใช้ของเก่า |
| เพิ่ม index | ต่ำ-กลาง (index ปกติ lock table ระหว่างสร้าง) | แก้ไข migration SQL ที่ Prisma generate ให้เติม `CONCURRENTLY` (ดูด้านล่าง) |

### การแก้ไข migration SQL ที่ Prisma generate ให้ใช้ CONCURRENTLY

Prisma Migrate ใช้ `CREATE INDEX` แบบปกติ (ซึ่ง lock table) โดยดีฟอลต์ หากต้องการสร้าง index บนตารางขนาดใหญ่ใน production โดยไม่ lock table ต้องแก้ไขไฟล์ migration ที่ generate มาด้วยมือ:

```bash
# สร้าง migration แต่ไม่ apply ทันที เพื่อแก้ไขก่อน
npx prisma migrate dev --create-only --name add_index_on_order_date
```

```sql
-- prisma/migrations/xxx_add_index_on_order_date/migration.sql (ก่อนแก้ไข)
CREATE INDEX "orders_order_date_idx" ON "orders"("order_date");
```

แก้ไขเป็น:

```sql
-- แก้ไขด้วยมือให้ใช้ CONCURRENTLY เพื่อไม่ lock table ระหว่างสร้าง index
CREATE INDEX CONCURRENTLY "orders_order_date_idx" ON "orders"("order_date");
```

> **ข้อจำกัดสำคัญ**: `CREATE INDEX CONCURRENTLY` ใช้ภายใน transaction ไม่ได้ ต้องตั้งค่าพิเศษในไฟล์ migration (`-- prisma-client-js` metadata) หรือรันแยกนอก Prisma Migrate pipeline ปกติในบางกรณี ควรทดสอบใน staging environment ก่อนนำเข้า production เสมอ

### migrate resolve และ migrate status: จัดการสถานการณ์ migration ล้มเหลว

```bash
# ตรวจสอบสถานะ migration ปัจจุบัน เทียบกับฐานข้อมูล
npx prisma migrate status

# ถ้า migration ถูกรันด้วยมือไปแล้วนอกระบบ Prisma (เช่น รัน SQL ตรงๆ)
# สามารถ "บอก" Prisma ว่า migration นี้ถือว่าสำเร็จแล้วโดยไม่ต้องรันซ้ำ
npx prisma migrate resolve --applied "20260925103000_add_order_notes"

# ถ้า migration ล้มเหลวกลางคันและต้องการ mark ว่า rolled back แล้ว
npx prisma migrate resolve --rolled-back "20260925103000_add_order_notes"
```

---

## Step 869: Prisma — Relation queries (include, nested writes) เทียบกับ JOIN เอง

### include: ดึงข้อมูลจากตารางที่เกี่ยวข้องมาพร้อมกัน

```typescript
const ordersWithCustomer = await prisma.order.findMany({
  include: {
    customer: true,
  },
});
```

```typescript
// ผลลัพธ์ที่ได้ (type-safe เต็มรูปแบบ):
[
  {
    orderId: 1,
    customerId: 1,
    orderDate: '2026-09-20T10:00:00.000Z',
    status: 'pending',
    customer: {
      customerId: 1,
      firstName: 'สมชาย',
      email: 'somchai@example.com',
    },
  },
  // ...
]
```

**SQL ที่ Prisma สร้างขึ้นเบื้องหลัง (โดยประมาณ — Prisma มักใช้ query แยกแล้ว join ในหน่วยความจำ หรือ LATERAL JOIN ขึ้นกับเวอร์ชัน):**

```sql
-- แนวทางที่เทียบเท่าด้วย SQL ธรรมดาที่เราคุ้นเคย
SELECT
  o.order_id, o.customer_id, o.order_date, o.status,
  c.customer_id, c.first_name, c.email
FROM orders o
LEFT JOIN customers c ON c.customer_id = o.customer_id;
```

### include แบบ nested ลึกหลายชั้น พร้อม filter

```typescript
const customersWithRecentOrders = await prisma.customer.findMany({
  include: {
    orders: {
      where: { status: { not: 'cancelled' } },
      orderBy: { orderDate: 'desc' },
      take: 5,
    },
  },
});
```

```sql
-- SQL เทียบเท่าโดยประมาณ (Prisma มักรันเป็น 2 query แล้ว join ผลลัพธ์ในแอป
-- เพื่อหลีกเลี่ยงปัญหา N+1 และ row duplication จาก JOIN ปกติ)
SELECT * FROM customers;

SELECT * FROM orders
WHERE customer_id = ANY($1)   -- ids ของ customers ที่ query ได้
  AND status != 'cancelled'
ORDER BY order_date DESC;
-- แล้ว Prisma จับคู่ orders กับ customer ที่ถูกต้องใน application layer
-- พร้อมจำกัดแค่ 5 รายการต่อ customer ด้วย window function ภายใน (เวอร์ชันใหม่ใช้ LATERAL)
```

> **ข้อสังเกตสำคัญ**: Prisma ไม่ได้แปล `include` เป็น SQL `JOIN` ตรงไปตรงมาเสมอไปเหมือนที่นักพัฒนาหลายคนเข้าใจผิด ในเวอร์ชันปัจจุบัน Prisma มักใช้กลยุทธ์ **query แยกกันหลาย query แล้วรวมผลในแอปพลิเคชัน** (คล้าย DataLoader pattern) เพื่อหลีกเลี่ยงปัญหา row duplication ที่เกิดจาก JOIN แบบ one-to-many ธรรมดา ผู้เรียนที่คุ้นเคยกับการเขียน JOIN เองควรใช้ `prisma.$queryRaw` ร่วมกับ `EXPLAIN ANALYZE` (ทบทวน Part การวิเคราะห์ execution plan) เพื่อตรวจสอบ performance จริงเมื่อ query ซับซ้อนมาก

### select: เลือกเฉพาะ field ที่ต้องการ (คล้าย SELECT column เจาะจง)

```typescript
const orderSummaries = await prisma.order.findMany({
  select: {
    orderId: true,
    status: true,
    customer: {
      select: { firstName: true, email: true },
    },
  },
});
```

```sql
SELECT o.order_id, o.status, c.first_name, c.email
FROM orders o
LEFT JOIN customers c ON c.customer_id = o.customer_id;
```

### Nested writes: สร้างข้อมูลในหลายตารางพร้อมกันในคำสั่งเดียว

Prisma รองรับ **nested writes** ที่ห่อหุ้ม transaction ไว้ให้อัตโนมัติ — สร้าง parent record พร้อม child record ในคำสั่งเดียว โดยไม่ต้องเขียน `BEGIN/COMMIT` เอง:

```typescript
const newOrderWithNewCustomer = await prisma.order.create({
  data: {
    status: 'pending',
    customer: {
      create: {
        firstName: 'ประภา',
        email: 'prapa@example.com',
      },
    },
  },
  include: { customer: true },
});
```

**เทียบเท่ากับ transaction ที่เขียนด้วย node-postgres:**

```javascript
// เทียบเท่าด้วย node-postgres (ต้องเขียน transaction เองทั้งหมด)
const client = await pool.connect();
try {
  await client.query('BEGIN');

  const customerResult = await client.query(
    `INSERT INTO customers (first_name, email) VALUES ($1, $2) RETURNING customer_id`,
    ['ประภา', 'prapa@example.com']
  );
  const customerId = customerResult.rows[0].customer_id;

  const orderResult = await client.query(
    `INSERT INTO orders (customer_id, status) VALUES ($1, $2) RETURNING *`,
    [customerId, 'pending']
  );

  await client.query('COMMIT');
  return orderResult.rows[0];
} catch (err) {
  await client.query('ROLLBACK');
  throw err;
} finally {
  client.release();
}
```

จะเห็นได้ว่า Prisma ย่อโค้ด boilerplate ของ transaction ทั้งหมดให้เหลือเพียงคำสั่งเดียวที่อ่านง่าย โดยที่ Prisma จัดการ `BEGIN/COMMIT/ROLLBACK` ให้อัตโนมัติภายใน (nested write ทั้งหมดถูกห่อด้วย transaction เสมอ)

### เชื่อมกับ record ที่มีอยู่แล้ว (connect) แทนการสร้างใหม่

```typescript
const orderForExistingCustomer = await prisma.order.create({
  data: {
    status: 'pending',
    customer: {
      connect: { customerId: 1 }, // เชื่อมกับ customer ที่มีอยู่แล้ว ไม่สร้างใหม่
    },
  },
});
```

```sql
INSERT INTO orders (customer_id, status) VALUES (1, 'pending');
```

### connectOrCreate: เชื่อมถ้ามีอยู่แล้ว หรือสร้างใหม่ถ้ายังไม่มี

```typescript
const order = await prisma.order.create({
  data: {
    status: 'pending',
    customer: {
      connectOrCreate: {
        where: { email: 'newcustomer@example.com' },
        create: { firstName: 'ลูกค้าใหม่', email: 'newcustomer@example.com' },
      },
    },
  },
});
```

Prisma แปล `connectOrCreate` เป็น logic เทียบเท่าการเช็คก่อนว่ามี record ที่ตรงเงื่อนไข `where` อยู่แล้วหรือไม่ ถ้ามีก็เชื่อม (connect) กับของเดิม ถ้าไม่มีก็ `INSERT` ใหม่ ทั้งหมดภายใน transaction เดียวเพื่อป้องกัน race condition

### interactive transaction: เมื่อ nested write ยังไม่พอ

สำหรับ logic ที่ซับซ้อนกว่าการสร้าง record ที่เกี่ยวข้องกันตรงๆ (เช่น ตรวจสอบเงื่อนไขระหว่างทาง, ต้องอ่านค่าก่อนตัดสินใจ) Prisma มี `$transaction` แบบ interactive ที่ทำงานคล้าย `withTransaction` ที่เราเขียนเองด้วย node-postgres:

```typescript
const result = await prisma.$transaction(async (tx) => {
  const product = await tx.product.findUnique({ where: { productId: 1 } });

  if (!product || product.stockQuantity! < 2) {
    throw new Error('สต็อกสินค้าไม่เพียงพอ');
  }

  await tx.product.update({
    where: { productId: 1 },
    data: { stockQuantity: { decrement: 2 } },
  });

  const order = await tx.order.create({
    data: { customerId: 1, status: 'pending' },
  });

  return order;
});
// ถ้า throw error เกิดขึ้นระหว่างทาง Prisma จะ ROLLBACK ทั้งหมดให้อัตโนมัติ
```

โค้ดนี้เทียบเท่ากับ pattern `withTransaction` ที่เขียนไว้ใน Step 864 ทุกประการ เพียงแต่ Prisma จัดการ `BEGIN/COMMIT/ROLLBACK/release` ให้ทั้งหมดโดยอัตโนมัติผ่าน callback pattern

### Batch transaction (array form): รันหลาย query เป็น atomic โดยไม่ต้องมี logic คั่นกลาง

```typescript
const [updatedProduct, newOrder] = await prisma.$transaction([
  prisma.product.update({
    where: { productId: 1 },
    data: { stockQuantity: { decrement: 1 } },
  }),
  prisma.order.create({
    data: { customerId: 1, status: 'pending' },
  }),
]);
```

รูปแบบนี้เหมาะกับกรณีที่ query แต่ละตัวไม่ต้องพึ่งผลลัพธ์ของกันและกัน (ไม่มี logic แทรกกลาง) ทำให้ Prisma ส่งคำสั่งทั้งหมดเป็น batch เดียวได้อย่างมีประสิทธิภาพ

---

## Step 870: แบบฝึกหัดรวม — สร้าง REST API ด้วย Express + node-postgres/Prisma

หัวข้อนี้เป็นการประยุกต์ความรู้ทั้งหมดจากบทนี้มาสร้าง REST API เล็กๆ สำหรับจัดการสินค้าและคำสั่งซื้อของระบบ e-commerce โดยจะแสดงทั้งสองแนวทาง (node-postgres และ Prisma) เพื่อให้เปรียบเทียบได้ชัดเจน

### โครงสร้างโปรเจกต์

```
ecommerce-api/
├── .env
├── package.json
├── prisma/
│   └── schema.prisma
├── db/
│   ├── pool.js              # สำหรับแนวทาง node-postgres
│   └── prisma.js            # สำหรับแนวทาง Prisma
├── routes/
│   ├── products.pg.js       # routes แบบ node-postgres
│   ├── products.prisma.js   # routes แบบ Prisma
│   └── orders.prisma.js
└── index.js
```

```bash
npm install express pg dotenv
npm install prisma --save-dev
npm install @prisma/client
```

### ส่วนที่ 1: REST API ด้วย node-postgres

```javascript
// db/pool.js
const { Pool } = require('pg');

const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  max: 20,
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 5000,
});

pool.on('error', (err) => {
  console.error('Unexpected error on idle client', err);
});

module.exports = pool;
```

```javascript
// routes/products.pg.js
const express = require('express');
const router = express.Router();
const pool = require('../db/pool');

// GET /products — รายการสินค้าทั้งหมด พร้อม pagination
router.get('/', async (req, res) => {
  const page = Math.max(1, parseInt(req.query.page) || 1);
  const pageSize = Math.min(100, parseInt(req.query.pageSize) || 20);
  const offset = (page - 1) * pageSize;

  try {
    const result = await pool.query(
      `SELECT product_id, product_name, unit_price, stock_quantity
       FROM products
       ORDER BY product_id
       LIMIT $1 OFFSET $2`,
      [pageSize, offset]
    );
    const countResult = await pool.query('SELECT COUNT(*) FROM products');

    res.json({
      data: result.rows,
      pagination: {
        page,
        pageSize,
        total: parseInt(countResult.rows[0].count, 10),
      },
    });
  } catch (err) {
    console.error(err);
    res.status(500).json({ error: 'เกิดข้อผิดพลาดในการดึงข้อมูลสินค้า' });
  }
});

// GET /products/:id — สินค้าชิ้นเดียว
router.get('/:id', async (req, res) => {
  const productId = parseInt(req.params.id, 10);
  if (Number.isNaN(productId)) {
    return res.status(400).json({ error: 'product id ไม่ถูกต้อง' });
  }

  try {
    const result = await pool.query(
      'SELECT * FROM products WHERE product_id = $1',
      [productId]
    );
    if (result.rowCount === 0) {
      return res.status(404).json({ error: 'ไม่พบสินค้า' });
    }
    res.json(result.rows[0]);
  } catch (err) {
    console.error(err);
    res.status(500).json({ error: 'เกิดข้อผิดพลาดในการดึงข้อมูลสินค้า' });
  }
});

// POST /products — สร้างสินค้าใหม่
router.post('/', async (req, res) => {
  const { productName, unitPrice, stockQuantity } = req.body;

  if (!productName || unitPrice == null || stockQuantity == null) {
    return res.status(400).json({ error: 'กรุณาระบุ productName, unitPrice, stockQuantity ให้ครบถ้วน' });
  }
  if (unitPrice < 0 || stockQuantity < 0) {
    return res.status(400).json({ error: 'unitPrice และ stockQuantity ต้องไม่ติดลบ' });
  }

  try {
    const result = await pool.query(
      `INSERT INTO products (product_name, unit_price, stock_quantity)
       VALUES ($1, $2, $3)
       RETURNING *`,
      [productName, unitPrice, stockQuantity]
    );
    res.status(201).json(result.rows[0]);
  } catch (err) {
    console.error(err);
    res.status(500).json({ error: 'เกิดข้อผิดพลาดในการสร้างสินค้า' });
  }
});

// PATCH /products/:id/stock — ปรับสต็อกสินค้า (atomic update)
router.patch('/:id/stock', async (req, res) => {
  const productId = parseInt(req.params.id, 10);
  const { delta } = req.body; // เช่น -3 (ขายออก 3 ชิ้น) หรือ +10 (รับสินค้าเข้า 10 ชิ้น)

  if (Number.isNaN(productId) || typeof delta !== 'number') {
    return res.status(400).json({ error: 'พารามิเตอร์ไม่ถูกต้อง' });
  }

  try {
    const result = await pool.query(
      `UPDATE products
       SET stock_quantity = stock_quantity + $1
       WHERE product_id = $2 AND stock_quantity + $1 >= 0
       RETURNING *`,
      [delta, productId]
    );
    if (result.rowCount === 0) {
      return res.status(409).json({ error: 'ไม่สามารถปรับสต็อกได้ (สต็อกไม่พอ หรือไม่พบสินค้า)' });
    }
    res.json(result.rows[0]);
  } catch (err) {
    console.error(err);
    res.status(500).json({ error: 'เกิดข้อผิดพลาดในการปรับสต็อกสินค้า' });
  }
});

// DELETE /products/:id
router.delete('/:id', async (req, res) => {
  const productId = parseInt(req.params.id, 10);
  try {
    const result = await pool.query('DELETE FROM products WHERE product_id = $1', [productId]);
    if (result.rowCount === 0) {
      return res.status(404).json({ error: 'ไม่พบสินค้า' });
    }
    res.status(204).send();
  } catch (err) {
    console.error(err);
    res.status(500).json({ error: 'เกิดข้อผิดพลาดในการลบสินค้า' });
  }
});

module.exports = router;
```

```javascript
// routes/orders.pg.js
const express = require('express');
const router = express.Router();
const pool = require('../db/pool');

// POST /orders — สร้างคำสั่งซื้อพร้อมตัดสต็อกในคราวเดียว (transaction)
router.post('/', async (req, res) => {
  const { customerId, items } = req.body;
  // items = [{ productId: 1, quantity: 2 }, ...]

  if (!customerId || !Array.isArray(items) || items.length === 0) {
    return res.status(400).json({ error: 'กรุณาระบุ customerId และ items อย่างน้อย 1 รายการ' });
  }

  const client = await pool.connect();
  try {
    await client.query('BEGIN');

    const customerCheck = await client.query(
      'SELECT 1 FROM customers WHERE customer_id = $1',
      [customerId]
    );
    if (customerCheck.rowCount === 0) {
      await client.query('ROLLBACK');
      return res.status(404).json({ error: 'ไม่พบลูกค้า' });
    }

    const orderResult = await client.query(
      `INSERT INTO orders (customer_id, status) VALUES ($1, 'pending') RETURNING order_id, order_date, status`,
      [customerId]
    );
    const order = orderResult.rows[0];

    for (const item of items) {
      const stockResult = await client.query(
        `UPDATE products
         SET stock_quantity = stock_quantity - $1
         WHERE product_id = $2 AND stock_quantity >= $1
         RETURNING product_id`,
        [item.quantity, item.productId]
      );
      if (stockResult.rowCount === 0) {
        await client.query('ROLLBACK');
        return res.status(409).json({
          error: `สต็อกสินค้า product_id=${item.productId} ไม่เพียงพอสำหรับจำนวนที่สั่ง`,
        });
      }
    }

    await client.query('COMMIT');
    res.status(201).json({ ...order, customerId, items });
  } catch (err) {
    await client.query('ROLLBACK');
    console.error(err);
    res.status(500).json({ error: 'เกิดข้อผิดพลาดในการสร้างคำสั่งซื้อ' });
  } finally {
    client.release();
  }
});

// GET /orders — รายการคำสั่งซื้อพร้อมข้อมูลลูกค้า
router.get('/', async (req, res) => {
  try {
    const result = await pool.query(
      `SELECT o.order_id, o.order_date, o.status, c.first_name, c.email
       FROM orders o
       JOIN customers c ON c.customer_id = o.customer_id
       ORDER BY o.order_date DESC`
    );
    res.json(result.rows);
  } catch (err) {
    console.error(err);
    res.status(500).json({ error: 'เกิดข้อผิดพลาดในการดึงข้อมูลคำสั่งซื้อ' });
  }
});

module.exports = router;
```

### ส่วนที่ 2: REST API เดียวกัน ด้วย Prisma

`GET /products`, `GET /products/:id`, และ `POST /products` เขียนด้วย Prisma โดยใช้ pattern เดียวกับที่แสดงไว้แล้วใน Step 867 ทุกประการ (`findMany` + `count` สำหรับ pagination, `findUnique` สำหรับดึงรายตัว, `create` สำหรับสร้างใหม่) จึงขอไม่แสดงซ้ำในที่นี้ — หัวข้อนี้แสดงเฉพาะส่วนที่มีความแตกต่างจาก node-postgres อย่างชัดเจน คือ conditional atomic update และ nested-write transaction:

```javascript
// db/prisma.js
const { PrismaClient } = require('@prisma/client');

const globalForPrisma = globalThis;
const prisma = globalForPrisma.prisma || new PrismaClient();
if (process.env.NODE_ENV !== 'production') {
  globalForPrisma.prisma = prisma;
}

module.exports = prisma;
```

```javascript
// routes/products.prisma.js (แสดงเฉพาะ endpoint ที่ต่างจาก node-postgres อย่างมีนัยสำคัญ)
const express = require('express');
const router = express.Router();
const prisma = require('../db/prisma');

// PATCH /products/:id/stock — ต้องใช้ raw SQL เพราะ Prisma query builder
// ยังไม่รองรับเงื่อนไข "stock_quantity + delta >= 0" ใน WHERE โดยตรง
router.patch('/:id/stock', async (req, res) => {
  const productId = parseInt(req.params.id, 10);
  const { delta } = req.body;

  if (Number.isNaN(productId) || typeof delta !== 'number') {
    return res.status(400).json({ error: 'พารามิเตอร์ไม่ถูกต้อง' });
  }

  try {
    const rows = await prisma.$queryRaw`
      UPDATE products
      SET stock_quantity = stock_quantity + ${delta}
      WHERE product_id = ${productId} AND stock_quantity + ${delta} >= 0
      RETURNING *
    `;
    if (rows.length === 0) {
      return res.status(409).json({ error: 'ไม่สามารถปรับสต็อกได้ (สต็อกไม่พอ หรือไม่พบสินค้า)' });
    }
    res.json(rows[0]);
  } catch (err) {
    console.error(err);
    res.status(500).json({ error: 'เกิดข้อผิดพลาดในการปรับสต็อกสินค้า' });
  }
});

// DELETE /products/:id — Prisma throw error code P2025 เมื่อไม่พบ record ที่จะลบ
router.delete('/:id', async (req, res) => {
  const productId = parseInt(req.params.id, 10);
  try {
    await prisma.product.delete({ where: { productId } });
    res.status(204).send();
  } catch (err) {
    if (err.code === 'P2025') {
      return res.status(404).json({ error: 'ไม่พบสินค้า' });
    }
    console.error(err);
    res.status(500).json({ error: 'เกิดข้อผิดพลาดในการลบสินค้า' });
  }
});

module.exports = router;
```

```javascript
// routes/orders.prisma.js — สร้างคำสั่งซื้อพร้อมตัดสต็อกด้วย interactive transaction
// (เทียบเท่ากับ routes/orders.pg.js ที่เขียนด้วยมือใน BEGIN/COMMIT/ROLLBACK)
const express = require('express');
const router = express.Router();
const prisma = require('../db/prisma');

router.post('/', async (req, res) => {
  const { customerId, items } = req.body;

  if (!customerId || !Array.isArray(items) || items.length === 0) {
    return res.status(400).json({ error: 'กรุณาระบุ customerId และ items อย่างน้อย 1 รายการ' });
  }

  try {
    const order = await prisma.$transaction(async (tx) => {
      const customer = await tx.customer.findUnique({ where: { customerId } });
      if (!customer) {
        throw Object.assign(new Error('ไม่พบลูกค้า'), { statusCode: 404 });
      }

      const newOrder = await tx.order.create({ data: { customerId, status: 'pending' } });

      for (const item of items) {
        const updated = await tx.product.updateMany({
          where: { productId: item.productId, stockQuantity: { gte: item.quantity } },
          data: { stockQuantity: { decrement: item.quantity } },
        });
        if (updated.count === 0) {
          throw Object.assign(
            new Error(`สต็อกสินค้า product_id=${item.productId} ไม่เพียงพอ`),
            { statusCode: 409 }
          );
        }
      }
      return newOrder;
    });

    res.status(201).json(order);
  } catch (err) {
    console.error(err);
    res.status(err.statusCode || 500).json({ error: err.message || 'เกิดข้อผิดพลาดในการสร้างคำสั่งซื้อ' });
  }
});

module.exports = router;
```

### ประกอบเข้าเป็นแอปเดียว (index.js)

```javascript
// index.js
require('dotenv').config();
const express = require('express');
const pool = require('./db/pool');

const app = express();
app.use(express.json());

// เลือกใช้ router ตัวใดตัวหนึ่ง — สลับกันดูความแตกต่างได้
app.use('/products', require('./routes/products.pg.js'));    // แนวทาง node-postgres
app.use('/orders', require('./routes/orders.pg.js'));
// app.use('/products', require('./routes/products.prisma.js')); // แนวทาง Prisma
// app.use('/orders', require('./routes/orders.prisma.js'));

app.get('/health', async (req, res) => {
  try {
    await pool.query('SELECT 1');
    res.json({ status: 'ok' });
  } catch (err) {
    res.status(503).json({ status: 'unavailable' });
  }
});

// error handler กลาง (fallback สำหรับ error ที่ไม่ได้ดักจับใน route)
app.use((err, req, res, next) => {
  console.error('Unhandled error:', err);
  res.status(500).json({ error: 'เกิดข้อผิดพลาดที่ไม่คาดคิด' });
});

const PORT = process.env.PORT || 3000;
const server = app.listen(PORT, () => {
  console.log(`API server กำลังทำงานที่ port ${PORT}`);
});

async function gracefulShutdown() {
  console.log('กำลังปิดระบบ...');
  server.close(async () => {
    await pool.end();
    process.exit(0);
  });
}

process.on('SIGTERM', gracefulShutdown);
process.on('SIGINT', gracefulShutdown);
```

### ทดสอบ API ด้วย curl

```bash
# ดึงรายการสินค้า
curl http://localhost:3000/products?page=1&pageSize=10

# สร้างสินค้าใหม่
curl -X POST http://localhost:3000/products \
  -H "Content-Type: application/json" \
  -d '{"productName":"ลำโพงบลูทูธ","unitPrice":1490.00,"stockQuantity":15}'

# ปรับสต็อก (ขายออก 2 ชิ้น)
curl -X PATCH http://localhost:3000/products/1/stock \
  -H "Content-Type: application/json" \
  -d '{"delta":-2}'

# สร้างคำสั่งซื้อ
curl -X POST http://localhost:3000/orders \
  -H "Content-Type: application/json" \
  -d '{"customerId":1,"items":[{"productId":1,"quantity":2},{"productId":2,"quantity":1}]}'
```

---

## สรุปท้ายบท

ในบทนี้เราได้เรียนรู้การเชื่อมต่อ PostgreSQL จาก Node.js ในสองระดับที่สำคัญ:

**node-postgres (pg)** เป็น driver ระดับพื้นฐานที่คุยกับ PostgreSQL protocol โดยตรง ให้ความเร็วสูงสุดและควบคุม SQL ได้เต็มที่ เราได้เรียนความแตกต่างระหว่าง `Client` (connection เดี่ยว) กับ `Pool` (กลุ่ม connection ที่ใช้ซ้ำได้ เหมาะกับ web application) การเขียน parameterized query ด้วย `$1, $2` เพื่อป้องกัน SQL Injection อย่างเด็ดขาด การจัดการ transaction ด้วย `BEGIN/COMMIT/ROLLBACK` ผ่าน pattern try/catch/finally ที่ปลอดภัย (ต้อง `release()` connection กลับสู่ pool เสมอ) และการจัดการ error รวมถึง connection pool อย่างเป็นระบบใน production

**Prisma** เป็น ORM ที่ยกระดับ developer experience ด้วย type-safety เต็มรูปแบบผ่าน `schema.prisma` และการ generate client อัตโนมัติ เราได้เปรียบเทียบ CRUD operation ของ Prisma กับ raw SQL ที่เทียบเท่ากันในทุกกรณี เพื่อให้เข้าใจว่า Prisma ทำงานอย่างไรเบื้องหลัง ได้เรียน Prisma Migrate ทั้ง `migrate dev` และ `migrate deploy` พร้อมเชื่อมโยงกับแนวคิด zero-downtime migration (expand-contract pattern) ที่เรียนมาใน Part 079 และปิดท้ายด้วย relation query (`include`, nested writes, interactive transaction) ที่เทียบเคียงกับการเขียน JOIN และ transaction เองด้วยมือ

จุดสำคัญที่สุดที่ควรจำจากบทนี้คือ **ไม่ว่าจะใช้เครื่องมือใด ความเข้าใจ SQL, index, transaction isolation, และ execution plan ที่เรียนมาตลอดหลักสูตรนี้ยังคงเป็นรากฐานที่ขาดไม่ได้** — ORM เพียงช่วยลดงาน boilerplate และเพิ่มความปลอดภัยด้าน type แต่การตัดสินใจเชิงสถาปัตยกรรม การ debug performance ปัญหา และการออกแบบ schema ที่ดียังคงต้องอาศัยความรู้พื้นฐานฐานข้อมูลอย่างลึกซึ้ง

บทถัดไปจะพาไปดูการเชื่อมต่อ PostgreSQL จาก Python ผ่าน psycopg และ SQLAlchemy ซึ่งเป็นอีกหนึ่ง ecosystem ที่ใช้กันอย่างแพร่หลายในงาน data engineering และ backend development

---

## แบบฝึกหัด

<details>
<summary><b>แบบฝึกหัดที่ 1:</b> อธิบายความแตกต่างระหว่าง `Client` และ `Pool` ใน node-postgres พร้อมยกตัวอย่างสถานการณ์ที่ควรใช้แต่ละแบบ</summary>

**เฉลย:**

`Client` คือ connection เดี่ยวไปยัง PostgreSQL ต้องเรียก `connect()` ก่อนใช้งานและ `end()` เมื่อเลิกใช้ด้วยตนเอง เหมาะกับ script สั้นๆ ที่รันครั้งเดียว เช่น migration script หรือ batch job ที่ต้องการ session state เฉพาะ (เช่น `SET search_path`, temporary table ที่ผูกกับ session)

`Pool` คือกลุ่มของ connection หลายเส้นที่ถูกจัดการให้อัตโนมัติ — เมื่อเรียก `pool.query()` จะขอ connection ว่างจาก pool มาใช้ แล้วคืนกลับให้อัตโนมัติเมื่อ query เสร็จ เหมาะกับ web application/API ที่ต้องรองรับหลาย request พร้อมกัน เพราะแต่ละ request สามารถใช้ connection คนละเส้นพร้อมกันได้ ทำให้ throughput สูงกว่าการใช้ Client เดี่ยว

**ตัวอย่าง**: ระบบ e-commerce API ที่ต้องรองรับผู้ใช้หลายคนสั่งซื้อพร้อมกันควรใช้ `Pool` เป็นค่าเริ่มต้น ส่วน script ที่รันครั้งเดียวเพื่อ seed ข้อมูลเริ่มต้นลง database สามารถใช้ `Client` ได้เพราะไม่ต้องการ concurrency

</details>

<details>
<summary><b>แบบฝึกหัดที่ 2:</b> โค้ดต่อไปนี้มีความเสี่ยงด้านความปลอดภัยหรือไม่ อย่างไร และควรแก้ไขอย่างไร

```javascript
app.get('/products/search', async (req, res) => {
  const keyword = req.query.q;
  const result = await pool.query(
    `SELECT * FROM products WHERE product_name LIKE '%${keyword}%'`
  );
  res.json(result.rows);
});
```
</summary>

**เฉลย:**

โค้ดนี้มีความเสี่ยง **SQL Injection** ร้ายแรง เพราะ `keyword` จาก `req.query.q` ถูกต่อเข้ากับ SQL string โดยตรงโดยไม่ผ่านการ sanitize หรือใช้ parameterized query หากผู้ใช้ส่ง query string เช่น `?q=x%'; DROP TABLE products; --` จะสามารถรันคำสั่ง SQL ใดๆ ก็ได้ตามที่ต้องการ รวมถึงลบตารางทิ้ง หรือดึงข้อมูลลับออกไป

**วิธีแก้ไข** ใช้ parameterized query แทน:

```javascript
app.get('/products/search', async (req, res) => {
  const keyword = req.query.q || '';
  const result = await pool.query(
    'SELECT * FROM products WHERE product_name ILIKE $1',
    [`%${keyword}%`]
  );
  res.json(result.rows);
});
```

การส่ง `%${keyword}%` เป็น parameter value (ไม่ใช่การต่อ string เข้าไปใน SQL text) ทำให้ PostgreSQL ปฏิบัติต่อค่านี้เป็นข้อมูลเสมอ ไม่มีทางถูกตีความเป็นคำสั่ง SQL ได้ไม่ว่าจะมีอักขระพิเศษอะไรอยู่ในนั้น

</details>

<details>
<summary><b>แบบฝึกหัดที่ 3:</b> เขียนฟังก์ชัน `moveStock(fromProductId, toProductId, quantity)` ด้วย node-postgres ที่ย้ายสต็อกจากสินค้าหนึ่งไปอีกสินค้าหนึ่งแบบ transaction ปลอดภัย โดยต้องตรวจสอบว่าสต็อกต้นทางมีเพียงพอก่อนย้าย</summary>

**เฉลย:**

```javascript
async function moveStock(fromProductId, toProductId, quantity) {
  const client = await pool.connect();
  try {
    await client.query('BEGIN');

    const decreaseResult = await client.query(
      `UPDATE products
       SET stock_quantity = stock_quantity - $1
       WHERE product_id = $2 AND stock_quantity >= $1
       RETURNING stock_quantity`,
      [quantity, fromProductId]
    );

    if (decreaseResult.rowCount === 0) {
      throw new Error('สต็อกสินค้าต้นทางไม่เพียงพอ หรือไม่พบสินค้า');
    }

    const increaseResult = await client.query(
      `UPDATE products
       SET stock_quantity = stock_quantity + $1
       WHERE product_id = $2
       RETURNING stock_quantity`,
      [quantity, toProductId]
    );

    if (increaseResult.rowCount === 0) {
      throw new Error('ไม่พบสินค้าปลายทาง');
    }

    await client.query('COMMIT');
    return { success: true };
  } catch (err) {
    await client.query('ROLLBACK');
    throw err;
  } finally {
    client.release();
  }
}
```

จุดสำคัญ: การตรวจสอบ `stock_quantity >= $1` ในเงื่อนไข `WHERE` ของคำสั่ง `UPDATE` ทำให้การตรวจสอบและการลดค่าเกิดขึ้นเป็น atomic operation เดียว (ไม่มีช่องว่างให้เกิด race condition ระหว่างการอ่านค่าตรวจสอบกับการเขียนค่าใหม่) ต่างจากการ `SELECT` มาตรวจสอบก่อนแล้วค่อย `UPDATE` แยกกัน ซึ่งเสี่ยงต่อ lost update เมื่อมีหลาย request เข้ามาพร้อมกัน

</details>

<details>
<summary><b>แบบฝึกหัดที่ 4:</b> ทำไม `NUMERIC` type ใน PostgreSQL จึงถูกแปลงเป็น string เมื่อดึงข้อมูลผ่าน node-postgres แทนที่จะเป็น number โดยตรง และมีวิธีจัดการอย่างไร</summary>

**เฉลย:**

`NUMERIC`/`DECIMAL` ใน PostgreSQL เป็นชนิดข้อมูลที่มีความแม่นยำสูง (arbitrary precision) สามารถเก็บตัวเลขทศนิยมได้แม่นยำโดยไม่มีปัญหา floating-point rounding error ในขณะที่ JavaScript `number` เป็น IEEE 754 double-precision floating point ซึ่งมีข้อจำกัดด้านความแม่นยำ (เช่น `0.1 + 0.2 !== 0.3` ใน JavaScript) หาก node-postgres แปลง `NUMERIC` เป็น `number` โดยอัตโนมัติ อาจทำให้ข้อมูลทางการเงินสูญเสียความแม่นยำโดยไม่รู้ตัว ซึ่งเป็นเรื่องอันตรายมากสำหรับระบบ e-commerce ที่เกี่ยวข้องกับราคาสินค้าและยอดเงิน ด้วยเหตุนี้ node-postgres จึงเลือกคืนค่าเป็น string เพื่อรักษาความแม่นยำไว้ให้ผู้พัฒนาตัดสินใจเอง

**วิธีจัดการ**:
1. แปลงเป็น number ด้วย `parseFloat()` เมื่อแน่ใจว่าความแม่นยำที่ได้เพียงพอสำหรับการแสดงผล (เช่น แสดงราคาในหน้าเว็บ)
2. ใช้ library เช่น `decimal.js` หรือ `big.js` เพื่อคำนวณตัวเลขทศนิยมอย่างแม่นยำเมื่อต้องทำการคำนวณทางการเงิน (เช่น รวมยอดคำสั่งซื้อ, คำนวณภาษี)
3. ปรับแต่ง type parser ของ `pg` ให้แปลง `NUMERIC` (OID 1700) เป็น float อัตโนมัติได้ด้วย `pg.types.setTypeParser(1700, parseFloat)` แต่ต้องระวังผลกระทบด้านความแม่นยำหากใช้ในจุดที่เกี่ยวกับเงิน

</details>

<details>
<summary><b>แบบฝึกหัดที่ 5:</b> เขียน `schema.prisma` model สำหรับตาราง `order_items` ใหม่ (ที่ไม่ได้อยู่ใน schema เดิมของบทนี้) ซึ่งมีโครงสร้าง SQL ดังนี้ พร้อมกำหนด relation กับ `Order` และ `Product` ให้ถูกต้อง

```sql
CREATE TABLE order_items (
    order_item_id SERIAL PRIMARY KEY,
    order_id INTEGER REFERENCES orders(order_id),
    product_id INTEGER REFERENCES products(product_id),
    quantity INTEGER NOT NULL,
    unit_price NUMERIC(10,2) NOT NULL
);
```
</summary>

**เฉลย:**

```prisma
model OrderItem {
  orderItemId Int      @id @default(autoincrement()) @map("order_item_id")
  orderId     Int?     @map("order_id")
  productId   Int?     @map("product_id")
  quantity    Int
  unitPrice   Decimal  @map("unit_price") @db.Decimal(10, 2)

  order       Order?   @relation(fields: [orderId], references: [orderId])
  product     Product? @relation(fields: [productId], references: [productId])

  @@map("order_items")
}
```

และต้องเพิ่ม relation field กลับใน model `Order` และ `Product` ด้วย (Prisma กำหนดให้ relation ต้องประกาศทั้งสองฝั่งเสมอ):

```prisma
model Order {
  // ... field เดิม
  orderItems OrderItem[]
}

model Product {
  // ... field เดิม
  orderItems OrderItem[]
}
```

จุดสำคัญ: `quantity` และ `unitPrice` ไม่มีเครื่องหมาย `?` เพราะเป็น `NOT NULL` ใน SQL ต้นฉบับ ในขณะที่ `orderId` และ `productId` มี `?` เพราะ column เหล่านี้อนุญาตให้เป็น NULL ได้ (ไม่มี NOT NULL constraint ในโจทย์)

</details>

<details>
<summary><b>แบบฝึกหัดที่ 6:</b> เขียนโค้ด Prisma ที่ทำสิ่งเดียวกันกับ SQL ต่อไปนี้

```sql
SELECT product_name, unit_price
FROM products
WHERE stock_quantity > 0 AND unit_price BETWEEN 500 AND 2000
ORDER BY unit_price DESC
LIMIT 5;
```
</summary>

**เฉลย:**

```typescript
const products = await prisma.product.findMany({
  select: {
    productName: true,
    unitPrice: true,
  },
  where: {
    stockQuantity: { gt: 0 },
    unitPrice: { gte: 500, lte: 2000 },
  },
  orderBy: { unitPrice: 'desc' },
  take: 5,
});
```

</details>

<details>
<summary><b>แบบฝึกหัดที่ 7:</b> อธิบายความแตกต่างระหว่าง `npx prisma migrate dev` กับ `npx prisma migrate deploy` ว่าควรใช้เมื่อใด และทำไม `migrate dev` จึงไม่ควรใช้ใน production</summary>

**เฉลย:**

`migrate dev` ออกแบบมาสำหรับ **สภาพแวดล้อมพัฒนา (development)** โดยจะ:
- เปรียบเทียบ schema กับฐานข้อมูลผ่าน **shadow database** (ฐานข้อมูลชั่วคราวที่ Prisma สร้างขึ้นเพื่อทดสอบ migration)
- สร้างไฟล์ migration ใหม่โดยอัตโนมัติจาก diff ของ schema
- อาจถาม prompt ให้ยืนยันเมื่อ migration มีความเสี่ยงต่อการสูญเสียข้อมูล (เช่น การลบ column)
- รัน `prisma generate` ให้อัตโนมัติ

`migrate deploy` ออกแบบมาสำหรับ **production/CI/CD** โดยจะ:
- ไม่สร้าง migration ใหม่ เพียงรัน migration ที่มีอยู่แล้วในโฟลเดอร์ `prisma/migrations/` ที่ยังไม่เคยถูก apply
- ไม่ใช้ shadow database และไม่มี interactive prompt ใดๆ (เหมาะกับการรันอัตโนมัติแบบไม่มีคนคอยตอบคำถาม)
- ไม่เรียก `prisma generate` อัตโนมัติ

**ทำไม `migrate dev` ไม่ควรใช้ใน production**: เพราะมันอาจสร้างและรัน migration ใหม่แบบ interactive กลางอากาศโดยไม่มีการ review ก่อน และการต้องสร้าง shadow database ใน production เป็นความเสี่ยงด้านความปลอดภัยและ operational overhead ที่ไม่จำเป็น นอกจากนี้ migration ควรถูกสร้างและ review (ผ่าน code review, PR) ในสภาพแวดล้อม development ก่อนนำไป deploy จริงเสมอ ไม่ใช่สร้างสดๆ ตอน deploy

</details>

<details>
<summary><b>แบบฝึกหัดที่ 8:</b> สมมติต้องการเปลี่ยนชื่อ column `status` ในตาราง `orders` เป็น `order_status` โดยไม่ให้แอปพลิเคชันที่กำลังรันอยู่ (ระหว่าง rolling deployment) เกิด error เลยแม้แต่วินาทีเดียว จงอธิบายขั้นตอนแบบ expand-contract อย่างละเอียด</summary>

**เฉลย:**

การ `ALTER TABLE ... RENAME COLUMN` แบบตรงไปตรงมาเป็นอันตราย เพราะระหว่าง rolling deployment จะมีทั้งแอปเวอร์ชันเก่าและใหม่รันพร้อมกันชั่วคราว ถ้า rename column ทันที แอปเวอร์ชันเก่าที่ยังอ้างอิง column เดิมจะ error ทันที ขั้นตอนที่ปลอดภัยมี 4 ขั้น:

**ขั้นที่ 1 (Expand)**: เพิ่ม column ใหม่ `order_status` โดยยังคง column เก่า `status` ไว้ deploy migration นี้ก่อน โดยที่โค้ดแอปยังไม่เปลี่ยนแปลง (ยังใช้ `status` เหมือนเดิม)

**ขั้นที่ 2 (Dual write + backfill)**: deploy โค้ดแอปเวอร์ชันใหม่ที่เขียนข้อมูลลงทั้ง `status` และ `order_status` พร้อมกันทุกครั้งที่มีการ update จากนั้นรัน backfill script แบบ batch เพื่อคัดลอกข้อมูลเก่าที่มีอยู่แล้วจาก `status` ไปยัง `order_status` (ทำแบบแบ่ง batch เพื่อไม่ lock table นานเกินไป)

**ขั้นที่ 3 (Switch read)**: เมื่อมั่นใจว่าข้อมูลใน `order_status` ตรงกับ `status` ครบถ้วนแล้ว deploy โค้ดแอปเวอร์ชันที่อ่านและเขียนเฉพาะ `order_status` เท่านั้น (หยุดใช้ `status` โดยสมบูรณ์ แต่ column เก่ายังคงอยู่ในฐานข้อมูล)

**ขั้นที่ 4 (Contract)**: เมื่อมั่นใจว่าไม่มีแอปเวอร์ชันไหนในระบบอ้างอิง `status` อีกต่อไป (รอให้ rolling deployment เสร็จสมบูรณ์และผ่านไปสักระยะเพื่อความปลอดภัย) จึงค่อย migration ลบ column `status` ทิ้ง

ทุกขั้นตอนสามารถ deploy แยกกันได้อิสระ และแต่ละขั้นตอนไม่ทำให้แอปเวอร์ชันใดๆ ที่กำลังรันอยู่พังเลย นี่คือแก่นของ zero-downtime migration ที่ต้องออกแบบเองแม้จะใช้ Prisma Migrate ซึ่งไม่ได้ทำสิ่งนี้ให้อัตโนมัติ

</details>

<details>
<summary><b>แบบฝึกหัดที่ 9:</b> โค้ด Prisma ต่อไปนี้มีปัญหาเรื่อง race condition หรือไม่ อธิบายเหตุผล และแก้ไขให้ถูกต้อง

```typescript
async function purchaseProduct(productId, quantity) {
  const product = await prisma.product.findUnique({ where: { productId } });
  if (product.stockQuantity < quantity) {
    throw new Error('สต็อกไม่พอ');
  }
  await prisma.product.update({
    where: { productId },
    data: { stockQuantity: product.stockQuantity - quantity },
  });
}
```
</summary>

**เฉลย:**

โค้ดนี้มีปัญหา **race condition** ชัดเจน เพราะมีช่องว่างระหว่างการอ่านค่า (`findUnique`) กับการเขียนค่าใหม่ (`update`) หากมีสอง request เข้ามาพร้อมกันในเวลาใกล้เคียงกัน (เช่น สต็อกมี 5 ชิ้น ทั้งสอง request ต่างต้องการซื้อ 3 ชิ้น) ทั้งสอง request อาจอ่านค่า `stockQuantity = 5` พร้อมกัน ผ่านเงื่อนไข `5 < 3` เป็น false ทั้งคู่ แล้วต่างคน update เป็น `5 - 3 = 2` พร้อมกัน ผลลัพธ์สุดท้ายคือสต็อกเหลือ 2 แต่ในความเป็นจริงขายไปแล้ว 6 ชิ้น (เกินสต็อกที่มีจริง) — นี่คือปัญหา **lost update**

**วิธีแก้ไข** ใช้ atomic update พร้อมเงื่อนไขตรวจสอบใน `where` clause เดียวกับที่ update (เทคนิคเดียวกับที่ใช้ใน node-postgres):

```typescript
async function purchaseProduct(productId, quantity) {
  const result = await prisma.product.updateMany({
    where: {
      productId,
      stockQuantity: { gte: quantity }, // ตรวจสอบเงื่อนไขในระดับ SQL WHERE
    },
    data: {
      stockQuantity: { decrement: quantity }, // atomic decrement
    },
  });

  if (result.count === 0) {
    throw new Error('สต็อกไม่พอ หรือไม่พบสินค้า');
  }
}
```

วิธีนี้ทำให้การตรวจสอบเงื่อนไขและการลดค่าเกิดขึ้นเป็น atomic operation เดียวในระดับฐานข้อมูล (คล้ายกับ `UPDATE ... WHERE stock_quantity >= $1` ใน SQL) ซึ่งปลอดภัยจาก race condition เพราะ PostgreSQL รับประกัน isolation ของแต่ละ UPDATE statement ผ่าน row-level locking โดยอัตโนมัติ

</details>

<details>
<summary><b>แบบฝึกหัดที่ 10:</b> อธิบายว่าทำไม `prisma.$queryRawUnsafe()` จึงอันตรายกว่า `prisma.$queryRaw` แบบ tagged template literal และยกตัวอย่างโค้ดที่แสดงความแตกต่างนี้</summary>

**เฉลย:**

`prisma.$queryRaw` เมื่อใช้กับ **tagged template literal** (syntax แบบ ```` prisma.$queryRaw`SELECT * FROM products WHERE product_id = ${id}` ````) จะทำให้ Prisma แยกค่าที่อยู่ใน `${}` ออกมาเป็น parameter โดยอัตโนมัติ แล้วส่งไปยัง PostgreSQL ผ่าน parameterized query (เหมือนกับ `$1, $2` ใน node-postgres) ทำให้ปลอดภัยจาก SQL Injection โดยอัตโนมัติ

ในขณะที่ `prisma.$queryRawUnsafe()` รับ SQL เป็น string ธรรมดา และค่าที่ส่งเข้าไปจะถูกต่อเข้ากับ string โดยตรง **ไม่มีการป้องกัน SQL Injection ให้อัตโนมัติเลย** — พฤติกรรมเดียวกับการต่อ string SQL ด้วยมือใน node-postgres ที่อันตราย

**ตัวอย่างเปรียบเทียบ**:

```typescript
// ❌ อันตราย — $queryRawUnsafe ต่อ string ตรงๆ ถ้า keyword มาจาก user โดยไม่ผ่านการตรวจสอบ
const keyword = req.query.q; // สมมติผู้ใช้ส่ง: x'; DROP TABLE products; --
const products = await prisma.$queryRawUnsafe(
  `SELECT * FROM products WHERE product_name = '${keyword}'`
);

// ✅ ปลอดภัย — $queryRaw แบบ tagged template literal escape parameter ให้อัตโนมัติ
const products = await prisma.$queryRaw`
  SELECT * FROM products WHERE product_name = ${keyword}
`;
```

หลักการที่ควรจำ: ใช้ `$queryRawUnsafe`/`$executeRawUnsafe` **เฉพาะกรณีที่ SQL ต้องเป็น dynamic ในส่วนที่ไม่ใช่ค่าข้อมูล** เช่น ชื่อ table หรือ column ที่มาจากค่าที่ผ่านการ whitelist ไว้แล้วเท่านั้น (เหมือนกับหลักการเดียวกันที่ใช้กับ node-postgres ใน Step 863) และห้ามส่งค่าจากผู้ใช้ (user input) เข้าไปใน `$queryRawUnsafe` โดยตรงเด็ดขาด

</details>

---

บทถัดไป: [Part 088 — เชื่อมต่อ PostgreSQL กับ Python (psycopg, SQLAlchemy)](./part-088-python-integration.md)
