# Clean Code - หลักการเขียนโค้ดที่สะอาด

Clean Code คือ โค้ดที่เขียนได้อย่างชัดเจน เข้าใจง่าย บำรุงรักษาได้ และดัดแปลงได้โดยไม่ซับซ้อน

## 📋 Table of Contents
- [หลักการสำคัญ](#หลักการสำคัญ)
  - [1. การตั้งชื่อที่มีความหมาย](#1-การตั้งชื่อที่มีความหมาย-meaningful-names)
  - [2. ฟังก์ชันที่เล็กและทำหน้าที่เดียว](#2-ฟังก์ชันที่เล็กและทำหน้าที่เดียว-small-functions-single-responsibility)
  - [3. หลีกเลี่ยงการซ้ำซ้อน](#3-หลีกเลี่ยงการซ้ำซ้อน-dry---dont-repeat-yourself)
  - [4. คอมเมนต์ที่เหมาะสม](#4-คอมเมนต์ที่เหมาะสม-good-comments)
  - [5. การจัดรูปแบบที่สม่ำเสมอ](#5-การจัดรูปแบบที่สม่ำเสมอ-consistent-formatting)
  - [6. การจัดการ Error ที่เหมาะสม](#6-การจัดการ-error-ที่เหมาะสม-proper-error-handling)
  - [7. หลักการ KISS - Keep It Simple](#7-หลักการ-kiss---keep-it-simple)
  - [8. การจำกัดจำนวน Parameters](#8-การจำกัดจำนวน-parameters)
  - [9. Return Early Pattern](#9-return-early-pattern)
  - [10. หลีกเลี่ยง Magic Numbers](#10-หลีกเลี่ยง-magic-numbers)
- [หลักการเพิ่มเติม](#หลักการเพิ่มเติม)
  - [SOLID Principles](#solid-principles)
  - [Clean Code Rules](#clean-code-rules)
- [ประโยชน์ของ Clean Code](#ประโยชน์ของ-clean-code)
- [สรุป](#สรุป)

---

## หลักการสำคัญ

### 1. การตั้งชื่อที่มีความหมาย (Meaningful Names)

#### ❌ ไม่ดี:
```go
var d time.Time // วันที่
var u []User
for _, x := range users {
    if x.a > 18 {
        u = append(u, x)
    }
}

func calc(x, y float64) float64 {
    return x * y * 0.1
}
```

#### ✅ ดี:
```go
var currentDate time.Time
var adultUsers []User
for _, user := range users {
    if user.Age > 18 {
        adultUsers = append(adultUsers, user)
    }
}

func calculateDiscount(price, quantity float64) float64 {
    const discountRate = 0.1
    return price * quantity * discountRate
}
```

### 2. ฟังก์ชันที่เล็กและทำหน้าที่เดียว (Small Functions, Single Responsibility)

#### ❌ ไม่ดี:
```go
func generateReport(orders []Order) string {
    // คำนวณยอดรวม
    total := 0.0
    for _, order := range orders {
        total += order.Amount
    }
    
    // หาค่าเฉลี่ย
    average := total / float64(len(orders))
    
    // หาออร์เดอร์ที่มากที่สุด
    maxOrder := orders[0]
    for _, order := range orders {
        if order.Amount > maxOrder.Amount {
            maxOrder = order
        }
    }
    
    // สร้างรายงาน
    report := fmt.Sprintf("Total: %.2f\n", total)
    report += fmt.Sprintf("Average: %.2f\n", average)
    report += fmt.Sprintf("Max Order: %.2f\n", maxOrder.Amount)
    
    return report
}
```

#### ✅ ดี:
```go
func calculateTotal(orders []Order) float64 {
    total := 0.0
    for _, order := range orders {
        total += order.Amount
    }
    return total
}

func calculateAverage(orders []Order) float64 {
    total := calculateTotal(orders)
    return total / float64(len(orders))
}

func findMaxOrder(orders []Order) Order {
    maxOrder := orders[0]
    for _, order := range orders {
        if order.Amount > maxOrder.Amount {
            maxOrder = order
        }
    }
    return maxOrder
}

func generateReport(orders []Order) string {
    total := calculateTotal(orders)
    average := calculateAverage(orders)
    maxOrder := findMaxOrder(orders)
    
    report := fmt.Sprintf("Total: %.2f\n", total)
    report += fmt.Sprintf("Average: %.2f\n", average)
    report += fmt.Sprintf("Max Order: %.2f\n", maxOrder.Amount)
    
    return report
}
```

### 3. หลีกเลี่ยงการซ้ำซ้อน (DRY - Don't Repeat Yourself)

#### ❌ ไม่ดี:
```go
type OrderService struct {
    orderRepo    OrderRepository
    emailService EmailService
}

func (s *OrderService) ProcessOnlineOrder(order *Order) error {
    s.validateOrder(order)
    s.calculateTax(order)
    s.calculateShipping(order)
    // บันทึกลง database
    s.orderRepo.Save(order)
    // ส่งอีเมลยืนยัน
    return s.emailService.SendConfirmation(order.CustomerEmail, order.ID)
}

func (s *OrderService) ProcessInStoreOrder(order *Order) error {
    s.validateOrder(order)
    s.calculateTax(order)
    // บันทึกลง database
    s.orderRepo.Save(order)
    // ส่งอีเมลยืนยัน
    return s.emailService.SendConfirmation(order.CustomerEmail, order.ID)
}
```

#### ✅ ดี:
```go
type OrderService struct {
    orderRepo    OrderRepository
    emailService EmailService
}

func (s *OrderService) ProcessOnlineOrder(order *Order) error {
    s.validateOrder(order)
    s.calculateTax(order)
    s.calculateShipping(order)
    return s.finalizeOrder(order)
}

func (s *OrderService) ProcessInStoreOrder(order *Order) error {
    s.validateOrder(order)
    s.calculateTax(order)
    return s.finalizeOrder(order)
}

func (s *OrderService) finalizeOrder(order *Order) error {
    s.orderRepo.Save(order)
    return s.emailService.SendConfirmation(order.CustomerEmail, order.ID)
}
```

### 4. คอมเมนต์ที่เหมาะสม (Good Comments)

#### ❌ ไม่ดี:
```go
// เพิ่ม 1 ให้กับ count
count++

// ลูปผ่าน products
for _, product := range products {
    // ถ้า product มีในคลัง
    if product.Stock > 0 {
        // เพิ่มเข้า list
        availableProducts = append(availableProducts, product)
    }
}
```

#### ✅ ดี:
```go
// คำนวณส่วนลดตาม business rule: ซื้อครบ 1000 บาทได้ส่วนลด 10%
func calculateDiscount(totalAmount float64) float64 {
    // TODO: รอ product owner confirm เรื่องส่วนลดสำหรับ VIP customer
    if totalAmount >= 1000 {
        return totalAmount * 0.1
    }
    return 0
}

// เฉพาะสินค้าที่มีสต็อกเหลือและไม่หมดอายุถึงจะแสดงให้ลูกค้า
for _, product := range products {
    if product.Stock > 0 && !product.IsExpired() {
        displayProducts = append(displayProducts, product)
    }
}
```

### 5. การจัดรูปแบบที่สม่ำเสมอ (Consistent Formatting)

#### ❌ ไม่ดี:
```go
type UserService struct{
users []User
db Database}

func NewUserService(db Database)*UserService{return &UserService{db:db}}

func(s*UserService)GetUser(id int)(*User,error){
if id==0{return nil,nil}
user,err:=s.db.FindByID(id)
if err!=nil{return nil,err}
if user!=nil&&user.IsActive{
return user,nil
}
return nil,nil
}
```

#### ✅ ดี:
```go
type UserService struct {
    users []User
    db    Database
}

func NewUserService(db Database) *UserService {
    return &UserService{
        db: db,
    }
}

func (s *UserService) GetUser(id int) (*User, error) {
    if id == 0 {
        return nil, nil
    }

    user, err := s.db.FindByID(id)
    if err != nil {
        return nil, err
    }
    
    if user != nil && user.IsActive {
        return user, nil
    }
    
    return nil, nil
}
```

### 6. การจัดการ Error ที่เหมาะสม (Proper Error Handling)

#### ❌ ไม่ดี:
```go
func saveUserProfile(user *User) bool {
    defer func() {
        if r := recover(); r != nil {
            // ignore error
        }
    }()
    
    file, _ := os.Create("user_" + user.ID + ".json")
    data, _ := json.Marshal(user)
    file.Write(data)
    file.Close()
    
    return true // เสมอ return true แม้เกิด error
}

// ใช้งาน
if saveUserProfile(user) {
    fmt.Println("บันทึกสำเร็จ")
} else {
    fmt.Println("บันทึกไม่สำเร็จ") // ไม่เคยทำงาน
}
```

#### ✅ ดี:
```go
import (
    "encoding/json"
    "errors"
    "os"
)

var ErrUserDataInvalid = errors.New("user data is invalid")

func saveUserProfile(user *User) error {
    if user == nil || user.ID == "" {
        return ErrUserDataInvalid
    }
    
    file, err := os.Create("user_" + user.ID + ".json")
    if err != nil {
        return fmt.Errorf("cannot create file: %w", err)
    }
    defer file.Close()
    
    data, err := json.Marshal(user)
    if err != nil {
        return fmt.Errorf("cannot marshal user data: %w", err)
    }
    
    _, err = file.Write(data)
    if err != nil {
        return fmt.Errorf("cannot write to file: %w", err)
    }
    
    return nil
}

// ใช้งาน
if err := saveUserProfile(user); err != nil {
    fmt.Printf("บันทึกไม่สำเร็จ: %v\n", err)
} else {
    fmt.Println("บันทึกสำเร็จ")
}
```

### 7. หลักการ KISS - Keep It Simple

#### ❌ ไม่ดี:
```go
func calculateShippingCost(weight float64, distance int, isExpress bool, 
                         isMember bool, orderValue float64) float64 {
    baseCost := 0.0
    
    if weight <= 1 {
        baseCost = 50
    } else if weight <= 5 {
        baseCost = 100
    } else if weight <= 10 {
        baseCost = 150
    } else {
        baseCost = 200
    }
    
    distanceCost := float64(distance) * 2.5
    if distance > 100 {
        distanceCost = float64(distance) * 3.0
    }
    
    total := baseCost + distanceCost
    
    if isExpress {
        total = total * 1.5
    }
    
    if isMember && orderValue > 1000 {
        total = total * 0.8
    } else if isMember {
        total = total * 0.9
    } else if orderValue > 2000 {
        total = total * 0.95
    }
    
    return total
}
```

#### ✅ ดี:
```go
func calculateBaseCost(weight float64) float64 {
    if weight <= 1 { return 50 }
    if weight <= 5 { return 100 }
    if weight <= 10 { return 150 }
    return 200
}

func calculateDistanceCost(distance int) float64 {
    rate := 2.5
    if distance > 100 {
        rate = 3.0
    }
    return float64(distance) * rate
}

func applyDiscounts(cost float64, isMember bool, orderValue float64) float64 {
    if isMember && orderValue > 1000 { return cost * 0.8 }
    if isMember { return cost * 0.9 }
    if orderValue > 2000 { return cost * 0.95 }
    return cost
}

func calculateShippingCost(weight float64, distance int, isExpress bool, 
                         isMember bool, orderValue float64) float64 {
    baseCost := calculateBaseCost(weight)
    distanceCost := calculateDistanceCost(distance)
    total := baseCost + distanceCost
    
    if isExpress {
        total = total * 1.5
    }
    
    return applyDiscounts(total, isMember, orderValue)
}
```

### 8. การจำกัดจำนวน Parameters

#### ❌ ไม่ดี:
```go
func createUser(firstName, lastName, email, phone, address, city, 
               country, zipCode string, age int, isActive, isPremium bool) *User {
    return &User{
        FirstName: firstName,
        LastName:  lastName,
        Email:     email,
        Phone:     phone,
        Address:   address,
        City:      city,
        Country:   country,
        ZipCode:   zipCode,
        Age:       age,
        IsActive:  isActive,
        IsPremium: isPremium,
    }
}
```

#### ✅ ดี:
```go
type UserDetails struct {
    FirstName string
    LastName  string
    Email     string
    Phone     string
}

type Address struct {
    Street  string
    City    string
    Country string
    ZipCode string
}

type UserConfig struct {
    Age       int
    IsActive  bool
    IsPremium bool
}

func createUser(details UserDetails, address Address, config UserConfig) *User {
    return &User{
        UserDetails: details,
        Address:     address,
        UserConfig:  config,
    }
}
```

### 9. Return Early Pattern

#### ❌ ไม่ดี:
```go
func validateAndProcessUser(user *User) error {
    var err error
    
    if user != nil {
        if user.Email != "" {
            if strings.Contains(user.Email, "@") {
                if user.Age >= 18 {
                    if user.IsActive {
                        // ประมวลผลหลักอยู่ที่นี่
                        err = processUser(user)
                    } else {
                        err = errors.New("user is not active")
                    }
                } else {
                    err = errors.New("user must be 18 or older")
                }
            } else {
                err = errors.New("invalid email format")
            }
        } else {
            err = errors.New("email is required")
        }
    } else {
        err = errors.New("user cannot be nil")
    }
    
    return err
}
```

#### ✅ ดี:
```go
func validateAndProcessUser(user *User) error {
    if user == nil {
        return errors.New("user cannot be nil")
    }
    
    if user.Email == "" {
        return errors.New("email is required")
    }
    
    if !strings.Contains(user.Email, "@") {
        return errors.New("invalid email format")
    }
    
    if user.Age < 18 {
        return errors.New("user must be 18 or older")
    }
    
    if !user.IsActive {
        return errors.New("user is not active")
    }
    
    return processUser(user)
}
```

### 10. หลีกเลี่ยง Magic Numbers

#### ❌ ไม่ดี:
```go
func calculateDiscount(price float64, customerType string) float64 {
    if customerType == "premium" {
        if price > 1000 {
            return price * 0.15
        }
        return price * 0.10
    }
    
    if customerType == "regular" {
        if price > 500 {
            return price * 0.05
        }
    }
    
    return 0
}

func validatePassword(password string) bool {
    return len(password) >= 8 && 
           regexp.MustCompile(`[A-Z]`).MatchString(password) &&
           regexp.MustCompile(`[0-9]`).MatchString(password)
}
```

#### ✅ ดี:
```go
const (
    PremiumMinPurchase   = 1000.0
    RegularMinPurchase   = 500.0
    PremiumHighDiscount  = 0.15
    PremiumDiscount      = 0.10
    RegularDiscount      = 0.05
    
    MinPasswordLength    = 8
)

const (
    CustomerTypePremium = "premium"
    CustomerTypeRegular = "regular"
)

func calculateDiscount(price float64, customerType string) float64 {
    switch customerType {
    case CustomerTypePremium:
        if price > PremiumMinPurchase {
            return price * PremiumHighDiscount
        }
        return price * PremiumDiscount
        
    case CustomerTypeRegular:
        if price > RegularMinPurchase {
            return price * RegularDiscount
        }
    }
    
    return 0
}

func validatePassword(password string) bool {
    if len(password) < MinPasswordLength {
        return false
    }
    
    hasUppercase := regexp.MustCompile(`[A-Z]`).MatchString(password)
    hasNumber := regexp.MustCompile(`[0-9]`).MatchString(password)
    
    return hasUppercase && hasNumber
}
```

---

## หลักการเพิ่มเติม

### SOLID Principles
- **S** - Single Responsibility Principle
- **O** - Open/Closed Principle  
- **L** - Liskov Substitution Principle
- **I** - Interface Segregation Principle
- **D** - Dependency Inversion Principle

### Clean Code Rules
1. **ทำให้โค้ดอ่านเหมือนร้อยแก้ว** - อ่านแล้วเข้าใจทันที
2. **ใช้ชื่อที่บอกความหมาย** - หลีกเลี่ยงชื่อที่กำกวม
3. **ฟังก์ชันเล็ก** - ทำหน้าที่เดียว ทำดี
4. **ลดการพึ่งพา** - โค้ดควรเป็นอิสระจากกันมากที่สุด
5. **ทดสอบได้** - เขียนโค้ดที่ทดสอบง่าย

---

## ประโยชน์ของ Clean Code

- 🔧 **บำรุงรักษาง่าย** - แก้ไขและเพิ่มฟีเจอร์ได้รวดเร็ว
- 🤝 **ทำงานร่วมกันได้ดี** - ทีมเข้าใจโค้ดของกันและกัน
- 🐛 **Bug น้อย** - โค้ดที่ชัดเจนลด error
- ⚡ **พัฒนาเร็วขึ้น** - ไม่เสียเวลาเดาความหมายของโค้ด
- 💰 **ประหยัดต้นทุน** - ลดเวลาในการพัฒนาและแก้ไข

---

## สรุป
Clean Code ไม่ใช่แค่การเขียนโค้ดให้ทำงานได้ แต่เป็นการเขียนโค้ดให้คนอื่น (รวมถึงตัวเราในอนาคต) อ่านเข้าใจได้ง่าย การลงทุนเวลาเขียน Clean Code จะคุ้มค่าในระยะยาว
