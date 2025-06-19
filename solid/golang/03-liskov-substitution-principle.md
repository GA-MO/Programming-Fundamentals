# Liskov Substitution Principle (LSP)

## ความหมาย
**"Subtype ต้องสามารถแทนที่ Base type ได้โดยไม่ทำให้โปรแกรมผิดพลาด"** - หมายความว่า subclass ต้องสามารถใช้แทน parent class ได้โดยไม่ทำให้โค้ดเสียหาย

## หลักการ
- **Behavioral Subtyping**: Subtype ต้องมีพฤติกรรมที่สอดคล้องกับ base type
- **Contract**: Subtype ต้องรักษา contract ของ base type
- **No Surprises**: การใช้ subtype ไม่ควรทำให้เกิดพฤติกรรมที่ไม่คาดคิด

---

## ❌ ตัวอย่างที่ไม่ดี (Bad Example)

```go
// Logger interface ที่มีปัญหา
type Logger interface {
    Log(message string) error
    GetLogCount() int
}

// FileLogger - บันทึกลงไฟล์
type FileLogger struct {
    logCount int
}

func (f *FileLogger) Log(message string) error {
    fmt.Printf("Writing to file: %s\n", message)
    f.logCount++
    return nil
}

func (f *FileLogger) GetLogCount() int {
    return f.logCount
}

// DatabaseLogger - บันทึกลงฐานข้อมูล
type DatabaseLogger struct{}

func (d *DatabaseLogger) Log(message string) error {
    fmt.Printf("Writing to database: %s\n", message)
    return nil
}

func (d *DatabaseLogger) GetLogCount() int {
    // ละเมิด LSP! ไม่สามารถนับจำนวน log ได้
    return -1 // หรือ panic!
}

// LogService ที่ใช้งาน
func ProcessLogs(logger Logger, messages []string) {
    for _, msg := range messages {
        logger.Log(msg)
    }
    
    count := logger.GetLogCount()
    fmt.Printf("Total logs written: %d\n", count)
    // ถ้าเป็น DatabaseLogger จะได้ -1 ซึ่งไม่ถูกต้อง
}
```

### ปัญหา:
- **DatabaseLogger ไม่สามารถนับ log ได้** แต่ต้อง implement `GetLogCount()`
- **ผิด contract** ของ Logger interface
- **ทำให้โปรแกรมผิดพลาด** เมื่อใช้ DatabaseLogger

---

## ✅ ตัวอย่างที่ดี (Good Example)

```go
// แยก interface ตามความสามารถ
type Logger interface {
    Log(message string) error
}

type LogCounter interface {
    GetLogCount() int
}

// FileLogger - สามารถบันทึกและนับได้
type FileLogger struct {
    logCount int
}

func (f *FileLogger) Log(message string) error {
    fmt.Printf("Writing to file: %s\n", message)
    f.logCount++
    return nil
}

func (f *FileLogger) GetLogCount() int {
    return f.logCount
}

// DatabaseLogger - สามารถบันทึกได้อย่างเดียว
type DatabaseLogger struct{}

func (d *DatabaseLogger) Log(message string) error {
    fmt.Printf("Writing to database: %s\n", message)
    return nil
}

// EmailSender interface
type EmailSender interface {
    Send(to, subject, message string) error
}

// SMTPSender
type SMTPSender struct{}

func (s *SMTPSender) Send(to, subject, message string) error {
    fmt.Printf("Sending email to %s: %s\n", to, subject)
    return nil
}

// MockSender สำหรับทดสอบ
type MockSender struct {
    emailsSent []string
}

func (m *MockSender) Send(to, subject, message string) error {
    m.emailsSent = append(m.emailsSent, to)
    fmt.Printf("Mock: Email sent to %s\n", to)
    return nil // ไม่ทำอะไรจริง แต่ไม่ error
}

// ฟังก์ชันที่ใช้งาน
func WriteLog(logger Logger, message string) {
    err := logger.Log(message)
    if err != nil {
        fmt.Printf("Error logging: %v\n", err)
    }
}

func CountLogs(counter LogCounter) {
    count := counter.GetLogCount()
    fmt.Printf("Total logs: %d\n", count)
}

func SendNotification(sender EmailSender, email, subject, message string) {
    err := sender.Send(email, subject, message)
    if err != nil {
        fmt.Printf("Error sending email: %v\n", err)
    }
}
```

### ข้อดี:
- **รักษา contract** แต่ละ interface ทำหน้าที่เฉพาะ
- **ไม่มีปัญหา** เมื่อแทนที่กัน
- **เข้าใจง่าย** แยกความรับผิดชอบชัดเจน

---

## ตัวอย่างการใช้งาน

```go
func main() {
    fileLogger := &FileLogger{}
    dbLogger := &DatabaseLogger{}
    smtpSender := &SMTPSender{}
    mockSender := &MockSender{}

    // ทุก logger สามารถบันทึกได้
    WriteLog(fileLogger, "File log message")
    WriteLog(dbLogger, "Database log message")

    // เฉพาะ FileLogger เท่านั้นที่นับได้
    CountLogs(fileLogger)
    // CountLogs(dbLogger) // Compile error - ดีแล้ว!

    // ทุก sender สามารถส่งอีเมลได้
    SendNotification(smtpSender, "user@example.com", "Welcome", "Hello!")
    SendNotification(mockSender, "test@example.com", "Test", "Test message")
}
```

---

## การทดสอบ

```go
func TestLoggers(t *testing.T) {
    fileLogger := &FileLogger{}
    dbLogger := &DatabaseLogger{}

    // ทั้งสองตัวควรบันทึกได้
    err1 := fileLogger.Log("test message")
    err2 := dbLogger.Log("test message")

    assert.NoError(t, err1)
    assert.NoError(t, err2)

    // เฉพาะ FileLogger เท่านั้นที่นับได้
    assert.Equal(t, 1, fileLogger.GetLogCount())
}

func TestEmailSenders(t *testing.T) {
    smtpSender := &SMTPSender{}
    mockSender := &MockSender{}

    // ทั้งสองตัวควรส่งได้โดยไม่ error
    err1 := smtpSender.Send("test@example.com", "Test", "Message")
    err2 := mockSender.Send("test@example.com", "Test", "Message")

    assert.NoError(t, err1)
    assert.NoError(t, err2)
}
```

---

## ตัวอย่างเพิ่มเติม: การบันทึกข้อมูล

```go
// DataSaver interface
type DataSaver interface {
    Save(data string) error
}

// FileSaver
type FileSaver struct{}

func (f *FileSaver) Save(data string) error {
    fmt.Printf("Saving to file: %s\n", data)
    return nil
}

// DatabaseSaver
type DatabaseSaver struct{}

func (d *DatabaseSaver) Save(data string) error {
    fmt.Printf("Saving to database: %s\n", data)
    return nil
}

// MockSaver
type MockSaver struct{}

func (m *MockSaver) Save(data string) error {
    fmt.Printf("Mock save: %s\n", data)
    return nil // สำหรับทดสอบ
}

func SaveUserData(saver DataSaver, userData string) {
    err := saver.Save(userData)
    if err != nil {
        fmt.Printf("Error saving: %v\n", err)
    }
}
```

---

## สรุป

**LSP ช่วยให้:**
- โค้ดทำงานได้ถูกต้องเมื่อแทนที่ type
- ลดข้อผิดพลาดในการใช้งาน
- ง่ายต่อการทดสอบ
- เพิ่มความยืดหยุ่นของโค้ด

**หลักการสำคัญ:**
- แยก interface ตามความสามารถ
- รักษา contract ของ interface
- ไม่ทำให้เกิดข้อผิดพลาดที่ไม่คาดคิด

---

## อ้างอิงไปยังหลักการอื่นๆ

- **[Interface Segregation Principle](./04-interface-segregation-principle.md)** - การแยก interface เล็กๆ ช่วยให้ LSP ทำงานได้ดีขึ้น
- **[Open/Closed Principle](./02-open-closed-principle.md)** - การใช้ polymorphism ต้องทำตาม LSP
- **[Dependency Inversion Principle](./05-dependency-inversion-principle.md)** - การใช้ interface ต้องรักษา contract