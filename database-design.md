# Database Design - การออกแบบฐานข้อมูลที่มีประสิทธิภาพ

## 📋 สารบัญ
- [Database Design คืออะไร?](#database-design-คืออะไร)
- [หลักการออกแบบฐานข้อมูล](#หลักการออกแบบฐานข้อมูล)
- [Entity Relationship Diagram (ERD)](#entity-relationship-diagram-erd)
- [Normalization](#normalization)
- [Data Types](#data-types)
- [Indexing](#indexing)
- [Constraints](#constraints)
- [Performance Considerations](#performance-considerations)

## Database Design คืออะไร?

**Database Design** คือการวางแผนและจัดโครงสร้างฐานข้อมูลให้เหมาะสมกับการใช้งาน โดยคำนึงถึงประสิทธิภาพ ความถูกต้อง และความง่ายในการบำรุงรักษา

### ทำไมต้องออกแบบฐานข้อมูล?
- 🎯 **ประสิทธิภาพ**: Query เร็วขึ้น
- 📊 **ความถูกต้อง**: ข้อมูลไม่ซ้ำซ้อน
- 🔧 **บำรุงรักษา**: แก้ไขง่าย
- 📈 **ขยายตัว**: รองรับข้อมูลเพิ่มขึ้น

## หลักการออกแบบฐานข้อมูล

### 1. **ระบุ Entities และ Relationships**

#### ตัวอย่าง: ระบบร้านค้าออนไลน์
```
Entities:
- User (ผู้ใช้)
- Product (สินค้า)
- Order (ออเดอร์)
- Category (หมวดหมู่)

Relationships:
- User สามารถมี Order หลายอัน (1:M)
- Order มี Product หลายตัว (M:N)
- Product อยู่ใน Category หนึ่งอัน (M:1)
```

### 2. **กำหนด Attributes**
```sql
-- User Entity
User:
- user_id (Primary Key)
- email
- password
- first_name
- last_name
- created_at

-- Product Entity  
Product:
- product_id (Primary Key)
- name
- description
- price
- stock_quantity
- category_id (Foreign Key)
```

### 3. **กำหนด Primary และ Foreign Keys**
```sql
-- Primary Keys
User: user_id
Product: product_id
Order: order_id
Category: category_id

-- Foreign Keys
Product.category_id → Category.category_id
Order.user_id → User.user_id
```

## Entity Relationship Diagram (ERD)

### สัญลักษณ์ใน ERD

| สัญลักษณ์ | ความหมาย |
|-----------|-----------|
| □ | Entity (ตาราง) |
| ◇ | Relationship |
| ○ | Attribute |
| ⚫ | Primary Key |

### ตัวอย่าง ERD ง่ายๆ
```
[User] ──1:M── [Order] ──M:N── [Product]
  │                              │
  │                              │
user_id                    product_id
email                      name
password                   price
```

### Relationship Types

#### 1. **One-to-One (1:1)**
```sql
-- หนึ่งผู้ใช้มีหนึ่งโปรไฟล์
User ←→ UserProfile

CREATE TABLE users (
    user_id INT PRIMARY KEY,
    email VARCHAR(255)
);

CREATE TABLE user_profiles (
    profile_id INT PRIMARY KEY,
    user_id INT UNIQUE,
    bio TEXT,
    FOREIGN KEY (user_id) REFERENCES users(user_id)
);
```

#### 2. **One-to-Many (1:M)**
```sql
-- หนึ่งหมวดหมู่มีหลายสินค้า
Category ←→ Product

CREATE TABLE categories (
    category_id INT PRIMARY KEY,
    name VARCHAR(255)
);

CREATE TABLE products (
    product_id INT PRIMARY KEY,
    name VARCHAR(255),
    category_id INT,
    FOREIGN KEY (category_id) REFERENCES categories(category_id)
);
```

#### 3. **Many-to-Many (M:N)**
```sql
-- หลายออเดอร์มีหลายสินค้า
Order ←→ Product

-- ต้องสร้างตารางกลาง
CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    user_id INT,
    order_date DATE
);

CREATE TABLE products (
    product_id INT PRIMARY KEY,
    name VARCHAR(255),
    price DECIMAL(10,2)
);

CREATE TABLE order_items (
    order_id INT,
    product_id INT,
    quantity INT,
    price DECIMAL(10,2),
    PRIMARY KEY (order_id, product_id),
    FOREIGN KEY (order_id) REFERENCES orders(order_id),
    FOREIGN KEY (product_id) REFERENCES products(product_id)
);
```

## Normalization

### ทำไมต้อง Normalize?
- ❌ **ก่อน**: ข้อมูลซ้ำซ้อน
- ✅ **หลัง**: ข้อมูลไม่ซ้ำ ประหยัดพื้นที่

### First Normal Form (1NF)
**กฎ**: แต่ละ cell มีค่าเดียว ไม่มี array

#### ❌ ไม่เป็น 1NF
```sql
-- ผิด: มีหลายค่าในช่องเดียว
orders
| order_id | customer | products           |
|----------|----------|--------------------|
| 1        | John     | Apple, Orange      |
| 2        | Jane     | Banana, Apple, Grape |
```

#### ✅ เป็น 1NF
```sql
-- ถูก: แยกเป็นแถวต่างหาก
order_items
| order_id | customer | product |
|----------|----------|---------|
| 1        | John     | Apple   |
| 1        | John     | Orange  |
| 2        | Jane     | Banana  |
| 2        | Jane     | Apple   |
```

### Second Normal Form (2NF)
**กฎ**: เป็น 1NF + ไม่มี partial dependency

#### ❌ ไม่เป็น 2NF
```sql
-- ผิด: customer_name ขึ้นกับแค่ order_id
order_items
| order_id | product_id | customer_name | product_name | quantity |
|----------|------------|---------------|--------------|----------|
| 1        | 101        | John          | Apple        | 2        |
| 1        | 102        | John          | Orange       | 1        |
```

#### ✅ เป็น 2NF
```sql
-- ถูก: แยกตาราง
orders
| order_id | customer_name |
|----------|---------------|
| 1        | John          |

order_items  
| order_id | product_id | quantity |
|----------|------------|----------|
| 1        | 101        | 2        |
| 1        | 102        | 1        |

products
| product_id | product_name |
|------------|-------------|
| 101        | Apple       |
| 102        | Orange      |
```

### Third Normal Form (3NF)
**กฎ**: เป็น 2NF + ไม่มี transitive dependency

#### ❌ ไม่เป็น 3NF
```sql
-- ผิด: category_name ขึ้นกับ category_id ซึ่งขึ้นกับ product_id
products
| product_id | product_name | category_id | category_name |
|------------|--------------|-------------|---------------|
| 1          | iPhone       | 1           | Electronics   |
| 2          | Samsung      | 1           | Electronics   |
```

#### ✅ เป็น 3NF
```sql
-- ถูก: แยกตาราง
products
| product_id | product_name | category_id |
|------------|--------------|-------------|
| 1          | iPhone       | 1           |
| 2          | Samsung      | 1           |

categories
| category_id | category_name |
|-------------|---------------|
| 1           | Electronics   |
```

## Data Types

### ข้อมูลตัวเลข

| Type | Size | Range | ใช้เมื่อไหร่ |
|------|------|-------|-------------|
| `TINYINT` | 1 byte | 0-255 | สถานะ, boolean |
| `INT` | 4 bytes | ±2 billion | ID, จำนวน |
| `BIGINT` | 8 bytes | ±9 quintillion | ID ขนาดใหญ่ |
| `DECIMAL(10,2)` | Variable | ทศนิยมแม่นยำ | เงิน, ราคา |
| `FLOAT` | 4 bytes | ทศนิยมประมาณ | คำนวณวิทยาศาสตร์ |

### ข้อมูลข้อความ

| Type | Size | ใช้เมื่อไหร่ |
|------|------|-------------|
| `CHAR(10)` | Fixed 10 chars | รหัสที่ความยาวคงที่ |
| `VARCHAR(255)` | Variable up to 255 | ชื่อ, email |
| `TEXT` | Up to 65KB | คำอธิบาย |
| `LONGTEXT` | Up to 4GB | เนื้อหายาว |

### ข้อมูลวันที่

| Type | Format | ใช้เมื่อไหร่ |
|------|---------|-------------|
| `DATE` | YYYY-MM-DD | วันเกิด |
| `TIME` | HH:MM:SS | เวลา |
| `DATETIME` | YYYY-MM-DD HH:MM:SS | วันเวลาสร้าง |
| `TIMESTAMP` | Auto update | วันเวลาอัพเดท |

### ตัวอย่างการเลือก Data Type
```sql
CREATE TABLE users (
    user_id BIGINT PRIMARY KEY AUTO_INCREMENT,
    email VARCHAR(255) NOT NULL UNIQUE,
    password VARCHAR(255) NOT NULL,
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    age TINYINT UNSIGNED,
    balance DECIMAL(10,2) DEFAULT 0,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

## Indexing

### Index คืออะไร?
**Index** เป็นโครงสร้างข้อมูลที่ช่วยให้ database หาข้อมูลได้เร็วขึ้น เหมือนดัชนีในหนังสือ

### ประเภท Index

#### 1. **Primary Index**
```sql
CREATE TABLE users (
    user_id INT PRIMARY KEY,  -- สร้าง primary index อัตโนมัติ
    email VARCHAR(255)
);
```

#### 2. **Unique Index**
```sql
-- ป้องกันข้อมูลซ้ำ
CREATE UNIQUE INDEX idx_email ON users(email);
```

#### 3. **Composite Index**
```sql
-- Index หลายคอลัมน์
CREATE INDEX idx_name ON users(first_name, last_name);
```

#### 4. **Partial Index**
```sql
-- Index เฉพาะข้อมูลที่ตรงเงื่อนไข
CREATE INDEX idx_active_users ON users(email) WHERE is_active = TRUE;
```

### เมื่อไหร่ควรใช้ Index?

#### ✅ ควรสร้าง Index เมื่อ:
```sql
-- คอลัมน์ที่ใช้ WHERE บ่อย
SELECT * FROM users WHERE email = 'john@example.com';
CREATE INDEX idx_email ON users(email);

-- คอลัมน์ที่ใช้ ORDER BY
SELECT * FROM products ORDER BY created_at DESC;
CREATE INDEX idx_created_at ON products(created_at);

-- Foreign Key
CREATE INDEX idx_user_id ON orders(user_id);
```

#### ❌ ไม่ควรสร้าง Index เมื่อ:
- ตารางมีข้อมูลน้อย (< 1000 แถว)
- คอลัมน์ที่ INSERT/UPDATE บ่อย
- คอลัมน์ที่มีค่าซ้ำเยอะ (เช่น gender)

## Constraints

### ประเภท Constraints

#### 1. **NOT NULL**
```sql
CREATE TABLE users (
    email VARCHAR(255) NOT NULL,  -- ห้ามเป็นค่าว่าง
    phone VARCHAR(20)             -- อนุญาตให้เป็นค่าว่าง
);
```

#### 2. **UNIQUE**
```sql
CREATE TABLE users (
    email VARCHAR(255) UNIQUE,    -- ห้ามซ้ำ
    username VARCHAR(50) UNIQUE
);
```

#### 3. **PRIMARY KEY**
```sql
CREATE TABLE users (
    user_id INT PRIMARY KEY,      -- NOT NULL + UNIQUE
    email VARCHAR(255)
);
```

#### 4. **FOREIGN KEY**
```sql
CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    user_id INT,
    FOREIGN KEY (user_id) REFERENCES users(user_id)
        ON DELETE CASCADE         -- ลบ user แล้วลบ order ด้วย
        ON UPDATE CASCADE         -- อัพเดท user_id แล้วอัพเดท order ด้วย
);
```

#### 5. **CHECK**
```sql
CREATE TABLE products (
    price DECIMAL(10,2) CHECK (price > 0),        -- ราคาต้องมากกว่า 0
    stock_quantity INT CHECK (stock_quantity >= 0), -- สต็อกไม่ติดลบ
    status ENUM('active', 'inactive', 'deleted')     -- จำกัดค่าที่เป็นไปได้
);
```

#### 6. **DEFAULT**
```sql
CREATE TABLE users (
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    role VARCHAR(20) DEFAULT 'user'
);
```

## Performance Considerations

### 1. **Query Optimization**

#### ใช้ EXPLAIN เพื่อวิเคราะห์
```sql
EXPLAIN SELECT * FROM users WHERE email = 'john@example.com';

-- ผลลัพธ์จะบอกว่า:
-- - ใช้ index หรือไม่
-- - scan กี่แถว
-- - ใช้เวลาเท่าไหร่
```

#### หลีกเลี่ยง SELECT *
```sql
-- ❌ ไม่ดี: ดึงข้อมูลทั้งหมด
SELECT * FROM users WHERE email = 'john@example.com';

-- ✅ ดี: ดึงเฉพาะที่ต้องการ
SELECT user_id, first_name, last_name FROM users WHERE email = 'john@example.com';
```

### 2. **Table Partitioning**
```sql
-- แบ่งตารางตามวันที่
CREATE TABLE orders (
    order_id BIGINT,
    user_id INT,
    order_date DATE,
    total DECIMAL(10,2)
) PARTITION BY RANGE (YEAR(order_date)) (
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025),
    PARTITION p2025 VALUES LESS THAN (2026)
);
```

### 3. **Denormalization เมื่อจำเป็น**
```sql
-- บางครั้งเก็บข้อมูลซ้ำเพื่อประสิทธิภาพ
CREATE TABLE order_summary (
    order_id INT,
    customer_name VARCHAR(255),  -- ซ้ำจาก users table
    total_amount DECIMAL(10,2),
    item_count INT               -- คำนวณจาก order_items
);
```

## 🎯 Best Practices

### 1. **การตั้งชื่อ**
```sql
-- ใช้ snake_case และชื่อที่สื่อความหมาย
✅ user_id, created_at, order_items
❌ UserId, createdAt, oi

-- ตารางใช้พหูพจน์
✅ users, products, orders
❌ user, product, order
```

### 2. **การออกแบบ Schema**
```sql
-- เก็บวันเวลาสร้างและอัพเดท
CREATE TABLE base_table (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

-- ใช้ soft delete แทนการลบจริง
ALTER TABLE users ADD COLUMN deleted_at TIMESTAMP NULL;
```

### 3. **Security**
```sql
-- สร้าง user เฉพาะสำหรับแอป
CREATE USER 'app_user'@'localhost' IDENTIFIED BY 'strong_password';
GRANT SELECT, INSERT, UPDATE, DELETE ON ecommerce.* TO 'app_user'@'localhost';

-- ไม่ให้สิทธิ์ DROP, ALTER ไปยัง application user
```

## 🚫 สิ่งที่ควรหลีกเลี่ยง

- ❌ ใช้ SELECT * ในโปรดักชัน
- ❌ ไม่มี index บน foreign key
- ❌ เก็บรูปภาพใน database
- ❌ ใช้ VARCHAR(255) สำหรับทุกอย่าง
- ❌ ไม่ตั้งชื่อ constraint ให้ชัดเจน
- ❌ ไม่ backup database
- ❌ ใช้ root user สำหรับ application

---

## 📚 แนวทางการเรียนรู้ต่อ

1. ✅ ฝึกออกแบบ ERD สำหรับ business ต่างๆ
2. ✅ เรียนรู้ SQL Query Optimization
3. ✅ ศึกษา Database Design Patterns
4. ✅ เรียนรู้ NoSQL เป็นทางเลือก
5. ✅ ฝึกใช้ Database Management Tools 