# Performance Optimization - การเพิ่มประสิทธิภาพระบบ

## 📋 สารบัญ
- [Performance Optimization คืออะไร?](#performance-optimization-คืออะไร)
- [Backend Performance](#backend-performance)
- [Database Performance](#database-performance)
- [Network Performance](#network-performance)
- [Caching Strategies](#caching-strategies)
- [Load Testing](#load-testing)

## Performance Optimization คืออะไร?

**Performance Optimization** คือกระบวนการปรับปรุงระบบให้ทำงานได้เร็วขึ้น ใช้ทรัพยากรน้อยลง และตอบสนองผู้ใช้ได้ดีขึ้น

### ทำไมต้อง Optimize?
- ⚡ **ประสบการณ์ผู้ใช้**: เว็บเร็ว = ผู้ใช้พอใจ
- 💰 **ต้นทุน**: ใช้ server น้อยลง
- 🔍 **SEO**: Google ชอบเว็บเร็ว
- 📈 **Conversion Rate**: เว็บช้า = ขายน้อย

### Performance Metrics ที่สำคัญ

| Metric | คำอธิบาย | เป้าหมาย |
|--------|-----------|----------|
| **Response Time** | เวลาตอบสนอง | < 200ms |
| **Throughput** | จำนวน request ต่อวินาที | > 1000 RPS |
| **CPU Usage** | การใช้งาน CPU | < 70% |
| **Memory Usage** | การใช้งาน RAM | < 80% |
| **Page Load Time** | เวลาโหลดหน้า | < 3 วินาที |




## Backend Performance

### 1. **API Optimization**

#### Response Optimization
```go
// ❌ ส่งข้อมูลเยอะเกินไป
func GetUsers(w http.ResponseWriter, r *http.Request) {
    users, err := userRepo.FindAll() // ข้อมูลทั้งหมด
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    json.NewEncoder(w).Encode(users)
}

// ✅ ส่งเฉพาะที่จำเป็น + Pagination
type UserResponse struct {
    Data       []User     `json:"data"`
    Pagination Pagination `json:"pagination"`
}

type Pagination struct {
    Page  int `json:"page"`
    Limit int `json:"limit"`
    Total int `json:"total"`
}

func GetUsersOptimized(w http.ResponseWriter, r *http.Request) {
    page, _ := strconv.Atoi(r.URL.Query().Get("page"))
    if page <= 0 {
        page = 1
    }
    
    limit, _ := strconv.Atoi(r.URL.Query().Get("limit"))
    if limit <= 0 {
        limit = 10
    }
    
    fields := r.URL.Query().Get("fields")
    if fields == "" {
        fields = "id,name,email"
    }
    
    offset := (page - 1) * limit
    
    users, err := userRepo.FindWithPagination(fields, limit, offset)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    
    total, _ := userRepo.Count()
    
    response := UserResponse{
        Data: users,
        Pagination: Pagination{
            Page:  page,
            Limit: limit,
            Total: total,
        },
    }
    
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(response)
}
```

#### Goroutine Optimization
```go
// ❌ Sequential execution
func GetUserData(userID int) (*UserData, error) {
    user, err := userRepo.FindByID(userID)        // 100ms
    if err != nil {
        return nil, err
    }
    
    orders, err := orderRepo.FindByUserID(userID) // 150ms
    if err != nil {
        return nil, err
    }
    
    profile, err := profileRepo.FindByUserID(userID) // 80ms
    if err != nil {
        return nil, err
    }
    // รวม: 330ms
    
    return &UserData{
        User:    user,
        Orders:  orders,
        Profile: profile,
    }, nil
}

// ✅ Parallel execution with goroutines
type UserData struct {
    User    *User    `json:"user"`
    Orders  []Order  `json:"orders"`
    Profile *Profile `json:"profile"`
}

func GetUserDataOptimized(userID int) (*UserData, error) {
    type result struct {
        user    *User
        orders  []Order
        profile *Profile
        err     error
    }
    
    resultChan := make(chan result, 3)
    
    // Goroutine สำหรับ user
    go func() {
        user, err := userRepo.FindByID(userID) // 100ms
        resultChan <- result{user: user, err: err}
    }()
    
    // Goroutine สำหรับ orders
    go func() {
        orders, err := orderRepo.FindByUserID(userID) // 150ms
        resultChan <- result{orders: orders, err: err}
    }()
    
    // Goroutine สำหรับ profile
    go func() {
        profile, err := profileRepo.FindByUserID(userID) // 80ms
        resultChan <- result{profile: profile, err: err}
    }()
    
    var userData UserData
    var errors []error
    
    // รับผลลัพธ์จาก 3 goroutines
    for i := 0; i < 3; i++ {
        res := <-resultChan
        if res.err != nil {
            errors = append(errors, res.err)
            continue
        }
        
        if res.user != nil {
            userData.User = res.user
        }
        if res.orders != nil {
            userData.Orders = res.orders
        }
        if res.profile != nil {
            userData.Profile = res.profile
        }
    }
    
    if len(errors) > 0 {
        return nil, errors[0] // return first error
    }
    
    // รวม: 150ms (เท่ากับ query ที่ช้าที่สุด)
    return &userData, nil
}
```

### 2. **Memory Management**

#### Memory Management
```go
// ❌ Memory leak
var cache = make(map[string]interface{}) // cache โตเรื่อยๆ

func GetData(w http.ResponseWriter, r *http.Request) {
    key := r.URL.Path
    if data, exists := cache[key]; exists {
        json.NewEncoder(w).Encode(data)
        return
    }
    
    heavyData := getHeavyData()
    cache[key] = heavyData // ไม่มีการลบ
    json.NewEncoder(w).Encode(heavyData)
}

// ✅ Limited cache with TTL
import (
    "sync"
    "time"
)

type CacheItem struct {
    Data      interface{}
    ExpiresAt time.Time
}

type TTLCache struct {
    items    map[string]CacheItem
    mutex    sync.RWMutex
    maxItems int
    ttl      time.Duration
}

func NewTTLCache(maxItems int, ttl time.Duration) *TTLCache {
    cache := &TTLCache{
        items:    make(map[string]CacheItem),
        maxItems: maxItems,
        ttl:      ttl,
    }
    
    // Background cleanup goroutine
    go cache.cleanup()
    return cache
}

func (c *TTLCache) Get(key string) (interface{}, bool) {
    c.mutex.RLock()
    defer c.mutex.RUnlock()
    
    item, exists := c.items[key]
    if !exists || time.Now().After(item.ExpiresAt) {
        return nil, false
    }
    return item.Data, true
}

func (c *TTLCache) Set(key string, data interface{}) {
    c.mutex.Lock()
    defer c.mutex.Unlock()
    
    // ถ้าเกิน maxItems ให้ลบ item เก่าที่สุด
    if len(c.items) >= c.maxItems {
        c.evictOldest()
    }
    
    c.items[key] = CacheItem{
        Data:      data,
        ExpiresAt: time.Now().Add(c.ttl),
    }
}

func (c *TTLCache) cleanup() {
    ticker := time.NewTicker(time.Minute)
    defer ticker.Stop()
    
    for range ticker.C {
        c.mutex.Lock()
        now := time.Now()
        for key, item := range c.items {
            if now.After(item.ExpiresAt) {
                delete(c.items, key)
            }
        }
        c.mutex.Unlock()
    }
}

func (c *TTLCache) evictOldest() {
    // ลบ item แรกที่เจอ (ใน production ควรใช้ LRU)
    for key := range c.items {
        delete(c.items, key)
        break
    }
}

var globalCache = NewTTLCache(1000, 10*time.Minute)

func GetDataOptimized(w http.ResponseWriter, r *http.Request) {
    key := r.URL.Path
    
    if data, exists := globalCache.Get(key); exists {
        w.Header().Set("Content-Type", "application/json")
        json.NewEncoder(w).Encode(data)
        return
    }
    
    heavyData := getHeavyData()
    globalCache.Set(key, heavyData)
    
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(heavyData)
}
```

#### Stream Processing
```go
// ❌ โหลดไฟล์ทั้งหมดเข้า memory
func DownloadFile(w http.ResponseWriter, r *http.Request) {
    data, err := os.ReadFile("large-file.txt") // อันตราย!
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    w.Write(data)
}

// ✅ ใช้ stream
func DownloadFileOptimized(w http.ResponseWriter, r *http.Request) {
    file, err := os.Open("large-file.txt")
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    defer file.Close()
    
    w.Header().Set("Content-Type", "application/octet-stream")
    w.Header().Set("Content-Disposition", "attachment; filename=large-file.txt")
    
    // Stream file directly to response
    io.Copy(w, file)
}

// ✅ CSV Processing with streaming
import (
    "encoding/csv"
    "encoding/json"
    "io"
)

type ProcessedRow struct {
    ID    string `json:"id"`
    Name  string `json:"name"`
    Email string `json:"email"`
}

func ProcessCSVStream(w http.ResponseWriter, r *http.Request) {
    file, err := os.Open("data.csv")
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    defer file.Close()
    
    reader := csv.NewReader(file)
    w.Header().Set("Content-Type", "application/json")
    
    // เริ่ม JSON array
    w.Write([]byte("["))
    
    first := true
    for {
        row, err := reader.Read()
        if err == io.EOF {
            break
        }
        if err != nil {
            http.Error(w, err.Error(), http.StatusInternalServerError)
            return
        }
        
        // Skip header row
        if first && row[0] == "id" {
            first = false
            continue
        }
        
        // Process row
        processed := ProcessedRow{
            ID:    row[0],
            Name:  row[1],
            Email: row[2],
        }
        
        // Add comma separator
        if !first {
            w.Write([]byte(","))
        }
        first = false
        
        // Stream JSON object
        jsonData, _ := json.Marshal(processed)
        w.Write(jsonData)
        
        // Flush to client (สำหรับ real-time streaming)
        if flusher, ok := w.(http.Flusher); ok {
            flusher.Flush()
        }
    }
    
    // ปิด JSON array
    w.Write([]byte("]"))
}

// ✅ Channel-based processing
func ProcessCSVWithChannels(w http.ResponseWriter, r *http.Request) {
    file, err := os.Open("data.csv")
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    defer file.Close()
    
    reader := csv.NewReader(file)
    rowChan := make(chan []string, 100)
    resultChan := make(chan ProcessedRow, 100)
    
    // Goroutine สำหรับอ่านไฟล์
    go func() {
        defer close(rowChan)
        for {
            row, err := reader.Read()
            if err == io.EOF {
                break
            }
            if err != nil {
                continue
            }
            rowChan <- row
        }
    }()
    
    // Goroutine สำหรับประมวลผล
    go func() {
        defer close(resultChan)
        for row := range rowChan {
            if len(row) >= 3 {
                processed := ProcessedRow{
                    ID:    row[0],
                    Name:  row[1], 
                    Email: row[2],
                }
                resultChan <- processed
            }
        }
    }()
    
    // ส่งผลลัพธ์
    w.Header().Set("Content-Type", "application/json")
    results := []ProcessedRow{}
    for processed := range resultChan {
        results = append(results, processed)
    }
    
    json.NewEncoder(w).Encode(results)
}
```

### 3. **Concurrency**

#### Worker Pool Pattern
```go
import (
    "context"
    "math/rand"
    "runtime"
    "sync"
)

// Job represents a heavy computation task
type Job struct {
    ID   int         `json:"id"`
    Data interface{} `json:"data"`
}

// Result represents the computation result
type Result struct {
    JobID  int         `json:"job_id"`
    Value  float64     `json:"value"`
    Error  error       `json:"error,omitempty"`
}

// Worker pool for heavy computations
type WorkerPool struct {
    jobs       chan Job
    results    chan Result
    workers    int
    ctx        context.Context
    cancel     context.CancelFunc
    wg         sync.WaitGroup
}

// NewWorkerPool creates a new worker pool
func NewWorkerPool(numWorkers int) *WorkerPool {
    ctx, cancel := context.WithCancel(context.Background())
    
    pool := &WorkerPool{
        jobs:    make(chan Job, 100),
        results: make(chan Result, 100),
        workers: numWorkers,
        ctx:     ctx,
        cancel:  cancel,
    }
    
    // Start workers
    for i := 0; i < numWorkers; i++ {
        go pool.worker(i)
    }
    
    return pool
}

// worker processes jobs
func (wp *WorkerPool) worker(id int) {
    defer wp.wg.Done()
    wp.wg.Add(1)
    
    for {
        select {
        case job := <-wp.jobs:
            result := wp.heavyComputation(job)
            wp.results <- result
            
        case <-wp.ctx.Done():
            return
        }
    }
}

// heavyComputation performs CPU-intensive task
func (wp *WorkerPool) heavyComputation(job Job) Result {
    // CPU-intensive task
    var result float64
    for i := 0; i < 1000000000; i++ {
        result += rand.Float64()
    }
    
    return Result{
        JobID: job.ID,
        Value: result,
    }
}

// SubmitJob adds a job to the queue
func (wp *WorkerPool) SubmitJob(job Job) {
    select {
    case wp.jobs <- job:
    case <-wp.ctx.Done():
    }
}

// GetResult retrieves a result
func (wp *WorkerPool) GetResult() Result {
    return <-wp.results
}

// Close shuts down the worker pool
func (wp *WorkerPool) Close() {
    wp.cancel()
    wp.wg.Wait()
    close(wp.jobs)
    close(wp.results)
}

// Global worker pool
var globalWorkerPool = NewWorkerPool(runtime.NumCPU())

// HTTP handler for heavy computation
func HeavyTaskHandler(w http.ResponseWriter, r *http.Request) {
    var requestData Job
    if err := json.NewDecoder(r.Body).Decode(&requestData); err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }
    
    // Submit job to worker pool
    globalWorkerPool.SubmitJob(requestData)
    
    // Wait for result
    result := globalWorkerPool.GetResult()
    
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(result)
}

// ✅ Alternative: Direct goroutine approach
func HeavyTaskDirectGoroutine(w http.ResponseWriter, r *http.Request) {
    var requestData Job
    if err := json.NewDecoder(r.Body).Decode(&requestData); err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }
    
    resultChan := make(chan Result, 1)
    
    // Launch goroutine for computation
    go func() {
        defer close(resultChan)
        
        // CPU-intensive task
        var value float64
        for i := 0; i < 1000000000; i++ {
            value += rand.Float64()
        }
        
        resultChan <- Result{
            JobID: requestData.ID,
            Value: value,
        }
    }()
    
    // Wait for result with timeout
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()
    
    select {
    case result := <-resultChan:
        w.Header().Set("Content-Type", "application/json")
        json.NewEncoder(w).Encode(result)
        
    case <-ctx.Done():
        http.Error(w, "Request timeout", http.StatusRequestTimeout)
    }
}
```

## Database Performance

### 1. **Query Optimization**

#### Index Usage
```sql
-- ❌ ไม่มี index
SELECT * FROM users WHERE email = 'john@example.com';
-- Full table scan

-- ✅ มี index
CREATE INDEX idx_users_email ON users(email);
SELECT * FROM users WHERE email = 'john@example.com';
-- Index seek
```

#### N+1 Problem
```go
// ❌ N+1 Query Problem
func GetUsersWithOrders() ([]UserWithOrders, error) {
    users, err := userRepo.FindAll() // 1 query
    if err != nil {
        return nil, err
    }
    
    var result []UserWithOrders
    for _, user := range users {
        orders, err := orderRepo.FindByUserID(user.ID) // N queries ❌
        if err != nil {
            return nil, err
        }
        result = append(result, UserWithOrders{
            User:   user,
            Orders: orders,
        })
    }
    return result, nil
}

// ✅ แก้ไขด้วย JOIN query
const getUsersWithOrdersQuery = `
    SELECT 
        u.id as user_id, u.name, u.email,
        o.id as order_id, o.total, o.created_at
    FROM users u
    LEFT JOIN orders o ON u.id = o.user_id
`

func GetUsersWithOrdersOptimized() ([]UserWithOrders, error) {
    rows, err := db.Query(getUsersWithOrdersQuery)
    if err != nil {
        return nil, err
    }
    defer rows.Close()
    
    userMap := make(map[int]*UserWithOrders)
    
    for rows.Next() {
        var userID, orderID sql.NullInt64
        var name, email string
        var total sql.NullFloat64
        var createdAt sql.NullTime
        
        err := rows.Scan(&userID, &name, &email, &orderID, &total, &createdAt)
        if err != nil {
            return nil, err
        }
        
        // Create user if not exists
        if _, exists := userMap[int(userID.Int64)]; !exists {
            userMap[int(userID.Int64)] = &UserWithOrders{
                User: User{
                    ID:    int(userID.Int64),
                    Name:  name,
                    Email: email,
                },
                Orders: []Order{},
            }
        }
        
        // Add order if exists
        if orderID.Valid {
            order := Order{
                ID:        int(orderID.Int64),
                UserID:    int(userID.Int64),
                Total:     total.Float64,
                CreatedAt: createdAt.Time,
            }
            userMap[int(userID.Int64)].Orders = append(userMap[int(userID.Int64)].Orders, order)
        }
    }
    
    // Convert map to slice
    var result []UserWithOrders
    for _, userWithOrders := range userMap {
        result = append(result, *userWithOrders)
    }
    
    return result, nil
}

// ✅ หรือใช้ Batch Loading Pattern
type BatchLoader struct {
    userRepo  UserRepository
    orderRepo OrderRepository
}

func (bl *BatchLoader) LoadUsersWithOrders(userIDs []int) (map[int][]Order, error) {
    // Load all orders for all users in one query
    orders, err := bl.orderRepo.FindByUserIDs(userIDs)
    if err != nil {
        return nil, err
    }
    
    // Group orders by user ID
    ordersByUser := make(map[int][]Order)
    for _, order := range orders {
        ordersByUser[order.UserID] = append(ordersByUser[order.UserID], order)
    }
    
    return ordersByUser, nil
}
```

### 2. **Connection Pooling**
```go
import (
    "database/sql"
    "time"
    _ "github.com/go-sql-driver/mysql"
)

// Database connection pool configuration
func setupDatabase() (*sql.DB, error) {
    dsn := "app_user:password@tcp(localhost:3306)/myapp?parseTime=true"
    
    db, err := sql.Open("mysql", dsn)
    if err != nil {
        return nil, err
    }
    
    // Connection pool settings
    db.SetMaxOpenConns(10)                 // จำกัดจำนวน connection สูงสุด
    db.SetMaxIdleConns(5)                  // จำนวน idle connections
    db.SetConnMaxLifetime(time.Hour)       // อายุของ connection
    db.SetConnMaxIdleTime(10 * time.Minute) // idle timeout
    
    // Test connection
    if err := db.Ping(); err != nil {
        return nil, err
    }
    
    return db, nil
}

// Global database instance
var db *sql.DB

func init() {
    var err error
    db, err = setupDatabase()
    if err != nil {
        panic("Failed to connect to database: " + err.Error())
    }
}

// Repository pattern with connection pooling
type UserRepository struct {
    db *sql.DB
}

func NewUserRepository(database *sql.DB) *UserRepository {
    return &UserRepository{db: database}
}

func (r *UserRepository) GetUsers() ([]User, error) {
    query := "SELECT id, name, email FROM users"
    
    rows, err := r.db.Query(query)
    if err != nil {
        return nil, err
    }
    defer rows.Close() // สำคัญ: ปิด rows เพื่อคืน connection
    
    var users []User
    for rows.Next() {
        var user User
        err := rows.Scan(&user.ID, &user.Name, &user.Email)
        if err != nil {
            return nil, err
        }
        users = append(users, user)
    }
    
    return users, rows.Err()
}

func (r *UserRepository) GetUserByID(id int) (*User, error) {
    query := "SELECT id, name, email FROM users WHERE id = ?"
    
    var user User
    err := r.db.QueryRow(query, id).Scan(&user.ID, &user.Name, &user.Email)
    if err != nil {
        return nil, err
    }
    
    return &user, nil
}

// ✅ Transaction with connection pooling
func (r *UserRepository) CreateUserWithProfile(user User, profile Profile) error {
    tx, err := r.db.Begin()
    if err != nil {
        return err
    }
    defer tx.Rollback() // rollback ถ้ามี error
    
    // Insert user
    result, err := tx.Exec(
        "INSERT INTO users (name, email) VALUES (?, ?)",
        user.Name, user.Email,
    )
    if err != nil {
        return err
    }
    
    userID, err := result.LastInsertId()
    if err != nil {
        return err
    }
    
    // Insert profile
    _, err = tx.Exec(
        "INSERT INTO profiles (user_id, bio, avatar) VALUES (?, ?, ?)",
        userID, profile.Bio, profile.Avatar,
    )
    if err != nil {
        return err
    }
    
    // Commit transaction
    return tx.Commit()
}

// ✅ Context-aware queries with timeout
import "context"

func (r *UserRepository) GetUsersWithContext(ctx context.Context) ([]User, error) {
    query := "SELECT id, name, email FROM users"
    
    rows, err := r.db.QueryContext(ctx, query)
    if err != nil {
        return nil, err
    }
    defer rows.Close()
    
    var users []User
    for rows.Next() {
        // Check if context is cancelled
        select {
        case <-ctx.Done():
            return nil, ctx.Err()
        default:
        }
        
        var user User
        err := rows.Scan(&user.ID, &user.Name, &user.Email)
        if err != nil {
            return nil, err
        }
        users = append(users, user)
    }
    
    return users, rows.Err()
}

// HTTP handler with timeout
func GetUsersHandler(w http.ResponseWriter, r *http.Request) {
    ctx, cancel := context.WithTimeout(r.Context(), 5*time.Second)
    defer cancel()
    
    userRepo := NewUserRepository(db)
    users, err := userRepo.GetUsersWithContext(ctx)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(users)
}
```

### 3. **Database Optimization**
```sql
-- Partitioning
CREATE TABLE orders (
    id BIGINT,
    user_id INT,
    created_at DATE,
    total DECIMAL(10,2)
) PARTITION BY RANGE (YEAR(created_at)) (
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025)
);

-- Archiving old data
CREATE EVENT archive_old_orders
ON SCHEDULE EVERY 1 MONTH
DO
BEGIN
    INSERT INTO orders_archive 
    SELECT * FROM orders 
    WHERE created_at < DATE_SUB(NOW(), INTERVAL 1 YEAR);
    
    DELETE FROM orders 
    WHERE created_at < DATE_SUB(NOW(), INTERVAL 1 YEAR);
END;
```

## Network Performance

### 1. **HTTP Optimization**

#### HTTP/2 & Compression
```go
import (
    "compress/gzip"
    "crypto/tls"
    "net/http"
    "strings"
)

// GZIP compression middleware
func gzipMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Check if client accepts gzip
        if !strings.Contains(r.Header.Get("Accept-Encoding"), "gzip") {
            next.ServeHTTP(w, r)
            return
        }
        
        // Set compression headers
        w.Header().Set("Content-Encoding", "gzip")
        w.Header().Set("Content-Type", "text/html")
        
        // Create gzip writer
        gzipWriter := gzip.NewWriter(w)
        defer gzipWriter.Close()
        
        // Wrap response writer
        gzipResponseWriter := &gzipResponseWriter{
            ResponseWriter: w,
            Writer:         gzipWriter,
        }
        
        next.ServeHTTP(gzipResponseWriter, r)
    })
}

type gzipResponseWriter struct {
    http.ResponseWriter
    Writer *gzip.Writer
}

func (grw *gzipResponseWriter) Write(data []byte) (int, error) {
    return grw.Writer.Write(data)
}

// HTTP/2 Server setup
func setupHTTP2Server() *http.Server {
    mux := http.NewServeMux()
    
    // Add gzip middleware
    gzipHandler := gzipMiddleware(mux)
    
    // Routes
    mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Content-Type", "text/html")
        w.Write([]byte("<h1>Hello HTTP/2!</h1>"))
    })
    
    mux.HandleFunc("/api/users", GetUsersHandler)
    
    // TLS configuration for HTTP/2
    tlsConfig := &tls.Config{
        NextProtos: []string{"h2", "http/1.1"}, // Enable HTTP/2
    }
    
    server := &http.Server{
        Addr:      ":8443",
        Handler:   gzipHandler,
        TLSConfig: tlsConfig,
    }
    
    return server
}

// Start HTTP/2 server
func main() {
    server := setupHTTP2Server()
    
    // Start HTTPS server with HTTP/2
    log.Println("Starting HTTP/2 server on :8443")
    log.Fatal(server.ListenAndServeTLS("certificate.pem", "private-key.pem"))
}

// ✅ Alternative: Using Gin framework with compression
import (
    "github.com/gin-contrib/gzip"
    "github.com/gin-gonic/gin"
)

func setupGinWithCompression() *gin.Engine {
    r := gin.Default()
    
    // Add gzip compression middleware
    r.Use(gzip.Gzip(gzip.DefaultCompression))
    
    // Routes
    r.GET("/", func(c *gin.Context) {
        c.HTML(http.StatusOK, "index.html", gin.H{
            "title": "Hello HTTP/2!",
        })
    })
    
    r.GET("/api/users", func(c *gin.Context) {
        users := []User{
            {ID: 1, Name: "John", Email: "john@example.com"},
            {ID: 2, Name: "Jane", Email: "jane@example.com"},
        }
        c.JSON(http.StatusOK, users)
    })
    
    return r
}
```

#### Resource Hints
```html
<!-- DNS prefetch -->
<link rel="dns-prefetch" href="//api.example.com">

<!-- Preconnect -->
<link rel="preconnect" href="https://fonts.googleapis.com">

<!-- Prefetch ทรัพยากรที่จะใช้หลังจากนี้ -->
<link rel="prefetch" href="/next-page.html">

<!-- Preload ทรัพยากรที่จะใช้ในหน้านี้ -->
<link rel="preload" href="/critical.css" as="style">
```

### 2. **CDN Configuration**
```go
import (
    "os"
    "path/filepath"
    "text/template"
)

// CDN configuration
const CDN_URL = "https://cdn.example.com"

type AssetConfig struct {
    IsProd bool
    CDNURL string
}

var assetConfig = AssetConfig{
    IsProd: os.Getenv("GO_ENV") == "production",
    CDNURL: CDN_URL,
}

// GetAssetURL returns the appropriate URL for assets
func GetAssetURL(path string) string {
    if assetConfig.IsProd {
        return assetConfig.CDNURL + path
    }
    return path
}

// Template helper functions
var funcMap = template.FuncMap{
    "asset": GetAssetURL,
}

// HTML template example
const templateHTML = `
<!DOCTYPE html>
<html>
<head>
    <title>{{.Title}}</title>
    <link rel="stylesheet" href="{{asset "/css/app.css"}}">
</head>
<body>
    <img src="{{asset "/images/logo.png"}}" alt="Logo">
    <script src="{{asset "/js/app.js"}}"></script>
</body>
</html>
`

// HTTP handler with CDN assets
func HomeHandler(w http.ResponseWriter, r *http.Request) {
    tmpl, err := template.New("home").Funcs(funcMap).Parse(templateHTML)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    
    data := struct {
        Title string
    }{
        Title: "My Go App",
    }
    
    w.Header().Set("Content-Type", "text/html")
    tmpl.Execute(w, data)
}

// ✅ Static file handler with CDN fallback
func StaticFileHandler(w http.ResponseWriter, r *http.Request) {
    // In production, redirect to CDN
    if assetConfig.IsProd {
        cdnURL := assetConfig.CDNURL + r.URL.Path
        http.Redirect(w, r, cdnURL, http.StatusMovedPermanently)
        return
    }
    
    // In development, serve locally
    http.ServeFile(w, r, filepath.Join("static", r.URL.Path))
}

// ✅ Middleware for CDN headers
func cdnCacheMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Set cache headers for static assets
        if filepath.Ext(r.URL.Path) != "" {
            // Cache static files for 1 year
            w.Header().Set("Cache-Control", "public, max-age=31536000")
            w.Header().Set("Expires", time.Now().AddDate(1, 0, 0).Format(http.TimeFormat))
        }
        
        next.ServeHTTP(w, r)
    })
}

// Setup routes with CDN configuration
func setupRoutes() *http.ServeMux {
    mux := http.NewServeMux()
    
    // Home page
    mux.HandleFunc("/", HomeHandler)
    
    // API routes
    mux.HandleFunc("/api/users", GetUsersHandler)
    
    // Static files (with CDN fallback)
    mux.HandleFunc("/static/", StaticFileHandler)
    mux.HandleFunc("/images/", StaticFileHandler)
    mux.HandleFunc("/css/", StaticFileHandler)
    mux.HandleFunc("/js/", StaticFileHandler)
    
    return mux
}

// ✅ Using Gin framework with CDN
import "github.com/gin-gonic/gin"

func setupGinWithCDN() *gin.Engine {
    r := gin.Default()
    
    // Load HTML templates with custom functions
    r.SetFuncMap(funcMap)
    r.LoadHTMLGlob("templates/*")
    
    // Static files
    if !assetConfig.IsProd {
        r.Static("/static", "./static")
    } else {
        // In production, redirect to CDN
        r.GET("/static/*filepath", func(c *gin.Context) {
            cdnURL := assetConfig.CDNURL + c.Request.URL.Path
            c.Redirect(http.StatusMovedPermanently, cdnURL)
        })
    }
    
    r.GET("/", func(c *gin.Context) {
        c.HTML(http.StatusOK, "index.html", gin.H{
            "title": "My Go App",
        })
    })
    
    return r
}
```

## Caching Strategies

### 1. **Browser Caching**
```go
import (
    "crypto/md5"
    "fmt"
    "net/http"
    "path/filepath"
    "time"
)

// Static file server with cache headers
func StaticCacheHandler(w http.ResponseWriter, r *http.Request) {
    filePath := filepath.Join("public", r.URL.Path)
    
    // Get file info for ETag and Last-Modified
    fileInfo, err := os.Stat(filePath)
    if err != nil {
        http.NotFound(w, r)
        return
    }
    
    // Generate ETag based on file modification time and size
    etag := fmt.Sprintf(`"%x-%x"`, fileInfo.ModTime().Unix(), fileInfo.Size())
    lastModified := fileInfo.ModTime().Format(http.TimeFormat)
    
    // Set cache headers
    w.Header().Set("Cache-Control", "public, max-age=31536000") // 1 year
    w.Header().Set("ETag", etag)
    w.Header().Set("Last-Modified", lastModified)
    
    // Check if client has cached version
    if clientETag := r.Header.Get("If-None-Match"); clientETag == etag {
        w.WriteHeader(http.StatusNotModified)
        return
    }
    
    if clientModified := r.Header.Get("If-Modified-Since"); clientModified != "" {
        if clientTime, err := time.Parse(http.TimeFormat, clientModified); err == nil {
            if !fileInfo.ModTime().After(clientTime) {
                w.WriteHeader(http.StatusNotModified)
                return
            }
        }
    }
    
    // Serve the file
    http.ServeFile(w, r, filePath)
}

// API cache headers
type Config struct {
    AppName string `json:"app_name"`
    Version string `json:"version"`
    Debug   bool   `json:"debug"`
}

func generateETag(data interface{}) string {
    hash := md5.Sum([]byte(fmt.Sprintf("%+v", data)))
    return fmt.Sprintf(`"%x"`, hash)
}

func ConfigHandler(w http.ResponseWriter, r *http.Request) {
    config := Config{
        AppName: "My Go App",
        Version: "1.0.0", 
        Debug:   false,
    }
    
    etag := generateETag(config)
    
    // Set cache headers
    w.Header().Set("Cache-Control", "public, max-age=3600") // 1 hour
    w.Header().Set("ETag", etag)
    
    // Check ETag
    if clientETag := r.Header.Get("If-None-Match"); clientETag == etag {
        w.WriteHeader(http.StatusNotModified)
        return
    }
    
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(config)
}

// ✅ Cache middleware
func cacheMiddleware(maxAge time.Duration) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            // Set cache headers
            w.Header().Set("Cache-Control", fmt.Sprintf("public, max-age=%d", int(maxAge.Seconds())))
            w.Header().Set("Expires", time.Now().Add(maxAge).Format(http.TimeFormat))
            
            next.ServeHTTP(w, r)
        })
    }
}

// Usage
func setupCachedRoutes() *http.ServeMux {
    mux := http.NewServeMux()
    
    // Static files with 1 year cache
    staticHandler := cacheMiddleware(365 * 24 * time.Hour)(http.HandlerFunc(StaticCacheHandler))
    mux.Handle("/static/", staticHandler)
    
    // API with 1 hour cache
    configHandler := cacheMiddleware(time.Hour)(http.HandlerFunc(ConfigHandler))
    mux.Handle("/api/config", configHandler)
    
    return mux
}
```

### 2. **Application Caching**
```go
import (
    "encoding/json"
    "sync"
    "time"
)

// Memory cache implementation
type CacheItem struct {
    Value     interface{}
    Timestamp time.Time
    TTL       time.Duration
}

type MemoryCache struct {
    items sync.Map
    mu    sync.RWMutex
}

func NewMemoryCache() *MemoryCache {
    cache := &MemoryCache{}
    
    // Background cleanup goroutine
    go cache.cleanup()
    
    return cache
}

func (mc *MemoryCache) Set(key string, value interface{}, ttl time.Duration) {
    item := CacheItem{
        Value:     value,
        Timestamp: time.Now(),
        TTL:       ttl,
    }
    mc.items.Store(key, item)
}

func (mc *MemoryCache) Get(key string) (interface{}, bool) {
    if value, exists := mc.items.Load(key); exists {
        item := value.(CacheItem)
        
        // Check if expired
        if time.Since(item.Timestamp) > item.TTL {
            mc.items.Delete(key)
            return nil, false
        }
        
        return item.Value, true
    }
    return nil, false
}

func (mc *MemoryCache) Delete(key string) {
    mc.items.Delete(key)
}

func (mc *MemoryCache) cleanup() {
    ticker := time.NewTicker(time.Minute)
    defer ticker.Stop()
    
    for range ticker.C {
        mc.items.Range(func(key, value interface{}) bool {
            item := value.(CacheItem)
            if time.Since(item.Timestamp) > item.TTL {
                mc.items.Delete(key)
            }
            return true
        })
    }
}

// Global cache instance
var globalCache = NewMemoryCache()

// Memoization function
func Memoize[T any](fn func(...interface{}) (T, error), ttl time.Duration) func(...interface{}) (T, error) {
    return func(args ...interface{}) (T, error) {
        var zero T
        
        // Create cache key
        keyBytes, _ := json.Marshal(args)
        key := string(keyBytes)
        
        // Check cache
        if cached, exists := globalCache.Get(key); exists {
            return cached.(T), nil
        }
        
        // Execute function
        result, err := fn(args...)
        if err != nil {
            return zero, err
        }
        
        // Cache result
        globalCache.Set(key, result, ttl)
        
        return result, nil
    }
}

// ✅ Example usage with database query
type User struct {
    ID    int    `json:"id"`
    Name  string `json:"name"`
    Email string `json:"email"`
}

// Original expensive function
func getExpensiveDataOriginal(id int) (User, error) {
    // Simulate expensive database query
    time.Sleep(100 * time.Millisecond)
    
    // Mock database query
    return User{
        ID:    id,
        Name:  fmt.Sprintf("User %d", id),
        Email: fmt.Sprintf("user%d@example.com", id),
    }, nil
}

// Memoized version
var getExpensiveData = Memoize(
    func(args ...interface{}) (User, error) {
        id := args[0].(int)
        return getExpensiveDataOriginal(id)
    },
    5*time.Minute,
)

// HTTP handler using cached function
func GetUserHandler(w http.ResponseWriter, r *http.Request) {
    idStr := r.URL.Query().Get("id")
    id, err := strconv.Atoi(idStr)
    if err != nil {
        http.Error(w, "Invalid ID", http.StatusBadRequest)
        return
    }
    
    user, err := getExpensiveData(id)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(user)
}

// ✅ Alternative: Direct cache usage
func GetUserHandlerDirect(w http.ResponseWriter, r *http.Request) {
    idStr := r.URL.Query().Get("id")
    id, err := strconv.Atoi(idStr)
    if err != nil {
        http.Error(w, "Invalid ID", http.StatusBadRequest)
        return
    }
    
    cacheKey := fmt.Sprintf("user:%d", id)
    
    // Check cache first
    if cached, exists := globalCache.Get(cacheKey); exists {
        w.Header().Set("Content-Type", "application/json")
        json.NewEncoder(w).Encode(cached)
        return
    }
    
    // Get from database
    user, err := getExpensiveDataOriginal(id)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    
    // Cache the result
    globalCache.Set(cacheKey, user, 5*time.Minute)
    
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(user)
}
```

### 3. **Redis Caching**
```go
import (
    "context"
    "encoding/json"
    "time"
    
    "github.com/go-redis/redis/v8"
)

// Redis client setup
var redisClient *redis.Client

func init() {
    redisClient = redis.NewClient(&redis.Options{
        Addr:     "localhost:6379",
        Password: "", // no password
        DB:       0,  // default DB
        PoolSize: 10, // connection pool size
    })
    
    // Test connection
    ctx := context.Background()
    _, err := redisClient.Ping(ctx).Result()
    if err != nil {
        panic("Failed to connect to Redis: " + err.Error())
    }
}

// Redis cache wrapper
type RedisCache struct {
    client *redis.Client
}

func NewRedisCache(client *redis.Client) *RedisCache {
    return &RedisCache{client: client}
}

func (rc *RedisCache) Set(ctx context.Context, key string, value interface{}, ttl time.Duration) error {
    data, err := json.Marshal(value)
    if err != nil {
        return err
    }
    
    return rc.client.Set(ctx, key, data, ttl).Err()
}

func (rc *RedisCache) Get(ctx context.Context, key string, dest interface{}) error {
    data, err := rc.client.Get(ctx, key).Result()
    if err != nil {
        return err
    }
    
    return json.Unmarshal([]byte(data), dest)
}

func (rc *RedisCache) Delete(ctx context.Context, key string) error {
    return rc.client.Del(ctx, key).Err()
}

func (rc *RedisCache) Exists(ctx context.Context, key string) (bool, error) {
    count, err := rc.client.Exists(ctx, key).Result()
    return count > 0, err
}

// Global Redis cache instance
var cache = NewRedisCache(redisClient)

// Cache middleware
func redisCacheMiddleware(ttl time.Duration) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            ctx := r.Context()
            key := fmt.Sprintf("cache:%s", r.URL.Path)
            
            // Try to get from cache
            var cachedData interface{}
            if err := cache.Get(ctx, key, &cachedData); err == nil {
                w.Header().Set("Content-Type", "application/json")
                w.Header().Set("X-Cache", "HIT")
                json.NewEncoder(w).Encode(cachedData)
                return
            }
            
            // Wrap response writer to capture response
            wrapper := &responseWrapper{
                ResponseWriter: w,
                body:          &bytes.Buffer{},
            }
            
            // Call next handler
            next.ServeHTTP(wrapper, r)
            
            // Cache the response if status is OK
            if wrapper.statusCode == 0 || wrapper.statusCode == http.StatusOK {
                var responseData interface{}
                if err := json.Unmarshal(wrapper.body.Bytes(), &responseData); err == nil {
                    cache.Set(ctx, key, responseData, ttl)
                }
                
                w.Header().Set("X-Cache", "MISS")
                w.Write(wrapper.body.Bytes())
            }
        })
    }
}

// Response wrapper to capture response data
type responseWrapper struct {
    http.ResponseWriter
    body       *bytes.Buffer
    statusCode int
}

func (rw *responseWrapper) Write(data []byte) (int, error) {
    return rw.body.Write(data)
}

func (rw *responseWrapper) WriteHeader(statusCode int) {
    rw.statusCode = statusCode
    rw.ResponseWriter.WriteHeader(statusCode)
}

// ✅ Direct Redis usage in handlers
func GetUsersWithRedisCache(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    cacheKey := "users:all"
    
    // Try cache first
    var users []User
    if err := cache.Get(ctx, cacheKey, &users); err == nil {
        w.Header().Set("Content-Type", "application/json")
        w.Header().Set("X-Cache", "HIT")
        json.NewEncoder(w).Encode(users)
        return
    }
    
    // Get from database
    users, err := userRepo.GetUsers()
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    
    // Cache for 10 minutes
    cache.Set(ctx, cacheKey, users, 10*time.Minute)
    
    w.Header().Set("Content-Type", "application/json")
    w.Header().Set("X-Cache", "MISS")
    json.NewEncoder(w).Encode(users)
}

// ✅ Cache invalidation
func CreateUserHandler(w http.ResponseWriter, r *http.Request) {
    var user User
    if err := json.NewDecoder(r.Body).Decode(&user); err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }
    
    // Create user in database
    createdUser, err := userRepo.CreateUser(user)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    
    // Invalidate cache
    ctx := r.Context()
    cache.Delete(ctx, "users:all")
    
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(createdUser)
}

// Setup routes with Redis caching
func setupRedisRoutes() *http.ServeMux {
    mux := http.NewServeMux()
    
    // Cached route with middleware
    usersHandler := redisCacheMiddleware(10 * time.Minute)(http.HandlerFunc(GetUsersHandler))
    mux.Handle("/api/users", usersHandler)
    
    // Direct Redis usage
    mux.HandleFunc("/api/users-redis", GetUsersWithRedisCache)
    mux.HandleFunc("/api/users-create", CreateUserHandler)
    
    return mux
}
```

## Load Testing

### 1. **Artillery**
```yaml
# load-test.yml
config:
  target: 'http://localhost:3000'
  phases:
    - duration: 60
      arrivalRate: 10 # 10 requests/second
    - duration: 120  
      arrivalRate: 50 # 50 requests/second

scenarios:
  - name: "API Load Test"
    flow:
      - get:
          url: "/api/users"
      - post:
          url: "/api/users"
          json:
            name: "Test User"
            email: "test@example.com"
```

```bash
# รัน load test
npm install -g artillery
artillery run load-test.yml
```

### 2. **Apache Bench (ab)**
```bash
# Test 1000 requests, 10 concurrent
ab -n 1000 -c 10 http://localhost:3000/api/users

# ผลลัพธ์:
# Requests per second: 150.23 [#/sec]
# Time per request: 66.565 [ms]
# Transfer rate: 45.67 [Kbytes/sec]
```

### 3. **Custom Load Testing**
```go
// load-test.go
package main

import (
    "fmt"
    "net/http"
    "sync"
    "sync/atomic"
    "time"
)

type LoadTestResult struct {
    TotalRequests   int64
    CompletedOK     int64
    Errors          int64
    Duration        time.Duration
    RequestsPerSec  float64
    AvgResponseTime time.Duration
}

func loadTest(url string, concurrent int, total int) LoadTestResult {
    var (
        completed    int64
        errors       int64
        totalRespTime int64
    )
    
    startTime := time.Now()
    
    // Channel to limit concurrent requests
    semaphore := make(chan struct{}, concurrent)
    var wg sync.WaitGroup
    
    // HTTP client with timeout
    client := &http.Client{
        Timeout: 30 * time.Second,
    }
    
    for i := 0; i < total; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            
            // Acquire semaphore
            semaphore <- struct{}{}
            defer func() { <-semaphore }()
            
            // Make request
            reqStart := time.Now()
            resp, err := client.Get(url)
            reqDuration := time.Since(reqStart)
            
            if err != nil {
                atomic.AddInt64(&errors, 1)
                return
            }
            defer resp.Body.Close()
            
            if resp.StatusCode == http.StatusOK {
                atomic.AddInt64(&completed, 1)
                atomic.AddInt64(&totalRespTime, int64(reqDuration))
            } else {
                atomic.AddInt64(&errors, 1)
            }
        }()
    }
    
    wg.Wait()
    duration := time.Since(startTime)
    
    result := LoadTestResult{
        TotalRequests:  int64(total),
        CompletedOK:    completed,
        Errors:         errors,
        Duration:       duration,
        RequestsPerSec: float64(completed) / duration.Seconds(),
    }
    
    if completed > 0 {
        result.AvgResponseTime = time.Duration(totalRespTime / completed)
    }
    
    return result
}

func printResults(result LoadTestResult) {
    fmt.Printf("Load Test Results:\n")
    fmt.Printf("==================\n")
    fmt.Printf("Total Requests: %d\n", result.TotalRequests)
    fmt.Printf("Completed OK: %d\n", result.CompletedOK)
    fmt.Printf("Errors: %d\n", result.Errors)
    fmt.Printf("Duration: %v\n", result.Duration)
    fmt.Printf("Requests/sec: %.2f\n", result.RequestsPerSec)
    fmt.Printf("Avg Response Time: %v\n", result.AvgResponseTime)
    fmt.Printf("Success Rate: %.2f%%\n", float64(result.CompletedOK)/float64(result.TotalRequests)*100)
}

// ✅ Advanced load testing with different scenarios
type LoadTestConfig struct {
    URL            string
    Concurrent     int
    TotalRequests  int
    RampUpDuration time.Duration
    TestDuration   time.Duration
}

func advancedLoadTest(config LoadTestConfig) LoadTestResult {
    var (
        completed    int64
        errors       int64
        totalRespTime int64
    )
    
    startTime := time.Now()
    
    // Channel for controlling request rate
    requestTicker := time.NewTicker(time.Duration(int64(time.Second) / int64(config.Concurrent)))
    defer requestTicker.Stop()
    
    // Test duration timer
    testTimer := time.NewTimer(config.TestDuration)
    defer testTimer.Stop()
    
    // HTTP client
    client := &http.Client{
        Timeout: 10 * time.Second,
    }
    
    var wg sync.WaitGroup
    requestCount := 0
    
    for {
        select {
        case <-testTimer.C:
            // Test duration reached
            wg.Wait()
            goto results
            
        case <-requestTicker.C:
            if requestCount >= config.TotalRequests {
                wg.Wait()
                goto results
            }
            
            requestCount++
            wg.Add(1)
            
            go func() {
                defer wg.Done()
                
                reqStart := time.Now()
                resp, err := client.Get(config.URL)
                reqDuration := time.Since(reqStart)
                
                if err != nil {
                    atomic.AddInt64(&errors, 1)
                    return
                }
                defer resp.Body.Close()
                
                if resp.StatusCode == http.StatusOK {
                    atomic.AddInt64(&completed, 1)
                    atomic.AddInt64(&totalRespTime, int64(reqDuration))
                } else {
                    atomic.AddInt64(&errors, 1)
                }
            }()
        }
    }
    
results:
    duration := time.Since(startTime)
    
    result := LoadTestResult{
        TotalRequests:  int64(requestCount),
        CompletedOK:    completed,
        Errors:         errors,
        Duration:       duration,
        RequestsPerSec: float64(completed) / duration.Seconds(),
    }
    
    if completed > 0 {
        result.AvgResponseTime = time.Duration(totalRespTime / completed)
    }
    
    return result
}

// Main function to run load test
func main() {
    url := "http://localhost:8080/api/users"
    
    fmt.Println("Running simple load test...")
    result := loadTest(url, 10, 1000)
    printResults(result)
    
    fmt.Println("\nRunning advanced load test...")
    config := LoadTestConfig{
        URL:            url,
        Concurrent:     50,
        TotalRequests:  5000,
        RampUpDuration: 30 * time.Second,
        TestDuration:   60 * time.Second,
    }
    
    advResult := advancedLoadTest(config)
    printResults(advResult)
}
```

## 🎯 Best Practices

### 1. **Monitor แล้วค่อย Optimize**
```go
import (
    "log"
    "runtime"
    "time"
)

// Performance monitor
type Monitor struct {
    timers map[string]time.Time
}

func NewMonitor() *Monitor {
    return &Monitor{
        timers: make(map[string]time.Time),
    }
}

func (m *Monitor) Time(label string) {
    m.timers[label] = time.Now()
}

func (m *Monitor) TimeEnd(label string) {
    if start, exists := m.timers[label]; exists {
        duration := time.Since(start)
        log.Printf("%s: %v", label, duration)
        delete(m.timers, label)
    }
}

func (m *Monitor) Memory() runtime.MemStats {
    var memStats runtime.MemStats
    runtime.ReadMemStats(&memStats)
    return memStats
}

func (m *Monitor) LogMemory() {
    memStats := m.Memory()
    log.Printf("Memory Usage - Alloc: %d KB, Sys: %d KB, NumGC: %d",
        memStats.Alloc/1024,
        memStats.Sys/1024,
        memStats.NumGC,
    )
}

// Global monitor instance
var monitor = NewMonitor()

// ✅ Usage example
func expensiveOperation() {
    monitor.Time("expensive-operation")
    defer monitor.TimeEnd("expensive-operation")
    
    // Simulate work
    time.Sleep(100 * time.Millisecond)
    
    // Log memory at the end
    monitor.LogMemory()
}

// ✅ HTTP middleware for monitoring
func monitoringMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        
        // Log memory before
        memBefore := monitor.Memory()
        
        // Call next handler
        next.ServeHTTP(w, r)
        
        // Log performance metrics
        duration := time.Since(start)
        memAfter := monitor.Memory()
        memDiff := memAfter.Alloc - memBefore.Alloc
        
        log.Printf("Request: %s %s | Duration: %v | Memory Diff: %d KB",
            r.Method, r.URL.Path, duration, memDiff/1024)
    })
}
```

### 2. **Optimize ที่มีผลกระทบมากที่สุดก่อน**
```
1. Database queries (มักช้าที่สุด)
2. Network requests 
3. File I/O
4. CPU-intensive computations
5. Memory allocations
```

### 3. **Set Performance Budget**
```go
import (
    "fmt"
    "runtime"
    "time"
)

// Performance budget configuration
type PerformanceBudget struct {
    MaxResponseTime time.Duration // 200ms
    MaxMemoryUsage  uint64        // 100 MB
    MaxCPUUsage     float64       // 70%
    MaxGoroutines   int           // 1000
}

var PERFORMANCE_BUDGET = PerformanceBudget{
    MaxResponseTime: 200 * time.Millisecond,
    MaxMemoryUsage:  100 * 1024 * 1024, // 100 MB
    MaxCPUUsage:     70.0,               // 70%
    MaxGoroutines:   1000,
}

// Performance checker
type PerformanceChecker struct {
    budget PerformanceBudget
}

func NewPerformanceChecker(budget PerformanceBudget) *PerformanceChecker {
    return &PerformanceChecker{budget: budget}
}

func (pc *PerformanceChecker) CheckResponseTime(duration time.Duration) bool {
    if duration > pc.budget.MaxResponseTime {
        log.Printf("❌ Response time budget exceeded: %v > %v", duration, pc.budget.MaxResponseTime)
        return false
    }
    return true
}

func (pc *PerformanceChecker) CheckMemoryUsage() bool {
    var memStats runtime.MemStats
    runtime.ReadMemStats(&memStats)
    
    if memStats.Alloc > pc.budget.MaxMemoryUsage {
        log.Printf("❌ Memory budget exceeded: %d MB > %d MB", 
            memStats.Alloc/1024/1024, 
            pc.budget.MaxMemoryUsage/1024/1024)
        return false
    }
    return true
}

func (pc *PerformanceChecker) CheckGoroutines() bool {
    numGoroutines := runtime.NumGoroutine()
    
    if numGoroutines > pc.budget.MaxGoroutines {
        log.Printf("❌ Goroutine budget exceeded: %d > %d", numGoroutines, pc.budget.MaxGoroutines)
        return false
    }
    return true
}

func (pc *PerformanceChecker) CheckAll() map[string]bool {
    results := make(map[string]bool)
    
    results["memory"] = pc.CheckMemoryUsage()
    results["goroutines"] = pc.CheckGoroutines()
    
    return results
}

// Global performance checker
var performanceChecker = NewPerformanceChecker(PERFORMANCE_BUDGET)

// ✅ Middleware with performance budget checking
func performanceBudgetMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        
        // Call next handler
        next.ServeHTTP(w, r)
        
        // Check response time budget
        duration := time.Since(start)
        if !performanceChecker.CheckResponseTime(duration) {
            w.Header().Set("X-Performance-Warning", "Response time budget exceeded")
        }
        
        // Check other budgets
        performanceChecker.CheckMemoryUsage()
        performanceChecker.CheckGoroutines()
    })
}

// ✅ Performance monitoring endpoint
func PerformanceStatusHandler(w http.ResponseWriter, r *http.Request) {
    var memStats runtime.MemStats
    runtime.ReadMemStats(&memStats)
    
    status := map[string]interface{}{
        "memory_usage_mb":    memStats.Alloc / 1024 / 1024,
        "memory_budget_mb":   PERFORMANCE_BUDGET.MaxMemoryUsage / 1024 / 1024,
        "goroutines":         runtime.NumGoroutine(),
        "goroutine_budget":   PERFORMANCE_BUDGET.MaxGoroutines,
        "response_time_budget_ms": PERFORMANCE_BUDGET.MaxResponseTime.Milliseconds(),
        "cpu_budget_percent": PERFORMANCE_BUDGET.MaxCPUUsage,
    }
    
    // Check if budgets are exceeded
    budgetStatus := performanceChecker.CheckAll()
    status["budget_status"] = budgetStatus
    
    // Set appropriate status code
    allGood := true
    for _, ok := range budgetStatus {
        if !ok {
            allGood = false
            break
        }
    }
    
    if !allGood {
        w.WriteHeader(http.StatusTooManyRequests) // 429
    }
    
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(status)
}
```

## 🚫 สิ่งที่ควรหลีกเลี่ยง

- ❌ **Premature optimization**: อย่า optimize ก่อนมีปัญหา
- ❌ **Over-optimization**: อย่า optimize จนโค้ดอ่านยาก
- ❌ **No measurement**: อย่า optimize โดยไม่วัด
- ❌ **Ignoring user experience**: อย่าลืมว่า UX สำคัญที่สุด
- ❌ **Optimization without monitoring**: อย่าไม่ติดตามผล

---

## 📚 แนวทางการเรียนรู้ต่อ

1. ✅ ศึกษา Browser Performance API
2. ✅ เรียนรู้ Database Query Optimization
3. ✅ ฝึกใช้ Performance Monitoring Tools
4. ✅ ศึกษา Microservices Performance
5. ✅ เรียนรู้ CDN และ Edge Computing 