# Code Smells (กลิ่นเหม็นของโค้ด)

Code Smells หรือ "กลิ่นเหม็นของโค้ด" คือสัญญาณเตือนที่บ่งบอกว่าโค้ดมีปัญหาด้านการออกแบบหรือโครงสร้าง แม้ว่าโค้ดจะยังทำงานได้ปกติ แต่การมี Code Smells จะทำให้โค้ดยากต่อการอ่าน เข้าใจ บำรุงรักษา และขยายความสามารถ

## สารบัญ (Table of Contents)

- [ประเภทของ Code Smells ที่พบบ่อย](#ประเภทของ-code-smells-ที่พบบ่อย)
  - [1. Long Method (เมธอดที่ยาวเกินไป)](#1-long-method-เมธอดที่ยาวเกินไป)
  - [2. Large Class (คลาสที่ใหญ่เกินไป)](#2-large-class-คลาสที่ใหญ่เกินไป)
  - [3. Duplicate Code (โค้ดที่ซ้ำกัน)](#3-duplicate-code-โค้ดที่ซ้ำกัน)
  - [4. Long Parameter List (พารามิเตอร์มากเกินไป)](#4-long-parameter-list-พารามิเตอร์มากเกินไป)
  - [5. Data Clumps (ข้อมูลที่มักอยู่ด้วยกันเสมอ)](#5-data-clumps-ข้อมูลที่มักอยู่ด้วยกันเสมอ)
  - [6. Feature Envy (ความอิจฉาฟีเจอร์)](#6-feature-envy-ความอิจฉาฟีเจอร์)
  - [7. Magic Numbers (ตัวเลขมหัศจรรย์)](#7-magic-numbers-ตัวเลขมหัศจรรย์)
- [เครื่องมือที่ช่วยตรวจจับ Code Smells](#เครื่องมือที่ช่วยตรวจจับ-code-smells)
  - [1. Static Code Analysis Tools](#1-static-code-analysis-tools)
  - [2. IDE Extensions](#2-ide-extensions)
  - [3. Metrics ที่ควรติดตาม](#3-metrics-ที่ควรติดตาม)
- [หลักการป้องกัน Code Smells](#หลักการป้องกัน-code-smells)
  - [1. ปฏิบัติตาม SOLID Principles](#1-ปฏิบัติตาม-solid-principles)
  - [2. Code Review ที่สม่ำเสมอ](#2-code-review-ที่สม่ำเสมอ)
  - [3. Refactoring อย่างสม่ำเสมอ](#3-refactoring-อย่างสม่ำเสมอ)
  - [4. การเขียน Unit Tests](#4-การเขียน-unit-tests)
- [สรุป](#สรุป)

## ประเภทของ Code Smells ที่พบบ่อย

### 1. Long Method (เมธอดที่ยาวเกินไป)

**อาการ:** เมธอดมีจำนวนบรรทัดมากเกินไป (มากกว่า 20-30 บรรทัด)

**ปัญหา:** 
- ยากต่อการเข้าใจ
- ยากต่อการดีบัก
- ยากต่อการทดสอบ
- ยากต่อการใช้งานซ้ำ

**ตัวอย่างปัญหา:**
```go
// ❌ Code Smell: Long Method
func ProcessOrder(order Order) (OrderResult, error) {
    // ตรวจสอบข้อมูลลูกค้า
    if order.Customer.Name == "" || len(order.Customer.Name) < 2 {
        return OrderResult{}, fmt.Errorf("invalid customer name")
    }
    if order.Customer.Email == "" || !strings.Contains(order.Customer.Email, "@") {
        return OrderResult{}, fmt.Errorf("invalid email")
    }
    
    // คำนวณราคา
    var total float64
    for _, item := range order.Items {
        if item.Quantity <= 0 {
            return OrderResult{}, fmt.Errorf("invalid quantity")
        }
        total += item.Price * float64(item.Quantity)
    }
    
    // คำนวณส่วนลด
    var discount float64
    if total > 1000 {
        discount = total * 0.1
    } else if total > 500 {
        discount = total * 0.05
    }
    
    // คำนวณภาษี
    tax := (total - discount) * 0.07
    finalAmount := total - discount + tax
    
    // บันทึกออเดอร์
    fmt.Printf("Processing order for %s\n", order.Customer.Name)
    fmt.Printf("Final amount: %.2f\n", finalAmount)
    
    return OrderResult{
        OrderID: generateOrderID(),
        Total:   finalAmount,
        Status:  "processed",
    }, nil
}
```

**วิธีแก้ไข: Extract Method**
```go
// ✅ แก้ไขแล้ว: แยกเป็นเมธอดย่อย
func ProcessOrder(order Order) (OrderResult, error) {
    if err := validateCustomer(order.Customer); err != nil {
        return OrderResult{}, err
    }
    
    subtotal, err := calculateSubtotal(order.Items)
    if err != nil {
        return OrderResult{}, err
    }
    
    discount := calculateDiscount(subtotal)
    tax := calculateTax(subtotal - discount)
    finalAmount := subtotal - discount + tax
    
    logOrderProcessing(order.Customer.Name, finalAmount)
    
    return OrderResult{
        OrderID: generateOrderID(),
        Total:   finalAmount,
        Status:  "processed",
    }, nil
}

// ตรวจสอบข้อมูลลูกค้า
func validateCustomer(customer Customer) error {
    if customer.Name == "" || len(customer.Name) < 2 {
        return fmt.Errorf("invalid customer name")
    }
    if customer.Email == "" || !strings.Contains(customer.Email, "@") {
        return fmt.Errorf("invalid email")
    }
    return nil
}

// คำนวณยอดรวม
func calculateSubtotal(items []Item) (float64, error) {
    var total float64
    for _, item := range items {
        if item.Quantity <= 0 {
            return 0, fmt.Errorf("invalid quantity")
        }
        total += item.Price * float64(item.Quantity)
    }
    return total, nil
}

// คำนวณส่วนลด
func calculateDiscount(total float64) float64 {
    if total > 1000 {
        return total * 0.1
    }
    if total > 500 {
        return total * 0.05
    }
    return 0
}

// คำนวณภาษี
func calculateTax(amount float64) float64 {
    return amount * 0.07
}

// บันทึกการประมวลผลออเดอร์
func logOrderProcessing(customerName string, amount float64) {
    fmt.Printf("Processing order for %s\n", customerName)
    fmt.Printf("Final amount: %.2f\n", amount)
}
```

### 2. Large Class (คลาสที่ใหญ่เกินไป)

**อาการ:** คลาสมีความรับผิดชอบมากเกินไป มีเมธอดและฟิลด์จำนวนมาก

**ปัญหา:**
- ละเมิด Single Responsibility Principle
- ยากต่อการเข้าใจและบำรุงรักษา
- เสี่ยงต่อการเกิดบั๊กเมื่อแก้ไข

**ตัวอย่างปัญหา:**
```go
// ❌ Code Smell: Large Class
type User struct {
    Name         string
    Email        string
    Password     string
    LoginAttempts int
    LastLogin    *time.Time
}

// การจัดการการล็อกอิน
func (u *User) Login(password string) bool {
    if u.Password == password {
        now := time.Now()
        u.LastLogin = &now
        u.LoginAttempts = 0
        return true
    } else {
        u.LoginAttempts++
        return false
    }
}

// การจัดการฐานข้อมูล
func (u *User) SaveToDatabase() error {
    // โค้ดสำหรับบันทึกลงฐานข้อมูล
    return nil
}

func (u *User) LoadFromDatabase(userID int) error {
    // โค้ดสำหรับโหลดจากฐานข้อมูล
    return nil
}

// การส่งอีเมล
func (u *User) SendWelcomeEmail() error {
    // โค้ดสำหรับส่งอีเมลต้อนรับ
    return nil
}

func (u *User) SendPasswordResetEmail() error {
    // โค้ดสำหรับส่งอีเมลรีเซ็ตรหัสผ่าน
    return nil
}

//  การสร้างรายงาน
func (u *User) GenerateActivityReport() (string, error) {
    // โค้ดสำหรับสร้างรายงานกิจกรรม
    return "", nil
}
```

**วิธีแก้ไข: Extract Class**
```go
// ✅ แก้ไขแล้ว: แยกความรับผิดชอบ
type User struct {
    Name         string
    Email        string
    Password     string
    LoginAttempts int
    LastLogin    *time.Time
}

// ✅ การจัดการการล็อกอิน
type UserAuthentication struct {
    user *User
}

func NewUserAuthentication(user *User) *UserAuthentication {
    return &UserAuthentication{user: user}
}

func (ua *UserAuthentication) Login(password string) bool {
    if ua.user.Password == password {
        now := time.Now()
        ua.user.LastLogin = &now
        ua.user.LoginAttempts = 0
        return true
    } else {
        ua.user.LoginAttempts++
        return false
    }
}

// ✅ การจัดการฐานข้อมูล
type UserRepository struct{}

func NewUserRepository() *UserRepository {
    return &UserRepository{}
}

func (ur *UserRepository) SaveUser(user *User) error {
    // โค้ดสำหรับบันทึกลงฐานข้อมูล
    return nil
}

func (ur *UserRepository) LoadUser(userID int) (*User, error) {
    // โค้ดสำหรับโหลดจากฐานข้อมูล
    return nil, nil
}

// ✅ การจัดการอีเมล
type UserEmailService struct{}

func NewUserEmailService() *UserEmailService {
    return &UserEmailService{}
}

func (ues *UserEmailService) SendWelcomeEmail(user *User) error {
    // โค้ดสำหรับส่งอีเมลต้อนรับ
    return nil
}

func (ues *UserEmailService) SendPasswordResetEmail(user *User) error {
    // โค้ดสำหรับส่งอีเมลรีเซ็ตรหัสผ่าน
    return nil
}

// ✅ การจัดการรายงาน
type UserReportService struct{}

func NewUserReportService() *UserReportService {
    return &UserReportService{}
}

func (urs *UserReportService) GenerateActivityReport(user *User) (string, error) {
    // โค้ดสำหรับสร้างรายงานกิจกรรม
    return "", nil
}
```

### 3. Duplicate Code (โค้ดที่ซ้ำกัน)

**อาการ:** มีโค้ดที่เหมือนกันหรือคล้ายกันปรากฏในหลายที่

**ปัญหา:**
- เมื่อต้องแก้ไข ต้องแก้หลายที่
- เสี่ยงต่อการลืมแก้ไขบางที่
- เพิ่มขนาดของโค้ดโดยไม่จำเป็น

**ตัวอย่างปัญหา:**
```go
// ❌ Code Smell: Duplicate Code
func RegisterUser(name, email string) error {
    if name == "" {
        return fmt.Errorf("name is required")
    }
    if len(name) < 2 {
        return fmt.Errorf("name must be at least 2 characters")
    }
    // สร้าง user
    return nil
}

func UpdateUserProfile(name, email string) error {
    if name == "" {
        return fmt.Errorf("name is required")
    }
    if len(name) < 2 {
        return fmt.Errorf("name must be at least 2 characters")
    }
    // อัปเดต profile
    return nil
}

func CreateAdmin(name, email string) error {
    if name == "" {
        return fmt.Errorf("name is required")
    }
    if len(name) < 2 {
        return fmt.Errorf("name must be at least 2 characters")
    }
    // สร้าง admin
    return nil
}
```

**วิธีแก้ไข: Extract Method**
```go
// ✅ แก้ไขแล้ว: แยกโค้ดที่ซ้ำออกมา
func RegisterUser(name, email string) error {
    if err := validateName(name); err != nil {
        return err
    }
    // สร้าง user
    return nil
}

func UpdateUserProfile(name, email string) error {
    if err := validateName(name); err != nil {
        return err
    }
    // อัปเดต profile
    return nil
}

func CreateAdmin(name, email string) error {
    if err := validateName(name); err != nil {
        return err
    }
    // สร้าง admin
    return nil
}

// ✅ ตรวจสอบชื่อ
func validateName(name string) error {
    if name == "" {
        return fmt.Errorf("name is required")
    }
    if len(name) < 2 {
        return fmt.Errorf("name must be at least 2 characters")
    }
    return nil
}
```

### 4. Long Parameter List (พารามิเตอร์มากเกินไป)

**อาการ:** เมธอดมีพารามิเตอร์มากเกินไป (มากกว่า 3-4 ตัว)

**ปัญหา:**
- ยากต่อการจำลำดับพารามิเตอร์
- เสี่ยงต่อการส่งพารามิเตอร์ผิดตำแหน่ง
- ยากต่อการเปลี่ยนแปลง

**ตัวอย่างปัญหา:**
```go
// ❌ Code Smell: Long Parameter List
func CreateUser(firstName, lastName, email, phone, address, city, 
               country, zipCode, role string, birthDate time.Time, 
               isActive bool) error {
    // สร้างผู้ใช้
    return nil
}

// การเรียกใช้งาน - ยากจำและเสี่ยงผิดพลาด
birthDate := time.Date(1990, 1, 1, 0, 0, 0, 0, time.UTC)
err := CreateUser("John", "Doe", "john@email.com", "123-456-7890", 
                 "123 Main St", "Bangkok", "Thailand", "10100", 
                 "admin", birthDate, true)
```

**วิธีแก้ไข: Parameter Object**
```go
// ✅ แก้ไขแล้ว: ใช้ Parameter Object
type Address struct {
    Street  string
    City    string
    Country string
    ZipCode string
}

type UserInfo struct {
    FirstName string
    LastName  string
    Email     string
    Phone     string
    BirthDate time.Time
    Address   Address
    IsActive  bool
    Role      string
}

func CreateUser(userInfo UserInfo) error {
    // สร้างผู้ใช้
    return nil
}

// การเรียกใช้งาน - ชัดเจนและไม่เสี่ยงผิดพลาด
userInfo := UserInfo{
    FirstName: "John",
    LastName:  "Doe",
    Email:     "john@email.com",
    Phone:     "123-456-7890",
    BirthDate: time.Date(1990, 1, 1, 0, 0, 0, 0, time.UTC),
    Address: Address{
        Street:  "123 Main St",
        City:    "Bangkok",
        Country: "Thailand",
        ZipCode: "10100",
    },
    IsActive: true,
    Role:     "admin",
}
err := CreateUser(userInfo)
```

### 5. Data Clumps (ข้อมูลที่มักอยู่ด้วยกันเสมอ)

**อาการ:** กลุ่มข้อมูลที่มักปรากฏด้วยกันในหลายที่

**ปัญหา:**
- การเปลี่ยนแปลงต้องแก้หลายที่
- ขาดการจัดกลุ่มที่มีความหมาย

**ตัวอย่างปัญหา:**
```go
// ❌ Code Smell: Data Clumps
func SendWelcomeEmail(firstName, lastName, email string) error {
    // ส่งอีเมลต้อนรับ
    return nil
}

func CreateUserProfile(firstName, lastName, email string) error {
    // สร้างโปรไฟล์ผู้ใช้
    return nil
}

func UpdateContactInfo(firstName, lastName, email string) error {
    // อัปเดตข้อมูลติดต่อ
    return nil
}

func GenerateReport(firstName, lastName, email string) error {
    // สร้างรายงาน
    return nil
}
```

**วิธีแก้ไข: Extract Class**
```go
// ✅ แก้ไขแล้ว: จัดกลุ่มข้อมูลที่เกี่ยวข้องกัน
type Contact struct {
    FirstName string
    LastName  string
    Email     string
}

func (c Contact) GetFullName() string {
    return c.FirstName + " " + c.LastName
}

func (c Contact) IsValid() bool {
    return c.FirstName != "" && c.LastName != "" && 
           strings.Contains(c.Email, "@")
}

// การใช้งาน
func SendWelcomeEmail(contact Contact) error {
    // ส่งอีเมลต้อนรับ
    return nil
}

func CreateUserProfile(contact Contact) error {
    // สร้างโปรไฟล์ผู้ใช้
    return nil
}

func UpdateContactInfo(contact Contact) error {
    // อัปเดตข้อมูลติดต่อ
    return nil
}

func GenerateReport(contact Contact) error {
    // สร้างรายงาน
    return nil
}
```

### 6. Feature Envy (ความอิจฉาฟีเจอร์)

**อาการ:** เมธอดใช้ข้อมูลของคลาสอื่นมากกว่าคลาสของตัวเอง

**ปัญหา:**
- ละเมิดหลัก Encapsulation
- เมธอดอยู่ผิดที่

**ตัวอย่างปัญหา:**
```go
// ❌ Code Smell: Feature Envy
type Item struct {
    Price    float64
    Quantity int
}

type Customer struct {
    Name           string
    MembershipType string
}

type Order struct {
    Customer Customer
    Items    []Item
}

func (o *Order) CalculateTotalWithDiscount() float64 {
    // เมธอดนี้ใช้ข้อมูลของ customer มากกว่า order
    var total float64
    for _, item := range o.Items {
        total += item.Price * float64(item.Quantity)
    }
    
    var discount float64
    switch o.Customer.MembershipType {
    case "GOLD":
        discount = 0.15
    case "SILVER":
        discount = 0.10
    case "BRONZE":
        discount = 0.05
    default:
        discount = 0
    }
    
    return total * (1 - discount)
}
```

**วิธีแก้ไข: Move Method**
```go
// ✅ แก้ไขแล้ว: ย้ายเมธอดไปที่เหมาะสม
type Item struct {
    Price    float64
    Quantity int
}

type Customer struct {
    Name           string
    MembershipType string
}

func (c *Customer) GetDiscountRate() float64 {
    // คำนวณอัตราส่วนลดตามประเภทสมาชิก
    discountRates := map[string]float64{
        "GOLD":   0.15,
        "SILVER": 0.10,
        "BRONZE": 0.05,
    }
    
    if rate, exists := discountRates[c.MembershipType]; exists {
        return rate
    }
    return 0
}

type Order struct {
    Customer Customer
    Items    []Item
}

func (o *Order) CalculateTotal() float64 {
    var total float64
    for _, item := range o.Items {
        total += item.Price * float64(item.Quantity)
    }
    return total
}

func (o *Order) CalculateTotalWithDiscount() float64 {
    total := o.CalculateTotal()
    discountRate := o.Customer.GetDiscountRate()
    return total * (1 - discountRate)
}
```

### 7. Magic Numbers (ตัวเลขมหัศจรรย์)

**อาการ:** ใช้ตัวเลขโดยตรงในโค้ดโดยไม่มีการอธิบาย

**ปัญหา:**
- ไม่ทราบความหมายของตัวเลข
- ยากต่อการบำรุงรักษา

**ตัวอย่างปัญหา:**
```go
// ❌ Code Smell: Magic Numbers
func calculateArea(radius float64) float64 {
    return 3.14159 * radius * radius
}

func checkAge(age int) bool {
    return age >= 18 && age <= 65
}

func calculateSalary(baseSalary float64, overtime int) float64 {
    overtimePay := float64(overtime) * 1.5 * 100
    return baseSalary + overtimePay
}
```

**วิธีแก้ไข: Extract Constants**
```go
// ✅ แก้ไขแล้ว: ใช้ค่าคงที่ที่มีชื่อ
package main

import "math"

const (
    PI = math.Pi
    MIN_WORKING_AGE = 18
    MAX_WORKING_AGE = 65
    OVERTIME_MULTIPLIER = 1.5
    HOURLY_RATE = 100.0
)

func calculateArea(radius float64) float64 {
    return PI * radius * radius
}

func isWorkingAge(age int) bool {
    return age >= MIN_WORKING_AGE && age <= MAX_WORKING_AGE
}

func calculateSalary(baseSalary float64, overtimeHours int) float64 {
    overtimePay := float64(overtimeHours) * OVERTIME_MULTIPLIER * HOURLY_RATE
    return baseSalary + overtimePay
}
```

## เครื่องมือที่ช่วยตรวจจับ Code Smells

### 1. Static Code Analysis Tools
- **SonarQube**: รองรับหลายภาษา มี rules สำหรับตรวจจับ code smells
- **ESLint**: สำหรับ JavaScript/TypeScript
- **RuboCop**: สำหรับ Ruby
- **PyLint**: สำหรับ Python
- **StyleCop**: สำหรับ C#

### 2. IDE Extensions
- **CodeClimate**: ปลั๊กอินสำหรับ VS Code
- **SonarLint**: ปลั๊กอินสำหรับ IDE ต่างๆ
- **Code Metrics**: สำหรับวัดความซับซ้อนของโค้ด

### 3. Metrics ที่ควรติดตาม
- **Cyclomatic Complexity**: วัดความซับซ้อนของโค้ด
- **Lines of Code per Method**: จำนวนบรรทัดต่อเมธอด
- **Number of Parameters**: จำนวนพารามิเตอร์
- **Class Coupling**: ระดับการเชื่อมโยงระหว่างคลาส

## หลักการป้องกัน Code Smells

### 1. ปฏิบัติตาม SOLID Principles
- Single Responsibility: คลาสควรมีเหตุผลเดียวในการเปลี่ยนแปลง
- Open/Closed: เปิดสำหรับการขยาย ปิดสำหรับการแก้ไข
- Liskov Substitution: อ็อบเจ็กต์ของ subclass ควรใช้แทน superclass ได้
- Interface Segregation: ไม่ควรบังคับให้ implement interface ที่ไม่ใช้
- Dependency Inversion: ขึ้นอยู่กับ abstraction ไม่ใช่ concrete class

### 2. Code Review ที่สม่ำเสมอ
- ตรวจสอบโค้ดก่อน merge
- มองหา code smells ในระหว่าง review
- ให้ feedback ที่สร้างสรรค์

### 3. Refactoring อย่างสม่ำเสมอ
- กำหนดเวลาสำหรับ refactoring
- แก้ไข code smells ทันทีที่พบ
- ใช้ automated tests เพื่อความมั่นใจ

### 4. การเขียน Unit Tests
- ช่วยให้มั่นใจในการ refactoring
- บังคับให้โค้ดมีความยืดหยุ่น
- เปิดเผย design problems

## สรุป

Code Smells เป็นสัญญาณเตือนที่บอกว่าโค้ดต้องการการปรับปรุง การรู้จักและแก้ไข Code Smells จะทำให้:

1. **โค้ดอ่านง่ายขึ้น**: เข้าใจได้ง่าย บำรุงรักษาง่าย
2. **ลด bugs**: โครงสร้างที่ดีช่วยลดโอกาสเกิดข้อผิดพลาด  
3. **เพิ่มประสิทธิภาพการพัฒนา**: เพิ่มฟีเจอร์ใหม่ได้เร็วขึ้น
4. **ลดต้นทุนการบำรุงรักษา**: แก้ไขและปรับปรุงได้ง่าย

จำไว้ว่า Code Smells ไม่ใช่ bugs แต่เป็นสัญญาณว่าโค้ดสามารถปรับปรุงได้ การแก้ไขอย่างสม่ำเสมอจะทำให้โค้ดมีคุณภาพดีขึ้นในระยะยาว
