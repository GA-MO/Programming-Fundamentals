# Single Responsibility Principle (SRP)

## ความหมาย
**"หนึ่งคลาสควรมีหน้าที่เดียว"** - หมายความว่าแต่ละ struct หรือ function ควรมีเหตุผลเดียวที่จะเปลี่ยนแปลง

## หลักการ
- หนึ่งคลาส = หนึ่งหน้าที่
- ถ้ามีเหตุผลหลายอย่างที่จะเปลี่ยนแปลงคลาส แสดงว่าผิดหลักการนี้
- ช่วยให้โค้ดอ่านง่าย ทดสอบง่าย และดูแลรักษาง่าย

---

## ❌ ตัวอย่างที่ไม่ดี (Bad Example)

```go
// UserService ทำหลายหน้าที่เกินไป
type UserService struct {
    db *sql.DB
}

func (s *UserService) CreateUser(user *User) error {
    // 1. ตรวจสอบข้อมูล
    if user.Name == "" || user.Email == "" {
        return errors.New("name and email are required")
    }
    
    // 2. เข้ารหัส password
    hashedPassword, err := bcrypt.GenerateFromPassword([]byte(user.Password), bcrypt.DefaultCost)
    if err != nil {
        return err
    }
    
    // 3. บันทึกลงฐานข้อมูล
    query := "INSERT INTO users (name, email, password) VALUES (?, ?, ?)"
    _, err = s.db.Exec(query, user.Name, user.Email, string(hashedPassword))
    if err != nil {
        return err
    }
    
    // 4. ส่งอีเมลยืนยัน
    s.sendEmail(user.Email, "Welcome!", fmt.Sprintf("Welcome %s!", user.Name))
    
    // 5. บันทึก log
    log.Printf("User created: %s", user.Email)
    
    return nil
}

func (s *UserService) sendEmail(to, subject, body string) error {
    // โค้ดส่งอีเมล
    return nil
}
```

### ปัญหาของโค้ดข้างต้น:
1. **หลายหน้าที่**: ตรวจสอบข้อมูล, เข้ารหัส, บันทึก DB, ส่งอีเมล, บันทึก log
2. **ยากต่อการทดสอบ**: ต้อง mock หลายอย่าง
3. **ยากต่อการแก้ไข**: ถ้าเปลี่ยนวิธีส่งอีเมลต้องแก้ไข UserService
4. **ยากต่อการนำกลับใช้**: ไม่สามารถใช้ส่วนการเข้ารหัสแยกได้

---

## ✅ ตัวอย่างที่ดี (Good Example)

```go
// แยกแต่ละหน้าที่ออกเป็น struct แยกกัน

// 1. UserValidator - ตรวจสอบข้อมูล
type UserValidator struct{}

func (v *UserValidator) Validate(user *User) error {
    if user.Name == "" || user.Email == "" {
        return errors.New("name and email are required")
    }
    return nil
}

// 2. PasswordHasher - เข้ารหัส password
type PasswordHasher struct{}

func (h *PasswordHasher) Hash(password string) (string, error) {
    return bcrypt.GenerateFromPassword([]byte(password), bcrypt.DefaultCost)
}

// 3. UserRepository - จัดการฐานข้อมูล
type UserRepository struct {
    db *sql.DB
}

func (r *UserRepository) Create(user *User) error {
    query := "INSERT INTO users (name, email, password) VALUES (?, ?, ?)"
    _, err := r.db.Exec(query, user.Name, user.Email, user.Password)
    return err
}

// 4. EmailService - ส่งอีเมล
type EmailService struct{}

func (e *EmailService) SendWelcomeEmail(user *User) error {
    return e.sendEmail(user.Email, "Welcome!", fmt.Sprintf("Welcome %s!", user.Name))
}

func (e *EmailService) sendEmail(to, subject, body string) error {
    // โค้ดส่งอีเมล
    return nil
}

// 5. Logger - บันทึก log
type Logger struct{}

func (l *Logger) LogUserCreated(email string) {
    log.Printf("User created: %s", email)
}

// 6. UserService - ประสานงานระหว่างส่วนต่างๆ
type UserService struct {
    validator  *UserValidator
    hasher     *PasswordHasher
    repository *UserRepository
    emailSvc   *EmailService
    logger     *Logger
}

func (s *UserService) CreateUser(user *User) error {
    // 1. ตรวจสอบข้อมูล
    if err := s.validator.Validate(user); err != nil {
        return err
    }
    
    // 2. เข้ารหัส password
    hashedPassword, err := s.hasher.Hash(user.Password)
    if err != nil {
        return err
    }
    user.Password = string(hashedPassword)
    
    // 3. บันทึกลงฐานข้อมูล
    if err := s.repository.Create(user); err != nil {
        return err
    }
    
    // 4. ส่งอีเมลยืนยัน
    if err := s.emailSvc.SendWelcomeEmail(user); err != nil {
        return err
    }
    
    // 5. บันทึก log
    s.logger.LogUserCreated(user.Email)
    
    return nil
}
```

### ข้อดีของโค้ดที่ดี:
1. **แต่ละ struct มีหน้าที่เดียว**: ง่ายต่อการเข้าใจและแก้ไข
2. **ทดสอบง่าย**: สามารถทดสอบแต่ละส่วนแยกกันได้
3. **นำกลับใช้ได้**: สามารถใช้ PasswordHasher ในที่อื่นได้
4. **ขยายง่าย**: ถ้าต้องการเปลี่ยนวิธีส่งอีเมล แค่แก้ไข EmailService
5. **Dependency Injection**: ง่ายต่อการ mock และทดสอบ

---

## ตัวอย่างการใช้งานจริง

```go
func main() {
    // สร้าง dependencies
    db, _ := sql.Open("mysql", "dsn")
    userService := &UserService{
        validator:  &UserValidator{},
        hasher:     &PasswordHasher{},
        repository: &UserRepository{db: db},
        emailSvc:   &EmailService{},
        logger:     &Logger{},
    }
    
    // ใช้งาน
    user := &User{
        Name:     "John Doe",
        Email:    "john@example.com",
        Password: "password123",
    }
    
    err := userService.CreateUser(user)
    if err != nil {
        log.Fatal(err)
    }
}
```

---

## การทดสอบ (Testing)

```go
func TestUserService_CreateUser(t *testing.T) {
    // Mock dependencies
    mockValidator := &MockUserValidator{}
    mockHasher := &MockPasswordHasher{}
    mockRepository := &MockUserRepository{}
    mockEmailSvc := &MockEmailService{}
    mockLogger := &MockLogger{}
    
    userService := &UserService{
        validator:  mockValidator,
        hasher:     mockHasher,
        repository: mockRepository,
        emailSvc:   mockEmailSvc,
        logger:     mockLogger,
    }
    
    // Setup expectations
    mockValidator.On("Validate", mock.Anything).Return(nil)
    mockHasher.On("Hash", "password123").Return([]byte("hashed"), nil)
    mockRepository.On("Create", mock.Anything).Return(nil)
    mockEmailSvc.On("SendWelcomeEmail", mock.Anything).Return(nil)
    mockLogger.On("LogUserCreated", "john@example.com").Return()
    
    // Test
    user := &User{
        Name:     "John Doe",
        Email:    "john@example.com",
        Password: "password123",
    }
    
    err := userService.CreateUser(user)
    
    // Assert
    assert.NoError(t, err)
    mockValidator.AssertExpectations(t)
    mockHasher.AssertExpectations(t)
    mockRepository.AssertExpectations(t)
    mockEmailSvc.AssertExpectations(t)
    mockLogger.AssertExpectations(t)
}
```

---

## สรุป

**SRP ช่วยให้:**
- โค้ดอ่านง่ายและเข้าใจง่าย
- ทดสอบได้ง่าย
- แก้ไขและขยายได้ง่าย
- ลดการพึ่งพากันระหว่างส่วนต่างๆ
- เพิ่มความสามารถในการนำกลับใช้

**เมื่อไหร่ควรใช้:**
- เมื่อคลาสหรือ function มีหน้าที่หลายอย่าง
- เมื่อยากต่อการทดสอบ
- เมื่อแก้ไขส่วนหนึ่งกระทบส่วนอื่น

---

## อ้างอิงไปยังหลักการอื่นๆ

- **[Dependency Inversion Principle](./05-dependency-inversion-principle.md)** - การใช้ dependency injection ช่วยให้ SRP ทำงานได้ดีขึ้น
- **[Interface Segregation Principle](./04-interface-segregation-principle.md)** - การแยก interface เล็กๆ ช่วยให้ SRP ชัดเจนขึ้น 