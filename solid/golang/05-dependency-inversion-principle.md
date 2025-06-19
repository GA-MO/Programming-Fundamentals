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

type EmailSender struct {
    smtpServer string
    port       int
}

func (e *EmailSender) Send(to, subject, body string) error {
    fmt.Printf("Sending email to %s: %s\n", to, subject)
    return nil
}

type SMSSender struct {
    apiKey string
    apiUrl string
}

func (s *SMSSender) Send(to, message string) error {
    fmt.Printf("Sending SMS to %s: %s\n", to, message)
    return nil
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

```go
// Notification interface - abstraction
type NotificationSender interface {
    Send(to, message string) error
}

// EmailSender implementation
type EmailSender struct {
    smtpServer string
    port       int
}

func (e *EmailSender) Send(to, message string) error {
    fmt.Printf("Sending email to %s: %s\n", to, message)
    return nil
}

// SMSSender implementation
type SMSSender struct {
    apiKey string
    apiUrl string
}

func (s *SMSSender) Send(to, message string) error {
    fmt.Printf("Sending SMS to %s: %s\n", to, message)
    return nil
}

// PushNotificationSender implementation
type PushNotificationSender struct {
    fcmKey string
}

func (p *PushNotificationSender) Send(to, message string) error {
    fmt.Printf("Sending push notification to %s: %s\n", to, message)
    return nil
}

// NotificationService ที่ขึ้นต่อ abstraction
type NotificationService struct {
    senders map[string]NotificationSender
}

func NewNotificationService() *NotificationService {
    return &NotificationService{
        senders: make(map[string]NotificationSender),
    }
}

func (n *NotificationService) AddSender(name string, sender NotificationSender) {
    n.senders[name] = sender
}

func (n *NotificationService) SendNotification(user *User, message, method string) error {
    sender, exists := n.senders[method]
    if !exists {
        return fmt.Errorf("unsupported notification method: %s", method)
    }
    
    var to string
    switch method {
    case "email":
        to = user.Email
    case "sms":
        to = user.Phone
    case "push":
        to = user.DeviceToken
    default:
        return fmt.Errorf("unknown notification method: %s", method)
    }
    
    return sender.Send(to, message)
}

// User struct
type User struct {
    Name        string
    Email       string
    Phone       string
    DeviceToken string
}
```

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
    // Setup Notification Service
    notificationService := NewNotificationService()
    
    // Add different notification senders
    emailSender := &EmailSender{
        smtpServer: "smtp.gmail.com",
        port:       587,
    }
    smsSender := &SMSSender{
        apiKey: "your-api-key",
        apiUrl: "https://api.sms.com",
    }
    pushSender := &PushNotificationSender{
        fcmKey: "your-fcm-key",
    }
    
    notificationService.AddSender("email", emailSender)
    notificationService.AddSender("sms", smsSender)
    notificationService.AddSender("push", pushSender)
    
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
    
    // ส่ง notification หลายวิธี
    notificationService.SendNotification(user, "Welcome!", "email")
    notificationService.SendNotification(user, "Welcome!", "sms")
    notificationService.SendNotification(user, "Welcome!", "push")
    
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
    emailSender := &EmailSender{smtpServer: "smtp.gmail.com", port: 587}
    smsSender := &SMSSender{apiKey: "your-api-key", apiUrl: "https://api.sms.com"}
    
    notificationService := NewNotificationService()
    notificationService.AddSender("email", emailSender)
    notificationService.AddSender("sms", smsSender)
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
    notificationService.SendNotification(user, "Welcome!", "email")
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

func TestNotificationService_SendNotification(t *testing.T) {
    notificationService := NewNotificationService()
    
    // ใช้ mock sender
    mockSender := &MockNotificationSender{}
    notificationService.AddSender("test", mockSender)
    
    user := &User{
        Name:  "John",
        Email: "john@example.com",
    }
    
    err := notificationService.SendNotification(user, "Test message", "test")
    
    assert.NoError(t, err)
    assert.Equal(t, "john@example.com", mockSender.LastTo)
    assert.Equal(t, "Test message", mockSender.LastMessage)
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

// Mock implementations สำหรับ testing
type MockNotificationSender struct {
    LastTo      string
    LastMessage string
}

func (m *MockNotificationSender) Send(to, message string) error {
    m.LastTo = to
    m.LastMessage = message
    return nil
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
```