# Database Migration - การย้ายและเปลี่ยนแปลงโครงสร้างฐานข้อมูลอย่างปลอดภัย

## 📋 สารบัญ
- [Database Migration คืออะไร?](#database-migration-คืออะไร)
- [ทำไมต้องใช้ Migration?](#ทำไมต้องใช้-migration)
- [หลักการและแนวทางปฏิบัติ](#หลักการและแนวทางปฏิบัติ)
- [การเพิ่ม Column](#การเพิ่ม-column)
- [การลบ Column](#การลบ-column)
- [การเปลี่ยนชนิดข้อมูล](#การเปลี่ยนชนิดข้อมูล)
- [การเปลี่ยนชื่อ Table/Column](#การเปลี่ยนชื่อ-tablecolumn)
- [การจัดการ Index](#การจัดการ-index)
- [Zero-Downtime Migration](#zero-downtime-migration)
- [การ Rollback](#การ-rollback)
- [Best Practices](#best-practices)
- [ตัวอย่างจริง](#ตัวอย่างจริง)

## Database Migration คืออะไร?

**Database Migration** คือกระบวนการเปลี่ยนแปลงโครงสร้างฐานข้อมูล (Schema) อย่างเป็นระบบและปลอดภัย เช่น การเพิ่ม/ลบตาราง การเพิ่ม/ลบคอลัมน์ การเปลี่ยนชนิดข้อมูล หรือการปรับปรุง Index

### ประเภทของ Migration
```sql
-- 1. Schema Migration: เปลี่ยนโครงสร้าง
ALTER TABLE users ADD COLUMN phone VARCHAR(20);

-- 2. Data Migration: ย้ายข้อมูล
UPDATE users SET status = 'active' WHERE created_at > '2024-01-01';

-- 3. Seed Migration: เพิ่มข้อมูลเริ่มต้น
INSERT INTO roles (name, description) VALUES 
  ('admin', 'System Administrator'),
  ('user', 'Regular User');
```

## ทำไมต้องใช้ Migration?

### 🎯 ข้อดีของการใช้ Migration

#### 1. **Version Control สำหรับฐานข้อมูล**
```sql
-- migration_001_create_users_table.sql
CREATE TABLE users (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    email VARCHAR(255) NOT NULL UNIQUE,
    password VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- migration_002_add_user_profile.sql
ALTER TABLE users ADD COLUMN first_name VARCHAR(100);
ALTER TABLE users ADD COLUMN last_name VARCHAR(100);
```

#### 2. **การประสานงานทีม**
- ทีมทุกคนได้โครงสร้างฐานข้อมูลเหมือนกัน
- ป้องกันความผิดพลาดจากการแก้ไขแบบเฉพาะหน้า
- ติดตามการเปลี่ยนแปลงได้ชัดเจน

#### 3. **การ Deploy ที่ปลอดภัย**
- ทำการเปลี่ยนแปลงแบบเป็นขั้นตอน
- สามารถ Rollback ได้เมื่อเกิดปัญหา
- ลดความเสี่ยงการสูญหายของข้อมูล

## หลักการและแนวทางปฏิบัติ

### 🔧 เครื่องมือ Migration ยอดนิยม

#### **Golang Migrate**
```go
// migrations/000001_create_users_table.up.sql
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    email VARCHAR(255) NOT NULL UNIQUE,
    password VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);

// migrations/000001_create_users_table.down.sql
DROP TABLE IF EXISTS users;
```

#### **Flyway (Java/SQL)**
```sql
-- V1__Create_users_table.sql
CREATE TABLE users (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    email VARCHAR(255) NOT NULL UNIQUE,
    password VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- V2__Add_user_profile_columns.sql
ALTER TABLE users 
ADD COLUMN first_name VARCHAR(100),
ADD COLUMN last_name VARCHAR(100);
```

### 📝 หลักการตั้งชื่อ Migration

```bash
# รูปแบบ: timestamp_description
2024_01_01_000001_create_users_table
2024_01_01_000002_add_phone_to_users
2024_01_01_000003_create_orders_table
2024_01_01_000004_add_index_to_orders

# รูปแบบ Semantic Versioning
V1.0__Create_initial_schema.sql
V1.1__Add_user_profiles.sql
V2.0__Restructure_orders.sql
```

## การเพิ่ม Column

### 🟢 วิธีที่ปลอดภัย

#### **1. เพิ่ม Column แบบ Optional ก่อน**
```sql
-- ขั้นตอนที่ 1: เพิ่ม column ที่ NULL ได้
ALTER TABLE users ADD COLUMN phone VARCHAR(20) NULL;

-- ขั้นตอนที่ 2: อัพเดทข้อมูลที่มีอยู่ (ถ้าจำเป็น)
UPDATE users SET phone = '' WHERE phone IS NULL;

-- ขั้นตอนที่ 3: เปลี่ยนเป็น NOT NULL (ถ้าต้องการ)
ALTER TABLE users MODIFY COLUMN phone VARCHAR(20) NOT NULL DEFAULT '';
```

#### **2. เพิ่ม Column พร้อม Default Value**
```sql
-- ปลอดภัย: มี default value
ALTER TABLE users 
ADD COLUMN status ENUM('active', 'inactive') NOT NULL DEFAULT 'active';

-- ปลอดภัย: เพิ่ม timestamp
ALTER TABLE orders 
ADD COLUMN updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP;
```

### 🔴 สิ่งที่ควรหลีกเลี่ยง

```sql
-- อันตราย: เพิ่ม NOT NULL column ไม่มี default
ALTER TABLE users ADD COLUMN phone VARCHAR(20) NOT NULL;
-- ❌ จะ error ถ้ามีข้อมูลเดิม

-- อันตราย: เพิ่ม UNIQUE constraint ทันที
ALTER TABLE users ADD COLUMN username VARCHAR(50) UNIQUE;
-- ❌ อาจขัดแย้งกับข้อมูลที่มีอยู่
```

### 📋 ตัวอย่างการเพิ่ม Column แบบขั้นตอน

```sql
-- Migration 1: เพิ่ม column
ALTER TABLE products ADD COLUMN category_id INT NULL;

-- Migration 2: เพิ่มข้อมูล default
UPDATE products SET category_id = 1 WHERE category_id IS NULL;

-- Migration 3: เพิ่ม constraint
ALTER TABLE products MODIFY COLUMN category_id INT NOT NULL;
ALTER TABLE products ADD FOREIGN KEY (category_id) REFERENCES categories(id);

-- Migration 4: เพิ่ม index
CREATE INDEX idx_products_category_id ON products(category_id);
```

## การลบ Column

### 🟡 วิธีการลบอย่างปลอดภัย

#### **แบบ Multi-Step Deployment**

```sql
-- Step 1: ปรับ Application ไม่ให้ใช้ column นี้แล้ว
-- (Deploy application code ก่อน)

-- Step 2: ลบ constraints และ indexes
ALTER TABLE users DROP INDEX idx_users_old_column;
ALTER TABLE users DROP FOREIGN KEY fk_users_old_column;

-- Step 3: ลบ column
ALTER TABLE users DROP COLUMN old_column;
```

#### **การลบด้วย Backup**
```sql
-- 1. Backup ข้อมูลก่อน
CREATE TABLE users_backup AS SELECT * FROM users;

-- 2. หรือ backup เฉพาะ column ที่จะลบ
CREATE TABLE deleted_user_data AS 
SELECT user_id, old_column FROM users WHERE old_column IS NOT NULL;

-- 3. ลบ column
ALTER TABLE users DROP COLUMN old_column;
```

### 🔴 สิ่งที่ควรหลีกเลี่ยง

```sql
-- อันตราย: ลบ column ทันทีโดยไม่มี backup
ALTER TABLE users DROP COLUMN important_data;
-- ❌ ข้อมูลหายถาวร ไม่สามารถกู้คืนได้

-- อันตราย: ลบ column ที่ application ยังใช้อยู่
ALTER TABLE users DROP COLUMN email;
-- ❌ Application error ทันที
```

## การเปลี่ยนชนิดข้อมูล

### 🟢 วิธีที่ปลอดภัย

#### **1. การขยายขนาด (Safe)**
```sql
-- ปลอดภัย: VARCHAR เล็ก → ใหญ่
ALTER TABLE users MODIFY COLUMN username VARCHAR(100);  -- จาก VARCHAR(50)

-- ปลอดภัย: INT → BIGINT
ALTER TABLE counters MODIFY COLUMN count BIGINT;
```

#### **2. การลดขนาด (ต้องระวัง)**
```sql
-- ตรวจสอบข้อมูลก่อน
SELECT MAX(LENGTH(username)) FROM users;
-- ถ้าไม่เกิน 30 ตัวอักษร

-- ปลอดภัย: ลดขนาดได้
ALTER TABLE users MODIFY COLUMN username VARCHAR(30);
```

#### **3. การเปลี่ยนประเภทข้อมูล (ซับซ้อน)**
```sql
-- ขั้นตอนที่ 1: เพิ่ม column ใหม่
ALTER TABLE users ADD COLUMN age_int INT NULL;

-- ขั้นตอนที่ 2: แปลงข้อมูล
UPDATE users SET age_int = CAST(age_string AS UNSIGNED) 
WHERE age_string REGEXP '^[0-9]+$';

-- ขั้นตอนที่ 3: ตรวจสอบข้อมูล
SELECT COUNT(*) FROM users WHERE age_string IS NOT NULL AND age_int IS NULL;

-- ขั้นตอนที่ 4: เปลี่ยนชื่อ column
ALTER TABLE users DROP COLUMN age_string;
ALTER TABLE users CHANGE COLUMN age_int age INT;
```

### 🔄 Zero-Downtime Column Type Change

```sql
-- 1. เพิ่ม column ใหม่
ALTER TABLE products ADD COLUMN price_decimal DECIMAL(10,2) NULL;

-- 2. แปลงข้อมูลทีละส่วน
UPDATE products SET price_decimal = price_int / 100 
WHERE id BETWEEN 1 AND 1000;

UPDATE products SET price_decimal = price_int / 100 
WHERE id BETWEEN 1001 AND 2000;

-- 3. ปรับ application ให้ใช้ column ใหม่
-- (Deploy application code)

-- 4. ลบ column เก่า
ALTER TABLE products DROP COLUMN price_int;
ALTER TABLE products CHANGE COLUMN price_decimal price DECIMAL(10,2);
```

## การเปลี่ยนชื่อ Table/Column

### 🔄 การเปลี่ยนชื่อ Column

```sql
-- MySQL
ALTER TABLE users CHANGE COLUMN old_name new_name VARCHAR(255);

-- PostgreSQL  
ALTER TABLE users RENAME COLUMN old_name TO new_name;

-- SQL Server
EXEC sp_rename 'users.old_name', 'new_name', 'COLUMN';
```

### 🔄 การเปลี่ยนชื่อ Table

```sql
-- MySQL
RENAME TABLE old_table_name TO new_table_name;

-- PostgreSQL
ALTER TABLE old_table_name RENAME TO new_table_name;

-- SQL Server
EXEC sp_rename 'old_table_name', 'new_table_name';
```

### 📋 Zero-Downtime Table Rename

```sql
-- 1. สร้างตารางใหม่
CREATE TABLE users_new LIKE users;

-- 2. Copy ข้อมูล
INSERT INTO users_new SELECT * FROM users;

-- 3. สร้าง trigger เพื่อ sync ข้อมูลใหม่
DELIMITER $$
CREATE TRIGGER users_sync_insert 
AFTER INSERT ON users
FOR EACH ROW
BEGIN
    INSERT INTO users_new VALUES (NEW.id, NEW.email, NEW.created_at);
END$$
DELIMITER ;

-- 4. ปรับ application ให้ใช้ตารางใหม่
-- (Deploy application code)

-- 5. ลบตารางเก่าและ trigger
DROP TRIGGER users_sync_insert;
DROP TABLE users;
RENAME TABLE users_new TO users;
```

## การจัดการ Index

### 🟢 การเพิ่ม Index อย่างปลอดภัย

```sql
-- สร้าง index แบบ ONLINE (MySQL 5.6+)
ALTER TABLE users ADD INDEX idx_email (email) ALGORITHM=INPLACE, LOCK=NONE;

-- สร้าง index แบบ CONCURRENT (PostgreSQL)
CREATE INDEX CONCURRENTLY idx_users_email ON users(email);

-- ตรวจสอบสถานะการสร้าง index
SHOW PROCESSLIST;  -- MySQL
SELECT * FROM pg_stat_activity;  -- PostgreSQL
```

### 🔴 การลบ Index

```sql
-- ตรวจสอบการใช้งาน index ก่อน
SELECT 
    TABLE_NAME,
    INDEX_NAME,
    CARDINALITY,
    INDEX_TYPE
FROM information_schema.STATISTICS 
WHERE TABLE_SCHEMA = 'database_name' 
AND INDEX_NAME = 'idx_to_remove';

-- ลบ index
DROP INDEX idx_unnecessary ON users;
```

### 📊 การวิเคราะห์ Performance Index

```sql
-- MySQL: ตรวจสอบการใช้ index
SELECT 
    object_schema,
    object_name,
    index_name,
    count_read,
    count_write
FROM performance_schema.table_io_waits_summary_by_index_usage
WHERE object_schema = 'your_database'
ORDER BY count_read DESC;

-- PostgreSQL: ตรวจสอบ index ที่ไม่ใช้
SELECT 
    indexrelname as index_name,
    relname as table_name,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch
FROM pg_stat_user_indexes
WHERE idx_scan = 0;
```

## Zero-Downtime Migration

### 🎯 หลักการ Zero-Downtime

#### **1. Blue-Green Deployment**
```sql
-- 1. เตรียม database schema ใหม่
CREATE DATABASE app_blue;
CREATE DATABASE app_green;

-- 2. Migration บน environment ที่ไม่ได้ใช้งาน
USE app_green;
ALTER TABLE users ADD COLUMN phone VARCHAR(20);

-- 3. Sync ข้อมูลระหว่าง blue และ green
-- 4. Switch traffic ไป green
-- 5. ปิด blue environment
```

#### **2. Shadow Columns**
```sql
-- 1. เพิ่ม column ใหม่โดยไม่มีผลกับ app
ALTER TABLE users ADD COLUMN phone_new VARCHAR(20) NULL;

-- 2. ปรับ application ให้เขียนข้อมูลลง both columns
-- UPDATE users SET phone = ?, phone_new = ? WHERE id = ?

-- 3. Backfill ข้อมูลเก่า
UPDATE users SET phone_new = phone WHERE phone_new IS NULL;

-- 4. ปรับ application ให้อ่านจาก column ใหม่
-- 5. ลบ column เก่า
ALTER TABLE users DROP COLUMN phone;
ALTER TABLE users CHANGE COLUMN phone_new phone VARCHAR(20);
```

### 🔄 Online Schema Change Tools

#### **pt-online-schema-change (Percona)**
```bash
# เปลี่ยน schema โดยไม่ lock table
pt-online-schema-change \
  --alter "ADD COLUMN phone VARCHAR(20)" \
  --execute \
  --host=localhost \
  --user=root \
  --ask-pass \
  D=mydb,t=users
```

#### **gh-ost (GitHub)**
```bash
# GitHub's online schema migration
gh-ost \
  --user="root" \
  --password="password" \
  --host="localhost" \
  --database="mydb" \
  --table="users" \
  --alter="ADD COLUMN phone VARCHAR(20)" \
  --execute
```

## การ Rollback

### 🔙 การเตรียม Rollback Plan

#### **1. สร้าง Reverse Migration**
```sql
-- Forward Migration (Up)
-- V001__add_phone_column.sql
ALTER TABLE users ADD COLUMN phone VARCHAR(20);

-- Reverse Migration (Down)  
-- V001__add_phone_column.down.sql
ALTER TABLE users DROP COLUMN phone;
```

#### **2. Backup Strategy**
```sql
-- 1. Full backup ก่อน migration
mysqldump --single-transaction --lock-tables=false mydb > backup_before_migration.sql

-- 2. Incremental backup สำหรับข้อมูลใหม่
-- 3. Point-in-time recovery plan
```

### 🚨 Emergency Rollback

```sql
-- 1. หยุด application ทันที
-- 2. Restore จาก backup
mysql mydb < backup_before_migration.sql

-- 3. หรือ manual rollback (ถ้าทำได้)
ALTER TABLE users DROP COLUMN problematic_column;

-- 4. เริ่ม application version เก่า
-- 5. วิเคราะห์ปัญหาและแก้ไข
```

### 📋 Rollback Checklist

```markdown
## Pre-Migration Checklist
- [ ] Full database backup
- [ ] Rollback plan เขียนไว้
- [ ] Test migration บน staging
- [ ] Application compatible กับ schema ใหม่และเก่า
- [ ] Monitor dashboard พร้อม
- [ ] Team standby

## Post-Migration Checklist  
- [ ] ตรวจสอบ application logs
- [ ] ตรวจสอบ database performance
- [ ] ทดสอบ key features
- [ ] Monitor error rates
- [ ] ทำความสะอาด backup เก่า
```

## Best Practices

### 🎯 Golden Rules

#### **1. Always Test First**
```sql
-- 1. Test บน development environment
-- 2. Test บน staging environment ที่เหมือน production
-- 3. Load test ด้วยข้อมูลจริง
-- 4. ทดสอบ rollback procedure
```

#### **2. Small, Incremental Changes**
```sql
-- ❌ ไม่ดี: เปลี่ยนทุกอย่างในครั้งเดียว
ALTER TABLE users 
  ADD COLUMN phone VARCHAR(20),
  ADD COLUMN address TEXT,
  CHANGE COLUMN email email_address VARCHAR(255),
  DROP COLUMN old_field;

-- ✅ ดี: แยกเป็น migrations หลายอัน
-- Migration 1:
ALTER TABLE users ADD COLUMN phone VARCHAR(20);

-- Migration 2:  
ALTER TABLE users ADD COLUMN address TEXT;

-- Migration 3:
ALTER TABLE users ADD COLUMN email_address VARCHAR(255);
UPDATE users SET email_address = email;

-- Migration 4:
ALTER TABLE users DROP COLUMN email;
```

#### **3. Backward Compatibility**
```sql
-- ระหว่าง deploy application ใหม่
-- database ต้องทำงานกับ app version เก่าได้

-- ✅ ดี: เพิ่ม column ที่ optional
ALTER TABLE users ADD COLUMN phone VARCHAR(20) NULL;

-- ❌ ไม่ดี: เปลี่ยน column ที่ app ใช้อยู่
ALTER TABLE users CHANGE COLUMN email email_address VARCHAR(255);
```

### 📊 Performance Considerations

#### **1. ทำใน Maintenance Window**
```sql
-- Migration ที่ใช้เวลานาน ควรทำตอนไม่มี traffic
-- - การเพิ่ม index ใน table ใหญ่
-- - การเปลี่ยน column type
-- - การ rebuild table
```

#### **2. Monitor Resource Usage**
```sql
-- ตรวจสอบ disk space
SELECT 
    table_schema,
    table_name,
    ROUND(((data_length + index_length) / 1024 / 1024), 2) AS table_size_mb
FROM information_schema.tables
WHERE table_schema = 'your_database'
ORDER BY table_size_mb DESC;

-- ตรวจสอบ memory usage
SHOW VARIABLES LIKE 'innodb_buffer_pool_size';
```

#### **3. Batch Processing**
```sql
-- ทำการอัพเดทข้อมูลทีละส่วน
UPDATE users SET status = 'migrated' 
WHERE id BETWEEN 1 AND 1000;

-- รอให้ระบบพัก
SELECT SLEEP(1);

UPDATE users SET status = 'migrated' 
WHERE id BETWEEN 1001 AND 2000;
```

### 🔒 Security Best Practices

```sql
-- 1. ใช้ separate user สำหรับ migration
CREATE USER 'migration_user'@'localhost' IDENTIFIED BY 'strong_password';
GRANT ALTER, CREATE, DROP, INDEX ON mydb.* TO 'migration_user'@'localhost';

-- 2. Log ทุกการเปลี่ยนแปลง
-- 3. Encrypt backup files
-- 4. ใช้ SSL connection
```

## ตัวอย่างจริง

### 🛍️ ตัวอย่าง: ระบบ E-commerce

#### **เพิ่ม Column สำหรับ Product Variants**

```sql
-- Migration 1: เพิ่ม variant columns
ALTER TABLE products ADD COLUMN has_variants BOOLEAN DEFAULT FALSE;
ALTER TABLE products ADD COLUMN variant_type ENUM('size', 'color', 'size_color') NULL;

-- Migration 2: สร้าง product_variants table
CREATE TABLE product_variants (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    product_id BIGINT NOT NULL,
    sku VARCHAR(100) NOT NULL UNIQUE,
    size VARCHAR(20),
    color VARCHAR(50),
    price_adjustment DECIMAL(10,2) DEFAULT 0,
    stock_quantity INT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (product_id) REFERENCES products(id) ON DELETE CASCADE,
    INDEX idx_product_variants_product_id (product_id),
    INDEX idx_product_variants_sku (sku)
);

-- Migration 3: ย้ายข้อมูลเก่า
INSERT INTO product_variants (product_id, sku, stock_quantity)
SELECT id, CONCAT('SKU-', id), stock_quantity 
FROM products 
WHERE has_variants = FALSE;

-- Migration 4: อัพเดท application เพื่อใช้ variants
-- (Deploy new application version)

-- Migration 5: ลบ column เก่า (ถ้าจำเป็น)
ALTER TABLE products DROP COLUMN stock_quantity;
```

### 👥 ตัวอย่าง: ระบบ User Management

#### **เปลี่ยนจาก username เป็น email login**

```sql
-- Migration 1: เพิ่ม email column
ALTER TABLE users ADD COLUMN email VARCHAR(255) NULL;
ALTER TABLE users ADD COLUMN email_verified_at TIMESTAMP NULL;

-- Migration 2: เพิ่มข้อมูล email จาก username
UPDATE users 
SET email = CONCAT(username, '@company.com') 
WHERE email IS NULL AND username NOT LIKE '%@%';

UPDATE users 
SET email = username 
WHERE email IS NULL AND username LIKE '%@%';

-- Migration 3: เพิ่ม unique constraint
CREATE UNIQUE INDEX idx_users_email_unique ON users(email);

-- Migration 4: ปรับ application ให้ support ทั้ง username และ email
-- (Deploy application ที่รองรับทั้งสองแบบ)

-- Migration 5: บังคับให้ user ต้องมี email
ALTER TABLE users MODIFY COLUMN email VARCHAR(255) NOT NULL;

-- Migration 6: ปรับ application ให้ใช้เฉพาะ email
-- (Deploy application ที่ใช้เฉพาะ email)

-- Migration 7: ลบ username column (optional)
ALTER TABLE users DROP COLUMN username;
```

### 📊 ตัวอย่าง: Performance Optimization

#### **เพิ่ม Partitioning สำหรับ Orders Table**

```sql
-- Migration 1: สร้าง table ใหม่แบบ partitioned
CREATE TABLE orders_new (
    id BIGINT AUTO_INCREMENT,
    user_id BIGINT NOT NULL,
    total_amount DECIMAL(10,2) NOT NULL,
    status ENUM('pending', 'paid', 'shipped', 'delivered') NOT NULL,
    order_date DATE NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (id, order_date),
    INDEX idx_orders_user_id (user_id),
    INDEX idx_orders_status (status)
) PARTITION BY RANGE (YEAR(order_date)) (
    PARTITION p2022 VALUES LESS THAN (2023),
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025),
    PARTITION p2025 VALUES LESS THAN (2026),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);

-- Migration 2: Copy ข้อมูลทีละส่วน
INSERT INTO orders_new 
SELECT * FROM orders 
WHERE order_date >= '2024-01-01'
LIMIT 10000;

-- Migration 3: สร้าง trigger เพื่อ sync ข้อมูลใหม่
DELIMITER $$
CREATE TRIGGER orders_sync AFTER INSERT ON orders
FOR EACH ROW
BEGIN
    INSERT INTO orders_new VALUES (
        NEW.id, NEW.user_id, NEW.total_amount, 
        NEW.status, NEW.order_date, NEW.created_at
    );
END$$
DELIMITER ;

-- Migration 4: เปลี่ยน application ให้อ่านจาก orders_new
-- (Deploy new application version)

-- Migration 5: ลบ table เก่า
DROP TRIGGER orders_sync;
DROP TABLE orders;
RENAME TABLE orders_new TO orders;
```

### 🔙 ตัวอย่าง: การ Rollback เมื่อเกิดปัญหา

#### **Scenario 1: Rollback การเพิ่ม Column ที่มีปัญหา**

```sql
-- สถานการณ์: เพิ่ม phone column แต่ app crash เพราะ validation
-- Migration ที่ทำไป:
ALTER TABLE users ADD COLUMN phone VARCHAR(20) NOT NULL DEFAULT '';

-- 🚨 Problem: Application error เพราะ validation ที่ไม่คาดคิด

-- Rollback Plan:
-- 1. หยุด application ทันที
-- 2. ลบ column ที่เพิ่งเพิ่ม
ALTER TABLE users DROP COLUMN phone;

-- 3. เริ่ม application version เก่า
-- 4. แก้ไข validation logic ใน code
-- 5. ทำ migration ใหม่ด้วย column เป็น NULL ก่อน
ALTER TABLE users ADD COLUMN phone VARCHAR(20) NULL;
```

#### **Scenario 2: Rollback การเปลี่ยนชนิดข้อมูลที่ล้มเหลว**

```sql
-- สถานการณ์: เปลี่ยน price จาก INT เป็น DECIMAL แต่มีข้อมูลผิดพลาด
-- Migration ที่ทำไป:
ALTER TABLE products ADD COLUMN price_decimal DECIMAL(10,2);
UPDATE products SET price_decimal = price_int / 100;

-- 🚨 Problem: พบข้อมูล price_int มีค่าลบที่ไม่ควรมี

-- Emergency Rollback:
-- 1. Restore จาก backup (ถ้ามี)
mysql mydb < backup_before_migration.sql;

-- หรือ Manual Rollback:
-- 2. ลบ column ใหม่
ALTER TABLE products DROP COLUMN price_decimal;

-- 3. แก้ไขข้อมูลที่ผิดพลาด
UPDATE products SET price_int = ABS(price_int) WHERE price_int < 0;

-- 4. ทำ migration ใหม่พร้อม validation
ALTER TABLE products ADD COLUMN price_decimal DECIMAL(10,2);
UPDATE products 
SET price_decimal = price_int / 100 
WHERE price_int >= 0;
```

#### **Scenario 3: Rollback การ Partitioning ที่ทำให้ Performance แย่ลง**

```sql
-- สถานการณ์: แบ่ง partition แต่ query performance แย่ลง
-- Migration ที่ทำไป: สร้าง orders_partitioned และย้ายข้อมูล

-- 🚨 Problem: Query ช้าลง เพราะ query ข้าม multiple partitions

-- Rollback Strategy:
-- 1. หยุดการเขียนข้อมูลใหม่ชั่วคราว
UPDATE application_config SET maintenance_mode = TRUE;

-- 2. สร้าง table แบบเดิม (ไม่มี partition)
CREATE TABLE orders_restored (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT NOT NULL,
    total_amount DECIMAL(10,2) NOT NULL,
    status ENUM('pending', 'paid', 'shipped', 'delivered') NOT NULL,
    order_date DATE NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_orders_user_id (user_id),
    INDEX idx_orders_status (status),
    INDEX idx_orders_date (order_date)
);

-- 3. Copy ข้อมูลกลับจาก partitioned table
INSERT INTO orders_restored SELECT * FROM orders_partitioned;

-- 4. Backup partitioned table และ drop
RENAME TABLE orders_partitioned TO orders_partitioned_backup;
RENAME TABLE orders_restored TO orders;

-- 5. เริ่ม application ปกติ
UPDATE application_config SET maintenance_mode = FALSE;

-- 6. วิเคราะห์ปัญหาและปรับปรุง partition strategy
```

#### **Scenario 4: Rollback การ Migration ที่ Corrupt ข้อมูล**

```sql
-- สถานการณ์: Migration script มี bug ทำให้ข้อมูลเพี้ยน
-- Migration ที่ทำไป:
UPDATE users SET status = 'active' WHERE last_login > '2024-01-01';
-- 🚨 Bug: ควรเป็น >= แต่เขียนเป็น >

-- Emergency Response:
-- 1. หยุด application ทันที
sudo systemctl stop myapp

-- 2. Assess damage
SELECT COUNT(*) FROM users WHERE status = 'active';
SELECT COUNT(*) FROM users WHERE last_login = '2024-01-01' AND status != 'active';

-- 3. Point-in-time recovery จาก backup
-- ใช้ backup ก่อนหน้า migration + binary log replay
mysqlbinlog --start-datetime="2024-01-01 08:00:00" \
            --stop-datetime="2024-01-01 09:30:00" \
            mysql-bin.000001 | mysql mydb

-- 4. Manual data correction (ถ้าทำได้)
UPDATE users SET status = 'active' 
WHERE last_login >= '2024-01-01' AND status != 'active';

-- 5. Verify data integrity
SELECT 
    status, 
    COUNT(*),
    MIN(last_login),
    MAX(last_login)
FROM users 
GROUP BY status;
```

#### **Scenario 5: Rollback Index ที่ทำให้ระบบช้า**

```sql
-- สถานการณ์: เพิ่ม composite index แต่ทำให้ INSERT/UPDATE ช้ามาก
-- Migration ที่ทำไป:
CREATE INDEX idx_users_complex ON users(email, status, created_at, last_login);

-- 🚨 Problem: Write operations ช้าลง 300%

-- Quick Rollback:
-- 1. ลบ index ทันที
DROP INDEX idx_users_complex ON users;

-- 2. Monitor performance recovery
SHOW PROCESSLIST;
SELECT * FROM INFORMATION_SCHEMA.INNODB_TRX;

-- 3. วิเคราะห์และสร้าง index ที่เหมาะสมกว่า
-- อาจแยกเป็น 2-3 index แทน
CREATE INDEX idx_users_email_status ON users(email, status);
CREATE INDEX idx_users_created_at ON users(created_at);
```

### 🛡️ Rollback Best Practices

#### **1. Pre-Migration Preparation**
```bash
#!/bin/bash
# migration-script.sh

# 1. สร้าง backup ก่อน migration
mysqldump --single-transaction --routines --triggers mydb > "backup_$(date +%Y%m%d_%H%M%S).sql"

# 2. เตรียม rollback script
cat > rollback.sql << EOF
-- Rollback commands here
ALTER TABLE users DROP COLUMN phone;
EOF

# 3. Test rollback script
mysql test_db < rollback.sql
```

#### **2. Monitoring During Migration**
```sql
-- ตรวจสอบ performance metrics
SELECT 
    VARIABLE_NAME,
    VARIABLE_VALUE
FROM INFORMATION_SCHEMA.GLOBAL_STATUS
WHERE VARIABLE_NAME IN (
    'Innodb_buffer_pool_read_requests',
    'Innodb_buffer_pool_reads',
    'Slow_queries',
    'Threads_running'
);

-- ตรวจสอบ lock ที่รอ
SELECT 
    r.trx_id waiting_trx_id,
    r.trx_mysql_thread_id waiting_thread,
    r.trx_query waiting_query,
    b.trx_id blocking_trx_id,
    b.trx_mysql_thread_id blocking_thread,
    b.trx_query blocking_query
FROM INFORMATION_SCHEMA.INNODB_LOCK_WAITS w
INNER JOIN INFORMATION_SCHEMA.INNODB_TRX b ON b.trx_id = w.blocking_trx_id
INNER JOIN INFORMATION_SCHEMA.INNODB_TRX r ON r.trx_id = w.requesting_trx_id;
```

#### **3. Post-Rollback Verification**
```sql
-- ตรวจสอบ data integrity หลัง rollback
SELECT 
    TABLE_NAME,
    TABLE_ROWS,
    DATA_LENGTH,
    INDEX_LENGTH
FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_SCHEMA = 'mydb'
ORDER BY TABLE_NAME;

-- ตรวจสอบ constraints
SELECT 
    CONSTRAINT_NAME,
    TABLE_NAME,
    CONSTRAINT_TYPE
FROM INFORMATION_SCHEMA.TABLE_CONSTRAINTS
WHERE TABLE_SCHEMA = 'mydb'
AND CONSTRAINT_TYPE IN ('PRIMARY KEY', 'FOREIGN KEY', 'UNIQUE');
```

## 🚀 สรุป

การทำ Database Migration ที่ดีต้อง:

### ✅ DO
- **วางแผน** ทุกขั้นตอนและทำ backup
- **ทดสอบ** บน environment ที่เหมือน production
- **ทำทีละขั้นตอน** แทนการเปลี่ยนทุกอย่างในครั้งเดียว
- **เตรียม rollback plan** ไว้เสมอ
- **Monitor** ระหว่างและหลัง migration
- **Backward compatible** ระหว่างการ deploy

### ❌ DON'T
- อย่าลบข้อมูลโดยไม่มี backup
- อย่าทำ migration ใหญ่ๆ ใน peak time
- อย่าข้าม testing environment
- อย่าเปลี่ยนแปลงที่ทำให้ app เก่า error
- อย่าลืมสื่อสารกับทีม

Migration ที่ดีจะช่วยให้ระบบพัฒนาได้อย่างมั่นคงและปลอดภัย! 🎯 