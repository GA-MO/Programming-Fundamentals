# Dependency Inversion Principle (DIP)

## ความหมาย
**"ขึ้นต่อ Abstraction ไม่ใช่ Concrete"** - หมายความว่าโมดูลระดับสูงไม่ควรขึ้นต่อโมดูลระดับต่ำ ทั้งสองควรขึ้นต่อ abstraction และ abstraction ไม่ควรขึ้นต่อรายละเอียด

## หลักการ
- **High-level modules** ไม่ควรขึ้นต่อ **low-level modules**
- ทั้งสองควรขึ้นต่อ **abstractions**
- **Abstractions** ไม่ควรขึ้นต่อ **details**
- **Details** ควรขึ้นต่อ **abstractions**

---

## ❌ ตัวอย่างที่ไม่ดี (Bad Example)

### 1. ขึ้นต่อ Concrete Implementation

```go
// NotificationService ที่ขึ้นต่อ concrete implementation
type NotificationService struct {
    emailSender *EmailSender
    smsSender   *SMSSender
}

func NewNotificationService() *NotificationService {
    return &NotificationService{
        emailSender: &EmailSender{
            smtpServer: "smtp.gmail.com",
            port:       587,
        },
        smsSender: &SMSSender{
            apiKey: "your-api-key",
            apiUrl: "https://api.sms.com",
        },
    }
}

func (n *NotificationService) SendEmailNotification(user *User, message string) error {
    return n.emailSender.Send(user.Email, "Notification", message)
}

func (n *NotificationService) SendSMSNotification(user *User, message string) error {
    return n.smsSender.Send(user.Phone, message)
}

// ทำหน้าที่ส่งอีเมล
type EmailSender struct {
    smtpServer string
    port       int
}

func (e *EmailSender) Send(to, subject, body string) error {
    fmt.Printf("Sending email to %s: %s\n", to, subject)
    return nil
}

// ทำหน้าที่ส่ง SMS
type SMSSender struct {
    apiKey string
    apiUrl string
}

func (s *SMSSender) Send(to, message string) error {
    fmt.Printf("Sending SMS to %s: %s\n", to, message)
    return nil
}

// User struct
type User struct {
    Name  string
    Email string
    Phone string
}
```

### ปัญหาของโค้ดข้างต้น:
1. **ขึ้นต่อ Concrete**: NotificationService ขึ้นต่อ EmailSender และ SMSSender โดยตรง
2. **ยากต่อการทดสอบ**: ต้อง mock concrete implementation
3. **ยากต่อการเปลี่ยน**: ถ้าต้องการเปลี่ยน SMS provider ต้องแก้ไข NotificationService
4. **Tight Coupling**: โมดูลระดับสูงและต่ำพึ่งพากันอย่างแน่นแฟ้น

### 2. ขึ้นต่อ Database โดยตรง

```go
// UserService ที่ขึ้นต่อ MySQL โดยตรง
type UserService struct {
    db *sql.DB
}

func NewUserService() *UserService {
    db, err := sql.Open("mysql", "user:password@tcp(localhost:3306)/dbname")
    if err != nil {
        log.Fatal(err)
    }
    return &UserService{db: db}
}

func (s *UserService) CreateUser(user *User) error {
    query := "INSERT INTO users (name, email) VALUES (?, ?)"
    _, err := s.db.Exec(query, user.Name, user.Email)
    return err
}

func (s *UserService) GetUser(id int) (*User, error) {
    query := "SELECT id, name, email FROM users WHERE id = ?"
    row := s.db.QueryRow(query, id)
    
    user := &User{}
    err := row.Scan(&user.ID, &user.Name, &user.Email)
    if err != nil {
        return nil, err
    }
    return user, nil
}
```

---

## ✅ ตัวอย่างที่ดี (Good Example)

### 1. ใช้ Interface สำหรับ Notification

เพื่อให้เข้าใจง่ายขึ้น เราจะสร้างตัวอย่างที่เน้นหลักการ DIP โดยตรง โดยแยกส่วนของ Notification ออกมาให้ชัดเจน

**หลักการ:** `NotificationService` (high-level module) จะไม่ขึ้นกับ `EmailSender` (low-level module) โดยตรง แต่จะขึ้นกับ `MessageSender` (abstraction) ซึ่งเป็น interface

```go
// 1. สร้าง Abstraction (Interface)
// เป็น Interface ที่กำหนดว่า object ที่จะส่งข้อความต้องทำอะไรได้บ้าง
type MessageSender interface {
    Send(recipient string, message string) error
}

// 2. สร้าง High-level Module ที่ขึ้นต่อ Abstraction
// Service ของเราจะรู้จักแค่ MessageSender ไม่สนใจว่าจริงๆ แล้วเป็น Email หรือ SMS
type NotificationService struct {
    emailSender MessageSender // ขึ้นอยู่กับ Interface, ไม่ใช่ Concrete Type
    smsSender   MessageSender // ขึ้นอยู่กับ Interface, ไม่ใช่ Concrete Type
}

// ใช้ Dependency Injection เพื่อส่ง dependency (sender) เข้ามาตอนสร้าง object
func NewNotificationService(emailSender MessageSender, smsSender MessageSender) *NotificationService {
    return &NotificationService{emailSender: emailSender, smsSender: smsSender}
}

func (s *NotificationService) SendEmailNotification(recipient string, message string) error {
    // เรียกใช้ method ผ่าน interface
    return s.emailSender.Send(recipient, message)
}

func (s *NotificationService) SendSMSNotification(recipient string, message string) error {
    // เรียกใช้ method ผ่าน interface
    return s.smsSender.Send(recipient, message)
    }

// 3. สร้าง Low-level Modules ที่ทำหน้าที่ส่งข้อความจริง และขึ้นอยู่กับ Abstraction
// EmailSender ทำตาม Interface ของ MessageSender
type EmailSender struct{}

func (s *EmailSender) Send(recipient string, message string) error {
    fmt.Printf("Sending email to %s: %s\n", recipient, message)
    return nil
}

// SMSSender ก็ทำตาม Interface ของ MessageSender
type SMSSender struct{}

func (s *SMSSender) Send(recipient string, message string) error {
    fmt.Printf("Sending SMS to %s: %s\n", recipient, message)
    return nil
}

// MockMessageSender implementation (สำหรับ testing)
type MockMessageSender struct {}
func (m *MockMessageSender) Send(recipient string, message string) error {
    fmt.Printf("Sending mock message to %s: %s\n", recipient, message)
    return nil
}

// --- ตัวอย่างการใช้งาน ---
func main_notification() {
    // สร้าง Service โดยส่ง EmailSender เข้าไป และ SMSSender เข้าไป
    emailSender := &EmailSender{}
    smsSender := &SMSSender{}
    emailService := NewNotificationService(emailSender, smsSender)
    emailService.SendEmailNotification("test@example.com", "Hello from Email!")
    emailService.SendSMSNotification("0812345678", "Hello from SMS!")
}
```

**ข้อดีของโค้ดนี้:**
- **Decoupled**: `NotificationService` ไม่ผูกติดกับ `EmailSender` หรือ `SMSSender`
- **Flexible**: เราสามารถเปลี่ยนไปใช้ `SMSSender` หรือ sender อื่นๆ ที่ implement `MessageSender` ได้ง่ายๆ โดยไม่ต้องแก้โค้ด `NotificationService` เลย
- **Testable**: สามารถสร้าง `MockMessageSender` เพื่อทดสอบ `NotificationService` ได้โดยง่าย (ดูในหัวข้อ "การทดสอบ")

### 2. ใช้ Interface สำหรับ Database

```go
// UserRepository interface - abstraction
type UserRepository interface {
    Create(user *User) error
    GetByID(id int) (*User, error)
    Update(user *User) error
    Delete(id int) error
}

// MySQLUserRepository implementation
type MySQLUserRepository struct {
    db *sql.DB
}

func NewMySQLUserRepository(dsn string) (*MySQLUserRepository, error) {
    db, err := sql.Open("mysql", dsn)
    if err != nil {
        return nil, err
    }
    return &MySQLUserRepository{db: db}, nil
}

func (r *MySQLUserRepository) Create(user *User) error {
    query := "INSERT INTO users (name, email) VALUES (?, ?)"
    result, err := r.db.Exec(query, user.Name, user.Email)
    if err != nil {
        return err
    }
    
    id, err := result.LastInsertId()
    if err != nil {
        return err
    }
    user.ID = int(id)
    return nil
}

func (r *MySQLUserRepository) GetByID(id int) (*User, error) {
    query := "SELECT id, name, email FROM users WHERE id = ?"
    row := r.db.QueryRow(query, id)
    
    user := &User{}
    err := row.Scan(&user.ID, &user.Name, &user.Email)
    if err != nil {
        return nil, err
    }
    return user, nil
}

func (r *MySQLUserRepository) Update(user *User) error {
    query := "UPDATE users SET name = ?, email = ? WHERE id = ?"
    _, err := r.db.Exec(query, user.Name, user.Email, user.ID)
    return err
}

func (r *MySQLUserRepository) Delete(id int) error {
    query := "DELETE FROM users WHERE id = ?"
    _, err := r.db.Exec(query, id)
    return err
}

// InMemoryUserRepository implementation (สำหรับ testing)
type InMemoryUserRepository struct {
    users map[int]*User
    nextID int
}

func NewInMemoryUserRepository() *InMemoryUserRepository {
    return &InMemoryUserRepository{
        users: make(map[int]*User),
        nextID: 1,
    }
}

func (r *InMemoryUserRepository) Create(user *User) error {
    user.ID = r.nextID
    r.users[user.ID] = user
    r.nextID++
    return nil
}

func (r *InMemoryUserRepository) GetByID(id int) (*User, error) {
    user, exists := r.users[id]
    if !exists {
        return nil, fmt.Errorf("user not found: %d", id)
    }
    return user, nil
}

func (r *InMemoryUserRepository) Update(user *User) error {
    if _, exists := r.users[user.ID]; !exists {
        return fmt.Errorf("user not found: %d", user.ID)
    }
    r.users[user.ID] = user
    return nil
}

func (r *InMemoryUserRepository) Delete(id int) error {
    if _, exists := r.users[id]; !exists {
        return fmt.Errorf("user not found: %d", id)
    }
    delete(r.users, id)
    return nil
}

// UserService ที่ขึ้นต่อ abstraction
type UserService struct {
    repository UserRepository
}

func NewUserService(repository UserRepository) *UserService {
    return &UserService{repository: repository}
}

func (s *UserService) CreateUser(user *User) error {
    return s.repository.Create(user)
}

func (s *UserService) GetUser(id int) (*User, error) {
    return s.repository.GetByID(id)
}

func (s *UserService) UpdateUser(user *User) error {
    return s.repository.Update(user)
}

func (s *UserService) DeleteUser(id int) error {
    return s.repository.Delete(id)
}
```

### 3. ใช้ Interface สำหรับ Logger

```go
// Logger interface - abstraction
type Logger interface {
    Log(message string) error
    Close() error
}

// FileLogger implementation
type FileLogger struct {
    filePath string
    file     *os.File
}

func NewFileLogger(filePath string) (*FileLogger, error) {
    file, err := os.OpenFile(filePath, os.O_APPEND|os.O_CREATE|os.O_WRONLY, 0644)
    if err != nil {
        return nil, err
    }
    return &FileLogger{
        filePath: filePath,
        file:     file,
    }, nil
}

func (l *FileLogger) Log(message string) error {
    timestamp := time.Now().Format("2006-01-02 15:04:05")
    logEntry := fmt.Sprintf("[%s] %s\n", timestamp, message)
    _, err := l.file.WriteString(logEntry)
    return err
}

func (l *FileLogger) Close() error {
    return l.file.Close()
}

// ConsoleLogger implementation
type ConsoleLogger struct{}

func NewConsoleLogger() *ConsoleLogger {
    return &ConsoleLogger{}
}

func (l *ConsoleLogger) Log(message string) error {
    timestamp := time.Now().Format("2006-01-02 15:04:05")
    fmt.Printf("[%s] %s\n", timestamp, message)
    return nil
}

func (l *ConsoleLogger) Close() error {
    return nil
}

// MockLogger implementation (สำหรับ testing)
type MockLogger struct {
    messages []string
}

func NewMockLogger() *MockLogger {
    return &MockLogger{
        messages: make([]string, 0),
    }
}

func (l *MockLogger) Log(message string) error {
    l.messages = append(l.messages, message)
    return nil
}

func (l *MockLogger) Close() error {
    return nil
}

func (l *MockLogger) GetMessages() []string {
    return l.messages
}

// Application ที่ขึ้นต่อ abstraction
type Application struct {
    logger Logger
}

func NewApplication(logger Logger) *Application {
    return &Application{logger: logger}
}

func (a *Application) DoSomething() {
    a.logger.Log("Application started")
    a.logger.Log("Application finished")
}
```

---

## ตัวอย่างการใช้งานจริง

```go
func main() {
    // Setup User Service with MySQL
    mysqlRepo, err := NewMySQLUserRepository("user:password@tcp(localhost:3306)/dbname")
    if err != nil {
        log.Fatal(err)
    }
    userService := NewUserService(mysqlRepo)
    
    // Setup Application with File Logger
    fileLogger, err := NewFileLogger("app.log")
    if err != nil {
        log.Fatal(err)
    }
    defer fileLogger.Close()
    
    app := NewApplication(fileLogger)
    
    // ใช้งาน
    user := &User{
        Name:        "John Doe",
        Email:       "john@example.com",
        Phone:       "+1234567890",
        DeviceToken: "device-token-123",
    }
    
    // สร้าง user
    err = userService.CreateUser(user)
    if err != nil {
        log.Printf("Error creating user: %v", err)
    }
    
    // ใช้งาน application
    app.DoSomething()
}
```

---

## การใช้ Dependency Injection Container

```go
// Simple DI Container
type Container struct {
    services map[string]interface{}
}

func NewContainer() *Container {
    return &Container{
        services: make(map[string]interface{}),
    }
}

func (c *Container) Register(name string, service interface{}) {
    c.services[name] = service
}

func (c *Container) Get(name string) interface{} {
    return c.services[name]
}

// Setup function
func SetupContainer() *Container {
    container := NewContainer()
    
    // Register repositories
    mysqlRepo, _ := NewMySQLUserRepository("user:password@tcp(localhost:3306)/dbname")
    container.Register("userRepository", mysqlRepo)
    
    // Register services
    userService := NewUserService(mysqlRepo)
    container.Register("userService", userService)
    
    // Register notification senders
    emailSender := &EmailSender{}
    smsSender := &SMSSender{}
    
    notificationService := NewNotificationService(emailSender)
    notificationService.sender = smsSender
    container.Register("notificationService", notificationService)
    
    // Register logger
    fileLogger, _ := NewFileLogger("app.log")
    container.Register("logger", fileLogger)
    
    return container
}

// ใช้งาน
func main() {
    container := SetupContainer()
    
    userService := container.Get("userService").(*UserService)
    notificationService := container.Get("notificationService").(*NotificationService)
    logger := container.Get("logger").(Logger)
    
    // ใช้งาน services
    user := &User{Name: "John", Email: "john@example.com"}
    userService.CreateUser(user)
    notificationService.SendNotification("john@example.com", "Welcome!")
    logger.Log("User created successfully")
}
```

---

## การทดสอบ (Testing)

```go
func TestUserService_CreateUser(t *testing.T) {
    // ใช้ mock repository
    mockRepo := NewInMemoryUserRepository()
    userService := NewUserService(mockRepo)
    
    user := &User{
        Name:  "John Doe",
        Email: "john@example.com",
    }
    
    err := userService.CreateUser(user)
    
    assert.NoError(t, err)
    assert.Equal(t, 1, user.ID)
    
    // ตรวจสอบว่าข้อมูลถูกบันทึกจริง
    savedUser, err := userService.GetUser(1)
    assert.NoError(t, err)
    assert.Equal(t, "John Doe", savedUser.Name)
    assert.Equal(t, "john@example.com", savedUser.Email)
}

// Mock implementations สำหรับ testing
type MockMessageSender struct {
    LastRecipient string
    LastMessage   string
}

func (m *MockMessageSender) Send(recipient, message string) error {
    m.LastRecipient = recipient
    m.LastMessage = message
    return nil
}

func TestNotificationService_SendNotification(t *testing.T) {
    // 1. Setup: สร้าง Mock Sender และฉีดเข้าไปใน Service
    mockEmailSender := &MockMessageSender{}
    mockSMSSender := &MockMessageSender{}
    notificationService := NewNotificationService(mockEmailSender, mockSMSSender)

    // 2. Act: เรียกใช้งาน Service
    recipient := "test@example.com"
    message := "Hello, this is a test"
    err := notificationService.SendEmailNotification(recipient, message)

    // 3. Assert: ตรวจสอบว่า Mock Sender ถูกเรียกด้วยข้อมูลที่ถูกต้อง
    assert.NoError(t, err)
    assert.Equal(t, recipient, mockEmailSender.LastRecipient)
    assert.Equal(t, message, mockEmailSender.LastMessage)
}

func TestApplication_DoSomething(t *testing.T) {
    // ใช้ mock logger
    mockLogger := NewMockLogger()
    app := NewApplication(mockLogger)
    
    app.DoSomething()
    
    messages := mockLogger.GetMessages()
    assert.Len(t, messages, 2)
    assert.Contains(t, messages[0], "Application started")
    assert.Contains(t, messages[1], "Application finished")
}

```

---

## สรุป

**DIP ช่วยให้:**
- โค้ดมีความยืดหยุ่นและขยายได้
- ง่ายต่อการทดสอบ
- ลดการพึ่งพากันระหว่างโมดูล
- ง่ายต่อการเปลี่ยน implementation
- เพิ่มความสามารถในการนำกลับใช้

**เมื่อไหร่ควรใช้:**
- เมื่อต้องการลดการพึ่งพากันระหว่างโมดูล
- เมื่อต้องการทดสอบโค้ดได้ง่าย
- เมื่อต้องการเปลี่ยน implementation ได้ง่าย
- เมื่อต้องการให้โค้ดมีความยืดหยุ่น

**หลักการสำคัญ:**
- **Dependency Injection**: ส่ง dependency เข้าไปใน constructor
- **Interface-based Design**: ใช้ interface แทน concrete implementation
- **Inversion of Control**: ให้ framework หรือ container จัดการ dependency

---

## อ้างอิงไปยังหลักการอื่นๆ

- **[Single Responsibility Principle](./01-single-responsibility-principle.md)** - การแยกหน้าที่ช่วยให้ DIP ทำงานได้ดีขึ้น
- **[Open/Closed Principle](./02-open-closed-principle.md)** - การใช้ abstraction ช่วยให้ OCP ทำงานได้ดีขึ้น
- **[Interface Segregation Principle](./04-interface-segregation-principle.md)** - การใช้ interface เล็กๆ ช่วยให้ DIP ทำงานได้ดีขึ้น