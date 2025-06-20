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
**ปัญหาในโค้ดแย่:**
- **ชื่อตัวแปรสั้นเกินไป**: `d`, `u`, `x`, `a` ไม่บอกความหมาย ทำให้ผู้อ่านต้องเดา
- **ต้องอาศัยคอมเมนต์**: ต้องเขียน `// วันที่` เพื่อบอกว่า `d` คืออะไร ซึ่งเป็นสัญญาณว่าชื่อตัวแปรไม่ดี
- **สับสนง่าย**: ใครอ่านโค้ดต้องเดาว่า `a` คือ `Age`, `u` คือ `User list`
- **เสี่ยงต่อความผิดพลาด**: การอ่าน `x.a > 18` ไม่ชัดเจนเท่า `user.Age > 18` ซึ่งอาจนำไปสู่การตีความผิดและเกิด Bug


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

**ประโยชน์ของการแก้ไข:**
- **อ่านเข้าใจทันที**: ชื่อ `currentDate`, `adultUsers`, `user`, `Age` สื่อความหมายชัดเจน
- **โค้ดอธิบายตัวเอง (Self-Documenting Code)**: ลดความจำเป็นในการเขียนคอมเมนต์เพื่ออธิบายตัวแปร
- **บำรุงรักษาง่าย (Maintainability)**: ง่ายต่อการดูแลและแก้ไขในอนาคต



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
**ปัญหาในโค้ดแย่:**
- **ฟังก์ชันทำหลายอย่างเกินไป**: `generateReport` รับผิดชอบทั้งคำนวณยอดรวม, หาค่าเฉลี่ย, หาออร์เดอร์สูงสุด และสร้างสตริงรายงาน
- **ยากต่อการทดสอบ (Hard to Test)**: การจะทดสอบ logic การคำนวณยอดรวม ต้องทำผ่านฟังก์ชัน `generateReport` เท่านั้น ไม่สามารถทดสอบแยกเฉพาะส่วนได้
- **นำกลับมาใช้ใหม่ไม่ได้ (Not Reusable)**: หากต้องการแค่ "ยอดรวมออร์เดอร์" ในส่วนอื่นของโปรแกรม จะไม่สามารถเรียกใช้ได้ ต้องเขียนโค้ดซ้ำ
- **แก้ไขยากและเสี่ยง**: การแก้ไข logic การหาค่าเฉลี่ย อาจส่งผลกระทบต่อส่วนอื่นๆ ในฟังก์ชันเดียวกันโดยไม่ตั้งใจ

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
**ประโยชน์ของการแก้ไข:**
- **ทดสอบง่าย**: สามารถเขียน Unit Test ให้แต่ละฟังก์ชันย่อย (`calculateTotal`, `calculateAverage`) ได้อย่างง่ายดาย
- **นำกลับมาใช้ใหม่ได้**: สามารถเรียกใช้ `calculateTotal` จากที่ไหนก็ได้ในโปรแกรม
- **แก้ไขได้อย่างปลอดภัย**: การเปลี่ยนวิธีคำนวณใน `calculateAverage` จะไม่กระทบฟังก์ชันอื่น
- **โค้ดสะอาดและเข้าใจง่าย**: แต่ละฟังก์ชันมีหน้าที่ชัดเจน ทำให้ไล่โค้ดได้ง่าย

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
**ปัญหาในโค้ดแย่:**
- **โค้ดซ้ำซ้อน (Duplicated Code)**: โค้ดส่วนของการตรวจสอบ (validate), คำนวณภาษี (tax), บันทึก (save) และส่งอีเมล (send confirmation) ถูกเขียนไว้ในทั้งสองฟังก์ชัน (`ProcessOnlineOrder` และ `ProcessInStoreOrder`)
- **แก้ไขลำบาก**: หาก business logic ของการส่งอีเมลยืนยันมีการเปลี่ยนแปลง เช่น ต้องเพิ่มการแนบใบเสร็จ จะต้องตามไปแก้ไข **ทุกที่** ที่มีโค้ดนี้อยู่
- **เสี่ยงต่อความผิดพลาด**: มีโอกาสสูงที่จะลืมแก้ไขโค้ดที่ซ้ำกันในบางจุด ทำให้ logic ของโปรแกรมไม่สอดคล้องกัน
- **โค้ดรกและยาว**: ทำให้ไฟล์ใหญ่ขึ้นโดยไม่จำเป็น

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

**ประโยชน์ของการแก้ไข:**
- **รวมศูนย์กลางของ Logic (Single Source of Truth)**: สร้างฟังก์ชัน `finalizeOrder` เพื่อรวมโค้ดที่ซ้ำกันไว้ในที่เดียว
- **แก้ไขง่าย**: เมื่อต้องการเปลี่ยน logic การจบออร์เดอร์ ก็แก้ไขที่ `finalizeOrder` ที่เดียว
- **ลดความผิดพลาด**: มั่นใจได้ว่าทุกกระบวนการจะใช้ logic เดียวกันเสมอ
- **โค้ดสั้นและสะอาด**: ลดจำนวนบรรทัดของโค้ดลงอย่างเห็นได้ชัด

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
**ปัญหาในคอมเมนต์แย่:**
- **คอมเมนต์ที่อธิบายว่า "ทำอะไร" (What)**: คอมเมนต์เช่น `// เพิ่ม 1 ให้กับ count` เป็นคอมเมนต์ที่ไม่จำเป็น เพราะคนเขียนโปรแกรมสามารถอ่านโค้ด `count++` แล้วเข้าใจได้ทันที คอมเมนต์แบบนี้เรียกว่า "Noise" หรือ "สิ่งรบกวน"
- **อาจล้าสมัย (Outdated Comments)**: หากมีการแก้ไขโค้ดแต่ลืมแก้คอมเมนต์ จะทำให้เกิดความสับสนอย่างมาก เช่น เปลี่ยนโค้ดเป็น `count += 2` แต่คอมเมนต์ยังเป็น `// เพิ่ม 1 ให้กับ count`
- **ไม่ได้ให้ข้อมูลเชิงลึก**: ไม่ได้อธิบายว่า **"ทำไม" (Why)** ถึงต้องทำสิ่งนั้น

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
**คอมเมนต์ที่ดีควรเป็นอย่างไร:**
- **อธิบาย "ทำไม" (Why)**: บอกเหตุผลเบื้องหลัง หรือ business logic ที่ซับซ้อนซึ่งไม่สามารถเข้าใจได้จากโค้ดเพียงอย่างเดียว เช่น `// คำนวณส่วนลดตาม business rule: ซื้อครบ 1000 บาทได้ส่วนลด 10%`
- **ใช้เป็นหมายเหตุ (Clarification)**: อธิบายสิ่งที่ไม่ชัดเจน หรือเตือนถึงผลข้างเคียงที่อาจเกิดขึ้น
- **TODO Comments**: ใช้ `// TODO:` หรือ `// FIXME:` เพื่อมาร์กจุดที่ต้องกลับมาทำหรือแก้ไขในอนาคต

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
**ปัญหาในโค้ดแย่:**
- **อ่านยากมาก**: ไม่มีการเว้นวรรค, ไม่มีการขึ้นบรรทัดใหม่, ปีกกาไม่ตรงกัน ทำให้สายตาต้องทำงานหนักเพื่อแยกแยะส่วนต่างๆ ของโค้ด
- **ดูไม่เป็นมืออาชีพ**: รูปแบบของโค้ดที่ไม่สม่ำเสมอสะท้อนถึงความไม่ใส่ใจ ซึ่งอาจส่งผลต่อความน่าเชื่อถือของโค้ด
- **ทำให้การหา Bug ยากขึ้น**: เมื่อโค้ดอ่านยาก การไล่หาจุดผิดพลาดทาง logic ก็จะทำได้ยากตามไปด้วย

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
**ประโยชน์ของการแก้ไข:**
- **อ่านง่ายและสบายตา**: การจัดรูปแบบที่สอดคล้องกันทำให้โครงสร้างของโค้ดชัดเจนขึ้น
- **เป็นมาตรฐานเดียวกันทั้งทีม**: เมื่อทุกคนในทีมใช้มาตรฐานการจัดรูปแบบเดียวกัน (เช่น ใช้เครื่องมือ `gofmt` สำหรับ Go หรือ `Prettier` สำหรับ JavaScript) จะทำให้โค้ดของทั้งโปรเจกต์ดูเป็นเอกภาพและทำงานร่วมกันง่ายขึ้น
- **ลดข้อถกเถียงที่ไม่จำเป็น**: ไม่ต้องเสียเวลามาเถียงกันเรื่องสไตล์การจัดโค้ด ให้เครื่องมือจัดการไป

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
**ปัญหาในโค้ดแย่:**
- **การเพิกเฉยต่อ Error (Ignoring Errors)**: การใช้ `_` เพื่อละเว้น `error` ที่ return กลับมาจากการเรียกใช้ฟังก์ชันเป็นสิ่งที่อันตรายมาก เช่น `os.Create` อาจล้มเหลวเพราะไม่มี permission ในการเขียนไฟล์ แต่โปรแกรมจะทำงานต่อเหมือนไม่มีอะไรเกิดขึ้น
- **การใช้ `panic` และ `recover` แบบผิดๆ**: `recover` ถูกใช้เพื่อดักจับ `panic` แต่ในที่นี้มันถูกใช้เพื่อ "ซ่อน" error ทั้งหมด ทำให้ผู้เรียกใช้ฟังก์ชันไม่รู้เลยว่าการทำงานล้มเหลว
- **ให้ข้อมูลกลับที่ไม่ถูกต้อง**: ฟังก์ชัน return `true` เสมอ ทำให้โค้ดที่เรียกใช้เข้าใจว่าการทำงาน "สำเร็จ" ทั้งที่จริงแล้วอาจจะล้มเหลวโดยสิ้นเชิง

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
**ประโยชน์ของการแก้ไข:**
- **จัดการ Error อย่างชัดเจน**: ตรวจสอบ `error` ที่ return กลับมาทุกครั้ง
- **ให้ข้อมูลที่เป็นประโยชน์เมื่อเกิด Error**: ใช้ `fmt.Errorf` และ `%w` (ใน Go 1.13+) เพื่อ "wrap" error เดิม ทำให้สามารถแกะรอยหาต้นตอของปัญหาได้ (Error Chaining)
- **กำหนดประเภทของ Error**: สร้างตัวแปร error เฉพาะ เช่น `ErrUserDataInvalid` เพื่อให้ผู้เรียกใช้สามารถตรวจสอบประเภทของ error และจัดการได้อย่างเหมาะสม
- **Fail Fast**: เมื่อเกิดข้อผิดพลาด ควรหยุดการทำงานและ return error ทันที ดีกว่าปล่อยให้โปรแกรมทำงานต่อด้วยข้อมูลที่อาจไม่ถูกต้อง

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

**ปัญหาในโค้ดแย่:**
- **มีความซับซ้อนสูง (High Cyclomatic Complexity)**: โค้ดมี `if-else if-else` ซ้อนกันหลายชั้น ทำให้มีเส้นทางการทำงาน (Execution Path) หลายแบบ ยากต่อการทำความเข้าใจและทดสอบให้ครบทุกกรณี
- **แก้ไขยาก**: หากต้องการเพิ่มเงื่อนไขใหม่ เช่น "ส่วนลดสำหรับสมาชิกประเภท Gold" จะต้องเข้าไปแก้ไขโครงสร้าง `if` ที่ซับซ้อนนี้ ซึ่งมีความเสี่ยงสูงที่จะทำให้ logic เดิมพัง
- **อ่านไม่รู้เรื่อง**: ต้องใช้เวลาในการไล่ทำความเข้าใจว่าแต่ละเงื่อนไขทำงานอย่างไร และมีความสัมพันธ์กันอย่างไร

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
**ประโยชน์ของการแก้ไข:**
- **แยกส่วนประกอบของ Logic**: แบ่งฟังก์ชันที่ซับซ้อนออกเป็นฟังก์ชันย่อยๆ ที่เข้าใจง่ายและมีหน้าที่เดียว:
    - `calculateBaseCost`: คำนวณราคาพื้นฐานจากน้ำหนัก
    - `calculateDistanceCost`: คำนวณค่าส่งจากระยะทาง
    - `applyDiscounts`: คำนวณส่วนลด
- **อ่านง่ายขึ้น**: โค้ดหลักใน `calculateShippingCost` จะเป็นการเรียกใช้ฟังก์ชันย่อยๆ ที่มีชื่อสื่อความหมาย ทำให้อ่านแล้วเข้าใจภาพรวมได้ทันที
- **ทดสอบและแก้ไขง่าย**: สามารถทดสอบและแก้ไขแต่ละฟังก์ชันย่อยได้อย่างอิสระ โดยไม่ต้องกังวลว่าจะกระทบส่วนอื่น

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
**ปัญหาในโค้ดแย่:**
- **Parameter List ยาวเกินไป**: การที่ฟังก์ชันมี Parameter มากเกินไป (ในกรณีนี้ 11 ตัว) ทำให้เรียกใช้งานได้ยาก
- **เสี่ยงต่อการใส่ค่าผิดตำแหน่ง**: มีโอกาสสูงมากที่จะใส่ค่าสลับตำแหน่งกันโดยที่ Compiler ไม่สามารถตรวจจับได้ เช่น สลับค่า `isActive` กับ `isPremium`
- **อ่าน Signature ของฟังก์ชันไม่รู้เรื่อง**: ทำให้ไม่ชัดเจนว่า Parameter ตัวไหนใช้ทำอะไรบ้าง

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

**ประโยชน์ของการแก้ไข:**
- **จัดกลุ่มข้อมูลที่เกี่ยวข้องกัน**: ใช้ `struct` เพื่อจัดกลุ่ม Parameter ที่มีความสัมพันธ์กันไว้ด้วยกัน เช่น `UserDetails`, `Address`, `UserConfig`
- **เรียกใช้งานง่ายและชัดเจน**: การเรียกใช้ฟังก์ชันจะสั้นลง และเห็นชัดเจนว่าข้อมูลชุดไหนถูกส่งเข้าไป
- **ลดความผิดพลาด**: การกำหนดค่าให้กับ `struct` โดยระบุชื่อ field จะช่วยป้องกันการใส่ค่าผิดตำแหน่งได้
- **มีความยืดหยุ่น**: หากในอนาคตต้องการเพิ่มข้อมูลเกี่ยวกับที่อยู่ เช่น `Province` ก็แค่เพิ่ม field ใหม่เข้าไปใน `Address struct` โดยไม่ต้องแก้ไข Signature ของฟังก์ชัน `createUser`

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
**ปัญหาในโค้ดแย่:**
- **โค้ดซ้อนกันเป็นลูกศร (Arrow Code)**: การใช้ `if-else` ซ้อนกันลึกๆ ทำให้โค้ดเยื้องเข้าไปทางขวาเรื่อยๆ เหมือนหัวลูกศร ซึ่งอ่านและทำความเข้าใจได้ยาก
- **Logic การทำงานหลักอยู่ลึกที่สุด**: ส่วนที่สำคัญที่สุดของฟังก์ชัน (`processUser(user)`) กลับถูกซ่อนอยู่ใน `if` ชั้นในสุด
- **จัดการ Error ไม่ดี**: ต้องใช้ตัวแปร `err` เพื่อเก็บค่า error ที่อาจเกิดขึ้นในแต่ละชั้นของ `if-else`

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
**ประโยชน์ของการแก้ไข (Return Early Pattern หรือ Guard Clauses):**
- **ตรวจสอบเงื่อนไขที่ไม่ถูกต้องก่อน**: ทำการตรวจสอบเงื่อนไขที่ผิดพลาด (Invalid Cases) และ `return` ออกจากฟังก์ชันทันที
- **ลดการเยื้องของโค้ด**: ทำให้โค้ดแบนราบ (Flat) และอ่านง่ายขึ้นอย่างมาก
- **Logic หลักอยู่ตอนท้ายและชัดเจน**: โค้ดส่วนที่ทำงานจริงๆ (Happy Path) จะอยู่ที่ส่วนท้ายของฟังก์ชัน โดยไม่มีการเยื้อง ทำให้เห็นได้ชัดเจน
- **จัดการ Error ง่ายขึ้น**: ไม่ต้องมีตัวแปร `err` มาคอยเก็บค่า สามารถ return error ได้โดยตรง

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
**ปัญหาในโค้ดแย่:**
- **ตัวเลขที่ไม่มีที่มา (Magic Numbers)**: ตัวเลข เช่น `1000`, `0.15`, `8` ปรากฏขึ้นมาในโค้ดโดยไม่มีคำอธิบาย ทำให้ไม่รู้ว่าตัวเลขเหล่านี้มีความหมายว่าอะไร
- **แก้ไขยากและเสี่ยง**: หากเงื่อนไขทางธุรกิจเปลี่ยน เช่น "ยอดซื้อขั้นต่ำสำหรับ Premium ต้องเป็น 1200" จะต้องไปค้นหาและแก้เลข `1000` ทุกจุดที่ปรากฏในโค้ด ซึ่งเสี่ยงต่อการแก้ไขผิดพลาดหรือลืมแก้บางจุด
- **เข้าใจยาก**: ผู้อ่านโค้ดต้องเดาความหมายของตัวเลขเอาเอง

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
**ประโยชน์ของการแก้ไข:**
- **ใช้ค่าคงที่ (Constants) ที่มีชื่อสื่อความหมาย**: ประกาศ `const` ที่มีชื่อชัดเจน เช่น `PremiumMinPurchase`, `MinPasswordLength`
- **ให้ความหมายกับตัวเลข**: ชื่อของ `const` ทำหน้าที่เป็นเอกสารอธิบายความหมายของค่านั้นๆ ไปในตัว
- **แก้ไขง่ายและปลอดภัย (Single Source of Truth)**: เมื่อต้องการเปลี่ยนแปลงค่า ก็แก้ไขที่การประกาศ `const` ที่เดียว ซึ่งจะส่งผลกับทุกที่ที่เรียกใช้ค่านั้น
- **ลดความผิดพลาด**: การใช้ชื่อ `const` แทนตัวเลขโดยตรงช่วยลดโอกาสในการพิมพ์ตัวเลขผิด

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