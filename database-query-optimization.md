# การปรับปรุงประสิทธิภาพของ Database Query

## สารบัญ (Table of Contents)

### 🔰 **พื้นฐาน (Beginner)**
1. [การติดตาม Query Performance](#1-การติดตาม-query-performance)
2. [การเลือกใช้ SELECT อย่างรอบคอบ](#2-การเลือกใช้-select-อย่างรอบคอบ)
3. [การใช้ WHERE Clause อย่างมีประสิทธิภาพ](#3-การใช้-where-clause-อย่างมีประสิทธิภาพ)

### 📚 **ระดับกลาง (Intermediate)**
4. [การใช้ Index อย่างมีประสิทธิภาพ](#4-การใช้-index-อย่างมีประสิทธิภาพ)
5. [การใช้ JOIN อย่างเหมาะสม](#5-การใช้-join-อย่างเหมาะสม)
6. [การใช้ LIMIT และ OFFSET อย่างเหมาะสม](#6-การใช้-limit-และ-offset-อย่างเหมาะสม)
7. [การใช้ Aggregate Functions อย่างชาญฉลาด](#7-การใช้-aggregate-functions-อย่างชาญฉลาด)
8. [การใช้ Query Caching](#8-การใช้-query-caching)

### 🎓 **ระดับสูง (Advanced)**
9. [Common Table Expressions (WITH Clause)](#9-common-table-expressions-with-clause)
10. [Window Functions สำหรับการวิเคราะห์ข้อมูล](#10-window-functions-สำหรับการวิเคราะห์ข้อมูล)
11. [การจัดการ Temporary Tables และ Memory](#11-การจัดการ-temporary-tables-และ-memory)
12. [Bulk Operations และ Batch Processing](#12-bulk-operations-และ-batch-processing)

### 🔧 **ผู้เชี่ยวชาญ (Expert)**
13. [การปรับปรุง Database Schema](#13-การปรับปรุง-database-schema)
14. [การใช้ Query Hints อย่างเหมาะสม](#14-การใช้-query-hints-อย่างเหมาะสม)
15. [Best Practices สรุป](#15-best-practices-สรุป)
16. [สรุป](#สรุป)

---

## 1. การติดตาม Query Performance

### การใช้ EXPLAIN:
```sql
-- ตรวจสอบ Execution Plan
EXPLAIN SELECT * FROM orders 
WHERE customer_id = 123 
  AND order_date BETWEEN '2024-01-01' AND '2024-01-31';

-- ตรวจสอบแบบละเอียด
EXPLAIN FORMAT=JSON 
SELECT * FROM orders 
WHERE customer_id = 123 
  AND order_date BETWEEN '2024-01-01' AND '2024-01-31';
```

### การใช้ Slow Query Log:
```sql
-- เปิด Slow Query Log
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1; -- Log queries ที่ใช้เวลามากกว่า 1 วินาที

-- ตรวจสอบ Slow Queries
SELECT * FROM mysql.slow_log 
WHERE start_time > DATE_SUB(NOW(), INTERVAL 1 DAY)
ORDER BY query_time DESC;
```

### การวัดประสิทธิภาพ:
```go
package main

import (
    "database/sql"
    "encoding/json"
    "fmt"
    "log"
    "runtime"
    "time"
    
    _ "github.com/go-sql-driver/mysql"
)

// Constants
const (
    BytesToMB           = 1024 * 1024
    SecondsDecimalPlace = 4
    MBDecimalPlace      = 2
    PerformanceDecimalPlace = 2
)

// Error constants
const (
    ErrQueryExecution = "query execution failed"
    ErrGetColumns     = "failed to get columns"
    ErrScanFailed     = "scan failed"
    ErrDatabaseConn   = "failed to connect to database"
)

// Interfaces for better testability
type DatabaseExecutor interface {
    Query(query string, args ...interface{}) (*sql.Rows, error)
}

type PerformanceLogger interface {
    LogPerformance(result *QueryResult)
}

// Structs
type QueryParams struct {
    Query string
    Args  []interface{}
}

type QueryResult struct {
    Data          []map[string]interface{} `json:"data"`
    ExecutionTime float64                  `json:"execution_time"`
    MemoryUsed    int64                    `json:"memory_used"`
    RowsReturned  int                      `json:"rows_returned"`
}

type PerformanceMeasurer struct {
    DB     DatabaseExecutor
    Logger PerformanceLogger
}

type ConsoleLogger struct{}

func (cl *ConsoleLogger) LogPerformance(result *QueryResult) {
    fmt.Printf("Execution time: %.*f seconds\n", SecondsDecimalPlace, result.ExecutionTime)
    fmt.Printf("Memory used: %.*f MB\n", MBDecimalPlace, float64(result.MemoryUsed)/BytesToMB)
    fmt.Printf("Rows returned: %d\n", result.RowsReturned)
    
    if result.ExecutionTime > 0 {
        performanceScore := float64(result.RowsReturned) / result.ExecutionTime
        fmt.Printf("Performance score: %.*f rows/sec\n", PerformanceDecimalPlace, performanceScore)
    }
}

func NewPerformanceMeasurer(db DatabaseExecutor) *PerformanceMeasurer {
    return &PerformanceMeasurer{
        DB:     db,
        Logger: &ConsoleLogger{},
    }
}

func (pm *PerformanceMeasurer) MeasureQueryPerformance(params QueryParams) (*QueryResult, error) {
    memBefore := pm.captureMemoryStats()
    startTime := time.Now()
    
    rows, err := pm.DB.Query(params.Query, params.Args...)
    if err != nil {
        return nil, fmt.Errorf("%s: %w", ErrQueryExecution, err)
    }
    defer rows.Close()
    
    results, err := pm.processRows(rows)
    if err != nil {
        return nil, err
    }
    
    result := pm.buildQueryResult(results, startTime, memBefore)
    pm.Logger.LogPerformance(result)
    
    return result, nil
}

func (pm *PerformanceMeasurer) captureMemoryStats() runtime.MemStats {
    var memStats runtime.MemStats
    runtime.GC()
    runtime.ReadMemStats(&memStats)
    return memStats
}

func (pm *PerformanceMeasurer) processRows(rows *sql.Rows) ([]map[string]interface{}, error) {
    columns, err := rows.Columns()
    if err != nil {
        return nil, fmt.Errorf("%s: %w", ErrGetColumns, err)
    }
    
    var results []map[string]interface{}
    for rows.Next() {
        row, err := pm.scanRow(rows, columns)
        if err != nil {
            return nil, err
        }
        results = append(results, row)
    }
    
    return results, nil
}

func (pm *PerformanceMeasurer) scanRow(rows *sql.Rows, columns []string) (map[string]interface{}, error) {
    values := make([]interface{}, len(columns))
    valuePtrs := make([]interface{}, len(columns))
    for i := range values {
        valuePtrs[i] = &values[i]
    }
    
    if err := rows.Scan(valuePtrs...); err != nil {
        return nil, fmt.Errorf("%s: %w", ErrScanFailed, err)
    }
    
    row := make(map[string]interface{})
    for i, col := range columns {
        row[col] = values[i]
    }
    
    return row, nil
}

func (pm *PerformanceMeasurer) buildQueryResult(results []map[string]interface{}, 
    startTime time.Time, memBefore runtime.MemStats) *QueryResult {
    
    executionTime := time.Since(startTime).Seconds()
    
    var memAfter runtime.MemStats
    runtime.ReadMemStats(&memAfter)
    memoryUsed := int64(memAfter.TotalAlloc - memBefore.TotalAlloc)
    
    return &QueryResult{
        Data:          results,
        ExecutionTime: executionTime,
        MemoryUsed:    memoryUsed,
        RowsReturned:  len(results),
    }
}

// ตัวอย่างการใช้งาน
func main() {
    db, err := sql.Open("mysql", "user:password@tcp(localhost:3306)/database")
    if err != nil {
        log.Fatalf("%s: %v", ErrDatabaseConn, err)
    }
    defer db.Close()
    
    measurer := NewPerformanceMeasurer(db)
    
    params := QueryParams{
        Query: "SELECT customer_id, SUM(total_amount) FROM orders WHERE order_date >= ? GROUP BY customer_id",
        Args:  []interface{}{"2024-01-01"},
    }
    
    result, err := measurer.MeasureQueryPerformance(params)
    if err != nil {
        log.Fatal(err)
    }
    
    jsonResult, _ := json.MarshalIndent(result, "", "  ")
    fmt.Printf("Result: %s\n", jsonResult)
}
```

### เครื่องมือสำหรับ Performance Monitoring:
```sql
-- MySQL: Performance Schema
SELECT 
    DIGEST_TEXT,
    COUNT_STAR,
    AVG_TIMER_WAIT/1000000000 as avg_seconds,
    SUM_TIMER_WAIT/1000000000 as total_seconds
FROM performance_schema.events_statements_summary_by_digest
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 10;

-- ตรวจสอบ Index Usage
SELECT 
    object_schema,
    object_name,
    index_name,
    count_read,
    count_write,
    count_fetch,
    count_insert,
    count_update,
    count_delete
FROM performance_schema.table_io_waits_summary_by_index_usage
WHERE object_schema = 'your_database'
ORDER BY count_read DESC;
```

## 2. การเลือกใช้ SELECT อย่างรอบคอบ

### ปัญหา: การใช้ SELECT *
```sql
-- ไม่ดี: ดึงข้อมูลทั้งหมดที่ไม่จำเป็น
SELECT * FROM orders 
WHERE customer_id = 123;
```

### วิธีแก้ไข:
```sql
-- ดี: ระบุเฉพาะ columns ที่ต้องการ
SELECT order_id, order_date, total_amount 
FROM orders 
WHERE customer_id = 123;
```

### ตัวอย่างจริง - ระบบ CRM:
```sql
-- ปัญหา: API ที่ดึงข้อมูลลูกค้าใช้ SELECT *
SELECT * FROM customers WHERE status = 'active';

-- แก้ไข: ดึงเฉพาะข้อมูลที่จำเป็นสำหรับการแสดงผล
SELECT customer_id, first_name, last_name, email, phone 
FROM customers 
WHERE status = 'active';
```

## 3. การใช้ WHERE Clause อย่างมีประสิทธิภาพ

### การใช้ LIKE อย่างผิดวิธี:
```sql
-- ช้า: LIKE ที่ขึ้นต้นด้วย %
SELECT * FROM products WHERE product_name LIKE '%phone%';

-- เร็วกว่า: LIKE ที่ไม่ขึ้นต้นด้วย %
SELECT * FROM products WHERE product_name LIKE 'iPhone%';
```

### ตัวอย่างจริง - ระบบค้นหา:
```sql
-- ปัญหา: การค้นหาแบบ Full-text ที่ไม่มีประสิทธิภาพ
SELECT * FROM articles 
WHERE title LIKE '%web development%' 
   OR content LIKE '%web development%';

-- แก้ไข: ใช้ Full-text Search Index
-- สร้าง Full-text Index
CREATE FULLTEXT INDEX idx_articles_search ON articles(title, content);

-- ใช้ Full-text Search
SELECT * FROM articles 
WHERE MATCH(title, content) AGAINST('web development' IN NATURAL LANGUAGE MODE);
```

## 4. การใช้ Index อย่างมีประสิทธิภาพ

### ปัญหาที่พบบ่อย:
```sql
-- Query ที่ช้า: ไม่มี Index
SELECT * FROM users WHERE email = 'john@example.com';
```

### วิธีแก้ไข:
```sql
-- สร้าง Index สำหรับ email column
CREATE INDEX idx_users_email ON users(email);

-- หรือใช้ Unique Index หากต้องการความเป็นเอกลักษณ์
CREATE UNIQUE INDEX idx_users_email_unique ON users(email);
```

### ตัวอย่างการใช้งานจริง - ระบบ E-commerce:
```sql
-- ปัญหา: การค้นหาสินค้าตาม category_id ช้า
SELECT product_id, product_name, price 
FROM products 
WHERE category_id = 5 
ORDER BY created_at DESC 
LIMIT 10;

-- แก้ไข: สร้าง Composite Index
CREATE INDEX idx_products_category_created ON products(category_id, created_at DESC);
```

### เมื่อไหร่ควรสร้าง Index:
```sql
-- ✅ ควรสร้าง Index สำหรับ:
-- 1. Columns ที่ใช้ใน WHERE clause บ่อยๆ
CREATE INDEX idx_orders_status ON orders(status);

-- 2. Foreign Key columns
CREATE INDEX idx_order_items_order_id ON order_items(order_id);

-- 3. Columns ที่ใช้ใน ORDER BY
CREATE INDEX idx_products_created_at ON products(created_at);

-- 4. Composite Index สำหรับ queries ที่ใช้หลาย columns
CREATE INDEX idx_orders_customer_date ON orders(customer_id, order_date);
```

### ข้อควรระวังเรื่อง Index:
```sql
-- ❌ ไม่ควรสร้าง Index มากเกินไป
-- การ INSERT/UPDATE/DELETE จะช้าลง

-- ตรวจสอบ Index ที่ไม่ได้ใช้
SELECT 
    s.schemaname,
    s.tablename,
    s.indexname,
    s.idx_tup_read,
    s.idx_tup_fetch
FROM pg_stat_user_indexes s
WHERE s.idx_tup_read = 0 AND s.idx_tup_fetch = 0;

-- ลบ Index ที่ไม่ได้ใช้
DROP INDEX IF EXISTS unused_index_name;
```

## 5. การใช้ JOIN อย่างเหมาะสม

### ปัญหา: การใช้ Subquery แทน JOIN
```sql
-- ช้า: ใช้ Subquery
SELECT p.product_name, p.price
FROM products p
WHERE p.category_id IN (
    SELECT c.category_id 
    FROM categories c 
    WHERE c.status = 'active'
);
```

### วิธีแก้ไข:
```sql
-- เร็วกว่า: ใช้ INNER JOIN
SELECT p.product_name, p.price
FROM products p
INNER JOIN categories c ON p.category_id = c.category_id
WHERE c.status = 'active';
```

### ตัวอย่างจริง - ระบบการขาย:
```sql
-- ปัญหา: รายงานยอดขายใช้ Multiple Subqueries
SELECT 
    s.sales_id,
    (SELECT customer_name FROM customers WHERE customer_id = s.customer_id) as customer_name,
    (SELECT product_name FROM products WHERE product_id = s.product_id) as product_name,
    s.quantity * s.unit_price as total
FROM sales s
WHERE s.sale_date BETWEEN '2024-01-01' AND '2024-01-31';

-- แก้ไข: ใช้ JOIN
SELECT 
    s.sales_id,
    c.customer_name,
    p.product_name,
    s.quantity * s.unit_price as total
FROM sales s
INNER JOIN customers c ON s.customer_id = c.customer_id
INNER JOIN products p ON s.product_id = p.product_id
WHERE s.sale_date BETWEEN '2024-01-01' AND '2024-01-31';
```

### ประเภทของ JOIN และการใช้งาน:
```sql
-- INNER JOIN: เฉพาะข้อมูลที่มีใน both tables
SELECT c.customer_name, o.total_amount
FROM customers c
INNER JOIN orders o ON c.customer_id = o.customer_id;

-- LEFT JOIN: ข้อมูลทั้งหมดจาก left table
SELECT c.customer_name, COUNT(o.order_id) as order_count
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id, c.customer_name;

-- EXISTS vs IN: สำหรับ subqueries
-- ใช้ EXISTS เมื่อไม่ต้องการข้อมูลจาก subquery
SELECT c.customer_name
FROM customers c
WHERE EXISTS (
    SELECT 1 FROM orders o 
    WHERE o.customer_id = c.customer_id 
    AND o.order_date >= '2024-01-01'
);
```

## 6. การใช้ LIMIT และ OFFSET อย่างเหมาะสม

### ปัญหา: การใช้ OFFSET ขนาดใหญ่
```sql
-- ช้ามาก: OFFSET ที่ใหญ่
SELECT * FROM products 
ORDER BY created_at 
LIMIT 10 OFFSET 50000;
```

### วิธีแก้ไข - Cursor-based Pagination:
```sql
-- เร็วกว่า: ใช้ Cursor-based pagination
SELECT * FROM products 
WHERE created_at < '2024-01-15 10:30:00'
ORDER BY created_at DESC 
LIMIT 10;
```

### ตัวอย่างจริง - API Pagination:
```sql
-- ปัญหา: การแบ่งหน้าของรายการโพสต์
SELECT post_id, title, created_at 
FROM posts 
ORDER BY created_at DESC 
LIMIT 10 OFFSET :page_offset;

-- แก้ไข: ใช้ ID-based pagination
SELECT post_id, title, created_at 
FROM posts 
WHERE post_id < :last_post_id 
ORDER BY post_id DESC 
LIMIT 10;
```

## 7. การใช้ Aggregate Functions อย่างชาญฉลาด

### ปัญหา: การนับแถวทั้งหมด
```sql
-- ช้า: COUNT(*) บนตารางใหญ่
SELECT COUNT(*) FROM users;
```

### วิธีแก้ไข:
```sql
-- เร็วกว่า: ใช้ Approximate count หรือ Cache
-- สำหรับ MySQL
SELECT table_rows 
FROM information_schema.tables 
WHERE table_name = 'users' 
  AND table_schema = DATABASE();
```

### ตัวอย่างจริง - Dashboard Analytics:
```sql
-- ปัญหา: การสร้าง Dashboard ที่ต้องนับข้อมูลหลายตาราง
SELECT 
    (SELECT COUNT(*) FROM users) as total_users,
    (SELECT COUNT(*) FROM orders) as total_orders,
    (SELECT COUNT(*) FROM products) as total_products;

-- แก้ไข: ใช้ Summary Table หรือ Materialized View
-- สร้างตาราง summary
CREATE TABLE dashboard_stats (
    stat_name VARCHAR(50),
    stat_value INT,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Update ข้อมูลด้วย Scheduled Job
INSERT INTO dashboard_stats (stat_name, stat_value)
VALUES 
    ('total_users', (SELECT COUNT(*) FROM users)),
    ('total_orders', (SELECT COUNT(*) FROM orders)),
    ('total_products', (SELECT COUNT(*) FROM products))
ON DUPLICATE KEY UPDATE 
    stat_value = VALUES(stat_value),
    updated_at = CURRENT_TIMESTAMP;
```

## 8. การใช้ Query Caching

### ตัวอย่างการใช้ Redis Cache:
```go
package main

import (
    "context"
    "database/sql"
    "encoding/json"
    "fmt"
    "log"
    "time"
    
    "github.com/go-redis/redis/v8"
    _ "github.com/go-sql-driver/mysql"
)

// Constants
const (
    DefaultCacheExpiration = 5 * time.Minute
    DefaultRedisDB         = 0
    CacheKeyPattern       = "popular_products:%d"
)

// Error constants
const (
    ErrDatabaseConnection = "failed to connect to database"
    ErrRedisConnection    = "failed to connect to redis"
    ErrDatabaseQuery      = "database query failed"
    ErrScanProducts       = "failed to scan product"
    ErrCacheUnmarshal     = "failed to unmarshal cached data"
    ErrCacheMarshal       = "failed to marshal data for cache"
)

// Messages
const (
    MsgDataFromCache    = "Data retrieved from cache"
    MsgQueryingDatabase = "Querying database..."
    MsgDataCached       = "Data cached for %v"
    MsgCacheTestSeparator = "\n--- Calling again to test cache ---"
)

// Interfaces for better testability
type DatabaseQuerier interface {
    Query(query string, args ...interface{}) (*sql.Rows, error)
    Close() error
}

type CacheManager interface {
    Get(ctx context.Context, key string) (string, error)
    SetEX(ctx context.Context, key string, value interface{}, expiration time.Duration) error
    Close() error
}

type ProductRepository interface {
    GetPopularProducts(limit int) ([]Product, error)
}

// Structs
type Product struct {
    ProductID   int     `json:"product_id"`
    ProductName string  `json:"product_name"`
    Price       float64 `json:"price"`
    OrderCount  int     `json:"order_count"`
}

type CacheConfig struct {
    RedisAddr    string
    RedisPassword string
    RedisDB      int
    Expiration   time.Duration
}

type CacheService struct {
    DB     DatabaseQuerier
    Cache  CacheManager
    Ctx    context.Context
    Config CacheConfig
}

func NewCacheConfig(redisAddr string) CacheConfig {
    return CacheConfig{
        RedisAddr:    redisAddr,
        RedisPassword: "",
        RedisDB:      DefaultRedisDB,
        Expiration:   DefaultCacheExpiration,
    }
}

func NewCacheService(dbConnection string, config CacheConfig) (*CacheService, error) {
    db, err := sql.Open("mysql", dbConnection)
    if err != nil {
        return nil, fmt.Errorf("%s: %w", ErrDatabaseConnection, err)
    }
    
    rdb := redis.NewClient(&redis.Options{
        Addr:     config.RedisAddr,
        Password: config.RedisPassword,
        DB:       config.RedisDB,
    })
    
    return &CacheService{
        DB:     db,
        Cache:  rdb,
        Ctx:    context.Background(),
        Config: config,
    }, nil
}

func (cs *CacheService) GetPopularProducts(limit int) ([]Product, error) {
    cacheKey := fmt.Sprintf(CacheKeyPattern, limit)
    
    products, err := cs.getProductsFromCache(cacheKey)
    if err == nil {
        fmt.Println(MsgDataFromCache)
        return products, nil
    }
    
    products, err = cs.getProductsFromDatabase(limit)
    if err != nil {
        return nil, err
    }
    
    cs.cacheProducts(cacheKey, products)
    
    return products, nil
}

func (cs *CacheService) getProductsFromCache(cacheKey string) ([]Product, error) {
    cachedResult, err := cs.Cache.Get(cs.Ctx, cacheKey)
    if err != nil {
        return nil, err
    }
    
    var products []Product
    if err := json.Unmarshal([]byte(cachedResult), &products); err != nil {
        return nil, fmt.Errorf("%s: %w", ErrCacheUnmarshal, err)
    }
    
    return products, nil
}

func (cs *CacheService) getProductsFromDatabase(limit int) ([]Product, error) {
    fmt.Println(MsgQueryingDatabase)
    
    query := `
        SELECT p.product_id, p.product_name, p.price, 
               COUNT(oi.product_id) as order_count
        FROM products p
        LEFT JOIN order_items oi ON p.product_id = oi.product_id
        GROUP BY p.product_id, p.product_name, p.price
        ORDER BY order_count DESC
        LIMIT ?
    `
    
    rows, err := cs.DB.Query(query, limit)
    if err != nil {
        return nil, fmt.Errorf("%s: %w", ErrDatabaseQuery, err)
    }
    defer rows.Close()
    
    return cs.scanProducts(rows)
}

func (cs *CacheService) scanProducts(rows *sql.Rows) ([]Product, error) {
    var products []Product
    for rows.Next() {
        var p Product
        if err := rows.Scan(&p.ProductID, &p.ProductName, &p.Price, &p.OrderCount); err != nil {
            return nil, fmt.Errorf("%s: %w", ErrScanProducts, err)
        }
        products = append(products, p)
    }
    return products, nil
}

func (cs *CacheService) cacheProducts(cacheKey string, products []Product) {
    jsonData, err := json.Marshal(products)
    if err != nil {
        log.Printf("%s: %v", ErrCacheMarshal, err)
        return
    }
    
    if err := cs.Cache.SetEX(cs.Ctx, cacheKey, string(jsonData), cs.Config.Expiration); err != nil {
        log.Printf("Failed to cache data: %v", err)
        return
    }
    
    fmt.Printf(MsgDataCached+"\n", cs.Config.Expiration)
}

func (cs *CacheService) Close() error {
    if err := cs.DB.Close(); err != nil {
        return err
    }
    return cs.Cache.Close()
}

// ตัวอย่างการใช้งาน
func main() {
    config := NewCacheConfig("localhost:6379")
    
    cacheService, err := NewCacheService(
        "user:password@tcp(localhost:3306)/database",
        config,
    )
    if err != nil {
        log.Fatal("Error creating cache service:", err)
    }
    defer cacheService.Close()
    
    // ดึงสินค้ายอดนิยม 10 อันดับแรก
    products, err := cacheService.GetPopularProducts(10)
    if err != nil {
        log.Fatal("Error:", err)
    }
    
    // แสดงผลลัพธ์
    jsonResult, _ := json.MarshalIndent(products, "", "  ")
    fmt.Printf("Popular Products:\n%s\n", jsonResult)
    
    // ลองเรียกอีกครั้งเพื่อทดสอบ cache
    fmt.Println(MsgCacheTestSeparator)
    products2, _ := cacheService.GetPopularProducts(10)
    jsonResult2, _ := json.MarshalIndent(products2, "", "  ")
    fmt.Printf("Popular Products (from cache):\n%s\n", jsonResult2)
}
```

## 9. Common Table Expressions (WITH Clause)

### การใช้ WITH เมื่อใด:

#### 1. แทนที่ Subqueries ที่ซับซ้อน:
```sql
-- ปัญหา: Nested Subqueries ที่อ่านยาก
SELECT 
    customer_id,
    total_orders,
    avg_order_value
FROM (
    SELECT 
        customer_id,
        COUNT(*) as total_orders,
        AVG(order_amount) as avg_order_value
    FROM (
        SELECT 
            o.customer_id,
            SUM(oi.quantity * oi.unit_price) as order_amount
        FROM orders o
        JOIN order_items oi ON o.order_id = oi.order_id
        WHERE o.order_date >= '2024-01-01'
        GROUP BY o.order_id, o.customer_id
    ) order_totals
    GROUP BY customer_id
    HAVING COUNT(*) >= 5
) customer_stats;

-- แก้ไข: ใช้ WITH ให้อ่านง่ายขึ้น
WITH order_totals AS (
    SELECT 
        o.customer_id,
        o.order_id,
        SUM(oi.quantity * oi.unit_price) as order_amount
    FROM orders o
    JOIN order_items oi ON o.order_id = oi.order_id
    WHERE o.order_date >= '2024-01-01'
    GROUP BY o.order_id, o.customer_id
),
customer_stats AS (
    SELECT 
        customer_id,
        COUNT(*) as total_orders,
        AVG(order_amount) as avg_order_value
    FROM order_totals
    GROUP BY customer_id
    HAVING COUNT(*) >= 5
)
SELECT customer_id, total_orders, avg_order_value
FROM customer_stats;
```

#### 2. การใช้ซ้ำในหลายที่:
```sql
-- ตัวอย่าง: รายงานเปรียบเทียบยอดขายรายเดือน
WITH monthly_sales AS (
    SELECT 
        DATE_FORMAT(order_date, '%Y-%m') as month,
        SUM(total_amount) as monthly_total,
        COUNT(*) as order_count
    FROM orders
    WHERE order_date >= DATE_SUB(CURDATE(), INTERVAL 12 MONTH)
    GROUP BY DATE_FORMAT(order_date, '%Y-%m')
)
SELECT 
    current_month.month,
    current_month.monthly_total,
    current_month.order_count,
    LAG(monthly_total) OVER (ORDER BY month) as previous_month_total,
    ROUND(
        (current_month.monthly_total - LAG(monthly_total) OVER (ORDER BY month)) 
        / LAG(monthly_total) OVER (ORDER BY month) * 100, 2
    ) as growth_percentage
FROM monthly_sales current_month
ORDER BY month;
```

#### 3. Recursive CTE สำหรับ Hierarchical Data:
```sql
-- ตัวอย่าง: Organization Chart
WITH RECURSIVE employee_hierarchy AS (
    -- Base case: Top-level managers
    SELECT 
        employee_id,
        employee_name,
        manager_id,
        0 as level,
        CAST(employee_name AS CHAR(1000)) as path
    FROM employees
    WHERE manager_id IS NULL
    
    UNION ALL
    
    -- Recursive case: Subordinates
    SELECT 
        e.employee_id,
        e.employee_name,
        e.manager_id,
        eh.level + 1,
        CONCAT(eh.path, ' -> ', e.employee_name)
    FROM employees e
    JOIN employee_hierarchy eh ON e.manager_id = eh.employee_id
    WHERE eh.level < 10  -- Prevent infinite recursion
)
SELECT employee_id, employee_name, level, path
FROM employee_hierarchy
ORDER BY level, employee_name;
```

## 10. Window Functions สำหรับการวิเคราะห์ข้อมูล

### การใช้ Window Functions แทน Self-Join:
```sql
-- ปัญหา: Self-join ที่ช้า
SELECT 
    o1.order_id,
    o1.order_date,
    o1.total_amount,
    (SELECT COUNT(*) FROM orders o2 WHERE o2.order_date <= o1.order_date) as running_count
FROM orders o1
ORDER BY o1.order_date;

-- แก้ไข: ใช้ Window Function
SELECT 
    order_id,
    order_date,
    total_amount,
    ROW_NUMBER() OVER (ORDER BY order_date) as running_count,
    SUM(total_amount) OVER (ORDER BY order_date ROWS UNBOUNDED PRECEDING) as running_total
FROM orders
ORDER BY order_date;
```

### ตัวอย่างจริง - Sales Analytics:
```sql
-- วิเคราะห์ยอดขายพนักงาน
SELECT 
    employee_id,
    employee_name,
    monthly_sales,
    -- Ranking
    ROW_NUMBER() OVER (ORDER BY monthly_sales DESC) as sales_rank,
    DENSE_RANK() OVER (ORDER BY monthly_sales DESC) as dense_rank,
    
    -- Percentile
    PERCENT_RANK() OVER (ORDER BY monthly_sales) as percentile_rank,
    
    -- Moving averages
    AVG(monthly_sales) OVER (
        ORDER BY employee_id 
        ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
    ) as moving_avg_3months,
    
    -- Comparison with previous period
    LAG(monthly_sales, 1) OVER (PARTITION BY employee_id ORDER BY month) as prev_month_sales,
    monthly_sales - LAG(monthly_sales, 1) OVER (PARTITION BY employee_id ORDER BY month) as month_over_month_change
FROM (
    SELECT 
        e.employee_id,
        e.employee_name,
        DATE_FORMAT(s.sale_date, '%Y-%m') as month,
        SUM(s.amount) as monthly_sales
    FROM employees e
    JOIN sales s ON e.employee_id = s.employee_id
    GROUP BY e.employee_id, e.employee_name, DATE_FORMAT(s.sale_date, '%Y-%m')
) monthly_data;
```

## 11. การจัดการ Temporary Tables และ Memory

### เปรียบเทียบ Temporary Tables vs CTEs vs Table Variables:

#### Temporary Tables (สำหรับข้อมูลขนาดใหญ่):
```sql
-- ใช้เมื่อต้องการ Index และประมวลผลข้อมูลมาก
CREATE TEMPORARY TABLE temp_sales_summary (
    customer_id INT,
    total_sales DECIMAL(15,2),
    order_count INT,
    INDEX idx_customer (customer_id),
    INDEX idx_sales (total_sales)
);

INSERT INTO temp_sales_summary
SELECT 
    customer_id,
    SUM(total_amount) as total_sales,
    COUNT(*) as order_count
FROM orders
WHERE order_date >= DATE_SUB(CURDATE(), INTERVAL 1 YEAR)
GROUP BY customer_id;

-- ใช้ในการประมวลผลต่อ
SELECT 
    t.customer_id,
    c.customer_name,
    t.total_sales,
    t.order_count,
    CASE 
        WHEN t.total_sales > 100000 THEN 'VIP'
        WHEN t.total_sales > 50000 THEN 'Premium'
        ELSE 'Standard'
    END as customer_tier
FROM temp_sales_summary t
JOIN customers c ON t.customer_id = c.customer_id
WHERE t.order_count >= 10;

DROP TEMPORARY TABLE temp_sales_summary;
```

#### Memory Engine Tables (สำหรับข้อมูลที่ต้องการความเร็วสูง):
```sql
-- สำหรับ Cache หรือ Session data
CREATE TABLE session_cart (
    session_id VARCHAR(64),
    product_id INT,
    quantity INT,
    added_at TIMESTAMP,
    INDEX idx_session (session_id)
) ENGINE=MEMORY;
```

## 12. Bulk Operations และ Batch Processing

### การใช้ BULK INSERT แทน Multiple INSERT:
```sql
-- ไม่ดี: Multiple INSERT statements
INSERT INTO products (name, price, category_id) VALUES ('Product 1', 99.99, 1);
INSERT INTO products (name, price, category_id) VALUES ('Product 2', 149.99, 1);
INSERT INTO products (name, price, category_id) VALUES ('Product 3', 199.99, 2);
-- ... repeat 1000 times

-- ดี: Single BULK INSERT
INSERT INTO products (name, price, category_id) VALUES 
('Product 1', 99.99, 1),
('Product 2', 149.99, 1),
('Product 3', 199.99, 2),
-- ... all 1000 records in one statement
('Product 1000', 299.99, 3);
```

### การใช้ MERGE สำหรับ UPSERT Operations:
```sql
-- ตัวอย่าง: อัปเดต inventory จาก external system
MERGE inventory AS target
USING (
    SELECT product_id, new_quantity, last_updated
    FROM temp_inventory_updates
) AS source ON target.product_id = source.product_id
WHEN MATCHED AND source.last_updated > target.last_updated THEN
    UPDATE SET 
        quantity = source.new_quantity,
        last_updated = source.last_updated
WHEN NOT MATCHED THEN
    INSERT (product_id, quantity, last_updated)
    VALUES (source.product_id, source.new_quantity, source.last_updated);
```

### Batch Processing สำหรับข้อมูลขนาดใหญ่:
```go
package main

import (
    "database/sql"
    "fmt"
    "log"
    "strings"
    "time"
    
    _ "github.com/go-sql-driver/mysql"
)

// Constants
const (
    DefaultBatchSize          = 1000
    BatchProcessingDelay      = 100 * time.Millisecond
    RowProcessingDelay        = 1 * time.Millisecond
    
    // Stats keys
    StatsKeyProcessed = "processed"
    StatsKeyPending   = "pending"
)

// Error constants
const (
    ErrDatabaseConnection    = "failed to connect to database"
    ErrQueryBatch           = "failed to query batch"
    ErrScanRow              = "failed to scan row"
    ErrBeginTransaction     = "failed to begin transaction"
    ErrUpdateStatus         = "failed to update processed status"
    ErrCommitTransaction    = "failed to commit transaction"
    ErrValidation          = "data_column is empty for ID %d"
)

// Messages
const (
    MsgProcessingComplete = "Batch processing completed. Total rows processed: %d"
    MsgProgressUpdate     = "Processed batch: %d rows, Total processed: %d"
    MsgProcessingError    = "Error processing row %d: %v"
    MsgBeforeProcessing   = "Before processing - Processed: %d, Pending: %d"
    MsgAfterProcessing    = "After processing - Processed: %d, Pending: %d"
    MsgTotalTime         = "Total processing time: %v"
    MsgBatchProcessingFailed = "Batch processing failed:"
)

// Interfaces for better testability
type DatabaseExecutor interface {
    Query(query string, args ...interface{}) (*sql.Rows, error)
    QueryRow(query string, args ...interface{}) *sql.Row
    Begin() (*sql.Tx, error)
    Close() error
}

type TransactionManager interface {
    Exec(query string, args ...interface{}) (sql.Result, error)
    Commit() error
    Rollback() error
}

type RowProcessor interface {
    ProcessRow(row DataRow) error
}

// Structs
type DataRow struct {
    ID         int    `db:"id"`
    DataColumn string `db:"data_column"`
}

type ProcessingStats map[string]int

type BatchConfig struct {
    BatchSize int
    Delay     time.Duration
}

type BatchProcessor struct {
    DB        DatabaseExecutor
    Config    BatchConfig
    Processor RowProcessor
}

type DefaultRowProcessor struct{}

func (drp *DefaultRowProcessor) ProcessRow(row DataRow) error {
    // Simulate processing time
    time.Sleep(RowProcessingDelay)
    
    // ตัวอย่างการ validate ข้อมูล
    if row.DataColumn == "" {
        return fmt.Errorf(ErrValidation, row.ID)
    }
    
    // ตัวอย่างการประมวลผลข้อมูล
    // processedData := strings.ToUpper(row.DataColumn)
    // ทำการประมวลผลเพิ่มเติมตามความต้องการ
    
    return nil
}

func NewBatchConfig(batchSize int) BatchConfig {
    return BatchConfig{
        BatchSize: batchSize,
        Delay:     BatchProcessingDelay,
    }
}

func NewBatchProcessor(dbConnection string, config BatchConfig) (*BatchProcessor, error) {
    db, err := sql.Open("mysql", dbConnection)
    if err != nil {
        return nil, fmt.Errorf("%s: %w", ErrDatabaseConnection, err)
    }
    
    return &BatchProcessor{
        DB:        db,
        Config:    config,
        Processor: &DefaultRowProcessor{},
    }, nil
}

func (bp *BatchProcessor) ProcessLargeDatasetInBatches() error {
    offset := 0
    totalProcessed := 0
    
    for {
        batch, err := bp.fetchBatch(offset)
        if err != nil {
            return err
        }
        
        if len(batch) == 0 {
            break
        }
        
        processedCount, err := bp.processBatch(batch)
        if err != nil {
            return err
        }
        
        totalProcessed += processedCount
        offset += bp.Config.BatchSize
        
        bp.logProgress(processedCount, totalProcessed)
        time.Sleep(bp.Config.Delay)
    }
    
    fmt.Printf(MsgProcessingComplete+"\n", totalProcessed)
    return nil
}

func (bp *BatchProcessor) fetchBatch(offset int) ([]DataRow, error) {
    query := `
        SELECT id, data_column 
        FROM large_table 
        WHERE processed = 0
        ORDER BY id 
        LIMIT ? OFFSET ?
    `
    
    rows, err := bp.DB.Query(query, bp.Config.BatchSize, offset)
    if err != nil {
        return nil, fmt.Errorf("%s: %w", ErrQueryBatch, err)
    }
    defer rows.Close()
    
    return bp.scanRows(rows)
}

func (bp *BatchProcessor) scanRows(rows *sql.Rows) ([]DataRow, error) {
    var batch []DataRow
    for rows.Next() {
        var row DataRow
        if err := rows.Scan(&row.ID, &row.DataColumn); err != nil {
            return nil, fmt.Errorf("%s: %w", ErrScanRow, err)
        }
        batch = append(batch, row)
    }
    return batch, nil
}

func (bp *BatchProcessor) processBatch(batch []DataRow) (int, error) {
    tx, err := bp.DB.Begin()
    if err != nil {
        return 0, fmt.Errorf("%s: %w", ErrBeginTransaction, err)
    }
    defer tx.Rollback()
    
    processedIDs := bp.processRows(batch)
    
    if len(processedIDs) > 0 {
        if err := bp.updateProcessedStatus(tx, processedIDs); err != nil {
            return 0, err
        }
    }
    
    if err := tx.Commit(); err != nil {
        return 0, fmt.Errorf("%s: %w", ErrCommitTransaction, err)
    }
    
    return len(processedIDs), nil
}

func (bp *BatchProcessor) processRows(batch []DataRow) []int {
    var processedIDs []int
    for _, row := range batch {
        if err := bp.Processor.ProcessRow(row); err != nil {
            log.Printf(MsgProcessingError, row.ID, err)
            continue
        }
        processedIDs = append(processedIDs, row.ID)
    }
    return processedIDs
}

func (bp *BatchProcessor) updateProcessedStatus(tx TransactionManager, ids []int) error {
    placeholders := make([]string, len(ids))
    args := make([]interface{}, len(ids))
    for i, id := range ids {
        placeholders[i] = "?"
        args[i] = id
    }
    
    updateQuery := fmt.Sprintf(`
        UPDATE large_table 
        SET processed = 1, 
            processed_at = NOW()
        WHERE id IN (%s)
    `, strings.Join(placeholders, ","))
    
    _, err := tx.Exec(updateQuery, args...)
    if err != nil {
        return fmt.Errorf("%s: %w", ErrUpdateStatus, err)
    }
    
    return nil
}

func (bp *BatchProcessor) logProgress(batchCount, totalCount int) {
    fmt.Printf(MsgProgressUpdate+"\n", batchCount, totalCount)
}

func (bp *BatchProcessor) GetProcessingStats() (ProcessingStats, error) {
    stats := make(ProcessingStats)
    
    processed, err := bp.getProcessedCount()
    if err != nil {
        return nil, err
    }
    stats[StatsKeyProcessed] = processed
    
    pending, err := bp.getPendingCount()
    if err != nil {
        return nil, err
    }
    stats[StatsKeyPending] = pending
    
    return stats, nil
}

func (bp *BatchProcessor) getProcessedCount() (int, error) {
    var processed int
    err := bp.DB.QueryRow("SELECT COUNT(*) FROM large_table WHERE processed = 1").Scan(&processed)
    return processed, err
}

func (bp *BatchProcessor) getPendingCount() (int, error) {
    var pending int
    err := bp.DB.QueryRow("SELECT COUNT(*) FROM large_table WHERE processed = 0").Scan(&pending)
    return pending, err
}

func (bp *BatchProcessor) Close() error {
    return bp.DB.Close()
}

// ตัวอย่างการใช้งาน
func main() {
    config := NewBatchConfig(DefaultBatchSize)
    
    processor, err := NewBatchProcessor(
        "user:password@tcp(localhost:3306)/database",
        config,
    )
    if err != nil {
        log.Fatal("Error creating batch processor:", err)
    }
    defer processor.Close()
    
    // แสดงสถานะก่อนเริ่มประมวลผล
    stats, _ := processor.GetProcessingStats()
    fmt.Printf(MsgBeforeProcessing+"\n", 
        stats[StatsKeyProcessed], stats[StatsKeyPending])
    
    // เริ่มประมวลผลแบบ batch
    startTime := time.Now()
    if err := processor.ProcessLargeDatasetInBatches(); err != nil {
        log.Fatal(MsgBatchProcessingFailed, err)
    }
    processingTime := time.Since(startTime)
    
    // แสดงสถานะหลังประมวลผล
    stats, _ = processor.GetProcessingStats()
    fmt.Printf(MsgAfterProcessing+"\n", 
        stats[StatsKeyProcessed], stats[StatsKeyPending])
    fmt.Printf(MsgTotalTime+"\n", processingTime)
}
```

## 13. การปรับปรุง Database Schema

### การใช้ Partitioning:
```sql
-- ตัวอย่าง: แบ่ง Partition ตามวันที่สำหรับ Log Table
CREATE TABLE user_logs (
    log_id INT AUTO_INCREMENT,
    user_id INT,
    action VARCHAR(100),
    log_date DATE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (log_id, log_date)
)
PARTITION BY RANGE (YEAR(log_date)) (
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025),
    PARTITION p2025 VALUES LESS THAN (2026),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);
```

### การใช้ Denormalization อย่างเหมาะสม:
```sql
-- ปัญหา: การ JOIN หลายตารางสำหรับรายงาน
SELECT 
    o.order_id,
    c.customer_name,
    c.customer_email,
    p.product_name,
    oi.quantity,
    oi.unit_price
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id;

-- แก้ไข: สร้าง Denormalized table สำหรับรายงาน
CREATE TABLE order_reports (
    order_id INT,
    customer_name VARCHAR(100),
    customer_email VARCHAR(100),
    product_name VARCHAR(200),
    quantity INT,
    unit_price DECIMAL(10,2),
    order_date DATE,
    INDEX idx_order_date (order_date),
    INDEX idx_customer_email (customer_email)
);
```

### การเลือก Storage Engine ที่เหมาะสม:
```sql
-- InnoDB: สำหรับ OLTP (Online Transaction Processing)
CREATE TABLE orders (
    order_id INT AUTO_INCREMENT PRIMARY KEY,
    customer_id INT,
    order_date DATE,
    total_amount DECIMAL(10,2),
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
) ENGINE=InnoDB;

-- MyISAM: สำหรับ Read-heavy workloads (deprecated in newer versions)
-- ไม่แนะนำให้ใช้แล้ว

-- Memory: สำหรับ Temporary data
CREATE TABLE session_data (
    session_id VARCHAR(64),
    data TEXT,
    expires_at TIMESTAMP
) ENGINE=MEMORY;

-- Archive: สำหรับ Historical data
CREATE TABLE audit_logs (
    log_id INT AUTO_INCREMENT PRIMARY KEY,
    table_name VARCHAR(64),
    action VARCHAR(20),
    old_values JSON,
    new_values JSON,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
) ENGINE=ARCHIVE;
```

## 14. การใช้ Query Hints อย่างเหมาะสม

### MySQL Query Hints:
```sql
-- ใช้ Specific Index
SELECT /*+ USE_INDEX(orders, idx_customer_date) */ 
    order_id, total_amount 
FROM orders 
WHERE customer_id = 123 AND order_date >= '2024-01-01';

-- Force Join Order
SELECT /*+ STRAIGHT_JOIN */
    c.customer_name, o.order_date, o.total_amount
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
WHERE c.status = 'active';

-- Use Temporary Table สำหรับ Subquery
SELECT /*+ USE_TMP_TABLE */ 
    customer_id, AVG(total_amount)
FROM orders
WHERE order_date >= '2024-01-01'
GROUP BY customer_id;
```

### SQL Server Query Hints:
```sql
-- Use Specific Join Algorithm
SELECT c.customer_name, o.total_amount
FROM customers c
INNER MERGE JOIN orders o ON c.customer_id = o.customer_id
WHERE c.status = 'active';

-- Parallel Processing
SELECT customer_id, SUM(total_amount)
FROM orders
WHERE order_date >= '2024-01-01'
GROUP BY customer_id
OPTION (MAXDOP 4);

-- Use Plan Guide
SELECT /*+ RECOMPILE */ 
    product_id, SUM(quantity)
FROM order_items
WHERE order_date BETWEEN @start_date AND @end_date
GROUP BY product_id;
```

### เมื่อไหร่ควรใช้ Hints:
```sql
-- ✅ ใช้เมื่อ: Query Optimizer เลือก plan ที่ไม่เหมาะสม
-- ❌ ไม่ใช้เมื่อ: เป็นการแก้ไขชั่วคราว แทนที่จะแก้ปัญหาที่ต้นเหตุ

-- ตัวอย่างการใช้ที่เหมาะสม: Force index เมื่อ optimizer เลือกผิด
SELECT /*+ USE_INDEX(orders, idx_status_date) */
    order_id, customer_id, total_amount
FROM orders
WHERE status IN ('pending', 'processing')
  AND order_date >= CURDATE() - INTERVAL 7 DAY;
```

## 15. Best Practices สรุป

### ✅ ควรทำ:
- **วัดประสิทธิภาพก่อนปรับปรุง** - ใช้ EXPLAIN และ monitoring tools
- **สร้าง Index ที่เหมาะสม** - สำหรับ columns ที่ใช้ใน WHERE, JOIN, ORDER BY
- **ใช้ LIMIT เมื่อไม่ต้องการข้อมูลทั้งหมด** - ลดการโหลดข้อมูลที่ไม่จำเป็น
- **ระบุ columns ที่ต้องการใน SELECT** - แทน SELECT *
- **ใช้ Prepared Statements** - ป้องกัน SQL Injection และปรับปรุงประสิทธิภาพ
- **ใช้ Connection Pooling** - ลดเวลาการเชื่อมต่อ Database
- **Cache ผลลัพธ์ที่ถูกใช้บ่อยๆ** - ใช้ Redis หรือ Memcached
- **ใช้ JOIN แทน Subqueries** - เมื่อเป็นไปได้
- **ปรับปรุงอย่างต่อเนื่อง** - ติดตามและวิเคราะห์ performance เป็นประจำ

### ❌ ไม่ควรทำ:
- **ใช้ SELECT * เมื่อไม่จำเป็น** - สิ้นเปลือง bandwidth และ memory
- **ใช้ LIKE ที่ขึ้นต้นด้วย %** - ทำให้ index ไม่ทำงาน
- **ใช้ Functions ใน WHERE clause กับ indexed columns** - ทำลาย index optimization
- **สร้าง Index มากเกินไปโดยไม่จำเป็น** - ทำให้ INSERT/UPDATE/DELETE ช้า
- **ใช้ OFFSET ขนาดใหญ่** - ใช้ cursor-based pagination แทน
- **ปรับปรุงก่อนวัด** - "Premature optimization is the root of all evil"
- **ใช้ Hints โดยไม่ระวัง** - อาจทำให้ performance แย่ลงเมื่อข้อมูลเปลี่ยนไป

### 🔄 กระบวนการปรับปรุงที่แนะนำ:
1. **📊 Measure** - วัดประสิทธิภาพปัจจุบัน
2. **🔍 Analyze** - หาจุดที่เป็นปัญหา (bottlenecks)
3. **⚙️ Optimize** - ใช้เทคนิคที่เหมาะสม
4. **🧪 Test** - ทดสอบผลลัพธ์และเปรียบเทียบ
5. **📝 Document** - บันทึกการเปลี่ยนแปลงและผลลัพธ์
6. **🔁 Repeat** - ทำซ้ำกระบวนการนี้เป็นประจำ

### 🏅 Performance Metrics ที่ควรติดตาม:
- **Query Response Time** - เวลาตอบสนองของ Query
- **Throughput** - จำนวน queries ต่อวินาที
- **Connection Pool Usage** - การใช้งาน connection pool
- **Index Hit Ratio** - อัตราการใช้ index
- **Memory Usage** - การใช้ RAM และ Buffer Pool
- **Disk I/O** - การอ่าน/เขียนข้อมูลจาก storage

### 🚀 เทคนิคขั้นสูงสำหรับ Experts:
- **Read Replicas** - แยก Read และ Write operations
- **Database Sharding** - แบ่งข้อมูลข้าม multiple databases
- **Query Result Caching** - Cache ผลลัพธ์ที่ซับซ้อนและใช้เวลานาน
- **Database Monitoring** - ใช้เครื่องมือเช่น Datadog, New Relic หรือ built-in tools
- **Auto-scaling** - ปรับขนาด database resources ตามความต้องการ

## 16. สรุป

การปรับปรุงประสิทธิภาพของ Database Query เป็นกระบวนการที่ต้องอาศัยการวิเคราะห์และทดสอบอย่างระมัดระวัง ขั้นตอนสำคัญและเทคนิคหลักคือ:

### 🎯 กลยุทธ์หลัก:
1. **วิเคราะห์ปัญหา** - ใช้ EXPLAIN และ Monitoring tools
2. **สร้าง Index ที่เหมาะสม** - ไม่มากเกินไป ไม่น้อยเกินไป
3. **เขียน Query ที่มีประสิทธิภาพ** - เลือกข้อมูลเฉพาะที่จำเป็น
4. **ใช้ Caching** - ลดการ Query ที่ซ้ำๆ
5. **ติดตามและปรับปรุง** - วัดผลและปรับปรุงอย่างต่อเนื่อง

### 🛠️ เทคนิคขั้นสูง:
- **Common Table Expressions (WITH)** - ทำให้ Query ซับซ้อนอ่านง่ายและมีประสิทธิภาพ
- **Window Functions** - แทนที่ Self-joins และ Subqueries ที่ช้า
- **Temporary Tables** - จัดการข้อมูลขนาดใหญ่อย่างมีประสิทธิภาพ
- **Bulk Operations** - ประมวลผลข้อมูลจำนวนมากในครั้งเดียว
- **Query Hints** - ควบคุม Query Optimizer เมื่อจำเป็น

### 📊 การวัดประสิทธิภาพ:
- **Execution Time** - เวลาในการทำงาน
- **Memory Usage** - การใช้หน่วยความจำ
- **I/O Operations** - การอ่าน/เขียนข้อมูล
- **CPU Usage** - การใช้งาน Processor
- **Concurrency** - การทำงานพร้อมกันของ Users

### 🔄 กระบวนการปรับปรุงอย่างต่อเนื่อง:
1. **Monitor** - ติดตามประสิทธิภาพอย่างสม่ำเสมอ
2. **Analyze** - วิเคราะห์ Bottlenecks และจุดที่ต้องปรับปรุง
3. **Optimize** - ใช้เทคนิคที่เหมาะสมกับปัญหา
4. **Test** - ทดสอบผลลัพธ์และวัดการปรับปรุง
5. **Document** - บันทึกการเปลี่ยนแปลงและผลลัพธ์

การปรับปรุงประสิทธิภาพที่ดีต้องสมดุลระหว่าง **ความเร็ว** **การใช้ Memory** **ความซับซ้อนในการบำรุงรักษา** และ **ความสามารถในการ Scale** 

> 💡 **หลักสำคัญ**: "Premature optimization is the root of all evil" - ควรปรับปรุงเมื่อมีปัญหาจริง และวัดผลได้อย่างชัดเจน
